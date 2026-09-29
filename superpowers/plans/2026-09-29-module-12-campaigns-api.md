# Módulo 12 — Campanhas (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Campanhas da secretaria para cidadãos: público por recorte geográfico × até 7 critérios clínicos (mínimo de 5 telefones), congelado no envio, aviso na caixa do wpda e SMS opcional por cidade (texto fixo, opt-in, janela 8h–20h), com papel `campaign_manager` e step-up (F-12.1 a F-12.7, ADR 0024).

**Architecture:** Uma migração de cidade só de expansão cria `campaigns`, `campaign_recipients` e `citizen_contact_preferences`, liga `campaigns_sms_enabled` em `city_profile` e acrescenta `campaign_manager` ao CHECK de papéis; dois triggers em `db/city_triggers.sql` congelam a campanha enviada e deixam o destinatário mudar só leitura e SMS. `Campaigns::AudienceSchema` valida o JSON do público; `Campaigns::Criteria::<Kind>` devolve relações SQL de `citizen_id`; `Campaigns::Audience` compõe recorte ∩ critérios ∖ revogados sem carregar cidadão em Ruby. Comandos em `app/commands/campaigns/` fazem as transições (com `FOR UPDATE`); `Campaigns::DispatchJob` congela o público num `INSERT ... SELECT`, `Campaigns::SmsBatchJob` entrega pelo `SmsGateway` plugável e `Campaigns::DueJob` (recorrente) solta os agendamentos. Rotas: `/campaigns` (dashboard) e `/citizen/notices`, `/citizen/contact_preferences` (wpda).

**Tech Stack:** Rails 8.1 (API), PostgreSQL (banco por cidade), Solid Queue, RSpec, Active Record Encryption determinística (telefone), ROTP (step-up).

**Spec:** `docs/.claude/mod12/superpowers/specs/2026-09-29-module-12-campaigns-design.md` e `docs/.claude/mod12/adr/0024.md` (leia os dois antes de começar). Contratos HTTP fixados com os planos do dashboard e do wpda: `mod12-contracts.md` (reproduzidos na seção "Contratos" abaixo — **não mude nomes nem formatos**).

## Contratos (fixados; os planos do dashboard e do wpda leem os mesmos)

Erro da casa: `{ "error": "<code>" }`. Step-up ausente: 401 `{ "error": "mfa_required" }`. Sem papel: 403 `{ "error": "missing_role" }`. Datas em ISO 8601 com fuso; datas de critério em `YYYY-MM-DD`.

```
CampaignSummary = { id, title, status, send_at: string|null, dispatched_at: string|null, recipients_count: int|null }
Campaign = CampaignSummary & {
  body, audience: Audience, failure_reason: "below_minimum"|null, sms_enabled: bool|null, phones_count: int|null,
  created_at,
  stats: null | { read_count: int, sms: { [SmsStatus]: int } }   // null antes do envio; as 7 chaves presentes depois
}
```

| Método e rota | Papel | Step-up | Corpo | 2xx |
|---|---|---|---|---|
| GET /campaigns | campaign_manager | | | `{ campaigns: CampaignSummary[] }` (mais novas primeiro) |
| GET /campaigns/options | campaign_manager | | | `{ protocols: string[], tiers: string[], outcomes: string[], neighborhoods: {id,name}[], units: {id,name}[] }` |
| POST /campaigns/preview | campaign_manager | | `{ audience }` | `{ citizens, phones }` ou `{ below_minimum: true }` |
| POST /campaigns | campaign_manager | | `{ title, body, audience }` | 201 `{ campaign }` |
| GET /campaigns/:id | campaign_manager | | | `{ campaign }` |
| PATCH /campaigns/:id | campaign_manager | | `{ title?, body?, audience? }` | `{ campaign }` |
| POST /campaigns/:id/send | campaign_manager | sim | | `{ campaign }` (status `sending`) |
| POST /campaigns/:id/schedule | campaign_manager | sim | `{ send_at }` | `{ campaign }` |
| POST /campaigns/:id/unschedule | campaign_manager | sim | | `{ campaign }` |
| POST /campaigns/:id/cancel | campaign_manager | sim | | `{ campaign }` |
| GET /campaigns/sms_setting | campaign_manager ou municipal_admin | | | `{ enabled, gateway_configured }` |
| PUT /campaigns/sms_setting | municipal_admin | sim | `{ enabled }` | `{ enabled, gateway_configured }` |
| GET /citizen/notices | sessão do cidadão | | | `{ notices: [{ id, title, body, dispatched_at, read, cpf_masked }], unread_count }` |
| POST /citizen/notices/:id/read | sessão do cidadão | | | `{ ok: true }`; outro telefone → 404 |
| GET /citizen/contact_preferences | sessão do cidadão | | | `{ sms_available, people: [{ citizen_id, cpf_masked, sms_opt_in, notices_muted }] }` |
| PUT /citizen/contact_preferences/:citizen_id | sessão do cidadão | | `{ sms_opt_in?, notices_muted? }` | a entrada da pessoa; outro telefone → 404 |

Erros 422: `invalid_audience` (com `details: [{ path, message }]`), `invalid_campaign` (título/texto; mesmo `details`), `below_minimum`, `not_editable`, `invalid_transition`, `invalid_send_at`. 404 `not_found`.

## Desvios da spec (e precisões de contrato)

Onde a spec é omissa ou o código real obrigou a escolher, a escolha está aqui:

1. **Cidadão não tem nome.** Onde a spec diz "nome da pessoa" (§6.2, §8), vale `cpf_masked` (`Citizen#cpf_masked`, o mesmo de `GET /citizen/people`), como no contrato.
2. **CHECK de papéis.** A spec §3.5 fala só de `Membership::ROLES`; o banco tem `ck_memberships_role` (`db/city_schema.rb:330`). A migração troca o CHECK (mesmo molde de `db/city_migrate/20260926000001_create_appointments.rb:131-135`). O `down` volta o CHECK antigo e **falha** se já houver `membership` `campaign_manager` (memberships não se apagam — trigger `memberships_guard`); é o comportamento desejado.
3. **Ação `send`.** `send` é método de `Object`: a rota `POST /campaigns/:id/send` aponta para a ação `send_now`. O caminho do contrato não muda.
4. **`details[].path`** é ponteiro JSON **relativo à raiz do público** para `invalid_audience` (ex.: `/geo/neighborhood_ids`, `/clinical/all/0/from`; o próprio objeto ausente ou que não é objeto: `/`), e `/title` ou `/body` para `invalid_campaign`. `message` é um código curto em inglês (`required`, `unknown_key`, `invalid_kind`, `invalid_uuid`, `empty`, `too_many`, `duplicate`, `not_a_list`, `not_a_string`, `invalid_value`, `invalid_date`, `future_date`, `inverted_period`, `must_be_1`, `invalid_scope`, `not_an_object`, `inactive_or_unknown`, `length`, `html_not_allowed`). Em `POST /campaigns`, título/texto são conferidos antes do público: a resposta traz um código só.
5. **Corpo inválido nas rotas de preferência e de chave** (o contrato não dá código): `PUT /campaigns/sms_setting` com `enabled` que não é booleano → 422 `invalid_setting`; `PUT /citizen/contact_preferences/:citizen_id` sem nenhuma das duas chaves, ou com valor não booleano → 422 `invalid_preferences`.
6. **Cidade sem `city_profile`.** Toda cidade provisionada tem a linha (Plano 4), mas os bancos de teste nascem sem ela. Leitura: sem perfil, a chave vale `false` e o nome do SMS cai no `City#name` do catálogo. Escrita: `PUT /campaigns/sms_setting` sem perfil → 409 `city_profile_missing` (não acontece em cidade real).
7. **"Primeiro cidadão daquele telefone" (§5.3):** é o menor `citizen_id` **entre os que têm opt-in** naquele telefone. Sem isso, um telefone cujo menor id não optou nunca receberia SMS, mesmo com outro CPF optando.
8. **Mínimo no congelamento:** o `DispatchJob` insere as linhas num savepoint e conta os telefones **das linhas inseridas**; abaixo de 5, desfaz o savepoint e marca `failed`. É a mesma regra da §5.3, sem janela entre contar e inserir.
9. **Gateway antes da janela:** no `SmsBatchJob`, gateway não configurado vira `unavailable` na hora, mesmo fora da janela (adiar para as 8h só para então marcar `unavailable` atrasa o alerta do painel sem ganho).
10. **Revogação:** o consumidor existente `AnonymizeRevokedTriageJob` (`app/jobs/anonymize_revoked_triage_job.rb`) passa a chamar `Campaigns::ForgetRevokedRecipients`; nenhum binding novo em `consent.revoked`.
11. **Período por critério:** `protocol_period`, `triage_tier` e `triaged_not_attended` leem `triages.completed_at`; `triage_incomplete` lê `triages.created_at` (a triagem abandonada não tem conclusão); `attendance_outcome` lê `attendances.closed_at`; `appointment_no_show` lê `appointments.scheduled_at`.
12. **Quem enviou:** não há evento no clique de enviar (a spec §5.7 não lista); `dispatched_by_user_id` é gravado na transição e sai no payload de `campaign.dispatched` e de `campaign.failed`.
13. **Apagar campanha:** não há rota; o trigger aceita `DELETE` só de `draft` e recusa nos demais.
14. **`sms_available`** do cidadão é só a chave da cidade (não olha o gateway). Opt-in com a chave desligada é aceito e guardado (vale quando a cidade ligar).
15. **Texto sem HTML:** título ou texto com `<` seguido de letra, `/` ou `!` → `invalid_campaign` (`html_not_allowed`). Quebras de linha preservadas; pontas aparadas.
16. **Histórico no passado da semente e das specs** é gravado direto no banco por `lib/campaign_history.rb`: os comandos recusam data passada (horário só se marca para frente, desfecho só "agora"). Os CHECKs e triggers continuam valendo (só `INSERT`).
17. **Bairro e unidade do recorte precisam estar ativos** (achado do plano do dashboard, 2026-09-29; a spec é omissa): prévia, criar, editar, enviar e agendar recusam `neighborhood_ids[i]` inativo ou inexistente e `health_unit_id` do recorte `unit` inativo ou inexistente com 422 `invalid_audience`, `{ path: "/geo/neighborhood_ids/<i>" | "/geo/health_unit_id", message: "inactive_or_unknown" }` (`Campaigns::AudienceValidation`, Task 8). Unidades **dentro dos critérios** (`attendance_outcome.health_unit_id`, `appointment_request_open.target_unit_id`) não passam por isso: filtram histórico. Campanha já agendada cujo bairro é desativado depois ainda é congelada pelo `DispatchJob` (o público é o que era no agendamento).
18. **Escrita sem corpo:** `Authentication#require_json_for_cookie_writes` (`app/controllers/concerns/authentication.rb:81-89`) recusa escrita por cookie sem `application/json` com 415 `json_required`. `send`/`unschedule`/`cancel` não têm corpo no contrato: o dashboard manda `{}` e os request specs também (`json_post`).

## Global Constraints

- Tudo no banco de cada cidade (`db/city_migrate`, `db/city_schema.rb` à mão, `db/city_triggers.sql`); nada no banco de plataforma. Migração só de expansão e reversível (`down`).
- `db/city_triggers.sql` é re-executado por migrações antigas (replay do zero): cada trigger novo fica em `DO $do$ ... END $do$` guardado por `to_regclass('public.<tabela>')`.
- O dump `db/city_schema.rb` é feito à mão; o juiz é `spec/services/city_schema_spec.rb`. Use no dump o mesmo texto de CHECK da migração.
- **Branch com migração deixa os bancos de teste à frente da main:** ao voltar para a main, `DROP DATABASE rota_saude_test_city_a` e `rota_saude_test_city_b` e rode `city:test_databases` de novo (não encoste em `rota_saude_no_city_selected` nem nos bancos de dev).
- Título: sem espaços nas pontas, 3–120. Texto: 10–2000, texto simples, quebras de linha preservadas, sem HTML.
- Público: `version: 1`; recorte `city` | `unit` (`health_unit_id`) | `neighborhoods` (`neighborhood_ids`, 1 a 50); `clinical.all` com 0 a 7 critérios, combinados por E; datas `from`/`to` inclusivas no fuso da cidade, `from <= to`, `to` não no futuro.
- **Mínimo de 5 telefones distintos**, sempre, com ou sem filtro clínico.
- Agendamento: `send_at` de 5 minutos a 90 dias à frente.
- SMS: só entre **8h e 20h** (8 ≤ hora < 20) no fuso da cidade (`Time.zone`, `America/Sao_Paulo`); lotes de 100; 1 retentativa; `sms_error` até 200 caracteres e nunca com telefone.
- Texto do SMS, sempre: `Secretaria de Saúde de {cidade}: você tem um aviso novo. Acesse {base do wpda da cidade}/avisos` — sem identificador de campanha nem de cidadão.
- `config.x.sms_gateway`: `:log` em development, `:test` em test, ausente (`nil` → `Unconfigured`) em production/staging. `campaigns_sms_enabled` nasce `false`.
- Argumento de job **nunca** carrega telefone nem CPF: só `city_slug:` e ids; o job decifra dentro.
- Evento de domínio carrega só ids, contagens, booleanos e o JSON do público — nunca CPF, telefone ou lista de cidadãos. Todo evento novo declarado em `config/initializers/domain_events.rb` com `to: []`. `DomainEvents.publish(` costuma ser multilinha: confira call sites com `grep -rn -A3 "DomainEvents.publish(" app | grep -oE '"[a-z_]+\.[a-z_]+"' | sort -u`.
- Nenhuma resposta do dashboard traz a lista de destinatários: só contagens.
- `/admin/api` não muda (continua só leitura). A escrita mora em `/campaigns`.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (guarda `spec/architecture/current_city_assignment_spec.rb`); jobs usam `CityScopedJob#with_city` / `EachCityJob`.
- Todo comando devolve `Result` (`app/commands/result.rb`).
- Specs de request precisam de `type: :request`. Arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Constante dentro de `describe` vaza: use `let`/método. Nenhuma data fixa contra o relógio real: `travel_to`/`freeze_time` com horários derivados de `Time.zone.now`/`Time.zone.today`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem de merge: api antes do dashboard e do wpda.
- Crie o worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod12 -b feat/mod-12-campaigns origin/main
  cp apps/api/config/master.key apps/api/.claude/mod12/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container; o worktree é visto como `/rails/.claude/mod12`. Todo comando Rails/RSpec roda no container, a partir da raiz do monorepo:

  ```bash
  docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` do plano usa `-C apps/api/.claude/mod12` (caminhos relativos ao worktree).
