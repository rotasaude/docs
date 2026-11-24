# Módulo 26c — Envio da regulação à RNDS (MIRA/RIRA) (api) Implementation Plan

> **Nota do coordenador (Q13, alinhamento com o plano do dashboard):** os formatos e os códigos de recusa das configurações e da credencial `rnds_regulation` são os da tabela do plano do dashboard: CBO de 6 dígitos, SIGTAP de 10, identificador do solicitante, CNS de 15, reenvio. Onde este plano diverge (identificador alfanumérico até 30, recusa genérica `invalid_request`), vale a tabela do dashboard, com o código específico e o `path`. O `fix` da lista também traz `offer_month`, para o atalho da oferta abrir no mês certo.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: o 27b na `origin/main` do api.** Ao escrever este plano (2026-11-24) a `origin/main` era `ca2a92f0`, **sem** o 27b: última migração de cidade `20261123900003_add_zoonoses_care.rb` (`db/city_schema.rb:13`, `define(version: 2026_11_23_900003)`), `IntegrationCredential::KINDS = %w[ledi cadsus rnds]` (`app/models/integration_credential.rb:8`) e o `CATALOG` terminando em `zoonoses` (`app/services/platform/features.rb:230`). O plano do 27b (`2026-11-24-module-27b-bnafar-api.md`) acrescenta as migrações `20261125900001..900004`, a `20261125600001` de plataforma, `bnafar` em `KINDS`, a coluna `integration_credentials.environment` (CHECK `kind = 'bnafar' OR environment = 'production'`, único `(kind, environment)`) e cinco interruptores depois de `zoonoses`. Este plano **parte desse estado**; a Task 0 confere e para se ele não estiver na main. Nomes conferidos no código (ca2a92f0):
> - **26a:** `CareRegulation::Catalog::{STATUSES, TERMINAL_STATUSES, COLORS, MAX_MISSES, CNES, SPECIALTY_CODE, PROCEDURE_CODE}` (`catalog.rb:5-33`); `CareRegulation::Transition::{EVENT_FOR, EVENTS}` — os 11 eventos `regulation_request.{created complemented external authorized returned denied cancelled scheduled completed missed closed}` (`transition.rb:13-18`); `CareRegulation::Misses.apply!` (só ele leva a `closed`, sempre com `misses = 2`: `misses.rb:42`; `appointment_sync.rb:177`); `CareRegulation::AppointmentSync` e `AppointmentSyncJob::RESOLVERS` (10 eventos do 17, `appointment_sync_job.rb:14-26`); `CareRegulation::{SaveOffer (IMMUTABLE, só `quantity` muda no PATCH: `save_offer.rb:10, 53-56`), SaveProvider, SaveDeadlines (publica `care_regulation.settings_changed` com `user_id`: `save_deadlines.rb:37`), ScheduleExternal.external_network?, Targets.resolve(type:, code:, on:), Json.{offer, provider}, Cursor.{encode_page, decode_page}}`; `CareRegulation::BaseController` (`render_failure`, `body`, `query_only!`, `page_limit`, `admin?`, `regulator?`, `uuid?`, `require_step_up!`); rotas em `scope "/regulation", module: :care_regulation` (`config/routes.rb:484-516`). Tabelas: `regulation_requests` (`state_regulated`, `care_provider_id`, `requester_unit_id`, `target_type/target_code`, `cid10/ciap2` cifrados, `color/suggested_color`, `requested_at`, `scheduled_at`, `misses`, `appointment_request_id`; **sem número de pedido**), `regulation_request_events`, `regulation_authorizations` (`care_offer_id`, `created_at`, `released_at`), `care_offers` (mensal), `care_providers` (`cnes` nulo no `own`: lido da unidade).
> - **Agenda (17):** `appointments.shift_id` (nulo no legado) → `professional_shifts.professional_link_id` → `professional_links.cbo_code`; `appointments.health_unit_id`.
> - **RNDS (29):** `Rnds::{Client (for(city, env:, settings:), CACHE por `[city.id, identity.version, simulated]`, `token!(fresh:)`, `post_bundle(bundle, practitioner_cns:)` → `Response(status:, rnds_id:, error:)`), Config.{for_city, real, simulated}, Mock::KEY, Gate, Backoff.{wait, CAP, BLOCKED_RECHECK} (o `GIVE_UP_AFTER` vem de `Vaccination::Catalog`), Errors.{sanitize, NotConfigured, Unauthorized, Unavailable}, Alert.counts}`; `Rnds::DeliverJob` (`prepend EachCityJob`, `limits_concurrency to: 1, key: "rnds_deliver"`); `RndsOutboxEntry.{claim!, release!, release_stale!}` (`rnds_outbox_entry.rb:63-86`, token = `last_attempted_at`); `lib/fake_rnds/app.rb` (marcadores no `lotNumber`, sem `GET /identifier`, `DELETE` nem EDS-MSG009); `spec/support/vaccination_helpers.rb#rnds_credential!`; fake nas specs em `http://fake-rnds.test:8094` com `FAKE_RNDS_URL`.
> - **Integração:** `Integrations::{SetCredential (audita `rnds.credential_set` só para `rnds`), CheckConnection.check_rnds}`; `IntegrationsController::ERROR_STATUS`; `Platform::Features.{missing_for, city_state, CITIZEN_HIDDEN, SIMULATION_ENVS}`; `rnds_mock` com `requires: %w[feature:rnds_immunization]` (`features.rb:167-170`); `Professionals::Cns.valid?`.
> - **Guardas:** `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb:15`); `CityEncryption::CITY_KEYED_TARGETS` (`city_encryption.rb:36`, `RegulationRequest` em `:149`); `DomainEvents.bind` dos eventos do 26a (`config/initializers/domain_events.rb:396-400`); **não existe** `health_unit.changed` (o CNES da unidade muda em `HealthUnitsController#assign_cnes`, `:170`, e em `Cnes::Apply#apply!`, `cnes/apply.rb:51`); `Citizens::RequestErasure.regulated?` (`request_erasure.rb:166`); `config/recurring.yml` (`rnds_deliver`, `:97`); `.github/workflows/ci.yml` (jobs `rspec`, `production-boot`, `staging-boot`). Não existe módulo `Regulation` no app (namespace livre).
> - **IG RIRA (lido em 2026-11-24, `https://fhir.saude.gov.br/rira/package.tgz`):** `package.json` = `br.gov.saude.rira.fhir` `1.0.0-release`, `notForPublication: true`, `date 20261123112114`, dependências `hl7.fhir.r4.core 4.0.1`, `hl7.terminology.r4 7.5.0`, `hl7.fhir.uv.extensions.r4 5.4.0`; SHA-256 do arquivo baixado `a0a1722bf8f310dfb7ee074caf44c2839ae570011a1fd92d380f5ac9164f920f`. O pacote **não** está no registro `packages.fhir.org` (404). Perfis `http://www.saude.gov.br/fhir/r4/StructureDefinition/{BRRegulacaoAssistencial, BRAgendamentoRegulacaoAssistencial, BRRequisicaoRegulacaoAssistencial, BRProblemaDiagnostico}`; `Composition.custodian` 0..0, `section.title` 0..0, `relatesTo` 0..1 com `targetReference.reference`; `Appointment.participant` 0..1 só `PCT`; `ServiceRequest.performer` 0..1 por identifier; `GET [ehr]/api/identifier?system=&value=&docType=` devolve um **array** `[{ id, cns, date, relatesTo, status }]`; `DELETE [ehr]/api/fhir/r4/Bundle/{id}` → 204 ou 422 EHR-ERR825. Relação de Status (figura `relacaoStatus.png` do `mc.html`, recomendação, não regra de entrada): `pending → proposed/active`, `booked → booked/active`, `attended → fulfilled/completed`, `cancelled → cancelled/revoked`, `absence → noshow/completed`, `excluded → cancelled/revoked`, `returned-to-requester → waitlist/on-hold`; `ServiceRequest.intent` sempre `proposal`.

**Goal:** O lado api do módulo 26c: interruptor e credencial `rnds_regulation`, estado de sincronização por pedido de regulação, documento RIRA conferido e enviado à RNDS em cadeia de substituição, com carga inicial, alarme D-1, telas de situação/reenvio/configurações, simulado ampliado e validação oficial na CI.

