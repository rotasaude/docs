# FEATURE (dashboard) — Password reset flow (request + reset page)

**Date:** 2026-07-02
**Status:** Approved (brainstorming → ready for writing-plans)
**Module:** mod-06 (frontend). Completes F-06.2 (backend endpoints shipped).
**Board:** FEATURE card (In Progress).
**Touches:** `apps/dashboard` only (separate git repo, branch `main`).

## Problem

F-06.2 shipped the backend password-reset endpoints (`POST /passwords` request,
`PUT /passwords/:token` reset) + the mailer (link → `${PUBLIC_DASHBOARD_URL}?reset=<token>`),
but the dashboard has no UI: a user who clicks the reset link lands anonymous and
gets the `<Login />` screen, and there is no way to request a reset at all. This
feature adds the two missing pieces: a "forgot password" request on the Login
screen and a reset page that consumes `?reset=<token>`.

## Current state (verified)

- `main.tsx` `AppRoot`: routes by auth state — `loading → <Splash/>`,
  `anonymous → <Login/>`, `authenticated → <App/>` (the shell). This is the
  pre-auth entry point.
- `lib/api.ts`: `jsonFetch<T>(input, init)` — sets `credentials: "include"`,
  JSON headers; on `!ok` throws `ApiError(status, body, msg)`; `204 → undefined`.
  Existing bases: `BASE=/admin/api`, `SESSION_BASE=/session`. `login`/`logout`/
  `fetchCurrentSession` use `jsonFetch`.
- `modules/Login.tsx`: a small email+password form; catches `ApiError`; inline styles.
- `vite.config.ts`: dev proxies `/up`, `/admin/api`, `/session` → `VITE_API_PROXY_TARGET`
  (`http://localhost:3030`). **`/passwords` is not proxied** (new endpoint).
- `lib/api.test.ts`: vitest, `vi.stubGlobal("fetch", …)` via a `mockFetch(status, body)`
  helper. Scripts: `test` (`vitest run`), `typecheck` (`tsc --noEmit`), `build`
  (`tsc -b && vite build`).
- Backend `PUT /passwords/:token` returns `204` (success) or `422` with body
  `{ error: "invalid_token" }` (bad/expired/used token) or `{ errors: [...] }`
  (validation); `POST /passwords` always `204` (no enumeration).

## Decisions (from brainstorming)

1. Include **both** halves: the reset page (`?reset=<token>` → `PUT`) and a
   "forgot password" request on Login (email → `POST`).
2. Pre-auth routing: `AppRoot` short-circuits to `<ResetPassword/>` when
   `?reset=<token>` is present, regardless of auth state.
3. The request form always shows a generic "if the email exists, we sent a link"
   (mirrors the backend's no-enumeration).
4. Reuse `jsonFetch` (handles 204 + ApiError). No migration.

## Design

### 1. `lib/api.ts` — two functions

```ts
const PASSWORDS_BASE = import.meta.env.VITE_PASSWORDS_BASE || "/passwords";

export async function requestPasswordReset(email_address: string): Promise<void> {
  await jsonFetch<void>(PASSWORDS_BASE, { method: "POST", body: JSON.stringify({ email_address }) });
}

export async function resetPassword(token: string, password: string, password_confirmation: string): Promise<void> {
  await jsonFetch<void>(`${PASSWORDS_BASE}/${encodeURIComponent(token)}`, {
    method: "PUT", body: JSON.stringify({ password, password_confirmation })
  });
}
```

### 2. `modules/ResetPassword.tsx` (new)

Props `{ token: string }`. A form (new password + confirmation), inline styles
mirroring `Login.tsx`:
- client-side: if password ≠ confirmation → inline error, don't submit.
- submit → `resetPassword(token, password, confirmation)`:
  - success (resolves) → success state: "Senha redefinida." + a button "Ir para o
    login" that clears the `?reset` query and reloads (`window.location.assign`
    to the app base without the param, or `history.replaceState` + reload).
  - `ApiError` 422 with `body.error === "invalid_token"` → "Link inválido ou
    expirado. Solicite um novo."
  - `ApiError` 422 with `body.errors` → render those messages.
  - other → generic error.

### 3. `modules/Login.tsx` — "forgot password" mode

Add a `mode: "login" | "forgot"` state. A "Esqueci minha senha" link switches to
`forgot`; in `forgot` mode, an email field + submit → `requestPasswordReset(email)`
→ always show "Se o e-mail existir, enviamos um link para redefinir a senha." +
a "Voltar ao login" link. The existing login form is unchanged in `login` mode.

### 4. `main.tsx` — pre-auth reset route

In `AppRoot`, before the auth-state switch:
```ts
const resetToken = new URLSearchParams(window.location.search).get("reset");
if (resetToken) return <ResetPassword token={resetToken} />;
```

### 5. `vite.config.ts` — proxy `/passwords`

Add `"/passwords": proxy(TARGET)` to the dev `server.proxy` map (production's
reverse proxy already routes it). Without this, the dev `POST/PUT /passwords`
would 404.

## Testing / verification

- **`lib/api.test.ts`** (extend): `requestPasswordReset` POSTs to `/passwords`
  and resolves on 204; `resetPassword` PUTs to `/passwords/:token` and resolves
  on 204; `resetPassword` rejects with `ApiError` on 422. (vitest `mockFetch`
  pattern.)
- **`npm run typecheck` + `npm run build`** green (types across the new module +
  main.tsx + api.ts).
- **`npm run test`** green.
- **Preview (best-effort):** the reset page is pre-auth, so if the dashboard dev
  server starts (port 5173/5175 — was blocked by Docker last time), navigate to
  `?reset=<any>` and confirm the `ResetPassword` form renders; screenshot. If the
  port is unavailable, the vitest + typecheck + build + code review are the gate.

## Out of scope / follow-ups

- Password-strength meter; i18n beyond inline pt-BR; a dedicated "email sent"
  route (an inline message suffices).
- Rate-limit feedback UI (the backend 429 is handled generically by `ApiError`).

## Workflow

FEATURE card In Progress. writing-plans → subagent-driven-development. Commits in
English; `apps/dashboard` on branch `main` (separate repo).
