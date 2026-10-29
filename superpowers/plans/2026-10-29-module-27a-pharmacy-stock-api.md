# Módulo 27a — Estoque e dispensação (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 26a em `origin/main` do api** (conferido ao escrever este plano em `ded8a0eb`: `Platform::Features::CATALOG` terminando em `care_regulation`; última migração de cidade `20261028900001_add_care_regulation.rb`; `db/city_schema.rb` em `define(version: 2026_10_28_900001)`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..44)`; `Membership::ROLES` com `care_regulator` e `ck_memberships_role` reescrito por `AddCareRegulation::ROLES` (restaurado em `spec/models/create_analytics_migration_spec.rb:33` e `create_campaigns_migration_spec.rb:64`, conferido em `spec/models/membership_roles_spec.rb:65`); `Membership::PRIVILEGED_ROLES` terminando em `care_regulator`; `Timeline::KINDS` terminando em `procedure` e `Timeline::Query.citizen_kinds` sem exclusão própria; `ClinicalDocuments::Json.medication(medication, catalog:)`; `ClinicalDocuments::Cancel` travando o documento com `lock`; conteúdo da receita com `valid_until` (nulo na comum sem prazo escolhido) e `category` (`common`/`antimicrobial`/`special_control`) e `sncr` (`{ kind, number, simulated }`); `MedicationCatalogItem` com `controlled`/`controlled_list` (CHECK `controlled = (controlled_list IS NOT NULL)`) e `unit` **da concentração** (`mg`, não da dispensação; Divergência P1); `HealthUnit.lock_active!`; `Patients::CreateTyped.call(attrs:, by:)`; `Patients::Resolve.call(citizen)`; `Patients::Pairs.of_cpf`; `Patients::IdentifierJson.call(person)`; `Citizens::RequestErasure.examined_patients?` terminando em `regulated?(ids)`; `RegulationCrew` como último crew em `db/seeds.rb`; `Lab::Inventory::RecordMovement` com trava do lote + trigger `rota_lab_supply_balance_guard`). **A Task 0 confere a base: se um nome mudou, PARE e reporte.**

**Goal:** O lado api do módulo 27a (F-27.1 a F-27.7, ADR 0045): interruptor `pharmacy`; papéis `pharmacist`, `pharmacy_assistant` e `stock_keeper` vinculados a locais de estoque; almoxarifado central e farmácias de unidade (`health_units.dispenses`); lotes CATMAT com validade; movimentos imutáveis com saldo derivado e nunca negativo; entradas com conversão de embalagem e inventário de abertura; transferência em dois passos com "em trânsito" e diferença; ajustes com step-up e segundo responsável acima de 10%; dispensação pela receita do 19c e pela receita externa, controlados só pelo farmacêutico com retenção; linha do tempo e última retirada na lista de medicamentos; retenção LGPD; semente de dev em Curitiba; runbook.

