# Cidades provisionadas + resumo de atividades — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adicionar ao Admin Console uma visão de leitura das cidades provisionadas (lista cross-tenant) e um detalhe rico por cidade (recursos provisionados + KPIs com sparklines + timeline de eventos).

**Architecture:** Endpoints read-only no namespace `Admin::Api` (operador apenas), três query objects isolados com agregação live `GROUP BY municipality_id`, reusando `Admin::Api::Period`. Frontend: novo módulo `Cities` no grupo Setup, lista↔detalhe por estado local, React Query + componentes existentes.

**Tech Stack:** Rails 8 / RSpec / PostgreSQL (RLS por ADR-0019) · React + TypeScript + Vite + @tanstack/react-query + Recharts.

## Global Constraints

- **Não é repositório git** (verificado). NÃO há passos de commit — cada task termina rodando seus testes/typecheck **verdes** (checkpoint). Se o repo for inicializado depois, commitar por task.
- **RLS (ADR-0019):** as 9 tabelas do data plane têm FORCE RLS. Toda **leitura/escrita** dessas tabelas em código e em specs deve rodar sob `ApplicationRecord.connected_to(role: :admin)` (rota_admin, BYPASSRLS). O controller herda isso via `with_admin_connection` do `Admin::Api::BaseController`; os specs devem envolver criação de fixtures **e** chamada de query em `connected_to(role: :admin)`.
- **Operador apenas:** endpoints respondem `403` para não-operador (`require_operator!`); item de nav `visible: (u) => u.operator`.
- **Read-only (§10):** nenhuma rota de escrita neste namespace.
- **Volumes respeitam período**; **estado** (canal, última atividade, conversas/protocolos ativos) é point-in-time.
- **Rodar specs backend:** `docker compose exec -T api bundle exec rspec <path>` (RAILS_ENV=test é setado pelo rails_helper; o banco `rota_saude_test` precisa estar migrado).
- **Typecheck frontend:** `docker compose exec -T admin npx tsc --noEmit -p tsconfig.json`.

---

## Task 1: `Admin::CitiesQuery` (lista + resumo)

**Files:**
- Create: `apps/api/app/queries/admin/cities_query.rb`
- Test: `apps/api/spec/queries/admin/cities_query_spec.rb`

**Interfaces:**
- Produces: `Admin::CitiesQuery.call(period:) -> Array<Hash>` onde cada hash tem chaves `:id, :name, :uf, :slug, :status, :channel ({active:, display_phone_number:} | nil), :last_activity_at (String ISO8601 | nil), :metrics ({conversations_active, protocols_active, triages_done, triages_in_progress, inbound, outbound, consents, events})`.
- Consumes: `Admin::Api::Period` (`#from`, `#to`).

- [ ] **Step 1: Write the failing test**

```ruby
# apps/api/spec/queries/admin/cities_query_spec.rb
require "rails_helper"

RSpec.describe Admin::CitiesQuery do
  TZ = ActiveSupport::TimeZone["America/Sao_Paulo"]

  def period(key = "7d")
    Admin::Api::Period.parse(key: key, from: nil, to: nil, tz: TZ)
  end

  # Cria um protocolo + conversa para pendurar uma triagem.
  def conversation_for(muni)
    Conversation.create!(municipality_id: muni.id, phone: "+55119#{rand(10_000_000)}", state: "greeting")
  end

  def protocol_for(muni)
    ProtocolDefinition.create!(municipality_id: muni.id, name: "dengue", version: 1,
                               status: "active", definition: {})
  end

  it "agrega métricas por cidade, respeitando período nos volumes e estado point-in-time" do
    result = ApplicationRecord.connected_to(role: :admin) do
      a = Municipality.create!(name: "Alpha", slug: "alpha", uf: "SP", status: "active")
      b = Municipality.create!(name: "Bravo", slug: "bravo", uf: "RJ", status: "active")

      MunicipalityChannel.create!(municipality_id: a.id, phone_number_id: "PN-A", waba_id: "W-A",
                                  display_phone_number: "+5511", access_token: "t", active: true)

      conv_a = conversation_for(a)
      prot_a = protocol_for(a)
      # triagem concluída DENTRO do período
      Triage.create!(municipality_id: a.id, conversation_id: conv_a.id, protocol_definition_id: prot_a.id,
                     protocol_name: "dengue", status: "completed", completed_at: 1.day.ago)
      # triagem concluída FORA do período (não deve contar em triages_done)
      Triage.create!(municipality_id: a.id, conversation_id: conv_a.id, protocol_definition_id: prot_a.id,
                     protocol_name: "dengue", status: "completed", completed_at: 60.days.ago)
      InboundMessage.create!(municipality_id: a.id, from: "+55", kind: "text", message_id: "m1",
                             raw: "oi", created_at: 1.day.ago)

      Admin::CitiesQuery.call(period: period)
    end

    alpha = result.find { |c| c[:slug] == "alpha" }
    bravo = result.find { |c| c[:slug] == "bravo" }

    expect(result.map { |c| c[:slug] }).to eq(%w[alpha bravo]) # ordenado por name
    expect(alpha[:metrics][:triages_done]).to eq(1)            # só a do período
    expect(alpha[:metrics][:conversations_active]).to eq(1)
    expect(alpha[:metrics][:inbound]).to eq(1)
    expect(alpha[:channel]).to eq(active: true, display_phone_number: "+5511")
    expect(alpha[:last_activity_at]).to be_present
    expect(bravo[:metrics][:triages_done]).to eq(0)
    expect(bravo[:channel]).to be_nil
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin/cities_query_spec.rb`
Expected: FAIL com `uninitialized constant Admin::CitiesQuery`.

