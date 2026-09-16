# F-07.3 — Recurring purge of domain_events (12-month retention)

**Date:** 2026-07-01
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-07
**Board:** F-07.3 (In Progress)
**Touches:** `apps/api` only.

## Problem

`domain_events` is an append-only audit log that grows without bound —
every published event (triage.completed, consent.revoked, etc.) inserts a row
and nothing ever removes them. The other retention surfaces already have
recurring purges (`processed_events` 60d, `inbound.raw` 90d, expired reports
30d); `domain_events` has none. F-07.3 adds a recurring job that deletes
`domain_events` beyond a 12-month retention window, matching the established
housekeeping pattern.

## Current state (verified)

- `domain_events`: append-only, columns include `occurred_at` (event time,
  **indexed** — `index_domain_events_on_occurred_at`), `created_at`,
  `published_at` (nullable), `municipality_id` (nullable for platform-scope).
  No purge today.
- Sibling purge jobs (`PurgeProcessedEventsJob`, `PurgeInboundRawJob`,
  `PurgeExpiredReportsJob`): each `prepend AdminRoleJob` (cross-tenant,
  `rota_admin`/BYPASSRLS), `queue_as :housekeeping`, a `perform(older_than_days:)`
  that computes a cutoff, runs a bulk `delete_all`/`update_all`, and logs one
  line. Scheduled in `config/recurring.yml` under `default: &default`.
- `AdminRoleJob` must be used via `prepend` (not `include`) — documented in the
  concern; `include` silently skips the admin-connection wrap.
- `recurring.yml` housekeeping slots in use: 2am, 3am, 4am, 4:30am, 5am, 5:30am.
  4:45am is free (between `purge_processed_events` 4:30 and `reconcile_consents` 5am).

## Decisions (from brainstorming)

1. Single recurring job `PurgeDomainEventsJob`, mirroring `PurgeProcessedEventsJob`.
2. Cut by **`occurred_at`** — semantic event age and the only indexed timestamp
   (so `delete_all` uses the index).
3. Retention **12 months**, expressed as `older_than_months: 12` (`12.months.ago`)
   rather than the siblings' `older_than_days`, for calendar-correct "12 months";
   still parameterizable.
4. **Hard delete** (`delete_all`), matching `PurgeProcessedEventsJob` — audit rows
   are append-only within the retention window and removed after it. 12 months is
   the compliance window.
5. **Cross-tenant** via `prepend AdminRoleJob` (BYPASSRLS), like all purge jobs.
6. No migration; no domain event emitted (housekeeping, like the siblings).

## Design

### `PurgeDomainEventsJob`

`app/jobs/purge_domain_events_job.rb`:

```ruby
# Purga domain_events além da janela de retenção (auditoria: 12 meses).
# Ver ADR-0005/0014. delete_all cross-tenant sob rota_admin (BYPASSRLS).
class PurgeDomainEventsJob < ApplicationJob
  prepend AdminRoleJob
  queue_as :housekeeping

  def perform(older_than_months: 12)
    cutoff = older_than_months.months.ago
    count = DomainEvent.where("occurred_at < ?", cutoff).delete_all
    Rails.logger.info("[purge_domain_events] deleted=#{count} cutoff=#{cutoff.iso8601}")
  end
end
```

### Recurring schedule

`config/recurring.yml`, inside `default: &default`:

```yaml
  purge_domain_events:
    class: PurgeDomainEventsJob
    queue: housekeeping
    schedule: "every day at 4:45am America/Sao_Paulo"
    args: { older_than_months: 12 }
```

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

Job runs under `AdminRoleJob` (admin/BYPASSRLS). Use the admin-connection
harness (`use_transactional_tests = false` + `as_admin` + `clean_admin_tables`),
matching `spec/jobs/purge_*`/`sweep_*` specs. `domain_events` rows are created
via the admin connection; set `occurred_at` explicitly (e.g. `update_columns` or
a create with `occurred_at:`), since the default is `Time.current`.

- **deletes events older than the retention window**: an event with
  `occurred_at` 13 months ago is deleted by `perform(older_than_months: 12)`.
- **keeps events within the window**: an event with `occurred_at` 1 month ago is
  untouched.
- **boundary**: an event exactly at/just inside the cutoff is kept (strictly
  `< cutoff` is deleted).
- **custom window**: `perform(older_than_months: 1)` deletes an event 2 months
  old but keeps one 1 week old.
- **returns/logs a count** (optional): the deleted count reflects only the old
  rows.

(Create rows with explicit `occurred_at` via `as_admin`; a `DomainEvent.create!`
with `name`, `payload`, `municipality_id`, `occurred_at` is sufficient.)

## Out of scope / follow-ups

- Configurable per-tenant retention (single global 12-month window for MVP).
- Archiving events before deletion (cold storage) — hard delete for now.
- Purging by `published_at` or `created_at` — `occurred_at` is the chosen axis.

## Workflow

Card F-07.3 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
