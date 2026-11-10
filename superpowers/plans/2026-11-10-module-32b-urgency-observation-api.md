# Módulo 32b — Observação, prescrição interna, checagem e estoque (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 32a em `origin/main` do api** (conferido ao escrever este plano em `69d283c4`): última migração de cidade `20261105900006_add_urgency_billing_source.rb` (`db/city_schema.rb` em `define(version: 2026_11_05_900006)`, `enable_extension "btree_gist"` na linha 15); nenhuma migração de plataforma nova no 32b; `db/city_triggers.sql` com 3490 linhas, terminando no bloco do 32a (`rota_urgency_encounter_guard()` em ~3398–3450: encontro fechado nunca muda, estado só para frente; o `DO $do$` dos triggers da urgência e da sala de espera em ~3452–3490), reexecutado no fim do `up` de cada migração; `rota_no_delete()`, `rota_append_only()` e `rota_reencryption_only(cols…)` (~1347) disponíveis; padrão CAPS: `db/city_migrate/20261104900002_add_psychosocial_care.rb` (`psychosocial_beds` com `idx_psychosocial_beds_label` e `updated_by_user_id`; `psychosocial_night_stays` com `excl_psychosocial_night_stays_bed` = `psychosocial_bed_id WITH =, tsrange(started_at, COALESCE(ended_at, 'infinity'::timestamp without time zone)) WITH &&` e `idx_psychosocial_night_stays_open_patient`), `rota_psychosocial_night_stay_guard()` (~3291: re-cifra primeiro, "a closed stay never changes", "an open stay only receives the closing"), `app/commands/psychosocial/start_night_stay.rb` (conferência em Ruby e `rescue ActiveRecord::StatementInvalid` pelo nome da exclusion → `bed_taken`), `app/commands/psychosocial/save_bed.rb` (rótulo repetido → **409** `bed_label_taken`), `HealthUnitsController#caps_change_conflict` (`caps_has_beds`, **409**) e `#deactivation_conflict` (409); `Urgency::Access` (`reception?`, `classifier_link`, `physician_link`, `clinical_link` = 2235\*/225\*, `refusal`, `link` privado), `Urgency::Catalog` (`CLASSIFIER_CBO_PREFIXES = %w[2235]`, `PHYSICIAN_CBO_PREFIX = "225"`, `NON_CARE_OUTCOMES`), `Urgency::Config::KEYS = %w[version review source classification_code care_code]`, `config/urgency/sigtap.yml`, `Urgency::Trail::RESOURCES = %w[person_lookup encounter patient_duplicates provisional_encounters]`, `Urgency::Json.encounter(encounter, clinical:)`, `Urgency::Queue.groups` (`in_care_item`), `Urgency::Dashboard.call(units:, month:)` (bloco `stay` em minutos, `Admin::SmallCount.small?`), `Urgency::RecordOutcome` (`attendance.lock!` → `encounter.lock!`, `encounter.update!(state: "closed")`, `Attendances::Close(..., urgency: true)`), `Urgency::BaseController` (`FAILURE_STATUS`, `render_failure`, `render_refusal`, `query_only!`, `body`, `render_encounter` com trilha), rotas em `scope "/urgency"` (`config/routes.rb:744`); `Billing::FromUrgency` (`lines` com `care_code`, idempotente por `idx_performed_procedures_urgency (urgency_encounter_id, sigtap_code)`); `lib/sigtap_sample.rb` com `0301060029` (instrumento `02`, CBOs `225125 225124 225142`, e **CIDs restritos a I10, E119, J069** na linha 145) e `0301100012`; `Timeline::Entry` (`Data.define`, 11 membros, 17 chamadas `Entry.new`), `Timeline::Json.professional` (chaves fixas; `spec/requests/timeline_spec.rb:25` fixa as chaves da 1ª entrada), `Timeline::Sources::UrgencySource`, `PecExternal::Page.items` (a página do PEC é HTML, lê `entry.summary`); `Pharmacy::Ledger` (`DOCUMENTS` com 6 modelos, `lock_lots!` → `lock_location!` → documento, `record!` em savepoint, `Insufficient`, `Inactive`), `Pharmacy::Catalog` (`MOVEMENT_KINDS` 13, `POSITIVE_KINDS`, `NEGATIVE_KINDS`, `QUANTITY`, `ROLES`), `Pharmacy::Quantity.parse` (só String), `Pharmacy::DispensingUnit.for`, `Pharmacy::Statement#document` (6 tipos), `Pharmacy::Alerts` (5 chaves), `Pharmacy::Access` (`allowed?`, `central_keeper?`), `LocationReading#required_location` (parâmetro **`location_id`**), `Pharmacy::Json.catalog_item`/`catalog_for`, `StockLotBlock.open_blocks`, `stock_lots.dispensing_unit`, `rota_stock_balance_guard()` (AFTER INSERT); o precedente do 29: `20261102900001_add_vaccination_room.rb#change_stock_movements` (~355–375, `replace_check` de `_kind`, `_sign`, `_document`) e `Vaccination::RecordImmunization` (`Ledger.record!(kind: "immunization", document: dose)`); `MedicationCatalogItem` (plataforma; `prescribable`, `controlled_list` string(3), `catmat_code` integer) lido FORA da transação da cidade; `PrescriptionItem::ROUTES`; `Platform::Features::CATALOG` terminando em `urgency_care`, `waiting_room_display`; `config/initializers/domain_events.rb` (urgência ~420–446); `CityEncryption::CITY_KEYED_TARGETS` (urgência ~197–199); `spec/support/{urgency_helpers,pharmacy_helpers}.rb` (`urgency_city!`, `upa_nurse!`, `upa_doctor!`, `called!`, `urgency_consultation!`, `finalized_urgency!`, `urgency_admin!`, `pharmacy_city!`, `unit_location!`, `stocked_lot!`, `pharmacy_staff!`, `losartana`, `diazepam`, `balance_of`); `lib/urgency_crew.rb` (Curitiba, "UPA 24h Centro") rodando depois do `PharmacyCrew` em `db/seeds.rb:444`. Specs que fixam listas e mudam neste plano: `spec/services/platform/features_spec.rb:121` (e o exemplo do 32a com `keys.last(2)`), `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30`, `spec/config/urgency_sigtap_spec.rb:9`, `spec/models/pharmacy_schema_spec.rb:236` (as listas de `AddVaccinationRoom` = `Pharmacy::Catalog`), `spec/requests/pharmacy/positions_spec.rb:515,517,534,557` (chaves de `/pharmacy/alerts`), `spec/initializers/domain_events_bindings_spec.rb:582` (os oito eventos do 32a), `spec/security/urgency_secrecy_spec.rb:211`. Guardas que **não** mudam: `app/services/urgency/merge.rb` e `spec/architecture/urgency_merge_tables_spec.rb` (nenhuma tabela nova tem `patient_id`).

**Goal:** O lado api do módulo 32b (F-32.7 e F-32.8, ADR 0050): leitos de observação da UPA com ocupante único (exclusion), entrada, troca e saída da observação com alertas de 20 h e 24 h calculados na leitura, prescrição interna em versões (só itens do catálogo, controlados aceitos pelo P1), aprazamento calculado, checagem da enfermagem imutável, baixa de estoque opcional (`urgency_stock_deduction`) por FEFO no `stock_location` da UPA com faltas (R7) nunca negativas, a fila "Faltas da UPA" da farmácia, o fechamento pelo desfecho, a observação no BPA, na linha do tempo e no painel do gestor, os três blocos do detalhe do encontro, a semente de dev e a suíte completa.

**Architecture:** Três migrações de cidade, uma por fronteira: a observação (`urgency_beds`, `urgency_bed_stays`, `900001`), a prescrição e a checagem (`urgency_prescriptions`, `urgency_prescription_items`, `urgency_administrations`, `900002`) e o estoque do 27a (`stock_movements` e `urgency_stock_shortfalls`, `900003`, a única transversal). Nenhuma tabela nova tem `patient_id`: o paciente é sempre o do `attendance` do encontro, então a fusão do 32a não muda. Quem está em observação é quem tem uma estadia aberta; o encontro segue `in_care` (nenhum estado novo, a guarda do encontro não muda). Horários, alertas e contagens são calculados na leitura, sem tabela, job ou evento. A administração grava o movimento `administration` pelo `Pharmacy::Ledger`, num savepoint junto com a própria administração: se o saldo não cobre, o savepoint volta e a administração é gravada de novo sem lote, com a falta. A ordem de trava de toda escrita clínica é **atendimento → encontro → estadia → prescrição → lotes → local → documento** (a do 32a seguida da do Ledger).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade, `btree_gist` já ligado), Solid Queue, RSpec via `bin/test`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (decisões 2, 3, 20–23, R7 e o §3 "Cidade (32b)") e `docs/adr/0050.md`. Contrato — **fonte única**: `docs/superpowers/plans/2026-11-10-module-32b-urgency-observation-contracts.md` (§1–§12, com P1 e P2); o do 32a (`2026-11-05-module-32a-urgency-contracts.md`, §1–Q31) continua valendo. Toda diferença está em "Divergências propostas ao contrato", no fim.

## Desvios e precisões

1. **Observação sem estado novo.** "Em observação" = existe `urgency_bed_stays` com `ended_at IS NULL` do encontro. Entrar exige o encontro `in_care` (trigger e Ruby). A troca de leito fecha a estadia atual com `bed_change` e abre a nova no **mesmo instante** (`now`): o `tsrange` meio-aberto `[started_at, ended_at)` não sobrepõe.
2. **Horas.** `hours = floor((now − primeiro started_at do encontro) / 1 h)` enquanto houver estadia aberta; nas leituras históricas (linha do tempo, painel), até o último `ended_at`. A troca de leito e a saída seguida de nova entrada não zeram a conta (contrato §4, literal; Divergência A25). `alert`: `null` abaixo de 20 h, `"near_limit"` a partir de 20 h, `"over_limit"` a partir de 24 h.
3. **Prescrição em versões.** Uma `active` por encontro (índice único parcial). Emitir com uma `active` = na mesma transação, a anterior vira `superseded` e a nova nasce `version + 1` com `supersedes_id`. Itens e administrações de versões antigas ficam como estão. O desfecho leva a `active` a `ended`. Suspender leva a `suspended`, com motivo cifrado. Transição só para a frente (trigger).
4. **Catálogo lido fora da transação.** `MedicationCatalogItem` é da plataforma: o controlador lê `MedicationCatalogItem.prescribable.where(id:)` antes de chamar o comando (o padrão do `Pharmacy::RecordEntry` e do `ClinicalDocuments::Issue#prescription_platform`). O item copia `catmat_code`, `controlled_list` (P1) e `stock_unit` = `Pharmacy::DispensingUnit.for(item)` (Divergência A7). O nome do medicamento não é gravado: sai do catálogo na leitura (`Pharmacy::Json.catalog_item`).
5. **Aprazamento.** `once` = um horário, `first_dose_at`. `interval` = `first_dose_at + k × interval_hours`, k ≥ 0, até `now + 24 h`. `if_needed` = sem horários; `remaining = max_per_day − administradas (status administered) nas últimas 24 h`. Horários com segundos zerados (`first_dose_at` é truncado ao minuto na gravação), comparados por igualdade exata.
6. **Checagem.** Imutável (trigger `rota_reencryption_only('reason')`). `administered_at` só em `administered` (padrão `now`; no máximo `now + 5 min` e nunca antes de `prescribed_at` da versão). `reason` (10..500) só e sempre fora de `administered`. Ordem das recusas: encontro fechado (409) → acesso (403) → item de versão não `active` (422) → forma → horário (`slot_invalid` 422, `slot_taken` 409, `max_per_day_reached` 422) → estoque.
7. **Estoque.** Só com `urgency_stock_deduction` utilizável (lido no controlador, fora da transação) **e** `administered`. Local = `StockLocation` `unit` **ativo** da UPA (inativo ou ausente, ou `Ledger::Inactive` → falta `no_location`, `stock_location_id` nulo). Lote: o do corpo (vencido, de outro item, de outra unidade de dispensação, bloqueado ou com saldo 0 → 422 `lot_invalid`, nada gravado; saldo > 0 mas < quantidade → falta `lot_unavailable`) ou o FEFO: os lotes do item, da mesma `dispensing_unit` do item, não vencidos, sem bloqueio aberto, com saldo > 0 no local, por `expires_on, lot_number, id`; **todos** travados por `Ledger.lock_lots!` (ordem de id), então `lock_location!`, então o saldo relido sob a trava; vale o primeiro com saldo ≥ quantidade; nenhum → falta `insufficient_balance`. Um movimento só, de um lote só (Divergência A10). Gravação: savepoint `{ administração com lote e quantidade; Ledger.record!(kind: "administration", document: administração) }`; `Ledger::Insufficient` (rede do trigger) → o savepoint volta e a administração é gravada sem lote, com falta `lot_unavailable`. A checagem nunca é recusada por falta de estoque.
8. **Desligado.** `stock_lot_id` ou `stock_quantity` no corpo com o interruptor desligado → 422 `stock_deduction_disabled` (path do campo). Fora de `administered`, os dois campos → 422 `invalid_request`. Sem movimento e sem falta.
9. **Quantidades como no papel.** `dose`, `stock_quantity_per_dose` e `stock_quantity` são String decimal com ponto (número JSON recusado), como no 27a (Divergência A6): `dose` em `DOSE` (até 3 casas), as do estoque em `Pharmacy::Catalog::QUANTITY` (`Pharmacy::Quantity.parse`). Saída: `dose` como gravada, estoque com duas casas (`Pharmacy::Quantity.dump`).
10. **Acesso** (`Urgency::Access`, novos): `nursing_or_physician_link` (2235\*, 3222\*, 225\*) para checar e para ler o aprazamento; `clinical_link` (2235\*, 225\*, o do 32a) para entrar, trocar e sair da observação; `physician_link` para prescrever e suspender; mapa = `reception?` **ou** `nursing_or_physician_link`. O detalhe do encontro continua no Q19 do 32a (2235\*/225\* completo; recepção sem o clínico; o técnico 3222\* não lê o detalhe — Divergência A17). Todo vínculo é o ativo na UPA do atendimento; o gravado em `professional_link_id`/`cbo_code` é o devolvido.
11. **Trilha.** O detalhe com os blocos novos grava a trilha que já grava (`resource: "encounter"`). O aprazamento grava `Urgency::Trail.viewed!(resource: "medication_schedule")` (recurso novo). Mapa, fila e faltas não gravam (polling ou sem dado clínico).
12. **Faltas na farmácia.** Rotas sob `/pharmacy` (o `Pharmacy::BaseController` exige `pharmacy` utilizável, não `urgency_stock_deduction`). Leitura e conciliação: farmacêutico ou almoxarife do local (`Pharmacy::Access.allowed?(roles: %w[pharmacist stock_keeper])`) ou o almoxarife do central (`central_keeper?`), que também lê as faltas `no_location` (Divergência A5).

