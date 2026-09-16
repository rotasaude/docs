# Consent Revocation Handling (F-07.15) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a consented citizen revoke consent mid-triage (LGPD, distinct from cancel), confirming inline, and handle the resulting `consent.revoked` event by anonymizing the aborted triage's clinical data and recording a revocation metric.

**Architecture:** A dedicated `Consents.revoke_intent?` detector triggers `RevokeConsent` from `ConversationAdvance#handle_consented` (which already does the synchronous state change + publishes `consent.revoked`); the confirmation is an inline reply. Two `IdempotentConsumer` jobs subscribe to `consent.revoked`: `AnonymizeRevokedTriageJob` (scrubs clinical fields, keeps the audit shell) and `RecordConsentRevocationJob` (bumps a `consents_revoked` dashboard metric).

**Tech Stack:** Rails 8.1 (RSpec, Solid Queue, run in the `api-dev` container).

## Global Constraints

- **Commits in English** (UI/i18n strings stay Portuguese). `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- No migration (reuses existing columns/tables).
- Revocation intent (`revogar`/`revogar consentimento`/`apagar meus dados`) must be distinct from `Consents.cancel?` (sair/parar/cancelar/encerrar) and from `não` (a valid triage answer). Checked BEFORE `cancel?` in `handle_consented`.
- Anonymization scrubs only the `aborted_by_revocation` triage of the conversation; append-only audit (consent row, event, triage status/timestamps/protocol_name) is kept — never delete rows.
- Subscribers use `include IdempotentConsumer` + `queue_as` + `handle(**payload)`; no HTTP/email (out of scope).
- The registry line `to: [AnonymizeRevokedTriageJob, RecordConsentRevocationJob]` references both job classes, so both must exist before the registry change lands (Task 3 does both + the wiring together).

---

### Task 1: `Consents.revoke_intent?`

**Files:**
- Modify: `apps/api/app/services/consents.rb`
- Test: `apps/api/spec/services/consents_spec.rb`

**Interfaces:**
- Produces: `Consents::REVOKE_INTENT_PATTERNS`; `Consents.revoke_intent?(text) -> Boolean` — `true` for `revogar` / `revogar consentimento` / `revogar meu consentimento` / `apagar meus dados`; `false` for `cancelar`/`sair`/`parar`/`encerrar`, `não`, `sim`, ordinary text, blank, nil.

- [ ] **Step 1: Write the failing test**

Append inside the top-level `describe Consents` block in `apps/api/spec/services/consents_spec.rb`:

```ruby
  describe ".revoke_intent?" do
    ["revogar", "revogar consentimento", "revogar meu consentimento", "apagar meus dados"].each do |phrase|
      it "is true for #{phrase.inspect}" do
        expect(described_class.revoke_intent?(phrase)).to be(true)
      end
    end

    it "is false for cancel words (those are cancel?, not revoke)" do
      %w[cancelar sair parar encerrar].each do |w|
        expect(described_class.revoke_intent?(w)).to be(false)
      end
    end

    it "is false for 'não'/'sim'/ordinary/blank/nil" do
      expect(described_class.revoke_intent?("não")).to be(false)
      expect(described_class.revoke_intent?("sim")).to be(false)
      expect(described_class.revoke_intent?("qualquer coisa")).to be(false)
      expect(described_class.revoke_intent?("")).to be(false)
      expect(described_class.revoke_intent?(nil)).to be(false)
    end
  end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/consents_spec.rb`
Expected: FAIL — `NoMethodError: undefined method 'revoke_intent?'`.

- [ ] **Step 3: Implement**

In `apps/api/app/services/consents.rb`, add near the other pattern constants (e.g. after `CANCEL_PATTERNS`):

```ruby
  # Intenção explícita de REVOGAR consentimento no meio da triagem (LGPD).
  # Distinta de cancel? (sair/parar/cancelar/encerrar) e de "não" (resposta válida).
  REVOKE_INTENT_PATTERNS = [
    /\A\s*(revogar|revogar consentimento|revogar meu consentimento|apagar meus dados)\s*\z/i
  ].freeze

  def self.revoke_intent?(text)
    return false if text.nil?
    REVOKE_INTENT_PATTERNS.any? { |re| text.match?(re) }
  end
