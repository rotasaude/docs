# Etapa 9 — Execução `api` (log) · PARADA PARA REPORTE

- **Movimento:** 2 · **Etapa:** 9 · Branch: `reconcile-v2-api` (de `setup-multitenant`).
- **Baseline:** suíte inteira **75 examples, 0 failures** (verde) antes de qualquer mudança.
- **Status:** **parado antes de editar código** — o código contradiz as premissas do plano
  (salvaguarda 6: "se o código revelar [premissa errada], PARAR e reportar").

## Setup feito (reversível, sem mudar código)

- Branch `reconcile-v2-api` criada a partir de `setup-multitenant`.
- Stack Docker subido (`docker compose up -d`); DB de teste preparado
  (`db:prepare` + `db:migrate:admin`, RAILS_ENV=test).
- Baseline verde estabelecido: **75/0**.

## Descoberta que parou a execução: o ledger está obsoleto

O inventário (Etapa 7) leu `progress.md`, que detalhava até ~Phase 4.5 e marcava Phase 5
como pendente. O código real (HEAD `bca2188`) está **muitos commits à frente**. O git log
revela trabalho não narrado no ledger:

```
bca2188 fix(setup): provision_params alert items
d502b87 feat(setup): POST /setup/municipalities devolve invitation
f69609b fix(admin): namespace read-only sob rota_admin (BYPASSRLS)
4bc394f feat(setup): endpoints setup multi-tenant
1d08cd0 feat(auth): gov.br OIDC real
7ae5daa feat(domain): ConversationAdvance follow-ups
5637240 feat(auth): Authenticator::GovBr OIDC
e3ecc1f feat(crypto): ReencryptionJob (rotação AR Encryption)
b6e10ea feat(domain): ConversationAdvance draft
8a1d01b feat(reliability): dedup HTTP em SendWhatsapp
8a62382 fix(controllers): skip_tenant_scope cleanup pós Phase 4.5
60fd19d feat(protocols): Protocols::Publish real com policy + audit
7487b4f fix(admin): resolve_municipality usa memberships
3611c5a fix(admin): Admin::Scoped filtram por muni
1380917 fix(admin): admin queries usam payload JSON no lugar de aggregate_* dropado
```

## Re-mapeamento dos passos api#1–#7 contra o código REAL

| Passo | Premissa do plano | Estado real no código | Veredito |
|---|---|---|---|
| api#1 filter-ordering | bug live, bloqueia produção | `Authentication` faz `prepend_before_action :require_authentication`; `within_tenant` pulado em todo controller de sessão; só `protocols_controller` o roda (auth por token de author, bug não se aplica); commit `8a62382` fez o cleanup | **já tratado** (latente) — no máximo um spec de regressão |
| api#2 Phase 5 webhook | pendente | `Whatsapp::Ingest.call` wired no webhook (HMAC real); `SendWhatsappJob` com corpo real (`with_tenant`+`MunicipalityChannel`+`Outbound`) | **substancialmente feito** — verificar vs ADR 0007 |
| api#3 C2 aggregate_* | confirmar removido | removido + admin queries reescritas p/ payload JSON (`1380917`) | **feito** |
| api#4 C3 protocol lifecycle | per-cidade + `active` status + R4 + ActivateProtocolVersion | schema ainda `status` `draft/active/retired` (modelo antigo); `Protocols::Publish` seta `status='active'` direto; **sem** `in_review/published`, **sem** `ActivateProtocolVersion`, **sem** guarda R4 | **GENUINAMENTE PENDENTE** |
| api#5 remover resíduo | `authors`, `protocol.created` | a verificar uso (prova de não-uso) | pendente (verificar) |
| api#6 rename C1 PT→EN | atômico, último | `triagens`/`triagem`/`complete_triagem`/`triagem.*` ainda PT | **GENUINAMENTE PENDENTE** (o passo grande) |
| api#7 cobertura invariantes | mapear | 75 specs existem; falta mapear invariantes v2 → spec | parcial |

## Trabalho de `api` genuinamente restante (vs v2)

1. **api#4 — C3 lifecycle de protocolo** (o código está no modelo antigo single-axis;
   o v2 quer `published ≠ active` com `active` como status + `ActivateProtocolVersion`
   + guarda R4). **Real.**
2. **api#5 — resíduo** (`authors`, `protocol.created`): prova de não-uso, então remover.
3. **api#6 — rename C1 PT→EN**: o passo de maior risco, ainda não feito. **Real.**
4. **api#7 — cobertura de invariantes**: mapear as do v2 que tocam o `api`.

api#1, #2, #3 estão feitos ou latentes — viram **verificação/confirmação**, não implementação.

## Recomendação

