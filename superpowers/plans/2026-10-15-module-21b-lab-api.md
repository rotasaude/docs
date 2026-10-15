# Módulo 21b — Laboratório municipal: fluxo analítico (api, contracts e fake-middleware) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 21a em `origin/main` do api** (conferido ao escrever este plano em `0e81173`: `Platform::Features::CATALOG` com 14 chaves terminando em `fake_lab`; `Exams::{Transition,Open,Steps,RegisterResult,Escalate,Files,Json,Release,Access,Trail}`; `Exams::Fhir::Inbound`; `ExamOrder`/`ExamResult`/`ExamProvider`/`ExamFile`; `Citizens::RequestErasure.retained?` com `examined_patients?`; `Signatures::{OpenRequest,SignPending,Signing,Documents,Canonical,DocumentTypes,ToPaper,Json}`; `ClinicalDocuments::{Codes,Qr,Verification,Page}`; `Consultations::Print.{document,safe}`; `Mfa::Verify.totp_step_for` e `User#consume_totp_step!`; `Professionals::Create`; `lib/fake_lab/*`; `lib/exam_crew.rb`; última migração de cidade `20261013900001_add_exam_tracking.rb`; última de plataforma `20261010700001_add_controlled_prescriptions_platform.rb`; `VALID_RANGE = (1..36)`). **A Task 0 confere a base e faz as duas provas técnicas: se um nome mudou, ou se uma prova falhar, PARE e reporte.**

**Goal:** O lado api do módulo 21b (F-21.14 a F-21.24, ADR 0037): interruptores `municipal_lab`, `lab_hl7_inbound` e `fake_middleware`; catálogo-base de exames na plataforma (com o GraphQL do maintenance); catálogo do laboratório com versões imutáveis; requisição (com pedido em papel), etiqueta Code 128, coleta, remessa e recebimento com rejeição e recoleta; mapa de trabalho e digitação com referência, limites e calculados; entrada HL7 v2.5.1 e o middleware simulado; liberação com assinatura ICP (19b) ou avançada, laudo do art. 167 com conferência por QR (19c); valores críticos; retificação no laboratório e no 21a; entrega e situação; registro do GAL; relatórios; os artefatos do repo `contracts`; semente realista; rollout.

**Architecture:** O 21a continua a fonte única do exame: a requisição do laboratório junta `exam_orders`, e cada passo físico move o exame pelo mesmo `Exams::Transition` (abrir a requisição = `scheduled` no laboratório; coleta = `collected`; recoleta = `scheduled` de novo). Resultados do laboratório ficam em `lab_results` (só acréscimo, valor cifrado) e nunca saem dele antes da liberação; a liberação monta o laudo (JSON canônico `lab-report-v1` congelado e cifrado + PDF Prawn com QR) e, já liberado (avançada) ou quando o 19b assina (ICP), `Lab::FinalizeReport` grava o PDF em `exam_files` e chama `Exams::RegisterResult` com `source: "lab"` — ou, na retificação, o novo `Exams::RectifyResult`, que acrescenta um `exam_result` com `replaces_result_id` (nada é alterado: o "vigente" é o que ninguém substituiu). A entrada HL7 é uma borda fina (`Lab::Hl7::{Message,Builder,Inbound}`) sobre os mesmos comandos da digitação. Uma migração de plataforma (`20261015900001`) e uma de cidade (`20261015900002`); triggers novos em `db/city_triggers.sql`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, GraphQL Ruby (maintenance), `prawn` e `rqrcode_core` (já no Gemfile), **`barby` 0.7.0 (gem nova, só o codificador Code 128; desenho em vetor no Prawn, como o QR)**, `json_schemer` (já no Gemfile), `rotp` (já), `Net::HTTP` (fake-middleware).

**Spec:** `docs/superpowers/specs/2026-10-15-module-21b-municipal-lab-design.md` e `docs/adr/0037.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-15-module-21b-lab-contracts.md` (com a base `2026-10-13-module-21a-exams-contracts.md` §15, A1–A49); toda diferença está em "Divergências propostas ao contrato", no fim. Planos irmãos que este api serve: `2026-10-15-module-21b-lab-dashboard.md` (as formas das Divergências D1–D17 dele, adotadas aqui) e `2026-10-15-module-21b-lab-maintenance.md` (Divergências M1–M7, adotadas aqui). Pesquisa: `docs/pesquisa/2026-10-15-laboratorio-municipal-21b.md` e `docs/pesquisa/2026-10-15-laboratorio-analisadores.md`.

## Desvios e precisões

1. **Perfil profissional já existe.** `professionals` já tem `council` (inclui `CRF`, `CRBM`, `CRM`), `council_state` e `registration_number`: nada novo no perfil. O liberador precisa de perfil com um dos três conselhos; sem perfil, ou com outro conselho, 422 `council_missing`. `Professionals::Create` passa a aceitar usuário com `lab_releaser` (hoje exige `health_professional`). Na API o par sai como `council`, `council_uf`, `council_number` (contrato §3).
2. **`lab_staff` atua em todo laboratório `uses_municipal_lab` ativo da cidade**; o liberador é ligado a um laboratório (`lab_releasers`, com as marcas `technical_lead`/`technical_deputy`). `GET /lab/context` diz à tela onde a sessão atua (dashboard D1).
3. **Recoleta reabre a coleta no 21a:** `Exams::Transition::FLOW["scheduled"]` passa a aceitar `collected` (evento `scheduled` com motivo `recollection`); `Exams::Steps.schedule` (rotas da unidade e do portal) continua recusando `collected` com 409 — só o laboratório reabre.
4. **Retificação no 21a sem UPDATE:** `exam_results.replaces_result_id` (único); o vigente é o resultado que nenhum outro substitui (`ExamResult.current`); `ExamOrder#result` passa a ser o vigente e `ExamOrder#results` o histórico. O índice único de `exam_order_id` vira "um original por pedido" (`WHERE replaces_result_id IS NULL`). Evento da trilha novo `result_rectified` (`resulted|acknowledged → resulted`), publicado como `exam_order.result_rectified`.
5. **PDF do laudo em `exam_files`** (cifrado, como o 21a): `lab_reports.pdf_file_id` e uma cópia por `exam_result` (o índice único de `exam_results.pdf_file_id` fica). Nenhuma coluna `pdf` nova.
6. **ICP pelo 19b:** `LabReport` entra em `Signatures::DocumentTypes`; `signature_requests.consultation_id` vira nulo só para `LabReport` (check). O 19b exige `digital_signature` utilizável (modo `record`): sem ele, `icp` → 422 `signature_unavailable`. O pedido ICP devolvido ao papel (19b `ToPaper`) põe o laudo em `withdrawn` e os itens voltam à fila. `signature_request_id` não é guardado (sai de `SignatureRequest` por `document_type`/`document_id`, índice único).
7. **Avançada:** TOTP do liberador verificado e CONSUMIDO (`Mfa::Verify.totp_step_for` + `User#consume_totp_step!`); código errado ou repetido → 401 `mfa_required`. O PDF leva quem, conselho, data/hora e o código de conferência; o SHA-256 do PDF fica em `lab_reports.pdf_sha256` e aparece na conferência pública.
8. **HL7:** OBX-3 = `analyte_key^<nome>^99RSL` (o exame vem de OBR-4 = `instrument_test_code`); ERR-3 = código da tabela HL7 0357 e ERR-5 = o nosso código (`unknown_barcode`…). Bruto guardado uma vez por mensagem aceita em `lab_hl7_receipts.raw` (cifrado), com `lab_results.lab_hl7_receipt_id`. Só mensagem aceita (`AA`) grava recibo: `MSH-10` repetido de mensagem aceita = `AA` sem efeito; a recusada é reavaliada. Mensagem inteira ou nada (transação). `OML^O33` fora (tipo não suportado → `AR`, ERR-3 200). Token ausente ou desconhecido → HTTP 401; revogado → `AR`. Corpo até 1 MB (413).
9. **Repetição e escolha:** o valor que vale de um componente é a **última linha** dele; "escolher" (`POST /lab/results/:id/choose`) acrescenta uma cópia da linha escolhida com `chosen: true` e `chosen_from_id`. Nada é apagado.
10. **Situação da amostra** sem estado "coletada": `awaiting` com `collected_at` = coletada. Remessa só de amostra coletada; recebimento de `in_transit`, de `awaiting` coletada (posto do laboratório) e de `not_received` (chegou depois).
11. **Número e código de barras:** `lab_requisitions.number` = 8 dígitos de uma sequência da cidade (`lab_requisition_number_seq`); `lab_samples.barcode` = número + 2 dígitos (índice da amostra na requisição; a recoleta pega o próximo). Code 128 (conjunto C, 10 dígitos = 90 módulos).
12. **Laudo e itens:** `lab_report_items` liga laudo e itens (parcial e retificação). Número de registro do paciente no laudo = número da requisição.
13. **Endereço e telefone do laboratório** (art. 167, II): `exam_providers.address` e `.phone`, aceitos em `PATCH /exams/admin/providers/:id` junto com `uses_municipal_lab`.
14. **Valor SIGTAP** (relatório de produção): coluna de plataforma `sigtap_procedures.amount_sa_cents`, lida de `VL_SA` do `tb_procedimento` quando o layout a traz (nulo se não); o recorte de dev passa a trazer a coluna.
15. **GAL:** `POST /exams/orders/:id/gal_requisition { "gal_requisition" }` (unidade ou `lab_staff`), uma vez por exame (409 `gal_already_registered`); o guard de `exam_orders` aceita só essa coluna mudando de nulo para valor, sem evento. `<exam_order>.gal_requisition`.
16. **21a recusa resultado de laboratório municipal** no portal, no FHIR **e** na rota da unidade (`provider_uses_municipal_lab`); cancelar exame com item já digitado/liberado = 409 `invalid_transition`; com item pendente, o item vira `cancelled`.
17. **Catálogo-base (maintenance, M1–M7):** SDL do plano do maintenance; setores `biochemistry|hematology|immunology|urinalysis|parasitology|microbiology`; faixas com `age_min_days` e `age_max_days` obrigatórios (0..43800); sem step-up; sem exclusão (`active: false`). Componentes calculados (LDL, VLDL) só nas versões da cidade (`formula`), não na base.
18. **Formas do dashboard adotadas** (D1–D17 do plano do dashboard): `GET /lab/context`, `GET /lab/exams`, `GET /lab/entry`, `GET /lab/shipments`, `GET /lab/rejection_reasons`, `GET /lab/criticals/unnotified/count`, `PATCH /lab/admin/exams/:id`, `GET /lab/requisitions/:id/results` não existe (os valores vêm de `/lab/entry` e `/lab/releases`, que gravam trilha), e as formas de resposta listadas em cada task.

## Valores fixados por este plano

1. **Interruptores** (depois de `fake_lab` no `CATALOG`): `municipal_lab` (`requires: ["feature:exam_tracking"]`), `lab_hl7_inbound` (`["feature:municipal_lab"]`), `fake_middleware` (`["feature:lab_hl7_inbound"]`, `environments: SIMULATION_ENVS`). `missing`: `exam_tracking_disabled`, `municipal_lab_disabled`, `lab_hl7_inbound_disabled` (mecanismo genérico).
2. **Códigos:** papéis `lab_staff`, `lab_releaser` (os dois em `PRIVILEGED_ROLES`); `collection_place` `ubs|lab`; requisição `open|in_analysis|typed|released|delivered`; item `pending|typed|released|cancelled`; amostra `awaiting|in_transit|received|rejected|accepted_with_restriction|not_received`; remessa `open|sent|received`; laudo `pending_signature|released|rectified|withdrawn`; `flag` `normal|low|high|critical_low|critical_high`; `source` do resultado `typed|hl7`; assinatura `icp|advanced`; conselho `CRF|CRBM|CRM`; canal do aviso `telefone|presencial|sistema`; `value_type` `numeric|text|choice`; setor `biochemistry|hematology|immunology|urinalysis|parasitology|microbiology`; sexo `male|female` (faixa: `male|female|any`); origem do exame `lab_reception`; fonte do resultado do 21a `lab`.
3. **Limites:** `delivery_days` 1..90; até 60 componentes por versão e 20 faixas por componente; idade 0..43800 dias; `decimals` 0..4; `formula` ≤ 200 caracteres (`+ - * / ( )`, números e chaves de componentes numéricos não calculados da mesma versão); amostras por requisição ≤ 99; `restriction_note` ≤ 300; `clinical_info` ≤ 1000; `mother_name` ≤ 120; temperatura −30..60 °C; mensagem HL7 ≤ 1 MB (1 048 576 bytes); `MSH-10` ≤ 64; limite 120/min por token HL7.
4. **Etiqueta:** 50 × 25 mm (141,73 × 70,87 pt), uma por página, margem 0; módulo de barra = 3 pontos de impressora de 203 dpi (72 × 3 / 203 ≈ 1,064 pt ≈ 0,375 mm); barra 9 mm de altura; zona de silêncio de 10 módulos de cada lado; texto Helvetica 6–7 pt.
5. **Token do middleware:** `SecureRandom.urlsafe_base64(32)`; só `SHA256(token)` hex em `lab_middleware_tokens.token_digest`; um ativo por laboratório; semente de dev com `FAKE_MIDDLEWARE_TOKEN` (padrão `dev-fake-middleware-token-0123456789abcdef01`) e `simulated: true`.
6. **HL7:** versão `2.5.1`; delimitadores `|^~\&` (outros → `AR`); `MSH-9` `QBP^Q11^QBP_Q11`, `OUL^R22^OUL_R22`; respostas `RSP^K11^RSP_K11` e `ACK^R22^ACK`; `MSH-3` da resposta `ROTA-SAUDE`, `MSH-4` = slug da cidade; sistema local de códigos `99RSL`; `Content-Type` `application/hl7-v2` (resposta `application/hl7-v2; charset=utf-8`).
7. **Dev:** api do worktree na porta **3042**; Vite do dashboard **5194**; contas `tecnica-lab@curitiba.demo` (`lab_staff`), `biomedica@curitiba.demo` (`lab_releaser`, CRBM, RT) e `patologista@curitiba.demo` (`lab_releaser`, CRM, substituto, só ICP).

## Global Constraints

- Banco de cidade: `db/city_migrate/20261015900002_add_municipal_lab.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_15_900002)`; paridade em `spec/services/city_schema_spec.rb`); triggers novos em `db/city_triggers.sql` (o dump não representa trigger). Irreversível. Banco de plataforma: `db/platform_migrate/20261015900001_create_lab_exam_templates.rb` e `db/platform_schema.rb` (`db:migrate:platform`, dev e test). Rollout: publicar a imagem e rodar `bin/migrate` dela (plataforma, `platform:triggers`, `city:migrate:all`) **antes** de cortar tráfego; **nunca** migrar cidade fora do rake.
- Erros `{ "error": "<reason>" }`; interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (inclusive DELETE, corpo `{}`).
- **Valor de resultado, CPF, CNS, nome do paciente junto com exame, nome da mãe, informação clínica, PDF, mensagem HL7 e token nunca em log, evento, Analytics, mensagem de erro ou `inspect`.** O corpo `application/hl7-v2` não é tipo registrado: o Rails não o põe em `params` (nem no log); `Authorization` não é logado. Filtros novos: `/\A(mother_name|clinical_info|delivered_to|notified_person|contacts|totp|restriction_note|gal_requisition)\z/` (`values`, `data`, `reason` e `content` já estão).
- Eventos só com ids, declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`. `DomainEvents.publish(` e `Platform.audit(` são multilinha: confira cada chamada inteira. Eventos de plataforma novos `lab.template_changed` e `maintenance.lab_template.changed` entram em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`) e em `MaintenanceAudit::NAMES` (o segundo) — declare, nunca afrouxe.
- Cifra por cidade (não determinística) em `CityEncryption::CITY_KEYED_TARGETS`: `LabRequisition#mother_name`, `#clinical_info`, `#delivered_to`; `LabResult#value`; `LabHl7Receipt#raw`; `LabReport#content`, `#rectification_reason`, `#rectification_contacts`; `LabCriticalNotice#notified_person` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (só lido). `Platform::Features` engole `StatementInvalid`: interruptores (`Lab::Gate`, `Signatures::Gate`) são lidos **fora** de transação.
- Specs de request com `type: :request` explícito; helpers novos em `spec/support/lab_helpers.rb` (`RSpec.configure { |c| c.include LabHelpers }`, como `exam_helpers.rb`). `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..37)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (**nunca** `-A`). Merge, push, tag (`lab-v1.0.0` e `exams-v1.1.0` inclusive) e mudança de card só com autorização explícita do usuário — exceto mover os cards F-21.14..F-21.24 para **In Progress** no início e para **Done** ao fim de cada um, que o brief manda.

## Ambiente de execução

- **Primeira ação:** mover os cards F-21.14..F-21.24 do board #1 para **In Progress** (comandos `gh project item-edit` começando com `gh`, um por vez — memória "Board: gh em comando simples"). Ao terminar a task dona de cada F-ID, mover o card para **Done**: F-21.14 → Task 4; F-21.15 → Task 6; F-21.16 → Task 9; F-21.17 → Task 10; F-21.18 → Task 11; F-21.19 → Task 14; F-21.20 → Task 18; F-21.21 → Task 16; F-21.22 → Task 19; F-21.23 → Task 21; F-21.24 → Task 20.
- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard e do maintenance do 21b. Ordem de merge: **contracts** (tags) → **api** → dashboard → maintenance → catálogo público.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod21b -b feat/mod-21b-lab origin/main
  cp apps/api/config/master.key apps/api/.claude/mod21b/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod21b`. **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod21b -w /rails/.claude/mod21b api bundle exec rspec <arquivos>
  ```

  e `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod21b <...>`. `rails <...>` abrevia `docker compose exec -T -w /rails/.claude/mod21b api bin/rails <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod21b`), criados no começo e recriados depois da migração de cidade (Task 5):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod21b" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod21b"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod21b -w /rails/.claude/mod21b api bin/rails city:test_databases
  ```

- O banco de plataforma de teste (`rota_saude_platform_test`) é **compartilhado** entre branches: a migração de plataforma só acrescenta (tabelas novas e uma coluna nula); rode `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod21b api bin/rails db:migrate:platform` uma vez (Task 3) e avise as outras sessões.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.
- O repo `contracts` (`rota-saude/contracts`) é um clone: edite e commite ali (branch `feat/mod-21b-lab`); o `docker-compose.yml` da raiz do monorepo **não** está em git (a Task 14 edita e não commita). No fim, os bancos `_mod21b` são apagados (memória "Apagar bancos com sufixo de módulo").

### Arquivos em comum com outros módulos (rebase)

Se a main andar, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` (`define(version:)` com o maior número; se outra branch usou `20261015900002`, renumere para o próximo livre e ajuste o `define`), `db/platform_schema.rb`, `db/city_triggers.sql`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `app/services/city_encryption.rb`, `app/models/membership.rb`, `spec/adr_pointers_spec.rb` (fica `1..37`), `config/routes.rb`, `app/services/platform/features.rb`, as três specs com a lista fixa do catálogo, `spec/events/platform_event_payload_guard_spec.rb`, `app/events/maintenance_audit.rb`, `app/graphql/maintenance/{types/query_type,types/mutation_type,analyzers/human_only}.rb`, `spec/models/{create_campaigns,create_analytics}_migration_spec.rb`, `app/services/signatures/*`, `lib/sigtap_sample.rb`, `db/seeds.rb`.

## Review Focus

1. **Resultado de aparelho na amostra errada** (o middleware manda `OUL^R22` com o código de barras de amostra de outro laboratório, de amostra rejeitada, com `OBR-4` de uma versão antiga do exame, com um componente calculado, ou repete o `MSH-10` com outro conteúdo): `AE` (ou `AA` sem efeito no `MSH-10` repetido) e **nenhuma** linha em `lab_results` de nenhum item; a mensagem com um OBX ruim não grava nem os bons. Teste: Task 13 ("troca de amostra").
2. **Corrida na liberação** (dois liberadores liberam o mesmo item ao mesmo tempo; um libera enquanto a unidade cancela o exame; um libera com ICP pendente e outro tenta avançada): um só laudo cobre o item, um só `exam_result` por exame; o outro recebe 409 `invalid_transition`; exame cancelado nunca ganha resultado. Teste: Task 17 ("corrida na liberação").
3. **Referência na borda** (paciente com idade exatamente no limite da faixa; faixa só `any`; sem data de nascimento; valor igual ao `entry_max`; vírgula decimal; valor igual ao crítico): a faixa é a do sexo e da idade na **data da coleta** (limites inclusivos), sem nascimento vale só faixa sem idade restrita (0..43800), igual ao limite de digitação passa, igual ao crítico é crítico. Teste: Task 11 ("borda da referência").
4. **O que cada um vê depois da retificação** (o cidadão abre o PDF antigo pelo id; o profissional lista o exame; a ciência dada antes): o cidadão só vê o vigente (o antigo é 404), o profissional vê o vigente e o substituído marcado, a ciência anterior não vale mais (exame volta a `resulted` com prazo novo). Teste: Task 19 ("o que cada um vê depois da retificação").
5. **Etiqueta no limite** (nome de 80 caracteres com acento, sem data de nascimento, a 99ª amostra, reimpressão): o código cabe nos 50 × 25 mm com as zonas de silêncio, o texto não invade a barra, a reimpressão deixa trilha, código além de 99 amostras é recusado. Teste: Task 9 ("etiqueta no limite").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-15-laboratorio-provas-tecnicas.md` (repo docs) | registro das provas técnicas | 0 |
| `contracts/lab/{hl7-v2-profile.md,lab-report-v1.json,README.md,CHANGELOG.md}`, `contracts/lab/examples/*`, `contracts/exams/{exam-order-v1.json,CHANGELOG.md}`, `contracts/README.md` | perfil HL7, laudo, exemplos; `exam-order-v1` do 21a | 1 |
| `app/services/platform/features.rb`, `app/services/lab/gate.rb`, `spec/adr_pointers_spec.rb`, 3 specs da lista do catálogo, `spec/services/lab/gate_spec.rb` | interruptores (F-21.14) | 2 |
| `db/platform_migrate/20261015900001_create_lab_exam_templates.rb`, `db/platform_schema.rb`, `app/models/{lab_exam_template,lab_analyte_template}.rb`, `app/services/lab/templates/{ranges,save_exam,save_analyte,load}.rb`, `config/lab/catalog-base.yml`, `lib/tasks/lab.rake`, `app/services/terminology/sigtap_reader.rb`, `lib/sigtap_sample.rb`, specs | catálogo-base e valor SIGTAP (F-21.14) | 3 |
| `app/graphql/maintenance/types/{lab_exam_template,lab_analyte_template,lab_reference_range}_type.rb`, `.../inputs/lab_reference_range_input.rb`, `.../mutations/upsert_lab_{exam,analyte}_template.rb`, `query_type.rb`, `mutation_type.rb`, `analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, specs | GraphQL do catálogo-base (F-21.14) | 4 |
| `db/city_migrate/20261015900002_add_municipal_lab.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/lab_*.rb`, `app/models/{membership,exam_order,exam_order_event,exam_result,exam_provider,health_unit,document_verification_lookup}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, 2 specs de migração antigas, `spec/support/lab_helpers.rb`, `spec/models/municipal_lab_schema_spec.rb` | dados da cidade | 5 |
| `app/controllers/lab/{base_controller,context_controller,lookups_controller}.rb`, `app/controllers/lab/admin/{base_controller,templates_controller,exams_controller,rejection_reasons_controller}.rb`, `app/services/lab/{access,analytes,formula,admin_json}.rb`, `app/commands/lab/{adopt_exam,create_version,save_rejection_reason}.rb`, `app/commands/exams/save_provider.rb`, `app/controllers/health_units_controller.rb`, `app/controllers/exams/{results_controller,provider/orders_controller}.rb`, `app/services/exams/fhir/inbound.rb`, specs | catálogo do laboratório, flags de prestador e unidade, recusas do 21a (F-21.15) | 6 |
| `app/commands/lab/{assign_releaser,rotate_middleware_token}.rb`, `app/commands/professionals/create.rb`, `app/controllers/lab/admin/{releasers_controller,middleware_tokens_controller}.rb`, specs | liberadores, RT e token | 7 |
| `app/commands/lab/open_requisition.rb`, `app/services/lab/{json,requisition_status,items}.rb`, `app/controllers/lab/requisitions_controller.rb`, `app/services/exams/steps.rb`, `app/services/exams/transition.rb`, `app/controllers/exams/lookups_controller.rb`, specs | requisição (F-21.16) | 8 |
| `Gemfile`, `Gemfile.lock`, `app/services/lab/{code128,labels_pdf,receipt_pdf}.rb`, `app/commands/lab/collect_sample.rb`, `app/controllers/lab/samples_controller.rb`, specs | etiqueta, comprovante, coleta (F-21.16) | 9 |
| `app/commands/lab/{create_shipment,send_shipment,receive_shipment,receive_sample}.rb`, `app/controllers/lab/shipments_controller.rb`, `samples_controller.rb` (`receive`, `pending`), specs | remessa, recebimento, rejeição, recoleta (F-21.17) | 10 |
| `app/services/lab/{reference,values,worklist,worklist_pdf,entry_json}.rb`, `app/commands/lab/{enter_results,choose_result}.rb`, `app/controllers/lab/{worklists_controller,results_controller}.rb`, specs | mapa e digitação (F-21.18) | 11 |
| `app/services/lab/hl7/{message,builder}.rb`, `spec/fixtures/lab/*.hl7`, specs | HL7 v2.5.1: ler e escrever | 12 |
| `app/services/lab/hl7/inbound.rb`, `app/controllers/hl7/{base_controller,v2_controller,pending_barcodes_controller}.rb`, specs | entrada HL7 (F-21.19) | 13 |
| `lib/fake_middleware/{client,messages,runner,run}.rb`, `spec/lib/fake_middleware_spec.rb`, `docker-compose.yml` (raiz, fora do git) | middleware simulado (F-21.19) | 14 |
| `config/lab/lab-report-v1.json`, `app/services/lab/{report_content,report_pdf}.rb`, specs | o laudo: JSON canônico e PDF | 15 |
| `app/commands/lab/record_critical_notice.rb`, `app/controllers/lab/criticals_controller.rb`, specs | valores críticos (F-21.21) | 16 |
| `app/commands/lab/{release,finalize_report}.rb`, `app/controllers/lab/{releases_controller,reports_controller}.rb`, `app/controllers/exams/files_controller.rb`, specs | liberação e volta ao 21a (F-21.20) | 17 |
| `app/services/signatures/{document_types,documents,canonical,signing,json}.rb`, `app/commands/signatures/to_paper.rb`, `app/controllers/signatures/{base_controller,signatures_controller}.rb`, `app/services/clinical_documents/{verification,page}.rb`, `app/controllers/document_verifications_controller.rb`, specs | ICP pelo 19b e conferência pública (F-21.20) | 18 |
| `app/commands/exams/rectify_result.rb`, `app/commands/lab/rectify.rb`, `app/models/{exam_order,exam_result}.rb`, `app/services/exams/json.rb`, specs | retificação (F-21.22) | 19 |
| `app/commands/lab/deliver.rb`, `app/commands/exams/record_gal.rb`, `app/controllers/exams/gal_controller.rb`, specs | entrega, situação e GAL (F-21.24) | 20 |
| `app/services/lab/production_report.rb`, `app/controllers/lab/production_reports_controller.rb`, specs | relatórios (F-21.23) | 21 |
| `app/commands/citizens/request_erasure.rb`, `spec/invariants/municipal_lab_invariants_spec.rb`, specs | LGPD e invariantes do ADR 0037 | 22 |
| `lib/lab_crew.rb`, `db/seeds.rb`, `spec/lib/lab_crew_spec.rb` | semente de dev | 23 |
| `docs/operacao/rollout-laboratorio-municipal.md` (repo docs) | revisão, suíte, porta 3042, prova, runbook | 24 |

---

## Fatia 0 — Base e provas técnicas

### Task 0: Conferir a base e fazer as duas provas técnicas

Esta task não escreve código do api. **Regra:** (a) o perfil HL7 diverge da norma num ponto que o Desvio 8 não cobre → **PARE e reporte** (o perfil entra em contrato, código e teste nas Tasks 1, 12 e 13); (b) a etiqueta não é lida, ou não cabe em 50 × 25 mm → **PARE e reporte** (a etiqueta é requisito de 7 dos 9 editais; sem ela a F-21.16 muda).

**Files:**
- Create: `docs/pesquisa/2026-10-15-laboratorio-provas-tecnicas.md` (repo `docs`)
- Create (fora de qualquer repo, apagado no fim): `<scratchpad>/label-probe/{Gemfile,probe.rb}`

- [ ] **Step 1: Confira os nomes da base** (anote o hash de `origin/main`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api log -1 --format=%h origin/main
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/city_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/platform_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api grep -n 'Entry.new(key: "fake_lab"' origin/main -- app/services/platform/features.rb
  /opt/homebrew/bin/git -C apps/api grep -n '"scheduled" => { from:\|"resulted" => { from:' origin/main -- app/services/exams/transition.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def call(order:, source:, released_on:" origin/main -- app/commands/exams/register_result.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def examined_patients?" origin/main -- app/commands/citizens/request_erasure.rb
  /opt/homebrew/bin/git -C apps/api grep -n '"ClinicalDocument" => "clinical_document"' origin/main -- app/services/signatures/document_types.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def store(request, docs, prepared, raw, certificate, info, signer, now)" origin/main -- app/services/signatures/signing.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.totp_step_for\|def consume_totp_step!" origin/main -- app/services/mfa/verify.rb app/models/user.rb
  /opt/homebrew/bin/git -C apps/api grep -n 'has_role?("health_professional")' origin/main -- app/commands/professionals/create.rb
  /opt/homebrew/bin/git -C apps/api grep -n "VALID_RANGE" origin/main -- spec/adr_pointers_spec.rb
  /opt/homebrew/bin/git -C apps/api grep -n "barby" origin/main -- Gemfile.lock
  ls docs/adr/0037.md
  ```
  Expected: cada grep acha a linha, exceto `barby` (nenhuma — a gem entra na Task 9); última de cidade `20261013900001_add_exam_tracking.rb`, de plataforma `20261010700001_add_controlled_prescriptions_platform.rb` (se houver maior, use o próximo número livre **acima** dela e ajuste o plano inteiro; Rails 8.1 recusa migração datada > 1 dia no futuro); `VALID_RANGE = (1..36)`; o ADR 0037 existe.

- [ ] **Step 2 (prova a, HL7 v2.5.1):** confira cada linha abaixo em fonte pública e anote a URL e a data. Fontes: as definições da HL7 v2.5.1 da Caristix (`https://hl7-definition.caristix.com/v2/HL7v2.5.1/TriggerEvents/QBP_Q11`, `.../RSP_K11`, `.../OUL_R22`, `.../ACK`, `.../Segments/MSH`, `MSA`, `ERR`, `QPD`, `QAK`, `RCP`, `SPM`, `OBR`, `ORC`, `OBX`, `.../Tables/0357`, `.../Tables/0008`, `.../Tables/0208`), a norma (`https://www.hl7.org/implement/standards/product_brief.cfm?product_id=144`, capítulos 2, 5 e 7) e o perfil IHE LAW (`https://www.ihe.net/uploadedFiles/Documents/PaLM/IHE_PaLM_TF_Vol2a.pdf`, transações LAB-27/LAB-29). A conferir:
  1. `QBP^Q11^QBP_Q11` = `MSH [SFT] QPD RCP [DSC]`; `QPD-1` nome da consulta (CE), `QPD-2` etiqueta (ST), `QPD-3…` parâmetros do usuário (o código de barras em `QPD-3`).
  2. `RSP^K11^RSP_K11` = `MSH [SFT] MSA [ERR] QAK QPD <padrão de segmentos da conformidade> [DSC]`; `QAK-2` ∈ tabela 0208 (`OK`, `NF`, `AE`, `AR`).
  3. `OUL^R22^OUL_R22` = `MSH [SFT] [NTE] [PID…] [PV1…] { SPM [{OBX}] [{SAC [{OBX}]}] { OBR [ORC] [{NTE}] [{TQ1…}] [{ OBX [TCD] [{SID}] [{NTE}] }] [{CTI}] } } [DSC]`; `SPM-2` (EIP: `placer^filler`); `OBR-4` (CE) serviço; `OBX-2` tipo, `OBX-3` (CE), `OBX-5` valor, `OBX-6` unidade (CE).
  4. `ACK^R22^ACK` = `MSH [SFT] MSA [ERR]`; `MSA-1` ∈ tabela 0008 (`AA`, `AE`, `AR`), `MSA-2` = `MSH-10` da mensagem respondida.
  5. `ERR` na 2.5.1: `ERR-2` local (ERL), `ERR-3` código HL7 (CWE, tabela **0357**, obrigatório), `ERR-4` severidade (`E|W|I`, obrigatório), `ERR-5` código da aplicação (CWE). Anote que pôr o nosso código em `ERR-3` (como diz o contrato §7) contraria a tabela 0357 — é a Divergência D1.
  6. `OBX-3` é CE (`identificador^texto^sistema^…`); `instrument_test_code^analyte_key` (contrato §7) põe a chave no **texto** — é a Divergência D2 (`analyte_key^nome^99RSL`, exame em `OBR-4`).
  7. Códigos locais de sistema começam com `99` (`99zzz`): `99RSL` é válido.
  8. `MSH-12` = `2.5.1`; `MSH-9` com o terceiro componente (estrutura) obrigatório na 2.5.1.
  9. Segmento separado por `\r`; sequências de escape `\F\ \S\ \T\ \R\ \E\`.
  10. Transporte: a norma não define HTTP; MLLP é do apêndice C. Anote que a entrada por HTTPS com o corpo ER7 puro (sem o envelope MLLP) é escolha nossa, como a do IHE para HTTP (sem norma que a proíba).

  **Passa** quando 1–9 conferem; um ponto diferente do que este plano assume (fora D1/D2) → **PARE**.

- [ ] **Step 3 (prova b, etiqueta):** crie `<scratchpad>/label-probe/Gemfile`:

  ```ruby
  source "https://rubygems.org"
  gem "prawn", "2.5.0"
  gem "barby", "0.7.0"
  ```

  e `<scratchpad>/label-probe/probe.rb` (o mesmo desenho da Task 9, sem Rails):

  ```ruby
  # Prova técnica do módulo 21b: etiqueta de tubo 50 x 25 mm com Code 128 lida por leitor.
  require "prawn"
  require "barby/barcode/code_128"

  MM = 72.0 / 25.4
  WIDTH = 50 * MM
  HEIGHT = 25 * MM
  MODULE = 72.0 * 3 / 203 # 3 pontos de impressora térmica de 203 dpi
  BAR_HEIGHT = 9 * MM
  QUIET = 10

  def label(pdf, value, lines)
    bars = Barby::Code128.new(value).encoding
    width = bars.length * MODULE
    abort "não cabe: #{(width + (2 * QUIET * MODULE)) / MM} mm" if width + (2 * QUIET * MODULE) > WIDTH
    x0 = (WIDTH - width) / 2
    top = HEIGHT - (2 * MM)
    bars.scan(/1+|0+/).each_with_object([ x0 ]) do |run, x|
      pdf.fill_rectangle([ x[0], top ], run.length * MODULE, BAR_HEIGHT) if run.start_with?("1")
      x[0] += run.length * MODULE
    end
    pdf.text_box value, at: [ 0, top - BAR_HEIGHT - 1 ], width: WIDTH, height: 8, size: 7, align: :center
    lines.each_with_index do |line, i|
      pdf.text_box line, at: [ 2 * MM, top - BAR_HEIGHT - 9 - (i * 7.5) ], width: WIDTH - (4 * MM), height: 7.5, size: 6.5, overflow: :shrink_to_fit
    end
  end

  pdf = Prawn::Document.new(page_size: [ WIDTH, HEIGHT ], margin: 0)
  label(pdf, "0000012301", [ "MARIA APARECIDA DA SILVA SAUZA", "Nasc. 12/03/1961  Soro / tubo amarelo", "Req. 00000123" ])
  pdf.start_new_page
  label(pdf, "9999999999", [ "JOSÉ DE ASSUNÇÃO PEREIRA DOS SANTOS ALBUQUERQUE NETO", "Nasc. não informado  Urina / frasco", "Req. 99999999" ])
  File.binwrite(File.join(__dir__, "labels.pdf"), pdf.render)
  puts Barby::Code128.new("0000012301").encoding.length
  ```

  Rode no container do api (o scratchpad montado só para isso, sem tocar o Gemfile do api):

  ```bash
  docker compose run --rm -v "<scratchpad>/label-probe:/probe" -w /probe api sh -c 'bundle config set path /tmp/probe-gems && bundle install --quiet && bundle exec ruby probe.rb'
  ```
  Expected: imprime `90`; `labels.pdf` com 2 páginas.

- [ ] **Step 4 (prova b, leitura):** instale o leitor `zbar` no host (Homebrew: `brew install zbar` — se o comando pedir permissão, peça ao usuário) e rasterize como uma térmica de 203 dpi (monocromático) e como uma de 300 dpi:

  ```bash
  cd <scratchpad>/label-probe
  pdftoppm -r 203 -mono -png labels.pdf t203 && pdftoppm -r 300 -mono -png labels.pdf t300
  for f in t203-*.png t300-*.png; do printf "%s: " "$f"; zbarimg --quiet --raw "$f"; done
  pdfinfo labels.pdf | grep "Page size"
  ```
  Expected: `t203-1.png: 0000012301`, `t203-2.png: 9999999999` e o mesmo a 300 dpi; `Page size: 141.73 x 70.87 pts`. Abra `labels.pdf` e confira a olho que o texto não invade as barras. **Passa** com as quatro leituras certas. Sem como instalar o `zbar`, peça ao usuário que leia `t203-1.png` na tela com um leitor de código de barras (ou o app de câmera do celular) e anote o lido — **não** declare a prova sem leitura.

- [ ] **Step 5: Registre** em `docs/pesquisa/2026-10-15-laboratorio-provas-tecnicas.md` (palavras do registro, sem copiar texto da norma):

  ```markdown
  # Módulo 21b — provas técnicas (2026-10-15)

  ## (a) Perfil HL7 v2.5.1 das mensagens QBP^Q11, RSP^K11, OUL^R22 e ACK
  - Fontes (URL e data): <Caristix …>, <HL7 product brief 144>, <IHE PaLM TF Vol 2a>.
  - Estruturas conferidas: QBP_Q11, RSP_K11, OUL_R22, ACK — <confere | diferenças>.
  - ERR na 2.5.1: ERR-3 = tabela 0357 (obrigatório), ERR-5 = código da aplicação → Divergência D1.
  - OBX-3 é CE: chave do componente no identificador, exame em OBR-4 → Divergência D2.
  - Códigos locais 99zzz; MSH-12 2.5.1; MSH-9 com estrutura; escapes \F\ \S\ \T\ \R\ \E\.
  - Transporte: HTTPS com ER7 puro, sem MLLP (escolha nossa).
  - Resultado: <PASSOU | FALHOU>.

  ## (b) Etiqueta Code 128 em PDF para impressora térmica
  - barby 0.7.0 (Code 128 automático; 10 dígitos = conjunto C, 90 módulos), Prawn 2.5.0, desenho em vetor.
  - Página 50 x 25 mm (141,73 x 70,87 pt); módulo 0,375 mm (3 pontos a 203 dpi); barra 9 mm; zona de silêncio 10 módulos.
  - Leitura (zbarimg 0.23.x): 203 dpi mono <lido>; 300 dpi mono <lido>.
  - Resultado: <PASSOU | FALHOU>. Limite: leitor físico e impressora térmica da cidade são gate de go-live.
  ```

- [ ] **Step 6:** apague `<scratchpad>/label-probe/` e commite no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add pesquisa/2026-10-15-laboratorio-provas-tecnicas.md
  /opt/homebrew/bin/git -C docs commit -m "docs: record module 21b technical proofs (HL7 profile and tube label)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 7:** aplique a regra (a)/(b). Depois crie o worktree e os bancos de teste ("Ambiente de execução") e rode `rspec spec/adr_pointers_spec.rb spec/requests/exams/order_steps_spec.rb` no worktree limpo. Expected: verde.

---

## Fatia 1 — Contrato, interruptores e dados

### Task 1: Repo `contracts` — `lab/` (perfil HL7, laudo, exemplos) e `exams/exam-order-v1.json`

**Files (repo `contracts`, branch `feat/mod-21b-lab`):**
- Create: `lab/hl7-v2-profile.md`, `lab/lab-report-v1.json`, `lab/README.md`, `lab/CHANGELOG.md`
- Create: `lab/examples/{qbp-q11.hl7,rsp-k11.hl7,rsp-k11-unknown-barcode.hl7,oul-r22.hl7,ack-aa.hl7,ack-ae.hl7,ack-ar.hl7,lab-report-advanced.json,invalid-lab-report-without-lead.json,manifest.json}`
- Modify: `exams/exam-order-v1.json` (origem `lab_reception`, fonte `lab`, evento `result_rectified`, `gal_requisition`, `superseded_results`, `replaces_result_id`), `exams/CHANGELOG.md`, `README.md`

**Interfaces:**
- Produces: `rotasaude.lab_report.v1` (contrato §8) — a Task 15 copia para `apps/api/config/lab/lab-report-v1.json` e prende o SHA-256; o perfil `lab/hl7-v2-profile.md` e os exemplos `.hl7` — a Task 12 copia os exemplos para `spec/fixtures/lab/` e os lê nos testes; `exams/exam-order-v1.json` v1.1 — a Task 19 copia para `spec/fixtures/exams/exam-order-v1.json`.

- [ ] **Step 1: Branch** `/opt/homebrew/bin/git -C contracts fetch origin && /opt/homebrew/bin/git -C contracts switch -c feat/mod-21b-lab origin/main`.

- [ ] **Step 2: Escreva `lab/hl7-v2-profile.md`:**

  ````markdown
  # Perfil HL7 v2.5.1 do laboratório municipal (rotasaude.lab.hl7.v1)

  ADR 0037. Porta única para o middleware do fornecedor do reagente. Conferido
  contra a HL7 v2.5.1 e o IHE LAW em 2026-10-15 (docs/pesquisa/2026-10-15-laboratorio-provas-tecnicas.md).

  ## Transporte
  - `POST https://<cidade>/hl7/v2`, `Authorization: Bearer <token do middleware>`,
    `Content-Type: application/hl7-v2`. Corpo = UMA mensagem ER7, sem envelope MLLP,
    segmentos separados por `\r` (o Rota Saúde aceita também `\n` e `\r\n`), UTF-8,
    até 1 MB.
  - Resposta HTTP 200 com `ACK^R22^ACK` ou `RSP^K11^RSP_K11`
    (`Content-Type: application/hl7-v2; charset=utf-8`). HTTP 401 sem token ou com
    token desconhecido; 413 acima de 1 MB; 429 acima de 120 mensagens/minuto por token.
  - Delimitadores só os padrão (`MSH-1` `|`, `MSH-2` `^~\&`); `MSH-12` = `2.5.1`.
  - Códigos locais no sistema `99RSL`.

  ## Mensagens
  ### Lista de trabalho: QBP^Q11 → RSP^K11
  ```
  MSH|^~\&|<app>|<instalação>|ROTA-SAUDE|<cidade>|<AAAAMMDDhhmmss>||QBP^Q11^QBP_Q11|<id>|P|2.5.1
  QPD|LabWorklist^Lista de trabalho^99RSL|<etiqueta>|<código de barras>
  RCP|I
  ```
  Resposta com a amostra recebida:
  ```
  MSH|^~\&|ROTA-SAUDE|<cidade>|<app>|<instalação>|<agora>||RSP^K11^RSP_K11|<id>|P|2.5.1
  MSA|AA|<MSH-10 da consulta>
  QAK|<etiqueta>|OK|LabWorklist^Lista de trabalho^99RSL
  QPD|LabWorklist^Lista de trabalho^99RSL|<etiqueta>|<código de barras>
  SPM|1|<código de barras>||<material>^^99RSL|||||||||||||<coleta AAAAMMDDhhmmss>
  ORC|NW|<id do item>
  OBR|1|<id do item>||<instrument_test_code>^<nome do exame>^99RSL
  ```
  Um par `ORC`/`OBR` por exame pendente da amostra. Sem `PID`: o Rota Saúde não
  manda identificação do paciente; o código de barras é a chave.
  Recusas (`MSA-1` `AE`, `QAK-2` `AE`, com `ERR`): `unknown_barcode`, `sample_not_received`.

  ### Resultado: OUL^R22 → ACK^R22
  ```
  MSH|^~\&|<app>|<instalação>|ROTA-SAUDE|<cidade>|<agora>||OUL^R22^OUL_R22|<id>|P|2.5.1
  SPM|1|<código de barras>||<material>^^99RSL
  OBR|1|||<instrument_test_code>^<nome>^99RSL
  OBX|1|NM|<analyte_key>^<nome>^99RSL||<valor>|<unidade>^^UCUM|||||F|||<AAAAMMDDhhmmss>
  ```
  - `SPM-2.1` (ou `SPM-2.2`, se o primeiro vier vazio) = código de barras.
  - `OBR-4.1` = `instrument_test_code` da versão vigente do exame.
  - `OBX-3.1` = `analyte_key`; `OBX-2` `NM` (numérico) ou `ST`/`CWE` (texto e lista,
    `OBX-5.1`); `OBX-6.1` deve ser a unidade do componente quando vier.
  - Vários `SPM`/`OBR`/`OBX` na mesma mensagem: tudo ou nada.
  - Componente calculado não é aceito (o Rota Saúde calcula).
  - Repetição: outro `OUL^R22` com o mesmo componente acrescenta uma linha (nada é sobrescrito).
  - Nenhum resultado é liberado sozinho: entra "aguardando conferência".

  ### ACK
  ```
  MSH|^~\&|ROTA-SAUDE|<cidade>|<app>|<instalação>|<agora>||ACK^R22^ACK|<id>|P|2.5.1
  MSA|AA|<MSH-10 recebido>
  ```
  Recusa:
  ```
  MSA|AE|<MSH-10 recebido>
  ERR||SPM^1^2|204^Unknown key identifier^HL70357|E|unknown_barcode^Codigo de barras desconhecido^99RSL
  ```

  ## Códigos de recusa (ERR-5, sistema 99RSL) e ERR-3 (tabela 0357)
  | ERR-5 | MSA-1 | ERR-3 | ERR-2 |
  |---|---|---|---|
  | `unknown_barcode` | AE | 204 | `SPM^n^2` ou `QPD^1^3` |
  | `sample_not_received` | AE | 207 | `SPM^n^2` ou `QPD^1^3` |
  | `exam_not_in_version` | AE | 204 | `OBR^n^4` |
  | `unknown_analyte` | AE | 204 | `OBX^n^3` |
  | `out_of_entry_range` | AE | 207 | `OBX^n^5` |
  | `unit_mismatch` | AE | 207 | `OBX^n^6` |
  | `invalid_value` | AE | 102 | `OBX^n^5` |
  | `invalid_message` | AR | 100 | — |
  | `unsupported_message_type` | AR | 200 | `MSH^1^9` |
  | `unsupported_version` | AR | 203 | `MSH^1^12` |
  | `token_revoked` | AR | 207 | — |
  | `feature_disabled` | AR | 207 | — |

  Nunca há dado do paciente em `ERR` nem no `ACK`.

  ## Idempotência
  `MSH-10` já ACEITO (`AA`) do mesmo token → `AA` sem efeito. Mensagem recusada não
  é guardada: reenviar com o mesmo `MSH-10` é avaliado de novo.
  ````

- [ ] **Step 3: Escreva os exemplos `.hl7`** (segmentos separados por `\r` — gere com `printf`, nunca com editor que troque o fim de linha):

  ```bash
  cd contracts/lab/examples
  printf 'MSH|^~\\&|MW-SIM|LAB-MUNICIPAL|ROTA-SAUDE|curitiba|20261015143000||QBP^Q11^QBP_Q11|Q0001|P|2.5.1\rQPD|LabWorklist^Lista de trabalho^99RSL|T0001|0000012301\rRCP|I\r' > qbp-q11.hl7
  printf 'MSH|^~\\&|ROTA-SAUDE|curitiba|MW-SIM|LAB-MUNICIPAL|20261015143001||RSP^K11^RSP_K11|R0001|P|2.5.1\rMSA|AA|Q0001\rQAK|T0001|OK|LabWorklist^Lista de trabalho^99RSL\rQPD|LabWorklist^Lista de trabalho^99RSL|T0001|0000012301\rSPM|1|0000012301||Soro^^99RSL|||||||||||||20261015090000\rORC|NW|8f9a2a5e-1c2b-4f6d-9a11-2c3d4e5f6a7b\rOBR|1|8f9a2a5e-1c2b-4f6d-9a11-2c3d4e5f6a7b||GLI^Glicose^99RSL\r' > rsp-k11.hl7
  printf 'MSH|^~\\&|ROTA-SAUDE|curitiba|MW-SIM|LAB-MUNICIPAL|20261015143001||RSP^K11^RSP_K11|R0002|P|2.5.1\rMSA|AE|Q0001\rERR||QPD^1^3|204^Unknown key identifier^HL70357|E|unknown_barcode^Codigo de barras desconhecido^99RSL\rQAK|T0001|AE|LabWorklist^Lista de trabalho^99RSL\rQPD|LabWorklist^Lista de trabalho^99RSL|T0001|0000012301\r' > rsp-k11-unknown-barcode.hl7
  printf 'MSH|^~\\&|MW-SIM|LAB-MUNICIPAL|ROTA-SAUDE|curitiba|20261015150000||OUL^R22^OUL_R22|M0001|P|2.5.1\rSPM|1|0000012301||Soro^^99RSL\rOBR|1|||GLI^Glicose^99RSL\rOBX|1|NM|glicose^Glicose^99RSL||98|mg/dL^^UCUM|||||F|||20261015145500\r' > oul-r22.hl7
  printf 'MSH|^~\\&|ROTA-SAUDE|curitiba|MW-SIM|LAB-MUNICIPAL|20261015150001||ACK^R22^ACK|A0001|P|2.5.1\rMSA|AA|M0001\r' > ack-aa.hl7
  printf 'MSH|^~\\&|ROTA-SAUDE|curitiba|MW-SIM|LAB-MUNICIPAL|20261015150001||ACK^R22^ACK|A0002|P|2.5.1\rMSA|AE|M0001\rERR||OBX^1^3|204^Unknown key identifier^HL70357|E|unknown_analyte^Componente desconhecido^99RSL\r' > ack-ae.hl7
  printf 'MSH|^~\\&|ROTA-SAUDE|curitiba|MW-SIM|LAB-MUNICIPAL|20261015150001||ACK^R22^ACK|A0003|P|2.5.1\rMSA|AR|M0001\rERR||MSH^1^12|203^Unsupported version id^HL70357|E|unsupported_version^Versao nao suportada^99RSL\r' > ack-ar.hl7
  od -c oul-r22.hl7 | head -3
  ```
  Expected: `od` mostra `\r` entre os segmentos e nenhum `\n`.

- [ ] **Step 4: Escreva `lab/lab-report-v1.json`** (fechado; contrato §8):

  ```json
  {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://rota-saude/schemas/lab/lab-report-v1.json",
    "title": "rotasaude.lab_report.v1",
    "description": "Laudo do laboratório municipal (ADR 0037; contrato §8; RDC 978/2025 art. 167). JSON canônico congelado na liberação: é o que a assinatura ICP (CAdES) cobre e de onde o PDF sai. Sem CPF.",
    "type": "object",
    "additionalProperties": false,
    "required": ["schema", "laboratory", "technical_lead", "released_by", "patient", "requisition_number", "exams", "partial",
                 "issued_at", "signature_kind", "replaces_report_id", "rectification_reason", "support_laboratory"],
    "properties": {
      "schema": { "const": "rotasaude.lab_report.v1" },
      "laboratory": { "type": "object", "additionalProperties": false, "required": ["name", "cnes", "address", "phone"],
        "properties": { "name": { "type": "string", "minLength": 1 }, "cnes": { "type": "string", "pattern": "^[0-9]{7}$" },
                        "address": { "type": "string", "minLength": 1 }, "phone": { "type": "string", "pattern": "^[0-9]{10,11}$" } } },
      "technical_lead": { "$ref": "#/$defs/professional" },
      "released_by": { "$ref": "#/$defs/professional" },
      "patient": { "type": "object", "additionalProperties": false, "required": ["name", "birth_date", "biological_sex", "record_number"],
        "properties": { "name": { "type": "string", "minLength": 1 }, "birth_date": { "type": ["string", "null"], "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
                        "biological_sex": { "enum": ["male", "female"] }, "record_number": { "type": "string", "pattern": "^[0-9]{8}$" } } },
      "requisition_number": { "type": "string", "pattern": "^[0-9]{8}$" },
      "exams": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/exam" } },
      "partial": { "type": "boolean" },
      "issued_at": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$" },
      "signature_kind": { "enum": ["icp", "advanced"] },
      "replaces_report_id": { "anyOf": [{ "$ref": "#/$defs/uuid" }, { "type": "null" }] },
      "rectification_reason": { "type": ["string", "null"], "maxLength": 500 },
      "support_laboratory": { "anyOf": [{ "type": "null" }, { "type": "object", "additionalProperties": false, "required": ["name", "cnes"],
        "properties": { "name": { "type": "string" }, "cnes": { "type": "string", "pattern": "^[0-9]{7}$" } } }] }
    },
    "$defs": {
      "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
      "professional": { "type": "object", "additionalProperties": false, "required": ["name", "council", "council_uf", "council_number"],
        "properties": { "name": { "type": "string", "minLength": 1 }, "council": { "enum": ["CRF", "CRBM", "CRM"] },
                        "council_uf": { "type": "string", "pattern": "^[A-Z]{2}$" }, "council_number": { "type": "string", "pattern": "^[0-9]{1,10}$" } } },
      "exam": { "type": "object", "additionalProperties": false,
        "required": ["sigtap_code", "name", "material", "method", "version", "collected_at", "restriction_note", "components"],
        "properties": { "sigtap_code": { "type": "string", "pattern": "^02[0-9]{8}$" }, "name": { "type": "string" },
                        "material": { "type": "string" }, "method": { "type": "string" }, "version": { "type": "integer", "minimum": 1 },
                        "collected_at": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$" },
                        "restriction_note": { "type": ["string", "null"] },
                        "components": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/component" } } } },
      "component": { "type": "object", "additionalProperties": false, "required": ["key", "name", "value", "unit", "flag", "reference_text", "loinc"],
        "properties": { "key": { "type": "string" }, "name": { "type": "string" }, "value": { "type": "string" }, "unit": { "type": ["string", "null"] },
                        "flag": { "enum": ["normal", "low", "high", "critical_low", "critical_high"] },
                        "reference_text": { "type": ["string", "null"] }, "loinc": { "type": ["string", "null"] } } }
    }
  }
  ```

  (`key` no componente e `support_laboratory` são a Divergência D16: o 21a precisa da chave; o art. 129/169 pede o laboratório de apoio quando houver — sempre `null` no 21b.)

- [ ] **Step 5: Exemplos do laudo e manifesto.** `lab/examples/lab-report-advanced.json` (dados fictícios):

  ```json
  { "schema": "rotasaude.lab_report.v1",
    "laboratory": { "name": "Laboratório Municipal Central", "cnes": "2385123", "address": "Rua Doutor Faivre, 1200 — Centro — Curitiba/PR", "phone": "4133501200" },
    "technical_lead": { "name": "Mariana Albuquerque Teles", "council": "CRBM", "council_uf": "PR", "council_number": "12345" },
    "released_by": { "name": "Mariana Albuquerque Teles", "council": "CRBM", "council_uf": "PR", "council_number": "12345" },
    "patient": { "name": "Maria Aparecida da Silva", "birth_date": "1961-03-12", "biological_sex": "female", "record_number": "00000123" },
    "requisition_number": "00000123",
    "exams": [ { "sigtap_code": "0202010473", "name": "Glicose", "material": "Soro", "method": "Enzimático (GOD-PAP)", "version": 1,
                 "collected_at": "2026-10-15T12:00:00Z", "restriction_note": null,
                 "components": [ { "key": "glicose", "name": "Glicose", "value": "98", "unit": "mg/dL", "flag": "normal",
                                   "reference_text": "70 a 99 mg/dL", "loinc": "2345-7" } ] } ],
    "partial": false, "issued_at": "2026-10-15T18:30:00Z", "signature_kind": "advanced",
    "replaces_report_id": null, "rectification_reason": null, "support_laboratory": null }
  ```
  `invalid-lab-report-without-lead.json` = o mesmo sem `technical_lead`. `manifest.json`:

  ```json
  { "suites": [
    { "schema": "lab-report-v1.json",
      "cases": [ { "file": "lab-report-advanced.json", "valid": true },
                 { "file": "invalid-lab-report-without-lead.json", "valid": false, "expect": "technical_lead" } ] },
    { "profile": "hl7-v2-profile.md",
      "cases": [ { "file": "qbp-q11.hl7", "message": "QBP^Q11^QBP_Q11" }, { "file": "rsp-k11.hl7", "message": "RSP^K11^RSP_K11", "msa": "AA" },
                 { "file": "rsp-k11-unknown-barcode.hl7", "message": "RSP^K11^RSP_K11", "msa": "AE", "err5": "unknown_barcode" },
                 { "file": "oul-r22.hl7", "message": "OUL^R22^OUL_R22" }, { "file": "ack-aa.hl7", "message": "ACK^R22^ACK", "msa": "AA" },
                 { "file": "ack-ae.hl7", "message": "ACK^R22^ACK", "msa": "AE", "err5": "unknown_analyte" },
                 { "file": "ack-ar.hl7", "message": "ACK^R22^ACK", "msa": "AR", "err5": "unsupported_version" } ] } ] }
  ```

- [ ] **Step 6: `exams/exam-order-v1.json` (exams-v1.1.0).** Mude, sem tirar nada:
  - `"origin": { "enum": [..., "lab_reception"] }`;
  - em `result_administrative` e `result_clinical`: `"source": { "enum": ["unit", "provider", "fhir", "consultation", "lab"] }`;
  - em `result_clinical`, acrescente a `properties` (não a `required`) `"replaces_result_id": { "anyOf": [{ "$ref": "#/$defs/uuid" }, { "type": "null" }] }` e `"superseded_at": { "anyOf": [{ "$ref": "#/$defs/instant" }, { "type": "null" }] }`;
  - em `$defs.event.properties.kind.enum`, `"result_rectified"`;
  - em `properties`: `"gal_requisition": { "type": ["string", "null"], "maxLength": 30 }` (em `required`) e `"superseded_results": { "type": "array", "items": { "$ref": "#/$defs/result_clinical" } }` (opcional; só na visão clínica com `events`).
  E acrescente `gal_requisition: null` aos dois exemplos `exams/examples/exam-order-*.json`. Em `exams/CHANGELOG.md`: `## exams-v1.1.0 — 2026-10-15` com as cinco mudanças (ADR 0037).

- [ ] **Step 7: Valide** com o `json_schemer` do container do api:

  ```bash
  docker compose run --rm -v "$PWD/contracts:/contracts" -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
    { "lab/lab-report-v1" => "lab/examples/*report*.json", "exams/exam-order-v1" => "exams/examples/exam-order-*.json" }.each do |schema, glob|
      s = JSONSchemer.schema(JSON.parse(File.read("/contracts/#{schema}.json")))
      Dir["/contracts/#{glob}"].sort.each { |f| puts "#{File.basename(f)}: #{s.validate(JSON.parse(File.read(f))).map { |e| e["data_pointer"].presence || e.dig("details", "missing_keys").inspect }.join(",").presence || "valid"}" }
    end
    Dir["/contracts/lab/examples/*.hl7"].sort.each { |f| segs = File.binread(f).split("\r"); puts "#{File.basename(f)}: #{segs.first.split("|")[8]} #{segs.find { |x| x.start_with?("MSA") }&.split("|")&.at(1)}" }'
  ```
  Expected: `lab-report-advanced.json: valid`, `invalid-lab-report-without-lead.json: ["technical_lead"]`, `exam-order-*.json: valid`, e uma linha por `.hl7` com o tipo e o `MSA-1` do manifesto.

- [ ] **Step 8: README e CHANGELOG.** `lab/README.md`: o que é cada arquivo; laudo fechado e sem CPF; perfil HL7 com as Divergências D1/D2; idempotência; recusa tudo-ou-nada. `lab/CHANGELOG.md`: `## lab-v1.0.0 — 2026-10-15`. No `README.md` da raiz, a linha `| [`lab/`](lab/README.md) | Laudo do laboratório municipal e perfil HL7 v2.5.1 de entrada (ADR 0037) | `lab-v1.0.0` | Materializado |`, a versão do `exams/` para `exams-v1.1.0` e "Oito domínios".

- [ ] **Step 9: Commit** (sem push, sem tag):

  ```bash
  /opt/homebrew/bin/git -C contracts add lab exams/exam-order-v1.json exams/examples exams/CHANGELOG.md README.md
  /opt/homebrew/bin/git -C contracts commit -m "feat: add municipal lab report schema and HL7 v2.5.1 profile (lab-v1.0.0, exams-v1.1.0)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  As tags `lab-v1.0.0` e `exams-v1.1.0` e o push **só com autorização do usuário**, depois do merge. A Task 15 prende o SHA-256 do `lab-report-v1.json` **commitado**.

### Task 2: Interruptores (F-21.14)

**Files:**
- Modify: `app/services/platform/features.rb` (`CATALOG`)
- Create: `app/services/lab/gate.rb`, `spec/services/lab/gate_spec.rb`
- Modify: `spec/adr_pointers_spec.rb:13`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30`

**Interfaces:**
- Produces: `Lab::Gate.usable?(city)`, `Lab::Gate.hl7_usable?(city)`, `Lab::Gate.fake_middleware_usable?(city)`; constantes `Lab::Gate::KEY = "municipal_lab"`, `HL7_KEY = "lab_hl7_inbound"`, `FAKE_KEY = "fake_middleware"`.

- [ ] **Step 1: Teste que falha** — `spec/services/lab/gate_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0037: os três interruptores, em cadeia (exam_tracking → municipal_lab →
  # lab_hl7_inbound → fake_middleware), e o simulado só fora de produção.
  RSpec.describe Lab::Gate do
    let(:maintainer) { ledi_maintainer! }
    let(:city) { exam_city! }

    def set(key, on) = Platform::Features.set!(city: city, key: key, enabled: on, maintainer: maintainer)

    it "liga em cadeia e diz o que falta" do
      set("municipal_lab", true)
      expect(described_class.usable?(city)).to be(true)
      set("lab_hl7_inbound", true)
      set("fake_middleware", true)
      expect(described_class.fake_middleware_usable?(city)).to be(true)
      set("exam_tracking", false)
      expect(described_class.usable?(city)).to be(false)
      expect(Platform::Features.missing(city, "municipal_lab")).to eq([ "exam_tracking_disabled" ])
      expect(Platform::Features.missing(city, "lab_hl7_inbound")).to eq([ "municipal_lab_disabled" ])
      expect(Platform::Features.missing(city, "fake_middleware")).to eq([ "lab_hl7_inbound_disabled" ])
    end

    it "fake_middleware não existe em produção" do
      expect(Platform::Features.find("fake_middleware", env: "production")).to be_nil
      expect(described_class.fake_middleware_usable?(city, env: "production")).to be(false)
      expect { Platform::Features.set!(city: city, key: "fake_middleware", enabled: true, maintainer: maintainer, env: "production") }
        .to raise_error(Platform::Features::UnknownFeature)
    end

    it "cidade nula ou sem id nunca é utilizável" do
      expect(described_class.usable?(nil)).to be(false)
      expect(described_class.hl7_usable?(City.new)).to be(false)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/lab/gate_spec.rb` → `NameError: uninitialized constant Lab`.

- [ ] **Step 3: Implemente.** Em `app/services/platform/features.rb`, depois da entrada `fake_lab`:

  ```ruby
      # ADR 0037: laboratório municipal (módulo 21b) sobre o rastreamento do 21a.
      Entry.new(key: "municipal_lab",
                description: "Laboratório municipal: requisição, amostra, mapa, resultado, liberação e laudo",
                requires: %w[feature:exam_tracking]),
      # ADR 0037: entrada HL7 v2.5.1 (POST /hl7/v2) para o middleware do fornecedor.
      Entry.new(key: "lab_hl7_inbound",
                description: "Entrada HL7 v2.5.1 dos resultados de aparelhos (middleware do laboratório)",
                requires: %w[feature:municipal_lab]),
      # ADR 0037: middleware simulado, só fora de produção.
      Entry.new(key: "fake_middleware",
                description: "Middleware de aparelhos simulado (desenvolvimento) — resultados de exemplo, sem validade",
                requires: %w[feature:lab_hl7_inbound],
                environments: SIMULATION_ENVS)
  ```
  (a vírgula depois do `Entry.new(key: "fake_lab", …)` existente). `app/services/lab/gate.rb`:

  ```ruby
  # Interruptores do módulo 21b (ADR 0037): LIGADO E UTILIZÁVEL, relido da
  # plataforma a cada chamada. Leia FORA de transação (Platform::Features
  # engole StatementInvalid). fake_middleware só fora de produção.
  module Lab
    module Gate
      KEY = "municipal_lab".freeze
      HL7_KEY = "lab_hl7_inbound".freeze
      FAKE_KEY = "fake_middleware".freeze

      module_function

      def usable?(city, env: Rails.env) = check(city, KEY, env)
      def hl7_usable?(city, env: Rails.env) = check(city, HL7_KEY, env)
      def fake_middleware_usable?(city, env: Rails.env) = check(city, FAKE_KEY, env)

      def check(city, key, env)
        return false if city.nil? || city.id.nil?

        Platform::Features.usable?(city, key, env: env)
      end
      private_class_method :check
    end
  end
  ```
  Nas três specs da lista fixa, acrescente `municipal_lab lab_hl7_inbound fake_middleware` depois de `fake_lab`; em `spec/adr_pointers_spec.rb`, `VALID_RANGE = (1..37).freeze`.

- [ ] **Step 4: Rode:** `rspec spec/services/lab/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/platform/features.rb app/services/lab/gate.rb spec/services/lab/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb
  git commit -m "feat: add municipal lab feature switches (municipal_lab, lab_hl7_inbound, fake_middleware)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 3: Catálogo-base na plataforma e valor SIGTAP (F-21.14)

**Files:**
- Create: `db/platform_migrate/20261015900001_create_lab_exam_templates.rb`; Modify: `db/platform_schema.rb` (gerado)
- Create: `app/models/lab_exam_template.rb`, `app/models/lab_analyte_template.rb`
- Create: `app/services/lab/ranges.rb`, `app/services/lab/templates/{save_exam,save_analyte,load}.rb`, `config/lab/catalog-base.yml`, `lib/tasks/lab.rake`
- Modify: `app/services/terminology/sigtap_reader.rb` (`VL_SA`), `lib/sigtap_sample.rb` (14 códigos e as colunas de valor), `spec/support/clinical_record_helpers.rb` (`SIGTAP`), `spec/services/terminology/sigtap_reader_spec.rb` (contagem)
- Test: `spec/services/lab/ranges_spec.rb`, `spec/services/lab/templates_spec.rb`, `spec/services/terminology/sigtap_amount_spec.rb`

**Interfaces:**
- Produces: `LabExamTemplate` (`SECTORS`), `LabAnalyteTemplate` (`VALUE_TYPES`); `Lab::Ranges.parse(raw) → Result.ok(ranges: [Hash])` | `fail(:invalid_reference_ranges)`, `Lab::Ranges::MAX_AGE_DAYS = 43_800`; `Lab::Templates::SaveExam.call(attrs) → Result.ok(template:)` | `fail(code, details: { path: "<camelCase>" })`; `Lab::Templates::SaveAnalyte.call(attrs)` igual com `analyte:`; `Lab::Templates::Load.call(path = DEFAULT_PATH) → { created: Integer }`; `SigtapProcedure#amount_sa_cents`; `Lab::SIGTAP_SAMPLE` (o recorte) via `SigtapSample::PROCEDURES`.

- [ ] **Step 1: Confira o layout oficial do SIGTAP:** em `http://sigtap.datasus.gov.br/tabela-unificada/app/download.jsp` (ZIP da competência corrente) leia `tb_procedimento_layout.txt` e anote o tamanho e o tipo de `VL_SH`, `VL_SA`, `VL_SP` e um valor de exemplo de `0202020380` (hemograma). Se `VL_SA` vier em centavos sem separador (ex.: `0000000411` = R$ 4,11), siga; se tiver casas decimais explícitas ou outro nome, ajuste `amount_cents` do Step 6 ao que leu e anote no commit. Sem acesso ao ZIP, siga com centavos e registre "não conferido" na pendência do runbook (Task 24).

- [ ] **Step 2: Testes que falham** — `spec/services/lab/ranges_spec.rb`:

  ```ruby
  require "rails_helper"

  # Divergência M5 do maintenance: idade obrigatória (0..43800 dias), low/high
  # nulos = sem limite daquele lado, text para referência não numérica.
  RSpec.describe Lab::Ranges do
    def parse(*ranges) = described_class.parse(ranges)

    it "normaliza e aceita sem limite de um lado" do
      result = parse({ "sex" => "female", "age_min_days" => 0, "age_max_days" => 43_800, "low" => "12,0", "high" => nil, "text" => nil },
                     { sex: "any", age_min_days: 6570, age_max_days: 43_800, low: nil, high: nil, text: "Negativo" })
      expect(result.payload[:ranges]).to eq([
        { "sex" => "female", "age_min_days" => 0, "age_max_days" => 43_800, "low" => 12.0, "high" => nil, "text" => nil },
        { "sex" => "any", "age_min_days" => 6570, "age_max_days" => 43_800, "low" => nil, "high" => nil, "text" => "Negativo" }
      ])
    end

    it "recusa sexo, idade e limites incoerentes" do
      base = { "sex" => "any", "age_min_days" => 0, "age_max_days" => 100, "low" => 1, "high" => 2, "text" => nil }
      [ base.merge("sex" => "M"), base.merge("age_min_days" => 101), base.merge("age_max_days" => 43_801),
        base.merge("age_min_days" => nil), base.merge("low" => 3), base.merge("low" => "abc"),
        base.merge("text" => "x" * 121) ].each do |bad|
        expect(parse(bad).reason).to eq(:invalid_reference_ranges), bad.inspect
      end
      expect(described_class.parse("nope").reason).to eq(:invalid_reference_ranges)
      expect(described_class.parse(Array.new(21) { base }).reason).to eq(:invalid_reference_ranges)
    end
  end
  ```

  `spec/services/lab/templates_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §10 com M1, M3, M7: catálogo-base da plataforma; o evento
  # lab.template_changed só com ids.
  RSpec.describe "Lab::Templates" do
    before { sigtap_release! }

    let(:exam_attrs) do
      { id: nil, sigtap_code: "0202010473", name: "Glicose", sector: "biochemistry", material: "Plasma",
        container: "Tubo cinza (fluoreto)", suggested_method: "Enzimático", preparation: "Jejum de 8 horas", active: true }
    end

    it "cria, edita e recusa com o caminho do campo" do
      created = Lab::Templates::SaveExam.call(exam_attrs)
      template = created.payload[:template]
      expect(template).to have_attributes(sigtap_code: "0202010473", sector: "biochemistry", active: true)
      expect(Lab::Templates::SaveExam.call(exam_attrs.merge(id: template.id, active: false)).payload[:template].active).to be(false)
      expect(Lab::Templates::SaveExam.call(exam_attrs).then { [ _1.reason, _1.details[:path] ] }).to eq([ :duplicate_sigtap, "sigtapCode" ])
      expect(Lab::Templates::SaveExam.call(exam_attrs.merge(sigtap_code: "0202999999")).reason).to eq(:sigtap_unknown)
      expect(Lab::Templates::SaveExam.call(exam_attrs.merge(sector: "hormones")).reason).to eq(:invalid_sector)
      expect(Lab::Templates::SaveExam.call(exam_attrs.merge(name: " ")).then { [ _1.reason, _1.details[:path] ] }).to eq([ :blank, "name" ])
      expect(Lab::Templates::SaveExam.call(exam_attrs.merge(id: SecureRandom.uuid)).reason).to eq(:not_found)
      payloads = PlatformEvent.where(name: "lab.template_changed").order(:created_at).map(&:payload)
      expect(payloads.size).to eq(2)
      expect(payloads.first.keys).to contain_exactly("lab_exam_template_id", "change")
    end

    it "componente: chave, tipo, lista, LOINC e faixas" do
      template = Lab::Templates::SaveExam.call(exam_attrs).payload[:template]
      ok = { id: nil, lab_exam_template_id: template.id, key: "glicose", name: "Glicose", loinc: "2345-7", unit: "mg/dL",
             value_type: "numeric", choices: [], decimals: 0, position: 1,
             reference_ranges: [ { sex: "any", age_min_days: 0, age_max_days: 43_800, low: 70, high: 99, text: nil } ] }
      analyte = Lab::Templates::SaveAnalyte.call(ok).payload[:analyte]
      expect(analyte.reference_ranges.first).to include("low" => 70.0, "high" => 99.0)
      expect(Lab::Templates::SaveAnalyte.call(ok).reason).to eq(:duplicate_analyte_key)
      expect(Lab::Templates::SaveAnalyte.call(ok.merge(id: analyte.id, name: "Glicemia")).payload[:analyte].name).to eq("Glicemia")
      { { key: "Glicose" } => :invalid_analyte_key, { key: "x2", value_type: "date" } => :invalid_value_type,
        { key: "x3", value_type: "choice", choices: [] } => :choices_required, { key: "x4", loinc: "abc" } => :invalid_loinc,
        { key: "x5", reference_ranges: [ { sex: "any", age_min_days: 10, age_max_days: 1, low: nil, high: nil, text: nil } ] } => :invalid_reference_ranges,
        { key: "x6", name: "" } => :blank, { key: "x7", lab_exam_template_id: SecureRandom.uuid } => :not_found }.each do |over, code|
        expect(Lab::Templates::SaveAnalyte.call(ok.merge(over)).reason).to eq(code), over.inspect
      end
    end

    it "carrega o catálogo-base do YAML uma vez (idempotente, nunca sobrescreve)" do
      sigtap_release!.tap { |r| Lab::Templates::Load.sigtap_codes.each { |c| SigtapProcedure.find_or_create_by!(release: r, code: c) { |p| p.name = "EXAME #{c}" } } }
      expect(Lab::Templates::Load.call[:created]).to eq(16)
      LabExamTemplate.find_by!(sigtap_code: "0202020380").update!(name: "Hemograma (editado)")
      expect(Lab::Templates::Load.call[:created]).to eq(0)
      expect(LabExamTemplate.find_by!(sigtap_code: "0202020380").name).to eq("Hemograma (editado)")
      hb = LabAnalyteTemplate.joins(:lab_exam_template).find_by!(lab_exam_templates: { sigtap_code: "0202020380" }, key: "hemoglobina")
      expect(hb.reference_ranges.map { |r| r["sex"] }).to eq(%w[male female])
      expect(LabExamTemplate.distinct.pluck(:sector)).to all(be_in(LabExamTemplate::SECTORS))
    end
  end
  ```

  `spec/services/terminology/sigtap_amount_spec.rb`:

  ```ruby
  require "rails_helper"

  # Desvio 14: valor SIGTAP (VL_SA, centavos) lido quando o layout traz a coluna.
  RSpec.describe "SIGTAP: valor do procedimento" do
    it "importa VL_SA do recorte" do
      Dir.mktmpdir do |dir|
        result = Terminology::Import.call(kind: "sigtap", version: "202610", path: SigtapSample.write_to(dir, competence: "202610"), by: "rspec")
        expect(result).to be_ok
        release = TerminologyRelease.find_by!(kind: "sigtap", version: "202610")
        expect(SigtapProcedure.find_by!(release: release, code: "0202020380").amount_sa_cents).to eq(411)
        expect(SigtapProcedure.find_by!(release: release, code: "0301010064").amount_sa_cents).to be_nil
      end
    end
  end
  ```

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/services/lab/ranges_spec.rb spec/services/lab/templates_spec.rb spec/services/terminology/sigtap_amount_spec.rb` → `NameError` (`Lab::Ranges`), e o de SIGTAP falha por coluna inexistente.

- [ ] **Step 4: Migração de plataforma** `db/platform_migrate/20261015900001_create_lab_exam_templates.rb`:

  ```ruby
  # Catálogo-base de exames de laboratório da plataforma (ADR 0037; spec §4):
  # o maintenance cura, a cidade adota e versiona. Dado público, sem pessoa.
  # Sem exclusão (M7): sai de uso por active. E o valor SIGTAP do procedimento
  # (VL_SA, centavos) para o relatório de produção.
  class CreateLabExamTemplates < ActiveRecord::Migration[8.1]
    SECTORS = %w[biochemistry hematology immunology urinalysis parasitology microbiology].freeze

    def up
      create_table :lab_exam_templates, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
        t.string :sigtap_code, limit: 10, null: false
        t.string :name, limit: 120, null: false
        t.string :sector, null: false
        t.string :material, limit: 60, null: false
        t.string :container, limit: 60, null: false
        t.string :suggested_method, limit: 160
        t.text :preparation
        t.boolean :active, null: false, default: true
        t.timestamps
        t.index :sigtap_code, unique: true
        t.check_constraint "sigtap_code::text ~ '^02[0-9]{8}$'::text", name: "ck_lab_exam_templates_sigtap"
        t.check_constraint "sector::text = ANY (ARRAY[#{SECTORS.map { |s| "'#{s}'::text" }.join(', ')}])", name: "ck_lab_exam_templates_sector"
      end

      create_table :lab_analyte_templates, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
        t.references :lab_exam_template, type: :uuid, null: false, foreign_key: true, index: false
        t.string :key, limit: 40, null: false
        t.string :name, limit: 120, null: false
        t.string :loinc, limit: 10
        t.string :unit, limit: 30
        t.string :value_type, null: false
        t.jsonb :choices, null: false, default: []
        t.integer :decimals, null: false, default: 0
        t.jsonb :reference_ranges, null: false, default: []
        t.integer :position, null: false, default: 0
        t.timestamps
        t.index %i[lab_exam_template_id key], unique: true, name: "idx_lab_analyte_templates_key"
        t.check_constraint "value_type::text = ANY (ARRAY['numeric'::text, 'text'::text, 'choice'::text])", name: "ck_lab_analyte_templates_type"
        t.check_constraint "decimals BETWEEN 0 AND 4", name: "ck_lab_analyte_templates_decimals"
        t.check_constraint "key::text ~ '^[a-z][a-z0-9_]{0,39}$'::text", name: "ck_lab_analyte_templates_key"
      end

      add_column :sigtap_procedures, :amount_sa_cents, :integer
    end

    def down
      remove_column :sigtap_procedures, :amount_sa_cents
      drop_table :lab_analyte_templates
      drop_table :lab_exam_templates
    end
  end
  ```
  Rode `rails db:migrate:platform` e `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod21b api bin/rails db:migrate:platform`. Expected: `db/platform_schema.rb` com `define(version: 2026_10_15_900001)` e as duas tabelas; confira o diff (só acréscimos).

- [ ] **Step 5: Modelos e faixas.** `app/models/lab_exam_template.rb`:

  ```ruby
  # Exame do catálogo-base da plataforma (ADR 0037; Divergências M1, M3, M7).
  class LabExamTemplate < PlatformRecord
    SECTORS = %w[biochemistry hematology immunology urinalysis parasitology microbiology].freeze

    has_many :analytes, -> { order(:position, :key) }, class_name: "LabAnalyteTemplate", inverse_of: :lab_exam_template,
                                                        dependent: :restrict_with_error
  end
  ```
  `app/models/lab_analyte_template.rb`:

  ```ruby
  # Componente do exame do catálogo-base: chave, LOINC, unidade, tipo, faixas de partida.
  class LabAnalyteTemplate < PlatformRecord
    VALUE_TYPES = %w[numeric text choice].freeze
    KEY = /\A[a-z][a-z0-9_]{0,39}\z/
    LOINC = /\A\d{1,7}-\d\z/

    belongs_to :lab_exam_template
  end
  ```
  `app/services/lab/ranges.rb`:

  ```ruby
  # Faixas de referência (spec §4; Divergência M5): lista de
  # { sex: male|female|any, age_min_days, age_max_days (0..43800, obrigatórios),
  #   low, high (nulos = sem limite daquele lado), text }. Vírgula decimal aceita.
  module Lab
    module Ranges
      SEXES = %w[male female any].freeze
      MAX_AGE_DAYS = 43_800
      MAX = 20

      module_function

      def parse(raw)
        return fail_ranges unless raw.is_a?(Array) && raw.size <= MAX

        ranges = raw.map do |entry|
          entry = entry.respond_to?(:to_h) ? entry.to_h.transform_keys(&:to_s) : {}
          min, max = entry["age_min_days"], entry["age_max_days"]
          return fail_ranges unless SEXES.include?(entry["sex"]) && age?(min) && age?(max) && min <= max

          low, high = number(entry["low"]), number(entry["high"])
          return fail_ranges if low == :invalid || high == :invalid || (low && high && low > high)

          text = entry["text"].nil? ? nil : entry["text"].to_s.strip.presence
          return fail_ranges if text && text.length > 120

          { "sex" => entry["sex"], "age_min_days" => min, "age_max_days" => max, "low" => low, "high" => high, "text" => text }
        end
        Result.ok(ranges: ranges)
      end

      # nil, número, ou :invalid. "12,5" vale 12.5.
      def number(value)
        return nil if value.nil? || (value.is_a?(String) && value.strip.empty?)
        return value.to_f if value.is_a?(Numeric)
        return :invalid unless value.is_a?(String) && value.strip.match?(/\A-?\d+([.,]\d+)?\z/)

        value.strip.tr(",", ".").to_f
      end

      def age?(value) = value.is_a?(Integer) && value.between?(0, MAX_AGE_DAYS)
      def fail_ranges = Result.fail(:invalid_reference_ranges)
      private_class_method :age?, :fail_ranges
    end
  end
  ```

- [ ] **Step 6: Salvar e carregar.** `app/services/lab/templates/save_exam.rb`:

  ```ruby
  # upsertLabExamTemplate (Divergências M1, M3, M4, M7): id nulo cria. Erros
  # com o campo em camelCase. Só na plataforma; auditoria lab.template_changed
  # (só ids — R18).
  module Lab
    module Templates
      module SaveExam
        REQUIRED = { name: "name", material: "material", container: "container" }.freeze

        module_function

        def call(attrs)
          attrs = attrs.to_h.symbolize_keys
          template = attrs[:id] ? LabExamTemplate.find_by(id: attrs[:id]) : LabExamTemplate.new
          return fail(:not_found, "id") unless template

          REQUIRED.each { |field, path| return fail(:blank, path) if attrs[field].to_s.strip.empty? }
          return fail(:invalid_sector, "sector") unless LabExamTemplate::SECTORS.include?(attrs[:sector])

          code = attrs[:sigtap_code].to_s
          return fail(:sigtap_unknown, "sigtapCode") unless ClinicalTerms::SigtapExams.find(code, on: Time.zone.today)
          return fail(:duplicate_sigtap, "sigtapCode") if LabExamTemplate.where(sigtap_code: code).where.not(id: template.id).exists?

          created = template.new_record?
          template.update!(sigtap_code: code, name: attrs[:name].strip[0, 120], sector: attrs[:sector],
                           material: attrs[:material].strip[0, 60], container: attrs[:container].strip[0, 60],
                           suggested_method: attrs[:suggested_method].to_s.strip.presence&.slice(0, 160),
                           preparation: attrs[:preparation].to_s.strip.presence, active: attrs[:active] != false)
          Platform.audit("lab.template_changed", lab_exam_template_id: template.id,
                                                 change: created ? "exam_created" : "exam_updated")
          Result.ok(template: template)
        rescue ActiveRecord::RecordNotUnique
          fail(:duplicate_sigtap, "sigtapCode")
        end

        def fail(code, path) = Result.fail(code, details: { path: path })
        private_class_method :fail
      end
    end
  end
  ```
  `app/services/lab/templates/save_analyte.rb`:

  ```ruby
  # upsertLabAnalyteTemplate (Divergências M1, M3, M5): id nulo cria no exame.
  module Lab
    module Templates
      module SaveAnalyte
        module_function

        def call(attrs)
          attrs = attrs.to_h.symbolize_keys
          template = LabExamTemplate.find_by(id: attrs[:lab_exam_template_id])
          return fail(:not_found, "labExamTemplateId") unless template

          analyte = attrs[:id] ? template.analytes.find_by(id: attrs[:id]) : template.analytes.new
          return fail(:not_found, "id") unless analyte

          key = attrs[:key].to_s
          return fail(:invalid_analyte_key, "key") unless key.match?(LabAnalyteTemplate::KEY)
          return fail(:duplicate_analyte_key, "key") if template.analytes.where(key: key).where.not(id: analyte.id).exists?
          return fail(:blank, "name") if attrs[:name].to_s.strip.empty?
          return fail(:invalid_value_type, "valueType") unless LabAnalyteTemplate::VALUE_TYPES.include?(attrs[:value_type])

          choices = Array(attrs[:choices]).map { |c| c.to_s.strip }.reject(&:empty?).uniq
          return fail(:choices_required, "choices") if attrs[:value_type] == "choice" && choices.empty?

          loinc = attrs[:loinc].to_s.strip.presence
          return fail(:invalid_loinc, "loinc") if loinc && !loinc.match?(LabAnalyteTemplate::LOINC)
          return fail(:invalid_decimals, "decimals") unless attrs[:decimals].is_a?(Integer) && attrs[:decimals].between?(0, 4)

          ranges = Ranges.parse(Array(attrs[:reference_ranges]))
          return fail(:invalid_reference_ranges, "referenceRanges") if ranges.failure?

          created = analyte.new_record?
          analyte.update!(key: key, name: attrs[:name].strip[0, 120], loinc: loinc, unit: attrs[:unit].to_s.strip.presence&.slice(0, 30),
                          value_type: attrs[:value_type], choices: attrs[:value_type] == "choice" ? choices : [],
                          decimals: attrs[:decimals], reference_ranges: ranges.payload[:ranges], position: attrs[:position].to_i)
          Platform.audit("lab.template_changed", lab_exam_template_id: template.id,
                                                 change: created ? "analyte_created" : "analyte_updated")
          Result.ok(analyte: analyte)
        rescue ActiveRecord::RecordNotUnique
          fail(:duplicate_analyte_key, "key")
        end

        def fail(code, path) = Result.fail(code, details: { path: path })
        private_class_method :fail
      end
    end
  end
  ```
  `app/services/lab/templates/load.rb`:

  ```ruby
  # Carga inicial do catálogo-base (spec §4): exames da APS com referência de
  # PARTIDA ("ajuste ao seu método"). Idempotente: cria o que falta pelo
  # sigtap_code e nunca sobrescreve (a curadoria é do maintenance). Faixa
  # compacta no YAML: [sexo, baixo, alto] ou [sexo, baixo, alto, texto];
  # idade 0..43800. Pula (e devolve) código fora da SIGTAP ativa.
  module Lab
    module Templates
      module Load
        DEFAULT_PATH = Rails.root.join("config/lab/catalog-base.yml")

        module_function

        def entries(path = DEFAULT_PATH) = YAML.safe_load_file(path)
        def sigtap_codes(path = DEFAULT_PATH) = entries(path).map { |e| e.fetch("sigtap_code") }

        def call(path = DEFAULT_PATH)
          created = 0
          skipped = []
          entries(path).each do |entry|
            next if LabExamTemplate.exists?(sigtap_code: entry["sigtap_code"])

            PlatformRecord.transaction do
              exam = SaveExam.call(entry.slice(*%w[sigtap_code name sector material container suggested_method preparation]).merge("active" => true))
              next skipped << entry["sigtap_code"] if exam.failure?

              entry.fetch("analytes").each_with_index do |analyte, index|
                saved = SaveAnalyte.call(analyte.except("ranges").merge(
                  "lab_exam_template_id" => exam.payload[:template].id, "position" => index + 1,
                  "decimals" => analyte.fetch("decimals", 0), "choices" => analyte.fetch("choices", []),
                  "reference_ranges" => Array(analyte["ranges"]).map { |sex, low, high, text| range(sex, low, high, text) }
                ))
                raise ArgumentError, "catálogo-base: #{entry['sigtap_code']}/#{analyte['key']} #{saved.reason}" if saved.failure?
              end
              created += 1
            end
          end
          { created: created, skipped: skipped }
        end

        def range(sex, low, high, text)
          { "sex" => sex, "age_min_days" => 0, "age_max_days" => Ranges::MAX_AGE_DAYS, "low" => low, "high" => high, "text" => text }
        end
        private_class_method :range
      end
    end
  end
  ```
  `lib/tasks/lab.rake`:

  ```ruby
  namespace :lab do
    namespace :templates do
      desc "Carrega o catálogo-base de exames de laboratório (idempotente; nunca sobrescreve)."
      task load: :environment do
        result = Lab::Templates::Load.call
        puts "[lab:templates:load] criados: #{result[:created]}; fora da SIGTAP ativa: #{result[:skipped].join(', ').presence || 'nenhum'}"
      end
    end
  end
  ```

- [ ] **Step 7: O catálogo-base** `config/lab/catalog-base.yml` (16 exames; referências de partida da literatura, a ajustar ao método; LOINC só os conferidos, os demais nulos):

  ```yaml
  # Catálogo-base de exames da APS (ADR 0037; spec §4). Referências de PARTIDA:
  # o laboratório ajusta ao seu método ao adotar. Faixa: [sexo, baixo, alto, texto?].
  - sigtap_code: "0202020380"
    name: Hemograma completo
    sector: hematology
    material: Sangue total
    container: Tubo roxo (EDTA)
    suggested_method: Contagem automatizada (impedância e citometria de fluxo)
    preparation: Não exige jejum.
    analytes:
      - { key: hemacias, name: Hemácias, loinc: "789-8", unit: "10*6/uL", value_type: numeric, decimals: 2, ranges: [[male, 4.5, 5.9], [female, 4.0, 5.2]] }
      - { key: hemoglobina, name: Hemoglobina, loinc: "718-7", unit: g/dL, value_type: numeric, decimals: 1, ranges: [[male, 13.5, 17.5], [female, 12.0, 15.5]] }
      - { key: hematocrito, name: Hematócrito, loinc: "4544-3", unit: "%", value_type: numeric, decimals: 1, ranges: [[male, 41.0, 53.0], [female, 36.0, 46.0]] }
      - { key: vcm, name: VCM, loinc: "787-2", unit: fL, value_type: numeric, decimals: 1, ranges: [[any, 80.0, 100.0]] }
      - { key: hcm, name: HCM, loinc: "785-6", unit: pg, value_type: numeric, decimals: 1, ranges: [[any, 27.0, 32.0]] }
      - { key: chcm, name: CHCM, loinc: "786-4", unit: g/dL, value_type: numeric, decimals: 1, ranges: [[any, 32.0, 36.0]] }
      - { key: rdw, name: RDW, loinc: "788-0", unit: "%", value_type: numeric, decimals: 1, ranges: [[any, 11.5, 14.5]] }
      - { key: leucocitos, name: Leucócitos, loinc: "6690-2", unit: "10*3/uL", value_type: numeric, decimals: 1, ranges: [[any, 4.0, 11.0]] }
      - { key: plaquetas, name: Plaquetas, loinc: "777-3", unit: "10*3/uL", value_type: numeric, decimals: 0, ranges: [[any, 150, 450]] }
  - sigtap_code: "0202010473"
    name: Glicose
    sector: biochemistry
    material: Plasma
    container: Tubo cinza (fluoreto)
    suggested_method: Enzimático (glicose oxidase)
    preparation: Jejum de 8 horas.
    analytes:
      - { key: glicose, name: Glicose, loinc: "2345-7", unit: mg/dL, value_type: numeric, decimals: 0, ranges: [[any, 70, 99]] }
  - sigtap_code: "0202010503"
    name: Hemoglobina glicada (HbA1c)
    sector: biochemistry
    material: Sangue total
    container: Tubo roxo (EDTA)
    suggested_method: Cromatografia líquida (HPLC)
    preparation: Não exige jejum.
    analytes:
      - { key: hba1c, name: Hemoglobina glicada, loinc: "4548-4", unit: "%", value_type: numeric, decimals: 1, ranges: [[any, 4.0, 5.6]] }
  - sigtap_code: "0202010295"
    name: Colesterol total
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Enzimático colorimétrico
    preparation: Jejum não obrigatório; siga a orientação do pedido.
    analytes:
      - { key: colesterol_total, name: Colesterol total, loinc: "2093-3", unit: mg/dL, value_type: numeric, decimals: 0, ranges: [[any, null, 190, "Desejável: inferior a 190 mg/dL"]] }
  - sigtap_code: "0202010279"
    name: Colesterol HDL
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Enzimático colorimétrico direto
    preparation: Jejum não obrigatório.
    analytes:
      - { key: hdl, name: Colesterol HDL, loinc: "2085-9", unit: mg/dL, value_type: numeric, decimals: 0, ranges: [[any, 40, null, "Desejável: superior a 40 mg/dL"]] }
  - sigtap_code: "0202010287"
    name: Colesterol LDL
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Enzimático colorimétrico direto
    preparation: Jejum não obrigatório.
    analytes:
      - { key: ldl, name: Colesterol LDL, loinc: "13457-7", unit: mg/dL, value_type: numeric, decimals: 0, ranges: [[any, null, 130, "Conforme a categoria de risco"]] }
  - sigtap_code: "0202010678"
    name: Triglicerídeos
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Enzimático colorimétrico
    preparation: Jejum não obrigatório.
    analytes:
      - { key: triglicerideos, name: Triglicerídeos, loinc: "2571-8", unit: mg/dL, value_type: numeric, decimals: 0, ranges: [[any, null, 150, "Desejável: inferior a 150 mg/dL"]] }
  - sigtap_code: "0202010317"
    name: Creatinina
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Jaffé cinético
    preparation: Não exige jejum.
    analytes:
      - { key: creatinina, name: Creatinina, loinc: "2160-0", unit: mg/dL, value_type: numeric, decimals: 2, ranges: [[male, 0.70, 1.30], [female, 0.60, 1.10]] }
  - sigtap_code: "0202010694"
    name: Ureia
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Enzimático (urease)
    preparation: Não exige jejum.
    analytes:
      - { key: ureia, name: Ureia, loinc: "3091-6", unit: mg/dL, value_type: numeric, decimals: 0, ranges: [[any, 15, 45]] }
  - sigtap_code: "0202010643"
    name: TGO (AST)
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Cinético UV (IFCC)
    preparation: Não exige jejum.
    analytes:
      - { key: tgo, name: TGO (AST), loinc: "1920-8", unit: U/L, value_type: numeric, decimals: 0, ranges: [[any, 5, 40]] }
  - sigtap_code: "0202010651"
    name: TGP (ALT)
    sector: biochemistry
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Cinético UV (IFCC)
    preparation: Não exige jejum.
    analytes:
      - { key: tgp, name: TGP (ALT), loinc: "1742-6", unit: U/L, value_type: numeric, decimals: 0, ranges: [[any, 7, 56]] }
  - sigtap_code: "0202060250"
    name: TSH
    sector: immunology
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Quimioluminescência
    preparation: Não exige jejum.
    analytes:
      - { key: tsh, name: TSH, loinc: "3016-3", unit: "m[IU]/L", value_type: numeric, decimals: 2, ranges: [[any, 0.40, 4.00]] }
  - sigtap_code: "0202060381"
    name: T4 livre
    sector: immunology
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Quimioluminescência
    preparation: Não exige jejum.
    analytes:
      - { key: t4_livre, name: T4 livre, loinc: "3024-7", unit: ng/dL, value_type: numeric, decimals: 2, ranges: [[any, 0.80, 1.80]] }
  - sigtap_code: "0202060217"
    name: Beta-hCG
    sector: immunology
    material: Soro
    container: Tubo amarelo (gel)
    suggested_method: Quimioluminescência
    preparation: Não exige jejum.
    analytes:
      - { key: beta_hcg, name: Beta-hCG, loinc: "21198-7", unit: "m[IU]/mL", value_type: numeric, decimals: 1, ranges: [[any, null, 5.0, "Não gestante: inferior a 5 mUI/mL"]] }
  - sigtap_code: "0202050017"
    name: Urina tipo 1
    sector: urinalysis
    material: Urina
    container: Frasco coletor estéril
    suggested_method: Tira reativa e sedimentoscopia
    preparation: Primeira urina da manhã, jato médio, após higiene.
    analytes:
      - { key: densidade, name: Densidade, loinc: "5811-5", unit: null, value_type: numeric, decimals: 3, ranges: [[any, 1.005, 1.030]] }
      - { key: ph, name: pH, loinc: "5803-2", unit: null, value_type: numeric, decimals: 1, ranges: [[any, 5.0, 8.0]] }
      - { key: proteinas, name: Proteínas, loinc: null, unit: null, value_type: choice, choices: [Negativo, Traços, "+", "++", "+++"], ranges: [[any, null, null, Negativo]] }
      - { key: glicose_urina, name: Glicose, loinc: null, unit: null, value_type: choice, choices: [Negativo, Traços, "+", "++", "+++"], ranges: [[any, null, null, Negativo]] }
      - { key: nitrito, name: Nitrito, loinc: null, unit: null, value_type: choice, choices: [Negativo, Positivo], ranges: [[any, null, null, Negativo]] }
      - { key: leucocitos_sedimento, name: Leucócitos (sedimento), loinc: null, unit: por campo, value_type: text, ranges: [[any, null, null, "Até 5 por campo"]] }
      - { key: hemacias_sedimento, name: Hemácias (sedimento), loinc: null, unit: por campo, value_type: text, ranges: [[any, null, null, "Até 3 por campo"]] }
  - sigtap_code: "0202040127"
    name: Parasitológico de fezes
    sector: parasitology
    material: Fezes
    container: Coletor universal
    suggested_method: Sedimentação espontânea (Hoffman)
    preparation: Amostra recente, sem laxante.
    analytes:
      - { key: resultado, name: Resultado, loinc: null, unit: null, value_type: text, ranges: [[any, null, null, Negativo]] }
  ```
  Confira os 16 códigos e nomes no SIGTAP real (a mesma consulta do Step 1, `tb_procedimento`): código que não bater sai do YAML e da lista do Step 8 (anote no commit).

- [ ] **Step 8: SIGTAP.** Em `app/services/terminology/sigtap_reader.rb#procedure_row`, acrescente ao hash `amount_sa_cents: amount_cents(r["VL_SA"], code)` e:

  ```ruby
      # Desvio 14: VL_SA em centavos (sem separador) quando o layout traz a coluna.
      def amount_cents(value, code)
        return nil if value.nil? || value.strip.empty?

        Integer(value.strip, 10).then { |cents| cents.positive? ? cents : nil }
      rescue ArgumentError
        raise Import::Invalid, "valor inválido em #{code}: #{value.inspect}"
      end
  ```
  (em `private_class_method`/`private` como os vizinhos; confira se o leitor usa `def` de instância — use o mesmo estilo de `age`). Em `lib/sigtap_sample.rb`: no layout `tb_procedimento`, entre `VL_IDADE_MAXIMA` e `DT_COMPETENCIA`, `[ "VL_SH", 12, "NUMBER" ], [ "VL_SA", 12, "NUMBER" ], [ "VL_SP", 12, "NUMBER" ]`; em `rows_for("tb_procedimento")`, `[ code, name, cx, sex, 9999, 0, 0, min, max, 0, AMOUNTS.fetch(code, 0), 0, competence ]`; acrescente a `PROCEDURES` (complexidade "2", sexo "I", 0..9999):

  ```ruby
    [ "0202010473", "DOSAGEM DE GLICOSE", "2", "I", 0, 9999 ],
    [ "0202010317", "DOSAGEM DE CREATININA", "2", "I", 0, 9999 ],
    [ "0202010295", "DOSAGEM DE COLESTEROL TOTAL", "2", "I", 0, 9999 ],
    [ "0202010279", "DOSAGEM DE COLESTEROL HDL", "2", "I", 0, 9999 ],
    [ "0202010287", "DOSAGEM DE COLESTEROL LDL", "2", "I", 0, 9999 ],
    [ "0202010678", "DOSAGEM DE TRIGLICERIDEOS", "2", "I", 0, 9999 ],
    [ "0202010694", "DOSAGEM DE UREIA", "2", "I", 0, 9999 ],
    [ "0202010643", "DOSAGEM DE TRANSAMINASE GLUTAMICO-OXALACETICA (TGO)", "2", "I", 0, 9999 ],
    [ "0202010651", "DOSAGEM DE TRANSAMINASE GLUTAMICO-PIRUVICA (TGP)", "2", "I", 0, 9999 ],
    [ "0202060250", "DOSAGEM DE HORMONIO TIREOESTIMULANTE (TSH)", "2", "I", 0, 9999 ],
    [ "0202060381", "DOSAGEM DE TIROXINA LIVRE (T4 LIVRE)", "2", "I", 0, 9999 ],
    [ "0202060217", "DOSAGEM DE GONADOTROFINA CORIONICA HUMANA (HCG, BETA HCG)", "2", "I", 0, 9999 ],
    [ "0202050017", "ANALISE DE CARACTERES FISICOS, ELEMENTOS E SEDIMENTO DA URINA", "2", "I", 0, 9999 ],
    [ "0202040127", "PESQUISA DE OVOS E CISTOS DE PARASITAS", "2", "I", 0, 9999 ]
  ```
  e o valor de dev (aproximado; a verdade é a competência importada):

  ```ruby
    # Desvio 14: VL_SA de dev, em centavos (aproximação da tabela vigente).
    AMOUNTS = { "0202020380" => 411, "0202010473" => 185, "0202010503" => 763, "0202010295" => 185, "0202010279" => 351,
                "0202010287" => 351, "0202010678" => 351, "0202010317" => 185, "0202010694" => 185, "0202010643" => 201,
                "0202010651" => 201, "0202060250" => 896, "0202060381" => 1152, "0202060217" => 785, "0202050017" => 370,
                "0202040127" => 165 }.freeze
  ```
  Em `spec/support/clinical_record_helpers.rb`, acrescente os 14 códigos novos ao `SIGTAP` (mesmos nomes); em `spec/services/terminology/sigtap_reader_spec.rb:20`, `sigtap_procedures: 23`.

- [ ] **Step 9: Rode:** `rspec spec/services/lab/ranges_spec.rb spec/services/lab/templates_spec.rb spec/services/terminology` → verde.

- [ ] **Step 10: Commit:**

  ```bash
  git add db/platform_migrate/20261015900001_create_lab_exam_templates.rb db/platform_schema.rb app/models/lab_exam_template.rb app/models/lab_analyte_template.rb app/services/lab/ranges.rb app/services/lab/templates config/lab/catalog-base.yml lib/tasks/lab.rake app/services/terminology/sigtap_reader.rb lib/sigtap_sample.rb spec/support/clinical_record_helpers.rb spec/services/terminology spec/services/lab/ranges_spec.rb spec/services/lab/templates_spec.rb
  git commit -m "feat: add platform lab exam templates and SIGTAP procedure amount

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 4: GraphQL do catálogo-base no maintenance (F-21.14)

**Files:**
- Create: `app/graphql/maintenance/types/{lab_exam_template_type,lab_analyte_template_type,lab_reference_range_type,lab_reference_range_input}.rb`
- Create: `app/graphql/maintenance/mutations/{upsert_lab_exam_template,upsert_lab_analyte_template}.rb`
- Modify: `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`
- Test: `spec/requests/maintenance/lab_templates_spec.rb`

**Interfaces:**
- Consumes: `Lab::Templates::SaveExam`, `SaveAnalyte` (Task 3).
- Produces (SDL **idêntico** à Divergência M1 do plano do maintenance): `type LabExamTemplate { active: Boolean! analytes: [LabAnalyteTemplate!]! container: String! id: ID! material: String! name: String! preparation: String sector: String! sigtapCode: String! suggestedMethod: String }`; `type LabAnalyteTemplate { choices: [String!]! decimals: Int! id: ID! key: String! loinc: String name: String! position: Int! referenceRanges: [LabReferenceRange!]! unit: String valueType: String! }`; `type LabReferenceRange { ageMaxDays: Int! ageMinDays: Int! high: Float low: Float sex: String! text: String }`; `input LabReferenceRangeInput { ageMaxDays: Int! ageMinDays: Int! high: Float low: Float sex: String! text: String }`; `UpsertLabExamTemplatePayload`/`UpsertLabAnalyteTemplatePayload { errors: [UserError!]! ok: Boolean! }`; `labExamTemplates: [LabExamTemplate!]!` (ativos e inativos, por nome); `upsertLabExamTemplate(active: Boolean!, container: String!, id: ID, material: String!, name: String!, preparation: String, sector: String!, sigtapCode: String!, suggestedMethod: String)`; `upsertLabAnalyteTemplate(choices: [String!] = [], decimals: Int!, id: ID, key: String!, labExamTemplateId: ID!, loinc: String, name: String!, position: Int!, referenceRanges: [LabReferenceRangeInput!]!, unit: String, valueType: String!)`. Sem `code` (M4). Auditoria `maintenance.lab_template.changed` (tentativa/resultado).

- [ ] **Step 1: Teste que falha** — `spec/requests/maintenance/lab_templates_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §10 com M1 (SDL), M3 (erros), M4 (sem step-up), M7 (sem exclusão).
  RSpec.describe "Maintenance: catálogo-base do laboratório", type: :request do
    let(:frontend) { "https://maintenance.rotasaude.app" }
    let(:password) { "s3nha-forte-1" }
    let!(:maintainer) do
      Maintainer.create!(email_address: "lab-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    end

    def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
    def json = JSON.parse(response.body)
    def gql!(query, variables = {}) = post("/graphql", params: { query: query, variables: variables.to_json }, headers: browser)

    EXAM = <<~GQL.freeze
      mutation UpsertLabExamTemplate($id: ID, $sigtapCode: String!, $name: String!, $sector: String!, $material: String!, $container: String!, $active: Boolean!) {
        upsertLabExamTemplate(id: $id, sigtapCode: $sigtapCode, name: $name, sector: $sector, material: $material, container: $container,
                              suggestedMethod: "Enzimático", preparation: null, active: $active) { ok errors { path message } }
      }
    GQL
    ANALYTE = <<~GQL.freeze
      mutation UpsertLabAnalyteTemplate($templateId: ID!, $key: String!, $ranges: [LabReferenceRangeInput!]!) {
        upsertLabAnalyteTemplate(labExamTemplateId: $templateId, key: $key, name: "Glicose", loinc: "2345-7", unit: "mg/dL",
                                 valueType: "numeric", decimals: 0, position: 1, referenceRanges: $ranges) { ok errors { path message } }
      }
    GQL
    LIST = "query LabExamTemplates { labExamTemplates { id sigtapCode name sector active analytes { key valueType choices referenceRanges { sex ageMinDays ageMaxDays low high text } } } }".freeze

    before do
      sigtap_release!
      host! "maintenance-api.rotasaude.app"
      allow(ENV).to receive(:[]).and_call_original
      allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
      post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
      post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now }, headers: browser
    end

    it "cria exame e componente, lista com faixas, recusa com path, desativa; audita sem abrir cidade" do
      expect(Maintenance::CityReader).not_to receive(:call)
      vars = { sigtapCode: "0202010473", name: "Glicose", sector: "biochemistry", material: "Plasma", container: "Tubo cinza", active: true }
      gql!(EXAM, vars)
      expect(json.dig("data", "upsertLabExamTemplate")).to eq("ok" => true, "errors" => [])
      template = LabExamTemplate.find_by!(sigtap_code: "0202010473")
      ranges = [ { sex: "any", ageMinDays: 0, ageMaxDays: 43_800, low: 70.0, high: 99.0, text: nil } ]
      gql!(ANALYTE, templateId: template.id, key: "glicose", ranges: ranges)
      expect(json.dig("data", "upsertLabAnalyteTemplate", "ok")).to be(true)
      gql!(EXAM, vars.merge(sector: "hormones"))
      expect(json.dig("data", "upsertLabExamTemplate")).to eq("ok" => false, "errors" => [ { "path" => "sector", "message" => "invalid_sector" } ])
      gql!(ANALYTE, templateId: template.id, key: "Bad Key", ranges: ranges)
      expect(json.dig("data", "upsertLabAnalyteTemplate", "errors")).to eq([ { "path" => "key", "message" => "invalid_analyte_key" } ])
      gql!(EXAM, vars.merge(id: template.id, active: false))
      gql!(LIST)
      expect(json.dig("data", "labExamTemplates")).to eq([ {
        "id" => template.id, "sigtapCode" => "0202010473", "name" => "Glicose", "sector" => "biochemistry", "active" => false,
        "analytes" => [ { "key" => "glicose", "valueType" => "numeric", "choices" => [],
                          "referenceRanges" => [ { "sex" => "any", "ageMinDays" => 0, "ageMaxDays" => 43_800, "low" => 70.0, "high" => 99.0, "text" => nil } ] } ] } ])
      outcomes = PlatformEvent.where(name: "maintenance.lab_template.changed").pluck(Arel.sql("payload->>'outcome'"))
      expect(outcomes).to include("attempted", "ok", "rejected")
    end

    it "token de serviço não alcança os três campos" do
      _record, secret = MaintenanceToken.issue!(maintainer: maintainer, name: "ci", access: "read_write", city_slugs: [], expires_at: 10.days.from_now)
      [ LIST, "mutation { upsertLabExamTemplate(sigtapCode: \"0202010473\", name: \"x\", sector: \"biochemistry\", material: \"x\", container: \"x\", active: true) { ok } }" ].each do |query|
        post "/graphql", params: { query: query, variables: "{}" }, headers: { "Authorization" => "Bearer #{secret}", "Cookie" => "" }
        expect(json["errors"]).to be_present
      end
      expect(LabExamTemplate.count).to eq(0)
    end

    it "o SDL é o da Divergência M1" do
      sdl = Maintenance::Schema.to_definition
      expect(sdl).to include("upsertLabAnalyteTemplate(choices: [String!] = [], decimals: Int!, id: ID, key: String!, labExamTemplateId: ID!, loinc: String, name: String!, position: Int!, referenceRanges: [LabReferenceRangeInput!]!, unit: String, valueType: String!): UpsertLabAnalyteTemplatePayload")
      expect(sdl).to include("upsertLabExamTemplate(active: Boolean!, container: String!, id: ID, material: String!, name: String!, preparation: String, sector: String!, sigtapCode: String!, suggestedMethod: String): UpsertLabExamTemplatePayload")
      expect(sdl).to include("labExamTemplates: [LabExamTemplate!]!")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/maintenance/lab_templates_spec.rb` → `Field 'upsertLabExamTemplate' doesn't exist`.

- [ ] **Step 3: Tipos.** `app/graphql/maintenance/types/lab_reference_range_type.rb`:

  ```ruby
  module Maintenance
    module Types
      class LabReferenceRangeType < BaseObject
        graphql_name "LabReferenceRange"
        description "Faixa de referência de partida (Divergência M5)"

        field :sex, String, null: false
        field :age_min_days, Int, null: false
        field :age_max_days, Int, null: false
        field :low, Float, null: true
        field :high, Float, null: true
        field :text, String, null: true

        def sex = object["sex"]
        def age_min_days = object["age_min_days"]
        def age_max_days = object["age_max_days"]
        def low = object["low"]
        def high = object["high"]
        def text = object["text"]
      end
    end
  end
  ```
  `app/graphql/maintenance/types/lab_reference_range_input.rb`:

  ```ruby
  module Maintenance
    module Types
      class LabReferenceRangeInput < GraphQL::Schema::InputObject
        graphql_name "LabReferenceRangeInput"

        argument :sex, String, required: true
        argument :age_min_days, Int, required: true
        argument :age_max_days, Int, required: true
        argument :low, Float, required: false
        argument :high, Float, required: false
        argument :text, String, required: false
      end
    end
  end
  ```
  `app/graphql/maintenance/types/lab_analyte_template_type.rb`:

  ```ruby
  module Maintenance
    module Types
      class LabAnalyteTemplateType < BaseObject
        graphql_name "LabAnalyteTemplate"

        field :id, ID, null: false
        field :key, String, null: false
        field :name, String, null: false
        field :loinc, String, null: true
        field :unit, String, null: true
        field :value_type, String, null: false
        field :choices, [ String ], null: false
        field :decimals, Int, null: false
        field :position, Int, null: false
        field :reference_ranges, [ Types::LabReferenceRangeType ], null: false
      end
    end
  end
  ```
  `app/graphql/maintenance/types/lab_exam_template_type.rb`:

  ```ruby
  module Maintenance
    module Types
      class LabExamTemplateType < BaseObject
        graphql_name "LabExamTemplate"
        description "Exame do catálogo-base (plataforma). A cidade adota e versiona; mudar aqui só vale na próxima adoção."

        field :id, ID, null: false
        field :sigtap_code, String, null: false
        field :name, String, null: false
        field :sector, String, null: false
        field :material, String, null: false
        field :container, String, null: false
        field :suggested_method, String, null: true
        field :preparation, String, null: true
        field :active, Boolean, null: false
        field :analytes, [ Types::LabAnalyteTemplateType ], null: false, description: "Em ordem de position"
      end
    end
  end
  ```
  (Os campos ficam na ordem alfabética do SDL ao imprimir — o `to_definition` ordena; confira no teste.)

- [ ] **Step 4: Mutations.** `app/graphql/maintenance/mutations/upsert_lab_exam_template.rb`:

  ```ruby
  module Maintenance
    module Mutations
      # Contrato §10 (M1, M3, M4, M7): cria (id nulo) ou altera um exame do
      # catálogo-base. Só plataforma; sem step-up; auditado pela base.
      class UpsertLabExamTemplate < BaseMutation
        description "Cria ou altera um exame do catálogo-base do laboratório."

        argument :id, ID, required: false
        argument :sigtap_code, String, required: true
        argument :name, String, required: true
        argument :sector, String, required: true
        argument :material, String, required: true
        argument :container, String, required: true
        argument :suggested_method, String, required: false
        argument :preparation, String, required: false
        argument :active, Boolean, required: true

        field :ok, Boolean, null: false
        field :errors, [ Types::UserErrorType ], null: false

        def resolve(**attrs)
          audited(event: "maintenance.lab_template.changed", module_name: "lab", sigtap_code: attrs[:sigtap_code]) do
            result = Lab::Templates::SaveExam.call(attrs)
            raise Rejected.new(result.reason.to_s, path: result.details[:path]) if result.failure?
          end
        end
      end
    end
  end
  ```
  `app/graphql/maintenance/mutations/upsert_lab_analyte_template.rb`:

  ```ruby
  module Maintenance
    module Mutations
      class UpsertLabAnalyteTemplate < BaseMutation
        description "Cria ou altera um componente de exame do catálogo-base."

        argument :id, ID, required: false
        argument :lab_exam_template_id, ID, required: true
        argument :key, String, required: true
        argument :name, String, required: true
        argument :loinc, String, required: false
        argument :unit, String, required: false
        argument :value_type, String, required: true
        argument :choices, [ String ], required: false, default_value: []
        argument :decimals, Int, required: true
        argument :position, Int, required: true
        argument :reference_ranges, [ Types::LabReferenceRangeInput ], required: true

        field :ok, Boolean, null: false
        field :errors, [ Types::UserErrorType ], null: false

        def resolve(**attrs)
          attrs[:reference_ranges] = attrs[:reference_ranges].map(&:to_h)
          audited(event: "maintenance.lab_template.changed", module_name: "lab", lab_exam_template_id: attrs[:lab_exam_template_id]) do
            result = Lab::Templates::SaveAnalyte.call(attrs)
            raise Rejected.new(result.reason.to_s, path: result.details[:path]) if result.failure?
          end
        end
      end
    end
  end
  ```
  Em `mutation_type.rb`: `field :upsert_lab_exam_template, mutation: Mutations::UpsertLabExamTemplate` e `field :upsert_lab_analyte_template, mutation: Mutations::UpsertLabAnalyteTemplate`. Em `query_type.rb`:

  ```ruby
        # ADR 0037 (contrato §10, M1): catálogo-base do laboratório, plataforma.
        field :lab_exam_templates, [ Types::LabExamTemplateType ], null: false,
              description: "Catálogo-base de exames de laboratório (ativos e inativos, por nome)"
  ```
  e `def lab_exam_templates = LabExamTemplate.includes(:analytes).order(:name)`. Em `analyzers/human_only.rb`, acrescente `labExamTemplates upsertLabExamTemplate upsertLabAnalyteTemplate` a `RESTRICTED`. Em `app/events/maintenance_audit.rb`, `maintenance.lab_template.changed` em `NAMES` e o ramo `when "maintenance.lab_template.changed" then Platform.audit("maintenance.lab_template.changed", **payload)`. Em `spec/events/platform_event_payload_guard_spec.rb`, acrescente `lab.template_changed maintenance.lab_template.changed` a `R18_PLATFORM_EVENT_NAMES`.

- [ ] **Step 5: Rode:** `rspec spec/requests/maintenance/lab_templates_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/graphql spec/requests/maintenance/graphql_spec.rb` → verde (a guarda de cobertura do `HumanOnly` passa). Gere o SDL e compare **linha a linha** com o bloco da Divergência M1 do plano do maintenance:

  ```bash
  rails runner 'puts Maintenance::Schema.to_definition' | grep -n -E 'labExamTemplates|upsertLab|^type Lab|^input Lab|^type UpsertLab' 
  ```
  Expected: os mesmos nomes, nulidades e argumentos. Diferença → corrija aqui (o maintenance espera o SDL dele).

- [ ] **Step 6: Commit** e mova o card **F-21.14** para **Done**:

  ```bash
  git add app/graphql/maintenance app/events/maintenance_audit.rb spec/events/platform_event_payload_guard_spec.rb spec/requests/maintenance/lab_templates_spec.rb
  git commit -m "feat: expose lab exam templates in the maintenance GraphQL API

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 5: Migração de cidade, triggers, modelos, papéis, eventos e filtros

**Files:**
- Create: `db/city_migrate/20261015900002_add_municipal_lab.rb`
- Modify: `db/city_schema.rb` (versão, `memberships`, `exam_providers`, `health_units`, `exam_orders`, `exam_order_events`, `exam_results`, `signature_requests`, `signatures`, `document_verification_lookups`, as tabelas `lab_*` e FKs), `db/city_triggers.sql`
- Create: `app/models/{lab_exam,lab_exam_version,lab_rejection_reason,lab_releaser,lab_middleware_token,lab_requisition,lab_requisition_item,lab_sample,lab_shipment,lab_result,lab_hl7_receipt,lab_report,lab_report_item,lab_critical_notice}.rb`
- Modify: `app/models/{membership,exam_order,exam_order_event,exam_provider,document_verification_lookup}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/models/{create_campaigns,create_analytics}_migration_spec.rb` (lista de papéis restaurada)
- Create: `spec/support/lab_helpers.rb`, `spec/models/municipal_lab_schema_spec.rb`

**Interfaces:**
- Produces: as tabelas e modelos acima (colunas no Step 3); `LabRequisition.next_number → "00000001"`; `LabSample.barcode_for(number, index) → "0000000101"`; `LabReport#author_user_id` (= `released_by_user_id`), `#consultation_id` (nil), `#content_data` (Hash), `#verification_url`; `LabExamVersion#analyte(key)`, `#entered_analytes` (sem `formula`); `LabMiddlewareToken.by_token(token)`, `.digest(token)`; `LabReleaser.active`; `ExamOrder::ORIGINS` com `lab_reception`; `ExamOrderEvent::KINDS` com `result_rectified`; `Membership::ROLES`/`PRIVILEGED_ROLES` com `lab_staff` e `lab_releaser`; `AddMunicipalLab::ROLES`. Helpers `LabHelpers#lab_city!(hl7: false, fake_middleware: false)`, `#lab_provider!(name:)`, `#lab_exam!(provider:, sigtap_code:, analytes:, material:, container:, instrument_test_code:, sector:, delivery_days:)`, `#lab_staff!`, `#lab_releaser!(provider, council:, lead:, deputy:)`, `#json_patch(path, params)`, `GLUCOSE`, `HEMOGRAM` (listas de componentes de teste).

- [ ] **Step 1: Teste que falha** — `spec/models/municipal_lab_schema_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0037 (Invariantes): versão, resultados, laudos e itens do laudo nunca
  # mudam nem somem; o laudo só anda pending_signature → released|withdrawn e
  # released → rectified; valores cifrados; um exame num item só; papéis novos.
  RSpec.describe "Banco do laboratório municipal" do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider) }
    let(:version) { exam.current_version }

    it "versão do exame é imutável" do
      expect { version.update!(method: "Outro") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { version.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    it "resultado imutável e cifrado; inspect sem valor" do
      req = lab_requisition_row!(provider: provider, version: version)
      item = req.items.first
      result = LabResult.create!(lab_requisition_item: item, analyte_key: "glicose", value: "123", unit: "mg/dL", flag: "high",
                                 source: "typed", entered_by_user: lab_staff!, entered_at: Time.current)
      raw = ApplicationRecord.connection.select_value("SELECT value FROM lab_results WHERE id = #{ApplicationRecord.connection.quote(result.id)}")
      expect(raw).not_to include("123")
      expect(result.inspect).not_to include("123")
      expect { result.update!(flag: "normal") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { LabRequisitionItem.create!(lab_requisition: req, exam_order_id: item.exam_order_id, lab_exam_version: version, lab_sample: item.lab_sample) }
        .to raise_error(ActiveRecord::RecordNotUnique)
    end

    it "laudo: só anda para a frente; conteúdo nunca muda; não some" do
      req = lab_requisition_row!(provider: provider, version: version)
      report = LabReport.create!(lab_requisition: req, exam_provider: provider, version: 1, partial: false, status: "pending_signature",
                                 signature_kind: "icp", released_by_user: lab_releaser!(provider), content: { "a" => 1 }.to_json,
                                 content_sha256: "a" * 64, verification_token: ClinicalDocuments::Codes.token,
                                 short_code: ClinicalDocuments::Codes.short_code)
      expect { report.update!(content: "{}") }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
      expect { report.update!(status: "rectified") }.to raise_error(ActiveRecord::StatementInvalid, /refused/)
      file = pdf_file!(user: report.released_by_user).tap { |f| f.update!(consumed_at: Time.current) }
      report.update!(status: "released", released_at: Time.current, pdf_file_id: file.id, pdf_sha256: "b" * 64)
      expect { report.update!(status: "pending_signature") }.to raise_error(ActiveRecord::StatementInvalid, /refused/)
      report.update!(status: "rectified")
      expect { report.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      expect { req.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    it "papéis, número da requisição, código de barras e GAL uma vez" do
      user = staff_with("lab-#{SecureRandom.hex(3)}@cidade.gov.br", "lab_staff", "lab_releaser")
      expect(user.has_role?("lab_staff")).to be(true)
      expect(Membership::PRIVILEGED_ROLES).to include("lab_staff", "lab_releaser")
      first = LabRequisition.next_number
      expect(LabRequisition.next_number.to_i).to eq(first.to_i + 1)
      expect(first).to match(/\A\d{8}\z/)
      expect(LabSample.barcode_for("00000123", 1)).to eq("0000012301")
      order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: create_unit("UBS GAL"), requester: doctor!(create_unit("UBS D")))
      order.update!(gal_requisition: "2026-000123")
      expect { order.update!(gal_requisition: "outro") }.to raise_error(ActiveRecord::StatementInvalid, /recorded once/)
    end
  end
  ```
  `lab_requisition_row!` fica no `spec/support/lab_helpers.rb` (Step 6).

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/models/municipal_lab_schema_spec.rb` → `NoMethodError: lab_city!`.

- [ ] **Step 3: A migração** `db/city_migrate/20261015900002_add_municipal_lab.rb`:

  ```ruby
  # Módulo 21b (ADR 0037): laboratório municipal sobre o rastreamento do 21a.
  # Papéis lab_staff e lab_releaser; flags de prestador e unidade; origem
  # lab_reception, fonte lab e evento result_rectified no 21a; retificação do
  # resultado por acréscimo (replaces_result_id); assinatura do laudo pelo 19b
  # (LabReport, sem consulta); conferência pública do laudo; as tabelas lab_*.
  # Triggers e a sequência do número em db/city_triggers.sql. Irreversível.
  class AddMunicipalLab < ActiveRecord::Migration[8.1]
    ROLES = %w[analyst campaign_manager citizen_verifier exam_provider exam_regulator health_professional lab_releaser lab_staff
               municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze
    ORIGINS = %w[consultation reception_guide pec lab_reception].freeze
    EVENT_KINDS = %w[requested authorized denied scheduled collected resulted acknowledged escalated cancelled result_rectified].freeze
    SOURCES = %w[unit provider fhir consultation lab].freeze
    SIGNABLE = %w[Consultation ConsultationAddendum ClinicalDocument LabReport].freeze
    SECTORS = %w[biochemistry hematology immunology urinalysis parasitology microbiology].freeze
    FLAGS = %w[normal low high critical_low critical_high].freeze

    def up
      replace_check :memberships, "ck_memberships_role", "role::text = ANY (ARRAY[#{quoted(ROLES)}]::text[])"

      add_column :exam_providers, :uses_municipal_lab, :boolean, null: false, default: false
      add_column :exam_providers, :address, :string, limit: 200
      add_column :exam_providers, :phone, :string, limit: 11
      add_check_constraint :exam_providers, "phone IS NULL OR phone::text ~ '^[0-9]{10,11}$'::text", name: "ck_exam_providers_phone"
      add_column :health_units, :collects_samples, :boolean, null: false, default: false
      add_column :exam_orders, :gal_requisition, :string, limit: 30
      add_check_constraint :exam_orders, "gal_requisition IS NULL OR gal_requisition::text ~ '^[0-9A-Za-z-]{1,30}$'::text", name: "ck_exam_orders_gal"
      replace_check :exam_orders, "ck_exam_orders_origin", "origin::text = ANY (ARRAY[#{quoted(ORIGINS)}]::text[])"
      replace_check :exam_order_events, "ck_exam_order_events_kind", "kind::text = ANY (ARRAY[#{quoted(EVENT_KINDS)}]::text[])"
      replace_check :exam_results, "ck_exam_results_source", "source::text = ANY (ARRAY[#{quoted(SOURCES)}]::text[])"

      add_reference :exam_results, :replaces_result, type: :uuid, foreign_key: { to_table: :exam_results }, index: { unique: true }
      remove_index :exam_results, name: "index_exam_results_on_exam_order_id"
      add_index :exam_results, :exam_order_id, name: "idx_exam_results_order"
      add_index :exam_results, :exam_order_id, unique: true, where: "replaces_result_id IS NULL", name: "idx_exam_results_order_original"
      add_check_constraint :exam_results, "replaces_result_id IS NULL OR origin::text = 'order'::text", name: "ck_exam_results_replaces"

      change_column_null :signature_requests, :consultation_id, true
      replace_check :signature_requests, "ck_signature_requests_document_type", "document_type::text = ANY (ARRAY[#{quoted(SIGNABLE)}]::text[])"
      add_check_constraint :signature_requests, "(document_type::text = 'LabReport'::text) = (consultation_id IS NULL)", name: "ck_signature_requests_consultation"
      replace_check :signatures, "ck_signatures_document_type", "document_type::text = ANY (ARRAY[#{quoted(SIGNABLE)}]::text[])"

      create_table :lab_exams, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.uuid :lab_exam_template_id
        t.string :sigtap_code, limit: 10, null: false
        t.string :name, limit: 120, null: false
        t.string :sector, null: false
        t.string :bench, limit: 60
        t.integer :delivery_days, null: false
        t.boolean :active, null: false, default: true
        t.uuid :current_version_id
        t.timestamps
      end
      add_index :lab_exams, %i[exam_provider_id sigtap_code], unique: true, name: "idx_lab_exams_provider_sigtap"
      add_check_constraint :lab_exams, "sigtap_code::text ~ '^02[0-9]{8}$'::text", name: "ck_lab_exams_sigtap"
      add_check_constraint :lab_exams, "sector::text = ANY (ARRAY[#{quoted(SECTORS)}]::text[])", name: "ck_lab_exams_sector"
      add_check_constraint :lab_exams, "delivery_days BETWEEN 1 AND 90", name: "ck_lab_exams_delivery_days"

      create_table :lab_exam_versions, id: :uuid do |t|
        t.references :lab_exam, type: :uuid, null: false, foreign_key: true, index: false
        t.integer :version, null: false
        t.string :method, limit: 160, null: false
        t.string :material, limit: 60, null: false
        t.string :container, limit: 60, null: false
        t.string :instrument_test_code, limit: 30
        t.jsonb :analytes, null: false
        t.references :created_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :created_at, null: false
      end
      add_index :lab_exam_versions, %i[lab_exam_id version], unique: true, name: "idx_lab_exam_versions_number"
      add_check_constraint :lab_exam_versions, "version >= 1", name: "ck_lab_exam_versions_version"
      add_check_constraint :lab_exam_versions, "jsonb_typeof(analytes) = 'array' AND jsonb_array_length(analytes) BETWEEN 1 AND 60", name: "ck_lab_exam_versions_analytes"
      add_check_constraint :lab_exam_versions, "instrument_test_code IS NULL OR instrument_test_code::text ~ '^[A-Za-z0-9._-]{1,30}$'::text", name: "ck_lab_exam_versions_instrument"
      add_foreign_key :lab_exams, :lab_exam_versions, column: :current_version_id

      create_table :lab_rejection_reasons, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.string :name, limit: 80, null: false
        t.boolean :active, null: false, default: true
        t.timestamps
      end
      add_index :lab_rejection_reasons, "exam_provider_id, lower((name)::text)", unique: true, name: "idx_lab_rejection_reasons_name"

      create_table :lab_releasers, id: :uuid do |t|
        t.references :user, type: :uuid, null: false, foreign_key: true, index: false
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.boolean :technical_lead, null: false, default: false
        t.boolean :technical_deputy, null: false, default: false
        t.references :created_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :ended_at
        t.references :ended_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :lab_releasers, %i[user_id exam_provider_id], unique: true, where: "ended_at IS NULL", name: "idx_lab_releasers_active"
      add_index :lab_releasers, :exam_provider_id, unique: true, where: "ended_at IS NULL AND technical_lead", name: "idx_lab_releasers_lead"
      add_index :lab_releasers, :exam_provider_id, unique: true, where: "ended_at IS NULL AND technical_deputy", name: "idx_lab_releasers_deputy"
      add_check_constraint :lab_releasers, "NOT (technical_lead AND technical_deputy)", name: "ck_lab_releasers_marks"

      create_table :lab_middleware_tokens, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.string :token_digest, limit: 64, null: false
        t.boolean :simulated, null: false, default: false
        t.references :created_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :revoked_at
        t.datetime :created_at, null: false
      end
      add_index :lab_middleware_tokens, :token_digest, unique: true
      add_index :lab_middleware_tokens, :exam_provider_id, unique: true, where: "revoked_at IS NULL", name: "idx_lab_middleware_tokens_active"

      create_table :lab_requisitions, id: :uuid do |t|
        t.string :number, limit: 8, null: false
        t.references :patient, type: :uuid, null: false, foreign_key: true, index: false
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.references :origin_unit, type: :uuid, null: false, foreign_key: { to_table: :health_units }, index: false
        t.string :collection_place, null: false
        t.string :biological_sex, null: false
        t.text :mother_name, null: false
        t.boolean :urgent, null: false, default: false
        t.text :clinical_info
        t.date :expected_delivery_on, null: false
        t.string :status, null: false, default: "open"
        t.text :delivered_to
        t.datetime :delivered_at
        t.references :delivered_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.references :opened_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :lab_requisitions, :number, unique: true
      add_index :lab_requisitions, %i[exam_provider_id status created_at], name: "idx_lab_requisitions_provider_status"
      add_index :lab_requisitions, %i[origin_unit_id created_at], name: "idx_lab_requisitions_origin"
      add_index :lab_requisitions, %i[patient_id created_at], name: "idx_lab_requisitions_patient"
      add_check_constraint :lab_requisitions, "number::text ~ '^[0-9]{8}$'::text", name: "ck_lab_requisitions_number"
      add_check_constraint :lab_requisitions, "collection_place::text = ANY (ARRAY['ubs'::text, 'lab'::text])", name: "ck_lab_requisitions_place"
      add_check_constraint :lab_requisitions, "biological_sex::text = ANY (ARRAY['male'::text, 'female'::text])", name: "ck_lab_requisitions_sex"
      add_check_constraint :lab_requisitions, "status::text = ANY (ARRAY['open'::text, 'in_analysis'::text, 'typed'::text, 'released'::text, 'delivered'::text])", name: "ck_lab_requisitions_status"
      add_check_constraint :lab_requisitions, "(status::text = 'delivered'::text) = (delivered_at IS NOT NULL AND delivered_to IS NOT NULL)", name: "ck_lab_requisitions_delivery"

      create_table :lab_shipments, id: :uuid do |t|
        t.references :origin_unit, type: :uuid, null: false, foreign_key: { to_table: :health_units }, index: false
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.decimal :temperature_celsius, precision: 4, scale: 1, null: false
        t.string :status, null: false, default: "open"
        t.references :created_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.references :sent_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :sent_at
        t.references :received_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :received_at
        t.timestamps
      end
      add_index :lab_shipments, %i[exam_provider_id status], name: "idx_lab_shipments_provider_status"
      add_index :lab_shipments, %i[origin_unit_id status], name: "idx_lab_shipments_origin_status"
      add_check_constraint :lab_shipments, "status::text = ANY (ARRAY['open'::text, 'sent'::text, 'received'::text])", name: "ck_lab_shipments_status"
      add_check_constraint :lab_shipments, "temperature_celsius BETWEEN -30 AND 60", name: "ck_lab_shipments_temperature"

      create_table :lab_samples, id: :uuid do |t|
        t.references :lab_requisition, type: :uuid, null: false, foreign_key: true, index: true
        t.string :barcode, limit: 10, null: false
        t.string :material, limit: 60, null: false
        t.string :container, limit: 60, null: false
        t.string :status, null: false, default: "awaiting"
        t.references :collected_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :collected_at
        t.references :lab_shipment, type: :uuid, foreign_key: true, index: true
        t.references :received_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :received_at
        t.references :lab_rejection_reason, type: :uuid, foreign_key: true, index: false
        t.string :restriction_note, limit: 300
        t.references :recollection_of, type: :uuid, foreign_key: { to_table: :lab_samples }, index: { unique: true }
        t.timestamps
      end
      add_index :lab_samples, :barcode, unique: true
      add_index :lab_samples, %i[status received_at], name: "idx_lab_samples_status_received"
      add_check_constraint :lab_samples, "barcode::text ~ '^[0-9]{10}$'::text", name: "ck_lab_samples_barcode"
      add_check_constraint :lab_samples, "status::text = ANY (ARRAY['awaiting'::text, 'in_transit'::text, 'received'::text, 'rejected'::text, 'accepted_with_restriction'::text, 'not_received'::text])", name: "ck_lab_samples_status"
      add_check_constraint :lab_samples, "(status::text = 'rejected'::text) = (lab_rejection_reason_id IS NOT NULL)", name: "ck_lab_samples_rejection"
      add_check_constraint :lab_samples, "(status::text = 'accepted_with_restriction'::text) = (restriction_note IS NOT NULL)", name: "ck_lab_samples_restriction"
      add_check_constraint :lab_samples, "status::text = 'awaiting'::text OR collected_at IS NOT NULL", name: "ck_lab_samples_collected"
      add_check_constraint :lab_samples, "status::text NOT IN ('received', 'rejected', 'accepted_with_restriction') OR received_at IS NOT NULL", name: "ck_lab_samples_received"

      create_table :lab_requisition_items, id: :uuid do |t|
        t.references :lab_requisition, type: :uuid, null: false, foreign_key: true, index: true
        t.references :exam_order, type: :uuid, null: false, foreign_key: true, index: { unique: true }
        t.references :lab_exam_version, type: :uuid, null: false, foreign_key: true, index: false
        t.references :lab_sample, type: :uuid, null: false, foreign_key: true, index: true
        t.string :status, null: false, default: "pending"
        t.timestamps
      end
      add_check_constraint :lab_requisition_items, "status::text = ANY (ARRAY['pending'::text, 'typed'::text, 'released'::text, 'cancelled'::text])", name: "ck_lab_requisition_items_status"

      create_table :lab_hl7_receipts, id: :uuid do |t|
        t.references :lab_middleware_token, type: :uuid, null: false, foreign_key: true, index: false
        t.string :control_id, limit: 64, null: false
        t.string :message_type, limit: 20, null: false
        t.text :raw, null: false
        t.datetime :created_at, null: false
      end
      add_index :lab_hl7_receipts, %i[lab_middleware_token_id control_id], unique: true, name: "idx_lab_hl7_receipts_control"

      create_table :lab_results, id: :uuid do |t|
        t.references :lab_requisition_item, type: :uuid, null: false, foreign_key: true, index: false
        t.string :analyte_key, limit: 40, null: false
        t.text :value, null: false
        t.string :unit, limit: 30
        t.string :flag, null: false
        t.string :source, null: false
        t.references :lab_hl7_receipt, type: :uuid, foreign_key: true, index: true
        t.references :entered_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :entered_at, null: false
        t.boolean :chosen, null: false, default: false
        t.references :chosen_from, type: :uuid, foreign_key: { to_table: :lab_results }, index: false
        t.datetime :created_at, null: false
      end
      add_index :lab_results, %i[lab_requisition_item_id analyte_key created_at], name: "idx_lab_results_item_analyte"
      add_check_constraint :lab_results, "flag::text = ANY (ARRAY[#{quoted(FLAGS)}]::text[])", name: "ck_lab_results_flag"
      add_check_constraint :lab_results, "source::text = ANY (ARRAY['typed'::text, 'hl7'::text])", name: "ck_lab_results_source"
      add_check_constraint :lab_results, "entered_by_user_id IS NOT NULL OR lab_hl7_receipt_id IS NOT NULL", name: "ck_lab_results_actor"
      add_check_constraint :lab_results, "chosen = (chosen_from_id IS NOT NULL)", name: "ck_lab_results_choice"
      add_check_constraint :lab_results, "source::text = 'typed'::text OR lab_hl7_receipt_id IS NOT NULL", name: "ck_lab_results_hl7"

      create_table :lab_reports, id: :uuid do |t|
        t.references :lab_requisition, type: :uuid, null: false, foreign_key: true, index: true
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.integer :version, null: false
        t.references :replaces_report, type: :uuid, foreign_key: { to_table: :lab_reports }, index: { unique: true }
        t.boolean :partial, null: false, default: false
        t.string :status, null: false
        t.string :signature_kind, null: false
        t.references :released_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :released_at
        t.text :content, null: false
        t.string :content_sha256, limit: 64, null: false
        t.references :pdf_file, type: :uuid, foreign_key: { to_table: :exam_files }, index: { unique: true }
        t.string :pdf_sha256, limit: 64
        t.string :verification_token, limit: 22, null: false
        t.string :short_code, limit: 10, null: false
        t.text :rectification_reason
        t.text :rectification_contacts
        t.timestamps
      end
      add_index :lab_reports, %i[lab_requisition_id version], unique: true, name: "idx_lab_reports_version"
      add_index :lab_reports, :verification_token, unique: true
      add_index :lab_reports, :short_code, unique: true
      add_index :lab_reports, %i[exam_provider_id released_at], name: "idx_lab_reports_provider_released"
      add_check_constraint :lab_reports, "status::text = ANY (ARRAY['pending_signature'::text, 'released'::text, 'rectified'::text, 'withdrawn'::text])", name: "ck_lab_reports_status"
      add_check_constraint :lab_reports, "signature_kind::text = ANY (ARRAY['icp'::text, 'advanced'::text])", name: "ck_lab_reports_signature"
      add_check_constraint :lab_reports, "status::text IN ('pending_signature', 'withdrawn') OR (released_at IS NOT NULL AND pdf_file_id IS NOT NULL AND pdf_sha256 IS NOT NULL)", name: "ck_lab_reports_released"
      add_check_constraint :lab_reports, "(replaces_report_id IS NULL) = (rectification_reason IS NULL)", name: "ck_lab_reports_rectification"
      add_check_constraint :lab_reports, "short_code::text ~ '^[23456789ABCDEFGHJKMNPQRSTUVWXYZ]{10}$'::text", name: "ck_lab_reports_short_code"

      create_table :lab_report_items, id: :uuid do |t|
        t.references :lab_report, type: :uuid, null: false, foreign_key: true, index: false
        t.references :lab_requisition_item, type: :uuid, null: false, foreign_key: true, index: true
        t.datetime :created_at, null: false
      end
      add_index :lab_report_items, %i[lab_report_id lab_requisition_item_id], unique: true, name: "idx_lab_report_items_pair"

      create_table :lab_critical_notices, id: :uuid do |t|
        t.references :lab_result, type: :uuid, null: false, foreign_key: true, index: true
        t.text :notified_person, null: false
        t.string :notified_role, limit: 60, null: false
        t.string :channel, null: false
        t.datetime :notified_at, null: false
        t.references :recorded_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :created_at, null: false
      end
      add_check_constraint :lab_critical_notices, "channel::text = ANY (ARRAY['telefone'::text, 'presencial'::text, 'sistema'::text])", name: "ck_lab_critical_notices_channel"

      add_reference :document_verification_lookups, :lab_report, type: :uuid, foreign_key: true, index: true
      replace_check :document_verification_lookups, "ck_document_verification_lookups_document",
                    "(outcome::text = 'found'::text) = (clinical_document_id IS NOT NULL OR lab_report_id IS NOT NULL) AND (clinical_document_id IS NULL OR lab_report_id IS NULL)"

      execute File.read(Rails.root.join("db/city_triggers.sql"))
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end

    private

    def quoted(list) = list.map { |value| "'#{value}'::text" }.join(", ")

    def replace_check(table, name, expression)
      remove_check_constraint table, name: name
      add_check_constraint table, expression, name: name
    end
  end
  ```

- [ ] **Step 4: Triggers** — em `db/city_triggers.sql`:
  1. Em `rota_exam_order_guard`, logo depois do bloco da re-cifra, antes da comparação de identidade:

  ```sql
    -- ADR 0037 (F-21.24): o número do GAL é registrado UMA vez, sem evento.
    IF OLD.gal_requisition IS NOT NULL AND NEW.gal_requisition IS DISTINCT FROM OLD.gal_requisition THEN
      RAISE EXCEPTION 'exam_orders: gal_requisition is recorded once';
    END IF;
    IF OLD.gal_requisition IS NULL AND NEW.gal_requisition IS NOT NULL
       AND (to_jsonb(NEW) - 'gal_requisition' - 'updated_at') = (to_jsonb(OLD) - 'gal_requisition' - 'updated_at') THEN
      RETURN NEW;
    END IF;
  ```
  2. No fim do arquivo:

  ```sql
  -- Módulo 21b (ADR 0037, Invariantes): versão do exame, resultados, recibos
  -- HL7, itens do laudo e avisos de crítico só crescem; o laudo só anda para a
  -- frente (pending_signature → released|withdrawn; released → rectified) e o
  -- conteúdo nunca muda; requisição, item, amostra e remessa nunca somem. A
  -- sequência do número da requisição mora aqui (o dump não a representa).
  CREATE OR REPLACE FUNCTION rota_lab_report_guard() RETURNS trigger AS $fn$
  DECLARE
    encrypted text[] := ARRAY['content', 'rectification_reason', 'rectification_contacts'];
    moving text[] := ARRAY['status', 'released_at', 'pdf_file_id', 'pdf_sha256', 'updated_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'lab_reports is append-only: DELETE refused';
    END IF;
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - encrypted) = (to_jsonb(OLD) - encrypted) THEN
      RETURN NEW;
    END IF;
    IF (to_jsonb(NEW) - moving) IS DISTINCT FROM (to_jsonb(OLD) - moving) THEN
      RAISE EXCEPTION 'lab_reports: a report never changes (only its status moves forward)';
    END IF;
    IF OLD.status = 'pending_signature' AND NEW.status IN ('released', 'withdrawn') THEN
      RETURN NEW;
    END IF;
    IF OLD.status = 'released' AND NEW.status = 'rectified'
       AND (NEW.released_at, NEW.pdf_file_id, NEW.pdf_sha256) IS NOT DISTINCT FROM (OLD.released_at, OLD.pdf_file_id, OLD.pdf_sha256) THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'lab_reports: status % -> % refused', OLD.status, NEW.status;
  END;
  $fn$ LANGUAGE plpgsql;

  DO $do$
  DECLARE
    item text;
  BEGIN
    IF to_regclass('public.lab_requisitions') IS NOT NULL THEN
      CREATE SEQUENCE IF NOT EXISTS lab_requisition_number_seq START 1 MAXVALUE 99999999;
      EXECUTE 'DROP TRIGGER IF EXISTS lab_reports_guard ON lab_reports';
      EXECUTE 'CREATE TRIGGER lab_reports_guard BEFORE UPDATE OR DELETE ON lab_reports
        FOR EACH ROW EXECUTE FUNCTION rota_lab_report_guard()';
      FOREACH item IN ARRAY ARRAY['lab_exam_versions', 'lab_report_items'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE UPDATE OR DELETE ON %I FOR EACH ROW EXECUTE FUNCTION rota_append_only()',
                       item || '_append_only', item);
      END LOOP;
      EXECUTE 'DROP TRIGGER IF EXISTS lab_results_reencryption_only ON lab_results';
      EXECUTE 'CREATE TRIGGER lab_results_reencryption_only BEFORE UPDATE OR DELETE ON lab_results
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''value'')';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_hl7_receipts_reencryption_only ON lab_hl7_receipts';
      EXECUTE 'CREATE TRIGGER lab_hl7_receipts_reencryption_only BEFORE UPDATE OR DELETE ON lab_hl7_receipts
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''raw'')';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_critical_notices_reencryption_only ON lab_critical_notices';
      EXECUTE 'CREATE TRIGGER lab_critical_notices_reencryption_only BEFORE UPDATE OR DELETE ON lab_critical_notices
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''notified_person'')';
      FOREACH item IN ARRAY ARRAY['lab_requisitions', 'lab_requisition_items', 'lab_samples', 'lab_shipments'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_no_delete', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE DELETE ON %I FOR EACH ROW EXECUTE FUNCTION rota_append_only()',
                       item || '_no_delete', item);
      END LOOP;
      FOREACH item IN ARRAY ARRAY['lab_exam_versions', 'lab_results', 'lab_hl7_receipts', 'lab_reports', 'lab_report_items',
                                  'lab_critical_notices', 'lab_requisitions', 'lab_requisition_items', 'lab_samples', 'lab_shipments'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only_truncate', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE TRUNCATE ON %I FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()',
                       item || '_append_only_truncate', item);
      END LOOP;
    END IF;
  END
  $do$;
  ```
  (`rota_append_only` responde `<tabela> is append-only: DELETE refused` — o teste do Step 1 casa com `/append-only/` e `/DELETE refused/`.)

- [ ] **Step 5: Modelos.** `app/models/lab_exam.rb`:

  ```ruby
  # Exame adotado pelo laboratório (ADR 0037; spec §4): SIGTAP, setor, bancada,
  # prazo de entrega e a versão vigente (as versões são imutáveis).
  class LabExam < ApplicationRecord
    SECTORS = %w[biochemistry hematology immunology urinalysis parasitology microbiology].freeze

    belongs_to :exam_provider
    belongs_to :current_version, class_name: "LabExamVersion", optional: true
    has_many :versions, -> { order(:version) }, class_name: "LabExamVersion", inverse_of: :lab_exam

    validates :sector, inclusion: { in: SECTORS }
    validates :delivery_days, numericality: { only_integer: true, in: 1..90 }
  end
  ```
  `app/models/lab_exam_version.rb`:

  ```ruby
  # Versão imutável do exame do laboratório (trigger): método, material,
  # recipiente, código do aparelho e componentes (chave, nome, LOINC, unidade,
  # tipo, casas, faixas, limites de digitação, crítico, fórmula). O laudo guarda a versão.
  class LabExamVersion < ApplicationRecord
    belongs_to :lab_exam
    belongs_to :created_by_user, class_name: "User"

    def analyte(key) = analytes.find { |a| a["key"] == key.to_s }
    def entered_analytes = analytes.reject { |a| a["formula"].present? }
    def calculated_analytes = analytes.select { |a| a["formula"].present? }
    def inspect = "#<LabExamVersion id=#{id} lab_exam_id=#{lab_exam_id} version=#{version}>"
  end
  ```
  `app/models/lab_rejection_reason.rb`: `class LabRejectionReason < ApplicationRecord; belongs_to :exam_provider; scope :active, -> { where(active: true) }; end` (com o cabeçalho "Motivo de rejeição de amostra do laboratório (spec §4)").
  `app/models/lab_releaser.rb`:

  ```ruby
  # Liberador ligado a um laboratório (ADR 0037): RT (technical_lead) e
  # substituto (technical_deputy), um ativo de cada por laboratório. Encerrar = ended_at.
  class LabReleaser < ApplicationRecord
    belongs_to :user
    belongs_to :exam_provider
    belongs_to :created_by_user, class_name: "User", optional: true
    belongs_to :ended_by_user, class_name: "User", optional: true

    scope :active, -> { where(ended_at: nil) }
  end
  ```
  `app/models/lab_middleware_token.rb`:

  ```ruby
  # Token do middleware do laboratório (ADR 0037): só o SHA-256; um ativo por
  # laboratório; `simulated` só a semente de dev grava (fake-middleware).
  class LabMiddlewareToken < ApplicationRecord
    belongs_to :exam_provider
    belongs_to :created_by_user, class_name: "User", optional: true

    scope :active, -> { where(revoked_at: nil) }

    def self.digest(token) = Digest::SHA256.hexdigest(token.to_s)

    # Ativo ou revogado (o revogado responde AR, contrato §7); nil se não existe.
    def self.by_token(token)
      return nil unless token.is_a?(String) && token.length.between?(32, 128)

      find_by(token_digest: digest(token))
    end

    def revoked? = revoked_at.present?
    def inspect = "#<LabMiddlewareToken id=#{id} exam_provider_id=#{exam_provider_id} revoked=#{revoked?}>"
  end
  ```
  `app/models/lab_requisition.rb`:

  ```ruby
  # Requisição do laboratório (ADR 0037; spec §4–§5): junta exames do 21a do
  # paciente. Nome da mãe, informação clínica e quem retirou, cifrados.
  class LabRequisition < ApplicationRecord
    STATUSES = %w[open in_analysis typed released delivered].freeze
    PLACES = %w[ubs lab].freeze
    SEXES = %w[male female].freeze

    encrypts :mother_name
    encrypts :clinical_info
    encrypts :delivered_to

    belongs_to :patient
    belongs_to :exam_provider
    belongs_to :origin_unit, class_name: "HealthUnit"
    belongs_to :opened_by_user, class_name: "User"
    belongs_to :delivered_by_user, class_name: "User", optional: true
    has_many :items, -> { order(:created_at, :id) }, class_name: "LabRequisitionItem", inverse_of: :lab_requisition
    has_many :samples, -> { order(:barcode) }, class_name: "LabSample", inverse_of: :lab_requisition
    has_many :reports, -> { order(:version) }, class_name: "LabReport", inverse_of: :lab_requisition

    def self.next_number = format("%08d", connection.select_value("SELECT nextval('lab_requisition_number_seq')"))

    def inspect = "#<LabRequisition id=#{id} number=#{number} status=#{status}>"
  end
  ```
  `app/models/lab_requisition_item.rb`:

  ```ruby
  class LabRequisitionItem < ApplicationRecord
    STATUSES = %w[pending typed released cancelled].freeze

    belongs_to :lab_requisition
    belongs_to :exam_order
    belongs_to :lab_exam_version
    belongs_to :lab_sample
    has_many :results, -> { order(:created_at, :id) }, class_name: "LabResult", inverse_of: :lab_requisition_item

    def active? = status != "cancelled"
  end
  ```
  `app/models/lab_sample.rb`:

  ```ruby
  # Amostra (ADR 0037; spec §4): código de barras = número da requisição + 2
  # dígitos. awaiting com collected_at = coletada (Desvio 10).
  class LabSample < ApplicationRecord
    STATUSES = %w[awaiting in_transit received rejected accepted_with_restriction not_received].freeze
    ACCEPTED = %w[received accepted_with_restriction].freeze
    MAX_PER_REQUISITION = 99

    belongs_to :lab_requisition
    belongs_to :lab_shipment, optional: true
    belongs_to :lab_rejection_reason, optional: true
    belongs_to :recollection_of, class_name: "LabSample", optional: true
    belongs_to :collected_by_user, class_name: "User", optional: true
    belongs_to :received_by_user, class_name: "User", optional: true
    has_many :items, class_name: "LabRequisitionItem", inverse_of: :lab_sample

    def self.barcode_for(number, index)
      raise ArgumentError, "amostra fora de 1..#{MAX_PER_REQUISITION}" unless index.between?(1, MAX_PER_REQUISITION)

      "#{number}#{format('%02d', index)}"
    end

    def collected? = collected_at.present?
    def accepted? = ACCEPTED.include?(status)
  end
  ```
  `app/models/lab_shipment.rb`: `belongs_to :origin_unit, class_name: "HealthUnit"; belongs_to :exam_provider; belongs_to :created_by_user/sent_by_user/received_by_user (class_name "User", os dois últimos optional); has_many :samples, class_name: "LabSample"`; `STATUSES = %w[open sent received]`.
  `app/models/lab_result.rb`:

  ```ruby
  # Resultado do laboratório (ADR 0037): só acréscimo (trigger); valor cifrado.
  # Vale a última linha do componente; escolher acrescenta cópia (Desvio 9).
  class LabResult < ApplicationRecord
    FLAGS = %w[normal low high critical_low critical_high].freeze
    CRITICAL = %w[critical_low critical_high].freeze

    encrypts :value

    belongs_to :lab_requisition_item
    belongs_to :lab_hl7_receipt, optional: true
    belongs_to :entered_by_user, class_name: "User", optional: true
    belongs_to :chosen_from, class_name: "LabResult", optional: true
    has_many :critical_notices, class_name: "LabCriticalNotice", inverse_of: :lab_result

    def critical? = CRITICAL.include?(flag)
    def inspect = "#<LabResult id=#{id} analyte_key=#{analyte_key} source=#{source} chosen=#{chosen}>"
  end
  ```
  `app/models/lab_hl7_receipt.rb`: `encrypts :raw; belongs_to :lab_middleware_token; def inspect = "#<LabHl7Receipt id=#{id} message_type=#{message_type}>"`.
  `app/models/lab_report.rb`:

  ```ruby
  # Laudo (ADR 0037; RDC 978 art. 167): conteúdo canônico congelado e cifrado
  # (lab-report-v1), PDF em exam_files. Só anda para a frente (trigger).
  # Assinável pelo 19b: author_user_id = quem liberou; sem consulta.
  class LabReport < ApplicationRecord
    STATUSES = %w[pending_signature released rectified withdrawn].freeze

    encrypts :content
    encrypts :rectification_reason
    encrypts :rectification_contacts

    belongs_to :lab_requisition
    belongs_to :exam_provider
    belongs_to :released_by_user, class_name: "User"
    belongs_to :replaces_report, class_name: "LabReport", optional: true
    belongs_to :pdf_file, class_name: "ExamFile", optional: true
    has_many :report_items, class_name: "LabReportItem", inverse_of: :lab_report
    has_many :items, through: :report_items, source: :lab_requisition_item

    def author_user_id = released_by_user_id
    def consultation_id = nil
    def content_data = JSON.parse(content)
    def verification_url = "#{CityPublicUrl.base(Current.city)}/v/#{verification_token}"
    def signature_request = SignatureRequest.find_by(document_type: "LabReport", document_id: id)
    def inspect = "#<LabReport id=#{id} version=#{version} status=#{status} signature_kind=#{signature_kind}>"
  end
  ```
  `app/models/lab_report_item.rb`: `belongs_to :lab_report; belongs_to :lab_requisition_item`.
  `app/models/lab_critical_notice.rb`: `CHANNELS = %w[telefone presencial sistema]; encrypts :notified_person; belongs_to :lab_result; belongs_to :recorded_by_user, class_name: "User"; def inspect = "#<LabCriticalNotice id=#{id} channel=#{channel}>"`.
  Modifique: `Membership::ROLES` = a lista de `AddMunicipalLab::ROLES` e `PRIVILEGED_ROLES` += `lab_staff lab_releaser` (comentário: "ADR 0037: veem nome e amostra do paciente; o liberador assina laudo — step-up; o mantenedor não concede"); `ExamOrder::ORIGINS` += `lab_reception` e `has_one :lab_item, class_name: "LabRequisitionItem"`; `ExamOrderEvent::KINDS` += `result_rectified`; `DocumentVerificationLookup`: `belongs_to :lab_report, optional: true`.
  Nas specs `spec/models/create_campaigns_migration_spec.rb` e `create_analytics_migration_spec.rb`, em `restore_exam_roles`, troque `AddExamTracking::ROLES` por `AddMunicipalLab::ROLES` e o comentário por "20261015900002 (módulo 21b) reescreve ck_memberships_role…".

- [ ] **Step 6: Cifra, eventos, filtros e helpers.** `app/services/city_encryption.rb`, depois de `[ ExamFile, :data ]`:

  ```ruby
      # ADR 0037: laboratório municipal.
      [ LabRequisition, :mother_name ],
      [ LabRequisition, :clinical_info ],
      [ LabRequisition, :delivered_to ],
      [ LabResult, :value ],
      [ LabHl7Receipt, :raw ],
      [ LabReport, :content ],
      [ LabReport, :rectification_reason ],
      [ LabReport, :rectification_contacts ],
      [ LabCriticalNotice, :notified_person ]
  ```
  `config/initializers/domain_events.rb`, depois dos eventos do 21a:

  ```ruby
    # Módulo 21b (ADR 0037): laboratório municipal, só trilha, só ids. Nunca
    # valor, nome, CPF, mensagem HL7 ou token.
    %w[lab_requisition.opened lab_requisition.delivered lab_sample.collected lab_sample.received lab_sample.rejected
       lab_sample.label_printed lab_shipment.sent lab_shipment.received lab_result.entered lab_report.released
       lab_report.rectified lab_report.withdrawn lab_critical.notified lab_exam.version_created lab_releaser.assigned
       lab_releaser.ended lab_middleware.token_rotated lab_middleware.token_revoked exam_order.result_rectified
       exam_order.gal_registered].each { |name| DomainEvents.bind name, to: [] }
  ```
  (`exam_order.result_rectified` não está no laço do 21a — `Exams::Transition` o publica pelo `kind`.) Em `spec/initializers/domain_events_bindings_spec.rb`, um `describe "municipal lab event bindings (ADR 0037)"` com a mesma lista, no molde do bloco do 21a (`registry.keys` inclui e nenhum consumidor). `config/initializers/filter_parameter_logging.rb`, no fim da lista:

  ```ruby
    # ADR 0037: nome da mãe, informação clínica, quem retirou, quem foi avisado,
    # contatos da retificação, o TOTP da assinatura avançada, a restrição da
    # amostra e o número do GAL.
    /\A(mother_name|clinical_info|delivered_to|notified_person|contacts|totp|restriction_note|gal_requisition)\z/
  ```
  `spec/support/lab_helpers.rb`:

  ```ruby
  # Módulo 21b (ADR 0037): cidade com exam_tracking e municipal_lab, o
  # laboratório municipal, exames adotados com versão, equipe do laboratório.
  module LabHelpers
    GLUCOSE = [ { "key" => "glicose", "name" => "Glicose", "loinc" => "2345-7", "unit" => "mg/dL", "value_type" => "numeric",
                  "choices" => [], "decimals" => 0,
                  "reference_ranges" => [ { "sex" => "any", "age_min_days" => 0, "age_max_days" => 43_800, "low" => 70.0, "high" => 99.0, "text" => nil } ],
                  "entry_min" => 10.0, "entry_max" => 1500.0, "critical_low" => 40.0, "critical_high" => 400.0, "formula" => nil } ].freeze
    LIPIDS = [
      { "key" => "ct", "name" => "Colesterol total", "loinc" => "2093-3", "unit" => "mg/dL", "value_type" => "numeric", "choices" => [], "decimals" => 0,
        "reference_ranges" => [], "entry_min" => nil, "entry_max" => nil, "critical_low" => nil, "critical_high" => nil, "formula" => nil },
      { "key" => "hdl", "name" => "HDL", "loinc" => "2085-9", "unit" => "mg/dL", "value_type" => "numeric", "choices" => [], "decimals" => 0,
        "reference_ranges" => [], "entry_min" => nil, "entry_max" => nil, "critical_low" => nil, "critical_high" => nil, "formula" => nil },
      { "key" => "tg", "name" => "Triglicerídeos", "loinc" => "2571-8", "unit" => "mg/dL", "value_type" => "numeric", "choices" => [], "decimals" => 0,
        "reference_ranges" => [], "entry_min" => nil, "entry_max" => nil, "critical_low" => nil, "critical_high" => nil, "formula" => nil },
      { "key" => "ldl", "name" => "LDL (Friedewald)", "loinc" => "13457-7", "unit" => "mg/dL", "value_type" => "numeric", "choices" => [], "decimals" => 0,
        "reference_ranges" => [], "entry_min" => nil, "entry_max" => nil, "critical_low" => nil, "critical_high" => nil, "formula" => "ct - hdl - tg / 5" }
    ].freeze

    def json_patch(path, params = {}) = patch(path, params: params.to_json, headers: { "CONTENT_TYPE" => "application/json" })

    def lab_city!(hl7: false, fake_middleware: false)
      city = exam_city!
      maintainer = ledi_maintainer!
      Platform::Features.set!(city: city, key: "municipal_lab", enabled: true, maintainer: maintainer)
      Platform::Features.set!(city: city, key: "lab_hl7_inbound", enabled: true, maintainer: maintainer) if hl7 || fake_middleware
      Platform::Features.set!(city: city, key: "fake_middleware", enabled: true, maintainer: maintainer) if fake_middleware
      CityCatalog.reset_cache!
      city
    end

    def lab_provider!(name: "Laboratório Municipal #{SecureRandom.hex(2)}")
      ExamProvider.create!(name: name, kind: "municipal", uses_municipal_lab: true, cnes: format("%07d", SecureRandom.random_number(9_000_000) + 1_000_000),
                           address: "Rua do Laboratório, 100 — Centro", phone: "4133501200")
    end

    def lab_exam!(provider:, sigtap_code: "0202010473", name: "Glicose", analytes: GLUCOSE, material: "Plasma",
                  container: "Tubo cinza (fluoreto)", instrument_test_code: "GLI", sector: "biochemistry", delivery_days: 3)
      exam = LabExam.create!(exam_provider: provider, sigtap_code: sigtap_code, name: name, sector: sector, delivery_days: delivery_days)
      version = LabExamVersion.create!(lab_exam: exam, version: 1, method: "Enzimático", material: material, container: container,
                                       instrument_test_code: instrument_test_code, analytes: analytes, created_by_user: lab_admin!)
      exam.update!(current_version: version)
      exam
    end

    def lab_admin! = (@lab_admin ||= staff_with("admin-lab-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin"))
    def lab_staff! = staff_with("tecnica-#{SecureRandom.hex(3)}@cidade.gov.br", "lab_staff")

    # Liberador com perfil (conselho do laboratório) e TOTP, ligado ao laboratório.
    def lab_releaser!(provider, council: "CRBM", lead: false, deputy: false)
      user = staff_with("liberador-#{SecureRandom.hex(3)}@cidade.gov.br", "lab_releaser")
      Professional.create!(user: user, professional_name: "Liberadora #{SecureRandom.hex(2)}", council: council, council_state: "PR",
                           registration_number: (SecureRandom.random_number(90_000) + 10_000).to_s,
                           cns: Professionals::Cns.generate("lab-#{user.id}"), cpf: nil)
      Mfa::Enroll.call(user)
      user.update!(otp_enabled: true)
      LabReleaser.create!(user: user, exam_provider: provider, technical_lead: lead, technical_deputy: deputy)
      user
    end

    def totp_for(user) = ROTP::TOTP.new(user.reload.otp_secret).now

    # Requisição gravada direto (antes da Task 8): uma amostra, um item.
    def lab_requisition_row!(provider:, version:, unit: create_unit("UBS #{SecureRandom.hex(2)}"))
      patient = exam_patient!(exam_seq!).first
      order = exam_order!(patient: patient, unit: unit, requester: doctor!(unit), sigtap_code: version.lab_exam.sigtap_code)
      req = LabRequisition.create!(number: LabRequisition.next_number, patient: patient, exam_provider: provider, origin_unit: unit,
                                   collection_place: "lab", biological_sex: "female", mother_name: "Joana", urgent: false,
                                   expected_delivery_on: Time.zone.today + 3, opened_by_user: lab_staff!)
      sample = LabSample.create!(lab_requisition: req, barcode: LabSample.barcode_for(req.number, 1), material: version.material,
                                 container: version.container)
      LabRequisitionItem.create!(lab_requisition: req, exam_order: order, lab_exam_version: version, lab_sample: sample)
      req
    end
  end

  RSpec.configure { |c| c.include LabHelpers }
  ```

- [ ] **Step 7: Rode a migração e escreva o dump.** Recrie os bancos de teste `_mod21b` ("Ambiente") e migre o de dev do worktree **só pelo rake**: `rails city:migrate:all`. **Escreva à mão** `db/city_schema.rb`: `define(version: 2026_10_15_900002)`; as colunas novas em `exam_providers`, `health_units`, `exam_orders`, `exam_results`, `signature_requests` (`consultation_id` sem `null: false`), `document_verification_lookups`; os `check_constraint` trocados; os blocos `create_table` novos em ordem alfabética (`lab_critical_notices`, `lab_exam_versions`, `lab_exams`, `lab_hl7_receipts`, `lab_middleware_tokens`, `lab_rejection_reasons`, `lab_releasers`, `lab_report_items`, `lab_reports`, `lab_requisition_items`, `lab_requisitions`, `lab_results`, `lab_samples`, `lab_shipments`) com colunas em ordem alfabética, índices e checks como o `pg_dump` os devolve — compare com

  ```bash
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod21b -w /rails/.claude/mod21b api bin/rails runner 'CityConnection.with(TEST_CITY_A) { puts ActiveRecord::SchemaDumper.dump }' > /tmp/city_dump.rb
  ```
  (se `TEST_CITY_A` não existir fora do rspec, use o `CitySchema.db_config_for` do banco `rota_saude_test_city_a_mod21b`) — e as `add_foreign_key` novas em ordem. A sequência não entra no dump: ela vem de `db/city_triggers.sql`.

- [ ] **Step 8: Rode:** `rspec spec/models/municipal_lab_schema_spec.rb spec/services/city_schema_spec.rb spec/architecture spec/initializers/domain_events_bindings_spec.rb spec/models/create_campaigns_migration_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/exam_tracking_schema_spec.rb` → verde.

- [ ] **Step 9: Commit:**

  ```bash
  git add db/city_migrate/20261015900002_add_municipal_lab.rb db/city_schema.rb db/city_triggers.sql app/models/lab_*.rb app/models/membership.rb app/models/exam_order.rb app/models/exam_order_event.rb app/models/document_verification_lookup.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/models/create_campaigns_migration_spec.rb spec/models/create_analytics_migration_spec.rb spec/support/lab_helpers.rb spec/models/municipal_lab_schema_spec.rb
  git commit -m "feat: add municipal lab city tables, roles and immutability triggers

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 2 — Catálogo do laboratório, equipe e token (F-21.15)

### Task 6: Base das rotas `/lab`, contexto, catálogo do laboratório com versões, motivos de rejeição, flags e recusas do 21a

**Files:**
- Create: `app/controllers/lab/base_controller.rb`, `app/controllers/lab/context_controller.rb`, `app/controllers/lab/lookups_controller.rb`
- Create: `app/controllers/lab/admin/{base_controller,templates_controller,exams_controller,rejection_reasons_controller}.rb`
- Create: `app/services/lab/{access,analytes,formula,admin_json}.rb`, `app/commands/lab/{adopt_exam,create_version,save_rejection_reason}.rb`
- Modify: `config/routes.rb`, `app/commands/exams/save_provider.rb`, `app/services/exams/admin_json.rb`, `app/controllers/health_units_controller.rb`, `app/controllers/exams/results_controller.rb`, `app/controllers/exams/provider/orders_controller.rb`, `app/services/exams/fhir/inbound.rb`
- Test: `spec/services/lab/formula_spec.rb`, `spec/services/lab/analytes_spec.rb`, `spec/requests/lab/admin_catalog_spec.rb`, `spec/requests/lab/context_spec.rb`, `spec/requests/lab/exams_21a_refusals_spec.rb`

**Interfaces:**
- Consumes: `Lab::Gate` (Task 2), `LabExamTemplate` e `Lab::Ranges` (Task 3), modelos (Task 5).
- Produces:
  - `Lab::BaseController` (antes de tudo `require_municipal_lab!`; `render_failure(result)`, `forbid`, `not_found`, `uuid?`, `body_params`, `provider_param!` → `@provider` ou 404); `FAILURE_STATUS`.
  - `Lab::Access.labs`, `.staff?(user)`, `.releaser_link(user, provider_id)`, `.provider_ids(user)`, `.member?(user, provider_id)`, `.collecting_unit_ids(user)`, `.council(user)` (`"CRF"|"CRBM"|"CRM"|nil`), `.admin?(user)`.
  - `Lab::Formula.compile(text, allowed: [String]) → Array|nil` (RPN), `.evaluate(rpn, values: { "key" => BigDecimal }) → BigDecimal|nil`.
  - `Lab::Analytes.parse(raw) → Result.ok(analytes: [Hash])` | `fail(:invalid_analytes|:invalid_reference_ranges|:invalid_formula)`.
  - `Lab::AdminJson.template(t)`, `.exam(e, versions: false)`, `.version(v)`, `.reason(r)`.
  - `Lab::AdoptExam.call(provider:, template_id:, sector:, bench:, delivery_days:, by:) → Result.ok(exam:)`; `Lab::CreateVersion.call(exam:, params:, by:) → Result.ok(version:)`; `Lab::SaveRejectionReason.call(params, reason: nil) → Result.ok(reason:)`.
  - Rotas: `GET /lab/context` → `{ "labs": [{ id, name }], "releaser_of": [id], "council": "CRBM"|null }` (dashboard D1); `GET /lab/rejection_reasons?provider_id=` → `{ "reasons": [<reason>] }` só ativos (D14); `GET /lab/admin/templates` → `{ "templates": [<template>] }`; `GET /lab/admin/exams?provider_id=` → `{ "exams": [<exam>] }`; `POST /lab/admin/exams`; `GET|PATCH /lab/admin/exams/:id`; `POST /lab/admin/exams/:id/versions` → `<version>`; `GET|POST /lab/admin/rejection_reasons`, `PATCH /lab/admin/rejection_reasons/:id`. Formas exatas nos `Lab::AdminJson` do Step 6 (dashboard D12).

- [ ] **Step 1: Testes que falham** — `spec/services/lab/formula_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe Lab::Formula do
    it "compila e calcula Friedewald; recusa chave fora da lista e expressão mal formada" do
      rpn = described_class.compile("ct - hdl - tg / 5", allowed: %w[ct hdl tg])
      values = { "ct" => BigDecimal("200"), "hdl" => BigDecimal("50"), "tg" => BigDecimal("150") }
      expect(described_class.evaluate(rpn, values: values)).to eq(BigDecimal("120"))
      expect(described_class.evaluate(described_class.compile("(ct + hdl) * 2", allowed: %w[ct hdl]), values: values)).to eq(500)
      expect(described_class.evaluate(rpn, values: values.except("tg"))).to be_nil
      expect(described_class.evaluate(described_class.compile("ct / (hdl - hdl)", allowed: %w[ct hdl]), values: values)).to be_nil
      [ "ct - x", "ct +", "(ct", "ct)", "ct hdl", "", "ct; system('x')", "-ct", "ct ** 2", "a" * 201 ].each do |bad|
        expect(described_class.compile(bad, allowed: %w[ct hdl])).to be_nil, bad
      end
    end
  end
  ```
  `spec/services/lab/analytes_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe Lab::Analytes do
    let(:base) { LabHelpers::LIPIDS.map(&:dup) }

    it "normaliza e aceita calculado sobre numéricos não calculados" do
      result = described_class.parse(base)
      expect(result).to be_ok
      expect(result.payload[:analytes].last).to include("key" => "ldl", "formula" => "ct - hdl - tg / 5")
    end

    it "recusa o que não fecha" do
      { [ base.first.merge("key" => "CT") ] => :invalid_analytes,
        [ base.first, base.first ] => :invalid_analytes,
        [ base.first.merge("value_type" => "choice", "choices" => []) ] => :invalid_analytes,
        [ base.first.merge("entry_min" => 10, "entry_max" => 5) ] => :invalid_analytes,
        [ base.first.merge("decimals" => 5) ] => :invalid_analytes,
        [ base.first.merge("reference_ranges" => [ { "sex" => "x", "age_min_days" => 0, "age_max_days" => 1 } ]) ] => :invalid_reference_ranges,
        base[0..2] + [ base[3].merge("formula" => "ct - ldl") ] => :invalid_formula,
        [ base.first.merge("value_type" => "text", "formula" => "1 + 1") ] => :invalid_formula,
        [] => :invalid_analytes, "nope" => :invalid_analytes }.each do |raw, code|
        expect(described_class.parse(raw).reason).to eq(code), raw.inspect[0, 120]
      end
    end
  end
  ```
  `spec/requests/lab/admin_catalog_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3 (F-21.15) com as formas do dashboard (D12): adotar do
  # catálogo-base, nova versão (nunca edita), motivos de rejeição, flags.
  RSpec.describe "Laboratório — admin do catálogo", type: :request do
    before do
      lab_city!
      Lab::Templates::SaveExam.call(sigtap_code: "0202010473", name: "Glicose", sector: "biochemistry", material: "Plasma",
                                    container: "Tubo cinza (fluoreto)", suggested_method: "Enzimático", active: true)
      Lab::Templates::SaveAnalyte.call(lab_exam_template_id: template.id, key: "glicose", name: "Glicose", loinc: "2345-7", unit: "mg/dL",
                                       value_type: "numeric", choices: [], decimals: 0, position: 1,
                                       reference_ranges: [ { sex: "any", age_min_days: 0, age_max_days: 43_800, low: 70, high: 99, text: nil } ])
    end

    let(:template) { LabExamTemplate.find_by!(sigtap_code: "0202010473") }
    let(:provider) { lab_provider! }
    let(:admin) { lab_admin! }

    it "adota, cria versão nova, edita prazo; o laudo antigo continua com a versão dele" do
      stepped_up!(admin)
      get "/lab/admin/templates"
      expect(json_body["templates"].first).to include("sigtap_code" => "0202010473", "sector" => "biochemistry",
                                                      "analytes" => [ hash_including("key" => "glicose", "value_type" => "numeric") ])
      json_post "/lab/admin/exams", provider_id: provider.id, template_id: template.id, sector: "biochemistry", bench: "Bancada 1", delivery_days: 2
      expect(response).to have_http_status(:created)
      exam = json_body
      expect(exam).to include("sigtap_code" => "0202010473", "name" => "Glicose", "delivery_days" => 2, "active" => true)
      expect(exam["current_version"]).to include("version" => 1, "method" => "Enzimático", "material" => "Plasma", "instrument_test_code" => nil)
      expect(exam["current_version"]["analytes"].first).to include("entry_min" => nil, "critical_high" => nil, "formula" => nil)
      json_post "/lab/admin/exams", provider_id: provider.id, template_id: template.id, sector: "biochemistry", bench: nil, delivery_days: 2
      expect(json_body["error"]).to eq("duplicate_sigtap")

      analytes = exam["current_version"]["analytes"].map { |a| a.merge("critical_high" => 400, "entry_max" => 1500) }
      json_post "/lab/admin/exams/#{exam['id']}/versions", method: "Enzimático (GOD-PAP)", material: "Plasma", container: "Tubo cinza",
                                                          instrument_test_code: "GLI", analytes: analytes
      expect(response).to have_http_status(:created)
      expect(json_body).to include("version" => 2, "instrument_test_code" => "GLI")
      expect(LabExam.find(exam["id"]).versions.map(&:version)).to eq([ 1, 2 ])
      json_post "/lab/admin/exams/#{exam['id']}/versions", method: "x", material: "Plasma", container: "x", instrument_test_code: "GLI",
                                                          analytes: analytes.map { |a| a.merge("formula" => "glicose * 2") }
      expect(json_body["error"]).to eq("invalid_formula")
      other = lab_exam!(provider: provider, sigtap_code: "0202010317", name: "Creatinina", instrument_test_code: "CRE")
      json_post "/lab/admin/exams/#{other.id}/versions", method: "Jaffé", material: "Soro", container: "Tubo amarelo",
                                                         instrument_test_code: "GLI", analytes: LabHelpers::GLUCOSE
      expect(json_body["error"]).to eq("duplicate_instrument_code")

      json_patch "/lab/admin/exams/#{exam['id']}", sector: "biochemistry", bench: nil, delivery_days: 5, active: false
      expect(json_body).to include("delivery_days" => 5, "active" => false, "bench" => nil)
      get "/lab/admin/exams/#{exam['id']}"
      expect(json_body["versions"].map { |v| v["version"] }).to eq([ 1, 2 ])
      expect(DomainEvent.where(name: "lab_exam.version_created").count).to eq(2)
    end

    it "motivos de rejeição; recepção lê só os ativos" do
      stepped_up!(admin)
      json_post "/lab/admin/rejection_reasons", provider_id: provider.id, name: "Amostra hemolisada", active: true
      reason = json_body
      json_post "/lab/admin/rejection_reasons", provider_id: provider.id, name: "amostra HEMOLISADA", active: true
      expect(json_body["error"]).to eq("duplicate_name")
      json_post "/lab/admin/rejection_reasons", provider_id: provider.id, name: "Volume insuficiente", active: true
      json_patch "/lab/admin/rejection_reasons/#{reason['id']}", name: "Amostra hemolisada", active: false
      sign_in_as(lab_staff!)
      get "/lab/rejection_reasons", params: { provider_id: provider.id }
      expect(json_body["reasons"].map { |r| r["name"] }).to eq([ "Volume insuficiente" ])
    end

    it "sem step-up 401; sem papel 403; interruptor desligado 403" do
      sign_in_as(admin)
      json_post "/lab/admin/exams", provider_id: provider.id, template_id: template.id, sector: "biochemistry", delivery_days: 2
      expect(json_body["error"]).to eq("mfa_required")
      sign_in_as(lab_staff!)
      get "/lab/admin/templates"
      expect(json_body["error"]).to eq("missing_role")
      Platform::Features.set!(city: City.find(Current.city.id), key: "municipal_lab", enabled: false, maintainer: ledi_maintainer!)
      get "/lab/context"
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "municipal_lab")
    end

    it "prestador marca uses_municipal_lab, endereço e telefone; unidade marca collects_samples" do
      stepped_up!(admin)
      lab = exam_provider!(name: "Lab Novo", items: [])
      json_patch "/exams/admin/providers/#{lab.id}", uses_municipal_lab: true, address: "Rua A, 1", phone: "4133334444"
      expect(json_body).to include("uses_municipal_lab" => true, "address" => "Rua A, 1", "phone" => "4133334444", "has_middleware_token" => false)
      json_patch "/exams/admin/providers/#{lab.id}", phone: "123"
      expect(json_body["error"]).to eq("invalid_phone")
      unit = create_unit("UBS Coleta")
      json_post "/attendance/units/#{unit.id}", name: unit.name, kind: "ubs", collects_samples: true
      expect(json_body.dig("unit", "collects_samples")).to be(true)
      json_post "/attendance/units/#{unit.id}", name: unit.name, kind: "ubs", collects_samples: "sim"
      expect(json_body["error"]).to eq("invalid_collects_samples")
    end
  end
  ```
  `spec/requests/lab/context_spec.rb`:

  ```ruby
  require "rails_helper"

  # Dashboard D1: onde a sessão atua e o conselho de quem libera.
  RSpec.describe "Laboratório — contexto da sessão", type: :request do
    before { lab_city! }

    let!(:lab) { lab_provider!(name: "Laboratório A") }
    let!(:other) { lab_provider!(name: "Laboratório B") }

    it "técnica vê todos; liberador vê o seu e o conselho; UBS que coleta vê os laboratórios; estranho 403" do
      sign_in_as(lab_staff!)
      get "/lab/context"
      expect(json_body).to eq("labs" => [ { "id" => lab.id, "name" => "Laboratório A" }, { "id" => other.id, "name" => "Laboratório B" } ],
                              "releaser_of" => [], "council" => nil)
      releaser = lab_releaser!(lab, council: "CRBM")
      sign_in_as(releaser)
      get "/lab/context"
      expect(json_body).to eq("labs" => [ { "id" => lab.id, "name" => "Laboratório A" } ], "releaser_of" => [ lab.id ], "council" => "CRBM")
      unit = create_unit("UBS Coletora").tap { |u| u.update!(collects_samples: true) }
      sign_in_as(doctor!(unit))
      get "/lab/context"
      expect(json_body["labs"].size).to eq(2)
      sign_in_as(doctor!(create_unit("UBS Sem Coleta")))
      get "/lab/context"
      expect(json_body["error"]).to eq("missing_role")
    end
  end
  ```
  `spec/requests/lab/exams_21a_refusals_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §9 + Desvio 16: laboratório municipal não recebe resultado pelo
  # portal, pelo FHIR nem pela unidade — só pela liberação.
  RSpec.describe "21a recusa resultado de laboratório municipal", type: :request do
    before { lab_city!(hl7: true) }

    let(:unit) { create_unit("UBS Recusa") }
    let(:doctor) { doctor!(unit) }
    let(:item) { exam_catalog_item!(sigtap_code: "0202010503") }
    let(:lab) { exam_provider!(token: ExamHelpers::FAKE_TOKEN, uses_municipal_lab: true).tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }
    let(:order) do
      exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor).tap do |o|
        Exams::Steps.schedule(order: o, actor_user: doctor, params: { provider_id: lab.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
      end
    end

    it "portal, unidade e FHIR: 422 provider_uses_municipal_lab" do
      Platform::Features.set!(city: City.find(Current.city.id), key: "exam_fhir_inbound", enabled: true, maintainer: ledi_maintainer!)
      portal = staff_with("portal-#{SecureRandom.hex(2)}@lab.gov.br", "exam_provider").tap { |u| ExamProviderUser.create!(user: u, exam_provider: lab) }
      sign_in_as(portal)
      json_post "/exams/provider/orders/#{order.id}/result", upload_id: pdf_file!(user: portal).id, released_on: Time.zone.today.iso8601,
                                                             laboratory_name: "Lab", critical: false, values: []
      expect([ response.status, json_body["error"] ]).to eq([ 422, "provider_uses_municipal_lab" ])
      sign_in_as(staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier"))
      json_post "/exams/orders/#{order.id}/result", upload_id: SecureRandom.uuid, released_on: Time.zone.today.iso8601,
                                                    laboratory_name: "Lab", critical: false, values: []
      expect(json_body["error"]).to eq("provider_uses_municipal_lab")
      post "/fhir/DiagnosticReport", params: { resourceType: "DiagnosticReport" }.to_json,
                                     headers: { "CONTENT_TYPE" => "application/fhir+json", "Authorization" => "Bearer #{ExamHelpers::FAKE_TOKEN}" }
      expect([ response.status, json_body["error"] ]).to eq([ 422, "provider_uses_municipal_lab" ])
      expect(ExamResult.count).to eq(0)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/lab/formula_spec.rb spec/services/lab/analytes_spec.rb spec/requests/lab/admin_catalog_spec.rb spec/requests/lab/context_spec.rb spec/requests/lab/exams_21a_refusals_spec.rb` → `NameError`/404.

- [ ] **Step 3: Fórmula e componentes.** `app/services/lab/formula.rb`:

  ```ruby
  # Fórmula de componente calculado (spec §4; Valores fixados 3): números,
  # chaves permitidas, + - * / e parênteses. Sem eval: shunting-yard para RPN
  # na versão e cálculo com BigDecimal na digitação. Divisão por zero ou
  # componente ausente = sem valor (nil).
  module Lab
    module Formula
      TOKEN = %r{\A\s*(?:(\d+(?:\.\d+)?)|([a-z][a-z0-9_]{0,39})|([-+*/()]))\s*}
      PRECEDENCE = { "+" => 1, "-" => 1, "*" => 2, "/" => 2 }.freeze
      MAX = 200

      module_function

      def compile(text, allowed:)
        tokens = tokenize(text)
        return nil unless tokens && well_formed?(tokens) && tokens.all? { |kind, v| kind != :var || allowed.include?(v) }

        to_rpn(tokens)
      end

      def evaluate(rpn, values:)
        stack = []
        rpn.each do |kind, v|
          case kind
          when :num then stack << v
          when :var then (value = values[v]) ? stack << value : (return nil)
          when :op
            right, left = stack.pop, stack.pop
            return nil if v == "/" && right.zero?

            stack << left.public_send(v, right)
          end
        end
        stack.size == 1 ? stack.first : nil
      end

      def tokenize(text)
        return nil unless text.is_a?(String) && text.strip.length.between?(1, MAX)

        rest = text.strip
        tokens = []
        until rest.empty?
          match = rest.match(TOKEN) or return nil
          tokens << if match[1] then [ :num, BigDecimal(match[1]) ]
                    elsif match[2] then [ :var, match[2] ]
                    elsif "()".include?(match[3]) then [ :paren, match[3] ]
                    else [ :op, match[3] ]
                    end
          rest = match.post_match
        end
        tokens
      end

      # Operando e operador alternados; parênteses equilibrados; sem unário.
      def well_formed?(tokens)
        expect_operand = true
        depth = 0
        tokens.each do |kind, v|
          if kind == :paren && v == "("
            return false unless expect_operand

            depth += 1
          elsif kind == :paren
            return false if expect_operand || depth.zero?

            depth -= 1
          elsif kind == :op
            return false if expect_operand

            expect_operand = true
          else
            return false unless expect_operand

            expect_operand = false
          end
        end
        !expect_operand && depth.zero?
      end

      def to_rpn(tokens)
        output = []
        ops = []
        tokens.each do |kind, v|
          case kind
          when :num, :var then output << [ kind, v ]
          when :op
            output << ops.pop while ops.last && ops.last[0] == :op && PRECEDENCE[ops.last[1]] >= PRECEDENCE[v]
            ops << [ :op, v ]
          when :paren
            if v == "(" then ops << [ :paren, v ]
            else
              output << ops.pop until ops.last[0] == :paren
              ops.pop
            end
          end
        end
        output.concat(ops.reverse)
      end
      private_class_method :tokenize, :well_formed?, :to_rpn
    end
  end
  ```
  `app/services/lab/analytes.rb`:

  ```ruby
  # Componentes de uma versão (contrato §3): chave, nome, LOINC, unidade, tipo,
  # lista, casas, faixas (Lab::Ranges), limites de digitação, crítico e fórmula
  # (só em numérico, sobre numéricos não calculados da MESMA versão).
  module Lab
    module Analytes
      KEY = /\A[a-z][a-z0-9_]{0,39}\z/
      LOINC = /\A\d{1,7}-\d\z/
      TYPES = %w[numeric text choice].freeze
      MAX = 60
      LIMITS = %w[entry_min entry_max critical_low critical_high].freeze

      module_function

      def parse(raw)
        return invalid unless raw.is_a?(Array) && raw.size.between?(1, MAX)

        list = raw.map { |entry| entry.respond_to?(:to_unsafe_h) ? entry.to_unsafe_h : entry }
        return invalid unless list.all?(Hash)

        list = list.map(&:stringify_keys)
        return invalid unless list.map { |a| a["key"] }.uniq.size == list.size

        analytes = []
        list.each do |a|
          one = analyte(a)
          return one if one.is_a?(Result)

          analytes << one
        end
        numeric_keys = analytes.select { |a| a["value_type"] == "numeric" && a["formula"].nil? }.map { |a| a["key"] }
        analytes.each do |a|
          next unless a["formula"]
          return Result.fail(:invalid_formula) unless a["value_type"] == "numeric" && Formula.compile(a["formula"], allowed: numeric_keys)
        end
        Result.ok(analytes: analytes)
      end

      def analyte(a)
        return invalid unless a["key"].is_a?(String) && a["key"].match?(KEY)
        return invalid unless a["name"].is_a?(String) && a["name"].strip.length.between?(1, 120)
        return invalid unless TYPES.include?(a["value_type"])
        return invalid unless a["loinc"].nil? || (a["loinc"].is_a?(String) && a["loinc"].match?(LOINC))
        return invalid unless a["unit"].nil? || (a["unit"].is_a?(String) && a["unit"].length <= 30)
        return invalid unless a["decimals"].is_a?(Integer) && a["decimals"].between?(0, 4)

        choices = Array(a["choices"]).map(&:to_s).map(&:strip).reject(&:empty?)
        return invalid if a["value_type"] == "choice" && !choices.size.between?(1, 30)
        return invalid if choices.any? { |c| c.length > 60 }

        ranges = Ranges.parse(Array(a["reference_ranges"]))
        return ranges if ranges.failure?

        limits = LIMITS.to_h { |k| [ k, Ranges.number(a[k]) ] }
        return invalid if limits.values.include?(:invalid)
        return invalid if a["value_type"] != "numeric" && limits.values.any?
        return invalid if limits["entry_min"] && limits["entry_max"] && limits["entry_min"] >= limits["entry_max"]

        formula = a["formula"].is_a?(String) ? a["formula"].strip.presence : nil
        return Result.fail(:invalid_formula) if a["formula"].present? && a["value_type"] != "numeric"

        { "key" => a["key"], "name" => a["name"].strip, "loinc" => a["loinc"], "unit" => a["unit"].presence,
          "value_type" => a["value_type"], "choices" => a["value_type"] == "choice" ? choices : [], "decimals" => a["decimals"],
          "reference_ranges" => ranges.payload[:ranges], **limits, "formula" => formula }
      end

      def invalid = Result.fail(:invalid_analytes)
      private_class_method :analyte, :invalid
    end
  end
  ```

- [ ] **Step 4: Acesso e base.** `app/services/lab/access.rb`:

  ```ruby
  # Quem atua no laboratório (ADR 0037; Desvio 2): lab_staff em todo
  # laboratório municipal ativo; lab_releaser no laboratório a que está ligado;
  # a UBS que coleta (profissional com vínculo ativo em unidade collects_samples).
  module Lab
    module Access
      COUNCILS = %w[CRF CRBM CRM].freeze

      module_function

      def labs = ExamProvider.where(uses_municipal_lab: true, active: true).order(:name)
      def staff?(user) = user.has_role?("lab_staff")
      def admin?(user) = user.has_role?("municipal_admin")

      def releaser_link(user, provider_id)
        return nil unless user.has_role?("lab_releaser")

        LabReleaser.active.find_by(user_id: user.id, exam_provider_id: provider_id)
      end

      def releaser_provider_ids(user)
        user.has_role?("lab_releaser") ? LabReleaser.active.where(user_id: user.id).pluck(:exam_provider_id) : []
      end

      def provider_ids(user)
        ids = releaser_provider_ids(user)
        staff?(user) ? (labs.pluck(:id) | ids) : ids
      end

      def member?(user, provider_id) = provider_ids(user).include?(provider_id)

      def collecting_unit_ids(user)
        return [] unless user.has_role?("health_professional") && user.professional

        ProfessionalLink.active.joins(:health_unit).where(professional_id: user.professional.id,
                                                          health_units: { collects_samples: true, active: true })
                        .distinct.pluck(:health_unit_id)
      end

      def council(user)
        council = user.professional&.council
        COUNCILS.include?(council) ? council : nil
      end
    end
  end
  ```
  `app/controllers/lab/base_controller.rb`:

  ```ruby
  # Base das rotas /lab/* (ADR 0037; contrato §3–§6): sessão do dashboard e
  # municipal_lab utilizável (que exige exam_tracking).
  module Lab
    class BaseController < ApplicationController
      include Authentication
      include MfaStepUp

      UUID = Exams::BaseController::UUID
      FAILURE_STATUS = {
        invalid_transition: :conflict, gal_already_registered: :conflict, not_found: :not_found,
        missing_role: :forbidden, not_unit_staff: :forbidden, mfa_required: :unauthorized
      }.freeze

      before_action :require_municipal_lab!

      private

      def require_municipal_lab!
        return if Lab::Gate.usable?(Current.city)

        render json: { error: "feature_disabled", feature: Lab::Gate::KEY }, status: :forbidden
      end

      def render_failure(result)
        payload = { error: result.reason.to_s }
        payload[:status] = result.details[:status] if result.reason == :invalid_transition && result.details[:status]
        payload[:analyte_key] = result.details[:analyte_key] if result.details[:analyte_key]
        payload[:barcode] = result.details[:barcode] if result.details[:barcode]
        render json: payload, status: FAILURE_STATUS.fetch(result.reason, :unprocessable_entity)
      end

      def forbid(reason = "missing_role") = render(json: { error: reason }, status: :forbidden)
      def not_found = render(json: { error: "not_found" }, status: :not_found)
      def uuid?(value) = value.is_a?(String) && value.match?(UUID)
      def body_params = params.except(:controller, :action, :id).permit!.to_h

      # O laboratório do parâmetro: municipal, ativo e da sessão; senão 404/403.
      def provider_param!(key = :provider_id)
        @provider = uuid?(params[key]) && Lab::Access.labs.find_by(id: params[key])
        return not_found unless @provider

        forbid unless Lab::Access.member?(Current.user, @provider.id) || Lab::Access.admin?(Current.user)
      end
    end
  end
  ```
  `app/controllers/lab/context_controller.rb`:

  ```ruby
  # GET /lab/context (dashboard D1): laboratórios em que a sessão atua, de quais
  # é liberadora e o conselho (CRF|CRBM|CRM ou null).
  module Lab
    class ContextController < BaseController
      def show
        user = Current.user
        ids = Lab::Access.provider_ids(user)
        ids = Lab::Access.labs.pluck(:id) if ids.empty? && (Lab::Access.collecting_unit_ids(user).any? || Lab::Access.admin?(user))
        return forbid if ids.empty?

        render json: { labs: Lab::Access.labs.where(id: ids).map { |p| { id: p.id, name: p.name } },
                       releaser_of: Lab::Access.releaser_provider_ids(user), council: Lab::Access.council(user) }
      end
    end
  end
  ```
  `app/controllers/lab/lookups_controller.rb` (cresce nas Tasks 8 e 10):

  ```ruby
  module Lab
    class LookupsController < BaseController
      before_action :provider_param!, only: %i[rejection_reasons]

      # Dashboard D14: motivos ATIVOS para o recebimento.
      def rejection_reasons
        render json: { reasons: LabRejectionReason.active.where(exam_provider_id: @provider.id).order(:name).map { |r| Lab::AdminJson.reason(r) } }
      end
    end
  end
  ```

- [ ] **Step 5: Comandos do catálogo.** `app/commands/lab/adopt_exam.rb`:

  ```ruby
  # Adotar um exame do catálogo-base (contrato §3): copia nome, material,
  # recipiente, método sugerido e componentes para a versão 1 (sem limites,
  # crítico ou fórmula — o laboratório ajusta numa versão nova).
  module Lab
    module AdoptExam
      DEFAULT_METHOD = "Conforme o fabricante do reagente".freeze

      module_function

      def call(provider:, template_id:, sector:, bench:, delivery_days:, by:)
        template = template_id.is_a?(String) && LabExamTemplate.includes(:analytes).find_by(id: template_id, active: true)
        return Result.fail(:invalid_template) unless template
        return Result.fail(:invalid_sector) unless LabExam::SECTORS.include?(sector)
        return Result.fail(:invalid_delivery_days) unless delivery_days.is_a?(Integer) && delivery_days.between?(1, 90)
        return Result.fail(:invalid_bench) unless bench.nil? || (bench.is_a?(String) && bench.strip.length <= 60)

        analytes = template.analytes.map do |a|
          { "key" => a.key, "name" => a.name, "loinc" => a.loinc, "unit" => a.unit, "value_type" => a.value_type, "choices" => a.choices,
            "decimals" => a.decimals, "reference_ranges" => a.reference_ranges, "entry_min" => nil, "entry_max" => nil,
            "critical_low" => nil, "critical_high" => nil, "formula" => nil }
        end
        return Result.fail(:invalid_template) if analytes.empty?

        ApplicationRecord.transaction do
          exam = LabExam.create!(exam_provider: provider, lab_exam_template_id: template.id, sigtap_code: template.sigtap_code,
                                 name: template.name, sector: sector, bench: bench&.strip.presence, delivery_days: delivery_days)
          version = LabExamVersion.create!(lab_exam: exam, version: 1, method: template.suggested_method.presence || DEFAULT_METHOD,
                                           material: template.material, container: template.container, analytes: analytes,
                                           created_by_user: by)
          exam.update!(current_version: version)
          DomainEvents.publish("lab_exam.version_created", lab_exam_id: exam.id, version: 1)
          Result.ok(exam: exam)
        end
      rescue ActiveRecord::RecordNotUnique
        Result.fail(:duplicate_sigtap)
      end
    end
  end
  ```
  `app/commands/lab/create_version.rb`:

  ```ruby
  # Nova versão do exame (contrato §3): nunca edita a vigente. O código do
  # aparelho é único entre as versões VIGENTES do laboratório (trava do prestador).
  module Lab
    module CreateVersion
      module_function

      def call(exam:, params:, by:)
        params = params.to_h.stringify_keys
        method, material, container = %w[method material container].map { |k| params[k].is_a?(String) ? params[k].strip : "" }
        return Result.fail(:invalid_version) unless method.length.between?(1, 160) && material.length.between?(1, 60) && container.length.between?(1, 60)

        code = params["instrument_test_code"].presence
        return Result.fail(:invalid_version) unless code.nil? || (code.is_a?(String) && code.match?(/\A[A-Za-z0-9._-]{1,30}\z/))

        analytes = Analytes.parse(params["analytes"])
        return analytes if analytes.failure?

        ApplicationRecord.transaction do
          exam.exam_provider.lock!
          if code && LabExam.where(exam_provider_id: exam.exam_provider_id).where.not(id: exam.id)
                            .joins(:current_version).where(lab_exam_versions: { instrument_test_code: code }).exists?
            next Result.fail(:duplicate_instrument_code)
          end

          exam.lock!
          number = exam.versions.maximum(:version).to_i + 1
          version = LabExamVersion.create!(lab_exam: exam, version: number, method: method, material: material, container: container,
                                           instrument_test_code: code, analytes: analytes.payload[:analytes], created_by_user: by)
          exam.update!(current_version: version)
          DomainEvents.publish("lab_exam.version_created", lab_exam_id: exam.id, version: number)
          Result.ok(version: version)
        end
      end
    end
  end
  ```
  `app/commands/lab/save_rejection_reason.rb`:

  ```ruby
  module Lab
    module SaveRejectionReason
      module_function

      def call(params, reason: nil)
        params = params.to_h.stringify_keys
        name = params["name"].is_a?(String) ? params["name"].strip : ""
        return Result.fail(:invalid_name) unless name.length.between?(1, 80)
        return Result.fail(:invalid_active) unless [ true, false ].include?(params.fetch("active", true))

        reason ||= LabRejectionReason.new(exam_provider_id: params["provider_id"])
        reason.update!(name: name, active: params.fetch("active", true))
        Result.ok(reason: reason)
      rescue ActiveRecord::RecordNotUnique
        Result.fail(:duplicate_name)
      end
    end
  end
  ```

- [ ] **Step 6: Formas e controladores do admin.** `app/services/lab/admin_json.rb`:

  ```ruby
  # Formas do admin do laboratório (contrato §3; dashboard D12).
  module Lab
    module AdminJson
      ANALYTE_TEMPLATE = %w[key name loinc unit value_type choices decimals reference_ranges].freeze

      module_function

      def template(t)
        { id: t.id, sigtap_code: t.sigtap_code, name: t.name, sector: t.sector, material: t.material, container: t.container,
          suggested_method: t.suggested_method, preparation: t.preparation,
          analytes: t.analytes.map { |a| a.attributes.slice(*ANALYTE_TEMPLATE) } }
      end

      def version(v)
        { id: v.id, version: v.version, method: v.method, material: v.material, container: v.container,
          instrument_test_code: v.instrument_test_code, analytes: v.analytes,
          created_by: Screenings::Json.staff_name(v.created_by_user), created_at: v.created_at.iso8601 }
      end

      def exam(e, versions: false)
        body = { id: e.id, provider_id: e.exam_provider_id, template_id: e.lab_exam_template_id, sigtap_code: e.sigtap_code, name: e.name,
                 sector: e.sector, bench: e.bench, delivery_days: e.delivery_days, active: e.active,
                 current_version: e.current_version && version(e.current_version) }
        if versions
          body[:versions] = e.versions.includes(:created_by_user).map do |v|
            { id: v.id, version: v.version, created_at: v.created_at.iso8601, created_by: Screenings::Json.staff_name(v.created_by_user) }
          end
        end
        body
      end

      def reason(r) = { id: r.id, provider_id: r.exam_provider_id, name: r.name, active: r.active }
    end
  end
  ```
  `app/controllers/lab/admin/base_controller.rb`:

  ```ruby
  # /lab/admin/* (contrato §3): municipal_admin; escrita com step-up.
  module Lab
    module Admin
      class BaseController < Lab::BaseController
        before_action { forbid unless Lab::Access.admin?(Current.user) }
        before_action :require_step_up!, if: -> { request.post? || request.patch? || request.delete? }

        private

        def admin_provider!(id = params[:provider_id])
          @provider = uuid?(id) && Lab::Access.labs.find_by(id: id)
          render(json: { error: "invalid_provider" }, status: :unprocessable_entity) unless @provider
        end
      end
    end
  end
  ```
  `app/controllers/lab/admin/templates_controller.rb`:

  ```ruby
  module Lab
    module Admin
      class TemplatesController < BaseController
        def index
          templates = LabExamTemplate.where(active: true).includes(:analytes).order(:name)
          render json: { templates: templates.map { |t| Lab::AdminJson.template(t) } }
        end
      end
    end
  end
  ```
  `app/controllers/lab/admin/exams_controller.rb`:

  ```ruby
  module Lab
    module Admin
      class ExamsController < BaseController
        before_action :admin_provider!, only: %i[index create]
        before_action :load_exam, only: %i[show update create_version]

        def index
          exams = LabExam.where(exam_provider_id: @provider.id).includes(current_version: :created_by_user).order(:name)
          render json: { exams: exams.map { |e| Lab::AdminJson.exam(e) } }
        end

        def show = render(json: Lab::AdminJson.exam(@exam, versions: true))

        def create
          result = Lab::AdoptExam.call(provider: @provider, template_id: params[:template_id], sector: params[:sector],
                                       bench: params[:bench], delivery_days: params[:delivery_days], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Lab::AdminJson.exam(result.payload[:exam].reload, versions: true), status: :created
        end

        def update
          sector, bench, days, active = params.values_at(:sector, :bench, :delivery_days, :active)
          return render_failure(Result.fail(:invalid_sector)) unless LabExam::SECTORS.include?(sector)
          return render_failure(Result.fail(:invalid_delivery_days)) unless days.is_a?(Integer) && days.between?(1, 90)
          return render_failure(Result.fail(:invalid_active)) unless [ true, false ].include?(active)
          return render_failure(Result.fail(:invalid_bench)) unless bench.nil? || (bench.is_a?(String) && bench.strip.length <= 60)

          @exam.update!(sector: sector, bench: bench&.strip.presence, delivery_days: days, active: active)
          render json: Lab::AdminJson.exam(@exam, versions: true)
        end

        def create_version
          result = Lab::CreateVersion.call(exam: @exam, params: body_params, by: Current.user)
          return render_failure(result) if result.failure?

          render json: Lab::AdminJson.version(result.payload[:version]), status: :created
        end

        private

        def load_exam
          @exam = uuid?(params[:id]) && LabExam.find_by(id: params[:id])
          not_found unless @exam
        end
      end
    end
  end
  ```
  `app/controllers/lab/admin/rejection_reasons_controller.rb`:

  ```ruby
  module Lab
    module Admin
      class RejectionReasonsController < BaseController
        before_action :admin_provider!, only: %i[index create]

        def index
          render json: { reasons: LabRejectionReason.where(exam_provider_id: @provider.id).order(:name).map { |r| Lab::AdminJson.reason(r) } }
        end

        def create = save(Lab::SaveRejectionReason.call(body_params), :created)

        def update
          reason = uuid?(params[:id]) && LabRejectionReason.find_by(id: params[:id])
          return not_found unless reason

          save(Lab::SaveRejectionReason.call(body_params, reason: reason), :ok)
        end

        private

        def save(result, status)
          return render_failure(result) if result.failure?

          render json: Lab::AdminJson.reason(result.payload[:reason]), status: status
        end
      end
    end
  end
  ```
  Em `config/routes.rb`, depois do `scope "/exams"`:

  ```ruby
    # Módulo 21b (ADR 0037): laboratório municipal. As literais antes de `:id`.
    scope "/lab", module: :lab, as: :lab do
      get "context",           to: "context#show"
      get "rejection_reasons", to: "lookups#rejection_reasons"
      scope "admin", module: :admin, as: :admin do
        get   "templates",                 to: "templates#index"
        get   "exams",                     to: "exams#index"
        post  "exams",                     to: "exams#create"
        get   "exams/:id",                 to: "exams#show"
        patch "exams/:id",                 to: "exams#update"
        post  "exams/:id/versions",        to: "exams#create_version"
        get   "rejection_reasons",         to: "rejection_reasons#index"
        post  "rejection_reasons",         to: "rejection_reasons#create"
        patch "rejection_reasons/:id",     to: "rejection_reasons#update"
      end
    end
  ```

- [ ] **Step 7: Prestador, unidade e recusas do 21a.** Em `Exams::SaveProvider.call`, antes de `items = …`:

  ```ruby
        # ADR 0037 (Desvio 13): laboratório municipal, endereço e telefone (art. 167, II).
        if params.key?("uses_municipal_lab")
          return Result.fail(:invalid_uses_municipal_lab) unless [ true, false ].include?(params["uses_municipal_lab"])
          # Dashboard D20: marcar só com municipal_lab utilizável (lido fora da transação).
          return Result.fail(:municipal_lab_disabled) if params["uses_municipal_lab"] && !Lab::Gate.usable?(Current.city)

          attrs[:uses_municipal_lab] = params["uses_municipal_lab"]
        end
        if params.key?("address")
          address = params["address"]
          return Result.fail(:invalid_address) unless address.nil? || (address.is_a?(String) && address.strip.length.between?(1, 200))

          attrs[:address] = address&.strip
        end
        if params.key?("phone")
          phone = params["phone"]
          return Result.fail(:invalid_phone) unless phone.nil? || (phone.is_a?(String) && phone.match?(/\A\d{10,11}\z/))

          attrs[:phone] = phone
        end
  ```
  Em `Exams::AdminJson.provider`, acrescente `uses_municipal_lab: provider.uses_municipal_lab, address: provider.address, phone: provider.phone, has_middleware_token: LabMiddlewareToken.active.exists?(exam_provider_id: provider.id)`. Em `HealthUnitsController`: `error = assign_address(unit) || assign_cnes(unit) || assign_screening_scope(unit) || assign_collects_samples(unit)` (em `create` e `update`),

  ```ruby
    # ADR 0037: a unidade coleta amostras para o laboratório municipal.
    def assign_collects_samples(unit)
      return nil unless params.key?("collects_samples")
      return "invalid_collects_samples" unless [ true, false ].include?(params["collects_samples"])

      unit.collects_samples = params["collects_samples"]
      nil
    end
  ```
  e `collects_samples: unit.collects_samples` em `unit_json`. Em `Exams::ResultsController#create`, depois do `unit_staff?`: `return render(json: { error: "provider_uses_municipal_lab" }, status: :unprocessable_entity) if order.exam_provider&.uses_municipal_lab`. Em `Exams::Provider::OrdersController#result`, no começo: `return render(json: { error: "provider_uses_municipal_lab" }, status: :unprocessable_entity) if @provider.uses_municipal_lab`. Em `Exams::Fhir::Inbound.call`, primeira linha: `return Result.fail(:provider_uses_municipal_lab) if provider.uses_municipal_lab` (o controlador já manda razão fora de `STATUS` para 422).

- [ ] **Step 8: Rode:** `rspec spec/services/lab spec/requests/lab spec/requests/exams spec/requests/fhir spec/requests/health_units_spec.rb` → verde (se o arquivo de spec de unidades tiver outro nome, rode o de `spec/requests` que cobre `/attendance/units`).

- [ ] **Step 9: Commit** e mova o card **F-21.15** para **Done**:

  ```bash
  git add app/controllers/lab app/services/lab/access.rb app/services/lab/analytes.rb app/services/lab/formula.rb app/services/lab/admin_json.rb app/commands/lab config/routes.rb app/commands/exams/save_provider.rb app/services/exams/admin_json.rb app/controllers/health_units_controller.rb app/controllers/exams/results_controller.rb app/controllers/exams/provider/orders_controller.rb app/services/exams/fhir/inbound.rb spec/services/lab/formula_spec.rb spec/services/lab/analytes_spec.rb spec/requests/lab
  git commit -m "feat: add municipal lab catalog with immutable versions and rejection reasons

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 7: Liberadores (RT e substituto) e token do middleware

**Files:**
- Create: `app/commands/lab/{assign_releaser,rotate_middleware_token}.rb`, `app/controllers/lab/admin/{releasers_controller,middleware_tokens_controller}.rb`
- Modify: `app/commands/professionals/create.rb`, `app/services/lab/admin_json.rb`, `config/routes.rb`
- Test: `spec/requests/lab/admin_releasers_spec.rb`, `spec/commands/professionals/create_lab_releaser_spec.rb`

**Interfaces:**
- Consumes: `Lab::Admin::BaseController`, `Lab::Access::COUNCILS` (Task 6).
- Produces: `Lab::AssignReleaser.call(user_id:, provider:, technical_lead:, technical_deputy:, by:) → Result.ok(releaser:)` | `fail(:not_found|:user_missing_role|:council_missing|:lead_taken|:deputy_taken|:invalid_marks|:already_assigned)`; `Lab::AssignReleaser.end!(releaser:, by:)`; `Lab::RotateMiddlewareToken.call(provider:, by:, simulated: false) → Result.ok(token:)`, `.revoke(provider:)`; `Lab::AdminJson.releaser(r)` = `{ id, user_id, name, email, provider_id, council, council_uf, council_number, technical_lead, technical_deputy }`; rotas `GET|POST /lab/admin/releasers`, `DELETE /lab/admin/releasers/:id`, `POST|DELETE /lab/admin/providers/:id/middleware_token`.

- [ ] **Step 1: Testes que falham** — `spec/requests/lab/admin_releasers_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3: liberadores com conselho do laboratório; um RT e um substituto
  # ativos por laboratório; token do middleware mostrado uma vez, só o hash fica.
  RSpec.describe "Laboratório — liberadores e token", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:admin) { lab_admin! }

    def releaser_user(council:)
      staff_with("bio-#{SecureRandom.hex(3)}@cidade.gov.br", "lab_releaser").tap do |user|
        Professional.create!(user: user, professional_name: "Bio #{SecureRandom.hex(2)}", council: council, council_state: "PR",
                             registration_number: (SecureRandom.random_number(90_000) + 10_000).to_s, cns: Professionals::Cns.generate(user.id))
      end
    end

    it "liga com RT, recusa RT repetido, conselho fora e sem perfil; encerra" do
      stepped_up!(admin)
      bio = releaser_user(council: "CRBM")
      json_post "/lab/admin/releasers", user_id: bio.id, provider_id: provider.id, technical_lead: true, technical_deputy: false
      expect(response).to have_http_status(:created)
      expect(json_body).to include("user_id" => bio.id, "council" => "CRBM", "council_uf" => "PR", "technical_lead" => true, "email" => bio.email_address)
      other = releaser_user(council: "CRF")
      json_post "/lab/admin/releasers", user_id: other.id, provider_id: provider.id, technical_lead: true, technical_deputy: false
      expect(json_body["error"]).to eq("lead_taken")
      nurse = releaser_user(council: "COREN")
      json_post "/lab/admin/releasers", user_id: nurse.id, provider_id: provider.id, technical_lead: false, technical_deputy: false
      expect(json_body["error"]).to eq("council_missing")
      bare = staff_with("semperfil-#{SecureRandom.hex(2)}@cidade.gov.br", "lab_releaser")
      json_post "/lab/admin/releasers", user_id: bare.id, provider_id: provider.id, technical_lead: false, technical_deputy: false
      expect(json_body["error"]).to eq("council_missing")
      json_post "/lab/admin/releasers", user_id: staff_with("x-#{SecureRandom.hex(2)}@c.gov.br", "lab_staff").id, provider_id: provider.id,
                                        technical_lead: false, technical_deputy: false
      expect(json_body["error"]).to eq("user_missing_role")
      get "/lab/admin/releasers"
      id = json_body["releasers"].sole["id"]
      json_delete "/lab/admin/releasers/#{id}"
      expect(response).to have_http_status(:no_content)
      json_post "/lab/admin/releasers", user_id: other.id, provider_id: provider.id, technical_lead: true, technical_deputy: false
      expect(response).to have_http_status(:created)
      expect(DomainEvent.where(name: %w[lab_releaser.assigned lab_releaser.ended]).count).to eq(3)
    end

    it "token do middleware: uma vez, troca o anterior, revoga" do
      stepped_up!(admin)
      json_post "/lab/admin/providers/#{provider.id}/middleware_token"
      first = json_body["token"]
      expect(first.length).to be >= 40
      json_post "/lab/admin/providers/#{provider.id}/middleware_token"
      expect(LabMiddlewareToken.by_token(first)).to be_revoked
      expect(LabMiddlewareToken.active.where(exam_provider: provider).count).to eq(1)
      expect(LabMiddlewareToken.pluck(:token_digest)).not_to include(first)
      json_delete "/lab/admin/providers/#{provider.id}/middleware_token"
      expect(LabMiddlewareToken.active.where(exam_provider: provider)).to be_empty
      expect(DomainEvent.where(name: "lab_middleware.token_rotated").map(&:payload).first.keys).to contain_exactly("exam_provider_id")
    end
  end
  ```
  `spec/commands/professionals/create_lab_releaser_spec.rb`:

  ```ruby
  require "rails_helper"

  # Desvio 1: o liberador tem perfil profissional sem ser health_professional.
  RSpec.describe Professionals::Create do
    before { lab_city! }

    it "aceita lab_releaser; recusa quem não tem nenhum dos dois papéis" do
      user = staff_with("bio-#{SecureRandom.hex(3)}@cidade.gov.br", "lab_releaser")
      attrs = { professional_name: "Mariana Teles", council: "CRBM", council_state: "PR", registration_number: "12345",
                cns: Professionals::Cns.generate("bio") }
      expect(described_class.call(user_id: user.id, attrs: attrs, by: lab_admin!)).to be_ok
      staff = staff_with("staff-#{SecureRandom.hex(3)}@cidade.gov.br", "lab_staff")
      expect(described_class.call(user_id: staff.id, attrs: attrs.merge(registration_number: "999"), by: lab_admin!).reason).to eq(:user_missing_role)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/admin_releasers_spec.rb spec/commands/professionals/create_lab_releaser_spec.rb`.

- [ ] **Step 3: Implemente.** Em `Professionals::Create.call`: `return Result.fail(:user_missing_role) unless user.active? && (user.has_role?("health_professional") || user.has_role?("lab_releaser"))` (comentário "ADR 0037: o liberador do laboratório também tem perfil"). `app/commands/lab/assign_releaser.rb`:

  ```ruby
  # Liga um liberador a um laboratório (contrato §3): papel lab_releaser,
  # perfil com CRF/CRBM/CRM; um RT e um substituto ativos por laboratório.
  module Lab
    module AssignReleaser
      module_function

      def call(user_id:, provider:, technical_lead:, technical_deputy:, by:)
        user = user_id.is_a?(String) && User.find_by(id: user_id, deactivated_at: nil)
        return Result.fail(:not_found) unless user
        return Result.fail(:user_missing_role) unless user.has_role?("lab_releaser")
        return Result.fail(:council_missing) unless Access.council(user)
        return Result.fail(:invalid_marks) unless [ true, false ].include?(technical_lead) && [ true, false ].include?(technical_deputy)
        return Result.fail(:invalid_marks) if technical_lead && technical_deputy

        releaser = LabReleaser.create!(user: user, exam_provider: provider, technical_lead: technical_lead,
                                       technical_deputy: technical_deputy, created_by_user: by)
        DomainEvents.publish("lab_releaser.assigned", lab_releaser_id: releaser.id, exam_provider_id: provider.id)
        Result.ok(releaser: releaser)
      rescue ActiveRecord::RecordNotUnique => e
        Result.fail({ "idx_lab_releasers_lead" => :lead_taken, "idx_lab_releasers_deputy" => :deputy_taken }
                      .find { |index, _| e.message.include?(index) }&.last || :already_assigned)
      end

      def end!(releaser:, by:)
        releaser.update!(ended_at: Time.current, ended_by_user: by)
        DomainEvents.publish("lab_releaser.ended", lab_releaser_id: releaser.id, exam_provider_id: releaser.exam_provider_id)
        Result.ok(releaser: releaser)
      end
    end
  end
  ```
  `app/commands/lab/rotate_middleware_token.rb`:

  ```ruby
  # Token do middleware (contrato §3): mostrado uma vez; só o SHA-256 fica;
  # girar revoga o anterior; evento só com o id do laboratório.
  module Lab
    module RotateMiddlewareToken
      module_function

      def call(provider:, by:, simulated: false, token: SecureRandom.urlsafe_base64(32))
        ApplicationRecord.transaction do
          provider.lock!
          LabMiddlewareToken.active.where(exam_provider_id: provider.id).update_all(revoked_at: Time.current)
          LabMiddlewareToken.create!(exam_provider: provider, token_digest: LabMiddlewareToken.digest(token), simulated: simulated,
                                     created_by_user: by, created_at: Time.current)
          DomainEvents.publish("lab_middleware.token_rotated", exam_provider_id: provider.id)
        end
        Result.ok(token: token)
      end

      def revoke(provider:)
        revoked = LabMiddlewareToken.active.where(exam_provider_id: provider.id).update_all(revoked_at: Time.current)
        DomainEvents.publish("lab_middleware.token_revoked", exam_provider_id: provider.id) if revoked.positive?
        Result.ok
      end
    end
  end
  ```
  Em `Lab::AdminJson`:

  ```ruby
      def releaser(r)
        professional = r.user.professional
        { id: r.id, user_id: r.user_id, name: Screenings::Json.staff_name(r.user), email: r.user.email_address,
          provider_id: r.exam_provider_id, council: professional&.council, council_uf: professional&.council_state,
          council_number: professional&.registration_number, technical_lead: r.technical_lead, technical_deputy: r.technical_deputy }
      end
  ```
  `app/controllers/lab/admin/releasers_controller.rb`:

  ```ruby
  module Lab
    module Admin
      class ReleasersController < BaseController
        def index
          rows = LabReleaser.active.joins(:exam_provider).merge(Lab::Access.labs).includes(user: :professional).order(:created_at)
          render json: { releasers: rows.map { |r| Lab::AdminJson.releaser(r) } }
        end

        def create
          admin_provider!(params[:provider_id])
          return if performed?

          result = Lab::AssignReleaser.call(user_id: params[:user_id], provider: @provider, technical_lead: params[:technical_lead],
                                            technical_deputy: params[:technical_deputy], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Lab::AdminJson.releaser(result.payload[:releaser]), status: :created
        end

        def destroy
          releaser = uuid?(params[:id]) && LabReleaser.active.find_by(id: params[:id])
          return not_found unless releaser

          Lab::AssignReleaser.end!(releaser: releaser, by: Current.user)
          head :no_content
        end
      end
    end
  end
  ```
  (`admin_provider!` renderiza 422 quando falha; `performed?` evita renderizar duas vezes.) `app/controllers/lab/admin/middleware_tokens_controller.rb`:

  ```ruby
  module Lab
    module Admin
      class MiddlewareTokensController < BaseController
        before_action { admin_provider!(params[:id]) }

        def create
          render json: { token: Lab::RotateMiddlewareToken.call(provider: @provider, by: Current.user).payload[:token] }, status: :created
        end

        def destroy
          Lab::RotateMiddlewareToken.revoke(provider: @provider)
          head :no_content
        end
      end
    end
  end
  ```
  Rotas, dentro de `scope "admin"` do `/lab`:

  ```ruby
        get    "releasers",                        to: "releasers#index"
        post   "releasers",                        to: "releasers#create"
        delete "releasers/:id",                    to: "releasers#destroy"
        post   "providers/:id/middleware_token",   to: "middleware_tokens#create"
        delete "providers/:id/middleware_token",   to: "middleware_tokens#destroy"
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/admin_releasers_spec.rb spec/commands/professionals` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/commands/lab/assign_releaser.rb app/commands/lab/rotate_middleware_token.rb app/controllers/lab/admin/releasers_controller.rb app/controllers/lab/admin/middleware_tokens_controller.rb app/commands/professionals/create.rb app/services/lab/admin_json.rb config/routes.rb spec/requests/lab/admin_releasers_spec.rb spec/commands/professionals/create_lab_releaser_spec.rb
  git commit -m "feat: add lab releasers with technical lead marks and middleware token

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 3 — Requisição, etiqueta e coleta (F-21.16)

### Task 8: Requisição (com pedido em papel), leitura e situação

**Files:**
- Create: `app/commands/lab/open_requisition.rb`, `app/services/lab/{json,requisition_status}.rb`, `app/controllers/lab/requisitions_controller.rb`
- Modify: `app/controllers/lab/lookups_controller.rb` (`candidate_orders`, `exams`), `app/services/exams/transition.rb` (`FLOW["scheduled"]`), `app/services/exams/steps.rb` (`schedule` recusa `collected`; `cancel` cuida do item), `app/controllers/exams/lookups_controller.rb` (`lab_staff`), `config/routes.rb`, `spec/support/lab_helpers.rb` (`open_requisition!`)
- Test: `spec/requests/lab/requisitions_spec.rb`, `spec/services/exams/steps_lab_spec.rb`

**Interfaces:**
- Consumes: `Lab::Access`, `Lab::BaseController` (Task 6); `Exams::Open`, `Exams::Transition`, `Exams::Cursor` (21a).
- Produces:
  - `Lab::OpenRequisition.call(params, by:, now: Time.current) → Result.ok(requisition:)` | `fail(:order_not_eligible|:exam_not_in_lab_catalog|:mother_name_required|:biological_sex_required|:invalid_collection_place|:unit_does_not_collect|:not_unit_staff|:missing_role|:invalid_requester|:invalid_cns|:invalid_orders|:invalid_provider|:not_found)`.
  - `Lab::RequisitionStatus.refresh!(requisition)`.
  - `Lab::Json.requisition(req, viewer:, full: true)`, `.item(item)`, `.sample(sample)`, `.report_summary(report, viewer:)`, `.lab_member?(viewer, req)`.
  - `Lab::RequisitionsController#index|show|create`; `GET /lab/candidate_orders?patient_id=&provider_id=` → `{ "orders": [ { id, short_code, sigtap_code, label, priority, status, requester_name } ] }` (dashboard D2); `GET /lab/exams?provider_id=` → `{ "exams": [ { sigtap_code, name, material, delivery_days } ] }` (D3).
  - `Exams::Transition::FLOW["scheduled"][:from]` inclui `collected` (Desvio 3).
  - Helper `LabHelpers#open_requisition!(provider:, exams:, unit: nil, place: "lab", by: nil) → LabRequisition`.

- [ ] **Step 1: Testes que falham** — `spec/requests/lab/requisitions_spec.rb`:

  ```ruby
  require "rails_helper"
  require "json_schemer"

  # Contrato §4 (F-21.16): a requisição junta exames do 21a do paciente; papel
  # vira lab_reception; uma amostra por material/recipiente; no 21a o exame vai
  # a `scheduled` no laboratório; previsão = maior prazo de entrega.
  RSpec.describe "Laboratório — requisição", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider!(name: "Laboratório Municipal Central") }
    let!(:glucose) { lab_exam!(provider: provider, delivery_days: 2) }
    let!(:lipids) do
      lab_exam!(provider: provider, sigtap_code: "0202010295", name: "Perfil lipídico", analytes: LabHelpers::LIPIDS,
                material: "Soro", container: "Tubo amarelo (gel)", instrument_test_code: "LIP", delivery_days: 4)
    end
    let(:unit) { create_unit("UBS Jardim").tap { |u| u.update!(collects_samples: true) } }
    let(:doctor) { doctor!(unit) }
    let(:patient) { exam_patient!(exam_seq!).first }
    let(:staff) { lab_staff! }
    let(:order_glucose) { exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: "0202010473") }
    let(:order_lipids) { exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: "0202010295") }

    def body(**over)
      { patient_id: patient.id, provider_id: provider.id, origin_unit_id: unit.id, collection_place: "lab", biological_sex: "female",
        mother_name: "Joana da Silva", urgent: false, clinical_info: "Em uso de metformina", exam_order_ids: [ order_glucose.id, order_lipids.id ] }.merge(over)
    end

    it "abre no posto do laboratório: duas amostras, itens, 21a agendado no laboratório, previsão pelo maior prazo" do
      sign_in_as(staff)
      get "/lab/candidate_orders", params: { patient_id: patient.id, provider_id: provider.id }
      expect(json_body["orders"].map { |o| o["id"] }).to contain_exactly(order_glucose.id, order_lipids.id)
      expect(json_body["orders"].first.keys).to contain_exactly(*%w[id short_code sigtap_code label priority status requester_name])
      json_post "/lab/requisitions", body
      expect(response).to have_http_status(:created)
      req = json_body
      expect(req).to include("number" => match(/\A\d{8}\z/), "collection_place" => "lab", "status" => "open",
                             "expected_delivery_on" => (Time.zone.today + 4).iso8601)
      expect(req["patient"]).to include("id" => patient.id, "biological_sex" => "female")
      expect(req["samples"].map { |s| [ s["material"], s["barcode"] ] })
        .to contain_exactly([ "Plasma", "#{req['number']}01" ], [ "Soro", "#{req['number']}02" ])
      expect(req["items"].map { |i| i["label"] }).to contain_exactly("Glicose", "Perfil lipídico")
      expect(order_glucose.reload).to have_attributes(status: "scheduled", exam_provider_id: provider.id)
      expect(LabRequisition.find(req["id"]).mother_name).to eq("Joana da Silva")
      expect(DomainEvent.where(name: "lab_requisition.opened").sole.payload.keys).to contain_exactly("lab_requisition_id", "collection_place", "urgent")
      get "/lab/candidate_orders", params: { patient_id: patient.id, provider_id: provider.id }
      expect(json_body["orders"]).to eq([])
      json_post "/lab/requisitions", body
      expect(json_body["error"]).to eq("order_not_eligible")
    end

    it "na UBS que coleta, pela enfermagem com vínculo; UBS que não coleta recusa" do
      sign_in_as(doctor)
      json_post "/lab/requisitions", body(collection_place: "ubs")
      expect(response).to have_http_status(:created)
      other = create_unit("UBS Sem Coleta")
      sign_in_as(doctor!(other))
      json_post "/lab/requisitions", body(collection_place: "ubs", origin_unit_id: other.id, exam_order_ids: [ exam_order!(patient: patient, unit: other, requester: doctor, sigtap_code: "0202010473").id ])
      expect(json_body["error"]).to eq("unit_does_not_collect")
      sign_in_as(doctor)
      json_post "/lab/requisitions", body(collection_place: "lab")
      expect(json_body["error"]).to eq("missing_role")
    end

    it "pedido em papel vira exame lab_reception; erros do corpo" do
      sign_in_as(staff)
      json_post "/lab/requisitions", body(exam_order_ids: [], paper_orders: [ { sigtap_code: "0202010473", requester: { name: "Dr. Externo", cns: Professionals::Cns.generate("externo") } } ])
      expect(response).to have_http_status(:created)
      order = ExamOrder.find(json_body["items"].sole["exam_order_id"])
      expect(order).to have_attributes(origin: "lab_reception", requester_user_id: nil, unit_id: unit.id, status: "scheduled")
      { { mother_name: " " } => "mother_name_required", { biological_sex: "x" } => "biological_sex_required",
        { exam_order_ids: [], paper_orders: [ { sigtap_code: "0202020380", requester: { user_id: doctor.id } } ] } => "exam_not_in_lab_catalog",
        { exam_order_ids: [], paper_orders: [ { sigtap_code: "0202010473", requester: { name: "X", cns: "123" } } ] } => "invalid_cns",
        { exam_order_ids: [], paper_orders: [] } => "invalid_orders",
        { collection_place: "casa" } => "invalid_collection_place" }.each do |over, error|
        json_post "/lab/requisitions", body(**over)
        expect(json_body["error"]).to eq(error), over.inspect
      end
    end

    it "lista e detalhe: laboratório vê os seus; UBS vê os da unidade; estranho 403; filtros por número e código de barras" do
      req = open_requisition!(provider: provider, exams: [ glucose ], unit: unit, by: staff)
      sign_in_as(staff)
      get "/lab/requisitions", params: { number: req.number }
      expect(json_body["requisitions"].sole).to include("id" => req.id, "status" => "open")
      expect(json_body["requisitions"].sole.keys).not_to include("items", "samples")
      get "/lab/requisitions", params: { barcode: req.samples.first.barcode }
      expect(json_body["requisitions"].size).to eq(1)
      sign_in_as(doctor)
      get "/lab/requisitions/#{req.id}"
      expect(json_body["number"]).to eq(req.number)
      get "/lab/requisitions", params: { origin_unit_id: unit.id }
      expect(json_body["requisitions"].map { |r| r["id"] }).to eq([ req.id ])
      sign_in_as(doctor!(create_unit("UBS Longe")))
      get "/lab/requisitions/#{req.id}"
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```
  `spec/services/exams/steps_lab_spec.rb`:

  ```ruby
  require "rails_helper"

  # Desvio 3 e 16: só o laboratório reabre a coleta; cancelar exame com item
  # pendente cancela o item; com item digitado, 409.
  RSpec.describe Exams::Steps do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider) }
    let(:req) { open_requisition!(provider: provider, exams: [ exam ]) }
    let(:item) { req.items.first }
    let(:order) { item.exam_order }
    let(:doctor) { order.requester_user }

    it "a unidade não reagenda exame coletado" do
      Exams::Transition.call(order: order, kind: "collected", actor_user: doctor)
      result = described_class.schedule(order: order, actor_user: doctor,
                                        params: { provider_id: provider.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "X" })
      expect([ result.reason, result.details[:status] ]).to eq([ :invalid_transition, "collected" ])
    end

    it "cancelar com item pendente cancela o item; com item digitado, 409" do
      expect(described_class.cancel(order: order, reason: "Desistência", actor_user: doctor)).to be_ok
      expect(item.reload.status).to eq("cancelled")
      other = open_requisition!(provider: provider, exams: [ exam ]).items.first
      other.update!(status: "typed")
      expect(described_class.cancel(order: other.exam_order, reason: "Desistência", actor_user: doctor).reason).to eq(:invalid_transition)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/requisitions_spec.rb spec/services/exams/steps_lab_spec.rb`.

- [ ] **Step 3: 21a.** Em `Exams::Transition::FLOW`: `"scheduled" => { from: %w[awaiting_schedule scheduled collected], to: "scheduled" }` (comentário: "ADR 0037: a recoleta do laboratório reabre a coleta; as rotas da unidade e do portal recusam `collected` em Steps.schedule"). Em `Exams::Steps.schedule`, dentro do `locked(order) do`, primeira linha: `next Result.fail(:invalid_transition, details: { status: order.status }) if order.status == "collected"`. `Exams::Steps.cancel` vira:

  ```ruby
      # ADR 0037 (Desvio 16): exame na requisição do laboratório — item pendente
      # é cancelado junto; digitado ou liberado, o laboratório já trabalhou: 409.
      def cancel(order:, reason:, actor_user:)
        locked(order) do
          item = LabRequisitionItem.lock.find_by(exam_order_id: order.id)
          next Result.fail(:invalid_transition, details: { status: order.status }) if item && %w[typed released].include?(item.status)

          moved = Transition.call(order: order, kind: "cancelled", actor_user: actor_user, reason: reason)
          if moved.ok? && item
            item.update!(status: "cancelled")
            Lab::RequisitionStatus.refresh!(item.lab_requisition)
          end
          moved
        end
      end
  ```
  Em `Exams::LookupsController`: `LOOKUP_ROLES = %w[citizen_verifier health_professional municipal_admin lab_staff]` e, em `requesters`, `return forbid("not_unit_staff") unless Exams::Access.unit_staff?(Current.user, unit_id) || Exams::Access.regulator?(Current.user) || Lab::Access.staff?(Current.user)`.

- [ ] **Step 4: Situação e formas.** `app/services/lab/requisition_status.rb`:

  ```ruby
  # Situação da requisição (spec §5, item 11), recalculada dos itens e das
  # amostras a cada passo: open → in_analysis → typed → released → delivered.
  module Lab
    module RequisitionStatus
      module_function

      def refresh!(requisition)
        items = requisition.items.reload.select(&:active?)
        status = if requisition.delivered_at then "delivered"
                 elsif items.any? && items.all? { |i| i.status == "released" } then "released"
                 elsif items.any? && items.all? { |i| %w[typed released].include?(i.status) } then "typed"
                 elsif requisition.samples.reload.any?(&:accepted?) then "in_analysis"
                 else "open"
                 end
        requisition.update!(status: status) unless requisition.status == status
        requisition
      end
    end
  end
  ```
  `app/services/lab/json.rb`:

  ```ruby
  # Formas do contrato §2 (laboratório). Sem valor de resultado aqui: valores
  # só em Lab::EntryJson (Task 11) e na fila de liberação (Task 17), que gravam trilha.
  module Lab
    module Json
      module_function

      def lab_member?(viewer, req) = Access.member?(viewer, req.exam_provider_id)

      def requisition(req, viewer:, full: true)
        patient = req.patient
        body = { id: req.id, number: req.number,
                 patient: { id: patient.id, display_name: patient.display_name, birth_date: patient.birth_date, biological_sex: req.biological_sex },
                 origin_unit: { id: req.origin_unit_id, name: req.origin_unit.name }, collection_place: req.collection_place,
                 urgent: req.urgent, status: req.status, expected_delivery_on: req.expected_delivery_on&.iso8601,
                 reports: req.reports.map { |r| report_summary(r, viewer: viewer) },
                 delivered_to: req.delivered_to, delivered_at: req.delivered_at&.iso8601 }
        return body unless full

        body.merge(items: req.items.includes(:exam_order, lab_exam_version: :lab_exam).map { |i| item(i) },
                   samples: req.samples.includes(:collected_by_user, :lab_rejection_reason).map { |s| sample(s) })
      end

      def item(i)
        version = i.lab_exam_version
        { id: i.id, exam_order_id: i.exam_order_id, sigtap_code: version.lab_exam.sigtap_code, label: version.lab_exam.name,
          version: version.version, sample_id: i.lab_sample_id, status: i.status }
      end

      def sample(s)
        { id: s.id, barcode: s.barcode, material: s.material, container: s.container, status: s.status,
          collected_at: s.collected_at&.iso8601, collected_by: Screenings::Json.staff_name(s.collected_by_user),
          received_at: s.received_at&.iso8601, rejection_reason: s.lab_rejection_reason&.name, restriction_note: s.restriction_note,
          recollection_of_id: s.recollection_of_id, shipment_id: s.lab_shipment_id }
      end

      # pdf_url (A4 do 21a) só para quem é do laboratório e só com PDF.
      def report_summary(report, viewer:)
        url = report.pdf_file && Access.member?(viewer, report.exam_provider_id) ? Exams::Files.url_for(report.pdf_file, user: viewer) : nil
        { id: report.id, version: report.version, partial: report.partial, status: report.status, signature_kind: report.signature_kind,
          released_at: report.released_at&.iso8601, released_by: Screenings::Json.staff_name(report.released_by_user),
          short_code: report.released_at ? report.short_code : nil, pdf_url: url, replaces_report_id: report.replaces_report_id }
      end
    end
  end
  ```

- [ ] **Step 5: Abrir a requisição.** `app/commands/lab/open_requisition.rb`:

  ```ruby
  # Abre a requisição (contrato §4; spec §5.1). Quem: lab_staff (posto do
  # laboratório, `lab`) ou profissional com vínculo na UBS de origem que coleta
  # (`ubs`). Exames do 21a do paciente (awaiting_schedule/scheduled, sem item),
  # mais pedidos em papel (viram exam_order lab_reception). Numa transação:
  # número, amostras (uma por material/recipiente da versão vigente), itens e,
  # no 21a, `scheduled` no laboratório. O índice único de exam_order_id fecha
  # a corrida de duas requisições com o mesmo exame.
  module Lab
    module OpenRequisition
      MAX_ORDERS = 30

      module_function

      def call(params, by:, now: Time.current)
        p = params.to_h.stringify_keys
        provider = p["provider_id"].is_a?(String) && Access.labs.find_by(id: p["provider_id"])
        return Result.fail(:invalid_provider) unless provider

        unit = p["origin_unit_id"].is_a?(String) && HealthUnit.find_by(id: p["origin_unit_id"], active: true)
        return Result.fail(:not_found) unless unit

        place = p["collection_place"]
        return Result.fail(:invalid_collection_place) unless LabRequisition::PLACES.include?(place)

        allowed = authorize(by, provider, unit, place)
        return allowed if allowed.is_a?(Result)

        patient = p["patient_id"].is_a?(String) && Patient.find_by(id: p["patient_id"])
        return Result.fail(:not_found) unless patient
        return Result.fail(:biological_sex_required) unless LabRequisition::SEXES.include?(p["biological_sex"])

        mother = p["mother_name"].is_a?(String) ? p["mother_name"].strip : ""
        return Result.fail(:mother_name_required) unless mother.length.between?(1, 120)

        info = p["clinical_info"].is_a?(String) ? p["clinical_info"].strip.presence : nil
        return Result.fail(:invalid_clinical_info) if info && info.length > 1000
        return Result.fail(:invalid_urgent) unless [ true, false ].include?(p.fetch("urgent", false))

        ids = Array(p["exam_order_ids"])
        papers = Array(p["paper_orders"])
        return Result.fail(:invalid_orders) if (ids.empty? && papers.empty?) || ids.size + papers.size > MAX_ORDERS
        return Result.fail(:invalid_orders) unless ids.all? { |id| id.is_a?(String) && id.match?(Exams::BaseController::UUID) }

        exams = LabExam.where(exam_provider_id: provider.id, active: true).where.not(current_version_id: nil).includes(:current_version)
                       .index_by(&:sigtap_code)
        paper_specs = papers.map { |paper| paper_spec(paper, exams) }
        failed = paper_specs.find { |spec| spec.is_a?(Result) }
        return failed if failed

        result = nil
        ApplicationRecord.transaction do
          orders = ExamOrder.lock.where(id: ids, patient_id: patient.id).to_a
          next result = Result.fail(:order_not_eligible) unless orders.size == ids.uniq.size && orders.all? { |o| eligible?(o) }
          next result = Result.fail(:exam_not_in_lab_catalog) unless orders.all? { |o| exams.key?(o.sigtap_code) }

          paper_specs.each do |spec|
            opened = Exams::Open.call(patient: patient, sigtap_code: spec[:sigtap_code], priority: p["urgent"] ? "urgent" : "routine",
                                      unit: unit, origin: "lab_reception", created_by: by, requester_user: spec[:user],
                                      requester_name: spec[:name], requester_cns: spec[:cns], require_catalog: false, now: now)
            raise ActiveRecord::Rollback if opened.failure? && (result = opened)

            orders << opened.payload[:order]
          end
          requisition = build!(provider, unit, patient, place, p, mother, info, orders, exams, by, now)
          DomainEvents.publish("lab_requisition.opened", lab_requisition_id: requisition.id, collection_place: place, urgent: requisition.urgent)
          result = Result.ok(requisition: requisition)
        end
        result
      rescue ActiveRecord::RecordNotUnique
        Result.fail(:order_not_eligible)
      end

      def authorize(user, provider, unit, place)
        if place == "lab"
          return Result.fail(:missing_role) unless Access.staff?(user) && Access.member?(user, provider.id)
        else
          return Result.fail(:unit_does_not_collect) unless unit.collects_samples
          return Result.fail(:missing_role) unless Access.collecting_unit_ids(user).include?(unit.id)
        end
        true
      end

      def eligible?(order)
        %w[awaiting_schedule scheduled].include?(order.status) && !LabRequisitionItem.exists?(exam_order_id: order.id)
      end

      # { sigtap_code, requester: { user_id } | { name, cns } } → spec ou falha.
      def paper_spec(raw, exams)
        raw = raw.respond_to?(:to_unsafe_h) ? raw.to_unsafe_h.stringify_keys : (raw.is_a?(Hash) ? raw.stringify_keys : {})
        return Result.fail(:exam_not_in_lab_catalog) unless exams.key?(raw["sigtap_code"])

        item = ExamCatalogItem.find_by(sigtap_code: raw["sigtap_code"], active: true)
        return Result.fail(:order_not_eligible) if item&.requires_authorization

        requester = raw["requester"].is_a?(Hash) ? raw["requester"].stringify_keys : {}
        if requester["user_id"]
          user = User.find_by(id: requester["user_id"], deactivated_at: nil)
          return Result.fail(:invalid_requester) unless user&.has_role?("health_professional")

          { sigtap_code: raw["sigtap_code"], user: user }
        else
          name = requester["name"].is_a?(String) ? requester["name"].strip : ""
          return Result.fail(:invalid_requester) unless name.length.between?(1, 120)
          return Result.fail(:invalid_cns) unless Professionals::Cns.valid?(requester["cns"].to_s)

          { sigtap_code: raw["sigtap_code"], name: name, cns: requester["cns"].to_s }
        end
      end

      def build!(provider, unit, patient, place, p, mother, info, orders, exams, by, now)
        days = orders.map { |o| exams.fetch(o.sigtap_code).delivery_days }.max
        requisition = LabRequisition.create!(number: LabRequisition.next_number, patient: patient, exam_provider: provider, origin_unit: unit,
                                             collection_place: place, biological_sex: p["biological_sex"], mother_name: mother,
                                             urgent: p.fetch("urgent", false), clinical_info: info,
                                             expected_delivery_on: now.in_time_zone.to_date + days, opened_by_user: by)
        groups = orders.group_by { |o| v = exams.fetch(o.sigtap_code).current_version; [ v.material, v.container ] }
        raise ArgumentError, "amostras demais" if groups.size > LabSample::MAX_PER_REQUISITION

        place_text = place == "lab" ? "#{provider.name} — coleta no laboratório" : "#{unit.name} — coleta na unidade"
        groups.each_with_index do |((material, container), group), index|
          sample = LabSample.create!(lab_requisition: requisition, barcode: LabSample.barcode_for(requisition.number, index + 1),
                                     material: material, container: container)
          group.each do |order|
            LabRequisitionItem.create!(lab_requisition: requisition, exam_order: order, lab_exam_version: exams.fetch(order.sigtap_code).current_version,
                                       lab_sample: sample)
            moved = Exams::Transition.call(order: order, kind: "scheduled", actor_user: by, now: now,
                                           changes: { exam_provider_id: provider.id, scheduled_at: now, scheduled_place: place_text })
            raise ActiveRecord::RecordNotUnique, "transição recusada" if moved.failure?
          end
        end
        requisition
      end
      private_class_method :authorize, :eligible?, :paper_spec, :build!
    end
  end
  ```
  (A recusa da transição dentro de `build!` só acontece em corrida — o exame mudou depois da trava; virar `RecordNotUnique` faz o comando responder `order_not_eligible` e desfazer tudo.)

- [ ] **Step 6: Controlador e rotas.** `app/controllers/lab/requisitions_controller.rb`:

  ```ruby
  # Requisição (contrato §4): abrir, listar com recorte, ler. A UBS só lê as
  # da unidade dela; o laboratório, as dele.
  module Lab
    class RequisitionsController < BaseController
      before_action :load_requisition, only: %i[show]

      def index
        scope = LabRequisition.includes(:patient, :origin_unit, reports: [ :released_by_user, :pdf_file ])
        ids = Lab::Access.provider_ids(Current.user)
        if ids.any?
          scope = scope.where(exam_provider_id: ids)
          scope = scope.where(exam_provider_id: params[:provider_id]) if params[:provider_id].present?
          scope = scope.where(origin_unit_id: params[:origin_unit_id]) if params[:origin_unit_id].present?
        else
          unit_id = params[:origin_unit_id]
          return render(json: { error: "filter_required" }, status: :unprocessable_entity) unless uuid?(unit_id)
          return forbid("not_unit_staff") unless Exams::Access.unit_staff?(Current.user, unit_id)

          scope = scope.where(origin_unit_id: unit_id)
        end
        statuses = Array(params[:status])
        return render(json: { error: "invalid_filter" }, status: :unprocessable_entity) unless (statuses - LabRequisition::STATUSES).empty?

        scope = scope.where(status: statuses) if statuses.any?
        scope = scope.where(number: params[:number].to_s) if params[:number].present?
        scope = scope.where(id: LabSample.where(barcode: params[:barcode].to_s).select(:lab_requisition_id)) if params[:barcode].present?
        cursor = Exams::Cursor.decode(params[:before])
        scope = scope.where("(lab_requisitions.created_at, lab_requisitions.id) < (?, ?)", *cursor) if cursor
        limit = Timeline::Query.limit(params[:limit])
        rows = scope.order(created_at: :desc, id: :desc).limit(limit + 1).to_a
        page = rows.first(limit)
        render json: { requisitions: page.map { |r| Lab::Json.requisition(r, viewer: Current.user, full: false) },
                       next_before: rows.size > limit ? Exams::Cursor.encode(page.last) : nil }
      rescue Exams::Cursor::Invalid
        render json: { error: "invalid_cursor" }, status: :unprocessable_entity
      end

      def show = render(json: Lab::Json.requisition(@requisition, viewer: Current.user))

      def create
        result = Lab::OpenRequisition.call(body_params, by: Current.user)
        return render_failure(result) if result.failure?

        render json: Lab::Json.requisition(result.payload[:requisition].reload, viewer: Current.user), status: :created
      end

      private

      def load_requisition
        @requisition = uuid?(params[:id]) && LabRequisition.find_by(id: params[:id])
        return not_found unless @requisition

        readable = Lab::Access.member?(Current.user, @requisition.exam_provider_id) ||
                   Exams::Access.unit_staff?(Current.user, @requisition.origin_unit_id)
        forbid("not_unit_staff") unless readable
      end
    end
  end
  ```
  Em `Lab::LookupsController`: `before_action :provider_param!, only: %i[rejection_reasons candidate_orders exams]` e:

  ```ruby
      # Dashboard D2: exames do 21a do paciente que cabem numa requisição deste laboratório.
      def candidate_orders
        patient = uuid?(params[:patient_id]) && Patient.find_by(id: params[:patient_id])
        return not_found unless patient

        codes = LabExam.where(exam_provider_id: @provider.id, active: true).where.not(current_version_id: nil).pluck(:sigtap_code)
        orders = ExamOrder.where(patient_id: patient.id, status: %w[awaiting_schedule scheduled], sigtap_code: codes)
                          .where.not(id: LabRequisitionItem.select(:exam_order_id)).includes(:requester_user).order(:created_at)
        render json: { orders: orders.map do |o|
          { id: o.id, short_code: o.short_code, sigtap_code: o.sigtap_code, label: Exams::Labels.exam(o.sigtap_code, o.sigtap_competence),
            priority: o.priority, status: o.status, requester_name: o.requester_display_name }
        end }
      end

      # Dashboard D3: exames ativos do laboratório (pedido em papel).
      def exams
        exams = LabExam.where(exam_provider_id: @provider.id, active: true).where.not(current_version_id: nil).includes(:current_version).order(:name)
        render json: { exams: exams.map { |e| { sigtap_code: e.sigtap_code, name: e.name, material: e.current_version.material, delivery_days: e.delivery_days } } }
      end
  ```
  e em `provider_param!` (Task 6) a UBS que coleta também passa: troque a condição por `Lab::Access.member?(Current.user, @provider.id) || Lab::Access.admin?(Current.user) || Lab::Access.collecting_unit_ids(Current.user).any?`. Rotas, no `scope "/lab"` antes do `scope "admin"`:

  ```ruby
      get  "candidate_orders",  to: "lookups#candidate_orders"
      get  "exams",             to: "lookups#exams"
      get  "requisitions",      to: "requisitions#index"
      post "requisitions",      to: "requisitions#create"
      get  "requisitions/:id",  to: "requisitions#show"
  ```
  Em `spec/support/lab_helpers.rb`:

  ```ruby
    # Requisição pelo caminho real (posto do laboratório por padrão).
    def open_requisition!(provider:, exams:, unit: nil, place: "lab", by: nil, patient: nil)
      unit ||= create_unit("UBS #{SecureRandom.hex(2)}").tap { |u| u.update!(collects_samples: true) }
      patient ||= exam_patient!(exam_seq!).first
      requester = doctor!(unit)
      orders = exams.map { |exam| exam_order!(patient: patient, unit: unit, requester: requester, sigtap_code: exam.sigtap_code) }
      by ||= place == "lab" ? lab_staff! : requester
      result = Lab::OpenRequisition.call({ patient_id: patient.id, provider_id: provider.id, origin_unit_id: unit.id, collection_place: place,
                                           biological_sex: "female", mother_name: "Joana da Silva", urgent: false,
                                           exam_order_ids: orders.map(&:id) }, by: by)
      raise "requisição recusada: #{result.reason}" if result.failure?

      result.payload[:requisition]
    end
  ```

- [ ] **Step 7: Rode:** `rspec spec/requests/lab/requisitions_spec.rb spec/services/exams/steps_lab_spec.rb spec/requests/exams spec/services/exams` → verde.

- [ ] **Step 8: Commit:**

  ```bash
  git add app/commands/lab/open_requisition.rb app/services/lab/json.rb app/services/lab/requisition_status.rb app/controllers/lab/requisitions_controller.rb app/controllers/lab/lookups_controller.rb app/controllers/lab/base_controller.rb app/services/exams/transition.rb app/services/exams/steps.rb app/controllers/exams/lookups_controller.rb config/routes.rb spec/support/lab_helpers.rb spec/requests/lab/requisitions_spec.rb spec/services/exams/steps_lab_spec.rb
  git commit -m "feat: open lab requisitions over exam orders, with paper orders and samples

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 9: Etiqueta Code 128, comprovante e coleta

**Files:**
- Modify: `Gemfile`, `Gemfile.lock` (`barby ~> 0.7`)
- Create: `app/services/lab/{code128,labels_pdf,receipt_pdf}.rb`, `app/commands/lab/collect_sample.rb`, `app/controllers/lab/samples_controller.rb`
- Modify: `app/controllers/lab/requisitions_controller.rb` (`receipt`, `labels`), `config/routes.rb`
- Test: `spec/services/lab/labels_pdf_spec.rb`, `spec/requests/lab/labels_and_collection_spec.rb`

**Interfaces:**
- Consumes: `Lab::Json.sample`, `Lab::Access`, `open_requisition!` (Task 8).
- Produces: `Lab::Code128.encoding(value) → "1101…"` (só dígitos, comprimento par; `ArgumentError` fora disso); `Lab::LabelsPdf.call(samples) → String` (PDF); `Lab::LabelsPdf::{WIDTH,HEIGHT,MODULE,QUIET}`; `Lab::ReceiptPdf.call(requisition) → String`; `Lab::CollectSample.call(sample:, by:, collected_at: nil, now: Time.current) → Result.ok(sample:)` | `fail(:invalid_transition, details: { status: })|:missing_role|:invalid_collected_at`; `Lab::CollectSample.collector?(user, sample)`; rotas `GET /lab/requisitions/:id/receipt.pdf`, `GET /lab/requisitions/:id/labels.pdf?sample_ids[]=`, `POST /lab/samples/:id/collect`, `POST /lab/samples/collect { barcode }` → `<lab_sample>` + `requisition_number`.

- [ ] **Step 1: A gem.** Em `Gemfile`, junto do `rqrcode_core`: `gem "barby", "~> 0.7" # ADR 0037: só o codificador Code 128 da etiqueta de tubo (desenhado em vetor no Prawn)`. Rode `docker compose exec -T -w /rails/.claude/mod21b api bundle install` e confira no `Gemfile.lock` `barby (0.7.0)` sem outra dependência nova.

- [ ] **Step 2: Testes que falham** — `spec/services/lab/labels_pdf_spec.rb`:

  ```ruby
  require "rails_helper"
  require "pdf/reader"

  # Review Focus 5 (etiqueta no limite) e Valores fixados 4: 50 x 25 mm, uma
  # por página, Code 128 de 10 dígitos com zona de silêncio, texto fora da barra.
  RSpec.describe Lab::LabelsPdf do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:req) { open_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }

    it "uma etiqueta por página, no tamanho do tubo, com o número e os quatro campos" do
      req.patient.update!(full_name: "JOSÉ DE ASSUNÇÃO PEREIRA DOS SANTOS ALBUQUERQUE NETO DA CONCEIÇÃO E SILVA FILHO", birth_date: nil)
      sample = req.samples.first
      reader = PDF::Reader.new(StringIO.new(described_class.call([ sample, sample ])))
      expect(reader.page_count).to eq(2)
      expect(reader.pages.first.attributes[:MediaBox].map { |n| n.round(2) }).to eq([ 0, 0, 141.73, 70.87 ])
      text = reader.pages.first.text
      expect(text).to include(sample.barcode, "Req. #{req.number}", "Nasc. não informado", "Plasma")
      expect(text).to include("JOSÉ DE ASSUNÇÃO")
    end

    it "o código cabe com as zonas de silêncio; só dígitos e comprimento par" do
      bars = Lab::Code128.encoding("9999999999")
      expect(bars.length).to eq(90)
      expect((bars.length + (2 * described_class::QUIET)) * described_class::MODULE).to be <= described_class::WIDTH
      expect { Lab::Code128.encoding("12345") }.to raise_error(ArgumentError)
      expect { Lab::Code128.encoding("12ab") }.to raise_error(ArgumentError)
      expect(LabSample.barcode_for("00000123", 99)).to eq("0000012399")
      expect { LabSample.barcode_for("00000123", 100) }.to raise_error(ArgumentError)
    end
  end
  ```
  `spec/requests/lab/labels_and_collection_spec.rb`:

  ```ruby
  require "rails_helper"
  require "pdf/reader"

  # Contrato §4: comprovante, etiquetas (reimpressão com trilha), coleta por id
  # ou por código de barras; no 21a o exame vai a `collected`.
  RSpec.describe "Laboratório — etiqueta, comprovante e coleta", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider, delivery_days: 3) }
    let(:staff) { lab_staff! }
    let(:req) { open_requisition!(provider: provider, exams: [ exam ], by: staff) }

    it "comprovante com previsão e preparo; etiquetas e reimpressão com trilha" do
      sign_in_as(staff)
      get "/lab/requisitions/#{req.id}/receipt.pdf"
      expect(response.media_type).to eq("application/pdf")
      text = PDF::Reader.new(StringIO.new(response.body)).pages.map(&:text).join
      expect(text).to include(req.number, provider.name, req.expected_delivery_on.strftime("%d/%m/%Y"), "Glicose")
      get "/lab/requisitions/#{req.id}/labels.pdf", params: { sample_ids: [ req.samples.first.id ] }
      expect(PDF::Reader.new(StringIO.new(response.body)).page_count).to eq(1)
      get "/lab/requisitions/#{req.id}/labels.pdf"
      printed = DomainEvent.where(name: "lab_sample.label_printed").order(:occurred_at, :created_at).map(&:payload)
      expect(printed.map { |p| p["reprint"] }).to eq([ false, true ])
      expect(printed.first.keys).to contain_exactly("lab_sample_id", "lab_requisition_id", "reprint")
      get "/lab/requisitions/#{req.id}/labels.pdf", params: { sample_ids: [ SecureRandom.uuid ] }
      expect(response).to have_http_status(:not_found)
    end

    it "coleta pela etiqueta: quem e quando; 21a coletado; repetir é 409; UBS não coleta amostra do posto do laboratório" do
      sign_in_as(staff)
      json_post "/lab/samples/collect", barcode: req.samples.first.barcode
      expect(json_body).to include("barcode" => req.samples.first.barcode, "status" => "awaiting", "requisition_number" => req.number)
      expect(json_body["collected_at"]).to be_present
      expect(req.items.first.exam_order.reload.status).to eq("collected")
      json_post "/lab/samples/#{req.samples.first.id}/collect"
      expect([ response.status, json_body["error"] ]).to eq([ 409, "invalid_transition" ])
      json_post "/lab/samples/collect", barcode: "0000000000"
      expect(response).to have_http_status(:not_found)
      other = open_requisition!(provider: provider, exams: [ exam ])
      sign_in_as(doctor!(other.origin_unit))
      json_post "/lab/samples/#{other.samples.first.id}/collect"
      expect(json_body["error"]).to eq("missing_role")
    end
  end
  ```

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/services/lab/labels_pdf_spec.rb spec/requests/lab/labels_and_collection_spec.rb`.

- [ ] **Step 4: Código e etiqueta.** `app/services/lab/code128.rb`:

  ```ruby
  # Code 128 da etiqueta (ADR 0037; prova técnica b): só o codificador da barby
  # (10 dígitos = conjunto C, 90 módulos); o desenho é nosso, em vetor.
  require "barby/barcode/code_128"

  module Lab
    module Code128
      module_function

      def encoding(value)
        raise ArgumentError, "código de barras só com dígitos, em número par" unless value.is_a?(String) && value.match?(/\A(\d\d)+\z/)

        Barby::Code128.new(value).encoding
      end
    end
  end
  ```
  `app/services/lab/labels_pdf.rb`:

  ```ruby
  # Etiqueta de tubo (spec §5.1; RDC 978 art. 138; Valores fixados 4): 50 x 25
  # mm, uma por página (impressora térmica), Code 128 em vetor com zona de
  # silêncio, o código legível, nome, nascimento, material/recipiente e número
  # da requisição. Texto encolhe para caber (nunca invade a barra).
  require "prawn"

  module Lab
    module LabelsPdf
      MM = 72.0 / 25.4
      WIDTH = 50 * MM
      HEIGHT = 25 * MM
      MODULE = 72.0 * 3 / 203
      BAR_HEIGHT = 9 * MM
      QUIET = 10
      TOP = HEIGHT - (2 * MM)

      module_function

      def call(samples)
        pdf = Prawn::Document.new(page_size: [ WIDTH, HEIGHT ], margin: 0, info: { Title: "Etiquetas", Producer: "Rota Saúde" })
        samples.each_with_index do |sample, index|
          pdf.start_new_page if index.positive?
          label(pdf, sample)
        end
        pdf.render
      end

      def label(pdf, sample)
        bars(pdf, Code128.encoding(sample.barcode))
        req = sample.lab_requisition
        patient = req.patient
        birth = patient.birth_date.present? ? Date.iso8601(patient.birth_date).strftime("%d/%m/%Y") : "não informado"
        y = TOP - BAR_HEIGHT - 1
        pdf.text_box sample.barcode, at: [ 0, y ], width: WIDTH, height: 8, size: 7, align: :center
        [ patient.display_name.to_s, "Nasc. #{birth}  #{sample.material} / #{sample.container}", "Req. #{req.number}" ]
          .each_with_index do |line, i|
          pdf.text_box Consultations::Print.safe(line), at: [ 2 * MM, y - 8 - (i * 7.5) ], width: WIDTH - (4 * MM), height: 7.5,
                                                        size: 6.5, overflow: :shrink_to_fit, min_font_size: 4
        end
      end

      # Uma barra por sequência de módulos escuros (sem frestas entre módulos).
      def bars(pdf, encoding)
        x = (WIDTH - (encoding.length * MODULE)) / 2
        pdf.fill_color "000000"
        encoding.scan(/1+|0+/).each do |run|
          pdf.fill_rectangle([ x, TOP ], run.length * MODULE, BAR_HEIGHT) if run.start_with?("1")
          x += run.length * MODULE
        end
      end
      private_class_method :label, :bars
    end
  end
  ```
  `app/services/lab/receipt_pdf.rb`:

  ```ruby
  # Comprovante de atendimento (RDC 978 art. 137): número, nome, data, data
  # prevista de entrega, exames, preparo e contato do laboratório. A4.
  module Lab
    module ReceiptPdf
      module_function

      def call(req)
        provider = req.exam_provider
        s = ->(text) { Consultations::Print.safe(text) }
        pdf = Consultations::Print.document(nil, title: "Comprovante de atendimento")
        pdf.text s.call(provider.name), style: :bold, size: 12
        pdf.text s.call([ provider.address, provider.phone && "Telefone #{provider.phone}", provider.cnes && "CNES #{provider.cnes}" ].compact.join(" — "))
        pdf.move_down 10
        pdf.text "COMPROVANTE DE ATENDIMENTO", style: :bold, size: 13
        pdf.text s.call("Requisição nº #{req.number} — #{req.created_at.in_time_zone.strftime('%d/%m/%Y %H:%M')}")
        pdf.text s.call("Paciente: #{req.patient.display_name}")
        pdf.text s.call("Resultado previsto para: #{req.expected_delivery_on.strftime('%d/%m/%Y')}"), style: :bold
        pdf.move_down 8
        req.items.select(&:active?).each do |item|
          exam = item.lab_exam_version.lab_exam
          preparation = exam.lab_exam_template_id && LabExamTemplate.find_by(id: exam.lab_exam_template_id)&.preparation
          pdf.text s.call("• #{exam.name} (#{item.lab_exam_version.material})#{preparation ? " — preparo: #{preparation}" : ''}")
        end
        pdf.move_down 10
        pdf.text s.call("Retire o resultado com este comprovante e um documento com foto."), size: 9
        pdf.render
      end
    end
  end
  ```

- [ ] **Step 5: Coleta.** `app/commands/lab/collect_sample.rb`:

  ```ruby
  # Coleta (spec §5.2): leitura da etiqueta registra quem e quando; no 21a os
  # exames da amostra vão a `collected`. Quem coleta: a UBS de origem (coleta
  # na unidade) ou o lab_staff (posto do laboratório).
  module Lab
    module CollectSample
      module_function

      def collector?(user, sample)
        req = sample.lab_requisition
        if req.collection_place == "lab"
          Access.staff?(user) && Access.member?(user, req.exam_provider_id)
        else
          Access.collecting_unit_ids(user).include?(req.origin_unit_id)
        end
      end

      def call(sample:, by:, collected_at: nil, now: Time.current)
        return Result.fail(:missing_role) unless collector?(by, sample)

        at = collected_at.nil? ? now : (Time.iso8601(collected_at.to_s) rescue nil)
        return Result.fail(:invalid_collected_at) unless at && at <= now && at >= sample.lab_requisition.created_at

        ApplicationRecord.transaction do
          sample.lock!
          next Result.fail(:invalid_transition, details: { status: sample.status }) if sample.collected? || sample.status != "awaiting"

          sample.update!(collected_by_user: by, collected_at: at)
          sample.items.select(&:active?).each do |item|
            order = item.exam_order
            next unless order.status == "scheduled"

            moved = Exams::Transition.call(order: order, kind: "collected", actor_user: by, now: at)
            raise ActiveRecord::Rollback if moved.failure?
          end
          DomainEvents.publish("lab_sample.collected", lab_sample_id: sample.id, lab_requisition_id: sample.lab_requisition_id)
          Result.ok(sample: sample)
        end || Result.fail(:invalid_transition, details: { status: sample.reload.status })
      end
    end
  end
  ```
  `app/controllers/lab/samples_controller.rb` (cresce na Task 10):

  ```ruby
  module Lab
    class SamplesController < BaseController
      def collect
        sample = find_sample
        return not_found unless sample

        respond(Lab::CollectSample.call(sample: sample, by: Current.user, collected_at: params[:collected_at]), sample)
      end

      private

      def find_sample
        if params[:id]
          uuid?(params[:id]) && LabSample.find_by(id: params[:id])
        else
          params[:barcode].is_a?(String) && LabSample.find_by(barcode: params[:barcode])
        end
      end

      def respond(result, sample)
        return render_failure(result) if result.failure?

        sample = result.payload[:sample] || sample
        render json: Lab::Json.sample(sample.reload).merge(requisition_number: sample.lab_requisition.number)
      end
    end
  end
  ```
  Em `Lab::RequisitionsController`, `before_action :load_requisition, only: %i[show receipt labels]` e:

  ```ruby
      def receipt
        response.headers["Cache-Control"] = "no-store"
        send_data Lab::ReceiptPdf.call(@requisition), type: "application/pdf", disposition: "inline",
                                                      filename: "comprovante-#{@requisition.number}.pdf"
      end

      # Etiquetas (todas, ou as pedidas); toda impressão deixa trilha, com `reprint`.
      def labels
        samples = @requisition.samples.where.not(status: %w[rejected not_received])
        if params[:sample_ids].present?
          ids = Array(params[:sample_ids])
          samples = @requisition.samples.where(id: ids.select { |id| uuid?(id) })
          return not_found unless samples.size == ids.size
        end
        samples.each do |sample|
          reprint = DomainEvent.where(name: "lab_sample.label_printed").where("payload->>'lab_sample_id' = ?", sample.id).exists?
          DomainEvents.publish("lab_sample.label_printed", lab_sample_id: sample.id, lab_requisition_id: @requisition.id, reprint: reprint)
        end
        response.headers["Cache-Control"] = "no-store"
        send_data Lab::LabelsPdf.call(samples.to_a), type: "application/pdf", disposition: "inline", filename: "etiquetas-#{@requisition.number}.pdf"
      end
  ```
  Rotas no `scope "/lab"` (as `.pdf` com `format: false` e **antes** de `requisitions/:id`):

  ```ruby
      get  "requisitions/:id/receipt.pdf", to: "requisitions#receipt", format: false
      get  "requisitions/:id/labels.pdf",  to: "requisitions#labels", format: false
      post "samples/collect",              to: "samples#collect"
      post "samples/:id/collect",          to: "samples#collect"
  ```

- [ ] **Step 6: Rode:** `rspec spec/services/lab/labels_pdf_spec.rb spec/requests/lab/labels_and_collection_spec.rb` → verde. Gere uma etiqueta da semente de teste e leia como na Task 0 (Step 4) para conferir que o desenho do api é o da prova: `rails runner 'File.binwrite("/rails/.claude/mod21b/tmp/l.pdf", Lab::LabelsPdf.call([LabSample.last]))'` num banco de dev com requisição (depois da Task 23), ou pule e faça na Task 24.

- [ ] **Step 7: Commit** e mova o card **F-21.16** para **Done**:

  ```bash
  git add Gemfile Gemfile.lock app/services/lab/code128.rb app/services/lab/labels_pdf.rb app/services/lab/receipt_pdf.rb app/commands/lab/collect_sample.rb app/controllers/lab/samples_controller.rb app/controllers/lab/requisitions_controller.rb config/routes.rb spec/services/lab/labels_pdf_spec.rb spec/requests/lab/labels_and_collection_spec.rb
  git commit -m "feat: print Code 128 tube labels and receipts and record sample collection

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 4 — Remessa e recebimento (F-21.17)

### Task 10: Remessa da UBS, conferência, recebimento com rejeição e recoleta

**Files:**
- Create: `app/commands/lab/{create_shipment,send_shipment,receive_shipment,receive_sample}.rb`, `app/controllers/lab/shipments_controller.rb`, `app/services/lab/shipment_json.rb`
- Modify: `app/controllers/lab/samples_controller.rb` (`receive`, `pending`), `config/routes.rb`
- Test: `spec/requests/lab/shipments_spec.rb`, `spec/requests/lab/receiving_spec.rb`

**Interfaces:**
- Consumes: `Lab::CollectSample.collector?`, `Lab::Json.sample`, `Lab::RequisitionStatus.refresh!`, `open_requisition!`.
- Produces: `Lab::CreateShipment.call(params, by:)`, `Lab::SendShipment.call(shipment:, by:)`, `Lab::ReceiveShipment.call(shipment:, barcodes:, by:)` → `Result.ok(shipment:)` | `fail(:shipment_empty|:sample_not_collected|:sample_in_other_shipment|:wrong_origin_unit (details: barcode)|:unit_does_not_collect|:invalid_samples|:invalid_temperature|:invalid_transition|:missing_role|:invalid_provider|:not_found)`; `Lab::ReceiveSample.call(barcode:, decision:, reason_id:, restriction_note:, by:, now:)` → `Result.ok(sample:, recollection:)` | `fail(:not_found|:invalid_transition|:reason_required|:restriction_note_required|:invalid_decision|:missing_role)`; `Lab::ShipmentJson.shipment(s)` (dashboard D4); rotas `GET|POST /lab/shipments`, `POST /lab/shipments/:id/send`, `POST /lab/shipments/:id/receive`, `POST /lab/samples/receive` → `{ "sample": <lab_sample>+requisition_number, "recollection": <lab_sample>|null }` (D6), `GET /lab/recollections?origin_unit_id=` → `{ "recollections": [...], "not_received": [...] }`, cada um `<lab_sample>` + `requisition: { id, number, patient_display_name }` (D5).

- [ ] **Step 1: Testes que falham** — `spec/requests/lab/shipments_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4 (F-21.17): a UBS monta a remessa pelas etiquetas, informa a
  # temperatura e envia; o laboratório confere lendo as etiquetas; a lida fica
  # em trânsito até a decisão, a não lida vira not_received e aparece para a UBS.
  RSpec.describe "Laboratório — remessa", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider) }
    let(:unit) { create_unit("UBS Remessa").tap { |u| u.update!(collects_samples: true) } }
    let(:nurse) { doctor!(unit) }
    let(:reqs) { Array.new(2) { open_requisition!(provider: provider, exams: [ exam ], unit: unit, place: "ubs", by: nurse) } }
    let(:barcodes) { reqs.map { |r| r.samples.first.barcode } }

    before { reqs.each { |r| Lab::CollectSample.call(sample: r.samples.first, by: nurse) } }

    it "monta, envia, confere com uma faltando; a UBS vê a não recebida" do
      sign_in_as(nurse)
      json_post "/lab/shipments", origin_unit_id: unit.id, provider_id: provider.id, barcodes: barcodes, temperature_celsius: 4.5
      expect(response).to have_http_status(:created)
      shipment = json_body
      expect(shipment).to include("status" => "open", "temperature_celsius" => 4.5)
      expect(shipment["samples"].map { |s| s["barcode"] }).to match_array(barcodes)
      json_post "/lab/shipments", origin_unit_id: unit.id, provider_id: provider.id, barcodes: [ barcodes.first ], temperature_celsius: 4
      expect(json_body).to eq("error" => "sample_in_other_shipment", "barcode" => barcodes.first)
      json_post "/lab/shipments/#{shipment['id']}/send"
      expect(json_body).to include("status" => "sent", "sent_by" => be_present)
      expect(LabSample.where(barcode: barcodes).pluck(:status).uniq).to eq([ "in_transit" ])
      sign_in_as(lab_staff!)
      get "/lab/shipments", params: { provider_id: provider.id, status: [ "sent" ] }
      expect(json_body["shipments"].map { |s| s["id"] }).to eq([ shipment["id"] ])
      json_post "/lab/shipments/#{shipment['id']}/receive", barcodes: [ barcodes.first ]
      expect(json_body["status"]).to eq("received")
      expect(LabSample.find_by(barcode: barcodes.first).status).to eq("in_transit")
      expect(LabSample.find_by(barcode: barcodes.last).status).to eq("not_received")
      json_post "/lab/shipments/#{shipment['id']}/receive", barcodes: []
      expect(response).to have_http_status(:conflict)
      sign_in_as(nurse)
      get "/lab/recollections", params: { origin_unit_id: unit.id }
      expect(json_body["not_received"].sole).to include("barcode" => barcodes.last, "requisition" => include("number" => reqs.last.number))
      expect(json_body["recollections"]).to eq([])
    end

    it "temperatura fora da faixa, amostra não coletada, outra unidade" do
      sign_in_as(nurse)
      json_post "/lab/shipments", origin_unit_id: unit.id, provider_id: provider.id, barcodes: barcodes, temperature_celsius: 90
      expect(json_body["error"]).to eq("invalid_temperature")
      fresh = open_requisition!(provider: provider, exams: [ exam ], unit: unit, place: "ubs", by: nurse)
      json_post "/lab/shipments", origin_unit_id: unit.id, provider_id: provider.id, barcodes: [ fresh.samples.first.barcode ], temperature_celsius: 4
      expect(json_body).to eq("error" => "sample_not_collected", "barcode" => fresh.samples.first.barcode)
      json_post "/lab/shipments", origin_unit_id: unit.id, provider_id: provider.id, barcodes: [], temperature_celsius: 4
      expect(json_body["error"]).to eq("shipment_empty")
      sign_in_as(doctor!(create_unit("UBS Outra").tap { |u| u.update!(collects_samples: true) }))
      json_post "/lab/shipments", origin_unit_id: unit.id, provider_id: provider.id, barcodes: barcodes, temperature_celsius: 4
      expect(json_body["error"]).to eq("missing_role")
    end
  end
  ```
  `spec/requests/lab/receiving_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4 (F-21.17): aceite, aceite com restrição (nota vai ao laudo) e
  # rejeição com motivo do cadastro, que cria a recoleta na UBS de origem e
  # reabre a coleta no 21a (prazo de coleta recomeça).
  RSpec.describe "Laboratório — recebimento", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider, sigtap_code: "0202010473") }
    let(:staff) { lab_staff! }
    let(:reason) { LabRejectionReason.create!(exam_provider: provider, name: "Amostra hemolisada") }
    let(:req) { open_requisition!(provider: provider, exams: [ exam ], by: staff) }
    let(:sample) { req.samples.first }

    before { Lab::CollectSample.call(sample: sample, by: staff) }

    it "aceita (posto do laboratório): requisição em análise" do
      sign_in_as(staff)
      json_post "/lab/samples/receive", barcode: sample.barcode, decision: "accept"
      expect(json_body["sample"]).to include("status" => "received", "requisition_number" => req.number)
      expect(json_body["recollection"]).to be_nil
      expect(req.reload.status).to eq("in_analysis")
      json_post "/lab/samples/receive", barcode: sample.barcode, decision: "accept"
      expect(json_body["error"]).to eq("invalid_transition")
    end

    it "aceita com restrição exige nota" do
      sign_in_as(staff)
      json_post "/lab/samples/receive", barcode: sample.barcode, decision: "accept_with_restriction"
      expect(json_body["error"]).to eq("restriction_note_required")
      json_post "/lab/samples/receive", barcode: sample.barcode, decision: "accept_with_restriction", restriction_note: "Lipemia discreta"
      expect(json_body["sample"]).to include("status" => "accepted_with_restriction", "restriction_note" => "Lipemia discreta")
    end

    it "rejeita: recoleta nova ligada à anterior, itens mudam de amostra, 21a volta a agendado com prazo de coleta novo" do
      order = req.items.first.exam_order
      expect(order.reload.status).to eq("collected")
      sign_in_as(staff)
      json_post "/lab/samples/receive", barcode: sample.barcode, decision: "reject"
      expect(json_body["error"]).to eq("reason_required")
      travel 1.day do
        json_post "/lab/samples/receive", barcode: sample.barcode, decision: "reject", reason_id: reason.id
      end
      expect(json_body["sample"]).to include("status" => "rejected", "rejection_reason" => "Amostra hemolisada")
      recollection = json_body["recollection"]
      expect(recollection).to include("status" => "awaiting", "recollection_of_id" => sample.id, "barcode" => "#{req.number}02")
      expect(req.items.first.reload.lab_sample_id).to eq(recollection["id"])
      expect(order.reload).to have_attributes(status: "scheduled", stage: "collect")
      expect(order.events.last).to have_attributes(kind: "scheduled", reason: "recollection")
      expect(DomainEvent.where(name: "lab_sample.rejected").sole.payload.keys).to contain_exactly("lab_sample_id", "lab_requisition_id", "recollection_id")
      json_post "/lab/samples/receive", barcode: "9999999999", decision: "accept"
      expect(response).to have_http_status(:not_found)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/shipments_spec.rb spec/requests/lab/receiving_spec.rb`.

- [ ] **Step 3: Comandos da remessa.** `app/commands/lab/create_shipment.rb`:

  ```ruby
  # Remessa da UBS (spec §5.3): amostras coletadas, da unidade, do laboratório,
  # com coleta na unidade e fora de outra remessa; temperatura -30..60 °C.
  module Lab
    module CreateShipment
      MAX = 200

      module_function

      def call(params, by:)
        p = params.to_h.stringify_keys
        unit = p["origin_unit_id"].is_a?(String) && HealthUnit.find_by(id: p["origin_unit_id"])
        return Result.fail(:not_found) unless unit
        return Result.fail(:missing_role) unless Access.collecting_unit_ids(by).include?(unit.id)

        provider = p["provider_id"].is_a?(String) && Access.labs.find_by(id: p["provider_id"])
        return Result.fail(:invalid_provider) unless provider

        temperature = p["temperature_celsius"]
        return Result.fail(:invalid_temperature) unless temperature.is_a?(Numeric) && temperature.between?(-30, 60)

        return Result.fail(:unit_does_not_collect) unless unit.collects_samples

        barcodes = Array(p["barcodes"]).map(&:to_s).uniq
        return Result.fail(:shipment_empty) if barcodes.empty?
        return Result.fail(:invalid_samples) if barcodes.size > MAX

        ApplicationRecord.transaction do
          samples = LabSample.lock.includes(:lab_requisition).where(barcode: barcodes).index_by(&:barcode)
          # Dashboard D4: a recusa diz QUAL amostra (código de barras no corpo).
          failure = barcodes.lazy.map { |code| problem(samples[code], code, unit, provider) }.find(&:itself)
          next failure if failure

          samples = samples.values

          shipment = LabShipment.create!(origin_unit: unit, exam_provider: provider, temperature_celsius: temperature, created_by_user: by)
          LabSample.where(id: samples.map(&:id)).update_all(lab_shipment_id: shipment.id, updated_at: Time.current)
          Result.ok(shipment: shipment)
        end
      end

      def problem(sample, code, unit, provider)
        req = sample&.lab_requisition
        if sample.nil? || req.origin_unit_id != unit.id || req.exam_provider_id != provider.id || req.collection_place != "ubs"
          return Result.fail(:wrong_origin_unit, details: { barcode: code })
        end
        return Result.fail(:sample_in_other_shipment, details: { barcode: code }) if sample.lab_shipment_id
        return Result.fail(:sample_not_collected, details: { barcode: code }) unless sample.status == "awaiting" && sample.collected?

        nil
      end
      private_class_method :problem
    end
  end
  ```
  `app/commands/lab/send_shipment.rb`:

  ```ruby
  module Lab
    module SendShipment
      module_function

      def call(shipment:, by:, now: Time.current)
        return Result.fail(:missing_role) unless Access.collecting_unit_ids(by).include?(shipment.origin_unit_id)

        ApplicationRecord.transaction do
          shipment.lock!
          next Result.fail(:invalid_transition, details: { status: shipment.status }) unless shipment.status == "open"

          shipment.update!(status: "sent", sent_by_user: by, sent_at: now)
          shipment.samples.update_all(status: "in_transit", updated_at: now)
          DomainEvents.publish("lab_shipment.sent", lab_shipment_id: shipment.id)
          Result.ok(shipment: shipment)
        end
      end
    end
  end
  ```
  `app/commands/lab/receive_shipment.rb`:

  ```ruby
  # Conferência da remessa (contrato §4): as lidas seguem em trânsito até a
  # decisão (aceite/rejeição), as não lidas viram not_received (a UBS vê).
  module Lab
    module ReceiveShipment
      module_function

      def call(shipment:, barcodes:, by:, now: Time.current)
        return Result.fail(:missing_role) unless Access.staff?(by) && Access.member?(by, shipment.exam_provider_id)

        read = Array(barcodes).map(&:to_s).uniq
        ApplicationRecord.transaction do
          shipment.lock!
          next Result.fail(:invalid_transition, details: { status: shipment.status }) unless shipment.status == "sent"

          listed = shipment.samples.pluck(:barcode)
          next Result.fail(:invalid_samples) unless (read - listed).empty?

          shipment.samples.where.not(barcode: read).where(status: "in_transit").update_all(status: "not_received", updated_at: now)
          shipment.update!(status: "received", received_by_user: by, received_at: now)
          DomainEvents.publish("lab_shipment.received", lab_shipment_id: shipment.id)
          Result.ok(shipment: shipment)
        end
      end
    end
  end
  ```

- [ ] **Step 4: Recebimento da amostra.** `app/commands/lab/receive_sample.rb`:

  ```ruby
  # Recebimento (spec §5.4): aceita, aceita com restrição (nota) ou rejeita
  # com motivo ativo do laboratório. A rejeição cria a recoleta (amostra nova
  # `awaiting`, recollection_of_id), passa os itens pendentes para ela e, no
  # 21a, reabre a coleta (`scheduled` com motivo `recollection`, prazo novo).
  module Lab
    module ReceiveSample
      DECISIONS = %w[accept reject accept_with_restriction].freeze
      FROM = %w[in_transit not_received].freeze

      module_function

      def call(barcode:, decision:, by:, reason_id: nil, restriction_note: nil, now: Time.current)
        sample = barcode.is_a?(String) && LabSample.find_by(barcode: barcode)
        return Result.fail(:not_found) unless sample

        req = sample.lab_requisition
        return Result.fail(:missing_role) unless Access.staff?(by) && Access.member?(by, req.exam_provider_id)
        return Result.fail(:invalid_decision) unless DECISIONS.include?(decision)

        note = restriction_note.is_a?(String) ? restriction_note.strip : ""
        return Result.fail(:restriction_note_required) if decision == "accept_with_restriction" && !note.length.between?(1, 300)

        reason = decision == "reject" && reason_id.is_a?(String) && LabRejectionReason.active.find_by(id: reason_id, exam_provider_id: req.exam_provider_id)
        return Result.fail(:reason_required) if decision == "reject" && !reason

        ApplicationRecord.transaction do
          sample.lock!
          receivable = FROM.include?(sample.status) || (sample.status == "awaiting" && sample.collected? && req.collection_place == "lab")
          next Result.fail(:invalid_transition, details: { status: sample.status }) unless receivable

          base = { received_by_user: by, received_at: now }
          recollection = nil
          case decision
          when "accept" then sample.update!(base.merge(status: "received"))
          when "accept_with_restriction" then sample.update!(base.merge(status: "accepted_with_restriction", restriction_note: note))
          else
            sample.update!(base.merge(status: "rejected", lab_rejection_reason: reason))
            recollection = recollect!(sample, req, by, now)
          end
          if recollection
            DomainEvents.publish("lab_sample.rejected", lab_sample_id: sample.id, lab_requisition_id: req.id, recollection_id: recollection.id)
          else
            DomainEvents.publish("lab_sample.received", lab_sample_id: sample.id, lab_requisition_id: req.id)
          end
          RequisitionStatus.refresh!(req)
          Result.ok(sample: sample, recollection: recollection)
        end || Result.fail(:invalid_transition, details: { status: sample.reload.status })
      end

      def recollect!(sample, req, by, now)
        req.lock!
        index = req.samples.pluck(:barcode).map { |b| b[-2, 2].to_i }.max + 1
        fresh = LabSample.create!(lab_requisition: req, barcode: LabSample.barcode_for(req.number, index), material: sample.material,
                                  container: sample.container, recollection_of: sample)
        place = req.collection_place == "lab" ? "#{req.exam_provider.name} — recoleta no laboratório" : "#{req.origin_unit.name} — recoleta na unidade"
        sample.items.where(status: "pending").find_each do |item|
          item.update!(lab_sample: fresh)
          order = item.exam_order
          next unless order.status == "collected"

          moved = Exams::Transition.call(order: order, kind: "scheduled", actor_user: by, reason: "recollection", now: now,
                                         changes: { scheduled_at: now, scheduled_place: place })
          raise ActiveRecord::Rollback if moved.failure?
        end
        fresh
      end
      private_class_method :recollect!
    end
  end
  ```
  `app/services/lab/shipment_json.rb`:

  ```ruby
  module Lab
    module ShipmentJson
      module_function

      def shipment(s)
        { id: s.id, origin_unit: { id: s.origin_unit_id, name: s.origin_unit.name },
          provider: { id: s.exam_provider_id, name: s.exam_provider.name }, status: s.status,
          temperature_celsius: s.temperature_celsius&.to_f, sent_at: s.sent_at&.iso8601,
          sent_by: Screenings::Json.staff_name(s.sent_by_user), received_at: s.received_at&.iso8601,
          received_by: Screenings::Json.staff_name(s.received_by_user),
          samples: s.samples.includes(:lab_requisition).order(:barcode).map do |x|
            { id: x.id, barcode: x.barcode, material: x.material, status: x.status, requisition_number: x.lab_requisition.number }
          end }
      end

      def pending(sample)
        req = sample.lab_requisition
        Json.sample(sample).merge(requisition: { id: req.id, number: req.number, patient_display_name: req.patient.display_name })
      end
    end
  end
  ```
  `app/controllers/lab/shipments_controller.rb`:

  ```ruby
  module Lab
    class ShipmentsController < BaseController
      before_action :load_shipment, only: %i[send_out receive]

      def index
        scope = LabShipment.includes(:origin_unit, :exam_provider, :sent_by_user, :received_by_user).order(created_at: :desc).limit(100)
        ids = Lab::Access.provider_ids(Current.user)
        if ids.any?
          scope = scope.where(exam_provider_id: params[:provider_id].presence || ids).where(exam_provider_id: ids)
        else
          units = Lab::Access.collecting_unit_ids(Current.user)
          return forbid if units.empty?

          scope = scope.where(origin_unit_id: params[:origin_unit_id].presence || units).where(origin_unit_id: units)
        end
        statuses = Array(params[:status])
        return render(json: { error: "invalid_filter" }, status: :unprocessable_entity) unless (statuses - LabShipment::STATUSES).empty?

        scope = scope.where(status: statuses) if statuses.any?
        render json: { shipments: scope.map { |s| Lab::ShipmentJson.shipment(s) } }
      end

      def create = respond(Lab::CreateShipment.call(body_params, by: Current.user), :created)
      def send_out = respond(Lab::SendShipment.call(shipment: @shipment, by: Current.user), :ok)
      def receive = respond(Lab::ReceiveShipment.call(shipment: @shipment, barcodes: params[:barcodes], by: Current.user), :ok)

      private

      def load_shipment
        @shipment = uuid?(params[:id]) && LabShipment.find_by(id: params[:id])
        not_found unless @shipment
      end

      def respond(result, status)
        return render_failure(result) if result.failure?

        render json: Lab::ShipmentJson.shipment(result.payload[:shipment].reload), status: status
      end
    end
  end
  ```
  Em `Lab::SamplesController`:

  ```ruby
      def receive
        result = Lab::ReceiveSample.call(barcode: params[:barcode], decision: params[:decision], reason_id: params[:reason_id],
                                         restriction_note: params[:restriction_note], by: Current.user)
        return render_failure(result) if result.failure?

        sample = result.payload[:sample].reload
        render json: { sample: Lab::Json.sample(sample).merge(requisition_number: sample.lab_requisition.number),
                       recollection: result.payload[:recollection] && Lab::Json.sample(result.payload[:recollection]) }
      end

      # Dashboard D5: recoletas a coletar e amostras não recebidas da UBS.
      def pending
        unit_id = params[:origin_unit_id]
        return render(json: { error: "filter_required" }, status: :unprocessable_entity) unless uuid?(unit_id)
        return forbid("not_unit_staff") unless Lab::Access.collecting_unit_ids(Current.user).include?(unit_id) || Lab::Access.provider_ids(Current.user).any?

        base = LabSample.joins(:lab_requisition).where(lab_requisitions: { origin_unit_id: unit_id }).includes(lab_requisition: :patient)
        render json: { recollections: base.where(status: "awaiting").where.not(recollection_of_id: nil).map { |s| Lab::ShipmentJson.pending(s) },
                       not_received: base.where(status: "not_received").map { |s| Lab::ShipmentJson.pending(s) } }
      end
  ```
  Rotas no `scope "/lab"` (`samples/receive` antes de `samples/:id/collect` — não colidem, mas mantenha as literais primeiro):

  ```ruby
      post "samples/receive",        to: "samples#receive"
      get  "recollections",          to: "samples#pending"
      get  "shipments",              to: "shipments#index"
      post "shipments",              to: "shipments#create"
      post "shipments/:id/send",     to: "shipments#send_out"
      post "shipments/:id/receive",  to: "shipments#receive"
  ```

- [ ] **Step 5: Rode:** `rspec spec/requests/lab/shipments_spec.rb spec/requests/lab/receiving_spec.rb spec/requests/lab` → verde.

- [ ] **Step 6: Commit** e mova o card **F-21.17** para **Done**:

  ```bash
  git add app/commands/lab/create_shipment.rb app/commands/lab/send_shipment.rb app/commands/lab/receive_shipment.rb app/commands/lab/receive_sample.rb app/controllers/lab/shipments_controller.rb app/controllers/lab/samples_controller.rb app/services/lab/shipment_json.rb config/routes.rb spec/requests/lab/shipments_spec.rb spec/requests/lab/receiving_spec.rb
  git commit -m "feat: add UBS shipments and sample receiving with rejection and recollection

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 5 — Mapa de trabalho e digitação (F-21.18)

### Task 11: Referência, limites, calculados, repetição; mapa (tela e PDF) e digitação

**Files:**
- Create: `app/services/lab/{reference,values,effective,entry_json,worklist,worklist_pdf}.rb`, `app/commands/lab/{enter_results,choose_result}.rb`, `app/controllers/lab/{worklists_controller,results_controller}.rb`
- Modify: `config/routes.rb`, `spec/support/lab_helpers.rb` (`received_requisition!`, `type!`)
- Test: `spec/services/lab/reference_spec.rb`, `spec/commands/lab/enter_results_spec.rb`, `spec/requests/lab/entry_and_worklist_spec.rb`

**Interfaces:**
- Consumes: modelos (Task 5), `Lab::Formula`, `Lab::Ranges.number` (Tasks 3, 6), `Lab::RequisitionStatus` (Task 8), `Lab::ReceiveSample` (Task 10).
- Produces:
  - `Lab::Reference.age_days(patient, on:) → Integer|nil`; `.range_for(analyte, sex:, age_days:) → Hash|nil`; `.flag(analyte, number, sex:, age_days:) → String`; `.text(analyte, range) → String|nil`.
  - `Lab::Values.normalize(analyte, raw) → Result.ok(value: String, number: BigDecimal|nil)` | `fail(:invalid_value|:out_of_entry_range, details: { analyte_key: })`.
  - `Lab::Effective.rows(item) → { "key" => LabResult }` (a última linha de cada componente); `.complete?(item)`.
  - `Lab::EnterResults.call(item:, values:, source: "typed", by: nil, receipt: nil, now: Time.current) → Result.ok(results: [LabResult])` | `fail(:unknown_analyte|:invalid_value|:out_of_entry_range (details: analyte_key)|:sample_not_received|:invalid_transition|:invalid_values)`; `.recalculate!(item, source:, by:, receipt:, now:)`; `.context(item) → { sex:, age_days: }`.
  - `Lab::ChooseResult.call(result:, by:) → Result.ok(result:)` | `fail(:missing_role|:invalid_transition)`.
  - `Lab::EntryJson.item(item)`, `.result_row(row)`, `.analyte(analyte, range)`; `Lab::Worklist.groups(provider:, filters:) → [Hash]`; `Lab::WorklistPdf.call(groups, provider:) → String`.
  - Rotas: `GET /lab/entry?item_id=|barcode=` → `{ "items": [<entry_item>] }` (trilha `clinical_record.viewed` `access: lab_report`; dashboard D7); `POST /lab/results { item_id | (barcode, sigtap_code), values }` → `{ "results": [<lab_result_row>] }` (todas as linhas do item); `POST /lab/results/:id/choose` → `<lab_result_row>`; `GET /lab/worklists` → `{ "groups": [...] }` (D8); `GET /lab/worklists.pdf`.
  - Helpers `LabHelpers#received_requisition!(provider:, exams:, sex: "female", **)` (coletada e aceita) e `#type!(item, values_hash, by: nil)`.

- [ ] **Step 1: Testes que falham** — `spec/services/lab/reference_spec.rb` (Review Focus 3):

  ```ruby
  require "rails_helper"

  # Review Focus 3: faixa do sexo e da idade NA DATA DA COLETA, limites
  # inclusivos; sem nascimento só a faixa sem restrição de idade; crítico vence.
  RSpec.describe Lab::Reference do
    let(:analyte) do
      { "key" => "hb", "value_type" => "numeric", "unit" => "g/dL", "critical_low" => 7.0, "critical_high" => 20.0,
        "reference_ranges" => [
          { "sex" => "female", "age_min_days" => 0, "age_max_days" => 4380, "low" => 11.5, "high" => 15.5, "text" => nil },
          { "sex" => "female", "age_min_days" => 4381, "age_max_days" => 43_800, "low" => 12.0, "high" => 15.5, "text" => nil },
          { "sex" => "any", "age_min_days" => 0, "age_max_days" => 43_800, "low" => 12.0, "high" => 17.5, "text" => nil } ] }
    end

    it "idade na borda, faixa any de reserva, crítico igual ao limite" do
      expect(described_class.range_for(analyte, sex: "female", age_days: 4380)["low"]).to eq(11.5)
      expect(described_class.range_for(analyte, sex: "female", age_days: 4381)["low"]).to eq(12.0)
      expect(described_class.range_for(analyte, sex: "male", age_days: 10_000)["high"]).to eq(17.5)
      expect(described_class.range_for(analyte, sex: "female", age_days: nil)["sex"]).to eq("any")
      expect(described_class.flag(analyte, BigDecimal("11.9"), sex: "female", age_days: 5000)).to eq("low")
      expect(described_class.flag(analyte, BigDecimal("12.0"), sex: "female", age_days: 5000)).to eq("normal")
      expect(described_class.flag(analyte, BigDecimal("15.6"), sex: "female", age_days: 5000)).to eq("high")
      expect(described_class.flag(analyte, BigDecimal("7.0"), sex: "female", age_days: 5000)).to eq("normal")
      expect(described_class.flag(analyte, BigDecimal("6.9"), sex: "female", age_days: 5000)).to eq("critical_low")
      expect(described_class.flag(analyte, BigDecimal("20.1"), sex: "male", age_days: 5000)).to eq("critical_high")
      expect(described_class.text(analyte, described_class.range_for(analyte, sex: "female", age_days: 5000))).to eq("12,0 a 15,5 g/dL")
      expect(described_class.text({ "unit" => "mg/dL", "decimals" => 0 }, { "low" => nil, "high" => 190.0, "text" => nil })).to eq("até 190 mg/dL")
      expect(described_class.flag({ "value_type" => "text" }, nil, sex: "male", age_days: 1)).to eq("normal")
    end

    it "sem nascimento, faixa que exige idade não serve" do
      only_adult = analyte.merge("reference_ranges" => [ analyte["reference_ranges"][1] ])
      expect(described_class.range_for(only_adult, sex: "female", age_days: nil)).to be_nil
      expect(described_class.flag(only_adult, BigDecimal("1"), sex: "female", age_days: nil)).to eq("normal")
    end
  end
  ```
  (A borda "crítico igual ao limite": `critical_low` 7,0 com valor 7,0 é `normal` — crítico é **estritamente** abaixo/acima; o teste fixa isso. Se o RT pedir o contrário, a mudança é uma linha em `flag`.)

  `spec/commands/lab/enter_results_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.6: limite de digitação bloqueia, calculado preenchido pela fórmula,
  # repetição não sobrescreve, item digitado quando tudo tem valor.
  RSpec.describe Lab::EnterResults do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:lipids) do
      lab_exam!(provider: provider, sigtap_code: "0202010295", name: "Perfil lipídico", analytes: LabHelpers::LIPIDS,
                material: "Soro", container: "Tubo amarelo (gel)", instrument_test_code: "LIP")
    end
    let(:glucose) { lab_exam!(provider: provider) }
    let(:staff) { lab_staff! }

    it "calcula LDL quando os três chegam; repetição acrescenta; item digitado" do
      item = received_requisition!(provider: provider, exams: [ lipids ]).items.first
      described_class.call(item: item, values: [ { "analyte_key" => "ct", "value" => "200" }, { "analyte_key" => "hdl", "value" => "50" } ], by: staff)
      expect(item.reload.status).to eq("pending")
      described_class.call(item: item, values: [ { "analyte_key" => "tg", "value" => "150" } ], by: staff)
      expect(Lab::Effective.rows(item.reload).transform_values(&:value)).to eq("ct" => "200", "hdl" => "50", "tg" => "150", "ldl" => "120")
      expect(item.status).to eq("typed")
      described_class.call(item: item, values: [ { "analyte_key" => "hdl", "value" => "60" } ], by: staff)
      expect(Lab::Effective.rows(item.reload)["ldl"].value).to eq("110")
      expect(item.results.where(analyte_key: "hdl").count).to eq(2)
      expect(described_class.call(item: item, values: [ { "analyte_key" => "ldl", "value" => "1" } ], by: staff).reason).to eq(:unknown_analyte)
    end

    it "limite de digitação, valor ruim, amostra não recebida" do
      item = received_requisition!(provider: provider, exams: [ glucose ]).items.first
      result = described_class.call(item: item, values: [ { "analyte_key" => "glicose", "value" => "1500,1" } ], by: staff)
      expect([ result.reason, result.details[:analyte_key] ]).to eq([ :out_of_entry_range, "glicose" ])
      expect(described_class.call(item: item, values: [ { "analyte_key" => "glicose", "value" => "1500" } ], by: staff)).to be_ok
      expect(described_class.call(item: item, values: [ { "analyte_key" => "glicose", "value" => "abc" } ], by: staff).reason).to eq(:invalid_value)
      expect(Lab::Effective.rows(item.reload)["glicose"].flag).to eq("critical_high")
      fresh = open_requisition!(provider: provider, exams: [ glucose ]).items.first
      expect(described_class.call(item: fresh, values: [ { "analyte_key" => "glicose", "value" => "90" } ], by: staff).reason).to eq(:sample_not_received)
    end
  end
  ```
  `spec/requests/lab/entry_and_worklist_spec.rb`:

  ```ruby
  require "rails_helper"
  require "pdf/reader"

  # Contrato §5 com dashboard D7/D8: mapa por setor e bancada (sem valor, sem
  # trilha); digitação lê a faixa já escolhida e grava trilha; escolha da repetição.
  RSpec.describe "Laboratório — mapa e digitação", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:glucose) { lab_exam!(provider: provider).tap { |e| e.update!(bench: "Bancada 1") } }
    let(:staff) { lab_staff! }
    let(:req) { received_requisition!(provider: provider, exams: [ glucose ]) }
    let(:item) { req.items.first }

    it "mapa por setor e bancada; PDF; filtros pendentes e urgência" do
      req
      sign_in_as(staff)
      expect { get "/lab/worklists", params: { provider_id: provider.id, pending: "true" } }
        .not_to(change { DomainEvent.where(name: "clinical_record.viewed").count })
      group = json_body["groups"].sole
      expect(group).to include("sector" => "biochemistry", "bench" => "Bancada 1")
      expect(group["items"].sole).to include("item_id" => item.id, "requisition_number" => req.number, "label" => "Glicose", "urgent" => false)
      get "/lab/worklists", params: { provider_id: provider.id, urgent: "true" }
      expect(json_body["groups"]).to eq([])
      get "/lab/worklists.pdf", params: { provider_id: provider.id }
      expect(PDF::Reader.new(StringIO.new(response.body)).pages.first.text).to include("Bancada 1", req.number, "Glicose")
    end

    it "digitação pelo item e pela etiqueta; trilha na leitura; escolha só do liberador" do
      sign_in_as(staff)
      expect { get "/lab/entry", params: { item_id: item.id } }.to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(1)
      entry = json_body["items"].sole
      expect(entry["analytes"].sole).to include("key" => "glicose", "reference" => { "low" => 70.0, "high" => 99.0, "text" => nil },
                                                "entry_max" => 1500.0, "critical_high" => 400.0, "formula" => nil)
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "lab_report", "lab_requisition_id" => req.id)
      json_post "/lab/results", item_id: item.id, values: [ { analyte_key: "glicose", value: "450" } ]
      expect(json_body["results"].sole).to include("value" => "450", "flag" => "critical_high", "source" => "typed", "chosen" => false)
      json_post "/lab/results", barcode: req.samples.first.barcode, sigtap_code: "0202010473", values: [ { analyte_key: "glicose", value: "98" } ]
      expect(json_body["results"].size).to eq(2)
      first_id = json_body["results"].first["id"]
      json_post "/lab/results/#{first_id}/choose"
      expect(json_body["error"]).to eq("missing_role")
      sign_in_as(lab_releaser!(provider))
      json_post "/lab/results/#{first_id}/choose"
      expect(json_body).to include("value" => "450", "chosen" => true)
      expect(Lab::Effective.rows(item.reload)["glicose"].value).to eq("450")
      sign_in_as(doctor!(req.origin_unit))
      get "/lab/entry", params: { item_id: item.id }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/lab/reference_spec.rb spec/commands/lab/enter_results_spec.rb spec/requests/lab/entry_and_worklist_spec.rb`.

- [ ] **Step 3: Referência, valores e linha que vale.** `app/services/lab/reference.rb`:

  ```ruby
  # Referência e marca do valor (spec §5.6; Review Focus 3): a faixa do sexo
  # (senão `any`) que cobre a idade em dias na data da coleta (limites
  # inclusivos); sem nascimento, só faixa de 0..43800. Crítico: estritamente
  # abaixo de critical_low / acima de critical_high, e vence a faixa.
  module Lab
    module Reference
      module_function

      def age_days(patient, on:)
        return nil if patient.birth_date.blank?

        (on - Date.iso8601(patient.birth_date)).to_i
      rescue Date::Error
        nil
      end

      def range_for(analyte, sex:, age_days:)
        ranges = Array(analyte["reference_ranges"])
        fits = lambda do |r|
          age_days.nil? ? (r["age_min_days"].to_i.zero? && r["age_max_days"].to_i >= Ranges::MAX_AGE_DAYS) : age_days.between?(r["age_min_days"], r["age_max_days"])
        end
        ranges.find { |r| r["sex"] == sex && fits.call(r) } || ranges.find { |r| r["sex"] == "any" && fits.call(r) }
      end

      def flag(analyte, number, sex:, age_days:)
        return "normal" unless analyte["value_type"] == "numeric" && number

        return "critical_low" if analyte["critical_low"] && number < BigDecimal(analyte["critical_low"].to_s)
        return "critical_high" if analyte["critical_high"] && number > BigDecimal(analyte["critical_high"].to_s)

        range = range_for(analyte, sex: sex, age_days: age_days)
        return "normal" unless range
        return "low" if range["low"] && number < BigDecimal(range["low"].to_s)
        return "high" if range["high"] && number > BigDecimal(range["high"].to_s)

        "normal"
      end

      # "12,0 a 15,5 g/dL", "até 190 mg/dL", "a partir de 40 mg/dL" ou o texto da faixa.
      def text(analyte, range)
        return nil unless range
        return range["text"] if range["text"].present?

        unit = analyte["unit"].present? ? " #{analyte['unit']}" : ""
        bounds = [ range["low"], range["high"] ]
        # Casas do componente; sem elas, as da faixa (12,0 a 15,5).
        decimals = analyte["decimals"] || bounds.compact.map { |n| (n % 1).zero? ? 0 : n.to_s.split(".").last.size }.max.to_i
        low, high = bounds.map { |n| n && format("%.#{decimals}f", n).tr(".", ",") }
        if low && high then "#{low} a #{high}#{unit}"
        elsif high then "até #{high}#{unit}"
        elsif low then "a partir de #{low}#{unit}"
        end
      end

    end
  end
  ```
  `app/services/lab/values.rb`:

  ```ruby
  # Valor digitado ou recebido (spec §5.6): numérico com vírgula ou ponto,
  # arredondado às casas do componente; bloqueio fora de entry_min/entry_max
  # (inclusivos); lista só com as opções; texto até 200 caracteres. O valor
  # recusado nunca volta em erro (só a chave do componente).
  module Lab
    module Values
      NUMBER = /\A-?\d+([.,]\d+)?\z/

      module_function

      def normalize(analyte, raw)
        key = analyte["key"]
        text = raw.is_a?(Numeric) ? raw.to_s : raw.to_s.strip
        case analyte["value_type"]
        when "numeric"
          return fail(:invalid_value, key) unless text.match?(NUMBER)

          raw_number = BigDecimal(text.tr(",", "."))
          min, max = analyte["entry_min"], analyte["entry_max"]
          # O limite vale para o valor como chegou (antes de arredondar às casas).
          return fail(:out_of_entry_range, key) if (min && raw_number < BigDecimal(min.to_s)) || (max && raw_number > BigDecimal(max.to_s))

          decimals = analyte["decimals"].to_i
          number = raw_number.round(decimals, BigDecimal::ROUND_HALF_UP)
          Result.ok(value: decimals.positive? ? format("%.#{decimals}f", number) : number.to_i.to_s, number: number)
        when "choice"
          Array(analyte["choices"]).include?(text) ? Result.ok(value: text, number: nil) : fail(:invalid_value, key)
        else
          text.length.between?(1, 200) ? Result.ok(value: text, number: nil) : fail(:invalid_value, key)
        end
      end

      def fail(code, key) = Result.fail(code, details: { analyte_key: key })
      private_class_method :fail
    end
  end
  ```
  `app/services/lab/effective.rb`:

  ```ruby
  # A linha que vale de cada componente (Desvio 9): a última (created_at, id).
  module Lab
    module Effective
      module_function

      def rows(item)
        item.results.to_a.group_by(&:analyte_key).transform_values { |list| list.max_by { |r| [ r.created_at, r.id ] } }
      end

      def complete?(item)
        present = rows(item).keys
        item.lab_exam_version.analytes.all? { |a| present.include?(a["key"]) }
      end
    end
  end
  ```

- [ ] **Step 4: Digitar e escolher.** `app/commands/lab/enter_results.rb`:

  ```ruby
  # Resultado digitado ou recebido (spec §5.6; contrato §5 e §7): a mesma
  # regra para a tela e para a HL7. Trava o item; amostra aceita; componente
  # da versão do item e não calculado; limites; marca pela faixa na data da
  # coleta; calculados refeitos; item `typed` quando todos têm valor. Tudo ou
  # nada. Nunca sobrescreve (só acrescenta).
  module Lab
    module EnterResults
      # Dashboard D7: item já liberado aceita valor novo (só vale numa retificação).
      OPEN = %w[pending typed released].freeze

      module_function

      def call(item:, values:, source: "typed", by: nil, receipt: nil, now: Time.current)
        entries = Array(values).map { |v| v.respond_to?(:to_unsafe_h) ? v.to_unsafe_h : v }
        return Result.fail(:invalid_values) unless entries.size.between?(1, Analytes::MAX) && entries.all?(Hash)

        result = nil
        ApplicationRecord.transaction(requires_new: true) do
          item.lock!
          next result = Result.fail(:invalid_transition, details: { status: item.status }) unless OPEN.include?(item.status)
          next result = Result.fail(:sample_not_received) unless item.lab_sample.accepted?

          version = item.lab_exam_version
          ctx = context(item)
          rows = []
          entries.each do |entry|
            entry = entry.stringify_keys
            analyte = version.analyte(entry["analyte_key"])
            if analyte.nil? || analyte["formula"].present?
              result = Result.fail(:unknown_analyte, details: { analyte_key: entry["analyte_key"].to_s[0, 40] })
              raise ActiveRecord::Rollback
            end
            normalized = Values.normalize(analyte, entry["value"])
            if normalized.failure?
              result = normalized
              raise ActiveRecord::Rollback
            end
            rows << create_row(item, analyte, normalized.payload, ctx, source, by, receipt, now)
          end
          rows.concat(recalculate!(item, source: source, by: by, receipt: receipt, now: now))
          DomainEvents.publish("lab_result.entered", lab_requisition_item_id: item.id, source: source)
          result = Result.ok(results: rows)
        end
        result
      end

      def context(item)
        req = item.lab_requisition
        on = (item.lab_sample.collected_at || item.lab_sample.received_at || Time.current).in_time_zone.to_date
        { sex: req.biological_sex, age_days: Reference.age_days(req.patient, on: on) }
      end

      # Refaz os calculados com as linhas que valem; marca o item digitado.
      def recalculate!(item, source:, by:, receipt:, now:)
        version = item.lab_exam_version
        ctx = context(item)
        effective = Effective.rows(item.reload)
        numbers = effective.transform_values { |row| row.value.match?(Values::NUMBER) ? BigDecimal(row.value.tr(",", ".")) : nil }.compact
        created = version.calculated_analytes.filter_map do |analyte|
          allowed = version.analytes.select { |a| a["value_type"] == "numeric" && a["formula"].nil? }.map { |a| a["key"] }
          number = Formula.evaluate(Formula.compile(analyte["formula"], allowed: allowed), values: numbers)
          next unless number

          value = number.round(analyte["decimals"].to_i, BigDecimal::ROUND_HALF_UP)
          next if effective[analyte["key"]] && BigDecimal(effective[analyte["key"]].value.tr(",", ".")) == value

          create_row(item, analyte, { value: Values.normalize(analyte.merge("entry_min" => nil, "entry_max" => nil), value.to_s("F")).payload[:value],
                                      number: value }, ctx, source, by, receipt, now)
        end
        item.update!(status: "typed") if item.status == "pending" && Effective.complete?(item.reload)
        RequisitionStatus.refresh!(item.lab_requisition)
        created
      end

      def create_row(item, analyte, normalized, ctx, source, by, receipt, now)
        LabResult.create!(lab_requisition_item: item, analyte_key: analyte["key"], value: normalized[:value], unit: analyte["unit"],
                          flag: Reference.flag(analyte, normalized[:number], **ctx), source: source, lab_hl7_receipt: receipt,
                          entered_by_user: by, entered_at: now, created_at: now)
      end
      private_class_method :create_row
    end
  end
  ```
  `app/commands/lab/choose_result.rb`:

  ```ruby
  # O liberador escolhe a repetição que vale (spec §5.6; Desvio 9): acrescenta
  # cópia `chosen` da linha escolhida e refaz os calculados.
  module Lab
    module ChooseResult
      module_function

      def call(result:, by:, now: Time.current)
        item = result.lab_requisition_item
        return Result.fail(:missing_role) unless Access.releaser_link(by, item.lab_requisition.exam_provider_id)

        ApplicationRecord.transaction do
          item.lock!
          next Result.fail(:invalid_transition, details: { status: item.status }) unless EnterResults::OPEN.include?(item.status)

          copy = LabResult.create!(lab_requisition_item: item, analyte_key: result.analyte_key, value: result.value, unit: result.unit,
                                   flag: result.flag, source: result.source, lab_hl7_receipt_id: result.lab_hl7_receipt_id,
                                   entered_by_user: by, entered_at: now, chosen: true, chosen_from: result, created_at: now)
          EnterResults.recalculate!(item, source: result.source, by: by, receipt: result.lab_hl7_receipt, now: now)
          Result.ok(result: copy)
        end
      end
    end
  end
  ```

- [ ] **Step 5: Formas, mapa e PDF.** `app/services/lab/entry_json.rb`:

  ```ruby
  # Item para digitar e conferir (dashboard D7): componentes com a faixa já
  # escolhida e as linhas (valor só para quem é do laboratório — quem chama confere).
  module Lab
    module EntryJson
      module_function

      def item(item)
        req = item.lab_requisition
        ctx = EnterResults.context(item)
        version = item.lab_exam_version
        notified = LabCriticalNotice.where(lab_result_id: item.results.map(&:id)).pluck(:lab_result_id).to_set
        { item_id: item.id, requisition_id: req.id, requisition_number: req.number, barcode: item.lab_sample.barcode,
          patient: { display_name: req.patient.display_name, birth_date: req.patient.birth_date, biological_sex: req.biological_sex },
          sigtap_code: version.lab_exam.sigtap_code, label: version.lab_exam.name, version: version.version, status: item.status,
          sample_status: item.lab_sample.status, restriction_note: item.lab_sample.restriction_note,
          analytes: version.analytes.map { |a| analyte(a, Reference.range_for(a, **ctx)) },
          results: item.results.includes(:entered_by_user).map { |row| result_row(row, version, notified) } }
      end

      def analyte(a, range)
        { key: a["key"], name: a["name"], unit: a["unit"], value_type: a["value_type"], choices: a["value_type"] == "choice" ? a["choices"] : nil,
          decimals: a["decimals"], reference: range && range.slice("low", "high", "text"), entry_min: a["entry_min"], entry_max: a["entry_max"],
          critical_low: a["critical_low"], critical_high: a["critical_high"], formula: a["formula"] }
      end

      def result_row(row, version = row.lab_requisition_item.lab_exam_version, notified = nil)
        body = { id: row.id, item_id: row.lab_requisition_item_id, analyte_key: row.analyte_key, name: version.analyte(row.analyte_key)&.fetch("name"),
                 value: row.value, unit: row.unit, flag: row.flag, source: row.source, entered_at: row.entered_at.iso8601,
                 entered_by: Screenings::Json.staff_name(row.entered_by_user), chosen: row.chosen }
        body[:notified] = (notified || LabCriticalNotice.where(lab_result_id: row.id).pluck(:lab_result_id).to_set).include?(row.id) if row.critical?
        body
      end
    end
  end
  ```
  `app/services/lab/worklist.rb`:

  ```ruby
  # Mapa de trabalho (spec §5.5; dashboard D8): itens com amostra aceita, por
  # setor e bancada; filtros de urgência, data de recebimento e pendentes.
  # Sem valor e sem trilha.
  module Lab
    module Worklist
      module_function

      def groups(provider:, filters:)
        scope = LabRequisitionItem.joins(:lab_sample, :lab_requisition, lab_exam_version: :lab_exam)
                                  .where(lab_requisitions: { exam_provider_id: provider.id }, lab_samples: { status: LabSample::ACCEPTED })
                                  .where(status: filters[:pending] ? %w[pending] : %w[pending typed])
                                  .includes(:lab_sample, lab_requisition: :patient, lab_exam_version: :lab_exam)
        scope = scope.where(lab_exams: { sector: filters[:sector] }) if filters[:sector]
        scope = scope.where(lab_exams: { bench: filters[:bench] }) if filters[:bench]
        scope = scope.where(lab_requisitions: { urgent: true }) if filters[:urgent]
        scope = scope.where(lab_samples: { received_at: filters[:date].in_time_zone.all_day }) if filters[:date]
        scope.order("lab_requisitions.urgent DESC, lab_samples.received_at ASC").limit(500).to_a
             .group_by { |i| [ i.lab_exam_version.lab_exam.sector, i.lab_exam_version.lab_exam.bench ] }
             .sort_by { |(sector, bench), _| [ sector, bench.to_s ] }
             .map do |(sector, bench), items|
          { sector: sector, bench: bench, items: items.map do |i|
            req = i.lab_requisition
            { item_id: i.id, requisition_id: req.id, requisition_number: req.number, barcode: i.lab_sample.barcode,
              patient_display_name: req.patient.display_name, sigtap_code: i.lab_exam_version.lab_exam.sigtap_code,
              label: i.lab_exam_version.lab_exam.name, urgent: req.urgent, received_at: i.lab_sample.received_at&.iso8601, status: i.status }
          end }
        end
      end
    end
  end
  ```
  `app/services/lab/worklist_pdf.rb`:

  ```ruby
  # Mapa de trabalho em PDF (spec §5.5): A4 paisagem, um bloco por setor e
  # bancada, uma linha por item com espaço para anotar o valor.
  module Lab
    module WorklistPdf
      SECTORS = { "biochemistry" => "Bioquímica", "hematology" => "Hematologia", "immunology" => "Imunologia",
                  "urinalysis" => "Urinálise", "parasitology" => "Parasitologia", "microbiology" => "Microbiologia" }.freeze
      COLUMNS = [ [ "Req.", 60 ], [ "Código", 70 ], [ "Paciente", 220 ], [ "Exame", 170 ], [ "Recebida", 80 ], [ "Urg.", 30 ], [ "Resultado", 130 ] ].freeze

      module_function

      def call(groups, provider:, now: Time.current)
        s = ->(text) { Consultations::Print.safe(text) }
        pdf = Prawn::Document.new(page_size: "A4", page_layout: :landscape, margin: 30, info: { Title: "Mapa de trabalho", Producer: "Rota Saúde" })
        pdf.font_size(9)
        pdf.text s.call("#{provider.name} — Mapa de trabalho — #{now.in_time_zone.strftime('%d/%m/%Y %H:%M')}"), style: :bold, size: 12
        groups.each do |group|
          pdf.move_down 10
          pdf.text s.call("#{SECTORS.fetch(group[:sector], group[:sector])}#{group[:bench] ? " — #{group[:bench]}" : ''}"), style: :bold
          row(pdf, COLUMNS.map(&:first), bold: true)
          group[:items].each do |item|
            received = item[:received_at] && Time.iso8601(item[:received_at]).in_time_zone.strftime("%d/%m %H:%M")
            row(pdf, [ item[:requisition_number], item[:barcode], s.call(item[:patient_display_name]), s.call(item[:label]), received.to_s,
                       item[:urgent] ? "SIM" : "", "" ])
          end
        end
        pdf.render
      end

      def row(pdf, cells, bold: false)
        pdf.start_new_page if pdf.cursor < 20
        top = pdf.cursor
        x = 0
        COLUMNS.each_with_index do |(_, width), i|
          pdf.text_box cells[i].to_s, at: [ x, top ], width: width - 4, height: 14, overflow: :shrink_to_fit, style: bold ? :bold : :normal
          x += width
        end
        pdf.stroke_horizontal_line 0, x, at: top - 14
        pdf.move_down 16
      end
      private_class_method :row
    end
  end
  ```

- [ ] **Step 6: Controladores e rotas.** `app/controllers/lab/worklists_controller.rb`:

  ```ruby
  module Lab
    class WorklistsController < BaseController
      before_action :provider_param!
      before_action :load_filters

      def show = render(json: { groups: Lab::Worklist.groups(provider: @provider, filters: @filters) })

      def pdf
        response.headers["Cache-Control"] = "no-store"
        send_data Lab::WorklistPdf.call(Lab::Worklist.groups(provider: @provider, filters: @filters), provider: @provider),
                  type: "application/pdf", disposition: "inline", filename: "mapa-de-trabalho.pdf"
      end

      private

      def load_filters
        date = params[:date].present? ? (Date.iso8601(params[:date].to_s) rescue :invalid) : nil
        sector = params[:sector].presence
        return render(json: { error: "invalid_filter" }, status: :unprocessable_entity) if date == :invalid || (sector && !LabExam::SECTORS.include?(sector))

        @filters = { sector: sector, bench: params[:bench].presence, urgent: params[:urgent] == "true", pending: params[:pending] == "true", date: date }
      end
    end
  end
  ```
  `app/controllers/lab/results_controller.rb`:

  ```ruby
  # Digitação e conferência (contrato §5; dashboard D7). Valor só para quem é
  # do laboratório do item; a leitura grava clinical_record.viewed (lab_report).
  module Lab
    class ResultsController < BaseController
      GRANT = ClinicalRecord::Access::Grant.new(kind: :lab_report, opening: nil, reason: nil)

      def entry
        items = if uuid?(params[:item_id]) then LabRequisitionItem.where(id: params[:item_id])
                elsif params[:barcode].is_a?(String) then LabRequisitionItem.joins(:lab_sample).where(lab_samples: { barcode: params[:barcode] })
                else LabRequisitionItem.none
                end
        items = items.where.not(status: "cancelled").includes(:results, :lab_sample, lab_requisition: :patient, lab_exam_version: :lab_exam).to_a
        return not_found if items.empty?

        req = items.first.lab_requisition
        return forbid unless Lab::Access.member?(Current.user, req.exam_provider_id)

        ClinicalRecord::Trail.viewed!(patient: req.patient, user: Current.user, grant: GRANT, extra: { lab_requisition_id: req.id })
        render json: { items: items.map { |i| Lab::EntryJson.item(i) } }
      end

      def create
        item = find_item
        return not_found unless item
        return forbid unless Lab::Access.member?(Current.user, item.lab_requisition.exam_provider_id)

        result = Lab::EnterResults.call(item: item, values: params[:values], by: Current.user)
        return render_failure(result) if result.failure?

        version = item.lab_exam_version
        render json: { results: item.results.reload.map { |row| Lab::EntryJson.result_row(row, version) } }, status: :created
      end

      def choose
        row = uuid?(params[:id]) && LabResult.find_by(id: params[:id])
        return not_found unless row

        result = Lab::ChooseResult.call(result: row, by: Current.user)
        return render_failure(result) if result.failure?

        render json: Lab::EntryJson.result_row(result.payload[:result]), status: :created
      end

      private

      def find_item
        return LabRequisitionItem.find_by(id: params[:item_id]) if uuid?(params[:item_id])
        return nil unless params[:barcode].is_a?(String) && params[:sigtap_code].is_a?(String)

        LabRequisitionItem.joins(:lab_sample, lab_exam_version: :lab_exam).where.not(status: "cancelled")
                          .find_by(lab_samples: { barcode: params[:barcode] }, lab_exams: { sigtap_code: params[:sigtap_code] })
      end
    end
  end
  ```
  Rotas no `scope "/lab"`:

  ```ruby
      get  "worklists.pdf",       to: "worklists#pdf", format: false
      get  "worklists",           to: "worklists#show"
      get  "entry",               to: "results#entry"
      post "results",             to: "results#create"
      post "results/:id/choose",  to: "results#choose"
  ```
  Helpers em `spec/support/lab_helpers.rb`:

  ```ruby
    # Requisição coletada e aceita no posto do laboratório.
    def received_requisition!(provider:, exams:, sex: "female", **opts)
      req = open_requisition!(provider: provider, exams: exams, **opts)
      req.update!(biological_sex: sex)
      staff = lab_staff!
      req.samples.each do |sample|
        Lab::CollectSample.call(sample: sample, by: staff)
        result = Lab::ReceiveSample.call(barcode: sample.barcode, decision: "accept", by: staff)
        raise "recebimento recusado: #{result.reason}" if result.failure?
      end
      req.reload
    end

    def type!(item, values, by: lab_staff!)
      result = Lab::EnterResults.call(item: item, values: values.map { |k, v| { "analyte_key" => k.to_s, "value" => v.to_s } }, by: by)
      raise "digitação recusada: #{result.reason}" if result.failure?

      item.reload
    end
  ```

- [ ] **Step 7: Rode:** `rspec spec/services/lab spec/commands/lab spec/requests/lab` → verde.

- [ ] **Step 8: Commit** e mova o card **F-21.18** para **Done**:

  ```bash
  git add app/services/lab/reference.rb app/services/lab/values.rb app/services/lab/effective.rb app/services/lab/entry_json.rb app/services/lab/worklist.rb app/services/lab/worklist_pdf.rb app/commands/lab/enter_results.rb app/commands/lab/choose_result.rb app/controllers/lab/worklists_controller.rb app/controllers/lab/results_controller.rb config/routes.rb spec/support/lab_helpers.rb spec/services/lab/reference_spec.rb spec/commands/lab/enter_results_spec.rb spec/requests/lab/entry_and_worklist_spec.rb
  git commit -m "feat: add lab worklists and result entry with references, limits and calculations

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 6 — Entrada HL7 e middleware simulado (F-21.19)

### Task 12: HL7 v2.5.1 — ler e escrever mensagens ER7

**Files:**
- Create: `app/services/lab/hl7/{message,builder}.rb`
- Create: `spec/fixtures/lab/*.hl7` (cópia byte a byte de `contracts/lab/examples/*.hl7`, Task 1)
- Test: `spec/services/lab/hl7_spec.rb`

**Interfaces:**
- Produces: `Lab::Hl7::Invalid` (`#code`); `Lab::Hl7::Message.parse(raw) → Message` (`#msh(n, component = nil)`, `#type` (`"OUL^R22"`), `#control_id`, `#version`, `#processing_id`, `#first(name)`, `#field(segment, n, component = nil)`, `#specimens → [Specimen]`); `Specimen = Data.define(:spm, :index, :orders)`, `Order = Data.define(:obr, :index, :results)`, `Obx = Data.define(:segment, :index)`; `Lab::Hl7::Builder.ack(request:, code:, city_slug:, error: nil, now:)`, `.rsp(request:, code:, city_slug:, query_tag:, barcode:, sample: nil, items: [], error: nil, now:)` → String ER7; `Lab::Hl7::Builder::Error = Data.define(:code, :location)`; `Builder::ERRORS` (ERR-3 e texto por código, tabela do perfil).

- [ ] **Step 1: Copie os exemplos do contracts** (sem editor): `mkdir -p spec/fixtures/lab && cp ../../../contracts/lab/examples/*.hl7 spec/fixtures/lab/` (a partir do worktree; confira com `cmp`).

- [ ] **Step 2: Teste que falha** — `spec/services/lab/hl7_spec.rb`:

  ```ruby
  require "rails_helper"

  # Perfil rotasaude.lab.hl7.v1 (Task 1; Divergências D1 e D2): lê os exemplos
  # do contracts e escreve respostas com a mesma estrutura.
  RSpec.describe "Lab::Hl7" do
    def fixture(name) = File.binread(Rails.root.join("spec/fixtures/lab/#{name}"))
    def segments(text) = text.split("\r").map { |s| s.split("|", -1) }

    it "lê OUL^R22: tipo, controle, versão, grupos SPM/OBR/OBX" do
      message = Lab::Hl7::Message.parse(fixture("oul-r22.hl7"))
      expect([ message.type, message.control_id, message.version ]).to eq([ "OUL^R22", "M0001", "2.5.1" ])
      specimen = message.specimens.sole
      expect(message.field(specimen.spm, 2, 1)).to eq("0000012301")
      order = specimen.orders.sole
      expect(message.field(order.obr, 4, 1)).to eq("GLI")
      obx = order.results.sole
      expect([ message.field(obx.segment, 3, 1), message.field(obx.segment, 5, 1), message.field(obx.segment, 6, 1), obx.index ])
        .to eq([ "glicose", "98", "mg/dL", 1 ])
    end

    it "aceita \\n como separador, desfaz escapes e recusa o que não é HL7" do
      message = Lab::Hl7::Message.parse(fixture("oul-r22.hl7").tr("\r", "\n").sub("|98|", "|5\\S\\3|"))
      expect(message.field(message.specimens.sole.orders.sole.results.sole.segment, 5)).to eq("5^3")
      [ "", "PID|1", "MSH|^~\\$|x", "MSH|^~\\&|a\rxx\r", "\xFF\xFE".b ].each do |raw|
        expect { Lab::Hl7::Message.parse(raw) }.to raise_error(Lab::Hl7::Invalid), raw.inspect
      end
    end

    it "escreve ACK e RSP na estrutura dos exemplos (menos hora e id)" do
      request = Lab::Hl7::Message.parse(fixture("oul-r22.hl7"))
      now = Time.zone.parse("2026-10-15 15:00:01")
      ae = Lab::Hl7::Builder.ack(request: request, code: "AE", city_slug: "curitiba", now: now,
                                 error: Lab::Hl7::Builder::Error.new(code: "unknown_analyte", location: "OBX^1^3"))
      expected = segments(fixture("ack-ae.hl7"))
      got = segments(ae)
      expect(got.map { |s| s.first }).to eq(%w[MSH MSA ERR])
      expect(got[0].values_at(0, 1, 2, 3, 4, 5, 8, 10, 11)).to eq(expected[0].values_at(0, 1, 2, 3, 4, 5, 8, 10, 11))
      expect(got[1..]).to eq(expected[1..])
      query = Lab::Hl7::Message.parse(fixture("qbp-q11.hl7"))
      nf = Lab::Hl7::Builder.rsp(request: query, code: "AE", city_slug: "curitiba", query_tag: "T0001", barcode: "0000012301", now: now,
                                 error: Lab::Hl7::Builder::Error.new(code: "unknown_barcode", location: "QPD^1^3"))
      expect(segments(nf)[1..]).to eq(segments(fixture("rsp-k11-unknown-barcode.hl7"))[1..])
      expect(ae).to end_with("\r")
    end

    it "escapa delimitadores no que escreve" do
      expect(Lab::Hl7::Builder.escape("a|b^c&d~e\\f")).to eq("a\\F\\b\\S\\c\\T\\d\\R\\e\\E\\f")
    end
  end
  ```

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/services/lab/hl7_spec.rb` → `NameError`.

- [ ] **Step 4: Implemente.** `app/services/lab/hl7/message.rb`:

  ```ruby
  # Leitura de uma mensagem HL7 v2 ER7 (perfil rotasaude.lab.hl7.v1): só os
  # delimitadores padrão; segmentos separados por \r (aceita \n e \r\n);
  # UTF-8. Nada da mensagem vai a log ou exceção: Invalid leva só o código.
  module Lab
    module Hl7
      class Invalid < StandardError
        attr_reader :code

        def initialize(code = "invalid_message")
          @code = code
          super(code)
        end
      end

      Obx = Data.define(:segment, :index)
      Order = Data.define(:obr, :index, :results)
      Specimen = Data.define(:spm, :index, :orders)

      class Message
        DELIMITERS = "|^~\\&".freeze
        SEGMENT = /\A[A-Z][A-Z0-9]{2}\|/
        UNESCAPE = { "\\F\\" => "|", "\\S\\" => "^", "\\T\\" => "&", "\\R\\" => "~", "\\E\\" => "\\" }.freeze

        attr_reader :segments

        def self.parse(raw)
          text = raw.to_s.dup.force_encoding(Encoding::UTF_8)
          raise Invalid unless text.valid_encoding?

          lines = text.split(/\r\n|\r|\n/).reject(&:empty?)
          raise Invalid unless lines.first&.start_with?("MSH") && lines.first[3, 5] == DELIMITERS
          raise Invalid unless lines.all? { |line| line.match?(SEGMENT) }

          new(lines.map { |line| line.split("|", -1) })
        end

        def self.unescape(text) = text.gsub(/\\[FSTRE]\\/) { |seq| UNESCAPE.fetch(seq) }

        def initialize(segments)
          @segments = segments
        end

        # MSH-n: o campo 1 é o próprio separador, então MSH-n = posição n - 1.
        def msh(n, component = nil) = pick(segments.first[n - 1], component)
        def type = [ msh(9, 1), msh(9, 2) ].join("^")
        def control_id = msh(10, 1).to_s
        def version = msh(12, 1).to_s
        def processing_id = msh(11, 1).presence || "P"
        def first(name) = segments.find { |s| s.first == name }
        def field(segment, n, component = nil) = segment && pick(segment[n], component)

        # Grupos do OUL^R22: SPM → OBR → OBX (OBX antes do 1º OBR é da amostra e é ignorado).
        def specimens
          counts = Hash.new(0)
          groups = []
          segments.each do |segment|
            name = segment.first
            counts[name] += 1
            case name
            when "SPM" then groups << Specimen.new(spm: segment, index: counts[name], orders: [])
            when "OBR" then groups.last&.orders&.push(Order.new(obr: segment, index: counts[name], results: []))
            when "OBX" then groups.last&.orders&.last&.results&.push(Obx.new(segment: segment, index: counts[name]))
            end
          end
          groups
        end

        private

        # Primeira repetição; componente 1-based; sem componente, o campo inteiro.
        def pick(value, component)
          first = value.to_s.split("~", -1).first.to_s
          raw = component ? first.split("^", -1)[component - 1].to_s : first
          self.class.unescape(raw)
        end
      end
    end
  end
  ```
  `app/services/lab/hl7/builder.rb`:

  ```ruby
  # Respostas ER7 (perfil rotasaude.lab.hl7.v1; Divergência D1): ACK^R22^ACK e
  # RSP^K11^RSP_K11; ERR-3 = tabela HL7 0357, ERR-5 = nosso código (99RSL).
  # Nunca dado do paciente: o RSP só leva amostra, material e exames.
  module Lab
    module Hl7
      module Builder
        Error = Data.define(:code, :location)
        ERRORS = {
          "unknown_barcode" => [ "204", "Unknown key identifier", "Codigo de barras desconhecido" ],
          "sample_not_received" => [ "207", "Application internal error", "Amostra nao recebida" ],
          "exam_not_in_version" => [ "204", "Unknown key identifier", "Exame fora da versao" ],
          "unknown_analyte" => [ "204", "Unknown key identifier", "Componente desconhecido" ],
          "out_of_entry_range" => [ "207", "Application internal error", "Valor fora do limite" ],
          "unit_mismatch" => [ "207", "Application internal error", "Unidade diferente" ],
          "invalid_value" => [ "102", "Data type error", "Valor invalido" ],
          "invalid_message" => [ "100", "Segment sequence error", "Mensagem invalida" ],
          "unsupported_message_type" => [ "200", "Unsupported message type", "Tipo de mensagem nao suportado" ],
          "unsupported_version" => [ "203", "Unsupported version id", "Versao nao suportada" ],
          "token_revoked" => [ "207", "Application internal error", "Token revogado" ],
          "feature_disabled" => [ "207", "Application internal error", "Entrada desligada" ]
        }.freeze
        QUERY_NAME = "LabWorklist^Lista de trabalho^99RSL".freeze

        module_function

        def escape(text) = text.to_s.gsub("\\", "\\E\\").gsub("|", "\\F\\").gsub("^", "\\S\\").gsub("&", "\\T\\").gsub("~", "\\R\\")

        def ack(request:, code:, city_slug:, now:, error: nil)
          lines = [ msh(request, "ACK^R22^ACK", city_slug, now), [ "MSA", code, escape(request&.control_id) ] ]
          lines << err(error) if error
          render(lines)
        end

        def rsp(request:, code:, city_slug:, query_tag:, barcode:, now:, sample: nil, items: [], error: nil)
          lines = [ msh(request, "RSP^K11^RSP_K11", city_slug, now), [ "MSA", code, escape(request&.control_id) ] ]
          lines << err(error) if error
          status = error ? "AE" : (items.empty? ? "NF" : "OK")
          lines << [ "QAK", escape(query_tag), status, QUERY_NAME ]
          lines << [ "QPD", QUERY_NAME, escape(query_tag), escape(barcode) ]
          if sample && items.any?
            lines << [ "SPM", "1", escape(sample.barcode), "", "#{escape(sample.material)}^^99RSL", *Array.new(12, ""),
                       sample.collected_at&.in_time_zone&.strftime("%Y%m%d%H%M%S").to_s ]
            items.each_with_index do |item, i|
              version = item.lab_exam_version
              lines << [ "ORC", "NW", item.id ]
              lines << [ "OBR", (i + 1).to_s, item.id, "", "#{escape(version.instrument_test_code)}^#{escape(version.lab_exam.name)}^99RSL" ]
            end
          end
          render(lines)
        end

        def msh(request, type, city_slug, now)
          [ "MSH", "^~\\&", "ROTA-SAUDE", escape(city_slug), escape(request&.msh(3)), escape(request&.msh(4)),
            now.in_time_zone.strftime("%Y%m%d%H%M%S%z"), "", type, SecureRandom.hex(10), request&.processing_id || "P", "2.5.1" ]
        end

        def err(error)
          hl7, name, text = ERRORS.fetch(error.code)
          [ "ERR", "", error.location.to_s, "#{hl7}^#{name}^HL70357", "E", "#{error.code}^#{text}^99RSL" ]
        end

        def render(lines) = lines.map { |fields| fields.join("|") }.join("\r") + "\r"
        private_class_method :msh, :err, :render
      end
    end
  end
  ```
  (No `msh`, `request&.msh(3)` lê o campo inteiro — o `escape` reescapa o que veio escapado; o teste compara com o exemplo.)

- [ ] **Step 5: Rode:** `rspec spec/services/lab/hl7_spec.rb` → verde. Se a comparação do `rsp` falhar só no `QAK`/`QPD`, confira contra `contracts/lab/examples/rsp-k11-unknown-barcode.hl7` — o exemplo é a fonte; ajuste o código, nunca o exemplo.

- [ ] **Step 6: Commit:**

  ```bash
  git add app/services/lab/hl7 spec/fixtures/lab spec/services/lab/hl7_spec.rb
  git commit -m "feat: read and write HL7 v2.5.1 lab messages (QBP, RSP, OUL, ACK)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 13: Entrada HL7 — `POST /hl7/v2` e `GET /lab/hl7/pending_barcodes`

**Files:**
- Create: `app/services/lab/hl7/inbound.rb`, `app/controllers/hl7/{base_controller,v2_controller,pending_barcodes_controller}.rb`
- Modify: `config/routes.rb`, `spec/support/lab_helpers.rb` (`hl7_post`, `middleware_token!`)
- Test: `spec/requests/hl7/v2_spec.rb`

**Interfaces:**
- Consumes: `Lab::Hl7::{Message,Builder}` (Task 12), `Lab::EnterResults` (Task 11), `LabMiddlewareToken` (Task 5), `Lab::Gate` (Task 2).
- Produces: `Lab::Hl7::Inbound.call(raw:, token:, city_slug:, fake_allowed:, now: Time.current) → String` (sempre um ACK/RSP); rotas `POST /hl7/v2` e `GET /lab/hl7/pending_barcodes` → `{ "barcodes": [String] }`; helpers `LabHelpers#middleware_token!(provider, simulated: false) → String` e `#hl7_post(body, token:)`.

- [ ] **Step 1: Teste que falha** — `spec/requests/hl7/v2_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §7 com D1/D2/D3 (perfil rotasaude.lab.hl7.v1). Review Focus 1:
  # troca de amostra; tudo ou nada; MSH-10 repetido; token revogado; bruto cifrado.
  RSpec.describe "HL7 v2 — entrada do middleware", type: :request do
    before { lab_city!(hl7: true) }

    let(:provider) { lab_provider! }
    let(:glucose) { lab_exam!(provider: provider) }
    let(:req) { received_requisition!(provider: provider, exams: [ glucose ]) }
    let(:barcode) { req.samples.first.barcode }
    let!(:token) { middleware_token!(provider) }

    def oul(barcode:, control: "M#{SecureRandom.hex(4)}", obr: "GLI", obx: [ "OBX|1|NM|glicose^Glicose^99RSL||98|mg/dL^^UCUM|||||F" ])
      ([ "MSH|^~\\&|MW|LAB|ROTA-SAUDE|test|20261015150000||OUL^R22^OUL_R22|#{control}|P|2.5.1",
         "SPM|1|#{barcode}||Plasma^^99RSL", "OBR|1|||#{obr}^Glicose^99RSL" ] + obx).join("\r") + "\r"
    end

    def msa = response.body.split("\r").find { |s| s.start_with?("MSA") }.split("|")[1]
    def err5 = response.body.split("\r").find { |s| s.start_with?("ERR") }&.split("|")&.at(5)&.split("^")&.first

    it "QBP^Q11 devolve a lista de trabalho; código desconhecido AE" do
      qbp = "MSH|^~\\&|MW|LAB|ROTA-SAUDE|test|20261015150000||QBP^Q11^QBP_Q11|Q1|P|2.5.1\rQPD|LabWorklist^Lista de trabalho^99RSL|T1|#{barcode}\rRCP|I\r"
      hl7_post(qbp, token: token)
      expect(response.media_type).to eq("application/hl7-v2")
      expect(msa).to eq("AA")
      expect(response.body).to include("OBR|1|#{req.items.first.id}||GLI^Glicose^99RSL")
      expect(response.body).not_to include(req.patient.display_name.to_s)
      hl7_post(qbp.sub(barcode, "0000000000"), token: token)
      expect([ msa, err5 ]).to eq([ "AE", "unknown_barcode" ])
    end

    it "OUL^R22 aceita, idempotente por MSH-10, bruto cifrado, nunca liberado" do
      hl7_post(oul(barcode: barcode, control: "M1"), token: token)
      expect(msa).to eq("AA")
      row = LabResult.sole
      expect(row).to have_attributes(source: "hl7", value: "98", entered_by_user_id: nil)
      receipt = LabHl7Receipt.sole
      raw = ApplicationRecord.connection.select_value("SELECT raw FROM lab_hl7_receipts WHERE id = #{ApplicationRecord.connection.quote(receipt.id)}")
      expect(raw).not_to include(barcode)
      hl7_post(oul(barcode: barcode, control: "M1", obx: [ "OBX|1|NM|glicose^Glicose^99RSL||300|mg/dL^^UCUM|||||F" ]), token: token)
      expect(msa).to eq("AA")
      expect(LabResult.count).to eq(1)
      expect(ExamResult.count).to eq(0)
      expect(req.items.first.reload.status).to eq("typed")
    end

    it "troca de amostra e recusas: AE e nada gravado (tudo ou nada)" do
      other_lab = lab_provider!
      foreign = received_requisition!(provider: other_lab, exams: [ lab_exam!(provider: other_lab) ])
      rejected = open_requisition!(provider: provider, exams: [ glucose ])
      staff = lab_staff!
      Lab::CollectSample.call(sample: rejected.samples.first, by: staff)
      Lab::ReceiveSample.call(barcode: rejected.samples.first.barcode, decision: "reject", by: staff,
                              reason_id: LabRejectionReason.create!(exam_provider: provider, name: "Hemólise").id)
      cases = {
        oul(barcode: foreign.samples.first.barcode) => "unknown_barcode",
        oul(barcode: rejected.samples.first.barcode) => "sample_not_received",
        oul(barcode: barcode, obr: "OLD") => "exam_not_in_version",
        oul(barcode: barcode, obx: [ "OBX|1|NM|ldl^LDL^99RSL||1|mg/dL^^UCUM|||||F" ]) => "unknown_analyte",
        oul(barcode: barcode, obx: [ "OBX|1|NM|glicose^Glicose^99RSL||1600|mg/dL^^UCUM|||||F" ]) => "out_of_entry_range",
        oul(barcode: barcode, obx: [ "OBX|1|NM|glicose^Glicose^99RSL||5|mmol/L^^UCUM|||||F" ]) => "unit_mismatch",
        oul(barcode: barcode, obx: [ "OBX|1|NM|glicose^Glicose^99RSL||98|mg/dL^^UCUM|||||F", "OBX|2|NM|glicose^Glicose^99RSL||abc|mg/dL^^UCUM|||||F" ]) => "invalid_value"
      }
      cases.each do |body, code|
        hl7_post(body, token: token)
        expect([ msa, err5 ]).to eq([ "AE", code ]), code
      end
      expect(LabResult.count).to eq(0)
      expect(LabHl7Receipt.count).to eq(0)
      hl7_post(oul(barcode: barcode).sub("2.5.1", "2.3.1"), token: token)
      expect([ msa, err5 ]).to eq([ "AR", "unsupported_version" ])
      hl7_post(oul(barcode: barcode).sub("OUL^R22^OUL_R22", "OML^O33^OML_O33"), token: token)
      expect([ msa, err5 ]).to eq([ "AR", "unsupported_message_type" ])
    end

    it "token: sem ou desconhecido 401; revogado AR; simulado sem fake_middleware AR; tipo e tamanho" do
      hl7_post(oul(barcode: barcode), token: nil)
      expect(response).to have_http_status(:unauthorized)
      hl7_post(oul(barcode: barcode), token: "x" * 40)
      expect(response).to have_http_status(:unauthorized)
      Lab::RotateMiddlewareToken.revoke(provider: provider)
      hl7_post(oul(barcode: barcode), token: token)
      expect([ response.status, msa, err5 ]).to eq([ 200, "AR", "token_revoked" ])
      simulated = middleware_token!(provider, simulated: true)
      hl7_post(oul(barcode: barcode), token: simulated)
      expect([ msa, err5 ]).to eq([ "AR", "feature_disabled" ])
      post "/hl7/v2", params: oul(barcode: barcode), headers: { "CONTENT_TYPE" => "text/plain", "Authorization" => "Bearer #{simulated}" }
      expect(response).to have_http_status(:unsupported_media_type)
      hl7_post("MSH|^~\\&|" + ("x" * 1_048_577), token: simulated)
      expect(response).to have_http_status(:payload_too_large)
    end

    it "lista de pendentes para o middleware (só códigos)" do
      req
      get "/lab/hl7/pending_barcodes", headers: { "Authorization" => "Bearer #{token}" }
      expect(JSON.parse(response.body)).to eq("barcodes" => [ barcode ])
      Platform::Features.set!(city: City.find(Current.city.id), key: "lab_hl7_inbound", enabled: false, maintainer: ledi_maintainer!)
      hl7_post(oul(barcode: barcode), token: token)
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/hl7/v2_spec.rb` → 404.

- [ ] **Step 3: O tradutor.** `app/services/lab/hl7/inbound.rb`:

  ```ruby
  # Entrada HL7 (ADR 0037; contrato §7 com D1–D3; Review Focus 1): sempre
  # responde ACK/RSP. Amostra só do laboratório do token, aceita; exame pelo
  # código do aparelho da versão do item; componente da versão e não
  # calculado; unidade igual quando vem; mesma regra da digitação. Mensagem
  # inteira numa transação: um OBX ruim desfaz tudo. MSH-10 aceito antes = AA
  # sem efeito. Bruto cifrado em lab_hl7_receipts. Nada da mensagem em log.
  module Lab
    module Hl7
      module Inbound
        Reject = Class.new(StandardError) do
          attr_reader :error

          def initialize(code, location)
            @error = Builder::Error.new(code: code, location: location)
            super(code)
          end
        end
        TYPES = %w[QBP^Q11 OUL^R22].freeze
        LOCATION = { out_of_entry_range: 5, invalid_value: 5, unknown_analyte: 3 }.freeze

        module_function

        def call(raw:, token:, city_slug:, fake_allowed:, now: Time.current)
          message = begin
            Message.parse(raw)
          rescue Invalid
            return Builder.ack(request: nil, code: "AR", city_slug: city_slug, now: now, error: Builder::Error.new(code: "invalid_message", location: nil))
          end
          ar = ->(code, location = nil) { Builder.ack(request: message, code: "AR", city_slug: city_slug, now: now, error: Builder::Error.new(code: code, location: location)) }
          return ar.call("unsupported_version", "MSH^1^12") unless message.version == "2.5.1"
          return ar.call("token_revoked") if token.revoked?
          return ar.call("feature_disabled") if token.simulated && !fake_allowed
          return ar.call("unsupported_message_type", "MSH^1^9") unless TYPES.include?(message.type)
          return ar.call("invalid_message", "MSH^1^10") unless message.control_id.length.between?(1, 64)

          message.type == "QBP^Q11" ? worklist(message, token, city_slug, now) : results(message, raw, token, city_slug, now)
        end

        def worklist(message, token, city_slug, now)
          qpd = message.first("QPD")
          tag = message.field(qpd, 2).to_s
          barcode = message.field(qpd, 3).to_s
          reply = ->(code, error: nil, sample: nil, items: []) do
            Builder.rsp(request: message, code: code, city_slug: city_slug, query_tag: tag, barcode: barcode, now: now,
                        error: error, sample: sample, items: items)
          end
          sample = find_sample(token, barcode)
          return reply.call("AE", error: Builder::Error.new(code: "unknown_barcode", location: "QPD^1^3")) unless sample
          return reply.call("AE", error: Builder::Error.new(code: "sample_not_received", location: "QPD^1^3")) unless sample.accepted?

          items = sample.items.where(status: "pending").includes(lab_exam_version: :lab_exam).to_a
                        .select { |i| i.lab_exam_version.instrument_test_code.present? }
          reply.call("AA", sample: sample, items: items)
        end

        def results(message, raw, token, city_slug, now)
          ack = ->(code, error = nil) { Builder.ack(request: message, code: code, city_slug: city_slug, now: now, error: error) }
          return ack.call("AA") if LabHl7Receipt.exists?(lab_middleware_token_id: token.id, control_id: message.control_id)

          specimens = message.specimens
          return ack.call("AR", Builder::Error.new(code: "invalid_message", location: "SPM^1^2")) if specimens.empty?

          ApplicationRecord.transaction do
            receipt = LabHl7Receipt.create!(lab_middleware_token: token, control_id: message.control_id, message_type: message.type,
                                            raw: raw.to_s.dup.force_encoding(Encoding::UTF_8), created_at: now)
            specimens.each { |specimen| specimen!(message, specimen, token, receipt, now) }
          end
          ack.call("AA")
        rescue Reject => e
          ack.call("AE", e.error)
        rescue ActiveRecord::RecordNotUnique
          ack.call("AA") # a mesma mensagem chegou duas vezes ao mesmo tempo: a outra gravou
        end

        def specimen!(message, specimen, token, receipt, now)
          barcode = message.field(specimen.spm, 2, 1).presence || message.field(specimen.spm, 2, 2).to_s
          sample = find_sample(token, barcode)
          raise Reject.new("unknown_barcode", "SPM^#{specimen.index}^2") unless sample
          raise Reject.new("sample_not_received", "SPM^#{specimen.index}^2") unless sample.accepted?

          specimen.orders.each do |order|
            code = message.field(order.obr, 4, 1).to_s
            item = sample.items.where.not(status: "cancelled").includes(:lab_exam_version).to_a
                         .find { |i| i.lab_exam_version.instrument_test_code.present? && i.lab_exam_version.instrument_test_code == code }
            raise Reject.new("exam_not_in_version", "OBR^#{order.index}^4") unless item

            values = order.results.map do |obx|
              key = message.field(obx.segment, 3, 1).to_s
              analyte = item.lab_exam_version.analyte(key)
              raise Reject.new("unknown_analyte", "OBX^#{obx.index}^3") if analyte.nil? || analyte["formula"].present?

              unit = message.field(obx.segment, 6, 1).to_s
              raise Reject.new("unit_mismatch", "OBX^#{obx.index}^6") if unit.present? && analyte["unit"].present? && unit != analyte["unit"]

              { "analyte_key" => key, "value" => message.field(obx.segment, 5, 1).to_s, "_index" => obx.index }
            end
            next if values.empty?

            entered = EnterResults.call(item: item, values: values.map { |v| v.except("_index") }, source: "hl7", receipt: receipt, now: now)
            next if entered.ok?

            index = values.find { |v| v["analyte_key"] == entered.details[:analyte_key] }&.fetch("_index") || 1
            column = LOCATION.fetch(entered.reason, 5)
            code = %i[out_of_entry_range invalid_value unknown_analyte].include?(entered.reason) ? entered.reason.to_s : "sample_not_received"
            raise Reject.new(code, "OBX^#{index}^#{column}")
          end
        end

        def find_sample(token, barcode)
          return nil unless barcode.match?(/\A\d{10}\z/)

          LabSample.joins(:lab_requisition).where(lab_requisitions: { exam_provider_id: token.exam_provider_id }).find_by(barcode: barcode)
        end
        private_class_method :worklist, :results, :specimen!, :find_sample
      end
    end
  end
  ```
  (O `raise Reject` dentro da transação a desfaz — recibo e linhas somem juntos; o resgate fica fora dela.)

- [ ] **Step 4: Controladores e rotas.** `app/controllers/hl7/base_controller.rb`:

  ```ruby
  # Borda HL7 (ADR 0037; contrato §7): host da cidade, sem cookie, token do
  # middleware no Authorization (só o hash existe). Ordem: limite por hash
  # (120/min) → interruptor lab_hl7_inbound → token (ausente ou desconhecido
  # 401; revogado e simulado sem fake_middleware respondem AR na mensagem).
  module Hl7
    class BaseController < ApplicationController
      module RateLimitStore
        def self.increment(...) = Rails.cache.increment(...)
      end

      rate_limit to: 120, within: 1.minute, name: "hl7_inbound", store: RateLimitStore,
                 by: -> { bearer_token ? LabMiddlewareToken.digest(bearer_token) : request.remote_ip },
                 with: -> { render json: { error: "rate_limited" }, status: :too_many_requests }
      before_action :require_hl7_inbound!
      before_action :authenticate_token!

      private

      def bearer_token
        header = request.authorization.to_s
        header.start_with?("Bearer ") ? header.delete_prefix("Bearer ").strip.presence : nil
      end

      def require_hl7_inbound!
        return if Lab::Gate.hl7_usable?(Current.city)

        render json: { error: "feature_disabled", feature: Lab::Gate::HL7_KEY }, status: :forbidden
      end

      def authenticate_token!
        @token = LabMiddlewareToken.by_token(bearer_token)
        return if @token && Lab::Access.labs.exists?(id: @token.exam_provider_id)

        response.headers["WWW-Authenticate"] = "Bearer"
        render json: { error: "invalid_token" }, status: :unauthorized
      end

      def fake_allowed? = Lab::Gate.fake_middleware_usable?(Current.city)
    end
  end
  ```
  `app/controllers/hl7/v2_controller.rb`:

  ```ruby
  # POST /hl7/v2: uma mensagem ER7 no corpo (application/hl7-v2, até 1 MB, lido
  # com teto). O tipo não é registrado: o Rails não o põe em params nem no log.
  module Hl7
    class V2Controller < BaseController
      MEDIA_TYPE = "application/hl7-v2".freeze
      MAX_BODY = 1_048_576

      def create
        return render(json: { error: "unsupported_media_type" }, status: :unsupported_media_type) unless request.media_type == MEDIA_TYPE

        request.body.rewind
        body = request.body.read(MAX_BODY + 1).to_s
        return render(json: { error: "payload_too_large" }, status: :payload_too_large) if body.bytesize > MAX_BODY

        reply = Lab::Hl7::Inbound.call(raw: body, token: @token, city_slug: Current.city.slug, fake_allowed: fake_allowed?)
        render plain: reply, content_type: "#{MEDIA_TYPE}; charset=utf-8"
      end
    end
  end
  ```
  `app/controllers/hl7/pending_barcodes_controller.rb`:

  ```ruby
  # GET /lab/hl7/pending_barcodes (contrato §12): códigos de amostras aceitas
  # com exame pendente do laboratório do token. Sem paciente.
  module Hl7
    class PendingBarcodesController < BaseController
      def index
        return render(json: { error: "token_revoked" }, status: :unauthorized) if @token.revoked?
        if @token.simulated && !fake_allowed?
          return render(json: { error: "feature_disabled", feature: Lab::Gate::FAKE_KEY }, status: :forbidden)
        end

        barcodes = LabSample.joins(:lab_requisition, :items)
                            .where(lab_requisitions: { exam_provider_id: @token.exam_provider_id }, status: LabSample::ACCEPTED,
                                   lab_requisition_items: { status: "pending" })
                            .distinct.order(:barcode).limit(200).pluck(:barcode)
        render json: { barcodes: barcodes }
      end
    end
  end
  ```
  Rotas (fora do `scope "/lab"`, antes dele, junto das rotas FHIR):

  ```ruby
    # Módulo 21b (ADR 0037): entrada HL7 do middleware do laboratório (token, sem cookie).
    post "/hl7/v2",                    to: "hl7/v2#create"
    get  "/lab/hl7/pending_barcodes",  to: "hl7/pending_barcodes#index"
  ```
  Em `spec/support/lab_helpers.rb`:

  ```ruby
    def middleware_token!(provider, simulated: false)
      Lab::RotateMiddlewareToken.call(provider: provider, by: nil, simulated: simulated).payload[:token]
    end

    def hl7_post(body, token:)
      headers = { "CONTENT_TYPE" => "application/hl7-v2" }
      headers["Authorization"] = "Bearer #{token}" if token
      post "/hl7/v2", params: body, headers: headers
    end
  ```
  (`RotateMiddlewareToken` revoga o anterior: no teste do token simulado o normal deixa de valer — por isso ele roda depois do revogado.)

- [ ] **Step 5: Rode:** `rspec spec/requests/hl7/v2_spec.rb spec/services/lab/hl7_spec.rb` → verde. Confira que nenhum valor, código de barras ou corpo apareceu no log: `capture_log { hl7_post(...) }` num exemplo extra que espera `not_to include(barcode)` e `not_to include("glicose")`.

- [ ] **Step 6: Commit:**

  ```bash
  git add app/services/lab/hl7/inbound.rb app/controllers/hl7 config/routes.rb spec/support/lab_helpers.rb spec/requests/hl7/v2_spec.rb
  git commit -m "feat: accept HL7 v2.5.1 worklist queries and lab results from middleware

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 14: Middleware simulado (`fake-middleware`) no compose de dev

**Files:**
- Create: `lib/fake_middleware/{client,messages,runner,run}.rb`, `spec/lib/fake_middleware_spec.rb`
- Modify (fora do git): `docker-compose.yml` da raiz do monorepo

**Interfaces:**
- Consumes: `POST /hl7/v2`, `GET /lab/hl7/pending_barcodes` (Task 13); os códigos de aparelho da semente (Task 23: `HEM GLI A1C COL HDL LDL TRI CRE URE TGO TGP TSH T4L HCG`).
- Produces: `FakeMiddleware::Client#pending_barcodes`, `#post(body) → [status, body]`, `#last_status`; `FakeMiddleware::Messages.qbp(barcode)`, `.oul(barcode, code, values, control:)`, `.codes(rsp) → [String]`, `.msa(reply)`, `VALUES`; `FakeMiddleware::Runner.tick(client:, logger:) → [[barcode, code, msa]]`. Variáveis `FAKE_MIDDLEWARE_API_URL`, `FAKE_MIDDLEWARE_CITY_HOST`, `FAKE_MIDDLEWARE_TOKEN`, `FAKE_MIDDLEWARE_INTERVAL` (contrato §12).

- [ ] **Step 1: Teste que falha** — `spec/lib/fake_middleware_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/fake_middleware/runner").to_s

  # Contrato §12: um ciclo lista pendentes, consulta a lista de trabalho e envia
  # resultados de exemplo com um crítico (glicose 450) e uma repetição (glicose 98).
  RSpec.describe FakeMiddleware::Runner do
    before { lab_city!(fake_middleware: true) }

    let(:provider) { lab_provider! }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }
    let(:token) { middleware_token!(provider, simulated: true) }
    let(:logger) { Logger.new(StringIO.new) }

    # O cliente fala com o próprio app de teste (sem rede).
    let(:client) do
      session = ActionDispatch::Integration::Session.new(Rails.application)
      session.host! test_city_host
      Class.new do
        define_method(:initialize) { |s, t| @s = s; @t = t }
        attr_reader :last_status

        define_method(:pending_barcodes) do
          @s.get "/lab/hl7/pending_barcodes", headers: { "Authorization" => "Bearer #{@t}" }
          @last_status = @s.response.status
          @last_status == 200 ? JSON.parse(@s.response.body)["barcodes"] : []
        end

        define_method(:post) do |body|
          @s.post "/hl7/v2", params: body, headers: { "CONTENT_TYPE" => "application/hl7-v2", "Authorization" => "Bearer #{@t}" }
          @last_status = @s.response.status
          [ @last_status, @s.response.body ]
        end
      end.new(session, token)
    end

    it "envia o resultado, o crítico e a repetição; nada liberado" do
      req
      outcome = described_class.tick(client: client, logger: logger)
      expect(outcome).to eq([ [ req.samples.first.barcode, "GLI", "AA" ], [ req.samples.first.barcode, "GLI", "AA" ] ])
      rows = req.items.first.results.reload
      expect(rows.map(&:value)).to eq(%w[450 98])
      expect(rows.first.flag).to eq("critical_high")
      expect(ExamResult.count).to eq(0)
      expect(described_class.tick(client: client, logger: logger)).to eq([])
    end

    it "com fake_middleware desligado, só registra no log" do
      req
      Platform::Features.set!(city: City.find(Current.city.id), key: "fake_middleware", enabled: false, maintainer: ledi_maintainer!)
      expect(described_class.tick(client: client, logger: logger)).to eq([])
      expect(LabResult.count).to eq(0)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/lib/fake_middleware_spec.rb` → `LoadError`.

- [ ] **Step 3: Implemente** (Ruby puro, sem Rails: o serviço roda com `bundle exec ruby`). `lib/fake_middleware/messages.rb`:

  ```ruby
  require "securerandom"
  require "time"

  # Mensagens de exemplo do middleware simulado (ADR 0037; contrato §12):
  # valores fixos por código de aparelho da semente, com um crítico (GLI 450)
  # e uma repetição (GLI 98). Nunca dado do paciente.
  module FakeMiddleware
    module Messages
      VALUES = {
        "GLI" => [ [ [ "glicose", "450", "mg/dL" ] ], [ [ "glicose", "98", "mg/dL" ] ] ],
        "A1C" => [ [ [ "hba1c", "6.8", "%" ] ] ],
        "COL" => [ [ [ "colesterol_total", "212", "mg/dL" ] ] ],
        "HDL" => [ [ [ "hdl", "44", "mg/dL" ] ] ],
        "LDL" => [ [ [ "ldl", "138", "mg/dL" ] ] ],
        "TRI" => [ [ [ "triglicerideos", "164", "mg/dL" ] ] ],
        "CRE" => [ [ [ "creatinina", "0.92", "mg/dL" ] ] ],
        "URE" => [ [ [ "ureia", "31", "mg/dL" ] ] ],
        "TGO" => [ [ [ "tgo", "22", "U/L" ] ] ],
        "TGP" => [ [ [ "tgp", "25", "U/L" ] ] ],
        "TSH" => [ [ [ "tsh", "2.15", "m[IU]/L" ] ] ],
        "T4L" => [ [ [ "t4_livre", "1.21", "ng/dL" ] ] ],
        "HCG" => [ [ [ "beta_hcg", "1.2", "m[IU]/mL" ] ] ],
        "HEM" => [ [ [ "hemacias", "4.61", "10*6/uL" ], [ "hemoglobina", "13.4", "g/dL" ], [ "hematocrito", "40.2", "%" ],
                     [ "vcm", "87.2", "fL" ], [ "hcm", "29.1", "pg" ], [ "chcm", "33.3", "g/dL" ], [ "rdw", "12.9", "%" ],
                     [ "leucocitos", "7.2", "10*3/uL" ], [ "plaquetas", "250", "10*3/uL" ] ] ]
      }.freeze

      module_function

      def header(type, control) = "MSH|^~\\&|MW-SIMULADO|LAB-MUNICIPAL|ROTA-SAUDE|CIDADE|#{Time.now.utc.strftime('%Y%m%d%H%M%S')}||#{type}|#{control}|P|2.5.1"

      def qbp(barcode)
        [ header("QBP^Q11^QBP_Q11", "Q#{SecureRandom.hex(6)}"), "QPD|LabWorklist^Lista de trabalho^99RSL|T#{SecureRandom.hex(3)}|#{barcode}", "RCP|I" ].join("\r") + "\r"
      end

      def oul(barcode, code, values, control: "M#{SecureRandom.hex(8)}")
        obx = values.each_with_index.map { |(key, value, unit), i| "OBX|#{i + 1}|NM|#{key}^#{key}^99RSL||#{value}|#{unit}^^UCUM|||||F" }
        ([ header("OUL^R22^OUL_R22", control), "SPM|1|#{barcode}", "OBR|1|||#{code}^#{code}^99RSL" ] + obx).join("\r") + "\r"
      end

      # OBR-4.1 de cada exame do RSP^K11.
      def codes(rsp) = rsp.to_s.split("\r").select { |s| s.start_with?("OBR|") }.map { |s| s.split("|")[4].to_s.split("^").first }

      def msa(reply) = reply.to_s.split("\r").find { |s| s.start_with?("MSA|") }&.split("|")&.at(1)
    end
  end
  ```
  `lib/fake_middleware/client.rb`:

  ```ruby
  require "net/http"
  require "uri"
  require "json"

  module FakeMiddleware
    # Fala com o api pelo endereço interno e diz a cidade pelo Host (como o fake-lab).
    class Client
      def initialize(api_url:, city_host:, token:)
        @uri = URI(api_url)
        @city_host = city_host
        @token = token
      end

      attr_reader :last_status

      def pending_barcodes
        response = request(Net::HTTP::Get.new("/lab/hl7/pending_barcodes"))
        response.code == "200" ? JSON.parse(response.body).fetch("barcodes", []) : []
      end

      def post(body)
        req = Net::HTTP::Post.new("/hl7/v2")
        req["Content-Type"] = "application/hl7-v2"
        req.body = body
        response = request(req)
        [ response.code.to_i, response.body.to_s ]
      end

      private

      def request(req)
        req["Host"] = @city_host
        req["Authorization"] = "Bearer #{@token}"
        Net::HTTP.start(@uri.host, @uri.port, use_ssl: @uri.scheme == "https", open_timeout: 5, read_timeout: 30) do |http|
          http.request(req).tap { |response| @last_status = response.code.to_i }
        end
      end
    end
  end
  ```
  `lib/fake_middleware/runner.rb`:

  ```ruby
  require_relative "client"
  require_relative "messages"

  module FakeMiddleware
    # Um ciclo: pendentes → QBP^Q11 → um OUL^R22 por exame conhecido (GLI manda
    # dois: o crítico e a repetição). Log só com código de barras, código e MSA.
    module Runner
      module_function

      def tick(client:, logger:)
        barcodes = client.pending_barcodes
        if client.last_status == 403
          logger.info("[fake-middleware] fake_middleware (ou lab_hl7_inbound) desligado nesta cidade: nada a fazer")
          return []
        end
        return [].tap { logger.warn("[fake-middleware] pendentes: #{client.last_status}") } unless client.last_status == 200

        barcodes.flat_map do |barcode|
          _status, rsp = client.post(Messages.qbp(barcode))
          Messages.codes(rsp).flat_map do |code|
            Array(Messages::VALUES[code]).map do |values|
              _status, reply = client.post(Messages.oul(barcode, code, values))
              msa = Messages.msa(reply)
              logger.info("[fake-middleware] #{barcode} #{code}: #{msa}")
              [ barcode, code, msa ]
            end
          end
        end
      end
    end
  end
  ```
  `lib/fake_middleware/run.rb`:

  ```ruby
  # Middleware simulado do compose de dev/staging (ADR 0037). Só em simulação
  # (a mesma lista de Platform::Features::SIMULATION_ENVS, sem carregar o Rails).
  abort "fake-middleware só roda em ambiente de simulação (development, test, staging)" unless
    %w[development test staging].include?((ENV["RAILS_ENV"] || ENV["RACK_ENV"] || "development").to_s)

  require "logger"
  $stdout.sync = true
  require_relative "runner"

  client = FakeMiddleware::Client.new(api_url: ENV.fetch("FAKE_MIDDLEWARE_API_URL", "http://api:3000"),
                                      city_host: ENV.fetch("FAKE_MIDDLEWARE_CITY_HOST", "curitiba.localhost"),
                                      token: ENV.fetch("FAKE_MIDDLEWARE_TOKEN"))
  logger = Logger.new($stdout)
  interval = Integer(ENV.fetch("FAKE_MIDDLEWARE_INTERVAL", "30"))
  loop do
    begin
      FakeMiddleware::Runner.tick(client: client, logger: logger)
    rescue StandardError => e
      logger.warn("[fake-middleware] ciclo falhou: #{e.class}")
    end
    sleep interval
  end
  ```

- [ ] **Step 4: Compose (fora do git).** No `docker-compose.yml` da raiz, no bloco `x-api-env`, depois de `FAKE_LAB_TOKEN`:

  ```yaml
    # Middleware simulado (serviço fake-middleware, ADR 0037): token do laboratório municipal de Curitiba (a semente lê o mesmo valor).
    FAKE_MIDDLEWARE_TOKEN: ${FAKE_MIDDLEWARE_TOKEN:-dev-fake-middleware-token-0123456789abcdef01}
  ```
  e o serviço, depois do `fake-lab`:

  ```yaml
    fake-middleware:
      container_name: fake-middleware-dev
      # Middleware de aparelhos simulado (ADR 0037, módulo 21b): a cada 30 s lista as amostras recebidas do laboratório
      # municipal de Curitiba, consulta a lista de trabalho (QBP^Q11) e envia resultados de exemplo (OUL^R22) com um
      # crítico e uma repetição. Com fake_middleware desligado, só loga. Sob demanda:
      # docker compose --profile fake-middleware up -d fake-middleware
      profiles: ["fake-middleware"]
      image: rota-saude/api:dev
      working_dir: ${FAKE_MIDDLEWARE_WORKDIR:-/rails}
      command: bundle exec ruby lib/fake_middleware/run.rb
      environment:
        FAKE_MIDDLEWARE_API_URL: ${FAKE_MIDDLEWARE_API_URL:-http://api:3000}
        FAKE_MIDDLEWARE_CITY_HOST: ${FAKE_MIDDLEWARE_CITY_HOST:-curitiba.localhost}
        FAKE_MIDDLEWARE_TOKEN: ${FAKE_MIDDLEWARE_TOKEN:-dev-fake-middleware-token-0123456789abcdef01}
        FAKE_MIDDLEWARE_INTERVAL: ${FAKE_MIDDLEWARE_INTERVAL:-30}
      volumes:
        - ./apps/api:/rails
      depends_on:
        api:
          condition: service_started
      restart: unless-stopped
  ```
  `docker compose config --services | grep fake-middleware` → a linha. Durante a prova (Task 24) o serviço aponta para o worktree: `FAKE_MIDDLEWARE_WORKDIR=/rails/.claude/mod21b FAKE_MIDDLEWARE_API_URL=http://api:3042`.

- [ ] **Step 5: Rode:** `rspec spec/lib/fake_middleware_spec.rb` → verde.

- [ ] **Step 6: Commit** (o compose não entra) e mova o card **F-21.19** para **Done**:

  ```bash
  git add lib/fake_middleware spec/lib/fake_middleware_spec.rb
  git commit -m "feat: add simulated lab middleware for development (fake-middleware)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 7 — Laudo, críticos, liberação e assinatura (F-21.20, F-21.21)

### Task 15: O laudo — JSON canônico `lab-report-v1` e PDF do art. 167

**Files:**
- Create: `config/lab/lab-report-v1.json` (cópia do contracts, Task 1), `app/services/lab/{report_content,report_pdf}.rb`
- Test: `spec/services/lab/report_content_spec.rb`, `spec/services/lab/report_pdf_spec.rb`

**Interfaces:**
- Consumes: `Lab::Effective`, `Lab::Reference`, `Lab::EnterResults.context` (Task 11); `Signatures::Jcs.dump`, `ClinicalDocuments::{Qr,Codes}`, `Consultations::Print.{document,safe}`, `Signatures::PdfFooter` (19b/19c).
- Produces: `Lab::ReportContent::SCHEMA = "rotasaude.lab_report.v1"`, `SCHEMA_PATH`, `SCHEMA_SHA256`; `Lab::ReportContent.build(requisition:, items:, released_by:, lead:, partial:, signature_kind:, issued_at:, replaces: nil, reason: nil) → Built(document:, json:, sha256:)` (levanta `Lab::ReportContent::Invalid` só com ponteiros); `Lab::ReportContent.laboratory_ready?(provider)`; `Lab::ReportPdf.call(report, footer: nil) → String` (PDF; `report` pode ser `LabReport.new` não salvo, com `content`, `short_code`, `verification_token`, `signature_kind`, `released_at` ou `created_at`).

- [ ] **Step 1: Copie e prenda o esquema:** `cp ../../../contracts/lab/lab-report-v1.json config/lab/lab-report-v1.json` e anote `shasum -a 256 config/lab/lab-report-v1.json` (o mesmo do arquivo **commitado** no contracts, Task 1).

- [ ] **Step 2: Testes que falham** — `spec/services/lab/report_content_spec.rb`:

  ```ruby
  require "rails_helper"
  require "json_schemer"

  # Contrato §8: o JSON canônico do laudo, fechado, sem CPF, válido no esquema
  # da tag lab-v1.0.0 (cópia presa por SHA-256).
  RSpec.describe Lab::ReportContent do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, council: "CRBM", lead: true) }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }

    it "monta, valida e é determinístico; sem CPF" do
      item = type!(req.items.first, { glicose: "98" })
      built = described_class.build(requisition: req, items: [ item ], released_by: lead, lead: lead, partial: false,
                                    signature_kind: "advanced", issued_at: Time.utc(2026, 10, 15, 18, 30))
      doc = built.document
      expect(doc).to include("schema" => "rotasaude.lab_report.v1", "requisition_number" => req.number, "partial" => false,
                             "issued_at" => "2026-10-15T18:30:00Z", "support_laboratory" => nil)
      expect(doc["laboratory"]).to include("cnes" => provider.cnes, "phone" => "4133501200")
      expect(doc["technical_lead"]).to include("council" => "CRBM", "council_uf" => "PR")
      component = doc["exams"].sole["components"].sole
      expect(component).to eq("key" => "glicose", "name" => "Glicose", "value" => "98", "unit" => "mg/dL", "flag" => "normal",
                              "reference_text" => "70 a 99 mg/dL", "loinc" => "2345-7")
      expect(built.json).not_to include(req.patient.cpf)
      expect(JSONSchemer.schema(JSON.parse(File.read(described_class::SCHEMA_PATH))).valid?(doc)).to be(true)
      again = described_class.build(requisition: req, items: [ item ], released_by: lead, lead: lead, partial: false,
                                    signature_kind: "advanced", issued_at: Time.utc(2026, 10, 15, 18, 30))
      expect(again.sha256).to eq(built.sha256)
    end

    it "esquema preso ao arquivo da tag" do
      expect(Digest::SHA256.file(described_class::SCHEMA_PATH).hexdigest).to eq(described_class::SCHEMA_SHA256)
    end

    it "laboratório sem CNES, endereço ou telefone não está pronto" do
      expect(described_class.laboratory_ready?(provider)).to be(true)
      provider.update!(phone: nil)
      expect(described_class.laboratory_ready?(provider)).to be(false)
    end
  end
  ```
  `spec/services/lab/report_pdf_spec.rb`:

  ```ruby
  require "rails_helper"
  require "pdf/reader"

  # RDC 978 art. 167 (I–XIV, menos XI que não se aplica): o PDF traz cada item.
  RSpec.describe Lab::ReportPdf do
    before { lab_city! }

    let(:provider) { lab_provider!(name: "Laboratório Municipal Central") }
    let(:lead) { lab_releaser!(provider, council: "CRBM", lead: true) }
    let(:releaser) { lab_releaser!(provider, council: "CRF") }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }

    it "os 14 itens do art. 167 e a conferência por QR" do
      req.samples.first.then { |s| s.update_columns(status: "accepted_with_restriction", restriction_note: "Lipemia discreta") }
      item = type!(req.items.first, { glicose: "120" })
      built = Lab::ReportContent.build(requisition: req, items: [ item ], released_by: releaser, lead: lead, partial: true,
                                       signature_kind: "advanced", issued_at: Time.current)
      report = LabReport.new(lab_requisition: req, exam_provider: provider, version: 1, partial: true, signature_kind: "advanced",
                             released_by_user: releaser, content: built.json, content_sha256: built.sha256,
                             verification_token: ClinicalDocuments::Codes.token, short_code: "ABCDE23456", created_at: Time.current)
      text = PDF::Reader.new(StringIO.new(described_class.call(report))).pages.map(&:text).join("\n")
      expected = [ "Laboratório Municipal Central", "CNES #{provider.cnes}", "Rua do Laboratório, 100", "(41) 3350-1200",
                   "Responsável técnico: #{lead.professional.professional_name}", "CRBM-PR #{lead.professional.registration_number}",
                   "Liberado por: #{releaser.professional.professional_name}", "CRF-PR",
                   req.patient.display_name, "Registro #{req.number}", "Coleta:", "Glicose", "Material: Plasma", "Método: Enzimático",
                   "120 mg/dL", "70 a 99 mg/dL", "ALTO", "Amostra aceita com restrição: Lipemia discreta", "Emitido em",
                   "LAUDO PARCIAL", "assinatura eletrônica avançada", "ABCDE-23456" ]
      expected.each { |fragment| expect(text).to include(fragment), fragment }
      expect(text).not_to include(req.patient.cpf)
    end
  end
  ```

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/services/lab/report_content_spec.rb spec/services/lab/report_pdf_spec.rb`.

- [ ] **Step 4: Implemente.** `app/services/lab/report_content.rb` (preencha `SCHEMA_SHA256` com o valor do Step 1):

  ```ruby
  # JSON canônico do laudo (contrato §8; tag lab-v1.0.0, cópia em
  # config/lab/lab-report-v1.json presa por SHA-256): o que o CAdES do 19b
  # assina e de onde o PDF sai. Congelado na liberação (lab_reports.content).
  # RFC 8785 (Signatures::Jcs). Sem CPF. Erro só com ponteiros, nunca valores.
  require "json_schemer"

  module Lab
    module ReportContent
      SCHEMA = "rotasaude.lab_report.v1".freeze
      SCHEMA_PATH = Rails.root.join("config/lab/lab-report-v1.json")
      SCHEMA_SHA256 = "<sha256 do Step 1>".freeze

      class Invalid < StandardError; end
      Built = Data.define(:document, :json, :sha256) do
        def inspect = "#<Lab::ReportContent::Built #{sha256}>"
        alias_method :to_s, :inspect
      end

      module_function

      def laboratory_ready?(provider)
        provider.cnes.present? && provider.address.present? && provider.phone.present?
      end

      def build(requisition:, items:, released_by:, lead:, partial:, signature_kind:, issued_at:, replaces: nil, reason: nil)
        provider = requisition.exam_provider
        patient = requisition.patient
        document = {
          "schema" => SCHEMA,
          "laboratory" => { "name" => provider.name, "cnes" => provider.cnes, "address" => provider.address, "phone" => provider.phone },
          "technical_lead" => professional(lead), "released_by" => professional(released_by),
          "patient" => { "name" => patient.display_name, "birth_date" => patient.birth_date.presence,
                         "biological_sex" => requisition.biological_sex, "record_number" => requisition.number },
          "requisition_number" => requisition.number,
          "exams" => items.sort_by { |i| i.lab_exam_version.lab_exam.name }.map { |item| exam(item) },
          "partial" => partial, "issued_at" => issued_at.utc.iso8601, "signature_kind" => signature_kind,
          "replaces_report_id" => replaces&.id, "rectification_reason" => reason, "support_laboratory" => nil
        }
        validate!(document)
        json = Signatures::Jcs.dump(document)
        Built.new(document: document, json: json, sha256: Digest::SHA256.hexdigest(json))
      end

      def professional(user)
        p = user.professional
        { "name" => p.professional_name, "council" => p.council, "council_uf" => p.council_state, "council_number" => p.registration_number }
      end

      def exam(item)
        version = item.lab_exam_version
        sample = item.lab_sample
        ctx = EnterResults.context(item)
        effective = Effective.rows(item)
        { "sigtap_code" => version.lab_exam.sigtap_code, "name" => version.lab_exam.name, "material" => version.material,
          "method" => version.method, "version" => version.version,
          "collected_at" => (sample.collected_at || sample.received_at).utc.iso8601, "restriction_note" => sample.restriction_note,
          "components" => version.analytes.map do |analyte|
            row = effective.fetch(analyte["key"])
            { "key" => analyte["key"], "name" => analyte["name"], "value" => row.value, "unit" => analyte["unit"], "flag" => row.flag,
              "reference_text" => Reference.text(analyte, Reference.range_for(analyte, **ctx)), "loinc" => analyte["loinc"] }
          end }
      end

      def validate!(document)
        errors = schema.validate(document).first(5)
        return if errors.empty?

        raise Invalid, "laudo fora do esquema: #{errors.map { |e| "#{e['data_pointer'].presence || '(root)'} #{e['type']}" }.join('; ')}"
      end

      def schema = (@schema ||= JSONSchemer.schema(JSON.parse(File.read(SCHEMA_PATH))))
      private_class_method :professional, :exam, :validate!, :schema
    end
  end
  ```
  `app/services/lab/report_pdf.rb`:

  ```ruby
  # PDF do laudo (RDC 978 art. 167; spec §5.7), gerado do JSON congelado:
  # I–II laboratório, CNES, endereço, telefone; III RT; IV quem libera; V–VI
  # paciente, registro, nascimento; VII coleta; VIII exame, material, método;
  # IX valor e unidade; X referência; XII restrição; XIII emissão; XIV
  # assinatura (avançada: bloco no corpo; ICP: rodapé NGS2 do 19b). QR e código
  # da conferência pública (19c). XI (metodologia própria) não se aplica.
  module Lab
    module ReportPdf
      FLAGS = { "low" => "BAIXO", "high" => "ALTO", "critical_low" => "CRÍTICO BAIXO", "critical_high" => "CRÍTICO ALTO" }.freeze
      QR_SIZE = 84

      module_function

      def call(report, footer: nil)
        c = report.content_data
        pdf = Consultations::Print.document(footer, title: "Laudo laboratorial")
        header(pdf, c, report)
        patient(pdf, c)
        c["exams"].each { |exam| exam(pdf, exam) }
        signers(pdf, c, report, footer)
        verification(pdf, report)
        pdf.render
      end

      def s(text) = Consultations::Print.safe(text)
      def date(iso) = Date.iso8601(iso).strftime("%d/%m/%Y")
      def time(iso) = Time.iso8601(iso).in_time_zone.strftime("%d/%m/%Y %H:%M")
      def phone(value) = value.to_s.sub(/\A(\d{2})(\d{4,5})(\d{4})\z/, '(\1) \2-\3')
      def council(p) = "#{p['council']}-#{p['council_uf']} #{p['council_number']}"

      def header(pdf, c, report)
        lab = c["laboratory"]
        pdf.text s(lab["name"]), style: :bold, size: 12
        pdf.text s("CNES #{lab['cnes']} — #{lab['address']} — Telefone #{phone(lab['phone'])}"), size: 9
        pdf.move_down 8
        title = "LAUDO DE EXAMES LABORATORIAIS"
        title += " — LAUDO PARCIAL" if c["partial"]
        pdf.text s(title), style: :bold, size: 13
        if c["replaces_report_id"]
          pdf.text s("RETIFICAÇÃO — substitui o laudo anterior. Motivo: #{c['rectification_reason']}"), style: :bold
        end
        pdf.text s("Laudo versão #{report.version} — Emitido em #{time(c['issued_at'])}"), size: 9
      end

      def patient(pdf, c)
        p = c["patient"]
        pdf.move_down 8
        pdf.text s("Paciente: #{p['name']}"), style: :bold
        birth = p["birth_date"] ? date(p["birth_date"]) : "não informado"
        sex = p["biological_sex"] == "female" ? "feminino" : "masculino"
        pdf.text s("Registro #{p['record_number']} — Nascimento: #{birth} — Sexo biológico: #{sex}")
      end

      def exam(pdf, exam)
        pdf.move_down 10
        pdf.text s(exam["name"]), style: :bold, size: 11
        pdf.text s("Material: #{exam['material']} — Método: #{exam['method']} — Coleta: #{time(exam['collected_at'])} — Versão do exame #{exam['version']}"), size: 9
        pdf.text s("Amostra aceita com restrição: #{exam['restriction_note']}"), size: 9, style: :italic if exam["restriction_note"]
        exam["components"].each do |comp|
          value = [ comp["value"], comp["unit"] ].compact.join(" ")
          mark = FLAGS[comp["flag"]]
          top = pdf.cursor
          pdf.text_box s(comp["name"]), at: [ 0, top ], width: 180, height: 12
          pdf.text_box s("#{value}#{mark ? "  (#{mark})" : ''}"), at: [ 180, top ], width: 150, height: 12, style: mark ? :bold : :normal
          pdf.text_box s(comp["reference_text"] ? "Referência: #{comp['reference_text']}" : ""), at: [ 330, top ], width: pdf.bounds.width - 330, height: 12, size: 8
          pdf.move_down 13
        end
      end

      def signers(pdf, c, report, footer)
        pdf.move_down 12
        pdf.text s("Responsável técnico: #{c.dig('technical_lead', 'name')} — #{council(c['technical_lead'])}"), size: 9
        pdf.text s("Liberado por: #{c.dig('released_by', 'name')} — #{council(c['released_by'])}"), size: 9
        if c["signature_kind"] == "advanced"
          pdf.text s("Assinado com assinatura eletrônica avançada (Lei 14.063/2020; RDC 978/2025, art. 5º, III) por " \
                     "#{c.dig('released_by', 'name')}, #{council(c['released_by'])}, em #{time(c['issued_at'])}."), size: 9
        elsif footer.nil?
          pdf.text s("Assinatura digital ICP-Brasil pendente."), size: 9
        else
          pdf.text s("Assinado digitalmente (ICP-Brasil) — ver rodapé."), size: 9
        end
      end

      def verification(pdf, report)
        pdf.start_new_page if pdf.cursor < QR_SIZE + 20
        pdf.move_down 12
        top = pdf.cursor
        url = "#{CityPublicUrl.base(Current.city)}/v/#{report.verification_token}"
        ClinicalDocuments::Qr.draw(pdf, url, at: [ 0, top ], size: QR_SIZE)
        pdf.bounding_box([ QR_SIZE + 10, top - 10 ], width: pdf.bounds.width - QR_SIZE - 10) do
          pdf.text s("Confira a autenticidade deste laudo:"), size: 9
          pdf.text s(url), size: 8
          pdf.text s("ou em #{CityPublicUrl.base(Current.city)}/v com o código #{ClinicalDocuments::Codes.display(report.short_code)} e o ano de nascimento do paciente."), size: 9
        end
      end
      private_class_method :s, :date, :time, :phone, :council, :header, :patient, :exam, :signers, :verification
    end
  end
  ```

- [ ] **Step 5: Rode:** `rspec spec/services/lab/report_content_spec.rb spec/services/lab/report_pdf_spec.rb` → verde. Se `pdf-reader` quebrar uma frase em duas linhas, compare sem quebras (`text.gsub(/\s+/, " ")`) — nunca afrouxe o fragmento.

- [ ] **Step 6: Commit:**

  ```bash
  git add config/lab/lab-report-v1.json app/services/lab/report_content.rb app/services/lab/report_pdf.rb spec/services/lab/report_content_spec.rb spec/services/lab/report_pdf_spec.rb
  git commit -m "feat: build the canonical lab report and its PDF with the RDC 978 items

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 16: Valores críticos (F-21.21)

**Files:**
- Create: `app/commands/lab/record_critical_notice.rb`, `app/controllers/lab/criticals_controller.rb`, `app/services/lab/criticals.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/lab/criticals_spec.rb`

**Interfaces:**
- Consumes: `LabResult#critical?`, `Lab::Effective` (Task 11), `Lab::Access`.
- Produces: `Lab::RecordCriticalNotice.call(result_id:, notified_person:, notified_role:, channel:, notified_at:, by:) → Result.ok(notice:)` | `fail(:not_found|:not_critical|:missing_role|:invalid_notice)`; `Lab::Criticals.unnotified(provider_ids:) → [LabResult]` (crítico, linha que vale, item liberado, sem aviso); rotas `POST /lab/criticals`, `GET /lab/criticals/unnotified` → `{ "criticals": [ { result_id, requisition_id, requisition_number, patient_display_name, analyte_name, flag, released_at } ] }` (sem valor; dashboard D10), `GET /lab/criticals/unnotified/count` → `{ "count" }` (sem trilha; D11).

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/criticals_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.7 e contrato §5: o aviso do crítico é registrado (quem, papel,
  # canal, quando); liberado sem aviso entra na fila; contador sem trilha.
  RSpec.describe "Laboratório — valores críticos", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:releaser) { lab_releaser!(provider, lead: true) }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }
    let(:item) { type!(req.items.first, { glicose: "450" }) }
    let(:critical) { item.results.last }

    it "liberado sem aviso aparece na fila e no contador; o aviso tira" do
      item.update!(status: "released")
      sign_in_as(releaser)
      expect { get "/lab/criticals/unnotified/count" }.not_to(change { DomainEvent.where(name: "clinical_record.viewed").count })
      expect(json_body).to eq("count" => 1)
      get "/lab/criticals/unnotified"
      expect(json_body["criticals"].sole).to include("result_id" => critical.id, "requisition_number" => req.number,
                                                     "analyte_name" => "Glicose", "flag" => "critical_high")
      expect(json_body["criticals"].sole.keys).not_to include("value")
      json_post "/lab/criticals", result_id: critical.id, notified_person: "Dra. Helena", notified_role: "Médica solicitante",
                                  channel: "telefone", notified_at: Time.current.iso8601
      expect(response).to have_http_status(:created)
      get "/lab/criticals/unnotified/count"
      expect(json_body).to eq("count" => 0)
      notice = LabCriticalNotice.sole
      expect(notice.notified_person).to eq("Dra. Helena")
      expect(DomainEvent.where(name: "lab_critical.notified").sole.payload.keys).to contain_exactly("lab_critical_notice_id", "lab_result_id")
    end

    it "recusa resultado não crítico, canal fora da lista e quem não é do laboratório" do
      sign_in_as(releaser)
      normal = type!(req.items.first, { glicose: "90" }).results.last
      json_post "/lab/criticals", result_id: normal.id, notified_person: "X", notified_role: "Y", channel: "telefone", notified_at: Time.current.iso8601
      expect(json_body["error"]).to eq("not_critical")
      json_post "/lab/criticals", result_id: critical.id, notified_person: "X", notified_role: "Y", channel: "whatsapp", notified_at: Time.current.iso8601
      expect(json_body["error"]).to eq("invalid_notice")
      sign_in_as(lab_releaser!(lab_provider!))
      json_post "/lab/criticals", result_id: critical.id, notified_person: "X", notified_role: "Y", channel: "telefone", notified_at: Time.current.iso8601
      expect(json_body["error"]).to eq("missing_role")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/criticals_spec.rb`.

- [ ] **Step 3: Implemente.** `app/commands/lab/record_critical_notice.rb`:

  ```ruby
  # Registro da comunicação do valor crítico (spec §5.7; RDC 978 art. 154, III
  # e 165): quem foi avisado, papel, canal, quando; quem registrou. Antes ou
  # depois da liberação. Nome de quem foi avisado cifrado; evento só com ids.
  module Lab
    module RecordCriticalNotice
      module_function

      def call(result_id:, notified_person:, notified_role:, channel:, notified_at:, by:, now: Time.current)
        result = result_id.is_a?(String) && LabResult.find_by(id: result_id)
        return Result.fail(:not_found) unless result
        return Result.fail(:missing_role) unless Access.member?(by, result.lab_requisition_item.lab_requisition.exam_provider_id)
        return Result.fail(:not_critical) unless result.critical?

        person = notified_person.is_a?(String) ? notified_person.strip : ""
        role = notified_role.is_a?(String) ? notified_role.strip : ""
        at = (Time.iso8601(notified_at.to_s) rescue nil)
        valid = person.length.between?(1, 120) && role.length.between?(1, 60) && LabCriticalNotice::CHANNELS.include?(channel) &&
                at && at <= now + 5.minutes
        return Result.fail(:invalid_notice) unless valid

        notice = LabCriticalNotice.create!(lab_result: result, notified_person: person, notified_role: role, channel: channel,
                                           notified_at: at, recorded_by_user: by, created_at: now)
        DomainEvents.publish("lab_critical.notified", lab_critical_notice_id: notice.id, lab_result_id: result.id)
        Result.ok(notice: notice)
      end
    end
  end
  ```
  `app/services/lab/criticals.rb`:

  ```ruby
  # Críticos sem aviso (spec §5.7): linha que vale de item liberado, crítica,
  # sem nenhum aviso registrado — nem para ela nem para a linha de onde foi escolhida.
  module Lab
    module Criticals
      module_function

      def unnotified(provider_ids:)
        items = LabRequisitionItem.joins(:lab_requisition).where(status: "released", lab_requisitions: { exam_provider_id: provider_ids })
                                  .includes(:results, lab_requisition: [ :patient, :reports ], lab_exam_version: :lab_exam)
        notified = ->(row) { LabCriticalNotice.where(lab_result_id: [ row.id, row.chosen_from_id ].compact).exists? }
        items.flat_map { |item| Effective.rows(item).values.select(&:critical?).reject(&notified).map { |row| [ item, row ] } }
      end
    end
  end
  ```
  `app/controllers/lab/criticals_controller.rb`:

  ```ruby
  # Críticos (contrato §5; dashboard D10/D11). A lista mostra paciente e
  # componente (sem valor) e grava trilha; o contador, não.
  module Lab
    class CriticalsController < BaseController
      GRANT = ClinicalRecord::Access::Grant.new(kind: :lab_report, opening: nil, reason: nil)

      before_action { forbid if Lab::Access.provider_ids(Current.user).empty? }

      def create
        result = Lab::RecordCriticalNotice.call(result_id: params[:result_id], notified_person: params[:notified_person],
                                                notified_role: params[:notified_role], channel: params[:channel],
                                                notified_at: params[:notified_at], by: Current.user)
        return render_failure(result) if result.failure?

        render json: { id: result.payload[:notice].id }, status: :created
      end

      def index
        rows = Lab::Criticals.unnotified(provider_ids: Lab::Access.provider_ids(Current.user))
        rows.map { |item, _| item.lab_requisition }.uniq.each do |req|
          ClinicalRecord::Trail.viewed!(patient: req.patient, user: Current.user, grant: GRANT, extra: { lab_requisition_id: req.id })
        end
        render json: { criticals: rows.map do |item, row|
          req = item.lab_requisition
          { result_id: row.id, requisition_id: req.id, requisition_number: req.number, patient_display_name: req.patient.display_name,
            analyte_name: item.lab_exam_version.analyte(row.analyte_key)&.fetch("name"), flag: row.flag,
            released_at: req.reports.select { |r| r.status == "released" }.map(&:released_at).compact.max&.iso8601 }
        end }
      end

      def count = render(json: { count: Lab::Criticals.unnotified(provider_ids: Lab::Access.provider_ids(Current.user)).size })
    end
  end
  ```
  Rotas no `scope "/lab"` (`unnotified/count` antes de `unnotified`):

  ```ruby
      get  "criticals/unnotified/count", to: "criticals#count"
      get  "criticals/unnotified",       to: "criticals#index"
      post "criticals",                  to: "criticals#create"
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/criticals_spec.rb` → verde.

- [ ] **Step 5: Commit** e mova o card **F-21.21** para **Done**:

  ```bash
  git add app/commands/lab/record_critical_notice.rb app/services/lab/criticals.rb app/controllers/lab/criticals_controller.rb config/routes.rb spec/requests/lab/criticals_spec.rb
  git commit -m "feat: record critical value notices and list unnotified criticals

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 17: Liberação (avançada e pedido ICP), finalização e volta ao 21a (F-21.20)

**Files:**
- Create: `app/commands/lab/{release,finalize_report}.rb`, `app/services/lab/release_json.rb`, `app/controllers/lab/{releases_controller,reports_controller}.rb`
- Modify: `app/controllers/exams/files_controller.rb` (trilha do PDF do laudo), `config/routes.rb`
- Test: `spec/requests/lab/release_spec.rb`, `spec/commands/lab/release_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Lab::ReportContent`, `Lab::ReportPdf` (Task 15); `Lab::Access`, `Lab::Json.report_summary`, `Lab::EntryJson.result_row`, `Lab::RequisitionStatus`; `Exams::RegisterResult`, `Exams::Files` (21a); `Signatures::{Gate,OpenRequest}`, `SignerCertificate` (19b); `Mfa::Verify.totp_step_for`, `User#consume_totp_step!`.
- Produces:
  - `Lab::Release.call(requisition:, item_ids:, signature:, totp:, by:, now: Time.current, rectifies: nil, reason: nil, contacts: nil) → Result.ok(report:)` | `fail(:missing_role|:council_missing|:invalid_signature|:icp_required|:laboratory_incomplete|:technical_lead_missing|:signature_unavailable|:no_active_certificate|:mfa_required|:invalid_transition)`.
  - `Lab::FinalizeReport.call(report:, pdf_bytes:, now: Time.current) → Result.ok(report:)` | `fail(<motivo>)`; `Lab::FinalizeReport::Failed`; `Lab::FinalizeReport.values_for(report, item) → [Hash]` (forma `values` do 21a, do JSON congelado).
  - `Lab::ReleaseJson.requisition(req, viewer:)` (dashboard D9).
  - Rotas: `GET /lab/releases?provider_id=` → `{ "requisitions": [...] }` (itens `typed` sem laudo pendente); `GET /lab/releases/:id` (mesma forma, qualquer situação); `POST /lab/requisitions/:id/release { item_ids, signature, totp? }` → 201 `<lab_report_summary>`; `GET /lab/reports/:id/pdf` → `{ "pdf_url" }` (URL curta A4; o download grava `clinical_record.viewed` `access: lab_report`).

- [ ] **Step 1: Testes que falham** — `spec/requests/lab/release_spec.rb`:

  ```ruby
  require "rails_helper"
  require "pdf/reader"

  # Contrato §5 e §9 (F-21.20): liberação parcial e total com assinatura
  # avançada (TOTP consumido, hash do PDF); médico só ICP; crítico libera e
  # vai à fila; o 21a recebe source lab, sem duplicar; não liberado invisível.
  RSpec.describe "Laboratório — liberação", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider!(name: "Laboratório Municipal Central") }
    let!(:lead) { lab_releaser!(provider, council: "CRBM", lead: true) }
    let(:glucose) { lab_exam!(provider: provider) }
    let(:creat) do
      lab_exam!(provider: provider, sigtap_code: "0202010317", name: "Creatinina", instrument_test_code: "CRE",
                analytes: [ LabHelpers::GLUCOSE.first.merge("key" => "creatinina", "name" => "Creatinina", "loinc" => "2160-0",
                                                            "decimals" => 2, "critical_high" => nil, "entry_min" => nil, "entry_max" => nil) ])
    end
    let(:req) { received_requisition!(provider: provider, exams: [ glucose, creat ]) }
    let(:items) { req.items.to_a }

    def release(item_ids, signature: "advanced", user: lead, totp: totp_for(user))
      json_post "/lab/requisitions/#{req.id}/release", item_ids: item_ids, signature: signature, totp: totp
    end

    it "parcial com avançada: laudo liberado, 21a com resultado lab, crítico na fila; depois o resto" do
      type!(items[0], { glicose: "450" })
      sign_in_as(lead)
      release([ items[0].id ], totp: "000000")
      expect([ response.status, json_body["error"] ]).to eq([ 401, "mfa_required" ])
      release([ items[1].id ])
      expect(json_body["error"]).to eq("invalid_transition")
      release([ items[0].id ])
      expect(response).to have_http_status(:created)
      summary = json_body
      expect(summary).to include("version" => 1, "partial" => true, "status" => "released", "signature_kind" => "advanced")
      expect(summary["pdf_url"]).to start_with("/exams/files/")
      report = LabReport.find(summary["id"])
      expect(report.pdf_sha256).to eq(Digest::SHA256.hexdigest(report.pdf_file.bytes))
      expect(items[0].reload.status).to eq("released")
      order = items[0].exam_order.reload
      expect(order.status).to eq("resulted")
      result = order.result
      expect(result).to have_attributes(source: "lab", critical: true, laboratory_name: "Laboratório Municipal Central")
      expect(result.values_list.sole).to include("key" => "glicose", "value" => "450", "loinc" => "2345-7")
      expect(order).to be_escalated
      expect(items[1].exam_order.reload.result).to be_nil
      expect(req.reload.status).to eq("in_analysis")
      get "/lab/criticals/unnotified/count"
      expect(json_body).to eq("count" => 1)
      release([ items[0].id ], totp: "999999")
      expect(json_body["error"]).to eq("mfa_required")
      travel 31.seconds do
        type!(items[1], { creatinina: "0,9" })
        release([ items[1].id ])
        expect(json_body).to include("version" => 2, "partial" => false)
      end
      expect(req.reload.status).to eq("released")
      expect(ExamResult.where(exam_order_id: items.map(&:exam_order_id)).count).to eq(2)
      expect(DomainEvent.where(name: "lab_report.released").map { |e| e.payload.keys.sort }.uniq)
        .to eq([ %w[lab_report_id lab_requisition_id partial signature_kind] ])
    end

    it "médico só com ICP; ICP sem assinatura digital utilizável 422; sem RT 422; laboratório incompleto 422" do
      type!(items[0], { glicose: "98" })
      doctor = lab_releaser!(provider, council: "CRM", deputy: true)
      sign_in_as(doctor)
      release([ items[0].id ], user: doctor)
      expect(json_body["error"]).to eq("icp_required")
      release([ items[0].id ], user: doctor, signature: "icp")
      expect(json_body["error"]).to eq("signature_unavailable")
      sign_in_as(lead)
      provider.update!(address: nil)
      release([ items[0].id ])
      expect(json_body["error"]).to eq("laboratory_incomplete")
      provider.update!(address: "Rua do Laboratório, 100 — Centro")
      bio = lab_releaser!(provider, council: "CRF")
      LabReleaser.find_by(user: lead).update!(ended_at: Time.current)
      sign_in_as(bio)
      release([ items[0].id ], user: bio)
      expect(json_body["error"]).to eq("technical_lead_missing")
      sign_in_as(lab_staff!)
      release([ items[0].id ], user: bio)
      expect(json_body["error"]).to eq("missing_role")
    end

    it "fila e conferência: itens digitados com linhas; trilha; PDF por URL curta com trilha" do
      type!(items[0], { glicose: "98" })
      sign_in_as(lead)
      expect { get "/lab/releases", params: { provider_id: provider.id } }
        .to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(1)
      row = json_body["requisitions"].sole
      expect(row).to include("id" => req.id, "number" => req.number)
      typed = row["items"].find { |i| i["id"] == items[0].id }
      expect(typed).to include("status" => "typed", "restriction_note" => nil)
      expect(typed["results"].sole).to include("value" => "98", "flag" => "normal")
      release([ items[0].id ])
      get "/lab/releases/#{req.id}"
      expect(json_body["reports"].sole["status"]).to eq("released")
      get "/lab/reports/#{json_body['reports'].sole['id']}/pdf"
      url = json_body["pdf_url"]
      expect { get url }.to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(1)
      expect(PDF::Reader.new(StringIO.new(response.body)).pages.first.text).to include("LAUDO DE EXAMES LABORATORIAIS")
    end

    it "não liberado é invisível fora do laboratório" do
      type!(items[0], { glicose: "98" })
      order = items[0].exam_order
      sign_in_as(order.requester_user)
      get "/exams/orders/#{order.id}"
      expect(json_body["result"]).to be_nil
      expect(json_body.to_s).not_to include("98")
    end
  end
  ```
  (Repetir o TOTP do mesmo passo é recusado (`consume_totp_step!`): por isso a segunda liberação roda 31 s depois, com `travel`.)

  `spec/commands/lab/release_concurrency_spec.rb` (Review Focus 2), no molde de `spec/commands/exams/register_result_concurrency_spec.rb` (mesmo `in_city`, `empty_tables`, `after` com `session_replication_role = replica`):

  ```ruby
  require "rails_helper"

  # Review Focus 2: dois liberadores liberando o mesmo item ao mesmo tempo, e
  # a unidade cancelando o exame enquanto o laboratório libera. Um laudo, um
  # resultado no 21a; o outro leva invalid_transition; exame cancelado sem resultado.
  RSpec.describe "Liberação sob disputa" do
    self.use_transactional_tests = false

    let(:release_gate) { Queue.new }
    let(:threads) { [] }

    def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }

    def empty_tables
      connection = ApplicationRecord.connection
      connection.tables.reject { |t| %w[schema_migrations ar_internal_metadata].include?(t) }
                .select { |t| connection.select_value("SELECT NOT EXISTS (SELECT 1 FROM #{connection.quote_table_name(t)})") }
    end

    before do
      @rows = in_city do
        @empty = empty_tables
        provider = lab_provider!
        a = lab_releaser!(provider, lead: true)
        b = lab_releaser!(provider)
        req = received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ])
        item = type!(req.items.first, { glicose: "98" })
        { req_id: req.id, item_id: item.id, a: a.id, b: b.id, order_id: item.exam_order_id, doctor: item.exam_order.requester_user_id }
      end
    end

    after do
      3.times { release_gate << true }
      threads.each { |t| t.join(5) || t.kill }
      in_city do
        ApplicationRecord.transaction do
          connection = ApplicationRecord.connection
          connection.execute("SET LOCAL session_replication_role = replica")
          @empty.to_a.each { |table| connection.execute("DELETE FROM #{connection.quote_table_name(table)}") }
        end
      end
    end

    def release_by(user_id)
      user = User.find(user_id)
      Lab::Release.call(requisition: LabRequisition.find(@rows[:req_id]), item_ids: [ @rows[:item_id] ], signature: "advanced",
                        totp: ROTP::TOTP.new(user.otp_secret).now, by: user)
    end

    def cancel = Exams::Steps.cancel(order: ExamOrder.find(@rows[:order_id]), reason: "Feito em outro serviço", actor_user: User.find(@rows[:doctor]))

    # A primeira segura a transação aberta até o release_gate; a segunda espera a trava.
    def race(first, second)
      holding = Queue.new
      first_out = Queue.new
      threads << Thread.new do
        in_city do
          ApplicationRecord.transaction do
            first_out << first.call
            holding << true
            release_gate.pop(timeout: 10) or raise "timeout"
          end
        end
      end
      holding.pop(timeout: 10) or raise "timeout"
      second_out = Queue.new
      threads << Thread.new { in_city { second_out << second.call } }
      sleep 0.3
      release_gate << true
      [ first_out.pop(timeout: 10), second_out.pop(timeout: 10) ]
    end

    it "dois liberadores: um laudo, um resultado; o outro 409" do
      first, second = race(-> { release_by(@rows[:a]) }, -> { release_by(@rows[:b]) })
      expect(first).to be_ok
      expect(second.reason).to eq(:invalid_transition)
      in_city do
        expect(LabReportItem.where(lab_requisition_item_id: @rows[:item_id]).count).to eq(1)
        expect(ExamResult.where(exam_order_id: @rows[:order_id]).count).to eq(1)
      end
    end

    it "liberação × cancelamento: quem vem depois perde; nunca cancelado com resultado" do
      first, second = race(-> { release_by(@rows[:a]) }, -> { cancel })
      expect(first).to be_ok
      expect(second.reason).to eq(:invalid_transition)
      in_city do
        order = ExamOrder.find(@rows[:order_id])
        expect(order.status).to eq("resulted")
      end
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/release_spec.rb spec/commands/lab/release_concurrency_spec.rb`.

- [ ] **Step 3: Finalizar.** `app/commands/lab/finalize_report.rb`:

  ```ruby
  # O laudo passa a valer (ADR 0037; contrato §9): na avançada, logo na
  # liberação; no ICP, quando o 19b grava a assinatura (Task 18). Grava o PDF
  # (o assinado, no ICP) em exam_files, marca os itens liberados e, no 21a,
  # registra o resultado de cada exame (source lab, valores do JSON
  # congelado, PDF próprio por resultado) — ou, na retificação, o substitui
  # (Exams::RectifyResult, Task 19) e o laudo anterior fica `rectified`. Tudo
  # ou nada: qualquer recusa levanta Failed e desfaz.
  module Lab
    module FinalizeReport
      class Failed < StandardError; end

      module_function

      def call(report:, pdf_bytes:, now: Time.current)
        ApplicationRecord.transaction(requires_new: true) do
          report.lock!
          raise Failed, "invalid_transition" unless report.status == "pending_signature"

          file = store!(pdf_bytes, report, now)
          report.update!(status: "released", released_at: now, pdf_file_id: file.id, pdf_sha256: file.sha256)
          previous = report.replaces_report
          previous&.lock!&.update!(status: "rectified")
          provider = report.exam_provider
          report.items.each do |item|
            item.update!(status: "released") unless item.status == "released"
            values = values_for(report, item)
            critical = values.any? { |v| LabResult::CRITICAL.include?(v.delete("flag")) }
            outcome = if previous
                        Exams::RectifyResult.call(order: item.exam_order, values: values, critical: critical, file: store!(pdf_bytes, report, now),
                                                  laboratory_name: provider.name, by_user: report.released_by_user, by_provider: provider, now: now)
                      else
                        Exams::RegisterResult.call(order: item.exam_order, source: "lab", released_on: now.in_time_zone.to_date,
                                                   laboratory_name: provider.name, critical: critical, values: Result.ok(values: values),
                                                   file: store!(pdf_bytes, report, now), by_user: report.released_by_user,
                                                   by_provider: provider, now: now)
                      end
            raise Failed, outcome.reason.to_s if outcome.failure?
          end
          RequisitionStatus.refresh!(report.lab_requisition)
          if previous
            DomainEvents.publish("lab_report.rectified", lab_report_id: report.id, replaces_report_id: previous.id)
          else
            DomainEvents.publish("lab_report.released", lab_report_id: report.id, lab_requisition_id: report.lab_requisition_id,
                                                        signature_kind: report.signature_kind, partial: report.partial)
          end
        end
        Result.ok(report: report)
      rescue Failed => e
        Result.fail(e.message.to_sym)
      end

      # A forma `values` do 21a (+ "flag", tirado por quem chama), do JSON
      # congelado: o 21a recebe exatamente o que o laudo diz.
      def values_for(report, item)
        version = item.lab_exam_version
        exam = report.content_data["exams"].find { |e| e["sigtap_code"] == version.lab_exam.sigtap_code && e["version"] == version.version }
        raise Failed, "invalid_content" unless exam

        exam["components"].map do |c|
          { "key" => c["key"], "label" => c["name"], "value" => c["value"], "unit" => c["unit"], "loinc" => c["loinc"], "flag" => c["flag"] }
        end
      end

      def store!(bytes, report, now)
        stored = Exams::Files.store(bytes: bytes, content_type: "application/pdf", user: report.released_by_user)
        raise Failed, "missing_pdf" if stored.failure?

        stored.payload[:file].tap { |file| file.update!(consumed_at: now) }
      end
      private_class_method :store!
    end
  end
  ```
  (`Exams::RegisterResult` aceita o arquivo já consumido: `file.update!(consumed_at: now) unless file.consumed?`.)

- [ ] **Step 4: Liberar.** `app/commands/lab/release.rb`:

  ```ruby
  # Liberação (contrato §5; spec §5.7; ADR 0037 Invariantes): só o liberador
  # ligado ao laboratório, com conselho do laboratório; médico (CRM) só ICP.
  # Avançada: TOTP do momento, consumido; o laudo nasce e já vale. ICP:
  # digital_signature utilizável e certificado ativo; o laudo nasce
  # pending_signature e o pedido do 19b vai na mesma transação. Crítico sem
  # aviso não bloqueia (vai à fila). Interruptores lidos fora da transação.
  module Lab
    module Release
      module_function

      def call(requisition:, item_ids:, signature:, totp:, by:, now: Time.current, rectifies: nil, reason: nil, contacts: nil)
        provider = requisition.exam_provider
        return Result.fail(:missing_role) unless Access.releaser_link(by, provider.id)

        council = Access.council(by)
        return Result.fail(:council_missing) unless council
        return Result.fail(:invalid_signature) unless %w[icp advanced].include?(signature)
        return Result.fail(:icp_required) if council == "CRM" && signature == "advanced"
        return Result.fail(:laboratory_incomplete) unless ReportContent.laboratory_ready?(provider)

        lead = LabReleaser.active.find_by(exam_provider_id: provider.id, technical_lead: true)&.user
        return Result.fail(:technical_lead_missing) unless lead && Access.council(lead)

        if signature == "icp"
          return Result.fail(:signature_unavailable) unless Signatures::Gate.usable?(Current.city)
          return Result.fail(:no_active_certificate) unless SignerCertificate.active.exists?(user_id: by.id)
        else
          step = by.mfa_enrolled? ? Mfa::Verify.totp_step_for(by, totp.to_s) : nil
          return Result.fail(:mfa_required) unless step && by.consume_totp_step!(step)
        end

        ids = Array(item_ids).map(&:to_s).uniq
        return Result.fail(:invalid_transition) if ids.empty?

        result = nil
        ApplicationRecord.transaction do
          requisition.lock!
          items = requisition.items.where(id: ids).includes(:lab_sample, lab_exam_version: :lab_exam).lock.to_a
          expected = rectifies ? "released" : "typed"
          bad = items.find { |i| i.status != expected || !i.lab_sample.accepted? }
          next result = Result.fail(:invalid_transition, details: { status: bad&.status }) if items.size != ids.size || bad
          if rectifies && !(ids - rectifies.items.pluck(:id)).empty?
            next result = Result.fail(:invalid_transition, details: { status: rectifies.status })
          end
          if LabReportItem.joins(:lab_report).where(lab_requisition_item_id: ids, lab_reports: { status: "pending_signature" }).exists?
            next result = Result.fail(:invalid_transition, details: { status: "pending_signature" })
          end

          others = requisition.items.reload.select(&:active?).reject { |i| ids.include?(i.id) }
          partial = rectifies ? rectifies.partial : others.any? { |i| i.status != "released" }
          built = ReportContent.build(requisition: requisition, items: items, released_by: by, lead: lead, partial: partial,
                                      signature_kind: signature, issued_at: now, replaces: rectifies, reason: reason)
          report = create_report!(requisition, provider, built, by, signature, partial, rectifies, reason, contacts)
          items.each { |item| LabReportItem.create!(lab_report: report, lab_requisition_item: item, created_at: now) }
          if signature == "advanced"
            finalized = FinalizeReport.call(report: report, pdf_bytes: ReportPdf.call(report.reload), now: now)
            raise ActiveRecord::Rollback if finalized.failure? && (result = finalized)
          else
            Signatures::OpenRequest.call(report, now: now, usable: true)
          end
          result = Result.ok(report: report.reload)
        end
        result
      end

      def create_report!(requisition, provider, built, by, signature, partial, rectifies, reason, contacts)
        attempts = 0
        begin
          attempts += 1
          ApplicationRecord.transaction(requires_new: true) do
            LabReport.create!(lab_requisition: requisition, exam_provider: provider, version: requisition.reports.maximum(:version).to_i + 1,
                              replaces_report: rectifies, partial: partial, status: "pending_signature", signature_kind: signature,
                              released_by_user: by, content: built.json, content_sha256: built.sha256,
                              verification_token: ClinicalDocuments::Codes.token, short_code: ClinicalDocuments::Codes.short_code,
                              rectification_reason: reason, rectification_contacts: contacts&.to_json)
          end
        rescue ActiveRecord::RecordNotUnique => e
          raise unless attempts < 3 && e.message.match?(/short_code|verification_token/)

          retry
        end
      end
      private_class_method :create_report!
    end
  end
  ```

- [ ] **Step 5: Fila, conferência, PDF e trilha.** `app/services/lab/release_json.rb`:

  ```ruby
  # Fila de liberação e conferência (dashboard D9): requisição com itens,
  # linhas de resultado (valor) e laudos. Quem chama grava a trilha.
  module Lab
    module ReleaseJson
      module_function

      def requisition(req, viewer:)
        patient = req.patient
        { id: req.id, number: req.number,
          patient: { id: patient.id, display_name: patient.display_name, birth_date: patient.birth_date, biological_sex: req.biological_sex },
          urgent: req.urgent,
          items: req.items.select(&:active?).map do |item|
            notified = LabCriticalNotice.where(lab_result_id: item.results.map(&:id)).pluck(:lab_result_id).to_set
            Json.item(item).merge(restriction_note: item.lab_sample.restriction_note,
                                  results: item.results.map { |row| EntryJson.result_row(row, item.lab_exam_version, notified) })
          end,
          reports: req.reports.map { |r| Json.report_summary(r, viewer: viewer) } }
      end
    end
  end
  ```
  `app/controllers/lab/releases_controller.rb`:

  ```ruby
  # Fila de liberação (contrato §5; dashboard D9) e o ato de liberar. Leitura
  # com valor: clinical_record.viewed (lab_report) por requisição.
  module Lab
    class ReleasesController < BaseController
      GRANT = ClinicalRecord::Access::Grant.new(kind: :lab_report, opening: nil, reason: nil)

      before_action :provider_param!, only: :index
      before_action :load_requisition, only: %i[show create]

      def index
        pending = LabReportItem.joins(:lab_report).where(lab_reports: { status: "pending_signature" }).select(:lab_requisition_item_id)
        ids = LabRequisitionItem.where(status: "typed").where.not(id: pending).select(:lab_requisition_id)
        reqs = LabRequisition.where(exam_provider_id: @provider.id, id: ids).order(urgent: :desc, created_at: :asc).limit(100)
                             .includes(:patient, reports: %i[released_by_user pdf_file], items: [ :results, :lab_sample, :exam_order, { lab_exam_version: :lab_exam } ])
        reqs.each { |req| trail(req) }
        render json: { requisitions: reqs.map { |req| Lab::ReleaseJson.requisition(req, viewer: Current.user) } }
      end

      def show
        trail(@requisition)
        render json: Lab::ReleaseJson.requisition(@requisition, viewer: Current.user)
      end

      def create
        result = Lab::Release.call(requisition: @requisition, item_ids: params[:item_ids], signature: params[:signature],
                                   totp: params[:totp], by: Current.user)
        return render_failure(result) if result.failure?

        render json: Lab::Json.report_summary(result.payload[:report], viewer: Current.user), status: :created
      end

      private

      def load_requisition
        @requisition = uuid?(params[:id]) && LabRequisition.find_by(id: params[:id])
        return not_found unless @requisition

        forbid unless Lab::Access.member?(Current.user, @requisition.exam_provider_id)
      end

      def trail(req)
        ClinicalRecord::Trail.viewed!(patient: req.patient, user: Current.user, grant: GRANT, extra: { lab_requisition_id: req.id })
      end
    end
  end
  ```
  `app/controllers/lab/reports_controller.rb` (cresce na Task 19):

  ```ruby
  module Lab
    class ReportsController < BaseController
      before_action :load_report

      # URL curta do PDF (A4 do 21a): a trilha é gravada no download.
      def pdf
        return not_found unless @report.pdf_file

        render json: { pdf_url: Exams::Files.url_for(@report.pdf_file, user: Current.user) }
      end

      private

      def load_report
        @report = uuid?(params[:id]) && LabReport.find_by(id: params[:id])
        return not_found unless @report

        forbid unless Lab::Access.member?(Current.user, @report.exam_provider_id)
      end
    end
  end
  ```
  Em `Exams::FilesController#record_result_view`, depois da linha do `ExamResult`:

  ```ruby
      # ADR 0037: o PDF do laudo do laboratório (lab_reports.pdf_file_id).
      report = result ? nil : LabReport.find_by(pdf_file_id: file.id)
      if report
        ClinicalRecord::Trail.viewed!(patient: report.lab_requisition.patient, user: Current.user,
                                      grant: ClinicalRecord::Access::Grant.new(kind: :lab_report, opening: nil, reason: nil),
                                      extra: { lab_report_id: report.id, lab_requisition_id: report.lab_requisition_id })
      end
  ```
  Rotas no `scope "/lab"`:

  ```ruby
      get  "releases",                  to: "releases#index"
      get  "releases/:id",              to: "releases#show"
      post "requisitions/:id/release",  to: "releases#create"
      get  "reports/:id/pdf",           to: "reports#pdf", format: false
  ```

- [ ] **Step 6: Rode:** `rspec spec/requests/lab/release_spec.rb spec/commands/lab/release_concurrency_spec.rb spec/requests/exams` → verde (o caminho ICP e a retificação são provados nas Tasks 18 e 19).

- [ ] **Step 7: Commit:**

  ```bash
  git add app/commands/lab/release.rb app/commands/lab/finalize_report.rb app/services/lab/release_json.rb app/controllers/lab/releases_controller.rb app/controllers/lab/reports_controller.rb app/controllers/exams/files_controller.rb config/routes.rb spec/requests/lab/release_spec.rb spec/commands/lab/release_concurrency_spec.rb
  git commit -m "feat: release lab reports with advanced signature and record results in exam tracking

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 18: Laudo ICP pelo 19b e conferência pública (`/v`) do laudo (F-21.20)

**Files:**
- Modify: `app/services/signatures/{document_types,documents,canonical,signing,json}.rb`, `app/commands/signatures/{to_paper,return_to_paper}.rb`, `app/controllers/signatures/{base_controller,signatures_controller}.rb`
- Modify: `app/services/clinical_documents/{verification,page}.rb`, `app/controllers/document_verifications_controller.rb`
- Test: `spec/requests/lab/release_icp_spec.rb`, `spec/requests/lab/report_verification_spec.rb`

**Interfaces:**
- Consumes: `Lab::Release`, `Lab::FinalizeReport` (Task 17), `Lab::ReportContent::SCHEMA_PATH`, `Lab::ReportPdf` (Task 15); helpers do 19b (`stub_psc_mock!`, `stub_signer!`, `linked_certificate!`, `fake_psc`, `sign_document!`).
- Produces: `Signatures::DocumentTypes::MODELS["LabReport"] = "lab_report"`; `Signatures::Canonical.lab_report(report) → Built` (sha256 = `content_sha256`); o 19b assina o laudo e chama `Lab::FinalizeReport` na mesma transação da gravação; `GET /signature/requests` lista o laudo pendente (`document_type: "lab_report"`, `consultation_id: null`, `patient_display_name`); `POST /signature/requests/:id/return_to_paper` de laudo → 422 `not_applicable` (dashboard D19); só o sistema (interruptor desligado) põe o laudo em `withdrawn`; `require_professional` do 19b aceita `lab_releaser`; `/v/:token`, `/v/lookup` aceitam o laudo: `{ kind: "lab_report", issued_at, professional: { name, council }, unit: <laboratório>, patient: { initials, birth_year }, status: "valid"|"rectified"|"awaiting_signature", mode: "digital"|"advanced", simulated, pdf_sha256 }` — sem PDF público.

- [ ] **Step 1: Testes que falham** — `spec/requests/lab/release_icp_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.7 (ICP pelo 19b): o laudo nasce pending_signature, aparece nas
  # pendências do liberador, o 21a só recebe depois da assinatura (PDF
  # assinado); voltar ao papel não se aplica; interruptor desligado retira.
  RSpec.describe "Laboratório — liberação com ICP", type: :request do
    before do
      lab_city!
      stub_psc_mock!(City.find(Current.city.id))
      stub_signer!
    end

    let(:provider) { lab_provider! }
    let(:doctor) do
      lab_releaser!(provider, council: "CRM", lead: true).tap { |u| u.professional.update!(cpf: SignatureHelpers::DOCTOR_CPF) }
    end
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }
    let(:item) { type!(req.items.first, { glicose: "98" }) }

    it "pendente → assinado: laudo liberado com o PDF assinado e resultado no 21a" do
      sign_in_as(doctor)
      json_post "/lab/requisitions/#{req.id}/release", item_ids: [ item.id ], signature: "icp"
      expect(json_body["error"]).to eq("no_active_certificate")
      linked_certificate!(doctor, provider: "simulated", leaf: fake_psc("simulated").leaf(doctor.professional.cpf))
      json_post "/lab/requisitions/#{req.id}/release", item_ids: [ item.id ], signature: "icp"
      expect(json_body).to include("status" => "pending_signature", "pdf_url" => nil, "short_code" => nil)
      report = LabReport.find(json_body["id"])
      expect(item.exam_order.reload.result).to be_nil
      get "/signature/requests"
      expect(json_body["items"].sole).to include("document_type" => "lab_report", "consultation_id" => nil,
                                                 "patient_display_name" => req.patient.display_name)
      json_post "/signature/requests/#{json_body['items'].sole['id']}/return_to_paper", reason: "Certificado com problema hoje"
      expect([ response.status, json_body["error"] ]).to eq([ 422, "not_applicable" ])
      json_post "/lab/requisitions/#{req.id}/release", item_ids: [ item.id ], signature: "icp"
      expect(json_body["error"]).to eq("invalid_transition")

      signature = sign_document!(report, author: doctor, provider: "simulated")
      expect(signature.canonical_sha256).to eq(report.content_sha256)
      expect(report.reload).to have_attributes(status: "released", pdf_sha256: Digest::SHA256.hexdigest(signature.signed_pdf_bytes))
      result = item.exam_order.reload.result
      expect(result.source).to eq("lab")
      expect(result.pdf_file.bytes).to eq(signature.signed_pdf_bytes)
    end

    it "interruptor desligado antes de assinar: o laudo sai (withdrawn) e o item volta à fila" do
      linked_certificate!(doctor, provider: "simulated", leaf: fake_psc("simulated").leaf(doctor.professional.cpf))
      report = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "icp", totp: nil, by: doctor).payload[:report]
      Platform::Features.set!(city: City.find(Current.city.id), key: "digital_signature", enabled: false, maintainer: ledi_maintainer!)
      ApplicationRecord.transaction { Signatures::SignPending.call(request_id: report.signature_request.id) }
      expect(report.reload.status).to eq("withdrawn")
      expect(DomainEvent.where(name: "lab_report.withdrawn").count).to eq(1)
      sign_in_as(doctor)
      get "/lab/releases", params: { provider_id: provider.id }
      expect(json_body["requisitions"].map { |r| r["id"] }).to eq([ req.id ])
    end
  end
  ```
  `spec/requests/lab/report_verification_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5 (conferência pública): a página /v do 19c aceita o laudo, como
  # "Laudo laboratorial", sem resultado, sem CPF e sem PDF público.
  RSpec.describe "Conferência pública do laudo", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider!(name: "Laboratório Municipal Central") }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }

    it "por token e por código + ano; sem valor; trilha sem dado pessoal" do
      item = type!(req.items.first, { glicose: "98" })
      report = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead).payload[:report]
      get "/v/#{report.verification_token}", headers: { "Accept" => "application/json" }
      expect(json_body).to include("kind" => "lab_report", "status" => "valid", "mode" => "advanced", "unit" => "Laboratório Municipal Central",
                                   "pdf_sha256" => report.pdf_sha256)
      expect(response.body).not_to include("98", req.patient.cpf)
      year = req.patient.birth_date[0, 4]
      post "/v/lookup", params: { short_code: report.short_code, birth_year: year }.to_json, headers: { "CONTENT_TYPE" => "application/json" }
      expect(json_body["kind"]).to eq("lab_report")
      get "/v/#{report.verification_token}"
      expect(response.body).to include("Laudo laboratorial", "Assinatura eletrônica avançada")
      get "/v/#{report.verification_token}/signed.pdf"
      expect(response).to have_http_status(:not_found)
      expect(DocumentVerificationLookup.where(lab_report_id: report.id).count).to eq(3)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/release_icp_spec.rb spec/requests/lab/report_verification_spec.rb`.

- [ ] **Step 3: O 19b assina o laudo.**
  - `Signatures::DocumentTypes::MODELS`: acrescente `"LabReport" => "lab_report"` (comentário "ADR 0037: laudo do laboratório municipal, sem consulta").
  - `Signatures::Documents.for`, no `case`:

    ```ruby
        when LabReport
          Prepared.new(request: request, canonical: Canonical.lab_report(document), pdf: Lab::ReportPdf.call(document, footer: footer))
    ```
  - `Signatures::Canonical`: `LAB_REPORT_SCHEMA = "rotasaude.lab_report.v1".freeze`; em `SCHEMA_FILES`, `LAB_REPORT_SCHEMA => Lab::ReportContent::SCHEMA_PATH`; em `for`, `when LabReport then lab_report(document)`; e

    ```ruby
      # ADR 0037: o laudo assina o JSON CONGELADO na liberação (lab_reports.content);
      # o sha256 é o content_sha256 (mesma serialização RFC 8785).
      def lab_report(report)
        raise Invalid, "not_signable" unless report.status == "pending_signature"

        built(LAB_REPORT_SCHEMA, report.content_data)
      end
    ```
  - `Signatures::Signing.chronological`: `when LabReport then document.created_at`; `chain_key`: `%w[ClinicalDocument LabReport].include?(request.document_type) ? [ request.document_type, request.document_id ] : request.consultation_id`. Em `store`, logo depois de `request.update!(status: "signed", …)`:

    ```ruby
          # ADR 0037: o laudo passa a valer com a assinatura, na mesma transação.
          if request.document_type == "LabReport"
            finalized = Lab::FinalizeReport.call(report: request.document, pdf_bytes: pades.signature, now: now)
            raise Lab::FinalizeReport::Failed, finalized.reason.to_s if finalized.failure?
          end
    ```
    e, junto dos `rescue` de `store`, `rescue Lab::FinalizeReport::Failed` → `"verification_failed"` (definitivo: o laudo não pode valer; o pedido fica pendente com o motivo).
  - `Signatures::ToPaper.call`, depois do bloco do `ClinicalDocument`:

    ```ruby
      # ADR 0037: laudo não vai ao papel. Só o sistema (interruptor desligado)
      # devolve o pedido: o laudo sai (withdrawn) e os itens voltam à fila.
      if request.document_type == "LabReport"
        LabReport.where(id: request.document_id, status: "pending_signature").find_each do |report|
          report.update!(status: "withdrawn")
          DomainEvents.publish("lab_report.withdrawn", lab_report_id: report.id)
        end
      end
    ```
  - `Signatures::ReturnToPaper.call`, depois de `next Result.fail(:not_pending) unless request.pending?`: `next Result.fail(:not_applicable) if request.document_type == "LabReport"`; e `not_applicable: :unprocessable_entity` em `Signatures::BaseController::ERROR_STATUS`.
  - `Signatures::Json.requests`:

    ```ruby
      lab_ids = rows.select { |row| row.document_type == "LabReport" }.map(&:document_id)
      labs = LabReport.where(id: lab_ids).includes(lab_requisition: :patient).index_by(&:id)
    ```
    no `case`, `when "LabReport" then labs[row.document_id]&.created_at`, e `patient_display_name: consultation&.patient&.display_name || labs[row.document_id]&.lab_requisition&.patient&.display_name`.
  - `Signatures::BaseController`:

    ```ruby
      # ADR 0037 (dashboard D19): o liberador do laboratório assina laudo.
      def require_professional
        return if Current.user.has_role?("lab_releaser")

        super
      end
    ```
    (como método privado da classe; `super` chama o de `AttendanceAccess`).
  - `Signatures::SignaturesController#set_signature`, antes do `Consultation.find`: `return render(json: { error: "not_found" }, status: :not_found) unless @signature.signature_request.consultation_id` (laudo se lê por `/lab`).

- [ ] **Step 4: Conferência pública.** Em `ClinicalDocuments::Verification`:

  ```ruby
      def by_token(token)
        return trail!(nil, "token") unless token.is_a?(String) && token.match?(/\A[A-Za-z0-9_-]{22}\z/)

        record = ClinicalDocument.where.not(kind: "controlled_notification_record").find_by(verification_token: token) ||
                 public_lab_reports.find_by(verification_token: token)
        trail!(record, "token")
      end

      def lookup(short_code:, birth_year:)
        code = Codes.normalize(short_code)
        year = birth_year.to_s.match?(/\A\d{4}\z/) ? birth_year.to_s.to_i : nil
        record = code && year ? (ClinicalDocument.where.not(kind: "controlled_notification_record").find_by(short_code: code) ||
                                 public_lab_reports.find_by(short_code: code)) : nil
        record = nil if record && birth_year_of(record) != year
        trail!(record, "short_code")
      end

      # ADR 0037: laudo pendente, liberado ou retificado (o retirado não existe para fora).
      def public_lab_reports = LabReport.where(status: %w[pending_signature released rectified])

      def lab_json(report)
        content = report.content_data
        signer = content["released_by"]
        signature = report.signature_kind == "icp" ? report.signature_request&.signature : nil
        status = { "pending_signature" => "awaiting_signature", "released" => "valid", "rectified" => "rectified" }.fetch(report.status)
        { kind: "lab_report", issued_at: (report.released_at || report.created_at).iso8601,
          professional: { name: signer["name"], council: "#{signer['council']}-#{signer['council_uf']} #{signer['council_number']}" },
          unit: content.dig("laboratory", "name"),
          patient: { initials: Patients::Initials.call(content.dig("patient", "name")), birth_year: birth_year_of(report) },
          status: status, mode: report.signature_kind == "icp" ? "digital" : "advanced",
          simulated: signature&.simulated? || false, pdf_sha256: report.pdf_sha256 }
      end
  ```
  `public_json(record)` passa a começar com `return lab_json(record) if record.is_a?(LabReport)`; `signature(record)` devolve `nil` para `LabReport`; `trail!` grava `clinical_document_id: (record.id if record.is_a?(ClinicalDocument)), lab_report_id: (record.id if record.is_a?(LabReport))`; `person(record)` para laudo = `record.lab_requisition.patient`; `birth_year_of` usa `person`. Acrescente `public_lab_reports` e `lab_json` ao `private_class_method`. Em `DocumentVerificationsController#signed_pdf`: `signature = document.is_a?(ClinicalDocument) && !document.cancelled? && ClinicalDocuments::Verification.signature(document)`. Em `ClinicalDocuments::Page`: `KINDS["lab_report"] = "Laudo laboratorial"`; `STATUS["rectified"] = "Retificado — substituído por um laudo novo"`; na linha "Modo": `data[:mode] == "advanced" ? "Assinatura eletrônica avançada (Lei 14.063/2020)" : (data[:mode] == "digital" ? … : "Papel (assinatura à mão)")`; linha `[ "SHA-256 do PDF", data[:pdf_sha256] ]` quando houver; no script do `form`, `const status = { valid: "Válido", cancelled: "Cancelado", rectified: "Retificado", awaiting_signature: "Aguardando assinatura digital" }[j.status] || j.status;`.

- [ ] **Step 5: Rode:** `rspec spec/requests/lab/release_icp_spec.rb spec/requests/lab/report_verification_spec.rb spec/requests/document_verifications_spec.rb spec/requests/signature* spec/services/signatures spec/commands/signatures spec/jobs/signatures` → verde (o 19b inteiro, porque `Signing`, `ToPaper` e `Json` mudaram).

- [ ] **Step 6: Commit** e mova o card **F-21.20** para **Done**:

  ```bash
  git add app/services/signatures app/commands/signatures/to_paper.rb app/commands/signatures/return_to_paper.rb app/controllers/signatures/base_controller.rb app/controllers/signatures/signatures_controller.rb app/services/clinical_documents/verification.rb app/services/clinical_documents/page.rb app/controllers/document_verifications_controller.rb spec/requests/lab/release_icp_spec.rb spec/requests/lab/report_verification_spec.rb
  git commit -m "feat: sign lab reports with ICP through digital signature and verify them publicly

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 8 — Retificação, entrega, GAL e relatórios

### Task 19: Retificação no laboratório e no 21a (F-21.22, parte do laboratório municipal de api#85)

**Files:**
- Create: `app/commands/exams/rectify_result.rb`
- Modify: `app/models/{exam_result,exam_order}.rb`, `app/services/exams/{transition,json}.rb`, `app/controllers/lab/reports_controller.rb` (`rectify`), `config/routes.rb`, `spec/fixtures/exams/exam-order-v1.json` (cópia do contracts v1.1, Task 1)
- Test: `spec/requests/lab/rectification_spec.rb`, `spec/commands/exams/rectify_result_spec.rb`

**Interfaces:**
- Consumes: `Lab::Release` com `rectifies:`, `Lab::FinalizeReport` (que já chama `Exams::RectifyResult` — Task 17), `Lab::EnterResults` em item `released` (D7).
- Produces: `Exams::RectifyResult.call(order:, values:, critical:, file:, laboratory_name:, by_user:, by_provider:, now: Time.current) → Result.ok(result:)` | `fail(:invalid_transition, details: { status: })`; `ExamResult.current` (escopo), `ExamResult#replaces_result`; `ExamOrder#result` (o vigente) e `#results` (todos); `Exams::Transition::FLOW["result_rectified"]`; `<exam_order>` clínico com `superseded_results` (com `superseded_at`) e `result.replaces_result_id` (dashboard D16); `POST /lab/reports/:id/rectify { item_ids, reason, contacts: [ { person, channel, at } ], signature, totp? }` → 201 `<lab_report_summary>`.

- [ ] **Step 1: Testes que falham** — `spec/commands/exams/rectify_result_spec.rb`:

  ```ruby
  require "rails_helper"

  # Desvio 4: retificação sem UPDATE — novo resultado com replaces_result_id; o
  # vigente é o que ninguém substituiu; um original por pedido; ciência perdida.
  RSpec.describe Exams::RectifyResult do
    before { lab_city! }

    let(:unit) { create_unit("UBS Retifica") }
    let(:doctor) { doctor!(unit) }
    let(:order) do
      exam_catalog_item!
      exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor).tap do |o|
        Exams::Steps.schedule(order: o, actor_user: doctor, params: { provider_id: exam_provider!.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
        Exams::RegisterResult.call(order: o.reload, source: "unit", file: pdf_file!(user: doctor), released_on: Time.zone.today, laboratory_name: "Lab",
                                   critical: false, values: [ { "key" => "hba1c", "value" => "6.1" } ], by_user: doctor)
        Exams::Acknowledge.call(order: o.reload, by: doctor)
      end
    end

    it "substitui o vigente, volta a resulted com prazo novo, publica result_rectified" do
      first = order.reload.result
      outcome = described_class.call(order: order, values: [ { "key" => "hba1c", "label" => "HbA1c", "value" => "9.1", "unit" => "%", "loinc" => "4548-4" } ],
                                     critical: false, file: pdf_file!(user: doctor), laboratory_name: "Lab", by_user: doctor, by_provider: nil)
      expect(outcome).to be_ok
      order.reload
      expect(order).to have_attributes(status: "resulted", stage: "acknowledge")
      expect(order.result).to eq(outcome.payload[:result])
      expect(order.result.replaces_result).to eq(first)
      expect(order.results.count).to eq(2)
      expect(ExamResult.current.where(exam_order_id: order.id).sole).to eq(order.result)
      expect(order.events.last.kind).to eq("result_rectified")
      expect(DomainEvent.where(name: "exam_order.result_rectified").sole.payload.keys).to contain_exactly("exam_order_id", "critical")
      expect { ExamResult.create!(first.attributes.except("id", "created_at", "pdf_file_id").merge("pdf_file_id" => pdf_file!(user: doctor).id)) }
        .to raise_error(ActiveRecord::RecordNotUnique)
    end
  end
  ```
  `spec/requests/lab/rectification_spec.rb` (Review Focus 4):

  ```ruby
  require "rails_helper"

  # Spec §5.9 e contrato §5/§9: retificação só por laudo novo com motivo e
  # contatos; o anterior fica `rectified`; no 21a o resultado é substituído,
  # o exame volta à ciência; o cidadão vê só o vigente; o profissional, os dois.
  RSpec.describe "Laboratório — retificação", type: :request do
    before do
      lab_city!
      exam_catalog_item!(sigtap_code: "0202010473", release_policy: "immediate", result_fields: [])
    end

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:patient_and_citizen) { exam_patient!(exam_seq!) }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ], patient: patient_and_citizen.first) }
    let(:item) { type!(req.items.first, { glicose: "98" }) }
    let(:order) { item.exam_order }

    it "o que cada um vê depois da retificação" do
      first = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead).payload[:report]
      Exams::Acknowledge.call(order: order.reload, by: order.requester_user)
      old_pdf = order.reload.result.pdf_file.bytes
      sign_in_as(lead)
      json_post "/lab/results", item_id: item.id, values: [ { analyte_key: "glicose", value: "450" } ]
      expect(response).to have_http_status(:created)
      body = { item_ids: [ item.id ], reason: "Erro de digitação do valor", signature: "advanced",
               contacts: [ { person: "Dra. Helena", channel: "telefone", at: Time.current.iso8601 } ] }
      travel 31.seconds do
        json_post "/lab/reports/#{first.id}/rectify", body.merge(reason: "x", totp: totp_for(lead))
        expect(json_body["error"]).to eq("reason_required")
        json_post "/lab/reports/#{first.id}/rectify", body.merge(contacts: [], totp: totp_for(lead))
        expect(json_body["error"]).to eq("contacts_required")
      end
      travel 62.seconds do
        json_post "/lab/reports/#{first.id}/rectify", body.merge(totp: totp_for(lead))
      end
      expect(response).to have_http_status(:created)
      expect(json_body).to include("version" => 2, "replaces_report_id" => first.id, "status" => "released")
      expect(first.reload.status).to eq("rectified")
      second = LabReport.find(json_body["id"])
      expect(second.content_data).to include("rectification_reason" => "Erro de digitação do valor", "replaces_report_id" => first.id)
      expect(JSON.parse(second.rectification_contacts).sole).to include("person" => "Dra. Helena")

      order.reload
      expect(order).to have_attributes(status: "resulted")
      expect(order.result.values_list.sole["value"]).to eq("450")
      expect(order.result).to be_critical

      sign_in_as(order.requester_user)
      get "/exams/orders/#{order.id}"
      expect(json_body["result"]["replaces_result_id"]).to be_present
      expect(json_body["superseded_results"].sole).to include("superseded_at" => be_present)
      expect(json_body["superseded_results"].sole["values"].sole["value"]).to eq("98")

      sign_in_citizen(patient_and_citizen.last.phone)
      get "/citizen/exams/#{order.id}/pdf", params: { citizen_id: patient_and_citizen.last.id }
      expect(response.body.b).to eq(order.result.pdf_file.bytes)
      expect(response.body.b).not_to eq(old_pdf)
      get "/citizen/exams", params: { citizen_id: patient_and_citizen.last.id }
      expect(json_body["exams"].size).to eq(1)
    end

    it "só de laudo liberado; itens do laudo; quem não libera 403" do
      report = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead).payload[:report]
      sign_in_as(lab_staff!)
      json_post "/lab/reports/#{report.id}/rectify", item_ids: [ item.id ], reason: "Erro de digitação", signature: "advanced",
                                                     contacts: [ { person: "X", channel: "telefone", at: Time.current.iso8601 } ]
      expect([ response.status, json_body["error"] ]).to eq([ 403, "missing_role" ])
      sign_in_as(lead)
      travel 31.seconds do
        json_post "/lab/reports/#{report.id}/rectify", item_ids: [ SecureRandom.uuid ], reason: "Erro de digitação", signature: "advanced",
                                                       contacts: [ { person: "X", channel: "telefone", at: Time.current.iso8601 } ], totp: totp_for(lead)
      end
      expect(json_body["error"]).to eq("invalid_transition")
    end
  end
  ```
  Copie `contracts/exams/exam-order-v1.json` (v1.1) para `spec/fixtures/exams/exam-order-v1.json` e confira que as specs do 21a que validam a forma continuam verdes.

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/commands/exams/rectify_result_spec.rb spec/requests/lab/rectification_spec.rb`.

- [ ] **Step 3: 21a.** `app/models/exam_result.rb`, acrescente:

  ```ruby
    # ADR 0037 (Desvio 4): a retificação acrescenta; o vigente é o que nenhum
    # outro substituiu.
    belongs_to :replaces_result, class_name: "ExamResult", optional: true
    scope :current, -> { where.not(id: ExamResult.where.not(replaces_result_id: nil).select(:replaces_result_id)) }
  ```
  `app/models/exam_order.rb`: troque `has_one :result, …` por

  ```ruby
    has_one :result, -> { current }, class_name: "ExamResult", inverse_of: :exam_order
    has_many :results, -> { order(:created_at, :id) }, class_name: "ExamResult", inverse_of: :exam_order
  ```
  `Exams::Transition::FLOW`: `"result_rectified" => { from: %w[resulted acknowledged], to: "resulted" }` (comentário: "ADR 0037: laudo retificado volta à ciência"). `app/commands/exams/rectify_result.rb`:

  ```ruby
  # Retificação do resultado (ADR 0037; contrato §9): novo exam_result com
  # replaces_result_id (nada muda no anterior), evento result_rectified (o
  # exame volta a `resulted` e perde a ciência), escalonamento por crítico
  # (um por motivo, A31). Mesma trava e savepoint do RegisterResult.
  module Exams
    module RectifyResult
      module_function

      def call(order:, values:, critical:, file:, laboratory_name:, by_user:, by_provider:, now: Time.current)
        result = nil
        ApplicationRecord.transaction(requires_new: true) do
          order.lock!
          previous = order.reload.result
          next result = Result.fail(:invalid_transition, details: { status: order.status }) unless previous && %w[resulted acknowledged].include?(order.status)

          file.update!(consumed_at: now) unless file.consumed?
          moved = Transition.call(order: order, kind: "result_rectified", actor_user: by_user, actor_provider: by_provider,
                                  event_payload: { critical: critical }, now: now)
          if moved.failure?
            result = moved
            raise ActiveRecord::Rollback
          end
          stored = ExamResult.create!(exam_order: order, patient: order.patient, sigtap_code: order.sigtap_code, origin: "order",
                                      source: previous.source, pdf_file: file, values: values.to_json, released_on: now.in_time_zone.to_date,
                                      laboratory_name: laboratory_name, critical: critical, exam_provider_id: by_provider&.id || order.exam_provider_id,
                                      registered_by_user: by_user, registered_by_provider: by_provider, replaces_result: previous)
          Escalate.call(order: order, reason: "critical", now: now) if critical
          result = Result.ok(result: stored)
        end
        result
      end
    end
  end
  ```
  `Exams::Json.order`: no bloco `if events`, depois de `result_fields`:

  ```ruby
        # Dashboard D16: laudos substituídos, só na visão clínica, do mais novo ao mais antigo.
        if clinical && order.result
          replacing = ExamResult.where(replaces_result_id: order.results.map(&:id)).pluck(:replaces_result_id, :created_at).to_h
          body[:superseded_results] = order.results.reject { |r| r.id == order.result.id }.reverse.map do |r|
            result(r, view: view, user: user).merge(superseded_at: replacing[r.id]&.iso8601)
          end
        end
  ```
  e em `Exams::Json.result`, na visão clínica, `replaces_result_id: result.replaces_result_id`. `body[:gal_requisition]` entra na Task 20.

- [ ] **Step 4: Rota da retificação.** Em `Lab::ReportsController`:

  ```ruby
      CHANNELS = LabCriticalNotice::CHANNELS

      # Contrato §5 (RDC 978 art. 170): laudo novo com motivo e quem foi avisado.
      def rectify
        return render_failure(Result.fail(:invalid_transition, details: { status: @report.status })) unless @report.status == "released"

        reason = params[:reason].is_a?(String) ? params[:reason].strip : ""
        return render_failure(Result.fail(:reason_required)) unless reason.length.between?(3, 500)

        contacts = Array(params[:contacts]).map { |c| c.respond_to?(:to_unsafe_h) ? c.to_unsafe_h : c }
        ok = contacts.size.between?(1, 10) && contacts.all? do |c|
          c.is_a?(Hash) && c["person"].is_a?(String) && c["person"].strip.length.between?(1, 120) && CHANNELS.include?(c["channel"]) &&
            (Time.iso8601(c["at"].to_s) rescue nil)
        end
        return render_failure(Result.fail(:contacts_required)) unless ok

        result = Lab::Release.call(requisition: @report.lab_requisition, item_ids: params[:item_ids], signature: params[:signature],
                                   totp: params[:totp], by: Current.user, rectifies: @report, reason: reason,
                                   contacts: contacts.map { |c| c.slice("person", "channel", "at") })
        return render_failure(result) if result.failure?

        render json: Lab::Json.report_summary(result.payload[:report], viewer: Current.user), status: :created
      end
  ```
  Rota: `post "reports/:id/rectify", to: "reports#rectify"`. (O `Lab::Release` com `rectifies:` exige itens `released` do laudo e liberador ligado; o laudo novo leva motivo e contatos cifrados.)

- [ ] **Step 5: Rode:** `rspec spec/commands/exams spec/requests/lab/rectification_spec.rb spec/requests/exams spec/requests/citizen_api/exams_spec.rb spec/services/timeline` → verde (a linha do tempo e "Meus exames" leem `order.result`, o vigente).

- [ ] **Step 6: Commit** e mova o card **F-21.22** para **Done** (comente em rotasaude/api#85 que a parte do laboratório municipal foi entregue — **só com autorização** para comentar):

  ```bash
  git add app/commands/exams/rectify_result.rb app/models/exam_result.rb app/models/exam_order.rb app/services/exams/transition.rb app/services/exams/json.rb app/controllers/lab/reports_controller.rb config/routes.rb spec/fixtures/exams/exam-order-v1.json spec/commands/exams/rectify_result_spec.rb spec/requests/lab/rectification_spec.rb
  git commit -m "feat: rectify municipal lab reports and supersede exam results

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 20: Entrega, situação e registro do GAL (F-21.24)

**Files:**
- Create: `app/commands/lab/deliver.rb`, `app/commands/exams/record_gal.rb`, `app/controllers/exams/gal_controller.rb`
- Modify: `app/controllers/lab/requisitions_controller.rb` (`deliver`), `app/services/exams/json.rb` (`gal_requisition`), `config/routes.rb`
- Test: `spec/requests/lab/delivery_and_gal_spec.rb`

**Interfaces:**
- Produces: `Lab::Deliver.call(requisition:, delivered_to:, by:) → Result.ok(requisition:)` | `fail(:invalid_transition|:invalid_delivered_to|:missing_role)`; `Exams::RecordGal.call(order:, value:, by:) → Result.ok(order:)` | `fail(:invalid_gal_requisition|:gal_already_registered|:not_unit_staff)`; rotas `POST /lab/requisitions/:id/deliver { delivered_to }` → `<lab_requisition>`; `POST /exams/orders/:id/gal_requisition { gal_requisition }` → `<exam_order>` (dashboard D17); `<exam_order>.gal_requisition` nas duas visões.

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/delivery_and_gal_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.10–§5.11 e §4 (GAL): entrega do impresso (quem retirou e quando),
  # situação por número; número do GAL registrado uma vez no exame do 21a.
  RSpec.describe "Laboratório — entrega e GAL", type: :request do
    before { lab_city! }

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:staff) { lab_staff! }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }

    it "entrega só liberada; situação delivered; quem retirou cifrado" do
      item = type!(req.items.first, { glicose: "98" })
      sign_in_as(staff)
      json_post "/lab/requisitions/#{req.id}/deliver", delivered_to: "Maria (a própria)"
      expect(json_body["error"]).to eq("invalid_transition")
      Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead)
      json_post "/lab/requisitions/#{req.id}/deliver", delivered_to: "Maria (a própria)"
      expect(json_body).to include("status" => "delivered", "delivered_to" => "Maria (a própria)")
      get "/lab/requisitions", params: { number: req.number }
      expect(json_body["requisitions"].sole["status"]).to eq("delivered")
      expect(DomainEvent.where(name: "lab_requisition.delivered").sole.payload.keys).to contain_exactly("lab_requisition_id")
    end

    it "GAL: unidade ou laboratório registra uma vez; formato; estranho 403" do
      order = req.items.first.exam_order
      sign_in_as(doctor!(create_unit("UBS Longe")))
      json_post "/exams/orders/#{order.id}/gal_requisition", gal_requisition: "2026-000123"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(staff)
      json_post "/exams/orders/#{order.id}/gal_requisition", gal_requisition: "2026/000123"
      expect(json_body["error"]).to eq("invalid_gal_requisition")
      json_post "/exams/orders/#{order.id}/gal_requisition", gal_requisition: "2026-000123"
      expect(json_body["gal_requisition"]).to eq("2026-000123")
      json_post "/exams/orders/#{order.id}/gal_requisition", gal_requisition: "2026-000124"
      expect([ response.status, json_body["error"] ]).to eq([ 409, "gal_already_registered" ])
      expect(DomainEvent.where(name: "exam_order.gal_registered").sole.payload.keys).to contain_exactly("exam_order_id")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/delivery_and_gal_spec.rb`.

- [ ] **Step 3: Implemente.** `app/commands/lab/deliver.rb`:

  ```ruby
  # Entrega do impresso (spec §5.10): requisição liberada; quem retirou
  # (cifrado) e quando; situação delivered.
  module Lab
    module Deliver
      module_function

      def call(requisition:, delivered_to:, by:, now: Time.current)
        return Result.fail(:missing_role) unless Access.member?(by, requisition.exam_provider_id) ||
                                                  Exams::Access.unit_staff?(by, requisition.origin_unit_id)

        name = delivered_to.is_a?(String) ? delivered_to.strip : ""
        return Result.fail(:invalid_delivered_to) unless name.length.between?(1, 120)

        ApplicationRecord.transaction do
          requisition.lock!
          next Result.fail(:invalid_transition, details: { status: requisition.status }) unless requisition.status == "released"

          requisition.update!(status: "delivered", delivered_to: name, delivered_at: now, delivered_by_user: by)
          DomainEvents.publish("lab_requisition.delivered", lab_requisition_id: requisition.id)
          Result.ok(requisition: requisition)
        end
      end
    end
  end
  ```
  `app/commands/exams/record_gal.rb`:

  ```ruby
  # Número da requisição no GAL (spec §4; F-21.24; dashboard D17): registrado
  # uma vez (o guard do banco também recusa trocar), pela unidade do exame ou
  # pelo laboratório municipal. O laudo do LACEN entra como resultado externo.
  module Exams
    module RecordGal
      FORMAT = /\A[0-9A-Za-z-]{1,30}\z/

      module_function

      def call(order:, value:, by:)
        allowed = Access.unit_staff?(by, order.unit_id) || Lab::Access.staff?(by)
        return Result.fail(:not_unit_staff) unless allowed

        number = value.is_a?(String) ? value.strip : ""
        return Result.fail(:invalid_gal_requisition) unless number.match?(FORMAT)

        ApplicationRecord.transaction do
          order.lock!
          next Result.fail(:gal_already_registered) if order.gal_requisition

          order.update!(gal_requisition: number)
          DomainEvents.publish("exam_order.gal_registered", exam_order_id: order.id)
          Result.ok(order: order)
        end
      end
    end
  end
  ```
  `app/controllers/exams/gal_controller.rb`:

  ```ruby
  module Exams
    class GalController < BaseController
      def create
        order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id])
        return not_found unless order

        result = Exams::RecordGal.call(order: order, value: params[:gal_requisition], by: Current.user)
        return render_failure(result) if result.failure?

        render json: Exams::Json.order(order.reload, view: :administrative, user: Current.user)
      end
    end
  end
  ```
  Em `Exams::BaseController::FAILURE_STATUS`: `gal_already_registered: :conflict`. Em `Exams::Json.order`, no hash inicial: `gal_requisition: order.gal_requisition`. Em `Lab::RequisitionsController`: `before_action :load_requisition, only: %i[show receipt labels deliver]` e

  ```ruby
      def deliver
        result = Lab::Deliver.call(requisition: @requisition, delivered_to: params[:delivered_to], by: Current.user)
        return render_failure(result) if result.failure?

        render json: Lab::Json.requisition(@requisition.reload, viewer: Current.user)
      end
  ```
  Rotas: `post "orders/:id/gal_requisition", to: "gal#create"` no `scope "/exams"`; `post "requisitions/:id/deliver", to: "requisitions#deliver"` no `scope "/lab"`.

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/delivery_and_gal_spec.rb spec/requests/exams` → verde.

- [ ] **Step 5: Commit** e mova o card **F-21.24** para **Done**:

  ```bash
  git add app/commands/lab/deliver.rb app/commands/exams/record_gal.rb app/controllers/exams/gal_controller.rb app/controllers/exams/base_controller.rb app/controllers/lab/requisitions_controller.rb app/services/exams/json.rb config/routes.rb spec/requests/lab/delivery_and_gal_spec.rb
  git commit -m "feat: record report delivery and the GAL requisition number

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 21: Relatórios de produção, rejeição e prazo (F-21.23)

**Files:**
- Create: `app/services/lab/production_report.rb`, `app/controllers/lab/production_reports_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/lab/reports_spec.rb`

**Interfaces:**
- Consumes: `SigtapProcedure#amount_sa_cents` e `Terminology::Sigtap.release_for` (Task 3); `Exams::Report.month_range`, `Exams::Report.safe_cell` (21a).
- Produces: `Lab::ProductionReport.production(provider_ids:, month:) → { rows:, total_quantity:, total_value_cents: }`, `.rejections(provider_ids:, month:) → [Hash]`, `.turnaround(provider_ids:, month:) → [Hash]` (formas do dashboard D13); rotas `GET /lab/reports/{production,rejections,turnaround}` e `.csv` (`provider_id` opcional, `month=AAAA-MM`).

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/reports_spec.rb`:

  ```ruby
  require "rails_helper"
  require "csv"

  # Spec §7 (relatórios) com dashboard D13: produção por unidade de origem,
  # exame, setor e liberador com valor SIGTAP; rejeições por motivo e unidade;
  # prazo cumprido × atrasado × pendente. Tela e CSV. Sem dado de paciente.
  RSpec.describe "Laboratório — relatórios", type: :request do
    before do
      lab_city!
      release = sigtap_release!
      SigtapProcedure.find_by!(release: release, code: "0202010473").update!(amount_sa_cents: 185)
    end

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:glucose) { lab_exam!(provider: provider, delivery_days: 1) }
    let(:month) { Time.zone.today.strftime("%Y-%m") }

    it "produção, rejeições e prazo, com CSV" do
      unit = create_unit("UBS Produção").tap { |u| u.update!(collects_samples: true) }
      2.times.each_with_index do |_, i|
        req = received_requisition!(provider: provider, exams: [ glucose ], unit: unit)
        item = type!(req.items.first, { glicose: "98" })
        travel((i * 31).seconds) { Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead) }
      end
      rejected = open_requisition!(provider: provider, exams: [ glucose ], unit: unit)
      staff = lab_staff!
      Lab::CollectSample.call(sample: rejected.samples.first, by: staff)
      reason = LabRejectionReason.create!(exam_provider: provider, name: "Amostra hemolisada")
      Lab::ReceiveSample.call(barcode: rejected.samples.first.barcode, decision: "reject", reason_id: reason.id, by: staff)

      sign_in_as(lead)
      get "/lab/reports/production", params: { provider_id: provider.id, month: month }
      expect(json_body["rows"].sole).to include("origin_unit" => { "id" => unit.id, "name" => "UBS Produção" }, "sigtap_code" => "0202010473",
                                                "exam_name" => "Glicose", "sector" => "biochemistry", "quantity" => 2, "sigtap_value_cents" => 370)
      expect(json_body["rows"].sole["releaser"]).to include("id" => lead.id)
      expect(json_body).to include("total_quantity" => 2, "total_value_cents" => 370)
      get "/lab/reports/rejections", params: { month: month }
      expect(json_body["rows"].sole).to include("reason" => { "id" => reason.id, "name" => "Amostra hemolisada" }, "count" => 1)
      get "/lab/reports/turnaround", params: { month: month }
      expect(json_body["rows"].sole).to include("sigtap_code" => "0202010473", "on_time" => 2, "late" => 0, "pending" => 1)
      get "/lab/reports/production.csv", params: { month: month }
      csv = CSV.parse(response.body, headers: true)
      expect(csv.headers).to eq(%w[origin_unit sigtap_code exam_name sector releaser quantity sigtap_value_cents])
      expect(csv.first["quantity"]).to eq("2")
      get "/lab/reports/rejections.csv", params: { month: month }
      expect(response.media_type).to eq("text/csv")
      get "/lab/reports/turnaround", params: { month: "2026-13" }
      expect(json_body["error"]).to eq("invalid_month")
      sign_in_as(lab_staff!)
      get "/lab/reports/production", params: { month: month }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/reports_spec.rb`.

- [ ] **Step 3: Implemente.** `app/services/lab/production_report.rb`:

  ```ruby
  # Relatórios do laboratório (spec §7; dashboard D13), mês no fuso da cidade.
  # Produção: itens do PRIMEIRO laudo (retificação não conta de novo) com
  # liberação no mês, por unidade de origem, exame, setor e liberador, com o
  # valor SIGTAP (VL_SA da competência do mês; 0 se a tabela não traz).
  # Rejeições: amostras rejeitadas no mês por motivo e unidade. Prazo: itens
  # liberados no mês no prazo (até expected_delivery_on) ou atrasados, e os
  # ainda abertos das requisições do mês.
  module Lab
    module ProductionReport
      PRODUCTION_HEADERS = %w[origin_unit sigtap_code exam_name sector releaser quantity sigtap_value_cents].freeze
      REJECTION_HEADERS = %w[reason origin_unit count].freeze
      TURNAROUND_HEADERS = %w[sigtap_code exam_name on_time late pending].freeze

      module_function

      def production(provider_ids:, month:)
        items = first_release_items(provider_ids, month)
        amounts = amounts(items.map { |item, _| item.lab_exam_version.lab_exam.sigtap_code }.uniq, month)
        rows = items.group_by { |item, report| [ item.lab_requisition.origin_unit, item.lab_exam_version.lab_exam, report.released_by_user ] }
                    .map do |(unit, exam, releaser), list|
          { origin_unit: { id: unit.id, name: unit.name }, sigtap_code: exam.sigtap_code, exam_name: exam.name, sector: exam.sector,
            releaser: releaser && { id: releaser.id, name: Screenings::Json.staff_name(releaser) }, quantity: list.size,
            sigtap_value_cents: list.size * amounts.fetch(exam.sigtap_code, 0) }
        end.sort_by { |r| [ r[:origin_unit][:name], r[:exam_name] ] }
        { rows: rows, total_quantity: rows.sum { |r| r[:quantity] }, total_value_cents: rows.sum { |r| r[:sigtap_value_cents] } }
      end

      def rejections(provider_ids:, month:)
        LabSample.joins(:lab_requisition).includes(:lab_rejection_reason, lab_requisition: :origin_unit)
                 .where(status: "rejected", received_at: month, lab_requisitions: { exam_provider_id: provider_ids })
                 .group_by { |s| [ s.lab_rejection_reason, s.lab_requisition.origin_unit ] }
                 .map { |(reason, unit), list| { reason: { id: reason.id, name: reason.name }, origin_unit: { id: unit.id, name: unit.name }, count: list.size } }
                 .sort_by { |r| [ -r[:count], r[:reason][:name] ] }
      end

      def turnaround(provider_ids:, month:)
        released = first_release_items(provider_ids, month)
        open_items = LabRequisitionItem.joins(:lab_requisition).includes(lab_exam_version: :lab_exam)
                                       .where(status: %w[pending typed], lab_requisitions: { exam_provider_id: provider_ids, created_at: month })
        exams = (released.map { |i, _| i.lab_exam_version.lab_exam } + open_items.map { |i| i.lab_exam_version.lab_exam }).uniq(&:sigtap_code)
        exams.sort_by(&:name).map do |exam|
          mine = released.select { |i, _| i.lab_exam_version.lab_exam.sigtap_code == exam.sigtap_code }
          on_time = mine.count { |i, r| r.released_at.in_time_zone.to_date <= i.lab_requisition.expected_delivery_on }
          { sigtap_code: exam.sigtap_code, exam_name: exam.name, on_time: on_time, late: mine.size - on_time,
            pending: open_items.count { |i| i.lab_exam_version.lab_exam.sigtap_code == exam.sigtap_code } }
        end
      end

      # [[item, primeiro laudo que o liberou]]
      def first_release_items(provider_ids, month)
        LabReportItem.joins(:lab_report)
                     .where(lab_reports: { exam_provider_id: provider_ids, replaces_report_id: nil, status: %w[released rectified], released_at: month })
                     .includes(lab_report: :released_by_user, lab_requisition_item: [ { lab_requisition: :origin_unit }, { lab_exam_version: :lab_exam } ])
                     .map { |ri| [ ri.lab_requisition_item, ri.lab_report ] }
      end

      def amounts(codes, month)
        release = Terminology::Sigtap.release_for(month.first.strftime("%Y%m"))
        return {} unless release

        SigtapProcedure.where(release_id: release.id, code: codes).pluck(:code, :amount_sa_cents).to_h.transform_values(&:to_i)
      end
      private_class_method :first_release_items, :amounts
    end
  end
  ```
  `app/controllers/lab/production_reports_controller.rb`:

  ```ruby
  # Relatórios (contrato §6; D13): municipal_admin ou lab_releaser; só leitura,
  # sem dado de paciente, sem step-up.
  require "csv"

  module Lab
    class ProductionReportsController < BaseController
      KINDS = %w[production rejections turnaround].freeze

      before_action :scope_providers
      before_action :load_month

      def show
        data = Lab::ProductionReport.public_send(kind, provider_ids: @provider_ids, month: @month)
        render json: kind == "production" ? data : { rows: data }
      end

      def csv
        data = Lab::ProductionReport.public_send(kind, provider_ids: @provider_ids, month: @month)
        safe = ->(v) { Exams::Report.safe_cell(v) }
        body = CSV.generate do |out|
          case kind
          when "production"
            out << Lab::ProductionReport::PRODUCTION_HEADERS
            data[:rows].each { |r| out << [ safe.(r[:origin_unit][:name]), r[:sigtap_code], safe.(r[:exam_name]), r[:sector], safe.(r.dig(:releaser, :name)), r[:quantity], r[:sigtap_value_cents] ] }
          when "rejections"
            out << Lab::ProductionReport::REJECTION_HEADERS
            data.each { |r| out << [ safe.(r[:reason][:name]), safe.(r[:origin_unit][:name]), r[:count] ] }
          else
            out << Lab::ProductionReport::TURNAROUND_HEADERS
            data.each { |r| out << [ r[:sigtap_code], safe.(r[:exam_name]), r[:on_time], r[:late], r[:pending] ] }
          end
        end
        send_data body, type: "text/csv; charset=utf-8", disposition: "attachment", filename: "laboratorio-#{kind}-#{params[:month]}.csv"
      end

      private

      def kind = params[:kind]

      def scope_providers
        ids = Lab::Access.admin?(Current.user) ? Lab::Access.labs.pluck(:id) : Lab::Access.releaser_provider_ids(Current.user)
        return forbid if ids.empty?

        if params[:provider_id].present?
          return forbid unless ids.include?(params[:provider_id])

          ids = [ params[:provider_id] ]
        end
        @provider_ids = ids
      end

      def load_month
        @month = Exams::Report.month_range(params[:month])
        render(json: { error: "invalid_month" }, status: :unprocessable_entity) unless @month
      end
    end
  end
  ```
  Rotas no `scope "/lab"` (as `.csv` antes):

  ```ruby
      get "reports/:kind.csv", to: "production_reports#csv", format: false, constraints: { kind: /production|rejections|turnaround/ }
      get "reports/:kind",     to: "production_reports#show", constraints: { kind: /production|rejections|turnaround/ }
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/reports_spec.rb` → verde.

- [ ] **Step 5: Commit** e mova o card **F-21.23** para **Done**:

  ```bash
  git add app/services/lab/production_report.rb app/controllers/lab/production_reports_controller.rb config/routes.rb spec/requests/lab/reports_spec.rb
  git commit -m "feat: add lab production, rejection and turnaround reports with CSV

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 9 — Invariantes, semente, prova e rollout

### Task 22: Exclusão LGPD e invariantes do ADR 0037

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb`
- Create: `spec/invariants/municipal_lab_invariants_spec.rb`, `spec/commands/citizens/request_erasure_lab_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 5–21.
- Produces: `Citizens::RequestErasure.examined_patients?` conta `LabRequisition` (regra da api#84/A49).

- [ ] **Step 1: Testes que falham** — `spec/commands/citizens/request_erasure_lab_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §8: requisição, amostras, resultados e laudos contam como documento
  # clínico na exclusão (regra única RequestErasure.retained?, api#84).
  RSpec.describe Citizens::RequestErasure do
    before { lab_city! }

    it "quem tem requisição do laboratório fica retido" do
      patient, citizen = exam_patient!(exam_seq!)
      provider = lab_provider!
      req = lab_requisition_row!(provider: provider, version: lab_exam!(provider: provider).current_version)
      req.update_columns(patient_id: patient.id) # o exame do 21a da requisição fica com outro paciente
      expect(ExamOrder.where(patient_id: patient.id)).to be_empty
      expect(described_class.retained?(Citizen.where(id: citizen.id), cpf: citizen.cpf)).to be(true)
    end
  end
  ```
  (O exame do 21a de `lab_requisition_row!` nasce para outro paciente: só a requisição prende este.)

  `spec/invariants/municipal_lab_invariants_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0037, Invariantes, provados de ponta a ponta num fluxo inteiro.
  RSpec.describe "Invariantes do laboratório municipal (ADR 0037)", type: :request do
    before { lab_city!(hl7: true) }

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:req) { received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ]) }
    let(:token) { middleware_token!(provider) }

    def oul(value, control) = "MSH|^~\\&|MW|LAB|ROTA-SAUDE|t|20261015150000||OUL^R22^OUL_R22|#{control}|P|2.5.1\rSPM|1|#{req.samples.first.barcode}\rOBR|1|||GLI\rOBX|1|NM|glicose^G^99RSL||#{value}|mg/dL|||||F\r"

    it "resultado de aparelho nunca liberado sozinho; não liberado não sai; valores fora de eventos e log" do
      log = capture_log { hl7_post(oul("123", "I1"), token: token) }
      item = req.items.first.reload
      expect(item.status).to eq("typed")
      expect(ExamResult.where(exam_order_id: item.exam_order_id)).to be_empty
      expect(LabReport.count).to eq(0)
      report = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead).payload[:report]
      expect(report.status).to eq("released")
      payloads = DomainEvent.where("name LIKE 'lab_%' OR name LIKE 'exam_order.%'").pluck(:payload).to_json
      [ "123", req.patient.cpf, req.mother_name, req.samples.first.barcode ].each { |secret| expect(payloads).not_to include(secret) }
      expect(log).not_to include("123", req.samples.first.barcode)
    end

    it "médico só com ICP; um exame num item só; nenhum middleware simulado em produção" do
      doctor = lab_releaser!(provider, council: "CRM")
      expect(Lab::Release.call(requisition: req, item_ids: [ req.items.first.id ], signature: "advanced", totp: totp_for(doctor), by: doctor).reason)
        .to eq(:icp_required)
      expect(LabRequisitionItem.connection.indexes(:lab_requisition_items).find { |i| i.columns == [ "exam_order_id" ] }.unique).to be(true)
      expect(Platform::Features.find("fake_middleware", env: "production")).to be_nil
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/commands/citizens/request_erasure_lab_spec.rb spec/invariants/municipal_lab_invariants_spec.rb` → o primeiro falha (a regra ainda não olha a requisição).

- [ ] **Step 3: Implemente.** Em `Citizens::RequestErasure`:

  ```ruby
      # ADR 0036 (A49) e ADR 0037: exame é registro clínico — pedido em qualquer
      # situação, resultado externo e requisição do laboratório municipal.
      def examined_patients?(ids)
        ExamOrder.where(patient_id: ids).exists? || ExamResult.where(patient_id: ids).exists? ||
          LabRequisition.where(patient_id: ids).exists?
      end
  ```

- [ ] **Step 4: Rode:** `rspec spec/commands/citizens spec/invariants` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/commands/citizens/request_erasure.rb spec/commands/citizens/request_erasure_lab_spec.rb spec/invariants/municipal_lab_invariants_spec.rb
  git commit -m "feat: retain erasure for people with lab requisitions and pin ADR 0037 invariants

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 23: Semente de dev realista

**Files:**
- Create: `lib/lab_crew.rb`, `spec/lib/lab_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- Consumes: `Lab::Templates::Load` (Task 3), `Lab::AdoptExam`, `Lab::CreateVersion`, `Lab::AssignReleaser`, `Lab::RotateMiddlewareToken`, `Lab::SaveRejectionReason` (Tasks 6–7), `Professionals::Create`, `SignatureCrew.ensure_totp`/`.otpauth_uri`, `DigitalSignatureCrew.free_cpf_for`, `ExamCrew` (o prestador "Laboratório Municipal Central" e as UBS do `ProfessionalCrew`).
- Produces: `LabCrew.seed_platform! → { created:, skipped:, sigtap: }`; `LabCrew.seed_current_city(slug:, password:, admin:) → { switch:, exams:, users: [ { email, role, otpauth_uri } ] }`; constantes `LabCrew::LAB = "Laboratório Municipal Central"`, `INSTRUMENT_CODES`, `DEFAULT_FAKE_MIDDLEWARE_TOKEN`.

- [ ] **Step 1: Teste que falha** — `spec/lib/lab_crew_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/lab_crew").to_s

  # Semente de dev (ADR 0037; dev imita o real): laboratório municipal de
  # Curitiba uses_municipal_lab com endereço e telefone; UBS que coleta e UBS
  # que não; catálogo adotado com códigos de aparelho e críticos; motivos;
  # RT biomédica (CRBM), médico substituto (CRM, CPF válido, só ICP); técnica;
  # token do middleware simulado. Idempotente. Nunca imprime o token.
  RSpec.describe LabCrew do
    before do
      lab_city!(fake_middleware: true)
      Lab::Templates::Load.sigtap_codes.each { |c| SigtapProcedure.find_or_create_by!(release: sigtap_release!, code: c) { |p| p.name = "EXAME #{c}" } }
      Maintainer.find_by(email_address: "dev@local") ||
        Maintainer.create!(email_address: "dev@local", password: "dev-password-123", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
      ExamProvider.create!(name: LabCrew::LAB, kind: "municipal", cnes: "2385123")
      %w[UBS\ Jardim\ das\ Flores UBS\ Vila\ Esperança].each { |name| HealthUnit.find_or_create_by!(name: name) { |u| u.kind = "ubs" } }
    end

    it "monta o laboratório, a equipe e o catálogo; segunda rodada não duplica" do
      described_class.seed_platform!
      result = described_class.seed_current_city(slug: "curitiba", password: "dev-password", admin: lab_admin!)
      lab = ExamProvider.find_by!(name: LabCrew::LAB)
      expect(lab).to have_attributes(uses_municipal_lab: true, phone: "4133501200")
      expect(HealthUnit.find_by!(name: "UBS Jardim das Flores").collects_samples).to be(true)
      expect(HealthUnit.find_by!(name: "UBS Vila Esperança").collects_samples).to be(false)
      glucose = LabExam.find_by!(exam_provider: lab, sigtap_code: "0202010473")
      expect(glucose.current_version).to have_attributes(version: 2, instrument_test_code: "GLI")
      expect(glucose.current_version.analyte("glicose")).to include("critical_high" => 400.0, "entry_max" => 1500.0)
      expect(LabExam.where(exam_provider: lab).count).to eq(16)
      expect(LabRejectionReason.where(exam_provider: lab).count).to eq(6)
      bio = User.find_by!(email_address: "biomedica@curitiba.demo")
      doctor = User.find_by!(email_address: "patologista@curitiba.demo")
      expect(bio.professional.council).to eq("CRBM")
      expect(LabReleaser.active.find_by!(user: bio)).to have_attributes(technical_lead: true)
      expect(LabReleaser.active.find_by!(user: doctor)).to have_attributes(technical_deputy: true)
      expect(CitizenIdentity::Cpf.normalize(doctor.professional.cpf)).to be_present
      expect(User.find_by!(email_address: "tecnica-lab@curitiba.demo").has_role?("lab_staff")).to be(true)
      expect(LabMiddlewareToken.active.find_by!(exam_provider: lab)).to have_attributes(simulated: true,
        token_digest: LabMiddlewareToken.digest(LabCrew::DEFAULT_FAKE_MIDDLEWARE_TOKEN))
      expect(result[:users].map { |u| u[:email] }).to contain_exactly("tecnica-lab@curitiba.demo", "biomedica@curitiba.demo", "patologista@curitiba.demo")
      expect(result.to_s).not_to include(LabCrew::DEFAULT_FAKE_MIDDLEWARE_TOKEN)
      expect { described_class.seed_current_city(slug: "curitiba", password: "dev-password", admin: lab_admin!) }
        .not_to(change { [ LabExam.count, LabExamVersion.count, LabReleaser.count, LabMiddlewareToken.count, LabRejectionReason.count ] })
    end

    it "fora de Curitiba não liga nada" do
      expect(described_class.seed_current_city(slug: "maringa", password: "x", admin: lab_admin!)[:switch]).to start_with("desligado")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/lib/lab_crew_spec.rb` → `LoadError`.

- [ ] **Step 3: Implemente** `lib/lab_crew.rb`:

  ```ruby
  require "tmpdir"
  require Rails.root.join("lib/sigtap_sample").to_s
  require_relative "signature_crew"
  require_relative "digital_signature_crew"

  # Semente de dev do módulo 21b (ADR 0037). Dev é fictício mas imita o real:
  # SIGTAP e LOINC reais; o "Laboratório Municipal Central" (do ExamCrew) vira o
  # laboratório municipal de Curitiba, com endereço e telefone; a UBS Jardim das
  # Flores coleta e a UBS Vila Esperança não; os 16 exames do catálogo-base
  # adotados, com versão 2 (códigos de aparelho, limites de digitação e
  # críticos); seis motivos de rejeição; tecnica-lab@ (lab_staff),
  # biomedica@ (CRBM, RT) e patologista@ (CRM, substituto: só ICP — o vínculo
  # do certificado do PSC simulado é feito no navegador, como no 19b, porque a
  # chave da folha só existe na memória do fake-psc); token do middleware
  # simulado (FAKE_MIDDLEWARE_TOKEN; só o hash vai à cidade, nunca impresso).
  # Só Curitiba (modo record). Idempotente. Roda depois do ExamCrew.
  class LabCrew
    DEV_MAINTAINER = "dev@local".freeze
    LAB = "Laboratório Municipal Central".freeze
    ADDRESS = "Rua Doutor Faivre, 1200 — Centro — Curitiba/PR".freeze
    PHONE = "4133501200".freeze
    COLLECTING_UNIT = "UBS Jardim das Flores".freeze
    OTHER_UNIT = "UBS Vila Esperança".freeze
    SWITCHES = %w[municipal_lab lab_hl7_inbound fake_middleware].freeze
    FAKE_MIDDLEWARE_TOKEN_ENV = "FAKE_MIDDLEWARE_TOKEN".freeze
    DEFAULT_FAKE_MIDDLEWARE_TOKEN = "dev-fake-middleware-token-0123456789abcdef01".freeze
    BENCHES = { "hematology" => "Hematologia 1", "biochemistry" => "Bioquímica 1", "immunology" => "Imunologia 1",
                "urinalysis" => "Uroanálise", "parasitology" => "Parasitologia" }.freeze
    DELIVERY_DAYS = { "hematology" => 1, "biochemistry" => 2, "immunology" => 5, "urinalysis" => 2, "parasitology" => 3 }.freeze
    INSTRUMENT_CODES = { "0202020380" => "HEM", "0202010473" => "GLI", "0202010503" => "A1C", "0202010295" => "COL", "0202010279" => "HDL",
                         "0202010287" => "LDL", "0202010678" => "TRI", "0202010317" => "CRE", "0202010694" => "URE", "0202010643" => "TGO",
                         "0202010651" => "TGP", "0202060250" => "TSH", "0202060381" => "T4L", "0202060217" => "HCG" }.freeze
    # Limites de digitação e críticos de partida (RT ajusta): chave → [entry_min, entry_max, critical_low, critical_high].
    LIMITS = { "glicose" => [ 10, 1500, 40, 400 ], "hemoglobina" => [ 1, 25, 7, 20 ], "plaquetas" => [ 1, 2000, 20, 1000 ],
               "leucocitos" => [ 0.1, 200, 1, 50 ], "creatinina" => [ 0.1, 30, nil, 7.4 ], "tsh" => [ 0.001, 150, nil, nil ] }.freeze
    REASONS = [ "Amostra hemolisada", "Amostra lipêmica", "Amostra coagulada", "Sem identificação", "Volume insuficiente",
                "Recipiente inadequado" ].freeze
    USERS = [
      { prefix: "tecnica-lab", role: "lab_staff", secret_env: "DEV_LAB_STAFF_OTP_SECRET", default_secret: "ORSWG3TJMNQS23DBMIWWIZLWEEQSCIJB" },
      { prefix: "biomedica", role: "lab_releaser", secret_env: "DEV_LAB_BIOMEDICAL_OTP_SECRET", default_secret: "MJUW63LFMRUWGYJNMRSXMIJBEEQSCIJB",
        profile: { professional_name: "Mariana Albuquerque Teles", council: "CRBM" }, lead: true },
      { prefix: "patologista", role: "lab_releaser", secret_env: "DEV_LAB_PATHOLOGIST_OTP_SECRET", default_secret: "OBQXI33MN5TWS43UMEWWIZLWEEQSCIJB",
        profile: { professional_name: "Otávio Lacerda Pires", council: "CRM" }, deputy: true }
    ].freeze

    class << self
      # Catálogo-base na plataforma. Republica o recorte SIGTAP de dev quando
      # ele não tem os códigos do catálogo (SIGTAP real nunca é tocada).
      def seed_platform!
        guard_environment!
        competence = Time.zone.today.strftime("%Y%m")
        release = Terminology::Sigtap.release_for(competence)
        codes = Lab::Templates::Load.sigtap_codes
        have = release ? SigtapProcedure.where(release_id: release.id, code: codes).count : 0
        if have < codes.size && !real_sigtap_active?
          Dir.mktmpdir do |dir|
            imported = Terminology::Import.call(kind: "sigtap", version: competence, path: SigtapSample.write_to(dir, competence: competence), by: "db:seed")
            raise "semente do laboratório: SIGTAP recusada (#{imported.reason})" if imported.failure?
          end
        end
        Lab::Templates::Load.call.merge(sigtap: competence)
      end

      def seed_current_city(slug:, password:, admin:)
        guard_environment!
        return { switch: "desligado (só Curitiba liga)", exams: 0, users: [] } unless slug == "curitiba"

        maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
        return { switch: "desligado (sem mantenedor #{DEV_MAINTAINER})", exams: 0, users: [] } unless maintainer

        city = City.find(Current.city.id)
        SWITCHES.each { |key| Platform::Features.set!(city: city, key: key, enabled: true, maintainer: maintainer) }
        lab = laboratory!
        units!
        exams = catalog!(lab, admin)
        reasons!(lab)
        users = USERS.map { |spec| ensure_user(spec, slug: slug, password: password, admin: admin, lab: lab) }
        token!(lab)
        missing = SWITCHES.flat_map { |key| Platform::Features.missing(city, key) }
        { switch: missing.empty? ? "ligado (#{SWITCHES.join(', ')})" : "ligado, falta: #{missing.join(', ')}", exams: exams, users: users }
      end

      private

      def guard_environment!
        raise "semente do laboratório: ambiente implantado; a semente é só de dev" if Rota.deployed?
      end

      def real_sigtap_active?
        TerminologyRelease.active.where(kind: "sigtap").pluck(:id).any? do |id|
          SigtapProcedure.where(release_id: id).count > SigtapSample::PROCEDURES.size
        end
      end

      def laboratory!
        lab = ExamProvider.find_by!(name: LAB)
        lab.update!(uses_municipal_lab: true, address: ADDRESS, phone: PHONE) unless lab.uses_municipal_lab && lab.phone == PHONE
        lab
      end

      def units!
        HealthUnit.find_by!(name: COLLECTING_UNIT).then { |u| u.update!(collects_samples: true) unless u.collects_samples }
        HealthUnit.find_by!(name: OTHER_UNIT).then { |u| u.update!(collects_samples: false) if u.collects_samples }
      end

      # Adota os 16 e cria a versão 2 com código de aparelho e limites (uma vez).
      def catalog!(lab, admin)
        LabExamTemplate.where(active: true).includes(:analytes).order(:name).count do |template|
          next false if LabExam.exists?(exam_provider_id: lab.id, sigtap_code: template.sigtap_code)

          adopted = Lab::AdoptExam.call(provider: lab, template_id: template.id, sector: template.sector, bench: BENCHES[template.sector],
                                        delivery_days: DELIVERY_DAYS.fetch(template.sector, 3), by: admin)
          raise "semente do laboratório: adoção de #{template.sigtap_code} recusada (#{adopted.reason})" if adopted.failure?

          exam = adopted.payload[:exam]
          v1 = exam.current_version
          analytes = v1.analytes.map do |a|
            min, max, low, high = LIMITS[a["key"]]
            a.merge("entry_min" => min, "entry_max" => max, "critical_low" => low, "critical_high" => high)
          end
          version = Lab::CreateVersion.call(exam: exam, by: admin, params: { "method" => v1.method, "material" => v1.material,
                                                                             "container" => v1.container,
                                                                             "instrument_test_code" => INSTRUMENT_CODES[template.sigtap_code],
                                                                             "analytes" => analytes })
          raise "semente do laboratório: versão de #{template.sigtap_code} recusada (#{version.reason})" if version.failure?

          true
        end
      end

      def reasons!(lab)
        REASONS.each do |name|
          next if LabRejectionReason.where(exam_provider_id: lab.id).where("lower(name) = lower(?)", name).exists?

          Lab::SaveRejectionReason.call({ "provider_id" => lab.id, "name" => name, "active" => true })
        end
      end

      def ensure_user(spec, slug:, password:, admin:, lab:)
        user = User.find_or_initialize_by(email_address: "#{spec[:prefix]}@#{slug}.demo")
        user.password = password if user.new_record?
        user.save!
        Membership.find_or_create_by!(user: user, role: spec[:role], revoked_at: nil) { |m| m.granted_at = Time.current }
        SignatureCrew.ensure_totp(user, secret_env: spec[:secret_env], default_secret: spec[:default_secret])
        if spec[:profile]
          profile!(user, spec, slug, admin)
          unless LabReleaser.active.exists?(user_id: user.id, exam_provider_id: lab.id)
            assigned = Lab::AssignReleaser.call(user_id: user.id, provider: lab, technical_lead: spec.fetch(:lead, false),
                                                technical_deputy: spec.fetch(:deputy, false), by: admin)
            raise "semente do laboratório: liberador #{spec[:prefix]} recusado (#{assigned.reason})" if assigned.failure?
          end
        end
        { email: user.email_address, role: spec[:role], otpauth_uri: SignatureCrew.otpauth_uri(user) }
      end

      def profile!(user, spec, slug, admin)
        return user.professional if user.professional

        seed = "#{slug}:#{spec[:prefix]}"
        attrs = spec[:profile].merge(council_state: CityProfile.current&.uf.presence || "PR", cns: Professionals::Cns.generate(seed),
                                     registration_number: (Zlib.crc32(seed) % 90_000 + 10_000).to_s)
        created = Professionals::Create.call(user_id: user.id, attrs: attrs, by: admin)
        raise "semente do laboratório: perfil de #{spec[:prefix]} recusado (#{created.reason})" if created.failure?

        professional = created.payload[:professional]
        professional.update!(cpf: DigitalSignatureCrew.free_cpf_for(slug, professional))
        professional
      end

      def token!(lab)
        digest = LabMiddlewareToken.digest(ENV.fetch(FAKE_MIDDLEWARE_TOKEN_ENV, DEFAULT_FAKE_MIDDLEWARE_TOKEN))
        return if LabMiddlewareToken.active.exists?(exam_provider_id: lab.id, token_digest: digest)

        Lab::RotateMiddlewareToken.call(provider: lab, by: nil, simulated: true,
                                        token: ENV.fetch(FAKE_MIDDLEWARE_TOKEN_ENV, DEFAULT_FAKE_MIDDLEWARE_TOKEN))
      end
    end
  end
  ```
  Em `db/seeds.rb`: na parte de plataforma, depois do `ExamCrew.seed_platform!`,

  ```ruby
    # Catálogo-base do laboratório (módulo 21b, ADR 0037).
    lab_templates = LabCrew.seed_platform!
    puts "[seeds] catálogo-base laboratório . #{lab_templates[:created]} exames novos; fora da SIGTAP: #{lab_templates[:skipped].join(', ').presence || 'nenhum'}"
  ```
  e, por cidade, depois do bloco do `ExamCrew`:

  ```ruby
        # ── Laboratório municipal (módulo 21b, ADR 0037) ──────────────────────
        # Depois do ExamCrew (o prestador) e do ProfessionalCrew (as UBS).
        lab = LabCrew.seed_current_city(slug: slug, password: password, admin: muni_admin)
        puts "[seeds] laboratório .. #{lab[:switch]}; #{lab[:exams]} exames adotados"
        Array(lab[:users]).each { |u| puts "[seeds]               #{u[:email]} (#{u[:role]}) / #{password} → #{u[:otpauth_uri]}" }
  ```
  e `require Rails.root.join("lib/lab_crew")` junto dos outros `require` de crews no topo (confira o padrão do arquivo).

- [ ] **Step 4: Rode:** `rspec spec/lib/lab_crew_spec.rb spec/lib/exam_crew_spec.rb` → verde. Depois, no dev do worktree: `rails db:seed` e confira a saída (as três contas com `otpauth_uri`, "16 exames adotados", "ligado (municipal_lab, lab_hl7_inbound, fake_middleware)").

- [ ] **Step 5: Commit:**

  ```bash
  git add lib/lab_crew.rb db/seeds.rb spec/lib/lab_crew_spec.rb
  git commit -m "feat: seed the Curitiba municipal lab, catalog, team and simulated middleware token

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 24: Revisão final, suíte completa, api na porta 3042, prova no navegador e runbook de rollout

**Files:**
- Create: `docs/operacao/rollout-laboratorio-municipal.md` (repo `docs`)

- [ ] **Step 1: Varreduras.**

  ```bash
  git grep -n "DomainEvents.publish(" -- app lib | grep -i "lab_\|result_rectified\|gal_registered"
  git grep -n "Platform.audit(" -- app lib | grep -i "lab"
  git grep -n "Current.city =" -- app lib
  git grep -n "Rails.logger" -- app/services/lab app/commands/lab app/controllers/lab app/controllers/hl7 lib/fake_middleware
  ```
  Confira cada chamada **inteira** (multilinha): só ids e os campos do contrato §11 (`source`, `signature_kind`, `partial`, `reason` de escalonamento, `reprint`, `collection_place`, `urgent`); nenhuma atribuição de `Current.city`; nenhum log com valor, código de barras, corpo HL7 ou token.

- [ ] **Step 2: Suíte completa** com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod21b -w /rails/.claude/mod21b api bundle exec rspec
  docker compose start worker
  docker compose exec -T -w /rails/.claude/mod21b api bin/rubocop
  ```
  Expected: verde; tempo por exemplo ~0,10 s com o host calmo (memória "BCrypt e custo em teste": acima de 0,15 s investigue).

- [ ] **Step 3: api do worktree na porta 3042** (container `api-dev`, o serviço `api` do compose; o worktree montado em `/rails/.claude/mod21b`):

  ```bash
  docker compose exec -d -w /rails/.claude/mod21b api bin/rails server -b 0.0.0.0 -p 3042
  docker compose exec -T -w /rails/.claude/mod21b api bin/rails db:migrate:platform
  docker compose exec -T -w /rails/.claude/mod21b api bin/rails platform:triggers
  docker compose exec -T -w /rails/.claude/mod21b api bin/rails city:migrate:all
  docker compose exec -T -w /rails/.claude/mod21b api bin/rails db:seed
  curl -s -H "Host: curitiba.localhost" http://localhost:3030/up >/dev/null; docker compose exec -T api curl -s -o /dev/null -w '%{http_code}\n' -H "Host: curitiba.localhost" http://localhost:3042/up
  ```
  Expected: `200`. O dashboard do worktree (`.worktrees/dashboard-mod21b`, Vite 5194) aponta o proxy para `http://api:3042`.

- [ ] **Step 4: Prova no navegador** (spec §9; dashboard do 21b): `FAKE_MIDDLEWARE_WORKDIR=/rails/.claude/mod21b FAKE_MIDDLEWARE_API_URL=http://api:3042 docker compose --profile fake-middleware up -d fake-middleware` e, em `http://curitiba.localhost:5194`:
  1. enfermeira@ (UBS Jardim das Flores, que coleta) abre requisição de glicose + hemograma de um paciente da semente → comprovante e etiquetas (gere o PDF das etiquetas e leia o código como na Task 0, Step 4);
  2. coleta pela etiqueta, monta a remessa (4 °C) e envia;
  3. tecnica-lab@ confere a remessa (uma lida), recebe uma amostra e rejeita a outra ("Amostra hemolisada") → recoleta aparece para a UBS; coleta da recoleta, nova remessa, recebe;
  4. mapa de trabalho (tela e PDF);
  5. o `fake-middleware` manda a glicose (450 crítico e a repetição 98) — log `docker compose logs fake-middleware` com `AA`; digitação manual do hemograma;
  6. biomedica@ escolhe a repetição 450, registra o aviso do crítico (telefone) e libera com assinatura avançada (TOTP); patologista@ vincula o certificado no PSC simulado (painel "Assinatura" do 19b), abre a sessão e libera outra requisição com ICP → laudo pendente nas Pendentes → assinado;
  7. no 21a, o exame mostra o laudo (profissional); no wpda, o cidadão vê conforme a política;
  8. `/v` com o código do laudo (sem resultado);
  9. retificação do laudo da glicose (motivo + contato) → profissional vê o substituído; ciência volta;
  10. entrega; relatório de produção do mês com valor SIGTAP e o CSV.
  Anote cada passo (OK/falha). Falha de api → corrija com teste antes; achado só visual → para o ciclo de interface (memória "Interface em ciclo próprio").

- [ ] **Step 5: Runbook** `docs/operacao/rollout-laboratorio-municipal.md` (repo docs):

  ```markdown
  # Rollout — laboratório municipal (módulo 21b, ADR 0037)

  ## Pré-requisitos
  - Tags do contracts publicadas: `lab-v1.0.0` e `exams-v1.1.0` (cópias em `apps/api/config/lab/lab-report-v1.json`
    e `spec/fixtures/exams/exam-order-v1.json`, SHA-256 preso).
  - 21a em produção; `exam_tracking` utilizável na cidade.
  - Imagem do api com a gem `barby` 0.7.0 (rebuild da imagem: `bundle install` no build).

  ## Ordem de deploy
  1. contracts (tags) → 2. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego
     (`db:migrate` da plataforma — 20261015900001 —, `platform:triggers`, `city:migrate:all` — 20261015900002,
     irreversível); depois `bin/rails lab:templates:load` (catálogo-base; idempotente).
  3. dashboard → 4. maintenance → 5. catálogo público. O wpda não muda.
  Nunca migrar cidade fora do rake. A migração de cidade troca checks e um índice de `exam_results`
  em tabelas existentes: rode fora do pico.

  ## Ligar numa cidade (maintenance)
  1. `municipal_lab` (exige `exam_tracking`).
  2. Admin da cidade: marcar o prestador `uses_municipal_lab` (com CNES, endereço e telefone), as UBS que
     coletam (`collects_samples`), adotar exames e ajustar versões (método, limites, críticos,
     código do aparelho), motivos de rejeição, liberadores (RT e substituto, com perfil e conselho).
  3. `lab_hl7_inbound` só depois do middleware real homologado; gerar o token (aparece uma vez) e
     entregá-lo ao fornecedor por canal seguro.
  4. `fake_middleware` NUNCA em produção (não existe lá); em staging, só com dados de teste.

  ## Gates de go-live (spec §12)
  - Primeiro middleware real: mapeamento de códigos de aparelho e componentes; teste de troca de
    paciente (amostra de outro laboratório, rejeitada, versão antiga) com o fornecedor.
  - Validação do sistema pelo laboratório (dossiê do art. 106 da RDC 978).
  - Certificados ICP dos liberadores médicos; aceitação da assinatura avançada pela VISA local.
  - Impressoras térmicas (203 dpi, etiqueta 50 x 25 mm) e leitores nas UBS e no laboratório: imprimir e
    ler uma etiqueta real de cada modelo.
  - Valor SIGTAP: conferir o `VL_SA` da competência importada (Task 3, Step 1).
  - Retificação de laudo de prestador externo continua em rotasaude/api#85.

  ## Verificação depois de ligar
  - `GET /lab/context` com uma conta do laboratório; abrir uma requisição de teste com paciente de teste,
    imprimir e ler a etiqueta; `POST /hl7/v2` com `QBP^Q11` do fornecedor → `RSP^K11`.
  - Conferir que nenhum valor aparece em log (`grep` do código de barras de teste no log do dia).

  ## Desligar
  - `municipal_lab` desligado: rotas /lab respondem 403; nada é apagado; laudos liberados seguem no 21a e na
    conferência pública. Laudo ICP pendente com `digital_signature` desligado sai (withdrawn) e os itens voltam à fila.
  - Migrações irreversíveis: não há rollback de schema; volte só o código (expand/contract).
  ```
  Commit no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add operacao/rollout-laboratorio-municipal.md
  /opt/homebrew/bin/git -C docs commit -m "docs: add municipal lab rollout runbook (module 21b)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 6: Fechamento.** Pare o servidor da 3042 e o `fake-middleware` (`docker compose --profile fake-middleware stop fake-middleware`); apague os bancos de teste `_mod21b` (conferindo conexões e sessões ativas — memória "Apagar bancos com sufixo de módulo"); confira os cards F-21.14..F-21.24 em **Done**. Merge, push, tags e o comentário em api#85 **só com autorização do usuário**, uma etapa de cada vez (memória "Sessões paralelas no push": `origin/main..main` antes de push).

---

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §3 interruptores → Task 2; §4 plataforma → Tasks 3–4; §4 cidade → Task 5; perfil e papéis → Tasks 5 e 7; §5.1 requisição, papel, comprovante, etiquetas, reimpressão → Tasks 8–9; §5.2 coleta → Task 9; §5.3 remessa → Task 10; §5.4 recebimento, rejeição, recoleta, restrição → Task 10; §5.5 mapa (tela e PDF) → Task 11; §5.6 digitação, referência por sexo e idade na coleta, limites, calculados, HL7 "aguardando conferência", repetição e escolha → Tasks 11 e 13; §5.7 liberação parcial/total, crítico, ICP/avançada, médico só ICP, PDF do art. 167, QR → Tasks 15–18; §5.8 volta ao 21a → Task 17; §5.9 retificação (21b e 21a) → Task 19; §5.10 entrega e §5.11 situação → Tasks 8 e 20; GAL → Task 20; §6 HL7 e fake-middleware → Tasks 12–14; §7 telas (formas para o dashboard) e relatórios → Tasks 6–21; §8 segurança/LGPD → Global Constraints, Tasks 5, 11, 13, 22; §9 testes → cada task; §10 provas técnicas → Task 0; entrega do contracts → Task 1; F-21.14..F-21.24 → mapa do "Ambiente de execução".
- **Placeholders:** só `SCHEMA_SHA256 = "<sha256 do Step 1>"` (Task 15), que é um valor a medir no próprio passo, não decisão pendente.
- **Tipos e nomes:** `Lab::Access.{labs,staff?,releaser_link,provider_ids,member?,collecting_unit_ids,council,admin?}`, `Lab::Effective.rows`, `Lab::EnterResults.{call,recalculate!,context,OPEN}`, `Lab::Release.call(rectifies:)`, `Lab::FinalizeReport.call(report:, pdf_bytes:)`, `Exams::RectifyResult.call(...)`, `Lab::Json.report_summary(report, viewer:)`, helpers `lab_city!`, `lab_provider!`, `lab_exam!`, `open_requisition!`, `received_requisition!`, `type!`, `middleware_token!`, `hl7_post`, `totp_for` são os mesmos em todas as tasks.
- **Review Focus:** 1 → Task 13 ("troca de amostra"); 2 → Task 17 (`release_concurrency_spec`); 3 → Task 11 (`reference_spec`); 4 → Task 19 ("o que cada um vê depois da retificação"); 5 → Task 9 (`labels_pdf_spec`).
- **SDL do catálogo-base:** a Task 4 produz exatamente o SDL da Divergência M1 do plano do maintenance (os mesmos tipos, nulidades e argumentos), com um teste que compara as linhas de `upsertLabExamTemplate`, `upsertLabAnalyteTemplate` e `labExamTemplates` e um passo que compara o bloco inteiro.

## Divergências propostas ao contrato

Numeradas como **P** (do plano do api) para não colidir com as D do dashboard e as M do maintenance.

1. **P1 — ERR da HL7 (D1 do perfil).** O §7 põe o nosso código em `ERR-3`; na 2.5.1 o `ERR-3` é a tabela 0357 (obrigatória). Proposta: `ERR-3` = 0357 (204/207/102/100/200/203), `ERR-4` `E`, `ERR-5` = nosso código no sistema `99RSL` (Task 0 confere).
2. **P2 — `OBX-3` (D2 do perfil).** O §7 diz `instrument_test_code^analyte_key` (a chave cairia no texto do CE). Proposta: `OBR-4.1` = `instrument_test_code`, `OBX-3` = `analyte_key^<nome>^99RSL`.
3. **P3 — Mensagem bruta e idempotência.** Bruto uma vez por mensagem aceita em `lab_hl7_receipts.raw` (cifrado), referenciado por `lab_results.lab_hl7_receipt_id` (não uma cópia por linha); só mensagem aceita grava recibo — `MSH-10` repetido de mensagem recusada é reavaliado. Mensagem inteira ou nada. Token ausente ou desconhecido → HTTP 401; revogado → `AR` (`token_revoked`); `lab_hl7_inbound` desligado → HTTP 403; token simulado com `fake_middleware` desligado → `AR` (`feature_disabled`). Corpo até 1 MB (413); tipo errado 415. Recusas novas `invalid_value` e as de mensagem (`invalid_message`, `unsupported_message_type`, `unsupported_version`).
4. **P4 — `RSP^K11` sem `PID`** (só `SPM`, `ORC`, `OBR`): o middleware não recebe identificação do paciente; `QAK-2` `NF` quando a amostra não tem exame pendente com código de aparelho. `OML^O33` fora do 21b (`AR`, 200).
5. **P5 — Conselho do liberador (concorda com D18).** Reusa `council`, `council_state` e `registration_number` do módulo 10; na API sai como `council`, `council_uf`, `council_number`; `council_missing` também quando o conselho não é CRF/CRBM/CRM; `Professionals::Create` aceita `lab_releaser`.
6. **P6 — Quem atua.** `lab_staff` vale em todo laboratório `uses_municipal_lab` ativo da cidade; o liberador, no laboratório da ligação. Abrir requisição: `lab` só `lab_staff`; `ubs` só profissional com vínculo na unidade de origem que coleta (`unit_does_not_collect`, `missing_role`, `invalid_collection_place`).
7. **P7 — Recoleta no 21a.** `Exams::Transition` aceita `scheduled` a partir de `collected` (motivo `recollection`); as rotas da unidade e do portal continuam recusando (409). Pedido em papel de exame que exige autorização → `order_not_eligible`.
8. **P8 — Cancelar exame em requisição.** Item pendente vira `cancelled`; item digitado ou liberado → 409 `invalid_transition`. A unidade também não registra resultado de laboratório municipal (`provider_uses_municipal_lab`, além do portal e do FHIR).
9. **P9 — Retificação no 21a sem UPDATE.** `exam_results.replaces_result_id` (único); o vigente é o não substituído; "um original por pedido" vira índice parcial; evento da trilha `result_rectified` (`resulted|acknowledged → resulted`). `superseded_at` = criação do resultado que substituiu (concorda com D16).
10. **P10 — Laudo.** `lab_report_items` (itens de cada laudo); PDF em `exam_files` (`pdf_file_id`) e uma cópia por `exam_result`; `content` (JSON congelado, cifrado) + `content_sha256`; `signature_request_id` não é coluna (sai do `SignatureRequest`). Situação extra `withdrawn` (o 19b devolve o pedido ICP quando `digital_signature` é desligado: os itens voltam à fila). `lab-report-v1` ganha `key` no componente e `support_laboratory` (sempre `null` no 21b; arts. 129/169). Registro do paciente no laudo = número da requisição.
11. **P11 — Liberação: erros novos.** `invalid_signature`, `laboratory_incomplete` (sem CNES, endereço ou telefone do laboratório), `technical_lead_missing`, `signature_unavailable` (ICP sem `digital_signature` utilizável — cidade integrada não tem ICP; médico não libera lá), `mfa_required` também para TOTP repetido (consumido). Itens não `typed`, de outro laudo pendente ou fora do laudo retificado → 409 `invalid_transition`.
12. **P12 — Endereço e telefone do laboratório** (art. 167, II): `exam_providers.address` e `.phone` no `PATCH /exams/admin/providers/:id`; `uses_municipal_lab` só com `municipal_lab` utilizável (`municipal_lab_disabled`, concorda com D20).
13. **P13 — Valor SIGTAP.** Coluna de plataforma `sigtap_procedures.amount_sa_cents` (VL_SA); 0 no relatório quando a tabela importada não traz.
14. **P14 — Catálogo do laboratório.** `lab_exams.name` (copiado do catálogo-base); adotar cria a v1 copiada (concorda com D12); versão nova com `invalid_version` (método/material/recipiente/código inválidos) além dos quatro do §3; `PATCH /lab/admin/exams/:id` só setor, bancada, prazo e situação (D12). Fórmula só nas versões da cidade (não no catálogo-base, que segue o SDL M1).
15. **P15 — Número e código de barras.** Requisição 8 dígitos (sequência da cidade); amostra = número + 2 dígitos (até 99 por requisição; recoleta pega o próximo).
16. **P16 — Remessa (concorda com D4) e recebimento.** Erros da criação com `barcode`: `wrong_origin_unit`, `sample_in_other_shipment`, `sample_not_collected`, mais `shipment_empty`, `invalid_temperature`, `unit_does_not_collect`; conferência com código fora da remessa → `invalid_samples`. Amostra `not_received` pode ser recebida depois (chegou atrasada). `restriction_note_required` (concorda com D6).
17. **P17 — Digitação por etiqueta** exige `sigtap_code` junto do `barcode` (a amostra pode ter vários exames). `POST /lab/results` aceita item `released` (concorda com D7); escolha da repetição só pelo liberador; o valor que vale é a última linha.
18. **P18 — Eventos a mais (só ids):** `lab_sample.label_printed` (`reprint`), `lab_requisition.delivered`, `lab_report.withdrawn`, `lab_exam.version_created`, `lab_releaser.assigned`/`.ended`, `lab_middleware.token_rotated`/`.token_revoked`, `exam_order.gal_registered`; de plataforma `lab.template_changed` e `maintenance.lab_template.changed` (R18).
19. **P19 — Conferência pública do laudo.** `/v` devolve `kind: "lab_report"`, `status` `valid|rectified|awaiting_signature` (o retirado não existe para fora), `mode` `digital|advanced`, `pdf_sha256`; sem PDF público (nem o ICP).
20. **P20 — GAL (concorda com D17)** com o formato `[0-9A-Za-z-]{1,30}` e 409 `gal_already_registered`; também `lab_staff` registra.
21. **P21 — Rotas e formas do dashboard adotadas sem mudança:** D1, D2, D3, D5, D8, D9, D10, D11, D12, D13 (o `sigtap_value_cents` da linha é quantidade × valor unitário), D14, D15 (`unit_does_not_collect`), D19 (as pendências são `GET /signature/requests`, a rota real do 19b; voltar ao papel → 422 `not_applicable`), D20.