- [ ] **Step 3: Write minimal implementation**

```ruby
# apps/api/app/queries/admin/cities_query.rb
# GET /admin/api/cities — catálogo de cidades provisionadas + resumo de atividade.
# Volumes respeitam o período; estado (canal, última atividade, conversas/protocolos
# ativos) é point-in-time. Agregação live cross-tenant — roda sob rota_admin
# (BYPASSRLS) via with_admin_connection do controller. Sem N+1: uma query
# agrupada por métrica, costurada por municipality_id.
class Admin::CitiesQuery
  ACTIVE_CONVERSATION_STATES = %w[greeting awaiting_consent consented].freeze

  def self.call(period:)
    new(period).call
  end

  def initialize(period)
    @from = period.from
    @to = period.to
  end

  def call
    munis = Municipality.order(:name).to_a

    conversations_active = Conversation.where(state: ACTIVE_CONVERSATION_STATES).group(:municipality_id).count
    triages_done         = Triage.where(status: "completed", completed_at: @from..@to).group(:municipality_id).count
    triages_in_progress  = Triage.where(status: "in_progress").group(:municipality_id).count
    inbound              = InboundMessage.where(created_at: @from..@to).group(:municipality_id).count
    outbound             = OutboundMessage.where(created_at: @from..@to).group(:municipality_id).count
    consents             = Consent.where(given_at: @from..@to).group(:municipality_id).count
    events               = DomainEvent.where(occurred_at: @from..@to).group(:municipality_id).count
    protocols_active     = ProtocolDefinition.where(status: "active").where.not(municipality_id: nil).group(:municipality_id).count
    channels             = MunicipalityChannel.order(:created_at).group_by(&:municipality_id)
    last_inbound         = InboundMessage.group(:municipality_id).maximum(:created_at)
    last_outbound        = OutboundMessage.group(:municipality_id).maximum(:created_at)
    last_event           = DomainEvent.group(:municipality_id).maximum(:occurred_at)

    munis.map do |m|
      channel = channels[m.id]&.find(&:active) || channels[m.id]&.first
      last_activity = [ last_inbound[m.id], last_outbound[m.id], last_event[m.id] ].compact.max
      {
        id: m.id,
        name: m.name,
        uf: m.uf,
        slug: m.slug,
        status: m.status,
        channel: channel && { active: channel.active, display_phone_number: channel.display_phone_number },
        last_activity_at: last_activity&.iso8601,
        metrics: {
          conversations_active: conversations_active[m.id] || 0,
          protocols_active:     protocols_active[m.id] || 0,
          triages_done:         triages_done[m.id] || 0,
          triages_in_progress:  triages_in_progress[m.id] || 0,
          inbound:              inbound[m.id] || 0,
          outbound:             outbound[m.id] || 0,
          consents:             consents[m.id] || 0,
          events:               events[m.id] || 0
        }
      }
    end
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin/cities_query_spec.rb`
Expected: PASS.

- [ ] **Step 5: Checkpoint** — confirme verde; sem git, não há commit.

---

## Task 2: rota + `Admin::Api::CitiesController#index` + gating de operador

**Files:**
- Create: `apps/api/app/controllers/admin/api/cities_controller.rb`
- Modify: `apps/api/config/routes.rb` (dentro de `namespace :admin { namespace :api }`, junto às demais rotas)
- Create: `apps/api/spec/support/admin_auth.rb`
- Test: `apps/api/spec/requests/admin/api/cities_spec.rb`

**Interfaces:**
- Consumes: `Admin::CitiesQuery.call(period:)` (Task 1); `Admin::Api::BaseController` (`with_admin_connection`, `resolve_scope`, `cross_tenant?`, `@period`).
- Produces: `GET /admin/api/cities?period=<key>` → `{ data: { cities: [...] }, as_of }`. Helper de spec `sign_in_as(user) -> Session`.

- [ ] **Step 1: Write the support helper (sign-in para request specs)**

```ruby
# apps/api/spec/support/admin_auth.rb
# Autentica request specs criando uma Session real e injetando o cookie
# ASSINADO que o concern Authentication resolve (cookies.signed[:session_id]).
# Não passa pelo fluxo de MFA — endpoints Admin::Api exigem apenas
# require_authentication + (no caso de cities) require_operator!.
module AdminAuth
  def sign_in_as(user)
    session = user.sessions.create!(user_agent: "rspec", ip_address: "127.0.0.1")
    jar = ActionDispatch::TestRequest.create.cookie_jar
    jar.signed[:session_id] = session.id
    cookies[:session_id] = jar[:session_id]
    session
  end

  def operator!(email: "op@local")
    user = User.create!(email_address: email, password: "secret123")
    Membership.create!(user: user, role: "platform_operator", municipality_id: nil, granted_at: Time.current)
    user
  end
end

RSpec.configure do |config|
  config.include AdminAuth, type: :request
  # Garante o require do support (rails_helper não auto-carrega spec/support).
end
```

