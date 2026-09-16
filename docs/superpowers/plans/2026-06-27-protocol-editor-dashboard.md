# Protocol Editor + Live Preview (F-03.12) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give municipal protocol authors a dashboard editor to write a protocol definition, validate it live against the full gate, preview the flow, and save it as a draft.

**Architecture:** A new session-authed, tenant-scoped `Authoring::ProtocolsController` (the `PublicationsController` pattern — NOT `/admin/api`, which is read-only) exposes `gate`, `preview`, and `draft` actions, authorized by `ProtocolPolicy.author?`. A new dashboard module `ProtocolEditor` (state-based nav) consumes them via `lib/api.ts` with the session cookie.

**Tech Stack:** Rails 8 (RSpec, run in the `api-dev` container), React + TypeScript + Vite (Vitest, run on host).

## Global Constraints

- **Commits in English.** `apps/api` work on branch `fix/migrations-owner-ddl-as-admin`; `apps/dashboard` on `main`. Use `git -C apps/api ...` / `git -C apps/dashboard ...` (do not `cd` for git). Do NOT create branches.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. Ruby edits are live (volume); no rebuild.
- **dashboard tests/build run ON THE HOST**: `cd apps/dashboard && npm run test` / `npm run build`.
- Authoring writes must NOT go under `/admin/api` (read-only by acceptance criterion §10 + BYPASSRLS). The new controller is session-authed + tenant-scoped (RLS via the inherited `within_tenant`).
- Authorization: every authoring action requires the author role —
  `ProtocolPolicy.new(Current.user, record).author?` (i.e. `role?(:protocol_author, municipality_id)`).
- A draft may be gate-invalid (work-in-progress); the only save floor is the model's minimal `before_save` Validator. The full gate is enforced only at publish (F-03.9).

---

### Task 1: Authoring controller foundation + `gate` action

**Files:**
- Create: `apps/api/app/controllers/authoring/protocols_controller.rb`
- Modify: `apps/api/config/routes.rb`
- Test: `apps/api/spec/requests/authoring/protocols_gate_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Gate.call(definition) -> Validator::Result`; `ProtocolPolicy`; `Authentication`; `TenantScopedRequest` (inherited `within_tenant`).
- Produces: `POST /authoring/protocols/gate` (session + author-gated) → `{valid:true}` (200) / `{valid:false, errors:[...]}` (422); 401 if unauthenticated; 403 if not an author. The controller exposes private `definition_param` and `require_author!` reused by Tasks 2–3.

- [ ] **Step 1: Add routes**

In `apps/api/config/routes.rb`, add a new scope (next to the existing `scope "/protocols"` block). Tasks 2 and 3 add the `preview` and `draft` routes — start with `gate` only:

```ruby
  # Autoria de protocolo (editor do dashboard) — sessão municipal + RLS + author.
  # Escrita NÃO entra em /admin/api (read-only §10). Ver F-03.12.
  scope "/authoring/protocols" do
    post "gate", to: "authoring/protocols#gate"
  end
```

- [ ] **Step 2: Write the failing request spec**

Create `apps/api/spec/requests/authoring/protocols_gate_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Authoring::Protocols gate", type: :request do
  let!(:muni) { create(:municipality) }

  let(:author) do
    u = User.create!(email_address: "author@example.org", password: "secret123")
    Membership.create!(user: u, municipality: muni, role: "protocol_author", granted_at: Time.current)
    u
  end

  let(:viewer) do
    u = User.create!(email_address: "viewer@example.org", password: "secret123")
    Membership.create!(user: u, municipality: muni, role: "viewer", granted_at: Time.current)
    u
  end

  def sign_in(user)
    session = user.sessions.create!(user_agent: "rspec", ip_address: "127.0.0.1")
    allow_any_instance_of(Authoring::ProtocolsController).to receive(:resume_session) { Current.session = session }
    allow_any_instance_of(Authoring::ProtocolsController).to receive(:current_municipality).and_return(muni)
  end

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

  it "401 when unauthenticated" do
    post "/authoring/protocols/gate", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:unauthorized)
  end

  it "403 when the session is not an author" do
    sign_in(viewer)
    post "/authoring/protocols/gate", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:forbidden)
  end

  it "200 valid:true for a gate-valid definition" do
    sign_in(author)
    post "/authoring/protocols/gate", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)).to eq("valid" => true)
  end

  it "422 with errors for a gate-invalid definition" do
    sign_in(author)
    bad = valid_def
    bad["scoring"]["priority_map"]["baixa"] = 99
    post "/authoring/protocols/gate", params: { definition: bad }, as: :json
    expect(response).to have_http_status(:unprocessable_entity)
    body = JSON.parse(response.body)
    expect(body["valid"]).to be false
    expect(body["errors"]).to be_present
  end
end
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_gate_spec.rb`
Expected: FAIL — routing error / `uninitialized constant Authoring::ProtocolsController`.