## Valores fixados por este plano

1. **Interruptor** (fim do `CATALOG`, depois de `waiting_room_display`): `Entry.new(key: "urgency_stock_deduction", description: "Atendimentos da UPA com prescrição de medicação atualizam o estoque", requires: %w[feature:urgency_care feature:pharmacy])`, com o comentário do padrão (sem cascata; desligar a farmácia ou a urgência deixa o interruptor ligado faltando pré-requisito: `urgency_care_disabled`, `pharmacy_disabled`). `Urgency::StockGate::KEY = "urgency_stock_deduction"`, `.usable?(city)` no padrão do `Urgency::Gate`.
2. **`Urgency::Catalog` (acréscimos):** `NURSING_TECH_CBO_PREFIX = "3222"`; `CHECKER_CBO_PREFIXES = %w[2235 3222 225]`; `BED_LABEL = (1..20)`; `STAY_END_KINDS = %w[bed_change observation_ended outcome]`; `OBSERVATION_NEAR_LIMIT_HOURS = 20`; `OBSERVATION_LIMIT_HOURS = 24`; `PRESCRIPTION_STATUSES = %w[active superseded suspended ended]`; `PRESCRIPTION_ITEMS = (1..20)`; `FREQUENCY_KINDS = %w[once interval if_needed]`; `INTERVAL_HOURS = (1..24)`; `MAX_PER_DAY = (1..24)`; `DOSE = /\A\d{1,5}(\.\d{1,3})?\z/`; `DOSE_UNITS = %w[mg mcg g mL UI mEq gota comprimido capsula ampola frasco jato aplicacao sache]` (Divergência A32); `INSTRUCTIONS_MAX = 500`; `SUSPENSION_REASON = (10..500)`; `FIRST_DOSE_WINDOW = (-1.hour..24.hours)` (relativo a `now`); `SCHEDULE_AHEAD = 24.hours`; `IF_NEEDED_WINDOW = 24.hours`; `ADMINISTRATION_STATUSES = %w[administered not_administered refused]`; `ADMINISTRATION_REASON = (10..500)`; `ADMINISTERED_AT_TOLERANCE = 5.minutes`; `DETAIL_ADMINISTRATIONS_WINDOW = 48.hours`; `SHORTFALL_CAUSES = %w[no_location insufficient_balance lot_unavailable]`; `RECONCILIATION_NOTE = (10..500)`.
3. **`config/urgency/sigtap.yml`** ganha `observation_code: "0301060029"` (o arquivo segue `review: true`); `Urgency::Config::KEYS = %w[version review source classification_code care_code observation_code]`.
4. **Pharmacy:** `Pharmacy::Catalog::MOVEMENT_KINDS` e `NEGATIVE_KINDS` terminam em `administration`; `Pharmacy::Ledger::DOCUMENTS` ganha `UrgencyAdministration => :urgency_administration`; `Pharmacy::Statement#document` ganha `{ type: "urgency_administration", id:, number: nil }`; `Pharmacy::Alerts` ganha `urgency_shortfalls` (6ª chave, as faltas abertas do local).
5. **Eventos** (contrato §10, todos `to: []`): `urgency.observation_started {encounter_id, stay_id}`, `urgency.observation_ended {encounter_id, stay_id}`, `urgency.prescription_issued {encounter_id, prescription_id}`, `urgency.prescription_suspended {encounter_id, prescription_id}`, `urgency.medication_checked {encounter_id, administration_id}`, `urgency.stock_shortfall_recorded {shortfall_id}`, `pharmacy.urgency_shortfall_reconciled {shortfall_id}`.
6. **Filtro de log** (acréscimo): `/\A(instructions|dose|dose_unit|frequency|first_dose_at|stock_lot_id|stock_quantity|stock_quantity_per_dose|bed_id)\z/` (`reason`, `note`, `items`, `route`, `catalog_item_id` já são filtrados).
7. **Dev:** api do worktree na porta **3055**; worktree `apps/api/.claude/mod32b` (com `config/master.key`); branch `feat/mod-32b-urgency-observation`; migrações `db/city_migrate/{20261110900001_add_urgency_observation,20261110900002_add_urgency_prescriptions,20261110900003_add_urgency_stock_deduction}.rb`; `db/city_schema.rb` termina em `define(version: 2026_11_10_900003)`. Semente (Task 18): Curitiba, "UPA 24h Centro".

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (os arquivos da task, por nome); commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge e push só com autorização do usuário.
- **Primeira ação da execução:** mover F-32.7 e F-32.8 do board #1 para In Progress (um comando `gh project item-edit …` por chamada, começando com `gh`). F-32.7 vai para Done quando o dashboard terminar; F-32.8 quando o maintenance terminar; Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod32b <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. Mudança transversal (`stock_movements`, `Pharmacy::Ledger`, fechamento do encontro, BPA, lista de interruptores, `spec/support`) → **suíte completa obrigatória antes do merge**, pedida à sessão "Testes" (Task 19).
- Migrações: dump à mão em `db/city_schema.rb`; triggers em `db/city_triggers.sql`, bloco novo "Módulo 32b" depois do bloco do 32a, executado no fim do `up` de cada migração; todas irreversíveis. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error", "path"? }` + `problems: [ { path, error } ]` nas recusas de campo; 409 de conflito; interruptor 403 `{ "error": "feature_disabled", "feature": "urgency_care" }` (o `urgency_stock_deduction` nunca dá 403: desligado, a checagem segue sem estoque); step-up 401 `mfa_required`; escritas que recusam chave desconhecida (Q23); listas com envelope `{ items }`.
- **Nome, CPF, CNS, CID, queixa, medicamento (nome, código, dose, via) e motivo nunca em log, evento, erro ou URL.** Mapa, aprazamento e faltas só com ids na query (`query_only!`). Modelos novos sobrescrevem `inspect`/`to_s`/`pretty_print`/`as_json` (só `id`), como `UrgencyEncounter`.
- Nunca `Current.city =` solto; interruptores e catálogo da plataforma lidos **fora** da transação da cidade.
- Cifra por cidade: `instructions`, `reason`, `suspension_reason`, `reconciliation_note` em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); toda guarda que deixa uma coluna cifrada mudar aceita a re-cifra (`rota.reencrypting`) antes das outras regras, no padrão do `rota_psychosocial_night_stay_guard`.
- `DomainEvents.publish(` é multilinha — confira cada chamada inteira. Nenhum `Platform.audit` novo (R18 intacto).
- Confira nomes reais no código antes de usá-los (o PRÉ-REQUISITO lista os conferidos).
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`, ao encerrar a task.

## Review Focus

1. **Dois profissionais põem pacientes diferentes no mesmo leito ao mesmo tempo:** um vence; o outro recebe 409 `bed_taken` (conferência em Ruby e, na corrida, o `excl_urgency_bed_stays_bed`); nenhuma estadia fica aberta sem leito. Teste na Task 6.
2. **Último comprimido, duas checagens simultâneas do mesmo lote (FEFO):** as duas administrações são gravadas; uma leva o movimento, a outra nasce sem lote com a falta `insufficient_balance`; o saldo nunca fica negativo e nenhuma das duas recebe erro. Teste na Task 10.
3. **Desfecho no meio de uma checagem:** a checagem que chegou depois do desfecho recebe 409 `encounter_closed`; a que chegou antes fica gravada e o desfecho fecha a estadia e leva a prescrição a `ended` na mesma transação; nenhum deadlock entre a trava do encontro e a do lote. Teste na Task 12.
4. **Troca de leito às 23 h de observação:** o mapa segue mostrando as horas desde a primeira entrada (23 → `near_limit`; 24 → `over_limit`), o leito antigo fica livre no mesmo instante, e o BPA sai com `0301060029` no lugar do `0301060096`. Testes nas Tasks 6 e 13.
5. **Interruptor desligado depois de faltas registradas:** a checagem segue sem estoque, `stock_lot_id` no corpo dá 422 `stock_deduction_disabled`, e a farmácia ainda lê e concilia as faltas antigas (só `pharmacy` exigido). Teste na Task 11.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod32b/`)
- Migrações: as três do Valor fixado 7.
- Modelos: `app/models/{urgency_bed,urgency_bed_stay,urgency_prescription,urgency_prescription_item,urgency_administration,urgency_stock_shortfall}.rb`.
- `app/services/urgency/{stock_gate,observation,schedule,input,prescription_json}.rb`.
- `app/commands/urgency/{save_bed,start_observation,end_observation,issue_prescription,suspend_prescription,record_administration,deduct_stock}.rb`; `app/commands/pharmacy/reconcile_urgency_shortfall.rb`.
- `app/controllers/urgency/{beds,observations,prescriptions,medication_schedules,administrations}_controller.rb`; `app/controllers/pharmacy/urgency_shortfalls_controller.rb`.
- Specs listados em cada task; `spec/support/urgency_observation_helpers.rb`.