**Architecture:** Uma migração de cidade irreversível (`regulation_rnds_settings`, `regulation_rnds_syncs`, `regulation_rnds_deliveries`, CBO do executante na oferta e no prestador, `rnds_regulation` no CHECK de credencial). O domínio mora em `Regulation::*`: regras puras (`RndsStatus`, `RiraBundle`, `RiraSkeleton`, `RndsValidation`, `RndsAlert`) sobre fatos já lidos (`RiraFacts`), o recálculo por pedido (`RndsSync`) e o envio (`RndsDelivery`, `RndsReconcile`). Três jobs: `Regulation::RndsSyncJob` (consumidor dos eventos, recalcula o desejado), `Regulation::RndsInitialLoadJob` e `Regulation::RndsDeliverJob` (por minuto, claim `SKIP LOCKED`, um documento por pedido). O transporte é o `Rnds::Client` do 29, agora pelo `kind` da credencial, com `GET /identifier` e `DELETE` novos.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), Solid Queue, RSpec via `bin/test`, WebMock `to_rack`, `Gem::Package::TarReader`/`Zlib` (stdlib) para ler o pacote do IG; na CI, Java 21 (Temurin) e `validator_cli.jar` 7.0.6 do HL7 **só no job `rira-validate`**. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-25-module-26c-mira-design.md` (decisões 1–18). Contrato entre apps — **fonte única**: `docs/superpowers/plans/2026-11-25-module-26c-mira-contracts.md` (§1–§10, Q1–Q14; docs `5d49ef7`). Pesquisa: `docs/pesquisa/2026-11-23-mira-regulacao-rnds-26c.md` (achados 1–15).

## Reconciliação (§10, Q6–Q14)

As divergências D1–D14 foram decididas no §10 do contrato (docs `5d49ef7`); as tasks já seguem as decisões. Onde a decisão difere da proposta ou acrescenta algo:
- **Q10:** as leituras `/status` e `/syncs` exigem `Regulation::RndsGate.enabled?`; o reenvio e os jobs, `usable?` (Tasks 12, 14). O `GET` e o `PUT` das configurações e a credencial valem antes de ligar, sem o gate (§5; ajuste do coordenador depois do `5d49ef7`) (Task 6).
- **Q11:** a lista traz `request_ref` (8 primeiros hex do id do pedido, em maiúsculas, calculado pelo api); as contagens trazem `accepted` (Tasks 13, 14).
- **Q12:** o CBO do alvo `specialty` é a **primeira** fonte do CBO do executante, antes da agenda, da oferta e do prestador; a oferta nova herda o `performer_cbo` do mês anterior (mesmo prestador e alvo); o POST de oferta aceita `performer_cbo` e o PATCH aceita mudar só ele, sem `quantity` (Valor 8; Tasks 6, 8).
- **Q13:** o `PUT` das configurações pede step-up (Task 6).
- **Q14:** o requisito alternativo do `rnds_mock` é `rnds_sending`, com o motivo `rnds_sending_disabled` (Valor 1; Task 2).
- **Q7–Q9** como propostos: externo = `state_regulated` em qualquer situação; `erased` só com documento aceito; `specialty_without_sigtap` quando o SIGTAP não é regulável ou não está vigente (Tasks 7, 8, 10).

## Valores fixados por este plano

1. **Interruptor** (fim do `CATALOG`, depois do último do 27b): `Entry.new(key: "rnds_regulation", description: "Envio da regulação à RNDS", requires: %w[feature:care_regulation credential:rnds_regulation ibge_code uf])`; `CITIZEN_HIDDEN` + `rnds_regulation`. Requisito alternativo novo `rnds_sending` no `missing_for` → `"rnds_sending_disabled"` salvo `usable?(city, "rnds_immunization") || usable?(city, "rnds_regulation")` (Q14); `rnds_mock` passa a `requires: %w[rnds_sending]`. `Regulation::RndsGate::KEY = "rnds_regulation"`, `usable?(city)`, `enabled?(city)` (molde `Rnds::Gate`).
2. **Credencial** `rnds_regulation`: `SECRET_KEYS["rnds_regulation"] = %w[pfx_base64 pfx_password requester_cnes requester_id responsible_cns]`; `REQUESTER_ID_FORMAT = /\A[0-9A-Za-z]{1,30}\z/`; `responsible_cns` por `Professionals::Cns.valid?`; erros novos `invalid_requester_id`, `invalid_responsible_cns` (422); `Platform.audit("rnds_regulation.credential_set", city_id:)`; ambiente sempre `production` (CHECK do 27b). Leitores `IntegrationCredential#requester_id`, `#responsible_cns`.
3. **`Regulation::RndsLimits`:** `BATCH_SIZE = 20`; `INITIAL_RATE_PER_MINUTE = 60`; `STALE_SENDING = 10.minutes`; `BUDGET = 4.minutes`; `GIVE_UP_AFTER = 15.days` (contado de `enqueued_at`); `BLOCKED_RECHECK = 1.hour`; `INITIAL_LOAD_DAYS = 30`; `INITIAL_GRACE = 72.hours`; `PAYLOAD_DAYS = 30`; `LATE_HOUR = 23`; `DEFAULT_SIGTAP = "0301010072"`; `MODALITY = "09"`; `CBO_GROUPS = %w[03 04]`; `SPECIALTY_SIGTAP_MAX = 500`; `ERROR_CODE = /\A(EHR-ERR|EDS-MSG)\d{3}\z/`; `REPLACED_CODE = "EDS-MSG009"`; `NOT_FINAL_CODE = "EHR-ERR825"`.
4. **`Regulation::RndsCatalog`:** `MIRA_STATUSES = %w[pending booked attended cancelled absence excluded returned-to-requester]`; `SYNC_STATUSES = %w[idle queued sending blocked rejected failed excluded erased]`; `BLOCK_REASONS = %w[missing_diagnosis requester_unit_without_cnes performer_without_cnes patient_without_cpf_or_cns performer_cbo_missing specialty_without_sigtap]` (+ `skeleton:<caminho>`); `DELIVERY_KINDS = %w[create replace delete]`; `PRIORITIES = %w[routine urgent]`; `DEFAULT_PRIORITY_BY_COLOR = { "red" => "urgent", "yellow" => "routine", "green" => "routine", "blue" => "routine" }`; `DOMAIN_EVENTS = %w[regulation_rnds.queued regulation_rnds.accepted regulation_rnds.blocked regulation_rnds.rejected regulation_rnds.excluded regulation_rnds.settings_changed]`; `STATUS_RELATION` (a figura do IG): `{ "pending" => %w[proposed active], "booked" => %w[booked active], "attended" => %w[fulfilled completed], "cancelled" => %w[cancelled revoked], "absence" => %w[noshow completed], "excluded" => %w[cancelled revoked], "returned-to-requester" => %w[waitlist on-hold] }` (Appointment, ServiceRequest).
5. **Correspondência** (`Regulation::RndsStatus.for(request, settings:) -> Target(status:, at:) | :none | :excluded`), na ordem: `draft` → `:none`; `state_regulated` com `send_external: false`, em qualquer situação → `:excluded` (Q7); `pending`, `authorized`, `external` → `pending`; `returned` → `returned-to-requester`; `scheduled` → `booked`; `completed` → `attended`; `denied`, `cancelled` → `cancelled`; `closed` com `misses >= MAX_MISSES` → `absence`; `closed` sem isso → `cancelled`. `at` = `created_at` da última linha de `regulation_request_events` cujo status leva à situação (o `requested_at` no `pending` sem linha).
6. **Passos** (`Regulation::RndsStatus.steps(request, since:) -> [{ "status" => "absence", "at" => iso8601 }]`): uma entrada por linha `missed` do andamento com `created_at > since` (o `sent_at` da última remessa aceita; nil = todas), **menos** a última quando o pedido está `closed`; passos iguais ao desejado no fim são descartados; consecutivos iguais viram um.
7. **Documento** (`Regulation::RiraBundle.build(facts:, sync:, status:, now:)`, Q6): Bundle `document`, `identifier { system: "http://www.saude.gov.br/fhir/r4/NamingSystem/BRRNDS-#{facts.requester_id}", value: sync.bundle_identifier }`, quatro entradas na ordem Composition, Appointment, ServiceRequest, Condition (`fullUrl` `urn:uuid:` novo a cada montagem). Composition: `status final`, `type RA` (`BRTipoDocumento`), `category 09` (`BRModalidadeAssistencial`), `subject.identifier` do paciente, `date now`, `author.identifier` = CNES `requester_cnes` da credencial (`BREstabelecimentoSaude-1.0`), `title "Regulação Assistencial"`, `event[0] { code: <situação> (BRStatusRegulacaoAssistencial), period.end: at }`, `section[0].entry[0]` → Appointment; `relatesTo [{ code: "replaces", targetReference: { reference: "Composition/#{sync.last_rnds_id}" } }]` quando há `last_rnds_id`. Appointment: `status` do `STATUS_RELATION`, `serviceCategory 09`, `serviceType` = SIGTAP, `specialty` = CBO do executante (quando há), `appointmentType` = caráter, `basedOn` → ServiceRequest, `participant [{ type PCT, actor.identifier paciente, status accepted }]`, `created` = `regulation_authorizations.created_at` vigente (quando há), `start` = `scheduled_at` (decisão 18, pelo `date_fields` do perfil). ServiceRequest: `status` do `STATUS_RELATION`, `intent proposal`, `category 09`, `priority` = caráter, `code` = SIGTAP, `subject`, `authoredOn` = `requested_at`, `requester.identifier` = CNES da unidade solicitante, `performer[0].identifier` = CNES do executante (só `booked`/`attended`), `performerType` = CBO (executante; no `pending` de pedido por especialidade, o CBO do alvo), `reasonReference` → Condition. Condition: `clinicalStatus active`, `code` CID-10 (`BRCID10`) se houver, senão CIAP-2 (`BRCIAP2`), `subject`. Paciente: `BRIndividuo-1.0` com o CPF; sem CPF, o CNS. Sistemas e canônicos só de `config/rnds/rira_1_0.yml`.
8. **SIGTAP e CBO:** alvo `procedure` → o próprio `target_code`; alvo `specialty` → `settings.specialty_sigtap[target_code] || DEFAULT_SIGTAP`. CBO do executante (decisão 14, Q2, Q12), nesta ordem: (1) o `target_code` do pedido por `specialty`; (2) `professional_links.cbo_code` do `appointments.shift_id` do horário vivo/último da cadeia (rede própria); (3) `care_offers.performer_cbo` da autorização vigente; (4) `care_providers.default_performer_cbo` do prestador (o da oferta, ou `regulation_requests.care_provider_id` no externo). Exigido só em `booked`/`attended` com SIGTAP de grupo em `CBO_GROUPS`. Executante: `own` → CNES da unidade do horário (ou da unidade do prestador); externo → `care_providers.cnes` (da oferta vigente ou de `regulation_requests.care_provider_id`).
9. **Impressão digital:** `Regulation::RiraBundle.digest(bundle) -> String` = SHA-256 hex do JSON com chaves ordenadas do Bundle **sem** `timestamp`, `identifier`, `entry[].fullUrl`, as referências `urn:uuid` (trocadas pela posição), `Composition.date`, `Composition.author` e `Composition.relatesTo`.
10. **Erros novos:** `invalid_requester_id`, `invalid_responsible_cns` (422); `expected_version_required` (422); `rnds_settings_changed` (409); `not_retryable` (409); `invalid_request` com `path` nos campos das configurações e de `performer_cbo`/`default_performer_cbo`.
11. **Dev:** worktree `apps/api/.claude/mod26c` (com `config/master.key`), branch `feat/mod-26c-mira`, api do worktree na porta **3063**; simulado `fake-rnds` (8094) já no compose da raiz, sem mudança de serviço.

## Global Constraints

- git `/opt/homebrew/bin/git`; nunca `git add -A` nem `git stash` sem nome. Commits em inglês, Conventional Commits com o tipo por extenso: `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge, push e cards só com autorização do usuário.
- **Testes só por `bin/test api -w mod26c <specs>`**, a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`; nunca `rspec` direto nem o Postgres do host; `--keep-db` só em rodada de arquivo isolado. A branch é transversal (migração de cidade, `CityEncryption`, `Platform::Features`, `IntegrationCredential`, 26a, `Rnds::Client`, `fake_rnds`, `rnds_mock`, CI, `spec/support`, initializers) → **suíte completa obrigatória, pedida à sessão "Testes" na Task 17**.
- Migração: dump à mão em `db/city_schema.rb`; triggers num bloco "Módulo 26c" ao fim de `db/city_triggers.sql`, dentro de `IF to_regclass('public.<tabela>') IS NOT NULL`; **irreversível** (`down` → `ActiveRecord::IrreversibleMigration`; o `up` termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`); CHECK trocado por `replace_check`. Banco de dev só migrado e semeado na prova, com autorização do usuário.
- Specs com tipo explícito (`type: :request`, `:model`, `:job`). **Nenhuma spec bate em URL real**: tudo por `stub_request(:any, /fake-rnds\.test/).to_rack(FakeRnds::App.new)` com `FAKE_RNDS_URL=http://fake-rnds.test:8094`.
- **Nunca `Current.city =` solto**; interruptores lidos fora da transação da cidade nos jobs de envio; nos consumidores, só `enabled?` (lê `city_features`). `spec/architecture/current_city_assignment_spec.rb` roda nas Tasks 10, 12 e 17.
- **Sigilo (§9):** CPF, CNS, CID, CIAP, justificativa, PFX, senha, token, `requester_id`, `responsible_cns` e payload nunca em log, evento, URL, `Result`, `inspect`, resposta de lista ou mensagem de erro. Modelo/`Data` com payload ou credencial sobrescreve `inspect`/`to_s`/`pretty_print`/`as_json`. Texto da RNDS nunca é gravado: só os códigos (`ERROR_CODE`). `filter_parameter_logging` ganha `responsible_cns` e `requester_id`.
- **R18:** `rnds_regulation.credential_set` entra em `R18_PLATFORM_EVENT_NAMES`.
- Eventos em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, só ids; `DomainEvents.publish(` é multilinha.
- Cifra por cidade: `encrypts :payload` em `CityEncryption::CITY_KEYED_TARGETS` e em `spec/jobs/reencryption_job_spec.rb`.
- Concorrência otimista nas configurações: `expected_version` (ausente → `422 expected_version_required`; diferente → `409 rnds_settings_changed`).
- Todo "hoje" é `Time.zone.today` dentro de `CityConnection.with`. O envio da vacina (29) não muda de comportamento: `spec/services/rnds spec/jobs/rnds spec/lib/fake_rnds_spec.rb spec/requests/integrations_spec.rb` ficam verdes em toda task que toca `Rnds::*` ou o fake.
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`.

## Review Focus

1. **Timeout depois de a RNDS aceitar** (o POST chegou, a resposta não): a próxima rodada consulta `GET /identifier` antes de reenviar, adota o documento achado como `last_rnds_id` e segue a cadeia; nunca nasce um segundo documento "novo" nem uma substituição do já substituído. Teste na Task 12 (marcador `TIMEOUT` do fake, que grava e não responde).
2. **Duas mudanças antes do envio, com falta no meio** (autorizado → agendado → 1ª falta → pendente → reautorizado, tudo antes do job): sai `absence` e depois só o estado final; a `pending` intermediária se junta; nunca dois envios do mesmo pedido em voo. Teste nas Tasks 7 e 12.
3. **EDS-MSG009 numa substituição** (o `last_rnds_id` gravado já foi substituído por uma tentativa anterior sem resposta): vira reconciliação pelo `GET /identifier`, não `rejected`. Teste na Task 12.
4. **Credencial recusada no meio do lote** (401/403): a cidade pausa (`credential_unauthorized:rnds_regulation`), as linhas voltam a `queued` sem contar tentativa, e a tela de situação continua respondendo para o admin ver o motivo (Q10). Testes nas Tasks 12 e 14.
5. **Desligar "Enviar também os pedidos regulados pelo estado" depois de enviar**: o pedido aceito recebe substituição `excluded` e para; o que nunca foi aceito vira `excluded` sem chamar a RNDS; religar volta a enviar substituindo o `excluded`. Teste na Task 10.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod26c/`)
- Migração: `db/city_migrate/20261126900001_add_regulation_rnds.rb` (número a conferir na Task 0).
- Modelos: `app/models/{regulation_rnds_setting,regulation_rnds_sync,regulation_rnds_delivery}.rb`.
- `app/services/regulation/{rnds_gate,rnds_limits,rnds_catalog,rira_profile,rnds_status,rira_facts,rira_bundle,rira_skeleton,rnds_validation,rnds_sync,rnds_delivery,rnds_reconcile,rnds_alert,rnds_json,rira_examples}.rb`.
- `app/commands/regulation/{save_rnds_settings,retry_rnds_sync}.rb`.
- `app/jobs/regulation/{rnds_sync_job,rnds_initial_load_job,rnds_deliver_job}.rb`.
- `app/controllers/concerns/regulation_rnds_gate.rb`; `app/controllers/care_regulation/{rnds_status,rnds_syncs}_controller.rb`; `app/controllers/care_regulation/admin/rnds_settings_controller.rb`.
- `config/rnds/rira_1_0.yml`; `spec/fixtures/rira/br.gov.saude.rira.fhir-1.0.0-release.tgz`; `lib/tasks/regulation.rake`; `lib/regulation_rnds_crew.rb`; `spec/support/regulation_rnds_helpers.rb`; `spec/security/regulation_rnds_secrecy_spec.rb`; specs listados em cada task.

