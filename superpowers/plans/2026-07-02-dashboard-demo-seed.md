# Dashboard Demo Seed Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An opt-in demo seed loader that populates every dashboard view (Aquisição / Triagem / Governança) for two municipalities with realistic, idempotent data spread over the last 30 days.

**Architecture:** A `DashboardDemo` Ruby module in `db/seeds/dashboard_demo.rb`, run by a `db:seed:demo` rake task, all writes wrapped in `ApplicationRecord.connected_to(role: :admin)` (BYPASSRLS). Rows are created idempotently via deterministic natural keys. Base `db/seeds.rb` invokes it only when `SEED_DASHBOARD_DEMO=1`. A companion `db:seed:demo:verify` rake task asserts every panel query is non-empty for both cities.

**Tech Stack:** Rails 8.1, PostgreSQL, run inside the `api-dev` Docker container.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api …` or run from `apps/api`.
- **Everything runs in the container:** `docker exec api-dev bundle exec rails …`.
- **All demo writes AND all verification reads run under `ApplicationRecord.connected_to(role: :admin)`** — tenant tables are RLS-enforced and the seed runs without `SET LOCAL`.
- **Idempotent by deterministic natural keys** (phone, `message_id`, `idempotency_key`, protocol `name+version+municipality_id`, report `triage_id`, event `payload.demo_id`). Re-running `db:seed:demo` must create nothing new. No `Math`/`rand` randomness — distributions are index-driven.
- **Tier vocabulary is `low/medium/high`** on `Triage.tier` (Classificação buckets `TIERS = %w[low medium high]`, `classification_query.rb:6`). `ReportSnapshot.outcome["tier"]` uses the same values.
- **Ack status is a literal small int 0–5** (`ingestion_query.rb:37-42`): `{0,1,2}`→ok, `{3}`→warn, `{4,5}`→err. NOT HTTP codes.
- **Trail events use BARE names** `scored / rule_matched / priority_rule / tier_assigned` with `payload.triage_id` (`triage_trail_query.rb:6,24-25`). Events-panel filters need SEPARATE prefixed events (`triage.* consent.* conversation.* protocol.* priority.*`, `events_query.rb:33`).
- **Extend, don't duplicate the baseline.** `db/seeds.rb` already creates Curitiba + 1 channel + `triage-respiratoria` v1 active + 1 conversation + 1 triage + 1 report; the loader adds rows with distinct keys and reuses the existing municipality/protocol.

---

### Task 1: Loader scaffold, rake task, seeds.rb hook

**Files:**
- Create: `apps/api/db/seeds/dashboard_demo.rb`
- Create: `apps/api/lib/tasks/seed_demo.rake`
- Modify: `apps/api/db/seeds.rb` (append a guarded call)

**Interfaces:**
- Produces: `DashboardDemo.run!` (builds all cities, returns a counts hash); helpers `DashboardDemo.at_days_ago(days, hour:)`, `spread_days(i, n)`, `scaled(base, city)`, `upsert_municipality(city)`; constants `CITIES`, `ACTOR_A`, `ACTOR_B`. Later tasks add `build_protocols`, `build_conversations_and_consents`, `build_ingestion`, `build_triages_and_reports`, `build_events` and wire them into `run!`.

- [ ] **Step 1: Create the loader skeleton**

Create `apps/api/db/seeds/dashboard_demo.rb`:

```ruby
# Opt-in demo seed: populates every dashboard view (Aquisição/Triagem/Governança)
# for two municipalities, idempotently, spread over the last 30 days.
# Run: bin/rails db:seed:demo   (verify: bin/rails db:seed:demo:verify)
#
# All writes run under the admin (BYPASSRLS) connection because conversations,
# triages, consents, inbound/outbound messages, protocol_definitions,
# report_snapshots and domain_events are RLS-enforced and the seed runs without
# SET LOCAL. Idempotent via deterministic natural keys.
module DashboardDemo
  module_function

  CITIES = [
    { slug: "curitiba", name: "Curitiba Demo", uf: "PR", scale: 1.0, code: "CWB", ddd: "41" },
    { slug: "londrina", name: "Londrina Demo", uf: "PR", scale: 0.4, code: "LDB", ddd: "43" }
  ].freeze

  ACTOR_A = "ana@curitiba.demo".freeze
  ACTOR_B = "bruno@curitiba.demo".freeze

  # Deterministic timestamp `days` ago at a fixed hour/minute (no randomness).
  def at_days_ago(days, hour: 10)
    (Time.current - days.to_i.days).change(hour: hour, min: (days.to_i * 7) % 60, sec: 0)
  end

  # Map index i in [0, n) to a day offset in [0, 29], deterministically.
  def spread_days(i, n)
    n <= 1 ? 0 : ((i * 29.0) / (n - 1)).round
  end

  # Scale a base count by the city's factor (min 1).
  def scaled(base, city)
    [(base * city[:scale]).round, 1].max
  end

  def upsert_municipality(city)
    Municipality.find_or_create_by!(slug: city[:slug]) do |m|
      m.name = city[:name]
      m.uf = city[:uf]
      m.status = "active"
    end
  end

  def run!
    ApplicationRecord.connected_to(role: :admin) do
      CITIES.each do |city|
        upsert_municipality(city)
      end
    end
    report_counts
  end

  def report_counts
    ApplicationRecord.connected_to(role: :admin) do
      counts = {
        municipalities: Municipality.where(slug: CITIES.map { |c| c[:slug] }).count
      }
      puts "[dashboard_demo] #{counts.inspect}"
      counts
    end
  end
