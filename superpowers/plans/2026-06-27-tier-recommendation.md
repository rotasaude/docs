# Tier-based Recommendation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the generic placeholder note on the citizen report page with a real, tier-specific recommendation authored in the protocol definition, frozen into the report snapshot.

**Architecture:** A new optional `recommendations` (tier → `{title, body}`) field on the protocol definition is the source of truth. `GenerateReportJob#build_payload` resolves the triage's tier to its recommendation and freezes it into the immutable snapshot payload. `ReportsController#show` passes it through `/r/:token`. The wpda renders title+body, falling back to the existing generic note when absent. The pure engine (`Outcome`/`Scoring`) is untouched.

**Tech Stack:** Rails 8 (RSpec, run in `api-dev` container), React + TypeScript + Vite (Vitest, run on host), JSON Schema.

## Global Constraints

- **Commits in English**, per-app-repo. `apps/api` is its own git repo (branch `fix/migrations-owner-ddl-as-admin`); `apps/wpda` is its own git repo (`main`). The monorepo root is **NOT** a git repo — files under `packages/` and `contracts/` are edited but cannot be committed there.
- **api specs run in the container:** `docker exec api-dev bundle exec rspec <path>`. The container mounts `apps/api` as a volume running the working tree — edits are live, no rebuild.
- **wpda tests/build run on host:** `cd apps/wpda && npm run test` / `npm run build`.
- `tier` is an arbitrary per-protocol string — never hardcode a tier list in the frontend.
- The two JSON schema files `packages/protocols/schema.json` and `contracts/protocols/schema.json` are currently **identical** and must stay in sync.
- Recommendations are **optional**; old snapshots and protocols without them must keep working (frozen `recommendation: nil` → generic fallback).

---

### Task 1: Protocol schema + validator (recommendations field & key↔tier lint)

**Files:**
- Modify: `packages/protocols/schema.json` (add `recommendations` def — root, uncommitted)
- Modify: `contracts/protocols/schema.json` (identical mirror — root, uncommitted)
- Modify: `apps/api/app/protocols/validator.rb`
- Test: `apps/api/spec/protocols/validator_spec.rb` (create)

**Interfaces:**
- Consumes: nothing.
- Produces: `Protocols::Validator.call(definition_hash)` now also rejects a definition whose `recommendations` keys are not all declared tiers. A valid `recommendations` shape is `{ "<tier>" => { "title" => String, "body" => String } }`.

- [ ] **Step 1: Write the failing validator spec**

Create `apps/api/spec/protocols/validator_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Validator do
  def base
    {
      "name" => "triagem-teste",
      "version" => 1,
      "start_step_id" => "s1",
      "steps" => [
        { "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 } }
    }
  end

  it "is valid without recommendations" do
    expect(Protocols::Validator.call(base)).to be_valid
  end

  it "is valid when recommendation keys match declared weighted tiers" do
    d = base.merge("recommendations" => {
      "baixa" => { "title" => "T", "body" => "B" },
      "alta"  => { "title" => "T", "body" => "B" }
    })
    expect(Protocols::Validator.call(d)).to be_valid
  end

  it "rejects a recommendation for an undeclared tier" do
    d = base.merge("recommendations" => { "media" => { "title" => "T", "body" => "B" } })
    result = Protocols::Validator.call(d)
    expect(result).not_to be_valid
    expect(result.errors.join).to include("unknown tier media")
  end

  it "collects decision_table tiers from rules and fallback" do
    d = base.merge(
      "scoring" => {
        "type" => "decision_table",
        "rules" => [{ "when" => { "s1" => "true" }, "tier" => "urgente", "priority" => 1 }],
        "fallback" => { "tier" => "rotina", "priority" => 9 }
      },
      "recommendations" => {
        "urgente" => { "title" => "T", "body" => "B" },
        "rotina"  => { "title" => "T", "body" => "B" }
      }
    )
    expect(Protocols::Validator.call(d)).to be_valid
  end
end
```

