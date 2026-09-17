# FEATURE — Dashboard demo seed (populates every dashboard view)

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Touches:** `apps/api` only (seed loader + rake task). No dashboard code changes.
**Repo/branch:** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` (commit there; do not branch).

## Problem

The dashboard has nine views across three groups — Aquisição (Ingestão,
Conversas, Consentimentos), Triagem (Triagens, Classificação, Relatórios),
Governança (Protocolos, Editor de Protocolos, Eventos e auditoria) — but the
dev database only holds the minimal Curitiba baseline (1 channel, 1 active
protocol, 1 conversation, 1 triage, 1 report). Most panels render empty, and
several sparklines/funnels/breakdowns never fill. We need a demo seed that
populates every view non-empty and representative, so the dashboard can be
demoed end-to-end.

## Decisions (from brainstorming)

1. **Two municipalities:** Curitiba (rich) + Londrina (~40% volume). Feeds the
   operator cross-tenant views (Cidades/Eventos) and city comparison too.
2. **Opt-in loader:** `db/seeds/dashboard_demo.rb` + rake task
   `bin/rails db:seed:demo`. The base `db/seeds.rb` stays lean and only invokes
   the demo when `SEED_DASHBOARD_DEMO=1` is set (default off).
3. **Rich data spread over the last 30 days** (daily buckets) so both the 7d
   and 30d period windows populate, and sparklines/funnels/breakdowns are alive.
4. Idempotent by deterministic natural keys; re-running is a no-op.

## Verified backing-data map (from exploration)

`Admin::Api::Period`: `today`→24 hourly buckets, `7d`→7 daily, `30d`→30 daily,
TZ America/São_Paulo; series returned as `Array<Integer>`. Municipal scoping via
`Admin::Scoped.*`. All writes under `ApplicationRecord.connected_to(role: :admin)`
(BYPASSRLS) because tenant tables are RLS-enforced and the seed runs without
`SET LOCAL`.

Per-view requirements (what makes each panel non-empty), with the query that
governs each:

- **Ingestão** (`ingestion_query.rb`): `InboundMessage` count + series by
  `created_at`; purge backlog needs ≥1 inbound `created_at > 24h` ago (overTtl)
  and ≥1 within 24h (pending); ack distribution buckets `OutboundMessage.status`
  as literal small ints (`ingestion_query.rb:37-42`): `values_at(0,1,2)`→ok,
  `values_at(3)`→warn, `values_at(4,5)`→err — status values are 0–5, NOT HTTP
  codes.
- **Conversas** (`conversations_query.rb`): funnel counts `state ∈ greeting|
  awaiting_consent|consented`; exits = `revoked`; live = `awaiting_consent +
  consented` (no time filter); avgToCompleteMin needs `Triage status=completed`
  with `completed_at` in period and `created_at < completed_at`.
- **Consentimentos** (`consent_query.rb`): `Consent` given (`given_at` in
  period, `revoked_at` NULL), revoked (`revoked_at` in period), grouped by
  `version`; revocations series by `revoked_at`.
- **Triagens** (`triages_query.rb`): `Triage` started (`created_at` in period),
  completed (`status=completed`), completion rate; volume series; pivot by
  `(protocol name, version, status)` via join to `protocol_definition`.
- **Classificação** (`classification_query.rb`): tier KPIs count
  `Triage.tier ∈ {low, medium, high}` with `status=completed`, `completed_at`
  in period; priority KPI/trend needs `priority > 0`; scoring-mode panel reads
  `outcome->'scoring'->>'mode' ∈ {weighted, decision_table}`; inspection sample;
  trail drawer reads `DomainEvent` names `{scored, rule_matched, priority_rule,
  tier_assigned}` with `triage_id` in payload.
- **Relatórios** (`reports_query.rb`): `ReportSnapshot` `created_at` in period;
  `live = expires_at IS NULL OR > now`; tier from `outcome.tier`.
- **Protocolos** (`protocols_query.rb`): all `ProtocolDefinition` rows;
  published count; four-eyes derived from `DomainEvent` `protocol.created` vs
  `protocol.published` actors (same actor → `fourEyes=false`); detail shows
  versions + `protocol.created|published|retired` events (actor, version in
  payload).
- **Editor de Protocolos**: read-only here — needs ≥1 published/active protocol
  to load/preview (covered by Protocolos).
- **Eventos e auditoria** (`events_query.rb`): `DomainEvent` count + count-by-
  type (top 24); stream (50 recent, allowlisted); filter prefixes `triage.*,
  consent.*, conversation.*, protocol.*, priority.*`; replay anchor = earliest
  event overall; stream `ref` extracted from first `*_id` key in payload,
  `actor` from `payload.actor`.

## Correctness notes (traps confirmed during exploration)

- **Two tier vocabularies.** Classificação buckets `Triage.tier` as
  **`low/medium/high`** (`classification_query.rb:6`, `TIERS = %w[low medium
  high]`); the current seed used `alta`, which would NOT count. The loader sets
  `Triage.tier ∈ {low, medium, high}`. Relatórios only displays
  `report.outcome["tier"]` as a free string (`reports_query.rb:26`), so the
  loader sets `outcome.tier` to the same `low/medium/high` for app-wide
  consistency. The existing Curitiba baseline triage's tier is reconciled to
  `low/medium/high` so it counts in Classificação.
- **Ack bucketing** (`ingestion_query.rb:37-42`): `OutboundMessage.status` is
  read as literal small ints — `values_at(0,1,2)`→ok, `values_at(3)`→warn,
  `values_at(4,5)`→err. Seed uses status values 0–5, NOT HTTP codes.
- **Consent version.** Use `Consents.current_version(municipality_id)` (or
  explicit `ConsentTerm` versions v1/v2) so grouping "Por versão" is meaningful.
- **Extend, don't duplicate.** Keep the existing Curitiba baseline block
  (1 conversation/triage/report); the demo loader adds records with distinct
  deterministic keys.

## Design

### Files

- `apps/api/db/seeds/dashboard_demo.rb` — the loader. Defines and runs the demo
  build for both cities, wrapped in `ApplicationRecord.connected_to(role: :admin)`.
- `apps/api/lib/tasks/seed_demo.rake` — `db:seed:demo` task that `load`s the
  loader within the Rails environment.
- `apps/api/db/seeds.rb` — add a guarded call: `load Rails.root.join("db/seeds/
  dashboard_demo.rb") if ENV["SEED_DASHBOARD_DEMO"] == "1"` (default off).

### Data volumes (Curitiba; Londrina ≈ 40%)

Time spread: `created_at`/`occurred_at`/`completed_at = now - offset` across the
last 30 days, deterministic offsets by index.

- **Conversations (~24):** greeting 4, awaiting_consent 3, consented 8,
  revoked 3, abandoned 3, completed 3. Deterministic phones (e.g.
  `+55 41 90000-00NN`).
- **Consents:** `ConsentTerm` v1 + v2; ~11 given (mix of versions, `revoked_at`
  NULL) tied to consented conversations, ~3 revoked (`revoked_at` spread).
- **InboundMessage (~40):** spread over 30d; ≥3 with `created_at > 24h` ago
  (overTtl), several within 24h (pending). Deterministic `message_id`
  (`IN-CWB-0001`…), `raw` a short non-PII placeholder.
- **OutboundMessage (~30):** spread over 30d across ack classes (mostly 2xx,
  some 3xx, some 4xx/5xx). Deterministic `idempotency_key` (`OUT-CWB-0001`…).
- **Triage (~30):** ~22 completed, ~5 in_progress, ~3 aborted_* (mix of
  aborted reasons); `tier ∈ {low, medium, high}` distributed; `priority` mixed
  (>0 and 0/nil); `outcome.scoring.mode ∈ {weighted, decision_table}`;
  `completed_at` spread over 30d with `created_at` a few minutes earlier;
  linked to 2+ protocol definitions/versions.
- **ReportSnapshot (~18):** one per completed triage subset; `expires_at` mix
  of future (live) and past (expired); `outcome.tier` varied. Generated via the
  same path as the baseline (`GenerateReportJob`) where practical, or built
  directly with deterministic `token`.
- **ProtocolDefinition:** `triage-respiratoria` v1 active (existing) + v2
  published + v3 draft; second protocol `triagem-dengue` v1 active + a retired
  version. (Optional: one platform-level `municipality_id = nil` protocol.)
- **DomainEvent (~70):** covers every filter prefix — `triage.scored`,
  `tier_assigned`, `priority_rule`, `rule_matched` (with `triage_id` +
  `actor` in payload, for trail); `consent.given`, `consent.revoked`;
  `conversation.*`; `protocol.created|published|retired` (distinct actors for
  four-eyes OK, one collapsed same-actor case). `occurred_at` spread over 30d;
  payload carries a `*_id` and `actor` for `ref`/actor extraction.

### Idempotency

Every row created via `find_or_create_by` on a deterministic natural key
(phone, `message_id`, `idempotency_key`, protocol `name+version+municipality`,
event `name + payload id + occurred_at` where needed). Re-running `db:seed:demo`
creates nothing new. Timestamps are set only on first create (find_or_create),
so re-runs keep existing rows; regenerate a fresh 30-day window by resetting the
DB then re-seeding.

## Testing / verification

- **Verification runner** (invoked after loading, not a committed spec unless
  cheap): run all nine `Admin::*Query` for both Curitiba and Londrina at 7d and
  30d, and assert each headline KPI/series is non-empty (funnel has each state,
  avgToCompleteMin numeric, tier KPIs each ≥1, ack has ≥2 classes, events count
  ≥1 per prefix, reports has live+expired). Same method used to verify the
  ambiguous-column fix.
- **Idempotency check:** run `db:seed:demo` twice; assert row counts identical.
- **Optional lightweight query spec** for the loader is out of scope unless a
  single focused example is cheap to add (the runner is the gate).

## Out of scope / follow-ups

- No dashboard code changes; no new panels.
- No write-flow seeding for the Protocol Editor beyond read data.
- Abandon-rate and declined-count KPIs stay "—" (queries return nil — not
  implemented server-side; seeding can't surface them).
- A dedicated `db:seed:demo:reset` convenience is a follow-up, not required.

## Workflow

FEATURE. Commits in English. `apps/api` on `fix/migrations-owner-ddl-as-admin`
(no new branch). api specs/runners execute in the `api-dev` container
(`docker exec api-dev …`). writing-plans → subagent-driven-development. Sync is a
separate user-authorized step.
