# Server-side Protocol Gate (F-03.9) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce a full server-side protocol validation gate (complete JSON Schema + graph linter + scoring semantics) at publish time and on the `/gate` preview endpoint.

**Architecture:** A new `Protocols::Gate` composes three focused units — `Validation::Schema` (json_schemer against a vendored `schema.json`), `Validation::Graph` (reachability + branch/weight key validity), `Validation::Scoring` (decision_table.when references + weighted tier consistency) — and reuses the existing `Protocols::Validator` (refs/cycles/recommendation) by composition. `Protocols::Validator` itself is unchanged and stays in `before_save`. `Protocols::Publish` and `ProtocolsController#gate` call `Gate`.

**Tech Stack:** Rails 8, RSpec (run in the `api-dev` container), `json_schemer` gem, JSON Schema draft 2020-12.

## Global Constraints

- **Commits in English**, on `apps/api` branch `fix/migrations-owner-ddl-as-admin`. Use `git -C apps/api ...` (do not `cd`). Do NOT create a new branch.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. The container mounts `apps/api` as a live volume — Ruby edits take effect immediately, no rebuild.
- **Gem install in the container**: after editing the Gemfile, run `docker exec api-dev bundle install` (the dev image has `BUNDLE_DEPLOYMENT=0`, so this works and updates `Gemfile.lock` on the host volume). The gem lands in the image layer at `/usr/local/bundle` and persists for this container's lifetime; to persist across `docker compose up` recreation, rebuild later with `docker compose build api` (not required for this plan).
- `Protocols::Validator` keeps its current public behavior (`Result` with `valid?`/`errors`); it is reused, not modified.
- The vendored `apps/api/config/protocols/schema.json` mirrors `packages/protocols/schema.json` (and the `contracts/protocols/schema.json` copy). All three must stay in sync — documented manual discipline; the api container cannot read the root copies.
- Tiers are arbitrary per-protocol strings — never hardcode a tier list.

---

### Task 1: json_schemer + vendored schema + `Validation::Schema`

**Files:**
- Modify: `apps/api/Gemfile` (add `gem "json_schemer"`)
- Modify: `apps/api/Gemfile.lock` (via bundle install)
- Create: `apps/api/config/protocols/schema.json` (copy of `packages/protocols/schema.json`)
- Create: `apps/api/app/protocols/validation/schema.rb`
- Test: `apps/api/spec/protocols/validation/schema_spec.rb`

**Interfaces:**
- Consumes: nothing.
- Produces: `Protocols::Validation::Schema.call(definition_hash) -> Array<String>` (empty when shape-valid; each element a human-readable schema error naming the JSON pointer).

- [ ] **Step 1: Add the gem and install it in the container**

Add to `apps/api/Gemfile` (near the other top-level gems, e.g. after the `gem "rails"` line group):

```ruby
# JSON Schema (draft 2020-12) validation for the protocol definition gate (F-03.9).
gem "json_schemer"
```

Run: `docker exec api-dev bundle install`
Expected: resolves and installs `json_schemer` (+ its deps such as `hana`, `regexp_parser`, `bigdecimal`); `Gemfile.lock` updated.

- [ ] **Step 2: Vendor the schema into apps/api**

Run (from the repo root, on the host):

```bash
mkdir -p apps/api/config/protocols
cp packages/protocols/schema.json apps/api/config/protocols/schema.json
```

Verify it is identical:
Run: `diff packages/protocols/schema.json apps/api/config/protocols/schema.json && echo IDENTICAL`
Expected: prints `IDENTICAL`.

- [ ] **Step 3: Write the failing test**

Create `apps/api/spec/protocols/validation/schema_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Validation::Schema do
  def valid_def
    {
      "name" => "respiratoria", "version" => 1, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }
  end

  it "accepts a valid definition" do
    expect(Protocols::Validation::Schema.call(valid_def)).to eq([])
  end

  it "rejects an invalid answer_type" do
    d = valid_def
    d["steps"][0]["answer_type"] = "color"
    expect(Protocols::Validation::Schema.call(d)).not_to be_empty
  end

  it "rejects a priority outside 1..9" do
    d = valid_def
    d["scoring"]["priority_map"]["baixa"] = 99
    expect(Protocols::Validation::Schema.call(d)).not_to be_empty
  end

  it "rejects an unknown top-level property (additionalProperties: false)" do
    expect(Protocols::Validation::Schema.call(valid_def.merge("bogus" => true))).not_to be_empty
  end
end
```

