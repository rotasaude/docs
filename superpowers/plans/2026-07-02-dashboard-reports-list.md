# Reports List in City Dashboard (F-04.6) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a metadata-only reports list to the city dashboard — a read-only `admin/api` endpoint over `report_snapshots` and a `Reports` panel — mirroring the existing Triages slice.

**Architecture:** Backend `Admin::Scoped.report_snapshots` + `Admin::ReportsQuery` (metadata rows) + `Admin::Api::ReportsController#show` (envelope), tested with a request spec. Frontend `useReports` hook + `Reports.tsx` panel + nav wiring, verified by TypeScript build + the running dashboard.

**Tech Stack:** Rails 8.1 (RSpec) in `apps/api`; React/Vite/TypeScript + @tanstack/react-query in `apps/dashboard`.

## Global Constraints

- **Commits in English.** `apps/api` on branch `fix/migrations-owner-ddl-as-admin` (`git -C apps/api ...`). `apps/dashboard` is a **separate git repo** on branch `main` (`git -C apps/dashboard ...`) — commit frontend changes there.
- **api specs run IN THE CONTAINER:** `docker exec api-dev bundle exec rspec <path>`.
- **LGPD:** the reports list is **metadata-only** — `id`, `createdAt`, `tier`, `protocol` (`name · version`), `expiresAt`, `live`. NEVER expose `token`/`url`/`payload`/`signature`.
- admin/api is **read-only**; scoped by `current_municipality` + `period`; envelope `render_envelope(data)` → `{ data:, as_of: }`. No migration.
- Frontend modules are not unit-tested (repo convention); the hard automated gate is the TypeScript build passing.

---

### Task 1: Backend — reports endpoint

**Files:**
- Modify: `apps/api/app/queries/admin/scoped.rb`
- Create: `apps/api/app/queries/admin/reports_query.rb`, `apps/api/app/controllers/admin/api/reports_controller.rb`
- Modify: `apps/api/config/routes.rb`
- Test: `apps/api/spec/requests/admin/api/reports_spec.rb` (create)

**Interfaces:**
- Produces: `GET /admin/api/reports` → `{ data: { reports: [{id, createdAt, tier, protocol, expiresAt, live}], total }, as_of }`. No token/payload.

- [ ] **Step 1: Write the failing test**

Create `apps/api/spec/requests/admin/api/reports_spec.rb` (mirror `spec/requests/admin/api/cities_spec.rb`):

```ruby
require "rails_helper"
require Rails.root.join("spec/support/admin_rls")
require Rails.root.join("spec/support/admin_auth")

RSpec.describe "Admin::Api::Reports", type: :request do
  self.use_transactional_tests = false
  before { clean_admin_tables }
  after  { clean_admin_tables }

  def seed_report(muni, tier: "alta", token: "tok-#{SecureRandom.hex(4)}")
    pd = ProtocolDefinition.create!(municipality_id: muni.id, name: "resp", version: 3, status: "active",
                                    definition: { "name" => "resp", "version" => 3, "start_step_id" => "s1",
                                                  "steps" => [{ "id" => "s1", "prompt" => "?", "answer_type" => "boolean" }] })
    convo = Conversation.create!(municipality_id: muni.id, phone: "+5511#{rand(1000..9999)}", state: "completed")
    triage = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "resp",
                            municipality_id: muni.id, status: "completed", tier: tier)
    ReportSnapshot.create!(triage: triage, protocol_definition: pd, municipality_id: muni.id,
                           token: token, signature: "sig-#{token}",
                           payload: { "secret_clinical" => "NEVER-EXPOSE" }, outcome: { "tier" => tier, "priority" => 2 },
                           expires_at: 10.days.from_now)
  end

  it "lists the city's reports as metadata, without token or payload" do
    muni = nil
    as_admin do
      muni = Municipality.create!(name: "RepCity", slug: "rep-city", uf: "SP", status: "active")
      seed_report(muni, tier: "alta", token: "TOK-SECRET-123")
    end
    sign_in_as(operator!)

    get "/admin/api/reports", params: { period: "30d", municipality_id: muni.id }

    expect(response).to have_http_status(:ok)
    body = JSON.parse(response.body)
    reports = body.dig("data", "reports")
    expect(reports.size).to eq(1)
    row = reports.first
    expect(row["tier"]).to eq("alta")
    expect(row["protocol"]).to eq("resp · 3")
    expect(row["createdAt"]).to be_present
    expect(row).to have_key("live")
    expect(body["as_of"]).to be_present
    # LGPD: no token / payload / signature anywhere in the response
    expect(response.body).not_to include("TOK-SECRET-123")
    expect(response.body).not_to include("NEVER-EXPOSE")
  end

  it "denies a non-authorized user" do
    user = User.create!(email_address: "muni-#{SecureRandom.hex(3)}@x.com", password: "secret123")
    sign_in_as(user)
    get "/admin/api/reports", params: { period: "30d" }
    expect(response).to have_http_status(:forbidden)
  end
end
```

