# Módulo 26a — Regulação municipal (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 25 em `origin/main` do api** (conferido ao escrever este plano em `139ab0e7`: `Platform::Features::CATALOG` terminando em `surveillance`; última migração de cidade `20261027900002_add_disease_surveillance.rb`; `db/city_schema.rb` em `define(version: 2026_10_27_900002)`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..43)`; `Membership::ROLES` com `surveillance` e `ck_memberships_role` reescrito por `AddDiseaseSurveillance::ROLES` (restaurado à mão em `spec/models/create_analytics_migration_spec.rb` e `create_campaigns_migration_spec.rb`); `AppointmentRequest::KINDS = %w[return referral triage screening care_line]` com `ck_appointment_requests_kind`, `ck_appointment_requests_origin` (`num_nonnulls(origin_attendance_id, origin_triage_id, origin_care_line_enrollment_id) = 1`) e `ck_appointment_requests_care_line` da migração `20261022900003`; `rota_appointment_request_guard` em `db/city_triggers.sql`; `HealthUnits::Drain.move` copiando as colunas de origem; `appointment.scheduled`/`.checked_in`/`.no_show`/`.expired`/`.cancelled`/`.moved`/`.booked`, `appointment_request.closed`/`.moved` ligados a `[]`; `consultation.finalized` e `consultation.addendum_added` ligados a `[Signatures::RequestJob, Surveillance::SafetyNetJob]`; `Attendances::Close.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:)`; `Citizens::RequestErasure.examined_patients?` terminando em `notified?(ids)`). **A Task 0 confere a base: se um nome mudou, PARE e reporte.**

**Goal:** O lado api do módulo 26a (F-26.1 a F-26.7, ADR 0044): interruptor `care_regulation` e papel `care_regulator`; catálogo de alvos reguláveis (especialidades CBO e procedimentos SIGTAP) e lista do enfermeiro; pedido de regulação em rascunho na consulta que vale na finalização e no adendo; fila regulada por cor e espera com autorização em oferta com cota; devolver, negar, cancelar e complementar; rede própria pelo pedido de agendamento do 17 e acompanhamento pelos eventos dele; prestador externo com data e desfecho; falta e segunda falta; convivência com o sistema do estado e CSV; guia de encaminhamento em PDF; situação para o cidadão; retenção LGPD; semente de dev em Curitiba; runbook.

**Architecture:** O pedido (`regulation_requests`) é o estado atual com `version` otimista; o andamento fica em `regulation_request_events` (só acréscimo, motivo cifrado) e cada autorização numa linha própria de `regulation_authorizations` — a cota usada de uma oferta é a contagem das autorizações não liberadas dela (falta não libera, cancelar libera), lida sob `FOR UPDATE` na oferta. Toda mudança passa por `CareRegulation::Transition` (trava, versão, evento, publicação só com ids). A rede própria abre um `AppointmentRequest` `kind: "regulated"` (origem nova `origin_regulation_request_id`) e um consumidor idempotente dos eventos do 17 reconcilia o pedido de regulação pelo **estado atual** do pedido de agendamento (ordem de chegada e redespacho não importam).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec via `bin/test`, Prawn (já no Gemfile, PDFs do 19c/25). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-28-module-26a-care-regulation-design.md` e `docs/adr/0044.md`. Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-28-module-26a-care-regulation-contracts.md` (convenções dos contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12, 24a §14 e 25 §15); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **R**). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md` (camada 2: SISREG, e-SUS Regulação, sistemas estaduais, PPI e consórcios). Lições que valem aqui: contadores sem trilha; ordem de travas estável (oferta → pedido de regulação → pedido de agendamento); recusar em vez de cortar (texto longo → 422 com `path`); vírgula decimal no PDF; todo `Data`/`Struct`/modelo com dado pessoal sem dado em `inspect`/`to_s`/`pretty_print`/`as_json`. "Todas as recusas HL7 de uma vez" e "estrutura HL7 sempre AR" não se aplicam (não há HL7): o validador devolve a **primeira** recusa com `path`, como o 22, o 24a e o 25.

## Desvios e precisões

1. **Rascunho como linha.** O contrato (§3) dá rotas próprias ao pedido na consulta: o rascunho é uma linha `status = 'draft'` (não um item de `draft_items`, como o exame do 19a). Como o pedido de exame, ele **só vale** na finalização ou no adendo: `CareRegulation::Promote` passa todos os `draft` da consulta para `pending` (ou `external` com `state_regulated`) na mesma transação, revalidando alvo e CBO (a lista do enfermeiro pode ter mudado). Rascunho só da autora da consulta; consulta descartada (api#51) apaga os rascunhos dela; designação a outro profissional mantém os rascunhos e a finalização revalida pelo CBO novo. Máximo de 10 pedidos por consulta (`MAX_PER_CONSULTATION`). Interruptor desligado na finalização: os rascunhos ficam como estão (não vencem nem somem) e voltam a aparecer quando ligar.
2. **"Encaminhado" com o interruptor ligado.** `Attendances::Close` ganha `regulation:` (nil = regra de hoje). Com o interruptor utilizável, `referred` **com** `referral_unit_id` segue sendo o `referral` do 17 (outra UBS); `referred` **sem** unidade exige pelo menos um pedido de regulação promovido na mesma finalização (`regulation: 0` → 422 `regulation_request_required`), e a nota fica opcional. Quem lê o interruptor é o chamador, fora do savepoint (Finalize e `AttendancesController#close`). Pedido de regulação convive com qualquer desfecho (`return`, `discharged`…).
3. **Cota.** Uma linha de `regulation_authorizations` por autorização. `used(offer)` = autorizações da oferta com `released_at IS NULL`. Cancelar antes do atendimento libera (`released_at`, `release_reason`); falta e atendimento não liberam; nova autorização depois de uma falta é outra linha. A oferta autorizável é a do **mês corrente no fuso da cidade** (`Time.zone.today.beginning_of_month`), de prestador ativo (unidade ativa, se `own`), do mesmo `target_type`/`target_code` (R10). `care_offers` não guarda `used` (R1).
4. **Ordem e motivo de pular.** A fila exibe por cor efetiva (`COALESCE(color, suggested_color)`: vermelho < amarelo < verde < azul), depois `requested_at`, depois `id` (contrato §4; R8). Autorizar exige `skip_reason` quando existe outro `pending` do **mesmo alvo**, mais antigo (`requested_at` menor, ou igual com `id` menor) e de cor efetiva igual ou mais grave que a cor **resultante** da autorização (`state_regulated` e `returned` não contam: não estão na fila).
5. **Prazo da cor.** `due_on` = data de `requested_at` no fuso da cidade + dias da cor efetiva, lidos de `regulation_risk_deadlines` no momento da conta; recalculado só quando a cor efetiva muda (autorização com cor nova, complemento com cor sugerida nova sem cor confirmada). Mudar a tabela de prazos não mexe em pedido já feito. **Vencido** (`overdue`) = `due_on < hoje` e `status ∈ WAITING_STATUSES` (`pending returned authorized external`).
6. **Rede própria.** Autorizar numa oferta de prestador `own` cria, na mesma transação, um `AppointmentRequest` `kind: "regulated"`, `origin_regulation_request_id`, `citizen` = o par validado mais antigo do paciente (sem par validado → 409 `citizen_not_verified`, nada gravado; R3), `origin_unit` = unidade solicitante, `target_unit` = a unidade do prestador, `appointment_type_key` = `consulta_odontologica` para especialidade `2232*`, senão `consulta_medica` (R13), `priority` `priority` para vermelho e `routine` para as outras, `due_on` = `due_on` do pedido de regulação. Ordem de travas: oferta (`FOR UPDATE`) → pedido de regulação → unidade (`FOR SHARE`) → pares (`FOR NO KEY UPDATE`, como `CareLines::OpenAppointmentRequest`) → criação. O `Drain` do 17 copia `origin_regulation_request_id`.
7. **Acompanhamento da rede própria (`CareRegulation::AppointmentSync`).** Reconcilia pelo estado atual do pedido de agendamento mais novo do pedido de regulação (seguindo `moved_from_request_id`), só com o pedido de regulação em `authorized`/`scheduled`: pedido `scheduled` com horário vivo → `scheduled` (`scheduled_at`, local = nome e endereço da unidade); `open` com o último horário `no_show` → falta (Desvio 8); `open` com o último horário `expired` ou sem horário → `authorized` (limpa a data); `closed` `fulfilled` → `completed`; `closed` `citizen_cancelled`, `consent_revoked` ou `dismissed` (que não seja de `SYNC_NOTES`) → `cancelled` com a cota liberada e `by` nulo (R5); `closed` `moved` → aponta para o pedido novo e reconcilia de novo. O consumidor roda com o interruptor em qualquer estado (é consistência do que já existe; R16).
8. **Falta.** `CareRegulation::Misses.apply!`: `misses += 1`, evento `missed`; com `misses >= 2` → `closed`; senão volta a `pending` (ou `external`, se `state_regulated`), mantendo `requested_at` e `due_on`; limpa data, local e `appointment_request_id`; cota não liberada. Na rede própria, o pedido de agendamento reaberto pelo `Lapse` é encerrado (`dismissed`, nota `SYNC_NOTES[:missed]`). `missed` nunca fica como situação atual — só no andamento (R7).
9. **Cancelar.** De `pending`, `returned`, `external`, `authorized`, `scheduled`. Libera a autorização vigente (`release_reason: "cancelled"`). Na rede própria o pedido de agendamento precisa estar `open` (encerrado como `dismissed`, nota `SYNC_NOTES[:cancelled]`); com horário vivo → 409 `appointment_scheduled` (o 17 não tem cancelamento de horário pela equipe; R4). Negar só de `pending`/`returned`; devolver só de `pending`.
10. **Complementar.** De `returned` para `pending`, por `health_professional` com vínculo ativo na unidade solicitante e CBO que pode pedir o alvo (`Requesters`). A justificativa nova substitui a do pedido; o texto vai também no `reason` do evento `pending` (histórico). `requested_at` não muda.
11. **Prestador externo.** `schedule` vale em `external` ou em `authorized`/`scheduled` de oferta não `own` (remarcar mantém a autorização e o mês); `outcome` vale em `scheduled` não `own`. Externo agendado vira `scheduled` com `state_regulated` mantido (R11).
12. **Grupo 02 do catálogo de exames.** Código SIGTAP do grupo 02 ativo em `exam_catalog_items` é recusado como alvo de regulação (422 `invalid_target`, `path: "target_code"`, `hint: "exam_order"`): ele segue pelo pedido de exame do 21a (R9).
13. **Trilha.** Fila, devolvidos e CSV: um `clinical_record.viewed` por requisição (`access: "regulation"`, `regulation_request_ids`, `user_id`). Detalhe e guia: um por pessoa (`patient_id`, `person_ref`, `access: "regulation"`). Rascunhos na consulta: sem trilha (a consulta já deixa). Contadores e catálogo: sem trilha. Cidadão: `Timeline::Trail.viewed!(person:, access: "citizen")`.

## Valores fixados por este plano