- Depois da migração (Task 1), recarregue os bancos de teste: `docker compose exec -T -w /rails/.claude/mod12 api bin/rails city:test_databases`.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte completa:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec
  docker compose start worker
  ```

## Review Focus

1. **O público encolhe entre a prévia e o congelamento** (alguém revogou, um pedido foi fechado): a campanha vai a `failed`/`below_minimum` sem uma linha de destinatário e sem SMS; nunca sai para 4 telefones. Teste: Task 11 ("público que encolheu").
2. **Telefone da família com vários CPFs e opt-in misto:** um SMS só por telefone, para o menor `citizen_id` com opt-in; os demais com opt-in ficam `duplicate_phone`; a caixa do wpda mostra os avisos de todos os CPFs do telefone, com `cpf_masked`. Testes: Task 11 ("estado do SMS por destinatário") e Task 16 ("telefone com duas pessoas").
3. **Público malformado vindo do editor** (texto no lugar de lista, id que não é UUID, `2026-02-30`, chave com símbolo, critério sem `kind`): 422 `invalid_audience` com o caminho, nunca 500, nada gravado. Testes: Task 4 e Task 9 ("público malformado").
4. **Toque duplo em "marcar lido" / duas abas:** a segunda chamada responde `{ ok: true }` sem erro de trigger (`notice_read_at` só de NULL para valor). Teste: Task 16 ("marcar lido duas vezes").
5. **Cancelar ou desagendar enquanto o horário vence:** quem pega a linha primeiro vence (lock); campanha cancelada ou desagendada não é solta pelo `DueJob`, e `cancel` em `sending` é `invalid_transition`. Testes: Task 12 ("cancelar em sending") e Task 13 ("cancelada antes do horário").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `db/city_migrate/20260929100001_create_campaigns.rb` | tabelas, coluna, CHECK de papéis | 1 |
| `db/city_triggers.sql` | `campaigns_frozen_after_send`, `campaign_recipients_append_only` | 1 |
| `db/city_schema.rb` | dump à mão | 1 |
| `app/models/campaign.rb`, `campaign_recipient.rb`, `citizen_contact_preference.rb`, `citizen.rb` | modelos | 1 |
| `config/initializers/domain_events.rb` | 9 eventos declarados | 1 |
| `spec/support/campaign_helpers.rb`, `spec/rails_helper.rb` | helpers de spec | 1, 3, 5 |
| `spec/adr_pointers_spec.rb` | `VALID_RANGE` 1..24 | 1 |
| `app/models/membership.rb`, `app/policies/campaign_policy.rb` | papel e política | 2 |
| `app/services/sms_gateway.rb`, `app/services/campaigns/sms_text.rb`, `app/services/campaigns/sms_setting.rb`, `config/environments/{development,test}.rb` | SMS | 3 |
| `app/services/campaigns/audience_schema.rb` | validação do público | 4 |
| `lib/campaign_history.rb` | histórico no passado (specs e semente) | 5 |
| `app/services/campaigns/criteria.rb`, `app/services/campaigns/criteria/*.rb` | 7 critérios | 5, 6 |
| `app/services/campaigns/audience.rb` | composição, revogados, resumo | 7 |
| `app/commands/campaigns/{create,update}.rb`, `app/services/campaigns/content_validation.rb`, `app/services/campaigns/presenter.rb` | rascunho | 8 |
| `app/controllers/campaigns_controller.rb`, `config/routes.rb` | `/campaigns` | 9, 12 |
| `app/jobs/campaigns/sms_batch_job.rb` | SMS | 10 |
| `app/jobs/campaigns/dispatch_job.rb` | congelamento | 11 |
| `app/commands/campaigns/{send,schedule,unschedule,cancel}.rb` | transições | 12 |
| `app/jobs/campaigns/due_job.rb`, `config/recurring.yml` | agendados | 13 |
| `app/commands/campaigns/set_sms_enabled.rb`, `app/controllers/campaign_sms_settings_controller.rb` | chave de SMS | 14 |
| `app/commands/citizens/update_contact_preferences.rb`, `app/controllers/citizen_api/contact_preferences_controller.rb` | preferências | 15 |
| `app/controllers/citizen_api/notices_controller.rb` | caixa de avisos | 16 |
| `app/services/campaigns/forget_revoked_recipients.rb`, `app/jobs/anonymize_revoked_triage_job.rb` | revogação | 17 |
| `spec/invariants/campaign_invariants_spec.rb` | invariantes + mutação | 18 |
| `lib/campaign_crew.rb`, `db/seeds.rb` | semente de dev | 19 |

---

## Fatia 1 — F-12.1 (dados, papel, SMS plugável)

### Task 1: Migração, triggers, modelos, eventos declarados e guarda de ADR

**Files:**
- Create: `db/city_migrate/20260929100001_create_campaigns.rb`
- Modify: `db/city_triggers.sql` (acrescentar ao fim), `db/city_schema.rb`
- Create: `app/models/campaign.rb`, `app/models/campaign_recipient.rb`, `app/models/citizen_contact_preference.rb`
- Modify: `app/models/citizen.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Create: `spec/support/campaign_helpers.rb`; Modify: `spec/rails_helper.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/models/campaign_tables_guard_spec.rb`

**Interfaces:**
- Produces:
  - tabelas `campaigns`, `campaign_recipients`, `citizen_contact_preferences` (PK `citizen_id`); coluna `city_profile.campaigns_sms_enabled`; papel `campaign_manager` aceito pelo `ck_memberships_role`.
  - `Campaign::STATUSES`, `Campaign::TITLE_LENGTH` (3..120), `Campaign::BODY_LENGTH` (10..2000), `Campaign::MINIMUM_PHONES` (5), `Campaign::SEND_AT_MIN_LEAD` (5 min), `Campaign::SEND_AT_MAX_AHEAD` (90 dias); `Campaign#recipients`, `#created_by_user`, `#dispatched_by_user`, `#cancelled_by_user`.
  - `CampaignRecipient::SMS_STATUSES`; `CampaignRecipient#campaign`, `#citizen`.
  - `CitizenContactPreference.for(citizen_id) → CitizenContactPreference` (a linha, ou uma nova com os padrões, sem salvar).
  - `Citizen#contact_preference`, `Citizen#campaign_recipients`.
  - Eventos declarados: `campaign.created`, `campaign.scheduled`, `campaign.unscheduled`, `campaign.cancelled`, `campaign.dispatched`, `campaign.failed`, `campaign.sms_unavailable`, `citizen.contact_preferences_changed`, `city.campaigns_sms_toggled`.
  - Helpers de spec: `city_audience(*criteria) → Hash`; `draft_campaign!(by: nil, title:, body:, audience:) → Campaign`; `sent_campaign!(by: nil, title:, sms_enabled: false, dispatched_at: Time.current) → Campaign` (passa por `sending` e chega a `sent` por `update_columns`); `recipient!(campaign, citizen, sms_status: "pending") → CampaignRecipient`; `sql_in_savepoint(statement)`.

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod12/db/city_migrate | tail -2`
Expected: a última é `20260928100001_create_territory.rb`. Se houver outra mais nova, use um número maior que ela (e ajuste `define(version:)` no Step 9).

- [ ] **Step 2: Escreva os helpers de spec e registre-os**

```ruby
# spec/support/campaign_helpers.rb
# Módulo 12 (ADR 0024): campanhas prontas para specs. A campanha enviada chega a
# `sent` pelo mesmo caminho que o trigger aceita (draft → sending → sent).
module CampaignHelpers
  def city_audience(*criteria)
    { "version" => 1, "geo" => { "scope" => "city" }, "clinical" => { "all" => criteria } }
  end

  def draft_campaign!(by: nil, title: "Vacinação contra a gripe",
                      body: "A campanha de vacinação começa na segunda-feira.", audience: city_audience)
    by ||= User.create!(email_address: "autor-#{SecureRandom.hex(4)}@cidade.gov.br", password: "senha-segura-123")
    Campaign.create!(title: title, body: body, audience: audience, created_by_user: by)
  end

  def sent_campaign!(by: nil, title: "Vacinação contra a gripe", sms_enabled: false, dispatched_at: Time.current)
    campaign = draft_campaign!(by: by, title: title)
    campaign.update_columns(status: "sending", dispatched_by_user_id: campaign.created_by_user_id)
    campaign.update_columns(status: "sent", sms_enabled: sms_enabled, recipients_count: 0, phones_count: 0,
                            dispatched_at: dispatched_at)
    campaign
  end

  def recipient!(campaign, citizen, sms_status: "pending")
    CampaignRecipient.create!(campaign: campaign, citizen: citizen, sms_status: sms_status)
  end

  # SQL cru num savepoint: um erro do trigger não aborta a transação do exemplo.
  def sql_in_savepoint(statement)
    ApplicationRecord.transaction(requires_new: true) { ApplicationRecord.connection.execute(statement) }
  end
end

RSpec.configure { |c| c.include CampaignHelpers }
```

Em `spec/rails_helper.rb`, logo depois de `require_relative "support/territory_helpers"`:

```ruby
require_relative "support/campaign_helpers"
```

- [ ] **Step 3: Escreva a spec de guarda (falha: tabelas não existem)**

```ruby
# spec/models/campaign_tables_guard_spec.rb
require "rails_helper"

# Módulo 12 (ADR 0024; spec §3): a campanha enviada é imutável e do
# destinatário só mudam a leitura (uma vez) e o SMS. O banco recusa por SQL
# direto, sem passar pelo modelo.
RSpec.describe "Guardas das tabelas de campanha" do
  let(:author) { staff_with("autor@cidade.gov.br") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "status, título, texto, agendamento, falha e cancelamento são garantidos por CHECK" do
    campaign = draft_campaign!(by: author)
    {
      "status = 'lixo'" => /ck_campaigns_status/,
      "status = 'scheduled'" => /ck_campaigns_send_at/,
      "status = 'failed'" => /ck_campaigns_failure/,
      "failure_reason = 'below_minimum'" => /ck_campaigns_failure/,
      "cancelled_at = now()" => /ck_campaigns_cancelled/,
      "title = ' Gripe'" => /ck_campaigns_title/,
      "title = 'ab'" => /ck_campaigns_title/,
      "body = 'curto'" => /ck_campaigns_body/
    }.each do |assignment, error|
      expect { sql_in_savepoint("UPDATE campaigns SET #{assignment} WHERE id = '#{campaign.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, error), assignment
    end
  end

  it "rascunho muda livremente" do
    campaign = draft_campaign!(by: author)
    expect { campaign.update!(title: "Outro título", audience: city_audience) }.not_to raise_error
  end

  it "enviada ou cancelada: nenhuma coluna muda e não se apaga" do
    sent = sent_campaign!(by: author)
    [ "title = 'Mudou depois'", "recipients_count = 99", "status = 'draft'" ].each do |assignment|
      expect { sql_in_savepoint("UPDATE campaigns SET #{assignment} WHERE id = '#{sent.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /frozen after send/), assignment
    end
    expect { sql_in_savepoint("DELETE FROM campaigns WHERE id = '#{sent.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /only a draft may be deleted/)

    cancelled = draft_campaign!(by: author)
    cancelled.update_columns(status: "cancelled", cancelled_by_user_id: author.id, cancelled_at: Time.current)
    expect do
      sql_in_savepoint("UPDATE campaigns SET status = 'draft', cancelled_at = NULL, cancelled_by_user_id = NULL " \
                       "WHERE id = '#{cancelled.id}'")
    end.to raise_error(ActiveRecord::StatementInvalid, /frozen after send/)
  end

  it "em sending, só vai a sent ou failed, e só com as colunas do congelamento" do
    campaign = draft_campaign!(by: author)
    campaign.update_columns(status: "sending", dispatched_by_user_id: author.id)
    expect { sql_in_savepoint("UPDATE campaigns SET status = 'draft' WHERE id = '#{campaign.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /sending only moves to sent or failed/)
    expect { sql_in_savepoint("UPDATE campaigns SET status = 'sent', title = 'Outro título' WHERE id = '#{campaign.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /only the freeze columns/)
    expect { sql_in_savepoint("UPDATE campaigns SET status = 'failed', failure_reason = 'below_minimum' WHERE id = '#{campaign.id}'") }
      .not_to raise_error
  end

  it "destinatário: só a leitura (uma vez) e o SMS mudam; apagar é permitido" do
    campaign = sent_campaign!(by: author)
    row = recipient!(campaign, citizen)
    expect { row.update!(sms_status: "sent", sms_sent_at: Time.current) }.not_to raise_error
    expect { row.update!(notice_read_at: Time.current) }.not_to raise_error
    expect { sql_in_savepoint("UPDATE campaign_recipients SET notice_read_at = NULL WHERE id = '#{row.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /notice_read_at is set once/)
    expect { sql_in_savepoint("UPDATE campaign_recipients SET notice_read_at = now() WHERE id = '#{row.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /notice_read_at is set once/)
    other = draft_campaign!(by: author)
    expect { sql_in_savepoint("UPDATE campaign_recipients SET campaign_id = '#{other.id}' WHERE id = '#{row.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /only the reading and the SMS columns/)
    expect { sql_in_savepoint("UPDATE campaign_recipients SET sms_status = 'lixo' WHERE id = '#{row.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_campaign_recipients_sms_status/)
    expect { row.destroy! }.not_to raise_error
  end

  it "um destinatário por cidadão e campanha" do
    campaign = sent_campaign!(by: author)
    recipient!(campaign, citizen)
    expect { recipient!(campaign, citizen) }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "preferência de contato: padrões desligados, uma linha por cidadão" do
    preference = CitizenContactPreference.create!(citizen: citizen)
    expect(preference).to have_attributes(sms_opt_in: false, notices_muted: false, citizen_id: citizen.id)
    expect { CitizenContactPreference.create!(citizen: citizen) }.to raise_error(ActiveRecord::RecordNotUnique)

    other = Citizen.create!(cpf: "11144477735", phone: "+5541998765433")
    expect(CitizenContactPreference.for(other.id))
      .to have_attributes(sms_opt_in: false, notices_muted: false, new_record?: true)
    expect(CitizenContactPreference.for(citizen.id)).to eq(preference)
  end

  it "city_profile nasce com o SMS de campanha desligado" do
    expect(CityProfile.create!(name: "Cidade Teste").campaigns_sms_enabled).to be(false)
  end

  it "o banco aceita o papel campaign_manager" do
    user = staff_with("campanhas@cidade.gov.br")
    expect do
      sql_in_savepoint("INSERT INTO memberships (id, user_id, role, granted_at, created_at, updated_at) " \
                       "VALUES (gen_random_uuid(), '#{user.id}', 'campaign_manager', now(), now(), now())")
    end.not_to raise_error
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/models/campaign_tables_guard_spec.rb`
Expected: FAIL com `uninitialized constant Campaign`.

- [ ] **Step 5: Escreva a migração**

```ruby
# db/city_migrate/20260929100001_create_campaigns.rb
# Campanhas (ADR 0024; spec 2026-09-29-module-12-campaigns §3): a campanha, o
# público congelado no envio (campaign_recipients), as preferências de contato
# do cidadão, a chave de SMS da cidade e o papel campaign_manager. Triggers em
# db/city_triggers.sql, a mesma fonte que load_city_schema executa depois de
# carregar o dump. Só expansão: nada existente muda de forma.
class CreateCampaigns < ActiveRecord::Migration[8.1]
  ROLES_BEFORE = %w[citizen_verifier health_professional municipal_admin protocol_author protocol_publisher
                    protocol_reviewer viewer].freeze
  ROLES_AFTER = %w[campaign_manager citizen_verifier health_professional municipal_admin protocol_author
                   protocol_publisher protocol_reviewer viewer].freeze

  def up
    replace_roles_check(ROLES_AFTER)

    add_column :city_profile, :campaigns_sms_enabled, :boolean, null: false, default: false

    create_table :campaigns, id: :uuid do |t|
      t.string :title, limit: 120, null: false
      t.text :body, null: false
      t.jsonb :audience, null: false
      t.string :status, null: false, default: "draft"
      t.datetime :send_at
      t.string :failure_reason
      t.boolean :sms_enabled
      t.integer :recipients_count
      t.integer :phones_count
      t.references :created_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
      t.references :dispatched_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.datetime :dispatched_at
      t.references :cancelled_by_user, type: :uuid, foreign_key: { to_table: :users }, index: true
      t.datetime :cancelled_at
      t.timestamps
    end
    add_index :campaigns, %i[status send_at], name: "idx_campaigns_status_send_at"
    add_index :campaigns, :created_at, name: "index_campaigns_on_created_at"
    add_check_constraint :campaigns,
                         "status::text = ANY (ARRAY['draft', 'scheduled', 'sending', 'sent', 'cancelled', 'failed']::text[])",
                         name: "ck_campaigns_status"
    add_check_constraint :campaigns,
                         "length(title::text) >= 3 AND length(title::text) <= 120 AND title::text = btrim(title::text)",
                         name: "ck_campaigns_title"
    add_check_constraint :campaigns, "length(body) >= 10 AND length(body) <= 2000", name: "ck_campaigns_body"
    add_check_constraint :campaigns, "status::text <> 'scheduled'::text OR send_at IS NOT NULL",
                         name: "ck_campaigns_send_at"
    add_check_constraint :campaigns,
                         "(status::text = 'failed'::text) = (failure_reason IS NOT NULL) AND " \
                         "(failure_reason IS NULL OR failure_reason::text = 'below_minimum'::text)",
                         name: "ck_campaigns_failure"
    add_check_constraint :campaigns,
                         "(cancelled_by_user_id IS NULL) = (cancelled_at IS NULL) AND " \
                         "(status::text = 'cancelled'::text) = (cancelled_at IS NOT NULL)",
                         name: "ck_campaigns_cancelled"

    create_table :campaign_recipients, id: :uuid do |t|
      t.references :campaign, type: :uuid, null: false, foreign_key: true, index: false
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
      t.datetime :notice_read_at
      t.string :sms_status, null: false
      t.datetime :sms_sent_at
      t.string :sms_error, limit: 200
      t.datetime :created_at, null: false
    end
    add_index :campaign_recipients, %i[campaign_id citizen_id], unique: true, name: "idx_campaign_recipients_pair"
    add_index :campaign_recipients, %i[campaign_id sms_status], name: "idx_campaign_recipients_sms"
    add_check_constraint :campaign_recipients,
                         "sms_status::text = ANY (ARRAY['not_opted_in', 'duplicate_phone', 'pending', 'deferred', " \
                         "'sent', 'failed', 'unavailable']::text[])",
                         name: "ck_campaign_recipients_sms_status"

    create_table :citizen_contact_preferences, id: :uuid, primary_key: :citizen_id, default: nil do |t|
      t.boolean :sms_opt_in, null: false, default: false
      t.datetime :sms_opt_in_changed_at
      t.boolean :notices_muted, null: false, default: false
      t.timestamps
    end
    add_foreign_key :citizen_contact_preferences, :citizens

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    drop_table :citizen_contact_preferences
    drop_table :campaign_recipients
    drop_table :campaigns
    execute "DROP FUNCTION IF EXISTS rota_campaign_recipient_guard()"
    execute "DROP FUNCTION IF EXISTS rota_campaign_guard()"
    remove_column :city_profile, :campaigns_sms_enabled
    # Falha se já houver membership campaign_manager (memberships não se apagam).
    replace_roles_check(ROLES_BEFORE)
  end

  private

  # Mesma forma de 20260926000001: ANY (ARRAY[...]::text[]) sobrevive ao round-trip.
  def replace_roles_check(roles)
    remove_check_constraint :memberships, name: "ck_memberships_role"
    add_check_constraint :memberships, "role::text = ANY (ARRAY[#{roles.map { |r| "'#{r}'" }.join(', ')}]::text[])",
                         name: "ck_memberships_role"
  end
end
```

- [ ] **Step 6: Acrescente os triggers ao fim de `db/city_triggers.sql`**

```sql
-- campaigns (ADR 0024; spec 2026-09-29-module-12-campaigns §3.1): a campanha
-- enviada é prova do que a secretaria mandou e para quem — em sent, cancelled
-- ou failed nenhuma coluna muda. Em sending, a única saída é o congelamento
-- (sent ou failed), mudando só as colunas dele (status, sms_enabled,
-- recipients_count, phones_count, dispatched_at, failure_reason, updated_at).
-- draft e scheduled seguem pelos comandos. Só draft se apaga (não há rota;
-- defesa contra acidente). Sem trigger de TRUNCATE, como em memberships.
CREATE OR REPLACE FUNCTION rota_campaign_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    IF OLD.status = 'draft' THEN
      RETURN OLD;
    END IF;
    RAISE EXCEPTION 'campaigns: only a draft may be deleted';
  END IF;
  IF OLD.status IN ('sent', 'cancelled', 'failed') THEN
    RAISE EXCEPTION 'campaigns: frozen after send (status %)', OLD.status;
  END IF;
  IF OLD.status = 'sending' THEN
    IF NEW.status NOT IN ('sent', 'failed') THEN
      RAISE EXCEPTION 'campaigns: sending only moves to sent or failed';
    END IF;
    IF NEW.id IS DISTINCT FROM OLD.id
       OR NEW.title IS DISTINCT FROM OLD.title
       OR NEW.body IS DISTINCT FROM OLD.body
       OR NEW.audience IS DISTINCT FROM OLD.audience
       OR NEW.send_at IS DISTINCT FROM OLD.send_at
       OR NEW.created_by_user_id IS DISTINCT FROM OLD.created_by_user_id
       OR NEW.dispatched_by_user_id IS DISTINCT FROM OLD.dispatched_by_user_id
       OR NEW.cancelled_by_user_id IS DISTINCT FROM OLD.cancelled_by_user_id
       OR NEW.cancelled_at IS DISTINCT FROM OLD.cancelled_at
       OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
      RAISE EXCEPTION 'campaigns: while sending only the freeze columns change';
    END IF;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- campaign_recipients (§3.2): o público congelado. Só mudam a leitura do
-- aviso (de NULL para um valor, UMA vez) e as colunas do SMS. DELETE passa: a
-- revogação que anonimiza o cidadão apaga as linhas dele (§5.6).
CREATE OR REPLACE FUNCTION rota_campaign_recipient_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.campaign_id IS DISTINCT FROM OLD.campaign_id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'campaign_recipients: only the reading and the SMS columns change';
  END IF;
  IF OLD.notice_read_at IS NOT NULL AND NEW.notice_read_at IS DISTINCT FROM OLD.notice_read_at THEN
    RAISE EXCEPTION 'campaign_recipients: notice_read_at is set once';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.campaigns') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS campaigns_frozen_after_send ON campaigns';
    EXECUTE 'CREATE TRIGGER campaigns_frozen_after_send
      BEFORE UPDATE OR DELETE ON campaigns
      FOR EACH ROW EXECUTE FUNCTION rota_campaign_guard()';
  END IF;
  IF to_regclass('public.campaign_recipients') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS campaign_recipients_append_only ON campaign_recipients';
    EXECUTE 'CREATE TRIGGER campaign_recipients_append_only
      BEFORE UPDATE OR DELETE ON campaign_recipients
      FOR EACH ROW EXECUTE FUNCTION rota_campaign_recipient_guard()';
  END IF;
END
$do$;
```

- [ ] **Step 7: Escreva os modelos**

```ruby
# app/models/campaign.rb
# Campanha da secretaria (ADR 0024): aviso na caixa do wpda para um público
# descrito em JSON (Campaigns::AudienceSchema) e congelado no envio
# (campaign_recipients). Escrita só pelos comandos de app/commands/campaigns;
# depois do envio, imutável (trigger campaigns_frozen_after_send).
class Campaign < ApplicationRecord
  STATUSES = %w[draft scheduled sending sent cancelled failed].freeze
  TITLE_LENGTH = 3..120
  BODY_LENGTH = 10..2000
  # Telefones distintos (D8, D12): um telefone pode ter vários CPFs.
  MINIMUM_PHONES = 5
  SEND_AT_MIN_LEAD = 5.minutes
  SEND_AT_MAX_AHEAD = 90.days

  belongs_to :created_by_user, class_name: "User"
  belongs_to :dispatched_by_user, class_name: "User", optional: true
  belongs_to :cancelled_by_user, class_name: "User", optional: true
  has_many :recipients, class_name: "CampaignRecipient", dependent: :restrict_with_error

  normalizes :title, with: ->(value) { value.to_s.strip }
  normalizes :body, with: ->(value) { value.to_s.strip }
end
```

```ruby
# app/models/campaign_recipient.rb
# Um cidadão do público congelado (ADR 0024 §3.2). Só mudam a leitura do aviso
# e o estado do SMS (trigger campaign_recipients_append_only); a revogação
# apaga a linha. Sem updated_at, de propósito.
class CampaignRecipient < ApplicationRecord
  SMS_STATUSES = %w[not_opted_in duplicate_phone pending deferred sent failed unavailable].freeze

  belongs_to :campaign
  belongs_to :citizen
end
```

```ruby
# app/models/citizen_contact_preference.rb
# Preferências de contato do cidadão (ADR 0024 §3.3): opt-in do SMS (desligado
# por padrão) e silêncio dos avisos. Ausência de linha = os dois desligados.
class CitizenContactPreference < ApplicationRecord
  self.primary_key = "citizen_id"

  belongs_to :citizen

  def self.for(citizen_id)
    find_by(citizen_id: citizen_id) || new(citizen_id: citizen_id)
  end
end
```

Em `app/models/citizen.rb`, logo depois de `belongs_to :neighborhood, optional: true`:

```ruby

  # ADR 0024: preferências de contato e avisos recebidos.
  has_one :contact_preference, class_name: "CitizenContactPreference"
  has_many :campaign_recipients, dependent: :restrict_with_error
```

- [ ] **Step 8: Declare os eventos e suba a guarda de ADR**

Em `config/initializers/domain_events.rb`, antes do `end` final:

```ruby

  # Campanhas (ADR 0024; spec 2026-09-29 §5.7): trilha, só ids, contagens,
  # booleanos e o público (JSON sem dado pessoal); sem consumidor, de propósito.
  DomainEvents.bind "campaign.created", to: []
  DomainEvents.bind "campaign.scheduled", to: []
  DomainEvents.bind "campaign.unscheduled", to: []
  DomainEvents.bind "campaign.cancelled", to: []
  DomainEvents.bind "campaign.dispatched", to: []
  DomainEvents.bind "campaign.failed", to: []
  DomainEvents.bind "campaign.sms_unavailable", to: []
  DomainEvents.bind "citizen.contact_preferences_changed", to: []
  DomainEvents.bind "city.campaigns_sms_toggled", to: []
```

Ao fim de `spec/initializers/domain_events_bindings_spec.rb`:

```ruby

# Módulo 12 (ADR 0024): eventos das campanhas declarados, só trilha.
RSpec.describe "campaign event bindings (ADR 0024)" do
  it "declares every campaign event with no consumer" do
    names = %w[campaign.created campaign.scheduled campaign.unscheduled campaign.cancelled campaign.dispatched
               campaign.failed campaign.sms_unavailable citizen.contact_preferences_changed
               city.campaigns_sms_toggled]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Em `spec/adr_pointers_spec.rb`:
- no comentário do topo, `O v2 vai de 0001 a 0023 (0020 banco\n# por cidade, 0021 profissionais, 0022 painéis ao vivo, 0023 território)` passa a `O v2 vai de 0001 a 0024 (0020 banco\n# por cidade, 0021 profissionais, 0022 painéis ao vivo, 0023 território, 0024 campanhas)`;
- `VALID_RANGE = (1..23).freeze` passa a `VALID_RANGE = (1..24).freeze`;
- `it "only points at ADRs that exist in the v2 corpus (0001..0023)"` passa a `(0001..0024)`.

- [ ] **Step 9: Migre os bancos de dev e faça o dump à mão**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod12 api bin/rails "city:migrate[curitiba]"
docker compose exec -T -w /rails/.claude/mod12 api bin/rails "city:migrate[maringa]"
```
Expected: `[city:migrate] curitiba → 20260929100001` (e maringa).

Em `db/city_schema.rb`:
- `define(version: 2026_09_28_100001)` passa a `define(version: 2026_09_29_100001)`;
- em `create_table "city_profile"`, a primeira coluna passa a ser `t.boolean "campaigns_sms_enabled", default: false, null: false` (antes de `created_at`);
- em `create_table "memberships"`, a linha do CHECK passa a:

```ruby
    t.check_constraint "role::text = ANY (ARRAY['campaign_manager'::text, 'citizen_verifier'::text, 'health_professional'::text, 'municipal_admin'::text, 'protocol_author'::text, 'protocol_publisher'::text, 'protocol_reviewer'::text, 'viewer'::text])", name: "ck_memberships_role"
```

- entre `authors` e `city_profile`, as duas tabelas novas:

```ruby
  create_table "campaign_recipients", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "campaign_id", null: false
    t.uuid "citizen_id", null: false
    t.datetime "created_at", null: false
    t.datetime "notice_read_at"
    t.string "sms_error", limit: 200
    t.datetime "sms_sent_at"
    t.string "sms_status", null: false
    t.index ["campaign_id", "citizen_id"], name: "idx_campaign_recipients_pair", unique: true
    t.index ["campaign_id", "sms_status"], name: "idx_campaign_recipients_sms"
    t.index ["citizen_id"], name: "index_campaign_recipients_on_citizen_id"
    t.check_constraint "sms_status::text = ANY (ARRAY['not_opted_in', 'duplicate_phone', 'pending', 'deferred', 'sent', 'failed', 'unavailable']::text[])", name: "ck_campaign_recipients_sms_status"
  end

  create_table "campaigns", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.jsonb "audience", null: false
    t.text "body", null: false
    t.datetime "cancelled_at"
    t.uuid "cancelled_by_user_id"
    t.datetime "created_at", null: false
    t.uuid "created_by_user_id", null: false
    t.datetime "dispatched_at"
    t.uuid "dispatched_by_user_id"
    t.string "failure_reason"
    t.integer "phones_count"
    t.integer "recipients_count"
    t.datetime "send_at"
    t.boolean "sms_enabled"
    t.string "status", default: "draft", null: false
    t.string "title", limit: 120, null: false
    t.datetime "updated_at", null: false
    t.index ["cancelled_by_user_id"], name: "index_campaigns_on_cancelled_by_user_id"
    t.index ["created_at"], name: "index_campaigns_on_created_at"
    t.index ["created_by_user_id"], name: "index_campaigns_on_created_by_user_id"
    t.index ["dispatched_by_user_id"], name: "index_campaigns_on_dispatched_by_user_id"
    t.index ["status", "send_at"], name: "idx_campaigns_status_send_at"
    t.check_constraint "status::text = ANY (ARRAY['draft', 'scheduled', 'sending', 'sent', 'cancelled', 'failed']::text[])", name: "ck_campaigns_status"
    t.check_constraint "length(title::text) >= 3 AND length(title::text) <= 120 AND title::text = btrim(title::text)", name: "ck_campaigns_title"
    t.check_constraint "length(body) >= 10 AND length(body) <= 2000", name: "ck_campaigns_body"
    t.check_constraint "status::text <> 'scheduled'::text OR send_at IS NOT NULL", name: "ck_campaigns_send_at"
    t.check_constraint "(status::text = 'failed'::text) = (failure_reason IS NOT NULL) AND (failure_reason IS NULL OR failure_reason::text = 'below_minimum'::text)", name: "ck_campaigns_failure"
    t.check_constraint "(cancelled_by_user_id IS NULL) = (cancelled_at IS NULL) AND (status::text = 'cancelled'::text) = (cancelled_at IS NOT NULL)", name: "ck_campaigns_cancelled"
  end
```

- entre `city_profile` e `citizen_sessions` (ordem alfabética: `citizen_contact_preferences` < `citizen_sessions`):

```ruby
  create_table "citizen_contact_preferences", primary_key: "citizen_id", id: :uuid, default: nil, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.boolean "notices_muted", default: false, null: false
    t.boolean "sms_opt_in", default: false, null: false
    t.datetime "sms_opt_in_changed_at"
    t.datetime "updated_at", null: false
  end
```

- os `add_foreign_key` novos na lista final, em ordem alfabética: `add_foreign_key "campaign_recipients", "campaigns"`, `add_foreign_key "campaign_recipients", "citizens"`, `add_foreign_key "campaigns", "users", column: "cancelled_by_user_id"`, `add_foreign_key "campaigns", "users", column: "created_by_user_id"`, `add_foreign_key "campaigns", "users", column: "dispatched_by_user_id"`, `add_foreign_key "citizen_contact_preferences", "citizens"`.

O juiz é a paridade (Step 10): se ela acusar diferença, compare com o banco migrado —

```bash
docker compose exec -T db psql -U postgres -d <banco_de_curitiba> -c "\d+ campaigns" -c "\d+ campaign_recipients" -c "\d+ citizen_contact_preferences"
```

(o nome do banco de Curitiba sai de `bin/rails runner 'puts URI(City.find_by(slug: "curitiba").database_url).path.delete_prefix("/")'`; não cole a URL em lugar nenhum) — e ajuste o dump, nunca a migração, salvo erro de verdade nela.

- [ ] **Step 10: Recarregue os bancos de teste e rode paridade, guarda de ADR, bindings e a spec de guarda**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod12 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/city_schema_spec.rb spec/adr_pointers_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/models/campaign_tables_guard_spec.rb
```
Expected: PASS.

- [ ] **Step 11: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add db/city_migrate/20260929100001_create_campaigns.rb db/city_triggers.sql db/city_schema.rb app/models/campaign.rb app/models/campaign_recipient.rb app/models/citizen_contact_preference.rb app/models/citizen.rb config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/support/campaign_helpers.rb spec/rails_helper.rb spec/adr_pointers_spec.rb spec/models/campaign_tables_guard_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add campaign tables, frozen-after-send triggers and campaign events

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Papel `campaign_manager` e `CampaignPolicy`

**Files:**
- Modify: `app/models/membership.rb`
- Create: `app/policies/campaign_policy.rb`
- Modify: `spec/models/membership_roles_spec.rb`
- Test: `spec/policies/campaign_policy_spec.rb`, `spec/requests/campaign_manager_grant_spec.rb`

**Interfaces:**
- Consumes: CHECK de papéis da Task 1.
- Produces: `"campaign_manager"` em `Membership::ROLES` e `Membership::PRIVILEGED_ROLES`; `CampaignPolicy.new(user, nil)#manage?` (só `campaign_manager`), `#read_sms_setting?` (`campaign_manager` ou `municipal_admin`), `#write_sms_setting?` (só `municipal_admin`); `user` `nil` → tudo `false`.

- [ ] **Step 1: Escreva as specs**

Ao fim do `RSpec.describe` de `spec/models/membership_roles_spec.rb`, antes do `end`:

```ruby

  it "conhece o papel campaign_manager e o trata como privilegiado (ADR 0024)" do
    expect(described_class::ROLES).to include("campaign_manager")
    expect(described_class::PRIVILEGED_ROLES).to include("campaign_manager")
    user = User.create!(email_address: "campanhas@cidade.gov.br", password: "senha-segura-123")
    expect { described_class.create!(user: user, role: "campaign_manager", granted_at: Time.current) }.not_to raise_error
  end
```

```ruby
# spec/policies/campaign_policy_spec.rb
require "rails_helper"

RSpec.describe CampaignPolicy do
  def policy_for(*roles)
    described_class.new(staff_with("p-#{SecureRandom.hex(4)}@cidade.gov.br", *roles), nil)
  end

  it "só campaign_manager monta e envia campanha" do
    expect(policy_for("campaign_manager").manage?).to be(true)
    (Membership::ROLES - %w[campaign_manager]).each do |role|
      expect(policy_for(role).manage?).to be(false), role
    end
  end

  it "a chave de SMS: campaign_manager e municipal_admin leem; só municipal_admin muda" do
    expect(policy_for("campaign_manager")).to have_attributes(read_sms_setting?: true, write_sms_setting?: false)
    expect(policy_for("municipal_admin")).to have_attributes(read_sms_setting?: true, write_sms_setting?: true)
    expect(policy_for("viewer")).to have_attributes(read_sms_setting?: false, write_sms_setting?: false)
  end

  it "sem usuário (sessão de operador por grant): nada" do
    expect(described_class.new(nil, nil))
      .to have_attributes(manage?: false, read_sms_setting?: false, write_sms_setting?: false)
  end
end
```

```ruby
# spec/requests/campaign_manager_grant_spec.rb
require "rails_helper"

# ADR 0024: campaign_manager é privilegiado — conceder pede step-up, como os
# demais de Membership::PRIVILEGED_ROLES (SetupController#privileged_role?).
RSpec.describe "Conceder campaign_manager", type: :request do
  let(:admin) do
    staff_with("admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  let(:person) { staff_with("comunicacao@cidade.gov.br", "viewer") }

  it "sem step-up: 401 mfa_required e nada é concedido" do
    sign_in_as(admin)
    post "/setup/memberships", params: { user_id: person.id, role: "campaign_manager" }, as: :json
    expect(response).to have_http_status(:unauthorized)
    expect(JSON.parse(response.body)).to eq("error" => "mfa_required")
    expect(person.reload.has_role?(:campaign_manager)).to be(false)
  end

  it "com step-up: 201 e o papel vale" do
    sign_in_as(admin).update!(mfa_verified_at: Time.current)
    post "/setup/memberships", params: { user_id: person.id, role: "campaign_manager" }, as: :json
    expect(response).to have_http_status(:created)
    expect(person.reload.has_role?(:campaign_manager)).to be(true)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/models/membership_roles_spec.rb spec/policies/campaign_policy_spec.rb spec/requests/campaign_manager_grant_spec.rb`
Expected: FAIL (`uninitialized constant CampaignPolicy`; `ROLES` sem `campaign_manager`; a concessão com step-up volta 422 `invalid_role`).

- [ ] **Step 3: Implemente**

Em `app/models/membership.rb`, troque as duas constantes por:

```ruby
  ROLES = %w[campaign_manager citizen_verifier health_professional municipal_admin protocol_author
             protocol_publisher protocol_reviewer viewer].freeze
```

e, no comentário de `PRIVILEGED_ROLES`, depois do parágrafo de `health_professional`:

```ruby
  # campaign_manager (ADR 0024): monta públicos com dado de saúde e fala em
  # nome da secretaria — step-up para conceder; o mantenedor não concede.
  PRIVILEGED_ROLES = %w[municipal_admin protocol_reviewer citizen_verifier health_professional
                        campaign_manager].freeze
```

```ruby
# app/policies/campaign_policy.rb
# Campanhas (ADR 0024; spec 2026-09-29 §6.1): o campaign_manager monta, agenda,
# cancela e envia; a chave de SMS da cidade é lida por ele e pelo
# municipal_admin, e só o municipal_admin a muda.
class CampaignPolicy < ApplicationPolicy
  def manage?
    role?(:campaign_manager)
  end

  def read_sms_setting?
    role?(:campaign_manager) || role?(:municipal_admin)
  end

  def write_sms_setting?
    role?(:municipal_admin)
  end
end
```

- [ ] **Step 4: Rode e veja passar, com a regressão de papéis**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/models/membership_roles_spec.rb spec/policies/campaign_policy_spec.rb spec/requests/campaign_manager_grant_spec.rb spec/commands/grant_role_spec.rb spec/commands/invite_member_privileged_roles_spec.rb spec/requests/setup_grant_role_spec.rb spec/requests/admin/api/membership_gate_spec.rb spec/requests/territory_spec.rb`
Expected: PASS. (`grant_role_spec` e `invite_member_privileged_roles_spec` iteram `PRIVILEGED_ROLES` e passam a provar que o mantenedor não concede nem convida `campaign_manager`; `membership_gate_spec` prova que o papel lê `/admin/api`, como todo papel ativo.)

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/models/membership.rb app/policies/campaign_policy.rb spec/models/membership_roles_spec.rb spec/policies/campaign_policy_spec.rb spec/requests/campaign_manager_grant_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add privileged campaign_manager role and campaign policy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: `SmsGateway`, texto fixo do SMS e leitura da chave

**Files:**
- Create: `app/services/sms_gateway.rb`, `app/services/campaigns/sms_text.rb`, `app/services/campaigns/sms_setting.rb`
- Modify: `config/environments/development.rb`, `config/environments/test.rb`
- Modify: `spec/support/campaign_helpers.rb`
- Test: `spec/services/sms_gateway_spec.rb`, `spec/services/campaigns/sms_text_spec.rb`

**Interfaces:**
- Produces:
  - `SmsGateway.deliver(phone:, body:)`; `SmsGateway.configured? → Boolean`; `SmsGateway::Unavailable`; `SmsGateway::Test.deliveries → [{ phone:, body: }]`, `SmsGateway::Test.reset!`. Backend por `Rails.configuration.x.sms_gateway` (`:log`, `:test`, outro → `Unconfigured`).
  - `Campaigns::SmsText.body(city) → String`; `Campaigns::SmsText.link(city) → String` (`<CityPublicUrl.wpda(city)>avisos`).
  - `Campaigns::SmsSetting.enabled? → Boolean` (`false` sem `city_profile`).
  - Helpers de spec: `with_sms_gateway(value) { }`; `sms_profile!(enabled:) → CityProfile`; antes de cada exemplo, `SmsGateway::Test.reset!`.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/services/sms_gateway_spec.rb
require "rails_helper"

RSpec.describe SmsGateway do
  it "test (backend da suíte): guarda a entrega e está configurado" do
    described_class.deliver(phone: "+5541998765432", body: "Aviso")
    expect(SmsGateway::Test.deliveries).to eq([ { phone: "+5541998765432", body: "Aviso" } ])
    expect(described_class.configured?).to be(true)
  end

  it "log (development): telefone mascarado e o texto, nunca o número inteiro" do
    with_sms_gateway(:log) do
      logged = []
      allow(Rails.logger).to receive(:info) { |message| logged << message }
      described_class.deliver(phone: "+5541998765432", body: "Aviso")
      expect(logged).to eq([ "[sms] (**) *****-5432 Aviso" ])
      expect(described_class.configured?).to be(true)
    end
  end

  it "sem backend (production): não configurado e deliver levanta Unavailable" do
    with_sms_gateway(nil) do
      expect(described_class.configured?).to be(false)
      expect { described_class.deliver(phone: "+5541998765432", body: "Aviso") }
        .to raise_error(SmsGateway::Unavailable)
    end
  end
end
```

```ruby
# spec/services/campaigns/sms_text_spec.rb
require "rails_helper"

RSpec.describe Campaigns::SmsText do
  let(:link) { "#{CityPublicUrl.wpda_base_for_slug(TEST_CITY_A.slug)}/wpda/avisos" }

  it "texto fixo com o nome do perfil da cidade e o link da caixa, sem identificador" do
    CityProfile.create!(name: "Curitiba")
    expect(described_class.body(TEST_CITY_A))
      .to eq("Secretaria de Saúde de Curitiba: você tem um aviso novo. Acesse #{link}")
    expect(described_class.link(TEST_CITY_A)).to eq(link)
  end

  it "sem perfil, usa o nome do catálogo" do
    expect(described_class.body(TEST_CITY_A))
      .to eq("Secretaria de Saúde de #{TEST_CITY_A.name}: você tem um aviso novo. Acesse #{link}")
  end

  describe Campaigns::SmsSetting do
    it "desligada sem perfil e por padrão; ligada quando o perfil liga" do
      expect(described_class.enabled?).to be(false)
      profile = CityProfile.create!(name: "Curitiba")
      expect(described_class.enabled?).to be(false)
      profile.update!(campaigns_sms_enabled: true)
      expect(described_class.enabled?).to be(true)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/sms_gateway_spec.rb spec/services/campaigns/sms_text_spec.rb`
Expected: FAIL (`uninitialized constant SmsGateway`, `undefined method 'with_sms_gateway'`).

- [ ] **Step 3: Acrescente os helpers de SMS**

Em `spec/support/campaign_helpers.rb`, dentro do `module CampaignHelpers`, antes do `end`:

```ruby

  def with_sms_gateway(value)
    previous = Rails.configuration.x.sms_gateway
    Rails.configuration.x.sms_gateway = value
    yield
  ensure
    Rails.configuration.x.sms_gateway = previous
  end

  def sms_profile!(enabled:)
    (CityProfile.current || CityProfile.new(name: "Curitiba")).tap { |p| p.update!(campaigns_sms_enabled: enabled) }
  end
```

e troque a última linha do arquivo por:

```ruby
RSpec.configure do |c|
  c.include CampaignHelpers
  c.before { SmsGateway::Test.reset! }
end
```

- [ ] **Step 4: Implemente**

```ruby
# app/services/sms_gateway.rb
# Envio de SMS das campanhas (ADR 0024; spec 2026-09-29 §5.5). O backend vem de
# `config.x.sms_gateway`: :log (development), :test (test); ausente (production
# e staging) → Unconfigured, e o deploy não envia SMS. O provedor real é um
# backend novo, escolhido no go-live. O OtpSender segue separado; os dois se
# unificam quando o provedor for escolhido.
module SmsGateway
  class Unavailable < StandardError; end

  def self.deliver(phone:, body:)
    backend.deliver(phone: phone, body: body)
  end

  def self.configured?
    backend.configured?
  end

  def self.backend
    case Rails.configuration.x.sms_gateway
    when :log  then Log
    when :test then Test
    else Unconfigured
    end
  end

  # Development: o texto aparece no log do api, com o telefone mascarado como
  # no OtpSender.
  module Log
    def self.configured? = true

    def self.deliver(phone:, body:)
      Rails.logger.info("[sms] #{CitizenIdentity::Phone.mask(phone)} #{body}")
    end
  end

  module Test
    def self.configured? = true

    def self.deliveries
      @deliveries ||= []
    end

    def self.deliver(phone:, body:)
      deliveries << { phone: phone, body: body }
    end

    def self.reset!
      deliveries.clear
    end
  end

  module Unconfigured
    def self.configured? = false

    def self.deliver(**)
      raise Unavailable, "no SMS provider configured"
    end
  end
end
```

```ruby
# app/services/campaigns/sms_text.rb
# O texto do SMS de campanha é sempre este (ADR 0024, D6): nunca carrega o
# conteúdo do aviso nem identificador — aparece na tela de bloqueio.
module Campaigns
  module SmsText
    module_function

    def body(city)
      name = CityProfile.current&.name.presence || city.name
      "Secretaria de Saúde de #{name}: você tem um aviso novo. Acesse #{link(city)}"
    end

    def link(city)
      "#{CityPublicUrl.wpda(city)}avisos"
    end
  end
end
```

```ruby
# app/services/campaigns/sms_setting.rb
# Chave de SMS da cidade (ADR 0024 §3.4): nasce desligada. Sem city_profile
# (bancos de teste), vale desligada.
module Campaigns
  module SmsSetting
    def self.enabled?
      CityProfile.current&.campaigns_sms_enabled == true
    end
  end
end
```

Em `config/environments/development.rb`, logo depois de `config.x.otp_sender = :log`:

```ruby
  # Campanhas (ADR 0024): o SMS vai para o log do api, com telefone mascarado.
  config.x.sms_gateway = :log
```

Em `config/environments/test.rb`, logo depois de `config.x.otp_sender = :test`:

```ruby
  config.x.sms_gateway = :test
```

(`production.rb` e `staging.rb` não mudam: sem a chave, o backend é `Unconfigured`.)

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/sms_gateway_spec.rb spec/services/campaigns/sms_text_spec.rb spec/architecture/staging_environment_parity_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/services/sms_gateway.rb app/services/campaigns/sms_text.rb app/services/campaigns/sms_setting.rb config/environments/development.rb config/environments/test.rb spec/support/campaign_helpers.rb spec/services/sms_gateway_spec.rb spec/services/campaigns/sms_text_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add pluggable SMS gateway and fixed campaign SMS text

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 2 — F-12.2 (público: schema, critérios, composição, mínimo)

### Task 4: Schema do público (`Campaigns::AudienceSchema`)

**Files:**
- Create: `app/services/campaigns/audience_schema.rb`
- Test: `spec/services/campaigns/audience_schema_spec.rb`

**Interfaces:**
- Produces: `Campaigns::AudienceSchema.errors(input) → Array<{ path: String, message: String }>` (vazio = válido; aceita Hash com chaves de texto ou símbolo, `HashWithIndifferentAccess`; qualquer outra coisa → `[{ path: "/", message: "not_an_object" }]`); `.valid?(input) → Boolean`; `.normalize(input) → Hash` (JSON puro, chaves de texto); `AudienceSchema::CRITERIA` (os 7 `kind` com campos obrigatórios e opcionais).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/campaigns/audience_schema_spec.rb
require "rails_helper"

RSpec.describe Campaigns::AudienceSchema do
  let(:uuid) { SecureRandom.uuid }
  let(:from) { (Time.zone.today - 30).iso8601 }
  let(:to) { (Time.zone.today - 1).iso8601 }

  def audience(geo: { "scope" => "city" }, all: [])
    { "version" => 1, "geo" => geo, "clinical" => { "all" => all } }
  end

  def errors_of(value) = described_class.errors(value)
  def paths(value) = errors_of(value).map { |e| e[:path] }

  it "aceita os três recortes e cada um dos sete critérios" do
    valid = [
      audience,
      audience(geo: { "scope" => "unit", "health_unit_id" => uuid }),
      audience(geo: { "scope" => "neighborhoods", "neighborhood_ids" => [ uuid ] }),
      audience(all: [
        { "kind" => "protocol_period", "protocol_name" => "triage-respiratoria", "from" => from, "to" => to },
        { "kind" => "triage_tier", "tiers" => [ "alta" ], "from" => from, "to" => to },
        { "kind" => "triage_incomplete", "from" => from, "to" => to },
        { "kind" => "attendance_outcome", "outcomes" => %w[discharged left], "health_unit_id" => uuid,
          "from" => from, "to" => to },
        { "kind" => "triaged_not_attended", "from" => from, "to" => to },
        { "kind" => "appointment_no_show", "from" => from, "to" => to },
        { "kind" => "appointment_request_open", "kinds" => [ "return" ], "target_unit_id" => uuid }
      ]),
      audience(all: [ { "kind" => "appointment_request_open" } ])
    ]
    valid.each { |a| expect(errors_of(a)).to eq([]), a.inspect }
  end

  it "aceita chaves de símbolo e período de um dia só, hoje" do
    today = Time.zone.today.iso8601
    value = { version: 1, geo: { scope: "city" }, clinical: { all: [ { kind: "appointment_no_show", from: today, to: today } ] } }
    expect(errors_of(value)).to eq([])
    expect(described_class.valid?(value)).to be(true)
  end

  it "recusa o que não é objeto, versão errada, chave extra e partes faltando" do
    expect(errors_of(nil)).to eq([ { path: "/", message: "not_an_object" } ])
    expect(paths([ 1 ])).to eq([ "/" ])
    expect(errors_of(audience.merge("version" => 2))).to eq([ { path: "/version", message: "must_be_1" } ])
    expect(errors_of(audience.merge("extra" => true))).to eq([ { path: "/extra", message: "unknown_key" } ])
    expect(errors_of(audience.except("clinical"))).to eq([ { path: "/clinical", message: "required" } ])
    expect(errors_of(audience(geo: "city"))).to eq([ { path: "/geo", message: "not_an_object" } ])
  end

  it "recorte: escopo desconhecido, unidade sem UUID, bairros vazios, repetidos ou mais de 50" do
    expect(errors_of(audience(geo: { "scope" => "estado" }))).to eq([ { path: "/geo/scope", message: "invalid_scope" } ])
    expect(errors_of(audience(geo: { "scope" => "unit" }))).to eq([ { path: "/geo/health_unit_id", message: "required" } ])
    expect(errors_of(audience(geo: { "scope" => "unit", "health_unit_id" => "ubs-1" })))
      .to eq([ { path: "/geo/health_unit_id", message: "invalid_uuid" } ])
    expect(errors_of(audience(geo: { "scope" => "city", "neighborhood_ids" => [ uuid ] })))
      .to eq([ { path: "/geo/neighborhood_ids", message: "unknown_key" } ])
    {
      [] => "empty", Array.new(51) { SecureRandom.uuid } => "too_many", [ uuid, uuid ] => "duplicate", "Centro" => "not_a_list"
    }.each do |ids, message|
      expect(errors_of(audience(geo: { "scope" => "neighborhoods", "neighborhood_ids" => ids })))
        .to eq([ { path: "/geo/neighborhood_ids", message: message } ]), message
    end
    expect(errors_of(audience(geo: { "scope" => "neighborhoods", "neighborhood_ids" => [ uuid, "x" ] })))
      .to eq([ { path: "/geo/neighborhood_ids/1", message: "invalid_uuid" } ])
  end

  it "critérios: mais de 7, kind desconhecido, campo extra, campo faltando, valor fora da lista" do
    no_show = { "kind" => "appointment_no_show", "from" => from, "to" => to }
    expect(errors_of(audience(all: Array.new(8) { no_show }))).to eq([ { path: "/clinical/all", message: "too_many" } ])
    expect(errors_of(audience(all: [ { "kind" => "idade" } ]))).to eq([ { path: "/clinical/all/0/kind", message: "invalid_kind" } ])
    expect(errors_of(audience(all: [ { "from" => from } ]))).to eq([ { path: "/clinical/all/0/kind", message: "invalid_kind" } ])
    expect(errors_of(audience(all: [ no_show.merge("cpf" => "52998224725") ])))
      .to eq([ { path: "/clinical/all/0/cpf", message: "unknown_key" } ])
    expect(errors_of(audience(all: [ no_show.except("to") ]))).to eq([ { path: "/clinical/all/0/to", message: "required" } ])
    expect(errors_of(audience(all: [ { "kind" => "attendance_outcome", "outcomes" => [ "curado" ], "from" => from, "to" => to } ])))
      .to eq([ { path: "/clinical/all/0/outcomes/0", message: "invalid_value" } ])
    expect(errors_of(audience(all: [ { "kind" => "triage_tier", "tiers" => [], "from" => from, "to" => to } ])))
      .to eq([ { path: "/clinical/all/0/tiers", message: "empty" } ])
    expect(errors_of(audience(all: [ { "kind" => "protocol_period", "protocol_name" => "  ", "from" => from, "to" => to } ])))
      .to eq([ { path: "/clinical/all/0/protocol_name", message: "invalid_value" } ])
    expect(errors_of(audience(all: [ { "kind" => "appointment_request_open", "kinds" => [ "exame" ] } ])))
      .to eq([ { path: "/clinical/all/0/kinds/0", message: "invalid_value" } ])
    expect(errors_of(audience(all: [ "appointment_no_show" ]))).to eq([ { path: "/clinical/all/0", message: "not_an_object" } ])
  end

  it "período: data inválida, futura ou invertida" do
    base = { "kind" => "triage_incomplete" }
    tomorrow = (Time.zone.today + 1).iso8601
    {
      base.merge("from" => "2026-02-30", "to" => to) => { path: "/clinical/all/0/from", message: "invalid_date" },
      base.merge("from" => from, "to" => "ontem") => { path: "/clinical/all/0/to", message: "invalid_date" },
      base.merge("from" => from, "to" => 20_260_101) => { path: "/clinical/all/0/to", message: "invalid_date" },
      base.merge("from" => from, "to" => tomorrow) => { path: "/clinical/all/0/to", message: "future_date" },
      base.merge("from" => to, "to" => from) => { path: "/clinical/all/0/from", message: "inverted_period" }
    }.each do |criterion, expected|
      expect(errors_of(audience(all: [ criterion ]))).to eq([ expected ]), criterion.inspect
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/audience_schema_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::AudienceSchema`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/campaigns/audience_schema.rb
# Formato do público, versão 1 (ADR 0024; spec 2026-09-29 §4.1). Validação à
# mão, com caminho determinístico (ponteiro JSON relativo à raiz do público)
# para o editor do dashboard marcar o campo. Critério novo = entrada em
# CRITERIA + classe em Campaigns::Criteria. O JSON já comporta `clinical.any`
# depois; hoje só `all` (E).
module Campaigns
  module AudienceSchema
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
    DATE = /\A\d{4}-\d{2}-\d{2}\z/
    MAX_CRITERIA = 7
    MAX_NEIGHBORHOODS = 50
    MAX_TIERS = 20
    MAX_TEXT = 200

    GEO = { "city" => [], "unit" => %w[health_unit_id], "neighborhoods" => %w[neighborhood_ids] }.freeze

    CRITERIA = {
      "protocol_period" => { required: %w[protocol_name from to], optional: [] },
      "triage_tier" => { required: %w[tiers from to], optional: [] },
      "triage_incomplete" => { required: %w[from to], optional: [] },
      "attendance_outcome" => { required: %w[outcomes from to], optional: %w[health_unit_id] },
      "triaged_not_attended" => { required: %w[from to], optional: [] },
      "appointment_no_show" => { required: %w[from to], optional: [] },
      "appointment_request_open" => { required: [], optional: %w[kinds target_unit_id] }
    }.freeze

    module_function

    def errors(input)
      audience = normalize(input)
      return [ error("/", "not_an_object") ] unless audience.is_a?(Hash)

      found = shape_errors(audience, required: %w[version geo clinical], optional: [], path: "")
      found << error("/version", "must_be_1") if audience.key?("version") && audience["version"] != 1
      found.concat(geo_errors(audience["geo"])) if audience.key?("geo")
      found.concat(clinical_errors(audience["clinical"])) if audience.key?("clinical")
      found
    end

    def valid?(input)
      errors(input).empty?
    end

    # Símbolos e HashWithIndifferentAccess viram JSON puro, com chaves de texto.
    def normalize(input)
      JSON.parse(input.to_json)
    end

    def shape_errors(object, required:, optional:, path:)
      missing = (required - object.keys).map { |key| error("#{path}/#{key}", "required") }
      unknown = (object.keys - required - optional).map { |key| error("#{path}/#{key}", "unknown_key") }
      missing + unknown
    end

    def geo_errors(geo)
      return [ error("/geo", "not_an_object") ] unless geo.is_a?(Hash)
      return [ error("/geo/scope", "invalid_scope") ] unless GEO.key?(geo["scope"])

      scope = geo["scope"]
      found = shape_errors(geo, required: [ "scope", *GEO[scope] ], optional: [], path: "/geo")
      if scope == "unit" && geo.key?("health_unit_id") && !uuid?(geo["health_unit_id"])
        found << error("/geo/health_unit_id", "invalid_uuid")
      end
      if scope == "neighborhoods" && geo.key?("neighborhood_ids")
        found.concat(list_errors(geo["neighborhood_ids"], "/geo/neighborhood_ids", max: MAX_NEIGHBORHOODS) do |id|
          uuid?(id) ? nil : "invalid_uuid"
        end)
      end
      found
    end

    def clinical_errors(clinical)
      return [ error("/clinical", "not_an_object") ] unless clinical.is_a?(Hash)

      found = shape_errors(clinical, required: %w[all], optional: [], path: "/clinical")
      return found unless clinical.key?("all")

      all = clinical["all"]
      return found << error("/clinical/all", "not_a_list") unless all.is_a?(Array)
      return found << error("/clinical/all", "too_many") if all.size > MAX_CRITERIA

      all.each_with_index { |criterion, i| found.concat(criterion_errors(criterion, "/clinical/all/#{i}")) }
      found
    end

    def criterion_errors(criterion, path)
      return [ error(path, "not_an_object") ] unless criterion.is_a?(Hash)

      rules = CRITERIA[criterion["kind"]]
      return [ error("#{path}/kind", "invalid_kind") ] unless rules

      found = shape_errors(criterion, required: [ "kind", *rules[:required] ], optional: rules[:optional], path: path)
      (rules[:required] + rules[:optional]).each do |key|
        found.concat(field_errors(key, criterion[key], "#{path}/#{key}")) if criterion.key?(key)
      end
      found.concat(period_errors(criterion, path)) if found.empty? && criterion.key?("from")
      found
    end

    def field_errors(key, value, path)
      case key
      when "protocol_name" then text?(value, MAX_TEXT) ? [] : [ error(path, "invalid_value") ]
      when "tiers" then list_errors(value, path, max: MAX_TIERS) { |v| text?(v, 50) ? nil : "invalid_value" }
      when "outcomes"
        list_errors(value, path, max: Attendance::OUTCOMES.size) { |v| Attendance::OUTCOMES.include?(v) ? nil : "invalid_value" }
      when "kinds"
        list_errors(value, path, max: AppointmentRequest::KINDS.size) do |v|
          AppointmentRequest::KINDS.include?(v) ? nil : "invalid_value"
        end
      when "health_unit_id", "target_unit_id" then uuid?(value) ? [] : [ error(path, "invalid_uuid") ]
      when "from", "to" then date(value) ? [] : [ error(path, "invalid_date") ]
      else []
      end
    end

    def list_errors(value, path, max:)
      return [ error(path, "not_a_list") ] unless value.is_a?(Array)
      return [ error(path, "empty") ] if value.empty?
      return [ error(path, "too_many") ] if value.size > max
      return [ error(path, "duplicate") ] if value.uniq.size != value.size

      value.each_with_index.filter_map do |item, i|
        message = yield(item)
        error("#{path}/#{i}", message) if message
      end
    end

    # Datas inclusivas no fuso da cidade; "hoje" é Time.zone.today.
    def period_errors(criterion, path)
      from = date(criterion["from"])
      to = date(criterion["to"])
      return [ error("#{path}/to", "future_date") ] if to > Time.zone.today
      return [ error("#{path}/from", "inverted_period") ] if from > to

      []
    end

    def date(value)
      return nil unless value.is_a?(String) && value.match?(DATE)

      Date.iso8601(value)
    rescue Date::Error
      nil
    end

    def uuid?(value)
      value.is_a?(String) && value.match?(UUID)
    end

    def text?(value, max)
      value.is_a?(String) && value.strip.present? && value.length <= max
    end

    def error(path, message)
      { path: path, message: message }
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/audience_schema_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/services/campaigns/audience_schema.rb spec/services/campaigns/audience_schema_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: validate campaign audience JSON with field paths

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Histórico de teste e critérios de triagem

**Files:**
- Create: `lib/campaign_history.rb`
- Create: `app/services/campaigns/criteria.rb`, `app/services/campaigns/criteria/protocol_period.rb`, `triage_tier.rb`, `triage_incomplete.rb`, `triaged_not_attended.rb`
- Modify: `spec/support/campaign_helpers.rb`
- Test: `spec/services/campaigns/criteria/triage_criteria_spec.rb`

**Interfaces:**
- Consumes: `Campaigns::AudienceSchema::CRITERIA` (Task 4).
- Produces:
  - `CampaignHistory.cpf_for(seed) → String` (11 dígitos, dígitos verificadores válidos); `.citizen!(cpf:, phone:, neighborhood: nil) → Citizen`; `.protocol → ProtocolDefinition`; `.triage!(citizen, status: "completed", protocol_name: "triage-respiratoria", tier: "alta", at:) → Triage` (`at` = `completed_at` da concluída, `created_at` da abandonada; conversa web própria, encerrada); `.attendance!(citizen, outcome:, at:, unit:, by:, triage: nil) → Attendance` (`closed`, `closed_at = at`); `.request!(citizen, kind: "return", unit:, target: unit, by:, at: 3.days.ago, status: "open", reopened_reason: nil) → AppointmentRequest`; `.no_show!(citizen, at:, unit:, by:) → Appointment` (`scheduled_at = at`).
  - `Campaigns::Criteria::KINDS` (kind → nome da classe); `Campaigns::Criteria.for(kind) → Class`; `Campaigns::Criteria.period(params) → Range<ActiveSupport::TimeWithZone>`; `Campaigns::Criteria.triage_citizens(triages_relation) → relação que seleciona conversations.citizen_id`.
  - `Campaigns::Criteria::<Kind>.relation(params) → ActiveRecord::Relation` que seleciona uma coluna de `citizen_id` (use em `Citizen.where(id: ...)`), com `params` já validados (chaves de texto).
  - Helpers de spec: `next_phone`, `person!(phone: next_phone, cpf: nil, neighborhood: nil)`, `staff!`, `unit!`, `opt_in!(citizen, value = true)`, `revoked_conversation!(citizen, at:)`, `anonymous_triage!(at:)`.

- [ ] **Step 1: Escreva `lib/campaign_history.rb`**

```ruby
# lib/campaign_history.rb
require "digest"

# Histórico clínico no passado, gravado direto no banco (módulo 12; desvio 16
# do plano): os critérios de campanha olham para trás, mas os comandos do
# domínio só aceitam "agora" (desfecho) ou o futuro (horário). Usado pelas
# specs e pela semente de dev. Só INSERT: os CHECKs e triggers de cada tabela
# continuam valendo, e é o que garante que a linha é coerente.
module CampaignHistory
  CONVERSATION_STATE = {
    "completed" => "completed", "aborted_by_timeout" => "abandoned",
    "aborted_by_cancellation" => "cancelled", "aborted_by_revocation" => "revoked"
  }.freeze

  module_function

  # CPF com dígitos verificadores válidos, determinístico pela semente.
  def cpf_for(seed)
    base = Digest::SHA256.hexdigest(seed.to_s).scan(/\d/).join[0, 9].ljust(9, "7")
    nums = base.chars.map(&:to_i)
    first = CitizenIdentity::Cpf.check_digit(nums)
    second = CitizenIdentity::Cpf.check_digit(nums + [ first ])
    "#{base}#{first}#{second}"
  end

  def citizen!(cpf:, phone:, neighborhood: nil)
    Citizen.find_or_create_by!(cpf: cpf, phone: phone) { |c| c.neighborhood = neighborhood }
  end

  def protocol
    ProtocolDefinition.find_by(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active") ||
      ProtocolDefinition.order(:created_at).first ||
      raise("CampaignHistory: nenhum protocolo na cidade")
  end

  # `at` é o instante que o critério lê: a conclusão da triagem concluída, a
  # criação da abandonada. Conversa web própria e encerrada; o bairro atual do
  # cidadão é copiado, como faz o StartTriage.
  def triage!(citizen, status: "completed", protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME, tier: "alta",
              at: 2.days.ago)
    completed = status == "completed"
    created_at = completed ? at - 5.minutes : at
    conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone,
                                        state: CONVERSATION_STATE.fetch(status), created_at: created_at)
    Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol_name,
                   status: status, tier: completed ? tier : nil, priority: completed ? 1 : nil, answers: {},
                   created_at: created_at, completed_at: completed ? at : nil,
                   neighborhood_id: citizen.neighborhood_id)
  end

  # Atendimento encerrado em `at`, a partir de uma triagem (a dada ou uma nova,
  # uma hora antes). "left" é saída antes da chamada: sem chamada.
  def attendance!(citizen, outcome:, at:, unit:, by:, triage: nil)
    triage ||= triage!(citizen, at: at - 1.hour)
    called = outcome != "left"
    Attendance.create!(triage: triage, citizen: citizen, health_unit: unit, checked_in_by_user: by,
                       checked_in_at: at - 50.minutes, check_in_method: "code", status: "closed",
                       called_by_user: called ? by : nil, called_at: called ? at - 30.minutes : nil,
                       outcome: outcome, closed_by_user: by, closed_at: at,
                       referral_note: outcome == "referred" ? "Encaminhado para avaliação especializada" : nil)
  end

  # Pedido de agendamento nascido de um desfecho: return (mesma unidade) ou
  # referral (encaminhamento para `target`).
  def request!(citizen, kind: "return", unit:, target: unit, by:, at: 3.days.ago, status: "open", reopened_reason: nil)
    origin = attendance!(citizen, outcome: kind == "return" ? "return" : "referred", at: at, unit: unit, by: by)
    AppointmentRequest.create!(origin_attendance: origin, citizen: citizen, root_triage: origin.triage,
                               origin_unit: unit, target_unit: kind == "return" ? unit : target, kind: kind,
                               status: status, reopened_reason: reopened_reason)
  end

  # Falta em horário confirmado (MarkNoShowAppointmentsJob): o horário fica
  # no_show e o pedido volta à fila (open, reopened_reason no_show).
  def no_show!(citizen, at:, unit:, by:)
    request = request!(citizen, unit: unit, by: by, at: at - 7.days, reopened_reason: "no_show")
    Appointment.create!(request: request, citizen: citizen, health_unit: unit, scheduled_by_user: by,
                        scheduled_at: at, status: "no_show", confirmed_at: at - 1.day,
                        confirmation_deadline_at: at - 1.day, ended_at: at.end_of_day)
  end
end
```

- [ ] **Step 2: Acrescente os helpers de histórico**

No topo de `spec/support/campaign_helpers.rb`, antes de `module CampaignHelpers`:

```ruby
require Rails.root.join("lib/campaign_history").to_s
```

e, dentro do módulo, antes do `end`:

```ruby

  # Telefones distintos por exemplo (+55 41 99xxxxxxx).
  def next_phone
    @campaign_phone_seq = (@campaign_phone_seq || 0) + 1
    format("+554199%07d", @campaign_phone_seq)
  end

  def person!(phone: next_phone, cpf: nil, neighborhood: nil)
    CampaignHistory.citizen!(cpf: cpf || CampaignHistory.cpf_for(phone), phone: phone, neighborhood: neighborhood)
  end

  def staff!
    @campaign_staff ||= staff_with("recepcao-#{SecureRandom.hex(4)}@cidade.gov.br", "citizen_verifier")
  end

  def unit!
    @campaign_unit ||= create_unit("UBS Campanha")
  end

  def opt_in!(citizen, value = true)
    CitizenContactPreference.for(citizen.id).tap { |p| p.update!(sms_opt_in: value) }
  end

  # Conversa revogada do cidadão, criada em `at` (a revogação vale para o
  # público só se esta for a conversa mais recente dele).
  def revoked_conversation!(citizen, at: 1.hour.ago)
    conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "revoked",
                                        created_at: at)
    Consent.create!(conversation: conversation, version: 1, policy_text_sha: "sha-teste", channel: "web",
                    given_at: at, revoked_at: at + 1.minute)
    conversation
  end

  # Triagem do WhatsApp antigo: conversa sem cidadão.
  def anonymous_triage!(at:)
    conversation = Conversation.create!(phone: "+5541911110000", state: "completed", created_at: at - 5.minutes)
    Triage.create!(conversation: conversation, protocol_definition: CampaignHistory.protocol,
                   protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "completed", tier: "alta", priority: 1,
                   answers: {}, created_at: at - 5.minutes, completed_at: at)
  end
```

- [ ] **Step 3: Escreva a spec dos critérios de triagem**

```ruby
# spec/services/campaigns/criteria/triage_criteria_spec.rb
require "rails_helper"

# ADR 0024 §4.1: bordas do período inclusivas no fuso da cidade; a triagem chega
# ao cidadão por conversations.citizen_id; triagem sem cidadão não conta.
RSpec.describe "Critérios de triagem" do
  before { create_default_protocol! }

  let(:from) { Time.zone.today - 30 }
  let(:to) { Time.zone.today - 10 }
  let(:first_instant) { from.in_time_zone.beginning_of_day }
  let(:last_instant) { to.in_time_zone.end_of_day }
  let(:inside) { first_instant + 2.days }
  let(:period) { { "from" => from.iso8601, "to" => to.iso8601 } }

  def ids(klass, params) = Citizen.where(id: klass.relation(params)).pluck(:id)

  describe Campaigns::Criteria::ProtocolPeriod do
    let(:params) { { "kind" => "protocol_period", "protocol_name" => "triage-respiratoria" }.merge(period) }

    it "entra quem concluiu triagem desse protocolo no período, com as bordas" do
      on_first = person!.tap { |c| CampaignHistory.triage!(c, at: first_instant) }
      on_last = person!.tap { |c| CampaignHistory.triage!(c, at: last_instant) }
      person!.tap { |c| CampaignHistory.triage!(c, at: first_instant - 1.second) }
      person!.tap { |c| CampaignHistory.triage!(c, at: last_instant + 1.second) }
      person!.tap { |c| CampaignHistory.triage!(c, protocol_name: "dengue", at: inside) }
      person!.tap { |c| CampaignHistory.triage!(c, status: "aborted_by_timeout", at: inside) }
      expect(ids(described_class, params)).to contain_exactly(on_first.id, on_last.id)
    end

    it "triagem sem cidadão não conta" do
      anonymous_triage!(at: inside)
      expect(ids(described_class, params)).to be_empty
    end
  end

  describe Campaigns::Criteria::TriageTier do
    let(:params) { { "kind" => "triage_tier", "tiers" => [ "alta" ] }.merge(period) }

    it "entra quem concluiu triagem com a faixa na lista, no período" do
      high = person!.tap { |c| CampaignHistory.triage!(c, tier: "alta", at: last_instant) }
      person!.tap { |c| CampaignHistory.triage!(c, tier: "baixa", at: inside) }
      person!.tap { |c| CampaignHistory.triage!(c, tier: "alta", at: last_instant + 1.second) }
      expect(ids(described_class, params)).to eq([ high.id ])
      anonymous_triage!(at: inside)
      expect(ids(described_class, params)).to eq([ high.id ])
    end
  end

  describe Campaigns::Criteria::TriageIncomplete do
    let(:params) { { "kind" => "triage_incomplete" }.merge(period) }

    it "entra quem abandonou (tempo ou cancelamento) no período e não concluiu outra depois" do
      timed_out = person!.tap { |c| CampaignHistory.triage!(c, status: "aborted_by_timeout", at: first_instant) }
      cancelled = person!.tap { |c| CampaignHistory.triage!(c, status: "aborted_by_cancellation", at: last_instant) }
      completed_before = person!.tap do |c|
        CampaignHistory.triage!(c, at: inside - 1.day)
        CampaignHistory.triage!(c, status: "aborted_by_timeout", at: inside)
      end
      person!.tap do |c|
        CampaignHistory.triage!(c, status: "aborted_by_timeout", at: inside)
        CampaignHistory.triage!(c, at: inside + 1.day)
      end
      person!.tap { |c| CampaignHistory.triage!(c, status: "aborted_by_timeout", at: first_instant - 1.second) }
      person!.tap { |c| CampaignHistory.triage!(c, status: "aborted_by_revocation", at: inside) }
      expect(ids(described_class, params)).to contain_exactly(timed_out.id, cancelled.id, completed_before.id)
    end
  end

  describe Campaigns::Criteria::TriagedNotAttended do
    let(:params) { { "kind" => "triaged_not_attended" }.merge(period) }

    it "entra quem concluiu triagem no período sem atendimento daquela triagem" do
      waiting = person!.tap { |c| CampaignHistory.triage!(c, at: inside) }
      person!.tap do |c|
        triage = CampaignHistory.triage!(c, at: inside)
        CampaignHistory.attendance!(c, outcome: "discharged", at: inside + 1.hour, unit: unit!, by: staff!, triage: triage)
      end
      person!.tap { |c| CampaignHistory.triage!(c, at: last_instant + 1.second) }
      anonymous_triage!(at: inside)
      expect(ids(described_class, params)).to eq([ waiting.id ])
    end
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/criteria/triage_criteria_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::Criteria`.

- [ ] **Step 5: Implemente o registro e os quatro critérios**

```ruby
# app/services/campaigns/criteria.rb
# Critérios clínicos do público (ADR 0024 §4.1). Cada kind é uma classe com
# `.relation(params)` que devolve uma relação SQL de citizen_id — nunca
# carrega cidadão em Ruby. `params` já passou por Campaigns::AudienceSchema.
module Campaigns
  module Criteria
    KINDS = {
      "protocol_period" => "ProtocolPeriod",
      "triage_tier" => "TriageTier",
      "triage_incomplete" => "TriageIncomplete",
      "attendance_outcome" => "AttendanceOutcome",
      "triaged_not_attended" => "TriagedNotAttended",
      "appointment_no_show" => "AppointmentNoShow",
      "appointment_request_open" => "AppointmentRequestOpen"
    }.freeze

    def self.for(kind)
      const_get(KINDS.fetch(kind))
    end

    # Datas inclusivas no fuso da cidade (Time.zone).
    def self.period(params)
      Date.iso8601(params.fetch("from")).in_time_zone.beginning_of_day..
        Date.iso8601(params.fetch("to")).in_time_zone.end_of_day
    end

    # Triagem → cidadão: triages.conversation_id → conversations.citizen_id.
    # Conversa sem cidadão (WhatsApp antigo) fica de fora.
    def self.triage_citizens(triages)
      Conversation.where.not(citizen_id: nil).where(id: triages.select(:conversation_id)).select(:citizen_id)
    end
  end
end
```

```ruby
# app/services/campaigns/criteria/protocol_period.rb
# Triagem concluída com esse protocolo, concluída no período.
module Campaigns
  module Criteria
    class ProtocolPeriod
      def self.relation(params)
        Criteria.triage_citizens(Triage.where(status: "completed", protocol_name: params.fetch("protocol_name"),
                                              completed_at: Criteria.period(params)))
      end
    end
  end
end
```

```ruby
# app/services/campaigns/criteria/triage_tier.rb
# Triagem concluída com a faixa na lista, concluída no período. `tier` é texto
# do protocolo: renomear a faixa num protocolo novo separa as triagens.
module Campaigns
  module Criteria
    class TriageTier
      def self.relation(params)
        Criteria.triage_citizens(Triage.where(status: "completed", tier: params.fetch("tiers"),
                                              completed_at: Criteria.period(params)))
      end
    end
  end
end
```

```ruby
# app/services/campaigns/criteria/triage_incomplete.rb
# Triagem abandonada (tempo ou cancelamento) criada no período, sem nenhuma
# triagem concluída do mesmo cidadão criada depois dela.
module Campaigns
  module Criteria
    class TriageIncomplete
      NO_LATER_COMPLETION = <<~SQL.squish.freeze
        NOT EXISTS (
          SELECT 1 FROM triages later
          JOIN conversations later_conversation ON later_conversation.id = later.conversation_id
          WHERE later_conversation.citizen_id = conversations.citizen_id
            AND later.status = 'completed'
            AND later.created_at > triages.created_at
        )
      SQL

      def self.relation(params)
        Triage.joins(:conversation)
              .where(status: %w[aborted_by_timeout aborted_by_cancellation], created_at: Criteria.period(params))
              .where.not(conversations: { citizen_id: nil })
              .where(NO_LATER_COMPLETION)
              .select("conversations.citizen_id")
      end
    end
  end
end
```

```ruby
# app/services/campaigns/criteria/triaged_not_attended.rb
# Triagem concluída no período sem nenhum atendimento com aquele triage_id.
module Campaigns
  module Criteria
    class TriagedNotAttended
      def self.relation(params)
        attended = Attendance.where.not(triage_id: nil).select(:triage_id)
        Criteria.triage_citizens(Triage.where(status: "completed", completed_at: Criteria.period(params))
                                       .where.not(id: attended))
      end
    end
  end
end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/criteria/triage_criteria_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add lib/campaign_history.rb app/services/campaigns/criteria.rb app/services/campaigns/criteria/protocol_period.rb app/services/campaigns/criteria/triage_tier.rb app/services/campaigns/criteria/triage_incomplete.rb app/services/campaigns/criteria/triaged_not_attended.rb spec/support/campaign_helpers.rb spec/services/campaigns/criteria/triage_criteria_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add triage-based campaign audience criteria

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Critérios de atendimento e agendamento

**Files:**
- Create: `app/services/campaigns/criteria/attendance_outcome.rb`, `appointment_no_show.rb`, `appointment_request_open.rb`
- Test: `spec/services/campaigns/criteria/care_criteria_spec.rb`

**Interfaces:**
- Consumes: `Campaigns::Criteria.period`, `CampaignHistory` e helpers (Task 5).
- Produces: `Campaigns::Criteria::AttendanceOutcome.relation(params)`, `::AppointmentNoShow.relation(params)`, `::AppointmentRequestOpen.relation(params)` — mesma forma da Task 5; e a garantia (testada) de que `AudienceSchema::CRITERIA.keys == Criteria::KINDS.keys` e cada kind resolve para uma classe.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/campaigns/criteria/care_criteria_spec.rb
require "rails_helper"

RSpec.describe "Critérios de atendimento e agendamento" do
  before { create_default_protocol! }

  let(:from) { Time.zone.today - 30 }
  let(:to) { Time.zone.today - 10 }
  let(:first_instant) { from.in_time_zone.beginning_of_day }
  let(:last_instant) { to.in_time_zone.end_of_day }
  let(:inside) { first_instant + 2.days }
  let(:period) { { "from" => from.iso8601, "to" => to.iso8601 } }

  def ids(klass, params) = Citizen.where(id: klass.relation(params)).pluck(:id)

  it "todo kind do schema tem classe, e vice-versa" do
    expect(Campaigns::AudienceSchema::CRITERIA.keys).to match_array(Campaigns::Criteria::KINDS.keys)
    Campaigns::Criteria::KINDS.each_key { |kind| expect(Campaigns::Criteria.for(kind)).to respond_to(:relation) }
  end

  describe Campaigns::Criteria::AttendanceOutcome do
    let(:other_unit) { create_unit("UPA Norte", kind: "upa") }

    it "entra quem teve atendimento encerrado com desfecho na lista, no período e na unidade dada" do
      discharged = person!.tap { |c| CampaignHistory.attendance!(c, outcome: "discharged", at: first_instant, unit: unit!, by: staff!) }
      left = person!.tap { |c| CampaignHistory.attendance!(c, outcome: "left", at: last_instant, unit: other_unit, by: staff!) }
      person!.tap { |c| CampaignHistory.attendance!(c, outcome: "referred", at: inside, unit: unit!, by: staff!) }
      person!.tap { |c| CampaignHistory.attendance!(c, outcome: "discharged", at: last_instant + 1.second, unit: unit!, by: staff!) }

      params = { "kind" => "attendance_outcome", "outcomes" => %w[discharged left] }.merge(period)
      expect(ids(described_class, params)).to contain_exactly(discharged.id, left.id)
      expect(ids(described_class, params.merge("health_unit_id" => other_unit.id))).to eq([ left.id ])
    end
  end

  describe Campaigns::Criteria::AppointmentNoShow do
    it "entra quem faltou a horário confirmado com scheduled_at no período" do
      on_first = person!.tap { |c| CampaignHistory.no_show!(c, at: first_instant, unit: unit!, by: staff!) }
      person!.tap { |c| CampaignHistory.no_show!(c, at: first_instant - 1.second, unit: unit!, by: staff!) }
      person!.tap { |c| CampaignHistory.request!(c, unit: unit!, by: staff!, at: inside) }
      expect(ids(described_class, { "kind" => "appointment_no_show" }.merge(period))).to eq([ on_first.id ])
    end
  end

  describe Campaigns::Criteria::AppointmentRequestOpen do
    let(:upa) { create_unit("UPA Norte", kind: "upa") }

    it "entra quem tem pedido aberto agora, filtrando tipo e unidade de destino quando dados" do
      back = person!.tap { |c| CampaignHistory.request!(c, kind: "return", unit: unit!, by: staff!) }
      referred = person!.tap { |c| CampaignHistory.request!(c, kind: "referral", unit: unit!, target: upa, by: staff!) }
      person!.tap { |c| CampaignHistory.request!(c, kind: "return", unit: unit!, by: staff!, status: "scheduled") }

      expect(ids(described_class, { "kind" => "appointment_request_open" })).to contain_exactly(back.id, referred.id)
      expect(ids(described_class, { "kind" => "appointment_request_open", "kinds" => [ "referral" ] })).to eq([ referred.id ])
      expect(ids(described_class, { "kind" => "appointment_request_open", "target_unit_id" => unit!.id })).to eq([ back.id ])
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/criteria/care_criteria_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::Criteria::AttendanceOutcome`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/campaigns/criteria/attendance_outcome.rb
# Atendimento encerrado com desfecho na lista (e na unidade, se dada),
# encerrado no período.
module Campaigns
  module Criteria
    class AttendanceOutcome
      def self.relation(params)
        scope = Attendance.where(status: "closed", outcome: params.fetch("outcomes"), closed_at: Criteria.period(params))
        scope = scope.where(health_unit_id: params["health_unit_id"]) if params["health_unit_id"]
        scope.select(:citizen_id)
      end
    end
  end
end
```

```ruby
# app/services/campaigns/criteria/appointment_no_show.rb
# Falta (no_show, só a partir de confirmed — MarkNoShowAppointmentsJob) com o
# horário marcado no período.
module Campaigns
  module Criteria
    class AppointmentNoShow
      def self.relation(params)
        Appointment.where(status: "no_show", scheduled_at: Criteria.period(params)).select(:citizen_id)
      end
    end
  end
end
```

```ruby
# app/services/campaigns/criteria/appointment_request_open.rb
# Pedido de agendamento aberto agora (sem período), por tipo e destino opcionais.
module Campaigns
  module Criteria
    class AppointmentRequestOpen
      def self.relation(params)
        scope = AppointmentRequest.where(status: "open")
        scope = scope.where(kind: params["kinds"]) if params["kinds"]
        scope = scope.where(target_unit_id: params["target_unit_id"]) if params["target_unit_id"]
        scope.select(:citizen_id)
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/criteria`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/services/campaigns/criteria/attendance_outcome.rb app/services/campaigns/criteria/appointment_no_show.rb app/services/campaigns/criteria/appointment_request_open.rb spec/services/campaigns/criteria/care_criteria_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add attendance and appointment campaign audience criteria

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: `Campaigns::Audience` (recorte ∩ critérios ∖ revogados, resumo e mínimo)

**Files:**
- Create: `app/services/campaigns/audience.rb`
- Test: `spec/services/campaigns/audience_spec.rb`

**Interfaces:**
- Consumes: `AudienceSchema.normalize` (Task 4); `Criteria.for` (Tasks 5–6).
- Produces: `Campaigns::Audience.new(audience)` (público **já validado**); `#citizen_ids → relação que seleciona citizens.id`; `#summary → { citizens: Integer, phones: Integer }`; `#below_minimum? → Boolean`; `#preview → { citizens:, phones: } | { below_minimum: true }`; `Campaigns::Audience::REVOKED_SQL` (subconsulta de citizen_id revogados).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/services/campaigns/audience_spec.rb
require "rails_helper"

RSpec.describe Campaigns::Audience do
  before { create_default_protocol! }

  let(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }
  let(:window) { { "from" => (Time.zone.today - 30).iso8601, "to" => Time.zone.today.iso8601 } }

  def audience(geo: { "scope" => "city" }, all: [])
    { "version" => 1, "geo" => geo, "clinical" => { "all" => all } }
  end

  def ids(value) = Citizen.where(id: described_class.new(value).citizen_ids).pluck(:id)

  it "recorte: cidade toda inclui quem não declarou bairro; bairros e unidade, não" do
    no_neighborhood = person!
    in_centro = person!(neighborhood: centro)
    in_batel = person!(neighborhood: batel)
    unit = create_unit("UBS Centro")
    NeighborhoodCoverage.create!(neighborhood: centro, health_unit: unit)

    expect(ids(audience)).to contain_exactly(no_neighborhood.id, in_centro.id, in_batel.id)
    expect(ids(audience(geo: { "scope" => "neighborhoods", "neighborhood_ids" => [ batel.id ] }))).to eq([ in_batel.id ])
    expect(ids(audience(geo: { "scope" => "unit", "health_unit_id" => unit.id }))).to eq([ in_centro.id ])
  end

  it "combina recorte e dois critérios por E" do
    both = person!(neighborhood: centro).tap do |c|
      CampaignHistory.no_show!(c, at: 3.days.ago, unit: unit!, by: staff!)
      CampaignHistory.triage!(c, status: "aborted_by_timeout", at: 2.days.ago)
    end
    person!(neighborhood: centro).tap { |c| CampaignHistory.no_show!(c, at: 3.days.ago, unit: unit!, by: staff!) }
    person!(neighborhood: centro).tap { |c| CampaignHistory.triage!(c, status: "aborted_by_timeout", at: 2.days.ago) }
    person!(neighborhood: batel).tap do |c|
      CampaignHistory.no_show!(c, at: 3.days.ago, unit: unit!, by: staff!)
      CampaignHistory.triage!(c, status: "aborted_by_timeout", at: 2.days.ago)
    end

    value = audience(geo: { "scope" => "neighborhoods", "neighborhood_ids" => [ centro.id ] },
                     all: [ { "kind" => "appointment_no_show" }.merge(window), { "kind" => "triage_incomplete" }.merge(window) ])
    expect(ids(value)).to eq([ both.id ])
  end

  it "exclui quem tem a conversa mais recente revogada; revogação antiga não exclui" do
    revoked = person!.tap { |c| CampaignHistory.triage!(c, at: 5.days.ago); revoked_conversation!(c, at: 1.day.ago) }
    healed = person!.tap { |c| revoked_conversation!(c, at: 5.days.ago); CampaignHistory.triage!(c, at: 1.day.ago) }
    plain = person!
    expect(ids(audience)).to contain_exactly(healed.id, plain.id)
    expect(ids(audience)).not_to include(revoked.id)
  end

  it "empate de created_at entre conversas: vale a de maior id" do
    at = 2.days.ago
    citizen = person!
    revoked = revoked_conversation!(citizen, at: at)
    other = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "completed", created_at: at)
    expect(ids(audience)).to eq(revoked.id > other.id ? [] : [ citizen.id ])
  end

  it "resumo conta telefones distintos; telefone compartilhado conta uma vez" do
    shared = next_phone
    person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-a"))
    person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-b"))
    person!
    expect(described_class.new(audience).summary).to eq(citizens: 3, phones: 2)
  end

  it "mínimo de 5 telefones: 4 → below_minimum; 5 → as contagens" do
    4.times { person! }
    expect(described_class.new(audience).preview).to eq(below_minimum: true)
    expect(described_class.new(audience)).to be_below_minimum
    person!
    expect(described_class.new(audience).preview).to eq(citizens: 5, phones: 5)
  end

  it "cinco CPFs num telefone só não chegam ao mínimo" do
    shared = next_phone
    5.times { |i| person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-#{i}")) }
    expect(described_class.new(audience).preview).to eq(below_minimum: true)
  end

  it "não carrega cidadão em Ruby: citizen_ids é uma relação SQL" do
    expect(described_class.new(audience).citizen_ids).to be_a(ActiveRecord::Relation)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/audience_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::Audience`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/campaigns/audience.rb
# Público de uma campanha (ADR 0024 §4.2–§4.4): recorte geográfico ∩ cada
# critério clínico ∖ revogados, tudo em SQL. Revogado (D11) = a conversa mais
# recente do cidadão (created_at, desempate por id) tem consentimento
# revogado. O mínimo conta TELEFONES distintos (D12), pela coluna cifrada
# determinística: igualdade funciona sem decifrar.
module Campaigns
  class Audience
    REVOKED_SQL = <<~SQL.squish.freeze
      SELECT latest.citizen_id FROM (
        SELECT DISTINCT ON (citizen_id) id, citizen_id FROM conversations
        WHERE citizen_id IS NOT NULL
        ORDER BY citizen_id, created_at DESC, id DESC
      ) latest
      WHERE EXISTS (
        SELECT 1 FROM consents WHERE consents.conversation_id = latest.id AND consents.revoked_at IS NOT NULL
      )
    SQL

    def initialize(audience)
      @audience = AudienceSchema.normalize(audience)
    end

    def citizen_ids
      scope = geo_scope
      @audience.dig("clinical", "all").each do |criterion|
        scope = scope.where(id: Criteria.for(criterion["kind"]).relation(criterion))
      end
      scope.where("citizens.id NOT IN (#{REVOKED_SQL})").select(:id)
    end

    def summary
      people = Citizen.where(id: citizen_ids)
      { citizens: people.count, phones: people.distinct.count(:phone) }
    end

    def below_minimum?
      summary[:phones] < Campaign::MINIMUM_PHONES
    end

    def preview
      counts = summary
      counts[:phones] < Campaign::MINIMUM_PHONES ? { below_minimum: true } : counts
    end

    private

    # Cidadão sem bairro declarado só entra no recorte city.
    def geo_scope
      geo = @audience["geo"]
      case geo["scope"]
      when "city" then Citizen.all
      when "neighborhoods" then Citizen.where(neighborhood_id: geo["neighborhood_ids"])
      when "unit"
        Citizen.where(neighborhood_id: NeighborhoodCoverage.where(health_unit_id: geo["health_unit_id"]).select(:neighborhood_id))
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/services/campaigns/audience.rb spec/services/campaigns/audience_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: resolve campaign audience in SQL excluding revoked citizens

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — F-12.3 (rascunho, rotas, transições com step-up)

### Task 8: Rascunho: validação de conteúdo e de referências, `Create`/`Update` e `Presenter`

**Files:**
- Create: `app/services/campaigns/content_validation.rb`, `app/services/campaigns/audience_validation.rb`, `app/services/campaigns/presenter.rb`
- Create: `app/commands/campaigns/create.rb`, `app/commands/campaigns/update.rb`
- Test: `spec/services/campaigns/audience_validation_spec.rb`, `spec/commands/campaigns/draft_commands_spec.rb`, `spec/services/campaigns/presenter_spec.rb`

**Interfaces:**
- Consumes: `AudienceSchema.errors/.normalize/.error` (Task 4); `Campaign` (Task 1).
- Produces:
  - `Campaigns::ContentValidation.errors(attrs, required: false) → [{ path:, message: }]` (`attrs` com chaves de texto; confere só `title`/`body` presentes, ou exige os dois com `required: true`).
  - `Campaigns::AudienceValidation.errors(input) → [{ path:, message: }]`: o schema e, se ele passar, as referências do recorte — cada `neighborhood_ids[i]` precisa ser bairro **ativo** e `health_unit_id` do recorte `unit`, unidade **ativa**; senão `{ path: "/geo/neighborhood_ids/<i>" | "/geo/health_unit_id", message: "inactive_or_unknown" }`. É a validação que toda rota de escrita e a prévia usam.
  - `Campaigns::Create.call(attrs:, by:) → Result` (ok: `campaign:`; fail: `:invalid_campaign`, `:invalid_audience`, com `details: { details: [...] }`); publica `campaign.created` `{ campaign_id, by_user_id }`.
  - `Campaigns::Update.call(campaign:, attrs:) → Result` (ok: `campaign:`; fail: `:not_editable`, `:invalid_campaign`, `:invalid_audience`); só as chaves presentes mudam; sem evento.
  - `Campaigns::Presenter.summary(campaign) → Hash` (`CampaignSummary`), `.full(campaign) → Hash` (`Campaign` do contrato; `stats` só em `sent`, com as 7 chaves de SMS).

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/services/campaigns/audience_validation_spec.rb
require "rails_helper"

# Achado do plano do dashboard: bairro ou unidade desativados depois de salvo o
# rascunho — prévia, criar, editar, enviar e agendar recusam com o caminho do item.
RSpec.describe Campaigns::AudienceValidation do
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:ahu) { Neighborhood.create!(name: "Ahu", source: "seed", active: false) }

  def geo(value) = { "version" => 1, "geo" => value, "clinical" => { "all" => [] } }

  it "bairros ativos passam; inativo ou inexistente é recusado no item" do
    expect(described_class.errors(geo("scope" => "neighborhoods", "neighborhood_ids" => [ centro.id.upcase ]))).to eq([])
    expect(described_class.errors(geo("scope" => "neighborhoods", "neighborhood_ids" => [ centro.id, ahu.id, SecureRandom.uuid ])))
      .to eq([ { path: "/geo/neighborhood_ids/1", message: "inactive_or_unknown" },
               { path: "/geo/neighborhood_ids/2", message: "inactive_or_unknown" } ])
  end

  it "unidade do recorte precisa estar ativa" do
    active = create_unit("UBS Centro")
    inactive = create_unit("UBS Antiga", active: false)
    expect(described_class.errors(geo("scope" => "unit", "health_unit_id" => active.id))).to eq([])
    [ inactive.id, SecureRandom.uuid ].each do |id|
      expect(described_class.errors(geo("scope" => "unit", "health_unit_id" => id)))
        .to eq([ { path: "/geo/health_unit_id", message: "inactive_or_unknown" } ])
    end
  end

  it "erro de formato vem antes e não consulta o banco" do
    expect(described_class.errors(geo("scope" => "neighborhoods", "neighborhood_ids" => "Centro")))
      .to eq([ { path: "/geo/neighborhood_ids", message: "not_a_list" } ])
  end
end
```

```ruby
# spec/commands/campaigns/draft_commands_spec.rb
require "rails_helper"

RSpec.describe "Rascunho da campanha" do
  let(:manager) { staff_with("campanhas@cidade.gov.br", "campaign_manager") }
  let(:valid) do
    { "title" => "  Vacinação contra a gripe ", "body" => "Procure a unidade.\nLeve a carteirinha.  ",
      "audience" => city_audience }
  end

  describe Campaigns::Create do
    it "cria draft com pontas aparadas, quebra de linha preservada e evento só com ids" do
      campaign = described_class.call(attrs: valid, by: manager).payload[:campaign]
      expect(campaign).to have_attributes(status: "draft", title: "Vacinação contra a gripe",
                                          body: "Procure a unidade.\nLeve a carteirinha.",
                                          created_by_user_id: manager.id, audience: city_audience)
      expect(DomainEvent.where(name: "campaign.created").map(&:payload))
        .to eq([ { "campaign_id" => campaign.id, "by_user_id" => manager.id } ])
    end

    it "título e texto: faltando, curto, longo, não texto ou com HTML → invalid_campaign com o caminho" do
      {
        valid.except("title") => [ "/title", "required" ],
        valid.merge("title" => "ab") => [ "/title", "length" ],
        valid.merge("title" => "x" * 121) => [ "/title", "length" ],
        valid.merge("title" => [ "Gripe" ]) => [ "/title", "not_a_string" ],
        valid.merge("body" => "curto") => [ "/body", "length" ],
        valid.merge("body" => "x" * 2001) => [ "/body", "length" ],
        valid.merge("body" => "Veja <a href='x'>aqui</a> o aviso") => [ "/body", "html_not_allowed" ]
      }.each do |attrs, (path, message)|
        result = described_class.call(attrs: attrs, by: manager)
        expect([ result.reason, result.details ]).to eq([ :invalid_campaign, { details: [ { path: path, message: message } ] } ]), attrs.inspect
      end
      expect(Campaign.count).to eq(0)
    end

    it "texto com '<' que não abre tag passa (ex.: 'idade < 60')" do
      expect(described_class.call(attrs: valid.merge("body" => "Pessoas com idade < 60 anos"), by: manager)).to be_ok
    end

    it "público ausente ou inválido → invalid_audience com o caminho; nada criado" do
      result = described_class.call(attrs: valid.except("audience"), by: manager)
      expect([ result.reason, result.details ]).to eq([ :invalid_audience, { details: [ { path: "/", message: "not_an_object" } ] } ])
      result = described_class.call(attrs: valid.merge("audience" => city_audience.merge("version" => 2)), by: manager)
      expect(result.details).to eq(details: [ { path: "/version", message: "must_be_1" } ])
      expect(Campaign.count).to eq(0)
    end

    it "chaves de símbolo no público são gravadas como texto" do
      attrs = valid.merge("audience" => { version: 1, geo: { scope: "city" }, clinical: { all: [] } })
      expect(described_class.call(attrs: attrs, by: manager).payload[:campaign].audience).to eq(city_audience)
    end
  end

  describe Campaigns::Update do
    let(:campaign) { draft_campaign!(by: manager) }

    it "muda só as chaves presentes" do
      result = described_class.call(campaign: campaign, attrs: { "body" => "Novo texto do aviso." })
      expect(result).to be_ok
      expect(campaign.reload).to have_attributes(title: "Vacinação contra a gripe", body: "Novo texto do aviso.")
    end

    it "recusa conteúdo e público inválidos sem mudar nada" do
      expect(described_class.call(campaign: campaign, attrs: { "title" => "" }).reason).to eq(:invalid_campaign)
      expect(described_class.call(campaign: campaign, attrs: { "audience" => "cidade" }).reason).to eq(:invalid_audience)
      expect(campaign.reload.title).to eq("Vacinação contra a gripe")
    end

    it "fora de draft: not_editable" do
      sent = sent_campaign!(by: manager)
      expect(described_class.call(campaign: sent, attrs: { "title" => "Outro título" }).reason).to eq(:not_editable)
    end
  end
end
```

```ruby
# spec/services/campaigns/presenter_spec.rb
require "rails_helper"

RSpec.describe Campaigns::Presenter do
  it "rascunho: campos nulos e stats nulo" do
    campaign = draft_campaign!
    expect(described_class.full(campaign)).to include(
      id: campaign.id, status: "draft", send_at: nil, dispatched_at: nil, recipients_count: nil, failure_reason: nil,
      sms_enabled: nil, phones_count: nil, stats: nil, created_at: campaign.created_at.iso8601
    )
    expect(described_class.summary(campaign).keys).to eq(%i[id title status send_at dispatched_at recipients_count])
  end

  it "enviada: lidos e as 7 chaves de SMS, sem lista de destinatários" do
    campaign = sent_campaign!
    recipient!(campaign, person!, sms_status: "sent").update!(notice_read_at: Time.current)
    recipient!(campaign, person!, sms_status: "unavailable")
    full = described_class.full(campaign)
    expect(full[:stats]).to eq(read_count: 1, sms: { "not_opted_in" => 0, "duplicate_phone" => 0, "pending" => 0,
                                                     "deferred" => 0, "sent" => 1, "failed" => 0, "unavailable" => 1 })
    expect(full.to_json).not_to include("citizen")
    expect(full[:dispatched_at]).to eq(campaign.dispatched_at.iso8601)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns/audience_validation_spec.rb spec/commands/campaigns/draft_commands_spec.rb spec/services/campaigns/presenter_spec.rb`
Expected: FAIL (`uninitialized constant Campaigns::AudienceValidation`, `Campaigns::Create`, `Campaigns::Presenter`).

- [ ] **Step 3: Implemente as validações e o presenter**

```ruby
# app/services/campaigns/content_validation.rb
# Título e texto da campanha (spec 2026-09-29 §3.1): pontas aparadas, título
# 3–120, texto 10–2000, texto simples (sem tag HTML; "idade < 60" passa).
module Campaigns
  module ContentValidation
    HTML = %r{<\s*[a-zA-Z/!]}
    LIMITS = { "title" => Campaign::TITLE_LENGTH, "body" => Campaign::BODY_LENGTH }.freeze

    module_function

    def errors(attrs, required: false)
      LIMITS.flat_map do |key, range|
        next(required ? [ error(key, "required") ] : []) unless attrs.key?(key)

        value = attrs[key]
        next [ error(key, "not_a_string") ] unless value.is_a?(String)
        next [ error(key, "length") ] unless range.cover?(value.strip.length)
        next [ error(key, "html_not_allowed") ] if value.match?(HTML)

        []
      end
    end

    def error(key, message)
      { path: "/#{key}", message: message }
    end
  end
end
```

```ruby
# app/services/campaigns/audience_validation.rb
# Público pronto para gravar ou contar: o formato (AudienceSchema) e, depois
# dele, as referências do recorte — bairro e unidade precisam estar ATIVOS
# (achado do plano do dashboard: desativados depois do rascunho, a prévia e o
# envio recusam em vez de contar um público que a tela não mostra mais). As
# unidades dentro dos critérios não passam por aqui: filtram histórico.
module Campaigns
  module AudienceValidation
    module_function

    def errors(input)
      found = AudienceSchema.errors(input)
      return found if found.any?

      geo = AudienceSchema.normalize(input)["geo"]
      case geo["scope"]
      when "neighborhoods"
        active = Neighborhood.active_neighborhoods.where(id: geo["neighborhood_ids"]).pluck(:id)
        geo["neighborhood_ids"].each_with_index.filter_map do |id, i|
          AudienceSchema.error("/geo/neighborhood_ids/#{i}", "inactive_or_unknown") unless active.include?(id.downcase)
        end
      when "unit"
        return [] if HealthUnit.where(active: true, id: geo["health_unit_id"]).exists?

        [ AudienceSchema.error("/geo/health_unit_id", "inactive_or_unknown") ]
      else
        []
      end
    end
  end
end
```

```ruby
# app/services/campaigns/presenter.rb
# JSON da campanha no dashboard (contrato do módulo 12). Nenhuma lista de
# destinatários, em nenhuma rota: só contagens (ADR 0024).
module Campaigns
  module Presenter
    module_function

    def summary(campaign)
      {
        id: campaign.id, title: campaign.title, status: campaign.status, send_at: campaign.send_at&.iso8601,
        dispatched_at: campaign.dispatched_at&.iso8601, recipients_count: campaign.recipients_count
      }
    end

    def full(campaign)
      summary(campaign).merge(
        body: campaign.body, audience: campaign.audience, failure_reason: campaign.failure_reason,
        sms_enabled: campaign.sms_enabled, phones_count: campaign.phones_count,
        created_at: campaign.created_at.iso8601, stats: stats(campaign)
      )
    end

    # Lidos e SMS contados ao vivo: encolhem quando a revogação apaga linhas.
    def stats(campaign)
      return nil unless campaign.status == "sent"

      counts = campaign.recipients.group(:sms_status).count
      {
        read_count: campaign.recipients.where.not(notice_read_at: nil).count,
        sms: CampaignRecipient::SMS_STATUSES.index_with { |status| counts.fetch(status, 0) }
      }
    end
  end
end
```

- [ ] **Step 4: Implemente os comandos**

```ruby
# app/commands/campaigns/create.rb
# Cria o rascunho (spec 2026-09-29 §6.1). Título/texto conferidos antes do
# público: a resposta traz um código só. Reasons: :invalid_campaign,
# :invalid_audience (details: { details: [{ path:, message: }] }).
module Campaigns
  class Create
    def self.call(attrs:, by:)
      attrs = attrs.to_h.stringify_keys
      content = ContentValidation.errors(attrs, required: true)
      return Result.fail(:invalid_campaign, details: { details: content }) if content.any?

      audience = AudienceValidation.errors(attrs["audience"])
      return Result.fail(:invalid_audience, details: { details: audience }) if audience.any?

      campaign = ApplicationRecord.transaction do
        Campaign.create!(title: attrs["title"], body: attrs["body"], created_by_user: by,
                         audience: AudienceSchema.normalize(attrs["audience"])).tap do |created|
          DomainEvents.publish("campaign.created", campaign_id: created.id, by_user_id: by.id)
        end
      end
      Result.ok(campaign: campaign)
    end
  end
end
```

```ruby
# app/commands/campaigns/update.rb
# Edita o rascunho: só draft, só as chaves presentes (title, body, audience).
# Reasons: :not_editable, :invalid_campaign, :invalid_audience.
module Campaigns
  class Update
    FIELDS = %w[title body audience].freeze

    def self.call(campaign:, attrs:)
      attrs = attrs.to_h.stringify_keys.slice(*FIELDS)
      ApplicationRecord.transaction do
        campaign.lock!
        next Result.fail(:not_editable) unless campaign.status == "draft"

        content = ContentValidation.errors(attrs)
        next Result.fail(:invalid_campaign, details: { details: content }) if content.any?

        if attrs.key?("audience")
          audience = AudienceValidation.errors(attrs["audience"])
          next Result.fail(:invalid_audience, details: { details: audience }) if audience.any?

          attrs["audience"] = AudienceSchema.normalize(attrs["audience"])
        end
        campaign.update!(attrs)
        Result.ok(campaign: campaign)
      end
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/services/campaigns spec/commands/campaigns`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/services/campaigns/content_validation.rb app/services/campaigns/audience_validation.rb app/services/campaigns/presenter.rb app/commands/campaigns/create.rb app/commands/campaigns/update.rb spec/services/campaigns/audience_validation_spec.rb spec/commands/campaigns/draft_commands_spec.rb spec/services/campaigns/presenter_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add campaign draft commands with content and audience validation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: `CampaignsController`: lista, opções, prévia, criar, ler e editar

**Files:**
- Create: `app/controllers/campaigns_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/campaigns_spec.rb`

**Interfaces:**
- Consumes: `CampaignPolicy#manage?` (Task 2); `AudienceValidation`, `Audience#preview`, `Create`, `Update`, `Presenter` (Tasks 7–8).
- Produces: `GET /campaigns`, `GET /campaigns/options`, `POST /campaigns/preview`, `POST /campaigns` (201), `GET /campaigns/:id`, `PATCH /campaigns/:id` nos formatos do contrato; `CampaignsController#body_params` (corpo JSON como Hash), `#respond(result, status:)`, `#set_campaign` e `ERROR_STATUS` — a Task 12 acrescenta as ações de transição neste controller.

- [ ] **Step 1: Escreva a spec de request**

```ruby
# spec/requests/campaigns_spec.rb
require "rails_helper"

RSpec.describe "Campanhas — rascunho e prévia", type: :request do
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:manager) { staff_with("campanhas@cidade.gov.br", "campaign_manager") }
  let(:valid) do
    { title: "Vacinação contra a gripe", body: "A campanha começa na segunda-feira.", audience: city_audience }
  end

  describe "com campaign_manager" do
    before { sign_in_as(manager) }

    it "cria, edita, lê e lista (mais nova primeiro), sem lista de destinatários" do
      json_post "/campaigns", valid
      expect(response).to have_http_status(:created)
      id = body.dig("campaign", "id")
      expect(body["campaign"].keys).to match_array(%w[id title status send_at dispatched_at recipients_count body audience
                                                      failure_reason sms_enabled phones_count created_at stats])
      expect(body["campaign"]).to include("status" => "draft", "send_at" => nil, "stats" => nil, "audience" => city_audience)

      patch "/campaigns/#{id}", params: { title: "Gripe: vacinação" }, as: :json
      expect(response).to have_http_status(:ok)
      expect(body.dig("campaign", "title")).to eq("Gripe: vacinação")

      older = draft_campaign!(by: manager)
      older.update_columns(created_at: 1.day.ago)
      get "/campaigns"
      expect(body["campaigns"].map { |c| c["id"] }).to eq([ id, older.id ])
      expect(body["campaigns"].first.keys).to match_array(%w[id title status send_at dispatched_at recipients_count])

      get "/campaigns/#{id}"
      expect(body.dig("campaign", "body")).to eq("A campanha começa na segunda-feira.")
    end

    it "prévia: contagens com 5 telefones ou mais; menos de 5 sem números" do
      4.times { person! }
      json_post "/campaigns/preview", audience: city_audience
      expect(body).to eq("below_minimum" => true)
      person!
      json_post "/campaigns/preview", audience: city_audience
      expect(body).to eq("citizens" => 5, "phones" => 5)
    end

    it "público malformado: 422 invalid_audience com o caminho, nunca 500, nada gravado" do
      bad = [
        [ city_audience.merge("geo" => { "scope" => "neighborhoods", "neighborhood_ids" => "Centro" }), "/geo/neighborhood_ids" ],
        [ city_audience.merge("geo" => { "scope" => "unit", "health_unit_id" => "ubs-1" }), "/geo/health_unit_id" ],
        [ city_audience({ "kind" => "appointment_no_show", "from" => "2026-02-30", "to" => Time.zone.today.iso8601 }),
          "/clinical/all/0/from" ],
        [ city_audience({ "from" => Time.zone.today.iso8601 }), "/clinical/all/0/kind" ],
        [ "cidade toda", "/" ]
      ]
      bad.each do |audience, path|
        json_post "/campaigns/preview", audience: audience
        expect(status_and_error).to eq([ 422, "invalid_audience" ]), path
        expect(body["details"].map { |d| d["path"] }).to eq([ path ])
        json_post "/campaigns", valid.merge(audience: audience)
        expect(status_and_error).to eq([ 422, "invalid_audience" ]), path
      end
      expect(Campaign.count).to eq(0)
    end

    it "bairro desativado depois do rascunho: prévia e edição recusam com o caminho do item" do
      centro = Neighborhood.create!(name: "Centro", source: "seed")
      audience = { "version" => 1, "geo" => { "scope" => "neighborhoods", "neighborhood_ids" => [ centro.id ] },
                   "clinical" => { "all" => [] } }
      json_post "/campaigns", valid.merge(audience: audience)
      id = body.dig("campaign", "id")
      centro.update!(active: false)

      json_post "/campaigns/preview", audience: audience
      expect(body).to eq("error" => "invalid_audience",
                         "details" => [ { "path" => "/geo/neighborhood_ids/0", "message" => "inactive_or_unknown" } ])
      patch "/campaigns/#{id}", params: { audience: audience }, as: :json
      expect(status_and_error).to eq([ 422, "invalid_audience" ])
    end

    it "título ou texto inválido: 422 invalid_campaign com o caminho" do
      json_post "/campaigns", valid.merge(title: "ab")
      expect(body).to eq("error" => "invalid_campaign", "details" => [ { "path" => "/title", "message" => "length" } ])
    end

    it "editar fora de draft: 422 not_editable" do
      sent = sent_campaign!(by: manager)
      patch "/campaigns/#{sent.id}", params: { title: "Outro título" }, as: :json
      expect(status_and_error).to eq([ 422, "not_editable" ])
    end

    it "campanha inexistente ou id que não é UUID: 404 not_found" do
      [ SecureRandom.uuid, "nao-e-uuid" ].each do |id|
        get "/campaigns/#{id}"
        expect(status_and_error).to eq([ 404, "not_found" ])
      end
    end

    it "opções: protocolos e faixas de triagens concluídas, desfechos, bairros e unidades ativos" do
      create_default_protocol!
      CampaignHistory.triage!(person!, tier: "alta", at: 2.days.ago)
      CampaignHistory.triage!(person!, protocol_name: "dengue", tier: "baixa", at: 2.days.ago)
      CampaignHistory.triage!(person!, protocol_name: "abandonado", status: "aborted_by_timeout", at: 2.days.ago)
      centro = Neighborhood.create!(name: "Centro", source: "seed")
      Neighborhood.create!(name: "Ahu", source: "seed", active: false)
      ubs = create_unit("UBS Centro")
      create_unit("UBS Antiga", active: false)

      get "/campaigns/options"
      expect(body).to eq(
        "protocols" => %w[dengue triage-respiratoria], "tiers" => %w[alta baixa],
        "outcomes" => %w[discharged referred return left],
        "neighborhoods" => [ { "id" => centro.id, "name" => "Centro" } ],
        "units" => [ { "id" => ubs.id, "name" => "UBS Centro" } ]
      )
    end
  end

  describe "só o campaign_manager" do
    (Membership::ROLES - %w[campaign_manager]).each do |role|
      it "#{role}: 403 missing_role em todas, nada muda" do
        campaign = draft_campaign!
        sign_in_as(staff_with("#{role}@cidade.gov.br", role))
        get "/campaigns"
        expect(status_and_error).to eq([ 403, "missing_role" ])
        get "/campaigns/options"
        expect(status_and_error).to eq([ 403, "missing_role" ])
        get "/campaigns/#{campaign.id}"
        expect(status_and_error).to eq([ 403, "missing_role" ])
        json_post "/campaigns/preview", audience: city_audience
        expect(status_and_error).to eq([ 403, "missing_role" ])
        json_post "/campaigns", valid
        expect(status_and_error).to eq([ 403, "missing_role" ])
        patch "/campaigns/#{campaign.id}", params: { title: "Outro título" }, as: :json
        expect(status_and_error).to eq([ 403, "missing_role" ])
        expect(Campaign.count).to eq(1)
      end
    end

    it "sem sessão: 401" do
      get "/campaigns"
      expect(response).to have_http_status(:unauthorized)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/campaigns_spec.rb`
Expected: FAIL (rota inexistente: 404 em todas).

- [ ] **Step 3: Implemente o controller**

```ruby
# app/controllers/campaigns_controller.rb
# Campanhas da cidade (ADR 0024; spec 2026-09-29 §6.1): só campaign_manager.
# Prefixo único /campaigns (uma entrada só no proxy de dev do dashboard); a
# escrita fica fora de /admin/api, que segue só leitura. Nenhuma resposta traz
# a lista de destinatários.
class CampaignsController < ApplicationController
  include Authentication
  include MfaStepUp

  wrap_parameters false

  ERROR_STATUS = {
    invalid_campaign: :unprocessable_entity, invalid_audience: :unprocessable_entity,
    below_minimum: :unprocessable_entity, not_editable: :unprocessable_entity,
    invalid_transition: :unprocessable_entity, invalid_send_at: :unprocessable_entity
  }.freeze

  before_action :require_campaign_manager
  before_action :set_campaign, only: %i[show update]

  def index
    campaigns = Campaign.order(created_at: :desc, id: :desc)
    render json: { campaigns: campaigns.map { |c| Campaigns::Presenter.summary(c) } }
  end

  def options
    completed = Triage.where(status: "completed")
    render json: {
      protocols: completed.distinct.order(:protocol_name).pluck(:protocol_name),
      tiers: completed.where.not(tier: nil).distinct.order(:tier).pluck(:tier),
      outcomes: Attendance::OUTCOMES,
      neighborhoods: Neighborhood.active_neighborhoods.order(:name).map { |n| { id: n.id, name: n.name } },
      units: HealthUnit.where(active: true).order(:name).map { |u| { id: u.id, name: u.name } }
    }
  end

  def preview
    audience = body_params["audience"]
    errors = Campaigns::AudienceValidation.errors(audience)
    return render(json: { error: "invalid_audience", details: errors }, status: :unprocessable_entity) if errors.any?

    render json: Campaigns::Audience.new(audience).preview
  end

  def create
    respond(Campaigns::Create.call(attrs: body_params, by: Current.user), status: :created)
  end

  def show
    render json: { campaign: Campaigns::Presenter.full(@campaign) }
  end

  def update
    respond(Campaigns::Update.call(campaign: @campaign, attrs: body_params))
  end

  private

  # Corpo JSON como Hash de chaves de texto (Authentication já exige
  # application/json em toda escrita).
  def body_params
    request.request_parameters
  end

  def require_campaign_manager
    render json: { error: "missing_role" }, status: :forbidden unless CampaignPolicy.new(Current.user, nil).manage?
  end

  def set_campaign
    @campaign = Campaign.find_by(id: params[:id])
    render json: { error: "not_found" }, status: :not_found unless @campaign
  end

  def respond(result, status: :ok)
    if result.failure?
      return render json: { error: result.reason.to_s }.merge(result.details),
                    status: ERROR_STATUS.fetch(result.reason, :unprocessable_entity)
    end

    render json: { campaign: Campaigns::Presenter.full(result.payload[:campaign].reload) }, status: status
  end
end
```

- [ ] **Step 4: Acrescente as rotas**

Em `config/routes.rb`, logo depois do bloco `scope "/territory" do ... end`:

```ruby

  # Campanhas (ADR 0024; spec 2026-09-29 §6.1). Prefixo único: uma entrada só
  # no proxy de dev do dashboard. Rotas literais (options, preview e, na Task
  # 14, sms_setting) ANTES de ":id".
  get  "/campaigns", to: "campaigns#index"
  post "/campaigns", to: "campaigns#create"
  scope "/campaigns" do
    get   "options", to: "campaigns#options"
    post  "preview", to: "campaigns#preview"
    get   ":id",     to: "campaigns#show"
    patch ":id",     to: "campaigns#update"
  end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/campaigns_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/controllers/campaigns_controller.rb config/routes.rb spec/requests/campaigns_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add campaign draft, preview and options routes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

## Fatia 4 — F-12.4 e F-12.6 (congelamento, SMS, transições, agendados, chave)

### Task 10: `Campaigns::SmsBatchJob`

**Files:**
- Create: `app/jobs/campaigns/sms_batch_job.rb`
- Modify: `spec/support/campaign_helpers.rb`
- Test: `spec/jobs/campaigns/sms_batch_job_spec.rb`

**Interfaces:**
- Consumes: `SmsGateway`, `Campaigns::SmsText` (Task 3); `CampaignRecipient`, `CitizenContactPreference` (Task 1).
- Produces: `Campaigns::SmsBatchJob.perform_later(city_slug:, campaign_id:)`; constantes `BATCH_SIZE` (100), `WINDOW_HOURS` (8...20), `ATTEMPTS` (2); publica `campaign.sms_unavailable` `{ campaign_id, count }`. Helper de spec `register_test_city!` (linha de `City` no catálogo com o slug de `TEST_CITY_A`, para `with_city`/`EachCityJob`).

- [ ] **Step 1: Acrescente o helper de cidade**

Em `spec/support/campaign_helpers.rb`, dentro do módulo, antes do `end`:

```ruby

  # Jobs de cidade (with_city, EachCityJob) procuram a cidade no catálogo;
  # TEST_CITY_A não é persistida lá. Mesmo slug e banco: reentra a sessão que
  # o harness já abriu (nota 1 de spec/support/city_test_databases.rb).
  def register_test_city!
    City.find_by(slug: TEST_CITY_A.slug) ||
      create(:city, slug: TEST_CITY_A.slug, database_url: TEST_CITY_A.database_url)
  end
```

- [ ] **Step 2: Escreva a spec**

```ruby
# spec/jobs/campaigns/sms_batch_job_spec.rb
require "rails_helper"

RSpec.describe Campaigns::SmsBatchJob do
  include ActiveSupport::Testing::TimeHelpers

  let!(:city_record) { register_test_city! }
  let(:campaign) { sent_campaign!(sms_enabled: true) }
  let(:inside) { Time.zone.now.change(hour: 10) }
  let(:text) { Campaigns::SmsText.body(city_record) }

  before { CityProfile.create!(name: "Curitiba", campaigns_sms_enabled: true) }

  def run = described_class.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
  def deliveries = SmsGateway::Test.deliveries
  def opted_person = person!.tap { |p| opt_in!(p) }

  it "dentro da janela: envia o texto fixo e marca sent com a hora" do
    citizen = opted_person
    row = recipient!(campaign, citizen)
    travel_to(inside) { run }
    expect(row.reload.sms_status).to eq("sent")
    expect(row.sms_sent_at).to be_within(1.second).of(inside)
    expect(deliveries).to eq([ { phone: citizen.phone, body: text } ])
  end

  it "antes das 8h e a partir das 20h: adia, não envia e se reagenda para as 8h seguintes" do
    row = recipient!(campaign, opted_person)
    early = Time.zone.now.change(hour: 7, min: 59)
    travel_to(early) do
      expect { run }.to have_enqueued_job(described_class)
        .with(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id).at(early.change(hour: 8, min: 0))
    end
    expect(row.reload.sms_status).to eq("deferred")

    late = Time.zone.now.change(hour: 20, min: 0)
    travel_to(late) { expect { run }.to have_enqueued_job(described_class).at((late + 1.day).change(hour: 8)) }
    expect(row.reload.sms_status).to eq("deferred")
    expect(deliveries).to be_empty

    travel_to((late + 1.day).change(hour: 8)) { run }
    expect(row.reload.sms_status).to eq("sent")
  end

  it "gateway não configurado: pendentes e adiados viram unavailable, com um evento só" do
    rows = [ recipient!(campaign, opted_person), recipient!(campaign, opted_person, sms_status: "deferred") ]
    with_sms_gateway(nil) { travel_to(inside) { run; run } }
    expect(rows.map { |r| r.reload.sms_status }).to eq(%w[unavailable unavailable])
    expect(DomainEvent.where(name: "campaign.sms_unavailable").map(&:payload))
      .to eq([ { "campaign_id" => campaign.id, "count" => 2 } ])
  end

  it "uma falha não interrompe o lote: tenta de novo uma vez e grava só a classe do erro" do
    bad = opted_person
    good = opted_person
    bad_row = recipient!(campaign, bad)
    good_row = recipient!(campaign, good)
    calls = Hash.new(0)
    allow(SmsGateway).to receive(:deliver).and_wrap_original do |original, **args|
      calls[args[:phone]] += 1
      raise Errno::ECONNRESET, "falhou para #{args[:phone]}" if args[:phone] == bad.phone

      original.call(**args)
    end
    travel_to(inside) { run }
    expect(calls[bad.phone]).to eq(2)
    expect(bad_row.reload).to have_attributes(sms_status: "failed", sms_error: "Errno::ECONNRESET")
    expect(good_row.reload.sms_status).to eq("sent")
  end

  it "opt-out depois do congelamento: não envia, vira not_opted_in" do
    citizen = opted_person
    row = recipient!(campaign, citizen)
    opt_in!(citizen, false)
    travel_to(inside) { run }
    expect(row.reload.sms_status).to eq("not_opted_in")
    expect(deliveries).to be_empty
  end

  it "lote cheio: envia o lote e reenfileira a si mesmo para o resto" do
    stub_const("Campaigns::SmsBatchJob::BATCH_SIZE", 2)
    3.times { recipient!(campaign, opted_person) }
    travel_to(inside) do
      expect { run }.to have_enqueued_job(described_class).with(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    end
    expect(campaign.recipients.group(:sms_status).count).to eq("sent" => 2, "pending" => 1)
  end

  it "não toca linha que não é pending/deferred, nem campanha que não está sent" do
    row = recipient!(campaign, opted_person, sms_status: "duplicate_phone")
    travel_to(inside) { expect { run }.not_to have_enqueued_job(described_class) }
    expect(row.reload.sms_status).to eq("duplicate_phone")
    expect { described_class.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: draft_campaign!.id) }.not_to raise_error
    expect(deliveries).to be_empty
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/campaigns/sms_batch_job_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::SmsBatchJob`.

- [ ] **Step 4: Implemente**

```ruby
# app/jobs/campaigns/sms_batch_job.rb
# Entrega do SMS de uma campanha já enviada (ADR 0024; spec 2026-09-29 §5.4).
# Recebe só ids (nunca telefone): decifra dentro. Até BATCH_SIZE linhas
# pending/deferred por vez, travadas com SKIP LOCKED (dois lotes não pegam a
# mesma linha). Gateway não configurado → unavailable na hora (desvio 9); fora
# da janela 8h–20h no fuso da cidade → deferred e reagenda para as 8h. O
# opt-in é conferido de novo aqui. A falha de um destinatário nunca interrompe
# o lote; sms_error guarda só a classe do erro (a mensagem pode ter telefone).
module Campaigns
  class SmsBatchJob < ApplicationJob
    include CityScopedJob
    queue_as :default

    BATCH_SIZE = 100
    WINDOW_HOURS = (8...20)
    ATTEMPTS = 2
    OPEN = %w[pending deferred].freeze

    def perform(city_slug:, campaign_id:)
      with_city(city_slug) do
        campaign = Campaign.find_by(id: campaign_id)
        next unless campaign&.status == "sent"

        batch = campaign.recipients.where(sms_status: OPEN).order(:id).limit(BATCH_SIZE)
                        .lock("FOR UPDATE SKIP LOCKED").includes(:citizen).to_a
        next if batch.empty?
        next mark_unavailable(campaign) unless SmsGateway.configured?

        now = Time.current
        unless WINDOW_HOURS.cover?(now.hour)
          campaign.recipients.where(sms_status: "pending").update_all(sms_status: "deferred")
          self.class.set(wait_until: next_window_start(now)).perform_later(city_slug: city_slug, campaign_id: campaign.id)
          next
        end

        body = SmsText.body(Current.city)
        opted = CitizenContactPreference.where(citizen_id: batch.map(&:citizen_id), sms_opt_in: true)
                                        .pluck(:citizen_id).to_set
        batch.each { |recipient| deliver(recipient, body, opted) }
        if campaign.recipients.where(sms_status: OPEN).exists?
          self.class.perform_later(city_slug: city_slug, campaign_id: campaign.id)
        end
      end
    end

    private

    def mark_unavailable(campaign)
      count = campaign.recipients.where(sms_status: OPEN).update_all(sms_status: "unavailable")
      DomainEvents.publish("campaign.sms_unavailable", campaign_id: campaign.id, count: count) if count.positive?
    end

    def deliver(recipient, body, opted)
      return recipient.update!(sms_status: "not_opted_in") unless opted.include?(recipient.citizen_id)

      case (error = attempt(recipient.citizen.phone, body))
      when nil then recipient.update!(sms_status: "sent", sms_sent_at: Time.current, sms_error: nil)
      when SmsGateway::Unavailable then recipient.update!(sms_status: "unavailable")
      else recipient.update!(sms_status: "failed", sms_error: error.class.name.truncate(200))
      end
    end

    # Uma retentativa. Devolve nil (entregue) ou o erro da última tentativa.
    def attempt(phone, body)
      ATTEMPTS.times do |i|
        SmsGateway.deliver(phone: phone, body: body)
        return nil
      rescue SmsGateway::Unavailable => e
        return e
      rescue StandardError => e
        return e if i == ATTEMPTS - 1
      end
    end

    def next_window_start(now)
      start = now.change(hour: WINDOW_HOURS.first)
      now < start ? start : start + 1.day
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/campaigns/sms_batch_job_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/jobs/campaigns/sms_batch_job.rb spec/support/campaign_helpers.rb spec/jobs/campaigns/sms_batch_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: deliver campaign SMS in batches within the city time window

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: `Campaigns::DispatchJob` (congelamento)

**Files:**
- Create: `app/jobs/campaigns/dispatch_job.rb`
- Test: `spec/jobs/campaigns/dispatch_job_spec.rb`

**Interfaces:**
- Consumes: `Audience#citizen_ids` (Task 7); `SmsSetting.enabled?` (Task 3); `SmsBatchJob` (Task 10).
- Produces: `Campaigns::DispatchJob.perform_later(city_slug:, campaign_id:)` — só age em `sending`; publica `campaign.dispatched` `{ campaign_id, recipients_count, phones_count, sms_enabled, dispatched_by_user_id, audience }` ou `campaign.failed` `{ campaign_id, reason: "below_minimum", dispatched_by_user_id }`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/jobs/campaigns/dispatch_job_spec.rb
require "rails_helper"

RSpec.describe Campaigns::DispatchJob do
  let!(:city_record) { register_test_city! }

  before { CityProfile.create!(name: "Curitiba") }

  def run(campaign) = described_class.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)

  def sending!(audience = city_audience)
    draft_campaign!(audience: audience).tap do |c|
      c.update_columns(status: "sending", dispatched_by_user_id: c.created_by_user_id)
    end
  end

  it "congela o público, grava contagens e a chave, publica campaign.dispatched; rodar de novo não faz nada" do
    5.times { person! }
    campaign = sending!
    run(campaign)
    expect(campaign.reload).to have_attributes(status: "sent", recipients_count: 5, phones_count: 5, sms_enabled: false)
    expect(campaign.dispatched_at).to be_present
    expect { run(campaign) }.not_to change(CampaignRecipient, :count)
    expect(DomainEvent.where(name: "campaign.dispatched").map(&:payload)).to eq([
      { "campaign_id" => campaign.id, "recipients_count" => 5, "phones_count" => 5, "sms_enabled" => false,
        "dispatched_by_user_id" => campaign.dispatched_by_user_id, "audience" => city_audience }
    ])
  end

  it "público que encolheu abaixo de 5 telefones: failed/below_minimum, nenhuma linha, nenhum SMS" do
    sms_profile!(enabled: true)
    4.times { person!.tap { |p| opt_in!(p) } }
    campaign = sending!
    expect { run(campaign) }.not_to have_enqueued_job(Campaigns::SmsBatchJob)
    expect(campaign.reload).to have_attributes(status: "failed", failure_reason: "below_minimum", recipients_count: nil)
    expect(CampaignRecipient.where(campaign_id: campaign.id)).to be_empty
    expect(DomainEvent.where(name: "campaign.failed").map(&:payload)).to eq([
      { "campaign_id" => campaign.id, "reason" => "below_minimum", "dispatched_by_user_id" => campaign.dispatched_by_user_id }
    ])
  end

  it "estado do SMS por destinatário: opt-in, sem opt-in e telefone repetido" do
    sms_profile!(enabled: true)
    shared = next_phone
    first, second = %w[a b].map { |s| person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-#{s}")) }.sort_by(&:id)
    [ first, second ].each { |p| opt_in!(p) }
    not_opted_same_phone = person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-c"))
    no_opt = person!
    3.times { person!.tap { |p| opt_in!(p) } }
    campaign = sending!

    expect { run(campaign) }.to have_enqueued_job(Campaigns::SmsBatchJob)
      .with(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    statuses = campaign.recipients.to_h { |r| [ r.citizen_id, r.sms_status ] }
    expect(statuses.values_at(first.id, second.id, not_opted_same_phone.id, no_opt.id))
      .to eq(%w[pending duplicate_phone not_opted_in not_opted_in])
    expect(statuses.values.tally).to eq("pending" => 4, "duplicate_phone" => 1, "not_opted_in" => 2)
    expect(campaign.reload).to have_attributes(sms_enabled: true, recipients_count: 7, phones_count: 5)
  end

  it "o menor citizen_id COM opt-in recebe, mesmo sem ser o menor do telefone" do
    sms_profile!(enabled: true)
    shared = next_phone
    lower, higher = %w[a b].map { |s| person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-#{s}")) }.sort_by(&:id)
    opt_in!(higher)
    4.times { person! }
    campaign = sending!
    run(campaign)
    statuses = campaign.recipients.to_h { |r| [ r.citizen_id, r.sms_status ] }
    expect(statuses.values_at(lower.id, higher.id)).to eq(%w[not_opted_in pending])
  end

  it "chave desligada no congelamento: todos not_opted_in, sem lote de SMS" do
    sms_profile!(enabled: false)
    5.times { person!.tap { |p| opt_in!(p) } }
    campaign = sending!
    expect { run(campaign) }.not_to have_enqueued_job(Campaigns::SmsBatchJob)
    expect(campaign.recipients.distinct.pluck(:sms_status)).to eq([ "not_opted_in" ])
    expect(campaign.reload.sms_enabled).to be(false)
  end

  it "revogado depois da prévia não é congelado" do
    people = Array.new(6) { person! }
    campaign = sending!
    revoked_conversation!(people.first)
    run(campaign)
    expect(campaign.recipients.pluck(:citizen_id)).not_to include(people.first.id)
    expect(campaign.reload.recipients_count).to eq(5)
  end

  it "só age em campanha sending" do
    5.times { person! }
    campaign = draft_campaign!
    run(campaign)
    expect(campaign.reload.status).to eq("draft")
    expect(campaign.recipients).to be_empty
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/campaigns/dispatch_job_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::DispatchJob`.

- [ ] **Step 3: Implemente**

```ruby
# app/jobs/campaigns/dispatch_job.rb
# Congelamento do público (ADR 0024, D7; spec 2026-09-29 §5.3). Numa
# transação: FOR UPDATE na campanha (fora de sending, sai — idempotente); lê a
# chave de SMS da cidade NESTE instante; insere as linhas num savepoint e
# conta os telefones DAS LINHAS (desvio 8) — abaixo de 5, desfaz e marca
# failed. O estado do SMS de cada linha sai do próprio INSERT:
#   chave desligada ou sem opt-in → not_opted_in;
#   com opt-in, menor citizen_id com opt-in daquele telefone → pending;
#   demais com opt-in do mesmo telefone → duplicate_phone.
# O aviso aparece no wpda assim que a transação comita; o lote de SMS entra na
# fila depois do commit (enqueue_after_transaction_commit).
module Campaigns
  class DispatchJob < ApplicationJob
    include CityScopedJob
    queue_as :default

    def perform(city_slug:, campaign_id:)
      with_city(city_slug) do
        campaign = Campaign.lock.find_by(id: campaign_id)
        next unless campaign&.status == "sending"

        sms_enabled = SmsSetting.enabled?
        counts = freeze_recipients(campaign, sms_enabled)
        next fail_below_minimum(campaign) unless counts

        campaign.update!(status: "sent", sms_enabled: sms_enabled, recipients_count: counts[:recipients],
                         phones_count: counts[:phones], dispatched_at: Time.current)
        DomainEvents.publish("campaign.dispatched", campaign_id: campaign.id, recipients_count: counts[:recipients],
                                                    phones_count: counts[:phones], sms_enabled: sms_enabled,
                                                    dispatched_by_user_id: campaign.dispatched_by_user_id,
                                                    audience: campaign.audience)
        if campaign.recipients.where(sms_status: "pending").exists?
          SmsBatchJob.perform_later(city_slug: city_slug, campaign_id: campaign.id)
        end
      end
    end

    private

    def freeze_recipients(campaign, sms_enabled)
      counts = nil
      ApplicationRecord.transaction(requires_new: true) do
        ApplicationRecord.connection.execute(insert_sql(campaign, sms_enabled))
        rows = CampaignRecipient.where(campaign_id: campaign.id).joins(:citizen)
        counts = { recipients: rows.count, phones: rows.distinct.count("citizens.phone") }
        raise ActiveRecord::Rollback if counts[:phones] < Campaign::MINIMUM_PHONES
      end
      counts if counts && counts[:phones] >= Campaign::MINIMUM_PHONES
    end

    def insert_sql(campaign, sms_enabled)
      connection = ApplicationRecord.connection
      <<~SQL
        INSERT INTO campaign_recipients (id, campaign_id, citizen_id, sms_status, created_at)
        SELECT gen_random_uuid(), #{connection.quote(campaign.id)}, c.id,
               CASE
                 WHEN NOT #{sms_enabled ? 'TRUE' : 'FALSE'} OR NOT COALESCE(p.sms_opt_in, FALSE) THEN 'not_opted_in'
                 WHEN row_number() OVER (PARTITION BY c.phone, COALESCE(p.sms_opt_in, FALSE) ORDER BY c.id) = 1
                   THEN 'pending'
                 ELSE 'duplicate_phone'
               END,
               #{connection.quote(Time.current)}
        FROM citizens c
        LEFT JOIN citizen_contact_preferences p ON p.citizen_id = c.id
        WHERE c.id IN (#{Audience.new(campaign.audience).citizen_ids.to_sql})
      SQL
    end

    def fail_below_minimum(campaign)
      campaign.update!(status: "failed", failure_reason: "below_minimum")
      DomainEvents.publish("campaign.failed", campaign_id: campaign.id, reason: "below_minimum",
                                              dispatched_by_user_id: campaign.dispatched_by_user_id)
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/campaigns`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/jobs/campaigns/dispatch_job.rb spec/jobs/campaigns/dispatch_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: freeze campaign recipients at dispatch with per-phone SMS status

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: Transições com step-up (enviar, agendar, desagendar, cancelar)

**Files:**
- Create: `app/commands/campaigns/send.rb`, `schedule.rb`, `unschedule.rb`, `cancel.rb`
- Modify: `app/controllers/campaigns_controller.rb`, `config/routes.rb`
- Test: `spec/requests/campaign_transitions_spec.rb`

**Interfaces:**
- Consumes: `AudienceValidation`, `Audience#below_minimum?` (Tasks 7–8); `DispatchJob` (Task 11); `CampaignsController#respond/#set_campaign/#body_params` (Task 9); `MfaStepUp#reauthenticated_recently?`, `#require_step_up!`.
- Produces:
  - `Campaigns::Send.call(campaign:, by:)` (draft → sending, grava `dispatched_by_user`, enfileira `DispatchJob`; fail: `:invalid_transition`, `:invalid_audience`, `:below_minimum`).
  - `Campaigns::Schedule.call(campaign:, send_at:, by:)` (draft → scheduled; fail: `:invalid_transition`, `:invalid_send_at`, `:invalid_audience`, `:below_minimum`; evento `campaign.scheduled` `{ campaign_id, send_at, by_user_id }`).
  - `Campaigns::Unschedule.call(campaign:, by:)` (scheduled → draft, limpa `send_at` e `dispatched_by_user`; evento `campaign.unscheduled` `{ campaign_id, by_user_id }`).
  - `Campaigns::Cancel.call(campaign:, by:)` (draft/scheduled → cancelled; evento `campaign.cancelled` `{ campaign_id, from_status, by_user_id }`).
  - Rotas `POST /campaigns/:id/send|schedule|unschedule|cancel`; step-up checado depois do papel e do 404.

- [ ] **Step 1: Escreva a spec de request**

As rotas sem corpo recebem `{}` (o dashboard manda `{}`: `Authentication#require_json_for_cookie_writes` recusa escrita por cookie sem `application/json` com 415 `json_required`); `json_post` já manda `"{}"`.

```ruby
# spec/requests/campaign_transitions_spec.rb
require "rails_helper"

RSpec.describe "Transições da campanha", type: :request do
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:manager) do
    staff_with("campanhas@cidade.gov.br", "campaign_manager").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  let(:campaign) { draft_campaign!(by: manager) }

  def sign_in!(stepped_up: true)
    session = sign_in_as(manager)
    session.update!(mfa_verified_at: Time.current) if stepped_up
  end

  def payloads(name) = DomainEvent.where(name: name).map(&:payload)

  before { 5.times { person! } }

  it "enviar: draft → sending, quem enviou gravado e o DispatchJob na fila" do
    sign_in!
    expect { json_post "/campaigns/#{campaign.id}/send" }
      .to have_enqueued_job(Campaigns::DispatchJob).with(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    expect(response).to have_http_status(:ok)
    expect(body.dig("campaign", "status")).to eq("sending")
    expect(campaign.reload.dispatched_by_user_id).to eq(manager.id)

    json_post "/campaigns/#{campaign.id}/send"
    expect(status_and_error).to eq([ 422, "invalid_transition" ])
  end

  it "sem escrita JSON: 415 json_required antes de tudo" do
    sign_in!
    post "/campaigns/#{campaign.id}/send"
    expect(status_and_error).to eq([ 415, "json_required" ])
  end

  it "sem step-up: 401 mfa_required em enviar, agendar, desagendar e cancelar; nada muda" do
    sign_in!(stepped_up: false)
    %w[send schedule unschedule cancel].each do |action|
      json_post "/campaigns/#{campaign.id}/#{action}", send_at: 1.day.from_now.iso8601
      expect(status_and_error).to eq([ 401, "mfa_required" ]), action
    end
    expect(campaign.reload.status).to eq("draft")
  end

  it "menos de 5 telefones no envio ou no agendamento: 422 below_minimum e segue draft" do
    sign_in!
    centro = Neighborhood.create!(name: "Centro", source: "seed")
    small = draft_campaign!(by: manager, audience: { "version" => 1, "clinical" => { "all" => [] },
                                                     "geo" => { "scope" => "neighborhoods", "neighborhood_ids" => [ centro.id ] } })
    json_post "/campaigns/#{small.id}/send"
    expect(status_and_error).to eq([ 422, "below_minimum" ])
    json_post "/campaigns/#{small.id}/schedule", send_at: 1.day.from_now.iso8601
    expect(status_and_error).to eq([ 422, "below_minimum" ])
    expect(small.reload.status).to eq("draft")
  end

  it "bairro desativado depois do rascunho: enviar recusa com invalid_audience" do
    sign_in!
    centro = Neighborhood.create!(name: "Centro", source: "seed")
    5.times { person!(neighborhood: centro) }
    draft = draft_campaign!(by: manager, audience: { "version" => 1, "clinical" => { "all" => [] },
                                                     "geo" => { "scope" => "neighborhoods", "neighborhood_ids" => [ centro.id ] } })
    centro.update!(active: false)
    json_post "/campaigns/#{draft.id}/send"
    expect(body).to eq("error" => "invalid_audience",
                       "details" => [ { "path" => "/geo/neighborhood_ids/0", "message" => "inactive_or_unknown" } ])
  end

  it "agendar: de 5 minutos a 90 dias; fora disso ou malformado, invalid_send_at" do
    sign_in!
    [ 4.minutes.from_now.iso8601, 91.days.from_now.iso8601, "amanhã", nil, 12_345 ].each do |send_at|
      json_post "/campaigns/#{campaign.id}/schedule", send_at: send_at
      expect(status_and_error).to eq([ 422, "invalid_send_at" ]), send_at.inspect
    end
    at = 1.day.from_now.change(usec: 0)
    json_post "/campaigns/#{campaign.id}/schedule", send_at: at.iso8601
    expect(body["campaign"]).to include("status" => "scheduled", "send_at" => at.iso8601)
    expect(payloads("campaign.scheduled"))
      .to eq([ { "campaign_id" => campaign.id, "send_at" => at.iso8601, "by_user_id" => manager.id } ])
  end

  it "desagendar volta a draft sem horário; em draft é invalid_transition" do
    sign_in!
    json_post "/campaigns/#{campaign.id}/unschedule"
    expect(status_and_error).to eq([ 422, "invalid_transition" ])
    json_post "/campaigns/#{campaign.id}/schedule", send_at: 1.day.from_now.iso8601
    json_post "/campaigns/#{campaign.id}/unschedule"
    expect(body["campaign"]).to include("status" => "draft", "send_at" => nil)
    expect(campaign.reload.dispatched_by_user_id).to be_nil
    expect(payloads("campaign.unscheduled")).to eq([ { "campaign_id" => campaign.id, "by_user_id" => manager.id } ])
  end

  it "cancelar de draft ou de scheduled; em sending ou sent é invalid_transition" do
    sign_in!
    json_post "/campaigns/#{campaign.id}/cancel"
    expect(body.dig("campaign", "status")).to eq("cancelled")

    scheduled = draft_campaign!(by: manager)
    json_post "/campaigns/#{scheduled.id}/schedule", send_at: 1.day.from_now.iso8601
    json_post "/campaigns/#{scheduled.id}/cancel"
    expect(body.dig("campaign", "status")).to eq("cancelled")
    expect(payloads("campaign.cancelled").map { |p| p["from_status"] }).to eq(%w[draft scheduled])

    sending = draft_campaign!(by: manager)
    sending.update_columns(status: "sending", dispatched_by_user_id: manager.id)
    json_post "/campaigns/#{sending.id}/cancel"
    expect(status_and_error).to eq([ 422, "invalid_transition" ])
    json_post "/campaigns/#{sent_campaign!(by: manager).id}/cancel"
    expect(status_and_error).to eq([ 422, "invalid_transition" ])
  end

  it "papel e 404 vêm antes do step-up" do
    sign_in_as(staff_with("admin@cidade.gov.br", "municipal_admin")).update!(mfa_verified_at: Time.current)
    json_post "/campaigns/#{campaign.id}/send"
    expect(status_and_error).to eq([ 403, "missing_role" ])
    sign_in!(stepped_up: false)
    json_post "/campaigns/#{SecureRandom.uuid}/send"
    expect(status_and_error).to eq([ 404, "not_found" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/campaign_transitions_spec.rb`
Expected: FAIL (rotas inexistentes).

- [ ] **Step 3: Implemente os comandos**

```ruby
# app/commands/campaigns/send.rb
# Enviar agora (spec 2026-09-29 §5.2): draft → sending com FOR UPDATE, público
# validado e com 5 telefones ou mais, DispatchJob na fila depois do commit.
# Reasons: :invalid_transition, :invalid_audience, :below_minimum.
module Campaigns
  class Send
    def self.call(campaign:, by:)
      ApplicationRecord.transaction do
        campaign.lock!
        next Result.fail(:invalid_transition) unless campaign.status == "draft"

        audience = AudienceValidation.errors(campaign.audience)
        next Result.fail(:invalid_audience, details: { details: audience }) if audience.any?
        next Result.fail(:below_minimum) if Audience.new(campaign.audience).below_minimum?

        campaign.update!(status: "sending", dispatched_by_user: by)
        DispatchJob.perform_later(city_slug: Current.city.slug, campaign_id: campaign.id)
        Result.ok(campaign: campaign)
      end
    end
  end
end
```

```ruby
# app/commands/campaigns/schedule.rb
# Agendar (§5.2): draft → scheduled, send_at de 5 minutos a 90 dias à frente.
# Reasons: :invalid_transition, :invalid_send_at, :invalid_audience, :below_minimum.
module Campaigns
  class Schedule
    def self.call(campaign:, send_at:, by:)
      ApplicationRecord.transaction do
        campaign.lock!
        next Result.fail(:invalid_transition) unless campaign.status == "draft"

        at = parse(send_at)
        unless at && at >= Campaign::SEND_AT_MIN_LEAD.from_now && at <= Campaign::SEND_AT_MAX_AHEAD.from_now
          next Result.fail(:invalid_send_at)
        end

        audience = AudienceValidation.errors(campaign.audience)
        next Result.fail(:invalid_audience, details: { details: audience }) if audience.any?
        next Result.fail(:below_minimum) if Audience.new(campaign.audience).below_minimum?

        campaign.update!(status: "scheduled", send_at: at, dispatched_by_user: by)
        DomainEvents.publish("campaign.scheduled", campaign_id: campaign.id, send_at: at.iso8601, by_user_id: by.id)
        Result.ok(campaign: campaign)
      end
    end

    def self.parse(value)
      return nil unless value.is_a?(String)

      Time.zone.iso8601(value)
    rescue ArgumentError
      nil
    end
  end
end
```

```ruby
# app/commands/campaigns/unschedule.rb
# Desagendar (§5.1): scheduled → draft, para editar. Reason: :invalid_transition.
module Campaigns
  class Unschedule
    def self.call(campaign:, by:)
      ApplicationRecord.transaction do
        campaign.lock!
        next Result.fail(:invalid_transition) unless campaign.status == "scheduled"

        campaign.update!(status: "draft", send_at: nil, dispatched_by_user: nil)
        DomainEvents.publish("campaign.unscheduled", campaign_id: campaign.id, by_user_id: by.id)
        Result.ok(campaign: campaign)
      end
    end
  end
end
```

```ruby
# app/commands/campaigns/cancel.rb
# Cancelar (§5.1): draft ou scheduled → cancelled, final. Quem pega a linha
# primeiro vence o DueJob (FOR UPDATE × SKIP LOCKED). Reason: :invalid_transition.
module Campaigns
  class Cancel
    def self.call(campaign:, by:)
      ApplicationRecord.transaction do
        campaign.lock!
        next Result.fail(:invalid_transition) unless %w[draft scheduled].include?(campaign.status)

        from = campaign.status
        campaign.update!(status: "cancelled", cancelled_by_user: by, cancelled_at: Time.current)
        DomainEvents.publish("campaign.cancelled", campaign_id: campaign.id, from_status: from, by_user_id: by.id)
        Result.ok(campaign: campaign)
      end
    end
  end
end
```

- [ ] **Step 4: Acrescente as ações e as rotas**

Em `app/controllers/campaigns_controller.rb`, troque o `before_action :set_campaign` por:

```ruby
  before_action :set_campaign, only: %i[show update send_now schedule unschedule cancel]
```

e acrescente, depois de `def update ... end`:

```ruby

  # Enviar, agendar, desagendar e cancelar pedem step-up (D5), depois do papel e
  # do 404. `send` é método de Object: a ação é send_now (desvio 3).
  def send_now
    transition { Campaigns::Send.call(campaign: @campaign, by: Current.user) }
  end

  def schedule
    transition { Campaigns::Schedule.call(campaign: @campaign, send_at: body_params["send_at"], by: Current.user) }
  end

  def unschedule
    transition { Campaigns::Unschedule.call(campaign: @campaign, by: Current.user) }
  end

  def cancel
    transition { Campaigns::Cancel.call(campaign: @campaign, by: Current.user) }
  end
```

e, na seção `private`:

```ruby

  def transition
    return require_step_up! unless reauthenticated_recently?

    respond(yield)
  end
```

Em `config/routes.rb`, dentro do `scope "/campaigns"`, depois de `patch ":id"`:

```ruby
    post  ":id/send",       to: "campaigns#send_now"
    post  ":id/schedule",   to: "campaigns#schedule"
    post  ":id/unschedule", to: "campaigns#unschedule"
    post  ":id/cancel",     to: "campaigns#cancel"
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/campaign_transitions_spec.rb spec/requests/campaigns_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/commands/campaigns/send.rb app/commands/campaigns/schedule.rb app/commands/campaigns/unschedule.rb app/commands/campaigns/cancel.rb app/controllers/campaigns_controller.rb config/routes.rb spec/requests/campaign_transitions_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add step-up campaign transitions to send, schedule, unschedule and cancel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: `Campaigns::DueJob` (agendados que venceram)

**Files:**
- Create: `app/jobs/campaigns/due_job.rb`
- Modify: `config/recurring.yml`
- Test: `spec/jobs/campaigns/due_job_spec.rb`

**Interfaces:**
- Consumes: `DispatchJob` (Task 11); `EachCityJob`.
- Produces: `Campaigns::DueJob.perform_now` (recorrente, a cada minuto, fila `default`): `scheduled` com `send_at <= agora` → `sending` com `FOR UPDATE SKIP LOCKED`, um `DispatchJob` por campanha.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/jobs/campaigns/due_job_spec.rb
require "rails_helper"

RSpec.describe Campaigns::DueJob do
  include ActiveSupport::Testing::TimeHelpers

  let!(:city_record) { register_test_city! }

  def scheduled!(send_at) = draft_campaign!.tap { |c| c.update_columns(status: "scheduled", send_at: send_at) }

  it "solta só o que venceu (inclusive no segundo exato) e enfileira um DispatchJob por campanha" do
    freeze_time do
      late = scheduled!(1.minute.ago)
      exact = scheduled!(Time.current)
      future = scheduled!(1.minute.from_now)
      draft = draft_campaign!

      expect { described_class.perform_now }.to have_enqueued_job(Campaigns::DispatchJob).exactly(2).times
      expect([ late, exact, future, draft ].map { |c| c.reload.status }).to eq(%w[sending sending scheduled draft])
    end
  end

  it "duas execuções seguidas não enfileiram em dobro" do
    scheduled!(1.minute.ago)
    expect { described_class.perform_now; described_class.perform_now }
      .to have_enqueued_job(Campaigns::DispatchJob).exactly(1).times
  end

  it "cancelada ou desagendada antes do horário não sai" do
    cancelled = scheduled!(1.minute.from_now)
    Campaigns::Cancel.call(campaign: cancelled, by: cancelled.created_by_user)
    unscheduled = scheduled!(1.minute.from_now)
    Campaigns::Unschedule.call(campaign: unscheduled, by: unscheduled.created_by_user)
    travel_to(2.minutes.from_now) do
      expect { described_class.perform_now }.not_to have_enqueued_job(Campaigns::DispatchJob)
    end
    expect([ cancelled.reload.status, unscheduled.reload.status ]).to eq(%w[cancelled draft])
  end

  it "está no recurring.yml, a cada minuto, na fila default" do
    task = YAML.load_file(Rails.root.join("config/recurring.yml"), aliases: true).dig("production", "campaigns_due")
    expect(task).to eq("class" => "Campaigns::DueJob", "queue" => "default", "schedule" => "every minute")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/campaigns/due_job_spec.rb`
Expected: FAIL com `uninitialized constant Campaigns::DueJob`.

- [ ] **Step 3: Implemente**

```ruby
# app/jobs/campaigns/due_job.rb
# Agendados que venceram (ADR 0024; spec 2026-09-29 §5.2). Recorrente, a cada
# minuto, em cada cidade (EachCityJob). FOR UPDATE SKIP LOCKED: duas execuções
# concorrentes não pegam a mesma linha, e o PostgreSQL reavalia o
# status = 'scheduled' depois do lock — quem cancelou ou desagendou antes
# vence. O DispatchJob também é idempotente (só age em sending).
module Campaigns
  class DueJob < ApplicationJob
    prepend EachCityJob
    queue_as :default

    def perform
      ApplicationRecord.transaction do
        due = Campaign.where(status: "scheduled").where(send_at: ..Time.current)
                      .order(:send_at, :id).lock("FOR UPDATE SKIP LOCKED").to_a
        due.each do |campaign|
          campaign.update!(status: "sending")
          DispatchJob.perform_later(city_slug: Current.city.slug, campaign_id: campaign.id)
        end
      end
    end
  end
end
```

Em `config/recurring.yml`, dentro de `default: &default`, depois de `mark_no_show_appointments`:

```yaml

  campaigns_due:
    class: Campaigns::DueJob
    queue: default
    schedule: "every minute"
```

- [ ] **Step 4: Rode e veja passar, com a guarda da configuração da fila**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/campaigns spec/config/solid_queue_configuration_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/jobs/campaigns/due_job.rb config/recurring.yml spec/jobs/campaigns/due_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: release due scheduled campaigns every minute

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 14: Chave de SMS da cidade (`/campaigns/sms_setting`)

**Files:**
- Create: `app/commands/campaigns/set_sms_enabled.rb`, `app/controllers/campaign_sms_settings_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/campaign_sms_setting_spec.rb`

**Interfaces:**
- Consumes: `CampaignPolicy#read_sms_setting?/#write_sms_setting?` (Task 2); `SmsSetting`, `SmsGateway.configured?` (Task 3).
- Produces: `Campaigns::SetSmsEnabled.call(enabled:, by:) → Result` (ok: `enabled:`; fail: `:city_profile_missing`); evento `city.campaigns_sms_toggled` `{ enabled, by_user_id }` só quando muda; `GET`/`PUT /campaigns/sms_setting` → `{ enabled, gateway_configured }`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/campaign_sms_setting_spec.rb
require "rails_helper"

RSpec.describe "Chave de SMS da cidade", type: :request do
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:admin) do
    staff_with("admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end

  def sign_in_admin!(stepped_up: true)
    session = sign_in_as(admin)
    session.update!(mfa_verified_at: Time.current) if stepped_up
  end

  before { CityProfile.create!(name: "Curitiba") }

  it "campaign_manager e municipal_admin leem; os demais, 403" do
    %w[campaign_manager municipal_admin].each do |role|
      sign_in_as(staff_with("#{role}@cidade.gov.br", role))
      get "/campaigns/sms_setting"
      expect(body).to eq("enabled" => false, "gateway_configured" => true), role
    end
    sign_in_as(staff_with("viewer@cidade.gov.br", "viewer"))
    get "/campaigns/sms_setting"
    expect(status_and_error).to eq([ 403, "missing_role" ])
  end

  it "municipal_admin liga com step-up; evento só quando muda" do
    sign_in_admin!
    put "/campaigns/sms_setting", params: { enabled: true }, as: :json
    expect(body).to eq("enabled" => true, "gateway_configured" => true)
    put "/campaigns/sms_setting", params: { enabled: true }, as: :json
    expect(DomainEvent.where(name: "city.campaigns_sms_toggled").map(&:payload))
      .to eq([ { "enabled" => true, "by_user_id" => admin.id } ])
    expect(CityProfile.current.campaigns_sms_enabled).to be(true)
  end

  it "ligar sem provedor é permitido e a resposta avisa" do
    sign_in_admin!
    with_sms_gateway(nil) { put "/campaigns/sms_setting", params: { enabled: true }, as: :json }
    expect(body).to eq("enabled" => true, "gateway_configured" => false)
  end

  it "sem step-up: 401; campaign_manager não muda: 403; valor não booleano: 422 invalid_setting" do
    sign_in_admin!(stepped_up: false)
    put "/campaigns/sms_setting", params: { enabled: true }, as: :json
    expect(status_and_error).to eq([ 401, "mfa_required" ])

    sign_in_as(staff_with("campanhas@cidade.gov.br", "campaign_manager")).update!(mfa_verified_at: Time.current)
    put "/campaigns/sms_setting", params: { enabled: true }, as: :json
    expect(status_and_error).to eq([ 403, "missing_role" ])

    sign_in_admin!
    [ "sim", nil, 1 ].each do |value|
      put "/campaigns/sms_setting", params: { enabled: value }, as: :json
      expect(status_and_error).to eq([ 422, "invalid_setting" ]), value.inspect
    end
    expect(CityProfile.current.campaigns_sms_enabled).to be(false)
  end

  it "cidade sem city_profile: 409 city_profile_missing" do
    CityProfile.delete_all
    sign_in_admin!
    put "/campaigns/sms_setting", params: { enabled: true }, as: :json
    expect(status_and_error).to eq([ 409, "city_profile_missing" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/campaign_sms_setting_spec.rb`
Expected: FAIL (`GET /campaigns/sms_setting` cai em `campaigns#show` com id "sms_setting" → 403/404; o PUT não tem rota).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/campaigns/set_sms_enabled.rb
# Liga/desliga o SMS de campanha da cidade (ADR 0024 §3.4; D1). Ligar sem
# provedor configurado é permitido: os avisos saem e o SMS vira unavailable.
# Reason: :city_profile_missing (não acontece em cidade provisionada).
module Campaigns
  class SetSmsEnabled
    def self.call(enabled:, by:)
      ApplicationRecord.transaction do
        profile = CityProfile.lock.first
        next Result.fail(:city_profile_missing) unless profile

        if profile.campaigns_sms_enabled != enabled
          profile.update!(campaigns_sms_enabled: enabled)
          DomainEvents.publish("city.campaigns_sms_toggled", enabled: enabled, by_user_id: by.id)
        end
        Result.ok(enabled: enabled)
      end
    end
  end
end
```

```ruby
# app/controllers/campaign_sms_settings_controller.rb
# Chave de SMS da cidade (spec 2026-09-29 §6.1): campaign_manager e
# municipal_admin leem; só o municipal_admin muda, com step-up.
class CampaignSmsSettingsController < ApplicationController
  include Authentication
  include MfaStepUp

  wrap_parameters false

  def show
    return forbid unless policy.read_sms_setting?

    render json: setting_json
  end

  def update
    return forbid unless policy.write_sms_setting?
    return require_step_up! unless reauthenticated_recently?

    enabled = request.request_parameters["enabled"]
    return render(json: { error: "invalid_setting" }, status: :unprocessable_entity) unless [ true, false ].include?(enabled)

    result = Campaigns::SetSmsEnabled.call(enabled: enabled, by: Current.user)
    return render(json: { error: result.reason.to_s }, status: :conflict) if result.failure?

    render json: setting_json
  end

  private

  def policy
    CampaignPolicy.new(Current.user, nil)
  end

  def forbid
    render json: { error: "missing_role" }, status: :forbidden
  end

  def setting_json
    { enabled: Campaigns::SmsSetting.enabled?, gateway_configured: SmsGateway.configured? }
  end
end
```

Em `config/routes.rb`, dentro do `scope "/campaigns"`, **antes** de `get ":id"`:

```ruby
    get   "sms_setting", to: "campaign_sms_settings#show"
    put   "sms_setting", to: "campaign_sms_settings#update"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/campaign_sms_setting_spec.rb spec/requests/campaigns_spec.rb spec/requests/campaign_transitions_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/commands/campaigns/set_sms_enabled.rb app/controllers/campaign_sms_settings_controller.rb config/routes.rb spec/requests/campaign_sms_setting_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: let the municipal admin toggle campaign SMS for the city

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 5 — F-12.4 e F-12.5 (cidadão: preferências e caixa de avisos) e revogação

### Task 15: Preferências de contato do cidadão

**Files:**
- Create: `app/commands/citizens/update_contact_preferences.rb`, `app/controllers/citizen_api/contact_preferences_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/commands/citizens/update_contact_preferences_spec.rb`, `spec/requests/citizen_api/contact_preferences_spec.rb`

**Interfaces:**
- Consumes: `CitizenContactPreference` (Task 1); `Campaigns::SmsSetting` (Task 3); `CitizenApi::BaseController#render_error`, `current_citizen_session.citizens`.
- Produces: `Citizens::UpdateContactPreferences.call(citizen:, changes:) → Result` (ok: `preference:`; fail: `:invalid_preferences`); só as chaves `sms_opt_in`/`notices_muted` presentes, booleanas; `sms_opt_in_changed_at` quando o opt-in muda; evento `citizen.contact_preferences_changed` `{ citizen_id, sms_opt_in, notices_muted }` só quando algo muda. Rotas `GET /citizen/contact_preferences`, `PUT /citizen/contact_preferences/:citizen_id`.

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/commands/citizens/update_contact_preferences_spec.rb
require "rails_helper"

RSpec.describe Citizens::UpdateContactPreferences do
  let(:citizen) { person! }

  def payloads = DomainEvent.where(name: "citizen.contact_preferences_changed").map(&:payload)

  it "liga o opt-in com a hora, silencia, e publica um evento por mudança, sem telefone nem CPF" do
    freeze_time do
      described_class.call(citizen: citizen, changes: { "sms_opt_in" => true })
      expect(citizen.reload.contact_preference).to have_attributes(sms_opt_in: true, sms_opt_in_changed_at: Time.current)
    end
    described_class.call(citizen: citizen, changes: { "notices_muted" => true })
    expect(payloads).to eq([
      { "citizen_id" => citizen.id, "sms_opt_in" => true, "notices_muted" => false },
      { "citizen_id" => citizen.id, "sms_opt_in" => true, "notices_muted" => true }
    ])
  end

  it "nada muda: ok, sem linha nova e sem evento" do
    result = described_class.call(citizen: citizen, changes: { "sms_opt_in" => false, "notices_muted" => false })
    expect(result).to be_ok
    expect(CitizenContactPreference.count).to eq(0)
    expect(payloads).to be_empty
  end

  it "silenciar não mexe na hora do opt-in" do
    described_class.call(citizen: citizen, changes: { "notices_muted" => true })
    expect(citizen.reload.contact_preference.sms_opt_in_changed_at).to be_nil
  end

  it "sem chave conhecida, ou valor não booleano: invalid_preferences" do
    [ {}, { "cpf" => "1" }, { "sms_opt_in" => "true" }, { "notices_muted" => nil } ].each do |changes|
      expect(described_class.call(citizen: citizen, changes: changes).reason).to eq(:invalid_preferences), changes.inspect
    end
  end
end
```

```ruby
# spec/requests/citizen_api/contact_preferences_spec.rb
require "rails_helper"

RSpec.describe "Preferências de contato do cidadão", type: :request do
  def body = JSON.parse(response.body)

  let(:phone) { "+5541998765432" }
  let!(:ana) { person!(phone: phone, cpf: "52998224725") }
  let!(:bia) { person!(phone: phone, cpf: "11144477735") }

  it "lista cada pessoa do telefone com os padrões e a chave da cidade" do
    CityProfile.create!(name: "Curitiba", campaigns_sms_enabled: true)
    sign_in_citizen(phone)
    get "/citizen/contact_preferences"
    expect(body).to eq(
      "sms_available" => true,
      "people" => [
        { "citizen_id" => ana.id, "cpf_masked" => ana.cpf_masked, "sms_opt_in" => false, "notices_muted" => false },
        { "citizen_id" => bia.id, "cpf_masked" => bia.cpf_masked, "sms_opt_in" => false, "notices_muted" => false }
      ]
    )
  end

  it "altera a pessoa do telefone e devolve a entrada" do
    sign_in_citizen(phone)
    put "/citizen/contact_preferences/#{bia.id}", params: { sms_opt_in: true }, as: :json
    expect(body).to eq("citizen_id" => bia.id, "cpf_masked" => bia.cpf_masked, "sms_opt_in" => true, "notices_muted" => false)
    get "/citizen/contact_preferences"
    expect(body["sms_available"]).to be(false)
    expect(body["people"].map { |p| p["sms_opt_in"] }).to eq([ false, true ])
  end

  it "pessoa de outro telefone, ou id que não é UUID: 404; corpo inválido: 422 invalid_preferences" do
    other = person!
    sign_in_citizen(phone)
    [ other.id, "nao-e-uuid" ].each do |id|
      put "/citizen/contact_preferences/#{id}", params: { sms_opt_in: true }, as: :json
      expect(response).to have_http_status(:not_found)
    end
    expect(CitizenContactPreference.for(other.id)).to be_new_record
    put "/citizen/contact_preferences/#{ana.id}", params: { sms_opt_in: "sim" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_preferences" ])
  end

  it "sem sessão: 401" do
    get "/citizen/contact_preferences"
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/commands/citizens/update_contact_preferences_spec.rb spec/requests/citizen_api/contact_preferences_spec.rb`
Expected: FAIL (`uninitialized constant Citizens::UpdateContactPreferences`; rotas inexistentes).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/citizens/update_contact_preferences.rb
# Preferências de contato do cidadão (ADR 0024 §3.3; F-12.5): opt-in do SMS
# (explícito, desligado por padrão) e silêncio dos avisos. Só as chaves
# presentes; valores booleanos. Evento só quando algo muda, só com o id e os
# booleanos. Reason: :invalid_preferences.
module Citizens
  class UpdateContactPreferences
    FIELDS = %w[sms_opt_in notices_muted].freeze

    def self.call(citizen:, changes:)
      changes = changes.to_h.stringify_keys.slice(*FIELDS)
      if changes.empty? || changes.values.any? { |value| ![ true, false ].include?(value) }
        return Result.fail(:invalid_preferences)
      end

      apply(citizen, changes)
    rescue ActiveRecord::RecordNotUnique
      # Duas requisições criaram a primeira linha ao mesmo tempo: a outra venceu.
      apply(citizen, changes)
    end

    def self.apply(citizen, changes)
      ApplicationRecord.transaction do
        preference = CitizenContactPreference.lock.find_by(citizen_id: citizen.id) ||
                     CitizenContactPreference.new(citizen_id: citizen.id)
        preference.assign_attributes(changes)
        preference.sms_opt_in_changed_at = Time.current if preference.sms_opt_in_changed?
        if preference.changed?
          preference.save!
          DomainEvents.publish("citizen.contact_preferences_changed", citizen_id: citizen.id,
                                                                     sms_opt_in: preference.sms_opt_in,
                                                                     notices_muted: preference.notices_muted)
        end
        Result.ok(preference: preference)
      end
    end
    private_class_method :apply
  end
end
```

```ruby
# app/controllers/citizen_api/contact_preferences_controller.rb
# GET /citizen/contact_preferences — por pessoa do telefone da sessão (o
#   cidadão não tem nome: cpf_masked), mais sms_available (a chave da cidade).
# PUT /citizen/contact_preferences/:citizen_id { sms_opt_in?, notices_muted? }
#   — pessoa de outro telefone: 404.
module CitizenApi
  class ContactPreferencesController < BaseController
    def index
      citizens = current_citizen_session.citizens.order(:created_at).to_a
      preferences = CitizenContactPreference.where(citizen_id: citizens.map(&:id)).index_by(&:citizen_id)
      render json: {
        sms_available: Campaigns::SmsSetting.enabled?,
        people: citizens.map { |c| person_json(c, preferences[c.id] || CitizenContactPreference.new(citizen_id: c.id)) }
      }
    end

    def update
      citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
      return render_error("not_found", :not_found) unless citizen

      result = Citizens::UpdateContactPreferences.call(citizen: citizen, changes: request.request_parameters)
      return render_error(result.reason, :unprocessable_entity) if result.failure?

      render json: person_json(citizen, result.payload[:preference])
    end

    private

    def person_json(citizen, preference)
      { citizen_id: citizen.id, cpf_masked: citizen.cpf_masked, sms_opt_in: preference.sms_opt_in,
        notices_muted: preference.notices_muted }
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope "/citizen"`, depois das rotas de `appointments`:

```ruby

    get "contact_preferences",             to: "contact_preferences#index"
    put "contact_preferences/:citizen_id", to: "contact_preferences#update"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/commands/citizens/update_contact_preferences_spec.rb spec/requests/citizen_api/contact_preferences_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/commands/citizens/update_contact_preferences.rb app/controllers/citizen_api/contact_preferences_controller.rb config/routes.rb spec/commands/citizens/update_contact_preferences_spec.rb spec/requests/citizen_api/contact_preferences_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: let citizens opt in to campaign SMS and mute notices

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 16: Caixa de avisos do cidadão (`/citizen/notices`)

**Files:**
- Create: `app/controllers/citizen_api/notices_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/citizen_api/notices_spec.rb`

**Interfaces:**
- Consumes: `CampaignRecipient`, `CitizenContactPreference` (Task 1); `current_citizen_session.citizens`.
- Produces: `GET /citizen/notices` → `{ notices: [{ id, title, body, dispatched_at, read, cpf_masked }], unread_count }` (mais novo primeiro; `cpf_masked` só quando o telefone tem mais de uma pessoa; `unread_count` ignora quem silenciou); `POST /citizen/notices/:id/read` → `{ ok: true }` (idempotente; outro telefone → 404).

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/requests/citizen_api/notices_spec.rb
require "rails_helper"

RSpec.describe "Caixa de avisos do cidadão", type: :request do
  def body = JSON.parse(response.body)

  let(:phone) { "+5541998765432" }
  let!(:ana) { person!(phone: phone, cpf: "52998224725") }

  def notice!(citizen, title:, at: 1.hour.ago)
    recipient!(sent_campaign!(title: title, dispatched_at: at), citizen, sms_status: "not_opted_in")
  end

  it "lista os avisos do telefone, mais novo primeiro; uma pessoa só: sem cpf_masked" do
    older = notice!(ana, title: "Aviso antigo", at: 2.days.ago)
    newer = notice!(ana, title: "Aviso novo")
    notice!(person!, title: "De outro telefone")
    sign_in_citizen(phone)

    get "/citizen/notices"
    expect(body["notices"].map { |n| n["id"] }).to eq([ newer.id, older.id ])
    expect(body["notices"].first).to eq(
      "id" => newer.id, "title" => "Aviso novo", "body" => newer.campaign.body,
      "dispatched_at" => newer.campaign.reload.dispatched_at.iso8601, "read" => false, "cpf_masked" => nil
    )
    expect(body["unread_count"]).to eq(2)
  end

  it "telefone com duas pessoas: avisos das duas, cada um com o cpf_masked da pessoa" do
    bia = person!(phone: phone, cpf: "11144477735")
    notice!(ana, title: "Para Ana")
    notice!(bia, title: "Para Bia", at: 2.hours.ago)
    sign_in_citizen(phone)
    get "/citizen/notices"
    expect(body["notices"].map { |n| [ n["title"], n["cpf_masked"] ] })
      .to eq([ [ "Para Ana", ana.cpf_masked ], [ "Para Bia", bia.cpf_masked ] ])
  end

  it "marcar lido: conta cai; marcar lido duas vezes responde ok sem erro" do
    row = notice!(ana, title: "Aviso novo")
    sign_in_citizen(phone)
    2.times do
      json_post "/citizen/notices/#{row.id}/read"
      expect(body).to eq("ok" => true)
    end
    expect(row.reload.notice_read_at).to be_present
    get "/citizen/notices"
    expect(body["unread_count"]).to eq(0)
    expect(body["notices"].first["read"]).to be(true)
  end

  it "aviso de outro telefone, ou id que não é UUID: 404 e nada muda" do
    foreign = notice!(person!, title: "De outro telefone")
    sign_in_citizen(phone)
    [ foreign.id, "nao-e-uuid" ].each do |id|
      json_post "/citizen/notices/#{id}/read"
      expect(response).to have_http_status(:not_found)
    end
    expect(foreign.reload.notice_read_at).to be_nil
  end

  it "silenciar tira do contador só a pessoa silenciada; os avisos continuam na lista" do
    bia = person!(phone: phone, cpf: "11144477735")
    notice!(ana, title: "Para Ana")
    notice!(bia, title: "Para Bia")
    Citizens::UpdateContactPreferences.call(citizen: bia, changes: { "notices_muted" => true })
    sign_in_citizen(phone)
    get "/citizen/notices"
    expect(body["notices"].size).to eq(2)
    expect(body["unread_count"]).to eq(1)
  end

  it "sem sessão: 401; escrita sem JSON: 415" do
    get "/citizen/notices"
    expect(response).to have_http_status(:unauthorized)
    sign_in_citizen(phone)
    post "/citizen/notices/#{notice!(ana, title: 'Aviso novo').id}/read"
    expect(response).to have_http_status(:unsupported_media_type)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/citizen_api/notices_spec.rb`
Expected: FAIL (rotas inexistentes).

- [ ] **Step 3: Implemente**

```ruby
# app/controllers/citizen_api/notices_controller.rb
# Caixa de avisos (ADR 0024; F-12.4). A sessão é do TELEFONE: a caixa mostra
# os avisos de todos os cidadãos dele (D12), com cpf_masked quando há mais de
# um. unread_count ignora quem silenciou; os avisos continuam na lista.
module CitizenApi
  class NoticesController < BaseController
    def index
      citizens = current_citizen_session.citizens.to_a
      ids = citizens.map(&:id)
      masked = citizens.size > 1 ? citizens.to_h { |c| [ c.id, c.cpf_masked ] } : {}
      muted = CitizenContactPreference.where(citizen_id: ids, notices_muted: true).pluck(:citizen_id)
      rows = visible(ids).preload(:campaign).order("campaigns.dispatched_at DESC, campaign_recipients.id DESC")
      render json: {
        notices: rows.map { |row| notice_json(row, masked[row.citizen_id]) },
        unread_count: visible(ids - muted).where(notice_read_at: nil).count
      }
    end

    # Idempotente e sem corrida: só grava quando ainda está NULL (o trigger
    # recusa trocar uma leitura já gravada).
    def read
      row = visible(current_citizen_session.citizens.select(:id)).find_by(id: params[:id])
      return render_error("not_found", :not_found) unless row

      CampaignRecipient.where(id: row.id, notice_read_at: nil).update_all(notice_read_at: Time.current)
      render json: { ok: true }
    end

    private

    def visible(citizen_ids)
      CampaignRecipient.joins(:campaign).where(citizen_id: citizen_ids, campaigns: { status: "sent" })
    end

    def notice_json(row, cpf_masked)
      {
        id: row.id, title: row.campaign.title, body: row.campaign.body,
        dispatched_at: row.campaign.dispatched_at.iso8601, read: row.notice_read_at.present?, cpf_masked: cpf_masked
      }
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope "/citizen"`, antes de `get "contact_preferences"`:

```ruby
    get  "notices",          to: "notices#index"
    post "notices/:id/read", to: "notices#read"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/requests/citizen_api/notices_spec.rb spec/requests/citizen_api/contact_preferences_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/controllers/citizen_api/notices_controller.rb config/routes.rb spec/requests/citizen_api/notices_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: add the citizen notice inbox scoped to the session phone

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 17: Revogação apaga os destinatários

**Files:**
- Create: `app/services/campaigns/forget_revoked_recipients.rb`
- Modify: `app/jobs/anonymize_revoked_triage_job.rb`
- Test: `spec/jobs/anonymize_revoked_triage_job_spec.rb` (acrescentar)

**Interfaces:**
- Consumes: `CampaignRecipient` (Task 1).
- Produces: `Campaigns::ForgetRevokedRecipients.call(conversation_id:) → Integer` (linhas apagadas; 0 quando a conversa não tem cidadão ou não é a mais recente dele). `AnonymizeRevokedTriageJob#handle` passa a chamá-lo.

- [ ] **Step 1: Escreva os exemplos**

Ao fim do `RSpec.describe` de `spec/jobs/anonymize_revoked_triage_job_spec.rb`, antes do `end` final:

```ruby

  # ADR 0024 §5.6: a revogação da conversa MAIS RECENTE do cidadão apaga as
  # linhas dele em campaign_recipients; os contadores da campanha não mudam.
  it "revogar a conversa mais recente apaga as linhas de campanha do cidadão; conversa antiga não" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    old = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "revoked",
                               created_at: 3.days.ago)
    latest = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "revoked",
                                  created_at: 1.day.ago)
    campaign = sent_campaign!
    recipient!(campaign, citizen, sms_status: "not_opted_in")
    other = recipient!(campaign, Citizen.create!(cpf: "11144477735", phone: "+5541998765433"), sms_status: "not_opted_in")

    described_class.new.perform(**event_args(old.id))
    expect(CampaignRecipient.where(citizen_id: citizen.id).count).to eq(1)

    described_class.new.perform(**event_args(latest.id))
    expect(CampaignRecipient.where(citizen_id: citizen.id)).to be_empty
    expect(other.reload).to be_present
    expect(campaign.reload).to have_attributes(recipients_count: 0, phones_count: 0, status: "sent")
  end

  it "conversa sem cidadão (WhatsApp antigo): nada a apagar" do
    convo = Conversation.create!(phone: "+551135", state: "revoked")
    expect(Campaigns::ForgetRevokedRecipients.call(conversation_id: convo.id)).to eq(0)
    expect { described_class.new.perform(**event_args(convo.id)) }.not_to raise_error
  end
```

O helper `sent_campaign!` grava `recipients_count: 0, phones_count: 0`; o exemplo prova que a revogação não os muda (a campanha enviada é imutável).

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/anonymize_revoked_triage_job_spec.rb`
Expected: FAIL nos exemplos novos (`uninitialized constant Campaigns::ForgetRevokedRecipients`; linhas continuam lá).

- [ ] **Step 3: Implemente**

```ruby
# app/services/campaigns/forget_revoked_recipients.rb
# Revogação (ADR 0024 §5.6): quando a conversa revogada é a mais recente do
# cidadão (created_at, desempate por id — a mesma regra do público), as linhas
# dele em campaign_recipients são apagadas. Os contadores gravados na campanha
# não mudam; leitura e SMS são contados ao vivo e encolhem.
module Campaigns
  module ForgetRevokedRecipients
    def self.call(conversation_id:)
      conversation = Conversation.find_by(id: conversation_id)
      return 0 unless conversation&.citizen_id

      latest_id = Conversation.where(citizen_id: conversation.citizen_id)
                              .order(created_at: :desc, id: :desc).pick(:id)
      return 0 unless latest_id == conversation.id

      CampaignRecipient.where(citizen_id: conversation.citizen_id).delete_all
    end
  end
end
```

Em `app/jobs/anonymize_revoked_triage_job.rb`, acrescente ao comentário do topo `# ADR 0024: apaga também as linhas de campanha do cidadão quando a conversa revogada é a mais recente dele.` e, no fim de `handle`, depois do `find_each`:

```ruby
    Campaigns::ForgetRevokedRecipients.call(conversation_id: conversation_id)
```

- [ ] **Step 4: Rode e veja passar, com a regressão da revogação**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/jobs/anonymize_revoked_triage_job_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/invariants/territory_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add app/services/campaigns/forget_revoked_recipients.rb app/jobs/anonymize_revoked_triage_job.rb spec/jobs/anonymize_revoked_triage_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "feat: delete campaign recipients when the latest conversation is revoked

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fechamento — invariantes, semente de dev e revisão

### Task 18: Suíte de invariantes com teste de mutação

**Files:**
- Create: `spec/invariants/campaign_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1–17.
- Produces: uma seção por invariante do ADR 0024 (mínimo, revogado, opt-in e chave, texto fixo, nenhuma lista no dashboard, imutabilidade, anonimização, eventos sem PII), mais produção sem gateway e argumentos de job sem PII — cada uma com a mutação que precisa deixá-la vermelha.

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/campaign_invariants_spec.rb
# Módulo 12, critério de fechamento (ADR 0024 "Invariantes"; spec 2026-09-29
# §9.2). Cada bloco tem a mutação que precisa deixá-lo vermelho (registrada no
# relatório da entrega).
require "rails_helper"

RSpec.describe "Invariantes das campanhas (ADR 0024)", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let!(:city_record) { register_test_city! }
  let(:manager) { staff_with("campanhas@cidade.gov.br", "campaign_manager") }

  before do
    create_default_protocol!
    CityProfile.create!(name: "Curitiba")
  end

  def dispatch!(campaign)
    campaign.update_columns(status: "sending", dispatched_by_user_id: campaign.created_by_user_id)
    Campaigns::DispatchJob.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    campaign.reload
  end

  def sms_batch!(campaign)
    travel_to(Time.zone.now.change(hour: 10)) do
      Campaigns::SmsBatchJob.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    end
  end

  # Mutação: tirar o `raise ActiveRecord::Rollback` de DispatchJob#freeze_recipients,
  # ou o `below_minimum?` de Campaigns::Send.
  it "nenhuma campanha sai com menos de 5 telefones distintos" do
    shared = next_phone
    5.times { |i| person!(phone: shared, cpf: CampaignHistory.cpf_for("#{shared}-#{i}")) }
    3.times { person! }
    campaign = draft_campaign!(by: manager)
    expect(Campaigns::Send.call(campaign: campaign, by: manager).reason).to eq(:below_minimum)
    expect(dispatch!(campaign).status).to eq("failed")
    expect(CampaignRecipient.where(campaign_id: campaign.id)).to be_empty
  end

  # Mutação: tirar o `where("citizens.id NOT IN (#{REVOKED_SQL})")` de
  # Campaigns::Audience#citizen_ids.
  it "cidadão com a conversa mais recente revogada nunca entra no público" do
    revoked = person!.tap { |c| revoked_conversation!(c) }
    5.times { person! }
    campaign = dispatch!(draft_campaign!(by: manager))
    expect(campaign.recipients.pluck(:citizen_id)).not_to include(revoked.id)
    expect(Citizen.where(id: Campaigns::Audience.new(city_audience).citizen_ids)).not_to include(revoked)
  end

  # Mutação: tirar a conferência `opted.include?` de SmsBatchJob#deliver; ou,
  # no INSERT do DispatchJob, trocar `NOT #{sms_enabled ? ...}` por `NOT TRUE`.
  it "nenhum SMS sem opt-in vigente, nem com a chave desligada no congelamento" do
    people = Array.new(5) { person!.tap { |p| opt_in!(p) } }
    sms_profile!(enabled: false)
    off = dispatch!(draft_campaign!(by: manager))
    sms_profile!(enabled: true)
    sms_batch!(off)
    expect(SmsGateway::Test.deliveries).to be_empty

    on = dispatch!(draft_campaign!(by: manager))
    opt_in!(people.first, false)
    sms_batch!(on)
    expect(SmsGateway::Test.deliveries.map { |d| d[:phone] }).to match_array(people.drop(1).map(&:phone))
  end

  # Mutação: em SmsBatchJob, trocar `SmsText.body(Current.city)` por
  # "#{campaign.title}: #{SmsText.link(Current.city)}?c=#{campaign.id}".
  it "o SMS é sempre o texto fixo, e o link não carrega identificador" do
    sms_profile!(enabled: true)
    5.times { person!.tap { |p| opt_in!(p) } }
    campaign = dispatch!(draft_campaign!(by: manager, title: "Dengue no bairro"))
    sms_batch!(campaign)
    bodies = SmsGateway::Test.deliveries.map { |d| d[:body] }.uniq
    expect(bodies).to eq([ Campaigns::SmsText.body(city_record) ])
    expect(bodies.first).not_to match(/\h{8}-\h{4}-\h{4}-\h{4}-\h{12}/)
    expect(bodies.first).not_to include("Dengue", campaign.id)
    expect(bodies.first).to end_with("/wpda/avisos")
  end

  # Mutação: acrescentar `recipients: campaign.recipients.pluck(:citizen_id)` em
  # Campaigns::Presenter.full.
  it "nenhuma resposta do dashboard traz a lista de destinatários" do
    people = Array.new(5) { person! }
    campaign = dispatch!(draft_campaign!(by: manager))
    sign_in_as(manager)
    bodies = []
    [ "/campaigns", "/campaigns/#{campaign.id}", "/campaigns/options", "/campaigns/sms_setting" ].each do |path|
      get path
      bodies << response.body
    end
    json_post "/campaigns/preview", audience: city_audience
    bodies << response.body
    secrets = people.flat_map { |p| [ p.id, p.cpf, p.phone ] }
    bodies.each do |text|
      secrets.each { |secret| expect(text).not_to include(secret) }
      expect(text).not_to match(/"(recipients|citizens?_ids?)"\s*:\s*\[/)
    end
  end

  # Mutação: apagar o bloco DO $do$ dos triggers de campanha em
  # db/city_triggers.sql e recarregar os bancos de teste.
  it "a campanha enviada é imutável; do destinatário só mudam leitura e SMS" do
    campaign = sent_campaign!(by: manager)
    row = recipient!(campaign, person!)
    expect { sql_in_savepoint("UPDATE campaigns SET body = 'Texto trocado depois' WHERE id = '#{campaign.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
    expect { sql_in_savepoint("UPDATE campaign_recipients SET citizen_id = '#{person!.id}' WHERE id = '#{row.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
    expect(campaign.reload.body).to eq("A campanha de vacinação começa na segunda-feira.")
  end

  # Mutação: tirar a chamada a Campaigns::ForgetRevokedRecipients de
  # AnonymizeRevokedTriageJob#handle.
  it "a revogação que anonimiza o cidadão apaga as linhas dele em campaign_recipients" do
    citizen = person!
    conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "consented")
    Consent.create!(conversation: conversation, version: 1, policy_text_sha: "sha-teste", channel: "web",
                    given_at: Time.current)
    recipient!(sent_campaign!(by: manager), citizen, sms_status: "not_opted_in")
    RevokeConsent.call(conversation: conversation, reason: "citizen_web")
    AnonymizeRevokedTriageJob.new.handle(conversation_id: conversation.id)
    expect(CampaignRecipient.where(citizen_id: citizen.id)).to be_empty
  end

  # Mutação: pôr `citizen_ids: campaign.recipients.pluck(:citizen_id)` em
  # campaign.dispatched, ou `phone: citizen.phone` em citizen.contact_preferences_changed.
  it "nenhum payload de evento de campanha carrega CPF, telefone ou lista de cidadãos" do
    people = Array.new(5) { person! }
    centro = Neighborhood.create!(name: "Centro", source: "seed")
    attrs = { "title" => "Vacinação contra a gripe", "body" => "Procure a unidade mais próxima.", "audience" => city_audience }
    campaign = Campaigns::Create.call(attrs: attrs, by: manager).payload[:campaign]
    Campaigns::Schedule.call(campaign: campaign, send_at: 1.day.from_now.iso8601, by: manager)
    Campaigns::Unschedule.call(campaign: campaign, by: manager)
    Campaigns::Send.call(campaign: campaign, by: manager)
    Campaigns::DispatchJob.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    empty = draft_campaign!(by: manager, audience: { "version" => 1, "clinical" => { "all" => [] },
                                                     "geo" => { "scope" => "neighborhoods", "neighborhood_ids" => [ centro.id ] } })
    dispatch!(empty)
    Campaigns::Cancel.call(campaign: draft_campaign!(by: manager), by: manager)
    Campaigns::SetSmsEnabled.call(enabled: true, by: manager)
    Citizens::UpdateContactPreferences.call(citizen: people.first, changes: { "sms_opt_in" => true })
    unavailable = sent_campaign!(by: manager, sms_enabled: true)
    recipient!(unavailable, people.last)
    with_sms_gateway(nil) { sms_batch!(unavailable) }

    names = %w[campaign.created campaign.scheduled campaign.unscheduled campaign.cancelled campaign.dispatched
               campaign.failed campaign.sms_unavailable citizen.contact_preferences_changed city.campaigns_sms_toggled]
    events = DomainEvent.where(name: names).to_a
    expect(events.map(&:name).uniq).to match_array(names)
    secrets = people.flat_map { |p| [ p.cpf, p.phone, p.phone.delete("+") ] }
    events.each do |event|
      json = event.payload.to_json
      secrets.each { |secret| expect(json).not_to include(secret), "#{event.name} vazou dado pessoal" }
      expect(event.payload.values.grep(Array)).to be_empty, "#{event.name} carrega lista"
      next if event.name == "citizen.contact_preferences_changed"

      people.each { |p| expect(json).not_to include(p.id), "#{event.name} carrega id de cidadão" }
    end
  end

  # Mutação: acrescentar `config.x.sms_gateway = :log` em config/environments/production.rb.
  it "production e staging não configuram gateway de SMS: o deploy não envia SMS" do
    %w[production staging].each do |env|
      expect(File.read(Rails.root.join("config/environments/#{env}.rb"))).not_to include("sms_gateway"), env
    end
  end

  # Mutação: acrescentar `phone: citizen.phone` aos argumentos do perform_later do SmsBatchJob.
  it "jobs de campanha recebem só o slug e o id" do
    sms_profile!(enabled: true)
    5.times { person!.tap { |p| opt_in!(p) } }
    campaign = draft_campaign!(by: manager)
    Campaigns::Send.call(campaign: campaign, by: manager)
    Campaigns::DispatchJob.perform_now(city_slug: TEST_CITY_A.slug, campaign_id: campaign.id)
    jobs = ActiveJob::Base.queue_adapter.enqueued_jobs.select { |j| j["job_class"].start_with?("Campaigns::") }
    expect(jobs.map { |j| j["job_class"] }).to include("Campaigns::DispatchJob", "Campaigns::SmsBatchJob")
    jobs.each do |job|
      expect(job["arguments"].first.keys - [ "_aj_ruby2_keywords" ]).to match_array(%w[city_slug campaign_id])
    end
  end
end
```

- [ ] **Step 2: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/invariants/campaign_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 3: Teste de mutação**

Para cada comentário `# Mutação:` da suíte: aplique a mutação, rode só aquele bloco, confirme que fica **vermelho** e desfaça (`/opt/homebrew/bin/git -C apps/api/.claude/mod12 checkout -- <arquivo>`; para o trigger, recarregue com `bin/rails city:test_databases` antes e depois). Registre no relatório uma linha por mutação: arquivo, mudança, exemplo que falhou, mensagem. Mutação que passar verde = a spec não prova a invariante: conserte a spec antes de seguir.

- [ ] **Step 4: Regressão, suíte completa e custo por exemplo**

```bash
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/invariants spec/architecture spec/events
docker compose stop worker
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec --profile 10
docker compose start worker
```
Expected: `spec/invariants`, `spec/architecture` (inclusive `current_city_assignment_spec.rb` e `city_encrypted_attributes_guard_spec.rb`) e `spec/events` (guarda R18 — nenhum `Platform.audit` novo) verdes; suíte completa com 0 falhas. Custo: "Finished in N seconds" ÷ exemplos perto de **0,10 s/exemplo** com o host calmo; acima de ~0,12 s, investigue pelo `--profile` (candidatos: `CampaignHistory.no_show!` em laço, `staff_with` com BCrypt).

- [ ] **Step 5: Busca por data fixa contra o relógio real**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 diff origin/main --name-only -- spec | sed 's#^#apps/api/.claude/mod12/#' | xargs grep -nE "Time\.zone\.parse\(\"20|Date\.new\(20|travel_to\(\"20" || true
```
Expected: nenhuma ocorrência (a única data literal permitida é o `"2026-02-30"` inválido de propósito nas specs de schema e de request).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add spec/invariants/campaign_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "test: add campaign invariants suite with mutation evidence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 19: Semente de dev realista

**Files:**
- Create: `lib/campaign_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/campaign_crew_spec.rb`

**Interfaces:**
- Consumes: `CampaignHistory` (Task 5); `Citizens::UpdateContactPreferences` (Task 15); `SignatureCrew.ensure_totp/.otpauth_uri`; bairros da `Territory::Seed` e unidades do `ProfessionalCrew` (rodam antes no `db/seeds.rb`).
- Produces: `CampaignCrew.seed_current_city(slug:, ddd:, password:) → { account: { email:, role:, otpauth_uri: }, citizens: Integer, opted_in: Integer, new_histories: Integer }`. Conta `campanhas@<slug>.demo` (`campaign_manager`, TOTP fixo, env `DEV_CAMPAIGN_MANAGER_OTP_SECRET`); em dois bairros por cidade, 12 cidadãos cada: 3 faltas, 2 triagens abandonadas, 2 pedidos abertos, 1 de cada desfecho (alta, encaminhamento, retorno com pedido, saída) e 1 triado sem atendimento; metade com opt-in de SMS; um telefone com dois CPFs (ambos com falta). Com os dois bairros, "bairro + falta" dá 7 cidadãos em 6 telefones (envio liberado); com um bairro só, 3 telefones ("menos de 5").

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/lib/campaign_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/signature_crew")
require Rails.root.join("lib/campaign_crew")

RSpec.describe CampaignCrew do
  let!(:boqueirao) { Neighborhood.create!(name: "Boqueirão", source: "seed") }
  let!(:santa) { Neighborhood.create!(name: "Santa Felicidade", source: "seed") }
  let(:window) { { "from" => (Time.zone.today - 60).iso8601, "to" => Time.zone.today.iso8601 } }

  before do
    create_default_protocol!
    CityProfile.create!(name: "Curitiba")
    [ [ "UBS Jardim das Flores", "ubs" ], [ "UBS Vila Esperança", "ubs" ], [ "UPA 24h Centro", "upa" ] ]
      .each { |name, kind| create_unit(name, kind: kind) }
    staff_with("recepcao@curitiba.demo", "citizen_verifier")
  end

  def seed = described_class.seed_current_city(slug: "curitiba", ddd: "41", password: "dev-password")

  def no_show_audience(*ids)
    { "version" => 1, "geo" => { "scope" => "neighborhoods", "neighborhood_ids" => ids },
      "clinical" => { "all" => [ { "kind" => "appointment_no_show" }.merge(window) ] } }
  end

  it "cria a conta campaign_manager com TOTP e casos para cada critério em dois bairros" do
    result = seed
    expect(result[:account]).to include(email: "campanhas@curitiba.demo", role: "campaign_manager")
    user = User.find_by!(email_address: "campanhas@curitiba.demo")
    expect(user).to be_mfa_enrolled
    expect(user.has_role?(:campaign_manager)).to be(true)
    expect(result[:citizens]).to eq(25)
    expect(result[:opted_in]).to eq(CitizenContactPreference.where(sms_opt_in: true).count)
    expect(result[:opted_in]).to be_positive

    expect(Campaigns::Audience.new(no_show_audience(boqueirao.id, santa.id)).summary).to eq(citizens: 7, phones: 6)
    expect(Campaigns::Audience.new(no_show_audience(boqueirao.id)).preview).to eq(below_minimum: true)

    {
      "protocol_period" => { "protocol_name" => "triage-respiratoria" }.merge(window),
      "triage_tier" => { "tiers" => [ "alta" ] }.merge(window),
      "triage_incomplete" => window,
      "attendance_outcome" => { "outcomes" => %w[discharged referred return left] }.merge(window),
      "triaged_not_attended" => window,
      "appointment_no_show" => window,
      "appointment_request_open" => {}
    }.each do |kind, params|
      count = Citizen.where(id: Campaigns::Criteria.for(kind).relation({ "kind" => kind }.merge(params))).count
      expect(count).to be_positive, kind
    end
    expect(Citizen.all.map(&:cpf)).to all(satisfy { |cpf| CitizenIdentity::Cpf.normalize(cpf) == cpf })
  end

  it "é idempotente" do
    seed
    counts = -> { [ Citizen.count, Conversation.count, Triage.count, Attendance.count, Appointment.count ] }
    before = counts.call
    expect(seed[:new_histories]).to eq(0)
    expect(counts.call).to eq(before)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/lib/campaign_crew_spec.rb`
Expected: FAIL com `cannot load such file -- .../lib/campaign_crew`.

- [ ] **Step 3: Implemente a semente**

```ruby
# lib/campaign_crew.rb
require_relative "campaign_history"

# Semente de dev do módulo 12 (spec 2026-09-29 §10). Dev é fictício, mas imita
# o real: bairros reais da TerritoryCrew, unidades do ProfessionalCrew, CPF com
# dígito válido, histórico coerente (a falta tem pedido reaberto; o retorno tem
# pedido aberto; o atendimento tem triagem). O histórico no passado é gravado
# pelo CampaignHistory (os comandos recusam data passada); o opt-in passa pelo
# comando do domínio. Idempotente. A chave de SMS da cidade fica desligada.
class CampaignCrew
  SECRET_ENV = "DEV_CAMPAIGN_MANAGER_OTP_SECRET"
  DEFAULT_SECRET = "MNQW24DBNZUGC4ZNMRSXMLLSN52GCIJB"

  # [bairro, unidade onde a pessoa é atendida]
  PLACES = {
    "curitiba" => [ [ "Boqueirão", "UBS Vila Esperança" ], [ "Santa Felicidade", "UBS Jardim das Flores" ] ],
    "maringa" => [ [ "Zona 07", "UBS Jardim das Flores" ], [ "Jardim Alvorada", "UBS Vila Esperança" ] ]
  }.freeze

  # 3 faltas por bairro: dois bairros juntos passam do mínimo; um só, não.
  CASES = %i[no_show no_show no_show incomplete incomplete open_request open_request
             discharged referred return left triaged_only].freeze

  class << self
    def seed_current_city(slug:, ddd:, password:)
      account = ensure_manager(slug: slug, password: password)
      by = User.find_by(email_address: "profissional@#{slug}.demo") || User.find_by!(email_address: "recepcao@#{slug}.demo")
      referral_target = HealthUnit.find_by(name: "UPA 24h Centro")
      citizens = 0
      new_histories = 0

      PLACES.fetch(slug, []).each_with_index do |(neighborhood_name, unit_name), place|
        neighborhood = Neighborhood.named(neighborhood_name).first ||
                       raise("semente: bairro #{neighborhood_name} ausente (rode Territory::Seed antes)")
        unit = HealthUnit.find_by(name: unit_name) || raise("semente: unidade #{unit_name} ausente (rode ProfessionalCrew antes)")
        CASES.each_with_index do |kind, i|
          index = place * CASES.size + i
          citizen = CampaignHistory.citizen!(cpf: CampaignHistory.cpf_for("#{slug}:campaign:#{index}"),
                                             phone: phone_for(ddd, index), neighborhood: neighborhood)
          citizens += 1
          new_histories += 1 if history!(citizen, kind, index, unit: unit, target: referral_target || unit, by: by)
          opt_in!(citizen) if index.even?
        end
      end

      if (first = PLACES.fetch(slug, []).first)
        shared = CampaignHistory.citizen!(cpf: CampaignHistory.cpf_for("#{slug}:campaign:shared"), phone: phone_for(ddd, 0),
                                          neighborhood: Neighborhood.named(first[0]).first)
        citizens += 1
        new_histories += 1 if history!(shared, :no_show, 0, unit: HealthUnit.find_by!(name: first[1]), target: nil, by: by)
      end

      { account: account, citizens: citizens, opted_in: CitizenContactPreference.where(sms_opt_in: true).count,
        new_histories: new_histories }
    end

    private

    def ensure_manager(slug:, password:)
      user = User.find_or_initialize_by(email_address: "campanhas@#{slug}.demo")
      user.password = password
      user.save!
      Membership.find_or_create_by!(user: user, role: "campaign_manager") { |m| m.granted_at = Time.current }
      SignatureCrew.ensure_totp(user, secret_env: SECRET_ENV, default_secret: DEFAULT_SECRET)
      { email: user.email_address, role: "campaign_manager", otpauth_uri: SignatureCrew.otpauth_uri(user) }
    end

    # true quando gravou o histórico agora; quem já tem conversa fica como está.
    def history!(citizen, kind, index, unit:, target:, by:)
      return false if Conversation.exists?(citizen_id: citizen.id)

      at = (Time.zone.today - (5 + (index % 20))).in_time_zone.change(hour: 9)
      case kind
      when :no_show then CampaignHistory.no_show!(citizen, at: at, unit: unit, by: by)
      when :incomplete
        CampaignHistory.triage!(citizen, status: index.odd? ? "aborted_by_timeout" : "aborted_by_cancellation", at: at)
      when :open_request
        CampaignHistory.request!(citizen, kind: index.odd? ? "return" : "referral", unit: unit, target: target, by: by, at: at)
      when :return then CampaignHistory.request!(citizen, kind: "return", unit: unit, by: by, at: at)
      when :triaged_only then CampaignHistory.triage!(citizen, tier: index.odd? ? "alta" : "baixa", at: at)
      else CampaignHistory.attendance!(citizen, outcome: kind.to_s, at: at, unit: unit, by: by)
      end
      true
    end

    def opt_in!(citizen)
      Citizens::UpdateContactPreferences.call(citizen: citizen, changes: { "sms_opt_in" => true })
    end

    def phone_for(ddd, index)
      format("+55%s96666%04d", ddd, index + 1)
    end
  end
end
```

Em `db/seeds.rb`:
- depois de `require Rails.root.join("lib/territory_crew").to_s`: `require Rails.root.join("lib/campaign_crew").to_s`;
- depois do bloco do território (antes de `puts "[seeds] cidade ...`):

```ruby

        # ── Campanhas (módulo 12, spec 2026-09-29 §10) ────────────────────────
        # Depois do território e dos profissionais: usa os bairros e as unidades.
        campaigns = CampaignCrew.seed_current_city(slug: slug, ddd: ddd, password: password)
        puts "[seeds] campaign_manager   #{campaigns[:account][:email]} / #{password} + MFA → #{campaigns[:account][:otpauth_uri]}"
        puts "[seeds] campanhas .. #{campaigns[:citizens]} cidadãos (#{campaigns[:opted_in]} com opt-in de SMS, " \
             "#{campaigns[:new_histories]} históricos novos); chave de SMS da cidade desligada"
```

- [ ] **Step 4: Rode e veja passar; semeie o dev**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec spec/lib/campaign_crew_spec.rb spec/lib/territory_crew_spec.rb spec/lib/professional_crew_spec.rb
docker compose exec -T -w /rails/.claude/mod12 api bin/rails db:seed
```
Expected: PASS; o `db:seed` imprime a linha `campaign_manager` e a de campanhas para curitiba e maringa. Rode o `db:seed` duas vezes: na segunda, `0 históricos novos`.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod12 add lib/campaign_crew.rb db/seeds.rb spec/lib/campaign_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod12 commit -m "chore: seed a campaign manager and campaign audiences for development

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 20: Revisão final do api

- [ ] **Step 1:** Um subagente revisor lê `origin/main..HEAD` (código de produção, migração, triggers, semente) contra a spec inteira, o ADR 0024, os contratos e a seção "Desvios da spec", com atenção a:
  - todo `DomainEvents.publish` novo declarado no initializer e sem CPF/telefone/lista — conferir por multilinha: `grep -rn -A3 "DomainEvents.publish(" app | grep -oE '"[a-z_]+\.[a-z_]+"' | sort -u` contra os binds;
  - nenhum `perform_later` com telefone, CPF ou texto do aviso nos argumentos;
  - nenhuma atribuição de `Current.city` em `app/` ou `lib/`;
  - `/admin/api` sem rota nova de escrita; nenhuma resposta com lista de destinatários;
  - N+1 em `GET /campaigns`, `GET /citizen/notices` e no `SmsBatchJob`;
  - SQL interpolado: só ids/tempos passados por `connection.quote` e o `to_sql` de relações;
  - rotas literais de `/campaigns` antes de `:id`.
- [ ] **Step 2:** Corrija os achados, rode de novo a suíte completa (worker parado) e escreva o relatório da entrega: commits, contagem e tempo da suíte (s/exemplo), evidência de mutação, specs existentes alteradas e por quê.
- [ ] **Step 3:** Para a prova no navegador (spec §9.5, com o usuário; planos do dashboard e do wpda), suba o api do worktree no container, na porta 3031, sem derrubar o servidor principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod12 api bin/rails server -b 0.0.0.0 -p 3031 -P tmp/pids/server-mod12.pid
  ```

  O Vite do worktree de cada front aponta o proxy para `http://api:3031` (`VITE_API_PROXY_TARGET`). Login e step-up são feitos pelo usuário. Roteiro: campanha "Boqueirão + Santa Felicidade × falta" → prévia (7 cidadãos, 6 telefones) → enviar com step-up; no wpda, entrar com um telefone da semente (`+55 41 96666 0001`, que tem dois CPFs) e ler o aviso; ligar a chave (admin, step-up) e reenviar outra campanha com `:log` → `[sms]` no log do api; com `config.x.sms_gateway = nil` (edite `development.rb` só no worktree e reinicie o servidor da 3031), o painel mostra `unavailable`. O `DueJob` só roda no worker da cidade: para agendamento, confirme com `bin/rails runner 'Campaigns::DueJob.perform_now'` no worktree.
- [ ] **Step 4:** **Pare.** Merge, push, board e docs só com autorização explícita do usuário, uma etapa de cada vez. Ordem: api antes do dashboard e do wpda. Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo.

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**

| Spec | Task |
|---|---|
| §3.1–§3.4 tabelas, colunas, CHECKs, triggers `campaigns_frozen_after_send` / `campaign_recipients_append_only` | 1 |
| §3.5 papel `campaign_manager` (ROLES, PRIVILEGED_ROLES, CHECK do banco) | 1, 2 |
| §4.1 formato do público e recusas | 4 (+ referências ativas: 8) |
| §4.1 sete critérios | 5, 6 |
| §4.2–§4.4 recorte, resolução, revogados, telefones distintos, mínimo | 7 |
| §5.1–§5.2 estados, enviar/agendar/desagendar/cancelar com step-up | 12 |
| §5.2 `DueJob` recorrente | 13 |
| §5.3 `DispatchJob` (idempotência, encolheu, 4 estados de SMS, chave no instante) | 11 |
| §5.4 `SmsBatchJob` (janela, gateway ausente, falha isolada, opt-out tardio, reenfileirar) | 10 |
| §5.5 `SmsGateway` e texto fixo | 3 |
| §5.6 revogação apaga destinatários | 17 |
| §5.7 eventos | 1 (declarados), 8, 10–15 (publicados), 18 (sem PII) |
| §6.1 `/campaigns` (papéis, step-up, recusas) | 9, 12, 14 |
| §6.2 `/citizen/notices`, `/citizen/contact_preferences` | 15, 16 |
| §6.3 agregados do painel | 8 (Presenter) |
| §9.1 testes do api; §9.2 suíte de invariante | todas; 18 |
| §10 semente de dev | 19 |
| guarda `adr_pointers_spec` 1..24 | 1 |

**Placeholders:** nenhum "TBD"/"implemente depois"; cada passo de código traz o código.

**Tipos e nomes:** `Campaigns::AudienceSchema.errors/.normalize/.error/::CRITERIA` (4, 6, 8); `Campaigns::AudienceValidation.errors` (8, 9, 12); `Campaigns::Criteria.for/.period/.triage_citizens/::KINDS` (5, 6, 7, 19); `Campaigns::Audience#citizen_ids/#summary/#below_minimum?/#preview` (7, 9, 11, 12, 18, 19); `Campaigns::Presenter.summary/.full` (8, 9); `Campaigns::SmsText.body/.link`, `Campaigns::SmsSetting.enabled?`, `SmsGateway.configured?/.deliver` (3, 10, 11, 14, 15, 18); `Campaigns::{Create,Update,Send,Schedule,Unschedule,Cancel,SetSmsEnabled}` (8, 12, 14, 18); `Citizens::UpdateContactPreferences.call(citizen:, changes:)` (15, 16, 18, 19); `Campaigns::ForgetRevokedRecipients.call(conversation_id:)` (17, 18); jobs com `(city_slug:, campaign_id:)` (10–13, 18). Helpers: `city_audience`, `draft_campaign!`, `sent_campaign!`, `recipient!`, `sql_in_savepoint` (1); `with_sms_gateway`, `sms_profile!` (3); `next_phone`, `person!`, `staff!`, `unit!`, `opt_in!`, `revoked_conversation!`, `anonymous_triage!` (5); `register_test_city!` (10).

**Review Focus:** as cinco linhas têm teste na task dona (Tasks 4, 9, 11, 12, 13, 16).

**Achados do plano do dashboard incorporados (2026-09-29):** corpo `{}` nas rotas sem corpo e o 415 `json_required` (Task 12); `details[].path` como ponteiro JSON relativo ao público (desvio 4); bairro/unidade do recorte inativos recusados em prévia, criar, editar, enviar e agendar (desvio 17; Tasks 8, 9, 12); `campaign_manager` em `ROLES` e `PRIVILEGED_ROLES` (Task 2); `outcomes` em `GET /campaigns/options` (Task 9).

## Fora deste plano

- **Runbook `docs/operacao/rollout-campanhas.md`** (repo docs; spec §11): (1) migração de cidade só aditiva — publicar a imagem nova e rodar `city:migrate:all` dela **antes** de cortar tráfego; (2) `campaigns_sms_enabled` nasce `false` e `config.x.sms_gateway` não existe em production: o deploy não envia SMS; (3) ordem api → dashboard → wpda; (4) gate de go-live do SMS: contratar provedor, escrever o backend, testar em staging, só então as cidades ligam a chave; (5) o `down` da migração falha depois que alguém recebeu `campaign_manager` (memberships não se apagam) — rollback do papel é por revogação, não por migração.
- Telas do dashboard e do wpda, proxy `/campaigns` no Vite, cards do board, emenda no ADR 0024 se o usuário aprovar os desvios 7, 8, 9 e 17.