end

DashboardDemo.run!
```

- [ ] **Step 2: Create the rake task**

Create `apps/api/lib/tasks/seed_demo.rake`:

```ruby
namespace :db do
  namespace :seed do
    desc "Load the dashboard demo dataset (idempotent)"
    task demo: :environment do
      load Rails.root.join("db/seeds/dashboard_demo.rb")
    end
  end
end
```

- [ ] **Step 3: Add the guarded hook to db/seeds.rb**

At the very end of `apps/api/db/seeds.rb` (after the existing `end` that closes the `if Rails.env.production? … else … end` block — i.e. the last line of the file), append:

```ruby

# Optional heavy demo dataset for the dashboard. Off by default; base seed stays
# lean. Enable with SEED_DASHBOARD_DEMO=1 bin/rails db:seed  (or bin/rails db:seed:demo).
if ENV["SEED_DASHBOARD_DEMO"] == "1" && !Rails.env.production?
  load Rails.root.join("db/seeds/dashboard_demo.rb")
end
```

- [ ] **Step 4: Run the loader and verify it creates Londrina, idempotently**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails db:seed:demo
```
Expected: both runs print `[dashboard_demo] {:municipalities=>2}` (Curitiba already existed from the baseline; Londrina created on the first run, unchanged on the second).

Confirm no duplicate municipality:
```bash
docker exec api-dev bundle exec rails runner 'ApplicationRecord.connected_to(role: :admin){ puts Municipality.where(slug: %w[curitiba londrina]).count }'
```
Expected: `2`.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb lib/tasks/seed_demo.rake db/seeds.rb
git -C apps/api commit -m "Scaffold dashboard demo seed loader and rake task"
```

---

### Task 2: Protocol definitions (Governança — Protocolos)

**Files:**
- Modify: `apps/api/db/seeds/dashboard_demo.rb`

**Interfaces:**
- Consumes: `upsert_municipality`, `CITIES`.
- Produces: `build_protocols(city, muni)` → returns `{ "triage-respiratoria" => <active PD>, "triagem-dengue" => <active PD> }`; helpers `demo_definition(name, version)`, `upsert_protocol(muni, name, version, status)`.

- [ ] **Step 1: Add protocol builders**

In `apps/api/db/seeds/dashboard_demo.rb`, add these methods inside the module (before `def run!`):

```ruby
  # A valid protocol definition (passes Protocols::Validator, same shape as the
  # baseline seed). Recommendations keyed in pt-BR for the public report.
  def demo_definition(name, version)
    {
      "name" => name, "version" => version, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Você está com tosse?", "answer_type" => "boolean",
          "branches" => { "true" => "febre", "false" => nil }, "weights" => { "true" => 3, "false" => 0 } },
        { "id" => "febre", "prompt" => "Está com febre alta?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                     "priority_map" => { "baixa" => 9, "alta" => 1 } },
      "recommendations" => {
        "alta"  => { "title" => "Procure atendimento hoje", "body" => "Prioridade alta. Procure a unidade mais próxima." },
        "baixa" => { "title" => "Cuidados em casa", "body" => "Repouso e hidratação; se piorar, procure sua unidade." }
      }
    }
  end

  def upsert_protocol(muni, name, version, status)
    ProtocolDefinition.find_or_create_by!(name: name, version: version, municipality_id: muni.id) do |p|
      p.status = status
      p.definition = demo_definition(name, version)
    end
  end

  # Multiple versions/statuses so Protocolos (list, published count, versions
  # detail) is rich. v1 active reuses the baseline row for Curitiba.
  def build_protocols(city, muni)
    resp1 = upsert_protocol(muni, "triage-respiratoria", 1, "active")
    upsert_protocol(muni, "triage-respiratoria", 2, "published")
    upsert_protocol(muni, "triage-respiratoria", 3, "draft")
    dengue1 = upsert_protocol(muni, "triagem-dengue", 1, "active")
    upsert_protocol(muni, "triagem-dengue", 2, "retired")
    { "triage-respiratoria" => resp1, "triagem-dengue" => dengue1 }
  end
