# CHORE — Publish-time validation of protocol conditions (when / priority_when)

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03
**Board:** CHORE card (In Progress). Deferred from F-03.2 and F-03.6.
**Touches:** `apps/api` only.

## Problem

Two gaps, one of them a **latent bug**:

1. **Condition-DSL protocols cannot be published.** F-03.2 widened the vendored
   `schema.json` and the engine to accept a condition-DSL `when`
   (`{gt: [...]}`, `{any: [...]}`, …), but did NOT update
   `Protocols::Validation::Scoring`. That linter runs inside `Protocols::Gate`
   (the publish/preview gate, called by `Protocols::Publish`) and still assumes
   `when` is a flat `{step_id => answer}` map — for `{"gt" => ["idade", 60]}` it
   reads `"gt"` as a step id → `by_id["gt"]` nil → **"references unknown step gt"**.
   So a valid condition-DSL decision table passes the schema but is **rejected by
   the Gate at publish**. Conditions work at runtime but are unpublishable.
2. **No semantic validation of condition/priority_when nodes** at publish
   (unknown operator, `gt`/`lt` on a non-integer step, `eq`/`in` answers outside
   the step's allowed set, a step id colliding with an operator name — the
   F-03.2/F-03.6 whole-branch reviews flagged all of these).

This chore makes the publish Gate correctly validate condition nodes: unblocking
valid condition protocols AND rejecting malformed ones.

## Current state (verified)

- `Protocols::Gate.call(definition)` composes: `Validation::Schema` (vendored
  JSON schema — shape, `priority` 1..9, `when` oneOf), then (if shape valid)
  `Validator` (refs/cycles/recommendation) + `Validation::Graph` +
  `Validation::Scoring`. Returns a `Validator::Result`.
- `Protocols::Publish` calls `Gate.call` and fails `:invalid` unless valid. So
  Gate IS the publish authority. (The runtime `Definitions.build` uses only the
  minimal `Validator`, and is total by design — out of scope here.)
- `Validation::Scoring.decision_table_errors(scoring, steps)` iterates
  `rule["when"]` as a flat `{step_id => answer}` map, checking the step exists and
  the answer ∈ `Validation::Answers.for(step)` (`%w[true false]` for boolean,
  `options` for enum, `nil` = unconstrained for integer/text). **Does not walk
  condition nodes.**
- `Validation::Scoring.call(definition)` reads only `definition["scoring"]` —
  `priority_when` (a top-level sibling, F-03.6) is **not validated anywhere**
  semantically (the schema validates its `priority` range and `when` shape only).
- `Protocols::Condition::OPERATORS = %w[eq in gt lt all any not]`; runtime
  `Condition.eval` treats a single-key hash whose key ∈ OPERATORS as an operator
  node, else as a legacy map.
- Steps have `answer_type` ∈ `boolean|enum|integer|text`.

## Decisions (from brainstorming)

1. Validate at the **publish Gate** only; runtime stays total (no `Definitions.build`
   change).
2. A reusable `Validation::Condition` walker validates a `when` node (condition
   or legacy); `Validation::Scoring` and a new `priority_when` validator both use
   it. No migration.
3. Rules: unknown operator; operand shape; `eq`/`in` step exists + value(s) in the
   allowed set; `gt`/`lt` step exists + `answer_type == "integer"`; `all`/`any`/
   `not` recurse; legacy map = existing per-pair checks; step id ∉ OPERATORS;
   `priority_when` priority present + 1..9.

## Design

### 1. `Protocols::Validation::Condition` (new)

`app/protocols/validation/condition.rb`:
- `self.errors(node, by_id) -> [String]` where `by_id` = `{ step_id => step_hash }`:
  - Not a Hash / empty → `["condition must be a non-empty object"]`.
  - Operator node (single key ∈ `Condition::OPERATORS`):
    - `eq`/`in`: operand must be `[step_id, value_or_values]`; step must exist;
      for a constrained step (`Answers.for` non-nil), each value must be allowed;
      else `"eq/in references unknown step …"` / `"invalid answer '…' for step …"`.
    - `gt`/`lt`: operand `[step_id, number]`; step must exist AND
      `step["answer_type"] == "integer"` (else `"gt/lt requires an integer step,
      got <type> for <step>"`); threshold must be numeric.
    - `all`/`any`: operand must be an Array; recurse `errors` into each element.
    - `not`: recurse `errors` into the operand (a node).
  - Legacy map (non-operator hash): each `{step_id => answer}` pair — step exists +
    answer ∈ allowed set (the current `decision_table_errors` logic, extracted).
- `self.step_id_collision_errors(steps) -> [String]`: a step whose `id` ∈
  `Condition::OPERATORS` → `"step id '<id>' collides with a condition operator"`.

### 2. `Validation::Scoring` — walk conditions

Rewrite `decision_table_errors(scoring, steps)` to build `by_id` and call
`Validation::Condition.errors(rule["when"], by_id)` per rule (replacing the flat
`when.flat_map`). Legacy flat `when` still validates (Condition handles it),
condition `when` now validates correctly — **the F-03.2 rejection bug is fixed**.

### 3. `Validation::PriorityWhen` (new), composed in `Gate`

`app/protocols/validation/priority_when.rb`:
- `self.call(definition) -> [String]`: for each `definition["priority_when"]` rule:
  `Validation::Condition.errors(rule["when"], by_id)` + `priority` present and an
  integer in 1..9 (`"priority_when priority must be 1..9"`). (The schema also
  bounds `priority`, but validating here keeps the semantic layer self-contained.)

### 4. `Gate` composition

`app/protocols/gate.rb`: after the existing semantic linters, add
`errors.concat(Validation::PriorityWhen.call(definition))` and
`errors.concat(Validation::Condition.step_id_collision_errors(definition["steps"] || []))`.

## Testing (RSpec, pure — run in the container)

- **`spec/protocols/validation/condition_spec.rb`**: eq/in with a known step +
  allowed answer → no error; unknown step → error; invalid enum/boolean answer →
  error; gt/lt on an integer step → no error; gt/lt on a boolean/enum step →
  error; all/any/not recurse (a nested bad node surfaces); legacy flat map
  (valid + invalid); operand shape errors; step_id_collision_errors flags a step
  named `eq`.
- **`spec/protocols/gate_spec.rb`** (extend): a decision_table with a **valid
  condition `when`** now passes the Gate (regression for the fix); a `gt` on a
  non-integer step fails; an `eq` with a disallowed answer fails; a step named
  `any` fails; a **legacy** flat-`when` protocol still passes.
- **`priority_when`** (in the gate spec or a dedicated spec): a `priority_when`
  with an invalid `when` fails; `priority` 10 or missing fails; a valid
  `priority_when` passes.

## Out of scope / follow-ups

- Validating conditions at runtime (`Definitions.build`) — the engine is total by
  design; keep publish as the authoritative gate.
- Auto-fixing / migrating already-stored malformed protocols.

## Workflow

CHORE card In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
