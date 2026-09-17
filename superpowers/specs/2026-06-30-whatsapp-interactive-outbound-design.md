# F-03.3 — Map answer_type to WhatsApp element (interactive outbound)

**Date:** 2026-06-30
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03
**Board:** F-03.3 (In Progress)
**Touches:** `apps/api` only.

## Problem

The citizen WhatsApp runtime is the thinnest part of the product: `Whatsapp::Outbound`
only implements `deliver_text`, so every triage question is sent as plain text. A
boolean step ("Você está com tosse?") or an `enum` step is sent as text and the citizen
must type a free-text answer, which then has to match the protocol's branch keys exactly.
F-03.3 makes the conversation render each step's `answer_type` as the right WhatsApp
interactive element (buttons / list / text), and is the foundation for F-02.4 (consent
buttons) and F-01.7 (templates) — the rest of the "citizen runtime" slice.

## Current architecture (verified)

The reply path is string-based end to end:
- `ConversationAdvance#call` returns `Result.new(reply: <String>)` (e.g.
  `t(:triage_next, prompt: step_prompt(triage, outcome.awaiting))`).
- `ProcessInboundMessageJob` does `SendWhatsappJob.perform_later(to:, body: result.reply, …)`
  when `result.reply.present?`.
- `SendWhatsappJob#perform(to:, body:, municipality_id:, dedup_key:)` inserts an
  `OutboundMessage` (idempotency key = SHA256 of `to|body|muni` or an explicit dedup_key),
  then calls `Whatsapp::Outbound#deliver_text(to:, body:)`.
- Inbound: `Whatsapp::Ingest::Parser#extract_body` already surfaces an interactive reply's
  `interactive.button_reply.id` / `interactive.list_reply.id` as the message `body`.

So the inbound side already understands interactive replies; only the outbound side and the
reply representation need to change.

## Decisions (from brainstorming)

1. **Full chain**: triage-step questions actually render as interactive elements — not just
   a mapper. `Result#reply` becomes a structured value object; `Outbound` gains interactive
   send; the change threads through `ConversationAdvance` and `SendWhatsappJob`.
2. **Mapping rules** (per WhatsApp Cloud constraints): boolean → 2 reply buttons;
   `enum` ≤3 options → reply buttons; `enum` 4–10 options → list; `enum` >10 / `integer` /
   `text` → plain text (free-text prompt).
3. **The button/row `id` is the protocol answer value** — so a tap returns exactly what the
   engine's `branches`/`weights` expect, via the existing inbound parser. No new inbound code.
4. Sub-decisions: a `Messaging::Reply` value object; boolean labels "Sim"/"Não" via i18n;
   an enum option string is both the `id` and the `title` (title truncated to WhatsApp's
   limit, id kept whole). F-02.4 and F-01.7 are out of scope (next cards in the slice).

## Design

### 1. `Messaging::Reply` value object (new)

An immutable VO carrying what to send. Fields: `kind` (`:text | :buttons | :list`), `body`
(String), and `options` (Array of `{ id:, title: }` for buttons/list; empty for text).
Constructors: `Messaging::Reply.text(body)`, `.buttons(body:, options:)`, `.list(body:,
options:)`. `#to_h` serializes for job args (`{ kind:, body:, options: }`) and `.from_h`
rebuilds it in the job. (Named `Messaging::Reply` to avoid colliding with the `OutboundMessage`
AR model.)

### 2. `Whatsapp::QuestionElement` mapper (new, pure — the core of F-03.3)

`Whatsapp::QuestionElement.for(step, body:) -> Messaging::Reply`, where `step` is a
`Protocols::Step` (has `answer_type`, `options`). Rules:
- `:boolean` → `Reply.buttons(body:, options: [{id:"true", title: t(:btn_yes)}, {id:"false",
  title: t(:btn_no)}])` (labels via i18n, pt_BR "Sim"/"Não").
- `:enum`, `options.size <= 3` → `Reply.buttons(body:, options: options.map { |o| {id: o,
  title: truncate(o, 20)} })`.
- `:enum`, `4..10` → `Reply.list(body:, options: options.map { |o| {id: o, title:
  truncate(o, 24)} })`.
