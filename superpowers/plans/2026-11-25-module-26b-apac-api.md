# Módulo 26b — APAC, OCI, teto em dinheiro e AIH (api) Implementation Plan

> **Nota do coordenador (Q14):** acrescente `GET /regulation/apac_options` (motivos de saída da OCI, clínicas do SIH, programas FAEC e fontes orçamentárias, lidos dos YAMLs do config), com spec de forma, na task das rotas de configuração. A rota de prévia (`authorization_preview`), o número inteiro da faixa e o `complement_cents_per_unit` já estão no plano (Q8, Q9, Q13).

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITOS: o 27b e o 26c na `origin/main` do api; os três insumos oficiais no `vendor/`.** Ao escrever este plano (2026-11-24) a `origin/main` era `ca2a92f0`, **sem** o 27b e **sem** o 26c: última migração de cidade `20261123900003_add_zoonoses_care.rb` (`db/city_schema.rb:13`), de plataforma `20261113900001_create_field_maps.rb`, o `CATALOG` terminando em `zoonoses` (`app/services/platform/features.rb:230`) e nenhum módulo `Regulation` no app. O 27b acrescenta as migrações de cidade `20261125900001..900004` e a de plataforma `20261125600001`; o 26c, a de cidade `20261126900001`, o interruptor `rnds_regulation` no fim do `CATALOG` e o namespace `Regulation::` (`RndsLimits`, `RndsCatalog`, `RiraFacts`, `RiraBundle`, `RndsValidation`, `RndsSyncJob`, `RegulationRndsSetting`). Este plano **parte desse estado**; a Task 0 confere e para se faltar algo. Nomes conferidos no código (`ca2a92f0`):
> - **SIGTAP (plataforma):** `Terminology::SigtapReader#write(release)` (lê `tb_registro`, `tb_procedimento`, `rl_procedimento_ocupacao`, `rl_procedimento_cid`, `rl_procedimento_registro` e, se houver, `rl_procedimento_detalhe` com `CO_DETALHE`; `sigtap_reader.rb:14-15, 23-47`); `each_fixed(base)` pelo `<base>_layout.txt` (`:73-85`); `amount_cents` (`:111-117`); `Terminology::Import.call(kind:, version:, path:, by:)` (`import.rb:19`), `Import::Invalid`, `Import.insert`, `Import::BATCH`; `Terminology::Sigtap.release_for(competence)` (última ativa ≤ competência, `sigtap.rb:12-16`); `TerminologyRelease.active`; tabelas `sigtap_procedures` (`amount_sa_cents`, `complexity`, `max_quantity`…), `sigtap_procedure_{attributes,cbos,cids,instruments}` (`db/platform_schema.rb:488-530`); `lib/sigtap_sample.rb` (`LAYOUTS` já traz `VL_SH`, `VL_SP` e `tb_detalhe`; `write_to(dir, competence:)`).
> - **26a:** `CareRegulation::Targets.{resolve(type:, code:, on:), search(type:, query:, on:, limit:)}`, `resolve_procedure` e `rules` privados (`targets.rb:19, 66, 70-82, 139`); `config/regulation/procedures.yml` (`groups`, `bpa_instruments`, `apac_principal_instruments` "só referência (26b)"); `CareRegulation::{SaveDraft::KEYS (:11), Promote.call(consultation:, by:, addendum:, now:, dental_care:) (promote.rb:21), Complement::KEYS (complement.rb:21), Authorize.call(request:, offer_id:, version:, by:, color:, color_reason:, skip_reason:, now:) (authorize.rb:34), Decide::LOCK = "FOR NO KEY UPDATE" (decide.rb:38), Misses.apply!/record!, AppointmentSync (libera em :211-215), Quota.used, Diagnosis.check, Requesters.check, Json.{request, list_item, authorization}, Catalog::{PROVIDER_KINDS, REASON, PROCEDURE_CODE, MONTH}}`; `CareRegulation::BaseController` (`render_failure`, `FAILURE_STATUS`, `body`, `body_param`, `query_only!`, `page_limit`, `regulator?`, `admin?`, `uuid?`) e `MfaStepUp#require_step_up!`; `CareRegulationGate#require_care_regulation!`; `CareRegulation::GuidePdf` (molde de PDF na folha de `Consultations::Print.document(footer, title:)`); rotas `scope "/regulation", module: :care_regulation` (`config/routes.rb:484-516`) e `consultations/:id/regulation_requests` (`:171-174`). `Consultations::Finalize` e `AddAddendum` leem os interruptores fora da transação e chamam `Promote` e `Signatures::OpenRequest` (`finalize.rb:43-72`).
> - **Faturamento (22):** `Billing::Critic::{REASONS, Facts (sem default), call(facts, closing:, cache:)}` (`critic.rb:15-47`); `Billing::Record.insert!` (`record.rb:43`); `Billing::Recalculate.{call, outcome}`; `Billing::Lines.build` e `Billing::BpaFile::ILine` (a autorização sai `blank(13)`, `bpa_file.rb:79`); `Billing::Establishment.of`; `BillingExport` (`encrypts :file`, `bytes`); `performed_procedures.authorization_number varchar(13)` (`city_schema.rb:3348`).
> - **Assinatura (19b):** `Signatures::DocumentTypes::MODELS` + `consultation_id(document)` (`document_types.rb:6-28`); `Signatures::Canonical.{for, header_for(unit:, author:, cbo_code:, patient:), validate!, SCHEMA_FILES}` (`canonical.rb:20-45, 148`); `Signatures::Documents.for` (`documents.rb:18-36`); `Signatures::OpenRequest.call(document, now:, usable:)` → `:unusable | :manual | :exists | :created` (`open_request.rb:18-33`); `Signatures::Signing::UNCHAINED` e `chronological` (`signing.rb:113-129`); CHECKs `ck_signature_requests_document_type` e `ck_signature_requests_consultation` (`city_schema.rb`, tabela `signature_requests`); vetor em `spec/services/signatures/canonical_vector_spec.rb` com os fixtures em `spec/fixtures/clinical/`.
> - **Guardas:** `CityEncryption::CITY_KEYED_TARGETS` (`city_encryption.rb:149-152` para o 26a); `DomainEvents.bind` dos eventos do 26a (`config/initializers/domain_events.rb:396-400`, que o 26c troca por `Regulation::RndsSyncJob`); `spec/initializers/domain_events_bindings_spec.rb`; `spec/jobs/reencryption_job_spec.rb`; `spec/architecture/{city_encrypted_attributes_guard,current_city_assignment}_spec.rb`; `CityProfile.current.{ibge_code, uf}`; `PecSetting.current = first` (molde de linha única).

**Goal:** O lado api do módulo 26b: SIGTAP completa, interruptor `care_regulation_apac`, pedido por laudo (APAC, OCI, AIH) na mesma fila do 26a, faixas de numeração com dígito verificador, espelho da FPO com reserva de teto, autorização "antes" ou "depois", OCI com secundários e encerramento, número no BPA-I e bloqueio do secundário, remessa APAC da rede própria, fila nominal da OCI, laudo em PDF assinável e a modalidade 04 do MIRA.

