# WhatsApp Interactive Outbound (F-03.3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Render each triage step's `answer_type` as the right WhatsApp element (buttons / list / text) so the citizen taps instead of typing, with the tap returning the exact protocol answer value.

**Architecture:** A `Messaging::Reply` value object replaces the string reply; `Whatsapp::QuestionElement` maps a `Protocols::Step` to a `Reply`; `Whatsapp::Outbound` gains `deliver_interactive`; the reply is threaded through `ConversationAdvance` → `ProcessInboundMessageJob` → `SendWhatsappJob`. The button/row `id` is the protocol answer value, so the existing inbound parser closes the loop with no new code.

**Tech Stack:** Rails 8 (RSpec, run in the `api-dev` container).

## Global Constraints

- **Commits in English** (UI/citizen strings stay Portuguese). `apps/api` on branch `fix/migrations-owner-ddl-as-admin`. Use `git -C apps/api ...` (do not `cd` for git). Do NOT create branches.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. Ruby edits are live (volume); no rebuild.
- WhatsApp Cloud limits: reply buttons max 3 (title ≤20 chars); list rows max 10 (title ≤24 chars).
- Mapping: `boolean`→2 buttons; `enum` ≤3→buttons; `enum` 4–10→list; `enum` >10 / `integer` / `text`→text.
- The button/row `id` is the protocol answer value (`"true"/"false"` for boolean; the option string for enum). Titles are truncated to the limit; the id is kept whole.

---

### Task 1: `Messaging::Reply` value object

**Files:**
- Create: `apps/api/app/messaging/reply.rb`
- Test: `apps/api/spec/messaging/reply_spec.rb`

**Interfaces:**
- Consumes: nothing.
- Produces: `Messaging::Reply` with `.text(body)`, `.buttons(body:, options:)`, `.list(body:, options:)`, readers `kind` (`:text|:buttons|:list`), `body`, `options` (`[{id:,title:}]`), `#text?`, `#to_h` (`{kind:String, body:, options:}`), and `.from_h(hash)`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/messaging/reply_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Messaging::Reply do
  it "builds a text reply" do
    r = described_class.text("oi")
    expect(r.kind).to eq(:text)
    expect(r.body).to eq("oi")
    expect(r.options).to eq([])
    expect(r.text?).to be true
  end

  it "builds a buttons reply" do
    r = described_class.buttons(body: "Tosse?", options: [{ id: "true", title: "Sim" }, { id: "false", title: "Não" }])
    expect(r.kind).to eq(:buttons)
    expect(r.text?).to be false
    expect(r.options.first).to eq(id: "true", title: "Sim")
  end

  it "round-trips through to_h / from_h" do
    r = described_class.list(body: "X", options: [{ id: "a", title: "A" }])
    back = described_class.from_h(r.to_h)
    expect(back.kind).to eq(:list)
    expect(back.body).to eq("X")
    expect(back.options).to eq([{ id: "a", title: "A" }])
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/messaging/reply_spec.rb`
Expected: FAIL — `uninitialized constant Messaging::Reply`.

- [ ] **Step 3: Implement the value object**

Create `apps/api/app/messaging/reply.rb`:

```ruby
# Value Object da resposta de conversa: o que enviar de volta ao cidadão.
# kind :text | :buttons | :list; options = [{id:, title:}] (vazio p/ text).
# Ver F-03.3 (camada de envio interativo do WhatsApp).
module Messaging
  class Reply
    attr_reader :kind, :body, :options

    def self.text(body)               = new(kind: :text, body: body)
    def self.buttons(body:, options:) = new(kind: :buttons, body: body, options: options)
    def self.list(body:, options:)    = new(kind: :list, body: body, options: options)

    def self.from_h(hash)
      h = hash.symbolize_keys
      new(
        kind: h[:kind].to_sym,
        body: h[:body],
        options: Array(h[:options]).map { |o| o.symbolize_keys.slice(:id, :title) }
      )
    end

    def initialize(kind:, body:, options: [])
      @kind = kind
      @body = body
      @options = options.freeze
      freeze
    end

    def text? = kind == :text

    def to_h
      { kind: kind.to_s, body: body, options: options.map { |o| { id: o[:id], title: o[:title] } } }
    end
  end
