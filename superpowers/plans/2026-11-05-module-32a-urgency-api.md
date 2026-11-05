# Módulo 32a — Urgência (UPA) (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 31 em `origin/main` do api** (conferido ao escrever este plano em `34ce852f`): última migração de cidade `20261104900004_add_psychosocial_documents.rb` (`db/city_schema.rb` em `define(version: 2026_11_04_900004)`); última de plataforma `20261104900001_add_sigtap_procedure_limits.rb`; `attendances` com `ck_attendances_origin` = `(triage_id IS NULL) <> (appointment_id IS NULL)`, `citizen_id NOT NULL`, `ck_attendances_method` (`code`, `cpf_exception`) e os guardas `rota_attendance_guard()` (colunas do check-in imutáveis; transições `waiting→in_care`, `waiting→closed` com `left/scheduled_from_screening/oriented/referred`, `in_care→closed` fora de `left/...`) e `rota_attendance_insert_guard()` (nasce `waiting`); `Attendance::METHODS`, `CLOSE_OUTCOMES = %w[discharged referred return left]`; `Attendances::{CheckIn,CheckInByException}.call` recusam CAPS com `caps_unit_not_allowed` (P32); `Attendances::Call.call(attendance:, health_unit_id:, by:)`, `Attendances::CallNext.call(health_unit_id:, by:)` sobre `Attendances::UnitQueue.lock_next_waiting` (`FOR UPDATE OF attendances SKIP LOCKED`); `Attendances::Close.call(attendance:, outcome:, referral_unit_id:, referral_note:, by:, regulation: nil)`; `AttendancesController#queue_json` lê `a.citizen.cpf_masked`; `attendances.referral_note` em texto claro; `patients` com `ck_patients_cpf_or_reason` (`num_nonnulls(cpf, no_cpf_reason) = 1`), `no_cpf_reason` cifrado NÃO determinístico (códigos LEDI `TerritoryRegistry::Catalog::NO_CPF_REASONS = %w[2..13]`), guarda `rota_patient_guard()` com a lista `encrypted`; `Patients::CreateTyped.call(attrs:, by:)` (recusas `invalid_patient` `path: new_patient.*`, `cpf_or_reason_required`, `invalid_cpf`, `invalid_cns`, `cpf_taken`, `cns_taken`); `Patients::LookupByIdentifier.call(cpf:, cns:)`; `Patients::IdentifierJson.call(person)`; `consultations.kind` com `ck_consultations_kind` (`general`, `dental`), `ck_consultations_care_type`, `ck_consultations_finalized_care_type` (`status <> 'finalized' OR care_type IS NOT NULL OR baseline`), `ck_consultations_dental_fields` (já aceita qualquer kind não dental com campos vazios), guarda `rota_consultation_guard()` (`patient_id` e `kind` identidade); `Consultations::Start.call(attendance:, by:, kind: "general", baseline: false, city:)` valida `kind` por `DentalCare::Catalog::KINDS` e resolve o paciente por `Patients::Resolve.call(attendance.citizen)`; `Consultations::Cbos.kinds_for(cbo)`; `Consultations::Finalize` chama `Attendances::Close` e `Ledi::ConsultationFichaJob.enqueue_for` (que pula CAPS por `Ledi::ConsultationFicha.caps?`); `Consultations::ItemsInput` com `priority` padrão `routine`; `ClinicalRecord::Access` e `ClinicalRecordsController#context` dependem de `attendance.citizen`; `ClinicalDocuments::Issue` (`replaced` compara `citizen_id`; declaração lê `attendance.citizen.patient_id`); `protocol_definitions` com `idx_protocol_definitions_one_active_per_name_muni`, `ProtocolDefinition.{screening_protocols,triage_protocols}` (este último `kind IS NULL OR kind <> 'screening'`), `Protocols::Validator#screening_result`, `Protocols::Gate` com o desvio `Validation::Screening.screening?`, `config/protocols/schema.json` cópia de `contracts` `protocols-v1.6.0`, `Protocols::ConditionContext::VITALS` (inclui `pain_score`), `Screenings::VitalSigns.parse`, `Screenings::Scope::REQUIRED_SQL`, `TriageCatalogCrew.run_cycle!(slug, name, version, definition)`; `performed_procedures` com `PerformedProcedure::SOURCES = %w[attendance consultation lab_report manual tfd psychosocial]`, `CHARACTERS = %w[elective urgent]`, `ck_performed_procedures_source_ref`, `Billing::Record.insert!(…, psychosocial_action: nil)`, `Billing::Pending.call(period)`, `Billing::Establishment.for_unit`, `Billing::Gate.enabled?`; SIGTAP de dev (`lib/sigtap_sample.rb`) já com `0301060118` (instrumento `01`, CBOs `223505 225125`), `0301060096` e `0301060029` (instrumento `02`); `Timeline::KINDS` termina em `transport`, `Timeline::Query::SOURCES`, `PecTimelinesController::KINDS = Timeline::KINDS - %w[transport]`; `CareLines::TeamPopulation.patient_teams(ids) -> { patient_id => health_team_id }`; `Platform::Features::CATALOG` termina em `psychosocial_care`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..49)`; `Admin::SmallCount.small?`; `LabMiddlewareToken` (só SHA-256, `by_token`); `ProfessionalCrew` semeia "UPA 24h Centro" (`kind: "upa"`) com a médica (225124) e a técnica (322205). Specs que fixam listas e mudam neste plano: `spec/invariants/attendance_invariants_spec.rb` (insere linhas cruas sem `origin_kind`), `spec/commands/billing/from_psychosocial_spec.rb:86` (`AddPsychosocialBillingSource::SOURCES == PerformedProcedure::SOURCES`), `spec/services/timeline/query_spec.rb:75` (`KINDS.last(3)`), `spec/requests/pec_timeline_spec.rb:114`, `spec/models/consultation_tables_guard_spec.rb:130`.

**Goal:** O lado api do módulo 32a (F-32.1 a F-32.6, ADR 0050): interruptores, protocolo de classificação de 5 cores no motor assinado do 18, chegada na UPA com senha e paciente provisório, identificação com fusão única (R4), classificação e reclassificação, fila por cor e "Chamar próximo" da UPA, chamada com local e "chamar de novo", painel de TV pareado sem dado pessoal, atendimento médico como consulta 19a `kind: urgency` fora do LEDI, desfecho mapeado para o 13 com contrarreferência, BPA pelo 22, fonte `urgency` da linha do tempo, painel do gestor, exclusão LGPD e semente de dev.

**Architecture:** Seis migrações de cidade, uma por fronteira, para que cada mudança transversal fique isolada e revisável: o 13 (`attendances`, `health_units`, `900001`), o paciente provisório e a fusão (`patients` e a marca `rota.patient_merge` nos guardas, `900002`), a urgência (`900003`), a sala de espera (`900004`), o 19a (`consultations.kind`, `900005`) e o 22 (`performed_procedures`, `900006`). O encontro (`urgency_encounters`) é 1:1 com um `attendance` de origem `walk_in`, nascido só por `Urgency::Arrive`; o estado da fila, da chamada e do fechamento continua no `attendance` (13), e o encontro guarda o que é clínico da urgência (classificações só de acréscimo, desfecho). O atendimento médico é uma `Consultation` do 19a com `kind: "urgency"`, que finaliza sem fechar o atendimento: quem fecha é `Urgency::RecordOutcome`, que grava o desfecho e chama `Attendances::Close` com o `outcome` equivalente. Senha, local e TV ficam atrás de `waiting_room_display` e valem também na UBS; a TV lê `GET /waiting_room/board` com token próprio (SHA-256), sem sessão e sem trilha.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), Solid Queue, RSpec via `bin/test`, `json_schemer` (já usado pelo `Protocols::Validation::Schema`). **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (só o 32a; as "Decisões do coordenador" R1–R7 prevalecem) e `docs/adr/0050.md`. Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-05-module-32a-urgency-contracts.md` (§1–§13); toda diferença está em "Divergências propostas ao contrato", no fim. Pesquisa: `docs/pesquisa/2026-11-05-urgencia-upa-32.md`.

## Desvios e precisões

1. **Fuso e dia.** Toda data/hora em ISO 8601 no fuso da cidade (`CityConnection.with` fixa `Time.zone`). O "dia" da senha é `checked_in_at.in_time_zone.to_date`, gravado em `attendances.ticket_on`: 23:59 local é do dia; 00:01 local começa a contagem de novo, mesmo que em UTC seja o mesmo dia.
2. **Origem.** `attendances.origin_kind` (`triage`, `appointment`, `walk_in`). O `Attendance` preenche pelo par (`before_validation`); o `rota_attendance_insert_guard()` também preenche quando vier nulo (as linhas cruas dos specs do 13 continuam válidas). `walk_in` só em unidade `upa` (trigger), sempre com `patient_id`, `check_in_method = "urgency_reception"` e `citizen_id` NULO: a pessoa do `walk_in` é o `Patient`, nunca o par do wpda.
3. **Recepção da UPA.** Com `urgency_care` utilizável, `Attendances::CheckIn` e `CheckInByException` numa UPA respondem 422 `urgency_reception_required`: todo atendimento de UPA nasce por `Urgency::Arrive`, então toda chegada à UPA tem encontro. Desligado, a UPA segue o 13 sem mudança.
4. **Paciente provisório.** `patients.unidentified` (bool, não cifrado — o `no_cpf_reason` é cifrado não determinístico e não se busca por ele) mais `no_cpf_reason = "unidentified"` (contrato §1) e `unidentified_description` cifrada. `Patient#display_name` do provisório sem nome = `"Paciente não identificado"`. Provisório nunca serve a outra chegada (`patient_id` de provisório → 422 `invalid_request` `path: "patient_id"`) e nunca é achado pelo `person_lookup` (não tem CPF nem CNS).
5. **Fusão (R4).** Tudo que tem `patient_id` do provisório pertence ao único encontro dele. `Urgency::Merge::TABLES` (movidas para o paciente real, sob a marca `rota.patient_merge`): `attendances consultations clinical_documents exam_orders exam_results lab_requisitions disease_notifications regulation_requests dispensations performed_procedures`. `Urgency::Merge::KEPT` (ficam com o provisório): `patient_problems patient_medications clinical_record_openings clinical_record_administrative_reads patient_profile_divergences` (trilhas são a verdade de quem leu o quê; problemas e medicamentos fora do R4 — Divergência 16). Qualquer outra tabela com linha do provisório bloqueia a fusão (409 `merge_blocked`, nada muda). Uma spec de arquitetura obriga toda tabela com `patient_id` a estar em uma das três listas (`TABLES`, `KEPT` ou bloqueante implícita) — tabela nova futura entra como bloqueante até alguém decidir.
6. **Marca da fusão no banco.** `rota_patient_merge_ok(old jsonb, new jsonb)`: verdadeira só com `rota.patient_merge = 'on'`, quando a linha muda SÓ `patient_id` (e `updated_at`), do provisório para o `merged_into_id` dele. Os guardas que recusam troca de paciente chamam a função no topo: `rota_attendance_guard`, `rota_consultation_guard`, `rota_clinical_document_guard`, `rota_exam_order_guard`, `rota_performed_procedure_guard`, `rota_disease_notification_guard`, `rota_regulation_request_guard` e o genérico `rota_reencryption_only` (`exam_results`, `dispensations`). `Urgency::Merge.with_marker { }` no padrão de `Consultation.with_draft_admin` (savepoint, `SET LOCAL`, volta a `off` no sucesso).
7. **Identificação.** Duas formas, uma vez só, sob `Patient.lock` do provisório: `{patient_id}` (paciente real existente, não provisório e não fundido → fusão) ou `{new_patient}` (`Patients::CreateTyped.parse`; CPF ou CNS que já existem → 409 `cpf_taken`/`cns_taken`, nunca fusão silenciosa pelo que foi digitado; senão o provisório recebe os dados e `unidentified = false`). Já identificado (`unidentified = false` ou `merged_into_id` presente) → 409 `already_identified`.
8. **Estados do encontro.** `waiting_classification → classified` (1ª classificação) `→ in_care` (chamada do médico, `Attendances::Call`) `→ closed` (desfecho); `waiting_classification|classified → closed` só por evasão. Reclassificar vale em qualquer estado antes de `closed` e não muda o estado. Trigger impede voltar.
9. **Relógio do estouro.** `overdue = minutos desde a 1ª classificação > max_wait_minutes da classificação vigente`. Cada classificação grava o `max_wait_minutes` da sua cor no momento (protocolo ativo, ou o padrão `0/10/60/120/240` sem protocolo) e o `protocol_definition_id`: trocar o protocolo ativo não muda o estouro de quem já foi classificado.
10. **Sugestão.** `Urgency::Suggest`: as regras do protocolo ativo `urgency_classification` sobre `Protocols::ConditionContext.build(vitals:, profile:)`, com `pain_scale` no lugar de `vitals.pain_score`; a mais grave entre as que casam, empate pela ordem das regras. A queixa (texto livre, cifrado) nunca entra na condição. Sem protocolo ativo → `nil`.
11. **Chamada do médico na UPA.** `Attendances::Call` num `walk_in` exige o encontro `classified` (422 `not_classified`) e vínculo 225\* na UPA (403 `cbo_not_allowed`), e leva o encontro a `in_care` na mesma transação. `CallNext` numa UPA com `urgency_care` utilizável usa `Urgency::Queue.lock_next_classified` (cor, chegada, id; `SKIP LOCKED`); só não classificados → 404 `queue_empty`.
12. **Local e senha (R6).** Só com `waiting_room_display` utilizável. Local obrigatório quando a unidade tem TV ativa e pareada (422 `call_location_required`); local de outra unidade ou inativo → 422 `invalid_call_location`. Cada chamada (`Call`, `CallNext`, `call_for_classification`, `recall`) grava uma `attendance_calls`; desligado, nenhuma linha e `call_location_id` é ignorado.
13. **Desfecho e o 13.** `Urgency::Catalog::OUTCOME_MAP`: `discharged`/`death` → `discharged`; `discharged_to_primary_care` → `referred` com `referral_note = "Retorno à UBS em até N dias"` (o motivo fica só cifrado em `urgency_outcomes`; `referral_note` do 13 é texto claro); `transferred` → `referred` com `referral_regulated = true`; `left_without_care`/`left_against_advice` → `left`. O `rota_attendance_guard` passa a aceitar `in_care → closed` com `left` quando existe desfecho `left_*` do encontro. Desfechos `discharged`, `discharged_to_primary_care` e `transferred` exigem paciente identificado (422 `patient_unidentified`); `death` e `left_*` não.
14. **Consulta de urgência.** `Consultations::Start`: `kind` omitido = `urgency` quando o atendimento tem encontro, `general` quando não; `urgency` sem encontro ou `general`/`dental` com encontro → 422 `invalid_request` `path: "kind"`. `urgency_care` utilizável (senão 403 `feature_disabled` com `feature: "urgency_care"`). Paciente = `attendance.patient` (sem `Patients::Resolve`). `care_type` nulo. `Finalize` da urgência: sem `care_type`, sem `patient_name_missing` para provisório, NÃO chama `Attendances::Close` (corpo com `outcome` → 422 `invalid_request` `path: "outcome"`), não enfileira LEDI. Exame pedido numa consulta `urgency` sem `priority` nasce `urgent`.
15. **Trilha.** `Urgency::Trail.viewed!(user:, patient_id:, resource:)` publica `clinical_record.viewed` com `access: "urgency"` e `person_ref`; `resource` ∈ `person_lookup`, `encounter`. Fila, painel, TV e listas não gravam trilha.

## Valores fixados por este plano

1. **Interruptores** (depois de `psychosocial_care` no `CATALOG`):
   - `Entry.new(key: "urgency_care", description: "Urgência (UPA): chegada com senha, paciente não identificado, classificação de risco em 5 cores pelo protocolo da cidade, fila por cor, atendimento médico, desfecho, contrarreferência e BPA", requires: %w[feature:clinical_record])`;
   - `Entry.new(key: "waiting_room_display", description: "Painel de chamada da sala de espera (UPA e UBS): senha no check-in, local da chamada e TV pareada, sem dado pessoal", requires: [])`.