**Modificar**
- Urgência: `app/services/urgency/{access,catalog,config,trail,json,queue,dashboard}.rb`, `app/commands/urgency/record_outcome.rb`, `app/controllers/urgency/{base,encounters}_controller.rb`, `app/models/urgency_encounter.rb`, `config/urgency/sigtap.yml`.
- 13: `app/commands/attendances/close.rb` (só se a Divergência A21 for aceita), `app/controllers/health_units_controller.rb`.
- 22: `app/commands/billing/from_urgency.rb`.
- 27a: `app/services/pharmacy/{catalog,ledger,statement,alerts}.rb`, `app/models/stock_movement.rb`.
- Linha do tempo: `app/services/timeline/{entry,json}.rb`, `app/services/timeline/sources/urgency_source.rb`.
- Plataforma e infra: `app/services/platform/features.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `lib/urgency_crew.rb`.
- Specs existentes: os listados no PRÉ-REQUISITO.

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-32.7 e F-32.8 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod32b -b feat/mod-32b-urgency-observation origin/main && cp config/master.key .claude/mod32b/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO com `grep` no worktree: `define(version: 2026_11_05_900006)`, `ls db/city_migrate | tail -1`, `excl_psychosocial_night_stays_bed`, `rota_psychosocial_night_stay_guard`, `DOCUMENTS = ` no Ledger, `MOVEMENT_KINDS` do `Pharmacy::Catalog`, `KEYS = ` do `Urgency::Config`, `RESOURCES = ` do `Urgency::Trail`, `CHECKER`/`CLASSIFIER_CBO_PREFIXES`, `"waiting_room_display"` como última `Entry`, `"0301060029"` no `lib/sigtap_sample.rb`, `positions_spec.rb:517`, `pharmacy_schema_spec.rb:236`, e se o técnico 322205 do `ProfessionalCrew` tem login de dev (a Task 18 depende disso). Diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod32b spec/services/platform/features_spec.rb spec/config/urgency_sigtap_spec.rb spec/models/pharmacy_schema_spec.rb spec/architecture/urgency_merge_tables_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS (linha de base).

**Relato ao coordenador:** "api 32b · Task 0 · worktree mod32b em origin/main <hash>, pré-requisito conferido (diferenças: …), linha de base verde."

---

### Task 1: Interruptor, catálogo do 32b, `observation_code` e filtro de log (transversal: lista de interruptores)

**Files:**
- Create: `app/services/urgency/stock_gate.rb`
- Modify: `app/services/platform/features.rb`, `app/services/urgency/{catalog,config,access}.rb`, `config/urgency/sigtap.yml`, `config/initializers/filter_parameter_logging.rb`, `spec/services/platform/features_spec.rb`, `spec/models/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/config/urgency_sigtap_spec.rb`, `spec/services/urgency/access_spec.rb` (criar se não existir)

**Interfaces:**
- Produces: Valor fixado 1 (`Urgency::StockGate.usable?(city)`); Valor fixado 2; `Urgency::Config.sigtap["observation_code"]`; `Urgency::Access.nursing_or_physician_link(user, health_unit_id) -> ProfessionalLink | nil` (prefixos `CHECKER_CBO_PREFIXES`) e `Urgency::Access.observation_reader?(user, health_unit_id) -> Boolean` (`reception?` ou o link acima).

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: o exemplo do 32a troca `keys.last(2)` por `keys.last(3).first(2)`; exemplo novo:

```ruby
it "módulo 32b: urgency_stock_deduction é a última chave e exige a urgência e a farmácia" do
  entry = Platform::Features::CATALOG.last
  expect(entry.key).to eq("urgency_stock_deduction")
  expect(entry.requires).to eq(%w[feature:urgency_care feature:pharmacy])
  expect(entry.description).to eq("Atendimentos da UPA com prescrição de medicação atualizam o estoque")
  expect(Platform::Features::CITIZEN_HIDDEN).not_to include("urgency_stock_deduction")
end

it "faltando a farmácia, o interruptor ligado fica inutilizável com pharmacy_disabled" do
  city = urgency_city!
  Platform::Features.set!(city: city, key: "urgency_stock_deduction", enabled: true, maintainer: ledi_maintainer!)
  expect(Platform::Features.missing(city, "urgency_stock_deduction")).to eq(%w[pharmacy_disabled])
  expect(Urgency::StockGate.usable?(city)).to be(false)
end
```

As três listas exatas (`city_record_settings_spec.rb:46`, `city_features_spec.rb:103`, `operators/city_record_settings_spec.rb:30`) ganham `urgency_stock_deduction` no fim. `urgency_sigtap_spec.rb`: chaves `%w[version review source classification_code care_code observation_code]`, `config["observation_code"] == "0301060029"`, `Urgency::Catalog::CHECKER_CBO_PREFIXES == %w[2235 3222 225]`, o filtro de log de `instructions`, `dose`, `stock_lot_id`, `bed_id`. `access_spec.rb`: o técnico 322205 vinculado à UPA tem `nursing_or_physician_link` e não tem `clinical_link`; a recepção é `observation_reader?` sem vínculo; um enfermeiro de outra unidade não.

- [ ] **Step 2:** `bin/test api -w mod32b spec/services/platform/features_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/config/urgency_sigtap_spec.rb spec/services/urgency/access_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a entrada, o portão, as constantes, a chave do YAML (com o comentário "0301060029: atendimento de urgência com observação até 24 h — conferência no gate SIGTAP, rotasaude/api#136"), os dois métodos do `Access` (o `link` privado já aceita a lista de prefixos) e o filtro.
- [ ] **Step 4:** os mesmos specs + `spec/requests/maintenance/city_features_spec.rb` inteiro → PASS.
- [ ] **Step 5: Commit** `feat: add the urgency stock deduction switch and observation billing code`.

**Relato ao coordenador:** "api 32b · Task 1 · `urgency_stock_deduction` no fim do CATALOG (requires urgency_care + pharmacy; missing `urgency_care_disabled`/`pharmacy_disabled` — útil ao maintenance), observation_code no sigtap.yml."

---

### Task 2: Esquema da observação — leitos e estadias (migração 900001)

**Files:**
- Create: `db/city_migrate/20261110900001_add_urgency_observation.rb`, `app/models/{urgency_bed,urgency_bed_stay}.rb`, `spec/models/urgency_observation_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (bloco "Módulo 32b — observação"), `app/models/urgency_encounter.rb`

**Interfaces:**
- Produces (banco):
  - `urgency_beds`: `id` uuid, `health_unit_id` (FK, NOT NULL), `label` string(20) NOT NULL, `active` bool NOT NULL DEFAULT true, `updated_by_user_id` (FK users, NOT NULL), timestamps; `idx_urgency_beds_label (health_unit_id, label)` único; CHECK `ck_urgency_beds_label` = `char_length(btrim(label)) BETWEEN 1 AND 20`.
  - `urgency_bed_stays`: `id`, `urgency_encounter_id` (FK), `urgency_bed_id` (FK), `health_unit_id` (FK), `started_at`, `ended_at`, `end_kind`, `started_by_user_id` (FK), `ended_by_user_id` (FK), timestamps; CHECKs `ck_urgency_bed_stays_closing` = `(ended_at IS NULL) = (end_kind IS NULL) AND (ended_at IS NULL) = (ended_by_user_id IS NULL)`, `ck_urgency_bed_stays_times` = `ended_at IS NULL OR ended_at >= started_at`, `ck_urgency_bed_stays_end_kind` (lista de `STAY_END_KINDS`); `excl_urgency_bed_stays_bed` = `urgency_bed_id WITH =, tsrange(started_at, COALESCE(ended_at, 'infinity'::timestamp without time zone)) WITH &&` (gist); `idx_urgency_bed_stays_open_encounter (urgency_encounter_id) WHERE ended_at IS NULL` único; índices `(health_unit_id, started_at)` e `(urgency_encounter_id, started_at)`.
  - Triggers: `rota_urgency_bed_guard()` — INSERT só em unidade `kind = 'upa'` (`'urgency_beds: only at an upa'`), UPDATE nunca muda `health_unit_id` nem `created_at`; `urgency_beds_no_delete` (`rota_no_delete`). `rota_urgency_bed_stay_guard()` — INSERT: o encontro existe, está `in_care` (`'urgency_bed_stays: the encounter is not in care'`), o leito está ativo e é da mesma unidade da estadia e do atendimento (`'urgency_bed_stays: bed of another unit or inactive'`), nasce aberta; UPDATE: fechada nunca muda (`'urgency_bed_stays: a closed stay never changes'`), aberta só recebe `ended_at`, `end_kind`, `ended_by_user_id`, `updated_at` (`'urgency_bed_stays: an open stay only receives the closing'`); DELETE recusado. TRUNCATE recusado nas duas (`rota_append_only`).
- Produces (Ruby): `UrgencyBed` (`belongs_to :health_unit`, `:updated_by_user`; `scope :active_beds`); `UrgencyBedStay` (`belongs_to :urgency_encounter, :urgency_bed, :health_unit, :started_by_user, :ended_by_user (optional)`; `scope :open_stays, -> { where(ended_at: nil) }`; `#open?`); `UrgencyEncounter has_many :bed_stays, -> { order(:started_at, :id) }` e `#open_stay`. `inspect`/`to_s`/`pretty_print`/`as_json` só com o id nos dois.

- [ ] **Step 1: Write the failing test** `spec/models/urgency_observation_schema_spec.rb`:

```ruby
RSpec.describe "Esquema da observação da UPA", type: :model do
  let!(:city) { urgency_city! }
  let(:upa) { create_unit("UPA Obs", kind: "upa") }
  let(:doctor) { upa_doctor!(upa) }
  let(:bed) { UrgencyBed.create!(health_unit: upa, label: "OBS 01", updated_by_user: urgency_admin!) }

  def in_care!
    e = arrived!(upa)
    classified!(e, color: "yellow", by: upa_nurse!(upa))
    called!(e, by: doctor)
    e.reload
  end

  def stay!(encounter, bed, at: Time.current)
    UrgencyBedStay.create!(urgency_encounter: encounter, urgency_bed: bed, health_unit: upa, started_at: at, started_by_user: doctor)
  end

  it "leito só na UPA, rótulo único por unidade e nunca apagado" do
    expect { UrgencyBed.create!(health_unit: create_unit("UBS"), label: "L1", updated_by_user: urgency_admin!) }
      .to raise_error(ActiveRecord::StatementInvalid, /only at an upa/)
    bed
    expect { UrgencyBed.create!(health_unit: upa, label: "OBS 01", updated_by_user: urgency_admin!) }.to raise_error(ActiveRecord::RecordNotUnique)
    expect { bed.delete }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "um ocupante por leito (exclusion); a troca no mesmo instante não sobrepõe" do
    a, b = in_care!, in_care!
    first = stay!(a, bed)
    expect { stay!(b, bed) }.to raise_error(ActiveRecord::StatementInvalid, /excl_urgency_bed_stays_bed/)
    now = Time.current
    first.update!(ended_at: now, end_kind: "bed_change", ended_by_user: doctor)
    expect { stay!(b, bed, at: now) }.not_to raise_error
  end

  it "nasce só com o encontro em atendimento; fechada nunca muda; uma aberta por encontro" do
    waiting = arrived!(upa)
    expect { stay!(waiting, bed) }.to raise_error(ActiveRecord::StatementInvalid, /not in care/)
    e = in_care!
    s = stay!(e, bed)
    expect { stay!(e, UrgencyBed.create!(health_unit: upa, label: "OBS 02", updated_by_user: urgency_admin!)) }
      .to raise_error(ActiveRecord::RecordNotUnique, /idx_urgency_bed_stays_open_encounter/)
    s.update!(ended_at: Time.current, end_kind: "observation_ended", ended_by_user: doctor)
    expect { s.update_column(:end_kind, "outcome") }.to raise_error(ActiveRecord::StatementInvalid, /closed stay never changes/)
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod32b spec/models/urgency_observation_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (o `create_table`/`add_exclusion_constraint` no padrão de `20261104900002`; `execute File.read(Rails.root.join("db/city_triggers.sql"))` no fim do `up`; `down` irreversível), o dump à mão e o bloco de triggers (com o comentário do padrão: ADR 0050, contrato §1 e §11, plano Desvio 1; `IF to_regclass('public.urgency_beds') IS NOT NULL`).
- [ ] **Step 4:** o spec + `spec/architecture/urgency_merge_tables_spec.rb spec/models/urgency_schema_spec.rb` (se existir; o guard das tabelas do 32a) → PASS. A guarda da fusão continua verde sem mudança (nenhum `patient_id`).
- [ ] **Step 5: Commit** `feat: add urgency observation beds and stays with one occupant per bed`.

**Relato ao coordenador:** "api 32b · Task 2 · migração 20261110900001 (urgency_beds, urgency_bed_stays com exclusion e uma aberta por encontro); merge_tables_spec intacto."

---

### Task 3: Esquema da prescrição e da checagem (migração 900002)

**Files:**
- Create: `db/city_migrate/20261110900002_add_urgency_prescriptions.rb`, `app/models/{urgency_prescription,urgency_prescription_item,urgency_administration}.rb`, `spec/models/urgency_prescription_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (bloco "Módulo 32b — prescrição"), `app/services/city_encryption.rb`, `app/models/urgency_encounter.rb`