```

- [ ] **Step 2: Wire into run!**

In `def run!`, change the city loop body to capture protocols:

```ruby
      CITIES.each do |city|
        muni = upsert_municipality(city)
        build_protocols(city, muni)
      end
```

- [ ] **Step 3: Run and verify protocols populate**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails runner '
ApplicationRecord.connected_to(role: :admin) do
  m = Municipality.find_by(slug: "curitiba")
  out = Admin::ProtocolsQuery.call(municipality: m)
  statuses = out[:protocols].map { |p| p[:status] }
  puts "rows=#{out[:protocols].size} published=#{statuses.count("published")} statuses=#{statuses.tally.inspect}"
end'
```
Expected: `rows` ≥ 5, `published` ≥ 1, statuses include `active`, `published`, `draft`, `retired`.

- [ ] **Step 4: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb
git -C apps/api commit -m "Seed protocol definitions across versions and statuses"
```

---

### Task 3: Conversations + consents (Aquisição — Conversas, Consentimentos)

**Files:**
- Modify: `apps/api/db/seeds/dashboard_demo.rb`

**Interfaces:**
- Consumes: `upsert_municipality`, `at_days_ago`, `spread_days`, `scaled`.
- Produces: `build_conversations_and_consents(city, muni)` → returns an array of `{ convo:, state:, days:, seq: }` for conversations in state `consented`/`completed` (the ones that will get triages); constant `STATE_DIST`.

- [ ] **Step 1: Add the conversations/consents builder**

In `apps/api/db/seeds/dashboard_demo.rb`, add (before `def run!`):

```ruby
  # FSM state distribution (Curitiba base; scaled per city). Covers every funnel
  # bucket (greeting/awaiting_consent/consented), live (awaiting_consent+
  # consented), and exits (revoked). "completed" is a terminal state kept out of
  # the funnel by design.
  STATE_DIST = { "greeting" => 4, "awaiting_consent" => 4, "consented" => 12,
                 "completed" => 8, "revoked" => 3, "abandoned" => 3 }.freeze

  def build_conversations_and_consents(city, muni)
    triage_convos = []
    seq = 0
    STATE_DIST.each do |state, base|
      scaled(base, city).times do
        seq += 1
        phone = format("+55%s9%08d", city[:ddd], seq)
        # Coprime scatter over [0,29]: interleaves states across the whole 30d
        # window (and the 7d sub-window) instead of clustering each state on one
        # day. 13 is coprime with 30 so 34 conversations spread evenly.
        days  = (seq * 13) % 30
        convo = Conversation.find_or_create_by!(municipality_id: muni.id, phone: phone) do |c|
          c.state = state
          c.created_at = at_days_ago(days, hour: 9)
        end

        if %w[consented completed revoked].include?(state)
          version = seq.even? ? 2 : 1
          Consent.where(conversation_id: convo.id).first || Consent.create!(
            conversation: convo, municipality_id: muni.id, version: version,
            channel: "whatsapp", policy_text_sha: "demo-policy-sha-v#{version}",
            given_at: at_days_ago(days, hour: 11),
            revoked_at: (state == "revoked" ? at_days_ago([days - 1, 0].max, hour: 12) : nil)
          )
        end

        triage_convos << { convo: convo, state: state, days: days, seq: seq } if %w[consented completed].include?(state)
      end
    end
    triage_convos
  end
```

Note: `ConsentTerm` is intentionally omitted — no dashboard view reads it (`consent_query` groups by `Consent.version` directly).

- [ ] **Step 2: Wire into run!**

Update the city loop in `def run!`:

```ruby
      CITIES.each do |city|
        muni = upsert_municipality(city)
        build_protocols(city, muni)
        build_conversations_and_consents(city, muni)
      end
```

- [ ] **Step 3: Run and verify funnel + consent panels**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails runner '
ApplicationRecord.connected_to(role: :admin) do
  m = Municipality.find_by(slug: "curitiba")
  p = Admin::Api::Period.parse(key: "30d", from: nil, to: nil, tz: ActiveSupport::TimeZone["America/Sao_Paulo"])
  cv = Admin::ConversationsQuery.call(municipality: m, period: p)
  co = Admin::ConsentQuery.call(municipality: m, period: p)
  puts "funnel=#{cv[:funnel].map { |f| [f[:key], f[:count]] }.to_h} live=#{cv[:live]} exits=#{cv[:exits].map { |e| [e[:key], e[:count]] }.to_h}"
  puts "consent given=#{co[:given]} revoked=#{co[:revoked]} byVersion=#{co[:byVersion].size}"
end'
```
Expected: every funnel key (`greeting`, `awaiting_consent`, `consented`) has count ≥ 1; `live` ≥ 1; `exits` `revoked` ≥ 1; consent output shows non-zero given and revoked counts.