```

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/consents_spec.rb`
Expected: PASS (new `.revoke_intent?` examples + existing `.interpret`/`.cancel?` examples green).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/consents.rb spec/services/consents_spec.rb
git -C apps/api commit -m "Add Consents.revoke_intent? for explicit mid-triage revocation"
git -C apps/api log --oneline -1
```

---

### Task 2: `ConversationAdvance` revocation trigger

**Files:**
- Modify: `apps/api/app/commands/conversation_advance.rb`
- Test: `apps/api/spec/commands/conversation_advance_spec.rb`

**Interfaces:**
- Consumes: `Consents.revoke_intent?` (Task 1); `RevokeConsent.call(conversation:, reason:)` (existing — revokes consent, sets conversation `revoked`, aborts the in-progress triage as `aborted_by_revocation`, publishes `consent.revoked`); i18n `conversation_advance.consent_revoked` (existing).
- Produces: a `consented` conversation receiving an explicit revocation intent → `revoked` + `aborted_by_revocation` triage + `consent.revoked` published + inline `consent_revoked` reply.

- [ ] **Step 1: Write the failing tests**

In `apps/api/spec/commands/conversation_advance_spec.rb`, inside `describe "estado :consented (fluxo com motor real)"` (the block with the real protocol + `let!(:consent)`), add:

```ruby
    context "quando o cidadão revoga o consentimento (revogar)" do
      let(:raw_body) { "true" } # 1ª chamada inicia a triagem

      it "revoga: conversa revoked, triage aborted_by_revocation, publica evento e responde" do
        described_class.call(conversation: conversation, inbound: inbound) # inicia a triagem

        revoke_inbound = InboundMessage.create!(
          message_id: "wamid.#{SecureRandom.hex(6)}",
          from: "+5511988888888",
          kind: "text",
          raw: { "type" => "text", "text" => { "body" => "revogar" } }.to_json,
          municipality_id: muni.id
        )
        result = described_class.call(conversation: conversation, inbound: revoke_inbound)

        expect(result.reply.body).to eq(I18n.t("conversation_advance.consent_revoked"))
        expect(conversation.reload.state).to eq("revoked")
        expect(conversation.triages.where(status: "aborted_by_revocation")).to be_present
        expect(DomainEvent.where(name: "consent.revoked", municipality_id: muni.id)).to be_present
      end

      it "não trata 'revogar' como cancelamento" do
        described_class.call(conversation: conversation, inbound: inbound)
        revoke_inbound = InboundMessage.create!(
          message_id: "wamid.#{SecureRandom.hex(6)}",
          from: "+5511988888888", kind: "text",
          raw: { "type" => "text", "text" => { "body" => "revogar" } }.to_json,
          municipality_id: muni.id
        )
        described_class.call(conversation: conversation, inbound: revoke_inbound)
        expect(conversation.reload.state).not_to eq("cancelled")
      end
    end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb`
Expected: FAIL — "revogar" is currently fed to the triage engine (conversation stays `consented`, no `consent.revoked` event, reply is not `consent_revoked`).

- [ ] **Step 3: Implement the trigger**

In `apps/api/app/commands/conversation_advance.rb`, add the revoke check as the FIRST line of `handle_consented` (before the cancel check):

```ruby
  def handle_consented
    return revoke_and_finish if Consents.revoke_intent?(text)
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

Add the private helper next to `cancel_and_finish`:

```ruby
  def revoke_and_finish
    RevokeConsent.call(conversation: @conversation, reason: text)
    Result.new(reply: Messaging::Reply.text(t(:consent_revoked)))
  end
```

