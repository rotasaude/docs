# Módulo 33 — Ouvidoria do SUS municipal (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 32b em `origin/main` do api** (conferido ao escrever este plano em `3e96796d`): última migração de cidade `20261110900003_add_urgency_stock_deduction.rb` (`db/city_schema.rb` em `define(version: 2026_11_10_900003)`); nenhuma migração de plataforma nova no 33. `Platform::Features::CATALOG` termina em `urgency_care`, `waiting_room_display`, `urgency_stock_deduction`; `CITIZEN_HIDDEN = %w[psychosocial_care]`; os portões de módulo respondem **403** `{ error: "feature_disabled", feature: KEY }` (`app/controllers/concerns/psychosocial_gate.rb:9`, `citizen_api/trips_controller.rb:28`). `Membership::ROLES` = `AddPsychosocialCare::ROLES` (24 papéis; `spec/models/membership_roles_spec.rb` fixa a cadeia de migrações que reescrevem `ck_memberships_role`; `spec/models/create_{analytics,campaigns}_migration_spec.rb` restauram a lista da última). `ExamFile::PURPOSES = %w[eqa_annual_report tfd_accountability]`, `ck_exam_files_purpose` reescrito por último em `20261103900001_add_sanitary_transport.rb`; `exam_files` só tem dono `uploaded_by_user_id` ou `exam_provider_id`, sem nome de arquivo; `rota_exam_file_guard()` só aceita `consumed_at` uma vez e apaga só o não consumido; `Exams::DailyJob` apaga todo `ExamFile` não consumido com mais de 24 h, sem portão (`app/jobs/exams/daily_job.rb:16`). `Exams::Files.{store,store_base64,claim!,MAGIC,MAX_BYTES,TTL}`; `Tfd::Files` (verifier e `TOKEN_PURPOSE` próprios, `url_for`/`resolve`, 410 `link_expired`). `CitizenAuthentication` (`allow_anonymous_citizen`, `require_json_for_writes` → 415 `json_required`, cookie só em `require_citizen_session`); `CitizenApi::BaseController::RateLimitStore`; `rate_limit … by: -> { current_citizen_session&.id || request.remote_ip }` (`citizen_api/appointments_controller.rb:11`). `CitizenSession#citizens` = `Citizen.where(phone:)`; `PurgeCitizenChannelJob` apaga sessões 30 dias depois do fim. `CitizenApi::NoticesController` com três fontes (campanha, lembrete, viagem), `cpf_masked` em todo aviso quando a sessão tem mais de um cidadão, `read` procurando nas três. `Transport::Notify` (`app/commands/transport/notify.rb`; ordem de trava: cidadãos `FOR KEY SHARE` antes do resto; `sms_allowed?` privado: `Campaigns::SmsSetting.enabled?`, `SmsGateway.configured?`, `Campaigns::SmsBatchJob::WINDOW_HOURS = (8...20)`, `CitizenContactPreference#sms_opt_in`/`notices_muted`). `Citizens::RequestErasure.retained?` (pedido nasce `retained` e o `Citizens::Erase` não apaga nada quando retém, `erase.rb:104`); `Citizens::Erase#erase_pair` apaga `TransportTripNotice` do par. `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`). `db/city_triggers.sql`: `rota_append_only()`, `rota_reencryption_only(cols…)`, `rota_no_delete()`, `rota_transport_trip_notice_guard()` (só `read_at`, uma vez), sequência criada dentro do bloco `to_regclass` (`lab_requisition_number_seq`, linha 1912) e `LabRequisition.next_number`. `Analytics::Suppression.{cell,rate,group,group_rate}` com `Admin::SmallCount::SUPPRESSED = { suppressed: true }` (`app/queries/admin/small_count.rb:5`). `ReportPeriod` (`from`/`to` no fuso da cidade, 422 `invalid_period`). `Surveillance::{AlertJob,MailJob,OverdueJob,Recipients}` (e-mail fora do consumidor, dedup por `ProcessedEvent` "<kind>:<ref>"). `EachCityJob`, `Psychosocial::CarePlanReviewJob` (7h). `Psychosocial::{Access,Gate}`, `Psychosocial::BaseController` (`render_failure`, `render_refusal`, `query_only!`, `body`, `body_param`, `wrap_parameters false`), cursor `CareRegulation::Cursor.{encode_page,decode_page}` com `next_before`. `RequestLogPathMask::FILE_TOKEN_PATH = %r{\A/(exams|tfd)/files/[^/?]+}i`. `filter_parameters`: `:body`, `:reason`, `:text`, `:justification`, `:cpf`, `:phone`, `/\Aq\z/`, `/\A(code|state)\z/`, `data`, `filename` já filtrados; `extract`, `reply`, `manifestant_*`, `contact`, `ouvidorsus_number`, `external_body_name` não. `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..50)`. Specs que fixam listas e mudam neste plano: `spec/services/platform/features_spec.rb:121,128`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/operators/city_record_settings_spec.rb:30`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/models/membership_roles_spec.rb` (último exemplo), `spec/models/create_analytics_migration_spec.rb` e `spec/models/create_campaigns_migration_spec.rb` (`restore_exam_roles`), `spec/lib/request_log_path_mask_spec.rb`.

**Goal:** O lado api do módulo 33 (F-33.1 a F-33.9, ADR 0051): interruptores `ombudsman` e `ombudsman_anonymous_web`, três papéis, setores da secretaria, respondentes, assuntos e prazos da cidade, manifestação pelo wpda (identificada ou com identidade preservada) com protocolo e anexos, denúncia anônima sem login com limites, registro pela equipe com comprovante, classificação, encaminhamento por extrato e ponto de resposta, a máquina de estados com prazos, complemento, prorrogação, respostas, arquivamento e reabertura, identidade preservada com trilha, avisos e SMS ao cidadão, pesquisa de satisfação, painel e relatório de gestão anual, job diário, exclusão LGPD e semente de dev.

**Architecture:** Duas migrações de cidade: `900001` é a fronteira transversal (CHECK de papéis, finalidade nova de `exam_files`, `departments`, genérica), `900002` é o domínio da ouvidoria. A manifestação (`ombudsman_manifestations`) guarda o estado e o que muda (classificação, ciclo, prazo); tudo o que é texto ao longo do tempo vai a `ombudsman_events` (só acréscimo, texto cifrado, `visible_to_citizen`). O prazo nunca é gravado: `Ombudsman::Deadlines.due_on_sql(settings)` é a fonte única (lista, filtro por cor, painel, job) e `Ombudsman::Deadlines.due_on(m, settings)` o espelho em Ruby, conferidos um contra o outro por spec. Toda ação passa por `Ombudsman::Lock.manifestation!(id)` (cidadão `FOR KEY SHARE` → manifestação `FOR UPDATE`, a ordem do `Transport::Notify` e do `Citizens::Erase`) e confere `expected_state`. O cidadão escreve pelo `CitizenApi::Ombudsman::*` (sessão do telefone); a denúncia anônima tem controlador próprio sem sessão, com limite de corpo, limite por IP, teto diário da cidade sob a trava da linha de `ombudsman_settings`. O anexo nasce com o upload (`ombudsman_attachments` com `ombudsman_manifestation_id` nulo, nome cifrado, dono = sessão ou usuário) e é ligado na criação; o órfão some em cascata com o `ExamFile` órfão do `Exams::DailyJob`. A identidade só sai por `POST …/identity` (ombudsman, justificativa, `ombudsman_identity_accesses`).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), Solid Queue, RSpec via `bin/test`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-11-module-33-ombudsman-design.md` (decisões 1–19) e `docs/adr/0051.md`. Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-11-module-33-ombudsman-contracts.md` (§1–§16); toda diferença está em "Divergências propostas ao contrato", no fim (A1..A31). Pesquisa: `docs/pesquisa/2026-11-11-ouvidoria-33.md`.

## Desvios e precisões

Cada item abaixo tem a divergência correspondente no fim; o plano segue a proposta até o coordenador decidir em §16 do contrato (quando a decisão for outra, a task dona muda só a constante ou a regra indicada).

1. **Portão.** 403 `{ error: "feature_disabled", feature: "ombudsman" | "ombudsman_anonymous_web" }`, como todos os portões do api (A1). O contrato diz 404 "como os outros gates"; os outros são 403.
2. **Datas.** Todo "hoje" é `Time.zone.today` dentro de `CityConnection.with` (fuso da cidade). `received_at` é `timestamptz`; `cycle_started_on = received_at.in_time_zone.to_date`; o ano do protocolo é `received_at.in_time_zone.year`.
3. **Ciclo e prazo.** `due_on = cycle_started_on + (urgent ? urgent_days : response_days) + (extended ? extension_days : 0) + suspended_days`. Em `awaiting_complement`: `color = "paused"` e o `due_on` mostrado soma também `today - complement_requested_at::date` (o recalculado na volta). Estados finais: `due_on` do ciclo, `color = nil`. `warning`: `due_on - today <= warning_days` (inclusive) → `yellow`; `today > due_on` → `red`. Encaminhamento: `due_on` gravado (`created_at` local + `forwarding_days`; `extend` soma `forwarding_extension_days`), cor só em `pending`. Os prazos são lidos das configurações vigentes: mudar `response_days` muda o prazo de quem está aberto (decisão 15 "calculados na leitura"; A19).
4. **Trava.** `Ombudsman::Lock.manifestation!(id) -> OmbudsmanManifestation | nil`: lê `citizen_id` sem trava, trava o cidadão `FOR KEY SHARE` (quando há), trava a manifestação `FOR UPDATE` e, se o `citizen_id` mudou no intervalo (exclusão), recomeça uma vez. `expected_state` ausente ou fora de `STATES` → 422 `invalid_request` `path: "expected_state"`; diferente do estado atual → 409 `manifestation_changed`, nada gravado.
5. **Eventos da linha do tempo** (`ombudsman_events.kind`, A6): `received classified forwarded forwarding_cancelled forwarding_replied forwarding_declined forwarding_extended complement_requested complement_received extended interim_response final_response archived reopen_requested reopen_rejected reopened resolution_changed identity_consent_recorded`. Visíveis ao cidadão (`Catalog::CITIZEN_EVENT_KINDS`): `received complement_requested complement_received extended interim_response final_response archived reopen_requested reopen_rejected reopened` — `final_response` no anônimo nasce `visible_to_citizen = false`. O `data` do evento leva só ids, rótulos de catálogo e estado anterior/novo.
6. **Anexo.** `ombudsman_attachments` nasce no upload: `exam_file_id` (único, FK `ON DELETE CASCADE`), `filename` cifrado (1–120, A4), dono `citizen_session_id` (uuid **sem FK**: a sessão é apagada pelo `PurgeCitizenChannelJob`) ou `uploaded_by_user_id`, `uploaded_by` (`citizen`|`staff`), `ombudsman_manifestation_id` e `event_id` nulos até a ligação. Ligar = travar as linhas do mesmo dono ainda sem manifestação + `ExamFile.where(id:).update_all(consumed_at:)` na mesma transação. Até 3 por manifestação (criação) e até 3 por complemento. `exam_files` não ganha coluna (A3).
7. **Exclusão LGPD.** A manifestação **não** retém o CPF (não entra em `RequestErasure.retained?`); quem é apagado pelo `Citizens::Erase` tem `full_name`, CPF e telefone copiados para os `manifestant_*` das manifestações dele, `citizen_id = NULL` e os `ombudsman_notices` apagados, na mesma transação, depois da trava dos pares (A7).
8. **Busca do cidadão pela equipe** (§5): `POST /ombudsman/person_lookup { cpf }` sobre `Citizen.not_erased.where(cpf:)` (pares declarados inclusive, "o par declarado basta"), na forma e com as recusas do `Transport::PersonLookupsController`; **não** reusa `Patients::LookupByIdentifier`, que cria paciente e exige par validado (A8).
9. **Opções da equipe.** `GET /ombudsman/options` → `{ subjects, health_units, professionals, departments }` só ativos, para o formulário de classificação e de encaminhamento (o `/professionals` é só do admin; A9).
10. **Respostas sem texto definido.** Limites dos textos que o contrato não fixa: `TEXT = (10..4000)` (pedido e registro de complemento, resposta intermediária e conclusiva, decisão de reabertura, motivo de recusa e de prorrogação do respondente, justificativa de `change_resolution` e de `extend`, texto do consentimento) (A10).
11. **Comprovante.** `GET /ombudsman/manifestations/:id/receipt` → 404 `not_found` na anônima (não 403: não revela que existe e é anônima).
12. **Recusas.** `{ error, path?, problems? }` (P23); conflitos 409; papel 403 `missing_role`; o resto 422; 429 com `{ error: "rate_limited" | "daily_cap_reached" }`; 413 `payload_too_large`.

## Valores fixados por este plano

1. **Interruptores** (depois de `urgency_stock_deduction` no `CATALOG`):
   - `Entry.new(key: "ombudsman", description: "Ouvidoria do SUS municipal", requires: [])`;
   - `Entry.new(key: "ombudsman_anonymous_web", description: "Denúncia anônima sem login no canal do cidadão", requires: %w[feature:ombudsman])`.
