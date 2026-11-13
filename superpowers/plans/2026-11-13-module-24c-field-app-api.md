# Módulo 24c — App de campo offline (api) Implementation Plan

> **Q51 do contrato prevalece:** sem `field_sync_receipts.result_patient_id`; `patient_client_uuid` resolve qualquer recibo aceito de `individual_registration` para o `patient_id` do cadastro; o app não reescreve dependentes em "É esta pessoa". Onde este plano diz o contrário (api Tasks 3 e 9; campo Tasks 21–22), vale a Q51.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: api em `origin/main` = `7a7343bc`** (conferido ao escrever este plano). Última migração de cidade `20261112900005_add_home_visit_indexes.rb` (`db/city_schema.rb:13`, `define(version: 2026_11_12_900005)`); última de plataforma `20261104900001_add_sigtap_procedure_limits.rb` (`db/platform_schema.rb:13`, `define(version: 2026_11_04_900001)`). Borda do app: `FieldAuthentication#authenticate_device!` (`app/controllers/concerns/field_authentication.rb:60-76`: revogado → `device_revoked` em `:63`; revogação na borda `membership_ended`/`user_disabled` e o mesmo `device_revoked` em `:69-70`); ordem limite → `require_home_visits!` → token → `query_only!` → JSON. `FieldSync::Catalog` (`ITEM_TYPES` `:11` sem `family`; `REVOKED_REASONS` `:22`; `HTTP_ERRORS` `:28`). `FieldSync::Schemas::SHA256` presos na tag `field-sync-v1.1.0` (`app/services/field_sync/schemas.rb:14-20`), conferidos literalmente em `spec/services/field_sync/schemas_spec.rb:13-21`; exemplos em `spec/fixtures/field_sync/examples/` + `manifest.json`; `spec/services/field_sync/domains_spec.rb` cruza `domains-v1.json` com o api (inclusive `HTTP_ERRORS ⊂ errors.http`). `FieldSync::Json.session` (`json.rb:35-42`, sem `map` nem `reference_version`) e `household_body` (`:64-69`, a única montagem do `<household>` do download e do `current`). `FieldSync::Upload` (`upload.rb`): `RESULT_TYPES` `:48-50`, `ledger` `:159-165` (o `duplicate` não tem versão), `cite` `:307-319` (só `patient`/`household`), `cited_id` `:326` (por `item_type`), `household` `:245-261`, `registration` `:266-293`, `accepted` `:383`, `result_json` `:400-414`. `field_sync_receipts` (`db/city_schema.rb:1571-1581`, CHECK `ck_field_sync_receipts_item_type` sem `family`, só acréscimo por `rota_append_only`). `FieldDevices::Register.call(user:, name:)` (trava o usuário, limite 3) e `FieldDevices::Revoke.call(device, by:, reason:)` (`requires_new`, publica `field_device.revoked`; recusa motivo fora de `REVOKED_REASONS`, `revoke.rb:16`; `field_device.rb:11` valida o mesmo); `field_devices` com `ck_field_devices_revoked_by` (só `admin`/`user` com `revoked_by_user_id`) e `ck_field_devices_revoked_reason` (`user admin membership_ended user_disabled`) em `db/city_schema.rb:1564-1566`; `FieldDevicesController#create` com `require_home_visits!` + `require_step_up!`. Borda da cidade: `CityResolution` responde `unknown_city` (404), `city_suspended` (403) e `city_schema_behind` (503) (`app/controllers/concerns/city_resolution.rb:25-30`). 24a: `TerritoryRegistry::SaveHousehold` (corpo inteiro a cada versão; `params_of` `:71-76` usado por `SaveFamily`/`ReplaceResponsible`; `parse_fields` `:99-126`; `persist!` `:181-197`), `SaveFamily.call(household:, params:, by:, family: nil)` (`responsible.patient_id` aninhado `:56-57`; só exige `Patient.exists?` `:61`; `link_responsible!` liga o cadastro ativo SE houver `:92-111`; devolve `ok(family:, household:, version:)`), `SaveRegistration` (`ok(registration:, version:)` `:356`). `TerritoryRegistry::Json.household` (`json.rb:25-33`, compartilhado pelo dashboard e pelo app). `HouseholdVersion` (`serialize` + `encrypts :address, :living_conditions, :contact_phone, :refused, :families_snapshot`), `HomeVisit` (`encrypts :reasons, :measurements, :note, :withdrawal_reason`, `home_visit.rb:14-16`); `CITY_KEYED_TARGETS` (`city_encryption.rb:137-138` e `:220`); triggers `household_versions_reencryption_only` (`db/city_triggers.sql:2320-2322`) e `rota_home_visit_guard` (`:3967-3996`, `ciphered` sem `location`). `HomeVisits::Record::ALLOWED` sem `location` (`record.rb:39-42`; chave desconhecida → `invalid_home_visit`). LEDI: `cadastro_domiciliar.thrift:63-64` (`17 latitude`, `18 longitude`, `double`), `ficha_visita_domiciliar.thrift:31-32` (`20 latitude`, `21 longitude`), `gen-rb` com as mesmas constantes; `Ledi::Fichas::HouseholdRegistration#to_thrift` (`household_registration.rb:45-56`), `Ledi::Fichas::HomeVisitDay#child` (`home_visit_day.rb:40-53`; o cabeçalho `:11` diz "Nunca … latitude, longitude"); `Ledi::HouseholdRegistrationFicha.values` (`household_registration_ficha.rb:28-39`), `Ledi::HomeVisitDayFicha.visit_values` (`home_visit_day_ficha.rb:88-110`). Referência: dicionários do 24a só em `config/ledi/registration_dictionaries.yml` lido por `TerritoryRegistry::Dictionary::{DATA,DOMAINS,STRING_DOMAINS,SECTIONS}` (`dictionary.rb:10-13`) — o dashboard **não** os lê do api: gera `territoryDictionaries.generated.ts` no build (`apps/dashboard/scripts/ledi-dictionaries.mjs`); `HomeVisits::Json.domains` e `CollectiveActivities::Json.domains` (um bloco só, `json.rb:95`); profissionais da atividade por `CollectiveActivities::Access.professional_memberships(team)` (`access.rb:45-51`; qualquer profissional ativo, `in_team`, `has_cns`; `options_controller.rb:56-62`); `Neighborhood.active_neighborhoods`, `HealthUnit.active_units`; `Signatures::Jcs.dump` (RFC 8785). Armazenamento: Active Storage com `minio` (S3) em dev/staging/produção e `Disk` em test (`config/storage.yml`; `config/environments/test.rb:26`), `ActiveStorage::Record` no banco de plataforma (`config/initializers/active_storage_platform.rb`); nenhum arquivo do api lido por intervalo hoje. Cache: `solid_cache_store` num banco `cache` compartilhado (`config/cache.yml`), `:null_store` em test. Plataforma: `PlatformRecord`; `PlatformQueue::JOBS = %w[ProvisionCityJob PurgePlatformAccessJob]` (`platform_queue.rb:26`); IBGE do município só no banco da cidade (`city_profile.ibge_code`, `db/city_schema.rb:533`), sem geometria nem caixa em lugar nenhum. GraphQL do maintenance **no api** (`app/graphql/maintenance/`; `BaseMutation#audited` com `MaintenanceAudit::NAMES` todos `maintenance.*`, `app/events/maintenance_audit.rb:15-40`; cidades por `citySlug`, `city_mutation.rb:33-36`). `R18_PLATFORM_EVENT_NAMES` em `spec/events/platform_event_payload_guard_spec.rb:15-38`. `spec/adr_pointers_spec.rb:20` `VALID_RANGE = (1..52)`; `docs/adr/0053.md` existe. `filter_parameters` sem `location`/`latitude`/`longitude`. `spec/spec_helper.rb:69` exclui `:pec` sem `LEDI_PEC_URL`. Dockerfile: estágio `base` (`Dockerfile:7-40`) comum a web, worker e dev. Helpers: `home_visit_city!`, `field_agent!`, `assignment!`, `registry_*`, `sign_in_as`, `stepped_up!`, `json_post`, `capture_log`.

**Goal:** O lado api do módulo 24c (F-24.19, F-24.20, F-24.21, F-24.24 — Q10; ADR 0053): `device_suspended` (revogação automática e por substituição, motivo novo `replaced`), sessão com `map` e `reference_version`, item `family` e `family_client_uuid`, citação de pessoa criada por cadastro individual (Q26), `version` no `accepted`/`duplicate` com `result_version` no livro, `location` cifrada no domicílio e na visita com latitude/longitude nas fichas LEDI 3 e 8 (erro de esquema sob `location` → `invalid_location`), `GET /field/v1/reference`, `replaces_device_id`, e o mapa por município (`field_maps`, jobs, CLI `pmtiles`, `GET /field/v1/map`, `GET /territory/map`, `GET /territory/map/households` com `map`, GraphQL do maintenance), tudo testado contra a cópia dos schemas `field-sync` v1.2.0.

**Architecture:** Uma migração de cidade (`20261113900001`, irreversível) acrescenta `location` cifrada a `household_versions` e `home_visits` e `result_version` + `family` ao livro; uma de plataforma (`20261113900001`) cria `field_maps` e `field_map_bases`. A coordenada é validada num único módulo puro (`FieldLocation`), usado pelo `SaveHousehold` (24a, também no `PATCH` do dashboard) e pelo `HomeVisits::Record` (só `origin: field_app`). O mapa é um objeto no serviço do Active Storage (MinIO), escrito e lido por chave (`FieldMaps::Storage`, sem `Blob`), recortado pelo CLI `pmtiles` num job de plataforma, e servido por um concern de entrega com `Range`/`ETag` usado pela rota do aparelho e pela do dashboard.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), Solid Queue (fila de plataforma), Active Storage (S3/MinIO; Disk em test), `json_schemer`, `graphql` 2.3, CLI `go-pmtiles` (binário fixo na imagem). **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-13-module-24c-field-app-design.md` (decisões 1A–6A e 1–28). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-13-module-24c-field-app-contracts.md` (§1–§13), sobre o contrato do 24b (`2026-11-12-module-24b-home-visit-contracts.md`, §9, §13 e §16 até a Q75). ADR `docs/adr/0053.md`. As lacunas G-1..G-24 foram decididas na §13 do contrato (Q1–Q49, que prevalece sobre o texto anterior); o plano já aplica cada decisão onde ela cai, com o número da Q no texto, e o resumo está em **"Decisões aplicadas (§13 do contrato)"**, no fim.

## Valores fixados por este plano

