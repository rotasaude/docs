# F-02.4 — Consent capture via interactive buttons

**Date:** 2026-06-30
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-02
**Board:** F-02.4 (In Progress)
**Touches:** `apps/api` only. Builds on F-03.3 (interactive outbound).

## Problem

The consent step asks "Você concorda? (sim/não)" as plain text; the citizen must
type a free-text answer that `Consents.interpret` matches with regex. Now that
F-03.3 gives us interactive buttons (`Messaging::Reply.buttons` +
`Whatsapp::Outbound#deliver_interactive`), the consent ask should render as two
buttons (Sim / Não) and a tap should be recognized deterministically.

## Current flow (verified)

- `ConversationAdvance#handle_greeting`: sets state `awaiting_consent`, replies
  `Messaging::Reply.text(t(:greeting))` ("…concorda? (sim/não)").
- `ConversationAdvance#handle_awaiting_consent`: `case Consents.interpret(text)` →
  `:give` (GiveConsent + begin triage), `:revoke` (RevokeConsent), `:unknown`
  (re-prompt `Messaging::Reply.text(t(:consent_prompt))`).
- `Consents.interpret(text)`: regex — `GIVE_PATTERNS` (sim/aceito/concordo/ok/de
  acordo) → `:give`; `REVOKE_PATTERNS` (não/sair/parar/cancelar/revogar) →
  `:revoke`; else `:unknown`.
- The inbound parser already surfaces `interactive.button_reply.id` as the
  message body, so a tapped button's id flows into `interpret` as `text`.

## Decisions (from brainstorming)

1. **Dedicated payload ids + explicit match in `interpret`** (not reusing the
   free-text regex): `Consents::GIVE_ID = "consent_give"`, `REVOKE_ID =
   "consent_revoke"`. `interpret` matches these exactly before the regex
   fallback. Deterministic for taps; typed free text still works.
2. The first ask (`handle_greeting`) and the `:unknown` re-prompt render as
   buttons; `consent_revoked`/`consent_failed` stay text.
3. Button labels reuse the F-03.3 locale (`whatsapp.btn_yes`/`btn_no` = "Sim"/
   "Não"); the reply builder lives in `ConversationAdvance` and references the
   `Consents` id constants (single source).

## Design

### 1. `Consents` payload constants + `interpret`

```ruby
GIVE_ID   = "consent_give".freeze
REVOKE_ID = "consent_revoke".freeze

def self.interpret(text)
  return :unknown if text.nil? || text.strip.empty?
  return :give    if text == GIVE_ID
  return :revoke  if text == REVOKE_ID
  return :revoke  if REVOKE_PATTERNS.any? { |re| text.match?(re) }
  return :give    if GIVE_PATTERNS.any?  { |re| text.match?(re) }
  :unknown
end
```

The id checks run before the regex so a button tap is unambiguous and
independent of future regex changes. (Existing precedence — revoke before give,
the "caution bias" noted in the file — is preserved for the regex fallback.)

### 2. Consent reply builder in `ConversationAdvance`

A private helper builds the two-button consent ask:

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

- `handle_greeting` → `Result.new(reply: consent_reply(t(:greeting)))` (state still
  moves to `awaiting_consent`).
- `handle_awaiting_consent` `:unknown` branch → `Result.new(reply:
  consent_reply(t(:consent_prompt)))`.
- `:give`/`:revoke` branches unchanged except `consent_revoked`/`consent_failed`
  stay `Messaging::Reply.text` (already are).

### 3. Closed loop (reuses F-03.3)

`deliver_interactive` sends the buttons; the inbound parser returns the tapped
`button_reply.id` (`"consent_give"`/`"consent_revoke"`) as the body;
`Consents.interpret` matches the id. A citizen who types "sim"/"não" instead
still works via the regex fallback — full backward compatibility.

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

- **`Consents.interpret`** spec: `"consent_give"` → `:give`; `"consent_revoke"` →
  `:revoke`; the existing free-text cases still pass (`"sim"` → `:give`, `"não"` →
  `:revoke`, `"qualquer coisa"` → `:unknown`, empty/nil → `:unknown`); a button-id
  match is not shadowed by the regex.
- **`ConversationAdvance`** spec: from `greeting`, the reply is a `:buttons`
  `Messaging::Reply` whose option ids are `["consent_give", "consent_revoke"]`
  and state becomes `awaiting_consent`; an inbound whose body is `"consent_give"`
  records consent and advances to the triage question; `"consent_revoke"` revokes;
  unrecognized text re-prompts with a `:buttons` reply (not plain text).

## Out of scope / follow-ups

- F-01.7 (approved template outside the 24h window) — next card in the slice.
- Richer consent button labels (e.g. "Sim, concordo") — labels reuse the generic
  yes/no for now.

## Workflow

Card F-02.4 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
