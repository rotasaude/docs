# Módulo 30 — Transporte sanitário e TFD (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulos 28 e 29 em `origin/main` do api** (conferido ao escrever este plano em `5a32ced0`): última migração de cidade `20261102900001_add_vaccination_room.rb`; `db/city_schema.rb` com `define(version: 2026_11_02_900001)` e `enable_extension "btree_gist"`; a última migração que reescreveu `ck_memberships_role` é `AddPharmacyStock` (27a), com `AddPharmacyStock::ROLES` (19 papéis) — referenciada em `spec/models/create_analytics_migration_spec.rb:32` e `spec/models/create_campaigns_migration_spec.rb` (`restore_exam_roles`); `Membership::ROLES`/`PRIVILEGED_ROLES` sem papel de transporte; `HealthUnit::KINDS = %w[ubs upa hospital other ceo]` (`ck_health_units_kind`); `PerformedProcedure::SOURCES = %w[attendance consultation lab_report manual]` (`ck_performed_procedures_domains`, `ck_performed_procedures_source_ref`); `Billing::Record.insert!(patient:, establishment:, professional:, cbo_code:, source:, performed_on:, sigtap_code:, quantity:, cid10:, character:, recorded_by:, attendance: nil, consultation: nil, lab_report_item: nil)`; `Billing::Establishment.for_unit(unit)` com `aps?` = `type == "health_unit" && kind == "ubs"` (a unidade `health_management` já não é marcada por `establishment_is_aps`); `Billing::LabProductionJob` (IdempotentConsumer, `Gate.enabled?`) ligado a `lab_report.released`; `CareRegulation::ScheduleExternal.call(request:, at:, place:, notes:, version:, by:, now:)` chamado em `CareRegulation::DecisionsController#schedule` (`SCHEDULE_KEYS`); `care_providers` sem município; `Exams::Files.store(bytes:, content_type:, user:, provider:, purpose:)` com `ck_exam_files_purpose` = `eqa_annual_report`; `Appointments::Remind` (regras de SMS: `Campaigns::SmsSetting.enabled?`, `SmsGateway.configured?`, `Campaigns::SmsBatchJob::WINDOW_HOURS`, `CitizenContactPreference#sms_opt_in`/`appointment_reminders_muted`); `CitizenApi::NoticesController` com duas fontes (campanha, `appointment_notices`); `Timeline::KINDS` terminando em `dispensation vaccination` e `spec/services/timeline/query_spec.rb:75` conferindo `KINDS.last(2)`; `Platform::Features::CATALOG` terminando em `rnds_mock`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..47)`; `Admin::SmallCount.small?` (1..4); `Patients::Pairs.of(patient)`; `Patient#age(on:)`, `#display_name`, `#cns_masked`; `CitizenIdentity::Phone.mask`; `ActiveRecord::ExclusionViolation` tratado em `Appointments::Book`.

**Goal:** O lado api do módulo 30 (F-30.1 a F-30.8, ADR 0048): frota (veículos, motoristas com CNH, destinos), viagens com paradas e passageiros sem dado clínico e sem sobreposição de veículo ou motorista no banco, séries recorrentes com job, tela do motorista (presença e fechamento), processo de TFD (laudo, comissão, trechos, ajuda de custo, pagamento registrado, prestação de contas), produção do 08.03.01 no BPA-I pelo módulo 22, "Minhas viagens" e avisos/SMS ao cidadão, fonte `transport` da linha do tempo, painel e exclusão LGPD.

**Architecture:** Tudo no banco da cidade numa migração só (`20261103900001`), atrás de `sanitary_transport` e `tfd` (este requer o primeiro e `clinical_record`); sem migração de plataforma (o catálogo de interruptores é código). Duas exclusion constraints GiST (`tsrange(departure_at, return_at)`) garantem veículo e motorista sem sobreposição; o Ruby confere antes para devolver todas as recusas em `problems[]`, e a constraint é a rede contra a corrida. O TFD é um processo com estado atual em `tfd_cases` e andamento imutável em `tfd_case_events`; trechos geram as linhas de ajuda de custo (`tfd_allowances`, uma por pessoa e código), e cada linha vira um `performed_procedure` `source: "tfd"` pelo caminho do laboratório (evento → `Billing::TfdProductionJob` → `Billing::Record.insert!`). Avisos ao cidadão nascem de eventos (`Transport::NoticeJob`) e do lembrete da véspera (`Transport::RemindJob`), com as regras de SMS do 17.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (cidade; `btree_gist` já habilitado), Solid Queue, Prawn (lista de passageiros), RSpec via `bin/test`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md` e `docs/adr/0048.md` (com as "Decisões do coordenador"). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-03-module-30-sanitary-transport-contracts.md`; toda diferença está em "Divergências propostas ao contrato", no fim. Pesquisa: `docs/pesquisa/2026-11-03-transporte-e-tfd-30.md`.

## Desvios e precisões

1. **Fuso.** "America/Sao_Paulo" do contrato = `Time.zone` da cidade (o `CityConnection.with` já fixa o fuso dela, como no `Appointments::RemindJob`). `transport_trips.date` = `departure_at.in_time_zone.to_date`, conferido no Ruby e por CHECK (`ck_transport_trips_date` com `AT TIME ZONE 'UTC' AT TIME ZONE <fuso da cidade>` escrito na migração a partir de `Current.city.time_zone`). Horas no corpo: ISO 8601 com fuso explícito (`CareRegulation::ScheduleExternal::ZONED`); horas da série: `"HH:MM"`.
2. **Sobreposição.** `excl_transport_trips_vehicle` (`vehicle_id WITH =, tsrange(departure_at, return_at) WITH &&`, `WHERE status <> 'cancelled' AND vehicle_id IS NOT NULL`) e `excl_transport_trips_driver` (o mesmo com `transport_driver_id`). Intervalo meio-aberto `[)`: retorno às 12:00 e saída às 12:00 não se cruzam. `Transport::Overlap.problems(trip)` faz a mesma pergunta em SQL antes do INSERT/UPDATE (todas as recusas de uma vez); `ActiveRecord::ExclusionViolation` vira `vehicle_overlap` ou `driver_overlap` pelo nome da constraint na mensagem (422, nunca 500).
3. **Ocupação.** `seats_used` = passageiros ativos (não retirados) com `takes_seat` + acompanhantes ativos; `wheelchair_used` = passageiros ativos com `wheelchair`. Viagem sem veículo não tem limite (planned sem escala). Toda escrita de passageiro trava a viagem (`FOR UPDATE`) antes de contar.
4. **Origem do passageiro** (contrato §1): `regulation` (pedido `scheduled` do paciente), `appointment` (horário do 17 `scheduled`/`confirmed` de um par validado do paciente), `tfd` (processo `authorized`/`scheduled`/`travelling` do paciente), `series` (série do paciente), `exception` (motivo de lista fechada; `exception_note` 10..500 obrigatório em `other`, opcional nos outros). Referência de outro paciente → `origin_mismatch`; sem origem e sem motivo → `origin_required`.
5. **Acompanhante.** Fora do TFD: permitido sem motivo se `patient.age(on: trip.date) <= companion_rule.max_age_child` ou `>= companion_rule.min_age_elderly` (padrão 15 e 60); fora disso exige `companion.reason` (10..500), senão `companion_not_allowed`. No TFD (`origin: "tfd"`): só com `report.companion_indicated`; o motivo não supre a indicação (`companion_not_allowed`).
6. **Destino do passageiro.** Sempre `destination_id` (município com km) + `provider_name`/`address` opcionais (cifrados). Do agendamento externo do 26a só vem o NOME do prestador (`care_providers.name`, quando `care_provider_id` existe); o texto livre `schedule_place` **nunca** é copiado (pode trazer especialidade, "Clínica de Olhos").
7. **Viagem `planned` sem escala** aceita `vehicle_id`/`driver_id` nulos; `confirm` exige os dois (`vehicle_required`, `driver_required`), motorista ativo com CNH não vencida na data (`driver_license_expired`), categoria `D`/`E` se `seats > 8` (`driver_license_category`), veículo ativo (`vehicle_inactive`) e lotação não excedida (`seats_full`/`wheelchair_full`). Em `confirmed`, `PATCH` pode trocar veículo/motorista (mesmas regras), nunca anular.
8. **Avisos** só para viagem `confirmed`/`in_progress`: `confirm` avisa todos (`added`); passageiro entrando em viagem confirmada → `added`; `departure_changed` quando a saída muda 30 min ou mais; `boarding_changed` quando o rótulo da parada do passageiro muda; `cancelled`; `removed`; `reminder` na véspera (17h–20h). Um aviso por (passageiro, cidadão, tipo, chave do evento) — índice único; `event_key` = `departure_at` ISO no `departure_changed`, `label` em hex SHA-256 curto no `boarding_changed`, `"once"` nos outros.
9. **Presença imutável depois de `done`** (trigger): `attendance`/`companion_attendance` só mudam com a viagem `in_progress`, ou sob `SET LOCAL rota.transport_correction = 'on'` (só o `Transport::CorrectAttendance`, que grava `transport_attendance_corrections`, imutável). Correção de `boarded` → `no_show` com trecho de frota do TFD já `done` → `tfd_leg_done` (a linha do BPA já existe).
10. **TFD — laudo.** `tfd_cases.report` é um JSON cifrado (`{ sigtap_code, cid10, justification, companion_indicated, companion_reason, transport_mode }`); a lista e os eventos nunca o leem. Autor: Membership `health_professional` + CBO `225*` (da consulta, quando `consultation_id`; senão do `ProfessionalLink` ativo mais antigo do usuário com CBO `225*`) gravado em `requester_cbo`. Exige `consultation_id` (consulta do autor, não descartada, do mesmo paciente) **ou** `regulation_request_id` (do mesmo paciente) — ao menos um (`invalid_request` com `path: "consultation_id"`).
11. **TFD — recusas no `submit` e de novo no `authorize`** (todas em `problems[]`): `distance_below_minimum` (`destination.distance_km < tfd_min_distance_km`), `metropolitan_destination`, `primary_care_procedure` (`Tfd::PrimaryCare.covers?(code)` sobre `config/tfd/primary_care_groups.yml`), `cid_required`, `justification_required` (20..2000), `invalid_sigtap_code` (fora da SIGTAP ativa). No `authorize`, ainda `appointment_required` e `billing_unit_missing` (com `procedure_billing` utilizável e sem `tfd_billing_health_unit_id`).
12. **Situações do TFD** (`Tfd::Transition`, uma linha em `tfd_case_events` por mudança): `draft → submitted` (autor) · `submitted → returned | authorized | denied` (comissão) · `returned → submitted` (autor) · `authorized → scheduled` (primeiro trecho registrado) · `scheduled → travelling` (primeiro trecho de ida `done`) · `travelling → completed` (prestação de contas) · `draft|submitted|returned|authorized|scheduled|travelling → cancelled` (comissão; o autor só cancela o próprio `draft`). `appointment` e `inpatient` gravam evento sem mudar a situação (`kind: "appointment_set"`, `"inpatient_recorded"`).
13. **Trechos.** `fleet` exige `trip_passenger_id` de passageiro ativo do mesmo paciente numa viagem não cancelada com `origin: "tfd"` e `tfd_case_id` deste processo; `done` é marcado no fechamento da viagem se o paciente `boarded` (`Tfd::FleetLegs.complete!(trip)`). Passagem (`bus_ticket`, `air`, `river`) exige `ticket: { amount_cents, carrier }` e aceita `companion: true`; `done` por `POST /tfd/legs/:id/done`. A volta se pareia sozinha com a ida mais antiga sem volta e com data ≤ a dela (`outbound_leg_id`); sem ida aberta → `no_outbound_leg`. `distance_km` copiado do destino no INSERT.
14. **Ajuda de custo** (`Tfd::Allowances`). No trecho `done` (ida ou volta): uma linha de deslocamento por pessoa que viajou — `ground_km` (`fleet`, `bus_ticket`) com `quantity = ceil(km / 50)`; `air_miles` = `ceil(km / 1.609344 / 200)`; `river_miles` = `ceil(km / 1.852 / 27)`; mínimo 1. Na volta `done`: alimentação por pessoa do par — `noites = volta.date - ida.date`; `0` → `meal_day` × 1; senão `meal_overnight` × (`noites` − noites do PACIENTE internado no intervalo `[ida, volta)`); quantidade 0 → sem linha; o acompanhante não desconta internação (gate: manual da UF). Pessoa "acompanhante viajou" = `companion_attendance == "boarded"` (frota) ou `leg.companion` (passagem). `table_cents` = `amount_sa_cents` da SIGTAP da competência da data do trecho (`Billing::SigtapCache`; nulo → 0), `complement_cents` = `allowance_complements[code]` (0 se ausente), `total_cents = (table + complement) * quantity`. Códigos em `config/tfd/sigtap_codes.yml`. Linha imutável, única por `(tfd_leg_id, person, kind)`.
15. **BPA.** Uma linha de `performed_procedures` por `tfd_allowance` (`source: "tfd"`, `tfd_allowance_id` único), BPA-I pela crítica, `performed_on` = data do trecho, `professional_user` = autor do laudo, `cbo_code` = `tfd_cases.requester_cbo` (o médico não tem vínculo com a unidade de gestão: `Billing::Staff.cbo_for` daria nulo), `cid10` = o do laudo, estabelecimento = `Billing::Establishment.for_unit(tfd_billing_health_unit)`, `recorded_by: nil`. Com `procedure_billing` desligado no momento: nada (sem retroativo, como o laboratório).
16. **Cidadão.** `GET /citizen/trips?citizen_id=` no padrão do `RegulationRequestsController` (par validado da sessão; 404/403 `not_verified`); `sanitary_transport` utilizável; trilha `access: "citizen"` quando há paciente. Avisos de viagem são a terceira fonte do `/citizen/notices` (`kind: "trip"`), um por par validado do paciente.
17. **Exclusão LGPD.** `RequestErasure.examined_patients?` ganha `tfd_cased?(ids)` (`TfdCase.where(patient_id: ids).exists?`, rascunho inclusive). No ramo não retido, `Citizens::Erase` anonimiza os passageiros dos pacientes travados (`Transport::Anonymize.patients!(ids)`): acompanhante, documento, motivo, prestador, endereço, nota de exceção e motivo de retirada → `NULL`, `anonymized_at` = agora; presença e contagens ficam; séries do paciente suspensas a partir de hoje; `transport_trip_notices` dos pares apagados com os outros avisos.
18. **Trilha.** `Tfd::Trail.person_viewed!(user:, patient_id:)` = `clinical_record.viewed` com `access: "tfd"` em `POST /tfd/cases/:id/read` e no `GET /tfd/prefill` não (a consulta já deixa). Central, motorista, lista impressa, faltas, painel: sem trilha (não há dado clínico).

## Valores fixados por este plano

1. **Interruptores** (depois de `rnds_mock` no `CATALOG`):
   - `Entry.new(key: "sanitary_transport", description: "Transporte sanitário: frota, motoristas, destinos, viagens com lista de passageiros, séries recorrentes e tela do motorista", requires: [])`
   - `Entry.new(key: "tfd", description: "Tratamento Fora de Domicílio: laudo, comissão, trechos, ajuda de custo, pagamento registrado, prestação de contas e BPA", requires: %w[feature:sanitary_transport feature:clinical_record])`
