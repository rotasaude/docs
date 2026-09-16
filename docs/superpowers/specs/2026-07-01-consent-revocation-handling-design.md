# F-07.15 — Consent revocation handling (trigger + consent.revoked subscribers)

**Date:** 2026-07-01
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-07 (touches mod-02 trigger + mod-03/04 projection)
**Board:** F-07.15 (In Progress)
**Touches:** `apps/api` only. Builds on F-02.2 (declined/cancelled), F-02.8 (atomic reply enqueue).

## Problem

`RevokeConsent` already does the synchronous side of revocation (revokes the
append-only consent, sets the conversation `revoked`, aborts the in-progress
triage as `aborted_by_revocation`, and publishes `consent.revoked`). But two
gaps leave the feature inert:

1. **No trigger.** After F-02.2 made the `awaiting_consent` refusal a
   `declined`, `RevokeConsent` has **no caller** — a consented citizen has no
   way to revoke mid-triage. `consent.revoked` therefore never fires.
2. **No handling.** `consent.revoked` is bound to `[]` in the registry
   (audit-only). Nothing reacts to a revocation.

F-07.15 adds the mid-triage revocation **trigger** (LGPD withdrawal, distinct
from cancel) and the **subscribers** that handle `consent.revoked`: anonymize
the citizen's in-flight clinical data and surface the revocation to the city.

## Current state (verified)

- `RevokeConsent#call`: `active = conversation.active_consent`; returns
  `fail(:no_active_consent)` if nil; else in a transaction `active.revoke!`,
  `conversation.update!(state: :revoked)`, aborts the in-progress triage
  (`status: :aborted_by_revocation, completed_at:`), and
  `DomainEvents.publish("consent.revoked", conversation_id:, consent_id:, reason:)`.
- `config/initializers/domain_events.rb`: `DomainEvents.bind "consent.revoked", to: []`
  (explicitly audit-only today).
- `ConversationAdvance#handle_consented` (F-02.2): checks `Consents.cancel?`
  first (`sair/parar/cancelar/encerrar`, NOT `não`), then runs the triage.
  `Consents` also has `interpret` (`:revoke` for `não/sair/parar/cancelar/revogar`
  or the revoke button) used at `awaiting_consent`.
- `IdempotentConsumer` concern: `include`s `TenantScopedJob`; provides
  `perform(event_id:, event_name:, municipality_id:, payload:)` → `with_tenant`
  → inserts a `ProcessedEvent` (exactly-once per consumer; `RecordNotUnique` →
  log+skip) → calls `handle(**payload.symbolize_keys)`. Subscribers implement
  `handle`. Out-of-band effects (HTTP/email) go to a dedicated job.
- `DomainEvents.publish` enqueues each bound consumer with
  `perform_later(event_id:, event_name:, municipality_id:, payload:)`.
- `Triage` clinical columns: `answers` (jsonb), `outcome` (jsonb), `tier`,
  `priority`, `current_step`. Audit columns kept: `status`, `completed_at`,
  `protocol_name`, timestamps.
- `DashboardMetric.bump!(municipality_id:, dimension:, period:, key:)` — the
  incremental projection primitive (`UpdateDashboardJob` uses it, `queue_as :reports`).
- i18n `conversation_advance.consent_revoked` already exists (orphaned after
  F-02.2) — reused here.

## Decisions (from brainstorming)

1. Handling = **all three**, resolved by architecture:
   - **Confirmation to the citizen** → **inline** reply from the trigger (atomic
     with the state change now that F-02.8 makes the reply enqueue transactional),
     NOT a subscriber.
   - **LGPD anonymization** → async subscriber `AnonymizeRevokedTriageJob`.
   - **City notification** → **lightweight** (audit/visibility): async subscriber
     `RecordConsentRevocationJob` bumping a `consents_revoked` metric. No email/push.
2. Mid-triage revocation trigger uses a dedicated `Consents.revoke_intent?`
   (`revogar`/`revogar consentimento`/`apagar meus dados`), distinct from
   `cancel?` and from `não` (a valid answer). Checked before `cancel?`.
3. Anonymization scrubs only the `aborted_by_revocation` triage of the
   conversation; append-only audit (consent row, event, triage shell) is kept.
4. No new migration (reuses existing columns/tables).

## Design

### 1. Trigger — `Consents.revoke_intent?` + `ConversationAdvance`

`app/services/consents.rb`:
```ruby
  # Intenção explícita de REVOGAR consentimento no meio da triagem (LGPD).
  # Distinta de cancel? (sair/parar/cancelar/encerrar) e de "não" (resposta).
  REVOKE_INTENT_PATTERNS = [
    /\A\s*(revogar|revogar consentimento|revogar meu consentimento|apagar meus dados)\s*\z/i
  ].freeze

  def self.revoke_intent?(text)
    return false if text.nil?
    REVOKE_INTENT_PATTERNS.any? { |re| text.match?(re) }
  end
```

