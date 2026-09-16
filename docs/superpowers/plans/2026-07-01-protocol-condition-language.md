# Protocol Condition Language (F-03.2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a reusable condition language (eq/in/gt/lt/all/any/not) to the protocol engine and use it for decision-table rule matching, backward-compatible with legacy flat `when` maps.

**Architecture:** A pure `Protocols::Condition.eval(node, answers)` evaluator; `Scoring::DecisionTable#matches?` delegates to it (legacy flat maps evaluate as AND-of-eq); the mirrored `schema.json` contract (3 copies) is widened so a rule `when` accepts the legacy map or a condition node.

**Tech Stack:** Ruby (pure modules), RSpec, JSONSchemer (draft 2020-12). Run specs in the `api-dev` container.

## Global Constraints

- **Commits in English.** `apps/api`, branch `fix/migrations-owner-ddl-as-admin` — commit there, do NOT branch. Use `git -C apps/api ...`.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- Runtime is **total**: `Condition.eval` never raises — a missing/non-numeric operand → `false`; an empty/`nil`/non-Hash/unknown-operator node → `false` (default-deny).
- Backward-compatible: a `when` that is not a single-key operator node is the legacy flat `{step_id => value}` map (AND-of-eq). Existing protocols must keep matching identically.
- Branching is untouched; no migration.
- The 3 `schema.json` copies (`contracts/protocols/`, `packages/protocols/`, `apps/api/config/protocols/`) are byte-identical (ADR-0016) and MUST stay identical.
- **Deferred (do NOT implement):** publish-time validation of condition nodes.

---

### Task 1: `Protocols::Condition` evaluator

**Files:**
- Create: `apps/api/app/protocols/condition.rb`
- Test: `apps/api/spec/protocols/condition_spec.rb` (create)

**Interfaces:**
- Produces: `Protocols::Condition.eval(node, answers) -> Boolean` where `answers` is `{step_id(String) => answer(String)}`. Operators: `{"eq"=>[key,val]}`, `{"in"=>[key,[vals]]}`, `{"gt"=>[key,n]}`, `{"lt"=>[key,n]}`, `{"all"=>[nodes]}`, `{"any"=>[nodes]}`, `{"not"=>node}`; legacy `{step_id=>val,...}` = AND-of-eq.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/condition_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Condition do
  let(:answers) { { "idade" => "70", "febre" => "true", "tosse" => "false" } }
  def ev(node) = described_class.eval(node, answers)

  it "eq: matches exact string, else false" do
    expect(ev({ "eq" => ["febre", "true"] })).to be(true)
    expect(ev({ "eq" => ["febre", "false"] })).to be(false)
  end

  it "in: membership over the set (stringified)" do
    expect(ev({ "in" => ["idade", [70, 80]] })).to be(true)
    expect(ev({ "in" => ["idade", ["10", "20"]] })).to be(false)
  end

  it "gt/lt: numeric comparison" do
    expect(ev({ "gt" => ["idade", 60] })).to be(true)
    expect(ev({ "gt" => ["idade", 90] })).to be(false)
    expect(ev({ "lt" => ["idade", 90] })).to be(true)
  end

  it "gt/lt: missing or non-numeric answer is false (default-deny)" do
    expect(ev({ "gt" => ["ausente", 1] })).to be(false)
    expect(ev({ "gt" => ["febre", 1] })).to be(false) # "true" não é número
  end

  it "all/any/not combinators" do
    expect(ev({ "all" => [{ "gt" => ["idade", 60] }, { "eq" => ["febre", "true"] }] })).to be(true)
    expect(ev({ "all" => [{ "gt" => ["idade", 60] }, { "eq" => ["tosse", "true"] }] })).to be(false)
    expect(ev({ "any" => [{ "eq" => ["tosse", "true"] }, { "eq" => ["febre", "true"] }] })).to be(true)
    expect(ev({ "not" => { "eq" => ["febre", "true"] } })).to be(false)
    expect(ev({ "not" => { "eq" => ["febre", "false"] } })).to be(true)
  end

  it "nested combinators" do
    node = { "any" => [{ "all" => [{ "gt" => ["idade", 65] }, { "eq" => ["febre", "true"] }] },
                       { "eq" => ["tosse", "true"] }] }
    expect(ev(node)).to be(true)
  end

  it "legacy flat map = AND of eq (single and multi key)" do
    expect(ev({ "febre" => "true" })).to be(true)
    expect(ev({ "febre" => "true", "tosse" => "false" })).to be(true)
    expect(ev({ "febre" => "true", "tosse" => "true" })).to be(false)
  end

  it "empty / nil / non-Hash / unknown operator => false (default-deny)" do
    expect(ev({})).to be(false)
    expect(ev(nil)).to be(false)
    expect(ev("x")).to be(false)
    expect(ev({ "zzz" => ["febre", "true"] })).to be(false) # zzz não é operador → mapa legado 1-chave, "febre"!=["febre","true"] stringificado → false
  end