**Modificar**
- `app/services/platform/features.rb`, `app/models/{integration_credential,care_offer,care_provider,health_unit}.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `.github/workflows/ci.yml`, `db/seeds.rb`.
- `app/services/rnds/{client,errors}.rb`, `lib/fake_rnds/app.rb`, `app/commands/integrations/{set_credential,check_connection}.rb`, `app/controllers/integrations_controller.rb`.
- `app/commands/care_regulation/{save_offer,save_provider}.rb`, `app/services/care_regulation/json.rb`.
- Specs existentes: `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`, `spec/requests/integrations_spec.rb`, `spec/services/rnds/client_spec.rb`, `spec/lib/fake_rnds_spec.rb`, `spec/requests/care_regulation/admin_spec.rb`, `spec/services/city_schema_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/jobs/reencryption_job_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`.

---

### Task 0: Worktree, pré-requisito e cards

**Files:** nenhum código.

- [ ] **Step 1:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod26c -b feat/mod-26c-mira origin/main && cp config/master.key .claude/mod26c/config/master.key`.
- [ ] **Step 2:** conferir no worktree: `ls db/city_migrate | tail -3` e o `define(version:` do `db/city_schema.rb` (esperado: a última do 27b, `20261125900004` ou posterior); `KINDS` com `bnafar`; `integration_credentials.environment` no dump; a última `Entry` do `CATALOG`; `CITIZEN_HIDDEN =`; e que `rnds_regulation`, `regulation_rnds_` e `module Regulation` não existem. **27b ausente da main → parar** e avisar o coordenador. Migração de cidade mais nova que `20261126900001` → o número do plano sobe para o dia seguinte ao dela (nota no topo antes da Task 1).
- [ ] **Step 3:** `bin/test api -w mod26c spec/services/platform/features_spec.rb spec/requests/integrations_spec.rb spec/services/rnds spec/jobs/rnds spec/lib/fake_rnds_spec.rb spec/requests/care_regulation spec/services/care_regulation spec/jobs/care_regulation spec/initializers/domain_events_bindings_spec.rb spec/events/platform_event_payload_guard_spec.rb` → PASS (linha de base).
- [ ] **Step 4 (só com autorização do usuário):** criar no board #1 os cards do 26c pela receita da memória `board-91-features-roadmap` (Module `mod-26`, Work type `feature`), numerando depois do último F-26 do board, um `gh project item-create …` por chamada, cada comando começando com `gh`: "Interruptor e credencial rnds_regulation"; "Estado por pedido e documento RIRA conferido"; "Envio contínuo, cadeia de substituição e reconciliação"; "Carga inicial, alarme D-1 e telas de envio"; "Simulado e validação oficial na CI". Sem autorização: anotar no relato e seguir.

**Relato ao coordenador:** "api 26c · Task 0 · worktree mod26c em origin/main <hash> (27b presente); migração de cidade nova = <número>; cards <criados | aguardando autorização>; linha de base verde."

---

### Task 1: Perfil RIRA confirmado pelo pacote oficial

**Files:**
- Create: `spec/fixtures/rira/br.gov.saude.rira.fhir-1.0.0-release.tgz`, `config/rnds/rira_1_0.yml`, `app/services/regulation/{rira_profile,rnds_catalog,rnds_limits}.rb`, `spec/services/regulation/rira_profile_spec.rb`

**Interfaces:**
- Produces: Valores 3–4; `Regulation::RiraProfile.current -> RiraProfile` (`Data`: `package` `{ id, version, sha256 }`, `profiles` `{ composition:, appointment:, service_request:, condition: }`, `code_systems` `{ document_type:, modality:, status:, sigtap:, cbo:, cid10:, ciap2:, participant_type:, condition_clinical: }`, `identifiers` `{ individual:, establishment:, requester_prefix: }`, `priority_system`, `appointment_status_system`, `service_request_status_system`, `participation_status_system`, `date_fields` `{ scheduled_at: "Appointment.start", requested_at: "ServiceRequest.authoredOn", authorized_at: "Appointment.created" }`, `doc_type` `{ value:, hypothesis: true }`, `confirmed: true`).

- [ ] **Step 1:** baixar `https://fhir.saude.gov.br/rira/package.tgz` para o caminho do fixture e conferir `package/package.json` (`name br.gov.saude.rira.fhir`, `version 1.0.0-release`); gravar o SHA-256 em `rira_1_0.yml` (`package.sha256`). O arquivo fica versionado (Q6).
- [ ] **Step 2: Write the failing test** (`type: :model`): "o SHA-256 do fixture é o do YAML"; "cada `profiles.*` é o `url` de um `StructureDefinition-*.json` do pacote"; "cada `code_systems.*` do IG (`BRTipoDocumento`, `BRModalidadeAssistencial`, `BRStatusRegulacaoAssistencial`, `BRTipoParticipante`) é o `url` de um `CodeSystem-*.json` do pacote"; "os sete `MIRA_STATUSES` são exatamente os `concept.code` de `CodeSystem-BRStatusRegulacaoAssistencial`"; "`09` e `04` estão em `ValueSet-BRModalidadeAssistencialMIRA`"; "os códigos do `STATUS_RELATION` existem nos ValueSets do pacote (`BRStatusAgendamentoRegulacaoAssistencial`, `BRStatusRequisicaoRegulacaoAssistencial`)"; "`confirmed` é true e só `doc_type` é hipótese". Leitura do tgz por `Gem::Package::TarReader.new(Zlib::GzipReader.open(path))`.
- [ ] **Step 3:** `bin/test api -w mod26c spec/services/regulation/rira_profile_spec.rb` → FAIL.
- [ ] **Step 4: Implementar** o YAML (canônicos e sistemas lidos do pacote e do exemplo `Bundle-example-rira-posterior-2025-com-cid.json`; `identifiers.individual = …/StructureDefinition/BRIndividuo-1.0`, `identifiers.establishment = …/StructureDefinition/BREstabelecimentoSaude-1.0`, `requester_prefix = http://www.saude.gov.br/fhir/r4/NamingSystem/BRRNDS-`; `doc_type: { value: "RIRA", hypothesis: true }`, Q6) e os três módulos de constantes.
- [ ] **Step 5:** o mesmo spec → PASS.
- [ ] **Step 6: Commit** `git commit -m "feat: pin the official rira package and profile" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 1 · pacote RIRA 1.0.0-release versionado com SHA-256 e perfil confirmado por spec contra o pacote (Q6); docType ainda hipótese, gate (Q6)."

---

### Task 2: Interruptor `rnds_regulation` e `rnds_mock` para os dois envios

**Files:**
- Create: `app/services/regulation/rnds_gate.rb`, `app/controllers/concerns/regulation_rnds_gate.rb`, `spec/services/regulation/rnds_gate_spec.rb`
- Modify: `app/services/platform/features.rb`, `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`

**Interfaces:**
- Produces: Valor 1; `RegulationRndsGate#require_rnds_regulation!` (403 `{ error: "feature_disabled", feature: "rnds_regulation" }` sem `usable?`) e `#require_rnds_regulation_enabled!` (o mesmo com `enabled?`, Q10).

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: índices contados do fim sobem um; "`rnds_regulation` fecha o catálogo com os `requires` do Valor 1"; "sem credencial → `credential_missing:rnds_regulation`; credencial `unauthorized` → `credential_unauthorized:rnds_regulation`; `care_regulation` desligado → `care_regulation_disabled`"; "`rnds_mock` com só `rnds_regulation` utilizável → `[]`; com nenhum dos dois → `[\"rnds_sending_disabled\"]`; com só `rnds_immunization` → `[]`"; "`rnds_mock` segue fora de produção". Listas exatas de `city_features_spec.rb` com a chave; `session_features_spec.rb`: "GET /citizen/session nunca traz `rnds_regulation`". `rnds_gate_spec.rb`: "`usable?` falso sem credencial e verdadeiro com credencial ok, IBGE e UF"; "`enabled?` só lê o interruptor".
- [ ] **Step 2:** `bin/test api -w mod26c spec/services/platform/features_spec.rb spec/services/regulation/rnds_gate_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/citizen_api/session_features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a `Entry` (comentário: "26c: decisões 11 e 15; desligar mantém as linhas e para o envio"), o requisito alternativo `rnds_sending`, o `rnds_mock` revisto, o gate e o concern.
- [ ] **Step 4:** os mesmos specs + `spec/services/rnds spec/jobs/rnds` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add the rnds_regulation switch and share rnds_mock" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 2 · rnds_regulation no fim do CATALOG e em CITIZEN_HIDDEN; rnds_mock exige rnds_immunization OU rnds_regulation (rnds_sending_disabled, Q14)."

---

### Task 3: Credencial `rnds_regulation` e cliente por `kind`

**Files:**
- Create: `spec/models/integration_credential_rnds_regulation_spec.rb`
- Modify: `app/models/integration_credential.rb`, `app/commands/integrations/{set_credential,check_connection}.rb`, `app/controllers/integrations_controller.rb`, `app/services/rnds/client.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/requests/integrations_spec.rb`, `spec/services/rnds/client_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`

**Interfaces:**
- Produces: Valor 2; `Rnds::Client.for(city, kind: "rnds", env:, settings:)` — lê `IntegrationCredential.find_by(kind:)`; `Identity` ganha `responsible_cns` (nil no `rnds`); chave do cache `[city.id, kind, identity.version, settings.simulated]`; `Rnds::Client#responsible_cns`. `Integrations::SetCredential.validate_rnds(secret, kind:)` com as duas chaves a mais no `rnds_regulation`; `CheckConnection` trata `rnds` e `rnds_regulation` por `check_rnds(city, credential)` com `Rnds::Client.for(city, kind: credential.kind).token!(fresh: true)`.

