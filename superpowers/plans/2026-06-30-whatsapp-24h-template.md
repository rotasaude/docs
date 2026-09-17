# WhatsApp 24h-Window Template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Send a pre-approved WhatsApp template when the 24h customer-service window has closed, instead of a free-form message that the Graph API would reject.

**Architecture:** Add a `:template` kind to `Messaging::Reply`, `Whatsapp::Outbound#deliver_template`, and `Whatsapp::SessionWindow.open?` (last inbound < 24h). `SendWhatsappJob` dispatches templates always, free-form within the window, and substitutes a `rota_saude_resume` re-engagement template for free-form sent outside the window.

**Tech Stack:** Rails 8 (RSpec, run in the `api-dev` container).

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. Ruby edits live; no rebuild.
- The window is a fixed 24h, measured from the most recent `InboundMessage.created_at` for the recipient phone (`from`) in the tenant. The re-engagement template name is `rota_saude_resume` (no params).
- `Messaging::Reply` autoloads under `Messaging`; `Whatsapp::*` under `app/services/whatsapp/`.
- When committing, run `git -C apps/api commit` and CONFIRM with `git -C apps/api log --oneline -1` before reporting DONE.

---

### Task 1: `Messaging::Reply` `:template` kind

**Files:**
- Modify: `apps/api/app/messaging/reply.rb`
- Test: `apps/api/spec/messaging/reply_spec.rb`

**Interfaces:**
- Produces: `Messaging::Reply.template(name:, params: [])` → a frozen reply with `kind == :template`, `name` (String), `params` (Array<String>); readers `name`/`params` (default `nil`/`[]` for other kinds); `template?`; `to_h` includes `name`/`params`; `from_h` restores them.

- [ ] **Step 1: Write the failing test**

Append to `apps/api/spec/messaging/reply_spec.rb` (inside the top-level `describe`):

```ruby
  describe ".template" do
    it "builds a template reply with name and params" do
      r = Messaging::Reply.template(name: "rota_saude_resume", params: ["Curitiba"])
      expect(r.kind).to eq(:template)
      expect(r.template?).to be(true)
      expect(r.name).to eq("rota_saude_resume")
      expect(r.params).to eq(["Curitiba"])
    end

    it "defaults params to []" do
      expect(Messaging::Reply.template(name: "rota_saude_resume").params).to eq([])
    end

    it "round-trips through to_h/from_h" do
      r = Messaging::Reply.template(name: "rota_saude_resume", params: ["x"])
      back = Messaging::Reply.from_h(r.to_h)
      expect(back.kind).to eq(:template)
      expect(back.name).to eq("rota_saude_resume")
      expect(back.params).to eq(["x"])
    end

    it "leaves name nil / params [] for a text reply round-trip" do
      back = Messaging::Reply.from_h(Messaging::Reply.text("oi").to_h)
      expect(back.name).to be_nil
      expect(back.params).to eq([])
    end
  end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/messaging/reply_spec.rb`
Expected: FAIL — `NoMethodError: undefined method 'template' for Messaging::Reply`.

- [ ] **Step 3: Implement**

Replace `apps/api/app/messaging/reply.rb` with:

```ruby
# Value Object da resposta de conversa: o que enviar de volta ao cidadão.
# kind :text | :buttons | :list | :template.
#   text/buttons/list: body (+ options [{id:, title:}] p/ buttons/list).
#   template: name (template aprovado) + params (parâmetros do componente body).
# Ver F-03.3 (envio interativo) e F-01.7 (template fora da janela 24h).
module Messaging
  class Reply
    attr_reader :kind, :body, :options, :name, :params

    def self.text(body)                  = new(kind: :text, body: body)
    def self.buttons(body:, options:)    = new(kind: :buttons, body: body, options: options)
    def self.list(body:, options:)       = new(kind: :list, body: body, options: options)
    def self.template(name:, params: []) = new(kind: :template, name: name, params: params)

    def self.from_h(hash)
      h = hash.symbolize_keys
      new(
        kind: h[:kind].to_sym,
        body: h[:body],
        options: Array(h[:options]).map { |o| o.symbolize_keys.slice(:id, :title) },
        name: h[:name],
        params: Array(h[:params])
      )
    end

    def initialize(kind:, body: nil, options: [], name: nil, params: [])
      @kind = kind
      @body = body
      @options = options.freeze
      @name = name
      @params = params.freeze
      freeze
    end

    def text?     = kind == :text
    def template? = kind == :template

    def to_h
      {
        kind: kind.to_s,
        body: body,
        options: options.map { |o| { id: o[:id], title: o[:title] } },
        name: name,
        params: params
      }
    end
  end
end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/messaging/reply_spec.rb`
Expected: PASS — the new template examples plus the pre-existing text/buttons/round-trip examples all green.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/messaging/reply.rb spec/messaging/reply_spec.rb
git -C apps/api commit -m "Add template kind to Messaging::Reply"
git -C apps/api log --oneline -1
```

---

### Task 2: `Whatsapp::Outbound#deliver_template`

