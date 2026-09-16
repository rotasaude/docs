# Conversation Terminal States (F-02.2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Complete the conversation state machine: mark a conversation `completed` when its triage finishes, `declined` when consent is refused, and `cancelled` when the citizen explicitly stops mid-triage.

**Architecture:** Add `completed`/`declined`/`cancelled` to the `Conversation` state enum (code-only) and an `aborted_by_cancellation` triage status (migration). `ConversationAdvance` sets `completed` on a terminal outcome, `declined` on a consent refusal, and `cancelled` when `Consents.cancel?` (explicit stop words, NOT "não") matches mid-triage.

**Tech Stack:** Rails 8 (RSpec, run in the `api-dev` container).

## Global Constraints

- **Commits in English** (UI strings stay Portuguese). `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. A NEW MIGRATION must be applied to BOTH the dev and test DBs: `docker exec api-dev bundle exec rails db:migrate` and `docker exec api-dev env RAILS_ENV=test bundle exec rails db:migrate`.
- The `triages` table is admin-owned (ADR-0019); its `ck_triagens_status` check constraint must be altered via raw SQL as `rota_admin` — mirror `db/migrate/20260630194633_allow_aborted_by_timeout_triage_status.rb` (a private `as_admin` PG.connect helper).
- The new terminal conversation states are outside the partial unique index `idx_conversations_active_per_tenant_phone` (greeting/awaiting_consent/consented) — no conversation migration needed.
- Mid-triage cancel detection uses `sair/parar/cancelar/encerrar` only — it MUST NOT match `"não"` (a valid boolean triage answer).
- When committing, run `git -C apps/api commit` and CONFIRM with `git -C apps/api log --oneline -1` before reporting DONE. Commit `db/schema.rb` and the three `db/*_schema.rb` dumps that `db:migrate` regenerates.

---

### Task 1: `aborted_by_cancellation` + terminal conversation states

**Files:**
- Create: `apps/api/db/migrate/<timestamp>_allow_aborted_by_cancellation_triage_status.rb`
- Modify: `apps/api/app/models/triage.rb`, `apps/api/app/models/conversation.rb`
- Test: `apps/api/spec/models/conversation_terminal_states_spec.rb` (create)

**Interfaces:**
- Produces: `Triage` status enum includes `aborted_by_cancellation` (DB constraint allows it); `Conversation` state enum includes `completed`/`declined`/`cancelled` (`state_completed?`/`state_declined?`/`state_cancelled?`). Persisting those values is valid.

- [ ] **Step 1: Generate the migration**

Run: `docker exec api-dev bundle exec rails generate migration AllowAbortedByCancellationTriageStatus`
Replace its body with (mirror of the F-02.7 migration; keep the generator's version line):

```ruby
class AllowAbortedByCancellationTriageStatus < ActiveRecord::Migration[8.1]
  def up
    as_admin do |c|
      c.exec("ALTER TABLE triages DROP CONSTRAINT ck_triagens_status")
      c.exec("ALTER TABLE triages ADD CONSTRAINT ck_triagens_status CHECK (status IN ('in_progress','completed','aborted_by_revocation','aborted_by_timeout','aborted_by_cancellation'))")
    end
  end

  def down
    as_admin do |c|
      c.exec("ALTER TABLE triages DROP CONSTRAINT ck_triagens_status")
      c.exec("ALTER TABLE triages ADD CONSTRAINT ck_triagens_status CHECK (status IN ('in_progress','completed','aborted_by_revocation','aborted_by_timeout'))")
    end
  end

  private

  # DDL de ownership exige rota_admin. Mesmo padrão de
  # 20260630194633_allow_aborted_by_timeout_triage_status.
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

Create `apps/api/spec/models/conversation_terminal_states_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Conversation terminal states (F-02.2)", type: :model do
  let(:muni) { create(:municipality) }

  let(:definition_hash) do
    {
      "name" => "terminal-demo", "version" => 1, "start_step_id" => "s1",
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

  it "accepts the aborted_by_cancellation triage status" do
    pd = ProtocolDefinition.create!(name: "terminal-demo", version: 1, status: "active",
                                    municipality_id: muni.id, definition: definition_hash)
    convo = Conversation.create!(municipality_id: muni.id, phone: "+5511990000010", state: "consented")
    triage = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "terminal-demo",
                            municipality_id: muni.id, status: "in_progress")
    expect { triage.update!(status: :aborted_by_cancellation) }.not_to raise_error
    expect(triage.reload.status).to eq("aborted_by_cancellation")
  end

  it "accepts completed / declined / cancelled conversation states" do
    convo = Conversation.create!(municipality_id: muni.id, phone: "+5511990000011", state: "consented")
    expect { convo.update!(state: :completed) }.not_to raise_error
    expect(convo.reload.state_completed?).to be(true)
    convo.update!(state: :declined);  expect(convo.reload.state_declined?).to be(true)
    convo.update!(state: :cancelled); expect(convo.reload.state_cancelled?).to be(true)
  end
end
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/models/conversation_terminal_states_spec.rb`
Expected: FAIL — `ArgumentError: 'aborted_by_cancellation' is not a valid status` and `'completed'/'declined'/'cancelled' is not a valid state`.

- [ ] **Step 4: Apply the migration and update the enums**

Run: `docker exec api-dev bundle exec rails db:migrate` and `docker exec api-dev env RAILS_ENV=test bundle exec rails db:migrate`.

In `apps/api/app/models/triage.rb`, extend the status enum:

```ruby
  enum :status, {
    in_progress: "in_progress",
    completed: "completed",
    aborted_by_revocation: "aborted_by_revocation",
    aborted_by_timeout: "aborted_by_timeout",
    aborted_by_cancellation: "aborted_by_cancellation"
  }, prefix: true
```

In `apps/api/app/models/conversation.rb`, extend the state enum:

```ruby
  enum :state, {
    greeting:         "greeting",
    awaiting_consent: "awaiting_consent",
    consented:        "consented",
    revoked:          "revoked",
    abandoned:        "abandoned",
    completed:        "completed",
    declined:         "declined",
    cancelled:        "cancelled"
  }, prefix: true
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/models/conversation_terminal_states_spec.rb`
Expected: PASS (2 examples, 0 failures).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add db/migrate app/models/triage.rb app/models/conversation.rb db/schema.rb db/admin_schema.rb db/cache_schema.rb db/queue_schema.rb spec/models/conversation_terminal_states_spec.rb
git -C apps/api commit -m "Add completed/declined/cancelled conversation states and aborted_by_cancellation triage status"
git -C apps/api log --oneline -1
```

---

### Task 2: `Consents.cancel?`

**Files:**
- Modify: `apps/api/app/services/consents.rb`
- Test: `apps/api/spec/services/consents_spec.rb`

**Interfaces:**
- Produces: `Consents::CANCEL_PATTERNS`; `Consents.cancel?(text) -> Boolean` — `true` for `sair`/`parar`/`cancelar`/`encerrar`; `false` for `"não"`, `"sim"`, ordinary text, blank, nil.

- [ ] **Step 1: Write the failing test**

Append to `apps/api/spec/services/consents_spec.rb` (inside the top-level `describe Consents`):

```ruby
  describe ".cancel?" do
    %w[sair parar cancelar encerrar].each do |word|
      it "is true for #{word}" do
        expect(described_class.cancel?(word)).to be(true)
      end
    end

    it "is false for 'não' (a valid boolean answer)" do
      expect(described_class.cancel?("não")).to be(false)
    end

    it "is false for an ordinary answer / blank / nil" do
      expect(described_class.cancel?("sim")).to be(false)
      expect(described_class.cancel?("qualquer coisa")).to be(false)
      expect(described_class.cancel?("")).to be(false)
      expect(described_class.cancel?(nil)).to be(false)
    end
  end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/consents_spec.rb`
Expected: FAIL — `NoMethodError: undefined method 'cancel?' for Consents`.

- [ ] **Step 3: Implement**

In `apps/api/app/services/consents.rb`, add the patterns near the other pattern constants and the predicate (e.g. after `REVOKE_PATTERNS`):

```ruby
  # Palavras-parada explícitas no MEIO da triagem (F-02.2). NÃO inclui "não",
  # que é uma resposta booleana válida.
  CANCEL_PATTERNS = [
    /\A\s*(sair|parar|cancelar|encerrar)\s*\z/i
  ].freeze

  def self.cancel?(text)
    return false if text.nil?
    CANCEL_PATTERNS.any? { |re| text.match?(re) }
  end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/consents_spec.rb`
Expected: PASS (the new `.cancel?` examples plus the existing `.interpret` examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/consents.rb spec/services/consents_spec.rb
git -C apps/api commit -m "Add Consents.cancel? for explicit mid-triage stop intent"
git -C apps/api log --oneline -1
```

---

### Task 3: `ConversationAdvance` terminal transitions + i18n

**Files:**
- Modify: `apps/api/app/commands/conversation_advance.rb`, `apps/api/config/locales/conversation_advance.pt-BR.yml`
- Test: `apps/api/spec/commands/conversation_advance_spec.rb`

**Interfaces:**
- Consumes: `Conversation` `completed`/`declined`/`cancelled` (Task 1); `Triage` `aborted_by_cancellation` (Task 1); `Consents.cancel?` (Task 2).
- Produces: a terminal triage outcome → conversation `completed`; consent refusal at `awaiting_consent` → conversation `declined`; an explicit cancel mid-triage → conversation `cancelled` + the in-progress triage `aborted_by_cancellation`.

- [ ] **Step 1: Add the i18n strings**

In `apps/api/config/locales/conversation_advance.pt-BR.yml`, under `pt-BR: conversation_advance:` (next to `consent_revoked`), add:

```yaml
    consent_declined: "Tudo bem, não vamos seguir. Envie \"olá\" se mudar de ideia."
    triage_cancelled: "Triagem cancelada. Envie \"olá\" para recomeçar."
```

- [ ] **Step 2: Add the failing spec examples**

In `apps/api/spec/commands/conversation_advance_spec.rb`, reuse the existing harness (the `muni`/`conversation`/`inbound` lets and the tenant `around` block). Add these examples. (a) In the `describe "estado :awaiting_consent"` block, the existing `raw_body "não"` revoke example should now assert `declined`:

```ruby
    context "quando recusa o consentimento" do
      let(:raw_body) { "não" }

      it "move a conversa para declined e responde" do
        result = described_class.call(conversation: conversation, inbound: inbound)
        expect(result.reply.body).to eq(I18n.t("conversation_advance.consent_declined"))
        expect(conversation.reload.state).to eq("declined")
      end
    end
```

(b) In the `describe "estado :consented (fluxo com motor real)"` block (the one with the real protocol fixture and `raw_body "true"` that advances the triage), add a completion example and the cancel examples. Match that block's existing fixture/consent setup exactly (copy its `let`s):

```ruby
    context "quando a triagem completa" do
      let(:raw_body) { "true" } # único step boolean → terminal

      it "move a conversa para completed" do
        described_class.call(conversation: conversation, inbound: inbound)
        expect(conversation.reload.state).to eq("completed")
      end
    end

    context "quando o cidadão cancela (cancelar)" do
      let(:raw_body) { "cancelar" }

      it "move a conversa para cancelled, aborta a triage e responde" do
        triage = conversation.triages.where(status: :in_progress).first ||
                 begin described_class.call(conversation: conversation, inbound: inbound); conversation.triages.order(created_at: :desc).first end
        result = described_class.call(conversation: conversation, inbound: inbound)
        expect(result.reply.body).to eq(I18n.t("conversation_advance.triage_cancelled"))
        expect(conversation.reload.state).to eq("cancelled")
        expect(conversation.triages.where(status: "aborted_by_cancellation")).to be_present
      end
    end

    context "quando responde 'não' (resposta válida, não cancela)" do
      let(:raw_body) { "não" }

      it "não cancela; a conversa segue consented" do
        described_class.call(conversation: conversation, inbound: inbound)
        expect(conversation.reload.state).not_to eq("cancelled")
      end
    end
```

Note: align the cancel-test setup with how the existing consented block seeds an in-progress triage (it has a real protocol + consent). If that block starts the triage lazily on the first inbound, send one inbound to start it before the `"cancelar"` inbound — copy the block's existing pattern rather than guessing.

- [ ] **Step 3: Run to verify the new examples fail**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb`
Expected: FAIL — the declined/completed/cancelled examples fail (current code leaves the conversation `awaiting_consent`/`consented`).

- [ ] **Step 4: Implement the transitions**

In `apps/api/app/commands/conversation_advance.rb`:

(a) `handle_awaiting_consent` `:revoke` branch — replace it with:

```ruby
    when :revoke
      @conversation.update!(state: :declined)
      Result.new(reply: Messaging::Reply.text(t(:consent_declined)))
```

(b) `handle_consented` — add the cancel check at the top and switch the terminal return:

```ruby
  def handle_consented
    return cancel_and_finish if Consents.cancel?(text)

    triage = active_triage || begin_triage_or_nil
    return Result.new(reply: Messaging::Reply.text(t(:no_protocol))) unless triage

    result = CompleteTriage.call(triage: triage, answer: text)
    return Result.new(reply: Messaging::Reply.text(reason_text(result.reason))) if result.failure?

    outcome = result.payload[:outcome]
    return complete_and_finish if outcome.terminal?

    triage.reload
    Result.new(reply: step_reply(triage, outcome.awaiting, :triage_next))
  end
```

(c) Add the two private helpers (next to the other private methods):

```ruby
  def complete_and_finish
    @conversation.update!(state: :completed)
    Result.new(reply: nil)
  end

  def cancel_and_finish
    @conversation.triages.where(status: :in_progress).order(created_at: :desc).first&.update!(
      status: :aborted_by_cancellation, completed_at: Time.current
    )
    @conversation.update!(state: :cancelled)
    Result.new(reply: Messaging::Reply.text(t(:triage_cancelled)))
  end
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb`
Expected: PASS (all examples, including the new declined/completed/cancelled + the "não does not cancel" guard).

- [ ] **Step 6: Run the conversation/consent regression group**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb spec/services/consents_spec.rb spec/models/conversation_terminal_states_spec.rb`
Expected: PASS — flow, consent, and states all green.

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/commands/conversation_advance.rb config/locales/conversation_advance.pt-BR.yml spec/commands/conversation_advance_spec.rb
git -C apps/api commit -m "Transition conversations to completed/declined/cancelled terminal states"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec` — expect green.
- Move the board card (F-02.2) to Done, then Verified.
- Sync is a separate explicit step. This (with F-02.7) completes the conversation lifecycle.

## Self-Review notes

- **Spec coverage:** states + migration → Task 1; `Consents.cancel?` → Task 2; the three transitions (completed/declined/cancelled) + i18n → Task 3. All spec sections mapped.
- **Type consistency:** `Conversation` `completed`/`declined`/`cancelled` and `Triage` `aborted_by_cancellation` defined in Task 1 and used in Task 3; `Consents.cancel?` defined in Task 2 and called in Task 3's `handle_consented`; the i18n keys `consent_declined`/`triage_cancelled` (Task 3 step 1) are referenced by the helpers (Task 3 step 4).
- **`não` safety:** `CANCEL_PATTERNS` excludes `não` (Task 2), and Task 3's guard example asserts a `"não"` mid-triage does NOT cancel.
- **Migration:** the `aborted_by_cancellation` migration mirrors the F-02.7 `as_admin` raw-SQL pattern; the 5-value `up` / 4-value `down` matches the current (post-F-02.7) constraint baseline.
- **Conversation index:** the three new states are terminal and outside `idx_conversations_active_per_tenant_phone` — no conversation migration needed (verified in the spec).
