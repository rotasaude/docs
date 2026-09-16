# Publish-time Condition Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the publish Gate correctly validate condition/priority_when nodes — unblocking valid condition-DSL protocols (a latent rejection bug) and rejecting malformed ones.

**Architecture:** A reusable `Protocols::Validation::Condition` walker validates a `when` node (condition or legacy). `Validation::Scoring.decision_table_errors` is rewritten to use it; a new `Validation::PriorityWhen` validates `priority_when`; both plus a step-id/operator-collision check are composed into `Protocols::Gate`.

**Tech Stack:** Ruby (pure modules), RSpec. Run specs in the `api-dev` container.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- Validation is **publish-only** (via `Protocols::Gate`, called by `Protocols::Publish`). Do NOT change the runtime `Definitions.build` (the engine is total by design). No migration.
- Operator set is `Protocols::Condition::OPERATORS` (`%w[eq in gt lt all any not]`) — reuse it, don't redefine.
- `Protocols::Validation::Answers.for(step)` → allowed answers (`%w[true false]` boolean, `options` enum, `nil` = unconstrained integer/text).
- Namespacing: the new module is `Protocols::Validation::Condition` (the validator) — distinct from `Protocols::Condition` (the runtime evaluator). Reference the operator list as `Protocols::Condition::OPERATORS`.

---

### Task 1: `Protocols::Validation::Condition`

**Files:**
- Create: `apps/api/app/protocols/validation/condition.rb`
- Test: `apps/api/spec/protocols/validation/condition_spec.rb` (create)

**Interfaces:**
- Produces: `Protocols::Validation::Condition.errors(node, by_id) -> [String]` (by_id = `{step_id => step_hash}`); `.step_id_collision_errors(steps) -> [String]`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/validation/condition_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Validation::Condition do
  let(:steps) do
    [
      { "id" => "febre", "answer_type" => "boolean" },
      { "id" => "idade", "answer_type" => "integer" },
      { "id" => "cor", "answer_type" => "enum", "options" => %w[verde amarelo vermelho] }
    ]
  end
  let(:by_id) { steps.to_h { |s| [s["id"], s] } }
  def errs(node) = described_class.errors(node, by_id)

  it "accepts a valid eq / in / gt / lt" do
    expect(errs({ "eq" => ["febre", "true"] })).to eq([])
    expect(errs({ "in" => ["cor", %w[verde amarelo]] })).to eq([])
    expect(errs({ "gt" => ["idade", 60] })).to eq([])
    expect(errs({ "lt" => ["idade", 5] })).to eq([])
  end

  it "flags eq/in on an unknown step or a disallowed answer" do
    expect(errs({ "eq" => ["ausente", "true"] })).not_to be_empty
    expect(errs({ "eq" => ["febre", "talvez"] })).not_to be_empty     # not in %w[true false]
    expect(errs({ "in" => ["cor", %w[verde roxo]] })).not_to be_empty # roxo not an option
  end

  it "requires gt/lt on an integer step" do
    expect(errs({ "gt" => ["febre", 1] })).not_to be_empty  # febre is boolean
    expect(errs({ "lt" => ["cor", 1] })).not_to be_empty    # cor is enum
  end

  it "recurses through all/any/not" do
    expect(errs({ "all" => [{ "gt" => ["idade", 60] }, { "eq" => ["febre", "true"] }] })).to eq([])
    expect(errs({ "any" => [{ "gt" => ["febre", 1] }] })).not_to be_empty  # nested bad node
    expect(errs({ "not" => { "eq" => ["idade", "x"] } })).to be_a(Array)   # recurses (idade unconstrained → [])
    expect(errs({ "all" => "notarray" })).not_to be_empty
  end

  it "validates a legacy flat when map" do
    expect(errs({ "febre" => "true" })).to eq([])
    expect(errs({ "febre" => "nope" })).not_to be_empty
    expect(errs({ "ausente" => "true" })).not_to be_empty
  end

  it "flags a non-hash / empty / malformed-operand node" do
    expect(errs({})).not_to be_empty
    expect(errs(nil)).not_to be_empty
    expect(errs({ "eq" => "notarray" })).not_to be_empty
  end

  it "step_id_collision_errors flags a step named like an operator" do
    expect(described_class.step_id_collision_errors([{ "id" => "eq" }, { "id" => "febre" }])).not_to be_empty
    expect(described_class.step_id_collision_errors([{ "id" => "febre" }])).to eq([])
  end