- [ ] **Step 4: Implement the controller**

Create `apps/api/app/controllers/authoring/protocols_controller.rb`:

```ruby
# Superfície de autoria de protocolo (editor do dashboard, F-03.12).
# Sessão municipal (ADR-0022) + tenant-scoped (RLS, ADR-0019) + ProtocolPolicy.author?.
# NÃO é /admin/api (read-only §10): aqui há escrita (draft), sob RLS.
module Authoring
  class ProtocolsController < ApplicationController
    include Authentication
    before_action :require_author!

    def gate
      render_gate(Protocols::Gate.call(definition_param))
    end

    private

    def definition_param
      params.require(:definition).to_unsafe_h
    end

    def render_gate(result)
      if result.valid?
        render json: { valid: true }
      else
        render json: { valid: false, errors: result.errors }, status: :unprocessable_entity
      end
    end

    def require_author!
      record = ProtocolDefinition.new(municipality_id: Current.municipality_id)
      head :forbidden unless ProtocolPolicy.new(Current.user, record).author?
    end
  end
end
```

Tasks 2 and 3 add the `preview` and `draft` actions (and their routes/specs) to this controller.

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_gate_spec.rb`
Expected: PASS (4 examples, 0 failures).

- [ ] **Step 6: Commit**

```bash
git -C apps/api add config/routes.rb app/controllers/authoring/protocols_controller.rb spec/requests/authoring/protocols_gate_spec.rb
git -C apps/api commit -m "Add session-authed authoring protocols controller with gate action"
```

---

### Task 2: `preview` action

**Files:**
- Modify: `apps/api/config/routes.rb` (add the `preview` route)
- Modify: `apps/api/app/controllers/authoring/protocols_controller.rb` (add `preview` + `answers_param`)
- Test: `apps/api/spec/requests/authoring/protocols_preview_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Gate.call`; `Protocols::Definitions.build(def).evaluate(answers)`; `require_author!` / `definition_param` / `render_gate` (Task 1).
- Produces: `POST /authoring/protocols/preview` with `{definition, answers}` → `{outcome: {...}}` (200) for a gate-valid definition; `{valid:false, errors:[...]}` (422) for an invalid one.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/requests/authoring/protocols_preview_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Authoring::Protocols preview", type: :request do
  let!(:muni) { create(:municipality) }

  let(:author) do
    u = User.create!(email_address: "author@example.org", password: "secret123")
    Membership.create!(user: u, municipality: muni, role: "protocol_author", granted_at: Time.current)
    u
  end

  def sign_in(user)
    session = user.sessions.create!(user_agent: "rspec", ip_address: "127.0.0.1")
    allow_any_instance_of(Authoring::ProtocolsController).to receive(:resume_session) { Current.session = session }
    allow_any_instance_of(Authoring::ProtocolsController).to receive(:current_municipality).and_return(muni)
  end

  def valid_def
    {
      "name" => "respiratoria", "version" => 1, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "alta" => 5 },
                     "priority_map" => { "baixa" => 9, "alta" => 1 } }
    }
  end

  before { sign_in(author) }

  it "returns the terminal outcome for a complete set of answers" do
    post "/authoring/protocols/preview",
         params: { definition: valid_def, answers: { "tosse" => "true" } }, as: :json
    expect(response).to have_http_status(:ok)
    outcome = JSON.parse(response.body)["outcome"]
    expect(outcome["status"]).to eq("terminal")
    expect(outcome["tier"]).to eq("alta")
  end

  it "returns the pending step when answers are incomplete" do
    post "/authoring/protocols/preview",
         params: { definition: valid_def, answers: {} }, as: :json
    expect(response).to have_http_status(:ok)
    outcome = JSON.parse(response.body)["outcome"]
    expect(outcome["status"]).to eq("pending")
    expect(outcome["awaiting"]).to eq("tosse")
  end

  it "422 with errors when the definition fails the gate" do
    bad = valid_def
    bad["steps"][0]["answer_type"] = "color"
    post "/authoring/protocols/preview",
         params: { definition: bad, answers: {} }, as: :json
    expect(response).to have_http_status(:unprocessable_entity)
    expect(JSON.parse(response.body)["errors"]).to be_present
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_preview_spec.rb`
Expected: FAIL — routing error (no `preview` route / action yet).