1. **Interruptor** (depois de `surveillance` no `CATALOG`): `Entry.new(key: "care_regulation", description: "Regulação municipal: pedido de regulação na consulta, fila por cor e espera, ofertas e cotas por prestador e convivência com o sistema do estado", requires: %w[feature:clinical_record])`.
2. **`CareRegulation::Catalog`:** `TYPES = %w[specialty procedure]`, `COLORS = %w[red yellow green blue]`, `COLOR_RANK = { "red" => 0, "yellow" => 1, "green" => 2, "blue" => 3 }`, `DEFAULT_DEADLINE_DAYS = { "red" => 7, "yellow" => 30, "green" => 90, "blue" => 180 }`, `DEADLINE_DAYS = 1..365`, `STATUSES = %w[draft pending returned authorized scheduled completed missed denied cancelled closed external]`, `WAITING_STATUSES = %w[pending returned authorized external]`, `QUEUE_STATUSES = STATUSES - %w[draft missed]`, `TERMINAL_STATUSES = %w[completed denied cancelled closed]`, `PROVIDER_KINDS = %w[own contracted consortium ppi]`, `EXTERNAL_SYSTEMS = %w[sisreg care cross other]`, `RELEASE_REASONS = %w[cancelled citizen_cancelled unit_dismissed consent_revoked]`, `PHYSICIAN_PREFIX = "225"`, `NURSE_PREFIX = "2235"`, `DENTIST_PREFIX = "2232"`, `JUSTIFICATION = 10..2000`, `REASON = 10..500`, `PLACE_MAX = 200`, `NOTES_MAX = 500`, `NAME_MAX = 120`, `QUANTITY = 0..9999`, `MAX_MISSES = 2`, `MAX_PER_CONSULTATION = 10`, `EXTERNAL_NUMBER = /\A[0-9A-Za-z.\/-]{1,30}\z/`, `CNES = /\A[0-9]{7}\z/`, `SPECIALTY_CODE = /\A[0-9]{6}\z/`, `PROCEDURE_CODE = /\A[0-9]{10}\z/`, `MONTH = /\A[0-9]{4}-(0[1-9]|1[0-2])\z/`, `TRAIL_ACCESS = "regulation"`, `SYNC_NOTES = { missed: "Falta no horário regulado: o pedido voltou à regulação", cancelled: "Pedido de regulação cancelado pela regulação" }`, `CITIZEN_STATUS = { "pending" => "waiting", "returned" => "returned", "authorized" => "waiting", "external" => "waiting", "scheduled" => "scheduled", "completed" => "completed", "denied" => "denied", "cancelled" => "closed", "closed" => "closed" }`.
3. **Listas:** `limit` padrão 50, máximo 100 (inválido → 422 `invalid_limit`); `next_before` = cursor opaco `CareRegulation::Cursor.encode([rank, requested_at, id])` (base64 url-safe; inválido → 422 `invalid_cursor`). Busca de procedimento: `q` com 3+ caracteres (senão 422 `invalid_query`), até 20 itens.
4. **Dev:** api do worktree na porta **3048**; worktree `apps/api/.claude/mod26a` (com `config/master.key`); branch `feat/mod-26a-care-regulation`; migração de cidade `20261028900001`. Semente em Curitiba (Task 15): reguladora "Rosângela Maria Kowalski" (`regulacao@curitiba.demo`, papel `care_regulator`, OTP fixo por `DEV_CARE_REGULATOR_OTP_SECRET` com padrão `REGULACAOMUNICIPALSEMENTEDEVCWBA`); unidade própria "Centro Municipal de Especialidades Vila Hauer" (`kind: "other"`, CNES de dev pelo `Zlib.crc32` como o `ExamCrew`); prestadores fictícios "Clínica de Olhos Araucária" (`contracted`), "Consórcio Intermunicipal de Saúde Vale do Iguaçu" (`consortium`), "Hospital Regional Campos do Tenente" (`ppi`).

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A`; commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-26.1..F-26.7 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada, começando com `gh`). Um card só vai para Done quando o ÚLTIMO app dele terminar; Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod26a <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host; sem bancos com sufixo; a suíte completa é pedida à sessão "Testes" (Task 16).
- Banco de cidade: `db/city_migrate/20261028900001_add_care_regulation.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_28_900001)`); triggers em `db/city_triggers.sql`, executado no fim do `up` (o dump não representa trigger). Irreversível. Sem migração de plataforma. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error": "<reason>" }` (com `path`/`hint` quando o contrato ou este plano manda); interruptor 403 `{ "error": "feature_disabled", "feature": "care_regulation" }` (sem prontuário utilizável, `feature: "clinical_record"`); step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (o `Authentication` já recusa com 415 `json_required`); registro de outro escopo por id → 404; lista com escopo alheio → 403.
- **CPF, CNS, nome, CID, justificativa, motivos, alvo do paciente (especialidade/procedimento) e notas nunca em log, evento, Analytics, mensagem de erro, URL ou `inspect`.** Só no corpo JSON (`request.request_parameters`). O filtro `target_code` da fila é o único alvo em query (contrato §4) e não leva paciente. Filtros de log novos: `:justification`, `/\Acid10\z/`, `:external_number`, `:schedule_notes`, `:place` (`:reason` já cobre `color_reason` e `skip_reason`; `:note` cobre `notes`).
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids (nunca alvo, cor, CID); `DomainEvents.publish(` é multilinha — confira cada chamada inteira (memória "Grep de linha única engana"). Nenhum `Platform.audit` novo (nada a declarar em `R18_PLATFORM_EVENT_NAMES`; a Task 16 confere).
- Nunca `Current.city =` solto: leia `Current.city`; em seed/job, `CityConnection.with`/`CityScopedJob`/`IdempotentConsumer`. Nenhuma leitura de interruptor dentro de savepoint (Finalize/AddAddendum leem antes).
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); colunas cifradas de tabela só-acréscimo no `rota_reencryption_only(...)` do trigger.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..44)`.
- Papel novo: `ck_memberships_role` reescrito pela migração de cidade com `AddCareRegulation::ROLES`; os dois specs de migração que restauram a lista à mão passam a usar a constante nova.

## Review Focus

1. **Autorização na virada do mês no fuso da cidade** (31/10 às 23:30 em São Paulo, já 01/11 em UTC): a oferta autorizável é a de outubro e a cota contada é a de outubro; a oferta de novembro é 422 `invalid_offer`. Teste na Task 8 (`travel_to` + `Time.use_zone`).
2. **Evento do 17 redespachado ou fora de ordem** (o `appointment.no_show` entregue de novo depois de o pedido ter voltado à fila e sido reautorizado; `appointment.scheduled` chegando depois do `checked_in`): nunca uma segunda falta nem regressão de `completed`. Teste na Task 10.
3. **Unidade própria esvaziada pelo `Drain`** com pedido regulado aberto e com horário: o pedido novo herda `origin_regulation_request_id`, o pedido de regulação passa a apontar para ele e continua `scheduled`. Teste na Task 10.
4. **Paciente sem par validado na hora da autorização** (validação revogada depois da consulta) numa oferta própria: 409 `citizen_not_verified`, nenhuma autorização gravada e a cota intacta. Teste na Task 8.
5. **Rascunho de consulta designada a uma enfermeira** com pedido de especialidade fora da lista municipal: a finalização dela recusa com 403 `cbo_not_allowed` e `index`, e nada é promovido; rascunho de consulta descartada some. Teste na Task 5.

## Mapa de arquivos

**Criar** (todos sob `apps/api/.claude/mod26a/`)
- `vendor/regulation/2026-10-28/` — `tb_registro.txt`, `tb_grupo.txt` e `rl_procedimento_registro.txt` da competência SIGTAP vigente + `SOURCE` (URL, data, SHA-256 do ZIP e dos arquivos); `config/regulation/specialties.yml`, `config/regulation/procedures.yml`.
- `db/city_migrate/20261028900001_add_care_regulation.rb`.
- `app/models/{regulation_request,regulation_request_event,regulation_authorization,care_provider,care_offer,regulation_risk_deadline,nurse_requestable_item}.rb`.
- `app/services/care_regulation/{gate,catalog,targets,requesters,deadlines,quota,transition,misses,own_network,appointment_sync,queue,cursor,json,trail,guide_pdf,state_csv,citizen_view}.rb`.
- `app/commands/care_regulation/{save_draft,discard_draft,promote,authorize,decide,complement,mark_external,schedule_external,record_outcome,save_provider,save_offer,save_deadlines,nurse_items}.rb`.
- `app/controllers/concerns/care_regulation_gate.rb`; `app/controllers/care_regulation/{base,consultation_requests,targets,requests,decisions,returned,guides,exports}_controller.rb`; `app/controllers/care_regulation/admin/{providers,offers,deadlines,nurse_items}_controller.rb`; `app/controllers/citizen_api/regulation_requests_controller.rb`.
- `app/jobs/care_regulation/appointment_sync_job.rb`.
- `lib/regulation_crew.rb`.
- Specs: `spec/services/care_regulation/{targets_yaml,targets,requesters,deadlines,queue,guide_pdf,state_csv,appointment_sync}_spec.rb`, `spec/models/care_regulation_schema_spec.rb`, `spec/commands/care_regulation/{promote,authorize,authorize_concurrency,decide,misses}_spec.rb`, `spec/requests/care_regulation/{consultation_requests,targets,admin,queue,decisions,returned,external,exports,guides}_spec.rb`, `spec/requests/citizen_api/regulation_requests_spec.rb`, `spec/jobs/care_regulation/appointment_sync_job_spec.rb`, `spec/commands/citizens/regulation_erasure_spec.rb`, `spec/lib/regulation_crew_spec.rb`, `spec/support/regulation_helpers.rb`.

**Modificar**
- `app/services/platform/features.rb` (CATALOG), `spec/adr_pointers_spec.rb` (1..44), `config/initializers/filter_parameter_logging.rb`.
- `app/models/membership.rb` (ROLES, PRIVILEGED_ROLES), `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`.
- `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`.
- `app/models/appointment_request.rb` (KINDS, `belongs_to :origin_regulation_request`, `#origin`), `app/commands/health_units/drain.rb` (copia a origem nova).
- `app/commands/consultations/{finalize,add_addendum}.rb`, `app/commands/attendances/close.rb`, `app/controllers/attendances_controller.rb`, `app/commands/consultation_drafts/discard.rb`.
- `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`.
- `app/commands/citizens/request_erasure.rb`.
- `spec/support/clinical_record_helpers.rb` (`SIGTAP` ganha os códigos de regulação da Task 0).
- `db/seeds.rb` (RegulationCrew depois do SurveillanceCrew).

---

### Task 0: Base, cards e catálogo de alvos reguláveis

**Files:**
- Create: `vendor/regulation/2026-10-28/*` + `SOURCE`, `config/regulation/specialties.yml`, `config/regulation/procedures.yml`, `app/services/care_regulation/catalog.rb` (só as constantes de "Valores fixados" item 2), `spec/services/care_regulation/targets_yaml_spec.rb`
- Modify: este plano ("Valores fixados" e Task 3, se a regra de instrumento divergir)

**Interfaces:**
- Produces: `config/regulation/specialties.yml` e `config/regulation/procedures.yml` (lidos por `CareRegulation::Targets`, Task 3); `CareRegulation::Catalog`; os códigos de teste `REGULATION_SIGTAP` (Step 6) para o helper da Task 2.

Forma de `specialties.yml` (uma linha por CBO 2002 regulável; fonte = a tabela de CBO vendorizada do LEDI, `vendor/ledi/8.7.0/docs/cbo.html`, que lista as ocupações `225*` e `2232*` com nome):

```yaml
source: "CBO 2002 — vendor/ledi/8.7.0/docs/cbo.html (Layout e-SUS APS 8.7.0, 'Grupos de CBOs por Modelo de Informação')"
specialties:
  - { code: "225120", name: "Médico cardiologista", group: medical }
  - { code: "225265", name: "Médico oftalmologista", group: medical }
  - { code: "223212", name: "Cirurgião-dentista - endodontista", group: dental }
excluded:
  - { code: "225125", reason: "atenção primária (clínico)" }
  - { code: "225106", reason: "não assistencial (legista)" }
```

Forma de `procedures.yml` (regra aplicada à release SIGTAP ativa da competência, não lista fechada):

```yaml
source: "SIGTAP competência AAAAMM — vendor/regulation/2026-10-28/ (tb_registro, tb_grupo, rl_procedimento_registro)"
groups: ["02", "03", "04"]
bpa_instruments: ["01", "02"]        # BPA consolidado e individualizado
apac_principal_instruments: ["06"]   # APAC (procedimento principal): 26b
examples:
  regulable: ["0301010072", "0209010037", "0409050083"]
  not_regulable: ["0305010107", "0101010010"]
```

Regra (pinada aqui; a Task 0 confere): procedimento regulável ⇔ grupo (2 primeiros dígitos) em `groups` **e** algum instrumento em `bpa_instruments` **e** nenhum instrumento em `apac_principal_instruments`. APAC só como secundário (`07`) **não** exclui (a consulta especializada `0301010072` entra como secundária das OCI e precisa ficar regulável). OCI (grupo `09`) e AIH sem BPA ficam fora pela regra.

- [ ] **Step 1: Mover os cards F-26.1..F-26.7 do board #1 para In Progress** (um `gh project item-edit` por card, comando começando com `gh`; ids dos campos na memória "board-91-features-roadmap").

- [ ] **Step 2: Criar o worktree e conferir a base** — `/opt/homebrew/bin/git -C apps/api worktree add .claude/mod26a -b feat/mod-26a-care-regulation origin/main`; copiar `config/master.key`; conferir `origin/main` e os nomes do PRÉ-REQUISITO. Divergiu → PARE e reporte.

- [ ] **Step 3: Baixar a SIGTAP vigente** num diretório novo e vazio do scratchpad (`curl -sSfL` da página de download da Tabela Unificada do DATASUS, `http://sigtap.datasus.gov.br/tabela-unificada/app/download.jsp`, ZIP `TabelaUnificada_AAAAMM_*.zip`); extrair só `tb_registro.txt`, `tb_registro_layout.txt`, `tb_grupo.txt`, `tb_grupo_layout.txt`, `rl_procedimento_registro.txt`, `rl_procedimento_registro_layout.txt` para `vendor/regulation/2026-10-28/`. `SOURCE`: `fetched_on: 2026-10-28`, `competence: AAAAMM`, `url:`, `sha256:` do ZIP e de cada arquivo (`shasum -a 256`). Fonte fora do ar → registrar `unavailable:` com a URL e PARAR (sem SIGTAP real a regra não é conferida).

- [ ] **Step 4: Conferir a regra** lendo os arquivos com o `Terminology::SigtapReader` (layout do próprio ZIP) num script do scratchpad: (a) `tb_registro` confirma `01`/`02` = BPA e `06`/`07` = APAC principal/secundário — divergiu, corrigir `procedures.yml` e este plano; (b) os 5 exemplos caem do lado certo (`0301010072` consulta médica em atenção especializada, `0209010037` esofagogastroduodenoscopia, `0409050083` postectomia reguláveis; `0305010107` hemodiálise e `0101010010` atividade educativa fora) — exemplo que não existe ou cai do outro lado é trocado por um real do mesmo papel, com nome; (c) contagens por grupo (regulável/total) anotadas no `SOURCE`. Montar `specialties.yml` a partir do `cbo.html`: todos os `225*` e `2232*` da página, menos os não assistenciais ou de APS (`excluded` com motivo: clínico, generalista, MFC e ESF; legista, auditor, sanitarista, do trabalho, de tráfego, epidemiologista, saúde coletiva, odontologia legal e do trabalho; medicina intensiva e anestesiologista; anatomopatologista). Nome em caixa de frase com acento (a página traz as duas grafias: vale a acentuada).

- [ ] **Step 5: Escrever o teste dos YAML**

```ruby
# spec/services/care_regulation/targets_yaml_spec.rb
require "rails_helper"

RSpec.describe "Catálogo de alvos reguláveis (Task 0)" do
  let(:specialties) { YAML.safe_load_file(Rails.root.join("config/regulation/specialties.yml")) }
  let(:procedures) { YAML.safe_load_file(Rails.root.join("config/regulation/procedures.yml")) }

  it "especialidades: CBO 2002 de 6 dígitos, médicas 225* e odontológicas 2232*, sem repetição nem APS" do
    rows = specialties.fetch("specialties")
    expect(rows).not_to be_empty
    rows.each do |row|
      expect(row["code"]).to match(CareRegulation::Catalog::SPECIALTY_CODE)
      expect(row["name"].length).to be_between(3, CareRegulation::Catalog::NAME_MAX)
      prefix = row["group"] == "dental" ? CareRegulation::Catalog::DENTIST_PREFIX : CareRegulation::Catalog::PHYSICIAN_PREFIX
      expect(row["code"]).to start_with(prefix)
    end
    codes = rows.map { |r| r["code"] }
    expect(codes.uniq.size).to eq(codes.size)
    expect(codes).to include("225120", "225265", "225270", "225133", "225250", "225124", "223212", "223248")
    expect(codes).not_to include("225125", "225142", "225170", "225130", "223208", "223293", "225106", "223204")
    expect(specialties.fetch("excluded")).to all(include("code", "reason"))
  end

  it "procedimentos: grupos 02–04, BPA sim e APAC principal não, com exemplos conferidos" do
    expect(procedures).to include("groups" => %w[02 03 04], "bpa_instruments" => %w[01 02], "apac_principal_instruments" => %w[06])
    expect(procedures.dig("examples", "regulable")).to all(match(CareRegulation::Catalog::PROCEDURE_CODE))
    expect(procedures.dig("examples", "not_regulable")).to all(match(CareRegulation::Catalog::PROCEDURE_CODE))
    expect(Rails.root.join("vendor/regulation/2026-10-28/SOURCE").read).to include("sha256:", "competence:")
  end
end
```