end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/condition_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::Validation::Condition`.

- [ ] **Step 3: Implement**

Create `apps/api/app/protocols/validation/condition.rb`:

```ruby
# Valida um nó de condição (when) no PUBLISH (Gate). Semântico, além do JSON
# Schema: operador conhecido, gt/lt só em step integer, eq/in no conjunto
# permitido, step referenciado existe. Mapa legado {step=>val} = validação
# por-par. Ver ADR-0013/0017. (chore de validação — F-03.2/F-03.6)
module Protocols
  module Validation
    module Condition
      OPERATORS = Protocols::Condition::OPERATORS

      module_function

      def errors(node, by_id)
        return ["condition must be a non-empty object"] unless node.is_a?(Hash) && node.any?
        return legacy_errors(node, by_id) unless operator_node?(node)

        op, operand = node.first
        case op.to_s
        when "eq", "in" then eq_in_errors(op.to_s, operand, by_id)
        when "gt", "lt" then gt_lt_errors(op.to_s, operand, by_id)
        when "all", "any"
          return ["condition '#{op}' operand must be an array"] unless operand.is_a?(Array)
          operand.flat_map { |sub| errors(sub, by_id) }
        when "not" then errors(operand, by_id)
        else ["unknown condition operator '#{op}'"]
        end
      end

      def step_id_collision_errors(steps)
        Array(steps).filter_map do |s|
          "step id '#{s["id"]}' collides with a condition operator" if OPERATORS.include?(s["id"].to_s)
        end
      end

      def operator_node?(node)
        node.size == 1 && OPERATORS.include?(node.keys.first.to_s)
      end

      def eq_in_errors(op, operand, by_id)
        return ["condition '#{op}' operand must be [step_id, value]"] unless operand.is_a?(Array) && operand.size == 2
        step_id, value = operand
        step = by_id[step_id.to_s]
        return ["condition '#{op}' references unknown step #{step_id}"] if step.nil?
        allowed = Answers.for(step)
        return [] if allowed.nil?
        values = op == "in" ? Array(value) : [value]
        values.map(&:to_s).reject { |v| allowed.include?(v) }
              .map { |v| "condition '#{op}' invalid answer '#{v}' for step #{step_id}" }
      end

      def gt_lt_errors(op, operand, by_id)
        return ["condition '#{op}' operand must be [step_id, number]"] unless operand.is_a?(Array) && operand.size == 2
        step_id, threshold = operand
        step = by_id[step_id.to_s]
        return ["condition '#{op}' references unknown step #{step_id}"] if step.nil?
        errs = []
        errs << "condition '#{op}' requires an integer step, got #{step["answer_type"]} for #{step_id}" unless step["answer_type"] == "integer"
        errs << "condition '#{op}' threshold must be numeric" unless threshold.is_a?(Numeric) || Float(threshold.to_s, exception: false)
        errs
      end

      def legacy_errors(map, by_id)
        map.flat_map do |step_id, answer|
          step = by_id[step_id.to_s]
          next ["decision_table rule references unknown step #{step_id}"] if step.nil?
          allowed = Answers.for(step)
          next [] if allowed.nil?
          allowed.include?(answer.to_s) ? [] : ["decision_table invalid answer '#{answer}' for step #{step_id}"]
        end
      end
    end
  end
end
```

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/condition_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/protocols/validation/condition.rb spec/protocols/validation/condition_spec.rb
git -C apps/api commit -m "Add Protocols::Validation::Condition (publish-time condition linter)"
git -C apps/api log --oneline -1
```

---

### Task 2: Wire into `Scoring`, `PriorityWhen`, and `Gate`