(`RevokeConsent` does the synchronous state change + publishes `consent.revoked`. If there is no active consent — an edge that shouldn't occur in `consented` — it no-ops and we still reply `consent_revoked`, a harmless confirmation.)

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb`
Expected: PASS (all examples, including the new revoke context and the "not cancelled" guard).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/commands/conversation_advance.rb spec/commands/conversation_advance_spec.rb
git -C apps/api commit -m "Trigger consent revocation on explicit mid-triage intent"
git -C apps/api log --oneline -1
```

---

### Task 3: `consent.revoked` subscribers + registry wiring

**Files:**
- Create: `apps/api/app/jobs/anonymize_revoked_triage_job.rb`
- Create: `apps/api/app/jobs/record_consent_revocation_job.rb`
- Modify: `apps/api/config/initializers/domain_events.rb`
- Test: `apps/api/spec/jobs/anonymize_revoked_triage_job_spec.rb` (create), `apps/api/spec/jobs/record_consent_revocation_job_spec.rb` (create), `apps/api/spec/initializers/domain_events_bindings_spec.rb` (create)

**Interfaces:**
- Consumes: `consent.revoked` payload `{conversation_id, consent_id, reason}`; `IdempotentConsumer` (perform → with_tenant → ProcessedEvent → `handle(**payload)`); `DashboardMetric.bump!(municipality_id:, dimension:, period:, key:, by: 1)`.
- Produces: `AnonymizeRevokedTriageJob`, `RecordConsentRevocationJob`; `DomainEvents.registry["consent.revoked"]` bound to both.

- [ ] **Step 1: Write the failing tests**

Create `apps/api/spec/jobs/anonymize_revoked_triage_job_spec.rb`:

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe AnonymizeRevokedTriageJob, type: :job do
  self.use_transactional_tests = false
  before { clean_admin_tables }
  after  { clean_admin_tables }

  def definition_hash
    {
      "name" => "rev-demo", "version" => 1, "start_step_id" => "s1",
      "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                    "branches" => { "true" => nil, "false" => nil } }],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 } }
    }
  end

  def event_args(conversation_id, municipality_id, event_id: SecureRandom.uuid)
    { event_id: event_id, event_name: "consent.revoked", municipality_id: municipality_id,
      payload: { "conversation_id" => conversation_id, "consent_id" => SecureRandom.uuid, "reason" => "revogar" } }
  end

  it "scrubs clinical fields of the aborted_by_revocation triage, keeps the audit shell" do
    ctx = nil
    as_admin do
      muni = Municipality.create!(name: "Rev City", slug: "rev-city", ibge_code: "3500050")
      pd = ProtocolDefinition.create!(name: "rev-demo", version: 1, status: "active",
                                      municipality_id: muni.id, definition: definition_hash)
      convo = Conversation.create!(municipality_id: muni.id, phone: "+551133", state: "revoked")
      triage = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "rev-demo",
                              municipality_id: muni.id, status: "aborted_by_revocation",
                              answers: { "s1" => "true" }, outcome: { "tier" => "baixa" },
                              tier: "baixa", priority: 9, current_step: "s1", completed_at: Time.current)
      ctx = { muni: muni.id, convo: convo.id, triage: triage.id }
    end

    described_class.new.perform(**event_args(ctx[:convo], ctx[:muni]))

    as_admin do
      t = Triage.find(ctx[:triage])
      expect(t.answers).to eq({})
      expect(t.outcome).to be_nil
      expect(t.tier).to be_nil
      expect(t.priority).to be_nil
      expect(t.current_step).to be_nil
      # casca de auditoria mantida:
      expect(t.status).to eq("aborted_by_revocation")
      expect(t.protocol_name).to eq("rev-demo")
      expect(t.completed_at).to be_present
    end
  end

  it "is idempotent across distinct deliveries (scrub of already-empty is a no-op)" do
    ctx = nil
    as_admin do
      muni = Municipality.create!(name: "Rev City 2", slug: "rev-city-2", ibge_code: "3500051")
      pd = ProtocolDefinition.create!(name: "rev-demo", version: 1, status: "active",
                                      municipality_id: muni.id, definition: definition_hash)
      convo = Conversation.create!(municipality_id: muni.id, phone: "+551134", state: "revoked")
      Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "rev-demo",
                     municipality_id: muni.id, status: "aborted_by_revocation",
                     answers: { "s1" => "true" }, tier: "baixa")
      ctx = { muni: muni.id, convo: convo.id }
    end
    described_class.new.perform(**event_args(ctx[:convo], ctx[:muni]))
    expect {
      described_class.new.perform(**event_args(ctx[:convo], ctx[:muni])) # distinct event_id
    }.not_to raise_error
    as_admin { expect(Triage.where(conversation_id: ctx[:convo]).first.answers).to eq({}) }
  end

  it "does not touch a completed triage or another conversation's triage" do
    ctx = nil
    as_admin do
      muni = Municipality.create!(name: "Rev City 3", slug: "rev-city-3", ibge_code: "3500052")
      pd = ProtocolDefinition.create!(name: "rev-demo", version: 1, status: "active",
                                      municipality_id: muni.id, definition: definition_hash)
      target = Conversation.create!(municipality_id: muni.id, phone: "+551135", state: "revoked")
      Triage.create!(conversation: target, protocol_definition: pd, protocol_name: "rev-demo",
                     municipality_id: muni.id, status: "aborted_by_revocation", answers: { "s1" => "true" })
      completed_convo = Conversation.create!(municipality_id: muni.id, phone: "+551136", state: "completed")
      done = Triage.create!(conversation: completed_convo, protocol_definition: pd, protocol_name: "rev-demo",
                            municipality_id: muni.id, status: "completed", answers: { "s1" => "true" }, tier: "baixa")
      ctx = { muni: muni.id, convo: target.id, done: done.id }
    end
    described_class.new.perform(**event_args(ctx[:convo], ctx[:muni]))
    as_admin { expect(Triage.find(ctx[:done]).answers).to eq({ "s1" => "true" }) } # completed untouched
  end
