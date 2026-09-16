# F-03.2 — Protocol condition language (eq/in/gt/lt/all/any/not)

**Date:** 2026-07-01
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03
**Board:** F-03.2 (In Progress)
**Touches:** `apps/api` (engine + tests) + the mirrored `schema.json` contract (3 copies).

## Problem

The protocol engine only matches answers by exact equality. `Step#next_step_id`
keys the answer straight into `branches`, and `Scoring::DecisionTable#matches?`
does `answers[step_id] == expected` across an implicit AND. There is no way to
match a range or a set — e.g. "tier alta if idade > 60 OR febre = true" — so a
decision table must enumerate every value, which is impossible for integer
answers. F-03.2 adds a small, reusable condition language (eq / in / gt / lt /
all / any / not) and uses it in decision-table rule matching, backward-compatible
with the existing flat `when` maps. It is also the building block F-03.6
(`priority_when`) will reuse.

## Current state (verified)

- `Scoring::DecisionTable#matches?(conditions, answers)`:
  `conditions.all? { |step_id, expected| answers[step_id.to_s] == expected.to_s }`
  — flat `{step_id => value}` map, exact-match AND. `answers` is
  `{step_id(str) => answer(str)}` built from the trail.
- `ProtocolDefinition#validate` (the live publish/runtime gate) calls
  `Protocols::Validator.call`, whose linter checks start_step / branches /
  cycles / recommendations only — it does **not** validate decision-table `when`
  semantics, and does **not** run `Validation::Scoring` or the vendored
  `schema.json`. So a `when` (legacy or condition-DSL) is not publish-validated
  today; a malformed one is only felt at runtime.
- `Protocols::Validation::Scoring` / `::Schema` exist but are **not wired** into
  `Validator` (no live caller found). The vendored `schema.json` exists in 3
  identical copies (`contracts/protocols/`, `packages/protocols/`,
  `apps/api/config/protocols/`) as the ADR-0016 contract source of truth; no live
  code currently consumes it (its `when` is `{type: object, additionalProperties:
  {type: string}}`).
- `Protocols::Definitions.build` / `Scoring.build` construct the engine from JSON;
  `DecisionTable` receives `rules` (each with a `when`) verbatim.

## Decisions (from brainstorming)

1. Scope = a reusable evaluator (`Protocols::Condition`) + wiring into
   `DecisionTable`'s `when`. Branching is untouched.
2. Backward-compatible: a `when` that is not a single-key operator node is the
   legacy flat map (implicit `all` of `eq`).
3. Runtime is total (never raises): a missing/non-numeric operand makes a
   comparison `false` (default-deny, matching the engine's cautious bias); a
   malformed condition simply never matches → decision-table `fallback`.
4. Update the mirrored `schema.json` (3 copies) so `when` accepts the condition
   DSL or the legacy map — contract hygiene (ADR-0016), not a live gate today.
5. **Deferred** (follow-up card): validating condition nodes at publish. It would
   be dead code today (`Validation::Scoring`/`schema.json` are unwired), and
   wiring the full validator into `Validator` is a larger, riskier change
   (could reject existing protocols). Consistent with `when` being unvalidated
   at publish today.
6. No migration.

## Design

### 1. `Protocols::Condition` — the evaluator

`app/protocols/condition.rb` (pure module):

```ruby
module Protocols
  module Condition
    OPERATORS = %w[eq in gt lt all any not].freeze

    module_function

    # node: a condition-DSL node OR a legacy flat {step_id => value} map.
    # answers: { step_id(String) => answer(String) }
    def eval(node, answers)
      return false unless node.is_a?(Hash)
      return legacy_all_eq(node, answers) unless operator_node?(node)

      op, operand = node.first
      case op.to_s
      when "eq"  then answers[operand[0].to_s] == operand[1].to_s
      when "in"  then Array(operand[1]).map(&:to_s).include?(answers[operand[0].to_s])
      when "gt"  then numeric(answers[operand[0].to_s]) { |v| v > Float(operand[1]) }
      when "lt"  then numeric(answers[operand[0].to_s]) { |v| v < Float(operand[1]) }
      when "all" then Array(operand).all? { |n| eval(n, answers) }
      when "any" then Array(operand).any? { |n| eval(n, answers) }
      when "not" then !eval(operand, answers)
      else false
      end
    end

    def operator_node?(node)
      node.size == 1 && OPERATORS.include?(node.keys.first.to_s)
    end

    def legacy_all_eq(map, answers)
      return false if map.empty?   # default-deny: um `when` vazio não casa (evita catch-all acidental)
      map.all? { |step_id, expected| answers[step_id.to_s] == expected.to_s }
    end

    def numeric(raw)
      yield Float(raw)
    rescue ArgumentError, TypeError
      false
    end
  end
end
```

Operand shapes: `eq`/`gt`/`lt` → `[key, value]`; `in` → `[key, [v1, v2, …]]`;
`all`/`any` → `[node, …]`; `not` → a single node. Legacy `when`
(`{step_id => value, …}`) is detected as a non-operator hash and evaluated as
AND-of-eq.

### 2. `DecisionTable` uses `Condition`

`app/protocols/scoring/decision_table.rb`: replace `matches?` with a delegation:

```ruby
      def matches?(conditions, answers)
        Protocols::Condition.eval(conditions, answers)
      end
```

Everything else (`call`, `to_h`, `fallback`) is unchanged; existing flat-`when`
protocols keep matching exactly as before (the legacy path).

### 3. Mirrored `schema.json` contract (3 copies)

In each of `contracts/protocols/schema.json`, `packages/protocols/schema.json`,
`apps/api/config/protocols/schema.json` (kept byte-identical), replace the
decision-table rule `when` schema (currently `{type: object,
additionalProperties: {type: string}}`) with a `oneOf`: the legacy string-map
**or** a `$ref` to a new recursive `condition` definition in `$defs`. The
`condition` `$def` allows the seven operators with their operand shapes
(`eq`/`gt`/`lt`: `[string, string|number]`; `in`: `[string, array]`;
`all`/`any`: array of conditions; `not`: a condition). This keeps the contract
accurate; nothing live enforces it yet.

## Testing (RSpec — pure modules, no DB, run in the container)

- **`spec/protocols/condition_spec.rb`** (new): each operator — `eq` true/false;
  `in` membership; `gt`/`lt` numeric true/false; `gt`/`lt` with a missing or
  non-numeric answer → `false`; `all` (AND), `any` (OR), `not`; nested
  combinators; a legacy flat map (single- and multi-key) → AND-of-eq; a
  non-Hash / empty node → `false`; an unknown single-key operator → `false`.
- **`spec/protocols/scoring/decision_table_spec.rb`** (extend or create):
  a condition-DSL rule matches (e.g. `{any: [{gt: ["idade", 60]}, {eq:
  ["febre", "true"]}]}` → its tier) and falls through to `fallback` when it
  doesn't; a **legacy** flat-`when` rule still matches exactly as before
  (regression). Build `trail` as `[{step:, answer:, weight:}]` like the engine.

## Out of scope / follow-ups

- Publish-time validation of condition nodes (unknown operator, `gt`/`lt` only on
  integer steps, `eq`/`in` answers within the step's allowed set, step-id ≠
  operator name). Deferred until the full validator (`Validation::Scoring`/
  `Schema`) is wired into `Protocols::Validator`.
- Condition-based **branching** (range routing in `Step`) — a larger graph change.
- `priority_when` (F-03.6) — will reuse `Protocols::Condition`.

## Workflow

Card F-03.2 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`.