2. **`Urgency::Catalog`:** `COLORS = %w[red orange yellow green blue]`; `DEFAULT_MAX_WAIT = { "red" => 0, "orange" => 10, "yellow" => 60, "green" => 120, "blue" => 240 }`; `STATES = %w[waiting_classification classified in_care closed]`; `OUTCOMES = %w[discharged discharged_to_primary_care transferred left_without_care left_against_advice death]`; `IDENTIFIED_OUTCOMES = %w[discharged discharged_to_primary_care transferred]`; `OUTCOME_MAP = { "discharged" => "discharged", "death" => "discharged", "discharged_to_primary_care" => "referred", "transferred" => "referred", "left_without_care" => "left", "left_against_advice" => "left" }`; `CLASSIFIER_CBO_PREFIXES = %w[2235 3222]`; `PHYSICIAN_CBO_PREFIX = "225"`; `COMPLAINT = (3..500)`; `JUSTIFICATION = (10..500)`; `DESCRIPTION = (10..500)`; `DESTINATION = (3..200)`; `VACANCY_REQUEST_MAX = 500`; `RETURN_DAYS = (1..90)`; `RETURN_REASON = (3..300)`; `PAIN = (0..10)`; `CALL_KINDS = %w[classification care]`; `TRAIL_ACCESS = "urgency"`; `UNIDENTIFIED_REASON = "unidentified"`; `COUNTER_REFERRAL_WINDOW = 60.days`.
3. **Protocolo** (`Protocols::Validation::Urgency`): `KIND = "urgency_classification"`; `NAME = "classificacao-urgencia"` (reservado; um ativo por cidade pelo índice único por nome); `levels` = exatamente as 5 cores, na ordem de `COLORS`, `max_wait_minutes` inteiro 0..1440, não decrescente; `rules` 1..100, cada `{ id, when, color, criterion }`, `id` único `/\A[a-z0-9][a-z0-9-]{0,39}\z/`, `criterion` 1..120 caracteres; variáveis do `when`: `vitals.*` de `ConditionContext::VITALS` e `profile.age`, `profile.sex`. (Formato do contracts suposto — Task 0 confere; Divergência 4.)
4. **`HealthUnit`:** `ticket_prefix` string(3) `NOT NULL DEFAULT 'A'`, CHECK `^[A-Z]{1,3}$`; `show_wait_times` bool `NOT NULL DEFAULT false`; a migração põe `'U'`/`true` nas UPAs existentes; o modelo, ao criar uma `upa` sem valor, `U`/`true`. `#upa?`.
5. **Senha:** `"#{prefix}-#{format('%03d', n)}"` com `n` = senhas da unidade em `ticket_on` + 1 (sob `HealthUnit.lock_active!`); CHECK `^[A-Z]{1,3}-[0-9]{3,4}$`; único `(health_unit_id, ticket_on, ticket) WHERE ticket IS NOT NULL`.
6. **TV:** código de pareamento de 6 dígitos (`SecureRandom.random_number(10**6)`, `%06d`), só o SHA-256, 10 min, uso único; token `SecureRandom.urlsafe_base64(48)` (64 caracteres), só o SHA-256; `POST /waiting_room/pair` limitado a 10/min por IP; `GET /waiting_room/board` limitado a 30/min por digest do token; `calls` = as 8 mais recentes dos últimos 30 min com senha; polling do dashboard a cada 5 s cabe no limite.
7. **`config/urgency/sigtap.yml`** (`version: 1`, `review: true`, `source: "Portaria SAS 819/2015 — conferir instrumento, CBOs e valores no SIGTAP oficial"`, `classification_code: "0301060118"`, `care_code: "0301060096"`). `Urgency::Config.sigtap` lê uma vez e congela; chave desconhecida falha o `spec/config/urgency_sigtap_spec.rb`.
8. **Dev:** api do worktree na porta **3054**; worktree `apps/api/.claude/mod32a` (com `config/master.key`); branch `feat/mod-32a-urgency`; migrações `db/city_migrate/{20261105900001_add_urgency_attendance_origin,20261105900002_add_urgency_patients,20261105900003_add_urgency_care,20261105900004_add_waiting_room,20261105900005_add_urgency_consultation_kind,20261105900006_add_urgency_billing_source}.rb`. Semente (Task 17): Curitiba, "UPA 24h Centro".

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (os arquivos da task, por nome); commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-32.1..F-32.6 do board #1 para In Progress (um comando `gh project item-edit …` por chamada, começando com `gh`). Card vai para Done quando o ÚLTIMO app dele terminar (F-32.1 também dashboard, maintenance e contracts; F-32.2..F-32.5 também dashboard; F-32.6 também dashboard e wpda); Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod32a <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. Mudança transversal (13, 19a, 22, 18, linha do tempo, `spec/support`) → **suíte completa obrigatória antes do merge**, pedida à sessão "Testes" (Task 18).
- Migrações: dump à mão em `db/city_schema.rb` (fim: `define(version: 2026_11_05_900006)`); triggers em `db/city_triggers.sql`, executado no fim do `up` de cada migração que o muda; todas irreversíveis. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error", "path"? }` + `problems: [ { path, error } ]` nas recusas de campo (P23); 409 de conflito; interruptor 403 `{ "error": "feature_disabled", "feature": "urgency_care" | "waiting_room_display" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (o pareamento da TV também é JSON); listas com envelope `{ items }`.
- **CPF, CNS, nome, queixa, CID, critério, sinais e a descrição do não identificado nunca em log, evento, Analytics, erro ou URL.** A busca de pessoa é só `POST` com corpo (`query_only!`). A TV nunca recebe nome, CPF, CNS, cor da pessoa, queixa nem id de paciente ou encontro. Modelos novos com dado do paciente sobrescrevem `inspect`/`to_s`/`pretty_print`/`as_json` (só `id`). Filtro de log novo: `/\A(complaint|criterion|vital_signs|pain_scale|destination|vacancy_request|return_guidance|description|unidentified|suggested)\z/` (`color`, `justification`, `reason`, `token` e `code` já são filtrados).
- Guardas do 31: nada em CAPS — chegada numa unidade `caps` → 422 `caps_unit_not_allowed` antes de `not_an_upa`.
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids (contrato §12); `DomainEvents.publish(` é multilinha — confira cada chamada inteira. Nenhum `Platform.audit` novo (R18 intacto); se surgir, entra em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`).
- Nunca `Current.city =` solto; interruptores lidos **fora** da transação da cidade.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); coluna cifrada de tabela imutável pelo padrão `rota_reencryption_only(...)`; `unidentified_description` entra na lista `encrypted` do `rota_patient_guard`.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..50)`.
- Confira nomes reais no código antes de usá-los (o PRÉ-REQUISITO lista os conferidos).

## Review Focus

1. **Duas recepcionistas identificam o mesmo provisório ao mesmo tempo, cada uma com um paciente real diferente:** uma vence (fusão completa), a outra recebe 409 `already_identified`; nenhuma linha fica dividida entre os dois reais, e o `merged_into_id` nunca muda depois. Teste na Task 9.
2. **Senha na virada do dia local:** chegada às 23:59 (02:59 UTC do dia seguinte) leva `U-0NN` do dia; a de 00:01 local leva `U-001`; duas chegadas simultâneas na mesma UPA nunca repetem a senha. Teste na Task 2.
3. **Reclassificação para cima de quem já espera:** verde há 50 min reclassificado para laranja passa ao topo da fila logo abaixo dos vermelhos, aparece `overdue` (50 > 10, relógio desde a 1ª classificação) e o "Chamar próximo" o leva antes de um amarelo que chegou antes. Teste na Task 8.
4. **Protocolo trocado com gente esperando:** ativar a versão 2 com amarelo em 30 min não torna `overdue` quem foi classificado amarelo na versão 1 há 40 min (snapshot 60); quem é classificado depois usa 30. Teste na Task 7.
5. **Óbito de não identificado identificado no dia seguinte:** o encerramento não gera linha BPA e a competência mostra a pendência `fix: "patient"`; a identificação (com fusão) gera as duas linhas no paciente real pelo evento `urgency.patient_identified`, sem duplicar se o evento for reprocessado. Teste na Task 13.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod32a/`)
- Migrações: as seis do Valor fixado 8.
- Modelos: `app/models/{urgency_encounter,urgency_classification,urgency_outcome,call_location,attendance_call,waiting_room_display}.rb`.
- `config/urgency/sigtap.yml`.
- `app/protocols/validation/urgency.rb`.
- `app/services/urgency/{gate,catalog,config,access,trail,patient_ref,active_protocol,levels,suggest,queue,merge,json,dashboard,counter_referrals,billing_pending}.rb`; `app/services/waiting_room/{gate,call_locations,board}.rb`; `app/services/attendances/{ticket,call_record}.rb`.
- `app/commands/urgency/{arrive,classify,call_for_classification,identify,record_outcome}.rb`; `app/commands/attendances/recall.rb`; `app/commands/waiting_room/{save_call_location,save_display,issue_pairing_code,pair,revoke}.rb`; `app/commands/billing/from_urgency.rb`.
- `app/controllers/concerns/{urgency_gate,waiting_room_gate}.rb`; `app/controllers/urgency/{base,person_lookups,arrivals,queue,suggestions,encounters,classifications,counter_referrals,dashboard}_controller.rb`; `app/controllers/waiting_room/{base,displays,pairings,boards}_controller.rb`; `app/controllers/call_locations_controller.rb`.
- `app/jobs/billing/urgency_production_job.rb`; `app/services/timeline/sources/urgency_source.rb`.
- `lib/urgency_crew.rb`; `spec/support/urgency_helpers.rb`; specs listados em cada task.

**Modificar**
- 13: `app/models/{attendance,health_unit}.rb`, `app/commands/attendances/{check_in,check_in_by_exception,call,call_next,close}.rb`, `app/controllers/{attendances,health_units,check_ins}_controller.rb`.
- Paciente: `app/models/patient.rb`.
- 18: `config/protocols/schema.json`, `app/protocols/{validator,gate}.rb`, `app/models/protocol_definition.rb`, `app/services/screenings/scope.rb`, `app/commands/screenings/start.rb`, `app/controllers/{screenings_controller,authoring/protocols_controller}.rb`.
- 19a/19c: `app/models/consultation.rb`, `app/services/consultations/{cbos,items_input}.rb`, `app/commands/consultations/{start,finalize,save_draft}.rb`, `app/services/ledi/consultation_ficha.rb`, `app/services/clinical_record/access.rb`, `app/controllers/clinical_records_controller.rb`, `app/commands/clinical_documents/issue.rb`.
- 22: `app/commands/billing/record.rb`, `app/models/performed_procedure.rb`, `app/services/billing/pending.rb`.
- Linha do tempo: `app/services/timeline.rb`, `app/services/timeline/{query,citizen_documents}.rb`, `app/services/timeline/sources/{consultation_source,clinical_document_source}.rb`, `app/controllers/pec_timelines_controller.rb`.
- LGPD: `app/commands/citizens/request_erasure.rb`.
- Plataforma e infra: `app/services/platform/features.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `db/seeds.rb`.
- Specs existentes: `spec/adr_pointers_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/invariants/attendance_invariants_spec.rb` (só se a derivação do trigger não bastar), `spec/commands/billing/from_psychosocial_spec.rb`, `spec/services/timeline/query_spec.rb`, `spec/requests/pec_timeline_spec.rb`, `spec/models/consultation_tables_guard_spec.rb`.

---

### Task 0: Worktree, cards, contracts e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-32.1..F-32.6 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod32a -b feat/mod-32a-urgency origin/main && cp config/master.key .claude/mod32a/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO com `grep` no worktree (`VALID_RANGE`, `ck_attendances_origin`, `rota_attendance_guard`, `rota_patient_guard` e o array `encrypted`, `ck_consultations_kind`, `triage_protocols`, `PerformedProcedure::SOURCES`, `Timeline::KINDS`, `PecTimelinesController::KINDS`, `"UPA 24h Centro"` no `lib/professional_crew.rb`, a linha `0301060118` do `lib/sigtap_sample.rb`); diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** contracts: `/opt/homebrew/bin/git -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts fetch --tags && /opt/homebrew/bin/git -C …/contracts show protocols-v1.7.0:protocols/schema.json > /dev/null`. Sem a tag, a Task 5 espera (as Tasks 1–4 não dependem dela). Com a tag, conferir no `$defs` do kind `urgency_classification` os nomes `levels[].color`, `levels[].max_wait_minutes`, `rules[].{id,when,color,criterion}` e o nome reservado do Valor fixado 3; diferença → parar e avisar o coordenador (Divergência 4), sem adaptar em silêncio.
- [ ] **Step 5:** `bin/test api -w mod32a spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb spec/invariants/attendance_invariants_spec.rb spec/models/consultation_tables_guard_spec.rb` → PASS (linha de base).

---

### Task 1: Interruptores, portões, catálogo e configuração do BPA

**Files:**
- Create: `app/services/urgency/{gate,catalog,config}.rb`, `app/services/waiting_room/gate.rb`, `app/controllers/concerns/{urgency_gate,waiting_room_gate}.rb`, `config/urgency/sigtap.yml`, `spec/config/urgency_sigtap_spec.rb`
- Modify: `app/services/platform/features.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/services/platform/features_spec.rb`, `spec/adr_pointers_spec.rb`

**Interfaces:**
- Produces: `Urgency::Gate::KEY = "urgency_care"`, `Urgency::Gate.usable?(city) -> Boolean` (o padrão de `Psychosocial::Gate`: `false` para `city` nil ou sem id); `WaitingRoom::Gate::KEY = "waiting_room_display"`, `WaitingRoom::Gate.usable?(city)`; concerns `UrgencyGate#require_urgency!` e `WaitingRoomGate#require_waiting_room!` (403 `{ error: "feature_disabled", feature: KEY }`); `Urgency::Catalog` (Valor fixado 2); `Urgency::Config.sigtap -> Hash` congelado (`"classification_code"`, `"care_code"`, `"review"`).

- [ ] **Step 1: Write the failing tests.** `features_spec.rb` ganha:

```ruby
it "módulo 32a: urgency_care exige o prontuário; waiting_room_display é independente" do
  keys = Platform::Features::CATALOG.map(&:key)
  expect(keys.last(2)).to eq(%w[urgency_care waiting_room_display])
  expect(Platform::Features::CATALOG.find { |e| e.key == "urgency_care" }.requires).to eq(%w[feature:clinical_record])
  expect(Platform::Features::CATALOG.find { |e| e.key == "waiting_room_display" }.requires).to eq([])
  expect(Platform::Features::CITIZEN_HIDDEN).not_to include("urgency_care", "waiting_room_display")
end
```

`spec/config/urgency_sigtap_spec.rb`: o YAML carrega; chaves exatamente `%w[version review source classification_code care_code]`; `review == true`; os dois códigos com 10 dígitos e iguais a `"0301060118"`/`"0301060096"`; `Urgency::Config.sigtap.frozen?`. Também: `Urgency::Catalog::COLORS == %w[red orange yellow green blue]` e `DEFAULT_MAX_WAIT.values == [0, 10, 60, 120, 240]`; `Rails.application.config.filter_parameters` filtra `complaint`, `criterion`, `vital_signs`, `description`, `return_guidance` (`ActiveSupport::ParameterFilter.new(...).filter("complaint" => "x")["complaint"] == "[FILTERED]"`).

- [ ] **Step 2:** `bin/test api -w mod32a spec/services/platform/features_spec.rb spec/config/urgency_sigtap_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** as duas entradas do Valor fixado 1 (com o comentário do padrão: sem cascata; desligar o prontuário deixa `urgency_care` ligado, faltando pré-requisito), os portões, os concerns, o YAML (Valor fixado 7), o filtro de log e `VALID_RANGE = (1..50)`.
- [ ] **Step 4:** os mesmos specs + `spec/adr_pointers_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add urgency care and waiting room display switches`.

---

### Task 2: Atendimento do 13 — origem `walk_in`, paciente, senha e a unidade (transversal, §1)

**Files:**
- Create: `db/city_migrate/20261105900001_add_urgency_attendance_origin.rb`, `app/services/attendances/ticket.rb`, `spec/models/urgency_attendance_origin_spec.rb`, `spec/services/attendances/ticket_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (`rota_attendance_guard`, `rota_attendance_insert_guard`), `app/models/{attendance,health_unit}.rb`, `app/commands/attendances/{check_in,check_in_by_exception}.rb`, `app/controllers/{attendances,health_units}_controller.rb`, `spec/requests/health_units_spec.rb`

**Interfaces:**
- Produces (banco): `attendances.origin_kind` (string), `patient_id` (uuid, FK `patients`, índice), `citizen_id` NULO, `ticket` (string 8), `ticket_on` (date); CHECKs `ck_attendances_origin` = `origin_kind IS NOT NULL AND ((origin_kind = 'triage' AND triage_id IS NOT NULL AND appointment_id IS NULL) OR (origin_kind = 'appointment' AND appointment_id IS NOT NULL AND triage_id IS NULL) OR (origin_kind = 'walk_in' AND triage_id IS NULL AND appointment_id IS NULL AND patient_id IS NOT NULL AND citizen_id IS NULL))`, `ck_attendances_citizen` = `origin_kind = 'walk_in' OR citizen_id IS NOT NULL`, `ck_attendances_method` + `urgency_reception`, `ck_attendances_exception_reason` + `OR (check_in_method = 'urgency_reception' AND exception_reason IS NULL)`, `ck_attendances_walk_in_method` = `(origin_kind = 'walk_in') = (check_in_method = 'urgency_reception')`, `ck_attendances_ticket` = `(ticket IS NULL) = (ticket_on IS NULL) AND (ticket IS NULL OR ticket ~ '^[A-Z]{1,3}-[0-9]{3,4}$')`; índices únicos `idx_attendances_ticket (health_unit_id, ticket_on, ticket) WHERE ticket IS NOT NULL` e `idx_attendances_open_walk_in (patient_id, health_unit_id) WHERE origin_kind = 'walk_in' AND status <> 'closed'`; backfill `origin_kind` das linhas existentes pelo par. `health_units.ticket_prefix`, `show_wait_times` (Valor fixado 4).
- Produces (travas): `rota_attendance_insert_guard()` preenche `NEW.origin_kind` nulo pelo par e recusa `walk_in` fora de unidade `upa` (`'attendances: walk_in only at an upa'`); `rota_attendance_guard()` acrescenta à lista "check-in never change" `origin_kind`, `ticket`, `ticket_on` e `patient_id` (a exceção da fusão entra na Task 3).
- Produces (Ruby): `Attendance::METHODS = %w[code cpf_exception urgency_reception]`, `ORIGIN_KINDS = %w[triage appointment walk_in]`, `belongs_to :citizen, optional: true`, `belongs_to :patient, optional: true`, `has_one :urgency_encounter` (a Task 4 cria o modelo), `#walk_in?`, `#person` (`citizen || patient`); `HealthUnit#upa?`, `TICKET_PREFIX = /\A[A-Z]{1,3}\z/`; `Attendances::Ticket.issue(unit, at: Time.current) -> [ticket, ticket_on]` (chame sob `HealthUnit.lock_active!`; só quem já leu `WaitingRoom::Gate.usable?` fora da transação chama); `CheckIn`/`CheckInByException` ganham `waiting_room:` e `urgency:` (booleans lidos pelo controlador) — com `urgency && unit.upa?` → `Result.fail(:urgency_reception_required)`; com `waiting_room` a linha nasce com `ticket`/`ticket_on`; `HealthUnitsController` aceita `ticket_prefix` (fora do formato → 422 `invalid_ticket_prefix`) e `show_wait_times` (não booleano → `invalid_show_wait_times`) e os devolve no `unit_json`; `AttendancesController#queue_json` usa `a.person` (o `walk_in` mostra o nome do paciente e `cpf_masked` dele) e devolve `ticket`.

- [ ] **Step 1: Write the failing tests.**

