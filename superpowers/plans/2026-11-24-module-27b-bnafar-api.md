# Módulo 27b — Envio à BNAFAR e saldo inicial (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: api em `origin/main` = `ca2a92f0`** (conferido ao escrever este plano; o 35b já está na main). Última migração de cidade `20261123900003_add_zoonoses_care.rb` (`db/city_schema.rb:13`, `define(version: 2026_11_23_900003)`); última de plataforma `20261113900001_create_field_maps.rb` (`db/platform_schema.rb:13`). Nomes conferidos no código:
> - **Estoque:** `Pharmacy::RecordEntry.call(location:, kind:, document:, items:, by:, catalog:, now:)` (linhas por domínio; `Row`; `mismatch_problems` = "lote gravado é imutável", `record_entry.rb:397`; `stock_receipt.recorded` com `user_id: by.id`, `:164`); `Pharmacy::Ledger.{lock_lots!, lock_location!, record!(location:, lot:, kind:, quantity:, by:, document:, reason:, package:, at:), balance, balances}` e `Ledger::DOCUMENTS` (`ledger.rb:39-44`); `Pharmacy::Catalog::{MOVEMENT_KINDS, POSITIVE_KINDS, NEGATIVE_KINDS, MAX_ENTRY_ITEMS = 100, MAX_TRANSFER_ITEMS, LOT_NUMBER, REASON = (10..500), STOCK_LOCATION_KINDS}`; `Stock::Domain::{MEDICATION, IMMUNOBIOLOGICAL, VECTOR_CONTROL, for, of_lot, usable?, feature_for}`; `StockGate#{require_stock_feature!, stock_feature_for!}`; `Pharmacy::BaseController` (`render_failure`, `render_refusal`, `query_only!`, `page_limit`, `body_param`, `admin?`, `FAILURE_STATUS`); `Pharmacy::Access.{roles_at, allowed?, roles_by_location, central_keeper?}`; `Pharmacy::SaveLocation`; `Vaccination::CreateColdChainCentral` (só criação; `routes.rb:873`). Triggers do 27a: `stock_lots`, `stock_entries`, `stock_openings` só acréscimo (`db/city_triggers.sql:2654`); `ck_stock_movements_{kind,document,sign}` (`city_schema.rb:4601-4603`); `stock_openings` únicos `(local, item)` (`:4613-4615`).
> - **Integração:** `IntegrationCredential::{KINDS = %w[ledi cadsus rnds], SECRET_KEYS, FIELD_MAX}` com `validates :kind, uniqueness: true` (`integration_credential.rb:24`), `ck_integration_credentials_kind` e índice único em `kind` (`city_schema.rb:2041-2043`); `Integrations::{SetCredential.call(kind:, secret:, by:, city:), CheckConnection.call(kind:, city:)}`; `IntegrationsController` (`credentials: KINDS.map`, `:26`); `Platform::Features.city_state` com `IntegrationCredential.pluck(:kind, :last_check_status).to_h` (`features.rb:295`) e `missing_for` (`credential:`, `feature:`, `ibge_code`); `Platform::Features::SIMULATION_ENVS`; `CITIZEN_HIDDEN` (`:239`); o `CATALOG` termina em `zoonoses`.
> - **RNDS (molde):** `Rnds::{Client (CACHE + CACHE_MUTEX, `for(city, env:, settings:)`, `Response` Data com `inspect` próprio), Config (`credentials`, `simulated` por `FAKE_RNDS_URL`, `url_ok?`), Mock::KEY = "rnds_mock", Gate, Alert.counts, Backoff.wait, Errors.sanitize}`; `Rnds::DeliverJob` (`prepend EachCityJob`, `limits_concurrency`); `RndsOutboxEntry.{claim!, release!, release_stale!}` com `FOR UPDATE SKIP LOCKED`; `lib/fake_rnds/{app.rb,config.ru}` e `spec/lib/fake_rnds_spec.rb`; serviço `fake-rnds` (8094) no `docker-compose.yml` da raiz (fora de git). `EachCityJob` roda o corpo dentro de `CityConnection.with` (fuso `cities.time_zone`).
> - **Catálogos:** `MedicationCatalogItem` (plataforma; `obm_code` com `ck_medication_catalog_items_obm_reserved`, `platform_schema.rb:387`); `Medications::CatalogImport` (audita `medication_catalog.imported` no serviço, `catalog_import.rb:73`); `lib/tasks/medications.rake` (`import_anvisa`, `import_catmat`); `Maintenance::Mutations::ImportMedicationCatalog`; `Vaccination::Immunobiologicals::{Product, Manufacturer, find}` por `Vaccination::YamlFiles` (fabricantes sem CNPJ); `vector_products` sem `catmat_code`; `VectorControl::SaveProduct::KEYS`.
> - **Maintenance:** `Maintenance::Mutations::{BaseMutation#audited, CityMutation#{in_city, on_platform}, AUDITED_FIELD_RULES}`; `MaintenanceAudit::NAMES`; `QueryType#city_pec_config(city_slug:)` como molde de consulta por cidade; `Maintenance::GraphqlController::MAX_BODY_BYTES = 64_000` (`graphql_controller.rb:12`); a API de manutenção só é desenhada em development, test e staging (`routes.rb:46-52`).
> - **Guardas:** `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb:15`; fragmento proibido `name`, `:12`); `CityEncryption::CITY_KEYED_TARGETS`; `spec/architecture/{city_encrypted_attributes_guard,current_city_assignment}_spec.rb`; `spec/jobs/reencryption_job_spec.rb`; `config/initializers/{domain_events,filter_parameter_logging}.rb`; helpers `pharmacy_city!`, `pharmacy_admin!`, `central_location!`, `unit_location!`, `pharmacy_staff!`, `losartana`, `stocked_lot!`, `balance_of`, `vaccination_city!`, `cold_chain_central!`, `vaccine_room!`, `vaccine_lot!`, `open_vial!`, `vector_city!`, `vector_product!`, `vector_location!`, `vector_lot!`, `zoonoses_city!`, `ledi_maintainer!`, `create_unit`, `sign_in_as`, `stepped_up!`, `json_post`, `capture_log`.

**Goal:** O lado api do módulo 27b: interruptores e credencial `bnafar` com ambiente, dados BNAFAR do lote, do local e do produto, saída externa, envio diário ao SI-BNAFAR (posição de estoque e saída) com protocolo, correção, exclusão e alerta, simulado `fake_bnafar`, carga OBM e saldo inicial por planilha conferida pela própria cidade.

