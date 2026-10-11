# Módulo 20 — Linha do tempo do paciente (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulos até o 19d em `origin/main` do api** (conferido ao escrever este plano em `874efd7`: `ClinicalRecord::{Access,Gate,Trail,Open,Json}`, `ClinicalRecordOpening.valid_for`, `ClinicalDocuments::{Gate,Pdf,Page,Verification,Uuid}`, `ClinicalTerms.label`, `ClinicalTerms::SigtapExams.label`, `Screenings::Ciap2.label`, `Scheduling::AppointmentTypes.catalog`, `Consultations::Authorization.link_for`, `Platform::Features::CATALOG` com 8 chaves, a migração de cidade `20261011800001_add_consultation_draft_actions`). **A Task 0 confere a base e a documentação oficial dos "sistemas externos" do PEC: se um nome mudou, ou se a documentação não estiver acessível ou disser outra coisa, PARE e reporte.**

**Goal:** O lado api do módulo 20 (F-20.1 a F-20.6, ADR 0035): interruptores `patient_timeline` e `pec_external_view`; configuração do PEC por cidade (`pec_origin` e chave HMAC cifrada, só pelo maintenance); "a pessoa" nos modos `record` e `integrated`; `ClinicalRecord::Access` generalizado para a pessoa sem mudar o 19a; os 8 fornecedores de entradas com os índices que faltam; junção e paginação por cursor composto; a linha do tempo do profissional (`/attendance/timeline`), o histórico, o resumo clínico e o PDF do cidadão (`/citizen/*`); a página para a tela de "sistemas externos" do e-SUS PEC (`/pec/timeline`); o GraphQL do maintenance; eventos, trilha, invariantes; semente de dev com uma cidade integrada; página de teste que simula o PEC (só em dev); prova e rollout.

**Architecture:** Montagem na leitura (ADR 0035): cada origem tem um fornecedor `Timeline::Sources::*Source.entries(person:, before:, limit:, from:, to:)` que devolve até `limit` objetos `Timeline::Entry` estritamente depois do cursor, na ordem `occurred_at DESC, kind ASC, id ASC`; `Timeline::Query` junta, ordena pela mesma chave e corta. A pessoa (`Timeline::Person`) é o conjunto dos pares **validados** do CPF (mais o paciente do 19a, quando existe). A leitura do profissional passa por `ClinicalRecord::Access.for_person` (novo, ao lado do `call` do 19a, que fica intocado); a abertura justificada passa a valer para uma pessoa sem paciente (`clinical_record_openings.citizen_id`). A página do PEC confere, nesta ordem, interruptor, HMAC-SHA256 da URL inteira até antes de `&pec_sign=` (comparação em tempo constante), `pec_timestamp` a até 5 min, uso único (tabela `pec_signature_uses`), unidade do CNES, vínculo do profissional e a pessoa; abre uma sessão curta (15 min) num token cifrado que o "carregar mais" manda em cabeçalho, e responde HTML do servidor com `Content-Security-Policy: frame-ancestors <pec_origin>`. Banco da cidade: migração `20261011800002` (índices, `clinical_record_openings.citizen_id`, `pec_settings`, `pec_signature_uses`). Plataforma: nenhuma migração (o catálogo de interruptores mora no código).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, `OpenSSL::HMAC`, `ActiveSupport::SecurityUtils.secure_compare`, `ActiveSupport::MessageEncryptor`, graphql-ruby 2.6 (maintenance). Nenhuma gem nova, nenhum framework de front novo (o HTML é string escapada no servidor, como `ClinicalDocuments::Page`).

**Spec:** `docs/superpowers/specs/2026-10-11-module-20-patient-timeline-design.md` e `docs/adr/0035.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-11-module-20-timeline-contracts.md`. Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` e a documentação oficial dos "sistemas externos" (`https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/{index,iframe,chave_assinatura,parametros_dinamicos}.html`), lida ao escrever este plano (Desvio 1). Contratos do 19a (§3, §9) prevalecem sobre os planos antigos.

## Desvios e precisões

1. **O que a documentação oficial diz (lida em 2026-10-11; "Alterado em" 09/02/2026 na chave, 19/06/2026 nos parâmetros, 11/09/2026 no índice, 16/09/2025 no iframe):**
   - O PEC substitui variáveis numa URL que **o integrador** entrega ao administrador da instalação. O formato previsto é `https://<sistema>?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]` — as chaves da query são **`documento-profissional`, `documento-cidadao` e `cnes`** (minúsculas, com hífen), não `documentoCidadao`/`documentoProfissional` como estão no contrato §5 (Divergência D1).
   - `documentoProfissional` e `documentoCidadao` são o CPF ou, sem CPF, o CNS; `documentoCidadao` só existe quando o "Local de acesso" do sistema externo é **"Atendimento"** (no menu principal ele não vem).
   - O PEC acrescenta **`pec_timestamp`** (segundos Unix) e **`pec_sign`** no fim. `pec_sign` é o **HMAC-SHA256 em hexadecimal** (minúsculo, `digest("hex")` no exemplo oficial) com a chave do integrador, calculado sobre **a URL completa — protocolo, host, caminho e query — até o `pec_timestamp` inclusive**, isto é, tudo o que vem antes de `&pec_sign=`. O exemplo oficial monta a URL com `req.protocol://req.get("host") + req.originalUrl` e corta em `&pec_sign=`; avisa que "a URL assinada no PEC é canônica" (codificar se houver caractere especial).
   - A chave é gerada pelo PEC ao cadastrar o integrador (Integração > Sistemas externos, só com HTTPS na instalação), entregue pelo administrador da instalação e **não pode ser recuperada** (perdida → novo cadastro). A documentação não fixa formato nem tamanho: tratamos como texto opaco (16–512 caracteres visíveis).
   - A janela do exemplo oficial é 30 min em valor absoluto; o ADR fixa **5 min** (`|agora − pec_timestamp| ≤ 300 s`).
   - O iframe só carrega se a resposta não tiver `X-Frame-Options` restritivo nem `frame-ancestors 'none'`.
2. **URL conferida:** `request.base_url + request.original_fullpath`, cortada no primeiro `&pec_sign=`. Atrás do proxy de produção, `base_url` depende de `Host` e `X-Forwarded-Proto` repassados — **gate de go-live** (Task 17, Step 7), provado em staging.
3. **Documento do cidadão por CNS não acha ninguém:** `citizens.cns` é cifrado **não determinístico** (ADR 0028), sem busca. Só CPF identifica a pessoa; CNS (15 dígitos) cai em `pec_citizen_not_verified`. O do profissional acha por CPF ou CNS (`professionals.cpf`/`cns` são determinísticos).
4. **Sessão curta sem cookie (Divergência D3):** o "carregar mais" leva um token cifrado (`ActiveSupport::MessageEncryptor`, 15 min, propósito `pec_view_session`, amarrado à cidade, ao usuário, à unidade e à pessoa) no cabeçalho `X-Pec-View-Session`. Motivo: o PEC é outro site; Safari e Firefox bloqueiam cookie de terceiro dentro de iframe (e o Chrome restringe), então um cookie `SameSite=None` não chegaria ao "carregar mais". O token mora no DOM da página do iframe (origem isolada da do PEC). A cada "mais" o interruptor e o vínculo são relidos.
5. **A pessoa:** `citizen_ids` = pares `verified` e não excluídos com o CPF (e os ligados ao paciente); `patient_id` = o paciente do 19a quando existe (com qualquer modo: uma cidade que saiu de `record` mantém as consultas). Par só declarado nunca entra; o cidadão de par declarado vê só o próprio par (`verified: false`).
6. **Abertura justificada sem paciente (Divergência D5):** `clinical_record_openings` ganha `citizen_id` (um par validado do CPF; exatamente um entre `patient_id` e `citizen_id`), em vez de gravar o CPF — a exclusão LGPD troca o CPF do par por marcador e a abertura não guarda CPF nenhum. `POST /clinical_record/openings` passa a valer com `patient_timeline` ligado mesmo sem prontuário utilizável, e cria a abertura pelo par quando o CPF não tem paciente. Com `patient_timeline` desligado, nada muda (regressão do 19a).
7. **`ClinicalRecord::Access.call` e `.for_consultation` ficam byte a byte iguais.** A generalização é o método novo `for_person`, que reusa `in_context?` e acrescenta a abertura pelo par. A leitura **administrativa** (municipal_admin) não passa pela linha do tempo: a rota exige `health_professional` (403 `missing_role`); o detalhe de cada entrada abre as telas existentes, com as regras delas (`author`, `administrative`).
8. **Rotas do cidadão levam `citizen_id` (Divergência D2):** a sessão do wpda é por **telefone** e cobre vários pares (a família no mesmo celular), como `GET /citizen/triages?citizen_id=` e `GET /citizen/appointments?citizen_id=` já fazem.
9. **Migração:** `20261011800001` já existe (`add_consultation_draft_actions`); a do módulo 20 é **`20261011800002`**. Sem migração de plataforma.
10. **Campanha — "participação"** = leitura do aviso no wpda (`campaign_recipients.notice_read_at`): não existe outro registro de participação.
11. **Sinais vitais do resumo do cidadão:** a aferição mais recente entre consultas finalizadas (19a) e escutas concluídas (18, revisão vigente).
12. **Exames solicitados:** por consulta e código SIGTAP vale a linha mais recente (o adendo cancela com uma linha `cancelled`); entram as `requested` dos últimos 12 meses.
13. **Agendamento:** entra todo horário do par, menos o `moved` (substituído pelo novo, que já aparece); horários futuros aparecem no topo (ordem decrescente).
14. **Interruptor `pec_external_view` sem configuração:** os `requires` ganham `pec_origin` e `pec_hmac_key` (pedido M3 do coordenador): sem um dos dois o interruptor não é utilizável (missing `pec_origin_missing`/`pec_hmac_key_missing`) e a página responde `pec_disabled`. `Platform::Features.city_state` passa a ler `pec_settings` (guardado por `data_source_exists?`, para uma cidade ainda não migrada não virar `city_unreachable` em todos os interruptores).
15. **Trocar a chave HMAC exige step-up** (decisão do coordenador, M4): `rotateCityPecHmacKey(citySlug, key, code)` com o TOTP do momento, pelo caminho da base (`step_up_code:`).
16. **Sessão do cidadão diz os interruptores:** `GET`/`POST /citizen/session` devolvem também `features: string[]` com os interruptores **utilizáveis** da cidade (pedido W2 do wpda).

## Valores fixados por este plano

1. **Interruptores:** `patient_timeline` (`requires: []`), `pec_external_view` (`requires: ["feature:patient_timeline", "record_mode:integrated", "pec_origin", "pec_hmac_key"]`, missing `patient_timeline_disabled`, `record_mode_not_integrated`, `pec_origin_missing` e `pec_hmac_key_missing` — os dois últimos lidos de `pec_settings` no banco da cidade, pedido do coordenador M3).
1b. **`status` por kind (códigos; as telas traduzem):** `triage` `completed` | `aborted_by_revocation` | `aborted_by_timeout` | `aborted_by_cancellation`; `attendance` `waiting` | `in_care` | `closed`; `screening` `completed`; `appointment` `scheduled` | `confirmed` | `checked_in` | `cancelled_by_citizen` | `expired` | `no_show`; `consultation` `finalized`; `clinical_document` e `controlled_notification` `issued` | `cancelled`; `campaign` `read` | `unread`. Na visão do cidadão, os mesmos códigos, menos `screening` e `consultation` (`null`).
1c. **`detail_ref` por kind:** `triage` → `{ type: "triage", id }`; `attendance` → `attendance`; `screening` → `screening`; `appointment` → `{ type: "appointment_request", id: <pedido> }` (a tela existente lê o pedido); `consultation` → `consultation`; `clinical_document` e `controlled_notification` → `clinical_document`; `campaign` → `{ type: "campaign", id: <campanha> }`. Fora de contexto (abertura justificada) o detalhe só abre a consulta (`/clinical_record/consultations/:id` do 19a); as telas de documento, escuta e pedido seguem com as regras delas (em contexto), e a própria entrada traz o resumo (Divergência D15).
2. **Kinds, nesta grafia:** `triage`, `attendance`, `screening`, `appointment`, `consultation`, `clinical_document`, `controlled_notification`, `campaign`. Ordem total: `occurred_at` decrescente, depois `kind` crescente, depois o id da origem crescente.
3. **`occurred_at` por kind:** `triages.created_at` (sem `in_progress`), `attendances.checked_in_at`, `screenings.completed_at` (só `completed`), `appointments.scheduled_at` (sem `moved`), `consultations.finalized_at` (só `finalized`), `clinical_documents.issued_at` (`kind <> 'controlled_notification_record'`), o mesmo para `controlled_notification` (`kind = 'controlled_notification_record'`), `campaign_recipients.created_at`.
4. **Cursor:** `Base64.urlsafe_encode64("<occurred_at UTC iso8601(6)>|<kind>|<uuid>", padding: false)`; inválido → 422 `invalid_cursor`. `limit` padrão 50, máximo 100 (acima → 100; não inteiro ou < 1 → 50).
5. **Cidadão:** par validado vê os 8 kinds (os 2 de documento só com `clinical_documents` utilizável); par declarado vê `triage` e `appointment` do próprio par.
6. **PEC:** parâmetros `documento-profissional`, `documento-cidadao`, `cnes` (7 dígitos), `pec_timestamp` (dígitos), `pec_sign` (64 hex; comparado em minúsculas); janela 300 s; uso único guardado até `pec_timestamp + 300 s`; sessão 15 min; cabeçalho `X-Pec-View-Session`; códigos `pec_disabled`, `pec_signature_invalid`, `pec_expired`, `pec_replayed`, `pec_unit_unknown`, `pec_professional_not_linked`, `pec_citizen_not_verified`, e os novos `pec_citizen_missing` e `pec_session_expired` (Divergências D6, D7). Falha → HTML genérico com o código, status 403.
7. **`pec_origin`:** `scheme://host[:port]` sem caminho, query, fragmento ou usuário; `https` sempre, e `http` só fora de `Rota.deployed?`. **Chave HMAC:** texto sem espaço nas pontas, 16–512 caracteres, só ASCII visível (`/\A[\x21-\x7E]{16,512}\z/`).
8. **Dev:** api do worktree na porta **3040**; simulador do PEC em `http://pec-sim.localhost:3040/pec-simulator`; Maringá em modo `integrated` com `pec_origin = http://pec-sim.localhost:3040` e chave de dev `dev-pec-hmac-key-maringa-0123456789` (override por `DEV_PEC_HMAC_KEY`).

## Global Constraints

- Banco de cidade: `db/city_migrate/20261011800002_add_patient_timeline.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_11_800002)`; paridade em `spec/services/city_schema_spec.rb`). Nenhum trigger novo (o `clinical_record_openings_reencryption_only` continua cobrindo a tabela). Irreversível. Rollout: publicar a imagem e rodar `city:migrate:all` dela **antes** de cortar tráfego; **nunca** migrar cidade fora do rake.
- Valores exatamente como em "Valores fixados" e no contrato: erros `{ "error": "<reason>" }`; 403 `{ "error": "feature_disabled", "feature": "patient_timeline" }`; 403 `out_of_context`, `opening_required`, `missing_role`, `not_verified`; 404 `not_found`, `not_available`; 422 `invalid_cursor`, `invalid_period`; novos (Divergências): 409 `citizen_not_verified` e 422 `invalid_kinds` na rota do profissional.
- Texto clínico livre (SOAP, adendo, conteúdo de documento além do tipo), CPF, CNS, nome, número SNCR e a chave HMAC **nunca** em log, evento, Analytics, mensagem de erro ou `inspect`. Os parâmetros `documento-cidadao`, `documento-profissional` e `pec_sign` entram em `filter_parameters` (a linha "Started GET" usa `request.filtered_path`).
- Eventos só com ids, declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`: `clinical_record.viewed` ganha `access` `timeline`/`citizen`/`pec_external` e `person_ref`; novos `pec.external_view_opened { user_id, unit_id, patient_id }` e `pec.external_view_denied { reason_code }`. Plataforma: `city.pec_config_changed { city_id, field, maintainer_id }` e `maintenance.city.pec_config_changed`, ambos em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`) e o segundo em `MaintenanceAudit::NAMES` + branch do `case`. `DomainEvents.publish(` e `Platform.audit(` são multilinha: confira cada chamada inteira.
- Cifra por cidade: `encrypts :hmac_key` (não determinístico) entra em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (só lido); nenhum job novo.
- Specs de request com `type: :request` explícito; arquivo novo em `spec/support/` entra com `require_relative` em `spec/rails_helper.rb` se o arquivo não for carregado por glob (confira como `clinical_record_helpers.rb` entra e faça igual). `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..35)`.
- Escrita por cookie exige `Content-Type: application/json` (guarda existente); as rotas novas são todas `GET`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (**nunca** `-A`). Merge, push, tag e cards só com autorização explícita do usuário.

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard, do wpda e do maintenance do módulo 20. Ordem de merge: **api** → dashboard e wpda → maintenance.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod20 -b feat/mod-20-timeline origin/main
  cp apps/api/config/master.key apps/api/.claude/mod20/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod20`. **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod20 -w /rails/.claude/mod20 api bundle exec rspec <arquivos>
  ```

  e `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod20 <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod20`), criados no começo e recriados depois da migração (Task 2):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod20" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod20"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod20 -w /rails/.claude/mod20 api bin/rails city:test_databases
  ```

- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.

### Arquivos em comum com outros módulos (rebase)

Se a main andar, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` (`define(version:)` com o maior número; se outra branch já usou `20261011800002`, renumere a migração deste módulo para o próximo livre e ajuste o `define`), `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`, `app/services/city_encryption.rb`, `spec/adr_pointers_spec.rb` (fica `1..35`), `config/routes.rb`, `app/services/platform/features.rb`, as três specs com a lista fixa do catálogo, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `app/graphql/maintenance/**`, `spec/architecture/maintenance_schema_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`, `db/seeds.rb`.

## Review Focus

1. **Histórico longo com empates de horário** (vários eventos no mesmo microssegundo, de kinds diferentes e do mesmo kind; o profissional percorre todas as páginas com e sem filtro de kind e de período): nenhuma entrada repetida nem perdida, `next_before` nulo só na última página. Teste: Task 7 ("percorre tudo com empates").
2. **A URL do PEC chega diferente do que foi assinado** (parâmetros em outra ordem, um parâmetro a mais depois de `pec_sign`, `pec_sign` em maiúsculas, assinatura com 63 caracteres, timestamp do futuro dentro e fora da janela): só a URL idêntica passa; maiúsculas passam; o resto cai em `pec_signature_invalid` ou `pec_expired` sem dizer qual parâmetro. Teste: Task 12 ("URL mexida").
3. **Família no mesmo celular** (a sessão do wpda tem um par validado e um declarado; alguém troca o `citizen_id` por um de outro telefone ou pede o PDF do documento do irmão): 404 `not_found` em tudo que não é da própria pessoa; o par declarado vê só triagens e agendamentos dele. Teste: Task 9 ("família no mesmo celular").
4. **A pessoa muda no meio do caminho** (um par validado e outro declarado com o mesmo CPF; a validação é revogada; o cadastro é excluído pela LGPD): o declarado e o revogado nunca entram; o excluído some da pessoa. Teste: Task 3 ("pares que entram e saem").
5. **O profissional perde o vínculo, ou a cidade desliga o interruptor, durante a sessão curta do PEC:** o "carregar mais" recusa (`pec_session_expired`), nada vaza, a primeira página já aberta continua só como estava. Teste: Task 14 ("sessão curta que cai").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-11-pec-sistemas-externos.md` (repo docs) | registro da leitura da documentação oficial | 0 |
| `app/services/platform/features.rb`, `app/services/timeline/gate.rb`, `spec/adr_pointers_spec.rb`, 3 specs com a lista do catálogo, `spec/services/timeline/gate_spec.rb` | interruptores | 1 |
| `db/city_migrate/20261011800002_add_patient_timeline.rb`, `db/city_schema.rb`, `app/models/{pec_setting,pec_signature_use}.rb`, `app/models/clinical_record_opening.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/support/timeline_helpers.rb`, `spec/models/pec_setting_spec.rb` | dados da cidade | 2 |
| `app/services/timeline/{person,entry,cursor}.rb`, `spec/services/timeline/{person,cursor}_spec.rb` | a pessoa, a entrada e o cursor | 3 |
| `app/services/clinical_record/access.rb`, `app/commands/clinical_record/open.rb`, `app/controllers/clinical_record_openings_controller.rb`, `spec/services/clinical_record/access_for_person_spec.rb`, `spec/commands/clinical_record/open_spec.rb`, `spec/requests/clinical_record_openings_timeline_spec.rb` | leitura e abertura pela pessoa | 4 |
| `app/services/timeline/sources/{window,labels,triage_source,attendance_source,screening_source,appointment_source}.rb`, `spec/services/timeline/sources/administrative_sources_spec.rb` | fornecedores administrativos | 5 |
| `app/services/timeline/sources/{consultation_source,clinical_document_source,controlled_notification_source,campaign_source}.rb`, `spec/services/timeline/sources/clinical_sources_spec.rb` | fornecedores clínicos e campanha | 6 |
| `app/services/timeline/{query,json,trail}.rb`, `spec/services/timeline/query_spec.rb` | junção, paginação, formas e trilha | 7 |
| `app/controllers/timelines_controller.rb`, `config/routes.rb`, `spec/requests/timeline_spec.rb` | linha do tempo do profissional | 8 |
| `app/controllers/citizen_api/history_controller.rb`, `app/services/timeline/citizen_documents.rb`, `config/routes.rb`, `spec/requests/citizen_api/history_spec.rb` | histórico e PDF do cidadão | 9 |
| `app/services/timeline/citizen_summary.rb`, `history_controller.rb`, `spec/services/timeline/citizen_summary_spec.rb`, `spec/requests/citizen_api/health_summary_spec.rb` | resumo clínico do cidadão | 10 |
| `app/commands/pec_external/configure.rb`, `app/graphql/maintenance/{mutations/set_city_pec_origin,mutations/rotate_city_pec_hmac_key,types/city_pec_config_type,types/query_type,types/mutation_type,mutations/city_mutation,analyzers/human_only}.rb`, `app/events/maintenance_audit.rb`, specs do maintenance | configuração do PEC no maintenance | 11 |
| `app/services/pec_external/signature.rb`, `spec/services/pec_external/signature_spec.rb` | HMAC, janela e uso único | 12 |
| `app/services/pec_external/{authorize,view_session}.rb`, `spec/services/pec_external/authorize_spec.rb` | ordem de conferência e sessão curta | 13 |
| `app/services/pec_external/page.rb`, `app/controllers/pec_timelines_controller.rb`, `config/routes.rb`, `spec/requests/pec_timeline_spec.rb` | página do PEC | 14 |
| `spec/invariants/timeline_invariants_spec.rb` | invariantes do ADR 0035 | 15 |
| `lib/timeline_crew.rb`, `lib/pec_simulator.rb`, `app/controllers/dev/pec_simulators_controller.rb`, `config/routes.rb`, `db/seeds.rb`, `spec/lib/timeline_crew_spec.rb`, `spec/lib/pec_simulator_spec.rb`, `spec/architecture/pec_simulator_route_spec.rb` | semente e simulador de dev | 16 |
| — | revisão, suíte, porta 3040, prova, rollout | 17 |

---

## Fatia 0 — Base e documentação do PEC

### Task 0: Conferir a base e registrar a documentação oficial dos "sistemas externos"

Esta task não escreve código do api. **Se algum nome do Step 1 não existir, se a documentação do Step 2 não abrir, ou se ela disser algo diferente do Desvio 1 (nomes dos parâmetros, `pec_sign`, `pec_timestamp` em segundos, HMAC-SHA256 hex sobre a URL até antes de `&pec_sign=`), PARE e reporte** ao coordenador e ao usuário: os nomes e o algoritmo entram em código e em teste nas Tasks 12–16.

**Files:**
- Create: `docs/pesquisa/2026-10-11-pec-sistemas-externos.md` (repo `docs`)

- [ ] **Step 1: Confira os nomes da base** (todos devem casar; anote o hash de `origin/main`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api log -1 --format=%h origin/main
  for pattern in "def for_consultation" "def open_attendances_of" "def in_context?" ; do /opt/homebrew/bin/git -C apps/api grep -n "$pattern" origin/main -- app/services/clinical_record/access.rb; done
  /opt/homebrew/bin/git -C apps/api grep -n "scope :valid_for" origin/main -- app/models/clinical_record_opening.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def usable?" origin/main -- app/services/clinical_documents/gate.rb app/services/clinical_record/gate.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def label" origin/main -- app/services/clinical_terms.rb app/services/clinical_terms/sigtap_exams.rb app/services/screenings/ciap2.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def staff_name" origin/main -- app/services/screenings/json.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def name_for\|def catalog" origin/main -- app/services/scheduling/appointment_types.rb
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/city_migrate/ | tail -2
  /opt/homebrew/bin/git -C apps/api grep -n "VALID_RANGE" origin/main -- spec/adr_pointers_spec.rb
  ls docs/adr/0035.md
  ```
  Expected: cada grep acha a linha; a última migração de cidade é `20261011800001_add_consultation_draft_actions.rb` (se houver outra `2026101180000x`, use o próximo número livre e ajuste o plano inteiro); `VALID_RANGE = (1..34)`; o ADR 0035 existe.

- [ ] **Step 2: Releia a documentação oficial** (navegador ou `curl -sL`): `https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/index.html`, `.../iframe.html`, `.../chave_assinatura.html`, `.../parametros_dinamicos.html`. Confira cada item do Desvio 1 e a data "Alterado em" de cada página. Se uma data mudou, compare o texto com o Desvio 1 antes de seguir.

- [ ] **Step 3: Registre a leitura** em `docs/pesquisa/2026-10-11-pec-sistemas-externos.md` (com as palavras do registro, sem copiar o texto oficial):

  ```markdown
  # e-SUS PEC — "Sistemas externos": parâmetros e assinatura (lido em 2026-10-11)

  Fonte: https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/ (páginas index, iframe,
  chave_assinatura e parametros_dinamicos; "Alterado em" 11/09/2026, 16/09/2025, 09/02/2026 e 19/06/2026).

  ## Cadastro
  - Feito pelo Administrador da Instalação em Integração > Sistemas externos, só em instalação com HTTPS.
  - Dados: URL de acesso, nome, CPF ou CNPJ, e-mail e nome do sistema; "Local de acesso": Atendimento
    (aba no atendimento individual e odontológico) ou menu principal.
  - Ao cadastrar, o PEC gera a chave de assinatura; o administrador a repassa ao integrador. Chave perdida
    não se recupera: novo cadastro.

  ## Parâmetros na URL
  - URL entregue ao administrador: `https://<host>/pec/timeline?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]`.
  - `documento-profissional` e `documento-cidadao`: CPF, ou CNS quando não há CPF. `documento-cidadao` só
    com "Local de acesso" = Atendimento.
  - `cnes`: CNES do estabelecimento da lotação do profissional.
  - O PEC acrescenta `&pec_timestamp=<segundos Unix>&pec_sign=<assinatura>`.

  ## Assinatura
  - HMAC-SHA256 com a chave do cadastro, saída hexadecimal minúscula.
  - Mensagem assinada: a URL completa (protocolo, host, caminho e query) até `pec_timestamp` inclusive —
    tudo antes de `&pec_sign=`. A URL assinada é canônica.
  - Janela do exemplo oficial: 30 min em valor absoluto. O Rota Saúde usa 5 min (ADR 0035), uso único.

  ## iframe
  - Não carrega com `X-Frame-Options` DENY/SAMEORIGIN nem com `frame-ancestors 'none'`.

  ## Consequências no Rota Saúde
  - Chaves da query com hífen (contrato do módulo 20, Divergência D1).
  - Documento do cidadão por CNS não identifica ninguém (CNS do par é cifrado não determinístico).
  - URL conferida = `request.base_url + request.original_fullpath`, cortada em `&pec_sign=`: o proxy
    de produção precisa repassar Host e X-Forwarded-Proto (gate de go-live).
  ```

- [ ] **Step 4: Commit no repo docs** (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add pesquisa/2026-10-11-pec-sistemas-externos.md
  /opt/homebrew/bin/git -C docs commit -m "docs: record PEC external systems parameters and signature

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 5: Crie o worktree e os bancos de teste** (seção "Ambiente de execução") e rode `rspec spec/adr_pointers_spec.rb spec/requests/clinical_record_spec.rb` no worktree limpo. Expected: verde (a base passa antes de qualquer mudança).

---

## Fatia 1 — Interruptores e dados

### Task 1: Interruptores `patient_timeline` e `pec_external_view`

**Files:**
- Modify: `app/services/platform/features.rb` (`CATALOG`, `city_state`, `missing_for`)
- Create: `app/services/timeline/gate.rb`
- Modify: `spec/adr_pointers_spec.rb:13`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30`
- Test: `spec/services/timeline/gate_spec.rb`

**Interfaces:**
- Produces: `Timeline::Gate::KEY = "patient_timeline"`, `Timeline::Gate::PEC_KEY = "pec_external_view"`, `Timeline::Gate.enabled?(city) -> Boolean`, `Timeline::Gate.pec_usable?(city) -> Boolean`, `Timeline::Gate.partial?(city) -> Boolean` (true quando `record_mode != "record"`).