- [ ] **Step 4: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/schema_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::Validation::Schema` (module not created yet).

- [ ] **Step 5: Implement `Validation::Schema`**

Create `apps/api/app/protocols/validation/schema.rb`:

```ruby
# Full JSON Schema (draft 2020-12) validation against the vendored contract.
# The schema file mirrors packages/protocols/schema.json (ADR-0016).
module Protocols
  module Validation
    module Schema
      PATH = Rails.root.join("config/protocols/schema.json")

      def self.call(definition)
        schemer.validate(definition || {}).map do |error|
          pointer = error["data_pointer"].presence || "(root)"
          "schema: #{pointer} #{error["type"]}"
        end
      end

      def self.schemer
        @schemer ||= JSONSchemer.schema(JSON.parse(File.read(PATH)))
      end
    end
  end
end
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/schema_spec.rb`
Expected: PASS (4 examples, 0 failures).

- [ ] **Step 7: Commit**

```bash
git -C apps/api add Gemfile Gemfile.lock config/protocols/schema.json app/protocols/validation/schema.rb spec/protocols/validation/schema_spec.rb
git -C apps/api commit -m "Add json_schemer schema validation for protocol gate"
```

---

### Task 2: `Validation::Answers` + `Validation::Graph` (reachability + key validity)

**Files:**
- Create: `apps/api/app/protocols/validation/answers.rb`
- Create: `apps/api/app/protocols/validation/graph.rb`
- Test: `apps/api/spec/protocols/validation/graph_spec.rb`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `Protocols::Validation::Answers.for(step_hash) -> Array<String> | nil` — valid answer values for a step (`%w[true false]` for boolean, the `options` array for enum), or `nil` when unconstrained (integer/text).
  - `Protocols::Validation::Graph.call(definition_hash) -> Array<String>` — errors for unreachable steps and for `branches`/`weights` keys that are not valid answers for the step's `answer_type`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/validation/graph_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Validation::Graph do
  def reachable_def
    {
      "start_step_id" => "a",
      "steps" => [
        { "id" => "a", "answer_type" => "boolean", "branches" => { "true" => "b", "false" => nil } },
        { "id" => "b", "answer_type" => "boolean", "branches" => { "true" => nil, "false" => nil } }
      ]
    }
  end

  it "accepts a fully reachable graph with valid keys" do
    expect(Protocols::Validation::Graph.call(reachable_def)).to eq([])
  end

  it "flags a step unreachable from start_step_id" do
    d = reachable_def
    d["steps"] << { "id" => "orphan", "answer_type" => "boolean", "branches" => { "true" => nil, "false" => nil } }
    expect(Protocols::Validation::Graph.call(d)).to include("unreachable step: orphan")
  end

  it "flags a branch key that is not valid for a boolean step" do
    d = reachable_def
    d["steps"][0]["branches"]["maybe"] = nil
    expect(Protocols::Validation::Graph.call(d)).to include(a_string_matching(/branch key 'maybe' invalid for boolean step a/))
  end

  it "flags a weight key outside the enum options" do
    d = {
      "start_step_id" => "s",
      "steps" => [
        { "id" => "s", "answer_type" => "enum", "options" => %w[low high],
          "branches" => { "low" => nil, "high" => nil }, "weights" => { "low" => 1, "mid" => 2 } }
      ]
    }
    expect(Protocols::Validation::Graph.call(d)).to include(a_string_matching(/weight key 'mid' invalid for enum step s/))
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/graph_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::Validation::Graph`.

- [ ] **Step 3: Implement `Validation::Answers`**

Create `apps/api/app/protocols/validation/answers.rb`:

```ruby
# Valid answer values for a step, shared by the graph and scoring linters.
module Protocols
  module Validation
    module Answers
      BOOLEAN = %w[true false].freeze

      # Returns the allowed answer strings for a step, or nil when the answer
      # space is unconstrained (integer/text) and key validation does not apply.
      def self.for(step)
        case step["answer_type"]
        when "boolean" then BOOLEAN
        when "enum" then Array(step["options"]).map(&:to_s)
        end
      end
    end
  end
end
```

