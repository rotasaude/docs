# Módulo 24b — Visita domiciliar e atividade coletiva (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: api em `origin/main` = `d36aef34`** (conferido ao escrever este plano; é a base do contrato). Última migração de cidade `20261111900002_add_ombudsman.rb` (`db/city_schema.rb:13` em `define(version: 2026_11_11_900002)`); as migrações 24a/33 levantam `IrreversibleMigration` no `down` e executam `db/city_triggers.sql` no fim do `up` (`20261023900001_add_territory_registry.rb:165-169`, `20261111900002_add_ombudsman.rb:68-72`). `Platform::Features::CATALOG` termina em `ombudsman`, `ombudsman_anonymous_web` (`app/services/platform/features.rb:206-211`); `CITIZEN_HIDDEN = %w[psychosocial_care]` (`:218`); `feature:<x>` exige a outra **utilizável** (`missing_for`, `:342-345`), e `territory_registry` requer `feature:clinical_record ibge_code uf` (`:126-128`), enquanto `TerritoryRegistry::Gate.usable?` ignora `ibge_code_missing`/`uf_missing` (`app/services/territory_registry/gate.rb:12-19`). `TerritoryRegistry::Access` (`REGISTRATION_CBOS` carrega **toda** chave do `config/ledi/registration_cbos.yml`; `membership(user, team)`; `registrar?` só conhece `:acs` e `:registrar`, `access.rb:12-44,99-104`). `TerritoryRegistry::Catalog::{ACS_CBO, REGISTRAR_CBO_PREFIXES = %w[2235 2251 2252 2253], TEAM_KINDS = %w[70 76], DUE_SOON_DAYS = 60, NO_CPF_REASONS, PROPERTY_TYPES, SEX_TO_LEDI}` (`catalog.rb:6-26`); `TerritoryRegistry::Validity.status(registration, today:, identified:)` (`validity.rb:16`); `TerritoryRegistry::Json.{micro_area,household,registration}` (`json.rb:7,25,54`); `TerritoryRegistry::Trail` (`access: "territory_registry"`). `TerritoryRegistry::SaveHousehold.call(params:, by:, household: nil, …)` (troca de microárea move os cadastros, `save_household.rb:19-20,48`) e `SaveRegistration` (`moved_in`, `save_registration.rb:347`). `micro_areas.acs_professional_id`, `health_teams.kind IN (70,71,76)`, `HealthTeamMember.active` (`ended_on`), `User#active?`/`#deactivate!` (`user.rb:27,35`). `Ledi::FichaTypes::REGISTRY` (5 tipos; `ficha_types.rb:8-26`), `Ledi::FichaSources::REGISTRY` (4 fontes; `ficha_sources.rb:6-8`, despachado pelo `ProductionController#retry_generation` e pelo `Ledi::Resend`), `Ledi::RegistrationFicha` (`generate`/`build`/`record_failure!`/`retry!`/`regenerate`/`originating_uuid`, `registration_ficha.rb`), `Ledi::HouseholdRegistrationFicha` (`extend Ledi::RegistrationFicha`, `sibling_ids`), `Ledi::Fichas::{RegistrationHeader, CitizenDocument}`, `Ledi::Enqueue.{call,accepting?}` (uuid `"#{cnes}-#{uuid}"`), `Ledi::Ficha.assert!`, `Ledi::Deadline.on(competence)`, `LediGenerationFailure::REASONS` (constante fechada, sem CHECK no banco; `ledi_generation_failure.rb:4-7`), `LediOutboxEntry::STATUSES`. IDL e `gen-rb` já têm `ficha_visita_domiciliar` e `ficha_atividade_coletiva` com os campos do contrato §10/§11 (`vendor/ledi/8.7.0/gen-rb/ficha_visita_domiciliar_types.rb:21-79`, `ficha_atividade_coletiva_types.rb:23-110`; `justificativaCidadaoNaoPossuiCpf` é I64). `vendor/ledi/8.7.0/docs` **não** tem `dicionario-fvd.html` nem `dicionario-fac.html`; o `SOURCE` lista `docs_sha256` e o `spec/services/ledi/registration_dictionaries_spec.rb:100-106` confere cada um. `cbo.html`: a legenda vem **depois** da tabela; Tabela 7 = 7 códigos; Tabela 5 = 117 linhas (115 numéricas + `2231F9` médico residente e `1312C1` sanitarista). `CareLines::Catalog.untracked?(source, vaccination: false)` com `UNTRACKED_SOURCE = /\A(acs_visit)\z/` (`catalog.rb:9,32`); `CareLines::Sources.load(patient_ids:, sources:, since:)` (`sources.rb:18-38`); `CareLines::Evaluate.dates` (`evaluate.rb:145-155`); `CareLines::Rules::PRACTICE_KEYS` (`rules.rb:6-7`); `CareLines::BaseController#vaccination_usable?` (`base_controller.rb:32`) e `CitizenApi::CareController` (`:20`); `config/care_lines/catalog-base.yml` com `acs_visit` em `c2_d c3_e c3_j c4_d c5_d c6_c` sem `citizen_label`. `Timeline::KINDS` (14 kinds, `app/services/timeline.rb:13`); `SOURCES`, `PROFESSIONAL_ONLY_KINDS`, `CITIZEN_FILTERED_KINDS` e `citizen_kinds` moram em `Timeline::Query` (`query.rb:12,23-37,76`); `PecTimelinesController::KINDS = (Timeline::KINDS - %w[transport])` (`:14`). `Citizens::RequestErasure.retained?` → `examined_patients?` (`request_erasure.rb:51,82-88`), `territory_registered?` retém quem tem qualquer versão de cadastro ou é responsável de família (`:161-163`). `Patients::CreateReviewed.call(new_patient:, reviewed:, by:)` (savepoint próprio; `fail(:duplicate_suspected)` **sem** candidatos; `create_reviewed.rb:14`), `Patients::Duplicates.call(...)`, `Patients::LookupByIdentifier`, `Patients::IdentifierJson`, `Pharmacy::Json.duplicate`; o padrão das buscas de pessoa é `Vaccination::{PersonLookupsController,DuplicatesController}` (404 `person_not_found`, 409 `citizen_not_verified`, 30/min, `too_many_requests`). Autenticação por token sem cookie: `Hl7::BaseController` (`rate_limit` por digest → interruptor → token pelo índice único, `WWW-Authenticate: Bearer`) e `LabMiddlewareToken.{digest,by_token}`. `MfaStepUp#require_step_up!` (401 `mfa_required`). `EachCityJob` (`app/jobs/concerns/each_city_job.rb`, `prepend`), `config/recurring.yml` (nenhuma tarefa às 3:20). `CityEncryption::CITY_KEYED_TARGETS` (`city_encryption.rb:36`); `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb:15`). `filter_parameters` já filtra `:token :body :items :reason :note :participants :procedure_code :new_patient :cpf :cns :authorization :birth_date :sex :full_name :mother_name :no_cpf_reason` (`config/initializers/filter_parameter_logging.rb`). Nenhum `access` de `clinical_record.viewed` tem lista fechada no api. Não há evento de fim de vínculo: `Cnes::Apply` encerra o membro com `ended_on` e só publica `cnes.proposals_applied` (`app/commands/cnes/apply.rb:42,60-62`). Specs que fixam listas e mudam: `spec/services/platform/features_spec.rb:121,128,137`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/operators/city_record_settings_spec.rb:30`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/services/timeline/query_spec.rb`, `spec/requests/pec_timeline_spec.rb`, `spec/services/territory_registry/access_spec.rb`, `spec/services/care_lines/{rules,preview}_spec.rb`. Helpers: `registry_city!`, `registry_team!`, `registry_acs!`, `registry_nurse!`, `registry_micro_area!`, `registry_household!`, `registry_family!`, `registry_registration!`, `registry_patient!`, `registry_no_cpf_patient!`, `valid_cpf`, `valid_cns` (`spec/support/territory_registry_helpers.rb`), `sign_in_as`, `stepped_up!`, `json_post`, `capture_log`, `spec/support/lock_wait.rb`.

**Goal:** O lado api do módulo 24b (F-24.9 a F-24.17, ADR 0052): interruptores `home_visits` e `collective_activities`, quem registra pelas Tabelas 7 e 5 do LEDI, atribuição de microárea, folha do dia com visitas (registro, retirada, fechamento pelo dono e pelo job, folha complementar), ficha LEDI tipo 8 por folha, atividade coletiva com participantes e ficha tipo 6, registro rápido de pessoa, visita nas linhas de cuidado (`acs_visit`, pressão, peso/altura, `min_gap_days`) e na linha do tempo, "quem visitar", aparelho de campo com token e a sincronização v1 (`/field/v1/{session,uploads,changes}`) testada contra a cópia dos schemas do `rotasaude/contracts`, retenção LGPD e sigilo.

**Architecture:** Três migrações de cidade (`900001` visitas, `900002` atividade coletiva, `900003` aparelho e sincronização), irreversíveis, com triggers em `db/city_triggers.sql`. A folha (`home_visit_days`) é a fonte da ficha 8: abre implicitamente na primeira visita do (profissional, equipe, dia) e fecha pelo dono, pelo job das 3h20 ou cheia (100ª visita). Toda gravação de visita passa por `HomeVisits::Record` (regras do §3 em `HomeVisits::Rules`, puras), com as travas em ordem paciente → domicílio → folha → visita. A ficha nasce em `Ledi::HomeVisitDayFicha.generate(day)` **dentro** da transação do fechamento (contrato do `Ledi::RegistrationFicha`: `generate`, `retry!`, `regenerate`). A atividade coletiva é uma gravação só (`CollectiveActivities::Record`, regras do §4 em `CollectiveActivities::Rules`, dirigidas por `config/ledi/visit_activity_dictionaries.yml`) e gera a ficha 6 na mesma transação. A sincronização tem autenticação própria (`FieldAuthentication`, sem cookie, no padrão do `Hl7::BaseController`), um livro de idempotência (`field_sync_receipts`) e um download por cursor opaco com `reset` por `scope_digest` e remoções por `field_sync_tombstones` gravadas pelos comandos do 24a.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), Solid Queue, RSpec via `bin/test`, `json_schemer` (já no Gemfile). **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md` (decisões 1A–7A e 1–23). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-12-module-24b-home-visit-contracts.md` (§1–§16); toda diferença está em "Divergências propostas ao contrato", no fim (W-1..W-13, A-1..A-12). ADR `docs/adr/0052.md`; pesquisa `docs/pesquisa/2026-11-12-visita-e-coletiva-24b.md`.

## Desvios e precisões

Cada item tem a divergência correspondente no fim; o plano segue a proposta até o coordenador decidir na §16 do contrato (quando a decisão for outra, a task dona muda só a constante ou a regra indicada).

**Reconciliação (§16 do contrato, Q1–Q30, 2026-11-12):** todas as W e A aceitas, salvo: A-4 decidida pela alternativa (Q4); A-7 com `status` calculado nas listas (Q7). Acréscimos que caem neste plano: leitura do 24a pelo CBO de campo nas microáreas atribuídas (Q11, Task 6); capacidades e `professionals`/`domains` no contexto (Q13, Q14, Tasks 6, 12, 15); busca com `sex` e candidatos com `sex`/`has_cns` (Q8, Task 7); corpo plano da atividade e `duplicates_reviewed` por participante (Q15, Tasks 13, 15); códigos novos da Q21; entrada do cidadão com meio-dia local e fora de `DECLARED_KINDS` (Q22, Q23, Task 17); `collective_activities` em `CITIZEN_HIDDEN` e descrição nova de `home_visits` (Q24, Task 2); `household_client_uuid` e `patient_client_uuid` no envio (Q27, Task 20).