1. **`FieldLocation`** (`app/services/field_location.rb`, puro): `REQUIRED = %w[latitude longitude source]`; `OPTIONAL = %w[accuracy_m]`; `ACCURACY_REQUIRED_SOURCES = %w[gps]`; `LATITUDE = (-90..90)`; `LONGITUDE = (-180..180)`; `DECIMALS = 8`; `ACCURACY = (0..10_000)`; `ACCURACY_DECIMALS = 1`; `SOURCES = %w[gps map]`; `VISIT_SOURCES = %w[gps]`. Número = `Integer` ou `Float` (nunca `String`/`true`); casas contadas em `BigDecimal(number.to_s)` (a forma mais curta que volta ao mesmo `Float`; `1.0e-09` conta 9) — mais casas → recusa, **sem arredondar**. `accuracy_m` obrigatório com `source: "gps"` e opcional com `"map"` (Q3). Erro → `Result.fail(:invalid_location, details: { path: })` com `path` relativo ao corpo: `location`, `location.latitude`, `location.longitude`, `location.accuracy_m`, `location.source`.
2. **`FieldSync::Catalog`:** `ITEM_TYPES` ganha `family` no fim; `REVOKED_REASONS = %w[user admin replaced membership_ended user_disabled]` e `SUSPENDED_REASONS = %w[replaced membership_ended user_disabled]` (Q17: `device_revoked` só para `user`/`admin`); `HTTP_ERRORS` ganha `device_suspended`, `map_not_found` e `range_not_satisfiable` (Q8; os de borda da cidade continuam da `CityResolution` e são conferidos contra `errors.http` no `domains_spec`); `REFERENCE_CACHE = 5.minutes`; `MAP_CONTENT_TYPE = "application/vnd.pmtiles"`; `MAP_CHUNK = 1.megabyte`.
3. **`FieldMaps::Catalog`:** `STATES = %w[building ready failed]`; `ERROR_CODES = %w[base_missing extract_failed boundary_missing]`; `BASE_STATES = %w[copying ready failed]`; `BASE_STALE_DAYS = 90`; `MARGIN_KM = 2`; `MAXZOOM = 15`; `KEY_BASE = "field-maps/base/%<version>s.pmtiles"`; `KEY_CITY = "field-maps/cities/%<city_id>s/%<version>s-%<sha12>s.pmtiles"` (Q31); `EVENTS = %w[field_map.built field_map.failed]`; `AUDIT = %w[maintenance.field_map.build_requested maintenance.field_map.base_refresh_requested]` (Q19).
4. **`config/field_maps.yml`** (sem segredo, por ambiente): `pmtiles_bin: /usr/local/bin/pmtiles`; `builds_index_url` e `build_url_template` do Protomaps (conferidos na Task 12); `base_bbox`: produção/staging Brasil `[-74.1, -33.8, -28.8, 5.3]`, development Paraná `[-54.7, -26.8, -48.0, -22.5]` (Curitiba e Maringá; a base do Brasil não cabe numa prova local); `cache_dir: tmp/field_maps`. Lido por `FieldMaps::Config`.
5. **CLI:** `go-pmtiles` na versão que a prova técnica do campo fixou no contrato (seção "Prova técnica do campo"; Q12), com o SHA-256 do `.tar.gz` Linux x86_64 fixo no `Dockerfile` (estágio `base`, `ARG PMTILES_VERSION`/`ARG PMTILES_SHA256`, o SHA conferido na Task 11 contra a página de releases do `protomaps/go-pmtiles`); chamado só por `FieldMaps::Cli` com `Open3.capture2e(*argv)` (lista de argumentos, nunca shell).
6. **Caixa do município:** `vendor/ibge/2026-11-13/municipios_bbox.csv` (`ibge_code,min_lon,min_lat,max_lon,max_lat`, os 5.570 municípios, tirados da API de malhas do IBGE) + `vendor/ibge/2026-11-13/SOURCE` (origem, data, SHA-256), lido por `FieldMaps::Boundaries` (Q33). Margem: `Δlat = 2 km / 111.32`, `Δlon = 2 km / (111.32 · cos(lat_média))`, cortado em ±90/±180.
7. **Rotas:** em `scope "/field/v1"`: `get "reference"`, `get "map"`; em `scope "/territory", module: "territory_registry"`: `get "map"` → `maps#show`, `get "map/households"` → `maps#households`.
8. **Trilha:** `TerritoryRegistry::Trail::MAP_ACCESS = "territory_map"`.
9. **Dev:** worktree `apps/api/.claude/mod24c` (com `config/master.key`), branch `feat/mod-24c-field-app`, api do worktree na porta **3058**; migrações `db/city_migrate/20261113900001_add_field_locations.rb` (dump `define(version: 2026_11_13_900001)`) e `db/platform_migrate/20261113900001_create_field_maps.rb` (dump `define(version: 2026_11_13_900001)`).

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (os arquivos da task, por nome). Commits em inglês, Conventional Commits por extenso, sempre `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge e push só com autorização do usuário.
- **Testes só por `bin/test api -w mod24c <specs>`** (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. `--keep-db` só em rodadas de arquivo isolado. A branch é transversal (migração de cidade e de plataforma, `CityEncryption`, `Ledi`, 24a, `FieldAuthentication`, `spec/spec_helper.rb`, `Dockerfile`, initializers) → **suíte completa obrigatória antes do merge, pedida à sessão "Testes" na Task 18**.
- Migrações: dump à mão em `db/city_schema.rb`/`db/platform_schema.rb`; triggers no `db/city_triggers.sql`; **irreversíveis** (`down` → `ActiveRecord::IrreversibleMigration`; o `up` da de cidade termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`). O banco de dev só é migrado na prova no navegador, com autorização do usuário.
- Specs com tipo explícito (`type: :request`, `type: :model`, `type: :job`): `infer_spec_type_from_file_location!` está desligado. Nunca simule cidade inalcançável com URL real.
- **Nunca `Current.city =` solto**; o job de plataforma abre a cidade só com `CityConnection.with(city)`; `spec/architecture/current_city_assignment_spec.rb` roda nas Tasks 12 e 18.
- **Sigilo da coordenada:** latitude, longitude, `accuracy_m` e o corpo de `location` nunca em URL, log, evento (de domínio ou de plataforma), `Result.details`, mensagem de erro, `inspect` ou chave de cache. `filter_parameters` ganha `/\A(location|latitude|longitude|accuracy_m)\z/`. O arquivo de mapa e `field_maps` nunca têm dado de pessoa.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS`; toda guarda de imutabilidade que cobre a coluna nova aceita a re-cifra (`rota.reencrypting`).
- **`Platform.audit`/`MaintenanceAudit`:** os quatro nomes novos (Valor fixado 3) entram em `R18_PLATFORM_EVENT_NAMES` no mesmo commit que os publica; payload só com `city_id`, `version`, `size_bytes`, `sha256`, `error_code`.
- Schemas: `config/field_sync/*.json` copiados do `rotasaude/contracts` (`field-sync/`, commit da v1.2.0 entregue pela sessão do contracts antes da tag), presos por SHA-256; a Task 18 prende os SHA da tag `field-sync-v1.2.0`.
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`, ao encerrar a task.

## Review Focus

1. **"Gerar de novo" com o app baixando:** durante `building` (ou depois de um `failed`), `GET /field/v1/session` continua devolvendo o `map` publicado e `GET /field/v1/map` os bytes dele; o objeto antigo só sai do MinIO depois que a linha aponta o novo; um `Range` que continua contra o arquivo novo leva outro `ETag` (o app confere o SHA-256). Teste na Task 13.
2. **Coordenada na fronteira:** `-90`, `180`, `-25.42847123` (8 casas) passam; `-25.428471234`, `1.0e-09`, `"−25.4"` (texto), `true`, só `latitude`, `accuracy_m` `10000.05` ou `-1`, `source: "map"` numa visita e chave a mais recusam com o `path` exato; o valor recusado nunca aparece na resposta nem no log. Teste nas Tasks 1, 4 e 5.
3. **Reenvio depois de outra gravação:** `household` aceito com `version` 4, o posto grava a 5, o app reenvia o mesmo lote → `duplicate` com `version` **4** (do livro), não 5; idem `family` e `individual_registration`. Teste na Task 9.
4. **Revogação disputada:** o dono revoga (`user`) enquanto o app faz uma requisição que acabaria em `membership_ended` → a resposta é pelo motivo gravado de fato (`device_revoked`), nunca `device_suspended` por engano; aparelho suspenso cujo vínculo voltou continua revogado; aparelho substituído (`replaced`) responde `device_suspended`, nunca `device_revoked` (o tablet guarda a fila, Q17). Teste nas Tasks 7 e 8.
5. **`Range` torto:** `bytes=0-`, `bytes=-500`, `bytes=5-2`, início além do tamanho, `bytes=0-1,5-6`, `If-None-Match` com e sem aspas e `W/`, `If-Range` com o `ETag` atual e com outro → 206/416/200/304 como no Valor da Task 13 (Q34), sem ler o objeto inteiro para responder um intervalo. Teste na Task 13.
6. **Pessoa nova citada pelo cadastro** (Q26): família, visita e atividade que citam o `client_uuid` de um `individual_registration` com `new_patient` resolvem para o paciente criado; o mesmo `client_uuid` de um cadastro de paciente existente (`patient_id`) não é citável (`not_found`). Teste na Task 9.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod24c/`)
- Migrações: as duas do Valor fixado 9.
- `app/services/field_location.rb`; `app/services/field_sync/reference.rb`; `app/services/field_maps/{catalog,config,storage,cli,boundaries,delivery,request}.rb`.
- `app/models/{field_map,field_map_base}.rb`.
- `app/jobs/field_maps/{build_job,refresh_base_job}.rb`.
- `app/controllers/concerns/field_map_delivery.rb`; `app/controllers/field_sync/v1/{references,maps}_controller.rb`; `app/controllers/territory_registry/maps_controller.rb`.
- `app/graphql/maintenance/types/{field_map_type,field_map_base_type}.rb`; `app/graphql/maintenance/mutations/{build_field_map,refresh_field_map_base}.rb`.
- `config/field_maps.yml`; `vendor/ibge/2026-11-13/{municipios_bbox.csv,SOURCE}`; `spec/fixtures/field_maps/sample.pmtiles`.
- Specs listados em cada task.

**Modificar**
- Sincronização: `app/controllers/concerns/field_authentication.rb`, `app/services/field_sync/{catalog,schemas,json,upload}.rb`, `app/models/field_sync_receipt.rb`, `config/field_sync/*.json`, `spec/fixtures/field_sync/examples/*`.
- Aparelho: `app/commands/field_devices/register.rb`, `app/controllers/field_devices_controller.rb`.
- 24a: `app/commands/territory_registry/save_household.rb`, `app/services/territory_registry/{json,trail,access}.rb`.
- 24b: `app/commands/home_visits/record.rb`.
- Cifra e banco: `app/models/{household_version,home_visit}.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `db/platform_schema.rb`.
- LEDI: `app/services/ledi/fichas/{household_registration,home_visit_day}.rb`, `app/services/ledi/{household_registration_ficha,home_visit_day_ficha}.rb`.
- Plataforma: `app/services/platform_queue.rb`, `app/events/maintenance_audit.rb`, `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `Dockerfile`.
- Infra: `config/routes.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/spec_helper.rb`, `db/seeds.rb` (ou `lib/home_visit_crew.rb`).
- Specs existentes: `spec/models/field_device_schema_spec.rb` (motivo `replaced`), `spec/adr_pointers_spec.rb`, `spec/services/field_sync/{schemas,domains,upload}_spec.rb`, `spec/requests/field_sync/{session,uploads,round_trip}_spec.rb`, `spec/requests/field_devices_spec.rb`, `spec/requests/territory_registry/households_spec.rb`, `spec/services/ledi/fichas/{household_registration,home_visit_day}_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`.

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-24.19, F-24.20, F-24.21 e F-24.24 para In Progress (um comando `gh` por chamada, começando com `gh`; o api é superfície dos quatro, Q10).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod24c -b feat/mod-24c-field-app origin/main && cp config/master.key .claude/mod24c/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO com `grep` no worktree (os dois `define(version:`, `ls db/city_migrate | tail -1`, `ls db/platform_migrate | tail -1`, `ITEM_TYPES`, `REVOKED_REASONS`, `HTTP_ERRORS`, os dois CHECK de `field_devices` (`ck_field_devices_revoked_by`, `ck_field_devices_revoked_reason`), os códigos da `CityResolution`, os SHA do `schemas.rb`, a seção "Prova técnica do campo" do contrato com a versão do `go-pmtiles` (sem ela, as Tasks 11–12 esperam), `ALLOWED` do `Record`, `ciphered` do `rota_home_visit_guard`, o `reencryption_only` de `household_versions`, `VALID_RANGE`, `PlatformQueue::JOBS`, que `location` e `field_map` não existem em `db/*_schema.rb`). Diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod24c spec/services/field_sync spec/requests/field_sync spec/requests/field_devices_spec.rb spec/requests/territory_registry/households_spec.rb spec/services/ledi/fichas/household_registration_spec.rb spec/services/ledi/fichas/home_visit_day_spec.rb spec/adr_pointers_spec.rb spec/events/platform_event_payload_guard_spec.rb` → PASS (linha de base).

**Relato ao coordenador:** "api 24c · Task 0 · worktree mod24c em origin/main <hash>, pré-requisito conferido (diferenças: …), linha de base verde."

---

### Task 1: Prova técnica — coordenada no IDL e no dicionário, e `FieldLocation`

**Files:**
- Create: `app/services/field_location.rb`, `spec/services/field_location_spec.rb`, `spec/services/ledi/field_location_idl_spec.rb`

**Interfaces:**
- Produces: Valor fixado 1. `FieldLocation.parse(raw, sources:, prefix: "location") -> Result.ok(location: Hash | nil) | Result.fail(:invalid_location, details: { path: })` — `raw` `nil` → `ok(location: nil)`; `Hash` com todas as `REQUIRED`, nenhuma chave fora de `REQUIRED + OPTIONAL` e `accuracy_m` presente quando `source ∈ ACCURACY_REQUIRED_SOURCES` → `ok(location: { "latitude" => Float, "longitude" => Float, "accuracy_m" => Float (ausente se não veio), "source" => String })`; o resto → `fail`. `FieldLocation.decimals(number) -> Integer`. `FieldLocation.ledi(location) -> { latitude: Float, longitude: Float }` (`{}` com `nil`).

- [ ] **Step 1: Write the failing tests.** `spec/services/ledi/field_location_idl_spec.rb` (`type: :model`):

```ruby
RSpec.describe "Coordenada nas fichas 3 e 8 (LEDI 8.7.0)", type: :model do
  def field(klass, id) = klass::FIELDS.fetch(id)

  it "ficha 3: latitude 17 e longitude 18, double opcional" do
    cd = Br::Gov::Saude::Esusab::Ras::Cadastrodomiciliar::CadastroDomiciliarThrift
    expect(field(cd, 17)).to include(name: "latitude", type: ::Thrift::Types::DOUBLE, optional: true)
    expect(field(cd, 18)).to include(name: "longitude", type: ::Thrift::Types::DOUBLE, optional: true)
  end

  it "ficha 8: latitude 20 e longitude 21 no filho, double opcional" do
    vd = Br::Gov::Saude::Esusab::Ras::Visitadomiciliar::FichaVisitaDomiciliarChildThrift
    expect(field(vd, 20)).to include(name: "latitude", type: ::Thrift::Types::DOUBLE, optional: true)
    expect(field(vd, 21)).to include(name: "longitude", type: ::Thrift::Types::DOUBLE, optional: true)
  end

  it "o dicionário vendorizado pede o par, as faixas e até 8 casas (fcd e fvd)" do
    %w[dicionario-fcd.html dicionario-fvd.html].each do |doc|
      text = Ledi::Version.root.join("docs", doc).read.gsub(/<[^>]+>/, " ").squish
      expect(text).to include("obrigatório se o campo longitude for preenchido")
      expect(text).to include("limitados por -90 a 90 e apresentam até 8 casas decimais")
      expect(text).to include("limitados por -180 a 180 e apresentam até 8 casas decimais")
    end
  end
end
```

`spec/services/field_location_spec.rb` (`type: :model`): "aceita o par com 8 casas, inteiros e os extremos −90/90/−180/180"; "`nil` = sem coordenada"; **Review Focus 2**: `-25.428471234` → `location.latitude`; `1.0e-09` → `location.latitude`; `"-25.4"` e `true` → o campo; só `latitude` → `location.longitude`; sem `source` → `location.source`; `source: "gps"` sem `accuracy_m` → `location.accuracy_m` (Q3); `source: "map"` sem `accuracy_m` → aceito, sem a chave no resultado (Q3; o dashboard não manda); `accuracy_m` `10000.05`, `8.55` e `-1` → `location.accuracy_m`; `source: "map"` com `sources: FieldLocation::VISIT_SOURCES` → `location.source`; chave `altitude` → `location`; `[]` → `location`; nenhum `Result` de recusa contém o valor (`expect(result.details.to_s).not_to include("25.4")`); `ledi(nil) == {}`.
- [ ] **Step 2:** `bin/test api -w mod24c spec/services/field_location_spec.rb spec/services/ledi/field_location_idl_spec.rb` → FAIL (a de IDL pode já passar: é a prova).
- [ ] **Step 3: Implementar** `FieldLocation` (puro, sem banco). **Ler** as duas seções do dicionário (fcd `#15`/`#16` com "Máximo 10/11"; fvd `#20`/`#21`) e montar a tabela da prova: número do campo no IDL × no dicionário, tipo, faixa, casas, "Máximo" (dígitos), obrigatoriedade em par, o que a fvd diz da latitude ("do imóvel onde a visita foi realizada") e se há restrição com `statusTermoRecusa` (a fcd não tem; o ponto vai também com recusa, Q36).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add field location parsing pinned to the LEDI coordinate rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.
- [ ] **Step 6: Publicar** a tabela ao coordenador (o §4 do contrato cita os números da IDL, Q40): IDL 17/18 e 20/21 conferem; o dicionário da fcd numera `#15`/`#16`; "Máximo 10/11" = dígitos, coerente com 8 casas. Diferença do que a Q40 fixou: pare e relate.

**Relato ao coordenador:** "api 24c · Task 1 · IDL confere (fcd 17/18, fvd 20/21, double opcional); dicionário: par obrigatório, faixas, 8 casas (fcd numera #15/#16); FieldLocation pronto; tabela anexada para o §4."

---

### Task 2: Schemas v1.2.0, catálogo, filtro de log e ADR 0053

**Files:**
- Modify: `config/field_sync/*.json` (cinco alterados + `reference-response-v1.json` novo), `spec/fixtures/field_sync/examples/*`, `app/services/field_sync/{schemas,catalog}.rb`, `app/models/field_sync_receipt.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/services/field_sync/{schemas,domains}_spec.rb`, `spec/adr_pointers_spec.rb`

**Interfaces:**
- Consumes: o commit da v1.2.0 do `rotasaude/contracts` (sem tag) que a sessão do contracts entrega, com os SHA-256 dos seis arquivos.
- Produces: `FieldSync::Schemas::SHA256` com a chave nova `"reference-response-v1"`; Valor fixado 2 (`ITEM_TYPES` com `family`, `REVOKED_REASONS` com `replaced`, `SUSPENDED_REASONS`, `HTTP_ERRORS` com `device_suspended`, `map_not_found`, `range_not_satisfiable`); filtro `/\A(location|latitude|longitude|accuracy_m)\z/`; `VALID_RANGE = (1..53)`.

- [ ] **Step 1: Write the failing tests.** `schemas_spec.rb`: o `pinned` literal passa a ter seis chaves com os SHA da v1.2.0 (comentário "commit <hash> do contracts; a tag field-sync-v1.2.0 é presa na Task 18"); os exemplos do manifest novo validam. `domains_spec.rb` ganha (chaves de `location` do Task 2 do plano do contracts-repo):

```ruby
it "location: os limites do §4 e da Q3 são os do FieldLocation" do
  location = domains.fetch("location")
  expect(location).to include("decimals" => FieldLocation::DECIMALS)
  expect(location.fetch("latitude")).to eq("min" => -90, "max" => 90)
  expect(location.fetch("longitude")).to eq("min" => -180, "max" => 180)
  expect(location.fetch("accuracy_m")).to eq("min" => 0, "max" => 10_000, "decimals" => FieldLocation::ACCURACY_DECIMALS,
                                             "required_for" => FieldLocation::ACCURACY_REQUIRED_SOURCES)
  expect(location.fetch("sources")).to match_array(FieldLocation::SOURCES)
  expect(location.fetch("visit_sources")).to match_array(FieldLocation::VISIT_SOURCES)
end

it "códigos novos do 24c em errors.http e errors.item (Q8, Q9)" do
  expect(domains.dig("errors", "http")).to include("device_suspended", "map_not_found", "range_not_satisfiable")
  expect(domains.dig("errors", "item")).to include("invalid_location", "family_not_found", "cpf_or_reason_required")
end

it "os códigos de borda da cidade que o /field/v1 devolve estão em errors.http (Q8)" do
  expect(domains.dig("errors", "http")).to include("unknown_city", "city_suspended", "city_schema_behind")
end
```

Um exemplo novo em `catalog_spec`/`domains_spec`: `ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters).filter("location" => { "latitude" => 1 })` → `"[FILTERED]"`; `FieldSyncReceipt.new(item_type: "family")` é válido. O spec dos códigos de borda lê os literais `error: "…"` de `app/controllers/concerns/city_resolution.rb` (leitura do arquivo, não lista à mão); código novo na borda sem entrada em `errors.http` quebra o spec.
- [ ] **Step 2:** `bin/test api -w mod24c spec/services/field_sync/schemas_spec.rb spec/services/field_sync/domains_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Copiar os seis JSON e os exemplos do commit do contracts; `SHA256` com a sexta chave; catálogo; filtro; `VALID_RANGE`. Se um exemplo inválido do contracts passar no `json_schemer` do api (ou o contrário), parar e relatar (é divergência de schema, não do api).
- [ ] **Step 4:** os mesmos specs + `spec/adr_pointers_spec.rb spec/services/field_sync/upload_spec.rb spec/requests/field_sync` → PASS (os specs do 24b continuam verdes: a v1.2.0 é MINOR).
- [ ] **Step 5: Commit** `git commit -m "chore: copy field sync v1.2.0 schemas and extend field sync catalog" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 2 · schemas v1.2.0 do commit <hash> presos (6 SHA); a v1.1.0 do 24b segue verde; filtro de location; ADR até 53."

---

### Task 3: Esquema da cidade — `location` cifrada, livro com `family`, `result_version` e `result_patient_id`, motivo `replaced` (migração 900001)

**Files:**
- Create: `db/city_migrate/20261113900001_add_field_locations.rb`, `spec/models/field_location_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `app/models/{household_version,home_visit,field_sync_receipt}.rb`, `spec/models/field_device_schema_spec.rb`

**Interfaces:**
- Colunas: `household_versions.location text` e `home_visits.location text` (JSON cifrado, `serialize :location, coder: JSON` antes de `encrypts :location`, nulos); `field_sync_receipts.result_version integer` com CHECK `ck_field_sync_receipts_result_version` = `result_version IS NULL OR result_version >= 1`; `field_sync_receipts.result_patient_id uuid` (nulo; preenchido só no recibo de `individual_registration` que criou o paciente por `new_patient` — é o que a citação da Q26 resolve) com CHECK `ck_field_sync_receipts_result_patient` = `result_patient_id IS NULL OR item_type = 'individual_registration'`; `ck_field_sync_receipts_item_type` trocado pelo da lista com `family`.
- `field_devices` (Q17): `ck_field_devices_revoked_reason` trocado pelo da lista com `replaced`; `ck_field_devices_revoked_by` trocado para aceitar `revoked_by_user_id` com `admin`, `user` **e** `replaced` (a substituição é revogação por pessoa, com `by`).
- Triggers: `household_versions_reencryption_only` ganha `''location''`; `rota_home_visit_guard` ganha `'location'` em `ciphered` (a re-cifra passa; fora dela `location` nunca muda — a regra "só os três de retirada" já cobre).
- `CITY_KEYED_TARGETS`: `[HouseholdVersion, :location]`, `[HomeVisit, :location]` (comentário "ADR 0053 (módulo 24c): coordenada do domicílio e da visita").
- `inspect`/`as_json` de `HouseholdVersion` e `HomeVisit` continuam sem a coordenada.

- [ ] **Step 1: Write the failing test** (`type: :model`): "location grava e volta como Hash; no banco é texto cifrado (não contém o número)"; "versão de domicílio com location não muda (UPDATE recusado) salvo a re-cifra (`SET LOCAL rota.reencrypting = 'on'`)"; "visita: location não muda, nem na folha aberta; a re-cifra passa"; "livro aceita item_type family e recusa result_version 0"; "livro recusa result_patient_id em recibo que não é individual_registration"; "`inspect` não mostra a coordenada". Em `field_device_schema_spec.rb`: "revoked_reason replaced com revoked_by_user_id é aceito; membership_ended com revoked_by_user_id segue recusado".
- [ ] **Step 2:** `bin/test api -w mod24c spec/models/field_location_schema_spec.rb spec/models/field_device_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`add_column` ×4, `remove_check_constraint`/`add_check_constraint` do livro e dos dois de `field_devices`, `execute` do `city_triggers.sql`, `down` irreversível), o dump, os triggers, a cifra, os modelos.
- [ ] **Step 4:** os mesmos specs + `spec/models/home_visit_schema_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add encrypted location to household versions and home visits" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 3 · migração de cidade 20261113900001 irreversível; location cifrada nas duas tabelas; livro com family, result_version e result_patient_id; field_devices aceita replaced."

---

### Task 4: Ponto do domicílio no 24a (transversal: 24a)

**Files:**
- Modify: `app/commands/territory_registry/save_household.rb`, `app/services/territory_registry/json.rb`, `spec/requests/territory_registry/households_spec.rb`, `spec/commands/territory_registry/{save_household,save_family,replace_responsible}_spec.rb`

**Interfaces:**
- Consumes: `FieldLocation.parse(raw, sources: FieldLocation::SOURCES)`.
- `SaveHousehold.parse_fields`: chave `location` ausente → `:keep` (na atualização copia `household.latest_version.location`; no domicílio novo, `nil`); presente com `null` → `nil` (apaga); presente com objeto → o valor; recusa → `Result.fail(:invalid_location, details: { path: })` (422 no `PATCH`, pelo `FAILURE_STATUS` padrão). `persist!` grava `location:` na `HouseholdVersion`. `params_of` **não** muda (sem `location` = mantém, então `SaveFamily` e `ReplaceResponsible` preservam o ponto).
- `TerritoryRegistry::Json.household` ganha `location: version.location` (`null` sem ponto) em toda forma (lista, detalhe, download, `current`) — Q37.

- [ ] **Step 1: Write the failing tests.** Comando (`type: :model`): "domicílio novo com location grava a versão com o ponto e gera a ficha"; "PATCH só com location muda a versão (+1) e gera ficha de atualização"; "PATCH sem a chave mantém o ponto da versão anterior"; "PATCH com `location: null` apaga"; "SaveFamily depois de um ponto mantém o ponto"; "source gps (com accuracy_m) e map (sem accuracy_m, como o dashboard manda; Q3) aceitos"; "PATCH com o corpo inteiro do 24a + location (a correção do dashboard, Q13) grava só o ponto novo". Request (`type: :request`): "PATCH com latitude de 9 casas → 422 `{ error: invalid_location, path: location.latitude }`, sem o valor na resposta nem no log (`capture_log`)"; "GET /territory/households/:id e a lista trazem `location`"; os specs que fixam as chaves do `<household>` ganham `location`.
- [ ] **Step 2:** `bin/test api -w mod24c spec/requests/territory_registry/households_spec.rb spec/commands/territory_registry/save_household_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/territory_registry spec/commands/territory_registry spec/services/field_sync/changes_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: keep, replace or clear the household point on every household version" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 4 · location no SaveHousehold (ausente mantém, null apaga), no <household> de todas as rotas do 24a e do download; PATCH 422 invalid_location."

---

### Task 5: Coordenada da visita do app

**Files:**
- Modify: `app/commands/home_visits/record.rb`, `spec/commands/home_visits/record_spec.rb`

**Interfaces:**
- Consumes: `FieldLocation.parse(raw, sources: FieldLocation::VISIT_SOURCES)`.
- `HomeVisits::Record.call(…, origin: "field_app")` aceita `location` (objeto; ausente = sem coordenada); com `origin: "dashboard"` a chave segue desconhecida (`invalid_home_visit` com `path: "location"`: a visita do dashboard não tem coordenada, contrato §4). `null` na visita → `invalid_location` com `path: "location"` (Q3; pelo envio do app, o esquema já recusa e o `Upload` traduz, Task 9). `insert!` grava `location:`. A validação roda no `shape_error` (antes da transação), na mesma ordem das outras chaves.
- `HomeVisits::Json` (rotas da equipe) **não** expõe `location` (Q37: a coordenada da visita só vai à ficha).

- [ ] **Step 1: Write the failing tests** (`type: :model`): "visita do app com GPS grava a coordenada cifrada"; "sem location grava sem"; "`location: null` → invalid_location path location"; "GPS sem accuracy_m → invalid_location path location.accuracy_m"; "`source: map` → invalid_location path location.source"; "visita do dashboard com location → invalid_home_visit path location"; "o evento home_visit.recorded e o Result não têm a coordenada".
- [ ] **Step 2:** `bin/test api -w mod24c spec/commands/home_visits/record_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/requests/home_visits spec/services/field_sync/upload_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: record the GPS point of field app home visits" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 5 · visita do app aceita location só gps; dashboard segue sem; nada da coordenada em evento ou resposta da equipe."

---

### Task 6: Latitude e longitude nas fichas 3 e 8 (transversal: Ledi)

**Files:**
- Modify: `app/services/ledi/household_registration_ficha.rb`, `app/services/ledi/fichas/household_registration.rb`, `app/services/ledi/home_visit_day_ficha.rb`, `app/services/ledi/fichas/home_visit_day.rb`, `spec/services/ledi/fichas/{household_registration,home_visit_day}_spec.rb`

**Interfaces:**
- `HouseholdRegistrationFicha.values(version)` ganha `"location" => version.location`; `Fichas::HouseholdRegistration#to_thrift` faz `attrs.merge!(FieldLocation.ledi(@values["location"]))` **também com recusa** (Q36: o ponto é do imóvel, não da família).
- `HomeVisitDayFicha.visit_values` ganha `"location" => visit.location`; `Fichas::HomeVisitDay#child` faz `attrs.merge!(FieldLocation.ledi(row["location"]))`; o comentário do topo troca "Nunca … latitude, longitude" por "latitude/longitude só da visita do app com GPS".

- [ ] **Step 1: Write the failing tests** (`type: :model`), campo a campo no Thrift serializado e lido de volta: "ficha 3 com ponto: latitude/longitude iguais aos gravados (8 casas, `Float`)"; "ficha 3 sem ponto: os campos 17 e 18 ausentes (`nil`)"; "ficha 3 de atualização só do ponto: `fichaAtualizada` true e o ponto novo"; "ficha 3 com `statusTermoRecusa` e ponto: o endereço e os campos 17/18 (Q36)"; "ficha 8 com duas visitas, uma com GPS e outra sem: só a primeira leva 20/21"; "nenhum outro campo das fichas muda (comparação com o exemplo de referência do 24a/24b)".
- [ ] **Step 2:** `bin/test api -w mod24c spec/services/ledi/fichas/household_registration_spec.rb spec/services/ledi/fichas/home_visit_day_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/ledi` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: send household and home visit coordinates in LEDI fichas 3 and 8" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 6 · fichas 3 e 8 com latitude/longitude quando há ponto, conferidas campo a campo; gate de go-live: PEC real com coordenada."

---

### Task 7: Revogação automática e por substituição respondem `device_suspended` (6A, Q17)

**Files:**
- Modify: `app/controllers/concerns/field_authentication.rb`, `spec/requests/field_sync/session_spec.rb`

**Interfaces:**
- `authenticate_device!`: revogado → `refuse_device(revoked_error(device))`; `revoked_error(device) = FieldSync::Catalog::SUSPENDED_REASONS.include?(device.revoked_reason) ? "device_suspended" : "device_revoked"`. Revogação na borda: chama `FieldDevices::Revoke` e responde `revoked_error(device.reload)` (o motivo gravado de fato, mesmo que outra requisição tenha revogado antes). `invalid_token` não muda. Comentário do topo atualizado (6A muda a Q7 só no código do 401; Q17: `replaced` entra no `device_suspended`).

- [ ] **Step 1: Write the failing tests** (`type: :request`): "revogado pelo dono → 401 device_revoked"; "revogado pelo admin → device_revoked"; "revogado por membership_ended → 401 device_suspended com WWW-Authenticate: Bearer"; "user_disabled → device_suspended"; "substituído (replaced) → device_suspended (Q17)"; "vínculo perdido agora → revoga membership_ended e responde device_suspended"; **Review Focus 4**: "o dono revoga (user) entre o carregamento e a revogação da borda (stub de `FieldDevices::Revoke` devolvendo `device_already_revoked` depois de gravar `user`) → device_revoked"; "aparelho suspenso cujo vínculo voltou continua device_suspended"; "a resposta nunca diz qual dos três motivos (replaced, membership_ended, user_disabled)".
- [ ] **Step 2:** `bin/test api -w mod24c spec/requests/field_sync/session_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/requests/field_sync spec/services/field_sync/domains_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: answer device_suspended for automatically revoked field devices" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 7 · 401 device_suspended (replaced/membership_ended/user_disabled) × device_revoked (user/admin), pelo motivo gravado."

---

### Task 8: Substituir aparelho — `replaces_device_id`

**Files:**
- Modify: `app/commands/field_devices/register.rb`, `app/controllers/field_devices_controller.rb`, `spec/requests/field_devices_spec.rb`, `spec/commands/field_devices/register_spec.rb` (criar se não houver)

**Interfaces:**
- `FieldDevices::Register.call(user:, name:, replaces_device_id: nil) -> ok(device:, token:, replaced:)`; ordem: nome → `replaces_device_id` (`nil` ou UUID, senão `fail(:invalid_request, path: "replaces_device_id")`) → trava do usuário → `device_owner?` (`cbo_not_allowed`) → aparelho substituído (`FieldDevice.active.find_by(id:, user_id: user.id)`, senão `fail(:invalid_request, path: "replaces_device_id")`; só aparelho ativo, Q16 — o revogado automaticamente é cadastrado de novo pelo cadastro simples) → limite contando os ativos **menos** o substituído → `FieldDevices::Revoke.call(replaced, by: user, reason: "replaced")` (Q17) → cria o novo, tudo na mesma transação; uma falha desfaz a revogação.
- `POST /field_devices` lê `replaces_device_id` do corpo; step-up e `home_visits` como antes; resposta igual (`{ device, token, qr_payload }`, `no-store`). `GET /field_devices` e `.../admin` mostram `revoked_reason: "replaced"` no antigo (o dashboard rotula, Task 6 do dashboard).

- [ ] **Step 1: Write the failing tests.** Comando (`type: :model`): "com 3 ativos, substituir um → 201, o antigo revogado (replaced, by: o dono) e 3 ativos"; "aparelho de outro usuário, revogado (inclusive por membership_ended) ou inexistente → invalid_request path replaces_device_id, nada revogado"; "texto que não é UUID → invalid_request"; "falha ao criar desfaz a revogação"; "eventos: field_device.revoked (reason replaced) e field_device.registered, só ids". Request: "sem step-up → 401 mfa_required e nada revogado"; "o token só no 201"; "o log não tem o token"; **Review Focus 4**: "o token do aparelho substituído recebe 401 device_suspended em `GET /field/v1/session`".
- [ ] **Step 2:** `bin/test api -w mod24c spec/commands/field_devices/register_spec.rb spec/requests/field_devices_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: replace a field device and revoke the old one in one transaction" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 8 · replaces_device_id com step-up; revoga (replaced) e cria juntos; limite sem o antigo; só aparelho ativo se substitui (Q16); o antigo recebe device_suspended (Q17)."

---

### Task 9: Envio — item `family`, `family_client_uuid`, citação pelo cadastro, `invalid_location` do esquema e `version` no resultado

**Files:**
- Modify: `app/services/field_sync/upload.rb`, `spec/services/field_sync/upload_spec.rb`, `spec/requests/field_sync/{uploads,round_trip}_spec.rb`

**Interfaces:**
- Consumes: `TerritoryRegistry::SaveFamily.call(household:, params:, by:, family:)`; `FieldSyncReceipt#result_version` e `#result_patient_id` (Task 3).
- **Erro de esquema do item (Q2):** a conferência contra `upload-request-v1.json` que hoje devolve `invalid_request` (`upload.rb:9-11`, `:418-420`) passa a olhar o primeiro erro do `json_schemer`: `data_pointer` igual a `/body/location` ou começando por `/body/location/` → `rejected invalid_location` com `path` = o ponteiro sem `/body/`, com `/` trocado por `.` (`/body/location/latitude` → `location.latitude`; `/body/location` → `location`); qualquer outro → `invalid_request` como na Q31 do 24b. Nenhum valor recusado na resposta.
- `RESULT_TYPES["family"] = "Family"`; `dispatch` → `family(body, context)`:
  - `registry_refusal` (feature e ACS) → `household_id` | `household_client_uuid` (pelo livro; não aceito → `not_found` path `household_client_uuid`; domicílio fora de `visible_micro_area_ids` → `not_found` path `household_id`) → `responsible.patient_client_uuid` pelo livro (`cite_patient`, abaixo; não aceito → `not_found` path `responsible.patient_client_uuid`), o corpo vai ao `SaveFamily` com `responsible.patient_id` (forma aninhada, Q1) → `family_id` presente: `Family.find_by(id:, household_id: household.id)` ou `rejected family_not_found` path `family_id` → `SaveFamily` (sem regra nova sobre o cadastro do responsável: o ativo no domicílio, se houver, entra na família, Q25); `stale_version` → `stale_household(household)` (`id` = domicílio, Q4); aceito → `accepted(result.payload[:family].id, version: result.payload[:household].version)`. A versão nova do cadastro do responsável que o `SaveFamily` grava não volta no resultado (Q35).
- **Citação de pessoa (Q26):** `cite_patient(client_uuid)` resolve, nesta ordem, o recibo aceito com `item_type: "patient"` (→ `result_id`) ou o recibo aceito com `item_type: "individual_registration"` e `result_patient_id` não nulo (→ `result_patient_id`); senão `not_found`. Vale para todo `patient_client_uuid` (cadastro individual, família, visita, participante da atividade). O recibo do `individual_registration` grava `result_patient_id` = `result.payload[:registration].patient_id` **só** quando o corpo aceito trazia `new_patient`.
- `cite` ganha o tipo `family` (`family_client_uuid` → `family_id`, `item_type: "family"`), usado por `home_visit` e `individual_registration`.
- `accepted(id, version: nil, replayed: false)`; `household` → `version: result.payload[:household].version`; `individual_registration` → `version: result.payload[:registration].version`. O recibo grava `result_version: outcome[:version]`. `ledger` devolve `{ status: "duplicate", id:, version: receipt.result_version }`. `result_json` põe `"version"` no `accepted` sempre (cadastros) e no `duplicate` quando o recibo a tem: recibo anterior à coluna sai sem `version` (Q5; o esquema da v1.2.0 aceita, o app marca a linha para recarga).

- [ ] **Step 1: Write the failing tests** (`type: :model`, respostas validadas contra `upload-response-v1`): "lote `patient` → `individual_registration` (patient_client_uuid, household_id) → `family` (responsible.patient_client_uuid) → `individual_registration` de outro morador (family_client_uuid) → quatro accepted, family com version = versão nova do domicílio"; **Review Focus 6** (Q26): "lote `individual_registration` com `new_patient` (com nome social) → `family` com `responsible.patient_client_uuid` = o `client_uuid` do cadastro → `home_visit` com `patient_client_uuid` = o mesmo → três accepted, todos com o paciente criado"; "cadastro com `patient_id` existente não é citável pelo `client_uuid` → not_found path patient_client_uuid"; "household accepted com version; registration accepted com version"; **Review Focus 3**: "household aceito com version 4, o posto grava a 5, reenvio → duplicate com version 4"; idem `family`; "recibo sem result_version (gravado antes da migração) → duplicate sem version, válido no esquema (Q5)"; "family de TACS → cbo_not_allowed"; "family_id de outro domicílio → family_not_found path family_id"; "family com version vencida → stale_version com id do domicílio, current = <household> (com families e location) e current_version"; "responsible.patient_client_uuid não aceito → not_found com o path"; "family_client_uuid não aceito na visita → not_found path family_client_uuid"; "família nova sem cadastro ativo do responsável no domicílio é aceita (Q25)"; **Q2**: "household com latitude de 9 casas → rejected invalid_location path location.latitude"; "visita com `location: null` → invalid_location path location"; "visita GPS sem accuracy_m → invalid_location path location"; "chave desconhecida fora de location → invalid_request (Q31 do 24b)"; "item family recusado não vai ao livro e o reenvio corrigido é aceito"; "home_visit não leva version". `round_trip_spec`: o ciclo família nova + download traz a família.
- [ ] **Step 2:** `bin/test api -w mod24c spec/services/field_sync/upload_spec.rb spec/requests/field_sync/uploads_spec.rb spec/requests/field_sync/round_trip_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/territory_registry spec/commands/territory_registry` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: accept offline families and return record versions in field uploads" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 9 · item family (aninhado, Q1), family_client_uuid, patient_client_uuid pelo cadastro com new_patient (Q26), erro de esquema em location → invalid_location (Q2), version no accepted/duplicate pelo livro (result_version; duplicate legado sem version, Q5)."

---

### Task 10: Referência offline — `GET /field/v1/reference` e `reference_version` na sessão

**Files:**
- Create: `app/services/field_sync/reference.rb`, `app/controllers/field_sync/v1/references_controller.rb`, `spec/services/field_sync/reference_spec.rb`, `spec/requests/field_sync/reference_spec.rb`
- Modify: `app/services/field_sync/json.rb`, `config/routes.rb`, `spec/requests/field_sync/session_spec.rb`

**Interfaces:**
- `FieldSync::Reference.build(user:, city:) -> Hash` na forma do §3.5, com `reference_version` = `SHA-256(Signatures::Jcs.dump(corpo sem reference_version))`:
  - `registration_dictionaries` = `{ version:, domains: { <nome> => [{ code, label }] }, sections: { <nome> => [<campo>] } }` de `TerritoryRegistry::Dictionary::DATA`, na forma do gerador do dashboard `ledi-dictionaries.mjs` (domínios como listas `[{ code, label }]` na ordem do YAML; `<campo>` = `key`, `ledi`, `type`, e `domain`/`max`/`min`/`max_items`/`required`/`fields` quando houver; ordem do YAML) (Q27);
  - `string_domains` = `{ uf:, tipo_logradouro: }` → `[{ code, label }]`;
  - `neighborhoods` = `Neighborhood.active_neighborhoods.order(:name, :id)` → `{ id, name }`;
  - `health_units` = `HealthUnit.active_units` → `{ id, cnes, name }` (`cnes` `null` sem CNES; Q6);
  - `team_professionals` = para cada equipe de `FieldSync::Json.field_teams(user)`, `CollectiveActivities::Access.professional_memberships(team)` **só com `in_team`** (vínculo nesta equipe), `{ team_id, professional_id, name, cbo, has_cns }`, ordem por equipe e nome (Q6);
  - `home_visit_domains` = `HomeVisits::Json.domains`; `collective_activity_domains` = `CollectiveActivities::Json.domains` (um bloco; Q7).
- `FieldSync::Reference.version(user:, city:) -> String`: `Rails.cache.fetch("field_sync/reference_version/#{city.id}/#{teams_digest}", expires_in: REFERENCE_CACHE)` guardando **só o hex** (nenhum nome no banco de cache; Q29); `teams_digest` = SHA-256 dos ids de equipe ordenados.
- `GET /field/v1/reference` (borda da `FieldAuthentication`, sem query, sem trilha) → `Reference.build`; `FieldSync::Json.session` ganha `reference_version: Reference.version(...)` (o `map` entra na Task 13).

- [ ] **Step 1: Write the failing tests.** Serviço (`type: :model`): "a referência valida contra reference-response-v1"; "dicionários e domínios de texto batem com o YAML, em ordem, sem cópia à mão (domínio sempre lista, nunca objeto)"; "só bairros e unidades ativos; unidade sem CNES com `cnes: null`"; "profissionais só das equipes do aparelho, com o CBO do vínculo nelas e `has_cns` igual ao das opções da atividade"; "mesmo conteúdo → mesmo reference_version; um bairro novo muda"; "o cache guarda só o hex" (`allow(Rails.cache).to receive(:fetch)` e conferir o bloco devolve `String` de 64 hex). Request: "GET reference 200 sem trilha (nenhum clinical_record.viewed)"; "com query → 422 invalid_request"; "sem token → 401"; "session traz reference_version igual ao da referência (sem cache em test)".
- [ ] **Step 2:** `bin/test api -w mod24c spec/services/field_sync/reference_spec.rb spec/requests/field_sync/reference_spec.rb spec/requests/field_sync/session_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`field_teams` deixa de ser privado ou ganha um irmão público).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: serve the field app offline reference with a content version" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 10 · GET /field/v1/reference (domínios em lista, Q27; has_cns, Q6; bloco único da atividade, Q7) e reference_version (JCS + SHA-256, cache só do hex, Q29)."

---

### Task 11: Plataforma — `field_maps`, `field_map_bases`, armazenamento e CLI

**Files:**
- Create: `db/platform_migrate/20261113900001_create_field_maps.rb`, `app/models/{field_map,field_map_base}.rb`, `app/services/field_maps/{catalog,config,storage,cli}.rb`, `config/field_maps.yml`, `spec/models/field_map_schema_spec.rb`, `spec/services/field_maps/storage_spec.rb`, `spec/services/field_maps/cli_spec.rb`, `spec/fixtures/field_maps/sample.pmtiles`
- Modify: `db/platform_schema.rb`, `Dockerfile`, `spec/spec_helper.rb`

**Interfaces:**
- `field_maps` (`FieldMap < PlatformRecord`): `city_id uuid NOT NULL` único, FK `cities`; `state` CHECK `IN ('building','ready','failed')`; `error_code` CHECK `IS NULL OR IN (<ERROR_CODES>)` e `(state = 'failed') = (error_code IS NOT NULL)`; **arquivo publicado** `version date`, `size_bytes bigint`, `sha256 string(64)` (CHECK `~ '^[0-9a-f]{64}$'`), `storage_key string`, `built_at datetime` — os cinco nulos juntos ou preenchidos juntos (`ck_field_maps_published`); `requested_at`, `created_at`, `updated_at`. O `state` descreve a última geração; o publicado continua servido em `building`/`failed` (Q30). `FieldMap#published?`, `#manifest -> { version: "YYYY-MM-DD", size_bytes:, sha256: } | nil`, `FieldMap.published_for(city_id) -> FieldMap | nil`, `inspect` só `city_id`/`state`.
- `field_map_bases` (`FieldMapBase < PlatformRecord`): `version date` único, `state` CHECK `IN ('copying','ready','failed')`, `error_code`, `source_url string`, `storage_key`, `size_bytes`, `sha256` (preenchidos juntos com `ready`), `ready_at`, timestamps. `FieldMapBase.current` = a `ready` de maior `version`; `#age_days(today)`.
- `FieldMaps::Storage`: `service = ActiveStorage::Blob.service`; `upload(key, path)` (stream do arquivo; `checksum: nil`); `chunk(key, range) -> String` (`service.download_chunk`); `fetch_to(key, path)` (download em blocos para o arquivo); `delete(key)`; `exist?(key)`. Sem `ActiveStorage::Blob` (nada no `active_storage_blobs`).
- `FieldMaps::Config.{pmtiles_bin, builds_index_url, build_url_template, base_bbox, cache_dir}` (Valor fixado 4).
- `FieldMaps::Cli.extract(input:, output:, bbox:, maxzoom:) -> Result.ok | Result.fail(:extract_failed)` = `Open3.capture2e(bin, "extract", input, output, "--bbox=#{bbox.join(',')}", "--maxzoom=#{maxzoom}")`; saída do processo nunca no log (só o código de saída).
- `Dockerfile` (estágio `base`): baixa o `go-pmtiles` `${PMTILES_VERSION}` (a versão da prova técnica do campo, Q12) Linux x86_64, confere `${PMTILES_SHA256}` com `sha256sum -c`, instala em `/usr/local/bin/pmtiles`.
- `spec/spec_helper.rb`: `config.filter_run_excluding :pmtiles_cli unless ENV["FIELD_MAPS_PMTILES_IT"].to_s.strip != ""`.

- [ ] **Step 1: Write the failing tests.** `field_map_schema_spec.rb` (`type: :model`): "uma linha por cidade"; "failed exige error_code e só ele"; "publicado é tudo ou nada"; "sha256 com 64 hex"; "`manifest` nil sem publicado e com a forma do §3.2 com publicado". `storage_spec.rb`: com o Disk de test, "upload + chunk(0..6) devolve `PMTiles`; chunk do fim; delete". `cli_spec.rb`: "monta o argv sem shell" (`expect(Open3).to receive(:capture2e).with("/usr/local/bin/pmtiles", "extract", …)`); "código de saída ≠ 0 → extract_failed e o log não tem a saída"; integração `:pmtiles_cli`: "recorta o fixture" (fora da suíte).
- [ ] **Step 2:** `bin/test api -w mod24c spec/models/field_map_schema_spec.rb spec/services/field_maps/storage_spec.rb spec/services/field_maps/cli_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** O fixture `sample.pmtiles` (poucos KB, PMTiles v3, cabeçalho `PMTiles` + byte 3) sai do CLI da imagem nova (`docker compose build api`, depois `pmtiles extract` de uma caixa minúscula de uma build pública num diretório vazio do scratchpad; conferir com `pmtiles show` antes de mover para `spec/fixtures/`; o `sample.pmtiles` da prova do campo serve, se couber em poucos KB). Usar a versão do CLI fixada no contrato pela prova (Q12), conferir na página de releases o SHA-256 do binário Linux e anotar no relato.
- [ ] **Step 4:** os mesmos specs + `spec/architecture/database_config_parity_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add platform field map tables, object storage access and the pmtiles CLI" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 11 · field_maps (publicado separado do estado, Q30) e field_map_bases; MinIO por chave; pmtiles <versão da prova> na imagem (sha …); fixture de <n> KB."

---

### Task 12: Jobs do mapa — base e recorte por município

**Files:**
- Create: `app/jobs/field_maps/{refresh_base_job,build_job}.rb`, `app/services/field_maps/{boundaries,request}.rb`, `vendor/ibge/2026-11-13/{municipios_bbox.csv,SOURCE}`, `spec/jobs/field_maps/{refresh_base_job,build_job}_spec.rb`, `spec/services/field_maps/boundaries_spec.rb`
- Modify: `app/services/platform_queue.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
- `FieldMaps::Boundaries.bbox(ibge_code) -> [min_lon, min_lat, max_lon, max_lat] | nil`; `.with_margin(bbox, km: MARGIN_KM) -> [..4]` (Valor fixado 6).
- `FieldMaps::Request.build!(city) -> FieldMap` (upsert `state: building`, `error_code: nil`, `requested_at`; já `building` → não enfileira de novo) e `.refresh_base! -> FieldMapBase | nil` (nenhuma base `copying` → enfileira); quem chama é o maintenance (Task 15).
- `FieldMaps::RefreshBaseJob` (plataforma, `queue_as :default`): lê o índice de builds (`builds_index_url`) e pega a mais recente; cria `FieldMapBase(version:, state: copying, source_url:)`; `Cli.extract(input: <URL da build>, output: <cache_dir>/base/<version>.pmtiles, bbox: base_bbox, maxzoom: 15)`; SHA-256; `Storage.upload(KEY_BASE)`; `ready`; apaga do MinIO e do cache local a base `ready` anterior. Falha → `failed` com `base_missing` (índice/URL) ou `extract_failed`. Sem job recorrente: o maintenance avisa quando a base passa de 90 dias (Q23).
- `FieldMaps::BuildJob.perform(city_id)` (plataforma): sem `FieldMapBase.current` → `failed base_missing` (Q24: o maintenance não oferece "Gerar" sem base, mas o api responde assim se chamado); `ibge_code` por `CityConnection.with(city) { CityProfile.current&.ibge_code }`, sem caixa → `failed boundary_missing`; base local em `<cache_dir>/base/<version>.pmtiles` (cache local do worker, baixado do MinIO se faltar e conferido pelo SHA-256; Q32); `Cli.extract` → SHA-256 → igual ao publicado: só `state: ready`, `built_at`; diferente: `Storage.upload(KEY_CITY)`, a linha passa a apontar o novo (`version` = data da base) e **depois** `Storage.delete` do antigo; `Platform.audit("field_map.built", city_id:, version:, size_bytes:, sha256:)` ou `("field_map.failed", city_id:, error_code:)`. Qualquer exceção não prevista → `failed extract_failed` e o evento; arquivos temporários sempre apagados.
- `PlatformQueue::JOBS` += `FieldMaps::BuildJob FieldMaps::RefreshBaseJob`; `R18_PLATFORM_EVENT_NAMES` += `field_map.built field_map.failed`.

- [ ] **Step 1: Write the failing tests** (`type: :job`, CLI simulado copiando o fixture para `output:`, Disk de test): "base: índice → copying → ready com sha e chave; a anterior sai do storage"; "índice indisponível → failed base_missing"; "build: sem base → failed base_missing e field_map.failed"; "IBGE ausente ou fora do CSV → boundary_missing"; "caixa com margem de 2 km passada ao CLI (`--bbox` exato) e `--maxzoom=15`"; "recorte → ready, manifest = tamanho e SHA do fixture, field_map.built só com city_id/version/size_bytes/sha256"; **Review Focus 1**: "regerar com SHA diferente: o publicado só muda no fim e o objeto antigo só sai depois"; "regerar com o mesmo SHA não sobe nada"; "CLI falha → failed extract_failed, o publicado anterior continua"; "nenhum evento ou log com dado de pessoa". `boundaries_spec`: "Curitiba (4106902) e Maringá (4115200) têm caixa"; "margem em latitude e longitude"; "o CSV bate com o SHA do SOURCE".
- [ ] **Step 2:** `bin/test api -w mod24c spec/jobs/field_maps spec/services/field_maps/boundaries_spec.rb spec/events/platform_event_payload_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Gerar o CSV a partir da API de malhas do IBGE (metadados de cada município, a região limítrofe) num diretório vazio do scratchpad, com um script fora dele; conferir à mão três municípios; mover para `vendor/ibge/2026-11-13/` com o SHA no `SOURCE`. Conferir e fixar no `config/field_maps.yml` a URL do índice de builds e o padrão da URL da build do Protomaps.
- [ ] **Step 4:** os mesmos specs + `spec/architecture/current_city_assignment_spec.rb spec/services/platform_queue_spec.rb spec/config/solid_queue_configuration_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: build per city field maps from a stored base with the pmtiles CLI" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 12 · RefreshBaseJob (extrai a caixa do Brasil da build mais recente, Q23) e BuildJob (caixa IBGE vendorizada + 2 km, Q33; base em cache local conferido, Q32; chave com sha12, Q31); field_map.built/failed no R18."

---

### Task 13: Entrega do mapa — `GET /field/v1/map`, `GET /territory/map` e `map` na sessão

**Files:**
- Create: `app/services/field_maps/delivery.rb`, `app/controllers/concerns/field_map_delivery.rb`, `app/controllers/field_sync/v1/maps_controller.rb`, `app/controllers/territory_registry/maps_controller.rb` (`show`), `spec/services/field_maps/delivery_spec.rb`, `spec/requests/field_sync/map_spec.rb`, `spec/requests/territory_registry/map_spec.rb`
- Modify: `app/services/field_sync/json.rb`, `app/services/territory_registry/access.rb`, `config/routes.rb`, `spec/requests/field_sync/session_spec.rb`

**Interfaces:**
- `FieldMaps::Delivery.plan(map:, range:, if_none_match:, if_range: nil) -> Delivery::Plan(status:, headers:, offset:, length:)` (Q34): `ETag` = `"<sha256>"`; `If-None-Match` que contém o ETag (aspas e `W/` tolerados, `*` vale) → 304; sem `Range` → 200 inteiro; `If-Range` presente e diferente do ETag atual → ignora o `Range` e responde 200 inteiro; um intervalo só por requisição, `bytes=a-b` / `bytes=a-` / `bytes=-n` válido → 206 com `Content-Range: bytes a-b/size`; inválido ou além do fim → 416 `{ error: "range_not_satisfiable" }` com `Content-Range: bytes */size`; mais de um intervalo → 200 inteiro; sempre `Accept-Ranges: bytes`, `Content-Type: application/vnd.pmtiles`, `Cache-Control: private, no-cache`.
- `FieldMaps::Delivery::Body.new(key, offset, length)`: `each` rende pedaços de `MAP_CHUNK` por `Storage.chunk` (Rack body; nada de ler o objeto inteiro).
- Concern `FieldMapDelivery#render_field_map(map)`: sem publicado → 404 `{ error: "map_not_found" }`; senão aplica o `Plan` (`self.response_body = Body…`).
- `FieldSync::V1::MapsController#show` (borda da `FieldAuthentication`, sem query) e `TerritoryRegistry::MapsController#show` (sessão, `territory_registry`, `TerritoryRegistry::Access.reads_households?(user)` senão 403 `missing_role`; Q39), os dois com `FieldMap.published_for(Current.city.id)`; nenhuma trilha.
- `TerritoryRegistry::Access.reads_households?(user) -> Boolean` (Q39): o usuário lê ao menos uma microárea — alguma equipe ativa dele com `visible_micro_area_ids` ou `field_reader_micro_area_ids` não vazios (o interruptor `home_visits` é lido dentro, como no `HouseholdsController#visible?`).
- `FieldSync::Json.session` ganha `map: FieldMap.published_for(city.id)&.manifest`.

