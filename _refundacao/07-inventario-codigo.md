# Etapa 7 — Inventário de código × corpus v2 (Fase B, READ-ONLY)

- **Movimento:** 2 · **Etapa:** 7 · **Fase A** (correção de lifecycle no v2): **aplicada, gate passado**.
- **Fase B:** read-only. Nenhum código modificado. Branch lida: `setup-multitenant` (HEAD `bca2188`).
- **Saída:** este inventário. Classifica cada divergência para a Etapa 8 (checkpoint) decidir.
- **Armadilha 2 confirmada:** `db/schema.rb` está **obsoleto** (versão `…000040`; o ledger registra "db/schema.rb NÃO atualizado, Rails 8 multi-db quirk"). O estado real é **migrations (até `…000120`) + models + db/admin_schema.rb**. Conclusões abaixo cruzam as três fontes, não só o schema.

---

## B.1 — Estrutural

| v2 espera | No disco | Estado |
|---|---|---|
| repo `api` | `apps/api/` (git próprio, branch `setup-multitenant`) | existe |
| repo `admin-console` | `apps/admin-console/` | existe (conteúdo não inventariado a fundo) |
| repo `dashboard` | `apps/dashboard/` | existe |
| repo `wpda` | `apps/wpda/` | existe |
| repo `contracts` (events/protocols/types/design-tokens) | `packages/protocols`, `packages/types`, **`packages/ui`** | **divergente** — ainda é `packages/` monorepo; `ui` ainda existe (v2 abandona `ui` → `contracts/design-tokens`) |
| repo `docs` | corpus v2 está em `rota-saude/rota-saude/docs/` (fora dos repos git) | existe (Movimento 1) |

Topologia atual = **monorepo `apps/` + `packages/`** (estado do ADR 0018), **não** o multi-repo do ADR 0002/0025. A migração de topologia é trabalho das Etapas 9–11.

## B.2 — Schema (tabelas) — estado real (migrations + models)

Cluster multi-tenant (as "6 tabelas") — **todas implementadas** por migrations além do schema.rb e com model:

| Tabela v2 | Existe? | Idioma | Via | Alinhado? |
|---|---|---|---|---|
| `municipalities` | sim | EN | schema + `…000120` (status) | sim |
| `municipality_channels` | sim | EN | `…000090` + model | sim |
| `consent_terms` | sim | EN | `…000120` + model | sim |
| `alert_recipients` | sim | EN | model | sim |
| `memberships` | sim | EN | `…000070` + model | sim |
| `invitations` | sim | EN | `…000080` + model | sim |
| `unknown_channels` | sim | EN | `…000100` + model | sim |

Domínio e plataforma:

| Tabela v2 | Existe? | Idioma | Observação | Classe |
|---|---|---|---|---|
| `consents` | sim | EN | unique 1-ativo-por-conversa ✓ | ADOTA |
| `conversations` | sim | EN | unique passou de `phone` para `(municipality_id, phone)` via `…000110` ✓ | ADOTA |
| **`triages`** | **`triagens` (PT)** | **PT** | tabela, FKs, `protocol_name`, status check | **AJUSTA/CONFLITO (idioma)** |
| `inbound_messages` | sim | EN | v2 usa `provider_message_id`; código usa `message_id` | AJUSTA (naming menor) |
| `outbound_messages` | sim | EN | idempotency_key único ✓ | ADOTA |
| `processed_events` | sim | EN | unique `(consumer, event_id)` + `municipality_id` ✓ | ADOTA |
| `domain_events` | sim | EN | `municipality_id` **nullable** ✓ (`…000060`); **sem `aggregate_type/aggregate_id`** (removidos na Phase 2.2) | **CONFLITO** (ver C2) |
| `report_snapshots` | sim | EN exceto | coluna **`triagem_id` (PT)** + FK p/ `triagens` | AJUSTA (idioma) |
| `dashboard_metrics` | sim | EN | — | ADOTA |
| `protocol_definitions` | sim | EN | **modelo antigo**: `name`, `municipality_id`, status `draft/active/retired`, índice parcial WHERE active | **AJUSTA + CONFLITO** (ver C3) |
| `active_protocol_versions` | **não** | — | tabela nova da Fase A | **CRIA** (esperado) |
| `users` | sim | EN | `email_address`, `otp_*`, `deactivated_at` ✓ | ADOTA |
| `sessions` / `identities` | sim | EN | gerador Rails 8 + seam ✓ | ADOTA |
| `authors` | **sim (extra)** | EN | tabela não modelada no v2 (token, municipality_id) | AJUSTA/revisar |

