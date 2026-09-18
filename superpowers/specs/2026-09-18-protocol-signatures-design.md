# Assinaturas de protocolo — design

**Data:** 2026-09-18
**Status:** aprovado (ADR 0016, 2026-09-18)
**Substitui:** o parágrafo "Quatro-olhos" do ADR 0012 (colapso author+publisher permitido; operador de plataforma como reserva do publisher)
**Afeta:** ADR 0009 (ciclo de vida do protocolo), ADR 0012 (RBAC), spec da API de manutenção (`2026-09-17-maintenance-graphql-api-design.md`, §8 e §10) e o Plano 5 da API de manutenção, que será reescrito sobre este design

## 1. Objetivo

Só pessoas autorizadas publicam protocolos. Todo protocolo, antes de ser **publicado** e antes de ser **ativado para uso**, precisa ser verificado e assinado por pelo menos **dois outros usuários da cidade** com o papel de revisor. O mantenedor da API de manutenção pode criar e editar protocolos, mas **nunca** assina nem aprova, e nenhum outro mantenedor pode fazer isso por ele.

## 2. Decisões

| # | Decisão | Escolha |
|---|---|---|
| S1 | Quem assina | Papel novo, `protocol_reviewer`, separado de quem publica. Acumula com `protocol_publisher` |
| S2 | Quem concede o papel de revisor | Só o `municipal_admin` da cidade |
| S3 | O que as assinaturas destravam | A publicação **e** a ativação. Cada uma exige as suas duas assinaturas |
| S4 | Quem não pode assinar uma versão | Qualquer pessoa que tenha **editado** aquela versão, não só quem a criou |
| S5 | Edição depois de assinado | Invalida as assinaturas: a assinatura vale para o conteúdo exato |
| S6 | Quem executa o ato | Um ato separado, de quem tem o papel (publisher publica; publisher ou `municipal_admin` ativa). A segunda assinatura **não** publica nem ativa sozinha |
| S7 | O mantenedor e o ato | Pode executar o ato de publicar ou ativar quando as assinaturas da cidade já existem. Nunca assina |
| S8 | Mesmo revisor na publicação e na ativação | Pode. São decisões diferentes: "o conteúdo está certo" e "é hora de pôr em uso nesta cidade" |
| S9 | Cidade sem dois revisores elegíveis | Fica bloqueada até o `municipal_admin` designar mais revisores. Sem exceção e sem reserva do operador de plataforma |
| S10 | Reversão de emergência | Voltar para a versão que estava em uso imediatamente antes dispensa assinaturas, com step-up e motivo obrigatório. Só anda para trás e não encadeia |
| S11 | Onde a assinatura vive | Tabelas próprias no banco da cidade, que só aceitam acréscimo |

## 3. Papéis

- **`protocol_reviewer`** entra em `Membership::ROLES` e no CHECK `ck_memberships_role` do banco da cidade.
- **Concessão:** um command novo, `GrantRole`, porque hoje um papel só nasce pelo convite (`InviteMember` → `AcceptInvitation`). A **revogação** usa o `RevokeMembership` existente (fim de vigência, nunca `DELETE`). Os dois atos publicam evento de domínio.
- **Só o `municipal_admin` concede ou revoga `protocol_reviewer`** (`MembershipPolicy#manage?`).
- **Validade no momento do ato:** só conta a assinatura de quem **ainda** é `protocol_reviewer` quando a publicação ou a ativação é executada. Um revisor revogado perde as assinaturas pendentes, e elas deixam de contar.

## 4. Ciclo de vida

Os estados continuam os de hoje (`draft`, `in_review`, `published`, `active`, `retired`). O `in_review`, que existia sem uso, passa a ser o estado em que se assina.

```
draft ──enviar p/ revisão──▶ in_review ──publicar (2 assinaturas de publicação)──▶ published ──ativar (2 assinaturas de ativação)──▶ active
  ▲                              │
  └──────────── editar ◀─────────┘   editar volta a draft; as assinaturas do conteúdo antigo deixam de contar
```

| Transição | Quem executa | Exige |
|---|---|---|
| Salvar rascunho | `protocol_author`, mantenedor | Versão em `draft` ou `in_review`. Registra uma contribuição. Em `in_review`, volta a `draft` |
| Enviar para revisão (`draft → in_review`) | `protocol_author`, mantenedor | Versão passa no portão `Protocols::Gate` |
| Assinar | `protocol_reviewer` (usuário da cidade) | Step-up de MFA. Publicação: versão em `in_review`. Ativação: versão em `published` |
| Publicar (`in_review → published`) | `protocol_publisher`, mantenedor | Step-up. Duas assinaturas de publicação válidas (§5). Portão. **Não aceita mais sair de `draft`** |
| Ativar (`published → active`) | `protocol_publisher`, `municipal_admin`, mantenedor | Step-up. Duas assinaturas de ativação válidas (§5). R1 continua valendo |
| Reverter (emergência) | `protocol_publisher`, `municipal_admin`, mantenedor | Step-up e motivo (§6) |
| Aposentar | `protocol_publisher`, mantenedor | Sem mudança. R4 continua: a versão `active` não se aposenta |