**Files:**
- Modify: `apps/api/app/services/whatsapp/outbound.rb`
- Test: `apps/api/spec/services/whatsapp/outbound_spec.rb`

**Interfaces:**
- Consumes: `Messaging::Reply` with `kind == :template` (`name`, `params`).
- Produces: `Whatsapp::Outbound#deliver_template(to:, reply:)` posts a `type:"template"` Graph payload and returns the `Result` struct (status, body); `template_payload(to:, reply:)` is the public builder used by the spec.

- [ ] **Step 1: Write the failing test**

Append to `apps/api/spec/services/whatsapp/outbound_spec.rb` (follow the existing file's setup for the channel/stub; mirror how the existing `interactive_payload` examples are written):

```ruby
  describe "#template_payload" do
    let(:reply) { Messaging::Reply.template(name: "rota_saude_resume", params: ["Curitiba"]) }

    it "builds a type:template payload with name, language and body params" do
      payload = described_class.new(channel).template_payload(to: "5511999", reply: reply)
      expect(payload[:type]).to eq("template")
      expect(payload[:template][:name]).to eq("rota_saude_resume")
      expect(payload[:template][:language]).to eq({ code: "pt_BR" })
      params = payload[:template][:components].first[:parameters]
      expect(params).to eq([{ type: "text", text: "Curitiba" }])
    end

    it "emits empty components when there are no params" do
      reply = Messaging::Reply.template(name: "rota_saude_resume")
      payload = described_class.new(channel).template_payload(to: "5511999", reply: reply)
      expect(payload[:template][:components]).to eq([])
    end
  end
```

(If the spec file references `channel` via a `let` already defined for the interactive examples, reuse it; otherwise copy that `let(:channel)` definition.)

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/outbound_spec.rb`
Expected: FAIL — `NoMethodError: undefined method 'template_payload'`.

- [ ] **Step 3: Implement**

In `apps/api/app/services/whatsapp/outbound.rb`, add (next to `deliver_interactive` / `interactive_payload`, above `private`):

```ruby
    def deliver_template(to:, reply:)
      post(template_payload(to: to, reply: reply))
    end

    # Corpo da requisição Graph para um template aprovado (F-01.7).
    def template_payload(to:, reply:)
      components = reply.params.empty? ? [] :
        [{ type: "body", parameters: reply.params.map { |p| { type: "text", text: p } } }]
      {
        messaging_product: "whatsapp", to: to, type: "template",
        template: { name: reply.name, language: { code: "pt_BR" }, components: components }
      }
    end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/outbound_spec.rb`
Expected: PASS (the two new template examples plus the existing ones).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/whatsapp/outbound.rb spec/services/whatsapp/outbound_spec.rb
git -C apps/api commit -m "Add template message sending to WhatsApp Outbound"
git -C apps/api log --oneline -1
```

---

### Task 3: `Whatsapp::SessionWindow`

**Files:**
- Create: `apps/api/app/services/whatsapp/session_window.rb`
- Test: `apps/api/spec/services/whatsapp/session_window_spec.rb`

**Interfaces:**
- Produces: `Whatsapp::SessionWindow.open?(phone:, municipality_id:) -> Boolean` — `true` iff the most recent `InboundMessage.created_at` for `from == phone` in the tenant is within 24h; `false` when none or all older.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/services/whatsapp/session_window_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Whatsapp::SessionWindow do
  let(:muni) { create(:municipality) }
  let(:phone) { "5511999990000" }

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

  def inbound(at:)
    InboundMessage.create!(
      message_id: "wamid.#{SecureRandom.hex(6)}", from: phone, kind: "text",
      raw: { "type" => "text" }.to_json, municipality_id: muni.id, created_at: at
    )
  end

  it "is open when the last inbound is within 24h" do
    inbound(at: 2.hours.ago)
    expect(described_class.open?(phone: phone, municipality_id: muni.id)).to be(true)
  end

  it "is closed when the last inbound is older than 24h" do
    inbound(at: 25.hours.ago)
    expect(described_class.open?(phone: phone, municipality_id: muni.id)).to be(false)
  end

  it "is closed when there is no inbound" do
    expect(described_class.open?(phone: phone, municipality_id: muni.id)).to be(false)
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/session_window_spec.rb`
Expected: FAIL — `uninitialized constant Whatsapp::SessionWindow`.

- [ ] **Step 3: Implement**

Create `apps/api/app/services/whatsapp/session_window.rb`:

```ruby
# Janela de atendimento de 24h do WhatsApp (F-01.7). Fora dela, só template
# aprovado pode ser enviado. Determinada pelo último inbound do telefone.
module Whatsapp
  module SessionWindow
    WINDOW = 24.hours

    def self.open?(phone:, municipality_id:)
      last = InboundMessage.where(from: phone, municipality_id: municipality_id).maximum(:created_at)
      last.present? && last > WINDOW.ago
    end
  end
end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/services/whatsapp/session_window_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/services/whatsapp/session_window.rb spec/services/whatsapp/session_window_spec.rb
git -C apps/api commit -m "Add WhatsApp 24h session window helper"
git -C apps/api log --oneline -1
```

---

### Task 4: `SendWhatsappJob` window guard + resume substitution

**Files:**
- Modify: `apps/api/app/jobs/send_whatsapp_job.rb`
- Test: `apps/api/spec/jobs/send_whatsapp_job_spec.rb`

**Interfaces:**
- Consumes: `Messaging::Reply.template` (Task 1), `Whatsapp::Outbound#deliver_template` (Task 2), `Whatsapp::SessionWindow.open?` (Task 3).
- Produces: `SendWhatsappJob` sends a `:template` message via `deliver_template` always; a free-form message via `deliver_text`/`deliver_interactive` when the window is open; and substitutes `RESUME_TEMPLATE` (`rota_saude_resume`) via `deliver_template` when the window is closed. `OutboundMessage.template` records what was actually sent.

- [ ] **Step 1: Write the failing test**

Add to `apps/api/spec/jobs/send_whatsapp_job_spec.rb` (reuse the file's existing harness: the `muni`, the `MunicipalityChannel` creation, and the tenant context already used by the existing dispatch examples — copy that setup verbatim into the new contexts):

```ruby
  describe "24h window guard" do
    let(:client) { instance_double(Whatsapp::Outbound) }

    before do
      allow(Whatsapp::Outbound).to receive(:new).and_return(client)
      allow(client).to receive(:deliver_text).and_return(Whatsapp::Outbound::Result.new(status: 200, body: "{}"))
      allow(client).to receive(:deliver_interactive).and_return(Whatsapp::Outbound::Result.new(status: 200, body: "{}"))
      allow(client).to receive(:deliver_template).and_return(Whatsapp::Outbound::Result.new(status: 200, body: "{}"))
    end

    it "sends a template message via deliver_template (any window state)" do
      allow(Whatsapp::SessionWindow).to receive(:open?).and_return(false)
      msg = Messaging::Reply.template(name: "rota_saude_ask").to_h
      described_class.new.perform(to: "5511999", message: msg, municipality_id: muni.id)
      expect(client).to have_received(:deliver_template)
    end

    it "sends free-form text within the window" do
      allow(Whatsapp::SessionWindow).to receive(:open?).and_return(true)
      msg = Messaging::Reply.text("Olá").to_h
      described_class.new.perform(to: "5511999", message: msg, municipality_id: muni.id)
      expect(client).to have_received(:deliver_text)
    end

    it "substitutes the resume template for free-form outside the window" do
      allow(Whatsapp::SessionWindow).to receive(:open?).and_return(false)
      msg = Messaging::Reply.text("Olá").to_h
      described_class.new.perform(to: "5511999", message: msg, municipality_id: muni.id)
      expect(client).to have_received(:deliver_template) do |to:, reply:|
        expect(reply.name).to eq("rota_saude_resume")
      end
      expect(client).not_to have_received(:deliver_text)
    end
  end
```

(If the existing file already stubs the channel via `MunicipalityChannel.find_by!`, the `instance_double(Whatsapp::Outbound)` + `Whatsapp::Outbound.new` stub above bypasses real HTTP; ensure a `MunicipalityChannel` for `muni` exists as the existing examples set up — copy that `let!`/`before` from the file.)

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/jobs/send_whatsapp_job_spec.rb -e "24h window guard"`
Expected: FAIL — the resume-substitution example fails (free-form outside the window currently calls `deliver_text`, not `deliver_template`).

- [ ] **Step 3: Implement**

In `apps/api/app/jobs/send_whatsapp_job.rb`:

(a) Add the resume constant at the top of the class (after `include TenantScopedJob`):

```ruby
  RESUME_TEMPLATE = Messaging::Reply.template(name: "rota_saude_resume").freeze
```

(b) Replace the dispatch block (the current lines that build `client` and the `result = reply.text? ? ... : ...` and `outbound.update!`):

```ruby
      client = Whatsapp::Outbound.new(channel)

      sent =
        if reply.kind != :template && !Whatsapp::SessionWindow.open?(phone: to, municipality_id: municipality_id)
          RESUME_TEMPLATE
        else
          reply
        end

      result =
        case sent.kind
        when :template then client.deliver_template(to: to, reply: sent)
        when :text     then client.deliver_text(to: to, body: sent.body)
        else                client.deliver_interactive(to: to, reply: sent)
        end

      outbound.update!(status: result.status, response: result.body, template: sent.to_h)
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/jobs/send_whatsapp_job_spec.rb`
Expected: PASS — the new window-guard examples plus the existing dispatch/dedup examples all green.

- [ ] **Step 5: Run the WhatsApp regression group**

Run: `docker exec api-dev bundle exec rspec spec/jobs/send_whatsapp_job_spec.rb spec/services/whatsapp spec/messaging`
Expected: PASS — reply, outbound, session_window, and job specs all green.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/jobs/send_whatsapp_job.rb spec/jobs/send_whatsapp_job_spec.rb
git -C apps/api commit -m "Substitute a resume template when the 24h window is closed"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec` — expect green.
- Move the board card (F-01.7) to Done, then Verified.
- Sync is a separate explicit step. This closes the citizen-runtime slice (F-03.3 + F-02.4 + F-01.7).

## Self-Review notes

- **Spec coverage:** `:template` reply kind → Task 1; `deliver_template` → Task 2; `SessionWindow.open?` → Task 3; the SendWhatsappJob guard (template always / free-form in-window / resume substitution) + `OutboundMessage.template` records what was sent → Task 4. All spec sections mapped.
- **Type consistency:** `Messaging::Reply.template(name:, params:)` / `.name` / `.params` / `.template?` defined in Task 1 and consumed in Tasks 2 & 4; `deliver_template(to:, reply:)` defined in Task 2 and called in Task 4; `SessionWindow.open?(phone:, municipality_id:)` defined in Task 3 and called in Task 4; `RESUME_TEMPLATE` is a `Messaging::Reply` whose `.name == "rota_saude_resume"`.
- **Placeholder scan:** complete code + exact commands. The two "reuse the existing file's harness" notes (outbound_spec `channel` let, send_whatsapp_job_spec channel/tenant setup) point the implementer at authoritative existing setup rather than guessing — not placeholders.
- **Idempotency:** the key stays computed from the requested `message` (Task 4 leaves `idempotency_key` untouched), so retries of the same requested send dedup even when substituted.