2. **`Ombudsman::Catalog`:** `KINDS = %w[complaint report praise suggestion request]`; `MODES = %w[identified preserved anonymous]`; `CHANNELS = %w[web in_person phone ouvidorsus]`; `STAFF_CHANNELS = %w[in_person phone ouvidorsus]`; `STATES = %w[received in_analysis forwarded awaiting_complement answered archived]`; `OPEN_STATES = STATES - FINAL_STATES`; `FINAL_STATES = %w[answered archived]`; `RESOLUTIONS = %w[resolved not_resolved]`; `RESOLUTION_KINDS = %w[complaint request report]`; `ARCHIVE_REASONS = %w[duplicate no_complement lack_of_clarity lack_of_civility improper loss_of_object forwarded_elsewhere]`; `FORWARDING_STATES = %w[pending replied declined cancelled]`; `EVENT_KINDS`/`CITIZEN_EVENT_KINDS` (Desvio 5); `NOTICE_KINDS = %w[interim_response complement_requested final_response archived reopen_decided]`; `MET = %w[yes partly no]`; `SCORE = (1..5)`; `BODY = (20..4000)`; `EXTRACT = (20..4000)`; `TEXT = (10..4000)`; `JUSTIFICATION = (10..500)`; `REOPEN_REASON = (10..2000)`; `NAME = (2..80)`; `FILENAME = (1..120)`; `OUVIDORSUS_NUMBER_MAX = 40`; `EXTERNAL_BODY_MAX = 120`; `REPORT_TEXT_MAX = 8000`; `MANIFESTANT_NAME = (2..120)`; `MANIFESTANT_CONTACT_MAX = 120`; `MAX_ATTACHMENTS = 3`; `REOPEN_WINDOW_DAYS = 30`; `SATISFACTION_WINDOW_DAYS = 30`; `ANONYMOUS_MAX_BODY_BYTES = 16_384`; `ANONYMOUS_KEYS = %w[body health_unit_id]`; `ROLES = %w[ombudsman ombudsman_staff ombudsman_responder]`; `FILE_PURPOSE = "ombudsman_attachment"`; `DOMAIN_EVENTS = %w[ombudsman.manifestation_received ombudsman.forwarded ombudsman.forwarding_replied ombudsman.concluded ombudsman.reopened ombudsman.identity_viewed ombudsman.anonymous_cap_reached]`; `SMS_TEXT = "Rota Saúde: há novidade na sua manifestação à Ouvidoria da Saúde. Veja no canal do cidadão."`.
3. **`ombudsman_settings`:** uma linha, `id = 1` (CHECK), padrões `response_days 30`, `extension_days 30`, `urgent_days 15`, `complement_days 20`, `forwarding_days 20`, `forwarding_extension_days 20`, `warning_days 5`, `anonymous_daily_cap 50`; CHECKs `BETWEEN 1 AND 120` e `anonymous_daily_cap BETWEEN 1 AND 1000`; `OmbudsmanSetting.current`/`ensure_defaults!` (padrão do `TransportSetting`: a migração insere; o banco do dump nasce sem).
4. **Protocolo:** `ombudsman_protocol_seq` (`START 1`, sem `MAXVALUE`), `"#{ano}-#{format('%06d', n)}"`; CHECK `protocol ~ '^[0-9]{4}-[0-9]{6,}$'`; índice único. `OmbudsmanManifestation.next_protocol(received_at)`.
5. **Limites (`rate_limit`, `CitizenApi::BaseController::RateLimitStore`):** `ombudsman_citizen_create` 10/dia por sessão; `ombudsman_citizen_attachment` 20/hora por sessão; `ombudsman_anonymous_hour` 3/hora e `ombudsman_anonymous_day` 10/dia por IP; `ombudsman_person_lookup` 30/min por usuário; `ombudsman_staff_attachment` 20/hora por usuário. Todos respondem 429 `{ error: "rate_limited" }` (os do canal do cidadão hoje respondem `too_many_requests`; o contrato pede `rate_limited` no §4 — o plano usa `rate_limited` em toda rota do 33).
6. **Arquivo:** `Ombudsman::Files` com `TOKEN_PURPOSE = "ombudsman_file"`, `TTL = Exams::Files::TTL`, rota `GET /ombudsman/files/:token` (410 `link_expired`; 404 falso/outro usuário/outro arquivo); `RequestLogPathMask::FILE_TOKEN_PATH` passa a `(exams|tfd|ombudsman)`.
7. **Filtro de log novo:** `/\A(extract|reply|manifestant_name|manifestant_contact|manifestant_cpf|contact|ouvidorsus_number|external_body_name|decision_text|recurring_problems|actions_taken|notes)\z/` (`body`, `reason`, `text`, `justification`, `cpf`, `phone`, `filename`, `data`, `q` já filtrados).
8. **Dev:** api do worktree na porta **3056**; worktree `apps/api/.claude/mod33` (com `config/master.key`); branch `feat/mod-33-ombudsman`; migrações `db/city_migrate/{20261111900001_add_ombudsman_roles_and_departments,20261111900002_add_ombudsman}.rb`; semente (Task 19) em Curitiba.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (os arquivos da task, por nome); commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge e push só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-33.1..F-33.9 do board #1 para In Progress (um comando `gh project item-edit …` por chamada, começando com `gh`). Card vai para Done quando o ÚLTIMO app dele terminar (F-33.1 também dashboard e maintenance; F-33.2, F-33.3 e F-33.8 também wpda; F-33.4..F-33.7 e F-33.9 também dashboard); Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod33 <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. Mudança transversal (`memberships`, `exam_files`, `NoticesController`, `Citizens::Erase`, `CityEncryption`, `Platform::Features`, `spec/support`, initializers) → **suíte completa obrigatória antes do merge**, pedida à sessão "Testes" (Task 20).
- Migrações: dump à mão em `db/city_schema.rb` (fim: `define(version: 2026_11_11_900002)`); triggers em `db/city_triggers.sql`, bloco novo "Módulo 33" depois do bloco do 32b, executado no fim do `up` de cada migração; ambas irreversíveis. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Controladores novos com `wrap_parameters false` (o `ParamsWrapper` põe a chave embrulhada em `request_parameters` e a recusa de chave desconhecida a veria).
- **Texto da manifestação, extrato, respostas, justificativas, nome, CPF, contato, assunto e nome de unidade nunca em log, evento, erro ou URL.** Ids de unidade e de assunto podem estar na query da lista da equipe (A20); nunca no payload de evento. Modelos novos com texto ou identidade sobrescrevem `inspect`/`to_s`/`pretty_print`/`as_json` (só `id` e `state`), como `UrgencyEncounter`.
- Eventos declarados em `config/initializers/domain_events.rb` (todos, com `to: []` quando sem consumidor — o padrão do arquivo) e em `spec/initializers/domain_events_bindings_spec.rb`; payload só `manifestation_id`/`forwarding_id` (e `outcome` em `ombudsman.concluded`); `DomainEvents.publish(` é multilinha — confira cada chamada inteira. **Nenhum `Platform.audit` novo** (R18 não se aplica: a troca de interruptor já é `city.feature_changed`); a Task 20 fixa isso.
- Nunca `Current.city =` solto; interruptores lidos **fora** da transação da cidade.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS`; toda guarda que deixa uma coluna cifrada mudar aceita a re-cifra (`rota.reencrypting`) antes das outras regras.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..51)`.
- Confira nomes reais no código antes de usá-los (o PRÉ-REQUISITO lista os conferidos).
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`, ao encerrar a task.

## Review Focus

1. **Duas técnicas respondem a mesma manifestação ao mesmo tempo** (uma `final_response`, outra `forward`, as duas com `expected_state: "in_analysis"`): uma vence; a outra recebe 409 `manifestation_changed` e nada dela é gravado (nem evento, nem aviso). Teste na Task 12.
2. **Teto anônimo na fronteira:** com `anonymous_daily_cap = 2` e já 1 do dia, duas denúncias simultâneas de IPs diferentes: exatamente uma grava; a outra recebe 429 `daily_cap_reached`; `ombudsman.anonymous_cap_reached` sai **uma** vez no dia, mesmo com um terceiro bloqueio. Às 00:01 locais a contagem recomeça. Teste na Task 8.
3. **Exclusão do cidadão com manifestação aberta e um aviso sendo gravado:** sem deadlock (cidadão → manifestação nas duas pontas); depois da exclusão a manifestação tem os `manifestant_*` preenchidos, `citizen_id` nulo, nenhum aviso do par, a satisfação intacta; o detalhe da equipe em `preserved` continua `identity: "preserved"`. Teste na Task 18.
4. **Complemento × job:** o cidadão responde o complemento no último dia; o job da manhã seguinte não arquiva quem já voltou a `in_analysis`, e quem não respondeu é arquivado uma vez só, mesmo com o job rodando duas vezes. `suspended_days` conta dias corridos locais. Teste nas Tasks 13 e 17.
5. **Identidade preservada:** nenhuma resposta do §5/§6 (lista, detalhe, comprovante, inbox, painel, `person_lookup` depois do registro) traz nome, CPF ou telefone de uma manifestação `preserved`, nem para o `ombudsman`, fora de `POST …/identity`; cada abertura grava uma linha e um evento; o anexo dela só abre pelo ombudsman com justificativa. Teste na Task 10 e na 20.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod33/`)
- Migrações: as duas do Valor fixado 8.
- Modelos: `app/models/{department,ombudsman_responder,ombudsman_subject,ombudsman_setting,ombudsman_manifestation,ombudsman_event,ombudsman_forwarding,ombudsman_identity_access,ombudsman_attachment,ombudsman_reopen_request,ombudsman_satisfaction_response,ombudsman_notice,ombudsman_annual_report,ombudsman_daily_run}.rb`.
- `app/services/ombudsman/{gate,anonymous_gate,catalog,access,deadlines,lock,json,citizen_json,timeline,files,attachments,notify,recipients,panel,input}.rb`.
- `app/commands/ombudsman/{register,register_anonymous,classify,forward,cancel_forwarding,reply_forwarding,decline_forwarding,extend_forwarding,request_complement,record_complement,extend,interim_response,final_response,archive,reopen,change_resolution,request_reopen,decide_reopen,record_satisfaction,view_identity,record_identity_consent,save_department,save_responder,save_subject,save_settings,save_annual_report,publish_annual_report}.rb`.
- `app/controllers/concerns/{ombudsman_gate}.rb`; `app/controllers/ombudsman/{base,manifestations,actions,forwardings,reopen_requests,identities,attachments,files,inbox,responders,subjects,settings,options,person_lookups,dashboard,annual_reports}_controller.rb`; `app/controllers/departments_controller.rb`; `app/controllers/citizen_api/ombudsman/{manifestations,attachments,public_configs,anonymous_reports}_controller.rb`.
- `app/jobs/ombudsman/{deadlines_job,cap_alert_job,mail_job}.rb`; `app/mailers/ombudsman_mailer.rb` + `app/views/ombudsman_mailer/{daily_summary,responder_summary,anonymous_cap}.{html,text}.erb`.
- `lib/ombudsman_crew.rb`; `spec/support/ombudsman_helpers.rb`; specs listados em cada task.

**Modificar**
- Plataforma e infra: `app/services/platform/features.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging,request_log_path_mask}.rb`, `db/seeds.rb`.
- Papéis e anexos: `app/models/{membership,exam_file}.rb`.
- Cidadão: `app/controllers/citizen_api/notices_controller.rb`.
- LGPD: `app/commands/citizens/erase.rb` (e o comentário de `request_erasure.rb`).
- Specs existentes: `spec/adr_pointers_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/models/city_record_settings_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/lib/request_log_path_mask_spec.rb`, `spec/requests/citizen_api/{notices_spec,session_features_spec}.rb`.

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-33.1..F-33.9 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod33 -b feat/mod-33-ombudsman origin/main && cp config/master.key .claude/mod33/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO com `grep` no worktree: `define(version: 2026_11_10_900003)`, `ls db/city_migrate | tail -1`, `"urgency_stock_deduction"` como última `Entry`, `CITIZEN_HIDDEN`, `ROLES = ` no `AddPsychosocialCare` e no `Membership`, `PURPOSES = ` do `ExamFile`, `ck_exam_files_purpose`, `def claim!` do `Exams::Files`, `delete_all` do `Exams::DailyJob`, `FILE_TOKEN_PATH`, `VALID_RANGE`, `TransportTripNotice` no `erase.rb`, `lab_requisition_number_seq`, e que nenhuma tabela `departments`/`ombudsman_*` existe. Diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod33 spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb spec/models/membership_roles_spec.rb spec/requests/citizen_api/notices_spec.rb spec/commands/citizens/erase_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS (linha de base).

**Relato ao coordenador:** "api 33 · Task 0 · worktree mod33 em origin/main <hash>, pré-requisito conferido (diferenças: …), linha de base verde."

---

### Task 1: Interruptores, portões, catálogo e filtro de log (transversal: lista de interruptores)

**Files:**
- Create: `app/services/ombudsman/{gate,anonymous_gate,catalog}.rb`, `app/controllers/concerns/ombudsman_gate.rb`, `spec/services/ombudsman/catalog_spec.rb`
- Modify: `app/services/platform/features.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/services/platform/features_spec.rb`, `spec/models/city_record_settings_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`, `spec/adr_pointers_spec.rb`

**Interfaces:**
- Produces: `Ombudsman::Gate::KEY = "ombudsman"`, `Ombudsman::Gate.usable?(city)`; `Ombudsman::AnonymousGate::KEY = "ombudsman_anonymous_web"`, `.usable?(city)` (o padrão do `Psychosocial::Gate`: `false` para `city` nil ou sem id); concern `OmbudsmanGate#require_ombudsman!` e `#require_ombudsman_anonymous!` (403 `{ error: "feature_disabled", feature: KEY }`); `Ombudsman::Catalog` (Valor fixado 2).

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: o exemplo do 32a troca `keys.last(3).first(2)` por `keys.last(5).first(2)`; o do 32b troca `CATALOG.last` por `CATALOG[-3]`; exemplo novo:

```ruby
it "módulo 33: ombudsman fecha o catálogo; a denúncia anônima exige a ouvidoria; as duas vão ao cidadão" do
  keys = Platform::Features::CATALOG.map(&:key)
  expect(keys.last(2)).to eq(%w[ombudsman ombudsman_anonymous_web])
  expect(described_class.find("ombudsman").requires).to eq([])
  expect(described_class.find("ombudsman").description).to eq("Ouvidoria do SUS municipal")
  expect(described_class.find("ombudsman_anonymous_web").requires).to eq(%w[feature:ombudsman])
  expect(described_class::CITIZEN_HIDDEN).not_to include("ombudsman", "ombudsman_anonymous_web")
end

it "denúncia anônima ligada sem a ouvidoria fica inutilizável com ombudsman_disabled" do
  city = clinical_city!
  Platform::Features.set!(city: city, key: "ombudsman_anonymous_web", enabled: true, maintainer: ledi_maintainer!)
  expect(Platform::Features.missing(city, "ombudsman_anonymous_web")).to eq(%w[ombudsman_disabled])
  expect(Ombudsman::AnonymousGate.usable?(city)).to be(false)
end
```

As três listas exatas ganham `ombudsman ombudsman_anonymous_web` no fim. `session_features_spec.rb`: com as duas ligadas, `GET /citizen/session` traz as duas. `catalog_spec.rb`: `KINDS`, `MODES`, `STATES`, `ARCHIVE_REASONS`, `CITIZEN_EVENT_KINDS ⊂ EVENT_KINDS`, `DOMAIN_EVENTS` exatamente os 7 do contrato; `ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters)` filtra `extract`, `reply`, `manifestant_name`, `manifestant_contact`, `contact`, `ouvidorsus_number`, `external_body_name`, `decision_text`, `recurring_problems`, `actions_taken`, `body`, `justification`.