Adjust `seed_report`'s `ProtocolDefinition`/`Triage`/`ReportSnapshot`/`Conversation` attributes to the real models if they differ (check the models / a factory) — the asserted behavior (metadata row present; token/payload absent; 403 for non-authorized) is the requirement. If `sign_in_as`/`operator!` differ from the cities spec, mirror that spec exactly.

- [ ] **Step 2: Run to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/requests/admin/api/reports_spec.rb`
Expected: FAIL — no route / `uninitialized constant Admin::Api::ReportsController`.

- [ ] **Step 3: Add the scope**

In `apps/api/app/queries/admin/scoped.rb`, add (next to `triages`/`conversations`):

```ruby
  def self.report_snapshots(municipality)
    return ReportSnapshot.all if municipality == :all || municipality.nil?
    ReportSnapshot.where(municipality_id: municipality.id)
  end
```

- [ ] **Step 4: Create the query**

Create `apps/api/app/queries/admin/reports_query.rb` (mirror `admin/triages_query.rb`):

```ruby
# GET /admin/api/reports — lista de relatórios (F-04.6). Metadados APENAS.
# NUNCA expõe token/url/payload/signature (LGPD, como o painel de triages).
class Admin::ReportsQuery
  def self.call(municipality:, period:)
    new(municipality, period).call
  end

  def initialize(municipality, period)
    @muni = municipality
    @period = period
  end

  def call
    rows = Admin::Scoped.report_snapshots(@muni)
             .where(created_at: @period.from..@period.to)
             .includes(:protocol_definition)
             .order(created_at: :desc)
    { reports: rows.map { |r| serialize(r) }, total: rows.size }
  end

  private

  def serialize(report)
    {
      id: report.id,
      createdAt: report.created_at.iso8601,
      tier: report.outcome["tier"],
      protocol: "#{report.protocol_definition.name} · #{report.protocol_definition.version}",
      expiresAt: report.expires_at&.iso8601,
      live: report.expires_at.nil? || report.expires_at > Time.current
    }
  end
end
```

- [ ] **Step 5: Create the controller + route**

`apps/api/app/controllers/admin/api/reports_controller.rb`:

```ruby
class Admin::Api::ReportsController < Admin::Api::BaseController
  def show
    render_envelope(Admin::ReportsQuery.call(municipality: current_municipality, period: period))
  end
end
```

In `apps/api/config/routes.rb`, inside `namespace :admin { namespace :api { … } }`, add (next to `get "triages", …`):

```ruby
      get "reports", to: "reports#show"
```

- [ ] **Step 6: Run to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/requests/admin/api/reports_spec.rb`
Expected: PASS (2 examples).

- [ ] **Step 7: Commit (apps/api)**

```bash
git -C apps/api add app/queries/admin/scoped.rb app/queries/admin/reports_query.rb app/controllers/admin/api/reports_controller.rb config/routes.rb spec/requests/admin/api/reports_spec.rb
git -C apps/api commit -m "Add admin/api reports list endpoint (metadata only)"
git -C apps/api log --oneline -1
```

---

### Task 2: Frontend — Reports panel

**Files (all in `apps/dashboard`, separate git repo on `main`):**
- Modify: `src/lib/types.ts`, `src/shell/modules.ts`, `src/App.tsx`
- Create: `src/hooks/useReports.ts`, `src/modules/Reports.tsx`

**Interfaces:**
- Consumes: `GET /admin/api/reports` (Task 1) → `{ data: ReportsData, as_of }`.

- [ ] **Step 1: Add the types**

In `apps/dashboard/src/lib/types.ts`, add:

```ts
export interface ReportRow {
  id: string;
  createdAt: string;
  tier: string | null;
  protocol: string;
  expiresAt: string | null;
  live: boolean;
}
export interface ReportsData {
  reports: ReportRow[];
  total: number;
}
```

- [ ] **Step 2: Add the hook**

Create `apps/dashboard/src/hooks/useReports.ts` (mirror `useTriages.ts`):

```ts
import { useQuery } from "@tanstack/react-query";
import { adminFetch } from "../lib/api";
import { scopeParams, useScope } from "../lib/scope";
import type { ReportsData } from "../lib/types";

export function useReports() {
  const scope = useScope();
  return useQuery({
    queryKey: [ "reports", scope.period, scope.municipalityId ],
    queryFn: () => adminFetch<ReportsData>("/reports", scopeParams(scope)),
    staleTime: 30_000
  });
}
```

- [ ] **Step 3: Add the panel**

Create `apps/dashboard/src/modules/Reports.tsx`, mirroring `Triages.tsx`'s imports and loading/error/empty states, rendering a `DataTable`. Read the real `DataTable`/`Tag`/`Panel`/`PageHeader`/`Skeleton`/`ErrorState`/`EmptyState` component APIs from `Triages.tsx` and align exactly (column shape `{ label, w, align?, render }`, `rows`, `rowKey`, `empty`). Use the app's date formatter from `../lib/format` if present (else `new Date(r.createdAt).toLocaleString("pt-BR")`):