**Architecture:** Uma migração de plataforma (SIGTAP) e uma de cidade (laudo, procedimentos, faixas, colunas da autorização, FPO, reservas, configurações, remessas, ligação do procedimento realizado, tipo assinável), ambas irreversíveis. O domínio novo mora em `Regulation::*` (o namespace do 26c): regras puras (`ApacCatalog`, `LaudoTargets`, `LaudoRules`, `FifthDigit`, `CheckDigit`, `FpoPick`, `Validity`) e serviços com trava (`AuthorizationNumber`, `Reservations`, `PerformedLink`, `OciBlock`). O `Authorize` do 26a ganha os passos do §4 do contrato; o 22 ganha a ligação, o número no BPA-I e a crítica `oci_secondary_active`; o 19b ganha o `RegulationLaudo` como tipo assinável.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade, `btree_gist` já habilitado), Solid Queue, Prawn (já usado), `json_schemer` (já usado), RSpec via `bin/test`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-25-module-26b-apac-oci-aih-design.md` (decisões 1–22). Contrato entre apps — **fonte única**: `docs/superpowers/plans/2026-11-25-module-26b-apac-contracts.md` (§1–§12, Q1–Q13). Pesquisa: `docs/pesquisa/2026-11-24-apac-oci-teto-mac-26b.md` (achados 1–18). Plano irmão: `docs/superpowers/plans/2026-11-25-module-26b-apac-contracts-repo.md` (esquema `rotasaude.regulation_laudo.v1`, `clinical-v1.3.0`). **Divergências** com o contrato: seção final (D1–D13), **todas resolvidas pelas Q6–Q13 do §12 do contrato (docs `60908e1`)**; as tasks já seguem as decisões. Onde a decisão completa ou muda a proposta: Q6 (ZIP completo da SIGTAP no vendor; feriados nacionais numa lista fixa; a busca devolve o `authorization_kind`); Q7 (listas `oci_consultation_codes.yml` e `oci_oncology.yml`; `oci_close` com `outcome` e `exit_reason`; 25 dias do 1º procedimento realizado; vencida no alarme; recusas próprias); Q8 (a faixa devolve também o número inteiro); Q9 (`other` sem reserva; `faec_programs.yml`; `complement_cents_per_unit` na linha; a reserva fica até o encerramento, inclusive depois da 1ª falta, e migra pelo procedimento realizado; AIH até 3 competências depois da alta); Q10 (remessa por procedimento realizado na competência, `municipal_admin` ou `billing_clerk`, `not_own_provider`); Q11 (`specialty_without_sigtap` só para alvo de especialidade); Q12 (paciente sem CPF → laudo impresso); Q13 (complemento aceita laudo e procedimentos; oferta do mês corrente e dos 2 seguintes; prévia da autorização; o regulador lê o momento e a FPO).

## Valores fixados por este plano

1. **Interruptor** (fim do `CATALOG`, depois de `rnds_regulation`): `Entry.new(key: "care_regulation_apac", description: "Regulação por laudo (APAC, OCI e AIH)", requires: %w[feature:care_regulation sigtap_billing_data])`. Requisito novo no `missing_for`: `when "sigtap_billing_data" then "sigtap_billing_data_missing" unless Terminology::SigtapBilling.loaded?` (plataforma, sem abrir a cidade). `Regulation::ApacGate::KEY = "care_regulation_apac"`, `usable?(city)`, `enabled?(city)` (molde `CareRegulation::Gate`).
2. **`Terminology::SigtapBilling.loaded?(today: Time.zone.today) -> Boolean`**: a release de `Terminology::Sigtap.release_for(today.strftime("%Y%m"))` tem ao menos uma linha em `sigtap_financings`.
3. **`Regulation::ApacCatalog`**: `KINDS = %w[bpa apac oci aih]`; `LAUDO_KINDS = %w[apac oci aih]`; `RANGE_KIND = { "apac" => "apac", "oci" => "apac", "aih" => "aih" }`; `FIFTH_DIGITS = { "apac" => %w[2 7 6 9], "aih" => %w[1 5 8] }`; `STAGES = %w[issued awaiting_review review_denied]`; `FINANCINGS = %w[mac faec credits other]`; `CEILING_FINANCINGS = %w[mac faec credits]` (`other` não reserva teto, Q9); `FINANCING_DEFAULT = { "04" => "faec", "06" => "mac" }` (outro código → `"other"`, Q9); `BUDGET_SOURCES = %w[01 02 03 04]` (Q9); `FPO_LEVELS = { "procedure" => 10, "organization_form" => 6, "subgroup" => 4, "group" => 2 }` (ordem = especificidade; valor = tamanho do código); `CEILING_PROVIDER_KINDS = %w[own contracted consortium]`; `MOMENTS = %w[before after]`; `ROLES = %w[principal secondary]`; `PERFORMED_SOURCES = %w[performed_procedure manual]`; `CHARACTERS = %w[elective urgent]`; `CLINICAL_SUMMARY = (20..4000)`; `FINDINGS_MAX = 4000`; `PROCEDURE_QUANTITY = (1..999)`; `MAX_PROCEDURES = 30`; `OCI_MIN_SECONDARIES = 2`; `ONCOLOGY_ALERT_DAYS = 25`; `VALIDITY_EXTRA_MONTHS = { "apac" => 1, "oci" => 1 }`; `AIH_AFTER_DISCHARGE_MONTHS = 3` (Q9); `OFFER_MONTHS_AHEAD = 2` (Q13); `OCI_OUTCOMES = %w[concluded not_concluded]` (Q7); `DOMAIN_EVENTS = %w[regulation_authorization.issued regulation_authorization.review_denied fpo_reservation.created fpo_reservation.released fpo_reservation.moved oci.closed apac_export.generated oci_queue_export.generated]`.
4. **`config/regulation/procedures.yml`** ganha `apac_instruments: ["06"]`, `aih_instruments: ["03"]` (Q6), `oci_group: "09"`; listas novas em `config/regulation/`: `oci_consultation_codes.yml` (`0301010072`, `0301010307`, conferidos na Task 0, Q7), `oci_oncology.yml` (os principais 09 de oncologia, Task 0, Q7), `oci_exit_reasons.yml` (Task 0: `1.1 1.2 1.4 1.5 4.1 4.2 4.3`, com os nomes do documento oficial), `sih_clinics.yml` (Task 0: códigos de 2 dígitos da clínica do laudo de AIH), `faec_programs.yml` (Task 0: código e nome dos programas FAEC, Q9) e `national_holidays.yml` (feriados nacionais fixos e móveis por ano, Q6).
5. **Alvo por laudo** (`Regulation::LaudoTargets.accepts?(kind:, code:, release:) -> Boolean`, Q6): `apac` → grupo em `groups`, algum instrumento em `apac_instruments`, nenhum em `bpa_instruments`; `aih` → grupo em `groups`, algum em `aih_instruments`, nenhum em `bpa_instruments`; `oci` → `code[0, 2] == oci_group` e algum em `apac_instruments`. `bpa` → a regra de hoje. Tipo por laudo exige `target_type == "procedure"`.
6. **Número** (`Regulation::AuthorizationNumber.next!(kind:, fifth_digit:, today:) -> String`, Q8): trava `FOR UPDATE` a faixa ativa de `(kind, fifth_digit, year: today.year)` com `next_number <= last_number` e menor `first_number`; monta `uf2 + yy + fifth_digit + format("%07d", next_number)` (12 dígitos; `uf2 = CityProfile.current.ibge_code[0, 2]`, `yy = format("%02d", today.year % 100)`), acrescenta `Regulation::CheckDigit.for(twelve) -> String` (algoritmo do documento da Task 0, com os vetores reais dela no spec) e grava `next_number + 1`. Sem faixa utilizável → `raise Regulation::AuthorizationNumber::Exhausted` (o comando devolve 409 `number_range_exhausted`).
7. **Quinto dígito** (`Regulation::FifthDigit.for(kind:, financing:, attributes:) -> String`, Q8): `oci` → `"7"`; `apac`: `credits` → `"9"`, atributo `052` → `"6"`, senão `"2"`; `aih`: `credits` → `"8"`, atributo `051` → `"5"`, senão `"1"`.
8. **Linha da FPO** (`Regulation::FpoPick.call(competence:, provider:, code:, financing:, faec_program:) -> FpoLine | nil`): a de nível mais específico em `FPO_LEVELS` cujo `code` é prefixo do procedimento (`procedure` = igual), mesma competência, prestador, financiamento e programa. **Valor** (Q9): `(amount_sa_cents + amount_sh_cents + amount_sp_cents + line.complement_cents_per_unit)` do principal na release da competência × `quantity` do principal; nulo vira 0. Pedido BPA por especialidade usa o SIGTAP de `RegulationRndsSetting.current.specialty_sigtap[target_code] || Regulation::RndsLimits::DEFAULT_SIGTAP` (Q9). **Saldo** = `amount_cents − Σ cents` das reservas da linha com `released_at IS NULL AND moved_to_reservation_id IS NULL`.
9. **Validade** (`Regulation::Validity.for(kind:, competence:) -> [from, until]`): `from` = competência da oferta (`AAAAMM`); APAC e OCI: `until` = `from + VALIDITY_EXTRA_MONTHS[kind]` meses; AIH: `until` nulo até o principal ser realizado, e então a competência do realizado (a internação, tomada como a da alta) + `AIH_AFTER_DISCHARGE_MONTHS` (Q9; `Validity.close_aih!(authorization:, performed_on:)`).
10. **Erros novos** (Q10): 403 `authorizer_not_allowed`; 409 `ceiling_exhausted`, `number_range_exhausted`, `number_range_overlap`, `duplicate_fpo_line`, `fpo_line_changed`, `fpo_import_changed`, `apac_settings_changed`; 422 `incompatible_secondary` (`path` `procedures[i]`), `oci_needs_two_secondaries`, `oci_needs_consultation`, `oci_principal_not_group_09` (`path` `procedures`, Q7), `not_own_provider` (`path` `health_unit_id`, Q10), `expected_version_required`, `invalid_request` com `path`. `CareRegulation::BaseController::FAILURE_STATUS` ganha os 409, o 403 e `mfa_required` → 401 (override do teto sem step-up).
11. **Rotas** (Q10; todas no `scope "/regulation"`, com `require_care_regulation!` e `require_care_regulation_apac!`, salvo onde dito):
    - `GET /regulation/targets?type=procedure&kind=apac|oci|aih&query=` (o `kind` ausente = `bpa`);
    - `GET|PUT /regulation/admin/apac_settings` (GET `municipal_admin` ou `care_regulator`, Q13; PUT só `municipal_admin`, com step-up e `expected_version`);
    - `GET|POST /regulation/admin/number_ranges`, `PATCH /regulation/admin/number_ranges/:id` (só `active`) (`municipal_admin`, step-up no POST e no PATCH);
    - `GET|POST /regulation/admin/fpo_lines` (`?competence=&care_provider_id=&before=&limit=`; GET também `care_regulator`, Q13), `PATCH /regulation/admin/fpo_lines/:id` (`quantity`, `amount_cents`, `complement_cents_per_unit`, `expected_version`); `GET /regulation/admin/fpo_imports/template.csv`, `POST /regulation/admin/fpo_imports` (`{ file_base64 }`, até 5 MB), `GET /regulation/admin/fpo_imports/:id`, `POST …/:id/apply` (step-up) e `POST …/:id/discard`, com `expected_version` (`municipal_admin`);
    - `GET /regulation/requests/:id/authorization_preview?offer_id=&financing=&faec_program=` (Q13; `care_regulator`) → `{ financing, financing_default, fpo_line: { id, level, code, amount_cents, complement_cents_per_unit } | null, balance_cents, reserved_cents, ceiling_applies, ceiling_ok, numbered, fifth_digit }`, sem gravar nada;
    - `POST /regulation/requests/:id/authorize` ganha `financing`, `faec_program`, `budget_source`, `ceiling_override_reason`;
    - `GET /regulation/authorizations?stage=awaiting_review&before=&limit=` e `POST /regulation/authorizations/:id/review` (`{ decision, reason?, version }`, `care_regulator`);
    - `GET /regulation/oci?before=&limit=`, `POST /regulation/requests/:id/procedures/:pid/performed` (`{ performed_at, version }`), `POST /regulation/requests/:id/oci_close` (`{ outcome, exit_reason?, version }`) (`care_regulator`);
    - `GET /regulation/requests/:id/laudo.pdf` (`format: false`; `care_regulator` ou o `requester_user`);
    - `GET|POST /regulation/apac_exports`, `GET /regulation/apac_exports/:id/file` (`municipal_admin` ou `billing_clerk`, Q10); `GET|POST /regulation/oci_queue_exports`, `GET /regulation/oci_queue_exports/:id/file` (`municipal_admin`).
12. **Dev:** worktree `apps/api/.claude/mod26b` (com `config/master.key`), branch `feat/mod-26b-apac`, api do worktree na porta **3064**; migrações `db/platform_migrate/20261127600001_add_sigtap_billing_data.rb` e `db/city_migrate/20261127900001_add_regulation_apac.rb` (números a conferir na Task 0).

## Global Constraints

- git `/opt/homebrew/bin/git`; nunca `git add -A` nem `git stash` sem nome. Commits em inglês, Conventional Commits com o tipo por extenso: `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge, push e cards só com autorização do usuário.
- **Testes só por `bin/test api -w mod26b <specs>`**, a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`; nunca `rspec` direto nem o Postgres do host; `--keep-db` só em rodada de arquivo isolado. A branch é transversal (migração de cidade e de plataforma, SIGTAP, G11, `Authorize`, `Billing::*`, signer, 26c, `CityEncryption`, `Platform::Features`, `spec/support`, initializers) → **suíte completa obrigatória, pedida à sessão "Testes" na Task 18**.
- Migração: dump à mão em `db/city_schema.rb` e `db/platform_schema.rb`; triggers num bloco "Módulo 26b" ao fim de `db/city_triggers.sql`, dentro de `IF to_regclass('public.<tabela>') IS NOT NULL`; **irreversíveis** (`down` → `ActiveRecord::IrreversibleMigration`; o `up` da cidade termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`); CHECK trocado por `replace_check`. Banco de dev só migrado e semeado na prova, com autorização do usuário.
- Specs com tipo explícito (`type: :request`, `:model`, `:job`). Nenhuma spec bate em URL real (DATASUS, FTP, SIA).
- **Nunca `Current.city =` solto.** Interruptores lidos fora da transação (`Finalize`, `AddAddendum`, `Authorize` pelo controller); consumidores de evento só `enabled?`.
- **Sigilo (contrato §11):** laudo (`clinical_summary`, `cid_principal`, `cid_secondary`, `findings`, `character`, `clinic`), CID, justificativa, `ceiling_override_reason`, motivos e arquivos nunca em log, URL, evento, lista, `Result` de recusa nem `inspect`. Modelos com esses campos sobrescrevem `inspect`/`to_s`/`pretty_print`. `filter_parameter_logging` ganha `clinical_summary`, `findings`, `cid_principal`, `cid_secondary`, `ceiling_override_reason`, `file_base64`.
- Eventos só com ids (e `user_id`), em `config/initializers/domain_events.rb` e `spec/initializers/domain_events_bindings_spec.rb`; `DomainEvents.publish(` é multilinha. Nenhum `Platform.audit` novo.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` e em `spec/jobs/reencryption_job_spec.rb`.
- Concorrência otimista: `expected_version` (ausente → 422 `expected_version_required`; diferente → 409 `<entidade>_changed`); decisões sobre o pedido seguem o `version` do pedido, como no 26a.
- **O 26a sem o interruptor não muda:** `spec/requests/care_regulation spec/services/care_regulation spec/commands/care_regulation spec/services/billing spec/commands/billing` ficam verdes em toda task que toca `CareRegulation::*` ou `Billing::*`.
- Cada task termina com o **Relato ao coordenador**: uma linha à sessão "Próximos módulos" por `SendMessage`.

## Review Focus

1. **Duas autorizações na última vaga de dinheiro da mesma linha da FPO, por ofertas diferentes** (linha de grupo cobre dois procedimentos): uma passa, a outra recebe 409 `ceiling_exhausted`; nunca as duas reservas. Teste na Task 8 (`authorize_apac_concurrency_spec.rb`, duas threads com `pop(timeout:)`).
2. **Número na virada de faixa e de ano:** a última posição de uma faixa sai, a próxima autorização usa a faixa seguinte do mesmo tipo e dígito, e em 1º de janeiro o ano do número e da faixa é o novo; duas threads nunca tiram o mesmo número. Teste na Task 6.
3. **Cancelar, faltar ou reautorizar com reserva viva:** cancelar devolve; a 1ª falta mantém a reserva (Q9); reautorizar depois da falta transfere a reserva, sem reservar de novo, e tira número novo, e o antigo nunca volta; a 2ª falta (`closed`) devolve. Teste na Task 8.
4. **Secundário de OCI ativa registrado no prontuário que também está no BPA** (consulta `0301010072`): fica fora do BPA-I enquanto a OCI está aberta, com `authorization_number` preenchido; com a OCI encerrada `not_concluded`, entra no fechamento seguinte. Teste na Task 10.
5. **Interruptor desligado depois de haver pedidos por laudo:** o rascunho por laudo não promove (`not_sent` com `invalid_target`), as rotas novas respondem 403 `feature_disabled` com `care_regulation_apac`, e a fila, o cancelamento e o desfecho dos pedidos já autorizados seguem funcionando (a devolução da reserva não depende do interruptor). Teste nas Tasks 5 e 8.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod26b/`)
- Migrações: `db/platform_migrate/20261127600001_add_sigtap_billing_data.rb`, `db/city_migrate/20261127900001_add_regulation_apac.rb`.
- Modelos: `app/models/{sigtap_financing,sigtap_procedure_compatibility,sigtap_procedure_habilitation,sigtap_attribute,regulation_laudo,regulation_requested_procedure,authorization_number_range,fpo_line,fpo_reservation,fpo_import,regulation_apac_setting,apac_export,oci_queue_export}.rb`.
- `app/services/terminology/sigtap_billing.rb`; `app/services/regulation/{apac_gate,apac_catalog,laudo_targets,laudo_rules,fifth_digit,check_digit,authorization_number,authorization_preview,fpo_pick,validity,reservations,authorizer,performed_link,oci_block,oci_list,apac_file,oci_queue_file,business_days,laudo_pdf,apac_json,fpo_csv}.rb`.
- `app/commands/regulation/{save_laudo,save_apac_settings,save_number_range,save_fpo_line,upload_fpo_import,apply_fpo_import,discard_fpo_import,review_authorization,mark_performed,close_oci,generate_apac_export,generate_oci_queue_export}.rb`.
- `app/jobs/regulation/oci_queue_export_job.rb`.
- `app/controllers/concerns/regulation_apac_gate.rb`; `app/controllers/care_regulation/{authorizations,oci,laudos,apac_exports,oci_queue_exports}_controller.rb`; `app/controllers/care_regulation/admin/{apac_settings,number_ranges,fpo_lines,fpo_imports}_controller.rb`.
- `config/regulation/{oci_consultation_codes,oci_oncology,oci_exit_reasons,sih_clinics,faec_programs,national_holidays}.yml`; `config/clinical/regulation-laudo-v1.json` (cópia da tag `clinical-v1.3.0`); `vendor/regulation/<data>/`; `lib/regulation_apac_crew.rb`; `spec/support/regulation_apac_helpers.rb`; `spec/security/regulation_apac_secrecy_spec.rb`; specs listados em cada task.