```ruby
RSpec.describe "Atendimento do 13 com a origem walk_in", type: :model do
  let!(:city) { clinical_city! }
  let(:upa) { create_unit("UPA Teste", kind: "upa") }
  let(:ubs) { create_unit("UBS Teste") }
  let(:patient) { Patient.create!(cpf: "52998224725", full_name: "Rosa Maria Campos", birth_date: "1970-05-05", sex: "female") }

  def walk_in!(unit, **over)
    Attendance.create!({ origin_kind: "walk_in", patient: patient, health_unit: unit, checked_in_by_user: reception!,
                         checked_in_at: Time.current, check_in_method: "urgency_reception" }.merge(over))
  end

  it "walk_in só na UPA, sem cidadão e sem triagem; o par do 13 continua obrigatório fora dele" do
    expect(walk_in!(upa).citizen_id).to be_nil
    expect { walk_in!(ubs) }.to raise_error(ActiveRecord::StatementInvalid, /walk_in only at an upa/)
    expect { walk_in!(upa, citizen: screening_citizen!(1)) }.to raise_error(ActiveRecord::StatementInvalid, /ck_attendances_origin/)
    expect { walk_in!(upa, check_in_method: "code") }.to raise_error(ActiveRecord::StatementInvalid, /ck_attendances_walk_in_method/)
    expect { Attendance.insert_all!([ { health_unit_id: ubs.id, checked_in_by_user_id: reception!.id, checked_in_at: Time.current,
                                         check_in_method: "code", status: "waiting", created_at: Time.current } ]) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_attendances_origin/)
  end

  it "um walk_in aberto por paciente e UPA; o paciente, a senha e a origem nunca mudam" do
    a = walk_in!(upa)
    expect { walk_in!(upa) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_attendances_open_walk_in/)
    expect { a.update_column(:origin_kind, "triage") }.to raise_error(ActiveRecord::StatementInvalid, /check-in columns never change/)
    expect { a.update_column(:patient_id, Patient.create!(cpf: "11144477735", full_name: "Outra Pessoa", birth_date: "1990-01-01", sex: "male").id) }
      .to raise_error(ActiveRecord::StatementInvalid, /check-in columns never change/)
  end
end
```

`spec/services/attendances/ticket_spec.rb` (Review Focus 2):

```ruby
RSpec.describe Attendances::Ticket do
  let!(:city) { clinical_city! }
  let(:upa) { create_unit("UPA Senha", kind: "upa") }

  it "conta por unidade e dia LOCAL; 23:59 é do dia, 00:01 recomeça" do
    travel_to(Time.zone.parse("2026-11-05 23:59")) do
      2.times { |i| walk_in_with_ticket!(upa) }
      expect(described_class.issue(upa)).to eq([ "U-003", Date.new(2026, 11, 5) ])
    end
    travel_to(Time.zone.parse("2026-11-06 00:01")) { expect(described_class.issue(upa)).to eq([ "U-001", Date.new(2026, 11, 6) ]) }
  end

  it "o prefixo é o da unidade; a UBS nasce com A" do
    expect(described_class.issue(create_unit("UBS Senha")).first).to eq("A-001")
  end
end
```

`spec/requests/health_units_spec.rb`: criar UPA devolve `"ticket_prefix" => "U", "show_wait_times" => true`; `ticket_prefix: "u1"` → `invalid_ticket_prefix`; check-in por código numa UPA com `urgency_care` ligado → 422 `urgency_reception_required` (em `spec/requests/check_ins_spec.rb`, novo exemplo); com `waiting_room_display` ligado, o check-in da UBS devolve `ticket: "A-001"`.

- [ ] **Step 2:** `bin/test api -w mod32a spec/models/urgency_attendance_origin_spec.rb spec/services/attendances/ticket_spec.rb spec/requests/health_units_spec.rb spec/requests/check_ins_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Migração (`replace_check` no padrão das anteriores; backfill `UPDATE attendances SET origin_kind = CASE WHEN triage_id IS NOT NULL THEN 'triage' ELSE 'appointment' END`; `UPDATE health_units SET ticket_prefix = 'U', show_wait_times = true WHERE kind = 'upa'`; `execute File.read(Rails.root.join("db/city_triggers.sql"))`; `down` irreversível). `walk_in!` usa um paciente com CPF: o provisório só existe na Task 3. `Attendance` com `before_validation { self.origin_kind ||= triage_id ? "triage" : (appointment_id ? "appointment" : nil) }` e `validates :citizen, presence: true, unless: :walk_in?`. `Ticket.issue`: `n = Attendance.where(health_unit_id: unit.id, ticket_on: day).count + 1`. `CheckInsController` lê `Urgency::Gate.usable?` e `WaitingRoom::Gate.usable?` antes de chamar o comando; `urgency_reception_required` → 422 no `ERROR_STATUS`.
- [ ] **Step 4:** os mesmos specs + `spec/invariants/attendance_invariants_spec.rb spec/requests/attendances_spec.rb spec/commands/attendances` → PASS.
- [ ] **Step 5: Commit** `feat: add walk-in origin, patient and daily ticket to attendances`.

---

### Task 3: Paciente provisório e a marca da fusão no banco

**Files:**
- Create: `db/city_migrate/20261105900002_add_urgency_patients.rb`, `spec/models/urgency_patient_merge_guard_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (`rota_patient_guard`, nova `rota_patient_merge_ok`, e o topo dos oito guardas do Desvio 6), `app/models/patient.rb`, `app/services/city_encryption.rb`

**Interfaces:**
- Produces (banco): `patients.unidentified` bool `NOT NULL DEFAULT false`, `merged_into_id` (uuid, FK `patients`, índice), `unidentified_description` (text cifrado); CHECKs `ck_patients_unidentified` = `NOT unidentified OR (cpf IS NULL AND cns IS NULL)`, `ck_patients_merged` = `merged_into_id IS NULL OR (unidentified AND merged_into_id <> id)`, `ck_patients_description` = `NOT unidentified OR unidentified_description IS NOT NULL`. `rota_patient_guard()`: `unidentified` nunca vai de `false` a `true`; `true → false` só com `merged_into_id` nulo; `merged_into_id` só de nulo a valor, uma vez, com o alvo existente, `NOT unidentified` e sem `merged_into_id` (`'patients: merge target must be an identified patient'`), e nunca muda depois (`'patients: merged_into never changes'`); colunas lidas por `to_jsonb` (o comentário do padrão do 24a); `unidentified_description` no array `encrypted`.
- Produces (função): `rota_patient_merge_ok(old_row jsonb, new_row jsonb) RETURNS boolean` (SQL, `STABLE`), o Desvio 6; chamada no topo de cada guarda listado como `IF TG_OP = 'UPDATE' AND rota_patient_merge_ok(to_jsonb(OLD), to_jsonb(NEW)) THEN RETURN NEW; END IF;` (no `rota_reencryption_only`, antes da conferência das colunas).
- Produces (Ruby): `Patient` com `encrypts :unidentified_description`, `belongs_to :merged_into, class_name: "Patient", optional: true`, `UNIDENTIFIED_NAME = "Paciente não identificado"`, `scope :identified, -> { where(unidentified: false) }`, `#merged?`, `display_name` = `UNIDENTIFIED_NAME` quando `unidentified && full_name.blank?`; `Patient#inspect` não traz a descrição; `[ Patient, :unidentified_description ]` em `CITY_KEYED_TARGETS`.

- [ ] **Step 1: Write the failing test** `spec/models/urgency_patient_merge_guard_spec.rb`:

```ruby
RSpec.describe "Fusão do paciente provisório no banco", type: :model do
  let!(:city) { clinical_city! }
  let(:upa) { create_unit("UPA Fusão", kind: "upa") }
  let(:real) { Patient.create!(cpf: "52998224725", full_name: "Carlos Eduardo Pinto", birth_date: "1980-02-01", sex: "male") }
  let(:provisional) { Patient.create!(no_cpf_reason: "unidentified", unidentified: true, unidentified_description: "Homem, 40 anos, rodoviária") }
  let!(:attendance) do
    Attendance.create!(origin_kind: "walk_in", patient: provisional, health_unit: upa, checked_in_by_user: reception!,
                       checked_in_at: Time.current, check_in_method: "urgency_reception")
  end

  def merging = ApplicationRecord.transaction(requires_new: true) { ApplicationRecord.connection.execute("SET LOCAL rota.patient_merge = 'on'"); yield }

  it "sem a marca, ninguém troca o paciente; com a marca, só para o alvo gravado no provisório" do
    expect { attendance.update_column(:patient_id, real.id) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { merging { attendance.update_column(:patient_id, real.id) } }.to raise_error(ActiveRecord::StatementInvalid)
    provisional.update!(merged_into: real)
    other = Patient.create!(cpf: "11144477735", full_name: "Outra Pessoa", birth_date: "1990-01-01", sex: "female")
    expect { merging { attendance.update_column(:patient_id, other.id) } }.to raise_error(ActiveRecord::StatementInvalid)
    expect { merging { attendance.update_columns(patient_id: real.id, ticket: "U-999", ticket_on: Date.current) } }
      .to raise_error(ActiveRecord::StatementInvalid)
    merging { attendance.update_column(:patient_id, real.id) }
    expect(attendance.reload.patient_id).to eq(real.id)
  end

  it "merged_into uma vez, só para paciente identificado e nunca para si mesmo" do
    expect { provisional.update!(merged_into: Patient.create!(no_cpf_reason: "unidentified", unidentified: true, unidentified_description: "Outro provisório")) }
      .to raise_error(ActiveRecord::StatementInvalid, /merge target must be an identified patient/)
    expect { provisional.update_column(:merged_into_id, provisional.id) }.to raise_error(ActiveRecord::StatementInvalid, /ck_patients_merged/)
    provisional.update!(merged_into: real)
    expect { provisional.update!(merged_into: Patient.create!(cpf: "11144477735", full_name: "Outra", birth_date: "1990-01-01", sex: "female")) }
      .to raise_error(ActiveRecord::StatementInvalid, /merged_into never changes/)
    expect { provisional.update!(unidentified: false) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { real.update!(unidentified: true) }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "provisório não tem CPF nem CNS; o nome de tela não expõe a descrição" do
    expect { provisional.update!(cpf: "52998224725", no_cpf_reason: nil) }.to raise_error(ActiveRecord::StatementInvalid, /ck_patients_unidentified/)
    expect(provisional.display_name).to eq("Paciente não identificado")
    expect(provisional.inspect).not_to include("rodoviária")
  end
end
```

Mais um exemplo por tabela de `Urgency::Merge::TABLES` com guarda (`consultations`, `clinical_documents`, `exam_orders`, `exam_results`, `performed_procedures`, `disease_notifications`, `regulation_requests`, `dispensations`): linha do provisório criada pelo helper do módulo dono (`started_consultation!`-like sobre o `walk_in`, `raw_procedure!`, etc.), troca com a marca e o alvo certo → aceita; qualquer outra coluna junto → recusada.

- [ ] **Step 2:** `bin/test api -w mod32a spec/models/urgency_patient_merge_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (colunas, CHECKs, `execute` do `db/city_triggers.sql`, irreversível), a função e os oito pontos de chamada (comentário "Módulo 32a (ADR 0050, R4): a fusão do provisório só troca `patient_id`, sob a marca"), o modelo e o alvo de cifra. `Urgency::Merge::TABLES`/`KEPT` ainda não existem: a lista de guardas segue o Desvio 6.
- [ ] **Step 4:** o spec + `spec/models/urgency_attendance_origin_spec.rb spec/models/consultation_tables_guard_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/models/patient_spec.rb spec/commands/patients` → PASS.
- [ ] **Step 5: Commit** `feat: add provisional patients and the merge marker to patient guards`.

---

### Task 4: Esquema da urgência e da sala de espera, modelos e helpers

**Files:**
- Create: `db/city_migrate/20261105900003_add_urgency_care.rb`, `db/city_migrate/20261105900004_add_waiting_room.rb`, os seis modelos do mapa, `spec/support/urgency_helpers.rb`, `spec/models/urgency_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (bloco "Módulo 32a (ADR 0050)"), `app/services/city_encryption.rb`, `app/models/attendance.rb` (`has_one :urgency_encounter`), `app/models/health_unit.rb` (`has_many :call_locations`, `:waiting_room_displays`)

**Interfaces:**
- Produces (tabelas, `id: :uuid`, `created_at` em todas):
  - `urgency_encounters` (`attendance_id` único NOT NULL FK, `consultation_id` nulo FK, `state` default `waiting_classification` CHECK em `STATES`, `current_classification_id` nulo FK, `first_classified_at` nulo, `updated_at`; CHECK `(state = 'waiting_classification') = (current_classification_id IS NULL) AND (current_classification_id IS NULL) = (first_classified_at IS NULL)`).
  - `urgency_classifications` (`urgency_encounter_id` NOT NULL FK, `position smallint` CHECK ≥ 1, único `(encounter, position)`, `color` CHECK em `COLORS`, `suggested_color` nulo CHECK em `COLORS`, `suggested_rule_id` string(40) nulo, `criterion` cifrado nulo, `complaint` cifrado, `vital_signs` cifrado (JSON), `pain_scale smallint` CHECK 0..10, `justification` cifrado nulo, `max_wait_minutes int` CHECK 0..1440, `protocol_definition_id` nulo FK, `classified_by_user_id` NOT NULL, `professional_link_id` NOT NULL, `cbo_code` string(6); CHECKs `ck_urgency_classifications_justification` = `suggested_color IS NOT DISTINCT FROM color OR justification IS NOT NULL` e `(suggested_color IS NULL) = (suggested_rule_id IS NULL)`).
  - `urgency_outcomes` (`urgency_encounter_id` único NOT NULL FK, `kind` CHECK em `OUTCOMES`, `destination` cifrado nulo, `vacancy_request` cifrado nulo, `return_days smallint` nulo CHECK 1..90, `return_reason` cifrado nulo, `counter_referral_health_team_id` nulo FK, `counter_referral_health_unit_id` nulo FK, `recorded_by_user_id` NOT NULL, `recorded_at`; CHECKs `(kind = 'transferred') = (destination IS NOT NULL)`, `(kind = 'discharged_to_primary_care') = (return_days IS NOT NULL) AND (return_days IS NULL) = (return_reason IS NULL)`, `vacancy_request IS NULL OR kind = 'transferred'`, `counter_referral_health_unit_id IS NULL OR kind = 'discharged_to_primary_care'`).
  - `call_locations` (`health_unit_id`, `name` string(40), `active` default true, `updated_by_user_id`, `updated_at`; único `idx_call_locations_name (health_unit_id, lower(name))`).
  - `attendance_calls` (`attendance_id`, `ticket` nulo, `call_location_id` nulo FK, `kind` CHECK em `CALL_KINDS`, `called_at`, `called_by_user_id`; índice `(attendance_id, called_at)`; índice `idx_attendance_calls_board` sobre `called_at` com `INCLUDE`/join por `attendances.health_unit_id` — a TV filtra por unidade e janela).
  - `waiting_room_displays` (`health_unit_id`, `name` string(60), `active` default true, `show_wait_times` default true, `token_digest` string(64) nulo único parcial, `pairing_code_digest` string(64) nulo, `pairing_expires_at` nulo, `paired_at` nulo, `revoked_at` nulo, `created_by_user_id`, `updated_at`; CHECK `(pairing_code_digest IS NULL) = (pairing_expires_at IS NULL)`).
- Produces (travas, `db/city_triggers.sql`): `rota_urgency_encounter_guard()` (DELETE recusado; `attendance_id`/`created_at` nunca mudam; `consultation_id` de nulo a valor uma vez; `first_classified_at` uma vez; estado só para frente pelo Desvio 8; `current_classification_id` aponta para classificação do MESMO encontro — `'urgency_encounters: classification of another encounter'`; `closed` nunca muda); `rota_append_only()` + `rota_reencryption_only('complaint','vital_signs','criterion','justification')` em `urgency_classifications`, idem `('destination','vacancy_request','return_reason')` em `urgency_outcomes`, `rota_append_only()` em `attendance_calls`; `rota_no_delete()` em `call_locations` e `waiting_room_displays`; TRUNCATE recusado nas seis.
- Produces (modelos): `UrgencyEncounter` (`belongs_to :attendance`, `:consultation, optional`, `:current_classification, class_name: "UrgencyClassification", optional`; `has_many :classifications, -> { order(:position) }`; `has_one :outcome, class_name: "UrgencyOutcome"`; `#patient` = `attendance.patient`; `#closed?`; `#color` = `current_classification&.color`); `UrgencyClassification` (`encrypts` das quatro; `vital_signs` com `serialize :vital_signs, coder: JSON`); `UrgencyOutcome`; `CallLocation` (`scope :active`); `AttendanceCall`; `WaitingRoomDisplay` (`scope :paired_active, -> { where(active: true, revoked_at: nil).where.not(token_digest: nil) }`, `self.digest(value)`, `self.by_token(token)` no padrão do `LabMiddlewareToken`, `#revoked?`). Os três primeiros com `inspect`/`to_s`/`pretty_print`/`as_json` só `id`.
- Produces (helpers, `spec/support/urgency_helpers.rb`, avaliados só na chamada): `UrgencyHelpers::LEVELS` e `::RULES` (abaixo); `urgency_city!(billing: false, waiting_room: false)` (`clinical_city!` ou `billing_city!`, liga `urgency_care` e, com `waiting_room: true`, `waiting_room_display`); `upa!(name: "UPA #{SecureRandom.hex(3)}")`; `upa_reception!` (`citizen_verifier`); `upa_nurse!(unit, cbo: "223505")`; `upa_doctor!(unit, cbo: "225125")`; `urgency_admin!` (`municipal_admin` com `mfa_verified_at` recente); `urgency_protocol!(rules: RULES, levels: LEVELS, version: 1)` (gravação direta `status: "active"`, como `acolhimento!`); `urgency_patient!(n, cpf: true)` (pelo `Patients::CreateTyped`, CPF/CNS válidos por `n`); `arrived!(unit, patient: nil, unidentified: nil, by: upa_reception!)` (pelo `Urgency::Arrive`, devolve o encontro); `classified!(encounter, color:, by:, vital_signs: {}, pain_scale: 0, justification: nil)` (pelo `Urgency::Classify`); `called!(encounter, by:)` (pelo `Attendances::Call`); `urgency_consultation!(encounter, doctor:)` e `finalized_urgency!(encounter, doctor:)` (pelos comandos do 19a); `call_location!(unit, name: "Consultório 1")`; `paired_display!(unit, show_wait_times: true) -> [display, token]` (pelos comandos da Task 12).

