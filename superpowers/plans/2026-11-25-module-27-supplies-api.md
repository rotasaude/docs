# Módulo 27 — Insumos (catálogo de materiais, diabetes e estomia) (api) Implementation Plan

> **Nota do coordenador (Q6, alinhamento com o plano do maintenance):** a Task 2 também cria a consulta GraphQL de leitura `supplyCatalog` (paginada, com filtro de `unit_review` e de `group`, só leitura), que a tela "Importar catálogo de materiais" do maintenance usa para a revisão. A mutation `importSupplyCatalog` não pede TOTP, e nenhum teste deve esperar o código.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: o 27b e o 26c na `origin/main` do api.** Ao escrever este plano (2026-11-24) a `origin/main` era `ca2a92f0`, **sem** os dois: última migração de cidade `20261123900003_add_zoonoses_care.rb` (`db/city_schema.rb:13`), última de plataforma `20261113900001`. O plano do 27b (`2026-11-24-module-27b-bnafar-api.md`) acrescenta `20261125900001..900004` (cidade) e `20261125600001` (plataforma); o do 26c (`2026-11-25-module-26c-mira-api.md`), `20261126900001`. Este plano **parte desse estado**; a Task 0 confere e para se faltar um dos dois. Nomes conferidos no código (ca2a92f0):
> - **Estoque:** `Stock::Domain::{MEDICATION, IMMUNOBIOLOGICAL, VECTOR_CONTROL, FEATURES, for, of_lot, allows?, usable?}` (`stock/domain.rb:36-42`: o `of_lot` devolve MEDICATION para todo lote que não é vacina nem vetorial); `Pharmacy::Ledger.{lock_lots!, lock_location!, record!, balance, balances}` (`record!` copia `catalog_item_id`, `immunobiological_code`, `vector_product_id` do lote, `ledger.rb:99-103`; `DOCUMENTS`, `:41-46`); `Pharmacy::RecordEntry` (`Row`, `FOREIGN_KEYS`, `OPENING_INDEXES`, `ensure_lots`, `find_lots`, `lot_key`, `mismatch_problems`, `mark_openings`; `record_entry.rb:66-110, 414-504`); `Pharmacy::Catalog::{MOVEMENT_KINDS, MAX_DISPENSE_ITEMS = 20, REASON, COUNCILS, UF, COUNCIL_NUMBER, DISPENSER_ROLES, MAX_LIMIT}`; `Pharmacy::Json.{item, catalog_item, lot, entry, transfer, dispensation, dispensation_item, catalog_for, DISPENSATION_PRELOAD}` (`catalog_for` só lê `MedicationCatalogItem`, `json.rb:221`); `Pharmacy::Positions` (CTEs por `catalog_item_id`, `positions.rb:75-118`); `Pharmacy::SaveMinimums`; `Pharmacy::Dispense` (molde de travas lotes → local → documento, `Refused`, `problem`, `failure`); `Pharmacy::RecordReturn.{call, controlled?}`; `Pharmacy::Dispensed.{last_on, returned_totals, outside_caps}` (`last_on` agrupa por `catalog_item_id`, `dispensed.rb:46-51`); `Pharmacy::RecordExternalPrescription` (prescritor `name`, `council`, `council_state`, `council_number`, `issued_on`); `Pharmacy::BaseController` (`render_failure`, `render_refusal`, `query_only!`, `page_limit`, `body_param`, `dispenser_at?`, `admin?`, `FAILURE_STATUS`, `RateLimitStore`); `Pharmacy::Access.allowed?`; `Pharmacy::Trail.person_viewed!`; `StockGate#{require_stock_feature!, stock_feature_for!}`; rotas em `scope "/pharmacy"` (`config/routes.rb:550-596`).
> - **Tabelas:** `ck_stock_lots_item` com `(catalog_item_id IS NULL) = (catmat_code IS NULL)` (`city_schema.rb:4546`); `ck_stock_minimums_item`, `ck_stock_openings_item`, `ck_stock_movements_item` com `num_nonnulls(catalog_item_id, immunobiological_code, vector_product_id) = 1`; `dispensations.ck_dispensations_prescription` (`:1366`); `dispensation_items.catalog_item_id NOT NULL` e `ck_dispensation_items_source` (`:1333, 1346`); `performed_procedures` com `ck_performed_procedures_source_ref` e `_domains` (`:3394-3401`) e `professional_user_id NOT NULL`; `health_units` sem campo de estomia (`:1761-1791`).
> - **Catálogo:** `medication_catalog_items` e `medication_catalog_releases` (índices únicos **globais** `one_active`/`one_importing`, `platform_schema.rb:407-408`); `Medications::Catmat::{Client (CLASS_CODE = 6505, build_row, fetch_all), Row, FixtureClient}`; `Medications::CatalogImport` (supersede **todas** as releases ativas, `catalog_import.rb:69`; audita `medication_catalog.imported`); `Medications::ReleaseLifecycle.{fail_stale!(release_class), mark_failed}`; `lib/tasks/medications.rake#import_catmat`; `Maintenance::Mutations::ImportMedicationCatalog`.
> - **Consulta:** `Consultations::SaveDraft` (bloco `dental` por `DentalCare::DraftInput`, `save_draft.rb:28-35`); `Consultations::Finalize` (`DentalCare::Materialize` na `:63`, `update!(status: "finalized", …, draft_items: {})` na `:64`; interruptores lidos fora, `:43-46`); `Consultation::KINDS = %w[general dental urgency]`; `consultations.supplies` (odontologia, `city_schema.rb:1069`).
> - **Faturamento:** `Billing::{Gate::KEY = "procedure_billing" (features.rb:111, gate.rb:8), Record.insert!, Establishment.for_unit, Staff.cbo_for, Critic, SigtapCache#max_quantity (QT_MAXIMA_EXECUCAO, terminology/sigtap_reader.rb:97), Json.source_ref}`; molde `Billing::FromTfd` + `Billing::TfdProductionJob` (`IdempotentConsumer`, `retry_on`); `PerformedProcedure::SOURCES`.
> - **Linha do tempo:** `Timeline::KINDS` (`timeline.rb:18`), `Timeline::Query::{SOURCES, PROFESSIONAL_ONLY_KINDS}`, `Sources::{DispensationSource, ProcedureSource, Labels::DISPENSATION_TITLE}`, `PecTimelinesController::KINDS` (`:16`).
> - **Outros:** `Platform::Features::{CATALOG, CITIZEN_HIDDEN, missing_for}` (`feature:` → `<chave>_disabled`); `HealthUnitsController#{update_locked, assign_caps, unit_json}`; `ClinicalRecord::{Access, Trail.viewed!}` (molde `PatientMedicationsController#index`); `Patients::{LookupByIdentifier, CreateTyped}`; `Mfa::Verify.totp_step_for` + `User#consume_totp_step!`; `IdempotentConsumer`, `EachCityJob`; `CityEncryption::CITY_KEYED_TARGETS`; JSON cifrado = coluna `text` com `serialize :x, coder: JSON` + `encrypts :x` (`tfd_case.rb:8-9`).

**Goal:** O lado api dos insumos do módulo 27: catálogo CATMAT de materiais (6515/6550) na plataforma, o insumo como quarta chave do estoque de medicamentos, inscrição em programa (consulta, laudo externo e planilha), regras de cota da cidade, retirada por inscrição com exceção e terceiro, devolução, comodato de aparelhos, a bolsa de estomia no faturamento, a linha do tempo e a inativação.