**Modificar**
- `app/services/terminology/sigtap_reader.rb`, `lib/sigtap_sample.rb`, `app/services/platform/features.rb`, `app/services/city_encryption.rb`, `db/{city_schema,platform_schema}.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `config/regulation/procedures.yml`, `db/seeds.rb`.
- `app/services/care_regulation/{targets,json}.rb`, `app/commands/care_regulation/{save_draft,promote,complement,authorize,decide}.rb`, `app/services/care_regulation/{misses,appointment_sync}.rb`, `app/controllers/care_regulation/{base,decisions,targets,consultation_requests}_controller.rb`, `app/models/{regulation_request,regulation_authorization,performed_procedure,signature_request}.rb`, `app/commands/consultations/{finalize,add_addendum}.rb`.
- `app/services/billing/{critic,recalculate,lines,bpa_file}.rb`, `app/commands/billing/record.rb`.
- `app/services/signatures/{document_types,canonical,documents,signing,json}.rb`.
- 26c: `app/services/regulation/{rira_facts,rira_bundle,rnds_validation,rnds_limits}.rb`.
- Specs existentes: `spec/services/terminology/sigtap_reader_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`, `spec/services/care_regulation/{targets,targets_yaml}_spec.rb`, `spec/requests/care_regulation/{consultation_requests,decisions,targets}_spec.rb`, `spec/commands/care_regulation/{authorize,decide,misses,promote}_spec.rb`, `spec/services/billing/{critic,bpa_file}_spec.rb`, `spec/services/signatures/canonical_vector_spec.rb`, `spec/services/regulation/{rira_bundle,rnds_validation}_spec.rb`, `spec/services/city_schema_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/jobs/reencryption_job_spec.rb`.

---

### Task 0: Worktree, pré-requisitos, insumos oficiais e cards

**Files:** `vendor/regulation/<AAAA-MM-DD>/` (só com os downloads autorizados), `config/regulation/{oci_consultation_codes,oci_oncology,oci_exit_reasons,sih_clinics,faec_programs,national_holidays}.yml`, `config/regulation/procedures.yml` (só valores conferidos).

- [ ] **Step 1:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod26b -b feat/mod-26b-apac origin/main && cp config/master.key .claude/mod26b/config/master.key`.
- [ ] **Step 2: Pré-requisitos.** No worktree: `ls db/city_migrate | tail -3` e `ls db/platform_migrate | tail -3`, o `define(version:` dos dois dumps (esperado: cidade `20261126900001` ou posterior; plataforma `20261125600001` ou posterior); a última `Entry` do `CATALOG` é `rnds_regulation`; existem `app/services/regulation/{rnds_limits,rira_facts,rira_bundle,rnds_validation}.rb` e `RegulationRndsSetting`; não existem `care_regulation_apac`, `regulation_laudos`, `fpo_lines` nem `authorization_number_ranges`. **26c ou 27b ausente da main → parar** e avisar o coordenador. Migração mais nova que os números do Valor 12 → o número sobe para o dia seguinte ao dela (nota no topo antes da Task 1).
- [ ] **Step 3: Insumos oficiais (cada download só com a autorização do usuário no chat, dizendo nome do arquivo, origem e tamanho).**
  1. **Layout do arquivo de APAC do SIA**, com a versão: o documento de exportação listado em `sia.datasus.gov.br/documentos/listar_ftp_apac.php` (a pesquisa cita `layout_Exportacao_APAC.pdf`). Transcrever a tabela de campos (registro, campo, início, tamanho, tipo, obrigatoriedade) em `vendor/regulation/<data>/apac_layout_<versão>.csv`, com o PDF ao lado.
  2. **Algoritmo do dígito verificador** do número de APAC e de AIH: documento oficial (manual do SIA/SIH, do Módulo Autorizador ou o próprio layout) e **pelo menos cinco números reais de 13 dígitos** (faixa da SES, documento publicado) para virar vetor de teste em `vendor/regulation/<data>/check_digit_vectors.txt`.
  3. **ZIP completo da SIGTAP** da competência corrente em `ftp://ftp2.datasus.gov.br/pub/sistemas/tup/downloads/`: conferir que traz `tb_financiamento`, `rl_procedimento_compativel`, `rl_procedimento_habilitacao`, `tb_detalhe`, `rl_procedimento_detalhe` e, em `tb_procedimento`, `VL_SH`, `VL_SP` e `CO_FINANCIAMENTO`, cada um com o `_layout.txt`. Guardar no vendor o **ZIP completo** e o `SOURCE` (URL, competência, SHA-256 do ZIP) (Q6).
  **Qualquer um dos três ausente → parar** e avisar o coordenador com o que faltou.
- [ ] **Step 4: Valores que o contrato manda conferir aqui (cada um falta só para a task dona, Q6):** `aih_instruments` = `03` em `tb_registro`; `oci_consultation_codes.yml` (`0301010072`, `0301010307`) e `oci_oncology.yml` (os principais 09 de oncologia) pela SIGTAP e pelas portarias da OCI (Tasks 5 e 11, Q7); `faec_programs.yml` (Task 8, Q9); `national_holidays.yml` com os feriados nacionais fixos e os móveis (Carnaval, Sexta-feira Santa, Corpus Christi) dos anos de uso (Task 14, Q6); `oci_exit_reasons.yml` com os nomes do layout/manual (Portaria SAES 1.640/2024 arts. 12–15: `1.1 1.2 1.4 1.5 4.1 4.2 4.3`); `sih_clinics.yml` do manual do SIH; o **layout do e-SUS Captação de Filas** (só a Task 14 depende dele; ausente → a Task 14 fica parada). Recontar em `SOURCE` os reguláveis por laudo por grupo com a regra do Valor 5.
- [ ] **Step 5:** `bin/test api -w mod26b spec/services/terminology spec/services/platform/features_spec.rb spec/requests/care_regulation spec/services/care_regulation spec/commands/care_regulation spec/services/billing spec/commands/billing spec/services/signatures spec/services/regulation spec/initializers/domain_events_bindings_spec.rb` → PASS (linha de base).
- [ ] **Step 6: Commit** (só os arquivos de vendor e config conferidos) `git add vendor/regulation/<data> config/regulation && git commit -m "chore: vendor official apac layout, check digit and full sigtap sources" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.
- [ ] **Step 7 (só com autorização do usuário):** cards do 26b no board #1 pela receita da memória `board-91-features-roadmap` (Module `mod-26`, Work type `feature`), um `gh project item-create …` por chamada, cada comando começando com `gh`: "SIGTAP completa e interruptor"; "Pedido por laudo (APAC, OCI, AIH)"; "Faixas de numeração e dígito verificador"; "Espelho da FPO e reserva de teto"; "OCI e faturamento"; "Remessa APAC e fila nominal da OCI"; "Laudo em PDF e assinatura". Sem autorização: anotar no relato.

**Relato ao coordenador:** "api 26b · Task 0 · worktree mod26b em origin/main <hash> (27b e 26c presentes); migrações <números>; layout APAC <versão>, DV <fonte, N vetores>, SIGTAP <competência> no vendor; e-SUS Captação <ok | ausente>; cards <criados | aguardando>; linha de base verde."

---

### Task 1: SIGTAP completa na plataforma

**Files:**
- Create: `db/platform_migrate/20261127600001_add_sigtap_billing_data.rb`, `app/models/{sigtap_financing,sigtap_procedure_compatibility,sigtap_procedure_habilitation,sigtap_attribute}.rb`, `app/services/terminology/sigtap_billing.rb`, `spec/services/terminology/sigtap_billing_reader_spec.rb`
- Modify: `db/platform_schema.rb`, `app/services/terminology/sigtap_reader.rb`, `lib/sigtap_sample.rb`, `spec/services/terminology/sigtap_reader_spec.rb`

**Interfaces:**
- `sigtap_procedures` + `amount_sh_cents integer`, `amount_sp_cents integer`, `financing_code varchar(2)`. Tabelas (todas com `release_id uuid NOT NULL` e índice único com ele): `sigtap_financings` (`code varchar(2)`, `name text`); `sigtap_procedure_compatibilities` (`principal_code`, `principal_instrument_code`, `secondary_code`, `secondary_instrument_code`, `compatibility_type varchar(1)`, `max_quantity integer`; os dois registros do `rl_procedimento_compativel`); `sigtap_procedure_habilitations` (`procedure_code`, `habilitation_code varchar(4)`, `habilitation_group varchar(4)` nulo); `sigtap_attribute_catalog` (`code varchar(3)`, `name text`; modelo `SigtapAttribute`).
- `SigtapReader#write` acrescenta as contagens `sigtap_financings`, `sigtap_procedure_compatibilities`, `sigtap_procedure_habilitations`, `sigtap_attributes`; arquivo ausente = 0 linhas (todos opcionais, contrato §2); colunas novas de `tb_procedimento` ausentes no layout = nulo (o `r["VL_SH"]` de layout sem a coluna é nil).
- Produces: Valor 2; `SigtapProcedure#amount_total_cents -> Integer` (SA + SH + SP, nulos como 0).

- [ ] **Step 1: Write the failing test** (`type: :model`): "o recorte com `tb_financiamento` grava `06 MEDIA E ALTA COMPLEXIDADE (MAC)` e `04 FUNDO DE ACOES ESTRATEGICAS E COMPENSACOES (FAEC)` na release"; "o procedimento ganha `amount_sh_cents`, `amount_sp_cents` e `financing_code`"; "`rl_procedimento_compativel` grava principal 0905… × secundário 0301010072 com tipo e quantidade"; "`rl_procedimento_habilitacao` grava 3801 para o principal da OCI"; "`tb_detalhe` grava `052`"; "compatibilidade que aponta para procedimento ausente → `Import::Invalid`"; "recorte antigo (sem os arquivos novos) importa como hoje, com zero linhas novas"; "`SigtapBilling.loaded?` falso sem financiamento e verdadeiro com"; "`amount_total_cents` soma os três e trata nulo como 0". O `SigtapSample` ganha os layouts e as linhas (códigos reais do vendor da Task 0) e um `write_to(dir, competence:, billing: true)`.
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/terminology/sigtap_billing_reader_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`AddSigtapBillingData`, dump `define(version: 2026_11_27_600001)`), os modelos (`PlatformRecord`), o leitor (nomes de coluna dos `_layout.txt` da Task 0) e o `SigtapBilling`.
- [ ] **Step 4:** o mesmo spec + `spec/services/terminology` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: import sigtap financing, values, compatibilities and habilitations" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 1 · 20261127600001: financiamento, SH/SP, compatibilidades, habilitações e atributos na SIGTAP; sigtap_billing_data = release vigente com financiamento."

---

### Task 2: Interruptor `care_regulation_apac`

**Files:**
- Create: `app/services/regulation/{apac_gate,apac_catalog}.rb`, `app/controllers/concerns/regulation_apac_gate.rb`, `spec/services/regulation/apac_gate_spec.rb`
- Modify: `app/services/platform/features.rb`, `spec/services/platform/features_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`

**Interfaces:**
- Produces: Valores 1 e 3; `RegulationApacGate#require_care_regulation_apac!` → 403 `{ error: "feature_disabled", feature: "care_regulation_apac" }` sem `usable?`.

- [ ] **Step 1: Write the failing tests.** `features_spec.rb`: índices contados do fim sobem um; "`care_regulation_apac` fecha o catálogo com os `requires` do Valor 1"; "sem release com financiamento → `[\"sigtap_billing_data_missing\"]`"; "`care_regulation` desligado → `care_regulation_disabled`"; "release antiga com financiamento e a da competência corrente sem → missing" (`release_for` pega a mais nova ≤ competência). `city_features_spec.rb`: listas exatas com a chave. `apac_gate_spec.rb`: "`usable?` verdadeiro com SIGTAP de faturamento e `care_regulation` utilizável"; "`enabled?` só lê o interruptor".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/platform/features_spec.rb spec/services/regulation/apac_gate_spec.rb spec/requests/maintenance/city_features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a `Entry` (comentário: "26b: decisões 1, 6, 14; desligado, o pedido por laudo é recusado e o 26a segue como está"), o requisito, o gate, o catálogo e o concern.
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add the care_regulation_apac switch" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 2 · care_regulation_apac no fim do CATALOG, requires feature:care_regulation + sigtap_billing_data (sigtap_billing_data_missing)."

---

### Task 3: Migração de cidade — laudo, procedimentos, faixas, autorização, FPO e remessas

**Files:**
- Create: `db/city_migrate/20261127900001_add_regulation_apac.rb`, os modelos de cidade do mapa, `spec/models/regulation_apac_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{regulation_request,regulation_authorization,performed_procedure,signature_request}.rb`, `app/services/city_encryption.rb`, `spec/services/city_schema_spec.rb`, `spec/jobs/reencryption_job_spec.rb`