- [ ] **Step 4: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb
git -C apps/api commit -m "Seed conversations across FSM states and their consents"
```

---

### Task 4: Ingestion (Aquisição — Ingestão)

**Files:**
- Modify: `apps/api/db/seeds/dashboard_demo.rb`

**Interfaces:**
- Consumes: `at_days_ago`, `spread_days`, `scaled`.
- Produces: `build_ingestion(city, muni)` (creates `InboundMessage` and `OutboundMessage` rows).

- [ ] **Step 1: Add the ingestion builder**

In `apps/api/db/seeds/dashboard_demo.rb`, add (before `def run!`):

```ruby
  # Inbound archive + outbound ack log. Inbound spread over 30d (day 0 → <24h
  # pending; days ≥ 2 → >24h backlog/overTtl). Outbound status is a literal
  # 0–5 int: {0,1,2}=ok, {3}=warn, {4,5}=err (ingestion_query.rb:37-42).
  ACK_STATUS_CYCLE = [0, 1, 2, 0, 1, 2, 3, 4, 5, 2, 0, 3, 4].freeze

  def build_ingestion(city, muni)
    n_in = scaled(40, city)
    n_in.times do |i|
      InboundMessage.find_or_create_by!(message_id: "IN-#{city[:code]}-#{format('%04d', i + 1)}") do |m|
        m.from = format("+55%s9%08d", city[:ddd], i + 1)
        m.kind = "text"
        m.municipality_id = muni.id
        m.raw = "demo inbound message #{i + 1}"
        m.created_at = at_days_ago(spread_days(i, n_in), hour: (i % 12) + 8)
      end
    end

    n_out = scaled(30, city)
    n_out.times do |i|
      OutboundMessage.find_or_create_by!(idempotency_key: "OUT-#{city[:code]}-#{format('%04d', i + 1)}") do |m|
        m.to = format("+55%s9%08d", city[:ddd], i + 1)
        m.status = ACK_STATUS_CYCLE[i % ACK_STATUS_CYCLE.length]
        m.template = { "name" => "rota_saude_ask", "language" => { "code" => "pt_BR" } }
        m.municipality_id = muni.id
        m.created_at = at_days_ago(spread_days(i, n_out), hour: (i % 12) + 8)
      end
    end
  end
```

- [ ] **Step 2: Wire into run!**

Update the city loop in `def run!`:

```ruby
      CITIES.each do |city|
        muni = upsert_municipality(city)
        build_protocols(city, muni)
        build_conversations_and_consents(city, muni)
        build_ingestion(city, muni)
      end
```

- [ ] **Step 3: Run and verify the ingestion panel**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails runner '
ApplicationRecord.connected_to(role: :admin) do
  m = Municipality.find_by(slug: "curitiba")
  p = Admin::Api::Period.parse(key: "30d", from: nil, to: nil, tz: ActiveSupport::TimeZone["America/Sao_Paulo"])
  out = Admin::IngestionQuery.call(municipality: m, period: p)
  ack = out[:ack].map { |a| [a[:code], a[:count]] }.to_h rescue out.inspect
  puts "inboundTotal=#{out[:inboundTotal]} ack=#{ack} purge=#{out[:purge].inspect rescue "n/a"}"
end'
```
Expected: `inboundTotal` ≥ 1; ack `ok`, `warn`, and `err` all ≥ 1; purge shows a non-zero oldest age / overTtl true.