**Files:**
- Modify: `apps/api/app/protocols/validation/scoring.rb`, `apps/api/app/protocols/gate.rb`
- Create: `apps/api/app/protocols/validation/priority_when.rb`
- Test: `apps/api/spec/protocols/gate_spec.rb` (extend), `apps/api/spec/protocols/validation/priority_when_spec.rb` (create)

**Interfaces:**
- Consumes: `Validation::Condition` (Task 1).
- Produces: `Gate.call` rejects malformed condition/priority_when and accepts valid condition-DSL protocols; `Validation::PriorityWhen.call(definition) -> [String]`.

- [ ] **Step 1: Write the failing tests**

Extend `apps/api/spec/protocols/gate_spec.rb` — add a helper for a decision_table def and new examples:

```ruby
  def dt_def(rule_when)
    {
      "name" => "dt", "version" => 1, "start_step_id" => "idade",
      "steps" => [{ "id" => "idade", "prompt" => "Idade?", "answer_type" => "integer", "branches" => {} }],
      "scoring" => { "type" => "decision_table",
                     "rules" => [{ "when" => rule_when, "tier" => "alta", "priority" => 1 }],
                     "fallback" => { "tier" => "baixa", "priority" => 9 } }
    }
  end

  it "accepts a valid condition-DSL when (regression: was wrongly rejected before)" do
    result = Protocols::Gate.call(dt_def({ "gt" => ["idade", 60] }))
    expect(result.valid?).to be(true), result.errors.inspect
  end

  it "rejects gt/lt on a non-integer step" do
    d = dt_def({ "gt" => ["idade", 60] })
    d["steps"][0]["answer_type"] = "boolean"
    # keep the branches valid for boolean so the shape passes the schema
    d["steps"][0]["branches"] = { "true" => nil, "false" => nil }
    result = Protocols::Gate.call(d)
    expect(result.valid?).to be(false)
    expect(result.errors.join).to match(/integer step/)
  end

  it "rejects an eq with a disallowed answer" do
    d = dt_def({ "eq" => ["grave", "sim"] })
    d["steps"] << { "id" => "grave", "prompt" => "Grave?", "answer_type" => "boolean", "branches" => { "true" => nil, "false" => nil } }
    result = Protocols::Gate.call(d)
    expect(result.valid?).to be(false)
    expect(result.errors.join).to match(/invalid answer 'sim' for step grave/)
  end

  it "rejects a step named like an operator" do
    d = dt_def({ "gt" => ["idade", 60] })
    d["steps"] << { "id" => "any", "prompt" => "?", "answer_type" => "boolean", "branches" => { "true" => nil, "false" => nil } }
    result = Protocols::Gate.call(d)
    expect(result.valid?).to be(false)
    expect(result.errors.join).to match(/collides with a condition operator/)
  end

  it "still accepts a legacy flat when" do
    d = dt_def({ "idade" => "60" })  # integer step is unconstrained → allowed
    expect(Protocols::Gate.call(d).valid?).to be(true)
  end
```

NOTE: each `dt_def`-based definition must be schema-valid before the semantic checks run (Gate short-circuits on schema errors). If the schema rejects any fixture for an unrelated reason, adjust the fixture (steps/branches/scoring) so only the intended semantic error remains — read the error to see which layer fired.

Create `apps/api/spec/protocols/validation/priority_when_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Validation::PriorityWhen do
  def defn(priority_when)
    {
      "steps" => [{ "id" => "idade", "answer_type" => "integer" }],
      "priority_when" => priority_when
    }
  end

  it "accepts a valid priority_when" do
    expect(described_class.call(defn([{ "when" => { "gt" => ["idade", 80] }, "priority" => 1 }]))).to eq([])
  end

  it "flags an invalid when" do
    expect(described_class.call(defn([{ "when" => { "gt" => ["ausente", 80] }, "priority" => 1 }]))).not_to be_empty
  end

  it "flags a priority outside 1..9 or missing" do
    expect(described_class.call(defn([{ "when" => { "gt" => ["idade", 80] }, "priority" => 10 }]))).not_to be_empty
    expect(described_class.call(defn([{ "when" => { "gt" => ["idade", 80] } }]))).not_to be_empty
  end

  it "is a no-op without priority_when" do
    expect(described_class.call({ "steps" => [] })).to eq([])
  end
end
```