end
```

Create `apps/api/spec/jobs/record_consent_revocation_job_spec.rb`:

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe RecordConsentRevocationJob, type: :job do
  self.use_transactional_tests = false
  before { clean_admin_tables }
  after  { clean_admin_tables }

  def event_args(municipality_id, event_id: SecureRandom.uuid)
    { event_id: event_id, event_name: "consent.revoked", municipality_id: municipality_id,
      payload: { "conversation_id" => SecureRandom.uuid, "consent_id" => SecureRandom.uuid, "reason" => "revogar" } }
  end

  it "bumps the consents_revoked/total metric for the day" do
    muni_id = as_admin { Municipality.create!(name: "Metric City", slug: "metric-city", ibge_code: "3500060").id }
    described_class.new.perform(**event_args(muni_id))
    metric = as_admin do
      DashboardMetric.find_by(municipality_id: muni_id, dimension: "consents_revoked", key: "total")
    end
    expect(metric).to be_present
    expect(metric.count).to eq(1)
  end

  it "does not double-count a re-delivered event (ProcessedEvent dedup)" do
    muni_id = as_admin { Municipality.create!(name: "Metric City 2", slug: "metric-city-2", ibge_code: "3500061").id }
    args = event_args(muni_id)
    described_class.new.perform(**args)
    described_class.new.perform(**args) # same event_id
    metric = as_admin { DashboardMetric.find_by(municipality_id: muni_id, dimension: "consents_revoked", key: "total") }
    expect(metric.count).to eq(1)
  end
end
```

Create `apps/api/spec/initializers/domain_events_bindings_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "consent.revoked bindings (F-07.15)" do
  it "binds consent.revoked to the anonymize + record jobs" do
    consumers = DomainEvents.registry["consent.revoked"].map(&:job)
    expect(consumers).to include("AnonymizeRevokedTriageJob", "RecordConsentRevocationJob")
  end
end
```

NOTE on `DashboardMetric#count`: this plan assumes `DashboardMetric` stores the accumulated value in a `count` column (that is what `bump!(by: 1)` increments). Before asserting, open `app/models/dashboard_metric.rb` and confirm the column name; if it differs (e.g. `value`), use the real column in the two `expect(metric.<col>).to eq(1)` lines. The behavior asserted (one bump → 1; duplicate event → still 1) is the requirement.

