# F-03.12 — Protocol editor + live preview (in the dashboard)

**Date:** 2026-06-27
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03
**Board:** F-03.12 (item `PVTI_lADOEbfGRc4BbxiBzgw_5lU`)
**Touches:** `apps/api` (new session-authed authoring controller), `apps/dashboard` (editor module)

## Problem

The server-side protocol gate (`Protocols::Gate`, F-03.9) and a preview engine
(`Protocols::Protocol#evaluate`) exist, but **no UI consumes them** — the
authoring endpoints have zero frontend consumer. Protocol authors in a city have
no way to write/edit a protocol definition, see validation errors as they type,
or simulate the flow before saving. F-03.12 builds that editor.

## Context discovered

- `apps/dashboard` is the municipal app with **session login** (`lib/auth.tsx`,
  `modules/Login.tsx`) and a read-only protocols view (`modules/Protocols.tsx`,
  `GET /admin/api/protocols`). Navigation is **state-based** (`active: ModuleId`
  + a `switch` in `App.tsx`; modules registered in `shell/modules.ts`) — no
  react-router. `lib/api.ts` sends the session cookie (`credentials: "include"`)
  and supports JSON POST.
- The `/admin/api` namespace is **read-only by acceptance criterion §10** and
  runs under `with_admin_connection` (**BYPASSRLS**) — safe *only* because it is
  read-only. It must NOT host writes.