```ruby
LEVELS = Urgency::Catalog::COLORS.zip([ 0, 10, 60, 120, 240 ]).map { |c, m| { "color" => c, "max_wait_minutes" => m } }.freeze
RULES = [
  { "id" => "spo2-critica", "when" => { "lt" => [ "vitals.spo2", 90 ] }, "color" => "red", "criterion" => "SpO2 abaixo de 90%" },
  { "id" => "dor-intensa", "when" => { "gte" => [ "vitals.pain_score", 8 ] }, "color" => "orange", "criterion" => "Dor intensa (8 a 10)" },
  { "id" => "febre-alta", "when" => { "gte" => [ "vitals.temperature_c", 39 ] }, "color" => "yellow", "criterion" => "Febre de 39 °C ou mais" },
  { "id" => "dor-moderada", "when" => { "gte" => [ "vitals.pain_score", 4 ] }, "color" => "green", "criterion" => "Dor moderada (4 a 7)" }
].freeze
```

- [ ] **Step 1: Write the failing test** `spec/models/urgency_schema_spec.rb` (gravações diretas, sem os comandos): encontro não volta de `classified` para `waiting_classification`; `current_classification_id` de outro encontro → `/classification of another encounter/`; classificação com cor diferente da sugerida e sem justificativa → `/ck_urgency_classifications_justification/`; sem sugestão e sem justificativa → mesma recusa; classificação e desfecho não mudam (`update_column` → erro) e não somem; dois desfechos para o mesmo encontro → `RecordNotUnique`; `transferred` sem destino → CHECK; `attendance_calls` só acréscimo; dois locais "consultório 1"/"Consultório 1" na mesma unidade → `/idx_call_locations_name/`; `UrgencyClassification#as_json == { "id" => … }` e `inspect` sem a queixa.
- [ ] **Step 2:** `bin/test api -w mod32a spec/models/urgency_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** as duas migrações (constantes espelhadas no topo — `COLORS`, `STATES`, `OUTCOMES`, `CALL_KINDS`, `TABLES` —, `execute` do triggers no fim, irreversíveis), os modelos, os alvos de cifra (`[UrgencyClassification, :complaint]`, `:vital_signs`, `:criterion`, `:justification`, `[UrgencyOutcome, :destination]`, `:vacancy_request`, `:return_reason`) e os helpers (os que dependem de comandos de tasks seguintes ficam no arquivo desde já; só são chamados pelos specs daquelas tasks).
- [ ] **Step 4:** o spec + `spec/architecture/city_encrypted_attributes_guard_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add urgency encounter and waiting room schema with immutability triggers`.

---

### Task 5: Protocolo `urgency_classification` no motor do 18, sugestão e a UPA fora da escuta (transversal, §3)

**Files:**
- Create: `app/protocols/validation/urgency.rb`, `app/services/urgency/{active_protocol,levels,suggest}.rb`, `spec/protocols/validation/urgency_spec.rb`, `spec/services/urgency/suggest_spec.rb`, `spec/requests/urgency/protocol_lifecycle_spec.rb`
- Modify: `config/protocols/schema.json` (cópia byte a byte de `contracts` `protocols-v1.7.0:protocols/schema.json`), `app/protocols/{validator,gate}.rb`, `app/models/protocol_definition.rb`, `app/services/screenings/scope.rb`, `app/commands/screenings/start.rb`, `app/controllers/screenings_controller.rb`, `app/controllers/authoring/protocols_controller.rb`

**Interfaces:**
- Consumes: contrato §3 (só o formato; a tag é do contracts).
- Produces: `Protocols::Validation::Urgency.{urgency?(definition), call(definition) -> Array<String>, reserved_name_errors(definition)}` (Valor fixado 3); `Protocols::Validator` com `urgency_result` (forma mínima no save: `name`, `version`, `levels` e `rules` arrays); `Protocols::Gate` desvia para `Validation::Urgency.call` depois do schema; o gate de triagem e o de escuta somam `Validation::Urgency.reserved_name_errors`; `ProtocolDefinition.urgency_protocols` (`definition->>'kind' = 'urgency_classification'`) e `triage_protocols` = `definition->>'kind' IS NULL` (**transversal**: o protocolo de urgência nunca entra no catálogo de triagem, em `StartTriage` nem em `Triages::Offer`); `POST /authoring/protocols/preview` de urgência → 422 `"preview is not available for urgency protocols"`; `Urgency::ActiveProtocol.current -> ProtocolDefinition|nil`; `Urgency::Levels.for(protocol) -> Hash{color => Integer}` (o padrão sem protocolo); `Urgency::Suggest.call(vitals:, pain_scale:, profile:, protocol: ActiveProtocol.current) -> { color:, criterion:, rule_id: } | nil` (Desvio 10; puro sobre a definição); `Screenings::Scope::REQUIRED_SQL` com `AND scope_units.kind <> 'upa'` e `required?` falso numa UPA; `Screenings::Start` numa UPA → `Result.fail(:screening_not_in_upa)` → 422.

- [ ] **Step 1: Write the failing tests.**

```ruby
RSpec.describe Urgency::Suggest do
  let(:protocol) { ProtocolDefinition.new(definition: { "kind" => "urgency_classification", "levels" => UrgencyHelpers::LEVELS, "rules" => UrgencyHelpers::RULES }) }

  it "a mais grave entre as que casam, com o critério e a regra" do
    expect(described_class.call(vitals: { "spo2" => 88, "temperature_c" => BigDecimal("39.5") }, pain_scale: 9, profile: {}, protocol: protocol))
      .to eq(color: "red", criterion: "SpO2 abaixo de 90%", rule_id: "spo2-critica")
    expect(described_class.call(vitals: {}, pain_scale: 5, profile: {}, protocol: protocol))
      .to eq(color: "green", criterion: "Dor moderada (4 a 7)", rule_id: "dor-moderada")
  end

  it "nada casa → nil; sem protocolo → nil; a dor entra como vitals.pain_score" do
    expect(described_class.call(vitals: { "spo2" => 98 }, pain_scale: 0, profile: {}, protocol: protocol)).to be_nil
    expect(described_class.call(vitals: { "spo2" => 80 }, pain_scale: 0, profile: {}, protocol: nil)).to be_nil
    expect(described_class.call(vitals: { "pain_score" => 10 }, pain_scale: 0, profile: {}, protocol: protocol)).to be_nil
  end
end
```

`spec/protocols/validation/urgency_spec.rb`: definição válida (nome reservado, 5 níveis em ordem, regras) → `[]`; nível faltando, cor fora de ordem, `max_wait_minutes` decrescente (`"levels must not decrease"`), `id` repetido, `criterion` vazio, variável `complaint.ciap2` no `when` → mensagens com o índice; triagem com o nome `classificacao-urgencia` → `reserved_name_errors` não vazio; `Protocols::Gate.call` de um exemplo válido do contracts (`spec/fixtures` não: lido de `config/protocols/schema.json` + definição inline) → válido.

`spec/requests/urgency/protocol_lifecycle_spec.rb` (o ciclo real do 18 com o kind novo): rascunho → revisão → duas assinaturas → publicação → ativação pelas rotas existentes (`/authoring/protocols/draft`, `/protocol_lifecycle/...`, helpers de `spec/support/protocol_signatures.rb`); `Urgency::ActiveProtocol.current` é a versão ativa; ativar a v2 aposenta a v1; `ProtocolDefinition.triage_protocols` não a contém; `Screenings::ActiveProtocol.current` segue nil. Escuta numa UPA: `POST /attendance/attendances/:id/screening` → 422 `screening_not_in_upa`; `GET /attendance/units/:upa/screening_queue` → `[]`.

Review Focus 4 (fica nesta task o snapshot de níveis; o uso pela fila é da Task 8): `Urgency::Levels.for(v1)["yellow"] == 60`, `Levels.for(v2)["yellow"] == 30`, `Levels.for(nil) == Urgency::Catalog::DEFAULT_MAX_WAIT`.

- [ ] **Step 2:** `bin/test api -w mod32a spec/protocols/validation/urgency_spec.rb spec/services/urgency/suggest_spec.rb spec/requests/urgency/protocol_lifecycle_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Copiar o schema da tag (`git -C …/contracts show protocols-v1.7.0:protocols/schema.json > config/protocols/schema.json`); o `Validation::Schema` relê o arquivo (`@schemer` é por processo). Os demais passos seguem os Interfaces.
- [ ] **Step 4:** os mesmos specs + `spec/protocols spec/services/screenings spec/requests/screenings_spec.rb spec/requests/triage_catalog_spec.rb spec/commands/start_triage_spec.rb spec/requests/authoring` → PASS.
- [ ] **Step 5: Commit** `feat: accept the urgency classification protocol kind in the signed protocol engine`.

---

### Task 6: Busca de pessoa e chegada na UPA (`Urgency::Arrive`, §4)

**Files:**
- Create: `app/services/urgency/{access,trail,patient_ref,json}.rb`, `app/commands/urgency/arrive.rb`, `app/controllers/urgency/{base,person_lookups,arrivals}_controller.rb`, `spec/requests/urgency/arrivals_spec.rb`
- Modify: `config/routes.rb` (`scope "/urgency", module: :urgency, as: :urgency`), `config/initializers/domain_events.rb` + bindings spec (`urgency.arrived`, `to: []`)

**Interfaces:**
- Consumes: Task 2 (`Attendances::Ticket.issue`, `idx_attendances_open_walk_in`), Task 3 (provisório), Task 4 (modelos).
- Produces: `Urgency::BaseController` (no padrão do `Tfd::BaseController`: `render_failure`, `render_refusal`, `forbid`, `not_found`, `uuid?`, `body`, `body_param`, `query_only!`; `FAILURE_STATUS`: `missing_role`/`missing_link`/`cbo_not_allowed` → 403, `not_found` → 404, `encounter_open`/`already_identified`/`merge_blocked`/`cpf_taken`/`cns_taken`/`citizen_not_verified`/`already_closed`/`consultation_in_progress` → 409, o resto 422; `before_action :require_urgency!`); `Urgency::Access.{reception?(user), classifier_link(user, unit_id), physician_link(user, unit_id), clinical_link(user, unit_id)} -> ProfessionalLink|nil` (vínculo ativo na unidade, papel `health_professional`, CBO por prefixo do Valor fixado 2; `clinical_link` = um dos dois); `Urgency::Trail.viewed!(user:, patient_id:, resource:)`; `Urgency::PatientRef.call(patient, detail: false) -> { patient_id:, name:, cns_masked:, unidentified: }` (+ `description:` só com `detail: true` e provisório); `POST /urgency/person_lookup {cpf}|{cns}` (`citizen_verifier`; chave fora de `cpf`/`cns` → 422 com o nome truncado; 30/min por usuário; recusas de forma do `Patients::LookupByIdentifier`; `citizen_not_verified` → 409; `person_not_found` → 200 `{ patient: nil, candidates: [] }`; achou → `{ patient: <patient-ref>, candidates: [] }` + trilha `resource: "person_lookup"`); `Urgency::Arrive.call(health_unit_id:, patient_id: nil, new_patient: nil, unidentified: nil, by:, waiting_room:, now: Time.current) -> Result.ok(encounter:, attendance:, patient:)`; `POST /urgency/arrivals` → 201 `{ attendance_id, encounter_id, ticket, patient: <patient-ref> }`.
- Regras do `Arrive` (ordem): exatamente uma das três formas (zero ou duas → 422 `invalid_request` `path: "patient"`); unidade ativa (senão `invalid_unit`), `caps?` → `caps_unit_not_allowed`, não `upa?` → `not_an_upa`; `patient_id` de provisório ou fundido → `invalid_request` `path: "patient_id"`; `new_patient` por `Patients::CreateTyped.call` (dentro da transação, savepoint próprio dele); `unidentified.description` em `DESCRIPTION` (senão `invalid_request` `path: "unidentified.description"`) → `Patient.create!(no_cpf_reason: "unidentified", unidentified: true, unidentified_description:)`; transação: `HealthUnit.lock_active!`, `Patient.lock` do existente, encontro aberto do paciente na UPA → `encounter_open` (o índice único é a última palavra: `RecordNotUnique` em `idx_attendances_open_walk_in` → `encounter_open`), `Attendance.create!(origin_kind: "walk_in", check_in_method: "urgency_reception", …, ticket/ticket_on se `waiting_room`)`, `UrgencyEncounter.create!`, `CheckIn.publish(attendance)` e `DomainEvents.publish("urgency.arrived", encounter_id:)`.

- [ ] **Step 1: Write the failing test** `spec/requests/urgency/arrivals_spec.rb`:

```ruby
RSpec.describe "Chegada na UPA", type: :request do
  let!(:city) { urgency_city!(waiting_room: true) }
  let(:upa) { upa! }
  before { sign_in_as(upa_reception!) }

  it "paciente achado pelo CPF chega com senha; a mesma pessoa não abre dois encontros na UPA" do
    patient = urgency_patient!(1)
    json_post "/urgency/person_lookup", cpf: patient.cpf
    expect(json_body["patient"]).to include("patient_id" => patient.id, "unidentified" => false)
    expect(json_body["patient"].keys).to contain_exactly("patient_id", "name", "cns_masked", "unidentified")
    expect { json_post "/urgency/arrivals", health_unit_id: upa.id, patient_id: patient.id }
      .to change(UrgencyEncounter, :count).by(1).and change { DomainEvent.where(name: "urgency.arrived").count }.by(1)
    expect(response).to have_http_status(:created)
    expect(json_body).to include("ticket" => "U-001")
    expect(UrgencyEncounter.last.state).to eq("waiting_classification")
    json_post "/urgency/arrivals", health_unit_id: upa.id, patient_id: patient.id
    expect(response).to have_http_status(:conflict)
    expect(json_body["error"]).to eq("encounter_open")
  end

  it "não identificado: paciente provisório com descrição, sem CPF; nada da descrição no evento nem no log" do
    log = capture_log do
      json_post "/urgency/arrivals", health_unit_id: upa.id, unidentified: { description: "Mulher, cerca de 70 anos, trazida pelo SAMU" }
    end
    expect(json_body["patient"]).to include("unidentified" => true, "name" => "Paciente não identificado")
    expect(json_body["patient"]).not_to have_key("description")
    expect(Patient.find(json_body.dig("patient", "patient_id")).no_cpf_reason).to eq("unidentified")
    expect(log).not_to include("SAMU")
    expect(DomainEvent.where(name: %w[urgency.arrived attendance.checked_in]).map(&:payload).to_s).not_to include("SAMU")
  end

  it "paciente novo pelo 24a; recusas na ordem: forma, CAPS, não UPA" do
    json_post "/urgency/arrivals", health_unit_id: upa.id,
              new_patient: { full_name: "Joana Ribeiro Matos", birth_date: "1975-04-03", sex: "female", no_cpf_reason: "3" }
    expect(response).to have_http_status(:created)
    json_post "/urgency/arrivals", health_unit_id: upa.id, patient_id: urgency_patient!(2).id, unidentified: { description: "Duas formas juntas" }
    expect(json_body).to include("error" => "invalid_request", "path" => "patient")
    caps = HealthUnit.create!(name: "CAPS X", kind: "caps", caps_modality: "caps_ii")
    json_post "/urgency/arrivals", health_unit_id: caps.id, patient_id: urgency_patient!(3).id
    expect(json_body["error"]).to eq("caps_unit_not_allowed")
    json_post "/urgency/arrivals", health_unit_id: create_unit("UBS Y").id, patient_id: urgency_patient!(4).id
    expect(json_body["error"]).to eq("not_an_upa")
  end

  it "provisório não serve a outra chegada; busca pela query é recusada; interruptor desligado → 403" do
    provisional = arrived!(upa, unidentified: "Homem, 30 anos").patient
    json_post "/urgency/arrivals", health_unit_id: upa.id, patient_id: provisional.id
    expect(json_body["path"]).to eq("patient_id")
    post "/urgency/person_lookup?cpf=52998224725", params: {}.to_json, headers: { "CONTENT_TYPE" => "application/json" }
    expect(json_body["path"]).to eq("cpf")
    Platform::Features.set!(city: city, key: "urgency_care", enabled: false, maintainer: ledi_maintainer!)
    json_post "/urgency/arrivals", health_unit_id: upa.id, patient_id: urgency_patient!(5).id
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "urgency_care")
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod32a spec/requests/urgency/arrivals_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces. O controlador lê `WaitingRoom::Gate.usable?(Current.city)` e passa `waiting_room:`; `new_patient`/`unidentified` só do corpo; recusas do `CreateTyped` traduzidas com o `path` dele.
- [ ] **Step 4:** o spec + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add urgency reception with person lookup, ticket and provisional patient`.

---

### Task 7: Classificação, reclassificação e prévia da cor (§5)

