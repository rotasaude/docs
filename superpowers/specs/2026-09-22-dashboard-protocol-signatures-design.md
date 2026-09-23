# Telas de assinatura de protocolo no dashboard — design

**Data:** 2026-09-22
**Status:** aprovado em conversa (2026-09-22), aguardando revisão do texto
**Afeta:**
- `apps/dashboard`;
- `apps/api`, com três ajustes pequenos (§4);
- o spec de assinaturas `2026-09-18-protocol-signatures-design.md` §8, onde conceder papel privilegiado passa a pedir step-up.

**Base:** ADR 0016 (assinaturas), ADR 0011 (step-up MFA), ADR 0012 (RBAC) e o spec de assinaturas, §8 "Telas do dashboard: spec própria".

## 1. Objetivo

Levar ao painel da cidade o ciclo de vida com assinaturas que a API da cidade já oferece: enviar para revisão, assinar, publicar, ativar, aposentar e reverter. Ao lado vão as duas peças sem as quais esse ciclo não roda numa cidade real:

- **cadastro de TOTP e step-up**, porque assinar e os demais atos exigem uma verificação recente;
- **gestão do papel `protocol_reviewer`**, porque uma cidade sem 2 revisores fica bloqueada e ninguém de fora a destrava.

## 2. Decisões

1. **Escopo:** as três peças (Segurança da conta, Equipe, Protocolos) num spec, entregues em três fatias (§9).
2. **TOTP:** uma página "Segurança da conta", sempre disponível. O login não muda: continua só com senha. Toda ação com step-up, numa conta sem TOTP, aponta para essa página.
3. **Step-up com consciência da janela:** o painel de confirmação só pede o código quando a janela de 5 minutos (`MfaStepUp::STEP_UP_WINDOW`) está fechada. Com a janela aberta, mostra o tempo que falta. Um `mfa_required` inesperado faz o painel pedir o código e repetir a ação **uma** vez.
4. **Protocolos na tela existente:**
   - a coluna "4-olhos", que vem de `domain_events`, dá lugar às colunas de assinatura;
   - o painel de detalhe mostra, por versão, quem assinou, quem editou, os revisores elegíveis e as ações;
   - um indicador "aguardando sua assinatura" filtra a lista.
5. **Equipe:** módulo novo, só para `municipal_admin`. Concede e revoga **apenas** `protocol_reviewer`. Convite, outros papéis e desativação de usuário ficam para outro spec.
6. **Step-up na concessão e na revogação de papel privilegiado** (`protocol_reviewer`, `municipal_admin`). Muda o §8 do spec de assinaturas, que dizia "não em conceder papel".
7. **Um componente compartilhado de ação sensível** (`SensitiveAction`) concentra step-up, repetição e tradução de erros. As telas só declaram a ação. As alternativas descartadas: fluxo próprio por tela, porque duplicaria a parte mais delicada; interceptador global no cliente HTTP, porque esconderia do usuário o que ele está confirmando.
8. **Novo cadastro de TOTP numa conta já cadastrada exige step-up** (§4.2). É uma brecha achada durante este design.

## 3. Papéis e ações

É a tabela do spec de assinaturas §8, conferida em `ProtocolPolicy`:

| status | ação | papel que vê a ação | step-up |
|---|---|---|---|
| `draft` | Enviar para revisão | `protocol_author` | não |
| `in_review` | Assinar publicação | `protocol_reviewer` | sim |
| `in_review` | Publicar | `protocol_publisher` | sim |
| `published` | Assinar ativação | `protocol_reviewer` | sim |
| `published` | Ativar | `protocol_publisher` ou `municipal_admin` | sim |
| `draft`, `in_review`, `published` | Aposentar | `protocol_publisher` | sim |
| `active` | Reverter, só quando `revertible`, com motivo | `protocol_publisher` ou `municipal_admin` | sim |

- A ação só aparece para quem tem o papel. Os papéis vêm de `memberships[].role`, no `/session`.
- A visibilidade é conveniência: **quem decide é a API**, e um 403 ou um 422 aparecem no painel com a mensagem dela.
- A versão ativa nunca oferece Aposentar (R4).

**Motivos de desabilitação**, calculados no front a partir dos dados da leitura:

| ação | motivo | condição |
|---|---|---|
| Assinar | "você editou esta versão" | o usuário atual está em `editors` com `kind: "user"` |
| Assinar | "você já assinou" | o usuário atual está nos `signers` da finalidade |
| Publicar / Ativar | "a cidade tem N revisor(es) elegível(is); são necessários 2" | `eligibleReviewers < 2` (tem precedência) |
| Publicar / Ativar | "falta 1 assinatura" / "faltam N assinaturas" | `missing > 0` na finalidade |