Adicione no topo do spec (Step 2) `require "support/admin_auth"` OU habilite o auto-load — ver Step 2.

- [ ] **Step 2: Write the failing request spec**

```ruby
# apps/api/spec/requests/admin/api/cities_spec.rb
require "rails_helper"
require Rails.root.join("spec/support/admin_auth")

RSpec.describe "Admin::Api::Cities", type: :request do
  it "operador vê todas as cidades (cross-tenant)" do
    ApplicationRecord.connected_to(role: :admin) do
      Municipality.create!(name: "Alpha", slug: "alpha", uf: "SP", status: "active")
      Municipality.create!(name: "Bravo", slug: "bravo", uf: "RJ", status: "active")
    end
    sign_in_as(operator!)

    get "/admin/api/cities", params: { period: "7d" }

    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    slugs = body.dig("data", "cities").map { |c| c["slug"] }
    expect(slugs).to contain_exactly("alpha", "bravo")
    expect(body["as_of"]).to be_present
  end

  it "nega acesso a não-operador" do
    user = User.create!(email_address: "muni@local", password: "secret123")
    sign_in_as(user)

    get "/admin/api/cities", params: { period: "7d" }

    expect(response).to have_http_status(:forbidden)
  end

  it "exige autenticação" do
    get "/admin/api/cities", params: { period: "7d" }
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 3: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/requests/admin/api/cities_spec.rb`
Expected: FAIL (rota inexistente → `404`/routing error).

- [ ] **Step 4: Add the route**

Em `apps/api/config/routes.rb`, dentro de `namespace :admin do namespace :api do … end end`, abaixo de `get "municipalities", to: "municipalities#index"`:

```ruby
      get "cities",     to: "cities#index"
      get "cities/:id", to: "cities#show"
```

- [ ] **Step 5: Write the controller (index)**

```ruby
# apps/api/app/controllers/admin/api/cities_controller.rb
# GET /admin/api/cities[/:id] — visão de plataforma (operador) das cidades
# provisionadas + resumo de atividade. Read-only (§10). Herda do BaseController:
# auth (cookie), with_admin_connection (rota_admin/BYPASSRLS → leitura
# cross-tenant) e resolve_scope (parse do period).
class Admin::Api::CitiesController < Admin::Api::BaseController
  before_action :require_operator!

  def index
    cities = Admin::CitiesQuery.call(period: @period)
    render json: { data: { cities: cities }, as_of: Time.current.iso8601 }
  end

  private

  def require_operator!
    head :forbidden unless cross_tenant?
  end
end
```

- [ ] **Step 6: Run test to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/requests/admin/api/cities_spec.rb`
Expected: PASS (3 exemplos).

- [ ] **Step 7: Checkpoint** — verde.

---

## Task 3: `Admin::CityTimelineQuery`

**Files:**
- Create: `apps/api/app/queries/admin/city_timeline_query.rb`
- Test: `apps/api/spec/queries/admin/city_timeline_query_spec.rb`

**Interfaces:**
- Produces: `Admin::CityTimelineQuery.call(municipality:, limit: 50) -> Array<{at:, type:, summary:}>` ordenado por `occurred_at` desc.

- [ ] **Step 1: Write the failing test**

```ruby
# apps/api/spec/queries/admin/city_timeline_query_spec.rb
require "rails_helper"

RSpec.describe Admin::CityTimelineQuery do
  it "devolve eventos da cidade em ordem desc, com limit" do
    rows = ApplicationRecord.connected_to(role: :admin) do
      m = Municipality.create!(name: "Alpha", slug: "alpha", status: "active")
      DomainEvent.create!(municipality_id: m.id, name: "triage.started", payload: {}, occurred_at: 2.hours.ago)
      DomainEvent.create!(municipality_id: m.id, name: "triage.completed", payload: { "tier" => "red" }, occurred_at: 1.hour.ago)
      Admin::CityTimelineQuery.call(municipality: m, limit: 50)
    end

    expect(rows.map { |r| r[:type] }).to eq(%w[triage.completed triage.started])
    expect(rows.first[:at]).to be_present
    expect(rows.first[:summary]).to be_a(String)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin/city_timeline_query_spec.rb`
Expected: FAIL com `uninitialized constant Admin::CityTimelineQuery`.

- [ ] **Step 3: Write minimal implementation**

```ruby
# apps/api/app/queries/admin/city_timeline_query.rb
# Timeline de domain_events de uma cidade (mais recentes primeiro).
class Admin::CityTimelineQuery
  def self.call(municipality:, limit: 50)
    DomainEvent
      .where(municipality_id: municipality.id)
      .order(occurred_at: :desc)
      .limit(limit)
      .map do |e|
        { at: e.occurred_at.iso8601, type: e.name, summary: summarize(e) }
      end
  end

  def self.summarize(event)
    return event.name if event.payload.blank?
    pairs = event.payload.first(3).map { |k, v| "#{k}: #{v}" }.join(" · ")
    pairs.presence || event.name
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin/city_timeline_query_spec.rb`
Expected: PASS.

- [ ] **Step 5: Checkpoint** — verde.

---

## Task 4: `Admin::CityDetailQuery` (recursos + KPIs com sparklines)

**Files:**
- Create: `apps/api/app/queries/admin/city_detail_query.rb`
- Test: `apps/api/spec/queries/admin/city_detail_query_spec.rb`

**Interfaces:**
- Consumes: `Admin::Api::Period` (`#from`, `#to`, `#series(relation, time_column)`).
- Produces: `Admin::CityDetailQuery.call(municipality:, period:) -> { city:, resources:, kpis: }` onde:
  - `city`: `{id, name, uf, slug, ibge_code, status, channel:{active,display_phone_number}|nil, last_activity_at}`
  - `resources`: `{channel:{display_phone_number,phone_number_id,active}|nil, consent_term:{version,published_at}|nil, alert_recipients:[{channel,destination,escalation_order}], protocols_active:[{name,version}], first_admin:{email,status}|nil}`
  - `kpis`: `Array<{id, label, value, spark:[Integer]}>`