**Interfaces:**
- `regulation_requests.authorization_kind varchar NOT NULL DEFAULT 'bpa'` (CHECK `KINDS`; CHECK `authorization_kind = 'bpa' OR target_type = 'procedure'`) e, para o encerramento da OCI (Q7), `oci_outcome` (`OCI_OUTCOMES` ou nulo), `oci_exit_reason varchar(3)`, `oci_closed_at` (CHECK: os três juntos, só com `authorization_kind = 'oci'`).
- `regulation_laudos`: `regulation_request_id` (FK, **único**), `clinical_summary text NOT NULL`, `cid_principal text NOT NULL`, `cid_secondary text`, `findings text`, `character text`, `clinic text` (todos cifrados), `version integer NOT NULL DEFAULT 1`, timestamps. `RegulationLaudo#author_user_id` = `regulation_request.requester_user_id`; `#consultation_id` = o do pedido; `inspect` só id e versão.
- `regulation_requested_procedures`: `regulation_request_id` (FK), `sigtap_code varchar(10)`, `role`, `quantity integer`, `performed_at date`, `performed_source`, `performed_procedure_id` (FK, único quando não nulo), `performed_by_user_id` (FK, nulo), timestamps. CHECKs: `role`/`performed_source` nos catálogos; `quantity` em `PROCEDURE_QUANTITY`; `(performed_at IS NULL) = (performed_source IS NULL)`; `(performed_source = 'performed_procedure') = (performed_procedure_id IS NOT NULL)`. Únicos `(regulation_request_id, sigtap_code)` e `(regulation_request_id) WHERE role = 'principal'`.
- `authorization_number_ranges`: `kind`, `fifth_digit varchar(1)`, `year integer`, `first_number`, `last_number`, `next_number` (inteiros; sequencial de 7 dígitos, Q8), `active boolean NOT NULL DEFAULT true`, `created_by_user_id` (FK), timestamps. CHECKs: par `kind`×`fifth_digit` de `FIFTH_DIGITS`; `1 <= first_number <= last_number <= 9999999`; `first_number <= next_number <= last_number + 1`; `year BETWEEN 2000 AND 2099`. `EXCLUDE USING gist (kind WITH =, fifth_digit WITH =, year WITH =, int4range(first_number, last_number, '[]') WITH &&)` (`number_range_overlap`). Trigger: sem DELETE; `UPDATE` só aumenta `next_number` e troca `active`.
- `regulation_authorizations` + `number varchar(13)` (único parcial `WHERE number IS NOT NULL`, CHECK `^[0-9]{13}$`), `authorization_kind varchar NOT NULL DEFAULT 'bpa'`, `stage`, `valid_from_competence varchar(6)`, `valid_until_competence varchar(6)`, `financing`, `faec_program varchar(6)`, `budget_source varchar(2)` (Q9), `fpo_line_id` (FK), `reserved_cents bigint`, `authorizer_professional_id` (FK `professionals`), `ceiling_override_reason text` (cifrado). CHECKs: `stage` em `STAGES` ou nulo; `authorization_kind = 'bpa'` → `stage`, `number` nulos; `number IS NOT NULL` → `stage = 'issued'`; `financing` em `FINANCINGS` ou nulo; `financing = 'credits'` ⇔ `budget_source IS NOT NULL`; `(fpo_line_id IS NULL) = (reserved_cents IS NULL)`. Trigger: `number` e `stage` só mudam de nulo para valor ou de `awaiting_review` para `issued`/`review_denied`.
- `fpo_lines`: `competence varchar(6)`, `care_provider_id` (FK), `level`, `code varchar(10)`, `financing`, `faec_program varchar(6)`, `quantity integer`, `amount_cents bigint`, `complement_cents_per_unit bigint NOT NULL DEFAULT 0` (Q9), `version integer NOT NULL DEFAULT 1`, `created_by_user_id`, `updated_by_user_id`, timestamps. Único `(competence, care_provider_id, level, code, financing, COALESCE(faec_program, ''))`. CHECKs: `length(code) = <tamanho do nível>` (Valor 3) e só dígitos; `financing` em `CEILING_FINANCINGS` (a linha `other` não existe, Q9); `financing = 'faec'` → `level = 'procedure'` (Q9); `quantity >= 0`, `amount_cents >= 0`, `complement_cents_per_unit >= 0`. Trigger: sem DELETE.
- `fpo_reservations`: `fpo_line_id` (FK), `regulation_authorization_id` (FK), `competence varchar(6)`, `cents bigint NOT NULL`, `quantity integer NOT NULL`, `released_at`, `moved_to_reservation_id` (FK à própria), `created_at`. Índice `(fpo_line_id) WHERE released_at IS NULL AND moved_to_reservation_id IS NULL`. Trigger: sem DELETE; só `released_at` ou `moved_to_reservation_id`, uma vez.
- `regulation_apac_settings` (linha única, `RegulationApacSetting.current` cria com o padrão): `authorization_moment varchar NOT NULL DEFAULT 'before'` (CHECK `MOMENTS`), `version integer NOT NULL DEFAULT 1`, `updated_by_user_id`, `updated_at`.
- `fpo_imports` (Q10): `status` (`validated | applied | discarded`), `file_sha256`, `rows jsonb` (linhas lidas, sem dado pessoal), `report jsonb`, `uploaded_by_user_id`, `applied_by_user_id`, `applied_at`, `version`, timestamps. Trigger: `validated → applied | discarded`.
- `apac_exports`: `competence varchar(6)`, `health_unit_id` (FK), `file text` (cifrado), `sha256 varchar(64)`, `layout_version varchar(20)`, `apacs integer`, `version integer`, `generated_by_user_id`, `generated_at`; único `(health_unit_id, competence, version)`. `oci_queue_exports`: `competence`, `file` (cifrado), `sha256`, `rows integer`, `version`, `generated_at`, `generated_by_user_id` (nulo no job); único `(competence, version)`. Trigger: sem DELETE nem UPDATE nas duas.
- `performed_procedures.regulation_authorization_id` (FK, nulo, índice).
- `replace_check` em `ck_signature_requests_document_type` (+ `RegulationLaudo`) e `ck_signature_requests_consultation` (`RegulationLaudo` com `consultation_id NOT NULL`).
- `CITY_KEYED_TARGETS` + os seis campos do laudo, `[ RegulationAuthorization, :ceiling_override_reason ]`, `[ ApacExport, :file ]`, `[ OciQueueExport, :file ]`.

- [ ] **Step 1: Write the failing test** (`type: :model`): "pedido `apac` com alvo `specialty` → CHECK"; "um laudo por pedido"; "laudo cifrado com a chave da cidade e `inspect` sem texto"; "dois principais no mesmo pedido → índice"; "`performed_source` sem `performed_at` → CHECK"; "faixa APAC com dígito 5 → CHECK"; "faixas 1–100 e 50–150 do mesmo tipo, dígito e ano → exclusão; 1–100 e 101–200 passam; ano diferente passa"; "`next_number` voltando → trigger"; "dois números iguais → índice único"; "autorização `bpa` com `stage` → CHECK"; "`credits` sem `budget_source` → CHECK"; "linha FAEC por grupo → CHECK"; "linha `other` → CHECK"; "`oci_outcome` em pedido `apac` → CHECK"; "linha duplicada com programa nulo → índice"; "reserva liberada duas vezes → trigger"; "DELETE de faixa, linha, reserva, remessa → trigger"; "assinatura de `RegulationLaudo` sem consulta → CHECK"; "`RegulationApacSetting.current` nasce `before`".
- [ ] **Step 2:** `bin/test api -w mod26b spec/models/regulation_apac_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`AddRegulationApac`), o dump (`define(version: 2026_11_27_900001)`), os triggers, os modelos e o `CityEncryption`.
- [ ] **Step 4:** o mesmo spec + `spec/services/city_schema_spec.rb spec/jobs/reencryption_job_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/requests/care_regulation spec/services/billing` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: add regulation laudo, number ranges, fpo mirror and apac exports" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 3 · 20261127900001: laudo cifrado, procedimentos, faixas com EXCLUDE, colunas da autorização, FPO, reservas, configurações, fpo_imports, remessas e RegulationLaudo assinável."

---

### Task 4: Alvo por laudo (G11, segundo caminho)

**Files:**
- Create: `app/services/regulation/laudo_targets.rb`, `spec/services/regulation/laudo_targets_spec.rb`
- Modify: `config/regulation/procedures.yml`, `app/services/care_regulation/targets.rb`, `app/controllers/care_regulation/targets_controller.rb`, `spec/services/care_regulation/{targets,targets_yaml}_spec.rb`, `spec/requests/care_regulation/targets_spec.rb`, `spec/support/regulation_apac_helpers.rb`

**Interfaces:**
- Produces: Valor 5; `CareRegulation::Targets.resolve(type:, code:, on: Time.zone.today, kind: "bpa")` (Q6) — `kind` fora de `KINDS` ou laudo com `type` `specialty` → `invalid_target`; `search(type:, query:, on:, limit:, kind: "bpa")` (índice por `[release.id, kind]`; cada item da busca traz `authorization_kind`, o `kind` pedido, Q6); `RegulationApacHelpers#apac_city!` (SIGTAP de faturamento com os códigos do vendor: um só-APAC, um só-AIH, um AIH + APAC sem BPA, uma OCI 09 com dois secundários compatíveis, a consulta e a teleconsulta) e `#apac_sigtap!`.
- O controller lê `kind` da query (`query_only!(:type, :query, :kind)`); `kind` por laudo exige `Regulation::ApacGate.usable?` → senão `[]`.

- [ ] **Step 1: Write the failing tests.** `laudo_targets_spec.rb` (`type: :model`): "`0305010107` (só 06) aceita `apac` e recusa `aih` e `bpa`"; "`0407030026` (só 03) aceita `aih`"; "`0409050083` (03 06) aceita `apac` e `aih`"; "`0209010037` (02 04 06 07, tem BPA) recusa `apac` e segue `bpa`"; "a OCI 09… aceita só `oci`"; "procedimento só com 04 (AIH especial) recusa `aih`" (Q6); "grupo 05 recusa todos". `targets_yaml_spec.rb`: chaves novas e o exemplo de cada caminho. `targets_spec.rb` (request): "`kind=oci` com o interruptor ligado lista a OCI com `authorization_kind: \"oci\"`; desligado → `[]`"; "`kind=xyz` → 422 path `kind`".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/laudo_targets_spec.rb spec/services/care_regulation/targets_spec.rb spec/services/care_regulation/targets_yaml_spec.rb spec/requests/care_regulation/targets_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o `rules` lê as chaves novas; o 26a sem `kind` não muda).
- [ ] **Step 4:** os mesmos specs + `spec/requests/care_regulation spec/services/care_regulation` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: resolve apac, oci and aih targets through the second g11 path" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 4 · Targets.resolve/search com kind; só-APAC, só-AIH (03) e OCI 09 por laudo; a busca devolve o authorization_kind; o 26a sem kind não muda (Q6)."

---

### Task 5: Pedido por laudo na consulta

**Files:**
- Create: `app/services/regulation/laudo_rules.rb`, `app/commands/regulation/save_laudo.rb`, `spec/services/regulation/laudo_rules_spec.rb`
- Modify: `app/commands/care_regulation/{save_draft,promote,complement}.rb`, `app/commands/consultations/{finalize,add_addendum}.rb`, `app/services/care_regulation/json.rb`, `app/controllers/care_regulation/consultation_requests_controller.rb`, `spec/requests/care_regulation/consultation_requests_spec.rb`, `spec/commands/care_regulation/promote_spec.rb`

**Interfaces:**
- `SaveDraft::KEYS` + `authorization_kind`, `laudo`, `procedures`; `SaveDraft.call(consultation:, attrs:, by:, request: nil, apac: false)` (`apac` = `ApacGate.usable?` lido pelo controller fora da transação). Sem `apac` e tipo por laudo → `invalid_target` (path `authorization_kind`).
- `laudo` = `{ clinical_summary, cid_secondary, findings, character?, clinic? }` (sem `cid_principal`: é o `cid10` do pedido, obrigatório nos tipos por laudo, Q6); `procedures` = `[{ sigtap_code, role, quantity }]`.
- `Regulation::LaudoRules.check(kind:, target_code:, cid10:, laudo:, procedures:, release:) -> Result.ok(laudo:, procedures:) | fail(:invalid_request | :invalid_cid10 | :incompatible_secondary | :oci_principal_not_group_09 | :oci_needs_two_secondaries | :oci_needs_consultation, details: { path: })`, nesta ordem: `cid10` presente (path `cid10`); `clinical_summary` em `CLINICAL_SUMMARY` depois do `strip` (path `laudo.clinical_summary`); `cid_secondary` por `ClinicalTerms.find("cid10", …)` (path `laudo.cid_secondary`); `findings` até `FINDINGS_MAX`; AIH: `character` em `CHARACTERS` e `clinic` em `sih_clinics.yml` (paths `laudo.character`, `laudo.clinic`); fora da AIH as duas chaves são recusadas; `procedures`: 1..`MAX_PROCEDURES`, exatamente um `principal` igual ao `target_code` (path `procedures`), `quantity` em `PROCEDURE_QUANTITY` (path `procedures[i].quantity`), códigos sem repetição; cada secundário com linha em `sigtap_procedure_compatibilities` do principal na release (senão `incompatible_secondary`, path `procedures[i]`); OCI (Q7, path `procedures`): principal fora do grupo 09 → `oci_principal_not_group_09`; menos de `OCI_MIN_SECONDARIES` secundários → `oci_needs_two_secondaries`; nenhum em `oci_consultation_codes.yml` → `oci_needs_consultation`.
- `Regulation::SaveLaudo.call(request:, values:) -> RegulationLaudo` (substitui laudo e procedimentos do rascunho; `version + 1`).
- `Promote.call(…, apac: false, signing: false)`: rascunho por laudo é revalidado com `kind:` e `LaudoRules`; sem `apac` → `not_sent` com `invalid_target`. A abertura da assinatura vem na Task 15.
- `Complement` (Q13): `KEYS` + `laudo` e `procedures`, revalidados pelo `LaudoRules` (`laudo.version + 1`); com o laudo já assinado (assinatura `signed` do `RegulationLaudo`), trocar o `cid10` → 422 `invalid_request` path `cid10` (Q6); a assinatura guarda o canônico do ato, que não muda.
- `Json.request` + `authorization_kind`, `laudo` (só no detalhe e na lista da consulta da autora), `procedures` (`id`, `sigtap_code`, `name`, `role`, `quantity`, `performed_at`, `performed_source`); `Json.list_item` + só `authorization_kind`.

- [ ] **Step 1: Write the failing tests.** `laudo_rules_spec.rb` (`type: :model`): uma linha por recusa da ordem acima, com o `path` exato; "OCI com dois secundários compatíveis e a consulta passa"; "OCI sem a consulta → `oci_needs_consultation`; com um secundário → `oci_needs_two_secondaries`; principal `0305…` → `oci_principal_not_group_09`"; "secundário sem compatibilidade → `incompatible_secondary` path `procedures[1]`"; "`character` numa APAC → path `laudo.character`". `consultation_requests_spec.rb` (`type: :request`): "médico cria rascunho `oci` com laudo e procedimentos → 201 com o laudo"; "com o interruptor desligado → 422 `invalid_target` path `authorization_kind`"; "`apac` com alvo de BPA → `invalid_target`"; "PATCH substitui o laudo e sobe a versão"; "a lista da fila não traz o laudo". `promote_spec.rb`: "finalizar promove o rascunho `aih` a `pending` com o laudo"; "interruptor desligado entre o rascunho e a finalização → `not_sent` `invalid_target`, rascunho intocado" (Review Focus 5); "complemento de pedido `oci` devolvido troca laudo e procedimentos"; "complemento de laudo assinado com outro `cid10` → 422 path `cid10`; sem assinatura, troca".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/laudo_rules_spec.rb spec/requests/care_regulation/consultation_requests_spec.rb spec/commands/care_regulation/promote_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o `Finalize` e o `AddAddendum` leem `Regulation::ApacGate.usable?` junto dos outros interruptores, fora da transação).
- [ ] **Step 4:** os mesmos specs + `spec/requests/care_regulation spec/commands/care_regulation spec/commands/consultations` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: request apac, oci and aih with a laudo from the consultation" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 5 · pedido por laudo no POST/PATCH da consulta; cid_principal = cid10 do pedido (Q6); OCI com recusas oci_* (Q7); complemento com laudo e procedimentos (Q13); laudo fora da fila."