**Interfaces:**
- Produces (banco):
  - `urgency_prescriptions`: `urgency_encounter_id` (FK), `version` integer NOT NULL, `status` NOT NULL DEFAULT 'active', `supersedes_id` (FK self), `prescribed_by_user_id` (FK), `professional_link_id` (FK), `cbo_code` string(6), `prescribed_at`, `suspension_reason` text (cifrado), `status_changed_at`, `status_changed_by_user_id` (FK users; Divergência A9), timestamps. CHECKs: status em `PRESCRIPTION_STATUSES`; `version >= 1 AND (version = 1) = (supersedes_id IS NULL)`; `(status = 'suspended') = (suspension_reason IS NOT NULL)`; `(status = 'active') = (status_changed_at IS NULL) AND (status_changed_at IS NULL) = (status_changed_by_user_id IS NULL)`. Índices únicos `idx_urgency_prescriptions_version (urgency_encounter_id, version)` e `idx_urgency_prescriptions_active (urgency_encounter_id) WHERE status = 'active'`.
  - `urgency_prescription_items`: `urgency_prescription_id` (FK), `position` smallint (1..20, único por prescrição), `catalog_item_id` uuid NOT NULL (sem FK: plataforma), `catmat_code` integer NOT NULL, `controlled_list` string(3) (P1; Divergência A8), `dose` numeric(10,3) > 0, `dose_unit` string(20) (lista `DOSE_UNITS`), `route` string(20) (`PrescriptionItem::ROUTES`), `frequency_kind`, `interval_hours` smallint, `max_per_day` smallint, `first_dose_at` NOT NULL, `instructions` text (cifrado), `stock_quantity_per_dose` numeric(12,2) NOT NULL DEFAULT 1 (`> 0 AND <= 99999.99`), `stock_unit` string(20) NOT NULL (Divergência A7), `created_at`. CHECKs: `(frequency_kind = 'interval') = (interval_hours IS NOT NULL)`, `interval_hours BETWEEN 1 AND 24`, `(frequency_kind = 'if_needed') = (max_per_day IS NOT NULL)`, `max_per_day BETWEEN 1 AND 24`, listas fechadas.
  - `urgency_administrations`: `urgency_prescription_item_id` (FK), `urgency_encounter_id` (FK), `scheduled_at`, `status`, `reason` text (cifrado), `administered_at`, `stock_lot_id` (FK `stock_lots`), `stock_quantity` numeric(12,2), `recorded_by_user_id` (FK), `professional_link_id` (FK), `cbo_code` string(6), `recorded_at`, `created_at`. CHECKs: status em `ADMINISTRATION_STATUSES`; `(status = 'administered') = (administered_at IS NOT NULL)`; `(status = 'administered') = (reason IS NULL)`; `(stock_lot_id IS NULL) = (stock_quantity IS NULL) AND (stock_lot_id IS NULL OR status = 'administered')`; `stock_quantity IS NULL OR stock_quantity > 0`. Índices `idx_urgency_administrations_slot (urgency_prescription_item_id, scheduled_at) WHERE scheduled_at IS NOT NULL` único, `(urgency_encounter_id, recorded_at)`, `(urgency_prescription_item_id, administered_at)`.
  - Triggers: `rota_urgency_prescription_guard()` — re-cifra de `suspension_reason` primeiro; INSERT: encontro não `closed`, nasce `active`, `supersedes_id` do mesmo encontro; UPDATE: só `status`, `status_changed_at`, `status_changed_by_user_id`, `suspension_reason`, `updated_at`; `OLD.status <> 'active'` → `'urgency_prescriptions: status only moves forward'`; DELETE recusado. `rota_urgency_prescription_item_insert_guard()` — a prescrição está `active` (`'urgency_prescription_items: prescription is not active'`); itens: `rota_reencryption_only('instructions')`. `rota_urgency_administration_insert_guard()` — o encontro não está `closed` (`'urgency_administrations: the encounter is closed'`), o item é de prescrição `active` do MESMO encontro; administrações: `rota_reencryption_only('reason')`. TRUNCATE recusado nas três.
- Produces (Ruby): `UrgencyPrescription` (`encrypts :suspension_reason`, `has_many :items, -> { order(:position) }`, `belongs_to :supersedes, optional`, `scope :active_versions`), `UrgencyPrescriptionItem` (`encrypts :instructions`, `#controlled?`), `UrgencyAdministration` (`encrypts :reason`, `belongs_to :stock_lot, optional`); `UrgencyEncounter has_many :prescriptions` e `#active_prescription`; os quatro `[Modelo, :coluna]` em `CITY_KEYED_TARGETS`.

- [ ] **Step 1: Write the failing test** `spec/models/urgency_prescription_schema_spec.rb`: (a) duas `active` no mesmo encontro → `idx_urgency_prescriptions_active`; (b) `active → superseded` aceito, `superseded → active` → `status only moves forward`; `suspended` sem motivo → `ck_..._suspended`; (c) item novo numa versão `superseded` → `prescription is not active`; `interval` sem `interval_hours` → CHECK; (d) administração: `update_column(:status, …)` → `append-only: UPDATE refused`; com `rota.reencrypting = 'on'` e só `reason` mudando, aceita; dois registros no mesmo `(item, scheduled_at)` → `idx_urgency_administrations_slot`; administração de encontro `closed` → `the encounter is closed`; `refused` sem `reason` → CHECK; (e) `spec/architecture/city_encrypted_attributes_guard_spec.rb` cobre os quatro atributos.
- [ ] **Step 2:** `bin/test api -w mod32b spec/models/urgency_prescription_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração, o dump, os triggers e os modelos.
- [ ] **Step 4:** os mesmos specs + `spec/architecture/urgency_merge_tables_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add versioned urgency prescriptions and append-only administrations`.

**Relato ao coordenador:** "api 32b · Task 3 · migração 20261110900002 (prescrições em versões, itens com controlled_list/stock_unit, administrações imutáveis); 4 campos cifrados em CITY_KEYED_TARGETS."

---

### Task 4: Estoque — `administration` no razão e as faltas (migração 900003, transversal 27a)

**Files:**
- Create: `db/city_migrate/20261110900003_add_urgency_stock_deduction.rb`, `app/models/urgency_stock_shortfall.rb`, `spec/models/urgency_stock_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql` (bloco "Módulo 32b — estoque"), `app/services/pharmacy/{catalog,ledger,statement}.rb`, `app/models/{stock_movement,urgency_administration}.rb`, `app/services/city_encryption.rb`, `spec/models/pharmacy_schema_spec.rb`, `spec/services/pharmacy/ledger_spec.rb` (ou o spec do Ledger existente)

**Interfaces:**
- Produces (banco): `stock_movements.urgency_administration_id` (FK, índice `idx_stock_movements_urgency_administration`); a migração define `MOVEMENT_KINDS`, `POSITIVE_KINDS`, `NEGATIVE_KINDS` (cópias de `Pharmacy::Catalog`) e refaz `ck_stock_movements_kind`, `ck_stock_movements_sign` e `ck_stock_movements_document` pelo `replace_check` do 29 — o `_document` passa a `num_nonnulls(…, immunization_id, urgency_administration_id) = 1 AND … AND (urgency_administration_id IS NOT NULL) = (kind::text = 'administration'::text)`. `urgency_stock_shortfalls`: `urgency_administration_id` (FK, único), `stock_location_id` (FK, nulo), `catalog_item_id` uuid NOT NULL, `quantity` numeric(12,2) > 0, `cause`, `reconciled_at`, `reconciled_by_user_id` (FK), `reconciliation_note` text (cifrado), timestamps; CHECKs `cause` em `SHORTFALL_CAUSES`, `(cause = 'no_location') = (stock_location_id IS NULL)`, `(reconciled_at IS NULL) = (reconciled_by_user_id IS NULL) AND (reconciled_at IS NULL) = (reconciliation_note IS NULL)`; índice `(stock_location_id, reconciled_at, created_at)`. Trigger `rota_urgency_shortfall_guard()`: re-cifra primeiro; conciliada nunca muda (`'urgency_stock_shortfalls: reconciled once'`); aberta só recebe os três campos da conciliação e `updated_at`; DELETE e TRUNCATE recusados.
- Produces (Ruby): Valor fixado 4; `StockMovement belongs_to :urgency_administration, optional: true`; `UrgencyAdministration has_one :stock_movement, has_one :shortfall`; `UrgencyStockShortfall` (`encrypts :reconciliation_note`, `scope :open_shortfalls`); `[UrgencyStockShortfall, :reconciliation_note]` em `CITY_KEYED_TARGETS`.

- [ ] **Step 1: Write the failing tests.** `pharmacy_schema_spec.rb:236`: a igualdade do 29 vira `[AddVaccinationRoom::MOVEMENT_KINDS, …] == [catalog::MOVEMENT_KINDS - %w[administration], catalog::POSITIVE_KINDS, catalog::NEGATIVE_KINDS - %w[administration]]` e entra `[AddUrgencyStockDeduction::MOVEMENT_KINDS, …POSITIVE…, …NEGATIVE…] == [catalog::…]`; o `vaccine` da linha 238 ganha `administration` na conta do 27a. `urgency_stock_schema_spec.rb`: um movimento `administration` positivo → `ck_stock_movements_sign`; `administration` sem `urgency_administration_id` → `ck_stock_movements_document`; `dispensing` com `urgency_administration_id` → `_document`; a falta conciliada uma vez e só. Ledger: `Pharmacy::Ledger.record!(location:, lot:, kind: "administration", quantity: BigDecimal("1"), by:, document: administração)` grava −1 com o documento; o saldo que ficaria negativo → `Insufficient`; `Pharmacy::Statement` devolve `document: { type: "urgency_administration", id:, number: nil }`.
- [ ] **Step 2:** `bin/test api -w mod32b spec/models/pharmacy_schema_spec.rb spec/models/urgency_stock_schema_spec.rb spec/services/pharmacy` → FAIL.
- [ ] **Step 3: Implementar** a migração (comentário: "ADR 0050, contrato §1 e §6: o mesmo caminho do 29, Desvio 7"), o dump, o trigger, o Catalog, o Ledger (`DOCUMENTS` e o comentário de `record!` com o 7º documento), o Statement e os modelos.
- [ ] **Step 4:** os mesmos specs + `spec/requests/pharmacy spec/commands/pharmacy spec/commands/vaccination` → PASS.
- [ ] **Step 5: Commit** `feat: add the administration stock movement and urgency shortfalls`.

**Relato ao coordenador:** "api 32b · Task 4 · migração 20261110900003: stock_movements kind administration (+ FK) e urgency_stock_shortfalls; extrato do 27a ganha document.type `urgency_administration` (dashboard precisa do rótulo)."

---

### Task 5: Cadastro de leitos (admin, §3)

**Files:**
- Create: `app/commands/urgency/save_bed.rb`, `app/controllers/urgency/beds_controller.rb`, `spec/requests/urgency/beds_spec.rb`
- Modify: `config/routes.rb` (scope `/urgency`), `app/controllers/urgency/base_controller.rb` (`FAILURE_STATUS`: `bed_occupied: :conflict`), `app/controllers/health_units_controller.rb` (`upa_change_conflict` e `deactivation_conflict`), `spec/requests/health_units_spec.rb`

**Interfaces:**
- Produces: `GET /urgency/beds?health_unit_id` → `{ items: [{ id, health_unit_id, label, active, occupied }] }` por rótulo; `POST /urgency/beds {health_unit_id, label, active?}` → 201 `{ bed }`; `PATCH /urgency/beds/:id {label?, active?}` → 200 `{ bed }`. Só `municipal_admin` (403 `missing_role`), escrita com step-up (401 `mfa_required`). `Urgency::SaveBed.call(attrs:, by:, bed: nil)` no padrão do `Psychosocial::SaveBed`: chave desconhecida → 422; unidade inexistente, inativa ou de outro id → 422 `invalid_request` `health_unit_id`; unidade que não é UPA → 422 `not_an_upa`; rótulo fora de 1..20 → 422; rótulo repetido → **422 `label_taken`** (contrato; o CAPS usa 409 `bed_label_taken` — Divergência A2); desativar com estadia aberta → 409 `bed_occupied` (sob `bed.lock!`, conferência da estadia aberta). Sem evento. `HealthUnitsController`: unidade `upa` com leito ativo não muda de `kind` nem é desativada → **409** `upa_has_beds` (Divergência A3).

- [ ] **Step 1: Write the failing test** `spec/requests/urgency/beds_spec.rb`: cria, lista, renomeia e desativa; UBS → `not_an_upa`; rótulo repetido → 422 `label_taken` com `problems`; leito ocupado → 409 `bed_occupied`; o enfermeiro → 403; sem step-up → 401. `health_units_spec.rb`: `PATCH` com `kind: "ubs"` numa UPA com leito ativo e `POST …/deactivate` → 409 `upa_has_beds`; com os leitos desativados, passam.
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/urgency/beds_spec.rb spec/requests/health_units_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/psychosocial/admin_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add upa observation bed registry for municipal admins`.

**Relato ao coordenador:** "api 32b · Task 5 · /urgency/beds pronto; label_taken 422, bed_occupied 409, upa_has_beds 409 (A2, A3)."

---

### Task 6: Observação — mapa, entrada, troca de leito, saída e o campo da fila (§4)

**Files:**
- Create: `app/services/urgency/{observation,input}.rb`, `app/commands/urgency/{start_observation,end_observation}.rb`, `app/controllers/urgency/observations_controller.rb`, `spec/services/urgency/observation_spec.rb`, `spec/requests/urgency/observation_spec.rb`, `spec/support/urgency_observation_helpers.rb`
- Modify: `config/routes.rb`, `app/services/urgency/queue.rb` (`in_care_item`), `app/controllers/urgency/base_controller.rb` (`bed_taken: :conflict`), `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/services/urgency/queue_spec.rb`