- [ ] **Step 1: Write the failing test** `spec/services/timeline/gate_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035 (spec §3; contrato §6): patient_timeline sem pré-requisito;
  # pec_external_view exige patient_timeline e o modo integrado.
  RSpec.describe Timeline::Gate do
    def set(city, key, enabled)
      Platform::Features.set!(city: city, key: key, enabled: enabled, maintainer: ledi_maintainer!)
    end

    it "patient_timeline liga sozinho, em qualquer modo" do
      city = clinical_city!(record_mode: "off", enabled: false)
      expect(described_class.enabled?(city)).to be(false)
      set(city, "patient_timeline", true)
      expect(Platform::Features.missing(city, "patient_timeline")).to eq([])
      expect(described_class.enabled?(city)).to be(true)
      expect(described_class.enabled?(nil)).to be(false)
    end

    it "pec_external_view exige patient_timeline, record_mode integrated e a configuração do PEC" do
      city = clinical_city!(record_mode: "record", enabled: false)
      set(city, "pec_external_view", true)
      expect(Platform::Features.missing(city, "pec_external_view"))
        .to eq(%w[patient_timeline_disabled record_mode_not_integrated pec_origin_missing pec_hmac_key_missing])
      expect(described_class.pec_usable?(city)).to be(false)

      set(city, "patient_timeline", true)
      city.update!(record_mode: "integrated")
      # pec_settings nasce na Task 2: sem a tabela, a configuração conta como ausente.
      expect(Platform::Features.missing(city, "pec_external_view")).to eq(%w[pec_origin_missing pec_hmac_key_missing])
      expect(described_class.pec_usable?(city)).to be(false)
      expect(described_class.pec_usable?(nil)).to be(false)
    end

    it "partial? é verdadeiro fora do modo record" do
      expect(described_class.partial?(clinical_city!(record_mode: "integrated", enabled: false))).to be(true)
      expect(described_class.partial?(clinical_city!(record_mode: "record"))).to be(false)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/timeline/gate_spec.rb`. Expected: FAIL com `uninitialized constant Timeline`.

- [ ] **Step 3: Implemente.** Em `app/services/platform/features.rb`, depois da entrada `sncr_mock` em `CATALOG`:

  ```ruby
      # ADR 0035: linha do tempo do paciente (módulo 20), para o profissional
      # (dashboard) e para o cidadão (wpda). Sem pré-requisito: no modo
      # integrado mostra o que passou pelo Rota Saúde (histórico parcial).
      Entry.new(key: "patient_timeline",
                description: "Linha do tempo do paciente (profissional no dashboard e cidadão no wpda)",
                requires: []),
      # ADR 0035: página para a tela de "sistemas externos" do e-SUS PEC, só no
      # modo integrado. Sem pec_origin e chave HMAC (pec_settings), a página
      # responde pec_disabled.
      Entry.new(key: "pec_external_view",
                description: "Página da linha do tempo para a tela de sistemas externos do e-SUS PEC",
                requires: %w[feature:patient_timeline record_mode:integrated pec_origin pec_hmac_key])
  ```

  em `city_state`, o hash lido dentro de `CityConnection.with(city)` ganha a configuração do PEC (guardada: antes da migração `20261011800002` a tabela não existe, e uma exceção ali faria a cidade inteira aparecer `city_unreachable`):

  ```ruby
          pec = PecConfigState.read
          { credentials: IntegrationCredential.pluck(:kind, :last_check_status).to_h,
            ibge_code: CityProfile.current&.ibge_code, pec_origin: pec[:origin], pec_hmac_key: pec[:key] }
  ```

  com, no fim do módulo `Platform::Features` (antes do último `end`):

  ```ruby
      # ADR 0035: só se há origem e chave do PEC — nunca o valor da chave.
      module PecConfigState
        module_function

        def read
          return { origin: false, key: false } unless ApplicationRecord.connection.data_source_exists?("pec_settings")

          row = ApplicationRecord.connection.select_one("SELECT pec_origin IS NOT NULL AS origin, hmac_key IS NOT NULL AS key FROM pec_settings LIMIT 1")
          cast = ActiveModel::Type::Boolean.new
          { origin: cast.cast(row&.fetch("origin")) == true, key: cast.cast(row&.fetch("key")) == true }
        end
      end
  ```

  e em `missing_for`, depois de `when "record_mode:record" ...`:

  ```ruby
      when "record_mode:integrated" then "record_mode_not_integrated" unless platform[:record_mode] == "integrated"
      when "pec_origin" then "pec_origin_missing" unless state[:pec_origin]
      when "pec_hmac_key" then "pec_hmac_key_missing" unless state[:pec_hmac_key]
  ```

  (A leitura é SQL cru de propósito: na Task 1 o modelo `PecSetting` ainda não existe, e o valor cifrado nunca é carregado.)

  Crie `app/services/timeline/gate.rb`:

  ```ruby
  # Linha do tempo do paciente (ADR 0035): patient_timeline LIGADO (sem
  # pré-requisito) e, para a página do PEC, pec_external_view LIGADO E
  # UTILIZÁVEL (patient_timeline + modo integrado). Relido da plataforma a cada
  # chamada; record_mode relido por id (Platform::Features.settings).
  module Timeline
    module Gate
      KEY = "patient_timeline".freeze
      PEC_KEY = "pec_external_view".freeze

      module_function

      def enabled?(city)
        return false if city.nil? || city.id.nil?

        Platform::Features.enabled?(city, KEY)
      end

      def pec_usable?(city)
        return false if city.nil? || city.id.nil?

        Platform::Features.usable?(city, PEC_KEY)
      end

      # Histórico parcial: fora do modo record as consultas da cidade estão no PEC.
      def partial?(city) = Platform::Features.settings(city)[:record_mode] != "record"
    end
  end
  ```

  Nas três specs com a lista fixa do catálogo, acrescente as duas chaves no fim:

  ```ruby
  %w[ledi_export cadsus_lookup clinical_record digital_signature signature_psc_mock clinical_documents controlled_prescriptions sncr_mock patient_timeline pec_external_view]
  ```

  Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..35).freeze`.

- [ ] **Step 4: Run** `rspec spec/services/timeline/gate_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/adr_pointers_spec.rb spec/services/platform`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/platform/features.rb app/services/timeline/gate.rb spec/services/timeline/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb
  git commit -m "feat: add patient timeline and pec external view feature switches

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 2: Migração de cidade, modelos, eventos e filtros

**Files:**
- Create: `db/city_migrate/20261011800002_add_patient_timeline.rb`
- Modify: `db/city_schema.rb` (versão, blocos de `attendances`, `campaign_recipients`, `clinical_documents`, `clinical_record_openings`, `consultations`, tabelas novas e FKs)
- Create: `app/models/pec_setting.rb`, `app/models/pec_signature_use.rb`
- Modify: `app/models/clinical_record_opening.rb`, `app/services/city_encryption.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Create: `spec/support/timeline_helpers.rb`
- Test: `spec/models/pec_setting_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features::PecConfigState` (Task 1).
- Produces: `PecSetting` (`self.current`, `#pec_origin`, `#hmac_key` cifrada, `#hmac_key_updated_at`, `#pec_origin_updated_at`, `#hmac_key_configured?`), `PecSignatureUse` (`signature_digest`, `expires_at`), `ClinicalRecordOpening#citizen` (opcional), helpers de spec `timeline_city!(record_mode: "record", clinical_record: true, documents: false, pec: false)`, `pec_settings!(origin:, key:)`, `TimelineHelpers::PEC_KEY`, `signed_pec_path(professional:, citizen:, cnes:, at: Time.current, key: PEC_KEY, host: test_city_host)`.

- [ ] **Step 1: Write the failing test** `spec/models/pec_setting_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035: configuração do PEC por cidade. A chave HMAC é cifrada com a
  # chave da cidade e nunca aparece em inspect; a linha é única.
  RSpec.describe PecSetting do
    it "cifra a chave, esconde no inspect e é linha única" do
      setting = PecSetting.create!(pec_origin: "https://pec.cidade.gov.br", hmac_key: "chave-secreta-0123456789",
                                   hmac_key_updated_at: Time.current, pec_origin_updated_at: Time.current)
      raw = ApplicationRecord.connection.select_value("SELECT hmac_key FROM pec_settings WHERE id = #{ApplicationRecord.connection.quote(setting.id)}")
      expect(raw).not_to include("chave-secreta")
      expect(setting.reload.hmac_key).to eq("chave-secreta-0123456789")
      expect(setting.inspect).not_to include("chave-secreta")
      expect(setting.inspect).to include("hmac_key_configured=true")
      expect(PecSetting.current).to eq(setting)
      expect { PecSetting.create! }.to raise_error(ActiveRecord::RecordNotUnique)
    end

    it "com origem e chave, pec_external_view fica utilizável (pedido M3)" do
      city = timeline_city!(record_mode: "integrated", clinical_record: false, pec: true)
      expect(Platform::Features.missing(city, "pec_external_view")).to eq(%w[pec_origin_missing pec_hmac_key_missing])
      pec_settings!
      expect(Platform::Features.missing(city, "pec_external_view")).to eq([])
      expect(Timeline::Gate.pec_usable?(city)).to be(true)
      PecSetting.current.update!(pec_origin: nil)
      expect(Platform::Features.missing(city, "pec_external_view")).to eq(%w[pec_origin_missing])
    end

    it "o banco recusa origem com caminho e chave sem data" do
      expect { PecSetting.create!(pec_origin: "https://pec.cidade.gov.br/esus") }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_pec_settings_origin/)
      expect { PecSetting.create!(hmac_key: "chave-secreta-0123456789") }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_pec_settings_key_updated/)
    end

    it "abertura justificada tem paciente OU par validado, nunca os dois nem nenhum" do
      user = staff_with("abre-#{SecureRandom.hex(3)}@cidade.gov.br", "health_professional")
      citizen = verified_citizen!(71)
      now = Time.current
      opening = ClinicalRecordOpening.create!(citizen: citizen, user: user, reason_code: "case_review",
                                              created_at: now, expires_at: now + 30.minutes)
      expect(opening.reload.citizen).to eq(citizen)
      expect { ClinicalRecordOpening.create!(user: user, reason_code: "case_review", created_at: now, expires_at: now + 1.minute) }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_clinical_record_openings_person/)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/models/pec_setting_spec.rb`. Expected: FAIL com `uninitialized constant PecSetting`.

- [ ] **Step 3: Escreva a migração** `db/city_migrate/20261011800002_add_patient_timeline.rb`:

  ```ruby
  # Módulo 20 (ADR 0035): linha do tempo do paciente montada na leitura.
  # - Índices por pessoa e data nas origens que não tinham (triagens,
  #   agendamentos, escutas e problemas já têm caminho indexado).
  # - Abertura justificada pela pessoa sem paciente: citizen_id (um par
  #   validado do CPF), exatamente um entre patient_id e citizen_id. O trigger
  #   clinical_record_openings_reencryption_only segue valendo (citizen_id não
  #   é cifrado).
  # - Configuração do PEC (linha única; chave HMAC cifrada pela aplicação) e o
  #   registro de uso único das assinaturas da página do PEC.
  # Tabelas pequenas por cidade: índices sem CONCURRENTLY. Irreversível.
  class AddPatientTimeline < ActiveRecord::Migration[8.1]
    def up
      add_index :attendances, %i[citizen_id checked_in_at], name: "idx_attendances_citizen_checked_in"
      add_index :clinical_documents, %i[citizen_id issued_at], name: "idx_clinical_documents_citizen_issued"
      add_index :consultations, %i[patient_id finalized_at], name: "idx_consultations_patient_finalized",
                                                             where: "((status)::text = 'finalized'::text)"
      add_index :campaign_recipients, %i[citizen_id created_at], name: "idx_campaign_recipients_citizen_created"

      change_column_null :clinical_record_openings, :patient_id, true
      add_column :clinical_record_openings, :citizen_id, :uuid
      add_foreign_key :clinical_record_openings, :citizens
      add_index :clinical_record_openings, %i[user_id citizen_id expires_at], name: "idx_clinical_record_openings_valid_citizen"
      add_check_constraint :clinical_record_openings, "(patient_id IS NULL) <> (citizen_id IS NULL)",
                           name: "ck_clinical_record_openings_person"

      create_table :pec_settings, id: :uuid do |t|
        t.boolean :singleton, null: false, default: true
        t.string :pec_origin, limit: 255
        t.text :hmac_key
        t.datetime :pec_origin_updated_at
        t.datetime :hmac_key_updated_at
        t.timestamps
      end
      add_index :pec_settings, :singleton, unique: true, name: "index_pec_settings_singleton"
      add_check_constraint :pec_settings, "singleton", name: "ck_pec_settings_singleton"
      add_check_constraint :pec_settings, "pec_origin IS NULL OR pec_origin::text ~ '^https?://[^/?#@ ]+$'::text",
                           name: "ck_pec_settings_origin"
      add_check_constraint :pec_settings, "(hmac_key IS NULL) = (hmac_key_updated_at IS NULL)",
                           name: "ck_pec_settings_key_updated"

      create_table :pec_signature_uses, id: :uuid do |t|
        t.string :signature_digest, limit: 64, null: false
        t.datetime :expires_at, null: false
        t.datetime :created_at, null: false
      end
      add_index :pec_signature_uses, :signature_digest, unique: true
      add_index :pec_signature_uses, :expires_at
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end
  end
  ```

- [ ] **Step 4: Atualize o dump** `db/city_schema.rb` à mão:
  - `ActiveRecord::Schema[8.1].define(version: 2026_10_11_800002) do`
  - em `create_table "attendances"`, depois do índice `index_attendances_on_citizen_id`: `t.index ["citizen_id", "checked_in_at"], name: "idx_attendances_citizen_checked_in"`
  - em `create_table "campaign_recipients"`, antes de `index_campaign_recipients_on_citizen_id`: `t.index ["citizen_id", "created_at"], name: "idx_campaign_recipients_citizen_created"`
  - em `create_table "clinical_documents"`, antes de `index_clinical_documents_on_citizen_id`: `t.index ["citizen_id", "issued_at"], name: "idx_clinical_documents_citizen_issued"`
  - em `create_table "consultations"`, antes de `index_consultations_on_patient_id`: `t.index ["patient_id", "finalized_at"], name: "idx_consultations_patient_finalized", where: "((status)::text = 'finalized'::text)"`
  - `create_table "clinical_record_openings"` vira:

    ```ruby
    create_table "clinical_record_openings", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
      t.uuid "citizen_id"
      t.datetime "created_at", null: false
      t.datetime "expires_at", null: false
      t.uuid "patient_id"
      t.string "reason_code", null: false
      t.text "reason_note"
      t.uuid "user_id", null: false
      t.index ["created_at"], name: "index_clinical_record_openings_on_created_at"
      t.index ["patient_id"], name: "index_clinical_record_openings_on_patient_id"
      t.index ["user_id", "citizen_id", "expires_at"], name: "idx_clinical_record_openings_valid_citizen"
      t.index ["user_id", "patient_id", "expires_at"], name: "idx_clinical_record_openings_valid"
      t.check_constraint "expires_at > created_at", name: "ck_clinical_record_openings_expiry"
      t.check_constraint "(reason_code::text = 'other'::text) = (reason_note IS NOT NULL)", name: "ck_clinical_record_openings_note"
      t.check_constraint "(patient_id IS NULL) <> (citizen_id IS NULL)", name: "ck_clinical_record_openings_person"
      t.check_constraint "reason_code::text = ANY (ARRAY['case_review'::text, 'active_search'::text, 'continuity_of_care'::text, 'other'::text])", name: "ck_clinical_record_openings_reason"
    end
    ```

  - tabelas novas, em ordem alfabética entre as existentes (`pec_settings` e `pec_signature_uses` depois de `patients`):

    ```ruby
    create_table "pec_settings", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
      t.datetime "created_at", null: false
      t.text "hmac_key"
      t.datetime "hmac_key_updated_at"
      t.string "pec_origin", limit: 255
      t.datetime "pec_origin_updated_at"
      t.boolean "singleton", default: true, null: false
      t.datetime "updated_at", null: false
      t.index ["singleton"], name: "index_pec_settings_singleton", unique: true
      t.check_constraint "(hmac_key IS NULL) = (hmac_key_updated_at IS NULL)", name: "ck_pec_settings_key_updated"
      t.check_constraint "pec_origin IS NULL OR pec_origin::text ~ '^https?://[^/?#@ ]+$'::text", name: "ck_pec_settings_origin"
      t.check_constraint "singleton", name: "ck_pec_settings_singleton"
    end

    create_table "pec_signature_uses", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
      t.datetime "created_at", null: false
      t.datetime "expires_at", null: false
      t.string "signature_digest", limit: 64, null: false
      t.index ["expires_at"], name: "index_pec_signature_uses_on_expires_at"
      t.index ["signature_digest"], name: "index_pec_signature_uses_on_signature_digest", unique: true
    end
    ```

  - FKs: depois de `add_foreign_key "clinical_record_openings", "patients"`, a linha `add_foreign_key "clinical_record_openings", "citizens"` (ordem alfabética do dump: `citizens` antes de `patients`).
  Rode a migração nos bancos de teste da branch (recrie-os como em "Ambiente de execução") e `rspec spec/services/city_schema_spec.rb`; se a paridade apontar diferença de ordem ou texto do CHECK, copie o que o banco devolveu.

- [ ] **Step 5: Modelos.** `app/models/pec_setting.rb`:

  ```ruby
  # Configuração do PEC da cidade (ADR 0035; contrato §6): endereço (origem)
  # do PEC, para o frame-ancestors, e a chave HMAC dos "sistemas externos",
  # cifrada com a chave da cidade. Linha única. Só o maintenance escreve
  # (PecExternal::Configure). A chave nunca sai: nem em resposta, nem em log,
  # nem em inspect.
  class PecSetting < ApplicationRecord
    encrypts :hmac_key

    def self.current = first

    def hmac_key_configured? = hmac_key_updated_at.present?

    def inspect
      "#<PecSetting id=#{id.inspect} pec_origin=#{pec_origin.inspect} hmac_key_configured=#{hmac_key_configured?}>"
    end
  end
  ```

  `app/models/pec_signature_use.rb`:

  ```ruby
  # Uso único das assinaturas da página do PEC (ADR 0035): o SHA-256 do
  # pec_sign, guardado até a janela da assinatura vencer. Nunca a assinatura.
  class PecSignatureUse < ApplicationRecord
  end
  ```

  Em `app/models/clinical_record_opening.rb`, troque `belongs_to :patient` por:

  ```ruby
    # ADR 0035: a abertura é de um paciente (19a) ou, sem paciente, de um par
    # validado do CPF (exatamente um dos dois; CHECK no banco).
    belongs_to :patient, optional: true
    belongs_to :citizen, optional: true
  ```

  Em `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS`:

  ```ruby
      # ADR 0035: chave HMAC dos "sistemas externos" do PEC.
      [ PecSetting, :hmac_key ]
  ```

- [ ] **Step 6: Eventos e filtros.** Em `config/initializers/domain_events.rb`, depois de `DomainEvents.bind "clinical_record.opened", to: []`:

  ```ruby
    # ADR 0035: página do PEC, só trilha (ids e código do motivo).
    DomainEvents.bind "pec.external_view_opened", to: []
    DomainEvents.bind "pec.external_view_denied", to: []
  ```

  Em `spec/initializers/domain_events_bindings_spec.rb`, no fim:

  ```ruby
  # Módulo 20 (ADR 0035): página do PEC, só trilha.
  RSpec.describe "patient timeline event bindings (ADR 0035)" do
    it "declares every module 20 city event with no consumer" do
      names = %w[clinical_record.viewed pec.external_view_opened pec.external_view_denied]
      expect(DomainEvents.registry.keys).to include(*names)
      expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
    end
  end
  ```

  Em `config/initializers/filter_parameter_logging.rb`, no fim da lista:

  ```ruby
    # ADR 0035: a chamada dos "sistemas externos" do PEC traz CPF/CNS do
    # cidadão e do profissional e a assinatura na query (a linha "Started GET"
    # usa request.filtered_path, que aplica esta lista à query).
    /\A(documento-cidadao|documento-profissional|pec_sign)\z/
  ```

- [ ] **Step 7: Helpers de spec** `spec/support/timeline_helpers.rb`:

  ```ruby
  # Módulo 20 (ADR 0035): a cidade com a linha do tempo, a configuração do PEC
  # e a URL assinada como o PEC monta (HMAC-SHA256 hex da URL até o
  # pec_timestamp; ver docs/pesquisa/2026-10-11-pec-sistemas-externos.md).
  module TimelineHelpers
    PEC_KEY = "chave-de-teste-do-pec-0123456789abcdef".freeze
    PEC_ORIGIN = "https://pec.cidade.gov.br".freeze

    def timeline_city!(record_mode: "record", clinical_record: true, documents: false, pec: false)
      city = clinical_city!(record_mode: record_mode, enabled: clinical_record)
      maintainer = ledi_maintainer!
      Platform::Features.set!(city: city, key: "patient_timeline", enabled: true, maintainer: maintainer)
      Platform::Features.set!(city: city, key: "clinical_documents", enabled: true, maintainer: maintainer) if documents
      Platform::Features.set!(city: city, key: "pec_external_view", enabled: true, maintainer: maintainer) if pec
      CityCatalog.reset_cache!
      city
    end

    def pec_settings!(origin: PEC_ORIGIN, key: PEC_KEY)
      setting = PecSetting.current || PecSetting.new
      setting.update!(pec_origin: origin, pec_origin_updated_at: Time.current, hmac_key: key,
                      hmac_key_updated_at: key && Time.current)
      setting
    end

    def signed_pec_path(professional:, citizen:, cnes:, at: Time.current, key: PEC_KEY, host: test_city_host)
      unsigned = "http://#{host}/pec/timeline?documento-profissional=#{professional}" \
                 "&documento-cidadao=#{citizen}&cnes=#{cnes}&pec_timestamp=#{at.to_i}"
      "#{unsigned}&pec_sign=#{OpenSSL::HMAC.hexdigest('SHA256', key, unsigned)}"
    end
  end

  RSpec.configure { |c| c.include TimelineHelpers }
  ```

  (Se `spec/rails_helper.rb` carrega `spec/support` por `require_relative` arquivo a arquivo, acrescente a linha deste arquivo junto das outras.)

- [ ] **Step 8: Run** `rspec spec/models/pec_setting_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/requests/clinical_record_spec.rb spec/commands/clinical_record_reencryption_spec.rb`. Expected: PASS.

- [ ] **Step 9: Commit**

  ```bash
  git add db/city_migrate/20261011800002_add_patient_timeline.rb db/city_schema.rb app/models/pec_setting.rb app/models/pec_signature_use.rb app/models/clinical_record_opening.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/timeline_helpers.rb spec/models/pec_setting_spec.rb
  git commit -m "feat: add patient timeline city schema, pec settings and person openings

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 2 — A pessoa e a leitura

### Task 3: `Timeline::Person`, `Timeline::Entry` e `Timeline::Cursor`

**Files:**
- Create: `app/services/timeline/person.rb`, `app/services/timeline/entry.rb`, `app/services/timeline/cursor.rb`
- Test: `spec/services/timeline/person_spec.rb`, `spec/services/timeline/cursor_spec.rb`

**Interfaces:**
- Produces:
  - `Timeline::KINDS` (os 8 kinds, nesta ordem: `triage attendance screening appointment consultation clinical_document controlled_notification campaign`).
  - `Timeline::Person = Data.define(:citizen_ids, :patient_id, :verified)`; `Person.for_patient(patient)`, `Person.for_cpf(cpf)`, `Person.for_citizen(citizen)`; `#empty?`, `#ref -> "patient:<uuid>" | "citizen:<uuid>"`, `#display_name -> String|nil`.
  - `Timeline::Entry = Data.define(:kind, :source_id, :occurred_at, :unit, :professional, :title, :summary, :status, :detail_ref, :citizen, :document)`; `#sort_key`. `unit` = `{ id:, name: }|nil`; `professional` = `{ name:, cbo_code: }|nil`; `detail_ref` = `{ type:, id: }`; `citizen` = `{ title:, summary:, status:, professional_name: }`; `document` = `{ id:, short_code:, verification_url:, downloadable: }|nil`.
  - `Timeline::Cursor = Data.define(:at, :kind, :id)`; `Cursor.encode(entry) -> String`, `Cursor.decode(raw) -> Cursor|nil` (nil para vazio), `Timeline::Cursor::Invalid < StandardError`.

- [ ] **Step 1: Write the failing tests.** `spec/services/timeline/person_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035 (spec §3): a pessoa é o conjunto dos pares VALIDADOS do CPF
  # (mais o paciente do 19a, quando existe). Par só declarado nunca entra.
  RSpec.describe Timeline::Person do
    before { clinical_city! }

    def declared_twin_of(citizen)
      Citizen.create!(cpf: citizen.cpf, phone: "+5541977770001", verification_level: "declared")
    end

    it "pares que entram e saem: declarado, revogado e excluído ficam de fora (Review Focus 4)" do
      verified = verified_citizen!(31)
      twin = declared_twin_of(verified)
      other = verified_citizen!(32)
      person = described_class.for_cpf(verified.cpf)
      expect(person.citizen_ids).to eq([ verified.id ])
      expect(person.citizen_ids).not_to include(twin.id, other.id)
      expect(person.verified).to be(true)
      expect(person.patient_id).to be_nil

      verified.update!(verification_level: "declared")
      expect(described_class.for_cpf(verified.cpf)).to be_empty

      verified.update!(verification_level: "verified", erased_at: Time.current)
      expect(described_class.for_cpf(verified.cpf)).to be_empty
    end

    it "com paciente, junta os pares ligados a ele e os validados do CPF" do
      citizen = verified_citizen!(33)
      patient = Patients::Resolve.call(citizen).payload[:patient]
      person = described_class.for_patient(patient)
      expect(person.patient_id).to eq(patient.id)
      expect(person.citizen_ids).to eq([ citizen.id ])
      expect(described_class.for_cpf("#{citizen.cpf[0, 3]}.#{citizen.cpf[3, 3]}.#{citizen.cpf[6, 3]}-#{citizen.cpf[9, 2]}"))
        .to eq(person)
      expect(person.ref).to eq("patient:#{patient.id}")
    end

    it "for_citizen: validado vira a pessoa do CPF; declarado vê só o próprio par" do
      verified = verified_citizen!(34)
      twin = declared_twin_of(verified)
      expect(described_class.for_citizen(verified).citizen_ids).to eq([ verified.id ])
      own = described_class.for_citizen(twin)
      expect([ own.citizen_ids, own.verified, own.patient_id ]).to eq([ [ twin.id ], false, nil ])
      expect(own.ref).to eq("citizen:#{twin.id}")
    end

    it "CPF inválido ou vazio é a pessoa vazia; nome de exibição do paciente ou do par" do
      expect(described_class.for_cpf("123")).to be_empty
      expect(described_class.for_cpf(nil)).to be_empty
      citizen = verified_citizen!(35, social_name: "Bia")
      expect(described_class.for_cpf(citizen.cpf).display_name).to eq("Bia")
    end
  end
  ```

  `spec/services/timeline/cursor_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe Timeline::Cursor do
    let(:at) { Time.utc(2026, 10, 11, 12, 0, 0, 123_456) }
    let(:entry) do
      Timeline::Entry.new(kind: "attendance", source_id: "0b5f8f4e-8f69-4f3e-9d2c-6f3c2c1f0a11", occurred_at: at,
                          unit: nil, professional: nil, title: "t", summary: nil, status: nil,
                          detail_ref: { type: "attendance", id: "x" }, citizen: { title: "t", summary: nil, status: nil, professional_name: nil },
                          document: nil)
    end

    it "ida e volta com microssegundos, kind e id" do
      cursor = described_class.decode(described_class.encode(entry))
      expect([ cursor.at, cursor.kind, cursor.id ]).to eq([ at, "attendance", entry.source_id ])
      expect(cursor.at.usec).to eq(123_456)
    end

    it "vazio é nil; lixo, kind desconhecido e id que não é uuid levantam Invalid" do
      expect(described_class.decode(nil)).to be_nil
      expect(described_class.decode("")).to be_nil
      bad = [ "%%%", Base64.urlsafe_encode64("2026-10-11T12:00:00.000000Z|exam|#{entry.source_id}", padding: false),
              Base64.urlsafe_encode64("2026-10-11T12:00:00.000000Z|triage|1; DROP", padding: false),
              Base64.urlsafe_encode64("ontem|triage|#{entry.source_id}", padding: false), [ "x" ] ]
      bad.each { |raw| expect { described_class.decode(raw) }.to raise_error(Timeline::Cursor::Invalid), raw.inspect }
    end

    it "sort_key ordena por data decrescente, kind e id" do
      later = entry.with(occurred_at: at + 1)
      same_time_other_kind = entry.with(kind: "appointment")
      expect([ entry, later, same_time_other_kind ].sort_by(&:sort_key)).to eq([ later, same_time_other_kind, entry ])
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/timeline/person_spec.rb spec/services/timeline/cursor_spec.rb`. Expected: FAIL (`uninitialized constant Timeline::Person`).