end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/messaging/reply_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/messaging/reply.rb spec/messaging/reply_spec.rb
git -C apps/api commit -m "Add Messaging::Reply value object for structured conversation replies"
```

---

### Task 2: `Whatsapp::QuestionElement` mapper + i18n labels

**Files:**
- Create: `apps/api/app/services/whatsapp/question_element.rb`
- Create: `apps/api/config/locales/whatsapp.pt-BR.yml`
- Test: `apps/api/spec/services/whatsapp/question_element_spec.rb`

**Interfaces:**
- Consumes: `Messaging::Reply` (Task 1); `Protocols::Step` (has `answer_type` Symbol, `options` Array|nil).
- Produces: `Whatsapp::QuestionElement.for(step, body:) -> Messaging::Reply` per the mapping rules.

- [ ] **Step 1: Add the i18n labels**

Create `apps/api/config/locales/whatsapp.pt-BR.yml`:

```yaml
# Labels dos elementos interativos do WhatsApp (F-03.3).
pt-BR:
  whatsapp:
    btn_yes: "Sim"
    btn_no: "Não"
    list_button: "Escolher"
```

- [ ] **Step 2: Write the failing test**

Create `apps/api/spec/services/whatsapp/question_element_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Whatsapp::QuestionElement do
  def step(answer_type:, options: nil)
    Protocols::Step.new(id: "s", prompt: "?", answer_type: answer_type, options: options)
  end

  it "boolean → 2 buttons with true/false ids and i18n titles" do
    r = described_class.for(step(answer_type: "boolean"), body: "Tosse?")
    expect(r.kind).to eq(:buttons)
    expect(r.body).to eq("Tosse?")
    expect(r.options.map { |o| o[:id] }).to eq(%w[true false])
    expect(r.options.map { |o| o[:title] }).to eq([I18n.t("whatsapp.btn_yes"), I18n.t("whatsapp.btn_no")])
  end

  it "enum with ≤3 options → buttons, id == option" do
    r = described_class.for(step(answer_type: "enum", options: %w[a b c]), body: "Q")
    expect(r.kind).to eq(:buttons)
    expect(r.options.map { |o| o[:id] }).to eq(%w[a b c])
  end

  it "enum with 4..10 options → list" do
    r = described_class.for(step(answer_type: "enum", options: (1..10).map(&:to_s)), body: "Q")
    expect(r.kind).to eq(:list)
    expect(r.options.size).to eq(10)
  end

  it "enum with >10 options → text" do
    r = described_class.for(step(answer_type: "enum", options: (1..11).map(&:to_s)), body: "Q")
    expect(r.kind).to eq(:text)
  end

  it "integer → text" do
    expect(described_class.for(step(answer_type: "integer"), body: "Q").kind).to eq(:text)
  end

  it "text → text" do
    expect(described_class.for(step(answer_type: "text"), body: "Q").kind).to eq(:text)
  end

  it "truncates a long option title but keeps the full id" do
    long = "x" * 30
    r = described_class.for(step(answer_type: "enum", options: [long, "b", "c"]), body: "Q")
    first = r.options.first
    expect(first[:id]).to eq(long)
    expect(first[:title].length).to eq(20)
    expect(first[:title]).to end_with("…")
  end
end
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/question_element_spec.rb`
Expected: FAIL — `uninitialized constant Whatsapp::QuestionElement`.

- [ ] **Step 4: Implement the mapper**

Create `apps/api/app/services/whatsapp/question_element.rb`:

```ruby
# Mapeia o answer_type de um Protocols::Step para o elemento WhatsApp adequado.
# Ver F-03.3. Limites do WhatsApp Cloud: botão título 20, row título 24,
# máx 3 botões, máx 10 rows.
module Whatsapp
  class QuestionElement
    BUTTON_TITLE = 20
    ROW_TITLE = 24
    MAX_BUTTONS = 3
    MAX_ROWS = 10

    def self.for(step, body:)
      new(step, body).call
    end

    def initialize(step, body)
      @step = step
      @body = body
    end

    def call
      case @step.answer_type
      when :boolean
        Messaging::Reply.buttons(body: @body, options: [
          { id: "true",  title: I18n.t("whatsapp.btn_yes") },
          { id: "false", title: I18n.t("whatsapp.btn_no") }
        ])
      when :enum
        opts = Array(@step.options)
        if opts.size <= MAX_BUTTONS
          Messaging::Reply.buttons(body: @body, options: opts.map { |o| option(o, BUTTON_TITLE) })
        elsif opts.size <= MAX_ROWS
          Messaging::Reply.list(body: @body, options: opts.map { |o| option(o, ROW_TITLE) })
        else
          Messaging::Reply.text(@body)
        end
      else
        Messaging::Reply.text(@body)
      end
    end

    private

    def option(value, limit)
      title = value.length > limit ? "#{value[0, limit - 1]}…" : value
      { id: value, title: title }
    end
  end