```tsx
import { useReports } from "../hooks/useReports";
import { Panel } from "../components/Panel";
import { PageHeader } from "../components/PageHeader";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { Skeleton } from "../components/Skeleton";
import { ErrorState } from "../components/ErrorState";
import { EmptyState } from "../components/EmptyState";

export function Reports() {
  const { data, isLoading, isError, error, refetch } = useReports();

  if (isLoading) return <Wrap><Panel title="Relatórios"><Skeleton rows={5} /></Panel></Wrap>;
  if (isError) return <Wrap><ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} /></Wrap>;
  if (!data) return <Wrap><EmptyState title="sem dados" /></Wrap>;

  const d = data.data;
  return (
    <Wrap>
      <Panel title="Relatórios" sub="report_snapshots (metadados)" asOf={data.as_of}>
        <DataTable
          cols={[
            { label: "Data", w: "2fr", render: (r) => <span className="mono">{fmtDate(r.createdAt)}</span> },
            { label: "Tier", w: "1fr", render: (r) => <Tag tone={tierTone(r.tier)}>{r.tier ?? "—"}</Tag> },
            { label: "Protocolo", w: "2fr", render: (r) => <span className="mono">{r.protocol}</span> },
            { label: "Expiração", w: "1fr", render: (r) => <Tag tone={r.live ? "ok" : "neutral"}>{r.live ? "ativo" : "expirado"}</Tag> }
          ]}
          rows={d.reports}
          rowKey={(r) => r.id}
          empty="nenhum relatório no período"
        />
      </Panel>
    </Wrap>
  );
}

function tierTone(t: string | null): string {
  if (t === "alta") return "down";
  if (t === "media") return "warn";
  if (t === "baixa") return "ok";
  return "neutral";
}
function fmtDate(iso: string): string {
  return new Date(iso).toLocaleString("pt-BR");
}
function Wrap({ children }: { children: React.ReactNode }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Relatórios" sub="reports" />
      {children}
    </div>
  );
}
```
If `Tag`'s `tone` type is a union that rejects a bare `string`, match the exact `Tone` type used in `Triages.tsx`/`theme/tokens`.

- [ ] **Step 4: Register the module**

In `apps/dashboard/src/shell/modules.ts`: add `"reports"` to the `ModuleId` union type, and add `{ id: "reports", label: "Relatórios", icon: "▤" }` to the "Triagem" `NAV_GROUPS` entry (next to Triagens/Classificação).

In `apps/dashboard/src/App.tsx`: add `import { Reports } from "./modules/Reports";` with the other module imports, and `case "reports": return <Reports />;` in the `switch`.

- [ ] **Step 5: Verify the TypeScript build passes**

Run: `sh -c "cd apps/dashboard && npm run build"`
Expected: build succeeds (types resolve; no TS errors). If the project has a `typecheck`/`tsc --noEmit` script, run that too. Fix any type mismatches by aligning to the real component/format APIs — do not `any`-cast to force it.

- [ ] **Step 6: Preview-verify the panel renders**

Start the dashboard dev server and confirm the panel mounts:
- `preview_start "dashboard"` (port 5175).
- Navigate to the app; confirm **"Relatórios"** appears in the Triagem nav group and selecting it mounts the `Reports` panel (a loading/empty/error state is acceptable if unauthenticated or no data — the point is the module resolves and renders without a crash).
- Check `preview_console_logs` for errors; take a `preview_screenshot`.
(Full data rendering needs a logged-in session + the api-dev backend + seeded reports; if the dashboard gates on login, verifying the nav item + panel mount + clean console + green build is sufficient for this task.)

- [ ] **Step 7: Commit (apps/dashboard)**

```bash
git -C apps/dashboard add src/lib/types.ts src/hooks/useReports.ts src/modules/Reports.tsx src/shell/modules.ts src/App.tsx
git -C apps/dashboard commit -m "Add Reports panel (metadata-only city reports list)"
git -C apps/dashboard log --oneline -1
```

---

## Wrap-up (after all tasks)

- Backend: `docker exec api-dev bundle exec rspec spec/requests/admin/api/reports_spec.rb` green.
- Frontend: `npm run build` green + preview screenshot.
- Move the board card (F-04.6) to Done, then Verified.
- Sync is a separate explicit step (user-authorized) — note the two repos (apps/api branch + apps/dashboard main).

## Self-Review notes

- **Spec coverage:** scope → Task 1 Step 3; query (metadata rows, no secrets) → Step 4; controller+route → Step 5; request spec (metadata present, token/payload absent, 403) → Step 1; frontend types/hook/panel/nav/App → Task 2; preview+build verification → Task 2 Steps 5-6. All mapped.
- **Type consistency:** `ReportsData`/`ReportRow` (Task 2 Step 1) match the backend serialize keys (Task 1 Step 4: `id/createdAt/tier/protocol/expiresAt/live`); `adminFetch<ReportsData>("/reports")` matches the route `get "reports"`.
- **LGPD:** the query omits token/url/payload/signature; the request spec asserts they're absent from the response body.
- **Two repos:** api commits under `apps/api`, frontend under `apps/dashboard` (main) — called out in Global Constraints + both tasks' commit steps.