- `:enum` with >10 options, or `:integer`, or `:text` → `Reply.text(body)`.
- Title truncation trims to WhatsApp's limit (buttons 20, list rows 24) with an ellipsis;
  the `id` is always the full option/value.

### 3. `Whatsapp::Outbound#deliver_interactive` (new)

`deliver_interactive(to:, reply:)` builds the Graph payload from a `Messaging::Reply`:
- `:buttons` → `type:"interactive", interactive:{ type:"button", body:{text: reply.body},
  action:{ buttons: reply.options.map { |o| {type:"reply", reply:{id: o[:id], title:
  o[:title]}} } } }`.
- `:list` → `type:"interactive", interactive:{ type:"list", body:{text: reply.body},
  action:{ button: t(:list_button), sections:[{ rows: reply.options.map { |o| {id: o[:id],
  title: o[:title]} } }] } }`.
`deliver_text` is unchanged. Same `Result` struct (status, body) return.

### 4. Threading the reply

- **`ConversationAdvance`**: the "ask a step" branches (`triage_next`, `triage_start`)
  build a `Messaging::Reply` from the awaiting/current `Step` via `QuestionElement.for(step,
  body: <localized prompt>)`. All other branches (greeting, consent_prompt, errors, etc.)
  return `Messaging::Reply.text(<string>)`. `Result#reply` is now always a `Messaging::Reply`
  (or `nil`). `result.reply.present?` becomes a `nil` check.
  - The localized prompt body keeps the existing i18n framing (`triage_next`/`triage_start`
    templates) as the `body`; the options become the interactive element.
- **`ProcessInboundMessageJob`**: `SendWhatsappJob.perform_later(to:, message: result.reply.to_h,
  municipality_id:)` when `result.reply` is non-nil.
- **`SendWhatsappJob`**: signature becomes `perform(to:, message:, municipality_id:,
  dedup_key: nil)` where `message` is the serialized `Reply` hash. Rebuild with
  `Messaging::Reply.from_h(message)`; idempotency key = SHA256 of `dedup_key` or
  `to | message.to_json | municipality_id`; `OutboundMessage.template` stores `message`;
  dispatch: `kind == :text` → `deliver_text(to:, body: reply.body)`, else
  `deliver_interactive(to:, reply:)`.

### 5. Closed loop (no new inbound code)

The inbound parser already maps a tapped button/row to its `id` as the message `body`.
Because `id` = the protocol answer value (`"true"/"false"` for boolean — matching the
existing branch keys — and the option string for enum), `ConversationAdvance` feeds the
tap straight into the engine. A citizen on an older client who types instead of tapping
still sends free text, which the engine matches as before.

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

- **`Messaging::Reply`** spec: constructors set `kind`/`body`/`options`; `to_h`/`from_h`
  round-trip.
- **`Whatsapp::QuestionElement`** spec (pure, the core): boolean → 2 buttons with ids
  `true`/`false` and Sim/Não titles; enum with 3 options → buttons (id == option); enum with
  10 → list; enum with 11 → text; integer → text; text → text; a long option title is
  truncated while the id stays whole.
- **`Whatsapp::Outbound#deliver_interactive`** spec: builds the correct `button` and `list`
  Graph payloads (stub HTTP, assert the request body shape).
- **`ConversationAdvance`** spec: an enum step yields a `:buttons`/`:list` reply with the
  right option ids; a text/integer step and the non-step branches yield `:text`.
- **`SendWhatsappJob`** spec: dispatches `deliver_interactive` for an interactive message
  and `deliver_text` for text; idempotency dedup works over the serialized message.

## Out of scope / follow-ups

- F-02.4 (consent prompt as interactive buttons + `Consents.interpret` button payloads) and
  F-01.7 (approved template outside the 24h window) — next cards in the slice; both reuse
  `deliver_interactive` / the structured reply.
- Multi-section lists, button >3 pagination beyond the >10→text fallback.

## Workflow

Card F-03.3 In Progress. writing-plans → subagent-driven-development. Commits in English;
api on branch `fix/migrations-owner-ddl-as-admin`.