1. **Migrações irreversíveis** (A-1): `down` → `ActiveRecord::IrreversibleMigration`, como 24a e 33; o `up` termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`.
2. **Folha cheia** (A-6): a 100ª visita não retirada de um (profissional, equipe, dia) fecha a folha aberta na mesma transação (`closed_by = "full"`, ficha gerada) e abre a `sequence + 1`. `closed_by IN ('user','job','full')`.
3. **Situação da ficha sem exportação** (A-5): folha fechada com visita válida, sem `ledi_outbox` e sem `LediGenerationFailure` (exportação não utilizável) → `ficha.status = "not_exported"`.
4. **Busca de pessoa** (A-8): `POST /home_visits/people/lookup` e `/collective_activities/people/lookup` seguem o `Vaccination::PersonLookupsController` (404 `person_not_found`, 409 `citizen_not_verified`, par validado vira paciente), com `birth_date` acrescentado à forma.
5. **Vínculo perdido** (A-7, Q7): revoga na próxima requisição do aparelho (§9.2), com o erro uniforme `401 device_revoked` — não há evento de fim de vínculo no api. `GET /field_devices*` devolve `status: active | inactive | revoked`, com `inactive` calculado na leitura (não revogado, mas sem vínculo ativo em equipe 70/76 com CBO da Tabela 7, ou usuário inativo) (Q7).
6. **Remoções** (A-9, A-10, A-11): `field_sync_tombstones.reason = "moved"` só quando o domicílio troca de microárea ou o cadastro individual sai da microárea (troca de domicílio ou de microárea); `erased` e `out_of_scope` ficam no enum e no schema sem produtor no 24b. Saída do cadastro (`moved`/`death`) e domicílio inativo vão como item, com o estado.
7. **Vitais da visita** (A-12): `blood_pressure` e `weight_height` da visita só com `home_visits` utilizável, pelo mesmo `home_visits:` que libera `acs_visit` (uma leitura do interruptor por chamada).
8. **Domínios** (A-2, A-3): até a task 1 decidir, `tipo_glicemia` = `0 Jejum, 1 Pós-prandial, 2 Pré-prandial, 3 Não especificado` (`tipoGlicemiaCapilar` do `dicionario.html`) e `publico_alvo` inclui `18 Outros`, `19` e `20`.
9. **Motivos de "não gerada"** (A-4, Q4): a ficha 8 reusa `citizen_without_birth_date` e `citizen_without_sex`; entram em `LediGenerationFailure::REASONS` só os seis novos: `siaps_deadline_passed`, `participant_without_identifier`, `participant_without_birth_date`, `participant_without_sex`, `participant_professional_without_cns`, `place_without_cnes`.
10. **Datas:** todo "hoje" é `Time.zone.today` dentro de `CityConnection.with`; dia da visita = `occurred_at.in_time_zone.to_date`; competência = `day.strftime("%Y%m")`; turno pelo `occurred_at.in_time_zone.hour` (§3).
11. **Recusas:** `{ error, path?, candidates? }`; 409 para `day_changed day_closed visit_changed activity_changed duplicate_suspected cpf_taken cns_taken already_assigned citizen_not_verified`; 403 para `cbo_not_allowed` (rota de registro e aparelho) e `out_of_scope`; 422 para `cbo_not_allowed` da atribuição e o resto; 404 `not_found` para registro de outro escopo por id.

## Valores fixados por este plano

1. **Interruptores** (depois de `ombudsman_anonymous_web` no `CATALOG`): `Entry.new(key: "home_visits", description: "Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo" (Q24), requires: %w[feature:territory_registry])`; `Entry.new(key: "collective_activities", description: "Atividade coletiva com ficha LEDI, sem PSE", requires: %w[feature:clinical_record])`. Gates `HomeVisits::Gate::KEY`/`CollectiveActivities::Gate::KEY` com `usable?(city) = city&.id && Platform::Features.usable?(city, KEY)` (sem a exceção de perfil do 24a; W-4).
2. **`HomeVisits::Catalog`:** `HOME_VISIT_CBOS = Access::REGISTRATION_CBOS[:home_visit]`; `ACS_CBO = "515105"`; `CARE_LINE_CBOS = %w[515105 322255]`; `OUTCOMES = { 1 => "done", 2 => "refused", 3 => "absent" }`; `SUBJECTS = %w[resident family_responsible none]`; `NON_RESIDENTIAL = %w[2 3 4 5 6 12 13]`; `RESIDENTIAL = %w[1 7 8 9 10 11 99]`; `NON_RESIDENTIAL_REASONS = [1, 27, 28, 31, 34, 35, 36, 37]`; `FEMALE_ONLY_REASONS = [5, 6]`; `MAX_VISITS_PER_DAY = 99`; `NOTE_MAX = 500`; `WITHDRAWAL_REASON = (10..500)`; `ORIGINS = %w[dashboard field_app]`; `CLOSED_BY = %w[user job full]`; `TRAIL_ACCESS = "home_visits"`; `FIELD_TRAIL_ACCESS = "field_sync"`; `MEASUREMENTS = { "weight_kg" => [0.5..500, 3], "height_cm" => [20..250, 1], "systolic" => [0..999, 0], "diastolic" => [0..999, 0], "temperature_c" => [20..45, 1], "glucose_mg_dl" => [0..800, 0], "glucose_type" => [nil, 0] }` (faixas confirmadas na task 1); `NOT_GENERATED = %w[unit_without_cnes professional_without_cns cbo_not_in_ledi_table city_profile_incomplete siaps_deadline_passed patient_without_identifier citizen_without_birth_date citizen_without_sex]` (Q4); `DOMAIN_EVENTS = %w[home_visit.recorded home_visit.withdrawn home_visit_day.closed micro_area.field_assignment_changed field_device.registered field_device.revoked]`.
3. **`CollectiveActivities::Catalog`:** `CBOS = Access::REGISTRATION_CBOS[:collective_activity]`; `TYPES = (1..7)`; `MEETING_TYPES = [1, 2, 3]`; `PARTICIPANTS_REQUIRED = [5, 6]`; `PRACTICE_TYPE6_ONLY = [2, 9, 33]`; `PROCEDURE_PRACTICE = 30`; `SMOKING_PRACTICES = [25, 26, 27, 28]` (34 pela task 1); `FORBIDDEN_HEALTH_TOPICS = [18]`; `PLACE_TEXT_MAX = 250`; `COUNT = (1..999)`; `MISTAKEN_REASON = (10..500)`; `TRAIL_ACCESS = "collective_activities"`; `NOT_GENERATED = %w[unit_without_cnes professional_without_cns cbo_not_in_ledi_table city_profile_incomplete siaps_deadline_passed participant_without_identifier participant_without_birth_date participant_without_sex participant_professional_without_cns place_without_cnes]` (Q4); `DOMAIN_EVENTS = %w[collective_activity.recorded collective_activity.marked_mistaken]`. As regras cruzadas do §4 moram em `config/ledi/visit_activity_dictionaries.yml` (`collective_rules`), lidas por `Rules` e devolvidas em `options.domains`.
4. **`FieldSync::Catalog`:** `CONTRACT = "field-sync-v1"`; `TOKEN_PREFIX = "rsf_"`; `TOKEN_BYTES = 32`; `QR_PREFIX = "rotasaude-field:v1:"`; `MAX_ACTIVE_DEVICES = 3`; `DEVICE_NAME = (1..60)`; `ITEM_TYPES = %w[home_visit home_visit_withdrawal collective_activity patient household individual_registration]`; `MAX_ITEMS = 50`; `MAX_BODY_BYTES = 2.megabytes`; `PAGE_LIMIT = (1..500)`; `DEFAULT_PAGE = 200`; `CUTOFF = 2.seconds`; `RATE = 120` por minuto; `LAST_USED_EVERY = 1.minute`; `REMOVAL_REASONS = %w[moved erased out_of_scope]`; `REVOKED_REASONS = %w[user admin membership_ended user_disabled]`.
5. **Rotas:** `scope "/home_visits", module: :home_visits`; `scope "/collective_activities", module: :collective_activities`; `resources :field_devices` + `admin`; `scope "/field/v1", module: "field_sync/v1"`. Controladores com `wrap_parameters false`.
6. **Arquivos de contrato:** `config/field_sync/{session-response-v1,upload-request-v1,upload-response-v1,changes-response-v1,domains-v1}.json`, cópia do `rotasaude/contracts/field-sync/` na tag `field-sync-v1.0.0`, presas por SHA-256 em `FieldSync::Schemas::SHA256` e conferidas em `spec/services/field_sync/schemas_spec.rb`.
7. **Dev:** api do worktree na porta **3057**; worktree `apps/api/.claude/mod24b` (com `config/master.key`); branch `feat/mod-24b-home-visits`; migrações `db/city_migrate/{20261112900001_add_home_visits,20261112900002_add_collective_activities,20261112900003_add_field_devices}.rb`; fim do dump `define(version: 2026_11_12_900003)`.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (os arquivos da task, por nome). Commits em inglês, Conventional Commits por extenso, sempre com o rodapé: `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge e push só com autorização do usuário.
- **Primeira ação da execução:** mover F-24.9..F-24.17 do board #1 para In Progress (um `gh project item-edit …` por chamada, começando com `gh`). Card vai a Done quando o último app dele terminar; Verified depois da prova no navegador.
- **Testes só por `bin/test api -w mod24b <specs>`** (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. `--keep-db` só em rodadas de arquivo isolado. Mudança transversal (migração de cidade, `CityEncryption`, `Platform::Features`, `Ledi`, `CareLines`, `Timeline`, `Citizens::RequestErasure`, 24a, `spec/support`, initializers) → **suíte completa obrigatória antes do merge, pedida ao usuário/sessão "Testes" na Task 26**.
- Migrações: dump à mão em `db/city_schema.rb`; triggers no bloco "Módulo 24b" do `db/city_triggers.sql` (depois do bloco do 33), cada um dentro de `IF to_regclass('public.<tabela>') IS NOT NULL`; **irreversíveis** (Desvio 1). **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs com tipo explícito (`type: :request`, `type: :model`, …): `infer_spec_type_from_file_location!` está desligado. Nunca simule cidade inalcançável com URL real.
- **Nunca `Current.city =` solto**; o job usa `EachCityJob`; interruptores lidos **fora** da transação da cidade e passados como argumento (`home_visits:`), no padrão do `vaccination:`. `spec/architecture/current_city_assignment_spec.rb` roda na Task 11 e na 26.
- **Sigilo:** motivo, medida, observação, tema, público, prática, procedimento, avaliação, nome, CPF, CNS e token nunca em URL, log, evento ou erro. Eventos só com ids (lista do contrato §1). Todo `Data`/`Struct`/modelo novo com dado pessoal sobrescreve `inspect`/`to_s`/`pretty_print`/`as_json` (só `id` e estado). Dado sensível só no corpo: `query_only!` em toda rota que lê pessoa ou texto.
- **`Platform.audit`:** nenhum nome novo (C17). Se uma task precisar de um, ele entra em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`) no mesmo commit — a Task 25 fixa que não houve.
- Eventos declarados em `config/initializers/domain_events.rb` (`to: []` sem consumidor) e em `spec/initializers/domain_events_bindings_spec.rb`; `DomainEvents.publish(` é multilinha — confira cada chamada inteira.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS`; toda guarda que deixa coluna cifrada mudar aceita a re-cifra (`rota.reencrypting`) antes das outras regras.
- `spec/adr_pointers_spec.rb` sobe `VALID_RANGE` para `(1..52)`.
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`, ao encerrar a task.

## Review Focus

1. **Duas visitas simultâneas no mesmo dia, uma delas a 100ª** (dois aparelhos ou aparelho + dashboard): a trava da folha (`FOR UPDATE` na linha `open`, depois do paciente e do domicílio) serializa; exatamente uma folha fica `closed_by = full` com 99 visitas e a outra visita vai para a `sequence + 1`; nenhuma visita fica sem folha nem em folha fechada. Teste na Task 8.
2. **Retirada × fechamento:** o dono retira enquanto o job fecha a folha: ou a retirada entra antes (a ficha sai sem ela) ou recebe 409 `day_closed`; a ficha nunca leva visita retirada. Teste na Task 10.
3. **Envio repetido do app:** o mesmo lote reenviado (rede caiu depois do commit) devolve `duplicate` com o mesmo `id` para cada aceito, sem nova visita, sem novo evento e sem nova trilha; um item `rejected` no meio não muda os outros; `patient_client_uuid` de um paciente aceito em lote anterior resolve. Teste na Task 20.
4. **Escopo do download:** atribuição encerrada entre duas páginas → a página seguinte responde `reset: true`; nenhuma página traz domicílio de microárea fora do escopo; a trilha de cada página tem exatamente os `household_ids`/`patient_ids` dela. Teste na Task 22.
5. **Cidadão:** nenhuma resposta do canal do cidadão traz visita recusada, ausente ou retirada, motivo, medida, observação ou nome do profissional; "Meu acompanhamento" só conta a visita com o interruptor ligado. Teste nas Tasks 17 e 25.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod24b/`)
- Migrações: as três do Valor fixado 7.
- Modelos: `app/models/{micro_area_field_assignment,home_visit_day,home_visit,collective_activity,collective_activity_professional,collective_activity_participant,field_device,field_sync_receipt,field_sync_tombstone}.rb`.
- `app/services/home_visits/{gate,catalog,access,rules,json,trail,blockers,who_to_visit,people}.rb`; `app/services/collective_activities/{gate,catalog,access,rules,json,trail}.rb`; `app/services/field_sync/{catalog,schemas,token,scope,cursor,changes,upload,json}.rb`.
- `app/commands/home_visits/{record,withdraw,close_day,assign_micro_area,end_assignment}.rb`; `app/commands/collective_activities/{record,mark_mistaken}.rb`; `app/commands/field_devices/{register,revoke}.rb`; `app/commands/field_sync/tombstone.rb`.
- `app/services/ledi/{home_visit_day_ficha,collective_activity_ficha}.rb`; `app/services/ledi/fichas/{home_visit_day,collective_activity}.rb`.
- `app/services/timeline/sources/{home_visit_source,collective_activity_source}.rb`.
- `app/jobs/home_visits/close_days_job.rb`.
- `app/controllers/concerns/{home_visits_gate,collective_activities_gate,field_authentication}.rb`; `app/controllers/home_visits/{base,context,days,visits,assignments,people,who_to_visit}_controller.rb`; `app/controllers/collective_activities/{base,options,activities,people}_controller.rb`; `app/controllers/field_devices_controller.rb`, `app/controllers/field_devices/admin_controller.rb`; `app/controllers/field_sync/v1/{base,sessions,uploads,changes}_controller.rb`.
- `config/ledi/visit_activity_dictionaries.yml`; `config/field_sync/*.json`; `vendor/ledi/8.7.0/docs/{dicionario-fvd,dicionario-fac}.html`.
- `lib/home_visit_crew.rb`; `spec/support/home_visit_helpers.rb`; specs listados em cada task.

**Modificar**
- Plataforma e infra: `app/services/platform/features.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `db/seeds.rb`.
- LEDI: `config/ledi/registration_cbos.yml`, `vendor/ledi/8.7.0/SOURCE`, `app/services/ledi/{ficha_types,ficha_sources}.rb`, `app/models/ledi_generation_failure.rb`.
- Linhas de cuidado: `app/services/care_lines/{catalog,sources,rules,evaluate,evaluation,preview,citizen_view,board}.rb`, `app/controllers/care_lines/base_controller.rb`, `app/controllers/citizen_api/care_controller.rb`, `config/care_lines/catalog-base.yml`.
- Linha do tempo: `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/controllers/pec_timelines_controller.rb`.
- 24a: `app/commands/territory_registry/{save_household,save_registration}.rb`.
- LGPD: `app/commands/citizens/request_erasure.rb`.
- Specs existentes: `spec/adr_pointers_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/models/city_record_settings_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`, `spec/services/ledi/registration_dictionaries_spec.rb`, `spec/services/territory_registry/access_spec.rb`, `spec/services/timeline/query_spec.rb`, `spec/requests/pec_timeline_spec.rb`, `spec/services/care_lines/{rules,preview}_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`.

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-24.9..F-24.17 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod24b -b feat/mod-24b-home-visits origin/main && cp config/master.key .claude/mod24b/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO com `grep` no worktree: `define(version: 2026_11_11_900002)`, `ls db/city_migrate | tail -1`, `ombudsman_anonymous_web` como última `Entry`, `UNTRACKED_SOURCE`, `REGISTRY = {` das duas tabelas LEDI, `REASONS = ` do `LediGenerationFailure`, `KINDS = ` do `Timeline` e do `PecTimelinesController`, `def territory_registered?`, `VALID_RANGE`, e que `home_visit`/`field_device` não existem em `db/city_schema.rb`. Diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod24b spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb spec/services/ledi/registration_dictionaries_spec.rb spec/services/territory_registry/access_spec.rb spec/services/care_lines/rules_spec.rb spec/services/timeline/query_spec.rb spec/commands/citizens/territory_registry_erasure_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS (linha de base).

**Relato ao coordenador:** "api 24b · Task 0 · worktree mod24b em origin/main <hash>, pré-requisito conferido (diferenças: …), linha de base verde."

---

### Task 1: Prova técnica — dicionários da FVD e da FAC, CBOs e tipos de ficha

**Files:**
- Create: `vendor/ledi/8.7.0/docs/{dicionario-fvd,dicionario-fac}.html`, `config/ledi/visit_activity_dictionaries.yml`, `spec/services/ledi/visit_activity_dictionaries_spec.rb`
- Modify: `vendor/ledi/8.7.0/SOURCE`, `config/ledi/registration_cbos.yml`, `app/services/ledi/ficha_types.rb`, `spec/services/ledi/registration_dictionaries_spec.rb`, `spec/services/territory_registry/access_spec.rb`

**Interfaces:**
- `SOURCE`: `docs_fetched_on` ganha "(dicionario-fvd.html e dicionario-fac.html: 2026-11-1x)" e `docs_sha256` as duas linhas novas (`docs/dicionario-fvd.html: <sha>`), baixadas de `https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/` (a mesma origem dos outros; arquivos novos num diretório vazio, conferidos à mão antes de mover para `vendor/`).
- `config/ledi/visit_activity_dictionaries.yml`: `source: { fvd: <sha>, fac: <sha> }`; `home_visit: { domains: { motivo_visita: { <código>: { label, group } }, desfecho, tipo_de_imovel_non_residential, turno, tipo_glicemia, justificativa_nao_possui_cpf: "territory" }, measurements: { <chave>: { ledi, min, max, decimals } }, rules: { non_residential_reasons, female_only_reasons, cns_required_with_no_cpf: <bool>, birth_and_sex_required: "with_person" | … } }`; `collective_activity: { domains: { tipo_atividade_coletiva, temas_para_reuniao, temas_para_saude, praticas_em_saude, publico_alvo }, rules: { <campo>: { "<tipo>": required | forbidden | optional } , health_topics_meeting_exception: 6, type6_only_practices, procedure_practice, smoking_practices, forbidden_health_topics, others_target_audience } }`.
- `registration_cbos.yml`: `home_visit` (7 códigos da Tabela 7) e `collective_activity` (as 117 linhas da Tabela 5, **strings**, inclusive `"2231F9"` e `"1312C1"`); comentário do topo cita as Tabelas 5 e 7. `TerritoryRegistry::Access.registrar_cbo?(cbo, ficha: :home_visit | :collective_activity)` passa a responder sem mudança de código (W-3).
- `Ledi::FichaTypes::REGISTRY`: `visita_domiciliar` → `{ code: 8, klass: "Br::Gov::Saude::Esusab::Ras::Visitadomiciliar::FichaVisitaDomiciliarMasterThrift", uuid_field: :uuidFicha }`; `atividade_coletiva` → `{ code: 6, klass: "Br::Gov::Saude::Esusab::Ras::Atividadecoletiva::FichaAtividadeColetivaThrift", uuid_field: :uuidFicha }`.

- [ ] **Step 1: Write the failing tests** `spec/services/ledi/visit_activity_dictionaries_spec.rb` (`type: :model`):

```ruby
RSpec.describe "Dicionários da FVD e da FAC (LEDI 8.7.0)", type: :model do
  let(:yaml) { YAML.load_file(Rails.root.join("config/ledi/visit_activity_dictionaries.yml")) }

  def cbo_table(caption)
    html = Ledi::Version.root.join("docs/cbo.html").read
    finish = html.index(caption)
    html[html.rindex("<table", finish)...finish].scan(%r{<tr>\s*<td>\s*([0-9A-Z]{6})\s*</td>}).flatten
  end

  it "registration_cbos: home_visit é a Tabela 7 e collective_activity é a Tabela 5, inteiras" do
    cbos = YAML.load_file(Rails.root.join("config/ledi/registration_cbos.yml"))
    expect(cbos.fetch("home_visit")).to eq(cbo_table("Tabela 7 -"))
    expect(cbos.fetch("home_visit")).to eq(%w[515105 515310 515140 515130 515125 322255 515120])
    expect(cbos.fetch("collective_activity")).to eq(cbo_table("Tabela 5 -"))
    expect(cbos.fetch("collective_activity").size).to eq(117)
    expect(cbos.fetch("collective_activity")).to include("2231F9", "1312C1", "352210")
  end

  it "tipos de ficha 8 e 6 batem com TipoDadoSerializado e com as classes do gen-rb" do
    expect(Ledi::FichaTypes.code("visita_domiciliar")).to eq(8)
    expect(Ledi::FichaTypes.code("atividade_coletiva")).to eq(6)
    expect(Ledi::FichaTypes.klass("visita_domiciliar")).to eq(Br::Gov::Saude::Esusab::Ras::Visitadomiciliar::FichaVisitaDomiciliarMasterThrift)
    expect(Ledi::FichaTypes.klass("atividade_coletiva")).to eq(Br::Gov::Saude::Esusab::Ras::Atividadecoletiva::FichaAtividadeColetivaThrift)
  end

  it "cada medida aponta um campo real de FichaVisitaDomiciliarChildThrift" do
    fields = Br::Gov::Saude::Esusab::Ras::Visitadomiciliar::FichaVisitaDomiciliarChildThrift::FIELDS.values.map { |f| f[:name] }
    yaml.dig("home_visit", "measurements").each_value { |m| expect(fields).to include(m.fetch("ledi")) }
  end

  it "os domínios da visita e da atividade são os do dicionário vendorizado (fvd, fac e dicionario.html)" do
    # parse das tabelas de código (helper do spec do 24a) para MotivoVisita, Turno, TipoGlicemia/tipoGlicemiaCapilar,
    # TipoAtividadeColetiva, TemasParaReuniao, TemasParaSaude, PraticasEmSaude, PublicoAlvo:
    # cada código do YAML existe no HTML com o mesmo rótulo; nenhum código do HTML falta, salvo o 18 dos temas (PSE).
  end

  it "o tema 18 (Semana saúde na escola) fica fora e o PSE nunca é opção" do
    expect(yaml.dig("collective_activity", "domains", "temas_para_saude").keys.map(&:to_i)).not_to include(18)
    expect(yaml.dig("collective_activity", "rules", "forbidden_health_topics")).to eq([ 18 ])
  end
end
```

`registration_dictionaries_spec.rb:102` ganha `"dicionario-fvd.html", "dicionario-fac.html"` no `include`. `access_spec.rb` ganha "registrar_cbo? aceita :home_visit (515105, 322255) e :collective_activity (2231F9), e recusa 225125 em :home_visit".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/ledi/visit_activity_dictionaries_spec.rb spec/services/ledi/registration_dictionaries_spec.rb spec/services/territory_registry/access_spec.rb` → FAIL.
- [ ] **Step 3: Implementar e ler.** Baixar e vendorizar os dois HTMLs; gravar o SHA; ler as regras oficiais das duas fichas e conferir contra o IDL e o `gen-rb`; preencher o YAML. Fechar, por escrito numa tabela (vai ao relato e ao coordenador), **cada** ponto "a confirmar" do contrato: `dataAtendimento` da folha (00h local em ms?); CNS obrigatório com `stCidadaoNaoPossuiCpf = true` na MIVDT; quando nascimento e sexo são obrigatórios (residencial com pessoa; nunca sem pessoa); domínio e ordem de `tipoGlicemia` (A-2); faixas das medidas; `health_topics` nos tipos 1–3 com o tema 6 e no tipo 6; código do "Outros" do público-alvo e os códigos 19/20 (A-3); `smoking` na prática 34; obrigatoriedade de cada lista por tipo; mais de uma ficha do mesmo profissional no mesmo dia (fica gate de go-live). Diferença da tabela do contrato muda **só** o YAML e vira item novo em "Divergências" (A-n).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: vendor LEDI home visit and collective activity dictionaries with field CBO tables" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.
- [ ] **Step 6: Publicar.** Mandar ao coordenador ("Próximos módulos") a tabela de domínios e regras (§10, §11 e as regras do §3/§4) para ele acrescentar ao contrato; o `domains-v1.json` é escrito pela sessão do contracts (Task 4 do plano dela), que devolve os SHA-256 novos para `FieldSync::Schemas::SHA256` (Q13); a tag `field-sync-v1.0.0` sai depois disso. As tasks do dashboard só começam depois desse acréscimo.

**Relato ao coordenador:** "api 24b · Task 1 · fvd/fac vendorizados (sha …); Tabela 7 = 7, Tabela 5 = 117 (2 alfanuméricos); tabela de domínios e regras anexada (tipoGlicemia …, público Outros …, 19/20 …, CNS sem CPF …, dataAtendimento …); pedir acréscimo ao §10–§11 e ao domains-v1.json antes da tag."

---

### Task 2: Interruptores, portões, catálogos e filtro de log (transversal: lista de interruptores)

**Files:**
- Create: `app/services/home_visits/{gate,catalog}.rb`, `app/services/collective_activities/{gate,catalog}.rb`, `app/services/field_sync/catalog.rb`, `app/controllers/concerns/{home_visits_gate,collective_activities_gate}.rb`, `spec/services/home_visits/catalog_spec.rb`
- Modify: `app/services/platform/features.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/services/platform/features_spec.rb`, `spec/models/city_record_settings_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`, `spec/adr_pointers_spec.rb`

**Interfaces:**
- Produces: Valores fixados 1–4; concerns `HomeVisitsGate#require_home_visits!` e `CollectiveActivitiesGate#require_collective_activities!` (403 `{ error: "feature_disabled", feature: KEY }`); filtro novo `/\A(measurements|meeting_topics|target_audience|health_topics|health_practices|altered_evaluation|smoking|qr_payload|place_text|candidates|withdrawal_reason|mistaken_reason)\z/` (W-7).

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: `keys.last(5).first(2)` → `keys.last(7).first(2)`; `CATALOG[-3]` → `CATALOG[-5]`; o do 33 passa a `keys.last(4).first(2)`; exemplo novo:

```ruby
it "módulo 24b: home_visits e collective_activities fecham o catálogo; só home_visits vai ao cidadão (Q24)" do
  keys = Platform::Features::CATALOG.map(&:key)
  expect(keys.last(2)).to eq(%w[home_visits collective_activities])
  expect(described_class.find("home_visits").requires).to eq(%w[feature:territory_registry])
  expect(described_class.find("collective_activities").requires).to eq(%w[feature:clinical_record])
  expect(described_class::CITIZEN_HIDDEN).not_to include("home_visits")
  expect(described_class::CITIZEN_HIDDEN).to include("collective_activities")
  expect(described_class.find("home_visits").description).to start_with("Folha do dia")
end

it "home_visits com o cadastro territorial sem IBGE fica inutilizável (territory_registry_disabled)" do
  city = registry_city!
  CityConnection.with(city) { CityProfile.current.update!(ibge_code: nil) }
  Platform::Features.set!(city: city, key: "home_visits", enabled: true, maintainer: ledi_maintainer!)
  expect(Platform::Features.missing(city, "home_visits")).to eq(%w[territory_registry_disabled])
  expect(HomeVisits::Gate.usable?(city)).to be(false)
  expect(TerritoryRegistry::Gate.usable?(city)).to be(true)
end
```

As três listas exatas ganham `home_visits collective_activities` no fim; `session_features_spec.rb` ganha "GET /citizen/session traz home_visits e não traz collective_activities quando ligados" (Q24). `catalog_spec.rb` (`type: :model`): `NON_RESIDENTIAL + RESIDENTIAL` = `TerritoryRegistry::Catalog::PROPERTY_TYPES` (sem sobra); `CARE_LINE_CBOS ⊂ HOME_VISIT_CBOS`; `DOMAIN_EVENTS` dos três catálogos = exatamente os 8 do contrato §1; `ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters)` filtra `measurements`, `meeting_topics`, `target_audience`, `health_topics`, `health_practices`, `altered_evaluation`, `smoking`, `qr_payload`, `place_text`, `note`, `reasons`, `token`, `items`, `body`, `new_patient`.
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/platform/features_spec.rb spec/services/home_visits/catalog_spec.rb spec/models/city_record_settings_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/citizen_api/session_features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** as duas entradas (comentário: "ADR 0052: sem cascata; desligar o cadastro territorial deixa home_visits ligado faltando pré-requisito; o job de fechamento roda mesmo assim"), os gates, os concerns, os catálogos, o filtro e `VALID_RANGE = (1..52)`.
- [ ] **Step 4:** os mesmos specs + `spec/adr_pointers_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add home visit and collective activity switches" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 2 · `home_visits` e `collective_activities` no fim do CATALOG (missing `territory_registry_disabled` com IBGE faltando, W-4); gates 403 feature_disabled; filtro de log novo (W-7)."

---

### Task 3: Esquema das visitas — atribuição, folha e visita (migração 900001)

**Files:**
- Create: `db/city_migrate/20261112900001_add_home_visits.rb`, `app/models/{micro_area_field_assignment,home_visit_day,home_visit}.rb`, `spec/models/home_visit_schema_spec.rb`, `spec/support/home_visit_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`

**Interfaces:**
- Tabelas do contrato §1, com: `home_visit_days.closed_by` CHECK `IN ('user','job','full')` (Desvio 2) e `ck_home_visit_days_closed` = `(state = 'closed') = (closed_at IS NOT NULL) AND (state = 'closed') = (closed_by IS NOT NULL) AND (closed_by_user_id IS NULL OR closed_by = 'user')`; único `(professional_id, team_id, day, sequence)`; único parcial `(professional_id, team_id, day) WHERE state = 'open'`; `cbo` `string(6)`. `home_visits`: os CHECKs do contrato §1 (`ck_home_visits_area`, `ck_home_visits_subject`, `ck_home_visits_family`, `ck_home_visits_outcome_reason`, `ck_home_visits_withdrawn`, `ck_home_visits_non_residential`), `outcome IN (1,2,3)`, `shift IN (1,2,3)`, `origin` CHECK e `(origin = 'field_app') = (field_device_id IS NOT NULL)` (a FK para `field_devices` entra na 900003); índices do contrato. `micro_area_field_assignments`: único parcial `(micro_area_id, professional_id) WHERE ended_at IS NULL`.
- Triggers (bloco "Módulo 24b — visitas"): `rota_home_visit_day_guard()` (DELETE recusado; só `open → closed`, com `closed_*` juntos; fechada não muda); `rota_home_visit_guard()` (DELETE recusado; re-cifra de `reasons measurements note withdrawal_reason` passa primeiro; fora dela, só `withdrawn_at`, `withdrawn_by_user_id`, `withdrawal_reason` de NULL a valor, uma vez, e só com a folha `open` — lida no trigger); `rota_micro_area_field_assignment_guard()` (DELETE recusado; só `ended_at`/`ended_by_user_id` de NULL a valor); TRUNCATE recusado (`rota_append_only()`) nas três.
- `CITY_KEYED_TARGETS` (comentário "ADR 0052 (módulo 24b): …"): `[HomeVisit, :reasons|:measurements|:note|:withdrawal_reason]`.
- Modelos: `HomeVisitDay` (`open?`, `closed?`, `competence`, `scope :open_on(professional, team, day)`, `inspect` só `id`/`state`); `HomeVisit` (`recorded?`, `withdrawn?`, `scope :valid` = não retirada; `encrypts :reasons, :measurements` (JSON), `:note`, `:withdrawal_reason`; `inspect` só `id`/`state`); `MicroAreaFieldAssignment` (`scope :active`).
- `spec/support/home_visit_helpers.rb`: `home_visit_city!(collective: false)` (`registry_city!` + interruptores), `field_agent!(cbo:, team:)`, `home_visit_day!(…)`, `home_visit!(…)` (pelo `HomeVisits::Record` a partir da Task 8; antes, `create!` cru), `assignment!(micro_area, professional)`.

- [ ] **Step 1: Write the failing test** `spec/models/home_visit_schema_spec.rb` (`type: :model`): "uma folha aberta por (profissional, equipe, dia); a segunda abre só com sequence nova depois de fechar"; "folha fechada não muda e não volta a open; closed_by_user_id só com user"; "visita: fora de área sem microárea; não residencial sem pessoa nem família; recusa sem motivo"; "visita só muda os três campos de retirada, uma vez, e só com a folha aberta" (com a folha fechada: `update_columns(withdrawn_at: …)` → `ActiveRecord::StatementInvalid, /home_visits/`); "nenhuma das três aceita DELETE nem TRUNCATE"; "atribuição só encerra, uma vez; duas ativas iguais → RecordNotUnique"; "re-cifra passa (`SET LOCAL rota.reencrypting = 'on'`) em visita de folha fechada".
- [ ] **Step 2:** `bin/test api -w mod24b spec/models/home_visit_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`create_table` das três, índices, CHECKs, `execute File.read(…city_triggers.sql)`, `down` → `IrreversibleMigration`), o dump (`define(version: 2026_11_12_900001)`), o bloco de triggers, os modelos, a cifra, o helper.
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add home visit day, home visit and micro area assignment schema" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 3 · migração 20261112900001 irreversível (A-1); `closed_by` aceita `full` (A-6); retirada só com folha aberta no trigger."

---

### Task 4: Esquema da atividade coletiva (migração 900002)

**Files:**
- Create: `db/city_migrate/20261112900002_add_collective_activities.rb`, `app/models/{collective_activity,collective_activity_professional,collective_activity_participant}.rb`, `spec/models/collective_activity_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`

**Interfaces:**
- Tabelas do contrato §1: CHECKs `activity_type BETWEEN 1 AND 7`, `shift IN (1,2,3)`, `num_nonnulls(place_health_unit_id, place_text) = 1`, `participants_count BETWEEN 1 AND 999`, `altered_count BETWEEN 0 AND participants_count`, `(mistaken_at IS NULL) = (mistaken_by_user_id IS NULL) AND (mistaken_at IS NULL) = (mistaken_reason IS NULL)`, origem como na visita; únicos `client_uuid`, `(collective_activity_id, professional_id)`, `(collective_activity_id, patient_id)`; índice `(team_id, held_on)`.
- Triggers: `rota_collective_activity_guard()` (DELETE recusado; re-cifra primeiro; fora dela só os três campos de engano de NULL a valor, uma vez); `rota_append_only()` em `collective_activity_professionals` e `collective_activity_participants` (UPDATE/DELETE/TRUNCATE; re-cifra do participante por `rota_reencryption_only('altered_evaluation','measurements','smoking')`).
- `CITY_KEYED_TARGETS`: `[CollectiveActivity, :meeting_topics|:target_audience|:health_topics|:health_practices|:procedure_code|:mistaken_reason]`, `[CollectiveActivityParticipant, :altered_evaluation|:measurements|:smoking]`.

- [ ] **Step 1: Write the failing test** `spec/models/collective_activity_schema_spec.rb` (`type: :model`): "local é unidade ou texto, nunca os dois"; "avaliações alteradas ≤ participantes"; "a atividade só muda para por engano, uma vez"; "profissionais e participantes são só acréscimo"; "participante repetido → RecordNotUnique"; "re-cifra passa".
- [ ] **Step 2:** `bin/test api -w mod24b spec/models/collective_activity_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_12_900002)`).
- [ ] **Step 4:** os mesmos specs + `spec/models/home_visit_schema_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add collective activity schema with participants and professionals" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 4 · migração 20261112900002 irreversível; atividade imutável salvo `mistaken_*`."

---

### Task 5: Esquema do aparelho e da sincronização (migração 900003)

**Files:**
- Create: `db/city_migrate/20261112900003_add_field_devices.rb`, `app/models/{field_device,field_sync_receipt,field_sync_tombstone}.rb`, `spec/models/field_device_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`

**Interfaces:**
- `field_devices` (contrato §1): `token_digest string(64)` único, CHECK `token_digest ~ '^[0-9a-f]{64}$'`, `length(btrim(name)) BETWEEN 1 AND 60`, `revoked_reason` CHECK, `(revoked_at IS NULL) = (revoked_reason IS NULL)`, `revoked_by_user_id` só com `admin|user`; índice `(user_id) WHERE revoked_at IS NULL`. FKs novas: `home_visits.field_device_id` e `collective_activities.field_device_id` → `field_devices`.
- `field_sync_receipts`: `client_uuid` único, `item_type` CHECK `IN (<ITEM_TYPES>)`, `result_type`, `result_id uuid NOT NULL`.
- `field_sync_tombstones`: `entity_type IN ('household','registration')`, `reason IN ('moved','erased')`, índice `(micro_area_id, created_at)`.
- Triggers: `rota_field_device_guard()` (DELETE recusado; `token_digest`, `user_id`, `professional_id`, `name`, `created_at` imutáveis; `revoked_*` de NULL a valor, uma vez; `last_used_at` livre); `rota_append_only()` em `field_sync_receipts` e `field_sync_tombstones`.
- `FieldDevice.digest(token)`, `.by_token(token)` (só `String` com prefixo `rsf_` e 40–60 caracteres; `find_by(token_digest:)`), `#revoked?`, `scope :active`, `inspect` sem o digest.

