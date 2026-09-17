# Protocol Editor Load Picker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a protocol author load an existing definition into the dashboard editor via a picker, instead of only the template.

**Architecture:** A new author-gated read action `GET /authoring/protocols/definition` returns the raw definition for a `(name, version)` in the tenant. The dashboard editor adds a `<select>` populated from the existing `GET /admin/api/protocols` metadata list; selecting an item fetches its definition and loads it into the textarea (the existing live gate re-runs).

**Tech Stack:** Rails 8 (RSpec in the `api-dev` container), React + TypeScript + Vite (Vitest on host).

## Global Constraints

- **Commits in English** (UI strings stay Portuguese). `apps/api` on branch `fix/migrations-owner-ddl-as-admin`; `apps/dashboard` on `main`. Use `git -C apps/api ...` / `git -C apps/dashboard ...` (do not `cd` for git). Do NOT create branches.
- **api specs run IN THE CONTAINER**: `docker exec api-dev bundle exec rspec <path>`. Ruby edits are live; no rebuild.
- **dashboard build/tests run ON THE HOST**: `cd apps/dashboard && npm run build` / `npm run test`.
- The read action lives on the authoring surface (session + tenant/RLS + author-gated via the existing `require_author!`), NOT `/admin/api`.

---

### Task 1: `GET /authoring/protocols/definition` read action

**Files:**
- Modify: `apps/api/config/routes.rb` (add the route to the `/authoring/protocols` scope)
- Modify: `apps/api/app/controllers/authoring/protocols_controller.rb` (add the `definition` action)
- Test: `apps/api/spec/requests/authoring/protocols_definition_spec.rb`

**Interfaces:**
- Consumes: `ProtocolDefinition`; `Current.municipality_id`; the existing `require_author!` before_action.
- Produces: `GET /authoring/protocols/definition?name=&version=` → `{ definition: <raw hash> }` (200) when found in the tenant; 404 when not found; 403 non-author; 401 unauth.

- [ ] **Step 1: Add the route**

In `apps/api/config/routes.rb`, inside the existing `scope "/authoring/protocols"` block, add:

```ruby
    get  "definition", to: "authoring/protocols#definition"
```

- [ ] **Step 2: Write the failing request spec**

Create `apps/api/spec/requests/authoring/protocols_definition_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Authoring::Protocols definition", type: :request do
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

  def create_pd
    ApplicationRecord.connection.execute(
      ApplicationRecord.sanitize_sql(["SET LOCAL app.municipality_id = ?", muni.id])
    )
    ProtocolDefinition.create!(
      name: "respiratoria", version: 1, status: "draft", municipality_id: muni.id,
      definition: {
        "name" => "respiratoria", "version" => 1, "start_step_id" => "s1",
        "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean",
                      "branches" => { "true" => nil, "false" => nil } }]
      }
    )
  end

  it "returns the raw definition for an existing (name, version)" do
    sign_in(author)
    create_pd
    get "/authoring/protocols/definition", params: { name: "respiratoria", version: 1 }
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body).dig("definition", "start_step_id")).to eq("s1")
  end

  it "404 for an unknown definition" do
    sign_in(author)
    get "/authoring/protocols/definition", params: { name: "nope", version: 9 }
    expect(response).to have_http_status(:not_found)
  end

  it "403 for a non-author session" do
    sign_in(viewer)
    get "/authoring/protocols/definition", params: { name: "respiratoria", version: 1 }
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_definition_spec.rb`
Expected: FAIL — routing error (no `definition` route/action yet).

- [ ] **Step 4: Implement the action**

In `apps/api/app/controllers/authoring/protocols_controller.rb`, add a public `definition` action (next to `gate`/`preview`/`draft`, above the `private` keyword):

```ruby
    def definition
      record = ProtocolDefinition.find_by(
        name: params[:name], version: params[:version], municipality_id: Current.municipality_id
      )
      return head :not_found unless record
      render json: { definition: record.definition }
    end
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring/protocols_definition_spec.rb`
Expected: PASS (3 examples, 0 failures).

- [ ] **Step 6: Run the whole authoring suite (regression)**