**Architecture:** Quatro migrações de cidade irreversíveis (`20261125900001` dados do estoque e credencial; `900002` saída externa e desfazer da abertura; `900003` envio; `900004` importação) e uma de plataforma (`20261125600001`, OBM). O domínio mora em `Bnafar::*`: regras puras (`LocationTypes`, `ExitTypes`, `ProductTypes`, `Units`, `Limits`) e leitores (`Snapshot`, `Exits`, `Pendencies`, `Alert`) recebem fatos já lidos; `Bnafar::Build` grava registros e lotes de envio; `Bnafar::Client` fala com o SI-BNAFAR (ou com o `FakeBnafar::App`). Dois jobs de cidade: `Bnafar::DailyJob` (monta D-1, fecha a janela, purga) e `Bnafar::DeliverJob` (claim `SKIP LOCKED`, envia, consulta o protocolo). A importação é REST na cidade e reaproveita o `Pharmacy::RecordEntry` com o usuário da cidade como autor.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), Solid Queue, RSpec via `bin/test`, WebMock `to_rack`, Ruby `csv`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-24-module-27b-bnafar-design.md` (decisões 1–14, com a 6 revista pela Q11; §4, §6). Contrato entre apps — **fonte única**: `docs/superpowers/plans/2026-11-24-module-27b-bnafar-contracts.md` (§1–§11, Q1–Q18; docs `ae1cfbc`). Pesquisa: `docs/pesquisa/2026-11-24-horus-e-si-bnafar-27b.md`.

## Reconciliação (§11, Q6–Q18)

As divergências D1–D22 foram decididas no §11 do contrato; as tasks já seguem as decisões. Onde a decisão difere da proposta ou acrescenta algo:
- **Q7:** o guard do lote aceita só o complemento (de nulo a valor) e a troca do valor unitário; por isso programa e IUM já gravados que divergem numa entrada nova também voltam `lot_mismatch` (consequência do guard, não só CNPJ/ANVISA/AMPP). O lote acima de 30 caracteres é a pendência nova `lot_number_too_long` (Tasks 3, 5, 10).
- **Q9:** em produção o ambiente é sempre `production`, e a credencial `homologation` só se grava fora de produção; `GET /integrations` mostra só a `bnafar` do ambiente em uso, e o `check` testa essa (Task 8).
- **Q11 (decisão 6 revista):** a importação vai para REST na cidade (`/pharmacy/opening_imports`, `municipal_admin` ou `stock_keeper` do central, step-up no apply e no revert); o autor é o usuário da cidade. Saem a autoria do mantenedor (Q3), as mutations de importação e os nomes `maintenance.opening_import.*` do §9 (Tasks 9, 14, 15). O limite de corpo do maintenance só muda para o `importObmCatalog` fora de produção (Task 4).
- **Q12:** em produção a carga OBM é a rake `medications:import_obm[path]`; a mutation só existe fora de produção; o `vmp_code` sai da coluna CATMAT do arquivo, e o que não casa vai à lista de revisão `obmUnmatched` (Task 4). A rake não tem mantenedor: o serviço audita `obm_catalog.imported` na plataforma, como o `CatalogImport` audita `medication_catalog.imported` (nome novo no R18; D24).
- **Q13:** CNES do central e da rede de frio pelos dois `PATCH`, pelo `municipal_admin` (Task 6).
- **Q14:** estorno de uso direto de saída aceita há menos de 30 dias → `DELETE` do registro original; depois disso, "estorno fora da janela" (Tasks 7, 10, 11).
- **Q15:** o AMPP só é exigido quando o item tem VMP; medicamento sem VMP entra no estoque e o lote fica com `product_without_code`; fabricante internacional pelo `nomeFabricanteInternacional` com o CNPJ nulo (coluna nova `manufacturer_international`; Tasks 3, 5, 10).
- **Q16:** registros e pendências com `location_name` e `item_label`; `protocol` por parte; `GET /pharmacy/bnafar/domains` e `GET /pharmacy/obm_presentations?vmp_code=`; registro perdido com `lost` e `lost_reason`; a saída externa não exige o `bnafar`; o lote antigo começa com `bnafar_version` 0 (Tasks 3, 9, 12, 13).
- **Q17:** `path` das recusas no formato do api, `items[i].campo`; os códigos de linha da planilha são os do Valor 11.
- **Q18:** o 27b herda o gate de go-live do maintenance em produção; não cria outro.
- **Pendente (D23, nova):** o §6 exige o `bnafar` em toda rota de cidade, e a Q11 não diz se a importação entra nessa regra. O plano a deixa **sem** o gate do BNAFAR (a abertura vem antes do primeiro envio, decisão 14), como a saída externa (Q16).

## Valores fixados por este plano

1. **Interruptores** (fim do `CATALOG`, depois de `zoonoses`, nesta ordem):
   - `bnafar`: `description: "Envio diário da posição de estoque e das saídas ao SI-BNAFAR"`, `requires: %w[credential:bnafar ibge_code]`;
   - `bnafar_medications`: `"BNAFAR: medicamentos da farmácia"`, `%w[feature:bnafar feature:pharmacy]`;
   - `bnafar_vector_supplies`: `"BNAFAR: produtos de endemias e zoonoses"`, `%w[feature:bnafar vector_supplies]`; requisito novo `vector_supplies` no `missing_for` → `"vector_supplies_disabled"` salvo `usable?(city, "vector_control") || usable?(city, "zoonoses")`;
   - `bnafar_vaccines`: `"BNAFAR: vacinas e rede de frio"`, `%w[feature:bnafar feature:vaccination]`;
   - `bnafar_mock` (Q6): `"BNAFAR simulado — envio sem validade"`, `%w[feature:bnafar]`, `environments: SIMULATION_ENVS`.
   - `CITIZEN_HIDDEN` + as quatro primeiras (o `bnafar_mock` fica fora, como o `rnds_mock`).
2. **`Bnafar::Gate`:** `KEY = "bnafar"`; `DOMAIN_KEYS = { Stock::Domain::MEDICATION => "bnafar_medications", Stock::Domain::VECTOR_CONTROL => "bnafar_vector_supplies", Stock::Domain::IMMUNOBIOLOGICAL => "bnafar_vaccines" }`; `usable?(city)`; `domains(city) -> [String]` (chave do domínio **utilizável**, Q6, com `bnafar` utilizável); `domains_through(city, day) -> [String]` (os de `domains(city)` cuja linha `city_features` de `bnafar` e do domínio tem `changed_at < (day + 1).in_time_zone(city.time_zone).beginning_of_day`: ligado no fim de D-1, §1); `required_for?(city, location) -> Boolean` = `domains(city).include?(Stock::Domain.for(location))`. Interruptor lido fora da transação da cidade e passado como argumento (`bnafar_required:`, `domains:`).
3. **`Bnafar::Catalog`:** `ENVIRONMENTS = %w[homologation production]`; `SUBJECTS = %w[position exit]`; `SUBMISSION_STATUSES = %w[building queued sending processing finished failed lost]`; `BATCH_STATUSES = %w[queued sending processing finished failed]`; `BATCH_OPERATIONS = %w[post put delete]` (Q8, Q14); `RECORD_STATUSES = %w[pending accepted rejected]` (exclusão em `deleted_at`); `LINK_KINDS = %w[exit deletion late_reversal]`; `PENDENCY_CODES = %w[lot_incomplete lot_number_too_long location_without_cnes product_without_code fractional_balance exit_type_missing domain_off]`; `EXTERNAL_EXIT_TYPES = %w[S-DF S-TR S-D S-E]`; `DESTINATION_KINDS = %w[cnes cnpj]`; `REASON = (10..500)`; `CNES = /\A\d{7}\z/`; `ANVISA = /\A(\d{11}|\d{13})\z/`; `AMPP = /\A[@0-9A-Za-z]{1,25}\z/`; `IUM = /\A[0-9A-Za-z]{1,20}\z/` (até 100 por lote); `LOT_NUMBER_MAX = 30`; `INTERNATIONAL_NAME_MAX = 120` (a coluna `manufacturer` do lote; o manual aceita 200); `UNIT_PRICE_CENTS = (1..2_000_000_000)`; `MSG_CODE = /\A[A-Za-z0-9_-]{2,20}\z/`; `BUILD_HOUR = 1`; `LATE_HOUR = 15`; `CORRECTION_DAYS = 30`; `PAYLOAD_DAYS = 30`; `DOMAIN_EVENTS` = os nove do contrato §9, na ordem, mais `bnafar_record.deleted` (Q14).
4. **`Bnafar::Limits`:** `RECORDS_PER_BATCH = 1_000`; `ITEMS_PER_RECORD = 20` (a MSG46 diz 20 e o §10.3 diz 60; parte em 20 até a homologação confirmar); `INCONSISTENCY_PAGE = 100`; `TOKEN_MARGIN = 120` (s); `MS_THRESHOLD = 86_400` (Q5); `STALE_SENDING = 10.minutes`; `BACKOFF_CAP = 30.minutes` (`[1.minute * 2**(n-1), cap].min`).
5. **Mapeamentos (`config/bnafar/domains.yml`, com spec):**
   - `Bnafar::LocationTypes` (DM01): `central` → `A`, `cold_chain_central` → `A`, `unit` → `F`, `vaccine_room` → `F`, `vector_control` → `F`.
   - `Bnafar::ExitTypes` (DM12): `transfer_out` → `S-T`; `loss` → `S-PE`, salvo `stock_adjustments.loss_code = "expired"` → `S-VV`; `expiry` → `S-VV`; `adjustment` negativo → `S-AE`; `external_exit` → o `exit_type`; uso direto `administration`, `vial_opening`, `immunization`, `vector_application`, `animal_vaccination`, `collar_placement` → `S-CA` (`review: true`, a conferir na homologação; `S-PA` fora); `IGNORED = %w[dispensing opening_reversal]`; positivo nunca vai; os três estornos de uso direto (`*_reversal`) disparam a regra da Q14. Tipo desconhecido → `exit_type_missing`.
   - Destino (Q13): `S-T` → CNES do local de destino; saída externa → o destino informado; o resto → o CNES do próprio estabelecimento.
   - `Bnafar::ProductTypes` (DM13 por DM14): `config/bnafar/programs.yml` com as siglas da DM14 (manual v1.25 §15), `{ code, label, product_type }`, `review: true`. Amostra fixada: `AFB`, `AF`, `HIPERTEN`, `DIABETES` → `B`; `ESP`, `ALZHE`, `GLAUC` → `E`; `TB`, `HANS`, `IMUN`, `ZOO`, `END`, `DENGUE`, `LEISH` → `S`; `JUD`, `HOSP`, `UPA` → `O`.
   - `Bnafar::Units`: fator da menor unidade de fornecimento — medicamento `1` (o lote conta em `dispensing_unit`); vetorial `1` (g, ml ou un); imunobiológico `doses_per_vial` (o lote conta doses; fornece-se o frasco). `to_bnafar(lot, quantity) -> Integer | nil` (nil → `fractional_balance`); `from_bnafar(lot_or_product, Integer)` na planilha.
6. **Dia de referência (R):** a posição de R soma os movimentos com `created_at < (R + 1).beginning_of_day` no fuso da cidade; a saída de R é o movimento **criado** em R (o `recorded_at` pode ser retroativo: `Ledger.record!(at:)`, `ledger.rb:91`). O job roda em R+1; o reenvio vai até o fim de R+1; às 00:00 de R+2 a remessa vira `lost`.
7. **Payload:** `estabelecimento { cnes, tipo }`; posição `caracterizacao { codigoOrigem, dataPosicaoEstoque }`; saída `caracterizacao { codigoOrigem, dataSaida, estabelecimentoDestino, tipoSaida }`. Item (Q15): todas as chaves sempre — `codigoOrigem` (`stock_lot_id` na posição, `stock_movement_id` na saída), `numero` (CATMAT no medicamento e no vetorial; `""` na vacina), `terminologia` (`OBM` no medicamento e na vacina; `CATMAT` no vetorial), `codigoAmpp` (do lote ou `null`), `codigoVmp` (catálogo ou YAML), `registroAnvisa`, `tipoProduto`, `lote`, `dataValidade`, `cnpjFabricante` (ou `null` no internacional), `nomeFabricanteInternacional` (o `manufacturer` do lote no internacional, senão `null`), `quantidade` (inteiro), `valorUnitario` (2 casas, ponto), `siglaProgramaSaude`, `iums: [{ ium }]` (`[]` sem IUM).
8. **Erros novos (422 salvo nota):** `required`, `invalid_environment`, `environment_not_allowed`, `ampp_unknown`, `ampp_not_in_vmp`, `anvisa_mismatch`, `program_unknown`, `invalid_cnpj`, `cnes_in_use` (409), `lot_bnafar_changed` (409), `external_exit_changed` (409), `retry_window_closed` (409), `nothing_to_retry` (409), `correction_window_closed` (409), `not_correctable` (409), `opening_import_changed` (409), `revert_blocked` (409), `invalid_file`, `file_too_large` (413). `path` no formato `items[i].campo` (Q17). Os da planilha no Valor 11.
9. **Rotas novas** (sessão do dashboard). Com o gate do BNAFAR (`Pharmacy::Bnafar::BaseController < Pharmacy::BaseController`, `skip_before_action :require_pharmacy!`, `before_action :require_bnafar!` → 403 `{ error: "feature_disabled", feature: "bnafar" }`; dentro de `Pharmacy::` o domínio é sempre `::Bnafar::…`): `GET /pharmacy/bnafar/status`, `GET /pharmacy/bnafar/submissions`, `GET /pharmacy/bnafar/submissions/:id`, `POST /pharmacy/bnafar/submissions/:id/retry`, `POST /pharmacy/bnafar/records/:id/correct`, `GET /pharmacy/bnafar/pendencies`. Só com o estoque do domínio (`require_stock_feature!`, Q16 e D23): `GET /pharmacy/bnafar/domains`, `GET /pharmacy/obm_presentations?vmp_code=`, `POST|GET /pharmacy/external_exits`, `GET /pharmacy/external_exits/:id`, `PATCH /pharmacy/lots/:id/bnafar`, `GET /pharmacy/openings`, `GET /pharmacy/opening_imports/template.csv`, `POST /pharmacy/opening_imports`, `GET /pharmacy/opening_imports/:id`, `POST /pharmacy/opening_imports/:id/{apply,revert,discard}`. Administração: `cnes` no `PATCH /pharmacy/admin/locations/:id` e `PATCH /vaccination/admin/cold_chain_central`. Listas `{ items, next_before }`, padrão 50, máximo 100.
10. **Ambiente e credencial (Q9):** `IntegrationCredential::ENVIRONMENTS = %w[homologation production]`; CHECK `kind = 'bnafar' OR environment = 'production'`; único `(kind, environment)`; `SECRET_KEYS["bnafar"] = %w[password username]`. Ambiente em uso: `Bnafar::Environment.current(env: Rails.env)` = `production` em produção; fora dela, `BnafarSetting.current.environment`. `homologation` só se grava (credencial ou ambiente) com `Rails.env` em `SIMULATION_ENVS`; senão `environment_not_allowed`. URLs em `Rails.application.credentials.bnafar.{homologation,production}.{auth_url,api_url}` (manual §8); simulado `FAKE_BNAFAR_URL` (`#{base}/jwtauth/auth`, `#{base}/bnafar`) com `bnafar_mock` ligado.
11. **Planilha:** cabeçalho exato do §5; UTF-8 (BOM aceito; Latin-1 recusado inteiro: `invalid_file`), `;`, CRLF ou LF, aspas RFC 4180; até 5 MB decodificados (`file_too_large`) e 20.000 linhas; `valor_unitario` com vírgula, até 2 casas, ponto de milhar aceito; `quantidade` na unidade BNAFAR (frasco na vacina), por `Units.from_bnafar`. Erros por linha: `cnes_unknown`, `product_unknown`, `invalid_product_type`, `item_not_allowed_here`, `invalid_lot_number`, `invalid_date`, `expired_lot`, `invalid_quantity`, `invalid_price`, `invalid_cnpj`, `invalid_anvisa`, `ampp_unknown`, `anvisa_mismatch`, `program_unknown`, `manufacturer_not_in_catalog`, `duplicate_line`, `opening_exists`, `too_many_lots` (mais de 100 lotes de um item num local).
12. **Dev:** worktree `apps/api/.claude/mod27b` (com `config/master.key`), branch `feat/mod-27b-bnafar`, api do worktree na porta **3062**; simulado `fake-bnafar-dev` na **8095**.

## Global Constraints

