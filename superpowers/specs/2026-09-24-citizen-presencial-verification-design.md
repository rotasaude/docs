# Validação presencial do cidadão (canal web, subprojeto 2) — design

**Data:** 2026-09-24
**Status:** aprovado em conversa (2026-09-24), aguardando revisão do texto
**Afeta:** `apps/api` (banco de cada cidade: `memberships`, tabelas novas; rotas `/attendance/*` e `/citizen/verification_codes`; histórico do cidadão verificado), `apps/dashboard` (módulo novo "Atendimento") e `apps/wpda` ("Validar no posto" e histórico completo).
**Card:** [rotasaude/dashboard#3](https://github.com/rotasaude/dashboard/issues/3)

**Origem:** o ADR 0017 criou o canal web com identidade em dois níveis. O subprojeto 1 entregou o nível `declared` (CPF declarado + celular confirmado por SMS). Este spec cria o nível `verified`: no primeiro atendimento na UBS, um servidor confere o documento com foto e marca o cidadão como verificado. A conferência é humana; o sistema registra a declaração do servidor. Não há foto guardada nem consulta externa, então tudo roda em desenvolvimento.

## 1. Ponto de partida

- O cidadão é o **par (CPF, celular)** (`citizens`, CPF e celular cifrados de forma determinística por cidade). O mesmo CPF pode estar em vários celulares, e um celular em vários CPFs.
- `citizens.verification_level` já existe com `declared | verified`; hoje todo cadastro nasce `declared` e nada o muda.
- Papéis da cidade (ADR 0012/0016): `municipal_admin`, `protocol_author`, `protocol_publisher`, `protocol_reviewer`, `viewer`. Nenhum é de atendimento. `Membership::PRIVILEGED_ROLES` (`municipal_admin`, `protocol_reviewer`) exige step-up para conceder e revogar.
- Escritas de servidor ficam fora de `/admin/api` (só leitura), em escopos próprios como `/setup` e `/authoring`, com a sessão da cidade.
- Regra do projeto (ADR 0016): as tabelas são a prova, os eventos são a trilha; nenhuma regra de domínio decide lendo `domain_events`.

## 2. Decisões

1. **Papel novo `citizen_verifier`**, concedido e revogado pelo `municipal_admin`. Entra em `PRIVILEGED_ROLES`: conceder e revogar exigem step-up. Validar um cidadão usa só a sessão normal do servidor (sem step-up por atendimento).
2. **Identificação no balcão por código na tela do cidadão.** No wpda, o cidadão gera um código de 6 dígitos (10 min). O servidor digita o CPF do documento e o código. Isso prova o documento (conferido pelo servidor) e a posse do celular com a sessão aberta, sem SMS e sem o servidor ver o telefone.
3. **Os outros pares do mesmo CPF continuam funcionando.** Seguem `declared` e vendo só as próprias triagens (uso pela família). O par verificado passa a ver o **histórico completo do CPF**, com as triagens de outros celulares marcadas pelo celular de origem mascarado.
4. **Troca de celular não tem tela própria.** O celular novo vira um par `declared` e é validado de novo no posto.
5. **O servidor vê só o necessário**: CPF digitado, celular mascarado do par, data do cadastro, nível e uma lista das triagens do CPF com **data** e **protocolo**. Sem prioridade, respostas nem resultado.
6. **Desfazer uma validação**: só o `municipal_admin`, com motivo obrigatório; o próprio validador nunca desfaz a validação que fez. O par volta a `declared` e pode ser validado de novo (linha nova).
7. **Unidade (UBS) não é registrada agora.** A validação registra servidor, cidade e hora. A unidade entra com o módulo 09.
8. **Modelo de dados: tabela de validações que só recebe acréscimos** (abordagem A), com `citizens.verification_level` como leitura rápida atualizada na mesma transação.
9. **Exclusão do cadastro fica fora.** Vai para o ADR do Art. 18 LGPD ([rotasaude/docs#2](https://github.com/rotasaude/docs/issues/2); item em aberto em `docs/adr/README.md`).

## 3. Dados (banco de cada cidade)

- **`memberships`**: `citizen_verifier` entra no CHECK de papéis e em `Membership::ROLES`; também em `Membership::PRIVILEGED_ROLES`.
- **`citizen_verification_codes`** (nova): `citizen_id` (FK), `code_digest`, `expires_at`, `attempts` (inteiro, padrão 0), `consumed_at`, `created_at`/`updated_at`. O digest é HMAC de `"#{citizen_id}:#{code}"` com chave derivada do `secret_key_base` (mesmo padrão do `OtpChallenge`). Gerar um código novo invalida os anteriores não consumidos daquele cidadão.
- **`citizen_verifications`** (nova, só acréscimos): `citizen_id` (FK), `verified_by_user_id` (FK users), `verified_at`, `revoked_at`, `revoked_by_user_id` (FK users), `revoke_reason`, `created_at`.
  - Trigger em `db/city_triggers.sql`: recusa `DELETE`; recusa `UPDATE` que mude qualquer coluna além de `revoked_at`, `revoked_by_user_id` e `revoke_reason`; recusa preencher essas três quando `revoked_at` já não é nulo.
  - Índice único parcial: uma validação ativa (`revoked_at IS NULL`) por cidadão.
  - CHECK: `revoked_at`, `revoked_by_user_id` e `revoke_reason` são todos nulos ou todos preenchidos, e `revoke_reason` tem pelo menos 10 caracteres depois de `btrim`.
- **`citizens.verification_level`**: vira `verified` na mesma transação que cria a validação, e volta a `declared` na que a desfaz.
- **Eventos de trilha**: `citizen.verified` (`citizen_id`, `verification_id`, `verified_by_user_id`) e `citizen.verification_revoked` (`citizen_id`, `verification_id`, `revoked_by_user_id`), declarados com `DomainEvents.bind ..., to: []`. O payload não carrega CPF nem celular.
- As tabelas novas não guardam CPF nem celular; nada entra em `CityEncryption::CITY_KEYED_TARGETS` (se isso mudar no plano, entra).

## 4. Fluxo e rotas

```
wpda — "Minhas triagens" (sessão do cidadão, par declarado)
  POST /citizen/verification_codes {citizen_id}        → 201 {code, expires_at}

dashboard — módulo "Atendimento" (papel citizen_verifier)
  POST /attendance/lookup        {cpf, code}           → 200 {citizen, triages}
       confere o código SEM consumir; conta uma tentativa quando errado
  POST /attendance/verifications {cpf, code, document_checked: true}
                                                       → 201 {verification}
       confere e consome o código sob lock; cria a validação

dashboard — municipal_admin
  POST /attendance/verifications/search {cpf}         → 200 {verifications: [...]}
       (POST e não GET ?cpf=: o CPF não vai na URL — revisto na execução, 2026-09-24)
  POST /attendance/verifications/:id/revoke {reason}   → 200 {verification}
```

- `/citizen/verification_codes` usa a sessão do cidadão (`CitizenAuthentication`); o `citizen_id` precisa ser de um par do celular da sessão (senão 404). Um par já verificado recebe 409 `already_verified`.
- `/attendance/*` usa a sessão da cidade (`Authentication`) e a checagem de papel da cidade; roda dentro de `CityResolution` como as outras rotas de servidor. Toda escrita é JSON.
- **Resposta do lookup:** `citizen` = `{id, cpf_masked, phone_masked, created_at, verification_level}` do par identificado pelo código; `triages` = `[{completed_at ou created_at, protocol_name}]` das triagens de **todos os pares** daquele CPF, mais recentes primeiro. O `protocol_name` é o nome cadastrado do protocolo (ex.: `triage-respiratoria`); não existe hoje um título legível.
- **Histórico do cidadão verificado** (`GET /citizen/triages?citizen_id=` e `GET /citizen/triages/:id`): para um par `verified`, o escopo passa a ser todas as triagens web dos pares com o mesmo CPF; cada item ganha `origin_phone_masked` quando veio de outro par. `revoke_consent` continua valendo só para as triagens do próprio par. Para um par `declared`, nada muda.

## 5. Telas

**wpda — "Minhas triagens"**
- Par `declared`: botão **"Validar no posto"**. Abre uma tela com o código em tamanho grande, contagem regressiva de 10:00, a instrução "Mostre este código e um documento com foto ao atendente" e **"Gerar outro código"**.
- Par `verified`: selo **"Cadastro verificado em dd/mm/aaaa"** no lugar do aviso de cadastro não verificado; o histórico mostra as triagens do CPF, com "feita no celular (**) *****-1234" nas que vieram de outro par.

**dashboard — "Atendimento"** (item de menu visível para `citizen_verifier` e `municipal_admin`)
1. Formulário: CPF (máscara e dígito verificador) e código (6 dígitos). Botão **"Buscar"**.
2. Cartão do cadastro: CPF, celular mascarado, data do cadastro, nível; lista de triagens (data e protocolo); caixa obrigatória **"Conferi o documento com foto e o CPF confere"**; botão **"Validar cadastro"**.
3. Confirmação: **"Cadastro validado"** e **"Próximo atendimento"**, que limpa a tela.

**dashboard — "Histórico de validações"** (só `municipal_admin`): busca por CPF; lista de validações (data, servidor, situação: ativa ou desfeita, com data, admin e motivo); ação **"Desfazer"** com motivo (≥ 10 caracteres) e confirmação.

## 6. Erros

| Situação | Resposta |
|---|---|
| CPF com dígito verificador inválido | 422 `invalid_cpf` (a tela também avisa antes de enviar) |
| Código errado, ou código de outro CPF | 422 `invalid_code` — mesma resposta nos dois casos; conta uma tentativa no código do par, se houver |
| Código vencido / 5 tentativas esgotadas / já usado | 422 `code_expired` / `code_exhausted` / `code_expired`, com a instrução "peça ao cidadão para gerar outro código" |
| `document_checked` ausente ou falso | 422 `document_check_required` |
| Par já verificado (lookup, validação ou gerar código) | 409 `already_verified`, com a data da validação |
| Dois atendentes com o mesmo código | o primeiro valida; o segundo recebe 422 `code_expired` (código já consumido, sob lock) |
| Desfazer validação já desfeita | 409 `already_revoked` |
| O validador tentando desfazer a própria validação | 403 `own_verification` |
| Motivo com menos de 10 caracteres | 422 `reason_too_short` |
| Sem o papel exigido | 403 `forbidden` |
| Sem sessão / banco da cidade atrasado | 401 / 503 (comportamento atual) |

## 7. Segurança e LGPD

- **O atendente não navega pelo cadastro.** O lookup exige CPF e código; sem o cidadão presente com o código, não se vê nada. O histórico por CPF sem código é só do `municipal_admin` e mostra validações, nunca triagens.
- **Código:** HMAC, 6 dígitos, 10 min, 5 tentativas, uso único, um ativo por cidadão. Código que não tem exatamente 6 dígitos é recusado sem contar tentativa (revisto na execução). Limites: geração por sessão do cidadão (10 por hora) e lookup/validação por servidor (30 a cada 10 minutos), com o `RateLimitStore` resolvido por requisição.
- **Logs:** `cpf` e `code` já estão em `filter_parameters`; `reason` também entra.
- **Auditoria:** `citizen_verifications` é a prova (quem validou, quando; quem desfez, quando, por quê); os eventos são trilha. Conceder o papel exige step-up.
- **Minimização:** o atendente vê celular mascarado e só data + protocolo das triagens.

## 8. Testes

### 8.1 API (RSpec, TDD; request specs com `type: :request`)

- **Código de validação:** hash, 10 min, 5 tentativas, uso único, código novo invalida o anterior, par verificado não gera código.
- **`citizen_verifications`:** o trigger recusa `DELETE`, recusa alterar colunas que não sejam as de "desfeita", recusa desfazer duas vezes; o índice impede duas ativas; o CHECK exige os três campos juntos e o motivo mínimo.
- **`verification_level`** acompanha a validação ativa em validar e desfazer.
- **Papel:** `citizen_verifier` na lista; conceder e revogar exigem step-up (specs de papel privilegiado existentes); sem o papel, `/attendance/*` responde 403.
- **Fluxo:** código gerado no wpda → lookup → validação; lookup sem código não devolve nada; código de outro CPF dá o mesmo `invalid_code` de um código errado; dois atendentes com o mesmo código → só um valida.
- **Desfazer:** só `municipal_admin`; o próprio validador recebe 403; motivo obrigatório; o par volta a `declared` e pode ser validado de novo (linha nova).
- **Histórico completo:** par verificado vê as triagens de todos os pares do CPF, com `origin_phone_masked`; par declarado vê só as próprias; depois de desfeita, volta a ver só as próprias; revogar consentimento continua restrito ao próprio par.
- **Contrato:** validar e desfazer não alteram triagens, consentimentos, relatórios nem métricas; os eventos publicados são só `citizen.verified` e `citizen.verification_revoked`.
- **Guardas:** paridade do schema de cidade, guarda de atributos cifrados, registro de eventos; suíte completa com o worker parado.

### 8.2 dashboard e wpda (Vitest + Testing Library)

- **dashboard:** máscara e dígito verificador do CPF; caixa "Conferi o documento" obrigatória; mensagem para cada código de erro; "Próximo atendimento" limpa a tela; o menu "Atendimento" só aparece com o papel; "Desfazer" exige motivo.
- **wpda:** "Validar no posto" só para `declared`; tela do código com contagem e "Gerar outro código"; selo para `verified`; histórico com o celular de origem.

### 8.3 Verificação em desenvolvimento

Em Curitiba: conceder `citizen_verifier` a uma conta da semente `SignatureCrew` (com step-up); gerar o código no wpda; buscar e validar no dashboard; conferir no wpda o selo e o histórico completo; como `municipal_admin`, desfazer e conferir a volta a `declared`.

## 9. Fora do escopo

- Exclusão do cadastro (ADR do Art. 18, [rotasaude/docs#2](https://github.com/rotasaude/docs/issues/2)).
- Status das triagens na lista (finalizada, cancelada, em andamento) — evolução futura.
- Título legível do protocolo — hoje aparece o nome cadastrado; entra junto com o status.
- Unidade (UBS) na validação — módulo 09.
- Foto do documento, consulta externa de CPF e reconciliação com CNS/SUS.
- Bloqueio de pares de intrusos — decidido que os outros pares continuam funcionando.
- Confirmação de atendimento e agendamento — subprojetos 3 e 4.
