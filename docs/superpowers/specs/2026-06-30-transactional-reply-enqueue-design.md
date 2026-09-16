# F-02.8 — Transactional reply enqueue + idempotent inbound processing

**Date:** 2026-06-30
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-02
**Board:** F-02.8 (In Progress)
**Touches:** `apps/api` only.

## Problem

`ProcessInboundMessageJob` advances the conversation under `conversation.with_lock`
and then enqueues the outbound reply with `SendWhatsappJob.perform_later`. The
global setting `config.active_job.enqueue_after_transaction_commit = :always`
(`config/application.rb:38`) defers every enqueue until **after** the lock
transaction commits. This opens two reliability gaps:

1. **Lost reply.** Sequence: open transaction T → advance state (e.g.
   `greeting → awaiting_consent`, or append a triage answer) → T commits (state
   persisted) → *then* the deferred `perform_later` fires. If the process dies
   between commit and the deferred enqueue, the state moved forward but the
   reply was **never enqueued**. The citizen is left hanging.

2. **Double-advance on reprocess.** `ProcessInboundMessageJob` is not
   idempotent. If the worker re-runs the job after the transaction already
   committed (crash before the job is marked finished), `ConversationAdvance`
   runs again on the **already-advanced** state — re-recording a triage answer
   on the next step, or mis-interpreting the inbound against the new state.

F-02.8 makes the reply enqueue **atomic** with the state change and makes
inbound processing **idempotent**, so a committed state change always has its
reply enqueued exactly once and a retry is a safe no-op.

## Current state (verified)

- `config/application.rb:38`: `config.active_job.enqueue_after_transaction_commit = :always`;
  `config.active_job.queue_adapter = :solid_queue` (`:test` in the test env).
  Solid Queue is backed by the same Postgres database, so an enqueue performed
  inside an open transaction is a plain row INSERT that commits/rolls back with it.
- `app/jobs/process_inbound_message_job.rb`: `with_tenant` → `Conversation.for`
  → `conversation.with_lock { result = ConversationAdvance.call(...); SendWhatsappJob.perform_later(... result.reply.to_h ...) if result&.reply }`.
- `app/jobs/send_whatsapp_job.rb`: takes serialized primitives (`to:`,
  `message:` hash, `municipality_id:`, optional `dedup_key:`) — **no
  ActiveRecord argument**. Already idempotent against worker crash-retry via a
  UNIQUE `idempotency_key` on `outbound_messages` (INSERT before HTTP;
  `RecordNotUnique`/`RecordInvalid :taken` → skip). So a transactional enqueue
  introduces no duplicate-send risk.
- `activejob 8.1.3`: `enqueue_after_transaction_commit` is a `class_attribute`
  (`active_job/enqueuing.rb:53`), so a **per-job override** is supported and does
  not affect the global default.
- `InboundMessage` (`inbound_messages`, owner `rota_admin`) has **no** processed
  marker — only `message_id` (UNIQUE, ingestion-time dedup). Tenant table; DDL on
  it uses the raw-SQL `as_admin` pattern (mirror
  `20260630194633_allow_aborted_by_timeout_triage_status` / the F-02.2 migration).
- `ApplicationJob`: `retry_on ActiveRecord::Deadlocked, attempts: 3`.

## Decisions (from brainstorming)

1. **Scope = atomic enqueue + reprocess idempotency** (both gaps, per user).
2. **Atomic enqueue** via a per-job override `SendWhatsappJob.enqueue_after_transaction_commit = false`
   — scoped, leaves the global `:always` untouched. Chosen over flipping the
   global (changes all jobs) and over a bespoke transactional-outbox table (the
   Solid Queue row already *is* the in-transaction outbox; YAGNI).
3. **Idempotency** via a new `inbound_messages.processed_at` column, set inside
   the lock transaction; a `processed_at?` guard short-circuits a re-run. Chosen
   over a dedicated guard table (heavier).
4. No domain event; no change to `SendWhatsappJob`'s send/idempotency logic.

## Design

### 1. Migration — `inbound_messages.processed_at`

`inbound_messages` is owned by `rota_admin`, so add the column via the raw-SQL
`as_admin` PG.connect helper (same private-method pattern as the F-02.2 / F-02.7
migrations), reversible `down`:

```ruby
def up
  as_admin do |c|
    c.exec("ALTER TABLE inbound_messages ADD COLUMN processed_at timestamptz")
    c.exec("CREATE INDEX idx_inbound_messages_unprocessed ON inbound_messages (created_at) WHERE processed_at IS NULL")
  end
end

def down
  as_admin do |c|
    c.exec("DROP INDEX IF EXISTS idx_inbound_messages_unprocessed")
    c.exec("ALTER TABLE inbound_messages DROP COLUMN processed_at")
  end
end
```