- git `/opt/homebrew/bin/git`; nunca `git add -A` nem `git stash` sem nome. Commits em inglês, Conventional Commits com o tipo por extenso: `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge, push e cards só com autorização do usuário.
- **Testes só por `bin/test api -w mod27b <specs>`**, a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`; nunca `rspec` direto nem o Postgres do host; `--keep-db` só em rodada de arquivo isolado. A branch é transversal (migração de cidade e de plataforma, `CityEncryption`, `Platform::Features`, `IntegrationCredential`, estoque do 27a/29/32b/35a/35b, catálogo da plataforma, YAML de imunobiológicos, `spec/support`, initializers, GraphQL do maintenance) → **suíte completa obrigatória antes do merge, pedida à sessão "Testes" na Task 18**.
- Migrações: dump à mão em `db/city_schema.rb` e `db/platform_schema.rb`; triggers num bloco "Módulo 27b" ao fim do `db/city_triggers.sql`, cada um dentro de `IF to_regclass('public.<tabela>') IS NOT NULL`, coluna nova lida por `to_jsonb` ou conferida em `information_schema.columns` onde o arquivo roda em migrações antigas; **irreversíveis** (`down` → `ActiveRecord::IrreversibleMigration`; o `up` de cidade termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`); CHECK trocado por `replace_check`. Banco de dev só migrado e semeado na prova, com autorização do usuário.
- Specs com tipo explícito (`type: :request`, `:model`, `:job`). **Nenhuma spec bate em URL real**: tudo por `stub_request(...).to_rack(FakeBnafar::App.new)` em `http://fake-bnafar.test:8095`.
- **Nunca `Current.city =` solto**; interruptores fora da transação da cidade. `spec/architecture/current_city_assignment_spec.rb` roda nas Tasks 10, 11 e 18.
- **Sigilo (§9):** usuário, senha e token só em memória do cliente e no segredo cifrado; nunca em log, evento, URL, cURL, `Result`, `inspect` ou resposta. Nenhum dado de pessoa vai à BNAFAR. `Data`/`Struct`/modelo com payload, credencial ou CSV sobrescreve `inspect`/`to_s`/`pretty_print`/`as_json`. Texto da BNAFAR nunca é gravado, só o código (`MSG_CODE`). `filter_parameter_logging` ganha `file_base64` e `fileBase64`.
- **R18:** entram em `R18_PLATFORM_EVENT_NAMES` `bnafar.credential_set`, `obm_catalog.imported`, `maintenance.obm_catalog.imported` e `maintenance.bnafar.environment_changed`; os dois `maintenance.*` também em `MaintenanceAudit::NAMES`. Campo auditado nunca contém `name` nem texto do cliente; campo novo de mutation de cidade ganha regra em `CityMutation::AUDITED_FIELD_RULES`.
- Eventos em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`, só ids; `DomainEvents.publish(` é multilinha.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` e em `spec/jobs/reencryption_job_spec.rb`; guarda de coluna cifrada aceita a re-cifra (`rota.reencrypting`) primeiro.
- Concorrência otimista: escrita sobre linha mutável leva `expected_version` (ausente → `422 expected_version_required`); diferente → o `<entidade>_changed`.
- Todo "hoje" é `Time.zone.today` dentro de `CityConnection.with`.
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`.

## Review Focus

1. **Movimento com `recorded_at` retroativo** (app de campo offline sincronizado em R+1 com a aplicação de R): a saída vai no dia em que o movimento foi **criado**, nunca num dia já enviado, e a posição de R usa o mesmo corte por `created_at`; nada some nem sai duas vezes. Teste na Task 10.
2. **Dois `DailyJob` ao mesmo tempo, ou um que morre no meio da montagem**: o único `(reference_on, subject, environment)` e a retomada do `building` fazem nascer uma remessa só, sem registro repetido nem movimento ligado duas vezes. Teste na Task 11.
3. **Lote complementado depois da recusa** (CNPJ errado → MSG12 em R+1 de manhã): o reenvio de R+1 remonta o registro com o dado novo e o `origin_code` de sempre; às 00:00 de R+2 o que sobrou vira `lost` com `lost_reason`, e a fotografia de R+1 já sai certa. Teste na Task 12.
4. **Estorno da dose no dia seguinte e 31 dias depois**: o primeiro gera um `DELETE` do registro aceito (e o registro sai `deleted_at`); o segundo não chama a BNAFAR e fica `late_reversal`; o estorno no mesmo dia compensa antes do envio. Teste na Task 11.
5. **Planilha "de Excel"** (BOM, CRLF, `;` dentro de aspas, `1.234,56`, linha em branco no fim, linha repetida, arquivo em Latin-1): as quatro primeiras passam; a repetida é `duplicate_line`; o Latin-1 é recusado inteiro, sem área de conferência; nada mexe no estoque antes do `apply`. Teste na Task 14.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod27b/`)
- Migrações: `db/platform_migrate/20261125600001_add_obm_presentations.rb`; `db/city_migrate/{20261125900001_add_bnafar_stock_data,20261125900002_add_stock_external_exits,20261125900003_add_bnafar_submissions,20261125900004_add_opening_imports}.rb`.
- Modelos: `app/models/{obm_presentation,bnafar_setting,bnafar_submission,bnafar_batch,bnafar_record,bnafar_movement_link,stock_external_exit,stock_external_exit_item,opening_import,opening_import_row}.rb`.
- `app/services/bnafar/{gate,catalog,limits,domains,location_types,exit_types,product_types,units,environment,config,mock,client,errors,evidence,homologation,snapshot,exits,pendencies,payload,build,delivery,window,alert,json,lot_fields}.rb`; `app/services/medications/obm_import.rb`; `app/services/opening_imports/{reader,validator,report}.rb`.
- `app/commands/bnafar/{complete_lot,retry_submission,correct_exit,set_environment}.rb`; `app/commands/pharmacy/record_external_exit.rb`; `app/commands/opening_imports/{upload,apply,discard,revert}.rb`.
- `app/jobs/bnafar/{daily_job,deliver_job}.rb`.
- `app/controllers/concerns/bnafar_gate.rb`; `app/controllers/pharmacy/bnafar/{base,status,submissions,records,pendencies,domains}_controller.rb`; `app/controllers/pharmacy/{external_exits,lot_complements,openings,opening_imports,obm_presentations}_controller.rb`; `app/controllers/vaccination/admin/cold_chain_central_controller.rb` (ação `update`).
- GraphQL: `app/graphql/maintenance/mutations/{import_obm_catalog,set_bnafar_environment}.rb`; `app/graphql/maintenance/types/{city_bnafar_status_type,obm_import_result_type,obm_presentation_type}.rb`.
- `lib/tasks/medications.rake` (task `import_obm`); `config/bnafar/{domains,programs,homologation}.yml`; `config/medications/obm_columns.yml`; `lib/fake_bnafar/{app.rb,config.ru}`; `lib/bnafar_crew.rb`; `spec/support/bnafar_helpers.rb`; `spec/security/bnafar_secrecy_spec.rb`; specs listados em cada task.

**Modificar**
- Plataforma e infra: `app/services/platform/features.rb`, `app/models/{integration_credential,medication_catalog_item,stock_lot,stock_location,stock_movement,stock_opening,vector_product}.rb`, `app/services/city_encryption.rb`, `db/{city_schema,platform_schema}.rb`, `db/{city,platform}_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `app/events/maintenance_audit.rb`, `db/seeds.rb`, `docker-compose.yml` da raiz (fora de git).
- Estoque: `app/commands/pharmacy/{record_entry,save_location}.rb`, `app/services/pharmacy/{catalog,ledger,json}.rb`, `app/commands/vaccination/create_cold_chain_central.rb`, `app/services/vaccination/immunobiologicals.rb`, `config/vaccination/immunobiologicals.yml`, `app/commands/vector_control/save_product.rb`, `app/services/vector_control/json.rb`.
- Integração: `app/commands/integrations/{set_credential,check_connection}.rb`, `app/controllers/integrations_controller.rb`.
- Maintenance: `app/controllers/maintenance/graphql_controller.rb`, `app/graphql/maintenance/types/{mutation_type,query_type,medication_catalog_type,medication_catalog_item_type}.rb`.
- Specs existentes: `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/{city_features,medication_catalog}_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`, `spec/models/{pharmacy_schema,medication_catalog}_spec.rb`, `spec/commands/pharmacy/record_entry_spec.rb`, `spec/requests/pharmacy/{entries,admin,positions}_spec.rb`, `spec/requests/integrations_spec.rb`, `spec/services/medications/catalog_import_spec.rb`, `spec/services/vaccination/catalog_files_spec.rb`, `spec/services/city_schema_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`.

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod27b -b feat/mod-27b-bnafar origin/main && cp config/master.key .claude/mod27b/config/master.key`.
- [ ] **Step 2:** conferir no worktree, com `grep`/`ls`: `define(version: 2026_11_23_900003)` e `ls db/city_migrate | tail -1`; `define(version: 2026_11_13_900001)` e `ls db/platform_migrate | tail -1`; `key: "zoonoses"` como última `Entry`; `CITIZEN_HIDDEN =`; `KINDS = %w[ledi cadsus rnds]`; `ck_medication_catalog_items_obm_reserved`; `MAX_BODY_BYTES = 64_000`; `'stock_lots', 'stock_entries', 'stock_openings'` no `city_triggers.sql`; e que `bnafar`, `obm_presentations`, `stock_external_exits`, `opening_imports` não existem. Migração mais nova que a do plano → os números sobem para depois dela (nota no topo antes da Task 1).
- [ ] **Step 3:** `bin/test api -w mod27b spec/services/platform/features_spec.rb spec/commands/pharmacy spec/requests/pharmacy spec/requests/integrations_spec.rb spec/services/rnds spec/jobs/rnds spec/lib/fake_rnds_spec.rb spec/requests/maintenance/medication_catalog_spec.rb spec/services/medications spec/models/pharmacy_schema_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb` → PASS (linha de base).
- [ ] **Step 4 (etapa anotada; só com autorização do usuário):** criar no board #1 os cards do 27b pela receita da memória `board-91-features-roadmap` (Module `mod-27`, Work type `feature`), um `gh project item-create …` por chamada, cada comando começando com `gh`: F-27.8 "Interruptores e credencial BNAFAR"; F-27.9 "Dados BNAFAR do lote, do local e do produto (OBM, CATMAT, vacina)"; F-27.10 "Saída externa"; F-27.11 "Envio diário ao SI-BNAFAR e simulado"; F-27.12 "Situação, pendências, reenvio, correção, exclusão e alerta"; F-27.13 "Saldo inicial por planilha". Sem autorização: anotar no relato e seguir.

**Relato ao coordenador:** "api 27b · Task 0 · worktree mod27b em origin/main <hash>; migrações conferidas (cidade <última>, plataforma <última>); cards <criados | aguardando autorização>; linha de base verde."

---

### Task 1: Domínios e mapeamentos (DM01, DM12, DM13/DM14, unidades, limites)

**Files:**
- Create: `config/bnafar/{domains,programs}.yml`, `app/services/bnafar/{catalog,limits,domains,location_types,exit_types,product_types,units}.rb`, `spec/services/bnafar/mappings_spec.rb`

**Interfaces:**
- Produces: Valores fixados 3–5 e 7; `Bnafar::Domains.{dm01, dm12, dm13, programs}` (congelados); `Bnafar::LocationTypes.for(location) -> "A" | "F"`; `Bnafar::ExitTypes.for(movement, adjustment: nil, external_exit: nil) -> String | :ignored | :reversal | nil`; `Bnafar::ProductTypes.for(program_code) -> "B" | "E" | "S" | "O" | nil`; `Bnafar::Units.{factor(lot_or_product), to_bnafar(lot, quantity) -> Integer | nil, from_bnafar(lot_or_product, Integer) -> BigDecimal}`.

- [ ] **Step 1: Write the failing test** (`type: :model`): "DM01: os cinco tipos de local mapeados"; "DM12: cada `NEGATIVE_KINDS` tem tipo ou `:ignored`; os três `*_reversal` → `:reversal`; `loss` com `loss_code expired` → `S-VV`; `adjustment` +5 → nil e −5 → `S-AE`; `external_exit` → o `exit_type`; nenhum código fora dos 21 da DM12; nenhum `S-PA`"; "DM14: `INSULDEP I`, `S/DOR`, `URG/EMERG`, `SM CVD-19` preservam espaço e barra; toda sigla tem `product_type`; a amostra do Valor 5"; "Units: vacina de 10 doses por frasco 30 → 3 e 25 → nil; medicamento 12.00 → 12 e 12.50 → nil; vetorial 500.00 → 500"; "`ITEMS_PER_RECORD == 20` e `RECORDS_PER_BATCH == 1000`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/bnafar/mappings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** os dois YAML (DM12 com as descrições do manual v1.25 §15; `programs.yml` com a DM14 inteira) e os módulos puros. O `loss_code` vem da `StockAdjustment` do movimento, lida pelo chamador.
- [ ] **Step 4:** o mesmo spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add bnafar domain tables and stock mappings" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 1 · DM01/DM12/DM13 no código com spec (uso direto → S-CA a conferir; estornos → regra da Q14); vacina em frasco; ITEMS_PER_RECORD 20."

---

### Task 2: Interruptores e gate

**Files:**
- Create: `app/services/bnafar/gate.rb`, `app/controllers/concerns/bnafar_gate.rb`, `spec/services/bnafar/gate_spec.rb`
- Modify: `app/services/platform/features.rb`, `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`

**Interfaces:**
- Produces: Valores fixados 1–2; `BnafarGate#require_bnafar!`. O `city_state` com ambiente é da Task 8.

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: índices contados do fim sobem cinco; "`bnafar`, `bnafar_medications`, `bnafar_vector_supplies`, `bnafar_vaccines` e `bnafar_mock` fecham o catálogo, nessa ordem, com os `requires` do Valor 1"; "`bnafar_mock` some de `catalog(env: "production")`"; "as quatro em `CITIZEN_HIDDEN`, o `bnafar_mock` não"; "`bnafar_vector_supplies` sem `vector_control` nem `zoonoses` → `vector_supplies_disabled`; com só `zoonoses` → vazio"; "`bnafar` sem credencial → `credential_missing:bnafar`; sem IBGE → `ibge_code_missing`". Listas exatas de `city_features_spec.rb` com as cinco chaves; `session_features_spec.rb`: "GET /citizen/session nunca traz as chaves bnafar". `gate_spec.rb`: "`domains` com `bnafar` desligado → []"; "domínio ligado e inutilizável (credencial recusada) → fora (Q6)"; "`domains_through(city, R)`: ligado em R às 23:00 → dentro; em R+1 às 00:30 → fora"; "`required_for?` num `vaccine_room` só com `bnafar_vaccines`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/platform/features_spec.rb spec/services/bnafar/gate_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/citizen_api/session_features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** as cinco `Entry` (comentário: "27b: sem cascata; desligar não apaga o que foi enviado; ligar vale a partir do dia seguinte"), o requisito `vector_supplies`, o `Bnafar::Gate` (`domains_through` lê `CityFeature.where(city_id:, key:).pluck(:key, :changed_at)` uma vez) e o concern.
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add bnafar feature switches and gate" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 2 · cinco interruptores no fim do CATALOG (bnafar_mock só fora de produção, Q6), vector_supplies_disabled, domains_through pelo changed_at."

---

### Task 3: Migração 900001 — dados BNAFAR do lote, CNES do local, CATMAT do produto e ambiente da credencial

**Files:**
- Create: `db/city_migrate/20261125900001_add_bnafar_stock_data.rb`, `spec/models/bnafar_stock_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{stock_lot,stock_location,vector_product,integration_credential}.rb`, `spec/services/city_schema_spec.rb`

**Interfaces:**
- `stock_lots` + `unit_price_cents integer CHECK (> 0)`, `manufacturer_cnpj varchar(14) CHECK (~ '^[0-9A-Z]{12}[0-9]{2}$')` (Q7), `manufacturer_international boolean NOT NULL DEFAULT false` com `CHECK (NOT manufacturer_international OR (manufacturer_cnpj IS NULL AND manufacturer IS NOT NULL))` (Q15), `anvisa_registration varchar(13) CHECK (~ '^([0-9]{11}|[0-9]{13})$')`, `health_program varchar(10)` (Q7), `ium text[]`, `ampp_code varchar(25)` (Q7; `CHECK (ampp_code IS NULL OR item_kind <> 'vector_control')`), `bnafar_version integer NOT NULL DEFAULT 0 CHECK (>= 0)` (Q16: o lote antigo começa em 0).
- Trigger (Q7): no bloco "Módulo 27b — lote", só quando a coluna `stock_lots.bnafar_version` existe (`information_schema.columns`), `DROP TRIGGER stock_lots_append_only` e cria `stock_lots_bnafar_guard`: DELETE recusado; UPDATE com `NEW.bnafar_version = OLD.bnafar_version + 1` e, nas colunas BNAFAR, só de NULL a valor (`ium` de NULL a lista; `manufacturer_international` de false a true com o CNPJ nulo), salvo `unit_price_cents`, que pode trocar; o resto do lote nunca muda.
- `stock_locations` + `cnes varchar(7) CHECK (~ '^[0-9]{7}$')` e `CHECK (cnes IS NULL OR kind IN ('central','cold_chain_central'))`.
- `vector_products` + `catmat_code integer CHECK (> 0)`.
- `integration_credentials` + `environment varchar NOT NULL DEFAULT 'production'`, `CHECK (environment IN ('homologation','production'))`, `CHECK (kind = 'bnafar' OR environment = 'production')`; `ck_integration_credentials_kind` aceita `bnafar`; único `(kind, environment)` no lugar de `index_integration_credentials_on_kind`.
- Modelos: `IntegrationCredential::{KINDS + bnafar, ENVIRONMENTS, SECRET_KEYS["bnafar"]}`, `validates :kind, uniqueness: { scope: :environment }`; `StockLot::BNAFAR_COLUMNS` e `#bnafar_fields`; `StockLocation#bnafar_cnes` (o próprio nos dois centrais; senão `health_unit&.cnes`).

- [ ] **Step 1: Write the failing test** (`type: :model`): "CNPJ `12ABC34501DE35` passa e `1234` cai"; "ANVISA de 12 dígitos cai"; "`ampp_code` num lote vetorial cai"; "internacional com CNPJ → CHECK"; "lote antigo nasce com `bnafar_version` 0"; "preencher o CNPJ nulo com versão + 1 passa; trocar um CNPJ preenchido → trigger; trocar o preço preenchido → passa; sem subir a versão → trigger; mudar `expires_on` → trigger; DELETE → trigger"; "CNES num `unit` → CHECK"; "duas `bnafar` (uma por ambiente) passam; `rnds` em `homologation` → CHECK; duas `bnafar production` → único"; "as credenciais antigas ficam `production`"; "`bnafar_cnes` do `unit` é o da unidade".
- [ ] **Step 2:** `bin/test api -w mod27b spec/models/bnafar_stock_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`AddBnafarStockData`), o dump (`define(version: 2026_11_25_900001)`), o bloco de trigger e os modelos.
- [ ] **Step 4:** o mesmo spec + `spec/models/pharmacy_schema_spec.rb spec/services/city_schema_spec.rb spec/commands/pharmacy/record_entry_spec.rb spec/requests/integrations_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add bnafar lot, location, product and credential columns" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 3 · 20261125900001: colunas BNAFAR no lote (bnafar_version 0, guard só de complemento e preço, Q7/Q16; fabricante internacional, Q15), CNES nos centrais, CATMAT no vetorial, credencial com ambiente."

---

### Task 4: Plataforma — VMP no catálogo, `obm_presentations`, rake e `importObmCatalog`

**Files:**
- Create: `db/platform_migrate/20261125600001_add_obm_presentations.rb`, `app/models/obm_presentation.rb`, `app/services/medications/obm_import.rb`, `config/medications/obm_columns.yml`, `app/graphql/maintenance/mutations/import_obm_catalog.rb`, `app/graphql/maintenance/types/{obm_import_result_type,obm_presentation_type}.rb`, `spec/services/medications/obm_import_spec.rb`, `spec/requests/maintenance/obm_catalog_spec.rb`, `spec/tasks/medications_import_obm_spec.rb`, `spec/fixtures/obm/sample.csv`
- Modify: `db/platform_schema.rb`, `db/platform_triggers.sql`, `app/models/medication_catalog_item.rb`, `lib/tasks/medications.rake`, `app/graphql/maintenance/types/{mutation_type,medication_catalog_type,medication_catalog_item_type}.rb`, `app/controllers/maintenance/graphql_controller.rb`, `app/events/maintenance_audit.rb`, `spec/models/medication_catalog_spec.rb`, `spec/services/medications/catalog_import_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`

**Interfaces:**
- Migração: `rename_column :medication_catalog_items, :obm_code, :vmp_code`; sai `ck_medication_catalog_items_obm_reserved`; entra `CHECK (vmp_code IS NULL OR vmp_code ~ '^[@0-9A-Za-z]{1,25}$')` e índice em `vmp_code`. `obm_presentations`: `ampp_code` (único), `vmp_code` (NOT NULL), `catmat_code integer`, `anvisa_registration`, `manufacturer_cnpj`, `description` (NOT NULL, até 500), `active` (default true), `imported_at`; sem DELETE (trigger).
- `Medications::ObmImport.call(csv:, by:, now:) -> Result.ok(added:, changed:, deactivated:, vmp_linked:, unmatched:) | fail(:invalid_file | :file_too_large | :suspicious_drop, details: { path: })` (Q12): CSV pelo mapa de cabeçalhos de `obm_columns.yml` (`ampp`, `vmp`, `catmat`, `registro_anvisa`, `cnpj_fabricante`, `descricao`); upsert por `ampp_code`; AMPP ausente da fonte → `active: false`; `medication_catalog_items.vmp_code` pela coluna CATMAT (fonte sem o par não apaga o gravado); apresentação cujo CATMAT não casa com item ativo conta em `unmatched` e fica na lista de revisão; menos da metade dos ativos → `suspicious_drop`; `Platform.audit("obm_catalog.imported", added:, changed:, deactivated:, unmatched:)`.
- Rake `medications:import_obm[path]` (produção, operador da plataforma): lê o arquivo do caminho, `by: "rake"`, imprime as contagens.
- `importObmCatalog(fileBase64: String!, code: String!)` (`BaseMutation`, step-up de TOTP; o campo só existe com `Rails.env` em `SIMULATION_ENVS` — fora disso, ausente do schema): `audited(event: "maintenance.obm_catalog.imported", module_name: "medications", rows:)`. `MedicationCatalogType` ganha `obmUnmatched(first:)` (a lista de revisão: apresentações sem item do catálogo) e `MedicationCatalogItemType` ganha `vmpCode`.
- `GraphqlController`: `UPLOAD_BODY_BYTES = 7_000_000` só para a operação cujo único campo de raiz é `importObmCatalog` (o analisador recusa outro campo junto); o resto segue em 64 KB.

- [ ] **Step 1: Write the failing tests.** `obm_import_spec.rb` (`type: :model`): "a amostra cria 3 apresentações e liga o VMP a 2 itens pelo CATMAT; 1 fica em `unmatched`"; "reimportar sem uma AMPP a desativa e não apaga"; "fonte sem o par CATMAT não apaga o `vmp_code` gravado"; "cabeçalho faltando → `invalid_file`"; "metade dos ativos some → `suspicious_drop`"; "`Platform.audit("obm_catalog.imported")` só com contagens". `medications_import_obm_spec.rb`: "a rake importa o arquivo do caminho". `obm_catalog_spec.rb` (`type: :request`): "sem TOTP → recusa e auditoria `rejected`"; "com TOTP → `ok`, auditoria com `rows`"; "corpo de 6 MB em `importObmCatalog` é lido; em outra operação → 413"; "`obmUnmatched` lista a apresentação sem item"; "com `Rails.env` production (stub do schema) o campo não existe". `catalog_import_spec.rb`: "a importação do CATMAT não toca `vmp_code`". `medication_catalog_spec.rb`: "vmp_code aceita `@brasil20038375080`". R18 + `obm_catalog.imported` e `maintenance.obm_catalog.imported`.
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/medications spec/tasks/medications_import_obm_spec.rb spec/requests/maintenance/obm_catalog_spec.rb spec/requests/maintenance/medication_catalog_spec.rb spec/models/medication_catalog_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/graphql/maintenance/analyzers_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_25_600001)`; `MaintenanceAudit::NAMES` + `maintenance.obm_catalog.imported`).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: load obm presentations and link vmp codes to the catalog" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 4 · obm_code → vmp_code; obm_presentations com lista de revisão; rake medications:import_obm em produção e importObmCatalog só fora dela (Q12)."

---

### Task 5: Campos BNAFAR na entrada e no complemento do lote

**Files:**
- Create: `app/services/bnafar/lot_fields.rb`, `app/commands/bnafar/complete_lot.rb`, `app/controllers/pharmacy/lot_complements_controller.rb`, `spec/services/bnafar/lot_fields_spec.rb`, `spec/requests/pharmacy/lot_complements_spec.rb`
- Modify: `app/commands/pharmacy/record_entry.rb`, `app/controllers/pharmacy/entries_controller.rb`, `app/services/pharmacy/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/commands/pharmacy/record_entry_spec.rb`, `spec/requests/pharmacy/entries_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- Consumes: Tasks 1–4.
- `Bnafar::LotFields.parse(raw, prefix:, domain:, required:, vmp_code:) -> [Hash, [problem]]`: `unit_price_cents` (`UNIT_PRICE_CENTS`), `manufacturer_cnpj` (`Cnpj.normalize`, `invalid_cnpj`) ou `manufacturer_international: true` com `manufacturer` (Q15), `anvisa_registration`, `health_program` (DM14, `program_unknown`), `ium`, `ampp_code` (`AMPP`; ativo em `obm_presentations` → senão `ampp_unknown`; VMP diferente do item → `ampp_not_in_vmp`; ANVISA diferente da apresentação → `anvisa_mismatch`). Com `required: true`, os obrigatórios ausentes → `required` com path `items[i].<campo>` (Q17): preço, fabricante (CNPJ ou internacional), ANVISA e programa em todo domínio; `ampp_code` só quando há `vmp_code` (Q15). `Bnafar::LotFields.missing(lot) -> [String]` (o que falta para o envio).
- `RecordEntry.call(…, bnafar_required: false)`: o controller lê `Bnafar::Gate.required_for?(Current.city, location)` fora da transação. Lote novo grava os campos; lote existente (Q7): campo nulo é preenchido; preço diferente → troca; qualquer outro campo BNAFAR já preenchido e diferente → `lot_mismatch` (path `items[i].lot_number`); toda mudança sobe `bnafar_version`, sob a trava do lote.
- `Bnafar::CompleteLot.call(lot:, attrs:, expected_version:, by:) -> Result.ok(lot:) | fail(:lot_bnafar_changed | :lot_mismatch | :invalid_request | :ampp_unknown | …, details: { path: })`; evento `stock_lot.bnafar_completed` (`stock_lot_id`, `user_id`).
- `PATCH /pharmacy/lots/:id/bnafar` corpo `{ location_id, expected_version, …campos }`: `require_stock_feature!` + `stock_feature_for!(location)`; `stock_keeper` ou `pharmacist` do `location_id` (com movimento do lote ali; senão 404) ou `municipal_admin`; sem o gate do BNAFAR. Responde `{ id, bnafar: { …campos, version, missing } }`.
- `Pharmacy::Json` do lote (posição e extrato) ganha `bnafar: { version, missing }`.