- [ ] **Step 3: Implemente.** `app/services/timeline/entry.rb`:

  ```ruby
  # Uma entrada da linha do tempo (ADR 0035; contrato §2), montada por um
  # fornecedor (Timeline::Sources::*). `citizen` é a visão do cidadão
  # (título, resumo e situação próprios, nunca texto clínico livre).
  module Timeline
    KINDS = %w[triage attendance screening appointment consultation clinical_document controlled_notification campaign].freeze

    Entry = Data.define(:kind, :source_id, :occurred_at, :unit, :professional, :title, :summary, :status, :detail_ref,
                        :citizen, :document) do
      # Ordem total: occurred_at decrescente, kind e id crescentes (a mesma do SQL dos fornecedores).
      def sort_key = [ -occurred_at.to_r, kind, source_id ]
    end
  end
  ```

  `app/services/timeline/cursor.rb`:

  ```ruby
  require "base64"

  # Cursor composto e opaco da linha do tempo (contrato §3): o instante com
  # microssegundos, o kind e o id da origem da última entrada da página.
  module Timeline
    Cursor = Data.define(:at, :kind, :id) do
      const_set(:Invalid, Class.new(StandardError))
      const_set(:UUID, /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/)
      const_set(:INSTANT, /\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{6}Z\z/)

      def self.encode(entry)
        Base64.urlsafe_encode64("#{entry.occurred_at.utc.iso8601(6)}|#{entry.kind}|#{entry.source_id}", padding: false)
      end

      def self.decode(raw)
        return nil if raw.nil? || raw == ""
        raise self::Invalid unless raw.is_a?(String)

        at, kind, id = Base64.urlsafe_decode64(raw).split("|", 3)
        raise self::Invalid unless at.to_s.match?(self::INSTANT) && Timeline::KINDS.include?(kind) && id.to_s.match?(self::UUID)

        new(at: Time.iso8601(at), kind: kind, id: id.downcase)
      rescue ArgumentError
        raise self::Invalid
      end
    end
  end
  ```

  `app/services/timeline/person.rb`:

  ```ruby
  # "A pessoa" da linha do tempo (ADR 0035; spec §3): os pares VALIDADOS e não
  # excluídos do CPF — no modo record também os ligados ao paciente do 19a —
  # e o paciente, quando existe. Par só declarado nunca entra; para o cidadão
  # de par declarado no wpda, a pessoa é só o próprio par (verified: false).
  module Timeline
    Person = Data.define(:citizen_ids, :patient_id, :verified) do
      def self.verified_pairs = Citizen.not_erased.verification_level_verified

      def self.for_patient(patient)
        ids = verified_pairs.where(patient_id: patient.id).or(verified_pairs.where(cpf: patient.cpf))
                            .order(:created_at, :id).pluck(:id)
        new(citizen_ids: ids, patient_id: patient.id, verified: true)
      end

      def self.for_cpf(cpf)
        digits = cpf.is_a?(String) ? CitizenIdentity::Cpf.normalize(cpf) : nil
        return new(citizen_ids: [], patient_id: nil, verified: true) unless digits

        patient = Patient.find_by(cpf: digits)
        return for_patient(patient) if patient

        new(citizen_ids: verified_pairs.where(cpf: digits).order(:created_at, :id).pluck(:id), patient_id: nil, verified: true)
      end

      def self.for_citizen(citizen)
        return for_cpf(citizen.cpf) if citizen.verification_level_verified? && citizen.erased_at.nil?

        new(citizen_ids: [ citizen.id ], patient_id: nil, verified: false)
      end

      def empty? = citizen_ids.empty? && patient_id.nil?

      # Referência da pessoa na trilha: só id.
      def ref = patient_id ? "patient:#{patient_id}" : "citizen:#{citizen_ids.min}"

      def display_name
        patient = patient_id && Patient.find_by(id: patient_id)
        return patient.display_name if patient

        Citizen.where(id: citizen_ids).order(created_at: :desc).first&.display_name
      end
    end
  end
  ```

  Confira `CitizenIdentity::Cpf.normalize` no código: ele devolve os 11 dígitos de um CPF válido (com ou sem máscara) e `nil` caso contrário; se a assinatura for outra, ajuste a chamada (não o teste).

