# F-03.6 — priority_when (mode-independent, escalate-only)

**Date:** 2026-07-02
**Status:** Approved (brainstorming → plan written)
**Module:** mod-03
**Board:** F-03.6 (In Progress)
**Touches:** `apps/api` (engine + tests) + the mirrored `schema.json` contract (3 copies).
**Builds on:** F-03.2 (`Protocols::Condition`).

## Problem

Priority is computed inside the scoring strategy and coupled to the mode:
`Scoring::Weighted` derives it from `priority_map[tier]`, `Scoring::DecisionTable`
from the matched rule. There is no way to say "these answers are always urgent,
regardless of the score" (e.g. `idade > 80 → priority 1`). F-03.6 adds a
`priority_when` layer — condition→priority rules applied after scoring,
independent of the mode — that can only **escalate** urgency.

## Current state (verified)

- `Protocol#evaluate(answers)` walks the trail, then returns
  `scoring ? scoring.call(trail) : Outcome.terminal(trail:)`. Pending outcomes
  return earlier in the loop.
- `Outcome` is immutable/frozen (`tier`, `priority`, `score`, `trail`, …);
  `Outcome.terminal(trail:, tier:, priority:, score:)` builds a new one.
- `Weighted#call` sets `priority: priority_map.fetch(tier, 5)`;
  `DecisionTable#call` sets `priority` from the matched rule / fallback.
- `Protocols::Condition.eval(node, answers)` (F-03.2) is a pure, total matcher;
  `answers` is `{step_id(String) => answer(String)}`, built from the trail.
- Protocol JSON schema: top-level `additionalProperties: false`, `$defs` includes
  `condition` (F-03.2); 3 byte-identical copies (ADR-0016), unwired at runtime
  (contract only).
- `Definitions.build` constructs `Protocol.new(...)` from the definition hash.

## Decisions (from brainstorming)

1. **Escalate-only (min):** final priority = `min(mode_priority, matched rule
   priorities)`; a `nil` base becomes the matched min. Never de-escalates —
   a clinical safety net aligned with the engine's cautious bias.
2. Applied **after scoring**, on the terminal Outcome only; mode-independent
   (identical for weighted and decision_table). Tier/score are unchanged — only
   priority.
3. Reuse `Protocols::Condition` for `when` (condition node or legacy flat map).
4. `priority_when` is a top-level protocol field (array of `{when, priority}`),
   round-tripped in `Protocol#to_h`; schema contract widened (3 copies).
5. **Deferred** (shared follow-up with F-03.2): publish-time validation of
   `priority_when`/condition nodes. No migration.

## Design

### 1. `Protocols::PriorityRules` (pure)
`override_for(rules, answers) -> Integer | nil` — the minimum `priority` among
rules whose `when` matches `Condition.eval`; `nil` if none match / rules
empty/nil. Supports string- and symbol-keyed rules. Total (via Condition).

### 2. `Protocol#evaluate` — escalate layer
After `outcome = scoring ? scoring.call(trail) : Outcome.terminal(trail:)`:
build `answers` from the trail (`{step=>answer}` strings), compute
`escalated = PriorityRules.override_for(priority_rules, answers)`, and if it is
more urgent than the base, return a new Outcome with
`priority = [outcome.priority, escalated].compact.min` (tier/score/trail
preserved). No-op when there are no rules, none match, or the base is already at
least as urgent. `Protocol.new` gains `priority_rules:`; `Definitions.build`
passes `hash["priority_when"]`; `to_h` round-trips `priority_when`.

### 3. `schema.json` contract (3 copies)
Add a top-level `priority_when`: `array` of `{ when: oneOf[legacy string-map,
#/$defs/condition], priority: integer 1..9 }`. Must be a declared property
(root `additionalProperties: false`); not `required`. Contract hygiene — nothing
live enforces it.

## Testing (RSpec, pure — run in the container)

- **PriorityRules**: min among matching rules; ignores non-matching; nil for
  none/empty/nil; legacy flat `when` + symbol keys.
- **Protocol#evaluate**: escalates (base 5 + matching rule 1 → 1, tier unchanged);
  never de-escalates (base 1 + rule 9 → 1); no-op when no rule matches; no-op with
  no `priority_when`; `to_h` round-trips `priority_when`. Same behavior under
  weighted and decision_table.
- **schema contract**: `priority_when` with a condition `when` and a legacy `when`
  validate; a `priority` out of 1..9 is rejected.

## Out of scope / follow-ups

- Publish-time validation of `priority_when` and condition nodes (shared card
  with F-03.2's deferred validation).
- De-escalation / full override semantics (explicitly rejected — escalate-only).
- Tier overrides (only priority is escalated).

## Workflow

Card F-03.6 In Progress. Plan: `docs/superpowers/plans/2026-07-02-protocol-priority-when.md`.
subagent-driven-development. Commits in English; api on branch
`fix/migrations-owner-ddl-as-admin`.