**Interfaces:**
- Produces: `Urgency::Observation.hours(first_started_at, now) -> Integer`, `.alert(hours) -> nil | "near_limit" | "over_limit"`, `.board(unit, now:) -> Hash` (forma do contrato §4: `{ beds: [{ bed_id, label, active, stay: null | { stay_id, encounter_id, ticket, patient: <patient-ref>, started_at, hours, alert } }], waiting_bed: [{ encounter_id, ticket, patient: <patient-ref>, called_at }] }` — `waiting_bed` = encontros `in_care` da UPA sem estadia aberta, por `called_at` (Divergência A1)); `.queue_fields(encounter_ids, now:) -> { encounter_id => { bed_label, hours, alert } }`.
- `GET /urgency/observation?health_unit_id` → `Observation.board`; leitura `observation_reader?` (senão `Access.refusal`); unidade que não é UPA → 422 `not_an_upa`; sem trilha.
- `POST /urgency/encounters/:id/observation {bed_id}` → 201 `{ encounter: <detalhe> }` pelo `Urgency::StartObservation.call(encounter:, bed_id:, by:, now:)`: só `clinical_link` (2235\*/225\*); `bed_id` malformado → 422 `invalid_request`; trava atendimento FOR SHARE → encontro FOR NO KEY UPDATE → estadia aberta FOR UPDATE; encontro `closed` → 409 `encounter_closed`; fora de `in_care` → 422 `not_in_care`; leito inexistente, inativo ou de outra unidade → 422 `bed_unavailable`; o mesmo leito da estadia aberta → 422 `invalid_request` `bed_id`; leito ocupado (Ruby) → 409 `bed_taken`; com estadia aberta, fecha com `bed_change` e abre a nova em `now`; `rescue StatementInvalid` com `excl_urgency_bed_stays_bed` → 409 `bed_taken` e `idx_urgency_bed_stays_open_encounter` → 409 `bed_taken`. Eventos: na troca `urgency.observation_ended` (estadia velha) e `urgency.observation_started` (nova); na entrada só o segundo.
- `POST /urgency/encounters/:id/observation/end {}` → 200 `{ encounter }` pelo `Urgency::EndObservation`: sem estadia aberta → 422 `not_in_observation`; fecha com `observation_ended`; evento `urgency.observation_ended`.
- Fila: `in_care[]` ganha `observation: { bed_label, hours, alert } | null`.

- [ ] **Step 1: Write the failing tests.** `observation_spec.rb` (serviço): `hours` com `travel_to` (19 h 59 → 19, `nil`; 20 h → `near_limit`; 24 h → `over_limit`); a troca de leito às 23 h mantém a contagem desde a primeira entrada (Review Focus 4). Requisição:

```ruby
it "dois profissionais, dois pacientes, o mesmo leito ao mesmo tempo: um vence, o outro 409 bed_taken (Review Focus 1)" do
  a, b = in_care_encounter!(upa), in_care_encounter!(upa)
  results = Concurrent::Array.new
  [ a, b ].map do |encounter|
    Thread.new do
      CityConnection.with(city) do
        results << Urgency::StartObservation.call(encounter: encounter, bed_id: bed.id, by: doctor)
      end
    end
  end.each(&:join)
  expect(results.map { |r| r.success? ? :ok : r.reason }).to contain_exactly(:ok, :bed_taken)
  expect(UrgencyBedStay.open_stays.where(urgency_bed_id: bed.id).count).to eq(1)
end
```

(no padrão do spec de concorrência do 09/31: `pop(timeout:)` e limpeza no `after`). Mais: entrada → 201 e evento `observation_started` só com `encounter_id`, `stay_id`; troca → dois eventos e o leito antigo livre no mapa; `not_in_care` (classificado, não chamado); `bed_unavailable` (leito de outra UPA e inativo); saída → `not_in_observation` na segunda vez; o técnico 3222 lê o mapa e não entra com paciente (403 `cbo_not_allowed`); a recepção lê o mapa; o mapa não grava `clinical_record.viewed`; `GET /urgency/observation?cpf=1` → 422 `path: "cpf"`; a fila traz `observation.bed_label` do paciente no leito e `null` dos demais.
- [ ] **Step 2:** `bin/test api -w mod32b spec/services/urgency/observation_spec.rb spec/requests/urgency/observation_spec.rb spec/services/urgency/queue_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`Urgency::Input` no padrão de `Psychosocial::Input`: `uuid?`, `time`; comentário no topo dos comandos com as recusas e a ordem de trava, como o `start_night_stay.rb`).
- [ ] **Step 4:** os mesmos specs + `spec/requests/urgency/calls_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add the upa observation map, bed entry, bed change and exit`.

**Relato ao coordenador:** "api 32b · Task 6 · mapa (`waiting_bed` = em atendimento sem leito, A1), entrada/troca/saída com exclusion; fila ganhou `observation`; eventos observation_started/ended."

---

### Task 7: Prescrição interna — emitir, alterar e suspender (§5)

**Files:**
- Create: `app/commands/urgency/{issue_prescription,suspend_prescription}.rb`, `app/services/urgency/prescription_json.rb`, `app/controllers/urgency/prescriptions_controller.rb`, `spec/requests/urgency/prescriptions_spec.rb`
- Modify: `config/routes.rb`, `app/controllers/urgency/base_controller.rb` (`prescription_not_active`, `consultation_required`, `catalog_item_invalid` → 422), `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `POST /urgency/encounters/:id/prescriptions {items: [...]}` → 201 `{ prescription: <prescription-json> }`. O controlador lê `MedicationCatalogItem.prescribable.where(id: ids)` **fora** da transação e passa `catalog:`. `Urgency::IssuePrescription.call(encounter:, items:, catalog:, by:, now:)`:
  - acesso: `physician_link` (senão `Access.refusal`);
  - forma (todas as recusas em `problems`, path `items.N.<campo>`): 1..20 itens (`items` → `invalid_request`); chave desconhecida no item ou em `frequency`; `catalog_item_id` fora do catálogo prescritível → `catalog_item_invalid`; `dose` fora de `DOSE` ou zero; `dose_unit` fora de `DOSE_UNITS`; `route` fora de `PrescriptionItem::ROUTES`; `frequency.kind` fora da lista; `interval_hours` só e sempre em `interval` (1..24); `max_per_day` só e sempre em `if_needed` (1..24); `first_dose_at` (padrão `now`, truncado ao minuto) fora de `FIRST_DOSE_WINDOW`; `instructions` > 500; `stock_quantity_per_dose` (String, `Pharmacy::Quantity.parse`, padrão `"1"`);
  - estado (atendimento FOR SHARE → encontro FOR NO KEY UPDATE → prescrição ativa FOR UPDATE): encontro `closed` → 409 `encounter_closed`; sem consulta `urgency` (`encounter.consultation_id` nulo) → 422 `consultation_required`;
  - grava: a `active` anterior → `superseded` (`status_changed_at`, `status_changed_by_user`), a nova `version = max + 1` com `supersedes`, os itens com `catmat_code`, `controlled_list` (P1: qualquer lista, sem a regra do 19d), `stock_unit = Pharmacy::DispensingUnit.for(catalog_item)`; evento `urgency.prescription_issued`.
- `POST /urgency/prescriptions/:id/suspend {reason}` → 200 `{ prescription }` pelo `Urgency::SuspendPrescription`: `physician_link` da UPA do encontro; `reason` fora de `SUSPENSION_REASON` → 422 `invalid_request` `reason`; encontro `closed` → 409 `encounter_closed`; versão não `active` → 422 `prescription_not_active`; evento `urgency.prescription_suspended`.
- `<prescription-json>` (`Urgency::PrescriptionJson.call(prescription, catalog:)`): `{ id, encounter_id, version, status, supersedes_id, prescribed_at, prescribed_by: { name, cbo_code }, status_changed_at, suspension_reason, items: [{ id, position, catalog_item: <Pharmacy::Json.catalog_item>, controlled: bool, controlled_list, dose, dose_unit, route, frequency: { kind, interval_hours, max_per_day }, first_dose_at, instructions, stock_quantity_per_dose, stock_unit }] }`.

- [ ] **Step 1: Write the failing test** `spec/requests/urgency/prescriptions_spec.rb`: emite v1 → 201 com os itens e o evento só com ids; emite de novo → v2 `active` com `supersedes_id` = v1, v1 `superseded` e os itens de v1 intactos; diazepam (B1) aceito, com `controlled: true, controlled_list: "B1"` (P1); item escondido → 422 `catalog_item_invalid` `path: "items.0.catalog_item_id"`; `dose: 500` (número) → 422 `items.0.dose`; `interval` sem `interval_hours` e `if_needed` com `interval_hours` → os dois em `problems`; sem consulta → `consultation_required`; enfermeiro → 403 `cbo_not_allowed`; 21 itens → 422; suspende com motivo curto → 422, com motivo → `suspended` e a segunda suspensão → `prescription_not_active`; encontro fechado → 409; nenhum log da requisição contém o `active_ingredient`, a dose ou as instruções (`capture_log`).
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/urgency/prescriptions_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add versioned internal prescriptions for upa physicians`.

**Relato ao coordenador:** "api 32b · Task 7 · prescrição em versões pronta; quantidades e dose como String (A6), item guarda controlled_list e stock_unit (A7, A8)."

---

### Task 8: Aprazamento (§5)

**Files:**
- Create: `app/services/urgency/schedule.rb`, `app/controllers/urgency/medication_schedules_controller.rb`, `spec/services/urgency/schedule_spec.rb`, `spec/requests/urgency/medication_schedule_spec.rb`
- Modify: `config/routes.rb`, `app/services/urgency/trail.rb` (`RESOURCES` + `medication_schedule`)

**Interfaces:**
- `Urgency::Schedule.slots(item, now:) -> [Time]` (Desvio 5); `.slot?(item, at, now:) -> Boolean`; `.if_needed_used(item, at:) -> Integer` (administradas nas 24 h antes de `at`); `.call(encounter, now:) -> { prescription_id, version, items: [{ item_id, …campos do item, slots: [{ scheduled_at, administration: <adm> | null }], recent: [<adm>] | null, remaining: Integer | null }] } | { prescription_id: nil, items: [] }`.
- `GET /urgency/encounters/:id/medication_schedule` → o `Schedule.call`; leitura `nursing_or_physician_link`; grava `Urgency::Trail.viewed!(resource: "medication_schedule")`; nada na query.
- `<adm>` = `{ id, status, scheduled_at, administered_at, reason, stock_lot: { id, lot_number, expires_on } | null, stock_quantity, recorded_at, by: { name, cbo_code } }`.

- [ ] **Step 1: Write the failing tests.** Serviço: `interval` 6 h com `first_dose_at` 08:00 e `now` 09:00 → 08:00, 14:00, 20:00, 02:00, 08:00 do dia seguinte; `once` → um horário; `if_needed` → `slots == []` e `remaining` desconta só `administered`. Requisição: o técnico lê e a trilha `medication_schedule` é gravada; a recepção → 403; a v2 ativa aparece e a v1 não.
- [ ] **Step 2:** `bin/test api -w mod32b spec/services/urgency/schedule_spec.rb spec/requests/urgency/medication_schedule_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: compute the urgency medication schedule on read`.

**Relato ao coordenador:** "api 32b · Task 8 · aprazamento calculado na leitura, com trilha `medication_schedule` (A19)."

---

### Task 9: Checagem sem estoque (§6)

**Files:**
- Create: `app/commands/urgency/record_administration.rb`, `app/controllers/urgency/administrations_controller.rb`, `spec/requests/urgency/administrations_spec.rb`
- Modify: `config/routes.rb`, `app/controllers/urgency/base_controller.rb` (`slot_taken: :conflict`; `slot_invalid`, `max_per_day_reached`, `stock_deduction_disabled`, `lot_invalid` → 422), `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `POST /urgency/administrations {prescription_item_id, scheduled_at?, status, reason?, administered_at?, stock_lot_id?, stock_quantity?}` → 201 `{ administration: <adm>, shortfall: { id, cause } | null }`. O controlador lê `Urgency::StockGate.usable?(Current.city)` fora da transação e passa `stock:`. `Urgency::RecordAdministration.call(attrs:, by:, stock:, now:)`:
  - item inexistente → 404; acesso `nursing_or_physician_link` na UPA do encontro do item;
  - travas: atendimento FOR SHARE → encontro FOR SHARE → prescrição FOR SHARE; encontro `closed` → 409 `encounter_closed`; prescrição não `active` → 422 `prescription_not_active`;
  - forma (Desvio 6): `status`; `reason` só e sempre fora de `administered`; `administered_at`; `scheduled_at` obrigatório em `once`/`interval` e proibido em `if_needed`; `stock:` falso com `stock_lot_id`/`stock_quantity` → 422 `stock_deduction_disabled`; os dois fora de `administered` → `invalid_request`;
  - horário: `Schedule.slot?` falso → 422 `slot_invalid`; já checado → 409 `slot_taken` (Ruby, e o `rescue RecordNotUnique` de `idx_urgency_administrations_slot`); `if_needed` + `administered` com `if_needed_used >= max_per_day` → 422 `max_per_day_reached`;
  - grava a administração (`recorded_at: now`, link e CBO) e publica `urgency.medication_checked`. Com `stock:` verdadeiro e `administered`, chama `Urgency::DeductStock` (Task 10) dentro da mesma transação, ANTES de gravar a administração (o savepoint do Desvio 7).