**RLS:** implementado (não aparece em schema.rb — armadilha 2). Migrations `…000001` (roles `rota_app`/`rota_admin`), `…000020` (enable RLS data plane), `…000050` (`tenant_isolation` em 9 tabelas). Spec `spec/rls/tenant_isolation_spec.rb` 4/4 verde (sem tenant levanta; USING; WITH CHECK; admin bypass). → **ADOTA** (invariantes 4, 5, 15 do v2 já cobertas no banco).

## B.3 — Estado em-voo da Phase 2 (e além) — ledger `progress.md`

| Phase | Tema | Estado |
|---|---|---|
| 0 | Higiene de ADRs | **fechada** |
| 1 | Infra de RLS (roles, conexões, `municipality_id`, políticas, `Current`, `within_tenant`) | **fechada** (1.1–1.9, specs verde) |
| 2 | Tenant em eventos e jobs | **fechada** (2.1–2.8, 14/14). Removeu `aggregate_*`; `publish` carimba tenant; `IdempotentConsumer`/`TenantScopedJob` reescritos; jobs de ingestão stubbed |
| 3 | Auth, sessão, MFA | **fechada** (8 commits, 27/27). `rotp`, identities, `Authenticator` seam, `Mfa::Enroll/Verify` |
| 4 | Memberships, RBAC, provisionamento, admin | **em curso** (4.5 completa; 4.6/4.7 em brief). memberships/invitations/provision existem; **bug de filter-ordering** (`within_tenant` antes de `require_authentication` → `current_municipality` lê `Current.user` nil) **bloqueia produção** |
| 5 | Borda WhatsApp (ingestão real) | **pendente** — `Whatsapp::Ingest`, `SendWhatsappJob` (corpo preservado em comentário), wiring de `municipality_channels`; webhook hoje levanta `TenantMissing` |

Pendências em-voo registradas no ledger (herdadas, Phase 5): webhook `TenantMissing`; `NotifyCitizenJob → SendWhatsappJob` signature mismatch; bind `inbound_message.received` stale; queries admin com `aggregate_*` quebradas; `Protocols::Publish` é stub (sem authz/audit completos).

## B.4 — Domínio (modelos, comandos, eventos)

**Comandos** (`app/commands/`): `complete_triagem` (**PT**), `give_consent`, `revoke_consent`, `provision_municipality`, `invite_member`, `accept_invitation`, `revoke_membership`, `deactivate_user`, `seed_protocol`, `conversation_advance`, `protocols/publish`, `result`.
- Falta: **`ActivateProtocolVersion`** (CRIA), **`RetireProtocolVersion`** (CRIA — hoje a aposentadoria é inline no `Protocols::Publish`).
- `CompleteTriage` v2 → `complete_triagem` PT (AJUSTA idioma).

**Eventos publicados** (grep): `consent.given`, `consent.revoked`, `membership.granted`, `membership.revoked`, `municipality.provisioned`, `protocol.created`, `protocol.published`, `triagem.completed` (**PT**), `triagem.urgent` (**PT**), `user.deactivated`, `user.invited`.
- `triage.completed`/`triage.urgent` v2 → `triagem.*` PT (AJUSTA idioma).
- `protocol.activated` (CRIA), `protocol.retired` (CRIA). `protocol.created` existe no código, não no v2 (revisar).
- Auditoria platform-scope: `events/platform.rb` (`Platform.audit`) existe ✓.

**Jobs:** cobertura boa — inclui `dispatch_municipality_alert_job` (que a nota v1 dizia "não existe ainda" — **agora existe**), `resend_pending_alerts_job`, `reencryption_job`, purges, `rebuild_dashboard_metrics`, `reconcile_consents`.

## B.5 — Matriz de divergência (classificada)