- [ ] **Step 4: Implement `Validation::Graph`**

Create `apps/api/app/protocols/validation/graph.rb`:

```ruby
# Semantic graph checks beyond JSON Schema and beyond Validator's refs/cycles:
# step reachability from start_step_id, and branches/weights key validity per
# answer_type.
module Protocols
  module Validation
    module Graph
      def self.call(definition)
        steps = definition["steps"] || []
        unreachable_errors(definition["start_step_id"], steps) + key_validity_errors(steps)
      end

      def self.unreachable_errors(start_id, steps)
        reachable = reachable_ids(start_id, steps)
        steps.map { |s| s["id"] }
             .reject { |id| reachable.include?(id) }
             .map { |id| "unreachable step: #{id}" }
      end

      def self.reachable_ids(start_id, steps)
        by_id = steps.to_h { |s| [s["id"], s] }
        seen = Set.new
        queue = [start_id].compact
        until queue.empty?
          id = queue.shift
          next if seen.include?(id)
          seen << id
          step = by_id[id]
          next unless step
          (step["branches"] || {}).values.compact.each { |nxt| queue << nxt }
        end
        seen
      end

      def self.key_validity_errors(steps)
        steps.flat_map do |step|
          allowed = Answers.for(step)
          next [] if allowed.nil?
          { "branches" => "branch", "weights" => "weight" }.flat_map do |field, label|
            (step[field] || {}).keys.reject { |k| allowed.include?(k.to_s) }
              .map { |k| "#{label} key '#{k}' invalid for #{step["answer_type"]} step #{step["id"]}" }
          end
        end
      end
    end
  end
end
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/graph_spec.rb`
Expected: PASS (4 examples, 0 failures).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/protocols/validation/answers.rb app/protocols/validation/graph.rb spec/protocols/validation/graph_spec.rb
git -C apps/api commit -m "Add graph reachability and answer-key linter for protocol gate"
```

---

### Task 3: `Validation::Scoring` (decision_table.when + weighted tier consistency)

**Files:**
- Create: `apps/api/app/protocols/validation/scoring.rb`
- Test: `apps/api/spec/protocols/validation/scoring_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Validation::Answers.for(step)` (Task 2).
- Produces: `Protocols::Validation::Scoring.call(definition_hash) -> Array<String>` — errors for `decision_table` `when` clauses referencing an unknown step or an invalid answer, and for `weighted` `priority_map` tiers not present in `thresholds`. Empty when there is no `scoring` block.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/validation/scoring_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Validation::Scoring do
  def steps
    [{ "id" => "tosse", "answer_type" => "boolean", "branches" => { "true" => nil, "false" => nil } }]
  end

  it "returns no errors when there is no scoring block" do
    expect(Protocols::Validation::Scoring.call({ "steps" => steps })).to eq([])
  end

  it "accepts a weighted scoring whose priority_map tiers are all thresholds" do
    d = { "steps" => steps,
          "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                         "priority_map" => { "baixa" => 9, "alta" => 1 } } }
    expect(Protocols::Validation::Scoring.call(d)).to eq([])
  end

  it "flags a priority_map tier not present in thresholds" do
    d = { "steps" => steps,
          "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 },
                         "priority_map" => { "media" => 5 } } }
    expect(Protocols::Validation::Scoring.call(d)).to include("priority_map tier 'media' not in thresholds")
  end

  it "flags a decision_table when-clause referencing an unknown step" do
    d = { "steps" => steps,
          "scoring" => { "type" => "decision_table",
                         "rules" => [{ "when" => { "ausente" => "true" }, "tier" => "alta", "priority" => 1 }] } }
    expect(Protocols::Validation::Scoring.call(d)).to include("decision_table rule references unknown step ausente")
  end

  it "flags a decision_table when-clause with an invalid answer for the step" do
    d = { "steps" => steps,
          "scoring" => { "type" => "decision_table",
                         "rules" => [{ "when" => { "tosse" => "maybe" }, "tier" => "alta", "priority" => 1 }] } }
    expect(Protocols::Validation::Scoring.call(d)).to include("decision_table invalid answer 'maybe' for step tosse")
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/scoring_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::Validation::Scoring`.