end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/condition_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::Condition`.

- [ ] **Step 3: Implement**

Create `apps/api/app/protocols/condition.rb`:

```ruby
# Avaliador de condições do motor de protocolos. Módulo puro — ver ADR-0013.
# eq/in/gt/lt/all/any/not; um `when` que não seja nó-operador de chave única é
# tratado como mapa legado {step_id => value} (AND de eq). Runtime total:
# nunca levanta — operando ausente/não-numérico ou nó inválido => false.
module Protocols
  module Condition
    OPERATORS = %w[eq in gt lt all any not].freeze

    module_function

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
      return false if map.empty?
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

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/condition_spec.rb`
Expected: PASS (all examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/protocols/condition.rb spec/protocols/condition_spec.rb
git -C apps/api commit -m "Add Protocols::Condition evaluator (eq/in/gt/lt/all/any/not)"
git -C apps/api log --oneline -1
```

---

### Task 2: `DecisionTable` uses `Condition`

**Files:**
- Modify: `apps/api/app/protocols/scoring/decision_table.rb`
- Test: `apps/api/spec/protocols/scoring/decision_table_spec.rb` (create)

**Interfaces:**
- Consumes: `Protocols::Condition.eval` (Task 1).
- Produces: `DecisionTable#call(trail)` matches a rule whose `when` is a condition node OR a legacy flat map, else `fallback`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/scoring/decision_table_spec.rb`. Build the `trail` as the engine does — `[{step:, answer:, weight:}]` (see `spec/protocols/scoring/weighted_spec.rb`); `DecisionTable#call` builds its `answers` from `entry[:step]`/`entry[:answer]`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Scoring::DecisionTable do
  def trail(pairs) = pairs.map { |step, answer| { step: step, answer: answer, weight: 0 } }

  describe "condition-DSL rules" do
    subject(:table) do
      described_class.new(
        rules: [
          { "when" => { "any" => [{ "gt" => ["idade", 60] }, { "eq" => ["febre", "true"] }] },
            "tier" => "alta", "priority" => 1 }
        ],
        fallback: { tier: "baixa", priority: 9 }
      )
    end

    it "matches when the condition holds (gt branch)" do
      outcome = table.call(trail([[:idade, "70"], [:febre, "false"]]))
      expect(outcome.tier).to eq("alta")
      expect(outcome.priority).to eq(1)
    end

    it "matches when the condition holds (eq branch)" do
      outcome = table.call(trail([[:idade, "20"], [:febre, "true"]]))
      expect(outcome.tier).to eq("alta")
    end

    it "falls back when the condition does not hold" do
      outcome = table.call(trail([[:idade, "20"], [:febre, "false"]]))
      expect(outcome.tier).to eq("baixa")
      expect(outcome.priority).to eq(9)
    end
  end

  describe "legacy flat when (regression)" do
    subject(:table) do
      described_class.new(
        rules: [{ "when" => { "febre" => "true" }, "tier" => "alta", "priority" => 1 }],
        fallback: { tier: "baixa", priority: 9 }
      )
    end

    it "still matches exact-equality maps" do
      expect(table.call(trail([[:febre, "true"]])).tier).to eq("alta")
      expect(table.call(trail([[:febre, "false"]])).tier).to eq("baixa")
    end
  end
