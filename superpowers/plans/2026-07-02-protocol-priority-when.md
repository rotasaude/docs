# Protocol priority_when (F-03.6) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a mode-independent `priority_when` layer that can only ESCALATE a triage's priority (never de-escalate), applied after scoring, reusing `Protocols::Condition`.

**Architecture:** A pure `Protocols::PriorityRules.override_for(rules, answers)` returns the minimum priority among rules whose `when` matches (via `Condition.eval`); `Protocol#evaluate` applies it as `min(outcome.priority, escalated)` on the terminal Outcome; `Definitions.build` reads `priority_when`; the mirrored `schema.json` (3 copies) gains a top-level `priority_when`.

**Tech Stack:** Ruby (pure modules), RSpec, JSONSchemer (draft 2020-12). Run specs in the `api-dev` container.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- **Escalate-only:** priority_when can only make priority MORE urgent (lower number). Final priority = `min(mode_priority, matched_rule_priorities)`; if the base priority is `nil`, it becomes the matched min. NEVER de-escalate.
- Reuse `Protocols::Condition.eval` (total) — a `when` is a condition node or a legacy flat map; support string- and symbol-keyed rules.
- `priority_when` applies only to terminal outcomes; branching and scoring are untouched; no migration.
- The 3 `schema.json` copies (`contracts/`, `packages/`, `apps/api/config/protocols/`) stay byte-identical (ADR-0016); only `apps/api` is a git repo.
- **Deferred (do NOT implement):** publish-time validation of `priority_when`/condition nodes (same follow-up as F-03.2).

---

### Task 1: `Protocols::PriorityRules`

**Files:**
- Create: `apps/api/app/protocols/priority_rules.rb`
- Test: `apps/api/spec/protocols/priority_rules_spec.rb` (create)

**Interfaces:**
- Produces: `Protocols::PriorityRules.override_for(rules, answers) -> Integer | nil` — the min `priority` among rules whose `when` matches `Condition.eval`; `nil` if no rule matches, or rules is empty/nil.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/priority_rules_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::PriorityRules do
  let(:answers) { { "idade" => "85", "febre" => "true" } }

  it "returns the min priority among matching rules" do
    rules = [
      { "when" => { "gt" => ["idade", 80] }, "priority" => 1 },
      { "when" => { "eq" => ["febre", "true"] }, "priority" => 3 }
    ]
    expect(described_class.override_for(rules, answers)).to eq(1)
  end

  it "ignores non-matching rules" do
    rules = [
      { "when" => { "gt" => ["idade", 90] }, "priority" => 1 },
      { "when" => { "eq" => ["febre", "true"] }, "priority" => 3 }
    ]
    expect(described_class.override_for(rules, answers)).to eq(3)
  end

  it "is nil when no rule matches / empty / nil" do
    none = [{ "when" => { "gt" => ["idade", 90] }, "priority" => 1 }]
    expect(described_class.override_for(none, answers)).to be_nil
    expect(described_class.override_for([], answers)).to be_nil
    expect(described_class.override_for(nil, answers)).to be_nil
  end

  it "supports a legacy flat when and symbol keys" do
    expect(described_class.override_for([{ "when" => { "febre" => "true" }, "priority" => 2 }], answers)).to eq(2)
    expect(described_class.override_for([{ when: { "eq" => ["febre", "true"] }, priority: 4 }], answers)).to eq(4)
  end
end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/priority_rules_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::PriorityRules`.

- [ ] **Step 3: Implement**

Create `apps/api/app/protocols/priority_rules.rb`:

```ruby
# Camada de prioridade independente do modo de scoring (ADR-0017). Escala-só:
# devolve a MENOR priority entre as regras cujo `when` casa (via Condition,
# total), ou nil se nenhuma casa. Módulo puro. (F-03.6)
module Protocols
  module PriorityRules
    module_function

    def override_for(rules, answers)
      Array(rules)
        .select { |rule| Condition.eval(rule["when"] || rule[:when], answers) }
        .map { |rule| (rule["priority"] || rule[:priority]).to_i }
        .min
    end
  end