- [ ] **Step 1: Write the failing test** `spec/models/field_device_schema_spec.rb` (`type: :model`): "o digest é único e nunca muda"; "revogar é final"; "by_token acha pelo SHA-256 e ignora texto malformado"; "livro e tombstones são só acréscimo"; "`inspect` não mostra o digest".
- [ ] **Step 2:** `bin/test api -w mod24b spec/models/field_device_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_12_900003)`).
- [ ] **Step 4:** o spec + os das Tasks 3–4 → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add field device, sync receipt and tombstone schema" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 5 · migração 20261112900003; token só como SHA-256; livro de idempotência e tombstones só acréscimo."

---

### Task 6: Acesso, contexto e atribuição de microárea (§1, §6)

**Files:**
- Create: `app/services/home_visits/{access,json,trail}.rb`, `app/services/collective_activities/access.rb`, `app/commands/home_visits/{assign_micro_area,end_assignment}.rb`, `app/controllers/home_visits/{base,context,assignments}_controller.rb`, `spec/services/home_visits/access_spec.rb`, `spec/requests/home_visits/assignments_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `HomeVisits::Access`: `member(user, team)` (= `TerritoryRegistry::Access.membership` com equipe em `TEAM_KINDS`); `role(member) -> :acs | :field | :reader | nil` (`515105` → `:acs`; outro da Tabela 7 → `:field`; `REGISTRAR_CBO_PREFIXES` → `:reader`); `scope(user, team) -> { micro_area_ids:, kind: "responsible" | "assigned" | "team" }`; `registers?(user, team, micro_area:, out_of_area:) -> :ok | :out_of_scope | :cbo_not_allowed`; `reads_micro_area?(user, micro_area)`; `reads_team?(user, team)`; `assigner?(user, team)` (enfermeiro `2235*` da equipe); `device_owner?(user) -> member | nil` (Tabela 7 com vínculo ativo em equipe 70/76).
- `CollectiveActivities::Access`: `registers?(user, team)` (membro ativo com CBO da Tabela 5); `reads?(user, team)` (registra, ou enfermeiro/médico da equipe); `marker?(user, activity)` (quem registrou).
- `HomeVisits::BaseController`: `include Authentication, MfaStepUp, ::HomeVisitsGate`; `wrap_parameters false`; `FAILURE_STATUS` (Desvio 11); `render_failure`, `render_refusal`, `body`, `body_param`, `query_only!`, `page_limit`, `load_team!` (403 `out_of_scope` fora da equipe), `uuid?`.
- `GET /home_visits/context` e `/assignments` (GET, POST, `/:id/end`) no formato do contrato §6; atribuição e encerramento com `require_step_up!`; `micro_area.field_assignment_changed` (`micro_area_id`, `professional_id`, `user_id`, `ended`). O contexto traz `can_assign`, `can_register_device`, `professionals` (só com `reads_whole_team`) e `domains` (do bloco `home_visit` do YAML da task 1, na forma do §7 do contrato; um spec confere que códigos e regras batem com a cópia de `domains-v1.json`, Q13) (Q13, Q14); encerrar já encerrada → `409 assignment_already_ended` (Q21).
- **Leitura do 24a pelo CBO de campo (Q11):** `TerritoryRegistry::Access` ganha a leitura (só leitura) de `GET /territory/households?micro_area_id=`, `/territory/households/:id` e `/territory/registrations?household_id=` para o CBO da Tabela 7 que não é ACS, nas microáreas com `micro_area_field_assignments` ativa dele (por `HomeVisits::Access.scope`); microárea não atribuída → 403 `out_of_scope`, domicílio fora por id → 404; trilha do 24a como está. Escrita, `person_lookup` e as demais rotas do 24a não mudam (`cbo_not_allowed`). Modify também `app/services/territory_registry/access.rb` e `spec/services/territory_registry/access_spec.rb` (transversal: 24a).

- [ ] **Step 1: Write the failing tests.** `access_spec.rb` (`type: :model`): (Q11) "TACS lê domicílios e moradores da microárea atribuída pelas rotas do 24a, não os da não atribuída (403), e não escreve (cbo_not_allowed); encerrar a atribuição tira a leitura na hora"; "ACS registra na sua microárea e fora de área; não na do colega (out_of_scope)"; "TACS só na atribuída; fora de área → out_of_scope"; "enfermeiro e médico leem a equipe e não registram (cbo_not_allowed)"; "municipal_admin não lê"; "atribuição encerrada tira o escopo na hora"; "vínculo encerrado (`ended_on`) tira tudo". `assignments_spec.rb` (`type: :request`): "enfermeiro atribui com step-up (201) e sem step-up recebe 401 mfa_required"; "repetido → 409 already_assigned"; "ACS responsável pela própria ou CBO fora da Tabela 7 → 422 cbo_not_allowed"; "candidates sem o ACS responsável"; "encerrar → 200 e evento com `ended: true` só com ids"; "ACS lê o contexto com `scope: responsible`, TACS com `assigned`, enfermeiro com `team`"; "interruptor desligado → 403 feature_disabled".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/home_visits/access_spec.rb spec/requests/home_visits/assignments_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add home visit access rules, context and micro area assignment" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 6 · acesso pela tabela do §1; `/home_visits/context` e `/assignments` prontos para o dashboard (step-up na escrita)."

---

### Task 7: Registro rápido e buscas de pessoa (§5)

**Files:**
- Create: `app/services/home_visits/people.rb`, `app/controllers/home_visits/people_controller.rb`, `app/controllers/collective_activities/{base,people}_controller.rb`, `spec/services/home_visits/people_spec.rb`, `spec/requests/home_visits/people_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `HomeVisits::People.resolve(patient_id: nil, new_patient: nil, reviewed: false, by:) -> Result.ok(patient:, created:) | fail(:duplicate_suspected, details: { candidates: }) | fail(:cpf_taken | :cns_taken, details: { path: }) | fail(:invalid_request, details: { path: })` — chama `Patients::CreateReviewed.call` **dentro** da transação de quem grava; no `duplicate_suspected`, os candidatos vêm de `Patients::Duplicates.call(…, limit: 10)` na forma `Pharmacy::Json.duplicate`, com `Trail.person_viewed!` por candidato. Usado pela visita, pela atividade e pelo item `patient` do envio.
- `POST /home_visits/people/{lookup,duplicates}` e `POST /collective_activities/people/{lookup,duplicates}`: o padrão do `Vaccination::{PersonLookupsController,DuplicatesController}` (Desvio 4), 30/min por usuário, `too_many_requests`, só corpo; a forma do lookup ganha `birth_date` e `sex`, e os candidatos (da busca e do `409 duplicate_suspected`) ganham `sex` e `has_cns` (Q8). O 409 traz `path` (`new_patient`, ou `participants[i].new_patient` na atividade, com `reviewed` lido de cada participante; Q15). Acesso: quem registra no módulo.