| Elemento v2 | Tipo | Existe? | Idioma | Classe | Observação |
|---|---|---|---|---|---|
| cluster MT (channels, consent_terms, alert_recipients, memberships, invitations, unknown_channels) | schema | sim | EN | **ADOTA** | Phase 1–4 já implementou |
| RLS + 2 papéis + políticas | infra | sim | — | **ADOTA** | invisível em schema.rb; specs verde |
| `triagens` → `triages` | schema/model | sim (PT) | PT | **CONFLITO** | idioma; Phase 2/3/4 construíram sobre `triagem` |
| `complete_triagem` → `CompleteTriage` | comando | sim (PT) | PT | AJUSTA | idioma |
| `triagem.completed`/`.urgent` → `triage.*` | evento | sim (PT) | PT | AJUSTA | idioma; muda contrato de evento |
| `report_snapshots.triagem_id` → `triage_id` | coluna | sim (PT) | PT | AJUSTA | idioma |
| `domain_events.aggregate_*` | schema | **removido** | — | **CONFLITO** | Phase 2.2 dropou; v2 0004 (reconciliação R-A) mantém |
| `protocol_definitions` modelo | schema | sim (antigo) | EN | **AJUSTA** | status `active`→`in_review/published`; `name`→`protocol_key` |
| `protocol_definitions.municipality_id` | schema | **presente** | EN | **CONFLITO** | código = per-cidade; nota v2 = global (sem municipality_id) |
| `active_protocol_versions` (ponteiro) | schema | **não** | — | **CRIA** | novo (Fase A) |
| `ActivateProtocolVersion` / `RetireProtocolVersion` | comando | **não** | — | **CRIA** | lifecycle novo |
| `protocol.activated` / `protocol.retired` | evento | **não** | — | **CRIA** | lifecycle novo |
| `PublishProtocol` (lifecycle) | comando | sim (stub) | EN | AJUSTA | hoje seta `status='active'`; novo = `in_review→published` |
| `authors` (tabela extra) | schema | sim | EN | AJUSTA | não modelada no v2 |
| topologia multi-repo + `contracts` | estrutural | não (monorepo) | — | **AGUARDA** | Etapas 9–11 (migração de topologia) |
| webhook ingestão real (`Whatsapp::Ingest`) | domínio | stub | — | **AGUARDA** | Phase 5 em-voo |

---

## Conflitos em destaque (input duro da Etapa 8)

**C1 — Idioma PT do núcleo `triagem`/`triagens`.** É a divergência mais espalhada e a
mais delicada. `triagens` (tabela), `triagem_id` (FK em `report_snapshots`),
`complete_triagem` (comando), `triagem.completed`/`triagem.urgent` (eventos),
`protocol_name`, mais specs e queries admin. **A Phase 2/3/4 construiu profundamente
sobre `triagem`** (27/27 specs verde sobre esses nomes). O v2 manda `triage`/`triages`.
Renomear **agora**, com Phase 4 em curso e Phase 5 pendente, é alto risco (toca
migration, model, comando, 2 contratos de evento, FKs, specs). **Decisão da Etapa 8:**
renomear já, renomear após Phase 5 fechar, ou aceitar e adiar. (As demais divergências
de idioma — `triagem_id`, `complete_triagem`, eventos — são satélites desta.)

**C2 — `domain_events.aggregate_type/aggregate_id`.** A Phase 2.2 **removeu**
deliberadamente essas colunas (e o código roda 27/27 verde sem elas). Meu v2 ADR 0004
trouxe a **reconciliação R-A** que as **manteve** (fundindo 0003 + 0020). O código foi
na direção oposta. **Decisão da Etapa 8:** alinhar o v2 ao código (dropar `aggregate_*`
do 0004 — reverter R-A) ou re-adicionar no código. O código verde sem elas sugere
dropar no v2.

**C3 — Protocolo: global (nota v2) × per-cidade (código).** O código tem
`protocol_definitions.municipality_id` (definições por cidade) e single-axis (status
`active`). O modelo da nota de lifecycle (Fase A) é **global** (sem `municipality_id`)
+ two-axis (ponteiro `active_protocol_versions`). **O código confirma que per-cidade
foi a intenção de implementação**, reforçando a tensão já sinalizada com o módulo 03
("autoria por cidade é da cidade"). **Decisão da Etapa 8:** o v2 adota o modelo global
da nota (e então o código migra: dropar municipality_id, criar pointer), ou a nota é
ajustada para per-cidade (mantém municipality_id + adiciona o ponteiro)? Esta decisão
muda o que "alinhado" significa para todo o subsistema de protocolo.

**C4 (operacional, não-bloqueante) — bug de filter-ordering da Phase 4.5.** `within_tenant`
roda antes de `require_authentication` → `current_municipality` lê `Current.user` nil
em produção. Documentado no ledger, contornado só em teste. Não é divergência v2×código,
mas bloqueia o uso real da Phase 4 — relevante para o sequenciamento das Etapas 9+.

---

## A Etapa 8 tem todo o input que precisa?

**Sim.** O inventário entrega: estrutura (B.1), estado real de schema cruzando
migrations+models+ledger (B.2/B.3), domínio (B.4) e a matriz classificada (B.5), com
os 3 conflitos duros (C1 idioma, C2 aggregate_*, C3 protocolo global×per-cidade) e o
blocker operacional (C4) destacados. As decisões de migração por repo (Etapa 8) têm
base para: (a) ordenar idioma vs. estabilidade da Phase em-voo; (b) reconciliar 0004
com a remoção de aggregate_*; (c) fechar o modelo de protocolo antes de implementar a
Fase A no código; (d) sequenciar o fix de filter-ordering e o fechamento da Phase 5.

**Nenhum código foi modificado.** Toda mudança é das Etapas 9–12, após o checkpoint da Etapa 8.