---

### Task 6: Faixas de numeração e número da autorização

**Files:**
- Create: `app/services/regulation/{check_digit,fifth_digit,authorization_number}.rb`, `app/commands/regulation/save_number_range.rb`, `app/controllers/care_regulation/admin/number_ranges_controller.rb`, `spec/services/regulation/{check_digit,authorization_number}_spec.rb`, `spec/requests/care_regulation/number_ranges_spec.rb`
- Modify: `config/routes.rb`, `app/controllers/care_regulation/base_controller.rb`

**Interfaces:**
- Produces: Valores 6 e 7. `Regulation::SaveNumberRange.call(attrs:, by:, range: nil) -> Result.ok(range:) | fail(:invalid_request | :number_range_overlap, details: { path: })`: POST `{ kind, fifth_digit, year, first_number, last_number }` (`next_number = first_number`); PATCH `{ active }`. A exclusão do banco vira `number_range_overlap` (rescue de `ActiveRecord::StatementInvalid` com `PG::ExclusionViolation`). `Regulation::ApacJson.range(r)` → `{ id, kind, fifth_digit, year, first_number, last_number, next_number, first_authorization_number, last_authorization_number, next_authorization_number, remaining, active }` (os três `*_authorization_number` são os 13 dígitos com o DV, Q8).

- [ ] **Step 1: Write the failing tests.** `check_digit_spec.rb` (`type: :model`): "cada número de `vendor/regulation/<data>/check_digit_vectors.txt` tem o DV calculado"; "12 dígitos só". `authorization_number_spec.rb`: "cidade com IBGE `41…`: a faixa APAC 7 de 2026 de 1 a 3 dá `412670000001` + DV, depois `…002`, `…003` e então `Exhausted`"; "a faixa esgotada passa para a seguinte do mesmo tipo e dígito" (Review Focus 2); "faixa inativa não é usada"; "em 2027-01-01 usa a faixa de 2027 e o `27`"; "duas threads na última posição → um número e um `Exhausted`" (`pop(timeout:)`); `FifthDigit` com cada linha do Valor 7. `number_ranges_spec.rb` (`type: :request`): "admin cria com step-up"; "sem step-up → 401 `mfa_required`"; "sobreposição → 409 `number_range_overlap`"; "regulador → 403"; "PATCH só `active`; outro campo → 422 path"; "interruptor desligado → 403 `feature_disabled` `care_regulation_apac`".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/check_digit_spec.rb spec/services/regulation/authorization_number_spec.rb spec/requests/care_regulation/number_ranges_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o algoritmo do DV é o do documento da Task 0, citado no comentário do `CheckDigit` com a fonte e a versão).
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: issue apac and aih numbers from city ranges with check digit" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 6 · faixas sem sobreposição (EXCLUDE), número UF+ano+dígito+sequencial+DV sob FOR UPDATE, number_range_exhausted; DV conferido com <N> vetores reais."

---

### Task 7: Espelho da FPO — linhas e importação

**Files:**
- Create: `app/services/regulation/{fpo_pick,fpo_csv}.rb`, `app/commands/regulation/{save_fpo_line,upload_fpo_import,apply_fpo_import,discard_fpo_import}.rb`, `app/controllers/care_regulation/admin/{fpo_lines,fpo_imports}_controller.rb`, `spec/services/regulation/{fpo_pick,fpo_csv}_spec.rb`, `spec/requests/care_regulation/fpo_spec.rb`, `spec/fixtures/regulation/{fpo_ok,fpo_excel}.csv`
- Modify: `config/routes.rb`

**Interfaces:**
- Produces: Valor 8. `SaveFpoLine.call(attrs:, by:, line: nil, expected_version: nil) -> Result.ok(line:) | fail(:invalid_request | :duplicate_fpo_line | :expected_version_required | :fpo_line_changed, details: { path: })` (POST com `competence AAAAMM`, `care_provider_id` de prestador em `CEILING_PROVIDER_KINDS`, `level`, `code`, `financing`, `faec_program` (de `faec_programs.yml`), `quantity`, `amount_cents`, `complement_cents_per_unit`; PATCH só `quantity`, `amount_cents` e `complement_cents_per_unit`).
- `Regulation::FpoCsv.read(bytes) -> Result.ok(rows:) | fail(:invalid_file | :file_too_large, details: { path: "file_base64" })`; cabeçalho `competencia;cnes;nivel;codigo;financiamento;programa_faec;quantidade;valor;complemento_unitario` (valores `1234,56`; complemento vazio = 0; BOM, CRLF e `;` aceitos, como o 27b). Validação por linha com códigos `provider_not_found` (CNES de prestador da cidade, `own` pelo CNES da unidade), `invalid_level`, `invalid_code`, `invalid_financing`, `invalid_faec_program`, `faec_not_procedure`, `invalid_quantity`, `invalid_amount`, `duplicate_line`.
- `UploadFpoImport.call(bytes:, by:)` grava `validated` com `rows` e `report` (`{ rows_valid, rows_error, errors_by_code, totals_by_provider: [{ care_provider_id, lines, amount_cents }] }`); `ApplyFpoImport.call(import:, expected_version:, by:)` cria a linha nova ou atualiza `quantity`/`amount_cents` da existente (versão + 1), numa transação, e grava `applied`; `DiscardFpoImport`. Versão diferente → `fpo_import_changed`.

- [ ] **Step 1: Write the failing tests.** `fpo_pick_spec.rb` (`type: :model`): "com linhas de grupo `04`, subgrupo `0407` e procedimento, o procedimento ganha"; "sem a do procedimento, o subgrupo"; "financiamento diferente não serve"; "FAEC com programa diferente não serve"; "outra competência não serve"; "o valor da reserva soma o `complement_cents_per_unit` da linha". `fpo_csv_spec.rb`: "o `fpo_excel.csv` dá as linhas do `fpo_ok.csv`"; um teste por código de linha. `fpo_spec.rb` (`type: :request`): "admin cria linha; duplicada → 409 `duplicate_fpo_line`"; "PATCH sem `expected_version` → 422; velha → 409 `fpo_line_changed`"; "prestador `ppi` → 422 path `care_provider_id`"; "upload → relatório sem conteúdo do arquivo; apply com step-up cria e atualiza; apply sem step-up → 401"; "6 MB → 413 `file_too_large`"; "`template.csv` é o cabeçalho exato".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/fpo_pick_spec.rb spec/services/regulation/fpo_csv_spec.rb spec/requests/care_regulation/fpo_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: mirror the fpo per provider with reviewed csv import" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 7 · fpo_lines com linha mais específica, complemento por unidade (Q9), importação CSV conferida (upload → apply com step-up, Q2) em fpo_imports (Q10)."

---

### Task 8: `Authorize` com autorizador, teto e número; prévia, cancelamento e falta

**Files:**
- Create: `app/services/regulation/{authorizer,reservations,validity,authorization_preview}.rb`, `app/commands/regulation/save_apac_settings.rb`, `app/controllers/care_regulation/admin/apac_settings_controller.rb`, `spec/services/regulation/reservations_spec.rb`, `spec/commands/care_regulation/{authorize_apac_spec,authorize_apac_concurrency_spec}.rb`, `spec/requests/care_regulation/{apac_settings,authorization_preview}_spec.rb`
- Modify: `app/commands/care_regulation/{authorize,decide}.rb`, `app/services/care_regulation/{misses,appointment_sync,json}.rb`, `app/controllers/care_regulation/decisions_controller.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/requests/care_regulation/decisions_spec.rb`

**Interfaces:**
- `Regulation::Authorizer.check(user:, request:) -> :ok | :authorizer_not_allowed`: `user.professional` com `ProfessionalLink.active` de `cbo_code` começando por `"2"` (Q1) e `user.id != request.requester_user_id`. Vale para `LAUDO_KINDS` (o BPA fica como está, decisão 21).
- `Regulation::Reservations.reserve!(authorization:, line:, cents:, quantity:, competence:)`; `.release!(request:, now:)` (marca `released_at` na reserva viva do pedido, publica `fpo_reservation.released`); `.transfer!(from:, to:)` (reautorização, abaixo); `.move!(authorization:, competence:, now:) -> :moved | :kept` (Task 10). Saldo pelo Valor 8, contado sob `FpoLine.lock`.
- `Regulation::AuthorizationPreview.call(request:, offer:, financing:, faec_program:, today:)` → o corpo da rota de prévia do Valor 11 (Q13), pelas mesmas regras do `Authorize`, sem trava e sem gravar.
- `Authorize.call(…, financing: nil, faec_program: nil, budget_source: nil, ceiling_override_reason: nil, step_up: false, apac: false)`. Ordem de travas **oferta → pedido → linha da FPO → faixa** (Q6). Oferta: tipo por laudo aceita a do mês corrente e dos `OFFER_MONTHS_AHEAD` seguintes (Q13); o BPA segue só o mês corrente. Depois dos passos 1–7 de hoje: (a) tipo por laudo sem `apac` → `invalid_target`; (b) `Authorizer.check` → 403 `authorizer_not_allowed`; (c) financiamento: o enviado, ou `FINANCING_DEFAULT[procedure.financing_code]` (outro → `other`); `credits` exige `budget_source` (path `budget_source`); `faec_program` de `faec_programs.yml` (path `faec_program`) (Q9); (d) teto: financiamento em `CEILING_FINANCINGS` e prestador em `CEILING_PROVIDER_KINDS` com alguma `fpo_lines` da competência → `FpoPick`; sem linha ou sem saldo → 409 `ceiling_exhausted`, salvo `ceiling_override_reason` em `REASON` com `step_up` (sem step-up → `mfa_required`, 401); vale também para o BPA (Q5) quando o interruptor está ligado, e o BPA por especialidade pelo SIGTAP do 26c (Valor 8); `other` → sem reserva; (e) número: tipo por laudo, prestador fora de `ppi`, momento `before` → `AuthorizationNumber.next!` com `FifthDigit.for` e `stage` `issued`; `after` → `awaiting_review`; `ppi` → sem número nem `stage`; (f) `RegulationAuthorization` com `authorization_kind`, `valid_from/until_competence` (`Validity`), `financing`, `faec_program`, `budget_source`, `fpo_line_id`, `reserved_cents`, `authorizer_professional_id`, `ceiling_override_reason`. Eventos: `fpo_reservation.created` (`fpo_reservation_id`, `regulation_authorization_id`) e, com número, `regulation_authorization.issued` (`regulation_authorization_id`, `user_id`).
- **Vida da reserva (Q9):** fica enquanto o pedido está aberto, **inclusive depois da 1ª falta** (`Misses.apply!` não mexe nela). Sai no cancelamento (`Decide`), na liberação do `AppointmentSync`, no encerramento (`closed` pela 2ª falta, `Misses.apply!`; `deny` da revisão, Task 9; OCI `not_concluded`, Task 11), sempre na mesma transação, depois da trava do pedido. `completed` não libera (o valor foi usado). Reautorização depois da 1ª falta: a reserva viva passa à autorização nova (`transfer!`: reserva nova com os mesmos `fpo_line_id`, `cents` e competência, a antiga com `moved_to_reservation_id`, sem conferir saldo de novo); se a linha escolhida agora é outra, a antiga é liberada e a nova passa pelo teto. O número da autorização antiga nunca volta; a nova tira número novo.
- `GET|PUT /regulation/admin/apac_settings` → `{ authorization_moment, version }`; `SaveApacSettings.call(attrs:, expected_version:, by:)` → `apac_settings_changed`.
- `Json.authorization` + `authorization_kind`, `number`, `stage`, `valid_from_competence`, `valid_until_competence`, `financing`, `faec_program`, `reserved_cents`, `fpo_line` (`{ id, level, code, amount_cents, complement_cents_per_unit, balance_cents }`), `ceiling_overridden` (booleano; o motivo nunca sai), `reservation_mismatch`.

- [ ] **Step 1: Write the failing tests.** `authorize_apac_spec.rb` (`type: :model`): "APAC em prestador `contracted` com FPO: reserva (SA+SH+SP + complemento da linha) × quantidade, número APAC-2, `issued`, validade de 2 competências"; "OCI: dígito 7"; "AIH com atributo 051: dígito 5, `valid_until` nulo"; "`credits` sem `budget_source` → path `budget_source`; com → dígito 9"; "procedimento de financiamento `01` → `other`, sem reserva, com número"; "oferta do mês +2 aceita no laudo; +3 → `invalid_offer`; pedido BPA com oferta do mês seguinte → `invalid_offer`" (Q13); "solicitante autorizando → `authorizer_not_allowed`"; "autorizador com vínculo só de CBO `322205` → `authorizer_not_allowed`"; "sem saldo → `ceiling_exhausted`, nada gravado, número não consumido"; "com motivo e step-up passa e marca `ceiling_overridden`; sem step-up → `mfa_required`"; "prestador sem FPO na competência → sem reserva, com número"; "prestador `ppi` → sem número e sem reserva"; "momento `after` → `awaiting_review` sem número"; "faixa esgotada → `number_range_exhausted` e reserva desfeita"; "pedido BPA por especialidade com FPO → reserva pelo SIGTAP do `specialty_sigtap`; interruptor desligado → como hoje" (Q5, Q9); "cancelar devolve a reserva"; "1ª falta mantém a reserva; reautorizar transfere a reserva sem conferir saldo e tira número novo; 2ª falta (`closed`) devolve" (Review Focus 3); "`completed` mantém a reserva". `authorize_apac_concurrency_spec.rb`: "duas ofertas, uma linha de grupo com saldo para uma: uma passa, a outra `ceiling_exhausted`" (Review Focus 1). `authorization_preview_spec.rb` (`type: :request`): "mostra linha, saldo e reserva sem gravar; o financiamento enviado troca a linha"; "não traz laudo". `apac_settings_spec.rb` e `decisions_spec.rb` (`type: :request`): step-up e versão; "regulador lê as configurações e não grava" (Q13); "o corpo do authorize aceita as chaves novas e recusa outra"; "a resposta não traz o motivo do teto"; "interruptor desligado: cancelar pedido `apac` autorizado devolve a reserva" (Review Focus 5). Bindings com os eventos novos (`to: []`).
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/reservations_spec.rb spec/commands/care_regulation/authorize_apac_spec.rb spec/commands/care_regulation/authorize_apac_concurrency_spec.rb spec/requests/care_regulation/apac_settings_spec.rb spec/requests/care_regulation/authorization_preview_spec.rb spec/requests/care_regulation/decisions_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o controller lê `Regulation::ApacGate.usable?` e `reauthenticated_recently?` antes de chamar; a trava da faixa é a última).
- [ ] **Step 4:** os mesmos specs + `spec/commands/care_regulation spec/services/care_regulation spec/requests/care_regulation` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: check the authorizer, reserve the fpo ceiling and number on authorize" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 8 · Authorize com autorizador (Q1), oferta até +2 no laudo (Q13), reserva com complemento da linha e other sem teto (Q9), ceiling_exhausted com override por step-up, número no modo before; travas oferta → pedido → FPO → faixa (Q6); reserva fica após a 1ª falta e sai no cancelamento/encerramento (Q9); prévia (Q13)."