- [ ] **Step 6: Rodar e ver falhar** — `bin/test api -w mod26a spec/services/care_regulation/targets_yaml_spec.rb` → FAIL (YAML e `Catalog` ausentes).

- [ ] **Step 7: Escrever os dois YAML e `CareRegulation::Catalog`**; anotar neste plano `REGULATION_SIGTAP` = os 5 códigos conferidos com nome e instrumentos (`{ code => [name, %w[instrumentos]] }`), usados pelo helper da Task 2.

- [ ] **Step 8: Rodar** — o spec → PASS.

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod26a add vendor/regulation config/regulation app/services/care_regulation/catalog.rb spec/services/care_regulation/targets_yaml_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod26a commit -m "feat: vendor the regulable specialty and SIGTAP procedure catalog for module 26a" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 1: Interruptor `care_regulation` e base das rotas

**Files:**
- Create: `app/services/care_regulation/gate.rb`, `app/controllers/concerns/care_regulation_gate.rb`, `app/controllers/care_regulation/base_controller.rb`
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/services/care_regulation/gate_spec.rb`

**Interfaces:**
- Produces: `CareRegulation::Gate::KEY = "care_regulation"`; `Gate.usable?(city) -> bool` (`Platform::Features.usable?`; o `requires` já cobre o prontuário); `Gate.enabled?(city) -> bool` (só `city_features`, para consumidor). Concern `CareRegulationGate#require_care_regulation!` (403 `feature_disabled` com `feature: "care_regulation"`, ou `"clinical_record"` quando só o prontuário está fora — o padrão do `SurveillanceGate#require_surveillance_consultation!`).
- `CareRegulation::BaseController < ApplicationController` com `Authentication`, `MfaStepUp`, `CareRegulationGate`, `wrap_parameters false`, `before_action :require_care_regulation!`, e os auxiliares do `Surveillance::BaseController` (`render_failure`, `render_error`, `forbid`, `not_found`, `uuid?`, `body_param`, `body`, `query_only!`, `page_limit`) com `FAILURE_STATUS = { stale_version: :conflict, invalid_transition: :conflict, quota_exhausted: :conflict, appointment_scheduled: :conflict, citizen_not_verified: :conflict, duplicate_offer: :conflict, cnes_taken: :conflict, unit_taken: :conflict, duplicate_item: :conflict, cbo_not_allowed: :forbidden, not_author: :forbidden, not_found: :not_found }` (o resto 422) e `render_failure` levando `path` e `hint` quando o resultado traz. `regulator? = Current.user.has_role?("care_regulator")`; `admin? = Current.user.has_role?("municipal_admin")`.

- [ ] **Step 1: Teste** — `spec/services/care_regulation/gate_spec.rb`: sem a linha da cidade → `false`; `care_regulation` ligado com `clinical_record` desligado → `usable?` `false` e `enabled?` `true`; os dois ligados → `true`; `Platform::Features.find("care_regulation").requires` = `["feature:clinical_record"]`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod26a spec/services/care_regulation/gate_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a entrada do CATALOG (Valores fixados 1), o `Gate`, o concern e a base; `VALID_RANGE = (1..44)`; filtros de log da Global Constraints.
- [ ] **Step 4: Rodar** — o spec + `spec/adr_pointers_spec.rb spec/services/platform` → PASS.
- [ ] **Step 5: Commit** — `feat: add the care_regulation switch and the regulation route base`.

---

### Task 2: Migração de cidade, modelos, papel `care_regulator` e pedido de agendamento `regulated`

**Files:**
- Create: `db/city_migrate/20261028900001_add_care_regulation.rb`, `app/models/{regulation_request,regulation_request_event,regulation_authorization,care_provider,care_offer,regulation_risk_deadline,nurse_requestable_item}.rb`, `spec/models/care_regulation_schema_spec.rb`, `spec/support/regulation_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `app/models/membership.rb`, `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`, `app/models/appointment_request.rb`, `app/commands/health_units/drain.rb`, `spec/support/clinical_record_helpers.rb`

**Interfaces:**
- Produces (colunas; `enc` = `encrypts` com a chave da cidade, coluna `text`):
  - `memberships`: `ck_memberships_role` com `AddCareRegulation::ROLES` (= `AddDiseaseSurveillance::ROLES` + `care_regulator`, ordenada).
  - `care_providers`: `id`, `name string(120)`, `cnes string(7)` null (CHECK formato; CHECK `kind = 'own' OR cnes IS NOT NULL`), `kind` (CHECK `PROVIDER_KINDS`), `health_unit_id → health_units` null (CHECK `(kind = 'own') = (health_unit_id IS NOT NULL)`), `active bool default true`, `updated_by_user_id → users`, timestamps. Únicos parciais: `cnes WHERE cnes IS NOT NULL` (`idx_care_providers_cnes`), `health_unit_id WHERE health_unit_id IS NOT NULL` (`idx_care_providers_unit`).
  - `care_offers`: `id`, `care_provider_id` null false, `target_type` (CHECK `TYPES`), `target_code string(10)`, `month date` (CHECK `month = date_trunc('month', month)::date`), `quantity int` (CHECK 0..9999), `updated_by_user_id → users`, timestamps. Único `(care_provider_id, target_type, target_code, month)` `idx_care_offers_unique`; índice `(target_type, target_code, month)`.
  - `regulation_risk_deadlines`: `color string` PK (CHECK `COLORS`), `days int` (CHECK 1..365), `updated_by_user_id` null, `updated_at`. O `up` insere os 4 padrões.
  - `nurse_requestable_items`: `id`, `target_type` (CHECK), `target_code string(10)`, `created_by_user_id → users`, `created_at`; único `(target_type, target_code)`.
  - `regulation_requests`: `id`, `patient_id → patients` null false, `consultation_id → consultations` null false, `addendum_id → consultation_addenda` null, `requester_user_id → users` null false, `requester_cbo string(6)` null false, `requester_unit_id → health_units` null false, `target_type` (CHECK), `target_code string(10)`, `target_name string(250)` (snapshot do nome), `cid10 enc`, `justification enc`, `suggested_color` (CHECK `COLORS`), `color` null (CHECK), `color_reason enc` null, `status` (CHECK `STATUSES - ['missed']`), `state_regulated bool default false`, `external_system` null (CHECK `EXTERNAL_SYSTEMS`), `external_number string(30)` null (CHECK `(external_system IS NULL) = (external_number IS NULL)`), `requested_at timestamptz` null (CHECK `(status = 'draft') = (requested_at IS NULL)`), `due_on date` null (CHECK `(status = 'draft') = (due_on IS NULL)`), `misses int default 0` (CHECK 0..2), `appointment_request_id → appointment_requests` null, `scheduled_at timestamptz` null, `schedule_place string(200)` null, `schedule_notes enc` null, `version int default 1` (≥ 1), timestamps. Índices: `(status, target_type, target_code, requested_at)` `idx_regulation_requests_queue`; `patient_id`; `consultation_id`; `(requester_unit_id, status)`; único parcial `appointment_request_id WHERE appointment_request_id IS NOT NULL`.
  - `regulation_request_events`: `id`, `regulation_request_id` null false, `status` (CHECK `STATUSES - ['draft']`), `by_user_id` null, `reason enc` null, `created_at`.
  - `regulation_authorizations`: `id`, `regulation_request_id` null false, `care_offer_id` null false, `month date`, `skip_reason enc` null, `authorized_by_user_id → users` null false, `created_at`, `released_at` null, `release_reason` null (CHECK `RELEASE_REASONS`; CHECK `(released_at IS NULL) = (release_reason IS NULL)`). Índice parcial `care_offer_id WHERE released_at IS NULL` `idx_regulation_authorizations_used`; índice `regulation_request_id`.
  - `appointment_requests`: `origin_regulation_request_id → regulation_requests` null; `ck_appointment_requests_kind` + `regulated`; `ck_appointment_requests_origin` = `num_nonnulls(origin_attendance_id, origin_triage_id, origin_care_line_enrollment_id, origin_regulation_request_id) = 1`; `ck_appointment_requests_care_line` reescrito com `regulated` ao lado de `care_line` (sem raiz, unidades de origem e destino obrigatórias) e `(kind = 'regulated') = (origin_regulation_request_id IS NOT NULL)` em `ck_appointment_requests_regulated`; único parcial `origin_regulation_request_id WHERE origin_regulation_request_id IS NOT NULL AND status IN ('open','scheduled')` `idx_appointment_requests_one_live_regulated`.
- Triggers (bloco `DO $do$ … IF to_regclass('public.regulation_requests') IS NOT NULL …`): `rota_regulation_request_guard` (BEFORE UPDATE OR DELETE: DELETE só de `OLD.status = 'draft'`; fora do rascunho, `patient_id`, `consultation_id`, `requester_unit_id`, `target_type`, `target_code`, `requested_at`, `created_at` imutáveis; `status` saindo de `completed`/`denied`/`cancelled`/`closed` recusado); `regulation_request_events` com `rota_reencryption_only('reason')`; `rota_regulation_authorization_guard` (sem DELETE; só `released_at`/`release_reason` mudam, uma vez de NULL; ou só a re-cifra de `skip_reason` sob a marca de `CityEncryption`); `care_providers`, `care_offers`, `regulation_risk_deadlines` com `rota_no_delete()`; TRUNCATE recusado nas seis (não em `nurse_requestable_items`, que é configuração). `rota_appointment_request_guard` ganha `OR NEW.origin_regulation_request_id IS DISTINCT FROM OLD.origin_regulation_request_id`.
- Modelos: `RegulationRequest` (`encrypts :cid10, :justification, :color_reason, :schedule_notes`; `belongs_to :patient, :consultation, :requester_user, :requester_unit`; `:addendum, :appointment_request` opcionais; `has_many :events, -> { order(:created_at, :id) }`; `has_many :authorizations, -> { order(:created_at, :id) }`; `#effective_color` = `color || suggested_color`; `#current_authorization` = a última sem `released_at`, ou nil; `#overdue?(today = Time.zone.today)`; `inspect`/`to_s`/`pretty_print` só `id`, `status`, `version`; `as_json` só `id`, `status`, `version`), `RegulationRequestEvent` (`encrypts :reason`; `inspect` sem `reason`), `RegulationAuthorization` (`encrypts :skip_reason`; `scope :unreleased`), `CareProvider`, `CareOffer` (`#month_label` `"YYYY-MM"`), `RegulationRiskDeadline` (`self.primary_key = "color"`), `NurseRequestableItem`. `CITY_KEYED_TARGETS` + as 6 colunas cifradas.
- `Membership::ROLES` + `care_regulator`; `PRIVILEGED_ROLES` + `care_regulator` (comentário: "ADR 0044: lê nome, CID e justificativa do pedido de regulação e autoriza vaga — step-up; o mantenedor não concede").
- `AppointmentRequest::KINDS` + `regulated`; `belongs_to :origin_regulation_request, class_name: "RegulationRequest", optional: true`; `#origin` devolve `"regulation"` com `origin_regulation_request_id`. `HealthUnits::Drain.move` copia `origin_regulation_request_id`.
- Helpers (`spec/support/regulation_helpers.rb`): `regulation_city!(enabled: true)` (`exam_city!` + `sigtap_release!` com `REGULATION_SIGTAP` e as linhas de `SigtapProcedureInstrument` de cada código + `Platform::Features.set!(... "care_regulation" ...)`), `care_regulator!` (`staff_with(..., "care_regulator")`), `regulation_patient!(n)` (= `exam_patient!(n)`), `own_provider!(unit)`, `external_provider!(kind: "contracted", cnes:)`, `offer!(provider:, target_type: "specialty", target_code: "225120", quantity: 5, month: Time.zone.today.beginning_of_month)`, `regulation_request!(status: "pending", **over)` (pela Task 5: consulta finalizada com rascunho; avaliado só na chamada), `json_patch(path, params)`, `json_delete(path, params = {})`.

- [ ] **Step 1: Teste** (`spec/models/care_regulation_schema_spec.rb`):

