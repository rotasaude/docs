# Tier-based recommendation in the triage result

**Date:** 2026-06-27
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03 (follow-up of F-03.17)
**Touches:** `apps/api`, `apps/wpda`, `packages/protocols` (+ `contracts/protocols` mirror)

## Problem

The citizen-facing report page (`apps/wpda/src/modules/Report.tsx`) renders the triage
`tier` + `priority` + date and a **generic placeholder note** ("Este é o resultado da sua
triagem. Siga as orientações da sua unidade de saúde. Se os sintomas piorarem, procure
atendimento."). The real, tier-specific health recommendation was deferred in F-03.17.

The reason it cannot be done safely in the frontend: **`tier` is an arbitrary string defined
per protocol**, not a fixed enum. In the `weighted` scorer the tiers are the keys of
`thresholds` (e.g. `"baixa"/"media"/"alta"`); in `decision_table` they are `rules[].tier` plus
`fallback.tier` (e.g. `"indefinido"`). Each city/protocol invents its own tiers. A tier→text map
in the wpda would silently give wrong health advice for any tier it does not know.

## Decisions (from brainstorming)

1. **Source of truth = the protocol definition.** A new optional top-level `recommendations`
   field (tier → text) in the protocol definition schema. The same medical authority that
   defines the tiers writes what each tier means. The text is versioned and frozen alongside
   the protocol. (Rejected: a derived field on the pure `Outcome` — mixes computation with
   human content; a separate config table — drift risk + extra source to sync.)
2. **Format = title + body.** Each tier → `{ title, body }`. The wpda emphasizes the title
   (clear/urgent directive) and shows the body below. (Rejected: plain string — no visual
   emphasis; title+body+steps[] — YAGNI for now.)
3. **Light validation included.** The validator checks that, *if* `recommendations` is present,
   every key maps to a tier declared by the scoring config — catches author typos before
   publication. Not made mandatory (backward compat with already-published protocols).

## Design

### 1. Schema (`packages/protocols/schema.json`, canonical per ADR-0016)

Add an optional top-level property. Verify and mirror to `contracts/protocols/schema.json`
during planning (confirm whether it is a generated copy or a hand-maintained duplicate).

```jsonc
"recommendations": {
  "type": "object",
  "additionalProperties": {
    "type": "object",
    "required": ["title", "body"],
    "additionalProperties": false,
    "properties": {
      "title": { "type": "string" },
      "body":  { "type": "string" }
    }
  }
}
```

Key = tier name (same strings as `thresholds` / `rules[].tier` / `fallback.tier`). Optional, so
existing published protocols without the field stay valid.

### 2. Engine stays pure

`Protocols::Outcome`, `Protocols::Scoring::{Weighted,DecisionTable}` and `Protocols::Protocol`
**do not change**. Recommendation text is human content, not computation — it must not enter the
pure engine. This preserves the existing separation (Outcome is a value object about scoring).

### 3. Freeze into the snapshot (`GenerateReportJob#build_payload`)

At report-generation time the job resolves the tier to its text and **freezes** it into the
immutable payload (ADR-0007):

```ruby
def build_payload(triage)
  recs = triage.protocol_definition.definition["recommendations"]
  {
    tier: triage.tier,
    priority: triage.priority,
    recommendation: recs&.dig(triage.tier),   # => { "title", "body" } or nil
    summary: triage.outcome.dig("trail")&.map { |e| { step: e["step"], answer: e["answer"] } },
    completed_at: triage.completed_at&.iso8601
  }
end
```

Old snapshots and protocols without `recommendations` → `recommendation: nil`.

### 4. Endpoint `/r/:token` (`ReportsController#show`)

Include the frozen field in the JSON:

```ruby
recommendation: snapshot.payload["recommendation"],
```

### 5. wpda render

- `apps/wpda/src/lib/report.ts`: extend the interface —
  `recommendation: { title: string; body: string } | null`.
- `apps/wpda/src/modules/Report.tsx`: if `recommendation` is present, render **title (emphasized)
  + body**; if `null`, **fall back to the current generic note** (backward compat with old
  snapshots / protocols without a recommendation). The frontend never fabricates health advice.

### 6. Validation (`Protocols::Validator`)

In `linter_errors` (semantic pass): if `definition["recommendations"]` is present, collect the
set of declared tiers from the scoring config and assert every recommendation key is a member.

- `weighted` → tiers = keys of `scoring.thresholds`.
- `decision_table` → tiers = `rules[].tier` ∪ `fallback.tier`.
- No `scoring` block → any recommendation key is "unknown" (error), since there are no tiers.

Error message shape: `recommendation for unknown tier <key>`. Recommendations remain optional
overall; only their *keys* are validated when present.

## Testing

**api** (run in container: `docker exec api-dev bundle exec rspec <path>`):
- New `spec/jobs/generate_report_job_spec.rb`: freezes the correct `{title, body}` for the
  triage's tier; `recommendation: nil` when the protocol has no `recommendations` or the tier is
  absent.
- Request spec for `GET /r/:token`: response includes `recommendation` (present and nil cases).
- `Protocols::Validator` spec: recommendation key not matching a declared tier → invalid;
  matching keys / absent field → valid.

**wpda** (run on host: `cd apps/wpda && npm run test`):
- `src/lib/report.test.ts`: parses the new `recommendation` field (object and null).
- `Report.tsx` test: renders title+body when present; renders the generic fallback when null.

## Out of scope / follow-ups

- Authoring UI for `recommendations` in the admin/editor (separate feature).
- Richer recommendation structure (steps[], CTAs, links).
- Making recommendations mandatory per tier at publish time.

## Workflow

Create a new card on board #1 (follow-up of F-03.17, **mod-03**, Work type = feature), move to
In Progress, then writing-plans → subagent-driven-development. Commits in English.