- [ ] **Step 4: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb
git -C apps/api commit -m "Seed inbound and outbound messages for the ingestion panel"
```

---

### Task 5: Triages + reports (Triagem — Triagens, Classificação, Relatórios)

**Files:**
- Modify: `apps/api/db/seeds/dashboard_demo.rb`

**Interfaces:**
- Consumes: `build_conversations_and_consents` return (`[{convo:, state:, days:, seq:}]`), `build_protocols` return (`{name => PD}`), `at_days_ago`.
- Produces: `build_triages_and_reports(city, muni, citizens, protocols)` → returns an array of `{ triage:, tier:, mode:, priority:, days:, completed: }`; helper `upsert_report(muni, triage, tier, created_at:, expires_at:)`; constants `TIER_CYCLE`, `MODE_CYCLE`, `TRIAGE_STATUS_CYCLE`.

- [ ] **Step 1: Add the triage/report builders**

In `apps/api/db/seeds/dashboard_demo.rb`, add (before `def run!`):

```ruby
  TIER_CYCLE = %w[low medium high medium low high medium high].freeze
  MODE_CYCLE = %w[weighted weighted decision_table].freeze
  # Mostly completed, with a few in_progress and aborted for status variety.
  TRIAGE_STATUS_CYCLE = %w[completed completed completed completed in_progress completed aborted_by_timeout completed].freeze

  def build_triages_and_reports(city, muni, citizens, protocols)
    protos = protocols.values
    built = []
    citizens.each_with_index do |c, i|
      convo = c[:convo]
      days  = c[:days]
      proto = protos[i % protos.size]
      status = c[:state] == "completed" ? "completed" : TRIAGE_STATUS_CYCLE[i % TRIAGE_STATUS_CYCLE.size]
      completed = status == "completed"
      tier = TIER_CYCLE[i % TIER_CYCLE.size]
      mode = MODE_CYCLE[i % MODE_CYCLE.size]
      priority = tier == "high" ? 1 : 0
      started_at = at_days_ago(days, hour: 9)
      completed_at = completed ? started_at + (3 + (i % 8)).minutes : nil

      triage = Triage.where(conversation_id: convo.id, protocol_definition_id: proto.id).first
      triage ||= Triage.create!(
        conversation: convo, protocol_definition: proto, protocol_name: proto.name,
        municipality_id: muni.id, status: status,
        tier: (completed ? tier : nil), priority: (completed ? priority : nil),
        current_step: "febre",
        answers: { "tosse" => "true", "febre" => (tier == "high" ? "true" : "false") },
        created_at: started_at, completed_at: completed_at,
        outcome: (completed ? {
          "status" => "terminal", "tier" => tier, "priority" => priority,
          "scoring" => { "mode" => mode, "score" => (tier == "high" ? 8 : tier == "medium" ? 4 : 1) },
          "trail" => [ { "step" => "tosse", "answer" => "true" },
                       { "step" => "febre", "answer" => (tier == "high" ? "true" : "false") } ]
        } : {})
      )

      built << { triage: triage, tier: tier, mode: mode, priority: priority, days: days, completed: completed }

      if completed
        expired = (i % 4).zero?
        upsert_report(muni, triage, tier,
                      created_at: completed_at,
                      expires_at: (expired ? at_days_ago(days + 2, hour: 9) : Time.current + 20.days))
      end
    end
    built
  end

  # Build the report snapshot directly (not via GenerateReportJob) so created_at
  # and expires_at can be backdated for a live/expired mix. token/signature per
  # the model contract.
  def upsert_report(muni, triage, tier, created_at:, expires_at:)
    return if ReportSnapshot.where(triage_id: triage.id).exists?
    token = "RPT-#{muni.slug[0, 3].upcase}-#{triage.id.to_s[0, 8]}"
    ReportSnapshot.create!(
      triage: triage, protocol_definition: triage.protocol_definition, municipality_id: muni.id,
      outcome: { "tier" => tier, "priority" => triage.priority, "status" => "terminal" },
      payload: { "tier" => tier, "priority" => triage.priority, "completed_at" => triage.completed_at&.iso8601 },
      token: token, signature: ReportSnapshot.sign(token),
      created_at: created_at, expires_at: expires_at
    )
  end
```

- [ ] **Step 2: Wire into run!**

Update the city loop in `def run!`:

```ruby
      CITIES.each do |city|
        muni = upsert_municipality(city)
        protocols = build_protocols(city, muni)
        citizens = build_conversations_and_consents(city, muni)
        build_ingestion(city, muni)
        build_triages_and_reports(city, muni, citizens, protocols)
      end
```

- [ ] **Step 3: Run and verify triages, classification, reports**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails runner '
ApplicationRecord.connected_to(role: :admin) do
  m = Municipality.find_by(slug: "curitiba")
  p = Admin::Api::Period.parse(key: "30d", from: nil, to: nil, tz: ActiveSupport::TimeZone["America/Sao_Paulo"])
  tr = Admin::TriagesQuery.call(municipality: m, period: p)
  cl = Admin::ClassificationQuery.call(municipality: m, period: p)
  rp = Admin::ReportsQuery.call(municipality: m, period: p)
  puts "triages started=#{tr[:started] rescue tr.inspect} completed=#{tr[:completed] rescue "?"}"
  puts "tiers=#{cl[:tiers].map { |t| [t[:key], t[:count]] }.to_h} priorityTrue=#{cl[:priorityTrue]} byMode=#{cl[:byMode].map { |x| x[:mode] }.inspect}"
  live = rp[:reports].count { |r| r[:live] }; exp = rp[:reports].count { |r| !r[:live] }
  puts "reports total=#{rp[:reports].size} live=#{live} expired=#{exp} tiers=#{rp[:reports].map { |r| r[:tier] }.uniq.inspect}"
end'
```
Expected: triages `started`/`completed` ≥ 1; every tier (`low`, `medium`, `high`) count ≥ 1; `priorityTrue` ≥ 1; `byMode` includes both `weighted` and `decision_table`; reports has both `live` ≥ 1 and `expired` ≥ 1.