```ruby
require "rails_helper"

RSpec.describe "Esquema da regulação (ADR 0044)" do
  before { regulation_city! }
  let(:doctor_unit) { HealthUnit.create!(name: "UBS Regula #{SecureRandom.hex(2)}", kind: "ubs") }
  let(:doctor) { doctor!(doctor_unit) }
  let(:citizen) { verified_citizen!(7) }
  let(:consultation) { started_consultation!(unit: doctor_unit, doctor: doctor, citizen: citizen) }

  def row(**over)
    RegulationRequest.create!({ patient: consultation.patient, consultation: consultation, requester_user: doctor, requester_cbo: "225125",
                                requester_unit: doctor_unit, target_type: "specialty", target_code: "225120",
                                target_name: "Médico cardiologista", cid10: "I10", justification: "Hipertensão resistente a três fármacos",
                                suggested_color: "yellow", status: "draft" }.merge(over))
  end

  it "rascunho sem data; fora do rascunho, com data e prazo" do
    expect { row(requested_at: Time.current) }.to raise_error(ActiveRecord::StatementInvalid, /ck_regulation_requests/)
    expect { row(status: "pending") }.to raise_error(ActiveRecord::StatementInvalid, /ck_regulation_requests/)
    expect { row(status: "pending", requested_at: Time.current, due_on: Date.current + 30) }.not_to raise_error
  end

  it "triggers: rascunho se apaga, pedido não; alvo e paciente imutáveis; encerrado não volta" do
    draft = row
    expect { draft.delete }.not_to raise_error
    live = row(status: "pending", requested_at: Time.current, due_on: Date.current + 30)
    expect { live.delete }.to raise_error(ActiveRecord::StatementInvalid)
    expect { live.update_columns(target_code: "225265") }.to raise_error(ActiveRecord::StatementInvalid)
    live.update!(status: "denied")
    expect { live.update_columns(status: "pending") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "autorização: libera uma vez; nunca some; oferta e prestador nunca somem" do
    provider = external_provider!
    offer = offer!(provider: provider)
    req = row(status: "authorized", requested_at: Time.current, due_on: Date.current + 30)
    auth = RegulationAuthorization.create!(regulation_request: req, care_offer: offer, month: offer.month, authorized_by_user: care_regulator!)
    auth.update!(released_at: Time.current, release_reason: "cancelled")
    expect { auth.update_columns(released_at: nil, release_reason: nil) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { auth.delete }.to raise_error(ActiveRecord::StatementInvalid)
    expect { offer.delete }.to raise_error(ActiveRecord::StatementInvalid)
    expect { CareProvider.create!(name: "Sem CNES", kind: "contracted") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { offer!(provider: provider) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_care_offers_unique/)
  end

  it "pedido de agendamento regulated: origem única, sem raiz, um vivo por pedido de regulação" do
    req = row(status: "authorized", requested_at: Time.current, due_on: Date.current + 30)
    attrs = { kind: "regulated", origin_regulation_request_id: req.id, citizen: citizen, origin_unit: doctor_unit, target_unit: doctor_unit,
              appointment_type_key: "consulta_medica", priority: "routine", due_on: Date.current + 30 }
    AppointmentRequest.create!(attrs)
    expect { AppointmentRequest.create!(attrs) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_appointment_requests_one_live_regulated/)
    expect { AppointmentRequest.create!(attrs.merge(kind: "referral")) }.to raise_error(ActiveRecord::StatementInvalid)
    expect(AppointmentRequest.new(origin_regulation_request_id: req.id).origin).to eq("regulation")
  end

  it "papel care_regulator aceito e privilegiado; prazos padrão; nada pessoal no inspect" do
    expect { care_regulator! }.not_to raise_error
    expect(Membership::PRIVILEGED_ROLES).to include("care_regulator")
    expect(RegulationRiskDeadline.order(:color).pluck(:color, :days)).to eq([ [ "blue", 180 ], [ "green", 90 ], [ "red", 7 ], [ "yellow", 30 ] ])
    expect(row.inspect).not_to include("Hipertensão", "I10", "225120")
  end
end
```

  E em `spec/commands/health_units/drain_spec.rb` (existente) um exemplo: pedido `regulated` aberto movido pelo `Drain` → o novo tem `origin_regulation_request_id` do antigo e `kind: "regulated"`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod26a spec/models/care_regulation_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`replace_check` do papel e dos CHECKs do pedido de agendamento como a `20261022900003`; `up` termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`; `down` irreversível), o dump à mão, os modelos, os triggers, a cifra, o `Membership`, o `AppointmentRequest`, o `Drain` e os helpers; nos dois specs de migração, a restauração do papel passa a usar `AddCareRegulation::ROLES` (comentário cita o 26a).
- [ ] **Step 4: Rodar** — o spec + `spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/commands/health_units/drain_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: add the care regulation city schema, the care_regulator role and regulated appointment requests`.

---

### Task 3: Alvos, quem pode pedir, lista do enfermeiro e prazos por cor

**Files:**
- Create: `app/services/care_regulation/{targets,requesters,deadlines}.rb`, `app/commands/care_regulation/{nurse_items,save_deadlines}.rb`, `app/controllers/care_regulation/targets_controller.rb`, `app/controllers/care_regulation/admin/{deadlines,nurse_items}_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/services/care_regulation/{targets,requesters,deadlines}_spec.rb`, `spec/requests/care_regulation/targets_spec.rb`, `spec/requests/care_regulation/admin_spec.rb` (parte de prazos e lista)

**Interfaces:**
- Produces:
  - `CareRegulation::Target = Data.define(:type, :code, :name, :group)` (`group` `medical`|`dental`|`procedure`) com `#to_json_hash` → `{ code, name }`.
  - `CareRegulation::Targets.resolve(type:, code:, on: Time.zone.today) -> Result` — `ok(target:)` | `fail(:invalid_target, details: { path: "target_code" })` | `fail(:invalid_target, details: { path: "target_code", hint: "exam_order" })` (Desvio 12). Especialidade: linha do `specialties.yml`. Procedimento: `Terminology::Sigtap.release_for(on.strftime("%Y%m"))`, `SigtapProcedure` da release, regra do `procedures.yml` sobre `SigtapProcedureInstrument` da release.
  - `Targets.search(type:, query:, on:, limit: 20) -> [Target]` (especialidade sem `q` = todas, por nome; procedimento exige `q` ≥ 3: código por prefixo ou nome dobrado com `ClinicalTerms.fold`, só reguláveis).
  - `CareRegulation::Requesters.check(cbo:, target:) -> :ok | :cbo_not_allowed` — `225*` tudo; `2235*` só `NurseRequestableItem` do alvo; `2232*` só especialidade `group == "dental"`; resto `:cbo_not_allowed`.
  - `CareRegulation::Deadlines.days(color) -> Integer`; `Deadlines.due_on(requested_at:, color:) -> Date` (`requested_at.in_time_zone.to_date + days`); `Deadlines.all -> { "red" => 7, … }`.
  - `SaveDeadlines.call(attrs:, by:) -> Result` (os 4 obrigatórios, inteiros 1..365; senão `fail(:invalid_request, details: { path: "<cor>" })`); `NurseItems.add(target_type:, target_code:, by:)` (`fail(:invalid_target)`, `fail(:duplicate_item)`), `NurseItems.remove(target_type:, target_code:)` (`fail(:not_found)`). Evento `care_regulation.settings_changed` (`user_id`).
- Rotas (escopo `/regulation`, módulo `care_regulation`, `as: :regulation`): `get "targets"` (`health_professional`, `care_regulator` ou `municipal_admin`; query `type`, `q`, `limit`) → `{ targets: [ { type, code, name, group, nurse_allowed } ] }` (R2); `get/patch "admin/deadlines"` (`{ red, yellow, green, blue }`); `get/post/delete "admin/nurse_items"` (corpo `{ target_type, target_code }`; GET → `{ items: [ { target_type, target_code, name } ] }`). Leitura do admin: `municipal_admin` ou `care_regulator`; escrita só `municipal_admin` com `require_step_up!`.

- [ ] **Step 1: Testes**
  - `targets_spec.rb`: `resolve(type: "specialty", code: "225120")` → nome do YAML; `"225125"` (excluída) → `invalid_target`; `resolve(type: "procedure", code: "0301010072")` → ok com o nome da SIGTAP; `"0305010107"` (APAC principal) e `"0101010010"` (grupo 01) → `invalid_target`; código regulável sem release ativa na competência → `invalid_target`; `"0202010503"` com `exam_catalog_item!` ativo → `invalid_target` com `hint: "exam_order"`, e com o item inativo → ok se a regra aceitar; `search(type: "procedure", query: "esofago")` traz `0209010037` e não `0305010107`.
  - `requesters_spec.rb`: médico `225125` pode `225120` e `0209010037`; enfermeira `223505` recebe `:cbo_not_allowed` em `225250` até `NurseItems.add(target_type: "specialty", target_code: "225250")`; dentista `223208` pode `223212` e não `225120` nem procedimento; técnico `322205` sempre `:cbo_not_allowed`.
  - `deadlines_spec.rb`: `due_on(requested_at: Time.zone.parse("2026-10-31 23:30"), color: "red")` com fuso `America/Sao_Paulo` → `2026-11-07`; depois de `SaveDeadlines` com `red: 3`, `days("red") == 3`.
  - `targets_spec.rb` (request): profissional lista especialidades (inclui `nurse_allowed`); `q` de 2 letras em procedimento → 422 `invalid_query`; papel sem acesso → 403 `missing_role`; interruptor desligado → 403 `feature_disabled`.
  - `admin_spec.rb` (prazos e lista): admin com step-up altera prazos (200, forma `{ red, yellow, green, blue }`); `blue: 0` → 422 `invalid_request` `path: "blue"`; sem step-up → 401 `mfa_required`; regulador lê (200) e recebe 403 ao escrever; lista do enfermeiro: POST (201), repetido → 409 `duplicate_item`, alvo inválido → 422 `invalid_target`, DELETE (204), DELETE de item ausente → 404.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: resolve regulable targets, requester CBO rules, nurse list and colour deadlines`.

---

### Task 4: Pedido em rascunho na consulta

**Files:**
- Create: `app/commands/care_regulation/{save_draft,discard_draft}.rb`, `app/services/care_regulation/{json,trail}.rb`, `app/controllers/care_regulation/consultation_requests_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/care_regulation/consultation_requests_spec.rb`

**Interfaces:**
- Consumes: `Targets.resolve`, `Requesters.check` (Task 3).
- Produces:
  - `SaveDraft.call(consultation:, attrs:, by:, request: nil) -> Result.ok(request:)` | `fail(:missing_role | :not_found | :not_author | :cbo_not_allowed | :invalid_target | :invalid_cid10 | :invalid_request | :too_many_requests | :invalid_transition)`. `attrs` = `{ type, target_code, cid10, justification, suggested_color, state_regulated? }`; `cid10` por `ClinicalTerms.find("cid10", …)` (gravado sem ponto; ausente na release → `invalid_cid10`); `justification` 10..2000 (fora → `invalid_request` `path: "justification"`); cor fora de `COLORS` → `invalid_request` `path: "suggested_color"`; chave desconhecida → `invalid_request` `path: "<chave>"`; mais de `MAX_PER_CONSULTATION` → `invalid_request` `path: "consultation"`. Consulta `draft` ou `finalized` (descartada → `not_found`); só a autora (`not_author`); CBO = `consultation.cbo_code`. Trava a consulta e depois o rascunho. `requester_*` = autora, `cbo_code` da consulta, unidade do atendimento; `target_name` do alvo. Atualizar pedido fora de `draft` → `invalid_transition`.
  - `DiscardDraft.call(request:, by:) -> Result` (só `draft`, só a autora; `delete`).
  - `CareRegulation::Json.request(r, events: true, today: Time.zone.today) -> Hash` — forma `<regulation_request>` do contrato §2 (paciente pelo `Patients::IdentifierJson` + `id`/`display_name`; `target` = `{ code, name }` do snapshot; `requester` = `{ name: Screenings::Json.staff_name(user), cbo, unit: { id, name } }`; `authorization` da `current_authorization` (ou da última, em `completed`/`closed`) com `provider` e `month` `"YYYY-MM"`; `schedule` com `scheduled_at`; `outcome` = `"completed"` em `completed`, `"missed"` quando o último evento `missed` levou a `closed`, senão `null`; `external` só com número; `events` sem `by` quando `by_user_id` nulo). `Json.list_item(r, today:)` = sem `events`.
  - `CareRegulation::Trail.person_viewed!(user:, patient_id:)` e `Trail.listed!(user:, regulation_request_ids:)` (o `clinical_record.viewed` com `access: "regulation"`, como o `Surveillance::Trail`).
- Rotas (escopo `/attendance`): `get/post "consultations/:id/regulation_requests"`, `patch/delete "consultations/:id/regulation_requests/:rid"` → `care_regulation/consultation_requests` (`health_professional`; `require_care_regulation!`). GET → `{ requests: [<regulation_request>] }` de todos os pedidos da consulta (rascunhos e promovidos), sem trilha; POST → 201 `<regulation_request>`; PATCH → 200; DELETE → 204.

- [ ] **Step 1: Teste** `consultation_requests_spec.rb` (`type: :request`): a médica autora cria rascunho (201; `status: "draft"`, `requested_at: null`, `due_on: null`, `target: { code: "225120", name: "Médico cardiologista" }`, `cid10: "I10"`); edita (`PATCH`, 200) e apaga (`DELETE`, 204); outra médica → 403 `not_author`; enfermeira autora pedindo `225120` → 403 `cbo_not_allowed`; `cid10: "Z999"` → 422 `invalid_cid10`; `justification` com 2001 caracteres → 422 `invalid_request` `path: "justification"` (recusado, não cortado); procedimento APAC → 422 `invalid_target` `path: "target_code"`; 11º pedido → 422; `PATCH` em pedido já `pending` → 409 `invalid_transition`; corpo com `text/plain` → 415 `json_required`; interruptor desligado → 403 `feature_disabled`; o log da requisição não contém a justificativa nem `I10` (`capture_log`); nenhum `clinical_record.viewed` publicado.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: let the consultation author draft regulation requests`.

---

### Task 5: Promoção na finalização e no adendo; "encaminhado" vira regulação

**Files:**
- Create: `app/commands/care_regulation/promote.rb`, `app/services/care_regulation/transition.rb`
- Modify: `app/commands/consultations/{finalize,add_addendum}.rb`, `app/commands/attendances/close.rb`, `app/controllers/attendances_controller.rb`, `app/commands/consultation_drafts/discard.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/care_regulation/promote_spec.rb`, `spec/requests/consultations_spec.rb` (existente: exemplos novos)