- [ ] **Step 1: Write the failing tests.** `delivery_spec.rb` (`type: :model`), **Review Focus 5**: `bytes=0-` → 206 inteiro; `bytes=-500` → os 500 últimos; `bytes=5-2` → 416; início ≥ tamanho → 416 com `bytes */size`; `bytes=0-1,5-6` → 200; `If-None-Match: "<sha>"`, `<sha>` sem aspas, `W/"<sha>"` e `*` → 304; outro → 200; `Range: bytes=100-` com `If-Range: "<sha>"` → 206; com `If-Range` de outro SHA → 200 inteiro (o app recomeça; Q34). `map_spec.rb` (field, `type: :request`): "sem mapa → 404 map_not_found"; "200 com Content-Type application/vnd.pmtiles, ETag e o corpo igual ao fixture"; "Range → 206 com os bytes certos"; "sem token → 401 invalid_token"; "suspenso → 401 device_suspended"; "home_visits desligado → 403 feature_disabled"; "query → 422"; **Review Focus 1**: "com a linha em building (publicado antigo), session.map e /map seguem o antigo". `territory_registry/map_spec.rb`: "ACS, enfermeiro e TACS atribuído leem; municipal_admin sem equipe → 403"; "sem sessão → 401"; "nenhum clinical_record.viewed". `session_spec.rb`: "map nulo sem mapa e `{ version, size_bytes, sha256 }` com mapa, validando contra session-response-v1".
- [ ] **Step 2:** `bin/test api -w mod24c spec/services/field_maps/delivery_spec.rb spec/requests/field_sync/map_spec.rb spec/requests/territory_registry/map_spec.rb spec/requests/field_sync/session_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: serve city field maps with ranges and etags to devices and the dashboard" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 13 · /field/v1/map e /territory/map (Range 206, If-Range, 304, 416 range_not_satisfiable, 404 map_not_found; Q8, Q34); session.map; reads_households?(user) (Q39)."

---

### Task 14: Pontos da microárea no dashboard — `GET /territory/map/households`

**Files:**
- Modify: `app/controllers/territory_registry/maps_controller.rb` (`households`), `app/services/territory_registry/trail.rb`
- Create: `spec/requests/territory_registry/map_households_spec.rb`

**Interfaces:**
- `GET /territory/map/households?micro_area_id=` (só essa chave na query; UUID ou 422 `invalid_request`; microárea inexistente → 404; fora de `visible?` — o mesmo do `HouseholdsController#visible?`, inclusive o leitor de campo — → 403 `out_of_scope`): `{ map: FieldMap.published_for(Current.city.id)&.manifest, items: [ { id, location, address: { street, number }, families_count, version } ] }` — `map` = `null | { version, size_bytes, sha256 }`, a mesma forma do `map` da sessão (Q14: o dashboard não sonda o arquivo); `items` = os domicílios **ativos** da microárea cuja última versão tem `location` (Q38), `families_count` = famílias não `moved_out`; ordem por `id`; sem paginação.
- Trilha: `TerritoryRegistry::Trail.map_households_listed!(user:, micro_area_id:, household_ids:)` → `clinical_record.viewed` `access: "territory_map"`, uma por requisição, só quando `items` não é vazio.
- A lógica `visible?` passa a um método do `BaseController` (ou de `Access`) usado pelos dois controladores, sem mudar o comportamento do 24a.

