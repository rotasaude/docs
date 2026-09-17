# F-02.7 — Timeout / abandoned sweep

**Date:** 2026-06-30
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-02
**Board:** F-02.7 (In Progress)
**Touches:** `apps/api` only.

## Problem

A conversation that the citizen starts but never finishes hangs forever: there
is no terminal `abandoned` state and no sweep to reach it. `Conversation.state`
is greeting/awaiting_consent/consented/revoked; `Triage.status` is in_progress/
completed/aborted_by_revocation. F-02.7 adds a recurring sweep that marks idle,
non-terminal conversations `abandoned` (and aborts their in-progress triage), so
the conversation lifecycle is complete and stale rows stop counting as active.

The sweep is **silent** (no outbound) — re-engagement nudges are out of scope.

## Current state (verified)

- `Conversation.state` is a plain string (default "greeting"), **no DB check
  constraint** — only a partial unique index `idx_conversations_active_per_tenant_phone`
  scoped to `state IN (greeting, awaiting_consent, consented)`. Adding `abandoned`
  to the enum is code-only; an `abandoned` conversation falls outside the active
  index (like `revoked`).
- `Triage.status` has a DB check constraint `ck_triagens_status`
  (in_progress/completed/aborted_by_revocation) — a new status needs a migration.
- Housekeeping jobs (`PurgeInboundRawJob` etc.) use `prepend AdminRoleJob`
  (admin/BYPASSRLS connection) + a bulk operation + a log line; scheduled in
  `config/recurring.yml`, queue `housekeeping`.
- `RevokeConsent` is the precedent for the terminal transition: it sets
  `conversation.update!(state: :revoked)` and aborts the in-progress triage
  (`status: :aborted_by_revocation`).
- Activity signals: `triage.updated_at` moves on each `append_answer!`;
  `conversation.updated_at` moves on state transitions.

## Decisions (from brainstorming)

1. **Silent**: the sweep only marks state; it sends nothing.
2. **`abandoned`** terminal state added to `Conversation` (code-only) and an
   `aborted_by_timeout` status added to `Triage` (migration), parallel to
   `aborted_by_revocation`.
3. Idle signal = `conversation.updated_at` + `triage.updated_at` (avoids the
   encrypted-phone ↔ plaintext-inbound mismatch). Conversations that COMPLETED a
   triage (success) or have a fresh in-progress triage are excluded.
4. Sub-decisions: `idle_hours: 24`, daily schedule 2am, no domain event (bulk +
   log, matching the purge jobs).

## Design

### 1. `abandoned` / `aborted_by_timeout` states

- `app/models/conversation.rb`: add `abandoned: "abandoned"` to the `state` enum
  (prefix: true → `state_abandoned?`). No migration.
- Migration: extend `ck_triagens_status` to also allow `aborted_by_timeout`. Add
  `aborted_by_timeout: "aborted_by_timeout"` to the `Triage` `status` enum.

### 2. `SweepAbandonedConversationsJob`

`app/jobs/sweep_abandoned_conversations_job.rb`: `prepend AdminRoleJob`,
`queue_as :housekeeping`.

```ruby
def perform(idle_hours: 24)
  cutoff = idle_hours.hours.ago

  fresh_triage_convo_ids = Triage.where(status: :in_progress)
                                 .where("updated_at >= ?", cutoff).select(:conversation_id)
  completed_convo_ids    = Triage.where(status: :completed).select(:conversation_id)

  scope = Conversation
            .where(state: %w[greeting awaiting_consent consented])
            .where("updated_at < ?", cutoff)
            .where.not(id: fresh_triage_convo_ids)
            .where.not(id: completed_convo_ids)

  abandoned = 0
  scope.find_each do |conversation|
    conversation.triages.where(status: :in_progress).update_all(status: "aborted_by_timeout", updated_at: Time.current)
    conversation.update_columns(state: "abandoned", updated_at: Time.current)
    abandoned += 1
  end
  Rails.logger.info("[sweep_abandoned] abandoned=#{abandoned} cutoff=#{cutoff.iso8601}")
end
```

- Runs under the admin (BYPASSRLS) connection (`AdminRoleJob`), so it operates
  cross-tenant — the same model as the purge jobs.
- `update_columns` / `update_all` skip validations/callbacks (a terminal bulk
  housekeeping transition, like the purges) and avoid re-triggering the conversation
  state-machine. No domain event in the silent MVP.
- The two `where.not(id: …)` subqueries exclude conversations that already
  completed a triage (success) or have a fresh in-progress triage (active).

### 3. Recurring schedule

`config/recurring.yml` (inside `default: &default`):

```yaml
  sweep_abandoned:
    class: SweepAbandonedConversationsJob
    queue: housekeeping
    schedule: "every day at 2am America/Sao_Paulo"
    args: { idle_hours: 24 }
```

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

The job runs under `AdminRoleJob` (admin connection). The spec creates fixtures
via the admin connection (or the established `as_admin`/tenant pattern) and
invokes `SweepAbandonedConversationsJob.new.perform(idle_hours: 24)`.

- **abandons an idle awaiting_consent conversation** (no triage, `updated_at`
  older than the cutoff) → `state == "abandoned"`.
- **abandons a consented conversation with a stale in-progress triage**
  (triage.updated_at older than cutoff) → triage `status == "aborted_by_timeout"`
  and conversation `state == "abandoned"`.
- **leaves a recent conversation untouched** (`updated_at` within the cutoff) →
  unchanged.
- **leaves a conversation that completed a triage untouched** (has a completed
  triage, even with an old conversation.updated_at) → stays `consented`.
- **leaves an already-terminal conversation untouched** (`revoked`) → unchanged.
- **Migration**: creating a `Triage` with `status: "aborted_by_timeout"` does not
  raise the check-constraint violation (covered implicitly by the abort test;
  add a focused model example if helpful).

(Set the relevant `updated_at` timestamps explicitly in the fixtures, e.g.
`update_columns(updated_at: 30.hours.ago)`, since `idle_hours` defaults to 24.)

## Out of scope / follow-ups

- Re-engagement nudge (template send before abandoning) — the F-01.7 layer exists;
  deferred per the silent decision.
- Dashboard `abandonRate` metric (currently `nil`) — the sweep only sets state.
- The other F-02.2 terminal states (declined/cancelled).
- Emitting a `conversation.abandoned` domain event per row.

## Workflow

Card F-02.7 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