2. **Papéis:** `transport_dispatcher` e `tfd_committee` em `Membership::ROLES` e `PRIVILEGED_ROLES`; `driver` só em `ROLES`. Lista nova (22, ordem alfabética) em `AddSanitaryTransport::ROLES`.
3. **`Transport::Catalog`:** `VEHICLE_KINDS = %w[van minibus bus car ambulance_a]`, `LICENSE_CATEGORIES = %w[B C D E]`, `LARGE_VEHICLE_SEATS = 8`, `LARGE_VEHICLE_CATEGORIES = %w[D E]`, `LICENSE_WARNING_DAYS = 30`, `TRIP_STATUSES = %w[planned confirmed in_progress done cancelled]`, `STOP_KINDS = %w[boarding destination]`, `ORIGINS = %w[regulation appointment tfd series exception]`, `EXCEPTION_REASONS = %w[follow_up_return legacy_patient scheduling_pending other]`, `ATTENDANCE = %w[pending boarded no_show]`, `NOTICE_KINDS = %w[added departure_changed boarding_changed cancelled removed reminder]`, `DEPARTURE_CHANGE = 30.minutes`, `SERIES_HORIZON_DAYS = 21`, `PENDING_REASONS = %w[no_seat no_wheelchair_spot]`, `EXPENSE_KINDS = %w[toll parking meal lodging other]`, `PLATE = /\A[A-Z]{3}\d[A-Z\d]\d{2}\z/` (depois de tirar `-` e pôr em maiúsculas), `RENAVAM = /\A\d{11}\z/`, `LICENSE_NUMBER = /\A\d{11}\z/`, `IBGE = /\A\d{7}\z/`, `CNES = /\A\d{7}\z/`, `SEATS = (1..60)`, `WHEELCHAIR_SPOTS = (0..10)`, `MAX_TRIP_HOURS = 72`, `ODOMETER = (0..9_999_999)`, `REASON = (10..500)`, `TEXT_MAX = 120`, `MODEL_MAX = 80`, `ADDRESS_MAX = 200`, `DEFAULT_COMPANION_RULE = { "max_age_child" => 15, "min_age_elderly" => 60 }`.
4. **`Tfd::Catalog`:** `STATUSES = %w[draft submitted returned authorized denied scheduled travelling completed cancelled]`, `EVENT_KINDS = STATUSES - %w[draft] + %w[created appointment_set inpatient_recorded]`, `DECISIONS = { "authorize" => "authorized", "return" => "returned", "deny" => "denied" }`, `MODES = %w[fleet bus_ticket air river]`, `TRANSPORT_MODES = MODES` (laudo), `DIRECTIONS = %w[outbound return]`, `ALLOWANCE_KINDS = %w[meal_overnight meal_day ground_km air_miles river_miles]`, `PERSONS = %w[patient companion]`, `PAYMENT_METHODS = %w[cash transfer pix check]`, `PHYSICIAN_PREFIX = "225"`, `DEFAULT_MIN_DISTANCE_KM = 50`, `KM_UNIT = 50`, `AIR_MILES_UNIT = 200`, `RIVER_NM_UNIT = 27`, `KM_PER_MILE = 1.609344`, `KM_PER_NM = 1.852`, `JUSTIFICATION = (20..2000)`, `REASON = (10..500)`, `RECEIPT_MAX = 40`, `CARRIER_MAX = 80`, `MAX_AMOUNT_CENTS = 10_000_000`, `TRAIL_ACCESS = "tfd"`, `FILE_PURPOSE = "tfd_accountability"`.
5. **YAML** (Task 9 e 11; todos com `version:`, `source:` e `review: true` — conferência no SIGTAP oficial é gate de go-live):
   - `config/tfd/sigtap_codes.yml`: `meal_overnight: { patient: "0803010010", companion: "0803010044" }`, `meal_day: { patient: "0803010028", companion: "0803010052" }`, `ground_km: { patient: "0803010125", companion: "0803010109" }`, `air_miles: { patient: "0803010087", companion: "0803010079" }`, `river_miles: { patient: "0803010117", companion: "0803010095" }`.
   - `config/tfd/primary_care_groups.yml`: `prefixes: ["01"]` (grupo 01, promoção e prevenção), `codes: ["0301010030", "0301010064"]` (consultas da atenção primária).
6. **Dev:** api do worktree na porta **3052**; worktree `apps/api/.claude/mod30` (com `config/master.key`); branch `feat/mod-30-sanitary-transport`; migração de cidade `20261103900001_add_sanitary_transport.rb`; nenhuma migração de plataforma. Semente (Task 18): Curitiba — `sanitary_transport` e `tfd` ligados; "Central de transporte" (`transporte@curitiba.dev`, `transport_dispatcher`), "Comissão de TFD" (`tfd@curitiba.dev`, `tfd_committee`), motorista "Davi Motorista" (`motorista@curitiba.dev`, `driver`, CNH D válida) e "Caio Condutor" (CNH B vencendo em 20 dias); van "ABC1D23" (15 lugares, 1 cadeirante) e carro "BRA2E19" (4 lugares); destinos Curitiba (4106902, 0 km), Ponta Grossa (4119905, 117 km), Londrina (4113700, 380 km) e Araucária (4101804, 27 km, metropolitana); unidade "Secretaria Municipal de Saúde" (`health_management`, CNES de dev) como `tfd_billing_health_unit_id`; uma série de hemodiálise seg/qua/sex para Ponta Grossa; um TFD `authorized` para Londrina com atendimento marcado em 7 dias.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (sempre os arquivos da task, por nome); commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-30.1..F-30.8 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada, começando com `gh`). Card vai para Done quando o ÚLTIMO app dele terminar (F-30.1 também dashboard e maintenance; F-30.2..F-30.6 também dashboard; F-30.7 só api — Done no fim deste plano; F-30.8 também wpda e dashboard); Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod30 <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. `spec/support/transport_helpers.rb` é arquivo novo de `spec/support` e a migração reescreve `ck_memberships_role` → a suíte completa é obrigatória no fim, pedida à sessão "Testes" (Task 18).
- Banco de cidade: `db/city_migrate/20261103900001_add_sanitary_transport.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_11_03_900001)`); triggers em `db/city_triggers.sql`, executado no fim do `up`. Irreversível. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error" }` (com `path` e `problems: [ { path, error } ]` quando há recusas de campo, todas de uma vez); interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie; registro de outro escopo por id → 404; lista com escopo alheio → 403; listas com envelope.
- **CPF, CNS, nome, CNH, documento do acompanhante, destino de tratamento (prestador, endereço), CID e justificativa nunca em log, evento, Analytics, erro ou URL.** Lista de passageiros, tela do motorista, aviso e SMS sem dado clínico (nunca procedimento, especialidade, CID, texto do pedido). Modelos novos com dado do paciente sobrescrevem `inspect`/`to_s`/`pretty_print`/`as_json` (só `id`). Filtros de log novos (Task 1): `/\A(license_number|companion|document|provider_name|address|exception_note|appointment_at|report|cid10|sigtap_code|justification|companion_reason|receipt_number|carrier|odometer_start|odometer_end|fuel|expenses|station|filename|data)\z/`.
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids e códigos de lista fechada; `DomainEvents.publish(` é multilinha — confira cada chamada inteira. Nenhum `Platform.audit` novo (R18 intacto); se surgir, entra em `R18_PLATFORM_EVENT_NAMES`.
- Nunca `Current.city =` solto; interruptores lidos **fora** da transação da cidade.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); coluna cifrada de tabela imutável no padrão `rota_reencryption_only(...)`.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..48)`.
- Confira nomes reais no código antes de usá-los (o PRÉ-REQUISITO lista os conferidos).

## Review Focus

1. **Viagens encostadas** (a van volta às 12:00 e sai de novo às 12:00 com outro grupo): aceitas pelo Ruby e pela constraint (`[)`); um minuto de cruzamento → `vehicle_overlap`, e com o mesmo motorista também `driver_overlap`, os dois em `problems[]`. Teste na Task 4.
2. **Série de hemodiálise às 05:30** (05:30 local = 08:30 UTC; um erro de fuso põe a viagem às 02:30 ou no dia errado): a viagem nasce com `date` do dia local e `departure_at.in_time_zone.strftime("%H:%M") == "05:30"`; rodar o job duas vezes no mesmo dia não duplica passageiro nem pendência. Teste na Task 7.
3. **Paciente internado o tempo todo no destino com acompanhante** (ida dia 10, volta dia 15, internado de 10 a 15): o paciente não recebe `meal_overnight`, o acompanhante recebe 5; os dois recebem o deslocamento. Teste na Task 11.
4. **Acompanhante faltou e o paciente embarcou** na viagem de frota do TFD: só o paciente tem `ground_km`, nenhuma linha de acompanhante no BPA, e a ajuda de custo do acompanhante não nasce na volta. Teste na Task 13.
5. **Central corrige a presença depois do BPA** (`boarded` → `no_show` num passageiro com trecho de frota `done`): 422 `tfd_leg_done`, a presença não muda, a linha do BPA fica; sem trecho de TFD, a correção grava `transport_attendance_corrections` e muda a presença sob o trigger. Teste na Task 8.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod30/`)
- `db/city_migrate/20261103900001_add_sanitary_transport.rb`.
- `config/tfd/{sigtap_codes,primary_care_groups}.yml`.
- `app/models/{transport_setting,transport_vehicle,transport_driver,transport_destination,transport_trip,transport_trip_stop,transport_trip_passenger,transport_trip_log,transport_series,transport_pending_item,transport_attendance_correction,transport_trip_notice,tfd_case,tfd_case_event,tfd_leg,tfd_allowance,tfd_payment,tfd_accountability}.rb`.
- `app/services/transport/{gate,catalog,access,json,overlap,occupancy,scheduled_care,manifest_pdf,companion_rule,no_shows,dashboard,citizen_view,sms_text,anonymize}.rb`; `app/services/tfd/{gate,catalog,json,trail,rules,primary_care,sigtap_codes,transition,allowances,fleet_legs,files}.rb`.
- `app/commands/transport/{save_settings,save_vehicle,save_driver,save_destination,save_trip,confirm_trip,cancel_trip,add_passenger,remove_passenger,save_series,suspend_series,materialize_series,start_trip,record_attendance,close_trip,correct_attendance,notify}.rb`; `app/commands/tfd/{save_case,submit_case,decide_case,cancel_case,set_appointment,record_inpatient,add_leg,complete_leg,record_payment,close_accountability}.rb`; `app/commands/billing/from_tfd.rb`.
- `app/controllers/concerns/{transport_gate,tfd_gate}.rb`; `app/controllers/transport/{base,vehicles,drivers,destinations,trips,passengers,scheduled_care,manifests,series,pending,no_shows,my_trips,dashboard}_controller.rb`, `app/controllers/transport/admin/settings_controller.rb`; `app/controllers/tfd/{base,cases,legs,prefill,files}_controller.rb`; `app/controllers/citizen_api/trips_controller.rb`.
- `app/jobs/transport/{materialize_series_job,notice_job,remind_job}.rb`; `app/jobs/billing/tfd_production_job.rb`.
- `app/services/timeline/sources/transport_source.rb`.
- `lib/transport_crew.rb`; `spec/support/transport_helpers.rb`; specs listados em cada task.