**Files:**
- Create: `app/commands/urgency/classify.rb`, `app/controllers/urgency/{classifications,suggestions}_controller.rb`, `spec/requests/urgency/classifications_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`urgency.classified`, `urgency.reclassified`, `to: []`)

**Interfaces:**
- Consumes: Task 5 (`Urgency::Suggest`, `Urgency::Levels`, `Urgency::ActiveProtocol`), Task 6 (`Urgency::Access.classifier_link`).
- Produces: `Urgency::Classify.call(encounter:, attrs:, by:, now: Time.current) -> Result.ok(classification:, suggested:)`; `POST /urgency/encounters/:id/classifications {complaint, vital_signs, pain_scale, color, justification?}` → 201 `{ classification: { id, color, position, max_wait_minutes, classified_at }, suggested: { color, criterion } | nil }`; `POST /urgency/suggest {encounter_id, vital_signs, pain_scale}` → 200 `{ suggested: { color, criterion, rule_id } | nil }` (sem gravar, sem trilha).
- Regras: `classifier_link` na UPA do encontro (sem papel → 403 `missing_role`, sem vínculo → `missing_link`, CBO fora → `cbo_not_allowed`); encontro `closed` → 409 `already_closed`; `complaint` em `COMPLAINT` (`path: "complaint"`); `vital_signs` por `Screenings::VitalSigns.parse` (recusa → `invalid_request` `path: "vital_signs.<campo>"`; `pain_score` dentro de `vital_signs` → `path: "vital_signs.pain_score"`); `pain_scale` inteiro em `PAIN`; `color` em `COLORS`; cor ≠ sugerida (ou sem sugestão) e `justification` fora de `JUSTIFICATION` → 422 `justification_required` `path: "justification"`; profile = `{ age: patient.age, sex: patient.sex }` (provisório: vazio). Transação: `attendance.lock!` → `encounter.lock!`; grava a classificação (`position` = última + 1, `max_wait_minutes` = `Levels.for(protocol)[color]`, `protocol_definition_id`, `criterion` da sugestão quando a cor final é a sugerida, senão nulo); atualiza o encontro (`current_classification`, `first_classified_at ||= now`, `state` de `waiting_classification` para `classified`); evento `urgency.classified` (1ª) ou `urgency.reclassified` com `encounter_id`, `classification_id`.

- [ ] **Step 1: Write the failing test** (trechos):

```ruby
RSpec.describe "Classificação de risco", type: :request do
  let!(:city) { urgency_city! }
  let(:upa) { upa! }
  let(:nurse) { upa_nurse!(upa) }
  let!(:protocol) { urgency_protocol! }
  let(:encounter) { arrived!(upa, patient: urgency_patient!(1)) }
  before { sign_in_as(nurse) }

  it "aceita a cor sugerida sem justificativa e leva o encontro a classified" do
    json_post "/urgency/encounters/#{encounter.id}/classifications",
              complaint: "Falta de ar há 2 horas", vital_signs: { spo2: 87 }, pain_scale: 2, color: "red"
    expect(response).to have_http_status(:created)
    expect(json_body["suggested"]).to eq("color" => "red", "criterion" => "SpO2 abaixo de 90%")
    expect(json_body.dig("classification", "max_wait_minutes")).to eq(0)
    expect(encounter.reload).to have_attributes(state: "classified", color: "red")
  end

  it "trocar a cor sugerida exige justificativa de 10+ caracteres; sem protocolo, toda cor exige" do
    json_post "/urgency/encounters/#{encounter.id}/classifications", complaint: "Dor lombar", vital_signs: {}, pain_scale: 9, color: "yellow"
    expect(json_body).to include("error" => "justification_required", "path" => "justification")
    json_post "/urgency/encounters/#{encounter.id}/classifications", complaint: "Dor lombar", vital_signs: {}, pain_scale: 9,
              color: "yellow", justification: "Dor crônica conhecida, sem sinal de alarme"
    expect(response).to have_http_status(:created)
    protocol.update_columns(status: "retired")
    other = arrived!(upa, patient: urgency_patient!(2))
    json_post "/urgency/encounters/#{other.id}/classifications", complaint: "Tosse", vital_signs: {}, pain_scale: 0, color: "blue"
    expect(json_body["error"]).to eq("justification_required")
  end

  it "reclassificar acrescenta; a cor vigente é a última; o evento leva só ids" do
    classified!(encounter, color: "green", by: nurse, pain_scale: 5)
    json_post "/urgency/encounters/#{encounter.id}/classifications", complaint: "Piora da dor", vital_signs: {}, pain_scale: 9, color: "orange"
    expect(encounter.reload.classifications.map(&:color)).to eq(%w[green orange])
    event = DomainEvent.where(name: "urgency.reclassified").last
    expect(event.payload.keys).to contain_exactly("encounter_id", "classification_id")
  end

  it "Review Focus 4: o estouro usa o tempo gravado na classificação, não o do protocolo de agora" do
    travel_to(40.minutes.ago) { classified!(encounter, color: "yellow", by: nurse, justification: "Conferido pela enfermeira") }
    protocol.update_columns(status: "retired")
    urgency_protocol!(levels: UrgencyHelpers::LEVELS.map { |l| l["color"] == "yellow" ? l.merge("max_wait_minutes" => 30) : l }, version: 2)
    expect(encounter.reload.current_classification.max_wait_minutes).to eq(60)
    later = arrived!(upa, patient: urgency_patient!(3))
    classified!(later, color: "yellow", by: nurse, justification: "Conferido pela enfermeira")
    expect(later.reload.current_classification.max_wait_minutes).to eq(30)
  end

  it "médico e recepção não classificam; técnico de enfermagem (3222) classifica; prévia não grava" do
    sign_in_as(upa_doctor!(upa))
    json_post "/urgency/encounters/#{encounter.id}/classifications", complaint: "X", vital_signs: {}, pain_scale: 0, color: "blue"
    expect(json_body["error"]).to eq("cbo_not_allowed")
    sign_in_as(upa_nurse!(upa, cbo: "322205"))
    expect { json_post "/urgency/suggest", encounter_id: encounter.id, vital_signs: { temperature_c: "39,4" }, pain_scale: 0 }
      .not_to change(UrgencyClassification, :count)
    expect(json_body["suggested"]).to include("color" => "yellow", "rule_id" => "febre-alta")
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod32a spec/requests/urgency/classifications_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces (`complaint`, `vital_signs`, `justification` só do corpo).
- [ ] **Step 4:** o spec + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add urgency risk classification and reclassification`.

---

### Task 8: Fila por cor, "Chamar próximo" da UPA, chamada com local e "chamar de novo" (transversal 13, §6)

**Files:**
- Create: `app/services/urgency/queue.rb`, `app/services/waiting_room/call_locations.rb`, `app/services/attendances/call_record.rb`, `app/commands/urgency/call_for_classification.rb`, `app/commands/attendances/recall.rb`, `app/controllers/urgency/queue_controller.rb`, `spec/services/urgency/queue_spec.rb`, `spec/requests/urgency/calls_spec.rb`
- Modify: `app/commands/attendances/{call,call_next}.rb`, `app/controllers/attendances_controller.rb` (`call`, `call_next` com `call_location_id`; `recall` novo; códigos `not_classified`, `call_location_required`, `invalid_call_location`, `not_called` → 422), `app/controllers/urgency/encounters_controller.rb` (`call_for_classification`), `config/routes.rb` (`post "attendances/:id/recall"` no `scope "/attendance"`), `config/initializers/domain_events.rb` + bindings spec (`attendance.recalled`, `to: []`)

**Interfaces:**
- Produces: `Urgency::Queue.groups(unit, now: Time.current) -> { awaiting_classification: [...], by_color: [...], in_care: [...] }` e `Urgency::Queue.lock_next_classified(unit_id) -> Attendance|nil` (mesma ordem em SQL: `CASE` de `COLORS` sobre a classificação vigente, depois `checked_in_at`, `id`; só `walk_in` `waiting` com encontro `classified`; `FOR UPDATE OF attendances SKIP LOCKED`); item = `{ encounter_id, attendance_id, ticket, patient: <patient-ref>, arrived_at, waited_minutes }` (+ `color`, `overdue`, `classified_at` no `by_color`; + `color`, `called_at`, `called_by_name` no `in_care`), `waited_minutes` = minutos inteiros desde `checked_in_at`, `overdue` pelo Desvio 9; `GET /urgency/queue?health_unit_id=` (recepção ou `clinical_link`; sem trilha; `query_only!("health_unit_id")`); `WaitingRoom::CallLocations.resolve(unit_id:, call_location_id:, enabled:) -> [:ok, CallLocation|nil] | [:call_location_required, nil] | [:invalid_call_location, nil]` (Desvio 12); `Attendances::CallRecord.record!(attendance:, kind:, location:, by:, enabled:) -> AttendanceCall|nil`; `Attendances::Call.call(attendance:, health_unit_id:, by:, call_location_id: nil, waiting_room: false)` (+ Desvio 11 para `walk_in`; o encontro vai a `in_care`; grava a chamada `care`); `Attendances::CallNext.call(health_unit_id:, by:, call_location_id: nil, waiting_room: false, urgency: false)` (`urgency && unit.upa?` → `Urgency::Queue.lock_next_classified`); `Urgency::CallForClassification.call(encounter:, call_location_id:, by:, waiting_room:) -> Result.ok(call: AttendanceCall|nil)` (`classifier_link`; encontro fechado → `already_closed`; estado não muda); `POST /urgency/encounters/:id/call_for_classification` → `{ call: { id, ticket, location, called_at } | null }`; `Attendances::Recall.call(attendance:, call_location_id:, by:)` (`WaitingRoom` utilizável, senão 403 `feature_disabled`; atendimento aberto; `kind` = o da última chamada, sem chamada anterior → 422 `not_called`; `ClinicalAuthorization`; evento `attendance.recalled` com `attendance_id`, `call_id`); `POST /attendance/attendances/:id/recall {call_location_id}` → 201 `{ call: {...} }`. Os controladores leem os dois portões fora da transação e passam `waiting_room:`/`urgency:`.

- [ ] **Step 1: Write the failing tests.** `spec/services/urgency/queue_spec.rb` (Review Focus 3):

```ruby
RSpec.describe Urgency::Queue do
  let!(:city) { urgency_city! }
  let(:upa) { upa! }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }
  before { urgency_protocol! }

  def classified_at(minutes_ago, color, n)
    travel_to(minutes_ago.minutes.ago) do
      arrived!(upa, patient: urgency_patient!(n)).tap { |e| classified!(e, color: color, by: nurse, justification: "Conferido na classificação") }
    end
  end

  it "ordena por cor e chegada; reclassificar para cima sobe e marca o estouro pelo relógio da 1ª classificação" do
    yellow = classified_at(55, "yellow", 1)
    green = classified_at(50, "green", 2)
    red = classified_at(5, "red", 3)
    waiting = arrived!(upa, patient: urgency_patient!(4))
    groups = described_class.groups(upa)
    expect(groups[:by_color].map { |i| i[:encounter_id] }).to eq([ red.id, yellow.id, green.id ])
    expect(groups[:awaiting_classification].map { |i| i[:encounter_id] }).to eq([ waiting.id ])
    classified!(green, color: "orange", by: nurse, justification: "Piora da dor torácica")
    items = described_class.groups(upa)[:by_color]
    expect(items.map { |i| i[:encounter_id] }).to eq([ red.id, green.id, yellow.id ])
    expect(items.find { |i| i[:encounter_id] == green.id }).to include(color: "orange", overdue: true)
    expect(items.find { |i| i[:encounter_id] == yellow.id }[:overdue]).to be(false)
    called = ApplicationRecord.transaction { described_class.lock_next_classified(upa.id) }
    expect(called.id).to eq(red.attendance_id)
  end

  it "nenhum item traz texto clínico" do
    e = classified_at(1, "green", 5)
    expect(described_class.groups(upa).to_s).not_to include(e.current_classification.complaint)
  end
end
```

`spec/requests/urgency/calls_spec.rb`: "Chamar próximo" com só não classificados → 404 `queue_empty`; médico chama o primeiro por cor e o encontro vai a `in_care`; `Call` de não classificado → 422 `not_classified`; enfermeira tentando `Call` de cuidado → 403 `cbo_not_allowed`; com `waiting_room_display` ligado e TV pareada na UPA, `call_next` sem local → 422 `call_location_required`, local de outra unidade → `invalid_call_location`, com local → `AttendanceCall` `kind: "care"` com a senha; `call_for_classification` grava `kind: "classification"` e o atendimento segue `waiting`; `recall` grava outra chamada com o mesmo `kind` e o estado não muda, sem chamada anterior → `not_called`, `waiting_room_display` desligado → 403; numa UBS com TV, `Call` do 13 também exige o local e grava a chamada; com o interruptor desligado, nenhuma `AttendanceCall` e `call_location_id` ignorado.

- [ ] **Step 2:** `bin/test api -w mod32a spec/services/urgency/queue_spec.rb spec/requests/urgency/calls_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces. A ordem mora no SQL (uma definição para a tela e para o lock, como o `UnitQueue`).
- [ ] **Step 4:** os specs + `spec/commands/attendances spec/requests/attendances_spec.rb spec/services/attendances` → PASS.
- [ ] **Step 5: Commit** `feat: order the upa queue by colour and record calls with location and recall`.

---

### Task 9: Identificação do não identificado com fusão única (R4)

**Files:**
- Create: `app/services/urgency/merge.rb`, `app/commands/urgency/identify.rb`, `spec/commands/urgency/identify_spec.rb`, `spec/architecture/urgency_merge_tables_spec.rb`
- Modify: `app/controllers/urgency/encounters_controller.rb` (`identify`), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`urgency.patient_identified`, `to: [ Billing::UrgencyProductionJob ]` — a Task 13 cria o job; até lá `to: []` e a Task 13 troca)

**Interfaces:**
- Consumes: Task 3 (marca e guardas), Task 6 (`Patients::CreateTyped`, `PatientRef`).
- Produces: `Urgency::Merge::TABLES`, `::KEPT` (Desvio 5); `Urgency::Merge.blocking_tables -> Array<String>` (as tabelas da cidade com coluna `patient_id`, lidas de `ApplicationRecord.connection`, menos `TABLES`, `KEPT` e `patients`); `Urgency::Merge.with_marker { }`; `Urgency::Merge.call(provisional:, into:) -> Result.ok(moved: { table => Integer }) | fail(:merge_blocked)` (chame com os dois pacientes travados; grava `merged_into` e só então faz `UPDATE <tabela> SET patient_id = into.id WHERE patient_id = provisional.id` para cada tabela de `TABLES`, sob a marca); `Urgency::Identify.call(encounter:, patient_id: nil, new_patient: nil, by:) -> Result.ok(patient:, merged_from_id: uuid|nil)`; `POST /urgency/encounters/:id/identify {patient_id} | {new_patient}` (`citizen_verifier`) → 200 `{ patient: <patient-ref>, merged: bool }`.
- Regras: exatamente uma forma (senão `invalid_request` `path: "patient"`); travas na ordem provisório → alvo (por id crescente quando os dois existem); encontro de paciente não provisório, ou provisório já identificado/fundido → 409 `already_identified`; `patient_id` inexistente → 404; alvo provisório ou fundido → `invalid_request` `path: "patient_id"`; alvo com encontro aberto na mesma UPA → 409 `encounter_open` (antes de mover); bloqueante com linha do provisório → 409 `merge_blocked`; `new_patient` com CPF/CNS existentes → 409 `cpf_taken`/`cns_taken`; `new_patient` novo → o provisório recebe os dados (`full_name`, `social_name`, `birth_date`, `sex`, `mother_name`, `cns`; CPF por `Patients::AddCpf` ou `no_cpf_reason` do 24a) e `unidentified: false`, mesma transação; evento `urgency.patient_identified` com `encounter_id`, `patient_id` e `merged_from_id` (só na fusão).

- [ ] **Step 1: Write the failing tests** `spec/commands/urgency/identify_spec.rb` (os que pegam fusão errada):

```ruby
RSpec.describe Urgency::Identify do
  let!(:city) { urgency_city! }
  let(:upa) { upa! }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }
  let(:real) { urgency_patient!(1) }
  let(:encounter) { arrived!(upa, unidentified: "Homem, 50 anos, desacordado na praça") }
  let(:provisional) { encounter.patient }

  def identify(**args) = described_class.call(encounter: encounter.reload, by: reception!, **args)

  it "funde: encontro, consulta, documento, exame e BPA passam ao real; nada do real nem de outro provisório muda" do
    classified!(encounter, color: "red", by: nurse, vital_signs: { spo2: 85 })
    called!(encounter, by: doctor)
    consultation = urgency_consultation!(encounter, doctor: doctor)
    exam_order = exam_order_for!(consultation)                  # helper do 21 sobre a consulta
    other = arrived!(upa, unidentified: "Mulher, 20 anos")
    real_consultation_ids = Consultation.where(patient_id: real.id).pluck(:id)
    result = identify(patient_id: real.id)
    expect(result).to be_ok
    expect(result.payload[:merged_from_id]).to eq(provisional.id)
    expect(encounter.attendance.reload.patient_id).to eq(real.id)
    expect(consultation.reload.patient_id).to eq(real.id)
    expect(exam_order.reload.patient_id).to eq(real.id)
    expect(provisional.reload).to have_attributes(merged_into_id: real.id, unidentified: true)
    expect(other.attendance.reload.patient_id).to eq(other.patient.id)
    expect(Consultation.where(patient_id: real.id).pluck(:id)).to match_array(real_consultation_ids + [ consultation.id ])
    Urgency::Merge::TABLES.each { |t| expect(ApplicationRecord.connection.select_value("SELECT count(*) FROM #{t} WHERE patient_id = '#{provisional.id}'")).to eq(0) }
  end

  it "uma vez só: a segunda identificação, para o mesmo ou outro paciente, é 409 e não move nada" do
    expect(identify(patient_id: real.id)).to be_ok
    second = urgency_patient!(2)
    expect(identify(patient_id: second.id).reason).to eq(:already_identified)
    expect(identify(new_patient: { "full_name" => "Fulano de Tal", "birth_date" => "1970-01-01", "sex" => "male", "no_cpf_reason" => "3" }).reason)
      .to eq(:already_identified)
    expect(encounter.attendance.reload.patient_id).to eq(real.id)
  end

  it "nunca funde em provisório, em fundido, em quem está com encontro aberto na mesma UPA, nem pelo CPF digitado" do
    other = arrived!(upa, unidentified: "Outro provisório")
    expect(identify(patient_id: other.patient.id).details[:path]).to eq("patient_id")
    busy = urgency_patient!(3)
    arrived!(upa, patient: busy)
    expect(identify(patient_id: busy.id).reason).to eq(:encounter_open)
    expect(identify(new_patient: { "full_name" => "Carlos Eduardo Pinto", "birth_date" => "1980-02-01", "sex" => "male", "cpf" => real.cpf }).reason)
      .to eq(:cpf_taken)
    expect(provisional.reload.merged_into_id).to be_nil
  end

  it "bloqueia a fusão quando o provisório tem registro fora das listas (ex.: vacina)" do
    immunization_for!(provisional)                              # helper do 29, gravação direta
    expect(identify(patient_id: real.id).reason).to eq(:merge_blocked)
    expect(encounter.attendance.reload.patient_id).to eq(provisional.id)
  end

  it "paciente novo: o provisório recebe os dados e deixa de ser provisório, sem fundir" do
    result = identify(new_patient: { "full_name" => "Benedito Ramos Filho", "birth_date" => "1960-07-07", "sex" => "male", "no_cpf_reason" => "3" })
    expect(result.payload).to include(merged_from_id: nil)
    expect(provisional.reload).to have_attributes(unidentified: false, full_name: "Benedito Ramos Filho", merged_into_id: nil)
  end

  it "Review Focus 1: duas identificações ao mesmo tempo, uma vence e a outra recebe already_identified" do
    second = urgency_patient!(4)
    results = Queue.new
    threads = [ real, second ].map do |target|
      Thread.new do
        CityConnection.with(city) { results << described_class.call(encounter: UrgencyEncounter.find(encounter.id), patient_id: target.id, by: reception!).reason }
      end
    end
    reasons = [ results.pop(timeout: 10), results.pop(timeout: 10) ]
    expect(reasons).to contain_exactly(nil, :already_identified)
    winner = provisional.reload.merged_into_id
    expect(Attendance.find(encounter.attendance_id).patient_id).to eq(winner)
  ensure
    threads&.each { |t| t.join(10) }
  end
end
```