**Interfaces:**
- Consumes: `Deadlines.due_on`, `Targets.resolve`, `Requesters.check`.
- Produces:
  - `CareRegulation::Transition.call(request:, to:, by:, reason: nil, version: nil, changes: {}) -> Result` — exige o pedido já travado pelo chamador; `version` dado e diferente → `fail(:stale_version)`; grava `status`, `changes`, `version + 1`, um `RegulationRequestEvent(status: to, by_user: by, reason:)` e publica `regulation_request.<evento>` com `regulation_request_id` e `user_id` (nulo sem autor); `EVENT_FOR = { "pending" => "created" (só na promoção) | "complemented", "external" => "external", "authorized" => "authorized", "returned" => "returned", "denied" => "denied", "cancelled" => "cancelled", "scheduled" => "scheduled", "completed" => "completed", "missed" => "missed", "closed" => "closed" }` — o chamador passa `event:` quando o mesmo `to` serve a dois eventos.
  - `CareRegulation::Promote.call(consultation:, by:, addendum: nil, now: Time.current) -> Result.ok(count:)` | `fail(reason, details: { index:, path: })` — dentro da transação do chamador; trava os `draft` da consulta em `order(:created_at, :id)`; revalida alvo (na data de hoje) e `Requesters.check` com o `cbo_code` atual da consulta; `requester_user` = `by`, `requester_cbo` = `consultation.cbo_code`; `requested_at = now`, `due_on = Deadlines.due_on(requested_at: now, color: suggested_color)`, `addendum_id`; `pending` (evento `created`) ou `external` com `state_regulated` (evento `created` e situação `external`).
  - `Attendances::Close.call(..., regulation: nil)` (Desvio 2): `regulation` inteiro ≥ 0 liga a regra nova; `fail(:regulation_request_required)` (422).
  - `Consultations::Finalize`: lê `CareRegulation::Gate.usable?(Current.city)` fora da transação; depois de `consultation.update!(status: "finalized", …)` e antes do `Close`, `Promote` (falha → rollback com o `Result`); `Close` recebe `regulation: regulating ? count : nil`. `AddAddendum`: depois do `apply!`, `Promote(addendum:)` com o interruptor lido fora. `AttendancesController#close` passa `regulation: (CareRegulation::Gate.usable?(Current.city) ? 0 : nil)`. `ConsultationDrafts::Discard` apaga os rascunhos de regulação da consulta na mesma transação.
  - Eventos novos (todos só ids, sem consumidor): `regulation_request.created`, `.complemented`, `.external`, `.authorized`, `.returned`, `.denied`, `.cancelled`, `.scheduled`, `.completed`, `.missed`, `.closed`, `care_regulation.settings_changed`, `care_offer.changed`, `care_provider.changed` (R12).

- [ ] **Step 1: Teste** (`spec/commands/care_regulation/promote_spec.rb`):

```ruby
require "rails_helper"

RSpec.describe "Promoção dos pedidos de regulação (consulta e adendo)" do
  before { regulation_city! }
  let(:unit) { HealthUnit.create!(name: "UBS Promove #{SecureRandom.hex(2)}", kind: "ubs") }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(11) }
  let(:attrs) { { "type" => "specialty", "target_code" => "225120", "cid10" => "I10",
                  "justification" => "Hipertensão resistente a três fármacos", "suggested_color" => "yellow" } }

  def draft!(consultation, by: doctor, **over)
    CareRegulation::SaveDraft.call(consultation: consultation, attrs: attrs.merge(over.transform_keys(&:to_s)), by: by).payload.fetch(:request)
  end

  def finalize!(consultation, outcome: { "outcome" => "referred" }, by: doctor)
    Consultations::SaveDraft.call(consultation: consultation, params: draft_body, by: by)
    Consultations::Finalize.call(consultation: consultation.reload, outcome_params: outcome, by: by)
  end

  it "finalizar com 'encaminhado' sem unidade promove o rascunho e não abre o referral do 17" do
    travel_to Time.zone.parse("2026-10-28 10:00") do
      consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      request = draft!(consultation)
      result = finalize!(consultation)
      expect(result).to be_ok
      expect(request.reload).to have_attributes(status: "pending", due_on: Date.new(2026, 11, 27), requester_user_id: doctor.id)
      expect(request.events.map(&:status)).to eq([ "pending" ])
      expect(result.payload[:appointment_request]).to be_nil
      expect(DomainEvent.where(name: "regulation_request.created").last.payload).to eq("regulation_request_id" => request.id, "user_id" => doctor.id)
    end
  end

  it "'encaminhado' sem unidade e sem pedido, com o interruptor ligado → regulation_request_required; com unidade, o referral de hoje" do
    consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    expect(finalize!(consultation, outcome: { "outcome" => "referred", "referral_note" => "Cardiologia" }).reason).to eq(:regulation_request_required)
    other = HealthUnit.create!(name: "UBS Destino #{SecureRandom.hex(2)}", kind: "ubs")
    result = finalize!(consultation.reload, outcome: { "outcome" => "referred", "referral_unit_id" => other.id })
    expect(result.payload[:appointment_request].kind).to eq("referral")
  end

  it "regulado pelo estado nasce external, fora da fila" do
    consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    request = draft!(consultation, state_regulated: true)
    finalize!(consultation, outcome: { "outcome" => "discharged" })
    expect(request.reload).to have_attributes(status: "external", state_regulated: true)
  end

  it "adendo promove rascunho criado depois da finalização" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    request = draft!(consultation)
    result = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "Pedido de cardiologia esquecido",
                                             text: "Acrescento encaminhamento à cardiologia.")
    expect(result).to be_ok
    expect(request.reload).to have_attributes(status: "pending", addendum_id: result.payload[:addendum].id)
  end

  it "Review Focus 5: consulta designada à enfermeira recusa o pedido fora da lista; descartada apaga o rascunho" do
    consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    request = draft!(consultation)
    nurse = screener!(unit, cbo: "223505")
    consultation.update_columns(author_user_id: nurse.id, cbo_code: "223505") # caminho real: ConsultationDrafts::Reassign
    result = finalize!(consultation.reload, by: nurse)
    expect(result.reason).to eq(:cbo_not_allowed)
    expect(result.details).to include(index: 0)
    expect(request.reload.status).to eq("draft")
    ConsultationDrafts::Discard.call(consultation_id: consultation.id, reason: "Rascunho órfão da enfermeira",
                                     by: staff_with("adm-#{SecureRandom.hex(2)}@cidade.gov.br", "municipal_admin"))
    expect(RegulationRequest.exists?(request.id)).to be(false)
  end

  it "interruptor desligado: o rascunho fica rascunho e o 'encaminhado' segue a regra de hoje" do
    consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    request = draft!(consultation)
    Platform::Features.set!(city: Current.city, key: "care_regulation", enabled: false, maintainer: ledi_maintainer!)
    expect(finalize!(consultation, outcome: { "outcome" => "referred", "referral_note" => "Cardiologia" })).to be_ok
    expect(request.reload.status).to eq("draft")
  end
end
```

  (`ConsultationDrafts::Discard.call(consultation_id:, reason:, by:, city: Current.city)` é a assinatura em `origin/main`; o `reason` segue o mínimo do comando.) Em `spec/requests/consultations_spec.rb`: `POST /attendance/attendances/:id/close` com `outcome: "referred"` só com nota e o interruptor ligado → 422 `regulation_request_required`; desligado → 200 como hoje.
- [ ] **Step 2: Rodar** — `bin/test api -w mod26a spec/commands/care_regulation/promote_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** `Transition`, `Promote`, as mudanças no `Finalize`/`AddAddendum`/`Close`/`AttendancesController`/`Discard` e os bindings (`to: []`, com o bloco de spec "care regulation bindings (ADR 0044)").
- [ ] **Step 4: Rodar** — o spec + `spec/requests/consultations_spec.rb spec/commands/consultations spec/commands/attendances spec/commands/consultation_drafts spec/jobs/signatures spec/jobs/surveillance/safety_net_job_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: promote regulation drafts on finalization and addendum and route specialist referrals through regulation`.

---

### Task 6: Prestadores, ofertas e cotas

**Files:**
- Create: `app/services/care_regulation/quota.rb`, `app/commands/care_regulation/{save_provider,save_offer}.rb`, `app/controllers/care_regulation/admin/{providers,offers}_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/care_regulation/admin_spec.rb` (parte de prestadores e ofertas)

**Interfaces:**
- Produces:
  - `CareRegulation::Quota.used(offer_ids) -> Hash{id => Integer}` (contagem de `RegulationAuthorization.unreleased` por oferta, uma consulta); `Quota.available(offer, used:) -> Integer` (`quantity - used`, nunca negativo).
  - `SaveProvider.call(attrs:, by:, provider: nil) -> Result.ok(provider:)` — `{ name, cnes, kind, health_unit_id, active }`; `own` exige `health_unit_id` de unidade ativa (`cnes` copiado da unidade quando ausente) e recusa `cnes` divergente; externo exige `cnes` válido; `kind` não muda no PATCH (`invalid_request` `path: "kind"`); `fail(:cnes_taken | :unit_taken | :invalid_request, details: { path: })`. Evento `care_provider.changed` (`care_provider_id`, `user_id`).
  - `SaveOffer.call(attrs:, by:, offer: nil, today: Time.zone.today) -> Result.ok(offer:)` — `{ provider_id, target_type, target_code, month, quantity }`; alvo por `Targets.resolve`; `month` `"YYYY-MM"` igual ou depois do mês corrente (`invalid_month`); no PATCH só `quantity` (os outros → `invalid_request`); trava a oferta (`FOR UPDATE`), conta o usado e recusa `quantity < used` com `fail(:quantity_below_used)`; repetida → `fail(:duplicate_offer)`. Evento `care_offer.changed` (`care_offer_id`, `user_id`).
  - `CareRegulation::Json.provider(p)` → `{ id, name, cnes, kind, health_unit: { id, name }|null, active }`; `Json.offer(o, used:)` → forma `<offer>` do contrato §2.
- Rotas: `get/post "admin/providers"`, `patch "admin/providers/:id"`, `get/post "admin/offers"`, `patch "admin/offers/:id"` (`GET admin/offers?month=&target_code=&provider_id=`; `month` padrão = corrente). Leitura `municipal_admin` ou `care_regulator`; escrita `municipal_admin` com step-up.

- [ ] **Step 1: Teste** (`admin_spec.rb`, `type: :request`): admin com step-up cria prestador `own` da UBS (201, `cnes` da unidade) e `contracted` com CNES; mesmo CNES → 409 `cnes_taken`; mesma unidade → 409 `unit_taken`; `own` sem unidade → 422 `invalid_request` `path: "health_unit_id"`; oferta do mês corrente (201, `used: 0`, `available: 5`, `month: "2026-10"` sob `travel_to`); oferta de mês passado → 422 `invalid_month`; repetida → 409 `duplicate_offer`; com 2 autorizações não liberadas (gravadas direto), `PATCH quantity: 1` → 422 `quantity_below_used` e `quantity: 2` → 200 com `available: 0`; uma autorização liberada não conta; regulador lê a lista com `used`/`available` e recebe 403 ao escrever; sem step-up → 401 `mfa_required`; eventos só com ids.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: manage regulation providers and monthly quantity offers`.

---

### Task 7: Fila do regulador — lista, detalhe e contadores

**Files:**
- Create: `app/services/care_regulation/{queue,cursor}.rb`, `app/controllers/care_regulation/requests_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/care_regulation/queue_spec.rb`, `spec/requests/care_regulation/queue_spec.rb`

**Interfaces:**
- Produces:
  - `CareRegulation::Queue.page(filters:, before:, limit:, today: Time.zone.today) -> { rows: [RegulationRequest], next_before: String | nil }` — `filters` = `{ status:, type:, target_code:, color:, overdue: }` (`status` padrão `pending`, em `QUEUE_STATUSES`; `color` filtra a cor efetiva; `overdue` `"true"`/`"false"`); inválido → `CareRegulation::Queue::InvalidFilter` com `field`. Ordem `rank ASC, requested_at ASC, id ASC` com `rank = CASE COALESCE(color, suggested_color) WHEN 'red' THEN 0 … END`; o cursor é a tupla.
  - `Queue.counts(today:) -> { pending:, overdue:, returned:, authorized:, external: }` (uma consulta agregada; `overdue` sobre `WAITING_STATUSES`).
  - `RegulationRequest.overdue_sql(today)` — a única definição em SQL de vencido (`due_on < ? AND status IN (WAITING_STATUSES)`); `#overdue?` é o espelho em Ruby.
  - `CareRegulation::Cursor.encode(tuple) -> String`, `Cursor.decode(raw) -> [Integer, Time, String]` (levanta `Cursor::Invalid`).
- Rotas: `get "requests"` → `{ requests: [<regulation_request> sem events], next_before }` (`care_regulator`; `Trail.listed!` com os ids, se houver); `get "requests/count"` (literal antes de `:id`; sem trilha); `get "requests/:id"` → `<regulation_request>` (`Trail.person_viewed!`). Escopo do regulador: a cidade toda (sem vínculo de unidade).

