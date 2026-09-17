# Dashboard Password Reset Flow (FEATURE) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the dashboard UI for password reset — a "forgot password" request on the Login screen and a pre-auth reset page that consumes `?reset=<token>` — completing the F-06.2 flow.

**Architecture:** Two `api.ts` functions over the F-06.2 endpoints (+ a dev proxy for `/passwords`); a `ResetPassword` page routed pre-auth from `main.tsx` when `?reset=` is present; a "forgot" mode on `Login`.

**Tech Stack:** React/Vite/TypeScript, @tanstack/react-query, vitest. All in `apps/dashboard`.

## Global Constraints

- **Commits in English** (UI copy stays Portuguese). `apps/dashboard` is a **separate git repo** on branch `main` — commit there (`git -C apps/dashboard ...`). Do NOT touch `apps/api`.
- Reuse `jsonFetch` (handles `204 → undefined`, throws `ApiError(status, body)` on non-ok). No new fetch machinery.
- The request form must NOT reveal whether an email exists — always show the same "sent" message.
- Backend contract: `PUT /passwords/:token` → `204` | `422 { error: "invalid_token" }` | `422 { errors: [...] }`; `POST /passwords` → `204`.
- Hard gates: `npm run typecheck` and `npm run build` and `npm run test` (vitest) all green. No `any`/`@ts-ignore` to force it.

---

### Task 1: `api.ts` functions + dev proxy + tests

**Files (in `apps/dashboard`):**
- Modify: `src/lib/api.ts`, `vite.config.ts`
- Test: `src/lib/api.test.ts` (extend)

**Interfaces:**
- Produces: `requestPasswordReset(email_address: string): Promise<void>`;
  `resetPassword(token: string, password: string, password_confirmation: string): Promise<void>`.

- [ ] **Step 1: Write the failing tests**

In `apps/dashboard/src/lib/api.test.ts`, add (importing the two new functions at the top: `import { requestPasswordReset, resetPassword } from "./api";`):

```ts
describe("password reset", () => {
  it("requestPasswordReset POSTs to /passwords and resolves on 204", async () => {
    const fetchMock = vi.fn(async () => new Response(null, { status: 204 }));
    vi.stubGlobal("fetch", fetchMock);
    await expect(requestPasswordReset("a@x.com")).resolves.toBeUndefined();
    const [url, init] = fetchMock.mock.calls[0];
    expect(String(url)).toContain("/passwords");
    expect(init.method).toBe("POST");
    expect(JSON.parse(init.body)).toEqual({ email_address: "a@x.com" });
  });

  it("resetPassword PUTs to /passwords/:token and resolves on 204", async () => {
    const fetchMock = vi.fn(async () => new Response(null, { status: 204 }));
    vi.stubGlobal("fetch", fetchMock);
    await expect(resetPassword("tok-1", "newpw", "newpw")).resolves.toBeUndefined();
    const [url, init] = fetchMock.mock.calls[0];
    expect(String(url)).toContain("/passwords/tok-1");
    expect(init.method).toBe("PUT");
    expect(JSON.parse(init.body)).toEqual({ password: "newpw", password_confirmation: "newpw" });
  });

  it("resetPassword rejects with ApiError on 422", async () => {
    mockFetch(422, { error: "invalid_token" });
    await expect(resetPassword("bad", "a", "a")).rejects.toBeInstanceOf(ApiError);
  });
});
```
(`mockFetch`/`ApiError` are already imported/defined in this file; add `requestPasswordReset, resetPassword` to the import from `./api`.)

- [ ] **Step 2: Run to verify they fail**

Run: `sh -c "cd apps/dashboard && npm run test -- src/lib/api.test.ts"`
Expected: FAIL — `requestPasswordReset`/`resetPassword` are not exported.

- [ ] **Step 3: Add the api functions**

In `apps/dashboard/src/lib/api.ts`, near the session helpers, add:

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

- [ ] **Step 4: Add the dev proxy**

In `apps/dashboard/vite.config.ts`, add `"/passwords": proxy(TARGET)` to the `server.proxy` map (alongside `/session`).

- [ ] **Step 5: Run to verify tests pass**

Run: `sh -c "cd apps/dashboard && npm run test -- src/lib/api.test.ts"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C apps/dashboard add src/lib/api.ts src/lib/api.test.ts vite.config.ts
git -C apps/dashboard commit -m "Add password-reset API client functions and dev proxy"
git -C apps/dashboard log --oneline -1
```

---

### Task 2: ResetPassword page + Login "forgot" mode + pre-auth routing

**Files (in `apps/dashboard`):**
- Create: `src/modules/ResetPassword.tsx`
- Modify: `src/modules/Login.tsx`, `src/main.tsx`

**Interfaces:**
- Consumes: `resetPassword`, `requestPasswordReset` (Task 1).

- [ ] **Step 1: Create `ResetPassword.tsx`**

Create `apps/dashboard/src/modules/ResetPassword.tsx` (inline styles mirroring `Login.tsx`):