- A finalidade de assinatura segue o status: `in_review` → `publication`, `published` → `activation`.
- O mesmo revisor pode assinar as duas finalidades (ADR 0016).

## 4. Ajustes na API (apps/api)

### 4.1 Step-up em papel privilegiado

`SetupController#grant_role` e `#revoke_membership` passam a exigir `reauthenticated_recently?` (via `MfaStepUp`) quando o papel é um de `Membership::PRIVILEGED_ROLES` (`municipal_admin`, `protocol_reviewer`). Fora da janela, a resposta é `401 { error: "mfa_required" }`, igual às ações de protocolo. Papel não privilegiado continua sem step-up. O §8 do spec de assinaturas é atualizado para dizer isso.

### 4.2 Novo cadastro de TOTP

Hoje, `POST /mfa/enroll` numa conta com `otp_enabled` gera um segredo novo e desliga `otp_enabled` (`Mfa::Enroll`) sem pedir nada. Quem tivesse só a senha, ou uma sessão roubada, cadastraria o próprio autenticador e passaria a fazer step-up. Isso anula o segundo fator.

- **Regra nova:** conta com `mfa_enrolled?` só cadastra de novo com a janela de step-up aberta; sem ela, `401 mfa_required`.
- **Primeiro cadastro:** continua só com a sessão, porque não há fator anterior a pedir.
- **Limite registrado:** quem roubar a sessão de uma conta que nunca cadastrou TOTP pode cadastrar o seu. Mitigar isso (cadastro obrigatório ao receber papel, aviso por e-mail ao cadastrar) fica fora deste spec.

### 4.3 Leitura

Nada novo. `/admin/api/protocols` e `/admin/api/protocols/:id` já trazem, por versão:
- `signatures.{publication,activation}.{signers[{id,email}],missing}`;
- `editors[{kind,id,email}]`;
- `eligibleReviewers`;
- `revertible` (Plano 2 da fatia de assinaturas).

`GET /session` já traz `id`, `mfa_enrolled`, `mfa_verified_at` e `memberships[].role`. `GET /setup/memberships` já lista as memberships ativas com o usuário.

## 5. Frontend (apps/dashboard)

### 5.1 Unidades

- **`src/lib/stepUp.ts`:** função pura `stepUpWindow(session, now)` → `{ enrolled, open, remainingMs }`, calculada de `mfa_enrolled` e `mfa_verified_at` com a janela de 5 min. O hook `useStepUp()` usa essa função, e a função `stepUp(code)` chama `POST /mfa/step_up` e recarrega a sessão (`auth.reload()`).
- **`src/components/SensitiveAction.tsx`:** o painel de confirmação.
  - Entradas: título, descrição, campos extras (ex.: motivo), `requiresStepUp` e `run(fields) → Promise`.
  - Com step-up e a janela fechada, mostra o campo de código; com a janela aberta, "verificação válida por mais N min".
  - Com step-up e sem TOTP, mostra "cadastre seu autenticador", com um link para a Segurança da conta, e não oferece confirmar.
  - Fluxo: se precisar, `stepUp(code)`, depois `run`. Em `mfa_required`, pede o código e repete uma vez.
  - Traduz os erros da §6.
- **`src/lib/protocolActions.ts`:** regra pura. `actionsFor(version, me)` → ações com `disabledReason`; `awaitingMySignature(version, me)` → boolean. `me` = `{ id, roles }`.
- **`src/modules/Security.tsx`:** Segurança da conta.
- **`src/modules/Team.tsx`:** Equipe.
- **`src/modules/Protocols.tsx`:** reescrito conforme a §5.2.
- **`src/lib/api.ts`:** uma função por endpoint novo:
  - `enrollMfa`, `confirmMfa`, `stepUpMfa`;
  - `submitProtocol`, `signProtocol`, `publishProtocol`, `activateProtocol`, `retireProtocol`, `revertProtocol`;
  - `listMemberships`, `grantRole`, `revokeMembership`.

  Todas mandam `Content-Type: application/json`, porque a API recusa com 415 uma escrita por cookie sem JSON.
- **`src/shell/modules.ts`:** os módulos `security` (todos) e `team` (visível só com `municipal_admin`).
- **`vite.config.ts`:** o proxy de dev passa a repassar `/protocols` e `/mfa` também.

### 5.2 Telas

**Protocolos**
- **Lista:**
  - colunas: status, "Publicação N/2", "Ativação N/2" (assinaturas válidas, de `signers.length`) e revisores elegíveis;
  - o KPI "4-olhos colapsado" sai; entra o KPI "aguardando sua assinatura", que ao ser clicado alterna o filtro;
  - `createdBy`/`publishedBy`/`fourEyes` deixam de ser lidos pelo dashboard.