- [ ] **Step 1: Testes**
  - `queue_spec.rb` (serviço): com pedidos `pending` verde antigo, vermelho novo, amarelo com cor confirmada `red` e amarelo sugerido → ordem: os dois vermelhos (o mais antigo primeiro), depois amarelo, depois verde; `limit: 2` devolve `next_before` e a segunda página continua sem repetir; `overdue: "true"` só traz `due_on < hoje`; `status: "draft"` → `InvalidFilter` (`field: "status"`); `counts` conta `external` em `overdue` quando vencido.
  - `queue_spec.rb` (request): regulador sem vínculo nenhum lista (200) e vê pedido de qualquer unidade (escopo cidade toda); lista sem `events`; detalhe com `events`; `count` não publica trilha e a lista publica um `clinical_record.viewed` com `regulation_request_ids` e sem `patient_id`; detalhe publica um com `patient_id` e `person_ref`; `before` adulterado → 422 `invalid_cursor`; `limit=101` → 422 `invalid_limit`; `health_professional` → 403 `missing_role`; id de rascunho → 404.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: serve the regulated queue ordered by risk colour and waiting time`.

---

### Task 8: Autorização com cota, cor, motivo de pular e rede própria

**Files:**
- Create: `app/commands/care_regulation/authorize.rb`, `app/services/care_regulation/own_network.rb`, `app/controllers/care_regulation/decisions_controller.rb` (ação `authorize`)
- Modify: `config/routes.rb`
- Test: `spec/commands/care_regulation/authorize_spec.rb`, `spec/commands/care_regulation/authorize_concurrency_spec.rb`, `spec/requests/care_regulation/decisions_spec.rb` (parte de autorizar)

**Interfaces:**
- Consumes: `Transition` (Task 5), `Quota` (Task 6), `Deadlines` (Task 3).
- Produces:
  - `CareRegulation::Authorize.call(request:, offer_id:, version:, by:, color: nil, color_reason: nil, skip_reason: nil, now: Time.current) -> Result.ok(request:, authorization:, appointment_request:)` | `fail(:invalid_request, details: { path: })` (cor fora da lista; motivos fora de `REASON`) | `fail(:invalid_offer)` | `fail(:stale_version)` | `fail(:invalid_transition)` (não `pending`, ou `state_regulated`) | `fail(:color_reason_required)` | `fail(:skip_reason_required)` | `fail(:quota_exhausted)` | `fail(:citizen_not_verified)`. Ordem: valida a entrada → `CareOffer.lock.find_by(id:)` → `request.lock!` → versão e situação → oferta elegível (Desvio 3) → cor resultante e `color_reason` (obrigatório quando a cor resultante difere da efetiva) → `skip_reason` (Desvio 4) → cota (`Quota.used([offer.id])` < `quantity`) → `RegulationAuthorization` (`month` = `offer.month`) → `OwnNetwork.open!` se `own` → `Transition` para `authorized` com `color`, `color_reason`, `due_on` (recalculado se a cor efetiva mudou), `appointment_request_id`.
  - `CareRegulation::OwnNetwork.open!(request:, offer:, by:) -> AppointmentRequest` (Desvio 6; levanta `CareRegulation::OwnNetwork::NotVerified` sem par validado, que o `Authorize` converte em `fail(:citizen_not_verified)` com rollback; publica `appointment_request.created` com `appointment_request_id`, `regulation_request_id`, `target_unit_id`, `kind`, `user_id`).
- Rota: `post "requests/:id/authorize"` → `<regulation_request>` (`care_regulator`; corpo `{ offer_id, color?, color_reason?, skip_reason?, version }`).

- [ ] **Step 1: Testes**
  - `authorize_spec.rb`: autoriza na oferta externa com saldo → `authorized`, `authorization.provider.kind == "contracted"`, `available` cai 1, evento `authorized`; mudar a cor de `yellow` para `red` sem motivo → `color_reason_required`, com motivo → `due_on` = `requested_at` + 7; existe `pending` mais antigo do mesmo alvo com cor igual → `skip_reason_required`, com motivo grava `skip_reason` (cifrado) na autorização; o mais antigo de cor menos grave (verde) não exige; `returned` e `external` mais antigos não exigem; oferta de outro alvo ou de prestador inativo → `invalid_offer`; `version` velha → `stale_version`; pedido `returned` → `invalid_transition`; oferta própria cria `AppointmentRequest` `regulated` com `target_unit` da unidade do prestador, `appointment_type_key: "consulta_medica"`, `priority: "priority"` para vermelho, `origin_unit` = unidade solicitante, e o pedido de regulação aponta para ele.
  - **Review Focus 1:** `travel_to Time.utc(2026, 11, 1, 2, 30)` (31/10 23:30 em São Paulo): oferta de outubro autoriza e a de novembro → `invalid_offer`.
  - **Review Focus 4:** paciente cujo único par teve a validação revogada (pelo caminho real, `Citizens::RevokeVerification`) e oferta própria → `citizen_not_verified`; `RegulationAuthorization.count` e `AppointmentRequest.count` inalterados; pedido segue `pending` com a mesma `version`.
  - `authorize_concurrency_spec.rb` (threads reais, `self.use_transactional_tests = false`, o padrão de `spec/models/health_unit_lock_spec.rb`: `Queue` com `pop(timeout: 5)`, `after` que solta, faz `join(5) || kill` e apaga o que commitou): oferta com `quantity: 1` e dois pedidos `pending` do mesmo alvo (o segundo com `skip_reason` para não esbarrar no Desvio 4); a thread 1 trava a oferta (`CareOffer.lock.find`) e espera; a thread 2 chama `Authorize` e fica bloqueada (`join(0.5)` nil); solta a 1, que autoriza; a 2 termina com `quota_exhausted`; ao final, `Quota.used([offer.id])` = 1 e exatamente um pedido `authorized`.
  - `decisions_spec.rb` (autorizar): 200 com a forma do contrato; 409 `quota_exhausted`; 422 `skip_reason_required`; o log não contém `skip_reason` nem `color_reason`; profissional → 403.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — os três specs → PASS.
- [ ] **Step 5: Commit** — `feat: authorize regulation requests against offer quotas and open regulated appointment requests`.

---

### Task 9: Devolver, negar, cancelar e complementar

**Files:**
- Create: `app/commands/care_regulation/{decide,complement}.rb`, `app/controllers/care_regulation/returned_controller.rb`
- Modify: `app/controllers/care_regulation/decisions_controller.rb` (ações `return`, `deny`, `cancel`, `complement`), `config/routes.rb`
- Test: `spec/commands/care_regulation/decide_spec.rb`, `spec/requests/care_regulation/{decisions,returned}_spec.rb`

**Interfaces:**
- Produces:
  - `CareRegulation::Decide.call(request:, decision:, reason:, version:, by:) -> Result.ok(request:)` — `decision` ∈ `return`/`deny`/`cancel`; `reason` em `REASON` (senão `invalid_request` `path: "reason"`); de onde vale: `return` ← `pending`; `deny` ← `pending`, `returned`; `cancel` ← `pending`, `returned`, `external`, `authorized`, `scheduled` (senão `invalid_transition`). `cancel` com autorização vigente: ordem oferta (`FOR UPDATE`, lida antes pelo id da autorização vigente) → pedido (relê a autorização: mudou → `stale_version`) → pedido de agendamento (`lock!`; `scheduled` → `fail(:appointment_scheduled)`; `open` → `AppointmentRequests::Lifecycle.close!(…, reason: "dismissed", by:, dismiss_reason: SYNC_NOTES[:cancelled])`) → `released_at`/`release_reason: "cancelled"` → `Transition`.
  - `CareRegulation::Complement.call(request:, attrs:, version:, by:) -> Result.ok(request:)` — `{ justification, cid10?, suggested_color?, version }`; só de `returned` (`invalid_transition`); `by` com `health_professional`, vínculo ativo (`ProfessionalLink.active`) na `requester_unit` e `Requesters.check(cbo: <cbo do vínculo>, target:)` (`fail(:cbo_not_allowed)`); `due_on` recalculado só se a cor efetiva mudou (Desvio 5, da `requested_at` original); `Transition` para `pending` com `event: "complemented"` e `reason` = a justificativa nova.
  - `GET /regulation/returned?unit_id=` → `{ requests: [<regulation_request> sem events] }` — profissional com vínculo ativo na unidade e CBO `225*`/`2235*`/`2232*` (senão 403 `missing_role`); `unit_id` ausente ou não uuid → 422 `invalid_request`; trilha de lista.
- Rotas: `post "requests/:id/return"`, `post "requests/:id/deny"`, `post "requests/:id/cancel"` (`care_regulator`; `{ reason, version }`), `post "requests/:id/complement"` (`health_professional`), `get "returned"`.

- [ ] **Step 1: Testes**
  - `decide_spec.rb`: devolver `pending` → `returned` com o motivo no evento (cifrado: a coluna crua não contém o texto); negar `returned` → `denied`; negar `authorized` → `invalid_transition`; cancelar `authorized` de oferta externa libera a cota (`Quota.used` volta) com `release_reason: "cancelled"`; cancelar `authorized` própria com o pedido de agendamento `open` encerra-o como `dismissed` com a nota `SYNC_NOTES[:cancelled]`; com horário marcado (`Appointments::Schedule`) → `appointment_scheduled` e nada muda; motivo de 9 caracteres → `invalid_request`.
  - `decisions_spec.rb` (resto): os três POST (200) e as recusas (409 `stale_version`, 409 `invalid_transition`, 409 `appointment_scheduled`); o log não contém o motivo; eventos com `regulation_request_id` e `user_id` só.
  - `returned_spec.rb`: a enfermeira da unidade solicitante vê o devolvido (200) e complementa (`POST complement`, 200 → `pending`, `requested_at` igual ao original, evento `pending` com a justificativa nova); profissional de outra unidade → 403 em `returned` e em `complement`; enfermeira da unidade com alvo fora da lista → 403 `cbo_not_allowed`; `complement` de `pending` → 409 `invalid_transition`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: return, deny, cancel and complement regulation requests`.

---

### Task 10: Rede própria acompanhada pelos eventos do 17; falta e segunda falta

**Files:**
- Create: `app/services/care_regulation/{appointment_sync,misses}.rb`, `app/jobs/care_regulation/appointment_sync_job.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/services/care_regulation/appointment_sync_spec.rb`, `spec/jobs/care_regulation/appointment_sync_job_spec.rb`, `spec/commands/care_regulation/misses_spec.rb`

**Interfaces:**
- Consumes: `Transition`, `Quota`.
- Produces:
  - `CareRegulation::Misses.apply!(request:, by:) -> RegulationRequest` (Desvio 8; o chamador já travou o pedido; `Transition` para `missed` e logo para `closed` ou `pending`/`external`, com eventos `missed` e `closed`/`pending`/`external`).
  - `CareRegulation::AppointmentSync.call(appointment_request_id:) -> :ignored | :unchanged | String (situação nova)` (Desvio 7). Acha o pedido de regulação por `AppointmentRequest#origin_regulation_request_id`; ignora `kind != "regulated"`; `:ignored` quando o pedido do evento não está na cadeia do pedido de regulação (cadeia = `request.appointment_request_id` e os pedidos ligados a ele por `moved_from_request_id`, para trás e para a frente; o `Drain` copia `created_at`, então nunca ordene por data) ou o pedido de regulação não está em `authorized`/`scheduled`; na transação: lê a autorização vigente → (se o caminho for liberar) `CareOffer.lock` → `request.lock!` → segue a cadeia para a frente enquanto o atual estiver encerrado como `moved` (atualizando `appointment_request_id`) → decide pelo estado dele (`:unchanged` quando nada muda); o pedido de agendamento reaberto pela falta é travado e encerrado depois do pedido de regulação. Local da rede própria = `[unit.name, Scheduling::UnitAddress.call(unit)].compact.join(" — ")` cortado em `PLACE_MAX` só se o endereço passar (o nome nunca é cortado).
  - `CareRegulation::AppointmentSyncJob < ApplicationJob` (`include IdempotentConsumer`, `queue_as :default`) com `handle(**payload)`: resolve o id do pedido de agendamento por evento — `appointment.scheduled`/`.checked_in`/`.no_show`/`.expired` (`appointment_request_id`), `appointment.booked` (`request_id`), `appointment.moved` (`to_appointment_id` → `Appointment#request_id`), `appointment.cancelled` (`appointment_id` → `Appointment#request_id`), `appointment_request.closed` (`appointment_request_id`), `appointment_request.moved` (`to_request_id`) — e chama `AppointmentSync`. Nunca atribui `Current.city`.
  - Bindings: os 9 eventos acima passam de `[]` para `[CareRegulation::AppointmentSyncJob]` (sai `appointment.booked` do bloco "scheduling event bindings (ADR 0029)" do spec, que continua para os demais).

- [ ] **Step 1: Testes** (`appointment_sync_spec.rb`; pedido autorizado em oferta própria pela Task 8, horário pelo caminho real do 17 — `Appointments::Schedule` em dia sem turno; check-in por `Attendances::CheckIn`; falta por `Appointments::Lapse`):

