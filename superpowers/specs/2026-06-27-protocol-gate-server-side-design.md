# F-03.9 — Server-side protocol gate (schema + graph linter + scoring)

**Date:** 2026-06-27
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03
**Board:** F-03.9 (item `PVTI_lADOEbfGRc4BbxiBzgw_5jg`)
**Touches:** `apps/api` (+ vendored copy of the protocol JSON Schema)

## Problem

A protocol definition is authored as a JSON document and stored per city
(`ProtocolDefinition`). Today the only server-side validation is
`Protocols::Validator`, which does a **minimal** shape subset (name/version/
start_step_id/steps presence) plus a graph linter (unknown refs, cycles) and
the recommendation↔tier check. The **full** JSON Schema (`packages/protocols/
schema.json`, draft 2020-12) only runs **offline**, and there is no validation
of scoring semantics or graph reachability. As a result an invalid definition
can be **published** — the publish flow (`Protocols::Publish`) only triggers the
minimal validator via the model's `before_save`.

F-03.9 brings the full gate **server-side** and enforces it at publish: schema
(complete shape) + graph linter (reachability, key validity) + scoring
semantics.

## Decisions (from brainstorming)

1. **Enforcement = publish + the `/gate` endpoint.** `Protocols::Publish` blocks
   publishing an invalid definition; `POST /protocols/:name/gate` runs the same
   full validation for author preview. The model's `before_save` stays
   **minimal** (it is the "can the engine safely load this" check; drafts may
   fail the full gate and still save). Rejected: running the full gate on every
   model save (pays full JSON Schema on every draft write — against the existing
   design); gate only at `/gate` (lets an invalid definition that skipped
   preview be published).
2. **Schema validation = `json_schemer` gem against a vendored `schema.json`.**
   The JSON Schema is the canonical cross-language contract (shared with TS), so
   re-implementing it in Ruby would duplicate and drift. Cost accepted: a new
   gem + a vendored copy of the schema inside `apps/api`. Rejected: hand-rolled
   Ruby shape checks.
3. **Semantic checks (all four selected):** unreachable steps; `branches`/
   `weights` keys valid for `answer_type`; `decision_table.when` references a
   valid step + valid answer; `weighted` `priority_map` tiers ⊆ `thresholds`.

## Design

### Architecture: `Protocols::Gate` (new) vs `Protocols::Validator` (existing)

- **`Protocols::Validator`** — **behavior unchanged** (its public `Result` stays
  identical): minimal shape subset + refs + cycles + recommendation↔tier. Keeps
  running in `ProtocolDefinition#before_save` and in `Protocols::Definitions.build`
  (engine load safety). Drafts that fail the full gate can still be saved.
  Planning *may* extract the refs/cycles graph helpers into a shared module that
  both `Validator` and the Gate's Graph unit call — a refactor that preserves
  `Validator`'s behavior, not a change to it.
- **`Protocols::Gate`** — new. The publish/preview quality gate. Returns the same
  `Protocols::Validator::Result` struct shape (`valid?`, `errors`). Composed of
  three focused, independently testable units. `Gate.call(definition)` runs all
  three, aggregates errors, returns a `Result`.

> Note on overlap: the full JSON Schema (unit A) is a superset of `Validator`'s
> minimal shape subset, and graph refs/cycles already live in `Validator`. The
> Gate's Graph unit (B) reuses `Validator`'s refs/cycles logic (extract a shared
> module method rather than duplicate) and adds the new graph checks. Decide the
> exact extraction during planning; do not duplicate the cycle/ref code verbatim.

### Unit A — `Protocols::Validation::Schema`

Loads the vendored schema once and validates with `json_schemer`:

```ruby
SCHEMA = JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json"))))
def self.call(definition)
  SCHEMA.validate(definition).map { |e| "schema: #{e["data_pointer"].presence || "/"} #{e["type"]}" }
end
```

Covers all pure shape: `answer_type` enum, `options` items, `weights`/`branches`
value types, `scoring` `oneOf` (weighted/decision_table), priority ranges 1–9,
`additionalProperties:false`, `recommendations` shape, etc. (Exact error-string
format is finalized in the plan; it must name the offending JSON pointer.)

