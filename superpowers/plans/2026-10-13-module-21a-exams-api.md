# Módulo 21a — Rastreamento do exame (api, contracts e fake-lab) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 20 e a rodada especial do 19 em `origin/main` do api** (conferido ao escrever este plano em `4ffe7c6`: `Platform::Features::CATALOG` com 10 chaves e `SIMULATION_ENVS`, `ClinicalRecord::{Access,Trail,Gate}`, `Timeline::{KINDS,Query,Person,Entry,Json,Trail}`, `Timeline::Sources::Window`, `PecExternal::{Authorize,ViewSession,Page}`, `PecTimelinesController`, `Patients::Resolve`, `ClinicalTerms::SigtapExams`, `Consultations::{Finalize,ItemsInput,Authorization}`, `Consultations::AddAddendum.apply!`, `ClinicalDocuments::Codes`, `EachCityJob`, `SignatureCrew.ensure_totp`, `TimelineCrew`, `SigtapSample`, a última migração de cidade `20261011800002_add_patient_timeline` e `VALID_RANGE = (1..35)`). **A Task 0 confere a base e faz as duas provas técnicas: se um nome mudou, ou se uma prova falhar, PARE e reporte.**

**Goal:** O lado api do módulo 21a (F-21.1 a F-21.13, ADR 0036): interruptores `exam_tracking`, `exam_fhir_inbound`, `exam_pec_bridge` e `fake_lab`; catálogo de exames e prestadores da cidade; o exame com ciclo de vida e trilha só de acréscimo (consulta, guia da recepção, PEC); regulação simples; prazos, atrasos e escalonamento diário; portal do prestador; resultado pela unidade, pelo prestador e pela entrada FHIR; ciência e fila da equipe; resultado externo; "Meus exames" do cidadão; relatório de conferência; fornecedor `Timeline::Sources::ExamSource` (fecha api#71 para exames); o botão no PEC (se a prova técnica passar); os dois esquemas do repo `contracts`; o laboratório simulado (`fake-lab`) no compose de dev; semente realista; rollout.

**Architecture:** Tabelas da cidade (ADR 0020): `exam_orders` guarda o exame e um **cache** da situação (`status`, `stage`, `stage_due_on`, `escalated_at`); `exam_order_events` é a trilha só de acréscimo, e um trigger impede que o cache mude sem um evento da mesma transação (`txid`). Um único comando, `Exams::Transition`, valida a transição pela tabela `FLOW`, grava o evento e atualiza o cache com o prazo da próxima etapa (`Exams::Deadlines`: prestador × exame → catálogo → plataforma; urgente = metade arredondada para cima). Arquivos (guia e laudo) ficam em `exam_files`, cifrados com a chave da cidade (Base64 em `text` + `encrypts`, como `Signature#signed_pdf`), e saem por URL curta assinada (5 min) amarrada ao usuário. A leitura segue as regras do 19a (`ClinicalRecord::Access`) mais duas próprias do exame (autoria do solicitante e fila da equipe); a unidade e o regulador veem uma forma administrativa, sem valores. O FHIR fica na borda: `POST /fhir/DiagnosticReport` valida contra a cópia do esquema do `contracts` (json_schemer) e chama o mesmo `Exams::RegisterResult` do portal e da unidade. O botão no PEC reaproveita a sessão curta do módulo 20 (token no formulário, sem cookie). Migração de cidade `20261013900001`; nenhuma migração de plataforma (o catálogo de interruptores mora no código).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, `json_schemer` (já no Gemfile), `ActiveSupport::MessageVerifier`, `Prawn` (já no Gemfile, só para PDFs de dev), `Net::HTTP` (fake-lab). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-13-module-21a-exam-tracking-design.md` e `docs/adr/0036.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-13-module-21a-exams-contracts.md`; toda diferença está em "Divergências propostas ao contrato" no fim deste plano. Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-3-exames.md` e `docs/pesquisa/2026-10-11-pec-sistemas-externos.md`.

## Desvios e precisões

1. **Interruptor `exam_tracking` não pode exigir `clinical_record`** (Divergência D1). Com `requires: ["feature:clinical_record"]` (modo `record`) e `exam_pec_bridge` exigindo também `pec_external_view` (modo `integrated`), o botão no PEC **nunca** fica utilizável, e a guia da recepção nunca chega às cidades integradas — o contrário da decisão 7 da spec. Este plano usa `exam_tracking` com `requires: ["record_mode"]` (record **ou** integrated; missing `record_mode_off`). O gancho da consulta só age onde há consulta (modo record). **Antes da Task 2, confirme D1 com o coordenador**; se ele mantiver `feature:clinical_record`, troque a linha do catálogo, retire a Task 20 (F-21.11) e registre que a guia só existe no modo record.
2. **`fake_lab` só fora de produção** usa o mecanismo existente `environments: SIMULATION_ENVS` (development, test, staging), igual ao `sncr_mock` — o contrato fala em `non_production: true`, que não existe no código. Em produção a chave nem aparece no catálogo e `Platform::Features.set!` levanta `UnknownFeature`.
3. **Paciente da guia e do PEC:** `exam_orders.patient_id` é sempre um `Patient` (19a). A guia e o PEC recebem um **par validado** e chamam `Patients::Resolve.call(citizen)` (acha ou cria o paciente pelo CPF, com trava por CPF). Par só declarado → 409 `citizen_not_verified`. Nas cidades integradas isso cria `patients` sem consulta — o `Timeline::Person.for_cpf` já trata.
4. **Escrita por cookie só aceita JSON** (guarda `require_json_for_cookie_writes`). O upload de guia e laudo é `POST /exams/uploads` com `application/json` e o arquivo em Base64 (Divergência D3), não multipart.
5. **Arquivos** (`exam_files`): uma tabela para guia e laudo; o resultado e o pedido apontam para ela (`pdf_file_id`, `guide_file_id`) em vez de copiar bytes para colunas `pdf`/`guide_attachment` (a spec nomeia colunas; o contrato não). Arquivo não usado em 24 h é apagado pelo job diário.
6. **Cache da situação × trilha:** o trigger `exam_orders_guard` recusa qualquer UPDATE de `exam_orders` sem uma linha de `exam_order_events` do mesmo pedido gravada na **mesma transação** (`txid_current()`), salvo a re-cifra. Identidade (paciente, exame, origem, solicitante, unidade, código curto) nunca muda.
7. **Prazo gravado na transição:** ao abrir uma etapa, `Exams::Transition` grava `stage`, `stage_opened_at` e `stage_due_on` (resolvidos naquele momento); mudar o catálogo depois não muda prazo já aberto. Cada evento guarda `closed_stage_due_on` (o prazo da etapa que ele fechou) — é com ele que o relatório diz "no prazo/em atraso" sem recalcular.
8. **Exame da consulta fora do catálogo da cidade** vira `exam_order` mesmo assim, com os padrões (sem autorização, prazos da plataforma, política `after_acknowledgement`): a finalização nunca falha por causa do catálogo. Guia e PEC exigem item ativo (422 `sigtap_not_in_catalog`).
9. **Quem é a "unidade":** `citizen_verifier` (recepção, que já atua em qualquer unidade no balcão) ou `health_professional` com vínculo ativo na unidade do exame. **Quem é a "equipe de referência":** não existe ligação paciente→equipe no código (o território do módulo 11 dá unidades por bairro, não equipes). Equipe = membros ativos (`health_team_members.ended_on IS NULL`) de equipes ativas da **unidade do exame**, com CBO de médico (regra `Ledi::ConsultationMapping.cid10_allowed?`) ou de enfermeiro (`2235xx`) (Divergência D10).
10. **Duas visões do exame** (Divergência D5): `clinical` (solicitante, leitura do 19a, equipe na fila) com `result` completo; `administrative` (unidade, regulador) com `result` reduzido a `{ id, origin, source, released_on, laboratory_name }` — sem valores, sem `critical`, sem PDF. Prestador só pelo portal (`<provider_order>`).
11. **Transições** (Divergência D12): `schedule` vale de `awaiting_schedule` e de `scheduled` (reagendar); `result` só de `scheduled` ou `collected` (a unidade agenda e registra coleta antes, mesmo no mesmo minuto); `cancel` antes do resultado; `escalated` não muda `status`.
12. **Prestador simulado** (Divergência D13): `exam_providers.simulated` (só a semente grava `true`; a API admin nunca). Requisição FHIR de prestador simulado exige `fake_lab` utilizável (403 `feature_disabled` com `feature: "fake_lab"`) — em produção, nunca.
13. **FHIR R4 real** (Divergência D7, a confirmar na Task 0b): `DiagnosticReport.code` é 1..1 no R4 e entra no perfil; `contained` só vale referenciado (`result[].reference = "#<id>"`, invariante dom-3); `Observation.status` e `.code` são 1..1; os sistemas de CPF e CNS da RNDS são `http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf` e `.../cns` (`uniqueId` lidos em 2026-10-13 nas páginas `NamingSystem-cpf.json.html` e `NamingSystem-cns.json.html` de `rnds-fhir.saude.gov.br`), não `https://saude.gov.br/...`.
14. **Botão no PEC sem chamada assinada nova** (Divergência D8): cada sistema externo cadastrado no PEC recebe a **sua** chave, e a cidade só guarda uma (`pec_settings`). O botão "Solicitar exames" fica na página `/pec/timeline` (quando `exam_pec_bridge` é utilizável) e envia o token da sessão curta do módulo 20 num formulário: `POST /pec/exams/form` devolve o formulário; `POST /pec/exams` cria o pedido. Sem cookie (iframe de outro site — módulo 20, D3); o token cifrado faz as vezes de CSRF. Só existe se a Task 0a passar.
15. **Rotas do cidadão levam `citizen_id`** (como as do módulo 20, D2). O resultado externo aparece em "Meus exames" (Divergência D9).
16. **Papéis novos privilegiados:** `exam_regulator` e `exam_provider` entram em `Membership::PRIVILEGED_ROLES` (step-up para conceder/convidar; o mantenedor não concede) — veem nome do paciente com o exame (Divergência D11).

## Valores fixados por este plano

1. **Interruptores:** `exam_tracking` (`requires: ["record_mode"]`), `exam_fhir_inbound` (`["feature:exam_tracking"]`), `exam_pec_bridge` (`["feature:exam_tracking", "feature:pec_external_view"]`), `fake_lab` (`["feature:exam_fhir_inbound"]`, `environments: SIMULATION_ENVS`). Ordem no `CATALOG`: depois de `pec_external_view`.
2. **Códigos** exatamente como o contrato §1: `kind` do prestador `municipal|accredited|consortium|lacen|imaging`; `release_policy` `immediate|after_acknowledgement|never_citizen` (padrão `after_acknowledgement`); `priority` `routine|urgent`; `origin` `consultation|reception_guide|pec`; eventos `requested|authorized|denied|scheduled|collected|resulted|acknowledged|escalated|cancelled`; escalonamento `overdue|critical|unlinked_requester`; resultado `origin` `order|external`, `source` `unit|provider|fhir|consultation`; `status` `awaiting_authorization|denied|awaiting_schedule|scheduled|collected|resulted|acknowledged|cancelled`; `stage` `authorize|schedule|collect|result|acknowledge`; padrão da plataforma 15/15/30/30/15 dias.
3. **Código curto:** `ClinicalDocuments::Codes.short_code` (10 caracteres de `23456789ABCDEFGHJKMNPQRSTUVWXYZ`), índice único.
4. **Arquivos:** `application/pdf`, `image/jpeg`, `image/png`; 1 byte a 10 MB (10 485 760); conferência pelos bytes mágicos (`%PDF-`, `\xFF\xD8\xFF`, `\x89PNG\r\n\x1A\n`); `pdf_url`/`guide_url` = `/exams/files/<token>`, `MessageVerifier` propósito `exam_file`, 5 min, amarrado a `{ city_id, file_id, user_id }`.
5. **Token FHIR:** `SecureRandom.urlsafe_base64(32)`; guardado só `SHA256(token)` hex em `exam_providers.fhir_token_digest` (único); rate limit 60/min por digest; corpo até 15 MB (15 728 640); `Content-Type: application/fhir+json` obrigatório.
6. **Interpretação crítica:** `HH`, `LL`, `AA` de `http://terminology.hl7.org/CodeSystem/v3-ObservationInterpretation` (o código basta; sistema não conferido).
7. **Job diário:** `Exams::DailyJob` (`EachCityJob`), fila `housekeeping`, `"every day at 6:30am America/Sao_Paulo"`.
8. **Dev:** api do worktree na porta **3041**; Vite do dashboard **5192**, wpda **5193**; `FAKE_LAB_TOKEN` padrão `dev-fake-lab-token-0123456789abcdef0123` (só dev, override por env); prestador simulado "Laboratório Simulado"; contas `laboratorio@<slug>.demo` (exam_provider) e `regulacao@<slug>.demo` (exam_regulator).

## Global Constraints

- Banco de cidade: `db/city_migrate/20261013900001_add_exam_tracking.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_13_900001)`; paridade em `spec/services/city_schema_spec.rb`); triggers novos em `db/city_triggers.sql` (o dump não representa trigger). Irreversível. Rollout: publicar a imagem e rodar `city:migrate:all` dela **antes** de cortar tráfego; **nunca** migrar cidade fora do rake. Nenhuma migração de plataforma.
- Erros `{ "error": "<reason>" }`; interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (inclusive DELETE).
- **Valor de resultado, CPF, CNS, nome do paciente junto com o exame, PDF e token FHIR nunca em log, evento, Analytics, mensagem de erro ou `inspect`.** `Authorization` não é logado pelo Rails; o corpo FHIR não entra em `params` (só `application/fhir+json` é aceito e esse tipo não é parseado). Novos filtros: `/\A(data|values|requester|requester_name|requester_cns|scheduled_place|external_reference)\z/`.
- Eventos só com ids, declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`: `exam_order.{requested,authorized,denied,scheduled,collected,resulted,acknowledged,escalated,cancelled}`, `exam_result.external_recorded`, `exam_provider.fhir_token_rotated`, `exam_provider.fhir_token_revoked`; `clinical_record.viewed` ganha `access` `exam_result`/`team`/`author` com `exam_order_id`/`exam_result_id`. `DomainEvents.publish(` e `Platform.audit(` são multilinha: confira cada chamada inteira. **Nenhum evento de plataforma novo** (a troca de interruptor já audita `city.feature_changed`); se surgir um, ele entra em `R18_PLATFORM_EVENT_NAMES` — declare, nunca afrouxe.
- Cifra por cidade (não determinística) em `CityEncryption::CITY_KEYED_TARGETS`: `ExamOrder#requester_name`, `#requester_cns`, `ExamOrderEvent#reason`, `ExamResult#values`, `ExamFile#data` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (só lido); o job é `prepend EachCityJob`.
- Specs de request com `type: :request` explícito; helpers novos em `spec/support/exam_helpers.rb` com `RSpec.configure { |c| c.include ExamHelpers }` (o `rails_helper` carrega `spec/support/**/*.rb` por glob — confira a linha do glob antes; se não houver, faça como `timeline_helpers.rb`). `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..36)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (**nunca** `-A`). Merge, push, tag (`exams-v1.0.0` inclusive) e mudança de card só com autorização explícita do usuário — exceto mover os cards F-21.x para **In Progress** no início e para **Done** ao fim de cada um, que o brief manda.

## Ambiente de execução

- **Primeira ação:** mover os cards F-21.1..F-21.13 do board #1 para **In Progress** (comandos `gh project item-edit` começando com `gh`, um por vez — ver memória "Board: gh em comando simples"). Ao terminar a task dona de cada F-ID, mover o card para **Done**.
- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard, do wpda e do maintenance do módulo 21a. Ordem de merge: **contracts** (tag) → **api** → dashboard e wpda → maintenance.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod21 -b feat/mod-21a-exams origin/main
  cp apps/api/config/master.key apps/api/.claude/mod21/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod21`. **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod21 -w /rails/.claude/mod21 api bundle exec rspec <arquivos>
  ```

  e `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod21 <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod21`), criados no começo e recriados depois da migração (Task 3):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod21" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod21"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod21 -w /rails/.claude/mod21 api bin/rails city:test_databases
  ```

- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.
- O repo `contracts` (`rota-saude/contracts`) é um clone: edite e commite ali (branch `feat/mod-21a-exams`); o `docker-compose.yml` da raiz do monorepo **não** está em git (a Task 15 edita e não commita).

### Arquivos em comum com outros módulos (rebase)

Se a main andar, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` (`define(version:)` com o maior número; se outra branch usou `20261013900001`, renumere para o próximo livre e ajuste o `define`), `db/city_triggers.sql`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`, `app/services/city_encryption.rb`, `app/models/membership.rb`, `spec/adr_pointers_spec.rb` (fica `1..36`), `config/routes.rb`, `config/recurring.yml`, `app/services/platform/features.rb`, as três specs com a lista fixa do catálogo, `app/services/timeline.rb`, `app/services/timeline/query.rb`, `lib/sigtap_sample.rb`, `spec/services/terminology/sigtap_reader_spec.rb`, `db/seeds.rb`.

## Review Focus

1. **Laudo no paciente errado pela entrada FHIR** (o laboratório manda o `short_code` certo com o CPF de outra pessoa; manda CNS de um par da pessoa certa; manda o `short_code` de uma guia de outro prestador; reenvia o mesmo `identifier` com outro PDF): só o CPF/CNS da pessoa do exame passa; guia de outro prestador é 404 indistinguível de inexistente; o reenvio devolve 200 com o mesmo `result_id` e não troca o PDF. Teste: Task 14 ("paciente e prestador errados").
2. **Corrida no registro do resultado** (o portal do prestador e a entrada FHIR registram o mesmo exame ao mesmo tempo; a unidade cancela enquanto o laboratório envia): um só resultado, o outro recebe 409 `invalid_transition` ou 422 `result_already_registered`; nunca exame cancelado com resultado. Teste: Task 13 ("dois resultados ao mesmo tempo").
3. **Política de liberação contornada pelo cidadão** (pedir `/citizen/exams/:id/pdf` de um anti-HIV `never_citizen`, de um `after_acknowledgement` ainda sem ciência, de um exame de outro telefone, do resultado externo de um par declarado): 404 `not_found` em todos, nada vaza (nem `critical`, nem motivo de negação). Teste: Task 17 ("política contornada").
4. **Ciência por quem não pode** (médico de outra unidade tenta a ciência da fila da equipe; enfermeira da equipe sem motivo; recepção; solicitante digitado sem vínculo; ciência duas vezes): só o solicitante, ou membro da equipe com motivo e exame escalado; repetição = 409. Teste: Task 13 ("ciência por quem não pode").
5. **Prazos na virada do dia e no urgente** (prazo de 15 dias urgente = 8; etapa aberta às 23h59 do fuso da cidade; prestador com prazo próprio só para um exame; o job rodando duas vezes no mesmo dia): atraso conta em dias corridos no fuso da cidade, o escalonamento acontece uma vez só. Teste: Task 11 ("virada do dia e job repetido").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-13-exames-provas-tecnicas.md` (repo docs) | registro das provas técnicas | 0 |
| `contracts/exams/{exam-order-v1,fhir-diagnostic-report-inbound}.json`, `contracts/exams/examples/*`, `contracts/exams/{README,CHANGELOG}.md`, `contracts/README.md` | esquemas e exemplos | 1 |
| `app/services/platform/features.rb`, `app/services/exams/gate.rb`, `spec/adr_pointers_spec.rb`, 3 specs da lista do catálogo, `spec/services/exams/gate_spec.rb` | interruptores (F-21.1) | 2 |
| `db/city_migrate/20261013900001_add_exam_tracking.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/exam_*.rb`, `app/models/{membership,invitation,consultation_exam_request}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/support/exam_helpers.rb`, `spec/models/exam_tracking_schema_spec.rb` | dados da cidade | 3 |
| `app/services/exams/{deadlines,catalog_input}.rb`, `app/controllers/exams/{base_controller,admin/base_controller,admin/catalog_items_controller}.rb`, `config/routes.rb`, specs | catálogo e prazos (F-21.2) | 4 |
| `app/commands/exams/{save_provider,rotate_fhir_token,invite_provider_user}.rb`, `app/commands/accept_invitation.rb`, `app/controllers/exams/admin/providers_controller.rb`, specs | prestadores, token, usuários (F-21.2) | 5 |
| `app/services/exams/files.rb`, `app/controllers/exams/{uploads,files}_controller.rb`, specs | arquivos cifrados e URL curta | 6 |
| `app/services/exams/{transition,open,escalate,access,json,trail,labels}.rb`, specs | núcleo do ciclo de vida (F-21.3) | 7 |
| `app/services/consultations/items_input.rb`, `app/commands/consultations/{finalize,add_addendum}.rb`, `app/services/exams/from_consultation.rb`, specs | pedido da consulta vira exame (F-21.3) | 8 |
| `app/controllers/exams/{orders,order_steps}_controller.rb`, `app/services/exams/order_query.rb`, specs | guia da recepção, leitura, agenda, coleta, cancelamento (F-21.3) | 9 |
| `app/controllers/exams/regulation_controller.rb`, specs | fila de regulação (F-21.4) | 10 |
| `app/jobs/exams/daily_job.rb`, `config/recurring.yml`, `app/services/exams/order_query.rb` (filtros de atraso), specs | prazos, atrasos, escalonamento (F-21.5) | 11 |
| `app/controllers/exams/provider/orders_controller.rb`, `app/services/exams/provider_scope.rb`, specs | portal do prestador (F-21.6) | 12 |
| `app/commands/exams/{register_result,acknowledge}.rb`, `app/services/exams/result_values.rb`, `app/controllers/exams/{results,acknowledgements}_controller.rb`, specs | resultado, ciência, fila da equipe (F-21.8) | 13 |
| `config/exams/fhir-diagnostic-report-inbound.json`, `app/services/exams/fhir/{inbound,service_requests,identifiers}.rb`, `app/controllers/fhir/{base,diagnostic_reports,service_requests}_controller.rb`, `spec/fixtures/exams/*.json`, specs | entrada FHIR (F-21.7) | 14 |
| `lib/fake_lab/{client,report,run}.rb`, `spec/lib/fake_lab_spec.rb`, `docker-compose.yml` (raiz, fora do git) | laboratório simulado (F-21.7) | 15 |
| `app/commands/exams/record_external_result.rb`, `app/controllers/exams/external_results_controller.rb`, specs | resultado externo (F-21.9) | 16 |
| `app/services/exams/release.rb`, `app/controllers/citizen_api/exams_controller.rb`, specs | exames do cidadão (F-21.10) | 17 |
| `app/services/exams/report.rb`, `app/controllers/exams/admin/reports_controller.rb`, specs | relatório de conferência (F-21.12) | 18 |
| `app/services/timeline.rb`, `app/services/timeline/{query.rb,sources/exam_source.rb}`, specs | fornecedor da linha do tempo (F-21.13) | 19 |
| `app/services/pec_external/{page,exam_page}.rb`, `app/controllers/{pec_timelines,pec_exams}_controller.rb`, `lib/pec_simulator.rb`, specs | botão no PEC (F-21.11) | 20 |
| `app/controllers/exams/lookups_controller.rb`, `external_results_controller.rb` (`index`), specs | rotas de apoio do dashboard | 19b |
| `spec/invariants/exam_tracking_invariants_spec.rb` | invariantes do ADR 0036 | 21 |
| `lib/{exam_crew,sigtap_sample}.rb`, `db/seeds.rb`, `spec/lib/exam_crew_spec.rb`, `spec/services/terminology/sigtap_reader_spec.rb` | semente de dev | 22 |
| `docs/operacao/rollout-rastreamento-de-exames.md` (repo docs) | revisão, suíte, porta 3041, prova, runbook | 23 |

---
## Fatia 0 — Base e provas técnicas

### Task 0: Conferir a base e fazer as duas provas técnicas

Esta task não escreve código do api. **Regra:** (a) falhou → **F-21.11 sai do escopo**: retire a Task 20, remova `exam_pec_bridge` do catálogo da Task 2 (e da semente da Task 22), **PARE e avise** o coordenador e o usuário antes de seguir (o contrato §8 e os planos do dashboard mudam); (b) se o FHIR R4 disser algo diferente do Desvio 13, **PARE e reporte** — o perfil entra em esquema, código e teste nas Tasks 1 e 14.

**Files:**
- Create: `docs/pesquisa/2026-10-13-exames-provas-tecnicas.md` (repo `docs`)
- Create (fora de qualquer repo, apagado no fim): `<scratchpad>/pec-probe/probe.rb`

- [ ] **Step 1: Confira os nomes da base** (anote o hash de `origin/main`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api log -1 --format=%h origin/main
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/city_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api grep -n "SIMULATION_ENVS = \|Entry.new(key: \"pec_external_view\"" origin/main -- app/services/platform/features.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def for_person\|def call(user:, patient:" origin/main -- app/services/clinical_record/access.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.materialize!\|def self.apply!" origin/main -- app/commands/consultations/finalize.rb app/commands/consultations/add_addendum.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def exams(list, cbo:, on:, kept: \[\])" origin/main -- app/services/consultations/items_input.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def resume\|def issue\|HEADER = " origin/main -- app/services/pec_external/view_session.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def timeline(display_name:, page:, token:, nonce:)" origin/main -- app/services/pec_external/page.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.call(citizen)\|def call(citizen)" origin/main -- app/commands/patients/resolve.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.ensure_totp\|def ensure_totp" origin/main -- lib/signature_crew.rb
  /opt/homebrew/bin/git -C apps/api grep -n "VALID_RANGE" origin/main -- spec/adr_pointers_spec.rb
  /opt/homebrew/bin/git -C apps/api grep -n "PRIVILEGED_ROLES = " origin/main -- app/models/membership.rb
  ls docs/adr/0036.md
  ```
  Expected: cada grep acha a linha; a última migração de cidade é `20261011800002_add_patient_timeline.rb` (se houver outra maior, use o próximo número livre **acima** dela e ajuste o plano inteiro — Rails 8.1 recusa migração datada >1 dia no futuro: `20261013900001` só vale a partir de 2026-10-12); `VALID_RANGE = (1..35)`; o ADR 0036 existe.

- [ ] **Step 2 (prova a, documentação):** releia `https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/index.html` e `.../iframe.html`. Ao escrever este plano (2026-10-13) as duas páginas ("Alterado em" 11/09/2026 e 16/09/2025) **não dizem nada** sobre `sandbox`, envio de formulário, POST ou cookies — só que o conteúdo não carrega com `X-Frame-Options` DENY/SAMEORIGIN nem com `frame-ancestors 'none'`. Se uma data mudou, leia de novo e anote o que mudou. A documentação, sozinha, não decide: a prova é no PEC local.

- [ ] **Step 3 (prova a, PEC local):** ligue o PEC local se estiver parado (regra do usuário: ligar sem pedir; **nunca** `down -v`):

  ```bash
  cd apps/api/.claude/mod16-exporter/deploy/development/pec
  docker compose --env-file local/pec.env up -d
  until curl -sk -o /dev/null -w '%{http_code}' https://localhost:8443 | grep -q 200; do sleep 15; done
  ```
  (boot ~5 min, emulado amd64; runbook `docs/operacao/pec-local-dev.md`).

- [ ] **Step 4 (prova a, página de prova):** crie `<scratchpad>/pec-probe/probe.rb` (fora de qualquer repo) — um servidor mínimo com um formulário GET e o eco do POST:

  ```ruby
  # Prova técnica do módulo 21a: a tela de "sistemas externos" do PEC deixa a
  # página embutida enviar um formulário (POST)? Só eco; nada é guardado.
  require "webrick"
  require "webrick/https"
  require "cgi"

  server = WEBrick::HTTPServer.new(Port: 8099, BindAddress: "127.0.0.1", SSLEnable: true,
                                   SSLCertName: [ %w[CN localhost] ], AccessLog: [])
  server.mount_proc("/") do |req, res|
    res["Content-Type"] = "text/html; charset=utf-8"
    res["Content-Security-Policy"] = "frame-ancestors https://localhost:8443"
    res.body = if req.request_method == "POST"
                 "<p id='ok'>POST recebido: #{CGI.escapeHTML(req.body.to_s)}</p><p>sandbox? veja o DOM do PEC</p>"
               else
                 "<p>Query: #{CGI.escapeHTML(req.query_string.to_s)}</p>" \
                   "<form method='post' action='/'><input name='exame' value='0202020380'><button>Enviar</button></form>" \
                   "<button id='js' type='button'>fetch POST</button>" \
                   "<script>document.getElementById('js').onclick=async()=>{const r=await fetch('/',{method:'POST',body:'via=fetch'});" \
                   "document.body.insertAdjacentHTML('beforeend',await r.text())}</script>"
               end
  end
  trap("INT") { server.shutdown }
  server.start
  ```
  Rode com `ruby <scratchpad>/pec-probe/probe.rb` (no host). Abra `https://localhost:8099/` uma vez no navegador e aceite o certificado autoassinado (vale para o iframe depois).

- [ ] **Step 5 (prova a, cadastro e execução no PEC):** no PEC local, como administrador da instalação: Integração > Sistemas externos > novo sistema "Rota Saúde — prova 21a", "Local de acesso" = Atendimento, URL `https://localhost:8099/?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]`. Abra um atendimento individual de um cidadão de treinamento, abra a aba do sistema externo e:
  1. no DevTools, inspecione o `<iframe>` do PEC e anote **literalmente** os atributos `sandbox`, `allow` e `referrerpolicy` (ou "ausente");
  2. clique "Enviar" (POST de formulário) e anote se aparece "POST recebido" **dentro** da aba;
  3. clique "fetch POST" e anote o resultado;
  4. anote a versão do PEC (rodapé) e se a URL chegou com `pec_timestamp` e `pec_sign`.

  **Passa** quando (2) mostra "POST recebido" dentro do iframe. Se o PEC recusar a URL com porta ou `https://localhost`, use `https://127.0.0.1:8099/` e anote. Se nem o cadastro for possível no PEC local (o login de administrador da instalação é do usuário), **PARE** e peça ao usuário que faça os passos com você no navegador; não declare a prova sem o passo 2.

- [ ] **Step 6 (prova b, FHIR R4):** confira, nas páginas oficiais, e anote com a URL: `https://hl7.org/fhir/R4/diagnosticreport.html` — `status` 1..1 (valores, incluindo `final` e `amended`), **`code` 1..1**, `subject` 0..1 `Reference`, `basedOn` 0..* `Reference(ServiceRequest…)`, `issued` 0..1 `instant`, `performer` 0..*, `result` 0..* `Reference(Observation)`, `presentedForm` 0..* `Attachment` (`contentType`, `data` base64Binary); `https://hl7.org/fhir/R4/references.html` — `Reference.identifier` permitido e `Reference.display`; `https://hl7.org/fhir/R4/domainresource.html` — invariante **dom-3** (todo recurso `contained` precisa ser referenciado de dentro do recurso, ex.: `"#obs1"`); `https://hl7.org/fhir/R4/observation.html` — `status` e `code` 1..1, `valueQuantity`/`valueString`, `interpretation` 0..*; `https://hl7.org/fhir/R4/v3/ObservationInterpretation/cs.html` — códigos `HH`, `LL`, `AA`; RNDS `https://rnds-fhir.saude.gov.br/NamingSystem-cpf.json.html` e `NamingSystem-cns.json.html` — `uniqueId` `http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf` e `.../cns`. Também `https://hl7.org/fhir/R4/servicerequest.html` — `status`, `intent` e `subject` 1..1 (o `GET /fhir/ServiceRequest` devolve os três).

- [ ] **Step 7: Registre** em `docs/pesquisa/2026-10-13-exames-provas-tecnicas.md` (palavras do registro, sem copiar texto oficial):

  ```markdown
  # Módulo 21a — provas técnicas (2026-10-13)

  ## (a) Formulário dentro da tela de "sistemas externos" do e-SUS PEC
  - Documentação oficial (index "Alterado em" 11/09/2026; iframe 16/09/2025): silenciosa sobre sandbox, formulário, POST e cookies.
  - PEC local: versão <x.y.z>, local de acesso Atendimento.
  - Atributos do iframe: sandbox=<valor literal ou ausente>; allow=<…>; referrerpolicy=<…>.
  - POST de formulário dentro do iframe: <recebido | bloqueado (mensagem do console)>.
  - fetch POST: <recebido | bloqueado>.
  - Resultado: <PASSOU | FALHOU>. Regra: falhou → F-21.11 sai do escopo e a recepção segue cadastrando pela guia.
  - Limite: prova local não garante a versão do PEC da cidade (gate de go-live).

  ## (b) Perfil mínimo do DiagnosticReport de entrada contra o FHIR R4
  - DiagnosticReport: status 1..1 (aceitos: final, amended); code 1..1 (entra no perfil); basedOn[0].identifier.value = short_code;
    subject.identifier (CPF ou CNS RNDS); issued; performer[0].display; presentedForm[0] (application/pdf, data);
    result[] → "#<id>" de cada Observation contida (dom-3).
  - Observation contida: status 1..1, code 1..1 (LOINC em code.coding[]), valueQuantity ou valueString, interpretation (HH/LL/AA = crítico).
  - Sistemas de identificador (RNDS, uniqueId): http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf e .../cns.
  - ServiceRequest (saída): status, intent e subject 1..1.
  - Consequência: divergências D7 do plano do api ao contrato §7.
  ```

- [ ] **Step 8: Desligue** a página de prova (Ctrl-C) e apague `<scratchpad>/pec-probe/`; desligue o PEC local **só** se ele estava desligado no Step 3: `docker compose --env-file local/pec.env down` (**nunca** `-v`). Remova o cadastro "Rota Saúde — prova 21a" do PEC local.

- [ ] **Step 9: Commit no repo docs** (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add pesquisa/2026-10-13-exames-provas-tecnicas.md
  /opt/homebrew/bin/git -C docs commit -m "docs: record module 21a technical proofs (PEC form post and FHIR profile)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 10:** aplique a regra (a)/(b) acima. Depois crie o worktree e os bancos de teste (seção "Ambiente de execução") e rode `rspec spec/adr_pointers_spec.rb spec/requests/timeline_spec.rb` no worktree limpo. Expected: verde.

---

## Fatia 1 — Contrato, interruptores e dados

### Task 1: Repo `contracts` — `exams/exam-order-v1.json` e `exams/fhir-diagnostic-report-inbound.json`

**Files (repo `contracts`, branch `feat/mod-21a-exams`):**
- Create: `exams/exam-order-v1.json`, `exams/fhir-diagnostic-report-inbound.json`
- Create: `exams/examples/exam-order-clinical.json`, `exams/examples/exam-order-administrative.json`, `exams/examples/diagnostic-report-hemograma.json`, `exams/examples/invalid-diagnostic-report-without-pdf.json`, `exams/examples/invalid-diagnostic-report-uncontained-observation.json`, `exams/examples/invalid-diagnostic-report-preliminary.json`, `exams/examples/manifest.json`
- Create: `exams/README.md`, `exams/CHANGELOG.md`
- Modify: `README.md` (tabela de domínios: linha `exams/`)

**Interfaces:**
- Produces: `rotasaude.exam_order.v1` (forma `<exam_order>` do contrato §2, com as Divergências D4/D5) e o perfil `rotasaude.fhir.diagnostic_report_inbound.v1` (§7 com D7). A Task 14 copia o segundo para `apps/api/config/exams/` e prende o SHA-256; a Task 9 valida respostas contra o primeiro.

- [ ] **Step 1: Branch** `/opt/homebrew/bin/git -C contracts fetch origin && /opt/homebrew/bin/git -C contracts switch -c feat/mod-21a-exams origin/main`.

- [ ] **Step 2: Escreva `exams/fhir-diagnostic-report-inbound.json`:**

  ```json
  {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://rota-saude/schemas/exams/fhir-diagnostic-report-inbound.json",
    "title": "rotasaude.fhir.diagnostic_report_inbound.v1",
    "description": "Perfil mínimo do DiagnosticReport FHIR R4 que um prestador envia ao Rota Saúde (POST /fhir/DiagnosticReport; ADR 0036). Aberto (additionalProperties livre): o laboratório pode mandar mais do que isto; o Rota Saúde lê só o que está aqui. Observations vão em contained e são referenciadas em result (invariante dom-3). Sistemas de CPF e CNS = uniqueId da RNDS.",
    "type": "object",
    "required": ["resourceType", "identifier", "status", "code", "basedOn", "subject", "issued", "performer", "presentedForm", "result", "contained"],
    "properties": {
      "resourceType": { "const": "DiagnosticReport" },
      "identifier": { "type": "array", "minItems": 1, "items": { "type": "object", "required": ["value"], "properties": { "system": { "type": "string" }, "value": { "type": "string", "minLength": 1, "maxLength": 128 } } } },
      "status": { "enum": ["final", "amended"] },
      "code": { "type": "object", "required": ["coding"], "properties": { "coding": { "type": "array", "minItems": 1, "items": { "type": "object", "required": ["code"], "properties": { "system": { "type": "string" }, "code": { "type": "string", "minLength": 1 } } } } } },
      "basedOn": { "type": "array", "minItems": 1, "items": { "type": "object", "required": ["identifier"], "properties": { "identifier": { "type": "object", "required": ["value"], "properties": { "value": { "type": "string", "pattern": "^[23456789ABCDEFGHJKMNPQRSTUVWXYZ]{10}$" } } } } } },
      "subject": { "type": "object", "required": ["identifier"], "properties": { "identifier": { "type": "object", "required": ["system", "value"], "oneOf": [
        { "properties": { "system": { "const": "http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf" }, "value": { "type": "string", "pattern": "^[0-9]{11}$" } } },
        { "properties": { "system": { "const": "http://rnds.saude.gov.br/fhir/r4/NamingSystem/cns" }, "value": { "type": "string", "pattern": "^[0-9]{15}$" } } }
      ] } } },
      "issued": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}(\\.[0-9]+)?(Z|[+-][0-9]{2}:[0-9]{2})$" },
      "performer": { "type": "array", "minItems": 1, "items": { "type": "object", "required": ["display"], "properties": { "display": { "type": "string", "minLength": 1, "maxLength": 160 } } } },
      "presentedForm": { "type": "array", "minItems": 1, "items": { "type": "object", "required": ["contentType", "data"], "properties": { "contentType": { "const": "application/pdf" }, "data": { "type": "string", "minLength": 8, "contentEncoding": "base64" } } } },
      "result": { "type": "array", "items": { "type": "object", "required": ["reference"], "properties": { "reference": { "type": "string", "pattern": "^#[A-Za-z0-9.-]{1,64}$" } } } },
      "contained": { "type": "array", "items": { "$ref": "#/$defs/observation" } }
    },
    "$defs": {
      "observation": {
        "type": "object",
        "required": ["resourceType", "id", "status", "code"],
        "properties": {
          "resourceType": { "const": "Observation" },
          "id": { "type": "string", "pattern": "^[A-Za-z0-9.-]{1,64}$" },
          "status": { "enum": ["final", "amended", "corrected", "preliminary"] },
          "code": { "type": "object", "required": ["coding"], "properties": { "coding": { "type": "array", "minItems": 1, "items": { "type": "object", "required": ["system", "code"], "properties": { "system": { "type": "string" }, "code": { "type": "string" }, "display": { "type": "string" } } } } } },
          "valueQuantity": { "type": "object", "required": ["value"], "properties": { "value": { "type": "number" }, "unit": { "type": "string" } } },
          "valueString": { "type": "string", "maxLength": 4000 },
          "interpretation": { "type": "array", "items": { "type": "object", "properties": { "coding": { "type": "array", "items": { "type": "object", "properties": { "code": { "type": "string" } } } } } } }
        },
        "oneOf": [ { "required": ["valueQuantity"] }, { "required": ["valueString"] } ]
      }
    }
  }
  ```

  O esquema não consegue provar dom-3 (que cada `contained[].id` aparece em `result`): isso fica no api (Task 14, `invalid_resource` com `details: ["/result"]`) e no exemplo inválido `invalid-diagnostic-report-uncontained-observation.json`, que o `manifest.json` marca com `"enforced_by": "api"`.

- [ ] **Step 3: Escreva `exams/examples/diagnostic-report-hemograma.json`** (dados fictícios; CPF válido de exemplo `52998224725`; PDF = `%PDF-1.4` mínimo em Base64):

  ```json
  {
    "resourceType": "DiagnosticReport",
    "identifier": [ { "system": "urn:lab:laboratorio-simulado", "value": "LS-2026-000123" } ],
    "status": "final",
    "code": { "coding": [ { "system": "http://www.saude.gov.br/fhir/r4/CodeSystem/BRTabelaSUS", "code": "0202020380", "display": "HEMOGRAMA COMPLETO" } ] },
    "basedOn": [ { "identifier": { "value": "K7M2Q9XR4T" } } ],
    "subject": { "identifier": { "system": "http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf", "value": "52998224725" } },
    "issued": "2026-10-13T14:05:00-03:00",
    "performer": [ { "display": "Laboratório Simulado" } ],
    "presentedForm": [ { "contentType": "application/pdf", "data": "JVBERi0xLjQKJcOkw7zDtsOfCjEgMCBvYmoKPDw+PgplbmRvYmoKdHJhaWxlcgo8PC9Sb290IDEgMCBSPj4KJSVFT0YK" } ],
    "result": [ { "reference": "#hb" }, { "reference": "#plt" } ],
    "contained": [
      { "resourceType": "Observation", "id": "hb", "status": "final",
        "code": { "coding": [ { "system": "http://loinc.org", "code": "718-7", "display": "Hemoglobin [Mass/volume] in Blood" } ] },
        "valueQuantity": { "value": 13.4, "unit": "g/dL" } },
      { "resourceType": "Observation", "id": "plt", "status": "final",
        "code": { "coding": [ { "system": "http://loinc.org", "code": "777-3", "display": "Platelets [#/volume] in Blood" } ] },
        "valueQuantity": { "value": 41, "unit": "10*3/uL" },
        "interpretation": [ { "coding": [ { "system": "http://terminology.hl7.org/CodeSystem/v3-ObservationInterpretation", "code": "LL" } ] } ] }
    ]
  }
  ```
  e os três inválidos: sem `presentedForm`; `contained` com uma Observation `id: "x"` fora de `result`; `status: "preliminary"`. `manifest.json`:

  ```json
  { "suites": [
    { "schema": "fhir-diagnostic-report-inbound.json",
      "cases": [
        { "file": "diagnostic-report-hemograma.json", "valid": true },
        { "file": "invalid-diagnostic-report-without-pdf.json", "valid": false, "expect": "/presentedForm" },
        { "file": "invalid-diagnostic-report-preliminary.json", "valid": false, "expect": "/status" },
        { "file": "invalid-diagnostic-report-uncontained-observation.json", "valid": true, "enforced_by": "api", "api_error": "invalid_resource" } ] },
    { "schema": "exam-order-v1.json",
      "cases": [
        { "file": "exam-order-clinical.json", "valid": true },
        { "file": "exam-order-administrative.json", "valid": true } ] } ] }
  ```

- [ ] **Step 4: Escreva `exams/exam-order-v1.json`** (fechado; `<exam_order>` do contrato §2 com D4, D5 e D16; `events` é opcional porque as listas o omitem):

  ```json
  {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://rota-saude/schemas/exams/exam-order-v1.json",
    "title": "rotasaude.exam_order.v1",
    "description": "O exame como o api devolve ao dashboard (ADR 0036; contrato §2). Duas visões: clínica (result com valores, crítico e PDF) e administrativa (result reduzido; cid10_justification e requester.cns nulos). events e result_fields só no detalhe (GET /exams/orders/:id).",
    "type": "object",
    "additionalProperties": false,
    "required": ["id", "short_code", "patient", "sigtap_code", "sigtap_competence", "label", "priority", "cid10_justification", "origin",
                 "consultation_id", "requester", "unit", "provider", "scheduled_at", "scheduled_place", "external_reference", "status", "status_changed_at",
                 "overdue", "escalated", "guide_url", "result"],
    "properties": {
      "id": { "$ref": "#/$defs/uuid" },
      "short_code": { "type": "string", "pattern": "^[23456789ABCDEFGHJKMNPQRSTUVWXYZ]{10}$" },
      "patient": { "type": "object", "additionalProperties": false, "required": ["id", "display_name"],
                   "properties": { "id": { "$ref": "#/$defs/uuid" }, "display_name": { "type": ["string", "null"] } } },
      "sigtap_code": { "type": "string", "pattern": "^02[0-9]{8}$" },
      "sigtap_competence": { "type": "string", "pattern": "^[0-9]{4}(0[1-9]|1[0-2])$" },
      "label": { "type": "string", "minLength": 1 },
      "priority": { "enum": ["routine", "urgent"] },
      "cid10_justification": { "type": ["string", "null"], "pattern": "^[A-Z][0-9]{2}[0-9X]?$" },
      "origin": { "enum": ["consultation", "reception_guide", "pec"] },
      "consultation_id": { "anyOf": [{ "$ref": "#/$defs/uuid" }, { "type": "null" }] },
      "result_fields": { "type": "array", "items": { "type": "object", "additionalProperties": false, "required": ["key", "label", "unit", "loinc"],
                         "properties": { "key": { "type": "string" }, "label": { "type": "string" }, "unit": { "type": ["string", "null"] },
                                         "loinc": { "type": ["string", "null"] } } } },
      "requester": { "type": "object", "additionalProperties": false, "required": ["user_id", "name", "cns"],
                     "properties": { "user_id": { "anyOf": [{ "$ref": "#/$defs/uuid" }, { "type": "null" }] },
                                     "name": { "type": ["string", "null"] }, "cns": { "type": ["string", "null"], "pattern": "^[0-9]{15}$" } } },
      "unit": { "$ref": "#/$defs/ref" },
      "provider": { "anyOf": [{ "$ref": "#/$defs/ref" }, { "type": "null" }] },
      "scheduled_at": { "anyOf": [{ "$ref": "#/$defs/instant" }, { "type": "null" }] },
      "scheduled_place": { "type": ["string", "null"], "maxLength": 200 },
      "external_reference": { "type": ["string", "null"], "maxLength": 60 },
      "status": { "enum": ["awaiting_authorization", "denied", "awaiting_schedule", "scheduled", "collected", "resulted", "acknowledged", "cancelled"] },
      "status_changed_at": { "$ref": "#/$defs/instant" },
      "overdue": { "anyOf": [{ "type": "null" }, { "type": "object", "additionalProperties": false, "required": ["stage", "due_on", "days_late"],
                   "properties": { "stage": { "$ref": "#/$defs/stage" }, "due_on": { "$ref": "#/$defs/date" },
                                   "days_late": { "type": "integer", "minimum": 1 } } }] },
      "escalated": { "type": "boolean" },
      "guide_url": { "type": ["string", "null"], "pattern": "^/exams/files/" },
      "result": { "anyOf": [{ "type": "null" }, { "$ref": "#/$defs/result_clinical" }, { "$ref": "#/$defs/result_administrative" }] },
      "events": { "type": "array", "items": { "$ref": "#/$defs/event" } }
    },
    "$defs": {
      "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
      "date": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
      "instant": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}(Z|[+-][0-9]{2}:[0-9]{2})$" },
      "stage": { "enum": ["authorize", "schedule", "collect", "result", "acknowledge"] },
      "ref": { "type": "object", "additionalProperties": false, "required": ["id", "name"],
               "properties": { "id": { "$ref": "#/$defs/uuid" }, "name": { "type": "string" } } },
      "result_administrative": { "type": "object", "additionalProperties": false,
        "required": ["id", "origin", "source", "released_on", "laboratory_name"],
        "properties": { "id": { "$ref": "#/$defs/uuid" }, "origin": { "enum": ["order", "external"] },
                        "source": { "enum": ["unit", "provider", "fhir", "consultation"] },
                        "released_on": { "$ref": "#/$defs/date" }, "laboratory_name": { "type": "string" } } },
      "result_clinical": { "type": "object", "additionalProperties": false,
        "required": ["id", "origin", "source", "released_on", "laboratory_name", "critical", "values", "pdf_url", "acknowledged_at"],
        "properties": { "id": { "$ref": "#/$defs/uuid" }, "origin": { "enum": ["order", "external"] },
                        "source": { "enum": ["unit", "provider", "fhir", "consultation"] },
                        "released_on": { "$ref": "#/$defs/date" }, "laboratory_name": { "type": "string" }, "critical": { "type": "boolean" },
                        "values": { "type": "array", "items": { "type": "object", "additionalProperties": false,
                          "required": ["key", "label", "value", "unit", "loinc"],
                          "properties": { "key": { "type": ["string", "null"] }, "label": { "type": "string" }, "value": { "type": "string" },
                                          "unit": { "type": ["string", "null"] }, "loinc": { "type": ["string", "null"] } } } },
                        "pdf_url": { "type": "string", "pattern": "^/exams/files/" },
                        "acknowledged_at": { "anyOf": [{ "$ref": "#/$defs/instant" }, { "type": "null" }] } } },
      "event": { "type": "object", "additionalProperties": false, "required": ["kind", "occurred_at", "actor", "reason"],
        "properties": { "kind": { "enum": ["requested", "authorized", "denied", "scheduled", "collected", "resulted", "acknowledged", "escalated", "cancelled"] },
                        "occurred_at": { "$ref": "#/$defs/instant" },
                        "actor": { "type": "object", "additionalProperties": false, "required": ["type", "name"],
                                   "properties": { "type": { "enum": ["user", "provider", "system"] }, "name": { "type": ["string", "null"] } } },
                        "reason": { "type": ["string", "null"] } } }
    }
  }
  ```

  (`result_clinical` e `result_administrative` são distinguíveis porque o administrativo é fechado sem `critical`; o `anyOf` aceita um ou outro.) Escreva os exemplos `exam-order-clinical.json` (um hemograma `resulted`, `result` com dois valores, `events` com `requested`, `scheduled`, `collected`, `resulted`) e `exam-order-administrative.json` (o mesmo, visão administrativa: `cid10_justification` e `requester.cns` nulos, `result` reduzido, sem `events`) (já listados no `manifest.json` do Step 3).

- [ ] **Step 5: Valide os exemplos** com o `json_schemer` do container do api (sem dependência nova no contracts):

  ```bash
  docker compose run --rm -v "$PWD/contracts:/contracts" -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
    %w[fhir-diagnostic-report-inbound exam-order-v1].each do |name|
      schema = JSONSchemer.schema(JSON.parse(File.read("/contracts/exams/#{name}.json")))
      Dir["/contracts/exams/examples/*.json"].reject { |f| f.end_with?("manifest.json") }.each do |f|
        next unless (name.start_with?("fhir") ? f.include?("diagnostic-report") : f.include?("exam-order"))
        errors = schema.validate(JSON.parse(File.read(f))).map { |e| e["data_pointer"] }
        puts "#{File.basename(f)}: #{errors.empty? ? "valid" : errors.join(",")}"
      end
    end'
  ```
  Expected: `diagnostic-report-hemograma.json: valid`, `invalid-…-without-pdf.json: …/presentedForm…`, `invalid-…-preliminary.json: /status`, `invalid-…-uncontained-observation.json: valid` (o api recusa), `exam-order-*.json: valid`.

- [ ] **Step 6: README e CHANGELOG.** `exams/README.md`: o que é cada arquivo, regras (perfil aberto × `exam-order` fechado; dom-3 no api; sistemas RNDS; `interpretation` HH/LL/AA = crítico; idempotência por `identifier` + prestador; corpo até 15 MB; só `application/fhir+json`). `exams/CHANGELOG.md`: `## exams-v1.0.0 — 2026-10-13` com a lista. No `README.md` da raiz, a linha `| [`exams/`](exams/README.md) | Pedido de exame (`<exam_order>`) e perfil FHIR de entrada do laudo (ADR 0036) | `exams-v1.0.0` | Materializado |` e "Sete domínios".

- [ ] **Step 7: Commit** (sem push, sem tag):

  ```bash
  /opt/homebrew/bin/git -C contracts add exams/exam-order-v1.json exams/fhir-diagnostic-report-inbound.json exams/examples exams/README.md exams/CHANGELOG.md README.md
  /opt/homebrew/bin/git -C contracts commit -m "feat: add exam order and inbound FHIR diagnostic report schemas (exams-v1.0.0)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  A tag `exams-v1.0.0` e o push **só com autorização do usuário**, uma etapa de cada vez (`git tag exams-v1.0.0 && git push origin feat/mod-21a-exams --tags` depois do merge). A Task 14 prende o SHA-256 do arquivo **commitado**; se o esquema mudar depois, a spec do api quebra de propósito.

### Task 2: Interruptores (F-21.1)

**Files:**
- Modify: `app/services/platform/features.rb` (`CATALOG`)
- Create: `app/services/exams/gate.rb`
- Modify: `spec/adr_pointers_spec.rb:13`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30`
- Test: `spec/services/exams/gate_spec.rb`

**Interfaces:**
- Produces: `Exams::Gate::KEY = "exam_tracking"`, `FHIR_KEY = "exam_fhir_inbound"`, `PEC_KEY = "exam_pec_bridge"`, `FAKE_LAB_KEY = "fake_lab"`; `Exams::Gate.usable?(city)`, `.fhir_usable?(city)`, `.pec_usable?(city)`, `.fake_lab_usable?(city)` → Boolean (falso para `nil`/sem id).

- [ ] **Step 1: Write the failing test** `spec/services/exams/gate_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0036 (contrato §11; Desvio 1 / Divergência D1): exam_tracking exige só
  # record_mode (record ou integrated); os outros três em cadeia; fake_lab
  # nunca em produção.
  RSpec.describe Exams::Gate do
    def set(city, key, enabled = true)
      Platform::Features.set!(city: city, key: key, enabled: enabled, maintainer: ledi_maintainer!)
    end

    it "exam_tracking vale em record e integrated, nunca em off" do
      city = clinical_city!(record_mode: "off", enabled: false)
      set(city, "exam_tracking")
      expect(Platform::Features.missing(city, "exam_tracking")).to eq(%w[record_mode_off])
      expect(described_class.usable?(city)).to be(false)
      city.update!(record_mode: "integrated")
      expect(described_class.usable?(city)).to be(true)
      city.update!(record_mode: "record")
      expect(described_class.usable?(city)).to be(true)
      expect(described_class.usable?(nil)).to be(false)
    end

    it "a cadeia: fhir exige exam_tracking; fake_lab exige fhir; pec exige exam_tracking e pec_external_view" do
      city = clinical_city!(record_mode: "integrated", enabled: false)
      %w[exam_fhir_inbound fake_lab exam_pec_bridge].each { |key| set(city, key) }
      expect(Platform::Features.missing(city, "exam_fhir_inbound")).to eq(%w[exam_tracking_disabled])
      expect(Platform::Features.missing(city, "fake_lab")).to eq(%w[exam_fhir_inbound_disabled])
      expect(Platform::Features.missing(city, "exam_pec_bridge")).to eq(%w[exam_tracking_disabled pec_external_view_disabled])
      set(city, "exam_tracking")
      expect(described_class.fhir_usable?(city)).to be(true)
      expect(described_class.fake_lab_usable?(city)).to be(true)
      expect(Platform::Features.missing(city, "exam_pec_bridge")).to eq(%w[pec_external_view_disabled])
      expect(described_class.pec_usable?(city)).to be(false)
    end

    it "fake_lab não existe em produção (nem no catálogo, nem para ligar)" do
      expect(Platform::Features.catalog(env: "production").map(&:key)).not_to include("fake_lab")
      expect(Platform::Features.catalog(env: "production").map(&:key)).to include("exam_tracking", "exam_fhir_inbound", "exam_pec_bridge")
      city = clinical_city!
      expect { Platform::Features.set!(city: city, key: "fake_lab", enabled: true, maintainer: ledi_maintainer!, env: "production") }
        .to raise_error(Platform::Features::UnknownFeature)
      expect(described_class.fake_lab_usable?(city, env: "production")).to be(false)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/exams/gate_spec.rb`. Expected: FAIL com `uninitialized constant Exams`.

- [ ] **Step 3: Implemente.** Em `app/services/platform/features.rb`, depois da entrada `pec_external_view`:

  ```ruby
      # ADR 0036 (Divergência D1): rastreamento do exame (módulo 21a). Vale no
      # modo record e no integrado (guia da recepção e botão no PEC); off não.
      Entry.new(key: "exam_tracking",
                description: "Rastreamento do exame (pedido, regulação, prazos, resultado e ciência)",
                requires: %w[record_mode]),
      # ADR 0036: entrada FHIR do laudo (POST /fhir/DiagnosticReport) por token do prestador.
      Entry.new(key: "exam_fhir_inbound",
                description: "Entrada FHIR do laudo do prestador (DiagnosticReport)",
                requires: %w[feature:exam_tracking]),
      # ADR 0036: "Solicitar exames" dentro da página do PEC (módulo 20).
      Entry.new(key: "exam_pec_bridge",
                description: "Solicitar exames pela tela de sistemas externos do e-SUS PEC",
                requires: %w[feature:exam_tracking feature:pec_external_view]),
      # ADR 0036: laboratório simulado, só fora de produção.
      Entry.new(key: "fake_lab",
                description: "Laboratório simulado (desenvolvimento) — laudos de exemplo, sem validade",
                requires: %w[feature:exam_fhir_inbound],
                environments: SIMULATION_ENVS)
  ```

  Crie `app/services/exams/gate.rb`:

  ```ruby
  # Interruptores do módulo 21a (ADR 0036): LIGADO E UTILIZÁVEL, relido da
  # plataforma a cada chamada (record_mode por id). fake_lab só existe fora de
  # produção (Platform::Features::SIMULATION_ENVS).
  module Exams
    module Gate
      KEY = "exam_tracking".freeze
      FHIR_KEY = "exam_fhir_inbound".freeze
      PEC_KEY = "exam_pec_bridge".freeze
      FAKE_LAB_KEY = "fake_lab".freeze

      module_function

      def usable?(city, env: Rails.env) = check(city, KEY, env)
      def fhir_usable?(city, env: Rails.env) = check(city, FHIR_KEY, env)
      def pec_usable?(city, env: Rails.env) = check(city, PEC_KEY, env)
      def fake_lab_usable?(city, env: Rails.env) = check(city, FAKE_LAB_KEY, env)

      def check(city, key, env)
        return false if city.nil? || city.id.nil?

        Platform::Features.usable?(city, key, env: env)
      end
      private_class_method :check
    end
  end
  ```

  Nas três specs com a lista fixa do catálogo, acrescente no fim: `exam_tracking exam_fhir_inbound exam_pec_bridge fake_lab`. Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..36).freeze`.

- [ ] **Step 4: Run** `rspec spec/services/exams/gate_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/adr_pointers_spec.rb spec/services/platform spec/invariants/controlled_prescriptions_invariants_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/platform/features.rb app/services/exams/gate.rb spec/services/exams/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb
  git commit -m "feat: add exam tracking, fhir inbound, pec bridge and fake lab feature switches

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.1 → Done (o lado do maintenance é do plano do maintenance; o card fecha quando os dois estiverem prontos — mova para Done só o que este plano entrega e anote no card).

### Task 3: Migração de cidade, triggers, modelos, eventos e filtros

**Files:**
- Create: `db/city_migrate/20261013900001_add_exam_tracking.rb`
- Modify: `db/city_schema.rb` (versão, `consultation_exam_requests`, `invitations`, `memberships`, tabelas novas e FKs), `db/city_triggers.sql`
- Create: `app/models/{exam_provider,exam_catalog_item,exam_provider_item,exam_provider_user,exam_file,exam_order,exam_order_event,exam_result}.rb`
- Modify: `app/models/{membership,invitation,consultation_exam_request}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Create: `spec/support/exam_helpers.rb`
- Test: `spec/models/exam_tracking_schema_spec.rb`

**Interfaces:**
- Consumes: `Exams::Gate` (Task 2).
- Produces: modelos `ExamProvider` (`KINDS`, `#fhir_token_active?`, `.by_token(token)`), `ExamCatalogItem` (`POLICIES`, `STAGES`, `#days_for(stage) -> Integer|nil`, `#result_field(key)`, `#result_field_by_loinc(code)`), `ExamProviderItem` (`#days_for(stage)`), `ExamProviderUser`, `ExamFile` (`#bytes`, `#consumed?`), `ExamOrder` (`STATUSES`, `ORIGINS`, `PRIORITIES`, `#overdue(today:) -> { stage:, due_on:, days_late: }|nil`, `#requester_display_name`, `has_many :events`, `has_one :result`), `ExamOrderEvent` (`KINDS`, `ESCALATION_REASONS`), `ExamResult` (`#values_list -> Array<Hash>`). Helpers de spec: `exam_city!(record_mode: "record", fhir: false, fake_lab: false, pec: false)`, `exam_catalog_item!(sigtap_code: "0202010503", **attrs)`, `exam_provider!(name: "Laboratório Central", token: nil, items: :all, **attrs)`, `exam_seq!`, `exam_patient!(n)` → `[patient, citizen]`, `exam_team!(unit, *users)`, `pdf_file!(user: nil, provider: nil)`, `PDF_BYTES`, `exam_order!(patient:, unit:, requester:, sigtap_code: "0202010503", priority: "routine", provider: nil, status: :awaiting_schedule)`.

- [ ] **Step 1: Write the failing test** `spec/models/exam_tracking_schema_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0036 (Invariantes; Desvio 6): a trilha só cresce; o cache da situação
  # só muda com um evento da mesma transação; o resultado não muda; o arquivo
  # só é apagado se nunca foi usado; valores e PDF cifrados.
  RSpec.describe "Banco do rastreamento de exames" do
    before { exam_city! }

    let(:unit) { create_unit("UBS Exames") }
    let(:doctor) { doctor!(unit) }
    let(:order) { exam_order!(patient: exam_patient!(1).first, unit: unit, requester: doctor) }

    it "exam_order_events é só acréscimo" do
      event = order.events.first
      expect { event.update!(occurred_at: 1.day.ago) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { event.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    # Dentro da fixture transacional todo evento tem o txid do teste: a recusa
    # "sem evento da mesma transação" é provada no describe não transacional abaixo.
    it "o cache muda com o evento; a identidade nunca; nunca some" do
      ApplicationRecord.transaction do
        ExamOrderEvent.create!(exam_order: order, kind: "scheduled", actor_user: doctor, occurred_at: Time.current)
        order.update!(status: "scheduled", status_changed_at: Time.current)
      end
      expect(order.reload.status).to eq("scheduled")
      expect do
        ApplicationRecord.transaction do
          ExamOrderEvent.create!(exam_order: order, kind: "collected", actor_user: doctor, occurred_at: Time.current)
          order.update!(sigtap_code: "0202020380")
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /identity never changes/)
      expect { order.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
    end

    it "pedido nasce aguardando; banco recusa solicitante ausente e origem incoerente" do
      expect { exam_order!(patient: order.patient, unit: unit, requester: doctor, status: :scheduled) }
        .to raise_error(ActiveRecord::StatementInvalid, /born awaiting/)
      attrs = order.attributes.except("id", "short_code", "created_at", "updated_at").merge("short_code" => "ABCDEFGH23")
      expect { ExamOrder.create!(attrs.merge("requester_user_id" => nil, "requester_name" => nil)) }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_exam_orders_requester/)
      expect { ExamOrder.create!(attrs.merge("origin" => "consultation")) }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_exam_orders_consultation/)
    end

    it "resultado imutável e cifrado; arquivo usado não some" do
      file = pdf_file!(user: doctor)
      result = ExamResult.create!(exam_order: order, patient: order.patient, sigtap_code: order.sigtap_code, origin: "order",
                                  source: "unit", pdf_file: file, values: [ { "key" => "hba1c", "value" => "9.1" } ].to_json,
                                  released_on: Time.zone.today, laboratory_name: "Lab", critical: false, registered_by_user: doctor)
      raw = ApplicationRecord.connection.select_value("SELECT values FROM exam_results WHERE id = #{ApplicationRecord.connection.quote(result.id)}")
      expect(raw).not_to include("9.1")
      expect(result.reload.values_list.first["value"]).to eq("9.1")
      expect(result.inspect).not_to include("9.1")
      expect { result.update!(critical: true) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      file.update!(consumed_at: Time.current)
      expect { file.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /consumed/)
      expect(ExamFile.find(file.id).bytes).to eq(ExamHelpers::PDF_BYTES)
      expect(ApplicationRecord.connection.select_value("SELECT data FROM exam_files WHERE id = #{ApplicationRecord.connection.quote(file.id)}"))
        .not_to include(Base64.strict_encode64(ExamHelpers::PDF_BYTES))
    end

    it "papéis novos no banco e privilegiados" do
      user = staff_with("regulacao-#{SecureRandom.hex(3)}@cidade.gov.br", "exam_regulator", "exam_provider")
      expect(user.has_role?("exam_regulator")).to be(true)
      expect(Membership::PRIVILEGED_ROLES).to include("exam_regulator", "exam_provider")
    end

    it "prioridade no pedido do 19a, padrão routine" do
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
      expect(consultation.exam_requests.first.priority).to eq("routine")
    end
  end

  # Mutação: tirar `e.txid = txid_current()` de rota_exam_order_guard. Fora de
  # fixture transacional (como o teste do patient_problems do 19a): o evento
  # commita numa transação e o UPDATE vem de outra. Linhas sintéticas com os
  # triggers desligados (replica dispensa as FKs).
  RSpec.describe "exam_orders: evento de outra transação não basta (ADR 0036)" do
    self.use_transactional_tests = false

    let(:order_id) { SecureRandom.uuid }
    let(:conn) { ApplicationRecord.connection }

    def as_replica(*statements)
      CityConnection.with(TEST_CITY_A) do
        ApplicationRecord.transaction do
          conn.execute("SET LOCAL session_replication_role = replica")
          statements.each { |sql| conn.execute(sql) }
        end
      end
    end

    before do
      as_replica(<<~SQL, "INSERT INTO exam_order_events (exam_order_id, kind, occurred_at, created_at) VALUES ('#{order_id}', 'requested', now(), now())")
        INSERT INTO exam_orders (id, patient_id, sigtap_code, sigtap_competence, priority, requester_name, unit_id, origin, short_code,
                                 status, status_changed_at, created_at, updated_at)
        VALUES ('#{order_id}', gen_random_uuid(), '0202010503', '202610', 'routine', 'x', gen_random_uuid(), 'pec', 'ABCDEFGH23',
                'awaiting_schedule', now(), now(), now())
      SQL
    end

    after do
      as_replica("DELETE FROM exam_order_events WHERE exam_order_id = '#{order_id}'", "DELETE FROM exam_orders WHERE id = '#{order_id}'")
    end

    it "recusa o UPDATE do cache quando o evento foi commitado em outra transação" do
      CityConnection.with(TEST_CITY_A) do
        expect { ApplicationRecord.transaction { conn.execute("UPDATE exam_orders SET status = 'scheduled' WHERE id = '#{order_id}'") } }
          .to raise_error(ActiveRecord::StatementInvalid, /status changes only with an event of the same transaction/)
      end
    end

    it "aceita o mesmo UPDATE com evento da mesma transação (controle)" do
      CityConnection.with(TEST_CITY_A) do
        ApplicationRecord.transaction do
          conn.execute("INSERT INTO exam_order_events (exam_order_id, kind, occurred_at, created_at) VALUES ('#{order_id}', 'scheduled', now(), now())")
          conn.execute("UPDATE exam_orders SET status = 'scheduled' WHERE id = '#{order_id}'")
          expect(conn.select_value("SELECT status FROM exam_orders WHERE id = '#{order_id}'")).to eq("scheduled")
          raise ActiveRecord::Rollback
        end
      end
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/models/exam_tracking_schema_spec.rb`. Expected: FAIL (`exam_city!` indefinido).

- [ ] **Step 3: A migração** `db/city_migrate/20261013900001_add_exam_tracking.rb`:

  ```ruby
  # Módulo 21a (ADR 0036): rastreamento do exame. Catálogo e prestadores da
  # cidade, o pedido com cache da situação, a trilha só de acréscimo, o
  # resultado imutável e os arquivos cifrados (guia e laudo). Prioridade no
  # pedido do 19a. Papéis exam_regulator e exam_provider. Convite ligado a um
  # prestador. Triggers em db/city_triggers.sql. Tabelas novas: índices sem
  # CONCURRENTLY. Irreversível.
  class AddExamTracking < ActiveRecord::Migration[8.1]
    ROLES = %w[analyst campaign_manager citizen_verifier exam_provider exam_regulator health_professional municipal_admin
               protocol_author protocol_publisher protocol_reviewer viewer].freeze
    STAGES = %w[authorize schedule collect result acknowledge].freeze

    def up
      add_column :consultation_exam_requests, :priority, :string, null: false, default: "routine"
      add_check_constraint :consultation_exam_requests, "priority::text = ANY (ARRAY['routine'::text, 'urgent'::text])",
                           name: "ck_consultation_exam_requests_priority"
      remove_check_constraint :memberships, name: "ck_memberships_role"
      add_check_constraint :memberships, "role::text = ANY (ARRAY[#{quoted(ROLES)}]::text[])", name: "ck_memberships_role"

      create_table :exam_providers, id: :uuid do |t|
        t.string :name, limit: 120, null: false
        t.string :cnes, limit: 7
        t.string :kind, null: false
        t.boolean :active, null: false, default: true
        t.boolean :simulated, null: false, default: false
        t.string :fhir_token_digest, limit: 64
        t.datetime :fhir_token_rotated_at
        t.datetime :fhir_token_revoked_at
        t.timestamps
      end
      add_index :exam_providers, "lower((name)::text)", unique: true, name: "idx_exam_providers_name_ci"
      add_index :exam_providers, :fhir_token_digest, unique: true, where: "fhir_token_digest IS NOT NULL",
                                                     name: "idx_exam_providers_token"
      add_check_constraint :exam_providers, "kind::text = ANY (ARRAY['municipal'::text, 'accredited'::text, 'consortium'::text, 'lacen'::text, 'imaging'::text])",
                           name: "ck_exam_providers_kind"
      add_check_constraint :exam_providers, "cnes IS NULL OR cnes::text ~ '^[0-9]{7}$'::text", name: "ck_exam_providers_cnes"

      create_table :exam_catalog_items, id: :uuid do |t|
        t.string :sigtap_code, limit: 10, null: false
        t.boolean :requires_authorization, null: false, default: false
        t.string :release_policy, null: false, default: "after_acknowledgement"
        STAGES.each { |stage| t.integer :"#{stage}_days" }
        t.jsonb :result_fields, null: false, default: []
        t.boolean :active, null: false, default: true
        t.timestamps
      end
      add_index :exam_catalog_items, :sigtap_code, unique: true
      add_check_constraint :exam_catalog_items, "sigtap_code::text ~ '^02[0-9]{8}$'::text", name: "ck_exam_catalog_items_sigtap"
      add_check_constraint :exam_catalog_items, "release_policy::text = ANY (ARRAY['immediate'::text, 'after_acknowledgement'::text, 'never_citizen'::text])",
                           name: "ck_exam_catalog_items_policy"
      add_check_constraint :exam_catalog_items, days_check, name: "ck_exam_catalog_items_days"

      create_table :exam_provider_items, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.references :exam_catalog_item, type: :uuid, null: false, foreign_key: true
        STAGES.each { |stage| t.integer :"#{stage}_days" }
        t.timestamps
      end
      add_index :exam_provider_items, %i[exam_provider_id exam_catalog_item_id], unique: true, name: "idx_exam_provider_items_pair"
      add_check_constraint :exam_provider_items, days_check, name: "ck_exam_provider_items_days"

      create_table :exam_provider_users, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true
        t.references :user, type: :uuid, null: false, foreign_key: true, index: { unique: true }
        t.datetime :created_at, null: false
      end

      add_reference :invitations, :exam_provider, type: :uuid, foreign_key: true

      create_table :exam_files, id: :uuid do |t|
        t.string :content_type, null: false
        t.integer :byte_size, null: false
        t.string :sha256, limit: 64, null: false
        t.text :data, null: false
        t.references :uploaded_by_user, type: :uuid, foreign_key: { to_table: :users }
        t.references :exam_provider, type: :uuid, foreign_key: true
        t.datetime :consumed_at
        t.datetime :created_at, null: false
      end
      add_index :exam_files, :created_at, where: "consumed_at IS NULL", name: "idx_exam_files_orphans"
      add_check_constraint :exam_files, "content_type::text = ANY (ARRAY['application/pdf'::text, 'image/jpeg'::text, 'image/png'::text])",
                           name: "ck_exam_files_content_type"
      add_check_constraint :exam_files, "byte_size > 0 AND byte_size <= 10485760", name: "ck_exam_files_size"

      create_table :exam_orders, id: :uuid do |t|
        t.references :patient, type: :uuid, null: false, foreign_key: true, index: false
        t.string :sigtap_code, limit: 10, null: false
        t.string :sigtap_competence, limit: 6, null: false
        t.string :priority, null: false, default: "routine"
        t.string :cid10_justification, limit: 4
        t.references :requester_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.text :requester_name
        t.text :requester_cns
        t.references :unit, type: :uuid, null: false, foreign_key: { to_table: :health_units }, index: false
        t.string :origin, null: false
        t.references :consultation_exam_request, type: :uuid, foreign_key: true, index: { unique: true }
        t.references :exam_provider, type: :uuid, foreign_key: true, index: false
        t.datetime :scheduled_at
        t.string :scheduled_place, limit: 200
        t.string :external_reference, limit: 60
        t.references :guide_file, type: :uuid, foreign_key: { to_table: :exam_files }, index: { unique: true }
        t.string :short_code, limit: 10, null: false
        t.string :status, null: false
        t.datetime :status_changed_at, null: false
        t.string :stage
        t.datetime :stage_opened_at
        t.date :stage_due_on
        t.datetime :escalated_at
        t.references :created_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :exam_orders, :short_code, unique: true
      add_index :exam_orders, %i[patient_id created_at], name: "idx_exam_orders_patient_created"
      add_index :exam_orders, %i[unit_id status], name: "idx_exam_orders_unit_status"
      add_index :exam_orders, %i[exam_provider_id status], name: "idx_exam_orders_provider_status"
      add_index :exam_orders, %i[requester_user_id status], name: "idx_exam_orders_requester_status"
      add_index :exam_orders, %i[status stage_due_on], name: "idx_exam_orders_status_due"
      add_check_constraint :exam_orders, "sigtap_code::text ~ '^02[0-9]{8}$'::text", name: "ck_exam_orders_sigtap"
      add_check_constraint :exam_orders, "sigtap_competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text", name: "ck_exam_orders_competence"
      add_check_constraint :exam_orders, "priority::text = ANY (ARRAY['routine'::text, 'urgent'::text])", name: "ck_exam_orders_priority"
      add_check_constraint :exam_orders, "origin::text = ANY (ARRAY['consultation'::text, 'reception_guide'::text, 'pec'::text])", name: "ck_exam_orders_origin"
      add_check_constraint :exam_orders, "status::text = ANY (ARRAY['awaiting_authorization'::text, 'denied'::text, 'awaiting_schedule'::text, 'scheduled'::text, 'collected'::text, 'resulted'::text, 'acknowledged'::text, 'cancelled'::text])",
                           name: "ck_exam_orders_status"
      add_check_constraint :exam_orders, "stage IS NULL OR stage::text = ANY (ARRAY[#{quoted(STAGES)}]::text[])", name: "ck_exam_orders_stage"
      add_check_constraint :exam_orders, "requester_user_id IS NOT NULL OR requester_name IS NOT NULL", name: "ck_exam_orders_requester"
      add_check_constraint :exam_orders, "(origin::text = 'consultation') = (consultation_exam_request_id IS NOT NULL)", name: "ck_exam_orders_consultation"
      add_check_constraint :exam_orders, "origin::text <> 'reception_guide' OR guide_file_id IS NOT NULL", name: "ck_exam_orders_guide"
      add_check_constraint :exam_orders, "short_code::text ~ '^[23456789ABCDEFGHJKMNPQRSTUVWXYZ]{10}$'::text", name: "ck_exam_orders_short_code"
      add_check_constraint :exam_orders, "cid10_justification IS NULL OR cid10_justification::text ~ '^[A-Z][0-9]{2}[0-9X]?$'::text",
                           name: "ck_exam_orders_cid10"

      create_table :exam_order_events, id: :uuid do |t|
        t.references :exam_order, type: :uuid, null: false, foreign_key: true, index: false
        t.string :kind, null: false
        t.references :actor_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.references :actor_provider, type: :uuid, foreign_key: { to_table: :exam_providers }, index: false
        t.text :reason
        t.string :escalation_reason
        t.date :closed_stage_due_on
        t.datetime :occurred_at, null: false
        t.bigint :txid, null: false, default: -> { "txid_current()" }
        t.datetime :created_at, null: false
      end
      add_index :exam_order_events, %i[exam_order_id occurred_at], name: "idx_exam_order_events_order"
      add_index :exam_order_events, %i[kind occurred_at], name: "idx_exam_order_events_kind"
      add_check_constraint :exam_order_events, "kind::text = ANY (ARRAY['requested'::text, 'authorized'::text, 'denied'::text, 'scheduled'::text, 'collected'::text, 'resulted'::text, 'acknowledged'::text, 'escalated'::text, 'cancelled'::text])",
                           name: "ck_exam_order_events_kind"
      add_check_constraint :exam_order_events, "actor_user_id IS NULL OR actor_provider_id IS NULL", name: "ck_exam_order_events_actor"
      add_check_constraint :exam_order_events, "kind::text NOT IN ('denied', 'cancelled') OR reason IS NOT NULL", name: "ck_exam_order_events_reason"
      add_check_constraint :exam_order_events, "(kind::text = 'escalated') = (escalation_reason IS NOT NULL)", name: "ck_exam_order_events_escalation"
      add_check_constraint :exam_order_events, "escalation_reason IS NULL OR escalation_reason::text = ANY (ARRAY['overdue'::text, 'critical'::text, 'unlinked_requester'::text])",
                           name: "ck_exam_order_events_escalation_reason"

      create_table :exam_results, id: :uuid do |t|
        t.references :exam_order, type: :uuid, foreign_key: true, index: { unique: true }
        t.references :patient, type: :uuid, null: false, foreign_key: true, index: false
        t.string :sigtap_code, limit: 10, null: false
        t.string :origin, null: false
        t.string :source, null: false
        t.references :pdf_file, type: :uuid, null: false, foreign_key: { to_table: :exam_files }, index: { unique: true }
        t.text :values, null: false
        t.date :released_on, null: false
        t.string :laboratory_name, limit: 160, null: false
        t.boolean :critical, null: false, default: false
        t.string :fhir_identifier, limit: 128
        t.references :exam_provider, type: :uuid, foreign_key: true, index: false
        t.references :registered_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.references :registered_by_provider, type: :uuid, foreign_key: { to_table: :exam_providers }, index: false
        t.references :consultation, type: :uuid, foreign_key: true, index: false
        t.datetime :created_at, null: false
      end
      add_index :exam_results, %i[patient_id created_at], name: "idx_exam_results_patient_created"
      add_index :exam_results, %i[exam_provider_id fhir_identifier], unique: true, where: "fhir_identifier IS NOT NULL",
                                                                      name: "idx_exam_results_fhir_identifier"
      add_check_constraint :exam_results, "origin::text = ANY (ARRAY['order'::text, 'external'::text])", name: "ck_exam_results_origin"
      add_check_constraint :exam_results, "source::text = ANY (ARRAY['unit'::text, 'provider'::text, 'fhir'::text, 'consultation'::text])", name: "ck_exam_results_source"
      add_check_constraint :exam_results, "(origin::text = 'order') = (exam_order_id IS NOT NULL)", name: "ck_exam_results_order"
      add_check_constraint :exam_results, "(fhir_identifier IS NULL) = (source::text <> 'fhir')", name: "ck_exam_results_fhir"
      add_check_constraint :exam_results, "registered_by_user_id IS NULL OR registered_by_provider_id IS NULL", name: "ck_exam_results_registrant"
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end

    private

    def quoted(list) = list.map { |value| "'#{value}'" }.join(", ")

    def days_check = STAGES.map { |stage| "(#{stage}_days IS NULL OR (#{stage}_days BETWEEN 1 AND 365))" }.join(" AND ")
  end
  ```

  Rode a migração nos bancos de teste da branch (recrie-os como na seção "Ambiente") e **escreva à mão** o dump em `db/city_schema.rb`: `define(version: 2026_10_13_900001)`; os blocos `create_table` novos em ordem alfabética (`exam_catalog_items`, `exam_files`, `exam_order_events`, `exam_orders`, `exam_provider_items`, `exam_provider_users`, `exam_providers`, `exam_results`) com colunas em ordem alfabética, índices e `check_constraint` como o `pg_dump` os devolve (compare com `bin/rails runner 'puts ActiveRecord::SchemaDumper.dump'` no banco de teste — `spec/services/city_schema_spec.rb` confere a paridade); `priority` em `consultation_exam_requests`; `exam_provider_id` e o índice em `invitations`; o `ck_memberships_role` novo; as `add_foreign_key` novas em ordem.

- [ ] **Step 4: Triggers** — acrescente ao fim de `db/city_triggers.sql`:

  ```sql
  -- Módulo 21a (ADR 0036, Invariantes; Desvio 6): o pedido de exame. Nasce
  -- aguardando; identidade fixa; o cache da situação (status, etapa, prazo,
  -- prestador, agenda, escalonamento) só muda com um evento do MESMO pedido
  -- gravado na MESMA transação; nunca some. Re-cifra: só as colunas cifradas.
  CREATE OR REPLACE FUNCTION rota_exam_order_guard() RETURNS trigger AS $fn$
  DECLARE
    encrypted text[] := ARRAY['requester_name', 'requester_cns'];
    identity text[] := ARRAY['id', 'patient_id', 'sigtap_code', 'sigtap_competence', 'priority', 'cid10_justification',
                             'requester_user_id', 'requester_name', 'requester_cns', 'unit_id', 'origin',
                             'consultation_exam_request_id', 'guide_file_id', 'short_code', 'created_by_user_id', 'created_at'];
  BEGIN
    IF TG_OP = 'INSERT' THEN
      IF NEW.status NOT IN ('awaiting_authorization', 'awaiting_schedule') THEN
        RAISE EXCEPTION 'exam_orders: an order is born awaiting authorization or schedule';
      END IF;
      RETURN NEW;
    END IF;
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'exam_orders is append-only: DELETE refused';
    END IF;
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - encrypted) = (to_jsonb(OLD) - encrypted) THEN
      RETURN NEW;
    END IF;
    IF (SELECT jsonb_object_agg(k, v) FROM jsonb_each(to_jsonb(NEW)) AS e(k, v) WHERE k = ANY(identity))
       IS DISTINCT FROM (SELECT jsonb_object_agg(k, v) FROM jsonb_each(to_jsonb(OLD)) AS e(k, v) WHERE k = ANY(identity)) THEN
      RAISE EXCEPTION 'exam_orders: identity never changes';
    END IF;
    IF NOT EXISTS (SELECT 1 FROM exam_order_events e WHERE e.exam_order_id = NEW.id AND e.txid = txid_current()) THEN
      RAISE EXCEPTION 'exam_orders: status changes only with an event of the same transaction';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  -- Arquivo do exame (guia, laudo): bytes cifrados nunca mudam (salvo a
  -- re-cifra); consumed_at só nasce uma vez; só o arquivo nunca usado pode
  -- ser apagado (purga dos órfãos de 24 h).
  CREATE OR REPLACE FUNCTION rota_exam_file_guard() RETURNS trigger AS $fn$
  BEGIN
    IF TG_OP = 'DELETE' THEN
      IF OLD.consumed_at IS NOT NULL THEN
        RAISE EXCEPTION 'exam_files: a consumed file is never deleted';
      END IF;
      RETURN OLD;
    END IF;
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - 'data') = (to_jsonb(OLD) - 'data') THEN
      RETURN NEW;
    END IF;
    IF OLD.consumed_at IS NULL AND NEW.consumed_at IS NOT NULL
       AND (to_jsonb(NEW) - 'consumed_at') = (to_jsonb(OLD) - 'consumed_at') THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'exam_files: a file never changes (only consumed once)';
  END;
  $fn$ LANGUAGE plpgsql;

  DO $do$
  DECLARE
    item text;
  BEGIN
    IF to_regclass('public.exam_orders') IS NOT NULL THEN
      EXECUTE 'DROP TRIGGER IF EXISTS exam_orders_guard ON exam_orders';
      EXECUTE 'CREATE TRIGGER exam_orders_guard
        BEFORE INSERT OR UPDATE OR DELETE ON exam_orders
        FOR EACH ROW EXECUTE FUNCTION rota_exam_order_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS exam_files_guard ON exam_files';
      EXECUTE 'CREATE TRIGGER exam_files_guard
        BEFORE UPDATE OR DELETE ON exam_files
        FOR EACH ROW EXECUTE FUNCTION rota_exam_file_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS exam_order_events_reencryption_only ON exam_order_events';
      EXECUTE 'CREATE TRIGGER exam_order_events_reencryption_only
        BEFORE UPDATE OR DELETE ON exam_order_events
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''reason'')';
      EXECUTE 'DROP TRIGGER IF EXISTS exam_results_reencryption_only ON exam_results';
      EXECUTE 'CREATE TRIGGER exam_results_reencryption_only
        BEFORE UPDATE OR DELETE ON exam_results
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''values'')';
      FOREACH item IN ARRAY ARRAY['exam_orders', 'exam_order_events', 'exam_results', 'exam_files'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only_truncate', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE TRUNCATE ON %I FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()',
                       item || '_append_only_truncate', item);
      END LOOP;
    END IF;
  END
  $do$;
  ```

  As mensagens têm de casar com a spec do Step 1 (`append-only`, `identity never changes`, `status changes only with an event`, `born awaiting`, `DELETE refused`, `consumed`). Confira como a limpeza da suíte (`spec/support/committed_rows_cleanup.rb`) trata tabelas com trigger de TRUNCATE (`exam_*` precisa entrar na mesma lista das outras tabelas só de acréscimo, se houver uma).

- [ ] **Step 5: Modelos.** `app/models/exam_provider.rb`:

  ```ruby
  # Prestador de exames da cidade (ADR 0036): laboratório, consórcio, LACEN,
  # clínica de imagem. Token FHIR guardado só como SHA-256; revogável.
  # `simulated` só a semente de dev grava (o laboratório simulado).
  class ExamProvider < ApplicationRecord
    KINDS = %w[municipal accredited consortium lacen imaging].freeze

    has_many :items, class_name: "ExamProviderItem", dependent: :restrict_with_error
    has_many :provider_users, class_name: "ExamProviderUser", dependent: :restrict_with_error

    validates :name, presence: true, length: { maximum: 120 }
    validates :kind, inclusion: { in: KINDS }
    validates :cnes, format: { with: /\A\d{7}\z/ }, allow_nil: true

    def self.digest(token) = Digest::SHA256.hexdigest(token.to_s)

    def self.by_token(token)
      return nil unless token.is_a?(String) && token.length.between?(32, 128)

      find_by(fhir_token_digest: digest(token), fhir_token_revoked_at: nil, active: true)
    end

    def fhir_token_active? = fhir_token_digest.present? && fhir_token_revoked_at.nil?
    def inspect = "#<ExamProvider id=#{id} kind=#{kind} active=#{active}>"
  end
  ```

  `app/models/exam_catalog_item.rb`:

  ```ruby
  # Exame no catálogo da cidade (ADR 0036): SIGTAP (grupo 02), exige
  # autorização, política de liberação ao cidadão, prazos por etapa (dias;
  # nil = padrão) e os campos do resultado (chave, rótulo, unidade, LOINC).
  class ExamCatalogItem < ApplicationRecord
    POLICIES = %w[immediate after_acknowledgement never_citizen].freeze
    STAGES = %w[authorize schedule collect result acknowledge].freeze

    validates :release_policy, inclusion: { in: POLICIES }
    validates :sigtap_code, format: { with: /\A02\d{8}\z/ }

    def days_for(stage) = public_send(:"#{stage}_days")
    def result_field(key) = result_fields.find { |field| field["key"] == key }
    def result_field_by_loinc(code) = result_fields.find { |field| field["loinc"].present? && field["loinc"] == code }
  end
  ```

  `app/models/exam_provider_item.rb`:

  ```ruby
  # Exame que um prestador faz, com prazos próprios (sobrepõem os do catálogo).
  class ExamProviderItem < ApplicationRecord
    belongs_to :exam_provider
    belongs_to :exam_catalog_item

    def days_for(stage) = public_send(:"#{stage}_days")
  end
  ```

  `app/models/exam_provider_user.rb`:

  ```ruby
  # Usuário do portal do prestador (papel exam_provider): um prestador por usuário.
  class ExamProviderUser < ApplicationRecord
    belongs_to :exam_provider
    belongs_to :user
  end
  ```

  `app/models/exam_file.rb`:

  ```ruby
  # Guia ou laudo (ADR 0036): bytes em Base64 cifrados com a chave da cidade
  # (como Signature#signed_pdf). Nunca mudam; consumed_at quando ligado a um
  # pedido ou resultado; o órfão de 24 h é apagado pelo job diário.
  class ExamFile < ApplicationRecord
    encrypts :data

    belongs_to :uploaded_by_user, class_name: "User", optional: true
    belongs_to :exam_provider, optional: true

    def bytes = Base64.strict_decode64(data)
    def consumed? = consumed_at.present?
    def inspect = "#<ExamFile id=#{id} content_type=#{content_type} byte_size=#{byte_size} consumed=#{consumed?}>"
  end
  ```

  `app/models/exam_order.rb`:

  ```ruby
  # O exame (ADR 0036): identidade fixa e um cache da situação que só muda com
  # um evento da trilha na mesma transação (trigger exam_orders_guard). Use
  # Exams::Transition/Exams::Escalate — nunca update! direto.
  class ExamOrder < ApplicationRecord
    STATUSES = %w[awaiting_authorization denied awaiting_schedule scheduled collected resulted acknowledged cancelled].freeze
    ORIGINS = %w[consultation reception_guide pec].freeze
    PRIORITIES = %w[routine urgent].freeze
    OPEN_STATUSES = %w[awaiting_authorization awaiting_schedule scheduled collected resulted].freeze

    encrypts :requester_name
    encrypts :requester_cns

    belongs_to :patient
    belongs_to :unit, class_name: "HealthUnit"
    belongs_to :requester_user, class_name: "User", optional: true
    belongs_to :exam_provider, optional: true
    belongs_to :consultation_exam_request, optional: true
    belongs_to :guide_file, class_name: "ExamFile", optional: true
    belongs_to :created_by_user, class_name: "User", optional: true
    has_many :events, -> { order(:occurred_at, :created_at, :id) }, class_name: "ExamOrderEvent", dependent: :restrict_with_exception
    has_one :result, class_name: "ExamResult", dependent: :restrict_with_exception

    def catalog_item = ExamCatalogItem.find_by(sigtap_code: sigtap_code)
    def escalated? = escalated_at.present?
    def requester_display_name = requester_user ? Screenings::Json.staff_name(requester_user) : requester_name

    # Atraso pelo prazo gravado na transição, em dias corridos no fuso da cidade.
    def overdue(today: Time.zone.today)
      return nil unless stage && stage_due_on && OPEN_STATUSES.include?(status) && today > stage_due_on

      { stage: stage, due_on: stage_due_on, days_late: (today - stage_due_on).to_i }
    end

    def inspect = "#<ExamOrder id=#{id} status=#{status} stage=#{stage} origin=#{origin}>"
  end
  ```

  `app/models/exam_order_event.rb`:

  ```ruby
  # Trilha do exame (ADR 0036): só acréscimo (trigger). O motivo é cifrado e
  # nunca sai em evento de domínio nem em log.
  class ExamOrderEvent < ApplicationRecord
    KINDS = %w[requested authorized denied scheduled collected resulted acknowledged escalated cancelled].freeze
    ESCALATION_REASONS = %w[overdue critical unlinked_requester].freeze

    encrypts :reason

    belongs_to :exam_order
    belongs_to :actor_user, class_name: "User", optional: true
    belongs_to :actor_provider, class_name: "ExamProvider", optional: true

    validates :kind, inclusion: { in: KINDS }

    def inspect = "#<ExamOrderEvent id=#{id} kind=#{kind} exam_order_id=#{exam_order_id}>"
  end
  ```

  `app/models/exam_result.rb`:

  ```ruby
  # Resultado (ADR 0036): imutável (trigger); valores em JSON cifrado
  # (`[{ key|null, label, value, unit|null, loinc|null }]`); PDF em exam_files.
  class ExamResult < ApplicationRecord
    encrypts :values

    belongs_to :exam_order, optional: true
    belongs_to :patient
    belongs_to :pdf_file, class_name: "ExamFile"
    belongs_to :exam_provider, optional: true
    belongs_to :registered_by_user, class_name: "User", optional: true
    belongs_to :registered_by_provider, class_name: "ExamProvider", optional: true
    belongs_to :consultation, optional: true

    def values_list = JSON.parse(values)
    def external? = origin == "external"
    def inspect = "#<ExamResult id=#{id} origin=#{origin} source=#{source} critical=#{critical}>"
  end
  ```

  Em `app/models/membership.rb`: `ROLES` ganha `exam_provider exam_regulator` (em ordem alfabética, espelhando o CHECK) e `PRIVILEGED_ROLES` ganha os dois, com o comentário "ADR 0036: veem o nome do paciente junto com o exame — step-up; o mantenedor não concede". Em `app/models/invitation.rb`: `belongs_to :exam_provider, optional: true`. Em `app/models/consultation_exam_request.rb`: `PRIORITIES = %w[routine urgent].freeze` e `validates :priority, inclusion: { in: PRIORITIES }`.

- [ ] **Step 6: Cifra, eventos e filtros.** `CityEncryption::CITY_KEYED_TARGETS` ganha:

  ```ruby
      # ADR 0036: rastreamento do exame.
      [ ExamOrder,      :requester_name ],
      [ ExamOrder,      :requester_cns ],
      [ ExamOrderEvent, :reason ],
      [ ExamResult,     :values ],
      [ ExamFile,       :data ]
  ```

  `config/initializers/domain_events.rb`, depois do bloco do módulo 19d:

  ```ruby
    # Módulo 21a (ADR 0036; contrato §12): só ids — nunca valor de resultado,
    # nome do exame com o paciente, CPF, motivo ou token.
    %w[requested authorized denied scheduled collected resulted acknowledged escalated cancelled].each do |kind|
      DomainEvents.bind "exam_order.#{kind}", to: []
    end
    DomainEvents.bind "exam_result.external_recorded", to: []
    DomainEvents.bind "exam_provider.fhir_token_rotated", to: []
    DomainEvents.bind "exam_provider.fhir_token_revoked", to: []
  ```

  `spec/initializers/domain_events_bindings_spec.rb` ganha:

  ```ruby
  RSpec.describe "exam event bindings (ADR 0036)" do
    it "declares every exam event with no consumer" do
      names = %w[requested authorized denied scheduled collected resulted acknowledged escalated cancelled].map { |k| "exam_order.#{k}" } +
              %w[exam_result.external_recorded exam_provider.fhir_token_rotated exam_provider.fhir_token_revoked]
      expect(DomainEvents.registry.keys).to include(*names)
      expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
    end
  end
  ```

  `config/initializers/filter_parameter_logging.rb`, no fim da lista:

  ```ruby
    # ADR 0036: o arquivo em Base64, os valores do resultado, o solicitante
    # digitado (nome, CNS), o local da coleta e a referência externa da guia.
    /\A(data|values|requester|requester_name|requester_cns|scheduled_place|external_reference)\z/
  ```

- [ ] **Step 7: Helpers** `spec/support/exam_helpers.rb`:

  ```ruby
  # Módulo 21a (ADR 0036): cidade com o rastreamento ligado, catálogo,
  # prestadores, paciente (par validado → Patients::Resolve), arquivo PDF e um
  # pedido aberto pelo caminho real (Exams::Open, Task 7) — até a Task 7 existir,
  # exam_order! grava direto.
  module ExamHelpers
    PDF_BYTES = "%PDF-1.4\n%\xE2\xE3\xCF\xD3\n1 0 obj<<>>endobj\ntrailer<</Root 1 0 R>>\n%%EOF\n".b.freeze
    FAKE_TOKEN = "token-de-teste-do-laboratorio-0123456789abcdef".freeze

    def exam_city!(record_mode: "record", fhir: false, fake_lab: false, pec: false)
      city = clinical_city!(record_mode: record_mode, enabled: record_mode == "record")
      maintainer = ledi_maintainer!
      Platform::Features.set!(city: city, key: "exam_tracking", enabled: true, maintainer: maintainer)
      Platform::Features.set!(city: city, key: "exam_fhir_inbound", enabled: true, maintainer: maintainer) if fhir || fake_lab
      Platform::Features.set!(city: city, key: "fake_lab", enabled: true, maintainer: maintainer) if fake_lab
      if pec
        Platform::Features.set!(city: city, key: "patient_timeline", enabled: true, maintainer: maintainer)
        Platform::Features.set!(city: city, key: "pec_external_view", enabled: true, maintainer: maintainer)
        Platform::Features.set!(city: city, key: "exam_pec_bridge", enabled: true, maintainer: maintainer)
      end
      sigtap_release!
      CityCatalog.reset_cache!
      city
    end

    def exam_catalog_item!(sigtap_code: "0202010503", **attrs)
      ExamCatalogItem.find_by(sigtap_code: sigtap_code) ||
        ExamCatalogItem.create!({ sigtap_code: sigtap_code,
                                  result_fields: [ { "key" => "hba1c", "label" => "Hemoglobina glicada", "unit" => "%", "loinc" => "4548-4" } ] }
                                .merge(attrs))
    end

    # items: :all habilita o prestador para todo exame JÁ no catálogo (crie o
    # item antes); items: [] = nenhum (para provar provider_not_enabled_for_exam).
    def exam_provider!(name: "Laboratório Central", token: nil, items: :all, **attrs)
      provider = ExamProvider.create!({ name: name, kind: "municipal" }.merge(attrs))
      provider.update!(fhir_token_digest: ExamProvider.digest(token), fhir_token_rotated_at: Time.current) if token
      (items == :all ? ExamCatalogItem.all : items).each do |item|
        ExamProviderItem.find_or_create_by!(exam_provider: provider, exam_catalog_item: item)
      end
      provider
    end

    # Número único por exemplo para exam_patient! (CPF e telefone saem dele).
    def exam_seq! = (@exam_seq = (@exam_seq || 500) + 1)

    # [paciente, par validado]: o caminho real do paciente (Patients::Resolve).
    def exam_patient!(n, **citizen_attrs)
      citizen = verified_citizen!(n, **citizen_attrs)
      [ Patients::Resolve.call(citizen).payload.fetch(:patient), citizen ]
    end

    # Equipe ativa da unidade com os usuários (vínculo ativo nela), INE aleatório
    # (exportable_unit! usa um INE fixo e não serve para duas unidades).
    def exam_team!(unit, *users)
      team = HealthTeam.find_by(health_unit: unit) ||
             HealthTeam.create!(health_unit: unit, ine: format("%010d", SecureRandom.random_number(10**10)), kind: "70",
                                name: "ESF #{unit.name}")
      users.each do |user|
        link = user.professional.links.active.find_by!(health_unit: unit)
        HealthTeamMember.create!(professional: user.professional, health_team: team, cbo_code: link.cbo_code,
                                 started_on: Time.zone.today - 30)
      end
      team
    end

    def pdf_file!(user: nil, provider: nil, bytes: PDF_BYTES, content_type: "application/pdf")
      ExamFile.create!(content_type: content_type, byte_size: bytes.bytesize, sha256: Digest::SHA256.hexdigest(bytes),
                       data: Base64.strict_encode64(bytes), uploaded_by_user: user, exam_provider: provider)
    end

    def exam_order!(patient:, unit:, requester:, sigtap_code: "0202010503", priority: "routine", provider: nil, status: :awaiting_schedule)
      if defined?(Exams::Open) && status == :awaiting_schedule && provider.nil?
        exam_catalog_item!(sigtap_code: sigtap_code)
        result = Exams::Open.call(patient: patient, sigtap_code: sigtap_code, priority: priority, unit: unit, origin: "pec",
                                  requester_user: requester, created_by: requester)
        raise "pedido recusado: #{result.reason}" if result.failure?

        return result.payload.fetch(:order)
      end
      ApplicationRecord.transaction do
        order = ExamOrder.create!(patient: patient, sigtap_code: sigtap_code, sigtap_competence: Time.zone.today.strftime("%Y%m"),
                                  priority: priority, requester_user: requester, unit: unit, origin: "pec",
                                  short_code: ClinicalDocuments::Codes.short_code, status: status.to_s,
                                  status_changed_at: Time.current, exam_provider: provider)
        ExamOrderEvent.create!(exam_order: order, kind: "requested", actor_user: requester, occurred_at: Time.current)
        order
      end
    end
  end

  RSpec.configure { |c| c.include ExamHelpers }
  ```

- [ ] **Step 8: Run** `rspec spec/models/exam_tracking_schema_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/requests/setup_spec.rb spec/commands`. Expected: PASS (a spec do `setup` cobre o step-up dos papéis privilegiados; se ela fixar a lista de `PRIVILEGED_ROLES`, acrescente os dois — é o contrato).

- [ ] **Step 9: Commit**

  ```bash
  git add db/city_migrate/20261013900001_add_exam_tracking.rb db/city_schema.rb db/city_triggers.sql app/models/exam_provider.rb app/models/exam_catalog_item.rb app/models/exam_provider_item.rb app/models/exam_provider_user.rb app/models/exam_file.rb app/models/exam_order.rb app/models/exam_order_event.rb app/models/exam_result.rb app/models/membership.rb app/models/invitation.rb app/models/consultation_exam_request.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/exam_helpers.rb spec/models/exam_tracking_schema_spec.rb
  git commit -m "feat: add exam tracking city tables, guards and models

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 2 — Catálogo, prestadores e arquivos (F-21.2)

### Task 4: Prazos e catálogo de exames da cidade

**Files:**
- Create: `app/services/exams/deadlines.rb`, `app/services/exams/catalog_input.rb`, `app/services/exams/admin_json.rb`
- Create: `app/controllers/exams/base_controller.rb`, `app/controllers/exams/admin/base_controller.rb`, `app/controllers/exams/admin/catalog_items_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/exams/deadlines_spec.rb`, `spec/requests/exams/admin/catalog_items_spec.rb`

**Interfaces:**
- Consumes: `ExamCatalogItem`, `ExamProviderItem` (Task 3), `ClinicalTerms::SigtapExams.find(code, on:)`, `.label(code, competence)`.
- Produces: `Exams::Deadlines::PLATFORM`, `STAGES`, `.days(stage:, sigtap_code:, provider_id:, priority:) -> Integer`, `.due_on(stage:, order:, opened_at:, provider_id: order.exam_provider_id) -> Date`; `Exams::CatalogInput.call(params, item: nil) -> Result(attrs:)` (falhas `sigtap_unknown`, `duplicate_sigtap`, `invalid_deadline`, `invalid_policy`, `invalid_result_fields`); `Exams::AdminJson.catalog_item(item)`, `.deadlines(record)`; `Exams::BaseController` (`require_exam_tracking!`, `render_failure(result)`, `forbid`, `not_found`, `uuid?`, `limit_param`, `FAILURE_STATUS`), `Exams::Admin::BaseController` (municipal_admin; escrita com step-up).

- [ ] **Step 1: Write the failing tests.** `spec/services/exams/deadlines_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0036 (spec §5; contrato §1): prestador × exame → catálogo → plataforma;
  # urgente = metade arredondada para cima; prazo em dias corridos no fuso da cidade.
  RSpec.describe Exams::Deadlines do
    before { exam_city! }

    let(:item) { exam_catalog_item!(result_days: 20, collect_days: nil) }
    let(:provider) { exam_provider! }

    it "resolve na ordem prestador → catálogo → plataforma, e urgente usa a metade para cima" do
      ExamProviderItem.find_or_initialize_by(exam_provider: provider, exam_catalog_item: item).update!(result_days: 7)
      args = { sigtap_code: item.sigtap_code, provider_id: provider.id }
      expect(described_class.days(stage: "result", **args, priority: "routine")).to eq(7)
      expect(described_class.days(stage: "result", **args, priority: "urgent")).to eq(4)
      expect(described_class.days(stage: "result", sigtap_code: item.sigtap_code, provider_id: nil, priority: "routine")).to eq(20)
      expect(described_class.days(stage: "collect", **args, priority: "routine")).to eq(30)
      expect(described_class.days(stage: "authorize", **args, priority: "urgent")).to eq(8)
      expect(described_class.days(stage: "acknowledge", sigtap_code: "0202020380", provider_id: nil, priority: "routine")).to eq(15)
      expect { described_class.days(stage: "bogus", **args, priority: "routine") }.to raise_error(ArgumentError)
    end

    it "conta no fuso da cidade: aberta às 23h59 de São Paulo vence no dia certo (Review Focus 5)" do
      order = Struct.new(:sigtap_code, :priority, :exam_provider_id).new(item.sigtap_code, "routine", nil)
      opened = Time.find_zone("America/Sao_Paulo").local(2026, 10, 13, 23, 59)
      Time.use_zone("America/Sao_Paulo") do
        expect(described_class.due_on(stage: "acknowledge", order: order, opened_at: opened.utc)).to eq(Date.new(2026, 10, 28))
      end
    end
  end
  ```

  `spec/requests/exams/admin/catalog_items_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5: catálogo da cidade, municipal_admin, escrita com step-up.
  RSpec.describe "Catálogo de exames (admin)", type: :request do
    before { exam_city! }

    let(:admin) { municipal_admin! }
    let(:body) do
      { sigtap_code: "0202010503", requires_authorization: false, release_policy: "immediate",
        deadlines: { "result" => 10 }, active: true,
        result_fields: [ { key: "hba1c", label: "Hemoglobina glicada", unit: "%", loinc: "4548-4" } ] }
    end

    it "cria, lista e altera; sem step-up 401; sem papel 403; interruptor desligado 403" do
      sign_in_as(step_up_enrolled!(admin))
      json_post "/exams/admin/catalog_items", body
      expect([ response.status, json_body["error"] ]).to eq([ 401, "mfa_required" ])
      stepped_up!(municipal_admin!)
      json_post "/exams/admin/catalog_items", body
      expect(response).to have_http_status(:created)
      created = json_body
      expect(created).to include("sigtap_code" => "0202010503", "label" => "DOSAGEM DE HEMOGLOBINA GLICOSILADA",
                                 "release_policy" => "immediate", "requires_authorization" => false, "active" => true)
      expect(created["deadlines"]).to eq("authorize" => nil, "schedule" => nil, "collect" => nil, "result" => 10, "acknowledge" => nil)
      json_post "/exams/admin/catalog_items", body
      expect([ response.status, json_body["error"] ]).to eq([ 422, "duplicate_sigtap" ])
      patch "/exams/admin/catalog_items/#{created['id']}", params: { release_policy: "never_citizen" }.to_json,
                                                         headers: { "CONTENT_TYPE" => "application/json" }
      expect(json_body["release_policy"]).to eq("never_citizen")
      get "/exams/admin/catalog_items"
      expect(json_body["items"].map { |i| i["sigtap_code"] }).to eq(%w[0202010503])

      sign_in_as(staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier"))
      get "/exams/admin/catalog_items"
      expect([ response.status, json_body["error"] ]).to eq([ 403, "missing_role" ])
      Platform::Features.set!(city: Current.city, key: "exam_tracking", enabled: false, maintainer: ledi_maintainer!)
      get "/exams/admin/catalog_items"
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "exam_tracking")
    end

    it "recusa SIGTAP fora da competência vigente, prazo fora de 1..365, política e campos ruins" do
      stepped_up!(admin)
      { { sigtap_code: "0202999999" } => "sigtap_unknown", { sigtap_code: "0301010064" } => "sigtap_unknown",
        { deadlines: { "result" => 0 } } => "invalid_deadline", { deadlines: { "lunch" => 3 } } => "invalid_deadline",
        { release_policy: "sometimes" } => "invalid_policy",
        { result_fields: [ { key: "Bad Key", label: "x" } ] } => "invalid_result_fields",
        { result_fields: [ { key: "a", label: "x" }, { key: "a", label: "y" } ] } => "invalid_result_fields" }.each do |over, error|
        json_post "/exams/admin/catalog_items", body.merge(over)
        expect([ response.status, json_body["error"] ]).to eq([ 422, error ]), over.inspect
      end
    end
  end
  ```

  (`json_post` vem de `CitizenRequestHelpers`, incluído em todo request spec; `municipal_admin!`, `stepped_up!`, `step_up_enrolled!`, `json_body` de `ClinicalDocumentHelpers`.)

- [ ] **Step 2: Run** `rspec spec/services/exams/deadlines_spec.rb spec/requests/exams/admin/catalog_items_spec.rb`. Expected: FAIL (`Exams::Deadlines` indefinido; rota inexistente).

- [ ] **Step 3: Implemente.** `app/services/exams/deadlines.rb`:

  ```ruby
  # Prazo por etapa (ADR 0036; spec §5): prestador × exame → catálogo →
  # padrão da plataforma; urgente = metade arredondada para cima. Dias
  # corridos no fuso da cidade (Time.zone é o da cidade dentro de
  # CityConnection.with).
  module Exams
    module Deadlines
      PLATFORM = { "authorize" => 15, "schedule" => 15, "collect" => 30, "result" => 30, "acknowledge" => 15 }.freeze
      STAGES = PLATFORM.keys.freeze

      module_function

      def days(stage:, sigtap_code:, provider_id:, priority:)
        raise ArgumentError, "etapa desconhecida: #{stage.inspect}" unless PLATFORM.key?(stage)

        item = ExamCatalogItem.find_by(sigtap_code: sigtap_code)
        own = provider_id && item && ExamProviderItem.find_by(exam_provider_id: provider_id, exam_catalog_item_id: item.id)
        base = own&.days_for(stage) || item&.days_for(stage) || PLATFORM.fetch(stage)
        priority == "urgent" ? (base / 2.0).ceil : base
      end

      def due_on(stage:, order:, opened_at:, provider_id: order.exam_provider_id)
        opened_at.in_time_zone.to_date + days(stage: stage, sigtap_code: order.sigtap_code, provider_id: provider_id,
                                              priority: order.priority)
      end
    end
  end
  ```

  `app/services/exams/catalog_input.rb`:

  ```ruby
  # Corpo do item do catálogo (contrato §5) → atributos. Só as chaves
  # presentes mudam (PATCH). Nunca levanta.
  module Exams
    module CatalogInput
      KEY = /\A[a-z][a-z0-9_]{0,39}\z/
      LOINC = /\A\d{1,7}-\d\z/

      module_function

      def call(params, item: nil, on: Time.zone.today)
        params = params.respond_to?(:to_unsafe_h) ? params.to_unsafe_h : params.to_h
        params = params.deep_stringify_keys
        attrs = {}

        if params.key?("sigtap_code") || item.nil?
          exam = ClinicalTerms::SigtapExams.find(params["sigtap_code"], on: on)
          return Result.fail(:sigtap_unknown) unless exam
          return Result.fail(:duplicate_sigtap) if ExamCatalogItem.where(sigtap_code: exam.code).where.not(id: item&.id).exists?

          attrs[:sigtap_code] = exam.code
        end
        if params.key?("release_policy")
          return Result.fail(:invalid_policy) unless ExamCatalogItem::POLICIES.include?(params["release_policy"])

          attrs[:release_policy] = params["release_policy"]
        end
        %w[requires_authorization active].each do |flag|
          next unless params.key?(flag)
          return Result.fail(:"invalid_#{flag}") unless [ true, false ].include?(params[flag])

          attrs[flag.to_sym] = params[flag]
        end
        if params.key?("deadlines")
          days = deadlines(params["deadlines"])
          return Result.fail(:invalid_deadline) unless days

          attrs.merge!(days)
        end
        if params.key?("result_fields")
          fields = result_fields(params["result_fields"])
          return Result.fail(:invalid_result_fields) unless fields

          attrs[:result_fields] = fields
        end
        Result.ok(attrs: attrs)
      end

      # { stage => int 1..365 | nil } → { "<stage>_days" => ... }; chave desconhecida = nil.
      def deadlines(raw)
        return nil unless raw.is_a?(Hash)
        return nil unless (raw.keys - Deadlines::STAGES).empty?

        raw.each_with_object({}) do |(stage, value), acc|
          return nil unless value.nil? || (value.is_a?(Integer) && value.between?(1, 365))

          acc[:"#{stage}_days"] = value
        end
      end

      def result_fields(raw)
        return nil unless raw.is_a?(Array) && raw.size <= 40

        fields = raw.map do |field|
          return nil unless field.is_a?(Hash)

          field = field.deep_stringify_keys
          key, label, unit, loinc = field.values_at("key", "label", "unit", "loinc")
          return nil unless key.is_a?(String) && key.match?(KEY) && label.is_a?(String) && label.strip.length.between?(1, 80)
          return nil unless unit.nil? || (unit.is_a?(String) && unit.length <= 20)
          return nil unless loinc.nil? || (loinc.is_a?(String) && loinc.match?(LOINC))

          { "key" => key, "label" => label.strip, "unit" => unit.presence, "loinc" => loinc }
        end
        fields.map { |f| f["key"] }.uniq.size == fields.size ? fields : nil
      end
      private_class_method :deadlines, :result_fields
    end
  end
  ```

  `app/services/exams/admin_json.rb`:

  ```ruby
  # Formas do contrato §5 (catálogo e prestadores).
  module Exams
    module AdminJson
      module_function

      def deadlines(record) = Deadlines::STAGES.to_h { |stage| [ stage, record.days_for(stage) ] }

      def catalog_item(item)
        { id: item.id, sigtap_code: item.sigtap_code, label: label(item.sigtap_code),
          requires_authorization: item.requires_authorization, release_policy: item.release_policy,
          deadlines: deadlines(item), result_fields: item.result_fields, active: item.active }
      end

      def provider(provider)
        { id: provider.id, name: provider.name, cnes: provider.cnes, kind: provider.kind, active: provider.active,
          simulated: provider.simulated, has_fhir_token: provider.fhir_token_active?,
          fhir_token_rotated_at: provider.fhir_token_rotated_at&.iso8601,
          items: provider.items.includes(:exam_catalog_item).map do |pi|
            { catalog_item_id: pi.exam_catalog_item_id, sigtap_code: pi.exam_catalog_item.sigtap_code, deadlines: deadlines(pi) }
          end,
          users: provider.provider_users.includes(:user).map { |pu| { user_id: pu.user_id, email: pu.user.email_address } } }
      end

      def label(code) = ClinicalTerms::SigtapExams.find(code, on: Time.zone.today)&.label || "Procedimento #{code}"
    end
  end
  ```

  `app/controllers/exams/base_controller.rb`:

  ```ruby
  # Base das rotas /exams/* (ADR 0036; contrato §3–§6): sessão do dashboard,
  # interruptor exam_tracking utilizável e a tradução de Result para JSON.
  module Exams
    class BaseController < ApplicationController
      include Authentication
      include MfaStepUp

      UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
      FAILURE_STATUS = {
        invalid_transition: :conflict, citizen_not_verified: :conflict, already_acknowledged: :conflict,
        not_found: :not_found,
        missing_role: :forbidden, not_requester: :forbidden, not_in_team: :forbidden, not_unit_staff: :forbidden,
        out_of_context: :forbidden, opening_required: :forbidden
      }.freeze

      before_action :require_exam_tracking!

      private

      def require_exam_tracking!
        return if Exams::Gate.usable?(Current.city)

        render json: { error: "feature_disabled", feature: Exams::Gate::KEY }, status: :forbidden
      end

      def render_failure(result)
        payload = { error: result.reason.to_s }
        payload[:status] = result.details[:status] if result.reason == :invalid_transition
        payload[:details] = result.details[:paths] if result.details[:paths]
        render json: payload, status: FAILURE_STATUS.fetch(result.reason, :unprocessable_entity)
      end

      def forbid(reason = "forbidden") = render(json: { error: reason }, status: :forbidden)
      def not_found = render(json: { error: "not_found" }, status: :not_found)
      def uuid?(value) = value.is_a?(String) && value.match?(UUID)
      def limit_param = Timeline::Query.limit(params[:limit]) # 1..100, padrão 50
      def body_params = params.except(:controller, :action, :id).permit!.to_h
    end
  end
  ```

  `app/controllers/exams/admin/base_controller.rb`:

  ```ruby
  # /exams/admin/* (contrato §5): municipal_admin; escrita com step-up.
  module Exams
    module Admin
      class BaseController < Exams::BaseController
        before_action { forbid("missing_role") unless Current.user.has_role?("municipal_admin") }
        before_action :require_step_up!, if: -> { request.post? || request.patch? || request.delete? }
      end
    end
  end
  ```

  `app/controllers/exams/admin/catalog_items_controller.rb`:

  ```ruby
  module Exams
    module Admin
      class CatalogItemsController < BaseController
        def index
          render json: { items: ExamCatalogItem.order(:sigtap_code).map { |item| Exams::AdminJson.catalog_item(item) } }
        end

        def create
          input = Exams::CatalogInput.call(body_params)
          return render_failure(input) if input.failure?

          item = ExamCatalogItem.create!(input.payload[:attrs])
          render json: Exams::AdminJson.catalog_item(item), status: :created
        rescue ActiveRecord::RecordNotUnique
          render json: { error: "duplicate_sigtap" }, status: :unprocessable_entity
        end

        def update
          item = uuid?(params[:id]) && ExamCatalogItem.find_by(id: params[:id])
          return not_found unless item

          input = Exams::CatalogInput.call(body_params, item: item)
          return render_failure(input) if input.failure?

          item.update!(input.payload[:attrs])
          render json: Exams::AdminJson.catalog_item(item)
        end
      end
    end
  end
  ```

  Em `config/routes.rb`, depois do bloco `/sncr` (o prefixo `/exams` é um só no proxy de dev do dashboard):

  ```ruby
  # Rastreamento do exame (ADR 0036; contrato §3–§6). Prefixo único no proxy
  # de dev do dashboard. As rotas das Tasks 6, 9–13, 16 e 18 entram aqui.
  scope "/exams", module: :exams, as: :exams do
    scope "admin", module: :admin, as: :admin do
      get   "catalog_items",     to: "catalog_items#index"
      post  "catalog_items",     to: "catalog_items#create"
      patch "catalog_items/:id", to: "catalog_items#update"
    end
  end
  ```

- [ ] **Step 4: Run** os dois arquivos do Step 2. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/deadlines.rb app/services/exams/catalog_input.rb app/services/exams/admin_json.rb app/controllers/exams/base_controller.rb app/controllers/exams/admin/base_controller.rb app/controllers/exams/admin/catalog_items_controller.rb config/routes.rb spec/services/exams/deadlines_spec.rb spec/requests/exams/admin/catalog_items_spec.rb
  git commit -m "feat: add city exam catalog and deadline resolution

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 5: Prestadores, token FHIR e usuários do prestador

**Files:**
- Create: `app/commands/exams/save_provider.rb`, `app/commands/exams/rotate_fhir_token.rb`, `app/commands/exams/invite_provider_user.rb`
- Modify: `app/commands/accept_invitation.rb`, `app/commands/invite_member.rb` (aceita `exam_provider:`)
- Create: `app/controllers/exams/admin/providers_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/exams/admin/providers_spec.rb`, `spec/commands/accept_invitation_exam_provider_spec.rb`

**Interfaces:**
- Consumes: `Exams::AdminJson.provider`, `Exams::CatalogInput` (prazos), `InviteMember`, `AcceptInvitation`.
- Produces: `Exams::SaveProvider.call(params, provider: nil) -> Result(provider:)` (`invalid_name`, `invalid_kind`, `invalid_cnes`, `duplicate_name`, `invalid_items`, `invalid_deadline`); `Exams::RotateFhirToken.call(provider:) -> Result(token:)`, `.revoke(provider:) -> Result`; `Exams::InviteProviderUser.call(provider:, email:, invited_by:) -> Result(invitation:)`; `InviteMember.call(..., exam_provider: nil)`; `AcceptInvitation` cria `ExamProviderUser` quando `invitation.exam_provider_id`.

- [ ] **Step 1: Write the failing tests.** `spec/requests/exams/admin/providers_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5: prestadores da cidade, itens com prazos próprios, token FHIR
  # mostrado uma vez (só o hash fica), revogação e convite do usuário do portal.
  RSpec.describe "Prestadores de exames (admin)", type: :request do
    before { exam_city! }

    let(:item) { exam_catalog_item! }
    let(:body) { { name: "Laboratório Central", cnes: "2345678", kind: "accredited", active: true,
                   items: [ { catalog_item_id: item.id, deadlines: { "result" => 5 } } ] } }

    before { stepped_up!(municipal_admin!) }

    it "cria com itens, gira e revoga o token, convida o usuário do portal" do
      json_post "/exams/admin/providers", body
      expect(response).to have_http_status(:created)
      provider = ExamProvider.find(json_body["id"])
      expect(json_body["items"]).to eq([ { "catalog_item_id" => item.id, "sigtap_code" => item.sigtap_code,
                                           "deadlines" => { "authorize" => nil, "schedule" => nil, "collect" => nil,
                                                            "result" => 5, "acknowledge" => nil } } ])
      expect(json_body).to include("simulated" => false, "has_fhir_token" => false)

      json_post "/exams/admin/providers/#{provider.id}/fhir_token"
      token = json_body["token"]
      expect(token.length).to be >= 40
      expect(provider.reload.fhir_token_digest).to eq(Digest::SHA256.hexdigest(token))
      expect(ExamProvider.by_token(token)).to eq(provider)
      expect(DomainEvent.where(name: "exam_provider.fhir_token_rotated").last.payload).to eq("exam_provider_id" => provider.id)
      expect(DomainEvent.last(3).map { |e| e.payload.to_json }.join).not_to include(token)

      json_post "/exams/admin/providers/#{provider.id}/fhir_token"
      expect(ExamProvider.by_token(token)).to be_nil
      json_delete "/exams/admin/providers/#{provider.id}/fhir_token"
      expect(response).to have_http_status(:no_content)
      expect(provider.reload.fhir_token_active?).to be(false)

      json_post "/exams/admin/providers/#{provider.id}/users", email: "Lab@Central.com.br"
      expect(response).to have_http_status(:created)
      expect(json_body).to include("email" => "lab@central.com.br", "role" => "exam_provider")
      expect(Invitation.find(json_body["id"])).to have_attributes(exam_provider_id: provider.id)
    end

    it "recusa nome repetido, tipo, CNES e item inexistente; a API nunca cria simulado" do
      json_post "/exams/admin/providers", body.merge(simulated: true)
      expect(ExamProvider.find(json_body["id"]).simulated).to be(false)
      { { name: "laboratório central" } => "duplicate_name", { kind: "hospital" } => "invalid_kind",
        { cnes: "12345" } => "invalid_cnes", { name: " " } => "invalid_name",
        { items: [ { catalog_item_id: SecureRandom.uuid } ] } => "invalid_items",
        { items: [ { catalog_item_id: item.id, deadlines: { "result" => 400 } } ] } => "invalid_deadline" }.each do |over, error|
        json_post "/exams/admin/providers", body.merge(over)
        expect([ response.status, json_body["error"] ]).to eq([ 422, error ]), over.inspect
      end
    end
  end
  ```

  `spec/commands/accept_invitation_exam_provider_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0036: o convite do prestador liga o usuário novo ao prestador, numa
  # transação só com o usuário e o papel.
  RSpec.describe AcceptInvitation do
    before { exam_city! }

    it "cria ExamProviderUser junto com o papel exam_provider" do
      provider = exam_provider!
      admin = municipal_admin!
      invitation = Exams::InviteProviderUser.call(provider: provider, email: "lab@central.com.br", invited_by: admin)
                                            .payload.fetch(:invitation)
      result = described_class.call(token: invitation.token, password: "senha-muito-segura-1")
      expect(result).to be_ok
      user = result.payload[:user]
      expect(user.has_role?("exam_provider")).to be(true)
      expect(ExamProviderUser.find_by(user: user).exam_provider).to eq(provider)
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL.

- [ ] **Step 3: Implemente.** `app/commands/exams/save_provider.rb`:

  ```ruby
  # Cria ou altera um prestador e os exames que ele faz (contrato §5). A API
  # nunca grava `simulated` (só a semente de dev). `items` presente = lista
  # final (os ausentes saem).
  module Exams
    module SaveProvider
      module_function

      def call(params, provider: nil)
        params = (params.respond_to?(:to_unsafe_h) ? params.to_unsafe_h : params.to_h).deep_stringify_keys
        attrs = {}
        if params.key?("name") || provider.nil?
          name = params["name"].is_a?(String) ? params["name"].strip : ""
          return Result.fail(:invalid_name) unless name.length.between?(1, 120)

          attrs[:name] = name
        end
        if params.key?("kind") || provider.nil?
          return Result.fail(:invalid_kind) unless ExamProvider::KINDS.include?(params["kind"])

          attrs[:kind] = params["kind"]
        end
        if params.key?("cnes")
          cnes = params["cnes"]
          return Result.fail(:invalid_cnes) unless cnes.nil? || (cnes.is_a?(String) && cnes.match?(/\A\d{7}\z/))

          attrs[:cnes] = cnes
        end
        if params.key?("active")
          return Result.fail(:invalid_active) unless [ true, false ].include?(params["active"])

          attrs[:active] = params["active"]
        end
        items = params.key?("items") ? items(params["items"]) : nil
        return items if items.is_a?(Result)

        ApplicationRecord.transaction do
          provider ||= ExamProvider.new
          provider.update!(attrs)
          replace_items!(provider, items) if items
        end
        Result.ok(provider: provider)
      rescue ActiveRecord::RecordNotUnique
        Result.fail(:duplicate_name)
      end

      def items(raw)
        return Result.fail(:invalid_items) unless raw.is_a?(Array) && raw.size <= 200

        raw.map do |entry|
          entry = entry.is_a?(Hash) ? entry.deep_stringify_keys : {}
          item = entry["catalog_item_id"].is_a?(String) && ExamCatalogItem.find_by(id: entry["catalog_item_id"])
          return Result.fail(:invalid_items) unless item

          days = CatalogInput.send(:deadlines, entry["deadlines"] || {})
          return Result.fail(:invalid_deadline) unless days

          [ item, days ]
        end.tap { |list| return Result.fail(:invalid_items) unless list.map { |i, _| i.id }.uniq.size == list.size }
      end

      def replace_items!(provider, items)
        keep = items.map { |item, _| item.id }
        provider.items.where.not(exam_catalog_item_id: keep).delete_all
        items.each do |item, days|
          row = provider.items.find_or_initialize_by(exam_catalog_item_id: item.id)
          row.update!(Deadlines::STAGES.to_h { |stage| [ :"#{stage}_days", nil ] }.merge(days))
        end
      end
      private_class_method :items, :replace_items!
    end
  end
  ```

  `app/commands/exams/rotate_fhir_token.rb`:

  ```ruby
  # Token FHIR do prestador (ADR 0036): mostrado uma vez; só o SHA-256 fica;
  # girar troca o anterior; revogar desliga. Evento só com o id.
  module Exams
    module RotateFhirToken
      module_function

      def call(provider:)
        token = SecureRandom.urlsafe_base64(32)
        provider.update!(fhir_token_digest: ExamProvider.digest(token), fhir_token_rotated_at: Time.current, fhir_token_revoked_at: nil)
        DomainEvents.publish("exam_provider.fhir_token_rotated", exam_provider_id: provider.id)
        Result.ok(token: token)
      end

      def revoke(provider:)
        return Result.ok unless provider.fhir_token_active?

        provider.update!(fhir_token_revoked_at: Time.current)
        DomainEvents.publish("exam_provider.fhir_token_revoked", exam_provider_id: provider.id)
        Result.ok
      end
    end
  end
  ```

  `app/commands/exams/invite_provider_user.rb`:

  ```ruby
  # Convite do usuário do portal do prestador (contrato §5): o mecanismo de
  # convite existente, papel exam_provider, ligado ao prestador.
  module Exams
    module InviteProviderUser
      module_function

      def call(provider:, email:, invited_by:)
        InviteMember.call(email: email, role: "exam_provider", invited_by: invited_by, exam_provider: provider)
      end
    end
  end
  ```

  Em `app/commands/invite_member.rb`: `self.call(..., exam_provider: nil)` e `initialize(..., exam_provider:)` guardam `@exam_provider`; `Invitation.create!` ganha `exam_provider: @exam_provider`; antes da transação, `return Result.fail(:invalid, message: "exam_provider exige prestador") if (@role.to_s == "exam_provider") != !@exam_provider.nil?`. Em `app/commands/accept_invitation.rb`, dentro da transação depois de `Membership.create!`: `ExamProviderUser.create!(exam_provider_id: inv.exam_provider_id, user: user) if inv.exam_provider_id`.

  `app/controllers/exams/admin/providers_controller.rb`:

  ```ruby
  module Exams
    module Admin
      class ProvidersController < BaseController
        before_action :load_provider, except: %i[index create]

        def index
          render json: { providers: ExamProvider.order(:name).map { |p| Exams::AdminJson.provider(p) } }
        end

        def create = save(nil, :created)
        def update = save(@provider, :ok)

        def rotate_token
          render json: { token: Exams::RotateFhirToken.call(provider: @provider).payload[:token] }, status: :created
        end

        def revoke_token
          Exams::RotateFhirToken.revoke(provider: @provider)
          head :no_content
        end

        def invite_user
          result = Exams::InviteProviderUser.call(provider: @provider, email: params[:email], invited_by: Current.user)
          return render(json: { error: result.reason.to_s }, status: :unprocessable_entity) if result.failure?

          invitation = result.payload[:invitation]
          render json: { id: invitation.id, email: invitation.email, role: invitation.role, expires_at: invitation.expires_at.iso8601 },
                 status: :created
        end

        private

        def save(provider, status)
          result = Exams::SaveProvider.call(body_params.except("simulated"), provider: provider)
          return render_failure(result) if result.failure?

          render json: Exams::AdminJson.provider(result.payload[:provider].reload), status: status
        end

        def load_provider
          @provider = uuid?(params[:id]) && ExamProvider.find_by(id: params[:id])
          not_found unless @provider
        end
      end
    end
  end
  ```

  (O e-mail do convite é enviado pelo caminho existente — confira se `SetupController#invite_member` chama `MemberInvitationMailer` depois de `InviteMember`; se sim, chame o mesmo mailer aqui com `deliver_later`, sem montar e-mail novo.)

  Rotas, dentro de `scope "admin"`:

  ```ruby
      get    "providers",                to: "providers#index"
      post   "providers",                to: "providers#create"
      patch  "providers/:id",            to: "providers#update"
      post   "providers/:id/fhir_token", to: "providers#rotate_token"
      delete "providers/:id/fhir_token", to: "providers#revoke_token"
      post   "providers/:id/users",      to: "providers#invite_user"
  ```

- [ ] **Step 4: Run** os dois arquivos do Step 1 e `rspec spec/commands/invite_member_spec.rb spec/commands/accept_invitation_spec.rb spec/requests/setup_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/commands/exams/save_provider.rb app/commands/exams/rotate_fhir_token.rb app/commands/exams/invite_provider_user.rb app/commands/invite_member.rb app/commands/accept_invitation.rb app/controllers/exams/admin/providers_controller.rb config/routes.rb spec/requests/exams/admin/providers_spec.rb spec/commands/accept_invitation_exam_provider_spec.rb
  git commit -m "feat: manage exam providers, fhir tokens and provider portal invitations

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.2 → Done (lado api).

### Task 6: Arquivos cifrados (guia e laudo) e URL curta assinada

**Files:**
- Create: `app/services/exams/files.rb`, `app/controllers/exams/uploads_controller.rb`, `app/controllers/exams/files_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/exams/files_spec.rb`, `spec/requests/exams/uploads_spec.rb`

**Interfaces:**
- Produces: `Exams::Files::MAX_BYTES`, `.store(bytes:, content_type:, user: nil, provider: nil) -> Result(file:)` (falha `invalid_file`), `.store_base64(data:, content_type:, user: nil, provider: nil) -> Result(file:)`, `.claim!(file_id:, user: nil, provider: nil) -> ExamFile|nil` (só arquivo não consumido do mesmo autor; trava e marca `consumed_at`), `.url_for(file, user:, now: Time.current) -> String` (`/exams/files/<token>`), `.resolve(token, user:, now: Time.current) -> ExamFile|:expired|nil`, `.filename(file) -> String`. `GET /exams/files/:token` vencido → 410 `link_expired`.

- [ ] **Step 1: Write the failing tests.** `spec/services/exams/files_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe Exams::Files do
    before { exam_city! }

    let(:user) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }

    it "guarda PDF/JPEG/PNG pelos bytes mágicos e recusa o resto e o tamanho" do
      expect(described_class.store(bytes: ExamHelpers::PDF_BYTES, content_type: "application/pdf", user: user)).to be_ok
      png = "\x89PNG\r\n\x1A\n".b + "x" * 10
      expect(described_class.store(bytes: png, content_type: "image/png", user: user)).to be_ok
      expect(described_class.store(bytes: png, content_type: "application/pdf", user: user).reason).to eq(:invalid_file)
      expect(described_class.store(bytes: "MZ\x90\x00".b, content_type: "application/pdf", user: user).reason).to eq(:invalid_file)
      big = "%PDF-".b + ("a" * described_class::MAX_BYTES)
      expect(described_class.store(bytes: big, content_type: "application/pdf", user: user).reason).to eq(:invalid_file)
      expect(described_class.store_base64(data: "não é base64!", content_type: "application/pdf", user: user).reason).to eq(:invalid_file)
    end

    it "claim! só do mesmo autor e uma vez" do
      file = described_class.store(bytes: ExamHelpers::PDF_BYTES, content_type: "application/pdf", user: user).payload[:file]
      other = staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier")
      expect(described_class.claim!(file_id: file.id, user: other)).to be_nil
      expect(described_class.claim!(file_id: file.id, user: user)).to eq(file)
      expect(described_class.claim!(file_id: file.id, user: user)).to be_nil
    end

    it "URL curta: 5 min, só para o mesmo usuário e a mesma cidade" do
      file = pdf_file!(user: user)
      url = described_class.url_for(file, user: user)
      token = url.delete_prefix("/exams/files/")
      expect(described_class.resolve(token, user: user)).to eq(file)
      expect(described_class.resolve(token, user: municipal_admin!)).to be_nil
      travel 6.minutes do
        expect(described_class.resolve(token, user: user)).to eq(:expired)
      end
    end
  end
  ```

  `spec/requests/exams/uploads_spec.rb`:

  ```ruby
  require "rails_helper"

  # Divergência D3: upload por JSON (Base64) — a guarda de escrita por cookie
  # recusa multipart. O download pela URL curta exige a sessão do mesmo usuário.
  RSpec.describe "Arquivos de exame", type: :request do
    before { exam_city! }

    let(:user) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }

    it "sobe em JSON, recusa multipart, baixa só pela URL curta" do
      sign_in_as(user)
      post "/exams/uploads", params: { file: Rack::Test::UploadedFile.new(StringIO.new(ExamHelpers::PDF_BYTES), "application/pdf", original_filename: "guia.pdf") }
      expect(response).to have_http_status(:unsupported_media_type)
      log = capture_log do
        json_post "/exams/uploads", content_type: "application/pdf", data: Base64.strict_encode64(ExamHelpers::PDF_BYTES)
      end
      expect(response).to have_http_status(:created)
      expect(log).not_to include(Base64.strict_encode64(ExamHelpers::PDF_BYTES)[0, 20])
      file = ExamFile.find(json_body["upload_id"])
      file.update!(consumed_at: Time.current) # arquivo solto não sai: só o ligado a pedido ou resultado
      get Exams::Files.url_for(file, user: user)
      expect(response).to have_http_status(:ok)
      expect(response.body.b).to eq(ExamHelpers::PDF_BYTES)
      expect(response.headers["Cache-Control"]).to include("no-store")
      expect(response.headers["Content-Disposition"]).to start_with("inline")
      sign_in_as(municipal_admin!)
      get Exams::Files.url_for(file, user: user)
      expect([ response.status, json_body["error"] ]).to eq([ 404, "not_found" ])
      sign_in_as(user)
      url = Exams::Files.url_for(file, user: user)
      travel(6.minutes) { get url }
      expect([ response.status, json_body["error"] ]).to eq([ 410, "link_expired" ])
      json_post "/exams/uploads", content_type: "application/zip", data: Base64.strict_encode64("PK\x03\x04")
      expect([ response.status, json_body["error"] ]).to eq([ 422, "invalid_file" ])
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL.

- [ ] **Step 3: Implemente** `app/services/exams/files.rb`:

  ```ruby
  # Arquivos do exame (ADR 0036; Desvio 5; Divergências D3/D4): guia (PDF,
  # JPEG, PNG) e laudo (PDF), conferidos pelos bytes mágicos, até 10 MB,
  # cifrados com a chave da cidade. Download por URL curta (5 min) amarrada à
  # cidade, ao arquivo e ao usuário; a trilha é de quem chama.
  module Exams
    module Files
      MAX_BYTES = 10 * 1024 * 1024
      MAGIC = { "application/pdf" => "%PDF-".b, "image/jpeg" => "\xFF\xD8\xFF".b, "image/png" => "\x89PNG\r\n\x1A\n".b }.freeze
      PURPOSE = "exam_file".freeze
      TTL = 5.minutes

      module_function

      def store(bytes:, content_type:, user: nil, provider: nil)
        bytes = bytes.to_s.b
        magic = MAGIC[content_type]
        return Result.fail(:invalid_file) unless magic && bytes.bytesize.between?(1, MAX_BYTES) && bytes.start_with?(magic)

        file = ExamFile.create!(content_type: content_type, byte_size: bytes.bytesize, sha256: Digest::SHA256.hexdigest(bytes),
                                data: Base64.strict_encode64(bytes), uploaded_by_user: user, exam_provider: provider)
        Result.ok(file: file)
      end

      def store_base64(data:, content_type:, user: nil, provider: nil)
        return Result.fail(:invalid_file) unless data.is_a?(String) && data.length <= (MAX_BYTES * 4 / 3) + 8

        store(bytes: Base64.strict_decode64(data), content_type: content_type, user: user, provider: provider)
      rescue ArgumentError
        Result.fail(:invalid_file)
      end

      # Liga o arquivo a um pedido/resultado: só o não consumido do mesmo autor.
      def claim!(file_id:, user: nil, provider: nil)
        return nil unless file_id.is_a?(String) && file_id.match?(Exams::BaseController::UUID)

        file = ExamFile.lock.find_by(id: file_id, consumed_at: nil)
        return nil unless file
        return nil unless (user && file.uploaded_by_user_id == user.id) || (provider && file.exam_provider_id == provider.id)

        file.update!(consumed_at: Time.current)
        file
      end

      # O vencimento vai no próprio payload assinado (não no expires_in do
      # verifier, que devolve nil igual para vencido e para falso): o vencido
      # responde 410 link_expired (dashboard D11) e o dashboard relê o exame.
      def url_for(file, user:, now: Time.current)
        token = verifier.generate({ "c" => Current.city.id, "f" => file.id, "u" => user.id, "x" => (now + TTL).to_i }, purpose: PURPOSE)
        "/exams/files/#{token}"
      end

      # ExamFile, :expired ou nil (assinatura falsa, outra cidade, outro usuário).
      def resolve(token, user:, now: Time.current)
        payload = verifier.verified(token.to_s, purpose: PURPOSE)
        return nil unless payload.is_a?(Hash) && payload["c"] == Current.city.id && payload["u"] == user.id
        return :expired if now.to_i > payload["x"].to_i

        ExamFile.find_by(id: payload["f"])
      end

      def filename(file) = "exame-#{file.id[0, 8]}.#{{ "application/pdf" => "pdf", "image/jpeg" => "jpg", "image/png" => "png" }.fetch(file.content_type)}"

      def verifier = Rails.application.message_verifier(PURPOSE)
      private_class_method :verifier
    end
  end
  ```

  `app/controllers/exams/uploads_controller.rb`:

  ```ruby
  # POST /exams/uploads (Divergência D3): JSON { content_type, data(base64) } → { upload_id }.
  # Quem sobe: a unidade (recepção ou profissional com vínculo) ou o prestador.
  module Exams
    class UploadsController < BaseController
      def create
        provider = ExamProviderUser.find_by(user_id: Current.user.id)&.exam_provider if Current.user.has_role?("exam_provider")
        allowed = provider || Current.user.has_role?("citizen_verifier") || Current.user.has_role?("health_professional")
        return forbid("missing_role") unless allowed

        result = Exams::Files.store_base64(data: params[:data], content_type: params[:content_type], user: Current.user)
        return render_failure(result) if result.failure?

        render json: { upload_id: result.payload[:file].id }, status: :created
      end
    end
  end
  ```

  `app/controllers/exams/files_controller.rb`:

  ```ruby
  # GET /exams/files/:token (Divergência D4): a URL curta do laudo ou da guia.
  # Quem gerou a URL já passou pela regra de leitura e pela trilha.
  module Exams
    class FilesController < BaseController
      def show
        file = Exams::Files.resolve(params[:token], user: Current.user)
        return render(json: { error: "link_expired" }, status: :gone) if file == :expired
        return not_found unless file&.consumed?

        response.headers["Cache-Control"] = "no-store"
        send_data file.bytes, type: file.content_type, disposition: "inline", filename: Exams::Files.filename(file)
      end
    end
  end
  ```

  Rotas dentro de `scope "/exams"`: `post "uploads", to: "uploads#create"` e `get "files/:token", to: "files#show", format: false, constraints: { token: %r{[^/]+} }`.

- [ ] **Step 4: Run** os dois. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/files.rb app/controllers/exams/uploads_controller.rb app/controllers/exams/files_controller.rb config/routes.rb spec/services/exams/files_spec.rb spec/requests/exams/uploads_spec.rb
  git commit -m "feat: store encrypted exam files with short signed download urls

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 3 — O exame e seu ciclo de vida (F-21.3)

### Task 7: Núcleo — abrir, transitar, escalar, quem lê, formas JSON e trilha

**Files:**
- Create: `app/services/exams/{open,transition,escalate,access,json,trail,labels}.rb`
- Test: `spec/services/exams/transition_spec.rb`, `spec/services/exams/access_spec.rb`, `spec/services/exams/json_spec.rb`

**Interfaces:**
- Consumes: modelos (Task 3), `Exams::Deadlines` (Task 4), `Exams::Files.url_for` (Task 6), `ClinicalRecord::Access.call(user:, patient:)`, `ClinicalTerms::SigtapExams`, `Screenings::Json.staff_name(user)`, `ClinicalDocuments::Codes.short_code`.
- Produces:
  - `Exams::Open.call(patient:, sigtap_code:, priority:, unit:, origin:, created_by:, requester_user: nil, requester_name: nil, requester_cns: nil, cid10: nil, consultation_exam_request: nil, guide_file: nil, external_reference: nil, sigtap_competence: nil, require_catalog: true, now: Time.current) -> Result(order:)` (`invalid_priority`, `sigtap_not_in_catalog`).
  - `Exams::Transition::FLOW`, `NEXT_STAGE`; `.call(order:, kind:, actor_user: nil, actor_provider: nil, reason: nil, changes: {}, event_payload: {}, now: Time.current) -> Result(order:)` (`invalid_transition` com `details: { status: }`, `reason_required`). **Pedido travado (`order.lock!`) dentro de transação de quem chama.**
  - `Exams::Escalate.call(order:, reason:, now: Time.current) -> Result(order:, escalated: Boolean)` (idempotente).
  - `Exams::Access::Grant = Data.define(:view, :kind, :opening)`; `.for_order(user:, order:) -> Grant|nil`; `.unit_staff?(user, unit_id)`, `.regulator?(user)`, `.team_member?(user, unit_id)`, `.team_unit_ids(user) -> Array<String>`, `.provider_of(user) -> ExamProvider|nil`.
  - `Exams::Json.order(order, view:, user:, events: true) -> Hash`, `.result(result, view:, user:) -> Hash`, `.event(event, view:) -> Hash`, `.provider_order(order) -> Hash`.
  - `Exams::Trail.viewed!(order:, user:, grant:)`, `.result_viewed!(result:, user:)`.
  - `Exams::Labels.exam(code, competence) -> String`.

- [ ] **Step 1: Write the failing tests.** `spec/services/exams/transition_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0036 (spec §5; contrato §1; Desvio 11): a tabela de transições, o
  # prazo da próxima etapa, motivo obrigatório e eventos só com ids.
  RSpec.describe Exams::Transition do
    before { exam_city! }

    let(:unit) { create_unit("UBS Transição") }
    let(:doctor) { doctor!(unit) }
    let(:patient) { exam_patient!(10).first }
    let(:provider) { exam_provider! }

    def open(sigtap: "0202010503", priority: "routine", **item)
      exam_catalog_item!(sigtap_code: sigtap, **item)
      Exams::Open.call(patient: patient, sigtap_code: sigtap, priority: priority, unit: unit, origin: "pec",
                       requester_user: doctor, created_by: doctor).payload.fetch(:order)
    end

    def step(order, kind, **opts)
      ApplicationRecord.transaction do
        order.lock!
        described_class.call(order: order, kind: kind, actor_user: doctor, **opts)
      end
    end

    it "abre aguardando agendamento (ou autorização), com prazo, evento requested e código curto" do
      travel_to Time.zone.local(2026, 10, 13, 10) do
        order = open
        expect(order).to have_attributes(status: "awaiting_schedule", stage: "schedule", stage_due_on: Date.new(2026, 10, 28))
        expect(order.short_code).to match(/\A[23456789ABCDEFGHJKMNPQRSTUVWXYZ]{10}\z/)
        expect(order.events.map(&:kind)).to eq(%w[requested])
        expect(DomainEvent.where(name: "exam_order.requested").last.payload)
          .to eq("exam_order_id" => order.id, "origin" => "pec", "priority" => "routine")
        urgent = open(sigtap: "0202020380", priority: "urgent", requires_authorization: true)
        expect(urgent).to have_attributes(status: "awaiting_authorization", stage: "authorize", stage_due_on: Date.new(2026, 10, 21))
      end
    end

    it "percorre o ciclo inteiro e cada evento guarda o prazo da etapa que fechou" do
      order = open(requires_authorization: true)
      expect(step(order, "authorized", changes: { exam_provider_id: provider.id })).to be_ok
      expect(order.reload).to have_attributes(status: "awaiting_schedule", exam_provider_id: provider.id, stage: "schedule")
      step(order, "scheduled", changes: { scheduled_at: 2.days.from_now, scheduled_place: "Rua A, 10" })
      step(order, "scheduled", changes: { scheduled_at: 3.days.from_now, scheduled_place: "Rua A, 10" })
      step(order, "collected")
      step(order, "resulted", event_payload: { source: "unit", critical: false })
      step(order, "acknowledged", event_payload: { by_team: false })
      expect(order.reload.status).to eq("acknowledged")
      expect(order.stage).to be_nil
      expect(order.events.map(&:kind)).to eq(%w[requested authorized scheduled scheduled collected resulted acknowledged])
      expect(order.events.second.closed_stage_due_on).to eq(Time.zone.today + 15)
      expect(DomainEvent.where(name: "exam_order.resulted").last.payload)
        .to eq("exam_order_id" => order.id, "source" => "unit", "critical" => false)
    end

    it "recusa transição fora da tabela com o status atual; negar e cancelar exigem motivo" do
      order = open
      expect(step(order, "collected").then { |r| [ r.reason, r.details ] }).to eq([ :invalid_transition, { status: "awaiting_schedule" } ])
      expect(step(order, "authorized").reason).to eq(:invalid_transition)
      expect(step(order, "cancelled", reason: " ").reason).to eq(:reason_required)
      expect(step(order, "cancelled", reason: "Paciente fez em outro serviço")).to be_ok
      expect(step(order, "scheduled").reason).to eq(:invalid_transition)
      expect(order.reload.events.last.reason).to eq("Paciente fez em outro serviço")
      expect(DomainEvent.where(name: "exam_order.cancelled").last.payload).to eq("exam_order_id" => order.id)
    end

    it "escalar é idempotente e não muda o status" do
      order = open
      step(order, "scheduled"); step(order, "collected"); step(order, "resulted", event_payload: { source: "unit", critical: true })
      first = ApplicationRecord.transaction { order.lock!; Exams::Escalate.call(order: order, reason: "critical") }
      again = ApplicationRecord.transaction { order.lock!; Exams::Escalate.call(order: order, reason: "overdue") }
      expect([ first.payload[:escalated], again.payload[:escalated] ]).to eq([ true, false ])
      expect(order.reload).to have_attributes(status: "resulted", escalated_at: be_present)
      expect(order.events.where(kind: "escalated").pluck(:escalation_reason)).to eq(%w[critical])
    end
  end
  ```

  `spec/services/exams/access_spec.rb`:

  ```ruby
  require "rails_helper"

  # Quem lê o exame (spec §7; Desvios 9 e 10): solicitante e leitura do 19a
  # veem a forma clínica; a equipe só com o exame escalado; unidade e
  # regulador a administrativa; prestador e estranho, nada.
  RSpec.describe Exams::Access do
    before { exam_city! }

    let(:unit) { create_unit("UBS Acesso") }
    let(:other_unit) { create_unit("UBS Longe") }
    let(:doctor) { doctor!(unit) }
    let(:order) { exam_order!(patient: exam_patient!(20).first, unit: unit, requester: doctor) }

    def team!(user, unit) = exam_team!(unit, user)

    it "matriz de leitura" do
      expect(described_class.for_order(user: doctor, order: order)).to have_attributes(view: :clinical, kind: :author)
      stranger = doctor!(other_unit)
      expect(described_class.for_order(user: stranger, order: order)).to be_nil
      nurse = nurse!(unit)
      team!(nurse, unit)
      expect(described_class.for_order(user: nurse, order: order)).to have_attributes(view: :administrative, kind: :unit)
      ExamOrderEvent.transaction do
        ExamOrderEvent.create!(exam_order: order, kind: "escalated", escalation_reason: "overdue", occurred_at: Time.current)
        order.update!(escalated_at: Time.current)
      end
      expect(described_class.for_order(user: nurse, order: order.reload)).to have_attributes(view: :clinical, kind: :team)
      reception = staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier")
      expect(described_class.for_order(user: reception, order: order)).to have_attributes(view: :administrative, kind: :unit)
      regulator = staff_with("reg-#{SecureRandom.hex(2)}@cidade.gov.br", "exam_regulator")
      expect(described_class.for_order(user: regulator, order: order)).to have_attributes(view: :administrative, kind: :regulator)
      lab = staff_with("lab-#{SecureRandom.hex(2)}@cidade.gov.br", "exam_provider")
      expect(described_class.for_order(user: lab, order: order)).to be_nil
    end

    it "equipe = médico ou enfermeiro ativo em equipe ativa da unidade do exame" do
      tech = screener!(unit, cbo: "322205")
      team!(tech, unit)
      expect(described_class.team_member?(tech, unit.id)).to be(false)
      far_doctor = doctor!(other_unit)
      team!(far_doctor, other_unit)
      expect(described_class.team_member?(far_doctor, unit.id)).to be(false)
      expect(described_class.team_unit_ids(far_doctor)).to eq([ other_unit.id ])
    end
  end
  ```

  `spec/services/exams/json_spec.rb`:

  ```ruby
  require "rails_helper"

  # Divergência D5: a forma administrativa nunca leva valores, crítico, PDF,
  # CID nem CNS do solicitante.
  RSpec.describe Exams::Json do
    before { exam_city! }

    it "clínica × administrativa" do
      unit = create_unit("UBS Json")
      doctor = doctor!(unit)
      order = exam_order!(patient: exam_patient!(30).first, unit: unit, requester: doctor)
      file = pdf_file!(user: doctor).tap { |f| f.update!(consumed_at: Time.current) }
      ExamResult.create!(exam_order: order, patient: order.patient, sigtap_code: order.sigtap_code, origin: "order", source: "unit",
                         pdf_file: file, values: [ { "key" => "hba1c", "label" => "HbA1c", "value" => "9.1", "unit" => "%", "loinc" => "4548-4" } ].to_json,
                         released_on: Time.zone.today, laboratory_name: "Lab", critical: true, registered_by_user: doctor)
      clinical = described_class.order(order.reload, view: :clinical, user: doctor)
      admin = described_class.order(order, view: :administrative, user: doctor)
      expect(clinical[:result]).to include(critical: true, values: [ hash_including("value" => "9.1") ])
      expect(clinical[:result][:pdf_url]).to start_with("/exams/files/")
      expect(admin[:result].keys).to eq(%i[id origin source released_on laboratory_name])
      expect(admin.to_json).not_to include("9.1", "4548-4")
      expect(admin[:requester][:cns]).to be_nil
      expect(described_class.order(order, view: :clinical, user: doctor, events: false)).not_to have_key(:events)
      expect(clinical[:label]).to eq("DOSAGEM DE HEMOGLOBINA GLICOSILADA")
      expect(clinical[:consultation_id]).to be_nil
      expect(clinical[:result_fields]).to eq([ { "key" => "hba1c", "label" => "Hemoglobina glicada", "unit" => "%", "loinc" => "4548-4" } ])
    end
  end
  ```

- [ ] **Step 2: Run** os três. Expected: FAIL.

- [ ] **Step 3: Implemente.** `app/services/exams/labels.rb`:

  ```ruby
  # Nome do exame (SIGTAP da competência gravada) e rótulos de situação.
  module Exams
    module Labels
      CITIZEN_STATUS = {
        "awaiting_authorization" => "Aguardando autorização", "denied" => "Pedido não autorizado — procure sua unidade",
        "awaiting_schedule" => "Pedido registrado", "scheduled" => "Agendado", "collected" => "Coletado",
        "resulted" => "Resultado registrado", "acknowledged" => "Resultado registrado", "cancelled" => "Cancelado"
      }.freeze
      STAGE = { "authorize" => "Autorização", "schedule" => "Agendamento", "collect" => "Coleta", "result" => "Resultado",
                "acknowledge" => "Ciência" }.freeze

      module_function

      def exam(code, competence) = ClinicalTerms::SigtapExams.label(code, competence) || "Procedimento #{code}"
    end
  end
  ```

  `app/services/exams/open.rb`:

  ```ruby
  # Abre o exame (ADR 0036; Desvio 8): status inicial pelo catálogo
  # (exige autorização → awaiting_authorization), prazo da primeira etapa,
  # evento requested e exam_order.requested (só ids). Savepoint próprio: roda
  # dentro da finalização da consulta sem derrubá-la.
  module Exams
    module Open
      module_function

      def call(patient:, sigtap_code:, priority:, unit:, origin:, created_by:, requester_user: nil, requester_name: nil,
               requester_cns: nil, cid10: nil, consultation_exam_request: nil, guide_file: nil, external_reference: nil,
               sigtap_competence: nil, require_catalog: true, now: Time.current)
        return Result.fail(:invalid_priority) unless ExamOrder::PRIORITIES.include?(priority)

        item = ExamCatalogItem.find_by(sigtap_code: sigtap_code, active: true)
        return Result.fail(:sigtap_not_in_catalog) if require_catalog && item.nil?

        competence = sigtap_competence || ClinicalTerms::SigtapExams.find(sigtap_code, on: now.in_time_zone.to_date)&.competence
        return Result.fail(:sigtap_not_in_catalog) unless competence

        status = item&.requires_authorization ? "awaiting_authorization" : "awaiting_schedule"
        stage = status == "awaiting_authorization" ? "authorize" : "schedule"
        due = now.in_time_zone.to_date + Deadlines.days(stage: stage, sigtap_code: sigtap_code, provider_id: nil, priority: priority)
        ApplicationRecord.transaction(requires_new: true) do
          order = ExamOrder.create!(patient: patient, sigtap_code: sigtap_code, sigtap_competence: competence, priority: priority,
                                    cid10_justification: cid10, requester_user: requester_user, requester_name: requester_name,
                                    requester_cns: requester_cns, unit: unit, origin: origin,
                                    consultation_exam_request: consultation_exam_request, guide_file: guide_file,
                                    external_reference: external_reference, short_code: short_code, status: status,
                                    status_changed_at: now, stage: stage, stage_opened_at: now, stage_due_on: due,
                                    created_by_user: created_by)
          ExamOrderEvent.create!(exam_order: order, kind: "requested", actor_user: requester_user || created_by, occurred_at: now)
          DomainEvents.publish("exam_order.requested", exam_order_id: order.id, origin: origin, priority: priority)
          Result.ok(order: order)
        end
      end

      def short_code
        loop do
          code = ClinicalDocuments::Codes.short_code
          return code unless ExamOrder.exists?(short_code: code)
        end
      end
      private_class_method :short_code
    end
  end
  ```

  `app/services/exams/transition.rb`:

  ```ruby
  # A única porta do cache da situação (ADR 0036; Desvios 6, 7 e 11): valida
  # pela tabela, grava o evento (com o prazo da etapa que fecha) e atualiza o
  # pedido com a etapa seguinte e o prazo dela, num savepoint. Quem chama
  # trava o pedido (order.lock!) na própria transação.
  module Exams
    module Transition
      FLOW = {
        "authorized" => { from: %w[awaiting_authorization], to: "awaiting_schedule" },
        "denied" => { from: %w[awaiting_authorization], to: "denied" },
        "scheduled" => { from: %w[awaiting_schedule scheduled], to: "scheduled" },
        "collected" => { from: %w[scheduled], to: "collected" },
        "resulted" => { from: %w[scheduled collected], to: "resulted" },
        "acknowledged" => { from: %w[resulted], to: "acknowledged" },
        "cancelled" => { from: %w[awaiting_authorization awaiting_schedule scheduled collected], to: "cancelled" }
      }.freeze
      NEXT_STAGE = { "awaiting_authorization" => "authorize", "awaiting_schedule" => "schedule", "scheduled" => "collect",
                     "collected" => "result", "resulted" => "acknowledge" }.freeze
      REASON_REQUIRED = %w[denied cancelled].freeze

      module_function

      def call(order:, kind:, actor_user: nil, actor_provider: nil, reason: nil, changes: {}, event_payload: {}, now: Time.current)
        flow = FLOW.fetch(kind)
        return Result.fail(:invalid_transition, details: { status: order.status }) unless flow[:from].include?(order.status)

        reason = reason.is_a?(String) ? reason.strip : nil
        return Result.fail(:reason_required) if REASON_REQUIRED.include?(kind) && reason.to_s.length < 3
        return Result.fail(:reason_too_long) if reason.to_s.length > 500

        to = flow[:to]
        stage = NEXT_STAGE[to]
        provider_id = changes.fetch(:exam_provider_id, order.exam_provider_id)
        due = stage && now.in_time_zone.to_date + Deadlines.days(stage: stage, sigtap_code: order.sigtap_code,
                                                                provider_id: provider_id, priority: order.priority)
        ApplicationRecord.transaction(requires_new: true) do
          ExamOrderEvent.create!(exam_order: order, kind: kind, actor_user: actor_user, actor_provider: actor_provider,
                                 reason: reason.presence, closed_stage_due_on: order.stage_due_on, occurred_at: now)
          order.update!(changes.merge(status: to, status_changed_at: now, stage: stage, stage_opened_at: stage && now,
                                      stage_due_on: due))
          DomainEvents.publish("exam_order.#{kind}", exam_order_id: order.id, **event_payload)
        end
        Result.ok(order: order)
      end
    end
  end
  ```

  `app/services/exams/escalate.rb`:

  ```ruby
  # Leva o exame à fila da equipe (ADR 0036; spec §5): uma vez só; não muda
  # o status. Quem chama trava o pedido.
  module Exams
    module Escalate
      module_function

      def call(order:, reason:, now: Time.current)
        raise ArgumentError, "motivo fora da lista: #{reason}" unless ExamOrderEvent::ESCALATION_REASONS.include?(reason)
        return Result.ok(order: order, escalated: false) if order.escalated?

        ApplicationRecord.transaction(requires_new: true) do
          ExamOrderEvent.create!(exam_order: order, kind: "escalated", escalation_reason: reason, occurred_at: now)
          order.update!(escalated_at: now)
          DomainEvents.publish("exam_order.escalated", exam_order_id: order.id, reason: reason)
        end
        Result.ok(order: order, escalated: true)
      end
    end
  end
  ```

  `app/services/exams/access.rb`:

  ```ruby
  # Quem lê o exame (ADR 0036; spec §7; Desvios 9 e 10). Clínica: o
  # solicitante (com health_professional), a leitura do 19a sobre o paciente
  # (em contexto ou abertura justificada) e, com o exame escalado, a equipe
  # da unidade do exame. Administrativa: a unidade (recepção ou profissional
  # com vínculo nela) e o regulador. Prestador só pelo portal.
  module Exams
    module Access
      Grant = Data.define(:view, :kind, :opening)
      NURSE_PREFIX = "2235".freeze

      module_function

      def for_order(user:, order:)
        professional = user.has_role?("health_professional")
        return Grant.new(view: :clinical, kind: :author, opening: nil) if professional && order.requester_user_id == user.id

        if professional
          grant = ClinicalRecord::Access.call(user: user, patient: order.patient)
          return Grant.new(view: :clinical, kind: grant.kind, opening: grant.opening) if grant.allowed?
          return Grant.new(view: :clinical, kind: :team, opening: nil) if order.escalated? && team_member?(user, order.unit_id)
        end
        return Grant.new(view: :administrative, kind: :unit, opening: nil) if unit_staff?(user, order.unit_id)
        return Grant.new(view: :administrative, kind: :regulator, opening: nil) if regulator?(user)

        nil
      end

      def unit_staff?(user, unit_id)
        return true if user.has_role?("citizen_verifier")

        user.has_role?("health_professional") &&
          ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id }, health_unit_id: unit_id).exists?
      end

      def regulator?(user) = user.has_role?("exam_regulator") || user.has_role?("municipal_admin")

      def team_member?(user, unit_id) = team_unit_ids(user).include?(unit_id)

      def team_unit_ids(user)
        return [] unless user.has_role?("health_professional") && user.professional

        HealthTeamMember.active.joins(:health_team)
                        .where(professional_id: user.professional.id, health_teams: { active: true })
                        .pluck(:cbo_code, "health_teams.health_unit_id")
                        .select { |cbo, _| clinician?(cbo) }.map(&:last).uniq
      end

      def clinician?(cbo) = Ledi::ConsultationMapping.cid10_allowed?(cbo) || cbo.to_s.start_with?(NURSE_PREFIX)

      def provider_of(user)
        return nil unless user.has_role?("exam_provider")

        provider = ExamProviderUser.find_by(user_id: user.id)&.exam_provider
        provider if provider&.active
      end
    end
  end
  ```

  `app/services/exams/trail.rb`:

  ```ruby
  # Trilha da leitura clínica do exame (ADR 0036; contrato §12): o mesmo
  # clinical_record.viewed do 19a, só ids. A forma administrativa não deixa
  # trilha (não tem conteúdo clínico).
  module Exams
    module Trail
      module_function

      def viewed!(order:, user:, grant:)
        return unless grant&.view == :clinical

        DomainEvents.publish("clinical_record.viewed", patient_id: order.patient_id, user_id: user.id, access: grant.kind.to_s,
                                                       reason_code: grant.opening&.reason_code, exam_order_id: order.id)
      end

      def result_viewed!(result:, user:)
        DomainEvents.publish("clinical_record.viewed", patient_id: result.patient_id, user_id: user.id, access: "exam_result",
                                                       reason_code: nil, exam_result_id: result.id)
      end
    end
  end
  ```

  `app/services/exams/json.rb`:

  ```ruby
  # Formas do contrato §2 (com as Divergências D4, D5 e D16).
  module Exams
    module Json
      module_function

      def order(order, view:, user:, events: true)
        clinical = view == :clinical
        body = {
          id: order.id, short_code: order.short_code,
          patient: { id: order.patient_id, display_name: order.patient.display_name },
          sigtap_code: order.sigtap_code, sigtap_competence: order.sigtap_competence,
          label: Labels.exam(order.sigtap_code, order.sigtap_competence), priority: order.priority,
          cid10_justification: clinical ? order.cid10_justification : nil, origin: order.origin,
          consultation_id: order.consultation_exam_request&.consultation_id,
          requester: { user_id: order.requester_user_id, name: order.requester_display_name, cns: clinical ? order.requester_cns : nil },
          unit: { id: order.unit_id, name: order.unit.name },
          provider: order.exam_provider && { id: order.exam_provider_id, name: order.exam_provider.name },
          scheduled_at: order.scheduled_at&.iso8601, scheduled_place: order.scheduled_place,
          external_reference: order.external_reference, status: order.status, status_changed_at: order.status_changed_at.iso8601,
          overdue: overdue(order), escalated: order.escalated?,
          guide_url: order.guide_file && Files.url_for(order.guide_file, user: user),
          result: order.result && result(order.result, view: view, user: user)
        }
        if events
          body[:events] = order.events.includes(:actor_user, :actor_provider).map { |e| event(e, view: view) }
          body[:result_fields] = result_fields(order) # dashboard D6: o formulário de resultado
        end
        body
      end

      def result_fields(order)
        Array(order.catalog_item&.result_fields).map { |f| f.slice("key", "label", "unit", "loinc") }
      end

      def overdue(order)
        late = order.overdue
        late && { stage: late[:stage], due_on: late[:due_on].iso8601, days_late: late[:days_late] }
      end

      def result(result, view:, user:)
        base = { id: result.id, origin: result.origin, source: result.source, released_on: result.released_on.iso8601,
                 laboratory_name: result.laboratory_name }
        return base unless view == :clinical

        acknowledged = result.exam_order&.events&.select { |e| e.kind == "acknowledged" }&.last
        base.merge(critical: result.critical, values: result.values_list, pdf_url: Files.url_for(result.pdf_file, user: user),
                   acknowledged_at: (acknowledged&.occurred_at || (result.external? ? result.created_at : nil))&.iso8601)
      end

      def event(event, view:)
        actor = if event.actor_provider then { type: "provider", name: event.actor_provider.name }
                elsif event.actor_user then { type: "user", name: Screenings::Json.staff_name(event.actor_user) }
                else { type: "system", name: nil }
                end
        reason = event.kind == "escalated" ? event.escalation_reason : (view == :clinical ? event.reason : nil)
        { kind: event.kind, occurred_at: event.occurred_at.iso8601, actor: actor, reason: reason }
      end

      # Portal do prestador (contrato §2): nunca prontuário — nome, nascimento
      # e os 3 últimos dígitos do CPF.
      def provider_order(order)
        patient = order.patient
        { id: order.id, short_code: order.short_code,
          patient: { name: patient.display_name, birth_date: patient.birth_date, cpf_last3: patient.cpf.to_s[-3, 3] },
          sigtap_code: order.sigtap_code, label: Labels.exam(order.sigtap_code, order.sigtap_competence),
          priority: order.priority, status: order.status, scheduled_at: order.scheduled_at&.iso8601,
          scheduled_place: order.scheduled_place, overdue: overdue(order), result_fields: result_fields(order) }
      end
    end
  end
  ```

- [ ] **Step 4: Run** os três arquivos do Step 1 e `spec/models/exam_tracking_schema_spec.rb` (o `exam_order!` passa a usar `Exams::Open`). Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/open.rb app/services/exams/transition.rb app/services/exams/escalate.rb app/services/exams/access.rb app/services/exams/json.rb app/services/exams/trail.rb app/services/exams/labels.rb spec/services/exams/transition_spec.rb spec/services/exams/access_spec.rb spec/services/exams/json_spec.rb
  git commit -m "feat: add exam order lifecycle, access rules and json shapes

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 8: Pedido da consulta vira exame; adendo cancela

**Files:**
- Modify: `app/services/consultations/items_input.rb` (`exams`: `priority`), `app/commands/consultations/finalize.rb` (`materialize!` + gancho), `app/commands/consultations/add_addendum.rb` (`apply!`)
- Create: `app/services/exams/from_consultation.rb`
- Test: `spec/services/exams/from_consultation_spec.rb`

**Interfaces:**
- Consumes: `Exams::Open`, `Exams::Transition`, `Exams::Gate.usable?`.
- Produces: `Exams::FromConsultation.open!(consultation:, requests:)` (cria um `exam_order` `origin: "consultation"` por pedido `requested`, sem catálogo obrigatório; levanta em falha — está dentro do savepoint da finalização) e `.cancel!(consultation:, sigtap_code:, by:)` (cancela o exame vivo daquele pedido, se ainda sem resultado). Item do rascunho aceita `"priority": "routine"|"urgent"` (ausente = `routine`; outro valor → `invalid_exam` com o índice).

- [ ] **Step 1: Write the failing test** `spec/services/exams/from_consultation_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §4 "Ligação com o 19a": finalizar cria um exam_order por pedido
  # requested, na mesma transação; o adendo que cancela o pedido cancela o
  # exame (se ainda sem resultado). Desligado: nada muda.
  RSpec.describe Exams::FromConsultation do
    before { exam_city! }

    let(:unit) { create_unit("UBS Consulta") }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(40) }

    it "a finalização cria o exame com prioridade, solicitante, unidade e competência" do
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen,
                                             exam_requests: [ { "sigtap_code" => "0202010503", "priority" => "urgent" } ])
      request = consultation.exam_requests.first
      order = ExamOrder.find_by!(consultation_exam_request_id: request.id)
      expect(request.priority).to eq("urgent")
      expect(order).to have_attributes(origin: "consultation", priority: "urgent", requester_user_id: doctor.id, unit_id: unit.id,
                                       patient_id: consultation.patient_id, sigtap_competence: request.sigtap_competence,
                                       status: "awaiting_schedule")
    end

    it "prioridade inválida recusa o rascunho com o índice" do
      consultation = started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      saved = Consultations::SaveDraft.call(consultation: consultation, by: doctor,
                                            params: draft_body(exam_requests: [ { "sigtap_code" => "0202010503", "priority" => "now" } ]))
      expect([ saved.reason, saved.details ]).to eq([ :invalid_exam, { index: 0 } ])
    end

    it "o adendo que tira o exame cancela o exam_order; com resultado, não" do
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      order = ExamOrder.find_by!(patient_id: consultation.patient_id)
      result = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "exame já feito em outro serviço",
                                               changes: { "exam_requests" => [] })
      expect(result).to be_ok
      expect(order.reload.status).to eq("cancelled")
      expect(order.events.last).to have_attributes(kind: "cancelled", actor_user_id: doctor.id)
    end

    it "com exam_tracking desligado a consulta segue como no 19a (Regressão)" do
      Platform::Features.set!(city: Current.city, key: "exam_tracking", enabled: false, maintainer: ledi_maintainer!)
      finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      expect(ExamOrder.count).to eq(0)
    end
  end
  ```

  (Confira a assinatura real de `Consultations::AddAddendum.call` — `consultation:, by:, reason:, text:`/`changes:` — no seed `ClinicalRecordCrew#consult!` e na spec do adendo, e use a mesma; a asserção não muda.)

- [ ] **Step 2: Run** `rspec spec/services/exams/from_consultation_spec.rb`. Expected: FAIL.

- [ ] **Step 3: Implemente.** Em `Consultations::ItemsInput.exams`, depois do bloco da justificativa:

  ```ruby
          priority = raw.key?("priority") ? raw["priority"] : "routine"
          return fail_index(:invalid_exam, index) unless ConsultationExamRequest::PRIORITIES.include?(priority)

          { "sigtap_code" => exam.code, "sigtap_competence" => exam.competence, "cid10_justification" => justification,
            "priority" => priority }
  ```
  (substitui o hash final). Em `Consultations::Finalize.materialize!`, o `ConsultationExamRequest.create!` ganha `priority: exam["priority"] || "routine"`, e no fim, antes do `nil`:

  ```ruby
      Exams::FromConsultation.open!(consultation: consultation, requests: consultation.exam_requests.where(status: "requested").to_a) if tracking
  ```
  com `tracking` lido **fora** da transação em `call` (como `signing`): `tracking = Exams::Gate.usable?(Current.city)` passado para `materialize!(consultation, patient, by, tracking:)`. Em `Consultations::AddAddendum.apply!`: o `create!` dos adicionados ganha `priority: exam["priority"] || "routine"`; os cancelados mantêm a de `previous`; depois do laço de `exams_removed`: `plan[:exams_removed].each { |code| Exams::FromConsultation.cancel!(consultation: consultation, sigtap_code: code, by: by) }`, e depois do laço de `exams_added`, se `Exams::Gate.usable?(Current.city)` (lido em `call`, fora da transação, passado em `plan[:tracking]`): `Exams::FromConsultation.open!(consultation:, requests: <as linhas criadas>)`.

  `app/services/exams/from_consultation.rb`:

  ```ruby
  # Ligação com o 19a (spec §4): o pedido `requested` da consulta vira
  # exam_order (origem consultation, solicitante = autora, unidade do
  # atendimento), sem exigir o catálogo (Desvio 8). Dentro do savepoint da
  # finalização/adendo: falha levanta e desfaz tudo.
  module Exams
    module FromConsultation
      module_function

      def open!(consultation:, requests:)
        unit = consultation.attendance.health_unit
        requests.each do |request|
          result = Open.call(patient: consultation.patient, sigtap_code: request.sigtap_code, priority: request.priority,
                             unit: unit, origin: "consultation", requester_user: consultation.author_user,
                             created_by: consultation.author_user, cid10: request.cid10_justification,
                             consultation_exam_request: request, sigtap_competence: request.sigtap_competence,
                             require_catalog: false)
          raise ActiveRecord::RecordInvalid, request if result.failure?
        end
      end

      def cancel!(consultation:, sigtap_code:, by:)
        order = ExamOrder.joins(:consultation_exam_request)
                         .where(consultation_exam_requests: { consultation_id: consultation.id, sigtap_code: sigtap_code })
                         .where(status: Transition::FLOW["cancelled"][:from]).lock.first
        return unless order

        Transition.call(order: order, kind: "cancelled", actor_user: by, reason: "Cancelado por adendo da consulta")
      end
    end
  end
  ```

  Confira `belongs_to :author_user` em `Consultation` (o `Finalize` usa `consultation.author_user_id`); se a associação não existir, use `User.find(consultation.author_user_id)`.

- [ ] **Step 4: Run** `rspec spec/services/exams/from_consultation_spec.rb spec/commands/consultations spec/services/consultations spec/requests/consultations_spec.rb spec/services/signatures/canonical_spec.rb`. Expected: PASS. O JSON canônico assinado do 19b **não** ganha `priority` (fica fora do que se assina: o esquema `consultation-v1` é fechado e a tag `clinical-v1.2.0` não muda) — se a spec do canônico quebrar, a mudança vazou para `Signatures::Canonical`; desfaça lá.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/consultations/items_input.rb app/commands/consultations/finalize.rb app/commands/consultations/add_addendum.rb app/services/exams/from_consultation.rb spec/services/exams/from_consultation_spec.rb
  git commit -m "feat: turn consultation exam requests into tracked exam orders

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 9: Guia da recepção, leitura, agendar, coletar e cancelar (rotas da unidade)

**Files:**
- Create: `app/services/exams/{cursor,order_query,steps,guide_input}.rb`
- Create: `app/controllers/exams/orders_controller.rb`, `app/controllers/exams/order_steps_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/exams/orders_spec.rb`, `spec/requests/exams/order_steps_spec.rb`

**Interfaces:**
- Consumes: `Exams::{Open,Transition,Access,Json,Trail,Files}`, `Patients::Resolve`, `Professionals::Cns.valid?`, `ClinicalTerms.find("cid10", code)`.
- Produces:
  - `Exams::Cursor.encode(order) -> String`, `.decode(raw) -> [Time, String]|nil` (levanta `Exams::Cursor::Invalid`).
  - `Exams::OrderQuery.call(scope, params, today: Time.zone.today) -> { orders:, next_before: }` com `status[]`, `overdue=true`, `stage`, `provider_id`, `sigtap_code`, `before`, `limit` (422 `invalid_filter` para valor fora da lista).
  - `Exams::Steps.schedule(order:, params:, actor_user: nil, actor_provider: nil) -> Result`, `.collect(order:, actor_user: nil, actor_provider: nil)`, `.cancel(order:, reason:, actor_user:)` — cada um trava o pedido e chama `Transition`; falhas `provider_not_enabled_for_exam`, `invalid_schedule`.
  - `Exams::GuideInput.call(params, user:) -> Result(attrs:)` (falhas `invalid_requester`, `invalid_cns`, `invalid_cid10`, `guide_missing`, `citizen_not_verified`, `not_found`).
  - Rotas `GET/POST /exams/orders`, `GET /exams/orders/:id`, `POST /exams/orders/:id/{schedule,collect,cancel}`.

- [ ] **Step 1: Write the failing tests.** `spec/requests/exams/orders_spec.rb`:

  ```ruby
  require "rails_helper"
  require "json_schemer"

  # Contrato §3 (Divergências D2, D5, D6): a guia da recepção nasce de um par
  # validado; a lista pede um recorte (paciente, unidade ou requester=me) e
  # devolve a visão de quem lê; leitura clínica deixa trilha.
  RSpec.describe "Exames — pedidos", type: :request do
    before { exam_city!; exam_catalog_item! }

    let(:unit) { create_unit("UBS Guia") }
    let(:reception) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(50) }
    let(:schema) { JSONSchemer.schema(Rails.root.join("spec/fixtures/exams/exam-order-v1.json")) }

    def upload!(user)
      ExamFile.create!(content_type: "application/pdf", byte_size: ExamHelpers::PDF_BYTES.bytesize, sha256: "0" * 64,
                       data: Base64.strict_encode64(ExamHelpers::PDF_BYTES), uploaded_by_user: user)
    end

    def guide(over = {})
      { citizen_id: citizen.id, unit_id: unit.id, sigtap_code: "0202010503", priority: "routine",
        requester: { name: "Dra. Ana Externa", cns: Professionals::Cns.generate("externa") },
        external_reference: "GUIA-123", guide_upload_id: upload!(reception).id }.merge(over)
    end

    it "a recepção cadastra a guia: paciente resolvido, solicitante digitado, forma administrativa válida no esquema" do
      sign_in_as(reception)
      json_post "/exams/orders", guide
      expect(response).to have_http_status(:created)
      order = ExamOrder.find(json_body["id"])
      expect(order).to have_attributes(origin: "reception_guide", requester_user_id: nil, requester_name: "Dra. Ana Externa",
                                       unit_id: unit.id, patient_id: citizen.reload.patient_id, status: "awaiting_schedule")
      expect(order.guide_file.consumed?).to be(true)
      expect(json_body["requester"]["cns"]).to be_nil
      expect(json_body["guide_url"]).to start_with("/exams/files/")
      expect(schema.valid?(json_body)).to be(true)
    end

    it "recusa: par declarado 409, catálogo, CNS, guia ausente ou de outro, solicitante sem papel, fora da unidade" do
      sign_in_as(reception)
      declared = Citizen.create!(cpf: "39053344705", phone: "+5541999990001", verification_level: "declared")
      { { citizen_id: declared.id } => [ 409, "citizen_not_verified" ],
        { sigtap_code: "0202020380" } => [ 422, "sigtap_not_in_catalog" ],
        { requester: { name: "Fulano", cns: "123" } } => [ 422, "invalid_cns" ],
        { requester: { user_id: reception.id } } => [ 422, "invalid_requester" ],
        { guide_upload_id: upload!(doctor).id } => [ 422, "guide_missing" ],
        { citizen_id: SecureRandom.uuid } => [ 404, "not_found" ] }.each do |over, (status, error)|
        json_post "/exams/orders", guide(over)
        expect([ response.status, json_body["error"] ]).to eq([ status, error ]), over.inspect
      end
      other_unit = create_unit("UBS Outra")
      sign_in_as(doctor)
      json_post "/exams/orders", guide(unit_id: other_unit.id, guide_upload_id: upload!(doctor).id)
      expect([ response.status, json_body["error"] ]).to eq([ 403, "not_unit_staff" ])
    end

    it "lista por paciente exige leitura do 19a e deixa trilha; requester=me; unidade vê administrativa" do
      patient, = exam_patient!(51)
      order = exam_order!(patient: patient, unit: unit, requester: doctor)
      outsider = doctor!(create_unit("UBS Fora"))
      sign_in_as(outsider)
      get "/exams/orders", params: { patient_id: patient.id }
      expect([ response.status, json_body["error"] ]).to eq([ 403, "out_of_context" ])
      sign_in_as(doctor)
      get "/exams/orders", params: { requester: "me" }
      expect(json_body["orders"].map { |o| o["id"] }).to eq([ order.id ])
      expect(json_body["orders"].first).not_to have_key("events")
      get "/exams/orders/#{order.id}"
      expect(json_body["events"].map { |e| e["kind"] }).to eq(%w[requested])
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload)
        .to include("access" => "author", "exam_order_id" => order.id, "user_id" => doctor.id)
      sign_in_as(reception)
      get "/exams/orders", params: { unit_id: unit.id }
      expect(json_body["orders"].first["result"]).to be_nil
      get "/exams/orders"
      expect([ response.status, json_body["error"] ]).to eq([ 422, "filter_required" ])
      get "/exams/orders", params: { unit_id: unit.id, status: [ "bogus" ] }
      expect([ response.status, json_body["error"] ]).to eq([ 422, "invalid_filter" ])
      get "/exams/orders/#{SecureRandom.uuid}"
      expect(response).to have_http_status(:not_found)
    end

    it "recorte por consulta (autora ou em contexto), com consultation_id no exame" do
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(exam_seq!))
      sign_in_as(doctor)
      get "/exams/orders", params: { consultation_id: consultation.id }
      expect(json_body["orders"].map { |o| o["consultation_id"] }).to eq([ consultation.id ])
      sign_in_as(doctor!(create_unit("UBS Alheia")))
      get "/exams/orders", params: { consultation_id: consultation.id }
      expect([ response.status, json_body["error"] ]).to eq([ 403, "out_of_context" ])
    end

    it "pagina por cursor sem repetir nem perder" do
      patient, = exam_patient!(52)
      ids = 5.times.map { exam_order!(patient: patient, unit: unit, requester: doctor).id }
      sign_in_as(reception)
      seen = []
      before = nil
      loop do
        get "/exams/orders", params: { unit_id: unit.id, limit: 2, before: before }.compact
        seen += json_body["orders"].map { |o| o["id"] }
        before = json_body["next_before"]
        break unless before
      end
      expect(seen.sort).to eq(ids.sort)
      expect(seen.uniq.size).to eq(5)
    end
  end
  ```

  Copie o esquema commitado no contracts (Task 1) para `spec/fixtures/exams/exam-order-v1.json` (cópia byte a byte; a Task 14 prende o SHA-256 dos dois esquemas).

  `spec/requests/exams/order_steps_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3: agendar (prestador habilitado), coletar, cancelar com motivo;
  # 409 com o status atual fora da ordem.
  RSpec.describe "Exames — etapas pela unidade", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Etapas") }
    let(:doctor) { doctor!(unit) }
    let(:reception) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }
    let(:item) { exam_catalog_item! }
    let(:provider) { exam_provider!.tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }
    let(:order) { exam_order!(patient: exam_patient!(60).first, unit: unit, requester: doctor) }

    it "agenda, reagenda, coleta; fora da ordem 409; prestador não habilitado 422" do
      sign_in_as(reception)
      json_post "/exams/orders/#{order.id}/collect"
      expect(json_body).to eq("error" => "invalid_transition", "status" => "awaiting_schedule")
      json_post "/exams/orders/#{order.id}/schedule", provider_id: exam_provider!(name: "Outro", items: []).id,
                                                      scheduled_at: 2.days.from_now.iso8601, scheduled_place: "Rua B"
      expect([ response.status, json_body["error"] ]).to eq([ 422, "provider_not_enabled_for_exam" ])
      json_post "/exams/orders/#{order.id}/schedule", provider_id: provider.id, scheduled_at: "amanhã", scheduled_place: "Rua B"
      expect(json_body["error"]).to eq("invalid_schedule")
      json_post "/exams/orders/#{order.id}/schedule", provider_id: provider.id, scheduled_at: 2.days.from_now.iso8601,
                                                      scheduled_place: "Laboratório Central, Rua B, 100"
      expect(json_body).to include("status" => "scheduled", "provider" => { "id" => provider.id, "name" => provider.name })
      json_post "/exams/orders/#{order.id}/collect"
      expect(json_body["status"]).to eq("collected")
    end

    it "cancela com motivo (unidade ou solicitante); sem motivo 422; estranho 403" do
      sign_in_as(doctor!(create_unit("UBS Longe")))
      json_post "/exams/orders/#{order.id}/cancel", reason: "Engano"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(doctor)
      json_post "/exams/orders/#{order.id}/cancel", reason: ""
      expect(json_body["error"]).to eq("reason_required")
      json_post "/exams/orders/#{order.id}/cancel", reason: "Pedido em duplicidade"
      expect(json_body["status"]).to eq("cancelled")
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL.

- [ ] **Step 3: Implemente.** `app/services/exams/cursor.rb`:

  ```ruby
  # Cursor das listas de exame: (created_at, id) decrescentes.
  module Exams
    module Cursor
      class Invalid < StandardError; end

      module_function

      def encode(order) = Base64.urlsafe_encode64("#{order.created_at.utc.iso8601(6)}|#{order.id}", padding: false)

      def decode(raw)
        return nil if raw.blank?
        raise Invalid unless raw.is_a?(String)

        at, id = Base64.urlsafe_decode64(raw).split("|", 2)
        raise Invalid unless id&.match?(BaseController::UUID)

        [ Time.iso8601(at), id ]
      rescue ArgumentError
        raise Invalid
      end
    end
  end
  ```

  `app/services/exams/order_query.rb`:

  ```ruby
  # Filtros e página das listas de exame (contrato §3; F-21.5 usa overdue/stage).
  module Exams
    module OrderQuery
      class InvalidFilter < StandardError; end

      module_function

      def call(scope, params, today: Time.zone.today)
        scope = filter(scope, params, today)
        at, id = Cursor.decode(params[:before])
        scope = scope.where("(exam_orders.created_at, exam_orders.id) < (?, ?)", at, id) if at
        limit = Timeline::Query.limit(params[:limit])
        rows = scope.includes(:patient, :unit, :exam_provider, :requester_user, :consultation_exam_request, result: :pdf_file)
                    .order(created_at: :desc, id: :desc).limit(limit + 1).to_a
        page = rows.first(limit)
        { orders: page, next_before: rows.size > limit ? Cursor.encode(page.last) : nil }
      end

      def filter(scope, params, today)
        statuses = params[:status]
        if statuses.present?
          raise InvalidFilter unless statuses.is_a?(Array) && (statuses - ExamOrder::STATUSES).empty?

          scope = scope.where(status: statuses)
        end
        if params[:stage].present?
          raise InvalidFilter unless Deadlines::STAGES.include?(params[:stage])

          scope = scope.where(stage: params[:stage])
        end
        scope = scope.where(status: ExamOrder::OPEN_STATUSES).where("exam_orders.stage_due_on < ?", today) if params[:overdue] == "true"
        if params[:provider_id].present?
          raise InvalidFilter unless params[:provider_id].to_s.match?(BaseController::UUID)

          scope = scope.where(exam_provider_id: params[:provider_id])
        end
        if params[:sigtap_code].present?
          raise InvalidFilter unless params[:sigtap_code].to_s.match?(/\A02\d{8}\z/)

          scope = scope.where(sigtap_code: params[:sigtap_code])
        end
        scope
      end
      private_class_method :filter
    end
  end
  ```

  `app/services/exams/steps.rb`:

  ```ruby
  # Agendar, coletar e cancelar (contrato §3, §6): a mesma regra para a
  # unidade e para o prestador. Trava o pedido; a transição faz o resto.
  module Exams
    module Steps
      module_function

      def schedule(order:, params:, actor_user: nil, actor_provider: nil)
        provider = actor_provider || (params[:provider_id].is_a?(String) && ExamProvider.find_by(id: params[:provider_id], active: true))
        return Result.fail(:provider_not_enabled_for_exam) unless provider && enabled?(provider, order)

        at = parse_time(params[:scheduled_at])
        place = params[:scheduled_place].is_a?(String) ? params[:scheduled_place].strip : ""
        return Result.fail(:invalid_schedule) unless at && place.length.between?(1, 200)

        locked(order) do
          next Result.fail(:provider_not_enabled_for_exam) if order.exam_provider_id && order.exam_provider_id != provider.id

          Transition.call(order: order, kind: "scheduled", actor_user: actor_user, actor_provider: actor_provider,
                          changes: { exam_provider_id: provider.id, scheduled_at: at, scheduled_place: place })
        end
      end

      def collect(order:, actor_user: nil, actor_provider: nil)
        locked(order) { Transition.call(order: order, kind: "collected", actor_user: actor_user, actor_provider: actor_provider) }
      end

      def cancel(order:, reason:, actor_user:)
        locked(order) { Transition.call(order: order, kind: "cancelled", actor_user: actor_user, reason: reason) }
      end

      # Prestador faz o exame: tem o item, ou o exame não está no catálogo.
      def enabled?(provider, order)
        item = ExamCatalogItem.find_by(sigtap_code: order.sigtap_code)
        item.nil? || ExamProviderItem.exists?(exam_provider_id: provider.id, exam_catalog_item_id: item.id)
      end

      def locked(order)
        ApplicationRecord.transaction do
          order.lock!
          yield
        end
      end

      def parse_time(raw)
        raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}T/) ? Time.iso8601(raw) : nil
      rescue ArgumentError
        nil
      end
      private_class_method :locked, :parse_time
    end
  end
  ```

  `app/services/exams/guide_input.rb`:

  ```ruby
  # Corpo da guia da recepção (contrato §3; Divergência D2) → atributos de
  # Exams::Open. Paciente pelo par validado (Patients::Resolve) ou pelo id.
  module Exams
    module GuideInput
      NAME = (3..120)

      module_function

      def call(params, user:)
        patient = patient_from(params)
        return patient if patient.is_a?(Result)

        unit = params[:unit_id].is_a?(String) && HealthUnit.find_by(id: params[:unit_id], active: true)
        return Result.fail(:not_found) unless unit
        return Result.fail(:not_unit_staff) unless Access.unit_staff?(user, unit.id)

        requester = requester(params[:requester])
        return requester if requester.is_a?(Result)

        cid10 = nil
        if params[:cid10_justification].present?
          cid10 = ClinicalTerms.find("cid10", params[:cid10_justification])&.code
          return Result.fail(:invalid_cid10) unless cid10
        end
        reference = params[:external_reference]
        return Result.fail(:invalid_external_reference) unless reference.nil? || (reference.is_a?(String) && reference.length <= 60)

        Result.ok(attrs: { patient: patient, unit: unit, sigtap_code: params[:sigtap_code].to_s, priority: params[:priority].presence || "routine",
                           cid10: cid10, external_reference: reference.presence, **requester })
      end

      def patient_from(params)
        if params[:citizen_id].present?
          citizen = params[:citizen_id].is_a?(String) && Citizen.not_erased.find_by(id: params[:citizen_id])
          return Result.fail(:not_found) unless citizen

          resolved = Patients::Resolve.call(citizen)
          return Result.fail(:citizen_not_verified) if resolved.failure?

          resolved.payload[:patient]
        else
          (params[:patient_id].is_a?(String) && Patient.find_by(id: params[:patient_id])) || Result.fail(:not_found)
        end
      end

      def requester(raw)
        raw = raw.respond_to?(:to_unsafe_h) ? raw.to_unsafe_h : raw
        return Result.fail(:invalid_requester) unless raw.is_a?(Hash)

        raw = raw.stringify_keys
        if raw["user_id"].present?
          user = User.find_by(id: raw["user_id"], deactivated_at: nil)
          return Result.fail(:invalid_requester) unless user&.has_role?("health_professional")

          { requester_user: user }
        else
          name = raw["name"].is_a?(String) ? raw["name"].strip : ""
          return Result.fail(:invalid_requester) unless NAME.cover?(name.length)
          return Result.fail(:invalid_cns) unless Professionals::Cns.valid?(raw["cns"])

          { requester_name: name, requester_cns: raw["cns"] }
        end
      end
      private_class_method :patient_from, :requester
    end
  end
  ```

  `app/controllers/exams/orders_controller.rb`:

  ```ruby
  # GET/POST /exams/orders e GET /exams/orders/:id (contrato §3; D2, D5, D6).
  module Exams
    class OrdersController < BaseController
      def index
        scope, view = scope_and_view
        return if performed?

        page = Exams::OrderQuery.call(scope, params)
        render json: { orders: page[:orders].map { |o| Exams::Json.order(o, view: view, user: Current.user, events: false) },
                       next_before: page[:next_before] }
      rescue Exams::OrderQuery::InvalidFilter
        render json: { error: "invalid_filter" }, status: :unprocessable_entity
      rescue Exams::Cursor::Invalid
        render json: { error: "invalid_cursor" }, status: :unprocessable_entity
      end

      def show
        order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id])
        return not_found unless order

        grant = Exams::Access.for_order(user: Current.user, order: order)
        return not_found unless grant

        Exams::Trail.viewed!(order: order, user: Current.user, grant: grant)
        render json: Exams::Json.order(order, view: grant.view, user: Current.user)
      end

      def create
        input = Exams::GuideInput.call(params, user: Current.user)
        return render_failure(input) if input.failure?

        result = nil
        ApplicationRecord.transaction do
          file = Exams::Files.claim!(file_id: params[:guide_upload_id], user: Current.user)
          next result = Result.fail(:guide_missing) unless file

          result = Exams::Open.call(**input.payload[:attrs], origin: "reception_guide", guide_file: file, created_by: Current.user)
          raise ActiveRecord::Rollback if result.failure?
        end
        return render_failure(result) if result.failure?

        render json: Exams::Json.order(result.payload[:order], view: :administrative, user: Current.user), status: :created
      end

      private

      # [scope, view]; senão a resposta de erro já saiu. Recortes: consulta
      # (dashboard D6), paciente, requester=me, unidade, ou — só admin e
      # regulador — atrasos da cidade inteira (dashboard D15).
      def scope_and_view
        if params[:consultation_id].present?
          consultation = uuid?(params[:consultation_id]) && Consultation.find_by(id: params[:consultation_id])
          return not_found unless consultation

          grant = ClinicalRecord::Access.for_consultation(user: Current.user, consultation: consultation)
          return forbid(grant.reason.to_s) unless grant.allowed?

          ClinicalRecord::Trail.viewed!(patient: consultation.patient, user: Current.user, grant: grant)
          [ ExamOrder.joins(:consultation_exam_request).where(consultation_exam_requests: { consultation_id: consultation.id }), :clinical ]
        elsif params[:patient_id].present?
          patient = uuid?(params[:patient_id]) && Patient.find_by(id: params[:patient_id])
          return not_found unless patient

          grant = ClinicalRecord::Access.call(user: Current.user, patient: patient)
          return forbid(grant.reason.to_s) unless grant.allowed?

          ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant)
          [ ExamOrder.where(patient_id: patient.id), :clinical ]
        elsif params[:requester] == "me"
          return forbid("missing_role") unless Current.user.has_role?("health_professional")

          [ ExamOrder.where(requester_user_id: Current.user.id), :clinical ]
        elsif params[:unit_id].present?
          return forbid("not_unit_staff") unless uuid?(params[:unit_id]) && Exams::Access.unit_staff?(Current.user, params[:unit_id])

          [ ExamOrder.where(unit_id: params[:unit_id]), :administrative ]
        elsif params[:overdue] == "true" && Exams::Access.regulator?(Current.user)
          [ ExamOrder.all, :administrative ]
        else
          render json: { error: "filter_required" }, status: :unprocessable_entity
        end
      end
    end
  end
  ```


  `app/controllers/exams/order_steps_controller.rb`:

  ```ruby
  # POST /exams/orders/:id/{schedule,collect,cancel} pela unidade (contrato §3).
  # Cancelar: unidade ou solicitante.
  module Exams
    class OrderStepsController < BaseController
      before_action :load_order

      def schedule
        return forbid("not_unit_staff") unless unit_staff?

        respond(Exams::Steps.schedule(order: @order, params: params, actor_user: Current.user))
      end

      def collect
        return forbid("not_unit_staff") unless unit_staff?

        respond(Exams::Steps.collect(order: @order, actor_user: Current.user))
      end

      def cancel
        return forbid("not_unit_staff") unless unit_staff? || @order.requester_user_id == Current.user.id

        respond(Exams::Steps.cancel(order: @order, reason: params[:reason], actor_user: Current.user))
      end

      private

      def unit_staff? = Exams::Access.unit_staff?(Current.user, @order.unit_id)

      def load_order
        @order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id])
        not_found unless @order
      end

      def respond(result)
        return render_failure(result) if result.failure?

        grant = Exams::Access.for_order(user: Current.user, order: @order.reload)
        render json: Exams::Json.order(@order, view: grant&.view || :administrative, user: Current.user)
      end
    end
  end
  ```

  Rotas dentro de `scope "/exams"`:

  ```ruby
    get  "orders",              to: "orders#index"
    post "orders",              to: "orders#create"
    get  "orders/:id",          to: "orders#show"
    post "orders/:id/schedule", to: "order_steps#schedule"
    post "orders/:id/collect",  to: "order_steps#collect"
    post "orders/:id/cancel",   to: "order_steps#cancel"
  ```

- [ ] **Step 4: Run** os dois arquivos do Step 1. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/cursor.rb app/services/exams/order_query.rb app/services/exams/steps.rb app/services/exams/guide_input.rb app/controllers/exams/orders_controller.rb app/controllers/exams/order_steps_controller.rb config/routes.rb spec/fixtures/exams/exam-order-v1.json spec/requests/exams/orders_spec.rb spec/requests/exams/order_steps_spec.rb
  git commit -m "feat: register reception guides and schedule, collect or cancel exams

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.3 → Done (lado api).

---

## Fatia 4 — Regulação, prazos e portal (F-21.4, F-21.5, F-21.6)

### Task 10: Fila de regulação

**Files:**
- Create: `app/controllers/exams/regulation_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/exams/regulation_spec.rb`

**Interfaces:**
- Consumes: `Exams::{Access.regulator?, Transition, Steps.enabled?, Json}`.
- Produces: `GET /exams/regulation/queue?before=&limit=` (urgente primeiro, depois o mais antigo; cursor `"<rank>|<created_at>|<id>"` em Base64 url-safe), `POST /exams/orders/:id/authorize { provider_id }`, `POST /exams/orders/:id/deny { reason }`.

- [ ] **Step 1: Write the failing test** `spec/requests/exams/regulation_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4 (F-21.4): exam_regulator ou municipal_admin; urgente primeiro,
  # depois o mais antigo; autorizar exige prestador habilitado; negar exige
  # motivo e fecha o exame; o motivo nunca vai ao cidadão (Task 17).
  RSpec.describe "Exames — regulação", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Regulação") }
    let(:doctor) { doctor!(unit) }
    let(:item) { exam_catalog_item!(requires_authorization: true) }
    let(:provider) { exam_provider!.tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }
    let(:regulator) { staff_with("reg-#{SecureRandom.hex(2)}@cidade.gov.br", "exam_regulator") }

    def order!(priority:, at:)
      travel_to(at) { exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor, priority: priority) }
    end

    it "ordena urgente → mais antigo; autoriza com prestador; nega com motivo" do
      item
      old_routine = order!(priority: "routine", at: 3.days.ago)
      new_urgent = order!(priority: "urgent", at: 1.hour.ago)
      old_urgent = order!(priority: "urgent", at: 2.days.ago)
      sign_in_as(regulator)
      get "/exams/regulation/queue", params: { limit: 2 }
      expect(json_body["orders"].map { |o| o["id"] }).to eq([ old_urgent.id, new_urgent.id ])
      get "/exams/regulation/queue", params: { limit: 2, before: json_body["next_before"] }
      expect(json_body["orders"].map { |o| o["id"] }).to eq([ old_routine.id ])
      expect(json_body["next_before"]).to be_nil

      json_post "/exams/orders/#{old_urgent.id}/authorize", provider_id: exam_provider!(name: "Sem item", items: []).id
      expect(json_body["error"]).to eq("provider_not_enabled_for_exam")
      json_post "/exams/orders/#{old_urgent.id}/authorize", provider_id: provider.id
      expect(json_body).to include("status" => "awaiting_schedule", "provider" => { "id" => provider.id, "name" => provider.name })
      json_post "/exams/orders/#{new_urgent.id}/deny", reason: ""
      expect(json_body["error"]).to eq("reason_required")
      json_post "/exams/orders/#{new_urgent.id}/deny", reason: "Sem indicação clínica descrita"
      expect(json_body["status"]).to eq("denied")
      json_post "/exams/orders/#{new_urgent.id}/authorize", provider_id: provider.id
      expect([ response.status, json_body["status"] ]).to eq([ 409, "denied" ])
    end

    it "só regulador ou admin municipal" do
      sign_in_as(doctor)
      get "/exams/regulation/queue"
      expect([ response.status, json_body["error"] ]).to eq([ 403, "missing_role" ])
      sign_in_as(municipal_admin!)
      get "/exams/regulation/queue"
      expect(response).to have_http_status(:ok)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/requests/exams/regulation_spec.rb`. Expected: FAIL.

- [ ] **Step 3: Implemente** `app/controllers/exams/regulation_controller.rb`:

  ```ruby
  # Fila de regulação (ADR 0036; contrato §4): exam_regulator ou municipal_admin.
  module Exams
    class RegulationController < BaseController
      RANK = Arel.sql("CASE exam_orders.priority WHEN 'urgent' THEN 0 ELSE 1 END")

      before_action { forbid("missing_role") unless Exams::Access.regulator?(Current.user) }
      before_action :load_order, only: %i[authorize deny]

      def queue
        scope = ExamOrder.where(status: "awaiting_authorization")
        if (cursor = decode(params[:before]))
          scope = scope.where("(#{RANK}, exam_orders.created_at, exam_orders.id) > (?, ?, ?)", *cursor)
        end
        limit = limit_param
        rows = scope.includes(:patient, :unit, :requester_user).order(RANK, :created_at, :id).limit(limit + 1).to_a
        page = rows.first(limit)
        render json: { orders: page.map { |o| Exams::Json.order(o, view: :administrative, user: Current.user, events: false) },
                       next_before: rows.size > limit ? encode(page.last) : nil }
      rescue ArgumentError
        render json: { error: "invalid_cursor" }, status: :unprocessable_entity
      end

      def authorize
        provider = params[:provider_id].is_a?(String) && ExamProvider.find_by(id: params[:provider_id], active: true)
        return render_failure(Result.fail(:provider_not_enabled_for_exam)) unless provider && Exams::Steps.enabled?(provider, @order)

        respond(step("authorized", changes: { exam_provider_id: provider.id }))
      end

      def deny = respond(step("denied", reason: params[:reason]))

      private

      def step(kind, **opts)
        ApplicationRecord.transaction do
          @order.lock!
          Exams::Transition.call(order: @order, kind: kind, actor_user: Current.user, **opts)
        end
      end

      def respond(result)
        return render_failure(result) if result.failure?

        render json: Exams::Json.order(@order.reload, view: :administrative, user: Current.user)
      end

      def load_order
        @order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id])
        not_found unless @order
      end

      def encode(order)
        rank = order.priority == "urgent" ? 0 : 1
        Base64.urlsafe_encode64("#{rank}|#{order.created_at.utc.iso8601(6)}|#{order.id}", padding: false)
      end

      def decode(raw)
        return nil if raw.blank?
        raise ArgumentError unless raw.is_a?(String)

        rank, at, id = Base64.urlsafe_decode64(raw).split("|", 3)
        raise ArgumentError unless %w[0 1].include?(rank) && id.to_s.match?(UUID)

        [ rank.to_i, Time.iso8601(at), id ]
      end
    end
  end
  ```

  Rotas dentro de `scope "/exams"`: `get "regulation/queue", to: "regulation#queue"`, `post "orders/:id/authorize", to: "regulation#authorize"`, `post "orders/:id/deny", to: "regulation#deny"`.

- [ ] **Step 4: Run.** Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/controllers/exams/regulation_controller.rb config/routes.rb spec/requests/exams/regulation_spec.rb
  git commit -m "feat: add exam regulation queue with authorize and deny

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.4 → Done (lado api).

### Task 11: Atrasos e job diário (escalonamento por prazo e purga de órfãos)

**Files:**
- Create: `app/jobs/exams/daily_job.rb`
- Modify: `config/recurring.yml`
- Test: `spec/jobs/exams/daily_job_spec.rb`, `spec/requests/exams/overdue_spec.rb`

**Interfaces:**
- Consumes: `Exams::Escalate`, `Exams::OrderQuery` (filtros `overdue`, `stage`, `provider_id`, `sigtap_code` — Task 9), `ExamOrder#overdue`.
- Produces: `Exams::DailyJob#perform(now: Time.current)` — escala (`overdue`) todo exame `resulted` sem escalonamento com `stage_due_on < hoje` (fuso da cidade); apaga `exam_files` não consumidos com mais de 24 h; idempotente.

- [ ] **Step 1: Write the failing tests.** `spec/jobs/exams/daily_job_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5 "Alertas" (F-21.5): job diário por cidade; ciência vencida vai à
  # fila da equipe uma vez só; órfãos de 24 h somem. Review Focus 5.
  RSpec.describe Exams::DailyJob do
    before { exam_city! }
    # O job roda no fuso da cidade (CityConnection.with); o arranjo também.
    around { |example| Time.use_zone("America/Sao_Paulo") { example.run } }

    let(:unit) { create_unit("UBS Prazo") }
    let(:doctor) { doctor!(unit) }

    def resulted_order(at:)
      travel_to(at) do
        order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor)
        ApplicationRecord.transaction do
          order.lock!
          %w[scheduled collected resulted].each do |kind|
            Exams::Transition.call(order: order, kind: kind, actor_user: doctor, event_payload: kind == "resulted" ? { source: "unit", critical: false } : {})
          end
        end
        order.reload
      end
    end

    it "escala a ciência vencida no dia seguinte ao prazo, uma vez só, mesmo rodando duas vezes" do
      zone = ActiveSupport::TimeZone["America/Sao_Paulo"]
      order = resulted_order(at: zone.local(2026, 10, 1, 23, 59))
      expect(order.stage_due_on).to eq(Date.new(2026, 10, 16))
      described_class.perform_now(now: zone.local(2026, 10, 16, 23, 0))
      expect(order.reload.escalated?).to be(false)
      described_class.perform_now(now: zone.local(2026, 10, 17, 6, 30))
      described_class.perform_now(now: zone.local(2026, 10, 17, 7, 0))
      expect(order.reload.events.where(kind: "escalated").pluck(:escalation_reason)).to eq(%w[overdue])
    end

    it "apaga só os arquivos órfãos com mais de 24 h" do
      old = travel_to(25.hours.ago) { pdf_file!(user: doctor) }
      fresh = pdf_file!(user: doctor)
      used = travel_to(30.hours.ago) { pdf_file!(user: doctor) }.tap { |f| f.update!(consumed_at: Time.current) }
      described_class.perform_now
      expect(ExamFile.where(id: [ old.id, fresh.id, used.id ]).pluck(:id)).to contain_exactly(fresh.id, used.id)
    end

    it "com exam_tracking desligado não escala (mas purga)" do
      order = resulted_order(at: 40.days.ago)
      Platform::Features.set!(city: Current.city, key: "exam_tracking", enabled: false, maintainer: ledi_maintainer!)
      described_class.perform_now
      expect(order.reload.escalated?).to be(false)
    end

    it "agendado todo dia às 6h30 em cada cidade (config/recurring.yml)" do
      entry = YAML.load_file(Rails.root.join("config/recurring.yml"), aliases: true).dig("production", "exams_daily")
      expect(entry).to eq("class" => "Exams::DailyJob", "queue" => "housekeeping", "schedule" => "every day at 6:30am America/Sao_Paulo")
      expect(described_class.ancestors).to include(EachCityJob)
    end
  end
  ```

  `spec/requests/exams/overdue_spec.rb`:

  ```ruby
  require "rails_helper"

  # Painel "Exames atrasados" (spec §5): a unidade filtra por etapa,
  # prestador e exame; cada pedido diz a etapa, o prazo e os dias de atraso.
  RSpec.describe "Exames — atrasos", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Atrasos") }
    let(:doctor) { doctor!(unit) }

    it "filtra os atrasados e mostra etapa, prazo e dias" do
      late = travel_to(20.days.ago) { exam_order!(patient: exam_patient!(70).first, unit: unit, requester: doctor) }
      exam_order!(patient: exam_patient!(71).first, unit: unit, requester: doctor)
      sign_in_as(staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier"))
      get "/exams/orders", params: { unit_id: unit.id, overdue: "true", stage: "schedule", sigtap_code: "0202010503" }
      expect(json_body["orders"].map { |o| o["id"] }).to eq([ late.id ])
      expect(json_body["orders"].first["overdue"]).to eq("stage" => "schedule", "due_on" => late.stage_due_on.iso8601, "days_late" => 5)
      get "/exams/orders", params: { unit_id: unit.id, overdue: "true", stage: "result" }
      expect(json_body["orders"]).to eq([])
    end

    it "coordenação: admin e regulador veem os atrasos da cidade inteira; a recepção não (dashboard D15)" do
      late = travel_to(20.days.ago) { exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor) }
      sign_in_as(staff_with("reg-#{SecureRandom.hex(2)}@cidade.gov.br", "exam_regulator"))
      get "/exams/orders", params: { overdue: "true" }
      expect(json_body["orders"].map { |o| [ o["id"], o["unit"]["name"] ] }).to eq([ [ late.id, "UBS Atrasos" ] ])
      get "/exams/orders"
      expect(json_body["error"]).to eq("filter_required")
      sign_in_as(staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier"))
      get "/exams/orders", params: { overdue: "true" }
      expect(json_body["error"]).to eq("filter_required")
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL (`Exams::DailyJob` indefinido; o request pode já passar — os filtros vêm da Task 9; ótimo).

- [ ] **Step 3: Implemente** `app/jobs/exams/daily_job.rb`:

  ```ruby
  # Job diário do rastreamento (ADR 0036; spec §5): ciência vencida vai à fila
  # da equipe (escalated overdue, uma vez só); arquivos nunca usados com mais
  # de 24 h somem (guia/laudo abandonados no upload). Dias no fuso da cidade
  # (CityConnection.with → Time.zone).
  module Exams
    class DailyJob < ApplicationJob
      prepend EachCityJob
      queue_as :housekeeping

      ORPHAN_TTL = 24.hours

      def perform(now: Time.current)
        ExamFile.where(consumed_at: nil).where("created_at < ?", now - ORPHAN_TTL).delete_all
        return unless Exams::Gate.usable?(Current.city)

        today = now.in_time_zone.to_date
        ExamOrder.where(status: "resulted", escalated_at: nil).where("stage_due_on < ?", today).find_each do |order|
          ApplicationRecord.transaction do
            order.lock!
            Exams::Escalate.call(order: order, reason: "overdue", now: now) if order.status == "resulted"
          end
        end
      end
    end
  end
  ```

  Em `config/recurring.yml`, depois de `signature_sweep`:

  ```yaml
    # ADR 0036 (spec §5): ciência vencida → fila da equipe; purga de arquivos órfãos.
    exams_daily:
      class: Exams::DailyJob
      queue: housekeeping
      schedule: "every day at 6:30am America/Sao_Paulo"
  ```

- [ ] **Step 4: Run** os dois e `rspec spec/config/solid_queue_configuration_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/jobs/exams/daily_job.rb config/recurring.yml spec/jobs/exams/daily_job_spec.rb spec/requests/exams/overdue_spec.rb
  git commit -m "feat: escalate overdue exam acknowledgements and purge orphan exam files daily

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.5 → Done (lado api).

### Task 12: Portal do prestador

**Files:**
- Create: `app/controllers/exams/provider/orders_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/exams/provider/orders_spec.rb`

**Interfaces:**
- Consumes: `Exams::Access.provider_of`, `Exams::OrderQuery`, `Exams::Steps`, `Exams::Json.provider_order`.
- Produces: `Exams::Provider::OrdersController` com `index`, `schedule`, `collect` (o `result` entra na Task 13) e `#provider_order!` (404 `not_found` para guia de outro prestador); cabeçalho `X-Overdue-Count`.

- [ ] **Step 1: Write the failing test** `spec/requests/exams/provider/orders_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §6 (F-21.6): o prestador só vê e mexe nas guias dele (404 para a
  # de outro, sem distinguir de inexistente); nunca prontuário; atrasos no cabeçalho.
  RSpec.describe "Portal do prestador", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Portal") }
    let(:doctor) { doctor!(unit) }
    let(:item) { exam_catalog_item! }
    let(:mine) { exam_provider!(name: "Lab A").tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }
    let(:theirs) { exam_provider!(name: "Lab B").tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }
    let(:lab_user) do
      staff_with("lab-#{SecureRandom.hex(2)}@lab.com.br", "exam_provider").tap { |u| ExamProviderUser.create!(exam_provider: mine, user: u) }
    end

    def scheduled_for(provider, n, at: Time.current)
      travel_to(at) do
        order = exam_order!(patient: exam_patient!(n).first, unit: unit, requester: doctor)
        Exams::Steps.schedule(order: order, actor_user: doctor,
                              params: { provider_id: provider.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
        order.reload
      end
    end

    it "lista só as guias do prestador, com nome, nascimento e 3 dígitos do CPF; atrasos no cabeçalho" do
      own = scheduled_for(mine, 80)
      late = scheduled_for(mine, 81, at: 40.days.ago)
      other = scheduled_for(theirs, 82)
      sign_in_as(lab_user)
      get "/exams/provider/orders"
      expect(json_body["orders"].map { |o| o["id"] }).to contain_exactly(own.id, late.id)
      expect(response.headers["X-Overdue-Count"]).to eq("1")
      row = json_body["orders"].find { |o| o["id"] == own.id }
      expect(row["patient"].keys).to eq(%w[name birth_date cpf_last3])
      expect(row["result_fields"].map { |f| f["key"] }).to eq(%w[hba1c])
      expect(json_body["overdue_count"]).to eq(1)
      expect(row["patient"]["cpf_last3"]).to eq(own.patient.cpf[-3, 3])
      expect(response.body).not_to include(own.patient.cpf)
      json_post "/exams/provider/orders/#{other.id}/collect"
      expect([ response.status, json_body["error"] ]).to eq([ 404, "not_found" ])
      json_post "/exams/provider/orders/#{SecureRandom.uuid}/collect"
      expect([ response.status, json_body["error"] ]).to eq([ 404, "not_found" ])
      json_post "/exams/provider/orders/#{own.id}/collect"
      expect(json_body["status"]).to eq("collected")
      expect(own.reload.events.last).to have_attributes(kind: "collected", actor_user_id: lab_user.id)
    end

    it "agenda a guia autorizada para ele; nunca lê pedido pelas rotas da unidade" do
      authorized = travel_to(1.hour.ago) { exam_order!(patient: exam_patient!(83).first, unit: unit, requester: doctor) }
      ApplicationRecord.transaction do
        authorized.lock!
        ExamOrderEvent.create!(exam_order: authorized, kind: "authorized", actor_user: doctor, occurred_at: Time.current)
        authorized.update!(exam_provider_id: mine.id)
      end
      sign_in_as(lab_user)
      json_post "/exams/provider/orders/#{authorized.id}/schedule", scheduled_at: 2.days.from_now.iso8601, scheduled_place: "Lab A, sala 2"
      expect(json_body).to include("status" => "scheduled", "scheduled_place" => "Lab A, sala 2")
      get "/exams/orders/#{authorized.id}"
      expect(response).to have_http_status(:not_found)
      get "/exams/orders", params: { unit_id: unit.id }
      expect(response).to have_http_status(:forbidden)
    end

    it "sem vínculo a prestador: 403 missing_role" do
      sign_in_as(staff_with("lab-#{SecureRandom.hex(2)}@lab.com.br", "exam_provider"))
      get "/exams/provider/orders"
      expect([ response.status, json_body["error"] ]).to eq([ 403, "missing_role" ])
    end
  end
  ```

- [ ] **Step 2: Run.** Expected: FAIL.

- [ ] **Step 3: Implemente** `app/controllers/exams/provider/orders_controller.rb`:

  ```ruby
  # Portal do prestador (ADR 0036; contrato §6): só as guias do prestador do
  # usuário (isolamento no servidor; 404 para a de outro). Nunca prontuário.
  module Exams
    module Provider
      class OrdersController < Exams::BaseController
        VISIBLE = %w[awaiting_schedule scheduled collected resulted acknowledged].freeze

        before_action :load_provider
        before_action :provider_order!, except: :index

        def index
          scope = ExamOrder.where(exam_provider_id: @provider.id, status: VISIBLE)
          page = Exams::OrderQuery.call(scope, params)
          response.headers["X-Overdue-Count"] =
            scope.where(status: ExamOrder::OPEN_STATUSES).where("stage_due_on < ?", Time.zone.today).count.to_s
          render json: { orders: page[:orders].map { |o| Exams::Json.provider_order(o) }, next_before: page[:next_before],
                         overdue_count: response.headers["X-Overdue-Count"].to_i }
        rescue Exams::OrderQuery::InvalidFilter
          render json: { error: "invalid_filter" }, status: :unprocessable_entity
        rescue Exams::Cursor::Invalid
          render json: { error: "invalid_cursor" }, status: :unprocessable_entity
        end

        def schedule
          respond(Exams::Steps.schedule(order: @order, actor_user: Current.user,
                                        params: { provider_id: @provider.id, scheduled_at: params[:scheduled_at],
                                                  scheduled_place: params[:scheduled_place] }))
        end

        def collect = respond(Exams::Steps.collect(order: @order, actor_user: Current.user))

        private

        def load_provider
          @provider = Exams::Access.provider_of(Current.user)
          forbid("missing_role") unless @provider
        end

        def provider_order!
          @order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id], exam_provider_id: @provider.id)
          not_found unless @order
        end

        def respond(result)
          return render_failure(result) if result.failure?

          render json: Exams::Json.provider_order(@order.reload)
        end
      end
    end
  end
  ```

  Rotas dentro de `scope "/exams"`:

  ```ruby
    scope "provider", module: :provider, as: :provider do
      get  "orders",              to: "orders#index"
      post "orders/:id/schedule", to: "orders#schedule"
      post "orders/:id/collect",  to: "orders#collect"
    end
  ```

  O prestador não lê `GET /exams/orders/:id` (Access devolve nil → 404) nem a lista por unidade (não é `unit_staff?` → 403): confira que `Exams::Access.unit_staff?` não trata `exam_provider` como recepção.

- [ ] **Step 4: Run.** Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/controllers/exams/provider/orders_controller.rb config/routes.rb spec/requests/exams/provider/orders_spec.rb
  git commit -m "feat: add provider portal for scheduling and collecting exams

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 5 — Resultado, ciência e entrada FHIR (F-21.8, F-21.7)

### Task 13: Resultado (unidade e prestador), ciência, escalonamento e fila da equipe

**Files:**
- Create: `app/services/exams/result_values.rb`, `app/commands/exams/register_result.rb`, `app/commands/exams/acknowledge.rb`
- Create: `app/controllers/exams/results_controller.rb`, `app/controllers/exams/acknowledgements_controller.rb`
- Modify: `app/controllers/exams/provider/orders_controller.rb` (`result`), `config/routes.rb`
- Test: `spec/commands/exams/register_result_spec.rb`, `spec/requests/exams/acknowledgements_spec.rb`

**Interfaces:**
- Consumes: `Exams::{Transition,Escalate,Files.claim!,Access,Json,Trail}`.
- Produces:
  - `Exams::ResultValues.from_input(item, raw) -> Result(values:)` (`invalid_values`); `.from_fhir(item, observations) -> Array<Hash>`.
  - `Exams::RegisterResult.call(order:, source:, released_on:, laboratory_name:, critical:, values:, file: nil, file_id: nil, by_user: nil, by_provider: nil, fhir_identifier: nil, now: Time.current) -> Result(result:)` (`result_already_registered`, `invalid_transition`, `missing_pdf`, `invalid_released_on`, `invalid_laboratory`, `invalid_critical`). Crítico ou solicitante digitado → `Exams::Escalate` (`critical` / `unlinked_requester`) na mesma transação.
  - `Exams::Acknowledge.call(order:, by:, reason: nil) -> Result(order:)` (`not_requester`, `not_in_team`, `reason_required`, `invalid_transition`).
  - Rotas `POST /exams/orders/:id/result`, `POST /exams/provider/orders/:id/result`, `POST /exams/orders/:id/acknowledge`, `GET /exams/acknowledgements/pending`, `GET /exams/acknowledgements/team`.

- [ ] **Step 1: Write the failing tests.** `spec/commands/exams/register_result_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5–§6 (F-21.8): o resultado nasce uma vez; campos do catálogo;
  # crítico e solicitante digitado vão à fila da equipe já no resultado.
  # Review Focus 2: dois resultados ao mesmo tempo.
  RSpec.describe Exams::RegisterResult do
    before { exam_city! }

    let(:unit) { create_unit("UBS Resultado") }
    let(:doctor) { doctor!(unit) }
    let(:reception) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }
    let(:order) do
      exam_order!(patient: exam_patient!(90).first, unit: unit, requester: doctor).tap do |o|
        Exams::Steps.schedule(order: o, actor_user: doctor, params: { provider_id: exam_provider!.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
      end
    end

    def register(order, **over)
      described_class.call(order: order.reload, source: "unit", file: pdf_file!(user: reception), released_on: Time.zone.today.iso8601,
                           laboratory_name: "Laboratório Central", critical: false,
                           values: [ { "key" => "hba1c", "value" => "7.9" } ], by_user: reception, **over)
    end

    it "grava o resultado com os campos do catálogo e evento só com ids" do
      result = register(order).payload[:result]
      expect(result.values_list).to eq([ { "key" => "hba1c", "label" => "Hemoglobina glicada", "value" => "7.9", "unit" => "%", "loinc" => "4548-4" } ])
      expect(order.reload.status).to eq("resulted")
      expect(result.pdf_file.consumed?).to be(true)
      expect(DomainEvent.where(name: "exam_order.resulted").last.payload).to eq("exam_order_id" => order.id, "source" => "unit", "critical" => false)
      expect(DomainEvent.last(5).map { |e| e.payload.to_json }.join).not_to include("7.9")
    end

    it "dois resultados ao mesmo tempo: o segundo não entra; cancelado não ganha resultado (Review Focus 2)" do
      register(order)
      expect(register(order).reason).to eq(:result_already_registered)
      expect(ExamResult.where(exam_order_id: order.id).count).to eq(1)
      expect { ExamResult.create!(ExamResult.find_by(exam_order_id: order.id).attributes.except("id").merge("pdf_file_id" => pdf_file!.id)) }
        .to raise_error(ActiveRecord::RecordNotUnique)
      other = exam_order!(patient: exam_patient!(91).first, unit: unit, requester: doctor)
      Exams::Steps.cancel(order: other, reason: "Feito em outro serviço", actor_user: doctor)
      expect(register(other).then { |r| [ r.reason, r.details ] }).to eq([ :invalid_transition, { status: "cancelled" } ])
      expect(ExamResult.where(exam_order_id: other.id)).to be_empty
    end

    it "crítico e solicitante digitado escalam no resultado; valor fora do catálogo recusa" do
      register(order, critical: true)
      expect(order.reload.events.where(kind: "escalated").pluck(:escalation_reason)).to eq(%w[critical])
      typed = Exams::Open.call(patient: order.patient, sigtap_code: "0202010503", priority: "routine", unit: unit, origin: "reception_guide",
                               requester_name: "Dr. Externo", requester_cns: Professionals::Cns.generate("x"),
                               guide_file: pdf_file!(user: reception).tap { |f| f.update!(consumed_at: Time.current) },
                               created_by: reception).payload[:order]
      Exams::Steps.schedule(order: typed, actor_user: reception, params: { provider_id: exam_provider!(name: "L2").id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      register(typed)
      expect(typed.reload.events.where(kind: "escalated").pluck(:escalation_reason)).to eq(%w[unlinked_requester])
      third = exam_order!(patient: exam_patient!(92).first, unit: unit, requester: doctor)
      Exams::Steps.schedule(order: third, actor_user: doctor, params: { provider_id: exam_provider!(name: "L3").id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      expect(register(third, values: [ { "key" => "glicose", "value" => "1" } ]).reason).to eq(:invalid_values)
      expect(register(third, file: nil).reason).to eq(:missing_pdf)
      expect(register(third, released_on: (Time.zone.today + 1).iso8601).reason).to eq(:invalid_released_on)
    end
  end
  ```

  `spec/requests/exams/acknowledgements_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5 etapa 6 (F-21.8) — Review Focus 4: ciência por quem não pode.
  RSpec.describe "Exames — ciência e fila da equipe", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Ciência") }
    let(:doctor) { doctor!(unit) }
    let(:nurse) { nurse!(unit).tap { |n| exam_team!(unit, n) } }
    let(:reception) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }

    def resulted!(critical: false)
      order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor)
      Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: exam_provider!(name: "L#{SecureRandom.hex(2)}").id,
                                                                        scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      Exams::RegisterResult.call(order: order.reload, source: "unit", file: pdf_file!(user: reception), released_on: Time.zone.today.iso8601,
                                 laboratory_name: "Lab", critical: critical, values: [], by_user: reception)
      order.reload
    end

    it "pendentes do solicitante; ciência pelo solicitante sem motivo; repetir = 409" do
      order = resulted!
      sign_in_as(doctor)
      get "/exams/acknowledgements/pending"
      expect(json_body["orders"].map { |o| o["id"] }).to eq([ order.id ])
      expect(json_body["denied"]).to eq([])
      expect(json_body["orders"].first["result"]).to include("critical" => false)
      json_post "/exams/orders/#{order.id}/acknowledge"
      expect(json_body["status"]).to eq("acknowledged")
      expect(DomainEvent.where(name: "exam_order.acknowledged").last.payload).to eq("exam_order_id" => order.id, "by_team" => false)
      json_post "/exams/orders/#{order.id}/acknowledge"
      expect([ response.status, json_body["status"] ]).to eq([ 409, "acknowledged" ])
    end

    it "equipe só com o exame escalado e com motivo; outra unidade, recepção e não escalado recusam" do
      calm = resulted!
      critical = resulted!(critical: true)
      sign_in_as(nurse)
      json_post "/exams/orders/#{calm.id}/acknowledge", reason: "Vi o resultado"
      expect([ response.status, json_body["error"] ]).to eq([ 403, "not_requester" ])
      get "/exams/acknowledgements/team"
      expect(json_body["orders"].map { |o| o["id"] }).to eq([ critical.id ])
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "team", "exam_order_id" => critical.id)
      json_post "/exams/orders/#{critical.id}/acknowledge"
      expect(json_body["error"]).to eq("reason_required")
      far = doctor!(create_unit("UBS Distante")).tap { |d| exam_team!(d.professional.links.active.first.health_unit, d) }
      sign_in_as(far)
      json_post "/exams/orders/#{critical.id}/acknowledge", reason: "Avaliado"
      expect([ response.status, json_body["error"] ]).to eq([ 403, "not_in_team" ])
      sign_in_as(reception)
      json_post "/exams/orders/#{critical.id}/acknowledge", reason: "Avaliado"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(nurse)
      json_post "/exams/orders/#{critical.id}/acknowledge", reason: "Paciente contatada; retorno amanhã"
      expect(json_body["status"]).to eq("acknowledged")
      expect(DomainEvent.where(name: "exam_order.acknowledged").last.payload["by_team"]).to be(true)
    end

    it "a unidade registra o resultado pela rota; o prestador pela dele" do
      order = exam_order!(patient: exam_patient!(95).first, unit: unit, requester: doctor)
      provider = exam_provider!(name: "Lab Portal")
      Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: provider.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      lab = staff_with("lab-#{SecureRandom.hex(2)}@lab.com.br", "exam_provider").tap { |u| ExamProviderUser.create!(exam_provider: provider, user: u) }
      sign_in_as(lab)
      json_post "/exams/uploads", content_type: "application/pdf", data: Base64.strict_encode64(ExamHelpers::PDF_BYTES)
      upload = json_body["upload_id"]
      json_post "/exams/provider/orders/#{order.id}/result", upload_id: upload, released_on: Time.zone.today.iso8601,
                                                             laboratory_name: "Lab Portal", critical: false, values: [ { key: "hba1c", value: "6.1" } ]
      expect(json_body["status"]).to eq("resulted")
      expect(ExamResult.find_by(exam_order_id: order.id)).to have_attributes(source: "provider", registered_by_user_id: lab.id)
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL.

- [ ] **Step 3: Implemente.** `app/services/exams/result_values.rb`:

  ```ruby
  # Valores do resultado (spec §4): pela chave do campo do catálogo (unidade,
  # prestador) ou pelo LOINC (FHIR; LOINC desconhecido fica sem chave).
  module Exams
    module ResultValues
      MAX = 60

      module_function

      def from_input(item, raw)
        raw = Array(raw).map { |v| v.respond_to?(:to_unsafe_h) ? v.to_unsafe_h : v }
        return Result.fail(:invalid_values) unless raw.size <= MAX && raw.all?(Hash)

        values = raw.map do |entry|
          entry = entry.stringify_keys
          field = item&.result_field(entry["key"].to_s)
          value = entry["value"]
          return Result.fail(:invalid_values) unless field && (value.is_a?(String) || value.is_a?(Numeric)) && value.to_s.strip.length.between?(1, 200)

          { "key" => field["key"], "label" => field["label"], "value" => value.to_s.strip, "unit" => field["unit"], "loinc" => field["loinc"] }
        end
        return Result.fail(:invalid_values) unless values.map { |v| v["key"] }.uniq.size == values.size

        Result.ok(values: values)
      end

      def from_fhir(item, observations)
        observations.first(MAX).map do |obs|
          coding = Array(obs.dig("code", "coding")).find { |c| c["system"] == "http://loinc.org" } || Array(obs.dig("code", "coding")).first || {}
          field = item&.result_field_by_loinc(coding["code"])
          value = obs["valueQuantity"] ? obs.dig("valueQuantity", "value").to_s : obs["valueString"].to_s
          { "key" => field&.fetch("key"), "label" => field&.fetch("label") || coding["display"].presence || coding["code"].to_s,
            "value" => value.strip[0, 200], "unit" => obs.dig("valueQuantity", "unit") || field&.fetch("unit"),
            "loinc" => coding["system"] == "http://loinc.org" ? coding["code"] : nil }
        end
      end
    end
  end
  ```

  `app/commands/exams/register_result.rb`:

  ```ruby
  # Registra o resultado (ADR 0036; spec §5 etapa 5, §6): uma vez por pedido
  # (checado sob a trava do pedido; índice único como última palavra), só de
  # agendado/coletado, com o PDF; crítico ou solicitante sem vínculo escalam
  # na mesma transação. Mesma porta para unidade, portal e FHIR.
  module Exams
    module RegisterResult
      module_function

      def call(order:, source:, released_on:, laboratory_name:, critical:, values:, file: nil, file_id: nil, by_user: nil,
               by_provider: nil, fhir_identifier: nil, now: Time.current)
        released = parse_date(released_on)
        return Result.fail(:invalid_released_on) unless released && released <= now.in_time_zone.to_date
        return Result.fail(:invalid_laboratory) unless laboratory_name.is_a?(String) && laboratory_name.strip.length.between?(1, 160)
        return Result.fail(:invalid_critical) unless [ true, false ].include?(critical)

        result = nil
        ApplicationRecord.transaction do
          order.lock!
          next result = Result.fail(:result_already_registered) if ExamResult.exists?(exam_order_id: order.id)

          checked = values.is_a?(Result) ? values : (source == "fhir" ? Result.ok(values: values) : ResultValues.from_input(order.catalog_item, values))
          next result = checked if checked.failure?

          file ||= Files.claim!(file_id: file_id, user: by_user, provider: by_provider)
          next result = Result.fail(:missing_pdf) unless file&.content_type == "application/pdf"

          file.update!(consumed_at: now) unless file.consumed?
          moved = Transition.call(order: order, kind: "resulted", actor_user: by_user, actor_provider: by_provider,
                                  event_payload: { source: source, critical: critical }, now: now)
          if moved.failure?
            result = moved
            raise ActiveRecord::Rollback
          end
          stored = ExamResult.create!(exam_order: order, patient: order.patient, sigtap_code: order.sigtap_code, origin: "order",
                                      source: source, pdf_file: file, values: checked.payload[:values].to_json, released_on: released,
                                      laboratory_name: laboratory_name.strip, critical: critical, fhir_identifier: fhir_identifier,
                                      exam_provider_id: by_provider&.id || order.exam_provider_id, registered_by_user: by_user,
                                      registered_by_provider: by_provider)
          reason = if critical then "critical"
                   elsif order.requester_user_id.nil? then "unlinked_requester"
                   end
          Escalate.call(order: order, reason: reason, now: now) if reason
          result = Result.ok(result: stored)
        end
        result
      end

      def parse_date(raw)
        raw.is_a?(Date) ? raw : (raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(raw) : nil)
      rescue Date::Error
        nil
      end
      private_class_method :parse_date
    end
  end
  ```

  Nota: a falha de `Files.claim!`/transição desfaz o `consumed_at` (mesma transação); o arquivo volta a ser órfão e o job o apaga em 24 h.

  `app/commands/exams/acknowledge.rb`:

  ```ruby
  # Ciência (spec §5 etapa 6): o solicitante (motivo opcional); com o exame
  # escalado, qualquer médico ou enfermeiro da equipe da unidade, com motivo.
  module Exams
    module Acknowledge
      module_function

      def call(order:, by:, reason: nil)
        ApplicationRecord.transaction do
          order.lock!
          by_team = false
          if order.requester_user_id == by.id && by.has_role?("health_professional")
            by_team = false
          elsif order.escalated? && Access.team_member?(by, order.unit_id)
            by_team = true
            next Result.fail(:reason_required) if reason.to_s.strip.length < 3 && order.status == "resulted"
          else
            next Result.fail(order.escalated? ? :not_in_team : :not_requester)
          end
          Transition.call(order: order, kind: "acknowledged", actor_user: by, reason: reason, event_payload: { by_team: by_team })
        end
      end
    end
  end
  ```

  `app/controllers/exams/results_controller.rb`:

  ```ruby
  # POST /exams/orders/:id/result pela unidade (contrato §3).
  module Exams
    class ResultsController < BaseController
      def create
        order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id])
        return not_found unless order
        return forbid("not_unit_staff") unless Exams::Access.unit_staff?(Current.user, order.unit_id)

        result = Exams::RegisterResult.call(order: order, source: "unit", file_id: params[:upload_id], released_on: params[:released_on],
                                            laboratory_name: params[:laboratory_name], critical: params[:critical],
                                            values: params[:values], by_user: Current.user)
        return render_failure(result) if result.failure?

        render json: Exams::Json.order(order.reload, view: :administrative, user: Current.user), status: :created
      end
    end
  end
  ```

  No `Exams::Provider::OrdersController`, a ação `result` (mesmo corpo, `source: "provider"`, `by_user: Current.user`; responde `Exams::Json.provider_order`), e a rota `post "orders/:id/result", to: "orders#result"` no `scope "provider"`.

  `app/controllers/exams/acknowledgements_controller.rb`:

  ```ruby
  # Ciência e filas (contrato §3): pendentes do solicitante; fila da equipe
  # (escalados das unidades onde o usuário é médico/enfermeiro de equipe).
  module Exams
    class AcknowledgementsController < BaseController
      DENIED_WINDOW = 30.days

      def create
        order = uuid?(params[:id]) && ExamOrder.find_by(id: params[:id])
        return not_found unless order

        result = Exams::Acknowledge.call(order: order, by: Current.user, reason: params[:reason])
        return render_failure(result) if result.failure?

        render json: Exams::Json.order(order.reload, view: :clinical, user: Current.user)
      end

      def pending
        return forbid("missing_role") unless Current.user.has_role?("health_professional")

        list(ExamOrder.where(requester_user_id: Current.user.id, status: "resulted"), :author) do
          ExamOrder.where(requester_user_id: Current.user.id, status: "denied")
                   .where("status_changed_at >= ?", DENIED_WINDOW.ago).order(status_changed_at: :desc).limit(50)
        end
      end

      def team
        units = Exams::Access.team_unit_ids(Current.user)
        return forbid("not_in_team") if units.empty?

        list(ExamOrder.where(unit_id: units, status: "resulted").where.not(escalated_at: nil), :team)
      end

      private

      # Dashboard D7: a ciência pendente devolve também os negados dos últimos 30 dias (o bloco).
      def list(scope, kind)
        page = Exams::OrderQuery.call(scope, params)
        grant = Exams::Access::Grant.new(view: :clinical, kind: kind, opening: nil)
        page[:orders].each { |order| Exams::Trail.viewed!(order: order, user: Current.user, grant: grant) }
        body = { orders: page[:orders].map { |o| Exams::Json.order(o, view: :clinical, user: Current.user, events: false) },
                 next_before: page[:next_before] }
        body[:denied] = yield.map { |o| Exams::Json.order(o, view: :clinical, user: Current.user, events: false) } if block_given?
        render json: body
      rescue Exams::Cursor::Invalid, Exams::OrderQuery::InvalidFilter
        render json: { error: "invalid_cursor" }, status: :unprocessable_entity
      end
    end
  end
  ```

  Rotas: `post "orders/:id/result", to: "results#create"`, `post "orders/:id/acknowledge", to: "acknowledgements#create"`, `get "acknowledgements/pending", to: "acknowledgements#pending"`, `get "acknowledgements/team", to: "acknowledgements#team"`.

- [ ] **Step 4: Run** os dois arquivos do Step 1 e `rspec spec/requests/exams spec/services/exams spec/commands/exams`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/result_values.rb app/commands/exams/register_result.rb app/commands/exams/acknowledge.rb app/controllers/exams/results_controller.rb app/controllers/exams/acknowledgements_controller.rb app/controllers/exams/provider/orders_controller.rb config/routes.rb spec/commands/exams/register_result_spec.rb spec/requests/exams/acknowledgements_spec.rb
  git commit -m "feat: register exam results, acknowledgements and the team queue

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Cards F-21.6 e F-21.8 → Done (lado api).

### Task 14: Entrada FHIR — `POST /fhir/DiagnosticReport` e `GET /fhir/ServiceRequest`

**Files:**
- Create: `config/exams/fhir-diagnostic-report-inbound.json` (cópia byte a byte do contracts, Task 1)
- Create: `app/services/exams/fhir/{identifiers,inbound,service_requests}.rb`
- Create: `app/controllers/fhir/{base,diagnostic_reports,service_requests}_controller.rb`
- Modify: `config/routes.rb`
- Create: `spec/fixtures/exams/diagnostic-report-hemograma.json` (cópia do exemplo do contracts)
- Test: `spec/requests/fhir/diagnostic_reports_spec.rb`, `spec/requests/fhir/service_requests_spec.rb`, `spec/services/exams/fhir/schema_copy_spec.rb`

**Interfaces:**
- Consumes: `ExamProvider.by_token`, `Exams::Gate.{fhir_usable?,fake_lab_usable?}`, `Exams::{Files.store,ResultValues.from_fhir,RegisterResult}`.
- Produces: `Exams::Fhir::Identifiers::CPF_SYSTEM`, `CNS_SYSTEM`, `.matches?(patient, identifier) -> Boolean`, `.for_patient(patient) -> Hash`; `Exams::Fhir::Inbound.call(provider:, resource:, now: Time.current) -> Result(result:, created:)` (`invalid_resource` com `details: { paths: }`, `missing_pdf`, `order_not_found`, `patient_mismatch`, `invalid_transition`); `Exams::Fhir::ServiceRequests.bundle(provider:) -> Hash`. O `fake-lab` (Task 15) consome as duas rotas.

- [ ] **Step 1: Write the failing tests.** `spec/services/exams/fhir/schema_copy_spec.rb`:

  ```ruby
  require "rails_helper"

  # O api valida contra a CÓPIA do esquema publicado no contracts (tag
  # exams-v1.0.0). Mudou o esquema lá: copie e troque os hashes aqui.
  RSpec.describe "Cópias dos esquemas do contracts (exams-v1.0.0)" do
    it "as cópias são as publicadas" do
      expect(Digest::SHA256.file(Rails.root.join("config/exams/fhir-diagnostic-report-inbound.json")).hexdigest)
        .to eq("<sha256 de contracts/exams/fhir-diagnostic-report-inbound.json no commit da Task 1>")
      expect(Digest::SHA256.file(Rails.root.join("spec/fixtures/exams/exam-order-v1.json")).hexdigest)
        .to eq("<sha256 de contracts/exams/exam-order-v1.json no commit da Task 1>")
    end
  end
  ```
  Os dois valores **não** são placeholders do plano: rode `shasum -a 256 contracts/exams/fhir-diagnostic-report-inbound.json contracts/exams/exam-order-v1.json` depois do commit da Task 1 e cole os hashes literais antes do Step 2.

  `spec/requests/fhir/diagnostic_reports_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §7 (com D7) — F-21.7. Review Focus 1: laudo no paciente errado.
  RSpec.describe "FHIR — DiagnosticReport de entrada", type: :request do
    before { exam_city!(fhir: true) }

    let(:unit) { create_unit("UBS FHIR") }
    let(:doctor) { doctor!(unit) }
    let(:item) do
      exam_catalog_item!(sigtap_code: "0202020380", result_fields: [
        { "key" => "hb", "label" => "Hemoglobina", "unit" => "g/dL", "loinc" => "718-7" } ])
    end
    let(:provider) { exam_provider!(token: ExamHelpers::FAKE_TOKEN).tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }
    let(:patient_and_citizen) { exam_patient!(exam_seq!) }
    let(:patient) { patient_and_citizen.first }
    let(:order) do
      exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: item.sigtap_code).tap do |o|
        Exams::Steps.schedule(order: o, actor_user: doctor, params: { provider_id: provider.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
      end
    end

    def report(**over)
      base = JSON.parse(File.read(Rails.root.join("spec/fixtures/exams/diagnostic-report-hemograma.json")))
      base["basedOn"][0]["identifier"]["value"] = order.short_code
      base["subject"]["identifier"]["value"] = patient.cpf
      base["issued"] = Time.current.iso8601
      base.merge(over.transform_keys(&:to_s))
    end

    def send_report(body, token: ExamHelpers::FAKE_TOKEN, type: "application/fhir+json")
      post "/fhir/DiagnosticReport", params: body.is_a?(String) ? body : body.to_json,
                                     headers: { "CONTENT_TYPE" => type, "Authorization" => "Bearer #{token}" }
    end

    it "aceita: resultado com LOINC no campo, LOINC desconhecido sem chave, crítico por LL, idempotente" do
      provider
      log = capture_log { send_report(report) }
      expect(response).to have_http_status(:created)
      result = ExamResult.find(json_body["result_id"])
      expect(result).to have_attributes(source: "fhir", critical: true, fhir_identifier: "LS-2026-000123", registered_by_provider_id: provider.id)
      expect(result.values_list.map { |v| v.values_at("key", "loinc", "value") }).to eq([ [ "hb", "718-7", "13.4" ], [ nil, "777-3", "41" ] ])
      expect(order.reload.status).to eq("resulted")
      expect(order.events.where(kind: "escalated").pluck(:escalation_reason)).to eq(%w[critical])
      expect(log).not_to include(patient.cpf, "13.4", ExamHelpers::FAKE_TOKEN, "JVBERi0")
      send_report(report(presentedForm: [ { contentType: "application/pdf", data: Base64.strict_encode64(ExamHelpers::PDF_BYTES + "outro") } ]))
      expect([ response.status, json_body["result_id"] ]).to eq([ 200, result.id ])
      expect(result.reload.pdf_file.bytes).to eq(Base64.strict_decode64(report["presentedForm"][0]["data"]))
    end

    it "paciente e prestador errados (Review Focus 1)" do
      provider
      other_patient, = exam_patient!(exam_seq!)
      send_report(report(subject: { identifier: { system: Exams::Fhir::Identifiers::CPF_SYSTEM, value: other_patient.cpf } }))
      expect([ response.status, json_body["error"] ]).to eq([ 422, "patient_mismatch" ])
      cns = Professionals::Cns.generate("cidada-fhir")
      patient_and_citizen.last.update!(cns: cns)
      send_report(report("identifier" => [ { "value" => "LS-CNS-1" } ],
                         "subject" => { "identifier" => { "system" => Exams::Fhir::Identifiers::CNS_SYSTEM, "value" => cns } }))
      expect(response).to have_http_status(:created)

      second = exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: item.sigtap_code)
      intruder = exam_provider!(name: "Intruso", token: "token-do-intruso-0123456789abcdef0123456789")
      Exams::Steps.schedule(order: second, actor_user: doctor, params: { provider_id: provider.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
      body = report("identifier" => [ { "value" => "LS-X" } ])
      body["basedOn"][0]["identifier"]["value"] = second.short_code
      send_report(body, token: "token-do-intruso-0123456789abcdef0123456789")
      expect([ response.status, json_body["error"] ]).to eq([ 404, "order_not_found" ])
      body["basedOn"][0]["identifier"]["value"] = "ZZZZZZZZZZ"
      send_report(body)
      expect([ response.status, json_body["error"] ]).to eq([ 404, "order_not_found" ])
      expect(second.reload.status).to eq("scheduled")
    end

    it "recusas: token, interruptor, tipo, recurso, PDF, dom-3, status preliminar, transição, taxa" do
      provider
      send_report(report, token: "nao-existe-0123456789abcdef0123456789abcd")
      expect([ response.status, json_body["error"] ]).to eq([ 401, "invalid_token" ])
      send_report(report, type: "application/json")
      expect([ response.status, json_body["error"] ]).to eq([ 415, "unsupported_media_type" ])
      send_report("{nope")
      expect([ response.status, json_body["error"] ]).to eq([ 422, "invalid_resource" ])
      send_report(report.except("presentedForm"))
      expect([ response.status, json_body["error"] ]).to eq([ 422, "missing_pdf" ])
      send_report(report(presentedForm: [ { contentType: "application/pdf", data: Base64.strict_encode64("GIF89a") } ]))
      expect(json_body["error"]).to eq("missing_pdf")
      send_report(report(status: "preliminary"))
      expect([ json_body["error"], json_body["details"] ]).to eq([ "invalid_resource", [ "/status" ] ])
      send_report(report(result: [ { reference: "#hb" } ]))
      expect([ json_body["error"], json_body["details"] ]).to eq([ "invalid_resource", [ "/result" ] ])
      expect(response.body).not_to include(patient.cpf, "13.4")
      Exams::Steps.cancel(order: order, reason: "Paciente desistiu", actor_user: doctor)
      send_report(report)
      expect([ response.status, json_body["error"] ]).to eq([ 409, "invalid_transition" ])
      provider.update!(fhir_token_revoked_at: Time.current)
      send_report(report)
      expect(json_body["error"]).to eq("invalid_token")
      Platform::Features.set!(city: Current.city, key: "exam_fhir_inbound", enabled: false, maintainer: ledi_maintainer!)
      send_report(report)
      expect([ response.status, json_body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "exam_fhir_inbound" } ])
    end

    it "limite de 60 por minuto por token" do
      provider
      allow(Fhir::BaseController::RateLimitStore).to receive(:increment).and_return(61)
      send_report(report)
      expect([ response.status, json_body["error"] ]).to eq([ 429, "rate_limited" ])
    end

    it "prestador simulado exige fake_lab" do
      provider.update!(simulated: true)
      send_report(report)
      expect([ response.status, json_body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "fake_lab" } ])
      Platform::Features.set!(city: Current.city, key: "fake_lab", enabled: true, maintainer: ledi_maintainer!)
      send_report(report)
      expect(response).to have_http_status(:created)
    end
  end
  ```

  `spec/requests/fhir/service_requests_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §7: o que o laboratório (e o fake-lab) usa para saber o que fazer.
  RSpec.describe "FHIR — ServiceRequest", type: :request do
    before { exam_city!(fhir: true) }

    it "Bundle searchset com as guias agendadas ou coletadas do prestador do token, e só dele" do
      unit = create_unit("UBS SR")
      doctor = doctor!(unit)
      exam_catalog_item! # antes dos prestadores: items: :all os habilita
      provider = exam_provider!(token: ExamHelpers::FAKE_TOKEN)
      other = exam_provider!(name: "Outro")
      patient, = exam_patient!(exam_seq!)
      mine = exam_order!(patient: patient, unit: unit, requester: doctor, priority: "urgent")
      theirs = exam_order!(patient: patient, unit: unit, requester: doctor)
      waiting = exam_order!(patient: patient, unit: unit, requester: doctor)
      [ [ mine, provider ], [ theirs, other ] ].each do |order, prov|
        Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: prov.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Lab" })
      end
      get "/fhir/ServiceRequest", params: { status: "active" }, headers: { "Authorization" => "Bearer #{ExamHelpers::FAKE_TOKEN}" }
      expect(response.media_type).to eq("application/fhir+json")
      bundle = JSON.parse(response.body)
      expect(bundle.slice("resourceType", "type", "total")).to eq("resourceType" => "Bundle", "type" => "searchset", "total" => 1)
      resource = bundle["entry"].first["resource"]
      expect(resource).to include("resourceType" => "ServiceRequest", "status" => "active", "intent" => "order", "priority" => "urgent")
      expect(resource["identifier"].first["value"]).to eq(mine.short_code)
      expect(resource.dig("code", "coding", 0, "code")).to eq(mine.sigtap_code)
      expect(resource.dig("subject", "identifier")).to eq("system" => Exams::Fhir::Identifiers::CPF_SYSTEM, "value" => patient.cpf)
      expect(response.body).not_to include(theirs.short_code, waiting.short_code)
      get "/fhir/ServiceRequest", params: { status: "completed" }, headers: { "Authorization" => "Bearer #{ExamHelpers::FAKE_TOKEN}" }
      expect([ response.status, JSON.parse(response.body)["error"] ]).to eq([ 422, "invalid_search" ])
    end
  end
  ```

- [ ] **Step 2: Run** os três. Expected: FAIL.

- [ ] **Step 3: Implemente.** `app/services/exams/fhir/identifiers.rb`:

  ```ruby
  # Identificação do paciente no FHIR (Divergência D7): sistemas da RNDS.
  # CPF compara com o do paciente; CNS com o de um par validado da pessoa
  # (citizens.cns é cifrado não determinístico: lido por par, nunca buscado).
  module Exams
    module Fhir
      module Identifiers
        CPF_SYSTEM = "http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf".freeze
        CNS_SYSTEM = "http://rnds.saude.gov.br/fhir/r4/NamingSystem/cns".freeze

        module_function

        def matches?(patient, identifier)
          value = identifier["value"].to_s
          case identifier["system"]
          when CPF_SYSTEM then ActiveSupport::SecurityUtils.secure_compare(value, patient.cpf.to_s)
          when CNS_SYSTEM
            pairs = Citizen.not_erased.verification_level_verified
            pairs.where(patient_id: patient.id).or(pairs.where(cpf: patient.cpf)).any? { |c| c.cns.present? && c.cns == value }
          else false
          end
        end

        def for_patient(patient) = { system: CPF_SYSTEM, value: patient.cpf }
      end
    end
  end
  ```

  `app/services/exams/fhir/inbound.rb`:

  ```ruby
  # Tradução do DiagnosticReport (ADR 0036, "FHIR só na borda"; contrato §7
  # com D7). Valida contra a cópia do esquema (só caminhos no erro, nunca
  # valores), confere dom-3, idempotência por (prestador, identifier), guia do
  # prestador, paciente por CPF/CNS, PDF; chama a mesma porta do portal.
  module Exams
    module Fhir
      module Inbound
        SCHEMA_PATH = Rails.root.join("config/exams/fhir-diagnostic-report-inbound.json")
        CRITICAL = %w[HH LL AA].freeze

        module_function

        def call(provider:, resource:, now: Time.current)
          return Result.fail(:invalid_resource, details: { paths: [ "" ] }) unless resource.is_a?(Hash)

          paths = schema.validate(resource).map { |error| error["data_pointer"].to_s }.uniq
          return Result.fail(:missing_pdf) if paths.any? { |p| p.start_with?("/presentedForm") } || (paths.include?("") && !resource.key?("presentedForm"))
          return Result.fail(:invalid_resource, details: { paths: paths.map { |p| top(p) }.uniq.first(20) }) if paths.any?

          observations = resource["contained"]
          refs = resource["result"].map { |r| r["reference"].delete_prefix("#") }
          return Result.fail(:invalid_resource, details: { paths: [ "/result" ] }) unless refs.sort == observations.map { |o| o["id"] }.sort

          identifier = resource.dig("identifier", 0, "value")
          existing = ExamResult.find_by(exam_provider_id: provider.id, fhir_identifier: identifier)
          return Result.ok(result: existing, created: false) if existing

          order = ExamOrder.find_by(short_code: resource.dig("basedOn", 0, "identifier", "value"), exam_provider_id: provider.id)
          return Result.fail(:order_not_found) unless order
          return Result.fail(:patient_mismatch) unless Identifiers.matches?(order.patient, resource.dig("subject", "identifier"))

          stored = Files.store(bytes: decode(resource.dig("presentedForm", 0, "data")), content_type: "application/pdf", provider: provider)
          return Result.fail(:missing_pdf) if stored.failure?

          registered = RegisterResult.call(
            order: order, source: "fhir", file: stored.payload[:file], released_on: Time.iso8601(resource["issued"]).in_time_zone.to_date,
            laboratory_name: resource.dig("performer", 0, "display"), critical: critical?(observations),
            values: ResultValues.from_fhir(order.catalog_item, observations), by_provider: provider, fhir_identifier: identifier, now: now
          )
          registered.ok? ? Result.ok(result: registered.payload[:result], created: true) : registered
        rescue ActiveRecord::RecordNotUnique
          again = ExamResult.find_by(exam_provider_id: provider.id, fhir_identifier: identifier)
          again ? Result.ok(result: again, created: false) : raise
        end

        def critical?(observations)
          observations.any? { |o| Array(o["interpretation"]).flat_map { |i| Array(i["coding"]) }.any? { |c| CRITICAL.include?(c["code"]) } }
        end

        def decode(data)
          Base64.strict_decode64(data.to_s)
        rescue ArgumentError
          ""
        end

        def top(pointer) = "/" + pointer.delete_prefix("/").split("/").first.to_s

        def schema = (@schema ||= JSONSchemer.schema(SCHEMA_PATH))
        private_class_method :critical?, :decode, :top, :schema
      end
    end
  end
  ```

  `app/services/exams/fhir/service_requests.rb`:

  ```ruby
  # GET /fhir/ServiceRequest?status=active (contrato §7): as guias agendadas
  # ou coletadas do prestador do token, como Bundle searchset.
  module Exams
    module Fhir
      module ServiceRequests
        SIGTAP_SYSTEM = "http://www.saude.gov.br/fhir/r4/CodeSystem/BRTabelaSUS".freeze
        LIMIT = 200

        module_function

        def bundle(provider:)
          orders = ExamOrder.where(exam_provider_id: provider.id, status: %w[scheduled collected])
                            .includes(:patient).order(:scheduled_at, :id).limit(LIMIT).to_a
          { resourceType: "Bundle", type: "searchset", total: orders.size,
            entry: orders.map { |order| { resource: resource(order) } } }
        end

        def resource(order)
          { resourceType: "ServiceRequest", id: order.id,
            identifier: [ { system: "urn:rotasaude:exam-order", value: order.short_code } ],
            status: "active", intent: "order", priority: order.priority,
            code: { coding: [ { system: SIGTAP_SYSTEM, code: order.sigtap_code,
                                display: Labels.exam(order.sigtap_code, order.sigtap_competence) } ] },
            subject: { identifier: Identifiers.for_patient(order.patient) },
            authoredOn: order.created_at.iso8601, occurrenceDateTime: order.scheduled_at&.iso8601 }.compact
        end
      end
    end
  end
  ```

  `app/controllers/fhir/base_controller.rb`:

  ```ruby
  # Borda FHIR (ADR 0036; contrato §7): host da cidade, sem cookie, token do
  # prestador no Authorization (só o hash existe no banco). Ordem: limite por
  # token (60/min) → interruptor → token → prestador simulado exige fake_lab.
  # Nada do corpo nem do token vai para log ou erro.
  module Fhir
    class BaseController < ApplicationController
      MEDIA_TYPE = "application/fhir+json".freeze

      module RateLimitStore
        def self.increment(...) = Rails.cache.increment(...)
      end

      rate_limit to: 60, within: 1.minute, name: "fhir_inbound", store: RateLimitStore,
                 by: -> { token ? ExamProvider.digest(token) : request.remote_ip },
                 with: -> { fhir_error("rate_limited", :too_many_requests) }
      before_action :require_fhir_inbound!
      before_action :authenticate_provider!

      private

      def token
        header = request.authorization.to_s
        header.start_with?("Bearer ") ? header.delete_prefix("Bearer ").strip.presence : nil
      end

      def require_fhir_inbound!
        return if Exams::Gate.fhir_usable?(Current.city)

        render json: { error: "feature_disabled", feature: Exams::Gate::FHIR_KEY }, status: :forbidden
      end

      def authenticate_provider!
        @provider = ExamProvider.by_token(token)
        unless @provider
          response.headers["WWW-Authenticate"] = "Bearer"
          return fhir_error("invalid_token", :unauthorized)
        end
        return if !@provider.simulated || Exams::Gate.fake_lab_usable?(Current.city)

        render json: { error: "feature_disabled", feature: Exams::Gate::FAKE_LAB_KEY }, status: :forbidden
      end

      def fhir_error(code, status, details: nil)
        render json: { error: code, details: details }.compact, status: status
      end
    end
  end
  ```

  `app/controllers/fhir/diagnostic_reports_controller.rb`:

  ```ruby
  module Fhir
    class DiagnosticReportsController < BaseController
      MAX_BODY = 15 * 1024 * 1024
      STATUS = { order_not_found: :not_found, invalid_transition: :conflict }.freeze

      def create
        return fhir_error("unsupported_media_type", :unsupported_media_type) unless request.media_type == MEDIA_TYPE
        return fhir_error("payload_too_large", :payload_too_large) if request.content_length.to_i > MAX_BODY

        resource = begin
          JSON.parse(request.raw_post)
        rescue JSON::ParserError
          nil
        end
        result = Exams::Fhir::Inbound.call(provider: @provider, resource: resource)
        if result.failure?
          return fhir_error(result.reason.to_s, STATUS.fetch(result.reason, :unprocessable_entity), details: result.details[:paths])
        end

        render json: { result_id: result.payload[:result].id }, status: result.payload[:created] ? :created : :ok
      end
    end
  end
  ```

  `app/controllers/fhir/service_requests_controller.rb`:

  ```ruby
  module Fhir
    class ServiceRequestsController < BaseController
      def index
        return fhir_error("invalid_search", :unprocessable_entity) unless params[:status] == "active"

        render body: Exams::Fhir::ServiceRequests.bundle(provider: @provider).to_json, content_type: MEDIA_TYPE
      end
    end
  end
  ```

  Rotas (fora de `/exams`, perto de `/pec`):

  ```ruby
  # Entrada FHIR do laudo (ADR 0036; contrato §7). Host da cidade, sem cookie:
  # token Bearer do prestador.
  post "/fhir/DiagnosticReport", to: "fhir/diagnostic_reports#create"
  get  "/fhir/ServiceRequest",   to: "fhir/service_requests#index"
  ```

  Confira que `application/fhir+json` **não** está registrado como tipo JSON em `config/initializers/mime_types.rb` (se estivesse, o Rails parsearia o corpo para `params` e ele cairia no log). Se um `rate_limit` do Rails 8.1 com `name:` exigir cache compartilhado, siga como `PecTimelinesController`.

- [ ] **Step 4: Run** os três. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add config/exams/fhir-diagnostic-report-inbound.json app/services/exams/fhir/identifiers.rb app/services/exams/fhir/inbound.rb app/services/exams/fhir/service_requests.rb app/controllers/fhir/base_controller.rb app/controllers/fhir/diagnostic_reports_controller.rb app/controllers/fhir/service_requests_controller.rb config/routes.rb spec/fixtures/exams/diagnostic-report-hemograma.json spec/services/exams/fhir/schema_copy_spec.rb spec/requests/fhir/diagnostic_reports_spec.rb spec/requests/fhir/service_requests_spec.rb
  git commit -m "feat: accept FHIR diagnostic reports and list service requests per provider

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 15: Laboratório simulado (`fake-lab`) no compose de dev

**Files:**
- Create: `lib/fake_lab/report.rb`, `lib/fake_lab/client.rb`, `lib/fake_lab/runner.rb`, `lib/fake_lab/run.rb`
- Modify (raiz do monorepo, **fora do git**): `docker-compose.yml`
- Test: `spec/lib/fake_lab_spec.rb`

**Interfaces:**
- Consumes: `GET /fhir/ServiceRequest?status=active` e `POST /fhir/DiagnosticReport` (Task 14).
- Produces: `FakeLab::Report.build(service_request, now: Time.now) -> Hash` (DiagnosticReport do perfil; hemograma `0202020380` e HbA1c `0202010503` com LOINC; outro SIGTAP → um `valueString`); `FakeLab::Client.new(api_url:, city_host:, token:)` com `#service_requests -> Array<Hash>` e `#send_report(hash) -> Integer` (status HTTP); `FakeLab::Runner.tick(client:, logger:) -> Array<[short_code, status]>`. Puro Ruby + Prawn (sem Rails).

- [ ] **Step 1: Write the failing test** `spec/lib/fake_lab_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/fake_lab/runner").to_s

  # Contrato §13 (F-21.7): o fake-lab lê as guias do prestador simulado e
  # envia um laudo do perfil; com fake_lab desligado, o api responde 403 e
  # ele só registra no log. Nunca sobe em produção.
  RSpec.describe "FakeLab" do
    before { exam_city!(fake_lab: true) }

    let(:unit) { create_unit("UBS Fake") }
    let(:doctor) { doctor!(unit) }
    let(:item) do
      exam_catalog_item!(sigtap_code: "0202020380", result_fields: [ { "key" => "hb", "label" => "Hemoglobina", "unit" => "g/dL", "loinc" => "718-7" } ])
    end
    let(:provider) do
      exam_provider!(name: "Laboratório Simulado", simulated: true, token: ExamHelpers::FAKE_TOKEN)
        .tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) }
    end
    let(:host) { "#{TEST_CITY_A.slug}.rotasaude.app" }
    let(:client) { FakeLab::Client.new(api_url: "http://#{host}", city_host: host, token: ExamHelpers::FAKE_TOKEN) }

    # O cliente fala HTTP; aqui o "servidor" é o próprio app Rails (WebMock → Rack),
    # no host da cidade de teste (o mesmo dos request specs).
    before { stub_request(:any, /#{Regexp.escape(host)}/).to_rack(Rails.application) }

    it "o laudo montado passa no esquema do perfil" do
      schema = JSONSchemer.schema(Rails.root.join("config/exams/fhir-diagnostic-report-inbound.json"))
      sr = { "identifier" => [ { "value" => "K7M2Q9XR4T" } ], "code" => { "coding" => [ { "code" => "0202010503" } ] },
             "subject" => { "identifier" => { "system" => "http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf", "value" => "52998224725" } } }
      report = FakeLab::Report.build(sr)
      expect(schema.validate(report).to_a).to eq([])
      expect(Base64.strict_decode64(report.dig("presentedForm", 0, "data"))).to start_with("%PDF-")
      expect(report.to_json).not_to include("Luiz", "Maria")
    end

    it "um ciclo: pede as guias, envia o laudo, o exame fica com resultado; de novo não duplica" do
      order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor, sigtap_code: item.sigtap_code)
      Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: provider.id, scheduled_at: 1.hour.from_now.iso8601, scheduled_place: "Lab simulado" })
      logger = Logger.new(StringIO.new)
      expect(FakeLab::Runner.tick(client: client, logger: logger)).to eq([ [ order.short_code, 201 ] ])
      expect(order.reload.status).to eq("resulted")
      expect(order.result).to have_attributes(source: "fhir", laboratory_name: "Laboratório Simulado")
      expect(FakeLab::Runner.tick(client: client, logger: logger)).to eq([])
    end

    it "fake_lab desligado: 403 e só log" do
      provider
      Platform::Features.set!(city: Current.city, key: "fake_lab", enabled: false, maintainer: ledi_maintainer!)
      log = StringIO.new
      expect(FakeLab::Runner.tick(client: client, logger: Logger.new(log))).to eq([])
      expect(log.string).to include("fake_lab desligado")
    end

    it "nunca em produção" do
      env = { "RAILS_ENV" => "production" }
      out = IO.popen(env, [ "ruby", Rails.root.join("lib/fake_lab/run.rb").to_s ], err: [ :child, :out ], &:read)
      expect(out).to include("fake-lab nunca sobe em produção")
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/lib/fake_lab_spec.rb`. Expected: FAIL.

- [ ] **Step 3: Implemente.** `lib/fake_lab/report.rb`:

  ```ruby
  require "json"
  require "base64"
  require "time"
  require "prawn"

  # Laudo de exemplo do laboratório simulado (ADR 0036; contrato §13): só o
  # perfil de entrada, valores fixos de demonstração, PDF "sem validade" sem
  # nenhum dado do paciente (o CPF vem e volta só no subject).
  module FakeLab
    module Report
      ANALYTES = {
        "0202020380" => [ [ "hb", "718-7", "Hemoglobin [Mass/volume] in Blood", 13.4, "g/dL" ],
                          [ "leuco", "6690-2", "Leukocytes [#/volume] in Blood", 7.2, "10*3/uL" ],
                          [ "plaq", "777-3", "Platelets [#/volume] in Blood", 250, "10*3/uL" ] ],
        "0202010503" => [ [ "hba1c", "4548-4", "Hemoglobin A1c/Hemoglobin.total in Blood", 6.8, "%" ] ]
      }.freeze

      module_function

      def build(service_request, now: Time.now)
        short_code = service_request.dig("identifier", 0, "value")
        sigtap = service_request.dig("code", "coding", 0, "code")
        observations = observations(sigtap)
        { "resourceType" => "DiagnosticReport", "identifier" => [ { "system" => "urn:fake-lab", "value" => "FAKE-#{short_code}" } ],
          "status" => "final", "code" => { "coding" => [ { "system" => "http://www.saude.gov.br/fhir/r4/CodeSystem/BRTabelaSUS", "code" => sigtap } ] },
          "basedOn" => [ { "identifier" => { "value" => short_code } } ], "subject" => service_request["subject"],
          "issued" => now.utc.iso8601, "performer" => [ { "display" => "Laboratório Simulado" } ],
          "presentedForm" => [ { "contentType" => "application/pdf", "data" => Base64.strict_encode64(pdf(short_code, sigtap)) } ],
          "result" => observations.map { |o| { "reference" => "##{o['id']}" } }, "contained" => observations }
      end

      def observations(sigtap)
        rows = ANALYTES[sigtap]
        return [ { "resourceType" => "Observation", "id" => "texto", "status" => "final",
                   "code" => { "coding" => [ { "system" => "http://loinc.org", "code" => "22637-3", "display" => "Pathology report final diagnosis" } ] },
                   "valueString" => "Resultado simulado — sem validade" } ] unless rows

        rows.map do |id, loinc, display, value, unit|
          { "resourceType" => "Observation", "id" => id, "status" => "final",
            "code" => { "coding" => [ { "system" => "http://loinc.org", "code" => loinc, "display" => display } ] },
            "valueQuantity" => { "value" => value, "unit" => unit } }
        end
      end

      def pdf(short_code, sigtap)
        Prawn::Document.new do |doc|
          doc.text "LABORATÓRIO SIMULADO — sem validade", size: 16
          doc.text "Guia #{short_code} · SIGTAP #{sigtap}"
          doc.text "Documento de desenvolvimento do Rota Saúde."
        end.render
      end
      private_class_method :observations, :pdf
    end
  end
  ```

  `lib/fake_lab/client.rb`:

  ```ruby
  require "net/http"
  require "uri"
  require "json"

  module FakeLab
    # Fala com o api pelo endereço interno (FAKE_LAB_API_URL) e diz a cidade
    # pelo Host (FAKE_LAB_CITY_HOST) — o api resolve a cidade pelo host.
    class Client
      MEDIA_TYPE = "application/fhir+json".freeze

      def initialize(api_url:, city_host:, token:)
        @uri = URI(api_url)
        @city_host = city_host
        @token = token
      end

      def service_requests
        response = request(Net::HTTP::Get.new("/fhir/ServiceRequest?status=active"))
        return [] unless response.code == "200"

        JSON.parse(response.body).fetch("entry", []).map { |entry| entry["resource"] }
      end

      def last_status = @last_status

      def send_report(report)
        req = Net::HTTP::Post.new("/fhir/DiagnosticReport")
        req["Content-Type"] = MEDIA_TYPE
        req.body = JSON.generate(report)
        request(req).code.to_i
      end

      private

      def request(req)
        req["Host"] = @city_host
        req["Authorization"] = "Bearer #{@token}"
        req["Accept"] = MEDIA_TYPE
        Net::HTTP.start(@uri.host, @uri.port, use_ssl: @uri.scheme == "https", open_timeout: 5, read_timeout: 30) do |http|
          http.request(req).tap { |response| @last_status = response.code.to_i }
        end
      end
    end
  end
  ```

  `lib/fake_lab/runner.rb`:

  ```ruby
  require_relative "client"
  require_relative "report"

  module FakeLab
    # Um ciclo: pede as guias e envia um laudo para cada uma ainda sem laudo.
    # Nunca loga corpo, CPF nem token — só o código curto e o status.
    module Runner
      module_function

      def tick(client:, logger:)
        requests = client.service_requests
        if client.last_status == 403
          logger.info("[fake-lab] fake_lab desligado (ou exam_fhir_inbound) nesta cidade: nada a fazer")
          return []
        end
        requests.map do |sr|
          code = sr.dig("identifier", 0, "value")
          status = client.send_report(Report.build(sr))
          logger.info("[fake-lab] guia #{code}: #{status}")
          [ code, status ]
        end
      end
    end
  end
  ```

  `lib/fake_lab/run.rb`:

  ```ruby
  # Laboratório simulado do compose de dev (ADR 0036). Nunca em produção.
  abort "fake-lab nunca sobe em produção" if (ENV["RAILS_ENV"] || ENV["RACK_ENV"]).to_s == "production"

  require "logger"
  require_relative "runner"

  client = FakeLab::Client.new(api_url: ENV.fetch("FAKE_LAB_API_URL", "http://api:3000"),
                               city_host: ENV.fetch("FAKE_LAB_CITY_HOST", "curitiba.localhost"),
                               token: ENV.fetch("FAKE_LAB_TOKEN"))
  logger = Logger.new($stdout)
  interval = Integer(ENV.fetch("FAKE_LAB_INTERVAL", "30"))
  loop do
    begin
      FakeLab::Runner.tick(client: client, logger: logger)
    rescue StandardError => e
      logger.warn("[fake-lab] ciclo falhou: #{e.class}")
    end
    sleep interval
  end
  ```

  Como as guias já com resultado saem do `ServiceRequest` (status `resulted`), o segundo ciclo não envia nada; um reenvio por corrida cai na idempotência do `identifier` (`FAKE-<short_code>`).

  No `docker-compose.yml` da **raiz** (fora do git; não commite), depois de `fake-sncr`:

  ```yaml
    fake-lab:
      container_name: fake-lab-dev
      # Laboratório simulado (ADR 0036, módulo 21a): a cada 30 s pede as guias agendadas do prestador
      # "Laboratório Simulado" (token da semente) e envia laudos de exemplo. Com fake_lab desligado, só loga.
      image: rota-saude/api:dev
      working_dir: /rails
      command: bundle exec ruby lib/fake_lab/run.rb
      environment:
        FAKE_LAB_API_URL: ${FAKE_LAB_API_URL:-http://api:3000}
        FAKE_LAB_CITY_HOST: ${FAKE_LAB_CITY_HOST:-curitiba.localhost}
        FAKE_LAB_TOKEN: ${FAKE_LAB_TOKEN:-dev-fake-lab-token-0123456789abcdef0123}
        FAKE_LAB_INTERVAL: ${FAKE_LAB_INTERVAL:-30}
      volumes:
        - ./apps/api:/rails
      depends_on:
        api:
          condition: service_started
      restart: unless-stopped
  ```
  e, no bloco `x-api-env`, `FAKE_LAB_TOKEN: ${FAKE_LAB_TOKEN:-dev-fake-lab-token-0123456789abcdef0123}` (a semente da Task 22 lê o mesmo valor). Confira que `curitiba.localhost` passa no `config.hosts` de development (o dashboard de dev já usa `<slug>.localhost`).

- [ ] **Step 4: Run** `rspec spec/lib/fake_lab_spec.rb`. Expected: PASS. Depois, `docker compose config --services | grep fake-lab` mostra o serviço.

- [ ] **Step 5: Commit** (só o api; o compose fica fora do git):

  ```bash
  git add lib/fake_lab/report.rb lib/fake_lab/client.rb lib/fake_lab/runner.rb lib/fake_lab/run.rb spec/lib/fake_lab_spec.rb
  git commit -m "feat: add simulated laboratory client for the dev compose

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.7 → Done (api + contracts; a tag do contracts segue pendente de autorização).

---

## Fatia 6 — Resultado externo, cidadão, relatório e linha do tempo (F-21.9, F-21.10, F-21.12, F-21.13)

### Task 16: Resultado externo

**Files:**
- Create: `app/commands/exams/record_external_result.rb`, `app/controllers/exams/external_results_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/exams/external_results_spec.rb`

**Interfaces:**
- Consumes: `Exams::{Files.claim!,ResultValues.from_input,Json.result,Access}`, `ClinicalRecord::Access.call`, `ClinicalTerms::SigtapExams.find`, `Patients::Resolve`.
- Produces: `Exams::RecordExternalResult.call(patient:, sigtap_code:, released_on:, laboratory_name:, file_id:, values:, by:, consultation: nil) -> Result(result:)` (`sigtap_unknown`, `missing_pdf`, `invalid_values`, `invalid_released_on`, `invalid_laboratory`); `POST /exams/external_results`.

- [ ] **Step 1: Write the failing test** `spec/requests/exams/external_results_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §6 "Resultado externo" (F-21.9): profissional com leitura do
  # paciente (na consulta) ou a unidade; origem external; ciência dada por
  # quem registra; nunca aparece para o prestador.
  RSpec.describe "Exames — resultado externo", type: :request do
    before { exam_city!; exam_catalog_item! }

    let(:unit) { create_unit("UBS Externo") }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(exam_seq!) }

    def upload(user)
      sign_in_as(user)
      json_post "/exams/uploads", content_type: "application/pdf", data: Base64.strict_encode64(ExamHelpers::PDF_BYTES)
      json_body["upload_id"]
    end

    it "o profissional em contexto registra; vira resultado externo com ciência; evento só com id" do
      attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
      patient = Patients::Resolve.call(citizen).payload[:patient]
      upload_id = upload(doctor)
      json_post "/exams/external_results", patient_id: patient.id, sigtap_code: "0202010503", released_on: Time.zone.today.iso8601,
                                           laboratory_name: "Laboratório Particular", upload_id: upload_id, values: [ { key: "hba1c", value: "8.2" } ]
      expect(response).to have_http_status(:created)
      result = ExamResult.find(json_body["id"])
      expect(result).to have_attributes(origin: "external", source: "unit", exam_order_id: nil, registered_by_user_id: doctor.id)
      expect(json_body).to include("origin" => "external", "acknowledged_at" => be_present, "values" => [ hash_including("value" => "8.2") ])
      expect(DomainEvent.where(name: "exam_result.external_recorded").last.payload).to eq("exam_result_id" => result.id)
      expect(attendance).to be_present
    end

    it "fora de contexto 403; SIGTAP fora do grupo 02 422; sem PDF 422; a recepção registra pelo par" do
      patient = Patients::Resolve.call(citizen).payload[:patient]
      stranger = doctor!(create_unit("UBS Fora"))
      upload_id = upload(stranger)
      json_post "/exams/external_results", patient_id: patient.id, sigtap_code: "0202010503", released_on: Time.zone.today.iso8601,
                                           laboratory_name: "Lab", upload_id: upload_id, values: []
      expect([ response.status, json_body["error"] ]).to eq([ 403, "out_of_context" ])
      reception = staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier")
      upload_id = upload(reception)
      json_post "/exams/external_results", citizen_id: citizen.id, sigtap_code: "0301010064", released_on: Time.zone.today.iso8601,
                                           laboratory_name: "Lab", upload_id: upload_id, values: []
      expect(json_body["error"]).to eq("sigtap_unknown")
      json_post "/exams/external_results", citizen_id: citizen.id, sigtap_code: "0202010503", released_on: Time.zone.today.iso8601,
                                           laboratory_name: "Lab", upload_id: SecureRandom.uuid, values: []
      expect(json_body["error"]).to eq("missing_pdf")
      json_post "/exams/external_results", citizen_id: citizen.id, sigtap_code: "0202010503", released_on: Time.zone.today.iso8601,
                                           laboratory_name: "Lab", upload_id: upload_id, values: []
      expect(response).to have_http_status(:created)
      expect(json_body.keys).to eq(%w[id origin source released_on laboratory_name])
    end
  end
  ```

- [ ] **Step 2: Run.** Expected: FAIL.

- [ ] **Step 3: Implemente** `app/commands/exams/record_external_result.rb`:

  ```ruby
  # Resultado sem pedido (ADR 0036; spec §6): origem external, ciência dada
  # por quem registra (o resultado nasce reconhecido), nunca para prestador.
  module Exams
    module RecordExternalResult
      module_function

      def call(patient:, sigtap_code:, released_on:, laboratory_name:, file_id:, values:, by:, consultation: nil)
        exam = ClinicalTerms::SigtapExams.find(sigtap_code, on: Time.zone.today)
        return Result.fail(:sigtap_unknown) unless exam

        released = parse_date(released_on)
        return Result.fail(:invalid_released_on) unless released && released <= Time.zone.today
        return Result.fail(:invalid_laboratory) unless laboratory_name.is_a?(String) && laboratory_name.strip.length.between?(1, 160)

        checked = ResultValues.from_input(ExamCatalogItem.find_by(sigtap_code: exam.code), values)
        return checked if checked.failure?

        result = nil
        ApplicationRecord.transaction do
          file = Files.claim!(file_id: file_id, user: by)
          next result = Result.fail(:missing_pdf) unless file&.content_type == "application/pdf"

          stored = ExamResult.create!(patient: patient, sigtap_code: exam.code, origin: "external",
                                      source: consultation ? "consultation" : "unit", pdf_file: file,
                                      values: checked.payload[:values].to_json, released_on: released,
                                      laboratory_name: laboratory_name.strip, critical: false, registered_by_user: by,
                                      consultation: consultation)
          DomainEvents.publish("exam_result.external_recorded", exam_result_id: stored.id)
          result = Result.ok(result: stored)
        end
        result
      end

      def parse_date(raw)
        raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(raw) : nil
      rescue Date::Error
        nil
      end
      private_class_method :parse_date
    end
  end
  ```

  `app/controllers/exams/external_results_controller.rb`:

  ```ruby
  # POST /exams/external_results (contrato §3): profissional com leitura do
  # paciente (regra do 19a) ou recepção; consultation_id só da própria autora.
  module Exams
    class ExternalResultsController < BaseController
      def create
        patient = target_patient
        return if performed?

        professional = Current.user.has_role?("health_professional")
        grant = professional ? ClinicalRecord::Access.call(user: Current.user, patient: patient) : nil
        view = grant&.allowed? ? :clinical : :administrative
        unless grant&.allowed? || Current.user.has_role?("citizen_verifier")
          return forbid(grant ? grant.reason.to_s : "missing_role")
        end

        consultation = nil
        if params[:consultation_id].present?
          consultation = uuid?(params[:consultation_id]) &&
                         Consultation.find_by(id: params[:consultation_id], patient_id: patient.id, author_user_id: Current.user.id)
          return not_found unless consultation
        end
        result = Exams::RecordExternalResult.call(patient: patient, sigtap_code: params[:sigtap_code], released_on: params[:released_on],
                                                  laboratory_name: params[:laboratory_name], file_id: params[:upload_id],
                                                  values: params[:values], by: Current.user, consultation: consultation)
        return render_failure(result) if result.failure?

        render json: Exams::Json.result(result.payload[:result], view: view, user: Current.user), status: :created
      end

      private

      # O paciente; nil quando a resposta de erro já saiu.
      def target_patient
        if params[:citizen_id].present?
          citizen = uuid?(params[:citizen_id]) && Citizen.not_erased.find_by(id: params[:citizen_id])
          unless citizen
            not_found
            return nil
          end
          resolved = Patients::Resolve.call(citizen)
          if resolved.failure?
            render_failure(Result.fail(:citizen_not_verified))
            return nil
          end
          resolved.payload[:patient]
        else
          patient = uuid?(params[:patient_id]) && Patient.find_by(id: params[:patient_id])
          not_found unless patient
          patient || nil
        end
      end
    end
  end
  ```

  Rota dentro de `scope "/exams"`: `post "external_results", to: "external_results#create"`.

- [ ] **Step 4: Run.** Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/commands/exams/record_external_result.rb app/controllers/exams/external_results_controller.rb config/routes.rb spec/requests/exams/external_results_spec.rb
  git commit -m "feat: record external exam results without an order

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.9 → Done (lado api).

### Task 17: "Meus exames" do cidadão (wpda)

**Files:**
- Create: `app/services/exams/release.rb`, `app/services/exams/citizen_list.rb`, `app/controllers/citizen_api/exams_controller.rb`
- Modify: `app/services/exams/trail.rb` (`citizen_viewed!`), `config/routes.rb`
- Test: `spec/services/exams/release_spec.rb`, `spec/requests/citizen_api/exams_spec.rb`

**Interfaces:**
- Consumes: `Timeline::Person.for_citizen`, `Timeline::Trail.viewed!`, `ExamOrder`, `ExamResult`.
- Produces: `Exams::Release.policy(sigtap_code) -> String`, `.available?(result) -> Boolean`, `.pickup_at_unit?(sigtap_code) -> Boolean`; `Exams::CitizenList.call(person:, before:, limit:) -> { exams:, next_before: }` (pedidos e resultados externos do paciente, `kind: "order"|"external_result"`, mais novos primeiro); `Exams::Trail.citizen_viewed!(person:, result:)`; rotas `GET /citizen/exams?citizen_id=&before=&limit=` e `GET /citizen/exams/:id/pdf?citizen_id=`.

- [ ] **Step 1: Write the failing tests.** `spec/services/exams/release_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §6 "Cidadão" (F-21.10): immediate ao registrar; after_acknowledgement
  # depois da ciência; never_citizen nunca; fora do catálogo = padrão
  # after_acknowledgement; resultado externo nasce com ciência.
  RSpec.describe Exams::Release do
    before { exam_city! }

    let(:unit) { create_unit("UBS Liberação") }
    let(:doctor) { doctor!(unit) }

    def resulted(sigtap:, policy:)
      exam_catalog_item!(sigtap_code: sigtap, release_policy: policy, result_fields: [])
      order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor, sigtap_code: sigtap)
      Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: exam_provider!(name: "L#{SecureRandom.hex(2)}").id,
                                                                        scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      Exams::RegisterResult.call(order: order.reload, source: "unit", file: pdf_file!(user: doctor), released_on: Time.zone.today.iso8601,
                                 laboratory_name: "Lab", critical: false, values: [], by_user: doctor).payload[:result]
    end

    it "as três políticas" do
      expect(described_class.available?(resulted(sigtap: "0202020380", policy: "immediate"))).to be(true)
      waiting = resulted(sigtap: "0202010503", policy: "after_acknowledgement")
      expect(described_class.available?(waiting)).to be(false)
      Exams::Acknowledge.call(order: waiting.exam_order, by: doctor)
      expect(described_class.available?(waiting.reload)).to be(true)
      never = resulted(sigtap: "0202030300", policy: "never_citizen")
      Exams::Acknowledge.call(order: never.exam_order, by: doctor)
      expect(described_class.available?(never)).to be(false)
      expect(described_class.pickup_at_unit?("0202030300")).to be(true)
      expect(described_class.policy("0202999999")).to eq("after_acknowledgement")
    end
  end
  ```

  (O `sigtap_release!` dos helpers só tem dois códigos de exame: acrescente em `ClinicalRecordHelpers::SIGTAP` `"0202020380" => "HEMOGRAMA COMPLETO"`, `"0202030300" => "PESQUISA DE ANTICORPOS ANTI-HIV-1 + HIV-2 (ELISA)"` e `"0203020030" => "EXAME ANATOMO-PATOLOGICO PARA CONGELAMENTO / PARAFINA POR PECA CIRURGICA OU POR BIOPSIA (EXCETO COLO UTERINO E MAMA)"` — é o mesmo recorte da semente da Task 22. Confira que nenhuma spec do 19 conta os itens desse hash.)

  `spec/requests/citizen_api/exams_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §9 (Divergências D9 e D15) — Review Focus 3: política contornada.
  RSpec.describe "Meus exames (cidadão)", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Cidadão") }
    let(:doctor) { doctor!(unit) }
    let(:patient_and_citizen) { exam_patient!(exam_seq!) }
    let(:patient) { patient_and_citizen.first }
    let(:citizen) { patient_and_citizen.last }

    def resulted(sigtap, policy, critical: false)
      exam_catalog_item!(sigtap_code: sigtap, release_policy: policy, result_fields: [])
      order = exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: sigtap)
      Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: exam_provider!(name: "L#{SecureRandom.hex(2)}").id,
                                                                        scheduled_at: 1.day.from_now.iso8601, scheduled_place: "Rua do Lab, 1" })
      Exams::RegisterResult.call(order: order.reload, source: "unit", file: pdf_file!(user: doctor), released_on: Time.zone.today.iso8601,
                                 laboratory_name: "Lab", critical: critical, values: [], by_user: doctor)
      order.reload
    end

    it "lista andamento e laudo por política; nunca crítico nem motivo de negação; trilha citizen" do
      immediate = resulted("0202020380", "immediate", critical: true)
      hiv = resulted("0202030300", "never_citizen")
      exam_catalog_item!(sigtap_code: "0202010503", requires_authorization: true, result_fields: [])
      denied = exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: "0202010503")
      ApplicationRecord.transaction do
        denied.lock!
        Exams::Transition.call(order: denied, kind: "denied", actor_user: doctor, reason: "Sem indicação clínica descrita")
      end
      sign_in_citizen(citizen.phone)
      get "/citizen/exams", params: { citizen_id: citizen.id }
      expect(response).to have_http_status(:ok)
      exams = json_body["exams"].index_by { |e| e["id"] }
      expect(json_body["verified"]).to be(true)
      expect(exams[immediate.id]).to include("kind" => "order", "label" => "HEMOGRAMA COMPLETO", "status_label" => "Resultado disponível",
                                             "scheduled_place" => "Rua do Lab, 1")
      expect(exams[immediate.id]["result"]).to eq("available" => true, "pickup_at_unit" => false,
                                                   "released_on" => Time.zone.today.iso8601, "pdf_available" => true)
      expect(exams[hiv.id]["result"]).to eq("available" => false, "pickup_at_unit" => true, "released_on" => nil, "pdf_available" => false)
      expect(exams[hiv.id]["status_label"]).to eq("Retire o resultado na sua unidade")
      expect(exams[denied.id]).to include("status_label" => "Pedido não autorizado — procure sua unidade", "result" => nil)
      expect(response.body).not_to include("critical", "Sem indicação", doctor.email_address)
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "citizen", "user_id" => nil)
      get "/citizen/exams/#{immediate.id}/pdf", params: { citizen_id: citizen.id }
      expect(response.media_type).to eq("application/pdf")
      expect(response.headers["Content-Disposition"]).to start_with("attachment")
    end

    it "política contornada: never_citizen, sem ciência, de outro telefone, par declarado (Review Focus 3)" do
      hiv = resulted("0202030300", "never_citizen")
      pending_ack = resulted("0202010503", "after_acknowledgement")
      stranger_patient, stranger = exam_patient!(exam_seq!)
      theirs = exam_order!(patient: stranger_patient, unit: unit, requester: doctor, sigtap_code: "0202010503")
      sign_in_citizen(citizen.phone)
      [ hiv.id, pending_ack.id, theirs.id, SecureRandom.uuid ].each do |id|
        get "/citizen/exams/#{id}/pdf", params: { citizen_id: citizen.id }
        expect([ response.status, json_body["error"] ]).to eq([ 404, "not_found" ]), id
      end
      get "/citizen/exams", params: { citizen_id: stranger.id }
      expect([ response.status, json_body["error"] ]).to eq([ 404, "not_found" ])
      declared = Citizen.create!(cpf: "39053344705", phone: citizen.phone, verification_level: "declared")
      get "/citizen/exams", params: { citizen_id: declared.id }
      expect(json_body).to eq("verified" => false, "exams" => [], "next_before" => nil)
      get "/citizen/exams/#{pending_ack.id}/pdf", params: { citizen_id: declared.id }
      expect([ response.status, json_body["error"] ]).to eq([ 403, "not_verified" ])
    end

    it "resultado externo aparece pela política do exame; interruptor desligado 403" do
      external = ExamResult.create!(patient: patient, sigtap_code: "0202010503", origin: "external", source: "unit",
                                    pdf_file: pdf_file!(user: doctor).tap { |f| f.update!(consumed_at: Time.current) }, values: "[]",
                                    released_on: Time.zone.today, laboratory_name: "Particular", registered_by_user: doctor)
      sign_in_citizen(citizen.phone)
      get "/citizen/exams", params: { citizen_id: citizen.id }
      expect(json_body["exams"].find { |e| e["id"] == external.id }).to include("kind" => "external_result", "status_label" => "Resultado disponível")
      get "/citizen/exams/#{external.id}/pdf", params: { citizen_id: citizen.id }
      expect(response).to have_http_status(:ok)
      Platform::Features.set!(city: Current.city, key: "exam_tracking", enabled: false, maintainer: ledi_maintainer!)
      get "/citizen/exams", params: { citizen_id: citizen.id }
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "exam_tracking")
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL.

- [ ] **Step 3: Implemente.** `app/services/exams/release.rb`:

  ```ruby
  # Liberação ao cidadão (ADR 0036; spec §6), aplicada no servidor: política
  # do exame no catálogo da cidade (padrão after_acknowledgement).
  module Exams
    module Release
      DEFAULT = "after_acknowledgement".freeze

      module_function

      def policy(sigtap_code) = ExamCatalogItem.find_by(sigtap_code: sigtap_code)&.release_policy || DEFAULT
      def pickup_at_unit?(sigtap_code) = policy(sigtap_code) == "never_citizen"

      def available?(result)
        rule = policy(result.sigtap_code)
        return false if rule == "never_citizen"
        return true if result.external?

        status = result.exam_order.status
        return false unless %w[resulted acknowledged].include?(status)

        rule == "immediate" || status == "acknowledged"
      end
    end
  end
  ```

  `app/services/exams/citizen_list.rb`:

  ```ruby
  # "Meus exames" (contrato §9 com D9): pedidos e resultados externos do
  # paciente da pessoa, mais novos primeiro; nada de motivo, crítico, valores
  # nem solicitante. Cursor "<instante>|<id>" (o mesmo das listas do exame).
  module Exams
    module CitizenList
      module_function

      def call(person:, before:, limit:)
        return { exams: [], next_before: nil } unless person.patient_id

        at, id = Cursor.decode(before)
        orders = ExamOrder.where(patient_id: person.patient_id).includes(result: :pdf_file)
        externals = ExamResult.where(patient_id: person.patient_id, origin: "external")
        if at
          orders = orders.where("(exam_orders.created_at, exam_orders.id) < (?, ?)", at, id)
          externals = externals.where("(exam_results.created_at, exam_results.id) < (?, ?)", at, id)
        end
        rows = (orders.order(created_at: :desc, id: :desc).limit(limit + 1).to_a +
                externals.order(created_at: :desc, id: :desc).limit(limit + 1).to_a)
               .sort { |a, b| [ b.created_at, b.id ] <=> [ a.created_at, a.id ] }
        page = rows.first(limit)
        { exams: page.map { |row| row.is_a?(ExamOrder) ? order(row) : external(row) },
          next_before: rows.size > limit ? Cursor.encode(page.last) : nil }
      end

      def order(order)
        result = order.result
        { id: order.id, kind: "order", label: Labels.exam(order.sigtap_code, order.sigtap_competence),
          status_label: status_label(order.status, result, order.sigtap_code), requested_at: order.created_at.iso8601,
          scheduled_at: order.scheduled_at&.iso8601, scheduled_place: order.scheduled_place, result: result && block(result) }
      end

      def external(result)
        { id: result.id, kind: "external_result", label: Labels.exam(result.sigtap_code, result.released_on.strftime("%Y%m")),
          status_label: status_label("acknowledged", result, result.sigtap_code), requested_at: result.created_at.iso8601,
          scheduled_at: nil, scheduled_place: nil, result: block(result) }
      end

      def block(result)
        available = Release.available?(result)
        { available: available, pickup_at_unit: Release.pickup_at_unit?(result.sigtap_code),
          released_on: available ? result.released_on.iso8601 : nil, pdf_available: available }
      end

      def status_label(status, result, sigtap_code)
        return Labels::CITIZEN_STATUS.fetch(status) unless result
        return "Retire o resultado na sua unidade" if Release.pickup_at_unit?(sigtap_code)

        Release.available?(result) ? "Resultado disponível" : "Resultado registrado — aguardando liberação"
      end
      private_class_method :order, :external, :block, :status_label
    end
  end
  ```

  Em `Exams::Trail`, acrescente:

  ```ruby
      def citizen_viewed!(person:, result:)
        DomainEvents.publish("clinical_record.viewed", patient_id: result.patient_id, user_id: nil, access: "citizen",
                                                       reason_code: nil, person_ref: person.ref, exam_result_id: result.id)
      end
  ```

  `app/controllers/citizen_api/exams_controller.rb`:

  ```ruby
  #   GET /citizen/exams?citizen_id=&before=&limit=
  #   GET /citizen/exams/:id/pdf?citizen_id=
  # "Meus exames" (ADR 0036; contrato §9 com D9/D15). Só pares do celular da
  # sessão (senão 404); laudo só para par validado e pela política, aplicada
  # aqui (404 indistinguível). Toda leitura deixa trilha citizen.
  module CitizenApi
    class ExamsController < BaseController
      rate_limit to: 30, within: 10.minutes, only: :pdf, name: "citizen_exam_pdf",
                 by: -> { current_citizen_session&.id || request.remote_ip }, store: RateLimitStore,
                 with: -> { render_error("too_many_requests", :too_many_requests) }
      before_action :require_exam_tracking!

      def index
        citizen = own_citizen
        return unless citizen
        return render(json: { verified: false, exams: [], next_before: nil }) unless citizen.verification_level_verified?

        person = Timeline::Person.for_citizen(citizen)
        page = Exams::CitizenList.call(person: person, before: params[:before], limit: Timeline::Query.limit(params[:limit]))
        Timeline::Trail.viewed!(person: person, access: "citizen")
        render json: { verified: true, exams: page[:exams], next_before: page[:next_before] }
      rescue Exams::Cursor::Invalid
        render_error("invalid_cursor", :unprocessable_entity)
      end

      def pdf
        citizen = own_citizen
        return unless citizen
        return render_error("not_verified", :forbidden) unless citizen.verification_level_verified?

        person = Timeline::Person.for_citizen(citizen)
        result = find_result(person)
        return render_error("not_found", :not_found) unless result && Exams::Release.available?(result)

        Exams::Trail.citizen_viewed!(person: person, result: result)
        response.headers["Cache-Control"] = "no-store"
        send_data result.pdf_file.bytes, type: "application/pdf", disposition: "attachment", filename: "exame-#{result.id[0, 8]}.pdf"
      end

      private

      def require_exam_tracking!
        return if Exams::Gate.usable?(Current.city)

        render json: { error: "feature_disabled", feature: Exams::Gate::KEY }, status: :forbidden
      end

      def own_citizen
        citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
        render_error("not_found", :not_found) unless citizen
        citizen
      end

      # O id é de um pedido (o resultado dele) ou de um resultado externo, do paciente da pessoa.
      def find_result(person)
        id = params[:id]
        return nil unless person.patient_id && id.is_a?(String) && id.match?(Exams::BaseController::UUID)

        ExamOrder.find_by(id: id, patient_id: person.patient_id)&.result ||
          ExamResult.find_by(id: id, patient_id: person.patient_id, origin: "external")
      end
    end
  end
  ```

  Confira como `CitizenApi::HistoryController` exige a sessão (`current_citizen_session` nil → 401 vem do `CitizenAuthentication`); a ordem dos `before_action` deve dar 401 antes do 403 do interruptor, como lá.

  Rotas dentro de `scope "/citizen"`:

  ```ruby
    # Meus exames (ADR 0036; contrato §9, Divergência D9: citizen_id).
    get "exams",         to: "exams#index"
    get "exams/:id/pdf", to: "exams#pdf"
  ```

- [ ] **Step 4: Run** os dois. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/release.rb app/services/exams/citizen_list.rb app/services/exams/trail.rb app/controllers/citizen_api/exams_controller.rb config/routes.rb spec/support/clinical_record_helpers.rb spec/services/exams/release_spec.rb spec/requests/citizen_api/exams_spec.rb
  git commit -m "feat: show citizens their exams with server-side release policies

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.10 → Done (lado api).

### Task 18: Relatório de conferência por prestador e mês (tela e CSV)

**Files:**
- Create: `app/services/exams/report.rb`, `app/controllers/exams/reports_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/exams/reports_spec.rb`

**Interfaces:**
- Consumes: `exam_order_events.closed_stage_due_on` (Task 3/7), `Exams::Access.regulator?`.
- Produces: `Exams::Report.rows(month:, provider_id: nil) -> Array<Hash>`, `.detail(month:, provider_id: nil) -> Array<Array>`, `.month_range(raw) -> Range|nil`; `GET /exams/admin/report?provider_id=&month=YYYY-MM` e `GET /exams/admin/report.csv?provider_id=&month=&detail=true` (municipal_admin ou exam_regulator; sem step-up — só leitura sem dado de paciente).

- [ ] **Step 1: Write the failing test** `spec/requests/exams/reports_spec.rb`:

  ```ruby
  require "rails_helper"
  require "csv"

  # Spec §6 "Relatório de conferência" (F-21.12): por prestador e mês —
  # autorizadas, coletadas, com resultado no prazo e em atraso, pendentes. O
  # CSV detalhado leva só short_code, SIGTAP e datas.
  RSpec.describe "Exames — relatório de conferência", type: :request do
    before { exam_city! }

    let(:unit) { create_unit("UBS Relatório") }
    let(:doctor) { doctor!(unit) }
    let(:item) { exam_catalog_item!(requires_authorization: true, result_days: 5) }
    let(:provider) { exam_provider!(name: "Lab Mês").tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }

    def flow(at:, result_after:)
      travel_to(at) do
        order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor)
        ApplicationRecord.transaction do
          order.lock!
          Exams::Transition.call(order: order, kind: "authorized", actor_user: doctor, changes: { exam_provider_id: provider.id })
          Exams::Transition.call(order: order, kind: "scheduled", actor_user: doctor, changes: { scheduled_at: at + 1.day, scheduled_place: "L" })
          Exams::Transition.call(order: order, kind: "collected", actor_user: doctor)
        end
        travel(result_after) do
          ApplicationRecord.transaction do
            order.lock!
            Exams::Transition.call(order: order, kind: "resulted", actor_user: doctor, event_payload: { source: "unit", critical: false })
          end
        end
        order
      end
    end

    it "conta por prestador no mês, com prazo pelo evento; CSV sem dado de paciente" do
      zone = ActiveSupport::TimeZone["America/Sao_Paulo"]
      on_time = flow(at: zone.local(2026, 10, 2, 9), result_after: 3.days)
      late = flow(at: zone.local(2026, 10, 3, 9), result_after: 9.days)
      sign_in_as(staff_with("reg-#{SecureRandom.hex(2)}@cidade.gov.br", "exam_regulator"))
      get "/exams/admin/report", params: { month: "2026-10" }
      expect(json_body["rows"]).to eq([ { "provider" => { "id" => provider.id, "name" => "Lab Mês" }, "authorized" => 2, "collected" => 2,
                                          "resulted_on_time" => 1, "resulted_late" => 1, "pending" => 0 } ])
      get "/exams/admin/report.csv", params: { month: "2026-10", detail: "true", provider_id: provider.id }
      expect(response.media_type).to eq("text/csv")
      csv = CSV.parse(response.body, headers: true)
      expect(csv.headers).to eq(%w[short_code sigtap_code authorized_on scheduled_on collected_on resulted_on on_time])
      expect(csv.map { |r| [ r["short_code"], r["on_time"] ] }).to contain_exactly([ on_time.short_code, "true" ], [ late.short_code, "false" ])
      expect(response.body).not_to include(on_time.patient.cpf, on_time.patient.display_name.to_s)
      get "/exams/admin/report", params: { month: "2026-13" }
      expect([ response.status, json_body["error"] ]).to eq([ 422, "invalid_month" ])
      sign_in_as(doctor)
      get "/exams/admin/report", params: { month: "2026-10" }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Run.** Expected: FAIL.

- [ ] **Step 3: Implemente** `app/services/exams/report.rb`:

  ```ruby
  # Relatório de conferência (ADR 0036; spec §6): por prestador (o do pedido)
  # e mês no fuso da cidade. "No prazo" = o evento resulted aconteceu até o
  # prazo da etapa que ele fechou (closed_stage_due_on, Desvio 7).
  module Exams
    module Report
      MONTH = /\A(\d{4})-(0[1-9]|1[0-2])\z/
      DETAIL_HEADERS = %w[short_code sigtap_code authorized_on scheduled_on collected_on resulted_on on_time].freeze

      module_function

      def month_range(raw)
        match = raw.to_s.match(MONTH)
        match && Time.zone.local(match[1].to_i, match[2].to_i).all_month
      end

      def rows(month:, provider_id: nil)
        providers = ExamProvider.order(:name)
        providers = providers.where(id: provider_id) if provider_id
        events = events(month, provider_id).group_by { |e| e.exam_order.exam_provider_id }
        pending = ExamOrder.where(status: %w[scheduled collected]).group(:exam_provider_id).count
        providers.filter_map do |provider|
          own = events.fetch(provider.id, [])
          next if own.empty? && pending[provider.id].to_i.zero?

          resulted = own.select { |e| e.kind == "resulted" }
          { provider: { id: provider.id, name: provider.name }, authorized: own.count { |e| e.kind == "authorized" },
            collected: own.count { |e| e.kind == "collected" }, resulted_on_time: resulted.count { |e| on_time?(e) },
            resulted_late: resulted.count { |e| !on_time?(e) }, pending: pending[provider.id].to_i }
        end
      end

      def detail(month:, provider_id: nil)
        events(month, provider_id).group_by(&:exam_order).map do |order, list|
          last = ->(kind) { list.select { |e| e.kind == kind }.max_by(&:occurred_at) }
          resulted = last.("resulted")
          [ order.short_code, order.sigtap_code, *%w[authorized scheduled collected resulted].map { |k| last.(k)&.occurred_at&.to_date&.iso8601 },
            resulted ? on_time?(resulted).to_s : nil ]
        end
      end

      def events(month, provider_id)
        scope = ExamOrderEvent.joins(:exam_order).includes(:exam_order)
                              .where(kind: %w[authorized scheduled collected resulted], occurred_at: month)
                              .where.not(exam_orders: { exam_provider_id: nil })
        scope = scope.where(exam_orders: { exam_provider_id: provider_id }) if provider_id
        scope.to_a
      end

      def on_time?(event) = event.closed_stage_due_on.nil? || event.occurred_at.in_time_zone.to_date <= event.closed_stage_due_on
      private_class_method :events, :on_time?
    end
  end
  ```

  `app/controllers/exams/reports_controller.rb`:

  ```ruby
  # Relatório de conferência (contrato §5): municipal_admin ou regulador.
  module Exams
    class ReportsController < BaseController
      before_action { forbid("missing_role") unless Exams::Access.regulator?(Current.user) }
      before_action :load_month

      def show
        render json: { rows: Exams::Report.rows(month: @month, provider_id: provider_id) }
      end

      def csv
        require "csv"
        body = CSV.generate do |out|
          if params[:detail] == "true"
            out << Exams::Report::DETAIL_HEADERS
            Exams::Report.detail(month: @month, provider_id: provider_id).each { |row| out << row }
          else
            out << %w[provider authorized collected resulted_on_time resulted_late pending]
            Exams::Report.rows(month: @month, provider_id: provider_id).each do |row|
              out << [ row[:provider][:name], *row.values_at(:authorized, :collected, :resulted_on_time, :resulted_late, :pending) ]
            end
          end
        end
        send_data body, type: "text/csv; charset=utf-8", disposition: "attachment", filename: "conferencia-exames-#{params[:month]}.csv"
      end

      private

      def provider_id = uuid?(params[:provider_id]) ? params[:provider_id] : nil

      def load_month
        @month = Exams::Report.month_range(params[:month])
        render(json: { error: "invalid_month" }, status: :unprocessable_entity) unless @month
      end
    end
  end
  ```

  Rotas dentro de `scope "/exams"` (fora do módulo `admin`, mas no caminho do contrato):

  ```ruby
    get "admin/report",     to: "reports#show"
    get "admin/report.csv", to: "reports#csv", format: false
  ```

- [ ] **Step 4: Run.** Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/exams/report.rb app/controllers/exams/reports_controller.rb config/routes.rb spec/requests/exams/reports_spec.rb
  git commit -m "feat: add monthly exam provider reconciliation report with csv

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.12 → Done (lado api).

### Task 19: Fornecedor da linha do tempo `Timeline::Sources::ExamSource` (fecha api#71 para exames)

**Files:**
- Modify: `app/services/timeline.rb` (`KINDS` + `exam`), `app/services/timeline/query.rb` (`SOURCES`)
- Create: `app/services/timeline/sources/exam_source.rb`
- Test: `spec/services/timeline/sources/exam_source_spec.rb`, `spec/requests/timeline_exams_spec.rb`

**Interfaces:**
- Consumes: `Timeline::Sources::Window.apply`, `Timeline::Entry`, `Exams::{Gate,Labels,Release,CitizenList}`.
- Produces: `Timeline::Sources::ExamSource::KIND = "exam"`, `.entries(person:, before:, limit:, from:, to:) -> Array<Timeline::Entry>` — um por pedido (`exam_orders.created_at`) e um por resultado externo (`exam_results.created_at`); `detail_ref` `{ type: "exam_order"|"exam_result", id }`; visão do cidadão com `status_label` e `document: { id, downloadable: true }` só quando `Exams::Release.available?`; vazio com `exam_tracking` inutilizável ou sem paciente.

- [ ] **Step 1: Write the failing tests.** `spec/services/timeline/sources/exam_source_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §10 (F-21.13, api#71): o exame na linha do tempo — profissional
  # vê pedido, etapa e "resultado registrado" (nunca valores); cidadão vê o
  # andamento e o laudo pela política; janela e cursor como os outros.
  RSpec.describe Timeline::Sources::ExamSource do
    before { exam_city!; Platform::Features.set!(city: Current.city, key: "patient_timeline", enabled: true, maintainer: ledi_maintainer!) }

    let(:unit) { create_unit("UBS Linha") }
    let(:doctor) { doctor!(unit) }
    let(:patient) { exam_patient!(exam_seq!).first }
    let(:person) { Timeline::Person.for_patient(patient) }

    it "pedidos e resultado externo, na ordem total, com a visão do cidadão pela política" do
      exam_catalog_item!(sigtap_code: "0202020380", release_policy: "immediate", result_fields: [])
      first = travel_to(2.days.ago) { exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: "0202020380") }
      Exams::Steps.schedule(order: first, actor_user: doctor, params: { provider_id: exam_provider!.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      Exams::RegisterResult.call(order: first.reload, source: "unit", file: pdf_file!(user: doctor), released_on: Time.zone.today.iso8601,
                                 laboratory_name: "Lab", critical: true, values: [], by_user: doctor)
      external = ExamResult.create!(patient: patient, sigtap_code: "0202010503", origin: "external", source: "unit",
                                    pdf_file: pdf_file!(user: doctor).tap { |f| f.update!(consumed_at: Time.current) }, values: "[]",
                                    released_on: Time.zone.today, laboratory_name: "Particular", registered_by_user: doctor)
      entries = described_class.entries(person: person, before: nil, limit: 10, from: nil, to: nil)
      expect(entries.map(&:source_id)).to eq([ external.id, first.id ])
      order_entry = entries.last
      expect(order_entry).to have_attributes(kind: "exam", title: "HEMOGRAMA COMPLETO", status: "resulted",
                                             summary: "Resultado registrado", detail_ref: { type: "exam_order", id: first.id })
      expect(order_entry.citizen).to include(status: "resulted", summary: "Resultado disponível")
      expect(order_entry.document).to eq(id: first.id, downloadable: true)
      expect(entries.first.detail_ref).to eq(type: "exam_result", id: external.id)
      expect(entries.map(&:to_h).to_s).not_to include("critical")
    end

    it "com exam_tracking desligado não entra nada; sem paciente, nada" do
      exam_order!(patient: patient, unit: unit, requester: doctor)
      Platform::Features.set!(city: Current.city, key: "exam_tracking", enabled: false, maintainer: ledi_maintainer!)
      expect(described_class.entries(person: person, before: nil, limit: 10, from: nil, to: nil)).to eq([])
      Platform::Features.set!(city: Current.city, key: "exam_tracking", enabled: true, maintainer: ledi_maintainer!)
      empty = Timeline::Person.new(citizen_ids: [], patient_id: nil, verified: true)
      expect(described_class.entries(person: empty, before: nil, limit: 10, from: nil, to: nil)).to eq([])
    end

    it "pagina pela janela comum sem repetir" do
      ids = 4.times.map { |i| travel_to(i.minutes.ago) { exam_order!(patient: patient, unit: unit, requester: doctor).id } }
      page = Timeline::Query.call(person: person, kinds: %w[exam], limit: 3)
      rest = Timeline::Query.call(person: person, kinds: %w[exam], limit: 3, before: Timeline::Cursor.decode(page.next_before))
      expect((page.entries + rest.entries).map(&:source_id)).to eq(ids)
      expect(rest.next_before).to be_nil
    end
  end
  ```

  `spec/requests/timeline_exams_spec.rb`:

  ```ruby
  require "rails_helper"

  # A linha do tempo do profissional e o histórico do cidadão passam a ter o kind exam.
  RSpec.describe "Linha do tempo com exames", type: :request do
    before do
      exam_city!
      Platform::Features.set!(city: Current.city, key: "patient_timeline", enabled: true, maintainer: ledi_maintainer!)
    end

    it "profissional em contexto e cidadão validado veem o exame; kinds aceita exam" do
      unit = create_unit("UBS LT")
      doctor = doctor!(unit)
      patient, citizen = exam_patient!(exam_seq!)
      order = exam_order!(patient: patient, unit: unit, requester: doctor)
      consulting_attendance!(unit, citizen: citizen, doctor: doctor)
      sign_in_as(doctor)
      get "/attendance/timeline", params: { patient_id: patient.id, kinds: %w[exam] }
      expect(json_body["entries"].map { |e| [ e["kind"], e["detail_ref"] ] }).to eq([ [ "exam", { "type" => "exam_order", "id" => order.id } ] ])
      sign_in_citizen(citizen.phone)
      get "/citizen/history", params: { citizen_id: citizen.id, kinds: %w[exam] }
      expect(json_body["entries"].first).to include("kind" => "exam", "status" => "awaiting_schedule", "document" => nil)
    end
  end
  ```

- [ ] **Step 2: Run** os dois. Expected: FAIL.

- [ ] **Step 3: Implemente.** Em `app/services/timeline.rb`: `KINDS = %w[triage attendance screening appointment consultation clinical_document controlled_notification campaign exam].freeze`. Em `Timeline::Query::SOURCES`: `"exam" => Sources::ExamSource`.

  `app/services/timeline/sources/exam_source.rb`:

  ```ruby
  # Exames (ADR 0036; contrato §10; fecha api#71 para exames). Profissional:
  # exame, situação e etapa (ou "Resultado registrado"); nunca valores nem
  # crítico. Cidadão: andamento e laudo pela política (Exams::Release). Vazio
  # sem exam_tracking utilizável ou sem paciente.
  module Timeline
    module Sources
      module ExamSource
        KIND = "exam".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          return [] unless person.patient_id && Exams::Gate.usable?(Current.city)

          orders = ::ExamOrder.where(patient_id: person.patient_id).includes(:unit, :requester_user, :result)
          externals = ::ExamResult.where(patient_id: person.patient_id, origin: "external").includes(registered_by_user: :professional)
          Window.apply(orders, kind: KIND, column: "exam_orders.created_at", id_column: "exam_orders.id", before: before, from: from, to: to)
                .limit(limit).map { |order| order_entry(order) } +
            Window.apply(externals, kind: KIND, column: "exam_results.created_at", id_column: "exam_results.id", before: before, from: from, to: to)
                  .limit(limit).map { |result| external_entry(result) }
        end

        def order_entry(order)
          title = Exams::Labels.exam(order.sigtap_code, order.sigtap_competence)
          summary = if order.result then "Resultado registrado"
                    elsif order.stage then "#{Exams::Labels::STAGE.fetch(order.stage)} pendente"
                    else Exams::Labels::CITIZEN_STATUS.fetch(order.status)
                    end
          Entry.new(kind: KIND, source_id: order.id, occurred_at: order.created_at, unit: { id: order.unit_id, name: order.unit.name },
                    professional: { name: order.requester_user && Labels.professional_name(order.requester_user), cbo_code: nil },
                    title: title, summary: summary, status: order.status, detail_ref: { type: "exam_order", id: order.id },
                    citizen: { title: title, summary: citizen_label(order.status, order.result, order.sigtap_code), status: order.status,
                               professional_name: order.requester_user && Labels.professional_name(order.requester_user) },
                    document: document(order.result, order.id))
        end

        def external_entry(result)
          title = Exams::Labels.exam(result.sigtap_code, result.released_on.strftime("%Y%m"))
          Entry.new(kind: KIND, source_id: result.id, occurred_at: result.created_at, unit: nil,
                    professional: { name: Labels.professional_name(result.registered_by_user), cbo_code: nil },
                    title: title, summary: "Resultado externo — #{result.laboratory_name}", status: "external",
                    detail_ref: { type: "exam_result", id: result.id },
                    citizen: { title: title, summary: citizen_label("acknowledged", result, result.sigtap_code), status: "external",
                               professional_name: nil },
                    document: document(result, result.id))
        end

        def citizen_label(status, result, sigtap_code)
          return Exams::Labels::CITIZEN_STATUS.fetch(status) unless result
          return "Retire o resultado na sua unidade" if Exams::Release.pickup_at_unit?(sigtap_code)

          Exams::Release.available?(result) ? "Resultado disponível" : "Resultado registrado — aguardando liberação"
        end

        def document(result, id) = result && Exams::Release.available?(result) ? { id: id, downloadable: true } : nil
        private_class_method :order_entry, :external_entry, :citizen_label, :document
      end
    end
  end
  ```

  Confira se `Timeline::Json.citizen` aceita `entry.unit` nulo (`entry.unit && { name: … }` — aceita) e se o cartão do cidadão no wpda trata `kind: "exam"` com `document` sem `short_code` (o plano do wpda é o dono).

- [ ] **Step 4: Run** os dois e `rspec spec/services/timeline spec/requests/timeline_spec.rb spec/requests/citizen_api/history_spec.rb spec/requests/pec_timeline_spec.rb`. Expected: PASS (as listas de `kinds` nas specs do módulo 20 são de "fora da lista" — `exam` agora é válido; se uma spec do 20 testava `kinds: ["exam"]` como inválido, troque por um valor de fato fora).

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/timeline.rb app/services/timeline/query.rb app/services/timeline/sources/exam_source.rb spec/services/timeline/sources/exam_source_spec.rb spec/requests/timeline_exams_spec.rb
  git commit -m "feat: add exams to the patient timeline (closes api#71 for exams)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.13 → Done. Com autorização, comente em api#71 que exames entraram (vacinas e procedimentos seguem abertos).

### Task 19b: Rotas de apoio das telas do dashboard (paciente, catálogo, prestadores, solicitantes, resultados externos)

Pedido do plano do dashboard (D2–D5, D8). Os outros pontos dele já estão nas tasks donas: upload JSON (Task 6), `label` no catálogo do admin (Task 4), `consultation_id` no exame e o filtro (Task 9), `result_fields` no detalhe e no portal (Tasks 7 e 12), `{ orders, denied }` na ciência pendente (Task 13), agendar no portal sem `provider_id` e `overdue_count` (Task 12), `has_fhir_token` e o convite respondendo `<invitation>` (Task 5), `exam_requests[].priority` (Task 8), 410 `link_expired` (Task 6), relatório também para `exam_regulator` (Task 18), atrasos da cidade inteira (Tasks 9 e 11), `unit_id` na guia (Task 9).

**Files:**
- Create: `app/controllers/exams/lookups_controller.rb`
- Modify: `app/controllers/exams/external_results_controller.rb` (`index`), `config/routes.rb`, `config/initializers/domain_events.rb` + `spec/initializers/domain_events_bindings_spec.rb` (`exam.patient_looked_up`)
- Test: `spec/requests/exams/lookups_spec.rb`

**Interfaces:**
- Consumes: `Patients::Resolve`, `CitizenIdentity::Cpf.normalize`, `Exams::{Access,Steps.enabled?,Labels,Json,Trail}`, `ClinicalRecord::{Access,Trail}`.
- Produces:
  - `POST /exams/patient_lookup { "cpf" }` → `{ "patient": { "id", "display_name", "cpf_masked" } }`; 404 `patient_not_found`; 409 `citizen_not_verified`; papéis `citizen_verifier`, `health_professional`, `municipal_admin` (senão 403 `missing_role`); 30/min por usuário (429 `too_many_requests`); evento `exam.patient_looked_up { patient_id, user_id }`.
  - `GET /exams/catalog` → `{ "items": [ { "sigtap_code", "label", "result_fields" } ] }` (ativos; qualquer papel de exames, inclusive `exam_provider`).
  - `GET /exams/providers?sigtap_code=` → `{ "providers": [ { "id", "name" } ] }` (ativos; com `sigtap_code`, só os habilitados — regra de `Exams::Steps.enabled?`); unidade, regulador e admin.
  - `GET /exams/requesters?unit_id=` → `{ "requesters": [ { "user_id", "name" } ] }` (usuários `health_professional` com vínculo ativo na unidade); quem é da unidade, regulador ou admin.
  - `GET /exams/external_results?patient_id=` → `{ "results": [ <exam_result> + "sigtap_code", "label" ] }` pela leitura do 19a, com trilha.

- [ ] **Step 1: Write the failing test** `spec/requests/exams/lookups_spec.rb`:

  ```ruby
  require "rails_helper"

  # Rotas de apoio pedidas pelo plano do dashboard (D2–D5, D8).
  RSpec.describe "Exames — rotas de apoio", type: :request do
    before { exam_city!; exam_catalog_item! }

    let(:unit) { create_unit("UBS Apoio") }
    let(:doctor) { doctor!(unit) }
    let(:reception) { staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier") }

    it "patient_lookup: par validado vira paciente; declarado 409; ninguém 404; sem papel 403; evento só com ids" do
      citizen = verified_citizen!(exam_seq!)
      sign_in_as(reception)
      json_post "/exams/patient_lookup", cpf: CitizenIdentity::Cpf.mask(citizen.cpf)
      expect(json_body["patient"]).to include("id" => citizen.reload.patient_id, "cpf_masked" => citizen.patient.cpf_masked)
      expect(DomainEvent.where(name: "exam.patient_looked_up").last.payload)
        .to eq("patient_id" => citizen.patient_id, "user_id" => reception.id)
      Citizen.create!(cpf: "39053344705", phone: "+5541999990002", verification_level: "declared")
      json_post "/exams/patient_lookup", cpf: "390.533.447-05"
      expect([ response.status, json_body["error"] ]).to eq([ 409, "citizen_not_verified" ])
      json_post "/exams/patient_lookup", cpf: "111.444.777-35"
      expect([ response.status, json_body["error"] ]).to eq([ 404, "patient_not_found" ])
      sign_in_as(staff_with("lab-#{SecureRandom.hex(2)}@lab.com.br", "exam_provider"))
      json_post "/exams/patient_lookup", cpf: "111.444.777-35"
      expect([ response.status, json_body["error"] ]).to eq([ 403, "missing_role" ])
    end

    it "catálogo legível, prestadores do exame e solicitantes da unidade" do
      enabled = exam_provider!(name: "Faz")
      exam_provider!(name: "Não faz", items: [])
      doctor
      sign_in_as(reception)
      get "/exams/catalog"
      expect(json_body["items"]).to eq([ { "sigtap_code" => "0202010503", "label" => "DOSAGEM DE HEMOGLOBINA GLICOSILADA",
                                           "result_fields" => [ { "key" => "hba1c", "label" => "Hemoglobina glicada", "unit" => "%", "loinc" => "4548-4" } ] } ])
      get "/exams/providers", params: { sigtap_code: "0202010503" }
      expect(json_body["providers"]).to eq([ { "id" => enabled.id, "name" => "Faz" } ])
      get "/exams/providers"
      expect(json_body["providers"].map { |p| p["name"] }).to eq([ "Faz", "Não faz" ])
      get "/exams/requesters", params: { unit_id: unit.id }
      expect(json_body["requesters"]).to eq([ { "user_id" => doctor.id, "name" => Screenings::Json.staff_name(doctor) } ])
      sign_in_as(doctor!(create_unit("UBS Outra")))
      get "/exams/requesters", params: { unit_id: unit.id }
      expect([ response.status, json_body["error"] ]).to eq([ 403, "not_unit_staff" ])
    end

    it "resultados externos do paciente pela leitura do 19a" do
      citizen = verified_citizen!(exam_seq!)
      consulting_attendance!(unit, citizen: citizen, doctor: doctor)
      patient = Patients::Resolve.call(citizen).payload[:patient]
      result = ExamResult.create!(patient: patient, sigtap_code: "0202010503", origin: "external", source: "unit",
                                  pdf_file: pdf_file!(user: doctor).tap { |f| f.update!(consumed_at: Time.current) }, values: "[]",
                                  released_on: Time.zone.today, laboratory_name: "Particular", registered_by_user: doctor)
      sign_in_as(doctor)
      get "/exams/external_results", params: { patient_id: patient.id }
      expect(json_body["results"].map { |r| r.values_at("id", "sigtap_code", "label") })
        .to eq([ [ result.id, "0202010503", "DOSAGEM DE HEMOGLOBINA GLICOSILADA" ] ])
      sign_in_as(doctor!(create_unit("UBS Fora")))
      get "/exams/external_results", params: { patient_id: patient.id }
      expect([ response.status, json_body["error"] ]).to eq([ 403, "out_of_context" ])
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/requests/exams/lookups_spec.rb`. Expected: FAIL.

- [ ] **Step 3: Implemente** `app/controllers/exams/lookups_controller.rb`:

  ```ruby
  # Rotas de apoio das telas (pedido do dashboard D2–D5): achar o paciente pelo
  # CPF (no corpo, nunca na URL), o catálogo legível, os prestadores de um
  # exame e os solicitantes de uma unidade.
  module Exams
    class LookupsController < BaseController
      LOOKUP_ROLES = %w[citizen_verifier health_professional municipal_admin].freeze
      STAFF_ROLES = %w[citizen_verifier health_professional municipal_admin exam_regulator].freeze

      module RateLimitStore
        def self.increment(...) = Rails.cache.increment(...)
      end

      rate_limit to: 30, within: 1.minute, only: :patient, name: "exam_patient_lookup", store: RateLimitStore,
                 by: -> { Current.user&.id || request.remote_ip },
                 with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

      def patient
        return forbid("missing_role") unless LOOKUP_ROLES.any? { |role| Current.user.has_role?(role) }

        digits = params[:cpf].is_a?(String) ? CitizenIdentity::Cpf.normalize(params[:cpf]) : nil
        pairs = digits ? Citizen.not_erased.where(cpf: digits) : Citizen.none
        citizen = pairs.verification_level_verified.order(:created_at, :id).first
        unless citizen
          status, error = pairs.exists? ? [ :conflict, "citizen_not_verified" ] : [ :not_found, "patient_not_found" ]
          return render(json: { error: error }, status: status)
        end
        resolved = Patients::Resolve.call(citizen)
        return render(json: { error: "citizen_not_verified" }, status: :conflict) if resolved.failure?

        patient = resolved.payload[:patient]
        DomainEvents.publish("exam.patient_looked_up", patient_id: patient.id, user_id: Current.user.id)
        render json: { patient: { id: patient.id, display_name: patient.display_name, cpf_masked: patient.cpf_masked } }
      end

      def catalog
        allowed = STAFF_ROLES.any? { |role| Current.user.has_role?(role) } || Exams::Access.provider_of(Current.user)
        return forbid("missing_role") unless allowed

        items = ExamCatalogItem.where(active: true).order(:sigtap_code).map do |item|
          { sigtap_code: item.sigtap_code, label: Exams::AdminJson.label(item.sigtap_code), result_fields: item.result_fields }
        end
        render json: { items: items }
      end

      def providers
        return forbid("missing_role") unless STAFF_ROLES.any? { |role| Current.user.has_role?(role) }

        scope = ExamProvider.where(active: true).order(:name)
        if params[:sigtap_code].present?
          item = ExamCatalogItem.find_by(sigtap_code: params[:sigtap_code].to_s)
          scope = scope.where(id: ExamProviderItem.where(exam_catalog_item_id: item.id).select(:exam_provider_id)) if item
        end
        render json: { providers: scope.map { |p| { id: p.id, name: p.name } } }
      end

      def requesters
        unit_id = params[:unit_id]
        return not_found unless uuid?(unit_id) && HealthUnit.exists?(id: unit_id)
        return forbid("not_unit_staff") unless Exams::Access.unit_staff?(Current.user, unit_id) || Exams::Access.regulator?(Current.user)

        users = User.where(deactivated_at: nil)
                    .joins(:memberships).merge(Membership.active.where(role: "health_professional"))
                    .where(id: Professional.joins(:links).merge(ProfessionalLink.active.where(health_unit_id: unit_id)).select(:user_id))
                    .distinct.includes(:professional).to_a
        render json: { requesters: users.map { |u| { user_id: u.id, name: Screenings::Json.staff_name(u) } }.sort_by { |r| r[:name].to_s } }
      end
    end
  end
  ```

  Em `Exams::ExternalResultsController`, a ação `index`:

  ```ruby
      def index
        patient = uuid?(params[:patient_id]) && Patient.find_by(id: params[:patient_id])
        return not_found unless patient

        grant = ClinicalRecord::Access.call(user: Current.user, patient: patient)
        return forbid(grant.reason.to_s) unless grant.allowed?

        ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant)
        results = ExamResult.where(patient_id: patient.id, origin: "external").order(created_at: :desc).limit(100).map do |result|
          Exams::Json.result(result, view: :clinical, user: Current.user)
                     .merge(sigtap_code: result.sigtap_code, label: Exams::Labels.exam(result.sigtap_code, result.released_on.strftime("%Y%m")))
        end
        render json: { results: results }
      end
  ```

  Rotas dentro de `scope "/exams"`:

  ```ruby
    post "patient_lookup",   to: "lookups#patient"
    get  "catalog",          to: "lookups#catalog"
    get  "providers",        to: "lookups#providers"
    get  "requesters",       to: "lookups#requesters"
    get  "external_results", to: "external_results#index"
  ```

  `config/initializers/domain_events.rb`: `DomainEvents.bind "exam.patient_looked_up", to: []`, e a lista da spec de bindings da Task 3 ganha o nome. O CPF vai no corpo (`:cpf` já é filtrado do log). `Professional has_many :links` existe (`app/models/professional.rb`); confira a associação `User has_one :professional`.

- [ ] **Step 4: Run** `rspec spec/requests/exams/lookups_spec.rb spec/initializers/domain_events_bindings_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/controllers/exams/lookups_controller.rb app/controllers/exams/external_results_controller.rb config/routes.rb config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/requests/exams/lookups_spec.rb
  git commit -m "feat: add exam lookup routes for patients, catalog, providers and requesters

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---


## Fatia 7 — Botão no PEC (F-21.11; só se a Task 0a passou)

### Task 20: "Solicitar exames" na página do PEC

**Files:**
- Modify: `app/services/pec_external/page.rb` (`timeline(... exams:)`), `app/controllers/pec_timelines_controller.rb` (`html(... form_action:)`, `show`)
- Create: `app/services/pec_external/exam_page.rb`, `app/controllers/pec_exams_controller.rb`
- Modify: `config/routes.rb`, `lib/pec_simulator.rb` (nada a mudar além de conferir: o botão vem da própria página)
- Test: `spec/requests/pec_exams_spec.rb`

**Interfaces:**
- Consumes: `PecExternal::ViewSession.resume(token, city:) -> Grant|nil`, `PecExternal::Page.layout`, `Exams::{Gate.pec_usable?,Open}`, `Patients::Resolve`.
- Produces: `PecExternal::Page.timeline(display_name:, page:, token:, nonce:, exams: false)`; `PecExternal::ExamPage.form(display_name:, items:, token:, nonce:, notice: nil, error: nil) -> String`; `PecExamsController#form` (`POST /pec/exams/form`) e `#create` (`POST /pec/exams`), corpo `application/x-www-form-urlencoded` com `token` (a sessão curta; `:token` já é filtrado do log), `sigtap_code`, `priority`.

- [ ] **Step 1: Write the failing test** `spec/requests/pec_exams_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §6 "Botão no PEC" (Divergência D8): o botão na página do PEC leva o
  # token da sessão curta num formulário; nasce o pedido com origem pec, o
  # profissional e o paciente da chamada; sem cookie; CSP form-action 'self'.
  RSpec.describe "Solicitar exames pelo PEC", type: :request do
    let(:unit) { create_unit("UBS Norte").tap { |u| u.update!(cnes: "1234567") } }
    let(:nurse) { screener!(unit).tap { |u| u.professional.update!(cpf: "52998224725") } }
    let(:citizen) { verified_citizen!(exam_seq!, social_name: "Seu Zé") }

    before do
      timeline_city!(record_mode: "integrated", clinical_record: false, pec: true)
      pec_settings!
      %w[exam_tracking exam_pec_bridge].each do |key|
        Platform::Features.set!(city: Current.city, key: key, enabled: true, maintainer: ledi_maintainer!)
      end
      sigtap_release!
      exam_catalog_item!
      nurse
      walk_in_attendance!(unit, citizen: citizen)
    end

    def open_page = get(signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567"))
    def token_in(body) = body[/name="token" value="([^"]+)"/, 1].then { |v| CGI.unescapeHTML(v.to_s) }

    it "a página traz o botão; o formulário lista o catálogo; o pedido nasce com origem pec" do
      open_page
      expect(response.body).to include("Solicitar exames")
      expect(response.headers["Content-Security-Policy"]).to include("form-action 'self'")
      token = token_in(response.body)
      post "/pec/exams/form", params: { token: token }
      expect(response.body).to include("DOSAGEM DE HEMOGLOBINA GLICOSILADA", "Seu Zé")
      expect(response.headers["Content-Security-Policy"]).to include("frame-ancestors https://pec.cidade.gov.br", "form-action 'self'")
      expect(response.headers).not_to have_key("Set-Cookie")
      post "/pec/exams", params: { token: token, sigtap_code: "0202010503", priority: "urgent" }
      order = ExamOrder.last
      expect(order).to have_attributes(origin: "pec", priority: "urgent", requester_user_id: nurse.id, unit_id: unit.id,
                                       patient_id: citizen.reload.patient_id)
      expect(response.body).to include("Pedido registrado", order.short_code)
      expect(response.body).not_to include(citizen.cpf)
    end

    it "sem token, interruptor desligado e exame fora do catálogo" do
      post "/pec/exams/form", params: { token: "lixo" }
      expect([ response.status, response.body ]).to match([ 401, a_string_including("pec_session_expired") ])
      open_page
      token = token_in(response.body)
      post "/pec/exams", params: { token: token, sigtap_code: "0202020380", priority: "routine" }
      expect(response.body).to include("sigtap_not_in_catalog")
      expect(ExamOrder.count).to eq(0)
      Platform::Features.set!(city: Current.city, key: "exam_pec_bridge", enabled: false, maintainer: ledi_maintainer!)
      post "/pec/exams/form", params: { token: token }
      expect([ response.status, response.body ]).to match([ 403, a_string_including("pec_exam_bridge_disabled") ])
      get signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567", at: 2.seconds.ago) # URL nova (uso único)
      expect(response.body).to include("Linha do tempo")
      expect(response.body).not_to include("Solicitar exames")
      expect(response.headers["Content-Security-Policy"]).to include("form-action 'none'")
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/requests/pec_exams_spec.rb`. Expected: FAIL.

- [ ] **Step 3: Implemente.** Em `PecExternal::Page.timeline`, a assinatura vira `timeline(display_name:, page:, token:, nonce:, exams: false)` e, logo depois do `<h1>`:

  ```ruby
        #{exams ? %(<form method="post" action="/pec/exams/form"><input type="hidden" name="token" value="#{h(token)}"><button type="submit">Solicitar exames</button></form>) : ""}
  ```

  Em `PecTimelinesController`: `html(body, status: :ok, form_action: "'none'")` usa `form-action #{form_action}` na CSP; em `show`:

  ```ruby
    exams = Exams::Gate.pec_usable?(Current.city)
    html(PecExternal::Page.timeline(display_name: grant.person.display_name, page: page, token: token, nonce: @nonce, exams: exams),
         form_action: exams ? "'self'" : "'none'")
  ```

  `app/services/pec_external/exam_page.rb`:

  ```ruby
  # Formulário "Solicitar exames" dentro da página do PEC (ADR 0036;
  # Divergência D8): HTML do servidor, escapado, sem script; o token da
  # sessão curta vai num campo oculto (sem cookie de terceiro).
  module PecExternal
    module ExamPage
      PRIORITIES = { "routine" => "Rotina", "urgent" => "Urgente" }.freeze

      module_function

      def form(display_name:, items:, token:, nonce:, notice: nil, error: nil)
        options = items.map { |item| %(<option value="#{h(item[:code])}">#{h(item[:label])}</option>) }.join
        priorities = PRIORITIES.map { |value, label| %(<option value="#{value}">#{label}</option>) }.join
        Page.layout(<<~HTML, nonce: nonce)
          <h1>Solicitar exames — #{h(display_name)}</h1>
          #{notice ? %(<p class="notice">#{h(notice)}</p>) : ""}
          #{error ? %(<p class="notice">Não foi possível registrar o pedido. Código: <code>#{h(error)}</code></p>) : ""}
          <form method="post" action="/pec/exams">
            <input type="hidden" name="token" value="#{h(token)}">
            <p><label>Exame <select name="sigtap_code" required>#{options}</select></label></p>
            <p><label>Prioridade <select name="priority">#{priorities}</select></label></p>
            <p><button type="submit">Registrar pedido</button></p>
          </form>
        HTML
      end

      def h(text) = ERB::Util.html_escape(text.to_s)
    end
  end
  ```

  `app/controllers/pec_exams_controller.rb`:

  ```ruby
  # "Solicitar exames" pela ponte do PEC (ADR 0036; contrato §8 com D8). Mesmo
  # host, cabeçalhos e página de recusa da página do PEC (herda de
  # PecTimelinesController); sem sessão do Rota Saúde: o token da sessão curta
  # do módulo 20 vem no corpo do formulário e é relido a cada passo (vínculo,
  # interruptores, pares). O pedido nasce com origem pec.
  class PecExamsController < PecTimelinesController
    rate_limit to: 120, within: 1.minute, only: %i[form create], name: "pec_exams", by: -> { request.remote_ip },
               store: RateLimitStore, with: -> { failure("too_many_requests", status: :too_many_requests) }
    before_action :resume_grant

    def form = render_form

    def create
      citizen = Citizen.where(id: @grant.person.citizen_ids).order(:created_at, :id).first
      resolved = citizen && Patients::Resolve.call(citizen)
      return render_form(error: "pec_citizen_not_verified") unless resolved&.ok?

      opened = Exams::Open.call(patient: resolved.payload[:patient], sigtap_code: params[:sigtap_code].to_s,
                                priority: params[:priority].presence || "routine", unit: @grant.unit, origin: "pec",
                                requester_user: @grant.user, created_by: @grant.user)
      return render_form(error: opened.reason.to_s) if opened.failure?

      order = opened.payload[:order]
      render_form(notice: "Pedido registrado: #{Exams::Labels.exam(order.sigtap_code, order.sigtap_competence)} — guia #{order.short_code}")
    end

    private

    def resume_grant
      @grant = PecExternal::ViewSession.resume(params[:token], city: Current.city)
      return failure("pec_session_expired", status: :unauthorized) unless @grant

      failure("pec_exam_bridge_disabled") unless Exams::Gate.pec_usable?(Current.city)
    end

    def render_form(notice: nil, error: nil)
      items = ExamCatalogItem.where(active: true).order(:sigtap_code).map do |item|
        { code: item.sigtap_code, label: Exams::Labels.exam(item.sigtap_code, Time.zone.today.strftime("%Y%m")) }
      end
      html(PecExternal::ExamPage.form(display_name: @grant.person.display_name, items: items, token: params[:token], nonce: @nonce,
                                      notice: notice, error: error),
           form_action: "'self'")
    end
  end
  ```

  Rotas, ao lado de `/pec/timeline`:

  ```ruby
  # "Solicitar exames" (ADR 0036; contrato §8 com D8): formulário com o token da sessão curta.
  post "/pec/exams/form", to: "pec_exams#form"
  post "/pec/exams",      to: "pec_exams#create"
  ```

  O `PecExamsController` herda `show`/`more` de `PecTimelinesController`, mas as rotas não apontam para eles (confira com `bin/rails routes -g pec`). Ele não inclui `Authentication`, então a guarda de JSON por cookie não se aplica (não há cookie). Se `failure` for `private` no pai, a herança basta.

- [ ] **Step 4: Run** `rspec spec/requests/pec_exams_spec.rb spec/requests/pec_timeline_spec.rb spec/services/pec_external`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/pec_external/page.rb app/services/pec_external/exam_page.rb app/controllers/pec_timelines_controller.rb app/controllers/pec_exams_controller.rb config/routes.rb spec/requests/pec_exams_spec.rb
  git commit -m "feat: request exams from the PEC external systems page

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  Card F-21.11 → Done (lado api). **Se a Task 0a falhou, esta task não existe:** o card F-21.11 vai para "fora do escopo" (com autorização) e o contrato §8 sai.

---

## Fatia 8 — Invariantes, semente, prova e rollout

### Task 21: Invariantes do ADR 0036

**Files:**
- Create: `spec/invariants/exam_tracking_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 2–20.
- Produces: a guarda de regressão dos cinco invariantes do ADR 0036 (cada exemplo diz a mutação que o derruba).

- [ ] **Step 1: Write the test** `spec/invariants/exam_tracking_invariants_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0036, Invariantes. Cada exemplo nomeia a mutação que o quebra.
  RSpec.describe "Invariantes do rastreamento de exames (ADR 0036)", type: :request do
    before { exam_city!(fhir: true) }

    let(:unit) { create_unit("UBS Invariantes") }
    let(:doctor) { doctor!(unit) }
    let(:item) { exam_catalog_item! }
    let(:provider) { exam_provider!(token: ExamHelpers::FAKE_TOKEN).tap { |p| ExamProviderItem.find_or_create_by!(exam_provider: p, exam_catalog_item: item) } }

    def full_cycle
      order = exam_order!(patient: exam_patient!(exam_seq!).first, unit: unit, requester: doctor)
      Exams::Steps.schedule(order: order, actor_user: doctor, params: { provider_id: provider.id, scheduled_at: 1.day.from_now.iso8601, scheduled_place: "L" })
      Exams::Steps.collect(order: order.reload, actor_user: doctor)
      Exams::RegisterResult.call(order: order.reload, source: "unit", file: pdf_file!(user: doctor), released_on: Time.zone.today.iso8601,
                                 laboratory_name: "Lab", critical: false, values: [ { "key" => "hba1c", "value" => "7.7" } ], by_user: doctor)
      Exams::Acknowledge.call(order: order.reload, by: doctor)
      order.reload
    end

    # Mutação: tirar o exam_orders_guard, ou gravar o status sem passar pela tabela FLOW.
    it "a situação é derivada da trilha, e a trilha não muda" do
      order = full_cycle
      status = "awaiting_schedule"
      (order.events.map(&:kind) - %w[requested escalated]).each do |kind|
        expect(Exams::Transition::FLOW.fetch(kind)[:from]).to include(status)
        status = Exams::Transition::FLOW.fetch(kind)[:to]
      end
      expect(status).to eq(order.status)
      # (o UPDATE do cache sem evento de outra transação é provado fora da fixture
      # transacional em spec/models/exam_tracking_schema_spec.rb)
      sql = ->(statement) { ApplicationRecord.transaction(requires_new: true) { ApplicationRecord.connection.execute(statement) } }
      expect { sql.("DELETE FROM exam_order_events WHERE exam_order_id = '#{order.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { sql.("UPDATE exam_orders SET sigtap_code = '0202020380' WHERE id = '#{order.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /identity never changes/)
    end

    # Mutação: tirar o exam_provider_id do find no portal, ou dar unit_staff? a exam_provider.
    it "prestador nunca vê guia de outro prestador nem prontuário" do
      order = full_cycle
      other = exam_provider!(name: "Outro Lab")
      lab = staff_with("lab-#{SecureRandom.hex(2)}@lab.com.br", "exam_provider").tap { |u| ExamProviderUser.create!(exam_provider: other, user: u) }
      sign_in_as(lab)
      json_post "/exams/provider/orders/#{order.id}/collect"
      expect(response).to have_http_status(:not_found)
      get "/exams/orders/#{order.id}"
      expect(response).to have_http_status(:not_found)
      get "/clinical_record/patients/#{order.patient_id}"
      expect(response.status).to be_in([ 403, 404 ])
      get "/exams/provider/orders"
      expect(json_body["orders"]).to eq([])
    end

    # Mutação: devolver o PDF sem Exams::Release, ou aceitar par declarado.
    it "laudo ao cidadão só pela política e só a par validado" do
      ExamCatalogItem::POLICIES.each do |policy|
        item.update!(release_policy: policy)
        result = full_cycle.result
        expected = policy != "never_citizen"
        expect(Exams::Release.available?(result)).to be(expected), policy
      end
    end

    # Mutação: tirar `environments:` do fake_lab, o abort do run.rb, ou a checagem de simulated no FHIR.
    it "nenhum fake_lab em produção" do
      expect(Platform::Features.catalog(env: "production").map(&:key)).not_to include("fake_lab")
      expect(Exams::Gate.fake_lab_usable?(Current.city, env: "production")).to be(false)
      provider.update!(simulated: true)
      post "/fhir/DiagnosticReport", params: "{}", headers: { "CONTENT_TYPE" => "application/fhir+json", "Authorization" => "Bearer #{ExamHelpers::FAKE_TOKEN}" }
      expect(JSON.parse(response.body)).to eq("error" => "feature_disabled", "feature" => "fake_lab")
    end

    # Mutação: pôr valor, CPF, nome, motivo ou token num evento; guardar o token em claro.
    it "eventos só com ids e códigos; token FHIR só como hash" do
      token = Exams::RotateFhirToken.call(provider: provider).payload[:token]
      order = full_cycle
      allowed = %w[exam_order_id exam_result_id exam_provider_id origin priority source critical by_team reason patient_id user_id]
      DomainEvent.where("name LIKE 'exam_%'").find_each do |event|
        expect(event.payload.keys - allowed).to eq([]), event.name
        expect(event.payload.to_json).not_to include("7.7", order.patient.cpf, token)
      end
      escalation = DomainEvent.where(name: "exam_order.escalated").pluck(:payload).map { |p| p["reason"] }
      expect(escalation - ExamOrderEvent::ESCALATION_REASONS).to eq([])
      expect(ExamProvider.where(id: provider.id).pluck(:fhir_token_digest).first).to eq(Digest::SHA256.hexdigest(token))
      expect(ApplicationRecord.connection.select_all("SELECT * FROM exam_providers").to_a.to_s).not_to include(token)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/invariants/exam_tracking_invariants_spec.rb`. Expected: PASS (as Tasks 2–20 já entregam o comportamento). Para cada exemplo, aplique a mutação do comentário à mão, rode e confirme que ele **falha**; desfaça.

- [ ] **Step 3: Commit**

  ```bash
  git add spec/invariants/exam_tracking_invariants_spec.rb
  git commit -m "test: guard ADR 0036 exam tracking invariants

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 22: Semente de dev realista

**Files:**
- Modify: `lib/sigtap_sample.rb` (`PROCEDURES` com os quatro exames), `spec/services/terminology/sigtap_reader_spec.rb` (contagem)
- Create: `lib/exam_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/exam_crew_spec.rb`

**Interfaces:**
- Consumes: `SigtapSample`, `Terminology::Import`, `ClinicalRecordCrew.ensure_citizen` (privado, via `send`, como a `TimelineCrew`), `SignatureCrew.ensure_totp`/`.otpauth_uri`, `Exams::{Open,Transition,Files}`, `TimelineCrew` (roda antes: Maringá já integrada com `pec_external_view`).
- Produces: `ExamCrew.seed_platform! -> { sigtap:, republished: Boolean }`, `ExamCrew.seed_current_city(slug:, ddd:, password:) -> Hash` (idempotente); `ExamCrew::FAKE_LAB_TOKEN_ENV = "FAKE_LAB_TOKEN"`, `DEFAULT_FAKE_LAB_TOKEN = "dev-fake-lab-token-0123456789abcdef0123"`.

- [ ] **Step 1: Write the failing test** `spec/lib/exam_crew_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/signature_crew").to_s
  require Rails.root.join("lib/professional_crew").to_s
  require Rails.root.join("lib/screening_crew").to_s
  require Rails.root.join("lib/clinical_record_crew").to_s
  require Rails.root.join("lib/exam_crew").to_s

  # Semente do módulo 21a no banco de teste (nunca no de dev). Dev é fictício
  # mas imita o real: SIGTAP reais, LOINC reais, prestadores coerentes,
  # anti-HIV e anatomopatológico sem laudo no wpda, o laboratório simulado
  # com o token do compose. Idempotente.
  RSpec.describe ExamCrew do
    let!(:city) { clinical_city!(record_mode: "record", enabled: true) }
    let(:slug) { city.slug }

    before do
      create_default_protocol!
      admin = staff_with("admin@#{slug}.demo", "municipal_admin")
      SignatureCrew.seed_current_city(slug: slug, password: "dev-password")
      ProfessionalCrew.seed_current_city(slug: slug, password: "dev-password", admin: admin)
      ScreeningCrew.seed_platform!
      ScreeningCrew.seed_current_city(slug: slug, ddd: "41")
      Maintainer.create!(email_address: "dev@local", password: "s3nha-forte-1", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    end
    after { Current.reset }

    it "SIGTAP com os quatro exames; catálogo, prestadores, usuários e pedidos de exemplo; duas vezes = mesma coisa" do
      expect(described_class.seed_platform![:sigtap]).to eq(Time.zone.today.strftime("%Y%m"))
      %w[0202020380 0202010503 0202030300 0203020030].each do |code|
        expect(ClinicalTerms::SigtapExams.find(code, on: Time.zone.today)).to be_present, code
      end
      2.times { described_class.seed_current_city(slug: slug, ddd: "41", password: "dev-password") }
      expect(Exams::Gate.usable?(city)).to be(true)
      expect(ExamCatalogItem.pluck(:sigtap_code, :release_policy, :requires_authorization)).to contain_exactly(
        [ "0202020380", "immediate", false ], [ "0202010503", "after_acknowledgement", false ],
        [ "0202030300", "never_citizen", false ], [ "0203020030", "never_citizen", true ])
      expect(ExamCatalogItem.find_by(sigtap_code: "0202010503").result_fields.first["loinc"]).to eq("4548-4")
      simulated = ExamProvider.find_by!(name: "Laboratório Simulado")
      expect(simulated).to have_attributes(simulated: true, fhir_token_digest: Digest::SHA256.hexdigest(ExamCrew::DEFAULT_FAKE_LAB_TOKEN))
      expect(ExamProvider.where(simulated: false).count).to eq(2)
      lab = User.find_by!(email_address: "laboratorio@#{slug}.demo")
      expect(lab.has_role?("exam_provider")).to be(true)
      expect(ExamProviderUser.find_by(user: lab).exam_provider.name).to eq("Laboratório Municipal Central")
      expect(User.find_by!(email_address: "regulacao@#{slug}.demo").has_role?("exam_regulator")).to be(true)
      expect(ExamOrder.where("external_reference LIKE 'SEED-%'").pluck(:status)).to contain_exactly("scheduled", "awaiting_authorization", "scheduled")
      expect(ExamOrder.find_by(external_reference: "SEED-3").overdue).to include(stage: "collect")
      expect(ExamOrder.find_by(external_reference: "SEED-1").exam_provider).to eq(simulated)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/lib/exam_crew_spec.rb`. Expected: FAIL.

- [ ] **Step 3: Implemente.** Em `SigtapSample::PROCEDURES` acrescente (códigos e nomes reais da Tabela SUS — confira cada um em `sigtap.datasus.gov.br` na competência corrente e corrija o nome se divergir):

  ```ruby
      [ "0202020380", "HEMOGRAMA COMPLETO", "2", "I", 0, 9999 ],
      [ "0202010503", "DOSAGEM DE HEMOGLOBINA GLICOSILADA", "2", "I", 0, 9999 ],
      [ "0202030300", "PESQUISA DE ANTICORPOS ANTI-HIV-1 + HIV-2 (ELISA)", "2", "I", 0, 9999 ],
      [ "0203020030", "EXAME ANATOMO-PATOLOGICO PARA CONGELAMENTO / PARAFINA POR PECA CIRURGICA OU POR BIOPSIA (EXCETO COLO UTERINO E MAMA)", "2", "I", 0, 9999 ]
  ```
  e, em `spec/services/terminology/sigtap_reader_spec.rb`, `sigtap_procedures: 9` (CBOs, CIDs e instrumentos não mudam).

  `lib/exam_crew.rb`:

  ```ruby
  require "tmpdir"
  require "prawn"
  require Rails.root.join("lib/sigtap_sample").to_s
  require_relative "clinical_record_crew"
  require_relative "signature_crew"

  # Semente de dev do módulo 21a (ADR 0036). Dev é fictício mas imita o real:
  # SIGTAP e LOINC reais; catálogo com hemograma (laudo imediato), HbA1c (após
  # ciência), anti-HIV e anatomopatológico (nunca no wpda; o anatomopatológico
  # exige autorização); três prestadores (municipal, credenciado e o
  # laboratório simulado, cujo token é o do compose — FAKE_LAB_TOKEN);
  # laboratorio@ (portal) e regulacao@ (regulação) com TOTP fixo; três guias
  # de exemplo em Curitiba (uma para o fake-lab, uma na fila de regulação,
  # uma atrasada). Curitiba (record) liga exam_tracking, exam_fhir_inbound e
  # fake_lab; Maringá (integrada) liga exam_tracking e exam_pec_bridge. Roda
  # depois da TimelineCrew. Idempotente. O token só vai para a cidade como hash.
  class ExamCrew
    DEV_MAINTAINER = "dev@local".freeze
    FAKE_LAB_TOKEN_ENV = "FAKE_LAB_TOKEN".freeze
    DEFAULT_FAKE_LAB_TOKEN = "dev-fake-lab-token-0123456789abcdef0123".freeze
    EXAM_CODES = %w[0202020380 0202010503 0202030300 0203020030].freeze
    CATALOG = [
      { sigtap_code: "0202020380", requires_authorization: false, release_policy: "immediate", result_days: 7,
        result_fields: [ { "key" => "hb", "label" => "Hemoglobina", "unit" => "g/dL", "loinc" => "718-7" },
                         { "key" => "ht", "label" => "Hematócrito", "unit" => "%", "loinc" => "4544-3" },
                         { "key" => "leuco", "label" => "Leucócitos", "unit" => "10*3/uL", "loinc" => "6690-2" },
                         { "key" => "plaq", "label" => "Plaquetas", "unit" => "10*3/uL", "loinc" => "777-3" } ] },
      { sigtap_code: "0202010503", requires_authorization: false, release_policy: "after_acknowledgement", result_days: 10,
        result_fields: [ { "key" => "hba1c", "label" => "Hemoglobina glicada", "unit" => "%", "loinc" => "4548-4" } ] },
      { sigtap_code: "0202030300", requires_authorization: false, release_policy: "never_citizen", result_days: 15,
        result_fields: [ { "key" => "anti_hiv", "label" => "Anti-HIV 1+2", "unit" => nil, "loinc" => "31201-7" } ] },
      { sigtap_code: "0203020030", requires_authorization: true, release_policy: "never_citizen", result_days: 45,
        result_fields: [ { "key" => "conclusao", "label" => "Conclusão", "unit" => nil, "loinc" => "22637-3" } ] }
    ].freeze
    PROVIDERS = [
      { name: "Laboratório Municipal Central", kind: "municipal", codes: EXAM_CODES - %w[0203020030], result_days: nil },
      { name: "Clínica Diagnóstica Credenciada", kind: "accredited", codes: %w[0203020030], result_days: 40 },
      { name: "Laboratório Simulado", kind: "accredited", codes: %w[0202020380 0202010503], result_days: 2, simulated: true }
    ].freeze
    USERS = [
      { prefix: "laboratorio", role: "exam_provider", provider: "Laboratório Municipal Central",
        secret_env: "DEV_LAB_OTP_SECRET", default_secret: "NRQWE33SMF2G64TJN4WXE33UMEWXGYLV" },
      { prefix: "regulacao", role: "exam_regulator", provider: nil,
        secret_env: "DEV_REGULATOR_OTP_SECRET", default_secret: "OJSWO5LMMFRWC3ZNOJXXIYJNONQXKZDF" }
    ].freeze

    class << self
      def seed_platform!
        competence = Time.zone.today.strftime("%Y%m")
        release = Terminology::Sigtap.release_for(competence)
        have = release ? SigtapProcedure.where(release_id: release.id, code: EXAM_CODES).count : 0
        return { sigtap: competence, republished: false } if release&.version == competence && have == EXAM_CODES.size

        Dir.mktmpdir do |dir|
          result = Terminology::Import.call(kind: "sigtap", version: competence,
                                            path: SigtapSample.write_to(dir, competence: competence), by: "db:seed")
          raise "semente de exames: SIGTAP recusada (#{result.reason} #{result.message})" if result.failure?
        end
        { sigtap: competence, republished: true }
      end

      def seed_current_city(slug:, ddd:, password:)
        maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
        return { switch: "desligado (sem mantenedor #{DEV_MAINTAINER})" } unless maintainer

        city = City.find(Current.city.id)
        switches = city.record_mode == "integrated" ? %w[exam_tracking exam_pec_bridge] : %w[exam_tracking exam_fhir_inbound fake_lab]
        switches.each { |key| Platform::Features.set!(city: city, key: key, enabled: true, maintainer: maintainer) }
        catalog!
        providers!
        users = USERS.map { |spec| ensure_user(spec, slug: slug, password: password) }
        orders = city.record_mode == "record" ? sample_orders!(slug, ddd) : 0
        { switch: switches.join(", "), users: users, orders: orders }
      end

      private

      def catalog!
        CATALOG.each do |attrs|
          item = ExamCatalogItem.find_or_initialize_by(sigtap_code: attrs[:sigtap_code])
          item.update!(attrs.except(:sigtap_code).merge(active: true)) if item.new_record?
        end
      end

      def providers!
        PROVIDERS.each do |spec|
          provider = ExamProvider.find_or_create_by!(name: spec[:name]) do |p|
            p.kind = spec[:kind]
            p.simulated = spec.fetch(:simulated, false)
            p.cnes = format("%07d", Zlib.crc32("exam-provider:#{spec[:name]}") % 9_000_000 + 1_000_000)
          end
          spec[:codes].each do |code|
            item = ExamCatalogItem.find_by!(sigtap_code: code)
            ExamProviderItem.find_or_create_by!(exam_provider: provider, exam_catalog_item: item) { |pi| pi.result_days = spec[:result_days] }
          end
          next unless provider.simulated

          digest = ExamProvider.digest(ENV.fetch(FAKE_LAB_TOKEN_ENV, DEFAULT_FAKE_LAB_TOKEN))
          provider.update!(fhir_token_digest: digest, fhir_token_rotated_at: Time.current, fhir_token_revoked_at: nil) if provider.fhir_token_digest != digest
        end
      end

      def ensure_user(spec, slug:, password:)
        user = User.find_or_initialize_by(email_address: "#{spec[:prefix]}@#{slug}.demo")
        user.password = password if user.new_record?
        user.save!
        Membership.find_or_create_by!(user: user, role: spec[:role], revoked_at: nil) { |m| m.granted_at = Time.current }
        if spec[:provider]
          ExamProviderUser.find_or_create_by!(user: user) { |pu| pu.exam_provider = ExamProvider.find_by!(name: spec[:provider]) }
        end
        SignatureCrew.ensure_totp(user, secret_env: spec[:secret_env], default_secret: spec[:default_secret])
        { email: user.email_address, role: spec[:role], otpauth_uri: SignatureCrew.otpauth_uri(user) }
      end

      # Três guias da recepção para a cidadã da semente do prontuário, pedidas
      # pela médica da semente: (1) hemograma agendado no laboratório simulado
      # (o fake-lab devolve o laudo); (2) anatomopatológico na fila de
      # regulação; (3) HbA1c agendada há 40 dias no laboratório municipal
      # (atrasada na coleta). Datas passadas pelo `now:` dos comandos.
      def sample_orders!(slug, ddd)
        citizen = ClinicalRecordCrew.send(:ensure_citizen, slug, ddd)
        patient = Patients::Resolve.call(citizen).payload.fetch(:patient)
        doctor = User.find_by!(email_address: "profissional@#{slug}.demo")
        reception = User.find_by!(email_address: "recepcao@#{slug}.demo")
        unit = HealthUnit.find_by!(name: ClinicalRecordCrew::UNIT)
        [ [ "SEED-1", "0202020380", "Laboratório Simulado", Time.current ],
          [ "SEED-2", "0203020030", nil, 2.days.ago ],
          [ "SEED-3", "0202010503", "Laboratório Municipal Central", 40.days.ago ] ].count do |reference, code, provider_name, at|
          next false if ExamOrder.exists?(external_reference: reference)

          guide = Exams::Files.store(bytes: guide_pdf(reference), content_type: "application/pdf", user: reception).payload.fetch(:file)
          guide.update!(consumed_at: at)
          order = Exams::Open.call(patient: patient, sigtap_code: code, priority: "routine", unit: unit, origin: "reception_guide",
                                   requester_user: doctor, guide_file: guide, external_reference: reference, created_by: reception, now: at)
                             .payload.fetch(:order)
          if provider_name
            provider = ExamProvider.find_by!(name: provider_name)
            ApplicationRecord.transaction do
              order.lock!
              Exams::Transition.call(order: order, kind: "scheduled", actor_user: reception, now: at,
                                     changes: { exam_provider_id: provider.id, scheduled_at: at + 1.day, scheduled_place: "#{provider_name} — recepção" })
            end
          end
          true
        end
      end

      def guide_pdf(reference)
        Prawn::Document.new do |doc|
          doc.text "GUIA DE EXAME SIMULADA — desenvolvimento", size: 14
          doc.text "Referência #{reference}"
        end.render
      end
    end
  end
  ```

  Em `db/seeds.rb`: `require Rails.root.join("lib/exam_crew").to_s` com os outros; depois do bloco de terminologias, `exams = ExamCrew.seed_platform!` e o `puts "[seeds] SIGTAP exames . #{exams.inspect}"`; dentro do laço das cidades, **depois** da `TimelineCrew`:

  ```ruby
        # ── Rastreamento de exames (módulo 21a, ADR 0036) ─────────────────────
        # Depois da linha do tempo: Maringá já é integrada (exam_pec_bridge).
        exams = ExamCrew.seed_current_city(slug: slug, ddd: ddd, password: password)
        puts "[seeds] exames ...... #{exams[:switch]}; #{exams[:orders]} guias novas"
        Array(exams[:users]).each { |u| puts "[seeds]               #{u[:email]} (#{u[:role]}) / #{password} → #{u[:otpauth_uri]}" }
  ```

  A semente nunca imprime o token do laboratório simulado (é o do compose, documentado no `docker-compose.yml`).

- [ ] **Step 4: Run** `rspec spec/lib/exam_crew_spec.rb spec/services/terminology/sigtap_reader_spec.rb spec/lib/record_mode_crew_spec.rb spec/lib/timeline_crew_spec.rb spec/lib/clinical_record_crew_spec.rb`. Expected: PASS. (Os crews que varrem `lib/*_crew.rb` atrás de prefixos de telefone não são afetados: a `ExamCrew` não cria cidadão novo.)

- [ ] **Step 5: Commit**

  ```bash
  git add lib/sigtap_sample.rb lib/exam_crew.rb db/seeds.rb spec/lib/exam_crew_spec.rb spec/services/terminology/sigtap_reader_spec.rb
  git commit -m "feat: seed realistic exam catalog, providers, users and sample guides

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 23: Revisão final, suíte completa, api na porta 3041, prova no navegador e runbook de rollout

**Files:**
- Create: `docs/operacao/rollout-rastreamento-de-exames.md` (repo `docs`)

- [ ] **Step 1: Varredura de vazamento.**
  - `grep -rn 'DomainEvents.publish("exam\|DomainEvents.publish("clinical_record' -A3 apps/api/.claude/mod21/app` — só ids, `origin`, `priority`, `source`, `critical`, `by_team`, `reason` (código de escalonamento), `access`, `person_ref` (multilinha: confira cada chamada inteira).
  - `grep -rn "Rails.logger" apps/api/.claude/mod21/app/services/exams apps/api/.claude/mod21/app/commands/exams apps/api/.claude/mod21/app/controllers/exams apps/api/.claude/mod21/app/controllers/fhir apps/api/.claude/mod21/app/controllers/pec_exams_controller.rb` — nenhum.
  - `grep -rn "Platform.audit(" apps/api/.claude/mod21/app | grep -i exam` — nenhum (nenhum evento de plataforma novo; se aparecer, ele entra em `R18_PLATFORM_EVENT_NAMES`).
  - `grep -rnE "fhir_token[^_]" apps/api/.claude/mod21/app | grep -v digest` — o token em claro só existe em `Exams::RotateFhirToken` (devolvido uma vez) e no `Authorization` lido por `Fhir::BaseController`.
- [ ] **Step 2: Suíte completa** (worker parado; avise as outras sessões; bancos `_mod21` recriados):

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod21 -w /rails/.claude/mod21 api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde. Spec antiga que fixa listas (catálogo de interruptores, bindings, papéis, rotas, `PRIVILEGED_ROLES`, contagem do SIGTAP de exemplo) → acrescente o do módulo 21a (é o contrato); nunca afrouxe asserção de texto livre. Meça o tempo por exemplo (~0,10 s com host calmo; >0,15 é regressão).
- [ ] **Step 3:** `docker compose exec -T -w /rails/.claude/mod21 api bundle exec rubocop <arquivos tocados>`; corrija só o que a regra do projeto aponta.
- [ ] **Step 4: api na porta 3041.** Com autorização do usuário, uma etapa de cada vez: `bin/rails city:migrate:all` (do worktree) e `bin/rails db:seed`. Depois, sem derrubar o principal:

  ```bash
  docker compose run -d --name api-mod21 -p 127.0.0.1:3041:3041 -w /rails/.claude/mod21 \
    -e CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5192 -e CITY_WPDA_BASE_TEMPLATE=http://%{slug}.localhost:5193 \
    -e PEC_SIMULATOR_CITY_BASE=http://%{slug}.localhost:3041 \
    api bin/rails server -b 0.0.0.0 -p 3041 -P tmp/pids/server-mod21.pid
  FAKE_LAB_API_URL=http://api-mod21:3041 docker compose up -d fake-lab
  ```
  Os Vite do dashboard (5192) e do wpda (5193) apontam `VITE_API_PROXY_TARGET` para `http://api-mod21:3041` (planos do dashboard e do wpda; o prefixo `/exams` entra no proxy do dashboard). `docker logs fake-lab-dev` mostra `guia <código>: 201` para a SEED-1 em até 30 s.
- [ ] **Step 5: Prova manual no navegador** (login, senha e TOTP da semente de dev podem ser mostrados no chat se o usuário pedir):
  1. **Pedido → regulação → agenda pelo prestador → resultado FHIR → ciência (Curitiba, `http://curitiba.localhost:5192/dashboard/`):** a médica (`profissional@`) finaliza uma consulta pedindo hemograma urgente e anatomopatológico → os dois aparecem com situação; `regulacao@` autoriza o anatomopatológico para a Clínica Credenciada; `laboratorio@` (portal) vê só as guias do Laboratório Municipal Central; o hemograma, agendado pela unidade no Laboratório Simulado, recebe o laudo do `fake-lab` (crítico se o fake-lab marcar) e entra em "resultados aguardando ciência" da médica; ela dá ciência.
  2. **Atraso e escalonamento:** a SEED-3 aparece no painel "Exames atrasados" (etapa coleta); rode `Exams::DailyJob.perform_now` no console do worktree com um exame `resulted` de prazo vencido → ele aparece na fila da equipe da enfermeira, que dá ciência com motivo.
  3. **Cidadão nas três políticas (`http://curitiba.localhost:5193/wpda/`):** hemograma com laudo (imediato), HbA1c "aguardando liberação" até a ciência, anti-HIV "retire o resultado na sua unidade"; nenhum motivo de negação nem "crítico".
  4. **Guia da recepção** (`recepcao@`): cadastra a guia com foto/PDF e solicitante digitado (nome + CNS) → o exame nasce com origem guia.
  5. **Resultado externo** na consulta → aparece na aba de exames e no wpda.
  6. **Relatório** do mês por prestador e o CSV detalhado (só código, SIGTAP e datas).
  7. **Botão no PEC (Maringá; só se a Task 0a passou):** `http://pec-sim.localhost:3041/pec-simulator?city=maringa` → "Abrir sistemas externos" → "Solicitar exames" → registrar → o pedido nasce com origem `pec`; `docker logs api-mod21 | grep "pec/exams"` sem CPF nem token.
  8. **Maintenance (`maintenance.localhost:5177`, `dev@local`):** os quatro interruptores com o `missing` do mecanismo genérico; `fake_lab` some se o maintenance apontar para um api em `RAILS_ENV=production` (não teste em produção: a spec prova).
- [ ] **Step 6: Runbook** `docs/operacao/rollout-rastreamento-de-exames.md` (repo docs), com:

  ```markdown
  # Rollout — rastreamento do exame (módulo 21a, ADR 0036)

  ## Ordem
  1. contracts: tag `exams-v1.0.0` (autorização do usuário).
  2. api: publicar a imagem; rodar `bin/rails city:migrate:all` DESSA imagem antes de cortar tráfego
     (migração de cidade 20261013900001, irreversível; triggers por db/city_triggers.sql no mesmo rake).
  3. dashboard e wpda; 4. maintenance; 5. catálogo público.

  ## Ligar numa cidade (produção não tem maintenance: rake/console, como os interruptores do 19c)
  - `exam_tracking` (modo record ou integrated) → catálogo de exames e prestadores pelo admin municipal
    (step-up) → usuários do portal (convite) e do regulador.
  - `exam_fhir_inbound` só com laboratório real homologado: token gerado no admin (mostrado uma vez),
    entregue ao laboratório por canal seguro; teste com um laudo de homologação.
  - `exam_pec_bridge` só com `pec_external_view` já em produção na cidade e a prova do PEC da cidade.
  - `fake_lab` não existe em produção (nem aparece no catálogo).

  ## Gates de go-live
  - Primeiro laboratório real na entrada FHIR: perfil, mapeamento SIGTAP↔LOINC dos campos do catálogo,
    identificação do paciente por CPF/CNS (sistemas RNDS), limite de 60/min e corpo até 15 MB no proxy.
  - Botão no PEC: prova no PEC da cidade (versão de produção; sandbox do iframe; envio de formulário).
  - Arranjo de exames e regulação de cada cidade-piloto (SISREG, consórcio, planilha).
  - Volume de PDFs por banco de cidade (até 10 MB por arquivo; acompanhar o tamanho de exam_files).
  - Proxy repassa `/fhir/*` e `/pec/exams*` ao api, com Host e X-Forwarded-Proto.

  ## Desligar
  - `exam_tracking` desligado: o pedido do 19a segue como requisição; exames existentes ficam guardados.
  - Token FHIR revogado no admin → 401 imediato para o laboratório.
  ```
  Commit no repo docs: `docs: add exam tracking rollout runbook` (com o `Co-Authored-By`).
- [ ] **Step 7: Pare.** Merge, push, tag e board (cards em **Verified** só depois da prova do usuário; página de status do módulo no padrão do módulo 01) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: **contracts** → **api** → dashboard e wpda → maintenance. Ao voltar o checkout para a main: `DROP DATABASE` dos bancos `_mod21` (sem pedir de novo, conferindo conexões ativas), `docker rm -f api-mod21`, `docker compose stop fake-lab` (o serviço fica no compose para os próximos).
- [ ] **Step 8: Pendências para o board de pendências de ciclo** (cards só com autorização): primeiro laboratório real (FHIR) e o mapeamento SIGTAP↔LOINC; botão no PEC real; REL/RELC na RNDS e quem é o integrador; arranjo de regulação por cidade-piloto; volume de PDFs (armazenamento fora do banco, se preciso); equipe de referência por paciente (hoje = equipes da unidade do exame); aviso por SMS/e-mail de resultado (fora do 21a); 21b (laboratório municipal).

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3 interruptores (os quatro, `fake_lab` só fora de produção) → Task 2 (+ Desvio 1/D1); maintenance pelo mecanismo genérico (plano do maintenance).
- §4 dados (catálogo, prestadores, itens com prazos, pedido com cache e trilha só de acréscimo com trigger, resultado imutável, arquivos cifrados, prioridade no pedido do 19a, ligação com o 19a e cancelamento por adendo) → Tasks 3, 4, 5, 6, 7, 8.
- §5 fluxo (etapas, autorização, agendamento, coleta, resultado, ciência; prazos na ordem prestador → catálogo → plataforma, urgente = metade; job diário; painel de atrasos; contador de negados por `requester=me&status[]=denied`; cancelamento antes do resultado) → Tasks 7, 9, 10, 11, 13.
- §6 portal (isolamento no servidor; nome, nascimento, 3 dígitos do CPF), FHIR (basedOn, paciente, LOINC, PDF, crítico, idempotência, limite, token revogado), `fake-lab`, botão no PEC, guia da recepção, resultado externo, cidadão (três políticas, nunca motivo nem crítico), relatório, linha do tempo → Tasks 12, 14, 15, 20, 9, 16, 17, 18, 19.
- §7 segurança e LGPD (leitura pelas regras do 19a com trilha; prestador só as guias dele; PDF/guia/valores cifrados; URL curta assinada; eventos só com ids; filtros de log; token só como hash; política no servidor) → Tasks 3, 6, 7, 13, 14, 17, 21.
- §8 testes do api → cada task; prova no navegador com `fake-lab` → Tasks 15, 23.
- §10 entrega: Task 0 (provas técnicas a e b) antes de tudo; contracts → Task 1.
- Rotas e formas pedidas pelo plano do dashboard → Task 19b (apoio) e ajustes nas Tasks 5, 6, 7, 9, 11, 12, 13 (D19–D21).
- §11 F-IDs: F-21.1 T2; F-21.2 T4–T5; F-21.3 T7–T9; F-21.4 T10; F-21.5 T11; F-21.6 T12–T13; F-21.7 T14–T15 (+T1); F-21.8 T13; F-21.9 T16; F-21.10 T17; F-21.11 T20; F-21.12 T18; F-21.13 T19.
- ADR 0036 invariantes → Task 21; `adr_pointers` → Task 2.

**Placeholders:** nenhum "TBD". Os dois SHA-256 da Task 14 dependem do commit da Task 1 e têm o comando exato para obtê-los. Onde o código real pode divergir de um helper (assinatura de `Consultations::AddAddendum.call`, `Consultation#author_user`, mailer do convite, glob do `rails_helper`, `config.hosts` de dev), o step diz o que conferir e que a asserção não muda.

**Consistência de nomes:** `Exams::{Gate,Deadlines,CatalogInput,AdminJson,Files,Open,Transition,Escalate,Access,Json,Trail,Labels,FromConsultation,Cursor,OrderQuery,Steps,GuideInput,ResultValues,Release,CitizenList,Report,DailyJob}`, `Exams::{SaveProvider,RotateFhirToken,InviteProviderUser,RegisterResult,Acknowledge,RecordExternalResult}`, `Exams::Fhir::{Identifiers,Inbound,ServiceRequests}`, `Exams::{BaseController,OrdersController,OrderStepsController,UploadsController,FilesController,RegulationController,ResultsController,AcknowledgementsController,ExternalResultsController,ReportsController}`, `Exams::Admin::{BaseController,CatalogItemsController,ProvidersController}`, `Exams::Provider::OrdersController`, `Fhir::{BaseController,DiagnosticReportsController,ServiceRequestsController}`, `CitizenApi::ExamsController`, `PecExamsController`, `PecExternal::ExamPage`, `Timeline::Sources::ExamSource`, `FakeLab::{Report,Client,Runner}`, `ExamCrew` — conferidos entre as tasks. `Exams::Transition.call(order:, kind:, actor_user:, actor_provider:, reason:, changes:, event_payload:, now:)` é a mesma em todas as chamadas.

**Review Focus:** as cinco linhas têm teste na task dona (14, 13, 17, 13, 11).

## Divergências propostas ao contrato

- **D1 — `exam_tracking` exige `record_mode`, não `feature:clinical_record` (contrato §11; spec §3):** com `clinical_record` (só modo record) e `exam_pec_bridge` exigindo `pec_external_view` (só modo integrado), o botão no PEC nunca fica utilizável e a guia da recepção nunca chega às cidades integradas (decisão 7 da spec). Proposta: `exam_tracking` `requires: ["record_mode"]` (record ou integrated; missing `record_mode_off`). **Confirmar com o coordenador antes da Task 2.** Mantido o contrato: troca-se uma linha, a Task 20 sai e a guia vale só no modo record.
- **D2 — Guia da recepção (`POST /exams/orders`, §3):** aceita `citizen_id` (par validado; o api resolve o paciente por `Patients::Resolve`) **ou** `patient_id`, e exige **`unit_id`** (a unidade do exame; o contrato não a tinha). Par só declarado → 409 `citizen_not_verified`; quem não é da unidade → 403 `not_unit_staff`; solicitante com `user_id` sem `health_professional` → 422 `invalid_requester`; `invalid_cid10`; o exame da consulta não exige catálogo (Desvio 8).
- **D3 — Upload em JSON (`POST /exams/uploads`, §3):** a guarda de escrita por cookie só aceita `application/json`; o corpo é `{ "content_type": "application/pdf"|"image/jpeg"|"image/png", "data": "<base64>" }` → 201 `{ "upload_id" }`; o arquivo vale para quem subiu, uma vez; não usado em 24 h é apagado. O prestador também sobe (laudo do portal).
- **D4 — URLs curtas (§2):** `pdf_url` e o novo `guide_url` são `/exams/files/<token>` (5 min, amarrados ao usuário que leu e à cidade); o download exige a sessão desse usuário (404 para outro). `<exam_order>` ganha `guide_url` (`null` fora da guia).
- **D5 — Duas visões de `<exam_order>` (§2):** clínica (solicitante, leitura do 19a, equipe na fila) com `result` completo; administrativa (unidade, regulador) com `result` reduzido a `{ id, origin, source, released_on, laboratory_name }` e `cid10_justification`/`requester.cns` nulos. A trilha `clinical_record.viewed` sai só na visão clínica, com `access` `author`/`in_context`/`justified`/`team` e `exam_order_id`.
- **D6 — Listas (§3):** `GET /exams/orders` exige um recorte — `consultation_id` (leitura da consulta do 19a: autora ou em contexto), `patient_id` (leitura do 19a; 403 `out_of_context`/`opening_required`/`missing_role`), `requester=me` (os do próprio profissional; é o contador de negados: `&status[]=denied`) ou `unit_id` (unidade), ou `overdue=true` sozinho para `municipal_admin`/`exam_regulator` (atrasos da cidade inteira) — senão 422 `filter_required`; valor de filtro fora da lista → 422 `invalid_filter`; cursor inválido → 422 `invalid_cursor`. `next_before` = Base64 url-safe de `"<created_at>|<id>"` (regulação: `"<rank>|<created_at>|<id>"`).
- **D7 — Perfil FHIR (§7):** sistemas de identificador da RNDS `http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf` e `.../cns` (uniqueId oficial), não `https://saude.gov.br/...`; `code` obrigatório (R4 1..1); `result[]` referencia cada `Observation` contida (dom-3; senão `invalid_resource` com `details: ["/result"]`); `Observation.status` e `.code` obrigatórios; só `Content-Type: application/fhir+json` (415 `unsupported_media_type`); corpo até 15 MB (413 `payload_too_large`); `details` são caminhos de topo (`"/status"`), nunca valores; 422 `invalid_released_on` para `issued` no futuro; `GET /fhir/ServiceRequest` exige `status=active` (422 `invalid_search`) e devolve `intent: "order"`, `code.coding[0].system` `http://www.saude.gov.br/fhir/r4/CodeSystem/BRTabelaSUS`, `identifier.system` `urn:rotasaude:exam-order`. A ordem das recusas: 429 → 403 interruptor → 401 token → 403 `fake_lab` (prestador simulado) → 415/413 → 422/404/409.
- **D8 — Botão no PEC (§8):** sem chamada assinada nova (cada cadastro no PEC tem chave própria, e a cidade guarda uma). A página `/pec/timeline` mostra "Solicitar exames" quando `exam_pec_bridge` é utilizável; `POST /pec/exams/form` (`token` = sessão curta do módulo 20) devolve o formulário; `POST /pec/exams` (`token`, `sigtap_code`, `priority`) cria o pedido `origin: pec`. Sem cookie (iframe de outro site); o token cifrado faz as vezes de CSRF; CSP `form-action 'self'` só nessas páginas. Recusas como as do módulo 20, mais `pec_exam_bridge_disabled` (403) e `pec_session_expired` (401); erros do pedido (`sigtap_not_in_catalog`…) voltam no próprio formulário. Só existe se a Task 0a passar.
- **D9 — Cidadão (§9):** `GET /citizen/exams?citizen_id=` e `GET /citizen/exams/:id/pdf?citizen_id=` (a sessão do wpda cobre vários pares — módulo 20, D2); par de outro telefone → 404; par declarado → `{ verified: false, exams: [], next_before: null }` e 403 `not_verified` no PDF. Cada item ganha `kind: "order"|"external_result"` (o resultado externo entra na lista; o `:id` do PDF aceita os dois); `status_label` "Resultado disponível" / "Resultado registrado — aguardando liberação" / "Retire o resultado na sua unidade"; limite de 30 PDFs por 10 min (429 `too_many_requests`).
- **D10 — Equipe de referência (spec §5):** não há ligação paciente→equipe no código; a fila da equipe é das equipes ativas da **unidade do exame**, para médico (regra de CBO do CID-10) ou enfermeiro (`2235xx`) com vínculo ativo na equipe. `GET /exams/acknowledgements/team` sem equipe → 403 `not_in_team`.
- **D11 — Papéis:** `exam_regulator` e `exam_provider` entram em `Membership::PRIVILEGED_ROLES` (step-up para conceder e convidar; o mantenedor não concede). O convite do prestador é `POST /exams/admin/providers/:id/users { "email" }` → 201 `{ "id", "email", "role", "expires_at" }`; `invitations.exam_provider_id` liga o usuário ao prestador no aceite (um prestador por usuário). O relatório (`/exams/admin/report[.csv]`) vale para `municipal_admin` **e** `exam_regulator`, sem step-up (só leitura, sem dado de paciente); mês inválido → 422 `invalid_month`.
- **D12 — Transições (§1/§3):** `schedule` também de `scheduled` (reagendar); `result` só de `scheduled`/`collected`; prestador do agendamento tem de fazer o exame (item do prestador, ou exame fora do catálogo) e, se o pedido já tem prestador (autorizado), tem de ser o mesmo — senão 422 `provider_not_enabled_for_exam`; `scheduled_at` ISO 8601 e `scheduled_place` 1–200 → senão 422 `invalid_schedule`; motivo de 3–500 caracteres (`reason_too_long`); ciência repetida → 409 `invalid_transition` com `status: "acknowledged"`.
- **D13 — Prestador simulado:** `exam_providers.simulated` (só a semente grava); a API admin ignora o campo; FHIR de prestador simulado exige `fake_lab` utilizável (403 `feature_disabled` com `feature: "fake_lab"`). O `<provider>` do admin traz `simulated`, `has_fhir_token`, `fhir_token_rotated_at`, `items[].sigtap_code` e `users[]`.
- **D14 — `fake-lab` (§13):** variáveis `FAKE_LAB_API_URL` (endereço interno do api; padrão `http://api:3000`), `FAKE_LAB_CITY_HOST` (Host da cidade; padrão `curitiba.localhost`), `FAKE_LAB_TOKEN` (padrão de dev `dev-fake-lab-token-0123456789abcdef0123`) e `FAKE_LAB_INTERVAL` (30). `identifier` do laudo = `FAKE-<short_code>`; outro SIGTAP além de hemograma e HbA1c recebe um `valueString` (LOINC 22637-3).
- **D15 — `fake_lab` "não produção" (§11):** o mecanismo do código é `environments: SIMULATION_ENVS` (development, test, **staging**), como `sncr_mock` e `signature_psc_mock`; não existe `non_production`.
- **D16 — Eventos e trilha (§12):** `<exam_event>.actor.type` ganha `"system"` (escalonamento pelo job, `name: null`); `reason` do evento `escalated` é o código (`overdue|critical|unlinked_requester`), e o motivo livre só sai na visão clínica. `clinical_record.viewed` do cidadão leva `exam_result_id` e `person_ref`; o da equipe, `access: "team"`.
- **D17 — Resultado (§3):** corpo com `upload_id` de PDF (guia pode ser imagem, laudo não: 422 `missing_pdf`); `values` só com chaves dos campos do catálogo (422 `invalid_values`; exame fora do catálogo aceita só `[]`); `released_on` não pode ser futuro (422 `invalid_released_on`); `laboratory_name` 1–160 (`invalid_laboratory`); `critical` booleano (`invalid_critical`). Resultado crítico **ou** de solicitante digitado já nasce escalado (`critical`/`unlinked_requester`). Resultado externo aceita `citizen_id` (par validado) além de `patient_id`, e responde `<exam_result>` na visão de quem registra; `source` é `consultation` com `consultation_id` (só da própria autora; senão 404), senão `unit`.
- **D18 — Migração e catálogo de exames de dev:** migração de cidade `20261013900001` (nenhuma de plataforma); o recorte SIGTAP de dev (`lib/sigtap_sample.rb`) ganha hemograma `0202020380`, HbA1c `0202010503`, anti-HIV `0202030300` e anatomopatológico `0203020030` (a semente republica a competência corrente se faltarem).
- **D19 — Rotas de apoio (pedido do dashboard D2–D5, D8; Task 19b):** `POST /exams/patient_lookup { cpf }` → `{ patient: { id, display_name, cpf_masked } }` (404 `patient_not_found`, 409 `citizen_not_verified`, 403 `missing_role`, 429; evento `exam.patient_looked_up { patient_id, user_id }`); `GET /exams/catalog` → `{ items: [ { sigtap_code, label, result_fields } ] }`; `GET /exams/providers?sigtap_code=` → `{ providers: [ { id, name } ] }`; `GET /exams/requesters?unit_id=` → `{ requesters: [ { user_id, name } ] }` (403 `not_unit_staff`); `GET /exams/external_results?patient_id=` → `{ results: [ <exam_result> + sigtap_code, label ] }`.
- **D20 — Formas pedidas pelo dashboard (D6, D7, D9, D11):** `<exam_order>` ganha `consultation_id` (sempre) e `result_fields` (só no detalhe, com `events`); `<provider_order>` ganha `result_fields`; a lista do portal devolve `overdue_count` no corpo além do cabeçalho `X-Overdue-Count`; `GET /exams/acknowledgements/pending` → `{ orders, denied, next_before }` (`denied` = negados do solicitante nos últimos 30 dias, até 50); `GET /exams/acknowledgements/team` → `{ orders, next_before }`; `<provider>` do admin usa `has_fhir_token` (não `fhir_token_active`); o convite responde `{ id, email, role, expires_at }`; `pdf_url`/`guide_url` vencido → 410 `link_expired`. O portal agenda sem `provider_id` (é o do usuário).
- **D21 — Exame pela abertura justificada (dashboard D13):** `GET /exams/orders/:id` já aceita, sem parâmetro, a abertura justificada válida do usuário para o paciente (é a regra do 19a em `ClinicalRecord::Access.call`); o dashboard pode seguir abrindo só em contexto — nada muda no api.
- **Nota sobre a D12 do dashboard:** a prova técnica do botão no PEC **ainda não foi feita** (é a Task 0a deste plano); a Task 20 e o contrato §8 seguem condicionais ao resultado dela.