`spec/architecture/urgency_merge_tables_spec.rb`: toda tabela da cidade com `patient_id` está em `TABLES`, em `KEPT` ou em `blocking_tables`; `TABLES & KEPT` vazio; `TABLES` não contém `citizens` nem `patients`; cada tabela de `TABLES` com guarda tem a chamada `rota_patient_merge_ok` no `db/city_triggers.sql` (leitura do arquivo).

O spec de requisição (`spec/requests/urgency/identify_spec.rb`, `citizen_verifier`) cobre a forma do corpo, o 404 e o evento com `merged_from_id` só na fusão (chaves `encounter_id`, `patient_id`, `merged_from_id`).

- [ ] **Step 2:** `bin/test api -w mod32a spec/commands/urgency/identify_spec.rb spec/architecture/urgency_merge_tables_spec.rb spec/requests/urgency/identify_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces (`exam_order_for!` e `immunization_for!` ficam no `urgency_helpers.rb` sobre os helpers existentes de `exam_helpers.rb`/`vaccination_helpers.rb`; o executor confere os nomes reais).
- [ ] **Step 4:** os specs + `spec/models/urgency_patient_merge_guard_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: identify provisional urgency patients with a single audited merge`.

---

### Task 10: Atendimento médico como consulta 19a `kind: urgency`, fora do LEDI (transversal 19a, §7)

**Files:**
- Create: `db/city_migrate/20261105900005_add_urgency_consultation_kind.rb`, `spec/requests/urgency/consultation_spec.rb`
- Modify: `db/city_schema.rb`, `app/models/consultation.rb` (`KINDS = %w[general dental urgency]`, `#urgency?`), `app/services/consultations/{cbos,items_input}.rb`, `app/commands/consultations/{start,finalize,save_draft}.rb`, `app/services/ledi/consultation_ficha.rb`, `app/services/clinical_record/access.rb`, `app/controllers/clinical_records_controller.rb`, `app/commands/clinical_documents/issue.rb`, `spec/models/consultation_tables_guard_spec.rb`

**Interfaces:**
- Produces (banco): `ck_consultations_kind` + `urgency`; `ck_consultations_care_type` + `OR (kind = 'urgency' AND care_type IS NULL)` (na forma: `care_type IS NULL OR (general …) OR (dental …)` já aceita nulo; a urgência nunca grava tipo — `kind <> 'urgency' OR care_type IS NULL`); `ck_consultations_finalized_care_type` = `status <> 'finalized' OR care_type IS NOT NULL OR baseline OR kind = 'urgency'`.
- Produces (Ruby): `Consultations::Cbos.kinds_for(cbo)` devolve `%w[general urgency]` para CBO 225\* da tabela do MIAI (o resto igual); `Consultations::Start.call(attendance:, by:, kind: nil, baseline: false, city:)` (Desvio 14; o encontro recebe `consultation_id` na mesma transação); `Finalize` (Desvio 14) devolve `Result.ok(consultation:, attendance:)` com o atendimento ainda `in_care`; `SaveDraft` da urgência recusa `care_type` (`path: "care_type"`); `ItemsInput#exams` com `priority` padrão `urgent` na urgência; `Ledi::ConsultationFicha.excluded?(consultation)` (CAPS ou `urgency?`) no lugar do `caps?` em `generate` e no `ConsultationFichaJob.enqueue_for`; `ClinicalRecord::Access`: `in_context?` de um `walk_in` compara `attendance.patient_id` com o paciente (sem par), `open_attendances_of(patient)` e `for_person` incluem `Attendance.open_attendances.where(origin_kind: "walk_in", patient_id: …)`; `ClinicalRecordsController#context` de `walk_in` usa `attendance.patient` (sem a recusa `citizen_not_verified`); `ClinicalDocuments::Issue`: declaração de `walk_in` lê `attendance.patient`, `replaced` de `walk_in` compara `patient_id` (nunca `nil == nil`).

- [ ] **Step 1: Write the failing test** `spec/requests/urgency/consultation_spec.rb`:

```ruby
RSpec.describe "Atendimento médico na UPA", type: :request do
  let!(:city) { urgency_city! }
  let(:upa) { upa! }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }
  let(:encounter) { arrived!(upa, unidentified: "Mulher, 30 anos, crise convulsiva").tap { |e| classified!(e, color: "orange", by: nurse, pain_scale: 8) } }
  before { called!(encounter, by: doctor); sign_in_as(doctor) }

  it "abre a consulta urgency do provisório, finaliza sem fechar o atendimento e sem LEDI" do
    json_post "/attendance/attendances/#{encounter.attendance_id}/consultation", {}
    expect(response).to have_http_status(:created)
    consultation = Consultation.find(json_body.dig("consultation", "id"))
    expect(consultation).to have_attributes(kind: "urgency", patient_id: encounter.patient.id, care_type: nil)
    expect(encounter.reload.consultation_id).to eq(consultation.id)
    json_patch "/attendance/consultations/#{consultation.id}", draft_body(exam_requests: [ { "sigtap_code" => "0202010473" } ]).except("care_type")
    expect(consultation.reload.draft_items["exam_requests"].first["priority"]).to eq("urgent")
    expect { json_post "/attendance/consultations/#{consultation.id}/finalize", {} }
      .not_to have_enqueued_job(Ledi::ConsultationFichaJob)
    expect(consultation.reload).to be_finalized
    expect(encounter.attendance.reload.status).to eq("in_care")
    json_post "/attendance/consultations/#{consultation.id}/finalize", outcome: "discharged"
    expect(response).to have_http_status(:conflict).or have_http_status(:unprocessable_entity)
  end

  it "kind errado e consulta urgency fora da UPA são recusados" do
    json_post "/attendance/attendances/#{encounter.attendance_id}/consultation", kind: "general"
    expect(json_body).to include("error" => "invalid_request", "path" => "kind")
    ubs = create_unit("UBS Z")
    attendance = consulting_attendance!(ubs, citizen: verified_citizen!(1), doctor: doctor!(ubs))
    sign_in_as(attendance.called_by_user)
    json_post "/attendance/attendances/#{attendance.id}/consultation", kind: "urgency"
    expect(json_body["path"]).to eq("kind")
  end

  it "prontuário em contexto pelo walk_in; vigilância sugere notificação pelo CID; atestado do provisório sai com o nome de tela" do
    get "/attendance/attendances/#{encounter.attendance_id}/record"
    expect(json_body.dig("patient", "display_name")).to eq("Paciente não identificado")
    consultation = urgency_consultation!(encounter, doctor: doctor)
    json_patch "/attendance/consultations/#{consultation.id}",
               draft_body(evaluated_problems: [ { "action" => "add", "terminology" => "cid10", "code" => "A90" } ]).except("care_type")
    get "/attendance/consultations/#{consultation.id}/notification_suggestions"
    expect(json_body["items"].map { |i| i["code"] }).to include("A90")
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { days: 1 }
    expect(response).to have_http_status(:created)
  end
end
```

`consultation_tables_guard_spec.rb`: a consulta `urgency` finalizada sem `care_type` é aceita; `urgency` com `care_type` → `/ck_consultations_care_type/`. Exemplo em `spec/services/ledi/consultation_ficha_spec.rb`: `generate` de consulta `urgency` → `:skipped`.

- [ ] **Step 2:** `bin/test api -w mod32a spec/requests/urgency/consultation_spec.rb spec/models/consultation_tables_guard_spec.rb spec/services/ledi/consultation_ficha_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`replace_check`, irreversível) e as mudanças dos Interfaces. Nomes reais das rotas, do corpo do rascunho e do código notificável de dev: conferir em `spec/requests/consultations_spec.rb` e `spec/support/surveillance_helpers.rb` antes de fixar os valores do teste.
- [ ] **Step 4:** os specs + `spec/commands/consultations spec/requests/consultations_spec.rb spec/requests/clinical_records_spec.rb spec/services/clinical_record spec/commands/clinical_documents spec/jobs/ledi spec/services/consultations` → PASS.
- [ ] **Step 5: Commit** `feat: record the upa physician care as an urgency consultation outside ledi`.

---

### Task 11: Desfecho, mapeamento para o 13, fechamento exigindo desfecho e contrarreferência (transversal 13, §7)

**Files:**
- Create: `app/commands/urgency/record_outcome.rb`, `app/services/urgency/counter_referrals.rb`, `app/controllers/urgency/counter_referrals_controller.rb`, `spec/requests/urgency/outcomes_spec.rb`
- Modify: `app/commands/attendances/close.rb`, `app/controllers/attendances_controller.rb` (`urgency_outcome_required` → 422), `app/controllers/urgency/encounters_controller.rb` (`outcome`), `db/city_triggers.sql` (transição `in_care → closed` com `left` do Desvio 13), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`urgency.encounter_closed`, `to: []` — a Task 13 troca para o job)

**Interfaces:**
- Produces: `Attendances::Close.call(…, urgency_outcome: nil)`: atendimento com encontro e sem `urgency_outcome` do MESMO encontro → `Result.fail(:urgency_outcome_required)`; com ele, `outcome`, `referral_note` e `referral_regulated` vêm do `Urgency::Catalog::OUTCOME_MAP` (Desvio 13), e `left` vale também de `in_care`; `Urgency::RecordOutcome.call(encounter:, attrs:, by:, team: nil) -> Result.ok(outcome:, attendance:)`; `POST /urgency/encounters/:id/outcome {kind, destination?, vacancy_request?, return_guidance?: {days, reason}}` → 201 `{ outcome: { id, kind, recorded_at }, attendance: { id, status, outcome } }`; `Urgency::CounterReferrals.for(unit:, now:) -> Array<Hash>`; `GET /urgency/counter_referrals?health_unit_id=` (health_professional com vínculo na UBS; `{ items: [ { encounter_id, patient: <patient-ref>, upa: { name }, closed_at, return_within_days, due_on } ] }`, últimos 60 dias, sem trilha, sem o motivo).
- Regras do `RecordOutcome`: `kind` em `OUTCOMES`; `transferred` exige `destination` em `DESTINATION` (`path: "destination"`), `vacancy_request` até 500 só nele; `discharged_to_primary_care` exige `return_guidance` (`days` em `RETURN_DAYS`, `reason` em `RETURN_REASON`; `path: "return_guidance.days"`/`.reason`); campo de outro kind → `invalid_request` com o path; autor: `left_without_care` por recepção ou `clinical_link`, os demais por `physician_link` (403); `left_without_care` em qualquer estado antes de `closed`; os demais exigem consulta `finalized` (422 `consultation_not_finalized`); `IDENTIFIED_OUTCOMES` com provisório → 422 `patient_unidentified`; encontro fechado → 409 `already_closed`; consulta em rascunho com `left_*` → 409 `consultation_in_progress` (a do Close). Equipe da contrarreferência lida FORA da transação pelo controlador: `CareLines::TeamPopulation.patient_teams([patient.id])[patient.id]` → `HealthTeam` → unidade (sem equipe → os dois nulos). Transação: `attendance.lock!` → `encounter.lock!` → grava o desfecho → `encounter.update!(state: "closed")` → `Attendances::Close.call(..., urgency_outcome:)` → `urgency.encounter_closed` com `encounter_id`.

- [ ] **Step 1: Write the failing test** (trechos):

```ruby
RSpec.describe "Desfecho do atendimento de urgência", type: :request do
  let!(:city) { urgency_city! }
  let(:upa) { upa! }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }
  let(:patient) { urgency_patient!(1) }
  let(:encounter) { arrived!(upa, patient: patient).tap { |e| classified!(e, color: "green", by: nurse, pain_scale: 5) } }

  it "alta com retorno à UBS fecha o 13 como referred, sem o motivo em texto claro, e avisa a UBS da equipe" do
    ubs, team = team_for_patient!(patient)                     # equipe da ESF pela consulta finalizada (helper do 23)
    called!(encounter, by: doctor)
    finalized_urgency!(encounter, doctor: doctor)
    sign_in_as(doctor)
    json_post "/urgency/encounters/#{encounter.id}/outcome", kind: "discharged_to_primary_care",
              return_guidance: { days: 7, reason: "Reavaliar pressão arterial após ajuste" }
    expect(response).to have_http_status(:created)
    attendance = encounter.attendance.reload
    expect(attendance).to have_attributes(status: "closed", outcome: "referred", referral_note: "Retorno à UBS em até 7 dias")
    expect(encounter.reload.outcome).to have_attributes(counter_referral_health_team_id: team.id, counter_referral_health_unit_id: ubs.id)
    sign_in_as(doctor!(ubs))
    get "/urgency/counter_referrals", params: { health_unit_id: ubs.id }
    expect(json_body["items"].first).to include("encounter_id" => encounter.id, "return_within_days" => 7)
    expect(response.body).not_to include("pressão")
  end

  it "evasão vale antes e depois da chamada; os demais exigem a consulta finalizada" do
    sign_in_as(upa_reception!)
    json_post "/urgency/encounters/#{encounter.id}/outcome", kind: "left_without_care"
    expect(encounter.attendance.reload).to have_attributes(status: "closed", outcome: "left")
    other = arrived!(upa, patient: urgency_patient!(2)).tap { |e| classified!(e, color: "yellow", by: nurse, justification: "Conferido na classificação") }
    called!(other, by: doctor)
    sign_in_as(doctor)
    json_post "/urgency/encounters/#{other.id}/outcome", kind: "discharged"
    expect(json_body["error"]).to eq("consultation_not_finalized")
    json_post "/urgency/encounters/#{other.id}/outcome", kind: "left_without_care"
    expect(other.attendance.reload).to have_attributes(status: "closed", outcome: "left")
  end

  it "transferência exige destino e fecha como referred regulado; alta de provisório é recusada, óbito não" do
    called!(encounter, by: doctor)
    finalized_urgency!(encounter, doctor: doctor)
    sign_in_as(doctor)
    json_post "/urgency/encounters/#{encounter.id}/outcome", kind: "transferred"
    expect(json_body["path"]).to eq("destination")
    json_post "/urgency/encounters/#{encounter.id}/outcome", kind: "transferred", destination: "Hospital do Trabalhador", vacancy_request: "Vaga pedida às 14h"
    expect(encounter.attendance.reload).to have_attributes(outcome: "referred", referral_regulated: true, referral_note: nil)
    unknown = arrived!(upa, unidentified: "Homem idoso").tap { |e| classified!(e, color: "red", by: nurse, vital_signs: { spo2: 80 }) }
    called!(unknown, by: doctor)
    finalized_urgency!(unknown, doctor: doctor)
    json_post "/urgency/encounters/#{unknown.id}/outcome", kind: "discharged"
    expect(json_body["error"]).to eq("patient_unidentified")
    json_post "/urgency/encounters/#{unknown.id}/outcome", kind: "death"
    expect(response).to have_http_status(:created)
  end

  it "o fechamento direto do 13 numa UPA exige o desfecho do 32" do
    called!(encounter, by: doctor)
    sign_in_as(doctor)
    json_post "/attendance/attendances/#{encounter.attendance_id}/close", outcome: "discharged"
    expect(json_body["error"]).to eq("urgency_outcome_required")
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod32a spec/requests/urgency/outcomes_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces (o helper `team_for_patient!` vem dos helpers do 23 — conferir o nome real em `spec/support/care_line_helpers.rb`).
- [ ] **Step 4:** o spec + `spec/commands/attendances spec/requests/attendances_spec.rb spec/invariants/attendance_invariants_spec.rb spec/commands/consultations/finalize_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: close urgency encounters with an outcome mapped to the attendance and counter-referral`.

---

### Task 12: TV da sala de espera — locais, dispositivos, pareamento, token e painel (§8)

**Files:**
- Create: `app/services/waiting_room/board.rb`, `app/commands/waiting_room/{save_call_location,save_display,issue_pairing_code,pair,revoke}.rb`, `app/controllers/waiting_room/{base,displays,pairings,boards}_controller.rb`, `app/controllers/call_locations_controller.rb`, `spec/requests/waiting_room/displays_spec.rb`, `spec/requests/waiting_room/board_spec.rb`
- Modify: `config/routes.rb` (`scope "/waiting_room", module: :waiting_room`; no `scope "/attendance"`: `get/post "call_locations"`, `patch "call_locations/:id"`), `config/initializers/domain_events.rb` + bindings spec (`waiting_room.display_paired`, `waiting_room.display_revoked`, `to: []`)

**Interfaces:**
- Produces: `GET/POST/PATCH /attendance/call_locations[/:id] {health_unit_id, name, active}` (`municipal_admin`, escrita com step-up; `?health_unit_id=` obrigatório no GET; nome fora de 1..40 → `invalid_request` `path: "name"`; repetido → 409 `call_location_name_taken`; unidade CAPS → `caps_unit_not_allowed`); `GET/POST/PATCH /waiting_room/displays[/:id] {health_unit_id, name, active, show_wait_times}` (idem; `{ id, health_unit_id, name, active, show_wait_times, paired: bool, paired_at, revoked: bool }`; `active: false` revoga); `POST /waiting_room/displays/:id/pairing_code` → 201 `{ code, expires_at }` (o código só nessa resposta); `POST /waiting_room/displays/:id/revoke` → 200 e `waiting_room.display_revoked`; `POST /waiting_room/pair {code}` (sem sessão, sem `Authentication`; 10/min por IP; código desconhecido, expirado ou já usado → 422 `invalid_pairing_code`) → 201 `{ display_token }`, troca `token_digest`, zera o código e `revoked_at`, grava `paired_at`, `waiting_room.display_paired`; `GET /waiting_room/board` com `X-Display-Token` (sem cookie; ausente/desconhecido → 401 `invalid_token`; revogado → 401 `display_revoked`; interruptor → 403 `feature_disabled`); `WaitingRoom::Board.call(display:, now: Time.current) -> Hash` exatamente `{ unit: { name }, calls: [ { ticket, location, kind, called_at } ], wait_times: [ { color, median_minutes } ] | nil, updated_at }`, com `calls` do Valor fixado 6 e `wait_times` só numa UPA com `health_units.show_wait_times` e `display.show_wait_times`: por cor de `COLORS`, a mediana em minutos inteiros de `now - checked_in_at` dos `walk_in` `waiting` com encontro `classified` nessa cor; menos de 5 → `"lt5_patients"`. Sem trilha, sem `DomainEvent`.