- [ ] **Step 1: Write the failing tests.** Modelo (`type: :model`): "segredo com as cinco chaves passa; sem `responsible_cns` → inválido; CNS com dígito errado → inválido"; "`inspect` sem nenhum valor". `integrations_spec.rb` (`type: :request`): "`PUT /integrations/credentials/rnds_regulation` sem step-up → `401 mfa_required`"; "com step-up grava e responde sem segredo; `Platform.audit("rnds_regulation.credential_set")` só com `city_id`"; "`requester_id` `12 34` → 422 `invalid_requester_id`; CNS inválido → 422 `invalid_responsible_cns`"; "`POST …/rnds_regulation/check` com `rnds_mock` emite token novo no fake e grava `ok`"; "`GET /integrations` lista `rnds_regulation`"; "a `rnds` continua com três chaves". `client_spec.rb`: "`for(city, kind: "rnds_regulation")` usa o PFX dessa credencial"; "token da `rnds` não serve à `rnds_regulation` (cache separado)"; "`responsible_cns` vem da credencial". R18 + `rnds_regulation.credential_set`.
- [ ] **Step 2:** `bin/test api -w mod26c spec/models/integration_credential_rnds_regulation_spec.rb spec/requests/integrations_spec.rb spec/services/rnds/client_spec.rb spec/events/platform_event_payload_guard_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o CHECK `ck_integration_credentials_kind` com `rnds_regulation` vai na migração da Task 5; até lá o spec de modelo usa `build`/`valid?`, e os de request entram verdes depois da Task 5 — marcar `pending` só os que gravam, removendo o `pending` na Task 5).
- [ ] **Step 4:** os mesmos specs (salvo os `pending`) + `spec/services/rnds spec/jobs/rnds` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add the rnds_regulation credential and per-kind rnds client" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 3 · credencial rnds_regulation (requester_id, responsible_cns) com step-up e auditoria; Rnds::Client.for(kind:) com cache por kind (Q5)."

---

### Task 4: Simulado e transporte da regulação (`GET /identifier`, `DELETE`, EDS-MSG009)

**Files:**
- Modify: `lib/fake_rnds/app.rb`, `app/services/rnds/{client,errors}.rb`, `spec/lib/fake_rnds_spec.rb`, `spec/services/rnds/client_spec.rb`

**Interfaces:**
- Produces: `FakeRnds::App` com documentos por `Bundle.identifier.value` (cadeia: `{ id, status: "final" | "entered-in-error", relatesTo, date }`); `POST` com `relatesTo` substitui (o anterior vira `entered-in-error`); substituir o já substituído → 422 `(EDS-MSG009) O documento <id> já foi substituído.`; `DELETE /api/fhir/r4/Bundle/:id` → 204 (vira `entered-in-error`) ou 422 `EHR-ERR825`; `GET /api/identifier?system=&value=&docType=` → array do IG. Marcadores no `Bundle.identifier.value`: contém `RECUSA` → 422 `EHR-ERR999`; contém `TIMEOUT` → **grava** e levanta `Net::ReadTimeout`; contém `DUPLICADO` → o primeiro `POST` grava dois documentos `final` sem relação. O caminho da vacina (`lotNumber`) não muda.
- `Rnds::Client#identifier_chain(system:, value:, doc_type:) -> IdentifierChain` (`Data`: `status` `:ok | :unauthorized | :transient`, `documents` `[Document(id:, status:, replaces:, date:)]`; o `cns` da resposta é descartado); `#delete_bundle(rnds_id) -> Response` (204 → `:accepted`); `Rnds::Errors.codes(text) -> [String]` (os `ERROR_CODE` achados no diagnóstico já limpo; vazio → `["rejected"]`).

- [ ] **Step 1: Write the failing tests.** `fake_rnds_spec.rb`: "Bundle RIRA aceito com `location`"; "substituição aceita e a cadeia mostra o anterior `entered-in-error`"; "substituir o mesmo de novo → 422 com `EDS-MSG009`"; "`DELETE` de final → 204; de `entered-in-error` → 422 `EHR-ERR825`"; "`GET /api/identifier` devolve a cadeia"; "`TIMEOUT` grava e levanta"; "`DUPLICADO` deixa dois finais"; "o `lotNumber` `RECUSA` da vacina segue igual". `client_spec.rb`: "`identifier_chain` lista os documentos sem o `cns`"; "`delete_bundle` 204 → `:accepted`"; "`Errors.codes` acha `EDS-MSG009` e `EHR-ERR825` e nunca devolve o texto".
- [ ] **Step 2:** `bin/test api -w mod26c spec/lib/fake_rnds_spec.rb spec/services/rnds/client_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (paths: `"#{ehr_url}/api/identifier"` e `"#{ehr_url}#{BUNDLE_PATH}/#{id}"`; mesmos headers do `post_bundle`; `ID_FORMAT` confere o id antes do `DELETE`).
- [ ] **Step 4:** os mesmos specs + `spec/services/rnds spec/jobs/rnds spec/lib/vaccination_crew_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: simulate rira chains and add identifier lookup and delete to the rnds client" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 4 · fake_rnds com cadeia por identificador, EDS-MSG009, DELETE/EHR-ERR825 e GET /identifier; marcadores RECUSA/TIMEOUT/DUPLICADO no Bundle.identifier.value."

---

### Task 5: Migração de cidade — configurações, estado por pedido, remessas e CBO do executante

**Files:**
- Create: `db/city_migrate/20261126900001_add_regulation_rnds.rb`, `app/models/{regulation_rnds_setting,regulation_rnds_sync,regulation_rnds_delivery}.rb`, `spec/models/regulation_rnds_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{care_offer,care_provider}.rb`, `app/services/city_encryption.rb`, `spec/services/city_schema_spec.rb`, `spec/jobs/reencryption_job_spec.rb`

**Interfaces:**
- `regulation_rnds_settings` (linha única, molde `PecSetting.current`): `priority_by_color jsonb NOT NULL` (padrão `DEFAULT_PRIORITY_BY_COLOR`), `specialty_sigtap jsonb NOT NULL DEFAULT '{}'`, `send_external boolean NOT NULL DEFAULT false`, `version integer NOT NULL DEFAULT 1`, `updated_by_user_id` (FK users, nulo), `updated_at`. `RegulationRndsSetting.current` cria com os padrões.
- `regulation_rnds_syncs`: `regulation_request_id` (FK, **único**), `bundle_identifier varchar(64) NOT NULL` (único; CHECK `^[A-Za-z0-9_-]{1,64}$`; padrão `SecureRandom.uuid` no modelo, Q6), `last_rnds_id varchar(100)`, `last_accepted_at`, `sent_status`, `sent_digest varchar(64)`, `desired_status`, `desired_digest varchar(64)`, `pending_steps jsonb NOT NULL DEFAULT '[]'`, `status varchar NOT NULL` (CHECK `SYNC_STATUSES`), `blocked_reasons text[] NOT NULL DEFAULT '{}'`, `last_error_codes text[] NOT NULL DEFAULT '{}'`, `attempts integer NOT NULL DEFAULT 0`, `next_attempt_at`, `initial_load boolean NOT NULL DEFAULT false`, `enqueued_at`, `claimed_at` (token do claim, Q6), `created_at`, `updated_at`; CHECKs dos dois status MIRA (`MIRA_STATUSES` ou nulo); índice parcial `(status, next_attempt_at) WHERE status IN ('queued','blocked','sending')`.
- `regulation_rnds_deliveries`: `regulation_rnds_sync_id` (FK), `kind` (CHECK `DELIVERY_KINDS`), `mira_status`, `payload text` (cifrado; nulo no `delete`), `digest varchar(64)`, `rnds_id varchar(100)`, `http_status integer`, `error_codes text[] NOT NULL DEFAULT '{}'`, `sent_at NOT NULL`, `accepted_at`; índice `(regulation_rnds_sync_id, sent_at)`.
- `care_offers.performer_cbo varchar(6)` e `care_providers.default_performer_cbo varchar(6)`, CHECK `^[0-9]{6}$` ou nulo.
- `replace_check :integration_credentials, "ck_integration_credentials_kind"` com a lista do 27b + `rnds_regulation`.
- Triggers (bloco "Módulo 26c"): `rota_no_delete` em `regulation_rnds_syncs` e `regulation_rnds_deliveries`; nas remessas, `UPDATE` só anula `payload` ou grava `accepted_at`/`rnds_id`/`http_status`/`error_codes` uma vez (o resto imutável).
- Modelos: `RegulationRndsDelivery` `encrypts :payload`, `inspect` só id/kind/status HTTP; `RegulationRndsSync` `belongs_to :regulation_request`, `has_many :deliveries`, `inspect` só id/status; `CITY_KEYED_TARGETS` + `[ RegulationRndsDelivery, :payload ]`.

- [ ] **Step 1: Write the failing test** (`type: :model`): "um sync por pedido (único)"; "`bundle_identifier` `RECUSA-<uuid>` passa e `a b` cai"; "status fora do catálogo → CHECK"; "`sent_status` `booked` passa, `scheduled` cai"; "DELETE de sync e de remessa → trigger"; "remessa: anular payload passa; trocar `kind` → trigger"; "`performer_cbo` `12345` → CHECK"; "credencial `rnds_regulation` grava (production)"; "`RegulationRndsSetting.current` nasce com vermelho → urgent"; "payload cifrado com a chave da cidade". `reencryption_job_spec.rb` com o alvo novo.
- [ ] **Step 2:** `bin/test api -w mod26c spec/models/regulation_rnds_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`AddRegulationRnds`), o dump (`define(version: 2026_11_26_900001)`), os triggers e os modelos; tirar o `pending` dos specs da Task 3.
- [ ] **Step 4:** o mesmo spec + `spec/services/city_schema_spec.rb spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/requests/integrations_spec.rb spec/requests/care_regulation/admin_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add regulation rnds settings, sync state and deliveries" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 5 · 20261126900001: regulation_rnds_{settings,syncs,deliveries}, performer_cbo/default_performer_cbo, rnds_regulation no CHECK; payload cifrado; bundle_identifier texto e claimed_at (Q6)."

---

### Task 6: Configurações do envio e CBO na oferta e no prestador