- [ ] **Step 4: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb
git -C apps/api commit -m "Seed triages with tier/priority/mode spread and report snapshots"
```

---

### Task 6: Domain events (Governança — Eventos/Protocolos four-eyes; Classificação trail)

**Files:**
- Modify: `apps/api/db/seeds/dashboard_demo.rb`

**Interfaces:**
- Consumes: `build_protocols` return, `build_conversations_and_consents` return, `build_triages_and_reports` return, `at_days_ago`, `ACTOR_A`, `ACTOR_B`.
- Produces: `build_events(city, muni, protocols, citizens, triages)`; helper `upsert_event(demo_id, name:, occurred_at:, municipality_id:, payload:)`.

- [ ] **Step 1: Add the events builders**

In `apps/api/db/seeds/dashboard_demo.rb`, add (before `def run!`):

```ruby
  # Idempotent by a synthetic payload.demo_id (domain_events has no natural key).
  def upsert_event(demo_id, name:, occurred_at:, municipality_id:, payload:)
    existing = DomainEvent.where("payload ->> 'demo_id' = ?", demo_id).first
    return existing if existing
    DomainEvent.create!(
      name: name, occurred_at: occurred_at, municipality_id: municipality_id,
      published_at: occurred_at, created_at: occurred_at,
      payload: payload.merge("demo_id" => demo_id)
    )
  end

  def build_events(city, muni, protocols, citizens, triages)
    code = city[:code]

    # --- Protocol audit events (Protocolos four-eyes + protocol_events) ---
    # protocols_query matches payload.protocol_definition_id (four_eyes) and
    # payload.name (protocol_events); reads actor + version.
    respv2 = ProtocolDefinition.find_by(name: "triage-respiratoria", version: 2, municipality_id: muni.id)
    dengue1 = protocols["triagem-dengue"]
    dengue2 = ProtocolDefinition.find_by(name: "triagem-dengue", version: 2, municipality_id: muni.id)

    # respiratoria v2: created by A, published by B → fourEyes = true (ok)
    upsert_event("#{code}-P-RESP2-C", name: "protocol.created", occurred_at: at_days_ago(20), municipality_id: muni.id,
                 payload: { "protocol_definition_id" => respv2.id, "name" => "triage-respiratoria", "version" => 2, "actor" => ACTOR_A })
    upsert_event("#{code}-P-RESP2-P", name: "protocol.published", occurred_at: at_days_ago(18), municipality_id: muni.id,
                 payload: { "protocol_definition_id" => respv2.id, "name" => "triage-respiratoria", "version" => 2, "actor" => ACTOR_B })
    # dengue v1: created + published by the SAME actor → fourEyes = false (collapsed)
    upsert_event("#{code}-P-DENG1-C", name: "protocol.created", occurred_at: at_days_ago(25), municipality_id: muni.id,
                 payload: { "protocol_definition_id" => dengue1.id, "name" => "triagem-dengue", "version" => 1, "actor" => ACTOR_A })
    upsert_event("#{code}-P-DENG1-P", name: "protocol.published", occurred_at: at_days_ago(24), municipality_id: muni.id,
                 payload: { "protocol_definition_id" => dengue1.id, "name" => "triagem-dengue", "version" => 1, "actor" => ACTOR_A })
    # dengue v2 retired
    upsert_event("#{code}-P-DENG2-R", name: "protocol.retired", occurred_at: at_days_ago(10), municipality_id: muni.id,
                 payload: { "protocol_definition_id" => dengue2.id, "name" => "triagem-dengue", "version" => 2, "actor" => ACTOR_B })

    # --- conversation.* and consent.* (Events filter prefixes) ---
    citizens.each_with_index do |c, i|
      convo = c[:convo]; days = c[:days]
      upsert_event("#{code}-CV-#{i}", name: "conversation.consented", occurred_at: at_days_ago(days, hour: 10),
                   municipality_id: muni.id, payload: { "conversation_id" => convo.id, "actor" => "sistema" })
      upsert_event("#{code}-CO-#{i}", name: "consent.given", occurred_at: at_days_ago(days, hour: 11),
                   municipality_id: muni.id, payload: { "conversation_id" => convo.id, "actor" => "cidadão" })
    end

    # --- Trail events (BARE names, per completed triage) + prefixed triage/priority ---
    triages.each_with_index do |t, i|
      next unless t[:completed]
      tri = t[:triage]; base = at_days_ago(t[:days], hour: 9)
      upsert_event("#{code}-T-SC-#{i}", name: "scored", occurred_at: base + 1.minute, municipality_id: muni.id,
                   payload: { "triage_id" => tri.id, "rule" => "weighted", "ref" => "mode:#{t[:mode]}", "out" => t[:tier], "actor" => "sistema" })
      upsert_event("#{code}-T-TA-#{i}", name: "tier_assigned", occurred_at: base + 2.minutes, municipality_id: muni.id,
                   payload: { "triage_id" => tri.id, "rule" => "threshold", "ref" => "tier", "out" => t[:tier], "actor" => "sistema" })
      upsert_event("#{code}-T-DONE-#{i}", name: "triage.completed", occurred_at: base + 3.minutes, municipality_id: muni.id,
                   payload: { "triage_id" => tri.id, "actor" => "sistema" })
      next unless t[:priority] == 1
      upsert_event("#{code}-T-PR-#{i}", name: "priority_rule", occurred_at: base + 2.minutes, municipality_id: muni.id,
                   payload: { "triage_id" => tri.id, "rule" => "escalate", "ref" => "priority", "out" => "1", "actor" => "sistema" })
      upsert_event("#{code}-PRI-#{i}", name: "priority.escalated", occurred_at: base + 3.minutes, municipality_id: muni.id,
                   payload: { "triage_id" => tri.id, "actor" => "sistema" })
    end
  end