**Architecture:** Uma migração de plataforma (`supply_catalog_items`, `supply_catalog_releases`) e quatro de cidade, irreversíveis: `900001` quarta chave do estoque; `900002` programas (regras, inscrições e eventos, dispensação por inscrição, ponto de estomia, fonte `supply` no faturamento); `900003` comodato; `900004` importação. O domínio mora em `Supplies::*` (catálogo e carga) e `SupplyPrograms::*` (regras puras de perfil, cota e saldo; comandos de inscrição). A retirada é `Pharmacy::DispenseSupplies`, irmã do `Pharmacy::Dispense`, com a mesma ordem de travas (lotes → local → inscrição). A bolsa vira BPA-I por `Billing::FromSupply`, consumidor de `dispensation.recorded`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), Solid Queue, RSpec via `bin/test`, WebMock, Ruby `csv`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-25-module-27-supplies-design.md` (decisões 1–18). Contrato entre apps — **fonte única**: `docs/superpowers/plans/2026-11-25-module-27-supplies-contracts.md` (§1–§11, Q1–Q13; docs `7348ae0`). Pesquisa: `docs/pesquisa/2026-11-23-insumos-e-bnafar-27.md`. Molde vizinho: `2026-11-24-module-27b-bnafar-api.md`.

## Reconciliação (§11, Q6–Q13)

As divergências D1–D18 foram decididas no §11 do contrato (docs `7348ae0`), quase todas pela proposta; as tasks já seguem as decisões. Onde a decisão acrescenta algo:
- **Q6:** a mutation `importSupplyCatalog` **não pede TOTP**, como a `importMedicationCatalog`; a revisão do grupo e da unidade no maintenance é só leitura (Task 2).
- **Q7:** a resposta do `PATCH /consultations/:id` traz a cota calculada em `draft_items.supply_programs[i].quotas`; a inscrição ganha `closed_in_consultation_id`, gravado no encerramento (Tasks 7 e 9).
- **Q8:** o `GET` das regras devolve a `version` do conjunto, que o `PUT` leva em `expected_version`, com step-up (Task 8).
- **Q10:** rota nova `GET /pharmacy/locations/:id/pharmacists`, para escolher o farmacêutico da exceção (Task 11).
- **Q12:** a conferência da importação roda em `dry_run` e não cria paciente; só a aplicação cria; **não há desfazer** (Task 16).
- **Q13:** o pré-requisito `feature:clinical_record` do `supply_programs` responde `clinical_record_disabled`, motivo que já tem frase (Task 1).

## Valores fixados por este plano

1. **Interruptores** (fim do `CATALOG`, depois do `rnds_regulation` do 26c): `Entry.new(key: "pharmacy_supplies", description: "Insumos na farmácia", requires: %w[feature:pharmacy])` e `Entry.new(key: "supply_programs", description: "Programas de insumos (diabetes e estomia)", requires: %w[feature:pharmacy_supplies feature:clinical_record])`. Fora do `CITIZEN_HIDDEN` (nada do cidadão). `Supplies::Gate::KEY = "pharmacy_supplies"`, `SupplyPrograms::Gate::KEY = "supply_programs"`, os dois com `usable?(city)`; lidos fora da transação da cidade e passados como argumento. Faturamento da bolsa: `Billing::Gate.enabled?(city)` (`procedure_billing`).
2. **`Supplies::Catalog`:** `CLASS_CODES = [6515, 6550]`; `SOURCE = "catmat_6515_6550"`; `GROUPS = %w[glucose_strip lancet insulin_syringe pen_needle ostomy_bag ostomy_accessory other]`; `PROGRAMS = %w[diabetes ostomy]`; `PROGRAM_GROUPS = { "diabetes" => %w[glucose_strip lancet insulin_syringe pen_needle], "ostomy" => %w[ostomy_bag ostomy_accessory] }`; `INSULIN_TYPES = %w[nph regular analog none]`; `STOMA_TYPES = %w[colostomy ileostomy urostomy other]`; `PER_DAY = (0..10)`; `VALIDITY_MONTHS = (3..12)`; `EXTERNAL_MAX_MONTHS = 3`; `INACTIVE_AFTER_DAYS = 180`; `PERIOD_DAYS = 30`; `PERIOD_RANGE = (1..365)`; `QUOTA_QUANTITY = (1..9_999)`; `MAX_RULES = 100`; `MAX_ITEMS = 20`; `ENROLLER_CBO_PREFIXES = %w[225 2235]`; `REPORT_COUNCILS = %w[CRM COREN]`; `COLLECTOR_NAME = (3..120)`; `COLLECTOR_DOCUMENT = %r{\A[0-9A-Za-z./-]{4,30}\z}`; `SERIAL = %r{\A[0-9A-Za-z./-]{1,40}\z}`; `POLICE_REPORT = %r{\A[0-9A-Za-z./-]{1,40}\z}`; `DEVICE_KINDS = %w[glucometer cgm_reader]`; `LOAN_STATUSES = %w[active returned replaced lost]`; `ENROLLMENT_STATUSES = %w[active inactive closed]`; `ORIGINS = %w[consultation external_report import]`; `EVENT_KINDS = %w[opened renewed closed inactivated]`; `DOMAIN_EVENTS` = os seis do contrato §9.
3. **YAML com spec:** `config/supplies/programs.yml` (diabetes: `cid_prefixes: [E10, E11, E12, E13, E14, O24]`; ostomy: `cid_prefixes: []` = qualquer CID-10 válido; D18); `config/supplies/groups.yml` (regras sobre o PDM, em ordem, a primeira que casa: `TIRA REAGENTE` + descrição com `GLICOSE|GLICEMIA` → `glucose_strip`; `LANCETA` → `lancet`; `SERINGA` + `INSULINA` → `insulin_syringe`; `AGULHA` + `CANETA` → `pen_needle`; `BOLSA` + `COLOSTOMIA|ILEOSTOMIA|UROSTOMIA|OSTOMIA|ESTOMIA` → `ostomy_bag`; `PLACA|ADJUVANTE|PASTA|PO|ANEL|PROTETOR` + `OSTOMIA|ESTOMIA` → `ostomy_accessory`; o resto → `other`; texto sem acento e maiúsculo); `config/supplies/ostomy_sigtap.yml` (regras `{ sigtap_code, pdm, description }` da correspondência da Task 0 sobre 0701050012, 0701050020, 0701050047, 0701060026, 0701060034, 0701060042; `review: true`).
4. **Frases e códigos de recusa** (422 salvo nota; `path` no formato `items[i].campo`): do contrato `quota_exceeded`, `ostomy_return_not_allowed`, `not_ostomy_service`, `patient_without_cns`, `sigtap_max_quantity`; deste plano `enrollment_not_active`, `enrollment_expired`, `enrollment_exists` (409), `supply_enrollment_changed` (409), `supply_quota_rules_changed` (409), `device_loan_changed` (409), `device_model_changed` (409), `serial_in_use` (409), `supply_enrollment_import_changed` (409), `item_not_in_program`, `invalid_profile`, `invalid_cid10`, `cid_not_in_program`, `invalid_validity`, `enroller_not_allowed` (403), `override_requires_pharmacist` (403), `override_code_invalid` (401), `supply_without_sigtap`, `invalid_quantity`, `invalid_file`, `file_too_large` (413). As frases ficam no `SUPPLIES_ERRORS` do dashboard.
5. **Bloco `supply_programs` do rascunho** (D4): lista de até 4 `{ action: "open"|"renew"|"close", program, enrollment_id? (renew/close), cid10, profile, valid_until, preferred_unit_id, quota_overrides?: [{ group, quantity }], override_reason? }`; o `DraftInput` grava em `draft_items["supply_programs"]` cada linha com `quotas` **calculada** (a cota à vista; a do cliente é ignorada); `close` só leva `enrollment_id`.
6. **Rotas novas** (sessão do dashboard; listas `{ items, next_before }`, padrão 50, máximo 100):
   - Farmácia (`Pharmacy::BaseController`, `pharmacy` + `pharmacy_supplies`): `POST /pharmacy/items/search { q, item_kind: "supply" }` (D9); `GET /pharmacy/supply_enrollments?patient_id=` (nominal, trilha, 30/min); `POST /pharmacy/supply_enrollments` (farmacêutico, laudo externo; `supply_programs`); `POST /pharmacy/supply_dispensations`; `GET /pharmacy/locations/:id/pharmacists` (Q10); `GET|PUT /pharmacy/admin/supply_quota_rules?program=` (`municipal_admin`, PUT com step-up); `GET /pharmacy/device_models`; `GET|POST /pharmacy/admin/device_models`, `PATCH /pharmacy/admin/device_models/:id` (D13); `GET /pharmacy/device_loans?health_unit_id=|patient_id=`, `POST /pharmacy/device_loans`, `POST /pharmacy/device_loans/:id/{return,replace,loss}`; `GET /pharmacy/supply_enrollment_imports/template.csv`, `POST /pharmacy/supply_enrollment_imports`, `GET /pharmacy/supply_enrollment_imports/:id`, `POST …/:id/{apply,discard}` (sem desfazer, Q12).
   - Prontuário (D5): `GET /patients/:id/supply_enrollments` (lista sem CID nem perfil) e `GET /supply_enrollments/:id` (com CID e perfil), pelo `ClinicalRecord::Access`, com trilha.
   - Unidade: `ostomy_service` e `ostomy_responsible_professional_id` no `POST`/`PATCH` de `HealthUnitsController` e no `unit_json`.
7. **Porta e simulado:** api do worktree na **3065**; nada externo nas specs (`stub_request` do CATMAT pelo `stub_catmat!` estendido).

## Global Constraints

- git `/opt/homebrew/bin/git`; nunca `git add -A` nem `git stash` sem nome. Commits em inglês, Conventional Commits com o tipo por extenso: `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge, push e cards só com autorização do usuário.
- **Testes só por `bin/test api -w mod27s <specs>`**, a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`; nunca `rspec` direto nem o Postgres do host; `--keep-db` só em rodada de arquivo isolado. A branch é transversal (migração de cidade e de plataforma, estoque do 27a/29/32b/35a/35b, 27b, `Consultations::Finalize`, faturamento, linha do tempo, `CityEncryption`, `Platform::Features`, `spec/support`, initializers) → **suíte completa obrigatória, pedida à sessão "Testes" na Task 20**.
- Migrações: dump à mão em `db/city_schema.rb` e `db/platform_schema.rb`; triggers num bloco "Módulo 27 — insumos" ao fim do `db/city_triggers.sql`, cada um dentro de `IF to_regclass('public.<tabela>') IS NOT NULL`; **irreversíveis** (`down` → `ActiveRecord::IrreversibleMigration`; o `up` de cidade termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`); CHECK trocado por `replace_check` com espelho das listas na migração (molde `20261116900002_add_vector_inspections.rb#change_stock_items`). Banco de dev só migrado e semeado na prova, com autorização do usuário.
- **O 27a não muda para medicamento** (spec §4): todo spec existente de `spec/commands/pharmacy` e `spec/requests/pharmacy` segue verde sem edição de asserção.
- Specs com tipo explícito (`type: :request`, `:model`, `:job`). Nenhuma spec bate em URL real.
- **Nunca `Current.city =` solto**; interruptores fora da transação. `spec/architecture/current_city_assignment_spec.rb` roda nas Tasks 9, 13, 15 e 20.
- **Sigilo (contrato §9):** CID, perfil, laudo externo, motivo de exceção, quem retira, boletim de ocorrência e o CSV cifrados (`encrypts`, chave da cidade, em `CITY_KEYED_TARGETS` e em `spec/jobs/reencryption_job_spec.rb`); nunca em log, URL, evento, `Result`, `inspect` nem lista. `Data`/`Struct`/modelo com esses campos sobrescreve `inspect`/`to_s`/`pretty_print`/`as_json`. `filter_parameter_logging` ganha `cid10`, `profile`, `external_report`, `collector`, `override`, `police_report_number`, `file_base64`.
- Eventos em `config/initializers/domain_events.rb` e `spec/initializers/domain_events_bindings_spec.rb`, só ids; `DomainEvents.publish(` é multilinha. R18: `supply_catalog.imported` e `maintenance.supply_catalog.imported` em `R18_PLATFORM_EVENT_NAMES`; o segundo também em `MaintenanceAudit::NAMES` (D17).
- Concorrência otimista: escrita sobre linha mutável leva `expected_version` (ausente → `422 expected_version_required`); diferente → `<entidade>_changed`.
- Todo "hoje" é `Time.zone.today` dentro de `CityConnection.with`.
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`.

## Review Focus

1. **Virada do período e devolução de retirada antiga:** paciente com cota 100/30 dias retira 60 no dia 1, 40 no dia 29, nada no dia 30, 50 no dia 31 (novo ciclo a partir dessa retirada); devolve no dia 35 10 da retirada do dia 29. O saldo do ciclo novo não muda (a devolução volta ao período da retirada original) e o do ciclo velho não acumula. Teste na Task 10.
2. **Exceção pelo auxiliar com o código de outro:** código TOTP de farmacêutico sem papel no local, código repetido (o mesmo passo duas vezes) e código do próprio auxiliar → recusados, nada gravado; o farmacêutico na própria sessão passa sem código. Teste na Task 11.
3. **Dispensação de medicamento convivendo com a de insumo:** o histórico `GET /pharmacy/dispensations`, o `Dispensed.last_on` (medicamentos em uso) e a linha do tempo não quebram nem misturam itens com `catalog_item_id` nulo. Teste na Task 11.
4. **Duas aberturas da mesma inscrição ao mesmo tempo** (finalização da consulta e laudo na farmácia): uma ativa só; a outra recebe `enrollment_exists` e a consulta volta a rascunho sem nada gravado. Teste nas Tasks 9 e 10.
5. **Bolsa fracionada e teto SIGTAP somado entre unidades:** quantidade `1.5` → `invalid_quantity`; 20 bolsas num ponto e 15 em outro no mesmo mês, com teto 30 → a segunda recusada com `sigtap_max_quantity`; com `procedure_billing` desligado, entrega sem BPA e sem essas duas conferências. Teste na Task 13.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod27s/`)
- Migrações: `db/platform_migrate/20261127600001_create_supply_catalog.rb`; `db/city_migrate/{20261127900001_add_supply_stock_key,20261127900002_add_supply_programs,20261127900003_add_device_loans,20261127900004_add_supply_enrollment_imports}.rb`.
- Modelos: `app/models/{supply_catalog_item,supply_catalog_release,supply_quota_rule,supply_enrollment,supply_enrollment_event,device_model,device_loan,supply_enrollment_import,supply_enrollment_import_row}.rb`.
- `app/services/supplies/{gate,catalog,groups,sigtap_map,units,catalog_import,search}.rb`; `app/services/supply_programs/{gate,programs,profile,external_report,quota,balance,draft_input,materialize,json}.rb`; `app/services/supply_enrollment_imports/{reader,validator}.rb`; `app/services/timeline/sources/supply_enrollment_source.rb`.
- `app/commands/supply_programs/{save_rules,record_external}.rb`; `app/commands/pharmacy/dispense_supplies.rb`; `app/commands/billing/from_supply.rb`; `app/commands/device_loans/{lend,give_back,replace,report_loss}.rb`; `app/commands/device_models/save.rb`; `app/commands/supply_enrollment_imports/{upload,apply,discard}.rb`.
- `app/jobs/billing/supply_production_job.rb`; `app/jobs/supply_programs/inactivate_job.rb`.
- `app/controllers/pharmacy/{supply_enrollments,supply_dispensations,device_loans,device_models,supply_enrollment_imports}_controller.rb`; `app/controllers/pharmacy/admin/{supply_quota_rules,device_models}_controller.rb`; `app/controllers/supply_enrollments_controller.rb`.
- `app/graphql/maintenance/mutations/import_supply_catalog.rb`; `lib/tasks/supplies.rake`; `config/supplies/{programs,groups,ostomy_sigtap}.yml`; `lib/supplies_crew.rb`; `spec/support/supplies_helpers.rb`; `spec/fixtures/supplies/{catmat_items,catmat_units}.json`, `spec/fixtures/supply_enrollment_imports/{ok,excel,latin1}.csv`; `spec/security/supplies_secrecy_spec.rb`.

**Modificar**
- Plataforma e infra: `app/services/platform/features.rb`, `app/services/medications/catmat/{client,row}.rb`, `app/services/city_encryption.rb`, `db/{city,platform}_schema.rb`, `db/{city,platform}_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `app/events/maintenance_audit.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `db/seeds.rb`.
- Estoque: `app/services/stock/domain.rb`, `app/models/{stock_lot,stock_movement,stock_minimum,stock_opening,dispensation,dispensation_item,health_unit,performed_procedure}.rb`, `app/services/pharmacy/{ledger,json,positions,dispensed}.rb`, `app/commands/pharmacy/{record_entry,save_minimums,record_return}.rb`, `app/controllers/pharmacy/{items,entries,positions,minimums,transfers,adjustments,returns,dispensations,base}_controller.rb`, `app/services/pharmacy/statement.rb`.
- 27b (nomes do plano do 27b): `app/services/bnafar/{payload,snapshot,pendencies,units}.rb`, `app/services/opening_imports/validator.rb`.
- Consulta, faturamento e linha do tempo: `app/commands/consultations/{save_draft,finalize}.rb`, `app/services/billing/json.rb`, `app/controllers/health_units_controller.rb`, `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/services/timeline/sources/{dispensation_source,procedure_source,labels}.rb`, `app/controllers/pec_timelines_controller.rb`, `app/commands/patients/lookup_by_identifier.rb`.

---

### Task 0: Worktree, pré-requisitos, chave do faturamento e cards

**Files:** nenhum código.