**Modificar**
- `app/services/platform/features.rb`, `app/models/{membership,health_unit,performed_procedure,care_provider,regulation_request,exam_file}.rb`, `app/commands/care_regulation/{schedule_external,save_provider}.rb`, `app/controllers/care_regulation/{decisions,admin/providers}_controller.rb`, `app/services/care_regulation/json.rb`, `app/commands/billing/record.rb`, `app/services/city_encryption.rb`, `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/controllers/citizen_api/notices_controller.rb`, `app/commands/citizens/{request_erasure,erase}.rb`, `app/services/exams/files.rb` (só aceitar o `purpose` novo), `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/adr_pointers_spec.rb`, `spec/models/{create_analytics_migration_spec,create_campaigns_migration_spec,membership_roles_spec}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/services/timeline/query_spec.rb:75`, `db/seeds.rb` (chama a crew).

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-30.1..F-30.8 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod30 -b feat/mod-30-sanitary-transport origin/main && cp config/master.key .claude/mod30/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO (cabeçalho) com `grep` no worktree (`VALID_RANGE`, `AddPharmacyStock::ROLES` nos dois specs de migração, `HealthUnit::KINDS`, `PerformedProcedure::SOURCES`, `Billing::Record.insert!`, `ScheduleExternal.call`, `ck_exam_files_purpose`, `Timeline::KINDS`); qualquer diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod30 spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb spec/models/membership_roles_spec.rb` → PASS (linha de base).

---

### Task 1: Esquema da cidade, modelos, triggers e helpers

**Files:**
- Create: `db/city_migrate/20261103900001_add_sanitary_transport.rb`; os 18 modelos do mapa; `spec/support/transport_helpers.rb`; `spec/models/transport_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{membership,health_unit,performed_procedure,care_provider,regulation_request,exam_file}.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/adr_pointers_spec.rb`, `spec/models/{create_analytics_migration_spec,create_campaigns_migration_spec,membership_roles_spec}.rb`

**Interfaces:**
- Produces (tabelas, todas `id: :uuid` salvo nota):
  - `transport_settings` (smallint `id` = 1, inserida na migração; `tfd_min_distance_km` int default 50 CHECK 1..1000; `companion_max_age_child` int default 15 CHECK 0..17; `companion_min_age_elderly` int default 60 CHECK 50..100; `tfd_billing_health_unit_id` FK `health_units` nulo; `allowance_complements jsonb` default `{}` CHECK `jsonb_typeof = 'object'`; `updated_by_user_id`; `updated_at`).
  - `transport_vehicles` (`plate` 7 único, `renavam` 11 único, `model` 80, `kind`, `renem_code` 20 nulo, `seats`, `wheelchair_spots`, `cnes` 7 nulo, `active`, `updated_by_user_id`, timestamps; CHECKs de domínio e faixas).
  - `transport_drivers` (`user_id` único FK, `license_number` text cifrado, `license_category`, `license_expires_on date`, `active`, `updated_by_user_id`, timestamps).
  - `transport_destinations` (`ibge_code` 7 único, `name` 120, `distance_km` int CHECK ≥ 0, `metropolitan` bool, `active`, `updated_by_user_id`, timestamps).
  - `transport_series` (`patient_id`, `transport_destination_id`, `provider_name`/`address` cifrados nulos, `weekdays smallint[]` CHECK `cardinality >= 1 AND weekdays <@ ARRAY[1,2,3,4,5,6,7]`, `appointment_time time`, `departure_time time`, `return_time time` CHECK `return_time > departure_time`, `boarding_label` 120, `starts_on`, `ends_on` nulo CHECK `>= starts_on`, `wheelchair`, `companion_name`/`companion_document`/`companion_reason` cifrados nulos, `suspended_from date` nulo, `suspend_reason` cifrado nulo, `created_by_user_id`, timestamps; CHECK `(suspended_from IS NULL) = (suspend_reason IS NULL)`).
  - `transport_trips` (`date`, `departure_at`, `return_at` datetime, `transport_vehicle_id`/`transport_driver_id` nulos, `status`, `transport_series_id` nulo, `cancel_reason` cifrado nulo, `cancelled_at`, `cancelled_by_user_id`, `confirmed_at`, `started_at`, `created_by_user_id`, timestamps; CHECKs `ck_transport_trips_status`, `ck_transport_trips_range` (`return_at > departure_at AND return_at <= departure_at + interval '72 hours'`), `ck_transport_trips_staffed` (`status NOT IN ('confirmed','in_progress','done') OR (transport_vehicle_id IS NOT NULL AND transport_driver_id IS NOT NULL)`), `ck_transport_trips_cancel` (`(status = 'cancelled') = (cancelled_at IS NOT NULL)`), `ck_transport_trips_date` (Desvio 1); exclusions do Desvio 2; índice `(date, status)`).
  - `transport_trip_stops` (`transport_trip_id`, `position` smallint, `kind`, `transport_destination_id` nulo, `label` 120; único `(trip, position)`; CHECK `(kind = 'destination') = (transport_destination_id IS NOT NULL)`).
  - `transport_trip_passengers` (`transport_trip_id`, `patient_id`, `transport_trip_stop_id`, `transport_destination_id`, `provider_name`/`address` cifrados, `appointment_at` nulo, `takes_seat`, `wheelchair`, `companion_name`/`companion_document`/`companion_reason` cifrados, `origin`, `regulation_request_id`/`appointment_id`/`tfd_case_id`/`transport_series_id` nulos, `exception_reason` nulo, `exception_note` cifrado, `attendance` default `pending`, `companion_attendance` nulo, `added_by_user_id` nulo (job), `removed_at`, `removed_by_user_id`, `remove_reason` cifrado, `anonymized_at`, timestamps; único parcial `(trip, patient) WHERE removed_at IS NULL` `idx_transport_trip_passengers_active`; CHECK `ck_transport_trip_passengers_origin` (`num_nonnulls(regulation_request_id, appointment_id, tfd_case_id, transport_series_id) = (origin <> 'exception')::int` e cada coluna ⇔ sua origem; `(origin = 'exception') = (exception_reason IS NOT NULL)`); CHECK `ck_transport_trip_passengers_companion` (`anonymized_at IS NOT NULL OR (companion_name IS NULL) = (companion_attendance IS NULL)`); CHECK de domínio de presença).
  - `transport_trip_logs` (`transport_trip_id` único, `odometer_start`, `odometer_end` CHECK `>= odometer_start`, `fuel jsonb`, `expenses jsonb` (arrays), `closed_at`, `closed_by_user_id`).
  - `transport_attendance_corrections` (`transport_trip_passenger_id`, `field` (`attendance`|`companion_attendance`), `from_value`, `to_value`, `reason` cifrado, `by_user_id`, `created_at`).
  - `transport_pending_items` (`transport_series_id`, `date`, `reason`, `resolved_at` nulo, `created_at`; único `(series, date)`).
  - `transport_trip_notices` (`transport_trip_passenger_id`, `citizen_id`, `kind`, `event_key` 80, `departure_at` (o da hora do aviso), `sms` bool, `read_at`, `created_at`; único `(passenger, citizen, kind, event_key)`).
  - `tfd_cases` (`patient_id`, `consultation_id`/`regulation_request_id` nulos, `requester_user_id`, `requester_cbo` 6, `report` text cifrado, `transport_destination_id`, `care_provider_id` nulo, `appointment_at` nulo, `inpatient_stays jsonb` default `[]`, `status`, `submitted_at`, `decided_at`, `version` default 1, timestamps; CHECK `num_nonnulls(consultation_id, regulation_request_id) >= 1`; CHECK `(status = 'draft') = (submitted_at IS NULL)`; índice `(status, submitted_at)`).
  - `tfd_case_events` (`tfd_case_id`, `kind`, `by_user_id` nulo, `reason` cifrado nulo, `created_at`).
  - `tfd_legs` (`tfd_case_id`, `direction`, `mode`, `date`, `distance_km`, `transport_trip_passenger_id` nulo único, `outbound_leg_id` nulo único (FK `tfd_legs`), `ticket_amount_cents` nulo, `ticket_carrier` 80 nulo, `companion` bool, `done_at` nulo, `created_by_user_id`, timestamps; CHECKs: `(mode = 'fleet') = (transport_trip_passenger_id IS NOT NULL)`, `(mode = 'fleet') = (ticket_amount_cents IS NULL)`, `(direction = 'return') = (outbound_leg_id IS NOT NULL)`, `mode <> 'fleet' OR NOT companion`).
  - `tfd_allowances` (`tfd_case_id`, `tfd_leg_id`, `person`, `kind`, `sigtap_code` 10, `competence` 6, `quantity`, `table_cents`, `complement_cents`, `total_cents`, `created_at`; único `(tfd_leg_id, person, kind)`).
  - `tfd_payments` (`tfd_case_id`, `paid_on`, `amount_cents` CHECK 1..10_000_000, `method`, `receipt_number` 40, `by_user_id`, `created_at`).
  - `tfd_accountability` (`tfd_case_id` único, `exam_file_id` único, `filename` cifrado, `closed_at`, `closed_by_user_id`).
- Produces (colunas e CHECKs): `care_providers.ibge_code` (7, nulo, CHECK formato); `regulation_requests.care_provider_id` (FK, nulo); `health_units.kind` aceita `health_management` (`HealthUnit::KINDS` + `health_management`); `performed_procedures.source` aceita `tfd` + `tfd_allowance_id` (FK, único parcial) + `tfd_leg_id` (FK, nulo), `ck_performed_procedures_source_ref` ganha `(source <> 'tfd' OR tfd_allowance_id IS NOT NULL)`; `PerformedProcedure::SOURCES` + `tfd`; `ck_memberships_role` com `AddSanitaryTransport::ROLES`; `ck_exam_files_purpose` aceita `tfd_accountability`.
- Produces (helpers de spec, `spec/support/transport_helpers.rb`): `transport_city!(tfd: false, billing: false)` (`clinical_city!`; liga `sanitary_transport` e, com `tfd: true`, `tfd`; com `billing: true`, `billing_city!` + `tfd_sigtap!` ANTES — a SIGTAP ativa é imutável); `tfd_sigtap!` (os 10 códigos do 08.03.01 com `amount_sa_cents` — `0803010125` = 495, `0803010028` = 840, `0803010044` = 2475, os demais 1000 — instrumento `02` e CBO `225125`); `dispatcher!`, `committee!`, `driver_user!(category: "D", expires_on: Date.current + 365)` (→ `TransportDriver`), `vehicle!(seats: 15, wheelchair_spots: 1, kind: "van")`, `destination!(name: "Ponta Grossa", ibge: "4119905", km: 117, metropolitan: false)`, `trip!(date: Date.current + 1, departure: "07:00", back: "18:00", vehicle: nil, driver: nil, status: "planned", destination: destination!)` (gravação direta, com uma parada de embarque "Praça Central" e uma de destino), `passenger!(trip, patient: transport_patient!, **over)` (gravação direta, `origin: "exception"`, `exception_reason: "legacy_patient"`), `transport_patient!(birth_date: "1980-05-10")` (paciente com par validado e telefone, pelo caminho do `exam_patient!`), `tfd_case!(status: "draft", patient: transport_patient!, destination: destination!(name: "Londrina", ibge: "4113700", km: 380), report: {})` (o `report` é mesclado sobre um laudo válido com CID `N185`, SIGTAP `0301010072`, `companion_indicated: true`; pelo `Tfd::SaveCase` da Task 9 quando existir; nesta task, gravação direta), `health_management_unit!(cnes: billing_cnes)`, `transport_admin!` (`municipal_admin` com `mfa_verified_at` recente). Helpers do 19a/21a/26a (`doctor!`, `started_consultation!`, `exam_patient!`) exigem as cidades deles: `transport_city!` monta sobre `billing_city!`/`clinical_city!` e, se algum helper pedir `exam_city!`, use-o como base — confira no Step 2.

- [ ] **Step 1: Write the failing test** `spec/models/transport_schema_spec.rb`:

```ruby
RSpec.describe "Esquema do módulo 30", type: :model do
  let!(:city) { transport_city! }

  it "o banco recusa o mesmo veículo em viagens que se cruzam e aceita as encostadas" do
    van = vehicle!
    trip!(vehicle: van, departure: "07:00", back: "12:00")
    expect { trip!(vehicle: van, departure: "12:00", back: "15:00") }.not_to raise_error
    expect { trip!(vehicle: van, departure: "11:59", back: "13:00") }
      .to raise_error(ActiveRecord::ExclusionViolation, /excl_transport_trips_vehicle/)
  end

  it "o mesmo motorista também, e viagem cancelada não ocupa" do
    driver = driver_user!.transport_driver
    first = trip!(driver: driver, departure: "07:00", back: "12:00")
    expect { trip!(driver: driver, departure: "08:00", back: "09:00") }
      .to raise_error(ActiveRecord::ExclusionViolation, /excl_transport_trips_driver/)
    first.update_columns(status: "cancelled", cancelled_at: Time.current)
    expect { trip!(driver: driver, departure: "08:00", back: "09:00") }.not_to raise_error
  end

  it "confirmada exige veículo e motorista; planned sem escala vale" do
    expect { trip!(status: "planned") }.not_to raise_error
    expect { trip!(status: "confirmed") }.to raise_error(ActiveRecord::StatementInvalid, /ck_transport_trips_staffed/)
  end

  it "passageiro sem origem precisa de motivo; um por viagem enquanto ativo" do
    trip = trip!
    patient = transport_patient!
    expect { passenger!(trip, patient: patient, exception_reason: nil) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_transport_trip_passengers_origin/)
    passenger!(trip, patient: patient)
    expect { passenger!(trip, patient: patient) }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "presença congela com a viagem done; correção só sob rota.transport_correction" do
    trip = trip!(vehicle: vehicle!, driver: driver_user!.transport_driver, status: "confirmed")
    rider = passenger!(trip)
    trip.update_columns(status: "in_progress")
    rider.update!(attendance: "boarded")
    trip.update_columns(status: "done")
    expect { rider.update!(attendance: "no_show") }.to raise_error(ActiveRecord::StatementInvalid, /attendance is frozen/)
    TransportTripPassenger.transaction do
      ApplicationRecord.connection.execute("SET LOCAL rota.transport_correction = 'on'")
      expect { rider.update!(attendance: "no_show") }.not_to raise_error
    end
  end

  it "evento e pagamento do TFD são só acréscimo; processo não some" do
    tfd = tfd_case!
    event = TfdCaseEvent.create!(tfd_case: tfd, kind: "created", by_user: tfd.requester_user)
    expect { event.update_column(:kind, "submitted") }.to raise_error(ActiveRecord::StatementInvalid)
    payment = TfdPayment.create!(tfd_case: tfd, paid_on: Date.current, amount_cents: 2475, method: "pix", receipt_number: "R-1", by_user: tfd.requester_user)
    expect { payment.delete }.to raise_error(ActiveRecord::StatementInvalid)
    expect { tfd.delete }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "papéis, unidade de gestão e fonte tfd entram nas travas" do
    expect(Membership::PRIVILEGED_ROLES).to include("transport_dispatcher", "tfd_committee")
    expect(Membership::PRIVILEGED_ROLES).not_to include("driver")
    expect { staff_with("motorista-#{SecureRandom.hex(2)}@cidade.gov.br", "driver") }.not_to raise_error
    expect { health_management_unit! }.not_to raise_error
  end

  it "nada pessoal em inspect/as_json" do
    rider = passenger!(trip!, companion_name: "Joana Acompanhante", companion_document: "12345678900", companion_attendance: "pending")
    expect([ rider.inspect, rider.as_json.to_s ].join).not_to include("Joana")
    expect(rider.as_json).to eq("id" => rider.id)
    expect(driver_user!.transport_driver.inspect).not_to match(/\d{11}/)
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod30 spec/models/transport_schema_spec.rb` → FAIL (`transport_city!` indefinido).
- [ ] **Step 3: Implementar** a migração no padrão da do 29 (constantes espelhadas no topo — `ROLES`, `HEALTH_UNIT_KINDS = %w[ubs upa hospital other ceo health_management]`, `SOURCES = %w[attendance consultation lab_report manual tfd]`, `EXAM_FILE_PURPOSES = %w[eqa_annual_report tfd_accountability]` e as listas dos Valores fixados 3 e 4 —, `replace_check`, `quoted`, `add_exclusion_constraint ... using: :gist`, `INSERT INTO transport_settings (id, updated_at) VALUES (1, now()) ON CONFLICT DO NOTHING`, `execute File.read(Rails.root.join("db/city_triggers.sql"))` no fim, `down` irreversível). `ck_transport_trips_date` usa `connection.quote(Current.city&.time_zone || "America/Sao_Paulo")`. Triggers novos em `db/city_triggers.sql` (bloco "Módulo 30 (ADR 0048)" no fim, no padrão do bloco do 29 com `to_regclass('public.transport_trips')`): `rota_transport_trip_passenger_guard()` (DELETE recusado; `transport_trip_id`, `patient_id`, `origin` e as quatro referências nunca mudam; `removed_at` uma vez; `attendance`/`companion_attendance` mudam só com a viagem `in_progress` ou com `current_setting('rota.transport_correction', true) = 'on'` — senão `'transport_trip_passengers: attendance is frozen'`; `anonymized_at` uma vez, e na mesma UPDATE as colunas cifradas só vão a NULL; sob `rota.reencrypting` só as cifradas); `rota_append_only()` em `transport_trip_logs`, `transport_attendance_corrections`, `tfd_case_events`, `tfd_allowances`, `tfd_payments` (as cifradas `reason` via `rota_reencryption_only`); `tfd_accountability` só acréscimo; `transport_trip_notices` só `read_at` uma vez (DELETE permitido, a exclusão apaga); `rota_no_delete()` em `transport_vehicles`, `transport_drivers`, `transport_destinations`, `transport_trips`, `transport_trip_stops` com passageiro (guarda própria: DELETE de parada só sem passageiro), `transport_series`, `tfd_cases`, `tfd_legs`, `transport_settings`; TRUNCATE recusado nas 18 tabelas. Modelos: `TfdAccountability.table_name = "tfd_accountability"`; `encrypts` (lista das Interfaces) todos em `CITY_KEYED_TARGETS`; `TfdCase#report` com `serialize :report, coder: JSON` sobre o texto cifrado; `TransportTripPassenger`, `TransportDriver`, `TransportSeries`, `TfdCase`, `TfdCaseEvent`, `TfdLeg`, `TfdAllowance`, `TfdPayment`, `TfdAccountability`, `TransportTripNotice` com `inspect`/`to_s`/`pretty_print`/`as_json` só `id`. `Membership::ROLES`/`PRIVILEGED_ROLES` com o comentário do padrão (`transport_dispatcher`: nome, telefone e embarque do paciente, monta a viagem; `tfd_committee`: lê CID e justificativa do laudo e registra pagamento — step-up, o mantenedor não concede; `driver` NÃO entra: só as próprias viagens do dia, sem dado clínico). Os dois specs de migração trocam `AddPharmacyStock::ROLES` por `AddSanitaryTransport::ROLES` (e o comentário ganha "transport_dispatcher/tfd_committee/driver (30)"); `membership_roles_spec.rb` ganha os três papéis. Dump à mão do `db/city_schema.rb`; filtros de log; `VALID_RANGE = (1..48)`.
- [ ] **Step 4:** `bin/test api -w mod30 spec/models/transport_schema_spec.rb spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/adr_pointers_spec.rb spec/requests/billing spec/requests/care_regulation` → PASS.
- [ ] **Step 5: Commit** `feat: add sanitary transport and tfd schema with overlap exclusion and immutability triggers`.

---

### Task 2: Interruptores, portões, bases e configuração da cidade

**Files:**
- Create: `app/services/transport/{gate,catalog,access}.rb`, `app/services/tfd/{gate,catalog}.rb`, `app/controllers/concerns/{transport_gate,tfd_gate}.rb`, `app/controllers/transport/base_controller.rb`, `app/controllers/tfd/base_controller.rb`, `app/commands/transport/save_settings.rb`, `app/controllers/transport/admin/settings_controller.rb`, `spec/services/transport/gate_spec.rb`, `spec/requests/transport/admin_settings_spec.rb`
- Modify: `app/services/platform/features.rb`, `spec/services/platform/features_spec.rb`, `config/routes.rb`

**Interfaces:**
- Produces: `Transport::Gate::KEY = "sanitary_transport"`, `Tfd::Gate::KEY = "tfd"`, `.usable?(city)`/`.enabled?(city)` (o padrão do `Vaccination::Gate`); concerns `TransportGate#require_transport!` e `TfdGate#require_tfd!` (403 `feature_disabled`); `Transport::Access.dispatcher?(user)`, `.committee?(user)`, `.driver_for(user) -> TransportDriver | nil` (Membership `driver` ativa E `TransportDriver` ativo); `Transport::BaseController` e `Tfd::BaseController` (os utilitários do `Vaccination::BaseController`: `render_failure` com `FAILURE_STATUS`, `render_refusal`, `body_param`, `body`, `query_only!`, `uuid?`, `forbid`, `not_found`, `require_step_up!` de `MfaStepUp`); `Transport::Settings.current -> TransportSetting` (linha 1); `GET/PATCH /transport/admin/settings` (`municipal_admin`; PATCH com step-up) com a forma do contrato §3.

- [ ] **Step 1: Write the failing tests:**

```ruby
# spec/services/transport/gate_spec.rb
RSpec.describe Transport::Gate do
  it "sanitary_transport sem pré-requisito; tfd exige sanitary_transport e clinical_record utilizáveis" do
    city = clinical_city!
    Platform::Features.set!(city: city, key: "tfd", enabled: true, maintainer: ledi_maintainer!)
    expect(Platform::Features.missing(city, "tfd")).to eq([ "sanitary_transport_disabled" ])
    Platform::Features.set!(city: city, key: "sanitary_transport", enabled: true, maintainer: ledi_maintainer!)
    expect(Tfd::Gate.usable?(city)).to be(true)
    city.update!(record_mode: "integrated")
    expect(Platform::Features.missing(city, "tfd")).to include("clinical_record_disabled")
  end
end
```