**Protocolos existentes.** O que já está `published` ou `active` continua como está, sem assinatura retroativa. As regras valem para todo ato novo, inclusive reativar uma versão antiga (fora a reversão de emergência).

## 5. Dados e regra de assinatura

**Três tabelas novas no banco de cada cidade.** As três só aceitam acréscimo: um trigger recusa `UPDATE` e `DELETE`, no mesmo padrão do `platform_events_maintenance_immutable`.

| Tabela | Uma linha por | Colunas |
|---|---|---|
| `protocol_contributions` | salvamento de rascunho | `protocol_definition_id`, `actor_id`, `actor_kind` (`user` \| `maintainer`), `content_digest`, `created_at` |
| `protocol_signatures` | assinatura | `protocol_definition_id`, `purpose` (`publication` \| `activation`), `signer_user_id`, `content_digest`, `created_at` |
| `protocol_activations` | ato de ativação | `protocol_definition_id`, `kind` (`signed` \| `emergency_revert`), `actor_id`, `actor_kind`, `reason` (só na reversão), `created_at` |

**Digest do conteúdo:** SHA-256 da `definition` serializada em JSON canônico (chaves ordenadas em todos os níveis). A assinatura aponta para esse conteúdo exato.

**Uma assinatura conta para um ato quando, no momento do ato:**

1. a `purpose` é a do ato;
2. o `content_digest` é o da `definition` atual da versão;
3. o signatário **ainda** tem `protocol_reviewer` ativo;
4. o signatário **não tem nenhuma** linha em `protocol_contributions` daquela versão;
5. para a ativação, a assinatura é **posterior à última linha de `protocol_activations` daquela versão**. É assim que o ato consome as assinaturas sem alterar nenhuma linha.

O ato exige **duas assinaturas de signatários distintos** que satisfaçam as cinco condições. Faltando, o command falha com um motivo que diz quantas faltam e quantos revisores elegíveis a cidade tem.

**Assinar (`Protocols::Sign`):**

- Só um `User` da cidade com `protocol_reviewer` assina. O mantenedor é recusado **pelo tipo de ator** (`actor_kind == "maintainer"`), não pelo papel: o ator mantenedor responde "sim" a toda pergunta de papel (D6 da spec da API de manutenção), e sem essa checagem ele passaria.
- Quem contribuiu para a versão é recusado na hora de assinar. A condição 4 é verificada de novo no ato, porque a contribuição pode vir depois da assinatura.
- Publica `protocol.signed` com `purpose`, `version`, `content_digest` e o ator.
- Assinar duas vezes o mesmo conteúdo para a mesma finalidade não soma. A contagem é por signatário distinto.

## 6. Reversão de emergência

`Protocols::RevertActivation` existe porque um protocolo com erro em uso prioriza triagens errado enquanto se esperam duas assinaturas.

- **Alvo:** a versão da linha de `protocol_activations` **anterior** à ativação atual daquele protocolo.
- **Só se a ativação atual for `signed`.** Reverter uma reversão é recusado: sem isso, reverter duas vezes reativaria a versão com erro sem nenhuma assinatura.
- Exige step-up e um motivo não vazio. Grava uma linha `emergency_revert` com o motivo e publica `protocol.activation_reverted` com o motivo, a versão de origem e a de destino.
- A versão com erro volta a `published`, pela mesma demoção da ativação normal.

## 7. A brecha do superusuário

O mantenedor pula a autorização (D6). Se ele pudesse conceder papel ou convidar membros, bastaria criar duas contas de revisor e assinar por elas. Três regras de domínio olham o **tipo de ator**:

1. **`Protocols::Sign`** recusa o mantenedor.
2. **`GrantRole`** recusa o mantenedor quando o papel é `protocol_reviewer` ou `municipal_admin`.
3. **`InviteMember`** recusa o mantenedor quando o convite é para `protocol_reviewer` ou `municipal_admin`.

Isso restringe a futura fatia de "membros" da API de manutenção: o mantenedor gere os demais papéis, mas nunca cria quem aprova nem quem concede aprovação. Não há como fechar isso contra acesso direto ao banco. O que se fecha é o caminho pela aplicação, e o que sobra fica nas tabelas que só aceitam acréscimo e nos eventos.

## 8. Superfícies