```ruby
require "rails_helper"

RSpec.describe CareRegulation::AppointmentSync do
  before { regulation_city! }
  let(:setup) { own_authorized_request! } # helper da Task 2: pedido de regulação autorizado em oferta própria + pedido de agendamento
  let(:request) { setup.fetch(:regulation_request) }
  let(:appointment_request) { setup.fetch(:appointment_request) }

  def sync! = described_class.call(appointment_request_id: request.reload.appointment_request_id)

  it "horário marcado → scheduled com data e local; atendimento → completed; redespacho não regride" do
    appointment = schedule_regulated!(appointment_request, at: 3.days.from_now.change(hour: 9))
    expect(sync!).to eq("scheduled")
    expect(request.reload).to have_attributes(status: "scheduled", scheduled_at: appointment.scheduled_at)
    expect(request.schedule_place).to include(appointment_request.target_unit.name)
    check_in_regulated!(appointment)
    expect(sync!).to eq("completed")
    expect(sync!).to eq(:unchanged)
    expect(request.reload.status).to eq("completed")
  end

  it "Review Focus 2: falta volta à fila uma vez; o no_show redespachado depois da reautorização não conta de novo" do
    appointment = schedule_regulated!(appointment_request, at: 2.days.from_now.change(hour: 9))
    sync!
    no_show!(appointment)
    expect(sync!).to eq("pending")
    expect(request.reload).to have_attributes(misses: 1, appointment_request_id: nil, scheduled_at: nil)
    expect(appointment_request.reload).to have_attributes(status: "closed", closed_reason: "dismissed")
    expect(CareRegulation::Quota.used([ setup.fetch(:offer).id ]).values.sum).to eq(1)
    reauthorize!(request.reload, offer: setup.fetch(:offer))
    expect(described_class.call(appointment_request_id: appointment_request.id)).to eq(:ignored)
    expect(request.reload).to have_attributes(status: "authorized", misses: 1)
  end

  it "segunda falta encerra (closed, outcome missed)" do
    2.times do |i|
      ar = AppointmentRequest.find(request.reload.appointment_request_id)
      no_show!(schedule_regulated!(ar, at: (2 + i).days.from_now.change(hour: 9)))
      described_class.call(appointment_request_id: ar.id)
      reauthorize!(request.reload, offer: setup.fetch(:offer)) if i.zero?
    end
    expect(request.reload.status).to eq("closed")
    expect(CareRegulation::Json.request(request)[:outcome]).to eq("missed")
  end

  it "horário expirado (sem confirmação) volta a authorized sem contar falta" do
    appointment = schedule_regulated!(appointment_request, at: 5.days.from_now.change(hour: 9))
    sync!
    expire!(appointment)
    expect(sync!).to eq("authorized")
    expect(request.reload).to have_attributes(misses: 0, scheduled_at: nil)
  end

  it "cancelado pelo cidadão → cancelled com a cota liberada e sem autor" do
    appointment = schedule_regulated!(appointment_request, at: 5.days.from_now.change(hour: 9))
    Appointments::CancelByCitizen.call(appointment: appointment, reason: "Não consigo ir nesse dia")
    expect(sync!).to eq("cancelled")
    expect(request.reload.events.last.by_user_id).to be_nil
    expect(CareRegulation::Quota.used([ setup.fetch(:offer).id ]).values.sum).to eq(0)
  end

  it "Review Focus 3: unidade esvaziada pelo Drain — o pedido de regulação segue o pedido novo" do
    appointment = schedule_regulated!(appointment_request, at: 5.days.from_now.change(hour: 9))
    sync!
    target = HealthUnit.create!(name: "UBS Recebe #{SecureRandom.hex(2)}", kind: "ubs")
    drain!(appointment_request.target_unit, to: target) # caminho real: HealthUnits::Drain
    fresh = AppointmentRequest.find_by!(moved_from_request_id: appointment_request.id)
    expect(fresh.origin_regulation_request_id).to eq(request.id)
    described_class.call(appointment_request_id: fresh.id)
    expect(request.reload).to have_attributes(status: "scheduled", appointment_request_id: fresh.id)
  end
end
```

  Os auxiliares `own_authorized_request!`, `schedule_regulated!(ar, at:)`, `check_in_regulated!(appointment)`, `no_show!(appointment)`, `expire!(appointment)`, `reauthorize!(request, offer:)` e `drain!(unit, to:)` entram em `spec/support/regulation_helpers.rb` chamando os comandos reais (`Authorize`, `Appointments::Schedule`, `Appointments::Confirm`, `Attendances::CheckIn`, `Appointments::Lapse` com `travel_to`, `HealthUnits::Drain`). `appointment_sync_job_spec.rb`: o mesmo evento entregue duas vezes processa uma vez (`ProcessedEvent`); evento de pedido não `regulated` não toca nada; payload de cada um dos 9 eventos resolve o pedido certo. `misses_spec.rb`: externo `state_regulated` com a primeira falta volta a `external`; `misses` nunca passa de 2 (CHECK).
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — os três specs + `spec/initializers/domain_events_bindings_spec.rb spec/commands/appointments spec/commands/attendances spec/commands/health_units` → PASS.
- [ ] **Step 5: Commit** — `feat: follow regulated appointment requests through scheduling events and handle missed visits`.

---

### Task 11: Prestador externo e convivência com o sistema do estado

**Files:**
- Create: `app/commands/care_regulation/{mark_external,schedule_external,record_outcome}.rb`, `app/services/care_regulation/state_csv.rb`, `app/controllers/care_regulation/exports_controller.rb`
- Modify: `app/controllers/care_regulation/decisions_controller.rb` (ações `external`, `schedule`, `outcome`), `config/routes.rb`
- Test: `spec/requests/care_regulation/{external,exports}_spec.rb`, `spec/services/care_regulation/state_csv_spec.rb`

**Interfaces:**
- Produces:
  - `MarkExternal.call(request:, system:, number:, version:, by:) -> Result` — de `pending`, `returned`, `external` (senão `invalid_transition`); `system` em `EXTERNAL_SYSTEMS`, `number` em `EXTERNAL_NUMBER` (senão `invalid_request` `path:`); grava `state_regulated: true`, sistema e número; `Transition` para `external` (evento `external`; de `external` para `external` só muda o número e a versão).
  - `ScheduleExternal.call(request:, at:, place:, notes:, version:, by:, now: Time.current) -> Result` — vale em `external` ou em `authorized`/`scheduled` com autorização vigente de prestador não `own` (Desvio 11; rede própria → `invalid_transition`); `at` ISO 8601 com fuso, entre `now - 30.days` e `now + 365.days` (senão `invalid_request` `path: "at"`); `place` 1..200; `notes` ≤ 500; `Transition` para `scheduled`.
  - `RecordOutcome.call(request:, result:, version:, by:) -> Result` — `scheduled` de prestador não `own` ou `state_regulated`; `completed` → `Transition`; `missed` → `Misses.apply!`.
  - `CareRegulation::StateCsv.build(status:) -> { csv: String, ids: [uuid] }` — UTF-8 com BOM, `;`, `\r\n`; `HEADER = %w[pedido_id data_pedido situacao tipo codigo_alvo alvo cor prazo cid10 justificativa paciente cns_final nascimento sexo unidade_solicitante unidade_cnes solicitante solicitante_cbo sistema_externo numero_externo]`; datas `dd/mm/aaaa`; CNS só os 4 últimos dígitos; sem CPF; toda célula por `Exams::Report.safe_cell`; `status` padrão `external`, em `QUEUE_STATUSES` (senão `invalid_request`); ordem `requested_at, id`.
- Rotas: `post "requests/:id/external"`, `post "requests/:id/schedule"`, `post "requests/:id/outcome"` (`care_regulator`); `get "exports/state.csv"` (`care_regulator`; `query_only!(:status)`; `Cache-Control: no-store`; `filename: "regulacao-estado-AAAAMMDD.csv"`; trilha de lista com os ids).

- [ ] **Step 1: Testes**
  - `external_spec.rb`: marcar `pending` como SISREG `"123456789"` → `external`, `state_regulated: true`, `external: { system: "sisreg", number: "123456789" }`, sai de `GET requests?status=pending` e não consome cota; número `"12 34"` → 422 `invalid_request` `path: "number"`; marcar `authorized` → 409 `invalid_transition`; agendar `external` (`at`, `place: "Hospital Regional — ambulatório 3"`) → `scheduled`; desfecho `missed` → volta a `external` com `misses: 1`; agendar e desfecho `completed` → `completed`; agendar pedido autorizado em oferta própria → 409 `invalid_transition`; autorizado em oferta contratada → agenda (200) e remarcar (200) mantém a mesma autorização (mesmo `month`).
  - `state_csv_spec.rb`: cabeçalho fixo; CNS `…1234` vira `1234`; nenhuma célula com o CPF do paciente; célula que começa com `=` sai protegida; BOM no início.
  - `exports_spec.rb`: 200 `text/csv`, `no-store`, nome sem dado pessoal, um `clinical_record.viewed` com os ids; `status=draft` → 422; outra chave de query → 422 `invalid_request`; profissional → 403.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: schedule external providers, record outcomes and export state-regulated requests`.

---

### Task 12: Guia de encaminhamento em PDF

**Files:**
- Create: `app/services/care_regulation/guide_pdf.rb`, `app/controllers/care_regulation/guides_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/care_regulation/guide_pdf_spec.rb`, `spec/requests/care_regulation/guides_spec.rb`

**Interfaces:**
- Produces: `CareRegulation::GuidePdf.call(request) -> String` (bytes) — folha, fonte e rodapé de `Consultations::Print.document(Footer.new(FOOTER_TEXT), title: "Guia de encaminhamento")`, texto por `Consultations::Print.safe`; `FOOTER_TEXT = "Gerado pelo Rota Saúde — guia de regulação municipal; não é autorização do sistema do estado"`. Blocos: cabeçalho (cidade e órgão de saúde, número curto = 8 primeiros do id, data do pedido `dd/mm/aaaa`); paciente (nome, nascimento, sexo, CNS completo quando houver — **nunca CPF**); pedido (tipo, código e nome do alvo, CID-10 com descrição, justificativa com quebra de linha, cor e prazo `dd/mm/aaaa`, "regulado pelo estado" e número externo quando houver); solicitante (nome profissional, CBO com título, unidade com CNES); autorização (prestador, CNES, mês, data e local quando agendado) ou "Aguardando regulação". Rascunho também imprime, com a marca "RASCUNHO — sem validade". Número de páginas em toda página.
- Rota: `get "requests/:id/guide.pdf", format: false` → quem lê: a `requester_user`, `health_professional` com vínculo ativo na unidade solicitante, ou `care_regulator` (senão 404, sem revelar existência); `Trail.person_viewed!`; `Cache-Control: no-store`; `filename: "guia-regulacao-#{id[0, 8]}.pdf"`; `query_only!`.

- [ ] **Step 1: Testes** — `guide_pdf_spec.rb` (texto extraído com `PDF::Reader`, já usado em `spec/requests/signature_reading_spec.rb`): contém o nome do alvo, o CID e a justificativa longa inteira em mais de uma linha; não contém o CPF; pedido autorizado traz o prestador e o mês `10/2026`; rascunho traz a marca. `guides_spec.rb`: solicitante 200 `application/pdf` com trilha por pessoa; colega da unidade 200; profissional de outra unidade 404; regulador 200; nome do arquivo sem nome do paciente; query extra → 422.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: print the regulation referral guide`.

---

### Task 13: Situação do pedido para o cidadão

**Files:**
- Create: `app/services/care_regulation/citizen_view.rb`, `app/controllers/citizen_api/regulation_requests_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/citizen_api/regulation_requests_spec.rb`

**Interfaces:**
- Produces: `CareRegulation::CitizenView.call(patient_id:) -> [Hash]` — pedidos do paciente fora de `draft`, por `updated_at DESC, id`; cada um `{ id, label: target_name, status: CITIZEN_STATUS[status], scheduled_at: (status == "scheduled" ? iso8601 : nil), place: (scheduled ? schedule_place : nil), updated_at }` — nunca cor, posição, CID, justificativa, motivo nem prestador fora do local.
- Rota (escopo `/citizen`): `get "regulation_requests"` → `{ requests: [...] }` — par do celular da sessão por `citizen_id` (senão 404 `not_found`); par não validado → 403 `not_verified`; `care_regulation` utilizável (senão 403 `feature_disabled`); trilha `Timeline::Trail.viewed!(person:, access: "citizen")` quando há paciente.

- [ ] **Step 1: Teste** — `regulation_requests_spec.rb` (`type: :request`, `sign_in_citizen`): pedidos `pending`, `authorized`, `scheduled` (com data e local), `returned`, `denied`, `cancelled`, `closed`, `completed` e um `draft` → o rascunho não aparece; `authorized` e `pending` saem `waiting`; `cancelled` e `closed` saem `closed`; as chaves de cada item são exatamente `id label status scheduled_at place updated_at`; o corpo não contém `I10`, a justificativa nem `"red"`; `citizen_id` de outro celular → 404; par declarado → 403 `not_verified`; interruptor desligado → 403 `feature_disabled`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: show citizens the status of their regulation requests`.

---

### Task 14: Exclusão LGPD — pedido de regulação retido

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb`
- Test: `spec/commands/citizens/regulation_erasure_spec.rb`

**Interfaces:**
- Produces: `Citizens::RequestErasure.regulated?(ids) -> bool` (`RegulationRequest.where(patient_id: ids).exists?`, rascunho inclusive), chamado no fim de `examined_patients?` (comentário "ADR 0044: o pedido de regulação é registro clínico retido, em qualquer situação").

- [ ] **Step 1: Teste** — `regulation_erasure_spec.rb`: paciente cujo único registro clínico é um pedido de regulação (gravado direto, sem atendimento no par — o caminho real sempre tem atendimento, então o teste isola a regra) → `retained?` verdadeiro e o pedido nasce `retained`; `Citizens::Erase` reconfere e não apaga; sem pedido → `pending` como antes.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — o spec + `spec/commands/citizens` → PASS.
- [ ] **Step 5: Commit** — `feat: retain citizens with regulation requests on erasure`.

---

### Task 15: Semente de dev realista (Curitiba) para a prova no navegador

**Files:**
- Create: `lib/regulation_crew.rb`, `spec/lib/regulation_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- Produces: `RegulationCrew.seed_current_city(slug:, ddd:, password:) -> { switch:, users: [ { email, role, otpauth_uri } ], providers:, offers:, requests: { status => count } }` — só Curitiba (outras cidades: `{ switch: "fora" }`); liga `care_regulation` pelo mantenedor de dev; cria a reguladora (Valores fixados 4) e o OTP fixo; a unidade própria e os três prestadores; ofertas do mês corrente: cardiologia `225120` própria 10, oftalmologia `225265` contratada 8, ortopedia `225270` consórcio 1, endoscopia `0209010037` PPI 5; lista do enfermeiro `225250` e `225124`; e os pedidos, **todos pelos comandos do app** (`Consultations::Start`/`SaveDraft`, `CareRegulation::SaveDraft`, `Consultations::Finalize`, `Authorize`, `Decide`, `MarkExternal`, `ScheduleExternal`, `RecordOutcome`) com a médica e as pessoas do `CareLinesCrew`/`TerritoryRegistryCrew`, datas passadas por `travel_to` em volta do comando:

  | pedido | alvo | cor | situação | para mostrar |
  |---|---|---|---|---|
  | 1 | cardiologia | vermelho | `pending`, pedido há 10 dias | vencido no topo |
  | 2 | cardiologia | amarelo | `pending`, há 40 dias | vencido; exige motivo de pular se autorizar o 4 antes |
  | 3 | oftalmologia | verde | `pending`, há 5 dias | fila normal |
  | 4 | cardiologia | amarelo | `pending`, há 3 dias | motivo de pular |
  | 5 | ortopedia | amarelo | `authorized` no consórcio (cota esgotada) | 409 `quota_exhausted` no próximo |
  | 6 | cardiologia | verde | `authorized` na própria, pedido de agendamento aberto | recepção marca no 17 |
  | 7 | oftalmologia | amarelo | `scheduled` na contratada, daqui a 12 dias | agendamento externo |
  | 8 | endoscopia | verde | `returned` | complemento pela unidade |
  | 9 | ortopedia | azul | `external` (SISREG `202610280001`) | convivência e CSV |
  | 10 | oftalmologia | verde | `completed` | desfecho |

  Idempotente (pedido existente da consulta da semente é retomado, nunca duplicado). Só e-mail, papel e otpauth no `puts`, nunca dado de paciente.

- [ ] **Step 1: Teste** — `spec/lib/regulation_crew_spec.rb`: numa cidade de teste com o slug `curitiba` simulado como os specs dos crews anteriores, roda duas vezes e na segunda não cria nada novo; contagens por situação conferem a tabela; `Quota.used` da ortopedia = 1 = `quantity`; a saída do `puts` não contém CPF nem nome de paciente.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar** e ligar em `db/seeds.rb` depois do `SurveillanceCrew` (bloco "Regulação municipal (módulo 26a, ADR 0044)", no padrão dos anteriores). **Não rodar `db:seed` no dev** (só na prova, com autorização). **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: seed a realistic care regulation scenario for Curitiba`.

