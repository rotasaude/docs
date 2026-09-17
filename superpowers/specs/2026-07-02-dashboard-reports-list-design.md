# F-04.6 — Reports list in the city dashboard

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-04 (backend `apps/api` + frontend `apps/dashboard`)
**Board:** F-04.6 (In Progress)
**Touches:** `apps/api` (new admin/api endpoint) + `apps/dashboard` (new panel).

## Problem

Reports (`report_snapshots`, the frozen per-triage output) are produced and
delivered to citizens, but the city dashboard has no way to see the reports its
triages generated — every other domain (triages, conversations, consent,
protocols…) has an `admin/api` read-only panel, reports do not. F-04.6 adds a
**metadata-only** reports list panel, LGPD-aligned with the existing Triages
panel ("references and counts, never `answers`").

## Current state (verified)

- **No** `Admin::Api::ReportsController` / `Admin::ReportsQuery` / reports route.
- `ReportSnapshot`: `belongs_to :triage, :protocol_definition`; columns include
  `municipality_id` (not null), `outcome` (jsonb — has `tier`), `expires_at`,
  `created_at`, plus `token`/`signature`/`payload` (the citizen's private
  report — NOT to be exposed). `#url` builds the public WPDA link. `scope :live`.
- `Admin::Scoped` (per-model tenant scopes) has triages/conversations/consents/
  protocol_definitions/dashboard_metrics/inbound_messages — **no `report_snapshots`**.
- `Admin::Api::BaseController`: read-only, `skip_tenant_scope`, admin connection,
  `resolve_scope` (`current_municipality`, `period`), `render_envelope(data)` →
  `{ data:, as_of: }`. Route pattern: `namespace :admin { namespace :api { get
  "triages", to: "triages#show" } } }`.
- Request-spec harness: `spec/support/admin_rls` + `spec/support/admin_auth`
  (`sign_in_as`, `operator!`), `use_transactional_tests = false` + `as_admin` +
  `clean_admin_tables`.
- Dashboard: `App.tsx` `switch(active)` router + module imports; `shell/modules.ts`
  (`ModuleId` union, `NAV_GROUPS`, `labelFor`); `hooks/useTriages.ts` (`useQuery` +
  `adminFetch<TriagesData>("/triages", scopeParams(scope))`); `modules/Triages.tsx`
  (`Panel`/`PageHeader`/`DataTable`/`Tag`/`Skeleton`/`ErrorState`/`EmptyState`,
  reads `data.data` + `data.as_of`). Modules are **not** unit-tested (tests live in
  `src/lib/`); panels are verified via the running dev server (`launch.json` →
  `dashboard`, port 5175).

## Decisions (from brainstorming)

1. **Metadata-only** rows (LGPD): `id`, `createdAt`, `tier`, `protocol`
   (`name · version`), `expiresAt`, `live`. **Never** `token`/`url`/`payload`/
   `signature`.
2. Full-stack slice mirroring the Triages panel end-to-end: backend query +
   endpoint + route + request spec; frontend hook + panel + nav wiring, verified
   via preview.
3. Scoped by municipality + period; ordered newest-first; simple list (no
   pagination for MVP).
4. No migration; no write (admin/api is read-only).

## Design

### Backend

1. **`Admin::Scoped.report_snapshots(municipality)`** — add to the module:
   `return ReportSnapshot.all if municipality == :all || municipality.nil?;
   ReportSnapshot.where(municipality_id: municipality.id)`.
2. **`Admin::ReportsQuery`** (`app/api`/queries dir, mirroring `Admin::TriagesQuery`):
   ```ruby
   def call
     rows = Admin::Scoped.report_snapshots(@muni)
              .where(created_at: @period.from..@period.to)
              .includes(:protocol_definition)
              .order(created_at: :desc)
     {
       reports: rows.map { |r| serialize(r) },
       total: rows.size
     }
   end
   # serialize(r): { id:, createdAt: r.created_at.iso8601,
   #   tier: r.outcome["tier"], protocol: "#{r.protocol_definition.name} · #{r.protocol_definition.version}",
   #   expiresAt: r.expires_at&.iso8601, live: r.expires_at.nil? || r.expires_at > Time.current }
   ```
   No token/url/payload/signature in the output.
3. **`Admin::Api::ReportsController < Admin::Api::BaseController`**: `def show;
   render_envelope(Admin::ReportsQuery.call(municipality: current_municipality,
   period: period)); end`.
4. **Route**: add `get "reports", to: "reports#show"` in the `admin/api` namespace.
5. **Request spec** (`spec/requests/admin/api/reports_spec.rb`): with a
   municipality + protocol_definition + triage + report_snapshot, an authorized
   admin gets `200`, `body.dig("data","reports")` has the row with `tier`/
   `protocol`/`createdAt`, `body["as_of"]` present; **assert the JSON does NOT
   contain the snapshot's `token` or `payload`**. A non-authorized user → 403
   (mirror the cities spec). Cross-tenant scoping (operator sees the city's
   reports; a report from another municipality is excluded).

### Frontend (`apps/dashboard`)

6. **`src/lib/types.ts`**: `export interface ReportRow { id: string; createdAt:
   string; tier: string | null; protocol: string; expiresAt: string | null; live:
   boolean }` and `export interface ReportsData { reports: ReportRow[]; total:
   number }`.
7. **`src/hooks/useReports.ts`** (mirror `useTriages`): `useQuery({ queryKey:
   ["reports", scope.period, scope.municipalityId], queryFn: () =>
   adminFetch<ReportsData>("/reports", scopeParams(scope)), staleTime: 30_000 })`.
8. **`src/modules/Reports.tsx`** (mirror `Triages.tsx`): `PageHeader title="Relatórios"`;
   a `Panel` with a `DataTable` — columns **Data** (`createdAt`, formatted),
   **Tier** (`Tag`), **Protocolo** (`protocol`), **Expiração** (`expiresAt` +
   `live` as an "ativo/expirado" `Tag`); `loading`/`error`/`empty` states like
   Triages (`Skeleton`/`ErrorState`/`EmptyState`), reading `data.data.reports`
   and `data.as_of`.
9. **`src/shell/modules.ts`**: add `"reports"` to the `ModuleId` union and a
   `NavItem { id: "reports", label: "Relatórios", icon: "▤" }` to the "Triagem"
   group.
10. **`src/App.tsx`**: `import { Reports } from "./modules/Reports"` +
    `case "reports": return <Reports />;`.

## Testing / verification

- **Backend**: the request spec above (metadata present, token/payload absent,
  scoping, 403 for non-authorized). Run in the container.
- **Frontend**: no module unit test (matches the repo). Verify via the running
  dashboard dev server (`preview_start "dashboard"` / port 5175): navigate to
  **Relatórios**, confirm the panel renders the list (seed a report or rely on
  dev data), and screenshot. TypeScript build/compile must pass
  (`npm run build` or `tsc`).

## Out of scope / follow-ups

- Opening a citizen's report from the panel (the public link stays with the
  citizen — LGPD).
- Pagination / filtering by tier/protocol (simple period-scoped list for MVP).
- Exposing `payload`/`outcome` detail (metadata only).

## Workflow

Card F-04.6 In Progress. writing-plans → subagent-driven-development. Commits in
English; api on branch `fix/migrations-owner-ddl-as-admin`; dashboard on its own
git repo (`apps/dashboard`, branch `main`) — commit there separately.
