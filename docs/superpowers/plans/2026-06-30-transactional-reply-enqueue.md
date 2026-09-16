# Transactional Reply Enqueue + Idempotent Inbound Processing (F-02.8) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `ProcessInboundMessageJob` advance the conversation, enqueue the outbound reply, and mark the inbound processed as one atomic transaction, so a committed state change always has its reply enqueued exactly once and a retry is a safe no-op.

**Architecture:** Add an `inbound_messages.processed_at` marker (admin-owned table → raw-SQL `as_admin` migration). Make `SendWhatsappJob` opt out of the global `enqueue_after_transaction_commit = :always` (per-job `= false`) so its enqueue joins the caller's open transaction. Rewrite `ProcessInboundMessageJob#perform` so the state change, the `SendWhatsappJob` row, and `processed_at` all commit inside the single `conversation.with_lock` transaction, with a `processed_at?` guard for idempotency.

**Tech Stack:** Rails 8.1 (RSpec, Solid Queue, run in the `api-dev` container).

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`. A NEW MIGRATION must be applied to BOTH DBs: `docker exec api-dev bundle exec rails db:migrate` and `docker exec api-dev env RAILS_ENV=test bundle exec rails db:migrate`.
- `inbound_messages` is owned by `rota_admin` (ADR-0019). DDL on it goes through raw SQL as `rota_admin` via a private `as_admin` PG.connect helper — mirror `db/migrate/20260630215915_allow_aborted_by_cancellation_triage_status.rb`.
- The reply enqueue must be **atomic** with the conversation state change; the inbound must be processed **idempotently** (a re-run after commit is a no-op).
- `SendWhatsappJob` already idempotent via UNIQUE `idempotency_key` — do NOT change its send/idempotency/session-window logic.
- Commit `db/schema.rb` and the three `db/*_schema.rb` dumps that `db:migrate` regenerates.

---

### Task 1: `inbound_messages.processed_at` marker

**Files:**
- Create: `apps/api/db/migrate/<timestamp>_add_processed_at_to_inbound_messages.rb`
- Test: `apps/api/spec/models/inbound_message_processed_at_spec.rb` (create)

**Interfaces:**
- Produces: `InboundMessage#processed_at` (nullable `timestamptz`) and `#processed_at?`; persisting/reading it is valid. A partial index `idx_inbound_messages_unprocessed` on `(created_at) WHERE processed_at IS NULL` exists.

- [ ] **Step 1: Generate the migration**

Run: `docker exec api-dev bundle exec rails generate migration AddProcessedAtToInboundMessages`
Replace its body with (keep the generator's `ActiveRecord::Migration[8.1]` version line):

```ruby
class AddProcessedAtToInboundMessages < ActiveRecord::Migration[8.1]
  def up
    as_admin do |c|
      c.exec("ALTER TABLE inbound_messages ADD COLUMN processed_at timestamptz")
      c.exec("CREATE INDEX idx_inbound_messages_unprocessed ON inbound_messages (created_at) WHERE processed_at IS NULL")
    end
  end

  def down
    as_admin do |c|
      c.exec("DROP INDEX IF EXISTS idx_inbound_messages_unprocessed")
      c.exec("ALTER TABLE inbound_messages DROP COLUMN processed_at")
    end
  end

  private

  # inbound_messages é owned por rota_admin; DDL exige esse papel.
  # Mesmo padrão de 20260630215915_allow_aborted_by_cancellation_triage_status.
  def as_admin
    require "pg"

    conn = PG.connect(
      host: ENV.fetch("DATABASE_HOST", "127.0.0.1"),
      port: ENV.fetch("DATABASE_PORT", 5432),
      dbname: connection.current_database,
      user: "rota_admin",
      password: ENV.fetch("ROTA_ADMIN_PASSWORD", "rota_admin")
    )

    yield conn
  ensure
    conn&.close
  end
end
```

- [ ] **Step 2: Write the failing test**

Create `apps/api/spec/models/inbound_message_processed_at_spec.rb`. This touches an RLS tenant table, so use the admin-connection harness (mirrors `spec/jobs/sweep_abandoned_conversations_job_spec.rb`):

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe "InboundMessage#processed_at (F-02.8)", type: :model do
  self.use_transactional_tests = false

  before { clean_admin_tables }
  after  { clean_admin_tables }

  it "round-trips a processed_at timestamp" do
    id = nil
    as_admin do
      muni = Municipality.create!(name: "Proc City", slug: "proc-city", ibge_code: "3500020")
      msg = InboundMessage.create!(
        message_id: "wamid.proc1", from: "+551100", kind: "text",
        raw: { "type" => "text", "text" => { "body" => "oi" } }.to_json,
        municipality_id: muni.id
      )
      id = msg.id
      expect(msg.processed_at).to be_nil
      expect(msg.processed_at?).to be(false)
    end

    as_admin do
      msg = InboundMessage.find(id)
      msg.update!(processed_at: Time.current)
      expect(InboundMessage.find(id).processed_at?).to be(true)
    end
  end
end
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/models/inbound_message_processed_at_spec.rb`
Expected: FAIL — `unknown attribute 'processed_at'` (column not yet added).

- [ ] **Step 4: Apply the migration**

Run: `docker exec api-dev bundle exec rails db:migrate` then `docker exec api-dev env RAILS_ENV=test bundle exec rails db:migrate`. No model change is needed — `processed_at` is a plain column Active Record picks up automatically.

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/models/inbound_message_processed_at_spec.rb`
Expected: PASS (1 example, 0 failures).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add db/migrate app/models/inbound_message.rb db/schema.rb db/admin_schema.rb db/cache_schema.rb db/queue_schema.rb spec/models/inbound_message_processed_at_spec.rb
git -C apps/api commit -m "Add processed_at marker to inbound_messages"
git -C apps/api log --oneline -1
```
(`app/models/inbound_message.rb` is listed only in case `git add` needs it; it is unchanged, so `git add db/migrate ...` without it is fine. Then run `git -C apps/api status --short` — if any schema dump was left unstaged, amend it in.)

---

### Task 2: Atomic, idempotent inbound processing

**Files:**
- Modify: `apps/api/app/jobs/send_whatsapp_job.rb` (add the per-job config override)
- Modify: `apps/api/app/jobs/process_inbound_message_job.rb` (the atomic unit)
- Test: `apps/api/spec/jobs/process_inbound_message_job_spec.rb` (create)

**Interfaces:**
- Consumes: `InboundMessage#processed_at?` / `#processed_at` (Task 1).
- Produces: `SendWhatsappJob.enqueue_after_transaction_commit == false`; `ProcessInboundMessageJob#perform(inbound_message_id, municipality_id:)` that commits the state change, the reply enqueue, and `processed_at` atomically and skips an already-processed inbound.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/jobs/process_inbound_message_job_spec.rb`. Job touches RLS tenant tables and `with_tenant` opens a real transaction, so use the admin-connection harness + the ActiveJob test helper:

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe ProcessInboundMessageJob, type: :job do
  include ActiveJob::TestHelper
  self.use_transactional_tests = false

  before { clean_admin_tables; clear_enqueued_jobs }
  after  { clean_admin_tables }

  # Cria muni + conversa (estado dado) + inbound (corpo dado) via conexão admin.
  # Retorna [muni_id, conversation_id, inbound_id].
  def seed(state:, body:, message_id:)
    ids = nil
    as_admin do
      muni = Municipality.create!(name: "PIMJ City", slug: "pimj-#{message_id}", ibge_code: "3500030")
      convo = Conversation.create!(municipality_id: muni.id, phone: "+5511990001", state: state)
      inbound = InboundMessage.create!(
        message_id: message_id, from: "+5511990001", kind: "text",
        raw: { "type" => "text", "text" => { "body" => body } }.to_json,
        municipality_id: muni.id
      )
      ids = [muni.id, convo.id, inbound.id]
    end
    ids
  end

  it "advances the conversation, enqueues the reply, and marks processed_at — atomically" do
    muni_id, convo_id, inbound_id = seed(state: "greeting", body: "oi", message_id: "wamid.h1")

    expect {
      described_class.new.perform(inbound_id, municipality_id: muni_id)
    }.to have_enqueued_job(SendWhatsappJob).exactly(:once)

    expect(as_admin { Conversation.find(convo_id).state }).to eq("awaiting_consent")
    expect(as_admin { InboundMessage.find(inbound_id).processed_at }).to be_present
  end

  it "is idempotent: a second run does not re-advance or re-enqueue" do
    muni_id, convo_id, inbound_id = seed(state: "greeting", body: "oi", message_id: "wamid.i1")

    described_class.new.perform(inbound_id, municipality_id: muni_id)
    expect(ConversationAdvance).not_to receive(:call)

    expect {
      described_class.new.perform(inbound_id, municipality_id: muni_id)
    }.not_to have_enqueued_job(SendWhatsappJob)

    expect(as_admin { Conversation.find(convo_id).state }).to eq("awaiting_consent")
  end

  it "does not enqueue or mark processed when ConversationAdvance raises (rollback)" do
    muni_id, convo_id, inbound_id = seed(state: "greeting", body: "oi", message_id: "wamid.r1")
    allow(ConversationAdvance).to receive(:call).and_raise(RuntimeError, "boom")

    expect {
      described_class.new.perform(inbound_id, municipality_id: muni_id)
    }.to raise_error(RuntimeError)

    expect(SendWhatsappJob).not_to have_been_enqueued
    expect(as_admin { InboundMessage.find(inbound_id).processed_at }).to be_nil
    expect(as_admin { Conversation.find(convo_id).state }).to eq("greeting")
  end

  it "marks processed and enqueues nothing when there is no reply" do
    muni_id, convo_id, inbound_id = seed(state: "revoked", body: "oi", message_id: "wamid.n1")

    expect {
      described_class.new.perform(inbound_id, municipality_id: muni_id)
    }.not_to have_enqueued_job(SendWhatsappJob)

    expect(as_admin { InboundMessage.find(inbound_id).processed_at }).to be_present
  end

  it "SendWhatsappJob enqueues within the transaction (no after-commit deferral)" do
    expect(SendWhatsappJob.enqueue_after_transaction_commit).to be(false)
  end
end
```

NOTE on matchers: `have_enqueued_job` / `have_been_enqueued` come from `ActiveJob::TestHelper` with the `:test` adapter (the test env's adapter). If a specific matcher form misbehaves under `use_transactional_tests = false`, keep the **behavioral** assertions (enqueue count via `enqueued_jobs.select { |j| j[:job] == SendWhatsappJob }.size`, `processed_at`, conversation state) — those are the requirement; adapt the matcher spelling, not the asserted behavior. Do not weaken an assertion to make it pass.

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/jobs/process_inbound_message_job_spec.rb`
Expected: FAIL — the config example fails (`enqueue_after_transaction_commit` is still the global `:always`/truthy), and the idempotency/rollback examples fail (no `processed_at` guard yet, so a second run re-advances and re-enqueues).

- [ ] **Step 3: Add the per-job config override to `SendWhatsappJob`**

In `apps/api/app/jobs/send_whatsapp_job.rb`, immediately after `include TenantScopedJob`, add:

```ruby
  # Enfileira DENTRO da transação aberta pelo caller (o with_lock do
  # ProcessInboundMessageJob), em vez de adiar para after_commit (config global
  # :always). Seguro: os args são primitivos (sem registro AR não-commitado) e o
  # Solid Queue é o mesmo Postgres, então a linha do job comita atomicamente com
  # o avanço de estado. Já é idempotente via idempotency_key. (F-02.8)
  self.enqueue_after_transaction_commit = false
```

- [ ] **Step 4: Rewrite `ProcessInboundMessageJob#perform`**

Replace the body of `apps/api/app/jobs/process_inbound_message_job.rb`'s `perform` with the atomic, idempotent unit:

```ruby
  def perform(inbound_message_id, municipality_id:)
    with_tenant(municipality_id) do
      inbound = InboundMessage.find(inbound_message_id)
      conversation = Conversation.for(inbound.from, municipality_id: municipality_id)
      conversation.with_lock do
        next if inbound.reload.processed_at?

        result = ConversationAdvance.call(conversation: conversation, inbound: inbound)
        if result&.reply
          SendWhatsappJob.perform_later(
            to: inbound.from, message: result.reply.to_h, municipality_id: municipality_id
          )
        end
        inbound.update!(processed_at: Time.current)
      end
    end
  end
```

(The previous `defined?(ConversationAdvance)` guard is dropped — the class is always loaded. The keep-the-comment header at the top of the file stays.)

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/jobs/process_inbound_message_job_spec.rb`
Expected: PASS (5 examples, 0 failures).

- [ ] **Step 6: Run the related job + send specs (regression)**

Run: `docker exec api-dev bundle exec rspec spec/jobs/process_inbound_message_job_spec.rb spec/jobs/send_whatsapp_job_spec.rb spec/commands/conversation_advance_spec.rb`
Expected: all green (the `SendWhatsappJob` override must not break its own spec or the conversation flow).

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/jobs/send_whatsapp_job.rb app/jobs/process_inbound_message_job.rb spec/jobs/process_inbound_message_job_spec.rb
git -C apps/api commit -m "Enqueue reply transactionally and process inbound idempotently"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec` — expect green.
- Move the board card (F-02.8) to Done, then Verified.
- Sync is a separate explicit step (user-authorized).

## Known limitation (documented, not a gap)

The "enqueue rolls back with the transaction" property is guaranteed in production by Solid Queue being DB-backed plus the `= false` override, but it is **not** unit-testable under the `:test` ActiveJob adapter (its `enqueued_jobs` is an in-memory array not tied to the AR transaction). The rollback test therefore stubs `ConversationAdvance` to raise *before* the enqueue is reached — proving no enqueue and no `processed_at` on a failed advance — rather than asserting an in-transaction enqueue is reverted. A full DB-adapter integration test is out of scope (YAGNI).

## Self-Review notes

- **Spec coverage:** migration + `processed_at` → Task 1; per-job `enqueue_after_transaction_commit = false` → Task 2 Step 3; atomic `perform` with `processed_at?` guard → Task 2 Step 4; the atomicity matrix (happy / idempotent / rollback / no-reply) → Task 2 spec examples. All spec sections mapped.
- **Type consistency:** `processed_at`/`processed_at?` defined in Task 1, consumed in Task 2's `perform` and spec; `SendWhatsappJob.enqueue_after_transaction_commit` set in Task 2 Step 3 and asserted in Task 2 Step 1.
- **Migration:** mirrors the verified `as_admin` raw-SQL pattern for the admin-owned `inbound_messages`; reversible `down` drops index then column.
- **Test honesty:** the rollback example's limitation under the `:test` adapter is documented above and the spec stubs the raise before the enqueue, so the assertion is real (not adapter-fooled).