- [ ] **Step 1: Write the failing test**

```ruby
# apps/api/spec/queries/admin/city_detail_query_spec.rb
require "rails_helper"

RSpec.describe Admin::CityDetailQuery do
  TZ = ActiveSupport::TimeZone["America/Sao_Paulo"]
  def period = Admin::Api::Period.parse(key: "7d", from: nil, to: nil, tz: TZ)

  it "devolve recursos provisionados e KPIs com sparkline" do
    out = ApplicationRecord.connected_to(role: :admin) do
      m = Municipality.create!(name: "Alpha", slug: "alpha", uf: "SP", ibge_code: "3500105", status: "active")
      MunicipalityChannel.create!(municipality_id: m.id, phone_number_id: "PN", waba_id: "W",
                                  display_phone_number: "+5511", access_token: "t", active: true)
      ConsentTerm.create!(municipality_id: m.id, version: "v1", body: "termo", published_at: 1.day.ago)
      AlertRecipient.create!(municipality_id: m.id, channel: "email", destination: "ops@x", escalation_order: 0, active: true)
      ProtocolDefinition.create!(municipality_id: m.id, name: "dengue", version: 2, status: "active", definition: {})
      conv = Conversation.create!(municipality_id: m.id, phone: "+5511999", state: "greeting")
      prot = ProtocolDefinition.create!(municipality_id: m.id, name: "covid", version: 1, status: "active", definition: {})
      Triage.create!(municipality_id: m.id, conversation_id: conv.id, protocol_definition_id: prot.id,
                     protocol_name: "covid", status: "completed", completed_at: 1.day.ago)
      Admin::CityDetailQuery.call(municipality: m, period: period)
    end

    expect(out[:city][:ibge_code]).to eq("3500105")
    expect(out[:resources][:channel]).to include(phone_number_id: "PN", active: true)
    expect(out[:resources][:consent_term]).to include(version: "v1")
    expect(out[:resources][:alert_recipients].first).to include(destination: "ops@x")
    expect(out[:resources][:protocols_active].map { |p| p[:name] }).to contain_exactly("dengue", "covid")
    done = out[:kpis].find { |k| k[:id] == "triages_done" }
    expect(done[:value]).to eq(1)
    expect(done[:spark]).to be_an(Array)
    expect(done[:spark].sum).to eq(1)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin/city_detail_query_spec.rb`
Expected: FAIL com `uninitialized constant Admin::CityDetailQuery`.

- [ ] **Step 3: Write minimal implementation**