- [ ] **Step 3: Implement `Validation::Scoring`**

Create `apps/api/app/protocols/validation/scoring.rb`:

```ruby
# Scoring semantics beyond JSON Schema: decision_table.when references and
# weighted priority_map / thresholds tier consistency.
module Protocols
  module Validation
    module Scoring
      def self.call(definition)
        scoring = definition["scoring"]
        return [] unless scoring.is_a?(Hash)

        case scoring["type"]
        when "weighted" then weighted_errors(scoring)
        when "decision_table" then decision_table_errors(scoring, definition["steps"] || [])
        else []
        end
      end

      def self.weighted_errors(scoring)
        thresholds = (scoring["thresholds"] || {}).keys.to_set
        (scoring["priority_map"] || {}).keys
          .reject { |tier| thresholds.include?(tier) }
          .map { |tier| "priority_map tier '#{tier}' not in thresholds" }
      end

      def self.decision_table_errors(scoring, steps)
        by_id = steps.to_h { |s| [s["id"], s] }
        Array(scoring["rules"]).flat_map do |rule|
          (rule["when"] || {}).flat_map do |step_id, answer|
            step = by_id[step_id]
            next ["decision_table rule references unknown step #{step_id}"] if step.nil?

            allowed = Answers.for(step)
            next [] if allowed.nil?
            allowed.include?(answer.to_s) ? [] : ["decision_table invalid answer '#{answer}' for step #{step_id}"]
          end
        end
      end
    end
  end
end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validation/scoring_spec.rb`
Expected: PASS (5 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/protocols/validation/scoring.rb spec/protocols/validation/scoring_spec.rb
git -C apps/api commit -m "Add scoring semantics linter for protocol gate"
```

---

### Task 4: `Protocols::Gate` (compose the units)

**Files:**
- Create: `apps/api/app/protocols/gate.rb`
- Test: `apps/api/spec/protocols/gate_spec.rb`

**Interfaces:**
- Consumes: `Validation::Schema.call`, `Validation::Graph.call`, `Validation::Scoring.call` (Tasks 1–3); `Protocols::Validator.call(def).errors` (existing — refs/cycles/recommendation); `Protocols::Validator::Result`.
- Produces: `Protocols::Gate.call(definition_hash) -> Protocols::Validator::Result` (responds to `valid?` and `errors`). When the JSON Schema fails, returns only the schema errors (semantic checks assume valid shape).

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/protocols/gate_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Gate do
  def valid_def
    {
      "name" => "respiratoria", "version" => 1, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }
  end

  it "passes a fully valid definition" do
    result = Protocols::Gate.call(valid_def)
    expect(result.valid?).to be true
    expect(result.errors).to eq([])
  end

  it "returns only schema errors when the shape is invalid (short-circuits semantics)" do
    d = valid_def
    d["steps"][0]["answer_type"] = "color"      # schema violation
    d["steps"] << { "id" => "orphan", "answer_type" => "boolean", "branches" => {} } # would be a graph error
    result = Protocols::Gate.call(d)
    expect(result.valid?).to be false
    expect(result.errors).to all(start_with("schema:"))
  end

  it "aggregates semantic errors from graph and scoring when the shape is valid" do
    d = valid_def
    d["steps"] << { "id" => "orphan", "answer_type" => "boolean",
                    "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 1, "false" => 0 } }
    d["scoring"]["priority_map"] = { "media" => 5 }   # tier not in thresholds
    result = Protocols::Gate.call(d)
    expect(result.valid?).to be false
    expect(result.errors).to include("unreachable step: orphan")
    expect(result.errors).to include("priority_map tier 'media' not in thresholds")
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/gate_spec.rb`
Expected: FAIL — `uninitialized constant Protocols::Gate`.

- [ ] **Step 3: Implement `Protocols::Gate`**

Create `apps/api/app/protocols/gate.rb`:

```ruby
# Full publish/preview quality gate for a protocol definition (F-03.9).
# Composes the JSON Schema check, the graph and scoring semantic linters, and
# reuses Protocols::Validator (refs/cycles/recommendation). Returns a
# Protocols::Validator::Result. If the shape (schema) is invalid, semantic
# checks are skipped — they assume a valid shape.
module Protocols
  module Gate
    def self.call(definition)
      definition ||= {}

      schema_errors = Validation::Schema.call(definition)
      return Validator::Result.new(errors: schema_errors) if schema_errors.any?

      errors = []
      errors.concat(Validator.call(definition).errors)   # refs, cycles, recommendation↔tier
      errors.concat(Validation::Graph.call(definition))
      errors.concat(Validation::Scoring.call(definition))
      Validator::Result.new(errors: errors)
    end
  end
end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/gate_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/protocols/gate.rb spec/protocols/gate_spec.rb
git -C apps/api commit -m "Compose protocol gate from schema, graph, scoring and validator"
```

---

### Task 5: Enforce the gate in `Protocols::Publish`

**Files:**
- Modify: `apps/api/app/commands/protocols/publish.rb`
- Test: `apps/api/spec/commands/protocols_publish_gate_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Gate.call(definition).valid?`/`.errors` (Task 4).
- Produces: `Protocols::Publish.call` returns `Result.fail(:invalid, message: <joined gate errors>)` when the target definition fails the gate; otherwise unchanged (transitions to `published`).

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/commands/protocols_publish_gate_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Protocols::Publish gate (F-03.9)" do
  let(:muni) { create(:municipality) }

  let(:publisher) do
    u = User.create!(email_address: "pub@example.org", password: "secret123")
    Membership.create!(user: u, municipality: muni, role: "protocol_publisher", granted_at: Time.current)
    u
  end

  # Passes the minimal before_save Validator (so it can be saved as in_review),
  # but fails the full gate: priority_map value 99 is out of the schema's 1..9.
  def invalid_definition
    {
      "name" => "respiratoria", "version" => 1, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 99 } }
    }
  end

  def valid_definition
    invalid_definition.merge(
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    )
  end

  def make_pd(definition)
    ProtocolDefinition.create!(
      municipality_id: muni.id, name: "respiratoria", version: 1,
      status: "in_review", definition: definition
    )
  end

  around do |ex|
    ApplicationRecord.transaction do
      Current.municipality_id = muni.id
      ApplicationRecord.connection.execute(
        ApplicationRecord.sanitize_sql(["SET LOCAL app.municipality_id = ?", muni.id])
      )
      ex.run
      raise ActiveRecord::Rollback
    end
  end

  after { Current.reset; Rails.cache.clear }

  it "rejects publishing a definition that fails the gate" do
    pd = make_pd(invalid_definition)
    result = Protocols::Publish.call(version: 1, by: publisher)
    expect(result.failure?).to be true
    expect(result.reason).to eq(:invalid)
    expect(pd.reload.status).to eq("in_review")
  end

  it "publishes a definition that passes the gate" do
    pd = make_pd(valid_definition)
    result = Protocols::Publish.call(version: 1, by: publisher)
    expect(result.ok?).to be true
    expect(pd.reload.status).to eq("published")
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/commands/protocols_publish_gate_spec.rb`
Expected: FAIL — the first example fails because `Publish` currently does not run the gate, so an invalid definition is published (`reason` is `nil`/`ok?` true, status becomes `published`).

- [ ] **Step 3: Implement the gate check in `Publish`**

In `apps/api/app/commands/protocols/publish.rb`, insert the gate check between the `PUBLISHABLE_FROM` guard and the `ApplicationRecord.transaction do` block:

```ruby
      unless PUBLISHABLE_FROM.include?(protocol.status)
        return Result.fail(:invalid_state, message: "só draft/in_review pode ser publicado (está #{protocol.status})")
      end

      gate = Protocols::Gate.call(protocol.definition)
      return Result.fail(:invalid, message: gate.errors.join("; ")) unless gate.valid?

      ApplicationRecord.transaction do
        protocol.update!(status: "published")