- [ ] **Step 1:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod27s -b feat/mod-27-supplies origin/main && cp config/master.key .claude/mod27s/config/master.key`.
- [ ] **Step 2: Conferir o 27b e o 26c na main** (com `grep`/`ls` no worktree): `db/city_migrate/20261125900001_add_bnafar_stock_data.rb` … `900004` e `db/platform_migrate/20261125600001_add_obm_presentations.rb`; `app/services/bnafar/{payload,snapshot,pendencies,units,lot_fields}.rb`; `app/services/opening_imports/validator.rb`; `db/city_migrate/20261126900001_add_regulation_rnds.rb`; a última `Entry` do `CATALOG` (`rnds_regulation`). **Algum ausente → parar** e avisar o coordenador. Migração mais nova que `20261127900001` (cidade) ou `20261127600001` (plataforma) → os números deste plano sobem para o dia seguinte ao dela (nota no topo antes da Task 1).
- [ ] **Step 3: Chave do faturamento do 22:** `grep -n 'KEY = ' app/services/billing/gate.rb` → `"procedure_billing"`; diferente → trocar no Valor 1 antes da Task 1.
- [ ] **Step 4: Correspondência SIGTAP × CATMAT da estomia:** pela API aberta (`/modulo-material/3_consultarPdmMaterial?codigoClasse=6515` e `4_consultarItemMaterial?codigoPdm=`), listar as PDMs de bolsa e adjuvante de estomia e casar com os seis códigos OPM do Valor 3 pelo nome do procedimento na SIGTAP ativa do dev (`SigtapProcedure.where(code: […]).pluck(:code, :name)`, leitura). Sem as seis na release → anotar e seguir (a Task 2 grava as regras com `review: true`).
- [ ] **Step 5:** `bin/test api -w mod27s spec/services/platform/features_spec.rb spec/commands/pharmacy spec/requests/pharmacy spec/services/stock spec/services/medications spec/commands/consultations spec/commands/billing spec/services/timeline spec/requests/timeline_spec.rb spec/services/bnafar spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb` → PASS (linha de base).
- [ ] **Step 6 (só com autorização do usuário):** criar no board #1 os cards dos insumos pela receita da memória `board-91-features-roadmap` (Module `mod-27`, Work type `feature`), numerando depois do último F-27 do board, um `gh project item-create …` por chamada, cada comando começando com `gh`: "Catálogo CATMAT de materiais e quarta chave do estoque"; "Inscrição em programa (consulta, laudo externo, planilha)"; "Regras de cota e retirada por inscrição"; "Comodato de aparelhos"; "Bolsa de estomia no faturamento"; "Linha do tempo e inativação". Sem autorização: anotar no relato e seguir.

**Relato ao coordenador:** "api insumos · Task 0 · worktree mod27s em origin/main <hash>; 27b e 26c <presentes | ausente: parado>; chave do faturamento <procedure_billing>; correspondência SIGTAP <n de 6>; cards <criados | aguardando autorização>; linha de base verde."

---

### Task 1: Interruptores e gates

**Files:**
- Create: `app/services/supplies/gate.rb`, `app/services/supply_programs/gate.rb`, `spec/services/supplies/gate_spec.rb`
- Modify: `app/services/platform/features.rb`, `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`

**Interfaces:**
- Produces: Valor 1. `Supplies::Gate.usable?(city)`, `SupplyPrograms::Gate.usable?(city)`.

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: índices contados do fim sobem dois; "`pharmacy_supplies` e `supply_programs` fecham o catálogo, nessa ordem, com os `requires` do Valor 1"; "`pharmacy` desligado → `pharmacy_supplies` com `pharmacy_disabled`"; "`supply_programs` sem prontuário → `clinical_record_disabled` (Q13); sem `pharmacy_supplies` → `pharmacy_supplies_disabled`". `city_features_spec.rb`: as listas exatas com as duas chaves. `session_features_spec.rb`: as duas aparecem para o cidadão só quando ligadas (fora do `CITIZEN_HIDDEN`). `gate_spec.rb`: "`usable?` falso com cidade nula; verdadeiro com a cadeia ligada".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/platform/features_spec.rb spec/services/supplies/gate_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/citizen_api/session_features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** as duas `Entry` (comentário: "Módulo 27, insumos: sem cascata; desligar não apaga inscrição nem lote") e os gates no molde do `Pharmacy::Gate`.
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add pharmacy supplies and supply programs feature switches" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 1 · pharmacy_supplies (requires pharmacy) e supply_programs (requires pharmacy_supplies e clinical_record) no fim do CATALOG."

---

### Task 2: Plataforma — catálogo de materiais, carga CATMAT 6515/6550, rake e `importSupplyCatalog`

**Files:**
- Create: `db/platform_migrate/20261127600001_create_supply_catalog.rb`, `app/models/{supply_catalog_item,supply_catalog_release}.rb`, `app/services/supplies/{catalog,groups,sigtap_map,units,catalog_import,search}.rb`, `config/supplies/{groups,ostomy_sigtap}.yml`, `lib/tasks/supplies.rake`, `app/graphql/maintenance/mutations/import_supply_catalog.rb`, `spec/fixtures/supplies/{catmat_items,catmat_units}.json`, `spec/services/supplies/{catalog_import,groups,units,search}_spec.rb`, `spec/requests/maintenance/supply_catalog_spec.rb`, `spec/tasks/supplies_import_catmat_spec.rb`
- Modify: `db/platform_schema.rb`, `db/platform_triggers.sql`, `app/services/medications/catmat/{client,row}.rb`, `spec/support/medication_helpers.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/events/maintenance_audit.rb`, `spec/services/medications/catalog_import_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
- `supply_catalog_releases` (molde de `medication_catalog_releases`, D1): `source` default `catmat_6515_6550`, os mesmos status, contagens e índices únicos `one_active`/`one_importing` **desta tabela**.
- `supply_catalog_items`: `catmat_code integer` único `> 0`, `class_code integer CHECK IN (6515, 6550)`, `pdm_code integer`, `pdm_name text NOT NULL`, `description text NOT NULL`, `supply_unit varchar(20) NOT NULL`, `unit_review boolean NOT NULL DEFAULT false` (D2), `sigtap_code varchar(10) CHECK (~ '^[0-9]{10}$')`, `group varchar NOT NULL CHECK IN GROUPS`, `status` (`active|removed`), `hidden boolean`, `search_text text NOT NULL`, `first_release_id`, `last_release_id`, `source_updated_at`, timestamps. Trigger: sem DELETE.
- `Medications::Catmat::Client.new(class_code: 6505, …)` (D2): `CLASS_CODE` vira o padrão do argumento; `fetch_all` igual; `Row` ganha `class_code`. Novo `Client#fetch_units -> { pdm_code => [UnitRow(sigla:, nome:, capacidade:, sequencia:)] }` lendo `/modulo-material/6_consultarMaterialUnidadeFornecimento?statusUnidadeFornecimentoPdm=true` paginado (mesmo retry, total conferido). `stub_catmat!` ganha `class_code:` e `stub_catmat_units!`.
- `Supplies::Units.pick(units) -> [nome_em_minusculas, unit_review]`: `UNIDADE` ativa → `["unidade", false]`; senão a de menor `sequencia` → `[nome, units.size > 1]`; nenhuma → `["unidade", true]`.
- `Supplies::Groups.for(pdm_name:, description:) -> String` (Valor 3). `Supplies::SigtapMap.for(pdm_name:, description:) -> String | nil`.
- `Supplies::CatalogImport.call(by:, class_codes: Supplies::Catalog::CLASS_CODES, client_for: ->(code) { Medications::Catmat::Client.new(class_code: code) }, now: Time.current) -> Result.ok(release:, added:, changed:, removed:) | fail(:import_in_progress | :source_unavailable | :incomplete_source | :suspicious_drop | :import_failed)`: molde do `Medications::CatalogImport` (release `importing` fora da transação, fonte inteira antes, `MIN_SHARE` 0,5, upsert por `catmat_code`, sumiu → `removed`, supersede só `SupplyCatalogRelease`); `Platform.audit("supply_catalog.imported", release_id:, added:, changed:, removed:)`.
- `Supplies::Search.call(q:) -> [SupplyCatalogItem]`: ativos e não escondidos, todos os termos em `search_text` (tokens como `Medications::Substances.key`, até 6, mínimo de `Medications::Search::MIN_QUERY`), até 20, os grupos de programa antes de `other`.
- Rake `supplies:import_catmat` (produção). Mutation `importSupplyCatalog` (`BaseMutation`, **sem TOTP**, como a `ImportMedicationCatalog` (Q6); `audited(event: "maintenance.supply_catalog.imported", module_name: "supplies")`; o campo só existe com `Rails.env` em `SIMULATION_ENVS`, como o `importObmCatalog` do 27b).

- [ ] **Step 1: Write the failing tests.** `catalog_import_spec.rb` (`type: :model`): "as duas classes do fixture criam os itens com `class_code`, `supply_unit` e grupo"; "tira de PDM 402 com `UN`, `RO` e `EMB` → `unidade`, sem `unit_review`"; "PDM só com `CX` e `FR` → a de menor sequência e `unit_review: true`"; "bolsa de colostomia recebe o `sigtap_code` do YAML; tira fica sem"; "reimportar sem um item → `removed`, nunca apagado"; "a carga de insumos não toca `medication_catalog_releases` (a release de medicamento ativa continua ativa)"; "metade some → `suspicious_drop`"; "audita só contagens". `groups_spec.rb`: um teste por regra do Valor 3 e "fralda → `other`". `units_spec.rb`: os três casos do `pick`. `search_spec.rb`: "`lanceta 28g` acha a lanceta; item `removed` fica fora". `catalog_import_spec.rb` do 6505: "`Client.new` sem argumento segue na 6505". `supplies_import_catmat_spec.rb`: "a rake importa e imprime as contagens". `supply_catalog_spec.rb` (`type: :request`): "sem TOTP → `ok` e auditoria `maintenance.supply_catalog.imported`"; "em produção (stub do schema) o campo não existe". R18 com os dois nomes.
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/supplies spec/services/medications spec/tasks/supplies_import_catmat_spec.rb spec/requests/maintenance/supply_catalog_spec.rb spec/events/platform_event_payload_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_27_600001)`; `MaintenanceAudit::NAMES` + `maintenance.supply_catalog.imported`).
- [ ] **Step 4:** os mesmos specs + `spec/requests/maintenance/medication_catalog_spec.rb spec/graphql/maintenance/analyzers_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: import the catmat supply catalog for classes 6515 and 6550" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 2 · supply_catalog_items/releases próprios (D1); unidade pela PDM (D2); grupo e SIGTAP por YAML; rake supplies:import_catmat e importSupplyCatalog fora de produção."

---

### Task 3: Migração 900001 — quarta chave do estoque e `Stock::Domain.of_lot`

**Files:**
- Create: `db/city_migrate/20261127900001_add_supply_stock_key.rb`, `spec/models/supply_stock_schema_spec.rb`, `spec/support/supplies_helpers.rb` (`supplies_city!`, `supply_item!(group:, catmat_code: nil, sigtap_code: nil, supply_unit: "unidade")` — grava `SupplyCatalogItem` na plataforma —, `supply_lot!(location, item:, quantity: "100")`)
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{stock_lot,stock_movement,stock_minimum,stock_opening}.rb`, `app/services/stock/domain.rb`, `app/services/pharmacy/ledger.rb`, `spec/services/stock/domain_spec.rb`, `spec/services/city_schema_spec.rb`

**Interfaces:**
- `stock_lots` + `supply_catalog_item_id uuid` (sem FK: é da plataforma, como `catalog_item_id`); `ck_stock_lots_item` trocado: `item_kind IN ('medication','immunobiological','vector_control','supply')`, `(item_kind = 'supply') = (supply_catalog_item_id IS NOT NULL)`, `(item_kind = 'medication') = (catalog_item_id IS NOT NULL)`, `(item_kind IN ('medication','supply')) = (catmat_code IS NOT NULL)`, o resto como hoje; índice único `idx_stock_lots_supply_number (supply_catalog_item_id, lot_number) WHERE supply_catalog_item_id IS NOT NULL`.
- `stock_minimums`, `stock_openings`, `stock_movements` + `supply_catalog_item_id`; `ck_*_item` → `num_nonnulls(catalog_item_id, immunobiological_code, vector_product_id, supply_catalog_item_id) = 1`; únicos parciais `idx_stock_minimums_supply` e `idx_stock_openings_supply_once` (`stock_location_id, supply_catalog_item_id`). O CHECK do AMPP do 27b passa a `ampp_code IS NULL OR item_kind NOT IN ('vector_control','supply')`.
- `StockLot#supply?`, `StockLot#item_id` (`catalog_item_id || supply_catalog_item_id`), `StockMovement#item_id` (o mesmo).
- `Stock::Domain`: `MEDICATION_ITEM_KINDS = %w[medication supply]`; `of_lot(lot)` explícito por `item_kind` (`medication`/`supply` → MEDICATION; `immunobiological`; `vector_control`); outro valor → `ArgumentError`.
- `Ledger.record!` copia também `supply_catalog_item_id`.

- [ ] **Step 1: Write the failing test** (`type: :model`): "lote `supply` com material e CATMAT passa"; "lote `supply` com `catalog_item_id` → CHECK"; "lote `supply` sem CATMAT → CHECK"; "lote `medication` sem CATMAT → CHECK (como antes)"; "dois lotes do mesmo material com o mesmo número → único"; "mínimo e abertura com duas chaves → CHECK"; "abertura do mesmo material duas vezes no local → único"; "movimento de lote `supply` grava a chave"; "AMPP em lote de insumo → CHECK". `domain_spec.rb`: "`of_lot` de insumo é MEDICATION e `allows?` no `central` e no `unit`; insumo numa sala de vacina → falso"; "`item_kind` desconhecido → ArgumentError".
- [ ] **Step 2:** `bin/test api -w mod27s spec/models/supply_stock_schema_spec.rb spec/services/stock/domain_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`AddSupplyStockKey`, espelhos das listas na migração; dump `define(version: 2026_11_27_900001)`).
- [ ] **Step 4:** os mesmos specs + `spec/models/pharmacy_schema_spec.rb spec/models/bnafar_stock_schema_spec.rb spec/services/city_schema_spec.rb spec/services/pharmacy/ledger_spec.rb spec/requests/pharmacy/vector_control_stock_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add supply as the fourth stock item key" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 3 · 20261127900001: supply nos CHECKs de lote, mínimo, abertura e movimento; of_lot com lista explícita."

---

### Task 4: Entrada, abertura, busca e formas JSON do insumo

**Files:**
- Create: `spec/requests/pharmacy/supply_stock_spec.rb`
- Modify: `app/commands/pharmacy/record_entry.rb`, `app/services/pharmacy/{json,statement}.rb`, `app/controllers/pharmacy/{items,entries,transfers,adjustments}_controller.rb`, `spec/commands/pharmacy/record_entry_spec.rb`