- [ ] **Step 3: Add the route and the action**

In `apps/api/config/routes.rb`, add the `preview` route to the authoring scope:

```ruby
  scope "/authoring/protocols" do
    post "gate",    to: "authoring/protocols#gate"
    post "preview", to: "authoring/protocols#preview"
  end
```

In `apps/api/app/controllers/authoring/protocols_controller.rb`, add the `preview` action (after `gate`) and the `answers_param` private helper (next to `definition_param`):

```ruby
    def preview
      result = Protocols::Gate.call(definition_param)
      return render_gate(result) unless result.valid?
      outcome = Protocols::Definitions.build(definition_param).evaluate(answers_param)
      render json: { outcome: outcome.to_h }
    end
```

```ruby
    def answers_param
      params.fetch(:answers, {}).to_unsafe_h
    end
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_preview_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add config/routes.rb app/controllers/authoring/protocols_controller.rb spec/requests/authoring/protocols_preview_spec.rb
git -C apps/api commit -m "Add authoring protocols preview action"
```

---

### Task 3: `Protocols::SaveDraft` command + `draft` action

**Files:**
- Modify: `apps/api/config/routes.rb` (add the `draft` route)
- Modify: `apps/api/app/controllers/authoring/protocols_controller.rb` (add the `draft` action)
- Create: `apps/api/app/commands/protocols/save_draft.rb`
- Test: `apps/api/spec/requests/authoring/protocols_draft_spec.rb`