O inventário herdou a defasagem do ledger. Antes de executar, recomendo tratar
api#1–#3 como "confirmar por teste" (rápido) e ir direto ao trabalho real (api#4 C3,
depois api#5, api#6 rename, api#7). Nenhum código foi modificado; baseline 75/0 verde.

**Decisão do autor:** ir direto ao api#4 (C3), confirmando api#1–#3 por teste.

---

## Execução

### api#3 — C2 (`domain_events` sem `aggregate_*`) — CONFIRMADO
- Grep `app/`+`db/`: `aggregate_*` só em (a) comentário, (b) migration `…000030` que os
  remove + seu `down` reversível, (c) migration original `…000009` (revertida pela 000030).
  No schema final estão **fora**. Suíte verde os exercita. **Alinhado ao v2 (C2).**

### api#1 / api#2 — confirmados via baseline verde + leitura de código
- api#1 (filter-ordering): `Authentication` faz `prepend_before_action`; `within_tenant`
  pulado nos controllers de sessão; único uso real é `protocols_controller` (auth por
  token de author). Baseline `spec/auth`+`spec/rls` verdes. **Latente, não bloqueia.**
- api#2 (Phase 5): `Whatsapp::Ingest` wired no webhook (HMAC real), `SendWhatsappJob`
  real. **Substancialmente feito.**

### api#4 — C3 (lifecycle de protocolo per-cidade) — IMPLEMENTADO ✅
**Baseline antes:** 75/0. **Depois:** **82/0** (+7 do spec novo). Nada quebrou.

| Arquivo | Mudança |
|---|---|
| `db/migrate/20260623000010_protocol_lifecycle_states.rb` | **novo** — estende o check constraint de `status` para `draft/in_review/published/active/retired` (reversível) |
| `app/models/protocol_definition.rb` | enum `status` atualizado; scope `published`; comentário de dois eixos |
| `app/commands/protocols/publish.rb` | refatorado: `draft/in_review → published` (não mais `active`); payload `protocol_key`; guarda `:invalid_state` |
| `app/commands/protocols/activate.rb` | **novo** — `ActivateProtocolVersion`: `published → active` (R1), demove a anterior, emite `protocol.activated` |
| `app/commands/protocols/retire.rb` | **novo** — `RetireProtocolVersion`: guarda R4 (falha se `active`), emite `protocol.retired` |
| `app/policies/protocol_policy.rb` | `activate?` (publisher ou municipal_admin) |
| `spec/commands/protocols_lifecycle_spec.rb` | **novo** — INV-protocol-1..4 + publish≠active (7 exemplos) |

**Validação:** as 4 invariantes em spec, todas verdes:
- INV-protocol-1 (R1): ativar `draft` falha `:not_published`; ativar `published` sucede.
- INV-protocol-2: ativar v2 demove v1→`published`; resta 1 active por `(muni, name)`.
- INV-protocol-3: ativar nova versão não altera `protocol_definition_id` de triagem em voo.
- INV-protocol-4 (R4): aposentar `active` falha `:active_in_city`.

**Quirk de migration (multi-db):** o DDL roda via **`db:migrate:admin`** (a conexão
`rota_admin` é dona da tabela); `db:migrate` (primary) falha com "must be owner" — é o
padrão do ledger. Aplicar com `db:migrate:admin`.

**Decisão de design registrada (gap aberto, não-bloqueante):** o fluxo fim-a-fim ganha
dois follow-ups não exigidos pelo gate da etapa: (a) endpoint de ativação (hoje só o
comando existe), (b) comando `Submit` (`draft → in_review`) para os quatro olhos. O
`name → protocol_key` (coluna) ficou como AJUSTA à parte (ambos EN; fora do escopo do C3).

### api#4 — integração
- Commit `d34b051` na branch `reconcile-v2-api`; depois **fast-forward de `main`**
  (decisão do autor): `main` foi de `b0e49ee` (pré-multitenant) para `d34b051`, trazendo
  62 commits (61 de multi-tenant em-voo + C3). Local apenas, sem push.

### api#5 — remover resíduo — **PARADO por prova de USO (não é resíduo)**
A etapa exige prova de não-uso antes de remover; a prova achou **uso vivo** dos dois:
- **`authors` (tabela):** auth por token de author do `protocols_controller`
  (`authenticate_author!`, `current_author = Author.find_by(token:)`, tenant derivado de
  `current_author.municipality`); + `ReencryptionJob` rotaciona `Author.token`. Subsistema funcional.
- **`protocol.created` (evento):** lido por `admin/protocols_query.rb` para `created_by`.