```ruby
# spec/requests/transport/admin_settings_spec.rb
RSpec.describe "Transporte — configuração", type: :request do
  let!(:city) { transport_city! }
  let(:admin) { staff_with("adm-transporte@cidade.gov.br", "municipal_admin").tap { |u| u.update!(mfa_verified_at: Time.current) } }

  it "lê o padrão e grava; a unidade do TFD precisa ser health_management com CNES" do
    sign_in_as(admin)
    get "/transport/admin/settings"
    expect(json_body).to eq("tfd_min_distance_km" => 50, "companion_rule" => { "max_age_child" => 15, "min_age_elderly" => 60 },
                            "tfd_billing_health_unit_id" => nil, "allowance_complements" => {})
    json_patch "/transport/admin/settings", tfd_billing_health_unit_id: create_unit("UBS Centro").id
    expect([ response.status, json_body["error"], json_body["path"] ]).to eq([ 422, "invalid_request", "tfd_billing_health_unit_id" ])
    unit = health_management_unit!
    json_patch "/transport/admin/settings", tfd_billing_health_unit_id: unit.id, allowance_complements: { "0803010044" => 5000 }, tfd_min_distance_km: 60
    expect(json_body).to include("tfd_min_distance_km" => 60, "allowance_complements" => { "0803010044" => 5000 })
  end

  it "complemento só para código do sigtap_codes.yml e em centavos inteiros; sem step-up → 401; desligado → 403" do
    sign_in_as(admin)
    json_patch "/transport/admin/settings", allowance_complements: { "0301010072" => 100 }
    expect(json_body["path"]).to eq("allowance_complements.0301010072")
    admin.update!(mfa_verified_at: 10.minutes.ago)
    json_patch "/transport/admin/settings", tfd_min_distance_km: 70
    expect(json_body).to eq("error" => "mfa_required")
    Platform::Features.set!(city: city, key: "sanitary_transport", enabled: false, maintainer: ledi_maintainer!)
    get "/transport/admin/settings"
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "sanitary_transport")
  end
end
```

`features_spec.rb`: o exemplo que confere o fim do catálogo (se houver) passa a terminar em `tfd`.

- [ ] **Step 2:** `bin/test api -w mod30 spec/services/transport/gate_spec.rb spec/requests/transport/admin_settings_spec.rb spec/services/platform/features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Entradas do Valor fixado 1; `Transport::SaveSettings.call(attrs:, by:)` → `Result.ok(settings:)` | `fail(:invalid_request, details: { path:, problems: })`; chaves desconhecidas → `invalid_request` com o nome; `companion_rule` com as duas chaves inteiras nas faixas da migração; `allowance_complements` só com chaves de `Tfd::SigtapCodes.all` (Task 11 — nesta task já nasce `Tfd::SigtapCodes` lendo o YAML do Valor fixado 5, com `all -> Array<String>` e `code(kind:, person:) -> String`) e valores inteiros 0..1_000_000. Rotas `scope "/transport", module: :transport, as: :transport` (`get/patch "admin/settings"`).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add sanitary transport and tfd switches with city settings`.

---

### Task 3: Frota, motoristas, destinos e prestador com município

**Files:**
- Create: `app/services/transport/json.rb`, `app/commands/transport/{save_vehicle,save_driver,save_destination}.rb`, `app/controllers/transport/{vehicles,drivers,destinations}_controller.rb`, `spec/requests/transport/fleet_spec.rb`, `spec/commands/care_regulation/schedule_external_provider_spec.rb`
- Modify: `config/routes.rb`, `app/commands/care_regulation/{schedule_external,save_provider}.rb`, `app/controllers/care_regulation/decisions_controller.rb` (`SCHEDULE_KEYS` + `care_provider_id`), `app/controllers/care_regulation/admin/providers_controller.rb` (aceita `ibge_code`), `app/services/care_regulation/json.rb` (`schedule.care_provider: { id, name }|null`; prestador com `ibge_code`), `config/initializers/domain_events.rb` + bindings spec (`transport.vehicle_saved`, `transport.driver_saved`, `transport.destination_saved`, `to: []`, só ids)

**Interfaces:**
- Produces: `Transport::Json.vehicle(v)`, `.driver(d, today: Time.zone.today)` (`license_status`: `expired` se `expires_on < today`, `expiring` se `<= today + 30`, senão `valid`; `license_number_last4`; nunca o número), `.destination(d)` — formas do contrato §2; rotas `GET/POST/PATCH /transport/{vehicles,drivers,destinations}[/:id]` (`transport_dispatcher`; listas `{ items: [...] }` ordenadas por nome/placa; `?active=true` filtra); recusas: `plate_taken`, `renavam_taken`, `user_already_driver`, `ibge_taken` → 409; `user_not_driver` (sem Membership `driver` ativa) → 422; `CareRegulation::ScheduleExternal.call(..., care_provider_id: nil)` (opcional: prestador ativo e não `own`; inválido → `invalid_request` `path: "care_provider_id"`); `CareRegulation::SaveProvider` aceita `ibge_code` (7 dígitos ou nulo; `own` → nulo forçado).

- [ ] **Step 1: Write the failing tests** (trechos de `fleet_spec.rb`):

```ruby
RSpec.describe "Transporte — frota", type: :request do
  let!(:city) { transport_city! }
  before { sign_in_as(dispatcher!) }

  it "cadastra veículo com placa Mercosul normalizada; placa repetida → 409" do
    json_post "/transport/vehicles", plate: "abc-1d23", renavam: "12345678901", model: "Sprinter 416", kind: "van", seats: 15, wheelchair_spots: 1, cnes: "1234567"
    expect(json_body).to include("plate" => "ABC1D23", "kind" => "van", "renem_code" => nil, "active" => true)
    json_post "/transport/vehicles", plate: "ABC1D23", renavam: "10987654321", model: "Outro", kind: "van", seats: 15, wheelchair_spots: 0
    expect([ response.status, json_body["error"] ]).to eq([ 409, "plate_taken" ])
  end

  it "recusas do veículo todas de uma vez" do
    json_post "/transport/vehicles", plate: "XX", renavam: "1", model: "", kind: "truck", seats: 0, wheelchair_spots: 11
    expect(json_body["problems"].map { |p| p["path"] }).to contain_exactly("plate", "renavam", "model", "kind", "seats", "wheelchair_spots")
  end

  it "motorista: só usuário com papel driver; CNH nunca sai inteira; status pela validade" do
    user = staff_with("davi@cidade.gov.br", "driver")
    other = staff_with("ana@cidade.gov.br", "viewer")
    json_post "/transport/drivers", user_id: other.id, license_number: "12345678901", license_category: "D", license_expires_on: (Date.current + 20).iso8601
    expect(json_body["error"]).to eq("user_not_driver")
    json_post "/transport/drivers", user_id: user.id, license_number: "12345678901", license_category: "D", license_expires_on: (Date.current + 20).iso8601
    expect(json_body).to include("license_status" => "expiring", "license_number_last4" => "8901", "user" => { "id" => user.id, "name" => user.name })
    expect(response.body).not_to include("12345678901")
  end

  it "destino com IBGE e km; intramunicipal com 0 km; papel errado → 403" do
    json_post "/transport/destinations", ibge_code: "4106902", name: "Curitiba", distance_km: 0, metropolitan: false
    expect(response).to have_http_status(:created)
    sign_in_as(committee!)
    get "/transport/destinations"
    expect([ response.status, json_body["error"] ]).to eq([ 403, "missing_role" ])
  end
end
```

`schedule_external_provider_spec.rb`: agendar com `care_provider_id` de prestador externo grava a coluna e o JSON traz `schedule.care_provider`; `own` ou inativo → `invalid_request` `path: "care_provider_id"`; sem a chave, tudo como antes (o spec existente do 26a segue verde).

- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/transport/fleet_spec.rb spec/commands/care_regulation/schedule_external_provider_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Um comando por recurso (`call(attrs:, by:, record: nil) -> Result.ok(record:)`), regras do Valor fixado 3, unicidade por `rescue ActiveRecord::RecordNotUnique` pelo nome do índice. Escritas sem step-up (o papel já exige segundo fator no login; contrato §4 não marca).
- [ ] **Step 4:** mesmos specs + `spec/requests/care_regulation spec/commands/care_regulation` → PASS.
- [ ] **Step 5: Commit** `feat: add transport fleet, drivers and destinations; providers gain the municipality`.

---

### Task 4: Viagens — criar, alterar, confirmar e cancelar

**Files:**
- Create: `app/services/transport/{overlap,occupancy}.rb`, `app/commands/transport/{save_trip,confirm_trip,cancel_trip}.rb`, `app/controllers/transport/trips_controller.rb`, `spec/requests/transport/trips_spec.rb`, `spec/services/transport/overlap_spec.rb`
- Modify: `app/services/transport/json.rb` (`trip`), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec

**Interfaces:**
- Consumes: Tasks 1–3.
- Produces: `Transport::Overlap.problems(trip) -> Array<{ path:, error: }>` (`vehicle_overlap` em `vehicle_id`, `driver_overlap` em `driver_id`; ignora a própria viagem e as canceladas); `Transport::Occupancy.of(trip) -> { seats_used:, seats:, wheelchair_used:, wheelchair_spots: }` (`seats`/`wheelchair_spots` nulos sem veículo); `Transport::SaveTrip.call(attrs:, by:, trip: nil) -> Result.ok(trip:)`; `Transport::ConfirmTrip.call(trip:, by:)`; `Transport::CancelTrip.call(trip:, reason:, by:)`; `Transport::Json.trip(trip, viewer: :dispatcher | :driver)`; rotas `GET /transport/trips?date=&vehicle_id=` (`date` obrigatória, `invalid_request` `path: "date"`), `POST /transport/trips`, `PATCH /transport/trips/:id`, `POST /transport/trips/:id/confirm`, `POST /transport/trips/:id/cancel`. Corpo `stops`: `[ { "id"?, "kind", "label", "destination_id"? } ]` na ordem (posição = índice + 1; pelo menos uma de cada tipo; embarques antes dos destinos); parada com passageiro não sai (`stop_in_use`). Eventos: `transport.trip_confirmed` (`trip_id`), `transport.trip_cancelled` (`trip_id`), `transport.trip_changed` (`trip_id`, `change: "departure" | "boarding"`, `stop_id`?) — só quando a viagem está `confirmed`/`in_progress` e a saída mudou ≥ 30 min ou o rótulo de uma parada de embarque mudou.

- [ ] **Step 1: Write the failing tests** (trechos de `trips_spec.rb`):

```ruby
RSpec.describe "Transporte — viagens", type: :request do
  let!(:city) { transport_city! }
  let(:van) { vehicle!(seats: 15) }
  let(:driver) { driver_user!(category: "D").transport_driver }
  let(:dest) { destination! }
  let(:day) { Date.current + 2 }
  def at(hhmm) = Time.zone.parse("#{day} #{hhmm}").iso8601
  def body(dep, back, **over)
    { date: day.iso8601, departure_at: at(dep), return_at: at(back), vehicle_id: van.id, driver_id: driver.id,
      stops: [ { kind: "boarding", label: "Praça Central" }, { kind: "destination", label: "Ponta Grossa", destination_id: dest.id } ] }.merge(over)
  end

  before { sign_in_as(dispatcher!) }

  it "Review Focus 1: encostadas valem; cruzando, veículo e motorista nas recusas juntos" do
    json_post "/transport/trips", body("07:00", "12:00")
    expect(response).to have_http_status(:created)
    json_post "/transport/trips", body("12:00", "15:00")
    expect(response).to have_http_status(:created)
    json_post "/transport/trips", body("11:59", "13:00")
    expect(json_body["problems"]).to contain_exactly({ "path" => "vehicle_id", "error" => "vehicle_overlap" }, { "path" => "driver_id", "error" => "driver_overlap" })
  end

  it "a corrida que escapa do Ruby vira 422 pela constraint, nunca 500" do
    json_post "/transport/trips", body("07:00", "12:00")
    allow(Transport::Overlap).to receive(:problems).and_return([])
    json_post "/transport/trips", body("08:00", "09:00")
    expect([ response.status, json_body["error"] ]).to eq([ 422, "vehicle_overlap" ])
  end

  it "planned sem escala; confirmar exige veículo, motorista, CNH válida na data e categoria D para mais de 8 lugares" do
    json_post "/transport/trips", body("07:00", "18:00", vehicle_id: nil, driver_id: nil)
    trip_id = json_body["id"]
    expect(json_body).to include("status" => "planned", "vehicle" => nil, "driver" => nil)
    json_post "/transport/trips/#{trip_id}/confirm"
    expect(json_body["problems"].map { |p| p["error"] }).to contain_exactly("vehicle_required", "driver_required")
    car_driver = driver_user!(category: "B", expires_on: day - 1).transport_driver
    json_patch "/transport/trips/#{trip_id}", vehicle_id: van.id, driver_id: car_driver.id
    json_post "/transport/trips/#{trip_id}/confirm"
    expect(json_body["problems"].map { |p| p["error"] }).to contain_exactly("driver_license_expired", "driver_license_category")
  end

  it "volta antes da ida, data fora do dia local e viagem de mais de 72 h" do
    json_post "/transport/trips", body("18:00", "07:00")
    expect(json_body["problems"]).to include({ "path" => "return_at", "error" => "return_before_departure" })
    json_post "/transport/trips", body("07:00", "18:00", date: (day + 1).iso8601)
    expect(json_body["problems"]).to include({ "path" => "date", "error" => "invalid_request" })
  end

  it "cancelar exige motivo, avisa por evento e libera o veículo" do
    json_post "/transport/trips", body("07:00", "12:00")
    id = json_body["id"]
    json_post "/transport/trips/#{id}/confirm"
    json_post "/transport/trips/#{id}/cancel", reason: "curto"
    expect(json_body["path"]).to eq("reason")
    expect { json_post "/transport/trips/#{id}/cancel", reason: "Veículo quebrado na véspera" }
      .to change { DomainEvent.where(name: "transport.trip_cancelled").count }.by(1)
    json_post "/transport/trips", body("08:00", "09:00")
    expect(response).to have_http_status(:created)
  end

  it "mudança da saída em 30 min ou mais na confirmada publica trip_changed; 29 min não" do
    json_post "/transport/trips", body("07:00", "18:00")
    id = json_body["id"]
    json_post "/transport/trips/#{id}/confirm"
    expect { json_patch "/transport/trips/#{id}", departure_at: at("07:29") }.not_to(change { DomainEvent.where(name: "transport.trip_changed").count })
    expect { json_patch "/transport/trips/#{id}", departure_at: at("08:00") }.to change { DomainEvent.where(name: "transport.trip_changed").count }.by(1)
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/transport/trips_spec.rb spec/services/transport/overlap_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `SaveTrip` valida forma (todas as recusas em `problems[]`), depois `Overlap.problems`, depois grava em `transaction(requires_new: true)` com `trip.lock!` no update; `rescue ActiveRecord::ExclusionViolation => e` → `Result.fail(e.message.include?("excl_transport_trips_driver") ? :driver_overlap : :vehicle_overlap)` (o `render_failure` põe `problems: [{ path, error }]`). Troca de veículo em viagem com passageiros confere a lotação nova (`seats_full`/`wheelchair_full`). Só `planned`/`confirmed` aceitam PATCH; `cancel` de `planned`/`confirmed` (`invalid_transition` nos outros). `GET` sem `sanitary_transport` → 403; motorista e comissão → 403 `missing_role`.
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add transport trips with database-enforced vehicle and driver overlap`.

---

### Task 5: Passageiros e atendimentos marcados

**Files:**
- Create: `app/services/transport/{scheduled_care,companion_rule}.rb`, `app/commands/transport/{add_passenger,remove_passenger}.rb`, `app/controllers/transport/{passengers,scheduled_care}_controller.rb`, `spec/requests/transport/passengers_spec.rb`, `spec/services/transport/scheduled_care_spec.rb`, `spec/services/transport/companion_rule_spec.rb`
- Modify: `app/services/transport/json.rb` (`passenger`), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec

**Interfaces:**
- Consumes: Task 4 (`Occupancy`, `Json.trip`); `Patients::Pairs.of`; `Tfd::Catalog`.
- Produces: `Transport::CompanionRule.check(patient:, on:, origin:, tfd_case:, reason:) -> :ok | :companion_not_allowed` (Desvio 5); `Transport::AddPassenger.call(trip:, attrs:, by:) -> Result.ok(passenger:)` (recusas do contrato §4 + `trip_not_open` fora de `planned`/`confirmed`, `stop_not_boarding`); `Transport::RemovePassenger.call(passenger:, reason:, by:)` (grava `removed_*`; `trip_not_open`); `Transport::ScheduledCare.call(date:, destination_id: nil, health_unit_id: nil) -> Array<Hash>`; `Transport::Json.passenger(p, viewer:)` — forma §2 (`viewer: :driver` sem `cns_masked`/`has_cpf`); rotas `POST /transport/trips/:id/passengers`, `DELETE /transport/trips/:id/passengers/:pid` (corpo `{ reason }`), `POST /transport/scheduled_care`. Eventos `transport.passenger_added` / `transport.passenger_removed` (`trip_id`, `passenger_id`).

- [ ] **Step 1: Write the failing tests** (trechos):

```ruby
RSpec.describe "Transporte — passageiros", type: :request do
  let!(:city) { transport_city!(tfd: true) }
  let(:trip) { trip!(vehicle: vehicle!(seats: 2, wheelchair_spots: 1)) }
  let(:stop) { trip.transport_trip_stops.find_by!(kind: "boarding") }
  def add(patient, **over)
    json_post "/transport/trips/#{trip.id}/passengers", { patient_id: patient.id, origin: "exception", exception_reason: "legacy_patient",
                                                          stop_id: stop.id, destination_id: trip.transport_trip_stops.find_by!(kind: "destination").transport_destination_id,
                                                          takes_seat: true, wheelchair: false }.merge(over)
  end

  before { sign_in_as(dispatcher!) }

  it "lotação conta passageiro e acompanhante; cadeira tem vaga própria" do
    add(transport_patient!(birth_date: "2015-01-01"), companion: { name: "Mãe da Criança", document: "12345678900" })
    expect(json_body).to include("companion" => { "name" => "Mãe da Criança", "document_last4" => "8900" }, "companion_attendance" => "pending")
    add(transport_patient!)
    expect(json_body["error"]).to eq("seats_full")
    add(transport_patient!, takes_seat: false, wheelchair: true)
    expect(response).to have_http_status(:created)
    add(transport_patient!, takes_seat: false, wheelchair: true)
    expect(json_body["error"]).to eq("wheelchair_full")
  end

  it "sem origem nem motivo → origin_required; other sem nota → exception_note; atendimento de outro paciente → origin_mismatch" do
    add(transport_patient!, origin: nil, exception_reason: nil)
    expect(json_body["error"]).to eq("origin_required")
    add(transport_patient!, exception_reason: "other")
    expect(json_body["path"]).to eq("exception_note")
    other_tfd = tfd_case!(status: "authorized")
    add(transport_patient!, origin: "tfd", origin_id: other_tfd.id, exception_reason: nil)
    expect(json_body["error"]).to eq("origin_mismatch")
  end

  it "acompanhante de adulto de 40 anos exige motivo; no TFD só com indicação no laudo" do
    add(transport_patient!(birth_date: "1986-01-01"), companion: { name: "Irmão" })
    expect(json_body["error"]).to eq("companion_not_allowed")
    patient = transport_patient!(birth_date: "1986-01-01")
    tfd = tfd_case!(status: "authorized", patient: patient, report: { "companion_indicated" => false })
    add(patient, origin: "tfd", origin_id: tfd.id, exception_reason: nil, companion: { name: "Irmão", reason: "Paciente com baixa visão" })
    expect(json_body["error"]).to eq("companion_not_allowed")
  end

  it "nunca dado clínico nem CPF na forma do passageiro" do
    patient = transport_patient!
    tfd = tfd_case!(status: "authorized", patient: patient, report: { "justification" => "Sem nefrologia no município para hemodiálise" })
    add(patient, origin: "tfd", origin_id: tfd.id, exception_reason: nil)
    expect(response.body).not_to include("N185", "nefrologia", "0301010072", patient.cpf)
    expect(json_body["patient"].keys).to contain_exactly("id", "display_name", "has_cpf", "cns_masked", "phone_masked")
  end

  it "retirar exige motivo e libera a vaga; mesmo paciente de volta entra" do
    patient = transport_patient!
    add(patient)
    pid = json_body["id"]
    json_delete_with_body "/transport/trips/#{trip.id}/passengers/#{pid}", reason: "Paciente avisou que não vai"
    expect(response).to have_http_status(:ok)
    add(patient)
    expect(response).to have_http_status(:created)
  end
end
```

`companion_rule_spec.rb`: 15 anos e 364 dias → `:ok`; 16 anos → recusa sem motivo; 60 anos completos no dia da viagem → `:ok`; 59 → recusa; regra da cidade 12/65 respeitada. `scheduled_care_spec.rb`: pedido do 26a `scheduled` com `care_provider_id` em município cadastrado volta com `destination.destination_id` e `provider_name` do prestador e **sem** `schedule_place`; horário do 17 só com `health_unit_id`; TFD `authorized` com `appointment_at` na data; quem já está em viagem ativa com a mesma origem fica de fora; nada de alvo, CID ou justificativa no Hash (`to_s`).

(`json_delete_with_body(path, params)` = o `json_delete` com corpo do `pharmacy_helpers`; se já existir com outro nome, use-o.)

- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/transport/passengers_spec.rb spec/services/transport/scheduled_care_spec.rb spec/services/transport/companion_rule_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Ordem no `AddPassenger`: forma → paciente/parada/destino (`not_found`) → origem (Desvio 4) → acompanhante → `trip.lock!` → `trip_not_open` → `patient_already_on_trip` (consulta + `rescue RecordNotUnique`) → lotação (Desvio 3) → INSERT. `appointment_at` padrão: o da origem (`scheduled_at` do pedido ou do horário, `appointment_at` do TFD); `provider_name` padrão: o do Desvio 6. Telefone mascarado do primeiro par validado com telefone (`CitizenIdentity::Phone.mask`).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add trip passengers from scheduled care with seat and companion rules`.

---

### Task 6: Lista de passageiros impressa e faltas

**Files:**
- Create: `app/services/transport/{manifest_pdf,no_shows}.rb`, `app/controllers/transport/{manifests,no_shows}_controller.rb`, `spec/services/transport/manifest_pdf_spec.rb`, `spec/requests/transport/no_shows_spec.rb`
- Modify: `config/routes.rb` (`get "trips/:id/manifest.pdf", to: "manifests#show", format: false`; `get "no_shows"`)

**Interfaces:**
- Produces: `Transport::ManifestPdf.call(trip) -> String` (bytes; folha de `Consultations::Print.document` com rodapé próprio; cabeçalho: cidade, data, saída, retorno, veículo (placa, modelo), motorista (nome); por parada de embarque, em ordem: nome do passageiro (`display_name`), destino (`destination.name` + `provider_name`), hora do atendimento, acompanhante (nome), "cadeira de rodas" quando for; linhas de assinatura vazias para embarque); `Transport::NoShows.call(from:, to:) -> Array<{ patient: { id, display_name }, count:, dates: [] }>` (passageiros `no_show` de viagens `done` no período, ordem por contagem decrescente); `GET /transport/no_shows?from=&to=` (até 366 dias; `invalid_request` fora).

- [ ] **Step 1: Write the failing tests:** `manifest_pdf_spec.rb` — com `PDF::Reader`, o texto contém o nome, "Praça Central", "Ponta Grossa", a hora e "Mãe da Criança"; **não** contém o CPF, o CNS, o CID e a justificativa do pedido do 26a usado como origem, nem `target_name` ("Consulta em Cardiologia"); passageiro retirado não aparece. `no_shows_spec.rb` — duas faltas do mesmo paciente em viagens `done` → `count: 2` com as duas datas; viagem `in_progress` não conta; comissão → 403.
- [ ] **Step 2:** `bin/test api -w mod30 spec/services/transport/manifest_pdf_spec.rb spec/requests/transport/no_shows_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (texto por `Consultations::Print.safe`; `send_data ..., type: "application/pdf", disposition: "inline"`; nada a log).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: print the passenger manifest without clinical data and list no-shows`.

---

### Task 7: Séries recorrentes, job e pendências

**Files:**
- Create: `app/commands/transport/{save_series,suspend_series,materialize_series}.rb`, `app/controllers/transport/{series,pending}_controller.rb`, `app/jobs/transport/materialize_series_job.rb`, `spec/commands/transport/materialize_series_spec.rb`, `spec/requests/transport/series_spec.rb`
- Modify: `config/routes.rb`, `config/recurring.yml` (`transport_materialize_series`: `Transport::MaterializeSeriesJob`, `housekeeping`, `"every day at 2am America/Sao_Paulo"`), `app/services/transport/json.rb` (`series`)

**Interfaces:**
- Consumes: Tasks 4–5 (`AddPassenger` com `by: nil` e `origin: "series"`).
- Produces: `Transport::SaveSeries.call(attrs:, by:, series: nil)` (corpo do contrato §4 + `return_time` e `boarding_label` obrigatórios — Divergência D5); `Transport::SuspendSeries.call(series:, from_on:, reason:, by:)` (retira o paciente das viagens `planned`/`confirmed` com `date >= from_on` e `transport_series_id` da série, `remove_reason` = o motivo; resolve as pendências futuras); `Transport::MaterializeSeries.call(series:, today:) -> { added:, created_trips:, pending: }` (algoritmo abaixo); `Transport::MaterializeSeriesJob` (`prepend EachCityJob`, `return unless Gate.usable?(Current.city)`, uma série por transação, falha por série no log só com id e classe); `GET /transport/pending` → `{ items: [ { series_id, date, reason } ] }` (não resolvidas, `date >= hoje`).

Algoritmo de `MaterializeSeries` (por data `d` em `today..today + 20`, `d.cwday` em `weekdays`, `starts_on <= d`, `ends_on.nil? || d <= ends_on`, `suspended_from.nil? || d < suspended_from`):

```ruby
next if TransportTripPassenger.joins(:transport_trip).where(transport_series_id: series.id, removed_at: nil, transport_trips: { date: d }).exists?
next if TransportPendingItem.exists?(transport_series_id: series.id, date: d, resolved_at: nil)
candidates = TransportTrip.where(date: d, status: %w[planned confirmed])
                          .where(id: TransportTripStop.where(kind: "destination", transport_destination_id: series.transport_destination_id).select(:transport_trip_id))
                          .order(Arel.sql("abs(extract(epoch from departure_at - #{quoted_departure(d)}))"), :id).lock.to_a
trip = candidates.find { |t| Occupancy.fits?(t, takes_seat: !series.wheelchair, wheelchair: series.wheelchair, companion: series.companion_name.present?) }
if trip.nil? && candidates.empty?
  trip = create_planned_trip!(series, d) # sem veículo e motorista; paradas [boarding_label, destino]; transport_series_id = série
elsif trip.nil?
  TransportPendingItem.create!(transport_series: series, date: d, reason: series.wheelchair ? "no_wheelchair_spot" : "no_seat")
  next
end
add_passenger!(trip, series, d) # parada de embarque de mesmo rótulo (sem distinção de caixa) ou nova antes do 1º destino
```

- [ ] **Step 1: Write the failing test** `materialize_series_spec.rb` (trechos):

```ruby
RSpec.describe Transport::MaterializeSeries do
  let!(:city) { transport_city! }
  let(:dest) { destination! }
  let(:monday) { Date.current.next_occurring(:monday) }
  let(:series) do
    Transport::SaveSeries.call(attrs: { "patient_id" => transport_patient!.id, "destination_id" => dest.id, "weekdays" => [ 1, 3, 5 ],
                                        "appointment_time" => "07:00", "departure_time" => "05:30", "return_time" => "13:00",
                                        "boarding_label" => "Posto do Distrito", "starts_on" => monday.iso8601, "wheelchair" => false },
                               by: dispatcher!).payload[:series]
  end

  it "Review Focus 2: cria planned às 05:30 locais nos dias da série, e rodar de novo não duplica" do
    described_class.call(series: series, today: monday)
    trips = TransportTrip.where(transport_series_id: series.id).order(:date)
    expect(trips.map { |t| t.date.cwday }.uniq).to eq([ 1, 3, 5 ])
    expect(trips.map { |t| t.departure_at.in_time_zone.strftime("%H:%M") }.uniq).to eq([ "05:30" ])
    expect(trips.all? { |t| t.departure_at.in_time_zone.to_date == t.date }).to be(true)
    expect(trips.count).to eq(9)
    expect { described_class.call(series: series, today: monday) }.not_to(change { TransportTripPassenger.count })
  end

  it "entra na viagem confirmada do dia com vaga; cheia vira pendência, nunca excesso" do
    full = trip!(date: monday, vehicle: vehicle!(seats: 1, wheelchair_spots: 0), driver: driver_user!.transport_driver, status: "confirmed", destination: dest)
    passenger!(full)
    described_class.call(series: series, today: monday)
    expect(TransportPendingItem.find_by(transport_series_id: series.id, date: monday).reason).to eq("no_seat")
    expect(full.transport_trip_passengers.where(removed_at: nil).count).to eq(1)
  end

  it "suspender tira das viagens futuras e para de criar" do
    described_class.call(series: series, today: monday)
    Transport::SuspendSeries.call(series: series, from_on: monday + 2, reason: "Transplante realizado, fim da diálise", by: dispatcher!)
    left = TransportTripPassenger.joins(:transport_trip).where(transport_series_id: series.id, removed_at: nil)
    expect(left.pluck("transport_trips.date")).to eq([ monday ])
  end