```ruby
# apps/api/app/queries/admin/city_detail_query.rb
# Detalhe de uma cidade: recursos provisionados + KPIs (valor no período +
# sparkline via Period#series). Roda sob rota_admin (BYPASSRLS).
class Admin::CityDetailQuery
  def self.call(municipality:, period:)
    new(municipality, period).call
  end

  def initialize(municipality, period)
    @m = municipality
    @period = period
  end

  def call
    { city: city_block, resources: resources_block, kpis: kpis_block }
  end

  private

  def in_period(relation, col)
    relation.where(col => @period.from..@period.to)
  end

  def channel
    @channel ||= MunicipalityChannel.where(municipality_id: @m.id).order(created_at: :desc)
                                    .find(&:active) || MunicipalityChannel.where(municipality_id: @m.id).order(created_at: :desc).first
  end

  def last_activity_at
    [ InboundMessage.where(municipality_id: @m.id).maximum(:created_at),
      OutboundMessage.where(municipality_id: @m.id).maximum(:created_at),
      DomainEvent.where(municipality_id: @m.id).maximum(:occurred_at) ].compact.max
  end

  def city_block
    {
      id: @m.id, name: @m.name, uf: @m.uf, slug: @m.slug,
      ibge_code: @m.ibge_code, status: @m.status,
      channel: channel && { active: channel.active, display_phone_number: channel.display_phone_number },
      last_activity_at: last_activity_at&.iso8601
    }
  end

  def resources_block
    term = ConsentTerm.where(municipality_id: @m.id).order(published_at: :desc).first
    first_admin = Membership.active.where(municipality_id: @m.id, role: "municipal_admin").order(:granted_at).first
    pending = Invitation.where(municipality_id: @m.id, role: "municipal_admin").order(created_at: :desc).first if first_admin.nil?
    {
      channel: channel && { display_phone_number: channel.display_phone_number, phone_number_id: channel.phone_number_id, active: channel.active },
      consent_term: term && { version: term.version, published_at: term.published_at.iso8601 },
      alert_recipients: AlertRecipient.where(municipality_id: @m.id, active: true).order(:escalation_order)
                          .map { |a| { channel: a.channel, destination: a.destination, escalation_order: a.escalation_order } },
      protocols_active: ProtocolDefinition.where(municipality_id: @m.id, status: "active").order(:name)
                          .map { |p| { name: p.name, version: p.version } },
      first_admin: first_admin_block(first_admin, pending)
    }
  end

  def first_admin_block(membership, pending)
    if membership
      { email: membership.user.email_address, status: "active" }
    elsif pending
      { email: pending.email, status: "invited" }
    end
  end

  def kpis_block
    triages_done = Triage.where(municipality_id: @m.id, status: "completed")
    inbound      = InboundMessage.where(municipality_id: @m.id)
    outbound     = OutboundMessage.where(municipality_id: @m.id)
    consents     = Consent.where(municipality_id: @m.id)
    events       = DomainEvent.where(municipality_id: @m.id)

    [
      kpi("triages_done", "Triagens concluídas", in_period(triages_done, :completed_at).count, @period.series(triages_done, :completed_at)),
      kpi("inbound",      "Mensagens recebidas", in_period(inbound, :created_at).count,         @period.series(inbound, :created_at)),
      kpi("outbound",     "Mensagens enviadas",  in_period(outbound, :created_at).count,        @period.series(outbound, :created_at)),
      kpi("consents",     "Consentimentos",      in_period(consents, :given_at).count,          @period.series(consents, :given_at)),
      kpi("events",       "Eventos",             in_period(events, :occurred_at).count,         @period.series(events, :occurred_at))
    ]
  end

  def kpi(id, label, value, spark)
    { id: id, label: label, value: value, spark: spark }
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin/city_detail_query_spec.rb`
Expected: PASS.

- [ ] **Step 5: Checkpoint** — verde.

---

## Task 5: `CitiesController#show`

**Files:**
- Modify: `apps/api/app/controllers/admin/api/cities_controller.rb` (adicionar `show`)
- Test: `apps/api/spec/requests/admin/api/cities_spec.rb` (adicionar exemplos)

**Interfaces:**
- Consumes: `Admin::CityDetailQuery.call(municipality:, period:)` (Task 4), `Admin::CityTimelineQuery.call(municipality:)` (Task 3).
- Produces: `GET /admin/api/cities/:id` → `{ data: { city:, resources:, kpis:, timeline: }, as_of }`; `404` para id inexistente.

- [ ] **Step 1: Add failing request specs**

Acrescente ao `spec/requests/admin/api/cities_spec.rb`:

```ruby
  it "show devolve recursos, kpis e timeline da cidade" do
    muni = ApplicationRecord.connected_to(role: :admin) do
      m = Municipality.create!(name: "Alpha", slug: "alpha", uf: "SP", status: "active")
      DomainEvent.create!(municipality_id: m.id, name: "triage.started", payload: {}, occurred_at: 1.hour.ago)
      m
    end
    sign_in_as(operator!)

    get "/admin/api/cities/#{muni.id}", params: { period: "7d" }

    expect(response).to have_http_status(:ok)
    data = JSON.parse(response.body)["data"]
    expect(data["city"]["slug"]).to eq("alpha")
    expect(data).to have_key("resources")
    expect(data["kpis"]).to be_an(Array)
    expect(data["timeline"].first["type"]).to eq("triage.started")
  end

  it "show retorna 404 para cidade inexistente" do
    sign_in_as(operator!)
    get "/admin/api/cities/00000000-0000-0000-0000-000000000000", params: { period: "7d" }
    expect(response).to have_http_status(:not_found)
  end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker compose exec -T api bundle exec rspec spec/requests/admin/api/cities_spec.rb -e "show"`
Expected: FAIL (action `show` inexistente → erro/`404` por motivo errado, ou `AbstractController::ActionNotFound`).

- [ ] **Step 3: Implement `show`**

Em `apps/api/app/controllers/admin/api/cities_controller.rb`, adicione antes do `private`:

```ruby
  def show
    municipality = Municipality.find_by(id: params[:id])
    return head(:not_found) unless municipality

    detail = Admin::CityDetailQuery.call(municipality: municipality, period: @period)
    timeline = Admin::CityTimelineQuery.call(municipality: municipality)
    render json: { data: detail.merge(timeline: timeline), as_of: Time.current.iso8601 }
  end
```

- [ ] **Step 4: Run to verify it passes**

Run: `docker compose exec -T api bundle exec rspec spec/requests/admin/api/cities_spec.rb`
Expected: PASS (todos os exemplos).

- [ ] **Step 5: Run the whole admin/query suite (regressão)**

Run: `docker compose exec -T api bundle exec rspec spec/queries/admin spec/requests/admin`
Expected: PASS.

- [ ] **Step 6: Checkpoint** — verde.

---

## Task 6: Tipos do frontend

**Files:**
- Modify: `apps/admin/src/lib/types.ts` (acrescentar ao final)

**Interfaces:**
- Produces: tipos `CitySummary`, `CitiesData`, `CityDetailData`, `CityResources`, `TimelineEntry`, `CityKpi` usados pelos hooks/módulo.