- [ ] **Step 2: Run the spec to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validator_spec.rb`
Expected: FAIL — the "rejects a recommendation for an undeclared tier" example fails (current validator ignores `recommendations`, so it is reported valid).

- [ ] **Step 3: Implement the validator change**

In `apps/api/app/protocols/validator.rb`, append `recommendation_errors` to `linter_errors` and add the helper methods. Change the `linter_errors` method to add one line before `errors` is returned:

```ruby
    def linter_errors
      errors = []
      step_ids = definition["steps"].map { |s| s["id"] }.to_set
      errors << "start_step_id refers to unknown step" unless step_ids.include?(definition["start_step_id"])

      definition["steps"].each do |s|
        (s["branches"] || {}).each_value do |next_id|
          next if next_id.nil?
          errors << "step #{s["id"]} branches to unknown step #{next_id}" unless step_ids.include?(next_id)
        end
      end

      errors << "graph has a cycle" if cycle?(definition["steps"])
      errors.concat(recommendation_errors)
      errors
    end
```

Then add these private methods (e.g. right after `linter_errors`):

```ruby
    def recommendation_errors
      recs = definition["recommendations"]
      return [] unless recs.is_a?(Hash)
      tiers = declared_tiers
      recs.keys.reject { |k| tiers.include?(k) }.map { |k| "recommendation for unknown tier #{k}" }
    end

    def declared_tiers
      scoring = definition["scoring"]
      return Set.new unless scoring.is_a?(Hash)
      case scoring["type"]
      when "weighted"
        (scoring["thresholds"] || {}).keys.to_set
      when "decision_table"
        tiers = Array(scoring["rules"]).map { |r| r["tier"] }
        tiers << scoring.dig("fallback", "tier")
        tiers.compact.to_set
      else
        Set.new
      end
    end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validator_spec.rb`
Expected: PASS (4 examples, 0 failures).

- [ ] **Step 5: Update both JSON schema mirrors**

In `packages/protocols/schema.json`, add a top-level property inside `"properties"` (after `"scoring"`):

```jsonc
    "recommendations": {
      "type": "object",
      "description": "tier -> { title, body }. Optional. Author-defined health advice per tier.",
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

Apply the identical edit to `contracts/protocols/schema.json`.

- [ ] **Step 6: Verify the two mirrors are still identical**

Run: `diff packages/protocols/schema.json contracts/protocols/schema.json && echo IDENTICAL`
Expected: prints `IDENTICAL`.

- [ ] **Step 7: Commit (api repo only)**

Note: the schema.json files live at the non-git monorepo root and cannot be committed; only the api files are committed.

```bash
git -C apps/api add app/protocols/validator.rb spec/protocols/validator_spec.rb
git -C apps/api commit -m "Validate recommendation keys map to declared tiers"
```

---

### Task 2: Freeze recommendation into the snapshot payload

**Files:**
- Modify: `apps/api/app/jobs/generate_report_job.rb` (`build_payload`)
- Test: `apps/api/spec/jobs/generate_report_job_spec.rb` (create)

**Interfaces:**
- Consumes: `triage.protocol_definition.definition["recommendations"]` (Task 1 shape), `triage.tier`.
- Produces: snapshot `payload["recommendation"]` is `{ "title" => String, "body" => String }` when the triage's tier has a recommendation, else `nil`.

- [ ] **Step 1: Write the failing job spec**

Create `apps/api/spec/jobs/generate_report_job_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe GenerateReportJob do
  let(:muni) { create(:municipality) }

  def definition_hash(with_recs:)
    base = {
      "name" => "triagem-rec",
      "version" => 1,
      "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                     "priority_map" => { "baixa" => 9, "alta" => 1 } }
    }
    return base unless with_recs
    base.merge("recommendations" => {
      "alta"  => { "title" => "Procure atendimento hoje", "body" => "Seus sintomas indicam prioridade alta." },
      "baixa" => { "title" => "Cuidados em casa", "body" => "Mantenha repouso e hidratacao." }
    })
  end

  def build_triage(tier:, with_recs:)
    pd = ProtocolDefinition.create!(
      name: "triagem-rec", version: 1, status: "active",
      definition: definition_hash(with_recs: with_recs), municipality_id: muni.id
    )
    convo = Conversation.create!(municipality_id: muni.id, phone: "+5511999990000", state: "greeting")
    Triage.create!(
      conversation: convo, protocol_definition: pd, protocol_name: "triagem-rec",
      municipality_id: muni.id, status: "completed", tier: tier, priority: 1,
      completed_at: Time.current,
      outcome: { "trail" => [{ "step" => "tosse", "answer" => "true" }] }
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

  after { Current.reset }

  it "freezes the tier's recommendation into the payload" do
    triage = build_triage(tier: "alta", with_recs: true)
    GenerateReportJob.new.handle(triage_id: triage.id)
    snap = ReportSnapshot.find_by!(triage_id: triage.id)
    expect(snap.payload["recommendation"]).to eq(
      "title" => "Procure atendimento hoje", "body" => "Seus sintomas indicam prioridade alta."
    )
  end

  it "freezes nil when the protocol has no recommendations" do
    triage = build_triage(tier: "alta", with_recs: false)
    GenerateReportJob.new.handle(triage_id: triage.id)
    snap = ReportSnapshot.find_by!(triage_id: triage.id)
    expect(snap.payload).to have_key("recommendation")
    expect(snap.payload["recommendation"]).to be_nil
  end
end
```

- [ ] **Step 2: Run the spec to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/jobs/generate_report_job_spec.rb`
Expected: FAIL — `payload` has no `"recommendation"` key yet.

- [ ] **Step 3: Implement the build_payload change**

In `apps/api/app/jobs/generate_report_job.rb`, replace `build_payload`:

```ruby
  def build_payload(triage)
    recs = triage.protocol_definition.definition["recommendations"]
    {
      tier: triage.tier,
      priority: triage.priority,
      recommendation: recs&.dig(triage.tier),
      summary: triage.outcome.dig("trail")&.map { |e| { step: e["step"], answer: e["answer"] } },
      completed_at: triage.completed_at&.iso8601
    }
  end
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/jobs/generate_report_job_spec.rb`
Expected: PASS (2 examples, 0 failures).

- [ ] **Step 5: Commit (api repo)**

```bash
git -C apps/api add app/jobs/generate_report_job.rb spec/jobs/generate_report_job_spec.rb
git -C apps/api commit -m "Freeze tier recommendation into report snapshot payload"
```

---

### Task 3: Expose recommendation in /r/:token

**Files:**
- Modify: `apps/api/app/controllers/reports_controller.rb` (`show`)
- Test: `apps/api/spec/requests/reports_spec.rb` (create)

**Interfaces:**
- Consumes: snapshot `payload["recommendation"]` (Task 2).
- Produces: `GET /r/:token` JSON now includes `recommendation` (object or `null`).

- [ ] **Step 1: Write the failing request spec**

Create `apps/api/spec/requests/reports_spec.rb`:

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")

RSpec.describe "Reports", type: :request do
  # ReportsController reads via connected_to(role: :admin); needs the real admin
  # connection — see spec/support/admin_rls.
  self.use_transactional_tests = false

  before { clean_admin_tables }
  after  { clean_admin_tables }

  def create_snapshot(payload:)
    as_admin do
      muni = Municipality.create!(name: "Rec City", slug: "rec-city", ibge_code: "3500001")
      pd = ProtocolDefinition.create!(
        name: "triagem-rec", version: 1, status: "active", municipality_id: muni.id,
        definition: {
          "name" => "triagem-rec", "version" => 1, "start_step_id" => "s1",
          "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                        "branches" => { "true" => nil, "false" => nil } }]
        }
      )
      convo = Conversation.create!(municipality_id: muni.id, phone: "+5511999990001", state: "greeting")
      triage = Triage.create!(
        conversation: convo, protocol_definition: pd, protocol_name: "triagem-rec",
        municipality_id: muni.id, status: "completed", tier: "alta", priority: 1,
        completed_at: Time.current, outcome: { "trail" => [] }
      )
      token = ReportSnapshot.mint_token
      ReportSnapshot.create!(
        triage: triage, protocol_definition: pd, municipality_id: muni.id,
        outcome: { "tier" => "alta" }, payload: payload,
        token: token, signature: ReportSnapshot.sign(token), expires_at: 30.days.from_now
      )
    end
  end

  it "includes recommendation in the JSON" do
    snap = create_snapshot(payload: {
      "tier" => "alta", "priority" => 1,
      "recommendation" => { "title" => "Procure atendimento hoje", "body" => "Va a UPA." },
      "summary" => [], "completed_at" => "2026-06-27T12:00:00Z"
    })
    get "/r/#{snap.token}"
    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    expect(body["recommendation"]).to eq("title" => "Procure atendimento hoje", "body" => "Va a UPA.")
  end

  it "returns recommendation nil when absent from the payload" do
    snap = create_snapshot(payload: {
      "tier" => "alta", "priority" => 1, "summary" => [], "completed_at" => nil
    })
    get "/r/#{snap.token}"
    body = JSON.parse(response.body)
    expect(body).to have_key("recommendation")
    expect(body["recommendation"]).to be_nil
  end
end
```

- [ ] **Step 2: Run the spec to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/reports_spec.rb`
Expected: FAIL — response JSON has no `recommendation` key.

- [ ] **Step 3: Implement the controller change**

In `apps/api/app/controllers/reports_controller.rb`, add the `recommendation` line to the rendered JSON:

```ruby
    render json: {
      tier: snapshot.payload["tier"],
      priority: snapshot.payload["priority"],
      recommendation: snapshot.payload["recommendation"],
      summary: snapshot.payload["summary"],
      completed_at: snapshot.payload["completed_at"],
      expires_at: snapshot.expires_at&.iso8601
    }
```

- [ ] **Step 4: Run the spec to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/reports_spec.rb`
Expected: PASS (2 examples, 0 failures).

- [ ] **Step 5: Run the full report-related api suite (regression)**

Run: `docker exec api-dev bundle exec rspec spec/protocols/validator_spec.rb spec/jobs/generate_report_job_spec.rb spec/requests/reports_spec.rb spec/models/report_snapshot_spec.rb`
Expected: PASS, all examples green.

- [ ] **Step 6: Commit (api repo)**

```bash
git -C apps/api add app/controllers/reports_controller.rb spec/requests/reports_spec.rb
git -C apps/api commit -m "Expose tier recommendation in public report endpoint"
```

---

### Task 4: wpda lib — Report interface + recommendation note helper

**Files:**
- Modify: `apps/wpda/src/lib/report.ts`
- Test: `apps/wpda/src/lib/report.test.ts`

**Interfaces:**
- Consumes: `/r/:token` JSON with `recommendation` (Task 3).
- Produces: `Recommendation` type `{ title: string; body: string }`; `Report.recommendation: Recommendation | null`; `reportNote(rec): { title: string | null; body: string }`; `GENERIC_NOTE` constant.

- [ ] **Step 1: Write the failing helper test**

Add to `apps/wpda/src/lib/report.test.ts` (extend the file; add the import to the existing top import or a new import line):

```ts
import { reportNote, GENERIC_NOTE } from "./report";

describe("reportNote", () => {
  it("uses the recommendation when present", () => {
    expect(reportNote({ title: "Procure atendimento", body: "Va a UPA." }))
      .toEqual({ title: "Procure atendimento", body: "Va a UPA." });
  });
  it("falls back to the generic note when null", () => {
    expect(reportNote(null)).toEqual({ title: null, body: GENERIC_NOTE });
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd apps/wpda && npm run test -- src/lib/report.test.ts`
Expected: FAIL — `reportNote` / `GENERIC_NOTE` are not exported (compile/import error).

- [ ] **Step 3: Implement the lib changes**

In `apps/wpda/src/lib/report.ts`, add the `Recommendation` type, extend `Report`, and add the helper + constant:

```ts
export interface TrailEntry { step: string; answer: string }

export interface Recommendation { title: string; body: string }

export interface Report {
  tier: string | null;
  priority: string | null;
  recommendation: Recommendation | null;
  summary: TrailEntry[] | null;
  completed_at: string | null;
  expires_at: string | null;
}

export const GENERIC_NOTE =
  "Este é o resultado da sua triagem. Siga as orientações da sua unidade de saúde. Se os sintomas piorarem, procure atendimento.";

export function reportNote(rec: Recommendation | null): { title: string | null; body: string } {
  if (rec && rec.title && rec.body) return { title: rec.title, body: rec.body };
  return { title: null, body: GENERIC_NOTE };
}
```

(Keep the existing `tokenFromUrl` and `fetchReport` functions unchanged below.)

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd apps/wpda && npm run test -- src/lib/report.test.ts`
Expected: PASS (existing tests + the two new `reportNote` examples).

- [ ] **Step 5: Commit (wpda repo)**

```bash
git -C apps/wpda add src/lib/report.ts src/lib/report.test.ts
git -C apps/wpda commit -m "Add recommendation type and note fallback helper"
```

---

### Task 5: wpda render — Report.tsx uses the recommendation

**Files:**
- Modify: `apps/wpda/src/modules/Report.tsx`

**Interfaces:**
- Consumes: `reportNote`, `Report.recommendation` (Task 4).
- Produces: the report page renders the recommendation title (emphasized) + body, or the generic note when absent.

- [ ] **Step 1: Implement the render change**

In `apps/wpda/src/modules/Report.tsx`:

1. Extend the import on line 2 to include `reportNote`:

```tsx
import { fetchReport, reportNote, type Report as ReportData } from "../lib/report";
```

2. Replace the hardcoded generic `<p>…</p>` block (the paragraph beginning "Este é o resultado da sua triagem.") with the note derived from the recommendation. Just before the `return (` of the success branch (after `const r = state.data;`), compute the note:

```tsx
  const r = state.data;
  const note = reportNote(r.recommendation);
```

Then replace the old paragraph:

```tsx
        <p style={{ fontSize: 15, lineHeight: 1.5, margin: "0 0 24px" }}>
          Este é o resultado da sua triagem. Siga as orientações da sua unidade de saúde.
          Se os sintomas piorarem, procure atendimento.
        </p>
```

with:

```tsx
        {note.title && (
          <p style={{ fontSize: 16, fontWeight: 600, lineHeight: 1.4, margin: "0 0 8px" }}>
            {note.title}
          </p>
        )}
        <p style={{ fontSize: 15, lineHeight: 1.5, margin: "0 0 24px" }}>
          {note.body}
        </p>
```

- [ ] **Step 2: Typecheck + build**

Run: `cd apps/wpda && npm run build`
Expected: `tsc -b` passes (no type errors — `r.recommendation` is a known field) and Vite build succeeds.

- [ ] **Step 3: Run the full wpda test suite (regression)**

Run: `cd apps/wpda && npm run test`
Expected: PASS — all lib tests green, including `reportNote`.

- [ ] **Step 4: Visual verification in the preview**

Confirm in the browser preview that a report with a recommendation shows the bold title + body, and one without falls back to the generic note. (The happy-path needs a real `ReportSnapshot`; if none is seeded, verify via the wpda dev server with a snapshot created through the api, or rely on the unit-tested helper + build.)

- [ ] **Step 5: Commit (wpda repo)**

```bash
git -C apps/wpda add src/modules/Report.tsx
git -C apps/wpda commit -m "Render tier recommendation on the report page"
```

---

## Wrap-up (after all tasks)

- Move the board card `Tier-based recommendation in triage result (F-03.17 follow-up)` (item `PVTI_lADOEbfGRc4BbxiBzgxAHeo`) to **Done**, then **Verified** once the preview check passes.
- Sync of `apps/api` / `apps/wpda` to their remotes is a separate, explicit step (not part of this plan).

## Self-Review notes

- **Spec coverage:** schema field → Task 1 (steps 5-6); engine untouched → no task needed (verified by not modifying `outcome.rb`/`scoring/*`); freeze in snapshot → Task 2; endpoint → Task 3; wpda lib + render → Tasks 4-5; light validation → Task 1 (steps 1-4); tests (job/request/validator/wpda) → Tasks 1-5. All spec sections mapped.
- **Backward compat:** `recs&.dig(tier)` → `nil`; `reportNote(null)` → generic note; both have explicit tests (Task 2 step 1 second example, Task 4 step 1 second example).
- **Type consistency:** `recommendation` is `{title, body}` end-to-end (Ruby symbol keys → JSON string keys → TS `Recommendation`); `reportNote` returns `{title: string|null, body: string}` consumed identically in Task 5.