- [ ] **Step 2: Run to verify they fail**

Run: `docker exec api-dev bundle exec rspec spec/protocols/gate_spec.rb spec/protocols/validation/priority_when_spec.rb`
Expected: FAIL — the condition-DSL `dt_def` is currently rejected ("unknown step gt"); `PriorityWhen` is undefined.

- [ ] **Step 3: Rewrite `Validation::Scoring.decision_table_errors`**

In `apps/api/app/protocols/validation/scoring.rb`, replace `decision_table_errors` with:

```ruby
      def self.decision_table_errors(scoring, steps)
        by_id = steps.to_h { |s| [s["id"], s] }
        Array(scoring["rules"]).flat_map { |rule| Condition.errors(rule["when"] || {}, by_id) }
      end
```
(`Condition` resolves to `Protocols::Validation::Condition` by lexical nesting. `weighted_errors`/`call` unchanged.)

- [ ] **Step 4: Create `Validation::PriorityWhen`**

Create `apps/api/app/protocols/validation/priority_when.rb`:

```ruby
# Valida priority_when no PUBLISH (F-03.6): cada regra tem um `when` válido
# (via Validation::Condition) e priority inteiro em 1..9. Ver ADR-0017.
module Protocols
  module Validation
    module PriorityWhen
      def self.call(definition)
        rules = definition["priority_when"]
        return [] unless rules.is_a?(Array)

        by_id = (definition["steps"] || []).to_h { |s| [s["id"], s] }
        rules.flat_map do |rule|
          errs = Condition.errors(rule["when"] || {}, by_id)
          priority = rule["priority"]
          errs << "priority_when priority must be an integer 1..9" unless priority.is_a?(Integer) && priority.between?(1, 9)
          errs
        end
      end
    end
  end
end
```

- [ ] **Step 5: Compose into `Gate`**

In `apps/api/app/protocols/gate.rb`, after `errors.concat(Validation::Scoring.call(definition))`, add:

```ruby
      errors.concat(Validation::PriorityWhen.call(definition))
      errors.concat(Validation::Condition.step_id_collision_errors(definition["steps"] || []))
```

- [ ] **Step 6: Run to verify they pass**

Run: `docker exec api-dev bundle exec rspec spec/protocols/gate_spec.rb spec/protocols/validation/priority_when_spec.rb`
Expected: PASS (the condition-DSL regression now passes; malformed cases rejected).

- [ ] **Step 7: Full protocols regression**

Run: `docker exec api-dev bundle exec rspec spec/protocols`
Expected: green (existing gate/validator/scoring/condition/priority_rules/decision_table all pass; the rewrite is behavior-preserving for legacy `when`).

- [ ] **Step 8: Commit**

```bash
git -C apps/api add app/protocols/validation/scoring.rb app/protocols/validation/priority_when.rb app/protocols/gate.rb spec/protocols/gate_spec.rb spec/protocols/validation/priority_when_spec.rb
git -C apps/api commit -m "Validate condition/priority_when at the publish gate"
git -C apps/api log --oneline -1
```

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec spec/protocols` — expect green.
- Move the CHORE card to Done, then Verified.
- Sync is a separate explicit step (user-authorized).

## Self-Review notes

- **Spec coverage:** the condition walker (all rules) → Task 1; the Scoring rewrite (unblocks condition `when`, fixes the latent bug) + PriorityWhen + Gate wiring + step-id collision → Task 2. All spec sections mapped.
- **Bug fix:** the "accepts a valid condition-DSL when (regression)" gate test proves the F-03.2 publish-rejection is fixed.
- **Type consistency:** `Validation::Condition.errors(node, by_id)` used by Scoring (Task 2 Step 3) and PriorityWhen (Step 4); `step_id_collision_errors(steps)` used in Gate (Step 5). Operator list reused from `Protocols::Condition::OPERATORS`.
- **Behavior-preserving:** legacy flat `when` validation is the same logic (moved into `Condition.legacy_errors`); the full `spec/protocols` regression covers it.