- [ ] **Step 1: Write the failing tests.** `people_spec.rb` (serviço, `type: :model`): "paciente novo com CPF cria"; "nome+nascimento de alguém existente sem `duplicates_reviewed` → duplicate_suspected com até 10 candidatos e trilha"; "com `true` cria"; "CPF de outro → cpf_taken com path new_patient.cpf"; "recusa desfaz só o savepoint". `people_spec.rb` (`type: :request`): "lookup por CPF → paciente com birth_date, has_cpf, cns_masked e trilha"; "ninguém → 404 person_not_found"; "CPF na query → 422 invalid_request"; "31ª busca no minuto → 429 too_many_requests"; "enfermeiro (não registra visita) → 403 cbo_not_allowed"; o mesmo par de exemplos para `/collective_activities/people/*`.
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/home_visits/people_spec.rb spec/requests/home_visits/people_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add quick patient registration and person lookups for field records" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 7 · `/home_visits/people/*` e `/collective_activities/people/*` no padrão do 29 (404 person_not_found, 409 citizen_not_verified; A-8); candidatos do duplicado com trilha."

---

### Task 8: Registrar visita — regras do §3, folha e complementar

**Files:**
- Create: `app/services/home_visits/rules.rb`, `app/commands/home_visits/record.rb`, `spec/services/home_visits/rules_spec.rb`, `spec/commands/home_visits/record_spec.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `HomeVisits::Rules.check(input, household:, family:, patient:, today:) -> [] | [{ error:, path: }]` (pura): data (`visit_date_out_of_range`), imóvel e microárea (`property_type` proibido com domicílio), pessoa (`non_residential_with_person`, `patient_in_other_household`, `family_without_responsible`), desfecho × motivos × medidas (`reasons_forbidden`, `reason_not_allowed` com `path: "reasons[i]"`, `measurements_forbidden`), medidas pelas faixas do YAML (`invalid_home_visit` com `path: "measurements.<chave>"`; par sistólica/diastólica e glicemia/tipo), `note` até 500 code points depois do `strip`; `shift(occurred_at)`.
- `HomeVisits::Record.call(params:, by:, origin: "dashboard", device: nil) -> Result.ok(visit:, day:, replayed: bool) | fail(...)`:
  1. `client_uuid` já gravado → `ok(replayed: true)` com a gravada (idempotência do dashboard; o envio do app usa o livro, Task 20);
  2. acesso (`Access.registers?`) e escopo da microárea;
  3. transação: trava paciente (`FOR UPDATE`, quando há) → domicílio (`FOR SHARE`) → folha: `HomeVisitDay.lock.open_on(...)`; sem aberta, `INSERT … sequence = max + 1` (o único parcial resolve a corrida com retry); com 99 válidas, fecha a cheia (`CloseDay.call(day, by: nil, closed_by: "full")`, ficha gerada) e abre a próxima (Desvio 2); dia já fechado → próxima `sequence`;
  4. `People.resolve` (savepoint) quando `new_patient`; regras; grava (`has_reason` pelo api; `patient_id` do responsável em `family_responsible`); `home_visit.recorded` com `home_visit_id`, `home_visit_day_id`, `user_id`.
- `cbo` da folha nova = `member.cbo_code`.

- [ ] **Step 1: Write the failing tests.** `rules_spec.rb` (`type: :model`), um exemplo por regra: "turno 11:59 → 1, 12:00 → 2, 18:00 → 3, no fuso da cidade"; "data futura e data além do prazo do SIAPS → visit_date_out_of_range"; "com domicílio, mandar property_type → 422 path property_type"; "não residencial com patient_id → non_residential_with_person"; "não residencial só aceita 1 27 28 31 34 35 36 37"; "masculino com motivo 5 → reason_not_allowed path reasons[0]"; "recusa com motivo → reasons_forbidden; ausente com peso → measurements_forbidden"; "sistólica sem diastólica → invalid_home_visit"; "peso 0.4 → invalid_home_visit path measurements.weight_kg"; "observação com 501 code points → recusada, nunca cortada"; "paciente com cadastro ativo em outro domicílio → patient_in_other_household"; "família com responsável que saiu → family_without_responsible". `record_spec.rb` (`type: :model`): "a primeira visita do dia abre a folha 1 com o CBO do vínculo"; "o mesmo client_uuid devolve a gravada sem novo evento"; "a 100ª visita fecha a folha cheia (closed_by full, ficha gerada) e abre a 2"; "visita para dia de folha fechada abre a complementar"; "fora de área pelo ACS grava micro_area nula; pelo TACS → out_of_scope"; "registro rápido com duplicado → duplicate_suspected e nada gravado"; "`family_responsible` grava o responsável em patient_id"; **Review Focus 1**: duas threads gravando a 99ª e a 100ª ao mesmo tempo (`lock_wait`) → uma folha `full` com 99 e a outra visita na sequence 2; "o evento só tem os três ids".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/home_visits/rules_spec.rb spec/commands/home_visits/record_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** O fechamento cheio chama o `HomeVisits::CloseDay` da Task 10 e a ficha da Task 9: implemente as Tasks 9 e 10 antes do Step 4 desta (sem `skip` nem `pending` nos exemplos).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: record home visits with day sheets, complementary sheets and visit rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 8 · regras do §3 com códigos e paths; folha cheia fecha com `closed_by: full` (A-6); idempotência por client_uuid no dashboard."

---

### Task 9: Ficha LEDI de visita (tipo 8) — montagem campo a campo

**Files:**
- Create: `app/services/ledi/fichas/home_visit_day.rb`, `app/services/ledi/home_visit_day_ficha.rb`, `spec/services/ledi/fichas/home_visit_day_spec.rb`
- Modify: `app/services/ledi/ficha_sources.rb`, `app/models/ledi_generation_failure.rb`

**Interfaces:**
- `Ledi::Fichas::HomeVisitDay` (`Data`): `type = "visita_domiciliar"`, `competence = day.strftime("%Y%m")`, `cnes`, `ine`, `source = { type: "HomeVisitDay", id: }`, `to_thrift(uuid:)` → `FichaVisitaDomiciliarMasterThrift` com `tpCdsOrigem: 3`, `headerTransport` = `Ledi::Fichas::RegistrationHeader` (`saved_at = day.in_time_zone.beginning_of_day`, W-10), `visitasDomiciliares` na ordem de `occurred_at`; cada filho pela tabela do contrato §10 (documento por `Ledi::Fichas::CitizenDocument.for(patient)`; `dtNascimento` e `sexo` só com pessoa; `microArea` nulo fora de área; `tipoDeImovel` inteiro; `uuidOrigemCadastroDomiciliar` por `Ledi::HouseholdRegistrationFicha.originating_uuid(versão atual)`, ausente sem ficha; nunca `numProntuario`, `latitude`, `longitude`, `ivcf`). `inspect`/`as_json` só tipo e `source_id`.
- `Ledi::HomeVisitDayFicha` (`extend` do comportamento do `Ledi::RegistrationFicha`: `generate(day)`, `build(day)`, `record_failure!`, `retry!`, `regenerate`), `SOURCE_TYPE = "HomeVisitDay"`, `version_class = HomeVisitDay`; `build` devolve os motivos de "não gerada" de `HomeVisits::Catalog::NOT_GENERATED` (`siaps_deadline_passed` quando `Ledi::Deadline.on(competence) < today`); `generate` devolve `:empty` sem visita válida (nada gravado).
- `Ledi::FichaSources::REGISTRY` + `"HomeVisitDay" => "Ledi::HomeVisitDayFicha"`; `LediGenerationFailure::REASONS` + os códigos novos dos dois módulos (Desvio 9).

- [ ] **Step 1: Write the failing test** `home_visit_day_spec.rb` (`type: :model`): "folha com três visitas (morador com CPF, família com responsável sem CPF com justificativa e CNS, imóvel comercial sem pessoa) → ficha igual, campo a campo, ao exemplo de referência" (o exemplo mora no spec como hash esperado, com os valores da task 1); "CPF e CNS nunca juntos"; "fora de área: microArea nulo e stForaArea true"; "uuidOrigemCadastroDomiciliar presente com ficha do domicílio aceita e ausente sem ela"; "visita retirada fica fora"; "não gerada: profissional sem CNS, unidade sem CNES, paciente sem identificador (com o visit_id em blockers), prazo vencido"; "folha só com retiradas → :empty"; "thrift serializa e volta (roundtrip)"; "`inspect` não mostra CPF".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/ledi/fichas/home_visit_day_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/services/ledi/registration_ficha_spec.rb spec/services/ledi/thrift_roundtrip_spec.rb spec/requests/production_generation_failures_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: build LEDI home visit ficha from a day sheet" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 9 · ficha 8 campo a campo; não gerada com os 8 motivos do §10 (A-4); header 00h local (W-10)."

---

### Task 10: Fechar o dia, retirar a visita, situação da ficha e "Reenviar"

**Files:**
- Create: `app/commands/home_visits/{close_day,withdraw}.rb`, `app/services/home_visits/blockers.rb`, `spec/commands/home_visits/{close_day,withdraw}_spec.rb`
- Modify: `app/services/home_visits/json.rb`

**Interfaces:**
- `HomeVisits::CloseDay.call(day, by:, closed_by:, expected_state: "open") -> Result.ok(day:) | fail(:day_changed)` — trava a folha; `open` → `closed`; gera a ficha na transação (`Ledi::HomeVisitDayFicha.generate`); `home_visit_day.closed` (`home_visit_day_id`, `closed_by`).
- `HomeVisits::Withdraw.call(visit, by:, expected_state:, reason:) -> ok(visit:) | fail(:visit_changed | :day_closed | :invalid_request, details: { path: "reason" })` — trava folha → visita; só o dono; `home_visit.withdrawn`.
- `HomeVisits::Blockers.for(day) -> [{ code, visit_id }]` (os mesmos códigos do `build`, sem gravar).
- `HomeVisits::Json.day(day)` com `ficha: { status, reasons }`: `open` | `empty` | `not_generated` (com os motivos) | `not_exported` (Desvio 3) | o status do `ledi_outbox` mais recente da fonte. "Reenviar" e "Gerar de novo" usam o `ProductionController` existente (despacho por `FichaSources`).

- [ ] **Step 1: Write the failing tests.** `close_day_spec.rb`: "o dono fecha com expected_state open → ficha pending no outbox, um evento"; "segunda vez → day_changed"; "prazo vencido fecha com não gerada siaps_deadline_passed, nunca descarta"; "exportação desligada → not_exported"; "ficha recusada: regenerate da folha cria uuid novo com replaces_outbox_id e a folha não muda"; "retry! de siaps_deadline_passed segue não gerada (W-11)". `withdraw_spec.rb`: "retira com motivo de 10–500 → withdrawn e evento só com ids"; "já retirada → visit_changed"; "folha fechada → day_closed"; "outro usuário → not_found"; **Review Focus 2**: retirada e fechamento simultâneos (`lock_wait`) → ou a ficha não tem a visita, ou 409 day_closed.
- [ ] **Step 2:** `bin/test api -w mod24b spec/commands/home_visits/close_day_spec.rb spec/commands/home_visits/withdraw_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/commands/home_visits/record_spec.rb spec/services/ledi/fichas/home_visit_day_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: close home visit day sheets and withdraw visits with ficha status" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 10 · fechar/retirar com 409 do §2; `ficha.status` ganha `not_exported` (A-5); Reenviar pelo /production existente."