**Interfaces:**
- Consumes: Tasks 2–3.
- `RecordEntry.call(…, supplies: false)`: no domínio MEDICATION a linha traz **um** de `catalog_item_id` ou `supply_catalog_item_id` (os dois → `invalid_request` path `items[i]`); `supply_catalog_item_id` com `supplies: false` → `item_not_allowed_here`; nos outros domínios `supply_catalog_item_id` entra em `FOREIGN_KEYS`. O material precisa estar `active` no `catalog` (`invalid_item` path `items[i].supply_catalog_item_id`); o lote nasce `item_kind: "supply"`, com `catmat_code` e `dispensing_unit` = `supply_unit`; chave de item `"supply:<id>"` (`SUPPLY_KEY`), índice `idx_stock_lots_supply_number`, abertura em `idx_stock_openings_supply_once` (entra em `OPENING_INDEXES`). Os campos BNAFAR do 27b valem igual, sem VMP (AMPP nunca exigido).
- `Pharmacy::Json.supply_item(item, dispensing_unit:) -> { catalog_item_id: nil, catmat_code, supply_catalog_item_id, item_kind: "supply", name: item.description, unit: item.supply_unit, dispensing_unit, group, controlled: false, controlled_list: nil }`; `Json.lot` com ramo `lot.supply?`; `Json.catalog_for(ids)` passa a devolver `MedicationCatalogItem` **e** `SupplyCatalogItem` pelos ids (uuid; nunca colidem). Os chamadores do estoque trocam `catalog_item_id` por `item_id` (entries `:64`, transfers `:135`, adjustments `:128`, statement `:34`).
- `POST /pharmacy/items/search { q, item_kind: "supply" }` (D9): com `pharmacy_supplies` utilizável (senão 403 `feature_disabled` `feature: "pharmacy_supplies"`), `Supplies::Search` em `Json.supply_item`; `item_kind` diferente → 422 path `item_kind`; `item_kind` junto com `domain` → 422 path `item_kind`.
- `EntriesController` lê `Supplies::Gate.usable?` fora e passa `supplies:`; o `catalog_for(items)` inclui os `supply_catalog_item_id`.

- [ ] **Step 1: Write the failing tests.** `record_entry_spec.rb`: "nota com uma linha de losartana e uma de lanceta grava dois lotes, a lanceta `supply` em `unidade`"; "linha com as duas chaves → `invalid_request` `items[0]`"; "lanceta com `supplies: false` → `item_not_allowed_here`"; "lanceta na rede de frio → `item_not_allowed_here`"; "abertura de lanceta duas vezes → `opening_exists` path `items[0].supply_catalog_item_id`"; "material `removed` → `invalid_item`". `supply_stock_spec.rb` (`type: :request`): "busca `item_kind: supply` acha a tira; sem `pharmacy_supplies` → 403 com a chave"; "nota com insumo responde o `<lot>` com `item_kind: supply`"; "transferência do central para a unidade leva o lote de insumo e o recebimento soma"; "ajuste de inventário de um lote de insumo responde o item"; "extrato do lote de insumo"; "a resposta de medicamento não muda (mesmas chaves)".
- [ ] **Step 2:** `bin/test api -w mod27s spec/commands/pharmacy/record_entry_spec.rb spec/requests/pharmacy/supply_stock_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/commands/pharmacy spec/requests/pharmacy` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: receive, open and search supply lots in the pharmacy stock" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 4 · entrada e abertura com supply_catalog_item_id; busca item_kind supply (D9); Json.catalog_for com os dois catálogos."

---

### Task 5: Posição e mínimos com o insumo

**Files:**
- Modify: `app/services/pharmacy/positions.rb`, `app/controllers/pharmacy/{positions,minimums}_controller.rb`, `app/commands/pharmacy/save_minimums.rb`, `spec/requests/pharmacy/positions_spec.rb`, `spec/commands/pharmacy/save_minimums_spec.rb`

**Interfaces:**
- `Pharmacy::Positions`: a chave de item das CTEs passa a `COALESCE(x.catalog_item_id, x.supply_catalog_item_id)` (o alias `catalog_item_id` fica como chave da posição e do cursor); `present` usa `Json.supply_item` (unidade = a do lote) para material; `item_ids_matching` procura nos dois catálogos.
- `GET /pharmacy/positions` aceita `supply_catalog_item_id` (uuid; 422 path) além de `catalog_item_id`; sem `pharmacy_supplies`, as posições de insumo **continuam** saindo (o lote existe; desligar não esconde estoque).
- `SaveMinimums`: linha `{ supply_catalog_item_id, minimum }` como a de medicamento (os dois → `invalid_request` `minimums[i]`); material inativo → `invalid_item`.

- [ ] **Step 1: Write the failing tests.** `positions_spec.rb`: "a unidade com losartana e lanceta lista as duas posições em ordem estável, a lanceta com `item_kind: supply`"; "`supply_catalog_item_id` filtra; `catalog_item_id` não casa o insumo"; "`q=lanceta` acha a posição do insumo"; "cursor de duas páginas misturando os dois"; "mínimo de lanceta acima do saldo → `below_minimum`". `save_minimums_spec.rb`: "linha de insumo grava; as duas chaves → `invalid_request`".
- [ ] **Step 2:** `bin/test api -w mod27s spec/requests/pharmacy/positions_spec.rb spec/commands/pharmacy/save_minimums_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/pharmacy spec/services/pharmacy` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: show supply positions and minimums" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 5 · posição e mínimos com o insumo (chave COALESCE, filtro supply_catalog_item_id)."

---

### Task 6: Insumo no envio à BNAFAR (27b)

**Files:**
- Modify: `app/services/bnafar/{payload,snapshot,pendencies,units}.rb`, `app/services/opening_imports/validator.rb`, `spec/services/bnafar/{payload,snapshot,pendencies,mappings}_spec.rb`, `spec/services/opening_imports/validator_spec.rb`

**Interfaces:**
- Consumes: os nomes do plano do 27b (Valores 5 e 7; Task 10 e 14 de lá).
- Domínio do insumo = MEDICATION → `bnafar_medications` (o `Bnafar::Gate::DOMAIN_KEYS` não muda). `item_key` `"supply:<id>"`. `Payload`: `numero` = `lot.catmat_code`, `terminologia` = `"CATMAT"`, `codigoVmp` e `codigoAmpp` `null`. `Units.factor` do insumo = 1. `Pendencies`: insumo nunca tem `product_without_code` (sempre há CATMAT); `lot_incomplete` como o medicamento. Planilha (D15): `tipo_produto` `insumo` com `codigo_produto` = CATMAT de `supply_catalog_items` ativo (`product_unknown` senão).

- [ ] **Step 1: Write the failing tests.** `payload_spec.rb`: "lote de lanceta → `numero` CATMAT, `terminologia: CATMAT`, `codigoVmp: null`". `snapshot_spec.rb`: "lanceta na unidade entra na posição com `bnafar_medications`; desligado → fora". `pendencies_spec.rb`: "lote de insumo sem preço → `lot_incomplete`; nunca `product_without_code`". `mappings_spec.rb`: "fator do insumo 1; 12.50 → nil". `validator_spec.rb`: "linha `insumo` com CATMAT ativo vira lote `supply`; CATMAT de medicamento como `insumo` → `product_unknown`".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/bnafar spec/services/opening_imports` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/jobs/bnafar spec/requests/pharmacy/opening_imports_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: send supply lots to bnafar with catmat codes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 6 · insumo no REPE/RESMPE com CATMAT pelo bnafar_medications; planilha do 27b com tipo insumo (D15)."

---

### Task 7: Migração 900002 — programas, dispensação por inscrição, ponto de estomia e fonte `supply`

**Files:**
- Create: `db/city_migrate/20261127900002_add_supply_programs.rb`, `app/models/{supply_quota_rule,supply_enrollment,supply_enrollment_event}.rb`, `spec/models/supply_programs_schema_spec.rb`
- Modify: `spec/support/supplies_helpers.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{dispensation,dispensation_item,health_unit,performed_procedure}.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/services/city_schema_spec.rb`

**Interfaces:**
- `supply_quota_rules` (D8): `program`, `position integer`, `profile_match jsonb NOT NULL DEFAULT '{}'`, `group`, `kind` (`fixed|per_prescription`), `quantity`, `factor_field` (`measurements_per_day|applications_per_day`), `period_days integer NOT NULL DEFAULT 30`, `cap`, `version integer NOT NULL`, `created_by_user_id`, `replaced_at`, `replaced_by_user_id`; CHECKs: `fixed` ⇔ `quantity` e sem `factor_field`/`cap`; `per_prescription` ⇔ `factor_field` e sem `quantity`; faixas do Valor 2. Trigger: sem DELETE; UPDATE só `replaced_at`/`replaced_by_user_id` de NULL a valor, uma vez.
- `supply_enrollments`: as colunas do contrato §3 + `closed_in_consultation_id` (FK `consultations`; `CHECK (closed_in_consultation_id IS NULL OR status = 'closed')`, Q7) + `activated_at timestamptz NOT NULL` (D6, Q8); `cid10`, `profile`, `override_reason`, `external_report` em `text` cifrado (`profile`/`external_report` com `serialize coder: JSON`; D16); `quotas jsonb NOT NULL DEFAULT '[]'`; `preferred_unit_id` FK `health_units`; `opened_by_user_id`; `version integer NOT NULL DEFAULT 1`. Único parcial `idx_supply_enrollments_one_active (patient_id, program) WHERE status = 'active'`. CHECKs: `origin = 'consultation'` ⇔ `opened_in_consultation_id IS NOT NULL`; `origin = 'external_report'` ⇒ `external_report IS NOT NULL`. Trigger: sem DELETE; `patient_id`, `program`, `origin`, `opened_in_consultation_id`, `opened_by_user_id` imutáveis; `closed_in_consultation_id` só de NULL a valor; `closed` terminal; `version` sobe a cada UPDATE (salvo só `last_withdrawal_at` e re-cifra).
- `supply_enrollment_events` (D6): `supply_enrollment_id`, `kind` (`EVENT_KINDS`), `consultation_id`, `user_id`, `origin`, `at`; só acréscimo.
- `dispensations` + `supply_enrollment_id` (FK) e `collector_name`, `collector_document` (cifrados; `CHECK ((collector_name IS NULL) = (collector_document IS NULL))`); `ck_dispensations_prescription` → `num_nonnulls(clinical_document_id, external_prescription_id, supply_enrollment_id) = 1`; coletor só com inscrição.
- `dispensation_items`: `catalog_item_id` aceita nulo; + `supply_catalog_item_id`, `supply_group` (D7), `quota_override_reason` (cifrado), `override_by_user_id`; `CHECK (num_nonnulls(catalog_item_id, supply_catalog_item_id) = 1)`; `ck_dispensation_items_source` → `num_nonnulls(prescription_item_id, external_prescription_item_id) = 1 OR (supply_catalog_item_id IS NOT NULL AND prescription_item_id IS NULL AND external_prescription_item_id IS NULL)`; `(quota_override_reason IS NULL) = (override_by_user_id IS NULL)`; `(supply_catalog_item_id IS NULL) = (supply_group IS NULL)`.
- `health_units` + `ostomy_service boolean NOT NULL DEFAULT false`, `ostomy_responsible_professional_id` (FK `professionals`); `CHECK (NOT ostomy_service OR kind <> 'ubs')`; `CHECK (ostomy_responsible_professional_id IS NULL OR ostomy_service)`.
- `performed_procedures` + `dispensation_item_id` (FK, único parcial `idx_performed_procedures_dispensation_item`); `ck_performed_procedures_source_ref` ganha `(source = 'supply') = (dispensation_item_id IS NOT NULL)`; `_domains` ganha `supply`; `PerformedProcedure::SOURCES` + `supply`; `Billing::Json.source_ref` → `{ type: "dispensation_item", id: }`.
- `CITY_KEYED_TARGETS` + `[SupplyEnrollment, :cid10]`, `[SupplyEnrollment, :profile]`, `[SupplyEnrollment, :override_reason]`, `[SupplyEnrollment, :external_report]`, `[Dispensation, :collector_name]`, `[Dispensation, :collector_document]`, `[DispensationItem, :quota_override_reason]`.
- Helpers (acrescentados ao arquivo da Task 3): `quota_rules!(program:, rules:)`, `supply_enrollment!(patient:, program:, quotas:, **over)`, `ostomy_unit!(responsible:)`.

- [ ] **Step 1: Write the failing test** (`type: :model`): "duas inscrições `active` do mesmo paciente e programa → único; uma `inactive` e uma `active` passam"; "`closed` voltando a `active` → trigger"; "trocar o paciente → trigger"; "DELETE de inscrição, evento ou regra → trigger"; "regra `fixed` com `factor_field` → CHECK"; "dispensação com receita e inscrição → CHECK"; "coletor só com nome → CHECK"; "item com as duas chaves → CHECK"; "item de insumo sem item de receita passa"; "exceção sem quem liberou → CHECK"; "UBS com `ostomy_service` → CHECK"; "procedimento `supply` sem item → CHECK"; "o CID decifrado volta; o texto no banco não contém o CID".
- [ ] **Step 2:** `bin/test api -w mod27s spec/models/supply_programs_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`AddSupplyPrograms`; dump `define(version: 2026_11_27_900002)`; bloco "Módulo 27 — programas").
- [ ] **Step 4:** o mesmo spec + `spec/models/pharmacy_schema_spec.rb spec/services/city_schema_spec.rb spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/commands/billing spec/requests/health_units_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add supply programs, enrollment dispensations and ostomy service columns" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 7 · 20261127900002: regras só de acréscimo (D8), inscrição com eventos e activated_at (D6), dispensação por inscrição, supply_group no item (D7), ponto de estomia, fonte supply."

---

### Task 8: Regras de cota e cálculo

**Files:**
- Create: `app/services/supply_programs/{programs,profile,quota}.rb`, `config/supplies/programs.yml`, `app/commands/supply_programs/save_rules.rb`, `app/controllers/pharmacy/admin/supply_quota_rules_controller.rb`, `spec/services/supply_programs/{profile,quota}_spec.rb`, `spec/requests/pharmacy/supply_quota_rules_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `SupplyPrograms::Programs.{cid_allowed?(program, cid10), groups(program)}`.
- `SupplyPrograms::Profile.parse(program, raw, prefix:) -> [Hash, [problem]]`: diabetes `insulin_type` (`INSULIN_TYPES`), `applications_per_day` e `measurements_per_day` (inteiros `PER_DAY`); ostomy `stoma_type`; chave a mais ou faltando → `invalid_profile` com o path da chave.
- `SupplyPrograms::Quota.compute(program:, profile:, rules:, overrides: []) -> [{ "rule_id", "group", "quantity", "period_days" }]`: por grupo, a **primeira** regra ativa (ordem `position`) cujo `profile_match` casa (cada chave: o valor do perfil está na lista; `{}` casa tudo); `fixed` → `quantity`; `per_prescription` → `profile[factor_field] × period_days`, limitado pelo `cap`; 0 → o grupo sai; `overrides` (`{ group, quantity }`) trocam a quantidade (grupo sem regra entra com `rule_id: nil` e `PERIOD_DAYS`). Puro.
- `SupplyPrograms::SaveRules.call(program:, rules:, expected_version:, by:) -> Result.ok(version:) | fail(:supply_quota_rules_changed | <primeiro erro>, details: { path:, problems: })`: valida tudo (`rules[i].campo`; `profile_match` com as chaves do perfil do programa e valores da lista; `group` do programa; `factor_field` só no diabetes; até `MAX_RULES`); sob `FOR UPDATE` das ativas do programa (e lock consultivo por programa para a lista vazia), `expected_version` = `MAX(version)` das ativas (0 sem nenhuma); grava as novas com `version + 1` e `replaced_at` nas antigas; `pharmacy.settings_changed` (`subject: "supply_quota_rules"`, `user_id`).
- `GET /pharmacy/admin/supply_quota_rules?program=` → `{ program, version, rules: [...] }`; `PUT` com step-up (`401 mfa_required`), corpo `{ program, expected_version, rules }`; só `municipal_admin` (403); `pharmacy_supplies` utilizável.