end
```

`series_spec.rb`: `weekdays` fora de 1..7, `return_time <= departure_time`, `ends_on < starts_on` → `problems[]`; `GET /transport/pending` traz a pendência; o job com `sanitary_transport` desligado não faz nada.

- [ ] **Step 2:** `bin/test api -w mod30 spec/commands/transport/materialize_series_spec.rb spec/requests/transport/series_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`Occupancy.fits?(trip, takes_seat:, wheelchair:, companion:) -> Boolean`; horas locais por `Time.zone.parse("#{d} #{hh:mm}")`; viagem nova também confere `Overlap` — sem escala não há).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: materialize recurring transport series into daily trips with pending items`.

---

### Task 8: Tela do motorista, presença, fechamento e correção

**Files:**
- Create: `app/commands/transport/{start_trip,record_attendance,close_trip,correct_attendance}.rb`, `app/services/tfd/fleet_legs.rb` (só a assinatura e o laço; o corpo de `complete!` chama `Tfd::CompleteLeg` da Task 11 — nesta task, stub que devolve `[]` e o spec da Task 11 o cobre), `app/controllers/transport/my_trips_controller.rb`, `spec/requests/transport/my_trips_spec.rb`, `spec/commands/transport/correct_attendance_spec.rb`
- Modify: `app/controllers/transport/trips_controller.rb` (`attendance_correction`), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`transport.trip_started`, `transport.trip_done`, `transport.attendance_corrected`, `to: []`)

**Interfaces:**
- Produces: `GET /transport/my_trips?date=` (padrão hoje; só `transport_driver_id` do motorista; `Json.trip(viewer: :driver)`), `POST /transport/my_trips/:id/start` (`confirmed` e `date == hoje`, senão `invalid_transition`/`not_today`), `POST /transport/my_trips/:id/attendance` (`in_progress`; `items[]` parciais; passageiro de outra viagem → 404; `companion_attendance` só onde há acompanhante), `POST /transport/my_trips/:id/close` (`odometer_invalid`, `attendance_pending`; grava `transport_trip_logs`, `done`, chama `Tfd::FleetLegs.complete!(trip)` na mesma transação); viagem de outro motorista → 404. `POST /transport/trips/:id/attendance_correction` (`transport_dispatcher`, step-up) `{ passenger_id, field, to, reason }` → `Transport::CorrectAttendance.call(...)` (Desvio 9). Fechamento: `fuel[]` = `{ liters (0 < x <= 1000, até 2 casas), amount_cents (1..10_000_000), station (1..120) }`, `expenses[]` = `{ kind ∈ EXPENSE_KINDS, amount_cents, note (até 120) }`, até 20 de cada.

- [ ] **Step 1: Write the failing tests** (trechos):

```ruby
RSpec.describe "Transporte — motorista", type: :request do
  let!(:city) { transport_city! }
  let(:davi) { driver_user! }
  let(:trip) { trip!(date: Date.current, vehicle: vehicle!, driver: davi.transport_driver, status: "confirmed") }
  let!(:rider) { passenger!(trip, companion_name: "Acompanhante", companion_attendance: "pending") }

  before { sign_in_as(davi) }

  it "vê só as próprias viagens do dia, sem CNS nem CPF" do
    trip!(date: Date.current, vehicle: vehicle!(plate_seed: 2), driver: driver_user!.transport_driver, status: "confirmed")
    get "/transport/my_trips"
    expect(json_body["items"].map { |t| t["id"] }).to eq([ trip.id ])
    expect(json_body["items"][0]["passengers"][0]["patient"].keys).to contain_exactly("id", "display_name", "phone_masked")
  end

  it "sai, marca presença do paciente e do acompanhante e fecha; pendente bloqueia o fechamento" do
    json_post "/transport/my_trips/#{trip.id}/start"
    expect(json_body["status"]).to eq("in_progress")
    json_post "/transport/my_trips/#{trip.id}/attendance", items: [ { passenger_id: rider.id, attendance: "boarded" } ]
    json_post "/transport/my_trips/#{trip.id}/close", odometer_start: 1000, odometer_end: 1240, fuel: [], expenses: []
    expect(json_body["error"]).to eq("attendance_pending")
    json_post "/transport/my_trips/#{trip.id}/attendance", items: [ { passenger_id: rider.id, attendance: "boarded", companion_attendance: "no_show" } ]
    json_post "/transport/my_trips/#{trip.id}/close", odometer_start: 1240, odometer_end: 1000, fuel: [], expenses: []
    expect(json_body["error"]).to eq("odometer_invalid")
    json_post "/transport/my_trips/#{trip.id}/close", odometer_start: 1000, odometer_end: 1240,
                                                       fuel: [ { liters: 40.5, amount_cents: 25_000, station: "Posto Rota" } ], expenses: [ { kind: "toll", amount_cents: 1250, note: nil } ]
    expect(json_body).to include("status" => "done")
    expect(json_body["log"]).to include("odometer_start" => 1000, "odometer_end" => 1240, "closed_by_name" => davi.name)
  end

  it "não sai em viagem de amanhã; viagem de outro motorista → 404" do
    tomorrow = trip!(date: Date.current + 1, vehicle: vehicle!(plate_seed: 3), driver: davi.transport_driver, status: "confirmed")
    json_post "/transport/my_trips/#{tomorrow.id}/start"
    expect(json_body["error"]).to eq("not_today")
    other = trip!(date: Date.current, vehicle: vehicle!(plate_seed: 4), driver: driver_user!.transport_driver, status: "confirmed")
    json_post "/transport/my_trips/#{other.id}/start"
    expect(response).to have_http_status(:not_found)
  end
end
```

`correct_attendance_spec.rb` (**Review Focus 5**): viagem `done` com passageiro `boarded` sem TFD → correção para `no_show` grava `TransportAttendanceCorrection` (`from_value: "boarded"`, `to_value: "no_show"`) e muda a presença; com `TfdLeg` de frota `done_at` preenchido ligado ao passageiro → `Result.fail(:tfd_leg_done)`, presença intacta; motivo com 9 caracteres → `invalid_request`; sem step-up → 401 (via request).

(`vehicle!(plate_seed:)` gera placas distintas; acrescente ao helper.)

- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/transport/my_trips_spec.rb spec/commands/transport/correct_attendance_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`CloseTrip`: `trip.lock!`, `in_progress`, valida log e presença, INSERT do log, `done`, `FleetLegs.complete!`, evento `transport.trip_done`; `CorrectAttendance` executa `SET LOCAL rota.transport_correction = 'on'` dentro de `transaction(requires_new: true)` e nada mais nessa transação além do UPDATE e do INSERT da correção; correção `no_show` → `boarded` com trecho de frota não `done` chama `Tfd::FleetLegs.complete_passenger!(passenger)`).
- [ ] **Step 4:** mesmos specs + `spec/models/transport_schema_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add the driver screen with attendance, trip closing and audited correction`.

---

### Task 9: TFD — laudo, envio, leitura com trilha e pré-preenchimento

**Files:**
- Create: `config/tfd/primary_care_groups.yml`, `app/services/tfd/{json,trail,rules,primary_care,transition}.rb`, `app/commands/tfd/{save_case,submit_case}.rb`, `app/controllers/tfd/{cases,prefill}_controller.rb`, `spec/requests/tfd/cases_spec.rb`, `spec/services/tfd/rules_spec.rb`
- Modify: `config/routes.rb` (`scope "/tfd", module: :tfd, as: :tfd`), `config/initializers/domain_events.rb` + bindings spec (`tfd.case_created`, `tfd.case_submitted`, `to: []`, payload `tfd_case_id`, `user_id`), `spec/support/transport_helpers.rb` (`tfd_case!` passa pelo `Tfd::SaveCase` + `Tfd::Transition` para `submitted`/`authorized`)

**Interfaces:**
- Produces: `Tfd::SaveCase.call(attrs:, by:, tfd_case: nil) -> Result.ok(tfd_case:)` (Desvio 10; só `draft`/`returned`, só o autor — `not_author` 403; corpo do contrato §6); `Tfd::Rules.problems(tfd_case, settings:, stage: :submit | :authorize) -> Array<{ path:, error: }>` (Desvio 11); `Tfd::PrimaryCare.covers?(sigtap_code) -> Boolean`; `Tfd::Transition.call(tfd_case:, to:, by:, reason: nil, kind: nil)` (Desvio 12; trava, confere a transição, grava evento, `version += 1`; recusa `invalid_transition`); `Tfd::SubmitCase.call(tfd_case:, by:)`; `Tfd::Json.case(tfd_case)` (forma `<tfd-case>`; `sigtap_name` da SIGTAP ativa; `events[].reason` só para a comissão e o autor), `Tfd::Json.list_item(tfd_case)` (`{ id, patient: { display_name }, destination: { name }, status, submitted_at }`); `Tfd::Trail.person_viewed!(user:, patient_id:)`; rotas `POST /tfd/cases`, `PATCH /tfd/cases/:id`, `POST /tfd/cases/:id/submit`, `GET /tfd/cases?status=&mine=` (comissão: todas exceto `draft`; `mine=true`: as do autor, qualquer situação), `POST /tfd/cases/:id/read` (comissão ou autor; trilha; outro → 404), `GET /tfd/prefill?consultation_id=` (autor da consulta; `{ patient_id, cid10, sigtap_code }`: `cid10` = primeiro `consultation_problems` `cid10` da consulta, `sigtap_code` = `target_code` do pedido de regulação `procedure` da consulta, senão nulos).

- [ ] **Step 1: Write the failing tests** (trechos de `cases_spec.rb`):

```ruby
RSpec.describe "TFD — laudo", type: :request do
  let!(:city) { transport_city!(tfd: true, billing: true) }
  let(:unit) { create_unit("UBS Laudo") }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { started_consultation!(unit: unit, doctor: doctor, citizen: regulation_patient!(exam_seq!).last) }
  let(:far) { destination!(name: "Londrina", ibge: "4113700", km: 380) }
  let(:report) { { sigtap_code: "0301010072", cid10: "N18.5", justification: "Hemodiálise indisponível no município, sem serviço de nefrologia", companion_indicated: true, companion_reason: "Paciente idoso", transport_mode: "fleet" } }

  before { sign_in_as(doctor) }

  it "médico cria rascunho a partir da consulta e envia; a lista da comissão não leva CID" do
    json_post "/tfd/cases", patient_id: consultation.patient_id, consultation_id: consultation.id, report: report, destination_id: far.id
    expect(json_body).to include("status" => "draft", "requested_by" => { "name" => doctor.professional_name, "cbo" => "225125" })
    id = json_body["id"]
    json_post "/tfd/cases/#{id}/submit"
    expect(json_body["status"]).to eq("submitted")
    sign_in_as(committee!)
    get "/tfd/cases", params: { status: "submitted" }
    expect(json_body["items"][0].keys).to contain_exactly("id", "patient", "destination", "status", "submitted_at")
    expect(response.body).not_to include("N185", "Hemodiálise")
  end

  it "recusas do envio todas juntas: abaixo de 50 km, metropolitano, atenção básica, sem CID, justificativa curta" do
    near = destination!(name: "Araucária", ibge: "4101804", km: 27, metropolitan: true)
    json_post "/tfd/cases", patient_id: consultation.patient_id, consultation_id: consultation.id, destination_id: near.id,
                            report: report.merge(sigtap_code: "0301010064", cid10: nil, justification: "curta")
    json_post "/tfd/cases/#{json_body['id']}/submit"
    expect(json_body["problems"].map { |p| p["error"] }).to contain_exactly(
      "distance_below_minimum", "metropolitan_destination", "primary_care_procedure", "cid_required", "justification_required"
    )
  end

  it "enfermeiro não pede; leitura nominal só da comissão e do autor, com trilha" do
    sign_in_as(screener!(unit, cbo: "223505"))
    json_post "/tfd/cases", patient_id: consultation.patient_id, consultation_id: consultation.id, report: report, destination_id: far.id
    expect(json_body["error"]).to eq("cbo_not_allowed")
    tfd = tfd_case!(status: "submitted")
    sign_in_as(committee!)
    expect { json_post "/tfd/cases/#{tfd.id}/read" }
      .to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(1)
    expect(json_body["report"]).to include("cid10" => "N185", "justification" => a_string_including("nefrologia"))
    sign_in_as(dispatcher!)
    json_post "/tfd/cases/#{tfd.id}/read"
    expect(response).to have_http_status(:forbidden)
  end

  it "pré-preenche pela consulta do autor; consulta de outro → 404" do
    add_problem!(consultation, terminology: "cid10", code: "N185") # helper do 19a
    get "/tfd/prefill", params: { consultation_id: consultation.id }
    expect(json_body).to eq("patient_id" => consultation.patient_id, "cid10" => "N185", "sigtap_code" => nil)
  end

  it "tfd desligado → 403 feature_disabled tfd mesmo com transporte ligado" do
    Platform::Features.set!(city: city, key: "tfd", enabled: false, maintainer: ledi_maintainer!)
    get "/tfd/cases", params: { mine: "true" }
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "tfd")
  end
end
```

`rules_spec.rb`: 49 km com limite 50 → `distance_below_minimum`; 50 km → nada; limite da cidade 60 → 55 km recusado; `0101010010` (grupo 01) e `0301010064` → `primary_care_procedure`; `0301010072` → nada; código fora da SIGTAP ativa → `invalid_sigtap_code`; `stage: :authorize` sem `appointment_at` → `appointment_required`.

(Os nomes `started_consultation!`, `doctor!`, `regulation_patient!`, `add_problem!` são helpers existentes; confira `add_problem!` em `spec/support/clinical_record_helpers.rb` e use o equivalente real.)

- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/tfd/cases_spec.rb spec/services/tfd/rules_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** CID pelo `Billing::Record.send(:cid10, ...)` não (privado): extraia o normalizador para `Tfd::Rules.cid10(value) -> String | nil | :invalid` com a mesma regra (CID-10 ativa); `invalid_cid10` na forma. `report` gravado normalizado (CID sem ponto, maiúsculas). PATCH substitui por inteiro (como o `SaveDraft` do 26a).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add the tfd medical report with distance and primary care refusals`.

---

### Task 10: TFD — comissão: decidir, marcar atendimento, internação e cancelamento