---

### Task 11: Job de fechamento das folhas

**Files:**
- Create: `app/jobs/home_visits/close_days_job.rb`, `spec/jobs/home_visits/close_days_job_spec.rb`
- Modify: `config/recurring.yml`

**Interfaces:**
- `HomeVisits::CloseDaysJob < ApplicationJob`, `prepend EachCityJob`, `queue_as :housekeeping`; fecha toda folha `open` com `day < Time.zone.today` por `CloseDay.call(day, by: nil, closed_by: "job")`, uma transação por folha; **não** consulta o interruptor (decisão 22); log só com contagens e ids; falha de uma folha não para as outras e o job falha no fim com a contagem.
- `config/recurring.yml`: `home_visits_close_days: { class: HomeVisits::CloseDaysJob, queue: housekeeping, schedule: "every day at 3:20am America/Sao_Paulo" }` com comentário "Módulo 24b (ADR 0052; contrato §2): fecha as folhas de dias anteriores, com home_visits ligado ou não".

- [ ] **Step 1: Write the failing test** (`type: :job`): "fecha a folha de ontem e deixa a de hoje"; "roda com home_visits desligado"; "rodar duas vezes fecha uma vez e gera uma ficha"; "a folha que levanta não impede as outras"; "o log não tem nome, motivo nem medida"; "não atribui Current.city".
- [ ] **Step 2:** `bin/test api -w mod24b spec/jobs/home_visits/close_days_job_spec.rb spec/architecture/current_city_assignment_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: close past home visit day sheets every night" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 11 · CloseDaysJob às 3h20 (C16), sem portão, idempotente."

---

### Task 12: Rotas da equipe — folhas e visitas (§6)

**Files:**
- Create: `app/controllers/home_visits/{days,visits}_controller.rb`, `spec/requests/home_visits/{days,visits}_spec.rb`
- Modify: `config/routes.rb`, `app/services/home_visits/{json,trail}.rb`

**Interfaces:**
- As rotas e formas do contrato §6 (`days/today`, `days`, `days/:id`, `days/:id/close`, `POST /home_visits`, `GET /home_visits/:id`, `withdraw`, `GET /home_visits`, `count`), com: `note` só no `GET /:id`; `withdrawn.reason` só para o dono e o leitor da equipe, `null` para os outros (Q17); `<home_visit_day>.own` e `<home_visit>.can_withdraw` (Q14); listas `{ items, next_before }` com cursor `CareLines::Cursor`; `HomeVisits::Trail.visits_viewed!(user:, visit_ids:, patient_ids:)` = `clinical_record.viewed` com `access: "home_visits"` (W-6), uma por resposta nominal, nenhuma em `count` e em lista de folhas; `POST /home_visits` → 201 (200 no replay).
- Registro de outro escopo por id → 404; lista de microárea/equipe alheia → 403 `out_of_scope`; filtros repetidos com `[]`.

- [ ] **Step 1: Write the failing tests** (`type: :request`): "ACS registra pelo celular e recebe visit + day (201); reenvio do mesmo client_uuid → 200 igual"; "days/today com visita a pessoa grava trilha; sem pessoa, não"; "enfermeiro lista as folhas da equipe sem pessoa e sem trilha"; "GET /home_visits/:id mostra a observação ao dono e grava trilha"; "visita de outra equipe por id → 404; lista de outra microárea → 403 out_of_scope"; "count devolve done/refused/absent/withdrawn sem trilha"; "filtro outcome[]=2 funciona e outcome=2 sem [] → 422"; "blockers aparecem na folha aberta (profissional sem CNS)"; "close pelo dono → 200 com ficha pending"; "nome ou CPF na query → 422 invalid_request".
- [ ] **Step 2:** `bin/test api -w mod24b spec/requests/home_visits/days_spec.rb spec/requests/home_visits/visits_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add home visit day and team visit routes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 12 · §6 completo (formas do contrato; ficha.status inclui not_exported); pronto para a folha do dia do dashboard."

---

### Task 13: Atividade coletiva — regras do §4, gravação e "por engano"