- **Painel de detalhe, por versão:**
  - assinaturas de publicação e de ativação, cada uma com os e-mails e o que falta;
  - editores, por e-mail, com o mantenedor aparecendo como "mantenedor";
  - revisores elegíveis;
  - as ações da §3, cada uma abrindo um `SensitiveAction`.

  Os eventos de `protocol.*` continuam, só como histórico.
- **Depois de cada ação bem-sucedida:** recarrega `protocols`, `protocol-detail` e a sessão.

**Equipe** (só `municipal_admin`)
- A tabela mostra as pessoas ativas: e-mail e papéis, vindos de `GET /setup/memberships` agrupado por usuário.
- Botões "Tornar revisor" (`POST /setup/memberships { user_id, role: "protocol_reviewer" }`) e "Remover revisor" (`POST /setup/memberships/:id/revoke`), ambos por `SensitiveAction` com step-up.
- Quando os revisores ativos são menos de 2, uma faixa avisa: "sem 2 revisores, nenhum protocolo é publicado ou ativado nesta cidade".

**Segurança da conta**
- **Sem TOTP:**
  - O botão "Cadastrar autenticador" chama `POST /mfa/enroll` **só no clique**, nunca num efeito, para o StrictMode não trocar o segredo duas vezes.
  - A tela mostra o QR, gerado localmente com a biblioteca `qrcode` como no maintenance, e a chave em texto. É uma dependência nova do dashboard, a mesma versão usada em `apps/maintenance`. O QR nunca vem de um serviço externo, porque ele carrega o segredo.
  - Os códigos de recuperação aparecem uma vez. Um aviso diz que **cada código vale como o autenticador, uma vez** (`Mfa::Verify` os aceita no step-up), e um "guardei os códigos" libera o campo de confirmação (`POST /mfa/confirm`).
- **Com TOTP:** mostra "autenticador ativo" e a janela, se estiver aberta. "Trocar autenticador" passa por `SensitiveAction`, com step-up (§4.2).
- A chave, o QR e os códigos de recuperação existem só no estado desta tela e somem ao sair dela.

### 5.2.1 Correções da onda final (2026-09-22)

A revisão final encontrou a premissa "nada muda na API" errada:
`Admin::ProtocolsQuery.status_label` colapsava `active` em `published`, e a
tela decidia tudo por essa string. Registrado aqui o que mudou:

- **A leitura passou a devolver `active`.** `status_label` não colapsa mais
  `active` em `published` — a API devolve o status real das cinco fases
  (`draft`, `in_review`, `published`, `active`, `retired`), e o dashboard
  distingue as duas (tom de destaque próprio para `active`, rótulos em
  português na tela). O painel de Eventos também tinha o mesmo problema por
  outro lado: filtrava por `payload ->> 'name'`, mas os commands publicam
  `protocol_key:` — o filtro nunca batia, e a lista de nomes aceitos não
  incluía os eventos de assinatura/ativação/reversão.
- **Schema e Linter saíram da tela.** `protocols_query.rb` sempre devolveu
  strings fixas `"ok"` para os dois — nenhum portão de fato roda por trás
  deles. A lista mostrava um "passou" que não significava nada; as colunas
  foram removidas do dashboard. A API continua devolvendo os dois campos no
  payload, sem uso hoje.
- **Pendência:** a leitura ainda não expõe qual é a versão-alvo de uma
  reversão (a que ficaria ativa depois de `Reverter`) — só que a ação está
  disponível e a regra geral (§6: reverte para a ativação imediatamente
  anterior, que precisa continuar publicada). A tela descreve a regra em
  texto no `SensitiveAction`, mas não nomeia a versão de antemão.
- **O ciclo rodou fim a fim no navegador (plano 2026-09-23).** A pendência
  registrada no §8 — "verificação manual, uma vez no fim da fatia 3" — estava
  em aberto por falta de um elenco de dev pronto: exercitar o ciclo exigia
  cadastrar autenticadores à mão para autor, duas revisoras e publisher a
  cada reset do banco. `SignatureCrew` (`apps/api/lib/signature_crew.rb`),
  ligada em `db/seeds.rb`, cria as quatro contas com TOTP fixo e a versão 2
  em rascunho pelo command de autoria; o `db:seed` imprime o `otpauth://` de
  cada uma. Com isso o ciclo foi percorrido no dashboard (`curitiba.demo`):
  enviar para revisão, as duas revisoras assinando a publicação, o publisher
  publicando, as duas assinando a ativação, o publisher ativando — a v2 foi a
  `active`, a v1 caiu para `published` (permanece revertível), "Aposentar"
  sumiu e "Reverter" apareceu na v2. Uma checagem não foi possível como
  descrita: o autor não tem papel `protocol_reviewer`, então "Assinar
  publicação" nem aparece para ele numa versão em revisão (a regra de
  visibilidade por papel do §3 precede o motivo de desabilitação "você
  editou esta versão") — nenhuma conta do elenco de dev acumula os dois
  papéis ao mesmo tempo para exercitar esse texto especificamente.