---

### Task 16: Revisão, runbook e suíte completa

**Files:**
- Create (worktree `docs-mod26a`, não publicado): `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/docs-mod26a/operacao/rollout-regulacao.md`

- [ ] **Step 1: Revisão do branch** contra a spec e o contrato (superpowers:requesting-code-review, um revisor no modelo mais capaz): Review Focus 1–5; `/opt/homebrew/bin/git -C apps/api/.claude/mod26a grep -n "DomainEvents.publish(" -- app/commands/care_regulation app/services/care_regulation app/controllers/care_regulation app/jobs/care_regulation app/commands/consultations app/commands/attendances` conferido chamada inteira (payload só com ids; nunca alvo, cor, CID); `grep -n "Platform.audit(" ` sem chamada nova; nenhum `Current.city =`; nenhum `justification`/`cid10`/`target_code` em `Rails.logger`; ordem de travas oferta → pedido → pedido de agendamento em `Authorize`, `Decide` e `AppointmentSync`; `R18`/`domain_events_bindings`/`city_encrypted_attributes_guard`/`adr_pointers` verdes.

- [ ] **Step 2: Runbook** `operacao/rollout-regulacao.md` (criar o worktree com `/opt/homebrew/bin/git -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/docs worktree add ../.worktrees/docs-mod26a origin/main` se não existir; no padrão dos `rollout-*.md`): pré-requisitos (`clinical_record` ligado; unidades e prestadores com CNES; quem terá `care_regulator`, concedido por admin com step-up; SIGTAP da competência importada — sem ela, procedimento é recusado); ordem (imagem do api → `bin/migrate` — **migração de cidade irreversível**, mexe em `appointment_requests` e no CHECK de papéis; dashboard e wpda; maintenance; cadastrar prestadores, ofertas do mês, prazos e lista do enfermeiro **antes** de ligar; ligar `care_regulation` por cidade pelo maintenance); conferências pós-deploy (`SELECT color, days FROM regulation_risk_deadlines`; `SELECT tgname FROM pg_trigger WHERE tgname LIKE '%regulation%' OR tgname LIKE '%care_offer%' OR tgname LIKE '%care_provider%'`; `ck_appointment_requests_regulated` presente; um pedido de teste em staging da consulta à autorização própria e ao check-in); operação (ofertas são mensais: cadastrar a do mês seguinte antes da virada; cota esgotada trava a autorização daquele alvo; falta não devolve cota; cancelar com horário marcado na rede própria exige que a unidade/cidadão cancele o horário primeiro); o que **não** faz (não escreve no SISREG, e-SUS Regulação, CARE ou CROSS: o CSV é para digitação; sem APAC/OCI/teto em dinheiro — 26b; sem MIRA — 26c); LGPD (pedido retém o cadastro); reversão (desligar o interruptor: a fila some, o "encaminhado" volta à regra de hoje, os pedidos e a sincronização com o 17 continuam; nada se apaga); gates de go-live (catálogo de alvos e lista do enfermeiro conferidos pela regulação da cidade piloto; prazos por cor aprovados; guia impressa conferida pela recepção do prestador).

- [ ] **Step 3: Escopo fechado verde** — `bin/test api -w mod26a spec/services/care_regulation spec/commands/care_regulation spec/requests/care_regulation spec/requests/citizen_api/regulation_requests_spec.rb spec/jobs/care_regulation spec/models/care_regulation_schema_spec.rb spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/commands/citizens spec/commands/consultations spec/commands/attendances spec/commands/appointments spec/commands/health_units spec/commands/consultation_drafts spec/requests/consultations_spec.rb spec/jobs/signatures spec/jobs/surveillance spec/lib/regulation_crew_spec.rb spec/architecture spec/initializers spec/events spec/adr_pointers_spec.rb` → PASS.

- [ ] **Step 4: Pedir a suíte completa** à sessão "Testes" (mudança transversal: migração de cidade, papel novo no CHECK, `appointment_requests` e o `Drain`, `spec/support`, consumidor novo de nove eventos do 17, `Finalize`/`Close`): SendMessage `api · .claude/mod26a/feat/mod-26a-care-regulation · --full · migração de cidade 20261028900001, papel care_regulator, kind regulated no 17 e módulo 26a`. Antes, `docker compose stop worker` (memória "Suíte com worker parado") e `start` depois. Falha → diagnóstico nesta sessão (a "Testes" só responde o resumo).

- [ ] **Step 5: Commit final** (se a revisão mudou algo) — `fix: address module 26a review findings`; avisar a coordenadora: plano do api concluído, runbook no `docs-mod26a` (não publicado), divergências R1–R19 pendentes de decisão no contrato, cards F-26.x seguem In Progress até o dashboard, o wpda e o maintenance.

---

## Self-review

- **Cobertura da spec:** §2 decisão 1 (escopo: Goal, runbook); 2 pedido na consulta e guia (4, 5, 12); 3 quem pede (3, 4, 5, 9); 4 cor e prazos (3, 5, 8, 9); 5 fila e ações (7, 8, 9); 6 oferta própria e externa com cota (6, 8, 9); 7 agendamento, falta e segunda falta (8, 10, 11); 8 convivência (11); 9 cidadão (13); 10 tabela própria de prestadores (2, 6); 11 LGPD (14); 12 interruptor (1). Decisões do coordenador: cota da rede própria e pedido `regulated` com ordem de travas (6, 8, 10); encaminhamento simples preservado e pedido que vale na finalização/adendo (5); cota por mês, falta não devolve, negar só antes (3 Desvios, 8, 9, 10); ordem de exibição e motivo de pular (7, 8); escopo do regulador cidade toda (7); regulado pelo estado sem cota (5, 11); complemento mantendo a data (9); CBO do vínculo da consulta (3, 5). §5 segurança (Global Constraints, 4, 7, 11, 12). §6 testes: transições (5, 8–11), última vaga com threads (8), prazos no fuso (3, 8), quem pode pedir (3, 4, 5), falta e reentrada (10, 11), retenção (14), escopo do regulador (7). §7 runbook (16). Catálogo de alvos do contrato §12 (0). F-26.1: 1, 2, 3; F-26.2: 4, 5, 12; F-26.3: 7, 8, 9; F-26.4: 6; F-26.5: 10, 11; F-26.6: 11; F-26.7: 13.
- **Tipos:** `CareRegulation::Target`/`Targets.resolve(type:, code:, on:)` em 3–6, 9; `Requesters.check(cbo:, target:)` em 3, 4, 5, 9; `Transition.call(request:, to:, by:, reason:, version:, changes:, event:)` em 5, 8–11; `Quota.used(offer_ids)` em 6, 8–10, 15; `Deadlines.due_on(requested_at:, color:)` em 3, 5, 8, 9; `Misses.apply!(request:, by:)` em 10, 11; `Trail.person_viewed!/listed!` em 4, 7, 9, 11, 12; `Json.request`/`list_item`/`offer`/`provider` em 4, 6, 7.
- **Proporção:** corpos só onde a regra não se deduz (regra de instrumento SIGTAP, ordem de travas, reconciliação pelo estado); o resto por assinatura, valores e teste.

## Divergências propostas ao contrato

- **R1 — Autorizações em tabela própria (§2, spec §3, ADR "Consequências").** `regulation_authorizations` (uma linha por autorização, liberada uma vez) é o que permite "falta não devolve, cancelar devolve, reautorizar depois da falta consome de novo" com saldo derivado; `care_offers` não guarda `used` (o `<offer>` calcula). Acrescentar a tabela à lista do ADR.
- **R2 — Catálogo de alvos (§3).** Falta rota para a tela escolher o alvo: `GET /regulation/targets?type=specialty|procedure&q=&limit=` → `{ targets: [ { type, code, name, group, nurse_allowed } ] }` (`health_professional`, `care_regulator`, `municipal_admin`; procedimento exige `q` ≥ 3, senão 422 `invalid_query`; sem trilha). `GET /regulation/admin/nurse_items` devolve `{ items: [ { target_type, target_code, name } ] }`.
- **R3 — Recusas novas.** 422 `invalid_offer` (oferta de outro alvo, de outro mês ou de prestador inativo); 409 `citizen_not_verified` (oferta própria sem par validado do paciente); 409 `appointment_scheduled` (cancelar na rede própria com horário vivo); 422 `regulation_request_required` (finalizar/encerrar com `referred` sem unidade e sem pedido, interruptor ligado); 409 `invalid_transition`; 422 `invalid_month`; 409 `duplicate_offer`, `cnes_taken`, `unit_taken`, `duplicate_item`; 422 `invalid_query`, `invalid_limit`, `invalid_cursor`; `invalid_target` pode trazer `hint: "exam_order"`.
- **R4 — Cancelar com horário marcado (§4).** O 17 não tem cancelamento de horário pela equipe: na rede própria, `cancel` só com o pedido de agendamento `open` (o cidadão cancela pelo wpda ou a falta/expiração reabre).
- **R5 — Encerramentos do 17 fora da regulação (§7).** Pedido de agendamento `regulated` encerrado por `citizen_cancelled`, `dismissed` pela unidade ou `consent_revoked` → pedido de regulação `cancelled`, cota liberada (`release_reason` `citizen_cancelled`/`unit_dismissed`/`consent_revoked`), evento sem autor.
- **R6 — Horário expirado (§7).** `appointment.expired` (cidadão não confirmou) devolve o pedido a `authorized` sem data; não é falta.
- **R7 — `missed` só no andamento (§1, §7).** A situação atual nunca fica `missed`: o evento `missed` é seguido de `pending` (ou `external`, se regulado pelo estado) ou de `closed` (segunda falta). `outcome` = `"completed"` | `"missed"` (quando a segunda falta encerrou) | `null`.
- **R8 — Vencidos (§4).** A spec (§4) fala em "vencidos no topo"; o contrato ordena por cor e data. Fica a ordem do contrato (vermelho novo antes de azul vencido); `overdue` vai no item, no filtro e no contador.
- **R9 — Exames do 21a (§3).** Procedimento do grupo 02 ativo no catálogo de exames é recusado como regulação (`invalid_target`, `hint: "exam_order"`): ele segue pelo pedido de exame e pela fila do 21a.
- **R10 — Mês da oferta (§4, §6).** Só a oferta do mês corrente no fuso da cidade é autorizável; ofertas de meses futuros podem ser cadastradas; mês passado não se cadastra.
- **R11 — Externo agendado (§7, §8).** `schedule` leva `external` e `authorized` não própria a `scheduled` (com `state_regulated` mantido); remarcar mantém a autorização e o mês. `schedule.at` ISO 8601 com fuso, de 30 dias atrás a 365 à frente.
- **R12 — Eventos (§11).** Acrescentar `regulation_request.complemented`, `regulation_request.external` e `care_regulation.settings_changed`. Payloads: `regulation_request.*` com `regulation_request_id` e `user_id` (nulo quando o 17 encerrou); `care_offer.changed` (`care_offer_id`, `user_id`); `care_provider.changed` (`care_provider_id`, `user_id`). Trilha: `clinical_record.viewed` com `access: "regulation"` (por pessoa ou `regulation_request_ids`). O `appointment_request.created` do pedido regulado leva `regulation_request_id`.
- **R13 — Pedido de agendamento `regulated` (§4, §7).** `kind: "regulated"`, origem `origin_regulation_request_id`, tipo `consulta_odontologica` para especialidade `2232*` e `consulta_medica` para o resto, prioridade `priority` para vermelho; o `<regulation_request>.schedule` da rede própria vem do horário do 17 (`place` = unidade e endereço). O dashboard da recepção precisa do rótulo de `kind: "regulated"` e `origin: "regulation"`.
- **R14 — Rascunho (§3).** Máximo de 10 pedidos por consulta; rascunho só da autora, em consulta `draft` ou `finalized` (esta, para o adendo); consulta descartada apaga os rascunhos; `GET` da consulta devolve rascunhos e promovidos, sem trilha. Com o interruptor desligado na finalização, os rascunhos ficam como estão.
- **R15 — Leitura por papel (§4, §5, §3).** Detalhe e fila: só `care_regulator`. Guia: a solicitante, profissional com vínculo na unidade solicitante ou `care_regulator` (outros: 404). `GET /regulation/returned?unit_id=` exige vínculo ativo na unidade e CBO `225*`/`2235*`/`2232*`. Complementar exige ainda que o CBO do vínculo possa pedir o alvo.
- **R16 — Sincronização independe do interruptor.** O consumidor dos eventos do 17 reconcilia pedidos `regulated` mesmo com `care_regulation` desligado (consistência do que já existe).
- **R17 — Prestador próprio (§6).** `own` exige `health_unit_id` ativo e herda o CNES da unidade; `kind` não muda depois de criado; CNES e unidade únicos entre prestadores. `<provider>` = `{ id, name, cnes, kind, health_unit: { id, name }|null, active }`.
- **R18 — CSV do estado (§8).** Colunas fixas da Task 11 (UTF-8 com BOM, `;`), com `justificativa` (necessária para digitar no sistema do estado); `status` padrão `external`.
- **R19 — Prazos e motivos.** Prazo por cor 1..365 dias; mudar a tabela não mexe em pedido já feito; `due_on` recalculado só quando a cor efetiva muda e sempre a partir da data do pedido original. `reason`, `color_reason`, `skip_reason` 10..500; `justification` 10..2000; `notes` ≤ 500; `place` 1..200.