**Files:**
- Create: `app/commands/tfd/{decide_case,cancel_case,set_appointment,record_inpatient}.rb`, `spec/requests/tfd/committee_spec.rb`
- Modify: `app/controllers/tfd/cases_controller.rb`, `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`tfd.case_decided`, `tfd.case_cancelled`, `tfd.case_updated`; payload `tfd_case_id`, `user_id`, `decision` só no `case_decided`)

**Interfaces:**
- Consumes: Task 9 (`Transition`, `Rules`).
- Produces: `POST /tfd/cases/:id/decide` `{ decision, reason }` (step-up; `reason` 10..500 sempre; `authorize` roda `Rules.problems(stage: :authorize)`); `POST /tfd/cases/:id/appointment` `{ appointment_at, provider_id? }` (step-up; `submitted`/`authorized`/`scheduled`; data de agora − 1 dia a agora + 365 dias; prestador ativo); `POST /tfd/cases/:id/inpatient` `{ from, to? }` (step-up; `scheduled`/`travelling`; regras: `from` abre uma internação, `to` com o mesmo `from` fecha a aberta; `inpatient_open` se já há uma aberta e o `from` é outro; `inpatient_overlap`; `to < from` → `invalid_request`); `POST /tfd/cases/:id/cancel` `{ reason }` (comissão com step-up, de qualquer situação não terminal; autor só no próprio `draft`). Eventos com `kind` do `Tfd::Catalog::EVENT_KINDS`.

- [ ] **Step 1: Write the failing test** `committee_spec.rb`: autorizar sem `appointment_at` → `appointment_required`; com `procedure_billing` ligado e sem `tfd_billing_health_unit_id` → `billing_unit_missing` (Divergência D9); marcar atendimento e autorizar → `authorized` com evento `authorized` e o motivo cifrado no banco (`TfdCaseEvent.last.reason_before_type_cast` sem o texto); devolver → `returned`, autor edita e reenvia → `submitted`; recusar → `denied` e nada mais aceita (`invalid_transition`); sem step-up → 401; médico autor tentando decidir → 403 `missing_role`; internação aberta e fechada aparece em `inpatient` do `<tfd-case>`; segunda aberta com outro `from` → `inpatient_open`.
- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/tfd/committee_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmo spec + `spec/requests/tfd/cases_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: let the tfd committee authorize, return, deny and schedule cases`.

---

### Task 11: TFD — trechos e ajuda de custo

**Files:**
- Create: `config/tfd/sigtap_codes.yml` (se a Task 2 só criou o leitor, o arquivo nasce lá; aqui os testes do cálculo), `app/services/tfd/allowances.rb`, `app/commands/tfd/{add_leg,complete_leg}.rb`, `app/controllers/tfd/legs_controller.rb`, `spec/services/tfd/allowances_spec.rb`, `spec/requests/tfd/legs_spec.rb`
- Modify: `app/services/tfd/fleet_legs.rb` (corpo real), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`tfd.leg_added` `to: []`; `tfd.leg_done` e `tfd.allowances_calculated` ficam `to: []` aqui e ganham o consumidor na Task 13)

**Interfaces:**
- Consumes: Tasks 5, 8, 10.
- Produces: `Tfd::AddLeg.call(tfd_case:, attrs:, by:) -> Result.ok(leg:)` (Desvio 13; `authorized`/`scheduled`/`travelling`; primeiro trecho move `authorized → scheduled`); `Tfd::CompleteLeg.call(leg:, by:, at: Time.current) -> Result.ok(leg:, allowances:)` (marca `done_at`, chama `Allowances.for_leg!`, na volta também `Allowances.meals!`, primeiro `outbound` move `scheduled → travelling`; publica `tfd.leg_done` (`tfd_leg_id`) e, se houve refeição, `tfd.allowances_calculated` (`tfd_case_id`, `tfd_leg_id`)); `Tfd::FleetLegs.complete!(trip) -> Array<TfdLeg>` (trechos `fleet` não `done` dos passageiros `boarded` da viagem) e `.complete_passenger!(passenger)`; `Tfd::Allowances.travellers(leg) -> Array<"patient" | "companion">`, `.units(leg) -> Integer` (Desvio 14), `.for_leg!(leg) -> Array<TfdAllowance>`, `.meals!(return_leg) -> Array<TfdAllowance>`, `.value(code:, on:, quantity:, settings:) -> { table_cents:, complement_cents:, total_cents:, competence: }`; `POST /tfd/cases/:id/legs`, `POST /tfd/legs/:id/done` (só passagem; frota → `fleet_leg_by_trip`), `GET /tfd/cases/:id/allowances` → `{ items: [<tfd-allowance>], total_cents }`.

- [ ] **Step 1: Write the failing test** `allowances_spec.rb` (trechos):

```ruby
RSpec.describe Tfd::Allowances do
  let!(:city) { transport_city!(tfd: true, billing: true) }
  let(:tfd) { tfd_case!(status: "authorized", destination: destination!(name: "Londrina", ibge: "4113700", km: 380)) }
  def leg(direction, on, mode: "bus_ticket", companion: false)
    Tfd::AddLeg.call(tfd_case: tfd.reload, by: committee!, attrs: { "direction" => direction, "mode" => mode, "date" => on.iso8601,
                                                                     "ticket" => { "amount_cents" => 15_000, "carrier" => "Viação Garcia" }, "companion" => companion }).payload[:leg]
  end

  it "380 km por terra = 8 unidades de 50 km, por pessoa e por trecho, com o valor da tabela e o complemento" do
    Transport::SaveSettings.call(attrs: { "allowance_complements" => { "0803010125" => 105 } }, by: transport_admin!)
    out = leg("outbound", Date.current, companion: true)
    rows = Tfd::CompleteLeg.call(leg: out, by: committee!).payload[:allowances]
    expect(rows.map { |r| [ r.person, r.kind, r.sigtap_code, r.quantity ] })
      .to contain_exactly([ "patient", "ground_km", "0803010125", 8 ], [ "companion", "ground_km", "0803010109", 8 ])
    patient_row = rows.find { |r| r.person == "patient" }
    expect([ patient_row.table_cents, patient_row.complement_cents, patient_row.total_cents ]).to eq([ 495, 105, 4800 ])
  end

  it "ida e volta no mesmo dia: meal_day; três noites fora: meal_overnight × 3" do
    out = leg("outbound", Date.current)
    Tfd::CompleteLeg.call(leg: out, by: committee!)
    back = leg("return", Date.current)
    expect(Tfd::CompleteLeg.call(leg: back, by: committee!).payload[:allowances].map(&:kind)).to include("meal_day")
    out2 = leg("outbound", Date.current + 10)
    Tfd::CompleteLeg.call(leg: out2, by: committee!)
    back2 = leg("return", Date.current + 13)
    meal = Tfd::CompleteLeg.call(leg: back2, by: committee!).payload[:allowances].find { |r| r.kind == "meal_overnight" }
    expect([ meal.person, meal.quantity ]).to eq([ "patient", 3 ])
  end

  it "Review Focus 3: internado a estadia inteira — paciente sem diária, acompanhante com as noites" do
    out = leg("outbound", Date.current + 10, companion: true)
    Tfd::CompleteLeg.call(leg: out, by: committee!)
    Tfd::RecordInpatient.call(tfd_case: tfd.reload, from: (Date.current + 10).iso8601, to: (Date.current + 15).iso8601, by: committee!)
    back = leg("return", Date.current + 15, companion: true)
    meals = Tfd::CompleteLeg.call(leg: back, by: committee!).payload[:allowances].select { |r| r.kind.start_with?("meal") }
    expect(meals.map { |r| [ r.person, r.kind, r.quantity ] }).to eq([ [ "companion", "meal_overnight", 5 ] ])
  end

  it "aéreo em milhas (200) e fluvial em milhas náuticas (27), mínimo 1" do
    expect(described_class.units(TfdLeg.new(mode: "air", distance_km: 380))).to eq(2)    # 236 mi
    expect(described_class.units(TfdLeg.new(mode: "river", distance_km: 30))).to eq(1)   # 16,2 nm
    expect(described_class.units(TfdLeg.new(mode: "bus_ticket", distance_km: 51))).to eq(2)
  end

  it "volta sem ida aberta → no_outbound_leg; completar de novo não duplica linha" do
    result = Tfd::AddLeg.call(tfd_case: tfd, by: committee!, attrs: { "direction" => "return", "mode" => "bus_ticket", "date" => Date.current.iso8601,
                                                                       "ticket" => { "amount_cents" => 100, "carrier" => "X" } })
    expect(result.reason).to eq(:no_outbound_leg)
    out = leg("outbound", Date.current)
    Tfd::CompleteLeg.call(leg: out, by: committee!)
    expect(Tfd::CompleteLeg.call(leg: out.reload, by: committee!).reason).to eq(:already_done)
  end
end
```

`legs_spec.rb`: trecho de frota com `trip_passenger_id` de passageiro de outro paciente ou sem `tfd_case_id` deste processo → `invalid_request` `path: "trip_passenger_id"`; o fechamento da viagem (Task 8) com o paciente `boarded` marca o trecho `done` e cria `ground_km`; `POST /tfd/legs/:id/done` num trecho de frota → `fleet_leg_by_trip`; `GET …/allowances` soma `total_cents`.

- [ ] **Step 2:** `bin/test api -w mod30 spec/services/tfd/allowances_spec.rb spec/requests/tfd/legs_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`units`: `ceil` sobre `BigDecimal`; noites internadas = interseção de cada `[from, to || volta.date)` com `[ida.date, volta.date)` em dias; `INSERT ... ON CONFLICT (tfd_leg_id, person, kind) DO NOTHING` via `insert_all` + releitura, para a corrida do fechamento com o botão).
- [ ] **Step 4:** mesmos specs + `spec/requests/transport/my_trips_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: calculate tfd travel legs and allowances by the federal rule`.

---

### Task 12: TFD — pagamento registrado e prestação de contas

**Files:**
- Create: `app/services/tfd/files.rb`, `app/commands/tfd/{record_payment,close_accountability}.rb`, `app/controllers/tfd/files_controller.rb`, `spec/requests/tfd/payments_spec.rb`
- Modify: `app/controllers/tfd/cases_controller.rb`, `app/services/exams/files.rb` (o `purpose` novo só como valor aceito), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`tfd.payment_recorded`, `tfd.accountability_closed`, `to: []`)

**Interfaces:**
- Produces: `POST /tfd/cases/:id/payments` `{ paid_on, amount_cents, method, receipt_number }` (step-up; só em `scheduled`/`travelling`, senão `invalid_transition`; `paid_on` ≤ hoje e ≥ hoje − 365) → 201 `<tfd-payment>`; `POST /tfd/cases/:id/accountability` `{ filename, content_type, data }` (base64; PDF/JPEG/PNG até 10 MB pelos bytes mágicos do `Exams::Files`; step-up; só `travelling` com todos os trechos `done`, senão `legs_pending`) → `completed` e `<tfd-case>`; `POST /tfd/cases/:id/accountability/url` → `{ url: "/tfd/files/<token>" }` (5 min, amarrado à cidade, ao arquivo e ao usuário); `GET /tfd/files/:token` (comissão; 410 `link_expired`); `Tfd::Files.store(...)`, `.url_for(file, user:)`, `.resolve(token, user:)` com `PURPOSE = "tfd_accountability"`.

- [ ] **Step 1: Write the failing test** `payments_spec.rb`: pagamento grava e aparece em `payments` com `by_name`; `method: "boleto"` → `problems[{ path: "method" }]`; o número do recibo não vai ao log (`capture_log`); prestação de contas com PNG falso (bytes de texto) → `invalid_file`; com trechos pendentes → `legs_pending`; com tudo feito → `completed` e `accountability.document.filename`; URL do comprovante baixa os bytes; link de outro usuário → 404; depois de 5 min → 410.
- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/tfd/payments_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`Exams::Files.store(..., purpose: Tfd::Catalog::FILE_PURPOSE)` e `consumed_at` gravado na mesma transação; `Tfd::Files` com verifier e purpose próprios).
- [ ] **Step 4:** mesmo spec + `spec/requests/exams` (o upload do 21a segue igual) → PASS.
- [ ] **Step 5: Commit** `feat: record tfd payments and close the accountability with the attendance proof`.

---

### Task 13: Produção do TFD no BPA-I

**Files:**
- Create: `app/commands/billing/from_tfd.rb`, `app/jobs/billing/tfd_production_job.rb`, `spec/commands/billing/from_tfd_spec.rb`
- Modify: `app/commands/billing/record.rb` (`insert!` ganha `tfd_allowance: nil, tfd_leg: nil`), `app/models/performed_procedure.rb` (`belongs_to :tfd_allowance, optional: true`, `:tfd_leg`), `app/services/billing/labels.rb` (fonte `tfd` → "TFD"; confira o nome real do rótulo de fonte), `config/initializers/domain_events.rb` (`tfd.leg_done` e `tfd.allowances_calculated` → `[ Billing::TfdProductionJob ]`), `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- Consumes: Task 11 (`TfdAllowance`), `Transport::Settings.current`, `Billing::Gate`, `Billing::Establishment.for_unit`.
- Produces: `Billing::FromTfd.call(allowances) -> Integer` (Desvio 15; idempotente por `PerformedProcedure.exists?(tfd_allowance_id:)` + índice único); `Billing::TfdProductionJob#handle(tfd_leg_id:, **)` (IdempotentConsumer; `return unless Gate.enabled?(Current.city)`; lê `TfdAllowance.where(tfd_leg_id:)` — os dois eventos levam `tfd_leg_id`, o job é o mesmo).

- [ ] **Step 1: Write the failing test** `from_tfd_spec.rb` (trechos):

```ruby
RSpec.describe Billing::FromTfd do
  let!(:city) { transport_city!(tfd: true, billing: true) }
  let(:unit) { health_management_unit! }
  before { Transport::SaveSettings.call(attrs: { "tfd_billing_health_unit_id" => unit.id }, by: transport_admin!) }

  it "uma linha BPA-I por ajuda, com o médico do laudo, a unidade de gestão e a data do trecho; sem establishment_is_aps" do
    tfd = tfd_case!(status: "authorized")
    out = tfd_leg_done!(tfd, "outbound", Date.current - 1)
    rows = PerformedProcedure.where(source: "tfd")
    expect(rows.pluck(:sigtap_code, :quantity, :performed_on, :health_unit_id, :professional_user_id, :cbo_code))
      .to eq([ [ "0803010125", 8, Date.current - 1, unit.id, tfd.requester_user_id, tfd.requester_cbo ] ]) # 380 km do destino padrão do tfd_case!
    expect(rows.first).to have_attributes(instrument: "bpa_i", tfd_leg_id: out.id)
    expect(rows.first.blocking_reasons).not_to include("establishment_is_aps", "missing_professional_cbo")
  end

  it "Review Focus 4: acompanhante faltou na frota — só a linha do paciente, e nenhuma refeição de acompanhante na volta" do
    patient = transport_patient!(birth_date: "1950-01-01")
    tfd = tfd_case!(status: "authorized", patient: patient, report: { "companion_indicated" => true })
    trip = trip!(date: Date.current, vehicle: vehicle!, driver: driver_user!.transport_driver, status: "confirmed", destination: tfd.transport_destination)
    rider = passenger!(trip, patient: patient, origin: "tfd", tfd_case: tfd, exception_reason: nil, companion_name: "Filha", companion_attendance: "pending")
    Tfd::AddLeg.call(tfd_case: tfd, by: committee!, attrs: { "direction" => "outbound", "mode" => "fleet", "date" => Date.current.iso8601, "trip_passenger_id" => rider.id })
    close_trip!(trip, rider => { attendance: "boarded", companion_attendance: "no_show" }) # Task 8 pelo caminho real
    perform_enqueued_jobs
    expect(PerformedProcedure.where(source: "tfd").pluck(:sigtap_code)).to eq([ "0803010125" ])
  end

  it "billing desligado no momento do trecho: nada (sem retroativo); rodar o job de novo não duplica" do
    Platform::Features.set!(city: city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
    tfd_leg_done!(tfd_case!(status: "authorized"), "outbound", Date.current)
    expect(PerformedProcedure.where(source: "tfd")).to be_empty
  end
end
```

(`tfd_leg_done!(tfd, direction, on)` (trecho de passagem sem acompanhante, `AddLeg` + `CompleteLeg` + `perform_enqueued_jobs`) e `close_trip!(trip, presence)` (start, attendance e close pelos comandos da Task 8) entram em `transport_helpers.rb`.)

- [ ] **Step 2:** `bin/test api -w mod30 spec/commands/billing/from_tfd_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`insert!` repassa `tfd_allowance:`/`tfd_leg:` ao `create!`; nenhuma mudança na crítica).
- [ ] **Step 4:** mesmo spec + `spec/commands/billing spec/requests/billing spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: send tfd travel and allowance production to bpa-i through billing`.

---

### Task 14: Avisos ao cidadão e SMS

**Files:**
- Create: `app/services/transport/sms_text.rb`, `app/commands/transport/notify.rb`, `app/jobs/transport/{notice_job,remind_job}.rb`, `spec/commands/transport/notify_spec.rb`, `spec/jobs/transport/remind_job_spec.rb`, `spec/requests/citizen_api/trip_notices_spec.rb`
- Modify: `app/controllers/citizen_api/notices_controller.rb` (terceira fonte), `config/initializers/domain_events.rb` (`transport.trip_confirmed`, `transport.trip_cancelled`, `transport.passenger_added`, `transport.passenger_removed`, `transport.trip_changed` → `[ Transport::NoticeJob ]`), bindings spec, `config/recurring.yml` (`transport_remind`: `Transport::RemindJob`, `housekeeping`, `"every 15 minutes"`)

**Interfaces:**
- Produces: `Transport::Notify.call(passenger:, kind:, event_key: "once", now: Time.current) -> Integer` (avisos criados; um por par validado do paciente — `Patients::Pairs.of(patient)`; trava o cidadão em `FOR KEY SHARE` antes da viagem, a ordem do `Appointments::Remind`; `INSERT ... ON CONFLICT DO NOTHING` no índice único; SMS pelas regras do 17 só quando o aviso nasceu — `sms` gravado; falha do provedor só com a classe no log); `Transport::SmsText.body(kind:, passenger:) -> String` — textos fixos (Divergência D14), prefixo `"Secretaria de Saúde de %{city}: "`:
  - `added`/`departure_changed`/`boarding_changed`/`reminder`: `"sua viagem de %{date} sai às %{time} de %{boarding}."`
  - `cancelled`/`removed`: `"sua viagem de %{date} foi cancelada. Veja em %{link}"`
  (`date` `dd/mm`, `time` `HH:MM` local, `boarding` = rótulo da parada truncado em 40; nunca destino nem prestador);
  `Transport::NoticeJob#handle(trip_id:, passenger_id: nil, change: nil, **)` (só viagem `confirmed`/`in_progress`, salvo `cancelled`; `trip_confirmed` → `added` para todos; `trip_changed` `departure` → `departure_changed` com `event_key` = `departure_at.iso8601`; `boarding` → `boarding_changed` para os passageiros da parada com `event_key` = `Digest::SHA256.hexdigest(label)[0, 16]`); `Transport::RemindJob` (EachCityJob; só das 17h às 20h; viagens `confirmed` de amanhã; `reminder` uma vez); `/citizen/notices` com `{ kind: "trip", id, notice:, trip_date, departure_at, boarding_label, destination_name, read, cpf_masked }` e `POST /citizen/notices/:id/read` procurando também em `transport_trip_notices`; `unread_count` soma os não lidos de quem não silenciou.