**Files:**
- Create: `app/commands/regulation/save_rnds_settings.rb`, `app/controllers/care_regulation/admin/rnds_settings_controller.rb`, `spec/requests/care_regulation/rnds_settings_spec.rb`
- Modify: `app/commands/care_regulation/{save_offer,save_provider}.rb`, `app/services/care_regulation/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/requests/care_regulation/admin_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `GET /regulation/admin/rnds_settings` (`municipal_admin` ou `care_regulator`; só `care_regulation`, sem o gate do RNDS: vale antes de ligar, como o PUT e a credencial, §5) → `{ priority_by_color, specialty_sigtap, default_sigtap: "0301010072", send_external, version }`.
- `PUT /regulation/admin/rnds_settings` (`municipal_admin`, step-up, Q13; sem o gate do RNDS, como no §5: vale antes de ligar) corpo `{ expected_version, priority_by_color, specialty_sigtap, send_external }` → `Regulation::SaveRndsSettings.call(attrs:, by:) -> Result.ok(settings:) | fail(:expected_version_required | :rnds_settings_changed | :invalid_request, details: { path: })`. Regras: as quatro cores, valores em `PRIORITIES` (path `priority_by_color.<cor>`); chaves `SPECIALTY_CODE`, valores aceitos por `CareRegulation::Targets.resolve(type: "procedure", code:)` (path `specialty_sigtap.<cbo>`), até `SPECIALTY_SIGTAP_MAX`; `send_external` booleano. Mudou → `version + 1` e `DomainEvents.publish("regulation_rnds.settings_changed", user_id:)`.
- `SaveOffer` (Q12): o POST aceita `performer_cbo` (6 dígitos ou `null`, path `performer_cbo`); ausente no POST, a oferta nova **herda** o `performer_cbo` da oferta do mês anterior (`month - 1.month`) do mesmo prestador, `target_type` e `target_code`; o PATCH aceita `quantity` e/ou `performer_cbo` (pelo menos um; só `performer_cbo` não exige `quantity`; nenhum dos dois → 422 path `quantity`), e os outros campos seguem imutáveis; `SaveProvider`: `default_performer_cbo` (6 dígitos ou `null`); os dois publicam o evento de sempre quando mudam. `Json.offer` + `performer_cbo`; `Json.provider` + `default_performer_cbo`.

- [ ] **Step 1: Write the failing tests** (`type: :request`): "regulador lê as configurações padrão"; "GET com `rnds_regulation` desligado → 200 com os padrões"; "GET com a credencial recusada → 200"; "admin grava com step-up, a versão sobe e o evento leva só `user_id`"; "sem `expected_version` → 422 `expected_version_required`; versão velha → 409 `rnds_settings_changed`"; "`orange` → 422 path `priority_by_color`; `urgent` em `blue` passa"; "SIGTAP `0101010010` (grupo 01) → 422 path `specialty_sigtap.225120`"; "regulador grava → 403"; "PUT sem step-up → `401 mfa_required`" (Q13); "PUT funciona com `rnds_regulation` desligado". `admin_spec.rb`: "POST de oferta com `performer_cbo` grava"; "POST sem `performer_cbo` herda o da oferta do mês anterior do mesmo prestador e alvo; de outro prestador não herda"; "PATCH só com `performer_cbo` (sem `quantity`) grava e publica `care_offer.changed`"; "PATCH vazio → 422 path `quantity`"; "PATCH com outro `month` segue recusado"; "`performer_cbo: "12a456"` → 422 path `performer_cbo`"; "prestador grava `default_performer_cbo`". Bindings com `regulation_rnds.settings_changed`.
- [ ] **Step 2:** `bin/test api -w mod26c spec/requests/care_regulation/rnds_settings_spec.rb spec/requests/care_regulation/admin_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o evento é ligado a `[]` aqui; a Task 10 troca pelo consumidor).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add regulation rnds settings and performer cbo on offers and providers" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 6 · GET e PUT (step-up, Q13) de /regulation/admin/rnds_settings sem o gate do RNDS (valem antes de ligar); performer_cbo no POST com herança do mês anterior e PATCH só dele (Q12); default_performer_cbo no prestador."

---

### Task 7: Correspondência de situação e passos da falta

**Files:**
- Create: `app/services/regulation/rnds_status.rb`, `spec/services/regulation/rnds_status_spec.rb`

**Interfaces:**
- Produces: Valores 5 e 6. `Regulation::RndsStatus::Target = Data.define(:status, :at)`.

- [ ] **Step 1: Write the failing test** (`type: :model`, pedidos pelos comandos reais do `RegulationHelpers`; uma linha por situação da tabela do contrato §2): "`draft` → `:none`"; "`pending` e `authorized` → `pending`"; "`returned` → `returned-to-requester`"; "`scheduled` → `booked` com `at` do andamento `scheduled`"; "`completed` → `attended`"; "`denied` e `cancelled` → `cancelled`"; "`closed` pela segunda falta → `absence`"; "`external` com `send_external: false` → `:excluded`; com `true` → `pending`; `scheduled` regulado pelo estado → `booked`; `completed` → `attended`" (Q7); "regulado pelo estado `cancelled` ou `closed` com `send_external` desligado → `:excluded`" (Q7); "1ª falta (volta a `pending`) depois do último aceite → `steps == [absence]`"; "falta, reautorização e agendamento antes do envio → `steps == [absence]` e desejado `booked`"; "duas faltas antes do envio → desejado `absence` e `steps == []`" (descartado por igual ao desejado); "falta anterior ao `since` não volta".
- [ ] **Step 2:** `bin/test api -w mod26c spec/services/regulation/rnds_status_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (`tick_regulation_events!` nas specs com relógio parado).
- [ ] **Step 4:** o mesmo spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: map regulation states to mira statuses with miss steps" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 7 · RndsStatus com a tabela do §2 inteira e a falta como passo; externo pelo state_regulated (Q7)."

---

### Task 8: Fatos, documento RIRA, esqueleto e conferência

**Files:**
- Create: `app/services/regulation/{rira_facts,rira_bundle,rira_skeleton,rnds_validation}.rb`, `spec/services/regulation/{rira_bundle,rnds_validation}_spec.rb`

**Interfaces:**
- Consumes: Tasks 1, 5, 6, 7.
- `Regulation::RiraFacts.for(request:, settings:, credential:) -> RiraFacts` (`Data` com `inspect` filtrado): `patient_cpf`, `patient_cns`, `requester_cnes` (unidade solicitante), `author_cnes` (credencial), `requester_id`, `target_type`, `target_code`, `sigtap`, `sigtap_valid`, `performer_cnes`, `performer_cbo`, `priority`, `cid10`, `ciap2`, `requested_at`, `authorized_at`, `scheduled_at`, `status_times` (`{ mira_status => at }`), `own_network`. Toda leitura de banco do documento mora aqui (Valor 8).
- `Regulation::RiraBundle.build(facts:, sync:, status:, now:) -> Hash` (puro; `sync` responde `bundle_identifier`, `last_rnds_id`; Valor 7); `.digest(bundle) -> String` (Valor 9); `.for(request:, sync:, status:, now:)` = `build(facts: RiraFacts.for(…), …)`.
- `Regulation::RiraSkeleton.problems(bundle) -> [String]` (caminhos 1..1 do perfil ausentes; nunca valores).
- `Regulation::RndsValidation.call(facts:, status:) -> :ok | [:blocked, [String]]`, todos os motivos de uma vez, na ordem do `BLOCK_REASONS`: `missing_diagnosis` (sem CID e sem CIAP); `requester_unit_without_cnes`; `performer_without_cnes` (só `booked`/`attended`); `patient_without_cpf_or_cns`; `performer_cbo_missing` (só `booked`/`attended` com grupo em `CBO_GROUPS`); `specialty_without_sigtap` (SIGTAP resolvido que o `Targets.resolve` recusa ou que não está vigente na competência, Q9).

- [ ] **Step 1: Write the failing tests** (`type: :model`). `rira_bundle_spec.rb`: "o `pending` tem 4 entradas na ordem, Appointment `proposed`, ServiceRequest `active`/`proposal`, sem `performer`"; "cada situação usa o par do `STATUS_RELATION`"; "vermelho → `urgent`; amarelo → `routine`; a cidade troca amarelo para `urgent` e o documento segue"; "especialidade sem entrada → `0301010072` e `performerType` = CBO do alvo; com entrada → o SIGTAP configurado"; "pedido por especialidade agendado: o CBO do executante é o do alvo, mesmo com outro CBO no turno (Q12)"; "`booked` por procedimento na rede própria: `performer` = CNES da unidade do horário, CBO do vínculo do turno; sem turno → `performer_cbo` da oferta → `default_performer_cbo`"; "`booked` externo pelo estado: CNES do `care_provider_id`"; "`scheduled_at` no `Appointment.start`, `requested_at` no `authoredOn`"; "CID-10 tem precedência sobre CIAP-2"; "paciente só com CNS → identifier com o CNS"; "com `last_rnds_id` há `relatesTo` `Composition/<id>`"; "`Bundle.identifier.system` termina em `BRRNDS-<requester_id>`"; "nenhum `custodian` nem `section.title`"; "`digest` igual com outro `now` e outro `last_rnds_id`; muda com a remarcação, a cor, o CID e o CNES"; "`RiraSkeleton.problems` vazio em todas as situações". `rnds_validation_spec.rb`: um exemplo por motivo; "`pending` sem executante não pede CBO nem CNES do executante"; "procedimento do grupo 02 agendado sem CBO passa"; "motivos juntos numa lista".
- [ ] **Step 2:** `bin/test api -w mod26c spec/services/regulation/rira_bundle_spec.rb spec/services/regulation/rnds_validation_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: build rira bundles from regulation facts with skeleton and block reasons" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 8 · RiraFacts/RiraBundle (puro, Q6), digest, esqueleto e os seis motivos de parado; CBO pelo alvo especialidade → agenda → oferta → prestador (Q12)."

---

### Task 9: Exemplos e validação oficial na CI (`rira-validate`)

**Files:**
- Create: `app/services/regulation/rira_examples.rb`, `lib/tasks/regulation.rake`, `spec/tasks/regulation_rira_examples_spec.rb`
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- `Regulation::RiraExamples::CASES` (fatos em memória, sem banco, dados fictícios e CPF de teste válido): `pending_routine`, `pending_urgent`, `returned`, `booked_own`, `booked_external`, `attended_procedure_group03`, `cancelled`, `absence`, `excluded`, `pending_ciap2`, `booked_replacing` (com `last_rnds_id`). `Regulation::RiraExamples.write(dir, now:) -> [Pathname]`.
- Rake `regulation:rira_examples[dir]` (padrão `tmp/rira-examples`): escreve um JSON por caso e imprime só os nomes.
- Job `rira-validate` (`runs-on: ubuntu-24.04`, **sem** Postgres): `actions/checkout@v4`; `ruby/setup-ruby@v1` (`bundler-cache: true`); `actions/setup-java@v4` (`distribution: temurin`, `java-version: "21"`); `actions/cache@v4` em `~/validator` (chave `validator-7.0.6`) e `~/.fhir/packages` (chave `fhir-packages-rira-<sha256 do fixture>`); download de `https://github.com/hapifhir/org.hl7.fhir.core/releases/download/7.0.6/validator_cli.jar` se ausente; `bin/rails "regulation:rira_examples[tmp/rira-examples]"` com `RAILS_ENV=test` e as variáveis do job `rspec` (`RAILS_MASTER_KEY`, `ROTA_APP_PASSWORD`, `ROTA_PLATFORM_PASSWORD`); `java -jar ~/validator/validator_cli.jar tmp/rira-examples/*.json -version 4.0.1 -ig spec/fixtures/rira/br.gov.saude.rira.fhir-1.0.0-release.tgz -output tmp/rira-validation.json`; falha se `jq '[.. | objects | select(.resourceType? == "OperationOutcome") | .issue[]? | select(.severity == "error" or .severity == "fatal")] | length'` > 0; `actions/upload-artifact@v4` com `tmp/rira-examples` e `tmp/rira-validation.json` (evidência da homologação, decisão 3).