- [ ] **Step 1: Write the failing tests** (`type: :request`): "map nulo sem mapa publicado; `{ version, size_bytes, sha256 }` com mapa, igual ao da sessão (Q14)"; "só domicílios ativos com ponto, com street/number e families_count"; "trilha com exatamente os household_ids e access territory_map"; "lista vazia não grava trilha"; "microárea alheia → 403 out_of_scope; inexistente → 404; sem micro_area_id → 422"; "?cpf= → 422"; "a resposta não traz complemento, CEP, telefone nem responsável"; "log sem a coordenada".
- [ ] **Step 2:** `bin/test api -w mod24c spec/requests/territory_registry/map_households_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec + `spec/requests/territory_registry/households_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: list micro area household points for the dashboard map with an audit trail" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 14 · /territory/map/households com map (Q14) e trilha territory_map; só ativos (Q38); corrigir o ponto é o PATCH do 24a com o corpo inteiro (Task 4, Q13)."

---

### Task 15: Maintenance — situação do mapa e "gerar de novo" (GraphQL no api)

**Files:**
- Create: `app/graphql/maintenance/types/{field_map_type,field_map_base_type}.rb`, `app/graphql/maintenance/mutations/{build_field_map,refresh_field_map_base}.rb`, `spec/requests/maintenance/field_maps_spec.rb`
- Modify: `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
SDL — **o mesmo que o plano do maintenance espera** (Global Constraints dele; a Task 0 do maintenance para se for outro):

```graphql
type FieldMap { citySlug: String! state: String! version: ISO8601Date sizeBytes: BigInt builtAt: ISO8601DateTime errorCode: String }
type FieldMapBase { version: ISO8601Date! ageDays: Int! }
fieldMaps: [FieldMap!]!                                   # Query
fieldMapBase: FieldMapBase                                # Query; null = base nunca copiada
buildFieldMap(citySlug: String!): BuildFieldMapPayload    # Mutation; { ok errors { path message } fieldMap }
refreshFieldMapBase: RefreshFieldMapBasePayload           # Mutation; { ok errors { path message } fieldMapBase }
```

- Query `fieldMaps` (plataforma, sem abrir nenhuma cidade): **uma entrada por linha de `field_maps`** (cidade sem linha não aparece; o maintenance junta com `cities` e mostra "sem mapa"); `citySlug` pelo `cities` da plataforma; `state`/`errorCode` `String` (valor novo aparece cru, Q21); `version`/`sizeBytes`/`builtAt` do arquivo publicado (nulos sem publicado, Q30); `sizeBytes` no escalar `GraphQL::Types::BigInt` (Q21). Query `fieldMapBase`: a `FieldMapBase.current` com `ageDays` (o aviso dos 90 dias é calculado no maintenance).
- Mutation `buildFieldMap(citySlug: String!)` por `BaseMutation#audited(event: "maintenance.field_map.build_requested", module_name: "field_maps")` (plataforma; cidade por `citySlug`, Q19; inexistente ou não ativa → `Rejected` com `path: "citySlug"`), chama `FieldMaps::Request.build!` (sem base, o job grava `failed base_missing`, Q24). Mutation `refreshFieldMapBase`, evento `maintenance.field_map.base_refresh_requested`. Sem TOTP nem step-up (Q22; como `importAnvisaLists`).
- `MaintenanceAudit::NAMES` e `R18_PLATFORM_EVENT_NAMES` += os dois nomes (Q19).