```tsx
import { useState, type FormEvent } from "react";
import { resetPassword, ApiError } from "../lib/api";

export function ResetPassword({ token }: { token: string }) {
  const [ password, setPassword ] = useState("");
  const [ confirm, setConfirm ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ done, setDone ] = useState(false);

  async function onSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    if (password !== confirm) { setError("As senhas não coincidem."); return; }
    setBusy(true);
    try {
      await resetPassword(token, password, confirm);
      setDone(true);
    } catch (err) {
      if (err instanceof ApiError && err.status === 422) {
        const body = err.body as { error?: string; errors?: string[] };
        if (body?.error === "invalid_token") setError("Link inválido ou expirado. Solicite um novo.");
        else if (body?.errors?.length) setError(body.errors.join(" "));
        else setError("Não foi possível redefinir a senha.");
      } else {
        setError("Não foi possível redefinir a senha. Tente novamente.");
      }
    } finally {
      setBusy(false);
    }
  }

  function goToLogin() {
    window.location.assign(window.location.pathname); // drops ?reset=, reloads to Login
  }

  const inputStyle = { width: "100%", padding: 8, marginTop: 4, borderRadius: 6, border: "1px solid var(--line, #ccc)" } as const;

  return (
    <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <div style={{ width: 320, display: "flex", flexDirection: "column", gap: 12, padding: 24,
        border: "1px solid var(--line, #e6e6e6)", borderRadius: 10 }}>
        <strong style={{ fontFamily: "var(--font-mono, monospace)", fontSize: 14 }}>Redefinir senha</strong>
        {done ? (
          <>
            <p style={{ fontSize: 13, margin: 0 }}>Senha redefinida com sucesso.</p>
            <button onClick={goToLogin} style={btnStyle}>Ir para o login</button>
          </>
        ) : (
          <form onSubmit={onSubmit} style={{ display: "flex", flexDirection: "column", gap: 12 }}>
            <label style={{ fontSize: 12, color: "var(--ink2, #444)" }}>
              Nova senha
              <input type="password" value={password} onChange={e => setPassword(e.target.value)} required autoFocus style={inputStyle} />
            </label>
            <label style={{ fontSize: 12, color: "var(--ink2, #444)" }}>
              Confirmar senha
              <input type="password" value={confirm} onChange={e => setConfirm(e.target.value)} required style={inputStyle} />
            </label>
            {error && <p role="alert" style={{ color: "var(--danger, #c0341d)", fontSize: 12, margin: 0 }}>{error}</p>}
            <button type="submit" disabled={busy} style={btnStyle}>{busy ? "Redefinindo…" : "Redefinir senha"}</button>
          </form>
        )}
      </div>
    </div>
  );
}

const btnStyle = { padding: "8px 12px", borderRadius: 6, border: "none", cursor: "pointer",
  background: "var(--accent, #2b59ff)", color: "#fff", fontSize: 13 } as const;
```

- [ ] **Step 2: Add the "forgot" mode to `Login.tsx`**

In `apps/dashboard/src/modules/Login.tsx`: add `requestPasswordReset` to the `../lib/api` import; add a `mode` state (`"login" | "forgot"`) and a `sent` state. Keep the existing login form for `mode === "login"`, and add:
- below the login button, a text button: `<button type="button" onClick={() => setMode("forgot")} …>Esqueci minha senha</button>`.
- when `mode === "forgot"`: render a small form with just the email field; on submit call `await requestPasswordReset(email)` in a try/finally, then `setSent(true)`; render (regardless of success/error) the message "Se o e-mail existir, enviamos um link para redefinir a senha." and a "Voltar ao login" button (`onClick={() => { setMode("login"); setSent(false); }}`). The catch may swallow errors — the message is intentionally identical (no enumeration).

Keep the styles consistent with the existing form (reuse `inputStyle`). Do not change the login-mode behavior.

- [ ] **Step 3: Route the reset page pre-auth in `main.tsx`**

In `apps/dashboard/src/main.tsx`, import `ResetPassword` and, at the TOP of `AppRoot` (before `if (auth.state.kind === "loading")`), add:

```tsx
  const resetToken = new URLSearchParams(window.location.search).get("reset");
  if (resetToken) return <ResetPassword token={resetToken} />;
```
(`import { ResetPassword } from "./modules/ResetPassword";` with the other imports.)

- [ ] **Step 4: Typecheck + build + test**

Run: `sh -c "cd apps/dashboard && npm run typecheck && npm run build && npm run test"`
Expected: all green (types resolve; vitest passes). Fix any type errors by aligning to the real component/api types — no `any`/`@ts-ignore`.

- [ ] **Step 5: Commit**

```bash
git -C apps/dashboard add src/modules/ResetPassword.tsx src/modules/Login.tsx src/main.tsx
git -C apps/dashboard commit -m "Add password reset page and forgot-password request UI"
git -C apps/dashboard log --oneline -1
```

---

## Wrap-up (after all tasks)

- `sh -c "cd apps/dashboard && npm run typecheck && npm run build && npm run test"` — green.
- Preview (best-effort): the reset page is pre-auth; if the dev server starts, navigate to `?reset=<any>` and confirm the form renders. If the port is unavailable, build+typecheck+vitest+review are the gate.
- Move the FEATURE card to Done, then Verified.
- Sync is a separate explicit step (user-authorized) — `apps/dashboard` main.

## Self-Review notes

- **Spec coverage:** api functions + proxy → Task 1; ResetPassword page + Login forgot mode + pre-auth routing → Task 2. All spec sections mapped.
- **Type consistency:** `resetPassword(token, password, password_confirmation)` / `requestPasswordReset(email_address)` signatures used identically in api.ts (Task 1), ResetPassword.tsx and Login.tsx (Task 2), and the tests (Task 1).
- **No enumeration:** the Login forgot form shows an identical message on success/error.
- **Error handling:** ResetPassword distinguishes `invalid_token`, `errors[]`, mismatch, and generic — per the backend contract.
- **Two repos:** all changes under `apps/dashboard` (main); no `apps/api` change.