- [ ] **Step 1: Write the failing tests.** `quota_spec.rb`: "Campinas: tira `fixed` 50/30 → 50"; "São Paulo: tira `per_prescription` com 4 medições → 120; com `cap` 100 → 100"; "seringa `per_prescription` por aplicações com `insulin_type: none` e `profile_match { insulin_type: [nph, regular, analog] }` → sem seringa"; "duas regras para tira, a primeira casa → vale a primeira"; "override de tira 200 → 200 com o `rule_id` da regra"; "sem regras → []". `profile_spec.rb`: "`measurements_per_day: 11` → `invalid_profile` path `profile.measurements_per_day`"; "ostomia sem `stoma_type` → `invalid_profile`". `supply_quota_rules_spec.rb` (`type: :request`): "admin grava com step-up e a versão sobe; GET mostra só as ativas"; "versão velha → 409 `supply_quota_rules_changed`"; "farmacêutico → 403"; "sem step-up → 401"; "`factor_field` na ostomia → 422 path `rules[0].factor_field`"; "as regras antigas ficam com `replaced_at` (nunca apagadas)".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/supply_programs spec/requests/pharmacy/supply_quota_rules_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: configure supply quota rules per program and compute quotas" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 8 · regras fixed/per_prescription com teto, primeira que casa vale; PUT com step-up e versão do conjunto (D8)."

---

### Task 9: Inscrição na consulta (rascunho, finalização e painel do paciente)

**Files:**
- Create: `app/services/supply_programs/{draft_input,materialize,json}.rb`, `app/controllers/supply_enrollments_controller.rb`, `spec/services/supply_programs/draft_input_spec.rb`, `spec/commands/consultations/supply_programs_finalize_spec.rb`, `spec/requests/supply_enrollments_spec.rb`
- Modify: `app/commands/consultations/{save_draft,finalize}.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/commands/consultations/save_draft_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- Consumes: Task 8 (`Profile.parse`, `Quota.compute`, `Programs`).
- `SupplyPrograms::DraftInput.call(raw, consultation:, rules:, on:) -> Result.ok(items:) | fail(<erro>, details: { path: })` (Valor 5, D4): só consulta `general` (outra → `invalid_request` path `supply_programs`); autor com CBO em `ENROLLER_CBO_PREFIXES` (senão `enroller_not_allowed`); `cid10` por `ClinicalTerms.cid10_code` (`invalid_cid10`) e `Programs.cid_allowed?` (`cid_not_in_program`); `valid_until` entre `on >> 3` e `on >> 12` (`invalid_validity`); `preferred_unit_id` unidade ativa; `renew`/`close` com `enrollment_id` do paciente (`not_found` path `supply_programs[i].enrollment_id`; `close` de `closed` → `enrollment_not_active`); `quota_overrides` com `override_reason` 10..500; cada linha sai com `quotas` calculada. Path `supply_programs[i].<campo>`, sem valor.
- `SaveDraft`: `params["supply_programs"]` com `SupplyPrograms::Gate.usable?` (lido fora; senão 403 `feature_disabled` `supply_programs`); regras lidas fora da transação; `items["supply_programs"] = …` (substitui a lista); a resposta do `PATCH /consultations/:id` traz a lista gravada, com a `quotas` calculada de cada linha (Q7).
- `SupplyPrograms::Materialize.call(consultation:, patient:, by:, rules:, now:) -> nil | Result.fail(:enrollment_exists | <erro de revalidação>, details: { path: })` dentro do `Finalize`, **depois** do `DentalCare::Materialize` e **antes** do `update!` de finalizado (o `Finalize` lê `SupplyPrograms::Gate` e as regras fora e devolve a falha com rollback, como o `materialize!`): trava as inscrições do paciente por id (depois do paciente); revalida (`valid_until`, cota recalculada pelas regras de agora); `open` → `active`, `origin: consultation`, `activated_at: now`; `renew` → perfil, CID, validade, cota e `renewed_in_consultation_id` novos, `status: active`, `activated_at: now`; `close` → `closed` com `closed_in_consultation_id` (Q7); um `supply_enrollment_events` por linha; `RecordNotUnique` no único parcial → `enrollment_exists`. Eventos `supply_enrollment.{opened,renewed,closed}` (`supply_enrollment_id`, `consultation_id`, `user_id`).
- `GET /patients/:id/supply_enrollments` (molde `PatientMedicationsController#index`: `ClinicalRecord::Access` + `ClinicalRecord::Trail.viewed!`) → `{ items: [SupplyPrograms::Json.summary], next_before: null }` (`id, program, status, valid_until, preferred_unit, quotas, last_withdrawal_at, version`; sem CID, perfil nem laudo); `GET /supply_enrollments/:id` → `SupplyPrograms::Json.detail` (+ `cid10`, `profile`, `origin`, `external_report`). Com `supply_programs` utilizável.

- [ ] **Step 1: Write the failing tests.** `draft_input_spec.rb`: "médico abre diabetes E11 com NPH e 4 medições → linha com `quotas` calculada"; "enfermeiro (223505) passa; dentista → `enroller_not_allowed`"; "I10 no diabetes → `cid_not_in_program` path `supply_programs[0].cid10`"; "validade de 2 meses → `invalid_validity`"; "override sem motivo → `invalid_request` path `supply_programs[0].override_reason`"; "consulta `urgency` → `invalid_request` `supply_programs`". `save_draft_spec.rb`: "com `supply_programs` desligado → 403 com a chave; o resto do rascunho não muda"; "o `PATCH` responde `draft_items.supply_programs[0].quotas` com a cota calculada, e a `quotas` do cliente é ignorada (Q7)". `supply_programs_finalize_spec.rb` (`type: :model`): "abrir cria `active` com a cota congelada, evento `opened` e `activated_at`"; "renovar recalcula a cota pelas regras de agora e grava `renewed_in_consultation_id`"; "encerrar → `closed` com `closed_in_consultation_id`"; "inscrição ativa criada pela farmácia entre o rascunho e a finalização → `enrollment_exists`, a consulta segue `draft` e nada gravado (Review Focus 4)"; "consulta odontológica não passa pelo bloco"; "falha no `Attendances::Close` desfaz a inscrição". `supply_enrollments_spec.rb` (`type: :request`): "o médico com acesso vê a lista sem `cid10` nem `profile` e grava trilha"; "o detalhe traz o CID"; "sem acesso ao prontuário → 403"; "sem `supply_programs` → 403".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/supply_programs/draft_input_spec.rb spec/commands/consultations spec/requests/supply_enrollments_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/consultations_spec.rb spec/requests/dental_consultations_spec.rb spec/architecture/current_city_assignment_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: open, renew and close supply enrollments on consultation finalize" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 9 · bloco supply_programs (D4) com cota à vista; Materialize depois do dental e antes do finalized; GET /patients/:id/supply_enrollments e detalhe (D5)."

---

### Task 10: Saldo, inscrição pela farmácia e leitura de retirada

**Files:**
- Create: `app/services/supply_programs/{balance,external_report}.rb`, `app/commands/supply_programs/record_external.rb`, `app/controllers/pharmacy/supply_enrollments_controller.rb`, `spec/services/supply_programs/balance_spec.rb`, `spec/requests/pharmacy/supply_enrollments_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `SupplyPrograms::Balance.for(enrollment, group, now:) -> Data(quantity:, delivered:, available:, period_start:, period_end:)` e `Balance.all(enrollment, now:) -> { group => Data }`: retiradas = itens de dispensação da inscrição com o `supply_group`, líquidos de devolução (`Dispensed.returned_totals`), só os de líquido > 0, em ordem de `dispensed_at`; o ciclo começa no dia local da primeira retirada; uma retirada no dia `>= início + period_days` abre ciclo novo nesse dia; sem ciclo vigente hoje → `available = quantity`, `period_start` nulo; a devolução abate a retirada original (fica no ciclo dela). Soma todas as unidades (decisão 16).
- `SupplyPrograms::ExternalReport.parse(raw) -> [Hash, [problem]]`: `professional_name` 3..120, `council` em `REPORT_COUNCILS`, `council_state` (`UF`), `council_number` (`COUNCIL_NUMBER`), `issued_on` não futura (path `external_report.<campo>`).
- `SupplyPrograms::RecordExternal.call(patient:, attrs:, by:, rules:, now:) -> Result.ok(enrollment:) | fail(...)`: `program`, `cid10`, `profile`, `preferred_unit_id`, `external_report`, `valid_until` (`> hoje` e `<= hoje >> 3`, senão `invalid_validity`); trava o paciente; ativa existente → 409 `enrollment_exists`; `inactive` do mesmo programa → reativa (renova, evento `renewed`, `origin` fica); senão cria `origin: external_report` (evento `opened`). `Pharmacy::Trail.person_viewed!`.
- `POST /pharmacy/supply_enrollments` `{ location_id, patient_id, program, cid10, profile, preferred_unit_id, valid_until, external_report }`: farmacêutico de um `unit` ativo (403); `supply_programs` utilizável; 201 `SupplyPrograms::Json.summary`.
- `GET /pharmacy/supply_enrollments?patient_id=` (30/min, `query_only!(:patient_id)`, dispensador em algum `unit`): inscrições `active` e `inactive` do paciente, cada uma com `quotas[].{ group, quantity, period_days, available, period_start, period_end }` e `expired` (`valid_until < hoje`); `devices: []` (a Task 14 preenche); trilha quando lista alguma; sem CID nem perfil.

- [ ] **Step 1: Write the failing tests.** `balance_spec.rb` (relógio fixo): "cota 100/30: 60 no dia 1 → 40"; "+40 no dia 29 → 0"; "dia 30 sem retirada → 100 e `period_start` nulo (não acumula)"; "50 no dia 31 → ciclo novo, 50"; "devolução de 10 da retirada do dia 29, feita no dia 35 → o ciclo novo segue 50 (Review Focus 1)"; "retirada devolvida por inteiro não abre ciclo"; "retiradas em duas unidades somam". `supply_enrollments_spec.rb` (`type: :request`): "farmacêutico inscreve com laudo de 3 meses → 201 `origin: external_report`"; "validade de 4 meses → `invalid_validity`"; "conselho `CRO` → path `external_report.council`"; "auxiliar → 403"; "ativa existente → 409 `enrollment_exists`"; "inativa → reativada, mesmo id, evento `renewed`"; "GET traz o saldo do ciclo e `expired`; grava trilha; sem CID"; "GET com `cpf=` na query → 422".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/supply_programs/balance_spec.rb spec/requests/pharmacy/supply_enrollments_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/supply_programs` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: enroll from an external report and show supply balances" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 10 · saldo por ciclo a partir da primeira retirada, sem acúmulo, devolução no ciclo original; inscrição por laudo até 3 meses e reativação."

---

### Task 11: Retirada por inscrição — `Pharmacy::DispenseSupplies`

**Files:**
- Create: `app/commands/pharmacy/dispense_supplies.rb`, `app/controllers/pharmacy/{supply_dispensations,location_pharmacists}_controller.rb`, `spec/commands/pharmacy/dispense_supplies_spec.rb`, `spec/requests/pharmacy/supply_dispensations_spec.rb`
- Modify: `app/services/pharmacy/{json,dispensed}.rb`, `app/controllers/pharmacy/{dispensations,base}_controller.rb`, `config/routes.rb`, `spec/requests/pharmacy/dispensations_spec.rb`