- [ ] **Step 1: Write the failing tests** (`type: :request`, padrão de `spec/requests/maintenance/medication_catalog_spec.rb`): "fieldMaps lista só as cidades com linha, com citySlug, estado e o publicado; sizeBytes acima de 2^31 sai inteiro (BigInt)"; "o SDL dos tipos e campos é exatamente o do bloco acima (`Maintenance::Schema.to_definition`)"; "fieldMapBase nulo sem base; com base, version e ageDays"; "buildFieldMap enfileira um FieldMaps::BuildJob na fila de plataforma e grava attempted/ok"; "segunda chamada com building não enfileira de novo"; "slug desconhecido → ok false com errors e auditoria rejected"; "refreshFieldMapBase enfileira o RefreshBaseJob"; "sem sessão de mantenedor → erro de autenticação"; "payload da auditoria passa no R18".
- [ ] **Step 2:** `bin/test api -w mod24c spec/requests/maintenance/field_maps_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture/maintenance_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Se `spec/architecture/maintenance_schema_spec.rb` (ou um dump do schema GraphQL usado pelo maintenance) fixar o SDL, regerar o dump e avisar a sessão do maintenance com o arquivo.
- [ ] **Step 4:** os mesmos specs + `spec/requests/maintenance/graphql_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: expose field map status and rebuild mutations to the maintenance API" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 15 · fieldMaps (só linhas), fieldMapBase, buildFieldMap(citySlug), refreshFieldMapBase; nomes maintenance.field_map.* (Q19); BigInt (Q21); sem TOTP (Q22); SDL igual ao do plano do maintenance; avisar a sessão do maintenance."