- [ ] **Step 1: Write the failing test** (`type: :model`): "a rake escreve os 11 arquivos"; "cada arquivo tem `RiraSkeleton.problems == []`"; "nenhum CPF além do de teste do caso"; "os sete `MIRA_STATUSES` aparecem; há `urgent` e `routine`; há rede própria e externa".
- [ ] **Step 2:** `bin/test api -w mod26c spec/tasks/regulation_rira_examples_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a rake, os casos e o job.
- [ ] **Step 4:** o mesmo spec → PASS; rodar à mão no worktree o validador (Java local, se houver) sobre os exemplos e corrigir o documento na Task 8 até zero erros; sem Java local, a primeira execução do job na PR é a prova (resultado no relato).
- [ ] **Step 5: Commit** `git commit -m "ci: validate generated rira examples with the official hl7 validator" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 9 · regulation:rira_examples (11 casos) e job rira-validate (Java 21 só ali, validator 7.0.6, pacote versionado); <erros 0 | pendente da PR>."

---

### Task 10: Recálculo por pedido e consumidor dos eventos

**Files:**
- Create: `app/services/regulation/rnds_sync.rb`, `app/jobs/regulation/rnds_sync_job.rb`, `spec/services/regulation/rnds_sync_spec.rb`, `spec/jobs/regulation/rnds_sync_job_spec.rb`, `spec/support/regulation_rnds_helpers.rb`
- Modify: `app/models/health_unit.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `Regulation::RndsSync.recompute!(request, create: Boolean, initial_load: false, now:) -> RegulationRndsSync | nil`: alvo pelo `RndsStatus.for`; `:none` sem linha → nil; `:excluded` sem `last_rnds_id` → linha `excluded` sem envio (cria só se já havia linha); `:excluded` com `last_rnds_id` → desejado `excluded`; `erased` quando havia `last_rnds_id` e o paciente perdeu CPF e CNS (Q8); grava `desired_status`, `desired_digest` (`RiraBundle.digest` do desejado), `pending_steps`; mudou o desejado → `enqueued_at = now`; desejado ≠ (`sent_status`, `sent_digest`) ou há passos → `queued` (`next_attempt_at = now`, `attempts = 0`) e `regulation_rnds.queued` na transição; igual → `idle` (ou `excluded`). Linha `sending` só tem os campos `desired_*`/`pending_steps` atualizados (o status é do envio).
- `RndsSync.affected_request_ids(event_name, payload) -> [String] | :all`: `regulation_request.*` → o pedido; `care_offer.changed` → pedidos com autorização nessa oferta; `care_provider.changed` → pela oferta ou por `regulation_requests.care_provider_id`; `health_unit.changed` → `requester_unit_id`, prestador `own` da unidade, horários na unidade; `appointment.*` → `AppointmentSyncJob::RESOLVERS` → `origin_regulation_request_id`; `care_regulation.settings_changed`, `regulation_rnds.settings_changed` → `:all`.
- `RndsSync.recompute_all!(now:)`: as linhas existentes; com `send_external` ligado e o interruptor ligado, também os regulados pelo estado não terminais sem linha (`initial_load: true`).
- `Regulation::RndsSyncJob` (`IdempotentConsumer`, molde `AppointmentSyncJob#perform(event_name:)`): cria linha só com `Regulation::RndsGate.enabled?(Current.city)`; linha existente é sempre recalculada (decisão 15). Ligado a: os 11 `regulation_request.*`, `care_offer.changed`, `care_provider.changed`, os 10 eventos de `AppointmentSyncJob::EVENTS`, `care_regulation.settings_changed`, `regulation_rnds.settings_changed`, `health_unit.changed`.
- `HealthUnit` `after_save` com `saved_change_to_cnes?` → `DomainEvents.publish("health_unit.changed", health_unit_id:)` (cobre o controller e o `Cnes::Apply`); binding novo.
- Helper: `rnds_regulation_city!(mock: true)` (`regulation_city!` + IBGE/UF + credencial `rnds_regulation` + `rnds_regulation` e `rnds_mock`), `rnds_regulation_credential!`, `fake_rnds!` (stub `to_rack` e `FAKE_RNDS_URL`).

- [ ] **Step 1: Write the failing tests.** `rnds_sync_spec.rb` (`type: :model`): "pedido promovido com o interruptor ligado → linha `queued` com `pending`"; "desligado → sem linha; linha antiga segue recalculada"; "remarcação de pedido aceito (mesma situação) → `queued` por digest (retificação)"; "trocar a cor de vermelho para azul num aceito → `queued`"; "nada mudou → `idle` e `enqueued_at` intacto"; "`external` com `send_external` desligado e aceito antes → desejado `excluded`; nunca aceito → `excluded` sem remessa; religado → volta a `pending` substituindo"; "paciente sem CPF/CNS num pedido já aceito → `erased`; nunca aceito → fica para a conferência (`blocked`)"; "linha `sending` não perde o status". `rnds_sync_job_spec.rb` (`type: :job`): "`care_offer.changed` recalcula só os pedidos da oferta"; "`health_unit.changed` do CNES da unidade solicitante recalcula o pedido"; "`regulation_rnds.settings_changed` recalcula todos"; "o mesmo evento duas vezes não duplica"; "evento publicado só com ids".
- [ ] **Step 2:** `bin/test api -w mod26c spec/services/regulation/rnds_sync_spec.rb spec/jobs/regulation/rnds_sync_job_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/requests/health_units_spec.rb spec/commands/cnes spec/jobs/care_regulation spec/architecture/current_city_assignment_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: recompute regulation rnds sync state from domain events" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 10 · RndsSync + RndsSyncJob (11 do pedido, oferta, prestador, 10 do 17, settings, health_unit.changed novo); retificação por digest, excluded (Q7) e erased (Q8)."

---

### Task 11: Carga inicial com ritmo

**Files:**
- Create: `app/jobs/regulation/rnds_initial_load_job.rb`, `spec/jobs/regulation/rnds_initial_load_job_spec.rb`
- Modify: `config/recurring.yml`

**Interfaces:**
- `Regulation::RndsInitialLoadJob` (`prepend EachCityJob`; `every 5 minutes`; `limits_concurrency to: 1, key: "rnds_regulation_initial_load"`): só com `RndsGate.enabled?` (lido fora da transação); cria, por `RndsSync.recompute!(…, create: true, initial_load: true)`, a linha dos pedidos sem linha que estão fora de `draft` e de `TERMINAL_STATUSES`, mais os terminais com `updated_at >= now - INITIAL_LOAD_DAYS`; regulados pelo estado só com `send_external`. Sem estado próprio: a linha existente é a marca de feito (religar depois de desligar pega o que nasceu no meio).
- O ritmo é do envio: o `RndsDeliverJob` (Task 12) reivindica no máximo `INITIAL_RATE_PER_MINUTE` linhas com `initial_load` por execução, depois das demais.

- [ ] **Step 1: Write the failing test** (`type: :job`): "ligar com 3 pendentes, 1 encerrado há 10 dias e 1 há 40 → 4 linhas `initial_load`"; "rodar de novo não cria nada"; "desligado → nada"; "sem credencial → cria as linhas (nada sai, Task 12)"; "regulado pelo estado só com `send_external`"; "70 linhas `initial_load` → a primeira execução do envio manda 60 (com o `RndsDeliverJob` da Task 12; marcar `pending` até lá)".
- [ ] **Step 2:** `bin/test api -w mod26c spec/jobs/regulation/rnds_initial_load_job_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** e a entrada `rnds_regulation_initial_load` no `recurring.yml`.
- [ ] **Step 4:** o mesmo spec → PASS (salvo o `pending`).
- [ ] **Step 5: Commit** `git commit -m "feat: load existing regulation requests when the rnds switch turns on" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 11 · carga inicial (não terminais + terminais de 30 dias) a cada 5 min, idempotente pela linha; ritmo de 60/min no envio."

---

### Task 12: Envio contínuo, cadeia de substituição e reconciliação

**Files:**
- Create: `app/services/regulation/{rnds_delivery,rnds_reconcile}.rb`, `app/jobs/regulation/rnds_deliver_job.rb`, `spec/jobs/regulation/rnds_deliver_job_spec.rb`, `spec/services/regulation/rnds_reconcile_spec.rb`
- Modify: `app/models/regulation_rnds_sync.rb`, `config/recurring.yml`, `config/initializers/domain_events.rb`

**Interfaces:**
- `RegulationRndsSync.claim!(limit:, initial_limit:, now:) -> [RegulationRndsSync]` (`queued`/`blocked` com `next_attempt_at <= now`, `FOR UPDATE SKIP LOCKED`, primeiro sem `initial_load` até `limit`, depois com até `initial_limit`; vira `sending` com `claimed_at = now`); `.release!(ids, token:)` (volta a `queued` sem contar tentativa); `.release_stale!(before:)`.
- `Regulation::RndsDeliverJob` (`prepend EachCityJob`; `every minute`; `limits_concurrency to: 1, key: "rnds_regulation_deliver", duration: STALE_SENDING`): só com `RndsGate.usable?`; anula `payload` de remessas aceitas há mais de `PAYLOAD_DAYS`; `release_stale!`; `claim!(limit: BATCH_SIZE, initial_limit: INITIAL_RATE_PER_MINUTE)`; `RndsDelivery.new(city).run(syncs)`; `ensure release!`.
- `Regulation::RndsDelivery#run(syncs) -> :done | :paused` (alvo por `Rnds::Mock.on?`, cliente `Rnds::Client.for(city, kind: "rnds_regulation", settings:)`, `BUDGET`), por linha:
  1. `RndsSync.recompute!` (estado fresco); nada a enviar → `idle`/`excluded`/`erased`.
  2. Próximo documento = `pending_steps.first` ou o desejado.
  3. `RndsValidation` → `[:blocked, reasons]` → `blocked`, `blocked_reasons`, `next_attempt_at = now + BLOCKED_RECHECK`, `regulation_rnds.blocked` na transição.
  4. `attempts.positive?` → `RndsReconcile.call(sync:, client:)` antes de montar.
  5. Monta (`RiraBundle.for`), `RiraSkeleton.problems` → `blocked` com `skeleton:<caminho>`.
  6. Remessa (`kind` `create` sem `last_rnds_id`, senão `replace`; `payload` cifrado; `sent_at`) e `post_bundle(bundle, practitioner_cns: client.responsible_cns)` fora de transação.
  7. `:accepted` → remessa `accepted_at`/`rnds_id`/`http_status 201`; linha `last_rnds_id`, `last_accepted_at`, `sent_status`, `sent_digest`, passo consumido, `attempts 0`; `queued` se ainda difere, senão `idle` (`excluded` quando enviou `excluded`); `regulation_rnds.accepted` (e `regulation_rnds.excluded`).
  8. `:rejected` com `REPLACED_CODE` → `RndsReconcile` e segue `queued`; outro → `rejected`, `last_error_codes` (`Rnds::Errors.codes`), `regulation_rnds.rejected`.
  9. `:unauthorized` → credencial `unauthorized` (só a do `set_at` lido, molde `Rnds::Delivery#pause!`), esta e as demais voltam a `queued`, `:paused`.
  10. `:transient` → `attempts + 1`, `next_attempt_at = now + Rnds::Backoff.wait(attempts)`; `enqueued_at <= now - GIVE_UP_AFTER` → `failed`.
- `Regulation::RndsReconcile.call(sync:, client:) -> :adopted | :none | :deduplicated | :transient`: `identifier_chain(system: BRRNDS-…, value: bundle_identifier, doc_type: profile.doc_type)`; o documento `final` que substitui o `last_rnds_id` (ou, sem `last_rnds_id`, o `final` sem `relatesTo`) é **adotado**: a última remessa não aceita vira aceita com esse `rnds_id`, a linha avança como no item 7; mais de um `final` sem relação → `delete_bundle` dos mais novos (remessa `delete`), fica o mais antigo; nenhum → `:none` (reenvia).