```

- [ ] **Step 2: Wire into run!**

Update the city loop in `def run!`:

```ruby
      CITIES.each do |city|
        muni = upsert_municipality(city)
        protocols = build_protocols(city, muni)
        citizens = build_conversations_and_consents(city, muni)
        build_ingestion(city, muni)
        triages = build_triages_and_reports(city, muni, citizens, protocols)
        build_events(city, muni, protocols, citizens, triages)
      end
```

- [ ] **Step 3: Run and verify events, trail, four-eyes**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails runner '
ApplicationRecord.connected_to(role: :admin) do
  m = Municipality.find_by(slug: "curitiba")
  p = Admin::Api::Period.parse(key: "30d", from: nil, to: nil, tz: ActiveSupport::TimeZone["America/Sao_Paulo"])
  ev = Admin::EventsQuery.call(municipality: m, period: p, name: nil)
  names = ev[:byType].map { |x| x[:name] }
  prefixes = %w[triage. consent. conversation. protocol. priority.]
  covered = prefixes.select { |pre| names.any? { |n| n.start_with?(pre) } }
  puts "event types=#{names.size} prefixes_covered=#{covered.inspect} stream=#{ev[:stream].size}"
  # trail for one completed triage
  tri = Admin::Scoped.triages(m).where(status: "completed").first
  tr = Admin::TriageTrailQuery.call(municipality: m, triage_id: tri.id)
  puts "trail steps=#{tr[:steps].size} evs=#{tr[:steps].map { |s| s[:ev] }.uniq.inspect}"
  # four-eyes both true and false present
  pr = Admin::ProtocolsQuery.index(municipality: m)
  fe = pr[:list].map { |x| x[:fourEyes] }.uniq
  puts "fourEyes values=#{fe.inspect}"
end'
```
Expected: all five prefixes covered; `stream` ≥ 1; trail `steps` ≥ 2 (includes `scored`/`tier_assigned`); `fourEyes` includes both `true` and `false`.

- [ ] **Step 4: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb
git -C apps/api commit -m "Seed domain events for audit stream, trail, and four-eyes"
```

---

### Task 7: Verification rake task + idempotency gate

**Files:**
- Modify: `apps/api/lib/tasks/seed_demo.rake`

**Interfaces:**
- Consumes: `DashboardDemo` (loaded), all `Admin::*Query` classes.
- Produces: `db:seed:demo:verify` rake task that asserts every panel is non-empty for both cities and raises on failure.

- [ ] **Step 1: Add the verify task**

In `apps/api/lib/tasks/seed_demo.rake`, add a second task inside the `db:seed` namespace:

```ruby
    desc "Verify the dashboard demo dataset populates every panel (both cities)"
    task "demo:verify": :environment do
      tz = ActiveSupport::TimeZone["America/Sao_Paulo"]
      failures = []
      assert = ->(cond, msg) { failures << msg unless cond }

      ApplicationRecord.connected_to(role: :admin) do
        %w[curitiba londrina].each do |slug|
          m = Municipality.find_by(slug: slug)
          assert.call(m.present?, "#{slug}: municipality missing") or next
          p = Admin::Api::Period.parse(key: "30d", from: nil, to: nil, tz: tz)

          cv = Admin::ConversationsQuery.call(municipality: m, period: p)
          assert.call(cv[:funnel].sum { |f| f[:count] }.positive?, "#{slug}: conversations funnel empty")
          assert.call(cv[:live].to_i.positive?, "#{slug}: no live conversations")

          co = Admin::ConsentQuery.call(municipality: m, period: p)
          assert.call(co[:given].to_i.positive?, "#{slug}: no consents given")
          assert.call(co[:revoked].to_i.positive?, "#{slug}: no consents revoked")

          ig = Admin::IngestionQuery.call(municipality: m, period: p)
          assert.call(ig[:inboundTotal].to_i.positive?, "#{slug}: no inbound messages")
          assert.call(ig[:ack].sum { |a| a[:count] }.positive?, "#{slug}: ack breakdown empty")

          tr = Admin::TriagesQuery.call(municipality: m, period: p)
          assert.call(tr[:started].to_i.positive?, "#{slug}: no triages started")

          cl = Admin::ClassificationQuery.call(municipality: m, period: p)
          assert.call(cl[:tiers].all? { |t| t[:count].to_i.positive? }, "#{slug}: a tier bucket is empty")
          assert.call(cl[:priorityTrue].to_i.positive?, "#{slug}: no priority triages")
          assert.call(cl[:byMode].size >= 2, "#{slug}: <2 scoring modes")

          rp = Admin::ReportsQuery.call(municipality: m, period: p)
          assert.call(rp[:reports].any? { |r| r[:live] }, "#{slug}: no live reports")
          assert.call(rp[:reports].any? { |r| !r[:live] }, "#{slug}: no expired reports")

          pr = Admin::ProtocolsQuery.index(municipality: m)
          assert.call(pr[:list].size >= 5, "#{slug}: <5 protocol rows")

          # EventsQuery has no municipality param — it relies on RLS (SET LOCAL)
          # in real requests; under the admin (BYPASSRLS) connection it reads all
          # cities' events. That is fine for a non-empty prefix gate.
          ev = Admin::EventsQuery.call(name: nil, from: nil, to: nil, period: p)
          names = ev[:byType].map { |x| x[:name] }
          %w[triage. consent. conversation. protocol. priority.].each do |pre|
            assert.call(names.any? { |n| n.start_with?(pre) }, "#{slug}: no events for prefix #{pre}")
          end
        end
      end

      if failures.empty?
        puts "[dashboard_demo:verify] OK — all panels populated for both cities"
      else
        abort "[dashboard_demo:verify] FAILURES:\n- #{failures.join("\n- ")}"
      end
    end