- The existing authoring endpoints (`ProtocolsController` — `/protocols/:name/{show,
  preview,gate}`) use a **stub** `Author` bearer-token auth ("autenticação real
  vira ADR próprio"). The write precedent is `PublicationsController`
  (`POST /protocols/:version/publish`): session auth + tenant-scoped (RLS) +
  `ProtocolPolicy`.
- `ProtocolPolicy` already has `author?` / `publish?` (role-based, per
  municipality). No "create draft" command exists yet (only seed/publish/activate
  /retire) — saving a draft is net-new backend.

## Decisions (from brainstorming)

1. **Editor lives in the dashboard** (municipal session), not the wpda.
2. **Scope = edit + live gate validation + live preview + save draft.**
   Publish/activate stay separate (existing flow). Not full lifecycle in the
   editor.
3. **Authoring endpoints live on a dedicated session-authed surface** (the
   `PublicationsController` pattern), outside `/admin/api`. `/admin/api` stays
   read-only. The Author-token stub controller is left as-is (not migrated).

## Design

### Backend — `Authoring::ProtocolsController` (new)

New controller at `/authoring/protocols`, `include Authentication`,
**tenant-scoped** (inherits `within_tenant` → RLS `SET LOCAL`), each action
authorized by the author role in the current municipality
(`role?(:protocol_author, Current.municipality_id)`, via `ProtocolPolicy`). This
is a Rails API app — no CSRF token is required (same as `PublicationsController`).

- **`POST /authoring/protocols/gate`** — body `{ definition }` →
  `Protocols::Gate.call(definition)` → `{ valid: true }` (200) or
  `{ valid: false, errors: [...] }` (422). Pure (no DB).
- **`POST /authoring/protocols/preview`** — body `{ definition, answers }` →
  run `Protocols::Gate.call`; if invalid return `{ valid: false, errors: [...] }`
  (422); if valid, `Protocols::Definitions.build(definition).evaluate(answers)`
  → `{ outcome: <outcome.to_h> }` (200). Pure (no DB).
- **`POST /authoring/protocols/draft`** — body `{ definition }` (the JSON carries
  its own `name` + `version`, required by the schema) → upsert a **draft**
  `ProtocolDefinition` for `(name, version, Current.municipality_id)`:
  - not found → create with `status: "draft"`;
  - found and `status == "draft"` → update its `definition`;
  - found and published/active/retired → `422` (`{ error: "version_not_editable" }`);
  → `{ id, name, version, status }` (200). Write under RLS, author-authorized.

  Authorization: build the candidate `ProtocolDefinition` (unsaved, with
  `municipality_id = Current.municipality_id`) and check `ProtocolPolicy.new(
  Current.user, record).author?` before writing. Non-author → `403`.

  **A draft may be gate-invalid** — drafts are work-in-progress, so saving does
  NOT require the full gate to pass. The only floor is the model's `before_save`
  minimal `Validator` (engine-load safety: name/version/start_step_id/steps +
  refs + no cycles). If that minimal validation fails, `create!`/`update!` raises
  `ActiveRecord::RecordInvalid`; the action rescues it → `422`
  (`{ error: "invalid_definition", message: <errors> }`). The full gate is shown
  live in the editor but is enforced only at publish (`Protocols::Publish`,
  F-03.9), so a gate-invalid draft is harmless — it cannot be published.

### Frontend — dashboard editor module

- Register a new `ModuleId` `"protocol-editor"` in `shell/modules.ts` (label e.g.
  "Editor"), add it to `AppHeader` nav and the `renderModule` switch in `App.tsx`.
- `modules/ProtocolEditor.tsx`:
  - A monospace `<textarea>` holding the definition JSON (no heavy code editor —
    YAGNI).
  - **Live gate**: on edit, debounced ~400ms, parse the JSON locally; if it
    parses, `POST /authoring/protocols/gate` and render the error list (or
    "válido ✓"). JSON parse errors are shown inline without a round-trip.
  - **Live preview**: a panel with an `answers` input (a small JSON map of
    `step_id → answer`); on submit, `POST /authoring/protocols/preview` and render
    the outcome (status, tier, priority, awaiting, trail). Disabled while the gate
    is invalid, with "corrija os erros para pré-visualizar".
  - **Save draft**: button → `POST /authoring/protocols/draft`; shows saved
    `name@version (status)` or the mapped error (version_not_editable / forbidden
    / invalid).
  - **Load**: a picker of existing definitions (from the existing
    `GET /admin/api/protocols` index) to load one into the textarea, plus a "nova"
    option seeding a minimal valid template.
- `lib/api.ts`: add `gateProtocol(definition)`, `previewProtocol(definition,
  answers)`, `saveProtocolDraft(definition)` using the existing `jsonFetch`
  (cookie included) against the `/authoring/protocols/*` paths (a new
  `AUTHORING_BASE = import.meta.env.VITE_AUTHORING_BASE || "/authoring/protocols"`).
  These do NOT use the `/admin/api` envelope.
- Pure helpers extracted for testability (no React Testing Library, matching the
  project's lib-test pattern): e.g. `parseDefinition(text) -> {ok, value|error}`
  and the gate-state reducer logic.

### Errors & states

- JSON unparseable → inline "JSON inválido: <message>", no network call.
- Gate invalid → error list; preview disabled.
- Save: 200 → saved badge; 422 `version_not_editable` → "essa versão já foi
  publicada; suba a versão"; 403 → "sem permissão de autor"; other → generic.
- Not-an-author session → the editor still renders but gate/preview/save return
  403; the module shows a "sem permissão de autoria nesta cidade" notice.

## Testing

**api** (run in the container: `docker exec api-dev bundle exec rspec`):
- Request specs for `Authoring::ProtocolsController` using the session + tenant
  pattern (as in `publications_controller_spec` / admin request specs):
  - `gate`: valid definition → 200 `{valid:true}`; invalid → 422 with `errors`.
  - `preview`: valid def + answers → 200 with `outcome`; invalid def → 422 errors.
  - `draft`: new → creates `status:"draft"` (verify row); existing draft →
    updates definition; published version → 422 `version_not_editable`;
    non-author session → 403.

**dashboard** (run on host: `cd apps/dashboard && npm run test`):
- `lib/api.ts`: the three new functions hit the right paths/methods and parse
  responses (mock `fetch`).
- Editor pure helpers: `parseDefinition` (valid/invalid JSON), the gate-state
  logic. Component wiring verified via `npm run build` (tsc) + the browser
  preview.

## Out of scope / follow-ups

- Publish/activate from the editor (F-03.13 / F-03.18 already Done — separate
  actions).
- Syntax-highlighting code editor.
- Real author authentication (still the stub `Author` path on the legacy
  controller; a dedicated auth ADR remains future work). The new authoring
  controller uses the municipal session, which is the real auth for the dashboard.

## Dev note

The dev login `admin@curitiba.demo` is a `municipal_admin`; it may lack the
`protocol_author` role. The editor happy-path in dev may require granting
`protocol_author` to that user/municipality (a `Membership` with role
`protocol_author`).

## Workflow

Card F-03.12 is In Progress. After spec approval: writing-plans →
subagent-driven-development. Commits in English; api on branch
`fix/migrations-owner-ddl-as-admin`, dashboard on `main`.