```

(Leave the rest of the method — the `DomainEvents.publish` call and the `rescue` — unchanged.)

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/commands/protocols_publish_gate_spec.rb`
Expected: PASS (2 examples, 0 failures).

- [ ] **Step 5: Run the existing lifecycle spec (regression)**

Run: `docker exec api-dev bundle exec rspec spec/commands/protocols_lifecycle_spec.rb`
Expected: PASS — the existing publish/activate/retire flows still work (their definitions are gate-valid).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/commands/protocols/publish.rb spec/commands/protocols_publish_gate_spec.rb
git -C apps/api commit -m "Enforce full validation gate before publishing a protocol"
```

---

### Task 6: Upgrade the `/gate` endpoint to the full gate

**Files:**
- Modify: `apps/api/app/controllers/protocols_controller.rb:31` (the `gate` action)
- Test: `apps/api/spec/requests/protocols_gate_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Gate.call(definition).valid?`/`.errors` (Task 4).
- Produces: `POST /protocols/:name/gate` returns `{ valid: true }` (200) for a gate-valid definition and `{ valid: false, errors: [...] }` (422) otherwise. The `authenticate_author!` before_action only requires an `Authorization` header to be present.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/requests/protocols_gate_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "POST /protocols/:name/gate", type: :request do
  let(:headers) { { "Authorization" => "Bearer any-token" } }

  def valid_def
    {
      "name" => "respiratoria", "version" => 1, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }
  end

  it "returns valid:true for a gate-valid definition" do
    post "/protocols/respiratoria/gate", params: { definition: valid_def }, as: :json, headers: headers
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)).to eq("valid" => true)
  end

  it "returns 422 with errors for a definition that fails the gate" do
    bad = valid_def
    bad["scoring"]["priority_map"]["baixa"] = 99   # out of schema range
    post "/protocols/respiratoria/gate", params: { definition: bad }, as: :json, headers: headers
    expect(response).to have_http_status(:unprocessable_entity)
    body = JSON.parse(response.body)
    expect(body["valid"]).to be false
    expect(body["errors"]).to be_present
  end

  it "requires an Authorization header" do
    post "/protocols/respiratoria/gate", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/protocols_gate_spec.rb`
