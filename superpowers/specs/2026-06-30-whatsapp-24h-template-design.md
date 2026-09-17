# F-01.7 — Approved template outside the 24h window

**Date:** 2026-06-30
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-01
**Board:** F-01.7 (In Progress)
**Touches:** `apps/api` only. Closes the "citizen runtime" slice (after F-03.3, F-02.4).

## Problem

WhatsApp Cloud only allows free-form messages within 24h of the citizen's last
inbound (the "customer service window"); outside it, only pre-approved **template**
messages may be sent. Today `Whatsapp::Outbound` sends only `deliver_text` /
`deliver_interactive` (F-03.3) — there is no template send and no window
detection. A free-form message sent outside the window would be rejected by the
Graph API. (`Triage#template` builds a template payload but it is never sent.)

The current reply path always replies to a just-received inbound, so it is always
in-window; the closed-window case arises only for out-of-band sends (future
re-engagement / the F-02.7 abandoned sweep). F-01.7 builds the template-send
capability and makes the send path window-aware so it degrades gracefully.

## Decisions (from brainstorming)

1. **Capability + guard with template substitution.** Add `deliver_template`, a
   window helper, and a `:template` reply kind; and a guard in `SendWhatsappJob`:
   a free-form message sent while the window is closed is substituted by a
   pre-approved re-engagement template.
2. Sub-decisions: re-engagement template name `rota_saude_resume`, no params;
   window = a fixed 24h.

## Design

### 1. `Messaging::Reply` `:template` kind

`Messaging::Reply.template(name:, params: [])` → `kind: :template` with `name`
(String, e.g. `"rota_saude_resume"`) and `params` (Array of String body
parameters). The VO gains `name` (default `nil`) and `params` (default `[]`)
readers; the existing `text`/`buttons`/`list` constructors leave them at the
defaults. `#to_h` includes `name` and `params`; `.from_h` restores them.

### 2. `Whatsapp::Outbound#deliver_template(to:, reply:)`

Builds the Graph `template` payload and posts it via the existing private `post`:

```ruby
{ messaging_product: "whatsapp", to: to, type: "template",
  template: { name: reply.name, language: { code: "pt_BR" },
              components: reply.params.empty? ? [] :
                [{ type: "body", parameters: reply.params.map { |p| { type: "text", text: p } } }] } }
```

Returns the same `Result` struct (status, body). `deliver_text` /
`deliver_interactive` are unchanged.

### 3. `Whatsapp::SessionWindow.open?(phone:, municipality_id:)`

Returns `true` iff the most recent `InboundMessage.created_at` for that `from`
phone in the tenant is within 24h:

```ruby
last = InboundMessage.where(from: phone, municipality_id: municipality_id).maximum(:created_at)
last.present? && last > 24.hours.ago
```

Called inside `SendWhatsappJob`'s `with_tenant` block (RLS set). No inbound → `false`.

### 4. `SendWhatsappJob` window guard

After rebuilding `reply = Messaging::Reply.from_h(message)` and resolving the
channel, the dispatch becomes:

```ruby
result =
  if reply.kind == :template
    Whatsapp::Outbound.new(channel).deliver_template(to: to, reply: reply)
  elsif Whatsapp::SessionWindow.open?(phone: to, municipality_id: municipality_id)
    reply.text? ? deliver_text(...) : deliver_interactive(..., reply)
  else
    Whatsapp::Outbound.new(channel).deliver_template(to: to, reply: RESUME_TEMPLATE)
  end
```

- `RESUME_TEMPLATE = Messaging::Reply.template(name: "rota_saude_resume")` (a job
  constant).
- A `:template` reply is always allowed (in or out of window).
- A free-form reply is sent as-is within the window; outside, it is substituted
  by `RESUME_TEMPLATE`.
- `OutboundMessage.template` records what was actually sent (the original message
  hash for in-window, or the resume template hash when substituted), so the
  stored record reflects the real send. The idempotency key stays keyed on the
  requested `message` (so retries of the same requested send still dedup).

### 5. Re-engagement loop

`rota_saude_resume` re-opens the window; the citizen's reply flows through
`ProcessInboundMessageJob` → `ConversationAdvance`, which resumes from the
conversation's current state (re-asking the pending step). The free-form content
that could not be sent is dropped — acceptable for MVP re-engagement.

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

- **`Messaging::Reply.template`**: sets `kind: :template`, `name`, `params`;
  `to_h`/`from_h` round-trip the template fields (and existing text/buttons still
  round-trip with `name: nil`, `params: []`).
- **`Whatsapp::Outbound#deliver_template`**: builds the correct `type:"template"`
  payload (name, `pt_BR`, body params present/absent) — HTTP stubbed, assert the
  request body.
- **`Whatsapp::SessionWindow`**: a recent inbound → `open? == true`; an inbound
  >24h old → `false`; no inbound → `false`.
- **`SendWhatsappJob`**: a `:template` message dispatches `deliver_template`; a
  free-form message with the window open dispatches `deliver_text`/
  `deliver_interactive`; a free-form message with the window closed dispatches
  `deliver_template` with `rota_saude_resume` (assert the substituted name).

## Out of scope / follow-ups

- Anti-spam dedup of the resume template (per conversation/day) once an
  out-of-band sender exists (F-02.7 abandoned sweep).
- Centralizing the template-payload shape shared with `Triage#template` (left
  as-is; not refactored here).
- Preserving/resending the dropped free-form content after re-engagement.

## Workflow

Card F-01.7 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