**Interfaces:**
- Consumes: Tasks 4, 7, 10.
- `Pharmacy::DispenseSupplies.call(location:, enrollment:, items:, collector:, override:, by:, role:, catalog:, billing:, now:) -> Result.ok(dispensation:) | fail(<erro>, details: { path:, problems: })` (D10: `override`, `catalog` e `billing` além da assinatura do contrato). Linhas `{ supply_catalog_item_id, lot_id, quantity }` (1..`MAX_ITEMS`): material ativo do `catalog` (`invalid_item`), grupo nos `quotas` da inscrição (`item_not_in_program`), lote do material, não vencido (`lot_expired`), quantidade **inteira** positiva (`invalid_quantity`). Inscrição `active` (`enrollment_not_active`) e `valid_until >= hoje` (`enrollment_expired`). Saldo por grupo somando as linhas: acima → `quota_exceeded` path `items[i].quantity`, salvo `override`. `collector` `{ name, document }` ou nulo (`COLLECTOR_*`, path `collector.<campo>`). Travas: lotes → local → inscrição `FOR UPDATE`; saldo e estado relidos sob elas. Grava `Dispensation` (`supply_enrollment_id`, coletor), um `DispensationItem` por linha (`supply_catalog_item_id`, `supply_group`, motivo e `override_by_user_id` nas linhas acima do saldo) e o movimento `dispensing`; `enrollment.update!(last_withdrawal_at: now)`; `dispensation.recorded` (`dispensation_id`, `user_id`).
- `override` `{ reason, pharmacist_user_id?, code? }` (D10): motivo 10..500; dispensador `pharmacist` → vale a própria sessão (sem código); `pharmacy_assistant` → `pharmacist_user_id` com papel `pharmacist` no local e `code` TOTP válido e não repetido (`Mfa::Verify.totp_step_for` + `consume_totp_step!`), senão 401 `override_code_invalid`; sem `pharmacist_user_id` → 403 `override_requires_pharmacist`. O `override_by_user_id` é o farmacêutico.
- `GET /pharmacy/locations/:id/pharmacists` (Q10): dispensador do local `unit` ativo (403 senão); `{ items: [ { user_id, name } ], next_before: null }` com quem tem o papel `pharmacist` válido no local (`Pharmacy::Access`), em ordem de nome; sem e-mail nem CPF.
- `POST /pharmacy/supply_dispensations` `{ location_id, enrollment_id, items, collector?, override? }`: dispensador do `unit` ativo; `pharmacy_supplies` utilizável; inscrição inexistente → 404; o catálogo e o `billing` (`Billing::Gate.enabled?`) lidos fora; 201 `<dispensation>`.
- `Json.dispensation`: com inscrição, `prescription: null` e `supply_enrollment: { id, program }`, `collector: { name } | null` (nunca o documento); item de insumo com `Json.supply_item`. `Dispensed.last_on` ignora linha sem `catalog_item_id`. O histórico `GET /pharmacy/dispensations` passa a ler os dois catálogos (`Json.catalog_for` por `item_id`).

- [ ] **Step 1: Write the failing tests.** `dispense_supplies_spec.rb`: "entrega 50 tiras e 50 lancetas dentro da cota: um movimento por linha, `supply_group` gravado, `last_withdrawal_at` atualizado"; "51 tiras com cota 50 → `quota_exceeded` path `items[0].quantity`, nada gravado"; "duas linhas de tira que juntas passam → a segunda recusada"; "fralda (`other`) → `item_not_in_program`"; "inscrição vencida → `enrollment_expired`; `inactive` → `enrollment_not_active`"; "`1.5` → `invalid_quantity`"; "duas entregas concorrentes do último saldo: uma passa, a outra `quota_exceeded` (lock da inscrição)"; "farmacêutico com `override` e motivo passa e grava o motivo cifrado". `supply_dispensations_spec.rb` (`type: :request`): "auxiliar com o código do farmacêutico do local passa; com o código de farmacêutico de outro local → 401; o mesmo código duas vezes → 401 na segunda; sem `pharmacist_user_id` → 403 (Review Focus 2)"; "retirada por terceiro grava nome e documento cifrados e responde só o nome"; "sem `pharmacy_supplies` → 403"; "`GET /pharmacy/locations/:id/pharmacists` lista só os farmacêuticos válidos do local; auxiliar de outro local → 403". `dispensations_spec.rb`: "o histórico com uma dispensação por receita e uma por inscrição responde as duas, a de inscrição com `prescription: null`"; "`Dispensed.last_on` não ganha chave nula (Review Focus 3)".
- [ ] **Step 2:** `bin/test api -w mod27s spec/commands/pharmacy/dispense_supplies_spec.rb spec/requests/pharmacy/supply_dispensations_spec.rb spec/requests/pharmacy/dispensations_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`FAILURE_STATUS` + `enrollment_exists`, `supply_enrollment_changed`, `override_requires_pharmacist: :forbidden`, `override_code_invalid: :unauthorized`).
- [ ] **Step 4:** os mesmos specs + `spec/commands/pharmacy spec/requests/pharmacy spec/requests/patient_medications_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: dispense supplies by program enrollment within quota" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 11 · POST /pharmacy/supply_dispensations com cota, exceção (farmacêutico ou TOTP dele, D10) e terceiro cifrado; histórico e last_on intactos para medicamento."

---

### Task 12: Devolução de insumo

**Files:**
- Modify: `app/commands/pharmacy/record_return.rb`, `app/controllers/pharmacy/returns_controller.rb`, `spec/commands/pharmacy/record_return_spec.rb`, `spec/requests/pharmacy/returns_spec.rb`

**Interfaces:**
- `RecordReturn.call` aceita item com `supply_catalog_item_id`; `supply_group == "ostomy_bag"` → `ostomy_return_not_allowed` path `dispensation_item_id`, antes das outras recusas; o resto igual (mesmo lote, `return`, `dispensation.returned`). `RecordReturn.controlled?` é falso para insumo. A resposta usa `Json.catalog_for` por `item_id`. O saldo da cota volta pela Task 10 (lê `returned_totals`).

- [ ] **Step 1: Write the failing tests.** `record_return_spec.rb`: "devolve 10 lancetas e o saldo do ciclo da retirada sobe 10"; "bolsa → `ostomy_return_not_allowed`, nada gravado"; "mais que o entregue → `return_exceeds`". `returns_spec.rb`: "auxiliar devolve tira (não é controlado) e recebe o `<dispensation>` com `returned_total`".
- [ ] **Step 2:** `bin/test api -w mod27s spec/commands/pharmacy/record_return_spec.rb spec/requests/pharmacy/returns_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/supply_programs/balance_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: return supplies to quota and refuse ostomy bag returns" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 12 · devolução de insumo volta à cota; bolsa → ostomy_return_not_allowed."

---

### Task 13: Bolsa de estomia — ponto de estomia, conferências e `Billing::FromSupply`

**Files:**
- Create: `app/commands/billing/from_supply.rb`, `app/jobs/billing/supply_production_job.rb`, `spec/commands/billing/from_supply_spec.rb`, `spec/jobs/billing/supply_production_job_spec.rb`, `spec/requests/health_units_ostomy_spec.rb`
- Modify: `app/controllers/health_units_controller.rb`, `app/commands/pharmacy/dispense_supplies.rb`, `config/initializers/domain_events.rb`, `spec/commands/pharmacy/dispense_supplies_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `HealthUnitsController`: `assign_ostomy(unit)` no `create` e no `update_locked` (molde `assign_caps`): `ostomy_service` booleano; `true` numa UBS → `ostomy_service_not_allowed_in_ubs`; sem CNES → `ostomy_service_requires_cnes`; `ostomy_responsible_professional_id` profissional com vínculo ativo na unidade (`Billing::Staff.linked?`) → senão `ostomy_responsible_not_linked` (D12); desligar o ponto limpa o responsável. `unit_json` + `ostomy_service`, `ostomy_responsible: { id, name } | null`.
- `DispenseSupplies` (com as linhas `ostomy_bag`): a unidade do local sem `ostomy_service` → `not_ostomy_service` path `location_id` (sempre). Com `billing: true` (D11): material sem `sigtap_code` → `supply_without_sigtap` path `items[i].supply_catalog_item_id`; paciente sem CNS → `patient_without_cns` path `enrollment_id`; soma do mês local (procedimentos `supply` `recorded` do paciente com o mesmo `sigtap_code` na competência + as linhas) acima do `SigtapCache#max_quantity` da release da competência → `sigtap_max_quantity` path `items[i].quantity`; sem `max_quantity` → sem limite.
- `Billing::FromSupply.call(dispensation) -> { created: Integer }`: itens com `sigtap_code` no material (lido da plataforma) e sem procedimento (`dispensation_item_id`); `Record.insert!` com `source: "supply"`, `dispensation_item:`, estabelecimento `Establishment.for_unit(location.health_unit)`, `professional: responsible.user`, `cbo_code: Staff.cbo_for(responsible.user, establishment)`, `performed_on` = dia local de `dispensed_at`, quantidade inteira, `cid10` da inscrição, `character: "elective"`, `recorded_by: nil`. Sem responsável → `ResponsibleMissing` (molde `BillingUnitMissing`, `retry_on`). Erro do banco → `InsertFailed` sem a causa.
- `Billing::SupplyProductionJob` (`IdempotentConsumer`, `handle(dispensation_id:, **)`; `return unless Gate.enabled?(Current.city)`; só dispensação com inscrição); `DomainEvents.bind "dispensation.recorded", to: [ Billing::SupplyProductionJob ]`.

- [ ] **Step 1: Write the failing tests.** `health_units_ostomy_spec.rb` (`type: :request`): "admin marca um ambulatório com CNES e responsável com vínculo → `ostomy_service: true`"; "UBS → 422 `ostomy_service_not_allowed_in_ubs`"; "sem CNES → `ostomy_service_requires_cnes`"; "responsável sem vínculo → `ostomy_responsible_not_linked`". `dispense_supplies_spec.rb`: "bolsa numa UBS → `not_ostomy_service`"; "com faturamento, paciente sem CNS → `patient_without_cns`; bolsa sem SIGTAP → `supply_without_sigtap`"; "teto 30: 20 num ponto e 15 em outro no mesmo mês → a segunda `sigtap_max_quantity`; no mês seguinte passa (Review Focus 5)"; "sem faturamento: entrega sem CNS passa". `from_supply_spec.rb`: "a bolsa vira uma linha `supply` BPA-I com CNS, CID da inscrição, CBO do responsável e a unidade; a lanceta não vira linha"; "reprocessar não duplica"; "sem responsável → `ResponsibleMissing`". `supply_production_job_spec.rb` (`type: :job`): "evento com faturamento desligado → nada"; "dispensação por receita → nada". Binding no spec dos eventos.
- [ ] **Step 2:** `bin/test api -w mod27s spec/requests/health_units_ostomy_spec.rb spec/commands/pharmacy/dispense_supplies_spec.rb spec/commands/billing/from_supply_spec.rb spec/jobs/billing/supply_production_job_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/commands/billing spec/jobs/billing spec/requests/billing spec/requests/health_units_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture/current_city_assignment_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: bill ostomy bag deliveries as bpa-i procedures" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 13 · ponto de estomia na unidade (CNES e responsável com vínculo, D12); not_ostomy_service sempre; CNS, SIGTAP e teto só com procedure_billing (D11); Billing::FromSupply."

---

### Task 14: Migração 900003 — comodato de aparelhos

**Files:**
- Create: `db/city_migrate/20261127900003_add_device_loans.rb`, `app/models/{device_model,device_loan}.rb`, `app/commands/device_models/save.rb`, `app/commands/device_loans/{lend,give_back,replace,report_loss}.rb`, `app/controllers/pharmacy/{device_loans,device_models}_controller.rb`, `app/controllers/pharmacy/admin/device_models_controller.rb`, `spec/models/device_loan_schema_spec.rb`, `spec/requests/pharmacy/device_loans_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `app/controllers/pharmacy/supply_enrollments_controller.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `device_models`: `name` (3..120, único sem caixa), `kind` (`DEVICE_KINDS`), `supplier` (até 120), `compatible_supply_catalog_item_id uuid`, `active`, `version`. `device_loans`: as colunas do contrato §5 + `created_by_user_id`, `term_accepted_at NOT NULL`; único parcial `idx_device_loans_active_serial (serial_number) WHERE status = 'active'`; `CHECK ((status = 'lost') = (police_report_number IS NOT NULL))`; `CHECK ((status = 'replaced') = (replaced_by_loan_id IS NOT NULL))`; `CHECK ((status = 'active') = (returned_at IS NULL))`. Trigger: sem DELETE; só de `active` para `returned|replaced|lost`, com `version + 1`. `CITY_KEYED_TARGETS` + `[DeviceLoan, :police_report_number]`.
- `DeviceModels::Save.call(model: nil, attrs:, expected_version: nil, by:)`; `GET|POST /pharmacy/admin/device_models`, `PATCH /pharmacy/admin/device_models/:id` (`municipal_admin`, step-up; D13); `GET /pharmacy/device_models` (dispensador; só ativos).
- `DeviceLoans::Lend.call(location:, patient:, attrs:, by:, now:)` (`device_model_id` ativo, `serial_number` (`SERIAL`), `term_accepted: true` → senão `term_required`); `GiveBack.call(loan:, expected_version:, by:, now:)`; `Replace.call(loan:, attrs:, expected_version:, by:, now:)` (cria o novo ativo, o antigo `replaced` com `replaced_by_loan_id`, mesma transação); `ReportLoss.call(loan:, police_report_number:, expected_version:, by:, now:)`. Serial ativo repetido → 409 `serial_in_use`; versão → 409 `device_loan_changed`. Evento `device_loan.changed` (`device_loan_id`, `status`, `user_id`).
- Rotas (dispensador do `unit` ativo; `health_unit_id` = a unidade do local): `POST /pharmacy/device_loans` `{ location_id, patient_id, device_model_id, serial_number, term_accepted }`; `POST /pharmacy/device_loans/:id/{return,replace,loss}`; `GET /pharmacy/device_loans?health_unit_id=&status=&before=&limit=` (sem nome do paciente nem boletim; `patient_id` e o aparelho) e `GET /pharmacy/device_loans?patient_id=` (nominal, trilha). `GET /pharmacy/supply_enrollments` ganha `devices: [{ loan_id, model, serial_number, compatible_item: <item> | null }]` dos empréstimos ativos.

- [ ] **Step 1: Write the failing tests.** Schema: "serial ativo repetido → único; o mesmo serial depois de devolvido passa"; "`lost` sem boletim → CHECK"; "voltar de `returned` a `active` → trigger". `device_loans_spec.rb` (`type: :request`): "auxiliar empresta com termo; sem termo → `term_required`"; "troca: o antigo `replaced` e o novo ativo, mesmo paciente"; "perda grava o boletim cifrado e a lista não o mostra"; "versão velha → 409"; "lista por unidade pagina com `next_before`"; "a retirada mostra a tira compatível do aparelho ativo"; "admin cria modelo com step-up; farmacêutico → 403".
- [ ] **Step 2:** `bin/test api -w mod27s spec/models/device_loan_schema_spec.rb spec/requests/pharmacy/device_loans_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_27_900003)`; bloco "Módulo 27 — comodato").
- [ ] **Step 4:** os mesmos specs + `spec/requests/pharmacy/supply_enrollments_spec.rb spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: track glucometer and reader loans with term, return, replacement and loss" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 14 · 20261127900003: comodato com serial único entre ativos, troca e perda com boletim cifrado; modelos pelo admin (D13); tira compatível na retirada."