| Ato | Painel da cidade (usuário) | API de manutenção (mantenedor) |
|---|---|---|
| Editar rascunho | autor | sim |
| Enviar para revisão | autor | sim |
| Assinar | revisor, com step-up | **nunca** |
| Publicar | publisher, com step-up | sim, com step-up |
| Ativar | publisher ou admin, com step-up | sim, com step-up |
| Reverter | publisher ou admin, com step-up e motivo | sim, com step-up e motivo |
| Aposentar | publisher | sim, com step-up |
| Conceder ou revogar `protocol_reviewer` | `municipal_admin` | **nunca** |

- **API da cidade:** ganha os endpoints que faltam (enviar para revisão, assinar, ativar, reverter, conceder papel). Hoje só existem salvar rascunho (`Authoring::ProtocolsController`) e publicar (`PublicationsController`). O step-up da cidade usa o `MfaStepUp` existente.
- **Leitura:** as superfícies de leitura passam a mostrar, por versão, as assinaturas válidas por finalidade, quantas faltam e quantos revisores elegíveis a cidade tem. Mostram quem assinou (nome de staff), nunca dado de cidadão.
- **Telas do dashboard:** spec própria, no repo do dashboard.

## 9. Auditoria

- Todo ato publica evento de domínio na cidade: `protocol.draft_saved`, `protocol.submitted_for_review`, `protocol.signed`, `protocol.published`, `protocol.activated`, `protocol.activation_reverted`, `protocol.retired`, `membership.granted`. Cada um leva o ator e o `actor_kind`, e, quando o ato vem da API de manutenção, o `correlation_id` da auditoria de plataforma.
- As três tabelas novas são a prova. Os eventos são a trilha, e nenhuma regra é decidida lendo evento.

## 10. Estratégia de teste

**Domínio (commands)**

- Duas assinaturas de revisores distintos que não editaram publicam; uma não publica.
- Quem editou a versão não assina, e quem assinou e depois editou deixa de contar.
- Editar depois de assinado derruba as assinaturas; reenviar o **mesmo** conteúdo para revisão faz as assinaturas daquele digest voltarem a contar.
- Revisor revogado entre a assinatura e o ato não conta.
- O mantenedor não assina, mesmo passando em toda pergunta de papel.
- O mantenedor publica e ativa quando as assinaturas existem, e é recusado quando não existem.
- A ativação consome as assinaturas: reativar a mesma versão depois exige assinar de novo.
- Assinaturas de publicação não servem para ativação, e vice-versa.
- Publicar a partir de `draft` é recusado.
- Reversão: volta para a ativação anterior sem assinaturas, exige motivo, e reverter uma reversão é recusado.
- `GrantRole` e `InviteMember` recusam o mantenedor para `protocol_reviewer` e `municipal_admin`; `GrantRole` recusa quem não é `municipal_admin`.
- Protocolos já `published`/`active` antes da migração continuam como estão.

**Banco**

- Os triggers recusam `UPDATE` e `DELETE` nas três tabelas.
- O CHECK de `memberships` aceita `protocol_reviewer`.
- A migração roda em todas as cidades.

**Guardas**

- Um spec varre `app/commands` e exige que todo command que publica ou ativa protocolo passe pela verificação de assinaturas, para que um caminho novo não contorne a regra.
- A checagem de tipo de ator em `Sign`, `GrantRole` e `InviteMember` tem exemplo próprio com `Maintenance::MaintainerActor`.

## 11. Fatias de entrega

1. **Domínio:** o papel novo, as três tabelas com trigger e migração em todas as cidades, o digest, e os commands (`GrantRole`, `SubmitForReview`, `Sign`, `RevertActivation`, com `SaveDraft`, `Publish`, `Activate` e `InviteMember` alterados). Pode virar dois planos se o tamanho pedir.
2. **API da cidade:** os endpoints que faltam, com step-up.
3. **API de manutenção (Plano 5, reescrito):** fundação da escrita, fechamento do resíduo do Plano 4 (só a classe de exceção que não é de conexão sai) e as mutations de protocolo (salvar rascunho, enviar para revisão, publicar, ativar, reverter, aposentar). Nenhuma mutation de assinatura.
4. **ADRs:** um ADR novo que substitui o parágrafo de quatro-olhos do ADR 0012, e a atualização do ADR 0009.
5. **Telas do dashboard:** spec própria.

## 12. Fora de escopo

- Revisor recusar formalmente ou devolver com comentário, e retirar uma assinatura já dada. O revisor que não concorda não assina. Dá para acrescentar depois sem mudar o modelo.
- Assinatura por token de serviço.
- Assinatura digital com certificado (ICP-Brasil). Aqui, "assinar" é um ato autenticado com step-up de MFA e registrado de forma que só aceita acréscimo.
- Proteção contra quem tem acesso direto ao banco da cidade.