Run: `docker exec api-dev bundle exec rspec spec/requests/authoring`
Expected: PASS — gate (4) + preview (3) + draft (4) + definition (3) green.

- [ ] **Step 7: Commit**

```bash
git -C apps/api add config/routes.rb app/controllers/authoring/protocols_controller.rb spec/requests/authoring/protocols_definition_spec.rb
git -C apps/api commit -m "Add authoring read endpoint for loading a definition into the editor"
```

---

### Task 2: dashboard api functions `listAuthorProtocols` + `loadProtocolDefinition`

**Files:**
- Modify: `apps/dashboard/src/lib/api.ts`
- Test: `apps/dashboard/src/lib/api.loadpicker.test.ts`

**Interfaces:**
- Consumes: `GET /admin/api/protocols` (existing, via `adminFetch`); `GET /authoring/protocols/definition` (Task 1); the existing `jsonFetch`/`ApiError`/`adminFetch`/`AUTHORING_BASE`.
- Produces:
  - `listAuthorProtocols() -> Promise<Array<{ name: string; version: string; status: string }>>`
  - `loadProtocolDefinition(name: string, version: string) -> Promise<unknown | null>` (null on 404)

- [ ] **Step 1: Write the failing test**

Create `apps/dashboard/src/lib/api.loadpicker.test.ts`:

```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { listAuthorProtocols, loadProtocolDefinition } from "./api";

afterEach(() => vi.unstubAllGlobals());

function mockFetch(status: number, body: unknown) {
  vi.stubGlobal("fetch", vi.fn(async () => new Response(
    body === undefined ? "" : JSON.stringify(body),
    { status, headers: { "Content-Type": "application/json" } }
  )));
}

describe("listAuthorProtocols", () => {
  it("flattens the admin envelope to {name, version, status} rows", async () => {
    mockFetch(200, { data: { list: [
      { name: "respiratoria", version: "2", status: "draft" },
      { name: "dengue", version: "1", status: "published" }
    ] }, as_of: "2026-06-27T00:00:00Z" });
    const rows = await listAuthorProtocols();
    expect(rows).toEqual([
      { name: "respiratoria", version: "2", status: "draft" },
      { name: "dengue", version: "1", status: "published" }
    ]);
  });
});

describe("loadProtocolDefinition", () => {
  it("returns the definition body on 200", async () => {
    mockFetch(200, { definition: { name: "respiratoria", version: 1 } });
    expect(await loadProtocolDefinition("respiratoria", "1")).toEqual({ name: "respiratoria", version: 1 });
  });
  it("returns null on 404", async () => {
    mockFetch(404, undefined);
    expect(await loadProtocolDefinition("nope", "9")).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd apps/dashboard && npm run test -- src/lib/api.loadpicker.test.ts`
Expected: FAIL — `listAuthorProtocols`/`loadProtocolDefinition` are not exported.

- [ ] **Step 3: Implement the functions**

Append to `apps/dashboard/src/lib/api.ts`:

```ts
export interface AuthorProtocolRow { name: string; version: string; status: string; }

export async function listAuthorProtocols(): Promise<AuthorProtocolRow[]> {
  const env = await adminFetch<{ list: Array<{ name: string; version: string; status: string }> }>("/protocols");
  return env.data.list.map(r => ({ name: r.name, version: r.version, status: r.status }));
}

export async function loadProtocolDefinition(name: string, version: string): Promise<unknown | null> {
  const url = `${AUTHORING_BASE}/definition?name=${encodeURIComponent(name)}&version=${encodeURIComponent(version)}`;
  try {
    const body = await jsonFetch<{ definition: unknown }>(url, { method: "GET" });
    return body.definition;
  } catch (err) {
    if (err instanceof ApiError && err.status === 404) return null;
    throw err;
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd apps/dashboard && npm run test -- src/lib/api.loadpicker.test.ts`
Expected: PASS (3 examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/dashboard add src/lib/api.ts src/lib/api.loadpicker.test.ts
git -C apps/dashboard commit -m "Add api functions to list and load protocol definitions for the editor"
```

---

### Task 3: editor picker wiring

**Files:**
- Modify: `apps/dashboard/src/modules/ProtocolEditor.tsx`

**Interfaces:**
- Consumes: `listAuthorProtocols`, `loadProtocolDefinition`, `type AuthorProtocolRow` (Task 2); `TEMPLATE` (existing).
- Produces: a `<select>` at the top of the editor that loads the template or an existing definition into the textarea.

- [ ] **Step 1: Extend the import**

In `apps/dashboard/src/modules/ProtocolEditor.tsx`, change the api import (lines 2-3) to also bring the picker functions:

```tsx
import { gateProtocol, previewProtocol, saveProtocolDraft,
  listAuthorProtocols, loadProtocolDefinition,
  type GateResult, type PreviewResult, type DraftResult, type AuthorProtocolRow } from "../lib/api";
```

- [ ] **Step 2: Add picker state + load-on-mount + handler**

Immediately after the existing state declarations (after the `timer` ref line), add:

```tsx
  const [ opts, setOpts ] = useState<AuthorProtocolRow[]>([]);
  const [ loadErr, setLoadErr ] = useState<string | null>(null);

  useEffect(() => {
    listAuthorProtocols().then(setOpts).catch(() => setOpts([]));
  }, []);

  function onPick(value: string) {
    setLoadErr(null);
    if (value === "__new__") { setText(TEMPLATE); return; }
    const [ name, version ] = value.split("@@");
    loadProtocolDefinition(name, version).then(def => {
      if (def) setText(JSON.stringify(def, null, 2));
      else setLoadErr("definição não encontrada");
    }).catch(() => setLoadErr("não foi possível carregar"));
  }
```

- [ ] **Step 3: Render the picker**

Inside the left `<section>` (the JSON editor column), immediately after its `<h2>Definição (JSON)</h2>` line, add the picker:

```tsx
        <select
          onChange={e => onPick(e.target.value)}
          defaultValue="__new__"
          style={{ display: "block", marginBottom: 8, fontSize: 13 }}
        >
          <option value="__new__">Nova (template)</option>
          {opts.map(o => (
            <option key={`${o.name}@@${o.version}`} value={`${o.name}@@${o.version}`}>
              {o.name}@{o.version} ({o.status})
            </option>
          ))}
        </select>
        {loadErr && <p style={{ color: "var(--danger, #c00)", fontSize: 13 }}>{loadErr}</p>}
```

- [ ] **Step 4: Typecheck + build**

Run: `cd apps/dashboard && npm run build`
Expected: `tsc -b` passes and Vite build succeeds.

- [ ] **Step 5: Run the full dashboard test suite (regression)**

Run: `cd apps/dashboard && npm run test`
Expected: PASS — all lib tests green (including the new `api.loadpicker` spec).

- [ ] **Step 6: Commit**

```bash
git -C apps/dashboard add src/modules/ProtocolEditor.tsx
git -C apps/dashboard commit -m "Add load-existing-definition picker to the protocol editor"
```

---

## Wrap-up (after all tasks)

- api: `docker exec api-dev bundle exec rspec spec/requests/authoring` green; dashboard: `npm run test` + `npm run build` green.
- Live check: as a `protocol_author`, the editor picker lists existing protocols and loading one populates the textarea + re-runs the gate.
- Move the board card (`PVTI_lADOEbfGRc4BbxiBzgxNRDE`) to Done, then Verified.
- Sync is a separate explicit step.

## Self-Review notes

- **Spec coverage:** authoring read endpoint → Task 1; api functions (list + load) → Task 2; editor picker (dropdown + load → textarea + gate re-run) → Task 3. All spec sections mapped.
- **Type consistency:** `AuthorProtocolRow {name, version, status}` defined in Task 2 and consumed in Task 3; `loadProtocolDefinition(name, version) -> unknown|null` defined in Task 2 and called in Task 3's `onPick`; the `definition` JSON shape `{ definition: <hash> }` is produced by Task 1 and parsed by Task 2's `loadProtocolDefinition`.
- **Placeholder scan:** every step has complete code + exact commands; no TBD/TODO.
- **Picker → gate:** `onPick` calls `setText`, which triggers the existing `useEffect([text])` live-gate — no separate gate call needed.