Por salvaguarda ("referência viva → parar e reportar"), **nada removido**. Os dois são
**extras funcionais não modelados no v2** (vindica a leitura "extras legítimos" da Etapa 8 c):
decisão do autor — modelar no v2 (ADR novo) ou remover as *features* (decisão maior, não
limpeza de resíduo). api#5 fica **em espera dessa decisão**.

### api#6 — Rename PT→EN (C1) — IMPLEMENTADO ✅
Branch `rename-pt-en` → commit `071b1e7` → ff `main`. **Suíte 82/0** (mesma contagem; nada perdido).

- **Código:** `triagens→triages`, `triagem_id→triage_id`, `Triagem→Triage`,
  `CompleteTriagem→CompleteTriage`, eventos `triagem.*→triage.*`, vars `triagem→triage`,
  strings-dado consistentes (`"triagem-respiratoria"→"triage-respiratoria"`,
  dimensões `triagens_by_tier→triages_by_tier`). 24 arquivos + 2 git mv.
- **Migration `…000020`:** `rename_table`, `rename_column`, + **UPDATE histórico autorizado**
  de `domain_events` (`triagem.*→triage.*`). Reversível (`down`).
- **Preservado (texto PT ao usuário):** label "Triagens concluídas", valores de locale.
  Keys i18n `triagem_*→triage_*` renomeadas; valores PT intactos. Inflection irregular removida.
- **Verificação:** grep de identificador PT (`triagem_id|complete_triagem|class Triagem|
  :triagens|triagem.completed`) → **ZERO**.

### api#7 — Cobertura de invariantes do v2 (mapa + lacunas)

| Invariante v2 (tocando o api) | Spec | Estado |
|---|---|---|
| Exactly-once por consumer | `jobs/concerns/idempotent_consumer_spec` | ✅ |
| RLS falha-fechada + cross-tenant explícito | `rls/tenant_isolation_spec` | ✅ |
| Consent antes de triagem | `commands/conversation_advance_spec` | ✅ |
| HTTP fora do lock | `jobs/send_whatsapp_job_spec` | ✅ |
| MFA operador/publisher (step-up) | `services/mfa/*`, `controllers/*mfa*`, `publications_controller_spec` | ✅ |
| Control plane RBAC | `policies/protocol_policy_spec`, `models/membership_spec` | ✅ |
| Publicação de protocolo rastreável | `protocols_lifecycle_spec` (emite evento) | ✅ |
| INV-protocol-1..4 | `protocols_lifecycle_spec` | ✅ |
| Auditoria platform-scope (nullable) | `events/platform_spec`, `events/domain_events_spec` | ✅ |
| `domain_events` imutável | `events/domain_events_spec` | a confirmar a asserção |
| **HMAC antes do INSERT** (webhook controller) | — | **LACUNA** (ingest_spec cobre a ingestão, não o 401-antes-da-escrita) |
| **ReportSnapshot imutável** | — | **LACUNA** (sem spec do `GenerateReportJob`/snapshot) |
| **Isolamento de SLA por fila** (queue_as por job) | — | **LACUNA** (sem spec de roteamento de fila) |
| Criptografia em repouso | `jobs/reencryption_job_spec` (rotação) | parcial |

**Lacunas (3):** HMAC-antes-INSERT, ReportSnapshot imutável, SLA por fila. Pela regra do
gate api#7, viram **tarefa**, não bloqueiam (o autor aceita ou agenda). Recomendo
priorizar a do HMAC (segurança) e a do snapshot (prova LGPD).

---

## Fecho da Etapa 9

**api#1–#7 concluídos.** Suíte **82/0**. `main` em `071b1e7` (rename → C3 → multitenant).
Nenhum push.

**Pendências/flags para o autor (não-bloqueantes):**
1. **`authors` / `protocol.created`** — extras funcionais vivos (não resíduo): modelar no
   v2 (ADR novo) ou remover as features? Decisão pendente.
2. **Exceção de imutabilidade** — o rename migrou eventos históricos em `domain_events`
   (autorizado). O invariante #7 do v2 (ADR 0004/0014) deve **registrar essa exceção**
   (não toquei o v2 no meio do código — salvaguarda 6).
3. **Schemas stale** — `db/schema.rb` está em `…000040` (pré-multitenant), não reflete o
   multitenant nem o rename. Fonte da verdade = migrations + `admin_schema`. Regenerar é
   item à parte (quirk multi-db, pré-existente do ledger).
4. **Follow-ups do C3** — endpoint de ativação, comando `Submit` (`draft→in_review`),
   `name→protocol_key` (coluna). Fora do gate da etapa.
5. **3 lacunas de spec** (api#7) acima.

>>> **ETAPA 9 COMPLETA: `api` reconciliado ao v2, suíte 82/0 em `main`. Branch pronta para revisão. Sem push, sem deploy.** <<<