**Files:**
- Create: `app/services/collective_activities/rules.rb`, `app/commands/collective_activities/{record,mark_mistaken}.rb`, `spec/services/collective_activities/rules_spec.rb`, `spec/commands/collective_activities/{record,mark_mistaken}_spec.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `CollectiveActivities::Rules.check(input, today:) -> [] | [{ error: "invalid_collective_activity" | "activity_date_out_of_range" | "duplicate_participant", path: }]` dirigida por `collective_rules` do YAML (tabela por tipo; práticas 2/9/33 só no 6; SIGTAP 10 dígitos ⇔ prática 30; `smoking` só com 25–28; tema 18 recusado; contagens; local unidade ativa com CNES ou texto até 250).
- `CollectiveActivities::Record.call(params:, by:, origin:, device:) -> ok(activity:, replayed:) | fail(...)`: acesso pela Tabela 5; `professionals` com vínculo ativo em qualquer equipe e CNS válido (`professional_without_cns`), CBO do vínculo; corpo plano do contrato §7 (`place_health_unit_id` | `place_text`, exatamente um; listas ausentes = vazias; Q15); participantes por `patient_id` ou `HomeVisits::People.resolve` com o `duplicates_reviewed` do próprio participante (savepoint por participante; recusa desfaz tudo; Q15); grava; gera a ficha (Task 14) na transação; `collective_activity.recorded`.
- `MarkMistaken.call(activity, by:, expected_state:, reason:)` → `activity_changed`; não desfaz a ficha; `collective_activity.marked_mistaken`.

- [ ] **Step 1: Write the failing tests.** `rules_spec.rb`, um exemplo por linha da tabela do §4: "reunião (1) exige meeting_topics e recusa target_audience"; "reunião com tema 6 aceita health_topics"; "tipo 5 sem participantes → path participants"; "tipo 6 exige health_practices"; "prática 9 no tipo 5 → path health_practices[i]"; "prática 30 sem SIGTAP → path procedure_code; SIGTAP sem prática 30 → recusado"; "smoking sem prática 25–28 → path participants[0].smoking"; "tema 18 → path health_topics[i]"; "altered_count diferente das avaliações alteradas da lista → recusado"; "unidade e texto juntos → path place"; "data além do prazo → activity_date_out_of_range". `record_spec.rb`: "grava tipo 5 com participante novo (registro rápido) e profissional de outra equipe"; "mesmo client_uuid → replay"; "CBO fora da Tabela 5 → cbo_not_allowed"; "participante repetido → duplicate_participant"; "profissional sem CNS → professional_without_cns". `mark_mistaken_spec.rb`: "quem registrou marca com motivo; outro → not_found; segunda vez → activity_changed; a ficha enfileirada fica".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/collective_activities/rules_spec.rb spec/commands/collective_activities/record_spec.rb spec/commands/collective_activities/mark_mistaken_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (com a Task 14 junto, se o executor preferir: o `Record` chama `Ledi::CollectiveActivityFicha.generate`).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: record collective activities with LEDI cross rules and mistaken marking" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 13 · regras do §4 em dado (YAML), códigos e paths; por engano não desfaz a ficha."

---

### Task 14: Ficha LEDI de atividade coletiva (tipo 6)

**Files:**
- Create: `app/services/ledi/fichas/collective_activity.rb`, `app/services/ledi/collective_activity_ficha.rb`, `spec/services/ledi/fichas/collective_activity_spec.rb`
- Modify: `app/services/ledi/ficha_sources.rb`

**Interfaces:**
- `Ledi::Fichas::CollectiveActivity` pela tabela do §11: cabeçalho de quem registrou (`RegistrationHeader`, `held_on` 00h local), `outraLocalidade` xor `cnesLocalAtividade`, nunca `inep`, `pseEducacao`/`pseSaude` sempre `false`, `tbCdsOrigem: 3`, profissionais com `cnsProfissional`/`codigoCbo2002`, participantes com CPF ou (`stCidadaoNaoPossuiCpf`, justificativa, CNS); fora dos tipos 5/6, participante sem CPF e sem CNS fica fora da lista e conta em `numParticipantes`.
- `Ledi::CollectiveActivityFicha` (`SOURCE_TYPE = "CollectiveActivity"`, mesmo contrato); "não gerada" com `CollectiveActivities::Catalog::NOT_GENERATED`; `FichaSources::REGISTRY` + `"CollectiveActivity" => "Ledi::CollectiveActivityFicha"`.

- [ ] **Step 1: Write the failing test** (`type: :model`): "tipo 6 com 3 participantes e 2 profissionais → igual ao exemplo de referência, campo a campo"; "CPF antes do CNS"; "PSE sempre false e sempre enviado"; "tipo 4 com participante sem documento: fora da lista, numParticipantes intacto"; "tipo 5 com participante sem documento → não gerada participant_without_identifier"; "local unidade sem CNES → place_without_cnes"; "tipo 5 com participante sem sexo → não gerada participant_without_sex" (Q4); "profissional participante sem CNS → participant_professional_without_cns"; "roundtrip thrift".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/ledi/fichas/collective_activity_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/commands/collective_activities/record_spec.rb spec/services/ledi/thrift_roundtrip_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: build LEDI collective activity ficha" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 14 · ficha 6 campo a campo; 9 motivos de não gerada do §11."

---

### Task 15: Rotas da atividade coletiva (§7)

**Files:**
- Create: `app/controllers/collective_activities/{options,activities}_controller.rb`, `app/services/collective_activities/{json,trail}.rb`, `spec/requests/collective_activities/activities_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- As rotas e formas do contrato §7; `options.domains` = o bloco `collective_activity` do YAML (domínios com rótulos e `rules`), o mesmo dado do `domains-v1.json`; lista sem participantes nem temas, sem trilha, `mistaken` fora por padrão (`state[]=mistaken` mostra); detalhe com trilha `access: "collective_activities"` e os pacientes.

- [ ] **Step 1: Write the failing test** (`type: :request`): "options traz equipes com can_register, unidades com has_cnes, profissionais com has_cns e domains com as regras"; "POST tipo 1 sem participantes → 201 com ficha pending"; "POST tipo 5 com new_patient duplicado → 409 duplicate_suspected com candidates"; "lista sem participantes e sem trilha; detalhe com trilha"; "mark_mistaken some da lista e aparece com state[]=mistaken"; "atividade de outra equipe por id → 404"; "interruptor desligado → 403 feature_disabled".
- [ ] **Step 2:** `bin/test api -w mod24b spec/requests/collective_activities/activities_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add collective activity routes with options and domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 15 · §7 completo; `options.domains` sai do YAML da task 1 (o dashboard não copia regras)."

---

### Task 16: Linhas de cuidado — `acs_visit`, vitais da visita e `min_gap_days` (§12)

**Files:**
- Create: `spec/services/care_lines/home_visit_source_spec.rb`
- Modify: `app/services/care_lines/{catalog,sources,rules,evaluate,evaluation,preview,citizen_view,board}.rb`, `app/controllers/care_lines/base_controller.rb`, `app/controllers/citizen_api/care_controller.rb`, `config/care_lines/catalog-base.yml`, `spec/services/care_lines/{rules,preview}_spec.rb`

**Interfaces:**
- `CareLines::Catalog.untracked?(source, vaccination: false, home_visits: false)`: `acs_visit` não rastreada só sem `home_visits`. `home_visits:` atravessa `Evaluate.{call,sources_of,since,evaluate}`, `Evaluation.{enrollments,items}`, `Preview.call`, `CitizenView.call`, `Board.{page,overdue_count}` e `Sources.load(…, home_visits: false)`; `CareLines::BaseController#home_visits_usable? = HomeVisits::Gate.usable?(Current.city)` ao lado de `vaccination_usable?`, e `CitizenApi::CareController` lê o mesmo (fora de transação; W-5).
- `CareLines::Sources.home_visits!(result, ids, from, sources)`: **uma** consulta (`home_visits` JOIN `home_visit_days`, `withdrawn_at IS NULL`, `outcome = 1`, `property_type IN RESIDENTIAL`, `patient_id IN ids`, `occurred_at >= from`); `acs_visit` com `has_reason AND cbo IN CARE_LINE_CBOS`; `blood_pressure` com sistólica e diastólica, `weight_height` com peso e altura (qualquer CBO da Tabela 7; decifra só `measurements` — nunca motivo nem nota). Só com `home_visits: true` (Desvio 7).
- `Rules::PRACTICE_KEYS` + `min_gap_days` (`int(…, 1..365)`, opcional); `Evaluate.dates` aplica o intervalo depois do filtro: em ordem, entra a data que dista ≥ `min_gap_days` da última que entrou.
- `catalog-base.yml`: `min_gap_days: 30` na `c5_d`; `citizen_label: "Visitas do agente de saúde"` nas seis práticas `acs_visit`. A versão nova das cinco linhas vai pelo maintenance (runbook), sem `Platform.audit` novo.

- [ ] **Step 1: Write the failing tests.** `home_visit_source_spec.rb` (`type: :model`): "sem home_visits a prática acs_visit é untracked"; "com home_visits conta só a realizada, residencial, com motivo, por 515105 ou 322255, não retirada"; "visita de ACE com pressão conta em blood_pressure e não em acs_visit"; "recusa e ausência não contam"; "retirada some na hora (decisão 23)"; "uma consulta SQL para mil pacientes (contagem de queries)"; "min_gap_days 30: visitas a 10 e 45 dias contam 2, a 10 e 20 contam 1". `rules_spec.rb`: "min_gap_days fora de 1..365 → Invalid"; `preview_spec.rb`: "acs_visit sai de excluded com home_visits"; `catalog-base`: "c5_d tem min_gap_days 30 e as seis acs_visit têm citizen_label".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/care_lines/home_visit_source_spec.rb spec/services/care_lines/rules_spec.rb spec/services/care_lines/preview_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/care_lines spec/requests/care_lines spec/requests/citizen_api/care_spec.rb spec/services/care_lines/templates/save_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: track ACS visits, visit vitals and minimum gap in care lines" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 16 · acs_visit rastreada com home_visits; vitais da visita só com o interruptor (A-12); min_gap_days 1..365; catálogo-base com citizen_label — passo do runbook para publicar as 5 versões."

---

**Q25 (decisão do usuário):** `CareLines::CitizenView` acrescenta a cada item `by_team`, `true` só quando a fonte da prática é `acs_visit`, sem expor o código da fonte. Spec em `spec/requests/citizen_api/care_spec.rb`: com `home_visits` utilizável, o item da visita vem `by_team: true` e os outros `false`; nenhum item traz `source` nem nome de profissional.

### Task 17: Linha do tempo — `home_visit` e `collective_activity` (§12)

**Files:**
- Create: `app/services/timeline/sources/{home_visit_source,collective_activity_source}.rb`, `spec/services/timeline/sources/{home_visit_source,collective_activity_source}_spec.rb`
- Modify: `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/controllers/pec_timelines_controller.rb`, `spec/services/timeline/query_spec.rb`, `spec/requests/pec_timeline_spec.rb`

**Interfaces:**
- `Timeline::KINDS` + `home_visit collective_activity`; em `Timeline::Query` (W-1): `SOURCES` + as duas fontes, `PROFESSIONAL_ONLY_KINDS` + `collective_activity`, `CITIZEN_FILTERED_KINDS` + `home_visit`; `PecTimelinesController::KINDS = (Timeline::KINDS - %w[transport home_visit collective_activity])`.
- `HomeVisitSource.entries(person:, before:, limit:, from:, to:, citizen: false)`: profissional = todos os desfechos e retiradas (título "Visita domiciliar" / "Visita ao imóvel", resumo com desfecho, rótulos dos motivos e medidas, `professional` = dono da folha, `status` `recorded|withdrawn`, `detail_ref { type: "home_visit", id }`); `citizen: true` = só `outcome = 1` não retirada, exatamente `{ id, kind: "home_visit", occurred_at, unit: null, professional_name: null, title: "Visita do agente de saúde", summary: null, status: null, document: null }`, com `occurred_at` = meio-dia local do dia da visita (Q22). `home_visit` fora de `DECLARED_KINDS` (Q23). Nunca a observação. Não consulta o interruptor (registro retido, como a vacina).
- `CollectiveActivitySource`: por participação; título = rótulo do tipo; resumo "Participou" (+ "avaliação alterada"); `mistaken` fora.

- [ ] **Step 1: Write the failing tests:** "profissional vê recusa, ausência e retirada com motivos e medidas, nunca a observação"; "cidadão vê só a realizada, sem nome, unidade, resumo nem status"; "visita à família aparece para o responsável"; "atividade por engano some"; "collective_activity nunca na visão do cidadão"; `query_spec`: as listas novas; `pec_timeline_spec`: "o PEC externo não recebe home_visit nem collective_activity"; `GET /citizen/history` **sem** `kinds[]` traz a entrada do cidadão, igual à forma do contrato chave por chave, com `occurred_at` ao meio-dia local (Q22); "par só declarado não vê a visita" (Q23); "com home_visits desligado a visita realizada continua no histórico" (Q23).
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/timeline spec/requests/pec_timeline_spec.rb spec/requests/citizen_api/history_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/security/urgency_secrecy_spec.rb spec/invariants/psychosocial_secrecy_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: show home visits and collective activities in the patient timeline" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 17 · kinds novos; constantes em Timeline::Query (W-1); cidadão só a realizada, meio-dia local, sem `kinds[]` (Q22)."

---

### Task 18: "Quem visitar" por microárea (§8)