- [ ] **Step 1: Write the failing tests.** `lot_fields_spec.rb`: "AMPP inexistente → `ampp_unknown`; de outro VMP → `ampp_not_in_vmp`; ANVISA ≠ → `anvisa_mismatch`"; "item sem VMP não exige AMPP (Q15)"; "`INSULDEP I` aceita, `XYZ` → `program_unknown`"; "internacional com nome e sem CNPJ passa; internacional sem nome → `required` path `items[0].manufacturer`". `record_entry_spec.rb`: "com `bnafar_required: true`, item sem preço → `required` path `items[0].unit_price_cents`, todas as recusas de uma vez em `problems`"; "com `false`, opcionais e gravados"; "segunda nota com outro preço → o lote guarda o novo e `bnafar_version` sobe"; "outro CNPJ, outro programa → `lot_mismatch`"; "vacina com `ampp_code` válido grava". `lot_complements_spec.rb` (`type: :request`): "almoxarife completa e recebe `missing: []`"; "versão velha → 409 `lot_bnafar_changed`"; "troca de CNPJ preenchido → 422 `lot_mismatch`"; "farmacêutico de outro local → 404"; "funciona com `bnafar` desligado"; "`pharmacy_assistant` → 403"; "evento só com ids".
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/bnafar/lot_fields_spec.rb spec/commands/pharmacy/record_entry_spec.rb spec/requests/pharmacy/lot_complements_spec.rb spec/requests/pharmacy/entries_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o `Row` ganha `bnafar:`; `ensure_lots` grava; `mismatch_problems` confere os campos preenchidos).
- [ ] **Step 4:** os mesmos specs + `spec/commands/pharmacy/record_entry_concurrency_spec.rb spec/requests/pharmacy/vector_control_stock_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: capture bnafar lot fields on entry and lot complement" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 5 · campos BNAFAR na entrada (AMPP só com VMP, Q15; path items[i].campo, Q17), PATCH /pharmacy/lots/:id/bnafar; preço troca, o resto preenchido dá lot_mismatch (Q7)."

---

### Task 6: CNES dos centrais, CATMAT do produto vetorial e OBM das vacinas

**Files:**
- Create: `spec/requests/pharmacy/location_cnes_spec.rb`, `spec/services/vaccination/obm_vmp_spec.rb`
- Modify: `app/commands/pharmacy/save_location.rb`, `app/controllers/pharmacy/admin/locations_controller.rb`, `app/controllers/vaccination/admin/cold_chain_central_controller.rb`, `app/commands/vaccination/create_cold_chain_central.rb`, `config/routes.rb`, `app/services/pharmacy/json.rb`, `app/commands/vector_control/save_product.rb`, `app/services/vector_control/json.rb`, `config/vaccination/immunobiologicals.yml`, `app/services/vaccination/immunobiologicals.rb`, `spec/requests/pharmacy/admin_spec.rb`, `spec/requests/vector_control/products_spec.rb`, `spec/services/vaccination/catalog_files_spec.rb`

**Interfaces:**
- Q13: `PATCH /pharmacy/admin/locations/:id` aceita `cnes` (7 dígitos ou `null`) só no `central`; `PATCH /vaccination/admin/cold_chain_central` `{ cnes, name? }` com `require_vaccination!`; os dois pelo `municipal_admin`, com step-up. CNES de uma `health_units.cnes` → 409 `cnes_in_use` (o par CNES × DM01 nunca diverge); os dois centrais podem dividir o mesmo CNES. `Pharmacy::Json.admin_locations` ganha `cnes` e `bnafar_type`.
- `VectorControl::SaveProduct::KEYS` + `catmat_code` (Integer > 0 ou `null`; versão + 1); `VectorControl::Json` do produto ganha `catmat_code`.
- YAML (Q13): cada produto ganha `obm_vmp: null` e cada fabricante `cnpj: null` (a preencher no gate de go-live; sem `review` novo, o `under_review?` não muda); `Product#obm_vmp`, `Manufacturer#cnpj`; `Vaccination::Immunobiologicals.manufacturer_by_cnpj(product, cnpj)`.

- [ ] **Step 1: Write the failing tests.** `location_cnes_spec.rb` (`type: :request`): "admin grava o CNES do central com step-up"; "CNES num `unit` → 422 path `cnes`"; "CNES de uma UBS → 409 `cnes_in_use`"; "rede de frio ganha CNES pelo PATCH novo; sem `vaccination` → 403; almoxarife (não admin) → 403"; "a lista traz `cnes` e `bnafar_type`". `products_spec.rb`: "`catmat_code` grava e sobe a versão; `0` → 422 path `catmat_code`". `obm_vmp_spec.rb`: "todo produto tem `obm_vmp` e todo fabricante `cnpj`; `under_review?` não muda"; "`manufacturer_by_cnpj` acha pela CNPJ normalizada".
- [ ] **Step 2:** `bin/test api -w mod27b spec/requests/pharmacy/location_cnes_spec.rb spec/requests/pharmacy/admin_spec.rb spec/requests/vector_control/products_spec.rb spec/services/vaccination` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: set cnes on central stores, catmat on vector products and obm on vaccines" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 6 · CNES no central e na rede de frio pelo municipal_admin (Q13, cnes_in_use), catmat_code no vetorial, obm_vmp e cnpj no YAML, ainda nulos."

---

### Task 7: Simulado `lib/fake_bnafar` e container `fake-bnafar-dev`

**Files:**
- Create: `lib/fake_bnafar/{app.rb,config.ru}`, `spec/lib/fake_bnafar_spec.rb`, `spec/support/bnafar_helpers.rb`
- Modify: `docker-compose.yml` da raiz (fora de git; sem commit)