- [ ] **Step 2: Run to verify they fail**

Run: `docker exec api-dev bundle exec rspec spec/jobs/anonymize_revoked_triage_job_spec.rb spec/jobs/record_consent_revocation_job_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: FAIL — the job classes are undefined and the binding is still `[]`.

- [ ] **Step 3: Create `AnonymizeRevokedTriageJob`**

Create `apps/api/app/jobs/anonymize_revoked_triage_job.rb`:

```ruby
# Consumidor de consent.revoked: apaga o conteúdo clínico da triage abortada
# por revogação (LGPD), mantendo a casca de auditoria. Ver ADR-0005/0020. (F-07.15)
class AnonymizeRevokedTriageJob < ApplicationJob
  include IdempotentConsumer
  queue_as :housekeeping

  def handle(conversation_id:, **)
    Triage.where(conversation_id: conversation_id, status: :aborted_by_revocation).find_each do |t|
      t.update_columns(
        answers: {}, outcome: nil, tier: nil, priority: nil,
        current_step: nil, updated_at: Time.current
      )
    end
  end
end
```

- [ ] **Step 4: Create `RecordConsentRevocationJob`**

Create `apps/api/app/jobs/record_consent_revocation_job.rb`:

```ruby
# Consumidor de consent.revoked: visibilidade leve — incrementa a métrica de
# revogações para o painel de consentimento da cidade. Auditoria bruta já está
# em domain_events. (F-07.15)
class RecordConsentRevocationJob < ApplicationJob
  include IdempotentConsumer
  queue_as :reports

  def handle(conversation_id:, **)
    DashboardMetric.bump!(
      municipality_id: Current.municipality_id,
      dimension: "consents_revoked",
      period: Time.current.to_date.iso8601,
      key: "total"
    )
  end
end
```

- [ ] **Step 5: Wire the registry**

In `apps/api/config/initializers/domain_events.rb`, replace the audit-only binding:

```ruby
  DomainEvents.bind "consent.revoked", to: [AnonymizeRevokedTriageJob, RecordConsentRevocationJob]
```

- [ ] **Step 6: Run to verify they pass**

Run: `docker exec api-dev bundle exec rspec spec/jobs/anonymize_revoked_triage_job_spec.rb spec/jobs/record_consent_revocation_job_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/jobs/anonymize_revoked_triage_job.rb app/jobs/record_consent_revocation_job.rb config/initializers/domain_events.rb spec/jobs/anonymize_revoked_triage_job_spec.rb spec/jobs/record_consent_revocation_job_spec.rb spec/initializers/domain_events_bindings_spec.rb
git -C apps/api commit -m "Handle consent.revoked: anonymize revoked triage and record revocation metric"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec` — expect green.
- Move the board card (F-07.15) to Done, then Verified.
- Sync is a separate explicit step (user-authorized). The F-02.8 final whole-branch review is still pending; it can run together with F-07.15's before sync.

## Self-Review notes

- **Spec coverage:** trigger detector → Task 1; `ConversationAdvance` trigger + inline confirmation → Task 2; both subscribers + registry wiring → Task 3. Anonymization scope (aborted_by_revocation only, keep shell), metric visibility, and idempotency all covered by Task 3 tests.
- **Type consistency:** `Consents.revoke_intent?` (Task 1) used in Task 2; `RevokeConsent.call` + `consent.revoked` payload keys (`conversation_id`) consumed by Task 3 jobs; `DashboardMetric.bump!` signature matches the model; registry references both job classes created in the same task (no boot-time NameError).
- **Precedence:** `revoke_intent?` before `cancel?` in `handle_consented`; "revogar" not in cancel patterns, "não" not in revoke patterns — asserted in Tasks 1 & 2.
- **No migration:** reuses `Triage` clinical columns, `DashboardMetric`, `ProcessedEvent`, existing `consent_revoked` i18n.
- **Idempotency:** `IdempotentConsumer` dedups per (consumer, event_id); the anonymization scrub is also naturally idempotent; both asserted.