end
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/question_element_spec.rb`
Expected: PASS (7 examples, 0 failures).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/services/whatsapp/question_element.rb config/locales/whatsapp.pt-BR.yml spec/services/whatsapp/question_element_spec.rb
git -C apps/api commit -m "Add WhatsApp question-element mapper for answer types"
```

---

### Task 3: `Whatsapp::Outbound#deliver_interactive`

**Files:**
- Modify: `apps/api/app/services/whatsapp/outbound.rb`
- Test: `apps/api/spec/services/whatsapp/outbound_spec.rb`

**Interfaces:**
- Consumes: `Messaging::Reply` (Task 1).
- Produces: `Whatsapp::Outbound#interactive_payload(to:, reply:) -> Hash` (the Graph request body) and `#deliver_interactive(to:, reply:) -> Result`. `#deliver_text(to:, body:)` keeps its existing behavior.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/services/whatsapp/outbound_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Whatsapp::Outbound do
  let(:channel) { Struct.new(:phone_number_id, :access_token).new("PNID", "tok") }
  subject(:outbound) { described_class.new(channel) }

  it "builds a button interactive payload (id/title from the reply)" do
    reply = Messaging::Reply.buttons(body: "Tosse?", options: [
      { id: "true", title: "Sim" }, { id: "false", title: "Não" }
    ])
    payload = outbound.interactive_payload(to: "+55119", reply: reply)
    expect(payload[:type]).to eq("interactive")
    expect(payload[:interactive][:type]).to eq("button")
    expect(payload[:interactive][:body]).to eq(text: "Tosse?")
    buttons = payload[:interactive][:action][:buttons]
    expect(buttons.map { |b| b[:reply][:id] }).to eq(%w[true false])
    expect(buttons.first[:type]).to eq("reply")
  end

  it "builds a list interactive payload" do
    reply = Messaging::Reply.list(body: "Escolha", options: [{ id: "a", title: "A" }, { id: "b", title: "B" }])
    payload = outbound.interactive_payload(to: "+55119", reply: reply)
    expect(payload[:interactive][:type]).to eq("list")
    rows = payload[:interactive][:action][:sections].first[:rows]
    expect(rows.map { |r| r[:id] }).to eq(%w[a b])
    expect(payload[:interactive][:action][:button]).to eq(I18n.t("whatsapp.list_button"))
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/outbound_spec.rb`
Expected: FAIL — `NoMethodError: undefined method 'interactive_payload'`.

- [ ] **Step 3: Implement deliver_interactive + payload builder (refactor deliver_text onto a shared post)**

Replace the body of `apps/api/app/services/whatsapp/outbound.rb` with:

```ruby
# Cliente WhatsApp Cloud por canal (ADR-0021/0024).
require "net/http"

module Whatsapp
  class Outbound
    GRAPH = "https://graph.facebook.com/v19.0"

    Result = Struct.new(:status, :body, keyword_init: true)

    def initialize(channel)
      @channel = channel
    end

    def deliver_text(to:, body:)
      post({ messaging_product: "whatsapp", to: to, type: "text", text: { body: body } })
    end

    def deliver_interactive(to:, reply:)
      post(interactive_payload(to: to, reply: reply))
    end

    # Corpo da requisição Graph para um Messaging::Reply interativo.
    def interactive_payload(to:, reply:)
      interactive =
        case reply.kind
        when :buttons
          { type: "button", body: { text: reply.body },
            action: { buttons: reply.options.map { |o| { type: "reply", reply: { id: o[:id], title: o[:title] } } } } }
        when :list
          { type: "list", body: { text: reply.body },
            action: { button: I18n.t("whatsapp.list_button"),
                      sections: [{ rows: reply.options.map { |o| { id: o[:id], title: o[:title] } } }] } }
        end
      { messaging_product: "whatsapp", to: to, type: "interactive", interactive: interactive }
    end

    private

    def post(payload)
      uri = URI("#{GRAPH}/#{@channel.phone_number_id}/messages")
      req = Net::HTTP::Post.new(uri,
        "Authorization" => "Bearer #{@channel.access_token}",
        "Content-Type"  => "application/json"
      )
      req.body = payload.to_json
      response = Net::HTTP.start(uri.host, uri.port, use_ssl: true) { |h| h.request(req) }
      Result.new(status: response.code.to_i, body: response.body.to_s)
    end
  end