**Interfaces:**
- `FakeBnafar::App.new(clock: -> { Time.now }, expires_in: 1_800_000, items_limit: 20)`, Rack, sem ActiveSupport, no molde do `FakeRnds::App`:
  - `POST /jwtauth/auth` com `Authorization: Basic` → `{ access_token, scope: "read write", expires_in }` (ms; `expires_in_unit = :s` troca); sem Basic → 401;
  - `POST|PUT /bnafar/produto/ibge/:ibge/{posicao-estoque-lote,saida-lote}` com `Authorization: jwt <token>` → `Protocolo { protocolo, situacao: "1", codigoIbge, dataProtocolo, tipoOperacao }`; `DELETE …/saida-lote?codigos=a,b` → `Protocolo` de exclusão (Q14); token vencido → 401; mais de 1.000 → 400 MSG62; sem `protocolo` nem `codigos` → 400 MSG38;
  - `GET /bnafar/protocolo/ibge/:ibge/detalhar-processamento/:protocolo` → situação `2`, depois `3` ou `4`, com `itensProcessados[{ codigoBnafar, codigoOrigem, posicaoEnvio, sucesso, protocoloExclusao }]`;
  - `GET …/inconsistencias/:protocolo?pageNumber&pageSize` → `{ content: [{ codigoOrigem, posicaoEnvio, inconsistencias: [{ codigo, mensagem, valorRejeitado }] }], totalPages }`;
  - marcadores: lote `RECUSA` → MSG09; CNES `0000000` → MSG06; CNPJ `00000000000000` → MSG12; data fora de hoje/ontem → MSG73; mais de `items_limit` itens → MSG46; todos os itens recusados → sem `codigoBnafar` (MSG47); PUT ou DELETE de registro aceito há mais de 30 dias (pelo `clock`) → MSG16/MSG18; `failures` como no `FakeSncr::App`;
  - `log` sem o header `Authorization`; `received` guarda os corpos para os specs.
- `config.ru` aborta em produção. Compose: serviço `fake-bnafar` (`container_name: fake-bnafar-dev`, `bundle exec puma -b tcp://0.0.0.0:8095 lib/fake_bnafar/config.ru`, `127.0.0.1:8095:8095`) e `FAKE_BNAFAR_URL: ${FAKE_BNAFAR_URL:-http://fake-bnafar:8095}`.
- `bnafar_helpers.rb`: `FAKE_BNAFAR_BASE = "http://fake-bnafar.test:8095"`, `fake_bnafar!(app = FakeBnafar::App.new)`; os demais helpers nas tasks seguintes.

- [ ] **Step 1: Write the failing test** (`type: :model`, `Rack::MockRequest`): "token com Basic; sem Basic → 401"; "`expires_in` em ms e em s"; "lote devolve protocolo; primeira consulta situação 2, segunda 3"; "`RECUSA` → MSG09 e `sucesso: false`"; "1.001 registros → MSG62"; "21 itens → MSG46"; "anteontem → MSG73"; "DELETE por `codigos` devolve protocolo de exclusão; de registro com 31 dias → MSG18"; "token vencido → 401"; "o `log` nunca contém `Basic `, `jwt ` nem a senha"; "`config.ru` aborta com `RACK_ENV=production`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/lib/fake_bnafar_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add simulated si-bnafar for specs and dev" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"` (só os arquivos do api).

**Relato ao coordenador:** "api 27b · Task 7 · FakeBnafar::App (token ms/s, protocolo, DELETE por códigos, MSG06/09/12/16/18/38/46/47/62/73) e fake-bnafar-dev na 8095 (compose da raiz, fora de git)."

---

### Task 8: Cliente, credencial com ambiente e teste de conexão

**Files:**
- Create: `app/services/bnafar/{environment,config,mock,client,errors,evidence,homologation}.rb`, `config/bnafar/homologation.yml`, `spec/services/bnafar/client_spec.rb`, `spec/requests/integrations_bnafar_spec.rb`
- Modify: `app/commands/integrations/{set_credential,check_connection}.rb`, `app/controllers/integrations_controller.rb`, `app/services/platform/features.rb`, `spec/requests/integrations_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
- `Bnafar::Environment.current(env: Rails.env) -> "production" | "homologation"` (Valor 10; lê `bnafar_settings` com `data_source_exists?`, como o `PecConfigState`); `Environment.allowed?(value, env:)`.
- `Bnafar::Config.for_city(city, environment:, env:) -> Settings(auth_url:, api_url:, simulated:) | nil` (simulado com `bnafar_mock` ligado); `Bnafar::Mock.on?(city)`.
- `Bnafar::Client.for(city, environment:, settings:)`; `#token!(fresh: false)`; `#post_batch(subject:, ibge:, records:)`, `#put_batch(…)`, `#delete_batch(subject:, ibge:, codes:)`, `#detail(ibge:, protocol:)`, `#inconsistencies(ibge:, protocol:, page:)`, todos `-> Response = Data(:status, :protocol, :situation, :items, :error_codes, :pages)` com `status ∈ %i[accepted rejected unauthorized transient]` e `inspect` vazio. Cache `[city.id, credential.set_at.to_f, environment]`, validade `expires_in` (s; acima de `MS_THRESHOLD`, ms) menos `TOKEN_MARGIN`; 401 → repete uma vez com token novo; 400/422 → `rejected` com os códigos que casam `MSG_CODE`; 401/403 → `unauthorized`; resto e rede → `transient`.
- `Bnafar::Evidence.curl(method:, url:, body:) -> String` (`-H 'Authorization: jwt <TOKEN>'`; login `-u '<USUARIO>:<SENHA>'`; `# HTTP <status>` e o corpo por `Rnds::Errors.sanitize`), só em homologação.
- `Bnafar::Homologation.apply(payload, mapping:)` (troca CNES de origem e destino, produto, AMPP/VMP e programa pela massa de `homologation.yml`: `ibge`, `cnes`, `products`, `program`; valores da coleção Postman v1.25 — IBGE 520010, CNES 2384043 — com `a_conferir: true`).
- `SetCredential.call(…, environment: "production")`: `bnafar` como `ledi`; `environment` fora de `ENVIRONMENTS` → `invalid_environment`; `homologation` com `Environment.allowed?` falso → `environment_not_allowed` (Q9); `Platform.audit("bnafar.credential_set", city_id:, environment:)`; `integration_credential.changed` com `environment`.
- `CheckConnection.call(kind: "bnafar", city:)` testa a credencial do ambiente em uso: `token!(fresh: true)`; mensagens nossas ("Token do SI-BNAFAR emitido", "O SI-BNAFAR recusou usuário ou senha", "SI-BNAFAR inalcançável", "SI-BNAFAR não configurado pela plataforma").
- `IntegrationsController` (Q9): `update` aceita `environment` só para `bnafar`; `show` lista uma `bnafar`, a do ambiente em uso, com `environment` (os outros itens com `environment: "production"`).
- `Platform::Features.city_state` (Q9): `credentials` = as `production` dos outros `kind` + a `bnafar` do ambiente em uso.

- [ ] **Step 1: Write the failing tests.** `client_spec.rb` (`fake_bnafar!`): "`expires_in` 1.800.000 e 1.800 → token vale 28 min"; "token guardado e 401 → uma repetição; dois 401 → `unauthorized`"; "422 com MSG12 → `error_codes == ["MSG12"]`, nunca a mensagem"; "timeout → `transient`"; "homologação e produção dão tokens diferentes"; "trocar a senha descarta o token"; "`inspect` sem senha, token nem URL"; "`Evidence.curl` sem o token real"; "`Homologation.apply` troca CNES, destino, produto e programa"; "`delete_batch` manda `?codigos=`". `integrations_bnafar_spec.rb` (`type: :request`): "PUT `bnafar` com step-up grava; sem step-up → 401 `mfa_required`"; "`homologation` em test passa; com `Rails.env` production (stub) → `environment_not_allowed`"; "resposta sem usuário nem senha"; "check no simulado → `ok`; senha errada → `unauthorized` e `credential_unauthorized:bnafar`"; "`show` traz uma `bnafar`, do ambiente em uso"; "auditoria só com `city_id` e `environment`". `features_spec.rb`: "só credencial de homologação com a cidade em produção → `credential_missing:bnafar`". R18 + `bnafar.credential_set`.
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/bnafar/client_spec.rb spec/requests/integrations_bnafar_spec.rb spec/requests/integrations_spec.rb spec/services/platform/features_spec.rb spec/events/platform_event_payload_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** (A tabela `bnafar_settings` nasce na Task 10; até lá `Environment.current` devolve `production`.)
- [ ] **Step 4:** os mesmos specs + `spec/services/rnds spec/commands/integrations` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add si-bnafar client and bnafar credential per environment" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 8 · Bnafar::Client (Basic → jwt, expires_in ms/s, 401 uma vez, DELETE por códigos); credencial por ambiente, homologation só fora de produção e GET /integrations com a do ambiente em uso (Q9)."

---

### Task 9: Migração 900002 — saída externa e desfazer da abertura

**Files:**
- Create: `db/city_migrate/20261125900002_add_stock_external_exits.rb`, `app/models/{stock_external_exit,stock_external_exit_item}.rb`, `app/commands/pharmacy/record_external_exit.rb`, `app/controllers/pharmacy/external_exits_controller.rb`, `spec/models/stock_external_exit_schema_spec.rb`, `spec/requests/pharmacy/external_exits_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/pharmacy/{catalog,ledger}.rb`, `app/models/{stock_movement,stock_opening}.rb`, `app/services/city_encryption.rb`, `config/routes.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `stock_external_exits` (§4): `stock_location_id`, `exit_type` (`EXTERNAL_EXIT_TYPES`), `destination_kind`, `destination_code` (CNES 7 dígitos; CNPJ alfanumérico), `reason` cifrado, `recorded_by_user_id`, `recorded_at`, `version integer NOT NULL DEFAULT 1` (Q8: a correção é da saída inteira). `stock_external_exit_items`: `stock_external_exit_id`, `stock_lot_id`, `quantity integer > 0`, único `(saída, lote)`. Trigger: sem DELETE; UPDATE só em `exit_type`, `destination_kind`, `destination_code`, `reason` com `version + 1`; itens só acréscimo.
- Movimentos: `MOVEMENT_KINDS` + `external_exit` (−) e `opening_reversal` (−, Q10); `stock_movements.stock_external_exit_id` (FK, índice); `ck_stock_movements_{kind,document,sign}` trocados (`(stock_external_exit_id IS NOT NULL) = (kind = 'external_exit')`; `stock_entry_id` vale também em `opening_reversal`). `Ledger::DOCUMENTS` + `StockExternalExit => :stock_external_exit`. Sem autoria do mantenedor (Q11).
- Desfazer (Q10): `stock_openings.reverted_at`; os três únicos de abertura passam a parciais `WHERE reverted_at IS NULL`; `stock_openings_append_only` trocado por um guard (só `reverted_at` de NULL a valor, uma vez), condicionado à coluna, como na Task 3.
- `Pharmacy::RecordExternalExit.call(location:, attrs:, by:, now:) -> Result.ok(exit:, movements:) | fail(:invalid_request | :insufficient_balance | :location_inactive | :item_not_allowed_here, details: { path:, problems: })`: até `MAX_TRANSFER_ITEMS` itens; quantidade inteira; vacina só em múltiplo de `doses_per_vial` (`invalid_quantity`); lotes → local; um movimento por item; evento `stock_external_exit.recorded` (`stock_external_exit_id`, `user_id`).
- Rotas (Q16, sem o `bnafar`): `POST /pharmacy/external_exits` (step-up; `stock_keeper` do local; `require_stock_feature!` + `stock_feature_for!`); `GET /pharmacy/external_exits?location_id=&before=&limit=` (o `stock_keeper` do local e o `municipal_admin`; sem `reason`) e `GET /pharmacy/external_exits/:id` (com `reason`). Os locais de todos os domínios utilizáveis servem.
- `CITY_KEYED_TARGETS` + `[ StockExternalExit, :reason ]`; filtro de log + `destination_code`.

- [ ] **Step 1: Write the failing tests.** Schema: "saída sem destino → CHECK"; "CNPJ alfanumérico passa"; "UPDATE do destino com `version + 1` passa; do local → trigger"; "`external_exit` positivo → CHECK; sem documento → CHECK"; "`opening_reversal` documentado pela entrada de abertura passa"; "abertura revertida libera o índice"; "`reverted_at` duas vezes → trigger". Request: "almoxarife registra doação a um CNPJ com step-up e o saldo cai"; "saldo insuficiente → 422 `insufficient_balance` com path `items[0].quantity`"; "motivo de 9 caracteres → 422 path `reason`"; "vacina com 15 doses num lote de 10 por frasco → `invalid_quantity`"; "`municipal_admin` lê e não grava (403)"; "com `bnafar` desligado funciona; sem o estoque do domínio → 403 `feature_disabled`"; "na base `vector_control` só com o `zoonoses` funciona"; "a lista não traz `reason`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/models/stock_external_exit_schema_spec.rb spec/requests/pharmacy/external_exits_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_25_900002)`; bloco "Módulo 27b — saída externa").
- [ ] **Step 4:** os mesmos specs + `spec/models/pharmacy_schema_spec.rb spec/commands/pharmacy spec/requests/pharmacy spec/services/pharmacy/ledger_spec.rb spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add external stock exits and opening reversal" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 9 · 20261125900002: saída externa sem o bnafar (Q16), opening_reversal e abertura revertida (Q10); sem autoria do mantenedor (Q11)."

---

### Task 10: Migração 900003 — remessas, fotografia, saídas e pendências

**Files:**
- Create: `db/city_migrate/20261125900003_add_bnafar_submissions.rb`, `app/models/{bnafar_setting,bnafar_submission,bnafar_batch,bnafar_record,bnafar_movement_link}.rb`, `app/services/bnafar/{snapshot,exits,pendencies,payload}.rb`, `spec/models/bnafar_submission_schema_spec.rb`, `spec/services/bnafar/{snapshot,exits,pendencies,payload}_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `spec/jobs/reencryption_job_spec.rb`