end
```

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/priority_rules_spec.rb`
Expected: PASS (all examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/protocols/priority_rules.rb spec/protocols/priority_rules_spec.rb
git -C apps/api commit -m "Add Protocols::PriorityRules (escalate-only priority_when override)"
git -C apps/api log --oneline -1
```

---

### Task 2: `Protocol#evaluate` applies priority_when

**Files:**
- Modify: `apps/api/app/protocols/protocol.rb`, `apps/api/app/protocols/definitions.rb`
- Test: `apps/api/spec/protocols/protocol_priority_when_spec.rb` (create)

**Interfaces:**
- Consumes: `Protocols::PriorityRules.override_for` (Task 1).
- Produces: `Protocol.new(..., priority_rules:)`; `Protocol#evaluate` escalates the terminal Outcome's priority via priority_when; `Definitions.build` reads `hash["priority_when"]`; `Protocol#to_h` round-trips `priority_when`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/protocol_priority_when_spec.rb`. Build protocols through `Protocols::Definitions.build` (exercises the wiring end-to-end):

```ruby
require "rails_helper"

RSpec.describe "Protocol priority_when (F-03.6)" do
  # single boolean step "grave"; weighted scoring; answers drive the base tier.
  def build(priority_when:, weights: { "true" => 1, "false" => 0 })
    Protocols::Definitions.build(
      "name" => "pw-demo", "version" => 1, "start_step_id" => "grave",
      "steps" => [{ "id" => "grave", "prompt" => "?", "answer_type" => "boolean",
                    "branches" => { "true" => nil, "false" => nil }, "weights" => weights }],
      "scoring" => { "type" => "weighted",
                     "thresholds" => { "baixa" => 0, "alta" => 1 },
                     "priority_map" => { "baixa" => 5, "alta" => 2 } },
      "priority_when" => priority_when
    )
  end

  it "escalates priority when a rule matches (min beats the mode)" do
    protocol = build(priority_when: [{ "when" => { "eq" => ["grave", "false"] }, "priority" => 1 }])
    outcome = protocol.evaluate("grave" => "false") # base tier baixa → priority 5
    expect(outcome.priority).to eq(1)
    expect(outcome.tier).to eq("baixa") # tier is NOT changed, only priority
  end

  it "never de-escalates (a higher-number rule does not raise a more-urgent base)" do
    protocol = build(priority_when: [{ "when" => { "eq" => ["grave", "true"] }, "priority" => 9 }])
    outcome = protocol.evaluate("grave" => "true") # base tier alta → priority 2
    expect(outcome.priority).to eq(2)
  end

  it "leaves priority unchanged when no rule matches" do
    protocol = build(priority_when: [{ "when" => { "eq" => ["grave", "true"] }, "priority" => 1 }])
    outcome = protocol.evaluate("grave" => "false") # rule needs grave=true; base priority 5
    expect(outcome.priority).to eq(5)
  end

  it "is a no-op when there is no priority_when" do
    protocol = build(priority_when: nil)
    outcome = protocol.evaluate("grave" => "false")
    expect(outcome.priority).to eq(5)
  end

  it "round-trips priority_when in to_h" do
    rules = [{ "when" => { "eq" => ["grave", "false"] }, "priority" => 1 }]
    expect(build(priority_when: rules).to_h[:priority_when]).to eq(rules)
  end
end
```

If `Definitions.build` rejects the fixture (the minimal Validator checks name/version/start_step_id/steps only — priority_when is not validated there), or `Outcome`/`to_h` keys differ, align the spec to reality — the asserted behavior (escalate / no de-escalate / no-op / round-trip) is the requirement.

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/protocol_priority_when_spec.rb`
Expected: FAIL — priority_when is ignored (escalate example keeps priority 5; to_h has no `:priority_when`).

- [ ] **Step 3: Implement — `protocol.rb`**

In `apps/api/app/protocols/protocol.rb`:

(a) add `priority_rules` to the reader and constructor:
```ruby
    attr_reader :name, :version, :steps, :start_step_id, :scoring, :priority_rules

    def initialize(name:, version:, steps:, start_step_id:, scoring: nil, priority_rules: nil)
      @name = name
      @version = version
      @steps = steps.each_with_object({}) { |s, acc| acc[s.id] = s }
      @start_step_id = start_step_id.to_sym
      @scoring = scoring
      @priority_rules = priority_rules
      freeze
    end
```

(b) apply priority_when at the end of `evaluate` (replace the final `scoring ? ... : ...` line):
```ruby
      outcome = scoring ? scoring.call(trail) : Outcome.terminal(trail: trail)
      apply_priority_when(outcome, trail)
    end

    private

    # Escala-só (F-03.6): priority_when só pode aumentar a urgência (min).
    def apply_priority_when(outcome, trail)
      return outcome unless outcome.terminal?
      answers = trail.to_h { |entry| [entry[:step].to_s, entry[:answer].to_s] }
      escalated = PriorityRules.override_for(priority_rules, answers)
      return outcome unless escalated
      final = [outcome.priority, escalated].compact.min
      return outcome if final == outcome.priority
      Outcome.terminal(trail: outcome.trail, tier: outcome.tier, priority: final, score: outcome.score)
    end

    public
```
(Place `apply_priority_when` as a private method; keep `to_h` public — see (c). If the file has no existing `private` section, wrapping as shown is fine; otherwise put the method under the existing private section and drop the extra `private`/`public` markers.)

(c) round-trip in `to_h` (add the key before `.compact`):
```ruby
    def to_h
      {
        name: name,
        version: version,
        start_step_id: start_step_id.to_s,
        steps: steps.values.map(&:to_h),
        scoring: scoring&.to_h,
        priority_when: priority_rules
      }.compact
    end
```
(`.compact` drops `priority_when` when `priority_rules` is `nil`.)

- [ ] **Step 4: Implement — `definitions.rb`**

In `apps/api/app/protocols/definitions.rb`, pass `priority_when` into `Protocol.new`:
```ruby
      Protocol.new(
        name: hash["name"],
        version: hash["version"],
        steps: steps,
        start_step_id: hash["start_step_id"],
        scoring: Scoring.build(hash["scoring"]),
        priority_rules: hash["priority_when"]
      )
```

- [ ] **Step 5: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/protocol_priority_when_spec.rb`
Expected: PASS (escalate / no de-escalate / no-match / no-op / round-trip).

- [ ] **Step 6: Regression**

Run: `docker exec api-dev bundle exec rspec spec/protocols`
Expected: green (existing protocol/weighted/decision_table/condition/gate/validator specs unaffected — priority_when defaults to nil → no-op).

- [ ] **Step 7: Commit**

```bash
git -C apps/api add app/protocols/protocol.rb app/protocols/definitions.rb spec/protocols/protocol_priority_when_spec.rb
git -C apps/api commit -m "Apply escalate-only priority_when after scoring in Protocol#evaluate"
git -C apps/api log --oneline -1
```

---

### Task 3: `schema.json` top-level `priority_when` contract

**Files:**
- Modify: `contracts/protocols/schema.json`, `packages/protocols/schema.json`, `apps/api/config/protocols/schema.json` (keep byte-identical)
- Test: `apps/api/spec/protocols/schema_priority_when_spec.rb` (create)

**Interfaces:**
- Produces: the top-level `priority_when` (array of `{when: legacy-map|condition, priority: 1..9}`) validates; the schema still loads. `additionalProperties: false` at the root means `priority_when` MUST be a declared property.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/schema_priority_when_spec.rb`:

```ruby
require "rails_helper"
require "json_schemer"

RSpec.describe "protocols schema.json priority_when contract (F-03.6)" do
  let(:schema) { JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json")))) }

  def base(priority_when)
    {
      "name" => "pw-contract", "version" => 1, "start_step_id" => "s1",
      "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                    "branches" => {}, "weights" => {} }],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 5 } },
      "priority_when" => priority_when
    }
  end

  it "accepts priority_when with a condition-DSL when" do
    expect(schema.valid?(base([{ "when" => { "gt" => ["s1", 80] }, "priority" => 1 }]))).to be(true)
  end

  it "accepts priority_when with a legacy flat when" do
    expect(schema.valid?(base([{ "when" => { "s1" => "true" }, "priority" => 2 }]))).to be(true)
  end

  it "rejects a priority out of range" do
    expect(schema.valid?(base([{ "when" => { "s1" => "true" }, "priority" => 10 }]))).to be(false)
  end
end
```
Confirm the `base` doc otherwise validates (name ≥2 chars per the existing `name` pattern, etc.) — adjust the fixture, not the asserted behavior.

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/schema_priority_when_spec.rb`
Expected: FAIL — `priority_when` is an undeclared top-level property under `additionalProperties: false`, so any doc with it is rejected.

- [ ] **Step 3: Edit the schema (apps/api copy first)**

In `apps/api/config/protocols/schema.json`, add a `priority_when` entry to the top-level `properties` object (alongside `name`/`version`/…/`recommendations`). It reuses the F-03.2 `#/$defs/condition`:

```json
    "priority_when": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["when", "priority"],
        "additionalProperties": false,
        "properties": {
          "when": {
            "oneOf": [
              { "type": "object", "minProperties": 1, "additionalProperties": { "type": "string" } },
              { "$ref": "#/$defs/condition" }
            ]
          },
          "priority": { "type": "integer", "minimum": 1, "maximum": 9 }
        }
      }
    }
```
Keep `additionalProperties: false` at the root, valid JSON, indentation consistent. Do NOT add `priority_when` to the root `required`.

- [ ] **Step 4: Mirror to the other two copies (byte-identical)**

```bash
cp apps/api/config/protocols/schema.json contracts/protocols/schema.json
cp apps/api/config/protocols/schema.json packages/protocols/schema.json
md5 -q contracts/protocols/schema.json packages/protocols/schema.json apps/api/config/protocols/schema.json 2>/dev/null || md5sum contracts/protocols/schema.json packages/protocols/schema.json apps/api/config/protocols/schema.json
```
Expected: three identical hashes.

- [ ] **Step 5: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/schema_priority_when_spec.rb`
Expected: PASS. If JSONSchemer reports the schema itself is invalid, fix the JSON; do NOT weaken the test.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add config/protocols/schema.json spec/protocols/schema_priority_when_spec.rb
git -C apps/api commit -m "Add priority_when to protocol schema contract"
git -C apps/api log --oneline -1
```
NOTE: `contracts/` and `packages/` are at the monorepo root (NOT git). Commit only the apps/api copy + spec; update the other two on disk for parity and report all three md5s (must match).

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec spec/protocols` — expect green.
- Move the board card (F-03.6) to Done, then Verified.
- Sync is a separate explicit step (user-authorized). Note the shared deferred follow-up: publish-time validation of condition/priority_when nodes.

## Self-Review notes

- **Spec coverage:** min-of-matching evaluator → Task 1; escalate-only application + wiring + round-trip → Task 2; schema contract (condition + legacy + range) → Task 3. Deferred validation explicitly excluded.
- **Type consistency:** `PriorityRules.override_for(rules, answers)` defined Task 1, called in `Protocol#apply_priority_when` Task 2; `Protocol.new(priority_rules:)` set Task 2 and fed by `Definitions.build` Task 2; schema `priority_when` shape (Task 3) matches the rules shape the evaluator reads (`when` + `priority`).
- **Escalate-only:** `min(outcome.priority, escalated)` with nil-base handling, asserted (escalate + never-de-escalate + no-op) in Task 2.
- **Reuse:** `Condition.eval` (F-03.2) is the matcher; no duplication.