end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/outbound_spec.rb`
Expected: PASS (2 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/whatsapp/outbound.rb spec/services/whatsapp/outbound_spec.rb
git -C apps/api commit -m "Add interactive message sending to WhatsApp Outbound"
```

---

### Task 4: Thread the structured reply end to end

**Files:**
- Modify: `apps/api/app/commands/conversation_advance.rb`
- Modify: `apps/api/app/jobs/send_whatsapp_job.rb`
- Modify: `apps/api/app/jobs/process_inbound_message_job.rb`
- Modify: `apps/api/app/jobs/notify_citizen_job.rb`
- Test (update): `apps/api/spec/commands/conversation_advance_spec.rb`
- Test (update): `apps/api/spec/jobs/send_whatsapp_job_spec.rb`

**Interfaces:**
- Consumes: `Messaging::Reply` (Task 1), `Whatsapp::QuestionElement.for` (Task 2), `Whatsapp::Outbound#deliver_interactive`/`#deliver_text` (Task 3).
- Produces: `ConversationAdvance::Result#reply` is now a `Messaging::Reply` (or `nil`); `SendWhatsappJob#perform(to:, message:, municipality_id:, dedup_key: nil)` where `message` is a serialized `Reply` hash.

This is an atomic flip: the reply type and the send-job signature change together so the suite stays green.

- [ ] **Step 1: Update `ConversationAdvance` to produce `Messaging::Reply`**

In `apps/api/app/commands/conversation_advance.rb`: wrap every text reply in `Messaging::Reply.text(...)`, and build step replies via `Whatsapp::QuestionElement.for`. Replace the relevant methods:

```ruby
  def handle_greeting
    @conversation.update!(state: :awaiting_consent)
    Result.new(reply: Messaging::Reply.text(t(:greeting)))
  end

  def handle_awaiting_consent
    case Consents.interpret(text)
    when :give
      result = GiveConsent.call(
        conversation: @conversation,
        version: Consents.current_version(@conversation.municipality_id),
        evidence: { text: text, message_id: @inbound.message_id, channel: "whatsapp" }
      )
      return Result.new(reply: Messaging::Reply.text(t(:consent_failed))) if result.failure?
      begin_triage_and_ask
    when :revoke
      RevokeConsent.call(conversation: @conversation, reason: text)
      Result.new(reply: Messaging::Reply.text(t(:consent_revoked)))
    else
      Result.new(reply: Messaging::Reply.text(t(:consent_prompt)))
    end
  end

  def handle_consented
    triage = active_triage || begin_triage_or_nil
    return Result.new(reply: Messaging::Reply.text(t(:no_protocol))) unless triage

    result = CompleteTriage.call(triage: triage, answer: text)
    return Result.new(reply: Messaging::Reply.text(reason_text(result.reason))) if result.failure?

    outcome = result.payload[:outcome]
    return Result.new(reply: nil) if outcome.terminal?

    triage.reload
    Result.new(reply: step_reply(triage, outcome.awaiting, :triage_next))
  end

  def begin_triage_and_ask
    triage = begin_triage_or_nil
    return Result.new(reply: Messaging::Reply.text(t(:no_protocol))) unless triage
    Result.new(reply: step_reply(triage, triage.current_step, :triage_start))
  end
```

Then replace the `step_prompt` private method with `step_reply`:

```ruby
  def step_reply(triage, step_id, template_key)
    step = triage.protocol.steps[step_id.to_sym]
    return Messaging::Reply.text(t(:triage_generic_error)) unless step
    body = t(template_key, prompt: step.prompt)
    Whatsapp::QuestionElement.for(step, body: body)
  end
```

