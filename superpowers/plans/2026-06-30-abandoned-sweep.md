# Timeout / Abandoned Sweep (F-02.7) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A recurring sweep marks idle, non-terminal conversations `abandoned` (and aborts their in-progress triage), completing the conversation lifecycle.

**Architecture:** Add an `abandoned` state to `Conversation` (code-only) and an `aborted_by_timeout` status to `Triage` (migration extends the `ck_triagens_status` check). A `SweepAbandonedConversationsJob` (cross-tenant, admin connection) bulk-marks stale non-terminal conversations that never completed a triage; scheduled in `config/recurring.yml`.

**Tech Stack:** Rails 8 (RSpec, run in the `api-dev` container), Solid Queue recurring tasks.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. Ruby edits live; a NEW MIGRATION must be applied with `docker exec api-dev bundle exec rails db:migrate` before its spec passes.
- The sweep is silent (no outbound). Idle = `conversation.updated_at` older than the cutoff; conversations that COMPLETED a triage or have a fresh in-progress triage are excluded. `idle_hours` default 24.
- The triages table is named `triages`; its status check constraint is named `ck_triagens_status` (legacy name kept after the `triagens`→`triages` rename).
- The job runs cross-tenant under the admin (BYPASSRLS) connection via `prepend AdminRoleJob` (NOT `include`).
- When committing, run `git -C apps/api commit` and CONFIRM with `git -C apps/api log --oneline -1` before reporting DONE.

---

### Task 1: `abandoned` / `aborted_by_timeout` states

**Files:**
- Create: `apps/api/db/migrate/<timestamp>_allow_aborted_by_timeout_triage_status.rb`
- Modify: `apps/api/app/models/triage.rb` (status enum), `apps/api/app/models/conversation.rb` (state enum)
- Test: `apps/api/spec/models/triage_status_spec.rb` (create)

**Interfaces:**
- Produces: `Triage` `status` enum includes `in_progress`/`completed`/`aborted_by_revocation`/`aborted_by_timeout` (matching the DB constraint); `Conversation` `state` enum includes `abandoned` (`state_abandoned?`). Persisting a triage with `status: "aborted_by_timeout"` and a conversation with `state: "abandoned"` is valid.

- [ ] **Step 1: Generate the migration file**

Run: `docker exec api-dev bundle exec rails generate migration AllowAbortedByTimeoutTriageStatus`
This creates `db/migrate/<timestamp>_allow_aborted_by_timeout_triage_status.rb`. Replace its body with:

```ruby
class AllowAbortedByTimeoutTriageStatus < ActiveRecord::Migration[8.1]
  def up
    remove_check_constraint :triages, name: "ck_triagens_status"
    add_check_constraint :triages,
      "status IN ('in_progress','completed','aborted_by_revocation','aborted_by_timeout')",
      name: "ck_triagens_status"
  end

  def down
    remove_check_constraint :triages, name: "ck_triagens_status"
    add_check_constraint :triages,
      "status IN ('in_progress','completed','aborted_by_revocation')",
      name: "ck_triagens_status"
  end
end
```