- [ ] **Step 1: Add types**

Acrescente ao final de `apps/admin/src/lib/types.ts`:

```ts
// ─── Cidades (Admin::Api::Cities) ────────────────────────────────────────────
export interface CityChannel { active: boolean; display_phone_number: string }

export interface CityMetrics {
  conversations_active: number;
  protocols_active: number;
  triages_done: number;
  triages_in_progress: number;
  inbound: number;
  outbound: number;
  consents: number;
  events: number;
}

export interface CitySummary {
  id: string;
  name: string;
  uf: string | null;
  slug: string;
  status: string;
  channel: CityChannel | null;
  last_activity_at: string | null;
  metrics: CityMetrics;
}

export interface CitiesData { cities: CitySummary[] }

export interface CityKpi { id: string; label: string; value: number; spark: number[] }
export interface CityResourceChannel { display_phone_number: string; phone_number_id: string; active: boolean }
export interface CityConsentTerm { version: string; published_at: string }
export interface CityAlertRecipient { channel: string; destination: string; escalation_order: number }
export interface CityProtocol { name: string; version: number }
export interface CityFirstAdmin { email: string; status: string }

export interface CityResources {
  channel: CityResourceChannel | null;
  consent_term: CityConsentTerm | null;
  alert_recipients: CityAlertRecipient[];
  protocols_active: CityProtocol[];
  first_admin: CityFirstAdmin | null;
}

export interface TimelineEntry { at: string; type: string; summary: string }

export interface CityDetailData {
  city: CitySummary & { ibge_code: string | null };
  resources: CityResources;
  kpis: CityKpi[];
  timeline: TimelineEntry[];
}
```

- [ ] **Step 2: Typecheck**

Run: `docker compose exec -T admin npx tsc --noEmit -p tsconfig.json`
Expected: EXIT 0 (tipos não usados ainda — sem erro).

- [ ] **Step 3: Checkpoint** — verde.

---

## Task 7: Hooks `useCities` / `useCityDetail`

**Files:**
- Create: `apps/admin/src/hooks/useCities.ts`
- Create: `apps/admin/src/hooks/useCityDetail.ts`

**Interfaces:**
- Consumes: `adminFetch` (lib/api), `useScope` (lib/scope), tipos da Task 6.
- Produces: `useCities()` e `useCityDetail(id: string | null)` — React Query results de `Envelope<CitiesData>` / `Envelope<CityDetailData>`.

- [ ] **Step 1: Write `useCities`**

```ts
// apps/admin/src/hooks/useCities.ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { useScope } from "../lib/scope";
import type { CitiesData } from "../lib/types";

// Cross-tenant (operador): envia só o período, nunca municipality_id.
export function useCities() {
  const scope = useScope();
  return useQuery({
    queryKey: [ "cities", scope.period ],
    queryFn: () => adminFetch<CitiesData>("/cities", { period: scope.period }),
    staleTime: 30_000
  });
}
```

- [ ] **Step 2: Write `useCityDetail`**

```ts
// apps/admin/src/hooks/useCityDetail.ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { useScope } from "../lib/scope";
import type { CityDetailData } from "../lib/types";

export function useCityDetail(id: string | null) {
  const scope = useScope();
  return useQuery({
    queryKey: [ "city", id, scope.period ],
    queryFn: () => adminFetch<CityDetailData>(`/cities/${id}`, { period: scope.period }),
    enabled: !!id,
    staleTime: 30_000
  });
}
```

- [ ] **Step 3: Typecheck**

Run: `docker compose exec -T admin npx tsc --noEmit -p tsconfig.json`
Expected: EXIT 0.

- [ ] **Step 4: Checkpoint** — verde.

---

## Task 8: Nav + `Cities.tsx` (lista)

**Files:**
- Modify: `apps/admin/src/shell/modules.ts` (ModuleId + item de nav no grupo Setup)
- Modify: `apps/admin/src/App.tsx` (import + case)
- Create: `apps/admin/src/modules/Cities.tsx` (lista; o detalhe entra na Task 9)

**Interfaces:**
- Consumes: `useCities` (Task 7), componentes `PageHeader/DataTable/StatusDot/EmptyState/ErrorState/Skeleton`, `fmtNumber/fmtTime`.
- Produces: módulo `Cities` registrado como `ModuleId "cities"`.

- [ ] **Step 1: Add ModuleId + nav item**

Em `apps/admin/src/shell/modules.ts`: no union `ModuleId`, abaixo de `| "setup_mfa";` troque por:

```ts
  | "setup_mfa"
  | "cities";
```

E no grupo Setup (`label: "Setup"`), como **primeiro** item de `items`:

```ts
      {
        id: "cities",
        label: "Cidades",
        icon: "▤",
        visible: (u) => u.operator
      },
```

- [ ] **Step 2: Wire App.tsx**

Em `apps/admin/src/App.tsx`: adicione o import junto aos demais de `modules/`:

```ts
import { Cities } from "./modules/Cities";
```

E no `switch (id)` de `renderModule`, abaixo de `case "setup_mfa":`:

```ts
    case "cities":          return <Cities onNavigate={onNavigate} />;
```

- [ ] **Step 3: Write `Cities.tsx` (lista)**