**Interfaces:**
- Tabelas (§3 e Q8): `bnafar_settings` (linha única: `environment`, `started_on`, `updated_by_maintainer_id`, `updated_at`; `BnafarSetting.current`); `bnafar_submissions` (`reference_on`, `subject`, `environment`, `status`, `attempts`, `next_attempt_at`, `sent_at`, `finished_at`, `last_error`, `building_started_at`; único `(reference_on, subject, environment)`); `bnafar_batches` (`bnafar_submission_id`, `operation` post|put|delete, `status`, `protocol bigint`, `attempts`, `next_attempt_at`, `last_attempted_at`, `sent_at`, `finished_at`, `last_error`, `evidence jsonb`, nula fora de homologação por trigger); `bnafar_records` (`bnafar_submission_id`, `bnafar_batch_id`, `origin_code uuid` único, `subject`, `stock_location_id`, `item_key`, `payload` cifrado, `bnafar_code bigint`, `status`, `error_codes text[]`, `rejected_items integer NOT NULL DEFAULT 0`, `replaces_record_id`, `stock_external_exit_id`, `sent_at`, `accepted_at`, `deleted_at`, `deletion_batch_id`); `bnafar_movement_links` (`stock_movement_id` único, `bnafar_record_id`, `kind` em `LINK_KINDS`). Triggers: nada some; `status` só para a frente (`rejected → pending` só no reenvio); `payload` só vira NULL ou re-cifra; `bnafar_code` gravado não muda; `deleted_at` uma vez, só em registro aceito.
- `Bnafar::Snapshot.call(reference_on:, domains:, cutoff:) -> [Row(location:, cnes:, type:, item_key:, item_label:, lot:, quantity:)]`: saldo por (local, lote) com `created_at < cutoff`, `> 0`, dos domínios ligados; agrupa por (CNES, item).
- `Bnafar::Exits.call(day:, domains:, from:, to:) -> { exits: [Row(movement:, cnes:, type:, exit_type:, destination:, lot:, quantity:)], reversals: [Reversal(movement:, original_link:)] }`: negativos criados no dia, fora de `IGNORED`, sem link; estorno no mesmo dia do mesmo documento e lote compensa antes do envio; estorno de saída já ligada vira `Reversal` (a Task 11 decide `deletion` ou `late_reversal`, Q14); destino pelo Valor 5.
- `Bnafar::Pendencies.call(rows:) -> [Pendency(code:, stock_location_id:, location_name:, item_key:, item_label:, stock_lot_id:, fields:)]` e `Pendencies.current(domains_all:, domains_on:) -> [Pendency]` (saldo de agora, saídas de ontem e hoje): `lot_incomplete` (`LotFields.missing`), `lot_number_too_long` (Q7), `location_without_cnes` (origem ou destino), `product_without_code` (medicamento sem `vmp_code`, Q15; vacina sem `obm_vmp`; vetorial sem `catmat_code`), `fractional_balance`, `exit_type_missing`, `domain_off`. Sem pessoa.
- `Bnafar::Payload.position(cnes:, type:, reference_on:, origin_code:, rows:)` e `Payload.exit(row, origin_code:)` (Valor 7; `tipoProduto` pela `ProductTypes`).
- `CITY_KEYED_TARGETS` + `[ BnafarRecord, :payload ]`.

- [ ] **Step 1: Write the failing tests.** Schema: "duas remessas do mesmo dia, assunto e ambiente → único"; "movimento ligado duas vezes → único"; "`bnafar_code` não muda"; "evidência em remessa de produção → trigger"; "payload → NULL passa; trocado → trigger"; "`deleted_at` em registro rejeitado → trigger". `snapshot_spec.rb`: "lote com 30 em R e saída de 10 criada em R+1 → 30"; "em trânsito fica fora"; "vacina sem `bnafar_vaccines` fica fora"; "farmácia e sala da mesma UBS no mesmo CNES, itens separados". `exits_spec.rb`: "dispensação nunca"; "aplicação criada em R+1 com `recorded_at` em R vai em R+1 (Review Focus 1)"; "dose aplicada e estornada no mesmo dia → nada"; "estorno de saída já ligada → `Reversal`"; "perda de vacina com `loss_code expired` → S-VV"; "transferência → S-T com o CNES do destino; destino sem CNES → pendência". `pendencies_spec.rb`: "lote sem preço → `lot_incomplete` com `fields`"; "lote de 31 caracteres → `lot_number_too_long`"; "frasco aberto → `fractional_balance`"; "central sem CNES → `location_without_cnes`"; "medicamento sem VMP → `product_without_code`"; "vetorial com `bnafar_vector_supplies` desligado → `domain_off`"; "toda pendência tem `location_name` e `item_label`, nenhuma traz pessoa". `payload_spec.rb`: "item com todas as chaves; `iums: []`"; "internacional → `cnpjFabricante: null` e o nome"; "`valorUnitario` 1990 → `19.90`"; "medicamento OBM com `numero` CATMAT e `codigoVmp`; vetorial CATMAT; vacina `numero: ""`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/models/bnafar_submission_schema_spec.rb spec/services/bnafar` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_25_900003)`; bloco "Módulo 27b — envio"). Os leitores recebem `domains:` e cortes já calculados.
- [ ] **Step 4:** os mesmos specs + `spec/architecture/current_city_assignment_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/jobs/reencryption_job_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add bnafar submissions with stock snapshot, exits and pendencies" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 10 · 20261125900003 com bnafar_batches post|put|delete e evidência por chamada (Q8); fotografia por created_at; sete pendências com location_name e item_label (Q16)."

---

### Task 11: Montagem e envio — `Bnafar::DailyJob` e `Bnafar::DeliverJob`

**Files:**
- Create: `app/services/bnafar/{build,delivery,window}.rb`, `app/jobs/bnafar/{daily_job,deliver_job}.rb`, `spec/jobs/bnafar/{daily_job,deliver_job}_spec.rb`, `spec/services/bnafar/build_spec.rb`
- Modify: `config/recurring.yml`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/support/bnafar_helpers.rb`

**Interfaces:**
- `Bnafar::Build.call(reference_on:, environment:, domains:, now:) -> Result.ok(submissions:)`: por assunto, cria (ou retoma `building`) a remessa; um registro por (CNES, item, parte de 20 lotes) na posição e um por movimento na saída (link `exit`); lotes de até 1.000 (`post`); os `Reversal` (Q14): registro aceito há menos de `CORRECTION_DAYS` → lote `delete` com o `bnafar_code` e link `deletion`; senão link `late_reversal`, sem chamada; `queued`; `bnafar_submission.queued`. Remessa sem nada nasce `finished`. Grava `started_on` no primeiro dia com registro. IBGE `CityProfile.current.ibge_code[0, 6]`.
- `Bnafar::Window.close!(today:)`: remessa de `reference_on <= today - 2` não `finished` (ou com `rejected`) → `lost` e `bnafar_submission.lost`; `Window.purge!(now:)`: `payload = NULL` em aceito há mais de `PAYLOAD_DAYS` dias.
- `Bnafar::DailyJob` (`prepend EachCityJob`, `:housekeeping`, `"every hour at minute 7"`): `return unless Bnafar::Gate.usable?(city)` → `close!` → `purge!` → com `Time.current.hour >= BUILD_HOUR` e sem remessa de `today - 1` no ambiente em uso, `Build.call(reference_on: today - 1, domains: Bnafar::Gate.domains_through(city, today - 1), environment: Bnafar::Environment.current, …)`.
- `Bnafar::DeliverJob` (`prepend EachCityJob`, `:default`, `"every 5 minutes"`, `limits_concurrency to: 1, key: "bnafar_deliver", duration: STALE_SENDING`): libera `sending` velho; claim `FOR UPDATE SKIP LOCKED` dos lotes `queued` vencidos (posse pelo `last_attempted_at`); POST/PUT/DELETE → `processing` com `protocol`; `detail`: 1, 2 ou 5 espera; 3 ou 4 lê `itensProcessados` e as `inconsistencias` paginadas; registro com `codigoBnafar` e item recusado → `accepted` com `rejected_items` (Q8); MSG15 → aceito; exclusão confirmada → `deleted_at` e `bnafar_record.deleted`; recusados → `bnafar_record.rejected`; todos os lotes acabados → `finished` e `bnafar_submission.finished`. Lote recusado inteiro → `failed`, registros `rejected`. `transient` → backoff. `unauthorized` → lote volta a `queued` e a credencial do ambiente ganha `last_check_status = "unauthorized"`. Em homologação, cada chamada acrescenta `{ curl, status, body }` em `evidence` e o payload passa por `Homologation.apply`.

- [ ] **Step 1: Write the failing tests.** `build_spec.rb`: "45 lotes de um item num CNES → 3 registros (20, 20, 5)"; "2.500 registros → 3 lotes de envio"; "item pendente fica fora e o resto vai"; "rodar duas vezes → uma remessa por assunto (Review Focus 2)"; "`building` pela metade é retomada sem repetir movimento"; "estorno de dose aceita ontem → lote `delete`; de dose aceita há 31 dias → link `late_reversal` e nenhum lote (Review Focus 4)". `daily_job_spec.rb` (`type: :job`): "às 00:40 local não monta; às 01:07 monta R = ontem"; "domínio ligado hoje não entra em R"; "`bnafar` desligado → nada novo"; "remessa de anteontem com recusa → `lost`"; "payload aceito há 31 dias → NULL". `deliver_job_spec.rb` (`type: :job`, `fake_bnafar!`): "fluxo completo até `finished` com `bnafar_code`"; "`RECUSA` → `rejected` com `["MSG09"]`, nunca o texto"; "posição com um item recusado → `accepted`, `rejected_items: 1`"; "DELETE confirmado → `deleted_at` e evento"; "400 MSG62 → `failed`"; "timeout → backoff de 1 min"; "dois 401 → credencial `unauthorized` e lote `queued`"; "homologação grava evidência com `jwt <TOKEN>` e CNES da massa; produção não"; "dois jobs ao mesmo tempo não enviam o mesmo lote".
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/bnafar/build_spec.rb spec/jobs/bnafar` → FAIL.
- [ ] **Step 3: Implementar** (duas entradas no `recurring.yml`, com comentário "Módulo 27b: age à 1h do fuso da cidade; o resto da hora é retomada"; eventos `bnafar_submission.{queued,finished,lost}`, `bnafar_record.{rejected,deleted}`).
- [ ] **Step 4:** os mesmos specs + `spec/architecture/current_city_assignment_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: build and deliver daily bnafar submissions" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 11 · DailyJob (1h local, retoma building, lost às 00:00 de R+2, purga 30 d) e DeliverJob (SKIP LOCKED, protocolo, aceite parcial, DELETE do estorno em 30 d, Q14)."

---

### Task 12: Situação, pendências, domínios, reenvio, correção e alerta

**Files:**
- Create: `app/services/bnafar/{alert,json}.rb`, `app/commands/bnafar/{retry_submission,correct_exit}.rb`, `app/controllers/pharmacy/bnafar/{base,status,submissions,records,pendencies,domains}_controller.rb`, `spec/services/bnafar/alert_spec.rb`, `spec/requests/pharmacy/bnafar_spec.rb`, `spec/commands/bnafar/{retry_submission,correct_exit}_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `Bnafar::Alert.call(environment:, now:) -> { late:, lost:, rejected:, last_finished_on: }`: `late` = remessa de ontem não `finished` depois das 15:00 no fuso da cidade (Q16); `lost` = registros não aceitos de remessas `lost` nos últimos 30 dias; `rejected` = `rejected` com `reference_on = today - 1`.
- `Bnafar::Json.record(record)` (Q16): `{ id, subject, status, lost, lost_reason, deleted, stock_location_id, location_name, item_key, item_label, error_codes, rejected_items, bnafar_code, protocol (do lote), retry_until, correct_until }`; `lost_reason` = o primeiro código ou `"not_sent"`; nunca o payload.
- `GET /pharmacy/bnafar/status` → `{ environment, alert, days: [{ reference_on, subjects: [{ subject, status, accepted, rejected, pending, lost }] }] }` (30 dias); `GET /pharmacy/bnafar/submissions?from=&to=&subject=&status=&before=&limit=` e `GET …/:id` (com `batches: [{ operation, status, protocol }]` e os registros); `GET /pharmacy/bnafar/pendencies?location_id=&code=` → `{ items, counts }`. Quem: `municipal_admin` ou `stock_keeper` de qualquer local; `require_bnafar!`.
- `GET /pharmacy/bnafar/domains` (Q16; sem o `bnafar`, com o estoque): `{ programs: [{ code, label, product_type }], exit_types: [{ code, label }] (os quatro da saída externa), location_types: [{ kind, code }] }`.
- `Bnafar::RetrySubmission.call(submission:, by:, today:, domains:) -> Result.ok(batch:) | fail(:retry_window_closed | :nothing_to_retry)`: só com `today == reference_on + 1`; remonta os `rejected` com o dado de hoje e a fotografia de R (mesmo `origin_code`), lote `post` novo. `POST /pharmacy/bnafar/submissions/:id/retry` (`municipal_admin`, step-up).
- `Bnafar::CorrectExit.call(record:, attrs:, expected_version:, by:, now:) -> Result.ok(records:) | fail(:not_correctable | :correction_window_closed | :external_exit_changed | :invalid_request, details: { path: })` (Q8): registro `accepted` de saída externa, até 30 dias do `sent_at` mais antigo da saída; corpo `{ expected_version, exit_type?, destination_kind?, destination_code?, reason }`; atualiza a saída (`version + 1`) e gera um registro novo com `replaces_record_id` por registro aceito dela, num lote `put`; `bnafar_record.corrected` (`bnafar_record_id`, `replaces_record_id`, `user_id`). `POST /pharmacy/bnafar/records/:id/correct` (`municipal_admin`, step-up).

- [ ] **Step 1: Write the failing tests.** `alert_spec.rb`: "remessa de ontem `processing` às 14:59 → `late` 0; às 15:00 → 1"; "três perdidos há 10 dias e um há 31 → `lost` 3". `retry_submission_spec.rb`: "em R+1, lote completado é reenviado com o mesmo `origin_code` e o CNPJ novo (Review Focus 3)"; "em R+2 → `retry_window_closed`"; "sem recusado → `nothing_to_retry`". `correct_exit_spec.rb`: "doação de dois itens aceita há 29 dias, destino corrigido → dois registros com `replaces_record_id` num lote `put`"; "há 31 dias → `correction_window_closed`"; "posição → `not_correctable`"; "versão velha → `external_exit_changed`". `bnafar_spec.rb` (`type: :request`): "admin e almoxarife leem o status; farmacêutico sem `stock_keeper` → 403"; "registro perdido traz `lost: true` e `lost_reason`; todos com `location_name` e `item_label`"; "detalhe com o `protocol` de cada lote e sem payload"; "reenvio sem step-up → 401 `mfa_required`"; "`code=lot_incomplete` filtra"; "`?cpf=` → 422 path `cpf`"; "`bnafar` desligado → 403 nas rotas de envio e 200 em `/pharmacy/bnafar/domains`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/services/bnafar/alert_spec.rb spec/commands/bnafar spec/requests/pharmacy/bnafar_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: expose bnafar status, pendencies, domains, retry and exit correction" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 12 · status, remessas (protocolo por parte), pendências e domains (Q16); reenvio só em R+1; correção da saída inteira por PUT até 30 d (Q8); alerta às 15h."

---

### Task 13: Leituras de apoio — aberturas e apresentações OBM

**Files:**
- Create: `app/controllers/pharmacy/{openings,obm_presentations}_controller.rb`, `spec/requests/pharmacy/{openings,obm_presentations}_spec.rb`
- Modify: `config/routes.rb`, `app/services/pharmacy/json.rb`

**Interfaces:**
- `GET /pharmacy/openings?location_id=&before=&limit=` (Q13): papel no local (ou o almoxarife do central ou da rede de frio, como a posição); `{ items: [{ stock_entry_id, created_at, recorded_by: { name }, from_import: bool, reverted: bool, lots: [{ stock_lot_id, item_label, lot_number, expires_on, quantity, bnafar: { missing } }] }], next_before }` (`from_import` pela `opening_import_rows.stock_entry_id`, Task 14).
- `GET /pharmacy/obm_presentations?vmp_code=` (Q16): `{ items: [{ ampp_code, vmp_code, anvisa_registration, manufacturer_cnpj, description }] }`, só ativas; `vmp_code` obrigatório (422 path `vmp_code`); quem tem algum papel de estoque.

- [ ] **Step 1: Write the failing tests** (`type: :request`): "abertura da planilha sai com `from_import: true` e o nome de quem aplicou"; "revertida → `reverted: true`"; "sem papel no local → 403"; "apresentações do VMP, sem as inativas"; "sem `vmp_code` → 422".
- [ ] **Step 2:** `bin/test api -w mod27b spec/requests/pharmacy/openings_spec.rb spec/requests/pharmacy/obm_presentations_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`from_import` fica `false` até a Task 14 criar a tabela; o spec dessa marca entra na Task 14).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: list stock openings and obm presentations" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 13 · GET /pharmacy/openings (Q13) e GET /pharmacy/obm_presentations?vmp_code= (Q16)."