---

### Task 9: Modo "autorizar depois"

**Files:**
- Create: `app/commands/regulation/review_authorization.rb`, `app/controllers/care_regulation/authorizations_controller.rb`, `spec/commands/regulation/review_authorization_spec.rb`, `spec/requests/care_regulation/authorizations_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `ReviewAuthorization.call(authorization:, decision:, reason:, version:, by:) -> Result.ok(request:, authorization:) | fail(:invalid_request | :stale_version | :invalid_transition | :authorizer_not_allowed | :number_range_exhausted)`: só `stage` `awaiting_review` e autorização viva; `Authorizer.check`; `issue` → número (`FifthDigit` com o financiamento gravado), `issued`, `regulation_authorization.issued`; `deny` (motivo obrigatório em `REASON`, Q10, cifrado no andamento do pedido) → `review_denied`, `Reservations.release!`, `regulation_authorization.review_denied`, e os procedimentos realizados ligados recalculados (Task 10). Travas: pedido → reserva → faixa.
- `GET /regulation/authorizations?stage=awaiting_review` (lista com `{ id, regulation_request_id, request_ref, authorization_kind, target_name, care_provider, performed: [n de m] }`, sem laudo).

- [ ] **Step 1: Write the failing tests.** `review_authorization_spec.rb` (`type: :model`): "`issue` tira o número e publica `issued`"; "`deny` sem motivo → path `reason`; com motivo devolve a reserva e publica `review_denied`"; "revisar `issued` → `invalid_transition`"; "o próprio solicitante → `authorizer_not_allowed`"; "versão velha → `stale_version`". `authorizations_spec.rb` (`type: :request`): lista e revisão; corpo com chave desconhecida → 422; o motivo não volta na resposta.
- [ ] **Step 2:** `bin/test api -w mod26b spec/commands/regulation/review_authorization_spec.rb spec/requests/care_regulation/authorizations_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/regulation` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: review authorizations after care in the after mode" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 9 · revisão issue/deny (com motivo, Q10) no modo after."

---

### Task 10: Faturamento — ligação, migração da reserva, número no BPA-I e `oci_secondary_active`

**Files:**
- Create: `app/services/regulation/{performed_link,oci_block}.rb`, `spec/services/regulation/performed_link_spec.rb`, `spec/services/billing/critic_oci_spec.rb`
- Modify: `app/commands/billing/record.rb`, `app/services/billing/{critic,recalculate,lines,bpa_file}.rb`, `spec/services/billing/{critic,bpa_file}_spec.rb`

**Interfaces:**
- `Regulation::PerformedLink.for(patient:, establishment:, sigtap_code:, performed_on:) -> RegulationRequestedProcedure | nil` (Q10): procedimento solicitado ainda não realizado, com o código, de pedido `apac`/`oci`/`aih` do paciente cuja autorização vigente é viva, `issued` ou `awaiting_review`, de prestador `own` cuja unidade é o estabelecimento, com `performed_on` a partir de `valid_from` e até `valid_until` (nulo na AIH = aberto). `Billing::Record.insert!` liga (`regulation_authorization_id`, `authorization_number` = `number` quando há) e marca o solicitado (`performed_at`, `performed_source` `performed_procedure`, `performed_procedure_id`), na mesma transação.
- **Migração da reserva (Q9):** quando o realizado (ligado aqui ou marcado à mão na Task 11) cai numa competência diferente da reserva viva, `Reservations.move!`: linha pela mesma regra na competência nova; com saldo → reserva nova, a antiga com `moved_to_reservation_id`, `fpo_reservation.moved`; sem linha ou saldo → `:kept`, com `reservation_mismatch: true` no `Json.authorization`. Na AIH, o principal realizado também fecha a validade (`Validity.close_aih!`, Valor 9).
- `Regulation::OciBlock.active?(procedure) -> Boolean`: ligado a um secundário de pedido `oci` não encerrado `not_concluded` (Task 11).
- `Billing::Critic::REASONS` + `oci_secondary_active` (no fim); `Critic::Facts` + `oci_secondary_active` (padrão `false`, por `initialize(oci_secondary_active: false, **rest)`); `Record` e `Recalculate` passam `OciBlock.active?`.
- `BpaFile::ILine` + `authorization_number`; a posição de 13 da autorização sai `opt_num(l.authorization_number, 13)` no lugar do `blank(13)`; `Lines` lê `p.authorization_number`.

- [ ] **Step 1: Write the failing tests.** `performed_link_spec.rb` (`type: :model`): "consulta 0301010072 registrada na unidade do prestador para o paciente da OCI emitida liga e marca o secundário"; "outra unidade, outro paciente, fora da validade ou já realizado → nil"; "autorização liberada → nil"; "realizado no mês seguinte com linha e saldo → reserva movida, a antiga com `moved_to_reservation_id`; sem linha → fica e `reservation_mismatch`"; "AIH: o principal realizado em 202612 fecha `valid_until` em 202703". `critic_oci_spec.rb`: "secundário de OCI ativa → `oci_secondary_active`, não faturável"; "OCI `not_concluded` → faturável" (Review Focus 4). `bpa_file_spec.rb`: "linha com `authorization_number` escreve os 13 dígitos na posição do layout; sem número, branco como hoje" (posição conferida contra o layout do BPA usado no 22). `critic_spec.rb`: a ordem de `REASONS`.
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/performed_link_spec.rb spec/services/billing/critic_oci_spec.rb spec/services/billing/critic_spec.rb spec/services/billing/bpa_file_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (o oráculo `contracts/billing/tools/bpa_oracle.py`, se o spec do 22 o usa, segue verde).
- [ ] **Step 4:** os mesmos specs + `spec/services/billing spec/commands/billing spec/requests/billing` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: link performed procedures to authorizations and block active oci secondaries" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 10 · procedimento realizado ligado à autorização (Q10), reserva migra pela competência do realizado (Q9), número no BPA-I, oci_secondary_active no Critic."

---

### Task 11: OCI — andamento, encerramento e alarme

**Files:**
- Create: `app/commands/regulation/{mark_performed,close_oci}.rb`, `app/services/regulation/oci_list.rb`, `app/controllers/care_regulation/oci_controller.rb`, `spec/commands/regulation/close_oci_spec.rb`, `spec/requests/care_regulation/oci_spec.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `MarkPerformed.call(request:, procedure:, performed_at:, version:, by:)`: só pedido `oci` (ou `apac`/`aih` de prestador externo) com autorização viva; data a partir de `valid_from`, até `valid_until` (quando há) e ≤ hoje; `performed_source` `manual`, `performed_by_user_id`; já realizado → `invalid_transition`; competência diferente da reserva → `Reservations.move!` (Task 10).
- `CloseOci.call(request:, outcome:, exit_reason:, version:, by:)` (Q7): `outcome` em `OCI_OUTCOMES` e `exit_reason` de `oci_exit_reasons.yml` nos dois (path `exit_reason`); `concluded` exige ≥ 2 secundários realizados (`oci_needs_two_secondaries`) e um deles em `oci_consultation_codes.yml` (`oci_needs_consultation`); `not_concluded` devolve a reserva e recalcula (`Billing::Recalculate`) os procedimentos ligados em competência aberta. A OCI vencida também se encerra por aqui. Grava `oci_outcome`, `oci_exit_reason`, `oci_closed_at` no pedido (colunas da Task 3) e publica `oci.closed` (`regulation_request_id`, `user_id`).
- `Regulation::OciList.page(before:, limit:, today:)` → `{ items: [{ id, request_ref, target_name, care_provider, authorization_number, valid_until_competence, secondaries: [{ id, sigtap_code, name, performed_at }], days_open, oncology_alert, expired }], next_before }`: `days_open` contado do primeiro procedimento realizado (nulo sem nenhum, Q7); `oncology_alert` = principal em `oci_oncology.yml` e `days_open > ONCOLOGY_ALERT_DAYS`; `expired` = hoje depois de `valid_until_competence`, e a vencida entra no alarme (Q7).

- [ ] **Step 1: Write the failing tests.** `close_oci_spec.rb` (`type: :model`): "`concluded` com os dois secundários e a consulta grava e publica"; "`concluded` com um secundário → `oci_needs_two_secondaries`; sem a consulta → `oci_needs_consultation`"; "`exit_reason` 2.6 ou ausente → path `exit_reason`, nos dois `outcome`"; "`not_concluded` devolve a reserva e o secundário volta ao BPA-I no recálculo"; "vencida encerra"; "encerrar duas vezes → `invalid_transition`". `oci_spec.rb` (`type: :request`): "marcar à mão um secundário de prestador externo"; "data fora da validade → 422 path `performed_at`"; "oncologia: `oncology_alert` no 26º dia do 1º realizado e não no 25º; sem realizado → `days_open` nulo, sem alerta"; "vencida aparece com `expired`"; "a lista não traz laudo nem CID".
- [ ] **Step 2:** `bin/test api -w mod26b spec/commands/regulation/close_oci_spec.rb spec/requests/care_regulation/oci_spec.rb spec/initializers/domain_events_bindings_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/billing` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: track oci secondaries, close oci and flag oncology deadlines" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 11 · secundários pelo prontuário ou à mão, oci_close com outcome e exit_reason (Q7), alarme de oncologia no 26º dia do 1º realizado e vencidas no alarme."

---

### Task 12: Remessa APAC da rede própria

**Files:**
- Create: `app/services/regulation/apac_file.rb`, `app/commands/regulation/generate_apac_export.rb`, `app/controllers/care_regulation/apac_exports_controller.rb`, `spec/services/regulation/apac_file_spec.rb`, `spec/requests/care_regulation/apac_exports_spec.rb`, `spec/fixtures/regulation/apac_layout_<versão>.csv` (cópia do vendor)
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- `Regulation::ApacFile::LAYOUT_VERSION` (a da Task 0); `ApacFile.build(header:, apacs:) -> Built(bytes:, apacs:)` com as linhas do layout (corpo, procedimentos e cabeçalho como o documento manda; `num`/`alfa`/`blank` no molde do `BpaFile`; `inspect` sem dado pessoal); dados: cabeçalho do `BillingSetting.current`; cada APAC com número, CNES da unidade, competência, paciente (CPF ou CNS), CID, autorizador (CNS), datas de validade, motivo de saída e procedimentos (OCI: secundários com valor zerado).
- Seleção (Q10): autorizações `apac`/`oci` `issued` de prestador `own` da unidade que tenham procedimento realizado na competência. Unidade sem prestador `own` → 422 `not_own_provider` (path `health_unit_id`). Quem gera e baixa: `municipal_admin` ou `billing_clerk` (`Billing::Access.clerk?`).
- `GenerateApacExport.call(health_unit:, competence:, by:, now:) -> Result.ok(export:) | fail(:settings_missing | :nothing_to_export | :not_own_provider | :invalid_request)`: versão = última + 1; `sha256`; `apac_export.generated` (`apac_export_id`, `user_id`).

- [ ] **Step 1: Write the failing tests.** `apac_file_spec.rb` (`type: :model`): "cada campo do `apac_layout_<versão>.csv` sai na posição, no tamanho e no tipo (numérico com zeros à esquerda, alfanumérico com espaços)"; "OCI com dois secundários: valor zerado"; "texto com acento vira o que o layout aceita"; "linha com o tamanho total do layout". `apac_exports_spec.rb` (`type: :request`): "admin gera para a unidade própria e baixa; o SHA-256 confere"; "`billing_clerk` gera"; "unidade de prestador contratado → 422 `not_own_provider`"; "autorização emitida sem procedimento realizado na competência fica fora"; "unidade sem APAC → 422 `nothing_to_export`"; "regulador → 403"; "a lista não traz o arquivo".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/apac_file_spec.rb spec/requests/care_regulation/apac_exports_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: generate the apac export of the own network in the official layout" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 12 · ApacFile no layout <versão>, campo a campo contra a tabela oficial; remessa versionada com SHA-256 (gate: VERSIA)."

---

### Task 13: Laudo em PDF