- [ ] **Step 1: Write the failing test** `spec/requests/urgency/administrations_spec.rb` (interruptor desligado): administra o horário das 08:00 → 201 sem movimento (`StockMovement.count` igual) e sem falta; o mesmo horário de novo → 409 `slot_taken`; 08:30 → 422 `slot_invalid`; `refused` sem motivo → 422 `reason`; com motivo → 201; `if_needed` com `max_per_day: 2` → a terceira em 24 h → 422 `max_per_day_reached` (a recusada não conta); `stock_lot_id` no corpo → 422 `stock_deduction_disabled`; o técnico 3222 checa (201, `cbo_code: "322205"`); a recepção → 403; item de v1 depois da v2 → `prescription_not_active`; o evento leva só `encounter_id` e `administration_id`.
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/urgency/administrations_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (com `DeductStock` ainda ausente: a chamada entra na Task 10; aqui o ramo `stock:` verdadeiro fica coberto por spec pendente).
- [ ] **Step 4:** o spec + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: record nursing medication checks against the schedule`.

**Relato ao coordenador:** "api 32b · Task 9 · checagem imutável sem estoque; códigos slot_invalid/slot_taken/max_per_day_reached/stock_deduction_disabled prontos para o dashboard."

---

### Task 10: Baixa de estoque por FEFO e faltas (§6, R7; transversal 27a)

**Files:**
- Create: `app/commands/urgency/deduct_stock.rb`, `spec/commands/urgency/deduct_stock_spec.rb`, `spec/requests/urgency/administrations_stock_spec.rb`
- Modify: `app/commands/urgency/record_administration.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `Urgency::DeductStock.call(item:, encounter:, lot_id:, quantity:, today:) -> { location:, lot:, quantity: } | { shortfall_cause:, location:, quantity: } | Result.fail(:lot_invalid)` — o Desvio 7 inteiro, sob a ordem lotes → local. `RecordAdministration` monta o savepoint: `UrgencyAdministration.transaction(requires_new: true) { adm = create!(…, stock_lot:, stock_quantity:); Pharmacy::Ledger.record!(location:, lot:, kind: "administration", quantity:, by:, document: adm, at: now) }`; `rescue Pharmacy::Ledger::Insufficient` (dentro da transação de fora) → grava a administração sem lote e a falta `lot_unavailable`. Falta: `UrgencyStockShortfall.create!(urgency_administration:, stock_location: (nil em no_location), catalog_item_id:, quantity:, cause:)` e `urgency.stock_shortfall_recorded {shortfall_id}`.

- [ ] **Step 1: Write the failing tests.**

```ruby
RSpec.describe Urgency::DeductStock do
  # pharmacy_city! + urgency_city! + StockGate ligado; a UPA com unit_location!(upa)
  it "FEFO: o lote não vencido de menor validade com saldo; empate pelo número e pelo id" do
    late = stocked_lot!(location, item: losartana, quantity: "30", lot_number: "B2", expires_on: today + 300)
    early = stocked_lot!(location, item: losartana, quantity: "30", lot_number: "B1", expires_on: today + 30)
    stocked_lot!(location, item: losartana, quantity: "30", lot_number: "A0", expires_on: today - 1)
    expect(record!(item).stock_lot_id).to eq(early.id)
    expect(balance_of(location, early)).to eq(29)
    expect(balance_of(location, late)).to eq(30)
  end

  it "sem local unit ativo: a administração fica e nasce no_location sem local" do
    location.update!(active: false)
    adm = record!(item)
    expect(adm.stock_lot_id).to be_nil
    expect(adm.shortfall).to have_attributes(cause: "no_location", stock_location_id: nil, quantity: 1)
  end

  it "sem lote com saldo suficiente: insufficient_balance; o lote do corpo curto: lot_unavailable; vencido: 422 lot_invalid" do
    lot = stocked_lot!(location, item: losartana, quantity: "1")
    expect(record!(item, stock_quantity: "2").shortfall.cause).to eq("insufficient_balance")
    expect(record!(item, slot: 1, stock_lot_id: lot.id, stock_quantity: "2").shortfall.cause).to eq("lot_unavailable")
    old = stocked_lot!(location, item: losartana, quantity: "5", expires_on: today - 1)
    expect(result!(item, slot: 2, stock_lot_id: old.id).reason).to eq(:lot_invalid)
    expect(balance_of(location, lot)).to eq(1)
  end
end
```

Requisição (Review Focus 2): um lote com saldo 1, duas checagens de itens diferentes em threads → as duas 201; uma com `stock_lot`, a outra com `shortfall.cause == "insufficient_balance"`; saldo 0, nunca −1; evento `urgency.stock_shortfall_recorded` só com `shortfall_id`. Controlado (diazepam B1) com o interruptor ligado gera o movimento `administration` (P1). Lote de unidade de dispensação diferente do `stock_unit` → FEFO o ignora; no corpo → `lot_invalid`. Lote com `StockLotBlock` aberto → ignorado / `lot_invalid`.
- [ ] **Step 2:** `bin/test api -w mod32b spec/commands/urgency/deduct_stock_spec.rb spec/requests/urgency/administrations_stock_spec.rb spec/requests/urgency/administrations_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (comentário no topo: a ordem de trava e por que o savepoint).
- [ ] **Step 4:** os mesmos specs + `spec/services/pharmacy spec/requests/pharmacy/positions_spec.rb spec/commands/vaccination/record_immunization_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: deduct upa medication checks from stock by fefo with shortfalls`.

**Relato ao coordenador:** "api 32b · Task 10 · baixa por FEFO no local unit da UPA, faltas no_location/insufficient_balance/lot_unavailable; um lote por administração, sem fracionar (A10, A11, A12)."

---

### Task 11: Faltas da UPA na farmácia (§7)

**Files:**
- Create: `app/commands/pharmacy/reconcile_urgency_shortfall.rb`, `app/controllers/pharmacy/urgency_shortfalls_controller.rb`, `spec/requests/pharmacy/urgency_shortfalls_spec.rb`
- Modify: `config/routes.rb` (scope `/pharmacy`), `app/services/pharmacy/alerts.rb`, `app/controllers/pharmacy/base_controller.rb` (`already_reconciled: :conflict`), `spec/requests/pharmacy/positions_spec.rb` (515, 517, 534, 557), `spec/requests/vaccination/cold_chain_spec.rb` (só se o `include` quebrar), `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `GET /pharmacy/urgency_shortfalls?location_id&state=open|reconciled&before&limit` (Divergência A4: `location_id`, o nome do 27a) → `{ items: [{ shortfall_id, created_at, stock_location_id, catalog_item: <Pharmacy::Json.catalog_item>, quantity, cause, administration: { administered_at, ticket }, reconciled_at, reconciled_by: { name } | null }], next_before }`, por `created_at DESC, id DESC`, cursor do 27a. `state` padrão `open`. Com `location_id`: `required_location` + `Pharmacy::Access.allowed?(roles: %w[pharmacist stock_keeper])` ou `central_keeper?` (senão 403); local de vacina → 404. Sem `location_id`: só o almoxarife do central, e a lista inclui as `no_location` (Divergência A5); os demais → 403. Sem nome, CPF, CNS, dose, via nem instruções; sem trilha.
- `POST /pharmacy/urgency_shortfalls/:id/reconcile {note}` → 200 `{ shortfall }` pelo `Pharmacy::ReconcileUrgencyShortfall.call(shortfall:, note:, by:)`: a mesma regra de papel (central keeper para `no_location`); `note` fora de `RECONCILIATION_NOTE` → 422 `note`; sob `shortfall.lock!`, já conciliada → 409 `already_reconciled`; nenhum movimento; evento `pharmacy.urgency_shortfall_reconciled {shortfall_id}`.
- `Pharmacy::Alerts`: `urgency_shortfalls` = `UrgencyStockShortfall.open_shortfalls.where(stock_location_id: location.id).count`.

- [ ] **Step 1: Write the failing test** `spec/requests/pharmacy/urgency_shortfalls_spec.rb`: a farmacêutica do local lê as abertas sem nome do paciente (`response.body` não contém o `full_name`); a auxiliar (`pharmacy_assistant`) → 403; o almoxarife do central sem `location_id` vê a `no_location`; concilia com nota curta → 422, com nota → 200 e `StockMovement.count` igual, de novo → 409; `state=reconciled` lista a conciliada; com `urgency_stock_deduction` desligado depois, a lista e a conciliação seguem (Review Focus 5); com `pharmacy` desligado → 403 `feature_disabled`; `/pharmacy/alerts` traz `urgency_shortfalls` (e o `positions_spec` com as 6 chaves).
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/pharmacy/urgency_shortfalls_spec.rb spec/requests/pharmacy/positions_spec.rb spec/requests/vaccination/cold_chain_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add the upa shortfall queue and reconciliation to the pharmacy`.

**Relato ao coordenador:** "api 32b · Task 11 · /pharmacy/urgency_shortfalls (parâmetro `location_id`, A4; no_location só para o almoxarife do central, A5) e alerts.urgency_shortfalls (6ª chave)."

---

### Task 12: Desfecho fecha a observação e a prescrição (§4, §5; transversal 32a)

**Files:**
- Modify: `app/commands/urgency/record_outcome.rb`, `spec/requests/urgency/outcomes_spec.rb`, `spec/requests/urgency/observation_spec.rb`; e, só se a Divergência A21 for aceita pelo coordenador, `app/commands/attendances/close.rb` + `spec/commands/attendances/close_spec.rb`

**Interfaces:**
- `Urgency::RecordOutcome#persist`: depois de `attendance.lock!` → `encounter.lock!` e do `state_refusal`, trava a estadia aberta (`UrgencyBedStay.open_stays.lock.find_by(urgency_encounter_id:)`) e a prescrição ativa (`…active_versions.lock.find_by`), nessa ordem; fecha a estadia com `ended_at: now, end_kind: "outcome", ended_by_user: by` e leva a prescrição a `ended` (`status_changed_at: now`, `status_changed_by_user: by`), antes do `encounter.update!(state: "closed")`. Publica `urgency.observation_ended` quando havia estadia (Divergência A20). Uma recusa do `Attendances::Close` faz `Rollback` de tudo (como hoje).
- (A21, condicional) `Attendances::Close` com `urgency_care` desligado numa UPA: fecha a estadia aberta do encontro do atendimento com `observation_ended`, sem tocar no encontro e na prescrição.

- [ ] **Step 1: Write the failing tests.** `outcomes_spec.rb`: alta de quem está no leito → a estadia fechada com `outcome` no mesmo `ended_at` do `recorded_at`, a prescrição `ended`, o leito livre no mapa, `observation_ended` publicado; `left_without_care` durante a observação também fecha; recusa do desfecho (`consultation_not_finalized`) deixa a estadia aberta e a prescrição `active`. Review Focus 3: uma thread grava o desfecho e outra checa o mesmo encontro → ou a checagem vem antes (201 e fica) ou depois (409 `encounter_closed`); nunca `PG::TRDeadlockDetected` (`expect { … }.not_to raise_error`), com o lote do FEFO envolvido (interruptor ligado).
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/urgency/outcomes_spec.rb spec/requests/urgency/observation_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (atualizar o comentário do topo: "Ordem de trava: attendance → encounter → estadia → prescrição (contrato 32b §4)").
- [ ] **Step 4:** os mesmos specs + `spec/requests/urgency spec/commands/attendances` → PASS.
- [ ] **Step 5: Commit** `feat: close the observation stay and end the prescription on urgency outcome`.

**Relato ao coordenador:** "api 32b · Task 12 · o desfecho fecha estadia (outcome) e prescrição (ended) na mesma transação; observation_ended publicado (A20); A21 <aplicada | aguardando>."

---

### Task 13: BPA da observação (§8; transversal 22)

**Files:**
- Modify: `app/commands/billing/from_urgency.rb`, `spec/commands/billing/from_urgency_spec.rb`

**Interfaces:**
- `Billing::FromUrgency#lines`: a linha do atendimento médico usa `codes["observation_code"]` no lugar de `codes["care_code"]` quando `UrgencyBedStay.exists?(urgency_encounter_id:)`; o resto da linha (médico, CBO, dia, CID) igual. A classificação não muda. Sem consulta finalizada, nenhuma das duas (a regra de hoje). `0301100012` nunca sai (P2).