```tsx
// apps/admin/src/modules/Cities.tsx
// Cidades provisionadas (operador). Lista cross-tenant + resumo de atividade.
// O detalhe por cidade entra na Task 9 (estado local selectedCityId).
import { useState } from "react";
import { useCities } from "../hooks/useCities";
import { PageHeader } from "../components/PageHeader";
import { DataTable, type Column } from "../components/DataTable";
import { StatusDot } from "../components/StatusDot";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";
import { Skeleton } from "../components/Skeleton";
import { fmtNumber, fmtTime } from "../lib/format";
import type { CitySummary } from "../lib/types";
import type { ModuleId } from "../shell/modules";

interface Props {
  onNavigate?: (id: ModuleId) => void;
}

export function Cities({ onNavigate }: Props) {
  const [ selectedCityId, setSelectedCityId ] = useState<string | null>(null);
  const { data, isLoading, isError, error, refetch } = useCities();

  if (selectedCityId) {
    // Substituído na Task 9 por <CityDetail .../>. Por ora, volta à lista.
    return (
      <div>
        <button onClick={() => setSelectedCityId(null)} style={backBtn}>← Cidades</button>
        <p className="mono" style={{ fontSize: 11, color: "var(--ink3)" }}>detalhe — Task 9</p>
      </div>
    );
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Cidades" sub="cities" />
      {isLoading && <Skeleton rows={6} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}
      {data && data.data.cities.length === 0 && (
        <EmptyState
          title="Nenhuma cidade provisionada"
          sub="use Setup → Provisionar cidade para criar a primeira"
        />
      )}
      {data && data.data.cities.length > 0 && (
        <DataTable<CitySummary>
          cols={COLS}
          rows={data.data.cities}
          rowKey={(c) => c.id}
          onRowClick={(c) => setSelectedCityId(c.id)}
        />
      )}
    </div>
  );
}

const COLS: Column<CitySummary>[] = [
  { label: "Cidade", w: "1.4fr", render: (c) => (
      <span>{c.name}{c.uf ? ` · ${c.uf}` : ""}</span>
    ) },
  { label: "Status", w: "0.8fr", render: (c) => (
      <span style={{ display: "inline-flex", alignItems: "center", gap: 6 }}>
        <StatusDot level={c.status === "active" ? "ok" : "warn"} /> {c.status}
      </span>
    ) },
  { label: "Canal", w: "1fr", render: (c) =>
      c.channel ? `${c.channel.active ? "ativo" : "inativo"} · ${c.channel.display_phone_number}` : "—" },
  { label: "Conversas", w: "0.7fr", align: "right", render: (c) => fmtNumber(c.metrics.conversations_active) },
  { label: "Triagens", w: "0.7fr", align: "right", render: (c) => fmtNumber(c.metrics.triages_done) },
  { label: "Mensagens", w: "0.7fr", align: "right", render: (c) => fmtNumber(c.metrics.inbound + c.metrics.outbound) },
  { label: "Eventos", w: "0.7fr", align: "right", render: (c) => fmtNumber(c.metrics.events) },
  { label: "Última atividade", w: "1fr", render: (c) => c.last_activity_at ? fmtTime(c.last_activity_at) : "—" }
];

const backBtn: React.CSSProperties = {
  padding: "6px 10px", borderRadius: 8, border: "1px solid var(--rule2)",
  background: "transparent", color: "var(--ink2)", fontSize: 12, cursor: "pointer"
};
```

> Confirmado: `fmtTime(iso: string | null | undefined): string` existe em `lib/format.ts` — o uso acima está correto.

- [ ] **Step 4: Typecheck**

Run: `docker compose exec -T admin npx tsc --noEmit -p tsconfig.json`
Expected: EXIT 0.

- [ ] **Step 5: Checkpoint** — verde.

---

## Task 9: Detalhe rico da cidade

**Files:**
- Modify: `apps/admin/src/modules/Cities.tsx` (substituir o stub de detalhe por `<CityDetail/>` e adicionar o componente)

**Interfaces:**
- Consumes: `useCityDetail` (Task 7), `KpiGrid`, `StatTile`, `Panel`, `KeyValue`, `Skeleton`, `ErrorState`, `fmtTime`, tipos da Task 6.

- [ ] **Step 1: Replace the detail stub**

Em `apps/admin/src/modules/Cities.tsx`, troque o bloco `if (selectedCityId) { … }` por:

```tsx
  if (selectedCityId) {
    return <CityDetail id={selectedCityId} onBack={() => setSelectedCityId(null)} />;
  }
```

- [ ] **Step 2: Add imports**

No topo de `Cities.tsx`, adicione:

```tsx
import { useCityDetail } from "../hooks/useCityDetail";
import { KpiGrid } from "../components/KpiGrid";
import { StatTile } from "../components/StatTile";
import { Panel } from "../components/Panel";
import { KeyValue } from "../components/KeyValue";
import type { CityDetailData } from "../lib/types";
```

- [ ] **Step 3: Add the `CityDetail` component**

Acrescente ao final de `Cities.tsx`:

```tsx
function CityDetail({ id, onBack }: { id: string; onBack: () => void }) {
  const { data, isLoading, isError, error, refetch } = useCityDetail(id);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <button onClick={onBack} style={backBtn}>← Cidades</button>
      {isLoading && <Skeleton rows={8} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}
      {data && <CityDetailBody d={data.data} asOf={data.as_of} />}
    </div>
  );
}

function CityDetailBody({ d, asOf }: { d: CityDetailData; asOf: string }) {
  const { city, resources, kpis, timeline } = d;
  return (
    <>
      <PageHeader title={`${city.name}${city.uf ? ` · ${city.uf}` : ""}`} sub={city.slug} />

      <KpiGrid>
        {kpis.map((k) => (
          <StatTile key={k.id} label={k.label} value={k.value} spark={k.spark} source="live" />
        ))}
      </KpiGrid>

      <div style={{ display: "grid", gridTemplateColumns: "1fr 1.2fr", gap: 14 }}>
        <Panel title="Recursos provisionados" asOf={asOf}>
          <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
            <KeyValue k="Status" v={city.status} />
            <KeyValue k="IBGE" v={city.ibge_code ?? "—"} />
            <KeyValue k="Canal" v={resources.channel ? `${resources.channel.display_phone_number} (${resources.channel.active ? "ativo" : "inativo"})` : "—"} />
            <KeyValue k="Termo LGPD" v={resources.consent_term ? `${resources.consent_term.version}` : "—"} />
            <KeyValue k="1º admin" v={resources.first_admin ? `${resources.first_admin.email} (${resources.first_admin.status})` : "—"} />
            <KeyValue k="Alertas" v={resources.alert_recipients.length ? resources.alert_recipients.map((a) => `${a.channel}:${a.destination}`).join(", ") : "—"} />
            <KeyValue k="Protocolos ativos" v={resources.protocols_active.length ? resources.protocols_active.map((p) => `${p.name} v${p.version}`).join(", ") : "—"} />
          </div>
        </Panel>

        <Panel title="Atividade recente" asOf={asOf}>
          {timeline.length === 0 ? (
            <EmptyState title="Sem eventos no histórico" />
          ) : (
            <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 8 }}>
              {timeline.map((e, i) => (
                <li key={i} style={{ display: "flex", flexDirection: "column", gap: 2, borderBottom: "1px solid var(--rule)", paddingBottom: 6 }}>
                  <span className="mono" style={{ fontSize: 11, color: "var(--accent)" }}>{e.type}</span>
                  <span style={{ fontSize: 12, color: "var(--ink2)" }}>{e.summary}</span>
                  <span className="mono" style={{ fontSize: 10.5, color: "var(--ink3)" }}>{fmtTime(e.at)}</span>
                </li>
              ))}
            </ul>
          )}
        </Panel>
      </div>
    </>
  );
}
```

> Confirmado: `KeyValue` usa props `k` (string) / `v` (ReactNode) — as chamadas acima estão corretas.

- [ ] **Step 4: Typecheck**

Run: `docker compose exec -T admin npx tsc --noEmit -p tsconfig.json`
Expected: EXIT 0.

- [ ] **Step 5: Checkpoint** — verde.

---

## Task 10: Verificação manual ponta a ponta

**Files:** nenhum (validação).

- [ ] **Step 1: Garantir operador logado** — `dev@local` é `platform_operator` com MFA (já configurado nesta sessão). Subir stack: `./start.sh` (ou `docker compose up -d`).

- [ ] **Step 2: Provisionar uma cidade de teste** — admin (http://localhost:5174/admin/) → login `dev@local` + TOTP → Setup → **Provisionar cidade** → "Preencher com dados de teste" → Provisionar.

- [ ] **Step 3: Abrir Cidades** — Setup → **Cidades**. Verificar que a cidade aparece na lista com Status/Canal/contadores/última atividade.

- [ ] **Step 4: Abrir o detalhe** — clicar na linha. Verificar: KPIs com sparkline, painel "Recursos provisionados" (canal, termo, 1º admin convidado, alertas, protocolos) e "Atividade recente" (eventos, ex.: `municipality.provisioned`). Botão "← Cidades" volta à lista.

- [ ] **Step 5: Conferir gating** — confirmar que o item "Cidades" só aparece para operador (não-operador não vê / endpoint `403`).

---

## Self-Review (preenchido)

**Cobertura do spec:** §3.1 backend → Tasks 1–5; §3.2 frontend → Tasks 6–9; §4 métricas → Tasks 1 e 4 (colunas/estados exatos); §5 contratos → Tasks 2/5 (envelopes); §6 erros → Tasks 2/5 (403/404/401) e 8/9 (Empty/Error/Skeleton); §7 testes → specs nas Tasks 1,3,4 (queries) e 2,5 (requests) + Task 10 (manual). Sem lacunas.

**Placeholders:** o detalhe da Task 8 usa um stub *intencional* removido na Task 9 (declarado). Sem TBD/TODO reais. Duas notas pedem confirmar assinaturas de `fmtTime` e `KeyValue` (componentes já existentes) — não são placeholders de lógica.

**Consistência de tipos:** `CitySummary/CitiesData/CityDetailData` (Task 6) batem com o que hooks (Task 7) e módulo (Tasks 8/9) consomem e com os envelopes do backend (Tasks 2/5). Nomes de métricas idênticos entre `CitiesQuery` (Task 1) e `CityMetrics` (Task 6).