end
```

If the `DecisionTable.new` keyword/args or `Outcome` accessors differ from the above, align to the real class (read `decision_table.rb` / `outcome.rb`) — the asserted behavior (condition match / fallback / legacy still works) is the requirement.

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/scoring/decision_table_spec.rb`
Expected: FAIL — the condition-DSL rule doesn't match (current `matches?` does exact-equality over the hash keys, so a `when` of `{"any"=>...}` compares `answers["any"]`, never matching).

- [ ] **Step 3: Implement**

In `apps/api/app/protocols/scoring/decision_table.rb`, replace the private `matches?` method body with a delegation to `Condition`:

```ruby
      def matches?(conditions, answers)
        Protocols::Condition.eval(conditions, answers)
      end
```

Leave `call`, `to_h`, `fallback`, and the `answers` construction in `call` unchanged.

- [ ] **Step 4: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/scoring/decision_table_spec.rb`
Expected: PASS (condition + legacy examples).

- [ ] **Step 5: Regression — the engine + weighted specs**

Run: `docker exec api-dev bundle exec rspec spec/protocols`
Expected: green (Condition, DecisionTable, weighted, protocol, validator, gate all pass).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/protocols/scoring/decision_table.rb spec/protocols/scoring/decision_table_spec.rb
git -C apps/api commit -m "Match decision_table rules via Protocols::Condition (condition DSL + legacy)"
git -C apps/api log --oneline -1
```

---

### Task 3: Widen the mirrored `schema.json` `when` contract

**Files:**
- Modify: `contracts/protocols/schema.json`, `packages/protocols/schema.json`, `apps/api/config/protocols/schema.json` (keep byte-identical)
- Test: `apps/api/spec/protocols/schema_contract_spec.rb` (create)

**Interfaces:**
- Produces: the decision-table rule `when` accepts the legacy string-map OR a `#/$defs/condition` node; the schema still loads under JSONSchemer.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/schema_contract_spec.rb`:

```ruby
require "rails_helper"
require "json_schemer"