- [ ] **Step 1: Write the failing test** `spec/requests/waiting_room/board_spec.rb` (sigilo da TV):

```ruby
RSpec.describe "Painel da TV", type: :request do
  let!(:city) { urgency_city!(waiting_room: true) }
  let(:upa) { upa! }
  let(:other_upa) { upa! }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }
  let!(:room) { call_location!(upa, name: "Consultório 1") }
  let!(:pair) { paired_display!(upa) }
  let(:token) { pair.last }

  def board(t = token) = get("/waiting_room/board", headers: { "X-Display-Token" => t })

  it "mostra senha e local da última chamada e nada da pessoa" do
    patient = urgency_patient!(1)
    e = arrived!(upa, patient: patient)
    classified!(e, color: "red", by: nurse, vital_signs: { spo2: 85 })
    sign_in_as(doctor)
    json_post "/attendance/units/#{upa.id}/call_next", call_location_id: room.id
    reset!                                             # sem cookie de sessão: a TV não tem login
    expect { board }.not_to change(DomainEvent, :count)
    expect(response).to have_http_status(:ok)
    expect(json_body.keys).to contain_exactly("unit", "calls", "wait_times", "updated_at")
    expect(json_body["calls"].first.keys).to contain_exactly("ticket", "location", "kind", "called_at")
    expect(json_body["calls"].first).to include("ticket" => "U-001", "location" => "Consultório 1", "kind" => "care")
    text = response.body
    [ patient.full_name, patient.cpf, patient.cns, e.id, e.attendance_id, patient.id, "SpO2", "red\"," ].compact.each { |s| expect(text).not_to include(s) }
    expect(json_body["wait_times"].map { |w| w["median_minutes"] }.uniq).to eq([ "lt5_patients" ])
  end

  it "o token de uma UPA não lê outra; revogado → 401 display_revoked; sem token → 401" do
    other_token = paired_display!(other_upa).last
    e = arrived!(upa, patient: urgency_patient!(2)).tap { |x| classified!(x, color: "green", by: nurse, pain_scale: 5) }
    called!(e, by: doctor, call_location: room)
    board(other_token)
    expect(json_body["calls"]).to eq([])
    WaitingRoom::Revoke.call(display: pair.first, by: urgency_admin!)
    board
    expect(response).to have_http_status(:unauthorized)
    expect(json_body["error"]).to eq("display_revoked")
    get "/waiting_room/board"
    expect(json_body["error"]).to eq("invalid_token")
  end

  it "com 5 ou mais numa cor sai a mediana; UBS nunca tem tempos" do
    5.times { |i| travel_to((10 + i).minutes.ago) { arrived!(upa, patient: urgency_patient!(10 + i)).tap { |x| classified!(x, color: "green", by: nurse, pain_scale: 5) } } }
    board
    expect(json_body["wait_times"].find { |w| w["color"] == "green" }["median_minutes"]).to eq(12)
    ubs_token = paired_display!(create_unit("UBS TV")).last
    board(ubs_token)
    expect(json_body["wait_times"]).to be_nil
  end
end
```

`spec/requests/waiting_room/displays_spec.rb`: só `municipal_admin` com step-up escreve; o código expira em 10 min (`travel 11.minutes` → `invalid_pairing_code`) e vale uma vez; o token não aparece em nenhuma resposta além do `pair`, nem em log (`capture_log`); `display_token` em claro nunca no banco (`WaitingRoomDisplay.pluck(:token_digest)` ≠ token); pareamento por cookie de sessão não é exigido (sem `sign_in_as`); 11ª tentativa no minuto → 429; interruptor desligado → 403 nas rotas de admin, do `pair` e do `board`.

- [ ] **Step 2:** `bin/test api -w mod32a spec/requests/waiting_room` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces (rate limit no padrão do `Hl7::BaseController`: `RateLimitStore` delegando a `Rails.cache`; `WaitingRoom::BaseController` sem `include Authentication` para `pair`/`board` — os de admin herdam um base com sessão). O helper `called!` aceita `call_location:` e passa `waiting_room: true`.
- [ ] **Step 4:** `bin/test api -w mod32a spec/requests/waiting_room spec/requests/urgency/calls_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add waiting room displays with pairing, revocable token and anonymous board`.

---

### Task 13: BPA pelo 22 — fonte `urgency`, só depois da identificação (transversal 22, §9)

**Files:**
- Create: `db/city_migrate/20261105900006_add_urgency_billing_source.rb`, `app/commands/billing/from_urgency.rb`, `app/jobs/billing/urgency_production_job.rb`, `app/services/urgency/billing_pending.rb`, `spec/commands/billing/from_urgency_spec.rb`
- Modify: `db/city_schema.rb`, `app/models/performed_procedure.rb` (`SOURCES` + `urgency`, `belongs_to :urgency_encounter, optional: true`), `app/commands/billing/record.rb` (`insert!` aceita `urgency_encounter:`), `app/services/billing/pending.rb`, `config/initializers/domain_events.rb` + bindings spec (`urgency.encounter_closed` e `urgency.patient_identified` → `[ Billing::UrgencyProductionJob ]`), `spec/commands/billing/from_psychosocial_spec.rb:86` (`AddPsychosocialBillingSource::SOURCES == PerformedProcedure::SOURCES - %w[urgency]`)

**Interfaces:**
- Produces (banco): `performed_procedures.urgency_encounter_id` (FK, índice); `ck_performed_procedures_source_ref` + `AND (source = 'urgency') = (urgency_encounter_id IS NOT NULL)`; único `idx_performed_procedures_urgency (urgency_encounter_id, sigtap_code) WHERE urgency_encounter_id IS NOT NULL`; `AddUrgencyBillingSource::SOURCES` espelha o modelo.
- Produces: `Billing::FromUrgency.call(encounter) -> { created: Integer, pending: :patient | nil }` — nada se o encontro não está `closed`; `pending: :patient` e nada se `encounter.patient.unidentified`; linha de classificação (`Urgency::Config.sigtap["classification_code"]`) quando há classificação: profissional e CBO da 1ª, `performed_on` = data local da 1ª; linha do atendimento (`care_code`) quando há consulta `finalized`: autor e CBO da consulta, `performed_on` = data local de `finalized_at`, `cid10` = o 1º CID-10 avaliado da consulta (por `created_at`, `id`) ou nulo; estabelecimento `Billing::Establishment.for_unit(upa)`; `quantity: 1`, `character: "urgent"`, `recorded_by: nil`, `attendance:`, `consultation:` (só no atendimento), `source: "urgency"`; idempotente pelos códigos já gravados do encontro (e o índice); `InsertFailed` sem a causa (padrão do `FromTfd`). `Billing::UrgencyProductionJob` (`IdempotentConsumer`, `handle(encounter_id:, **)`, `return unless Gate.enabled?(Current.city)`). `Urgency::BillingPending.call(period) -> Array<Hash>`: encontros fechados da UPA do estabelecimento do período, com classificação, paciente provisório e `closed_at` na competência → `{ procedure_id: nil, urgency_encounter_id:, reasons: ["patient_unidentified"], fix: "patient", late: false, performed_on:, sigtap_code: classification_code, label:, patient: { id:, display_name:, has_cpf: false, cns_masked: nil, address_from_household: false }, professional: { user_id:, name: } }`; `Billing::Pending.call` soma essas linhas ao fim.

- [ ] **Step 1: Write the failing test** `spec/commands/billing/from_urgency_spec.rb` (Review Focus 5):

```ruby
RSpec.describe Billing::FromUrgency do
  let!(:city) { urgency_city!(billing: true) }
  let(:upa) { upa!.tap { |u| u.update!(cnes: billing_cnes) } }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }

  def closed_with(encounter, kind)
    called!(encounter, by: doctor)
    finalized_urgency!(encounter, doctor: doctor)
    Urgency::RecordOutcome.call(encounter: encounter.reload, attrs: { "kind" => kind }, by: doctor)
  end

  it "identificado: duas linhas, uma por profissional; reprocessar não duplica" do
    e = arrived!(upa, patient: urgency_patient!(1)).tap { |x| classified!(x, color: "yellow", by: nurse, justification: "Conferido na classificação") }
    closed_with(e, "discharged")
    perform_enqueued_jobs
    rows = PerformedProcedure.where(urgency_encounter_id: e.id).order(:sigtap_code)
    expect(rows.map { |r| [ r.sigtap_code, r.professional_user_id, r.source, r.character ] })
      .to eq([ [ "0301060096", doctor.id, "urgency", "urgent" ], [ "0301060118", nurse.id, "urgency", "urgent" ] ])
    expect { described_class.call(e.reload) }.not_to change(PerformedProcedure, :count)
  end

  it "óbito de provisório: nenhuma linha, pendência fix patient; a identificação gera no real" do
    e = arrived!(upa, unidentified: "Homem, 60 anos").tap { |x| classified!(x, color: "red", by: nurse, vital_signs: { spo2: 80 }) }
    closed_with(e, "death")
    perform_enqueued_jobs
    expect(PerformedProcedure.where(urgency_encounter_id: e.id)).to be_empty
    period, = Billing::Periods.assign!(Billing::Establishment.for_unit(upa), Time.zone.today.strftime("%Y%m"))
    expect(Billing::Pending.call(period).find { |r| r[:urgency_encounter_id] == e.id }).to include(fix: "patient", procedure_id: nil)
    real = urgency_patient!(2)
    travel 1.day do
      Urgency::Identify.call(encounter: e.reload, patient_id: real.id, by: reception!)
      perform_enqueued_jobs
    end
    expect(PerformedProcedure.where(urgency_encounter_id: e.id).pluck(:patient_id).uniq).to eq([ real.id ])
    expect(Billing::Pending.call(period).none? { |r| r[:urgency_encounter_id] == e.id }).to be(true)
  end

  it "evasão antes da classificação não gera linha nem pendência" do
    e = arrived!(upa, unidentified: "Mulher, 25 anos")
    Urgency::RecordOutcome.call(encounter: e, attrs: { "kind" => "left_without_care" }, by: reception!)
    perform_enqueued_jobs
    expect(PerformedProcedure.where(urgency_encounter_id: e.id)).to be_empty
  end
end
```

Mais: o banco recusa `source: "urgency"` sem `urgency_encounter_id` e encontro em outra fonte (`/ck_performed_procedures_source_ref/`); `AddUrgencyBillingSource::SOURCES == PerformedProcedure::SOURCES`; o job com `procedure_billing` desligado não grava. (O helper `billing_cnes`: conferir o nome real em `spec/support/billing_helpers.rb`.)

- [ ] **Step 2:** `bin/test api -w mod32a spec/commands/billing/from_urgency_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces.
- [ ] **Step 4:** o spec + `spec/commands/billing spec/services/billing spec/requests/billing spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: bill urgency classification and physician care through procedure billing`.

---

### Task 14: Linha do tempo — fonte `urgency`, documentos do `walk_in` e fora do PEC (transversal, §10)

**Files:**
- Create: `app/services/timeline/sources/urgency_source.rb`, `spec/services/timeline/sources/urgency_source_spec.rb`
- Modify: `app/services/timeline.rb` (`KINDS` + `urgency` no fim), `app/services/timeline/query.rb` (`SOURCES`), `app/services/timeline/sources/{consultation_source,clinical_document_source,labels}.rb`, `app/services/timeline/citizen_documents.rb`, `app/controllers/pec_timelines_controller.rb` (`KINDS = Timeline::KINDS - %w[transport urgency]`), `spec/services/timeline/query_spec.rb` (`KINDS.last(4) == %w[dispensation vaccination transport urgency]`), `spec/requests/pec_timeline_spec.rb` (`Timeline::KINDS - %w[transport urgency]`)

**Interfaces:**
- Produces: `Timeline::Sources::UrgencySource.entries(person:, before:, limit:, from:, to:)` sobre `UrgencyEncounter.joins(:attendance).where(attendances: { patient_id: person.patient_id })` (vazio sem paciente), coluna `attendances.checked_in_at`, `id_column` `urgency_encounters.id`, `kind: "urgency"`; profissional: `title` "Atendimento de urgência — <UPA>", `summary` = cor vigente (`Labels::URGENCY_COLORS`: Vermelho, Laranja, Amarelo, Verde, Azul), o 1º CID-10 avaliado "CÓDIGO nome", o desfecho (`Labels::URGENCY_OUTCOMES`) e "Retorno à UBS em até N dias: <motivo>", juntos por " · " sem os nulos; `status` = estado do encontro; `professional` = autor da consulta (nome, CBO) ou nulo; `detail_ref: { type: "urgency_encounter", id: }`; cidadão: `title` igual, `summary` = `Labels::CITIZEN_URGENCY_OUTCOMES[kind]` (Alta; Alta com retorno à UBS; Transferido(a) para outro serviço; Saiu antes do atendimento; Saiu antes da alta; Óbito) + " · Procure sua UBS em até N dias" quando houver, nunca CID, cor, critério, queixa nem motivo; `ConsultationSource` exclui `kind: "urgency"` (a entrada da urgência já a representa); `ClinicalDocumentSource` e `CitizenDocuments.find` incluem `citizen_id IS NULL AND patient_id = person.patient_id` (sempre com `psychosocial_action_id: nil`).

- [ ] **Step 1: Write the failing test:**

```ruby
RSpec.describe Timeline::Sources::UrgencySource do
  let!(:city) { urgency_city! }
  let(:upa) { upa!(name: "UPA Boqueirão") }
  let(:nurse) { upa_nurse!(upa) }
  let(:doctor) { upa_doctor!(upa) }
  let(:patient) { urgency_patient!(1) }

  it "profissional vê cor, CID e orientação; cidadão só unidade, data, desfecho simples e o prazo" do
    e = arrived!(upa, patient: patient).tap { |x| classified!(x, color: "orange", by: nurse, pain_scale: 9) }
    called!(e, by: doctor)
    finalized_urgency!(e, doctor: doctor, cid10: "I10")
    Urgency::RecordOutcome.call(encounter: e.reload, attrs: { "kind" => "discharged_to_primary_care",
                                                             "return_guidance" => { "days" => 5, "reason" => "Ajustar anti-hipertensivo" } }, by: doctor)
    entry = described_class.entries(person: Timeline::Person.for_patient(patient), before: nil, limit: 10, from: nil, to: nil).first
    expect(entry.title).to eq("Atendimento de urgência — UPA Boqueirão")
    expect(entry.summary).to include("Laranja", "I10", "Retorno à UBS em até 5 dias: Ajustar anti-hipertensivo")
    expect(entry.citizen[:summary]).to eq("Alta com retorno à UBS · Procure sua UBS em até 5 dias")
    expect(entry.citizen.to_s).not_to include("I10", "Laranja", "orange", "anti-hipertensivo")
  end

  it "provisório não aparece para o cidadão; depois da fusão, aparece no histórico do real e some a consulta duplicada" do
    e = arrived!(upa, unidentified: "Homem, 45 anos").tap { |x| classified!(x, color: "green", by: nurse, pain_scale: 5) }
    expect(Timeline::Query.call(person: Timeline::Person.for_cpf(patient.cpf), kinds: Timeline::Query.citizen_kinds(verified: true, documents: true)).entries).to be_empty
    Urgency::Identify.call(encounter: e, patient_id: patient.id, by: reception!)
    kinds = Timeline::Query.call(person: Timeline::Person.for_patient(patient)).entries.map(&:kind)
    expect(kinds).to include("urgency")
    expect(kinds).not_to include("consultation")
  end
end
```

`spec/requests/citizen_timeline_spec.rb` (ou o arquivo real da rota do wpda, conferir): o "Meu histórico" do cidadão do CPF traz o item `urgency` sem `CID`, sem cor e sem a queixa da classificação (`response.body` não contém a queixa nem o critério); `spec/requests/pec_timeline_spec.rb`: nunca `urgency`.

- [ ] **Step 2:** `bin/test api -w mod32a spec/services/timeline spec/requests/pec_timeline_spec.rb spec/requests/citizen_timeline_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** conforme os Interfaces (`finalized_urgency!` aceita `cid10:`).
- [ ] **Step 4:** `bin/test api -w mod32a spec/services/timeline spec/requests/timelines_spec.rb spec/requests/pec_timeline_spec.rb spec/requests/citizen_timeline_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: show urgency encounters on the patient timeline without clinical detail for citizens`.

---

### Task 15: Painel do gestor e o detalhe do encontro (§11)

**Files:**
- Create: `app/services/urgency/dashboard.rb`, `app/controllers/urgency/dashboard_controller.rb`, `spec/requests/urgency/dashboard_spec.rb`, `spec/requests/urgency/encounter_detail_spec.rb`
- Modify: `app/controllers/urgency/encounters_controller.rb` (`show`), `config/routes.rb`

**Interfaces:**
- Produces: `GET /urgency/dashboard?month=YYYY-MM&health_unit_id=` (`municipal_admin` e `analyst`; o resto 403 `missing_role`; `month` padrão = mês corrente; formato inválido ou futuro → 422 `invalid_request` `month`; `health_unit_id` fora das UPAs ativas → 422; sem trilha) → `Urgency::Dashboard.call(units:, month:)` = `{ month, units: [ { id, name } ], door_to_classification: [ { color, count, median_minutes, p90_minutes } ], classification_to_physician: [ … ], over_max_wait: [ { color, count, over } ], by_color: { red:, orange:, yellow:, green:, blue:, unclassified: }, by_outcome: { <OUTCOMES> }, evasion: { before_classification:, after_classification: }, stay: { count, median_minutes, p90_minutes } }` sobre os encontros com chegada (data local) no mês: porta→classificação pela cor da 1ª classificação; classificação→médico (`called_at - first_classified_at`) e estouro pela classificação vigente na chamada (a última com `created_at <= called_at`, com o `max_wait_minutes` dela); `by_color` pela cor vigente; permanência = `closed_at - checked_in_at` dos fechados; `percentile_cont(0.5|0.9)` em minutos inteiros; supressão DEPOIS de agregar: toda contagem de 1 a 4 vira `"lt5"`, mediana/p90 com `count` de 1 a 4 viram `"lt5"`, com 0 viram `nil`. `GET /urgency/encounters/:id` (`clinical_link` na UPA; senão 403) → `{ encounter: { id, state, attendance_id, ticket, arrived_at, patient: <patient-ref detail>, color, overdue, classifications: [ { id, position, color, suggested_color, criterion, complaint, vital_signs, pain_scale, justification, max_wait_minutes, classified_at, by: { name, cbo_code } } ], consultation_id, outcome: { kind, destination, vacancy_request, return_guidance, recorded_at } | nil } }` + `Urgency::Trail.viewed!(resource: "encounter")`.