- [ ] **Step 2:** `bin/test api -w mod33 spec/services/platform/features_spec.rb spec/services/ombudsman/catalog_spec.rb spec/models/city_record_settings_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/citizen_api/session_features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** as duas entradas (comentário do padrão: "ADR 0051: sem cascata; desligar a ouvidoria deixa a denúncia anônima ligada faltando pré-requisito, e o formulário público responde feature_disabled"), os portões, o concern, o catálogo, o filtro (Valor fixado 7, com o comentário "ADR 0051: …") e `VALID_RANGE = (1..51)`.
- [ ] **Step 4:** os mesmos specs + `spec/adr_pointers_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman and anonymous web report switches`.

**Relato ao coordenador:** "api 33 · Task 1 · `ombudsman` e `ombudsman_anonymous_web` no fim do CATALOG (missing `ombudsman_disabled` — o rótulo do maintenance), portão 403 feature_disabled (A1), filtro de log novo."

---

### Task 2: Papéis, finalidade do anexo e setores da secretaria (migração 900001, transversal)

**Files:**
- Create: `db/city_migrate/20261111900001_add_ombudsman_roles_and_departments.rb`, `app/models/department.rb`, `spec/models/department_spec.rb`
- Modify: `app/models/{membership,exam_file}.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`

**Interfaces:**
- Produces: `AddOmbudsmanRolesAndDepartments::ROLES = (AddPsychosocialCare::ROLES + %w[ombudsman ombudsman_responder ombudsman_staff]).sort`; `::PURPOSES = %w[eqa_annual_report tfd_accountability ombudsman_attachment]`; `Membership::ROLES` igual; `Membership::PRIVILEGED_ROLES` + `ombudsman` (comentário: "ADR 0051: o ouvidor abre a identidade preservada e arquiva — step-up para conceder; o mantenedor não concede"; `ombudsman_staff` e `ombudsman_responder` NÃO entram, pelo contrato — ver A2); `ExamFile::PURPOSES` + `ombudsman_attachment`; `departments` (`id uuid`, `name string(80) NOT NULL`, `active bool NOT NULL DEFAULT true`, `updated_by_user_id` FK users, timestamps; índice único `lower(name)` `idx_departments_name`); trigger `departments_no_delete` (`rota_no_delete`) e `_append_only_truncate`; `Department` (`validates :name, length: Catalog::NAME`, `scope :active`).

- [ ] **Step 1: Write the failing tests.** `membership_roles_spec.rb`: exemplo novo "ombudsman é privilegiado; ombudsman_staff e ombudsman_responder não; o banco aceita os três"; o último exemplo ganha `expect(AddOmbudsmanRolesAndDepartments::ROLES).to eq((AddPsychosocialCare::ROLES + %w[ombudsman ombudsman_staff ombudsman_responder]).sort)` e passa a comparar `described_class::ROLES` com a lista nova. Os dois specs de migração passam a `require` a nova migração e a restaurar `AddOmbudsmanRolesAndDepartments::ROLES` (comentário: "… e ombudsman/ombudsman_staff/ombudsman_responder (33)"). `department_spec.rb`: nome repetido com outra caixa → `ActiveRecord::RecordNotUnique`; `Department.first.destroy` → `ActiveRecord::StatementInvalid` (`DELETE refused`); `ExamFile.create!(…, purpose: "ombudsman_attachment")` passa e `purpose: "x"` viola `ck_exam_files_purpose`.
- [ ] **Step 2:** `bin/test api -w mod33 spec/models/membership_roles_spec.rb spec/models/department_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`replace_check` copiado do 31 para `ck_memberships_role` e `ck_exam_files_purpose`; `create_table :departments`; `execute File.read(Rails.root.join("db/city_triggers.sql"))`; `down` → `IrreversibleMigration`), o dump, o bloco "Módulo 33 — setores" no `city_triggers.sql` (`IF to_regclass('public.departments')`), os modelos.
- [ ] **Step 4:** os mesmos specs + `spec/models/exam_tracking_schema_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman roles, attachment purpose and secretariat departments`.

**Relato ao coordenador:** "api 33 · Task 2 · migração 20261111900001: três papéis (só `ombudsman` privilegiado, A2), `exam_files.purpose` ombudsman_attachment, `departments` sem DELETE; ROLE_OPTIONS do dashboard pode usar os três nomes."

---

### Task 3: Esquema da ouvidoria, modelos, triggers e cifra (migração 900002)

**Files:**
- Create: `db/city_migrate/20261111900002_add_ombudsman.rb`, os 13 modelos `ombudsman_*` do Mapa, `spec/models/ombudsman_schema_spec.rb`, `spec/support/ombudsman_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`

**Interfaces:**
- Produces (tabelas, com as colunas do contrato §1 e estas precisões):
  - `ombudsman_responders`: CHECK `ck_ombudsman_responders_destination` = `num_nonnulls(department_id, health_unit_id) = 1`; únicos parciais `(user_id, department_id) WHERE department_id IS NOT NULL` e `(user_id, health_unit_id) WHERE health_unit_id IS NOT NULL`;
  - `ombudsman_subjects`: `name string(80)`, `parent_id` FK self, `active`, `position int`; único `(COALESCE(parent_id, '00000000-0000-0000-0000-000000000000'), lower(name))`; insert/update guard `rota_ombudsman_subject_guard()` (pai não pode ter pai; pai não pode virar filho de outro);
  - `ombudsman_settings` (Valor fixado 3);
  - `ombudsman_manifestations`: CHECKs de `kind`, `identification_mode`, `channel`, `state`, `resolution`, `archive_reason`; `ck_ombudsman_manifestations_anonymous` = `identification_mode <> 'anonymous' OR (kind = 'report' AND citizen_id IS NULL AND manifestant_name IS NULL AND manifestant_contact IS NULL AND manifestant_cpf IS NULL)`; `ck_…_ouvidorsus` = `(channel = 'ouvidorsus') = (ouvidorsus_number IS NOT NULL)`; `ck_…_duplicate` = `(archive_reason = 'duplicate') = (duplicate_of_id IS NOT NULL)`; `ck_…_elsewhere` = `(archive_reason = 'forwarded_elsewhere') = (external_body_name IS NOT NULL)`; `ck_…_archived` = `(state = 'archived') = (archive_reason IS NOT NULL)`; `ck_…_web_creator` = `(channel = 'web') = (created_by_user_id IS NULL)`; `ck_…_cycle` (`cycle >= 1`, `suspended_days >= 0`); índices `(state, received_at)`, `(citizen_id) WHERE citizen_id IS NOT NULL`, `(channel, identification_mode, received_at)` (teto anônimo), `(protocol)` único;
  - `ombudsman_events`: `kind` CHECK (Desvio 5), `actor` CHECK, `ck_ombudsman_events_actor_user` = `(actor = 'staff') = (actor_user_id IS NOT NULL)`; índice `(ombudsman_manifestation_id, created_at, id)`;
  - `ombudsman_forwardings`: `ck_…_destination` (um de dois), `state` CHECK, `ck_…_reply` = `(state IN ('replied','declined')) = (reply IS NOT NULL)`; `decline` grava o motivo em `reply` (A11); índice `(state, due_on)`;
  - `ombudsman_identity_accesses`: `scope` CHECK, `ck_…_file` = `(scope = 'attachment') = (exam_file_id IS NOT NULL)`;
  - `ombudsman_attachments` (Desvio 6): `ck_…_owner` = `(uploaded_by = 'citizen') = (citizen_session_id IS NOT NULL) AND (uploaded_by = 'staff') = (uploaded_by_user_id IS NOT NULL)`; FK `exam_file_id` `ON DELETE CASCADE`; índice `(created_at) WHERE ombudsman_manifestation_id IS NULL`;
  - `ombudsman_reopen_requests`: `cycle int NOT NULL` (A12); únicos `(ombudsman_manifestation_id) WHERE state = 'pending'` e `(ombudsman_manifestation_id, cycle)`;
  - `ombudsman_satisfaction_responses`: único `ombudsman_manifestation_id`; CHECKs de `met` e `score BETWEEN 1 AND 5`;
  - `ombudsman_notices`: `kind` CHECK; índice `(citizen_id, read_at)`;
  - `ombudsman_annual_reports`: `year` único, CHECK `year BETWEEN 2000 AND 2999`, `state` CHECK, `ck_…_published` = `(state = 'published') = (published_at IS NOT NULL AND snapshot IS NOT NULL)`;
  - `ombudsman_daily_runs`: `run_on date`, `kind string(30)` CHECK `kind IN ('deadlines','anonymous_cap')`, `created_at`; único `(run_on, kind)` (A13).
- Triggers (bloco "Módulo 33" no `city_triggers.sql`, dentro de `IF to_regclass('public.ombudsman_manifestations') IS NOT NULL`):
  - `CREATE SEQUENCE IF NOT EXISTS ombudsman_protocol_seq START 1;`
  - `rota_ombudsman_manifestation_guard()` (BEFORE UPDATE OR DELETE): DELETE recusado; re-cifra (`rota.reencrypting`, só `body`, `extension_reason`, `manifestant_*` diferentes) passa primeiro; `protocol`, `body`, `channel`, `identification_mode`, `received_at`, `created_by_user_id`, `ouvidorsus_number` nunca mudam; `citizen_id` só de valor a NULL, e então os `manifestant_*` podem passar de NULL a preenchidos na MESMA linha (fora disso, `manifestant_*` não mudam);
  - `ombudsman_events`: `rota_reencryption_only('text')`; `ombudsman_identity_accesses`: `rota_reencryption_only('justification')`; `ombudsman_satisfaction_responses`: `rota_append_only()`;
  - `rota_ombudsman_forwarding_guard()`: DELETE recusado; re-cifra (`extract`, `reply`, `extension_reason`) primeiro; `extract`, destino, `created_by_user_id`, `ombudsman_manifestation_id` imutáveis; `reply` só NULL → valor; estado só `pending` → outro, uma vez; `extended` só `false → true`, uma vez, com `due_on` e `extension_reason` juntos;
  - `rota_ombudsman_attachment_guard()`: DELETE só com `ombudsman_manifestation_id IS NULL` (a cascata do órfão); UPDATE só da ligação (`ombudsman_manifestation_id`/`event_id` de NULL a valor, uma vez) ou re-cifra de `filename`;
  - `rota_ombudsman_reopen_request_guard()`: DELETE recusado; re-cifra (`reason`, `decision_text`) primeiro; `pending` → `accepted|rejected` uma vez, com `decided_*`; `reason` imutável;
  - `rota_ombudsman_notice_guard()`: o do `transport_trip_notices` (só `read_at`, uma vez); DELETE permitido (exclusão);
  - `rota_ombudsman_annual_report_guard()`: DELETE recusado; o publicado nunca muda;
  - `rota_no_delete()` em `ombudsman_responders`, `ombudsman_subjects`, `ombudsman_settings`, `ombudsman_daily_runs`; `rota_append_only()` de TRUNCATE em todas as tabelas do módulo.
- `CITY_KEYED_TARGETS` (comentário "ADR 0051 (módulo 33): …"): `[OmbudsmanManifestation, :manifestant_name|:manifestant_contact|:manifestant_cpf|:body|:extension_reason]`, `[OmbudsmanEvent, :text]`, `[OmbudsmanForwarding, :extract|:extension_reason|:reply]`, `[OmbudsmanIdentityAccess, :justification]`, `[OmbudsmanAttachment, :filename]`, `[OmbudsmanReopenRequest, :reason|:decision_text]`.
- Modelos: `OmbudsmanManifestation.next_protocol(received_at)` (`"#{received_at.in_time_zone.year}-#{format('%06d', connection.select_value("SELECT nextval('ombudsman_protocol_seq')"))}"`); `#open?`, `#final?`, `#anonymous?`, `#identity_protected?` (`preserved` ou `anonymous`); `OmbudsmanSetting.current`/`ensure_defaults!`; `OmbudsmanSubject.ensure_defaults!` (insere a árvore do contrato §7 só quando a tabela está vazia; nunca mexe na gravada); `inspect` só com `id`/`state`.
- `spec/support/ombudsman_helpers.rb`: `ombudsman_city!(anonymous: false)` (clínica + interruptores + `OmbudsmanSetting.ensure_defaults!` + `OmbudsmanSubject.ensure_defaults!`), `ombudsman!`, `ombudsman_staff!`, `responder!(destination)`, `department!(name)`, `web_citizen!(cpf:, phone:)`, `manifestation!(**attrs)` (pelo `Ombudsman::Register` a partir da Task 7; antes, `create!` cru com protocolo), `subject_leaf!`.

- [ ] **Step 1: Write the failing test** `spec/models/ombudsman_schema_spec.rb`:

```ruby
RSpec.describe "Esquema da ouvidoria (ADR 0051)" do
  before { ombudsman_city! }

  it "protocolo AAAA-NNNNNN sequencial e único; o ano é o local" do
    travel_to Time.zone.local(2026, 12, 31, 23, 59) do
      a = OmbudsmanManifestation.next_protocol(Time.current)
      b = OmbudsmanManifestation.next_protocol(Time.current + 2.minutes)
      expect(a).to match(/\A2026-\d{6}\z/)
      expect(b).to start_with("2027-")
      expect(b.split("-").last.to_i).to eq(a.split("-").last.to_i + 1)
    end
  end

  it "colunas do registro nunca mudam; citizen_id só vai a nulo, com os manifestant_* juntos" do
    m = raw_manifestation!(citizen: web_citizen!)
    expect { m.update_columns(body: "outro texto da manifestação") }.to raise_error(ActiveRecord::StatementInvalid, /ombudsman_manifestations/)
    expect { m.update_columns(manifestant_name: "Fulano") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { m.update_columns(citizen_id: nil, manifestant_name: "Maria Souza") }.not_to raise_error
    expect { m.update_columns(citizen_id: web_citizen!(cpf: "11144477735").id) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { m.destroy }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "anônima só denúncia e sem dado de pessoa; eventos, acessos e satisfação são só acréscimo" do
    expect { raw_manifestation!(kind: "complaint", identification_mode: "anonymous") }.to raise_error(ActiveRecord::StatementInvalid, /ck_ombudsman_manifestations_anonymous/)
    m = raw_manifestation!
    event = OmbudsmanEvent.create!(ombudsman_manifestation: m, kind: "received", actor: "system", data: {}, visible_to_citizen: true)
    expect { event.update_columns(visible_to_citizen: false) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { event.destroy }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "encaminhamento: extrato imutável, resposta uma vez; aviso só read_at; relatório publicado congela" do
    # extract/reply/state/extended pelos triggers; OmbudsmanNotice só read_at uma vez e DELETE ok;
    # OmbudsmanAnnualReport publicado: update_columns(notes:) recusado.
  end

  it "o anexo órfão some com o arquivo órfão; o ligado nunca" do
    orphan = raw_attachment!
    linked = raw_attachment!(manifestation: raw_manifestation!)
    ExamFile.where(id: orphan.exam_file_id).delete_all
    expect(OmbudsmanAttachment.exists?(orphan.id)).to be(false)
    expect { linked.destroy }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "re-cifra passa nas tabelas imutáveis" do
    # SET LOCAL rota.reencrypting = 'on' + update da coluna cifrada com o mesmo valor claro → ok,
    # para manifestations.body, events.text, forwardings.extract, identity_accesses.justification.
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod33 spec/models/ombudsman_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (tabelas, semente de assuntos e a linha de `ombudsman_settings` por SQL fixo na própria migração — sem ler código da aplicação, como o 31 —, `execute File.read(…city_triggers.sql)`), o dump, o bloco de triggers, os modelos, a cifra, o helper.
- [ ] **Step 4:** os mesmos specs + `spec/models/department_spec.rb spec/models/exam_tracking_schema_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman schema with protocol sequence, immutability triggers and city encryption`.

**Relato ao coordenador:** "api 33 · Task 3 · migração 20261111900002: 14 tabelas, `ombudsman_protocol_seq`, triggers de imutabilidade; `ombudsman_attachments` com `filename` e dono (A3/A4), `reopen_requests.cycle` (A12), `daily_runs.kind` (A13)."

---

### Task 4: Acesso, prazos, trava, base do controlador e formatos

**Files:**
- Create: `app/services/ombudsman/{access,deadlines,lock,input,json}.rb`, `app/controllers/ombudsman/base_controller.rb`, `spec/services/ombudsman/{access,deadlines,lock}_spec.rb`

**Interfaces:**
- Produces:
  - `Ombudsman::Access`: `staff?(user)` (`ombudsman_staff` ou `ombudsman`), `ombudsman?(user)`, `responder?(user)` (papel `ombudsman_responder`), `destinations(user) -> { department_ids: [], health_unit_ids: [] }` (linhas `active` de `ombudsman_responders` do usuário, só com o papel ativo), `responder_ids_for(department_id:, health_unit_id:)` (respondentes ativos com papel ativo e usuário não desativado), `admin?(user)` (`municipal_admin`), `config_reader?(user)` (admin, staff ou ombudsman);
  - `Ombudsman::Deadlines`: `due_on_sql(settings, today:)` (expressão SQL com os inteiros ligados por `sanitize_sql_array`, Desvio 3), `color_sql(settings, today:)` (`'paused' | 'red' | 'yellow' | 'green' | NULL`), `due_on(manifestation, settings, today:)` e `color(…)` (espelho), `forwarding_due_on(created_on, settings)`, `forwarding_color(forwarding, settings, today:)`, `complement_due_on(m, settings)`;
  - `Ombudsman::Lock.manifestation!(id) -> OmbudsmanManifestation | nil` e `Ombudsman::Lock.check(m, expected_state) -> Result | nil` (Desvio 4);
  - `Ombudsman::BaseController`: `include Authentication`, `MfaStepUp`, `::OmbudsmanGate`; `wrap_parameters false`; `before_action :require_ombudsman!`; `FAILURE_STATUS` (`manifestation_changed`, `already_answered`, `reopen_pending`, `forwarding_not_pending`, `already_decided` → 409; `missing_role`, `not_own_citizen` → 403; `not_found` → 404; `rate_limited`, `daily_cap_reached` → 429; o resto 422); `render_failure`, `render_refusal`, `forbid`, `not_found`, `uuid?`, `body`, `body_param`, `query_only!`, `require_staff!`, `require_ombudsman_role!`, `load_manifestation` (404 quando não existe);
  - `Ombudsman::Json`: `list_item(m, settings, today, pending_counts)`, `detail(m, viewer:, settings:, today:)` (identidade pela regra do §8: `identified` → `{ name, cpf_masked, phone_masked, contact }`, demais → `"preserved"`), `event(e)`, `forwarding(f, settings, today)`, `subject(s)`, `destination(f)`.
- Regra do espelho: `Ombudsman::Deadlines.due_on` e `due_on_sql` dão o mesmo valor para as combinações (urgente × prorrogada × suspensa × aguardando complemento × final), conferido no spec com `OmbudsmanManifestation.select("#{due_on_sql} AS d")`.

- [ ] **Step 1: Write the failing tests.** `deadlines_spec.rb`: recebida hoje → `+30`, verde; urgente → `+15`; prorrogada → `+60`; `suspended_days: 7` → `+37`; a 5 dias do fim → amarela; um dia depois do fim → vermelha; em `awaiting_complement` pedida há 4 dias → `paused` e `due_on` `+34`; `answered` → cor nil; o SQL e o Ruby iguais em todas; `warning_days: 1` muda a amarela. `access_spec.rb`: `ombudsman` é staff; respondente sem linha ativa não tem destino; linha ativa com papel revogado não conta; usuário desativado fora de `responder_ids_for`. `lock_spec.rb`: `expected_state` errado → `manifestation_changed`; malformado → `invalid_request` `path: "expected_state"`; com o cidadão travado em `FOR UPDATE` por outra conexão (`spec/support/lock_wait.rb`), `Lock.manifestation!` espera.
- [ ] **Step 2:** `bin/test api -w mod33 spec/services/ombudsman/deadlines_spec.rb spec/services/ombudsman/access_spec.rb spec/services/ombudsman/lock_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces (o padrão de `RegulationRequest.overdue_sql`/`#overdue?` para o par SQL/Ruby — A5).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman access, deadline sql with ruby mirror and manifestation lock`.

**Relato ao coordenador:** "api 33 · Task 4 · prazos em SQL+Ruby (`paused` soma os dias do complemento em curso), trava cidadão→manifestação; erros 409/403/404/422 fixados."

---

### Task 5: Configuração — setores, respondentes, assuntos, prazos e opções (§7)

**Files:**
- Create: `app/controllers/departments_controller.rb`, `app/controllers/ombudsman/{responders,subjects,settings,options}_controller.rb`, `app/commands/ombudsman/{save_department,save_responder,save_subject,save_settings}.rb`, `spec/requests/ombudsman/configuration_spec.rb`
- Modify: `config/routes.rb` (`get/post "departments"`, `patch "departments/:id"`; `scope "/ombudsman", module: :ombudsman, as: :ombudsman`)

**Interfaces:**
- `GET /departments` → `{ items: [{ id, name, active }] }` (admin, staff, ombudsman; `?active=true` opcional); `POST /departments { name }` e `PATCH /departments/:id { name?, active? }` (só `municipal_admin`; nome repetido → 409 `name_taken`; desativar setor com encaminhamento `pending` é permitido). **Sem portão** (tabela genérica, A14).
- `GET /ombudsman/responders` → `{ items: [{ id, user: { id, email, name }, destination: { kind: "department" | "health_unit", id, name }, active }] }`; `POST { user_id, department_id? | health_unit_id? }` (usuário com o papel `ombudsman_responder` ativo, senão 422 `missing_responder_role` `path: "user_id"`; destino ativo; repetido → 409 `already_responder`); `PATCH /:id { active }`. Só ombudsman.
- `GET /ombudsman/subjects` → `{ items: [{ id, name, active, children: [{ id, name, active }] }] }` (`OmbudsmanSubject.ensure_defaults!` se vazio — A15); `POST { name, parent_id? }` (neto → 422 `too_deep`); `PATCH /:id { name?, active? }`. Leitura para staff; escrita só ombudsman.
- `GET /ombudsman/settings` → os oito inteiros; `PATCH` (cada chave opcional; fora de 1..120 / 1..1000 → 422 com `path`; chave desconhecida → 422). Só ombudsman.
- `GET /ombudsman/options` → `{ subjects: [...ativos, com children ativos], health_units: [{ id, name, kind }], professionals: [{ id, name }], departments: [{ id, name }] }` (staff e ombudsman; A9).

- [ ] **Step 1: Write the failing test** `configuration_spec.rb`: admin cria e renomeia setor; técnico lê e não cria (403); ombudsman cadastra respondente num setor e numa unidade, desliga; papel faltando → 422; assunto com subassunto; neto → `too_deep`; prazos fora de faixa → 422 `path: "response_days"`; `options` só ativos; com `ombudsman` desligado → 403 `feature_disabled` em `/ombudsman/*` e `/departments` segue respondendo.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/configuration_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman configuration for departments, responders, subjects and deadlines`.

**Relato ao coordenador:** "api 33 · Task 5 · `/departments` (sem portão, A14), `/ombudsman/{responders,subjects,settings,options}` (options é novo, A9) — formatos para o dashboard."

---

### Task 6: Anexos — upload do cidadão e da equipe, ligação e link curto

**Files:**
- Create: `app/services/ombudsman/{attachments,files}.rb`, `app/controllers/citizen_api/ombudsman/attachments_controller.rb`, `app/controllers/ombudsman/{attachments,files}_controller.rb`, `spec/requests/ombudsman/attachments_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/request_log_path_mask.rb`, `spec/lib/request_log_path_mask_spec.rb`

**Interfaces:**
- `Ombudsman::Attachments.upload(data:, content_type:, filename:, citizen_session: nil, user: nil) -> Result.ok(attachment:) | fail(:invalid_file | :invalid_request, details: { path: })` — `Exams::Files.store_base64(…, purpose: "ombudsman_attachment")` + `OmbudsmanAttachment.create!` na mesma transação; `filename` em `FILENAME` (sem barra nem caractere de controle).
- `Ombudsman::Attachments.link!(ids:, manifestation:, owner:, event: nil) -> Result.ok(attachments:) | fail(:invalid_request, details: { path: "attachment_ids" }) | fail(:too_many_attachments)` — ids uuid únicos, até `MAX_ATTACHMENTS`, todos do dono, sem manifestação e com o `ExamFile` não consumido, sob trava (`FOR UPDATE` por id); grava `ombudsman_manifestation_id`/`event_id` e `consumed_at`.
- `POST /citizen/ombudsman/attachments { filename, content_type, data }` → 201 `{ id }` (sessão; portão `ombudsman`; 20/h por sessão); `POST /ombudsman/attachments` igual (staff; dono = usuário).
- `POST /ombudsman/attachments/:id/link { justification? }` → `{ url, expires_at }`: anexo de manifestação `identified` → staff ou ombudsman; `preserved`/`anonymous` → só ombudsman (403 `missing_role` para o técnico), com `justification` em `JUSTIFICATION` (senão 422 `path: "justification"`), gravando `ombudsman_identity_accesses(scope: "attachment", exam_file_id:)` e `ombudsman.identity_viewed`; anexo órfão → 404.
- `GET /ombudsman/files/:token` → bytes (`inline`, `Cache-Control: no-store`, nome do anexo); 410 `link_expired`; 404 falso, de outro usuário ou de arquivo que não é da ouvidoria.

- [ ] **Step 1: Write the failing test** `attachments_spec.rb`: PNG válido sobe (201) e nasce órfão; texto com cabeçalho falso de PDF → 422 `invalid_file`; 11 MB → 422; 21º upload na hora → 429 `rate_limited` (com `MemoryStore`); link de anexo `identified` pelo técnico abre o arquivo; de `preserved` pelo técnico → 403; pelo ombudsman sem justificativa → 422 e com justificativa → URL + uma linha de acesso + evento só com ids; o token vencido → 410; o caminho `/ombudsman/files/<token>` sai mascarado no log (`capture_log`); o órfão de 25 h some pelo `Exams::DailyJob.perform_now` sem erro (cascata); o `filename` nunca aparece no log.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/attachments_spec.rb spec/lib/request_log_path_mask_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o `Ombudsman::Files` no padrão do `Tfd::Files`, verifier próprio; `FILE_TOKEN_PATH` com `ombudsman`).
- [ ] **Step 4:** os mesmos specs + `spec/jobs/exams/daily_job_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman attachments with owner, encrypted name and short signed links`.

**Relato ao coordenador:** "api 33 · Task 6 · anexos: upload devolve `{id}`, ligação na criação/complemento, link curto `/ombudsman/files/:token` (rota nova, A16); anexo protegido só pelo ombudsman com justificativa."

---

### Task 7: Avisos ao cidadão e SMS — 4ª fonte da caixa (§9)

**Files:**
- Create: `app/services/ombudsman/notify.rb`, `spec/services/ombudsman/notify_spec.rb`, `spec/requests/citizen_api/ombudsman_notices_spec.rb`
- Modify: `app/controllers/citizen_api/notices_controller.rb`, `spec/requests/citizen_api/notices_spec.rb` (só se algum exemplo contar as fontes)

**Interfaces:**
- `Ombudsman::Notify.call(manifestation:, kind:, now: Time.current) -> Boolean` — chamado DENTRO da transação do comando, depois da trava (o cidadão já está em `FOR KEY SHARE`); sem `citizen_id` → `false` e nada; `kind` fora de `NOTICE_KINDS` → `ArgumentError`; insere `ombudsman_notices`; SMS (`Catalog::SMS_TEXT`, sem protocolo) só com `Campaigns::SmsSetting.enabled?`, `SmsGateway.configured?`, hora local em `Campaigns::SmsBatchJob::WINDOW_HOURS`, `sms_opt_in` e sem `notices_muted` (o `appointment_reminders_muted` não vale aqui — A17); falha do provedor só com a classe no log; fora da janela não envia nem reagenda.
- `NoticesController#index`: 4ª fonte `OmbudsmanNotice.where(citizen_id: ids).includes(:ombudsman_manifestation)` → `{ kind: "ombudsman", id, event: <kind>, protocol, created_at, read, cpf_masked }` (`cpf_masked` como as outras fontes — A18); entra no `unread_count` (com `counted`); `read` procura também nela (só `read_at` NULL → agora).

- [ ] **Step 1: Write the failing tests.** `notify_spec.rb`: aviso nasce sem texto; sem `citizen_id` nada; SMS sai às 10h com opt-in e não às 21h; com `notices_muted` não sai; o texto do SMS é exatamente `SMS_TEXT` (sem protocolo). `ombudsman_notices_spec.rb`: o aviso aparece na caixa com `kind: "ombudsman"`, conta no `unread_count`, `POST /citizen/notices/:id/read` marca; o JSON tem exatamente as chaves `kind id event protocol created_at read cpf_masked`; o aviso de outro telefone → 404.
- [ ] **Step 2:** `bin/test api -w mod33 spec/services/ombudsman/notify_spec.rb spec/requests/citizen_api/ombudsman_notices_spec.rb spec/requests/citizen_api/notices_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (comentário do topo do `NoticesController` passa a "Quatro fontes … e avisos da ouvidoria (ombudsman_notices, módulo 33: só o protocolo e o tipo)").
- [ ] **Step 4:** os mesmos specs + `spec/requests/citizen_api/{trip_notices_spec,appointment_notices_spec}.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman notices as the fourth citizen inbox source with optional sms`.

**Relato ao coordenador:** "api 33 · Task 7 · caixa do cidadão com `kind: "ombudsman"` `{id, event, protocol, created_at, read, cpf_masked}` (cpf_masked a mais, A18) — rótulos para o wpda."

---

### Task 8: Registro pelo cidadão no wpda — criar, listar, detalhar (§3)

**Files:**
- Create: `app/commands/ombudsman/register.rb`, `app/services/ombudsman/{citizen_json,timeline}.rb`, `app/controllers/citizen_api/ombudsman/manifestations_controller.rb`, `spec/requests/citizen_api/ombudsman/manifestations_spec.rb`, `spec/commands/ombudsman/register_spec.rb`
- Modify: `config/routes.rb` (dentro do `scope "/citizen"`: `scope "ombudsman", module: "ombudsman", as: "ombudsman"`), `config/initializers/domain_events.rb` + bindings spec (`ombudsman.manifestation_received`, `to: []`)

**Interfaces:**
- `Ombudsman::Register.call(attrs:, channel:, citizen: nil, citizen_session: nil, by: nil, now: Time.current) -> Result.ok(manifestation:) | fail(:invalid_request, details: { path:, problems: })` — valida `kind`, modo (no canal `web` com sessão: `identified|preserved`; `anonymous` → 422 `invalid_request` `path: "identification_mode"`), `body` em `BODY`, `health_unit_id` (unidade ativa), `attachment_ids`; numa transação: `next_protocol`, `create!` (`state: "received"`, `cycle: 1`, `cycle_started_on`), evento `received` (`actor: citizen|staff`, `visible_to_citizen: true`), `Attachments.link!`, `DomainEvents.publish("ombudsman.manifestation_received", manifestation_id:)`. Sem aviso (o cidadão já vê o protocolo).
- `GET /citizen/ombudsman/manifestations` → `{ manifestations: [{ id, protocol, kind, identification_mode, state, received_at, due_on, color, cpf_masked, has_unread }] }`, todos os cidadãos da sessão, `received_at DESC, id DESC`; `cpf_masked` só quando a sessão tem mais de um.
- `GET /citizen/ombudsman/manifestations/:id` → o formato do §3; `timeline` = eventos `visible_to_citizen` (`{ kind, at, text? }`, com `text` só em `complement_requested`, `extended`, `interim_response`, `final_response`, `reopen_rejected` e o rótulo do motivo em `archived`); `can_complement`, `can_request_reopen`, `can_answer_satisfaction` pela regra das Tasks 13–15; manifestação de cidadão fora da sessão → 404.
- `POST /citizen/ombudsman/manifestations { citizen_id, kind, identification_mode, body, health_unit_id?, attachment_ids? }` → 201 `{ id, protocol, due_on }`; `citizen_id` fora da sessão → 403 `not_own_citizen` (contrato; A21); 10/dia por sessão; chave desconhecida → 422.

- [ ] **Step 1: Write the failing test** `manifestations_spec.rb`:

```ruby
RSpec.describe "Manifestação pelo canal do cidadão", type: :request do
  let!(:city) { ombudsman_city! }
  let!(:citizen) { web_citizen!(cpf: "52998224725", phone: "+5541998765432") }
  before { sign_in_citizen("+5541998765432") }

  it "identificada com anexo: protocolo, prazo de 30 dias e linha do tempo com 'recebida'" do
    upload = json_post("/citizen/ombudsman/attachments", filename: "foto.png", content_type: "image/png", data: png_base64) && json_body["id"]
    expect {
      json_post "/citizen/ombudsman/manifestations", citizen_id: citizen.id, kind: "complaint", identification_mode: "identified",
                body: "Esperei quatro horas na recepção da UBS sem atendimento.", attachment_ids: [ upload ]
    }.to change { DomainEvent.where(name: "ombudsman.manifestation_received").count }.by(1)
    expect(response).to have_http_status(:created)
    expect(json_body["protocol"]).to match(/\A\d{4}-\d{6}\z/)
    expect(json_body["due_on"]).to eq((Time.zone.today + 30).iso8601)
    get "/citizen/ombudsman/manifestations/#{json_body["id"]}"
    expect(json_body["timeline"].map { |e| e["kind"] }).to eq(%w[received])
    expect(json_body["attachments"]).to eq([ { "id" => upload, "filename" => "foto.png" } ])
    expect(DomainEvent.where(name: "ombudsman.manifestation_received").last.payload.keys).to eq(%w[manifestation_id])
  end

  it "anônimo pela sessão é recusado; cidadão de outro telefone → 403 not_own_citizen" do
    json_post "/citizen/ombudsman/manifestations", citizen_id: citizen.id, kind: "report", identification_mode: "anonymous", body: "x" * 30
    expect(json_body).to include("error" => "invalid_request", "path" => "identification_mode")
    other = web_citizen!(cpf: "11144477735", phone: "+5541911112222")
    json_post "/citizen/ombudsman/manifestations", citizen_id: other.id, kind: "praise", identification_mode: "identified", body: "x" * 30
    expect(response).to have_http_status(:forbidden)
    expect(json_body["error"]).to eq("not_own_citizen")
  end

  it "lista traz os cidadãos da sessão, sem nada do anônimo; o 11º do dia → 429" do
    # dois pares no mesmo telefone → cpf_masked em cada item; anônima de mesmo IP não aparece;
    # MemoryStore: 10 criações e a 11ª → 429 rate_limited.
  end

  it "o texto, o CPF e o nome do arquivo não vão ao log" do
    log = capture_log { json_post "/citizen/ombudsman/manifestations", citizen_id: citizen.id, kind: "complaint", identification_mode: "preserved", body: "Texto sigiloso de teste com mais de vinte" }
    expect(log).not_to include("sigiloso")
  end
end
```

`register_spec.rb`: corpo com 19 e 4001 caracteres → 422 `path: "body"`; unidade inativa → `path: "health_unit_id"`; quatro anexos → `too_many_attachments`; anexo de outra sessão → `path: "attachment_ids"`.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/citizen_api/ombudsman/manifestations_spec.rb spec/commands/ombudsman/register_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os specs + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add citizen ombudsman manifestations with protocol, attachments and timeline`.

**Relato ao coordenador:** "api 33 · Task 8 · `/citizen/ombudsman/manifestations` (lista, detalhe, criação) no formato do §3; 403 `not_own_citizen` (A21)."

---

### Task 9: Denúncia anônima sem login — configuração pública, limites e teto (§4)

**Files:**
- Create: `app/commands/ombudsman/register_anonymous.rb`, `app/controllers/citizen_api/ombudsman/{public_configs,anonymous_reports}_controller.rb`, `app/jobs/ombudsman/{cap_alert_job,mail_job}.rb`, `app/services/ombudsman/recipients.rb`, `app/mailers/ombudsman_mailer.rb`, `app/views/ombudsman_mailer/anonymous_cap.{html,text}.erb`, `spec/requests/citizen_api/ombudsman/anonymous_reports_spec.rb`, `spec/jobs/ombudsman/mail_job_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`ombudsman.anonymous_cap_reached` → `Ombudsman::CapAlertJob`)

**Interfaces:**
- Os dois controladores herdam de `CitizenApi::BaseController` com `allow_anonymous_citizen` (nenhum cookie é lido nem gravado) e `wrap_parameters false`.
- `GET /citizen/ombudsman/public_config` → `{ anonymous_web: Ombudsman::AnonymousGate.usable?, units: [{ id, name }] }` (unidades ativas, por nome — A22); 403 `feature_disabled` sem `ombudsman`.
- `POST /citizen/ombudsman/anonymous_reports { body, health_unit_id? }` → 201 `{ ok: true }`. Ordem: `rate_limit` 3/h e 10/dia por IP (`with: -> { render_error("rate_limited", :too_many_requests) }`) → portão `ombudsman_anonymous_web` (403) → JSON (415 pelo `require_json_for_writes`) → tamanho (`request.content_length` e `request.raw_post.bytesize` > `ANONYMOUS_MAX_BODY_BYTES` → 413 `payload_too_large`) → chaves (`request.request_parameters.keys - ANONYMOUS_KEYS` → 422 com o nome truncado) → `RegisterAnonymous`.
- `Ombudsman::RegisterAnonymous.call(body:, health_unit_id:, now:) -> Result.ok(manifestation:) | fail(:daily_cap_reached) | fail(:invalid_request, details: { path: })` — transação: `OmbudsmanSetting.lock.find(1)` (serializa o teto), conta as `channel: "web", identification_mode: "anonymous"` de hoje local; no teto → `OmbudsmanDailyRun.insert_all([{ run_on: today, kind: "anonymous_cap" }], unique_by: …)` e, se inseriu, `DomainEvents.publish("ombudsman.anonymous_cap_reached")` (payload vazio — A23); `fail(:daily_cap_reached)` com a transação COMMITADA (a marca do dia fica); abaixo do teto → `Register.call(…, channel: "web")` com `kind: "report"`, `identification_mode: "anonymous"`, sem anexo.
- `Ombudsman::CapAlertJob` (`IdempotentConsumer`) → `MailJob.perform_later(city_slug:, kind: "anonymous_cap", ref: today.iso8601)`; `Ombudsman::MailJob` no padrão do `Surveillance::MailJob` (`CityScopedJob`, `ProcessedEvent` consumer `ombudsman_mail`, um e-mail por destinatário, só contagem); `Ombudsman::Recipients.emails(roles)`.

- [ ] **Step 1: Write the failing test** `anonymous_reports_spec.rb`:

```ruby
RSpec.describe "Denúncia anônima sem login", type: :request do
  let!(:city) { ombudsman_city!(anonymous: true) }
  before { allow(Rails).to receive(:cache).and_return(ActiveSupport::Cache::MemoryStore.new) }

  it "grava denúncia anônima sem protocolo na resposta, sem cookie e sem o texto no log" do
    log = capture_log { json_post "/citizen/ombudsman/anonymous_reports", body: "Funcionário cobra para furar a fila do posto." }
    expect(response).to have_http_status(:created)
    expect(json_body).to eq("ok" => true)
    expect(response.headers["Set-Cookie"]).to be_nil
    expect(log).not_to include("furar a fila")
    m = OmbudsmanManifestation.last
    expect([ m.kind, m.identification_mode, m.channel, m.citizen_id ]).to eq([ "report", "anonymous", "web", nil ])
    expect(m.protocol).to be_present
  end

  it "quarta da hora pelo mesmo IP → 429 rate_limited" do
    3.times { json_post "/citizen/ombudsman/anonymous_reports", body: "Denúncia de teste número suficiente." }
    json_post "/citizen/ombudsman/anonymous_reports", body: "Denúncia de teste número suficiente."
    expect(response).to have_http_status(:too_many_requests)
    expect(json_body).to eq("error" => "rate_limited")
  end

  it "teto da cidade: bloqueia e avisa uma vez no dia" do
    OmbudsmanSetting.current.update!(anonymous_daily_cap: 1)
    json_post "/citizen/ombudsman/anonymous_reports", body: "Primeira denúncia do dia, dentro do teto."
    2.times do |i|
      post "/citizen/ombudsman/anonymous_reports", params: { body: "Acima do teto #{i} com texto bastante" }.to_json,
           headers: { "CONTENT_TYPE" => "application/json", "REMOTE_ADDR" => "10.0.0.#{i + 2}" }
      expect(json_body).to eq("error" => "daily_cap_reached")
    end
    expect(DomainEvent.where(name: "ombudsman.anonymous_cap_reached").count).to eq(1)
    expect(DomainEvent.where(name: "ombudsman.anonymous_cap_reached").last.payload).to eq({})
  end

  it "recusas: chave a mais, corpo grande, formulário, interruptor desligado" do
    json_post "/citizen/ombudsman/anonymous_reports", body: "Texto com mais de vinte letras.", kind: "complaint"
    expect(json_body).to include("error" => "invalid_request", "path" => "kind")
    json_post "/citizen/ombudsman/anonymous_reports", body: "a" * 17_000
    expect(response).to have_http_status(:payload_too_large)
    post "/citizen/ombudsman/anonymous_reports", params: { body: "x" * 30 }
    expect(response).to have_http_status(:unsupported_media_type)
    Platform::Features.set!(city: city, key: "ombudsman_anonymous_web", enabled: false, maintainer: ledi_maintainer!)
    get "/citizen/ombudsman/public_config"
    expect(json_body["anonymous_web"]).to be(false)
    json_post "/citizen/ombudsman/anonymous_reports", body: "Texto com mais de vinte letras."
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "ombudsman_anonymous_web")
  end
end
```

Mais um exemplo de corrida (Review Focus 2): duas threads com conexões próprias, teto 2 e 1 já gravada → uma 201 e uma 429; um evento. `mail_job_spec.rb`: o e-mail do teto vai a cada `ombudsman` ativo, só com a contagem, sem protocolo nem texto; duas execuções no mesmo dia → um envio.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/citizen_api/ombudsman/anonymous_reports_spec.rb spec/jobs/ombudsman/mail_job_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os specs + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add public anonymous ombudsman report with ip limits and city daily cap`.

**Relato ao coordenador:** "api 33 · Task 9 · formulário público: `public_config`, `anonymous_reports` (201 `{ok:true}`, 413/415/422/429), teto com e-mail ao ombudsman; unidades com CAPS inclusive (A22)."

---

### Task 10: Registro pela equipe, busca do cidadão e comprovante (§5)

**Files:**
- Create: `app/controllers/ombudsman/{person_lookups,manifestations}_controller.rb` (só `create` e `receipt` nesta task), `spec/requests/ombudsman/staff_registration_spec.rb`
- Modify: `app/commands/ombudsman/register.rb`, `config/routes.rb`

**Interfaces:**
- `POST /ombudsman/person_lookup { cpf }` (staff; 30/min por usuário; só corpo; chave fora de `cpf` → 422) → `{ citizens: [{ id, name, cpf_masked, phone_masked }] }` de `Citizen.not_erased.where(cpf:)` por `created_at` (lista vazia quando ninguém — o atendente segue com `manifestant_*`); CPF malformado → 422 `invalid_cpf` `path: "cpf"`. Sem trilha (não é prontuário, §13).
- `POST /ombudsman/manifestations` (staff) → 201 `{ id, protocol, due_on }`: `channel` em `STAFF_CHANNELS`; `ouvidorsus_number` obrigatório (1..40) só em `ouvidorsus`; `identified`/`preserved` exigem `citizen_id` (par não excluído) ou `manifestant_name` (A24); `manifestant_cpf` validado por `CitizenIdentity::Cpf.normalize`; `anonymous` só com `report` e sem nenhum dado de pessoa (422 `path` do primeiro campo); anexos do usuário.
- `GET /ombudsman/manifestations/:id/receipt` (staff) → `{ protocol, kind, received_at, due_on, channel }`; anônima → 404.

- [ ] **Step 1: Write the failing test** `staff_registration_spec.rb`: presencial identificado com cidadão achado pelo CPF (a lista traz o par declarado também); por telefone sem cidadão com nome e contato; OuvidorSUS sem número → 422 `path: "ouvidorsus_number"`; anônimo com `manifestant_name` → 422; elogio anônimo → 422 `path: "kind"`; comprovante com o prazo; comprovante da anônima → 404; responder sem `ombudsman_staff` → 403 em todas; o CPF não vai ao log.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/staff_registration_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/commands/ombudsman/register_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add staff ombudsman registration with citizen lookup and receipt`.

**Relato ao coordenador:** "api 33 · Task 10 · `POST /ombudsman/person_lookup {cpf}` → `{citizens:[…]}` (busca nova sobre pares, A8), registro pela equipe e comprovante — formatos para o dashboard."

---

### Task 11: Caixa e detalhe da equipe, identidade preservada e consentimento (§5, §8)

**Files:**
- Create: `app/commands/ombudsman/{view_identity,record_identity_consent}.rb`, `app/controllers/ombudsman/identities_controller.rb`, `spec/requests/ombudsman/inbox_list_spec.rb`, `spec/requests/ombudsman/identity_spec.rb`
- Modify: `app/controllers/ombudsman/manifestations_controller.rb` (`index`, `show`), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings (`ombudsman.identity_viewed`, `to: []`)

**Interfaces:**
- `GET /ombudsman/manifestations?state[]&color[]&kind[]&channel[]&subject_id&health_unit_id&q&before` (staff) → `{ items: [<list_item>], next_before }` (A25); `color[]` pelo `Deadlines.color_sql`; `subject_id` casa o assunto e os filhos; `q` só protocolo exato ou prefixo `AAAA-`; ordem `received_at DESC, id DESC`, página de 50; valor fora do catálogo → 422 `path`.
- `GET /ombudsman/manifestations/:id` (staff) → dados, `body`, `events` (todos), `forwardings` (com `extract`, `reply`, `due_on`, `color`), `attachments` (`{ id, filename, uploaded_by }`), `reopen_request` (pendente), `identity` (Desvio do §8), `subject`, `health_unit`, `professional`, `due_on`, `color`, `can` (ações permitidas ao usuário no estado — para o dashboard não adivinhar).
- `POST /ombudsman/manifestations/:id/identity { justification }` (só ombudsman; técnico → 403 `missing_role`) → `{ name, cpf, phone, manifestant_name, manifestant_contact, manifestant_cpf }`; justificativa fora de `JUSTIFICATION` → 422; manifestação sem nenhum dado de pessoa (anônima) → 422 `no_identity` sem gravar (A26); grava `ombudsman_identity_accesses(scope: "identity")` e `ombudsman.identity_viewed { manifestation_id, access_id }`. Resposta com `Cache-Control: no-store`.
- `POST /ombudsman/manifestations/:id/identity_consent { granted, text }` (ombudsman) → 201; evento `identity_consent_recorded` (`data: { granted }`, `text` cifrado, não visível ao cidadão); não revela nada.

- [ ] **Step 1: Write the failing tests.** `inbox_list_spec.rb`: filtros por estado, cor (uma vermelha, uma amarela, uma pausada), tipo, canal, assunto pai, protocolo; paginação com `next_before`; `invalid_cursor`. `identity_spec.rb` (Review Focus 5): para uma `preserved` de cidadão do wpda, lista e detalhe pelo técnico e pelo ombudsman têm `identity: "preserved"` e nenhum nome/CPF/telefone em lugar nenhum do JSON (`expect(response.body).not_to include(citizen.cpf, citizen.full_name)`); a `identified` traz `cpf_masked`; o técnico em `/identity` → 403; o ombudsman sem justificativa → 422, com → dados e uma linha nova por chamada; a anônima → 422 `no_identity`; o evento só com ids; a justificativa não vai ao log.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/inbox_list_spec.rb spec/requests/ombudsman/identity_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os specs → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman staff inbox, detail and preserved identity with access trail`.

**Relato ao coordenador:** "api 33 · Task 11 · lista com `{items,next_before}` (A25), detalhe com `can`, `/identity` só ombudsman com trilha (anônima → 422 no_identity, A26)."

---

### Task 12: Classificar, encaminhar, cancelar e o ponto de resposta (§2, §6)

**Files:**
- Create: `app/commands/ombudsman/{classify,forward,cancel_forwarding,reply_forwarding,decline_forwarding,extend_forwarding}.rb`, `app/controllers/ombudsman/{actions,forwardings,inbox}_controller.rb`, `spec/requests/ombudsman/forwarding_spec.rb`, `spec/requests/ombudsman/concurrency_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb` + bindings (`ombudsman.forwarded`, `ombudsman.forwarding_replied`, `to: []`)

**Interfaces:**
- `POST /ombudsman/manifestations/:id/classify { expected_state, subject_id, kind?, health_unit_id?, professional_id?, urgent? }` (staff): `subject_id` ativo e folha (assunto com filho → 422 `subject_not_leaf`); muda `kind` (anônima continua `report`; outro → 422); uma linha de evento `classified` com `data: { changes: { campo: [antes, depois] } }` (só ids/rótulos); `received → in_analysis`.
- `POST …/forward { expected_state, department_id | health_unit_id, extract }` (staff): destino ativo; sem respondente ativo → 422 `destination_without_responder`; `extract` em `EXTRACT`; `due_on = hoje + forwarding_days`; estado `forwarded`; evento `forwarded` (`data: { forwarding_id }`); `ombudsman.forwarded { manifestation_id, forwarding_id }`.
- `POST /ombudsman/forwardings/:id/cancel { reason? }` (staff): só `pending` (senão 409 `forwarding_not_pending`); sem outro `pending` → manifestação `in_analysis`; evento `forwarding_cancelled`.
- `GET /ombudsman/inbox?state` (respondente) → `{ items: [{ id, destination, extract, due_on, color, state, created_at, reply? }] }` só dos destinos ativos dele; nunca protocolo, identidade, `body`, assunto ou anexos.
- `POST /ombudsman/inbox/:id/reply { body }`, `/decline { reason }`, `/extend { reason }` (respondente do destino; outro → 404): `reply`/`decline` só `pending` (409); o último `pending` resolvido devolve a manifestação a `in_analysis`, sob `Lock.manifestation!`; `extend` uma vez (`already_extended`); eventos `forwarding_replied|forwarding_declined|forwarding_extended` (`actor: staff`, o respondente); `ombudsman.forwarding_replied { manifestation_id, forwarding_id }` no `reply`.
- `Ombudsman::ActionsController` recebe `POST /ombudsman/manifestations/:id/:action` para todas as ações do §2 (rota com `constraints: { action_name: /classify|forward|…/ }`), uma ação por método, cada uma chamando o comando; o comando sempre começa por `Lock.manifestation!` + `Lock.check`.

- [ ] **Step 1: Write the failing tests.** `forwarding_spec.rb`: classificar e reclassificar (dois eventos com antes/depois); encaminhar a um setor sem respondente → 422; a um setor com respondente → `forwarded`; dois encaminhamentos paralelos; o respondente do setor vê só o dele e o JSON tem exatamente as chaves do §6; o respondente de outra unidade → 404; resposta do primeiro deixa `forwarded`, a do segundo leva a `in_analysis`; recusa grava o motivo; prorrogação dupla → 422; o respondente sem `ombudsman_staff` em `GET /ombudsman/manifestations` → 403. `concurrency_spec.rb` (Review Focus 1): duas threads, `final_response` e `forward` com `expected_state: "in_analysis"`: uma 200, a outra 409 `manifestation_changed`; contagem de eventos +1.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/forwarding_spec.rb spec/requests/ombudsman/concurrency_spec.rb` → FAIL (o exemplo de concorrência fica pendente até a Task 13 trazer o `final_response`; marcar com `skip` e retirar lá).
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** `forwarding_spec.rb` + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman classification, forwarding by extract and responder inbox`.

**Relato ao coordenador:** "api 33 · Task 12 · classify/forward/cancel e `/ombudsman/inbox` (reply/decline/extend); motivo da recusa em `reply` (A11)."

---

### Task 13: Complemento, prorrogação, respostas, resolutividade (§2, §9)

**Files:**
- Create: `app/commands/ombudsman/{request_complement,record_complement,extend,interim_response,final_response,change_resolution}.rb`, `spec/requests/ombudsman/responses_spec.rb`
- Modify: `app/controllers/ombudsman/actions_controller.rb`, `config/initializers/domain_events.rb` + bindings (`ombudsman.concluded`, `to: []`)

**Interfaces:**
- `request_complement { expected_state, text }`: `in_analysis` → `awaiting_complement`; `complement_used` → 422 `complement_already_used`; anônima → 422 `complement_not_allowed`; grava `complement_requested_at = now`, `complement_used = true`; evento visível; `Notify(kind: "complement_requested")`.
- `complement { expected_state, text, attachment_ids? }` (staff registra o recebido por outro canal) e o do cidadão (Task 15) chamam `RecordComplement.call(manifestation:, text:, attachment_ids:, actor:, by:, owner:)`: `awaiting_complement` → `in_analysis`, `suspended_days += today - complement_requested_at.in_time_zone.to_date`, `complement_requested_at = nil` (A27), evento `complement_received` visível, anexos com `event_id`.
- `extend { expected_state, text }` (ombudsman): estados `in_analysis|forwarded|awaiting_complement`; `extended` → 422 `already_extended`; `extension_reason` cifrado; evento `extended` visível com o texto; `Notify(kind: "interim_response")`.
- `interim_response { expected_state, text }` (staff): qualquer aberto; anônima → 422 `not_allowed_anonymous`; evento visível; `Notify(kind: "interim_response")`.
- `final_response { expected_state, text, resolution? }` (staff): `in_analysis` → `answered`; `resolution` obrigatória em `RESOLUTION_KINDS`, proibida nos outros (422 `path: "resolution"`); com encaminhamento `pending` → 422 `forwarding_pending`; `concluded_at = now`; anônima: evento `visible_to_citizen = false`; `Notify(kind: "final_response")`; `ombudsman.concluded { manifestation_id, outcome: "answered" }`.
- `change_resolution { expected_state: "answered", resolution, text }` (ombudsman): evento `resolution_changed` (não visível).

- [ ] **Step 1: Write the failing test** `responses_spec.rb`: complemento pedido → aviso ao cidadão, prazo `paused`; segundo pedido no mesmo ciclo → 422; registro pela equipe 6 dias depois (`travel_to`) → `suspended_days == 6` e `due_on` +6; prorrogação → +30, aviso `interim_response`, segunda → 422; resposta intermediária não muda o prazo; conclusiva de reclamação sem `resolution` → 422, de elogio com `resolution` → 422, com encaminhamento pendente → 422 `forwarding_pending`; conclusiva → aviso e `ombudsman.concluded` com `outcome`; anônima: conclusiva interna, sem aviso; `change_resolution` pelo técnico → 403. Retirar o `skip` do `concurrency_spec.rb`.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/responses_spec.rb spec/requests/ombudsman/concurrency_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os specs + `spec/services/ombudsman/deadlines_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman complement, extension, interim and final responses`.

**Relato ao coordenador:** "api 33 · Task 13 · complemento suspende e soma os dias corridos locais, prorrogação vira aviso `interim_response`, conclusiva com resolutividade; textos 10..4000 (A10)."

---

### Task 14: Arquivamento, reabertura e pedido de reabertura (§2)

**Files:**
- Create: `app/commands/ombudsman/{archive,reopen,request_reopen,decide_reopen}.rb`, `app/controllers/ombudsman/reopen_requests_controller.rb`, `spec/requests/ombudsman/archive_reopen_spec.rb`
- Modify: `app/controllers/ombudsman/actions_controller.rb`, `config/routes.rb`, `config/initializers/domain_events.rb` + bindings (`ombudsman.reopened`, `to: []`)

**Interfaces:**
- `archive { expected_state, reason, duplicate_of_protocol? | duplicate_of_id?, external_body_name? }` (ombudsman; o job chama com `actor: system`): estados `received|in_analysis|awaiting_complement`; `duplicate` exige outra manifestação (por id; a tela pode mandar o protocolo, resolvido no controlador — A28); `forwarded_elsewhere` exige `external_body_name` (1..120); `concluded_at = now`; evento `archived` visível com `data: { reason }`; `Notify(kind: "archived")`; `ombudsman.concluded { manifestation_id, outcome: "archived" }`.
- `reopen { expected_state, text? }` (ombudsman): `answered|archived` → `in_analysis`; `cycle += 1`, `cycle_started_on = hoje`, zera `extended`, `extension_reason`, `complement_used`, `complement_requested_at`, `suspended_days`, `resolution`, `archive_reason`, `duplicate_of_id`, `external_body_name`, `concluded_at`; evento `reopened` visível; `ombudsman.reopened { manifestation_id }`.
- `Ombudsman::RequestReopen.call(manifestation:, reason:, now:)` (cidadão, Task 15): final, `concluded_at` há no máximo 30 dias (senão 422 `reopen_window_closed`), sem pedido no ciclo (409 `reopen_pending` ou 422 `reopen_already_requested`), `reason` em `REOPEN_REASON`; evento `reopen_requested` visível.
- `POST /ombudsman/reopen_requests/:id/decide { decision: accept | reject, text }` (ombudsman): só `pending` (409 `already_decided`); `accept` → `Reopen` na mesma transação; `reject` → evento `reopen_rejected` visível com o texto; nos dois, `Notify(kind: "reopen_decided")`.

- [ ] **Step 1: Write the failing test** `archive_reopen_spec.rb`: arquivar por duplicidade sem a original → 422; com → `archived`; por outro órgão sem nome → 422; arquivar `forwarded` (com `expected_state: "forwarded"`) → 422 `invalid_transition` (estado fora da tabela do §2); reabrir de ofício → ciclo 2, prazo novo de 30 dias contando de hoje, `extended` e `complement_used` zerados (pode pedir complemento de novo); pedido do cidadão no dia 31 → `reopen_window_closed`; aceitar → ciclo novo e aviso `reopen_decided`; recusar → texto na linha do tempo do cidadão; segundo pedido no mesmo ciclo → recusado.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/ombudsman/archive_reopen_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (erro de estado fora da tabela do §2 com o `expected_state` certo → 422 `invalid_transition`).
- [ ] **Step 4:** o spec + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman archiving, reopening and citizen reopen requests`.

**Relato ao coordenador:** "api 33 · Task 14 · arquivar/reabrir e decisão do pedido; `invalid_transition` 422 para ação fora da tabela; duplicidade pelo id (A28)."

---

### Task 15: Cidadão — complemento, pedido de reabertura e pesquisa de satisfação (§3, §10)

**Files:**
- Create: `app/commands/ombudsman/record_satisfaction.rb`, `spec/requests/citizen_api/ombudsman/followup_spec.rb`
- Modify: `app/controllers/citizen_api/ombudsman/manifestations_controller.rb`, `config/routes.rb`

**Interfaces:**
- `POST /citizen/ombudsman/manifestations/:id/complement { body, attachment_ids? }` → 200 (só `awaiting_complement`, senão 409 `manifestation_changed`; anexos da sessão, até 3).
- `POST …/reopen_request { reason }` → 201.
- `POST …/satisfaction { met, clear, score }` → 201; `answered`, `citizen_id` presente, até 30 dias de `concluded_at` (422 `satisfaction_window_closed`), uma vez (409 `already_answered`; o índice único é a última palavra); sem texto livre (chave a mais → 422).
- Os três: manifestação de cidadão fora da sessão → 404; o cidadão age pelo `Lock.manifestation!` e `RecordComplement`/`RequestReopen` com `actor: "citizen"`.

- [ ] **Step 1: Write the failing test** `followup_spec.rb`: o cidadão responde o complemento com anexo (volta a `in_analysis`, dias suspensos contados); responde fora do estado → 409; pede reabertura; responde a pesquisa uma vez, a segunda → 409; no dia 31 → 422; `can_*` do detalhe acompanham cada passo.
- [ ] **Step 2:** `bin/test api -w mod33 spec/requests/citizen_api/ombudsman/followup_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/requests/citizen_api/ombudsman/manifestations_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add citizen ombudsman complement, reopen request and satisfaction survey`.

**Relato ao coordenador:** "api 33 · Task 15 · complemento, pedido de reabertura e pesquisa pelo wpda; `can_*` no detalhe."

---

### Task 16: Painel e relatório de gestão anual (§11)

**Files:**
- Create: `app/services/ombudsman/panel.rb`, `app/commands/ombudsman/{save_annual_report,publish_annual_report}.rb`, `app/controllers/ombudsman/{dashboard,annual_reports}_controller.rb`, `spec/services/ombudsman/panel_spec.rb`, `spec/requests/ombudsman/annual_reports_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `Ombudsman::Panel.call(from:, to:, today:) -> Hash`: universo = manifestações com `received_at` no período; `{ period: { from, to }, total, by_kind: {k => cell}, by_channel, by_state, by_subject: [{ id, name, count }] (assunto de 1º nível; sem assunto em `unclassified`), by_health_unit: [{ id, name, count }], on_time_rate, avg_days_to_conclude, resolution_rate, satisfaction: { responses, met: {yes, partly, no}, clear: {yes, no}, avg_score } }`; contagens por `Analytics::Suppression.cell`; `total` de cada grupo por `group`; taxas por `rate`/`group_rate`; médias (`avg_days_to_conclude`, `avg_score`) `SUPPRESSED` quando o número de casos está em 1..4, `nil` sem caso; `on_time_rate` = concluídas no período com `concluded_at` local ≤ `due_on` do ciclo vigente (A19).
- `GET /ombudsman/dashboard?from=&to=` (staff; `ReportPeriod`; padrão: ano corrente) e `GET /ombudsman/dashboard/professionals?from=&to=` (ombudsman) → `{ items: [{ professional: { id, name }, by_kind: {…}, total }] }` sem supressão.
- `GET /ombudsman/annual_reports/:year` (ombudsman) → `{ year, state, recurring_problems, actions_taken, notes, numbers, published_at }` (`numbers` = o `snapshot` publicado, ou o `Panel` vivo do ano no rascunho; ano sem linha → rascunho vazio, sem gravar); `PATCH` grava o rascunho (cada texto até 8000); `POST …/publish` → `snapshot = Panel.call(ano inteiro)`, `state = published`; ano corrente ou futuro → 422 `year_not_closed` (A29); publicado → 409 `already_published`.

- [ ] **Step 1: Write the failing tests.** `panel_spec.rb`: 7 reclamações e 2 elogios → `complaint: 7`, `praise: SUPPRESSED`, `total` do `by_kind` `SUPPRESSED`; taxa no prazo com 6 de 8; média do escore com 3 respostas → `SUPPRESSED`; sem casos → `nil`; o painel não traz nenhum texto. `annual_reports_spec.rb`: rascunho, edição, publicação congela os números (uma manifestação nova do ano depois não muda o `numbers`); PATCH depois de publicado → 409; ano corrente → 422; técnico → 403; o painel por profissional só pelo ombudsman.
- [ ] **Step 2:** `bin/test api -w mod33 spec/services/ombudsman/panel_spec.rb spec/requests/ombudsman/annual_reports_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (consultas agregadas em SQL, o prazo pelo `Deadlines.due_on_sql`).
- [ ] **Step 4:** os specs → PASS.
- [ ] **Step 5: Commit** `feat: add ombudsman dashboard with small-count suppression and annual management report`.

**Relato ao coordenador:** "api 33 · Task 16 · `/ombudsman/dashboard` (formato fixado aqui, A30), por profissional, relatório anual com snapshot; publicar só ano encerrado (A29)."

---

### Task 17: Job diário — complemento vencido e resumo por e-mail (§12)

**Files:**
- Create: `app/jobs/ombudsman/deadlines_job.rb`, `app/views/ombudsman_mailer/{daily_summary,responder_summary}.{html,text}.erb`, `spec/jobs/ombudsman/deadlines_job_spec.rb`
- Modify: `app/mailers/ombudsman_mailer.rb`, `app/jobs/ombudsman/mail_job.rb`, `config/recurring.yml` (`ombudsman_deadlines`, `Ombudsman::DeadlinesJob`, `housekeeping`, `"every day at 7am America/Sao_Paulo"`, com o comentário do padrão)

**Interfaces:**
- `Ombudsman::DeadlinesJob` (`prepend EachCityJob`; `return unless Gate.usable?(Current.city)` fora da transação): (1) `OmbudsmanDailyRun.insert_all([{ run_on: today, kind: "deadlines" }])` — já existe → termina (idempotente no dia); (2) para cada manifestação `awaiting_complement` com `complement_requested_at::date + complement_days < today`, `Archive.call(reason: "no_complement", actor: "system")` com `expected_state: "awaiting_complement"` (quem voltou não casa), falha de uma não para as outras (log só com id e classe); (3) `MailJob.perform_later(kind: "daily_summary", ref: today)` com amarelas e vermelhas abertas (sem `paused`) a `ombudsman` e `ombudsman_staff`, e `kind: "responder_summary"` com as contagens por respondente dos `pending` dele; nada quando tudo é zero; e-mail só com números.

- [ ] **Step 1: Write the failing test** `deadlines_job_spec.rb`: com complemento pedido há 21 dias → arquivada `no_complement`, evento `system`, aviso ao cidadão, `ombudsman.concluded`; há 20 dias → intacta; a que respondeu no último dia → intacta (Review Focus 4); rodar duas vezes no dia → um arquivamento e um e-mail; respondente recebe só a contagem dos encaminhamentos dele; interruptor desligado → nada; a chave `ombudsman_deadlines` do `recurring.yml` aponta para a classe às 7h.
- [ ] **Step 2:** `bin/test api -w mod33 spec/jobs/ombudsman/deadlines_job_spec.rb spec/jobs/ombudsman/mail_job_spec.rb spec/config/solid_queue_configuration_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add daily ombudsman job for expired complements and deadline summaries`.

**Relato ao coordenador:** "api 33 · Task 17 · `Ombudsman::DeadlinesJob` às 7h, idempotente por `ombudsman_daily_runs` (A13), e-mails só com contagens."

---

### Task 18: Exclusão LGPD (ADR 0026; §13)

**Files:**
- Create: `spec/commands/citizens/ombudsman_erasure_spec.rb`
- Modify: `app/commands/citizens/erase.rb`, `app/commands/citizens/request_erasure.rb` (só o comentário do topo)

**Interfaces:**
- `Citizens::Erase#erase_pair`: antes do `citizen.update_columns(…)` (enquanto CPF, nome e telefone ainda estão no par), `Ombudsman::Erasure.unlink!(citizen)`: `OmbudsmanManifestation.where(citizen_id: citizen.id).order(:id).lock` (o par já está travado `FOR UPDATE` — a ordem cidadão → manifestação do Desvio 4), `update_columns(citizen_id: nil, manifestant_name: citizen.full_name, manifestant_cpf: citizen.cpf, manifestant_contact: citizen.phone, updated_at:)` linha a linha (o trigger aceita só nessa forma); `OmbudsmanNotice.where(citizen_id: citizen.id).delete_all`; a satisfação fica. Comentário no topo: "ADR 0051: e as manifestações da ouvidoria desligadas do par, com a identidade copiada para a manifestação (registro de obrigação legal retido); os avisos da ouvidoria apagados."
- `RequestErasure.retained?` **não muda** (Desvio 7; A7): a manifestação não retém o CPF.

- [ ] **Step 1: Write the failing test** `ombudsman_erasure_spec.rb`: cidadão só com manifestação (sem atendimento) pede exclusão → `pending` → confirmada → a manifestação tem os três `manifestant_*` iguais aos dados antigos, `citizen_id` nulo, sem aviso, a satisfação existe; o detalhe da equipe de uma `identified` passa a mostrar `name` e `cpf_masked` dos `manifestant_*`; de uma `preserved` segue `"preserved"`; o cidadão excluído não acha mais nada em `/citizen/ombudsman/manifestations` (a sessão é apagada). Concorrência (Review Focus 3): uma thread no `Ombudsman::InterimResponse` (que trava cidadão → manifestação e grava aviso) e outra no `Citizens::Erase`: as duas terminam, sem `deadlock detected`, e o aviso ou foi apagado pela exclusão ou não nasceu.
- [ ] **Step 2:** `bin/test api -w mod33 spec/commands/citizens/ombudsman_erasure_spec.rb spec/commands/citizens/erase_spec.rb spec/commands/citizens/request_erasure_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`app/services/ombudsman/erasure.rb`, chamado do `erase_pair`).
- [ ] **Step 4:** os mesmos specs + `spec/commands/citizens/{erase_transport_spec,erase_psychosocial_spec,erase_remind_concurrency_spec}.rb` → PASS.
- [ ] **Step 5: Commit** `feat: unlink ombudsman manifestations on citizen erasure and keep the identity in the record`.

**Relato ao coordenador:** "api 33 · Task 18 · exclusão desliga e copia a identidade para a manifestação, sem reter o CPF (A7 — decisão do coordenador pendente se o contrato insistir em `retained`)."

---

### Task 19: Semente de dev realista

**Files:**
- Create: `lib/ombudsman_crew.rb`, `spec/lib/ombudsman_crew_spec.rb`
- Modify: `db/seeds.rb` (depois do `UrgencyCrew`, com o comentário do padrão)

**Interfaces:**
- `OmbudsmanCrew.seed_current_city(slug:, ddd:, password:) -> Hash` e `OmbudsmanCrew.summary_lines(result, password:)`; só em Curitiba; idempotente (acha pelo que gravou); tudo pelos comandos do app:
  - interruptores `ombudsman` e `ombudsman_anonymous_web` ligados pelo mantenedor de dev; prazos padrão; assuntos semeados;
  - equipe com TOTP fixo: `ouvidor@curitiba.dev` (`ombudsman`), `ouvidoria@curitiba.dev` (`ombudsman_staff`), `farmacia.resp@curitiba.dev` (`ombudsman_responder` no setor "Assistência Farmacêutica") e `ubs.resp@curitiba.dev` (respondente da "UBS Centro" do `ProfessionalCrew`);
  - setores "Assistência Farmacêutica", "Regulação", "Atenção Primária", "Vigilância Sanitária";
  - cenário com cidadãos do wpda já semeados (CPF fictício válido por `cpf_for`): uma reclamação identificada encaminhada à farmácia (amarela), uma solicitação com identidade preservada aguardando complemento, um elogio respondido com pesquisa de satisfação, uma denúncia anônima web recebida, uma reclamação por telefone sem cidadão (vermelha, `received_at` 35 dias atrás), uma arquivada por duplicidade; e o relatório de 2025 em rascunho.
- Resumo: só e-mail, papel e otpauth; nunca protocolo, texto ou nome.

- [ ] **Step 1: Write the failing test** `ombudsman_crew_spec.rb`: duas execuções não duplicam manifestações, setores nem respondentes; há uma de cada cor (verde, amarela, vermelha, pausada); o resumo não contém nenhum nome de cidadão nem texto de manifestação.
- [ ] **Step 2:** `bin/test api -w mod33 spec/lib/ombudsman_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (os comandos aceitam `now:` para escalonar as datas).
- [ ] **Step 4:** o spec + `spec/lib/urgency_crew_spec.rb spec/lib/professional_crew_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: seed a realistic dev ombudsman with staff, responders and manifestations`.

**Relato ao coordenador:** "api 33 · Task 19 · semente: logins ouvidor/ouvidoria/2 respondentes, 6 manifestações de cores e estados diferentes."

---

### Task 20: Specs de sigilo, guarda de eventos e suíte completa

**Files:**
- Create: `spec/security/ombudsman_secrecy_spec.rb`
- Modify: `spec/initializers/domain_events_bindings_spec.rb` (conferência final)

- [ ] **Step 1: Write the test** `spec/security/ombudsman_secrecy_spec.rb` (cenário completo: registro web identificado e preservado com anexo, anônimo, pela equipe com CPF; classificação com assunto e unidade; encaminhamento com extrato; resposta do ponto; complemento; prorrogação; conclusiva; identidade aberta; satisfação):
  - os nomes de `DomainEvent` com prefixo `ombudsman.` publicados no cenário ⊂ `Ombudsman::Catalog::DOMAIN_EVENTS`, e `DOMAIN_EVENTS` é exatamente a lista de 7 do contrato; uma varredura (Prism, o padrão de `spec/events/platform_event_payload_guard_spec.rb`) de `app/**/*.rb` acha `DomainEvents.publish("ombudsman.…"` só com esses nomes;
  - todo payload tem só as chaves `manifestation_id`, `forwarding_id`, `access_id` e `outcome` (este em `answered|archived`), e nenhum contém o texto, o extrato, a resposta, a justificativa, o nome, o CPF, o telefone, o nome do assunto, o id do assunto, o nome ou o id da unidade;
  - `capture_log` de todas as requisições do cenário não contém o texto, o extrato, a resposta, a justificativa, o nome, o CPF, o telefone, o nome do arquivo nem o token do link;
  - toda recusa 4xx forçada (texto curto, CPF malformado, justificativa curta, chave a mais) traz só `error`/`path`/`problems`, sem o valor enviado;
  - nenhuma rota de `/ombudsman` e `/citizen/ombudsman` aceita texto ou CPF na query (`query_only!`: `GET /ombudsman/manifestations?cpf=…` → 422 `path: "cpf"`);
  - em `preserved`, nenhuma resposta da equipe (lista, detalhe, comprovante, inbox, painel, por profissional) contém o nome, o CPF ou o telefone; na inbox, nem o protocolo nem o `body`;
  - o aviso ao cidadão e o SMS não têm texto além de `SMS_TEXT`;
  - `grep` multilinha de `Platform.audit(` nos arquivos novos = 0 e `R18_PLATFORM_EVENT_NAMES` sem mudança (R18 não se aplica).
- [ ] **Step 2:** `bin/test api -w mod33 spec/security/ombudsman_secrecy_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture` → PASS (se falhar, a correção vai à task dona e volta aqui).
- [ ] **Step 3: Commit** `test: pin ombudsman secrecy for events, logs, errors and preserved identity`.
- [ ] **Step 4: Suíte completa** (transversal: `memberships`, `exam_files`, `NoticesController`, `Citizens::Erase`, `CityEncryption`, `Platform::Features`, initializers, `spec/support`): pedir à sessão "Testes" por `SendMessage`, no formato `api · mod33/feat/mod-33-ombudsman · --full · módulo 33 (memberships, exam_files, caixa de avisos do cidadão, Citizens::Erase, CityEncryption, Platform::Features, filtro de log, spec/support)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador ("Próximos módulos") com o resumo e o hash, e a sessão do dashboard de que o api está pronto.
- [ ] **Step 5:** merge, push e cards só com autorização do usuário; F-33.x seguem In Progress até o último app de cada card.

**Relato ao coordenador:** "api 33 · Task 20 · sigilo fixado (7 eventos, payload só ids); suíte completa <verde N/0 | falhas>; hash <…>; pronto para o dashboard e o wpda."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptores (Task 1), papéis (2), tabelas (2, 3), sequence (3), tabelas existentes (2), eventos (8, 9, 11–14; guarda 20); §2 estados e ações (12–14), prazos (4, 13), reabertura (14, 15); §3 cidadão (6, 8, 15), limites (6, 8), avisos (7); §4 público (9); §5 equipe (10, 11, 12–14, 6); §6 ponto de resposta (12); §7 configuração (5); §8 identidade (11, 6); §9 avisos e SMS (7, 13, 14); §10 satisfação (15); §11 painel e relatório (16); §12 job (17); §13 cifra e imutabilidade (3), LGPD (18), logs (1, 20), trilha (6, 11); §15 suíte completa (20).
2. **Spec §6 "Testes" do api:** protocolo (3), três modos e identidade invisível (11, 20), trilha (11, 6), anônimo com limites e sem log (9), registro pela equipe e comprovante (10), máquina de estados (12–14), prazos e cores (4, 13, 17), extrato e respondente (12), avisos sem conteúdo e SMS pelo gate (7), satisfação (15), painel e relatório (16), anexos (6), exclusão (18), eventos e interruptores (1, 20).
3. **Nomes:** `Ombudsman::{Register,RegisterAnonymous,Classify,Forward,…,PublishAnnualReport}`, `Ombudsman::{Lock,Deadlines,Access,Attachments,Files,Notify,Panel,Erasure}`, `Ombudsman::{DeadlinesJob,CapAlertJob,MailJob}` — usados com a mesma assinatura nas tasks seguintes.
4. **Ordem das migrações:** a 900002 depende da 900001 (papéis no `ombudsman_responders` só por regra de app; `departments` como FK).

## Divergências propostas ao contrato

A1. **Status do portão.** O §1 diz "404 `feature_disabled`, como os outros gates", mas todos os portões respondem 403 (`app/controllers/concerns/psychosocial_gate.rb:9`, `app/controllers/concerns/feature_gate.rb:18`, `app/controllers/citizen_api/trips_controller.rb:28`). O plano usa 403 em toda rota do 33, inclusive na pública. Proposta: 403 no contrato (§1, §4). Se o coordenador quiser 404 só na pública, muda uma linha do `require_ombudsman_anonymous!`.

A2. **`ombudsman_staff` não é privilegiado.** O comentário de `PRIVILEGED_ROLES` (`app/models/membership.rb:12-38`) diz que todo papel que vê nome do paciente com dado de saúde é privilegiado (o mantenedor não concede, step-up para conceder), e o técnico vê a identidade no modo `identified` junto com o texto da manifestação. Proposta: `ombudsman_staff` também em `PRIVILEGED_ROLES`. O plano segue o contrato (só `ombudsman`).

A3. **Dono do anexo do cidadão.** `exam_files` só conhece `uploaded_by_user_id` e `exam_provider_id` (`db/city_schema.rb:1276-1291`), e `Exams::Files.claim!` exige um dos dois (`app/services/exams/files.rb:39`). Uma FK para `citizen_sessions` quebraria o `PurgeCitizenChannelJob` (`app/jobs/purge_citizen_channel_job.rb:33`), e `rota_exam_file_guard` não deixa anular a coluna. Proposta: o dono fica em `ombudsman_attachments` (`citizen_session_id` sem FK ou `uploaded_by_user_id`), `exam_files` só ganha a finalidade, e a ligação não usa `claim!`.

A4. **Nome do arquivo.** O §3 devolve `attachments: [{ id, filename }]`, mas `exam_files` não tem nome. O TFD guarda o nome na tabela de domínio, cifrado (`tfd_accountability.filename`, `db/city_triggers.sql`, `rota_reencryption_only('filename')`). Proposta: `ombudsman_attachments.filename` cifrado (1..120). A linha nasce no upload, ainda órfã, e o órfão some em cascata com o `ExamFile` órfão do `Exams::DailyJob` (`app/jobs/exams/daily_job.rb:16`). Isso concretiza o "`ombudsman_attachments` órfão" do §13.

A5. **Padrão do prazo em SQL.** O §2 cita `CareRegulation::Deadlines` como padrão de "SQL com espelho em Ruby". Ele só tem Ruby (`app/services/care_regulation/deadlines.rb`). O par SQL+Ruby que existe é `RegulationRequest.overdue_sql`/`#overdue?` (`app/models/regulation_request.rb:28-42`). O plano segue esse par, com spec que compara os dois. Basta corrigir o texto do contrato.

A6. **Tipos de evento da linha do tempo.** O §1 diz "`kind` (§2)", mas o §2 não lista os tipos. Proposta: a lista do Desvio 5 e o subconjunto visível ao cidadão. Inclui `reopen_requested`, `reopen_rejected`, `resolution_changed` e `identity_consent_recorded`, que o §2 e o §5 pedem.

A7. **Exclusão LGPD contraditória.** O §13 pede duas coisas que não cabem juntas:
   - que o `RequestErasure` marque a manifestação como retida (`ombudsman_record`);
   - que o `Erase` copie a identidade e anule `citizen_id`.

   Hoje, pedido retido nasce `retained` (`request_erasure.rb:32`) e o `Erase` sai sem apagar nada (`erase.rb:104-108`). Além disso, não existe campo de motivo de retenção, só o booleano `retained?`. Proposta: a manifestação não retém o CPF (reclamar não pode impedir a pessoa de sair do canal). O `Erase` desliga e copia, e a manifestação fica, com a identidade dentro dela, como registro de obrigação legal (a guarda da Portaria CGU 116 segue na pendência da purga). O plano implementa assim.

A8. **Busca do cidadão no registro pela equipe.** A busca de paciente existente (`Patients::LookupByIdentifier`, `app/commands/patients/lookup_by_identifier.rb:31-40`) não serve, por três razões:
   - devolve `Patient`, e a manifestação aponta para `Citizen`;
   - cria paciente pelo `Patients::Resolve`;
   - exige par validado, e o contrato aceita o par declarado.

   Proposta: `POST /ombudsman/person_lookup { cpf }`, na forma do `Transport::PersonLookupsController` (só corpo, 30/min, `invalid_cpf`), sobre `Citizen.not_erased.where(cpf:)`. Devolve `{ citizens: [{ id, name, cpf_masked, phone_masked }] }`, e a lista vazia não é erro.

A9. **Opções do formulário.** Classificar exige escolher profissional e unidade, mas `/professionals` é só do admin (`app/controllers/professionals_controller.rb:17`). Proposta: `GET /ombudsman/options` com assuntos, unidades, profissionais e setores ativos, no padrão de `psychosocial/options_controller.rb`.

A10. **Tamanho dos textos.** O §2 não fixa o tamanho de pedido de complemento, respostas, decisão, motivo do respondente nem justificativa de prorrogação e de mudança de resolutividade. Proposta: `TEXT = 10..4000`.

A11. **Motivo da recusa do respondente.** `ombudsman_forwardings` não tem coluna para o motivo do `decline`. Proposta: o motivo vai em `reply` (cifrado), com `state = declined`.

A12. **"Uma vez por ciclo" no pedido de reabertura.** `ombudsman_reopen_requests` não tem o ciclo, então o índice "um `pending`" não impede um segundo pedido no mesmo ciclo depois de uma recusa. Proposta: coluna `cycle`, único `(manifestation_id, cycle)`.

A13. **`ombudsman_daily_runs`.** O §12 pede um índice único por dia, e o §4 pede que o primeiro bloqueio do dia publique o evento uma vez. Proposta: a mesma tabela com `kind` (`deadlines`, `anonymous_cap`) e o único `(run_on, kind)`. Alternativa mais simples para o job: o dedup por `ProcessedEvent` do `Surveillance::MailJob` (`app/jobs/surveillance/mail_job.rb:65-73`) mais arquivar só por `expected_state`. Aí a tabela só serviria ao teto.

A14. **Portão de `/departments`.** A spec diz que o setor é genérico, "para outros módulos usarem depois". Proposta: `/departments` sem o portão `ombudsman`. O resto do §7 fica com o portão.

A15. **Semente de assuntos fora da migração.** O banco carregado do dump (testes, cidade nova) nasce sem a linha de configuração e sem a semente, como `TransportSetting` (`app/models/transport_setting.rb:4-5,12-16`). Proposta: `OmbudsmanSetting.ensure_defaults!` e `OmbudsmanSubject.ensure_defaults!` (este só com a tabela vazia), chamados pela leitura de `settings`/`subjects`/`options`.

A16. **Download do anexo.** O §5 pede uma URL curta, mas não define a rota que a serve. Proposta: `GET /ombudsman/files/:token`, com verifier próprio e o caminho mascarado no log (`config/initializers/request_log_path_mask.rb:26`). O cidadão não baixa os próprios anexos: o §3 só lista o nome. Confirmar se o wpda precisa baixar.

A17. **Silêncio do SMS.** O `Transport::Notify` também respeita `appointment_reminders_muted` (`app/commands/transport/notify.rb:108`). Para a ouvidoria, o plano usa só `notices_muted`: o aviso de ouvidoria não é lembrete de horário.

A18. **`cpf_masked` no aviso.** Todas as fontes da caixa trazem `cpf_masked` quando a sessão tem mais de um cidadão (`app/controllers/citizen_api/notices_controller.rb:56,71`), e o §3 não traz. Proposta: incluir.

A19. **Prazo da manifestação já concluída e mudança de configuração.** O prazo é sempre calculado na leitura. Por isso:
   - mudar `response_days` muda o prazo das manifestações abertas;
   - o `on_time_rate` usa o prazo do ciclo vigente com a configuração atual;
   - reabrir apaga o prazo do ciclo anterior.

   Proposta: aceitar e registrar; ou gravar `due_on_at_conclusion` na conclusão, se o relatório anual precisar ser estável antes da publicação (o snapshot já congela).

A20. **Id de unidade e de assunto na URL e no log.** O §13 proíbe "assunto ou unidade" em URL e log, mas o §5 filtra a lista por `subject_id` e `health_unit_id` na query, e a denúncia anônima loga `health_unit_id` nos "Parameters:". O plano permite ids em URL e log (nunca nomes), e nunca ids em payload de evento. Proposta: escrever "nome" no §13. Alternativa: filtrar `health_unit_id` só na rota anônima.

A21. **Cidadão fora da sessão.** O padrão do canal do cidadão é 404 `not_found` (`app/controllers/citizen_api/trips_controller.rb:13-14`, `exams_controller.rb:50`), que não revela se o id existe. O plano segue o contrato no `POST` (403 `not_own_citizen`) e usa 404 nas rotas por `:id` de manifestação. Proposta: 404 também no `POST`, por consistência.

A22. **Unidades públicas incluem CAPS.** O `public_config` lista as unidades ativas, CAPS inclusive. A lista é pública (CNES), e o cidadão precisa poder reclamar do CAPS. O P46/P48 do 31 trata dos documentos do próprio cidadão, não da lista de unidades. Proposta: manter; ou excluir `kind: "health_management"`.

A23. **Payload de `ombudsman.anonymous_cap_reached`.** O contrato diz "sem dado", e o plano publica `{}`. `DomainEvents.publish` aceita payload vazio (`app/events/domain_events.rb:24`). O consumidor lê a data na cidade.

A24. **Registro identificado sem nada.** O §5 não diz o mínimo para `identified`/`preserved` pela equipe. Proposta: `citizen_id` ou `manifestant_name`; sem nenhum dos dois, 422 `path: "manifestant_name"`.

A25. **Envelope da lista da equipe.** O §5 descreve o item, mas não o envelope. As listas do 31 usam `{ items, next_before }` com `before` (`app/controllers/psychosocial/enrollments_controller.rb:33-43`). Proposta: o mesmo.

A26. **Identidade da anônima.** A anônima nunca tem dado de pessoa (o CHECK da Task 3 garante). Proposta: `POST …/identity` responde 422 `no_identity`, sem gravar acesso.

A27. **`complement_requested_at` na volta.** O §1 não diz se zera. Proposta: zera ao receber o complemento (o pedido seguinte do ciclo já é recusado por `complement_used`). O job usa só as `awaiting_complement`.

A28. **Duplicidade pelo protocolo.** A tela conhece o protocolo, não o id. Proposta: `archive` aceita `duplicate_of_protocol` ou `duplicate_of_id`. O protocolo é resolvido no controlador, e a manifestação não pode ser ela mesma.

A29. **Publicar relatório do ano corrente.** O §11 não diz quando o relatório pode ser publicado. Proposta: só ano encerrado (422 `year_not_closed`); o rascunho do ano corrente mostra os números vivos.

A30. **Formato do painel.** O §11 lista as medidas, mas não o formato. Proposta: o do Task 16, com estas regras de supressão:
   - médias com 1 a 4 casos saem `SUPPRESSED` (`{ suppressed: true }`, `app/queries/admin/small_count.rb:5`);
   - sem caso, o valor é `null`;
   - `by_subject` com `unclassified` para as que ainda não têm assunto.

A31. **Corpo da denúncia anônima antes do limite.** O "Parameters:" do log é gerado no `start_processing`, que já interpreta o JSON antes de qualquer `before_action`. O 413 do controlador protege a gravação, mas não a interpretação. Proposta: aceitar (o corpo é filtrado no log, e o limite por IP vem antes). Se o coordenador quiser cortar antes, um middleware de limite só para `/citizen/ombudsman/anonymous_reports`, no padrão de `config/initializers/hl7_raw_body.rb`.