---

### Task 16: Semente de dev realista

**Files:**
- Modify: `lib/home_visit_crew.rb` (ou a semente do 24a que cria os domicílios de Curitiba), `spec/lib/home_visit_crew_spec.rb`

**Interfaces:**
- Domicílios de Curitiba semeados ganham `location` plausível (dentro da caixa do município, nas ruas do endereço semeado, `source: "gps"`, `accuracy_m` 5–15), pelo `SaveHousehold` (versão nova e ficha), idempotente; uma visita do app semeada com GPS; nenhum `FieldMap` semeado (o mapa nasce pelo maintenance na prova).

- [ ] **Step 1: Write the failing test** (`type: :model`): "semear duas vezes não cria versão nova de ponto"; "todo ponto semeado cai na caixa IBGE de Curitiba (`FieldMaps::Boundaries`)".
- [ ] **Step 2:** `bin/test api -w mod24c spec/lib/home_visit_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: seed dev household and visit points inside Curitiba" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 16 · semente com pontos em Curitiba e uma visita do app com GPS."

---

### Task 17: Sigilo da coordenada, eventos e R18

**Files:**
- Create: `spec/security/field_location_secrecy_spec.rb`

- [ ] **Step 1: Write the test** (`type: :request`), cenário completo (domicílio com ponto pelo app e pelo `PATCH`, família nova, visita com GPS, download, referência, mapa, pontos do dashboard, build do mapa):
  - `capture_log` de todas as requisições e jobs não contém nenhuma latitude/longitude usada (como texto com 4+ casas) nem o token;
  - nenhum `DomainEvent` nem `PlatformEvent` publicado tem `location`, `latitude`, `longitude` ou `accuracy_m` em qualquer profundidade; os nomes de `PlatformEvent` novos ⊂ Valor fixado 3;
  - nenhuma recusa (`invalid_location`, `family_not_found`, 416, 404) traz o valor enviado;
  - nenhuma rota nova aceita coordenada na query; o cursor do download decodificado não tem coordenada;
  - `field_maps`, `field_map_bases` e o objeto do mapa não têm nenhum id de domicílio ou pessoa; nenhuma chave de cache contém coordenada ou nome;
  - grep multilinha dos `Platform.audit(`/`MaintenanceAudit.record(`/`audited(` dos arquivos do módulo = exatamente os quatro nomes.
- [ ] **Step 2:** `bin/test api -w mod24c spec/security/field_location_secrecy_spec.rb spec/security/home_visit_secrecy_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture` → PASS (falha → correção na task dona, volta aqui).
- [ ] **Step 3: Commit** `git commit -m "test: pin field location secrecy across logs, events and errors" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 24c · Task 17 · coordenada fora de log, evento, erro e cache; quatro nomes de plataforma novos, todos no R18."

---

### Task 18: Fechamento — SHA da tag, suíte completa e passagem aos fronts

**Files:** `app/services/field_sync/schemas.rb`, `spec/services/field_sync/schemas_spec.rb` (só se a tag mudou o SHA).

- [ ] **Step 1:** conferir `config/field_sync/` contra a tag `field-sync-v1.2.0` (publicada pelo coordenador depois das decisões); SHA diferente → atualizar `SHA256`, o `pinned` e os exemplos, `bin/test api -w mod24c spec/services/field_sync spec/requests/field_sync` → PASS e `git commit -m "chore: pin field sync schemas to field-sync-v1.2.0" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.
- [ ] **Step 2: Suíte completa** (transversal: migração de cidade e de plataforma, `CityEncryption`, `Ledi`, 24a, `FieldAuthentication`, `spec/spec_helper.rb`, `Dockerfile`, initializers): **pedir ao usuário** a rodada pela sessão "Testes", por `SendMessage`: `api · mod24c/feat/mod-24c-field-app · --full · módulo 24c (migração de cidade 20261113900001 e de plataforma 20261113900001, CityEncryption, Ledi fichas 3 e 8, SaveHousehold do 24a, FieldAuthentication, spec_helper, Dockerfile, initializers)`. Antes, `docker compose build api` (CLI novo na imagem). Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador com o resumo e o hash, e as sessões do campo, do dashboard e do maintenance de que o api está pronto (porta 3058).
- [ ] **Step 3:** merge, push e cards só com autorização do usuário; F-24.19/20/21/24 seguem In Progress até o último app de cada card (superfícies da Q10). No fechamento, abrir no board #2 os gates do api: prova no PEC real das fichas 3 e 8 com coordenada; tamanho do mapa de Curitiba e Maringá medido em staging; disco do worker para a cópia local da base (Q32, também no runbook).

**Relato ao coordenador:** "api 24c · Task 18 · schemas presos na tag v1.2.0; suíte completa <verde N/0 | falhas>; hash <…>; pronto para campo, dashboard e maintenance."

---

## Self-review

1. **Cobertura do contrato:** §1 nomes (2, 3, 11, 12, 15); §2 schemas presos (2, 18); §3.1 revogação (7); §3.2 sessão `map` (13) e `reference_version` (10); §3.3 item `family`, `family_client_uuid`, `version`/`result_version` (3, 9), `location` no domicílio (4) e na visita (5); §3.4 `<household>` com `location` (4); §3.5 referência (10); §3.6 substituir (8); §4 cifra (3), fichas 3 e 8 com prova contra a IDL (1, 6); §5.1 `field_maps`, base, recorte, CLI, fixture e spec de integração marcado (11, 12); §5.2 entrega (13); §5.3 pontos e trilha (14); §5.4 GraphQL e auditoria (15); §10 códigos (1, 7, 9, 13); eventos e R18 (12, 15, 17); ADR até 53 (2); suíte completa (18).
2. **Spec §6 "api":** item `family` (idempotência, dependentes, `cbo_not_allowed`) (9); `location` (par, faixa, casas, cifra, sem log) (1, 3, 4, 5, 17); fichas 3 e 8 com e sem coordenada, campo a campo (6); `field_maps` (recorte pequeno, versão, SHA-256, `Range`, sem token → 401) (11–13); `map` na sessão (13); respostas contra a v1.2.0 (2, 9, 10, 13).
3. **Nomes:** `FieldLocation.{parse,decimals,ledi}`, `FieldSync::Reference.{build,version}`, `FieldMap.published_for`/`#manifest`, `FieldMapBase.current`, `FieldMaps::{Storage,Cli,Boundaries,Request,Delivery}`, `FieldMaps::{BuildJob,RefreshBaseJob}`, `FieldDevices::Register.call(user:, name:, replaces_device_id:)`, `TerritoryRegistry::Access.reads_households?(user)`, `TerritoryRegistry::Trail.map_households_listed!`, `FieldSync::Upload#cite_patient` — mesma assinatura em todas as tasks. Entre planos: os códigos de `HTTP_ERRORS` e da `CityResolution` = `errors.http` do contracts = frases do campo; `replaced` = rótulo do dashboard; `map` de `/territory/map/households` = `MapHouseholdsResponse.map` do dashboard; o SDL da Task 15 = o do maintenance.
4. **Ordem:** 2 antes de tudo que valida contra schema; 3 antes de 4–6 e 9; 11 antes de 12–15; 13 consome o `published_for` da 11.

## Decisões aplicadas (§13 do contrato)

| Código | Q | Onde o plano aplica |
|---|---|---|
| G-1 | Q25 | Task 9: sem regra nova; o cadastro ativo do responsável, se houver, entra na família (L28) |
| G-2 | Q1 | Task 9: `responsible: { patient_id \| patient_client_uuid }` aninhado |
| G-3 | Q26 | Tasks 3 e 9: `field_sync_receipts.result_patient_id`; `cite_patient` resolve `patient` e `individual_registration` com `new_patient` |
| G-4 | Q5 | Task 9: `version` obrigatório no `accepted`, opcional no `duplicate` (recibo legado sem `result_version`) |
| G-5 | Q27 | Task 10: `registration_dictionaries` de `Dictionary::DATA`, listas `[{ code, label }]` na ordem do YAML |
| G-6 | Q6 | Task 10: `professional_memberships` filtrado a `in_team`, com `has_cns`; `cnes` anulável |
| G-7 | Q7 | Task 10: `collective_activity_domains` num bloco só |
| G-8 | Q29 | Task 10: cache só do hex |
| G-9 | Q30 | Tasks 11–13: publicado separado do estado da última geração |
| G-10 | Q31 | Valor fixado 3, Task 12: `<version>-<sha12>.pmtiles`; SHA igual não sobe |
| G-11 | Q23 | Task 12: sem job recorrente; build mais recente pelo índice, caixa do Brasil (Paraná em dev) |
| G-12 | Q32 | Task 12: cache local da base no worker, conferido por SHA-256; disco no runbook (Task 18) |
| G-13 | Q33 | Valor fixado 6, Task 12: CSV vendorizado da API de malhas do IBGE |
| G-14 | Q19 | Valor fixado 3, Task 15: `citySlug`; `maintenance.field_map.*`; jobs gravam `field_map.built`/`failed` |
| G-15 | Q8 | Valor fixado 2, Tasks 2 e 13: `map_not_found`, `range_not_satisfiable` em `HTTP_ERRORS`; borda da cidade conferida contra `errors.http` |
| G-16 | Q34 | Task 13: um intervalo por requisição, 416 com `Content-Range`, `Cache-Control: private, no-cache`, `If-Range` aceito |
| G-17 | Q16 | Task 8: só aparelho ativo se substitui |
| — (D-7 do dashboard) | Q17 | Valor fixado 2, Tasks 3, 7 e 8: motivo `replaced` (CHECKs de `field_devices`), `device_suspended` para `replaced`/`membership_ended`/`user_disabled` |
| G-18 | Q35 | Task 9: a versão do cadastro do responsável não volta; README do `field-sync` documenta |
| G-19 | Q3 | Valor fixado 1, Tasks 1, 4, 5: `accuracy_m` obrigatório só com `gps`; visita `null` → `invalid_location` |
| — (C-1 do contracts) | Q2 | Task 9: erro de esquema sob `/body/location` → `rejected invalid_location` com o mesmo `path` |
| G-20 | Q36 | Task 6: ponto na ficha 3 também com recusa |
| G-21 | Q37 | Tasks 4 e 5: `location` em todo `<household>`; nunca nas rotas da equipe para a visita |
| G-22 | Q38 | Task 14: só domicílios ativos |
| G-23 | Q39 | Task 13: `TerritoryRegistry::Access.reads_households?(user)` |
| G-24 | Q40 | Task 1: o contrato cita os números da IDL |
| — (D-4 do dashboard) | Q14 | Task 14: `map` na resposta de `/territory/map/households` |
| — (D-3 do dashboard) | Q13 | Task 4: correção do dashboard = corpo inteiro do 24a + `location` |
| — (D-2 do dashboard) | Q12 | Valor fixado 5, Task 11: versão do `go-pmtiles` da prova do campo |
| — | Q10 | Tasks 0 e 18: api em F-24.19, F-24.20, F-24.21, F-24.24 |
| — (M-*) | Q21, Q22, Q24 | Task 15: tipos do SDL, sem TOTP, `base_missing` sem base |