---

### Task 14: Migração 900004 — saldo inicial pela planilha (REST na cidade)

**Files:**
- Create: `db/city_migrate/20261125900004_add_opening_imports.rb`, `app/models/{opening_import,opening_import_row}.rb`, `app/services/opening_imports/{reader,validator,report}.rb`, `app/commands/opening_imports/{upload,apply,discard,revert}.rb`, `app/controllers/pharmacy/opening_imports_controller.rb`, `spec/models/opening_import_schema_spec.rb`, `spec/services/opening_imports/{reader,validator}_spec.rb`, `spec/commands/opening_imports/{apply,revert}_spec.rb`, `spec/requests/pharmacy/opening_imports_spec.rb`, `spec/fixtures/opening_imports/{ok,excel,latin1}.csv`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/requests/pharmacy/openings_spec.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- Tabelas (§5 com a Q11): `opening_imports` (`status`, `file_sha256`, `file_data` cifrado, `rows_total`, `rows_valid`, `report jsonb` (códigos, contagens, ids de local, códigos de produto), `uploaded_by_user_id`, `applied_by_user_id`, `applied_at`, `reverted_at`, `version`); `opening_import_rows` (`opening_import_id`, `line`, as onze colunas lidas como texto, `stock_location_id`, `item_kind`, `catalog_item_id`, `immunobiological_code`, `vector_product_id`, `manufacturer_key`, `status valid|error`, `error_codes text[]`, `stock_entry_id`). Trigger: sem DELETE; `uploaded → validated → applied | discarded`, `applied → reverted`; `file_data` só vira NULL (no `applied` e no `discarded`).
- `OpeningImports::Reader.call(bytes) -> Result.ok(rows:) | fail(:invalid_file | :file_too_large, details: { path: "file_base64" })` (Valor 11).
- `OpeningImports::Validator.call(rows:, today:) -> [RowResult]`: local pelo CNES (`bnafar_cnes`, ativo); item pelo `tipo_produto`; fabricante da vacina pela CNPJ (Q13); quantidade por `Units.from_bnafar`; campos por `LotFields.parse(required: true)`; `opening_exists` contra abertura não revertida; `duplicate_line`; `too_many_lots`.
- `OpeningImports::Report.call(import) -> { rows_valid, rows_error, errors_by_code, products_without_catalog, locations_without_cnes: [{ id, name, kind }], totals_by_location: [{ stock_location_id, location_name, rows, quantity, value_cents }] }`.
- Comandos: `Upload.call(bytes:, by:, now:)` (grava `validated` e as linhas); `Apply.call(import:, expected_version:, by:, now:)` — por local, numa transação cada, `Pharmacy::RecordEntry.call(kind: "opening", by:, items:, bnafar_required: false, …)` em entradas de até 100 linhas sem partir os lotes de um item; linha que falha vira `error`; `applied`, `file_data = NULL`, `opening_import.applied` (`opening_import_id`, `user_id`); `Discard.call(import:, expected_version:, by:)`; `Revert.call(import:, expected_version:, by:, now:) -> fail(:revert_blocked, details: { path: "id", locations: [id] })` sem movimento novo nos locais depois de `applied_at` (fora os da importação) nem `bnafar_records` desses locais com `sent_at >= applied_at`; grava `opening_reversal`, `stock_openings.reverted_at`, `reverted`, `opening_import.reverted`. Versão diferente → `opening_import_changed`.
- Rotas (Q11; sem o `bnafar`, D23; `require_stock_feature!`): `GET /pharmacy/opening_imports/template.csv` (o cabeçalho, `text/csv`; antes da rota `:id`), `POST /pharmacy/opening_imports` `{ file_base64 }` (413 `file_too_large` acima de 5 MB decodificados), `GET /pharmacy/opening_imports/:id?before=&limit=` (relatório e linhas paginadas), `POST …/:id/apply` e `POST …/:id/revert` com step-up e `expected_version`, `POST …/:id/discard` com `expected_version`. Quem: `municipal_admin` ou `stock_keeper` do central (`Pharmacy::Access.central_keeper?`); outro → 403 `missing_role`.
- `CITY_KEYED_TARGETS` + `[ OpeningImport, :file_data ]`.

- [ ] **Step 1: Write the failing tests.** `reader_spec.rb`: "o `excel.csv` (BOM, CRLF, `;` entre aspas, `1.234,56`, linha vazia no fim) dá as linhas do `ok.csv` (Review Focus 5)"; "`latin1.csv` → `invalid_file`"; "coluna a menos → `invalid_file`"; "5 MB + 1 byte → `file_too_large`". `validator_spec.rb`: um teste por código do Valor 11; "vacina com 3 frascos de 10 doses → 30 doses e a `manufacturer_key` pela CNPJ". Schema: "aplicar duas vezes → trigger"; "`file_data` volta a ter valor → trigger". `apply_spec.rb`: "250 linhas em dois locais → três entradas `opening` com `recorded_by_user_id` de quem aplicou, saldo certo, `file_data` NULL"; "item que ganhou abertura entre a conferência e a aplicação → linha `opening_exists`, o resto entra"; "nada mexe no estoque antes do apply". `revert_spec.rb`: "sem movimento novo → saldo zerado por `opening_reversal`, abertura liberada, reaplicável"; "com dispensação depois → `revert_blocked` com o local"; "com registro BNAFAR enviado do local → `revert_blocked`". `opening_imports_spec.rb` (`type: :request`): "almoxarife do central envia, confere e aplica com step-up"; "apply sem step-up → 401 `mfa_required`"; "farmacêutico de UBS → 403"; "`template.csv` é o cabeçalho exato"; "6 MB → 413"; "o relatório não traz o conteúdo do arquivo"; "versão velha → 409 `opening_import_changed`". `openings_spec.rb`: "abertura aplicada pela planilha sai com `from_import: true`".
- [ ] **Step 2:** `bin/test api -w mod27b spec/models/opening_import_schema_spec.rb spec/services/opening_imports spec/commands/opening_imports spec/requests/pharmacy/opening_imports_spec.rb spec/requests/pharmacy/openings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (dump `define(version: 2026_11_25_900004)`; bloco "Módulo 27b — importação").
- [ ] **Step 4:** os mesmos specs + `spec/commands/pharmacy/record_entry_spec.rb spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: import opening stock from a reviewed spreadsheet" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 14 · 20261125900004; /pharmacy/opening_imports REST (municipal_admin ou almoxarife do central, step-up no apply e no revert, Q11); autor é o usuário da cidade; 18 códigos de linha."

---

### Task 15: GraphQL do maintenance — ambiente e situação do envio

**Files:**
- Create: `app/graphql/maintenance/mutations/set_bnafar_environment.rb`, `app/graphql/maintenance/types/city_bnafar_status_type.rb`, `app/commands/bnafar/set_environment.rb`, `spec/requests/maintenance/bnafar_status_spec.rb`
- Modify: `app/graphql/maintenance/types/{mutation_type,query_type}.rb`, `app/graphql/maintenance/mutations/city_mutation.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
- `setBnafarEnvironment(citySlug, environment)` (`CityMutation#in_city`): `Bnafar::SetEnvironment.call(environment:, by:)` grava `bnafar_settings.environment` e `updated_by_maintainer_id`; `homologation` só com `Environment.allowed?` (Q9), senão recusa `environment_not_allowed` com path `environment`; auditoria `maintenance.bnafar.environment_changed` com `environment`. `AUDITED_FIELD_RULES` + `environment` (`Bnafar::Catalog::ENVIRONMENTS`).
- `cityBnafarStatus(citySlug)` (molde `cityPecConfig`) → `{ environment, environmentEditable, enabledDomains, alert { late lost rejected lastFinishedOn }, startedOn }`.
- `MaintenanceAudit::NAMES` + `maintenance.bnafar.environment_changed`; o mesmo no R18.

- [ ] **Step 1: Write the failing tests** (`type: :request`): "`setBnafarEnvironment(homologation)` em test passa e audita só `environment`"; "com `Rails.env` production (stub) → `environment_not_allowed`"; "`cityBnafarStatus` mostra alerta, ambiente e `environmentEditable`"; "token sem escopo da cidade → CITY_OUT_OF_SCOPE"; "o schema não tem mutation de importação de saldo (Q11)". R18 com o nome.
- [ ] **Step 2:** `bin/test api -w mod27b spec/requests/maintenance/bnafar_status_spec.rb spec/events/platform_event_payload_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/maintenance spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance/analyzers_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add maintenance bnafar environment and status" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 15 · setBnafarEnvironment só fora de produção (Q9) e cityBnafarStatus; importação fora do maintenance (Q11)."

---

### Task 16: Semente de dev para a prova contra o simulado

**Files:**
- Create: `lib/bnafar_crew.rb`, `spec/lib/bnafar_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- `BnafarCrew.seed_platform!` (três `obm_presentations` fictícias com o VMP de dois itens da semente do 27a e uma sem item, para a lista de revisão); `BnafarCrew.seed_current_city(slug:)` depois do `PharmacyCrew`, `VaccinationCrew` e `ZoonosesCrew`: CNES no central e na rede de frio, campos BNAFAR em metade dos lotes (a outra metade vira pendência), um lote internacional, `catmat_code` num produto vetorial, credencial `bnafar` nos dois ambientes com senha de dev, `bnafar`, `bnafar_medications` e `bnafar_mock` ligados (vacinas e vetorial desligados, para `domain_off`), e um `ok.csv` de exemplo em `tmp/` para a tela do dashboard. `summary_lines` sem senha de staging ou produção.

- [ ] **Step 1: Write the failing test** (`type: :model`): "idempotente"; "há pendência de cada código, salvo `exit_type_missing` e `lot_number_too_long`"; "credencial `bnafar` nos dois ambientes"; "nenhum CNES de central é de unidade".
- [ ] **Step 2:** `bin/test api -w mod27b spec/lib/bnafar_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec + `spec/lib/pharmacy_crew_spec.rb spec/lib/vaccination_crew_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "chore: seed bnafar dev data for the simulator proof" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 16 · BnafarCrew (CNES, lotes metade completos, credencial nos dois ambientes, bnafar_mock, CSV de exemplo)."

---

### Task 17: Sigilo, eventos e R18 (§9)

**Files:**
- Create: `spec/security/bnafar_secrecy_spec.rb`

- [ ] **Step 1: Write the test** (`type: :request`, molde do `vector_control_secrecy_spec.rb`, constantes num módulo `BnafarSecrecy`, ASCII, palavras únicas): usuário e senha marcados, token do simulado marcado, CPF e nome de um paciente com dispensação no dia, motivo da saída externa marcado. Cenário pelas rotas reais: credencial nos dois ambientes e testada; entrada com os campos; complemento; saída externa; `DailyJob` e `DeliverJob` contra o `FakeBnafar::App` (aceite, `RECUSA`, 401 duas vezes, estorno com DELETE) em produção e em homologação; reenvio; correção; leituras; upload, apply e revert da planilha por `/pharmacy/opening_imports`; `importObmCatalog`. Provas:
  - `capture_log` sem usuário, senha, token, `Basic `, `jwt ` seguido de token real, CPF, nome do paciente, motivo marcado nem `file_base64`;
  - eventos publicados do módulo ⊂ `Bnafar::Catalog::DOMAIN_EVENTS` + `stock_receipt.recorded` + `integration_credential.changed`, só ids (e `environment`, `subject`);
  - `bnafar_batches.evidence` só em homologação, sem senha nem token real;
  - nenhum payload decifrado contém CPF, CNS ou nome de pessoa, e nenhum tem `S-PA`;
  - nenhuma resposta traz usuário, senha, token, payload ou conteúdo do CSV; toda recusa 4xx só com `error`, `path` e `problems`;
  - `opening_imports.file_data` NULL depois do apply e do discard; payload aceito NULL depois de 30 dias;
  - grep multilinha de `Platform.audit(` nos arquivos do módulo: só `bnafar.credential_set` e `obm_catalog.imported`; os dois `maintenance.*` em `MaintenanceAudit::NAMES`; os quatro nomes no R18; nenhum campo auditado com `name`.