- [ ] **Step 1: Write the failing tests:** `notify_spec.rb` — confirmar a viagem cria um aviso `added` por par validado, e confirmar de novo (evento repetido) não duplica; com SMS ligado, opt-in e dentro da janela → `SmsGateway.deliver` com o texto exato do `added` (`"Secretaria de Saúde de Curitiba: sua viagem de 05/11 sai às 07:00 de Praça Central."` — data e hora do cenário) e sem o nome do destino; sem opt-in ou às 21h → aviso sem SMS; `appointment_reminders_muted` → sem SMS; viagem `planned` → nenhum aviso; mudança de 30 min → `departure_changed` uma vez por horário novo. `remind_job_spec.rb` — 18h da véspera → `reminder`; 16h → nada; rodar duas vezes → um. `trip_notices_spec.rb` — a caixa do wpda traz o aviso `trip` sem prestador, CID nem placa, marca como lido e o `unread_count` cai.
- [ ] **Step 2:** `bin/test api -w mod30 spec/commands/transport/notify_spec.rb spec/jobs/transport/remind_job_spec.rb spec/requests/citizen_api/trip_notices_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs + `spec/requests/citizen_api/notices_spec.rb` (o existente) → PASS.
- [ ] **Step 5: Commit** `feat: notify citizens of trip changes in the inbox and by sms`.

---

### Task 15: "Minhas viagens" e linha do tempo

**Files:**
- Create: `app/services/transport/citizen_view.rb`, `app/controllers/citizen_api/trips_controller.rb`, `app/services/timeline/sources/transport_source.rb`, `spec/requests/citizen_api/trips_spec.rb`, `spec/services/timeline/sources/transport_source_spec.rb`
- Modify: `app/services/timeline.rb` (`KINDS` + `transport` no fim), `app/services/timeline/query.rb` (`SOURCES`), `spec/services/timeline/query_spec.rb:75` (`KINDS.last(3) == %w[dispensation vaccination transport]`), `config/routes.rb` (`get "trips", to: "trips#index"` no escopo `/citizen`)

**Interfaces:**
- Produces: `GET /citizen/trips?citizen_id=` (Desvio 16) → forma do contrato §8 (`items`: passageiros ativos do paciente em viagens de hoje − 90 dias em diante, mais novas primeiro, até 50; `status` = da viagem; `companion` = há acompanhante; `tfd`: processos não `draft` do paciente, `allowance_total_cents` = soma das ajudas ou nulo, `paid` = há pagamento; vazio com `tfd` não utilizável); `Transport::CitizenView.call(patient_id:, tfd: Boolean) -> Hash`; `Timeline::Sources::TransportSource.entries(person:, before:, limit:, from:, to:)` → `Entry(kind: "transport", source_id: passageiro.id, occurred_at: trip.departure_at, unit: nil, professional: nil, title: "Viagem para <destino>" | "TFD para <destino>" (origem `tfd`), summary: nil, status: attendance, detail_ref: { type: "trip_passenger", id: }, citizen: { title:, summary: nil, status: nil, professional_name: nil, unit: nil }, document: nil)` — passageiros retirados e viagens canceladas fora; trechos de passagem do TFD `done` entram com `source_id` = trecho, `occurred_at` = meio-dia local da data, `detail_ref: { type: "tfd_leg", id: }`, título "TFD para <destino>"; não consulta o interruptor (registro retido).

- [ ] **Step 1: Write the failing tests:** cidadão validado vê a viagem com embarque, destino (nome e prestador) e situação, sem placa, motorista, procedimento ou CID; o TFD aparece com `allowance_total_cents` e `paid: false`; par não validado → 403 `not_verified`; `citizen_id` de outra sessão → 404; transporte desligado → 403 `feature_disabled`; trilha `access: "citizen"` uma por requisição; a linha do tempo do profissional traz `kind: "transport"`, título "TFD para Londrina" no passageiro de origem `tfd` e "Viagem para Ponta Grossa" nos outros; `Timeline::Query.citizen_kinds(verified: true, documents: true)` inclui `transport`.
- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/citizen_api/trips_spec.rb spec/services/timeline/sources/transport_source_spec.rb spec/services/timeline/query_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs + `spec/services/timeline spec/requests/citizen_api/history_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: show trips to the citizen and add the transport timeline source`.

---

### Task 16: Painel de transporte e TFD

**Files:**
- Create: `app/services/transport/dashboard.rb`, `app/controllers/transport/dashboard_controller.rb`, `spec/requests/transport/dashboard_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Produces: `GET /transport/dashboard?month=YYYY-MM` (`municipal_admin`, `transport_dispatcher`, `tfd_committee`; sem trilha) → `{ month, trips, occupancy_pct, no_shows, km, fuel_cents, tfd_by_status, tfd_paid_cents, exceptions }` — `trips` = viagens `done` com `date` no mês; `occupancy_pct` = `round(100 * Σ seats_used / Σ seats)` das `done` com veículo (nulo sem nenhuma); `no_shows` = passageiros `no_show` dessas viagens; `km` = Σ (`odometer_end − odometer_start`); `fuel_cents` = Σ `fuel[].amount_cents`; `exceptions` = passageiros `origin: "exception"` dessas viagens; `tfd_by_status` = processos com `created_at` no mês por situação atual (`{}` sem `tfd` utilizável); `tfd_paid_cents` = Σ pagamentos com `paid_on` no mês. Contagens 1..4 (`Admin::SmallCount.small?`) saem como `"lt5"`; 0 fica 0; somas nunca são suprimidas.

- [ ] **Step 1: Write the failing test:** cenário com 2 viagens `done` (15 lugares, 10 e 5 ocupados; 240 e 100 km; R$ 250 de combustível) → `trips: "lt5"`, `occupancy_pct: 50`, `km: 340`, `fuel_cents: 25_000`; 6 faltas → `no_shows: 6`; motorista → 403; `month` inválido → 422 `path: "month"`; nenhum nome, CPF ou destino de tratamento no JSON.
- [ ] **Step 2:** `bin/test api -w mod30 spec/requests/transport/dashboard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmo spec → PASS.
- [ ] **Step 5: Commit** `feat: add the transport and tfd panel with small-count suppression`.

---

### Task 17: Exclusão LGPD

**Files:**
- Create: `app/services/transport/anonymize.rb`, `spec/commands/citizens/erase_transport_spec.rb`
- Modify: `app/commands/citizens/request_erasure.rb` (`tfd_cased?` em `examined_patients?`), `app/commands/citizens/erase.rb` (`TransportTripNotice.where(citizen_id: citizen.id).delete_all` em `erase_pair`; `Transport::Anonymize.patients!(locked_patients.map(&:id))` no ramo não retido, depois das sugestões), `spec/commands/citizens/request_erasure_spec.rb`

**Interfaces:**
- Produces: `Citizens::RequestErasure.tfd_cased?(ids) -> Boolean`; `Transport::Anonymize.patients!(patient_ids, now: Time.current) -> Integer` (Desvio 17; um `update_all` com `anonymized_at: now` e as colunas cifradas a NULL onde `anonymized_at IS NULL`; séries ativas recebem `suspended_from = hoje` e `suspend_reason = "Exclusão do cadastro (LGPD)"`, e saem das viagens futuras pelo `SuspendSeries`).

- [ ] **Step 1: Write the failing tests:** pedido de exclusão de quem tem TFD (rascunho inclusive) → `retained`; de quem só viajou → `confirmed`, passageiro com `companion_name`, `companion_document`, `provider_name` e `address` nulos, `anonymized_at` preenchido, `attendance` mantida e contada no painel; avisos de viagem dos pares apagados; série suspensa e fora das viagens futuras; o trigger aceita a anonimização uma vez e recusa uma segunda UPDATE das cifradas.
- [ ] **Step 2:** `bin/test api -w mod30 spec/commands/citizens/erase_transport_spec.rb spec/commands/citizens/request_erasure_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs + `spec/commands/citizens` → PASS.
- [ ] **Step 5: Commit** `feat: retain tfd cases and anonymize trip passengers on erasure`.

---

### Task 18: Semente de dev, guarda de eventos e suíte completa

**Files:**
- Create: `lib/transport_crew.rb`, `spec/lib/transport_crew_spec.rb`
- Modify: `db/seeds.rb` (depois do `VaccinationCrew`, `TransportCrew.seed_current_city(slug:, ddd:, password:)` + `summary_lines`)

**Interfaces:**
- Consumes: tudo acima. Produces: `TransportCrew.seed_current_city(slug:, ddd:, password:)` idempotente (Valores fixados 6, só Curitiba, tudo pelos comandos do app — `SaveVehicle`, `SaveDriver`, `SaveDestination`, `SaveSettings`, `SaveSeries`, `MaterializeSeries`, `SaveTrip`, `AddPassenger`, `SaveCase`, `SubmitCase`, `SetAppointment`, `DecideCase`), usuários com TOTP fixo da semente; `summary_lines` só com e-mail, papel e otpauth.

- [ ] **Step 1:** spec da crew: duas rodadas → mesmas contagens de veículos, motoristas, destinos, séries, viagens e processos; `sanitary_transport` e `tfd` utilizáveis; a série gerou viagens nas próximas 3 semanas; `tfd_billing_health_unit_id` aponta para a unidade `health_management`; nenhuma linha da saída com CPF ou nome de paciente.
- [ ] **Step 2–4:** FAIL → implementar → PASS (`bin/test api -w mod30 spec/lib/transport_crew_spec.rb`).
- [ ] **Step 5:** `bin/test api -w mod30 spec/initializers/domain_events_bindings_spec.rb spec/initializers/filter_parameter_logging_spec.rb spec/events spec/architecture spec/adr_pointers_spec.rb spec/config spec/invariants` → PASS; `rg -n "DomainEvents.publish\(" app/commands/transport app/commands/tfd app/services/transport app/services/tfd app/jobs/transport -A4` conferindo cada chamada inteira (só ids e códigos de lista fechada).
- [ ] **Step 6: Commit** `chore: seed a dev sanitary transport fleet and a tfd case`.
- [ ] **Step 7:** pedir à sessão "Testes" (`SendMessage`): `api · mod30/feat/mod-30-sanitary-transport · --full · módulo 30 (migração de cidade, ck_memberships_role, spec/support, Billing::Record, Citizens::Erase, Timeline)`. Falha → diagnóstico nesta sessão, correção com teste, nova rodada.
- [ ] **Step 8:** F-30.7 → Done (só api). Prova no navegador só com autorização do usuário (migrar e semear o dev); depois disso o plano do dashboard. Gates de go-live a registrar no board #2: códigos e valores do 08.03.01 no SIGTAP oficial (inclusive paciente × acompanhante por modal, CBOs compatíveis e unidade por trecho de frota), manual estadual de TFD da UF piloto (diária do acompanhante de paciente internado, valores do complemento), lista de procedimentos da atenção básica (`primary_care_groups.yml`), CNES da unidade de gestão no BPA, cadastro dos veículos no CNES (Portaria SAES 4.904/2026), km dos destinos.

---

## Divergências propostas ao contrato

- **D1 — Detalhe da linha do tempo.** O 20 usa `detail_ref: { "type", "id" }`, não `{ "kind", "id" }` (o 29 já corrigiu igual, D7). Proposta: `{ "type": "trip_passenger" | "tfd_leg", "id" }`; o trecho de passagem do TFD `done` também entra na linha do tempo (não tem passageiro).
- **D2 — `GET /citizen/trips?citizen_id=`.** As rotas do cidadão levam `citizen_id` do par validado (sessão com vários pares). Proposta: query obrigatória, 404/403 `not_verified` como em "Meus encaminhamentos" e na caderneta.
- **D3 — Comprovante da prestação de contas.** Escrita por cookie é sempre JSON (regra comum) e o armazenamento real é o `Exams::Files` do 21a (não há um do 19c). Proposta: `POST /tfd/cases/:id/accountability` `{ filename, content_type, data (base64) }`, e o download por URL curta: `POST /tfd/cases/:id/accountability/url` → `{ url }` e `GET /tfd/files/:token` (5 min; 410 `link_expired`). `filename` cifrado (pode trazer o nome do paciente).
- **D4 — Atendimentos marcados da agenda.** Sem filtro, `POST /transport/scheduled_care` traria todos os horários do 17 da cidade no dia. Proposta: `health_unit_id?` no corpo — horários do 17 só com ele (a unidade é o destino intramunicipal). E o `schedule_place` (texto livre) do 26a nunca vira `provider_name`; só o nome do `care_provider`.
- **D5 — Série.** O job precisa da hora de volta e da parada para criar a viagem `planned`. Proposta: `return_time` (`"HH:MM"`, > `departure_time`) e `boarding_label` (1..120) obrigatórios no corpo de `/transport/series`.
- **D6 — Trecho do TFD.** `<tfd-leg>` ganha `companion: bool` (passagem com acompanhante; na frota vem da presença) e `outbound_leg_id|null` (a volta se pareia sozinha com a ida mais antiga sem volta; sem ida → `no_outbound_leg`). `POST /tfd/legs/:id/done` num trecho de frota → `fleet_leg_by_trip`.
- **D7 — Linha do BPA sempre por ajuda.** O deslocamento também vira linha de `tfd_allowances` (kinds `ground_km`/`air_miles`/`river_miles`, já no §1), criada quando o trecho fica `done`; o `performed_procedure` sempre referencia `tfd_allowance_id` (único) e leva `tfd_leg_id` só como rastro. Os dois eventos (`tfd.leg_done`, `tfd.allowances_calculated`) levam `tfd_leg_id` e vão ao mesmo job.
- **D8 — Cancelamento e lista do autor.** A situação `cancelled` não tinha rota: `POST /tfd/cases/:id/cancel` `{ reason }` (comissão com step-up; autor só no próprio rascunho). O médico precisa achar os próprios rascunhos e devolvidos: `GET /tfd/cases?mine=true` (mesma forma da lista).
- **D9 — Autorização sem unidade do BPA.** Com `procedure_billing` utilizável e sem `tfd_billing_health_unit_id`, `authorize` recusa `billing_unit_missing` (senão a produção se perde em silêncio). Desligado o BPA, autoriza.
- **D10 — Tabelas além do ADR.** `transport_settings` (configuração), `transport_pending_items` (pendência da série), `transport_attendance_corrections` (correção da presença com motivo, imutável). A internação fica em `tfd_cases.inpatient_stays` (jsonb), sem tabela. `transport_trip_passengers.transport_series_id` sai como `series_id` no JSON.
- **D11 — Quando o cidadão é avisado.** Só viagem `confirmed`/`in_progress` gera aviso (a `planned` muda demais); o `confirm` avisa todos (`added`); "mudança da parada de embarque" = mudança do rótulo da parada do passageiro (não há troca de parada sem retirar e incluir). Lembrete da véspera das 17h às 20h, como o do 17.
- **D12 — Contagem pequena.** "abaixo de 5" = 1 a 4 (`Admin::SmallCount`); 0 continua 0; somas (km, combustível, valores pagos) não são suprimidas.
- **D13 — Acompanhante.** Faixa da CIT 13/2017 lida como `idade <= 15` ou `idade >= 60` (a chave é `min_age_elderly`). No TFD, o desconto da internação vale só para o paciente; o acompanhante recebe as noites (gate: manual da UF).
- **D14 — Textos de SMS.** O §8 tem um texto só. Proposta: o do §8 para `added`, `departure_changed`, `boarding_changed` e `reminder`; "sua viagem de <data> foi cancelada. Veja em <link>" para `cancelled` e `removed`; sempre com o prefixo "Secretaria de Saúde de <cidade>: " do 17.
- **D15 — Origem do laudo.** O laudo exige `consultation_id` **ou** `regulation_request_id` (ao menos um; decisão 15 da spec), do mesmo paciente; o CBO `225*` vem da consulta ou do vínculo ativo do médico e é gravado no processo (`requested_by.cbo`), porque o BPA usa esse CBO (o médico não tem vínculo com a unidade de gestão).
- **D16 — Crítica do 22.** Nada muda no `Billing::Critic`: `establishment_is_aps` já só marca `kind == "ubs"`, e a unidade `health_management` passa sem regra nova. O §7 pode trocar "sem a regra" por "a regra já não se aplica".
- **D17 — Recusas extras.** `vehicle_required`, `driver_required`, `seats_full`/`wheelchair_full` (troca de veículo), `stop_in_use`, `trip_not_open`, `not_today`, `tfd_leg_done` (correção), `legs_pending` (contas), `inpatient_open`/`inpatient_overlap`, `user_not_driver`, `plate_taken`/`renavam_taken`/`ibge_taken`/`user_already_driver` (409). `GET /transport/trips` exige `date`.