`app/commands/conversation_advance.rb` `handle_consented`, at the top:
```ruby
  def handle_consented
    return revoke_and_finish if Consents.revoke_intent?(text)
    return cancel_and_finish if Consents.cancel?(text)
    # ... existing triage flow ...
  end
```
with:
```ruby
  def revoke_and_finish
    RevokeConsent.call(conversation: @conversation, reason: text)
    Result.new(reply: Messaging::Reply.text(t(:consent_revoked)))
  end
```
`RevokeConsent` does the sync work and publishes `consent.revoked`. If there is
no active consent (edge), `RevokeConsent` no-ops and we still reply
`consent_revoked` (a harmless confirmation). The conversation becomes `revoked`
(terminal, outside the active index) so a later message re-onboards (F-02.8).

Semantics: at `awaiting_consent`, "revogar" → `interpret` → `:revoke` →
`declined` (F-02.2, pre-consent refusal); at `consented`, "revogar" →
`revoke_intent?` → actual LGPD revocation. Consistent and intentional.

### 2. Subscriber — `AnonymizeRevokedTriageJob`

`app/jobs/anonymize_revoked_triage_job.rb`:
```ruby
class AnonymizeRevokedTriageJob < ApplicationJob
  include IdempotentConsumer
  queue_as :housekeeping

  # LGPD: apaga o conteúdo clínico da triage abortada por revogação, mantendo
  # a casca de auditoria (status, timestamps, protocol_name). Não deleta a linha.
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
- Runs under `with_tenant` (RLS scopes to the city). `update_columns` skips
  callbacks/validations (bulk housekeeping, matching the sweep/purge jobs).
- Idempotent: re-scrubbing already-empty fields is a no-op; `ProcessedEvent`
  also dedups the event per consumer.
- Scope: only the conversation's `aborted_by_revocation` triage — completed
  triages live on terminal `completed` conversations with immutable snapshots,
  never reachable by a `consented`-state revocation.

### 3. Subscriber — `RecordConsentRevocationJob`

`app/jobs/record_consent_revocation_job.rb`:
```ruby
class RecordConsentRevocationJob < ApplicationJob
  include IdempotentConsumer
  queue_as :reports

  # Visibilidade leve: incrementa a métrica de revogações para o painel de
  # consentimento da cidade. Auditoria bruta já está em domain_events.
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
- `Current.municipality_id` is set by `IdempotentConsumer#perform`'s
  `with_tenant`. Idempotent via `ProcessedEvent` (a re-delivered event does not
  double-count).

### 4. Registry wiring

`config/initializers/domain_events.rb`:
```ruby
  DomainEvents.bind "consent.revoked", to: [AnonymizeRevokedTriageJob, RecordConsentRevocationJob]
```
(replacing `to: []`).

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

- **`Consents.revoke_intent?`**: `revogar` / `revogar consentimento` /
  `apagar meus dados` → `true`; `cancelar`/`sair`/`parar`/`encerrar` → `false`
  (those are cancel); `não`/`sim`/ordinary/blank/nil → `false`.
- **`ConversationAdvance` (consented + "revogar")**: `RevokeConsent` is invoked;
  conversation → `revoked`; the in-progress triage → `aborted_by_revocation`;
  reply body == `consent_revoked`; a `consent.revoked` DomainEvent is published;
  it does NOT take the cancel path (state is not `cancelled`). Use the existing
  consented-block harness (real protocol + consent).
- **`ConversationAdvance` precedence**: "revogar" is handled as revoke, not
  cancel; "cancelar" still → cancel (`cancelled`); "não" still → triage answer.
- **`AnonymizeRevokedTriageJob`**: given an `aborted_by_revocation` triage with
  `answers`/`outcome`/`tier`/`priority`/`current_step` set, `handle` nulls/empties
  them and leaves `status`/`protocol_name`/timestamps; a second run is a no-op
  (idempotent); it does NOT touch a `completed` triage or another conversation's
  triage. (Admin-connection job harness: `use_transactional_tests = false` +
  `as_admin` + `clean_admin_tables`, per the sweep/PIMJ specs.)
- **`RecordConsentRevocationJob`**: `handle` bumps `DashboardMetric`
  `consents_revoked/total` for the day; delivering the same `event_id` twice
  increments once (ProcessedEvent dedup) — assert via
  `perform(event_id:, event_name: "consent.revoked", municipality_id:, payload:)`.
- **Registry**: `DomainEvents.registry["consent.revoked"]` contains both
  `AnonymizeRevokedTriageJob` and `RecordConsentRevocationJob`.

## Out of scope / follow-ups

- Active push to the city (email/WhatsApp to `alert_recipients`) — deferred; the
  domain event + dashboard panel + the new metric cover visibility.
- A dashboard panel widget rendering the `consents_revoked` metric (the metric is
  produced now; wiring it into the consent panel UI is a dashboard-app card).
- Erasure of the citizen's phone/contact info — LGPD withdrawal stops clinical
  processing; contact identity is retained for re-onboarding.
- A recurring re-anonymization sweep (the synchronous subscriber suffices).

## Workflow

Card F-07.15 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