---

### Task 15: Inativação em 180 dias

**Files:**
- Create: `app/jobs/supply_programs/inactivate_job.rb`, `spec/jobs/supply_programs/inactivate_job_spec.rb`
- Modify: `config/recurring.yml`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `SupplyPrograms::InactivateJob` (`prepend EachCityJob`, `:housekeeping`, `"every day at 3:15am America/Sao_Paulo"`): `return unless SupplyPrograms::Gate.usable?(city)` (lido antes da cidade); `active` com `GREATEST(last_withdrawal_at, activated_at) < now - INACTIVE_AFTER_DAYS.days` → `inactive`, em lotes de 500 por id com `FOR UPDATE SKIP LOCKED`; evento `supply_enrollment.inactivated` (`supply_enrollment_id`) e `supply_enrollment_events` `inactivated`. Idempotente.

- [ ] **Step 1: Write the failing test** (`type: :job`): "sem retirada há 181 dias → `inactive` com evento"; "aberta há 200 dias com retirada há 10 → fica"; "renovada há 5 dias depois de 300 sem retirada → fica"; "rodar duas vezes não duplica evento"; "interruptor desligado → nada".
- [ ] **Step 2:** `bin/test api -w mod27s spec/jobs/supply_programs/inactivate_job_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec + `spec/architecture/current_city_assignment_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: inactivate supply enrollments after 180 days without withdrawal" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 15 · InactivateJob diário (180 dias desde a última retirada ou ativação)."

---

### Task 16: Migração 900004 — importação dos pacientes atuais

**Files:**
- Create: `db/city_migrate/20261127900004_add_supply_enrollment_imports.rb`, `app/models/{supply_enrollment_import,supply_enrollment_import_row}.rb`, `app/services/supply_enrollment_imports/{reader,validator}.rb`, `app/commands/supply_enrollment_imports/{upload,apply,discard}.rb`, `app/controllers/pharmacy/supply_enrollment_imports_controller.rb`, `spec/fixtures/supply_enrollment_imports/{ok,excel,latin1}.csv`, `spec/services/supply_enrollment_imports/{reader,validator}_spec.rb`, `spec/commands/supply_enrollment_imports/apply_spec.rb`, `spec/requests/pharmacy/supply_enrollment_imports_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `app/commands/patients/lookup_by_identifier.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/commands/patients/lookup_by_identifier_spec.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- Cabeçalho (D14), exato: `programa;cpf;cns;cid10;tipo_insulina;aplicacoes_dia;medicoes_dia;tipo_estoma;validade;cnes_unidade`. UTF-8 (BOM aceito; Latin-1 → `invalid_file`), `;`, CRLF ou LF, RFC 4180; até 2 MB decodificados (`file_too_large`) e 5.000 linhas. Erros por linha: `invalid_program`, `invalid_cpf`, `invalid_cns`, `person_not_found`, `citizen_not_verified`, `invalid_cid10`, `cid_not_in_program`, `invalid_profile`, `invalid_validity` (validade `> hoje` e `<= hoje >> 12`), `unit_unknown`, `enrollment_exists`, `duplicate_line`.
- Tabelas (molde `opening_imports` do 27b): `supply_enrollment_imports` (`status uploaded|validated|applied|discarded`, `file_sha256`, `file_data` cifrado, `rows_total`, `rows_valid`, `report jsonb` só códigos e contagens, `uploaded_by_user_id`, `applied_by_user_id`, `applied_at`, `version`); `supply_enrollment_import_rows` (`line`, `status valid|error|applied`, `error_codes text[]`, `supply_enrollment_id`; sem os campos lidos: a linha é relida do arquivo no apply). Trigger: sem DELETE; status só para a frente; `file_data` só vira NULL.
- `Patients::LookupByIdentifier.call(cpf:, cns:, dry_run: false)`: com `dry_run: true` nunca cria; o par validado sem paciente → `Result.ok(patient: nil, creatable: true)`.
- `Upload.call(bytes:, by:, now:)` (valida com `dry_run: true`); `Apply.call(import:, expected_version:, by:, now:)` (relê o arquivo; por linha válida, numa transação cada: `LookupByIdentifier` cria o paciente, ativa existente → linha `enrollment_exists`, senão `supply_enrollments` `origin: import` com a cota das regras de agora e evento `opened`; `applied`, `file_data = NULL`, `supply_enrollment_import.applied` (`supply_enrollment_import_id`, `user_id`)); `Discard.call(import:, expected_version:, by:)`. Versão → 409 `supply_enrollment_import_changed`. Não há desfazer (Q12): a inscrição aplicada é encerrada só na consulta.
- Rotas: `GET …/template.csv` (antes de `:id`), `POST /pharmacy/supply_enrollment_imports { file_base64 }`, `GET …/:id?before=&limit=` (relatório e linhas, sem CPF/CNS/CID), `POST …/:id/apply` (step-up), `POST …/:id/discard`. Só `municipal_admin`; `supply_programs` utilizável.

- [ ] **Step 1: Write the failing tests.** `reader_spec.rb`: "`excel.csv` (BOM, CRLF, aspas, linha vazia no fim) = as linhas do `ok.csv`"; "`latin1.csv` → `invalid_file`"; "2 MB + 1 → `file_too_large`". `validator_spec.rb`: um teste por código de linha; "par validado sem paciente → válido e nenhum paciente criado". `lookup_by_identifier_spec.rb`: "`dry_run` não cria". `apply_spec.rb`: "200 linhas → inscrições `import` com cota das regras, pacientes do par criados só agora"; "inscrição ativa criada entre o upload e o apply → linha `enrollment_exists`, o resto entra"; "`file_data` NULL depois". `supply_enrollment_imports_spec.rb` (`type: :request`): "admin envia, confere e aplica com step-up"; "apply sem step-up → 401"; "farmacêutico → 403"; "`template.csv` é o cabeçalho exato"; "o relatório não traz CPF, CNS nem CID".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/supply_enrollment_imports spec/commands/supply_enrollment_imports spec/requests/pharmacy/supply_enrollment_imports_spec.rb spec/commands/patients/lookup_by_identifier_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_27_900004)`).
- [ ] **Step 4:** os mesmos specs + `spec/requests/pharmacy/person_lookups_spec.rb spec/requests/vaccination spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: import current program patients from a reviewed spreadsheet" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 16 · 20261127900004: CSV de pacientes (D14) com conferência e apply com step-up; paciente criado só no apply (dry_run)."

---

### Task 17: Linha do tempo

**Files:**
- Create: `app/services/timeline/sources/supply_enrollment_source.rb`, `spec/services/timeline/sources/supply_enrollment_source_spec.rb`
- Modify: `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/services/timeline/sources/{dispensation_source,procedure_source,labels}.rb`, `app/controllers/pec_timelines_controller.rb`, `spec/requests/timeline_spec.rb`, `spec/requests/pec_timeline_spec.rb`, `spec/services/timeline/query_spec.rb`

**Interfaces:**
- `Timeline::KINDS` + `supply_enrollment`; `Query::PROFESSIONAL_ONLY_KINDS` + `supply_enrollment`; `SOURCES["supply_enrollment"]`; `PecTimelinesController::KINDS` sem `supply_enrollment` (D18).
- `SupplyEnrollmentSource.entries(person:, before:, limit:, from:, to:)`: um por `supply_enrollment_events` `opened|renewed|closed` (o `inactivated` fica fora: não é ato clínico), pela `at`; título `Labels::SUPPLY_ENROLLMENT_TITLES` = `{ "opened" => "Inscrição em programa de insumos", "renewed" => "Renovação em programa de insumos", "closed" => "Encerramento em programa de insumos" }`, resumo = o rótulo do programa (`"Diabetes"`, `"Estomia"`); nunca CID nem perfil; não lê interruptor (registro retido).
- `DispensationSource`: com inscrição, título `Labels::SUPPLY_DISPENSATION_TITLE = "Retirada de insumos"` e resumo pelos rótulos do material (`SupplyCatalogItem#description`), lidos de uma vez.
- `ProcedureSource`: `citizen: true` exclui `source: "supply"` (D18).

- [ ] **Step 1: Write the failing tests.** `supply_enrollment_source_spec.rb`: "abertura e renovação viram duas entradas sem CID"; "inativação não entra". `timeline_spec.rb`: "o profissional vê `supply_enrollment` e a retirada de insumos com o rótulo da lanceta"; "o cidadão (`/citizen/history`) não vê nenhum dos dois nem a linha BPA da bolsa". `pec_timeline_spec.rb`: "a página do PEC externo não traz `supply_enrollment`". `query_spec.rb`: "cursor aceita o kind novo".
- [ ] **Step 2:** `bin/test api -w mod27s spec/services/timeline spec/requests/timeline_spec.rb spec/requests/pec_timeline_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/citizen_api/history_spec.rb spec/requests/timeline_exams_spec.rb spec/requests/timeline_vaccination_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: show supply enrollments and supply withdrawals in the professional timeline" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 17 · supply_enrollment só do profissional, fora do PEC; retirada de insumos com rótulo; linha BPA da bolsa fora do cidadão (D18)."

---

### Task 18: Semente de dev para a prova no navegador

**Files:**
- Create: `lib/supplies_crew.rb`, `spec/lib/supplies_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- `SuppliesCrew.seed_platform!` (itens fictícios com CATMAT reservado 99xxxxx: tira, lanceta, seringa, agulha de caneta, bolsa com `sigtap_code` 0701050012 e um `other`); `SuppliesCrew.seed_current_city(slug:)` depois do `PharmacyCrew` e do `BillingCrew`: os dois interruptores ligados, lotes de cada insumo no central e numa UBS, regras de diabetes (São Paulo `per_prescription` com teto e Campinas `fixed`) e de estomia, um ambulatório como ponto de estomia com responsável vinculado, três inscrições (diabetes ativa com retirada, diabetes vencida, estomia ativa), um modelo de glicosímetro com a tira compatível e um empréstimo ativo, e um `ok.csv` em `tmp/` para a tela de importação. Dados fictícios e coerentes (memória `semente-dev-realista`).

- [ ] **Step 1: Write the failing test** (`type: :model`): "idempotente"; "há inscrição com saldo parcial e uma vencida"; "o ponto de estomia não é UBS e tem CNES".
- [ ] **Step 2:** `bin/test api -w mod27s spec/lib/supplies_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec + `spec/lib/pharmacy_crew_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "chore: seed supply programs dev data" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 18 · SuppliesCrew (catálogo fictício, regras SP/Campinas, ponto de estomia, inscrições, comodato, CSV)."

---

### Task 19: Sigilo, eventos e R18 (§9)

**Files:**
- Create: `spec/security/supplies_secrecy_spec.rb`

- [ ] **Step 1: Write the test** (`type: :request`, molde `vector_control_secrecy_spec.rb`, constantes num módulo `SuppliesSecrecy`, ASCII, palavras únicas): CID marcado (E109), perfil, nome e documento de quem retira, motivo de exceção, boletim, laudo externo e um CSV com CPF marcado. Cenário pelas rotas reais: regras; inscrição na consulta (rascunho e finalização); laudo externo; retirada com terceiro e exceção pelo auxiliar com TOTP; devolução; bolsa faturada; comodato com perda; importação (upload e apply); leituras da farmácia e do prontuário; `InactivateJob`. Provas:
  - `capture_log` sem CID, perfil, nome ou documento de quem retira, motivo, boletim, laudo, CPF, código TOTP nem `file_base64`;
  - eventos do módulo ⊂ `Supplies::Catalog::DOMAIN_EVENTS` + `dispensation.recorded` + `dispensation.returned` + `pharmacy.settings_changed` + `consultation.finalized` + `performed_procedure.recorded`, só ids;
  - nenhuma URL do módulo com CPF, CNS ou CID; toda lista sem CID, perfil, laudo, documento de quem retira e boletim; recusas 4xx só com `error`, `path`, `problems`, `feature`;
  - colunas cifradas no banco não contêm o texto marcado; `file_data` NULL depois do apply e do discard;
  - grep multilinha de `Platform.audit(` nos arquivos do módulo: só `supply_catalog.imported`; `maintenance.supply_catalog.imported` em `MaintenanceAudit::NAMES`; os dois no R18.