- [ ] **Step 4: Run** `rspec spec/services/timeline/person_spec.rb spec/services/timeline/cursor_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/timeline/person.rb app/services/timeline/entry.rb app/services/timeline/cursor.rb spec/services/timeline/person_spec.rb spec/services/timeline/cursor_spec.rb
  git commit -m "feat: add timeline person, entry and composite cursor

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 4: `ClinicalRecord::Access.for_person` e a abertura pela pessoa, sem quebrar o 19a

**Files:**
- Modify: `app/services/clinical_record/access.rb` (método novo `for_person`; `call` e `for_consultation` intocados)
- Modify: `app/models/clinical_record_opening.rb` (scope `valid_for_person`)
- Modify: `app/commands/clinical_record/open.rb`, `app/controllers/clinical_record_openings_controller.rb`
- Test: `spec/services/clinical_record/access_for_person_spec.rb`, `spec/commands/clinical_record/open_spec.rb` (exemplos novos), `spec/requests/clinical_record_openings_timeline_spec.rb`

**Interfaces:**
- Consumes: `Timeline::Person` (Task 3), `Timeline::Gate.enabled?` (Task 1).
- Produces: `ClinicalRecord::Access.for_person(user:, person:, attendance: nil, now: Time.current) -> Grant` (`kind` `:in_context` | `:justified` | `:denied`, `reason` `:missing_role` | `:out_of_context`); `ClinicalRecordOpening.valid_for_person(user_id:, person:, now: Time.current)`; `ClinicalRecord::Open.call` cria abertura com `citizen` quando o CPF não tem paciente e `patient_timeline` está ligado.

- [ ] **Step 1: Write the failing tests.** `spec/services/clinical_record/access_for_person_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035 (spec §5): a leitura da linha do tempo usa as regras do 19a sobre
  # A PESSOA. `call` e `for_consultation` do 19a ficam iguais (regressão abaixo).
  RSpec.describe ClinicalRecord::Access do
    include ActiveSupport::Testing::TimeHelpers
    before { timeline_city!(record_mode: "integrated", clinical_record: false) }

    let(:unit) { create_unit }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(41) }
    let(:person) { Timeline::Person.for_cpf(citizen.cpf) }

    it "em contexto pelo atendimento chamado; outro profissional, fora de contexto" do
      attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
      expect(described_class.for_person(user: doctor, person: person).kind).to eq(:in_context)
      expect(described_class.for_person(user: doctor, person: person, attendance: attendance).kind).to eq(:in_context)
      other = doctor!(unit, cbo: "225142")
      expect(described_class.for_person(user: other, person: person, attendance: attendance).reason).to eq(:out_of_context)
    end

    it "atendimento de outra pessoa não dá contexto" do
      attendance = consulting_attendance!(unit, citizen: verified_citizen!(42), doctor: doctor)
      expect(described_class.for_person(user: doctor, person: person, attendance: attendance).reason).to eq(:out_of_context)
    end

    it "abertura pelo par, válida 30 min; recepção sem papel" do
      now = Time.current
      opening = ClinicalRecordOpening.create!(citizen: citizen, user: doctor, reason_code: "active_search",
                                              created_at: now, expires_at: now + 30.minutes)
      grant = described_class.for_person(user: doctor, person: person)
      expect([ grant.kind, grant.opening ]).to eq([ :justified, opening ])
      travel_to(31.minutes.from_now) { expect(described_class.for_person(user: doctor, person: person).reason).to eq(:out_of_context) }
      expect(described_class.for_person(user: reception!, person: person).reason).to eq(:missing_role)
    end
  end

  RSpec.describe "ClinicalRecord::Access do 19a (regressão)" do
    before { clinical_city!; ciap2_release!; cid10_release!; sigtap_release! }

    let(:unit) { create_unit }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(43) }

    it "call e for_consultation respondem como antes" do
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      patient = consultation.patient
      attendance = consulting_attendance!(unit, citizen: citizen.reload, doctor: doctor)
      expect(ClinicalRecord::Access.call(user: doctor, patient: patient, attendance: attendance).kind).to eq(:in_context)
      expect(ClinicalRecord::Access.call(user: doctor!(unit, cbo: "225142"), patient: patient).reason).to eq(:out_of_context)
      expect(ClinicalRecord::Access.call(user: reception!, patient: patient).reason).to eq(:missing_role)
      expect(ClinicalRecord::Access.for_consultation(user: doctor, consultation: consultation).kind).to eq(:author)
    end
  end
  ```

  Em `spec/commands/clinical_record/open_spec.rb`, acrescente (use o `let`/`before` do arquivo para o usuário com vínculo; os nomes abaixo supõem `user` com step-up desnecessário, pois o comando não faz step-up):

  ```ruby
  describe "pela pessoa sem paciente (ADR 0035)" do
    let(:unit) { create_unit }
    let(:nurse) { doctor!(unit, cbo: "223505") }

    it "com patient_timeline ligado, abre pelo par validado; sem par validado, patient_not_found" do
      timeline_city!(record_mode: "integrated", clinical_record: false)
      citizen = verified_citizen!(44)
      result = ClinicalRecord::Open.call(user: nurse, cpf: citizen.cpf, reason_code: "active_search")
      expect(result).to be_success
      opening = result.payload[:opening]
      expect([ opening.patient_id, opening.citizen_id ]).to eq([ nil, citizen.id ])
      expect(DomainEvent.where(name: "clinical_record.opened").last.payload)
        .to include("opening_id" => opening.id, "patient_id" => nil, "citizen_id" => citizen.id)
      declared = screening_citizen!(45)
      expect(ClinicalRecord::Open.call(user: nurse, cpf: declared.cpf, reason_code: "active_search").reason)
        .to eq(:patient_not_found)
    end

    it "com patient_timeline desligado, CPF sem paciente segue patient_not_found (19a)" do
      clinical_city!
      citizen = verified_citizen!(46)
      expect(ClinicalRecord::Open.call(user: nurse, cpf: citizen.cpf, reason_code: "active_search").reason)
        .to eq(:patient_not_found)
    end
  end
  ```

  `spec/requests/clinical_record_openings_timeline_spec.rb`:

  ```ruby
  require "rails_helper"

  # Divergência D5: a abertura vale com patient_timeline ligado mesmo sem o
  # prontuário utilizável; patient_id nulo quando a pessoa não tem paciente; o
  # relatório mostra o CPF mascarado do par.
  RSpec.describe "Abertura justificada pela pessoa", type: :request do
    let(:unit) { create_unit }
    let(:nurse) { doctor!(unit, cbo: "223505") }
    def body = JSON.parse(response.body)

    it "modo integrado: cria pelo par e aparece no relatório" do
      timeline_city!(record_mode: "integrated", clinical_record: false)
      citizen = verified_citizen!(47)
      stepped_up!(nurse)
      json_post "/clinical_record/openings", cpf: citizen.cpf, reason_code: "active_search"
      expect(response).to have_http_status(:created)
      expect(body.keys).to match_array(%w[opening_id patient_id expires_at])
      expect(body["patient_id"]).to be_nil

      sign_in_as(staff_with("adm-tl@cidade.gov.br", "municipal_admin"))
      get "/clinical_record/openings"
      expect(body["items"].sole).to include("kind" => "justified_opening", "cpf_masked" => citizen.cpf_masked)
    end

    it "sem patient_timeline e sem prontuário: 403 feature_disabled como no 19a" do
      clinical_city!(record_mode: "integrated", enabled: false)
      stepped_up!(nurse)
      json_post "/clinical_record/openings", cpf: verified_citizen!(48).cpf, reason_code: "active_search"
      expect([ response.status, body["error"], body["feature"] ]).to eq([ 403, "feature_disabled", "clinical_record" ])
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/clinical_record/access_for_person_spec.rb spec/commands/clinical_record/open_spec.rb spec/requests/clinical_record_openings_timeline_spec.rb`. Expected: FAIL (`undefined method 'for_person'`).

- [ ] **Step 3: Implemente.** Em `app/services/clinical_record/access.rb`, logo depois de `for_consultation` (sem tocar em `call`, `for_consultation`, `open_attendances_of` nem `in_context?`):

  ```ruby
      # ADR 0035 (spec §5): a mesma regra sobre A PESSOA da linha do tempo
      # (Timeline::Person): em contexto por um atendimento aberto de um par dela,
      # ou abertura justificada válida do paciente OU de um par dela.
      def for_person(user:, person:, attendance: nil, now: Time.current)
        return deny(:missing_role) unless user&.has_role?("health_professional")

        candidates = attendance ? [ attendance ] : Attendance.open_attendances.where(citizen_id: person.citizen_ids).includes(:citizen).to_a
        in_context = candidates.any? { |a| person.citizen_ids.include?(a.citizen_id) && in_context?(user, a, nil) }
        return Grant.new(kind: :in_context, opening: nil, reason: nil) if in_context

        opening = ClinicalRecordOpening.valid_for_person(user_id: user.id, person: person, now: now).order(created_at: :desc).first
        opening ? Grant.new(kind: :justified, opening: opening, reason: nil) : deny(:out_of_context)
      end
  ```

  Em `app/models/clinical_record_opening.rb`, depois de `scope :valid_for`:

  ```ruby
    # ADR 0035: abertura válida da pessoa — do paciente ou de um par dela.
    scope :valid_for_person, ->(user_id:, person:, now: Time.current) {
      base = where(user_id: user_id).where("expires_at > ?", now)
      by_pair = base.where(citizen_id: person.citizen_ids)
      person.patient_id ? base.where(patient_id: person.patient_id).or(by_pair) : by_pair
    }
  ```

  Em `app/commands/clinical_record/open.rb`, troque o trecho do paciente por:

  ```ruby
        digits = CitizenIdentity::Cpf.normalize(cpf)
        patient = digits && Patient.find_by(cpf: digits)
        # ADR 0035 (Divergência D5): sem paciente, com a linha do tempo ligada, a
        # abertura é de um par VALIDADO do CPF (a pessoa da linha do tempo).
        citizen = nil
        if patient.nil? && digits && Timeline::Gate.enabled?(Current.city)
          citizen = Citizen.not_erased.verification_level_verified.where(cpf: digits).order(:created_at, :id).first
        end
        return Result.fail(:patient_not_found) unless patient || citizen

        now = Time.current
        opening = ClinicalRecordOpening.create!(patient: patient, citizen: citizen, user: user, reason_code: code, reason_note: note,
                                                created_at: now, expires_at: now + ClinicalRecordOpening::VALIDITY)
        DomainEvents.publish("clinical_record.opened", opening_id: opening.id, patient_id: patient&.id, citizen_id: citizen&.id,
                                                       user_id: user.id, reason_code: code)
        Result.ok(opening: opening)
  ```

  Atualize o comentário do topo do comando ("CPF que não tem paciente... = patient_not_found" ganha "salvo com patient_timeline ligado e par validado (ADR 0035)"). Confira na spec existente do 19a se algum exemplo prende o payload exato de `clinical_record.opened`; se prender, acrescente `"citizen_id" => nil` ao esperado (é o contrato novo, não afrouxamento).

  Em `app/controllers/clinical_record_openings_controller.rb`:
  - troque `before_action :require_clinical_record!` por `before_action :require_record_or_timeline!` e acrescente no `private`:

    ```ruby
    # ADR 0035 (Divergência D5): a abertura também serve à linha do tempo, que
    # existe sem o prontuário (modo integrado). Sem nenhum dos dois, o 403 do 19a.
    def require_record_or_timeline!
      return if ClinicalRecord::Gate.usable?(Current.city) || Timeline::Gate.enabled?(Current.city)

      render json: { error: "feature_disabled", feature: ClinicalRecord::Gate::KEY }, status: :forbidden
    end
    ```

  - em `openings`, `includes(:patient, :citizen, user: :professional)`; em `item`, `cpf_masked: (opening.patient || opening.citizen)&.cpf_masked`.

- [ ] **Step 4: Run** `rspec spec/services/clinical_record spec/commands/clinical_record spec/requests/clinical_record_spec.rb spec/requests/clinical_record_openings_timeline_spec.rb spec/requests/clinical_record_documents_spec.rb spec/invariants/clinical_record_invariants_spec.rb spec/invariants/clinical_regression_spec.rb`. Expected: PASS (o 19a inteiro verde).

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/clinical_record/access.rb app/models/clinical_record_opening.rb app/commands/clinical_record/open.rb app/controllers/clinical_record_openings_controller.rb spec/services/clinical_record/access_for_person_spec.rb spec/commands/clinical_record/open_spec.rb spec/requests/clinical_record_openings_timeline_spec.rb
  git commit -m "feat: generalize clinical record access and openings to the timeline person

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 3 — Fornecedores, junção e paginação

### Task 5: Janela comum e fornecedores administrativos (`triage`, `attendance`, `screening`, `appointment`)

**Files:**
- Create: `app/services/timeline/sources/window.rb`, `app/services/timeline/sources/labels.rb`
- Create: `app/services/timeline/sources/triage_source.rb`, `attendance_source.rb`, `screening_source.rb`, `appointment_source.rb`
- Test: `spec/services/timeline/sources/administrative_sources_spec.rb`

**Interfaces:**
- Consumes: `Timeline::Entry`, `Timeline::Cursor`, `Timeline::Person` (Task 3).
- Produces: `Timeline::Sources::Window.apply(scope, kind:, column:, id_column:, before:, from:, to:) -> ActiveRecord::Relation` (filtro de período, "estritamente depois do cursor" e ordem `column DESC, id_column ASC`); cada fornecedor `Timeline::Sources::<X>Source` com `KIND` e `entries(person:, before:, limit:, from:, to:) -> Array<Timeline::Entry>`; `Timeline::Sources::Labels` (mapas pt-BR).

Os fornecedores se chamam `*Source` para não sombrear os modelos e os módulos de serviço de mesmo nome (`::Screening`, `Screenings::Json`).

- [ ] **Step 1: Write the failing test** `spec/services/timeline/sources/administrative_sources_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035 (spec §4): cada fornecedor devolve as entradas da pessoa, depois
  # do cursor, na ordem occurred_at DESC, id ASC, com a visão do profissional
  # e a do cidadão. Nenhum texto livre na visão do cidadão.
  RSpec.describe "Fornecedores administrativos da linha do tempo" do
    include ActiveSupport::Testing::TimeHelpers
    before do
      timeline_city!(record_mode: "integrated", clinical_record: false)
      ciap2_release!
      acolhimento!
      ensure_appointment_types!
    end

    let(:unit) { create_unit("UBS Vila Nova") }
    let(:nurse) { screener!(unit) }
    let(:citizen) { verified_citizen!(51) }
    let(:person) { Timeline::Person.for_cpf(citizen.cpf) }
    let(:stranger) { verified_citizen!(52) }

    def entries(source, **opts) = source.entries(person: person, before: nil, limit: 50, from: nil, to: nil, **opts)

    it "triage: sem as em andamento, com protocolo e prioridade; nunca a de outra pessoa" do
      triage = completed_web_triage_for(citizen)
      completed_web_triage_for(stranger)
      entry = entries(Timeline::Sources::TriageSource).sole
      expect(entry.to_h.slice(:kind, :source_id, :status)).to eq(kind: "triage", source_id: triage.id, status: triage.status)
      expect(entry.occurred_at).to eq(triage.created_at)
      expect(entry.detail_ref).to eq(type: "triage", id: triage.id)
      expect(entry.citizen[:title]).to start_with("Triagem")
    end

    it "attendance: unidade, chegada e desfecho em termos simples para o cidadão" do
      attendance = walk_in_attendance!(unit, citizen: citizen)
      attendance.update_columns(status: "closed", outcome: "discharged", closed_by_user_id: nurse.id, closed_at: Time.current)
      entry = entries(Timeline::Sources::AttendanceSource).sole
      expect(entry.unit).to eq(id: unit.id, name: "UBS Vila Nova")
      expect(entry.occurred_at).to eq(attendance.checked_in_at)
      expect(entry.summary).to eq("Alta")
      expect(entry.citizen).to include(title: "Atendimento na UBS Vila Nova", summary: "Atendimento concluído")
    end

    it "screening: só concluída; profissional vê cor, destino e CIAP-2; cidadão vê 'acolhimento realizado'" do
      attendance = walk_in_attendance!(unit, citizen: citizen)
      started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
      Screenings::Complete.call(screening: started, revision_params: revision_params, destination: "oriented",
                                destination_params: { "orientation_note" => "repouso e hidratação" }, by: nurse)
      entry = entries(Timeline::Sources::ScreeningSource).sole
      expect(entry.title).to eq("Escuta inicial — verde")
      expect(entry.summary).to include("Orientado", "K86 Hipertensão sem complicações")
      expect(entry.professional).to eq(name: nurse.professional.professional_name, cbo_code: "223505")
      expect(entry.citizen).to eq(title: "Acolhimento realizado", summary: nil, status: nil,
                                  professional_name: nurse.professional.professional_name)
      expect(entry.to_h.to_s).not_to include("repouso e hidratação")
    end

    it "appointment: tipo e situação; o horário movido some" do
      request = triage_request!(citizen, unit: unit)
      link = doctor_link!(unit)
      shift = shift!(link, starts_at: 2.days.from_now.change(hour: 8))
      appointment = appointment_row!(request, shift, starts_at: shift.starts_at)
      entry = entries(Timeline::Sources::AppointmentSource).sole
      expect([ entry.source_id, entry.occurred_at, entry.status ]).to eq([ appointment.id, appointment.scheduled_at, "confirmed" ])
      expect(entry.title).to start_with("Agendamento — ")
      expect(entry.detail_ref).to eq(type: "appointment_request", id: request.id)
      appointment.update_columns(status: "moved", ended_at: Time.current)
      expect(entries(Timeline::Sources::AppointmentSource)).to be_empty
    end

    it "janela: período, cursor estritamente depois e desempate por id" do
      at = Time.zone.parse("2026-09-01 10:00")
      rows = Array.new(3) { walk_in_attendance!(unit, citizen: citizen, checked_in_at: at) }
      older = walk_in_attendance!(unit, citizen: citizen, checked_in_at: at - 1.day)
      all = entries(Timeline::Sources::AttendanceSource)
      expect(all.map(&:source_id)).to eq(rows.map(&:id).sort + [ older.id ])
      cursor = Timeline::Cursor.decode(Timeline::Cursor.encode(all.first))
      after = Timeline::Sources::AttendanceSource.entries(person: person, before: cursor, limit: 50, from: nil, to: nil)
      expect(after.map(&:source_id)).to eq(all.drop(1).map(&:source_id))
      ranged = Timeline::Sources::AttendanceSource.entries(person: person, before: nil, limit: 50,
                                                           from: (at - 1.hour), to: (at + 1.hour))
      expect(ranged.map(&:source_id)).to match_array(rows.map(&:id))
      earlier_kind = Timeline::Cursor.new(at: at, kind: "appointment", id: rows.first.id)
      expect(Timeline::Sources::AttendanceSource.entries(person: person, before: earlier_kind, limit: 50, from: nil, to: nil).size).to eq(4)
      later_kind = Timeline::Cursor.new(at: at, kind: "campaign", id: rows.first.id)
      expect(Timeline::Sources::AttendanceSource.entries(person: person, before: later_kind, limit: 50, from: nil, to: nil).map(&:source_id))
        .to eq([ older.id ])
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/timeline/sources/administrative_sources_spec.rb`. Expected: FAIL (`uninitialized constant Timeline::Sources`).

- [ ] **Step 3: Implemente.** `app/services/timeline/sources/window.rb`:

  ```ruby
  # Janela comum dos fornecedores (ADR 0035; spec §4): período [from, to],
  # "estritamente depois do cursor" na ordem total occurred_at DESC, kind ASC,
  # id ASC, e a mesma ordem no SQL. Para um fornecedor de kind K e o cursor
  # (t, k, i): K antes de k → só o que é mais antigo que t; K depois de k → t
  # também entra; K igual → mais antigo que t, ou t com id maior que i.
  module Timeline
    module Sources
      module Window
        module_function

        def apply(scope, kind:, column:, id_column:, before:, from:, to:)
          scope = scope.where("#{column} >= ?", from) if from
          scope = scope.where("#{column} <= ?", to) if to
          scope = after(scope, kind, column, id_column, before) if before
          scope.reorder(Arel.sql("#{column} DESC, #{id_column} ASC"))
        end

        def after(scope, kind, column, id_column, before)
          if kind < before.kind
            scope.where("#{column} < ?", before.at)
          elsif kind > before.kind
            scope.where("#{column} <= ?", before.at)
          else
            scope.where("#{column} < :at OR (#{column} = :at AND #{id_column} > :id)", at: before.at, id: before.id)
          end
        end
        private_class_method :after
      end
    end
  end
  ```

  `app/services/timeline/sources/labels.rb`:

  ```ruby
  # Textos pt-BR das entradas (ADR 0035). O cidadão recebe termos simples;
  # nada aqui é texto livre gravado por alguém.
  module Timeline
    module Sources
      module Labels
        ATTENDANCE_OUTCOMES = { "discharged" => "Alta", "referred" => "Encaminhado", "return" => "Retorno",
                                "left" => "Saiu sem atendimento", "scheduled_from_screening" => "Agendado na escuta",
                                "oriented" => "Orientado na escuta" }.freeze
        CITIZEN_OUTCOMES = { "discharged" => "Atendimento concluído", "referred" => "Encaminhado(a) para outro serviço",
                             "return" => "Retorno combinado", "left" => "Saiu antes do atendimento",
                             "scheduled_from_screening" => "Consulta agendada no acolhimento",
                             "oriented" => "Orientado(a) no acolhimento" }.freeze
        COLORS = { "red" => "vermelha", "yellow" => "amarela", "green" => "verde", "blue" => "azul" }.freeze
        DESTINATIONS = { "same_day" => "Atendimento no dia", "schedule" => "Agendamento", "oriented" => "Orientado",
                         "referred" => "Encaminhado" }.freeze
        DOCUMENT_KINDS = { "sick_note" => "Atestado", "attendance_declaration" => "Declaração de comparecimento",
                           "prescription" => "Receita", "exam_requisition" => "Requisição de exames" }.freeze
        CATEGORIES = { "special_control" => "Receita de controle especial", "antimicrobial" => "Receita de antimicrobiano" }.freeze
        MODES = { "digital" => "Assinado digitalmente", "paper" => "Papel" }.freeze
        DOCUMENT_STATUS = { "issued" => "Emitido", "cancelled" => "Cancelado" }.freeze

        module_function

        def professional_name(user) = user&.professional&.professional_name
      end
    end
  end
  ```

  `app/services/timeline/sources/triage_source.rb`:

  ```ruby
  # Triagens (módulos 01/03/15): da conversa de um par da pessoa, sem as em
  # andamento. Profissional: protocolo, prioridade, classificação. Cidadão:
  # protocolo, data, prioridade.
  module Timeline
    module Sources
      module TriageSource
        KIND = "triage".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          scope = ::Triage.joins(:conversation).where(conversations: { citizen_id: person.citizen_ids })
                          .where.not(status: "in_progress").includes(:protocol_definition)
          Window.apply(scope, kind: KIND, column: "triages.created_at", id_column: "triages.id", before: before, from: from, to: to)
                .limit(limit).map { |triage| entry(triage) }
        end

        def entry(triage)
          definition = triage.protocol_definition&.definition
          protocol = (definition.is_a?(Hash) && definition.dig("offer", "title").presence) || triage.protocol_name
          priority = triage.priority && "Prioridade #{triage.priority}"
          summary = [ priority, triage.tier && "Classificação #{triage.tier}" ].compact.join(" · ").presence
          Entry.new(kind: KIND, source_id: triage.id, occurred_at: triage.created_at, unit: nil, professional: nil,
                    title: "Triagem — #{protocol}", summary: summary, status: triage.status,
                    detail_ref: { type: "triage", id: triage.id },
                    citizen: { title: "Triagem — #{protocol}", summary: priority, status: triage.status, professional_name: nil },
                    document: nil)
        end
      end
    end
  end
  ```

  `app/services/timeline/sources/attendance_source.rb`:

  ```ruby
  # Atendimentos (módulo 13): chegada na unidade e desfecho. A nota de
  # encaminhamento e o motivo de exceção (texto livre) nunca entram.
  module Timeline
    module Sources
      module AttendanceSource
        KIND = "attendance".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          scope = ::Attendance.where(citizen_id: person.citizen_ids).includes(:health_unit, called_by_user: :professional)
          Window.apply(scope, kind: KIND, column: "attendances.checked_in_at", id_column: "attendances.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |attendance| entry(attendance) }
        end

        def entry(attendance)
          unit = attendance.health_unit
          caller = attendance.called_by_user
          Entry.new(kind: KIND, source_id: attendance.id, occurred_at: attendance.checked_in_at,
                    unit: { id: unit.id, name: unit.name },
                    professional: caller && { name: Screenings::Json.staff_name(caller), cbo_code: nil },
                    title: "Atendimento — #{unit.name}", summary: Labels::ATTENDANCE_OUTCOMES[attendance.outcome],
                    status: attendance.status, detail_ref: { type: "attendance", id: attendance.id },
                    citizen: { title: "Atendimento na #{unit.name}", summary: Labels::CITIZEN_OUTCOMES[attendance.outcome],
                               status: attendance.status, professional_name: Labels.professional_name(caller) },
                    document: nil)
        end
      end
    end
  end
  ```

  `app/services/timeline/sources/screening_source.rb`:

  ```ruby
  # Escuta inicial (módulo 18): só a concluída. Profissional: cor, destino e
  # CIAP-2 com nome. Cidadão: "Acolhimento realizado". Queixa, orientação e
  # motivo da troca de cor (texto livre) nunca entram.
  module Timeline
    module Sources
      module ScreeningSource
        KIND = "screening".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          scope = ::Screening.joins(:attendance).where(attendances: { citizen_id: person.citizen_ids }, status: "completed")
                             .includes(:current_revision, attendance: :health_unit, professional_link: :professional)
          Window.apply(scope, kind: KIND, column: "screenings.completed_at", id_column: "screenings.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |screening| entry(screening) }
        end

        def entry(screening)
          revision = screening.current_revision
          unit = screening.attendance.health_unit
          name = screening.professional_link.professional.professional_name
          ciap = "#{revision.ciap2_code} #{Screenings::Ciap2.label(revision.ciap2_code, revision.ciap2_release_id)}".strip
          Entry.new(kind: KIND, source_id: screening.id, occurred_at: screening.completed_at,
                    unit: { id: unit.id, name: unit.name }, professional: { name: name, cbo_code: screening.cbo_code },
                    title: "Escuta inicial — #{Labels::COLORS.fetch(revision.final_color, revision.final_color)}",
                    summary: "#{Labels::DESTINATIONS.fetch(screening.destination, screening.destination)} · CIAP-2 #{ciap}",
                    status: screening.status, detail_ref: { type: "screening", id: screening.id },
                    citizen: { title: "Acolhimento realizado", summary: nil, status: nil, professional_name: name },
                    document: nil)
        end
      end
    end
  end
  ```

  `app/services/timeline/sources/appointment_source.rb`:

  ```ruby
  # Agendamentos (módulos 08/17): data, tipo e situação; o horário movido sai
  # (vale o novo). O detalhe é o do pedido (a tela existente lê o pedido). Motivo de cancelamento e de encaixe (texto livre) nunca entram.
  module Timeline
    module Sources
      module AppointmentSource
        KIND = "appointment".freeze
        LEGACY_TITLE = "Consulta agendada".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          catalog = Scheduling::AppointmentTypes.catalog
          scope = ::Appointment.where(citizen_id: person.citizen_ids).where.not(status: "moved").includes(:health_unit, :professional)
          Window.apply(scope, kind: KIND, column: "appointments.scheduled_at", id_column: "appointments.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |appointment| entry(appointment, catalog) }
        end

        def entry(appointment, catalog)
          type = appointment.appointment_type_key && catalog.name_for(appointment.appointment_type_key)
          title = "Agendamento — #{type.presence || LEGACY_TITLE}"
          unit = appointment.health_unit
          name = appointment.professional&.professional_name
          Entry.new(kind: KIND, source_id: appointment.id, occurred_at: appointment.scheduled_at,
                    unit: { id: unit.id, name: unit.name }, professional: name && { name: name, cbo_code: nil },
                    title: title, summary: nil, status: appointment.status,
                    detail_ref: { type: "appointment_request", id: appointment.request_id },
                    citizen: { title: title, summary: nil, status: appointment.status, professional_name: name },
                    document: nil)
        end
      end
    end
  end
  ```

- [ ] **Step 4: Run** `rspec spec/services/timeline/sources/administrative_sources_spec.rb`. Expected: PASS. Se `completed_web_triage_for` deixar a triagem `in_progress` com o protocolo padrão, troque a asserção do exemplo de triagem por uma triagem concluída do mesmo helper com o protocolo de uma pergunta (`create_default_protocol!`), sem afrouxar o "sem as em andamento" (acrescente uma em andamento e espere que não apareça).

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/timeline/sources/window.rb app/services/timeline/sources/labels.rb app/services/timeline/sources/triage_source.rb app/services/timeline/sources/attendance_source.rb app/services/timeline/sources/screening_source.rb app/services/timeline/sources/appointment_source.rb spec/services/timeline/sources/administrative_sources_spec.rb
  git commit -m "feat: add administrative timeline sources with cursor window

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 6: Fornecedores clínicos e de campanha (`consultation`, `clinical_document`, `controlled_notification`, `campaign`)

**Files:**
- Create: `app/services/timeline/sources/consultation_source.rb`, `clinical_document_source.rb`, `controlled_notification_source.rb`, `campaign_source.rb`
- Test: `spec/services/timeline/sources/clinical_sources_spec.rb`

**Interfaces:**
- Consumes: `Timeline::Sources::{Window,Labels}` (Task 5), `Timeline::Entry` (Task 3).
- Produces: `Timeline::Sources::{ConsultationSource,ClinicalDocumentSource,ControlledNotificationSource,CampaignSource}` com `KIND` e `entries(person:, before:, limit:, from:, to:)`; `ClinicalDocumentSource.document_block(document) -> { id:, short_code:, verification_url:, downloadable: }`; `ClinicalDocumentSource.downloadable?(document) -> Boolean`.

- [ ] **Step 1: Write the failing test** `spec/services/timeline/sources/clinical_sources_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035 (spec §4, §9): consulta só finalizada e só da pessoa com paciente;
  # documentos com PDF e código para o cidadão; o registro da Notificação sem
  # número para o cidadão; campanha com convite e leitura. Nunca SOAP, CID sem
  # nome nem número de talão/SNCR na visão do cidadão.
  RSpec.describe "Fornecedores clínicos da linha do tempo" do
    before do
      timeline_city!(documents: true)
      ciap2_release!; cid10_release!; sigtap_release!
    end

    let(:unit) { create_unit("UBS Centro") }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(61) }
    let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen) }
    let(:person) { Timeline::Person.for_cpf(citizen.cpf) }

    def entries(source) = source.entries(person: person, before: nil, limit: 50, from: nil, to: nil)

    it "consultation: problemas avaliados com nome, condutas e adendos; cidadão vê 'consulta com <profissional>'" do
      consultation
      entry = entries(Timeline::Sources::ConsultationSource).sole
      expect([ entry.source_id, entry.occurred_at ]).to eq([ consultation.id, consultation.finalized_at ])
      expect(entry.summary).to include("T90 Diabetes não insulino-dependente", "condutas: 1", "adendos: 0")
      expect(entry.professional).to eq(name: doctor.professional.professional_name, cbo_code: consultation.cbo_code)
      expect(entry.citizen).to eq(title: "Consulta com #{doctor.professional.professional_name}", summary: nil, status: nil,
                                  professional_name: doctor.professional.professional_name)
      expect(entry.to_h.to_s).not_to include("Refere sede", "Metformina")
    end

    it "consultation: rascunho não entra; pessoa sem paciente não tem consulta" do
      started_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      expect(entries(Timeline::Sources::ConsultationSource)).to be_empty
      expect(Timeline::Sources::ConsultationSource.entries(person: Timeline::Person.for_cpf(verified_citizen!(62).cpf),
                                                           before: nil, limit: 50, from: nil, to: nil)).to eq([])
    end

    it "clinical_document: tipo, modo e situação; bloco do documento para o cidadão" do
      document = raw_document!(consultation: consultation, kind: "sick_note", issue_mode: "paper")
      entry = entries(Timeline::Sources::ClinicalDocumentSource).sole
      expect([ entry.title, entry.summary, entry.status ]).to eq([ "Atestado", "Papel · Emitido", "issued" ])
      expect(entry.document).to eq(id: document.id, short_code: document.short_code,
                                   verification_url: document.verification_url, downloadable: true)
      expect(entry.citizen).to include(title: "Atestado", status: "issued")
      document.update_columns(status: "cancelled", cancelled_at: Time.current, cancel_reason: "emitido por engano",
                              cancelled_by_user_id: doctor.id)
      expect(entries(Timeline::Sources::ClinicalDocumentSource).sole.document[:downloadable]).to be(false)
    end

    it "clinical_document digital sem assinatura não é baixável" do
      raw_document!(consultation: consultation, kind: "sick_note", issue_mode: "digital")
      expect(entries(Timeline::Sources::ClinicalDocumentSource).sole.document[:downloadable]).to be(false)
    end

    it "controlled_notification: profissional vê tipo, item e talão; cidadão não vê o talão" do
      content = { "notification_type" => "B", "paper_number" => "PR-1234567", "numbering_uf" => "PR",
                  "item" => { "printed_description" => "DIAZEPAM 10 mg COMPRIMIDO" } }
      raw_document!(consultation: consultation, kind: "controlled_notification_record", issue_mode: "paper", content: content)
      entry = entries(Timeline::Sources::ControlledNotificationSource).sole
      expect(entry.title).to eq("Notificação de receita B (papel)")
      expect(entry.summary).to eq("DIAZEPAM 10 mg COMPRIMIDO · talão PR-1234567")
      expect(entry.citizen).to eq(title: "Receita de controle especial (talão)", summary: nil, status: "issued",
                                  professional_name: doctor.professional.professional_name)
      expect(entry.document).to be_nil
      expect(entries(Timeline::Sources::ClinicalDocumentSource)).to be_empty
    end

    it "campaign: convite e leitura do aviso" do
      campaign = sent_campaign!(title: "Vacinação contra a gripe")
      recipient = recipient!(campaign, citizen)
      entry = entries(Timeline::Sources::CampaignSource).sole
      expect([ entry.title, entry.summary, entry.status ]).to eq([ "Campanha — Vacinação contra a gripe", "Aviso não lido", "unread" ])
      recipient.update_columns(notice_read_at: Time.zone.parse("2026-10-01 09:00"))
      expect(entries(Timeline::Sources::CampaignSource).sole.citizen).to include(summary: "Aviso lido em 01/10/2026", status: "read")
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/timeline/sources/clinical_sources_spec.rb`. Expected: FAIL (`uninitialized constant Timeline::Sources::ConsultationSource`).

- [ ] **Step 3: Implemente.** `app/services/timeline/sources/consultation_source.rb`:

  ```ruby
  # Consultas (19a): só finalizadas, do paciente da pessoa. Profissional:
  # problemas avaliados (código e nome da release gravada), quantas condutas e
  # quantos adendos. Cidadão: "Consulta com <profissional>". SOAP nunca entra.
  module Timeline
    module Sources
      module ConsultationSource
        KIND = "consultation".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          return [] unless person.patient_id

          scope = ::Consultation.finalized_consultations.where(patient_id: person.patient_id)
                                .includes(:problem_items, :conducts, :addenda, attendance: :health_unit, author_user: :professional)
          Window.apply(scope, kind: KIND, column: "consultations.finalized_at", id_column: "consultations.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |consultation| entry(consultation) }
        end

        def entry(consultation)
          unit = consultation.attendance.health_unit
          name = Labels.professional_name(consultation.author_user)
          problems = consultation.problem_items.select { |p| p.addendum_id.nil? }.sort_by { |p| [ p.created_at, p.id ] }.map do |p|
            "#{p.code} #{ClinicalTerms.label(p.terminology, p.code, p.terminology_release_id)}".strip
          end
          conducts = consultation.conducts.count { |c| c.addendum_id.nil? }
          summary = [ problems.join("; ").presence, "condutas: #{conducts}", "adendos: #{consultation.addenda.size}" ].compact.join(" · ")
          Entry.new(kind: KIND, source_id: consultation.id, occurred_at: consultation.finalized_at,
                    unit: { id: unit.id, name: unit.name },
                    professional: { name: Screenings::Json.staff_name(consultation.author_user), cbo_code: consultation.cbo_code },
                    title: "Consulta", summary: summary, status: consultation.status,
                    detail_ref: { type: "consultation", id: consultation.id },
                    citizen: { title: name ? "Consulta com #{name}" : "Consulta", summary: nil, status: nil, professional_name: name },
                    document: nil)
        end
      end
    end
  end
  ```

  `app/services/timeline/sources/clinical_document_source.rb`:

  ```ruby
  # Documentos clínicos (19c/19d), menos o registro da Notificação (kind
  # próprio). Profissional: tipo/categoria, modo e situação. Cidadão: tipo,
  # data, situação, código e conferência; baixável quando não cancelado e
  # (papel, ou digital já assinado). Conteúdo do documento nunca entra.
  module Timeline
    module Sources
      module ClinicalDocumentSource
        KIND = "clinical_document".freeze
        NOTIFICATION = "controlled_notification_record".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          scope = ::ClinicalDocument.where(citizen_id: person.citizen_ids).where.not(kind: NOTIFICATION)
                                    .includes(attendance: :health_unit, author_user: :professional)
          Window.apply(scope, kind: KIND, column: "clinical_documents.issued_at", id_column: "clinical_documents.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |document| entry(document) }
        end

        def entry(document)
          unit = document.attendance.health_unit
          name = Labels.professional_name(document.author_user)
          title = title(document)
          Entry.new(kind: KIND, source_id: document.id, occurred_at: document.issued_at,
                    unit: { id: unit.id, name: unit.name },
                    professional: { name: Screenings::Json.staff_name(document.author_user), cbo_code: document.cbo_code },
                    title: title, summary: "#{Labels::MODES[document.issue_mode]} · #{Labels::DOCUMENT_STATUS[document.status]}",
                    status: document.status, detail_ref: { type: "clinical_document", id: document.id },
                    citizen: { title: title, summary: nil, status: document.status, professional_name: name },
                    document: document_block(document))
        end

        def title(document)
          category = document.prescription? ? document.content_data["category"] : nil
          Labels::CATEGORIES[category] || Labels::DOCUMENT_KINDS.fetch(document.kind, document.kind)
        end

        def document_block(document)
          { id: document.id, short_code: document.short_code, verification_url: document.verification_url,
            downloadable: downloadable?(document) }
        end

        def downloadable?(document)
          return false if document.cancelled?

          document.digital? ? document.signed_signature.present? : true
        end
      end
    end
  end
  ```

  `app/services/timeline/sources/controlled_notification_source.rb`:

  ```ruby
  # Registro da Notificação de Receita de papel (19d). Profissional: tipo,
  # item e número do talão. Cidadão: "Receita de controle especial (talão)",
  # sem número, sem PDF (o registro não tem impresso nem conferência).
  module Timeline
    module Sources
      module ControlledNotificationSource
        KIND = "controlled_notification".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          scope = ::ClinicalDocument.where(citizen_id: person.citizen_ids, kind: ClinicalDocumentSource::NOTIFICATION)
                                    .includes(attendance: :health_unit, author_user: :professional)
          Window.apply(scope, kind: KIND, column: "clinical_documents.issued_at", id_column: "clinical_documents.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |document| entry(document) }
        end

        def entry(document)
          content = document.content_data
          unit = document.attendance.health_unit
          item = content.dig("item", "printed_description")
          Entry.new(kind: KIND, source_id: document.id, occurred_at: document.issued_at,
                    unit: { id: unit.id, name: unit.name },
                    professional: { name: Screenings::Json.staff_name(document.author_user), cbo_code: document.cbo_code },
                    title: "Notificação de receita #{content['notification_type']} (papel)",
                    summary: [ item, "talão #{content['paper_number']}" ].compact.join(" · "),
                    status: document.status, detail_ref: { type: "clinical_document", id: document.id },
                    citizen: { title: "Receita de controle especial (talão)", summary: nil, status: document.status,
                               professional_name: Labels.professional_name(document.author_user) },
                    document: nil)
        end
      end
    end
  end
  ```

  `app/services/timeline/sources/campaign_source.rb`:

  ```ruby
  # Campanhas (módulo 12): o convite (a linha do público congelado) e a
  # participação, que é a leitura do aviso no wpda. O corpo da campanha não entra.
  module Timeline
    module Sources
      module CampaignSource
        KIND = "campaign".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          scope = ::CampaignRecipient.where(citizen_id: person.citizen_ids).includes(:campaign)
          Window.apply(scope, kind: KIND, column: "campaign_recipients.created_at", id_column: "campaign_recipients.id",
                              before: before, from: from, to: to)
                .limit(limit).map { |recipient| entry(recipient) }
        end

        def entry(recipient)
          read_at = recipient.notice_read_at
          summary = read_at ? "Aviso lido em #{read_at.in_time_zone.strftime('%d/%m/%Y')}" : "Aviso não lido"
          status = read_at ? "read" : "unread"
          title = "Campanha — #{recipient.campaign.title}"
          Entry.new(kind: KIND, source_id: recipient.id, occurred_at: recipient.created_at, unit: nil, professional: nil,
                    title: title, summary: summary, status: status, detail_ref: { type: "campaign", id: recipient.campaign_id },
                    citizen: { title: title, summary: summary, status: status, professional_name: nil },
                    document: nil)
        end
      end
    end
  end
  ```

- [ ] **Step 4: Run** `rspec spec/services/timeline/sources`. Expected: PASS. Se `raw_document!` recusar o registro da Notificação por CHECK (o 19d exige `paper` e `consultation`), confira a linha do 19d em `spec/support/controlled_prescription_helpers.rb` e use o helper de lá no lugar — sem mudar as asserções.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/timeline/sources/consultation_source.rb app/services/timeline/sources/clinical_document_source.rb app/services/timeline/sources/controlled_notification_source.rb app/services/timeline/sources/campaign_source.rb spec/services/timeline/sources/clinical_sources_spec.rb
  git commit -m "feat: add clinical and campaign timeline sources

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 7: Junção, paginação, formas JSON e trilha

**Files:**
- Create: `app/services/timeline/query.rb`, `app/services/timeline/json.rb`, `app/services/timeline/trail.rb`
- Test: `spec/services/timeline/query_spec.rb`

**Interfaces:**
- Consumes: os 8 fornecedores (Tasks 5–6), `Timeline::Cursor` (Task 3).
- Produces:
  - `Timeline::Query::SOURCES` (kind → fornecedor), `Timeline::Query::DEFAULT_LIMIT = 50`, `MAX_LIMIT = 100`, `Timeline::Query.limit(raw) -> Integer`, `Timeline::Query.call(person:, kinds: Timeline::KINDS, before: nil, limit: 50, from: nil, to: nil) -> Timeline::Query::Page(entries:, next_before:)`.
  - `Timeline::Query.citizen_kinds(verified:, documents:) -> Array<String>`.
  - `Timeline::Json.professional(entry) -> Hash` (contrato §2 `<entry>`), `Timeline::Json.citizen(entry) -> Hash` (`<citizen_entry>`).
  - `Timeline::Trail.viewed!(person:, access:, user: nil, reason_code: nil)` (`access` em `Timeline::Trail::ACCESS = %w[timeline citizen pec_external]`).

- [ ] **Step 1: Write the failing test** `spec/services/timeline/query_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035 (spec §4; contrato §2–§3): junta os fornecedores, ordena por
  # occurred_at DESC, kind, id; corta em limit; next_before é o cursor da
  # última entrada quando há mais. Review Focus 1.
  RSpec.describe Timeline::Query do
    include ActiveSupport::Testing::TimeHelpers
    before do
      timeline_city!(record_mode: "integrated", clinical_record: false)
      ensure_appointment_types!
    end

    let(:unit) { create_unit }
    let(:citizen) { verified_citizen!(81) }
    let(:person) { Timeline::Person.for_cpf(citizen.cpf) }
    let(:at) { Time.zone.parse("2026-09-15 08:00:00.123456") }
    # walk_in_attendance! cria também a triagem de origem (agora): o recorte de
    # kinds deixa o exemplo sobre os empates que ele monta.
    let(:kinds) { %w[attendance campaign] }

    def walk(n, at:) = Array.new(n) { walk_in_attendance!(unit, citizen: citizen, checked_in_at: at) }

    # campaign_recipients só aceita mudar a leitura e o SMS (trigger): o
    # instante vem do relógio, com microssegundos.
    def campaign_at(at)
      travel_to(at, with_usec: true) { recipient!(sent_campaign!(title: "Campanha #{SecureRandom.hex(2)}"), citizen) }
    end

    def drain(**opts)
      seen = []
      before = nil
      loop do
        page = described_class.call(person: person, before: before, limit: 2, **{ kinds: kinds }.merge(opts))
        seen.concat(page.entries.map { |e| "#{e.kind}:#{e.source_id}" })
        break unless page.next_before

        before = Timeline::Cursor.decode(page.next_before)
      end
      seen
    end

    it "percorre tudo com empates de horário entre kinds e no mesmo kind, sem repetir nem perder (Review Focus 1)" do
      walk(3, at: at)
      2.times { campaign_at(at) }
      walk(1, at: at - 1.day)
      campaign_at(at + 1.day)
      all = described_class.call(person: person, kinds: kinds, limit: 100).entries
      expect(all.size).to eq(7)
      expect(all.map(&:sort_key)).to eq(all.map(&:sort_key).sort)
      expect(drain).to eq(all.map { |e| "#{e.kind}:#{e.source_id}" })
      expect(drain(kinds: %w[attendance]).size).to eq(4)
      expect(drain(from: at.beginning_of_day, to: at.end_of_day).size).to eq(5)
    end

    it "limit corta e só a última página tem next_before nulo" do
      walk(3, at: at)
      first = described_class.call(person: person, kinds: kinds, limit: 2)
      expect(first.entries.size).to eq(2)
      expect(first.next_before).to be_present
      last = described_class.call(person: person, kinds: kinds, limit: 2, before: Timeline::Cursor.decode(first.next_before))
      expect([ last.entries.size, last.next_before ]).to eq([ 1, nil ])
    end

    it "limit: padrão 50, teto 100, lixo vira padrão" do
      expect([ nil, "", "abc", "0", "-3", 7, "7", "150" ].map { |raw| described_class.limit(raw) }).to eq([ 50, 50, 50, 50, 50, 7, 7, 100 ])
    end

    it "kinds do cidadão: declarado só triage e appointment; documentos só com o interruptor" do
      expect(described_class.citizen_kinds(verified: false, documents: true)).to eq(%w[triage appointment])
      expect(described_class.citizen_kinds(verified: true, documents: false)).not_to include("clinical_document", "controlled_notification")
      expect(described_class.citizen_kinds(verified: true, documents: true)).to eq(Timeline::KINDS)
    end

    it "formas JSON do profissional e do cidadão" do
      walk(1, at: at)
      entry = described_class.call(person: person, kinds: %w[attendance]).entries.sole
      expect(Timeline::Json.professional(entry).keys)
        .to eq(%i[id kind occurred_at unit professional title summary status detail_ref])
      expect(Timeline::Json.professional(entry)[:id]).to eq("attendance:#{entry.source_id}")
      expect(Timeline::Json.citizen(entry).keys).to eq(%i[id kind occurred_at unit professional_name title summary status document])
      expect(Timeline::Json.citizen(entry)[:document]).to be_nil
      expect(Timeline::Json.citizen(entry)[:unit]).to eq(name: unit.name)
    end

    it "trilha só com ids, acesso da lista e person_ref" do
      Timeline::Trail.viewed!(person: person, access: "citizen")
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload)
        .to eq("patient_id" => nil, "user_id" => nil, "access" => "citizen", "reason_code" => nil,
               "person_ref" => "citizen:#{citizen.id}")
      expect { Timeline::Trail.viewed!(person: person, access: "outro") }.to raise_error(ArgumentError)
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/timeline/query_spec.rb`. Expected: FAIL (`uninitialized constant Timeline::Query`).

- [ ] **Step 3: Implemente.** `app/services/timeline/query.rb`:

  ```ruby
  # Junção da linha do tempo (ADR 0035; spec §4): cada fornecedor devolve até
  # limit + 1 entradas depois do cursor; aqui junta, ordena pela chave total e
  # corta. Se sobrou alguma, há próxima página e next_before é o cursor da
  # última entrada mostrada.
  module Timeline
    module Query
      DEFAULT_LIMIT = 50
      MAX_LIMIT = 100
      DOCUMENT_KINDS = %w[clinical_document controlled_notification].freeze
      DECLARED_KINDS = %w[triage appointment].freeze
      SOURCES = {
        "triage" => Sources::TriageSource, "attendance" => Sources::AttendanceSource,
        "screening" => Sources::ScreeningSource, "appointment" => Sources::AppointmentSource,
        "consultation" => Sources::ConsultationSource, "clinical_document" => Sources::ClinicalDocumentSource,
        "controlled_notification" => Sources::ControlledNotificationSource, "campaign" => Sources::CampaignSource
      }.freeze

      Page = Data.define(:entries, :next_before)

      module_function

      def call(person:, kinds: Timeline::KINDS, before: nil, limit: DEFAULT_LIMIT, from: nil, to: nil)
        rows = kinds.flat_map do |kind|
          SOURCES.fetch(kind).entries(person: person, before: before, limit: limit + 1, from: from, to: to)
        end
        rows.sort_by!(&:sort_key)
        page = rows.first(limit)
        Page.new(entries: page, next_before: rows.size > limit ? Cursor.encode(page.last) : nil)
      end

      def limit(raw)
        value = raw.is_a?(Integer) ? raw : (raw.is_a?(String) && raw.match?(/\A\d+\z/) ? raw.to_i : nil)
        return DEFAULT_LIMIT if value.nil? || value < 1

        [ value, MAX_LIMIT ].min
      end

      def citizen_kinds(verified:, documents:)
        return DECLARED_KINDS.dup unless verified

        documents ? Timeline::KINDS.dup : Timeline::KINDS - DOCUMENT_KINDS
      end
    end
  end
  ```

  `app/services/timeline/json.rb`:

  ```ruby
  # Formas do contrato §2: <entry> do profissional e <citizen_entry> do cidadão
  # (`document` sempre presente, null fora dos documentos — pedido W4 do wpda).
  module Timeline
    module Json
      module_function

      def professional(entry)
        { id: "#{entry.kind}:#{entry.source_id}", kind: entry.kind, occurred_at: entry.occurred_at.iso8601,
          unit: entry.unit, professional: entry.professional, title: entry.title, summary: entry.summary,
          status: entry.status, detail_ref: entry.detail_ref }
      end

      def citizen(entry)
        view = entry.citizen
        { id: "#{entry.kind}:#{entry.source_id}", kind: entry.kind, occurred_at: entry.occurred_at.iso8601,
          unit: entry.unit && { name: entry.unit[:name] }, professional_name: view[:professional_name],
          title: view[:title], summary: view[:summary], status: view[:status], document: entry.document }
      end
    end
  end
  ```

  `app/services/timeline/trail.rb`:

  ```ruby
  # Trilha de toda leitura da linha do tempo (ADR 0035, Invariantes): o mesmo
  # clinical_record.viewed do 19a, com access timeline (profissional), citizen
  # (wpda) ou pec_external (página do PEC) e a referência da pessoa. Só ids.
  module Timeline
    module Trail
      ACCESS = %w[timeline citizen pec_external].freeze

      module_function

      def viewed!(person:, access:, user: nil, reason_code: nil)
        raise ArgumentError, "acesso fora da lista: #{access}" unless ACCESS.include?(access)

        DomainEvents.publish("clinical_record.viewed", patient_id: person.patient_id, user_id: user&.id, access: access,
                                                       reason_code: reason_code, person_ref: person.ref)
      end
    end
  end
  ```

- [ ] **Step 4: Run** `rspec spec/services/timeline`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/timeline/query.rb app/services/timeline/json.rb app/services/timeline/trail.rb spec/services/timeline/query_spec.rb
  git commit -m "feat: merge and paginate timeline sources with json shapes and trail

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 4 — Rotas do profissional e do cidadão

### Task 8: `GET /attendance/timeline` (profissional)

**Files:**
- Create: `app/controllers/timelines_controller.rb`
- Modify: `config/routes.rb` (escopo `/attendance`)
- Test: `spec/requests/timeline_spec.rb`

**Interfaces:**
- Consumes: `Timeline::{Gate,Person,Cursor,Query,Json,Trail}` (Tasks 1, 3, 7), `ClinicalRecord::Access.for_person` (Task 4), concerns `Authentication`, `AttendanceAccess`, `FeatureGate`, `ReportPeriod`.
- Produces: `GET /attendance/timeline?patient_id=|attendance_id=|opening_id=&before=&limit=&kinds[]=&from=&to=` → `{ person: { display_name, partial, patient_id }, entries: [<entry>], next_before }` (contrato §3, com as Divergências D4).

- [ ] **Step 1: Write the failing test** `spec/requests/timeline_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3 (ADR 0035): a linha do tempo do profissional com as regras de
  # leitura do 19a sobre a pessoa, trilha `timeline`, filtros e cursor.
  RSpec.describe "Linha do tempo do profissional", type: :request do
    include ActiveSupport::Testing::TimeHelpers

    let(:unit) { create_unit("UBS Centro") }
    let(:doctor) { doctor!(unit) }
    let(:citizen) { verified_citizen!(101, social_name: "Bia") }
    def body = JSON.parse(response.body)
    def status_and_error = [ response.status, body["error"] ]

    context "modo record" do
      before { timeline_city!; ciap2_release!; cid10_release!; sigtap_release! }

      it "em contexto pelo atendimento: entradas, pessoa, trilha timeline" do
        consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
        attendance = consulting_attendance!(unit, citizen: citizen.reload, doctor: doctor)
        sign_in_as(doctor)
        get "/attendance/timeline", params: { attendance_id: attendance.id }
        expect(response).to have_http_status(:ok)
        expect(body["person"]).to eq("display_name" => "Bia", "partial" => false, "patient_id" => consultation.patient_id)
        expect(body["entries"].map { |e| e["kind"] }).to include("consultation", "attendance", "triage")
        expect(body["entries"].first.keys).to eq(%w[id kind occurred_at unit professional title summary status detail_ref])
        expect(response.body).not_to include("Refere sede", "Metformina")
        expect(DomainEvent.where(name: "clinical_record.viewed").last.payload)
          .to eq("patient_id" => consultation.patient_id, "user_id" => doctor.id, "access" => "timeline", "reason_code" => nil,
                 "person_ref" => "patient:#{consultation.patient_id}")
      end

      it "fora de contexto: 403; com abertura válida (opening_id ou patient_id), justificada com o motivo na trilha" do
        consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
        other = doctor!(unit, cbo: "225142")
        sign_in_as(other)
        get "/attendance/timeline", params: { patient_id: consultation.patient_id }
        expect(status_and_error).to eq([ 403, "out_of_context" ])
        now = Time.current
        opening = ClinicalRecordOpening.create!(patient: consultation.patient, user: other, reason_code: "case_review",
                                                created_at: now, expires_at: now + 30.minutes)
        get "/attendance/timeline", params: { opening_id: opening.id }
        expect(response).to have_http_status(:ok)
        expect(DomainEvent.where(name: "clinical_record.viewed").last.payload["reason_code"]).to eq("case_review")
        get "/attendance/timeline", params: { patient_id: consultation.patient_id }
        expect(response).to have_http_status(:ok)
        sign_in_as(doctor!(unit, cbo: "223505"))
        get "/attendance/timeline", params: { opening_id: opening.id }
        expect(status_and_error).to eq([ 403, "opening_required" ])
        travel_to(31.minutes.from_now) do
          sign_in_as(other)
          get "/attendance/timeline", params: { opening_id: opening.id }
          expect(status_and_error).to eq([ 403, "opening_required" ])
        end
      end

      it "atendimento de par declarado: 409; sem alvo ou alvo inexistente: 404; recepção: 403 missing_role" do
        declared = consulting_attendance!(unit, citizen: screening_citizen!(102), doctor: doctor)
        sign_in_as(doctor)
        get "/attendance/timeline", params: { attendance_id: declared.id }
        expect(status_and_error).to eq([ 409, "citizen_not_verified" ])
        get "/attendance/timeline"
        expect(status_and_error).to eq([ 404, "not_found" ])
        get "/attendance/timeline", params: { patient_id: SecureRandom.uuid }
        expect(status_and_error).to eq([ 404, "not_found" ])
        get "/attendance/timeline", params: { attendance_id: "nao-e-uuid" }
        expect(status_and_error).to eq([ 404, "not_found" ])
        sign_in_as(reception!)
        get "/attendance/timeline", params: { attendance_id: declared.id }
        expect(status_and_error).to eq([ 403, "missing_role" ])
      end

      it "filtros, limite e cursor; erros de período, kinds e cursor" do
        attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
        sign_in_as(doctor)
        get "/attendance/timeline", params: { attendance_id: attendance.id, kinds: %w[attendance], limit: 1 }
        expect(body["entries"].map { |e| e["kind"] }).to eq(%w[attendance])
        expect(body["next_before"]).to be_nil
        get "/attendance/timeline", params: { attendance_id: attendance.id, limit: 1 }
        expect(body["next_before"]).to be_present
        get "/attendance/timeline", params: { attendance_id: attendance.id, limit: 1, before: body["next_before"] }
        expect(response).to have_http_status(:ok)
        get "/attendance/timeline", params: { attendance_id: attendance.id, from: Time.zone.tomorrow.iso8601 }
        expect(body["entries"]).to eq([])
        [ [ { from: "ontem" }, "invalid_period" ], [ { from: Time.zone.today.iso8601, to: Time.zone.yesterday.iso8601 }, "invalid_period" ],
          [ { kinds: %w[exam] }, "invalid_kinds" ], [ { kinds: "attendance" }, "invalid_kinds" ], [ { before: "lixo" }, "invalid_cursor" ] ]
          .each do |extra, error|
            get "/attendance/timeline", params: { attendance_id: attendance.id }.merge(extra)
            expect(status_and_error).to eq([ 422, error ]), extra.inspect
          end
      end
    end

    context "modo integrado" do
      before { timeline_city!(record_mode: "integrated", clinical_record: false) }

      it "pessoa sem paciente, histórico parcial; interruptor desligado: 403 feature_disabled" do
        attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
        sign_in_as(doctor)
        get "/attendance/timeline", params: { attendance_id: attendance.id }
        expect(body["person"]).to eq("display_name" => "Bia", "partial" => true, "patient_id" => nil)
        expect(DomainEvent.where(name: "clinical_record.viewed").last.payload["person_ref"]).to eq("citizen:#{citizen.id}")
        Platform::Features.set!(city: Current.city, key: "patient_timeline", enabled: false, maintainer: ledi_maintainer!)
        get "/attendance/timeline", params: { attendance_id: attendance.id }
        expect([ response.status, body["error"], body["feature"] ]).to eq([ 403, "feature_disabled", "patient_timeline" ])
      end
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/requests/timeline_spec.rb`. Expected: FAIL (404 de rota).

- [ ] **Step 3: Implemente.** Em `config/routes.rb`, no escopo `/attendance`, depois de `get "attendances/:id/record"`:

  ```ruby
      # Linha do tempo do paciente (ADR 0035; contrato §3).
      get   "timeline",                        to: "timelines#show"
  ```

  `app/controllers/timelines_controller.rb`:

  ```ruby
  # Linha do tempo do profissional (ADR 0035; spec §5–§6; contrato §3): a pessoa
  # pelo atendimento (em contexto), pelo paciente ou pela abertura justificada;
  # as regras de leitura do 19a sobre a pessoa (ClinicalRecord::Access.for_person);
  # toda leitura deixa trilha `timeline`. Sem texto clínico livre: o detalhe
  # abre as telas existentes, com as regras delas.
  class TimelinesController < ApplicationController
    include Authentication
    include AttendanceAccess
    include FeatureGate
    include ReportPeriod

    require_feature :patient_timeline
    before_action :require_professional

    def show
      from, to = period
      return render_invalid_period if invalid_period?(from, to) || (from && to && from > to)

      kinds = requested_kinds
      return render(json: { error: "invalid_kinds" }, status: :unprocessable_entity) unless kinds

      before = Timeline::Cursor.decode(params[:before])
      person, grant = target
      return if performed?

      page = Timeline::Query.call(person: person, kinds: kinds, before: before, limit: Timeline::Query.limit(params[:limit]),
                                  from: from, to: to)
      Timeline::Trail.viewed!(person: person, access: "timeline", user: Current.user, reason_code: grant.opening&.reason_code)
      render json: { person: { display_name: person.display_name, partial: Timeline::Gate.partial?(Current.city),
                               patient_id: person.patient_id },
                     entries: page.entries.map { |entry| Timeline::Json.professional(entry) }, next_before: page.next_before }
    rescue Timeline::Cursor::Invalid
      render json: { error: "invalid_cursor" }, status: :unprocessable_entity
    end

    private

    # kinds[] ausente = todos; qualquer valor fora da lista (ou kinds escalar) = nil.
    def requested_kinds
      raw = params[:kinds]
      return Timeline::KINDS if raw.blank?
      return nil unless raw.is_a?(Array) && raw.all? { |kind| Timeline::KINDS.include?(kind) }

      Timeline::KINDS & raw
    end

    # [pessoa, grant]; senão a resposta de erro já saiu.
    def target
      if params[:opening_id].present? then by_opening
      elsif params[:patient_id].present? then by_patient
      elsif params[:attendance_id].present? then by_attendance
      else not_found
      end
    end

    def by_opening
      id = params[:opening_id]
      opening = uuid?(id) && ClinicalRecordOpening.where(user_id: Current.user.id).where("expires_at > ?", Time.current).find_by(id: id)
      return forbid("opening_required") unless opening

      person = opening.patient ? Timeline::Person.for_patient(opening.patient) : Timeline::Person.for_cpf(opening.citizen.cpf)
      [ person, ClinicalRecord::Access::Grant.new(kind: :justified, opening: opening, reason: nil) ]
    end

    def by_patient
      patient = uuid?(params[:patient_id]) && Patient.find_by(id: params[:patient_id])
      return not_found unless patient

      person = Timeline::Person.for_patient(patient)
      granted(person, ClinicalRecord::Access.for_person(user: Current.user, person: person))
    end

    def by_attendance
      attendance = uuid?(params[:attendance_id]) && Attendance.find_by(id: params[:attendance_id])
      return not_found unless attendance
      return render(json: { error: "citizen_not_verified" }, status: :conflict) unless attendance.citizen.verification_level_verified?

      person = Timeline::Person.for_citizen(attendance.citizen)
      granted(person, ClinicalRecord::Access.for_person(user: Current.user, person: person, attendance: attendance))
    end

    def granted(person, grant)
      return forbid(grant.reason.to_s) unless grant.allowed?

      [ person, grant ]
    end

    def uuid?(value) = value.is_a?(String) && ClinicalDocuments::Uuid.valid?(value)
    def not_found = render(json: { error: "not_found" }, status: :not_found)
  end
  ```

- [ ] **Step 4: Run** `rspec spec/requests/timeline_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/controllers/timelines_controller.rb config/routes.rb spec/requests/timeline_spec.rb
  git commit -m "feat: add professional patient timeline endpoint

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 9: `GET /citizen/history` e `GET /citizen/documents/:id/pdf` (cidadão)

**Files:**
- Create: `app/controllers/citizen_api/history_controller.rb`, `app/services/timeline/citizen_documents.rb`
- Modify: `config/routes.rb` (escopo `/citizen`), `app/controllers/citizen_api/sessions_controller.rb` (`features`)
- Test: `spec/requests/citizen_api/history_spec.rb`, `spec/requests/citizen_api/session_features_spec.rb`

**Interfaces:**
- Consumes: `Timeline::{Person,Query,Json,Trail,Gate,Cursor}`, `ClinicalDocuments::{Gate,Pdf,Uuid}`, `CitizenApi::BaseController`.
- Produces: `GET /citizen/history?citizen_id=&before=&limit=&kinds[]=` → `{ verified, partial, entries: [<citizen_entry>], next_before }` (`kinds[]` recorta os kinds que a pessoa pode ver; fora da lista dos 8 → 422 `invalid_kinds`; `limit` 1–100, pedido W3); `GET`/`POST /citizen/session` ganham `features: string[]` (interruptores utilizáveis da cidade, pedido W2); `GET /citizen/documents/:id/pdf?citizen_id=` → `application/pdf`; `Timeline::CitizenDocuments.find(person:, id:) -> ClinicalDocument|nil`, `Timeline::CitizenDocuments.pdf(document) -> String|nil`; `CitizenApi::HistoryController#own_citizen` (usado pela Task 10).

- [ ] **Step 1: Write the failing test** `spec/requests/citizen_api/history_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4 (ADR 0035): o histórico do cidadão. Par validado vê a pessoa
  # inteira (sem texto clínico livre); par declarado vê só triagens e
  # agendamentos dele. Review Focus 3: a família no mesmo celular.
  RSpec.describe "Histórico do cidadão", type: :request do
    before { timeline_city!(documents: true); ciap2_release!; cid10_release!; sigtap_release! }

    let(:unit) { create_unit("UBS Centro") }
    let(:doctor) { doctor!(unit) }
    let(:mother) { verified_citizen!(111, social_name: "Dona Lu") }
    let!(:child) { Citizen.create!(cpf: "39053344705", phone: mother.phone, verification_level: "declared") }
    let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: mother) }
    def body = JSON.parse(response.body)
    def status_and_error = [ response.status, body["error"] ]

    it "sem sessão: 401; interruptor desligado: 403 feature_disabled" do
      get "/citizen/history", params: { citizen_id: mother.id }
      expect(status_and_error).to eq([ 401, "unauthenticated" ])
      sign_in_citizen(mother.phone)
      Platform::Features.set!(city: Current.city, key: "patient_timeline", enabled: false, maintainer: ledi_maintainer!)
      get "/citizen/history", params: { citizen_id: mother.id }
      expect([ response.status, body["error"], body["feature"] ]).to eq([ 403, "feature_disabled", "patient_timeline" ])
    end

    it "par validado: a pessoa inteira, documento com PDF e código, trilha citizen, sem texto livre" do
      document = raw_document!(consultation: consultation, kind: "sick_note", issue_mode: "paper")
      other_phone_pair = Citizen.create!(cpf: mother.cpf, phone: "+5541988880001", verification_level: "verified",
                                         patient_id: consultation.patient_id)
      completed_web_triage_for(other_phone_pair)
      sign_in_citizen(mother.phone)
      get "/citizen/history", params: { citizen_id: mother.id }
      expect(response).to have_http_status(:ok)
      expect(body.slice("verified", "partial")).to eq("verified" => true, "partial" => false)
      kinds = body["entries"].map { |e| e["kind"] }
      expect(kinds).to include("consultation", "clinical_document", "attendance", "triage")
      expect(kinds.count("triage")).to eq(3)
      doc = body["entries"].find { |e| e["kind"] == "clinical_document" }
      expect(doc["document"]).to eq("id" => document.id, "short_code" => document.short_code,
                                    "verification_url" => document.verification_url, "downloadable" => true)
      expect(body["entries"].first.keys).to include("professional_name")
      expect(body["entries"].first.keys).not_to include("detail_ref", "professional")
      expect(response.body).not_to include("Refere sede", "Metformina", "T90", "Diabetes")
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload)
        .to include("access" => "citizen", "user_id" => nil, "person_ref" => "patient:#{consultation.patient_id}")
    end

    it "família no mesmo celular: o declarado vê só triagens e agendamentos dele; de outro telefone, 404 (Review Focus 3)" do
      consultation
      completed_web_triage_for(child)
      stranger = verified_citizen!(112)
      sign_in_citizen(mother.phone)
      get "/citizen/history", params: { citizen_id: child.id }
      expect(body["verified"]).to be(false)
      expect(body["entries"].map { |e| e["kind"] }.uniq).to eq(%w[triage])
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload["person_ref"]).to eq("citizen:#{child.id}")
      get "/citizen/history", params: { citizen_id: stranger.id }
      expect(status_and_error).to eq([ 404, "not_found" ])
      get "/citizen/history"
      expect(status_and_error).to eq([ 404, "not_found" ])
    end

    it "PDF: só documento da própria pessoa, não cancelado e baixável" do
      paper = raw_document!(consultation: consultation, kind: "sick_note", issue_mode: "paper")
      unsigned = raw_document!(consultation: consultation, kind: "sick_note", issue_mode: "digital")
      strangers = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(113))
      foreign = raw_document!(consultation: strangers, kind: "sick_note", issue_mode: "paper")
      sign_in_citizen(mother.phone)
      get "/citizen/documents/#{paper.id}/pdf", params: { citizen_id: mother.id }
      expect(response).to have_http_status(:ok)
      expect(response.media_type).to eq("application/pdf")
      expect(response.headers["Cache-Control"]).to include("no-store")
      [ unsigned.id, foreign.id, SecureRandom.uuid, "nao-e-uuid" ].each do |id|
        get "/citizen/documents/#{id}/pdf", params: { citizen_id: mother.id }
        expect(status_and_error).to eq([ 404, "not_found" ]), id
      end
      paper.update_columns(status: "cancelled", cancelled_at: Time.current, cancel_reason: "emitido por engano",
                           cancelled_by_user_id: doctor.id)
      get "/citizen/documents/#{paper.id}/pdf", params: { citizen_id: mother.id }
      expect(status_and_error).to eq([ 404, "not_found" ])
      get "/citizen/documents/#{paper.id}/pdf", params: { citizen_id: child.id }
      expect(status_and_error).to eq([ 403, "not_verified" ])
    end

    it "cursor e kinds inválidos: 422; kinds recorta só dentro do que a pessoa pode ver" do
      consultation
      completed_web_triage_for(child)
      sign_in_citizen(mother.phone)
      get "/citizen/history", params: { citizen_id: mother.id, before: "lixo" }
      expect(status_and_error).to eq([ 422, "invalid_cursor" ])
      get "/citizen/history", params: { citizen_id: mother.id, kinds: %w[exam] }
      expect(status_and_error).to eq([ 422, "invalid_kinds" ])
      get "/citizen/history", params: { citizen_id: mother.id, kinds: %w[consultation], limit: 1 }
      expect(body["entries"].map { |e| e["kind"] }).to eq(%w[consultation])
      get "/citizen/history", params: { citizen_id: child.id, kinds: %w[consultation triage] }
      expect(body["entries"].map { |e| e["kind"] }.uniq).to eq(%w[triage])
    end
  end
  ```

  (Se o banco recusar `Citizen.create!` de par `verified` com `patient_id` sem a validação, troque `other_phone_pair` por `verified_citizen!(114)` com o CPF da mãe — o helper aceita só `n`; nesse caso crie o par pelo caminho do helper e `update!(cpf: mother.cpf)` antes de validar. A asserção — três triagens da pessoa, inclusive a do outro telefone — não muda.)

  `spec/requests/citizen_api/session_features_spec.rb` (pedido W2 do wpda):

  ```ruby
  require "rails_helper"

  RSpec.describe "Sessão do cidadão com os interruptores", type: :request do
    it "GET /citizen/session devolve os interruptores utilizáveis da cidade" do
      timeline_city!(record_mode: "integrated", clinical_record: false)
      Platform::Features.set!(city: Current.city, key: "pec_external_view", enabled: true, maintainer: ledi_maintainer!)
      sign_in_citizen
      get "/citizen/session"
      features = JSON.parse(response.body)["features"]
      expect(features).to include("patient_timeline")
      expect(features).not_to include("pec_external_view", "clinical_record")
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/requests/citizen_api/history_spec.rb spec/requests/citizen_api/session_features_spec.rb`. Expected: FAIL (404 de rota; `features` ausente).

- [ ] **Step 3: Implemente.** Em `config/routes.rb`, no escopo `/citizen`, depois de `put "contact_preferences/:citizen_id"`:

  ```ruby
      # Linha do tempo do cidadão (ADR 0035; contrato §4, Divergência D2: citizen_id).
      get "history",           to: "history#index"
      get "health_summary",    to: "history#health_summary"
      get "documents/:id/pdf", to: "history#document_pdf"
  ```

  `app/services/timeline/citizen_documents.rb`:

  ```ruby
  # Documento da própria pessoa para o cidadão (ADR 0035; contrato §4): nunca
  # o registro da Notificação; o PDF assinado (PAdES gravado) do digital já
  # assinado, ou o de papel gerado agora. Digital pendente e cancelado: nil.
  module Timeline
    module CitizenDocuments
      module_function

      def find(person:, id:)
        return nil unless id.is_a?(String) && ClinicalDocuments::Uuid.valid?(id)

        ClinicalDocument.where(citizen_id: person.citizen_ids)
                        .where.not(kind: Sources::ClinicalDocumentSource::NOTIFICATION).find_by(id: id)
      end

      def pdf(document)
        return nil if document.cancelled?

        document.digital? ? document.signed_signature&.signed_pdf_bytes : ClinicalDocuments::Pdf.call(document)
      end
    end
  end
  ```

  `app/controllers/citizen_api/history_controller.rb`:

  ```ruby
  #   GET /citizen/history?citizen_id=&before=&limit=
  #   GET /citizen/health_summary?citizen_id=          (Task 10)
  #   GET /citizen/documents/:id/pdf?citizen_id=
  # Linha do tempo do cidadão (ADR 0035; contrato §4). Só pares do celular da
  # sessão (senão 404). Par validado: a pessoa inteira, sem texto clínico
  # livre; par declarado: triagens e agendamentos do próprio par. Toda leitura
  # deixa trilha `citizen`.
  module CitizenApi
    class HistoryController < BaseController
      include FeatureGate

      require_feature :patient_timeline
      rate_limit to: 30, within: 10.minutes, only: :document_pdf, name: "citizen_document_pdf",
                 by: -> { current_citizen_session&.id || request.remote_ip }, store: RateLimitStore,
                 with: -> { render_error("too_many_requests", :too_many_requests) }

      def index
        citizen = own_citizen
        return unless citizen

        requested = requested_kinds
        return render_error("invalid_kinds", :unprocessable_entity) unless requested

        person = Timeline::Person.for_citizen(citizen)
        kinds = Timeline::Query.citizen_kinds(verified: person.verified, documents: ClinicalDocuments::Gate.usable?(Current.city)) & requested
        page = Timeline::Query.call(person: person, kinds: kinds, before: Timeline::Cursor.decode(params[:before]),
                                    limit: Timeline::Query.limit(params[:limit]))
        Timeline::Trail.viewed!(person: person, access: "citizen")
        render json: { verified: person.verified, partial: Timeline::Gate.partial?(Current.city),
                       entries: page.entries.map { |entry| Timeline::Json.citizen(entry) }, next_before: page.next_before }
      rescue Timeline::Cursor::Invalid
        render_error("invalid_cursor", :unprocessable_entity)
      end

      def document_pdf
        citizen = own_citizen
        return unless citizen
        return render_error("not_verified", :forbidden) unless citizen.verification_level_verified?

        person = Timeline::Person.for_citizen(citizen)
        document = ClinicalDocuments::Gate.usable?(Current.city) ? Timeline::CitizenDocuments.find(person: person, id: params[:id]) : nil
        bytes = document && Timeline::CitizenDocuments.pdf(document)
        return render_error("not_found", :not_found) unless bytes

        Timeline::Trail.viewed!(person: person, access: "citizen")
        response.headers["Cache-Control"] = "no-store"
        send_data bytes, type: "application/pdf", disposition: "attachment", filename: "documento-#{document.short_code}.pdf"
      end

      private

      # kinds[] ausente = todos (W3); fora da lista dos 8, ou escalar, = nil.
      def requested_kinds
        raw = params[:kinds]
        return Timeline::KINDS if raw.blank?

        raw.is_a?(Array) && raw.all? { |kind| Timeline::KINDS.include?(kind) } ? raw : nil
      end

      def own_citizen
        citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
        render_error("not_found", :not_found) unless citizen
        citizen
      end
    end
  end
  ```

  Em `app/controllers/citizen_api/sessions_controller.rb`, `session_json` ganha a lista (o `create` e o `show` usam o mesmo método; uma leitura da cidade só, por `summary`):

  ```ruby
      # ADR 0035 (pedido W2): os interruptores UTILIZÁVEIS da cidade, para o
      # wpda mostrar "Meu histórico" só quando existe.
      def session_json(session)
        features = Platform::Features.summary(Current.city).select { |row| row[:usable] }.map { |row| row[:key] }
        { phone_masked: CitizenIdentity::Phone.mask(session.phone), time_zone: Current.city.time_zone, features: features }
      end
  ```

  Se alguma spec de `/citizen/session` fixa as chaves exatas da resposta, acrescente `features` (é o contrato novo).

- [ ] **Step 4: Run** `rspec spec/requests/citizen_api/history_spec.rb spec/requests/citizen_api`. Expected: PASS (as rotas antigas do `/citizen` seguem verdes).

- [ ] **Step 5: Commit**

  ```bash
  git add app/controllers/citizen_api/history_controller.rb app/controllers/citizen_api/sessions_controller.rb app/services/timeline/citizen_documents.rb config/routes.rb spec/requests/citizen_api/history_spec.rb spec/requests/citizen_api/session_features_spec.rb
  git commit -m "feat: add citizen history timeline and own document pdf

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 10: Resumo clínico do cidadão (`GET /citizen/health_summary`)

**Files:**
- Create: `app/services/timeline/citizen_summary.rb`
- Modify: `app/controllers/citizen_api/history_controller.rb` (ação `health_summary`)
- Test: `spec/services/timeline/citizen_summary_spec.rb`, `spec/requests/citizen_api/health_summary_spec.rb`

**Interfaces:**
- Consumes: `Timeline::Person`, `ClinicalRecord::Gate.usable?`, `ClinicalTerms.label`, `ClinicalTerms::SigtapExams.label`, `HistoryController#own_citizen` (Task 9).
- Produces: `Timeline::CitizenSummary.call(patient:, person:, now: Time.current) -> Hash` na forma do contrato §4 (`problems`, `medications`, `exam_requests`, `vitals`); `GET /citizen/health_summary?citizen_id=`.

- [ ] **Step 1: Write the failing tests.** `spec/services/timeline/citizen_summary_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §4 (ADR 0035): problemas ativos (nome), medicamentos em uso, exames
  # solicitados nos últimos 12 meses e a última PA, peso e glicemia com data.
  # Nunca CID sem nome; nunca texto da consulta.
  RSpec.describe Timeline::CitizenSummary do
    include ActiveSupport::Testing::TimeHelpers
    before { timeline_city!(documents: true); ciap2_release!; cid10_release!; sigtap_release!; acolhimento! }

    let(:unit) { create_unit }
    let(:doctor) { doctor!(unit) }
    let(:nurse) { screener!(unit) }
    let(:citizen) { verified_citizen!(121) }
    let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen) }
    let(:patient) { consultation.patient }
    let(:person) { Timeline::Person.for_patient(patient) }

    it "monta as quatro partes a partir da consulta, da lista de medicamentos e da escuta" do
      ApplicationRecord.transaction do
        Patients::ApplyMedicationEvent.call(patient: patient, action: "add", by: doctor, source: { consultation: consultation },
                                            origin: "external", free_text: "Losartana 50 mg", dosage_summary: "1 comprimido pela manhã")
      end
      travel_to(1.hour.from_now) do
        attendance = walk_in_attendance!(unit, citizen: citizen)
        started = Screenings::Start.call(attendance: attendance, by: nurse).payload[:screening]
        Screenings::Complete.call(screening: started,
                                  revision_params: revision_params(vitals: { "systolic" => 128, "diastolic" => 82,
                                                                             "capillary_glucose" => 110, "glucose_moment" => "fasting" }),
                                  destination: "oriented", destination_params: { "orientation_note" => "retorno se piorar" }, by: nurse)
      end
      summary = described_class.call(patient: patient, person: person, now: 2.hours.from_now)
      expect(summary[:problems]).to eq([ { terminology: "ciap2", code: "T90", label: "Diabetes não insulino-dependente",
                                           since: "2025-08-01" } ])
      expect(summary[:medications]).to eq([ { label: "Losartana 50 mg", dosage_summary: "1 comprimido pela manhã", since: Time.zone.today.iso8601 } ])
      expect(summary[:exam_requests].map { |e| e.slice(:sigtap_code, :label) })
        .to eq([ { sigtap_code: "0202010503", label: "DOSAGEM DE HEMOGLOBINA GLICOSILADA" } ])
      expect(summary[:vitals][:blood_pressure]).to include(systolic: 128, diastolic: 82)
      expect(summary[:vitals][:weight_kg]).to eq(value: 82.5, measured_at: consultation.finalized_at.iso8601)
      expect(summary[:vitals][:capillary_glucose]).to include(value: 110, moment: "fasting")
      expect(summary.to_s).not_to include("Refere sede", "retorno se piorar")
    end

    it "problema sem nome não aparece; exame de mais de 12 meses não aparece" do
      allow(ClinicalTerms).to receive(:label).and_return(nil)
      expect(described_class.call(patient: patient, person: person)[:problems]).to eq([])
      expect(described_class.call(patient: patient, person: person, now: 13.months.from_now)[:exam_requests]).to eq([])
    end
  end
  ```

  `spec/requests/citizen_api/health_summary_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4: 403 not_verified para par declarado; 404 not_available sem
  # paciente ou sem prontuário; trilha citizen.
  RSpec.describe "Resumo clínico do cidadão", type: :request do
    let(:unit) { create_unit }
    let(:doctor) { doctor!(unit) }
    def body = JSON.parse(response.body)
    def status_and_error = [ response.status, body["error"] ]

    it "validado com paciente: o resumo; declarado: 403; sem paciente: 404 not_available" do
      timeline_city!
      ciap2_release!; cid10_release!; sigtap_release!
      citizen = verified_citizen!(131)
      finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
      declared = Citizen.create!(cpf: "11144477735", phone: citizen.phone, verification_level: "declared")
      fresh = verified_citizen!(132)
      sign_in_citizen(citizen.phone)
      get "/citizen/health_summary", params: { citizen_id: citizen.id }
      expect(response).to have_http_status(:ok)
      expect(body.keys).to match_array(%w[problems medications exam_requests vitals])
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload["access"]).to eq("citizen")
      get "/citizen/health_summary", params: { citizen_id: declared.id }
      expect(status_and_error).to eq([ 403, "not_verified" ])
      sign_in_citizen(fresh.phone)
      get "/citizen/health_summary", params: { citizen_id: fresh.id }
      expect(status_and_error).to eq([ 404, "not_available" ])
    end

    it "modo integrado (sem prontuário): 404 not_available" do
      timeline_city!(record_mode: "integrated", clinical_record: false)
      citizen = verified_citizen!(133)
      sign_in_citizen(citizen.phone)
      get "/citizen/health_summary", params: { citizen_id: citizen.id }
      expect(status_and_error).to eq([ 404, "not_available" ])
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/timeline/citizen_summary_spec.rb spec/requests/citizen_api/health_summary_spec.rb`. Expected: FAIL (`uninitialized constant Timeline::CitizenSummary`).

- [ ] **Step 3: Implemente.** `app/services/timeline/citizen_summary.rb`:

  ```ruby
  # Resumo clínico do cidadão (ADR 0035; spec §4; contrato §4). Só com
  # paciente e prontuário utilizável (quem chama confere). Problema sem nome
  # na release gravada não aparece (nunca CID sem nome). Exame: por consulta e
  # código vale a linha mais recente (o adendo cancela com uma linha
  # `cancelled`); entram as `requested` dos últimos 12 meses. Sinais: a
  # aferição mais recente entre consultas finalizadas e escutas concluídas.
  module Timeline
    module CitizenSummary
      EXAM_WINDOW = 12.months

      module_function

      def call(patient:, person:, now: Time.current)
        { problems: problems(patient), medications: medications(patient), exam_requests: exam_requests(patient, now),
          vitals: vitals(patient, person) }
      end

      def problems(patient)
        patient.problems.where(status: "active").order(:terminology, :code).filter_map do |problem|
          label = ClinicalTerms.label(problem.terminology, problem.code, problem.terminology_release_id)
          label && { terminology: problem.terminology, code: problem.code, label: label, since: problem.onset_on&.iso8601 }
        end
      end

      def medications(patient)
        PatientMedication.where(patient_id: patient.id, status: "active").order(:created_at, :id).map do |medication|
          { label: medication.label, dosage_summary: medication.dosage_summary, since: medication.started_on&.iso8601 }
        end
      end

      def exam_requests(patient, now)
        rows = ConsultationExamRequest.joins(:consultation)
                                      .where(consultations: { patient_id: patient.id, status: "finalized" })
                                      .where("consultation_exam_requests.created_at >= ?", now - EXAM_WINDOW)
                                      .order(:created_at, :id).to_a
        latest = rows.group_by { |row| [ row.consultation_id, row.sigtap_code ] }.values.map(&:last)
        latest.select { |row| row.status == "requested" }.sort_by { |row| [ -row.created_at.to_r, row.id ] }.map do |row|
          { sigtap_code: row.sigtap_code,
            label: ClinicalTerms::SigtapExams.label(row.sigtap_code, row.sigtap_competence) || "Procedimento #{row.sigtap_code}",
            requested_at: row.created_at.iso8601 }
        end
      end

      def vitals(patient, person)
        out = {}
        bp, at = latest(patient, person, :systolic)
        out[:blood_pressure] = { systolic: bp.systolic, diastolic: bp.diastolic, measured_at: at.iso8601 } if bp
        weight, at = latest(patient, person, :weight_kg)
        out[:weight_kg] = { value: weight.weight_kg.to_f, measured_at: at.iso8601 } if weight
        glucose, at = latest(patient, person, :capillary_glucose)
        out[:capillary_glucose] = { value: glucose.capillary_glucose, moment: glucose.glucose_moment, measured_at: at.iso8601 } if glucose
        out
      end

      # [registro, instante] da aferição mais recente da coluna, ou nil.
      def latest(patient, person, column)
        consultation = Consultation.finalized_consultations.where(patient_id: patient.id).where.not(column => nil)
                                   .order(finalized_at: :desc, id: :asc).first
        revision = ScreeningRevision.joins(screening: :attendance)
                                    .where(screenings: { status: "completed" }, attendances: { citizen_id: person.citizen_ids })
                                    .where("screenings.current_revision_id = screening_revisions.id")
                                    .where.not(column => nil).order("screenings.completed_at DESC").includes(:screening).first
        [ consultation && [ consultation, consultation.finalized_at ], revision && [ revision, revision.screening.completed_at ] ]
          .compact.max_by { |_, at| at }
      end
      private_class_method :latest
    end
  end
  ```

  Em `app/controllers/citizen_api/history_controller.rb`, acrescente a ação (depois de `index`):

  ```ruby
      def health_summary
        citizen = own_citizen
        return unless citizen
        return render_error("not_verified", :forbidden) unless citizen.verification_level_verified?

        person = Timeline::Person.for_citizen(citizen)
        patient = person.patient_id && ClinicalRecord::Gate.usable?(Current.city) ? Patient.find_by(id: person.patient_id) : nil
        return render_error("not_available", :not_found) unless patient

        Timeline::Trail.viewed!(person: person, access: "citizen")
        render json: Timeline::CitizenSummary.call(patient: patient, person: person)
      end
  ```

  Confira no código o nome do atributo de início em `PatientMedication` (`started_on`, no dump) e a forma como `Patients::ApplyMedicationEvent` grava o rótulo do texto livre (o `label` cifrado); se o rótulo for outro texto (normalizado), ajuste o esperado do exemplo para o que o comando grava.

- [ ] **Step 4: Run** `rspec spec/services/timeline/citizen_summary_spec.rb spec/requests/citizen_api/health_summary_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/timeline/citizen_summary.rb app/controllers/citizen_api/history_controller.rb spec/services/timeline/citizen_summary_spec.rb spec/requests/citizen_api/health_summary_spec.rb
  git commit -m "feat: add citizen clinical health summary

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 5 — PEC: configuração e página

### Task 11: Configuração do PEC no maintenance (GraphQL)

**Files:**
- Create: `app/commands/pec_external/configure.rb`
- Create: `app/graphql/maintenance/mutations/set_city_pec_origin.rb`, `app/graphql/maintenance/mutations/rotate_city_pec_hmac_key.rb`, `app/graphql/maintenance/types/city_pec_config_type.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/mutations/city_mutation.rb` (`AUDITED_FIELD_RULES`), `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`
- Modify: `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`
- Test: `spec/commands/pec_external/configure_spec.rb`, `spec/requests/maintenance/city_pec_config_spec.rb`

**Interfaces:**
- Consumes: `PecSetting` (Task 2), `Maintenance::Mutations::CityMutation#in_city`, `Maintenance::CityReader`.
- Produces: `PecExternal::Configure.set_origin(origin:) -> Result`, `.rotate_key(key:) -> Result` (falhas `:invalid_origin`, `:invalid_key`), `.normalize_origin(raw) -> String|nil`, `.audit!(city_slug:, field:, maintainer_id:)`, `PecExternal::Configure::KEY_FORMAT`; GraphQL `cityPecConfig(citySlug) { pecOrigin hmacKeyConfigured hmacKeyUpdatedAt }`, `setCityPecOrigin(citySlug, pecOrigin) { ok errors }`, `rotateCityPecHmacKey(citySlug, key, code) { ok errors }` (com step-up TOTP, decisão M4).

- [ ] **Step 1: Write the failing tests.** `spec/commands/pec_external/configure_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe PecExternal::Configure do
    it "normaliza a origem: esquema e host minúsculos, sem porta padrão; recusa caminho, query, usuário" do
      expect(described_class.normalize_origin("HTTPS://PEC.Cidade.gov.br")).to eq("https://pec.cidade.gov.br")
      expect(described_class.normalize_origin("https://pec.cidade.gov.br:443/")).to eq("https://pec.cidade.gov.br")
      expect(described_class.normalize_origin("https://pec.cidade.gov.br:8443")).to eq("https://pec.cidade.gov.br:8443")
      expect(described_class.normalize_origin("http://pec-sim.localhost:3040")).to eq("http://pec-sim.localhost:3040")
      [ "https://pec.cidade.gov.br/esus", "https://pec.cidade.gov.br?x=1", "https://u:p@pec.cidade.gov.br", "ftp://pec",
        "pec.cidade.gov.br", "", "https://", 42, "https://#{'a' * 260}.br" ].each do |raw|
        expect(described_class.normalize_origin(raw)).to be_nil, raw.inspect
      end
      allow(Rota).to receive(:deployed?).and_return(true)
      expect(described_class.normalize_origin("http://pec.cidade.gov.br")).to be_nil
    end

    it "grava a origem e troca a chave sem nunca devolvê-la" do
      expect(described_class.set_origin(origin: "https://pec.cidade.gov.br")).to be_success
      expect(described_class.set_origin(origin: "https://pec.cidade.gov.br/x").reason).to eq(:invalid_origin)
      expect(described_class.rotate_key(key: "curta").reason).to eq(:invalid_key)
      expect(described_class.rotate_key(key: "com espaço no meio 0123456789").reason).to eq(:invalid_key)
      result = described_class.rotate_key(key: "chave-nova-0123456789abcdef")
      expect(result).to be_success
      expect(result.payload.to_s).not_to include("chave-nova")
      setting = PecSetting.current
      expect([ setting.pec_origin, setting.hmac_key, setting.hmac_key_configured? ])
        .to eq([ "https://pec.cidade.gov.br", "chave-nova-0123456789abcdef", true ])
      expect(described_class.set_origin(origin: nil)).to be_success
      expect(PecSetting.current.pec_origin).to be_nil
    end
  end
  ```

  `spec/requests/maintenance/city_pec_config_spec.rb` (mesmo arranjo de `spec/requests/maintenance/city_features_spec.rb`: `register_city!`, host `maintenance-api.rotasaude.app`, `login!`, `gql!`):

  ```ruby
  require "rails_helper"

  # Contrato §6: só sessão humana; a chave nunca volta, nem em resposta, nem
  # em log, nem em auditoria; as duas escritas auditam o campo.
  RSpec.describe "Configuração do PEC na API de manutenção", type: :request do
    include ActiveSupport::Testing::TimeHelpers
    let(:frontend) { "https://maintenance.rotasaude.app" }
    let(:password) { "s3nha-forte-1" }
    let!(:maintainer) do
      Maintainer.create!(email_address: "pec-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    end
    let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }

    def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
    def json = JSON.parse(response.body)
    def gql!(query, **variables) = post("/graphql", params: { query: query, variables: variables.to_json }, headers: browser)

    def login!
      post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
      post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
                                 headers: browser
    end

    # O passo do TOTP do login não vale de novo no step-up: copie o
    # `with_fresh_totp` de spec/requests/maintenance/protocol_mutations_spec.rb
    # (o código do passo seguinte ao do login, que roda um minuto atrás).

    before do
      unless City.exists?(slug: TEST_CITY_A.slug)
        City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active", database_url: TEST_CITY_A.database_url,
                     encryption_key: TEST_CITY_A.encryption_key, schema_version: CitySchema.expected_version.to_s)
      end
      host! "maintenance-api.rotasaude.app"
      allow(ENV).to receive(:[]).and_call_original
      allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
      travel_to(1.minute.ago) { login! }
    end
    after { CityCatalog.reset_cache! }

    it "grava origem e chave, lê sem a chave, audita o campo e não loga a chave" do
      gql!(<<~GQL, citySlug: city.slug, pecOrigin: "https://PEC.cidade.gov.br")
        mutation($citySlug: String!, $pecOrigin: String) { setCityPecOrigin(citySlug: $citySlug, pecOrigin: $pecOrigin) { ok errors { path message } } }
      GQL
      expect(json.dig("data", "setCityPecOrigin")).to eq("ok" => true, "errors" => [])
      gql!(<<~GQL, citySlug: city.slug, key: "chave-do-pec-MARCADOR-0123456789", code: "000000")
        mutation($citySlug: String!, $key: String!, $code: String!) { rotateCityPecHmacKey(citySlug: $citySlug, key: $key, code: $code) { ok errors { path message } } }
      GQL
      expect(json.dig("data", "rotateCityPecHmacKey", "ok")).to be(false)
      expect(PecSetting.current.hmac_key_configured?).to be(false)
      log = capture_log do
        with_fresh_totp do |code|
          gql!(<<~GQL, citySlug: city.slug, key: "chave-do-pec-MARCADOR-0123456789", code: code)
            mutation($citySlug: String!, $key: String!, $code: String!) { rotateCityPecHmacKey(citySlug: $citySlug, key: $key, code: $code) { ok errors { path message } } }
          GQL
        end
      end
      expect(json.dig("data", "rotateCityPecHmacKey", "ok")).to be(true)
      expect(log).not_to include("MARCADOR")
      gql!(<<~GQL, citySlug: city.slug)
        query($citySlug: String!) { cityPecConfig(citySlug: $citySlug) { pecOrigin hmacKeyConfigured hmacKeyUpdatedAt } }
      GQL
      expect(json.dig("data", "cityPecConfig")).to include("pecOrigin" => "https://pec.cidade.gov.br", "hmacKeyConfigured" => true)
      expect(response.body).not_to include("MARCADOR")
      expect(PlatformEvent.where(name: "city.pec_config_changed").order(:created_at).map { |e| e.payload["field"] })
        .to eq(%w[pec_origin hmac_key])
      expect(PlatformEvent.where(name: "maintenance.city.pec_config_changed").order(:created_at).pluck(Arel.sql("payload->>'outcome'")))
        .to eq(%w[attempted ok attempted rejected attempted ok])
      expect(PlatformEvent.where("payload::text LIKE ?", "%MARCADOR%")).to be_empty
    end

    it "valores inválidos: erro no campo, nada gravado, sem city.pec_config_changed" do
      gql!(<<~GQL, citySlug: city.slug, pecOrigin: "https://pec.cidade.gov.br/esus")
        mutation($citySlug: String!, $pecOrigin: String) { setCityPecOrigin(citySlug: $citySlug, pecOrigin: $pecOrigin) { ok errors { path message } } }
      GQL
      expect(json.dig("data", "setCityPecOrigin")).to eq("ok" => false, "errors" => [ { "path" => "pecOrigin", "message" => "invalid_origin" } ])
      with_fresh_totp do |code|
        gql!(<<~GQL, citySlug: city.slug, key: "curta", code: code)
          mutation($citySlug: String!, $key: String!, $code: String!) { rotateCityPecHmacKey(citySlug: $citySlug, key: $key, code: $code) { ok errors { path message } } }
        GQL
      end
      expect(json.dig("data", "rotateCityPecHmacKey")).to eq("ok" => false, "errors" => [ { "path" => "key", "message" => "invalid_key" } ])
      expect(PecSetting.current).to be_nil
      expect(PlatformEvent.where(name: "city.pec_config_changed")).to be_empty
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/commands/pec_external/configure_spec.rb spec/requests/maintenance/city_pec_config_spec.rb`. Expected: FAIL (`uninitialized constant PecExternal`).

- [ ] **Step 3: Implemente o comando** `app/commands/pec_external/configure.rb`:

  ```ruby
  # Configuração do PEC da cidade (ADR 0035; contrato §6), só pelo maintenance,
  # dentro da conexão da cidade (CityMutation#in_city). A chave HMAC é a do
  # cadastro do integrador no PEC (texto opaco; docs/pesquisa/2026-10-11-pec-
  # sistemas-externos.md); nunca volta em Result, log ou evento.
  module PecExternal
    module Configure
      KEY_FORMAT = /\A[\x21-\x7E]{16,512}\z/
      DEFAULT_PORTS = { "https" => 443, "http" => 80 }.freeze
      MAX_ORIGIN = 255

      module_function

      def set_origin(origin:)
        normalized = origin.nil? ? nil : normalize_origin(origin)
        return Result.fail(:invalid_origin) if !origin.nil? && normalized.nil?

        setting = PecSetting.lock.first || PecSetting.new
        setting.update!(pec_origin: normalized, pec_origin_updated_at: Time.current)
        Result.ok(pec_origin: normalized)
      end

      def rotate_key(key:)
        return Result.fail(:invalid_key) unless key.is_a?(String) && key.match?(KEY_FORMAT)

        setting = PecSetting.lock.first || PecSetting.new
        setting.update!(hmac_key: key, hmac_key_updated_at: Time.current)
        Result.ok(hmac_key_configured: true)
      end

      # scheme://host[:port], https sempre e http só fora de ambiente publicado.
      def normalize_origin(raw)
        return nil unless raw.is_a?(String) && raw.strip.length.between?(1, MAX_ORIGIN)

        uri = URI.parse(raw.strip)
        schemes = Rota.deployed? ? %w[https] : %w[https http]
        return nil unless schemes.include?(uri.scheme&.downcase) && uri.host.present?
        return nil unless uri.userinfo.nil? && uri.query.nil? && uri.fragment.nil? && [ "", "/" ].include?(uri.path.to_s)

        scheme = uri.scheme.downcase
        port = uri.port == DEFAULT_PORTS[scheme] ? "" : ":#{uri.port}"
        "#{scheme}://#{uri.host.downcase}#{port}"
      rescue URI::InvalidURIError
        nil
      end

      # Plataforma (contrato §7): o campo mudado, nunca o valor.
      def audit!(city_slug:, field:, maintainer_id:)
        Platform.audit("city.pec_config_changed", city_id: City.where(slug: city_slug).pick(:id), field: field,
                                                  maintainer_id: maintainer_id)
      end
    end
  end
  ```

- [ ] **Step 4: GraphQL.** `app/graphql/maintenance/types/city_pec_config_type.rb`:

  ```ruby
  module Maintenance
    module Types
      # ADR 0035 (contrato §6): a configuração do PEC sem a chave — só se há uma e quando mudou.
      class CityPecConfigType < BaseObject
        graphql_name "CityPecConfig"
        description "Endereço do PEC e estado da chave HMAC dos sistemas externos (a chave nunca sai)"

        field :pec_origin, String, null: true
        field :hmac_key_configured, Boolean, null: false
        field :hmac_key_updated_at, GraphQL::Types::ISO8601DateTime, null: true
      end
    end
  end
  ```

  Em `app/graphql/maintenance/types/query_type.rb`, depois de `field :medication_catalog ...`:

  ```ruby
        # ADR 0035 (contrato §6): abre a cidade; só sessão humana (HumanOnly).
        field :city_pec_config, Types::CityPecConfigType, null: true,
              description: "Configuração do PEC da cidade, sem a chave" do
          argument :city_slug, String, required: true
        end
  ```

  e o resolver (depois de `def city`):

  ```ruby
        def city_pec_config(city_slug:)
          unless context.fetch(:credential).allows_city?(city_slug)
            raise GraphQL::ExecutionError.new("cidade fora do escopo do token",
                                              extensions: { "code" => Analyzers::Refusal::CITY_OUT_OF_SCOPE,
                                                            Analyzers::Refusal::FIELDS => [ "cityPecConfig" ] })
          end
          city = City.find_by(slug: city_slug)
          return nil unless city

          CityReader.call(city) do
            setting = PecSetting.current
            { pec_origin: setting&.pec_origin, hmac_key_configured: setting&.hmac_key_configured? || false,
              hmac_key_updated_at: setting&.hmac_key_updated_at }
          end
        rescue CityReader::Archived => e
          raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_ARCHIVED" })
        rescue CityReader::Unreachable => e
          raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_UNREACHABLE" })
        rescue CityReader::Failed => e
          raise GraphQL::ExecutionError.new(e.message, extensions: { "code" => "CITY_READ_FAILED" })
        end
  ```

  `app/graphql/maintenance/mutations/set_city_pec_origin.rb`:

  ```ruby
  module Maintenance
    module Mutations
      # ADR 0035 (contrato §6): grava (ou apaga, com null) o endereço do PEC,
      # usado no frame-ancestors da página do PEC.
      class SetCityPecOrigin < CityMutation
        description "Grava o endereço (origem) do PEC da cidade. Nulo apaga."

        argument :pec_origin, String, required: false

        def resolve(city_slug:, pec_origin: nil)
          outcome = in_city(city_slug: city_slug, event: "maintenance.city.pec_config_changed", module_name: "city",
                            rejection_path: "pecOrigin", pec_field: "pec_origin") do |_actor, _correlation_id|
            PecExternal::Configure.set_origin(origin: pec_origin)
          end
          PecExternal::Configure.audit!(city_slug: city_slug, field: "pec_origin", maintainer_id: credential.maintainer.id) if outcome[:ok]
          outcome
        end
      end
    end
  end
  ```

  `app/graphql/maintenance/mutations/rotate_city_pec_hmac_key.rb`:

  ```ruby
  module Maintenance
    module Mutations
      # ADR 0035 (contrato §6): grava ou troca a chave HMAC do cadastro do
      # integrador no PEC. A chave nunca volta (payload só { ok, errors }) e
      # nunca entra na auditoria (só o campo). Step-up pela base (decisão M4).
      class RotateCityPecHmacKey < CityMutation
        description "Grava ou troca a chave HMAC dos sistemas externos do PEC da cidade. A chave nunca volta. " \
                    "Exige o TOTP do momento."

        argument :key, String, required: true
        argument :code, String, required: true, description: "TOTP do momento"

        def resolve(city_slug:, key:, code:)
          outcome = in_city(city_slug: city_slug, step_up_code: code, event: "maintenance.city.pec_config_changed",
                            module_name: "city", rejection_path: "key", pec_field: "hmac_key") do |_actor, _correlation_id|
            PecExternal::Configure.rotate_key(key: key)
          end
          PecExternal::Configure.audit!(city_slug: city_slug, field: "hmac_key", maintainer_id: credential.maintainer.id) if outcome[:ok]
          outcome
        end
      end
    end
  end
  ```

  Em `mutation_type.rb`: `field :set_city_pec_origin, mutation: Mutations::SetCityPecOrigin` e `field :rotate_city_pec_hmac_key, mutation: Mutations::RotateCityPecHmacKey`.
  Em `city_mutation.rb`, em `AUDITED_FIELD_RULES`:

  ```ruby
          # ADR 0035: qual campo da configuração do PEC mudou — nunca o valor.
          pec_field: lambda do |value|
            "pec_field inválido" unless %w[pec_origin hmac_key].include?(value)
          end,
  ```

  Em `human_only.rb`, `RESTRICTED` ganha `cityPecConfig setCityPecOrigin rotateCityPecHmacKey`.
  Em `app/events/maintenance_audit.rb`, `NAMES` ganha `maintenance.city.pec_config_changed` e o `case` ganha:

  ```ruby
      when "maintenance.city.pec_config_changed" then Platform.audit("maintenance.city.pec_config_changed", **payload)
  ```

  Em `spec/events/platform_event_payload_guard_spec.rb`, `R18_PLATFORM_EVENT_NAMES` ganha `city.pec_config_changed maintenance.city.pec_config_changed`.
  Em `spec/architecture/maintenance_schema_spec.rb`: `"Query"` ganha `cityPecConfig`; `"Mutation"` ganha `setCityPecOrigin rotateCityPecHmacKey`; tipos novos `"CityPecConfig" => %w[pecOrigin hmacKeyConfigured hmacKeyUpdatedAt]`, `"SetCityPecOriginPayload" => %w[ok errors]`, `"RotateCityPecHmacKeyPayload" => %w[ok errors]`; `ALLOWED_NAMES` ganha `RotateCityPecHmacKeyPayload` (nome de tipo revisado: a mutation escreve a chave, nunca a lê); `forbidden_name_exempt_fields` ganha `Mutation.rotateCityPecHmacKey Mutation.rotateCityPecHmacKey.key CityPecConfig.hmacKeyConfigured CityPecConfig.hmacKeyUpdatedAt` (com um comentário "ADR 0035: escreve a chave e diz SE há chave; o valor nunca sai").
  Em `spec/graphql/maintenance/analyzers_spec.rb`, `step_up_exempt` ganha `setCityPecOrigin` e `step_up_required` ganha `rotateCityPecHmacKey` (decisão M4: a troca da chave exige o TOTP do momento).

- [ ] **Step 5: Run** `rspec spec/commands/pec_external spec/requests/maintenance spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance spec/events/platform_event_payload_guard_spec.rb`. Expected: PASS.

- [ ] **Step 6: Commit**

  ```bash
  git add app/commands/pec_external/configure.rb app/graphql/maintenance/mutations/set_city_pec_origin.rb app/graphql/maintenance/mutations/rotate_city_pec_hmac_key.rb app/graphql/maintenance/types/city_pec_config_type.rb app/graphql/maintenance/types/query_type.rb app/graphql/maintenance/types/mutation_type.rb app/graphql/maintenance/mutations/city_mutation.rb app/graphql/maintenance/analyzers/human_only.rb app/events/maintenance_audit.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance/analyzers_spec.rb spec/commands/pec_external/configure_spec.rb spec/requests/maintenance/city_pec_config_spec.rb
  git commit -m "feat: add pec origin and hmac key configuration to maintenance graphql

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 12: Assinatura do PEC — HMAC em tempo constante, janela de 5 min e uso único

**Files:**
- Create: `app/services/pec_external/signature.rb`
- Test: `spec/services/pec_external/signature_spec.rb`

**Interfaces:**
- Consumes: `PecSignatureUse` (Task 2).
- Produces: `PecExternal::Signature::WINDOW = 300`, `PecExternal::Signature.verify(url:, key:, timestamp:, now: Time.current) -> :ok | :pec_signature_invalid | :pec_expired | :pec_replayed` (consome o uso só quando devolve `:ok`).

- [ ] **Step 1: Write the failing test** `spec/services/pec_external/signature_spec.rb`:

  ```ruby
  require "rails_helper"

  # Documentação oficial (docs/pesquisa/2026-10-11-pec-sistemas-externos.md):
  # HMAC-SHA256 hex da URL completa até antes de &pec_sign=. O vetor abaixo foi
  # calculado ao escrever o plano com
  #   printf '%s' '<URL sem pec_sign>' | openssl dgst -sha256 -hmac '<chave>'
  RSpec.describe PecExternal::Signature do
    KEY = "chave-de-teste-do-pec-0123456789abcdef".freeze
    UNSIGNED = "https://curitiba.rotasaude.app/pec/timeline?documento-profissional=52998224725" \
               "&documento-cidadao=11144477735&cnes=1234567&pec_timestamp=1791720000".freeze
    SIGN = "a96bfb4684cd099acce004db814a296a59854f21cd1b9d7f4cb589d59ef00f9b".freeze
    let(:at) { Time.at(1_791_720_000) }

    def verify(url, timestamp: "1791720000", now: at, key: KEY) = described_class.verify(url: url, key: key, timestamp: timestamp, now: now)

    it "confere o vetor independente; a mesma assinatura de novo é reenvio" do
      expect(verify("#{UNSIGNED}&pec_sign=#{SIGN}")).to eq(:ok)
      expect(verify("#{UNSIGNED}&pec_sign=#{SIGN}")).to eq(:pec_replayed)
      expect(PecSignatureUse.sole.signature_digest).to eq(Digest::SHA256.hexdigest(SIGN))
      expect(PecSignatureUse.sole.expires_at).to eq(at + 300)
    end

    it "URL mexida: ordem, parâmetro a mais, tamanho, chave errada; maiúsculas passam (Review Focus 2)" do
      reordered = UNSIGNED.sub("documento-profissional=52998224725&documento-cidadao=11144477735",
                               "documento-cidadao=11144477735&documento-profissional=52998224725")
      [ "#{reordered}&pec_sign=#{SIGN}", "#{UNSIGNED}&pec_sign=#{SIGN}&extra=1", "#{UNSIGNED}&pec_sign=#{SIGN[0, 63]}",
        "#{UNSIGNED}&pec_sign=", UNSIGNED, "#{UNSIGNED.sub('1234567', '7654321')}&pec_sign=#{SIGN}" ].each do |url|
        expect(verify(url)).to eq(:pec_signature_invalid), url
      end
      expect(verify("#{UNSIGNED}&pec_sign=#{SIGN}", key: "outra-chave-0123456789")).to eq(:pec_signature_invalid)
      expect(verify("#{UNSIGNED}&pec_sign=#{SIGN}", key: nil)).to eq(:pec_signature_invalid)
      expect(verify("#{UNSIGNED}&pec_sign=#{SIGN}", timestamp: "1791720001")).to eq(:pec_signature_invalid)
      expect(PecSignatureUse.count).to eq(0)
      expect(verify("#{UNSIGNED}&pec_sign=#{SIGN.upcase}")).to eq(:ok)
    end

    it "janela de 5 minutos para os dois lados" do
      url = "#{UNSIGNED}&pec_sign=#{SIGN}"
      expect(verify(url, now: at + 301)).to eq(:pec_expired)
      expect(verify(url, now: at - 301)).to eq(:pec_expired)
      expect(verify(url, now: at + 300)).to eq(:ok)
    end

    it "apaga usos vencidos antes de gravar" do
      PecSignatureUse.create!(signature_digest: "0" * 64, expires_at: at - 1, created_at: at - 600)
      verify("#{UNSIGNED}&pec_sign=#{SIGN}")
      expect(PecSignatureUse.pluck(:signature_digest)).to eq([ Digest::SHA256.hexdigest(SIGN) ])
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/pec_external/signature_spec.rb`. Expected: FAIL (`uninitialized constant PecExternal::Signature`).

- [ ] **Step 3: Implemente** `app/services/pec_external/signature.rb`:

  ```ruby
  require "openssl"

  # Assinatura da chamada dos "sistemas externos" do e-SUS PEC (ADR 0035;
  # docs/pesquisa/2026-10-11-pec-sistemas-externos.md): HMAC-SHA256 em hex da
  # URL completa até antes de &pec_sign=, com a chave do cadastro do
  # integrador. Compara em tempo constante; a janela é de 5 minutos para os dois
  # lados; cada assinatura vale uma vez (pec_signature_uses, com o SHA-256 dela).
  module PecExternal
    module Signature
      WINDOW = 300
      SIGN_PARAM = "&pec_sign=".freeze
      HEX = /\A\h{64}\z/
      TIMESTAMP = /\A\d{1,12}\z/

      module_function

      def verify(url:, key:, timestamp:, now: Time.current)
        unsigned, sign = url.to_s.split(SIGN_PARAM, 2)
        return :pec_signature_invalid unless key.is_a?(String) && key.present? && sign.to_s.match?(HEX)
        return :pec_signature_invalid unless timestamp.is_a?(String) && timestamp.match?(TIMESTAMP)
        return :pec_signature_invalid unless unsigned.end_with?("&pec_timestamp=#{timestamp}")

        expected = OpenSSL::HMAC.hexdigest("SHA256", key, unsigned)
        return :pec_signature_invalid unless ActiveSupport::SecurityUtils.secure_compare(expected, sign.downcase)
        return :pec_expired if (now.to_i - timestamp.to_i).abs > WINDOW

        consume!(sign.downcase, Time.zone.at(timestamp.to_i + WINDOW), now) ? :ok : :pec_replayed
      end

      # true se a assinatura ainda não tinha sido usada (INSERT ... ON CONFLICT DO NOTHING).
      def consume!(sign, expires_at, now)
        PecSignatureUse.where("expires_at < ?", now).delete_all
        PecSignatureUse.insert({ signature_digest: Digest::SHA256.hexdigest(sign), expires_at: expires_at, created_at: now },
                               unique_by: :signature_digest, returning: [ :id ]).rows.any?
      end
      private_class_method :consume!
    end
  end
  ```

- [ ] **Step 4: Run** `rspec spec/services/pec_external/signature_spec.rb`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/pec_external/signature.rb spec/services/pec_external/signature_spec.rb
  git commit -m "feat: verify pec external signature in constant time with replay guard

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 13: Ordem de conferência da página do PEC e sessão curta

**Files:**
- Create: `app/services/pec_external/authorize.rb`, `app/services/pec_external/view_session.rb`
- Test: `spec/services/pec_external/authorize_spec.rb`

**Interfaces:**
- Consumes: `PecExternal::Signature` (Task 12), `Timeline::{Gate,Person}`, `PecSetting`, `Consultations::Authorization.link_for`.
- Produces: `PecExternal::Authorize::Grant = Data.define(:user, :unit, :person)`; `PecExternal::Authorize.call(city:, url:, params:, now: Time.current) -> Result` (ok: `payload[:grant]`; falha: `reason` = um dos códigos do PEC) que publica `pec.external_view_opened`/`pec.external_view_denied`; `PecExternal::Authorize.linked?(user, unit) -> Boolean`; `PecExternal::ViewSession::HEADER = "X-Pec-View-Session"`, `ViewSession::TTL = 15.minutes`, `ViewSession.issue(city:, grant:) -> String`, `ViewSession.resume(token, city:) -> Grant|nil`.

- [ ] **Step 1: Write the failing test** `spec/services/pec_external/authorize_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §7 (ADR 0035): interruptor; assinatura; horário; uso único; CNES de
  # unidade ativa; profissional com vínculo ativo nessa unidade; pessoa com par
  # validado. Cada recusa publica pec.external_view_denied só com o código.
  RSpec.describe PecExternal::Authorize do
    include ActiveSupport::Testing::TimeHelpers
    let(:city) { timeline_city!(record_mode: "integrated", clinical_record: false, pec: true) }
    let(:unit) { create_unit("UBS Norte").tap { |u| u.update!(cnes: "1234567") } }
    let(:nurse) { screener!(unit).tap { |u| u.professional.update!(cpf: "52998224725") } }
    let(:citizen) { verified_citizen!(141) }

    before { city; pec_settings!; nurse }

    def call(professional: "52998224725", citizen_doc: citizen.cpf, cnes: "1234567", at: Time.current, key: TimelineHelpers::PEC_KEY)
      url = signed_pec_path(professional: professional, citizen: citizen_doc, cnes: cnes, at: at, key: key)
      params = Rack::Utils.parse_query(URI.parse(url).query)
      described_class.call(city: city, url: url, params: params)
    end

    def denied_codes = DomainEvent.where(name: "pec.external_view_denied").map { |e| e.payload["reason_code"] }

    it "válida: o profissional, a unidade e a pessoa; evento de abertura só com ids" do
      result = call
      expect(result).to be_success
      grant = result.payload[:grant]
      expect([ grant.user, grant.unit, grant.person.citizen_ids ]).to eq([ nurse, unit, [ citizen.id ] ])
      expect(DomainEvent.where(name: "pec.external_view_opened").last.payload)
        .to eq("user_id" => nurse.id, "unit_id" => unit.id, "patient_id" => nil)
    end

    it "profissional pelo CNS também vale" do
      expect(call(professional: nurse.professional.cns)).to be_success
    end

    it "cada recusa na ordem, com o código e o evento" do
      expect(call(key: "outra-chave-0123456789").reason).to eq(:pec_signature_invalid)
      expect(call(at: 6.minutes.ago).reason).to eq(:pec_expired)
      expect(call(cnes: "7654321").reason).to eq(:pec_unit_unknown)
      other_unit = create_unit("UBS Sul").tap { |u| u.update!(cnes: "7654321") }
      expect(call(cnes: other_unit.cnes).reason).to eq(:pec_professional_not_linked)
      expect(call(professional: "11144477735").reason).to eq(:pec_professional_not_linked)
      expect(call(citizen_doc: screening_citizen!(142).cpf).reason).to eq(:pec_citizen_not_verified)
      expect(call(citizen_doc: "898001160660003").reason).to eq(:pec_citizen_not_verified)
      expect(denied_codes).to eq(%w[pec_signature_invalid pec_expired pec_unit_unknown pec_professional_not_linked
                                    pec_professional_not_linked pec_citizen_not_verified pec_citizen_not_verified])
    end

    it "sem documento do cidadão (acesso pelo menu): pec_citizen_missing" do
      unsigned = "http://#{test_city_host}/pec/timeline?documento-profissional=52998224725&cnes=1234567&pec_timestamp=#{Time.current.to_i}"
      url = "#{unsigned}&pec_sign=#{OpenSSL::HMAC.hexdigest('SHA256', TimelineHelpers::PEC_KEY, unsigned)}"
      result = described_class.call(city: city, url: url, params: Rack::Utils.parse_query(URI.parse(url).query))
      expect(result.reason).to eq(:pec_citizen_missing)
    end

    it "vínculo encerrado ou usuário desativado: pec_professional_not_linked" do
      nurse.professional.links.active.sole.update!(ended_at: Time.current, ended_by_user: reception!)
      expect(call.reason).to eq(:pec_professional_not_linked)
    end

    it "desligado, fora do modo integrado ou sem configuração: pec_disabled" do
      PecSetting.current.update!(hmac_key: nil, hmac_key_updated_at: nil)
      expect(call.reason).to eq(:pec_disabled)
      pec_settings!
      city.update!(record_mode: "record")
      expect(call.reason).to eq(:pec_disabled)
    end

    it "sessão curta: volta o mesmo grant; vence em 15 min, troca de cidade, vínculo encerrado ou interruptor desligado" do
      grant = call.payload[:grant]
      token = PecExternal::ViewSession.issue(city: city, grant: grant)
      resumed = PecExternal::ViewSession.resume(token, city: city)
      expect([ resumed.user, resumed.unit, resumed.person.citizen_ids ]).to eq([ nurse, unit, [ citizen.id ] ])
      expect(PecExternal::ViewSession.resume("lixo", city: city)).to be_nil
      expect(PecExternal::ViewSession.resume(nil, city: city)).to be_nil
      expect(PecExternal::ViewSession.resume(token, city: City.new(id: SecureRandom.uuid))).to be_nil
      travel_to(16.minutes.from_now) { expect(PecExternal::ViewSession.resume(token, city: city)).to be_nil }
      Platform::Features.set!(city: city, key: "pec_external_view", enabled: false, maintainer: ledi_maintainer!)
      expect(PecExternal::ViewSession.resume(token, city: city)).to be_nil
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/services/pec_external/authorize_spec.rb`. Expected: FAIL (`uninitialized constant PecExternal::Authorize`).

- [ ] **Step 3: Implemente** `app/services/pec_external/authorize.rb`:

  ```ruby
  # Conferência da chamada do PEC (ADR 0035; spec §7), nesta ordem:
  # interruptor (e configuração), assinatura, horário, uso único (Signature),
  # CNES de unidade ativa, profissional (CPF ou CNS) com vínculo ativo nessa
  # unidade (a regra do 19a: health_professional, vínculo, CBO permitido) e a
  # pessoa com ao menos um par validado (só por CPF: o CNS do par é cifrado
  # não determinístico). O código da recusa é genérico — nunca qual parâmetro.
  module PecExternal
    module Authorize
      DOCUMENT = /\A(\d{11}|\d{15})\z/
      CNES = /\A\d{7}\z/

      Grant = Data.define(:user, :unit, :person)

      module_function

      def call(city:, url:, params:, now: Time.current)
        result = evaluate(city, url, params, now)
        if result.failure?
          DomainEvents.publish("pec.external_view_denied", reason_code: result.reason.to_s)
        else
          grant = result.payload[:grant]
          DomainEvents.publish("pec.external_view_opened", user_id: grant.user.id, unit_id: grant.unit.id,
                                                            patient_id: grant.person.patient_id)
        end
        result
      end

      def linked?(user, unit)
        ApplicationRecord.transaction { Consultations::Authorization.link_for(user: user, health_unit_id: unit.id).first == :ok }
      end

      def evaluate(city, url, params, now)
        setting = PecSetting.current
        configured = setting&.pec_origin.present? && setting.hmac_key_configured?
        return Result.fail(:pec_disabled) unless configured && Timeline::Gate.pec_usable?(city)

        professional_doc, citizen_doc, cnes, timestamp =
          %w[documento-profissional documento-cidadao cnes pec_timestamp].map { |name| params[name].is_a?(String) ? params[name] : nil }
        return Result.fail(:pec_signature_invalid) unless professional_doc.to_s.match?(DOCUMENT) && cnes.to_s.match?(CNES)

        checked = Signature.verify(url: url, key: setting.hmac_key, timestamp: timestamp, now: now)
        return Result.fail(checked) unless checked == :ok

        unit = HealthUnit.find_by(cnes: cnes, active: true)
        return Result.fail(:pec_unit_unknown) unless unit

        user = professional_user(professional_doc)
        return Result.fail(:pec_professional_not_linked) unless user && linked?(user, unit)
        return Result.fail(:pec_citizen_missing) if citizen_doc.blank?

        person = citizen_doc.match?(/\A\d{11}\z/) ? Timeline::Person.for_cpf(citizen_doc) : nil
        return Result.fail(:pec_citizen_not_verified) if person.nil? || person.citizen_ids.empty?

        Result.ok(grant: Grant.new(user: user, unit: unit, person: person))
      end

      def professional_user(document)
        professional = document.length == 11 ? Professional.find_by(cpf: document) : Professional.find_by(cns: document)
        user = professional&.user
        user if user && user.deactivated_at.nil?
      end
      private_class_method :evaluate, :professional_user
    end
  end
  ```

  `app/services/pec_external/view_session.rb`:

  ```ruby
  # Sessão curta da página do PEC (ADR 0035; Desvio 4, Divergência D3): token
  # cifrado e assinado (MessageEncryptor, 15 min, propósito próprio) com a
  # cidade, o usuário, a unidade e a pessoa. Vai no DOM da página e volta no
  # cabeçalho X-Pec-View-Session do "carregar mais" — sem cookie de terceiro.
  # A cada uso relê o usuário, a unidade, o vínculo e o interruptor.
  module PecExternal
    module ViewSession
      HEADER = "X-Pec-View-Session".freeze
      TTL = 15.minutes
      PURPOSE = "pec_view_session".freeze

      module_function

      def issue(city:, grant:)
        encryptor.encrypt_and_sign({ "c" => city.id, "u" => grant.user.id, "h" => grant.unit.id,
                                     "p" => grant.person.patient_id, "i" => grant.person.citizen_ids },
                                   expires_in: TTL, purpose: PURPOSE)
      end

      def resume(token, city:)
        return nil unless token.is_a?(String) && token.present?

        payload = encryptor.decrypt_and_verify(token, purpose: PURPOSE)
        return nil unless payload.is_a?(Hash) && payload["c"] == city.id

        user = User.find_by(id: payload["u"], deactivated_at: nil)
        unit = HealthUnit.find_by(id: payload["h"], active: true)
        return nil unless user && unit && Timeline::Gate.pec_usable?(city) && Authorize.linked?(user, unit)

        person = Timeline::Person.new(citizen_ids: Array(payload["i"]), patient_id: payload["p"], verified: true)
        Authorize::Grant.new(user: user, unit: unit, person: person)
      rescue ActiveSupport::MessageEncryptor::InvalidMessage, ActiveSupport::MessageVerifier::InvalidSignature
        nil
      end

      def encryptor
        key = Rails.application.key_generator.generate_key("pec-view-session", ActiveSupport::MessageEncryptor.key_len)
        ActiveSupport::MessageEncryptor.new(key)
      end
      private_class_method :encryptor
    end
  end
  ```

- [ ] **Step 4: Run** `rspec spec/services/pec_external`. Expected: PASS. (Se `screener!` criar o vínculo com um CBO fora de `Consultations::Cbos`, use `doctor!(unit)` no `let(:nurse)` — a regra é a do 19a, não do plano.)

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/pec_external/authorize.rb app/services/pec_external/view_session.rb spec/services/pec_external/authorize_spec.rb
  git commit -m "feat: authorize pec external view in order with short encrypted session

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 14: A página do PEC (`GET /pec/timeline` e `GET /pec/timeline/more`)

**Files:**
- Create: `app/services/pec_external/page.rb`, `app/controllers/pec_timelines_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/pec_timeline_spec.rb`

**Interfaces:**
- Consumes: `PecExternal::{Authorize,ViewSession}` (Task 13), `Timeline::{Query,Trail,Cursor}` (Task 7), `PecSetting`.
- Produces: `GET /pec/timeline?documento-profissional=&documento-cidadao=&cnes=&pec_timestamp=&pec_sign=` → HTML; `GET /pec/timeline/more?before=` (cabeçalho `X-Pec-View-Session`) → fragmento `<li>…</li>` com `X-Next-Before`; `PecExternal::Page.timeline(display_name:, page:, token:, nonce:)`, `.items(entries)`, `.failure(code, nonce:)`; `PecTimelinesController::PAGE_LIMIT = 50`.

- [ ] **Step 1: Write the failing test** `spec/requests/pec_timeline_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §7 (ADR 0035): HTML só leitura para a tela de "sistemas externos";
  # frame-ancestors do PEC da cidade, sem X-Frame-Options; recusa genérica com
  # código; uso único; sessão curta; CPF/CNS mascarados no log.
  RSpec.describe "Página do PEC", type: :request do
    let(:unit) { create_unit("UBS Norte").tap { |u| u.update!(cnes: "1234567") } }
    let(:nurse) { screener!(unit).tap { |u| u.professional.update!(cpf: "52998224725") } }
    let(:citizen) { verified_citizen!(151, social_name: "Seu Zé") }

    before do
      timeline_city!(record_mode: "integrated", clinical_record: false, pec: true)
      pec_settings!
      nurse
      walk_in_attendance!(unit, citizen: citizen)
    end

    def open_page(**opts) = get(signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567", **opts))

    it "válida: HTML com aviso, a pessoa e as entradas; cabeçalhos de iframe; trilha pec_external" do
      open_page
      expect(response).to have_http_status(:ok)
      expect(response.media_type).to eq("text/html")
      expect(response.body).to include("Registros do Rota Saúde — consultas desta cidade estão no PEC.", "Seu Zé",
                                       "Atendimento — UBS Norte", "Triagem — ")
      expect(response.headers["Content-Security-Policy"]).to include("frame-ancestors https://pec.cidade.gov.br")
      expect(response.headers).not_to have_key("X-Frame-Options")
      expect(response.headers["Cache-Control"]).to include("no-store")
      expect(response.headers["Referrer-Policy"]).to eq("no-referrer")
      expect(response.body).not_to include(citizen.cpf, "52998224725")
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload)
        .to include("access" => "pec_external", "user_id" => nurse.id, "person_ref" => "citizen:#{citizen.id}")
    end

    it "reenviada, vencida, inválida e sem vínculo: página genérica com o código, 403, sem dizer o parâmetro" do
      path = signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567")
      get path
      get path
      expect([ response.status, response.body ]).to match([ 403, a_string_including("pec_replayed") ])
      open_page(at: 10.minutes.ago)
      expect(response.body).to include("pec_expired")
      get path.sub(/pec_sign=\h{4}/, "pec_sign=0000")
      expect(response.body).to include("pec_signature_invalid")
      expect(response.body).not_to include("documento", "assinatura inválida", "timestamp")
      get signed_pec_path(professional: "11144477735", citizen: citizen.cpf, cnes: "1234567")
      expect(response.body).to include("pec_professional_not_linked")
      expect(response.headers["Content-Security-Policy"]).to include("frame-ancestors https://pec.cidade.gov.br")
      expect(response.headers).not_to have_key("X-Frame-Options")
    end

    it "interruptor desligado: pec_disabled; sem configuração, frame-ancestors 'none'" do
      PecSetting.current.update!(pec_origin: nil)
      open_page
      expect(response.body).to include("pec_disabled")
      expect(response.headers["Content-Security-Policy"]).to include("frame-ancestors 'none'")
    end

    it "carregar mais com a sessão curta; sem token, vínculo encerrado ou interruptor desligado: 401 (Review Focus 5)" do
      stub_const("PecTimelinesController::PAGE_LIMIT", 1)
      open_page
      token = response.body[/data-session="([^"]+)"/, 1]
      before = response.body[/data-before="([^"]+)"/, 1]
      expect([ token, before ]).to all(be_present)
      get "/pec/timeline/more", params: { before: before }, headers: { "X-Pec-View-Session" => CGI.unescapeHTML(token) }
      expect(response).to have_http_status(:ok)
      expect(response.body).to start_with("<li")
      expect(response.headers).to have_key("X-Next-Before")
      get "/pec/timeline/more", params: { before: before }
      expect([ response.status, response.body ]).to match([ 401, a_string_including("pec_session_expired") ])
      nurse.professional.links.active.sole.update!(ended_at: Time.current, ended_by_user: reception!)
      get "/pec/timeline/more", params: { before: before }, headers: { "X-Pec-View-Session" => CGI.unescapeHTML(token) }
      expect(response).to have_http_status(:unauthorized)
    end

    it "CPF, CNS e assinatura mascarados no log de requisições" do
      log = capture_log { open_page }
      expect(log).to include("/pec/timeline")
      expect(log).not_to include(citizen.cpf, "52998224725")
      expect(log).to include("[FILTERED]")
    end
  end
  ```

- [ ] **Step 2: Run** `rspec spec/requests/pec_timeline_spec.rb`. Expected: FAIL (rota inexistente).

- [ ] **Step 3: Implemente.** Em `config/routes.rb`, depois das rotas `/v/...` da conferência pública:

  ```ruby
    # Página para a tela de "sistemas externos" do e-SUS PEC (ADR 0035;
    # contrato §5). Host da cidade; sem sessão do Rota Saúde: assinatura HMAC
    # da URL, uso único e sessão curta em cabeçalho.
    get "/pec/timeline",      to: "pec_timelines#show"
    get "/pec/timeline/more", to: "pec_timelines#more"
  ```

  `app/services/pec_external/page.rb`:

  ```ruby
  # HTML da página do PEC (ADR 0035): sem asset externo, tudo escapado, estilo
  # e script com nonce (a CSP do controller só aceita os deles). Destaca o que
  # o PEC não tem: triagens, escutas, agendamentos e campanhas.
  module PecExternal
    module Page
      HIGHLIGHT = %w[triage screening appointment campaign].freeze
      NOTICE = "Registros do Rota Saúde — consultas desta cidade estão no PEC.".freeze
      FAILURE = "Não foi possível abrir a linha do tempo do Rota Saúde. Abra de novo pelo PEC; " \
                "se continuar, informe o código abaixo à coordenação.".freeze
      SCRIPT = <<~JS.freeze
        const list = document.getElementById("t");
        const more = document.getElementById("more");
        if (more) more.addEventListener("click", async () => {
          more.disabled = true;
          const r = await fetch("/pec/timeline/more?before=" + encodeURIComponent(more.dataset.before),
            { headers: { "X-Pec-View-Session": list.dataset.session } });
          if (!r.ok) { const p = document.createElement("p"); p.textContent = "Sessão encerrada. Abra de novo pelo PEC."; more.replaceWith(p); return; }
          list.insertAdjacentHTML("beforeend", await r.text());
          const next = r.headers.get("X-Next-Before");
          if (next) { more.dataset.before = next; more.disabled = false; } else { more.remove(); }
        });
      JS

      module_function

      def h(text) = ERB::Util.html_escape(text.to_s)

      def timeline(display_name:, page:, token:, nonce:)
        more = page.next_before ? %(<button id="more" type="button" data-before="#{h(page.next_before)}">Carregar mais</button>) : ""
        layout(<<~HTML, nonce: nonce)
          <p class="notice">#{h(NOTICE)}</p>
          <h1>Linha do tempo — #{h(display_name)}</h1>
          <ol id="t" data-session="#{h(token)}">#{items(page.entries)}</ol>
          #{more}
          <script nonce="#{h(nonce)}">#{SCRIPT}</script>
        HTML
      end

      def items(entries) = entries.map { |entry| item(entry) }.join

      def item(entry)
        at = entry.occurred_at.in_time_zone.strftime("%d/%m/%Y %H:%M")
        meta = [ entry.unit&.dig(:name), entry.professional&.dig(:name) ].compact.join(" · ")
        summary = entry.summary ? " — #{h(entry.summary)}" : ""
        details = meta.empty? ? "" : "<br><small>#{h(meta)}</small>"
        %(<li class="#{HIGHLIGHT.include?(entry.kind) ? 'hl' : 'e'}"><time>#{h(at)}</time> <strong>#{h(entry.title)}</strong>#{summary}#{details}</li>)
      end

      def failure(code, nonce:) = layout(%(<p>#{h(FAILURE)}</p><p>Código: <code>#{h(code)}</code></p>), nonce: nonce)

      def layout(body, nonce:)
        <<~HTML
          <!doctype html><html lang="pt-BR"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
          <title>Rota Saúde — linha do tempo</title>
          <style nonce="#{h(nonce)}">body{font-family:system-ui,sans-serif;margin:1rem;color:#111;background:#fff}
          .notice{background:#fff7d6;padding:.5rem;border-left:4px solid #c90}ol{padding-left:1.25rem}li{margin:.4rem 0}
          li.hl strong{color:#0a4d8c}small{color:#555}</style></head>
          <body>#{body}</body></html>
        HTML
      end
    end
  end
  ```

  `app/controllers/pec_timelines_controller.rb`:

  ```ruby
  # Página para a tela de "sistemas externos" do e-SUS PEC (ADR 0035; spec §7;
  # contrato §5). Host da cidade (CityResolution), sem sessão do Rota Saúde. A
  # URL conferida é a que chegou (base_url + original_fullpath): atrás do proxy,
  # Host e X-Forwarded-Proto precisam ser repassados (gate de go-live). Toda
  # resposta: CSP com frame-ancestors do PEC da cidade ('none' sem
  # configuração), sem X-Frame-Options, no-store e no-referrer (a URL tem CPF).
  class PecTimelinesController < ApplicationController
    PAGE_LIMIT = 50

    before_action { @nonce = SecureRandom.base64(16) }

    def show
      url = "#{request.base_url}#{request.original_fullpath}"
      result = PecExternal::Authorize.call(city: Current.city, url: url, params: request.query_parameters)
      return failure(result.reason.to_s) if result.failure?

      grant = result.payload[:grant]
      page = Timeline::Query.call(person: grant.person, limit: self.class::PAGE_LIMIT)
      Timeline::Trail.viewed!(person: grant.person, access: "pec_external", user: grant.user)
      token = PecExternal::ViewSession.issue(city: Current.city, grant: grant)
      html(PecExternal::Page.timeline(display_name: grant.person.display_name, page: page, token: token, nonce: @nonce))
    end

    def more
      grant = PecExternal::ViewSession.resume(request.headers[PecExternal::ViewSession::HEADER], city: Current.city)
      return failure("pec_session_expired", status: :unauthorized) unless grant

      page = Timeline::Query.call(person: grant.person, before: Timeline::Cursor.decode(params[:before]), limit: self.class::PAGE_LIMIT)
      Timeline::Trail.viewed!(person: grant.person, access: "pec_external", user: grant.user)
      response.headers["X-Next-Before"] = page.next_before.to_s
      html(PecExternal::Page.items(page.entries))
    rescue Timeline::Cursor::Invalid
      failure("invalid_cursor", status: :unprocessable_entity)
    end

    private

    def failure(code, status: :forbidden) = html(PecExternal::Page.failure(code, nonce: @nonce), status: status)

    def html(body, status: :ok)
      origin = PecSetting.current&.pec_origin
      response.headers.delete("X-Frame-Options")
      response.headers["Content-Security-Policy"] =
        "default-src 'none'; style-src 'nonce-#{@nonce}'; script-src 'nonce-#{@nonce}'; connect-src 'self'; " \
        "base-uri 'none'; form-action 'none'; frame-ancestors #{origin.presence || "'none'"}"
      response.headers["Cache-Control"] = "no-store"
      response.headers["Referrer-Policy"] = "no-referrer"
      render body: body, content_type: "text/html; charset=utf-8", status: status
    end
  end
  ```

  Se o exemplo do log mostrar o CPF na linha "Started GET" mesmo com o filtro da Task 2 (versão do Rails que não aplica `filter_parameters` à query em `filtered_path`), estenda `RequestLogPathMask` (`config/initializers/request_log_path_mask.rb`) com uma troca das três chaves (`/(documento-cidadao|documento-profissional|pec_sign)=[^&]*/` → `\1=[FILTERED]`) e um exemplo em `spec/initializers/request_log_path_mask_spec.rb` (se existir; senão no próprio `pec_timeline_spec.rb`).

- [ ] **Step 4: Run** `rspec spec/requests/pec_timeline_spec.rb spec/services/pec_external`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add app/services/pec_external/page.rb app/controllers/pec_timelines_controller.rb config/routes.rb spec/requests/pec_timeline_spec.rb
  git commit -m "feat: serve read-only timeline page for the pec external systems screen

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 6 — Invariantes, semente e prova

### Task 15: Invariantes do ADR 0035

**Files:**
- Test: `spec/invariants/timeline_invariants_spec.rb`

**Interfaces:**
- Consumes: as rotas das Tasks 8, 9, 10, 14 e o maintenance da Task 11.

- [ ] **Step 1: Write the test** `spec/invariants/timeline_invariants_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0035, Invariantes. Prova por comportamento (rotas reais) e, para a
  # comparação em tempo constante e a chave, também pelo código.
  RSpec.describe "Invariantes da linha do tempo (ADR 0035)", type: :request do
    let(:unit) { create_unit("UBS Centro").tap { |u| u.update!(cnes: "1234567") } }
    let(:doctor) { doctor!(unit).tap { |u| u.professional.update!(cpf: "52998224725") } }
    let(:citizen) { verified_citizen!(161) }
    def body = JSON.parse(response.body)

    def viewed_accesses = DomainEvent.where(name: "clinical_record.viewed").map { |e| e.payload["access"] }

    context "modo record com documentos" do
      before { timeline_city!(documents: true); ciap2_release!; cid10_release!; sigtap_release! }

      it "toda leitura gera trilha: timeline e citizen" do
        attendance = consulting_attendance!(unit, citizen: citizen, doctor: doctor)
        sign_in_as(doctor)
        get "/attendance/timeline", params: { attendance_id: attendance.id }
        sign_in_citizen(citizen.phone)
        get "/citizen/history", params: { citizen_id: citizen.id }
        get "/citizen/health_summary", params: { citizen_id: citizen.id }
        expect(viewed_accesses).to eq(%w[timeline citizen])
      end

      it "o cidadão nunca vê texto clínico livre, CID sem nome, número SNCR, talão nem dado de outra pessoa" do
        allow(ClinicalTerms).to receive(:label).and_call_original
        allow(ClinicalTerms).to receive(:label).with("cid10", "N390", anything).and_return(nil)
        consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen,
                                               subjective: "MARCADOR-SOAP queixa", plan: "MARCADOR-PLANO",
                                               evaluated_problems: [ { "terminology" => "cid10", "code" => "N390", "action" => "add" } ])
        raw_document!(consultation: consultation, kind: "prescription", issue_mode: "paper",
                      content: { "items" => [ { "printed_description" => "MARCADOR-ITEM" } ],
                                 "sncr" => { "number" => "2602.6-53.0000001" } })
        raw_document!(consultation: consultation, kind: "controlled_notification_record", issue_mode: "paper",
                      content: { "notification_type" => "B", "paper_number" => "TALAO-MARCADOR", "item" => {} })
        other = verified_citizen!(162, social_name: "OUTRA-PESSOA")
        finalized_consultation!(unit: unit, doctor: doctor, citizen: other)
        sign_in_citizen(citizen.phone)
        get "/citizen/history", params: { citizen_id: citizen.id }
        history = response.body
        get "/citizen/health_summary", params: { citizen_id: citizen.id }
        [ history, response.body ].each do |text|
          expect(text).not_to include("MARCADOR-SOAP", "MARCADOR-PLANO", "MARCADOR-ITEM", "N390", "2602.6-53", "TALAO-MARCADOR",
                                      "OUTRA-PESSOA")
        end
      end

      it "par só declarado nunca entra na pessoa nem vê resumo ou documento" do
        consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
        document = raw_document!(consultation: consultation, kind: "sick_note", issue_mode: "paper")
        declared = Citizen.create!(cpf: citizen.cpf, phone: "+5541966660001", verification_level: "declared")
        completed_web_triage_for(declared)
        expect(Timeline::Person.for_patient(consultation.patient).citizen_ids).not_to include(declared.id)
        sign_in_citizen(declared.phone)
        get "/citizen/history", params: { citizen_id: declared.id }
        expect(body["entries"].map { |e| e["kind"] }.uniq).to eq(%w[triage])
        get "/citizen/health_summary", params: { citizen_id: declared.id }
        expect(response).to have_http_status(:forbidden)
        get "/citizen/documents/#{document.id}/pdf", params: { citizen_id: declared.id }
        expect(response).to have_http_status(:forbidden)
      end
    end

    context "modo integrado com o PEC" do
      before do
        timeline_city!(record_mode: "integrated", clinical_record: false, pec: true)
        pec_settings!
      end

      it "a página do PEC gera trilha pec_external e só abre com assinatura válida, recente e de uso único" do
        doctor
        path = signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567")
        get path
        expect(response).to have_http_status(:ok)
        get path
        expect(response).to have_http_status(:forbidden)
        get signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567", at: 6.minutes.ago)
        expect(response).to have_http_status(:forbidden)
        expect(viewed_accesses).to eq(%w[pec_external])
      end

      it "a chave HMAC nunca é devolvida nem logada; a comparação é em tempo constante" do
        expect(PecSetting.current.inspect).not_to include(TimelineHelpers::PEC_KEY)
        log = capture_log { get signed_pec_path(professional: "52998224725", citizen: citizen.cpf, cnes: "1234567") }
        expect(log).not_to include(TimelineHelpers::PEC_KEY)
        expect(response.body).not_to include(TimelineHelpers::PEC_KEY)
        fields = Maintenance::Schema.types.values.select { |t| t.respond_to?(:fields) }.flat_map { |t| t.fields.keys }
        expect(fields.grep(/hmacKey\z|hmac_key\z/)).to be_empty
        source = Rails.root.join("app/services/pec_external/signature.rb").read
        expect(source).to include("ActiveSupport::SecurityUtils.secure_compare")
        expect(source).not_to match(/expected\s*==|==\s*expected/)
        # Quem LÊ o valor (`.hmac_key`, não `hmac_key_configured?`/`_updated_at`): só a conferência da assinatura.
        readers = Dir[Rails.root.join("app/**/*.rb")].select { |f| File.read(f).match?(/\.hmac_key\b(?!_)/) }
                                                    .map { |f| f.delete_prefix("#{Rails.root}/") }
        expect(readers).to eq(%w[app/services/pec_external/authorize.rb])
      end
    end
  end
  ```

  `finalized_consultation!` repassa `**over` para `draft_body` (as chaves sobrescrevem as do rascunho); `evaluated_problems` com CID-10 `N390` existe em `ClinicalRecordHelpers::CID10`.

- [ ] **Step 2: Run** `rspec spec/invariants/timeline_invariants_spec.rb`. Expected: PASS. Uma falha aqui é bug na task dona (fornecedor, controller ou página), nunca motivo para afrouxar a asserção. Se aparecer outro arquivo que lê `.hmac_key`, ele é o bug (só a conferência da assinatura lê o valor; o simulador de dev mora em `lib/`).

- [ ] **Step 3: Commit**

  ```bash
  git add spec/invariants/timeline_invariants_spec.rb
  git commit -m "test: pin patient timeline invariants from adr 0035

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 16: Semente de dev e simulador do PEC (só em dev)

**Files:**
- Create: `lib/timeline_crew.rb`, `lib/pec_simulator.rb`, `app/controllers/dev/pec_simulators_controller.rb`
- Modify: `config/routes.rb` (bloco `if Rails.env.development?`), `db/seeds.rb`
- Test: `spec/lib/timeline_crew_spec.rb`, `spec/lib/pec_simulator_spec.rb`, `spec/architecture/pec_simulator_route_spec.rb`

**Interfaces:**
- Consumes: `PecSetting`, `Platform::Features.set!`, `UpdateCityRecordSettings`, `ClinicalRecordCrew` (privados `ensure_citizen`, `checked_in!`, como `ScreeningCrew.send(:cpf_for, …)` já é usado), `PecExternal::Signature` (no teste).
- Produces: `TimelineCrew.seed_current_city(slug:, ddd:) -> Hash`; `PecSimulator.signed_url(city:, professional:, citizen:, cnes:, at: Time.current, base: …) -> String`; `PecSimulator.defaults(city) -> Hash`; `PecSimulator.page(city:, form:, url:) -> String`; rota de dev `GET http://pec-sim.localhost:3040/pec-simulator`.

- [ ] **Step 1: Write the failing tests.** `spec/lib/pec_simulator_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/pec_simulator").to_s

  # O simulador monta a URL como o PEC (docs/pesquisa/2026-10-11-pec-sistemas-externos.md):
  # a assinatura que ele gera é a que a página aceita.
  RSpec.describe PecSimulator do
    let(:city) { timeline_city!(record_mode: "integrated", clinical_record: false, pec: true) }

    it "assina a URL inteira até o pec_timestamp, com a chave da cidade" do
      pec_settings!(key: "dev-pec-hmac-key-teste-0123456789")
      at = Time.zone.at(1_791_720_000)
      url = described_class.signed_url(city: city, professional: "529.982.247-25", citizen: "111.444.777-35", cnes: "1234567",
                                       at: at, base: "http://maringa.localhost:3040")
      expect(url).to start_with("http://maringa.localhost:3040/pec/timeline?documento-profissional=52998224725" \
                                "&documento-cidadao=11144477735&cnes=1234567&pec_timestamp=1791720000&pec_sign=")
      expect(PecExternal::Signature.verify(url: url, key: "dev-pec-hmac-key-teste-0123456789", timestamp: "1791720000", now: at))
        .to eq(:ok)
      without_citizen = described_class.signed_url(city: city, professional: "52998224725", citizen: "", cnes: "1234567", at: at)
      expect(without_citizen).not_to include("documento-cidadao")
    end

    it "a página tem o formulário e o iframe da URL" do
      html = described_class.page(city: city, form: { professional: "1", citizen: "2", cnes: "3", minutes_ago: "0" },
                                  url: "http://x.localhost/pec/timeline?a=1&pec_sign=f")
      expect(html).to include("<form", %(<iframe src="http://x.localhost/pec/timeline?a=1&amp;pec_sign=f"))
    end
  end
  ```

  `spec/architecture/pec_simulator_route_spec.rb`:

  ```ruby
  require "rails_helper"

  # Mesmo princípio de maintenance_route_spec.rb: o simulador do PEC assina URL
  # com a chave da cidade — só existe em development, e a garantia é o roteador.
  RSpec.describe "the PEC simulator is development-only" do
    it "does not exist outside development" do
      expect(Rails.env.development?).to be(false)
      expect { Rails.application.routes.recognize_path("http://pec-sim.localhost/pec-simulator") }
        .to raise_error(ActionController::RoutingError)
    end
  end
  ```

  `spec/lib/timeline_crew_spec.rb` (no padrão de `spec/lib/clinical_record_crew_spec.rb`: confira lá como a spec prepara a cidade, o mantenedor `dev@local`, a recepção e os profissionais da semente — reuse o mesmo arranjo):

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/timeline_crew").to_s

  RSpec.describe TimelineCrew do
    # Arranjo de spec/lib/clinical_record_crew_spec.rb (cidade, dev@local,
    # recepcao@/enfermeira@<slug>.demo com vínculo e CPF). Ver aquele arquivo.

    it "fora da cidade integrada só liga patient_timeline" do
      result = described_class.seed_current_city(slug: "curitiba", ddd: "41")
      expect(result[:switch]).to start_with("patient_timeline ligado")
      expect(Platform::Features.enabled?(Current.city, "patient_timeline")).to be(true)
      expect(Platform::Features.enabled?(Current.city, "pec_external_view")).to be(false)
    end

    it "na cidade integrada: modo integrated, pec_external_view, configuração e CNES; idempotente" do
      2.times { described_class.seed_current_city(slug: "maringa", ddd: "44") }
      expect(City.find(Current.city.id).record_mode).to eq("integrated")
      expect(Timeline::Gate.pec_usable?(Current.city)).to be(true)
      expect(PecSetting.current.pec_origin).to eq("http://pec-sim.localhost:3040")
      expect(PecSetting.current.hmac_key).to eq("dev-pec-hmac-key-maringa-0123456789")
      expect(HealthUnit.where(cnes: "7700001").count).to eq(1)
    end
  end
  ```

  (Se o arranjo de `clinical_record_crew_spec.rb` não servir à Maringá por slug, use o slug que ele usa e passe-o como `TimelineCrew::INTEGRATED` por `stub_const`; a asserção não muda.)

- [ ] **Step 2: Run** `rspec spec/lib/pec_simulator_spec.rb spec/lib/timeline_crew_spec.rb spec/architecture/pec_simulator_route_spec.rb`. Expected: FAIL (arquivos inexistentes); a de rota passa já (a rota não existe) e continua passando depois.

- [ ] **Step 3: Implemente.** `lib/pec_simulator.rb`:

  ```ruby
  require "openssl"
  require "erb"

  # Simulador do e-SUS PEC para a prova em dev (ADR 0035): monta a URL como a
  # tela de "sistemas externos" (docs/pesquisa/2026-10-11-pec-sistemas-externos.md)
  # e assina com a chave gravada na cidade. Só a rota de development o usa
  # (Dev::PecSimulatorsController); nunca roda em ambiente publicado.
  module PecSimulator
    BASE_TEMPLATE = "http://%{slug}.localhost:3040".freeze

    module_function

    def signed_url(city:, professional:, citizen:, cnes:, at: Time.current,
                   base: format(ENV.fetch("PEC_SIMULATOR_CITY_BASE", BASE_TEMPLATE), slug: city.slug))
      key = CityConnection.with(city) { PecSetting.current&.hmac_key }
      raise ArgumentError, "cidade sem chave do PEC (rode db:seed)" if key.to_s.empty?

      query = { "documento-profissional" => professional, "documento-cidadao" => citizen, "cnes" => cnes }
              .filter_map { |name, value| (digits = value.to_s.gsub(/\D/, "")).empty? ? nil : "#{name}=#{digits}" }.join("&")
      unsigned = "#{base}/pec/timeline?#{query}&pec_timestamp=#{at.to_i}"
      "#{unsigned}&pec_sign=#{OpenSSL::HMAC.hexdigest('SHA256', key, unsigned)}"
    end

    # Os valores da semente (TimelineCrew): enfermeira, cidadã validada e o CNES da unidade dela.
    def defaults(city)
      CityConnection.with(city) do
        nurse = User.find_by(email_address: "enfermeira@#{city.slug}.demo")&.professional
        unit = nurse&.links&.active&.order(:started_at)&.first&.health_unit
        citizen = Citizen.not_erased.verification_level_verified.order(:created_at).first
        { professional: nurse&.cpf.to_s, citizen: citizen&.cpf.to_s, cnes: unit&.cnes.to_s, minutes_ago: "0" }
      end
    end

    def page(city:, form:, url:)
      h = ->(value) { ERB::Util.html_escape(value.to_s) }
      fields = { professional: "CPF/CNS do profissional", citizen: "CPF do cidadão (vazio = acesso pelo menu)",
                 cnes: "CNES", minutes_ago: "Minutos atrás (6+ = vencida)" }
      inputs = fields.map { |name, label| %(<label>#{h.(label)} <input name="#{name}" value="#{h.(form[name])}"></label>) }.join
      frame = url ? %(<p><a href="#{h.(url)}" target="pec">Reenviar a mesma URL</a></p><iframe name="pec" src="#{h.(url)}" width="100%" height="600"></iframe>) : ""
      <<~HTML
        <!doctype html><html lang="pt-BR"><head><meta charset="utf-8"><title>PEC simulado — #{h.(city.name)}</title>
        <style>body{font-family:system-ui,sans-serif;margin:1rem}label{display:block;margin:.25rem 0}iframe{border:1px solid #999}</style></head>
        <body><h1>PEC simulado (desenvolvimento) — #{h.(city.name)}</h1>
        <form method="get"><input type="hidden" name="city" value="#{h.(city.slug)}"><input type="hidden" name="go" value="1">#{inputs}
        <button>Abrir "sistemas externos"</button></form>#{frame}</body></html>
      HTML
    end
  end
  ```

  `app/controllers/dev/pec_simulators_controller.rb`:

  ```ruby
  require Rails.root.join("lib/pec_simulator").to_s

  # Simulador do PEC (ADR 0035), só em development (a rota só é desenhada lá;
  # a ação confere de novo). Fica num host próprio (pec-sim.localhost) para
  # que o iframe da cidade seja de outra origem, como no PEC real, e o
  # frame-ancestors seja provado de verdade. Não resolve cidade pelo host.
  module Dev
    class PecSimulatorsController < ActionController::Base
      def show
        return head(:not_found) unless Rails.env.development?

        city = City.find_by!(slug: params.fetch(:city, "maringa"))
        form = PecSimulator.defaults(city).merge(params.permit(:professional, :citizen, :cnes, :minutes_ago).to_h.symbolize_keys)
        url = if params[:go]
                PecSimulator.signed_url(city: city, professional: form[:professional], citizen: form[:citizen], cnes: form[:cnes],
                                        at: form[:minutes_ago].to_i.minutes.ago)
              end
        render html: PecSimulator.page(city: city, form: form, url: url).html_safe # rubocop:disable Rails/OutputSafety
      end
    end
  end
  ```

  Em `config/routes.rb`, dentro de `if Rails.env.development?`, depois do `constraints(PlatformConsoleHost)`:

  ```ruby
      # Simulador do e-SUS PEC (ADR 0035): host próprio, para o iframe da
      # cidade ser de outra origem como no PEC real.
      constraints(->(request) { request.host == "pec-sim.localhost" }) do
        get "/pec-simulator", to: "dev/pec_simulators#show"
      end
  ```

  `lib/timeline_crew.rb`:

  ```ruby
  require_relative "clinical_record_crew"

  # Semente de dev do módulo 20 (ADR 0035). Dev é fictício mas imita o real:
  # patient_timeline ligado em todas as cidades pelo mantenedor de dev; Maringá
  # vira a cidade INTEGRADA (record_mode integrated pelo comando do console),
  # com pec_external_view ligado, a origem do simulador do PEC, uma chave de dev
  # e um CNES fictício na unidade da enfermeira; a cidadã validada da semente
  # do prontuário ganha um atendimento lá. Idempotente. Roda por último.
  class TimelineCrew
    DEV_MAINTAINER = "dev@local".freeze
    INTEGRATED = "maringa".freeze
    PEC_ORIGIN = "http://pec-sim.localhost:3040".freeze
    KEY_ENV = "DEV_PEC_HMAC_KEY".freeze
    DEFAULT_KEY = "dev-pec-hmac-key-maringa-0123456789".freeze
    DEV_CNES = "7700001".freeze

    class << self
      def seed_current_city(slug:, ddd:)
        maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
        return { switch: "desligado (sem mantenedor dev@local)" } unless maintainer

        city = City.find(Current.city.id)
        Platform::Features.set!(city: city, key: "patient_timeline", enabled: true, maintainer: maintainer)
        return { switch: "patient_timeline ligado (#{city.record_mode})" } unless slug == INTEGRATED

        integrate!(city, maintainer, slug, ddd)
      end

      private

      def integrate!(city, maintainer, slug, ddd)
        unless city.record_mode == "integrated"
          updated = UpdateCityRecordSettings.call(city: city, attrs: { record_mode: "integrated" })
          raise "semente da linha do tempo: modo integrado recusado (#{updated.reason})" if updated.failure?
        end
        Platform::Features.set!(city: city, key: "pec_external_view", enabled: true, maintainer: maintainer)
        setting = PecSetting.current || PecSetting.new
        unless setting.persisted? && setting.hmac_key_configured?
          setting.update!(pec_origin: PEC_ORIGIN, pec_origin_updated_at: Time.current,
                          hmac_key: ENV.fetch(KEY_ENV, DEFAULT_KEY), hmac_key_updated_at: Time.current)
        end
        nurse = User.find_by!(email_address: "enfermeira@#{slug}.demo")
        unit = nurse.professional.links.active.order(:started_at).first!.health_unit
        unit.update!(cnes: DEV_CNES) if unit.cnes.blank?
        citizen = ClinicalRecordCrew.send(:ensure_citizen, slug, ddd)
        unless Attendance.exists?(citizen_id: citizen.id, health_unit_id: unit.id)
          ClinicalRecordCrew.send(:checked_in!, slug, unit, citizen, "linha-do-tempo")
        end
        { switch: "patient_timeline e pec_external_view ligados (integrated)", unit: unit.name, cnes: unit.cnes,
          simulator: "http://pec-sim.localhost:3040/pec-simulator?city=#{slug}" }
      end
    end
  end
  ```

  Em `db/seeds.rb`: `require Rails.root.join("lib/timeline_crew").to_s` junto dos outros e, depois do bloco da receita controlada:

  ```ruby
          # ── Linha do tempo (módulo 20, ADR 0035) ─────────────────────────────
          # Por último: Maringá vira a cidade integrada com o PEC simulado.
          timeline = TimelineCrew.seed_current_city(slug: slug, ddd: ddd)
          puts "[seeds] linha do tempo .. #{timeline[:switch]}#{timeline[:simulator] ? " — simulador: #{timeline[:simulator]} (CNES #{timeline[:cnes]})" : ''}"
  ```

  Acrescente no comentário do topo de `db/seeds.rb` uma linha: "Linha do tempo (módulo 20): patient_timeline ligado; Maringá em modo integrado com o PEC simulado — ver lib/timeline_crew.rb."

- [ ] **Step 4: Run** `rspec spec/lib/pec_simulator_spec.rb spec/lib/timeline_crew_spec.rb spec/architecture/pec_simulator_route_spec.rb spec/architecture/maintenance_route_spec.rb spec/lib`. Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add lib/timeline_crew.rb lib/pec_simulator.rb app/controllers/dev/pec_simulators_controller.rb config/routes.rb db/seeds.rb spec/lib/timeline_crew_spec.rb spec/lib/pec_simulator_spec.rb spec/architecture/pec_simulator_route_spec.rb
  git commit -m "feat: seed patient timeline and add development pec simulator

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 17: Revisão final, suíte completa, api na porta 3040, prova no navegador e rollout

- [ ] **Step 1: Varredura de vazamento.**
  - `grep -rn 'DomainEvents.publish("pec\|DomainEvents.publish("clinical_record' -A3 apps/api/.claude/mod20/app` — só ids, `access`, `reason_code`, `person_ref` (multilinha: confira cada chamada inteira).
  - `grep -rn 'Platform.audit("city.pec_config_changed\|maintenance.city.pec_config_changed' -A2 apps/api/.claude/mod20/app` — `city_id`, `field`, `maintainer_id`; nunca a chave nem a origem.
  - `grep -rn "Rails.logger" apps/api/.claude/mod20/app/services/timeline apps/api/.claude/mod20/app/services/pec_external apps/api/.claude/mod20/app/controllers/pec_timelines_controller.rb` — nenhum.
  - `grep -rnE "\.hmac_key\b" apps/api/.claude/mod20/app | grep -v "hmac_key_"` — só `app/services/pec_external/authorize.rb`.
- [ ] **Step 2: Suíte completa** (worker parado; avise as outras sessões; bancos `_mod20` recriados):

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod20 -w /rails/.claude/mod20 api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde. Spec antiga que fixa listas (catálogo de interruptores, bindings, schema do maintenance, R18, rotas, payload de `clinical_record.opened`) → acrescente o do módulo 20 (é o contrato); nunca afrouxe asserção de texto livre.
- [ ] **Step 3:** `docker compose exec -T -w /rails/.claude/mod20 api bundle exec rubocop <arquivos tocados>`; corrija só o que a regra do projeto aponta.
- [ ] **Step 4: api na porta 3040.** Com autorização do usuário, uma etapa de cada vez: `bin/rails city:migrate:all` (do worktree) e `bin/rails db:seed`. Depois, sem derrubar o principal, um container do worktree com a porta publicada (o navegador precisa chegar à 3040 para o simulador e o iframe):

  ```bash
  docker compose run -d --name api-mod20 -p 127.0.0.1:3040:3040 -w /rails/.claude/mod20 \
    -e CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5190 -e CITY_WPDA_BASE_TEMPLATE=http://%{slug}.localhost:5191 \
    api bin/rails server -b 0.0.0.0 -p 3040 -P tmp/pids/server-mod20.pid
  ```
  Os Vite do dashboard (5190) e do wpda (5191) apontam `VITE_API_PROXY_TARGET` para `http://api-mod20:3040` (planos do dashboard e do wpda; as rotas novas usam prefixos que já têm proxy: `/attendance`, `/clinical_record`, `/citizen`).
- [ ] **Step 5: Prova manual no navegador** (login, OTP e TOTP são do usuário; senhas e TOTP da semente de dev podem ser mostrados no chat se ele pedir):
  1. **Profissional, modo record (Curitiba, `http://curitiba.localhost:5190/dashboard/`):** a médica abre o painel da paciente da semente → aba "Linha do tempo" com consulta, atendimentos, triagens; filtro por kind; "carregar mais"; sem aviso de parcial. A recepção recebe 403.
  2. **Profissional, modo integrado (Maringá, `http://maringa.localhost:5190/dashboard/`):** a enfermeira, em contexto do atendimento da semente, vê o aviso de histórico parcial; fora de contexto, abertura justificada (step-up) pelo CPF da cidadã → a linha do tempo abre com `patient_id` nulo.
  3. **Cidadão validado (Curitiba, `http://curitiba.localhost:5191/wpda/`):** "Meu histórico" mostra o resumo clínico (diabetes, exames, PA e peso com data), a linha do tempo sem texto da consulta e o documento com "baixar PDF" (abre o PDF) e o código de conferência.
  4. **Par declarado:** um CPF novo no mesmo celular vê só triagens e agendamentos dele e o convite para validar; o resumo responde "não validado".
  5. **PEC simulado:** `http://pec-sim.localhost:3040/pec-simulator?city=maringa` → "Abrir sistemas externos" carrega no iframe a linha do tempo da cidadã com o aviso "Registros do Rota Saúde — consultas desta cidade estão no PEC."; "Reenviar a mesma URL" mostra `pec_replayed`; minutos atrás = 6 mostra `pec_expired`; CPF de profissional sem vínculo mostra `pec_professional_not_linked`; CPF do cidadão vazio mostra `pec_citizen_missing`. Abrir a URL do iframe direto numa aba de outra origem qualquer e embutir com um HTML local (outra origem) é recusado pelo navegador (frame-ancestors). No console do container, `docker logs api-mod20 | grep pec/timeline` mostra `[FILTERED]` no lugar do CPF, do CNS e da assinatura.
  6. **Maintenance (`maintenance.localhost:5177`, `dev@local`):** em Maringá, `patient_timeline` e `pec_external_view` ligados e utilizáveis; a tela do PEC mostra a origem e "chave configurada" com a data, nunca a chave; trocar a chave pede o TOTP do momento e faz o simulador antigo cair em `pec_signature_invalid` até o simulador reler (ele lê a chave da cidade a cada clique).
- [ ] **Step 6: Pare.** Merge, push, board e docs (página de status do módulo no padrão do módulo 01) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: **api** → dashboard e wpda → maintenance. Rollout: publicar a imagem nova e rodar `city:migrate:all` dela **antes** de cortar tráfego (a migração de cidade é irreversível); os interruptores nascem desligados e são ligados por cidade (em produção não há maintenance: interruptor pela rake/console, como os do 19c; a origem e a chave do PEC por `CityConnection.with(city) { PecExternal::Configure.set_origin(origin: …); PecExternal::Configure.rotate_key(key: …) }` no console, com `PecExternal::Configure.audit!` em seguida, até existir caminho próprio). Ao voltar o checkout para a main: `DROP DATABASE` dos bancos `_mod20` (sem pedir de novo, conferindo conexões ativas) e `docker rm -f api-mod20`.
- [ ] **Step 7: Gates de go-live por cidade (antes de ligar `pec_external_view`):** o proxy de produção encaminha `/pec/*` ao api e repassa `Host` e `X-Forwarded-Proto: https` (prova em staging: a URL conferida — `request.base_url + request.original_fullpath` — tem de ser exatamente a cadastrada no PEC); cadastro do Rota Saúde como sistema externo no PEC da cidade (Local de acesso "Atendimento"; URL `https://<cidade>/pec/timeline?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]`), chave entregue pelo administrador da instalação e gravada no Rota Saúde; `pec_origin` = origem real do PEC; prova da chamada e do iframe num PEC real (5.5.x); CNES das unidades preenchido; profissionais com CPF ou CNS iguais aos do PEC e vínculo ativo nas unidades; relógio do servidor sincronizado (janela de 5 min).
- [ ] **Step 8: Pendências para o board de pendências de ciclo** (cards só com autorização): cópia integral do prontuário a pedido; desempenho da junção para histórico muito longo (projeção como cache, se preciso); exames, vacinas e procedimentos como fornecedores dos módulos que os criarem; cidadão identificado por CNS na chamada do PEC (exige CNS buscável); caminho de produção para gravar a configuração do PEC sem console.

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3 interruptores e "a pessoa" → Tasks 1, 2, 3; `pec_hmac_key` cifrada e `pec_origin` por cidade → Tasks 2, 11.
- §4 fornecedores (8 kinds, entrada, visões), junção e paginação (limite 50/100, cursor composto, empate, filtros), índices novos conferidos no `db/city_schema.rb` real (attendances, clinical_documents, consultations, campaign_recipients; triagens, agendamentos, escutas e listas do paciente já indexados) → Tasks 2, 5, 6, 7; resumo clínico do cidadão → Task 10.
- §5 Access generalizado (in_context, justified; author e administrative ficam nas telas de detalhe — Desvio 7) e trilha `timeline`/`citizen`/`pec_external` → Tasks 4, 7, 8, 9, 10, 14.
- §6 API do profissional, do cidadão (histórico, resumo, PDF) e do PEC → Tasks 8, 9, 10, 14.
- §7 página do PEC (ordem de conferência, HMAC em tempo constante, 5 min, uso único, sessão curta de 15 min, `frame-ancestors`, aviso, mascaramento no log) → Tasks 12, 13, 14.
- §8 telas: lado api das três telas (rotas); maintenance → Task 11.
- §9 segurança e LGPD → Tasks 2 (cifra, filtros), 9, 10, 15.
- §10 testes do api → cada task; prova no navegador com o PEC simulado → Tasks 16, 17.
- §11 entrega: Task 0 (documentação oficial do PEC) antes de tudo.
- ADR 0035 invariantes → Task 15; `adr_pointers` → Task 1.

**Placeholders:** nenhum "TBD". O vetor HMAC da Task 12 foi calculado com `openssl dgst -sha256 -hmac` ao escrever o plano. Onde o código real pode divergir de um helper de spec (formato de `completed_web_triage_for`, `raw_document!` do registro da Notificação, arranjo do `clinical_record_crew_spec`, rótulo gravado pelo `ApplyMedicationEvent`), o step diz o que conferir e que a asserção não muda.

**Consistência de nomes:** `Timeline::{KINDS,Gate,Person,Entry,Cursor,Query,Json,Trail,CitizenSummary,CitizenDocuments}`, `Timeline::Sources::{Window,Labels,TriageSource,AttendanceSource,ScreeningSource,AppointmentSource,ConsultationSource,ClinicalDocumentSource,ControlledNotificationSource,CampaignSource}`, `ClinicalRecord::Access.for_person`, `ClinicalRecordOpening.valid_for_person`, `PecSetting`, `PecSignatureUse`, `PecExternal::{Configure,Signature,Authorize,ViewSession,Page}`, `Platform::Features::PecConfigState`, `PecExternal::Authorize::Grant`, `PecTimelinesController::PAGE_LIMIT`, `TimelinesController`, `CitizenApi::HistoryController`, `TimelineCrew`, `PecSimulator`, `TimelineHelpers::{PEC_KEY,PEC_ORIGIN}` — conferidos entre as tasks.

**Review Focus:** as cinco linhas têm teste na task dona (7, 12, 9, 3, 14).

## Divergências propostas ao contrato

- **D1 — Nomes reais da chamada do PEC (contrato §5):** `GET /pec/timeline?documento-profissional=&documento-cidadao=&cnes=&pec_timestamp=&pec_sign=` — chaves com hífen e minúsculas (a documentação oficial dá o formato da URL que o integrador entrega ao administrador); `pec_timestamp` em segundos Unix; `pec_sign` = HMAC-SHA256 hex da URL inteira (protocolo, host, caminho e query) até antes de `&pec_sign=`.
- **D2 — Rotas do cidadão levam `citizen_id`:** `GET /citizen/history?citizen_id=&before=&limit=`, `GET /citizen/health_summary?citizen_id=`, `GET /citizen/documents/:id/pdf?citizen_id=`. A sessão do wpda é por telefone e cobre vários pares (como `GET /citizen/triages?citizen_id=`). `citizen_id` ausente ou de outro telefone → 404 `not_found`.
- **D3 — Sessão curta da página do PEC sem cookie:** token cifrado de 15 min, entregue no DOM da página e devolvido pelo "carregar mais" no cabeçalho `X-Pec-View-Session` (Safari, Firefox e Chrome bloqueiam ou restringem cookie de terceiro dentro de iframe de outro site). `GET /pec/timeline/more?before=` devolve o fragmento `<li>…</li>` e o próximo cursor no cabeçalho `X-Next-Before` (vazio na última página); sem token válido, vínculo encerrado ou interruptor desligado → 401 com o código `pec_session_expired`; cursor inválido → 422 `invalid_cursor`.
- **D4 — `GET /attendance/timeline`:** 409 `citizen_not_verified` quando o atendimento é de par só declarado (como `GET /attendance/attendances/:id/record` do 19a); 422 `invalid_kinds` para `kinds` fora da lista (ou escalar); sem `patient_id`/`attendance_id`/`opening_id`, ou alvo inexistente/id malformado → 404 `not_found`; `patient_id` sem contexto nem abertura → 403 `out_of_context`; `opening_id` vencido, de outro usuário ou inexistente → 403 `opening_required`; `from > to` → 422 `invalid_period`. Prioridade quando vêm vários: `opening_id`, depois `patient_id`, depois `attendance_id`.
- **D5 — Abertura justificada pela pessoa:** `POST /clinical_record/openings` passa a valer com `patient_timeline` ligado mesmo sem prontuário utilizável; sem paciente, cria a abertura por um par validado do CPF e responde `patient_id: null` (as chaves `{ opening_id, patient_id, expires_at }` não mudam); `clinical_record.opened` ganha `citizen_id`; o relatório das aberturas mostra o CPF mascarado do par. Sem `patient_timeline`, tudo como no 19a.
- **D6 — Código novo `pec_citizen_missing`:** chamada sem `documento-cidadao` (sistema externo cadastrado no menu principal do PEC, não no Atendimento). Documento do cidadão por CNS → `pec_citizen_not_verified` (o CNS do par não é buscável).
- **D7 — Código novo `pec_session_expired`** (ver D3) e a página de recusa com status HTTP 403 (401 no "carregar mais").
- **D8 — Trilha:** `clinical_record.viewed` da linha do tempo leva `{ patient_id|null, user_id|null, access, reason_code|null, person_ref }`, com `person_ref` = `"patient:<uuid>"` ou `"citizen:<uuid>"` (o par de menor id); `user_id` nulo no acesso `citizen`; `reason_code` da abertura no acesso `timeline` justificado. Cada página lida (inclusive o "carregar mais" e o PDF) gera um evento.
- **D9 — Plataforma e maintenance:** `city.pec_config_changed` leva também `maintainer_id`; a tentativa da mutation é auditada como `maintenance.city.pec_config_changed` (campo `pec_field`). `setCityPecOrigin` aceita `pecOrigin: null` (apaga); recusas em `{ ok: false, errors: [{ path: "pecOrigin", message: "invalid_origin" }] }` e `{ path: "key", message: "invalid_key" }`. **`rotateCityPecHmacKey(citySlug, key, code)`** exige o TOTP do momento (decisão M4; código errado → `{ ok: false }` pelo caminho de step-up da base, nada gravado). `cityPecConfig` responde `null` para cidade inexistente; as três operações são só de sessão humana.
- **D10 — PDF do cidadão:** digital ainda não assinado, cancelado, registro da Notificação, documento de outra pessoa e `clinical_documents` não utilizável → 404 `not_found` (sem distinguir); `Content-Disposition: attachment`; limite de 30 por 10 min por sessão → 429 `too_many_requests`. O `<citizen_entry>.document.downloadable` segue a mesma regra.
- **D11 — Resumo do cidadão:** ordem 404 `not_found` (par fora da sessão) → 403 `feature_disabled` → 403 `not_verified` → 404 `not_available`; problema sem nome na release não aparece (nunca CID sem nome); exame sem nome SIGTAP sai como `"Procedimento <código>"`; sinais vitais da consulta finalizada ou da escuta concluída mais recente; `weight_kg.value` número.
- **D12 — `<entry>`:** `professional.cbo_code` nulo em `attendance` e `appointment` (a origem não guarda CBO); `appointment` aponta `detail_ref { type: "appointment_request", id: <pedido> }` (pedido D2 do dashboard); `controlled_notification` aponta `detail_ref { type: "clinical_document" }`; `campaign` aponta `detail_ref { type: "campaign", id: <campaign_id> }`; `occurred_at` com segundos (o cursor guarda os microssegundos). Códigos de `status` por kind no contrato (pedido D3): `triage` `completed|aborted_by_revocation|aborted_by_timeout|aborted_by_cancellation`; `attendance` `waiting|in_care|closed`; `screening` `completed`; `appointment` `scheduled|confirmed|checked_in|cancelled_by_citizen|expired|no_show`; `consultation` `finalized`; `clinical_document`/`controlled_notification` `issued|cancelled`; `campaign` `read|unread`; no cidadão, `screening` e `consultation` com `status: null`.
- **D13 — Migração de cidade `20261011800002`** (a `20261011800001` já é do card api#51); nenhuma migração de plataforma.
- **D14 — `pec_external_view` exige a configuração** (pedido M3): `requires: ["feature:patient_timeline", "record_mode:integrated", "pec_origin", "pec_hmac_key"]`, missing também `pec_origin_missing` e `pec_hmac_key_missing`; sem um dos dois a página responde `pec_disabled`.
- **D15 — Detalhe fora de contexto só abre a consulta** (resposta ao pedido D2b): com abertura justificada, o detalhe de `consultation` abre como no 19a; documento, escuta e pedido de agendamento **não** ganham leitura pela abertura neste módulo — as rotas deles seguem com as próprias regras, e a entrada da linha do tempo já traz o resumo (tipo, situação, unidade, profissional). Generalizar essas três rotas fica como pendência de ciclo, se a tela precisar.
- **D16 — Cidadão:** `GET /citizen/history` aceita `kinds[]` (recorte dentro do que a pessoa pode ver; valor fora dos 8 → 422 `invalid_kinds`) e `limit` 1–100 (pedido W3); no `<citizen_entry>` a chave `document` vem sempre, `null` fora dos documentos (pedido W4); `GET`/`POST /citizen/session` devolvem também `features: string[]` com os interruptores utilizáveis da cidade (pedido W2).