(The partial index on un-processed rows is cheap and supports a future
"reprocess pending inbound" sweep; it stays tiny because rows are marked
processed almost immediately.) Regenerates `db/schema.rb` plus the three
multi-DB dumps. No model enum/validation change needed; `processed_at` is a
plain nullable timestamp.

### 2. `SendWhatsappJob` — transactional enqueue

Add the per-job override at the top of the class, with a comment explaining
why this job opts out of the global deferral:

```ruby
class SendWhatsappJob < ApplicationJob
  include TenantScopedJob

  # Enfileira DENTRO da transação aberta pelo caller (o with_lock do
  # ProcessInboundMessageJob), em vez de adiar para after_commit (global
  # :always). Seguro: os args são primitivos (sem registro AR não-commitado) e
  # o Solid Queue é o mesmo Postgres, então a linha do job comita atomicamente
  # com o avanço de estado. Já é idempotente via idempotency_key. (F-02.8)
  self.enqueue_after_transaction_commit = false
  ...
```

When `SendWhatsappJob` is enqueued outside any transaction (no current caller,
but defensively), `false` just means "enqueue immediately" — no behavior change.

### 3. `ProcessInboundMessageJob` — atomic, idempotent unit

```ruby
def perform(inbound_message_id, municipality_id:)
  with_tenant(municipality_id) do
    inbound = InboundMessage.find(inbound_message_id)
    conversation = Conversation.for(inbound.from, municipality_id: municipality_id)
    conversation.with_lock do
      next if inbound.reload.processed_at?

      result = ConversationAdvance.call(conversation: conversation, inbound: inbound)
      if result&.reply
        SendWhatsappJob.perform_later(
          to: inbound.from, message: result.reply.to_h, municipality_id: municipality_id
        )
      end
      inbound.update!(processed_at: Time.current)
    end
  end
end
```

All three writes — the `ConversationAdvance` state change, the `SendWhatsappJob`
row, and `inbound.processed_at` — commit in the single `with_lock` transaction.
The `defined?(ConversationAdvance)` guard from the current code is dropped (the
class is always loaded).

### Atomicity matrix

| Failure | Outcome |
|---|---|
| Crash **before** commit | Nothing persisted → Solid Queue retries the job → clean re-run |
| Crash **after** commit | State + reply-job row + `processed_at` all persisted → retry sees `processed_at?` → **skip** (no double-advance); reply already enqueued |
| Transaction rolls back (e.g. `ConversationAdvance` raises) | No reply enqueued, `processed_at` stays nil → job fails → retried |

### Error handling

A raise inside `with_lock` rolls back the transaction (state, reply row, and
`processed_at` all revert) and propagates so the job fails and Solid Queue
retries it. `ApplicationJob`'s existing `retry_on ActiveRecord::Deadlocked`
covers lock contention.

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

Note: the test env uses the `:test` queue adapter; `enqueue_after_transaction_commit`
deferral runs on AR transaction callbacks, which still fire under transactional
specs. Prefer behavioral assertions over adapter internals.

- **Happy path**: a fresh inbound (e.g. `greeting`, body "oi") → `SendWhatsappJob`
  is enqueued once with the expected `to`/`message`, and `inbound.processed_at`
  becomes present.
- **Idempotency**: running `perform` twice for the same inbound advances the
  conversation **once** and enqueues `SendWhatsappJob` **once** — the second call
  is a no-op (state unchanged, `processed_at` unchanged, no second enqueue).
  (Assert via `have_enqueued_job(SendWhatsappJob).once` across both calls, or by
  spying that `ConversationAdvance` is not called the second time.)
- **Rollback atomicity**: stub `ConversationAdvance.call` to raise; `perform`
  raises, no `SendWhatsappJob` is enqueued, and `inbound.reload.processed_at` is
  nil. (Optionally: stub it to raise *after* a state write to prove the write
  reverts.)
- **Config override**: `expect(SendWhatsappJob.enqueue_after_transaction_commit).to be(false)`.
- **No-reply inbound**: an inbound that yields `result.reply == nil` (e.g. a
  `revoked` conversation) still marks `processed_at` and enqueues nothing.
- **Migration**: an `InboundMessage` accepts and round-trips `processed_at`
  (covered implicitly by the job specs; a focused model example is optional).

## Out of scope / follow-ups

- A recurring "reprocess pending inbound" sweep over the partial index (the index
  is added now to make that cheap later, but no sweeper is built).
- Changing `SendWhatsappJob`'s send path, session-window logic, or idempotency.
- Revisiting the global `enqueue_after_transaction_commit` for other jobs.

## Workflow

Card F-02.8 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