- [ ] **Step 2:** `bin/test api -w mod27s spec/security/supplies_secrecy_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture` → PASS (falha → correção na task dona, volta aqui).
- [ ] **Step 3: Commit** `git commit -m "test: pin supply programs secrecy across logs, events, lists and responses" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api insumos · Task 19 · sigilo fixado (CID, perfil, laudo, terceiro, exceção, boletim e CSV fora de log/URL/evento/lista)."

---

### Task 20: Entrega — suíte completa e passagem aos fronts (para antes do merge)

**Files:** nenhum código novo.

- [ ] **Step 1: Suíte completa:** pedir à sessão "Testes", por `SendMessage`: `api · mod27s/feat/mod-27-supplies · --full · insumos do módulo 27 (1 migração de plataforma e 4 de cidade, quarta chave do estoque do 27a/29/32b/35a/35b, 27b, Consultations::Finalize, faturamento, linha do tempo, CityEncryption, Platform::Features, spec/support, initializers)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador com o resumo e o hash, e as sessões do dashboard e do maintenance (porta 3065).
- [ ] **Step 2: Parar.** Merge, push e cards só com autorização do usuário; os cards dos insumos seguem **In Progress** até o último app de cada card; nenhum vai a Verified por este plano. Os gates de go-live do spec §6 (cotas com a assistência farmacêutica, teto das bolsas na SIGTAP da competência, carga CATMAT conferida), a correspondência SIGTAP com `review: true` e a unidade de fornecimento com `unit_review` vão ao board #2 no fechamento (outra sessão).

**Relato ao coordenador:** "api insumos · Task 20 · suíte completa <verde N/0 | falhas>; hash <…>; pronto para dashboard e maintenance; parado antes do merge."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptores e papéis (1, 9, 10, 11, 14); §2 catálogo e carga (2), estoque e `of_lot` (3–5), unidade de dispensação (2, 4), 27b (6); §3 tabela (7), consulta (9), farmácia (10), importação (16), inativação (15); §4 regras (8), saldo (10), dispensação e rotas (11), devolução (12); §5 comodato (14); §6 estomia e faturamento (7, 13); §7 linha do tempo e painel (9, 17); §8 telas — as rotas que o dashboard e o maintenance consomem (2, 4, 5, 8–16); §9 eventos e sigilo (7–16, 19); §10 testes (todas) e suíte (20); Q1–Q13 (Reconciliação; 1, 2, 7–13, 16).
2. **Testes do §10:** quarta chave (3, 5); carga 6515/6550 (2); inscrição na finalização, pela farmácia e por importação (9, 10, 16); `fixed` e `per_prescription` com teto (8); período desde a primeira retirada (10); exceção (11); devolução à cota e bolsa recusada (12); comodato (14); estomia no faturamento (13); 180 dias (15); sigilo (19); suíte (20).
3. **Nomes:** `Supplies::{Gate, Catalog, Groups, SigtapMap, Units, CatalogImport, Search}`, `SupplyPrograms::{Gate, Programs, Profile, ExternalReport, Quota, Balance, DraftInput, Materialize, Json, SaveRules, RecordExternal, InactivateJob}`, `Pharmacy::DispenseSupplies`, `Billing::{FromSupply, SupplyProductionJob}`, `DeviceModels::Save`, `DeviceLoans::{Lend, GiveBack, Replace, ReportLoss}`, `SupplyEnrollmentImports::{Reader, Validator, Upload, Apply, Discard}`, `Timeline::Sources::SupplyEnrollmentSource`, `SuppliesCrew`; `RecordEntry.call(…, supplies:)` (4).
4. **Ordem:** 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 (regras, usadas pela 9, 10 e 16) → 9 → 10 (saldo, usado pela 11) → 11 → 12 → 13 → 14 → 15 → 16 → 17 → 18 → 19 → 20.

## Divergências propostas ao contrato

> D1–D18 **resolvidas no §11 do contrato (Q6–Q13; docs `7348ae0`)**. Cada item guarda o problema, a evidência e a proposta; a marca diz qual Q resolveu e o que ela acrescenta.

D1. **[Resolvida pela Q6.]** **Releases do catálogo de materiais na tabela de medicamentos.** O §2 manda criar "a release `source: catmat_6515_6550`", mas `medication_catalog_releases` tem índices únicos globais `idx_medication_catalog_releases_one_active` e `one_importing` (`db/platform_schema.rb:407-408`), e o `Medications::CatalogImport` passa a `superseded` toda release ativa (`app/services/medications/catalog_import.rb:69`): a carga de materiais derrubaria a de medicamentos. **Proposta:** tabela própria `supply_catalog_releases` no mesmo molde (`ReleaseLifecycle.fail_stale!` já recebe a classe, `release_lifecycle.rb:11`).

D2. **[Resolvida pela Q6.] No maintenance, a revisão é só leitura e a carga não pede TOTP.** **A unidade de fornecimento não vem no item do CATMAT.** `4_consultarItemMaterial` não tem unidade (schema `DmMaterialItemDTO` da API, lido em 2026-11-24: `codigoItem … dataHoraAtualizacao`), e o `Client.build_row` só lê código, PDM e descrição (`app/services/medications/catmat/client.rb:31-35`). A unidade é da PDM, e uma PDM tem várias: `6_consultarMaterialUnidadeFornecimento?codigoPdm=402` (TIRA REAGENTE) devolve `UN`, `RO` (5 m) e `EMB` (50 UN). **Proposta:** a carga lê a tabela de unidades (paginada, 38.155 linhas ativas) e grava `supply_unit` = `unidade` quando a PDM a tem ativa, senão a de menor sequência com `unit_review: true`; a conferência entra no gate de go-live.

D3. **[Resolvida pela Q6.]** **Grupo "revisável" sem mecanismo.** O §2 diz que o grupo é "atribuído na carga por regra sobre o PDM, revisável", sem rota, mutation nem campo de revisão. **Proposta:** a revisão é editar `config/supplies/groups.yml` e recarregar; sem override manual por item.

D4. **[Resolvida pela Q7.] A resposta do `PATCH` traz a cota calculada; a inscrição ganha `closed_in_consultation_id`.** **Forma do bloco `supply_programs` do rascunho.** O §3 nomeia o bloco e o `DraftInput`, mas não diz os campos, as ações nem os tipos de consulta. **Proposta:** o Valor 5 (`open|renew|close`, `enrollment_id`, `cid10`, `profile`, `valid_until`, `preferred_unit_id`, `quota_overrides` com `override_reason`), `quotas` calculada pelo api, só na consulta `general`.

D5. **[Resolvida pela Q7.]** **Rota do painel do paciente na consulta.** O §7 manda o painel mostrar as inscrições ativas, e o §9 proíbe CID e perfil em lista; a única rota de leitura do contrato é a da farmácia. **Proposta:** `GET /patients/:id/supply_enrollments` (lista sem CID) e `GET /supply_enrollments/:id` (com CID), no molde do `PatientMedicationsController#index` (`app/controllers/patient_medications_controller.rb:21-30`).

D6. **[Resolvida pela Q8.]** **Histórico da inscrição.** A linha do tempo pede "aberta, renovada, encerrada", mas a tabela do §3 só guarda o último `renewed_in_consultation_id`, e a regra dos 180 dias precisa de uma data de ativação para quem nunca retirou. **Proposta:** `supply_enrollment_events` só de acréscimo e a coluna `activated_at`.

D7. **[Resolvida pela Q8.]** **Grupo do item na dispensação.** O grupo mora na plataforma (`supply_catalog_items.group`), e o saldo e a recusa da devolução da bolsa são da cidade. **Proposta:** `dispensation_items.supply_group`, cópia congelada na entrega.

D8. **[Resolvida pela Q8.]** **`version` por regra num `PUT` da lista.** O §4 dá `version` à regra e grava a lista por `PUT`. Como a cota congelada guarda o `rule_id`, a regra não pode sumir. **Proposta:** regras só de acréscimo com `replaced_at`; versão do conjunto = `MAX(version)` das ativas do programa; corpo `{ program, expected_version, rules }`; havendo sobreposição, vale a primeira na ordem gravada.

D9. **[Resolvida pela Q9.]** **Busca do insumo nas telas de estoque.** O §8 diz que entradas, transferências, posição e inventário "passam a buscar o insumo", mas o contrato não diz a rota nem as chaves. **Proposta:** `POST /pharmacy/items/search { q, item_kind: "supply" }`; linhas de entrada e mínimo com `supply_catalog_item_id`; posição com filtro `supply_catalog_item_id`.

D10. **[Resolvida pela Q10.] Acrescenta `GET /pharmacy/locations/:id/pharmacists`.** **Forma da exceção e assinatura do comando.** O §4 diz que "a rota recebe `override_by` com o código", sem forma, e a assinatura de `DispenseSupplies` não tem onde levá-lo. **Proposta:** `override: { reason, pharmacist_user_id?, code? }`, com o TOTP do farmacêutico por `Mfa::Verify.totp_step_for` e `consume_totp_step!` (`app/models/user.rb:66`). A assinatura ganha `override:`, `catalog:` e `billing:`.

D11. **[Resolvida pela Q11.]** **Quais conferências da bolsa dependem do faturamento.** O §1 diz que a bolsa no faturamento exige também o `procedure_billing`, e o §6 recusa sem CNS e acima do teto sem dizer se vale com o faturamento desligado. A bolsa sem SIGTAP no item não tem recusa. O `max_quantity` é o `QT_MAXIMA_EXECUCAO` (`app/services/terminology/sigtap_reader.rb:97`), usado como teto mensal. **Proposta:** `not_ostomy_service` sempre; `patient_without_cns`, `supply_without_sigtap` (novo) e `sigtap_max_quantity` só com o `procedure_billing` ligado; o teto conta o mês do paciente, a conferir no gate.

D12. **[Resolvida pela Q11.]** **Responsável da estomia sem regra de cadastro.** O `performed_procedures.professional_user_id` é NOT NULL (`db/city_schema.rb:3369`), e o CBO sai do vínculo ativo na unidade (`app/services/billing/staff.rb:9-14`). A decisão 13 exige ponto "com CNES". **Proposta:** a unidade recusa o responsável sem vínculo (`ostomy_responsible_not_linked`) e o ponto sem CNES (`ostomy_service_requires_cnes`).

D13. **[Resolvida pela Q12.]** **Modelos de aparelho sem rota.** O §5 cria `device_models`, mas só dá rotas a `device_loans`. **Proposta:** `GET|POST /pharmacy/admin/device_models` e `PATCH …/:id` pelo `municipal_admin` com step-up, e `GET /pharmacy/device_models` para quem entrega.

D14. **[Resolvida pela Q12.] A aplicação é a única que cria paciente; não há desfazer.** **CSV de importação sem cabeçalho e criação antecipada de paciente.** O §3 não fixa as colunas. O `Patients::LookupByIdentifier` cria o paciente já na busca (`app/commands/patients/lookup_by_identifier.rb:38-41`), o que, na conferência, criaria pacientes antes da aplicação. **Proposta:** o cabeçalho da Task 16 e `dry_run:` no lookup durante a conferência.

D15. **[Resolvida pela Q12.]** **A planilha de saldo inicial do 27b não tem insumo.** O `tipo_produto` aceita só `medicamento | vacina | vetorial` (`2026-11-24-module-27b-bnafar-contracts.md:193`). **Proposta:** acrescentar `insumo`, com `codigo_produto` = CATMAT do material.

D16. **[Resolvida pela Q13.]** **"jsonb cifrado".** O molde de JSON cifrado na cidade é uma coluna `text` com `serialize coder: JSON` e `encrypts` (`app/models/tfd_case.rb:8-9`; `db/city_schema.rb:4694`). Um jsonb cifrado não seria consultável e quebraria o padrão. **Proposta:** `profile` e `external_report` em `text` cifrado; `quotas` em jsonb sem cifra.

D17. **[Resolvida pela Q13.] O pré-requisito do `supply_programs` usa `clinical_record_disabled`.** **Nomes de auditoria da plataforma.** O §9 lista só eventos de domínio. A rake não tem mantenedor, então o serviço audita, como o `medication_catalog.imported` (`catalog_import.rb:73`). **Proposta:** `supply_catalog.imported` e `maintenance.supply_catalog.imported` no R18; o segundo também em `MaintenanceAudit::NAMES`.

D18. **[Resolvida pela Q7.]** **Visões fora do profissional e lista de CIDs "da cidade".** A linha BPA da bolsa entraria na visão do cidadão pela `ProcedureSource` (`app/services/timeline/sources/procedure_source.rb:26`), e a página do PEC externo herda `Timeline::KINDS` (`app/controllers/pec_timelines_controller.rb:16`). A spec §3.2 fala em "lista da cidade" de CIDs, sem tabela no contrato. **Proposta:** o cidadão não vê a linha `supply`; o PEC externo não vê `supply_enrollment`; a lista de CIDs fica num YAML fixo da plataforma (E10–E14 e O24; estomia sem restrição).
