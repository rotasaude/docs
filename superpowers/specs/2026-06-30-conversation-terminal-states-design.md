# F-02.2 — Conversation terminal states (completed / declined / cancelled)

**Date:** 2026-06-30
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-02
**Board:** F-02.2 (In Progress)
**Touches:** `apps/api` only. Builds on F-02.7 (`abandoned` already exists).

## Problem

The conversation state machine never reaches three of its terminal states:
- **completed**: when a triage finishes, the conversation stays `consented` —
  `CompleteTriage` aborts/completes the *triage* and emits `triage.completed`, but
  the *conversation* is never marked done.
- **declined**: refusing consent at `awaiting_consent` calls `RevokeConsent`, which
  fails `:no_active_consent` (no consent exists yet) and changes nothing — the
  conversation stays `awaiting_consent` while replying "consent revoked" (a bug).
- **cancelled**: there is no detection of an explicit stop ("cancelar"/"parar")
  mid-triage; such text is fed to the triage engine as an answer.

F-02.2 adds these terminal states and their triggers, completing the lifecycle
(`abandoned` was added by F-02.7).

## Current state (verified)

- `Conversation.state` enum: greeting / awaiting_consent / consented / revoked /
  abandoned. Plain string, **no DB check constraint**; the partial unique index
  `idx_conversations_active_per_tenant_phone` covers only greeting/awaiting_consent/
  consented — so any new terminal state is outside it (a fresh conversation can
  start for the phone), like `revoked`/`abandoned`.
- `Triage.status` enum (after F-02.7): in_progress / completed / aborted_by_revocation
  / aborted_by_timeout, with the DB check constraint `ck_triagens_status` listing
  those four. A new status needs a migration (admin-owned table, raw-SQL `as_admin`
  pattern — see `20260630194633_allow_aborted_by_timeout_triage_status`).
- `ConversationAdvance#handle_consented`: terminal outcome → `Result.new(reply: nil)`
  (conversation unchanged); otherwise asks the next step.
- `ConversationAdvance#handle_awaiting_consent` `:revoke` branch → `RevokeConsent.call`
  (a no-op here) + reply `consent_revoked`.
- `Consents.interpret` maps text/button-ids to `:give`/`:revoke`/`:unknown`;
  `REVOKE_PATTERNS` includes `não/sair/parar/cancelar/revogar`. **`"não"` is also a
  valid boolean triage answer**, so mid-triage cancel detection must NOT use the
  revoke patterns — it uses explicit stop words only.

## Decisions (from brainstorming)

1. All three terminal states (completed/declined/cancelled).
2. `cancelled` mid-triage uses dedicated `CANCEL_PATTERNS` =
   `sair/parar/cancelar/encerrar` — explicitly **excluding `não`** (a valid answer).
3. `cancelled` aborts the in-progress triage with a new `aborted_by_cancellation`
   status (migration). `cancelled` ≠ `revoked`: cancelling stops the triage without
   an LGPD data withdrawal; explicit revocation keeps its own (currently
   trigger-less) path.
4. No domain events in this MVP (state + reply only).

## Design

### 1. States

- `app/models/conversation.rb`: add `completed`, `declined`, `cancelled` to the
  `state` enum (prefix: true). No migration (string column, terminal → outside the
  active index).
- Migration `allow_aborted_by_cancellation_triage_status`: extend `ck_triagens_status`
  on `triages` to also allow `aborted_by_cancellation`, mirroring the
  `aborted_by_timeout` migration (raw SQL via the private `as_admin` PG.connect as
  `rota_admin`, reversible `down`). Add `aborted_by_cancellation` to the `Triage`
  status enum.

### 2. `completed`

In `ConversationAdvance#handle_consented`, the terminal branch becomes:

```ruby
    return complete_and_finish if outcome.terminal?
```

with a private helper:

```ruby
  def complete_and_finish
    @conversation.update!(state: :completed)
    Result.new(reply: nil)
  end
```

The snapshot link still ships via `NotifyCitizenJob` (the `triage.completed` event).

### 3. `declined`

In `ConversationAdvance#handle_awaiting_consent`, the `:revoke` branch becomes:

```ruby
    when :revoke
      @conversation.update!(state: :declined)
      Result.new(reply: Messaging::Reply.text(t(:consent_declined)))
```

(The `RevokeConsent.call` is removed here — there is no active consent at
`awaiting_consent`, so it was a no-op.)

### 4. `cancelled`

- `app/services/consents.rb`: add
  ```ruby
  CANCEL_PATTERNS = [/\A\s*(sair|parar|cancelar|encerrar)\s*\z/i].freeze
  def self.cancel?(text)
    return false if text.nil?
    CANCEL_PATTERNS.any? { |re| text.match?(re) }
  end
  ```
- `ConversationAdvance#handle_consented`, at the very top (before resolving/advancing
  the triage):
  ```ruby
    return cancel_and_finish if Consents.cancel?(text)
  ```
  with:
  ```ruby
    def cancel_and_finish
      @conversation.triages.where(status: :in_progress)
                   .order(created_at: :desc).first&.update!(status: :aborted_by_cancellation, completed_at: Time.current)
      @conversation.update!(state: :cancelled)
      Result.new(reply: Messaging::Reply.text(t(:triage_cancelled)))
    end
  ```
  A `"não"` (boolean answer) does NOT match `CANCEL_PATTERNS`, so it flows to the
  triage engine as the answer — unchanged.

### 5. i18n (`config/locales/conversation_advance.pt-BR.yml`)

```yaml
    consent_declined: "Tudo bem, não vamos seguir. Envie \"olá\" se mudar de ideia."
    triage_cancelled: "Triagem cancelada. Envie \"olá\" para recomeçar."
```

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

- **Migration**: creating a `Triage` with `status: "aborted_by_cancellation"` does not
  raise (focused model example, like the F-02.7 status test).
- **`Consents.cancel?`**: `sair`/`parar`/`cancelar`/`encerrar` → `true`; `"não"`,
  `"sim"`, an ordinary word, blank/nil → `false`.
- **`ConversationAdvance`** (existing tenant-transaction spec pattern):
  - a terminal triage outcome → conversation `completed`.
  - refusing consent at `awaiting_consent` → conversation `declined` + the
    `consent_declined` reply (and it does NOT stay `awaiting_consent`).
  - `"cancelar"` while consented mid-triage → conversation `cancelled`, the in-progress
    triage `aborted_by_cancellation`, and the `triage_cancelled` reply.
  - `"não"` while consented (a valid boolean answer) → does NOT cancel; the triage
    advances/records the answer as before.

## Out of scope / follow-ups

- Domain events `conversation.declined` / `conversation.cancelled` (audit).
- An explicit LGPD-revocation trigger mid-triage (semantically distinct from cancel;
  keeps the existing `revoked` state + `RevokeConsent`/`consent.revoked`).

## Workflow

Card F-02.2 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