### Unit B — `Protocols::Validation::Graph`

Semantic graph checks the schema cannot express:
- **Refs** (reuse): `start_step_id` and every `branches` target is a known step.
- **Cycles** (reuse): the branch graph is acyclic.
- **Unreachable steps** (new): every step is reachable from `start_step_id` via
  `branches`; report each orphan — `"unreachable step: <id>"`.
- **Branch/weight key validity** (new): for each step, the keys of `branches` and
  `weights` must match `answer_type` — `boolean` → subset of `{"true","false"}`;
  `enum` → subset of `options`. Report `"branch key '<k>' invalid for <type>
  step <id>"` (and the analogous `weight key` message).

### Unit C — `Protocols::Validation::Scoring`

Scoring semantics:
- **`decision_table.when`** (new): each `when` entry `{step_id => answer}` must
  reference an existing step **and** a valid answer value for that step
  (`boolean` → `"true"/"false"`; `enum` → in `options`). Report
  `"decision_table rule references unknown step <id>"` / `"... invalid answer
  '<a>' for step <id>"`.
- **`weighted` tier consistency** (new): every key of `priority_map` must be a
  key of `thresholds`. Report `"priority_map tier '<t>' not in thresholds"`.

(Recommendation↔tier consistency already ships in `Validator` from F-03.17
follow-up; it is not duplicated here.)

### Vendored schema (multi-repo reality)

`apps/api` is its own repo/container and cannot read `packages/`. Vendor a copy
at **`apps/api/config/protocols/schema.json`**, mirroring `packages/protocols/
schema.json` (same mirror discipline already used between `packages/` and
`contracts/`). The spec/plan documents that the three copies must stay in sync.
Add **`json_schemer`** to `apps/api/Gemfile`; installing it in the running
container is an operational step (`docker exec api-dev bundle install`, or image
rebuild if the bundle path is baked in).

### Enforcement points

- **`Protocols::Publish`**: before `protocol.update!(status: "published")`, run
  `result = Protocols::Gate.call(protocol.definition)`; if invalid return
  `Result.fail(:invalid, message: result.errors.join("; "))`. The existing
  `PublicationsController` maps `:invalid` (the `else` branch) to HTTP 422 with
  the message — no controller change required.
- **`ProtocolsController#gate`**: replace `Protocols::Validator.call` with
  `Protocols::Gate.call`. Same JSON contract (`{valid:true}` / `{valid:false,
  errors:[...]}`, 422).
- **`before_save`**: unchanged (minimal `Validator`).

## Testing (RSpec, run in the container: `docker exec api-dev bundle exec rspec`)

- **Unit specs** per validation unit (pure, no DB):
  - `Validation::Schema`: a valid definition → no errors; bad `answer_type`,
    missing `options` on enum, out-of-range priority, malformed `scoring` → the
    corresponding schema error.
  - `Validation::Graph`: valid graph passes; unreachable step, bad branch key
    for boolean, bad weight key for enum → the right error.
  - `Validation::Scoring`: valid scoring passes; `decision_table.when` with
    unknown step / invalid answer, `priority_map` tier not in `thresholds` → the
    right error.
- **`Protocols::Gate`** spec: a fully valid definition passes; a definition with
  one error from each unit aggregates all three.
- **`Protocols::Publish`** spec: publishing an invalid definition returns
  `:invalid` and does not transition status; a valid one reaches `published`
  (extend the existing lifecycle spec or add a focused one; uses the tenant
  transaction pattern).
- **Request spec** for `POST /protocols/:name/gate`: valid → `{valid:true}`;
  invalid → 422 with `errors`.

## Out of scope / follow-ups

- F-03.2 (condition language eq/in/gt/lt/all/any/not), F-03.6 (`priority_when`),
  F-07.2 (event insert in the publish transaction) — separate Not Started cards.
- Wiring an automated drift check between the three `schema.json` copies (the api
  container cannot read the root copies; left as documented manual discipline).

## Workflow

Card F-03.9 is In Progress. After spec approval: writing-plans →
subagent-driven-development. Commits in English; api work on branch
`fix/migrations-owner-ddl-as-admin`.