- [ ] **Step 1: Write the failing tests** (`type: :job`, contra o `FakeRnds::App`): "pedido pendente → `POST` sem `relatesTo`, 201, `idle` com `last_rnds_id` do `Location`, payload cifrado e evento só com ids"; "agendado depois → substituição com `relatesTo Composition/<id>` e o mesmo `Bundle.identifier.value`"; "1ª falta → dois envios em duas execuções: `absence` e depois `pending`, cada um substituindo o anterior" (substituição em ordem); "dois pedidos, um parado e um ok, na mesma execução → só o ok sai"; "`RECUSA` → `rejected` com `EHR-ERR999`; mudar o CID → volta a `queued` e sai"; "`TIMEOUT` → `attempts 1`; a próxima execução consulta `GET /identifier`, adota sem novo `POST` e fica `idle`" (Review Focus 1); "`DUPLICADO` → reconciliação apaga o mais novo (204) e segue"; "EDS-MSG009 (o `last_rnds_id` apontando para um já substituído) → adota a cabeça da cadeia, não fica `rejected`" (Review Focus 3); "401 duas vezes → credencial `unauthorized`, linhas `queued` sem tentativa, `RndsGate.usable?` falso" (Review Focus 4); "5xx → backoff; com `enqueued_at` de 16 dias → `failed`"; "parado → 1 h depois revalida e sai quando o CNES aparece"; "sem credencial → nada sai e as linhas ficam"; "`rnds_mock` ligado → vai ao simulado"; "payload aceito de 31 dias → NULL"; "duas execuções simultâneas nunca enviam o mesmo pedido"; o `pending` da Task 11 (60 por minuto). `rnds_reconcile_spec.rb`: os quatro resultados.
- [ ] **Step 2:** `bin/test api -w mod26c spec/jobs/regulation/rnds_deliver_job_spec.rb spec/services/regulation/rnds_reconcile_spec.rb spec/jobs/regulation/rnds_initial_load_job_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** e a entrada `rnds_regulation_deliver` (`every minute`, `queue: default`) no `recurring.yml`; os cinco `regulation_rnds.*` de envio ligados a `[]`.
- [ ] **Step 4:** os mesmos specs + `spec/jobs/rnds spec/services/rnds spec/architecture/current_city_assignment_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: deliver regulation documents to rnds as an ordered replacement chain" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 12 · RndsDeliverJob por minuto (SKIP LOCKED, 20 + 60 da carga), cadeia de substituição, GET /identifier no timeout e no EDS-MSG009, DELETE só de duplicado, pausa em 401/403, failed em 15 dias."

---

### Task 13: Alarme D-1

**Files:**
- Create: `app/services/regulation/rnds_alert.rb`, `spec/services/regulation/rnds_alert_spec.rb`

**Interfaces:**
- `Regulation::RndsAlert.counts(scope = RegulationRndsSync.all, now:) -> { queued:, blocked:, rejected:, failed:, late:, accepted: }` (`queued` inclui `sending`; `accepted` = `idle` com `last_rnds_id`, Q11). `late`: status fora de `idle`/`excluded`/`erased`, com prazo = `(enqueued_at no fuso da cidade).to_date + 1` às `LATE_HOUR`:00, e `now >= prazo`; linha `initial_load` com `created_at > now - INITIAL_GRACE` fica fora. `RndsAlert.late?(sync, now:)` para a lista.

- [ ] **Step 1: Write the failing test** (`type: :model`, `travel_to` no fuso da cidade): "mudança às 10:00 de D: não atrasada às 22:59 de D+1; atrasada às 23:00 de D+1"; "`blocked` e `rejected` também atrasam"; "`idle` nunca"; "`initial_load` criada há 71 h → fora; há 73 h → dentro"; "`erased` e `excluded` fora"; "as contagens por status".
- [ ] **Step 2:** `bin/test api -w mod26c spec/services/regulation/rnds_alert_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: count late regulation rnds deliveries against the next-day deadline" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 13 · RndsAlert com prazo às 23:00 do dia seguinte no fuso da cidade; carga inicial fora por 72 h."

---

### Task 14: Rotas de situação, lista e reenvio

**Files:**
- Create: `app/services/regulation/rnds_json.rb`, `app/commands/regulation/retry_rnds_sync.rb`, `app/controllers/care_regulation/{rnds_status,rnds_syncs}_controller.rb`, `spec/requests/care_regulation/rnds_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- No `scope "/regulation"`: `get "rnds/status"`, `get "rnds/syncs"`, `post "rnds/syncs/:id/retry"` (antes de rotas com `:id` genérico). Os dois controllers herdam `CareRegulation::BaseController` e incluem `RegulationRndsGate`; leitura (`municipal_admin`, `care_regulator`) com `require_rnds_regulation_enabled!`; reenvio com `require_rnds_regulation!`, `municipal_admin` e step-up (Q10). Nomes sem `Rnds` como módulo dentro de `CareRegulation` (evita sombrear `::Rnds`).
- `GET /regulation/rnds/status` → `{ usable, missing, counts: RndsAlert.counts, late_banner: counts[:late].positive?, initial_load_pending: <linhas initial_load não idle> }`.
- `GET /regulation/rnds/syncs?status=&reason=&before=&limit=` (`query_only!("status", "reason", "before", "limit")`; `status` em `SYNC_STATUSES`; `reason` em `BLOCK_REASONS` ou `ERROR_CODE`; ordem `enqueued_at DESC, id DESC`; cursor `CareRegulation::Cursor.encode_page`) → `{ items, next_before }`; item `RndsJson.item(sync)`: `{ id, regulation_request_id, request_ref, status, sent_status, desired_status, blocked_reasons, last_error_codes, attempts, enqueued_at, last_accepted_at, next_attempt_at, late, fix: { offer_id, provider_id, unit_id, settings } }` — **sem paciente, CID, alvo nem payload**; `request_ref` = `regulation_request_id.delete("-")[0, 8].upcase` (Q11), calculado pelo api, para o rótulo e o link ao detalhe.
- `POST /regulation/rnds/syncs/:id/retry` → `Regulation::RetryRndsSync.call(sync:, by:) -> Result.ok(sync:) | fail(:not_retryable)`: só `rejected`/`failed` → `queued`, `attempts 0`, `next_attempt_at now`, conteúdo e `enqueued_at` intactos; `regulation_rnds.queued`.

- [ ] **Step 1: Write the failing tests** (`type: :request`): "regulador vê as contagens e a faixa"; "interruptor desligado → 403 `feature_disabled` com `feature: rnds_regulation`"; "credencial recusada → a situação responde com `usable: false` e `missing`" (Q10); "item traz `request_ref` com 8 hex maiúsculos do id do pedido" (Q11); "lista filtra por `status=blocked` e `reason=performer_cbo_missing` e traz o `fix.offer_id`"; "`reason=EHR-ERR999` filtra recusados"; "cursor de duas páginas"; "query com `cpf=` → 422 `invalid_request` path `cpf`"; "nenhum item tem CPF, CNS, CID nem nome (varredura do JSON)"; "reenvio de `rejected` com step-up → `queued`; de `idle` → 409 `not_retryable`; sem step-up → `401 mfa_required`; regulador → 403; credencial recusada → 403".
- [ ] **Step 2:** `bin/test api -w mod26c spec/requests/care_regulation/rnds_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec + `spec/requests/care_regulation` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add regulation rnds status, list and retry routes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 14 · GET /regulation/rnds/status e /syncs (sem paciente, com atalho de correção) e POST retry com step-up; leituras com enabled? mesmo pausado, reenvio com usable? (Q10); request_ref e accepted (Q11)."

---

### Task 15: Semente de dev para a prova contra o simulado

**Files:**
- Create: `lib/regulation_rnds_crew.rb`, `spec/lib/regulation_rnds_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- `RegulationRndsCrew.seed_current_city(slug:)` depois da semente da regulação: credencial `rnds_regulation` de dev (PFX autoassinado, `requester_id` `12345`, CNS válido de dev), `rnds_regulation` e `rnds_mock` ligados, configurações com uma especialidade mapeada, `performer_cbo` numa oferta e `default_performer_cbo` num prestador, e pedidos que caiam em cada motivo de parado, um com `bundle_identifier` `RECUSA-<uuid>` e um `TIMEOUT-<uuid>`. `summary_lines` sem segredo de staging ou produção.

- [ ] **Step 1: Write the failing test** (`type: :model`): "idempotente"; "há linha para cada motivo de parado"; "os marcadores existem"; "nenhum segredo nas `summary_lines`".
- [ ] **Step 2:** `bin/test api -w mod26c spec/lib/regulation_rnds_crew_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** o mesmo spec → PASS.
- [ ] **Step 5: Commit** `git commit -m "chore: seed regulation rnds dev data for the simulator proof" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 15 · RegulationRndsCrew (credencial de dev, mock, cada motivo de parado, RECUSA/TIMEOUT)."

---

### Task 16: Sigilo, eventos e R18 (§9)

**Files:**
- Create: `spec/security/regulation_rnds_secrecy_spec.rb`

- [ ] **Step 1: Write the test** (`type: :request`, molde do `vector_control_secrecy_spec.rb`, constantes num módulo `RegulationRndsSecrecy`, ASCII, palavras únicas): CPF e CNS do paciente, CID marcado, justificativa marcada, `requester_id` e `responsible_cns` marcados, senha do PFX marcada. Cenário pelas rotas e jobs reais: credencial gravada e testada; configurações; pedido promovido, autorizado, agendado, com falta, recusado (`RECUSA`), com timeout (`TIMEOUT`) e reenviado; carga inicial; leituras e reenvio. Provas:
  - `capture_log` sem nenhuma marca nem `Bearer ` seguido de token real;
  - eventos publicados do módulo ⊂ `RndsCatalog::DOMAIN_EVENTS` + `integration_credential.changed` + `health_unit.changed`, só ids (e `user_id`);
  - `regulation_rnds_syncs` e `regulation_rnds_deliveries` sem texto da RNDS (só `ERROR_CODE`);
  - payload só cifrado no banco; NULL 30 dias depois do aceite;
  - nenhuma resposta traz marca, payload nem paciente; toda recusa 4xx só com `error`, `path`, `feature`;
  - nenhuma URL chamada no fake leva CPF, CNS ou CID (só `system`, `value` do `bundle_identifier` e `docType`);
  - grep multilinha de `Platform.audit(` nos arquivos do módulo: só `rnds_regulation.credential_set`, presente no R18, sem campo com `name`.
- [ ] **Step 2:** `bin/test api -w mod26c spec/security/regulation_rnds_secrecy_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture` → PASS (falha → correção na task dona, volta aqui).
- [ ] **Step 3: Commit** `git commit -m "test: pin regulation rnds secrecy across logs, events, urls and responses" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26c · Task 16 · sigilo fixado (paciente, CID, credencial e texto da RNDS fora de log/evento/URL/resposta; payload cifrado e purgado)."

---

### Task 17: Entrega — suíte completa e passagem aos fronts (para antes do merge)

**Files:** nenhum código novo.

- [ ] **Step 1: Suíte completa:** pedir à sessão "Testes", por `SendMessage`: `api · mod26c/feat/mod-26c-mira · --full · módulo 26c (migração de cidade, CityEncryption, Platform::Features e rnds_mock, IntegrationCredential, 26a (oferta, prestador, eventos), Rnds::Client e fake_rnds do 29, health_unit.changed, spec/support, initializers, CI rira-validate)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador com o resumo e o hash, e as sessões do dashboard e do maintenance (porta 3063; simulado na 8094).
- [ ] **Step 2: Parar.** Merge, push e cards só com autorização do usuário; os cards do 26c seguem **In Progress** até o último app de cada um; nenhum vai a Verified por este plano. Os gates de go-live da spec §6 (credencial de homologação e produção, header `Authorization` com o CNS do responsável, IP de saída fixo, cores e SIGTAP conferidos com a cidade piloto, ausência de envio paralelo), o `docType` (Q6), a data em `Appointment.start` (decisão 18) e o card da correção do `requester_id` na vacina (spec "Fora desta entrega") vão ao board #2 no fechamento do módulo (outra sessão), com o ADR novo do 26c.

