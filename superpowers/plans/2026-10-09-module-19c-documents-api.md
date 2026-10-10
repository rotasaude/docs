# Módulo 19 (19c) — Documentos clínicos (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO:** 19a e 19b em `origin/main` do api (`c9ccd88` ou posterior) e a tag **`clinical-v1.1.0`** publicada no `contracts` (esquema `clinical/clinical-document-v1.json` e o vetor em `clinical/examples/canonical/`). A Task 0 confere os dois antes de tudo. Este plano usa o código real do 19a/19b pelos nomes de hoje: `ClinicalRecord::{Gate,Access,Trail}`, `ClinicalRecordGate`, `Patients::ApplyProblemEvent` (modelo do `ApplyMedicationEvent`), `Consultations::{Authorization,Effective,Print}` (`Print.safe`), `Signatures::{Gate,OpenRequest,ToPaper,SignPending,RunBatch,SweepJob,DocumentTypes,Documents,Canonical,Jcs,Signing,Mode,Json,PrintReport,PdfFooter}`, `Platform::Features` (pré-requisito `feature:<key>`), `CityEncryption::CITY_KEYED_TARGETS`, `MaintenanceAudit`, `Maintenance::Mutations::BaseMutation#audited`, `CityPublicUrl.base`, `AttendanceAccess`, `MfaStepUp`, e os helpers de spec `ClinicalRecordHelpers` (`clinical_city!`, `doctor!`, `verified_citizen!`, `consulting_attendance!`, `started_consultation!`, `finalized_consultation!`, `capture_log`, `cid10_release!`, `sigtap_release!`) e `SignatureHelpers` (`signature_city!`, `signer_doctor!`, `stub_psc!`, `stub_signer!`, `sign_document!`, `linked_certificate!`, `signature_session!`).

**Goal:** O lado api dos documentos clínicos do 19c (F-19.16 a F-19.23, ADR 0033): interruptor `clinical_documents`; catálogo de medicamentos da plataforma importado do CATMAT (classe 6505) pela API aberta do Compras.gov, com analisador da descrição, listas da Anvisa (antimicrobianos e controlados) e release com diferença; REMUME e protocolos de enfermagem com versões; CNPJ no perfil da cidade; os quatro documentos (atestado, declaração de comparecimento, receita comum, requisição de exames) num cabeçalho comum imutável, com a matriz de quem emite e as validações; a lista de medicamentos em uso por eventos; renovação de receita; PDF por tipo com QR code; integração com a assinatura do 19b; página pública de conferência; cancelamento; declaração pela recepção; maintenance; LGPD; invariantes; semente; prova e rollout.

**Architecture:** Plataforma (migração `20261009600001` em `db/platform_migrate`): `medication_catalog_releases`/`medication_catalog_items` (um item estável por código CATMAT, atualizado a cada importação, nunca apagado) e `anvisa_list_releases`/`anvisa_list_substances`; o maintenance aciona a importação (cliente HTTP com paginação + analisador + diferença) e a das listas (arquivos versionados no repositório). Cidade (migração `20261009600001`): `clinical_documents` (cabeçalho comum, conteúdo JSON cifrado, só `issued → cancelled` e `digital → paper` por trigger), `prescription_items` (nascem na mesma transação do documento), `patient_medications` + `patient_medication_events` (estado só muda com evento da mesma transação, como os problemas), `city_medication_list` (REMUME), `nursing_protocols` → `nursing_protocol_versions` → `nursing_protocol_version_items`, `document_verification_lookups`, `city_profile.cnpj`; `signature_requests`/`signatures` aceitam `ClinicalDocument`. Um comando de emissão (`ClinicalDocuments::Issue`) valida o conteúdo por tipo, decide o modo **antes** da transação (como a finalização do 19b), grava documento + itens + eventos de medicamento e abre o pedido de assinatura na mesma transação. A página `/v/:token` é servida pelo api no host da cidade (como `/r/:token`).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, WebMock, Active Record Encryption (chave da cidade), Net::HTTP (cliente do CATMAT), Prawn (do 19a), `json_schemer` e `Signatures::Jcs` (do 19b), **uma gem nova: `rqrcode_core` (~> 2.1)** — ver Task 17.

**Spec:** `docs/superpowers/specs/2026-10-09-module-19c-clinical-documents-design.md` e `docs/adr/0033.md` (leia os dois antes de começar). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts.md`; o que o código real obrigou a precisar está em "Desvios e precisões" e, quando toca formato, em "Divergências propostas ao contrato" (fim do arquivo). Pesquisa: `docs/pesquisa/2026-10-09-documentos-clinicos-19c.md`. Contratos do 19a (§9) e do 19b (§13) prevalecem sobre os planos antigos.

## Desvios e precisões

1. **API do CATMAT conferida numa chamada real só de leitura (2026-10-09):** `GET https://dadosabertos.compras.gov.br/modulo-material/4_consultarItemMaterial?pagina=<n>&tamanhoPagina=500&codigoClasse=6505&statusItem=true` → 200 `{ "resultado": [ { "codigoItem", "codigoGrupo", "nomeGrupo", "codigoClasse", "nomeClasse", "codigoPdm", "nomePdm", "descricaoItem", "statusItem", "itemSustentavel", "codigo_ncm", "descricao_ncm", "aplica_margem_preferencia", "dataHoraAtualizacao" } ], "totalRegistros": 6778, "totalPaginas": 14, "paginasRestantes": 12 }`. `pagina` começa em 1; `tamanhoPagina=500` foi aceito (14 páginas, 6.778 itens, códigos únicos); página além da última volta `resultado: []` e `paginasRestantes: 0`. Sem autenticação.
2. **Analisador:** sobre os 6.778 itens reais, a regra desta entrega entende 4.098 (60%) e manda 2.680 a revisão (`hidden`, `parse_status: review`) com o motivo (`manipulated` 687, `unknown_field` ~1.700, os demais de força/forma). A revisão não tem mutation de correção nesta entrega (o contrato §8 só lista); curadoria é pendência de go-live (spec §12).
3. **Item do catálogo estável por código CATMAT.** Uma linha por `catmat_code`, atualizada pela importação (`changed`), marcada `removed` quando some da fonte (nunca apagada): a REMUME, os protocolos e as receitas guardam o id. A receita guarda também o código e a release da emissão (`catalog_release_id`) e o texto impresso — a release no JSON canônico é o id da release (uuid).
4. **Listas da Anvisa** não têm API aberta: vão como arquivos versionados no repositório (`config/medications/anvisa/`), transcritos da fonte oficial (Task 5), e a importação grava release + substâncias e reaplica as marcas a todo o catálogo. Casamento por nome normalizado (sem acento, maiúsculo), por palavra inteira no começo do ingrediente ("CIPROFLOXACINO" casa "CIPROFLOXACINO CLORIDRATO"); em associação, qualquer ingrediente marcado marca o item. O texto livre da receita também é varrido: controlado em texto livre é recusado.
5. **Modo `digital → paper`** é a única mudança de modo aceita, e só pela volta ao papel do 19b (spec §4) e sem assinatura gravada (trigger); `paper → digital` nunca. A volta ao papel de um documento clínico passa `issue_mode` a `paper` dentro de `Signatures::ToPaper`.
6. **Pedido pendente de documento cancelado sai da fila** com `returned_to_paper` e o motivo novo `document_cancelled` (sem mudar o modo): na hora, se ninguém está assinando (SKIP LOCKED); senão pelo job/lote que o tocar e pela varredura de 10 min. Nunca fica pendente para sempre.
7. **Impresso de documento digital ainda não assinado → 409 `awaiting_signature`** (o papel de um documento digital criaria duas vias válidas com o mesmo código; spec §4 "modo nunca convertido"). O profissional assina (sessão/lote) ou devolve ao papel (19b) e imprime. Documento cancelado → 409 `already_cancelled`.
8. **Declaração pela rota do atendimento** (`POST /attendance/attendances/:id/declarations`) sai sempre em papel (não tem consulta; `signature_requests.consultation_id` é obrigatório). A da consulta segue a regra do modo.
9. **Quem emite (matriz):** médico = CBO `2251*`/`2252*`/`2253*`; dentista = `2232*`; enfermeiro = `2235*`. Atestado: médico e dentista. Receita: médico, dentista, enfermeiro (por protocolo). Declaração e requisição de exames: a autora da consulta, qualquer CBO. Hoje a consulta do 19a recusa `2232` (`Consultations::Cbos`): o dentista está na matriz para o módulo 28 e só é provado na spec da matriz.
10. **Dose máxima do protocolo** é a quantidade máxima por item de receita: `max_dose: { quantity, unit }`. Receita de enfermeiro com unidade diferente da do protocolo → `above_protocol_max_dose` (não dá para provar que está dentro).
11. **Leitura de documento:** a autora do documento lê e imprime sempre (inclusive a recepção, a própria declaração); os demais pela regra do 19a sobre a consulta (`ClinicalRecord::Access.for_consultation`) ou sobre o paciente/atendimento (declaração da recepção). O `municipal_admin` lê o conteúdo por `GET /clinical_record/consultations/:id/documents` (step-up, linha em `clinical_record_administrative_reads`, trilha `administrative`), sem impresso.
12. **CNPJ** aceita o formato alfanumérico da IN RFB 2.229/2024 (12 caracteres `[0-9A-Z]` + 2 dígitos verificadores; o numérico é o caso particular), já em uso desde julho de 2026. A rota é `GET/PUT /clinical_documents/city_profile` (o `/admin/*` do api é só leitura por regra do projeto).
13. **Interruptor desligado:** **cancelar continua permitido** (protege o paciente); emitir, medicamentos (escrita) e busca do catálogo recusam com 403 `feature_disabled`; leituras, impresso, REMUME/protocolos/CNPJ (preparação da cidade) e a página pública seguem respondendo, atrás do `clinical_record`.
14. **Entrada do conteúdo** = a forma de saída do contrato §2, com os campos calculados pelo servidor ignorados (`label`, `printed_description`, `antimicrobial`, `position`, `copies`, `city_cnpj`, `unit_name`, `date`, `exams`); item da receita entra com `catalog_item: { id }` **ou** `free_text`; protocolo com `nursing_protocol: { version_id }`; CID com `cid10: { code }`.
15. **Produção não tem maintenance:** o catálogo e as listas entram em produção pelos rakes `bin/rails medications:import_anvisa` e `bin/rails medications:import_catmat` (Task 6), rodados na imagem publicada; o maintenance aciona os mesmos serviços em dev e staging. Duas importações ao mesmo tempo → `import_in_progress` (índice único na release `importing`).
16. **Confirmados com os planos do dashboard e do maintenance:** CNPJ em `GET/PUT /clinical_documents/city_profile`, alfanumérico (IN RFB 2.229/2024); 409 `not_draft` em `POST /attendance/consultations/:id/medications` com a consulta finalizada; 409 `awaiting_signature` no impresso de documento digital não assinado; a enfermagem lê `GET /clinical_documents/nursing_protocols`; a busca do catálogo aceita também o `municipal_admin`.

## Valores fixados por este plano (para o contrato e para o esquema do `contracts`)

1. **JSON canônico `rotasaude.clinical_document.v1`** = o esquema da tag `clinical-v1.1.0` (plano `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts-repo.md`; o **esquema prevalece**). Raiz `{ schema, city { ibge_code|null, name }, unit { cnes|null, name }, professional { name, cpf, cbo_code, council { name, state, registration_number } } (council obrigatório), patient { display_name, cpf, birth_date|null } (obrigatório), document { id, kind, issued_at, replaces_document_id|null }, content }`, e `content` = **a forma do contrato §2**:
   - `sick_note`: `{ type, days?, start_on?, companion_name?, companion_kinship?, companion_reason?, cid10?: { code, label }, cid_authorized, note|null }`;
   - `attendance_declaration`: `{ date, arrived_at?: "HH:MM", left_at?: "HH:MM", period?, unit_name, companion_name?, issuer_registration? }` (horas de parede da cidade);
   - `prescription`: `{ items: [ { position, catalog_item?: { id, catmat_code (inteiro), label, active_ingredient, strength, dosage_form }, free_text? (texto), printed_description, quantity, quantity_unit, route, dosage_instructions, duration_days?, continuous, antimicrobial, reason_problem_id? } ], catalog_release|null (uma vez por receita: o id da release do catálogo na emissão; null se todos os itens são texto livre), nursing_protocol?: { id, title, number, year, version_id }, city_cnpj?, antimicrobial, copies, valid_until? }`;
   - `exam_requisition`: `{ exams: [ { sigtap_code, competence, label, cid10_justification? } ], note|null }`.
   Opcional que não se aplica = chave **ausente**; `note` sempre presente (`null` vazio). A declaração emitida pela recepção não é assinada e não tem canônico. O gerador reproduz byte a byte o vetor `clinical/examples/canonical/{prescription-doctor,sick-note-leave}.jcs` (Task 18).
2. **Código curto:** 10 caracteres do alfabeto `23456789ABCDEFGHJKMNPQRSTUVWXYZ` (sem 0, 1, I, L, O), impresso `XXXXX-XXXXX`; a entrada aceita minúsculas, espaços e hífen. **Token:** `SecureRandom.urlsafe_base64(16)` (22 caracteres, 128 bits). **URL:** `<CityPublicUrl.base(cidade)>/v/<token>`.
3. **Limites da página pública por IP:** `POST /v/lookup` 10 por 10 min; `GET /v/:token` 60 por 10 min; `GET /v/:token/signed.pdf` 30 por 10 min → 429 `{ "error": "rate_limited" }`.
4. **Limites de entrada:** receita até 20 itens; `quantity` > 0 e ≤ 9999 com até 2 casas; `quantity_unit` 1–30; `dosage_instructions` 3–500; `duration_days` 1–365; `free_text` 3–200; atestado `days` 1–365, `start_on` entre hoje − 30 dias e hoje; `note` ≤ 500; acompanhante 3–120; parentesco 2–60; `issuer_registration` 1–30; motivo do cancelamento 10–500; `valid_until` entre hoje e hoje + 365; antimicrobiano → `copies: 2`, `valid_until` = data da emissão + 10 dias. Declaração do atendimento: até 30 dias depois do check-in.
5. **Importação:** falha (nada muda no catálogo ativo) se a fonte cair, se o total lido ≠ `totalRegistros`, ou se o lido for menor que 50% dos itens ativos atuais (`suspicious_drop`).

## Global Constraints

- Banco de cidade: migração em `db/city_migrate/20261009600001_add_clinical_documents.rb` (acima de `20261008500001`), dump à mão em `db/city_schema.rb` (`define(version: 2026_10_09_600001)`; a paridade compara o catálogo do banco migrado com o do dump, `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql` (blocos novos no fim, guardados por `to_regclass`; a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`). Irreversível (`down` levanta `ActiveRecord::IrreversibleMigration`). Rollout: publicar a imagem e rodar `city:migrate:all` dela **antes** de cortar tráfego; **nunca** migrar cidade fora do rake (a cidade trava em 503).
- Banco de plataforma: `db/platform_migrate/20261009600001_create_medication_catalog.rb`, `db/platform_schema.rb` (`define(version: 2026_10_09_600001)`), triggers em `db/platform_triggers.sql` (blocos novos guardados). **A migração de plataforma mexe no banco de teste de plataforma COMPARTILHADO** entre sessões: avise no balanço e às sessões "API" e do 19b antes de rodá-la em teste.
- As tabelas do 19a não mudam (nenhuma migração toca `consultations`, `consultation_*`, `patients`, `patient_problem*`). Do 19b, só os CHECKs de `signature_requests.document_type`, `signature_requests.reason_code` e `signatures.document_type` (acrescentam valores).
- Valores, exatamente: `kind` `sick_note` | `attendance_declaration` | `prescription` | `exam_requisition`; `issue_mode` `digital` | `paper`; `status` `issued` | `cancelled`; `sick_note.type` `leave` | `companion`; `companion_reason` `clt_473_x` | `clt_473_xi` | `clt_473_xii` | `other`; `period` `morning` | `afternoon` | `full_day`; via `oral` | `sublingual` | `topical` | `ophthalmic` | `otic` | `nasal` | `inhalation` | `vaginal` | `rectal` | `intramuscular` | `intravenous` | `subcutaneous` | `other`; medicamento `status` `active` | `suspended`, `origin` `prescription` | `external`; evento `added` | `suspended` | `reactivated` | `changed`; catálogo `parse_status` `ok` | `review`, item `status` `active` | `removed`, release `importing` | `active` | `superseded` | `failed`; lista Anvisa `antimicrobial` | `controlled`; conferência `via` `token` | `short_code`, `outcome` `found` | `not_found`; documento no banco `ClinicalDocument`, na API `clinical_document`.
- Erros do contrato, exatamente: 403 `not_author`, `cbo_not_allowed`, `missing_role`, `out_of_context`/`opening_required`, `feature_disabled` (`{ "error": "feature_disabled", "feature": "clinical_documents" }`); 401 `mfa_required`; 404 `not_found`; 409 `already_cancelled`, `already_active`, `already_exists`, `attendance_not_found_or_closed_long_ago`, `not_draft`; 422 `invalid_content` (com `field`), `invalid_item` (com `index` e `field`), `cid_requires_authorization`, `companion_cid_not_allowed`, `controlled_not_allowed` (com `index`), `not_in_nursing_protocol`, `above_protocol_max_dose` (com `index`), `free_text_not_allowed` (com `index`), `city_cnpj_missing`, `no_exam_requests`, `invalid_reason`, `invalid_cnpj`; 429 `rate_limited`; 503 `catalog_unavailable`. Escrita devolve o objeto puro; toda escrita por cookie exige `Content-Type: application/json`, inclusive DELETE (guarda existente `require_json_for_cookie_writes`).
- Cifra por cidade: `encrypts` (não determinístico) em `ClinicalDocument#content`, `#cancel_reason`; `PrescriptionItem#free_text`, `#printed_description`, `#dosage_instructions`; `PatientMedication#free_text`, `#label`, `#dosage_summary`; `PatientMedicationEvent#dosage_summary`. Todo `encrypts` novo entra em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`), e o trigger da tabela aceita a re-cifra sob `rota.reencrypting`.
- Eventos só com ids (contrato §10), declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`: `clinical_document.issued { document_id, kind, consultation_id | attendance_id, issue_mode }`, `clinical_document.cancelled { document_id, kind }`, `patient_medication.changed { patient_medication_id, kind, document_id | consultation_id | addendum_id }`. Plataforma: `Platform.audit("medication_catalog.imported", release_id:, added:, changed:, removed:)` e `Platform.audit("anvisa_lists.imported", release_id:, antimicrobials:, controlled:)`; auditoria do maintenance `maintenance.medication_catalog.imported` e `maintenance.anvisa_lists.imported` — **nome novo de `Platform.audit` entra em `R18_PLATFORM_EVENT_NAMES`** (`spec/events/platform_event_payload_guard_spec.rb`) e, os de manutenção, também em `MaintenanceAudit::NAMES` + o `case` de despacho. `DomainEvents.publish(` e `Platform.audit(` são multilinha: confira cada chamada inteira.
- Texto clínico, CID, medicamento, nome, CPF, código curto, ano de nascimento e o motivo nunca em log (`filter_parameters`), evento, Analytics, mensagem de erro ou `inspect`.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`); nenhum job novo.
- Specs de request com `type: :request` explícito; arquivo novo em `spec/support/` entra com `require_relative` em `spec/rails_helper.rb`. Leitura de plataforma (catálogo, listas, interruptor) **fora** da transação da cidade (como `Finalize` decide `Signatures::Gate.usable?` antes).
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..33)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` sempre com caminhos explícitos (nunca `-A`). Merge, push, tag e cards só com autorização explícita do usuário.

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard e do maintenance do 19c. Ordem de merge: `contracts` (tag `clinical-v1.1.0`) → **api** → dashboard → maintenance.
- Worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod19c -b feat/mod-19c-documents origin/main
  cp apps/api/config/master.key apps/api/.claude/mod19c/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod19c`. **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod19c -w /rails/.claude/mod19c api bundle exec rspec <arquivos>
  ```

  e `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod19c <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod19c`, `lib/test_database_suffix.rb`), criados uma vez no começo e recriados depois da migração de cidade (Task 8):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod19c" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod19c"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod19c -w /rails/.claude/mod19c api bin/rails city:test_databases
  ```

- Migração de plataforma em teste (Task 2; banco compartilhado — avise antes): `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19c api bin/rails db:migrate:platform`.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.
- Chamada real ao CATMAT: só a Task 4 (Step 1), só leitura, uma página.
- Prova no navegador: o api do worktree sobe na porta **3038** (Task 25), com `CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5188` (o Vite do dashboard do 19c). Os prefixos `/clinical_documents` e `/v` precisam de entrada no proxy de dev do dashboard (plano do dashboard).

### Arquivos em comum com o 19a/19b (e como rebasear)

Se a main receber correções, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` e `db/platform_schema.rb` (`define(version:)` com o maior número), `db/city_triggers.sql` e `db/platform_triggers.sql` (blocos novos no fim), `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`, `app/services/city_encryption.rb`, `spec/adr_pointers_spec.rb` (fica `1..33`), `config/routes.rb`, `app/services/platform/features.rb`, `app/services/signatures/{document_types,documents,canonical,signing,json,print_report}.rb`, `app/commands/signatures/{to_paper,sign_pending,run_batch}.rb`, `app/jobs/signatures/sweep_job.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `app/graphql/maintenance/**`, `spec/architecture/maintenance_schema_spec.rb`, `Gemfile`/`Gemfile.lock`.

## Review Focus

1. **A API do CATMAT falha no meio** (página 9 de 14 dá 503, volta `resultado: []` antes do fim, ou o total muda entre páginas): a importação falha inteira, a release fica `failed`, o catálogo ativo e a REMUME ficam como estavam, e nunca milhares de itens viram `removed`. Testes: Task 6 ("fonte cai no meio", "fonte encolhe").
2. **Dois cliques em "Emitir receita"** com item de uso contínuo (ou a mesma receita renovada duas vezes): dois documentos podem nascer, mas a lista de medicamentos fica com **um** ativo por item — o segundo vira `changed` ou nada, nunca duplicado e nunca 500. Testes: Task 13 ("receita repetida não duplica").
3. **O código curto digitado por gente:** minúsculas, hífen, espaços, ano com 2 dígitos, código certo com ano errado, código de outra cidade → sempre o mesmo 404 `not_found` (sem dizer o que errou); o 11º palpite do mesmo IP em 10 min → 429; nada da pessoa no log. Testes: Task 21 ("código digitado").
4. **Cancelar enquanto o job assina** (ou logo depois de emitir com a sessão aberta): nunca 500, nunca um documento cancelado preso em "Pendentes de assinatura" — sai na hora, pelo job/lote que o tocar ou pela varredura; se a assinatura terminou antes, o documento fica assinado **e** cancelado, e a página mostra "cancelado". Testes: Task 20 ("cancelar com o pedido travado").
5. **Texto que a fonte do PDF não tem e receita grande:** emoji, "≥", `\r\n`, 20 itens com posologia de 500 caracteres, nome social com acento — o PDF sai (com `?` no que falta), o QR e o código curto aparecem em toda via, e o antimicrobiano sai com as duas vias. Testes: Task 17 ("receita cheia").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferir main, `contracts`, worktree | 0 |
| `app/services/platform/features.rb`, `app/services/clinical_documents/gate.rb`, `app/controllers/concerns/clinical_documents_gate.rb`, `spec/adr_pointers_spec.rb` | interruptor | 1 |
| `db/platform_migrate/20261009600001_create_medication_catalog.rb`, `db/platform_schema.rb`, `db/platform_triggers.sql`, `app/models/{medication_catalog_release,medication_catalog_item,anvisa_list_release,anvisa_list_substance}.rb` | tabelas de plataforma | 2 |
| `app/services/medications/catmat/description_parser.rb`, `spec/fixtures/medications/catmat/samples.yml` | analisador da descrição | 3 |
| `app/services/medications/catmat/{client,fixture_client}.rb`, `spec/fixtures/medications/catmat/page-{1,2}.json` | cliente com paginação | 4 |
| `config/medications/anvisa/{antimicrobials.csv,controlled.csv,SOURCES.yml}`, `app/services/medications/{substances,anvisa_import}.rb`, `spec/fixtures/medications/anvisa/*` | listas Anvisa e marcas | 5 |
| `app/services/medications/catalog_import.rb`, `lib/tasks/medications.rake`, `spec/support/medication_helpers.rb` | importação com diferença | 6 |
| `app/graphql/maintenance/types/medication_catalog*_type.rb`, `app/graphql/maintenance/mutations/import_{medication_catalog,anvisa_lists}.rb`, `app/events/maintenance_audit.rb`, analisador `HumanOnly`, guardas | maintenance | 7 |
| `db/city_migrate/20261009600001_add_clinical_documents.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, modelos da cidade, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `app/models/signature_request.rb`, `spec/support/clinical_document_helpers.rb` | dados da cidade | 8 |
| `app/services/cnpj.rb`, `app/controllers/clinical_documents/{base,city_profiles}_controller.rb` | CNPJ | 9 |
| `app/controllers/clinical_documents/remume_controller.rb`, `app/services/medications/search.rb`, `app/controllers/medication_searches_controller.rb`, `app/services/clinical_documents/json.rb` (catálogo) | REMUME e busca | 10 |
| `app/commands/nursing_protocols/save.rb`, `app/controllers/clinical_documents/nursing_protocols_controller.rb` | protocolos de enfermagem | 11 |
| `app/commands/patients/apply_medication_event.rb` | caminho único da lista de medicamentos | 12 |
| `app/services/clinical_documents/{issuers,codes}.rb`, `app/services/clinical_documents/content/{sick_note,declaration,exam_requisition}.rb` | matriz e conteúdo (três tipos) | 13 |
| `app/services/clinical_documents/content/prescription.rb` | validação da receita | 14 |
| `app/commands/clinical_documents/issue.rb`, `app/services/clinical_documents/json.rb` (documento) | emissão | 15 |
| `app/controllers/clinical_documents_controller.rb`, `app/controllers/patient_medications_controller.rb`, `app/services/clinical_documents/{read,renewal}.rb` | rotas de documentos e medicamentos | 16 |
| `Gemfile`, `app/services/clinical_documents/{pdf,qr}.rb` | PDF por tipo com QR; impresso | 17 |
| `config/clinical/clinical-document-v1.json`, `spec/fixtures/clinical/**`, `app/services/signatures/{document_types,canonical,documents,signing,json,print_report}.rb`, `app/commands/signatures/to_paper.rb` | integração com o 19b | 18 |
| `app/commands/signatures/{sign_pending,run_batch}.rb`, `app/jobs/signatures/sweep_job.rb`, `app/services/signatures/cancelled_documents.rb` | o pedido de documento cancelado sai da fila | 19 |
| `app/commands/clinical_documents/cancel.rb` | cancelamento | 20 |
| `app/controllers/document_verifications_controller.rb`, `app/services/clinical_documents/{verification,page}.rb` | página pública | 21 |
| `app/controllers/clinical_record_documents_controller.rb`, `app/commands/citizens/request_erasure.rb` | leitura administrativa e LGPD | 22 |
| `spec/invariants/clinical_documents_invariants_spec.rb` | invariantes do ADR 0033 | 23 |
| `lib/clinical_documents_crew.rb`, `db/seeds/medications/catmat-dev.json`, `db/seeds.rb` | semente de dev | 24 |
| — | revisão final, suíte, porta 3038, prova, rollout | 25 |

---

## Fatia 0 — Pré-requisitos e interruptor (F-19.16)

### Task 0: Conferir a main do api, a tag do `contracts` e criar o worktree

Esta task não escreve código.

- [ ] **Step 1: Confira os nomes do 19a/19b em `origin/main`**

```bash
/opt/homebrew/bin/git -C apps/api fetch origin
/opt/homebrew/bin/git -C apps/api merge-base --is-ancestor c9ccd88 origin/main && echo "ok c9ccd88"
for f in app/commands/patients/apply_problem_event.rb app/services/clinical_record/access.rb app/services/clinical_record/trail.rb \
         app/services/consultations/print.rb app/services/consultations/effective.rb app/services/signatures/document_types.rb \
         app/services/signatures/canonical.rb app/services/signatures/documents.rb app/services/signatures/signing.rb \
         app/services/signatures/json.rb app/services/signatures/print_report.rb app/commands/signatures/open_request.rb \
         app/commands/signatures/to_paper.rb app/commands/signatures/sign_pending.rb app/commands/signatures/run_batch.rb \
         app/jobs/signatures/sweep_job.rb app/events/maintenance_audit.rb spec/support/clinical_record_helpers.rb \
         spec/support/signature_helpers.rb db/city_migrate/20261008500001_add_digital_signatures.rb \
         db/platform_migrate/20261008500001_create_signature_provider_checks.rb; do
  /opt/homebrew/bin/git -C apps/api cat-file -e "origin/main:$f" && echo "ok $f" || echo "FALTA $f"
done
/opt/homebrew/bin/git -C apps/api grep -n 'when /\\Afeature:(\\w+)\\z/' origin/main -- app/services/platform/features.rb
/opt/homebrew/bin/git -C apps/api grep -n "def call(document, now: Time.current, usable: :load)" origin/main -- app/commands/signatures/open_request.rb
/opt/homebrew/bin/git -C apps/api grep -n "MODELS = " origin/main -- app/services/signatures/document_types.rb
/opt/homebrew/bin/git -C apps/api grep -n "def safe(text)" origin/main -- app/services/consultations/print.rb
```
Expected: `ok c9ccd88`; todos `ok`; o pré-requisito `feature:` no catálogo; `OpenRequest.call(document, now:, usable:)`; `MODELS = { "Consultation" => …, "ConsultationAddendum" => … }`; `Print.safe`. **Pare** e reporte ao coordenador se algo faltar ou mudou de nome.

- [ ] **Step 2: Confira o `contracts`**

```bash
/opt/homebrew/bin/git -C contracts fetch --tags origin
/opt/homebrew/bin/git -C contracts tag -l clinical-v1.1.0
/opt/homebrew/bin/git -C contracts ls-tree -r --name-only clinical-v1.1.0 clinical/
/opt/homebrew/bin/git -C contracts show clinical-v1.1.0:clinical/examples/canonical/SHA256SUMS
```
Expected: a tag existe; `clinical/clinical-document-v1.json`, exemplos do documento (um diretório `clinical/examples/<…>/` com `manifest.json` e os `.json` válidos) e os `.jcs` novos listados no `SHA256SUMS` (além dos dois do 19b). Anote os **nomes reais** do diretório de exemplos e dos `.jcs` (a Task 18 os copia). Leia o esquema e compare com "Valores fixados" 1: diferença de **nome** de campo — anote; a Task 18 segue o esquema. Diferença de **forma** (um campo obrigatório que o api não tem como preencher) — **pare** e reporte. Sem a tag, **pare**.

- [ ] **Step 3: Crie o worktree e os bancos de teste da branch** (Ambiente de execução) e confira as últimas migrações:

Run: `ls apps/api/.claude/mod19c/db/city_migrate | tail -1; ls apps/api/.claude/mod19c/db/platform_migrate | tail -1`
Expected: `20261008500001_add_digital_signatures.rb` e `20261008500001_create_signature_provider_checks.rb`. Se houver mais nova que `20261009600001`, use um número maior nas Tasks 2 e 8 (e no `define(version:)`).

Run: `rspec spec/services/platform spec/adr_pointers_spec.rb`
Expected: PASS (a base está verde antes de mexer).

---

### Task 1: Interruptor `clinical_documents` (requer `feature:clinical_record`)

**Files:**
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`
- Create: `app/services/clinical_documents/gate.rb`, `app/controllers/concerns/clinical_documents_gate.rb`
- Test: `spec/services/platform/clinical_documents_feature_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features` (`feature:<key>` → `"<key>_disabled"`), `clinical_city!`, `ledi_maintainer!`.
- Produces: `ClinicalDocuments::Gate::KEY == "clinical_documents"`, `ClinicalDocuments::Gate.usable?(city) -> bool` (nil ou sem id → false); concern `ClinicalDocumentsGate#require_clinical_documents!` (403 `{ error: "feature_disabled", feature: "clinical_documents" }`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/platform/clinical_documents_feature_spec.rb
require "rails_helper"

# ADR 0033 (spec §3): documentos clínicos atrás do interruptor
# clinical_documents, utilizável só com o prontuário utilizável.
RSpec.describe "Interruptor clinical_documents" do
  let(:city) { clinical_city! }

  def set(key, enabled) = Platform::Features.set!(city: city, key: key, enabled: enabled, maintainer: ledi_maintainer!)

  it "está no catálogo e exige o prontuário utilizável" do
    expect(Platform::Features::KEYS).to include("clinical_documents")
    set("clinical_documents", true)
    expect(Platform::Features.missing(city, "clinical_documents")).to eq([])
    expect(ClinicalDocuments::Gate.usable?(city)).to be(true)

    set("clinical_record", false)
    expect(Platform::Features.missing(city, "clinical_documents")).to eq([ "clinical_record_disabled" ])
    expect(ClinicalDocuments::Gate.usable?(city)).to be(false)
  end

  it "desligado ou sem cidade não é utilizável" do
    expect(ClinicalDocuments::Gate.usable?(city)).to be(false)
    expect(ClinicalDocuments::Gate.usable?(nil)).to be(false)
  end

  it "não depende da assinatura digital" do
    set("clinical_documents", true)
    expect(Platform::Features.enabled?(city, "digital_signature")).to be(false)
    expect(ClinicalDocuments::Gate.usable?(city)).to be(true)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/platform/clinical_documents_feature_spec.rb`
Expected: FAIL (`interruptor fora do catálogo: clinical_documents` / `uninitialized constant ClinicalDocuments`).

- [ ] **Step 3: Implemente**

Em `app/services/platform/features.rb`, acrescente ao `CATALOG`, depois de `signature_psc_mock`:

```ruby
      # ADR 0033: documentos clínicos da consulta (19c); só com o prontuário
      # utilizável. Desligado: nada novo é emitido; os documentos existentes
      # seguem legíveis e a página de conferência segue respondendo.
      Entry.new(key: "clinical_documents",
                description: "Documentos clínicos (atestado, declaração, receita, requisição) e lista de medicamentos",
                requires: %w[feature:clinical_record])
```

```ruby
# app/services/clinical_documents/gate.rb
# Os documentos clínicos só são EMITIDOS com o interruptor clinical_documents
# LIGADO e UTILIZÁVEL (ADR 0033; requer clinical_record utilizável). Relido
# da plataforma a cada chamada. Leitura e conferência não dependem dele.
module ClinicalDocuments
  module Gate
    KEY = "clinical_documents".freeze

    module_function

    def usable?(city)
      return false if city.nil? || city.id.nil?

      Platform::Features.usable?(city, KEY)
    end
  end
end
```

```ruby
# app/controllers/concerns/clinical_documents_gate.rb
# Escritas dos documentos clínicos (ADR 0033; contrato): interruptor desligado
# ou sem o prontuário utilizável → 403 feature_disabled.
module ClinicalDocumentsGate
  extend ActiveSupport::Concern

  private

  def require_clinical_documents!
    return if ClinicalDocuments::Gate.usable?(Current.city)

    render json: { error: "feature_disabled", feature: ClinicalDocuments::Gate::KEY }, status: :forbidden
  end
end
```

Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..33).freeze`.

- [ ] **Step 4: Rode e veja passar (e as specs do catálogo e do maintenance)**

Run: `rspec spec/services/platform spec/requests/maintenance/city_features_spec.rb spec/adr_pointers_spec.rb`
Expected: PASS. Spec que fixa a lista de chaves do catálogo → acrescente `clinical_documents` (mecanismo genérico, contrato §8).

- [ ] **Step 5: Commit**

```bash
git add app/services/platform/features.rb app/services/clinical_documents/gate.rb app/controllers/concerns/clinical_documents_gate.rb spec/adr_pointers_spec.rb spec/services/platform/clinical_documents_feature_spec.rb
git commit -m "feat: add the clinical_documents switch that requires the clinical record

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
(Inclua no `add` a spec do maintenance se ela mudou no Step 4.)

---

## Fatia 1 — Catálogo de medicamentos da plataforma (F-19.16)

### Task 2: Tabelas de plataforma do catálogo e das listas Anvisa

**Files:**
- Create: `db/platform_migrate/20261009600001_create_medication_catalog.rb`, `app/models/medication_catalog_release.rb`, `app/models/medication_catalog_item.rb`, `app/models/anvisa_list_release.rb`, `app/models/anvisa_list_substance.rb`
- Modify: `db/platform_schema.rb`, `db/platform_triggers.sql`
- Test: `spec/models/medication_catalog_spec.rb`

**Interfaces:**
- Consumes: `PlatformRecord`.
- Produces:
  - `MedicationCatalogRelease` (`STATUSES`, `scope :active`, `.current -> release|nil`, colunas `source`, `imported_by`, `imported_at`, `status`, `items_count`, `hidden_count`, `added_count`, `changed_count`, `removed_count`, `activated_at`).
  - `MedicationCatalogItem` (`catmat_code` Integer único, `pdm_code`, `source_description`, `source_updated_at`, `active_ingredient`, `strength`, `dosage_form`, `unit`, `antimicrobial`, `controlled`, `controlled_list`, `obm_code` (sempre nil), `hidden`, `parse_status`, `review_reason`, `status`, `search_text`, `first_release_id`, `last_release_id`); `scope :prescribable` (`status: "active", hidden: false`); `#label -> String`; `#ingredients -> Array<String>`.
  - `AnvisaListRelease` (`.current`, `antimicrobials_count`, `controlled_count`, `source_sha256`, `sources` jsonb) e `AnvisaListSubstance` (`release_id`, `kind`, `substance`, `substance_key`, `controlled_list`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/medication_catalog_spec.rb
require "rails_helper"

# Catálogo da plataforma (ADR 0033; spec §3): um item estável por código
# CATMAT, nunca apagado (receitas, REMUME e protocolos guardam o id); uma
# release ativa; listas Anvisa por release.
RSpec.describe "Catálogo de medicamentos (plataforma)" do
  def release!(status: "active")
    MedicationCatalogRelease.create!(source: "catmat_6505", imported_by: "rspec", imported_at: Time.current, status: status,
                                     activated_at: (Time.current if status == "active"))
  end

  def item!(release, code: 268_856, **over)
    MedicationCatalogItem.create!({ catmat_code: code, pdm_code: 8922, source_description: "LOSARTANA POTÁSSICA, DOSAGEM: 50 MG",
                                    active_ingredient: "LOSARTANA POTÁSSICA", strength: "50 MG", unit: "mg",
                                    parse_status: "ok", hidden: false, status: "active", search_text: "LOSARTANA POTASSICA 50 MG",
                                    first_release_id: release.id, last_release_id: release.id }.merge(over))
  end

  it "rótulo, ingredientes e o escopo prescritível" do
    release = release!
    item = item!(release, dosage_form: "comprimido")
    expect(item.label).to eq("LOSARTANA POTÁSSICA 50 MG comprimido")
    expect(item!(release, code: 287_471, active_ingredient: "LOSARTANA POTÁSSICA + HIDROCLOROTIAZIDA").ingredients)
      .to eq([ "LOSARTANA POTÁSSICA", "HIDROCLOROTIAZIDA" ])
    item!(release, code: 384_258, hidden: true, parse_status: "review", review_reason: "manipulated", strength: nil)
    item!(release, code: 999_999, status: "removed")
    expect(MedicationCatalogItem.prescribable.pluck(:catmat_code)).to contain_exactly(268_856, 287_471)
    expect(MedicationCatalogRelease.current).to eq(release)
  end

  it "o banco recusa apagar item, código repetido, obm_code preenchido e duas releases ativas" do
    release = release!
    item = item!(release)
    expect { item.destroy }.to raise_error(ActiveRecord::StatementInvalid, /never deleted/)
    expect { item!(release) }.to raise_error(ActiveRecord::RecordNotUnique)
    expect { item.update_columns(obm_code: "123") }.to raise_error(ActiveRecord::StatementInvalid, /ck_medication_catalog_items_obm_reserved/)
    expect { release! }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "release ativa ou substituída não volta; com falha não ativa" do
    failed = release!(status: "failed")
    expect { failed.update!(status: "active", activated_at: Time.current) }.to raise_error(ActiveRecord::StatementInvalid, /release/)
    active = release!
    expect { active.update!(status: "importing") }.to raise_error(ActiveRecord::StatementInvalid, /release/)
    active.update!(status: "superseded")
    expect { active.update!(status: "active") }.to raise_error(ActiveRecord::StatementInvalid, /release/)
  end

  it "listas Anvisa: uma release ativa, substâncias da release em importação" do
    list = AnvisaListRelease.create!(imported_by: "rspec", imported_at: Time.current, status: "importing",
                                     source_sha256: "a" * 64, sources: { "antimicrobials" => "RDC 471/2021" })
    AnvisaListSubstance.create!(release: list, kind: "controlled", substance: "DIAZEPAM", substance_key: "DIAZEPAM",
                                controlled_list: "B1")
    list.update!(status: "active", activated_at: Time.current, antimicrobials_count: 0, controlled_count: 1)
    expect(AnvisaListRelease.current).to eq(list)
    expect { AnvisaListSubstance.create!(release: list, kind: "antimicrobial", substance: "X", substance_key: "X") }
      .to raise_error(ActiveRecord::StatementInvalid, /being imported/)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/models/medication_catalog_spec.rb`
Expected: FAIL (`uninitialized constant MedicationCatalogRelease`).

- [ ] **Step 3: A migração de plataforma**

```ruby
# db/platform_migrate/20261009600001_create_medication_catalog.rb
# Catálogo de medicamentos da plataforma (ADR 0033; spec §3): o CATMAT classe
# 6505 importado pelo maintenance, com o analisador e as marcas das listas da
# Anvisa (antimicrobianos RDC 471, controlados Portaria 344). Um item por
# código CATMAT, estável (a REMUME, os protocolos e as receitas das cidades
# guardam o id), atualizado a cada release e nunca apagado. Dado público, sem
# dado de pessoa. Triggers: db/platform_triggers.sql.
class CreateMedicationCatalog < ActiveRecord::Migration[8.1]
  def self.text_in(column, values)
    "#{column}::text = ANY (ARRAY[#{values.map { |v| "'#{v}'::text" }.join(', ')}])"
  end

  def up
    create_table :medication_catalog_releases, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string :source, null: false, default: "catmat_6505"
      t.string :imported_by, null: false, limit: 80
      t.datetime :imported_at, null: false
      t.string :status, null: false, default: "importing"
      t.integer :items_count, null: false, default: 0
      t.integer :hidden_count, null: false, default: 0
      t.integer :added_count, null: false, default: 0
      t.integer :changed_count, null: false, default: 0
      t.integer :removed_count, null: false, default: 0
      t.datetime :activated_at
      t.timestamps
      t.index :status, unique: true, where: "((status)::text = 'active'::text)", name: "idx_medication_catalog_releases_one_active"
      # Uma importação por vez (import_in_progress); a presa há mais de 1 h vira failed antes da próxima.
      t.index :status, unique: true, where: "((status)::text = 'importing'::text)", name: "idx_medication_catalog_releases_one_importing"
      t.check_constraint text_in("status", %w[importing active superseded failed]), name: "ck_medication_catalog_releases_status"
      t.check_constraint "(status::text = 'active'::text) <= (activated_at IS NOT NULL)", name: "ck_medication_catalog_releases_activation"
    end

    create_table :medication_catalog_items, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.integer :catmat_code, null: false
      t.integer :pdm_code
      t.text :source_description, null: false
      t.datetime :source_updated_at
      t.string :active_ingredient, null: false
      t.string :strength
      t.string :dosage_form
      t.string :unit, limit: 20
      t.boolean :antimicrobial, null: false, default: false
      t.boolean :controlled, null: false, default: false
      t.string :controlled_list, limit: 3
      t.string :obm_code
      t.boolean :hidden, null: false, default: false
      t.string :parse_status, null: false
      t.string :review_reason
      t.string :status, null: false, default: "active"
      t.text :search_text, null: false
      t.uuid :first_release_id, null: false
      t.uuid :last_release_id, null: false
      t.timestamps
      t.index :catmat_code, unique: true
      t.index %i[status hidden], name: "idx_medication_catalog_items_prescribable"
      t.index :parse_status
      t.check_constraint text_in("parse_status", %w[ok review]), name: "ck_medication_catalog_items_parse_status"
      t.check_constraint text_in("status", %w[active removed]), name: "ck_medication_catalog_items_status"
      t.check_constraint "parse_status::text = 'ok'::text OR hidden", name: "ck_medication_catalog_items_review_hidden"
      t.check_constraint "parse_status::text = 'review'::text OR strength IS NOT NULL", name: "ck_medication_catalog_items_strength"
      t.check_constraint "controlled = (controlled_list IS NOT NULL)", name: "ck_medication_catalog_items_controlled_list"
      t.check_constraint "obm_code IS NULL", name: "ck_medication_catalog_items_obm_reserved"
      t.check_constraint "catmat_code > 0", name: "ck_medication_catalog_items_catmat_code"
    end
    add_foreign_key :medication_catalog_items, :medication_catalog_releases, column: :first_release_id
    add_foreign_key :medication_catalog_items, :medication_catalog_releases, column: :last_release_id

    create_table :anvisa_list_releases, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string :imported_by, null: false, limit: 80
      t.datetime :imported_at, null: false
      t.string :status, null: false, default: "importing"
      t.string :source_sha256, null: false, limit: 64
      t.jsonb :sources, null: false, default: {}
      t.integer :antimicrobials_count, null: false, default: 0
      t.integer :controlled_count, null: false, default: 0
      t.datetime :activated_at
      t.timestamps
      t.index :status, unique: true, where: "((status)::text = 'active'::text)", name: "idx_anvisa_list_releases_one_active"
      t.index :status, unique: true, where: "((status)::text = 'importing'::text)", name: "idx_anvisa_list_releases_one_importing"
      t.check_constraint text_in("status", %w[importing active superseded failed]), name: "ck_anvisa_list_releases_status"
    end

    create_table :anvisa_list_substances do |t|
      t.uuid :release_id, null: false
      t.string :kind, null: false
      t.string :substance, null: false
      t.string :substance_key, null: false
      t.string :controlled_list, limit: 3
      t.index %i[release_id kind substance_key], unique: true, name: "idx_anvisa_list_substances_unique"
      t.check_constraint text_in("kind", %w[antimicrobial controlled]), name: "ck_anvisa_list_substances_kind"
      t.check_constraint "(kind::text = 'controlled'::text) = (controlled_list IS NOT NULL)", name: "ck_anvisa_list_substances_list"
      t.check_constraint "controlled_list IS NULL OR controlled_list::text ~ '^[A-F][0-9]?$'::text", name: "ck_anvisa_list_substances_list_code"
    end
    add_foreign_key :anvisa_list_substances, :anvisa_list_releases, column: :release_id

    execute File.read(Rails.root.join("db/platform_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- [ ] **Step 4: Os triggers (fim de `db/platform_triggers.sql`)**

```sql
-- Catálogo de medicamentos (ADR 0033; spec §3): item nunca é apagado (receitas,
-- REMUME e protocolos das cidades guardam o id); release ativa ou substituída
-- nunca volta, release com falha nunca ativa; as substâncias das listas Anvisa
-- só nascem numa release em importação e nunca mudam.
CREATE OR REPLACE FUNCTION medication_catalog_items_no_delete() RETURNS trigger AS $fn$
BEGIN
  RAISE EXCEPTION 'medication_catalog_items are never deleted';
END;
$fn$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION medication_release_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION '% release is never deleted', TG_TABLE_NAME;
  END IF;
  IF OLD.status IS DISTINCT FROM NEW.status AND NOT (
       (OLD.status = 'importing' AND NEW.status IN ('active', 'failed'))
    OR (OLD.status = 'active' AND NEW.status = 'superseded')) THEN
    RAISE EXCEPTION '% release: % -> % refused', TG_TABLE_NAME, OLD.status, NEW.status;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION anvisa_list_substances_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'INSERT' AND EXISTS (SELECT 1 FROM anvisa_list_releases r WHERE r.id = NEW.release_id AND r.status = 'importing') THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'anvisa_list_substances accept rows only for a release being imported, and never change';
END;
$fn$ LANGUAGE plpgsql;

DO $$
BEGIN
  IF to_regclass('public.medication_catalog_items') IS NOT NULL THEN
    DROP TRIGGER IF EXISTS medication_catalog_items_no_delete ON medication_catalog_items;
    CREATE TRIGGER medication_catalog_items_no_delete BEFORE DELETE ON medication_catalog_items
      FOR EACH ROW EXECUTE FUNCTION medication_catalog_items_no_delete();
    DROP TRIGGER IF EXISTS medication_catalog_releases_guard ON medication_catalog_releases;
    CREATE TRIGGER medication_catalog_releases_guard BEFORE UPDATE OR DELETE ON medication_catalog_releases
      FOR EACH ROW EXECUTE FUNCTION medication_release_guard();
    DROP TRIGGER IF EXISTS anvisa_list_releases_guard ON anvisa_list_releases;
    CREATE TRIGGER anvisa_list_releases_guard BEFORE UPDATE OR DELETE ON anvisa_list_releases
      FOR EACH ROW EXECUTE FUNCTION medication_release_guard();
    DROP TRIGGER IF EXISTS anvisa_list_substances_guard ON anvisa_list_substances;
    CREATE TRIGGER anvisa_list_substances_guard BEFORE INSERT OR UPDATE OR DELETE ON anvisa_list_substances
      FOR EACH ROW EXECUTE FUNCTION anvisa_list_substances_guard();
  END IF;
END $$;
```

(O TRUNCATE dos itens não é barrado de propósito: a suíte limpa a plataforma por transação; um TRUNCATE em produção é do dono das tabelas, como diz o cabeçalho do arquivo.)

- [ ] **Step 5: O dump em `db/platform_schema.rb`**

Troque o `define(version:)` por `2026_10_09_600001` e acrescente, em ordem alfabética entre as tabelas existentes:

```ruby
  create_table "anvisa_list_releases", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "activated_at"
    t.integer "antimicrobials_count", default: 0, null: false
    t.integer "controlled_count", default: 0, null: false
    t.datetime "created_at", null: false
    t.datetime "imported_at", null: false
    t.string "imported_by", limit: 80, null: false
    t.string "source_sha256", limit: 64, null: false
    t.jsonb "sources", default: {}, null: false
    t.string "status", default: "importing", null: false
    t.datetime "updated_at", null: false
    t.index ["status"], name: "idx_anvisa_list_releases_one_active", unique: true, where: "((status)::text = 'active'::text)"
    t.index ["status"], name: "idx_anvisa_list_releases_one_importing", unique: true, where: "((status)::text = 'importing'::text)"
    t.check_constraint "status::text = ANY (ARRAY['importing'::text, 'active'::text, 'superseded'::text, 'failed'::text])", name: "ck_anvisa_list_releases_status"
  end

  create_table "anvisa_list_substances", force: :cascade do |t|
    t.string "controlled_list", limit: 3
    t.string "kind", null: false
    t.uuid "release_id", null: false
    t.string "substance", null: false
    t.string "substance_key", null: false
    t.index ["release_id", "kind", "substance_key"], name: "idx_anvisa_list_substances_unique", unique: true
    t.check_constraint "(kind::text = 'controlled'::text) = (controlled_list IS NOT NULL)", name: "ck_anvisa_list_substances_list"
    t.check_constraint "controlled_list IS NULL OR controlled_list::text ~ '^[A-F][0-9]?$'::text", name: "ck_anvisa_list_substances_list_code"
    t.check_constraint "kind::text = ANY (ARRAY['antimicrobial'::text, 'controlled'::text])", name: "ck_anvisa_list_substances_kind"
  end

  create_table "medication_catalog_items", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "active_ingredient", null: false
    t.boolean "antimicrobial", default: false, null: false
    t.integer "catmat_code", null: false
    t.boolean "controlled", default: false, null: false
    t.string "controlled_list", limit: 3
    t.datetime "created_at", null: false
    t.string "dosage_form"
    t.uuid "first_release_id", null: false
    t.boolean "hidden", default: false, null: false
    t.uuid "last_release_id", null: false
    t.string "obm_code"
    t.string "parse_status", null: false
    t.integer "pdm_code"
    t.string "review_reason"
    t.text "search_text", null: false
    t.text "source_description", null: false
    t.datetime "source_updated_at"
    t.string "status", default: "active", null: false
    t.string "strength"
    t.string "unit", limit: 20
    t.datetime "updated_at", null: false
    t.index ["catmat_code"], name: "index_medication_catalog_items_on_catmat_code", unique: true
    t.index ["parse_status"], name: "index_medication_catalog_items_on_parse_status"
    t.index ["status", "hidden"], name: "idx_medication_catalog_items_prescribable"
    t.check_constraint "(parse_status::text = 'review'::text) OR strength IS NOT NULL", name: "ck_medication_catalog_items_strength"
    t.check_constraint "catmat_code > 0", name: "ck_medication_catalog_items_catmat_code"
    t.check_constraint "controlled = (controlled_list IS NOT NULL)", name: "ck_medication_catalog_items_controlled_list"
    t.check_constraint "obm_code IS NULL", name: "ck_medication_catalog_items_obm_reserved"
    t.check_constraint "parse_status::text = 'ok'::text OR hidden", name: "ck_medication_catalog_items_review_hidden"
    t.check_constraint "parse_status::text = ANY (ARRAY['ok'::text, 'review'::text])", name: "ck_medication_catalog_items_parse_status"
    t.check_constraint "status::text = ANY (ARRAY['active'::text, 'removed'::text])", name: "ck_medication_catalog_items_status"
  end

  create_table "medication_catalog_releases", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "activated_at"
    t.integer "added_count", default: 0, null: false
    t.integer "changed_count", default: 0, null: false
    t.datetime "created_at", null: false
    t.integer "hidden_count", default: 0, null: false
    t.datetime "imported_at", null: false
    t.string "imported_by", limit: 80, null: false
    t.integer "items_count", default: 0, null: false
    t.integer "removed_count", default: 0, null: false
    t.string "source", default: "catmat_6505", null: false
    t.string "status", default: "importing", null: false
    t.datetime "updated_at", null: false
    t.index ["status"], name: "idx_medication_catalog_releases_one_active", unique: true, where: "((status)::text = 'active'::text)"
    t.index ["status"], name: "idx_medication_catalog_releases_one_importing", unique: true, where: "((status)::text = 'importing'::text)"
    t.check_constraint "(status::text = 'active'::text) <= (activated_at IS NOT NULL)", name: "ck_medication_catalog_releases_activation"
    t.check_constraint "status::text = ANY (ARRAY['importing'::text, 'active'::text, 'superseded'::text, 'failed'::text])", name: "ck_medication_catalog_releases_status"
  end
```

e, no bloco de `add_foreign_key` (ordem alfabética):

```ruby
  add_foreign_key "anvisa_list_substances", "anvisa_list_releases", column: "release_id"
  add_foreign_key "medication_catalog_items", "medication_catalog_releases", column: "first_release_id"
  add_foreign_key "medication_catalog_items", "medication_catalog_releases", column: "last_release_id"
```

Run (avise antes — banco de teste de plataforma compartilhado): `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19c api bin/rails db:migrate:platform`, depois `git diff --stat db/platform_schema.rb`: se o `db:migrate:platform` regravou o dump, confira que só entraram estas tabelas e a versão; nada mais.

- [ ] **Step 6: Os modelos**

```ruby
# app/models/medication_catalog_release.rb
# Uma importação do CATMAT (ADR 0033): contagens da diferença; uma ativa.
# Escrita só por Medications::CatalogImport; transições por trigger.
class MedicationCatalogRelease < PlatformRecord
  STATUSES = %w[importing active superseded failed].freeze

  validates :status, inclusion: { in: STATUSES }
  validates :imported_by, :imported_at, presence: true

  scope :active, -> { where(status: "active") }

  def self.current = active.order(activated_at: :desc).first
end
```

```ruby
# app/models/medication_catalog_item.rb
# Um medicamento do catálogo da plataforma (ADR 0033; spec §3): um por código
# CATMAT, estável, nunca apagado. `hidden` = fora da busca (revisão do
# analisador); `controlled` = bloqueado na receita (19d); `obm_code` reservado.
class MedicationCatalogItem < PlatformRecord
  PARSE_STATUSES = %w[ok review].freeze
  STATUSES = %w[active removed].freeze

  belongs_to :first_release, class_name: "MedicationCatalogRelease"
  belongs_to :last_release, class_name: "MedicationCatalogRelease"

  scope :prescribable, -> { where(status: "active", hidden: false) }

  def label = [ active_ingredient, strength, dosage_form ].compact_blank.join(" ")
  def ingredients = active_ingredient.split(" + ")
end
```

```ruby
# app/models/anvisa_list_release.rb
# Uma importação das listas da Anvisa (ADR 0033): antimicrobianos (RDC 471) e
# controlados (Portaria 344, listas A1–C5), com a fonte de cada uma.
class AnvisaListRelease < PlatformRecord
  has_many :substances, class_name: "AnvisaListSubstance", foreign_key: :release_id, inverse_of: :release

  scope :active, -> { where(status: "active") }

  def self.current = active.order(activated_at: :desc).first
end
```

```ruby
# app/models/anvisa_list_substance.rb
class AnvisaListSubstance < PlatformRecord
  KINDS = %w[antimicrobial controlled].freeze

  belongs_to :release, class_name: "AnvisaListRelease", inverse_of: :substances
end
```

- [ ] **Step 7: Rode e veja passar**

Run: `rspec spec/models/medication_catalog_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add db/platform_migrate/20261009600001_create_medication_catalog.rb db/platform_schema.rb db/platform_triggers.sql app/models/medication_catalog_release.rb app/models/medication_catalog_item.rb app/models/anvisa_list_release.rb app/models/anvisa_list_substance.rb spec/models/medication_catalog_spec.rb
git commit -m "feat: add the platform medication catalog and Anvisa list tables

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Analisador da descrição do CATMAT (amostras reais)

As amostras abaixo são itens reais da API (lidos em 2026-10-09; `descricaoItem` copiada sem alteração, inclusive o espaço no fim). O resultado esperado de cada uma foi conferido contra a regra: o que a regra não entende vai a revisão com o motivo, nunca um palpite.

**Files:**
- Create: `app/services/medications/catmat/description_parser.rb`, `spec/fixtures/medications/catmat/samples.yml`
- Test: `spec/services/medications/catmat/description_parser_spec.rb`

**Interfaces:**
- Produces: `Medications::Catmat::DescriptionParser.call(pdm_name, description) -> Parsed` (`Data` com `active_ingredient`, `strength`, `dosage_form`, `unit`, `parse_status` (`"ok"`|`"review"`), `review_reason` (`manipulated` | `unknown_field` | `missing_strength` | `unparsed_strength` | `ingredient_count` | `duplicated_strength` | `duplicated_form` | nil), `#ok?`).

- [ ] **Step 1: As amostras (fixture)**

```yaml
# spec/fixtures/medications/catmat/samples.yml
# Itens reais do CATMAT classe 6505 (API do Compras.gov, 2026-10-09).
- { code: 268856, pdm: "LOSARTANA POTÁSSICA", description: "LOSARTANA POTÁSSICA, DOSAGEM: 50 MG",
    expected: { active_ingredient: "LOSARTANA POTÁSSICA", strength: "50 MG", unit: "mg", parse_status: "ok" } }
- { code: 287471, pdm: "LOSARTANA POTÁSSICA", description: "LOSARTANA POTÁSSICA, APRESENTAÇÃO: ASSOCIADO à HIDROCLOROTIAZIDA , DOSAGEM: 100 MG + 25 MG ",
    expected: { active_ingredient: "LOSARTANA POTÁSSICA + HIDROCLOROTIAZIDA", strength: "100 MG + 25 MG", unit: "mg", parse_status: "ok" } }
- { code: 270788, pdm: "LOSARTANA POTÁSSICA", description: "LOSARTANA POTÁSSICA, APRESENTAÇÃO: ASSOCIADO à HIDROCLOROTIAZIDA , DOSAGEM: 50MG + 12,5MG ",
    expected: { active_ingredient: "LOSARTANA POTÁSSICA + HIDROCLOROTIAZIDA", strength: "50 MG + 12,5 MG", unit: "mg", parse_status: "ok" } }
- { code: 384258, pdm: "LOSARTANA POTÁSSICA", description: "LOSARTANA POTÁSSICA, COMPOSIÇÃO: ASSOC. À FUROSEMIDA,AMIODARONA E ESPIRONOLACTONA , CONCENTRAÇÃO: 50 MG + 60 MG + 150 MG + 25 MG, CARACTERÍSTICA ADICIONAL: FORMULAÇÃO ESPECIALMENTE MANIPULADA ",
    expected: { active_ingredient: "LOSARTANA POTÁSSICA", parse_status: "review", review_reason: "manipulated" } }
- { code: 268252, pdm: "DIPIRONA SÓDICA", description: "DIPIRONA SÓDICA, DOSAGEM: 500 MG/ML, APRESENTAÇÃO: SOLUÇÃO INJETÁVEL ",
    expected: { active_ingredient: "DIPIRONA SÓDICA", strength: "500 MG/ML", dosage_form: "solução injetável", unit: "mg/mL", parse_status: "ok" } }
- { code: 448841, pdm: "AMOXICILINA", description: "AMOXICILINA, PRINCÍPIO ATIVO: ASSOCIADA COM CLAVULANATO DE POTÁSSIO , CONCENTRAÇÃO: 50 MG/ML + 12,5 MG/ML, FORMA FARMACÊUTICA: SUSPENSÃO ORAL ",
    expected: { active_ingredient: "AMOXICILINA + CLAVULANATO DE POTÁSSIO", strength: "50 MG/ML + 12,5 MG/ML", dosage_form: "suspensão oral", unit: "mg/mL", parse_status: "ok" } }
- { code: 271113, pdm: "AMOXICILINA", description: "AMOXICILINA, CONCENTRAÇÃO: 100 MG/ML, APRESENTAÇÃO: PÓ PARA SUSPENSÃO ORAL ",
    expected: { active_ingredient: "AMOXICILINA", strength: "100 MG/ML", dosage_form: "pó para suspensão oral", unit: "mg/mL", parse_status: "ok" } }
- { code: 271089, pdm: "AMOXICILINA", description: "AMOXICILINA, CONCENTRAÇÃO: 500MG ",
    expected: { active_ingredient: "AMOXICILINA", strength: "500 MG", unit: "mg", parse_status: "ok" } }
- { code: 270120, pdm: "CLONAZEPAM", description: "CLONAZEPAM, DOSAGEM: 2,5 MG/ML, APRESENTAÇÃO: SOLUÇÃO ORAL- GOTAS ",
    expected: { active_ingredient: "CLONAZEPAM", strength: "2,5 MG/ML", dosage_form: "solução oral- gotas", unit: "mg/mL", parse_status: "ok" } }
- { code: 294887, pdm: "SALBUTAMOL", description: "SALBUTAMOL, DOSAGEM: 100MCG/DOSE , FORMA FARMACÊUTICA: AEROSOL ORAL ",
    expected: { active_ingredient: "SALBUTAMOL", strength: "100 MCG/DOSE", dosage_form: "aerosol oral", unit: "mcg/dose", parse_status: "ok" } }
- { code: 629308, pdm: "SULFAMETOXAZOL", description: "SULFAMETOXAZOL, COMPOSIÇÃO: ASSOCIADO À TRIMETOPRIMA , CONCENTRAÇÃO: 200 mg + 40 MG, FORMA FARMACÊUTICA: SUSPENSÃO ORAL ",
    expected: { active_ingredient: "SULFAMETOXAZOL + TRIMETOPRIMA", strength: "200 MG + 40 MG", dosage_form: "suspensão oral", unit: "mg", parse_status: "ok" } }
- { code: 277513, pdm: "FLUOXETINA", description: "FLUOXETINA, DOSAGEM: 20 MG/ML, APRESENTAÇÃO: SOLUÇÃO ORAL, GOTAS ",
    expected: { active_ingredient: "FLUOXETINA", strength: "20 MG/ML", dosage_form: "solução oral, gotas", unit: "mg/mL", parse_status: "ok" } }
- { code: 266788, pdm: "NISTATINA", description: "NISTATINA, DOSAGEM: 25.000 UI/G , APRESENTAÇÃO: CREME VAGINAL ",
    expected: { active_ingredient: "NISTATINA", strength: "25.000 UI/G", dosage_form: "creme vaginal", unit: "UI/g", parse_status: "ok" } }
- { code: 438153, pdm: "INSULINA", description: "INSULINA, TIPO: GLARGINA , CONCENTRAÇÃO: 100 UI/ML, FORMA FARMACEUTICA: SOLUÇÃO INJETÁVEL , CARACTERISTICA ADICIONAL: REFIL ",
    expected: { active_ingredient: "INSULINA GLARGINA", strength: "100 UI/ML", dosage_form: "solução injetável", unit: "UI/mL", parse_status: "ok" } }
- { code: 396051, pdm: "INSULINA", description: "INSULINA, ORIGEM: ASPART , CONCENTRAÇÃO: 100 UI/ML, FORMA FARMACEUTICA: SOLUÇÃO INJETÁVEL , CARACTERISTICA ADICIONAL: COM APLICADOR ",
    expected: { active_ingredient: "INSULINA ASPART", strength: "100 UI/ML", dosage_form: "solução injetável", unit: "UI/mL", parse_status: "ok" } }
- { code: 296649, pdm: "LEVOTIROXINA SÓDICA", description: "LEVOTIROXINA SÓDICA, DOSAGEM: 88 MCG ",
    expected: { active_ingredient: "LEVOTIROXINA SÓDICA", strength: "88 MCG", unit: "mcg", parse_status: "ok" } }
- { code: 267187, pdm: "DEXAMETASONA", description: "DEXAMETASONA, DOSAGEM: 0,1% , APRESENTAÇÃO: SOLUÇÃO OFTÁLMICA ",
    expected: { active_ingredient: "DEXAMETASONA", strength: "0,1 %", dosage_form: "solução oftálmica", unit: "%", parse_status: "ok" } }
- { code: 270906, pdm: "PARACETAMOL", description: "PARACETAMOL, APRESENTAÇÃO: ASSOCIADO COM CODEÍNA , DOSAGEM: 500MG + 7,5MG ",
    expected: { active_ingredient: "PARACETAMOL + CODEÍNA", strength: "500 MG + 7,5 MG", unit: "mg", parse_status: "ok" } }
- { code: 267778, pdm: "PARACETAMOL", description: "PARACETAMOL, DOSAGEM COMPRIMIDO: 500 MG",
    expected: { active_ingredient: "PARACETAMOL", strength: "500 MG", dosage_form: "comprimido", unit: "mg", parse_status: "ok" } }
- { code: 474749, pdm: "DIPIRONA SÓDICA", description: "DIPIRONA SÓDICA, CONCENTRAÇÃO: 1 G, FORMA FARMACÊUTICA: COMPRIMIDO EFERVESCENTE ",
    expected: { active_ingredient: "DIPIRONA SÓDICA", strength: "1 G", dosage_form: "comprimido efervescente", unit: "g", parse_status: "ok" } }
- { code: 268493, pdm: "DOXAZOSINA MESILATO", description: "DOXAZOSINA MESILATO, COMPOSIÇÃO: 2 MG ",
    expected: { active_ingredient: "DOXAZOSINA MESILATO", strength: "2 MG", unit: "mg", parse_status: "ok" } }
- { code: 268162, pdm: "MICONAZOL NITRATO", description: "MICONAZOL NITRATO, DOSAGEM: 2% , APRESENTAÇÃO: CREME VAGINAL ",
    expected: { active_ingredient: "MICONAZOL NITRATO", strength: "2 %", dosage_form: "creme vaginal", unit: "%", parse_status: "ok" } }
- { code: 271172, pdm: "ENALAPRIL MALEATO", description: "ENALAPRIL MALEATO, APRESENTAÇÃO: ASSOCIADO COM HIDROCLOROTIAZIDA , CARACTERÍSTICAS ADICIONAIS: 10MG + 25MG ",
    expected: { active_ingredient: "ENALAPRIL MALEATO", parse_status: "review", review_reason: "missing_strength" } }
- { code: 406477, pdm: "DEXAMETASONA", description: "DEXAMETASONA, COMPOSIÇÃO: ACETATO, ASSOCIADA À NEOMICINA SULFATO , CONCENTRAÇAO: 1 MG + 5 MG/G, FORMA FARMACEUTICA: CREME ",
    expected: { active_ingredient: "DEXAMETASONA", parse_status: "review", review_reason: "unknown_field" } }
- { code: 273621, pdm: "SULFATO FERROSO", description: "SULFATO FERROSO, DOSAGEM FERRO: 300 MG",
    expected: { active_ingredient: "SULFATO FERROSO", parse_status: "review", review_reason: "unknown_field" } }
- { code: 471885, pdm: "MEDICAMENTO HOMEOPÁTICO", description: "MEDICAMENTO HOMEOPÁTICO, COMPOSIÇÃO: À BASE DE ANTIMONIUM TARTARICUM , ESCALA: 30 CH ",
    expected: { active_ingredient: "MEDICAMENTO HOMEOPÁTICO", parse_status: "review", review_reason: "unknown_field" } }
- { code: 381881, pdm: "OMEPRAZOL", description: "OMEPRAZOL, COMPOSIÇÃO: OMEPRAZOL MAGNÉSICO , CONCENTRAÇÃO: 41,3 MG",
    expected: { active_ingredient: "OMEPRAZOL", parse_status: "review", review_reason: "unknown_field" } }
- { code: 292778, pdm: "METOPROLOL", description: "METOPROLOL, PRINCÍPIO ATIVO: TARTARATO, ASSOCIADO À HIDROCLOROTIAZIDA , DOSAGEM: 100 MG + 12,5 MG",
    expected: { active_ingredient: "METOPROLOL", parse_status: "review", review_reason: "unknown_field" } }
- { code: 267777, pdm: "PARACETAMOL", description: "PARACETAMOL, DOSAGEM SOLUÇÃO ORAL: 200 MG/ML, APRESENTAÇÃO: SOLUÇÃO ORAL ",
    expected: { active_ingredient: "PARACETAMOL", parse_status: "review", review_reason: "duplicated_form" } }
```

> A última amostra: `DOSAGEM SOLUÇÃO ORAL` é chave de força com forma, e `APRESENTAÇÃO` traz a forma de novo — duas formas → revisão `duplicated_form`.

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/services/medications/catmat/description_parser_spec.rb
require "rails_helper"

# ADR 0033 (spec §3, §9): o analisador contra amostras REAIS do CATMAT. O que
# a regra não entende vai a revisão com o motivo — nunca um palpite.
RSpec.describe Medications::Catmat::DescriptionParser do
  samples = YAML.load_file(Rails.root.join("spec/fixtures/medications/catmat/samples.yml"), symbolize_names: true)

  samples.each do |sample|
    it "#{sample[:code]}: #{sample[:description].strip}" do
      parsed = described_class.call(sample[:pdm], sample[:description])
      expect(parsed.to_h.compact).to eq(sample[:expected])
    end
  end

  it "revisão nunca traz força, forma nem unidade" do
    parsed = described_class.call("X", "X, ESCALA: 30 CH")
    expect(parsed).to have_attributes(parse_status: "review", strength: nil, dosage_form: nil, unit: nil)
    expect(parsed.ok?).to be(false)
  end

  it "descrição vazia ou sem PDM não levanta" do
    expect(described_class.call(nil, "").parse_status).to eq("review")
    expect(described_class.call("", "AMOXICILINA, DOSAGEM: 500 MG").active_ingredient).to eq("AMOXICILINA")
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `rspec spec/services/medications/catmat/description_parser_spec.rb`
Expected: FAIL (`uninitialized constant Medications`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/medications/catmat/description_parser.rb
# Analisador da descrição semiestruturada do CATMAT classe 6505 (ADR 0033;
# spec §3). Entende o recorte que a APS prescreve: princípio ativo (o nome do
# PDM, mais os associados), concentração, forma e unidade. O resto vai a
# revisão (parse_status review → hidden) com o motivo — nunca um palpite.
# Sobre os 6.778 itens reais de 2026-10-09: 4.098 ok, 2.680 em revisão.
module Medications
  module Catmat
    module DescriptionParser
      Parsed = Data.define(:active_ingredient, :strength, :dosage_form, :unit, :parse_status, :review_reason) do
        def ok? = parse_status == "ok"
      end

      # Vírgula seguida de "CHAVE:" (maiúscula, com acento, espaço, ponto e um
      # número no fim). Vírgula dentro de valor ("SOLUÇÃO ORAL, GOTAS") não é
      # seguida de chave e não divide.
      SPLIT = /,\s*(?=[A-ZÀ-Ü][A-ZÀ-Ü .]*?(?:\s\d)?\s*:)/
      STRENGTH_KEYS = %w[CONCENTRACAO DOSAGEM TEOR].freeze
      # "DOSAGEM COMPRIMIDO: 500 MG": a chave traz a forma.
      STRENGTH_WITH_FORM = /\ADOSAGEM (COMPRIMIDO|CAPSULA|DRAGEA|SOLUCAO ORAL|SUSPENSAO ORAL|XAROPE)\z/
      FORM_KEYS = [ "FORMA FARMACEUTICA", "APRESENTACAO", "USO", "TIPO USO", "APLICACAO", "INDICACAO",
                    "TIPO MEDICAMENTO" ].freeze
      ASSOCIATION_KEYS = [ "COMPOSICAO", "PRINCIPIO ATIVO", "APRESENTACAO" ].freeze
      VARIANT_KEYS = %w[TIPO ORIGEM].freeze
      EXTRA_KEYS = [ "CARACTERISTICA ADICIONAL", "CARACTERISTICAS ADICIONAIS", "ADICIONAL" ].freeze
      ASSOCIATION = /\AASSOC(?:IAD[AO]S?|\.)?\s*(?:COM|AO|AOS|A|AS|À|ÀS|E)?\s+/i
      COMPONENT = %r{\A(\d[\d.]*(?:,\d+)?)\s*(MCG|MG|G|UI|U|MEQ|MMOL|ML|%)(?:\s*/\s*(ML|G|DOSE|L))?\z}i
      UNITS = { "MCG" => "mcg", "MG" => "mg", "G" => "g", "UI" => "UI", "U" => "U", "MEQ" => "mEq", "MMOL" => "mmol",
                "ML" => "mL", "%" => "%" }.freeze
      PER = { "ML" => "mL", "G" => "g", "DOSE" => "dose", "L" => "L" }.freeze
      MANIPULATED = /MANIPULAD/i

      module_function

      def call(pdm_name, description)
        head, *pairs = description.to_s.strip.split(SPLIT)
        name = (pdm_name.to_s.strip.presence || head.to_s.strip).squeeze(" ")
        fields = pairs.map do |pair|
          key, value = pair.split(":", 2)
          [ normalize_key(key), key.to_s.strip.squeeze(" "), value.to_s.strip.squeeze(" ") ]
        end
        return review(name, "manipulated") if fields.any? { |_key, _original, value| value.match?(MANIPULATED) }

        ingredients = [ name ]
        strength = nil
        form = nil
        fields.each do |key, original, value|
          if STRENGTH_KEYS.include?(key)
            return review(name, "duplicated_strength") if strength

            strength = value
          elsif key.match?(STRENGTH_WITH_FORM)
            return review(name, "duplicated_strength") if strength || form

            strength = value
            form = original.sub(/\ADOSAGEM\s+/i, "")
          elsif ASSOCIATION_KEYS.include?(key) && strength.nil? && strength?(value)
            strength = value
          elsif ASSOCIATION_KEYS.include?(key) && value.match?(ASSOCIATION)
            ingredients.concat(value.sub(ASSOCIATION, "").split(/\s*(?:,|\+|\sE\s)\s*/i).map(&:strip).reject(&:empty?))
          elsif FORM_KEYS.include?(key)
            return review(name, "duplicated_form") if form

            form = value
          elsif VARIANT_KEYS.include?(key)
            ingredients[0] = "#{name} #{value}"
          elsif !EXTRA_KEYS.include?(key)
            return review(name, "unknown_field")
          end
        end
        return review(name, "missing_strength") unless strength

        components = strength.split(/\s*\+\s*/).map(&:strip)
        matches = components.map { |component| component.match(COMPONENT) }
        return review(name, "unparsed_strength") if matches.any?(&:nil?)
        return review(name, "ingredient_count") if components.size != ingredients.size

        units = matches.map { |m| [ UNITS.fetch(m[2].upcase), m[3] && PER.fetch(m[3].upcase) ].compact.join("/") }.uniq
        Parsed.new(active_ingredient: ingredients.join(" + "),
                   strength: components.map { |c| c.upcase.gsub(/(\d)\s*([A-Z%])/, '\1 \2').gsub(%r{\s*/\s*}, "/") }.join(" + "),
                   dosage_form: form&.downcase, unit: units.one? ? units.first : nil, parse_status: "ok", review_reason: nil)
      end

      def strength?(value) = value.split(/\s*\+\s*/).all? { |component| component.strip.match?(COMPONENT) }

      def normalize_key(key) = I18n.transliterate(key.to_s).upcase.sub(/\s+\d+\z/, "").squeeze(" ").strip

      def review(name, reason)
        Parsed.new(active_ingredient: name, strength: nil, dosage_form: nil, unit: nil, parse_status: "review",
                   review_reason: reason)
      end
      private_class_method :strength?, :normalize_key, :review
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `rspec spec/services/medications/catmat/description_parser_spec.rb`
Expected: PASS (29 amostras + 2).

- [ ] **Step 6: Commit**

```bash
git add app/services/medications/catmat/description_parser.rb spec/fixtures/medications/catmat/samples.yml spec/services/medications/catmat/description_parser_spec.rb
git commit -m "feat: parse CATMAT medication descriptions with real samples

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 4: Cliente do CATMAT com paginação (e o recorte real de dev)

**Files:**
- Create: `app/services/medications/catmat/client.rb`, `app/services/medications/catmat/fixture_client.rb`, `db/seeds/medications/catmat-dev.json`, `spec/support/medication_helpers.rb`
- Modify: `spec/rails_helper.rb` (`require_relative "support/medication_helpers"`)
- Test: `spec/services/medications/catmat/client_spec.rb`

**Interfaces:**
- Produces:
  - `Medications::Catmat::Client.new(base_url: ENV.fetch("CATMAT_API_URL", BASE_URL), page_size: 500, sleeper: ->(seconds) { sleep(seconds) })` com `#fetch_all -> Array<Medications::Catmat::Row>`; levanta `Medications::Catmat::Client::Unavailable` (rede, timeout, HTTP ≠ 200 depois de 2 novas tentativas, JSON inválido) ou `Medications::Catmat::Client::Incomplete` (página vazia antes do fim, total que muda entre páginas, lido ≠ `totalRegistros`).
  - `Medications::Catmat::Row = Data.define(:catmat_code, :pdm_code, :pdm_name, :description, :updated_at)`.
  - `Medications::Catmat::FixtureClient.new(path)` com o mesmo `#fetch_all` (lê um array JSON de linhas no formato da API).
  - Helpers de spec: `catmat_rows -> Array<Hash>` (o recorte de dev), `stub_catmat!(rows = catmat_rows, per_page: 9) -> WebMock stubs`.

- [ ] **Step 1: Confira a API real (uma chamada, só leitura)**

Run: `curl -s -m 60 "https://dadosabertos.compras.gov.br/modulo-material/4_consultarItemMaterial?pagina=1&tamanhoPagina=5&codigoClasse=6505&statusItem=true" | head -c 700`
Expected: JSON com `resultado` (itens com `codigoItem`, `codigoPdm`, `nomePdm`, `descricaoItem`, `statusItem`, `dataHoraAtualizacao`), `totalRegistros`, `totalPaginas`, `paginasRestantes` — o envelope do Desvio 1. Nome diferente: ajuste só `Client#row`/`#fetch_all` e o helper `stub_catmat!` (o resto do plano não conhece esses nomes). Fora do ar: siga com os fixtures e anote para a Task 25.

- [ ] **Step 2: O recorte real de dev (também é o fixture das specs)**

Itens reais (2026-10-09), só os campos que o api lê:

```json
[{"codigoItem":268856,"codigoPdm":8922,"nomePdm":"LOSARTANA POTÁSSICA","descricaoItem":"LOSARTANA POTÁSSICA, DOSAGEM: 50 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267674,"codigoPdm":8220,"nomePdm":"HIDROCLOROTIAZIDA","descricaoItem":"HIDROCLOROTIAZIDA, DOSAGEM: 25 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267690,"codigoPdm":5206,"nomePdm":"METFORMINA CLORIDRATO","descricaoItem":"METFORMINA CLORIDRATO, DOSAGEM: 500 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267747,"codigoPdm":12096,"nomePdm":"SINVASTATINA","descricaoItem":"SINVASTATINA, DOSAGEM: 20 MG ","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":271089,"codigoPdm":2436,"nomePdm":"AMOXICILINA","descricaoItem":"AMOXICILINA, CONCENTRAÇÃO: 500MG ","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267625,"codigoPdm":4720,"nomePdm":"CEFALEXINA","descricaoItem":"CEFALEXINA, DOSAGEM: 500 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267778,"codigoPdm":10422,"nomePdm":"PARACETAMOL","descricaoItem":"PARACETAMOL, DOSAGEM COMPRIMIDO: 500 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267203,"codigoPdm":17708,"nomePdm":"DIPIRONA SÓDICA","descricaoItem":"DIPIRONA SÓDICA, DOSAGEM: 500 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267503,"codigoPdm":354,"nomePdm":"ÁCIDO FÓLICO","descricaoItem":"ÁCIDO FÓLICO, DOSAGEM: 5 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":268162,"codigoPdm":9717,"nomePdm":"MICONAZOL NITRATO","descricaoItem":"MICONAZOL NITRATO, DOSAGEM: 2% , APRESENTAÇÃO: CREME VAGINAL ","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":294887,"codigoPdm":11917,"nomePdm":"SALBUTAMOL","descricaoItem":"SALBUTAMOL, DOSAGEM: 100MCG/DOSE , FORMA FARMACÊUTICA: AEROSOL ORAL ","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267712,"codigoPdm":10216,"nomePdm":"OMEPRAZOL","descricaoItem":"OMEPRAZOL, CONCENTRAÇÃO: 20 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267671,"codigoPdm":8000,"nomePdm":"GLIBENCLAMIDA","descricaoItem":"GLIBENCLAMIDA, DOSAGEM: 5 MG ","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267772,"codigoPdm":5235,"nomePdm":"PROPRANOLOL CLORIDRATO","descricaoItem":"PROPRANOLOL CLORIDRATO, DOSAGEM: 40 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":267197,"codigoPdm":6168,"nomePdm":"DIAZEPAM","descricaoItem":"DIAZEPAM, DOSAGEM: 10 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":273009,"codigoPdm":7667,"nomePdm":"FLUOXETINA","descricaoItem":"FLUOXETINA, DOSAGEM: 20 MG","statusItem":true,"dataHoraAtualizacao":"2021-10-16T09:43:08.030221"},
{"codigoItem":384258,"codigoPdm":8922,"nomePdm":"LOSARTANA POTÁSSICA","descricaoItem":"LOSARTANA POTÁSSICA, COMPOSIÇÃO: ASSOC. À FUROSEMIDA,AMIODARONA E ESPIRONOLACTONA , CONCENTRAÇÃO: 50 MG + 60 MG + 150 MG + 25 MG, CARACTERÍSTICA ADICIONAL: FORMULAÇÃO ESPECIALMENTE MANIPULADA ","statusItem":true,"dataHoraAtualizacao":"2022-07-02T03:00:20.533284"}]
```

Grave como `db/seeds/medications/catmat-dev.json` (17 itens: 2 antimicrobianos, 2 controlados, 1 em revisão).

- [ ] **Step 3: Os helpers de spec**

```ruby
# spec/support/medication_helpers.rb
# Catálogo de medicamentos (ADR 0033): a API do CATMAT servida por WebMock com
# o recorte REAL de dev (db/seeds/medications/catmat-dev.json), paginado como a
# API; as listas Anvisa de teste (spec/fixtures/medications/anvisa); e o
# catálogo importado pelo caminho real (Medications::AnvisaImport +
# Medications::CatalogImport).
module MedicationHelpers
  CATMAT_URL = %r{\Ahttps://dadosabertos\.compras\.gov\.br/modulo-material/4_consultarItemMaterial}

  def catmat_rows = JSON.parse(File.read(Rails.root.join("db/seeds/medications/catmat-dev.json")))

  # pages: substitui páginas inteiras ({ 2 => { status: 503 } } ou { 2 => body }).
  def stub_catmat!(rows = catmat_rows, per_page: 9, total: rows.size, pages: {})
    slices = rows.each_slice(per_page).to_a
    stub_request(:get, CATMAT_URL).to_return do |request|
      page = Integer(URI.decode_www_form(URI(request.uri.to_s).query).to_h.fetch("pagina"))
      override = pages[page]
      next override if override.is_a?(Hash) && override.key?(:status)

      body = override || { "resultado" => slices[page - 1] || [], "totalRegistros" => total,
                           "totalPaginas" => slices.size, "paginasRestantes" => [ slices.size - page, 0 ].max }
      { status: 200, body: body.to_json, headers: { "Content-Type" => "application/json" } }
    end
  end

  def anvisa_fixture_dir = Rails.root.join("spec/fixtures/medications/anvisa")

  # Listas de teste + o recorte de dev, pelo caminho real. Idempotente no exemplo.
  def import_test_catalog!
    unless AnvisaListRelease.current
      result = Medications::AnvisaImport.call(by: "rspec", dir: anvisa_fixture_dir)
      raise "listas não importadas: #{result.reason} #{result.message}" if result.failure?
    end
    unless MedicationCatalogRelease.current
      client = Medications::Catmat::FixtureClient.new(Rails.root.join("db/seeds/medications/catmat-dev.json"))
      result = Medications::CatalogImport.call(by: "rspec", client: client)
      raise "catálogo não importado: #{result.reason} #{result.message}" if result.failure?
    end
    MedicationCatalogItem.all.index_by(&:catmat_code)
  end

  def catalog_item(code) = MedicationCatalogItem.find_by!(catmat_code: code)
end

RSpec.configure { |c| c.include MedicationHelpers }
```

(`import_test_catalog!` usa `Medications::AnvisaImport` e `Medications::CatalogImport` das Tasks 5 e 6; esta task só usa `catmat_rows`/`stub_catmat!`.)

- [ ] **Step 4: Escreva a spec que falha**

```ruby
# spec/services/medications/catmat/client_spec.rb
require "rails_helper"

# ADR 0033 (spec §3): a API aberta do Compras.gov, classe 6505, paginada.
# Falha passageira tenta de novo; falha que dura vira Unavailable; fonte que
# encolhe ou muda no meio vira Incomplete — nunca um catálogo pela metade.
RSpec.describe Medications::Catmat::Client do
  let(:client) { described_class.new(sleeper: ->(_seconds) {}) }

  it "lê todas as páginas com os parâmetros da classe 6505" do
    stub_catmat!
    rows = client.fetch_all
    expect(rows.size).to eq(17)
    expect(rows.first).to have_attributes(catmat_code: 268_856, pdm_code: 8922, pdm_name: "LOSARTANA POTÁSSICA",
                                          description: "LOSARTANA POTÁSSICA, DOSAGEM: 50 MG")
    expect(rows.first.updated_at).to eq(Time.utc(2021, 10, 16, 9, 43, 8, 30_221))
    expect(a_request(:get, MedicationHelpers::CATMAT_URL)
             .with(query: hash_including("codigoClasse" => "6505", "statusItem" => "true", "tamanhoPagina" => "500",
                                         "pagina" => "2"))).to have_been_made.once
  end

  it "falha passageira tenta de novo (até 2 vezes) e segue" do
    stub_catmat!
    page2 = { "resultado" => catmat_rows.last(8), "totalRegistros" => 17, "totalPaginas" => 2, "paginasRestantes" => 0 }
    # O stub mais recente vence: só a página 2 falha duas vezes e depois responde.
    stub_request(:get, MedicationHelpers::CATMAT_URL).with(query: hash_including("pagina" => "2"))
      .to_return({ status: 503, body: "indisponível" }, { status: 503, body: "indisponível" },
                 { status: 200, body: page2.to_json, headers: { "Content-Type" => "application/json" } })
    expect(client.fetch_all.size).to eq(17)
    expect(a_request(:get, MedicationHelpers::CATMAT_URL).with(query: hash_including("pagina" => "2"))).to have_been_made.times(3)
  end

  it "fora do ar depois das novas tentativas → Unavailable, sem o corpo na mensagem" do
    stub_catmat!(pages: { 2 => { status: 503, body: "segredo-do-corpo" } })
    expect { client.fetch_all }.to raise_error(described_class::Unavailable) { |e| expect(e.message).not_to include("segredo") }
  end

  it "timeout e JSON inválido → Unavailable" do
    stub_request(:get, MedicationHelpers::CATMAT_URL).to_timeout
    expect { client.fetch_all }.to raise_error(described_class::Unavailable)
    WebMock.reset!
    stub_request(:get, MedicationHelpers::CATMAT_URL).to_return(status: 200, body: "<html>")
    expect { client.fetch_all }.to raise_error(described_class::Unavailable)
  end

  it "página vazia antes do fim, total que muda ou lido ≠ total → Incomplete" do
    stub_catmat!(pages: { 2 => { "resultado" => [], "totalRegistros" => 17, "totalPaginas" => 2, "paginasRestantes" => 0 } })
    expect { client.fetch_all }.to raise_error(described_class::Incomplete)
    WebMock.reset!
    stub_catmat!(pages: { 2 => { "resultado" => catmat_rows.last(8), "totalRegistros" => 18, "totalPaginas" => 2,
                                 "paginasRestantes" => 0 } })
    expect { client.fetch_all }.to raise_error(described_class::Incomplete)
    WebMock.reset!
    stub_catmat!(catmat_rows.first(16), total: 17)
    expect { client.fetch_all }.to raise_error(described_class::Incomplete)
  end

  it "o FixtureClient devolve as mesmas linhas, sem rede" do
    rows = Medications::Catmat::FixtureClient.new(Rails.root.join("db/seeds/medications/catmat-dev.json")).fetch_all
    expect(rows.map(&:catmat_code)).to include(268_856, 384_258)
    expect(rows.size).to eq(17)
  end
end
```

- [ ] **Step 5: Rode e veja falhar**

Run: `rspec spec/services/medications/catmat/client_spec.rb`
Expected: FAIL (`uninitialized constant Medications::Catmat::Client`).

- [ ] **Step 6: Implemente**

```ruby
# app/services/medications/catmat/client.rb
# Cliente da API aberta do Compras.gov para o CATMAT classe 6505 (ADR 0033;
# spec §3; conferida em 2026-10-09, Desvio 1). Lê todas as páginas (pagina a
# partir de 1, até totalPaginas) e confere o total: o catálogo nunca é
# importado pela metade. Passageiro (rede, timeout, 5xx, JSON inválido) tenta
# de novo 2 vezes com espera; o corpo da resposta nunca vai a mensagem ou log.
require "net/http"

module Medications
  module Catmat
    Row = Data.define(:catmat_code, :pdm_code, :pdm_name, :description, :updated_at)

    class Client
      class Unavailable < StandardError; end
      class Incomplete < StandardError; end

      BASE_URL = "https://dadosabertos.compras.gov.br".freeze
      PATH = "/modulo-material/4_consultarItemMaterial".freeze
      CLASS_CODE = 6505
      PAGE_SIZE = 500
      MAX_PAGES = 100
      RETRIES = 2
      BACKOFF = [ 2, 10 ].freeze

      def initialize(base_url: ENV.fetch("CATMAT_API_URL", BASE_URL), page_size: PAGE_SIZE, sleeper: ->(seconds) { sleep(seconds) })
        @base_url = base_url.chomp("/")
        @page_size = page_size
        @sleeper = sleeper
      end

      def fetch_all
        rows = []
        total = nil
        page = 1
        loop do
          body = get_page(page)
          page_total = Integer(body.fetch("totalRegistros"))
          pages = Integer(body.fetch("totalPaginas"))
          total ||= page_total
          raise Incomplete, "totalRegistros mudou entre páginas" if page_total != total

          batch = Array(body.fetch("resultado"))
          raise Incomplete, "página #{page} vazia antes da última (#{pages})" if batch.empty? && page <= pages && total.positive?

          rows.concat(batch)
          break if page >= pages

          page += 1
          raise Incomplete, "mais de #{MAX_PAGES} páginas" if page > MAX_PAGES
        end
        read = rows.map { |row| Row.new(**row_attrs(row)) }.uniq(&:catmat_code)
        raise Incomplete, "lidos #{read.size}, total #{total}" if read.size != total

        # A consulta já pede statusItem=true; item inativo que vier mesmo assim não entra.
        inactive = rows.select { |row| row["statusItem"] == false }.to_set { |row| Integer(row["codigoItem"]) }
        read.reject { |row| inactive.include?(row.catmat_code) }
      rescue KeyError, ArgumentError, TypeError
        raise Unavailable, "resposta fora do formato esperado"
      end

      private

      def row_attrs(row)
        { catmat_code: Integer(row.fetch("codigoItem")), pdm_code: row["codigoPdm"]&.then { |code| Integer(code) },
          pdm_name: row["nomePdm"].to_s, description: row.fetch("descricaoItem").to_s,
          updated_at: row["dataHoraAtualizacao"].presence&.then { |value| Time.find_zone("UTC").parse(value) } }
      end

      def get_page(page)
        attempts = 0
        begin
          attempts += 1
          response = http_get(page)
          raise Unavailable, "HTTP #{response.code} na página #{page}" unless response.code == "200"

          JSON.parse(response.body)
        rescue Unavailable, JSON::ParserError, Net::OpenTimeout, Net::ReadTimeout, SocketError, SystemCallError, OpenSSL::SSL::SSLError => e
          if attempts <= RETRIES && !client_error?(e)
            @sleeper.call(BACKOFF.fetch(attempts - 1, BACKOFF.last))
            retry
          end
          raise Unavailable, "CATMAT indisponível na página #{page} (#{e.class.name.demodulize})"
        end
      end

      def client_error?(error) = error.is_a?(Unavailable) && error.message.match?(/HTTP 4\d\d/)

      def http_get(page)
        uri = URI("#{@base_url}#{PATH}")
        uri.query = URI.encode_www_form(pagina: page, tamanhoPagina: @page_size, codigoClasse: CLASS_CODE, statusItem: true)
        Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https", open_timeout: 10, read_timeout: 60) do |http|
          http.request(Net::HTTP::Get.new(uri, "Accept" => "application/json"))
        end
      end
    end
  end
end
```

```ruby
# app/services/medications/catmat/fixture_client.rb
# O mesmo contrato do Client lendo um arquivo com as linhas da API (o recorte
# real de dev, db/seeds/medications/catmat-dev.json): a semente e as specs
# importam pelo caminho real sem chamar a API externa.
module Medications
  module Catmat
    class FixtureClient
      def initialize(path) = @path = path

      def fetch_all
        JSON.parse(File.read(@path)).map do |row|
          Row.new(catmat_code: Integer(row.fetch("codigoItem")), pdm_code: row["codigoPdm"], pdm_name: row["nomePdm"].to_s,
                  description: row.fetch("descricaoItem").to_s,
                  updated_at: row["dataHoraAtualizacao"].presence&.then { |value| Time.find_zone("UTC").parse(value) })
        end
      end
    end
  end
end
```

- [ ] **Step 7: Rode e veja passar**

Run: `rspec spec/services/medications/catmat/client_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add app/services/medications/catmat/client.rb app/services/medications/catmat/fixture_client.rb db/seeds/medications/catmat-dev.json spec/support/medication_helpers.rb spec/rails_helper.rb spec/services/medications/catmat/client_spec.rb
git commit -m "feat: add the paginated CATMAT client for the Compras.gov open API

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Listas da Anvisa (antimicrobianos e controlados) e as marcas do catálogo

**Files:**
- Create: `config/medications/anvisa/antimicrobials.csv`, `config/medications/anvisa/controlled.csv`, `config/medications/anvisa/SOURCES.yml`, `app/services/medications/substances.rb`, `app/services/medications/anvisa_import.rb`, `spec/fixtures/medications/anvisa/{antimicrobials.csv,controlled.csv,SOURCES.yml}`
- Modify: `spec/events/platform_event_payload_guard_spec.rb` (`anvisa_lists.imported`)
- Test: `spec/services/medications/substances_spec.rb`, `spec/services/medications/anvisa_import_spec.rb`, `spec/config/anvisa_lists_spec.rb`

**Interfaces:**
- Consumes: `AnvisaListRelease`, `AnvisaListSubstance`, `MedicationCatalogItem` (Task 2).
- Produces:
  - `Medications::Substances.key(name) -> String` (sem acento, maiúsculo, só `[A-Z0-9 ]`, espaços únicos).
  - `Medications::Substances::Flags = Data.define(:antimicrobial, :controlled, :controlled_list)`.
  - `Medications::Substances.matcher(release = AnvisaListRelease.current) -> Matcher`; `Matcher#flags_for(ingredients) -> Flags`, `Matcher#scan(text) -> Flags`, `Matcher#empty? -> bool`.
  - `Medications::Substances.apply_to_catalog!(matcher) -> Integer` (itens com marca alterada).
  - `Medications::AnvisaImport.call(by:, dir: Medications::AnvisaImport::DIR) -> Result` (`ok(release:, antimicrobials:, controlled:)` | `fail(:import_in_progress)` | `fail(:invalid_file, message:)` | `fail(:import_failed, message:)`).
  - `Platform.audit("anvisa_lists.imported", release_id:, antimicrobials:, controlled:)`.

- [ ] **Step 1: Transcreva as listas oficiais (dados, não código)**

Fontes (as mesmas da pesquisa §3; leia com WebFetch, só leitura; se a fonte só existir em PDF, **peça autorização ao usuário** para baixar, dizendo arquivo, origem e tamanho):
- **Antimicrobianos:** RDC Anvisa nº 471/2021, Anexo I (lista de substâncias antimicrobianas de uso sob prescrição, com retenção), na redação vigente (RDC 973/2025 / IN 360/2025, se alteraram a lista).
- **Controlados:** Portaria SVS/MS nº 344/1998, Anexo I, listas **A1, A2, A3, B1, B2, C1, C2, C3, C4, C5** (as que exigem notificação ou receita de controle especial), na atualização mais recente publicada no DOU (a Anvisa publica a lista atualizada por RDC; use a última).

Formato (UTF-8, uma substância por linha, nome como na lista oficial, em maiúsculas; sais e ésteres só quando a lista os nomeia à parte; nada de comentário no CSV):

```csv
substance
AMICACINA
AMOXICILINA
```

```csv
substance,list
ALFENTANILA,A1
CODEÍNA,A2
DIAZEPAM,B1
```

e `config/medications/anvisa/SOURCES.yml`:

```yaml
antimicrobials:
  label: "RDC Anvisa nº 471/2021, Anexo I (redação vigente em <data da leitura>)"
  read_at: "<AAAA-MM-DD>"
  source: "<endereço da página ou do PDF oficial lido>"
  count: <número de linhas do CSV, sem o cabeçalho>
controlled:
  label: "Portaria SVS/MS nº 344/1998, Anexo I, listas A1–C5, atualizada pela RDC nº <número>/<ano>"
  read_at: "<AAAA-MM-DD>"
  source: "<endereço>"
  counts: { A1: <n>, A2: <n>, A3: <n>, B1: <n>, B2: <n>, C1: <n>, C2: <n>, C3: <n>, C4: <n>, C5: <n> }
```

Os `<…>` acima são os valores que o executor lê na fonte — o arquivo final não tem nenhum. Uma substância que aparece em duas listas de controle fica uma vez, na lista mais restritiva (A antes de B antes de C). A lista é **pendência de go-live**: o farmacêutico da cidade piloto confere antes de ligar o interruptor (Task 25).

- [ ] **Step 2: Os fixtures de teste (pequenos, conhecidos)**

```csv
substance
AMOXICILINA
AZITROMICINA
CEFALEXINA
CIPROFLOXACINO
NITROFURANTOÍNA
SULFAMETOXAZOL
TRIMETOPRIMA
```
(`spec/fixtures/medications/anvisa/antimicrobials.csv`)

```csv
substance,list
CLONAZEPAM,B1
CODEÍNA,A2
DIAZEPAM,B1
FLUOXETINA,C1
METADONA,A1
MORFINA,A1
```
(`spec/fixtures/medications/anvisa/controlled.csv`)

```yaml
antimicrobials:
  label: "Fixture de teste (recorte da RDC 471/2021)"
  read_at: "2026-10-09"
  source: "spec/fixtures"
  count: 7
controlled:
  label: "Fixture de teste (recorte da Portaria 344/1998)"
  read_at: "2026-10-09"
  source: "spec/fixtures"
  counts: { A1: 2, A2: 1, B1: 2, C1: 1 }
```
(`spec/fixtures/medications/anvisa/SOURCES.yml`)

- [ ] **Step 3: Escreva as specs que falham**

```ruby
# spec/config/anvisa_lists_spec.rb
require "rails_helper"

# ADR 0033: as listas transcritas da fonte oficial (Task 5) batem com o que o
# SOURCES.yml diz ter lido, e contêm as substâncias que a APS mais encontra.
RSpec.describe "config/medications/anvisa" do
  dir = Rails.root.join("config/medications/anvisa")
  sources = YAML.load_file(dir.join("SOURCES.yml"))
  antimicrobials = CSV.read(dir.join("antimicrobials.csv"), headers: true).map { |row| row["substance"] }
  controlled = CSV.read(dir.join("controlled.csv"), headers: true).map { |row| [ row["substance"], row["list"] ] }

  it "as contagens batem com a fonte declarada e nada se repete" do
    expect(antimicrobials.size).to eq(sources.dig("antimicrobials", "count"))
    expect(controlled.group_by(&:last).transform_values(&:size)).to eq(sources.dig("controlled", "counts").transform_keys(&:to_s))
    expect(antimicrobials).to eq(antimicrobials.uniq)
    expect(controlled.map(&:first)).to eq(controlled.map(&:first).uniq)
    expect((antimicrobials + controlled.map(&:first)).all? { |name| name == name.upcase && name.strip == name }).to be(true)
    expect(controlled.map(&:last).uniq - %w[A1 A2 A3 B1 B2 C1 C2 C3 C4 C5]).to eq([])
    expect(sources.to_yaml).not_to include("<")
  end

  it "traz as substâncias de referência da APS" do
    keys = antimicrobials.map { |name| Medications::Substances.key(name) }
    expect(keys).to include("AMOXICILINA", "AZITROMICINA", "CEFALEXINA", "CIPROFLOXACINO", "NITROFURANTOINA", "SULFAMETOXAZOL")
    lists = controlled.to_h { |name, list| [ Medications::Substances.key(name), list ] }
    expect(lists).to include("DIAZEPAM" => "B1", "CLONAZEPAM" => "B1", "FENOBARBITAL" => "B1", "MORFINA" => "A1",
                             "METADONA" => "A1", "CODEINA" => "A2", "TRAMADOL" => "A2", "FLUOXETINA" => "C1",
                             "AMITRIPTILINA" => "C1", "CARBAMAZEPINA" => "C1")
  end
end
```

```ruby
# spec/services/medications/substances_spec.rb
require "rails_helper"

# ADR 0033: marca por nome normalizado, palavra inteira no começo do
# ingrediente; em associação, qualquer ingrediente marca; o texto livre é
# varrido por palavra inteira; a lista de controle mais restritiva vence.
RSpec.describe Medications::Substances do
  let(:matcher) do
    described_class::Matcher.new(antimicrobials: %w[AMOXICILINA CIPROFLOXACINO SULFAMETOXAZOL],
                                 controlled: { "DIAZEPAM" => "B1", "CODEINA" => "A2", "FLUOXETINA" => "C1" })
  end

  it "normaliza" do
    expect(described_class.key(" Codeína  fosfato ")).to eq("CODEINA FOSFATO")
    expect(described_class.key("ÁCIDO FÓLICO")).to eq("ACIDO FOLICO")
  end

  it "marca por ingrediente, com sal e em associação" do
    expect(matcher.flags_for([ "CIPROFLOXACINO CLORIDRATO" ])).to have_attributes(antimicrobial: true, controlled: false)
    expect(matcher.flags_for([ "PARACETAMOL", "CODEÍNA" ])).to have_attributes(controlled: true, controlled_list: "A2")
    expect(matcher.flags_for([ "FLUOXETINA", "DIAZEPAM" ]).controlled_list).to eq("B1")
    expect(matcher.flags_for([ "AMOXICILINAS" ]).antimicrobial).to be(false) # palavra inteira
    expect(matcher.flags_for([ "LOSARTANA POTÁSSICA" ])).to eq(described_class::Flags.new(antimicrobial: false, controlled: false, controlled_list: nil))
  end

  it "varre texto livre por palavra inteira" do
    expect(matcher.scan("Diazepam 10mg 1 cp à noite").controlled).to be(true)
    expect(matcher.scan("amoxicilina 500 mg 8/8h").antimicrobial).to be(true)
    expect(matcher.scan("Chá de camomila").controlled).to be(false)
  end

  it "sem release ativa o matcher é vazio" do
    expect(described_class.matcher(nil)).to be_empty
  end
end
```

```ruby
# spec/services/medications/anvisa_import_spec.rb
require "rails_helper"

RSpec.describe Medications::AnvisaImport do
  it "importa as duas listas, ativa a release, audita só contagens e remarca o catálogo" do
    item = import_test_catalog![267_197] # DIAZEPAM, já marcado pela importação do catálogo
    expect(item).to have_attributes(controlled: true, controlled_list: "B1")
    release = AnvisaListRelease.current
    expect(release).to have_attributes(antimicrobials_count: 7, controlled_count: 6)
    expect(release.sources.dig("antimicrobials", "label")).to include("Fixture")
    event = PlatformEvent.where(name: "anvisa_lists.imported").last
    expect(event.payload).to eq("release_id" => release.id, "antimicrobials" => 7, "controlled" => 6)
  end

  it "reimportar substitui a ativa e remarca itens que mudaram de lista" do
    import_test_catalog!
    first = AnvisaListRelease.current
    Dir.mktmpdir do |dir|
      FileUtils.cp_r("#{anvisa_fixture_dir}/.", dir)
      File.write(File.join(dir, "controlled.csv"), "substance,list\nCODEÍNA,A2\nMORFINA,A1\nMETADONA,A1\nCLONAZEPAM,B1\nFLUOXETINA,C1\n")
      sources = YAML.load_file(File.join(dir, "SOURCES.yml"))
      sources["controlled"]["counts"] = { "A1" => 2, "A2" => 1, "B1" => 1, "C1" => 1 }
      File.write(File.join(dir, "SOURCES.yml"), sources.to_yaml)
      expect(described_class.call(by: "rspec", dir: dir)).to be_ok
    end
    expect(first.reload.status).to eq("superseded")
    expect(catalog_item(267_197)).to have_attributes(controlled: false, controlled_list: nil)
  end

  it "arquivo que não bate com o SOURCES.yml falha sem ativar nada" do
    Dir.mktmpdir do |dir|
      FileUtils.cp_r("#{anvisa_fixture_dir}/.", dir)
      File.write(File.join(dir, "antimicrobials.csv"), "substance\nAMOXICILINA\n")
      result = described_class.call(by: "rspec", dir: dir)
      expect([ result.reason, AnvisaListRelease.current ]).to eq([ :invalid_file, nil ])
      expect(result.message).to include("antimicrobials")
    end
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `rspec spec/config/anvisa_lists_spec.rb spec/services/medications/substances_spec.rb spec/services/medications/anvisa_import_spec.rb`
Expected: FAIL (constantes inexistentes; o import_spec depende também da Task 6 — ele passa no fim da Task 6).

- [ ] **Step 5: Implemente**

```ruby
# app/services/medications/substances.rb
# Marcas de antimicrobiano (RDC 471/2021) e de controlado (Portaria 344/1998)
# pelas listas da Anvisa importadas (ADR 0033). Casamento por nome
# normalizado, palavra inteira no começo do ingrediente ("CIPROFLOXACINO"
# casa "CIPROFLOXACINO CLORIDRATO"); em associação, qualquer ingrediente
# marca; a lista de controle mais restritiva vence (A1 < … < C5). O texto
# livre da receita é varrido por palavra inteira.
module Medications
  module Substances
    Flags = Data.define(:antimicrobial, :controlled, :controlled_list)
    NONE = Flags.new(antimicrobial: false, controlled: false, controlled_list: nil)

    module_function

    def key(name) = I18n.transliterate(name.to_s).upcase.gsub(/[^A-Z0-9 ]/, " ").squish

    def matcher(release = AnvisaListRelease.current)
      return Matcher.new(antimicrobials: [], controlled: {}) unless release

      rows = AnvisaListSubstance.where(release_id: release.id).pluck(:kind, :substance_key, :controlled_list)
      Matcher.new(antimicrobials: rows.select { |kind, *| kind == "antimicrobial" }.map(&:second),
                  controlled: rows.select { |kind, *| kind == "controlled" }.to_h { |_kind, substance, list| [ substance, list ] })
    end

    # Reaplica as marcas a todo o catálogo (depois de importar listas ou itens).
    def apply_to_catalog!(matcher)
      changed = 0
      MedicationCatalogItem.find_each do |item|
        flags = matcher.flags_for(item.ingredients)
        next if [ item.antimicrobial, item.controlled, item.controlled_list ] == [ flags.antimicrobial, flags.controlled, flags.controlled_list ]

        item.update!(antimicrobial: flags.antimicrobial, controlled: flags.controlled, controlled_list: flags.controlled_list)
        changed += 1
      end
      changed
    end

    class Matcher
      def initialize(antimicrobials:, controlled:)
        @antimicrobials = antimicrobials.map { |name| Substances.key(name) }.uniq
        @controlled = controlled.transform_keys { |name| Substances.key(name) }
      end

      def empty? = @antimicrobials.empty? && @controlled.empty?

      def flags_for(ingredients)
        keys = Array(ingredients).map { |name| Substances.key(name) }
        lists = keys.flat_map { |ingredient| @controlled.filter_map { |substance, list| list if starts?(ingredient, substance) } }
        Flags.new(antimicrobial: keys.any? { |ingredient| @antimicrobials.any? { |substance| starts?(ingredient, substance) } },
                  controlled: lists.any?, controlled_list: lists.min)
      end

      def scan(text)
        padded = " #{Substances.key(text)} "
        lists = @controlled.filter_map { |substance, list| list if padded.include?(" #{substance} ") }
        Flags.new(antimicrobial: @antimicrobials.any? { |substance| padded.include?(" #{substance} ") },
                  controlled: lists.any?, controlled_list: lists.min)
      end

      def inspect = "#<Medications::Substances::Matcher #{@antimicrobials.size}/#{@controlled.size}>"

      private

      def starts?(ingredient, substance) = ingredient == substance || ingredient.start_with?("#{substance} ")
    end
  end
end
```

```ruby
# app/services/medications/anvisa_import.rb
# Importação das listas da Anvisa (ADR 0033; contrato §8 importAnvisaLists):
# os arquivos versionados em config/medications/anvisa (transcritos da fonte
# oficial, com SOURCES.yml dizendo de onde e quantos). A release nasce
# `importing` FORA da transação (a falha fica registrada); substâncias,
# ativação, marcas do catálogo e auditoria numa transação só.
require "csv"

module Medications
  module AnvisaImport
    DIR = Rails.root.join("config/medications/anvisa")
    LISTS = %w[A1 A2 A3 B1 B2 C1 C2 C3 C4 C5].freeze
    class Invalid < StandardError; end

    module_function

    def call(by:, dir: DIR)
      dir = Pathname(dir)
      sources, antimicrobials, controlled = read(dir)
      AnvisaListRelease.where(status: "importing", imported_at: ...1.hour.ago).find_each { |old| old.update!(status: "failed") }
      release = begin
        AnvisaListRelease.create!(imported_by: by.to_s.first(80), imported_at: Time.current, status: "importing",
                                  source_sha256: sha256(dir), sources: sources)
      rescue ActiveRecord::RecordNotUnique
        return Result.fail(:import_in_progress)
      end
      PlatformRecord.transaction(requires_new: true) do
        rows = antimicrobials.map { |name| { release_id: release.id, kind: "antimicrobial", substance: name, substance_key: Substances.key(name), controlled_list: nil } } +
               controlled.map { |name, list| { release_id: release.id, kind: "controlled", substance: name, substance_key: Substances.key(name), controlled_list: list } }
        rows.each_slice(1_000) { |slice| AnvisaListSubstance.insert_all!(slice) }
        AnvisaListRelease.active.where.not(id: release.id).lock.each { |old| old.update!(status: "superseded") }
        release.update!(status: "active", activated_at: Time.current, antimicrobials_count: antimicrobials.size,
                        controlled_count: controlled.size)
        Substances.apply_to_catalog!(Substances.matcher(release))
        Platform.audit("anvisa_lists.imported", release_id: release.id, antimicrobials: antimicrobials.size,
                                                controlled: controlled.size)
      end
      Result.ok(release: release.reload, antimicrobials: antimicrobials.size, controlled: controlled.size)
    rescue Invalid, Errno::ENOENT, CSV::MalformedCSVError, Psych::SyntaxError => e
      Result.fail(:invalid_file, message: e.message.truncate(300))
    rescue StandardError => e
      mark_failed(release)
      Result.fail(:import_failed, message: e.class.name)
    end

    # Nunca mascara o erro original: se nem o `failed` grava, segue com o motivo.
    def mark_failed(release)
      release&.update!(status: "failed")
    rescue StandardError => e
      Rails.logger.error("[medications:anvisa] não marcou failed: #{e.class}")
    end

    def read(dir)
      sources = YAML.safe_load_file(dir.join("SOURCES.yml"))
      antimicrobials = CSV.read(dir.join("antimicrobials.csv"), headers: true, encoding: "UTF-8").map { |row| row.fetch("substance").to_s.strip }
      controlled = CSV.read(dir.join("controlled.csv"), headers: true, encoding: "UTF-8").map { |row| [ row.fetch("substance").to_s.strip, row.fetch("list").to_s.strip ] }
      raise Invalid, "antimicrobials: contagem #{antimicrobials.size} ≠ SOURCES #{sources.dig('antimicrobials', 'count')}" if antimicrobials.size != sources.dig("antimicrobials", "count")

      counts = controlled.group_by(&:last).transform_values(&:size)
      raise Invalid, "controlled: contagens #{counts} ≠ SOURCES" if counts != sources.dig("controlled", "counts").to_h.transform_keys(&:to_s)
      raise Invalid, "controlled: lista fora de A1–C5" if (counts.keys - LISTS).any?
      raise Invalid, "nome vazio" if (antimicrobials + controlled.map(&:first)).any?(&:blank?)

      [ sources, antimicrobials, controlled ]
    end

    def sha256(dir) = Digest::SHA256.hexdigest(%w[SOURCES.yml antimicrobials.csv controlled.csv].map { |name| File.binread(dir.join(name)) }.join)
    private_class_method :read, :sha256, :mark_failed
  end
end
```

Em `spec/events/platform_event_payload_guard_spec.rb`, acrescente `anvisa_lists.imported` a `R18_PLATFORM_EVENT_NAMES` (junto de `terminology.release_activated`).

- [ ] **Step 6: Rode e veja passar (a do import passa ao fim da Task 6)**

Run: `rspec spec/config/anvisa_lists_spec.rb spec/services/medications/substances_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add config/medications/anvisa/antimicrobials.csv config/medications/anvisa/controlled.csv config/medications/anvisa/SOURCES.yml app/services/medications/substances.rb app/services/medications/anvisa_import.rb spec/fixtures/medications/anvisa/antimicrobials.csv spec/fixtures/medications/anvisa/controlled.csv spec/fixtures/medications/anvisa/SOURCES.yml spec/config/anvisa_lists_spec.rb spec/services/medications/substances_spec.rb spec/services/medications/anvisa_import_spec.rb spec/events/platform_event_payload_guard_spec.rb
git commit -m "feat: import the Anvisa antimicrobial and controlled substance lists

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Importação do catálogo com diferença (entrou, mudou, saiu)

**Files:**
- Create: `app/services/medications/catalog_import.rb`, `lib/tasks/medications.rake`
- Modify: `spec/events/platform_event_payload_guard_spec.rb` (`medication_catalog.imported`)
- Test: `spec/services/medications/catalog_import_spec.rb`

**Interfaces:**
- Consumes: `Medications::Catmat::{Client,FixtureClient,DescriptionParser,Row}`, `Medications::Substances.matcher` (Tasks 3–5).
- Produces:
  - `Medications::CatalogImport.call(by:, client: Medications::Catmat::Client.new, now: Time.current) -> Result` — `ok(release:, added:, changed:, removed:)` | `fail(:import_in_progress | :source_unavailable | :incomplete_source | :suspicious_drop | :import_failed, message:)` (uma importação por vez: índice único em `importing`; a presa há mais de 1 h vira `failed`).
  - `Medications::CatalogImport.search_text(parsed, description) -> String`.
  - `Platform.audit("medication_catalog.imported", release_id:, added:, changed:, removed:)`.
  - Rake: `bin/rails medications:import_anvisa`, `bin/rails medications:import_catmat`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/medications/catalog_import_spec.rb
require "rails_helper"

# ADR 0033 (spec §3, §9): release com diferença; item estável por código; a
# fonte que falha no meio nunca derruba o catálogo ativo (Review Focus 1).
RSpec.describe Medications::CatalogImport do
  before { Medications::AnvisaImport.call(by: "rspec", dir: anvisa_fixture_dir) }

  def import!(rows = catmat_rows, **stub)
    stub_catmat!(rows, **stub)
    described_class.call(by: "rspec", client: Medications::Catmat::Client.new(sleeper: ->(_) {}))
  end

  it "primeira importação: tudo entra, revisão escondida, marcas Anvisa, auditoria só com contagens" do
    result = import!
    expect(result.payload.slice(:added, :changed, :removed)).to eq(added: 17, changed: 0, removed: 0)
    release = result.payload[:release]
    expect(release).to have_attributes(status: "active", items_count: 17, hidden_count: 1, added_count: 17)
    expect(catalog_item(267_778)).to have_attributes(active_ingredient: "PARACETAMOL", strength: "500 MG", dosage_form: "comprimido",
                                                     parse_status: "ok", hidden: false, status: "active", obm_code: nil)
    expect(catalog_item(384_258)).to have_attributes(parse_status: "review", hidden: true, review_reason: "manipulated")
    expect(catalog_item(271_089)).to have_attributes(antimicrobial: true, controlled: false)
    expect(catalog_item(273_009)).to have_attributes(controlled: true, controlled_list: "C1")
    expect(catalog_item(267_778).search_text).to include("PARACETAMOL 500 MG COMPRIMIDO")
    expect(PlatformEvent.where(name: "medication_catalog.imported").last.payload)
      .to eq("release_id" => release.id, "added" => 17, "changed" => 0, "removed" => 0)
  end

  it "segunda importação: mudou, saiu e entrou; o id do item não muda; a anterior é substituída" do
    first = import!.payload[:release]
    losartana_id = catalog_item(268_856).id
    rows = catmat_rows.reject { |row| row["codigoItem"] == 267_772 }
    rows.find { |row| row["codigoItem"] == 268_856 }["descricaoItem"] = "LOSARTANA POTÁSSICA, CONCENTRAÇÃO: 50 MG, FORMA FARMACÊUTICA: COMPRIMIDO"
    rows << { "codigoItem" => 267_506, "codigoPdm" => 354, "nomePdm" => "ALBENDAZOL", "descricaoItem" => "ALBENDAZOL, DOSAGEM: 400 MG",
              "statusItem" => true, "dataHoraAtualizacao" => "2021-10-16T09:43:08.030221" }
    WebMock.reset!
    result = import!(rows)
    expect(result.payload.slice(:added, :changed, :removed)).to eq(added: 1, changed: 1, removed: 1)
    expect(first.reload.status).to eq("superseded")
    losartana = catalog_item(268_856)
    expect([ losartana.id, losartana.dosage_form, losartana.first_release_id ]).to eq([ losartana_id, "comprimido", first.id ])
    expect(catalog_item(267_772)).to have_attributes(status: "removed")
    expect(MedicationCatalogItem.prescribable.where(catmat_code: 267_772)).to be_empty
  end

  it "fonte cai no meio → falha inteira, release failed, catálogo ativo intacto (Review Focus 1)" do
    active = import!.payload[:release]
    WebMock.reset!
    result = import!(catmat_rows, pages: { 2 => { status: 503 } })
    expect(result.reason).to eq(:source_unavailable)
    expect(MedicationCatalogRelease.current).to eq(active)
    expect(MedicationCatalogRelease.order(:created_at).last.status).to eq("failed")
    expect(MedicationCatalogItem.where(status: "removed")).to be_empty
  end

  it "fonte encolhe (menos da metade do ativo) → suspicious_drop; total que muda → incomplete_source" do
    import!
    WebMock.reset!
    expect(import!(catmat_rows.first(5)).reason).to eq(:suspicious_drop)
    WebMock.reset!
    expect(import!(catmat_rows, pages: { 2 => { "resultado" => catmat_rows.last(8), "totalRegistros" => 99,
                                                 "totalPaginas" => 2, "paginasRestantes" => 0 } }).reason).to eq(:incomplete_source)
    expect(MedicationCatalogItem.where(status: "removed")).to be_empty
  end

  it "uma importação por vez: outra em andamento → import_in_progress; presa há mais de 1 h não trava" do
    running = MedicationCatalogRelease.create!(imported_by: "outra aba", imported_at: Time.current, status: "importing")
    expect(import!.reason).to eq(:import_in_progress)
    running.update_columns(imported_at: 2.hours.ago)
    WebMock.reset!
    expect(import!).to be_ok
    expect(running.reload.status).to eq("failed")
  end

  it "o FixtureClient importa sem rede (a semente de dev)" do
    client = Medications::Catmat::FixtureClient.new(Rails.root.join("db/seeds/medications/catmat-dev.json"))
    expect(described_class.call(by: "db:seed", client: client)).to be_ok
    expect(WebMock).not_to have_requested(:get, MedicationHelpers::CATMAT_URL)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/medications/catalog_import_spec.rb`
Expected: FAIL (`uninitialized constant Medications::CatalogImport`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/medications/catalog_import.rb
# Importação do CATMAT classe 6505 (ADR 0033; spec §3; contrato §8
# importMedicationCatalog). A release nasce `importing` FORA da transação (a
# falha fica registrada); a fonte é lida INTEIRA antes de tocar o catálogo
# (Review Focus 1); a diferença (entrou, mudou, saiu), as marcas Anvisa, a
# ativação e a auditoria entram numa transação só. Item estável por código:
# `changed` = descrição diferente (ou volta de `removed`); `removed` = sumiu
# da fonte (nunca apagado). Fonte que encolhe para menos da metade do ativo
# é recusada (`suspicious_drop`): um catálogo some por erro da fonte, não por
# decisão do Ministério.
module Medications
  module CatalogImport
    MIN_SHARE = 0.5
    class SuspiciousDrop < StandardError; end
    UPDATABLE = %i[pdm_code source_description source_updated_at active_ingredient strength dosage_form unit parse_status
                   review_reason hidden antimicrobial controlled controlled_list status search_text last_release_id
                   updated_at].freeze

    module_function

    STALE = 1.hour

    def call(by:, client: Catmat::Client.new, now: Time.current)
      matcher = Substances.matcher
      # A importação que morreu no meio (processo derrubado) não trava a próxima.
      MedicationCatalogRelease.where(status: "importing", imported_at: ...(now - STALE)).find_each { |old| old.update!(status: "failed") }
      release = begin
        MedicationCatalogRelease.create!(source: "catmat_6505", imported_by: by.to_s.first(80), imported_at: now,
                                         status: "importing")
      rescue ActiveRecord::RecordNotUnique
        return Result.fail(:import_in_progress)
      end
      rows = client.fetch_all
      active = MedicationCatalogItem.where(status: "active").count
      raise SuspiciousDrop, "lidos #{rows.size} de #{active} ativos" if active.positive? && rows.size < active * MIN_SHARE

      counts = PlatformRecord.transaction(requires_new: true) { apply!(release, rows, matcher, now) }
      Result.ok(release: release.reload, **counts)
    rescue Catmat::Client::Unavailable => e
      failed(release, :source_unavailable, e)
    rescue Catmat::Client::Incomplete => e
      failed(release, :incomplete_source, e)
    rescue SuspiciousDrop => e
      failed(release, :suspicious_drop, e)
    rescue StandardError => e
      failed(release, :import_failed, e, message: e.class.name)
    end

    def apply!(release, rows, matcher, now)
      existing = MedicationCatalogItem.lock.pluck(:catmat_code, :source_description, :status).to_h { |code, *rest| [ code, rest ] }
      added = changed = 0
      records = rows.map do |row|
        parsed = Catmat::DescriptionParser.call(row.pdm_name, row.description)
        flags = matcher.flags_for(parsed.active_ingredient.split(" + "))
        description = row.description.strip
        before = existing[row.catmat_code]
        if before.nil? then added += 1
        elsif before != [ description, "active" ] then changed += 1
        end
        { catmat_code: row.catmat_code, pdm_code: row.pdm_code, source_description: description, source_updated_at: row.updated_at,
          active_ingredient: parsed.active_ingredient, strength: parsed.strength, dosage_form: parsed.dosage_form, unit: parsed.unit,
          parse_status: parsed.parse_status, review_reason: parsed.review_reason, hidden: !parsed.ok?,
          antimicrobial: flags.antimicrobial, controlled: flags.controlled, controlled_list: flags.controlled_list,
          status: "active", search_text: search_text(parsed, description), first_release_id: release.id,
          last_release_id: release.id, created_at: now, updated_at: now }
      end
      records.each_slice(1_000) { |slice| MedicationCatalogItem.upsert_all(slice, unique_by: :catmat_code, update_only: UPDATABLE, record_timestamps: false) }
      seen = rows.map(&:catmat_code)
      gone = MedicationCatalogItem.where(status: "active").where.not(catmat_code: seen)
      removed = gone.update_all(status: "removed", last_release_id: release.id, updated_at: now)
      MedicationCatalogRelease.active.where.not(id: release.id).lock.each { |old| old.update!(status: "superseded") }
      release.update!(status: "active", activated_at: now, items_count: rows.size,
                      hidden_count: MedicationCatalogItem.where(status: "active", hidden: true).count,
                      added_count: added, changed_count: changed, removed_count: removed)
      Platform.audit("medication_catalog.imported", release_id: release.id, added: added, changed: changed,
                                                    removed: removed)
      { added: added, changed: changed, removed: removed }
    end

    def search_text(parsed, description)
      I18n.transliterate([ parsed.active_ingredient, parsed.strength, parsed.dosage_form, description ].compact.join(" ")).upcase.squish
    end

    # Nunca mascara o erro original; a mensagem não leva corpo de resposta.
    def failed(release, reason, error, message: error.message)
      begin
        release&.update!(status: "failed")
      rescue StandardError => e
        Rails.logger.error("[medications:import] não marcou failed: #{e.class}")
      end
      Result.fail(reason, message: message.to_s.truncate(300))
    end
    private_class_method :apply!, :failed
  end
end
```

> `upsert_all(..., update_only:)` preserva `first_release_id` e `created_at` do item existente (só as colunas de `UPDATABLE` mudam no conflito) — é o que mantém o id e a release de origem estáveis. O `gone.update_all` contorna os callbacks de propósito: não há trigger que barre `status` em itens, só DELETE.

```ruby
# lib/tasks/medications.rake
# Catálogo de medicamentos da plataforma (ADR 0033). O caminho normal é o
# maintenance (importMedicationCatalog / importAnvisaLists); estas tarefas
# servem à operação e ao primeiro provisionamento. As listas Anvisa ANTES do
# catálogo (as marcas saem delas).
namespace :medications do
  desc "Importa as listas da Anvisa de config/medications/anvisa"
  task import_anvisa: :environment do
    result = Medications::AnvisaImport.call(by: "rake")
    abort("[medications] listas: #{result.reason} #{result.message}") if result.failure?
    puts "[medications] listas ativas: #{result.payload[:antimicrobials]} antimicrobianos, #{result.payload[:controlled]} controlados"
  end

  desc "Importa o CATMAT classe 6505 da API aberta do Compras.gov"
  task import_catmat: :environment do
    result = Medications::CatalogImport.call(by: "rake")
    abort("[medications] catálogo: #{result.reason} #{result.message}") if result.failure?
    puts "[medications] catálogo ativo: +#{result.payload[:added]} ~#{result.payload[:changed]} -#{result.payload[:removed]}"
  end
end
```

Em `spec/events/platform_event_payload_guard_spec.rb`, acrescente `medication_catalog.imported` a `R18_PLATFORM_EVENT_NAMES`.

- [ ] **Step 4: Rode e veja passar (com a do import da Anvisa)**

Run: `rspec spec/services/medications spec/events/platform_event_payload_guard_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/medications/catalog_import.rb lib/tasks/medications.rake spec/services/medications/catalog_import_spec.rb spec/events/platform_event_payload_guard_spec.rb
git commit -m "feat: import the CATMAT catalog as a release with added, changed and removed counts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Catálogo e importações na API de manutenção (GraphQL)

**Files:**
- Create: `app/graphql/maintenance/types/medication_catalog_type.rb`, `app/graphql/maintenance/types/medication_catalog_release_type.rb`, `app/graphql/maintenance/types/medication_catalog_item_type.rb`, `app/graphql/maintenance/mutations/import_medication_catalog.rb`, `app/graphql/maintenance/mutations/import_anvisa_lists.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/medication_catalog_spec.rb`

**Interfaces:**
- Consumes: `Medications::{CatalogImport,AnvisaImport}`, `MedicationCatalog{Release,Item}`, `Maintenance::Mutations::BaseMutation#audited`.
- Produces (contrato §8): `Query.medicationCatalog: MedicationCatalog!` → `currentRelease: MedicationCatalogRelease` (`id`, `importedAt`, `itemsCount`, `hiddenCount`) e `reviewItems(first: Int = 50): [MedicationCatalogItem!]!` (`catmatCode`, `sourceDescription`, `parseStatus`); `Mutation.importMedicationCatalog → { ok, errors, releaseId, added, changed, removed }`; `Mutation.importAnvisaLists → { ok, errors, antimicrobials, controlled }`. Os três só para sessão humana. Auditoria `maintenance.medication_catalog.imported` / `maintenance.anvisa_lists.imported` (módulo `medications`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/maintenance/medication_catalog_spec.rb
require "rails_helper"

# Contrato §8: o maintenance lê a release ativa e a lista de revisão e aciona
# as duas importações. Sessão humana só; nenhuma cidade é aberta.
RSpec.describe "Maintenance: catálogo de medicamentos", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "med-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)
  def gql!(query) = post("/graphql", params: { query: query, variables: "{}" }, headers: browser)

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    stub_const("Medications::AnvisaImport::DIR", anvisa_fixture_dir)
    allow(Medications::Catmat::Client).to receive(:new).and_wrap_original { |m, **kw| m.call(**kw, sleeper: ->(_) {}) }
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
  end

  it "importa as listas, depois o catálogo; lê a release e a revisão; audita tentativa e resultado" do
    expect(Maintenance::CityReader).not_to receive(:call)
    gql!("mutation { importAnvisaLists { ok errors { path message } antimicrobials controlled } }")
    expect(json.dig("data", "importAnvisaLists")).to eq("ok" => true, "errors" => [], "antimicrobials" => 7, "controlled" => 6)

    stub_catmat!
    gql!("mutation { importMedicationCatalog { ok errors { message } releaseId added changed removed } }")
    payload = json.dig("data", "importMedicationCatalog")
    expect(payload).to include("ok" => true, "added" => 17, "changed" => 0, "removed" => 0)

    gql!("{ medicationCatalog { currentRelease { id importedAt itemsCount hiddenCount } reviewItems(first: 10) { catmatCode sourceDescription parseStatus } } }")
    catalog = json.dig("data", "medicationCatalog")
    expect(catalog["currentRelease"]).to include("id" => payload["releaseId"], "itemsCount" => 17, "hiddenCount" => 1)
    expect(catalog["reviewItems"]).to eq([ { "catmatCode" => 384_258, "parseStatus" => "review",
                                             "sourceDescription" => catalog_item(384_258).source_description } ])
    names = PlatformEvent.where("name LIKE 'maintenance.%imported'").pluck(:name, Arel.sql("payload->>'outcome'"))
    expect(names).to include([ "maintenance.anvisa_lists.imported", "ok" ], [ "maintenance.medication_catalog.imported", "ok" ])
  end

  it "fonte fora do ar: ok false com o motivo, nada ativo muda" do
    stub_catmat!(catmat_rows, pages: { 1 => { status: 503 } })
    gql!("mutation { importMedicationCatalog { ok errors { path message } releaseId } }")
    expect(json.dig("data", "importMedicationCatalog"))
      .to eq("ok" => false, "errors" => [ { "path" => "catalog", "message" => "source_unavailable" } ], "releaseId" => nil)
    expect(MedicationCatalogRelease.current).to be_nil
  end

  it "token de serviço não alcança os três campos" do
    expect(Maintenance::Analyzers::HumanOnly::RESTRICTED).to include("medicationCatalog", "importMedicationCatalog", "importAnvisaLists")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/maintenance/medication_catalog_spec.rb`
Expected: FAIL (campo `importAnvisaLists` inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/graphql/maintenance/types/medication_catalog_release_type.rb
module Maintenance
  module Types
    class MedicationCatalogReleaseType < BaseObject
      description "Release ativa do catálogo de medicamentos (CATMAT 6505; ADR 0033)"

      field :id, ID, null: false
      field :imported_at, GraphQL::Types::ISO8601DateTime, null: false
      field :items_count, Integer, null: false
      field :hidden_count, Integer, null: false
    end
  end
end
```

```ruby
# app/graphql/maintenance/types/medication_catalog_item_type.rb
module Maintenance
  module Types
    class MedicationCatalogItemType < BaseObject
      description "Item do catálogo em revisão (o analisador não entendeu a descrição)"

      field :catmat_code, Integer, null: false
      field :source_description, String, null: false
      field :parse_status, String, null: false
    end
  end
end
```

```ruby
# app/graphql/maintenance/types/medication_catalog_type.rb
# Catálogo de medicamentos da plataforma (contrato §8): sem cidade, sem dado de pessoa.
module Maintenance
  module Types
    class MedicationCatalogType < BaseObject
      description "Catálogo de medicamentos da plataforma (ADR 0033)"

      field :current_release, MedicationCatalogReleaseType, null: true
      field :review_items, [ MedicationCatalogItemType ], null: false do
        argument :first, Integer, required: false, default_value: 50
      end

      def current_release = MedicationCatalogRelease.current

      def review_items(first:)
        MedicationCatalogItem.where(parse_status: "review", status: "active").order(:catmat_code).limit(first.clamp(1, 200))
      end
    end
  end
end
```

Em `query_type.rb`, depois de `signer_status`:

```ruby
      # ADR 0033 (contrato §8): catálogo de medicamentos da plataforma.
      field :medication_catalog, Types::MedicationCatalogType, null: false,
            description: "Release ativa do CATMAT e itens em revisão"
```
e o resolver `def medication_catalog = {}` (o tipo resolve tudo sozinho).

```ruby
# app/graphql/maintenance/mutations/import_medication_catalog.rb
module Maintenance
  module Mutations
    # ADR 0033 (contrato §8): importa o CATMAT 6505 da API aberta (síncrono; a
    # API leva segundos). Escreve só na plataforma; auditado pela base.
    class ImportMedicationCatalog < BaseMutation
      description "Importa o catálogo de medicamentos (CATMAT classe 6505) e mostra a diferença."

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false
      field :release_id, ID, null: true
      field :added, Integer, null: true
      field :changed, Integer, null: true
      field :removed, Integer, null: true

      def resolve
        outcome = nil
        result = audited(event: "maintenance.medication_catalog.imported", module_name: "medications") do
          outcome = Medications::CatalogImport.call(by: "maintenance:#{credential.maintainer.id}")
          raise Rejected.new(outcome.reason.to_s, path: "catalog") if outcome.failure?
        end
        return result.merge(release_id: nil, added: nil, changed: nil, removed: nil) unless outcome&.ok?

        result.merge(release_id: outcome.payload[:release].id, **outcome.payload.slice(:added, :changed, :removed))
      end
    end
  end
end
```

```ruby
# app/graphql/maintenance/mutations/import_anvisa_lists.rb
module Maintenance
  module Mutations
    # ADR 0033 (contrato §8): importa as listas versionadas em config/medications/anvisa.
    class ImportAnvisaLists < BaseMutation
      description "Importa as listas da Anvisa (antimicrobianos e controlados) e remarca o catálogo."

      field :ok, Boolean, null: false
      field :errors, [ Types::UserErrorType ], null: false
      field :antimicrobials, Integer, null: true
      field :controlled, Integer, null: true

      def resolve
        outcome = nil
        result = audited(event: "maintenance.anvisa_lists.imported", module_name: "medications") do
          outcome = Medications::AnvisaImport.call(by: "maintenance:#{credential.maintainer.id}",
                                                   dir: Medications::AnvisaImport::DIR)
          raise Rejected.new(outcome.reason.to_s, path: "lists") if outcome.failure?
        end
        result.merge(antimicrobials: outcome&.ok? ? outcome.payload[:antimicrobials] : nil,
                     controlled: outcome&.ok? ? outcome.payload[:controlled] : nil)
      end
    end
  end
end
```

`mutation_type.rb`: `field :import_medication_catalog, mutation: Mutations::ImportMedicationCatalog` e `field :import_anvisa_lists, mutation: Mutations::ImportAnvisaLists`.

`human_only.rb`: acrescente `medicationCatalog importMedicationCatalog importAnvisaLists` a `RESTRICTED`.

`app/events/maintenance_audit.rb`: acrescente a `NAMES` `maintenance.medication_catalog.imported` e `maintenance.anvisa_lists.imported`, e ao `case`:

```ruby
    when "maintenance.medication_catalog.imported" then Platform.audit("maintenance.medication_catalog.imported", **payload)
    when "maintenance.anvisa_lists.imported" then Platform.audit("maintenance.anvisa_lists.imported", **payload)
```

`spec/events/platform_event_payload_guard_spec.rb`: os dois nomes em `R18_PLATFORM_EVENT_NAMES`.

`spec/architecture/maintenance_schema_spec.rb` (`EXPECTED_TYPES`): em `"Query"` acrescente `medicationCatalog`; em `"Mutation"`, `importMedicationCatalog importAnvisaLists`; e os tipos novos:

```ruby
    "MedicationCatalog" => %w[currentRelease reviewItems],
    "MedicationCatalogRelease" => %w[id importedAt itemsCount hiddenCount],
    "MedicationCatalogItem" => %w[catmatCode sourceDescription parseStatus],
    "ImportMedicationCatalogPayload" => %w[ok errors releaseId added changed removed],
    "ImportAnvisaListsPayload" => %w[ok errors antimicrobials controlled],
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/maintenance spec/architecture/maintenance_schema_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/graphql`
Expected: PASS. A guarda de cobertura dos analisadores (`spec/graphql/maintenance/analyzers_spec.rb`) exige o campo novo em `RESTRICTED` ou `TOKEN_ALLOWED` — já está.

- [ ] **Step 5: Commit**

```bash
git add app/graphql/maintenance/types/medication_catalog_type.rb app/graphql/maintenance/types/medication_catalog_release_type.rb app/graphql/maintenance/types/medication_catalog_item_type.rb app/graphql/maintenance/mutations/import_medication_catalog.rb app/graphql/maintenance/mutations/import_anvisa_lists.rb app/graphql/maintenance/types/query_type.rb app/graphql/maintenance/types/mutation_type.rb app/graphql/maintenance/analyzers/human_only.rb app/events/maintenance_audit.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/medication_catalog_spec.rb
git commit -m "feat: expose the medication catalog and its imports in the maintenance API

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 2 — Dados da cidade e configuração (F-19.17)

### Task 8: Tabelas da cidade, triggers, cifra, eventos e helpers

**Files:**
- Create: `db/city_migrate/20261009600001_add_clinical_documents.rb`, `app/models/{clinical_document,prescription_item,patient_medication,patient_medication_event,city_medication,nursing_protocol,nursing_protocol_version,nursing_protocol_version_item,document_verification_lookup}.rb`, `app/services/clinical_documents/codes.rb`, `spec/support/clinical_document_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/signature_request.rb`, `app/services/city_encryption.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/rails_helper.rb`
- Test: `spec/models/clinical_document_tables_guard_spec.rb`

**Interfaces:**
- Consumes: `CityEncryption.allowing_reencryption`, `clinical_city!`, `finalized_consultation!`, `screener!`, `staff_with`, `ledi_maintainer!`.
- Produces:
  - Modelos: `ClinicalDocument` (`KINDS`, `MODES`, `STATUSES`, `CANCEL_REASON = (10..500)`, `#content_data -> Hash`, `#issued?`, `#cancelled?`, `#digital?`, `#prescription?`, `#verification_url -> String`, `#signature_request -> SignatureRequest|nil`, `has_many :prescription_items`); `PrescriptionItem` (`ROUTES`); `PatientMedication` (`STATUSES`, `ORIGINS`, `scope :active_medications`, `#active?`); `PatientMedicationEvent` (`KINDS`); `CityMedication` (tabela `city_medication_list`); `NursingProtocol` (`#current_version(on:) -> NursingProtocolVersion|nil`); `NursingProtocolVersion` (`has_many :items`); `NursingProtocolVersionItem`; `DocumentVerificationLookup`.
  - `ClinicalDocuments::Codes.token -> String` (22), `.short_code -> String` (10), `.display(code) -> "XXXXX-XXXXX"`, `.normalize(input) -> String|nil`.
  - `SignatureRequest::REASONS` com `document_cancelled`.
  - Helpers: `documents_city!(enabled: true)`, `nurse!(unit)`, `dentist!(unit)`, `city_profile!`, `city_cnpj!(cnpj = "76001234000115")`, `raw_document!(consultation: nil, attendance: nil, kind: "sick_note", issue_mode: "paper", author: nil, content: {...})`, `municipal_admin!`, `step_up_enrolled!(user)`, e nos request specs `stepped_up!(user) -> Session`, `json_put(path, params)`, `json_delete(path)`, `json_body` (além do `json_post` existente).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/clinical_document_tables_guard_spec.rb
require "rails_helper"

# ADR 0033 (Invariantes): o banco é a última palavra. Documento emitido não
# muda (só issued → cancelled e a volta ao papel digital → paper, sem
# assinatura); itens da receita nascem com o documento; a lista de
# medicamentos só muda com evento da mesma transação; tudo só acréscimo; a
# re-cifra continua possível.
RSpec.describe "Tabelas dos documentos clínicos (triggers)" do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def sql(statement) = ApplicationRecord.connection.execute(statement)

  it "documento nasce emitido e só passa a cancelado, com motivo, data e usuário" do
    document = raw_document!(consultation: consultation)
    expect { document.update_columns(issue_mode: "digital") }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
    expect { document.update!(content: { "type" => "leave", "days" => 9 }.to_json) }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
    expect { document.update!(status: "cancelled") }.to raise_error(ActiveRecord::StatementInvalid, /ck_clinical_documents_cancellation/)
    document.update!(status: "cancelled", cancel_reason: "Emitido para o paciente errado", cancelled_at: Time.current,
                     cancelled_by_user_id: doctor.id)
    expect { document.update!(status: "issued", cancel_reason: nil, cancelled_at: nil, cancelled_by_user_id: nil) }
      .to raise_error(ActiveRecord::StatementInvalid, /never changes/)
    expect { document.destroy }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
    expect { sql("TRUNCATE clinical_documents CASCADE") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { raw_document!(consultation: consultation).tap { |d| d.update_columns(status: "cancelled") } }
      .to raise_error(ActiveRecord::StatementInvalid)
  end

  it "nasce emitido; documento de consulta exige CBO; fora da consulta só papel e só declaração" do
    expect { ClinicalDocument.create!(raw_document!(consultation: consultation).attributes.except("id", "txid").merge("status" => "cancelled", "verification_token" => ClinicalDocuments::Codes.token, "short_code" => ClinicalDocuments::Codes.short_code, "cancel_reason" => "x" * 10, "cancelled_at" => Time.current, "cancelled_by_user_id" => doctor.id)) }
      .to raise_error(ActiveRecord::StatementInvalid, /born issued/)
    attendance = consultation.attendance
    expect { raw_document!(attendance: attendance, kind: "sick_note", author: verifier!) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_clinical_documents_consultation/)
    expect { raw_document!(attendance: attendance, kind: "attendance_declaration", issue_mode: "digital", author: verifier!) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_clinical_documents_paper_outside_consultation/)
    expect(raw_document!(attendance: attendance, kind: "attendance_declaration", author: verifier!).cbo_code).to be_nil
  end

  it "volta ao papel: digital → paper sem assinatura; nunca paper → digital" do
    document = raw_document!(consultation: consultation, issue_mode: "digital")
    document.update!(issue_mode: "paper")
    expect { document.update!(issue_mode: "digital") }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
  end

  it "itens da receita nascem na transação do documento e nunca mudam" do
    item = ApplicationRecord.transaction do
      document = raw_document!(consultation: consultation, kind: "prescription", content: { "items" => [] })
      PrescriptionItem.create!(clinical_document: document, position: 1, free_text: "Chá de camomila",
                               printed_description: "Chá de camomila", quantity: 1, quantity_unit: "sachê",
                               route: "oral", dosage_instructions: "1 sachê à noite")
    end
    expect { item.update!(quantity: 2) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { item.destroy }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
  end

  it "item de receita de documento de outra transação ou de outro tipo é recusado" do
    document = raw_document!(consultation: consultation, kind: "sick_note")
    expect { PrescriptionItem.create!(clinical_document: document, position: 1, free_text: "x" * 3, printed_description: "x",
                                      quantity: 1, quantity_unit: "cp", route: "oral", dosage_instructions: "1 cp") }
      .to raise_error(ActiveRecord::StatementInvalid, /born with their prescription/)
  end

  it "medicamento em uso só nasce e muda com evento da mesma transação" do
    expect { PatientMedication.create!(patient: consultation.patient, free_text: "Chá", label: "Chá", status: "active", origin: "external") }
      .to raise_error(ActiveRecord::StatementInvalid, /event of the same transaction/)
  end

  it "re-cifra: o conteúdo cifrado pode ser regravado sob a marca; nada mais" do
    document = raw_document!(consultation: consultation)
    CityEncryption.allowing_reencryption { document.update_columns(content: document.content) }
    expect(document.reload.content_data["days"]).to eq(1)
    expect { document.update_columns(content: document.content) }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
  end

  it "protocolos e conferências são só acréscimo; itens nascem com a versão" do
    protocol = NursingProtocol.create!(title: "Saúde da mulher", number: "PE-01", year: 2026, created_by_user_id: doctor.id)
    version = NursingProtocolVersion.create!(nursing_protocol: protocol, version: 1, valid_from: Date.new(2026, 1, 1),
                                             created_by_user_id: doctor.id)
    NursingProtocolVersionItem.create!(nursing_protocol_version: version, catalog_item_id: SecureRandom.uuid)
    expect { protocol.update!(title: "Outro") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { version.update!(valid_until: Date.new(2026, 12, 31)) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    lookup = DocumentVerificationLookup.create!(via: "short_code", outcome: "not_found")
    expect { lookup.destroy }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect(protocol.current_version(on: Date.new(2026, 10, 9))).to eq(version)
  end

  it "o pedido de assinatura aceita o documento clínico e o motivo document_cancelled" do
    document = raw_document!(consultation: consultation, issue_mode: "digital")
    request = SignatureRequest.create!(document_type: "ClinicalDocument", document_id: document.id,
                                       consultation_id: consultation.id, author_user_id: doctor.id)
    request.update!(status: "returned_to_paper", reason_code: "document_cancelled", resolved_at: Time.current)
    expect(SignatureRequest::REASONS).to include("document_cancelled")
  end
end
```

> As specs rodam dentro da transação do exemplo (fixture transacional): ali todo documento e todo item nascem na mesma transação, então só a recusa por **tipo** é provável aqui. A recusa por **outra** transação é provada na suíte de invariantes (Task 23), que roda com `use_transactional_tests = false`.

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/models/clinical_document_tables_guard_spec.rb`
Expected: FAIL (`undefined method 'raw_document!'`).

- [ ] **Step 3: A migração de cidade**

```ruby
# db/city_migrate/20261009600001_add_clinical_documents.rb
# Documentos clínicos da consulta (ADR 0033; spec 2026-10-09 §3–§8). Cabeçalho
# comum (clinical_documents, conteúdo JSON cifrado), itens da receita, lista de
# medicamentos em uso por eventos, REMUME, protocolos de enfermagem com versões,
# trilha da página pública e o CNPJ da cidade. O 19b passa a aceitar o
# documento clínico como assinável e o motivo document_cancelled. As tabelas
# do 19a não mudam. Triggers em db/city_triggers.sql.
class AddClinicalDocuments < ActiveRecord::Migration[8.1]
  KINDS = %w[sick_note attendance_declaration prescription exam_requisition].freeze
  ROUTES = %w[oral sublingual topical ophthalmic otic nasal inhalation vaginal rectal intramuscular intravenous
              subcutaneous other].freeze
  SIGNABLE = %w[Consultation ConsultationAddendum ClinicalDocument].freeze
  SIGNATURE_REASONS = %w[no_session session_expired provider_unavailable provider_rejected signer_unavailable
                         verification_failed certificate_expired certificate_revoked certificate_cpf_mismatch
                         feature_disabled user_request document_cancelled].freeze

  def text_in(column, values) = "#{column}::text = ANY (ARRAY[#{values.map { |v| "'#{v}'::text" }.join(', ')}])"

  def checks(table, map) = map.each { |name, expression| add_check_constraint table, expression, name: name }

  def up
    add_column :city_profile, :cnpj, :string, limit: 14
    add_check_constraint :city_profile, "cnpj IS NULL OR cnpj::text ~ '^[0-9A-Z]{12}[0-9]{2}$'::text", name: "ck_city_profile_cnpj"

    create_table :city_medication_list, id: :uuid do |t|
      t.uuid :catalog_item_id, null: false
      t.uuid :unit_ids, array: true, null: false, default: []
      t.uuid :added_by_user_id, null: false
      t.timestamps
    end
    add_index :city_medication_list, :catalog_item_id, unique: true
    add_index :city_medication_list, :added_by_user_id
    add_foreign_key :city_medication_list, :users, column: :added_by_user_id

    create_table :nursing_protocols, id: :uuid do |t|
      t.string :title, null: false
      t.string :number, null: false, limit: 30
      t.integer :year, null: false
      t.uuid :created_by_user_id, null: false
      t.datetime :created_at, null: false
    end
    add_index :nursing_protocols, %i[number year], unique: true
    add_index :nursing_protocols, :created_by_user_id
    add_foreign_key :nursing_protocols, :users, column: :created_by_user_id
    checks(:nursing_protocols,
           "ck_nursing_protocols_title" => "length(btrim(title::text)) BETWEEN 3 AND 200",
           "ck_nursing_protocols_number" => "length(btrim(number::text)) >= 1",
           "ck_nursing_protocols_year" => "year BETWEEN 1990 AND 2100")

    create_table :nursing_protocol_versions, id: :uuid do |t|
      t.uuid :nursing_protocol_id, null: false
      t.integer :version, null: false
      t.date :valid_from, null: false
      t.date :valid_until
      t.uuid :created_by_user_id, null: false
      t.bigint :txid, null: false, default: -> { "txid_current()" }
      t.datetime :created_at, null: false
    end
    add_index :nursing_protocol_versions, %i[nursing_protocol_id version], unique: true, name: "idx_nursing_protocol_versions_unique"
    add_index :nursing_protocol_versions, :created_by_user_id
    add_foreign_key :nursing_protocol_versions, :nursing_protocols
    add_foreign_key :nursing_protocol_versions, :users, column: :created_by_user_id
    checks(:nursing_protocol_versions,
           "ck_nursing_protocol_versions_validity" => "valid_until IS NULL OR valid_until >= valid_from",
           "ck_nursing_protocol_versions_version" => "version >= 1")

    create_table :nursing_protocol_version_items, id: :uuid do |t|
      t.uuid :nursing_protocol_version_id, null: false
      t.uuid :catalog_item_id, null: false
      t.decimal :max_quantity, precision: 10, scale: 2
      t.string :max_quantity_unit, limit: 30
      t.datetime :created_at, null: false
    end
    add_index :nursing_protocol_version_items, %i[nursing_protocol_version_id catalog_item_id], unique: true,
                                                                                                name: "idx_nursing_protocol_version_items_unique"
    add_foreign_key :nursing_protocol_version_items, :nursing_protocol_versions
    checks(:nursing_protocol_version_items,
           "ck_nursing_protocol_version_items_max" => "(max_quantity IS NULL) = (max_quantity_unit IS NULL) AND (max_quantity IS NULL OR max_quantity > 0)")

    create_table :clinical_documents, id: :uuid do |t|
      t.string :kind, null: false
      t.uuid :patient_id
      t.uuid :citizen_id, null: false
      t.uuid :consultation_id
      t.uuid :attendance_id, null: false
      t.uuid :author_user_id, null: false
      t.string :cbo_code
      t.string :issue_mode, null: false
      t.string :status, null: false, default: "issued"
      t.text :cancel_reason
      t.datetime :cancelled_at
      t.uuid :cancelled_by_user_id
      t.string :verification_token, null: false
      t.string :short_code, null: false, limit: 10
      t.uuid :replaces_document_id
      t.datetime :issued_at, null: false
      t.text :content, null: false
      t.bigint :txid, null: false, default: -> { "txid_current()" }
      t.datetime :created_at, null: false
    end
    add_index :clinical_documents, :verification_token, unique: true
    add_index :clinical_documents, :short_code, unique: true
    add_index :clinical_documents, :replaces_document_id, unique: true, where: "replaces_document_id IS NOT NULL"
    %i[patient_id citizen_id consultation_id attendance_id author_user_id cancelled_by_user_id].each do |column|
      add_index :clinical_documents, column
    end
    add_foreign_key :clinical_documents, :patients
    add_foreign_key :clinical_documents, :citizens
    add_foreign_key :clinical_documents, :consultations
    add_foreign_key :clinical_documents, :attendances
    add_foreign_key :clinical_documents, :users, column: :author_user_id
    add_foreign_key :clinical_documents, :users, column: :cancelled_by_user_id
    add_foreign_key :clinical_documents, :clinical_documents, column: :replaces_document_id
    cancelled = "(status::text = 'cancelled'::text)"
    checks(:clinical_documents,
           "ck_clinical_documents_kind" => text_in("kind", KINDS),
           "ck_clinical_documents_issue_mode" => text_in("issue_mode", %w[digital paper]),
           "ck_clinical_documents_status" => text_in("status", %w[issued cancelled]),
           "ck_clinical_documents_cancellation" => "#{cancelled} = (cancelled_at IS NOT NULL) AND #{cancelled} = (cancel_reason IS NOT NULL) AND #{cancelled} = (cancelled_by_user_id IS NOT NULL)",
           "ck_clinical_documents_short_code" => "short_code::text ~ '^[2-9ABCDEFGHJKMNPQRSTUVWXYZ]{10}$'::text",
           "ck_clinical_documents_token" => "verification_token::text ~ '^[A-Za-z0-9_-]{22}$'::text",
           "ck_clinical_documents_consultation" => "consultation_id IS NOT NULL OR kind::text = 'attendance_declaration'::text",
           "ck_clinical_documents_patient" => "patient_id IS NOT NULL OR kind::text = 'attendance_declaration'::text",
           "ck_clinical_documents_cbo" => "cbo_code IS NULL OR cbo_code::text ~ '^[0-9]{6}$'::text",
           "ck_clinical_documents_cbo_required" => "consultation_id IS NULL OR cbo_code IS NOT NULL",
           "ck_clinical_documents_paper_outside_consultation" => "consultation_id IS NOT NULL OR issue_mode::text = 'paper'::text")

    create_table :prescription_items, id: :uuid do |t|
      t.uuid :clinical_document_id, null: false
      t.integer :position, null: false
      t.uuid :medication_catalog_item_id
      t.integer :catmat_code
      t.uuid :catalog_release_id
      t.text :free_text
      t.text :printed_description, null: false
      t.decimal :quantity, precision: 10, scale: 2, null: false
      t.string :quantity_unit, null: false, limit: 30
      t.string :route, null: false
      t.text :dosage_instructions, null: false
      t.integer :duration_days
      t.boolean :continuous, null: false, default: false
      t.boolean :antimicrobial, null: false, default: false
      t.uuid :nursing_protocol_version_item_id
      t.uuid :reason_problem_id
      t.datetime :created_at, null: false
    end
    add_index :prescription_items, %i[clinical_document_id position], unique: true, name: "idx_prescription_items_position"
    add_index :prescription_items, :medication_catalog_item_id
    add_index :prescription_items, :nursing_protocol_version_item_id
    add_index :prescription_items, :reason_problem_id
    add_foreign_key :prescription_items, :clinical_documents
    add_foreign_key :prescription_items, :nursing_protocol_version_items
    add_foreign_key :prescription_items, :patient_problems, column: :reason_problem_id
    checks(:prescription_items,
           "ck_prescription_items_source" => "(medication_catalog_item_id IS NULL) <> (free_text IS NULL)",
           "ck_prescription_items_catalog" => "(medication_catalog_item_id IS NULL) = (catmat_code IS NULL) AND (medication_catalog_item_id IS NULL) = (catalog_release_id IS NULL)",
           "ck_prescription_items_route" => text_in("route", ROUTES),
           "ck_prescription_items_quantity" => "quantity > 0::numeric AND quantity <= 9999::numeric",
           "ck_prescription_items_duration" => "duration_days IS NULL OR duration_days BETWEEN 1 AND 365",
           "ck_prescription_items_position" => "position >= 1")

    create_table :patient_medications, id: :uuid do |t|
      t.uuid :patient_id, null: false
      t.uuid :catalog_item_id
      t.integer :catmat_code
      t.text :free_text
      t.text :label, null: false
      t.text :dosage_summary
      t.boolean :continuous, null: false, default: true
      t.string :status, null: false
      t.string :origin, null: false
      t.date :started_on
      t.timestamps
    end
    add_index :patient_medications, :patient_id
    add_index :patient_medications, %i[patient_id catalog_item_id], unique: true,
                                                                    where: "status::text = 'active'::text AND catalog_item_id IS NOT NULL",
                                                                    name: "idx_patient_medications_one_active"
    add_foreign_key :patient_medications, :patients
    checks(:patient_medications,
           "ck_patient_medications_status" => text_in("status", %w[active suspended]),
           "ck_patient_medications_origin" => text_in("origin", %w[prescription external]),
           "ck_patient_medications_source" => "(catalog_item_id IS NULL) <> (free_text IS NULL) AND (catalog_item_id IS NULL) = (catmat_code IS NULL)")

    create_table :patient_medication_events, id: :uuid do |t|
      t.uuid :patient_medication_id, null: false
      t.string :kind, null: false
      t.uuid :consultation_id
      t.uuid :addendum_id
      t.uuid :clinical_document_id
      t.uuid :user_id, null: false
      t.string :status_after, null: false
      t.text :dosage_summary
      t.bigint :txid, null: false, default: -> { "txid_current()" }
      t.datetime :created_at, null: false
    end
    %i[patient_medication_id consultation_id addendum_id clinical_document_id user_id].each do |column|
      add_index :patient_medication_events, column
    end
    add_foreign_key :patient_medication_events, :patient_medications, deferrable: :deferred
    add_foreign_key :patient_medication_events, :consultations, deferrable: :deferred
    add_foreign_key :patient_medication_events, :consultation_addenda, column: :addendum_id, deferrable: :deferred
    add_foreign_key :patient_medication_events, :clinical_documents, deferrable: :deferred
    add_foreign_key :patient_medication_events, :users
    checks(:patient_medication_events,
           "ck_patient_medication_events_kind" => text_in("kind", %w[added suspended reactivated changed]),
           "ck_patient_medication_events_status" => text_in("status_after", %w[active suspended]),
           "ck_patient_medication_events_source" => "num_nonnulls(consultation_id, addendum_id, clinical_document_id) = 1")

    create_table :document_verification_lookups, id: :uuid do |t|
      t.uuid :clinical_document_id
      t.string :via, null: false
      t.string :outcome, null: false
      t.datetime :created_at, null: false
    end
    add_index :document_verification_lookups, :clinical_document_id
    add_index :document_verification_lookups, :created_at
    add_foreign_key :document_verification_lookups, :clinical_documents
    checks(:document_verification_lookups,
           "ck_document_verification_lookups_via" => text_in("via", %w[token short_code]),
           "ck_document_verification_lookups_outcome" => text_in("outcome", %w[found not_found]),
           "ck_document_verification_lookups_document" => "(outcome::text = 'found'::text) = (clinical_document_id IS NOT NULL)")

    # 19b (ADR 0032, aberto ao 19c): o documento clínico é assinável; o pedido
    # de documento cancelado sai da fila com document_cancelled.
    remove_check_constraint :signature_requests, name: "ck_signature_requests_document_type"
    add_check_constraint :signature_requests, text_in("document_type", SIGNABLE), name: "ck_signature_requests_document_type"
    remove_check_constraint :signature_requests, name: "ck_signature_requests_reason"
    add_check_constraint :signature_requests, "reason_code IS NULL OR #{text_in('reason_code', SIGNATURE_REASONS)}",
                         name: "ck_signature_requests_reason"
    remove_check_constraint :signatures, name: "ck_signatures_document_type"
    add_check_constraint :signatures, text_in("document_type", SIGNABLE), name: "ck_signatures_document_type"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- [ ] **Step 4: Os triggers (fim de `db/city_triggers.sql`)**

```sql
-- Documentos clínicos (ADR 0033, Invariantes; spec 2026-10-09 §4): nascem
-- emitidos; nunca somem; só mudam (a) de issued para cancelled, com motivo,
-- data e usuário e nada mais; (b) de digital para paper, pela volta ao papel
-- do 19b, se ainda não há assinatura gravada; (c) na re-cifra (rota.reencrypting),
-- só o conteúdo e o motivo cifrados. paper → digital nunca.
CREATE OR REPLACE FUNCTION rota_clinical_document_guard() RETURNS trigger AS $fn$
DECLARE
  cancel_cols text[] := ARRAY['status', 'cancel_reason', 'cancelled_at', 'cancelled_by_user_id'];
  encrypted text[] := ARRAY['content', 'cancel_reason'];
BEGIN
  IF TG_OP = 'INSERT' THEN
    IF NEW.status <> 'issued' THEN
      RAISE EXCEPTION 'clinical_documents: a document is born issued';
    END IF;
    RETURN NEW;
  END IF;
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'clinical_documents is append-only: DELETE refused';
  END IF;
  IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - encrypted) = (to_jsonb(OLD) - encrypted) THEN
    RETURN NEW;
  END IF;
  IF OLD.status = 'issued' AND NEW.status = 'cancelled' AND (to_jsonb(NEW) - cancel_cols) = (to_jsonb(OLD) - cancel_cols) THEN
    RETURN NEW;
  END IF;
  IF OLD.issue_mode = 'digital' AND NEW.issue_mode = 'paper' AND (to_jsonb(NEW) - 'issue_mode') = (to_jsonb(OLD) - 'issue_mode')
     AND NOT EXISTS (SELECT 1 FROM signatures s WHERE s.document_type = 'ClinicalDocument' AND s.document_id = NEW.id) THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'clinical_documents: an issued document never changes; it is only cancelled by its author';
END;
$fn$ LANGUAGE plpgsql;

-- Itens da receita: nascem na MESMA transação do documento (que é uma
-- receita) e nunca mudam, salvo a re-cifra dos textos.
CREATE OR REPLACE FUNCTION rota_prescription_item_insert_guard() RETURNS trigger AS $fn$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM clinical_documents d WHERE d.id = NEW.clinical_document_id AND d.kind = 'prescription'
                 AND d.txid = txid_current()) THEN
    RAISE EXCEPTION 'prescription_items are born with their prescription, in the same transaction';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Itens da versão do protocolo: nascem com a versão (mesma transação).
CREATE OR REPLACE FUNCTION rota_nursing_protocol_item_insert_guard() RETURNS trigger AS $fn$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM nursing_protocol_versions v WHERE v.id = NEW.nursing_protocol_version_id
                 AND v.txid = txid_current()) THEN
    RAISE EXCEPTION 'nursing_protocol_version_items are born with their version, in the same transaction';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- Lista de medicamentos em uso (ADR 0033, Invariantes): o estado só nasce e só
-- muda com um evento da MESMA transação com o mesmo status
-- (Patients::ApplyMedicationEvent grava o evento antes; a FK do evento é
-- DEFERRABLE). Identidade fixa; nunca some; a re-cifra regrava só os textos.
CREATE OR REPLACE FUNCTION rota_patient_medication_guard() RETURNS trigger AS $fn$
DECLARE
  encrypted text[] := ARRAY['free_text', 'label', 'dosage_summary'];
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'patient_medications is append-only: DELETE refused';
  END IF;
  IF TG_OP = 'UPDATE' THEN
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - encrypted) = (to_jsonb(OLD) - encrypted) THEN
      RETURN NEW;
    END IF;
    IF NEW.id IS DISTINCT FROM OLD.id OR NEW.patient_id IS DISTINCT FROM OLD.patient_id
       OR NEW.catalog_item_id IS DISTINCT FROM OLD.catalog_item_id OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
      RAISE EXCEPTION 'patient_medications: identity columns never change';
    END IF;
  END IF;
  IF NOT EXISTS (SELECT 1 FROM patient_medication_events e WHERE e.patient_medication_id = NEW.id
                 AND e.txid = txid_current() AND e.status_after = NEW.status) THEN
    RAISE EXCEPTION 'patient_medications: changes only through an event of the same transaction';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
DECLARE
  item text;
BEGIN
  IF to_regclass('public.clinical_documents') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS clinical_documents_guard ON clinical_documents';
    EXECUTE 'CREATE TRIGGER clinical_documents_guard
      BEFORE INSERT OR UPDATE OR DELETE ON clinical_documents
      FOR EACH ROW EXECUTE FUNCTION rota_clinical_document_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS prescription_items_born_with_document ON prescription_items';
    EXECUTE 'CREATE TRIGGER prescription_items_born_with_document
      BEFORE INSERT ON prescription_items
      FOR EACH ROW EXECUTE FUNCTION rota_prescription_item_insert_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS prescription_items_reencryption_only ON prescription_items';
    EXECUTE 'CREATE TRIGGER prescription_items_reencryption_only
      BEFORE UPDATE OR DELETE ON prescription_items
      FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''free_text'', ''printed_description'', ''dosage_instructions'')';
    EXECUTE 'DROP TRIGGER IF EXISTS patient_medications_guard ON patient_medications';
    EXECUTE 'CREATE TRIGGER patient_medications_guard
      BEFORE INSERT OR UPDATE OR DELETE ON patient_medications
      FOR EACH ROW EXECUTE FUNCTION rota_patient_medication_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS patient_medication_events_reencryption_only ON patient_medication_events';
    EXECUTE 'CREATE TRIGGER patient_medication_events_reencryption_only
      BEFORE UPDATE OR DELETE ON patient_medication_events
      FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''dosage_summary'')';
    EXECUTE 'DROP TRIGGER IF EXISTS nursing_protocol_version_items_born_with_version ON nursing_protocol_version_items';
    EXECUTE 'CREATE TRIGGER nursing_protocol_version_items_born_with_version
      BEFORE INSERT ON nursing_protocol_version_items
      FOR EACH ROW EXECUTE FUNCTION rota_nursing_protocol_item_insert_guard()';
    FOREACH item IN ARRAY ARRAY['nursing_protocols', 'nursing_protocol_versions', 'nursing_protocol_version_items',
                                'document_verification_lookups'] LOOP
      EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only', item);
      EXECUTE format('CREATE TRIGGER %I BEFORE UPDATE OR DELETE ON %I FOR EACH ROW EXECUTE FUNCTION rota_append_only()',
                     item || '_append_only', item);
    END LOOP;
    FOREACH item IN ARRAY ARRAY['clinical_documents', 'prescription_items', 'patient_medications', 'patient_medication_events',
                                'nursing_protocols', 'nursing_protocol_versions', 'nursing_protocol_version_items',
                                'document_verification_lookups'] LOOP
      EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only_truncate', item);
      EXECUTE format('CREATE TRIGGER %I BEFORE TRUNCATE ON %I FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()',
                     item || '_append_only_truncate', item);
    END LOOP;
  END IF;
END
$do$;
```

> `rota_append_only()` (mensagem `'% is append-only: % refused'`) e `rota_reencryption_only()` já existem no arquivo (19a); as specs casam `/append-only/`.

- [ ] **Step 5: O dump à mão em `db/city_schema.rb`**

`define(version: 2026_10_09_600001)`. Em `city_profile`, acrescente `t.string "cnpj", limit: 14` (ordem alfabética, antes de `created_at`) e `t.check_constraint "cnpj IS NULL OR cnpj::text ~ '^[0-9A-Z]{12}[0-9]{2}$'::text", name: "ck_city_profile_cnpj"`. Em `signature_requests`, troque os dois CHECKs (`ck_signature_requests_document_type` com `'ClinicalDocument'::text` a mais; `ck_signature_requests_reason` com `'document_cancelled'::text` a mais); em `signatures`, o `ck_signatures_document_type`. Acrescente as tabelas (ordem alfabética entre as existentes):

```ruby
  create_table "city_medication_list", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "added_by_user_id", null: false
    t.uuid "catalog_item_id", null: false
    t.datetime "created_at", null: false
    t.uuid "unit_ids", default: [], null: false, array: true
    t.datetime "updated_at", null: false
    t.index ["added_by_user_id"], name: "index_city_medication_list_on_added_by_user_id"
    t.index ["catalog_item_id"], name: "index_city_medication_list_on_catalog_item_id", unique: true
  end

  create_table "clinical_documents", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "attendance_id", null: false
    t.uuid "author_user_id", null: false
    t.text "cancel_reason"
    t.datetime "cancelled_at"
    t.uuid "cancelled_by_user_id"
    t.string "cbo_code"
    t.uuid "citizen_id", null: false
    t.uuid "consultation_id"
    t.text "content", null: false
    t.datetime "created_at", null: false
    t.string "issue_mode", null: false
    t.datetime "issued_at", null: false
    t.string "kind", null: false
    t.uuid "patient_id"
    t.uuid "replaces_document_id"
    t.string "short_code", limit: 10, null: false
    t.string "status", default: "issued", null: false
    t.bigint "txid", default: -> { "txid_current()" }, null: false
    t.string "verification_token", null: false
    t.index ["attendance_id"], name: "index_clinical_documents_on_attendance_id"
    t.index ["author_user_id"], name: "index_clinical_documents_on_author_user_id"
    t.index ["cancelled_by_user_id"], name: "index_clinical_documents_on_cancelled_by_user_id"
    t.index ["citizen_id"], name: "index_clinical_documents_on_citizen_id"
    t.index ["consultation_id"], name: "index_clinical_documents_on_consultation_id"
    t.index ["patient_id"], name: "index_clinical_documents_on_patient_id"
    t.index ["replaces_document_id"], name: "index_clinical_documents_on_replaces_document_id", unique: true, where: "(replaces_document_id IS NOT NULL)"
    t.index ["short_code"], name: "index_clinical_documents_on_short_code", unique: true
    t.index ["verification_token"], name: "index_clinical_documents_on_verification_token", unique: true
    t.check_constraint "(status::text = 'cancelled'::text) = (cancelled_at IS NOT NULL) AND (status::text = 'cancelled'::text) = (cancel_reason IS NOT NULL) AND (status::text = 'cancelled'::text) = (cancelled_by_user_id IS NOT NULL)", name: "ck_clinical_documents_cancellation"
    t.check_constraint "cbo_code IS NULL OR cbo_code::text ~ '^[0-9]{6}$'::text", name: "ck_clinical_documents_cbo"
    t.check_constraint "consultation_id IS NULL OR cbo_code IS NOT NULL", name: "ck_clinical_documents_cbo_required"
    t.check_constraint "consultation_id IS NOT NULL OR issue_mode::text = 'paper'::text", name: "ck_clinical_documents_paper_outside_consultation"
    t.check_constraint "consultation_id IS NOT NULL OR kind::text = 'attendance_declaration'::text", name: "ck_clinical_documents_consultation"
    t.check_constraint "issue_mode::text = ANY (ARRAY['digital'::text, 'paper'::text])", name: "ck_clinical_documents_issue_mode"
    t.check_constraint "kind::text = ANY (ARRAY['sick_note'::text, 'attendance_declaration'::text, 'prescription'::text, 'exam_requisition'::text])", name: "ck_clinical_documents_kind"
    t.check_constraint "patient_id IS NOT NULL OR kind::text = 'attendance_declaration'::text", name: "ck_clinical_documents_patient"
    t.check_constraint "short_code::text ~ '^[2-9ABCDEFGHJKMNPQRSTUVWXYZ]{10}$'::text", name: "ck_clinical_documents_short_code"
    t.check_constraint "status::text = ANY (ARRAY['issued'::text, 'cancelled'::text])", name: "ck_clinical_documents_status"
    t.check_constraint "verification_token::text ~ '^[A-Za-z0-9_-]{22}$'::text", name: "ck_clinical_documents_token"
  end

  create_table "document_verification_lookups", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "clinical_document_id"
    t.datetime "created_at", null: false
    t.string "outcome", null: false
    t.string "via", null: false
    t.index ["clinical_document_id"], name: "index_document_verification_lookups_on_clinical_document_id"
    t.index ["created_at"], name: "index_document_verification_lookups_on_created_at"
    t.check_constraint "(outcome::text = 'found'::text) = (clinical_document_id IS NOT NULL)", name: "ck_document_verification_lookups_document"
    t.check_constraint "outcome::text = ANY (ARRAY['found'::text, 'not_found'::text])", name: "ck_document_verification_lookups_outcome"
    t.check_constraint "via::text = ANY (ARRAY['token'::text, 'short_code'::text])", name: "ck_document_verification_lookups_via"
  end

  create_table "nursing_protocol_version_items", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "catalog_item_id", null: false
    t.datetime "created_at", null: false
    t.decimal "max_quantity", precision: 10, scale: 2
    t.string "max_quantity_unit", limit: 30
    t.uuid "nursing_protocol_version_id", null: false
    t.index ["nursing_protocol_version_id", "catalog_item_id"], name: "idx_nursing_protocol_version_items_unique", unique: true
    t.check_constraint "(max_quantity IS NULL) = (max_quantity_unit IS NULL) AND (max_quantity IS NULL OR max_quantity > 0::numeric)", name: "ck_nursing_protocol_version_items_max"
  end

  create_table "nursing_protocol_versions", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.uuid "created_by_user_id", null: false
    t.uuid "nursing_protocol_id", null: false
    t.bigint "txid", default: -> { "txid_current()" }, null: false
    t.date "valid_from", null: false
    t.date "valid_until"
    t.integer "version", null: false
    t.index ["created_by_user_id"], name: "index_nursing_protocol_versions_on_created_by_user_id"
    t.index ["nursing_protocol_id", "version"], name: "idx_nursing_protocol_versions_unique", unique: true
    t.check_constraint "valid_until IS NULL OR valid_until >= valid_from", name: "ck_nursing_protocol_versions_validity"
    t.check_constraint "version >= 1", name: "ck_nursing_protocol_versions_version"
  end

  create_table "nursing_protocols", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.uuid "created_by_user_id", null: false
    t.string "number", limit: 30, null: false
    t.string "title", null: false
    t.integer "year", null: false
    t.index ["created_by_user_id"], name: "index_nursing_protocols_on_created_by_user_id"
    t.index ["number", "year"], name: "index_nursing_protocols_on_number_and_year", unique: true
    t.check_constraint "length(btrim(number::text)) >= 1", name: "ck_nursing_protocols_number"
    t.check_constraint "length(btrim(title::text)) >= 3 AND length(btrim(title::text)) <= 200", name: "ck_nursing_protocols_title"
    t.check_constraint "year >= 1990 AND year <= 2100", name: "ck_nursing_protocols_year"
  end

  create_table "patient_medication_events", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "addendum_id"
    t.uuid "clinical_document_id"
    t.uuid "consultation_id"
    t.datetime "created_at", null: false
    t.text "dosage_summary"
    t.string "kind", null: false
    t.uuid "patient_medication_id", null: false
    t.string "status_after", null: false
    t.bigint "txid", default: -> { "txid_current()" }, null: false
    t.uuid "user_id", null: false
    t.index ["addendum_id"], name: "index_patient_medication_events_on_addendum_id"
    t.index ["clinical_document_id"], name: "index_patient_medication_events_on_clinical_document_id"
    t.index ["consultation_id"], name: "index_patient_medication_events_on_consultation_id"
    t.index ["patient_medication_id"], name: "index_patient_medication_events_on_patient_medication_id"
    t.index ["user_id"], name: "index_patient_medication_events_on_user_id"
    t.check_constraint "kind::text = ANY (ARRAY['added'::text, 'suspended'::text, 'reactivated'::text, 'changed'::text])", name: "ck_patient_medication_events_kind"
    t.check_constraint "num_nonnulls(consultation_id, addendum_id, clinical_document_id) = 1", name: "ck_patient_medication_events_source"
    t.check_constraint "status_after::text = ANY (ARRAY['active'::text, 'suspended'::text])", name: "ck_patient_medication_events_status"
  end

  create_table "patient_medications", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "catalog_item_id"
    t.integer "catmat_code"
    t.boolean "continuous", default: true, null: false
    t.datetime "created_at", null: false
    t.text "dosage_summary"
    t.text "free_text"
    t.text "label", null: false
    t.string "origin", null: false
    t.uuid "patient_id", null: false
    t.date "started_on"
    t.string "status", null: false
    t.datetime "updated_at", null: false
    t.index ["patient_id", "catalog_item_id"], name: "idx_patient_medications_one_active", unique: true, where: "(((status)::text = 'active'::text) AND (catalog_item_id IS NOT NULL))"
    t.index ["patient_id"], name: "index_patient_medications_on_patient_id"
    t.check_constraint "(catalog_item_id IS NULL) <> (free_text IS NULL) AND (catalog_item_id IS NULL) = (catmat_code IS NULL)", name: "ck_patient_medications_source"
    t.check_constraint "origin::text = ANY (ARRAY['prescription'::text, 'external'::text])", name: "ck_patient_medications_origin"
    t.check_constraint "status::text = ANY (ARRAY['active'::text, 'suspended'::text])", name: "ck_patient_medications_status"
  end

  create_table "prescription_items", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.boolean "antimicrobial", default: false, null: false
    t.uuid "catalog_release_id"
    t.integer "catmat_code"
    t.uuid "clinical_document_id", null: false
    t.boolean "continuous", default: false, null: false
    t.datetime "created_at", null: false
    t.text "dosage_instructions", null: false
    t.integer "duration_days"
    t.text "free_text"
    t.uuid "medication_catalog_item_id"
    t.uuid "nursing_protocol_version_item_id"
    t.integer "position", null: false
    t.text "printed_description", null: false
    t.decimal "quantity", precision: 10, scale: 2, null: false
    t.string "quantity_unit", limit: 30, null: false
    t.uuid "reason_problem_id"
    t.string "route", null: false
    t.index ["clinical_document_id", "position"], name: "idx_prescription_items_position", unique: true
    t.index ["medication_catalog_item_id"], name: "index_prescription_items_on_medication_catalog_item_id"
    t.index ["nursing_protocol_version_item_id"], name: "index_prescription_items_on_nursing_protocol_version_item_id"
    t.index ["reason_problem_id"], name: "index_prescription_items_on_reason_problem_id"
    t.check_constraint "(medication_catalog_item_id IS NULL) <> (free_text IS NULL)", name: "ck_prescription_items_source"
    t.check_constraint "(medication_catalog_item_id IS NULL) = (catmat_code IS NULL) AND (medication_catalog_item_id IS NULL) = (catalog_release_id IS NULL)", name: "ck_prescription_items_catalog"
    t.check_constraint "duration_days IS NULL OR duration_days >= 1 AND duration_days <= 365", name: "ck_prescription_items_duration"
    t.check_constraint "position >= 1", name: "ck_prescription_items_position"
    t.check_constraint "quantity > 0::numeric AND quantity <= 9999::numeric", name: "ck_prescription_items_quantity"
    t.check_constraint "route::text = ANY (ARRAY['oral'::text, 'sublingual'::text, 'topical'::text, 'ophthalmic'::text, 'otic'::text, 'nasal'::text, 'inhalation'::text, 'vaginal'::text, 'rectal'::text, 'intramuscular'::text, 'intravenous'::text, 'subcutaneous'::text, 'other'::text])", name: "ck_prescription_items_route"
  end
```

e, no bloco de `add_foreign_key` (ordem alfabética):

```ruby
  add_foreign_key "city_medication_list", "users", column: "added_by_user_id"
  add_foreign_key "clinical_documents", "attendances"
  add_foreign_key "clinical_documents", "citizens"
  add_foreign_key "clinical_documents", "clinical_documents", column: "replaces_document_id"
  add_foreign_key "clinical_documents", "consultations"
  add_foreign_key "clinical_documents", "patients"
  add_foreign_key "clinical_documents", "users", column: "author_user_id"
  add_foreign_key "clinical_documents", "users", column: "cancelled_by_user_id"
  add_foreign_key "document_verification_lookups", "clinical_documents"
  add_foreign_key "nursing_protocol_version_items", "nursing_protocol_versions"
  add_foreign_key "nursing_protocol_versions", "nursing_protocols"
  add_foreign_key "nursing_protocol_versions", "users", column: "created_by_user_id"
  add_foreign_key "nursing_protocols", "users", column: "created_by_user_id"
  add_foreign_key "patient_medication_events", "clinical_documents", deferrable: :deferred
  add_foreign_key "patient_medication_events", "consultation_addenda", column: "addendum_id", deferrable: :deferred
  add_foreign_key "patient_medication_events", "consultations", deferrable: :deferred
  add_foreign_key "patient_medication_events", "patient_medications", deferrable: :deferred
  add_foreign_key "patient_medication_events", "users"
  add_foreign_key "patient_medications", "patients"
  add_foreign_key "prescription_items", "clinical_documents"
  add_foreign_key "prescription_items", "nursing_protocol_version_items"
  add_foreign_key "prescription_items", "patient_problems", column: "reason_problem_id"
```

- [ ] **Step 6: Modelos, códigos e o 19b**

```ruby
# app/models/clinical_document.rb
# Documento clínico (ADR 0033; spec §4): cabeçalho comum dos quatro tipos.
# Imutável (trigger): só issued → cancelled pela autora, com motivo, e a volta
# ao papel do 19b (digital → paper, sem assinatura). Conteúdo do tipo em JSON
# cifrado com a chave da cidade (a forma de saída do contrato §2).
class ClinicalDocument < ApplicationRecord
  KINDS = %w[sick_note attendance_declaration prescription exam_requisition].freeze
  MODES = %w[digital paper].freeze
  STATUSES = %w[issued cancelled].freeze
  CANCEL_REASON = (10..500)

  encrypts :content
  encrypts :cancel_reason

  belongs_to :patient, optional: true
  belongs_to :citizen
  belongs_to :consultation, optional: true
  belongs_to :attendance
  belongs_to :author_user, class_name: "User"
  belongs_to :cancelled_by_user, class_name: "User", optional: true
  belongs_to :replaces_document, class_name: "ClinicalDocument", optional: true
  has_many :prescription_items, -> { order(:position) }, dependent: :restrict_with_exception, inverse_of: :clinical_document

  def content_data = JSON.parse(content)
  def issued? = status == "issued"
  def cancelled? = status == "cancelled"
  def digital? = issue_mode == "digital"
  def prescription? = kind == "prescription"
  def verification_url = "#{CityPublicUrl.base(Current.city)}/v/#{verification_token}"
  def signature_request = SignatureRequest.find_by(document_type: "ClinicalDocument", document_id: id)
end
```

```ruby
# app/models/prescription_item.rb
# Item da receita (ADR 0033; spec §5): nasce com o documento, nunca muda. Do
# catálogo (código e release da emissão) OU texto livre (marcado).
class PrescriptionItem < ApplicationRecord
  ROUTES = %w[oral sublingual topical ophthalmic otic nasal inhalation vaginal rectal intramuscular intravenous
              subcutaneous other].freeze

  encrypts :free_text
  encrypts :printed_description
  encrypts :dosage_instructions

  belongs_to :clinical_document, inverse_of: :prescription_items
  belongs_to :reason_problem, class_name: "PatientProblem", optional: true
  belongs_to :nursing_protocol_version_item, optional: true
end
```

```ruby
# app/models/patient_medication.rb
# Medicamento em uso do paciente (ADR 0033; spec §5): estado atual resultado de
# patient_medication_events; só Patients::ApplyMedicationEvent escreve aqui
# (trigger exige o evento na mesma transação). Um ativo por paciente + item.
class PatientMedication < ApplicationRecord
  STATUSES = %w[active suspended].freeze
  ORIGINS = %w[prescription external].freeze

  encrypts :free_text
  encrypts :label
  encrypts :dosage_summary

  belongs_to :patient
  has_many :events, class_name: "PatientMedicationEvent", dependent: :restrict_with_error

  scope :active_medications, -> { where(status: "active") }

  def active? = status == "active"
end
```

```ruby
# app/models/patient_medication_event.rb
# Um evento da lista de medicamentos (ADR 0033): só acréscimo, ligado a UMA
# consulta, adendo ou documento. O txid amarra o evento à transação do estado.
class PatientMedicationEvent < ApplicationRecord
  KINDS = %w[added suspended reactivated changed].freeze

  encrypts :dosage_summary

  # Opcional no modelo: o evento nasce ANTES do medicamento novo (FK DEFERRABLE).
  belongs_to :patient_medication, optional: true
  belongs_to :user
end
```

```ruby
# app/models/city_medication.rb
# Um item da REMUME da cidade (ADR 0033; spec §3): recorte do catálogo da
# plataforma, por id; unidades opcionais (vazio = toda a rede).
class CityMedication < ApplicationRecord
  self.table_name = "city_medication_list"

  belongs_to :added_by_user, class_name: "User"
end
```

```ruby
# app/models/nursing_protocol.rb
# Protocolo municipal de enfermagem (COFEN 801/2026; ADR 0033): título, número
# e ano nunca mudam; editar é criar versão. Vigente = a versão de maior número
# cuja vigência cobre a data.
class NursingProtocol < ApplicationRecord
  has_many :versions, class_name: "NursingProtocolVersion", dependent: :restrict_with_exception

  def current_version(on: Time.zone.today)
    versions.where(valid_from: ..on).where("valid_until IS NULL OR valid_until >= ?", on).order(version: :desc).first
  end
end
```

```ruby
# app/models/nursing_protocol_version.rb
class NursingProtocolVersion < ApplicationRecord
  belongs_to :nursing_protocol
  has_many :items, class_name: "NursingProtocolVersionItem", dependent: :restrict_with_exception
end
```

```ruby
# app/models/nursing_protocol_version_item.rb
class NursingProtocolVersionItem < ApplicationRecord
  belongs_to :nursing_protocol_version
end
```

```ruby
# app/models/document_verification_lookup.rb
# Trilha da página pública (ADR 0033; spec §6): só o documento achado (ou
# nenhum), a via e o resultado — nunca IP, código digitado ou ano.
class DocumentVerificationLookup < ApplicationRecord
  belongs_to :clinical_document, optional: true
end
```

```ruby
# app/services/clinical_documents/codes.rb
# Códigos de conferência (ADR 0033; Valores fixados 2): token de 128 bits para
# a URL do QR code e código curto de 10 caracteres sem ambíguos (0, 1, I, L,
# O) para digitar; a entrada aceita minúsculas, espaços e hífen.
module ClinicalDocuments
  module Codes
    ALPHABET = "23456789ABCDEFGHJKMNPQRSTUVWXYZ".chars.freeze
    LENGTH = 10

    module_function

    def token = SecureRandom.urlsafe_base64(16)
    def short_code = Array.new(LENGTH) { ALPHABET[SecureRandom.random_number(ALPHABET.size)] }.join
    def display(code) = "#{code[0, 5]}-#{code[5, 5]}"

    def normalize(input)
      code = input.to_s.upcase.gsub(/[\s-]/, "")
      code.size == LENGTH && code.chars.all? { |char| ALPHABET.include?(char) } ? code : nil
    end
  end
end
```

Em `app/models/signature_request.rb`, acrescente `document_cancelled` ao fim de `REASONS`.

Em `app/services/city_encryption.rb` (`CITY_KEYED_TARGETS`), depois do bloco do ADR 0032:

```ruby
    # ADR 0033: documentos clínicos e lista de medicamentos.
    [ ClinicalDocument, :content ],
    [ ClinicalDocument, :cancel_reason ],
    [ PrescriptionItem, :free_text ],
    [ PrescriptionItem, :printed_description ],
    [ PrescriptionItem, :dosage_instructions ],
    [ PatientMedication, :free_text ],
    [ PatientMedication, :label ],
    [ PatientMedication, :dosage_summary ],
    [ PatientMedicationEvent, :dosage_summary ]
```

Em `config/initializers/domain_events.rb`, depois dos eventos do 19b:

```ruby
  # Módulo 19c (ADR 0033): documentos clínicos e lista de medicamentos, só trilha.
  DomainEvents.bind "clinical_document.issued", to: []
  DomainEvents.bind "clinical_document.cancelled", to: []
  DomainEvents.bind "patient_medication.changed", to: []
```

e em `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 19c (ADR 0033): documentos clínicos, só trilha.
RSpec.describe "clinical document event bindings (ADR 0033)" do
  it "declares every module 19c event with no consumer" do
    names = %w[clinical_document.issued clinical_document.cancelled patient_medication.changed]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Em `config/initializers/filter_parameter_logging.rb`, ao fim da lista:

```ruby
  # ADR 0033: conteúdo dos documentos clínicos, medicamentos e a conferência
  # pública (`:reason`, `:note`, `:cpf` e `/\Aq\z/` já estavam na lista).
  :content, :items, :free_text, :dosage_instructions, :dosage_summary, :printed_description, :cid10,
  :companion_name, :companion_kinship, :short_code, :birth_year
```

- [ ] **Step 7: Os helpers de spec**

```ruby
# spec/support/clinical_document_helpers.rb
# Módulo 19c (ADR 0033): a cidade com documentos ligados, os papéis, o perfil
# com CNPJ e um documento cru (sem o comando, para as specs de banco).
module ClinicalDocumentHelpers
  def documents_city!(enabled: true)
    city = clinical_city!
    Platform::Features.set!(city: city, key: "clinical_documents", enabled: enabled, maintainer: ledi_maintainer!)
    CityCatalog.reset_cache!
    city
  end

  # O helper de vínculo cria o perfil com CRM; enfermeira e dentista levam o conselho deles.
  def nurse!(unit) = screener!(unit, cbo: "223505").tap { |user| user.professional.update!(council: "COREN") }
  def dentist!(unit) = screener!(unit, cbo: "223208").tap { |user| user.professional.update!(council: "CRO") }

  def city_profile! = CityProfile.current || CityProfile.create!(name: "Cidade de Teste", uf: "PR", ibge_code: "4106902")
  def city_cnpj!(cnpj = "76001234000115") = city_profile!.tap { |profile| profile.update!(cnpj: cnpj) }

  def municipal_admin! = staff_with("admin-doc-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin")

  def raw_document!(consultation: nil, attendance: nil, kind: "sick_note", issue_mode: "paper", author: nil,
                    content: { "type" => "leave", "days" => 1, "start_on" => Time.zone.today.iso8601, "cid_authorized" => false })
    attendance ||= consultation.attendance
    ClinicalDocument.create!(kind: kind, patient_id: consultation&.patient_id || attendance.citizen.patient_id,
                             citizen_id: attendance.citizen_id, consultation: consultation, attendance: attendance,
                             author_user_id: (author || consultation.author_user).id, cbo_code: consultation&.cbo_code,
                             issue_mode: issue_mode, verification_token: ClinicalDocuments::Codes.token,
                             short_code: ClinicalDocuments::Codes.short_code, issued_at: Time.current, content: content.to_json)
  end

  # MFA cadastrado (o step-up exige o autenticador).
  def step_up_enrolled!(user)
    Mfa::Enroll.call(user)
    user.update!(otp_enabled: true)
    user
  end

  # Só em request specs: sessão com step-up recente; PUT/DELETE com corpo JSON
  # (a guarda de escrita por cookie exige Content-Type JSON, inclusive no DELETE).
  def stepped_up!(user) = sign_in_as(step_up_enrolled!(user)).tap { |session| session.update!(mfa_verified_at: Time.current) }
  def json_put(path, params = {}) = put(path, params: params.to_json, headers: { "CONTENT_TYPE" => "application/json" })
  def json_delete(path) = delete(path, params: "{}", headers: { "CONTENT_TYPE" => "application/json" })
  def json_body = JSON.parse(response.body)
end

RSpec.configure { |c| c.include ClinicalDocumentHelpers }
```

Em `spec/rails_helper.rb`: `require_relative "support/clinical_document_helpers"` (junto dos outros `support/`). Confira o CBO de dentista: `ScreeningHelpers#screener!` cria o vínculo com o CBO dado; `223208` (cirurgião-dentista clínico geral) passa no `ck_professional_links_cbo_code`.

- [ ] **Step 8: Bancos de teste e a spec**

Rode o `DROP DATABASE` + `city:test_databases` do Ambiente de execução. Depois:

Run: `rspec spec/models/clinical_document_tables_guard_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/models/signature_tables_guard_spec.rb`
Expected: PASS. Se a paridade falhar, a diferença está no dump: compare `\d+ <tabela>` dos dois bancos de rascunho e corrija o `db/city_schema.rb` (nunca a migração).

- [ ] **Step 9: Commit**

```bash
git add db/city_migrate/20261009600001_add_clinical_documents.rb db/city_schema.rb db/city_triggers.sql app/models/clinical_document.rb app/models/prescription_item.rb app/models/patient_medication.rb app/models/patient_medication_event.rb app/models/city_medication.rb app/models/nursing_protocol.rb app/models/nursing_protocol_version.rb app/models/nursing_protocol_version_item.rb app/models/document_verification_lookup.rb app/services/clinical_documents/codes.rb app/models/signature_request.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/clinical_document_helpers.rb spec/rails_helper.rb spec/models/clinical_document_tables_guard_spec.rb
git commit -m "feat: add the clinical document, medication list and nursing protocol city tables

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: CNPJ da cidade (com DV, numérico e alfanumérico)

**Files:**
- Create: `app/services/cnpj.rb`, `app/controllers/clinical_documents/base_controller.rb`, `app/controllers/clinical_documents/city_profiles_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/cnpj_spec.rb`, `spec/requests/clinical_documents/city_profile_spec.rb`

**Interfaces:**
- Consumes: `CityProfile`, `AttendanceAccess#forbid`, `MfaStepUp#require_step_up!`, `ClinicalRecordGate`.
- Produces:
  - `Cnpj.normalize(value) -> String|nil` (14 caracteres, DV válido; aceita pontuação), `Cnpj.with_check_digits(base12) -> String`, `Cnpj.display(cnpj) -> "AA.AAA.AAA/AAAA-DD"`.
  - `ClinicalDocuments::BaseController` (prontuário utilizável, autenticação, `policy`, `require_admin_role`, `require_reader_role`, `body`, `unprocessable(error, **extra)`, `not_found`, `uuid?(value)`).
  - `GET /clinical_documents/city_profile` → `{ name, cnpj }` (admin ou profissional); `PUT /clinical_documents/city_profile { cnpj }` (admin, step-up) → `{ name, cnpj }`; 422 `invalid_cnpj`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/cnpj_spec.rb
require "rails_helper"

# COFEN 801/2026 (receita de enfermagem leva o CNPJ da instituição). DV pelo
# módulo 11; formato alfanumérico da IN RFB 2.229/2024 (valor de cada
# caractere = código ASCII − 48), com o exemplo oficial da Receita.
RSpec.describe Cnpj do
  it "aceita CNPJ numérico e alfanumérico válidos, com ou sem pontuação" do
    expect(described_class.normalize("11.222.333/0001-81")).to eq("11222333000181")
    expect(described_class.normalize("76001234000115")).to eq("76001234000115")
    expect(described_class.normalize("12.ABC.345/01DE-35")).to eq("12ABC34501DE35")
    expect(described_class.normalize("12abc34501de35")).to eq("12ABC34501DE35")
  end

  it "recusa DV errado, tamanho errado, repetidos e lixo" do
    [ "11222333000182", "1122233300018", "00000000000000", "AAAAAAAAAAAA00", "", nil, 123, "11.222.333/0001-8X" ].each do |value|
      expect(described_class.normalize(value)).to be_nil, value.inspect
    end
  end

  it "calcula os dígitos e formata" do
    expect(described_class.with_check_digits("760012340001")).to eq("76001234000115")
    expect(described_class.display("76001234000115")).to eq("76.001.234/0001-15")
  end
end
```

```ruby
# spec/requests/clinical_documents/city_profile_spec.rb
require "rails_helper"

# Contrato §6 (Desvio 12): o CNPJ da cidade (secretaria ou fundo), exigido na
# receita de enfermagem. Escrita do municipal_admin com step-up; leitura
# também do profissional (a tela da receita avisa quando falta).
RSpec.describe "CNPJ da cidade", type: :request do
  before { clinical_city!; city_profile! }

  it "admin grava (numérico ou alfanumérico); DV errado 422; sem step-up 401; sem JSON 415" do
    admin = municipal_admin!
    sign_in_as(step_up_enrolled!(admin))
    json_put "/clinical_documents/city_profile", cnpj: "76.001.234/0001-15"
    expect([ response.status, json_body ]).to eq([ 401, { "error" => "mfa_required" } ])

    stepped_up!(admin)
    json_put "/clinical_documents/city_profile", cnpj: "76.001.234/0001-15"
    expect(response).to have_http_status(:ok)
    expect(json_body).to eq("name" => CityProfile.current.name, "cnpj" => "76001234000115")
    json_put "/clinical_documents/city_profile", cnpj: "12ABC34501DE35"
    expect(CityProfile.current.cnpj).to eq("12ABC34501DE35")
    json_put "/clinical_documents/city_profile", cnpj: "76001234000116"
    expect([ response.status, json_body ]).to eq([ 422, { "error" => "invalid_cnpj" } ])
    put "/clinical_documents/city_profile", params: { cnpj: "76001234000115" }
    expect(response).to have_http_status(:unsupported_media_type)
  end

  it "profissional lê; recepção não; profissional não escreve" do
    unit = create_unit
    sign_in_as(doctor!(unit))
    get "/clinical_documents/city_profile"
    expect(json_body).to include("cnpj" => nil)
    json_put "/clinical_documents/city_profile", cnpj: "76001234000115"
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "missing_role" } ])
    sign_in_as(verifier!)
    get "/clinical_documents/city_profile"
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "missing_role" } ])
  end

  it "prontuário desligado → 403 feature_disabled" do
    clinical_city!(enabled: false)
    sign_in_as(municipal_admin!)
    get "/clinical_documents/city_profile"
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "clinical_record")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/cnpj_spec.rb spec/requests/clinical_documents/city_profile_spec.rb`
Expected: FAIL (`uninitialized constant Cnpj` / rota inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/services/cnpj.rb
# CNPJ com dígitos verificadores (módulo 11, pesos 2..9 da direita para a
# esquerda). Desde julho de 2026 a Receita emite CNPJ alfanumérico (IN RFB
# 2.229/2024): os 12 primeiros caracteres em [0-9A-Z], os 2 últimos dígitos;
# o valor de cada caractere é o código ASCII − 48 (o numérico é o caso
# particular). Exemplo oficial: 12.ABC.345/01DE-35.
module Cnpj
  WEIGHTS = (2..9).to_a.freeze
  FORMAT = /\A[0-9A-Z]{12}[0-9]{2}\z/

  module_function

  def normalize(value)
    return nil unless value.is_a?(String)

    raw = value.upcase.gsub(%r{[.\-/\s]}, "")
    return nil unless raw.match?(FORMAT) && raw.chars.uniq.size > 1

    raw == with_check_digits(raw[0, 12]) ? raw : nil
  end

  def with_check_digits(base12)
    first = digit(base12)
    "#{base12}#{first}#{digit("#{base12}#{first}")}"
  end

  def display(cnpj) = "#{cnpj[0, 2]}.#{cnpj[2, 3]}.#{cnpj[5, 3]}/#{cnpj[8, 4]}-#{cnpj[12, 2]}"

  def digit(chars)
    sum = chars.chars.reverse.each_with_index.sum { |char, index| (char.ord - 48) * WEIGHTS[index % WEIGHTS.size] }
    rest = sum % 11
    rest < 2 ? 0 : 11 - rest
  end
  private_class_method :digit
end
```

```ruby
# app/controllers/clinical_documents/base_controller.rb
# Configuração dos documentos clínicos da cidade (ADR 0033; contrato §6):
# CNPJ, REMUME e protocolos de enfermagem. Atrás do prontuário (a cidade
# prepara antes de ligar clinical_documents); escrita só do municipal_admin,
# com step-up; leitura também do profissional.
module ClinicalDocuments
  class BaseController < ApplicationController
    include Authentication
    include AttendanceAccess
    include ClinicalRecordGate
    include MfaStepUp

    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

    wrap_parameters false
    before_action :require_clinical_record!

    private

    def policy = (@policy ||= CitizenVerificationPolicy.new(Current.user, nil))
    def require_admin_role = (forbid("missing_role") unless policy.manage?)
    def require_reader_role = (forbid("missing_role") unless policy.manage? || policy.care?)
    def body = params.to_unsafe_h.except("controller", "action", "id", "catalog_item_id")
    def unprocessable(error, **extra) = render(json: { error: error, **extra }, status: :unprocessable_entity)
    def not_found = render(json: { error: "not_found" }, status: :not_found)
    def uuid?(value) = value.is_a?(String) && value.match?(UUID)
  end
end
```

```ruby
# app/controllers/clinical_documents/city_profiles_controller.rb
# CNPJ da cidade (Desvio 12; contrato §6): obrigatório na receita de
# enfermagem (city_cnpj_missing). O resto do perfil não muda por aqui.
module ClinicalDocuments
  class CityProfilesController < BaseController
    before_action :require_reader_role, only: :show
    before_action :require_admin_role, only: :update
    before_action :require_step_up!, only: :update
    before_action :set_profile

    def show = render(json: json)

    def update
      cnpj = Cnpj.normalize(body["cnpj"])
      return unprocessable("invalid_cnpj") unless cnpj

      @profile.update!(cnpj: cnpj)
      render json: json
    end

    private

    def set_profile
      @profile = CityProfile.current
      not_found unless @profile
    end

    def json = { name: @profile.name, cnpj: @profile.cnpj }
  end
end
```

Em `config/routes.rb`, logo depois do bloco `scope "/clinical_record"`:

```ruby
  # Documentos clínicos (ADR 0033; contrato §6): configuração da cidade (CNPJ,
  # REMUME, protocolos de enfermagem). Prefixo próprio no proxy do dashboard.
  scope "/clinical_documents", module: :clinical_documents, as: :clinical_documents do
    get "city_profile", to: "city_profiles#show"
    put "city_profile", to: "city_profiles#update"
  end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/cnpj_spec.rb spec/requests/clinical_documents/city_profile_spec.rb spec/requests/cookie_write_requires_json_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/cnpj.rb app/controllers/clinical_documents/base_controller.rb app/controllers/clinical_documents/city_profiles_controller.rb config/routes.rb spec/services/cnpj_spec.rb spec/requests/clinical_documents/city_profile_spec.rb
git commit -m "feat: store the city CNPJ with check digits for nursing prescriptions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: REMUME e busca no catálogo

**Files:**
- Create: `app/services/clinical_documents/json.rb`, `app/services/medications/search.rb`, `app/controllers/clinical_documents/remume_controller.rb`, `app/controllers/medication_searches_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/clinical_documents/remume_spec.rb`, `spec/requests/medication_search_spec.rb`

**Interfaces:**
- Consumes: `MedicationCatalogItem` (`.prescribable`, `#label`), `MedicationCatalogRelease.current`, `AnvisaListRelease.current`, `CityMedication`, `import_test_catalog!`, `catalog_item(code)`.
- Produces:
  - `ClinicalDocuments::Json.catalog_item(item, in_network: nil) -> Hash` (`{ id, catmat_code, label, active_ingredient, strength, dosage_form, antimicrobial, controlled }` + `in_network` quando dado); `ClinicalDocuments::Json.catalog_ref(item) -> Hash` (`{ id, catmat_code, label, active_ingredient, strength, dosage_form }`, a forma de `<item>.catalog_item`); `ClinicalDocuments::Json.number(decimal) -> Integer|Float`.
  - `Medications::Search.available? -> bool` (release do catálogo **e** listas Anvisa ativas); `Medications::Search.call(q:, unit_id: nil) -> Array<Hash>` (levanta `Medications::Search::Unavailable`).
  - `GET/POST /clinical_documents/remume`, `DELETE /clinical_documents/remume/:catalog_item_id` (204); 422 `invalid_catalog_item`, `invalid_unit`.
  - `POST /attendance/medications/search { q, unit_id? }` (profissional **ou** `municipal_admin`) → `{ items }`; 503 `catalog_unavailable`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/clinical_documents/remume_spec.rb
require "rails_helper"

# Contrato §6: a REMUME é o recorte da cidade sobre o catálogo da plataforma,
# por id; unidades opcionais. municipal_admin com step-up.
RSpec.describe "REMUME", type: :request do
  before { clinical_city!; import_test_catalog! }

  let(:admin) { municipal_admin! }
  let(:unit) { create_unit }

  it "inclui, atualiza as unidades, lista e remove" do
    stepped_up!(admin)
    losartana = catalog_item(268_856)
    json_post "/clinical_documents/remume", catalog_item_id: losartana.id
    expect(response).to have_http_status(:created)
    expect(json_body).to eq("catalog_item" => { "id" => losartana.id, "catmat_code" => 268_856, "label" => "LOSARTANA POTÁSSICA 50 MG",
                                                "active_ingredient" => "LOSARTANA POTÁSSICA", "strength" => "50 MG",
                                                "dosage_form" => nil, "antimicrobial" => false, "controlled" => false },
                            "unit_ids" => [])
    json_post "/clinical_documents/remume", catalog_item_id: losartana.id, unit_ids: [ unit.id ]
    expect([ response.status, json_body["unit_ids"] ]).to eq([ 200, [ unit.id ] ])
    get "/clinical_documents/remume"
    expect(json_body["items"].map { |row| row.dig("catalog_item", "catmat_code") }).to eq([ 268_856 ])
    json_delete "/clinical_documents/remume/#{losartana.id}"
    expect(response).to have_http_status(:no_content)
    expect(CityMedication.count).to eq(0)
    json_delete "/clinical_documents/remume/#{losartana.id}"
    expect(response).to have_http_status(:not_found)
  end

  it "recusa item escondido, inexistente ou removido e unidade desconhecida" do
    stepped_up!(admin)
    json_post "/clinical_documents/remume", catalog_item_id: catalog_item(384_258).id
    expect(json_body).to eq("error" => "invalid_catalog_item")
    json_post "/clinical_documents/remume", catalog_item_id: "nao-e-uuid"
    expect(json_body).to eq("error" => "invalid_catalog_item")
    json_post "/clinical_documents/remume", catalog_item_id: catalog_item(268_856).id, unit_ids: [ SecureRandom.uuid ]
    expect(json_body).to eq("error" => "invalid_unit")
  end

  it "profissional lê; escrita exige admin e step-up" do
    sign_in_as(doctor!(unit))
    get "/clinical_documents/remume"
    expect(response).to have_http_status(:ok)
    json_post "/clinical_documents/remume", catalog_item_id: catalog_item(268_856).id
    expect(json_body).to eq("error" => "missing_role")
    sign_in_as(step_up_enrolled!(admin))
    json_post "/clinical_documents/remume", catalog_item_id: catalog_item(268_856).id
    expect(json_body).to eq("error" => "mfa_required")
  end
end
```

```ruby
# spec/requests/medication_search_spec.rb
require "rails_helper"

# Contrato §5: busca no catálogo da plataforma; REMUME primeiro ("na rede",
# pela unidade); controlado aparece marcado (bloqueado na emissão); escondido
# e removido nunca; sem catálogo ou sem listas Anvisa → 503.
RSpec.describe "Busca de medicamentos", type: :request do
  before { documents_city! }

  let(:unit) { create_unit }
  let(:other_unit) { create_unit("UBS Norte") }
  let(:doctor) { doctor!(unit) }

  def search(q, unit_id: nil) = json_post("/attendance/medications/search", { q: q, unit_id: unit_id }.compact)

  it "sem catálogo ativo → 503 catalog_unavailable" do
    sign_in_as(doctor)
    search("losartana")
    expect([ response.status, json_body ]).to eq([ 503, { "error" => "catalog_unavailable" } ])
  end

  it "REMUME primeiro (por unidade), acento e caixa ignorados, controlado marcado, escondido fora" do
    import_test_catalog!
    CityMedication.create!(catalog_item_id: catalog_item(267_674).id, unit_ids: [ other_unit.id ], added_by_user: doctor)
    CityMedication.create!(catalog_item_id: catalog_item(267_772).id, unit_ids: [], added_by_user: doctor)
    sign_in_as(doctor)

    search("losartana")
    expect(json_body["items"].map { |item| item["catmat_code"] }).to eq([ 268_856 ]) # 384258 (manipulada) está escondida
    search("diazepam")
    expect(json_body["items"].first).to include("catmat_code" => 267_197, "controlled" => true, "in_network" => false)
    search("ácido fólico")
    expect(json_body["items"].first["catmat_code"]).to eq(267_503)
    search("o")
    expect(json_body["items"]).to eq([])

    search("mg", unit_id: unit.id)
    first = json_body["items"].first
    expect(first).to include("catmat_code" => 267_772, "in_network" => true)        # REMUME de toda a rede
    expect(json_body["items"].find { |item| item["catmat_code"] == 267_674 }["in_network"]).to be(false) # só na outra UBS
  end

  it "interruptor desligado → 403; recepção → 403 missing_role" do
    import_test_catalog!
    documents_city!(enabled: false)
    sign_in_as(doctor)
    search("losartana")
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "clinical_documents")
    documents_city!
    sign_in_as(verifier!)
    search("losartana")
    expect(json_body).to eq("error" => "missing_role")
    sign_in_as(municipal_admin!) # monta a REMUME e os protocolos
    search("losartana")
    expect(json_body["items"].map { |item| item["catmat_code"] }).to eq([ 268_856 ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/clinical_documents/remume_spec.rb spec/requests/medication_search_spec.rb`
Expected: FAIL (rotas inexistentes).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/json.rb
# Formas do contrato do 19c (§2, §5, §6). Este arquivo cresce nas Tasks 11,
# 15 e 16 (protocolo, documento, medicamento).
module ClinicalDocuments
  module Json
    module_function

    # <item> da busca (contrato §5) e da REMUME (§6).
    def catalog_item(item, in_network: nil)
      json = catalog_ref(item).merge(antimicrobial: item.antimicrobial, controlled: item.controlled)
      in_network.nil? ? json : json.merge(in_network: in_network)
    end

    # `catalog_item` dentro do <item> da receita e do <medication> (§2).
    def catalog_ref(item)
      { id: item.id, catmat_code: item.catmat_code, label: item.label, active_ingredient: item.active_ingredient,
        strength: item.strength, dosage_form: item.dosage_form }
    end

    # Número do JSON: inteiro quando não tem casas (30, não 30.0).
    def number(decimal)
      return nil if decimal.nil?

      value = BigDecimal(decimal.to_s)
      value.frac.zero? ? value.to_i : value.to_f
    end
  end
end
```

```ruby
# app/services/medications/search.rb
# Busca no catálogo da plataforma (ADR 0033; contrato §5): todos os termos
# (sem acento, maiúsculos) no texto de busca; só itens ativos e não
# escondidos; os da REMUME da cidade primeiro ("na rede": REMUME sem
# unidades = toda a rede; com unidades, só nelas). Catálogo ou listas Anvisa
# ausentes → Unavailable (sem as listas, controlado não seria bloqueado).
module Medications
  module Search
    LIMIT = 30
    MIN_QUERY = 2
    class Unavailable < StandardError; end

    module_function

    def available? = MedicationCatalogRelease.current.present? && AnvisaListRelease.current.present?

    def call(q:, unit_id: nil)
      raise Unavailable unless available?

      tokens = Substances.key(q).split.first(6)
      return [] if tokens.join.size < MIN_QUERY

      scope = tokens.reduce(MedicationCatalogItem.prescribable) do |relation, token|
        relation.where("search_text LIKE ?", "%#{MedicationCatalogItem.sanitize_sql_like(token)}%")
      end
      network = network_ids(unit_id)
      ordered = ->(relation) { relation.order(:active_ingredient, :strength, :catmat_code) }
      inside = network.empty? ? [] : ordered.(scope.where(id: network.to_a)).limit(LIMIT).to_a
      outside = ordered.(scope.where.not(id: inside.map(&:id))).limit(LIMIT - inside.size).to_a
      inside.map { |item| ClinicalDocuments::Json.catalog_item(item, in_network: true) } +
        outside.map { |item| ClinicalDocuments::Json.catalog_item(item, in_network: false) }
    end

    def network_ids(unit_id)
      CityMedication.pluck(:catalog_item_id, :unit_ids)
                    .select { |_id, units| units.empty? || unit_id.nil? || units.include?(unit_id) }
                    .to_set(&:first)
    end
    private_class_method :network_ids
  end
end
```

```ruby
# app/controllers/clinical_documents/remume_controller.rb
# REMUME (ADR 0033; contrato §6): o recorte da cidade sobre o catálogo da
# plataforma. Item escondido (revisão), removido ou inexistente → 422
# invalid_catalog_item; controlado pode entrar (a cidade dispensa; a receita
# do 19c o bloqueia).
module ClinicalDocuments
  class RemumeController < BaseController
    before_action :require_reader_role, only: :index
    before_action :require_admin_role, except: :index
    before_action :require_step_up!, except: :index

    def index
      rows = CityMedication.order(:created_at, :id).to_a
      items = MedicationCatalogItem.where(id: rows.map(&:catalog_item_id)).index_by(&:id)
      render json: { items: rows.filter_map { |row| (item = items[row.catalog_item_id]) && row_json(item, row) } }
    end

    def create
      item = uuid?(body["catalog_item_id"]) && MedicationCatalogItem.prescribable.find_by(id: body["catalog_item_id"])
      return unprocessable("invalid_catalog_item") unless item

      unit_ids = body.fetch("unit_ids", [])
      return unprocessable("invalid_unit") unless valid_units?(unit_ids)

      row = CityMedication.find_or_initialize_by(catalog_item_id: item.id)
      created = row.new_record?
      row.update!(unit_ids: unit_ids.uniq, added_by_user: Current.user)
      render json: row_json(item, row), status: created ? :created : :ok
    end

    def destroy
      row = uuid?(params[:catalog_item_id]) && CityMedication.find_by(catalog_item_id: params[:catalog_item_id])
      return not_found unless row

      row.destroy!
      head :no_content
    end

    private

    def valid_units?(ids)
      ids.is_a?(Array) && ids.all? { |id| uuid?(id) } && HealthUnit.where(id: ids).count == ids.uniq.size
    end

    def row_json(item, row) = { catalog_item: Json.catalog_item(item), unit_ids: row.unit_ids }
  end
end
```

```ruby
# app/controllers/medication_searches_controller.rb
# Busca de medicamentos para a receita, o "medicamento em uso de fora", a
# REMUME e os protocolos (ADR 0033; contrato §5). Profissional ou
# municipal_admin; interruptor clinical_documents.
class MedicationSearchesController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalDocumentsGate

  wrap_parameters false
  before_action :require_clinical_documents!
  before_action :require_searcher

  def search
    unit_id = params[:unit_id].is_a?(String) ? params[:unit_id] : nil
    render json: { items: Medications::Search.call(q: params[:q].to_s, unit_id: unit_id) }
  rescue Medications::Search::Unavailable
    render json: { error: "catalog_unavailable" }, status: :service_unavailable
  end

  private

  # Profissional (receita, medicamento de fora) ou municipal_admin (REMUME e
  # protocolos): o catálogo não tem dado de paciente.
  def require_searcher
    policy = CitizenVerificationPolicy.new(Current.user, nil)
    forbid("missing_role") unless policy.care? || policy.manage?
  end
end
```

Em `config/routes.rb`, no `scope "/clinical_documents"`:

```ruby
    get    "remume",                  to: "remume#index"
    post   "remume",                  to: "remume#create"
    delete "remume/:catalog_item_id", to: "remume#destroy"
```

e no `scope "/attendance"`, depois de `sigtap/search`:

```ruby
    # Busca no catálogo de medicamentos (ADR 0033; contrato §5).
    post "medications/search", to: "medication_searches#search"
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/clinical_documents/remume_spec.rb spec/requests/medication_search_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/json.rb app/services/medications/search.rb app/controllers/clinical_documents/remume_controller.rb app/controllers/medication_searches_controller.rb config/routes.rb spec/requests/clinical_documents/remume_spec.rb spec/requests/medication_search_spec.rb
git commit -m "feat: add the city REMUME and the medication catalog search

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Protocolos de enfermagem com versões

**Files:**
- Create: `app/commands/nursing_protocols/save.rb`, `app/controllers/clinical_documents/nursing_protocols_controller.rb`
- Modify: `app/services/clinical_documents/json.rb`, `config/routes.rb`
- Test: `spec/requests/clinical_documents/nursing_protocols_spec.rb`

**Interfaces:**
- Consumes: `NursingProtocol{,Version,VersionItem}` (Task 8), `MedicationCatalogItem.prescribable`.
- Produces:
  - `NursingProtocols::Save.create(params:, by:, catalog:, today: Time.zone.today) -> Result(protocol:)`, `NursingProtocols::Save.add_version(protocol:, params:, by:, catalog:) -> Result(protocol:)`; `NursingProtocols::Save.catalog_ids(params) -> Array<String>` (os uuids de `items[*].catalog_item_id`, para ler o catálogo FORA da transação). Falhas: `invalid_content` (`field`), `invalid_item` (`index`, `field`), `controlled_not_allowed` (`index`), `already_exists`.
  - `ClinicalDocuments::Json.protocol(protocol, catalog:, on: Time.zone.today) -> Hash` (contrato §6 + `current_version.version`).
  - `GET /clinical_documents/nursing_protocols` (admin ou profissional), `POST /clinical_documents/nursing_protocols`, `POST /clinical_documents/nursing_protocols/:id/versions` (admin, step-up).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/clinical_documents/nursing_protocols_spec.rb
require "rails_helper"

# COFEN 801/2026; contrato §6: protocolo municipal (título, número, ano) →
# versões (vigência, itens com dose máxima opcional). Editar = versão nova;
# a vigente é a de maior número que cobre hoje.
RSpec.describe "Protocolos de enfermagem", type: :request do
  before { clinical_city!; import_test_catalog! }

  let(:admin) { municipal_admin! }

  def protocol_body(**over)
    { title: "Saúde da mulher na APS", number: "PE-01", year: 2026, valid_from: "2026-01-01",
      items: [ { catalog_item_id: catalog_item(267_503).id, max_dose: { quantity: 30, unit: "comprimido" } },
               { catalog_item_id: catalog_item(268_162).id } ] }.merge(over)
  end

  it "cria com itens e dose máxima; número + ano repetido → 409; nova versão vira a vigente" do
    stepped_up!(admin)
    json_post "/clinical_documents/nursing_protocols", protocol_body
    expect(response).to have_http_status(:created)
    protocol = json_body
    expect(protocol).to include("title" => "Saúde da mulher na APS", "number" => "PE-01", "year" => 2026)
    expect(protocol["current_version"]).to include("version" => 1, "valid_from" => "2026-01-01", "valid_until" => nil)
    expect(protocol["current_version"]["items"].first)
      .to eq("catalog_item" => ClinicalDocuments::Json.catalog_ref(catalog_item(267_503)).as_json,
             "max_dose" => { "quantity" => 30, "unit" => "comprimido" })
    expect(protocol["current_version"]["items"].second["max_dose"]).to be_nil

    json_post "/clinical_documents/nursing_protocols", protocol_body
    expect([ response.status, json_body ]).to eq([ 409, { "error" => "already_exists" } ])

    json_post "/clinical_documents/nursing_protocols/#{protocol['id']}/versions",
              valid_from: "2026-02-01", items: [ { catalog_item_id: catalog_item(267_778).id } ]
    expect(json_body["current_version"]).to include("version" => 2)
    json_post "/clinical_documents/nursing_protocols/#{protocol['id']}/versions",
              valid_from: (Time.zone.today + 30).iso8601, items: [ { catalog_item_id: catalog_item(267_203).id } ]
    expect(json_body["current_version"]["version"]).to eq(2) # a 3 ainda não vale
    get "/clinical_documents/nursing_protocols"
    expect(json_body["items"].map { |row| row.dig("current_version", "version") }).to eq([ 2 ])
  end

  it "valida cabeçalho, vigência e itens; controlado e escondido recusados" do
    stepped_up!(admin)
    json_post "/clinical_documents/nursing_protocols", protocol_body(title: "x")
    expect(json_body).to eq("error" => "invalid_content", "field" => "title")
    json_post "/clinical_documents/nursing_protocols", protocol_body(valid_until: "2025-12-31")
    expect(json_body).to eq("error" => "invalid_content", "field" => "valid_until")
    json_post "/clinical_documents/nursing_protocols", protocol_body(items: [])
    expect(json_body).to eq("error" => "invalid_content", "field" => "items")
    json_post "/clinical_documents/nursing_protocols", protocol_body(items: [ { catalog_item_id: catalog_item(267_197).id } ])
    expect(json_body).to eq("error" => "controlled_not_allowed", "index" => 0)
    json_post "/clinical_documents/nursing_protocols", protocol_body(items: [ { catalog_item_id: catalog_item(384_258).id } ])
    expect(json_body).to eq("error" => "invalid_item", "index" => 0, "field" => "catalog_item_id")
    json_post "/clinical_documents/nursing_protocols",
              protocol_body(items: [ { catalog_item_id: catalog_item(267_503).id, max_dose: { quantity: 0, unit: "cp" } } ])
    expect(json_body).to eq("error" => "invalid_item", "index" => 0, "field" => "max_dose")
  end

  it "enfermeira lê; recepção não; escrita exige admin com step-up" do
    unit = create_unit
    sign_in_as(nurse!(unit))
    get "/clinical_documents/nursing_protocols"
    expect(response).to have_http_status(:ok)
    json_post "/clinical_documents/nursing_protocols", protocol_body
    expect(json_body).to eq("error" => "missing_role")
    sign_in_as(verifier!)
    get "/clinical_documents/nursing_protocols"
    expect(json_body).to eq("error" => "missing_role")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/clinical_documents/nursing_protocols_spec.rb`
Expected: FAIL (rota inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/nursing_protocols/save.rb
# Protocolos municipais de enfermagem (COFEN 801/2026; ADR 0033; contrato §6).
# Criar = cabeçalho + versão 1; editar = versão nova (número seguinte, com o
# protocolo travado). Itens: do catálogo da plataforma (lido FORA da
# transação pelo chamador), ativos, não escondidos e nunca controlados; dose
# máxima opcional = quantidade máxima por item de receita (Desvio 10).
module NursingProtocols
  module Save
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
    MAX_ITEMS = 100

    module_function

    def catalog_ids(params)
      Array(params["items"]).filter_map { |item| item.is_a?(Hash) && item["catalog_item_id"].to_s.match?(UUID) ? item["catalog_item_id"] : nil }
    end

    def create(params:, by:, catalog:, today: Time.zone.today)
      title = params["title"].is_a?(String) ? params["title"].squish : ""
      number = params["number"].is_a?(String) ? params["number"].squish : ""
      year = params["year"]
      return invalid("title") unless title.size.between?(3, 200)
      return invalid("number") unless number.size.between?(1, 30)
      return invalid("year") unless year.is_a?(Integer) && year.between?(1990, today.year + 1)

      result = nil
      ApplicationRecord.transaction(requires_new: true) do
        protocol = NursingProtocol.create!(title: title, number: number, year: year, created_by_user_id: by.id)
        result = version!(protocol, params, by, catalog, 1)
        raise ActiveRecord::Rollback if result.failure?
      end
      result
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:already_exists)
    end

    def add_version(protocol:, params:, by:, catalog:)
      result = nil
      ApplicationRecord.transaction(requires_new: true) do
        protocol.lock!
        result = version!(protocol, params, by, catalog, protocol.versions.maximum(:version).to_i + 1)
        raise ActiveRecord::Rollback if result.failure?
      end
      result
    end

    def version!(protocol, params, by, catalog, number)
      valid_from = date(params["valid_from"])
      return invalid("valid_from") unless valid_from

      valid_until = params["valid_until"].nil? ? nil : date(params["valid_until"])
      return invalid("valid_until") if !params["valid_until"].nil? && (valid_until.nil? || valid_until < valid_from)

      items = params["items"]
      return invalid("items") unless items.is_a?(Array) && items.size.between?(1, MAX_ITEMS)

      rows = []
      items.each_with_index do |item, index|
        row = item_row(item, index, catalog, rows)
        return row if row.is_a?(Result)

        rows << row
      end
      version = NursingProtocolVersion.create!(nursing_protocol: protocol, version: number, valid_from: valid_from,
                                               valid_until: valid_until, created_by_user_id: by.id)
      rows.each { |row| NursingProtocolVersionItem.create!(row.merge(nursing_protocol_version: version)) }
      Result.ok(protocol: protocol)
    end

    def item_row(item, index, catalog, rows)
      bad = ->(field) { Result.fail(:invalid_item, details: { index: index, field: field }) }
      return bad.("catalog_item_id") unless item.is_a?(Hash)

      entry = catalog[item["catalog_item_id"].to_s]
      return bad.("catalog_item_id") unless entry && entry.status == "active" && !entry.hidden
      return Result.fail(:controlled_not_allowed, details: { index: index }) if entry.controlled
      return bad.("catalog_item_id") if rows.any? { |row| row[:catalog_item_id] == entry.id }

      max = item["max_dose"]
      return { catalog_item_id: entry.id } if max.nil?

      quantity = max.is_a?(Hash) && max["quantity"].is_a?(Numeric) ? BigDecimal(max["quantity"].to_s) : nil
      unit = max.is_a?(Hash) && max["unit"].is_a?(String) ? max["unit"].squish : ""
      return bad.("max_dose") unless quantity && quantity.positive? && quantity <= 9999 && unit.size.between?(1, 30)

      { catalog_item_id: entry.id, max_quantity: quantity, max_quantity_unit: unit }
    end

    def date(value)
      value.is_a?(String) && value.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(value) : nil
    rescue Date::Error
      nil
    end

    def invalid(field) = Result.fail(:invalid_content, details: { field: field })
    private_class_method :version!, :item_row, :date, :invalid
  end
end
```

Em `app/services/clinical_documents/json.rb`, acrescente:

```ruby
    # Protocolo de enfermagem (contrato §6) com a versão vigente na data.
    def protocol(protocol, catalog:, on: Time.zone.today)
      version = protocol.current_version(on: on)
      { id: protocol.id, title: protocol.title, number: protocol.number, year: protocol.year,
        current_version: version && {
          id: version.id, version: version.version, valid_from: version.valid_from.iso8601,
          valid_until: version.valid_until&.iso8601,
          items: version.items.order(:created_at, :id).filter_map do |item|
            entry = catalog[item.catalog_item_id]
            entry && { catalog_item: catalog_ref(entry),
                       max_dose: item.max_quantity && { quantity: number(item.max_quantity), unit: item.max_quantity_unit } }
          end
        } }
    end
```

```ruby
# app/controllers/clinical_documents/nursing_protocols_controller.rb
# Protocolos de enfermagem (ADR 0033; contrato §6). Leitura: admin e
# profissional (a enfermeira escolhe o protocolo na receita). Escrita: admin
# com step-up. O catálogo é lido FORA da transação da cidade.
module ClinicalDocuments
  class NursingProtocolsController < BaseController
    ERRORS = { already_exists: :conflict }.freeze

    before_action :require_reader_role, only: :index
    before_action :require_admin_role, except: :index
    before_action :require_step_up!, except: :index

    def index
      protocols = NursingProtocol.includes(versions: :items).order(:year, :number).to_a
      ids = protocols.flat_map { |p| p.versions.flat_map { |v| v.items.map(&:catalog_item_id) } }
      catalog = MedicationCatalogItem.where(id: ids.uniq).index_by(&:id)
      render json: { items: protocols.map { |protocol| Json.protocol(protocol, catalog: catalog) } }
    end

    def create
      respond(NursingProtocols::Save.create(params: body, by: Current.user, catalog: catalog_for(body)), :created)
    end

    def add_version
      protocol = uuid?(params[:id]) && NursingProtocol.find_by(id: params[:id])
      return not_found unless protocol

      respond(NursingProtocols::Save.add_version(protocol: protocol, params: body, by: Current.user, catalog: catalog_for(body)), :created)
    end

    private

    def catalog_for(params) = MedicationCatalogItem.where(id: NursingProtocols::Save.catalog_ids(params)).index_by(&:id)

    def respond(result, status)
      return render_failure(result, ERRORS) if result.failure?

      protocol = result.payload[:protocol].reload
      ids = NursingProtocolVersionItem.joins(:nursing_protocol_version)
                                      .where(nursing_protocol_versions: { nursing_protocol_id: protocol.id }).pluck(:catalog_item_id)
      render json: Json.protocol(protocol, catalog: MedicationCatalogItem.where(id: ids.uniq).index_by(&:id)), status: status
    end
  end
end
```

Em `config/routes.rb`, no `scope "/clinical_documents"`:

```ruby
    get  "nursing_protocols",              to: "nursing_protocols#index"
    post "nursing_protocols",              to: "nursing_protocols#create"
    post "nursing_protocols/:id/versions", to: "nursing_protocols#add_version"
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/clinical_documents/nursing_protocols_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/commands/nursing_protocols/save.rb app/controllers/clinical_documents/nursing_protocols_controller.rb app/services/clinical_documents/json.rb config/routes.rb spec/requests/clinical_documents/nursing_protocols_spec.rb
git commit -m "feat: add versioned municipal nursing protocols with item limits

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — Lista de medicamentos em uso (F-19.18)

### Task 12: `Patients::ApplyMedicationEvent`, o único caminho de escrita

**Files:**
- Create: `app/commands/patients/apply_medication_event.rb`
- Test: `spec/commands/patients/apply_medication_event_spec.rb`

**Interfaces:**
- Consumes: `PatientMedication`, `PatientMedicationEvent` (Task 8), `MedicationCatalogItem` (`#label`, `#id`, `#catmat_code`).
- Produces: `Patients::ApplyMedicationEvent.call(patient:, action:, by:, source:, origin: "prescription", catalog_item: nil, free_text: nil, dosage_summary: nil, continuous: true, medication: nil, on_active: :change, on: Time.zone.today) -> Result` — `ok(medication:, event:)` (`event` nil quando nada mudou) | `fail(:already_active | :invalid_medication)`. `action` `add` | `suspend` | `reactivate`; `source` com exatamente uma chave entre `consultation:`, `addendum:`, `document:` (senão `ArgumentError`); `on_active: :change` (receita: ativo igual → `changed` ou nada) ou `:fail` (informado de fora: ativo igual → `already_active`). Quem chama abriu a transação e **travou o paciente** (`Patient.lock`). Publica `patient_medication.changed { patient_medication_id, kind, document_id | consultation_id | addendum_id }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/patients/apply_medication_event_spec.rb
require "rails_helper"

# ADR 0033 (spec §5; Invariantes): a lista de medicamentos em uso é estado
# atual reconstruído de eventos (só acréscimos), como a de problemas; um
# ativo por paciente + item; cada evento ligado a UMA consulta, adendo ou
# documento.
RSpec.describe Patients::ApplyMedicationEvent do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release!; import_test_catalog! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:patient) { consultation.patient }
  let(:losartana) { catalog_item(268_856) }

  def apply(source: { consultation: consultation }, **args)
    ApplicationRecord.transaction do
      Patient.lock.find(patient.id)
      described_class.call(patient: patient, by: doctor, source: source, **args)
    end
  end

  it "inclui, muda a posologia, suspende e reativa — cada passo um evento" do
    added = apply(action: "add", catalog_item: losartana, dosage_summary: "1 cp de manhã")
    medication = added.payload[:medication]
    expect(medication).to have_attributes(status: "active", origin: "prescription", label: "LOSARTANA POTÁSSICA 50 MG",
                                          catmat_code: 268_856, continuous: true, started_on: Time.zone.today)
    expect(apply(action: "add", catalog_item: losartana, dosage_summary: "1 cp de manhã").payload[:event]).to be_nil
    expect(apply(action: "add", catalog_item: losartana, dosage_summary: "1 cp 12/12h").payload[:event].kind).to eq("changed")
    expect(apply(action: "suspend", medication: medication.reload).payload[:medication].status).to eq("suspended")
    reactivated = apply(action: "add", catalog_item: losartana, dosage_summary: "1 cp de manhã")
    expect([ reactivated.payload[:event].kind, reactivated.payload[:medication].id ]).to eq([ "reactivated", medication.id ])
    expect(medication.events.order(:created_at).pluck(:kind)).to eq(%w[added changed suspended reactivated])
    expect(PatientMedication.where(patient: patient).count).to eq(1)
    payloads = DomainEvent.where(name: "patient_medication.changed").pluck(:payload)
    expect(payloads.last).to eq("patient_medication_id" => medication.id, "kind" => "reactivated", "consultation_id" => consultation.id)
    expect(payloads.to_json).not_to include("LOSARTANA", "12/12h")
  end

  it "informado de fora com o mesmo item ativo → already_active; reativar ativo → already_active" do
    medication = apply(action: "add", catalog_item: losartana, dosage_summary: "1 cp").payload[:medication]
    expect(apply(action: "add", origin: "external", catalog_item: losartana, on_active: :fail).reason).to eq(:already_active)
    expect(apply(action: "reactivate", medication: medication).reason).to eq(:already_active)
  end

  it "texto livre: casa sem acento, caixa ou espaço extra" do
    first = apply(action: "add", origin: "external", free_text: "Chá de camomila", dosage_summary: "à noite", on_active: :fail)
    expect(first.payload[:medication]).to have_attributes(free_text: "Chá de camomila", catalog_item_id: nil, origin: "external")
    expect(apply(action: "add", origin: "external", free_text: "  CHA  de Camomila ", on_active: :fail).reason).to eq(:already_active)
  end

  it "medicamento de outro paciente, ação desconhecida ou suspender o que não está ativo → invalid_medication" do
    medication = apply(action: "add", catalog_item: losartana).payload[:medication]
    other = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2, full_name: "João Souza")).patient
    expect(ApplicationRecord.transaction { described_class.call(patient: other, action: "suspend", by: doctor, source: { consultation: consultation }, medication: medication) }.reason)
      .to eq(:invalid_medication)
    expect(apply(action: "apagar", medication: medication).reason).to eq(:invalid_medication)
    apply(action: "suspend", medication: medication)
    expect(apply(action: "suspend", medication: medication.reload).reason).to eq(:invalid_medication)
  end

  it "fonte: exatamente uma entre consulta, adendo e documento" do
    expect { apply(action: "add", catalog_item: losartana, source: {}) }.to raise_error(ArgumentError)
    expect do
      described_class.call(patient: patient, action: "add", by: doctor, catalog_item: losartana,
                           source: { consultation: consultation, document: raw_document!(consultation: consultation) })
    end.to raise_error(ArgumentError)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/commands/patients/apply_medication_event_spec.rb`
Expected: FAIL (`uninitialized constant Patients::ApplyMedicationEvent`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/patients/apply_medication_event.rb
# Único caminho de escrita da lista de medicamentos em uso (ADR 0033; spec
# §5; Invariantes). Grava o evento ANTES de mudar o estado — o trigger
# patient_medications_guard exige um evento da mesma transação com o mesmo
# status. Quem chama abriu a transação e travou o paciente
# (ClinicalDocuments::Issue, PatientMedicationsController).
#   add        — sem igual: incluir; igual suspenso: reativar (com a posologia
#                nova); igual ativo: on_active :change → changed (ou nada, se
#                igual), :fail → already_active
#   suspend    — ativo → suspenso
#   reactivate — suspenso → ativo (already_active se já há um ativo igual)
# "Igual" = mesmo item do catálogo; no texto livre, o texto sem acento, caixa
# nem espaço extra (o texto é cifrado: compara em Ruby, com o paciente travado).
module Patients
  module ApplyMedicationEvent
    ACTIONS = %w[add suspend reactivate].freeze
    SOURCES = %i[consultation addendum document].freeze
    MAX_SUMMARY = 200

    module_function

    def call(patient:, action:, by:, source:, origin: "prescription", catalog_item: nil, free_text: nil,
             dosage_summary: nil, continuous: true, medication: nil, on_active: :change, on: Time.zone.today)
      link = source.slice(*SOURCES).compact
      raise ArgumentError, "source: consulta, adendo OU documento" unless link.size == 1 && source.keys.size == 1
      return Result.fail(:invalid_medication) unless ACTIONS.include?(action.to_s)
      return Result.fail(:invalid_medication) if medication && medication.patient_id != patient.id

      summary = dosage_summary.to_s.squish.first(MAX_SUMMARY).presence
      origin_data = { by: by, link: link }
      case action.to_s
      when "add" then add(patient, catalog_item, free_text, summary, continuous, origin, on_active, on, origin_data)
      when "suspend"
        return Result.fail(:invalid_medication) unless medication&.lock!&.active?

        write(medication, "suspended", { status: "suspended" }, origin_data)
      when "reactivate"
        return Result.fail(:invalid_medication) unless medication

        medication.lock!
        return Result.fail(:already_active) if medication.active? || same(patient, medication).any?(&:active?)

        write(medication, "reactivated", { status: "active" }, origin_data)
      end
    end

    def add(patient, catalog_item, free_text, summary, continuous, origin, on_active, on, origin_data)
      text = catalog_item ? nil : free_text.to_s.squish
      return Result.fail(:invalid_medication) if catalog_item.nil? && text.blank?

      existing = catalog_item ? PatientMedication.where(patient_id: patient.id, catalog_item_id: catalog_item.id).lock.to_a : same_text(patient, text)
      active = existing.find(&:active?)
      if active
        return Result.fail(:already_active) if on_active == :fail
        return Result.ok(medication: active, event: nil) if active.dosage_summary == summary && active.continuous == continuous

        return write(active, "changed", { dosage_summary: summary, continuous: continuous }, origin_data)
      end
      suspended = existing.max_by(&:updated_at)
      return write(suspended, "reactivated", { status: "active", dosage_summary: summary, continuous: continuous }, origin_data) if suspended

      medication = PatientMedication.new(id: SecureRandom.uuid, patient: patient, catalog_item_id: catalog_item&.id,
                                         catmat_code: catalog_item&.catmat_code, free_text: text,
                                         label: catalog_item ? catalog_item.label : text, dosage_summary: summary,
                                         continuous: continuous, status: "active", origin: origin, started_on: on)
      write(medication, "added", {}, origin_data)
    end

    def same(patient, medication)
      return PatientMedication.where(patient_id: patient.id, catalog_item_id: medication.catalog_item_id).where.not(id: medication.id).to_a if medication.catalog_item_id

      same_text(patient, medication.free_text).reject { |row| row.id == medication.id }
    end

    def same_text(patient, text)
      key = normalize(text)
      PatientMedication.where(patient_id: patient.id, catalog_item_id: nil).lock.to_a.select { |row| normalize(row.free_text) == key }
    end

    def normalize(text) = I18n.transliterate(text.to_s).downcase.squish

    def write(medication, kind, attrs, origin_data)
      medication.assign_attributes(attrs)
      link = origin_data[:link]
      event = PatientMedicationEvent.create!(
        patient_medication_id: medication.id, kind: kind, consultation_id: link[:consultation]&.id,
        addendum_id: link[:addendum]&.id, clinical_document_id: link[:document]&.id, user: origin_data[:by],
        status_after: medication.status, dosage_summary: medication.dosage_summary
      )
      medication.save!
      ids = { document_id: link[:document]&.id, consultation_id: link[:consultation]&.id, addendum_id: link[:addendum]&.id }.compact
      DomainEvents.publish("patient_medication.changed", patient_medication_id: medication.id, kind: kind, **ids)
      Result.ok(medication: medication, event: event)
    end
    private_class_method :add, :same, :same_text, :normalize, :write
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/commands/patients/apply_medication_event_spec.rb spec/models/clinical_document_tables_guard_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/commands/patients/apply_medication_event.rb spec/commands/patients/apply_medication_event_spec.rb
git commit -m "feat: apply patient medication changes only through events

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — Documentos (F-19.19 a F-19.22)

### Task 13: Matriz de quem emite e o conteúdo do atestado, da declaração e da requisição

**Files:**
- Create: `app/services/clinical_documents/issuers.rb`, `app/services/clinical_documents/content/input.rb`, `app/services/clinical_documents/content/sick_note.rb`, `app/services/clinical_documents/content/declaration.rb`, `app/services/clinical_documents/content/exam_requisition.rb`
- Test: `spec/services/clinical_documents/issuers_spec.rb`, `spec/services/clinical_documents/content_spec.rb`

**Interfaces:**
- Consumes: `ClinicalTerms.find("cid10", code)`, `ClinicalTerms::SigtapExams.label`, `Consultations::Effective.call`, `ProfessionalLink.active`.
- Produces:
  - `ClinicalDocuments::Issuers.role(cbo) -> :physician | :dentist | :nurse | :other`; `.allowed?(kind, cbo) -> bool`; `.declaration_staff(user, attendance) -> [:ok, cbo_or_nil] | [:missing_role, nil]`.
  - `ClinicalDocuments::Content::Input.hash(value) -> Hash`, `.invalid(field) -> Result`, `.text(value, range) -> String | nil | :invalid`, `.date(value) -> Date|nil`.
  - `ClinicalDocuments::Content::SickNote.call(input, today:) -> Result(content:)`; `Declaration.call(input, attendance:) -> Result(content:)`; `ExamRequisition.call(input, consultation:) -> Result(content:)`. Falhas: `invalid_content` (`field`), `cid_requires_authorization`, `companion_cid_not_allowed`, `no_exam_requests`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/clinical_documents/issuers_spec.rb
require "rails_helper"

# Spec §4 (Desvio 9): atestado só médico (2251–2253) e dentista (2232);
# receita médico, dentista e enfermeiro (2235, por protocolo); declaração e
# requisição, a autora da consulta. Declaração pelo atendimento: recepção ou
# profissional com vínculo ativo na unidade.
RSpec.describe ClinicalDocuments::Issuers do
  MATRIX = {
    "225125" => [ :physician, { "sick_note" => true, "prescription" => true } ],
    "225250" => [ :physician, { "sick_note" => true, "prescription" => true } ],
    "225320" => [ :physician, { "sick_note" => true, "prescription" => true } ],
    "223208" => [ :dentist, { "sick_note" => true, "prescription" => true } ],
    "223505" => [ :nurse, { "sick_note" => false, "prescription" => true } ],
    "223605" => [ :other, { "sick_note" => false, "prescription" => false } ],
    "251510" => [ :other, { "sick_note" => false, "prescription" => false } ]
  }.freeze

  MATRIX.each do |cbo, (role, kinds)|
    it "CBO #{cbo}: #{role}" do
      expect(described_class.role(cbo)).to eq(role)
      kinds.each { |kind, allowed| expect(described_class.allowed?(kind, cbo)).to be(allowed), "#{kind} #{cbo}" }
      expect(described_class.allowed?("attendance_declaration", cbo)).to be(true)
      expect(described_class.allowed?("exam_requisition", cbo)).to be(true)
    end
  end

  describe ".declaration_staff" do
    before { clinical_city! }

    let(:unit) { create_unit }
    let(:attendance) { walk_in_attendance!(unit, citizen: verified_citizen!(1)) }

    it "recepção sem CBO; profissional com vínculo na unidade com o CBO; o resto recusado" do
      expect(described_class.declaration_staff(verifier!, attendance)).to eq([ :ok, nil ])
      expect(described_class.declaration_staff(nurse!(unit), attendance)).to eq([ :ok, "223505" ])
      expect(described_class.declaration_staff(doctor!(create_unit("UBS Norte")), attendance)).to eq([ :missing_role, nil ])
      expect(described_class.declaration_staff(municipal_admin!, attendance)).to eq([ :missing_role, nil ])
    end
  end
end
```

```ruby
# spec/services/clinical_documents/content_spec.rb
require "rails_helper"

# Spec §4; contrato §2/§3: o conteúdo de cada tipo, validado campo a campo.
RSpec.describe "Conteúdo dos documentos" do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:today) { Date.new(2026, 10, 9) }

  describe ClinicalDocuments::Content::SickNote do
    def call(input) = described_class.call(input, today: today)

    it "afastamento: dias, início (padrão hoje) e CID só com autorização" do
      expect(call("type" => "leave", "days" => 3).payload[:content])
        .to eq("type" => "leave", "days" => 3, "start_on" => "2026-10-09", "cid_authorized" => false)
      content = call("type" => "leave", "days" => 2, "start_on" => "2026-10-08", "cid10" => { "code" => "I10" },
                     "cid_authorized" => true, "note" => "Repouso relativo").payload[:content]
      expect(content).to include("cid10" => { "code" => "I10", "label" => "Hipertensão essencial (primária)" },
                                 "cid_authorized" => true, "note" => "Repouso relativo")
      expect(call("type" => "leave", "days" => 2, "cid10" => { "code" => "I10" }).reason).to eq(:cid_requires_authorization)
      expect(call("type" => "leave", "days" => 2, "cid10" => { "code" => "Z999" }, "cid_authorized" => true).details).to eq(field: "cid10")
    end

    it "afastamento recusa dias e início fora da faixa e campos do acompanhante" do
      expect(call("type" => "leave", "days" => 0).details).to eq(field: "days")
      expect(call("type" => "leave", "days" => "3").details).to eq(field: "days")
      expect(call("type" => "leave", "days" => 3, "start_on" => "2026-08-01").details).to eq(field: "start_on")
      expect(call("type" => "leave", "days" => 3, "start_on" => "2026-10-10").details).to eq(field: "start_on")
      expect(call("type" => "leave", "days" => 3, "companion_name" => "Ana").details).to eq(field: "companion_name")
      expect(call("type" => "outro").details).to eq(field: "type")
    end

    it "acompanhante: nome, parentesco, motivo CLT; nunca CID nem dias" do
      ok = call("type" => "companion", "companion_name" => "Ana Paula Souza", "companion_kinship" => "mãe",
                "companion_reason" => "clt_473_xi")
      expect(ok.payload[:content]).to eq("type" => "companion", "companion_name" => "Ana Paula Souza",
                                         "companion_kinship" => "mãe", "companion_reason" => "clt_473_xi", "cid_authorized" => false)
      expect(call("type" => "companion", "companion_name" => "Ana Paula", "companion_kinship" => "mãe",
                  "companion_reason" => "clt_473_xi", "cid10" => { "code" => "I10" }, "cid_authorized" => true).reason)
        .to eq(:companion_cid_not_allowed)
      expect(call("type" => "companion", "companion_name" => "Ana Paula", "companion_kinship" => "mãe",
                  "companion_reason" => "clt_473_xi", "days" => 1).details).to eq(field: "days")
      expect(call("type" => "companion", "companion_name" => "An", "companion_kinship" => "mãe",
                  "companion_reason" => "clt_473_xi").details).to eq(field: "companion_name")
      expect(call("type" => "companion", "companion_name" => "Ana Paula", "companion_kinship" => "mãe",
                  "companion_reason" => "art_999").details).to eq(field: "companion_reason")
    end
  end

  describe ClinicalDocuments::Content::Declaration do
    let(:unit) { create_unit }
    let(:attendance) { walk_in_attendance!(unit, citizen: verified_citizen!(1), checked_in_at: Time.zone.parse("2026-10-09 08:12")) }

    it "horário de chegada do check-in por padrão; período no lugar do horário; unidade do atendimento" do
      content = described_class.call({}, attendance: attendance).payload[:content]
      expect(content).to eq("date" => "2026-10-09", "arrived_at" => "08:12", "unit_name" => unit.name)
      content = described_class.call({ "period" => "morning", "companion_name" => "Carlos Lima",
                                       "issuer_registration" => "12345" }, attendance: attendance).payload[:content]
      expect(content).to eq("date" => "2026-10-09", "period" => "morning", "unit_name" => unit.name,
                            "companion_name" => "Carlos Lima", "issuer_registration" => "12345")
    end

    it "recusa período com horário, horário inválido e saída antes da chegada" do
      expect(described_class.call({ "period" => "morning", "arrived_at" => "08:00" }, attendance: attendance).details).to eq(field: "period")
      expect(described_class.call({ "arrived_at" => "25:00" }, attendance: attendance).details).to eq(field: "arrived_at")
      expect(described_class.call({ "arrived_at" => "10:00", "left_at" => "09:00" }, attendance: attendance).details).to eq(field: "left_at")
      expect(described_class.call({ "period" => "night" }, attendance: attendance).details).to eq(field: "period")
    end
  end

  describe ClinicalDocuments::Content::ExamRequisition do
    let(:unit) { create_unit }
    let(:doctor) { doctor!(unit) }

    it "renderiza os exames vigentes da consulta, com a competência; sem exame → no_exam_requests" do
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
      content = described_class.call({ "note" => "Jejum de 8 horas" }, consultation: consultation).payload[:content]
      expect(content["exams"]).to eq([ { "sigtap_code" => "0202010503", "label" => "DOSAGEM DE HEMOGLOBINA GLICOSILADA",
                                         "competence" => Time.zone.today.strftime("%Y%m") } ])
      expect(content["note"]).to eq("Jejum de 8 horas")
      empty = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2), exam_requests: [])
      expect(described_class.call({}, consultation: empty).reason).to eq(:no_exam_requests)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/issuers_spec.rb spec/services/clinical_documents/content_spec.rb`
Expected: FAIL (constantes inexistentes).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/issuers.rb
# Quem emite o quê (ADR 0033; spec §4; Desvio 9). O CBO é o da consulta (o
# vínculo com que a autora atendeu). Atestado: médico e dentista (CFM
# 2.382/2024 art. 9; Lei 5.081/1966). Receita: médico, dentista e enfermeiro —
# este só por protocolo (COFEN 801/2026; a trava é da receita). Declaração e
# requisição: qualquer autora de consulta. Declaração pelo atendimento:
# recepção (citizen_verifier) ou profissional com vínculo ativo na unidade.
module ClinicalDocuments
  module Issuers
    PREFIXES = { physician: %w[2251 2252 2253], dentist: %w[2232], nurse: %w[2235] }.freeze
    MATRIX = { "sick_note" => %i[physician dentist], "prescription" => %i[physician dentist nurse],
               "attendance_declaration" => :any, "exam_requisition" => :any }.freeze

    module_function

    def role(cbo)
      code = cbo.to_s
      PREFIXES.find { |_role, prefixes| prefixes.any? { |prefix| code.start_with?(prefix) } }&.first || :other
    end

    def allowed?(kind, cbo)
      allowed = MATRIX.fetch(kind.to_s)
      allowed == :any || allowed.include?(role(cbo))
    end

    def declaration_staff(user, attendance)
      if user.has_role?("health_professional")
        link = ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id })
                               .where(health_unit_id: attendance.health_unit_id).order(:started_at, :id).first
        return [ :ok, link.cbo_code ] if link
      end
      user.has_role?("citizen_verifier") ? [ :ok, nil ] : [ :missing_role, nil ]
    end
  end
end
```

```ruby
# app/services/clinical_documents/content/input.rb
# Leitura defensiva do `content` do contrato (§2): tipos exatos, textos
# aparados com limite, datas ISO. Nenhum valor vai a mensagem de erro — só o
# nome do campo.
module ClinicalDocuments
  module Content
    module Input
      module_function

      def hash(value)
        value = value.to_unsafe_h if value.respond_to?(:to_unsafe_h)
        value.is_a?(Hash) ? value.to_h.stringify_keys : {}
      end

      def invalid(field) = Result.fail(:invalid_content, details: { field: field })

      # nil quando ausente ou vazio; :invalid quando não é texto ou passa do limite.
      def text(value, range)
        return nil if value.nil?
        return :invalid unless value.is_a?(String)

        trimmed = value.gsub(/\r\n?/, "\n").strip
        return nil if trimmed.empty?

        range.cover?(trimmed.size) ? trimmed : :invalid
      end

      def date(value)
        value.is_a?(String) && value.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(value) : nil
      rescue Date::Error
        nil
      end
    end
  end
end
```

```ruby
# app/services/clinical_documents/content/sick_note.rb
# Atestado (spec §4; CFM 1.658/2002 art. 3º): afastamento (dias e início) ou
# acompanhante (nome, parentesco, motivo da CLT art. 473 X/XI/XII). CID só com
# autorização expressa do paciente (registrada no documento); acompanhante
# nunca com CID.
module ClinicalDocuments
  module Content
    module SickNote
      TYPES = %w[leave companion].freeze
      COMPANION_REASONS = %w[clt_473_x clt_473_xi clt_473_xii other].freeze
      COMPANION_KEYS = %w[companion_name companion_kinship companion_reason].freeze
      MAX_DAYS = 365
      BACKDATE_DAYS = 30

      module_function

      def call(input, today: Time.zone.today)
        input = Input.hash(input)
        return Input.invalid("type") unless TYPES.include?(input["type"])
        return Input.invalid("cid_authorized") unless [ nil, true, false ].include?(input["cid_authorized"])

        note = Input.text(input["note"], 1..500)
        return Input.invalid("note") if note == :invalid

        input["type"] == "leave" ? leave(input, note, today) : companion(input, note)
      end

      def leave(input, note, today)
        extra = COMPANION_KEYS.find { |key| input.key?(key) }
        return Input.invalid(extra) if extra

        days = input["days"]
        return Input.invalid("days") unless days.is_a?(Integer) && days.between?(1, MAX_DAYS)

        start_on = input.key?("start_on") ? Input.date(input["start_on"]) : today
        return Input.invalid("start_on") unless start_on&.between?(today - BACKDATE_DAYS, today)

        authorized = input["cid_authorized"] == true
        cid = nil
        unless input["cid10"].nil?
          return Result.fail(:cid_requires_authorization) unless authorized

          code = input["cid10"].is_a?(Hash) ? input["cid10"]["code"] : nil
          found = code.is_a?(String) ? ClinicalTerms.find("cid10", code) : nil
          return Input.invalid("cid10") unless found

          cid = { "code" => found.code, "label" => found.label }
        end
        Result.ok(content: { "type" => "leave", "days" => days, "start_on" => start_on.iso8601, "cid10" => cid,
                             "cid_authorized" => authorized, "note" => note }.compact)
      end

      def companion(input, note)
        return Result.fail(:companion_cid_not_allowed) unless input["cid10"].nil?
        return Input.invalid("days") if input.key?("days")
        return Input.invalid("start_on") if input.key?("start_on")

        name = Input.text(input["companion_name"], 3..120)
        return Input.invalid("companion_name") unless name.is_a?(String)

        kinship = Input.text(input["companion_kinship"], 2..60)
        return Input.invalid("companion_kinship") unless kinship.is_a?(String)
        return Input.invalid("companion_reason") unless COMPANION_REASONS.include?(input["companion_reason"])

        Result.ok(content: { "type" => "companion", "companion_name" => name, "companion_kinship" => kinship,
                             "companion_reason" => input["companion_reason"], "cid_authorized" => false, "note" => note }.compact)
      end
      private_class_method :leave, :companion
    end
  end
end
```

```ruby
# app/services/clinical_documents/content/declaration.rb
# Declaração de comparecimento (spec §4; Parecer Cofen 44/2025; modelo do
# PEC): data do atendimento, chegada/saída (padrão: check-in e encerramento)
# ou período, unidade do atendimento, acompanhante opcional e, no papel da
# recepção, a matrícula de quem emite.
module ClinicalDocuments
  module Content
    module Declaration
      PERIODS = %w[morning afternoon full_day].freeze
      HHMM = /\A([01]\d|2[0-3]):[0-5]\d\z/

      module_function

      def call(input, attendance:)
        input = Input.hash(input)
        content = { "date" => attendance.checked_in_at.in_time_zone.to_date.iso8601 }
        if input.key?("period")
          return Input.invalid("period") unless PERIODS.include?(input["period"])
          return Input.invalid("period") if input.key?("arrived_at") || input.key?("left_at")

          content["period"] = input["period"]
        else
          arrived = input.fetch("arrived_at") { attendance.checked_in_at.in_time_zone.strftime("%H:%M") }
          left = input.fetch("left_at") { attendance.closed_at&.in_time_zone&.strftime("%H:%M") }
          return Input.invalid("arrived_at") unless arrived.is_a?(String) && arrived.match?(HHMM)
          return Input.invalid("left_at") unless left.nil? || (left.is_a?(String) && left.match?(HHMM) && left >= arrived)

          content.merge!("arrived_at" => arrived, "left_at" => left)
        end
        companion = Input.text(input["companion_name"], 3..120)
        return Input.invalid("companion_name") if companion == :invalid

        registration = Input.text(input["issuer_registration"], 1..30)
        return Input.invalid("issuer_registration") if registration == :invalid

        Result.ok(content: content.merge("unit_name" => attendance.health_unit.name, "companion_name" => companion,
                                         "issuer_registration" => registration).compact)
      end
    end
  end
end
```

```ruby
# app/services/clinical_documents/content/exam_requisition.rb
# Requisição de exames (spec §4): a renderização dos pedidos estruturados da
# consulta (consultation_exam_requests, os vigentes depois dos adendos), com a
# competência SIGTAP gravada no ato. O módulo 21 consome a entidade, não o PDF.
module ClinicalDocuments
  module Content
    module ExamRequisition
      module_function

      def call(input, consultation:)
        input = Input.hash(input)
        exams = Consultations::Effective.call(consultation)[:exam_requests]
        return Result.fail(:no_exam_requests) if exams.empty?

        note = Input.text(input["note"], 1..500)
        return Input.invalid("note") if note == :invalid

        rows = exams.map do |exam|
          { "sigtap_code" => exam.sigtap_code, "label" => ClinicalTerms::SigtapExams.label(exam.sigtap_code, exam.sigtap_competence),
            "competence" => exam.sigtap_competence, "cid10_justification" => exam.cid10_justification }.compact
        end
        Result.ok(content: { "exams" => rows, "note" => note }.compact)
      end
    end
  end
end
```

> `walk_in_attendance!(unit, citizen:, checked_in_at:)` é o helper existente (`spec/support/screening_helpers.rb`) — o check-in nunca muda depois (trigger `attendances_guard`), então a data entra na criação. `finalized_consultation!(..., exam_requests: [])` passa a chave para o `draft_body` (sobrescreve a lista de exames).

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/issuers_spec.rb spec/services/clinical_documents/content_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/issuers.rb app/services/clinical_documents/content/input.rb app/services/clinical_documents/content/sick_note.rb app/services/clinical_documents/content/declaration.rb app/services/clinical_documents/content/exam_requisition.rb spec/services/clinical_documents/issuers_spec.rb spec/services/clinical_documents/content_spec.rb
git commit -m "feat: validate who issues each clinical document and its content

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Conteúdo da receita (catálogo, texto livre, controlado, antimicrobiano, enfermagem)

**Files:**
- Create: `app/services/clinical_documents/content/prescription.rb`
- Test: `spec/services/clinical_documents/prescription_content_spec.rb`

**Interfaces:**
- Consumes: `Content::Input` (Task 13), `MedicationCatalogItem`, `Medications::Substances::Matcher#scan`, `NursingProtocolVersion` (`#nursing_protocol`, `#items`), `NursingProtocol#current_version`, `PrescriptionItem::ROUTES`, `ClinicalDocuments::Json.{catalog_ref,number}`.
- Produces: `ClinicalDocuments::Content::Prescription.call(input, role:, patient:, catalog:, matcher:, today: Time.zone.today, city_cnpj: nil) -> Result` — `ok(content: Hash, items: Array<Hash>, antimicrobial: bool)`, onde cada item de `items` tem as colunas de `PrescriptionItem` (`position`, `medication_catalog_item_id`, `catmat_code`, `catalog_release_id`, `free_text`, `printed_description`, `quantity`, `quantity_unit`, `route`, `dosage_instructions`, `duration_days`, `continuous`, `antimicrobial`, `nursing_protocol_version_item_id`, `reason_problem_id`); falhas `invalid_content` (`field`), `invalid_item` (`index`, `field`), `controlled_not_allowed` (`index`), `free_text_not_allowed` (`index`), `not_in_nursing_protocol` (`index` quando o item está fora), `above_protocol_max_dose` (`index`), `city_cnpj_missing`. `catalog` = `{ id => MedicationCatalogItem }` lido fora da transação. Ordem das checagens (para a mensagem ser previsível): forma dos itens → controlado → enfermagem (texto livre, protocolo, itens, dose, CNPJ) → validade.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/clinical_documents/prescription_content_spec.rb
require "rails_helper"

# Spec §5; ADR 0033 (Invariantes): nenhum controlado no 19c; antimicrobiano só
# papel, 2 vias, 10 dias; texto livre permitido (menos para enfermagem) e
# varrido nas listas; enfermeiro só itens do protocolo vigente, dentro da dose
# máxima, com o CNPJ da cidade.
RSpec.describe ClinicalDocuments::Content::Prescription do
  before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let!(:catalog_by_code) { import_test_catalog! }
  let(:catalog) { catalog_by_code.values.index_by(&:id) }
  let(:matcher) { Medications::Substances.matcher }
  let(:unit) { create_unit }
  let(:patient) { finalized_consultation!(unit: unit, doctor: doctor!(unit), citizen: verified_citizen!(1)).patient }
  let(:today) { Time.zone.today }

  def item(code = 268_856, **over)
    { "catalog_item" => { "id" => catalog_item(code).id }, "quantity" => 30, "quantity_unit" => "comprimido",
      "route" => "oral", "dosage_instructions" => "1 comprimido pela manhã", "duration_days" => 30,
      "continuous" => true }.merge(over.transform_keys(&:to_s))
  end

  def call(items, role: :physician, **input)
    described_class.call({ "items" => items }.merge(input.transform_keys(&:to_s)), role: role, patient: patient,
                         catalog: catalog, matcher: matcher, today: today, city_cnpj: input.delete(:cnpj) || "76001234000115")
  end

  it "médico: item do catálogo vira item da receita com o código e a release da emissão" do
    result = call([ item ])
    row = result.payload[:items].sole
    losartana = catalog_item(268_856)
    expect(row).to include(position: 1, medication_catalog_item_id: losartana.id, catmat_code: 268_856,
                           catalog_release_id: losartana.last_release_id, free_text: nil,
                           printed_description: "LOSARTANA POTÁSSICA 50 MG", quantity: BigDecimal("30"),
                           route: "oral", continuous: true, antimicrobial: false)
    expect(result.payload[:content]).to eq(
      "items" => [ { "position" => 1, "catalog_item" => ClinicalDocuments::Json.catalog_ref(losartana).deep_stringify_keys,
                     "printed_description" => "LOSARTANA POTÁSSICA 50 MG", "quantity" => 30, "quantity_unit" => "comprimido",
                     "route" => "oral", "dosage_instructions" => "1 comprimido pela manhã", "duration_days" => 30,
                     "continuous" => true, "antimicrobial" => false } ],
      "catalog_release" => losartana.last_release_id, "antimicrobial" => false, "copies" => 1
    )
    expect(call([ item ], valid_until: (today + 60).iso8601).payload[:content]["valid_until"]).to eq((today + 60).iso8601)
    expect(call([ item ], valid_until: (today - 1).iso8601).details).to eq(field: "valid_until")
  end

  it "controlado do catálogo ou em texto livre → controlled_not_allowed com o índice" do
    expect(call([ item, item(267_197) ]).then { |r| [ r.reason, r.details ] }).to eq([ :controlled_not_allowed, { index: 1 } ])
    free = { "free_text" => "Diazepam 10 mg", "quantity" => 10, "quantity_unit" => "comprimido", "route" => "oral",
             "dosage_instructions" => "1 à noite" }
    expect(call([ free ]).reason).to eq(:controlled_not_allowed)
  end

  it "antimicrobiano (catálogo ou texto livre) → 2 vias, validade de 10 dias" do
    content = call([ item(271_089, continuous: false, duration_days: 7) ]).payload[:content]
    expect(content).to include("antimicrobial" => true, "copies" => 2, "valid_until" => (today + 10).iso8601)
    free = { "free_text" => "Azitromicina 500mg", "quantity" => 3, "quantity_unit" => "comprimido", "route" => "oral",
             "dosage_instructions" => "1 ao dia" }
    expect(call([ free ]).payload[:content]).to include("antimicrobial" => true, "copies" => 2)
  end

  it "forma do item: um e só um entre catálogo e texto; escondido; quantidade; via; motivo de outro paciente; limite" do
    expect(call([ item.merge("free_text" => "Losartana") ]).details).to eq(index: 0, field: "catalog_item")
    expect(call([ item(384_258) ]).details).to eq(index: 0, field: "catalog_item")
    expect(call([ item(quantity: 0) ]).details).to eq(index: 0, field: "quantity")
    expect(call([ item(quantity: 1.555) ]).details).to eq(index: 0, field: "quantity")
    expect(call([ item(route: "anal") ]).details).to eq(index: 0, field: "route")
    expect(call([ item(dosage_instructions: "x") ]).details).to eq(index: 0, field: "dosage_instructions")
    expect(call([ item(reason_problem_id: SecureRandom.uuid) ]).details).to eq(index: 0, field: "reason_problem_id")
    expect(call([]).details).to eq(field: "items")
    expect(call(Array.new(21) { item }).details).to eq(field: "items")
  end

  describe "enfermagem" do
    let(:admin) { municipal_admin! }
    let!(:protocol_version) do
      params = { "title" => "Saúde da mulher", "number" => "PE-01", "year" => 2026, "valid_from" => (today - 30).iso8601,
                 "items" => [ { "catalog_item_id" => catalog_item(267_503).id, "max_dose" => { "quantity" => 30, "unit" => "comprimido" } },
                              { "catalog_item_id" => catalog_item(268_162).id } ] }
      NursingProtocols::Save.create(params: params, by: admin, catalog: catalog).payload[:protocol].current_version(on: today)
    end

    def nursing(items, **input) = call(items, role: :nurse, nursing_protocol: { "version_id" => protocol_version.id }, **input)

    it "item do protocolo vigente, dentro da dose, com CNPJ → conteúdo com protocolo e CNPJ" do
      result = nursing([ item(267_503, quantity: 30, continuous: false) ])
      expect(result.payload[:content]).to include(
        "nursing_protocol" => { "id" => protocol_version.nursing_protocol_id, "title" => "Saúde da mulher", "number" => "PE-01",
                                "year" => 2026, "version_id" => protocol_version.id },
        "city_cnpj" => "76001234000115"
      )
      expect(result.payload[:items].sole[:nursing_protocol_version_item_id]).to eq(protocol_version.items.find_by(catalog_item_id: catalog_item(267_503).id).id)
    end

    it "texto livre, sem protocolo, item fora, acima da dose ou unidade diferente, sem CNPJ" do
      free = { "free_text" => "Ácido fólico 5 mg", "quantity" => 30, "quantity_unit" => "comprimido", "route" => "oral",
               "dosage_instructions" => "1 ao dia" }
      expect([ nursing([ free ]).reason, nursing([ free ]).details ]).to eq([ :free_text_not_allowed, { index: 0 } ])
      expect(call([ item(267_503) ], role: :nurse).reason).to eq(:not_in_nursing_protocol)
      expect(nursing([ item(267_503), item(268_856) ]).then { |r| [ r.reason, r.details ] }).to eq([ :not_in_nursing_protocol, { index: 1 } ])
      expect(nursing([ item(267_503, quantity: 31) ]).then { |r| [ r.reason, r.details ] }).to eq([ :above_protocol_max_dose, { index: 0 } ])
      expect(nursing([ item(267_503, quantity: 1, quantity_unit: "caixa") ]).reason).to eq(:above_protocol_max_dose)
      expect(nursing([ item(268_162, quantity: 5, quantity_unit: "bisnaga") ])).to be_ok # sem dose máxima
      expect(described_class.call({ "items" => [ item(267_503) ], "nursing_protocol" => { "version_id" => protocol_version.id } },
                                  role: :nurse, patient: patient, catalog: catalog, matcher: matcher, today: today, city_cnpj: nil).reason)
        .to eq(:city_cnpj_missing)
    end

    it "versão que não é a vigente (substituída ou vencida) → not_in_nursing_protocol" do
      protocol = protocol_version.nursing_protocol
      NursingProtocols::Save.add_version(protocol: protocol, params: { "valid_from" => today.iso8601, "items" => [ { "catalog_item_id" => catalog_item(267_503).id } ] },
                                         by: admin, catalog: catalog)
      expect(nursing([ item(267_503) ]).reason).to eq(:not_in_nursing_protocol)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/prescription_content_spec.rb`
Expected: FAIL (`uninitialized constant ClinicalDocuments::Content::Prescription`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/content/prescription.rb
# Receita comum (ADR 0033; spec §5; Lei 5.991 art. 35; Lei 9.787 — DCB):
# itens do catálogo (código CATMAT e release da emissão) ou texto livre
# (marcado e varrido nas listas Anvisa). Nenhum controlado (19d). Antimicrobiano
# força papel no comando, 2 vias e validade de 10 dias (RDC 471/2021).
# Enfermeiro (COFEN 801/2026): só itens da versão VIGENTE do protocolo
# escolhido, dentro da dose máxima (quantidade por item, mesma unidade), sem
# texto livre, com o CNPJ da cidade.
module ClinicalDocuments
  module Content
    module Prescription
      MAX_ITEMS = 20
      ANTIMICROBIAL_DAYS = 10
      MAX_VALIDITY_DAYS = 365
      UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

      module_function

      def call(input, role:, patient:, catalog:, matcher:, today: Time.zone.today, city_cnpj: nil)
        input = Input.hash(input)
        raw = input["items"]
        return Input.invalid("items") unless raw.is_a?(Array) && raw.size.between?(1, MAX_ITEMS)

        problems = PatientProblem.where(patient_id: patient.id).pluck(:id).to_set
        items = []
        raw.each_with_index do |entry, index|
          parsed = item(entry, index, catalog, matcher, problems)
          return parsed if parsed.is_a?(Result)

          items << parsed
        end
        items.each_with_index { |it, index| return Result.fail(:controlled_not_allowed, details: { index: index }) if it[:controlled] }

        version = nil
        if role == :nurse
          version = nursing(input, items, today, city_cnpj)
          return version if version.is_a?(Result)
        end
        antimicrobial = items.any? { |it| it[:antimicrobial] }
        valid_until = validity(input, antimicrobial, today)
        return valid_until if valid_until.is_a?(Result)

        rows = rows(items)
        content = { "items" => rows.zip(items).map { |row, it| content_item(row, it) },
                    "nursing_protocol" => version && protocol_json(version),
                    "city_cnpj" => (city_cnpj if version), "antimicrobial" => antimicrobial,
                    "copies" => antimicrobial ? 2 : 1, "valid_until" => valid_until&.iso8601 }.compact
        # Uma vez por receita (esquema clinical-v1.1.0): a release do catálogo na emissão; null se só texto livre.
        content["catalog_release"] = rows.filter_map { |row| row[:catalog_release_id] }.first
        Result.ok(content: content, items: rows, antimicrobial: antimicrobial)
      end

      def item(entry, index, catalog, matcher, problems)
        entry = Input.hash(entry)
        bad = ->(field) { Result.fail(:invalid_item, details: { index: index, field: field }) }
        reference = entry["catalog_item"].is_a?(Hash) ? entry["catalog_item"]["id"] : nil
        free = entry["free_text"].is_a?(String) ? entry["free_text"].squish : nil
        return bad.("catalog_item") if reference.nil? == free.nil?

        catalog_entry = nil
        if reference
          catalog_entry = catalog[reference.to_s]
          return bad.("catalog_item") unless catalog_entry && catalog_entry.status == "active" && !catalog_entry.hidden
        else
          return bad.("free_text") unless free.size.between?(3, 200)
        end
        quantity = decimal(entry["quantity"])
        return bad.("quantity") unless quantity && quantity.positive? && quantity <= 9999 && quantity.scale <= 2

        unit = Input.text(entry["quantity_unit"], 1..30)
        return bad.("quantity_unit") unless unit.is_a?(String)
        return bad.("route") unless PrescriptionItem::ROUTES.include?(entry["route"])

        dosage = Input.text(entry["dosage_instructions"], 3..500)
        return bad.("dosage_instructions") unless dosage.is_a?(String)

        duration = entry["duration_days"]
        return bad.("duration_days") unless duration.nil? || (duration.is_a?(Integer) && duration.between?(1, 365))
        return bad.("continuous") unless [ nil, true, false ].include?(entry["continuous"])

        reason = entry["reason_problem_id"]
        return bad.("reason_problem_id") unless reason.nil? || (reason.is_a?(String) && problems.include?(reason))

        flags = catalog_entry ? catalog_entry : matcher.scan(free)
        { catalog: catalog_entry, free_text: free, quantity: quantity, quantity_unit: unit, route: entry["route"],
          dosage_instructions: dosage, duration_days: duration, continuous: entry["continuous"] == true,
          reason_problem_id: reason, antimicrobial: flags.antimicrobial, controlled: flags.controlled }
      end

      def nursing(input, items, today, city_cnpj)
        items.each_with_index { |it, index| return Result.fail(:free_text_not_allowed, details: { index: index }) if it[:free_text] }
        version_id = input["nursing_protocol"].is_a?(Hash) ? input["nursing_protocol"]["version_id"].to_s : ""
        version = version_id.match?(UUID) ? NursingProtocolVersion.includes(:nursing_protocol, :items).find_by(id: version_id) : nil
        return Result.fail(:not_in_nursing_protocol) unless version && version.nursing_protocol.current_version(on: today)&.id == version.id

        allowed = version.items.index_by(&:catalog_item_id)
        items.each_with_index do |it, index|
          entry = allowed[it[:catalog].id]
          return Result.fail(:not_in_nursing_protocol, details: { index: index }) unless entry

          if entry.max_quantity && (entry.max_quantity_unit.casecmp?(it[:quantity_unit]) == false || it[:quantity] > entry.max_quantity)
            return Result.fail(:above_protocol_max_dose, details: { index: index })
          end

          it[:protocol_item] = entry
        end
        return Result.fail(:city_cnpj_missing) if city_cnpj.blank?

        version
      end

      def validity(input, antimicrobial, today)
        return today + ANTIMICROBIAL_DAYS if antimicrobial
        return nil if input["valid_until"].nil?

        date = Input.date(input["valid_until"])
        date&.between?(today, today + MAX_VALIDITY_DAYS) ? date : Input.invalid("valid_until")
      end

      def rows(items)
        items.each_with_index.map do |it, index|
          entry = it[:catalog]
          { position: index + 1, medication_catalog_item_id: entry&.id, catmat_code: entry&.catmat_code,
            catalog_release_id: entry&.last_release_id, free_text: it[:free_text],
            printed_description: entry ? entry.label : it[:free_text], quantity: it[:quantity], quantity_unit: it[:quantity_unit],
            route: it[:route], dosage_instructions: it[:dosage_instructions], duration_days: it[:duration_days],
            continuous: it[:continuous], antimicrobial: it[:antimicrobial],
            nursing_protocol_version_item_id: it[:protocol_item]&.id, reason_problem_id: it[:reason_problem_id] }
        end
      end

      def content_item(row, it)
        { "position" => row[:position], "catalog_item" => it[:catalog] && Json.catalog_ref(it[:catalog]).deep_stringify_keys,
          "free_text" => row[:free_text], "printed_description" => row[:printed_description],
          "quantity" => Json.number(row[:quantity]), "quantity_unit" => row[:quantity_unit], "route" => row[:route],
          "dosage_instructions" => row[:dosage_instructions], "duration_days" => row[:duration_days],
          "continuous" => row[:continuous], "antimicrobial" => row[:antimicrobial],
          "reason_problem_id" => row[:reason_problem_id] }.compact
      end

      def protocol_json(version)
        protocol = version.nursing_protocol
        { "id" => protocol.id, "title" => protocol.title, "number" => protocol.number, "year" => protocol.year,
          "version_id" => version.id }
      end

      def decimal(value)
        return nil unless value.is_a?(Numeric) || (value.is_a?(String) && value.match?(/\A\d+(\.\d+)?\z/))

        BigDecimal(value.to_s)
      end
      private_class_method :item, :nursing, :validity, :rows, :content_item, :protocol_json, :decimal
    end
  end
end
```

> `flags = catalog_entry ? catalog_entry : matcher.scan(free)`: o item do catálogo responde `antimicrobial`/`controlled` como o `Flags` do `scan` (mesmos nomes) — é por isso que os dois cabem na mesma variável. `BigDecimal#scale` (casas decimais) existe desde o Ruby 2.7 (`bigdecimal/util`); se o executor preferir, troque por `quantity.round(2) == quantity`.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/prescription_content_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/content/prescription.rb spec/services/clinical_documents/prescription_content_spec.rb
git commit -m "feat: validate prescriptions against the catalog, Anvisa lists and nursing protocols

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Emissão (`ClinicalDocuments::Issue`), modo fixo e o pedido de assinatura

**Files:**
- Create: `app/commands/clinical_documents/issue.rb`
- Modify: `app/services/clinical_documents/json.rb`, `app/services/signatures/document_types.rb`
- Test: `spec/commands/clinical_documents/issue_spec.rb`

**Interfaces:**
- Consumes: Tasks 8, 12, 13, 14; `Signatures::{Gate,OpenRequest,Mode}`, `SignerCertificate.active`, `Medications::Search.available?`, `Medications::Substances.matcher`, `CityProfile.current`.
- Produces:
  - `ClinicalDocuments::Issue.call(kind:, content:, by:, consultation: nil, attendance: nil, replaces_document_id: nil, now: Time.current) -> Result` — `ok(document:)` | `fail(:invalid_content | :not_author | :cbo_not_allowed | :missing_role | :attendance_not_found_or_closed_long_ago | :catalog_unavailable | <falhas de conteúdo das Tasks 13–14>)`. Da consulta: autora (rascunho ou finalizada); do atendimento: só `attendance_declaration`, sempre papel.
  - Modo: `digital` ⇔ consulta presente ∧ `Signatures::Gate.usable?` (lido FORA da transação) ∧ autora com certificado ativo ∧ não antimicrobiano; senão `paper`. Digital → `Signatures::OpenRequest.call(document, now:, usable: true)` na mesma transação.
  - Receita: cria `PrescriptionItem`s; item `continuous` → `Patients::ApplyMedicationEvent` (`add`, `source: { document: }`, `on_active: :change`).
  - Evento `clinical_document.issued { document_id, kind, consultation_id | attendance_id, issue_mode }`.
  - `Signatures::DocumentTypes::MODELS["ClinicalDocument"] == "clinical_document"`.
  - `ClinicalDocuments::Json.document(document, signature: :load) -> Hash` (contrato §2) e `ClinicalDocuments::Json.medication(medication, catalog:) -> Hash`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/clinical_documents/issue_spec.rb
require "rails_helper"

# Spec §4–§5; ADR 0033: emissão da consulta (autora, também depois de
# finalizada) e do atendimento (declaração, sempre papel); modo escolhido no
# ato e nunca convertido; antimicrobiano sempre papel; receita de uso
# contínuo alimenta a lista de medicamentos.
RSpec.describe ClinicalDocuments::Issue do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:leave) { { "type" => "leave", "days" => 2 } }

  def issue(kind: "sick_note", content: leave, by: doctor, **args) = described_class.call(kind: kind, content: content, by: by, consultation: consultation, **args)

  it "atestado da autora: papel, códigos, conteúdo cifrado, evento só com ids" do
    document = issue.payload[:document]
    expect(document).to have_attributes(kind: "sick_note", status: "issued", issue_mode: "paper", author_user_id: doctor.id,
                                        cbo_code: "225125", patient_id: consultation.patient_id,
                                        attendance_id: consultation.attendance_id)
    expect(document.short_code).to match(/\A[2-9ABCDEFGHJKMNPQRSTUVWXYZ]{10}\z/)
    expect(document.verification_token.size).to eq(22)
    expect(document.content_data).to include("type" => "leave", "days" => 2)
    expect(ClinicalDocument.where(id: document.id).pick(Arel.sql("content"))).not_to include("leave")
    expect(DomainEvent.where(name: "clinical_document.issued").last.payload)
      .to eq("document_id" => document.id, "kind" => "sick_note", "consultation_id" => consultation.id, "issue_mode" => "paper")
  end

  it "não autora → not_author; enfermeira não emite atestado → cbo_not_allowed; falha não grava nada" do
    expect(issue(by: doctor!(unit)).reason).to eq(:not_author)
    nurse = nurse!(unit)
    nursing = finalized_consultation!(unit: unit, doctor: nurse, citizen: verified_citizen!(2, full_name: "Rita Souza"))
    expect(described_class.call(kind: "sick_note", content: leave, by: nurse, consultation: nursing).reason).to eq(:cbo_not_allowed)
    expect { issue(content: { "type" => "leave", "days" => 0 }) }.not_to change(ClinicalDocument, :count)
  end

  it "digital quando a assinatura está utilizável e a autora tem certificado; nasce o pedido pending" do
    signature_city!
    linked_certificate!(doctor.tap { |u| u.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) })
    document = issue.payload[:document]
    expect(document.issue_mode).to eq("digital")
    expect(SignatureRequest.find_by(document_type: "ClinicalDocument", document_id: document.id))
      .to have_attributes(status: "pending", consultation_id: consultation.id, author_user_id: doctor.id)
    expect(Signatures::SignJob).to have_been_enqueued
  end

  it "antimicrobiano sai em papel mesmo com assinatura e certificado" do
    import_test_catalog!
    signature_city!
    linked_certificate!(doctor.tap { |u| u.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) })
    content = { "items" => [ { "catalog_item" => { "id" => catalog_item(271_089).id }, "quantity" => 21, "quantity_unit" => "cápsula",
                               "route" => "oral", "dosage_instructions" => "1 cápsula de 8/8h por 7 dias" } ] }
    document = issue(kind: "prescription", content: content).payload[:document]
    expect([ document.issue_mode, document.content_data["copies"] ]).to eq([ "paper", 2 ])
    expect(SignatureRequest.where(document_id: document.id)).to be_empty
  end

  it "receita: itens gravados; uso contínuo entra na lista de medicamentos; a receita repetida não duplica (Review Focus 2)" do
    import_test_catalog!
    content = { "items" => [ { "catalog_item" => { "id" => catalog_item(268_856).id }, "quantity" => 30, "quantity_unit" => "comprimido",
                               "route" => "oral", "dosage_instructions" => "1 comprimido pela manhã", "continuous" => true },
                             { "free_text" => "Chá de camomila", "quantity" => 1, "quantity_unit" => "caixa", "route" => "oral",
                               "dosage_instructions" => "1 sachê à noite por 5 dias" } ] }
    first = issue(kind: "prescription", content: content).payload[:document]
    expect(first.prescription_items.map(&:position)).to eq([ 1, 2 ])
    medication = PatientMedication.sole
    expect(medication).to have_attributes(catmat_code: 268_856, status: "active", origin: "prescription", dosage_summary: "1 comprimido pela manhã")
    expect(medication.events.sole.clinical_document_id).to eq(first.id)

    second = issue(kind: "prescription", content: content).payload[:document]
    expect(PatientMedication.count).to eq(1)
    expect(medication.events.count).to eq(1) # mesma posologia: nada muda
    content["items"][0]["dosage_instructions"] = "1 comprimido de 12/12h"
    issue(kind: "prescription", content: content)
    expect(medication.events.order(:created_at).last).to have_attributes(kind: "changed")
    expect(second.id).not_to eq(first.id)
  end

  it "sem catálogo ativo → catalog_unavailable" do
    content = { "items" => [ { "free_text" => "Chá de camomila", "quantity" => 1, "quantity_unit" => "caixa", "route" => "oral",
                               "dosage_instructions" => "1 sachê" } ] }
    expect(issue(kind: "prescription", content: content).reason).to eq(:catalog_unavailable)
  end

  it "declaração pelo atendimento: recepção, papel, sem consulta nem CBO; atendimento antigo → 409; outro tipo recusado" do
    attendance = consultation.attendance
    document = described_class.call(kind: "attendance_declaration", content: { "issuer_registration" => "4471" },
                                    by: verifier!, attendance: attendance).payload[:document]
    expect(document).to have_attributes(issue_mode: "paper", consultation_id: nil, cbo_code: nil,
                                        patient_id: consultation.patient_id, citizen_id: attendance.citizen_id)
    expect(DomainEvent.where(name: "clinical_document.issued").last.payload).to include("attendance_id" => attendance.id)
    expect(described_class.call(kind: "sick_note", content: leave, by: verifier!, attendance: attendance).details).to eq(field: "kind")
    old_visit = walk_in_attendance!(unit, citizen: verified_citizen!(3, full_name: "Ivo Lima"), checked_in_at: 31.days.ago)
    expect(described_class.call(kind: "attendance_declaration", content: {}, by: verifier!, attendance: old_visit).reason)
      .to eq(:attendance_not_found_or_closed_long_ago)
    expect(described_class.call(kind: "attendance_declaration", content: {}, by: municipal_admin!, attendance: consultation.attendance).reason)
      .to eq(:missing_role)
  end

  it "substituição: só de um documento cancelado do mesmo tipo e da mesma autora" do
    old = issue.payload[:document]
    expect(issue(replaces_document_id: old.id).details).to eq(field: "replaces_document_id")
    old.update!(status: "cancelled", cancel_reason: "Dias errados no atestado", cancelled_at: Time.current, cancelled_by_user_id: doctor.id)
    expect(issue(replaces_document_id: old.id).payload[:document].replaces_document_id).to eq(old.id)
    expect(issue(kind: "attendance_declaration", content: {}, replaces_document_id: old.id).details).to eq(field: "replaces_document_id")
  end

  it "o JSON do documento segue o contrato §2" do
    document = issue.payload[:document]
    json = ClinicalDocuments::Json.document(document).as_json
    expect(json.keys).to match_array(%w[id kind status issue_mode issued_at cancelled_at cancel_reason author patient consultation_id
                                        attendance_id short_code verification_url replaces_document_id content signature])
    expect(json["author"]).to eq("id" => doctor.id, "name" => doctor.professional.professional_name,
                                 "council" => "#{doctor.professional.council}-#{doctor.professional.council_state} #{doctor.professional.registration_number}",
                                 "cbo_code" => "225125")
    expect(json["short_code"]).to match(/\A[2-9A-Z]{5}-[2-9A-Z]{5}\z/)
    expect(json["verification_url"]).to end_with("/v/#{document.verification_token}")
    expect(json["signature"]).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/commands/clinical_documents/issue_spec.rb`
Expected: FAIL (`uninitialized constant ClinicalDocuments::Issue`).

- [ ] **Step 3: Implemente**

Em `app/services/signatures/document_types.rb`:

```ruby
    MODELS = { "Consultation" => "consultation", "ConsultationAddendum" => "consultation_addendum",
               "ClinicalDocument" => "clinical_document" }.freeze
```

```ruby
# app/commands/clinical_documents/issue.rb
# Emissão de documento clínico (ADR 0033; spec §4–§5; contrato §3). Da
# consulta, pela autora (rascunho ou finalizada — emitir depois não muda a
# consulta); do atendimento, só a declaração de comparecimento (recepção ou
# profissional da unidade), sempre em papel. Interruptor da assinatura,
# catálogo e listas Anvisa lidos FORA da transação (como Consultations::Finalize:
# Platform::Features engole StatementInvalid, e um erro do PG engolido dentro
# do savepoint abortaria a emissão). Numa transação só: documento, itens da
# receita, eventos da lista de medicamentos, pedido de assinatura e evento.
# O modo é decidido aqui e nunca convertido (só a volta ao papel do 19b).
module ClinicalDocuments
  module Issue
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
    DECLARATION_DAYS = 30

    module_function

    def call(kind:, content:, by:, consultation: nil, attendance: nil, replaces_document_id: nil, now: Time.current)
      kind = kind.to_s
      return invalid("kind") unless ClinicalDocument::KINDS.include?(kind)
      return invalid("kind") if attendance && kind != "attendance_declaration"

      signing = consultation ? Signatures::Gate.usable?(Current.city) : false
      platform = kind == "prescription" ? prescription_platform(content) : nil
      return platform if platform.is_a?(Result)

      result = nil
      ApplicationRecord.transaction(requires_new: true) do
        result = issue(kind, content, by, consultation, attendance, replaces_document_id, signing, platform, now)
        raise ActiveRecord::Rollback if result.failure?
      end
      result
    end

    def prescription_platform(content)
      return Result.fail(:catalog_unavailable) unless Medications::Search.available?

      items = Content::Input.hash(content)["items"]
      ids = Array(items).filter_map do |item|
        reference = item.is_a?(Hash) && item["catalog_item"].is_a?(Hash) ? item["catalog_item"]["id"].to_s : nil
        reference if reference&.match?(UUID)
      end
      { catalog: MedicationCatalogItem.where(id: ids.uniq).index_by(&:id), matcher: Medications::Substances.matcher }
    end

    def issue(kind, input, by, consultation, attendance, replaces_id, signing, platform, now)
      if consultation
        return Result.fail(:not_author) unless consultation.author_user_id == by.id

        attendance = consultation.attendance
        cbo = consultation.cbo_code
        return Result.fail(:cbo_not_allowed) unless Issuers.allowed?(kind, cbo)

        patient = Patient.lock.find(consultation.patient_id)
      else
        status, cbo = Issuers.declaration_staff(by, attendance)
        return Result.fail(status) unless status == :ok
        return Result.fail(:attendance_not_found_or_closed_long_ago) if attendance.checked_in_at < now - DECLARATION_DAYS.days

        patient = attendance.citizen.patient_id && Patient.lock.find(attendance.citizen.patient_id)
      end
      replaces = replaced(replaces_id, kind, attendance, by)
      return replaces if replaces.is_a?(Result)

      built = build(kind, input, consultation, attendance, patient, cbo, platform, now)
      return built if built.failure?

      content, rows, antimicrobial = built.payload.values_at(:content, :items, :antimicrobial)
      digital = signing && !antimicrobial && SignerCertificate.active.exists?(user_id: by.id)
      document = create!(kind: kind, patient_id: patient&.id, citizen_id: attendance.citizen_id, consultation: consultation,
                         attendance: attendance, author_user_id: by.id, cbo_code: cbo, issue_mode: digital ? "digital" : "paper",
                         replaces_document_id: replaces&.id, issued_at: now, content: content.to_json)
      rows.each { |row| PrescriptionItem.create!(row.merge(clinical_document: document)) }
      failure = medications!(document, patient, rows, platform, by, now)
      return failure if failure

      Signatures::OpenRequest.call(document, now: now, usable: true) if digital
      link = consultation ? { consultation_id: consultation.id } : { attendance_id: attendance.id }
      DomainEvents.publish("clinical_document.issued", document_id: document.id, kind: kind, **link,
                                                       issue_mode: document.issue_mode)
      Result.ok(document: document)
    end

    def build(kind, input, consultation, attendance, patient, cbo, platform, now)
      today = now.in_time_zone.to_date
      result =
        case kind
        when "sick_note" then Content::SickNote.call(input, today: today)
        when "attendance_declaration" then Content::Declaration.call(input, attendance: attendance)
        when "exam_requisition" then Content::ExamRequisition.call(input, consultation: consultation)
        when "prescription"
          return Content::Prescription.call(input, role: Issuers.role(cbo), patient: patient, catalog: platform[:catalog],
                                                   matcher: platform[:matcher], today: today, city_cnpj: CityProfile.current&.cnpj)
        end
      result.ok? ? Result.ok(content: result.payload[:content], items: [], antimicrobial: false) : result
    end

    # "Cancelar e emitir outro" (spec §6): o substituído é um cancelado do
    # mesmo tipo, da mesma autora e do mesmo cidadão.
    def replaced(id, kind, attendance, by)
      return nil if id.nil?

      old = id.to_s.match?(UUID) ? ClinicalDocument.find_by(id: id) : nil
      valid = old&.cancelled? && old.kind == kind && old.author_user_id == by.id && old.citizen_id == attendance.citizen_id
      valid ? old : invalid("replaces_document_id")
    end

    # Código curto/token repetido (improvável): tenta de novo com outros.
    def create!(attrs)
      attempts = 0
      begin
        attempts += 1
        ApplicationRecord.transaction(requires_new: true) do
          ClinicalDocument.create!(attrs.merge(verification_token: Codes.token, short_code: Codes.short_code))
        end
      rescue ActiveRecord::RecordNotUnique => e
        raise unless attempts < 3 && e.message.match?(/short_code|verification_token/)

        retry
      end
    end

    def medications!(document, patient, rows, platform, by, now)
      rows.select { |row| row[:continuous] }.each do |row|
        result = Patients::ApplyMedicationEvent.call(
          patient: patient, action: "add", by: by, source: { document: document }, origin: "prescription",
          catalog_item: row[:medication_catalog_item_id] && platform[:catalog].fetch(row[:medication_catalog_item_id]),
          free_text: row[:free_text], dosage_summary: row[:dosage_instructions], continuous: true, on_active: :change,
          on: now.in_time_zone.to_date
        )
        return result if result.failure?
      end
      nil
    end

    def invalid(field) = Result.fail(:invalid_content, details: { field: field })
    private_class_method :prescription_platform, :issue, :build, :replaced, :create!, :medications!, :invalid
  end
end
```

Em `app/services/clinical_documents/json.rb`, acrescente:

```ruby
    # <document> (contrato §2). `signature` = o bloco do 19b quando há pedido
    # (documento digital, ou que voltou ao papel); nil no papel puro.
    def document(document, signature: :load)
      author = document.author_user
      professional = author.professional
      signature = document.signature_request ? Signatures::Mode.for(document) : nil if signature == :load
      { id: document.id, kind: document.kind, status: document.status, issue_mode: document.issue_mode,
        issued_at: document.issued_at.iso8601, cancelled_at: document.cancelled_at&.iso8601,
        cancel_reason: document.cancel_reason,
        author: { id: author.id, name: Screenings::Json.staff_name(author),
                  council: professional && "#{professional.council}-#{professional.council_state} #{professional.registration_number}",
                  cbo_code: document.cbo_code },
        patient: document.patient && { id: document.patient.id, display_name: document.patient.display_name },
        consultation_id: document.consultation_id, attendance_id: document.attendance_id,
        short_code: Codes.display(document.short_code), verification_url: document.verification_url,
        replaces_document_id: document.replaces_document_id, content: document.content_data, signature: signature }
    end

    # <medication> (contrato §2); `catalog` = { id => MedicationCatalogItem }.
    def medication(medication, catalog:)
      item = medication.catalog_item_id && catalog[medication.catalog_item_id]
      { id: medication.id, catalog_item: item && catalog_ref(item), free_text: medication.free_text, label: medication.label,
        dosage_summary: medication.dosage_summary, continuous: medication.continuous, status: medication.status,
        origin: medication.origin, started_on: medication.started_on&.iso8601, updated_at: medication.updated_at.iso8601 }
    end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/commands/clinical_documents/issue_spec.rb spec/services/signatures`
Expected: PASS (as specs do 19b continuam verdes com o tipo novo no catálogo).

- [ ] **Step 5: Commit**

```bash
git add app/commands/clinical_documents/issue.rb app/services/clinical_documents/json.rb app/services/signatures/document_types.rb spec/commands/clinical_documents/issue_spec.rb
git commit -m "feat: issue clinical documents with a fixed issue mode and signature request

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Rotas dos documentos e da lista de medicamentos (com a renovação)

**Files:**
- Create: `app/controllers/clinical_documents_controller.rb`, `app/controllers/patient_medications_controller.rb`, `app/services/clinical_documents/read.rb`, `app/services/clinical_documents/renewal.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/clinical_documents_spec.rb`, `spec/requests/patient_medications_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocuments::{Issue,Json}`, `Patients::ApplyMedicationEvent`, `ClinicalRecord::{Access,Trail}`, `ClinicalDocumentsGate`, `ClinicalRecordGate`, `AttendanceAccess#render_failure`.
- Produces:
  - `ClinicalDocuments::Read.consultation_grant(user:, consultation:) -> ClinicalRecord::Access::Grant` (rascunho: só a autora; finalizada: `Access.for_consultation`); `ClinicalDocuments::Read.document_grant(user:, document:) -> Grant` (autora do documento → `:author`; senão pela consulta, ou pelo paciente/atendimento na declaração da recepção).
  - `ClinicalDocuments::Renewal.items(consultation) -> Array<Hash>` (`<item>` do contrato §2, para preencher a receita).
  - Rotas (contrato §3–§4): `GET/POST /attendance/consultations/:id/documents`, `POST /attendance/attendances/:id/declarations`, `GET /attendance/documents/:id`, `GET /attendance/patients/:id/medications`, `POST /attendance/consultations/:id/medications`, `GET /attendance/consultations/:id/renewal`. (O impresso e o cancelamento entram nas Tasks 17 e 20, no mesmo controller.)

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/clinical_documents_spec.rb
require "rails_helper"

# Contrato §3: emitir e ler documentos da consulta e do atendimento. Leitura
# pelas regras do 19a (autora sempre; outro profissional em contexto ou por
# abertura), com trilha.
RSpec.describe "Documentos clínicos", type: :request do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  it "a autora emite (201 <document>), lista e lê; trilha author" do
    sign_in_as(doctor)
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 3 }
    expect(response).to have_http_status(:created)
    document = json_body
    expect(document).to include("kind" => "sick_note", "status" => "issued", "issue_mode" => "paper",
                                "consultation_id" => consultation.id, "content" => include("days" => 3))
    get "/attendance/consultations/#{consultation.id}/documents"
    expect(json_body["items"].map { |item| item["id"] }).to eq([ document["id"] ])
    get "/attendance/documents/#{document['id']}"
    expect(json_body["id"]).to eq(document["id"])
    expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "author")
  end

  it "erros do contrato: 403 not_author/cbo_not_allowed, 422 com field/index, 503 sem catálogo, 403 interruptor" do
    sign_in_as(doctor!(unit))
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 3 }
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "not_author" } ])
    sign_in_as(doctor)
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 0 }
    expect([ response.status, json_body ]).to eq([ 422, { "error" => "invalid_content", "field" => "days" } ])
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "prescription",
              content: { items: [ { free_text: "Chá", quantity: 1, quantity_unit: "caixa", route: "oral", dosage_instructions: "1 sachê" } ] }
    expect([ response.status, json_body ]).to eq([ 503, { "error" => "catalog_unavailable" } ])
    import_test_catalog!
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "prescription",
              content: { items: [ { catalog_item: { id: catalog_item(267_197).id }, quantity: 10, quantity_unit: "comprimido",
                                    route: "oral", dosage_instructions: "1 à noite" } ] }
    expect([ response.status, json_body ]).to eq([ 422, { "error" => "controlled_not_allowed", "index" => 0 } ])
    documents_city!(enabled: false)
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 3 }
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "clinical_documents")
    get "/attendance/consultations/#{consultation.id}/documents"
    expect(response).to have_http_status(:ok) # leitura segue com o interruptor desligado
  end

  it "outro profissional: fora de contexto 403; em contexto lê com trilha in_context; recepção não lê documento alheio" do
    sign_in_as(doctor)
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 3 }
    id = json_body["id"]
    nurse = nurse!(unit)
    sign_in_as(nurse)
    get "/attendance/documents/#{id}"
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "out_of_context" } ])
    walk_in_attendance!(unit, citizen: consultation.attendance.citizen)
    get "/attendance/documents/#{id}"
    expect(response).to have_http_status(:ok)
    expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "in_context", "user_id" => nurse.id)
    sign_in_as(verifier!)
    get "/attendance/documents/#{id}"
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "missing_role" } ])
  end

  it "declaração pela recepção: 201 em papel; ela lê a própria; atendimento inexistente 404" do
    reception = verifier!
    sign_in_as(reception)
    json_post "/attendance/attendances/#{consultation.attendance_id}/declarations", content: { issuer_registration: "4471" }
    expect(response).to have_http_status(:created)
    expect(json_body).to include("kind" => "attendance_declaration", "issue_mode" => "paper", "consultation_id" => nil,
                                 "author" => include("id" => reception.id, "cbo_code" => nil))
    get "/attendance/documents/#{json_body['id']}"
    expect(response).to have_http_status(:ok)
    json_post "/attendance/attendances/#{SecureRandom.uuid}/declarations", content: {}
    expect([ response.status, json_body ]).to eq([ 404, { "error" => "not_found" } ])
    sign_in_as(municipal_admin!)
    json_post "/attendance/attendances/#{consultation.attendance_id}/declarations", content: {}
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "missing_role" } ])
  end
end
```

```ruby
# spec/requests/patient_medications_spec.rb
require "rails_helper"

# Contrato §4: medicamentos em uso (leitura como o prontuário; escrita da
# autora na consulta em rascunho) e a renovação dos contínuos.
RSpec.describe "Medicamentos em uso", type: :request do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release!; import_test_catalog! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }

  it "na consulta em rascunho: informar de fora, suspender, reativar; erros 409/422; lista do paciente" do
    draft = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    sign_in_as(doctor)
    json_post "/attendance/consultations/#{draft.id}/medications", action: "add_external",
              catalog_item_id: catalog_item(267_772).id, dosage_summary: "1 cp 12/12h"
    expect(response).to have_http_status(:ok)
    medication = json_body
    expect(medication).to include("label" => "PROPRANOLOL CLORIDRATO 40 MG", "status" => "active", "origin" => "external",
                                  "catalog_item" => include("catmat_code" => 267_772))
    json_post "/attendance/consultations/#{draft.id}/medications", action: "add_external", catalog_item_id: catalog_item(267_772).id
    expect([ response.status, json_body ]).to eq([ 409, { "error" => "already_active" } ])
    json_post "/attendance/consultations/#{draft.id}/medications", action: "suspend", medication_id: medication["id"]
    expect(json_body["status"]).to eq("suspended")
    json_post "/attendance/consultations/#{draft.id}/medications", action: "reactivate", medication_id: medication["id"]
    expect(json_body["status"]).to eq("active")
    json_post "/attendance/consultations/#{draft.id}/medications", action: "suspend", medication_id: SecureRandom.uuid
    expect([ response.status, json_body ]).to eq([ 422, { "error" => "invalid_medication" } ])
    json_post "/attendance/consultations/#{draft.id}/medications", action: "add_external", free_text: "x"
    expect(json_body).to eq("error" => "invalid_content", "field" => "free_text")

    get "/attendance/patients/#{draft.patient_id}/medications"
    expect(json_body["items"].map { |item| item["id"] }).to eq([ medication["id"] ])
  end

  it "consulta finalizada não muda a lista por esta rota (409 not_draft); outro profissional 403" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    sign_in_as(doctor)
    json_post "/attendance/consultations/#{consultation.id}/medications", action: "add_external", free_text: "Chá de boldo"
    expect([ response.status, json_body ]).to eq([ 409, { "error" => "not_draft" } ])
    sign_in_as(doctor!(unit))
    get "/attendance/patients/#{consultation.patient_id}/medications"
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "out_of_context" } ])
  end

  it "renovação: os contínuos ativos, com a última posologia da receita" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    content = { "items" => [ { "catalog_item" => { "id" => catalog_item(268_856).id }, "quantity" => 30, "quantity_unit" => "comprimido",
                               "route" => "oral", "dosage_instructions" => "1 comprimido pela manhã", "duration_days" => 30,
                               "continuous" => true } ] }
    ClinicalDocuments::Issue.call(kind: "prescription", content: content, by: doctor, consultation: consultation)
    sign_in_as(doctor)
    get "/attendance/consultations/#{consultation.id}/renewal"
    expect(json_body["items"]).to eq([ { "position" => 1, "catalog_item" => ClinicalDocuments::Json.catalog_ref(catalog_item(268_856)).as_json,
                                         "printed_description" => "LOSARTANA POTÁSSICA 50 MG", "quantity" => 30,
                                         "quantity_unit" => "comprimido", "route" => "oral",
                                         "dosage_instructions" => "1 comprimido pela manhã", "duration_days" => 30,
                                         "continuous" => true, "antimicrobial" => false } ])
    sign_in_as(doctor!(unit))
    get "/attendance/consultations/#{consultation.id}/renewal"
    expect(json_body).to eq("error" => "not_author")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/clinical_documents_spec.rb spec/requests/patient_medications_spec.rb`
Expected: FAIL (rotas inexistentes).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/read.rb
# Quem lê um documento clínico (ADR 0033; contrato §3, preâmbulo): a autora do
# documento sempre (inclusive a recepção, a própria declaração); os demais
# pelas regras do 19a — pela consulta (rascunho só da autora; finalizada por
# ClinicalRecord::Access.for_consultation) ou, na declaração do atendimento,
# pelo paciente e pelo atendimento (ClinicalRecord::Access.call).
module ClinicalDocuments
  module Read
    Grant = ClinicalRecord::Access::Grant

    module_function

    def consultation_grant(user:, consultation:)
      return deny(:missing_role) unless user.has_role?("health_professional")
      return consultation.author_user_id == user.id ? Grant.new(kind: :in_context, opening: nil, reason: nil) : deny(:not_author) if consultation.draft?

      ClinicalRecord::Access.for_consultation(user: user, consultation: consultation)
    end

    def document_grant(user:, document:)
      staff = user.has_role?("health_professional") || user.has_role?("citizen_verifier")
      return Grant.new(kind: :author, opening: nil, reason: nil) if staff && document.author_user_id == user.id
      return consultation_grant(user: user, consultation: document.consultation) if document.consultation

      ClinicalRecord::Access.call(user: user, patient: document.patient, attendance: document.attendance)
    end

    def deny(reason) = Grant.new(kind: :denied, opening: nil, reason: reason)
    private_class_method :deny
  end
end
```

```ruby
# app/services/clinical_documents/renewal.rb
# "Renovar receita" (spec §5): os contínuos ATIVOS do paciente, na forma do
# <item> (contrato §2), com a última posologia receitada do mesmo item do
# catálogo (o informado de fora vem só com o resumo). É só um rascunho para
# revisão: a emissão valida tudo de novo.
module ClinicalDocuments
  module Renewal
    module_function

    def items(consultation)
      medications = PatientMedication.where(patient_id: consultation.patient_id, status: "active", continuous: true)
                                     .order(:created_at, :id).to_a
      catalog = MedicationCatalogItem.where(id: medications.filter_map(&:catalog_item_id)).index_by(&:id)
      last = PrescriptionItem.joins(:clinical_document)
                             .where(clinical_documents: { patient_id: consultation.patient_id, status: "issued" },
                                    medication_catalog_item_id: catalog.keys)
                             .order(created_at: :desc, id: :desc).to_a.uniq(&:medication_catalog_item_id)
                             .index_by(&:medication_catalog_item_id)
      medications.each_with_index.map do |medication, index|
        item = catalog[medication.catalog_item_id]
        prior = medication.catalog_item_id && last[medication.catalog_item_id]
        { position: index + 1, catalog_item: item && Json.catalog_ref(item), free_text: medication.free_text,
          printed_description: medication.label, quantity: prior && Json.number(prior.quantity), quantity_unit: prior&.quantity_unit,
          route: prior&.route, dosage_instructions: prior&.dosage_instructions || medication.dosage_summary,
          duration_days: prior&.duration_days, continuous: true, antimicrobial: item&.antimicrobial || false,
          reason_problem_id: prior&.reason_problem_id }.compact
      end
    end
  end
end
```

```ruby
# app/controllers/clinical_documents_controller.rb
# Documentos clínicos (ADR 0033; contrato §3). Emitir: da consulta (autora) ou
# do atendimento (declaração); interruptor clinical_documents. Ler: autora
# sempre, demais pelas regras do 19a, com trilha; segue com o interruptor
# desligado (atrás só do clinical_record). Impresso (Task 17) e cancelamento
# (Task 20) moram aqui.
class ClinicalDocumentsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate
  include ClinicalDocumentsGate
  include MfaStepUp

  UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
  ERROR_STATUS = {
    not_author: :forbidden, cbo_not_allowed: :forbidden, missing_role: :forbidden, out_of_context: :forbidden,
    feature_disabled: :forbidden, not_found: :not_found, already_cancelled: :conflict, awaiting_signature: :conflict,
    attendance_not_found_or_closed_long_ago: :conflict, catalog_unavailable: :service_unavailable
  }.freeze

  wrap_parameters false
  before_action :require_clinical_record!
  before_action :require_clinical_documents!, only: %i[create declaration]
  before_action :require_staff

  def index
    consultation = find(Consultation, params[:id])
    return not_found unless consultation

    grant = ClinicalDocuments::Read.consultation_grant(user: Current.user, consultation: consultation)
    return forbid(grant.reason.to_s) unless grant.allowed?

    ClinicalRecord::Trail.viewed!(patient: consultation.patient, user: Current.user, grant: grant)
    documents = ClinicalDocument.where(consultation_id: consultation.id).order(issued_at: :desc, id: :desc)
    render json: { items: documents.map { |document| ClinicalDocuments::Json.document(document) } }
  end

  def create
    consultation = find(Consultation, params[:id])
    return not_found unless consultation

    result = ClinicalDocuments::Issue.call(kind: body["kind"], content: body["content"], by: Current.user,
                                           consultation: consultation, replaces_document_id: body["replaces_document_id"])
    respond_created(result)
  end

  def declaration
    attendance = find(Attendance, params[:id])
    return not_found unless attendance

    result = ClinicalDocuments::Issue.call(kind: "attendance_declaration", content: body["content"], by: Current.user,
                                           attendance: attendance)
    respond_created(result)
  end

  def show
    document = readable_document
    render json: ClinicalDocuments::Json.document(document) if document
  end

  private

  def require_staff
    policy = CitizenVerificationPolicy.new(Current.user, nil)
    allowed = %w[declaration show print].include?(action_name) ? policy.care? || policy.verify? : policy.care?
    forbid("missing_role") unless allowed
  end

  # O documento com a trilha da leitura, ou nil (a resposta de erro já saiu).
  def readable_document
    document = find(ClinicalDocument, params[:id])
    return not_found && nil unless document

    grant = ClinicalDocuments::Read.document_grant(user: Current.user, document: document)
    return forbid(grant.reason.to_s) && nil unless grant.allowed?

    ClinicalRecord::Trail.viewed!(patient: document.patient, user: Current.user, grant: grant)
    document
  end

  def respond_created(result)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: ClinicalDocuments::Json.document(result.payload[:document]), status: :created
  end

  def find(model, id) = id.to_s.match?(UUID) ? model.find_by(id: id) : nil
  def body = params.to_unsafe_h.except("controller", "action", "id")
  def not_found = render(json: { error: "not_found" }, status: :not_found)
end
```

```ruby
# app/controllers/patient_medications_controller.rb
# Medicamentos em uso (ADR 0033; contrato §4). Leitura: regra do prontuário
# (contexto ou abertura), com trilha. Escrita: a autora, na consulta em
# rascunho, pelo caminho único (Patients::ApplyMedicationEvent), com o
# paciente travado; catálogo lido FORA da transação. Renovação: a autora.
class PatientMedicationsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate
  include ClinicalDocumentsGate

  UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
  ERROR_STATUS = { not_author: :forbidden, not_draft: :conflict, already_active: :conflict }.freeze

  wrap_parameters false
  before_action :require_clinical_record!
  before_action :require_clinical_documents!, only: :create
  before_action :require_professional

  def index
    patient = uuid?(params[:id]) ? Patient.find_by(id: params[:id]) : nil
    return not_found unless patient

    grant = ClinicalRecord::Access.call(user: Current.user, patient: patient)
    return forbid(grant.reason.to_s) unless grant.allowed?

    ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant)
    medications = PatientMedication.where(patient_id: patient.id).order(:status, updated_at: :desc, id: :desc).to_a
    catalog = MedicationCatalogItem.where(id: medications.filter_map(&:catalog_item_id)).index_by(&:id)
    render json: { items: medications.map { |medication| ClinicalDocuments::Json.medication(medication, catalog: catalog) } }
  end

  # O corpo do contrato usa a chave `action`, que no `params` é o nome da ação
  # do Rails (o roteamento vence): o corpo é lido de request.request_parameters.
  def create
    consultation = uuid?(params[:id]) ? Consultation.find_by(id: params[:id]) : nil
    return not_found unless consultation
    return forbid("not_author") unless consultation.author_user_id == Current.user.id
    return render(json: { error: "not_draft" }, status: :conflict) unless consultation.draft?

    input = request.request_parameters
    catalog_item = nil
    if input["catalog_item_id"].present?
      catalog_item = uuid?(input["catalog_item_id"]) ? MedicationCatalogItem.where(status: "active").find_by(id: input["catalog_item_id"]) : nil
      return render(json: { error: "invalid_content", field: "catalog_item_id" }, status: :unprocessable_entity) unless catalog_item
    end
    result = apply(consultation, input["action"], catalog_item, input)
    return render_failure(result, ERROR_STATUS) if result.failure?

    medication = result.payload[:medication]
    catalog = MedicationCatalogItem.where(id: medication.catalog_item_id).index_by(&:id)
    render json: ClinicalDocuments::Json.medication(medication.reload, catalog: catalog)
  end

  def renewal
    consultation = uuid?(params[:id]) ? Consultation.find_by(id: params[:id]) : nil
    return not_found unless consultation
    return forbid("not_author") unless consultation.author_user_id == Current.user.id

    render json: { items: ClinicalDocuments::Renewal.items(consultation) }
  end

  private

  def apply(consultation, action, catalog_item, input)
    ApplicationRecord.transaction do
      patient = Patient.lock.find(consultation.patient_id)
      common = { patient: patient, by: Current.user, source: { consultation: consultation } }
      case action
      when "add_external"
        free_text = input["free_text"]
        if catalog_item.nil? && !(free_text.is_a?(String) && free_text.squish.size.between?(3, 200))
          next Result.fail(:invalid_content, details: { field: "free_text" })
        end
        summary = input["dosage_summary"]
        next Result.fail(:invalid_content, details: { field: "dosage_summary" }) unless summary.nil? || (summary.is_a?(String) && summary.size <= 200)

        Patients::ApplyMedicationEvent.call(**common, action: "add", origin: "external", catalog_item: catalog_item,
                                            free_text: catalog_item ? nil : free_text, dosage_summary: summary,
                                            continuous: input.fetch("continuous", true) == true, on_active: :fail)
      when "suspend", "reactivate"
        medication = uuid?(input["medication_id"]) ? PatientMedication.find_by(id: input["medication_id"], patient_id: patient.id) : nil
        next Result.fail(:invalid_medication) unless medication

        Patients::ApplyMedicationEvent.call(**common, action: action, medication: medication)
      else
        Result.fail(:invalid_content, details: { field: "action" })
      end
    end
  end

  def uuid?(value) = value.to_s.match?(UUID)
  def not_found = render(json: { error: "not_found" }, status: :not_found)
end
```

Em `config/routes.rb`, no `scope "/attendance"`, depois de `consultations/:id/print`:

```ruby
    # Documentos clínicos e medicamentos em uso (ADR 0033; contrato §3–§4).
    get  "consultations/:id/documents",    to: "clinical_documents#index"
    post "consultations/:id/documents",    to: "clinical_documents#create"
    post "attendances/:id/declarations",   to: "clinical_documents#declaration"
    get  "documents/:id",                  to: "clinical_documents#show"
    get  "patients/:id/medications",       to: "patient_medications#index"
    post "consultations/:id/medications",  to: "patient_medications#create"
    get  "consultations/:id/renewal",      to: "patient_medications#renewal"
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/clinical_documents_spec.rb spec/requests/patient_medications_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/controllers/clinical_documents_controller.rb app/controllers/patient_medications_controller.rb app/services/clinical_documents/read.rb app/services/clinical_documents/renewal.rb config/routes.rb spec/requests/clinical_documents_spec.rb spec/requests/patient_medications_spec.rb
git commit -m "feat: add the clinical document and patient medication routes with renewal

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: PDF por tipo (Prawn) com QR code, e o impresso

**Escolha da gem de QR:** `rqrcode_core` (~> 2.1; 2.1.0 publicada). É o núcleo do `rqrcode` (a gem de QR mais usada em Ruby), em **Ruby puro e sem nenhuma dependência de execução** — só gera a matriz de módulos. O api desenha cada módulo como um quadrado vetorial no Prawn (`fill_rectangle`): nítido em qualquer impressora, sem PNG, sem `chunky_png` e sem binário nativo (o mesmo critério que levou ao Prawn no 19a: "Ruby puro, sem binário"). Descartadas: `rqrcode` (traz `chunky_png` para PNG/SVG que não usamos), `prawn-qrcode` (pouco mantida; arrasta `rqrcode` + `chunky_png`), gerar QR à mão (erro de correção Reed-Solomon não é coisa para reescrever).

**Files:**
- Modify: `Gemfile`, `Gemfile.lock`, `app/controllers/clinical_documents_controller.rb`, `config/routes.rb`
- Create: `app/services/clinical_documents/qr.rb`, `app/services/clinical_documents/pdf.rb`
- Test: `spec/services/clinical_documents/pdf_spec.rb`, `spec/requests/clinical_document_print_spec.rb`

**Interfaces:**
- Consumes: `Consultations::Print.{safe,stamp_footer}` (19a/19b), `Signatures::PdfFooter`, `ClinicalDocuments::Codes.display`, `Cnpj.display`, `Professionals::Cbo.find`.
- Produces:
  - `ClinicalDocuments::Qr.matrix(text) -> Array<Array<Boolean>>`; `ClinicalDocuments::Qr.draw(pdf, text, at: [x, y], size:)`.
  - `ClinicalDocuments::Pdf.call(document, footer: nil) -> String` (bytes do PDF); levanta `ClinicalDocuments::Pdf::NotPrintable` em documento cancelado. `footer` (um `Signatures::PdfFooter`) = PDF que vai ao PAdES (rodapé NGS2 em toda página, sem espaço de assinatura à mão).
  - `GET /attendance/documents/:id/print` → `application/pdf` (`Cache-Control: no-store`, `documento.pdf`): digital assinado → o PAdES gravado; digital sem assinatura → 409 `awaiting_signature`; papel → o PDF do papel; cancelado → 409 `already_cancelled`.

- [ ] **Step 1: A gem**

Run: `docker compose exec -T -w /rails/.claude/mod19c api bundle add rqrcode_core --version "~> 2.1"`
Expected: `Gemfile` ganha `gem "rqrcode_core", "~> 2.1"` e o `Gemfile.lock` a 2.1.x. Acrescente o comentário na linha do `Gemfile`: `# ADR 0033: QR code do documento clínico (Ruby puro, sem dependências; desenhado como vetor no Prawn)`. Confira a API: `docker compose exec -T -w /rails/.claude/mod19c api bin/rails runner 'q = RQRCodeCore::QRCode.new("https://x.test/v/abc", level: :m); puts q.modules.size, q.modules.first.first.inspect'` → um número (ex.: `25`) e `true`.

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/services/clinical_documents/pdf_spec.rb
require "rails_helper"
require "pdf/reader"

# Spec §6: PDF por tipo — cabeçalho (unidade, CNES, cidade), corpo do tipo,
# profissional (conselho, CBO), QR code e código curto em toda via; papel com
# espaço para assinatura; antimicrobiano em 2 vias; digital com o rodapé NGS2.
RSpec.describe ClinicalDocuments::Pdf do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release!; city_profile! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def text_of(bytes) = PDF::Reader.new(StringIO.new(bytes)).pages.map(&:text)
  def issue!(kind, content, by: doctor) = ClinicalDocuments::Issue.call(kind: kind, content: content, by: by, consultation: consultation).payload.fetch(:document)

  it "atestado: título, paciente, dias, CID com a autorização, profissional, código e endereço de conferência" do
    document = issue!("sick_note", { "type" => "leave", "days" => 3, "cid10" => { "code" => "I10" }, "cid_authorized" => true })
    pages = text_of(described_class.call(document))
    text = pages.join("\n")
    expect(text).to include("ATESTADO", "Maria Aparecida da Silva", "3 dia(s)", "I10", "autorização expressa do paciente",
                            ClinicalDocuments::Codes.display(document.short_code), "/v/#{document.verification_token}",
                            "Assinatura e carimbo", doctor.professional.professional_name, "CBO 225125")
  end

  it "receita com antimicrobiano: duas vias, cada uma com o código; validade de 10 dias" do
    import_test_catalog!
    document = issue!("prescription", { "items" => [ { "catalog_item" => { "id" => catalog_item(271_089).id }, "quantity" => 21,
                                                       "quantity_unit" => "cápsula", "route" => "oral",
                                                       "dosage_instructions" => "1 cápsula de 8/8h por 7 dias" } ] })
    pages = text_of(described_class.call(document))
    code = ClinicalDocuments::Codes.display(document.short_code)
    expect(pages.size).to eq(2)
    expect(pages[0]).to include("1ª via — farmácia", code, "AMOXICILINA 500 MG", "21 cápsula", "Validade")
    expect(pages[1]).to include("2ª via — paciente", code)
  end

  it "receita cheia (Review Focus 5): 20 itens, emoji, ≥, \\r\\n e posologia longa; o código em toda via" do
    import_test_catalog!
    items = Array.new(20) do |i|
      { "free_text" => "Chá de camomila #{i} 🌼", "quantity" => 1, "quantity_unit" => "caixa", "route" => "oral",
        "dosage_instructions" => "Tomar ≥ 1 sachê\r\nà noite. #{'x' * 440}" }
    end
    document = issue!("prescription", { "items" => items })
    pages = text_of(described_class.call(document))
    expect(pages.size).to be >= 2
    expect(pages.join).to include("Chá de camomila 19 ?", ClinicalDocuments::Codes.display(document.short_code))
  end

  it "declaração da recepção: matrícula de quem emite; enfermagem: protocolo, CNPJ e Coren" do
    declaration = ClinicalDocuments::Issue.call(kind: "attendance_declaration", content: { "issuer_registration" => "4471" },
                                                by: verifier!, attendance: consultation.attendance).payload[:document]
    expect(text_of(described_class.call(declaration)).join).to include("DECLARAÇÃO DE COMPARECIMENTO", "Matrícula: 4471")

    import_test_catalog!
    city_cnpj!
    nurse = nurse!(unit)
    params = { "title" => "Saúde da mulher", "number" => "PE-01", "year" => 2026, "valid_from" => (Time.zone.today - 1).iso8601,
               "items" => [ { "catalog_item_id" => catalog_item(267_503).id } ] }
    version = NursingProtocols::Save.create(params: params, by: municipal_admin!, catalog: { catalog_item(267_503).id => catalog_item(267_503) })
                                    .payload[:protocol].current_version
    nursing = finalized_consultation!(unit: unit, doctor: nurse, citizen: verified_citizen!(2, full_name: "Rita Souza"))
    document = ClinicalDocuments::Issue.call(kind: "prescription", by: nurse, consultation: nursing,
                                             content: { "nursing_protocol" => { "version_id" => version.id },
                                                        "items" => [ { "catalog_item" => { "id" => catalog_item(267_503).id }, "quantity" => 30,
                                                                       "quantity_unit" => "comprimido", "route" => "oral",
                                                                       "dosage_instructions" => "1 comprimido ao dia" } ] }).payload[:document]
    text = text_of(described_class.call(document)).join
    expect(text).to include("Saúde da mulher", "PE-01/2026", "76.001.234/0001-15", "COREN")
  end

  it "requisição de exames lista os exames; digital leva o rodapé NGS2 em toda página e não o espaço à mão" do
    document = issue!("exam_requisition", {})
    footer = Signatures::PdfFooter.new(signer_name: "MÉDICA DE TESTE", signer_cpf: "52998224725", signed_at: Time.current, simulated: true)
    pages = text_of(described_class.call(document, footer: footer))
    expect(pages.join).to include("REQUISIÇÃO DE EXAMES", "0202010503", "DOSAGEM DE HEMOGLOBINA GLICOSILADA")
    expect(pages).to all(include("Documento assinado digitalmente por"))
    expect(pages.join).not_to include("Assinatura e carimbo")
  end

  it "cancelado não imprime" do
    document = issue!("sick_note", { "type" => "leave", "days" => 1 })
    document.update!(status: "cancelled", cancel_reason: "Emitido por engano aqui", cancelled_at: Time.current, cancelled_by_user_id: doctor.id)
    expect { described_class.call(document) }.to raise_error(described_class::NotPrintable)
  end

  it "QR: um quadrado por módulo escuro, dentro da área pedida" do
    url = "https://curitiba.rotasaude.app/v/#{ClinicalDocuments::Codes.token}"
    modules = ClinicalDocuments::Qr.matrix(url)
    pdf = instance_double(Prawn::Document, fill_color: nil)
    calls = []
    allow(pdf).to receive(:fill_rectangle) { |point, w, h| calls << [ point, w, h ] }
    ClinicalDocuments::Qr.draw(pdf, url, at: [ 100, 200 ], size: 84)
    expect(calls.size).to eq(modules.flatten.count(true))
    expect(calls.map { |(x, _y), _w, _h| x }.min).to be >= 100
    expect(calls.map { |(x, _y), w, _h| x + w }.max).to be <= 184.0001
  end
end
```

```ruby
# spec/requests/clinical_document_print_spec.rb
require "rails_helper"

# Contrato §3 (Desvio 7): impresso do documento — papel na hora; digital
# assinado = o PAdES; digital sem assinatura → 409 awaiting_signature;
# cancelado → 409. Sem cache; nome do arquivo sem dado da pessoa; trilha.
RSpec.describe "Impresso do documento clínico", type: :request do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def issue! = ClinicalDocuments::Issue.call(kind: "sick_note", content: { "type" => "leave", "days" => 1 }, by: doctor, consultation: consultation).payload[:document]

  it "papel: PDF, no-store, nome neutro, trilha; cancelado 409" do
    document = issue!
    sign_in_as(doctor)
    get "/attendance/documents/#{document.id}/print"
    expect(response.media_type).to eq("application/pdf")
    expect(response.headers["Cache-Control"]).to include("no-store")
    expect(response.headers["Content-Disposition"]).to include("documento.pdf")
    expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "author")
    document.update!(status: "cancelled", cancel_reason: "Emitido por engano aqui", cancelled_at: Time.current, cancelled_by_user_id: doctor.id)
    get "/attendance/documents/#{document.id}/print"
    expect([ response.status, JSON.parse(response.body) ]).to eq([ 409, { "error" => "already_cancelled" } ])
  end

  it "digital pendente → 409 awaiting_signature; assinado → o PDF assinado" do
    stub_psc!
    stub_signer!
    signature_city!
    doctor.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF)
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    document = issue!
    sign_in_as(doctor)
    get "/attendance/documents/#{document.id}/print"
    expect([ response.status, JSON.parse(response.body) ]).to eq([ 409, { "error" => "awaiting_signature" } ])
    signature = sign_document!(document, author: doctor)
    get "/attendance/documents/#{document.id}/print"
    expect(response.body.b).to eq(signature.signed_pdf_bytes.b)
  end
end
```

> O segundo exemplo depende da integração com o 19b (Task 18): rode-o ao fim da Task 18; nesta task, o primeiro.

- [ ] **Step 3: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/pdf_spec.rb spec/requests/clinical_document_print_spec.rb:20`
Expected: FAIL (`uninitialized constant ClinicalDocuments::Pdf`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/clinical_documents/qr.rb
# QR code do endereço de conferência (ADR 0033; spec §6), desenhado como
# vetor: um quadrado por módulo escuro, com a zona de silêncio de 4 módulos.
# Nível M de correção (15%): sobrevive a dobra e carimbo no papel.
require "rqrcode_core"

module ClinicalDocuments
  module Qr
    QUIET = 4

    module_function

    def matrix(text) = RQRCodeCore::QRCode.new(text, level: :m).modules

    def draw(pdf, text, at:, size:)
      modules = matrix(text)
      cell = size.to_f / (modules.size + (2 * QUIET))
      x0, y0 = at
      pdf.fill_color "000000"
      modules.each_with_index do |row, r|
        row.each_with_index do |dark, c|
          pdf.fill_rectangle([ x0 + ((c + QUIET) * cell), y0 - ((r + QUIET) * cell) ], cell, cell) if dark
        end
      end
    end
  end
end
```

```ruby
# app/services/clinical_documents/pdf.rb
# PDF do documento clínico (ADR 0033; spec §6), gerado na hora e nunca
# gravado (o assinado mora em signatures). Cabeçalho (cidade, unidade, CNES,
# endereço), corpo do tipo, profissional (conselho, CBO), e em TODA via o QR
# code do endereço de conferência e o código curto. Papel: espaço para
# assinatura e carimbo. Digital (footer): rodapé NGS2 em toda página (e o
# aviso de simulada), sem espaço à mão. Antimicrobiano: 1ª via farmácia, 2ª
# via paciente. A fonte embutida (WinAnsi) não tem emoji nem "≥":
# Consultations::Print.safe troca o que falta por "?".
require "prawn"

module ClinicalDocuments
  module Pdf
    class NotPrintable < StandardError; end

    TITLES = { "sick_note" => "ATESTADO", "attendance_declaration" => "DECLARAÇÃO DE COMPARECIMENTO",
               "prescription" => "RECEITUÁRIO", "exam_requisition" => "REQUISIÇÃO DE EXAMES" }.freeze
    ANTIMICROBIAL_TITLE = "RECEITUÁRIO DE ANTIMICROBIANO — RETENÇÃO DA 2ª VIA".freeze
    COPIES = [ "1ª via — farmácia", "2ª via — paciente" ].freeze
    ROUTES = { "oral" => "via oral", "sublingual" => "via sublingual", "topical" => "uso tópico", "ophthalmic" => "uso oftálmico",
               "otic" => "uso otológico", "nasal" => "uso nasal", "inhalation" => "via inalatória", "vaginal" => "via vaginal",
               "rectal" => "via retal", "intramuscular" => "via intramuscular", "intravenous" => "via intravenosa",
               "subcutaneous" => "via subcutânea", "other" => "outra via" }.freeze
    COMPANION_REASONS = { "clt_473_x" => "acompanhar consulta ou exame de gestante (CLT, art. 473, X)",
                          "clt_473_xi" => "acompanhar filho de até 6 anos em consulta (CLT, art. 473, XI)",
                          "clt_473_xii" => "realizar exames preventivos de câncer (CLT, art. 473, XII)",
                          "other" => "acompanhar o paciente" }.freeze
    PERIODS = { "morning" => "no período da manhã", "afternoon" => "no período da tarde", "full_day" => "em período integral" }.freeze
    QR_SIZE = 84

    module_function

    def call(document, footer: nil)
      raise NotPrintable, "cancelled" if document.cancelled?

      content = document.content_data
      labels = document.prescription? && content["copies"] == 2 ? COPIES : [ nil ]
      pdf = Prawn::Document.new(page_size: "A4", margin: footer ? [ 40, 40, 80, 40 ] : 40,
                                info: { Title: "Documento clínico", Producer: "Rota Saúde" })
      pdf.font_size(10)
      Consultations::Print.stamp_footer(pdf, footer) if footer
      labels.each_with_index do |label, index|
        pdf.start_new_page if index.positive?
        copy(pdf, document, content, label, footer)
      end
      pdf.render
    end

    def copy(pdf, document, content, label, footer)
      header(pdf, document, content, label)
      person(pdf, document)
      send(:"#{document.kind}_body", pdf, document, content)
      professional(pdf, document, content)
      hand_signature(pdf) unless footer
      verification(pdf, document)
    end

    def s(text) = Consultations::Print.safe(text)
    def date(iso) = Date.iso8601(iso).strftime("%d/%m/%Y")

    def header(pdf, document, content, label)
      unit = document.attendance.health_unit
      pdf.text s(CityProfile.current&.name.to_s), style: :bold, size: 12
      pdf.text s("#{unit.name}#{unit.cnes ? " — CNES #{unit.cnes}" : ''}")
      address = [ unit.address_street, unit.address_number, unit.address_complement ].compact_blank.join(", ")
      pdf.text s("#{address}#{unit.address_zip ? " — CEP #{unit.address_zip}" : ''}") if address.present?
      pdf.move_down 8
      title = document.prescription? && content["antimicrobial"] ? ANTIMICROBIAL_TITLE : TITLES.fetch(document.kind)
      pdf.text s(title), style: :bold, size: 13
      pdf.text s(label), style: :bold if label
      pdf.text s("Emitido em #{document.issued_at.in_time_zone.strftime('%d/%m/%Y %H:%M')}"), size: 9
    end

    def person(pdf, document)
      person = document.patient || document.citizen
      pdf.move_down 8
      pdf.text s("Paciente: #{person.display_name}"), style: :bold
      birth = person.birth_date.present? ? date(person.birth_date) : "não informado"
      # O impresso é do paciente: CPF completo, como o impresso da consulta do 19a.
      pdf.text s("CPF: #{person.cpf.to_s.sub(/\A(\d{3})(\d{3})(\d{3})(\d{2})\z/, '\1.\2.\3-\4')} — Nascimento: #{birth}")
      pdf.text s("Endereço: ________________________________________________") if document.prescription?
    end

    def sick_note_body(pdf, document, content)
      pdf.move_down 10
      if content["type"] == "leave"
        pdf.text s("Atesto, para os devidos fins, que o(a) paciente acima foi atendido(a) nesta unidade e necessita de " \
                   "afastamento de suas atividades por #{content['days']} dia(s), a partir de #{date(content['start_on'])}.")
        if content["cid10"]
          pdf.move_down 6
          pdf.text s("CID-10: #{content.dig('cid10', 'code')} — #{content.dig('cid10', 'label')} " \
                     "(informado com autorização expressa do paciente).")
        end
      else
        reason = COMPANION_REASONS.fetch(content["companion_reason"])
        pdf.text s("Declaro, para os devidos fins, que #{content['companion_name']} (#{content['companion_kinship']}) " \
                   "esteve nesta unidade em #{document.issued_at.in_time_zone.strftime('%d/%m/%Y')} para #{reason}.")
      end
      note(pdf, content)
    end

    def attendance_declaration_body(pdf, _document, content)
      when_text = content["period"] ? PERIODS.fetch(content["period"]) : "das #{content['arrived_at']}#{content['left_at'] ? " às #{content['left_at']}" : ''}"
      pdf.move_down 10
      pdf.text s("Declaro, para os devidos fins, que o(a) paciente acima esteve nesta unidade de saúde (#{content['unit_name']}) " \
                 "em #{date(content['date'])}, #{when_text}" \
                 "#{content['companion_name'] ? ", acompanhado(a) de #{content['companion_name']}" : ''}.")
    end

    def prescription_body(pdf, _document, content)
      if (protocol = content["nursing_protocol"])
        pdf.move_down 6
        pdf.text s("Prescrição de enfermagem conforme o protocolo \"#{protocol['title']}\" nº #{protocol['number']}/#{protocol['year']}."), size: 9
        pdf.text s("Instituição: #{CityProfile.current&.name} — CNPJ #{Cnpj.display(content['city_cnpj'])}"), size: 9
      end
      pdf.move_down 8
      content["items"].each do |item|
        pdf.text s("#{item['position']}. #{item['printed_description']}#{item['free_text'] ? ' (registro manual)' : ''} — " \
                   "#{item['quantity']} #{item['quantity_unit']}"), style: :bold
        details = [ "Uso: #{ROUTES.fetch(item['route'])}", item["dosage_instructions"],
                    item["duration_days"] && "por #{item['duration_days']} dia(s)", item["continuous"] && "uso contínuo" ].compact
        pdf.indent(14) { pdf.text s(details.join(". ")) }
        pdf.move_down 4
      end
      pdf.text s("Validade: até #{date(content['valid_until'])}#{content['antimicrobial'] ? ' (10 dias — antimicrobiano)' : ''}") if content["valid_until"]
    end

    def exam_requisition_body(pdf, _document, content)
      pdf.move_down 8
      content["exams"].each do |exam|
        pdf.text s("#{exam['sigtap_code']} — #{exam['label']} (competência #{exam['competence']})" \
                   "#{exam['cid10_justification'] ? " — CID-10 #{exam['cid10_justification']}" : ''}")
      end
      note(pdf, content)
    end

    def note(pdf, content)
      return unless content["note"]

      pdf.move_down 6
      pdf.text s("Observação: #{content['note']}")
    end

    def professional(pdf, document, content)
      author = document.author_user
      professional = author.professional
      pdf.move_down 14
      pdf.text s(Screenings::Json.staff_name(author)), style: :bold
      if professional
        pdf.text s("#{professional.council}-#{professional.council_state} #{professional.registration_number}")
      end
      pdf.text s("CBO #{document.cbo_code} #{Professionals::Cbo.find(document.cbo_code)&.title}") if document.cbo_code
      pdf.text s("Matrícula: #{content['issuer_registration']}") if content["issuer_registration"]
    end

    def hand_signature(pdf)
      pdf.move_down 30
      pdf.stroke_horizontal_line 0, 250
      pdf.move_down 4
      pdf.text s("Assinatura e carimbo")
    end

    def verification(pdf, document)
      pdf.start_new_page if pdf.cursor < QR_SIZE + 20
      pdf.move_down 12
      top = pdf.cursor
      Qr.draw(pdf, document.verification_url, at: [ 0, top ], size: QR_SIZE)
      pdf.bounding_box([ QR_SIZE + 10, top - 10 ], width: pdf.bounds.width - QR_SIZE - 10) do
        pdf.text s("Confira a autenticidade e a situação deste documento:"), size: 9
        pdf.text s(document.verification_url), size: 8
        pdf.text s("ou em #{CityPublicUrl.base(Current.city)}/v com o código #{Codes.display(document.short_code)} e o ano de nascimento do paciente."), size: 9
      end
      pdf.move_cursor_to(top - QR_SIZE) if pdf.cursor > top - QR_SIZE
    end
    private_class_method :copy, :s, :date, :header, :person, :sick_note_body, :attendance_declaration_body,
                         :prescription_body, :exam_requisition_body, :note, :professional, :hand_signature, :verification
  end
end
```

> `Consultations::Print.stamp_footer` e `.safe` são `module_function` públicos no 19a/19b (conferido na Task 0).

No `ClinicalDocumentsController`, acrescente a ação (o `require_staff` da Task 16 já aceita a recepção no `print`, para a própria declaração):

```ruby
  # Contrato §3 (Desvio 7): o PDF do modo do documento, na hora, sem cache.
  def print
    document = readable_document
    return unless document
    return render(json: { error: "already_cancelled" }, status: :conflict) if document.cancelled?

    bytes =
      if document.digital?
        signature = SignatureRequest.find_by(document_type: "ClinicalDocument", document_id: document.id, status: "signed")&.signature
        return render(json: { error: "awaiting_signature" }, status: :conflict) unless signature

        signature.signed_pdf_bytes
      else
        ClinicalDocuments::Pdf.call(document)
      end
    response.headers["Cache-Control"] = "no-store"
    send_data bytes, type: "application/pdf", disposition: "inline", filename: "documento.pdf"
  end
```

Rota (no `scope "/attendance"`, junto das outras de documento): `get "documents/:id/print", to: "clinical_documents#print"`.

- [ ] **Step 5: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/pdf_spec.rb spec/requests/clinical_document_print_spec.rb:20`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add Gemfile Gemfile.lock app/services/clinical_documents/qr.rb app/services/clinical_documents/pdf.rb app/controllers/clinical_documents_controller.rb config/routes.rb spec/services/clinical_documents/pdf_spec.rb spec/requests/clinical_document_print_spec.rb
git commit -m "feat: render clinical document PDFs with a verification QR code

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 5 — Assinatura, conferência e cancelamento (F-19.23)

### Task 18: Integração com o 19b — JSON canônico `clinical_document.v1`, PAdES e volta ao papel

**Files:**
- Create: `config/clinical/clinical-document-v1.json` (cópia da tag), `spec/fixtures/clinical/<diretório de exemplos da tag>/**`, `app/services/clinical_documents/canonical.rb`
- Modify: `spec/fixtures/clinical/canonical/SHA256SUMS` e os `.jcs` novos (cópia da tag), `app/services/signatures/canonical.rb`, `app/services/signatures/documents.rb`, `app/services/signatures/signing.rb`, `app/services/signatures/json.rb`, `app/services/signatures/print_report.rb`, `app/commands/signatures/to_paper.rb`, `spec/services/signatures/canonical_vector_spec.rb`
- Test: `spec/services/clinical_documents/canonical_spec.rb`, `spec/services/clinical_documents/signature_integration_spec.rb`

**Interfaces:**
- Consumes: `Signatures::{Canonical,Jcs,Documents,Signing,Json,PrintReport,ToPaper,SignPending,Mode}`, `ClinicalDocuments::Pdf`, `SignatureHelpers` (`stub_psc!`, `stub_signer!`, `fake_psc`, `linked_certificate!`, `sign_document!`, `signature_city!`).
- Produces:
  - `Signatures::Canonical::DOCUMENT_SCHEMA == "rotasaude.clinical_document.v1"`; `Signatures::Canonical.clinical_document(document) -> Built` (levanta `Canonical::Invalid, "not_signable"` sem consulta ou sem paciente); `Signatures::Canonical.for(ClinicalDocument)`.
  - `ClinicalDocuments::Canonical.content(document) -> Hash` (o `content` de "Valores fixados" 1).
  - `Signatures::Documents.for(request)` monta o documento clínico (canônico + `ClinicalDocuments::Pdf.call(document, footer:)`).
  - `Signatures::Signing`: ordem cronológica pelo `issued_at` do documento; a falha de um documento clínico não segura os outros pedidos da consulta (chave da cadeia por documento).
  - `Signatures::Json.requests` com `finalized_at` = `issued_at` do documento clínico.
  - `Signatures::PrintReport.for(consultation)` só olha pedidos da consulta e dos adendos (o impresso do 19a não muda com documentos).
  - `Signatures::ToPaper.call(request, reason_code:, …)` passa o documento clínico a `paper` (menos com `document_cancelled`).

- [ ] **Step 1: Copie o esquema e os exemplos da tag**

```bash
mkdir -p apps/api/.claude/mod19c/config/clinical
/opt/homebrew/bin/git -C contracts show clinical-v1.1.0:clinical/clinical-document-v1.json > apps/api/.claude/mod19c/config/clinical/clinical-document-v1.json
for f in $(/opt/homebrew/bin/git -C contracts ls-tree -r --name-only clinical-v1.1.0 clinical/examples/ | grep -v "examples/consultation"); do
  target="apps/api/.claude/mod19c/spec/fixtures/clinical/${f#clinical/examples/}"
  mkdir -p "$(dirname "$target")"
  /opt/homebrew/bin/git -C contracts show "clinical-v1.1.0:$f" > "$target"
done
cat apps/api/.claude/mod19c/spec/fixtures/clinical/canonical/SHA256SUMS
```
Expected: o esquema copiado; o diretório de exemplos do documento (o nome anotado na Task 0, abaixo chamado `<EX>`) com `manifest.json`; o `SHA256SUMS` com os dois do 19b **e** os novos. O destino é `spec/fixtures/clinical/…`, o mesmo do 19b; o `canonical/SHA256SUMS` e os `.jcs` do 19b são regravados com o conteúdo da tag nova (iguais, mais as linhas novas).

- [ ] **Step 2: Escreva as specs que falham**

Em `spec/services/signatures/canonical_vector_spec.rb`, troque o `eq` do primeiro exemplo por `include` (o arquivo agora tem mais linhas):

```ruby
    expect(sums).to include("consultation-full.jcs" => "4caa5160378b1ed8a3a29a6338af1709dd6adbf7744a79e2996e5019d91f4e70",
                            "addendum-structured.jcs" => "6ba35a6bd3dab76bdbe515769608fe30473dd3acc15e280672b8b41e54259279")
```

```ruby
# spec/services/clinical_documents/canonical_spec.rb
require "rails_helper"

# ADR 0033 (spec §6; contrato §9): o JSON canônico do documento clínico na
# forma EXATA do esquema da tag clinical-v1.1.0 (cópia em config/clinical/),
# provado byte a byte contra o vetor do contracts; o construtor produz JSON
# válido para os quatro tipos e é determinístico.
RSpec.describe "JSON canônico rotasaude.clinical_document.v1" do
  dir = Rails.root.join("spec/fixtures/clinical")
  examples = "clinical-document" # clinical/examples/clinical-document/ da tag (plano do contracts-repo)
  sums = File.read(dir.join("canonical/SHA256SUMS")).lines.to_h { |line| line.split.then { |sha, name| [ name, sha ] } }
  schema = Signatures::Canonical::DOCUMENT_SCHEMA

  JSON.parse(File.read(dir.join(examples, "manifest.json"))).fetch("cases").each do |c|
    it "#{examples}/#{c['file']}: #{c['expect']}" do
      document = JSON.parse(File.read(dir.join(examples, c["file"])))
      if c["expect"] == "valid"
        expect { Signatures::Canonical.validate!(schema, document) }.not_to raise_error
        jcs = "#{File.basename(c['file'], '.json')}.jcs"
        if sums.key?(jcs)
          out = Signatures::Jcs.dump(document)
          expect(out.b).to eq(File.binread(dir.join("canonical", jcs)))
          expect(Digest::SHA256.hexdigest(out)).to eq(sums.fetch(jcs))
        end
      else
        expect { Signatures::Canonical.validate!(schema, document) }.to raise_error(Signatures::Canonical::Invalid, /#{Regexp.escape(c['expect'].split.first)}/)
      end
    end
  end

  it "o vetor do documento clínico é o publicado (prescription-doctor, sick-note-leave)" do
    expect(sums).to include("prescription-doctor.jcs" => "62bd59faef736229bfc314adb643a128fa557f4434c6277085f031efa1c170f5",
                            "sick-note-leave.jcs" => "171c46109f960cdebb7c78dba67a29d2d6e548cb84bd29961145a8cb3bc77afc")
  end

  # O construtor reproduz o vetor: o conteúdo do exemplo, passado pelo
  # ClinicalDocuments::Canonical (normalização de note/catalog_release), dá os
  # mesmos bytes — prova de que a normalização não muda documento já canônico.
  %w[prescription-doctor sick-note-leave].each do |name|
    it "#{name}: a normalização do construtor preserva os bytes do vetor" do
      example = JSON.parse(File.read(dir.join(examples, "#{name}.json")))
      kind = example.dig("document", "kind")
      document = instance_double(ClinicalDocument, kind: kind, content_data: example["content"].deep_dup)
      rebuilt = example.merge("content" => ClinicalDocuments::Canonical.content(document))
      expect(Signatures::Jcs.dump(rebuilt).b).to eq(File.binread(dir.join("canonical", "#{name}.jcs")))
    end
  end

  describe "construtor" do
    before { documents_city!; ciap2_release!; cid10_release!; sigtap_release!; city_profile!; import_test_catalog! }

    let(:unit) { create_unit }
    let(:doctor) { doctor!(unit).tap { |user| user.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) } }
    let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

    def issue!(kind, content) = ClinicalDocuments::Issue.call(kind: kind, content: content, by: doctor, consultation: consultation).payload.fetch(:document)

    it "os quatro tipos saem válidos no esquema, com o cabeçalho da consulta" do
      documents = [
        issue!("sick_note", { "type" => "leave", "days" => 2, "cid10" => { "code" => "I10" }, "cid_authorized" => true }),
        issue!("attendance_declaration", {}),
        issue!("prescription", { "items" => [ { "catalog_item" => { "id" => catalog_item(267_778).id }, "quantity" => 30,
                                                "quantity_unit" => "comprimido", "route" => "oral",
                                                "dosage_instructions" => "1 comprimido se dor", "continuous" => true },
                                              { "free_text" => "Chá de camomila", "quantity" => 1.5, "quantity_unit" => "caixa",
                                                "route" => "oral", "dosage_instructions" => "1 sachê à noite" } ] }),
        issue!("exam_requisition", { "note" => "" })
      ]
      documents.each do |document|
        built = Signatures::Canonical.clinical_document(document)
        expect(built.document).to include("schema" => "rotasaude.clinical_document.v1",
                                          "document" => { "id" => document.id, "kind" => document.kind,
                                                          "issued_at" => document.issued_at.utc.iso8601, "replaces_document_id" => nil },
                                          "professional" => include("cpf" => SignatureHelpers::DOCTOR_CPF, "cbo_code" => "225125"))
        expect(Signatures::Canonical.clinical_document(document.reload).sha256).to eq(built.sha256) # determinístico
      end
      prescription = Signatures::Canonical.clinical_document(documents[2]).document["content"]
      expect(prescription["catalog_release"]).to eq(catalog_item(267_778).last_release_id)
      expect(prescription["items"].first).to include("catalog_item" => include("catmat_code" => 267_778, "dosage_form" => "comprimido"),
                                                     "quantity" => 30, "continuous" => true)
      expect(prescription["items"].first).not_to have_key("free_text")
      expect(prescription["items"].second).to include("free_text" => "Chá de camomila", "quantity" => 1.5)
      expect(prescription["items"].second).not_to have_key("catalog_item")
      expect(Signatures::Canonical.clinical_document(documents[3]).document["content"]["note"]).to be_nil
    end

    it "item do catálogo sem forma farmacêutica sai com dosage_form null e assina (decisão do usuário, 2026-10-10)" do
      document = issue!("prescription", { "items" => [ { "catalog_item" => { "id" => catalog_item(268_856).id }, "quantity" => 30,
                                                         "quantity_unit" => "comprimido", "route" => "oral",
                                                         "dosage_instructions" => "1 comprimido pela manhã" } ] })
      item = Signatures::Canonical.clinical_document(document).document["content"]["items"].first
      expect(item["catalog_item"]).to include("catmat_code" => 268_856, "dosage_form" => nil)
      expect(item["catalog_item"]).to have_key("dosage_form")
    end

    it "declaração da recepção não é assinável" do
      reception = ClinicalDocuments::Issue.call(kind: "attendance_declaration", content: {}, by: verifier!,
                                                attendance: consultation.attendance).payload[:document]
      expect { Signatures::Canonical.clinical_document(reception) }.to raise_error(Signatures::Canonical::Invalid, /not_signable/)
    end
  end
end
```

```ruby
# spec/services/clinical_documents/signature_integration_spec.rb
require "rails_helper"

# ADR 0033 + ADR 0032: o documento clínico digital é assinado como a consulta
# (JSON canônico em CAdES, PDF com rodapé NGS2 em PAdES), pelo job ou pelo
# lote; a volta ao papel passa o documento a paper; o impresso da consulta do
# 19a não muda por causa de documentos.
RSpec.describe "Assinatura de documento clínico" do
  before do
    documents_city!; ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    stub_psc!
    stub_signer!
    signature_city!
  end

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit).tap { |user| user.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def issue! = ClinicalDocuments::Issue.call(kind: "sick_note", content: { "type" => "leave", "days" => 2 }, by: doctor, consultation: consultation).payload[:document]

  it "assina ponta a ponta com o PSC falso: assinatura do documento, bloco digital, fila com issued_at" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    document = issue!
    request = document.signature_request
    expect(Signatures::Json.requests([ request ]).sole).to include(document_type: "clinical_document", document_id: document.id,
                                                                  finalized_at: document.issued_at.iso8601)
    signature = sign_document!(document, author: doctor)
    expect(signature).to have_attributes(document_type: "ClinicalDocument", document_id: document.id)
    expect(JSON.parse(signature.canonical_json)).to include("schema" => "rotasaude.clinical_document.v1")
    expect(ClinicalDocuments::Json.document(document.reload)[:signature]).to include(mode: "digital", simulated: false)
    expect { document.update!(issue_mode: "paper") }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
  end

  it "volta ao papel passa o documento a paper; cancelado sai sem mudar o modo" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    first = issue!
    Signatures::ReturnToPaper.call(request_id: first.signature_request.id, by: doctor, reason: "Paciente pediu o papel agora")
    expect(first.reload.issue_mode).to eq("paper")
    second = issue!
    Signatures::ToPaper.call(second.signature_request.lock!, reason_code: "document_cancelled")
    expect(second.reload.issue_mode).to eq("digital")
  end

  it "o impresso da consulta do 19a ignora pedidos de documentos" do
    consultation # finalizada antes do certificado: a consulta não tem pedido
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    issue!
    expect(Signatures::PrintReport.for(consultation)).to be_nil
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/canonical_spec.rb spec/services/clinical_documents/signature_integration_spec.rb spec/services/signatures/canonical_vector_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::Canonical::DOCUMENT_SCHEMA`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/clinical_documents/canonical.rb
# O `content` do JSON canônico do documento clínico (ADR 0033; esquema
# clinical-document-v1 da tag clinical-v1.1.0): a forma do contrato §2, a
# mesma gravada no documento. Só normaliza o que o esquema fixa: `note`
# sempre presente (null quando vazio) no atestado e na requisição; na receita,
# `catalog_release` sempre presente (null se só texto livre) e opcional
# ausente quando não se aplica; `catalog_item` vai inteiro, com
# `dosage_form: null` quando o CATMAT não traz a forma (decisão do usuário,
# 2026-10-10). O que o esquema não aceita faz o construtor levantar Invalid —
# o pedido fica pendente com verification_failed, nunca assina fora do esquema.
module ClinicalDocuments
  module Canonical
    module_function

    def content(document)
      data = document.content_data
      case document.kind
      when "sick_note", "exam_requisition" then data.except("note").merge("note" => data["note"].presence)
      when "attendance_declaration" then data
      when "prescription"
        items = data["items"].map(&:compact) # catalog_item inteiro: dosage_form pode ser null
        data.except("items", "catalog_release").merge("items" => items, "catalog_release" => data["catalog_release"])
      end
    end
  end
end
```

Em `app/services/signatures/canonical.rb`:

1. Constantes:

```ruby
    DOCUMENT_SCHEMA = "rotasaude.clinical_document.v1".freeze
    SCHEMA_FILES = { CONSULTATION_SCHEMA => Rails.root.join("config/clinical/consultation-v1.json"),
                     ADDENDUM_SCHEMA => Rails.root.join("config/clinical/consultation-addendum-v1.json"),
                     DOCUMENT_SCHEMA => Rails.root.join("config/clinical/clinical-document-v1.json") }.freeze
```

2. `for` ganha `when ClinicalDocument then clinical_document(document)`.

3. O método novo, depois de `addendum`:

```ruby
    # ADR 0033: documento clínico da consulta (o da recepção nunca é assinado).
    def clinical_document(document)
      raise Invalid, "not_signable" unless document.consultation_id && document.patient

      head = header_for(unit: document.attendance.health_unit, author: document.author_user, cbo_code: document.cbo_code,
                        patient: document.patient)
      body = { "id" => document.id, "kind" => document.kind, "issued_at" => time(document.issued_at),
               "replaces_document_id" => document.replaces_document_id }
      built(DOCUMENT_SCHEMA, head.merge("schema" => DOCUMENT_SCHEMA, "document" => body,
                                        "content" => ClinicalDocuments::Canonical.content(document)))
    end
```

4. `header(consultation, author)` passa a delegar, e o corpo antigo vira `header_for` (mesmo conteúdo, com unidade, CBO e paciente por parâmetro):

```ruby
    def header(consultation, author)
      header_for(unit: consultation.attendance.health_unit, author: author, cbo_code: consultation.cbo_code,
                 patient: consultation.patient)
    end

    def header_for(unit:, author:, cbo_code:, patient:)
      profile = CityProfile.current
      professional = author.professional
      council = professional && { "name" => professional.council, "state" => professional.council_state,
                                  "registration_number" => professional.registration_number }
      { "city" => { "ibge_code" => profile&.ibge_code.presence, "name" => profile&.name.presence || Current.city&.name },
        "unit" => { "cnes" => unit.cnes.presence, "name" => unit.name },
        "professional" => { "name" => professional&.professional_name, "cpf" => professional&.cpf,
                            "cbo_code" => cbo_code, "council" => council },
        "patient" => { "display_name" => patient.display_name, "cpf" => patient.cpf, "birth_date" => patient.birth_date.presence } }
    end
```

e acrescente `:header_for` ao `private_class_method`. (O vetor do 19b prova que a consulta e o adendo saem byte a byte iguais.)

Em `app/services/signatures/documents.rb`, no `case document`:

```ruby
      when ClinicalDocument
        Prepared.new(request: request, canonical: Canonical.clinical_document(document),
                     pdf: ClinicalDocuments::Pdf.call(document, footer: footer))
```

Em `app/services/signatures/signing.rb`:
- em `chronological`, `when ClinicalDocument then document.issued_at` no `case`;
- troque as quatro ocorrências de `request.consultation_id` usadas como chave de `blocked` e `unstored` por `chain_key(request)` (`next block(...) if blocked.key?(chain_key(request))`, `*blocked[chain_key(request)]`, `blocked[chain_key(request)] ||= …` nos dois `rescue`, `unstored.key?(chain_key(request))`, `unstored[chain_key(request)]`, `unstored[chain_key(request)] = result`);
- o `rescue Canonical::Invalid, Consultations::Print::NotPrintable` vira `rescue Canonical::Invalid, Consultations::Print::NotPrintable, ClinicalDocuments::Pdf::NotPrintable`;
- o método novo (e `:chain_key` no `private_class_method`):

```ruby
    # A cadeia (ADR 0032) é da consulta e dos adendos dela; o documento clínico
    # não encadeia — a falha de um não segura os outros pedidos da consulta.
    def chain_key(request) = request.document_type == "ClinicalDocument" ? [ "ClinicalDocument", request.document_id ] : request.consultation_id
```

Em `app/services/signatures/json.rb` (`requests`):

```ruby
      document_ids = rows.select { |row| row.document_type == "ClinicalDocument" }.map(&:document_id)
      documents = ClinicalDocument.where(id: document_ids).pluck(:id, :issued_at).to_h
```

e o `finalized_at` por tipo:

```ruby
        finalized_at = case row.document_type
                       when "Consultation" then consultation&.finalized_at
                       when "ConsultationAddendum" then addenda[row.document_id]
                       else documents[row.document_id]
                       end
```

Em `app/services/signatures/print_report.rb`: `CONSULTATION_TYPES = %w[Consultation ConsultationAddendum].freeze`; em `for`, `return nil unless SignatureRequest.exists?(consultation_id: consultation.id, document_type: CONSULTATION_TYPES)` e o laço de revalidação com `SignatureRequest.where(consultation_id: consultation.id, status: "signed", document_type: CONSULTATION_TYPES)`.

Em `app/commands/signatures/to_paper.rb`:

```ruby
    def call(request, reason_code:, note: nil, now: Time.current)
      request.update!(status: "returned_to_paper", reason_code: reason_code, return_note: note, resolved_at: now)
      # ADR 0033 (spec §4): a volta ao papel passa o documento clínico a paper
      # (o trigger só aceita sem assinatura gravada); o cancelamento tira o
      # pedido da fila sem mudar o modo.
      if request.document_type == "ClinicalDocument" && reason_code != "document_cancelled"
        ClinicalDocument.where(id: request.document_id, issue_mode: "digital").find_each { |document| document.update!(issue_mode: "paper") }
      end
      DomainEvents.publish("signature.returned_to_paper", request_id: request.id, reason_code: reason_code)
      request
    end
```

- [ ] **Step 5: Rode e veja passar (e a suíte do 19b)**

Run: `rspec spec/services/clinical_documents spec/services/signatures spec/commands/signatures spec/requests/signature_queue_spec.rb spec/requests/consultation_print_spec.rb spec/requests/clinical_document_print_spec.rb`
Expected: PASS (inclusive o segundo exemplo do impresso, da Task 17).

- [ ] **Step 6: Commit**

```bash
git add config/clinical/clinical-document-v1.json spec/fixtures/clinical app/services/clinical_documents/canonical.rb app/services/signatures/canonical.rb app/services/signatures/documents.rb app/services/signatures/signing.rb app/services/signatures/json.rb app/services/signatures/print_report.rb app/commands/signatures/to_paper.rb spec/services/signatures/canonical_vector_spec.rb spec/services/clinical_documents/canonical_spec.rb spec/services/clinical_documents/signature_integration_spec.rb
git commit -m "feat: sign clinical documents with the canonical clinical_document.v1 JSON and PAdES

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: O pedido de documento cancelado sai da fila (job, lote e varredura)

**Files:**
- Create: `app/services/signatures/cancelled_documents.rb`
- Modify: `app/commands/signatures/sign_pending.rb`, `app/commands/signatures/run_batch.rb`, `app/jobs/signatures/sweep_job.rb`
- Test: `spec/services/signatures/cancelled_documents_spec.rb`

**Interfaces:**
- Consumes: `Signatures::ToPaper` (Task 18), `ClinicalDocument`.
- Produces: `Signatures::CancelledDocuments.retire!(requests, now: Time.current) -> Array<SignatureRequest>` — recebe pedidos **já travados** pelo chamador; os de documento clínico cancelado voltam ao papel com `document_cancelled` e saem da lista devolvida. `SignPending` devolve `:returned_to_paper` para eles; `RunBatch` os tira do lote; `SweepJob` os aposenta a cada volta (SKIP LOCKED, um por transação).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/signatures/cancelled_documents_spec.rb
require "rails_helper"

# Desvio 6 (Review Focus 4): pedido pendente de documento cancelado nunca
# fica na fila — sai pelo job, pelo lote ou pela varredura, com
# document_cancelled e sem mudar o modo do documento.
RSpec.describe Signatures::CancelledDocuments do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release!; stub_psc!; stub_signer!; signature_city! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit).tap { |user| user.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def cancelled_digital!
    consultation
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF)) unless SignerCertificate.active.exists?(user_id: doctor.id)
    document = ClinicalDocuments::Issue.call(kind: "sick_note", content: { "type" => "leave", "days" => 1 }, by: doctor,
                                             consultation: consultation).payload[:document]
    document.update!(status: "cancelled", cancel_reason: "Emitido para o paciente errado", cancelled_at: Time.current,
                     cancelled_by_user_id: doctor.id)
    document
  end

  it "o job devolve ao papel sem assinar" do
    document = cancelled_digital!
    outcome = ApplicationRecord.transaction { Signatures::SignPending.call(request_id: document.signature_request.id) }
    expect(outcome).to eq(:returned_to_paper)
    expect(document.signature_request.reload).to have_attributes(status: "returned_to_paper", reason_code: "document_cancelled")
    expect(document.reload.issue_mode).to eq("digital")
    expect(Signature.where(document_type: "ClinicalDocument", document_id: document.id)).to be_empty
  end

  it "o lote tira o cancelado e assina o resto" do
    document = cancelled_digital!
    other = ClinicalDocuments::Issue.call(kind: "sick_note", content: { "type" => "leave", "days" => 2 }, by: doctor,
                                          consultation: consultation).payload[:document]
    token = Signatures::Psc::Token.new(access_token: fake_psc.token_for!(cpf: SignatureHelpers::DOCTOR_CPF), expires_in: 300,
                                       scope: "multi_signature")
    result = Signatures::RunBatch.call(user: doctor, request_ids: [ document.signature_request.id, other.signature_request.id ],
                                       token: token, provider: "vidaas")
    expect(result.payload[:record][:signed]).to eq(1)
    expect(document.signature_request.reload.reason_code).to eq("document_cancelled")
  end

  it "a varredura aposenta o que sobrou" do
    document = cancelled_digital!
    Signatures::SweepJob.new.send(:retire_cancelled_documents, Time.current)
    expect(document.signature_request.reload.status).to eq("returned_to_paper")
  end
end
```

(O exemplo da varredura chama o método privado novo do job direto, na conexão da cidade do exemplo — o `perform` passa pelo `EachCityJob`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/signatures/cancelled_documents_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::CancelledDocuments`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/signatures/cancelled_documents.rb
# Desvio 6 (ADR 0033): pedido pendente de documento clínico cancelado volta ao
# papel com document_cancelled (o modo do documento não muda) e sai da lista.
# Quem chama já travou os pedidos (job, lote, varredura, cancelamento).
module Signatures
  module CancelledDocuments
    module_function

    def retire!(requests, now: Time.current)
      ids = requests.select { |request| request.document_type == "ClinicalDocument" }.map(&:document_id)
      return requests if ids.empty?

      cancelled = ClinicalDocument.where(id: ids, status: "cancelled").pluck(:id).to_set
      requests.reject do |request|
        next false unless request.document_type == "ClinicalDocument" && cancelled.include?(request.document_id)

        ToPaper.call(request, reason_code: "document_cancelled", now: now)
        true
      end
    end
  end
end
```

Em `app/commands/signatures/sign_pending.rb`, logo depois de `return :skipped unless request&.pending?`:

```ruby
      return :returned_to_paper if CancelledDocuments.retire!([ request ], now: now).empty?
```

Em `app/commands/signatures/run_batch.rb`, troque a linha do `requests = …lock(...).to_a` por ela seguida de

```ruby
        requests = CancelledDocuments.retire!(requests, now: now)
```

(antes do `next Result.ok(record: { signed: 0, failed: [] }) if requests.empty?`).

Em `app/jobs/signatures/sweep_job.rb`, `perform` ganha `retire_cancelled_documents(now)` como primeira linha, e o método privado:

```ruby
    # Desvio 6 (ADR 0033): documento cancelado com pedido ainda pendente (o
    # job estava assinando quando o cancelamento tentou tirá-lo da fila).
    def retire_cancelled_documents(now)
      ids = SignatureRequest.pending.where(document_type: "ClinicalDocument")
                            .where(document_id: ClinicalDocument.where(status: "cancelled").select(:id)).pluck(:id)
      ids.each do |id|
        ApplicationRecord.transaction do
          request = SignatureRequest.lock("FOR UPDATE SKIP LOCKED").find_by(id: id)
          CancelledDocuments.retire!([ request ], now: now) if request&.pending?
        end
      end
    end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/signatures/cancelled_documents_spec.rb spec/commands/signatures spec/jobs/signatures`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/signatures/cancelled_documents.rb app/commands/signatures/sign_pending.rb app/commands/signatures/run_batch.rb app/jobs/signatures/sweep_job.rb spec/services/signatures/cancelled_documents_spec.rb
git commit -m "fix: retire signature requests of cancelled clinical documents from the queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 20: Cancelamento (autora, motivo, step-up) e "cancelar e emitir outro"

**Files:**
- Create: `app/commands/clinical_documents/cancel.rb`
- Modify: `app/controllers/clinical_documents_controller.rb`, `config/routes.rb`
- Test: `spec/requests/clinical_document_cancel_spec.rb`

**Interfaces:**
- Consumes: `Signatures::CancelledDocuments.retire!` (Task 19), `MfaStepUp#require_step_up!`, `ClinicalDocuments::Issue` (`replaces_document_id`).
- Produces: `ClinicalDocuments::Cancel.call(document_id:, by:, reason:, now: Time.current) -> Result` — `ok(document:)` | `fail(:invalid_reason | :not_found | :not_author | :already_cancelled)`; publica `clinical_document.cancelled { document_id, kind }`; tira o pedido pendente da fila na hora se ninguém o travou (SKIP LOCKED). Rota `POST /attendance/documents/:id/cancel { reason }` → `<document>` (step-up; permitido com o interruptor desligado, Desvio 13).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/clinical_document_cancel_spec.rb
require "rails_helper"

# Contrato §3, spec §6: só a autora, motivo ≥ 10, step-up; 409 já cancelado;
# o pedido pendente sai da fila; "cancelar e emitir outro" é um novo com
# replaces_document_id. Review Focus 4: pedido travado pelo job não trava o
# cancelamento e não fica preso na fila.
RSpec.describe "Cancelamento de documento clínico", type: :request do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit).tap { |user| user.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def issue! = ClinicalDocuments::Issue.call(kind: "sick_note", content: { "type" => "leave", "days" => 2 }, by: doctor, consultation: consultation).payload[:document]

  it "autora com step-up cancela; motivo curto 422; segunda vez 409; sem step-up 401; outra pessoa 403" do
    document = issue!
    sign_in_as(step_up_enrolled!(doctor))
    json_post "/attendance/documents/#{document.id}/cancel", reason: "Dias errados no atestado"
    expect(json_body).to eq("error" => "mfa_required")
    stepped_up!(doctor)
    json_post "/attendance/documents/#{document.id}/cancel", reason: "curto"
    expect([ response.status, json_body ]).to eq([ 422, { "error" => "invalid_reason" } ])
    json_post "/attendance/documents/#{document.id}/cancel", reason: "Dias errados no atestado"
    expect(json_body).to include("status" => "cancelled", "cancel_reason" => "Dias errados no atestado")
    expect(DomainEvent.where(name: "clinical_document.cancelled").last.payload).to eq("document_id" => document.id, "kind" => "sick_note")
    json_post "/attendance/documents/#{document.id}/cancel", reason: "Dias errados no atestado"
    expect([ response.status, json_body ]).to eq([ 409, { "error" => "already_cancelled" } ])
    stepped_up!(doctor!(unit))
    json_post "/attendance/documents/#{issue!.id}/cancel", reason: "Dias errados no atestado"
    expect([ response.status, json_body ]).to eq([ 403, { "error" => "not_author" } ])
  end

  it "cancelar e emitir outro; cancelar segue com o interruptor desligado" do
    document = issue!
    stepped_up!(doctor)
    documents_city!(enabled: false)
    json_post "/attendance/documents/#{document.id}/cancel", reason: "Dias errados no atestado"
    expect(response).to have_http_status(:ok)
    documents_city!
    json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 3 },
                                                                       replaces_document_id: document.id
    expect(json_body["replaces_document_id"]).to eq(document.id)
  end

  it "pedido pendente sai da fila; travado pelo job, sai depois pelo job (Review Focus 4)" do
    stub_psc!; stub_signer!; signature_city!
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    free = issue!
    held = issue!
    stepped_up!(doctor)
    json_post "/attendance/documents/#{free.id}/cancel", reason: "Emitido para o paciente errado"
    expect(free.signature_request.reload).to have_attributes(status: "returned_to_paper", reason_code: "document_cancelled")

    skipped = SignatureRequest.none
    allow(SignatureRequest).to receive(:lock).and_call_original
    allow(SignatureRequest).to receive(:lock).with("FOR UPDATE SKIP LOCKED").and_return(skipped) # o job segura o pedido
    json_post "/attendance/documents/#{held.id}/cancel", reason: "Emitido para o paciente errado"
    expect(response).to have_http_status(:ok)
    expect(held.signature_request.reload.status).to eq("pending")
    allow(SignatureRequest).to receive(:lock).and_call_original
    expect(ApplicationRecord.transaction { Signatures::SignPending.call(request_id: held.signature_request.id) }).to eq(:returned_to_paper)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/clinical_document_cancel_spec.rb`
Expected: FAIL (rota inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/clinical_documents/cancel.rb
# Cancelamento (ADR 0033; spec §6; contrato §3): só a autora, com motivo de
# 10 a 500 caracteres (cifrado; nunca em log ou evento). O documento fica
# `cancelled` (o trigger só aceita esta transição). Depois do commit, o pedido
# de assinatura pendente sai da fila se ninguém o estiver assinando (SKIP
# LOCKED); senão o job, o lote ou a varredura o tiram (Desvio 6). Cancelar
# não desfaz medicamento já entregue (aviso na tela).
module ClinicalDocuments
  module Cancel
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

    module_function

    def call(document_id:, by:, reason:, now: Time.current)
      note = reason.is_a?(String) ? reason.squish : ""
      return Result.fail(:invalid_reason) unless ClinicalDocument::CANCEL_REASON.cover?(note.size)
      return Result.fail(:not_found) unless document_id.to_s.match?(UUID)

      result = ApplicationRecord.transaction do
        document = ClinicalDocument.lock.find_by(id: document_id)
        next Result.fail(:not_found) unless document
        next Result.fail(:not_author) unless document.author_user_id == by.id
        next Result.fail(:already_cancelled) if document.cancelled?

        document.update!(status: "cancelled", cancel_reason: note, cancelled_at: now, cancelled_by_user_id: by.id)
        DomainEvents.publish("clinical_document.cancelled", document_id: document.id, kind: document.kind)
        Result.ok(document: document)
      end
      retire(result.payload[:document], now) if result.ok?
      result
    end

    def retire(document, now)
      ApplicationRecord.transaction do
        request = SignatureRequest.lock("FOR UPDATE SKIP LOCKED")
                                  .find_by(document_type: "ClinicalDocument", document_id: document.id, status: "pending")
        Signatures::CancelledDocuments.retire!([ request ], now: now) if request
      end
    end
    private_class_method :retire
  end
end
```

No `ClinicalDocumentsController`: `before_action :require_step_up!, only: :cancel` (depois de `require_staff`) e a ação:

```ruby
  def cancel
    result = ClinicalDocuments::Cancel.call(document_id: params[:id], by: Current.user, reason: body["reason"])
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: ClinicalDocuments::Json.document(result.payload[:document].reload)
  end
```

Rota: `post "documents/:id/cancel", to: "clinical_documents#cancel"`.

> No spec, `SignatureRequest.none.find_by(...)` devolve `nil` — o mesmo que o SKIP LOCKED pulando o pedido travado.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/clinical_document_cancel_spec.rb spec/requests/clinical_documents_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/commands/clinical_documents/cancel.rb app/controllers/clinical_documents_controller.rb config/routes.rb spec/requests/clinical_document_cancel_spec.rb
git commit -m "feat: cancel clinical documents by their author with a reason and step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 21: Página pública de conferência (`/v`) com limite de tentativas

**Files:**
- Create: `app/controllers/document_verifications_controller.rb`, `app/services/clinical_documents/verification.rb`, `app/services/clinical_documents/page.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/document_verification_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocument`, `ClinicalDocuments::Codes.normalize`, `SignatureRequest`/`Signature` (`#simulated?`, `#signed_pdf_bytes`), `DocumentVerificationLookup`.
- Produces:
  - `ClinicalDocuments::Verification.public_json(document) -> Hash` (contrato §7); `.lookup(short_code:, birth_year:) -> ClinicalDocument|nil` (grava a trilha); `.by_token(token) -> ClinicalDocument|nil` (grava a trilha); `.signature(document) -> Signature|nil`.
  - `ClinicalDocuments::Page.result(data) -> String` e `.form -> String` (HTML com escape).
  - Rotas sem login, no host da cidade: `GET /v` (formulário HTML), `POST /v/lookup` (JSON), `GET /v/:token` (HTML ou JSON por `Accept`), `GET /v/:token/signed.pdf`. Limites por IP (Valores fixados 3) → 429 `rate_limited`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/document_verification_spec.rb
require "rails_helper"

# Spec §6; contrato §7; ADR 0033 (Invariantes): a página pública mostra
# autenticidade, profissional, unidade, iniciais e ano de nascimento,
# situação e modo — nunca CID, medicamento, dias de afastamento ou CPF. O
# código curto digitado por gente (Review Focus 3) e o limite por IP.
RSpec.describe "Conferência pública de documento", type: :request do
  before do
    documents_city!; ciap2_release!; cid10_release!; sigtap_release!; import_test_catalog!
    Rails.cache.clear
    allow(Rails).to receive(:cache).and_return(ActiveSupport::Cache::MemoryStore.new)
  end

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(1) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen) }
  let(:json_headers) { { "Accept" => "application/json" } }

  def issue!(kind, content) = ClinicalDocuments::Issue.call(kind: kind, content: content, by: doctor, consultation: consultation).payload[:document]
  def sick_note! = issue!("sick_note", { "type" => "leave", "days" => 5, "cid10" => { "code" => "I10" }, "cid_authorized" => true })

  it "pelo token: só o permitido; nada clínico nem pessoal, em JSON e em HTML" do
    document = sick_note!
    get "/v/#{document.verification_token}", headers: json_headers
    body = JSON.parse(response.body)
    expect(body).to eq("kind" => "sick_note", "issued_at" => document.issued_at.iso8601,
                       "professional" => { "name" => doctor.professional.professional_name,
                                           "council" => "CRM-PR #{doctor.professional.registration_number}" },
                       "unit" => unit.name, "patient" => { "initials" => "M.A.S.", "birth_year" => Date.iso8601(citizen.birth_date).year },
                       "status" => "valid", "mode" => "paper", "simulated" => false)
    get "/v/#{document.verification_token}"
    expect(response.media_type).to eq("text/html")
    [ response.body, body.to_json ].each do |text|
      expect(text).not_to include("I10", "Hipertensão", "5 dia", citizen.cpf, "Maria Aparecida")
    end
    expect(DocumentVerificationLookup.last).to have_attributes(clinical_document_id: document.id, via: "token", outcome: "found")
  end

  it "receita: nenhum medicamento na página; cancelado mostra cancelado e quando" do
    document = issue!("prescription", { "items" => [ { "catalog_item" => { "id" => catalog_item(268_856).id }, "quantity" => 30,
                                                       "quantity_unit" => "comprimido", "route" => "oral",
                                                       "dosage_instructions" => "1 comprimido pela manhã" } ] })
    document.update!(status: "cancelled", cancel_reason: "Emitido por engano aqui", cancelled_at: Time.current, cancelled_by_user_id: doctor.id)
    get "/v/#{document.verification_token}", headers: json_headers
    expect(JSON.parse(response.body)).to include("status" => "cancelled", "cancelled_at" => document.cancelled_at.iso8601)
    expect(response.body).not_to include("LOSARTANA")
    get "/v/#{document.verification_token}"
    expect(response.body).to include("Cancelado").and(not_include("LOSARTANA"))
  end

  it "código digitado: caixa, hífen e espaço tolerados; ano errado, ano com 2 dígitos ou código inexistente → o mesmo 404" do
    document = sick_note!
    year = Date.iso8601(citizen.birth_date).year
    typed = ClinicalDocuments::Codes.display(document.short_code).downcase.sub("-", " - ")
    json_post "/v/lookup", short_code: typed, birth_year: year
    expect(JSON.parse(response.body)["status"]).to eq("valid")
    [ [ typed, year + 1 ], [ typed, year % 100 ], [ "AAAAA-AAAAA", year ], [ "", "" ] ].each do |code, birth|
      json_post "/v/lookup", short_code: code, birth_year: birth
      expect([ response.status, JSON.parse(response.body) ]).to eq([ 404, { "error" => "not_found" } ])
    end
    expect(DocumentVerificationLookup.where(outcome: "not_found").count).to eq(4)
  end

  it "o código de outra cidade não existe aqui" do
    document = sick_note!
    create(:city, slug: TEST_CITY_B.slug, status: "active", database_url: city_database_url(TEST_CITY_B_DATABASE))
    json_post "/v/lookup", { short_code: document.short_code, birth_year: Date.iso8601(citizen.birth_date).year }
    expect(response).to have_http_status(:ok)
    post "/v/lookup", params: { short_code: document.short_code, birth_year: Date.iso8601(citizen.birth_date).year }.to_json,
                      headers: { "CONTENT_TYPE" => "application/json", "HOST" => "#{TEST_CITY_B.slug}.rotasaude.app" }
    expect(response).to have_http_status(:not_found)
  end

  it "o 11º palpite do mesmo IP em 10 minutos → 429; nada da pessoa no log" do
    log = capture_log do
      10.times { json_post "/v/lookup", short_code: "AAAAA-AAAAA", birth_year: 1980 }
      json_post "/v/lookup", short_code: "AAAAA-AAAAA", birth_year: 1980
    end
    expect([ response.status, JSON.parse(response.body) ]).to eq([ 429, { "error" => "rate_limited" } ])
    expect(log).not_to include("AAAAA", "1980")
  end

  it "digital assinado: status valid, simulated, PDF assinado baixável; aguardando assinatura antes" do
    stub_psc!; stub_signer!; signature_city!
    doctor.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF)
    linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    document = sick_note!
    get "/v/#{document.verification_token}", headers: json_headers
    expect(JSON.parse(response.body)).to include("status" => "awaiting_signature", "mode" => "digital")
    get "/v/#{document.verification_token}/signed.pdf"
    expect(response).to have_http_status(:not_found)
    signature = sign_document!(document, author: doctor)
    get "/v/#{document.verification_token}", headers: json_headers
    body = JSON.parse(response.body)
    expect(body).to include("status" => "valid", "mode" => "digital", "simulated" => false)
    expect(body["signed_pdf_url"]).to end_with("/v/#{document.verification_token}/signed.pdf")
    get "/v/#{document.verification_token}/signed.pdf"
    expect(response.body.b).to eq(signature.signed_pdf_bytes.b)
  end

  it "token desconhecido → 404; o formulário existe" do
    get "/v/#{ClinicalDocuments::Codes.token}", headers: json_headers
    expect(response).to have_http_status(:not_found)
    get "/v"
    expect(response.body).to include("<form", "/v/lookup")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/document_verification_spec.rb`
Expected: FAIL (rotas inexistentes).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/verification.rb
# Conferência pública (ADR 0033; spec §6; contrato §7; Invariantes): só
# tipo, data, profissional e conselho, unidade, INICIAIS e ANO de nascimento,
# situação e modo. Nunca CID, medicamento, dias, CPF ou nome. Toda consulta
# deixa trilha sem dado pessoal (document_verification_lookups). O banco é o
# da cidade do host: o código de outra cidade simplesmente não existe aqui.
module ClinicalDocuments
  module Verification
    PARTICLES = %w[da das de do dos e].freeze

    module_function

    def by_token(token)
      document = token.is_a?(String) && token.match?(/\A[A-Za-z0-9_-]{22}\z/) ? ClinicalDocument.find_by(verification_token: token) : nil
      trail!(document, "token")
    end

    def lookup(short_code:, birth_year:)
      code = Codes.normalize(short_code)
      year = birth_year.is_a?(Integer) ? birth_year : Integer(birth_year.to_s, 10, exception: false)
      document = code && year ? ClinicalDocument.find_by(short_code: code) : nil
      document = nil if document && birth_year_of(document) != year
      trail!(document, "short_code")
    end

    def signature(document)
      return nil unless document.digital?

      SignatureRequest.find_by(document_type: "ClinicalDocument", document_id: document.id, status: "signed")&.signature
    end

    def public_json(document)
      signature = signature(document)
      author = document.author_user
      professional = author.professional
      status = if document.cancelled? then "cancelled"
               elsif document.digital? && signature.nil? then "awaiting_signature"
               else "valid"
               end
      json = { kind: document.kind, issued_at: document.issued_at.iso8601,
               professional: { name: Screenings::Json.staff_name(author),
                               council: professional && "#{professional.council}-#{professional.council_state} #{professional.registration_number}" },
               unit: document.attendance.health_unit.name,
               patient: { initials: initials(person(document).display_name), birth_year: birth_year_of(document) },
               status: status, mode: document.issue_mode, simulated: signature&.simulated? || false }
      json[:cancelled_at] = document.cancelled_at.iso8601 if document.cancelled?
      json[:signed_pdf_url] = "#{CityPublicUrl.base(Current.city)}/v/#{document.verification_token}/signed.pdf" if signature && !document.cancelled?
      json
    end

    def trail!(document, via)
      DocumentVerificationLookup.create!(clinical_document_id: document&.id, via: via, outcome: document ? "found" : "not_found")
      document
    end

    def person(document) = document.patient || document.citizen

    def birth_year_of(document)
      value = person(document).birth_date.to_s
      value.match?(/\A\d{4}-/) ? value[0, 4].to_i : nil
    end

    def initials(name)
      name.to_s.split.reject { |word| PARTICLES.include?(word.downcase) }.map { |word| "#{word[0].upcase}." }.join
    end
    private_class_method :trail!, :person, :birth_year_of, :initials
  end
end
```

```ruby
# app/services/clinical_documents/page.rb
# HTML da página pública (ADR 0033): sem assets externos, tudo escapado. O
# formulário posta JSON em /v/lookup com um script mínimo e mostra o resultado.
module ClinicalDocuments
  module Page
    KINDS = { "sick_note" => "Atestado", "attendance_declaration" => "Declaração de comparecimento",
              "prescription" => "Receita", "exam_requisition" => "Requisição de exames" }.freeze
    STATUS = { "valid" => "Válido", "cancelled" => "Cancelado", "awaiting_signature" => "Aguardando assinatura digital" }.freeze

    module_function

    def h(text) = ERB::Util.html_escape(text.to_s)

    def result(data)
      issued = Time.iso8601(data[:issued_at]).in_time_zone.strftime("%d/%m/%Y %H:%M")
      rows = [ [ "Documento", KINDS.fetch(data[:kind]) ], [ "Emitido em", issued ],
               [ "Profissional", [ data.dig(:professional, :name), data.dig(:professional, :council) ].compact.join(" — ") ],
               [ "Unidade", data[:unit] ],
               [ "Paciente", "#{data.dig(:patient, :initials)} (nascido(a) em #{data.dig(:patient, :birth_year) || '—'})" ],
               [ "Situação", STATUS.fetch(data[:status]) + (data[:cancelled_at] ? " em #{Time.iso8601(data[:cancelled_at]).in_time_zone.strftime('%d/%m/%Y %H:%M')}" : "") ],
               [ "Modo", data[:mode] == "digital" ? "Assinado digitalmente#{' (simulada — sem validade jurídica)' if data[:simulated]}" : "Papel (assinatura à mão)" ] ]
      table = rows.map { |label, value| "<tr><th>#{h(label)}</th><td>#{h(value)}</td></tr>" }.join
      link = data[:signed_pdf_url] ? %(<p><a href="#{h(data[:signed_pdf_url])}">Baixar o PDF assinado</a></p>) : ""
      layout("<table>#{table}</table>#{link}")
    end

    def form
      layout(<<~HTML)
        <form id="f"><label>Código <input name="short_code" autocomplete="off" required></label>
        <label>Ano de nascimento do paciente <input name="birth_year" inputmode="numeric" maxlength="4" required></label>
        <button>Conferir</button></form><div id="r" role="status"></div>
        <script>
        document.getElementById("f").addEventListener("submit", async (e) => {
          e.preventDefault();
          const d = new FormData(e.target);
          const r = await fetch("/v/lookup", { method: "POST", headers: { "Content-Type": "application/json", "Accept": "application/json" },
            body: JSON.stringify({ short_code: d.get("short_code"), birth_year: d.get("birth_year") }) });
          const out = document.getElementById("r");
          if (r.status === 429) { out.textContent = "Muitas tentativas. Tente de novo em alguns minutos."; return; }
          if (!r.ok) { out.textContent = "Documento não encontrado. Confira o código e o ano."; return; }
          const j = await r.json();
          out.textContent = `${j.status === "valid" ? "Válido" : j.status === "cancelled" ? "Cancelado" : "Aguardando assinatura digital"} — ${j.professional.name} — ${j.unit}`;
        });
        </script>
      HTML
    end

    def layout(body)
      <<~HTML
        <!doctype html><html lang="pt-BR"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
        <title>Conferência de documento</title>
        <style>body{font-family:system-ui,sans-serif;max-width:40rem;margin:2rem auto;padding:0 1rem;color:#111;background:#fff}
        th{text-align:left;padding:.25rem 1rem .25rem 0;vertical-align:top}label{display:block;margin:.5rem 0}</style></head>
        <body><h1>Conferência de documento — Rota Saúde</h1>#{body}</body></html>
      HTML
    end
  end
end
```

```ruby
# app/controllers/document_verifications_controller.rb
# Página pública de conferência (ADR 0033; contrato §7): sem login, no host
# da cidade (CityResolution), como /r/:token. Limites por IP (Valores fixados
# 3). JSON com Accept: application/json; HTML no navegador (o QR code).
class DocumentVerificationsController < ApplicationController
  module RateLimitStore
    def self.increment(...) = Rails.cache.increment(...)
  end

  LIMITED = -> { render json: { error: "rate_limited" }, status: :too_many_requests }
  rate_limit to: 10, within: 10.minutes, only: :lookup, name: "document_lookup", by: -> { request.remote_ip },
             store: RateLimitStore, with: LIMITED
  rate_limit to: 60, within: 10.minutes, only: :show, name: "document_show", by: -> { request.remote_ip },
             store: RateLimitStore, with: LIMITED
  rate_limit to: 30, within: 10.minutes, only: :signed_pdf, name: "document_signed_pdf", by: -> { request.remote_ip },
             store: RateLimitStore, with: LIMITED

  def form = html(ClinicalDocuments::Page.form)

  def show
    document = ClinicalDocuments::Verification.by_token(params[:token])
    return not_found unless document

    data = ClinicalDocuments::Verification.public_json(document)
    json? ? render(json: data) : html(ClinicalDocuments::Page.result(data))
  end

  def lookup
    input = request.request_parameters
    document = ClinicalDocuments::Verification.lookup(short_code: input["short_code"], birth_year: input["birth_year"])
    return render(json: { error: "not_found" }, status: :not_found) unless document

    render json: ClinicalDocuments::Verification.public_json(document)
  end

  def signed_pdf
    document = ClinicalDocuments::Verification.by_token(params[:token])
    signature = document && !document.cancelled? && ClinicalDocuments::Verification.signature(document)
    return render(json: { error: "not_found" }, status: :not_found) unless signature

    response.headers["Cache-Control"] = "no-store"
    send_data signature.signed_pdf_bytes, type: "application/pdf", disposition: "inline", filename: "documento-assinado.pdf"
  end

  private

  def json? = request.headers["Accept"].to_s.include?("application/json")
  def html(body, status: :ok) = render(body: body, content_type: "text/html; charset=utf-8", status: status)

  def not_found
    json? ? render(json: { error: "not_found" }, status: :not_found) : html(ClinicalDocuments::Page.layout("<p>Documento não encontrado.</p>"), status: :not_found)
  end
end
```

Em `config/routes.rb`, logo depois de `get "/r/:token"`:

```ruby
  # Conferência pública de documento clínico (ADR 0033; contrato §7). Host da
  # cidade; sem login. O token tem 22 caracteres url-safe (sem ponto).
  get  "/v",                     to: "document_verifications#form"
  post "/v/lookup",              to: "document_verifications#lookup"
  get  "/v/:token/signed.pdf",   to: "document_verifications#signed_pdf", format: false, constraints: { token: /[A-Za-z0-9_-]{22}/ }
  get  "/v/:token",              to: "document_verifications#show", constraints: { token: /[A-Za-z0-9_-]+/ }
```

> O `before` da spec troca o `Rails.cache` (`:null_store` em teste) por um `MemoryStore`: o `RateLimitStore` resolve `Rails.cache` a cada requisição (mesmo truque do `MfaController`), então o limite é exercitável. O `Accept` sem JSON (navegador) recebe HTML; o `capture_log` prova que `short_code` e `birth_year` ficam filtrados (Task 8).

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/document_verification_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/controllers/document_verifications_controller.rb app/services/clinical_documents/verification.rb app/services/clinical_documents/page.rb config/routes.rb spec/requests/document_verification_spec.rb
git commit -m "feat: add the public clinical document verification page with rate limits

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 6 — LGPD, invariantes, semente, prova e fechamento

### Task 22: Leitura administrativa dos documentos e LGPD (paciente com documento é retido)

**Files:**
- Create: `app/controllers/clinical_record_documents_controller.rb`
- Modify: `app/commands/citizens/request_erasure.rb`, `config/routes.rb`
- Test: `spec/requests/clinical_record_documents_spec.rb`, `spec/commands/citizens/request_erasure_documents_spec.rb`

**Interfaces:**
- Consumes: `ClinicalRecordAdministrativeRead`, `ClinicalRecord::Trail`, `ClinicalDocuments::Json.document`, `Citizens::RequestErasure`.
- Produces: `GET /clinical_record/consultations/:id/documents` (municipal_admin, step-up, consulta finalizada; linha em `clinical_record_administrative_reads` + trilha `administrative`; sem impresso) → `{ items: [<document>] }`; `Citizens::RequestErasure` retém o CPF com documento clínico (além de atendimento).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/clinical_record_documents_spec.rb
require "rails_helper"

# Contrato §3 (preâmbulo; Desvio 11): o municipal_admin lê o conteúdo dos
# documentos da consulta finalizada, com step-up, linha para sempre e trilha
# administrative; rascunho ou inexistente → 404.
RSpec.describe "Leitura administrativa dos documentos", type: :request do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  it "lê com step-up, deixa linha e trilha; sem step-up 401; profissional 403" do
    unit = create_unit
    doctor = doctor!(unit)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    document = ClinicalDocuments::Issue.call(kind: "sick_note", content: { "type" => "leave", "days" => 2 }, by: doctor,
                                             consultation: consultation).payload[:document]
    admin = municipal_admin!
    sign_in_as(step_up_enrolled!(admin))
    get "/clinical_record/consultations/#{consultation.id}/documents"
    expect(response).to have_http_status(:unauthorized)
    stepped_up!(admin)
    expect { get "/clinical_record/consultations/#{consultation.id}/documents" }.to change(ClinicalRecordAdministrativeRead, :count).by(1)
    expect(json_body["items"].map { |item| item["id"] }).to eq([ document.id ])
    expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "administrative", "consultation_id" => consultation.id)
    get "/clinical_record/consultations/#{SecureRandom.uuid}/documents"
    expect(response).to have_http_status(:not_found)
    sign_in_as(doctor)
    get "/clinical_record/consultations/#{consultation.id}/documents"
    expect(json_body).to eq("error" => "forbidden").or eq("error" => "missing_role")
  end
end
```

```ruby
# spec/commands/citizens/request_erasure_documents_spec.rb
require "rails_helper"

# ADR 0026 + ADR 0033 (spec §8): paciente com documento clínico é registro retido.
RSpec.describe Citizens::RequestErasure do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  it "par com documento clínico nasce retido" do
    unit = create_unit
    citizen = verified_citizen!(1)
    consultation = finalized_consultation!(unit: unit, doctor: doctor!(unit), citizen: citizen)
    ClinicalDocuments::Issue.call(kind: "attendance_declaration", content: {}, by: verifier!, attendance: consultation.attendance)
    allow(described_class).to receive(:attended?).and_return(false) # isola a regra nova da do atendimento
    result = described_class.call(cpf: citizen.cpf, document_checked: true, by: verifier!)
    expect(result.payload[:request].status).to eq("retained")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/clinical_record_documents_spec.rb spec/commands/citizens/request_erasure_documents_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implemente**

Em `app/commands/citizens/request_erasure.rb`, `retained = attended?(pairs) || documented?(pairs)` e:

```ruby
    # ADR 0033 (spec §8): documento clínico emitido é registro retido.
    def documented?(pairs) = ClinicalDocument.where(citizen_id: pairs.select(:id)).exists?
```

```ruby
# app/controllers/clinical_record_documents_controller.rb
# Leitura administrativa dos documentos clínicos (ADR 0033; Desvio 11), no
# mesmo desenho da leitura administrativa da consulta do 19a: municipal_admin,
# step-up, consulta finalizada; a linha em clinical_record_administrative_reads
# e a trilha administrative saem juntas, antes de renderizar. Só leitura.
class ClinicalRecordDocumentsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate
  include MfaStepUp

  before_action :require_clinical_record!
  before_action :require_admin
  before_action :require_step_up!

  def index
    consultation = params[:id].to_s.match?(/\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/) && Consultation.finalized_consultations.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless consultation

    grant = ClinicalRecord::Access::Grant.new(kind: :administrative, opening: nil, reason: nil)
    ApplicationRecord.transaction do
      ClinicalRecordAdministrativeRead.create!(user: Current.user, patient: consultation.patient, consultation: consultation)
      ClinicalRecord::Trail.viewed!(patient: consultation.patient, user: Current.user, grant: grant, consultation_id: consultation.id)
    end
    documents = ClinicalDocument.where(consultation_id: consultation.id).order(issued_at: :desc, id: :desc)
    render json: { items: documents.map { |document| ClinicalDocuments::Json.document(document) } }
  end
end
```

Rota, no `scope "/clinical_record"`: `get "consultations/:id/documents", to: "clinical_record_documents#index"`.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/clinical_record_documents_spec.rb spec/commands/citizens spec/requests/erasure_requests_spec.rb`
Expected: PASS. (O `require_admin` do `AttendanceAccess` responde 403 `forbidden`; a spec aceita os dois códigos que o projeto usa.)

- [ ] **Step 5: Commit**

```bash
git add app/controllers/clinical_record_documents_controller.rb app/commands/citizens/request_erasure.rb config/routes.rb spec/requests/clinical_record_documents_spec.rb spec/commands/citizens/request_erasure_documents_spec.rb
git commit -m "feat: add administrative reading of clinical documents and retain patients with documents

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 23: Suíte de invariantes do ADR 0033

**Files:**
- Test: `spec/invariants/clinical_documents_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 8–22. Produces: nada novo (cada bloco tem a mutação que precisa deixá-lo vermelho).

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/clinical_documents_invariants_spec.rb
require "rails_helper"

# Critério de fechamento do 19c (ADR 0033, "Invariantes"). Cada bloco diz a
# mutação que precisa deixá-lo vermelho.
RSpec.describe "Invariantes dos documentos clínicos (ADR 0033)", type: :request do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release!; import_test_catalog!; city_cnpj! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:marker) { "MARCADOR#{SecureRandom.hex(4)}" }

  def issue(kind, content, by: doctor, on: consultation) = ClinicalDocuments::Issue.call(kind: kind, content: content, by: by, consultation: on)

  # Mutação: tirar o ramo issued → cancelled de rota_clinical_document_guard, ou o not_author do Cancel.
  it "documento emitido não muda; só é cancelado, pela autora, com motivo" do
    document = issue("sick_note", { "type" => "leave", "days" => 1 }).payload[:document]
    expect { document.update!(content: { "type" => "leave", "days" => 9 }.to_json) }.to raise_error(ActiveRecord::StatementInvalid)
    expect(ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor!(unit), reason: "Motivo qualquer longo").reason).to eq(:not_author)
    expect(ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor, reason: "curto").reason).to eq(:invalid_reason)
    expect(ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor, reason: "Motivo qualquer longo")).to be_ok
  end

  # Mutação: aceitar paper → digital, ou digital → paper com assinatura gravada.
  it "o modo nunca é convertido (só a volta ao papel sem assinatura)" do
    paper = issue("sick_note", { "type" => "leave", "days" => 1 }).payload[:document]
    expect { paper.update!(issue_mode: "digital") }.to raise_error(ActiveRecord::StatementInvalid)
    signed = raw_document!(consultation: consultation, issue_mode: "digital")
    request = SignatureRequest.create!(document_type: "ClinicalDocument", document_id: signed.id, consultation_id: consultation.id,
                                       author_user_id: doctor.id)
    signature_row!(request, certificate: linked_certificate!(doctor.tap { |u| u.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) }))
    expect { signed.update!(issue_mode: "paper") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  # Mutação: tirar a checagem de controlado (catálogo ou texto) ou o `!antimicrobial` do modo.
  it "nenhum controlado emitido; antimicrobiano só em papel" do
    item = ->(code) { { "catalog_item" => { "id" => catalog_item(code).id }, "quantity" => 10, "quantity_unit" => "comprimido",
                        "route" => "oral", "dosage_instructions" => "1 ao dia" } }
    expect(issue("prescription", { "items" => [ item.(273_009) ] }).reason).to eq(:controlled_not_allowed)
    expect(issue("prescription", { "items" => [ { "free_text" => "Clonazepam 2mg", "quantity" => 10, "quantity_unit" => "cp",
                                                  "route" => "oral", "dosage_instructions" => "1 à noite" } ] }).reason).to eq(:controlled_not_allowed)
    signature_city!
    linked_certificate!(doctor.tap { |u| u.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) })
    expect(issue("prescription", { "items" => [ item.(267_625) ] }).payload[:document].issue_mode).to eq("paper")
    expect(ClinicalDocument.where(kind: "prescription").map { |d| d.content_data["items"].any? { |i| i.dig("catalog_item", "catmat_code").in?([ 267_197, 273_009 ]) } }).to all(be(false))
  end

  # Mutação: aceitar versão não vigente ou item fora do protocolo.
  it "enfermeiro só prescreve itens de protocolo vigente" do
    nurse = nurse!(unit)
    nursing = finalized_consultation!(unit: unit, doctor: nurse, citizen: verified_citizen!(2, full_name: "Rita Souza"))
    catalog = { catalog_item(267_503).id => catalog_item(267_503) }
    expired = NursingProtocols::Save.create(params: { "title" => "Antigo", "number" => "PE-00", "year" => 2025, "valid_from" => "2025-01-01",
                                                      "valid_until" => "2025-12-31", "items" => [ { "catalog_item_id" => catalog_item(267_503).id } ] },
                                            by: municipal_admin!, catalog: catalog).payload[:protocol].versions.sole
    content = { "nursing_protocol" => { "version_id" => expired.id },
                "items" => [ { "catalog_item" => { "id" => catalog_item(267_503).id }, "quantity" => 30, "quantity_unit" => "comprimido",
                               "route" => "oral", "dosage_instructions" => "1 ao dia" } ] }
    expect(issue("prescription", content, by: nurse, on: nursing).reason).to eq(:not_in_nursing_protocol)
  end

  # Mutação: pôr cid10, items ou days no public_json/na página.
  it "a página pública nunca mostra CID, medicamento, dias ou CPF" do
    document = issue("sick_note", { "type" => "leave", "days" => 7, "cid10" => { "code" => "E119" }, "cid_authorized" => true }).payload[:document]
    get "/v/#{document.verification_token}", headers: { "Accept" => "application/json" }
    json = response.body
    get "/v/#{document.verification_token}"
    expect(json + response.body).not_to include("E119", "Diabetes", "\"days\"", "7 dia", consultation.patient.cpf)
    expect(JSON.parse(json).keys - %w[kind issued_at professional unit patient status mode simulated cancelled_at signed_pdf_url]).to eq([])
  end

  # Mutação: tirar o patient_medications_guard ou o CHECK de fonte do evento.
  it "patient_medications só muda por evento ligado a consulta, adendo ou documento" do
    expect { PatientMedication.create!(patient: consultation.patient, free_text: "Chá", label: "Chá", status: "active", origin: "external") }
      .to raise_error(ActiveRecord::StatementInvalid)
    expect { PatientMedicationEvent.create!(patient_medication_id: SecureRandom.uuid, kind: "added", user: doctor, status_after: "active") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_patient_medication_events_source/)
  end

  # Mutação: pôr content/cancel_reason/nome em evento, log ou mensagem; tirar :content/:short_code do filtro.
  it "nenhum conteúdo de documento em log, evento, job, Analytics ou erro" do
    log = capture_log do
      sign_in_as(doctor)
      json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note",
                content: { type: "leave", days: 2, note: "nota #{marker}" }
      id = json_body["id"]
      json_post "/attendance/consultations/#{consultation.id}/documents", kind: "sick_note", content: { type: "leave", days: 2, note: "#{marker} " * 200 }
      expect(response.body).not_to include(marker)
      stepped_up!(doctor)
      json_post "/attendance/documents/#{id}/cancel", reason: "motivo #{marker}"
      json_post "/attendance/consultations/#{consultation.id}/documents", kind: "prescription",
                content: { items: [ { free_text: "Remédio #{marker}", quantity: 1, quantity_unit: "caixa", route: "oral", dosage_instructions: "1 ao dia" } ] }
      json_post "/v/lookup", short_code: marker, birth_year: 1980
    end
    [ log, DomainEvent.pluck(:payload).to_json, PlatformEvent.pluck(:payload).to_json,
      ActiveJob::Base.queue_adapter.enqueued_jobs.to_json ].each { |text| expect(text).not_to include(marker) }
    analytics = Dir[Rails.root.join("app/services/analytics/**/*.rb")].map { |f| File.read(f) }.join
    expect(analytics).not_to match(/clinical_documents|prescription_items|patient_medications/)
  end
end

# Mutação: tirar o `d.txid = txid_current()` de rota_prescription_item_insert_guard.
RSpec.describe "prescription_items: documento de outra transação não basta (ADR 0033)" do
  self.use_transactional_tests = false

  let(:document_id) { SecureRandom.uuid }
  let(:conn) { ApplicationRecord.connection }

  before do
    CityConnection.with(TEST_CITY_A) do
      ApplicationRecord.transaction do
        conn.execute("SET LOCAL session_replication_role = replica")
        conn.execute(<<~SQL)
          INSERT INTO clinical_documents (id, kind, patient_id, citizen_id, consultation_id, attendance_id, author_user_id, cbo_code,
                                          issue_mode, verification_token, short_code, issued_at, content, created_at)
          VALUES ('#{document_id}', 'prescription', gen_random_uuid(), gen_random_uuid(), gen_random_uuid(), gen_random_uuid(),
                  gen_random_uuid(), '225125', 'paper', '#{'A' * 22}', '#{'2' * 10}', now(), 'x', now())
        SQL
      end
    end
  end

  after do
    CityConnection.with(TEST_CITY_A) do
      ApplicationRecord.transaction do
        conn.execute("SET LOCAL session_replication_role = replica")
        conn.execute("DELETE FROM prescription_items WHERE clinical_document_id = '#{document_id}'")
        conn.execute("DELETE FROM clinical_documents WHERE id = '#{document_id}'")
      end
    end
  end

  it "recusa item novo de receita commitada antes" do
    CityConnection.with(TEST_CITY_A) do
      expect do
        ApplicationRecord.transaction do
          conn.execute(<<~SQL)
            INSERT INTO prescription_items (id, clinical_document_id, position, free_text, printed_description, quantity, quantity_unit,
                                            route, dosage_instructions, created_at)
            VALUES (gen_random_uuid(), '#{document_id}', 1, 'x', 'x', 1, 'cx', 'oral', 'x', now())
          SQL
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /born with their prescription/)
    end
  end
end
```

- [ ] **Step 2: Rode**

Run: `rspec spec/invariants/clinical_documents_invariants_spec.rb`
Expected: PASS. Faça as mutações do comentário de cada bloco, uma de cada vez, e confira que o bloco fica vermelho; desfaça.

- [ ] **Step 3: Commit**

```bash
git add spec/invariants/clinical_documents_invariants_spec.rb
git commit -m "test: pin the ADR 0033 clinical document invariants

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 24: Semente de dev

**Files:**
- Create: `lib/clinical_documents_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/clinical_documents_crew_spec.rb`

**Interfaces:**
- Consumes: `Medications::{AnvisaImport,CatalogImport}`, `Medications::Catmat::FixtureClient` (recorte real de `db/seeds/medications/catmat-dev.json`, **sem rede**), `Platform::Features.set!`, `NursingProtocols::Save`, `Cnpj.with_check_digits`, `DigitalSignatureCrew::DEV_MAINTAINER`/`dev@local`.
- Produces: `ClinicalDocumentsCrew.seed_platform! -> { anvisa:, catalog: }`; `ClinicalDocumentsCrew.seed_current_city(slug:) -> { switch:, cnpj:, remume:, protocol: }` — só Curitiba: liga `clinical_documents` (mantenedor `dev@local`), CNPJ fictício válido `76001234000115`, REMUME com 13 itens do recorte, o protocolo "Saúde da mulher na APS" (PE-01/2026, ácido fólico 5 mg até 30 comprimidos, miconazol creme vaginal, paracetamol 500 mg até 20 comprimidos, dipirona 500 mg até 20 comprimidos). Idempotente.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/clinical_documents_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/clinical_documents_crew")

# Semente de dev do 19c: catálogo de fixture (sem chamar a API externa),
# listas Anvisa do repositório, e em Curitiba o interruptor, o CNPJ, a REMUME
# e um protocolo de enfermagem. Idempotente.
RSpec.describe ClinicalDocumentsCrew do
  before do
    clinical_city!
    Maintainer.create!(email_address: "dev@local", password: "dev-password", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    stub_const("Medications::AnvisaImport::DIR", anvisa_fixture_dir)
    city_profile!
    create_unit
  end

  it "semeia a plataforma sem rede e a cidade, duas vezes sem duplicar" do
    expect(described_class.seed_platform!).to eq(anvisa: "importadas", catalog: "importado (recorte de dev)")
    expect(WebMock).not_to have_requested(:get, MedicationHelpers::CATMAT_URL)
    allow(Current.city).to receive(:slug).and_return("curitiba")
    2.times { described_class.seed_current_city(slug: "curitiba") }
    expect(ClinicalDocuments::Gate.usable?(Current.city)).to be(true)
    expect(Cnpj.normalize(CityProfile.current.cnpj)).to eq("76001234000115")
    expect(CityMedication.count).to eq(13)
    protocol = NursingProtocol.sole
    expect(protocol.current_version.items.count).to eq(4)
    expect(described_class.seed_platform!).to eq(anvisa: "já ativas", catalog: "já ativo")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/lib/clinical_documents_crew_spec.rb`
Expected: FAIL (arquivo inexistente).

- [ ] **Step 3: Implemente**

```ruby
# lib/clinical_documents_crew.rb
# Semente de dev do 19c (ADR 0033). Dev é fictício mas imita o real: o
# catálogo vem do RECORTE REAL do CATMAT em db/seeds/medications/catmat-dev.json
# pelo FixtureClient (nunca chama a API externa na semente); as listas Anvisa
# vêm do repositório. Em Curitiba: liga clinical_documents (mantenedor de dev),
# CNPJ fictício com DV válido, REMUME e um protocolo de enfermagem. Idempotente.
# Roda depois do DigitalSignatureCrew.
module ClinicalDocumentsCrew
  DEV_MAINTAINER = "dev@local".freeze
  CATALOG = Rails.root.join("db/seeds/medications/catmat-dev.json")
  CNPJ = Cnpj.with_check_digits("760012340001") # 76.001.234/0001-15, fictício
  REMUME = [ 268_856, 267_674, 267_690, 267_747, 271_089, 267_778, 267_203, 267_503, 268_162, 294_887, 267_712, 267_671, 267_772 ].freeze
  PROTOCOL = { "title" => "Saúde da mulher na APS", "number" => "PE-01", "year" => 2026, "valid_from" => "2026-01-01" }.freeze
  PROTOCOL_ITEMS = { 267_503 => [ 30, "comprimido" ], 268_162 => nil, 267_778 => [ 20, "comprimido" ], 267_203 => [ 20, "comprimido" ] }.freeze

  module_function

  def seed_platform!
    anvisa = if AnvisaListRelease.current then "já ativas"
             else Medications::AnvisaImport.call(by: "db:seed").then { |r| r.ok? ? "importadas" : "falhou: #{r.reason}" }
             end
    catalog = if MedicationCatalogRelease.current then "já ativo"
              else Medications::CatalogImport.call(by: "db:seed", client: Medications::Catmat::FixtureClient.new(CATALOG))
                                             .then { |r| r.ok? ? "importado (recorte de dev)" : "falhou: #{r.reason}" }
              end
    { anvisa: anvisa, catalog: catalog }
  end

  def seed_current_city(slug:)
    return { switch: "desligado (só Curitiba liga)" } unless slug == "curitiba"

    maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
    switch = if maintainer
               Platform::Features.set!(city: City.find(Current.city.id), key: "clinical_documents", enabled: true, maintainer: maintainer)
               missing = Platform::Features.missing(City.find(Current.city.id), "clinical_documents")
               missing.empty? ? "ligado" : "ligado, falta: #{missing.join(', ')}"
             else
               "desligado (sem mantenedor)"
             end
    CityProfile.current&.update!(cnpj: CNPJ)
    admin = User.joins(:memberships).merge(Membership.active.where(role: "municipal_admin")).order(:created_at).first
    items = MedicationCatalogItem.where(catmat_code: REMUME + PROTOCOL_ITEMS.keys).index_by(&:catmat_code)
    REMUME.each do |code|
      next unless items[code]

      CityMedication.find_or_create_by!(catalog_item_id: items[code].id) { |row| row.added_by_user = admin }
    end
    unless NursingProtocol.exists?(number: PROTOCOL["number"], year: PROTOCOL["year"])
      params = PROTOCOL.merge("items" => PROTOCOL_ITEMS.filter_map do |code, max|
        items[code] && { "catalog_item_id" => items[code].id, "max_dose" => max && { "quantity" => max[0], "unit" => max[1] } }.compact
      end)
      NursingProtocols::Save.create(params: params, by: admin, catalog: items.values.index_by(&:id))
    end
    { switch: switch, cnpj: CNPJ, remume: CityMedication.count, protocol: NursingProtocol.count }
  end
end
```

Em `db/seeds.rb`: `require_relative "../lib/clinical_documents_crew"` junto dos outros; no bloco de plataforma, depois das terminologias do 19a, `puts "[seeds] medicamentos . #{ClinicalDocumentsCrew.seed_platform!.inspect}"`; no bloco por cidade, depois da assinatura digital:

```ruby
        # ── Documentos clínicos (módulo 19c, ADR 0033) ────────────────────────
        documents = ClinicalDocumentsCrew.seed_current_city(slug: slug)
        puts "[seeds] documentos .. #{documents.inspect}"
```

(Se o `admin` de dev não existir quando a cidade não tem `municipal_admin`, a semente da cidade já o criou antes — `admin@<slug>.demo`.)

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/lib/clinical_documents_crew_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/clinical_documents_crew.rb db/seeds.rb spec/lib/clinical_documents_crew_spec.rb
git commit -m "chore: seed the dev medication catalog, REMUME and a nursing protocol

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 25: Revisão final, suíte completa, api na porta 3038, prova no navegador e rollout

- [ ] **Step 1: Varredura de vazamento.** `grep -rn 'DomainEvents.publish("clinical_document\|DomainEvents.publish("patient_medication' -A3 apps/api/.claude/mod19c/app` — só ids, `kind`, `issue_mode` (multilinha: confira cada chamada inteira). `grep -rn "Platform.audit(" -A2 apps/api/.claude/mod19c/app/services/medications` — só ids e contagens. `grep -rn "Rails.logger" apps/api/.claude/mod19c/app/services/medications apps/api/.claude/mod19c/app/services/clinical_documents apps/api/.claude/mod19c/app/commands/clinical_documents` — só classe de erro.
- [ ] **Step 2: Suíte completa** (worker parado; avise as outras sessões; bancos `_mod19c` recriados):

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod19c -w /rails/.claude/mod19c api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde. Spec antiga que fixa listas (catálogo de interruptores, bindings, schema do maintenance, `R18_PLATFORM_EVENT_NAMES`, `MaintenanceAudit::NAMES`, rotas) → acrescente o do 19c (é o contrato); nunca afrouxe asserção de texto livre.
- [ ] **Step 3:** `docker compose exec -T -w /rails/.claude/mod19c api bundle exec rubocop <arquivos tocados>`; corrija só o que a regra do projeto aponta.
- [ ] **Step 4: api na porta 3038.** Com autorização do usuário, uma etapa de cada vez: `bin/rails db:migrate:platform` (catálogo e listas na plataforma de dev), `bin/rails city:migrate:all` (as cidades de dev ganham as tabelas) e `bin/rails db:seed` (Task 24). Depois, sem derrubar o principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod19c -e CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5188 api \
    bin/rails server -b 0.0.0.0 -p 3038 -P tmp/pids/server-mod19c.pid
  ```
  O Vite do worktree do dashboard aponta `VITE_API_PROXY_TARGET` para `http://api:3038`, com `/clinical_documents` e `/v` no proxy (plano do dashboard). Para o `SignJob`, rode o job na mão no console do worktree (`Signatures::SignJob.perform_now(city_slug: "curitiba", request_id: …)`) ou um worker do worktree, como no 19b.
- [ ] **Step 5: Prova manual no navegador** (`http://curitiba.localhost:5188/dashboard/`; login, OTP e TOTP são do usuário — as senhas e o TOTP da semente de dev podem ser mostrados no chat se ele pedir):
  1. A médica (`profissional@curitiba.demo`), com certificado e sessão de assinatura (PSC simulado do 19b), atende e, na aba Documentos, emite um **atestado** (afastamento, 2 dias, CID com autorização) e uma **receita** com losartana (contínua) e amoxicilina: o atestado nasce digital (pendente → assinado; "Imprimir" baixa o PDF assinado com rodapé NGS2 e o aviso de simulada); a receita nasce em **papel** (antimicrobiano), com 2 vias, validade de 10 dias, QR code e código. A losartana aparece em "Medicamentos em uso".
  2. Tenta receitar diazepam: a busca o mostra bloqueado; a emissão responde "controlado".
  3. A enfermeira (`enfermeira@curitiba.demo`) atende outra cidadã e prescreve ácido fólico pelo protocolo PE-01/2026: o PDF traz protocolo, CNPJ 76.001.234/0001-15 e COREN; com 31 comprimidos, "acima da dose do protocolo"; texto livre, recusado.
  4. A recepção (`recepcao@curitiba.demo`) emite a declaração de comparecimento no atendimento (com matrícula) e imprime.
  5. Pelo celular, o QR code de cada PDF abre `http://curitiba.localhost:5188/v/<token>`: tipo, data, profissional, unidade, iniciais e ano, situação e modo — nenhum CID, medicamento, dias ou CPF. `/v` com o código digitado em minúsculas e o ano certo confere; o ano errado diz "não encontrado".
  6. A médica cancela o atestado (step-up, motivo): a página pública passa a "Cancelado em …"; "Cancelar e emitir outro" abre um novo com `replaces_document_id`.
  7. O admin (`admin@curitiba.demo`) ajusta a REMUME e cria a versão 2 do protocolo; o CNPJ aparece no perfil.
  8. No maintenance (`maintenance.localhost:5177`, `dev@local`): o interruptor `clinical_documents` ligado; `importAnvisaLists`; `importMedicationCatalog` **real** contra a API do Compras.gov (só leitura; com autorização do usuário) mostra +6.7 mil / mudou / saiu e a lista de revisão; desligar o interruptor recusa emitir e mantém leitura, impresso e página pública.
- [ ] **Step 6: Pare.** Merge, push, board e docs (página de status do módulo no padrão do módulo 01) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` (tag `clinical-v1.1.0`) → **api** → dashboard → maintenance. Rollout: publicar a imagem nova (com `rqrcode_core` no `Gemfile.lock`) e rodar `db:migrate:platform` e `city:migrate:all` dela **antes** de cortar tráfego (a migração de cidade é irreversível); depois, **na imagem publicada (produção não tem maintenance)**, `bin/rails medications:import_anvisa` e `bin/rails medications:import_catmat` na plataforma (`import_in_progress` se repetido ao mesmo tempo); o proxy do host de cada cidade encaminha `/v` ao api (como `/r`); o interruptor nasce desligado e é ligado por cidade pelo maintenance. Ao voltar o checkout para a main: `DROP DATABASE` dos bancos `_mod19c` (sem pedir de novo, conferindo conexões ativas) e derrube o servidor da 3038 (`kill $(cat tmp/pids/server-mod19c.pid)` no container).
- [ ] **Step 7: Gates de go-live por cidade (antes de ligar `clinical_documents`):** listas Anvisa conferidas por farmacêutico (controlado que faltar na lista passa na receita); curadoria mínima do catálogo (itens da REMUME com `parse_status: ok`); CNPJ e protocolos de enfermagem cadastrados e conferidos; norma interna da cidade para a declaração pela recepção; endereço residencial do paciente na receita (hoje linha em branco no papel — pendência para o digital).
- [ ] **Step 8: Pendências para o board de pendências de ciclo** (cards só com autorização): mutation de curadoria do catálogo (corrigir/mostrar item em revisão); endereço do paciente na receita digital (Lei 5.991 art. 35); OBM e REPM/REDFM na RNDS; antimicrobiano digital e controlados (19d, SNCR); CFO 295/2026; Atesta CFM.

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3 interruptor → Task 1; catálogo (CATMAT 6505, analisador, revisão, diferença, listas Anvisa, controlado bloqueado) → Tasks 2–7, 10; REMUME → Task 10; protocolos com versões → Task 11; CNPJ → Task 9.
- §4 documentos (cabeçalho, trigger, modo fixo, quem emite, conteúdo por tipo, emissão depois de finalizada, declaração pela recepção) → Tasks 8, 13–16.
- §5 lista de medicamentos por eventos, receita, validações, renovação → Tasks 12, 14–16.
- §6 PDF + QR, assinatura (tipos novos, canônico, PAdES, volta ao papel), página pública, limite, trilha, cancelamento → Tasks 17–21.
- §7 telas (lado api) → rotas das Tasks 9–11, 16, 17, 20, 21; maintenance → Task 7.
- §8 segurança e LGPD → Tasks 8 (cifra, filtro), 16 (trilha), 21 (página), 22 (administrativa, retenção), 23.
- §9 testes da spec → matriz (13), trava do protocolo (14), controlado (14, 23), antimicrobiano (15, 23), modo fixo (8, 18, 23), cancelamento (19, 20), página (21), eventos de medicamento (12, 15), canônico contra o vetor (18), analisador com amostras reais (3), ponta a ponta com PSC simulado (18).
- ADR 0033: invariantes → Task 23; `adr_pointers` → Task 1.

**Placeholders:** os valores das listas Anvisa são **dados** transcritos da fonte oficial na Task 5 (com formato, fonte, conferência por contagem e substâncias de referência); o nome do diretório de exemplos do `contracts` é anotado na Task 0 e usado na Task 18. Nenhum "TBD" em código.

**Consistência de nomes:** `ClinicalDocuments::{Gate,Codes,Issuers,Json,Read,Renewal,Pdf,Qr,Canonical,Verification,Page}`, `ClinicalDocuments::Content::{Input,SickNote,Declaration,ExamRequisition,Prescription}`, `ClinicalDocuments::{Issue,Cancel}`, `Patients::ApplyMedicationEvent`, `NursingProtocols::Save`, `Medications::{Catmat::{DescriptionParser,Client,FixtureClient,Row},Substances,AnvisaImport,CatalogImport,Search}`, `Signatures::CancelledDocuments`, `Cnpj` — conferidos entre as tasks.

**Review Focus:** as cinco linhas têm teste na task dona (6, 15, 21, 20, 17).

## Divergências propostas ao contrato

- **D1 — `/admin/city_profile` não existe** (o `/admin/*` do api é só leitura por regra do projeto): o CNPJ vai em **`GET/PUT /clinical_documents/city_profile`** → `{ name, cnpj }` (leitura também do profissional).
- **D2 — CNPJ alfanumérico** (IN RFB 2.229/2024, em vigor desde julho de 2026): `cnpj` = 12 caracteres `[0-9A-Z]` + 2 dígitos verificadores (o numérico é o caso particular); a entrada aceita pontuação e minúsculas; a resposta devolve sem máscara.
- **D3 — Forma de entrada do `content`** = a forma de saída do §2 com os campos calculados ignorados; item da receita com `catalog_item: { id }` **ou** `free_text`; protocolo como `nursing_protocol: { version_id }`; CID como `cid10: { code }`. Protocolo (§6) entra com `items: [ { catalog_item_id, max_dose? } ]`.
- **D4 — `max_dose` = `{ quantity, unit }`**, a quantidade máxima por item de receita; unidade diferente da do protocolo → `above_protocol_max_dose`. `current_version` ganha `version` (número).
- **D5 — Erros novos:** 503 `catalog_unavailable` também em `POST …/documents` (receita sem catálogo ou sem listas Anvisa ativas); 409 `awaiting_signature` no `GET /attendance/documents/:id/print` de documento digital ainda não assinado (Desvio 7) e 409 `already_cancelled` no impresso de cancelado; 422 `invalid_catalog_item` e `invalid_unit` na REMUME; 422 `invalid_medication` (medicamento inexistente, de outro paciente ou suspender o que não está ativo) e 409 `not_draft` em `POST …/medications`; 422 `invalid_item` com `field` além de `index`; `already_exists` (409) para protocolo com número + ano repetido; `invalid_content` com `field: "replaces_document_id"`.
- **D6 — `reason_code` novo `document_cancelled`** em `signature_requests` (contrato do 19b): o pedido pendente de documento cancelado sai da fila como `returned_to_paper` sem mudar o modo (Desvio 6). Volta ao papel de documento clínico passa `issue_mode` a `paper` (Desvio 5).
- **D7 — Declaração pela rota do atendimento sai sempre em papel** (`signature`: `null`); `author.cbo_code` nulo quando quem emite é a recepção; o `signature` do `<document>` é `null` no papel puro e o bloco do 19b quando há pedido.
- **D8 — `short_code` na API sai formatado `XXXXX-XXXXX`**; a entrada de `/v/lookup` aceita minúsculas, espaços e hífen; `birth_year` inteiro de 4 dígitos.
- **D9 — Página pública:** `GET /v` (formulário HTML) além das três rotas do §7; `GET /v/:token` sem `Accept: application/json` devolve HTML; 404 de token também em JSON `{ "error": "not_found" }`; `signed_pdf_url` só quando assinado e não cancelado; limites 10/60/30 por 10 min por IP.
- **D10 — Leitura administrativa dos documentos:** `GET /clinical_record/consultations/:id/documents` (municipal_admin, step-up, linha em `clinical_record_administrative_reads`, trilha `administrative`), sem impresso.
- **D11 — Eventos de plataforma além do §10:** `anvisa_lists.imported { release_id, antimicrobials, controlled }` e as auditorias de manutenção `maintenance.medication_catalog.imported` / `maintenance.anvisa_lists.imported`; `patient_medication.changed` aceita também `addendum_id` (ADR: consulta, adendo ou documento).
- **D12 — JSON canônico:** segue o esquema da tag (`clinical-v1.1.0`, plano do contracts-repo), que prevalece: `council` e `patient` obrigatórios; `content` = a forma do §2 (item com `catalog_item { id, catmat_code, label, active_ingredient, strength, dosage_form }` ou `free_text` texto; `catalog_release` uma vez por receita = id da release do catálogo na emissão, `null` só com texto livre; `nursing_protocol { id, title, number, year, version_id }`; horas `HH:MM` locais; opcional ausente; `note` sempre, `null` vazio). A receita **gravada e devolvida pela API** também ganha `catalog_release` (§2). Declaração da recepção não é assinada nem tem canônico.
- **D14 — RESOLVIDA pelo usuário (2026-10-10): `dosage_form` aceita `null` já na `clinical-v1.1.0`; a receita assina normalmente e o impresso usa a descrição original. Texto original:** `catalog_item.dosage_form` obrigatório no esquema, mas o CATMAT real não traz forma em boa parte dos itens** (ex.: `LOSARTANA POTÁSSICA, DOSAGEM: 50 MG`, 268856). Proposta ao contracts: `dosage_form` aceitar `null` (MINOR `clinical-v1.1.1`). Até lá o api **não assina** receita digital com item sem forma: o construtor levanta `Invalid` e o pedido fica pendente com `verification_failed` (o profissional devolve ao papel). Alternativa, se o usuário preferir: a importação manda a revisão (`hidden`) todo item sem forma — esvazia boa parte do catálogo.
- **D15 — `import_in_progress`** (M2 do plano do maintenance) nas duas importações; produção carrega o catálogo pelos rakes (`medications:import_anvisa`, `medications:import_catmat`).
- **D13 — Interruptor desligado:** cancelar continua permitido; REMUME, protocolos e CNPJ respondem atrás só do `clinical_record` (a cidade prepara antes de ligar).