## 6. Erros

Todos traduzidos no `SensitiveAction`, num lugar só:

| resposta | tela |
|---|---|
| `401 { error: "mfa_required" }` | pede o código e repete **uma** vez; na segunda vez, mostra o erro |
| `422 { error: "invalid_code" }` do step-up | "código inválido" debaixo do campo; campo limpo |
| `403` | "seu papel não permite esta ação" |
| `422`/`409` com mensagem do domínio | a mensagem da API, no painel, que continua aberto |
| `401` sem ser `mfa_required` | sessão expirada: volta ao login, como hoje |
| rede ou 5xx | "não foi possível concluir — tente de novo" |

- O código TOTP é limpo a cada tentativa e nunca entra no cache do React Query.
- A janela calculada no front é só uma previsão: se ela fechar entre a leitura e o clique, a API responde `mfa_required` e o fluxo da primeira linha resolve.

## 7. Privacidade

- As telas mostram só e-mails de staff da cidade: quem assina, quem edita, a equipe. Nenhum dado de cidadão.
- Os segredos do cadastro de TOTP (chave, códigos de recuperação) nunca são registrados, persistidos nem mandados a outro lugar.

## 8. Estratégia de teste

- **API (rspec, request specs):**
  - Conceder e revogar `protocol_reviewer` e `municipal_admin`: sem janela → `401 mfa_required`, sem membership criada ou revogada; com janela → ok. Um papel não privilegiado continua sem step-up.
  - `POST /mfa/enroll`: conta sem TOTP → ok, sem step-up; conta com TOTP e sem janela → `401 mfa_required`, e o segredo e `otp_enabled` ficam intactos; com janela → ok.
- **Dashboard (Vitest):**
  - `stepUpWindow`: sem TOTP, janela aberta, fechada e exatamente no limite, com relógio falso.
  - `SensitiveAction`: pede ou não o código conforme a janela, não oferece confirmar sem TOTP, repete uma vez em `mfa_required` e não duas, trata cada linha da §6, limpa o código a cada tentativa e usa o campo de motivo.
  - `protocolActions`: ações por status e papel, cada motivo de desabilitação e `awaitingMySignature`, incluindo o revisor que assinou a publicação e ainda pode assinar a ativação.
  - Telas, com fetch simulado: Protocolos (colunas, filtro, detalhe, cada ação chamando o endpoint certo e recarregando), Equipe (agrupamento, conceder, revogar, faixa de menos de 2) e Segurança (cadastro chamado uma vez só sob StrictMode, códigos exibidos uma vez, confirmação e troca com step-up).
  - `modules.ts`: Equipe some para quem não é `municipal_admin`.
- **Verificação manual no navegador, com o stack de dev**, uma vez no fim da fatia 3:
  - o admin concede revisor a duas pessoas;
  - as duas cadastram TOTP e assinam a publicação;
  - o publisher publica;
  - as duas assinam a ativação;
  - o publisher ativa.

  O dashboard não tem Playwright hoje, e montar um e2e fica fora deste spec.

## 9. Fatias de entrega (um plano cada, nesta ordem)

1. **Step-up e Segurança da conta:** §4.2 na API, o proxy, `stepUp.ts`/`useStepUp`, `SensitiveAction`, o módulo Segurança e as funções de MFA em `api.ts`.
2. **Equipe:** §4.1 na API, o módulo Equipe, as funções de membership em `api.ts` e a atualização do §8 do spec de assinaturas.
3. **Assinaturas e ciclo:** `protocolActions.ts`, a tela Protocolos reescrita e as funções de ciclo em `api.ts`.

Cada fatia entrega software que funciona sozinho. A 3 depende da 1; a 2 depende da 1.

## 10. Fora de escopo

- Mudar o login da cidade (pedir TOTP no login, cadastro obrigatório para papel privilegiado).
- Convite de membro, concessão de outros papéis e desativação de usuário na Equipe.
- O editor de protocolo (`ProtocolEditor`), que continua como está.
- Aviso por e-mail ao cadastrar ou trocar o TOTP.
- E2E com Playwright no dashboard.