RSpec.describe "protocols schema.json condition contract (F-03.2)" do
  let(:schema) { JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json")))) }

  def base(rule_when)
    {
      "name" => "c", "version" => 1, "start_step_id" => "s1",
      "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                    "branches" => {}, "weights" => {} }],
      "scoring" => { "type" => "decision_table",
                     "rules" => [{ "when" => rule_when, "tier" => "alta", "priority" => 1 }],
                     "fallback" => { "tier" => "baixa", "priority" => 9 } }
    }
  end

  it "accepts a legacy flat when map" do
    expect(schema.valid?(base({ "s1" => "true" }))).to be(true)
  end

  it "accepts a condition-DSL when node" do
    expect(schema.valid?(base({ "any" => [{ "gt" => ["s1", 60] }, { "eq" => ["s1", "true"] }] }))).to be(true)
  end

  it "accepts nested all/not" do
    expect(schema.valid?(base({ "all" => [{ "not" => { "eq" => ["s1", "false"] } }] }))).to be(true)
  end
end
```

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/schema_contract_spec.rb`
Expected: FAIL — the condition-DSL cases are rejected (current `when` allows only string-valued maps).

- [ ] **Step 3: Edit the schema (one copy first)**

In `apps/api/config/protocols/schema.json`:

(a) Add two `$defs` alongside `step`/`scoring_weighted`/`scoring_decision_table`:

```json
    "condition_operand": {
      "type": "array",
      "prefixItems": [{ "type": "string" }, { "type": ["string", "number"] }],
      "minItems": 2,
      "maxItems": 2
    },
    "condition": {
      "type": "object",
      "minProperties": 1,
      "maxProperties": 1,
      "oneOf": [
        { "required": ["eq"], "additionalProperties": false, "properties": { "eq": { "$ref": "#/$defs/condition_operand" } } },
        { "required": ["gt"], "additionalProperties": false, "properties": { "gt": { "$ref": "#/$defs/condition_operand" } } },
        { "required": ["lt"], "additionalProperties": false, "properties": { "lt": { "$ref": "#/$defs/condition_operand" } } },
        { "required": ["in"], "additionalProperties": false, "properties": { "in": { "type": "array", "prefixItems": [{ "type": "string" }, { "type": "array" }], "minItems": 2, "maxItems": 2 } } },
        { "required": ["all"], "additionalProperties": false, "properties": { "all": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/condition" } } } },
        { "required": ["any"], "additionalProperties": false, "properties": { "any": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/condition" } } } },
        { "required": ["not"], "additionalProperties": false, "properties": { "not": { "$ref": "#/$defs/condition" } } }
      ]
    }
```

(b) Change the decision-table rule `when` (currently `{ "type": "object", "additionalProperties": { "type": "string" } }`) to:

```json
              "when": {
                "oneOf": [
                  { "type": "object", "minProperties": 1, "additionalProperties": { "type": "string" } },
                  { "$ref": "#/$defs/condition" }
                ]
              }
```

Preserve all surrounding JSON (indentation, the `required`/`tier`/`priority` siblings). Keep valid JSON.

- [ ] **Step 4: Mirror to the other two copies (byte-identical)**

Run:
```bash
cp apps/api/config/protocols/schema.json contracts/protocols/schema.json
cp apps/api/config/protocols/schema.json packages/protocols/schema.json
```
Then confirm all three match:
```bash
md5 -q contracts/protocols/schema.json packages/protocols/schema.json apps/api/config/protocols/schema.json
```
(three identical hashes; use `md5sum` if `md5` is unavailable).

- [ ] **Step 5: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/schema_contract_spec.rb`
Expected: PASS (legacy + condition + nested cases). If JSONSchemer rejects the schema itself (e.g. `prefixItems` support), the schema is malformed — fix the JSON, do not weaken the test.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add config/protocols/schema.json spec/protocols/schema_contract_spec.rb
git add contracts/protocols/schema.json packages/protocols/schema.json 2>/dev/null || true
git -C apps/api commit -m "Widen decision_table when contract to accept condition DSL"
git -C apps/api log --oneline -1
```
NOTE: `contracts/` and `packages/` are at the monorepo root, which is NOT a git repo (only `apps/api` is). Stage/commit only what lives under `apps/api` (the `config/protocols/schema.json` copy + the spec); the other two copies are updated on disk for contract parity but are outside git — mention this in the report. Verify with `git -C apps/api status` that the api copy + spec are committed.

---

## Wrap-up (after all tasks)

- Run `docker exec api-dev bundle exec rspec spec/protocols` — expect green.
- Move the board card (F-03.2) to Done, then Verified.
- Sync is a separate explicit step (user-authorized).

## Self-Review notes

- **Spec coverage:** evaluator (all operators + legacy + default-deny) → Task 1; decision_table wiring + legacy regression → Task 2; schema contract (legacy + condition) → Task 3. All spec sections mapped; deferred validation explicitly excluded.
- **Type consistency:** `Condition.eval(node, answers)` signature is used identically in Task 1 (def), Task 2 (`matches?` delegation), and implicitly by the schema shape in Task 3 (operand `[key, value]`, `in` `[key, array]`, `all/any` arrays, `not` single node).
- **Placeholder scan:** none.
- **Runtime totality:** empty/nil/non-Hash/unknown → false is defined in Task 1 code and asserted in Task 1 tests.
- **Contract parity:** Task 3 keeps the 3 copies byte-identical (cp + md5 check); notes the git boundary (only apps/api is versioned).