**Files:**
- Create: `app/services/home_visits/who_to_visit.rb`, `app/controllers/home_visits/who_to_visit_controller.rb`, `spec/services/home_visits/who_to_visit_spec.rb`, `spec/requests/home_visits/who_to_visit_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `HomeVisits::WhoToVisit.page(micro_area:, before:, limit:, today:, care_lines:, home_visits:) -> { items:, next_before: }` e `.count(...)`: população = cadastro individual ativo com domicílio na microárea, sem óbito; `care_line` pelas inscrições ativas e `Evaluation.enrollments(…, home_visits: true)` (só com `care_lines` utilizável; só nome da linha e rótulo da prática); `registration_due` por `TerritoryRegistry::Validity` (`due_soon`/`expired`); `never_visited` sem visita realizada válida; ordem `display_name, patient_id`; cursor opaco do `person_ref`; `limit` 1–100 (padrão 50). Reusado pelo download (Task 22) com `limit: nil`.
- Rotas do contrato §8; trilha `clinical_record.viewed` `access: "home_visits"` com os ids mostrados; `count` sem trilha.

- [ ] **Step 1: Write the failing tests:** "gestante com c3_e vencida entra com care_line, nome da linha e prática, sem CID"; "cadastro a 50 dias de vencer entra com registration_due"; "quem tem visita realizada não entra por never_visited; quem só tem recusa entra"; "sem motivo, fora"; "óbito fora"; "care_lines desligado: só as duas outras fontes"; "ACS de outra microárea → 403 out_of_scope; enfermeiro lê"; "trilha com os patient_ids da página; count sem trilha"; "limit 101 → 422 invalid_limit".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/home_visits/who_to_visit_spec.rb spec/requests/home_visits/who_to_visit_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add who to visit list per micro area" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 18 · quem visitar com 3 fontes, cursor opaco e trilha; contagem sem trilha."

---

### Task 19: Aparelho de campo, autenticação do app e `GET /field/v1/session` (§9.1, §9.2)

**Files:**
- Create: `app/services/field_sync/{token,schemas,json}.rb`, `app/commands/field_devices/{register,revoke}.rb`, `app/controllers/field_devices_controller.rb`, `app/controllers/field_devices/admin_controller.rb`, `app/controllers/concerns/field_authentication.rb`, `app/controllers/field_sync/v1/{base,sessions}_controller.rb`, `config/field_sync/*.json`, `spec/services/field_sync/schemas_spec.rb`, `spec/requests/field_devices_spec.rb`, `spec/requests/field_sync/session_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `FieldSync::Token.generate -> "rsf_" + Base64.urlsafe_encode64(SecureRandom.bytes(32), padding: false)`; `qr_payload(host:, token:)`.
- `FieldDevices::Register.call(user:, name:) -> ok(device:, token:)` — trava a linha do usuário; `device_owner?` (senão `cbo_not_allowed`); 3 ativos → `device_limit`; `field_device.registered`. `FieldDevices::Revoke.call(device, by:, reason:)` → `field_device.revoked` (`field_device_id`, `user_id`, `reason`).
- Rotas do §9.1: `POST /field_devices` com `require_step_up!`, resposta com `token` e `qr_payload` uma vez (`Cache-Control: no-store`); `POST /field_devices/:id/revoke` sem step-up (só o dono, 404 para os outros); `/field_devices/admin` para `municipal_admin`, revogar com step-up. As duas listas trazem `status: active | inactive | revoked` (`inactive` calculado na leitura; Q7); revogar já revogado → `409 device_already_revoked` (Q21).
- `FieldAuthentication` (concern, no padrão do `Hl7::BaseController`): `rate_limit to: 120, within: 1.minute, name: "field_sync", by: digest do Bearer ou IP, with: 429 too_many_requests` → `require_home_visits!` → `authenticate_device!` (ausente/desconhecido → 401 `invalid_token`; revogado → 401 `device_revoked`; usuário inativo, sem `health_professional` ou sem vínculo da Tabela 7 → revoga com `user_disabled`/`membership_ended` e 401 `device_revoked`; sempre `WWW-Authenticate: Bearer`) → `last_used_at` no máximo uma vez por minuto; JSON obrigatório nas escritas (415), corpo até 2 MB (413 `payload_too_large`). Nunca `Current.session`: `@device`/`@device_user` passados como `by:` (W-8).
- `FieldSync::Schemas`: carrega os cinco JSON de `config/field_sync/` com `SHA256` fixo; `validate!(name, payload)` em spec.
- `GET /field/v1/session` na forma do §9.3, sem trilha.

- [ ] **Step 1: Write the failing tests.** `schemas_spec.rb`: "a cópia bate com o SHA-256 da tag field-sync-v1.0.0"; "os exemplos do manifest do contracts validam" (cópia em `spec/fixtures/field_sync/examples/`). `field_devices_spec.rb` (`type: :request`): "ACS cadastra com step-up e recebe token rsf_ e qr_payload uma vez; o banco só tem o digest"; "sem step-up → 401 mfa_required"; "enfermeiro → 403 cbo_not_allowed"; "4º ativo → 422 device_limit"; "lista sem token"; "revoga o próprio; o do colega → 404"; "admin lista a cidade e revoga com step-up". `session_spec.rb`: "Bearer válido → sessão com microáreas do escopo, validada contra session-response-v1"; "sem Bearer → 401 invalid_token com WWW-Authenticate"; "revogado → 401 device_revoked"; "vínculo encerrado → revoga membership_ended e 401"; "usuário desativado → user_disabled"; "home_visits desligado → 403 feature_disabled"; "121ª requisição → 429"; "token na query nunca é aceito e o log não tem o token".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/field_sync/schemas_spec.rb spec/requests/field_devices_spec.rb spec/requests/field_sync/session_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Copiar os schemas do `rotasaude/contracts` (tag `field-sync-v1.0.0`; se a tag ainda não saiu, do commit dos schemas, e o SHA é atualizado na Task 26 depois da tag).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add field devices with bearer tokens and field sync session" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 19 · aparelho com step-up, token só SHA-256, revogação na próxima requisição (A-7); /field/v1/session validada contra o schema."

---

### Task 20: Envio do app — `POST /field/v1/uploads` (§9.3)

**Files:**
- Create: `app/services/field_sync/upload.rb`, `app/controllers/field_sync/v1/uploads_controller.rb`, `spec/services/field_sync/upload_spec.rb`, `spec/requests/field_sync/uploads_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `FieldSync::Upload.call(items:, device:, user:) -> [result]`: envelope (1–50, `client_uuid` uuid, `type` em `ITEM_TYPES`, `body` objeto) → senão `422 invalid_request` com `path`; cada item em ordem, na sua transação:
  - livro: `client_uuid` em `field_sync_receipts` com o mesmo `type` → `duplicate` com o `id`; outro `type` → `rejected client_uuid_conflict`;
  - `home_visit` → `HomeVisits::Record` (`origin: "field_app"`, `device:`), `patient_client_uuid` resolvido pelo livro (`item_type = "patient"`) e `household_client_uuid` pelo livro (`item_type = "household"`), também no `individual_registration`; citação de item não aceito → `rejected not_found` (Q27); `home_visit_withdrawal` → `HomeVisits::Withdraw` por `visit_client_uuid` ou `visit_id`; `collective_activity` → `CollectiveActivities::Record` (exige `collective_activities` utilizável, lido uma vez por lote fora da transação; senão `rejected feature_disabled`); `patient` → `HomeVisits::People.resolve`; `household` e `individual_registration` → `TerritoryRegistry::SaveHousehold`/`SaveRegistration` com `by: user` (exigem `territory_registry` utilizável; `stale_version` com `current_version` e `current` na forma do 24a; CBO de campo que não é ACS → `rejected cbo_not_allowed`, W-9);
  - aceito → linha no livro na mesma transação; recusado e `stale_version` não vão ao livro.
- Resposta 200 `{ results }`, validada contra `upload-response-v1.json`.

- [ ] **Step 1: Write the failing tests.** `upload_spec.rb` (`type: :model`): "lote com paciente novo e visita que o cita por patient_client_uuid → dois accepted"; "item recusado no meio não derruba os outros"; **Review Focus 3**: "reenvio do lote inteiro → duplicate com os mesmos ids, nenhuma linha nova, nenhum evento novo"; "mesmo client_uuid com outro type → client_uuid_conflict"; "recusado reenviado corrigido → accepted"; "visita de dia já fechado entra na complementar"; "household com version vencida → stale_version com current"; "TACS mandando household → rejected cbo_not_allowed"; "collective_activities desligado → item rejected feature_disabled". `uploads_spec.rb` (`type: :request`): "resposta valida contra upload-response-v1"; "51 itens → 422 invalid_request path items"; "corpo de 2 MB+1 → 413 payload_too_large"; "Content-Type texto → 415"; "o log não tem motivo, nome, CPF nem token".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/field_sync/upload_spec.rb spec/requests/field_sync/uploads_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/territory_registry/households_spec.rb spec/requests/territory_registry/registrations_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: accept idempotent field sync uploads with per-item results" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 20 · envio idempotente pelo livro; item recusado não derruba o lote; stale_version com current; cadastro só pelo ACS (W-9)."

---

### Task 21: Tombstones nos comandos do 24a (transversal: 24a)

**Files:**
- Create: `app/commands/field_sync/tombstone.rb`, `spec/commands/field_sync/tombstone_spec.rb`
- Modify: `app/commands/territory_registry/{save_household,save_registration}.rb`

**Interfaces:**
- `FieldSync::Tombstone.moved!(entity_type:, entity_id:, micro_area_id:)` (insert só acréscimo, na transação de quem chama). `SaveHousehold`: na troca de microárea, tombstone do domicílio na microárea de origem (e os cadastros movidos pelo `move_registrations!`); `SaveRegistration`: cadastro que sai da microárea (troca de domicílio para outra microárea, ou `micro_area_id` diferente) → tombstone na de origem. Saída do cadastro e domicílio inativo não geram tombstone (Desvio 6).

- [ ] **Step 1: Write the failing test** (`type: :model`): "domicílio movido de 01 para 02 grava tombstone household em 01 e registration dos moradores em 01"; "cadastro que muda para domicílio de outra microárea grava tombstone em 01"; "mudança dentro da microárea não grava"; "saída por mudança ou óbito não grava"; "recusa do SaveHousehold desfaz o tombstone".
- [ ] **Step 2:** `bin/test api -w mod24b spec/commands/field_sync/tombstone_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/requests/territory_registry spec/commands/territory_registry` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: record field sync tombstones when registry entries leave a micro area" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 21 · tombstones `moved` no SaveHousehold/SaveRegistration; `erased`/`out_of_scope` sem produtor (A-9, A-10, A-11)."

---

### Task 22: Download incremental — `GET /field/v1/changes` (§9.3)

**Files:**
- Create: `app/services/field_sync/{scope,cursor,changes}.rb`, `app/controllers/field_sync/v1/changes_controller.rb`, `spec/services/field_sync/changes_spec.rb`, `spec/requests/field_sync/changes_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `FieldSync::Scope.for(user) -> { micro_area_ids:, digest: }` (microáreas em que é ACS responsável ou atribuídas e ativas; `digest = SHA-256` dos ids ordenados, sem dado pessoal).
- `FieldSync::Cursor.{encode,decode}` = base64url de `{ v: 1, scope_digest, since, after_id }`; inválido → `422 invalid_cursor`.
- `FieldSync::Changes.page(user:, device:, cursor:, limit:) -> page`: sem cursor ou `scope_digest` diferente → carga completa (`reset: true` quando havia cursor); `UNION` de microáreas, domicílios (com famílias e moradia, forma do 24a) e cadastros do escopo por (`updated_at`, `id`) até `agora − 2 s`; `removals` de `field_sync_tombstones` das microáreas do escopo depois de `since`; `who_to_visit` (Task 18) só na última página; `next_cursor`, `has_more`; `limit` 1–500 (padrão 200, `invalid_limit`). Trilha por página com pessoa: `clinical_record.viewed` `access: "field_sync"`, `user_id`, `field_device_id`, `household_ids`, `patient_ids`.

- [ ] **Step 1: Write the failing tests.** `changes_spec.rb` (`type: :model`): "carga completa do ACS traz só as suas microáreas, em páginas, sem repetir nem pular (empate de updated_at)"; "cursor seguinte traz só o que mudou e as removals"; **Review Focus 4**: "atribuição encerrada entre duas páginas → reset: true e a carga recomeça só com o escopo novo"; "who_to_visit só na última página"; "trilha por página com exatamente os ids dela; página só de microáreas não grava". `changes_spec.rb` (`type: :request`): "resposta valida contra changes-response-v1"; "cursor adulterado → 422 invalid_cursor"; "limit 501 → 422 invalid_limit"; "o cursor decodificado não tem nome, CPF nem endereço".
- [ ] **Step 2:** `bin/test api -w mod24b spec/services/field_sync/changes_spec.rb spec/requests/field_sync/changes_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/commands/field_sync/tombstone_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: serve incremental field sync downloads by device scope" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 22 · download por cursor opaco com reset por scope_digest, removals e trilha por página; o contrato de sincronização está todo testado sem o app."

---

### Task 23: Retenção LGPD (§13)

**Files:**
- Create: `spec/commands/citizens/home_visit_erasure_spec.rb`
- Modify: `app/commands/citizens/request_erasure.rb`

**Interfaces:**
- `examined_patients?` + `home_visited?(ids)` (`HomeVisit.where(patient_id: ids).exists?`, retiradas inclusive) e `collectively_attended?(ids)` (`CollectiveActivityParticipant.where(patient_id: ids).exists?`, por engano inclusive), com o comentário "ADR 0052: visita e participação são registro retido (fichas enfileiradas), como o cadastro do 24a (L13)".

- [ ] **Step 1: Write the failing test** (`type: :model`): "paciente com visita retirada → pedido nasce retained"; "participante de atividade por engano → retained"; "visita a imóvel sem pessoa não retém ninguém"; "paciente com cadastro individual já era retido (nenhum tombstone erased nasce, A-10)".
- [ ] **Step 2:** `bin/test api -w mod24b spec/commands/citizens/home_visit_erasure_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/commands/citizens/request_erasure_spec.rb spec/commands/citizens/territory_registry_erasure_spec.rb spec/commands/citizens/erase_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: retain citizens with home visits or collective activity participation" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 23 · visita e participação retêm o paciente (ADR 0026)."

---

### Task 24: Semente de dev realista

**Files:**
- Create: `lib/home_visit_crew.rb`, `spec/lib/home_visit_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- `HomeVisitCrew.seed!(city)` (Curitiba, idempotente, só em development): liga os dois interruptores; um TACS com microárea atribuída pela enfermeira da equipe; folha de ontem fechada (visitas a morador, à família e a imóvel comercial; uma recusa; uma retirada) e folha de hoje aberta; uma atividade tipo 5 com 3 participantes e uma tipo 1; um aparelho revogado (sem token em claro). CPF/CNS com dígito válido, CBO real, vínculos coerentes, pelos comandos (`now:` para escalonar).

- [ ] **Step 1: Write the failing test** (`type: :model`): "semear duas vezes não duplica"; "a folha de ontem tem ficha"; "nenhum token em claro no banco".
- [ ] **Step 2:** `bin/test api -w mod24b spec/lib/home_visit_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: seed realistic dev home visits and collective activities" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 24 · semente: TACS atribuído, folhas de ontem e hoje, atividades 5 e 1, aparelho revogado."

---

### Task 25: Sigilo, eventos e R18

**Files:**
- Create: `spec/security/home_visit_secrecy_spec.rb`
- Modify: `spec/initializers/domain_events_bindings_spec.rb` (conferência final)

- [ ] **Step 1: Write the test** `spec/security/home_visit_secrecy_spec.rb` (`type: :request`), cenário completo (atribuição, visita com motivos, medidas e observação, registro rápido com CPF, retirada, fechamento, atividade tipo 6 com SIGTAP e participante, por engano, aparelho, envio e download):
  - os nomes de `DomainEvent` com prefixo `home_visit`, `home_visit_day`, `collective_activity`, `field_device`, `micro_area.field_assignment_changed` publicados ⊂ a lista do contrato §1, e uma varredura (Prism, padrão de `spec/events/platform_event_payload_guard_spec.rb`) de `app/**/*.rb` acha `DomainEvents.publish(` só com esses nomes;
  - todo payload tem só as chaves do contrato §1, e nenhum contém motivo, medida, observação, tema, prática, procedimento, nome, CPF, CNS ou token;
  - `capture_log` de todas as requisições não contém o token, o CPF, o nome, a observação, os motivos nem as medidas, nem o `Authorization`;
  - toda recusa 4xx forçada traz só `error`/`path`/`candidates`, sem o valor enviado;
  - nenhuma rota de `/home_visits`, `/collective_activities` ou `/field/v1` aceita pessoa ou texto na query (`?cpf=` → 422 `invalid_request`);
  - `GET /citizen/history` e `GET /citizen/care` não trazem recusa, ausência, retirada, motivo, medida, observação nem nome do profissional;
  - nenhum `Platform.audit(` novo nos arquivos do módulo (grep multilinha = 0) e `R18_PLATFORM_EVENT_NAMES` inalterado.
- [ ] **Step 2:** `bin/test api -w mod24b spec/security/home_visit_secrecy_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture` → PASS (falha → correção na task dona, volta aqui).
- [ ] **Step 3: Commit** `git commit -m "test: pin home visit and field sync secrecy for events, logs and citizen views" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24b · Task 25 · sigilo fixado (8 eventos, payload só ids, nenhum Platform.audit novo)."

---

### Task 26: Fechamento — SHA da tag, suíte completa e passagem aos fronts

**Files:** `app/services/field_sync/schemas.rb` (só se a tag mudou o SHA).

- [ ] **Step 1:** conferir a cópia de `config/field_sync/` contra a tag `field-sync-v1.0.0` (publicada pelo coordenador depois da Task 1); SHA diferente → atualizar `FieldSync::Schemas::SHA256`, `bin/test api -w mod24b spec/services/field_sync spec/requests/field_sync` → PASS e commit `git commit -m "chore: pin field sync schemas to field-sync-v1.0.0" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.
- [ ] **Step 2: Suíte completa** (transversal: três migrações de cidade, `CityEncryption`, `Platform::Features`, `Ledi`, `CareLines`, `Timeline`, `Citizens::RequestErasure`, comandos do 24a, `spec/support`, initializers): **pedir ao usuário** a rodada pela sessão "Testes", e enviar por `SendMessage` no formato `api · mod24b/feat/mod-24b-home-visits · --full · módulo 24b (3 migrações de cidade, CityEncryption, Platform::Features, Ledi, CareLines, Timeline, RequestErasure, 24a, spec/support, initializers)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador com o resumo e o hash, e a sessão do dashboard de que o api está pronto (porta 3057).
- [ ] **Step 3:** merge, push e cards só com autorização do usuário; F-24.x seguem In Progress até o último app de cada card. No fechamento, abrir as pendências do contrato §15 no board #2 (correção de ficha enviada, `min_gap_days` de C2/C3/C4/C6, PSE, IVCF-20, agendamento, latitude/longitude) e os gates de go-live (PEC real das fichas 8 e 6, duas fichas do mesmo profissional no mesmo dia, SIAPS contando, C5 contra o relatório).

**Relato ao coordenador:** "api 24b · Task 26 · schemas presos na tag; suíte completa <verde N/0 | falhas>; hash <…>; pronto para o dashboard, o wpda e o maintenance."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptores (2), CBOs e tipos de ficha (1), acesso (6), tabelas (3, 4, 5), migrações (3–5), mudanças em código existente (1, 9, 14, 16, 17, 23, 11), eventos (6, 8, 10, 13, 19; guarda 25); §2 estados (8, 10, 11, 13, 19); §3 regras da visita (8); §4 regras da atividade (13); §5 registro rápido e buscas (7); §6 rotas da visita (6, 12); §7 rotas da atividade (15); §8 quem visitar (18); §9 aparelho, autenticação, envio, download e cópia dos schemas (19–22, 26); §10 ficha 8 (9, 10); §11 ficha 6 (14); §12 linhas de cuidado e linha do tempo (16, 17); §13 cifra, imutabilidade, LGPD, sigilo, trilha, escopo (3–5, 23, 25); §15 suíte completa (26).
2. **Spec §6 "Testes" do api:** prova técnica (1); ficha de visita campo a campo, 99 e complementar, fechamento pelo ACS e pelo job, visita tardia (8–11); regras da visita (8); quem registra (6); atividade coletiva e ficha 6 (13, 14); não gerada e regerar (9, 10, 14); linhas de cuidado (16); linha do tempo (17); quem visitar (18); sincronização (19–22); sigilo, interruptores, R18, imutabilidade, LGPD (2–5, 23, 25).
3. **Nomes:** `HomeVisits::{Record,Withdraw,CloseDay,AssignMicroArea,EndAssignment}`, `HomeVisits::{Gate,Catalog,Access,Rules,People,Blockers,WhoToVisit,Json,Trail}`, `CollectiveActivities::{Record,MarkMistaken,Rules,Access}`, `Ledi::{HomeVisitDayFicha,CollectiveActivityFicha}`, `FieldSync::{Upload,Changes,Scope,Cursor,Token,Schemas,Tombstone}`, `FieldDevices::{Register,Revoke}` — mesma assinatura em todas as tasks.
4. **Ordem:** 900002 e 900003 dependem da 900001 (a FK de `field_device_id` entra na 900003); as Tasks 9 e 10 vêm antes do Step 4 da Task 8 (o fechamento cheio usa `CloseDay` e gera a ficha).

## Divergências propostas ao contrato

> **Decididas na §16 do contrato (2026-11-12):** W-1..W-8, W-13 → Q1; W-9 → Q11; W-10 → Q3; W-11 → Q12; W-12 → Q30; A-1 → Q2; A-2, A-3 → Q3; A-4 → Q4 (alternativa: reusar `citizen_without_*`); A-5 → Q5; A-6 → Q6; A-7 → Q7 (com `status` calculado); A-8 → Q8; A-9..A-11 → Q9; A-12 → Q10.

**W — precisões de redação (sem mudança de comportamento):**

W-1. **Onde moram as listas da linha do tempo.** `SOURCES`, `PROFESSIONAL_ONLY_KINDS` e `CITIZEN_FILTERED_KINDS` estão em `Timeline::Query` (`app/services/timeline/query.rb:12,23,37`), não em `Timeline`; só `KINDS` é de `Timeline` (`app/services/timeline.rb:13`). Proposta: escrever `Timeline::Query::…` no §12.

W-2. **Tabela 5 com códigos alfanuméricos.** As 117 linhas incluem `2231F9` (médico residente) e `1312C1` (sanitarista) — 115 numéricas (`vendor/ledi/8.7.0/docs/cbo.html`). Proposta: o §1 diz "117 códigos (strings; dois alfanuméricos)"; nenhum campo do 24b supõe CBO numérico.

W-3. **`registrar_cbo?` não muda.** `TerritoryRegistry::Access::REGISTRATION_CBOS` já carrega toda chave do YAML (`access.rb:12-13`); `ficha: :home_visit | :collective_activity` passa a valer só com o YAML. Proposta: trocar "passa a aceitar" por "aceita, pelo YAML".

W-4. **Gate do `home_visits` ≠ gate do 24a.** `TerritoryRegistry::Gate.usable?` ignora `ibge_code_missing`/`uf_missing` (`gate.rb:12-19`), mas `feature:territory_registry` exige o cadastro **utilizável** com IBGE e UF (`features.rb:342-345`). Com IBGE faltando, as rotas do 24a respondem e as da visita não. É o que o C1 diz; proposta: registrar que `HomeVisits::Gate` não copia a exceção de perfil do 24a.

W-5. **Quem chama `untracked?`.** Além de `Evaluation`, `Preview`, `CitizenView` e o quadro, passam o interruptor o `CareLines::BaseController` (quadro, inscrições, sugestões, prévia; `base_controller.rb:32`) e o `CitizenApi::CareController` (`:20`). Proposta: citar os dois no §12.

W-6. **Lista de `access`.** O api não tem lista fechada de valores de `access` do `clinical_record.viewed` (cada módulo publica a sua constante). Proposta: o §13 diz "os três valores novos são `HomeVisits::Catalog::TRAIL_ACCESS`, `CollectiveActivities::Catalog::TRAIL_ACCESS` e `FIELD_TRAIL_ACCESS`" em vez de "entram na lista".

W-7. **Filtro de log.** `:token`, `:body`, `:items`, `:reason`, `:note`, `:participants`, `:procedure_code`, `:new_patient`, `:cpf`, `:cns` e `:authorization` já são filtrados (`config/initializers/filter_parameter_logging.rb`); o Rails não loga cabeçalhos. O plano acrescenta só `measurements meeting_topics target_audience health_topics health_practices altered_evaluation smoking qr_payload place_text candidates withdrawal_reason mistaken_reason`. Proposta: trocar "mascara o cabeçalho de autorização no log" por "o spec de sigilo prova que o `Authorization` não aparece".

W-8. **Sem `Current.session` no app.** A requisição do aparelho não tem sessão: `Current.user` é nil. Todo comando recebe `by:` explícito, como no `Hl7::BaseController`. Proposta: registrar no §9.2 (nada de `Current` novo; `Current` é transversal).

W-9. **Cadastro pelo app só do ACS.** `TerritoryRegistry::Access.registrar?` só conhece ACS e enfermeiro/médico (`access.rb:35-41,99-104`): itens `household`/`individual_registration` de TACS, ACE e demais voltam `rejected cbo_not_allowed`. Proposta: dizer isso no §9.3, para o 24c não oferecer cadastro a esses CBOs.

W-10. **Cabeçalho da ficha 8.** O plano reusa `Ledi::Fichas::RegistrationHeader` com `saved_at` = `day` às 00h locais (`dataAtendimento` em ms). Fica até a task 1 confirmar.

W-11. **"Gerar de novo" com prazo vencido.** `siaps_deadline_passed` nunca se resolve: o `retry!` regrava a mesma falha. Proposta: o dashboard não oferece "Gerar de novo" para esse motivo.

W-12. **Eventos: spec × contrato.** A spec §5 lista 7 eventos; o contrato §1 tem 8 (`micro_area.field_assignment_changed`). O plano segue o contrato.

W-13. **Ordem na borda do app.** O §9.2 diz "gate antes de tudo", mas o `rate_limit` do Rails roda antes dos outros `before_action` (padrão do `Hl7::BaseController`): limite → interruptor → token. Proposta: escrever essa ordem.

**A — mudanças propostas:**

A-1. **Migrações irreversíveis.** O §1 diz "reversíveis, sem dado"; as do 24a e do 33 levantam `IrreversibleMigration` e executam o `db/city_triggers.sql` no `up` (`20261023900001_add_territory_registry.rb:165-169`, `20261111900002_add_ombudsman.rb:68-72`) — o `down` não desfaz triggers instalados pelo arquivo inteiro. Proposta: irreversíveis. O plano implementa assim.

A-2. **Domínio de `tipoGlicemia`.** O `dicionario.html` vendorizado traz `tipoGlicemiaCapilar` = `0 Jejum, 1 Pós-prandial, 2 Pré-prandial, 3 Não especificado`; o §10 diz `0 Não especificado, 1 Jejum, 2 Pós-prandial, 3 Pré-prandial`. Proposta: o do dicionário, salvo o `dicionario-fvd.html` dizer outro (task 1 decide).

A-3. **Público-alvo.** O `dicionario.html` tem `18 Outros` e também `19` e `20` (adolescente em atendimento socioeducativo, meio aberto e fechado), ausentes do §11; o `74` não aparece. Proposta: `18` como "Outros" e incluir `19` e `20`, salvo o `dicionario-fac.html` dizer outro.

A-4. **Nomes dos motivos de "não gerada".** `LediGenerationFailure::REASONS` já tem `citizen_without_birth_date` e `citizen_without_sex` (`ledi_generation_failure.rb:4-7`) com rótulo no dashboard; o §10 cria `patient_without_birth_date` e `patient_without_sex`. O plano segue o contrato (os nove códigos novos entram na constante). Proposta alternativa: reusar os dois existentes e economizar rótulos.

A-5. **Situação da ficha sem exportação.** Com a exportação não utilizável, `generate` devolve `:unusable` e não grava ficha nem falha (`registration_ficha.rb:31`): a folha fechada com visitas não cabe em nenhum `ficha.status` do §6. Proposta: `not_exported`. O plano implementa.

A-6. **Folha cheia e o índice de folha aberta.** "No máximo uma `open` por (profissional, equipe, dia)" e "a folha aberta com 99 abre a próxima `sequence`" só cabem juntos se a cheia fechar antes. Proposta: a 100ª visita fecha a cheia na mesma transação, com `closed_by = "full"` e a ficha gerada; `closed_by` aceita `user | job | full`. O plano implementa.

A-7. **Fim de vínculo sem evento.** Não há evento de fim de vínculo: o `Cnes::Apply` encerra o membro com `ended_on` e só publica `cnes.proposals_applied` (`app/commands/cnes/apply.rb:42,60-62`); `User#deactivate!` não publica nada. Proposta: revogar só na próxima requisição do aparelho (o §9.2 já cobre) e tirar "e no evento de fim de vínculo" do §2. Alternativa: revogar no `Cnes::Apply` (`member end`) e no `User#deactivate!`, tocando dois pontos transversais.

A-8. **Forma da busca de pessoa.** O §5 devolve o paciente ou `null`; o padrão das buscas existentes (`Vaccination::PersonLookupsController`, 27a) responde 404 `person_not_found`, 409 `citizen_not_verified` e transforma par validado em paciente (`Patients::LookupByIdentifier`), sem `birth_date`. Proposta: o padrão existente, mais `birth_date` na forma. O plano implementa.

A-9. **`out_of_scope` nas remoções.** Mudança de escopo responde `reset: true` (§9.3), então nada produz `removals.reason = "out_of_scope"`. Proposta: tirar do v1, ou mantê-lo reservado no schema. O plano mantém no enum e nunca emite.

A-10. **Tombstone `erased` sem produtor.** Quem tem qualquer versão de cadastro individual ou é responsável de família é retido (`request_erasure.rb:161-163`); a exclusão nunca apaga cadastro nem domicílio. Proposta: `erased` reservado, sem gancho no `Citizens::Erase` no 24b. O spec da Task 23 prova a retenção.

A-11. **Saída do cadastro não é remoção.** O §1 grava tombstone quando o cadastro "sai de uma microárea"; a saída por mudança ou óbito e o domicílio inativo continuam na microárea, com estado. Proposta: vão como item (com `exit`/`status`); tombstone `moved` só quando o domicílio troca de microárea ou o cadastro passa a outra microárea. O plano implementa.

A-12. **Vitais da visita e o interruptor.** O §12 diz que `blood_pressure` e `weight_height` "ganham as visitas", sem dizer se dependem de `home_visits`. Proposta: só com `home_visits` utilizável, pelo mesmo `home_visits:` do `acs_visit` (uma leitura; desligado, a visita não entra em nenhuma fonte). O plano implementa.
