# Consent Interactive Buttons (F-02.4) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Render the consent ask as two WhatsApp buttons (Sim / Não) and recognize the tap deterministically, while keeping free-text consent working.

**Architecture:** `Consents` gains dedicated payload-id constants and matches them in `interpret` before the free-text regex. `ConversationAdvance` sends the greeting and the consent re-prompt as a `Messaging::Reply.buttons` (built via F-03.3's interactive layer) whose option ids are those constants; the existing inbound parser surfaces a tap as the button id, which `interpret` matches.

**Tech Stack:** Rails 8 (RSpec, run in the `api-dev` container).

## Global Constraints

- **Commits in English** (UI labels stay Portuguese). `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. Ruby edits live; no rebuild.
- Reuse F-03.3: `Messaging::Reply.buttons(body:, options: [{id:,title:}])`, the i18n labels `whatsapp.btn_yes` / `whatsapp.btn_no` ("Sim"/"Não").
- A button tap arrives (via the inbound parser) as the message body equal to the button `id`; free-text "sim"/"não" must still work via the regex fallback.
- When committing, run `git -C apps/api commit` and CONFIRM with `git -C apps/api log --oneline -1` before reporting DONE.

---

### Task 1: `Consents` payload ids + `interpret` match

**Files:**
- Modify: `apps/api/app/services/consents.rb`
- Test: `apps/api/spec/services/consents_spec.rb` (create)

**Interfaces:**
- Consumes: nothing.
- Produces: `Consents::GIVE_ID == "consent_give"`, `Consents::REVOKE_ID == "consent_revoke"`; `Consents.interpret(text)` returns `:give` for `GIVE_ID`, `:revoke` for `REVOKE_ID` (matched before the free-text regex), and the existing regex behavior otherwise.

- [ ] **Step 1: Write the failing spec**

Create `apps/api/spec/services/consents_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Consents do
  describe ".interpret" do
    it "maps the give button id to :give" do
      expect(described_class.interpret(Consents::GIVE_ID)).to eq(:give)
    end

    it "maps the revoke button id to :revoke" do
      expect(described_class.interpret(Consents::REVOKE_ID)).to eq(:revoke)
    end

    it "still maps free-text affirmation to :give" do
      expect(described_class.interpret("sim")).to eq(:give)
    end

    it "still maps free-text refusal to :revoke" do
      expect(described_class.interpret("não")).to eq(:revoke)
    end

    it "returns :unknown for unrecognized text" do
      expect(described_class.interpret("talvez")).to eq(:unknown)
    end

    it "returns :unknown for blank and nil" do
      expect(described_class.interpret("")).to eq(:unknown)
      expect(described_class.interpret(nil)).to eq(:unknown)
    end
  end
end
```

- [ ] **Step 2: Run the spec to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/consents_spec.rb`
Expected: FAIL — `uninitialized constant Consents::GIVE_ID` (the constants don't exist yet).

- [ ] **Step 3: Implement the constants + interpret change**

In `apps/api/app/services/consents.rb`, add the constants near the top of the module (after the existing `REVOKE_PATTERNS` block):

```ruby
  # IDs de payload dos botões interativos de consentimento (F-02.4).
  GIVE_ID   = "consent_give".freeze
  REVOKE_ID = "consent_revoke".freeze
```

Then replace `interpret` so the button ids match before the free-text regex (preserving the existing revoke-before-give caution bias for the regex fallback):

```ruby
  def self.interpret(text)
    return :unknown if text.nil? || text.strip.empty?
    return :give    if text == GIVE_ID
    return :revoke  if text == REVOKE_ID
    return :revoke  if REVOKE_PATTERNS.any? { |re| text.match?(re) }
    return :give    if GIVE_PATTERNS.any?  { |re| text.match?(re) }
    :unknown
  end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/consents_spec.rb`
Expected: PASS (6 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/consents.rb spec/services/consents_spec.rb
git -C apps/api commit -m "Recognize consent button payload ids in Consents.interpret"
git -C apps/api log --oneline -1
```

---

### Task 2: `ConversationAdvance` sends consent as buttons

**Files:**
- Modify: `apps/api/app/commands/conversation_advance.rb`
- Test: `apps/api/spec/commands/conversation_advance_spec.rb`

**Interfaces:**
- Consumes: `Consents::GIVE_ID` / `Consents::REVOKE_ID` (Task 1); `Messaging::Reply.buttons`; `I18n.t("whatsapp.btn_yes"/"btn_no")`.
- Produces: `handle_greeting` and the `:unknown` consent re-prompt return a `:buttons` `Messaging::Reply` whose option ids are `["consent_give", "consent_revoke"]`.

- [ ] **Step 1: Update the greeting and re-prompt specs + add button-tap tests**

In `apps/api/spec/commands/conversation_advance_spec.rb`:

(a) Extend the `:greeting` example (currently asserts `result.reply.body`) to also assert the interactive shape:

```ruby
  describe "estado :greeting" do
    let(:raw_body) { "oi" }

    it "responde com greeting em botões e move para awaiting_consent" do
      result = described_class.call(conversation: conversation, inbound: inbound)
      expect(result.reply.body).to eq(I18n.t("conversation_advance.greeting"))
      expect(result.reply.kind).to eq(:buttons)
      expect(result.reply.options.map { |o| o[:id] }).to eq(%w[consent_give consent_revoke])
      expect(conversation.reload.state).to eq("awaiting_consent")
    end
  end
```

(b) Extend the `:unknown` re-prompt example (the `raw_body "talvez"` one) to assert buttons:

```ruby
      it "responde com prompt re-perguntando consent em botões" do
        result = described_class.call(conversation: conversation, inbound: inbound)
        expect(result.reply.body).to eq(I18n.t("conversation_advance.consent_prompt"))
        expect(result.reply.kind).to eq(:buttons)
        expect(result.reply.options.map { |o| o[:id] }).to eq(%w[consent_give consent_revoke])
        expect(conversation.reload.state).to eq("awaiting_consent")
      end
```

(c) Add a button-tap give example inside the `describe "estado :awaiting_consent"` block (mirrors the existing "sim" no-protocol context — a tap whose body is the give id records consent and moves to `consented`):

```ruby
    context "quando toca o botão Sim (id consent_give)" do
      let(:raw_body) { "consent_give" }

      it "registra consent e move para consented" do
        result = described_class.call(conversation: conversation, inbound: inbound)
        expect(conversation.reload.state).to eq("consented")
        expect(conversation.consents.count).to eq(1)
      end
    end

    context "quando toca o botão Não (id consent_revoke)" do
      let(:raw_body) { "consent_revoke" }

      it "revoga e move para revoked" do
        conversation.consents.create!(
          version: 1, evidence: { text: "sim" }, given_at: Time.current
        )
        result = described_class.call(conversation: conversation, inbound: inbound)
        expect(result.reply.body).to eq(I18n.t("conversation_advance.consent_revoked"))
        expect(conversation.reload.state).to eq("revoked")
      end
    end
```

Note: match the `consents.create!` evidence/columns to whatever the existing revoke example in this file uses (copy that example's setup verbatim if the attributes differ).

- [ ] **Step 2: Run the spec to verify the new assertions fail**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb`
Expected: FAIL — the greeting/re-prompt examples fail on `kind == :buttons` (they are currently `:text`).

- [ ] **Step 3: Implement the consent buttons**

In `apps/api/app/commands/conversation_advance.rb`:

(a) Change `handle_greeting` (the `Result.new(reply: ...)` line) to:

```ruby
  def handle_greeting
    @conversation.update!(state: :awaiting_consent)
    Result.new(reply: consent_reply(t(:greeting)))
  end
```

(b) Change the `:unknown`/`else` branch of `handle_awaiting_consent` to:

```ruby
    else
      Result.new(reply: consent_reply(t(:consent_prompt)))
    end
```

(c) Add the private helper (next to the other private helpers, e.g. after `t`):

```ruby
  def consent_reply(body)
    Messaging::Reply.buttons(
      body: body,
      options: [
        { id: Consents::GIVE_ID,   title: I18n.t("whatsapp.btn_yes") },
        { id: Consents::REVOKE_ID, title: I18n.t("whatsapp.btn_no") }
      ]
    )
  end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb`
Expected: PASS (all examples, 0 failures).

- [ ] **Step 5: Run the conversation/consent regression group**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb spec/services/consents_spec.rb`
Expected: PASS — both green.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/commands/conversation_advance.rb spec/commands/conversation_advance_spec.rb
git -C apps/api commit -m "Send consent prompt as interactive buttons"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec` — expect green.
- Live check (optional, reuses the dev `protocol_author`/channel work): a button tap body `consent_give` advances to the triage question; typed "sim" still works.
- Move the board card (F-02.4) to Done, then Verified.
- Sync is a separate explicit step.

## Self-Review notes

- **Spec coverage:** payload ids + interpret match → Task 1; greeting & re-prompt as buttons + tap handling → Task 2. All spec sections mapped.
- **Type consistency:** `Consents::GIVE_ID`/`REVOKE_ID` defined in Task 1 and consumed by `consent_reply` + the spec option-id assertions in Task 2; `consent_reply` returns a `Messaging::Reply` (kind `:buttons`) consumed by the existing send path.
- **Placeholder scan:** complete code + exact commands throughout. The one note (copy the revoke example's `consents.create!` attributes if they differ) points the implementer at the authoritative existing example rather than guessing.
- **Backward compat:** free-text "sim"/"não" still resolve via the regex fallback (Task 1 spec asserts this); only the outbound representation and the id-match are added.
