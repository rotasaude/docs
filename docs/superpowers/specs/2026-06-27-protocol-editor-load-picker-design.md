# Protocol editor — load existing definition picker (F-03.12 follow-up)

**Date:** 2026-06-27
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-03
**Board:** "Protocol editor: load existing definition picker (F-03.12 follow-up)" (`PVTI_lADOEbfGRc4BbxiBzgxNRDE`)
**Touches:** `apps/api` (one authoring read action), `apps/dashboard` (editor picker wiring)

## Problem

The F-03.12 editor seeds only the `TEMPLATE`; the spec's "load an existing
definition to edit" picker was deferred. This adds it.

## Key finding

The dashboard read endpoint `GET /admin/api/protocols` (`Admin::ProtocolsQuery`)
exposes **only metadata** (name, version, status, audit, schema/linter/gates
flags) — NOT the raw `definition` JSON. So the editor cannot load a definition
from it. A new authoring read endpoint is required.

## Decisions (from brainstorming)

- The "read a definition for editing" endpoint lives on the **authoring surface**
  (session + tenant/RLS + `ProtocolPolicy.author?`), not `/admin/api` — it is an
  authoring read, and the author role is the right gate.
- The picker's dropdown is populated from the existing `GET /admin/api/protocols`
  list (name + version metadata); selecting an item fetches that definition via
  the new authoring read endpoint.

## Design

### Backend — `Authoring::ProtocolsController#definition` (new read action)

- Route: `GET /authoring/protocols/definition` (author-gated via the existing
  `require_author!` before_action; tenant-scoped via the inherited `within_tenant`).
- Params: `name`, `version`. Looks up
  `ProtocolDefinition.find_by(name:, version:, municipality_id: Current.municipality_id)`.
- Found → `{ definition: <record.definition> }` (200). Not found → `head :not_found`
  (404). Non-author → 403 (from `require_author!`). Unauthenticated → 401.
- Pure read; no command needed.

### Frontend — picker in `ProtocolEditor`

- `lib/api.ts`:
  - `loadProtocolDefinition(name: string, version: string) -> Promise<unknown | null>`
    — `GET /authoring/protocols/definition?name=&version=`; returns the `definition`
    body, or `null` on 404.
  - `listAuthorProtocols() -> Promise<Array<{name, version, status}>>` — wraps the
    existing `adminFetch<ProtocolsListData>("/protocols")` and flattens to the
    `{name, version, status}` rows the dropdown needs. (Reuses the existing admin
    list — no new list endpoint.)
- `ProtocolEditor.tsx`:
  - A `<select>` at the top: a "Nova (template)" option plus one option per
    existing protocol version (label `name@version (status)`), loaded on mount via
    `listAuthorProtocols`.
  - On selecting "Nova" → set the textarea to `TEMPLATE`.
  - On selecting an existing one → `loadProtocolDefinition(name, version)`; if it
    returns a definition, `setText(JSON.stringify(def, null, 2))`; if `null`, show
    "definição não encontrada". The live gate then re-runs automatically (the
    existing `useEffect` on `text`).

### Errors & states

- Load 404 → inline "definição não encontrada"; textarea unchanged.
- Non-author → the list/load calls 403; the editor already shows a no-permission
  path (the picker simply stays empty / disabled).

## Testing

**api** (run in the container):
- Request spec for `GET /authoring/protocols/definition`: existing (name, version)
  in the tenant → 200 `{definition: {...}}` (assert a known field round-trips);
  unknown → 404; non-author session → 403. (Same session/tenant stub pattern as
  the other authoring specs.)

**dashboard** (run on host):
- `lib/api.ts`: `loadProtocolDefinition` returns the body on 200 and `null` on 404;
  `listAuthorProtocols` flattens the admin envelope to `{name, version, status}`
  rows (mock `fetch`).
- Editor picker verified via `npm run build` + the full `npm run test` suite
  staying green (no RTL component test, per the project pattern).

## Out of scope

- Editing a published version (still rejected by `SaveDraft` → `version_not_editable`).
- A dedicated authoring list endpoint (the picker reuses the admin metadata list).

## Workflow

Card is In Progress. writing-plans → subagent-driven-development. Commits in
English; api on `fix/migrations-owner-ddl-as-admin`, dashboard on `main`.