**Interfaces:**
- Consumes: `ProtocolDefinition`; `ProtocolPolicy`; `Result`; `Current.municipality_id`; `definition_param` (Task 1).
- Produces: `Protocols::SaveDraft.call(definition:, by:) -> Result` — `Result.ok(protocol_definition:)` on success; `Result.fail(:tenant_missing | :forbidden | :version_not_editable | :invalid_definition, message:)`. Upserts a `draft` `ProtocolDefinition` for `(name, version, Current.municipality_id)`; rejects if the version exists in a non-draft status.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/requests/authoring/protocols_draft_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Authoring::Protocols draft", type: :request do
  let!(:muni) { create(:municipality) }

  let(:author) do
    u = User.create!(email_address: "author@example.org", password: "secret123")
    Membership.create!(user: u, municipality: muni, role: "protocol_author", granted_at: Time.current)
    u
  end

  let(:viewer) do
    u = User.create!(email_address: "viewer@example.org", password: "secret123")
    Membership.create!(user: u, municipality: muni, role: "viewer", granted_at: Time.current)
    u
  end

  def sign_in(user)
    session = user.sessions.create!(user_agent: "rspec", ip_address: "127.0.0.1")
    allow_any_instance_of(Authoring::ProtocolsController).to receive(:resume_session) { Current.session = session }
    allow_any_instance_of(Authoring::ProtocolsController).to receive(:current_municipality).and_return(muni)
  end

  def valid_def(version: 1)
    {
      "name" => "respiratoria", "version" => version, "start_step_id" => "tosse",
      "steps" => [
        { "id" => "tosse", "prompt" => "Tosse?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }
  end

  def find_pd(version:)
    ApplicationRecord.connected_to(role: :admin) do
      ProtocolDefinition.find_by(name: "respiratoria", version: version, municipality_id: muni.id)
    end
  end

  it "creates a draft for a new (name, version)" do
    sign_in(author)
    post "/authoring/protocols/draft", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    expect(body["status"]).to eq("draft")
    expect(find_pd(version: 1).status).to eq("draft")
  end

  it "updates the definition of an existing draft" do
    sign_in(author)
    post "/authoring/protocols/draft", params: { definition: valid_def }, as: :json
    changed = valid_def
    changed["steps"][0]["prompt"] = "Está tossindo?"
    post "/authoring/protocols/draft", params: { definition: changed }, as: :json
    expect(response).to have_http_status(:ok)
    expect(find_pd(version: 1).definition["steps"][0]["prompt"]).to eq("Está tossindo?")
  end

  it "422 version_not_editable when the version is already published" do
    ApplicationRecord.connected_to(role: :admin) do
      ProtocolDefinition.create!(name: "respiratoria", version: 1, status: "published",
                                 municipality_id: muni.id, definition: valid_def)
    end
    sign_in(author)
    post "/authoring/protocols/draft", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:unprocessable_entity)
    expect(JSON.parse(response.body)["error"]).to eq("version_not_editable")
  end

  it "403 for a non-author session" do
    sign_in(viewer)
    post "/authoring/protocols/draft", params: { definition: valid_def }, as: :json
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_draft_spec.rb`
Expected: FAIL — routing error (no `draft` route / action yet).

- [ ] **Step 3: Add the route and the draft action**

In `apps/api/config/routes.rb`, add the `draft` route to the authoring scope (now all three):

```ruby
  scope "/authoring/protocols" do
    post "gate",    to: "authoring/protocols#gate"
    post "preview", to: "authoring/protocols#preview"
    post "draft",   to: "authoring/protocols#draft"
  end
```

In `apps/api/app/controllers/authoring/protocols_controller.rb`, add the `draft` action (after `preview`):

```ruby
    def draft
      result = Protocols::SaveDraft.call(definition: definition_param, by: Current.user)
      case result.reason
      when nil
        pd = result.payload[:protocol_definition]
        render json: { id: pd.id, name: pd.name, version: pd.version, status: pd.status }
      when :forbidden
        head :forbidden
      when :version_not_editable
        render json: { error: "version_not_editable", message: result.message }, status: :unprocessable_entity
      else
        render json: { error: "invalid_definition", message: result.message }, status: :unprocessable_entity
      end
    end
```

- [ ] **Step 4: Implement the command**

Create `apps/api/app/commands/protocols/save_draft.rb`:

```ruby
# Cria ou atualiza uma versão DRAFT de protocolo a partir do editor (F-03.12).
# Rascunho é work-in-progress: NÃO exige o gate completo (só a validação mínima
# do before_save). Recusa editar uma versão já publicada/ativa/aposentada.
module Protocols
  module SaveDraft
    EDITABLE = "draft".freeze

    def self.call(definition:, by:)
      return Result.fail(:tenant_missing) if Current.municipality_id.nil?

      record = ProtocolDefinition.find_or_initialize_by(
        name: definition["name"],
        version: definition["version"],
        municipality_id: Current.municipality_id
      )
      return Result.fail(:forbidden) unless ProtocolPolicy.new(by, record).author?

      if record.persisted? && record.status != EDITABLE
        return Result.fail(:version_not_editable, message: "versão #{record.version} está #{record.status}")
      end

      record.status = EDITABLE
      record.definition = definition
      record.save!

      Result.ok(protocol_definition: record)
    rescue ActiveRecord::RecordInvalid => e
      Result.fail(:invalid_definition, message: e.record.errors.full_messages.join(", "))
    end
  end
end
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_draft_spec.rb`
Expected: PASS (4 examples, 0 failures).

- [ ] **Step 6: Run the whole authoring suite (regression)**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring`
Expected: PASS — gate (4) + preview (3) + draft (4) all green.

- [ ] **Step 7: Commit**

```bash
git -C apps/api add config/routes.rb app/controllers/authoring/protocols_controller.rb app/commands/protocols/save_draft.rb spec/requests/authoring/protocols_draft_spec.rb
git -C apps/api commit -m "Add SaveDraft command and draft authoring endpoint"
```

---

### Task 4: dashboard `lib/api.ts` authoring functions

**Files:**
- Modify: `apps/dashboard/src/lib/api.ts`
- Test: `apps/dashboard/src/lib/api.authoring.test.ts`

**Interfaces:**
- Consumes: the `/authoring/protocols/*` endpoints (Tasks 1–3); the existing `jsonFetch`/`ApiError`.
- Produces:
  - `gateProtocol(definition: unknown) -> Promise<{ valid: boolean; errors?: string[] }>`
  - `previewProtocol(definition: unknown, answers: Record<string,string>) -> Promise<{ outcome?: Record<string,unknown>; valid?: boolean; errors?: string[] }>`
  - `saveProtocolDraft(definition: unknown) -> Promise<{ id?: string; name?: string; version?: number; status?: string; error?: string; message?: string }>`

- [ ] **Step 1: Write the failing test**

Create `apps/dashboard/src/lib/api.authoring.test.ts`:

```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { gateProtocol, previewProtocol, saveProtocolDraft } from "./api";

afterEach(() => vi.unstubAllGlobals());

function mockFetch(status: number, body: unknown) {
  vi.stubGlobal("fetch", vi.fn(async () => new Response(
    body === undefined ? "" : JSON.stringify(body),
    { status, headers: { "Content-Type": "application/json" } }
  )));
}

describe("gateProtocol", () => {
  it("valid:true on 200", async () => {
    mockFetch(200, { valid: true });
    expect(await gateProtocol({})).toEqual({ valid: true });
  });
  it("valid:false + errors on 422", async () => {
    mockFetch(422, { valid: false, errors: ["schema: /x"] });
    expect(await gateProtocol({})).toEqual({ valid: false, errors: ["schema: /x"] });
  });
});

describe("previewProtocol", () => {
  it("returns outcome on 200", async () => {
    mockFetch(200, { outcome: { status: "terminal", tier: "alta" } });
    const r = await previewProtocol({}, { tosse: "true" });
    expect(r.outcome?.tier).toBe("alta");
  });
  it("returns errors on 422", async () => {
    mockFetch(422, { valid: false, errors: ["schema: /y"] });
    const r = await previewProtocol({}, {});
    expect(r.errors).toEqual(["schema: /y"]);
  });
});

describe("saveProtocolDraft", () => {
  it("returns the saved descriptor on 200", async () => {
    mockFetch(200, { id: "abc", name: "respiratoria", version: 1, status: "draft" });
    expect((await saveProtocolDraft({})).status).toBe("draft");
  });
  it("maps 422 version_not_editable", async () => {
    mockFetch(422, { error: "version_not_editable", message: "versão 1 está published" });
    expect((await saveProtocolDraft({})).error).toBe("version_not_editable");
  });
  it("maps 403 forbidden", async () => {
    mockFetch(403, undefined);
    expect((await saveProtocolDraft({})).error).toBe("forbidden");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd apps/dashboard && npm run test -- src/lib/api.authoring.test.ts`
Expected: FAIL — `gateProtocol`/`previewProtocol`/`saveProtocolDraft` are not exported.

- [ ] **Step 3: Implement the functions**

Append to `apps/dashboard/src/lib/api.ts`:

```ts
const AUTHORING_BASE = import.meta.env.VITE_AUTHORING_BASE || "/authoring/protocols";

export interface GateResult { valid: boolean; errors?: string[]; }
export interface PreviewResult { outcome?: Record<string, unknown>; valid?: boolean; errors?: string[]; }
export interface DraftResult {
  id?: string; name?: string; version?: number; status?: string;
  error?: string; message?: string;
}

export async function gateProtocol(definition: unknown): Promise<GateResult> {
  try {
    await jsonFetch<unknown>(`${AUTHORING_BASE}/gate`, {
      method: "POST", body: JSON.stringify({ definition })
    });
    return { valid: true };
  } catch (err) {
    if (err instanceof ApiError && err.status === 422) {
      const body = (err.body ?? {}) as GateResult;
      return { valid: false, errors: body.errors ?? [] };
    }
    throw err;
  }
}

export async function previewProtocol(
  definition: unknown,
  answers: Record<string, string>
): Promise<PreviewResult> {
  try {
    return await jsonFetch<PreviewResult>(`${AUTHORING_BASE}/preview`, {
      method: "POST", body: JSON.stringify({ definition, answers })
    });
  } catch (err) {
    if (err instanceof ApiError && err.status === 422) return (err.body ?? {}) as PreviewResult;
    throw err;
  }
}

export async function saveProtocolDraft(definition: unknown): Promise<DraftResult> {
  try {
    return await jsonFetch<DraftResult>(`${AUTHORING_BASE}/draft`, {
      method: "POST", body: JSON.stringify({ definition })
    });
  } catch (err) {
    if (err instanceof ApiError && (err.status === 422 || err.status === 403)) {
      const body = (typeof err.body === "object" && err.body) ? (err.body as DraftResult) : {};
      return { error: "forbidden", ...body };
    }
    throw err;
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd apps/dashboard && npm run test -- src/lib/api.authoring.test.ts`
Expected: PASS (7 examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/dashboard add src/lib/api.ts src/lib/api.authoring.test.ts
git -C apps/dashboard commit -m "Add authoring protocol api client functions"
```

---

### Task 5: editor pure helpers (`lib/editor.ts`)

**Files:**
- Create: `apps/dashboard/src/lib/editor.ts`
- Test: `apps/dashboard/src/lib/editor.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `parseDefinition(text: string) -> { ok: true; value: unknown } | { ok: false; error: string }`
  - `TEMPLATE: string` — a pretty-printed minimal valid definition to seed a new protocol.

- [ ] **Step 1: Write the failing test**

Create `apps/dashboard/src/lib/editor.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { parseDefinition, TEMPLATE } from "./editor";

describe("parseDefinition", () => {
  it("ok for valid JSON", () => {
    const r = parseDefinition('{"a":1}');
    expect(r).toEqual({ ok: true, value: { a: 1 } });
  });
  it("error for invalid JSON", () => {
    const r = parseDefinition("{not json");
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.error).toBeTruthy();
  });
});

describe("TEMPLATE", () => {
  it("parses to a definition with name/version/steps", () => {
    const r = parseDefinition(TEMPLATE);
    expect(r.ok).toBe(true);
    if (r.ok) {
      const v = r.value as Record<string, unknown>;
      expect(v.name).toBeTruthy();
      expect(v.version).toBe(1);
      expect(Array.isArray(v.steps)).toBe(true);
    }
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd apps/dashboard && npm run test -- src/lib/editor.test.ts`
Expected: FAIL — module `./editor` not found.

- [ ] **Step 3: Implement the helpers**

Create `apps/dashboard/src/lib/editor.ts`:

```ts
// Helpers puros do editor de protocolo (F-03.12). Sem dependência de React,
// para serem testáveis isoladamente.
export type ParseResult =
  | { ok: true; value: unknown }
  | { ok: false; error: string };

export function parseDefinition(text: string): ParseResult {
  try {
    return { ok: true, value: JSON.parse(text) };
  } catch (e) {
    return { ok: false, error: e instanceof Error ? e.message : "JSON inválido" };
  }
}

export const TEMPLATE = JSON.stringify(
  {
    name: "novo-protocolo",
    version: 1,
    start_step_id: "s1",
    steps: [
      {
        id: "s1",
        prompt: "Pergunta inicial?",
        answer_type: "boolean",
        branches: { true: null, false: null },
        weights: { true: 0, false: 0 }
      }
    ],
    scoring: { type: "weighted", thresholds: { baixa: 0 }, priority_map: { baixa: 9 } }
  },
  null,
  2
);
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd apps/dashboard && npm run test -- src/lib/editor.test.ts`
Expected: PASS (3 examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/dashboard add src/lib/editor.ts src/lib/editor.test.ts
git -C apps/dashboard commit -m "Add protocol editor pure helpers"
```

---

### Task 6: `ProtocolEditor` dashboard module + nav

**Files:**
- Create: `apps/dashboard/src/modules/ProtocolEditor.tsx`
- Modify: `apps/dashboard/src/shell/modules.ts` (add the `protocol-editor` ModuleId + nav item)
- Modify: `apps/dashboard/src/App.tsx` (import + render case)

**Interfaces:**
- Consumes: `gateProtocol`, `previewProtocol`, `saveProtocolDraft` (Task 4); `parseDefinition`, `TEMPLATE` (Task 5).
- Produces: a navigable "Editor" module under the "Governança" group.

- [ ] **Step 1: Register the module id and nav item**

In `apps/dashboard/src/shell/modules.ts`, add `"protocol-editor"` to the `ModuleId` union (end of the union), and add a nav item to the existing "Governança" group:

```ts
export type ModuleId =
  | "overview" | "ingestion" | "conversations" | "consent"
  | "triages" | "classification" | "protocols" | "events"
  | "queues" | "health" | "protocol-editor";
```

```ts
  { label: "Governança", items: [
    { id: "protocols", label: "Protocolos", icon: "❏" },
    { id: "protocol-editor", label: "Editor de protocolo", icon: "✎" },
    { id: "events", label: "Eventos & auditoria", icon: "❖" }
  ]},
```

- [ ] **Step 2: Implement the editor module**

Create `apps/dashboard/src/modules/ProtocolEditor.tsx`:

```tsx
import { useEffect, useRef, useState } from "react";
import { gateProtocol, previewProtocol, saveProtocolDraft,
  type GateResult, type PreviewResult, type DraftResult } from "../lib/api";
import { parseDefinition, TEMPLATE } from "../lib/editor";

export function ProtocolEditor() {
  const [ text, setText ] = useState<string>(TEMPLATE);
  const [ parseError, setParseError ] = useState<string | null>(null);
  const [ gate, setGate ] = useState<GateResult | null>(null);
  const [ answers, setAnswers ] = useState<string>("{}");
  const [ preview, setPreview ] = useState<PreviewResult | null>(null);
  const [ saved, setSaved ] = useState<DraftResult | null>(null);
  const timer = useRef<number | undefined>(undefined);

  // Live gate: debounced 400ms. Parse errors short-circuit (no network call).
  useEffect(() => {
    window.clearTimeout(timer.current);
    const parsed = parseDefinition(text);
    if (!parsed.ok) { setParseError(parsed.error); setGate(null); return; }
    setParseError(null);
    timer.current = window.setTimeout(() => {
      gateProtocol(parsed.value).then(setGate).catch(() => setGate(null));
    }, 400);
    return () => window.clearTimeout(timer.current);
  }, [ text ]);

  const valid = gate?.valid === true && !parseError;

  function runPreview() {
    const parsed = parseDefinition(text);
    const ans = parseDefinition(answers);
    if (!parsed.ok || !ans.ok) return;
    previewProtocol(parsed.value, ans.value as Record<string, string>).then(setPreview);
  }

  function save() {
    const parsed = parseDefinition(text);
    if (!parsed.ok) return;
    saveProtocolDraft(parsed.value).then(setSaved);
  }

  return (
    <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 16 }}>
      <section>
        <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>Definição (JSON)</h2>
        <textarea
          value={text}
          onChange={e => setText(e.target.value)}
          spellCheck={false}
          style={{ width: "100%", height: 360, fontFamily: "monospace", fontSize: 13 }}
        />
        <div style={{ marginTop: 8 }}>
          {parseError && <p style={{ color: "var(--danger, #c00)" }}>JSON inválido: {parseError}</p>}
          {!parseError && gate?.valid && <p style={{ color: "var(--ok, #2a7) " }}>válido ✓</p>}
          {!parseError && gate && !gate.valid && (
            <ul style={{ color: "var(--danger, #c00)", margin: 0, paddingLeft: 18 }}>
              {(gate.errors ?? []).map((e, i) => <li key={i}>{e}</li>)}
            </ul>
          )}
        </div>
        <button onClick={save} style={{ marginTop: 12 }}>Salvar rascunho</button>
        {saved && (
          <p style={{ marginTop: 8 }}>
            {saved.status
              ? `Salvo: ${saved.name}@${saved.version} (${saved.status})`
              : saved.error === "version_not_editable"
                ? "Essa versão já foi publicada; suba a versão."
                : saved.error === "forbidden"
                  ? "Sem permissão de autoria nesta cidade."
                  : `Erro: ${saved.message ?? saved.error}`}
          </p>
        )}
      </section>

      <section>
        <h2 style={{ fontSize: 16, margin: "0 0 8px" }}>Preview ao vivo</h2>
        <label style={{ fontSize: 13 }}>Respostas (JSON step → resposta)</label>
        <textarea
          value={answers}
          onChange={e => setAnswers(e.target.value)}
          spellCheck={false}
          style={{ width: "100%", height: 80, fontFamily: "monospace", fontSize: 13 }}
        />
        <button onClick={runPreview} disabled={!valid} style={{ marginTop: 8 }}>Pré-visualizar</button>
        {!valid && <p style={{ color: "var(--ink3, #888)", fontSize: 13 }}>Corrija os erros para pré-visualizar.</p>}
        {preview?.outcome && (
          <pre style={{ marginTop: 8, fontSize: 12, background: "var(--surface, #f6f6f6)", padding: 8 }}>
            {JSON.stringify(preview.outcome, null, 2)}
          </pre>
        )}
      </section>
    </div>
  );
}
```

- [ ] **Step 3: Wire the module into App.tsx**

In `apps/dashboard/src/App.tsx`, add the import (next to the other module imports):

```tsx
import { ProtocolEditor } from "./modules/ProtocolEditor";
```

And add the case to `renderModule`'s switch (next to `case "protocols":`):

```tsx
    case "protocol-editor": return <ProtocolEditor />;
```

- [ ] **Step 4: Typecheck + build**

Run: `cd apps/dashboard && npm run build`
Expected: `tsc -b` passes (the new module + nav id typecheck) and Vite build succeeds.

- [ ] **Step 5: Run the full dashboard test suite (regression)**

Run: `cd apps/dashboard && npm run test`
Expected: PASS — all lib tests green, including the new api.authoring and editor specs.

- [ ] **Step 6: Visual verification in the preview**

Start the dashboard dev server (host) and the api (container is already up). Log in as a `protocol_author` of a city (grant the role in dev if needed — see the spec's dev note). Open the "Editor de protocolo" module, confirm: the template validates (válido ✓), introducing `priority_map` value 99 shows schema errors live, preview with `{"tosse":"true"}` shows a terminal outcome, and "Salvar rascunho" reports the saved draft. (If no `protocol_author` membership exists in dev, gate/preview/save return 403 and the editor shows the no-permission notice — grant the role to verify the happy path.)

- [ ] **Step 7: Commit**

```bash
git -C apps/dashboard add src/modules/ProtocolEditor.tsx src/shell/modules.ts src/App.tsx
git -C apps/dashboard commit -m "Add protocol editor module with live gate and preview"
```

---

## Wrap-up (after all tasks)

- api suite: `docker exec api-dev bundle exec rspec` — expect green.
- dashboard: `cd apps/dashboard && npm run test` and `npm run build` — expect green.
- Move board card F-03.12 (item `PVTI_lADOEbfGRc4BbxiBzgw_5lU`) to Done, then Verified after the preview check.
- Sync of `apps/api` / `apps/dashboard` to their remotes is a separate, explicit step.

## Self-Review notes

- **Spec coverage:** session-authed authoring controller (gate/preview/draft) → Tasks 1–3; `/admin/api` untouched (no task modifies it); author authorization → Task 1 `require_author!` + Task 3 command check; draft upsert + reject-published + invalid-422 → Task 3; dashboard api client → Task 4; pure helpers → Task 5; editor module + nav + live gate/preview/save → Task 6. All spec sections mapped.
- **Type consistency:** `gateProtocol/previewProtocol/saveProtocolDraft` signatures and `GateResult/PreviewResult/DraftResult` shapes are defined in Task 4 and consumed identically in Task 6; `parseDefinition/TEMPLATE` defined in Task 5 and consumed in Task 6; `Protocols::SaveDraft.call(definition:, by:)` defined in Task 3 and called by the controller in Task 1.
- **Cross-task note:** each backend task is a clean RED→GREEN — Task 1 adds the `gate` route+action (+ shared private helpers), Task 2 adds the `preview` route+action+`answers_param`, Task 3 adds the `draft` route+action+`SaveDraft` command. No task ships a method referencing a not-yet-defined constant.
- **Placeholder scan:** every code/test step has complete code and exact commands; no TBD/TODO.
- **Auth/tenant:** the controller relies on the inherited `within_tenant` (RLS) + `Authentication`; request specs stub `resume_session` + `current_municipality` exactly like `publications_controller_spec`.