**Architecture:** O saldo nunca é coluna: é `SUM(stock_movements.quantity)` por (local, lote), lido sob `FOR UPDATE` nos lotes envolvidos (ordenados por id), e um trigger `AFTER INSERT` recusa saldo negativo (cinto de segurança, como o 21c). Toda escrita de estoque passa por `Pharmacy::Ledger` com a ordem de travas **lotes → local → documento** (receita, receita externa, transferência ou ajuste). A dispensação lê a receita do 19c (`clinical_documents` + `prescription_items`) ou a externa por um envelope único `Pharmacy::Prescription`, e grava `dispensations`/`dispensation_items` e um movimento `dispensing` por item na mesma transação.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec via `bin/test`. Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md` e `docs/adr/0045.md`. Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-29-module-27a-pharmacy-stock-contracts.md` (convenções dos contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12, 24a §14, 25 §15 e 26a §13); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **P**). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-6-mercado.md`; ADRs 0033, 0034, 0038 (padrão de movimentos do laboratório — o padrão, não o código). Lições que valem aqui: contadores sem trilha; ordem de travas estável; recusar em vez de cortar (texto longo, quantidade com 3 casas → 422 com `path`); quantidade como string decimal (vírgula só na tela); nada pessoal em `inspect`/`to_s`/`pretty_print`/`as_json`. "Todas as recusas HL7 de uma vez" e "estrutura HL7 sempre AR" não se aplicam (não há HL7): o validador devolve a **primeira** recusa com `path`, como o 22, o 24a, o 25 e o 26a.

## Desvios e precisões

1. **Unidade de dispensação.** `medication_catalog_items.unit` é a unidade da **concentração** (`mg`, `mg/ml`), não a de dispensação. A unidade de dispensação sai de `dosage_form` por `config/pharmacy/dispensing_units.yml` (palavra-chave → unidade; sem casamento → `"unidade"`) e é **gravada no lote** (`stock_lots.dispensing_unit`) na primeira entrada: mudar a regra depois não muda o sentido de saldo antigo (P1).
2. **Quem faz o quê.** Papel = `Membership` ativa **e** vínculo ativo em `stock_location_staff` do local, com o mesmo papel (`Pharmacy::Access`). Entrada (nota/doação): `stock_keeper` do local. Abertura: `stock_keeper` ou `pharmacist` do local. Transferência: `stock_keeper` do central envia; `stock_keeper`, `pharmacist` ou `pharmacy_assistant` do destino recebe. Ajuste/perda/vencimento: `stock_keeper` ou `pharmacist` do local, com step-up. Aprovação: outro `stock_keeper` ou `pharmacist` do mesmo local, com step-up. Dispensação e receita externa: `pharmacist` ou `pharmacy_assistant` de local `unit`. Posição, extrato e alertas: qualquer papel do local; `stock_keeper` do central lê todos os locais (P6, P18).
3. **Vínculo do farmacêutico.** Vincular `pharmacist` exige `professional_links` ativo com `cbo_code` começando por `2234` (qualquer unidade), senão 422 `cbo_not_allowed`; vincular qualquer papel exige a `Membership` ativa do mesmo papel, senão 422 `role_not_granted` (P18).
4. **Entrada com embalagem.** Item com `package: { units_per_package, packages }` grava `quantity = units_per_package × packages` (inteiros 1..10000); com `quantity` e `package` juntos, os dois precisam bater (senão `invalid_quantity` `path: "items[i].quantity"`). O movimento guarda `package_units` e `packages`. Lote: `(catalog_item_id, lot_number)` único na cidade; o mesmo número com validade ou fabricante diferente → 422 `lot_mismatch` `path: "items[i].lot_number"`. Lote vencido na entrada → `expired_lot`.
5. **Abertura.** Uma por (local, item): `stock_openings` único; repetir → 409 `opening_exists` `path: "items[i].catalog_item_id"`. Várias linhas do mesmo item (lotes diferentes) na mesma abertura valem.
6. **Transferência.** Só do central para local `unit` ativo. Enviar grava um `transfer_out` por lote no central. "Em trânsito" = itens de transferência `sent` (derivado, sem movimento). Receber exige **todos** os lotes da transferência, uma vez cada (`invalid_items`), `quantity_received` de `0` até o enviado (mais que o enviado → `invalid_quantity`); grava `transfer_in` do **enviado** no destino e, havendo diferença, `loss` da diferença no destino com o motivo (10..500, obrigatório; `reason_required` com `path`). O extrato mostra as duas linhas (P7). Receber duas vezes → 409 `already_received`.
7. **Ajustes.** `adjustment` recebe `counted_quantity` (pode ser `"0"`): `delta = contado − saldo`; `delta = 0` → 422 `no_difference`; `|delta| > percent% × saldo` (saldo zero conta como acima) → `pending`, sem movimento; senão `applied` com movimento. `loss` e `expiry` recebem `quantity` positiva, nunca vão a aprovação (P6); `expiry` só com `expires_on < hoje` (`lot_not_expired`). Aprovar aplica o **delta guardado** sob a trava do lote; saldo que ficaria negativo → 422 `insufficient_balance` e o ajuste continua `pending`. Rejeitar exige motivo.
8. **Validade da receita (fuso da cidade, dia inclusivo).** Por item: `valid_until` do conteúdo quando existe; senão `issued_at` (data na cidade) + `continuous_validity_days` (item contínuo) ou `common_validity_days`. Receita externa: `issued_on` + os mesmos prazos, e **30 dias fixos** para item controlado (Portaria 344). A receita mostra `valid_until` = o maior dos itens e `expired` = todos vencidos (P4). Cancelada (`status = cancelled`) → `prescription_cancelled`, antes de qualquer outra regra.
9. **Uso contínuo.** Item contínuo: a próxima retirada do **mesmo item da receita** só a partir de `última retirada + continuous_interval_days` (`too_early` com `next_allowed_on`) (P5). Item não contínuo: retiradas parciais até alguém marcar `complete: true`; depois → `item_completed`. Nenhuma conversão de quantidade (decisão do coordenador): o total dispensado por item é mostrado, não limitado.
10. **Texto livre e controlados.** Item da receita em texto livre: só `pharmacist`, que manda `catalog_item_id` (`free_text_requires_pharmacist` 403; sem item → `invalid_item` `path`); a escolha fica em `mapped_from_free_text`. Item cujo catálogo é `controlled` (inclusive o escolhido para texto livre): só `pharmacist` (`controlled_requires_pharmacist` 403) e `retention` obrigatório (`retention_required`). O número retido: receita digital → o `sncr.number` do conteúdo (o corpo não o substitui; divergente → `invalid_request` `path: "retention.notification_number"`); externa → o `notification_number` da receita externa (P8). Antimicrobial não é controlado aqui (P8).
11. **Mesmo lote em duas linhas.** As quantidades por lote são **somadas** antes de conferir o saldo (Review Focus 3).
12. **Devolução** (`return`): `POST /pharmacy/returns` — o paciente devolve parte de um item dispensado em condições de uso; `pharmacist` ou `pharmacy_assistant` do local da dispensação; até o dispensado menos o já devolvido (`return_exceeds`); lote vencido → `lot_expired`; motivo 10..500 (P3).
13. **Trilha.** Busca da receita (por código ou paciente), receita externa criada, histórico do paciente e busca de pessoa: um `clinical_record.viewed` por pessoa mostrada (`access: "pharmacy"`, `patient_id`, `person_ref`, `user_id`). Posição, extrato, alertas, transferências, ajustes e contadores: sem trilha e sem dado pessoal. Dispensação gravada: sem trilha nova (a busca já deixou).
14. **Interruptor.** `pharmacy` sem `requires`. Desligado: todas as rotas `/pharmacy/*` → 403 `feature_disabled`; a linha do tempo e a `last_dispensed_on` continuam mostrando o que já existe (registro clínico retido; P10).

## Valores fixados por este plano

1. **Interruptor** (depois de `care_regulation` no `CATALOG`): `Entry.new(key: "pharmacy", description: "Farmácia e almoxarifado: estoque por local com lotes e validade, transferências, inventário e dispensação pela receita", requires: [])`.
2. **`Pharmacy::Catalog`:** `LOCATION_KINDS = %w[central unit]`, `ROLES = %w[pharmacist pharmacy_assistant stock_keeper]`, `DISPENSER_ROLES = %w[pharmacist pharmacy_assistant]`, `MOVEMENT_KINDS = %w[opening receipt donation transfer_out transfer_in dispensing loss expiry adjustment return]`, `POSITIVE_KINDS = %w[opening receipt donation transfer_in return]`, `NEGATIVE_KINDS = %w[transfer_out dispensing loss expiry]`, `ENTRY_KINDS = %w[invoice donation opening]`, `ENTRY_MOVEMENT = { "invoice" => "receipt", "donation" => "donation", "opening" => "opening" }`, `TRANSFER_STATUSES = %w[sent received received_with_difference]`, `ADJUSTMENT_KINDS = %w[adjustment loss expiry]`, `ADJUSTMENT_STATUSES = %w[applied pending approved rejected]`, `PRESCRIPTION_KINDS = %w[digital external]`, `PHARMACIST_CBO_PREFIX = "2234"`, `COUNCILS = %w[CRM CRO COREN]`, `NOTIFICATION_LISTS = %w[A1 A2 A3 B1 B2]`, `CONTROLLED_VALIDITY_DAYS = 30`, `DEFAULT_SETTINGS = { "common_validity_days" => 30, "continuous_validity_days" => 180, "continuous_interval_days" => 25, "expiry_alert_days" => 90, "adjustment_approval_percent" => 10 }`, `SETTING_RANGES = { "common_validity_days" => 1..365, "continuous_validity_days" => 1..365, "continuous_interval_days" => 1..90, "expiry_alert_days" => 1..365, "adjustment_approval_percent" => 0..100 }`, `QUANTITY = /\A\d{1,5}(\.\d{1,2})?\z/`, `REASON = 10..500`, `NAME_MAX = 120`, `LOT_NUMBER = /\A[0-9A-Za-z.\/-]{1,40}\z/`, `DOCUMENT_NUMBER = /\A[0-9A-Za-z.\/-]{1,60}\z/`, `SUPPLIER_MAX = 120`, `MANUFACTURER_MAX = 120`, `NOTIFICATION_NUMBER = /\A[0-9A-Za-z.\/-]{1,30}\z/`, `COUNCIL_NUMBER = /\A[0-9]{1,10}\z/`, `UF = /\A[A-Z]{2}\z/`, `PACKAGE_RANGE = 1..10_000`, `MAX_ENTRY_ITEMS = 100`, `MAX_TRANSFER_ITEMS = 100`, `MAX_DISPENSE_ITEMS = 20`, `MAX_EXTERNAL_ITEMS = 20`, `DOSAGE = 3..500`, `QUANTITY_UNIT_MAX = 30`, `LOOKUP_DAYS = 365`, `LOOKUP_MAX = 20`, `TRAIL_ACCESS = "pharmacy"`, `DEFAULT_LIMIT = 50`, `MAX_LIMIT = 100`.
3. **Quantidade:** entrada só `String` casando `QUANTITY` (número JSON, vírgula, 3 casas, sinal → `invalid_quantity` com `path`); positiva, exceto `counted_quantity` e `quantity_received`, que aceitam `"0"`. Saída: `Pharmacy::Quantity.dump` sempre com duas casas e ponto (`"30.00"`, `"2.50"`) (P13). Colunas `numeric(12,2)`.
4. **Listas:** `limit` padrão 50, máximo 100 (`invalid_limit`); posições em ordem `(stock_location_id, catalog_item_id)` com `next_before` = `Pharmacy::Cursor.encode([location_id, catalog_item_id])` (base64 url-safe; `invalid_cursor`); transferências, ajustes, extrato e dispensações por `(created_at DESC, id DESC)` com o mesmo cursor de `[iso8601(6), id]`.
5. **Dev:** api do worktree na porta **3049**; worktree `apps/api/.claude/mod27a` (com `config/master.key`); branch `feat/mod-27a-pharmacy-stock`; migração de cidade `20261029900001`. Semente (Task 15): central "Central de Abastecimento Farmacêutico de Curitiba"; farmácia da "UBS Jardim das Flores" (`CareLinesCrew::UNIT`); farmacêutica "Ana Paula Wisniewski" (`farmacia@<slug>.demo`, `pharmacist`, CBO `223405`, OTP fixo por `DEV_PHARMACIST_OTP_SECRET` com padrão `FARMACEUTICASEMENTEDEVCWBAPHARMA`); auxiliar "Marcos Vinícius Andrade" (`farmacia.auxiliar@`, `pharmacy_assistant`, `DEV_PHARMACY_ASSISTANT_OTP_SECRET`/`AUXILIARFARMACIASEMENTEDEVCWBAPH`); almoxarife "Luiz Fernando Kuster" (`almoxarifado@`, `stock_keeper` no central e na UBS, `DEV_STOCK_KEEPER_OTP_SECRET`/`ALMOXARIFESEMENTEDEVCWBAPHARMACY`); segundo almoxarife "Cristiane Bortolini" (`almoxarifado.2@`, `stock_keeper` no central, para aprovar ajuste).

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A`; commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-27.1..F-27.7 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada, começando com `gh`). Um card só vai para Done quando o ÚLTIMO app dele terminar (todos têm dashboard; F-27.1 também maintenance); Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod27a <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host; sem bancos com sufixo; a suíte completa é pedida à sessão "Testes" (Task 16).
- Banco de cidade: `db/city_migrate/20261029900001_add_pharmacy_stock.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_29_900001)`); triggers em `db/city_triggers.sql`, executado no fim do `up` (o dump não representa trigger nem linha: `PharmacySetting.ensure_defaults!` nos helpers de teste). Irreversível. Sem migração de plataforma. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error": "<reason>" }` (com `path` quando o contrato ou este plano manda; `next_allowed_on` em `too_early`); interruptor 403 `{ "error": "feature_disabled", "feature": "pharmacy" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (o `Authentication` já recusa com 415); registro de outro escopo por id → 404; lista com escopo alheio → 403.
- **CPF, CNS, nome, medicamento do paciente, número de notificação, receita (`short_code`), motivos e posologia nunca em log, evento, Analytics, erro ou URL.** Só no corpo JSON. `patient_id` em query só em `GET /pharmacy/dispensations` (contrato §7; é id). Filtros de log novos: `/\A(prescriber|retention|notification_number|counted_quantity|quantity_received)\z/` (`:short_code`, `:items`, `:reason`, `:new_patient`, `:cpf`, `:cns`, `:dosage_instructions` já estão).
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids (nunca item, lote com paciente, quantidade, motivo); `DomainEvents.publish(` é multilinha — confira cada chamada inteira (memória "Grep de linha única engana"). Nenhum `Platform.audit` novo (nada a declarar em `R18_PLATFORM_EVENT_NAMES`; a Task 16 confere).
- Nunca `Current.city =` solto: leia `Current.city`; em seed, `CityConnection.with`. Catálogo da plataforma (`MedicationCatalogItem`) e interruptores lidos **fora** da transação da cidade (como `ClinicalDocuments::Issue`).
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); colunas cifradas de tabela só-acréscimo no `rota_reencryption_only(...)` ou sob a marca `rota.reencrypting` nos guardas próprios.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..45)`.
- Papéis novos: `ck_memberships_role` reescrito pela migração com `AddPharmacyStock::ROLES`; os dois specs de migração que restauram a lista à mão e o `membership_roles_spec` passam a usar a constante nova.

## Review Focus

1. **Validade na virada do dia no fuso da cidade** (receita comum sem prazo emitida em 30/09 às 23:30 em São Paulo — já 01/10 em UTC): vale até 30/10 inclusive; dispensar em 30/10 às 23:50 passa, em 31/10 às 00:10 é `prescription_expired`. Teste na Task 9 (`travel_to` + `Time.use_zone`).
2. **Receita cancelada entre a busca e a dispensação** (a médica cancela com o farmacêutico na tela): 422 `prescription_cancelled`, nenhum movimento, saldo intacto; e o `Cancel` concorrente espera a trava do documento. Teste na Task 11.
3. **O mesmo lote em duas linhas** da mesma dispensação, cada uma cabendo no saldo e a soma não: 422 `insufficient_balance` com `path` da segunda linha, nada gravado. Teste na Task 11.
4. **Papel revogado com vínculo ao local ainda ativo** (o admin revogou a `Membership` `pharmacist`, o `stock_location_staff` ficou): 403 `missing_role` na dispensação; e farmacêutico sem CBO `2234*` não se vincula. Teste na Task 4.
5. **Quantidade digitada como no papel** (`"1,5"`, número JSON `1.5`, `"0.001"`, `"-2"`, `"100000"`): 422 `invalid_quantity` com o `path` do item, nunca arredondada. Teste na Task 5 (entrada) e na Task 11 (dispensação).

## Mapa de arquivos

**Criar** (todos sob `apps/api/.claude/mod27a/`)
- `db/city_migrate/20261029900001_add_pharmacy_stock.rb`; `config/pharmacy/dispensing_units.yml`.
- `app/models/{stock_location,stock_location_staff,pharmacy_setting,stock_minimum,stock_lot,stock_entry,stock_opening,stock_movement,stock_transfer,stock_transfer_item,stock_adjustment,external_prescription,external_prescription_item,dispensation,dispensation_item}.rb`.
- `app/services/pharmacy/{gate,catalog,quantity,cursor,dispensing_unit,access,ledger,settings,json,trail,prescription,validity,dispensed,positions,alerts,statement}.rb`.
- `app/commands/pharmacy/{save_location,link_staff,save_settings,save_minimums,record_entry,send_transfer,receive_transfer,request_adjustment,decide_adjustment,record_external_prescription,dispense,record_return}.rb`.
- `app/controllers/concerns/pharmacy_gate.rb`; `app/controllers/pharmacy/{base,locations,items,entries,positions,statements,transfers,adjustments,person_lookups,prescriptions,external_prescriptions,dispensations,returns,alerts,minimums}_controller.rb`; `app/controllers/pharmacy/admin/{locations,staff,settings}_controller.rb`.
- `app/services/timeline/sources/dispensation_source.rb`.
- `lib/pharmacy_crew.rb`.
- Specs: `spec/services/pharmacy/{gate,quantity,dispensing_unit,access,ledger,validity,positions}_spec.rb`, `spec/models/pharmacy_schema_spec.rb`, `spec/commands/pharmacy/{record_entry,transfers,adjustments,dispense,stock_concurrency}_spec.rb`, `spec/requests/pharmacy/{admin,locations,entries,transfers,adjustments,positions,lookup,external_prescriptions,dispensations,returns}_spec.rb`, `spec/services/timeline/sources/dispensation_source_spec.rb`, `spec/commands/citizens/pharmacy_erasure_spec.rb`, `spec/lib/pharmacy_crew_spec.rb`, `spec/support/pharmacy_helpers.rb`.

**Modificar**
- `app/services/platform/features.rb` (CATALOG), `spec/adr_pointers_spec.rb` (1..45), `config/initializers/filter_parameter_logging.rb`.
- `app/models/membership.rb` (ROLES, PRIVILEGED_ROLES), `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`.
- `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `app/models/health_unit.rb` (comentário de `dispenses`).
- `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`.
- `app/services/timeline.rb` (KINDS), `app/services/timeline/query.rb` (SOURCES, `PROFESSIONAL_ONLY_KINDS`), `spec/services/timeline/query_spec.rb`.
- `app/services/clinical_documents/json.rb` (`medication(…, last_dispensed:)`), `app/controllers/patient_medications_controller.rb`, `spec/requests/patient_medications_spec.rb`.
- `app/commands/citizens/request_erasure.rb`.
- `db/seeds.rb` (PharmacyCrew depois do RegulationCrew).

---

### Task 0: Base, cards e worktree

**Files:** nenhum arquivo do app (conferência).

- [ ] **Step 1: Mover os cards F-27.1..F-27.7 do board #1 para In Progress** (um `gh project item-edit` por card, comando começando com `gh`; ids dos campos na memória "board-91-features-roadmap").
- [ ] **Step 2: Criar o worktree** — `/opt/homebrew/bin/git -C apps/api fetch origin` e `/opt/homebrew/bin/git -C apps/api worktree add .claude/mod27a -b feat/mod-27a-pharmacy-stock origin/main`; copiar `apps/api/config/master.key` para `apps/api/.claude/mod27a/config/master.key`.
- [ ] **Step 3: Conferir a base** — cada nome do PRÉ-REQUISITO em `origin/main` (`git show origin/main:<arquivo>` / `git grep`); conferir também que o `ControlledPrescriptionsCrew` liga `controlled_prescriptions` em Curitiba e que FLUOXETINA (`273009`) fica `controlled_list: "C1"` e DIAZEPAM (`267197`) `"B1"` depois do `AnvisaImport` (a semente da Task 15 depende disso; divergiu → anote na Task 15). Nome divergente do PRÉ-REQUISITO → PARE e reporte.
- [ ] **Step 4: Rodar a base** — `bin/test api -w mod27a spec/adr_pointers_spec.rb spec/models/membership_roles_spec.rb` → PASS (base verde antes de mexer).

---

### Task 1: Interruptor `pharmacy`, catálogo de constantes e base das rotas

**Files:**
- Create: `app/services/pharmacy/{gate,catalog,quantity,cursor}.rb`, `app/controllers/concerns/pharmacy_gate.rb`, `app/controllers/pharmacy/base_controller.rb`
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/services/pharmacy/{gate,quantity}_spec.rb`

**Interfaces:**
- Produces: `Pharmacy::Gate::KEY = "pharmacy"`; `Gate.usable?(city) -> bool`. Concern `PharmacyGate#require_pharmacy!` (403 `feature_disabled`, `feature: "pharmacy"`). `Pharmacy::Catalog` (Valores fixados 2). `Pharmacy::Quantity.parse(raw, allow_zero: false) -> BigDecimal | nil` e `Quantity.dump(BigDecimal) -> String`. `Pharmacy::Cursor.encode(Array) -> String` / `decode(String) -> Array | nil`.
- `Pharmacy::BaseController < ApplicationController` com `Authentication`, `MfaStepUp`, `PharmacyGate`, `wrap_parameters false`, `before_action :require_pharmacy!` e os auxiliares do `CareRegulation::BaseController` (`render_failure` levando `path` e `next_allowed_on` quando o resultado traz, `render_error`, `forbid`, `not_found`, `uuid?`, `body_param`, `body`, `query_only!`, `page_limit`), com `FAILURE_STATUS = { already_received: :conflict, opening_exists: :conflict, location_has_stock: :conflict, location_inactive: :conflict, not_pending: :conflict, central_exists: :conflict, unit_taken: :conflict, same_user: :forbidden, missing_role: :forbidden, controlled_requires_pharmacist: :forbidden, free_text_requires_pharmacist: :forbidden, not_found: :not_found, prescription_not_found: :not_found, catalog_unavailable: :service_unavailable }` (o resto 422); `admin? = Current.user.has_role?("municipal_admin")`; `location_param(key = "location_id") -> StockLocation | nil` (ativo, por uuid; senão `nil`).

- [ ] **Step 1: Testes**
  - `gate_spec.rb`: sem linha da cidade → `false`; ligado → `true`; `Platform::Features.find("pharmacy").requires == []`.
  - `quantity_spec.rb`: `parse("30") == BigDecimal("30")`; `parse("2.5")` e `parse("0.25")` aceitos; `parse("1,5")`, `parse(1.5)`, `parse("0.001")`, `parse("-2")`, `parse("100000")`, `parse("0")`, `parse(" 3")` → `nil`; `parse("0", allow_zero: true) == 0`; `dump(BigDecimal("30")) == "30.00"`, `dump(BigDecimal("2.5")) == "2.50"`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod27a spec/services/pharmacy/gate_spec.rb spec/services/pharmacy/quantity_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a entrada do CATALOG (Valores fixados 1, comentário "ADR 0045"), o `Gate`, o concern, a base, `Catalog`, `Quantity`, `Cursor` (o padrão do `CareRegulation::Cursor`); `VALID_RANGE = (1..45)`; filtros de log da Global Constraints.
- [ ] **Step 4: Rodar** — os dois specs + `spec/adr_pointers_spec.rb spec/services/platform` → PASS.
- [ ] **Step 5: Commit** — `feat: add the pharmacy switch and the pharmacy route base`.

---

### Task 2: Migração de cidade, modelos, papéis e triggers

**Files:**
- Create: `db/city_migrate/20261029900001_add_pharmacy_stock.rb`, os 15 modelos do mapa, `spec/models/pharmacy_schema_spec.rb`, `spec/support/pharmacy_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `app/models/membership.rb`, `app/models/health_unit.rb`, `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`

**Interfaces:**
- Produces (colunas; `enc` = `encrypts` com a chave da cidade, coluna `text`; toda tabela `id: :uuid`, salvo `pharmacy_settings`; `catalog_item_id` é uuid da plataforma, **sem FK**; `numeric(12,2)` em toda quantidade):
  - `memberships`: `ck_memberships_role` com `AddPharmacyStock::ROLES` (= `AddCareRegulation::ROLES + %w[pharmacist pharmacy_assistant stock_keeper]`, ordenada).
  - `health_units`: `dispenses boolean default false null false`.
  - `stock_locations`: `kind` (CHECK `LOCATION_KINDS`), `name string(120)`, `health_unit_id → health_units` null (CHECK `(kind = 'unit') = (health_unit_id IS NOT NULL)`), `active bool default true`, `updated_by_user_id → users`, timestamps. Únicos: `health_unit_id WHERE health_unit_id IS NOT NULL` (`idx_stock_locations_unit`), `kind WHERE kind = 'central'` (`idx_stock_locations_one_central`), `lower(name)` (`idx_stock_locations_name_ci`).
  - `stock_location_staff`: `stock_location_id`, `user_id → users`, `role` (CHECK `ROLES`), `granted_by_user_id → users`, `granted_at`, `revoked_at` null, `revoked_by_user_id` null (CHECK `(revoked_at IS NULL) = (revoked_by_user_id IS NULL)`). Único `(stock_location_id, user_id, role) WHERE revoked_at IS NULL` (`idx_stock_location_staff_active`); índice `user_id`.
  - `pharmacy_settings`: `id smallint` PK (CHECK `id = 1`), as 5 colunas de `DEFAULT_SETTINGS` (`integer`, CHECK de `SETTING_RANGES`), `updated_by_user_id` null, `updated_at`. O `up` insere a linha padrão.
  - `stock_minimums`: `stock_location_id`, `catalog_item_id`, `minimum numeric` (CHECK `>= 0`), `updated_by_user_id`, timestamps; único `(stock_location_id, catalog_item_id)`.
  - `stock_lots`: `catalog_item_id`, `catmat_code int`, `lot_number string(40)`, `expires_on date`, `manufacturer string(120)` null, `dispensing_unit string(20)`, `created_by_user_id`, `created_at`; único `(catalog_item_id, lot_number)` (`idx_stock_lots_number`); índice `expires_on`.
  - `stock_entries`: `stock_location_id`, `kind` (CHECK `ENTRY_KINDS`), `document_number string(60)` null, `supplier string(120)` null (CHECK `kind = 'opening' OR (document_number IS NOT NULL AND supplier IS NOT NULL)`), `recorded_by_user_id`, `created_at`.
  - `stock_openings`: `stock_location_id`, `catalog_item_id`, `stock_entry_id`, `created_at`; único `(stock_location_id, catalog_item_id)` (`idx_stock_openings_once`).
  - `stock_transfers`: `from_location_id → stock_locations`, `to_location_id → stock_locations` (CHECK diferentes), `status` (CHECK `TRANSFER_STATUSES`), `sent_by_user_id`, `sent_at`, `received_by_user_id` null, `received_at` null (CHECK `(status = 'sent') = (received_at IS NULL)` e `(received_at IS NULL) = (received_by_user_id IS NULL)`), `created_at`; índices `(to_location_id, status)`, `(from_location_id, created_at)`.
  - `stock_transfer_items`: `stock_transfer_id`, `stock_lot_id`, `quantity_sent` (CHECK `> 0`), `quantity_received` null (CHECK `quantity_received IS NULL OR (quantity_received >= 0 AND quantity_received <= quantity_sent)`), `difference_reason enc` null; único `(stock_transfer_id, stock_lot_id)`.
  - `stock_adjustments`: `stock_location_id`, `stock_lot_id`, `kind` (CHECK `ADJUSTMENT_KINDS`), `status` (CHECK `ADJUSTMENT_STATUSES`; CHECK `kind = 'adjustment' OR status = 'applied'`), `quantity numeric` (delta com sinal, CHECK `<> 0`), `counted_quantity` null (CHECK `(kind = 'adjustment') = (counted_quantity IS NOT NULL)`), `balance_at_request numeric`, `reason enc`, `requested_by_user_id`, `decided_by_user_id` null (CHECK `decided_by_user_id IS NULL OR decided_by_user_id <> requested_by_user_id`), `decided_at` null (CHECK `(status IN ('approved','rejected')) = (decided_at IS NOT NULL)`), `decision_reason enc` null, `created_at`; índice `(stock_location_id, status)`.
  - `external_prescriptions`: `patient_id → patients`, `stock_location_id`, `prescriber_name string(120)`, `council` (CHECK `COUNCILS`), `council_state string(2)` (CHECK UF), `council_number string(10)` (CHECK dígitos), `issued_on date`, `notification_number enc` null, `recorded_by_user_id`, `created_at`; índice `patient_id`.
  - `external_prescription_items`: `external_prescription_id`, `position int` (≥ 1; único com a receita), `catalog_item_id`, `catmat_code int`, `quantity numeric(10,2)` (CHECK `> 0 AND <= 9999`), `quantity_unit string(30)`, `dosage_instructions enc`, `continuous bool default false`, `controlled_list string(3)` null (retrato do catálogo).
  - `dispensations`: `stock_location_id`, `patient_id → patients`, `clinical_document_id → clinical_documents` null, `external_prescription_id → external_prescriptions` null (CHECK `num_nonnulls(clinical_document_id, external_prescription_id) = 1`), `dispensed_by_user_id`, `dispensed_by_role` (CHECK `DISPENSER_ROLES`), `dispensed_at`, `retained bool default false`, `notification_number enc` null (CHECK `notification_number IS NULL OR retained`); índices `(patient_id, dispensed_at)`, `clinical_document_id`, `external_prescription_id`.
  - `dispensation_items`: `dispensation_id`, `prescription_item_id → prescription_items` null, `external_prescription_item_id → external_prescription_items` null (CHECK `num_nonnulls(…) = 1`), `catalog_item_id`, `stock_lot_id`, `quantity` (CHECK `> 0`), `mapped_from_free_text bool default false` (CHECK `NOT mapped_from_free_text OR prescription_item_id IS NOT NULL`), `completes_item bool default false`, `created_at`; índices `prescription_item_id`, `external_prescription_item_id`.
  - `stock_movements`: `stock_location_id`, `stock_lot_id`, `catalog_item_id`, `kind` (CHECK `MOVEMENT_KINDS`), `quantity` (CHECK `quantity <> 0 AND abs(quantity) <= 99999.99`, `kind NOT IN POSITIVE OR quantity > 0`, `kind NOT IN NEGATIVE OR quantity < 0`), `stock_entry_id` null, `stock_transfer_id` null, `dispensation_item_id` null, `stock_adjustment_id` null (CHECK `num_nonnulls(…) = 1` e o documento do tipo: entrada ⇔ `opening/receipt/donation`; transferência ⇔ `transfer_out/transfer_in` ou `loss` de diferença; item de dispensação ⇔ `dispensing/return`; ajuste ⇔ `adjustment/expiry` ou `loss` de ajuste), `reason enc` null (CHECK `kind NOT IN ('loss','expiry','adjustment','return') OR reason IS NOT NULL`), `package_units int` null, `packages int` null (CHECK juntos), `recorded_by_user_id`, `recorded_at`, `created_at`; índices `(stock_location_id, stock_lot_id)` `idx_stock_movements_balance`, `(stock_lot_id, recorded_at, id)`, `dispensation_item_id`.
- Triggers (bloco `DO $do$ … IF to_regclass('public.stock_movements') IS NOT NULL …`, depois do bloco do 26a):
  - `rota_stock_balance_guard()` `AFTER INSERT ON stock_movements`: `SUM(quantity)` por `(stock_location_id, stock_lot_id)` de `NEW` `< 0` → `RAISE EXCEPTION 'stock_movements: negative balance refused'`.
  - `stock_movements`, `dispensations`, `external_prescriptions`, `external_prescription_items`: `rota_reencryption_only('reason')`, `('notification_number')`, `('notification_number')`, `('dosage_instructions')`; `stock_lots`, `stock_entries`, `stock_openings`, `dispensation_items`: `rota_append_only()`.
  - `rota_stock_transfer_guard()`: sem DELETE; só `status`, `received_by_user_id`, `received_at` mudam, uma vez, de `sent`. `rota_stock_transfer_item_guard()`: sem DELETE; só `quantity_received` e `difference_reason`, uma vez, de nulo; sob `rota.reencrypting` só `difference_reason`.
  - `rota_stock_adjustment_guard()`: sem DELETE; só `status`, `decided_by_user_id`, `decided_at`, `decision_reason`, uma vez, de `pending` para `approved`/`rejected`; sob `rota.reencrypting` só `reason`/`decision_reason`.
  - `rota_stock_location_staff_guard()`: sem DELETE; só `revoked_at`/`revoked_by_user_id`, uma vez.
  - `stock_locations`, `stock_minimums`, `pharmacy_settings`: `rota_no_delete()`. TRUNCATE recusado nas 15.
- Modelos: `StockLocation` (`belongs_to :health_unit, optional`; `central?`/`unit?`; `scope :active_locations`), `StockLocationStaff` (`self.table_name = "stock_location_staff"`; `scope :active`), `PharmacySetting` (`self.primary_key = "id"`; `self.current -> PharmacySetting`; `self.ensure_defaults!`), `StockMinimum`, `StockLot` (`#expired?(today = Time.zone.today)`), `StockEntry`, `StockOpening`, `StockMovement` (`encrypts :reason`; `inspect` sem `reason`), `StockTransfer` (`has_many :items`), `StockTransferItem` (`encrypts :difference_reason`), `StockAdjustment` (`encrypts :reason, :decision_reason`), `ExternalPrescription` (`encrypts :notification_number`; `has_many :items, -> { order(:position) }`), `ExternalPrescriptionItem` (`encrypts :dosage_instructions`), `Dispensation` (`encrypts :notification_number`; `has_many :items`), `DispensationItem`. Todo modelo com dado pessoal ou clínico (`ExternalPrescription*`, `Dispensation*`, `StockAdjustment`, `StockMovement`, `StockTransferItem`): `inspect`/`to_s`/`pretty_print` só com `id` (e `status`/`kind` quando houver); `as_json` só `id`. `CITY_KEYED_TARGETS` + as 7 colunas cifradas.
- `Membership::ROLES` + os três; `PRIVILEGED_ROLES` + `pharmacist pharmacy_assistant` (comentário: "ADR 0045: veem nome do paciente e o que ele retira; o farmacêutico dispensa controlado — step-up; o mantenedor não concede"); `stock_keeper` **não** (só estoque, sem dado pessoal; P17).
- Helpers (`spec/support/pharmacy_helpers.rb`): `pharmacy_city!(enabled: true)` (`documents_city!` + `import_test_catalog!` + `Platform::Features.set!(... "pharmacy" ...)` + `PharmacySetting.ensure_defaults!` + `CityCatalog.reset_cache!`); `central_location!`; `unit_location!(unit = create_unit("UBS Farmácia #{SecureRandom.hex(2)}"))`; `pharmacy_staff!(location, role)` (pharmacist: `screener!(location.health_unit || create_unit(...), cbo: "223405")`; os outros: `staff_with`; mais a `Membership` do papel e o vínculo, avaliados na chamada); `losartana = catalog_item(268_856)`, `amoxicilina = catalog_item(271_089)`, `fluoxetina = controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1")`, `diazepam = controlled_item!(code: 9_900_002, ingredient: "DIAZEPAM", list: "B1", strength: "10 mg")`; `stocked_lot!(location, item:, quantity: "30", lot_number: "L#{SecureRandom.hex(3).upcase}", expires_on: Time.zone.today + 180, by: nil)` (pela abertura da Task 5 quando não há abertura do item no local, senão por nota; avaliado na chamada); `balance_of(location, lot)`.

- [ ] **Step 1: Teste** (`spec/models/pharmacy_schema_spec.rb`):

```ruby
require "rails_helper"

RSpec.describe "Esquema da farmácia (ADR 0045)" do
  before { pharmacy_city! }
  let(:keeper) { staff_with("almox-#{SecureRandom.hex(2)}@cidade.gov.br", "stock_keeper") }
  let(:central) { central_location! }
  let(:lot) { StockLot.create!(catalog_item_id: losartana.id, catmat_code: 268_856, lot_number: "LS01", expires_on: Date.current + 200, dispensing_unit: "comprimido", created_by_user: keeper) }
  let(:entry) { StockEntry.create!(stock_location: central, kind: "invoice", document_number: "NF-1", supplier: "Distribuidora", recorded_by_user: keeper) }

  def movement(**over)
    StockMovement.create!({ stock_location: central, stock_lot: lot, catalog_item_id: lot.catalog_item_id, kind: "receipt", quantity: 10,
                            stock_entry: entry, recorded_by_user: keeper, recorded_at: Time.current }.merge(over))
  end

  it "movimento nunca muda nem some; sinal pelo tipo; documento obrigatório" do
    row = movement
    expect { row.update_columns(quantity: 20) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { row.delete }.to raise_error(ActiveRecord::StatementInvalid)
    expect { movement(quantity: -1) }.to raise_error(ActiveRecord::StatementInvalid, /ck_stock_movements/)
    expect { movement(stock_entry: nil) }.to raise_error(ActiveRecord::StatementInvalid, /ck_stock_movements/)
  end

  it "o trigger recusa saldo negativo por local e lote" do
    movement
    expect { movement(kind: "loss", quantity: -11, stock_entry: nil, stock_adjustment: loss_adjustment!(central, lot, keeper), reason: "Frascos quebrados no transporte") }
      .to raise_error(ActiveRecord::StatementInvalid, /negative balance refused/)
    expect(StockMovement.where(stock_location: central, stock_lot: lot).sum(:quantity)).to eq(10)
  end

  it "um central só; unidade com health_unit; abertura uma vez por local e item; lote único por item" do
    central
    expect { StockLocation.create!(kind: "central", name: "Outro central", updated_by_user: keeper) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_stock_locations_one_central/)
    expect { StockLocation.create!(kind: "unit", name: "Sem unidade", updated_by_user: keeper) }.to raise_error(ActiveRecord::StatementInvalid)
    StockOpening.create!(stock_location: central, catalog_item_id: losartana.id, stock_entry: entry)
    expect { StockOpening.create!(stock_location: central, catalog_item_id: losartana.id, stock_entry: entry) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_stock_openings_once/)
    lot
    expect { StockLot.create!(lot.attributes.except("id", "created_at")) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_stock_lots_number/)
  end

  it "ajuste decide uma vez e nunca pelo autor; dispensação e receita externa imutáveis" do
    adjustment = StockAdjustment.create!(stock_location: central, stock_lot: lot, kind: "adjustment", status: "pending", quantity: -5, counted_quantity: 5,
                                         balance_at_request: 10, reason: "Contagem do inventário de outubro", requested_by_user: keeper)
    expect { adjustment.update!(status: "approved", decided_by_user: keeper, decided_at: Time.current) }.to raise_error(ActiveRecord::StatementInvalid)
    other = staff_with("almox2-#{SecureRandom.hex(2)}@cidade.gov.br", "stock_keeper")
    adjustment.update!(status: "rejected", decided_by_user: other, decided_at: Time.current, decision_reason: "Recontar a prateleira B")
    expect { adjustment.update_columns(status: "approved") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "papéis novos aceitos; farmacêutico e auxiliar privilegiados, almoxarife não; nada pessoal no inspect" do
    %w[pharmacist pharmacy_assistant stock_keeper].each { |role| expect { staff_with("#{role}-#{SecureRandom.hex(2)}@c.gov.br", role) }.not_to raise_error }
    expect(Membership::PRIVILEGED_ROLES).to include("pharmacist", "pharmacy_assistant")
    expect(Membership::PRIVILEGED_ROLES).not_to include("stock_keeper")
    expect(PharmacySetting.current.attributes.slice(*Pharmacy::Catalog::DEFAULT_SETTINGS.keys)).to eq(Pharmacy::Catalog::DEFAULT_SETTINGS)
    external = ExternalPrescription.new(prescriber_name: "Dr. Fulano", notification_number: "PR-123456")
    expect(external.inspect).not_to include("PR-123456")
    expect(external.as_json.keys).to eq([ "id" ])
  end
end
```

  `loss_adjustment!(location, lot, by)` é auxiliar local do spec (um `StockAdjustment` `loss` `applied` cru). E em `spec/models/membership_roles_spec.rb`: `AddPharmacyStock::ROLES == (AddCareRegulation::ROLES + %w[pharmacist pharmacy_assistant stock_keeper]).sort` e `Membership::ROLES == AddPharmacyStock::ROLES`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod27a spec/models/pharmacy_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`replace_check`/`quoted` como a `20261028900001`; `up` termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`; `down` irreversível), o dump à mão, os modelos, os triggers, a cifra, o `Membership` e os helpers; nos dois specs de migração a restauração do papel passa a usar `AddPharmacyStock::ROLES` (comentário cita o 27a).
- [ ] **Step 4: Rodar** — o spec + `spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: add the pharmacy stock city schema, roles and immutability triggers`.

---

### Task 3: Livro de movimentos, acesso por local e unidade de dispensação

**Files:**
- Create: `app/services/pharmacy/{ledger,access,dispensing_unit,settings}.rb`, `config/pharmacy/dispensing_units.yml`
- Test: `spec/services/pharmacy/{ledger,access,dispensing_unit}_spec.rb`

**Interfaces:**
- Consumes: modelos da Task 2; `Pharmacy::Catalog`, `Pharmacy::Quantity`.
- Produces:
  - `Pharmacy::Ledger::Insufficient < StandardError` (com `#lot_id`); `Ledger::Inactive < StandardError`.
  - `Ledger.balance(location_id:, lot_id:) -> BigDecimal`; `Ledger.balances(location_id:, lot_ids:) -> { lot_id => BigDecimal }` (lotes sem movimento = 0).
  - `Ledger.lock_lots!(lot_ids) -> { id => StockLot }` — `StockLot.where(id: lot_ids.uniq.sort).order(:id).lock("FOR UPDATE")` (ordem por id: duas operações com lotes em ordem inversa nunca se travam).
  - `Ledger.lock_location!(location_id) -> StockLocation` — `FOR SHARE` em local ativo; local `unit` também `HealthUnit.lock_active!(health_unit_id)`; inativo → `Inactive`.
  - `Ledger.record!(location:, lot:, kind:, quantity:, by:, document:, reason: nil, package: nil, at: Time.current) -> StockMovement` — `quantity` positiva (exceto `adjustment`, com sinal); grava com o sinal do tipo; `document` é `StockEntry | StockTransfer | DispensationItem | StockAdjustment`; **pressupõe a trava do lote**; confere `balance + signed >= 0` antes (senão `Insufficient`); erro do trigger (`/negative balance refused/`) também vira `Insufficient`.
  - `Pharmacy::Access.roles_at(user:, location:) -> Set[String]` — papéis com `StockLocationStaff.active` no local **e** `user.has_role?(role)`; `Access.allowed?(user:, location:, roles:) -> bool`; `Access.central_keeper?(user) -> bool`; `Access.locations_for(user) -> [StockLocation]` (ativos onde tem papel; o central keeper vê todos os ativos).
  - `Pharmacy::DispensingUnit.for(catalog_item) -> String` — primeira regra do YAML cuja palavra (sem acento, minúscula) está em `dosage_form`; senão `"unidade"`.
  - `Pharmacy::Settings.current -> PharmacySetting` (sem cache: lido por requisição).

`config/pharmacy/dispensing_units.yml` (ordem importa; primeira que casar vence):

```yaml
source: "dosage_form do catálogo CATMAT da plataforma (Medications::Catmat::DescriptionParser)"
rules:
  - { match: ["comprimido", "dragea", "drágea"], unit: "comprimido" }
  - { match: ["capsula", "cápsula"], unit: "cápsula" }
  - { match: ["creme", "pomada", "gel"], unit: "bisnaga" }
  - { match: ["aerosol", "spray"], unit: "frasco" }
  - { match: ["solucao", "solução", "suspensao", "suspensão", "xarope", "elixir", "colirio", "colírio"], unit: "frasco" }
  - { match: ["ampola", "injetavel", "injetável"], unit: "ampola" }
  - { match: ["envelope", "sache", "sachê", "po ", "pó "], unit: "envelope" }
fallback: "unidade"
```

- [ ] **Step 1: Testes**
  - `ledger_spec.rb`: depois de uma entrada de 10 (movimento cru com entrada), `record!(kind: "dispensing", quantity: BigDecimal("4"), …)` grava `-4` e `balance == 6`; `record!(kind: "loss", quantity: BigDecimal("7"), …)` levanta `Insufficient` e nada é gravado; com `Ledger.balance` simulado em 100 (`allow`), o trigger recusa e ainda vira `Insufficient`; `lock_location!` de local inativo e de local cuja unidade foi desativada levantam `Inactive`; `balances` de lote sem movimento devolve 0.
  - `access_spec.rb`: com `Membership` e vínculo → `{"pharmacist"}`; **Review Focus 4**: `Membership#revoke!` com o vínculo ativo → conjunto vazio; vínculo revogado com `Membership` ativa → vazio; `central_keeper?` só com vínculo `stock_keeper` no local `central`; `locations_for` do central keeper lista todos os ativos.
  - `dispensing_unit_spec.rb`: `"comprimido revestido"` → `"comprimido"`; `"cápsula"` → `"cápsula"`; `"creme vaginal"` → `"bisnaga"`; `"aerosol oral"` → `"frasco"`; `nil` → `"unidade"`; a LOSARTANA do recorte de dev (`dosage_form` do catálogo importado) → `"comprimido"`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod27a spec/services/pharmacy/ledger_spec.rb spec/services/pharmacy/access_spec.rb spec/services/pharmacy/dispensing_unit_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: add the pharmacy ledger, location access and dispensing units`.

---

### Task 4: Administração — locais, vínculos, parâmetros, mínimos, meus locais e busca de item

**Files:**
- Create: `app/commands/pharmacy/{save_location,link_staff,save_settings,save_minimums}.rb`, `app/services/pharmacy/json.rb`, `app/controllers/pharmacy/admin/{locations,staff,settings}_controller.rb`, `app/controllers/pharmacy/{locations,items,minimums}_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/requests/pharmacy/{admin,locations}_spec.rb`

**Interfaces:**
- Consumes: `Pharmacy::Access`, `Pharmacy::Ledger.balances`, `Pharmacy::DispensingUnit`.
- Produces:
  - `SaveLocation.call(attrs:, by:, location: nil) -> Result` — `kind` em `LOCATION_KINDS` (não muda depois de criado), `name` 3..120, `health_unit_id` (só `unit`; unidade ativa; já usada → `unit_taken`), `active`; segundo central → `central_exists`; desativar com saldo > 0 em algum lote → `location_has_stock`. Criar/ativar `unit` grava `health_units.dispenses = true`; desativar grava `false` (mesma transação). Evento `pharmacy.settings_changed` (`subject: "location"`, `stock_location_id`, `user_id`).
  - `LinkStaff.add(location:, user_id:, role:, by:)` / `LinkStaff.remove(location:, user_id:, role:, by:)` — Desvio 3 (`role_not_granted`, `cbo_not_allowed`, `already_linked`, `not_found`); evento `pharmacy.settings_changed` (`subject: "staff"`).
  - `SaveSettings.call(attrs:, by:)` — as 5 chaves obrigatórias, inteiros em `SETTING_RANGES` (senão `invalid_request` `path: "<chave>"`); evento (`subject: "settings"`).
  - `SaveMinimums.call(location:, rows:, by:)` — substitui a lista do local; cada linha `{ catalog_item_id, minimum }` (item ativo do catálogo; `minimum` por `Quantity.parse(allow_zero: true)`); repetido → `invalid_request` `path: "[i].catalog_item_id"`; evento (`subject: "minimums"`).
  - `Pharmacy::Json.location(location) -> { id, name, kind, health_unit: { id, name } | null, active }`; `Json.lot(lot, catalog:)` = `<lot>` do contrato com `item.unit` = `lot.dispensing_unit`; `Json.item(catalog_item, unit:)`.
- Rotas (escopo `/pharmacy`, módulo `pharmacy`, `as: :pharmacy`; admin em `scope "admin", module: :admin`): `get/post "admin/locations"`, `patch "admin/locations/:id"`, `post/delete "admin/locations/:id/staff"` (`{ user_id, role }`; `GET admin/locations` traz `staff: [ { user_id, name, role } ]`), `get/patch "admin/settings"` — leitura `municipal_admin`, escrita `municipal_admin` com `require_step_up!`. `get "locations"` (meus locais: `{ locations: [ <location> + roles ] }`; P3). `get/put "locations/:id/minimums"` (`stock_keeper` do local, ou o central keeper; `PUT` com step-up não — mínimo não é dado sensível). `post "items/search"` (`{ q }` no corpo; qualquer papel da farmácia ou admin; `Medications::Search.call(q:)` sem filtro de REMUME; `Unavailable` → 503 `catalog_unavailable`; P3).

- [ ] **Step 1: Testes**
  - `admin_spec.rb`: admin com step-up cria o central e a farmácia da UBS (`health_unit.reload.dispenses == true`); sem step-up → 401 `mfa_required`; segundo central → 409 `central_exists`; unidade já com local → 409 `unit_taken`; desativar com saldo → 409 `location_has_stock`, depois de zerar → 200 e `dispenses == false`; vincular `pharmacist` sem CBO `2234*` → 422 `cbo_not_allowed`; sem a `Membership` → 422 `role_not_granted`; desvincular duas vezes → 404; `PATCH admin/settings` com `adjustment_approval_percent: 101` → 422 `path: "adjustment_approval_percent"`; sem uma chave → 422; ok → evento `pharmacy.settings_changed` com payload só `subject`, `user_id` (e o id do alvo quando houver); profissional → 403.
  - `locations_spec.rb`: farmacêutica vê só a UBS dela com `roles: ["pharmacist"]`; almoxarife do central vê todos; **Review Focus 4** (rota): `Membership` revogada → `GET locations` vazio e `POST /pharmacy/dispensations` → 403 `missing_role` (o teste da dispensação entra na Task 11, aqui só a lista); `PUT locations/:id/minimums` de outro local → 403; repetido → 422 com `path`; `POST items/search` com `{ q: "losart" }` traz a LOSARTANA com `catalog_item_id` e sem chave de paciente; interruptor desligado → 403 `feature_disabled`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — os dois specs + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: manage pharmacy locations, staff, settings and minimums`.

---

### Task 5: Entradas, lotes e inventário de abertura

**Files:**
- Create: `app/commands/pharmacy/record_entry.rb`, `app/controllers/pharmacy/entries_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/pharmacy/record_entry_spec.rb`, `spec/requests/pharmacy/entries_spec.rb`

**Interfaces:**
- Consumes: `Ledger.lock_lots!`, `Ledger.lock_location!`, `Ledger.record!`, `DispensingUnit.for`, `Quantity.parse`.
- Produces: `Pharmacy::RecordEntry.call(location:, kind:, document:, items:, by:, catalog:, now: Time.current) -> Result` — `ok(entry:, movements:)`. `catalog` = `{ catalog_item_id => MedicationCatalogItem }` lido pelo controlador **fora** da transação (item `active`; `hidden` vale — o catálogo esconde da busca, não do estoque). Validação na ordem e primeira recusa com `path`: `document` (`number` em `DOCUMENT_NUMBER`, `supplier` 1..120; `opening` ignora `document`) → `items` (array 1..`MAX_ENTRY_ITEMS`) → por item: `catalog_item_id` (`invalid_item`), `lot_number` (`LOT_NUMBER`), `expires_on` ISO (`invalid_request`) e `< hoje` (`expired_lot`), `manufacturer` ≤ 120, `quantity`/`package` (Desvio 4; `invalid_quantity`). Transação: lotes existentes por `(catalog_item_id, lot_number)` com `find_or_create` sob `idx_stock_lots_number` (corrida → relê); `lot_mismatch` se a validade ou o fabricante informado divergem; `lock_lots!` → `lock_location!` → `StockEntry` → `StockOpening` por item na abertura (`opening_exists`) → `Ledger.record!` por linha. Evento `stock_receipt.recorded` (`stock_entry_id`, `user_id`).
- Rotas: `post "receipts"` (`stock_keeper` do local; `kind` em `invoice`/`donation` no `document.kind`) e `post "openings"` (`stock_keeper` ou `pharmacist` do local; `kind: "opening"`), ambos → 201 `{ entry: { id, kind, location, items: [ { lot: <lot>, quantity } ] } }`.

- [ ] **Step 1: Testes**
  - `record_entry_spec.rb`: nota com `package: { units_per_package: 30, packages: 10 }` → movimento `receipt` de `300` com `package_units: 30`, `packages: 10`; `quantity: "300"` + a mesma embalagem passa, `"299"` → `invalid_quantity` `path: "items[0].quantity"`; mesmo lote de novo com outra validade → `lot_mismatch`; lote vencido → `expired_lot` `path: "items[0].expires_on"`; abertura com dois lotes do mesmo item passa; segunda abertura do item no local → `opening_exists` e nada gravado; doação vira movimento `donation`; lote existente é reaproveitado (um `StockLot`, dois movimentos); `dispensing_unit` do lote = `"comprimido"`.
  - `entries_spec.rb`: 201 com `quantity: "300.00"` e `item.unit: "comprimido"`; **Review Focus 5**: `"1,5"`, `1.5` (número JSON), `"0.001"`, `"-2"`, `"100000"` → 422 `invalid_quantity` com `path: "items[0].quantity"`; auxiliar de farmácia em `receipts` → 403; farmacêutica em `openings` → 201; local de outro → 403; evento `stock_receipt.recorded` com payload exatamente `stock_entry_id`, `user_id`; a linha "Started POST" do log não contém o número do lote nem o fornecedor (`capture_log`).
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: record pharmacy receipts, donations and opening inventory with package conversion`.

---

### Task 6: Transferência do central para a unidade

**Files:**
- Create: `app/commands/pharmacy/{send_transfer,receive_transfer}.rb`, `app/controllers/pharmacy/transfers_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/pharmacy/transfers_spec.rb`, `spec/requests/pharmacy/transfers_spec.rb`

**Interfaces:**
- Consumes: `Ledger.*`; `Access.allowed?`.
- Produces:
  - `SendTransfer.call(from:, to:, items:, by:, now:) -> Result` — `from.central?` (senão `invalid_request` `path: "from"`), `to` `unit` ativo (`path: "to_location_id"`); itens 1..100, `lot_id` uuid de lote existente, sem repetição (`invalid_items` `path: "items[i].lot_id"`), `quantity` positiva; transação `lock_lots!` → `lock_location!(from)` → `lock_location!(to)` → `StockTransfer` `sent` → itens → `transfer_out` por lote (`Insufficient` → `insufficient_balance` `path: "items[i].quantity"`). Evento `stock_transfer.sent` (`stock_transfer_id`, `user_id`).
  - `ReceiveTransfer.call(transfer:, items:, by:, now:) -> Result` — Desvio 6; ordem `lock_lots!` (lotes da transferência) → `lock_location!(to)` → `StockTransfer.lock` (relê `status`; não `sent` → `already_received`). Evento `stock_transfer.received` (`stock_transfer_id`, `user_id`, `with_difference: bool`).
  - `Json.transfer(transfer, catalog:) -> { id, from: <location>, to: <location>, status, sent_at, received_at, items: [ { lot: <lot>, quantity_sent, quantity_received|null, difference|null } ] }` (sem o motivo na lista; o motivo vai só no detalhe para quem pode receber).
- Rotas: `post "transfers"` (`stock_keeper` do central), `post "transfers/:id/receive"` (papéis do destino; Desvio 2), `get "transfers"` (`location_id` obrigatório e do usuário — origem ou destino; `status` opcional em `TRANSFER_STATUSES`; `before`, `limit`), `get "transfers/:id"` (origem ou destino; senão 404).

- [ ] **Step 1: Testes**
  - `transfers_spec.rb` (comando): enviar 100 de um lote com 120 no central → central 20, transferência `sent`, posição do destino com `in_transit` 100 e saldo 0 (pela `Positions` da Task 8 — aqui conferido por `Ledger.balance` e pelos itens `sent`); receber 100 → `received`, destino 100; receber 95 com motivo → `received_with_difference`, destino 95, extrato do destino `[transfer_in 100, loss -5]` e o `loss` com o motivo; 95 sem motivo → `reason_required` `path: "items[0].reason"`; 101 → `invalid_quantity`; faltando um lote → `invalid_items`; receber de novo → `already_received`; enviar mais que o saldo → `insufficient_balance` e nada gravado.
  - `transfers_spec.rb` (request): auxiliar do destino recebe (200); auxiliar de outra UBS → 404; almoxarife da UBS envia → 403; `GET transfers?location_id=<outro>` → 403; eventos com payload só ids e o booleano.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: send pharmacy transfers in transit and receive them with differences`.

---

### Task 7: Inventário, ajustes, perdas e vencimentos

**Files:**
- Create: `app/commands/pharmacy/{request_adjustment,decide_adjustment}.rb`, `app/controllers/pharmacy/adjustments_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/pharmacy/adjustments_spec.rb`, `spec/requests/pharmacy/adjustments_spec.rb`

**Interfaces:**
- Consumes: `Ledger.*`, `Settings.current`, `Access.allowed?`.
- Produces:
  - `RequestAdjustment.call(location:, lot:, kind:, counted_quantity:, quantity:, reason:, by:, settings:, now:) -> Result` — Desvio 7. Ordem: `lock_lots!([lot.id])` → `lock_location!` → saldo → decisão. Regra do limite: `pending` ⇔ `balance.zero? || delta.abs * 100 > balance * settings.adjustment_approval_percent`. `ok(adjustment:)`. Evento `stock_adjustment.recorded` (`stock_adjustment_id`, `user_id`, `pending: bool`).
  - `DecideAdjustment.call(adjustment:, decision:, reason:, by:, now:) -> Result` — `decision` `approve`|`reject`; `by` ≠ autor (`same_user`); `by` com `stock_keeper` ou `pharmacist` no local (`missing_role`); ordem `lock_lots!` → `lock_location!` → `StockAdjustment.lock` (não `pending` → `not_pending`); aprovar aplica o delta guardado (`Insufficient` → `insufficient_balance`, continua `pending`); rejeitar exige motivo (`reason_required`). Eventos `stock_adjustment.approved` / `.rejected` (`stock_adjustment_id`, `user_id`).
- Rotas (step-up em todas as escritas): `post "adjustments"`, `post "adjustments/:id/approve"`, `post "adjustments/:id/reject"`, `get "adjustments"` (`location_id` do usuário, `status` opcional; P3) → `{ adjustments: [ { id, lot: <lot>, kind, status, quantity, counted_quantity|null, balance_at_request, requested_by: { name }, decided_by|null, created_at } ], next_before }` (motivo só no detalhe `GET adjustments/:id`).

- [ ] **Step 1: Testes** (`adjustments_spec.rb`, comando): saldo 100, contado `"95"` → `applied`, saldo 95, movimento `adjustment -5`; contado `"85"` (15%) → `pending`, saldo 100; aprovado por outro → `approved`, saldo 85; pelo autor → `same_user`; aprovar duas vezes → `not_pending`; entre o pedido e a aprovação saem 90 por dispensação: aprovar → `insufficient_balance` e continua `pending`; saldo 0 e contado `"3"` → `pending`; contado igual → `no_difference`; `loss` `"200"` → `insufficient_balance`; `expiry` de lote não vencido → `lot_not_expired`; com `adjustment_approval_percent` 0 todo ajuste é `pending`. Request: sem step-up → 401; auxiliar → 403; aprovação por almoxarife de outro local → 403; motivo com 9 caracteres → 422 `invalid_reason`; payloads só ids.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: record stock adjustments, losses and expiries with second-person approval`.

---

### Task 8: Posição, extrato e alertas

**Files:**
- Create: `app/services/pharmacy/{positions,statement,alerts}.rb`, `app/controllers/pharmacy/{positions,statements,alerts}_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/pharmacy/positions_spec.rb`, `spec/requests/pharmacy/positions_spec.rb`

**Interfaces:**
- Consumes: `Ledger.balances`, `Settings.current`, `Json.lot`, `Cursor`.
- Produces:
  - `Pharmacy::Positions.call(location_ids:, catalog_item_ids: nil, expiring: false, below_minimum: false, before: nil, limit:, today:, settings:) -> { positions: [<position>], next_before }` — agrupa movimentos por `(stock_location_id, catalog_item_id, stock_lot_id)` em SQL (`SUM`); `in_transit` por lote = itens `sent` com destino no local; inclui lote com saldo 0 só se tiver `in_transit > 0`; `expiring_soon` = algum lote com saldo > 0 e `expires_on <= today + expiry_alert_days`; `below_minimum` = mínimo cadastrado e `balance < minimum`; item com mínimo e sem movimento entra com saldo 0.
  - `Pharmacy::Statement.call(location:, lot:, before:, limit:) -> { lot: <lot>, balance, movements: [ { id, kind, quantity, at, by: { name }, document: { type: "entry"|"transfer"|"dispensation"|"adjustment", id, number|null } } ], next_before }` — dispensação só com o **id da dispensação** (nunca paciente nem receita).
  - `Pharmacy::Alerts.call(location:, today:, settings:) -> { expiring, expired, below_minimum, in_transit, pending_adjustments }` (contagens inteiras; lotes com saldo > 0).
- Rotas (sem trilha, sem dado pessoal): `get "positions"` (`location_id` opcional: central keeper sem ele → todos; outros → o seu, e `location_id` alheio → 403; `q` ≥ 2 caracteres resolvido em ids por `Medications::Search` antes da consulta; `expiring`, `below_minimum` `"true"`; `before`, `limit`), `get "lots/:id/movements"` (`location_id` obrigatório e do usuário), `get "alerts"` (`location_id` obrigatório e do usuário).

- [ ] **Step 1: Testes** — `positions_spec.rb` (serviço): dois lotes no central e um em trânsito para a UBS → posição do central com `balance` somado e a da UBS com `in_transit` e `balance "0.00"`; lote vencendo em 60 dias → `expiring_soon`; mínimo 50 com saldo 40 → `below_minimum`; paginação com `limit: 1` devolve `next_before` e a segunda página o outro item (sem repetir). Request: almoxarife do central vê todos sem `location_id`; farmacêutica da UBS com `location_id` do central → 403; extrato de lote dispensado mostra `document: { type: "dispensation", id }` e o corpo não contém o nome nem o CPF do paciente; alertas com as cinco contagens; nenhum `clinical_record.viewed` publicado por essas rotas; `before` inválido → 422 `invalid_cursor`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: show pharmacy stock positions, lot statements and alerts`.

---

### Task 9: Receita para a farmácia — validade, busca e trilha

**Files:**
- Create: `app/services/pharmacy/{prescription,validity,dispensed,trail}.rb`, `app/controllers/pharmacy/{prescriptions,person_lookups}_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/pharmacy/validity_spec.rb`, `spec/requests/pharmacy/lookup_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocument`, `PrescriptionItem`, `ExternalPrescription` (Task 2), `Settings.current`, `Patients::IdentifierJson`, `Patients::Pairs`, `Patients::Resolve`.
- Produces:
  - `Pharmacy::Prescription = Data.define(:kind, :id, :record, :patient_id, :issued_on, :cancelled, :sncr_number, :items, :prescriber)` e `Prescription::Item = Data.define(:id, :catalog_item_id, :free_text, :printed_description, :quantity, :quantity_unit, :dosage_instructions, :continuous, :controlled, :controlled_list, :valid_until)` — `inspect`/`to_s`/`pretty_print` só `kind` e `id`. `Prescription.digital(document, catalog:, settings:) -> Prescription` (só `kind == "prescription"`; `issued_on` = `issued_at.in_time_zone.to_date`; `sncr_number` = `content_data.dig("sncr", "number")`; `controlled` do item pelo catálogo atual); `Prescription.external(record, catalog:, settings:) -> Prescription`; `Prescription.find(kind:, id:, settings:) -> Prescription | nil` (lê o catálogo da plataforma; usada fora da transação).
  - `Pharmacy::Validity.item_until(issued_on:, explicit:, continuous:, controlled:, external:, settings:) -> Date` (Desvio 8); `Validity.expired?(until_date, today: Time.zone.today) -> bool` (`today > until_date`).
  - `Pharmacy::Dispensed.summary(prescription) -> { item_id => { total: BigDecimal, last_on: Date|nil, completed: bool } }` (de `dispensation_items` + `dispensations.dispensed_at` no fuso da cidade); `next_allowed_on` = `last_on + continuous_interval_days` só para item contínuo.
  - `Pharmacy::Json.prescription_view(prescription, patient:, summary:, settings:, today:)` = `<prescription_view>` do contrato + `valid_until` e `expired` por item (P4); `prescriber` = `{ name, council: "CRM-PR 12345" | null }`.
  - `Pharmacy::Trail.person_viewed!(user:, patient_id:)` (`clinical_record.viewed`, `access: "pharmacy"`, `person_ref: "patient:<id>"`, `reason_code: nil`).
- Rotas (`pharmacist` ou `pharmacy_assistant` de local `unit` ativo = `location_id` do corpo; senão 403): `post "prescriptions/lookup"` — `query_only!`; corpo `{ location_id, short_code }` (normalizado para maiúsculas; formato do CHECK do 19c, senão `invalid_request` `path: "short_code"`; inexistente ou não receita → 404 `prescription_not_found`) ou `{ location_id, patient_id }` (receitas `prescription` emitidas nos últimos `LOOKUP_DAYS`, mais as externas do paciente, mais novas primeiro, até `LOOKUP_MAX`; canceladas e vencidas vêm marcadas, P16) → `{ prescriptions: [<prescription_view>] }`, uma trilha por paciente; 30/min por usuário (`too_many_requests`). `post "person_lookup"` (P3) — corpo `{ cpf }` ou `{ cns }`, a forma e as recusas do `Surveillance::PersonLookupsController` (`invalid_cpf`/`invalid_cns`, `citizen_not_verified` 409, `person_not_found` 404), par validado sem paciente → `Patients::Resolve.call(citizen)` cria o paciente aqui; → `{ patient_id, display_name, has_cpf, cns_masked }`; trilha; 30/min.

- [ ] **Step 1: Testes**
  - `validity_spec.rb`: comum sem prazo emitida hoje → hoje + 30; contínua → + 180; com `valid_until` explícito → ele; externa controlada → + 30 mesmo contínua; mudar `common_validity_days` para 60 muda a conta (lida na hora); **Review Focus 1**: com `Time.use_zone("America/Sao_Paulo")`, receita emitida em `2026-09-30 23:30 -03:00` → `item_until == Date.new(2026, 10, 30)`; `travel_to("2026-10-30 23:50 -03:00")` → não vencida; `travel_to("2026-10-31 00:10 -03:00")` → vencida.
  - `lookup_spec.rb`: busca por `short_code` em minúsculas acha; o corpo traz `items[].dispensed_total "0.00"`, `next_allowed_on: null`, `valid_until` da receita e do item; atestado (`sick_note`) pelo código → 404 `prescription_not_found`; receita cancelada vem com `cancelled: true`; busca por paciente lista digital e externa; `short_code` na query → 422 `invalid_request`; almoxarife → 403; local `central` → 403; uma trilha `clinical_record.viewed` com `access: "pharmacy"` por paciente e nenhuma no `person_lookup` que não achou; o corpo nunca traz `patient_identification` nem CPF (só `has_cpf`/`cns_masked`); a 31ª busca no minuto → 429; `person_lookup` de par validado sem paciente cria o paciente e devolve o id.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: look up prescriptions for dispensing with validity and continuous-use windows`.

---

### Task 10: Receita externa

**Files:**
- Create: `app/commands/pharmacy/record_external_prescription.rb`, `app/controllers/pharmacy/external_prescriptions_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/requests/pharmacy/external_prescriptions_spec.rb`

**Interfaces:**
- Consumes: `Patients::CreateTyped.call(attrs:, by:)`, `Prescription.external`, `Json.prescription_view`, `Trail.person_viewed!`.
- Produces: `RecordExternalPrescription.call(location:, patient_id:, new_patient:, prescriber:, issued_on:, notification_number:, items:, by:, role:, catalog:, today:) -> Result` — ordem de validação e primeira recusa com `path`: paciente (`patient_id` existente **ou** `new_patient` pelo `CreateTyped`, cujas recusas voltam com o `path` prefixado `new_patient.`; os dois ou nenhum → `invalid_request` `path: "patient_id"`) → `prescriber` (`name` 3..120, `council` em `COUNCILS`, `council_state` UF, `council_number` dígitos) → `issued_on` (ISO, não futura, não vencida pela regra do Desvio 8 → `prescription_expired`) → `items` 1..20 (`catalog_item_id` ativo, `quantity` positiva, `quantity_unit` 1..30, `dosage_instructions` 3..500, `continuous` booleano) → item controlado com `role != "pharmacist"` → `controlled_requires_pharmacist` → item de lista `NOTIFICATION_LISTS` sem `notification_number` → `notification_required` `path: "notification_number"` (formato `NOTIFICATION_NUMBER`). Grava a receita e os itens (com `controlled_list` do catálogo), evento `external_prescription.recorded` (`external_prescription_id`, `user_id`).
- Rota: `post "external_prescriptions"` (`pharmacist`/`pharmacy_assistant` do local `unit` do corpo) → 201 `<prescription_view>` (`kind: "external"`, `short_code: null`), trilha por pessoa.

- [ ] **Step 1: Teste** — farmacêutica registra DIAZEPAM B1 com `notification_number: "PR-2026-000123"` → 201, `items[0].controlled: true`, `valid_until` = `issued_on + 30`; sem número → 422 `notification_required`; auxiliar registrando DIAZEPAM → 403 `controlled_requires_pharmacist`; auxiliar com LOSARTANA contínua para paciente sem CPF criado por `new_patient` (`no_cpf_reason` do 24a) → 201 e `patient.has_cpf: false`; `new_patient` com nome inválido → 422 `path: "new_patient.full_name"`; `issued_on` de amanhã → 422 `path: "issued_on"`; `council: "CRF"` → 422 `path: "prescriber.council"`; evento com payload exatamente `external_prescription_id`, `user_id`; a linha de log da requisição não contém o número da notificação nem o nome do prescritor.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: record external prescriptions for dispensing`.

---

### Task 11: Dispensação, devolução e histórico do paciente

**Files:**
- Create: `app/commands/pharmacy/{dispense,record_return}.rb`, `app/controllers/pharmacy/{dispensations,returns}_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/pharmacy/dispense_spec.rb`, `spec/requests/pharmacy/{dispensations,returns}_spec.rb`

**Interfaces:**
- Consumes: `Prescription.find`, `Validity`, `Dispensed.summary`, `Ledger.*`, `Access.roles_at`, `Settings.current`.
- Produces:
  - `Pharmacy::Dispense.call(location:, prescription:, items:, retention:, by:, role:, catalog:, settings:, now: Time.current) -> Result` — `ok(dispensation:)`. `prescription` é o `Pharmacy::Prescription` lido fora da transação; `catalog` cobre os itens da receita e os `catalog_item_id` do corpo. **Validação antes da transação**, na ordem, primeira recusa com `path`: `items` 1..20 → por linha: `prescription_item_id` da receita (`invalid_item` `path: "items[i].prescription_item_id"`), texto livre (Desvio 10: papel e `catalog_item_id`), `catalog_item_id` diferente do item de catálogo da receita → `invalid_item`, `quantity` (`invalid_quantity`), `complete` booleano opcional; controlado → papel e `retention` (Desvio 10). **Transação**, ordem de travas: `lock_lots!(lot_ids)` → `lock_location!(location)` → documento (`ClinicalDocument.lock` / `ExternalPrescription.lock`), e **sob as travas**: cancelada → `prescription_cancelled`; vencida por item → `prescription_expired` `path: "items[i]"`; `too_early` (`next_allowed_on`) e `item_completed` pelo `Dispensed.summary` relido; lote do item (`invalid_item` `path: "items[i].lot_id"`) e não vencido (`lot_expired`); saldo pela **soma por lote** (Desvio 11; `insufficient_balance` com o `path` da linha que estoura). Grava `Dispensation` (`dispensed_by_role` = `role`, `retained`, `notification_number`), itens e um `dispensing` por item. Evento `dispensation.recorded` (`dispensation_id`, `user_id`).
  - `Pharmacy::RecordReturn.call(location:, dispensation_item:, quantity:, reason:, by:, now:) -> Result` — Desvio 12; ordem `lock_lots!` → `lock_location!` → `Dispensation.lock`; movimento `return` positivo com o item como documento. Evento `dispensation.returned` (`dispensation_id`, `user_id`).
  - `Pharmacy::Json.dispensation(dispensation, catalog:, patient:)` = `<dispensation>` do contrato com `retention: { notification_number }` só quando `retained`.
- Rotas: `post "dispensations"` (corpo do contrato §7; `role` = `pharmacist` se o usuário tem os dois papéis no local; sem papel dispensador no local `unit` → 403 `missing_role`) → 201; `get "dispensations"` (`patient_id` obrigatório; dispensador de qualquer local `unit`; trilha por pessoa; `before`, `limit`) → `{ dispensations: [<dispensation>], next_before }`; `post "returns"` (`{ location_id, dispensation_item_id, quantity, reason }`) → 201 `{ movement_id, balance }`.

- [ ] **Step 1: Testes** (`dispense_spec.rb`, sobre receitas emitidas por `ClinicalDocuments::Issue` numa `finalized_consultation!`):
  - LOSARTANA contínua: dispensar `"30"` → saldo cai 30, `dispensed_total "30.00"`, `last_dispensed_on` hoje; de novo no 24º dia (`travel_to`) → `too_early` com `next_allowed_on` = dia 25; no 25º dia → ok.
  - AMOXICILINA comum: duas parciais e `complete: true` na segunda → a terceira → `item_completed`.
  - Receita vencida → `prescription_expired` `path: "items[0]"`; lote vencido → `lot_expired`; lote de outro item → `invalid_item` `path: "items[0].lot_id"`.
  - Texto livre: auxiliar → 403 `free_text_requires_pharmacist`; farmacêutica com `catalog_item_id` → ok e `mapped_from_free_text: true`.
  - FLUOXETINA C1 (receita de controle especial digital com SNCR simulado; `controlled_city!(mock: true)`): auxiliar → `controlled_requires_pharmacist`; farmacêutica sem `retention` → `retention_required`; com `retention: {}` → `retained: true` e `notification_number` = o número SNCR do documento; com número diferente → `invalid_request` `path: "retention.notification_number"`.
  - Receita externa de DIAZEPAM: farmacêutica com `retention: {}` → número da notificação da receita externa.
  - **Review Focus 3:** lote com 10, duas linhas do mesmo lote com `"6"` e `"5"` → `insufficient_balance` `path: "items[1].quantity"`, nenhum movimento nem dispensação.
  - **Review Focus 2:** dispensação depois de `ClinicalDocuments::Cancel` (receita lida antes do cancelamento: `Prescription.find` fora, `Cancel`, depois `Dispense`) → `prescription_cancelled`, saldo intacto; e numa thread: `Dispense` segurando a transação (gate com `Queue`, como o `inventory_concurrency_spec`) → o `Cancel` concorrente espera a trava do documento e termina depois (`use_transactional_tests = false`, limpeza do `after` como no spec do 21c).
  - Local `unit` cuja unidade foi desativada → `location_inactive`.
- Requests: `dispensations_spec.rb` — 201 com o `<dispensation>`; **Review Focus 4**: farmacêutica com a `Membership` revogada e o vínculo ativo → 403 `missing_role`; **Review Focus 5**: `"1,5"` e `1.5` → 422 `invalid_quantity` `path: "items[0].quantity"`; `GET dispensations?patient_id=` com trilha; evento `dispensation.recorded` só com `dispensation_id`, `user_id`; corpo de erro sem nome de medicamento. `returns_spec.rb` — devolver `"5"` de 30 → saldo +5 e extrato com `return`; devolver mais que o restante → `return_exceeds`; motivo curto → `invalid_reason`; auxiliar de outra UBS → 404.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — os três specs + `spec/commands/clinical_documents` → PASS.
- [ ] **Step 5: Commit** — `feat: dispense prescriptions from lots, with controlled retention, returns and patient history`.

---

### Task 12: Concorrência com threads reais

**Files:**
- Create: `spec/commands/pharmacy/stock_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Pharmacy::Dispense`, `Pharmacy::ReceiveTransfer`, `Pharmacy::SendTransfer`, helpers da Task 2. Mesmo esqueleto do `spec/commands/lab/inventory_concurrency_spec.rb` (`use_transactional_tests = false`, `in_city`, `empty_tables`, `release_gate`, `pop(timeout: 10)`, limpeza com `session_replication_role = replica` no `after`).

- [ ] **Step 1: Escrever o spec** com quatro exemplos:
  1. **Último comprimido, duas dispensações ao mesmo tempo** (duas receitas de LOSARTANA de pacientes diferentes, lote com `"1"` na UBS, duas farmacêuticas): resultados `contain_exactly(:ok, :insufficient_balance)`; saldo `0`; um movimento `dispensing -1`; um evento `dispensation.recorded`.
  2. **A segunda espera a trava do lote**: a primeira segura a transação (gate); a segunda fica parada (`sleep 0.3`, fila vazia); liberada, a segunda leva `insufficient_balance`.
  3. **Transferência × dispensação** (UBS com saldo 0 e transferência `sent` de 10 do mesmo lote): `ReceiveTransfer` segura a transação depois de gravar o `transfer_in`; a `Dispense` de `"4"` na UBS fica parada na trava do lote; liberado o recebimento, a dispensação passa com saldo final `6`. E o inverso: a dispensação começa antes do recebimento ser confirmado e o recebimento é desfeito (`raise ActiveRecord::Rollback` depois do gate) → a dispensação leva `insufficient_balance`.
  4. **Lotes em ordem inversa não travam**: dispensação A com linhas [lote1, lote2] e dispensação B com [lote2, lote1], ao mesmo tempo, com saldo para as duas → as duas `ok` em menos de 5 s, nenhum `ActiveRecord::Deadlocked`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod27a spec/commands/pharmacy/stock_concurrency_spec.rb` → PASS (o código já existe; se falhar, o defeito é da trava: corrigir em `Ledger`/`Dispense`/`ReceiveTransfer` e repetir). Rodar 3 vezes seguidas (`--keep-db`) para descartar flake.
- [ ] **Step 3: Commit** — `test: prove pharmacy stock never goes negative under concurrent dispensing and transfers`.

---

### Task 13: Linha do tempo e última retirada na lista de medicamentos

**Files:**
- Create: `app/services/timeline/sources/dispensation_source.rb`
- Modify: `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/services/timeline/sources/labels.rb`, `app/services/clinical_documents/json.rb`, `app/controllers/patient_medications_controller.rb`
- Test: `spec/services/timeline/sources/dispensation_source_spec.rb`, `spec/services/timeline/query_spec.rb`, `spec/requests/patient_medications_spec.rb`

**Interfaces:**
- Consumes: `Dispensation`, `DispensationItem`, `Timeline::Entry`, `Timeline::Sources::Window`.
- Produces:
  - `Timeline::KINDS` + `"dispensation"` (último); `Timeline::Query::SOURCES["dispensation"] = Sources::DispensationSource`; `Timeline::Query::PROFESSIONAL_ONLY_KINDS = %w[dispensation]`, subtraído em `citizen_kinds` (P10).
  - `DispensationSource.entries(person:, before:, limit:, from:, to:)` — `[]` sem `person.patient_id`; **sem** checar o interruptor (Desvio 14); uma entrada por dispensação (`occurred_at: dispensed_at`, `unit` = unidade do local, `professional: { name, cbo_code: nil }`, `title: "Dispensação de medicamentos"`, `summary` = rótulos do catálogo dos itens (`MedicationCatalogItem#label`) unidos por `"; "`, **sem lote nem quantidade**, `status: "dispensed"`, `detail_ref: { type: "dispensation", id }`, `citizen: nil`, `document: nil`); catálogo lido de uma vez para a página.
  - `ClinicalDocuments::Json.medication(medication, catalog:, last_dispensed: {})` ganha `last_dispensed_on` (`last_dispensed[medication.catalog_item_id]&.iso8601`; texto livre → `null`). `PatientMedicationsController#index` monta `last_dispensed` = `DispensationItem.joins(:dispensation).where(dispensations: { patient_id: }).group(:catalog_item_id).maximum("dispensations.dispensed_at")` convertido para data na cidade. `patient_medications` não muda.

- [ ] **Step 1: Testes** — `dispensation_source_spec.rb`: paciente com uma dispensação de LOSARTANA e AMOXICILINA → entrada com `summary` contendo os dois rótulos e sem o número do lote; ordem com outras entradas pela chave total; interruptor desligado depois → a entrada continua; `query_spec.rb`: `citizen_kinds(verified: true, documents: true) == Timeline::KINDS - %w[dispensation]` (a expectativa antiga da linha 71 muda com comentário "27a: dispensação só para o profissional"); `patient_medications_spec.rb`: LOSARTANA em uso dispensada há 3 dias → `last_dispensed_on` = essa data; item sem dispensação → `null`; nenhuma linha nova em `patient_medication_events`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — os três specs + `spec/requests/timeline_spec.rb spec/requests/pec_timeline_spec.rb spec/invariants/timeline_invariants_spec.rb spec/services/timeline` → PASS.
- [ ] **Step 5: Commit** — `feat: show dispensations on the patient timeline and last pickup on the medication list`.

---

### Task 14: Exclusão LGPD — dispensação retida

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb`
- Test: `spec/commands/citizens/pharmacy_erasure_spec.rb`

**Interfaces:**
- Produces: `Citizens::RequestErasure.dispensed?(ids) -> bool` (`Dispensation.where(patient_id: ids).exists? || ExternalPrescription.where(patient_id: ids).exists?`), chamado no fim de `examined_patients?` (comentário "ADR 0045: dispensação e receita externa são registro clínico retido, como o prontuário").

- [ ] **Step 1: Teste** — no molde do `regulation_erasure_spec.rb` (`allow(...).to receive_messages(attended?: false, documented?: false)`): paciente cujo único registro é uma receita externa dispensada → pedido nasce `retained`; só a receita externa, sem dispensação → `retained`; `Citizens::Erase` reconfere e não apaga; movimentos de estoque intactos depois; sem nada → `pending` como antes.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — o spec + `spec/commands/citizens` → PASS.
- [ ] **Step 5: Commit** — `feat: retain citizens with pharmacy dispensations on erasure`.

---

### Task 15: Semente de dev realista (Curitiba) para a prova no navegador

**Files:**
- Create: `lib/pharmacy_crew.rb`, `spec/lib/pharmacy_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- Produces: `PharmacyCrew.seed_current_city(slug:, ddd:, password:) -> { switch:, users: [ { email, role, otpauth_uri } ], locations:, lots:, transfers: { status => count }, adjustments: { status => count }, dispensations: }` e `PharmacyCrew.summary_lines(result, password:) -> [String]` — só Curitiba (outras: `{ switch: "fora" }`); liga `pharmacy` pelo mantenedor de dev; usuários e OTP fixos (Valores fixados 5; vínculo profissional CBO `223405` da farmacêutica na UBS); locais (Valores fixados 5); **tudo pelos comandos do app** (`SaveLocation`, `LinkStaff`, `SaveSettings` só se faltar a linha, `SaveMinimums`, `RecordEntry`, `SendTransfer`, `ReceiveTransfer`, `RequestAdjustment`, `ClinicalDocuments::Issue` em consultas da médica do `CareLinesCrew`, `RecordExternalPrescription`, `Dispense`), datas passadas por `travel_to` em volta do comando:

  | o quê | onde | para mostrar |
  |---|---|---|
  | abertura: LOSARTANA 50 mg (2 lotes, um vencendo em 60 dias), METFORMINA 500 mg, SINVASTATINA 20 mg, AMOXICILINA 500 mg, FLUOXETINA 20 mg, DIAZEPAM 10 mg | central (há 40 dias) | inventário de abertura; alerta de validade |
  | NF `000123456` da "Distribuidora de Medicamentos Paraná Ltda.", LOSARTANA 30 × 20 caixas | central (há 20 dias) | conversão de embalagem |
  | transferência recebida inteira | central → UBS (há 30 dias) | saldo da UBS |
  | transferência recebida com diferença ("Duas caixas avariadas no transporte") | central → UBS (há 10 dias) | `received_with_difference` |
  | transferência `sent` | central → UBS (hoje) | em trânsito, recebimento na prova |
  | ajuste `pending` de 20% (contagem de SINVASTATINA) | central | aprovação pela segunda almoxarife |
  | mínimo de METFORMINA acima do saldo | UBS | abaixo do mínimo |
  | receita de LOSARTANA contínua do José, dispensada há 26 dias | UBS | próxima retirada liberada |
  | receita de LOSARTANA contínua do Sebastião, dispensada há 10 dias | UBS | `too_early` |
  | receita de AMOXICILINA comum (pessoa nova do 26a) emitida há 40 dias | — | `prescription_expired` |
  | receita de controle especial de FLUOXETINA (SNCR simulado), não dispensada | — | dispensação só pela farmacêutica, com retenção |
  | receita externa de DIAZEPAM B1 (notificação `PR-2026-000777`) para paciente sem CPF criado como no 24a | UBS | receita externa de controlado |

  Idempotente (local, lote, transferência, receita e dispensação da semente são achados pelo número/código e retomados; nunca duplicados). Sem `controlled_prescriptions` em Curitiba (Task 0, Step 3), a linha da FLUOXETINA é pulada e o `summary_lines` diz por quê. Só e-mail, papel e otpauth na saída, nunca dado de paciente.

- [ ] **Step 1: Teste** — `spec/lib/pharmacy_crew_spec.rb`: numa cidade de teste com o slug `curitiba` simulado como o `regulation_crew_spec.rb`, roda duas vezes e na segunda não cria movimento, transferência, ajuste nem dispensação novos; contagens conferem a tabela (`transfers` `{ "received" => 1, "received_with_difference" => 1, "sent" => 1 }`, `adjustments` `{ "pending" => 1 }`); saldo de nenhum (local, lote) negativo; a saída do `summary_lines` não contém CPF, CNS nem nome de paciente.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar** e ligar em `db/seeds.rb` depois do `RegulationCrew` (bloco "Farmácia e almoxarifado (módulo 27a, ADR 0045)", no padrão dos anteriores). **Não rodar `db:seed` no dev** (só na prova, com autorização). **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: seed a realistic pharmacy stock and dispensing scenario for Curitiba`.

---

### Task 16: Revisão, runbook e suíte completa

**Files:**
- Create (worktree `docs-mod27a`, não publicado): `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/docs-mod27a/operacao/rollout-farmacia.md`

- [ ] **Step 1: Revisão do branch** contra a spec e o contrato (superpowers:requesting-code-review, um revisor no modelo mais capaz): Review Focus 1–5; `/opt/homebrew/bin/git -C apps/api/.claude/mod27a grep -n "DomainEvents.publish(" -- app/commands/pharmacy app/services/pharmacy app/controllers/pharmacy` conferido chamada inteira (payload só com ids e booleanos); `grep -n "Platform.audit("` sem chamada nova; nenhum `Current.city =`; nenhum `notification_number`/`short_code`/`reason`/`dosage` em `Rails.logger`; ordem de travas lotes → local → documento em `RecordEntry`, `SendTransfer`, `ReceiveTransfer`, `RequestAdjustment`, `DecideAdjustment`, `Dispense` e `RecordReturn`; catálogo e interruptores lidos fora das transações; `domain_events_bindings`/`city_encrypted_attributes_guard`/`adr_pointers`/`platform_event_payload_guard` verdes.

- [ ] **Step 2: Runbook** `operacao/rollout-farmacia.md` (criar o worktree com `/opt/homebrew/bin/git -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/docs worktree add ../.worktrees/docs-mod27a origin/main` se não existir; no padrão dos `rollout-*.md`): pré-requisitos (catálogo CATMAT e listas Anvisa ativos na plataforma; unidades que dispensam com cadastro; farmacêuticos com vínculo profissional CBO `2234*`; quem terá `pharmacist`/`pharmacy_assistant` — privilegiados, concedidos por admin com step-up — e `stock_keeper`); ordem (imagem do api → `bin/migrate` — **migração de cidade irreversível**, mexe no CHECK de papéis e em `health_units`; dashboard; maintenance; criar o central e as farmácias, vincular as pessoas, conferir os parâmetros — validade 30/180, intervalo 25, alerta 90, aprovação 10% — e os mínimos **antes** de ligar); **inventário de abertura** (contagem física por local e lote, com validade e fabricante, digitada em `POST /pharmacy/openings` por local antes de ligar a dispensação; uma por item e local — corrigir depois só por ajuste; conferir a posição contra a contagem com o farmacêutico responsável e guardar a planilha assinada da contagem fora do sistema; nada de saldo por planilha importada); ligar `pharmacy` por cidade pelo maintenance; conferências pós-deploy (`SELECT * FROM pharmacy_settings`; `SELECT tgname FROM pg_trigger WHERE tgname LIKE 'stock_%' OR tgname LIKE 'dispensation%' OR tgname LIKE 'external_prescription%'` com o `stock_movements_balance` presente; `SELECT stock_location_id, stock_lot_id, SUM(quantity) FROM stock_movements GROUP BY 1, 2 HAVING SUM(quantity) < 0` vazio; uma dispensação de teste em staging da busca da receita ao extrato); operação (transferência fica "em trânsito" até a unidade receber; diferença vira perda com motivo; ajuste acima de 10% espera outra pessoa do local; controlado só pela farmacêutica, com retenção; receita de papel A/B entra como receita externa com o número da notificação; devolução volta ao mesmo lote); o que **não** faz (não envia ao BNAFAR/Hórus — 27b; não substitui o SNGPC; sem compras, empenho, componente especializado ou Farmácia Popular; sem insumos — entrega seguinte); LGPD (dispensação e receita externa retêm o cadastro; movimentos não dependem do paciente); reversão (desligar o interruptor: as telas somem, linha do tempo e última retirada seguem; nada se apaga; o estoque fica parado como está); gates de go-live (inventário de abertura conferido por local; regra de unidade de dispensação conferida pelo farmacêutico responsável contra a REMUME; prazos de validade aprovados pela assistência farmacêutica da cidade; impressão/retensão de controlados conferida com a vigilância sanitária).

- [ ] **Step 3: Escopo fechado verde** — `bin/test api -w mod27a spec/services/pharmacy spec/commands/pharmacy spec/requests/pharmacy spec/models/pharmacy_schema_spec.rb spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/commands/citizens spec/commands/clinical_documents spec/requests/patient_medications_spec.rb spec/requests/timeline_spec.rb spec/requests/pec_timeline_spec.rb spec/services/timeline spec/invariants/timeline_invariants_spec.rb spec/lib/pharmacy_crew_spec.rb spec/architecture spec/initializers spec/events spec/adr_pointers_spec.rb` → PASS.

- [ ] **Step 4: Pedir a suíte completa** à sessão "Testes" (mudança transversal: migração de cidade, papéis novos no CHECK, `health_units`, `spec/support`, linha do tempo e lista de medicamentos): SendMessage `api · .claude/mod27a/feat/mod-27a-pharmacy-stock · --full · migração de cidade 20261029900001, papéis pharmacist/pharmacy_assistant/stock_keeper, linha do tempo e módulo 27a`. Antes, `docker compose stop worker` (memória "Suíte com worker parado") e `start` depois. Falha → diagnóstico nesta sessão (a "Testes" só responde o resumo).

- [ ] **Step 5: Commit final** (se a revisão mudou algo) — `fix: address module 27a review findings`; avisar a coordenadora: plano do api concluído, runbook no `docs-mod27a` (não publicado), divergências P1–P18 pendentes de decisão no contrato, cards F-27.x seguem In Progress até o dashboard e o maintenance.

---

## Self-review

- **Cobertura da spec:** §2 decisão 1 (escopo, Goal, runbook); 2 (27a só; runbook "não faz"); 3 locais (2, 4); 4 itens CATMAT com lote e validade (2, 5); 5 quantidade com 2 casas (1, 5, 11); 6 movimentos imutáveis, entrada, transferência, dispensação, perda, vencimento, ajuste, devolução, saldo nunca negativo (2, 3, 5, 6, 7, 11, 12); lote que vence primeiro — a sugestão é da tela, o api dá `expires_on` na posição (8); 7 alertas (8); 8 dispensação total/parcial, cancelada/vencida, contínuo 25 dias, receita externa (9, 10, 11); 9 controlados (10, 11); 10 papéis (2, 3, 4); 11 linha do tempo e lista (13); 12 interruptor e LGPD (1, 14). Decisões do coordenador: validade `valid_until`/30/180 por cidade (4, 9); quantidade digitada sem conversão, total e "por completo" (9, 11); texto livre só farmacêutico (11); controlados e notificação A/B como externa (10, 11); step-up e segundo responsável (7); lista de medicamentos só leitura (13); paciente do 24a (9, 10); quem atua onde (3, 4, 8); LGPD (14). §3 dados (2). §5 trilha e ordem de travas (Desvio 13, 3, 9, 11, 12). §6 testes: concorrência (12), trânsito e diferença (6), segundo responsável (7), cancelada/vencida/contínua (9, 11), controlado (11), imutabilidade (2), LGPD (14). §7 runbook com abertura (16). §9 riscos: abertura (5, 16), embalagem (5), concorrência (12). F-27.1: 1, 2, 3, 4; F-27.2: 5; F-27.3: 6; F-27.4: 7; F-27.5: 9, 11, 13; F-27.6: 10, 11; F-27.7: 8.
- **Tipos:** `Ledger.lock_lots!/lock_location!/record!/balance(s)` em 3, 5–8, 11, 12; `Access.roles_at/allowed?/central_keeper?/locations_for` em 3, 4, 6–11; `Pharmacy::Prescription`/`Prescription.find` em 9, 11, 12; `Validity.item_until` em 9, 10, 11; `Dispensed.summary` em 9, 11; `Quantity.parse/dump` em 1, 4–11; `Json.lot/location/transfer/prescription_view/dispensation` em 4–11; `Trail.person_viewed!` em 9, 10, 11.
- **Proporção:** código só no esquema (Task 2) e na regra de unidade de dispensação (Task 3); o resto por assinatura, valores e testes.

## Divergências propostas ao contrato

- **P1 — Unidade de dispensação (§2 `<lot>.item.unit`, spec decisão 5).** O catálogo não tem unidade de dispensação: `medication_catalog_items.unit` é a da concentração (`mg`). `item.unit` passa a ser a unidade derivada de `dosage_form` por `config/pharmacy/dispensing_units.yml` (fallback `"unidade"`) e gravada no lote na primeira entrada (`stock_lots.dispensing_unit`).
- **P2 — Tabelas além das do ADR.** `stock_location_staff`, `pharmacy_settings`, `stock_entries`, `stock_openings`, `stock_transfer_items`, `stock_adjustments`, `dispensation_items`, `external_prescription_items`; `health_units.dispenses`. Acrescentar à lista de "Consequências" do ADR 0045.
- **P3 — Rotas que faltam.** `GET /pharmacy/locations` (meus locais com papéis); `POST /pharmacy/items/search` (`{ q }`, catálogo para quem não é profissional; 503 `catalog_unavailable`); `POST /pharmacy/person_lookup` (`{ cpf }` ou `{ cns }`, forma do 24a/25; par validado sem paciente vira paciente); `GET /pharmacy/adjustments` e `GET /pharmacy/adjustments/:id`; `GET /pharmacy/transfers/:id`; `POST /pharmacy/returns` (`{ location_id, dispensation_item_id, quantity, reason }`) para o movimento `return`, que o contrato lista sem rota.
- **P4 — Validade por item (§2 `<prescription_view>`).** Cada item ganha `valid_until` e `expired` (comum 30, contínuo 180, explícito do 19c vence tudo, externa controlada 30); a receita mostra o maior `valid_until` e `expired` = todos vencidos. Receita mista (aguda + contínua) não vence inteira no 30º dia.
- **P5 — Intervalo do uso contínuo.** Contado por item da receita (não por paciente × medicamento): receita nova do mesmo medicamento não herda o intervalo.
- **P6 — Aprovação só no ajuste de contagem (§6).** `loss` e `expiry` são aplicados na hora (motivo e step-up); só `adjustment` vai a `pending` acima do percentual (saldo 0 conta como acima). Ajuste e abertura também pelo `pharmacist` do local; entrada (nota/doação) só `stock_keeper`. `expiry` só de lote já vencido (`lot_not_expired`).
- **P7 — Diferença da transferência (§5).** O recebimento grava `transfer_in` do enviado e `loss` da diferença no destino (com o motivo); recebido acima do enviado é recusado; a transferência é recebida inteira, uma vez (`invalid_items`, `already_received`).
- **P8 — Retenção.** Digital: o número retido é o `sncr.number` do documento (o corpo pode mandar `retention: {}`; número divergente é recusado); externa: o `notification_number` da receita externa. Antimicrobiano não é tratado como controlado no 27a (auxiliar dispensa, sem retenção): **decisão pedida** — a RDC 471/2021 manda reter a 2ª via; propor `retention` também para `category: antimicrobial`.
- **P9 — Receita externa.** `notification_number` obrigatório com item de lista A/B (`notification_required`); validade fixa de 30 dias para item controlado; conselho em `CRM`/`CRO`/`COREN`.
- **P10 — Linha do tempo.** `dispensation` só para o profissional (fora de `citizen_kinds`; o wpda não muda) e visível com o interruptor desligado (registro retido); `summary` com os nomes dos medicamentos, sem lote nem quantidade.
- **P11 — Eventos (§11).** Acrescentar `dispensation.returned`. Payloads: `stock_receipt.recorded` (`stock_entry_id`, `user_id`); `stock_transfer.sent`/`.received` (`stock_transfer_id`, `user_id`; `.received` com `with_difference`); `stock_adjustment.recorded` (`stock_adjustment_id`, `user_id`, `pending`), `.approved`/`.rejected` (`stock_adjustment_id`, `user_id`); `dispensation.recorded`/`.returned` (`dispensation_id`, `user_id`); `external_prescription.recorded` (`external_prescription_id`, `user_id`); `pharmacy.settings_changed` (`subject` ∈ `settings location staff minimums`, `user_id`, id do alvo). Trilha: `clinical_record.viewed` com `access: "pharmacy"`.
- **P12 — Recusas novas.** 422 `lot_mismatch`, `no_difference`, `lot_not_expired`, `item_completed`, `notification_required`, `return_exceeds`, `invalid_items`, `invalid_item` (com `path`), `invalid_reason`, `reason_required`, `cbo_not_allowed`, `role_not_granted`, `invalid_cursor`, `invalid_limit`, `invalid_request`; 409 `opening_exists`, `already_received`, `not_pending`, `location_has_stock`, `location_inactive`, `central_exists`, `unit_taken`, `already_linked`; 403 `same_user`, `missing_role`; 404 `prescription_not_found`; 503 `catalog_unavailable`; 429 `too_many_requests` na busca de receita e de pessoa. `prescription_expired` traz `path: "items[i]"`.
- **P13 — Formato da quantidade.** Entrada só string (`QUANTITY`, até 5 inteiros e 2 casas); saída sempre com duas casas e ponto (`"30.00"`).
- **P14 — Ordem das posições.** `(stock_location_id, catalog_item_id)` com cursor opaco (o nome do item mora na plataforma); a tela ordena a página.
- **P15 — Receita do 19c em papel.** `kind: "digital"` significa "documento do 19c": a receita emitida em papel pelo sistema (mesmo `short_code`) também é dispensável; o registro da Notificação (`controlled_notification_record`) não é receita e entra como externa.
- **P16 — Busca por paciente.** Receitas dos últimos 365 dias e externas, até 20, mais novas primeiro, com canceladas e vencidas marcadas.
- **P17 — Papéis privilegiados.** `pharmacist` e `pharmacy_assistant` entram em `PRIVILEGED_ROLES` (step-up para conceder; o mantenedor não concede); `stock_keeper` não.
- **P18 — Vínculo ao local.** Vincular exige a `Membership` ativa do papel (`role_not_granted`) e, para `pharmacist`, vínculo profissional ativo com CBO `2234*` (`cbo_not_allowed`); papel efetivo = `Membership` ativa **e** vínculo ativo. `GET admin/locations` traz `staff: [ { user_id, name, role } ]`.