**Relato ao coordenador:** "api 26c · Task 17 · suíte completa <verde N/0 | falhas>; hash <…>; rira-validate <verde | erros>; pronto para dashboard e maintenance; parado antes do merge."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptor (2), `rnds_mock` (2), credencial e auditoria (3), cliente por `kind` e cache (3); §2 tabela e remessas (5), correspondência (7), passos (7, 10), gatilhos (10), saída da regulação municipal (10), paciente apagado (10, D7), carga inicial (11); §3 job e claim (12), primeiro envio e substituição (8, 12), timeout e `GET /identifier` (4, 12), `DELETE` só de duplicado (4, 12), documento (1, 8), CBO (8), configurações (5, 6), colunas do 26a (5, 6), conferência e `BLOCKED_RECHECK` (8, 12), respostas (12); §4 alarme (13); §5 rotas (6, 14, 3); §8 fake (4), CI (9), specs (todas); §9 eventos (6, 10, 12, 14) e sigilo (16); suíte (17); Q1–Q14 seguidas (Q1: 5; Q2: 8; Q3: 10; Q4: 9; Q5: 3; Q6: 1, 5, 8, 9; Q7: 7, 10; Q8: 10; Q9: 8; Q10: 14; Q11: 13, 14; Q12: 6, 8; Q13: 6; Q14: 2).
2. **Testes do §8:** cada linha da tabela de situação (7); falta dupla (7, 12); substituição em ordem (12); timeout com `GET /identifier` (12); retificação por digest (10); `excluded` (10); carga inicial com ritmo (11, 12); alarme com `initial_load` (13); motivos de parado (8); sigilo (16); suíte (17).
3. **Nomes:** `Regulation::{RndsGate, RndsLimits, RndsCatalog, RiraProfile, RndsStatus, RiraFacts, RiraBundle, RiraSkeleton, RndsValidation, RndsSync, RndsDelivery, RndsReconcile, RndsAlert, RndsJson, RiraExamples, SaveRndsSettings, RetryRndsSync, RndsSyncJob, RndsInitialLoadJob, RndsDeliverJob}`; `Rnds::Client.for(city, kind:)`, `#identifier_chain`, `#delete_bundle`, `#responsible_cns`; `Rnds::Errors.codes`; `RegulationRndsSync.{claim!, release!, release_stale!}`; `RegulationRndsGate#{require_rnds_regulation!, require_rnds_regulation_enabled!}`; `CareRegulation::{RndsStatusController, RndsSyncsController, Admin::RndsSettingsController}`.
4. **Ordem:** 0 → 1 → 2 → 3 → 4 → 5 (destrava os `pending` da 3) → 6 → 7 → 8 → 9 → 10 → 11 → 12 (destrava o `pending` da 11) → 13 → 14 → 15 → 16 → 17.

## Divergências propostas ao contrato

> D1–D14 **resolvidas no §10 do contrato (Q6–Q14; docs `5d49ef7`)**. Cada item guarda o problema, a evidência e a proposta; a marca diz qual Q resolveu e, quando a decisão difere da proposta, o que vale.

D1. **[Resolvida pela Q6.]** **`bundle_identifier` uuid × marcador do simulado.** O §2 fixa `bundle_identifier (uuid, fixo)` e o §8 põe o marcador de falha (`RECUSA`, `TIMEOUT`, `DUPLICADO`) no `Bundle.identifier.value`, que é esse campo; uma coluna `uuid` não guarda `RECUSA-…`. O exemplo oficial usa valor livre (`AMB_271831370_0701172`, `Bundle-example-rira-posterior-2025-com-cid.json`). **Proposta:** `varchar(64)` com CHECK `^[A-Za-z0-9_-]{1,64}$`, uuid por padrão; marcador só na semente e nas specs (Tasks 4, 5, 15).

D2. **[Resolvida pela Q6.]** **O pacote `br.gov.saude.rira.fhir#1.0.0-release` não está no registro de pacotes FHIR.** `packages.fhir.org/br.gov.saude.rira.fhir` e `packages2.fhir.org/packages/br.gov.saude.rira.fhir` respondem 404; o `package.tgz` do IG traz `"notForPublication": true` e `"date": "20261123112114"` (o build é refeito com a mesma versão; pesquisa, linha 9). O validador com `-ig br.gov.saude.rira.fhir#1.0.0-release` não acharia o pacote, e baixar do site a cada execução deixaria a CI mudar sozinha. **Proposta:** versionar o tgz em `spec/fixtures/rira/` (2,8 MB, licença CC0) com o SHA-256 em `config/rnds/rira_1_0.yml`, e passar o arquivo ao `-ig` (Tasks 1, 9). Atualizar é uma mudança consciente, com spec.

D3. **[Resolvida pela Q7.]** **"Externo" não é só a situação `external`.** O pedido regulado pelo estado segue em `scheduled` e `completed` com `state_regulated = true` (`schedule_external.rb:48-50`, `record_outcome.rb:36`), e a 1ª falta o devolve a `external` (`misses.rb:44`). A tabela do §2 só fala de `external`. **Proposta:** o critério é `state_regulated`: desligado → fora (ou `excluded` se já enviado); ligado → a mesma tabela do municipal (`external` → `pending`, `scheduled` → `booked`, `completed` → `attended`, `cancelled` → `cancelled`, `closed` por falta → `absence`) (Task 7).

D4. **[Resolvida pela Q6.]** **Assinatura do montador.** O §3 dá `Regulation::RiraBundle.build(request:, sync:, status:, now:)`. A rake `regulation:rira_examples` roda no job `rira-validate`, que não tem Postgres (Q4 isola o Java num job à parte). **Proposta:** `build(facts:, sync:, status:, now:)` puro sobre `Regulation::RiraFacts`, com `RiraBundle.for(request:, sync:, status:, now:)` como o atalho do contrato (Task 8).

D5. **[Resolvida pela Q6.]** **Colunas de controle do claim.** O §2 não lista `claimed_at` nem `created_at`/`updated_at`; o molde do 29 usa um token de posse no claim (`rnds_outbox_entry.rb:63-86`) e o `updated_at` para devolver `sending` esquecido. **Proposta:** `claimed_at` (token) e os timestamps (Task 5).

D6. **[Resolvida pela Q6.]** **`docType` do `GET /identifier`.** O IG mostra `docType={{modelo}}` sem o valor para o RIRA (`integracao.html`). O §1 pede o perfil com `confirmed: true`. **Proposta:** `doc_type` no YAML marcado `hypothesis: true` (valor inicial `RIRA`), o resto confirmado; gate de homologação (Tasks 1, 17).

D7. **[Resolvida pela Q8.]** **Quando o estado `erased` acontece.** O §2 e a Q3 reservam `erased` para "o pedido perder a identificação por outro caminho", mas paciente sem CPF e sem CNS também é o motivo `patient_without_cpf_or_cns`. O `Citizens::Erase` mantém o CPF do paciente (`erase.rb:28`, `rota_patient_guard`). **Proposta:** `erased` só quando o pedido já tinha documento aceito (`last_rnds_id`) e o paciente ficou sem CPF e CNS; nunca aceito → `blocked` com o motivo (Task 10).

D8. **[Resolvida pela Q9.] Também: vale quando o SIGTAP não está vigente na competência.** **`specialty_without_sigtap` nunca dispararia.** O §3 dá o padrão `0301010072` "para todo CBO sem entrada", então toda especialidade sempre tem SIGTAP. **Proposta:** o motivo vale quando o SIGTAP resolvido (configurado ou padrão) não passa no `CareRegulation::Targets.resolve(type: "procedure")` da competência ativa; o PUT das configurações já recusa SIGTAP não regulável (Tasks 6, 8).

D9. **[Resolvida pela Q10.] Só `/status` e `/syncs` (e o reenvio, com `usable?`) exigem o gate; as configurações valem antes de ligar.** **Leituras com a credencial recusada.** O §5 exige `RndsGate.usable?` em todas as rotas, salvo credencial e configurações. Em 401/403 o envio pausa marcando a credencial `unauthorized` (§3), o que derruba o `usable?`: a tela de situação viraria `feature_disabled` justo quando há pendência a mostrar. **Proposta:** `GET status` e `GET syncs` com `enabled?` (e `usable`/`missing` na resposta); o reenvio com `usable?` (Task 14).

D10. **[Resolvida pela Q11.]** **Total de aceitos.** O §4 conta `queued`, `blocked`, `rejected`, `failed` e `late`; a spec §3.5 e o §6 pedem "totais aceitos". **Proposta:** `accepted` (linhas `idle` com `last_rnds_id`) em `RndsAlert.counts` (Task 13).

D11. **[Resolvida pela Q11.] Diferente: em vez do `id`, a lista mostra `request_ref` (8 hex maiúsculos do id), com o link para o detalhe.** **"Número do pedido" na lista.** O §5 pede a lista com o número do pedido; `regulation_requests` não tem número (`db/city_schema.rb`, tabela `regulation_requests`), só o `id` uuid, que é o que o dashboard já usa. **Proposta:** `regulation_request_id` como identificação na lista; um número curto, se quiserem, é outra decisão (Task 14).

D12. **[Resolvida pela Q12.] Diferente: o CBO do alvo `specialty` é a primeira fonte (antes da agenda); a herança do mês anterior entra; o POST aceita `performer_cbo` e o PATCH muda só ele, sem `quantity`.** **CBO do executante no pedido por especialidade e oferta mensal.** A decisão 14 dá três fontes (agenda, oferta, prestador). `care_offers` é mensal (`idx_care_offers_unique` com `month`), então o `performer_cbo` teria de ser regravado todo mês. No pedido por especialidade, o próprio CBO do alvo é o executante natural. **Proposta (a decidir; o plano segue as três fontes do contrato):** (a) o CBO do alvo `specialty` como quarta fonte; e/ou (b) a oferta nova herdar o `performer_cbo` da oferta do mês anterior do mesmo prestador e alvo.

D13. **[Resolvida pela Q8.]** **`closed` "por outro motivo".** O §2 separa `closed` pela segunda falta de `closed` por outro motivo, mas no código só `Misses.apply!` e o `AppointmentSync` levam a `closed`, sempre com `misses = 2` (`misses.rb:42`, `appointment_sync.rb:177`). **Proposta:** manter a linha `closed` sem falta → `cancelled` como defesa, sem caminho de teste pelo produto (spec com `update_columns`), e registrar que hoje não ocorre.

D14. **[Resolvida pela Q6.]** **Campos do documento que o contrato não fixa.** O perfil exige `Composition.author` por identifier (1..1) e proíbe `custodian` e `section.title` (0..0; `StructureDefinition-BRRegulacaoAssistencial.json`); `Appointment.participant` só aceita o paciente (`PCT`), então o executante vai em `ServiceRequest.performer`. **Proposta:** `author` = `requester_cnes` da credencial; executante só no `ServiceRequest.performer`; sem `supportingInformation:regulatoryOrganization` nem `event.detail` (opcionais; o 26a não tem CNES de central de regulação) (Task 8).