- [ ] **Step 2:** `bin/test api -w mod27b spec/security/bnafar_secrecy_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture` → PASS (falha → correção na task dona, volta aqui).
- [ ] **Step 3: Commit** `git commit -m "test: pin bnafar secrecy across logs, events, evidence and responses" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 27b · Task 17 · sigilo fixado (credencial e token fora de log/evento/cURL, payload sem pessoa e sem S-PA, CSV apagado, R18 com 4 nomes)."

---

### Task 18: Entrega — suíte completa e passagem aos fronts (para antes do merge)

**Files:** nenhum código novo.

- [ ] **Step 1: Suíte completa:** pedir à sessão "Testes", por `SendMessage`: `api · mod27b/feat/mod-27b-bnafar · --full · módulo 27b (4 migrações de cidade e 1 de plataforma, CityEncryption, Platform::Features, IntegrationCredential, estoque do 27a/29/32b/35a/35b, catálogo da plataforma, YAML de imunobiológicos, GraphQL do maintenance, spec/support, initializers)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador com o resumo e o hash, e as sessões do dashboard e do maintenance (porta 3062; simulado na 8095).
- [ ] **Step 2: Parar.** Merge, push e cards só com autorização do usuário; F-27.8..F-27.13 seguem **In Progress** até o último app de cada card; nenhum card vai a Verified por este plano. Os gates de go-live do spec §7, o do maintenance em produção (Q18), o layout do arquivo OBM (Q12), a unidade por produto, o S-CA e o `numero` vazio na vacina (homologação) vão ao board #2 no fechamento do módulo (outra sessão).

**Relato ao coordenador:** "api 27b · Task 18 · suíte completa <verde N/0 | falhas>; hash <…>; pronto para dashboard e maintenance; parado antes do merge."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptores e gate (2), credencial com ambiente (3, 8), cliente e cache (8); §2 lote (3, 5), local (3, 6), catálogo e OBM (4), vacinas (6), vetoriais (3, 6); §3 tabelas (10), granularidade e divisão (11), jobs (11), pendências (10, 12), recusas, correções e exclusões (11, 12), alerta (12), mapeamentos (1), quantidade (1, 10), IBGE (11), homologação (8, 11); §4 saída externa (9); §5 planilha (14), conferência no dashboard (13, 14); §6 rotas (5, 6, 9, 12, 13, 14, 15); §9 eventos (5, 9, 11, 12, 14), R18 (4, 8, 15, 17), sigilo (17); §10 testes (todas) e suíte (18); Q6–Q18 na Reconciliação.
2. **Testes do §10:** DM01/DM12/DM13 (1); granularidade e divisão (11); D+1 e `lost` (11, 12); correção de 30 dias (12); `expires_in` ms e s (8); planilha com cada erro, abertura repetida, local sem CNES e desfazer bloqueado (14); sigilo (17); suíte (18).
3. **Nomes:** `Bnafar::{Gate, Catalog, Limits, Domains, LocationTypes, ExitTypes, ProductTypes, Units, Environment, Config, Mock, Client, Errors, Evidence, Homologation, Snapshot, Exits, Pendencies, Payload, Build, Delivery, Window, Alert, Json, LotFields}`, `Bnafar::{CompleteLot, RetrySubmission, CorrectExit, SetEnvironment}`, `Bnafar::{DailyJob, DeliverJob}`, `Pharmacy::RecordExternalExit`, `Medications::ObmImport`, `OpeningImports::{Reader, Validator, Report, Upload, Apply, Discard, Revert}`, `FakeBnafar::App`, `BnafarCrew`; `RecordEntry.call(…, bnafar_required:)` nas Tasks 5 e 14.
4. **Ordem:** 1 → 2 → 3 → 4 (OBM, usado pela 5) → 5 → 6 → 7 → 8 → 9 (kinds, usados pelas 10 e 14) → 10 → 11 → 12 → 13 → 14 → 15 → 16 → 17 → 18.

## Divergências propostas ao contrato

> D1–D22 **resolvidas no §11 do contrato (Q6–Q18)**. Cada item guarda o problema, a evidência e a proposta; a marca diz qual Q resolveu e, quando a decisão difere da proposta, o que vale. D23 e D24 são novas e estão abertas.

D1. **[Resolvida pela Q6.] Com o rótulo "BNAFAR simulado"; "domínio ligado" = interruptor utilizável.** **`bnafar_mock` fora da tabela dos interruptores.** O §1 cita o interruptor de simulado (contrato §1, linha 77), mas a tabela só tem quatro chaves (linhas 54-60); o molde `rnds_mock` fica fora do `CITIZEN_HIDDEN` (`app/services/platform/features.rb:167-171`, `:239`). **Proposta:** quinta entrada, `requires: %w[feature:bnafar]`, `environments: SIMULATION_ENVS`, fora do `CITIZEN_HIDDEN`.

D2. **[Resolvida pela Q7.] Diferente: o guard aceita só o complemento (de nulo a valor) e a troca do preço; programa e IUM preenchidos e diferentes também dão `lot_mismatch`.** **O lote é imutável, e o contrato manda gravar nele.** `stock_lots` é só acréscimo (`db/city_triggers.sql:2654`) e a entrada recusa divergência com o lote gravado (`app/commands/pharmacy/record_entry.rb:397`). **Proposta:** guard só das colunas BNAFAR com versão; CNPJ, ANVISA e AMPP diferentes → `lot_mismatch`; preço, programa e IUM seguem a entrada mais recente.

D3. **[Resolvida pela Q7.]** **CNPJ "14 dígitos".** O CNPJ é alfanumérico desde julho de 2026 (`app/services/cnpj.rb:1-8`; `ck_city_profile_cnpj`, `db/city_schema.rb:623`). **Proposta:** `Cnpj.normalize` e CHECK `^[0-9A-Z]{12}[0-9]{2}$`.

D4. **[Resolvida pela Q7.] Diferente: o lote acima de 30 caracteres é a pendência própria `lot_number_too_long`.** **Tamanhos que não batem com o manual.** AMPP 25 no manual (`manual_v125.txt:710, 716`), programa da DM14 com 10 (`:784`), lote BNAFAR com 30 (`:732`) contra 40 no Rota Saúde (`Pharmacy::Catalog::LOT_NUMBER`). **Proposta:** AMPP até 25; programa da DM14; lote longo como `lot_incomplete` com `fields: ["lot_number"]`.

D5. **[Resolvida pela Q8.]** **Uma remessa, vários protocolos.** O §3 dá um `protocol` por remessa, mas a remessa acima de 1.000 registros se divide em partes (contrato §3, linha 138), e reenvio e PUT são chamadas novas do mesmo dia. **Proposta:** `bnafar_batches` e `bnafar_records.bnafar_batch_id`.

D6. **[Resolvida pela Q8.]** **Evidência por registro.** O cURL é de cada chamada HTTP (manual §17; pesquisa achado 14). **Proposta:** `bnafar_batches.evidence`, nula fora de homologação.

D7. **[Resolvida pela Q8.]** **Registro de posição aceito em parte.** O SI-BNAFAR salva o registro e recusa só os itens inconsistentes (MSG47 só com todos; pesquisa linha 124). **Proposta:** `accepted` com `error_codes` e `rejected_items`.

D8. **[Resolvida pela Q9.] Diferente: `GET /integrations` mostra só a `bnafar` do ambiente em uso, e o `check` testa essa.** **Credencial com ambiente nas leituras.** `GET /integrations` monta um item por `kind` (`app/controllers/integrations_controller.rb:26`) e o `city_state` faz `pluck(...).to_h` (`app/services/platform/features.rb:295`). **Proposta:** `environment` em cada item, a `bnafar` duas vezes, e `check` com `{ environment }`.

D9. **[Resolvida pela Q9.] Também: a credencial `homologation` só se grava fora de produção.** **Não existe "cidade de staging".** `cities` não tem marca de ambiente (`db/platform_schema.rb:105-125`). **Proposta:** `homologation` só com `Rails.env` em `SIMULATION_ENVS`.

D10. **[Resolvida pela Q10 e pela Q11.] Diferente: a autoria do mantenedor some inteira (o autor é o usuário da cidade); o desfazer é o `opening_reversal`, como proposto.** **O mantenedor como autor ia além de entrada e movimento.** `stock_lots.created_by_user_id` é `NOT NULL`, e o desfazer por ajuste exige `stock_adjustments.requested_by_user_id` e as regras de decisão (`db/city_schema.rb:4442-4470`). **Proposta:** autoria do mantenedor também no lote; desfazer por `opening_reversal`.

D11. **[Resolvida pela Q10.]** **Reaplicar depois de desfazer.** `stock_openings` é só acréscimo e única por (local, item) (`db/city_schema.rb:4613-4615`). **Proposta:** `stock_openings.reverted_at` e únicos parciais.

D12. **[Resolvida pela Q11.] Diferente: a planilha sai do maintenance para REST na cidade; o limite próprio de corpo fica só para o `importObmCatalog`, fora de produção.** **O corpo do GraphQL do maintenance tem 64 KB.** `MAX_BODY_BYTES = 64_000` (`app/controllers/maintenance/graphql_controller.rb:12`). **Proposta:** `UPLOAD_BODY_BYTES` para `uploadOpeningImport` e `importObmCatalog`.

D13. **[Resolvida pela Q12.] Também: em produção, pela rake `medications:import_obm[path]`; a mutation só fora de produção; o que não casa vai à lista de revisão.** **O portal OBM não tem API acessível, e ninguém preenchia o VMP do catálogo.** O portal exige login gov.br e não documenta API nem download; o §2 só carregava `obm_presentations`. **Proposta:** arquivo com mapa de colunas e o VMP do catálogo pela coluna CATMAT.

D14. **[Resolvida pela Q11.] Diferente: o modelo sai em `GET /pharmacy/opening_imports/template.csv`.** **A rota do modelo da planilha não tinha casa.** A API de manutenção só desenha `/session`, `/invitations/*` e `/graphql` (`config/routes.rb:52-60`), e `/maintenance` nos hosts de cidade é a tela de dev (`:1384`). **Proposta:** consulta GraphQL `openingImportTemplate`.

D15. **[Resolvida pela Q13.]** **A planilha não identificava o fabricante da vacina.** A linha de vacina exige `manufacturer_key` (`app/commands/pharmacy/record_entry.rb:305-311`), e o YAML não tem CNPJ de fabricante. **Proposta:** `cnpj` por fabricante no YAML e a `key` resolvida pela CNPJ.

D16. **[Resolvida pela Q13.]** **Não havia rota para o CNES dos centrais.** A rede de frio só tem criação (`config/routes.rb:873`). **Proposta:** `cnes` no `PATCH /pharmacy/admin/locations/:id` e `PATCH /vaccination/admin/cold_chain_central`.

D17. **[Resolvida pela Q13.]** **O dashboard não tinha como ler as aberturas.** Só há `POST /pharmacy/receipts` e `POST /pharmacy/openings` (`config/routes.rb:565-566`). **Proposta:** `GET /pharmacy/openings`.

D18. **[Resolvida pela Q14.] Diferente: estorno de saída aceita há menos de 30 dias faz `DELETE` do registro original; depois disso, "estorno fora da janela".** **Estorno de uso direto.** Os três `*_reversal` são positivos (`app/services/pharmacy/catalog.rb:21-27`), e o contrato não dizia o que fazer com a saída já enviada. **Proposta:** compensar no mesmo dia; estorno posterior não enviado; DELETE fora.

D19. **[Resolvida pela Q13.]** **Destino das saídas internas.** `estabelecimentoDestino` é obrigatório em toda saída (manual §11.1, 3.3). **Proposta:** `S-T` com o CNES do destino; o resto com o próprio CNES.

D20. **[Resolvida pela Q11 e pela Q18.] Diferente: a importação vai para a cidade; o 27b herda o gate do maintenance em produção sem criar outro.** **A API de manutenção não existe em produção** (`config/routes.rb:46-52`). **Proposta:** gate de go-live explícito.

D21. **[Resolvida pela Q8.]** **A correção é da saída, não do registro.** Uma saída de N itens vira N registros (§3). **Proposta:** a correção vale para a saída inteira, com `version` na saída e um PUT por registro aceito.

D22. **[Resolvida pela Q15.] Diferente: o AMPP só é exigido quando o item tem VMP; medicamento sem VMP entra no estoque com `product_without_code`; fabricante internacional pelo `nomeFabricanteInternacional`.** **`numero`, `codigoVmp` e "produto sem código" no medicamento.** O manual exige `numero` (§11, 4.2), e o Swagger só exige `codigoVmp` sem AMPP. **Proposta:** OBM com AMPP do lote, `numero` com o CATMAT; `product_without_code` só na vacina sem `obm_vmp` e no vetorial sem CATMAT.

D23. **[Aberta, nova.] A importação e o gate do BNAFAR.** O §6 exige `Bnafar::Gate.usable?` em toda rota de cidade, salvo credencial e complemento (contrato §6, linha 237), e a Q16 abriu a exceção só para a saída externa; a Q11 põe a importação em `/pharmacy/opening_imports` sem dizer o gate. A abertura vem antes do primeiro envio (decisão 14), quando o `bnafar` ainda pode estar inutilizável (sem credencial). **Proposta:** a importação, o modelo e as leituras de apoio (`/pharmacy/bnafar/domains`, `/pharmacy/obm_presentations`, `/pharmacy/openings`) exigem só o estoque do domínio, como a saída externa.

D24. **[Aberta, nova.] Nome de auditoria da rake da OBM.** A rake da Q12 roda sem mantenedor, então não passa pelo `MaintenanceAudit`, e o §9 só lista `maintenance.obm_catalog.imported`. O `CatalogImport` audita `medication_catalog.imported` no próprio serviço (`app/services/medications/catalog_import.rb:73`). **Proposta:** o `Medications::ObmImport` audita `obm_catalog.imported` (nome novo no R18), e a mutation fora de produção continua com `maintenance.obm_catalog.imported`.