- [ ] **Step 1: Write the failing test** (trechos): com 6 verdes (porta→classificação 10..15 min) e 2 vermelhos no mês, `door_to_classification` do verde traz `count: 6` e a mediana certa; o vermelho traz `count: "lt5"`, `median_minutes: "lt5"`; `by_color[:orange] == 0`; um verde chamado aos 130 min conta em `over_max_wait` (`over` suprimido se 1..4); evasões separadas antes/depois da classificação; `health_professional` → 403; `response.body` nunca contém nome nem id de paciente. Detalhe: a enfermeira lê a queixa e a descrição do provisório e isso publica `clinical_record.viewed` com `access: "urgency"`, `resource: "encounter"`; a recepção recebe 403.
- [ ] **Step 2:** `bin/test api -w mod32a spec/requests/urgency/dashboard_spec.rb spec/requests/urgency/encounter_detail_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (SQL com `LATERAL` para a classificação vigente na chamada; fuso pelo padrão do `Psychosocial::Dashboard`).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add the urgency manager dashboard and encounter detail`.

---

### Task 16: Exclusão LGPD (ADR 0026)

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb` (`examined_patients?` + `urgency_recorded?(ids)`), `spec/commands/citizens/request_erasure_spec.rb`

**Interfaces:**
- Produces: `Citizens::RequestErasure.urgency_recorded?(ids) -> Boolean` = existe `walk_in` desses pacientes com encontro que tem classificação ou consulta (retido); a evasão antes da classificação não retém, e o `Citizens::Erase` segue apagando os dados demográficos do paciente como hoje (a linha do `walk_in` fica, só com ids).

- [ ] **Step 1: Write the failing test:** pedido de exclusão do CPF de quem foi classificado na UPA → `retained`; de quem só chegou e saiu antes da classificação → `pending` e, confirmado, o paciente fica sem nome/nascimento/sexo e o atendimento continua, sem CPF em lugar nenhum do encontro; o provisório nunca é atingido pela exclusão por CPF.
- [ ] **Step 2:** `bin/test api -w mod32a spec/commands/citizens/request_erasure_spec.rb spec/commands/citizens/erase_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** com o comentário do padrão ("ADR 0050: o encontro com classificação ou consulta é registro de saúde retido; a evasão antes da classificação segue a triagem sem atendimento").
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: retain classified urgency encounters on citizen erasure`.

---

### Task 17: Semente de dev realista

**Files:**
- Create: `lib/urgency_crew.rb`, `spec/lib/urgency_crew_spec.rb`
- Modify: `db/seeds.rb` (depois do `PsychosocialCrew`, com o comentário do padrão)

**Interfaces:**
- Produces: `UrgencyCrew.seed_current_city(slug:, ddd:, password:) -> Hash` e `UrgencyCrew.summary_lines(result, password:) -> Array<String>`; só em Curitiba (nas demais, o rascunho do protocolo, como o `ScreeningCrew`). Tudo pelos comandos do app, idempotente (acha pelo que já gravou):
  - `urgency_care` e `waiting_room_display` ligados pelo mantenedor de dev; "UPA 24h Centro" (do `ProfessionalCrew`) com `ticket_prefix: "U"`, `show_wait_times: true`;
  - equipe pelos comandos do 10, TOTP fixo, perfil e vínculo com CBO real e conselho coerente: `upa@curitiba.dev` (`citizen_verifier`, recepção), `upaenf@curitiba.dev` (`health_professional`, 223505), `upamed@curitiba.dev` (`health_professional`, 225125); a médica (225124) e a técnica (322205) do `ProfessionalCrew` seguem lá;
  - protocolo `classificacao-urgencia` v1 pelo `TriageCatalogCrew.run_cycle!` (assinado e ativo), com os 5 níveis padrão e regras de exemplo (rótulo no comentário: "exemplo de dev, não é o Manchester; revisão clínica antes de cidade real"): SpO2 < 90 ou PAS ≥ 200 ou FR ≥ 30 → vermelho; dor ≥ 8, PAS ≥ 180, FC ≥ 130, glicemia < 60 → laranja; temperatura ≥ 39, dor ≥ 5 → amarelo; temperatura ≥ 37,8, dor ≥ 2 → verde;
  - locais "Classificação de risco", "Consultório 1", "Consultório 2", "Sala de medicação"; TV "TV da recepção" (com tempos), com um código de pareamento novo a cada semente (no resumo: "TV: código NNNNNN, vale 10 min");
  - cenário (CPF e CNS fictícios válidos por `cpf_for`/`Professionals::Cns.generate`, pessoas pelo `Urgency::Arrive` com `new_patient`): "Roberto Carlos Nogueira" (66, dor torácica, PA 182/104, dor 8 → laranja), "Ana Beatriz Souza" (28, febre 38,4, dor 3 → verde), "Francisco das Chagas Lima" (81, febre 39,2 → amarelo), "Juliana Martins Prado" (34, azul reclassificada para verde com justificativa), um não identificado aguardando classificação ("Homem, aparenta 40 anos, encontrado na rodoviária, confuso"), e um atendimento de ontem de "Antônio" (o paciente do `CareLinesCrew` com equipe da ESF) fechado com `discharged_to_primary_care` em 7 dias (consulta `urgency` finalizada pela `upamed`), que aparece na contrarreferência da UBS e no "Meu histórico";
  - chegadas com `now:` escalonado (40, 25, 15, 5 e 2 min atrás) para a fila mostrar espera e um estouro (o laranja de 25 min).
- Resumo: só e-mail, papel, otpauth e o código da TV; nunca dado de paciente.

- [ ] **Step 1: Write the failing test** `spec/lib/urgency_crew_spec.rb`: rodar duas vezes não duplica encontros, classificações nem locais; o protocolo ativo é o da semente; a fila da UPA tem 4 classificados e 1 aguardando; o laranja está `overdue`; o resumo não contém nenhum nome de paciente.
- [ ] **Step 2:** `bin/test api -w mod32a spec/lib/urgency_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** no padrão do `PsychosocialCrew`/`ScreeningCrew` (`Arrive`/`Classify` aceitam `now:`).
- [ ] **Step 4:** o spec + `spec/lib/psychosocial_crew_spec.rb spec/lib/screening_crew_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: seed a realistic dev upa with protocol, staff, waiting room and patients`.

---

### Task 18: Specs de sigilo, guarda de eventos e suíte completa

**Files:**
- Create: `spec/security/urgency_secrecy_spec.rb`
- Modify: `spec/initializers/domain_events_bindings_spec.rb` (conferência final)

- [ ] **Step 1: Write the test** `spec/security/urgency_secrecy_spec.rb` (um cenário completo: chegada de paciente com CPF e de provisório, classificação com queixa, chamada com local, identificação com fusão, consulta com CID, desfecho com orientação):
  - todo `DomainEvent` com nome `urgency.*`, `attendance.recalled` e `waiting_room.*` tem só chaves de id (`/_id\z/`) — e nenhum payload contém o CPF, o CNS, o nome, a queixa, o critério, o CID ou a descrição;
  - `capture_log` de todas as requisições do cenário não contém CPF, CNS, nome, queixa, critério, CID, descrição nem o token da TV;
  - toda recusa 4xx do cenário (forçadas: CPF malformado, justificativa curta, descrição curta, destino ausente) traz só `error`/`path`/`problems` sem o valor enviado;
  - nenhuma rota de `/urgency` e `/waiting_room` aceita CPF, CNS ou nome na query (`Rails.application.routes` + `query_only!`: `GET /urgency/queue?cpf=…` → 422 `path: "cpf"`);
  - o `board` do cenário: o mesmo conjunto de chaves da Task 12 e nenhum dos valores acima, nem `red|orange|yellow|green|blue` fora de `wait_times`;
  - o histórico do cidadão (rota do wpda) e a linha do tempo do PEC externo não trazem CID, cor, queixa, critério nem o motivo da orientação; o provisório nunca aparece ao cidadão;
  - `Platform::Features::CATALOG` e `R18_PLATFORM_EVENT_NAMES` sem `Platform.audit` novo (grep de `Platform.audit(` multilinha nos arquivos novos = 0).
- [ ] **Step 2:** `bin/test api -w mod32a spec/security/urgency_secrecy_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture` → PASS (se falhar, a correção vai à task dona e volta aqui).
- [ ] **Step 3: Commit** `test: pin urgency secrecy for events, logs, errors, tv board and citizen timeline`.
- [ ] **Step 4: Suíte completa** (transversal: 13, 19a, 22, 18, linha do tempo, `spec/support`): pedir à sessão "Testes" por `SendMessage`, no formato `api · mod32a/feat/mod-32a-urgency · --full · módulo 32a (attendances, patients, consultations, performed_procedures, protocolo do 18, linha do tempo)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador ("Próximos módulos") com o resumo e o hash.
- [ ] **Step 5:** merge, push e o card só com autorização do usuário; F-32.x seguem In Progress até o último app de cada card.

---

## Self-review

1. **Cobertura do contrato:** §1 (Tasks 1–5, 10, 13, 14); §2 (Tasks 6, 15 — `<patient-ref>`, erros, POST com corpo); §3 (Task 5; Suggest na 5 e o uso na 7); §4 (Tasks 6 e 9); §5 (Task 7); §6 (Task 8); §7 (Tasks 10 e 11); §8 (Task 12; os parâmetros do 13 na 8); §9 (Task 13); §10 (Tasks 10 e 14; vigilância na 10); §11 (Task 15); §12 (eventos em cada task dona; guarda na 18); §13 (Tasks 3, 4, 16, 18). Spec §6 "Testes": encontro só na UPA (2, 6), não identificado e identificação única (6, 9), cor sugerida/justificativa/reclassificação (7), ordem e "Chamar próximo" (8), desfecho obrigatório (11), contrarreferência (11), senha única por dia (2), chamada com local e de novo (8), TV (12), BPA (13), linha do tempo do cidadão (14), vigilância (10), painel com "menos de 5" (15).
2. **Nomes:** `Urgency::Arrive/Classify/CallForClassification/Identify/RecordOutcome`, `Urgency::Queue.lock_next_classified`, `WaitingRoom::CallLocations.resolve`, `Attendances::CallRecord.record!`, `Urgency::Merge::{TABLES,KEPT}`, `Billing::FromUrgency`, `Billing::UrgencyProductionJob`, `Urgency::BillingPending` — usados com a mesma assinatura nas tasks seguintes.
3. **Ordem das migrações** segue a ordem das tasks (900001..900006); a 900002 depende da 900001 (`attendances.patient_id`), a 900003 da 900002, a 900005 da 900003 (o encontro existe antes da consulta urgency), a 900006 da 900003.

## Divergências propostas ao contrato

1. **`patients.unidentified` (bool).** O contrato §1 só muda `no_cpf_reason` (`"unidentified"`) e acrescenta `unidentified_description`. `no_cpf_reason` é cifrado não determinístico: não dá para filtrar nem para escrever CHECK sobre ele. Proposta: coluna `unidentified` (não cifrada) mais o valor `"unidentified"` (mantido). O `<patient-ref>.unidentified` vem dela.
2. **`person_lookup` sem achar.** O padrão do 29/30/31 responde 404 `person_not_found`; o contrato §4 pede `{patient: null, candidates: []}`. O plano segue o contrato (200). `candidates` sai sempre `[]`, porque a busca só recebe CPF ou CNS e não há nome nem nascimento para comparar. Proposta: tirar `candidates` ou definir de onde vêm.
3. **Caminhos sob `/attendance`.** As rotas do 13 moram no `scope "/attendance"`. Por isso `recall` fica em `POST /attendance/attendances/:id/recall`, e os locais em `GET/POST/PATCH /attendance/call_locations`. `call_location_id` entra no corpo de `POST /attendance/attendances/:id/call` e de `POST /attendance/units/:id/call_next`.
4. **Formato do protocolo.** O §3 não fixa os nomes. O plano supõe:
   - nome reservado `classificacao-urgencia`;
   - `levels[]` com `{color, max_wait_minutes}`, as 5 cores em ordem, 0 a 1440 min, sem diminuir de uma cor para a seguinte;
   - `rules[]` com `{id, when, color, criterion}`;
   - variáveis `vitals.*`, com a dor em `vitals.pain_score`, e `profile.age`/`profile.sex`.
   A queixa (texto livre) nunca entra na condição. Se a tag `protocols-v1.7.0` divergir, a Task 0 para e avisa o coordenador.
5. **Fila.** Proposta: um terceiro grupo `in_care[]` com `color`, `called_at` e `called_by_name` (a tela do médico precisa dele), e `attendance_id` em todo item, para chamar e fechar pelo 13.
6. **Detalhe do encontro.** O §2 cita "o detalhe", mas não define a rota. Proposta: `GET /urgency/encounters/:id`, no formato da Task 15, com trilha `access: "urgency"`, `resource: "encounter"`. Só lê quem tem vínculo clínico com a UPA.
7. **Aviso interno da contrarreferência.** O §7 não define o canal. Proposta:
   - `GET /urgency/counter_referrals?health_unit_id=` para os profissionais com vínculo na UBS, com os itens dos últimos 60 dias, sem o motivo e sem trilha;
   - o motivo aparece só na linha do tempo profissional, que grava trilha;
   - nenhum evento novo.
8. **`referral_note` do 13.** É texto claro. No `discharged_to_primary_care`, o plano grava `"Retorno à UBS em até N dias"`; o motivo fica só cifrado em `urgency_outcomes`. `transferred` vira `referred` com `referral_regulated = true`, sem nota. (O §7 pede "a orientação" na nota.)
9. **Evasão depois da chamada.** O 13 não fecha `in_care` com `left`. O plano relaxa o `rota_attendance_guard` só quando o encontro tem desfecho `left_*`.
10. **Check-in do 13 numa UPA.** Com `urgency_care` utilizável, o check-in por código ou por exceção numa UPA responde 422 `urgency_reception_required`. Assim, todo atendimento de UPA tem encontro, e o fechamento sempre exige o desfecho.
11. **Pendência do BPA sem linha.** O §9 pede "pending com `fix: "patient"`", mas a pendência do 22 nasce de uma linha. Proposta:
    - `Billing::Pending` passa a trazer itens com `procedure_id: null`, `urgency_encounter_id` e `reasons: ["patient_unidentified"]`;
    - o dashboard do faturamento precisa aceitar `procedure_id` nulo nesses itens.
12. **Tempos na TV.** Só numa UPA, e só com o `show_wait_times` da unidade e o do dispositivo ligados. Numa UBS sai sempre `null`, porque a UBS não tem as 5 cores. Proposta: manter um flag só, o do dispositivo, ou deixar claro que valem os dois.
13. **`attendances.ticket_on` (date).** Coluna nova, para o índice único de senha por unidade e dia local.
14. **Classificação pelo técnico (3222\*).** A Resolução COFEN 661/2021 torna a classificação de risco privativa do enfermeiro. Além disso, o SIGTAP de dev do 0301060118 não lista o 322205, então a linha BPA de quem é técnico sai bloqueada por CBO. O plano segue o contrato (2235\* e 3222\*). Proposta: só 2235\*, a confirmar no gate do SIGTAP.
15. **Formato do painel.** O §11 lista as medidas, mas não o formato. Proposta: o formato da Task 15. "lt5" também vale para mediana e p90 de grupos com 1 a 4 casos; sem nenhum caso, o valor é `null`.
16. **Fusão e a lista de problemas e de medicamentos.** O R4 não os cita, e eles ficam com o provisório. A lista de problemas e de medicamentos do paciente real não recebe o que o médico registrou para o provisório; o CID continua visível pela linha do tempo. Proposta: um card de pendência no board #2 para decidir se a fusão também leva esses dados.
17. **Relógio do estouro.** O estouro conta desde a 1ª classificação e usa o tempo máximo gravado na classificação vigente. O tempo máximo é uma foto: trocar de protocolo não muda o estouro de quem já foi classificado.
18. **TV.**
    - Desativar o dispositivo também o revoga.
    - Um dispositivo revogado pode ser pareado de novo, com token novo.
    - Token desconhecido → 401 `invalid_token`; token revogado → 401 `display_revoked`.
    - Código errado ou vencido → 422 `invalid_pairing_code`.
    - Limites: 10 pareamentos por minuto por IP e 30 leituras do painel por minuto por token.
19. **Simulador do editor.** O contrato não prevê prévia da definição em edição, como o `simulate_screening` do 18. O plano não a implementa e propõe `POST /authoring/protocols/simulate_urgency` para o dashboard, se ele precisar.
20. **`POST /urgency/suggest`.** Exige `encounter_id`, porque a idade e o sexo do paciente entram no perfil da regra. A queixa é aceita e ignorada.
21. **Orientação ao cidadão.** O cidadão vê "Procure sua UBS em até N dias", sem o motivo: texto clínico livre nunca vai ao cidadão, pela regra da `Timeline::Entry`.
22. **Identificação exigida no desfecho.** O §7 não diz nada. O plano exige paciente identificado em `discharged`, `discharged_to_primary_care` e `transferred` (422 `patient_unidentified`). `death` e `left_*` dispensam, como na decisão 7 da spec e no R4.
23. **`kind` da consulta.** Sem `kind`, um atendimento com encontro abre consulta `urgency`, e um sem encontro abre `general`. `kind` errado → 422 `invalid_request` `path: "kind"`. A finalização da consulta `urgency` não aceita `outcome`: o desfecho sai sempre pela rota do 32.
24. **Chamada do médico.** Chamar um `walk_in` exige o encontro `classified` (422 `not_classified`) e vínculo 225\* (403 `cbo_not_allowed`). Assim, ninguém além do médico fica como "quem chamou", e a consulta pode ser aberta.