Expected: FAIL — the second example fails: the current `gate` action uses `Protocols::Validator` (minimal), which does not look at `scoring`, so `priority_map` 99 is accepted (returns 200 `valid:true` instead of 422).

- [ ] **Step 3: Implement the controller change**

In `apps/api/app/controllers/protocols_controller.rb`, change the `gate` action's validator call from `Protocols::Validator` to `Protocols::Gate`:

```ruby
  # POST /protocols/:name/gate — valida uma definição candidata (gate completo)
  def gate
    definition = params.require(:definition).to_unsafe_h
    result = Protocols::Gate.call(definition)

    if result.valid?
      render json: { valid: true }
    else
      render json: { valid: false, errors: result.errors }, status: :unprocessable_entity
    end
  end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/protocols_gate_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 5: Run the full protocol-gate suite (regression)**

Run: `docker exec api-dev bundle exec rspec spec/protocols spec/commands/protocols_publish_gate_spec.rb spec/commands/protocols_lifecycle_spec.rb spec/requests/protocols_gate_spec.rb`
Expected: PASS, all green.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/controllers/protocols_controller.rb spec/requests/protocols_gate_spec.rb
git -C apps/api commit -m "Use full gate on the protocol /gate preview endpoint"
```

---

## Wrap-up (after all tasks)

- Run the whole api suite once: `docker exec api-dev bundle exec rspec` — expect green.
- Move board card F-03.9 (item `PVTI_lADOEbfGRc4BbxiBzgw_5jg`) to Done, then Verified.
- Sync of `apps/api` to its remote is a separate, explicit step (not part of this plan).

## Self-Review notes

- **Spec coverage:** schema via json_schemer + vendored copy → Task 1; graph reachability + branch/weight key validity → Task 2; scoring decision_table.when + priority_map tiers → Task 3; `Gate` composition reusing `Validator` → Task 4; publish enforcement → Task 5; `/gate` endpoint → Task 6. `before_save`/`Validator` unchanged (no task touches them). All spec sections mapped.
- **Type consistency:** `Validation::{Schema,Graph,Scoring}.call -> Array<String>`; `Answers.for -> Array<String>|nil` (defined Task 2, consumed Task 3); `Gate.call -> Validator::Result` consumed identically in Tasks 5–6.
- **Placeholder scan:** every code/test step contains complete code and exact commands; no TBD/TODO.
- **Reuse note:** `Gate` reuses `Validator.call` by composition — `Validator` is not modified, satisfying the spec's "behavior unchanged".