(If `ActiveRecord::Migration[8.1]` does not match the version the generator emitted, keep the generator's version line.)

- [ ] **Step 2: Write the failing test**

Create `apps/api/spec/models/triage_status_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Triage/Conversation terminal states", type: :model do
  let(:muni) { create(:municipality) }

  let(:definition_hash) do
    {
      "name" => "sweep-demo", "version" => 1, "start_step_id" => "s1",
      "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                    "branches" => { "true" => nil, "false" => nil } }],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 } }
    }
  end

  around do |ex|
    ApplicationRecord.transaction do
      Current.municipality_id = muni.id
      ApplicationRecord.connection.execute(
        ApplicationRecord.sanitize_sql(["SET LOCAL app.municipality_id = ?", muni.id])
      )
      ex.run
      raise ActiveRecord::Rollback
    end
  end

  after { Current.reset }

  it "accepts the aborted_by_timeout triage status" do
    pd = ProtocolDefinition.create!(name: "sweep-demo", version: 1, status: "active",
                                    municipality_id: muni.id, definition: definition_hash)
    convo = Conversation.create!(municipality_id: muni.id, phone: "+5511990000001", state: "consented")
    triage = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "sweep-demo",
                            municipality_id: muni.id, status: "in_progress")
    expect { triage.update!(status: :aborted_by_timeout) }.not_to raise_error
    expect(triage.reload.status).to eq("aborted_by_timeout")
  end

  it "accepts the abandoned conversation state" do
    convo = Conversation.create!(municipality_id: muni.id, phone: "+5511990000002", state: "consented")
    expect { convo.update!(state: :abandoned) }.not_to raise_error
    expect(convo.reload.state_abandoned?).to be(true)
  end
end
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/models/triage_status_spec.rb`
Expected: FAIL — `ArgumentError: 'aborted_by_timeout' is not a valid status` (enum) and/or a check-constraint violation, plus `'abandoned' is not a valid state`.

- [ ] **Step 4: Apply the migration and update the enums**

Run the migration: `docker exec api-dev bundle exec rails db:migrate`

In `apps/api/app/models/triage.rb`, replace the status enum line with the full set (this also closes a latent gap: `RevokeConsent` sets `:aborted_by_revocation`, which the 2-value enum would have rejected):

```ruby
  enum :status, {
    in_progress: "in_progress",
    completed: "completed",
    aborted_by_revocation: "aborted_by_revocation",
    aborted_by_timeout: "aborted_by_timeout"
  }, prefix: true
```

In `apps/api/app/models/conversation.rb`, add `abandoned` to the state enum:

```ruby
  enum :state, {
    greeting:         "greeting",
    awaiting_consent: "awaiting_consent",
    consented:        "consented",
    revoked:          "revoked",
    abandoned:        "abandoned"
  }, prefix: true
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/models/triage_status_spec.rb`
Expected: PASS (2 examples, 0 failures).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add db/migrate app/models/triage.rb app/models/conversation.rb db/schema.rb spec/models/triage_status_spec.rb
git -C apps/api commit -m "Add abandoned conversation state and aborted_by_timeout triage status"
git -C apps/api log --oneline -1
```

---

### Task 2: `SweepAbandonedConversationsJob` + schedule

**Files:**
- Create: `apps/api/app/jobs/sweep_abandoned_conversations_job.rb`
- Modify: `apps/api/config/recurring.yml`
- Test: `apps/api/spec/jobs/sweep_abandoned_conversations_job_spec.rb` (create)

**Interfaces:**
- Consumes: the `abandoned`/`aborted_by_timeout` states (Task 1); `AdminRoleJob` (`prepend`); `Conversation`/`Triage`.
- Produces: `SweepAbandonedConversationsJob.new.perform(idle_hours: 24)` marks each stale, non-terminal, never-completed conversation `abandoned` and aborts its in-progress triage (`aborted_by_timeout`); logs the count.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/jobs/sweep_abandoned_conversations_job_spec.rb`:

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe SweepAbandonedConversationsJob, type: :job do
  self.use_transactional_tests = false

  before { clean_admin_tables }
  after  { clean_admin_tables }

  def definition_hash
    {
      "name" => "sweep-demo", "version" => 1, "start_step_id" => "s1",
      "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                    "branches" => { "true" => nil, "false" => nil } }],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 } }
    }
  end

  # Builds fixtures via the real admin (BYPASSRLS) connection.
  def setup_fixtures
    as_admin do
      muni = Municipality.create!(name: "Sweep City", slug: "sweep-city", ibge_code: "3500010")
      pd = ProtocolDefinition.create!(name: "sweep-demo", version: 1, status: "active",
                                      municipality_id: muni.id, definition: definition_hash)
      yield(muni, pd)
    end
  end

  def make_convo(muni, phone:, state:, updated_at:)
    c = Conversation.create!(municipality_id: muni.id, phone: phone, state: state)
    c.update_columns(updated_at: updated_at)
    c
  end

  def make_triage(muni, pd, convo, status:, updated_at:)
    t = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "sweep-demo",
                       municipality_id: muni.id, status: status)
    t.update_columns(updated_at: updated_at)
    t
  end

  it "abandons an idle awaiting_consent conversation with no triage" do
    convo = nil
    setup_fixtures { |muni, _pd| convo = make_convo(muni, phone: "+551100", state: "awaiting_consent", updated_at: 30.hours.ago) }
    described_class.new.perform(idle_hours: 24)
    expect(as_admin { Conversation.find(convo.id).state }).to eq("abandoned")
  end

  it "abandons a consented conversation and aborts its stale in-progress triage" do
    convo = triage = nil
    setup_fixtures do |muni, pd|
      convo = make_convo(muni, phone: "+551101", state: "consented", updated_at: 30.hours.ago)
      triage = make_triage(muni, pd, convo, status: "in_progress", updated_at: 30.hours.ago)
    end
    described_class.new.perform(idle_hours: 24)
    expect(as_admin { Conversation.find(convo.id).state }).to eq("abandoned")
    expect(as_admin { Triage.find(triage.id).status }).to eq("aborted_by_timeout")
  end

  it "leaves a recent conversation untouched" do
    convo = nil
    setup_fixtures { |muni, _pd| convo = make_convo(muni, phone: "+551102", state: "awaiting_consent", updated_at: 1.hour.ago) }
    described_class.new.perform(idle_hours: 24)
    expect(as_admin { Conversation.find(convo.id).state }).to eq("awaiting_consent")
  end

  it "leaves a conversation that completed a triage untouched" do
    convo = nil
    setup_fixtures do |muni, pd|
      convo = make_convo(muni, phone: "+551103", state: "consented", updated_at: 30.hours.ago)
      make_triage(muni, pd, convo, status: "completed", updated_at: 30.hours.ago)
    end
    described_class.new.perform(idle_hours: 24)
    expect(as_admin { Conversation.find(convo.id).state }).to eq("consented")
  end

  it "leaves a conversation with a fresh in-progress triage untouched" do
    convo = nil
    setup_fixtures do |muni, pd|
      convo = make_convo(muni, phone: "+551104", state: "consented", updated_at: 30.hours.ago)
      make_triage(muni, pd, convo, status: "in_progress", updated_at: 1.hour.ago)
    end
    described_class.new.perform(idle_hours: 24)
    expect(as_admin { Conversation.find(convo.id).state }).to eq("consented")
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/jobs/sweep_abandoned_conversations_job_spec.rb`
Expected: FAIL — `uninitialized constant SweepAbandonedConversationsJob`.

- [ ] **Step 3: Implement the job**

Create `apps/api/app/jobs/sweep_abandoned_conversations_job.rb`:

```ruby
# Varredura recorrente (ADR-0019): marca conversas ociosas não-terminais como
# `abandoned` e aborta a triage in_progress (`aborted_by_timeout`). Cross-tenant
# sob a conexão admin (BYPASSRLS) — ver AdminRoleJob. Silenciosa (sem outbound).
class SweepAbandonedConversationsJob < ApplicationJob
  prepend AdminRoleJob
  queue_as :housekeeping

  NON_TERMINAL = %w[greeting awaiting_consent consented].freeze

  def perform(idle_hours: 24)
    cutoff = idle_hours.hours.ago

    fresh_triage_ids = Triage.where(status: :in_progress).where("updated_at >= ?", cutoff).select(:conversation_id)
    completed_ids    = Triage.where(status: :completed).select(:conversation_id)

    scope = Conversation
              .where(state: NON_TERMINAL)
              .where("updated_at < ?", cutoff)
              .where.not(id: fresh_triage_ids)
              .where.not(id: completed_ids)

    abandoned = 0
    scope.find_each do |conversation|
      conversation.triages.where(status: :in_progress)
                  .update_all(status: "aborted_by_timeout", updated_at: Time.current)
      conversation.update_columns(state: "abandoned", updated_at: Time.current)
      abandoned += 1
    end

    Rails.logger.info("[sweep_abandoned] abandoned=#{abandoned} idle_hours=#{idle_hours} cutoff=#{cutoff.iso8601}")
  end