**Files:**
- Create: `app/services/regulation/laudo_pdf.rb`, `app/controllers/care_regulation/laudos_controller.rb`, `spec/services/regulation/laudo_pdf_spec.rb`, `spec/requests/care_regulation/laudos_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- `Regulation::LaudoPdf.call(laudo, footer: nil) -> String` (bytes), na folha de `Consultations::Print.document(footer, title:)` e com `Print.safe`: título por tipo ("Laudo para solicitação de APAC", "… de OCI", "… de AIH"); estabelecimento solicitante (nome, CNES); paciente (nome, CPF ou CNS, nascimento); procedimentos (código, nome, quantidade, papel); CID principal e secundário; justificativa clínica; achados; caráter e clínica (AIH); solicitante (nome, CBO, conselho); bloco do autorizador com o número, a validade e o executante quando emitido; linhas de assinatura quando `footer` é nil.
- `GET /regulation/requests/:id/laudo.pdf` (regulador ou o solicitante; outro → 403); `Cache-Control: no-store`.

- [ ] **Step 1: Write the failing tests.** `laudo_pdf_spec.rb` (`type: :model`, texto extraído com `PDF::Reader` como no `guide_pdf_spec.rb`): "OCI traz os secundários e a justificativa"; "AIH traz caráter e clínica"; "emitida traz o número; sem número, o bloco vazio"; "sem rodapé, as linhas de assinatura". `laudos_spec.rb` (`type: :request`): "o regulador baixa; o solicitante baixa; outro profissional → 403"; "`no-store`".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/laudo_pdf_spec.rb spec/requests/care_regulation/laudos_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: print the regulation laudo pdf" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 13 · LaudoPdf na folha do Consultations::Print, gerado na hora, com o número quando emitido."

---

### Task 14: Fila nominal da OCI

**Files:**
- Create: `app/services/regulation/{oci_queue_file,business_days}.rb`, `app/commands/regulation/generate_oci_queue_export.rb`, `app/jobs/regulation/oci_queue_export_job.rb`, `app/controllers/care_regulation/oci_queue_exports_controller.rb`, `spec/services/regulation/oci_queue_file_spec.rb`, `spec/jobs/regulation/oci_queue_export_job_spec.rb`, `spec/requests/care_regulation/oci_queue_exports_spec.rb`
- Modify: `config/routes.rb`, `config/recurring.yml`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- **Depende do layout do e-SUS Captação de Filas da Task 0; sem ele, só esta task para e o relato diz isso (Q6).**
- `OciQueueFile.build(competence:, requests:) -> String` (CSV no layout da Task 0; uma linha por pedido `oci` com o último status do mês, CPF ou CNS, OCI e município de origem; `inspect` sem dado pessoal).
- `GenerateOciQueueExport.call(competence:, by:, now:)`; `oci_queue_export.generated` (`oci_queue_export_id`, `user_id` nulo no job).
- `Regulation::OciQueueExportJob` (`prepend EachCityJob`; `config/recurring.yml` diário às 06:00): no 1º dia útil do mês (segunda a sexta, sem os feriados de `config/regulation/national_holidays.yml`, Q6; `Regulation::BusinessDays.first_of(month) -> Date`), com o interruptor `usable?`, gera a do mês anterior se ainda não existe.

- [ ] **Step 1: Write the failing tests.** `oci_queue_file_spec.rb`: "cabeçalho e colunas do layout"; "último status do mês"; "pedido sem CPF sai com CNS". `oci_queue_export_job_spec.rb` (`type: :job`): "segunda, 1º dia útil, gera a do mês anterior uma vez"; "sábado dia 1 → nada; segunda dia 3 → gera"; "1º de maio numa segunda → gera no dia 2" (Q6); "interruptor desligado → nada". `oci_queue_exports_spec.rb` (`type: :request`): "admin gera à mão e baixa"; "regulador → 403".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/oci_queue_file_spec.rb spec/jobs/regulation/oci_queue_export_job_spec.rb spec/requests/care_regulation/oci_queue_exports_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/architecture/current_city_assignment_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: export the nominal oci queue on the first business day" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 14 · fila nominal no layout e-SUS Captação de Filas, job no 1º dia útil e geração à mão (plano B, Q3)."

---

### Task 15: Assinatura do laudo (`rotasaude.regulation_laudo.v1`)

**Files:**
- Create: `config/clinical/regulation-laudo-v1.json`, `spec/fixtures/clinical/regulation-laudo/*` (cópias da branch do contracts, presas pelo SHA-256 do plano irmão, Task 4 dele), `spec/services/signatures/regulation_laudo_canonical_spec.rb`
- Modify: `app/services/signatures/{document_types,canonical,documents,signing,json}.rb`, `app/commands/care_regulation/promote.rb`, `app/commands/consultations/{finalize,add_addendum}.rb`, `spec/services/signatures/canonical_vector_spec.rb`, `spec/fixtures/clinical/canonical/{SHA256SUMS,laudo-oci.jcs,laudo-aih.jcs}`

**Interfaces:**
- `DocumentTypes::MODELS` + `"RegulationLaudo" => "regulation_laudo"` (no banco o nome do modelo, como `LabReport`; na API e nos eventos `regulation_laudo`, Q10); `consultation_id(laudo)` = o do pedido. `Canonical::LAUDO_SCHEMA = "rotasaude.regulation_laudo.v1"` em `SCHEMA_FILES`; `Canonical.regulation_laudo(laudo)` = `header_for(unit: request.requester_unit, author: request.requester_user, cbo_code: request.requester_cbo, patient: request.patient)` + `request` e `laudo` na forma do esquema do plano irmão; `Canonical.for` e `Documents.for` (PDF = `Regulation::LaudoPdf.call(laudo, footer:)`) com o tipo; `Signing::UNCHAINED` + `RegulationLaudo`; `chronological` pelo `requested_at`.
- `Promote` abre a assinatura de cada laudo promovido: `Signatures::OpenRequest.call(laudo, now:, usable: signing)` (`signing` lido fora da transação pelo `Finalize`/`AddAddendum`); `:manual` → nada (o PDF sai para assinar à mão, Task 13) (Q10). Paciente sem CPF → não abre, e o laudo sai impresso para assinatura à mão (o `$defs/patient` do domínio exige CPF; Q12).

- [ ] **Step 1: Write the failing tests.** `regulation_laudo_canonical_spec.rb` (`type: :model`): "o canônico do laudo OCI passa no esquema copiado"; "AIH com `character` e `clinic`; APAC sem as duas chaves"; "determinístico"; "`Documents.for` monta canônico e PDF"; "finalizar com certificado abre o pedido de assinatura do laudo; sem certificado → nenhum (manual); paciente sem CPF → nenhum". `canonical_vector_spec.rb`: "os SHA-256 dos dois vetores do laudo são os do contracts"; os manifestos de `regulation-laudo/` contra a cópia.
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/signatures/regulation_laudo_canonical_spec.rb spec/services/signatures/canonical_vector_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/signatures spec/commands/signatures spec/jobs/signatures` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: sign the regulation laudo with the clinical-v1.3.0 schema" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 15 · RegulationLaudo assinável (CAdES do JCS + PAdES do LaudoPdf), aberto na promoção; esquema e vetores presos ao SHA-256 do contracts (sem tag ainda)."

---

### Task 16: MIRA (26c) — modalidade 04 e alvo por laudo

**Files:**
- Modify: `app/services/regulation/{rnds_limits,rira_facts,rira_bundle,rnds_validation}.rb`, `spec/services/regulation/{rira_bundle,rnds_validation}_spec.rb`

**Interfaces:**
- `Regulation::RndsLimits::MODALITY_BY_KIND = { "bpa" => "09", "apac" => "09", "oci" => "09", "aih" => "04" }` (o `MODALITY` do 26c sai, Q11); `RiraFacts` + `modality`; o `RiraBundle` usa `facts.modality` em `Composition.category`, `Appointment.serviceCategory` e `ServiceRequest.category`.
- `RndsValidation`: `specialty_without_sigtap` só vale para alvo `specialty` (Q11); o pedido por procedimento (BPA ou laudo) não passa mais por essa conferência.

- [ ] **Step 1: Write the failing tests.** `rira_bundle_spec.rb`: "pedido `aih` sai com a categoria 04 nas três"; "`oci` e `bpa` seguem 09"; "o digest do pedido BPA não muda". `rnds_validation_spec.rb`: "pedido `apac` só-APAC não cai em `specialty_without_sigtap`"; "pedido por especialidade com SIGTAP fora da vigência segue caindo".
- [ ] **Step 2:** `bin/test api -w mod26b spec/services/regulation/rira_bundle_spec.rb spec/services/regulation/rnds_validation_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** os mesmos specs + `spec/services/regulation spec/jobs/regulation spec/lib/fake_rnds_spec.rb` → PASS.
- [ ] **Step 5: Commit** `git commit -m "feat: send aih regulation requests to mira with modality 04" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 16 · RIRA lê authorization_kind (aih → 04); specialty_without_sigtap só para alvo de especialidade (Q11)."

---

### Task 17: Semente de dev, sigilo e eventos

**Files:**
- Create: `lib/regulation_apac_crew.rb`, `spec/lib/regulation_apac_crew_spec.rb`, `spec/security/regulation_apac_secrecy_spec.rb`
- Modify: `db/seeds.rb`, `config/initializers/filter_parameter_logging.rb`

**Interfaces:**
- `RegulationApacCrew.seed_platform!` (SIGTAP de dev com financiamento, valores e compatibilidades, pelo `SigtapSample` com `billing: true`) e `.seed_current_city(slug:)` depois do `RegulationCrew`: interruptor ligado, faixas APAC 2/7 e AIH 5 de dev (marcadas "dev" no `summary_lines`), FPO de um prestador contratado com saldo para duas autorizações, momento `before`, uma OCI emitida com um secundário feito, uma AIH `awaiting_review` (em outra cidade de dev com `after`, se houver) e um prestador sem FPO. `summary_lines` sem dado de staging ou produção.
- `spec/security/regulation_apac_secrecy_spec.rb` (`type: :request`, molde do `vector_control_secrecy_spec.rb`, constantes num módulo `RegulationApacSecrecy`, ASCII, palavras únicas): justificativa clínica, achados, CID principal e secundário, motivo do teto, motivo da revisão negada, CPF e CNS do paciente marcados; cenário pelas rotas reais (rascunho por laudo, promoção, autorização com teto estourado e override, revisão negada, OCI marcada e encerrada, remessa, fila nominal, PDF).

- [ ] **Step 1: Write the failing tests.** `regulation_apac_crew_spec.rb` (`type: :model`): "idempotente"; "o teto do prestador contratado não é zero" (pesquisa, achado 3); "nenhum segredo nas `summary_lines`". `regulation_apac_secrecy_spec.rb`: `capture_log` sem marca; eventos publicados do módulo ⊂ `ApacCatalog::DOMAIN_EVENTS` + os do 26a, só ids e `user_id`; nenhuma lista (fila, `authorizations`, `oci`, `apac_exports`, `oci_queue_exports`) traz marca; toda recusa 4xx só com `error`, `path`, `feature`; nenhuma rota com marca na URL; grep multilinha de `Platform.audit(` nos arquivos do módulo: nenhum; laudo, motivo e arquivos só cifrados no banco.
- [ ] **Step 2:** `bin/test api -w mod26b spec/lib/regulation_apac_crew_spec.rb spec/security/regulation_apac_secrecy_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture` → FAIL (a semente) e PASS (o sigilo, se as tasks donas estiverem certas; falha → correção na task dona, volta aqui).
- [ ] **Step 3: Implementar** a semente e o filtro de log.
- [ ] **Step 4:** os mesmos specs → PASS.
- [ ] **Step 5: Commit** `git commit -m "test: pin regulation apac secrecy and seed dev data" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato ao coordenador:** "api 26b · Task 17 · semente com teto não zero e sigilo fixado (laudo, CID, motivos e arquivos fora de log, evento, URL e lista)."

---

### Task 18: Entrega — suíte completa e passagem aos fronts (para antes do merge)

**Files:** nenhum código novo.

- [ ] **Step 1: Suíte completa:** pedir à sessão "Testes", por `SendMessage`: `api · mod26b/feat/mod-26b-apac · --full · módulo 26b (migração de cidade e de plataforma, SIGTAP, G11, Authorize/Decide/Misses/AppointmentSync do 26a, Billing::Critic/Record/BpaFile do 22, signer e Canonical do 19b, RIRA do 26c, CityEncryption, Platform::Features, spec/support, initializers)`. Falha → diagnóstico nesta sessão, correção na task dona, novo pedido. Verde → avisar o coordenador com o resumo e o hash, a sessão do contracts (o api usou o SHA-256 da branch; a tag `clinical-v1.3.0` pode sair) e as do dashboard e do maintenance (porta 3064).
- [ ] **Step 2: Parar.** Merge, push e cards só com autorização do usuário; nenhum card vai a Verified por este plano. Os gates de go-live da spec §6 (faixas da SES da cidade piloto, FPO conferida, arquivo APAC aprovado no VERSIA, canal da fila nominal com o Ministério, habilitações 38.01 e 29.02 no CNES) e a APAC de continuidade (decisão 15) vão ao board #2 no fechamento do módulo (outra sessão), com o ADR novo do 26b.

**Relato ao coordenador:** "api 26b · Task 18 · suíte completa <verde N/0 | falhas>; hash <…>; contracts liberado para a tag; pronto para dashboard e maintenance; parado antes do merge."

---

## Self-review

1. **Cobertura do contrato:** §1 interruptor (2); §2 SIGTAP (1); §3 alvo (4), `authorization_kind`, laudo e procedimentos (3, 5), regras da OCI (5); §4 faixas (3, 6), número (6), colunas (3), FPO (3, 7), `Authorize` (8), onde vale o teto (8), cancelamento (8), migração (9), configurações (8); §5 modo depois (9); §6 OCI (10, 11); §7 faturamento (10) e remessa (12); §8 fila nominal (14); §9 AIH e MIRA (5, 8, 16); §10 PDF (13) e assinatura (15); §11 eventos (8, 9, 11, 12, 14) e LGPD (17); Q1 (8), Q2 (7), Q3 (14), Q4 (15), Q5 (8), Q6 (0, 4, 5, 8, 14), Q7 (5, 11), Q8 (6), Q9 (3, 7, 8, 10), Q10 (9, 10, 12, 15), Q11 (16), Q12 (15), Q13 (5, 8).
2. **Nomes:** `Regulation::{ApacGate, ApacCatalog, LaudoTargets, LaudoRules, FifthDigit, CheckDigit, AuthorizationNumber, FpoPick, FpoCsv, Validity, Reservations, Authorizer, PerformedLink, OciBlock, OciList, ApacFile, OciQueueFile, LaudoPdf, ApacJson}`; comandos `Regulation::{SaveLaudo, SaveApacSettings, SaveNumberRange, SaveFpoLine, UploadFpoImport, ApplyFpoImport, DiscardFpoImport, ReviewAuthorization, MarkPerformed, CloseOci, GenerateApacExport, GenerateOciQueueExport}`; job `Regulation::OciQueueExportJob`; `Regulation::{AuthorizationPreview, BusinessDays}`; `Terminology::SigtapBilling.loaded?`; `RegulationApacGate#require_care_regulation_apac!`.
3. **Ordem:** 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12 → 13 → 14 (para sem o layout do e-SUS, sem travar as outras) → 15 (depende da Task 4 do plano do contracts) → 16 → 17 → 18.

## Divergências propostas ao contrato

> D1–D13 **resolvidas no §12 do contrato (Q6–Q11 e Q13; docs `60908e1`)**. Cada item guarda o problema, a evidência e a proposta; a marca diz qual Q resolveu e, quando a decisão difere da proposta ou a completa, o que vale.

D1. **[Resolvida pela Q6.]** **Ordem de travas do `Authorize`.** O §4 fixa "oferta → linha da FPO → faixa → pedido", mas todo caminho que libera a vaga trava o pedido antes: o `Decide` (oferta → pedido → liberação, `decide.rb:9-11, 53-55, 71`), o `AppointmentSync` (`appointment_sync.rb:93-95, 215`) e o `Misses.record!` (`misses.rb:69`). Com a liberação da reserva nesses caminhos, um `Authorize` que segura a linha esperando o pedido e um `AppointmentSync` que segura o pedido esperando a linha fecham um ciclo. **Proposta:** oferta → pedido → linha da FPO → faixa em todos os caminhos (Tasks 8, 9, 11).

D2. **[Resolvida pela Q6.] A busca também devolve o `authorization_kind`.** **Alvo por laudo e `aih_instruments`.** (a) O `tb_registro` traz `03 AIH (Proc. Principal)` e `04 AIH (Proc. Especial)` (`vendor/regulation/2026-10-28/tb_registro.txt:3-4`); a decisão 6 da spec diz "03 e 04". Procedimento especial não é pedido sozinho. (b) O §3 não diz se procedimento com BPA e APAC (ex.: `0209010037`, `procedures.yml:11`) também entra por laudo, nem em que grupos vale o caminho de APAC/AIH; a pesquisa diz "só-APAC, só-AIH e grupo 09, sem relaxar o 26a" (achado 4). (c) `resolve_procedure` é privado e sem tipo, e a busca não recebe tipo (`targets.rb:70, 139`; `routes.rb:485`). **Proposta:** `aih_instruments: ["03"]`; APAC/AIH só nos grupos 02–04 e sem instrumento BPA; OCI = grupo 09 com 06; `resolve(…, kind:)`, `search(…, kind:)` e `GET /regulation/targets?kind=` (Task 4).

D3. **[Resolvida pela Q6; Q13 completa.] Diferente: o complemento aceita laudo e procedimentos (Q13) e só não troca o CID de laudo já assinado (Q6).** **CID e justificativa do laudo.** O pedido já guarda `cid10` cifrado e validado por CBO (`Diagnosis.check`, `diagnosis.rb:19`) e `justification` 10–2000 obrigatória (`save_draft.rb:64-66`); o §3 põe `cid_principal` e `clinical_summary` no laudo sem dizer a relação. Com o laudo assinado na promoção, o complemento que troca o `cid10` (`complement.rb:21`) deixaria laudo e pedido diferentes. **Proposta:** `cid_principal` = o `cid10` do pedido (obrigatório nos tipos por laudo; o corpo não traz `cid_principal`); a `justification` do 26a continua obrigatória e separada; o complemento de pedido por laudo não troca o `cid10` (Task 5).

D4. **[Resolvida pela Q7.] Diferente: as listas `oci_consultation_codes.yml` e `oci_oncology.yml` (oncologia pelo principal, não pelo subgrupo); `exit_reason` nos dois `outcome`; os 25 dias contam do 1º procedimento realizado (sem nenhum, sem contagem); a vencida entra no alarme e o regulador a encerra; recusas `oci_needs_two_secondaries`, `oci_needs_consultation`, `oci_principal_not_group_09`.** **Regras da OCI que faltam no §6.** (a) A decisão 10 da spec exige que um dos secundários seja consulta ou teleconsulta (pesquisa: `0301010072` ou `0301010307`), e o §3 não. (b) O §6 diz que a OCI "não concluída" vai ao BPA-I, mas o `oci_close` só tem `exit_reason`; não há como dizer "não concluída", nem colunas para o encerramento. (c) O alarme não diz como reconhecer a OCI de oncologia nem de quando contar os 25 dias; a OCI que passa da validade sem encerramento não tem regra. **Proposta:** (a) `invalid_request` path `procedures`; (b) `oci_close { outcome: concluded | not_concluded, exit_reason (só no concluded), version }` e as colunas `oci_outcome`, `oci_exit_reason`, `oci_closed_at` em `regulation_requests`; `not_concluded` devolve a reserva e desbloqueia o BPA-I; (c) subgrupos de oncologia em `procedures.yml` (`oci_oncology_subgroups`, conferidos no `tb_sub_grupo` na Task 0), dias contados do primeiro secundário realizado (sem nenhum, da emissão; pesquisa, achado 14), e a vencida só marcada `expired` na lista (Tasks 5, 11).

D5. **[Resolvida pela Q8.] A faixa devolve também o número inteiro (13 dígitos).** **Quinto dígito e forma da faixa.** O §4 dá `next!(kind:, fifth_digit:)` sem a regra de escolha; a decisão 22 diz "sai do tipo do pedido" e a pesquisa, do atributo SIGTAP (achado 2: 052 → 6, 051 → 5). O §4 diz "`first_number`/`last_number`/`next_number` sem o dígito verificador" e "UF + ano + tipo + sequencial" — não diz se a faixa guarda o sequencial de 7 dígitos ou os 12. A UF da cidade é sigla (`CityProfile.uf`), e o número usa o código IBGE. **Proposta:** Valor 7 (`FifthDigit.for`); a faixa guarda o sequencial de 7 dígitos com `year` de 4; UF = `ibge_code[0, 2]` (Tasks 3, 6).

D6. **[Resolvida pela Q9.] Diferente: outro código → `other`, sem reserva de teto; `faec_program` de `config/regulation/faec_programs.yml` (sem `rubric_code`); o valor soma o `complement_cents_per_unit` da linha da FPO.** **Financiamento e valor da reserva.** (a) O padrão vem do `financing_code` da SIGTAP, mas só `04 → faec` e `06 → mac` têm par; os outros códigos (`01`, `05`, `07`…) não. (b) `faec_program` não tem domínio. (c) Créditos (dígitos 9 e 8) exigem a "Fonte Orçamentária" 01–04 (pesquisa, linha 18), que o §4 não guarda. (d) O FAEC só se programa por procedimento (pesquisa, linha 50). (e) A spec §3.2 reserva "SIGTAP mais complemento"; o §4 reserva só "SA + SH + SP × quantidade", e não diz se os secundários entram. **Proposta:** (a) outro código → `mac`; (b) `faec_program` = `CO_RUBRICA` da SIGTAP (`sigtap_procedures.rubric_code`, 6 dígitos); (c) `budget_source` na autorização, obrigatória com `credits`; (d) CHECK `faec` → `procedure`; (e) só o principal × quantidade, sem complemento (o complemento vai ao board #2) (Tasks 1, 3, 8).

D7. **[Resolvida pela Q9.] Vale o mapeamento do 26c (`specialty_sigtap`).** **Teto do pedido BPA por especialidade.** A Q5 põe o teto nos pedidos BPA com FPO, mas o pedido por especialidade (`target_type` `specialty`, CBO) não tem código SIGTAP para achar a linha nem o valor. **Proposta:** usar o SIGTAP da especialidade das configurações do 26c (`RegulationRndsSetting.current.specialty_sigtap[code] || RndsLimits::DEFAULT_SIGTAP`), sem configuração nova; a alternativa mais simples é deixar a especialidade fora do teto (Task 8).

D8. **[Resolvida pela Q9.] Diferente: a reserva fica enquanto o pedido está aberto, inclusive depois da 1ª falta, e sai no cancelamento ou no encerramento; migra quando o procedimento realizado cai em outra competência (não no agendamento: sai o `ReservationMoveJob`); a AIH vale até 3 competências depois da competência da alta (o `valid_until` fecha no principal realizado).** **Vida da reserva: falta, migração e validade da AIH.** (a) A falta consome a vaga (`misses.rb:18`), e o §4 não diz o que acontece com a reserva nem com o número quando o pedido volta a `pending` e é reautorizado. (b) "A reserva migra" não diz quando, nem o que fazer se a competência nova não tem linha ou saldo. (c) "AIH: até 3 depois da alta" não se sabe na autorização (o 26b não registra a alta). **Proposta:** (a) a falta devolve a reserva e mantém a vaga; a reautorização reserva de novo e tira número novo; o antigo nunca volta; (b) migra no `regulation_request.scheduled` quando o mês do horário é outro, só com linha e saldo; senão fica, com `reservation_mismatch`; (c) validade da AIH = competência da oferta + 3 (Tasks 8, 9).

D9. **[Resolvida pela Q10.] Diferente na remessa: entram as `apac`/`oci` emitidas para unidade própria com procedimento realizado na competência; gera o `municipal_admin` ou o `billing_clerk`; unidade não própria → 422 `not_own_provider`.** **Ligação do procedimento realizado e seleção da remessa.** (a) O §6 e o §7 dizem que o secundário é marcado "pelo `performed_procedure` ligado", mas não dizem quem liga nem como. (b) O §7 não diz quais APAC entram na remessa de uma competência. **Proposta:** (a) ligação automática no `Billing::Record.insert!` (`record.rb:43`) pela regra do `Regulation::PerformedLink` (paciente, unidade do prestador `own`, código solicitado ainda não feito, validade); (b) OCI encerrada `concluded` naquela competência e APAC com o principal realizado nela (Tasks 10, 12).

D10. **[Resolvida pela Q10; Q13 completa.] Acrescenta a rota de prévia da autorização (Q13), a leitura do momento e da FPO pelo regulador (Q13) e o motivo obrigatório na negativa da revisão (Q10).** **Rotas, corpos, códigos e a conferência da FPO que o contrato não fixa.** O contrato nomeia `apac_settings`, `review`, `performed`, `oci_close` e `POST apac_exports`. Faltam as faixas, as linhas e a importação da FPO, a lista de revisão, a lista da OCI, a baixa das remessas, a fila nominal e o PDF; o corpo novo do `authorize`; o motivo do `deny`; e os códigos de sobreposição de faixa, linha duplicada e versão. A importação "como o saldo inicial do 27b" pede uma tabela de conferência (o 27b usa `opening_imports`), que o §4 não lista. **Proposta:** Valores 10 e 11 e a tabela `fpo_imports` (Tasks 3, 6–9, 11–14).

D11. **[Resolvida pela Q10.]** **Laudo como documento assinável.** O §10 põe o `RegulationLaudo` em `DocumentTypes`, mas não diz quando a assinatura abre, quem é o autor, nem a consulta: a tabela `signature_requests` tem CHECK de tipo e de consulta (`ck_signature_requests_document_type`, `ck_signature_requests_consultation`), e o `OpenRequest` lê `author_user_id` (`open_request.rb:21-25`). **Proposta:** abre na promoção, dentro da transação do `Finalize`/`AddAddendum` (como a consulta); autor = `requester_user_id`; `consultation_id` = o do pedido; fora da cadeia (`UNCHAINED`) (Tasks 3, 15).

D12. **[Resolvida pela Q11.] Diferente: em vez de passar o `kind`, `specialty_without_sigtap` só vale para alvo de especialidade.** **Efeitos no 26c além da modalidade.** O §9 só fala da modalidade 04. O plano do 26c fixa `RndsLimits::MODALITY = "09"` (Valor 3 dele) e calcula `specialty_without_sigtap` pelo `CareRegulation::Targets.resolve(type: "procedure")` (Q9, Task 8 dele). Sem o tipo, esse cálculo recusa todo alvo só-APAC/AIH/OCI, e todo pedido por laudo ficaria parado no envio. **Proposta:** `MODALITY_BY_KIND` e o `resolve` com `kind:` (Task 16).

D13. **[Resolvida pela Q6.] Diferente: a SIGTAP vai em ZIP completo no `vendor`, com SHA-256; o "1º dia útil" desconta os feriados nacionais de `config/regulation/national_holidays.yml`.** **Task 0, vendor e SIGTAP.** (a) O contrato diz que só os três insumos param a execução, mas manda conferir na Task 0 também `aih_instruments`, `oci_exit_reasons.yml` e o layout do e-SUS Captação de Filas, sem dizer se param. (b) "O ZIP completo… fica em `vendor/regulation/<data>/`": o do 26a guardou só os arquivos usados (180 KB, `vendor/regulation/2026-10-28/`), e o ZIP completo tem dezenas de MB. (c) O §2 dá "`sigtap_procedure_compatibilities` (principal, secundário, registro…)" com um registro só, e o `rl_procedimento_compativel` traz o do principal e o do secundário; o nome `sigtap_attribute_catalog` destoa das outras tabelas. (d) "1º dia útil" sem calendário de feriados. **Proposta:** (a) só a task dona para (a 14 sem o layout do e-SUS); (b) `SOURCE` com URL e SHA-256 do ZIP e de cada arquivo, e no vendor só os `_layout.txt` e as tabelas pequenas; (c) os dois registros e o nome `sigtap_attributes`; (d) segunda a sexta, sem feriados (Tasks 0, 1, 14).