(Leave `begin_triage_or_nil`, `active_triage`, `reason_text`, `t`, `text`, `extract_body` unchanged. Update the class doc comment's "Result#reply: String" line to say it now returns a `Messaging::Reply` or nil.)

- [ ] **Step 2: Update `SendWhatsappJob` to take a structured `message:`**

In `apps/api/app/jobs/send_whatsapp_job.rb`, change `perform` and the idempotency key, and dispatch by reply kind:

```ruby
  def perform(to:, message:, municipality_id:, dedup_key: nil)
    with_tenant(municipality_id) do
      reply = Messaging::Reply.from_h(message)
      key = idempotency_key(to: to, message: message, municipality_id: municipality_id, dedup_key: dedup_key)

      outbound = nil
      begin
        outbound = OutboundMessage.create!(
          to: to,
          template: message,
          idempotency_key: key,
          municipality_id: municipality_id,
          status: 0,
          context: { dedup_key: dedup_key }.compact
        )
      rescue ActiveRecord::RecordNotUnique
        Rails.logger.info("[SendWhatsappJob] skip duplicate key=#{key}")
        return
      rescue ActiveRecord::RecordInvalid => e
        if e.record.errors.where(:idempotency_key, :taken).any?
          Rails.logger.info("[SendWhatsappJob] skip duplicate key=#{key}")
          return
        end
        raise
      end

      channel = ApplicationRecord.connected_to(role: :admin) do
        MunicipalityChannel.find_by!(municipality_id: municipality_id, active: true)
      end

      client = Whatsapp::Outbound.new(channel)
      result = reply.text? ? client.deliver_text(to: to, body: reply.body)
                           : client.deliver_interactive(to: to, reply: reply)
      outbound.update!(status: result.status, response: result.body)
    end
  end

  private

  def idempotency_key(to:, message:, municipality_id:, dedup_key:)
    digest_input = dedup_key.presence || [to, message.to_json, municipality_id].join("|")
    Digest::SHA256.hexdigest(digest_input)
  end
```

- [ ] **Step 3: Update the two callers**

In `apps/api/app/jobs/process_inbound_message_job.rb`, replace the send line:

```ruby
        SendWhatsappJob.perform_later(to: inbound.from, message: result.reply.to_h, municipality_id: municipality_id) if result&.reply
```

In `apps/api/app/jobs/notify_citizen_job.rb`, replace the `SendWhatsappJob.perform_later(...)` call:

```ruby
    SendWhatsappJob.perform_later(
      to: phone,
      message: Messaging::Reply.text("Sua triage (#{triage.tier}): #{snapshot.url}").to_h,
      municipality_id: triage.municipality_id
    )
```

- [ ] **Step 4: Update `send_whatsapp_job_spec.rb`**

In `apps/api/spec/jobs/send_whatsapp_job_spec.rb`, change every `perform(to:, body: <str>, ...)` call to pass a serialized text reply, and assert dispatch. Replace the three examples' `perform` calls and add an interactive case:

```ruby
  def text_msg(body) = Messaging::Reply.text(body).to_h

  it "escopa MunicipalityChannel por município (escopo manual)" do
    deliver_result = Whatsapp::Outbound::Result.new(status: 200, body: '{"ok":true}')
    outbound = instance_double(Whatsapp::Outbound, deliver_text: deliver_result)
    expect(Whatsapp::Outbound).to receive(:new).with(having_attributes(municipality_id: muni.id)).and_return(outbound)
    described_class.new.perform(to: "+5511988", message: text_msg("ola"), municipality_id: muni.id)
    om = OutboundMessage.last
    expect(om.to).to eq("+5511988")
    expect(om.status).to eq(200)
  end

  it "dedup contra crash-retry: 2ª chamada idêntica não bate HTTP" do
    deliver_result = Whatsapp::Outbound::Result.new(status: 200, body: "ok")
    outbound = instance_double(Whatsapp::Outbound, deliver_text: deliver_result)
    expect(Whatsapp::Outbound).to receive(:new).once.and_return(outbound)
    described_class.new.perform(to: "+551188", message: text_msg("ola"), municipality_id: muni.id)
    described_class.new.perform(to: "+551188", message: text_msg("ola"), municipality_id: muni.id)
    expect(OutboundMessage.where(to: "+551188").count).to eq(1)
  end

  it "despacha interativo quando o reply tem botões" do
    deliver_result = Whatsapp::Outbound::Result.new(status: 200, body: "ok")
    outbound = instance_double(Whatsapp::Outbound)
    expect(outbound).to receive(:deliver_interactive).and_return(deliver_result)
    expect(Whatsapp::Outbound).to receive(:new).and_return(outbound)
    msg = Messaging::Reply.buttons(body: "Tosse?", options: [{ id: "true", title: "Sim" }, { id: "false", title: "Não" }]).to_h
    described_class.new.perform(to: "+551177", message: msg, municipality_id: muni.id)
    expect(OutboundMessage.where(to: "+551177").count).to eq(1)
  end

  it "levanta TenantMissing sem municipality_id" do
    expect {
      described_class.new.perform(to: "+5511988", message: text_msg("ola"), municipality_id: nil)
    }.to raise_error(TenantScopedJob::TenantMissing)
  end
```

- [ ] **Step 5: Update `conversation_advance_spec.rb`**

In `apps/api/spec/commands/conversation_advance_spec.rb`, the reply is now a `Messaging::Reply`. Update each assertion:

- Every `expect(result.reply).to eq(I18n.t("conversation_advance.<key>"))` for a plain-text branch becomes:
  `expect(result.reply.body).to eq(I18n.t("conversation_advance.<key>"))`.
  Apply to: `greeting`, `no_protocol`, `consent_revoked`, `consent_prompt` (and any other text branch).
- The "triage_start" assertion (first step `tosse` is boolean in `protocol_definition_hash`) becomes:
  ```ruby
  expect(result.reply.body).to eq(I18n.t("conversation_advance.triage_start", prompt: "Você está com tosse?"))
  expect(result.reply.kind).to eq(:buttons)
  expect(result.reply.options.map { |o| o[:id] }).to eq(%w[true false])
  ```
- The "triage_next" assertion (next step `febre` is boolean) becomes:
  ```ruby
  expect(result.reply.body).to eq(I18n.t("conversation_advance.triage_next", prompt: "Está com febre alta?"))
  expect(result.reply.kind).to eq(:buttons)
  ```
- `expect(result.reply).to be_nil` (terminal branch, and the `:revoked`/greeting-state-machine no-reply cases) stays unchanged.

- [ ] **Step 6: Run the affected specs to verify they pass**

Run: `docker exec api-dev bundle exec rspec spec/commands/conversation_advance_spec.rb spec/jobs/send_whatsapp_job_spec.rb spec/jobs/process_inbound_message_job_spec.rb`
Expected: PASS. (If `process_inbound_message_job_spec.rb` does not exist, omit it.)

- [ ] **Step 7: Run the full suite (regression — the reply type changed widely)**

Run: `docker exec api-dev bundle exec rspec`
Expected: PASS, 0 failures.

- [ ] **Step 8: Commit**

```bash
git -C apps/api add app/commands/conversation_advance.rb app/jobs/send_whatsapp_job.rb app/jobs/process_inbound_message_job.rb app/jobs/notify_citizen_job.rb spec/commands/conversation_advance_spec.rb spec/jobs/send_whatsapp_job_spec.rb
git -C apps/api commit -m "Thread structured interactive reply through conversation and send path"
```

---

## Wrap-up (after all tasks)

- Full api suite green in the container.
- Live check: a boolean/enum step in a triage produces an interactive payload (verify via a `Whatsapp::Outbound#interactive_payload` call or a dev send). The citizen's tap returns the `id` (= answer value), which the existing inbound parser feeds back to the engine.
- Move board card F-03.3 to Done, then Verified.
- Next in the slice: F-02.4 (consent buttons), F-01.7 (template outside 24h). Sync is a separate explicit step.

## Self-Review notes

- **Spec coverage:** Reply VO → Task 1; mapper + i18n → Task 2; Outbound interactive → Task 3; threading (ConversationAdvance produce, SendWhatsappJob consume, both callers, both spec updates) → Task 4. Closed loop relies on the existing inbound parser (no task needed). All spec sections mapped.
- **Type consistency:** `Messaging::Reply` (`kind`/`body`/`options`, `to_h`/`from_h`, `text?`) defined in Task 1 and consumed identically in Tasks 2–4; `QuestionElement.for(step, body:)` defined in Task 2, called in Task 4 `step_reply`; `Outbound#deliver_interactive(to:, reply:)`/`#deliver_text(to:, body:)` defined in Task 3, dispatched in Task 4; `SendWhatsappJob#perform(message:)` defined in Task 4 and called by both updated callers.
- **Atomic flip:** Task 4 changes the reply type and the job signature together (with both spec updates) so the suite is never left red between tasks.
- **Placeholder scan:** every code/test step has complete content and exact commands; no TBD/TODO.