end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/jobs/sweep_abandoned_conversations_job_spec.rb`
Expected: PASS (5 examples, 0 failures).

- [ ] **Step 5: Add the recurring schedule**

In `apps/api/config/recurring.yml`, inside the `default: &default` block (next to the other housekeeping entries), add:

```yaml
  sweep_abandoned:
    class: SweepAbandonedConversationsJob
    queue: housekeeping
    schedule: "every day at 2am America/Sao_Paulo"
    args: { idle_hours: 24 }
```

- [ ] **Step 6: Run the regression group**

Run: `docker exec api-dev bundle exec rspec spec/jobs/sweep_abandoned_conversations_job_spec.rb spec/models/triage_status_spec.rb spec/commands/conversation_advance_spec.rb`
Expected: PASS — the sweep, the states, and the conversation flow all green.

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/jobs/sweep_abandoned_conversations_job.rb config/recurring.yml spec/jobs/sweep_abandoned_conversations_job_spec.rb
git -C apps/api commit -m "Add recurring sweep that abandons idle conversations"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec` — expect green.
- Move the board card (F-02.7) to Done, then Verified.
- Sync is a separate explicit step.

## Self-Review notes

- **Spec coverage:** `abandoned`/`aborted_by_timeout` states + migration → Task 1; the sweep job (idle detection, exclusions for completed/fresh triages, abort + mark, log) + recurring schedule → Task 2. All spec sections mapped.
- **Type consistency:** `Conversation` `abandoned` / `Triage` `aborted_by_timeout` defined in Task 1 and consumed by the job in Task 2; the job's exclusion subqueries use `Triage.where(status: :in_progress|:completed)` (enum symbols valid after Task 1).
- **Enum vs constraint:** the job writes the new statuses via `update_all`/`update_columns` (raw strings, bypassing the enum cast) — safe even before the enum lists them — but Task 1 still adds them to the enums so reads (`triage.status`, `conversation.state_abandoned?`) and the model spec are clean, and the latent `RevokeConsent` `:aborted_by_revocation` gap is closed.
- **RLS:** the job is cross-tenant under `AdminRoleJob`; the spec creates fixtures via `as_admin` with `use_transactional_tests = false` + `clean_admin_tables` (the established admin request-spec pattern), so the admin-connection sweep sees them.
- **Placeholder scan:** complete code + exact commands; migration generated then filled, applied with `db:migrate` before its spec.