- [ ] **Step 1: Write the failing test:** encontro com estadia e consulta finalizada com CID I10 → duas linhas, `0301060118` e `0301060029`, e nenhuma `0301060096`; reprocessar o evento não duplica; sem estadia → `0301060096` como hoje; com estadia e `registered_in_error`… impossível depois da consulta finalizada (Q30) — o caso coberto é a evasão `left_without_care` com estadia e sem consulta finalizada: só a classificação. Nenhuma linha `0301100012` com administrações gravadas. A crítica da SIGTAP de dev aceita a linha com I10 (a amostra restringe os CIDs do `0301060029`; Divergência A26).
- [ ] **Step 2:** `bin/test api -w mod32b spec/commands/billing/from_urgency_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (comentário: "Contrato 32b §8: observação até 24 h no lugar do atendimento médico; gate SIGTAP api#136").
- [ ] **Step 4:** o spec + `spec/jobs/billing spec/services/billing spec/config/urgency_sigtap_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: bill urgency observation in place of the physician care line`.

**Relato ao coordenador:** "api 32b · Task 13 · BPA: 0301060029 no lugar do 0301060096 quando há estadia; a amostra SIGTAP de dev só aceita CID I10/E119/J069 nesse código (A26)."

---

### Task 14: Linha do tempo — `observation_hours` (§9)

**Files:**
- Modify: `app/services/timeline/{entry,json}.rb`, `app/services/timeline/sources/urgency_source.rb`, `spec/services/timeline/sources/urgency_source_spec.rb` (ou o spec da fonte existente), `spec/requests/pec_timeline_spec.rb`

**Interfaces:**
- `Timeline::Entry` ganha o membro `observation_hours` com padrão nulo (`def initialize(observation_hours: nil, **rest) = super`), sem tocar nas 17 chamadas existentes. `Timeline::Json.professional` acrescenta `observation_hours` só quando `entry.kind == "urgency"` (as chaves das outras fontes, fixadas em `timeline_spec.rb:25`, não mudam). `UrgencySource` preload `:bed_stays`; `observation_hours` = horas inteiras do Desvio 2 (até o último `ended_at`, ou `now` com estadia aberta), nulo sem estadia; o resumo ganha "observação N h" (a página do PEC é HTML e lê só o resumo; Divergência A22). O cidadão não recebe nada novo (`Json.citizen` intocado).

- [ ] **Step 1: Write the failing test:** encontro com 6 h de observação → a entrada profissional traz `"observation_hours" => 6` e o resumo "observação 6 h"; sem estadia → `nil`; a página do PEC mostra "observação 6 h"; o histórico do cidadão não traz `observation_hours` nem o texto.
- [ ] **Step 2:** `bin/test api -w mod32b spec/services/timeline spec/requests/pec_timeline_spec.rb spec/requests/timeline_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/citizen` (o histórico do wpda) → PASS.
- [ ] **Step 5: Commit** `feat: show urgency observation hours on the professional and pec timelines`.

**Relato ao coordenador:** "api 32b · Task 14 · `observation_hours` só nas entradas urgency (JSON) e no resumo (PEC HTML, A22); cidadão sem mudança."

---

### Task 15: Painel do gestor — bloco `observation` (§9)

**Files:**
- Modify: `app/services/urgency/dashboard.rb`, `spec/requests/urgency/dashboard_spec.rb`

**Interfaces:**
- `Urgency::Dashboard.call` ganha `observation: { stays:, over_24h:, median_hours:, p90_hours:, bed_hours: }` sobre o mesmo recorte (chegada no mês, `NON_CARE_OUTCOMES` fora): por encontro com estadia, `hours = (COALESCE(último ended_at, now) − primeiro started_at)` em horas; `stays` = encontros com estadia; `over_24h` = os de `hours >= 24`; mediana e p90 por `percentile_cont` com uma casa decimal; `bed_hours` = soma das durações das estadias, uma casa. Supressão: `stays` e `over_24h` de 1 a 4 → `"lt5"`; mediana, p90 e `bed_hours` com `stays` de 1 a 4 → `"lt5"`, com 0 → `nil` (Divergência A30).

- [ ] **Step 1: Write the failing test:** 6 encontros com observação (2 acima de 24 h) → `stays: 6`, `over_24h: "lt5"`, mediana e p90 conferidas à mão, `bed_hours` com a troca de leito somando as duas estadias; 3 encontros → tudo `"lt5"`; nenhum → `stays: 0, median_hours: nil`; o `registered_in_error` não conta.
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/urgency/dashboard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (CTE por encontro a partir do `BASE_SQL`).
- [ ] **Step 4:** o spec → PASS.
- [ ] **Step 5: Commit** `feat: add observation occupancy to the urgency manager dashboard`.

**Relato ao coordenador:** "api 32b · Task 15 · painel: observation {stays, over_24h, median_hours, p90_hours, bed_hours}, horas com uma casa, lt5 pelo `stays` (A30)."

---

### Task 16: Detalhe do encontro — `observation`, `prescription`, `administrations` (§2)

**Files:**
- Modify: `app/services/urgency/json.rb`, `app/controllers/urgency/{base,encounters}_controller.rb` (o catálogo dos itens lido fora da transação), `spec/requests/urgency/encounter_detail_spec.rb`

**Interfaces:**
- `Urgency::Json.encounter(encounter, clinical:, catalog: {})` ganha:
  - `observation: { current: <stay> | null, stays: [<stay>], hours, alert }` com `<stay> = { stay_id, bed_id, bed_label, started_at, ended_at, end_kind, started_by: { name }, ended_by: { name } | null }` — também sem acesso clínico (sem texto clínico; Divergência A18);
  - `prescription: <prescription-json> | null` (a versão `active`) — só com `clinical: true`;
  - `administrations: [<adm> + { prescription_item_id, catalog_item, controlled }]` das últimas 48 h, de qualquer versão, por `recorded_at DESC` — só com `clinical: true`.
  - Sem acesso clínico, as chaves `prescription` e `administrations` não saem (o padrão do Q19). A trilha continua a do `render_encounter`.
- O catálogo: `Pharmacy::Json.catalog_for(ids)` dos itens da versão ativa e das administrações, lido no controlador antes de renderizar (fora de transação). Todas as respostas que devolvem o detalhe (classificação, identificação, desfecho, observação) passam pelo mesmo caminho.

- [ ] **Step 1: Write the failing test:** o médico vê os três blocos com o item controlado marcado; a recepção vê `observation` e não tem as chaves `prescription` nem `administrations`; administração de 49 h atrás não aparece; a trilha `encounter` é gravada uma vez por leitura; o desfecho devolve o detalhe com a estadia fechada e a prescrição `null`.
- [ ] **Step 2:** `bin/test api -w mod32b spec/requests/urgency/encounter_detail_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/requests/urgency` → PASS.
- [ ] **Step 5: Commit** `feat: add observation, prescription and administrations to the encounter detail`.

**Relato ao coordenador:** "api 32b · Task 16 · detalhe com observation (todos), prescription e administrations (só 2235/225; técnico usa o aprazamento, A17, A18)."

---

### Task 17: Sigilo e guarda de eventos do 32b

**Files:**
- Modify: `spec/security/urgency_secrecy_spec.rb` (cenário 32b), `spec/initializers/domain_events_bindings_spec.rb`

- [ ] **Step 1: Write the tests.**
  - `domain_events_bindings_spec.rb:582`: o regex passa a `/\A(urgency|waiting_room)\.|\Aattendance\.recalled\z|\Apharmacy\.urgency_shortfall_reconciled\z/`, a lista exata ganha os sete do Valor fixado 5 (15 nomes) e `consumed` continua só os dois do 32a (Divergência A27).
  - `urgency_secrecy_spec.rb`, `RSpec.describe "Sigilo da urgência (32b)"`, um cenário pelas rotas com o interruptor ligado: leito, entrada, troca, prescrição com instruções e um controlado, aprazamento, checagem com baixa, checagem recusada com motivo, uma falta, a conciliação com nota, o desfecho. Confere: os eventos do cenário são exatamente os sete, todos só com `_id` uuid; nenhum payload, log (`capture_log`), recusa 4xx forçada (dose número, motivo curto, nota curta, lote vencido) ou URL contém o nome, o CPF, o CNS, o `active_ingredient`, a dose, as instruções, o motivo ou a nota; o mapa e a lista de faltas não gravam `clinical_record.viewed`; `GET /urgency/observation?name=x`, `/medication_schedule?cpf=1` e `/pharmacy/urgency_shortfalls?cpf=1` → 422 com só o nome da chave; o `MODULE_GLOBS` alcança os arquivos novos (o `files.size` sobe) e nenhum chama `Platform.audit`.
- [ ] **Step 2:** `bin/test api -w mod32b spec/security/urgency_secrecy_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture` → PASS (falha → a correção vai à task dona e volta aqui).
- [ ] **Step 3: Commit** `test: pin urgency observation secrecy for events, logs, errors and urls`.

**Relato ao coordenador:** "api 32b · Task 17 · sigilo 32b fixado; lista exata de eventos com 15 nomes (A27); R18 intacto."

---

### Task 18: Semente de dev realista

**Files:**
- Modify: `lib/urgency_crew.rb`, `spec/lib/urgency_crew_spec.rb`

**Interfaces:**
- Em Curitiba, tudo pelos comandos do app, idempotente (acha pelo que já gravou), com `travel_to` até um segundo antes de agora (o `at` do crew):
  - `urgency_stock_deduction` ligado pelo mantenedor de dev (a `pharmacy` já está ligada pelo `PharmacyCrew`);
  - leitos "OBS 01" a "OBS 04" na "UPA 24h Centro" (`Urgency::SaveBed`, pelo admin de dev);
  - local "Farmácia da UPA 24h Centro" (`Pharmacy::SaveLocation` `unit`), a farmacêutica do `PharmacyCrew` vinculada como `pharmacist` (`Pharmacy::LinkStaff`), inventário de abertura (`Pharmacy::RecordEntry` `opening`) de LOSARTANA 50 mg, DIAZEPAM 10 mg (B1), PARACETAMOL 500 mg e AMOXICILINA 500 mg — e **sem** SALBUTAMOL, para nascer uma falta;
  - o técnico de enfermagem: o 322205 do `ProfessionalCrew` se tiver login (Task 0); senão `upatec@curitiba.dev` (`health_professional`, 322205, TOTP fixo);
  - "Maria Aparecida Ferreira" (58, crise hipertensiva com ansiedade): chegada há 22 h, amarela, chamada pela `upamed`, consulta `urgency` iniciada, no "OBS 01" há 20 h 30 (`near_limit`); prescrição v1: LOSARTANA 50 mg VO `once`, DIAZEPAM 10 mg VO `once` (controlado), PARACETAMOL 500 mg VO `if_needed` até 4/dia; checagens: as duas `once` administradas (com baixa) e um paracetamol `refused` com motivo;
  - "Benedito Alves Rocha" (72, pneumonia, aguardando vaga): no "OBS 02" há 25 h (`over_limit`), com troca de leito do "OBS 03" no meio; prescrição v2 (a v1 substituída): AMOXICILINA 500 mg VO a cada 8 h e SALBUTAMOL inalatório a cada 6 h; uma checagem de salbutamol administrada → falta `insufficient_balance` na "Faltas da UPA";
  - o "José Carlos Pereira" de ontem (32a) segue sem observação.
- CPF e CNS fictícios válidos (`cpf_for`, `Professionals::Cns.generate`), CID da consulta compatível com a SIGTAP de dev (I10 na Maria) — a semente não finaliza a consulta, então nada de BPA nasce dela.
- Resumo: os e-mails, papéis e otpauth novos, "leitos: 4", "faltas abertas: 1"; nunca nome de paciente nem medicamento.

- [ ] **Step 1: Write the failing test** `spec/lib/urgency_crew_spec.rb`: rodar duas vezes não duplica leitos, estadias, prescrições, administrações, movimentos nem faltas; o mapa tem 2 leitos ocupados (`near_limit` e `over_limit`) e 2 livres; a farmacêutica vê 1 falta aberta; o resumo não contém nenhum nome de paciente nem "LOSARTANA"/"DIAZEPAM".
- [ ] **Step 2:** `bin/test api -w mod32b spec/lib/urgency_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/lib/pharmacy_crew_spec.rb spec/lib/professional_crew_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: seed upa observation beds, prescriptions, checks and a stock shortfall`.

**Relato ao coordenador:** "api 32b · Task 18 · semente: 4 leitos, 2 em observação (near/over_limit), prescrições com controlado, baixa e 1 falta; logins novos: <lista>."

---

### Task 19: Suíte completa

- [ ] **Step 1:** pedir à sessão "Testes" por `SendMessage`: `api · mod32b/feat/mod-32b-urgency-observation · --full · módulo 32b (stock_movements e Pharmacy::Ledger do 27a, fechamento do encontro do 32a, BPA do 22, lista de interruptores, spec/support)`.
- [ ] **Step 2:** falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador ("Próximos módulos") com o resumo e o hash, e a sessão do dashboard de que o api está pronto (as formas e os códigos de erro do Mapa de interfaces abaixo).
- [ ] **Step 3:** merge, push e cards só com autorização do usuário; F-32.7/F-32.8 seguem In Progress até o último app de cada card.

**Relato ao coordenador:** "api 32b · Task 19 · suíte completa <verde N/0 | falhas>; hash <…>; pronto para o dashboard."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptor (Task 1), papéis (1, 5–11), tabelas (2–4), tabelas existentes (4, 1); §2 detalhe (16); §3 leitos (5); §4 mapa, alertas, entrada/troca, saídas (6, 12), fila (6); §5 prescrição (7), fim (12), aprazamento (8); §6 checagem (9), estoque (10), desligado (9); §7 faltas e alerta (11); §8 BPA (13); §9 linha do tempo (14), painel (15); §10 eventos (6, 7, 9–12; guarda 17); §11 cifra (3, 4), sigilo (17), imutabilidade (2–4), exclusão (nada muda — o encontro com consulta já é retido no 32a); §12 P1 (7, 10), P2 (13).
2. **Spec §6 "Testes" do 32b:** leito sem sobreposição (2, 6), alertas (6), prescrição em versões (3, 7), checagem (9), estoque ligado e desligado (9, 10, 11), sem saldo (10).
3. **Nomes:** `Urgency::{SaveBed,StartObservation,EndObservation,IssuePrescription,SuspendPrescription,RecordAdministration,DeductStock}`, `Urgency::{Observation,Schedule,StockGate,PrescriptionJson}`, `Pharmacy::ReconcileUrgencyShortfall` — usados com a mesma assinatura nas tasks seguintes.
4. **Ordem das migrações** segue a das tasks (900001..900003); a 900002 não depende da 900001; a 900003 depende da 900002 (FK para `urgency_administrations`).

## Divergências propostas ao contrato

A1. **`waiting_bed` sem definição.** O §4 põe `waiting_bed: []` no mapa sem dizer o que é. Não existe "indicação de observação" sem leito. Proposta: os encontros `in_care` da UPA sem estadia aberta, `{ encounter_id, ticket, patient: <patient-ref>, called_at }`, por `called_at` — a lista de onde o mapa tira quem pôr num leito. Evidência: `app/services/urgency/queue.rb:30` (o grupo `in_care` já existe na fila).

A2. **Rótulo repetido.** O §3 pede 422 `label_taken`; o CAPS responde 409 `bed_label_taken` (`app/commands/psychosocial/save_bed.rb:42-45`). O plano segue o contrato (422 `label_taken`); o dashboard não pode reaproveitar o texto do CAPS. Proposta: manter, e registrar a diferença.

A3. **`upa_has_beds` é 409 e vale em dois caminhos.** O `caps_has_beds` é 409 e só na troca de modalidade (`app/controllers/health_units_controller.rb:72,241`); a desativação também devolve conflito em 409 (`:96`, `#deactivation_conflict`). Proposta: 409 `upa_has_beds` ao desativar a UPA com leito ativo **e** ao trocar o `kind` dela, no padrão do controlador.