```

- [ ] **Step 2: Run verify (expect green)**

Run:
```bash
docker exec api-dev bundle exec rails db:seed:demo
docker exec api-dev bundle exec rails db:seed:demo:verify
```
Expected: `[dashboard_demo:verify] OK — all panels populated for both cities` and exit status 0. If any assertion prints, fix the corresponding builder (Tasks 2–6) and re-run.

- [ ] **Step 3: Verify idempotency (double-run, stable counts)**

Run:
```bash
docker exec api-dev bundle exec rails runner '
ApplicationRecord.connected_to(role: :admin) do
  f = -> { { conv: Conversation.count, tri: Triage.count, con: Consent.count, inb: InboundMessage.count,
             out: OutboundMessage.count, rep: ReportSnapshot.count, ev: DomainEvent.count, pd: ProtocolDefinition.count } }
  before = f.call
  load Rails.root.join("db/seeds/dashboard_demo.rb")
  after = f.call
  puts "before=#{before}"
  puts "after =#{after}"
  abort "NOT IDEMPOTENT" unless before == after
  puts "IDEMPOTENT OK"
end'
```
Expected: `before` == `after`, prints `IDEMPOTENT OK`.

- [ ] **Step 4: Commit**

```bash
git -C apps/api add lib/tasks/seed_demo.rake
git -C apps/api commit -m "Add dashboard demo verification rake task"
```

---

## Wrap-up (after all tasks)

- `docker exec api-dev bundle exec rails db:seed:demo && docker exec api-dev bundle exec rails db:seed:demo:verify` — green.
- Idempotency double-run — stable counts.
- Optional: browse the dashboard (`http://localhost:5175/dashboard/`, login `admin@curitiba.demo` / `dev-password`) and spot-check panels.
- Sync is a separate explicit, user-authorized step (`apps/api`, FF-only, secret gate).

## Self-Review notes

- **Spec coverage:** Ingestão → Task 4; Conversas/Consentimentos → Task 3; Triagens/Classificação/Relatórios → Task 5; Protocolos/Editor → Task 2 (editor needs only a published/active protocol, provided); Eventos/trail/four-eyes → Task 6; opt-in loader + rake + guarded hook → Task 1; verification + idempotency → Task 7. Two cities handled by the `CITIES` loop in every builder.
- **Correctness traps encoded:** tier `low/medium/high` (Task 5); ack status 0–5 (Task 4); trail bare event names + prefixed events (Task 6); `priority` int `1/0` matching `where(priority: true)`→`= 1` (Task 5); reports built directly for live/expired mix (Task 5).
- **Type consistency:** builder names and return shapes match across Interfaces blocks — `build_conversations_and_consents` returns `[{convo:,state:,days:,seq:}]` consumed by `build_triages_and_reports`; that returns `[{triage:,tier:,mode:,priority:,days:,completed:}]` consumed by `build_events`; `build_protocols` returns `{name => PD}` consumed by both.
- **Placeholder scan:** none — every step has full code and exact commands.
- **YAGNI:** `ConsentTerm` dropped (no dashboard view reads it); Protocol Editor write-flow not seeded (read data suffices).
```