A4. **Parâmetro da lista de faltas.** O §7 usa `stock_location_id`; todo o 27a lê o local por `location_id` (`app/controllers/concerns/location_reading.rb:31`, `/pharmacy/alerts`). Proposta: `GET /pharmacy/urgency_shortfalls?location_id&state`.

A5. **Falta `no_location` sem leitor.** A falta sem local não pertence a farmácia nenhuma, então "farmacêutico e almoxarife do local" nunca a veria. Proposta: o almoxarife do central (`Pharmacy::Access.central_keeper?`, `app/services/pharmacy/access.rb:35`), que já lê todos os locais, lê e concilia as faltas sem local, pela lista sem `location_id`. O `/pharmacy/alerts` conta só as do local.

A6. **Quantidades em String.** O §1 e o §6 tratam `dose`, `stock_quantity_per_dose` e `stock_quantity` como numéricos. O 27a só aceita quantidade como String decimal com ponto e recusa número JSON (`app/services/pharmacy/quantity.rb:1-15`), para nunca arredondar. Proposta: as três como String; `dose` com até 3 casas (`DOSE`), as do estoque no `Pharmacy::Catalog::QUANTITY`.

A7. **Unidade da quantidade.** O §1 diz "na `dispensing_unit` do item" e o §6, "na `dispensing_unit` do lote". A unidade de dispensação mora no lote (`stock_lots.dispensing_unit`) e, fora dele, sai do `Pharmacy::DispensingUnit.for` (`app/services/pharmacy/dispensing_unit.rb:1-6`). Proposta: o item grava `stock_unit` = `DispensingUnit.for(item)` na prescrição; o FEFO só considera lotes com a mesma unidade, e o lote do corpo com outra unidade → 422 `lot_invalid`.

A8. **`controlled_list` no item.** O P1 manda o item guardar `controlled_list`, mas a lista de colunas do §1 não a tem. Proposta: `urgency_prescription_items.controlled_list` string(3), cópia de `medication_catalog_items.controlled_list` (`db/platform_schema.rb:328`).

A9. **Quem suspendeu.** A prescrição tem `status_changed_at`, mas não quem mudou o status. Proposta: `status_changed_by_user_id`, preenchido na troca de versão, na suspensão e no fim pelo desfecho.

A10. **FEFO sem fracionar.** O §6 não diz se uma dose pode sair de dois lotes. O plano grava um movimento de um lote só: se nenhum lote cobre a quantidade sozinho, nasce `insufficient_balance`, mesmo que a soma cubra. Proposta: aceitar no 32b; o fracionamento fica como pendência de ciclo se a farmácia pedir.

A11. **Quando é `lot_unavailable`.** O §6 cita a causa sem dizer quando. Proposta: o lote do corpo com saldo positivo, mas menor que a quantidade, e o `Pharmacy::Ledger::Insufficient` levantado pela rede do trigger (`app/services/pharmacy/ledger.rb:93-96`). O lote do corpo com saldo 0 continua sendo 422 `lot_invalid`.

A12. **Lote bloqueado.** O 29 bloqueia lote no local por excursão de temperatura (`StockLotBlock.open_blocks`, `app/commands/vaccination/record_immunization.rb:176`). Proposta: o FEFO ignora o lote bloqueado no local, e o lote bloqueado vindo do corpo → 422 `lot_invalid`.

A13. **Checagem antecipada.** "Um dos horários do aprazamento" deixa checar um horário até 24 h à frente. O plano segue a letra. Proposta: limitar a antecipação, por exemplo a 2 h, com 422 `slot_too_early`; isso é decisão clínica.

A14. **Janela de `first_dose_at`.** O contrato não a limita. Proposta: de `now − 1 h` a `now + 24 h`, truncado ao minuto; fora dela → 422 `invalid_request` `items.N.first_dose_at`.

A15. **`administered_at`.** O contrato não traz as regras. Proposta: só em `administered`; padrão `now`; no máximo `now + 5 min`; nunca antes do `prescribed_at` da versão.

A16. **O que conta no `max_per_day`.** Proposta: só as `administered` nas 24 h antes do novo `administered_at`; `refused` e `not_administered` não contam.

A17. **O técnico não lê o detalhe.** O Q19 do 32a dá o detalhe clínico só a 2235\*/225\* (`app/services/urgency/access.rb:34`). O técnico, que agora checa, lê o aprazamento (com trilha) e o mapa, mas não o detalhe. Proposta: manter; o painel do enfermeiro, no dashboard, usa o aprazamento.

A18. **Detalhe sem acesso clínico.** Para a recepção, o bloco `observation` sai (leito e horas, sem texto clínico); `prescription` e `administrations` não saem (as chaves somem, no padrão do Q19, `app/services/urgency/json.rb:52-56`).

A19. **Trilha do aprazamento.** O §5 manda gravar trilha, mas o recurso não está em `Urgency::Trail::RESOURCES` (`app/services/urgency/trail.rb:9`). Proposta: um recurso novo, `medication_schedule`.

A20. **`observation_ended` na saída com desfecho.** O §10 só cita a troca de leito. Proposta: o desfecho que fecha uma estadia também publica `urgency.observation_ended`, para a trilha de toda estadia ter começo e fim.

A21. **Fechamento pelo 13 com a urgência desligada.** Com `urgency_care` desligado, `Attendances::Close` fecha o atendimento sem o desfecho do 32 (Q10 do 32a), e as rotas `/urgency` respondem 403. Uma estadia aberta ficaria aberta, com o leito ocupado para sempre. Proposta: nesse caso, o `Attendances::Close` fecha a estadia com `observation_ended`. A Task 12 só aplica isso com o aceite do coordenador.

A22. **`observation_hours` no PEC.** A página do PEC é HTML e só lê o resumo (`app/services/pec_external/page.rb:46-50`); a chave JSON não chega lá. Proposta: a chave `observation_hours` só nas entradas `urgency` do JSON profissional, e o texto "observação N h" no resumo, que aparece no PEC. As chaves das outras fontes continuam fixas (`spec/requests/timeline_spec.rb:25`).

A23. **Extrato do 27a.** O movimento `administration` aparece no extrato com um documento novo, que o contrato não cita (`app/services/pharmacy/statement.rb:10,55-66`). Proposta: `document: { type: "urgency_administration", id, number: null }`; o dashboard precisa do rótulo, por exemplo "Medicação na UPA".

A24. **Chaves do `/pharmacy/alerts`.** A 6ª chave, `urgency_shortfalls`, muda as listas exatas de `spec/requests/pharmacy/positions_spec.rb:515,517,534,557`. O dashboard do 27a precisa aceitar a chave nova.

A25. **Horas depois de sair e voltar.** O §4 conta desde a primeira entrada. Assim, quem sai da observação e volta horas depois tem a pausa contada. O plano segue a letra, porque o `0301060029` é observação até 24 h na permanência inteira. Proposta: confirmar.

A26. **BPA só com a consulta finalizada; CID da amostra.** A observação substitui a linha do atendimento médico, que só nasce com a consulta finalizada (`app/commands/billing/from_urgency.rb:80-84`). Assim, a evasão durante a observação, sem consulta finalizada, não fatura a observação. Além disso, a amostra SIGTAP de dev restringe os CIDs do `0301060029` a I10, E119 e J069 (`lib/sigtap_sample.rb:145`): em dev, outro CID faz a crítica bloquear a linha. Proposta: registrar os dois pontos no gate api#136.

A27. **"A lista exata" são duas.** Os nomes dos eventos estão fixados no registro (`spec/initializers/domain_events_bindings_spec.rb:582`, regex `(urgency|waiting_room)\.`) e na lista do cenário do 32a (`spec/security/urgency_secrecy_spec.rb:211`). Proposta: o registro passa a 15 nomes, com o regex ampliado para `pharmacy.urgency_shortfall_reconciled`; o `urgency_secrecy_spec` ganha um cenário do 32b com a lista exata dos sete. O cenário do 32a continua com a dele.

A28. **Administrações no detalhe.** Proposta: as das últimas 48 h de qualquer versão, porque a enfermagem precisa ver o que foi dado antes de uma alteração.

A29. **Recepção no mapa.** A recepção (`citizen_verifier`) não tem vínculo de unidade e lê o mapa de qualquer UPA, como a fila (Q26 do 32a, `app/controllers/urgency/queue_controller.rb:17`). Proposta: manter; o filtro por unidade entra no mesmo card de pendência do Q26.

A30. **Unidades e supressão do painel.** Proposta:
- `median_hours`, `p90_hours` e `bed_hours` com uma casa decimal;
- `stays` e `over_24h` de 1 a 4 → `"lt5"`;
- as três medidas seguem a supressão de `stays`: `"lt5"` com 1 a 4 e `null` com 0.

A31. **Códigos HTTP de prescrição e observação.**
- `prescription_not_active` é 422 também na suspensão.
- `encounter_closed` é 409 na observação, na prescrição e na suspensão.
- `consultation_required` é 422.
- `catalog_item_invalid` usa o path `items.N.catalog_item_id`.
- O mesmo leito da estadia aberta → 422 `invalid_request` `bed_id`.

A32. **Domínio de `dose_unit`.** O contrato não o define. Proposta: a lista fechada `DOSE_UNITS` do Valor fixado 2; o dashboard usa a mesma lista.
