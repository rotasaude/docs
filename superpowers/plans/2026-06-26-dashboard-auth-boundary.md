# Dashboard — Fronteira de auth (módulo 06, fatia) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dar login ao dashboard (usuário municipal) com tenant vindo da sessão, destravando dados reais nos painéis.

**Architecture:** Adaptar a auth do `admin` (decisão C): `AuthProvider` (loading/anonymous/authenticated, sem MFA), gate no `main.tsx`, funções de sessão no `api.ts`. Aposentar `VITE_MUNICIPALITY_ID` — o tenant vem da membership do usuário (backend deriva). Vitest cobre a lógica de auth.

**Tech Stack:** React 18 + TypeScript + @tanstack/react-query + vitest + @testing-library/react.

## Global Constraints

- Trabalhar em `apps/dashboard`. **O repo git root É `apps/dashboard`** (branch `main`), NÃO o monorepo. Para git: `cd apps/dashboard` e use paths relativos (plano diz `git add src/X`).
- Commit identity: `git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit ...`
- Auth backend já existe: `POST/GET/DELETE /session` (base `VITE_SESSION_BASE` default `/session`), cookie HttpOnly, `credentials: "include"`.
- **Sem MFA** (usuário municipal não faz MFA no login). **Sem ScopePicker.** **Sem** gestão de usuários/convites.
- Tenant vem da sessão (`user.memberships[0].municipality_id`); `municipality_id` no request é inofensivo (backend ignora p/ municipal).
- Gate: `npm run typecheck` limpo ao fim de cada task; `npm run test` verde nas tasks com teste.
- Fatia 1 já presente: `src/lib/{api,scope,format,types,tenant}.ts`, components, hooks, shell, Overview, App, main — todos commitados. `@testing-library/react`, `vitest`, `jsdom` instalados.

---

### Task 1: `lib/api.ts` — funções de sessão (TDD)

**Files:**
- Modify: `apps/dashboard/src/lib/api.ts`
- Modify (test): `apps/dashboard/src/lib/api.test.ts`

**Interfaces:**
- Produces: `Membership`, `SessionUser`, `login(email, password): Promise<SessionUser>`, `fetchCurrentSession(): Promise<SessionUser|null>`, `logout(): Promise<void>`. `adminFetch` inalterado.

- [ ] **Step 1: Acrescentar testes que falham**

Append to `apps/dashboard/src/lib/api.test.ts` (mantenha o conteúdo existente; adicione os imports `login`, `fetchCurrentSession` ao import existente de `./api`):
```ts
import { login, fetchCurrentSession } from "./api";

describe("sessão", () => {
  it("login faz POST e devolve SessionUser", async () => {
    mockFetch(200, { id: "u1", email_address: "a@curitiba.demo", operator: false, memberships: [] });
    const u = await login("a@curitiba.demo", "pw");
    expect(u.email_address).toBe("a@curitiba.demo");
  });
  it("fetchCurrentSession devolve null em 401", async () => {
    mockFetch(401, { error: "unauth" });
    expect(await fetchCurrentSession()).toBe(null);
  });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/dashboard && npx vitest run src/lib/api.test.ts`
Expected: FAIL (login/fetchCurrentSession não exportados).

- [ ] **Step 3: Implementar — substituir `apps/dashboard/src/lib/api.ts` por:**

```ts
// Cliente HTTP do dashboard. Read API /admin/api/* (envelope { data, as_of })
// + sessão /session. credentials: "include" (cookie HttpOnly, ADR-0022).
const BASE = import.meta.env.VITE_ADMIN_API_BASE || "/admin/api";
const SESSION_BASE = import.meta.env.VITE_SESSION_BASE || "/session";

export interface ScopeBlock {
  municipality: { id: string | null; name: string; cross_tenant: boolean };
  period: { key: string; label: string; axis: string };
  tz: string;
}

export interface Envelope<T> {
  data: T & { scope?: ScopeBlock };
  as_of: string;
}

export class ApiError extends Error {
  status: number;
  body: unknown;
  constructor(status: number, body: unknown, message: string) {
    super(message);
    this.status = status;
    this.body = body;
  }
}

export interface Membership {
  municipality_id: string;
  municipality_name: string;
  municipality_uf: string | null;
  role: string;
}

export interface SessionUser {
  id: string;
  email_address: string;
  operator: boolean;
  memberships: Membership[];
}

async function jsonFetch<T>(input: string, init?: RequestInit): Promise<T> {
  const res = await fetch(input, {
    ...init,
    credentials: "include",
    headers: {
      Accept: "application/json",
      ...(init?.body ? { "Content-Type": "application/json" } : {}),
      ...(init?.headers || {})
    }
  });
  if (!res.ok) {
    const text = await res.text().catch(() => "");
    let body: unknown = text;
    if (text) { try { body = JSON.parse(text); } catch { /* deixa string */ } }
    throw new ApiError(res.status, body, `${res.status} on ${input}`);
  }
  if (res.status === 204) return undefined as T;
  return res.json() as Promise<T>;
}

export async function adminFetch<T>(
  path: string,
  params?: Record<string, string | undefined>
): Promise<Envelope<T>> {
  const url = new URL(BASE + path, window.location.origin);
  if (params) {
    Object.entries(params).forEach(([ k, v ]) => {
      if (v !== undefined && v !== "") url.searchParams.set(k, v);
    });
  }
  return jsonFetch<Envelope<T>>(url.toString());
}

export async function login(email_address: string, password: string): Promise<SessionUser> {
  return jsonFetch<SessionUser>(SESSION_BASE, {
    method: "POST",
    body: JSON.stringify({ email_address, password })
  });
}

export async function fetchCurrentSession(): Promise<SessionUser | null> {
  try {
    return await jsonFetch<SessionUser>(SESSION_BASE);
  } catch (err) {
    if (err instanceof ApiError && err.status === 401) return null;
    throw err;
  }
}

export async function logout(): Promise<void> {
  await jsonFetch<void>(SESSION_BASE, { method: "DELETE" });
}
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/api.test.ts`
Expected: PASS (4 tests: os 2 antigos + os 2 novos).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard
git add src/lib/api.ts src/lib/api.test.ts
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(dashboard): funções de sessão em lib/api (login/fetchCurrentSession/logout)"
```

---

### Task 2: `lib/auth.tsx` — AuthProvider (TDD)

**Files:**
- Create: `apps/dashboard/src/lib/auth.tsx`
- Test: `apps/dashboard/src/lib/auth.test.tsx`

**Interfaces:**
- Consumes: `login`, `logout`, `fetchCurrentSession`, `SessionUser` (Task 1).
- Produces: `<AuthProvider>`, `useAuth(): { state, user, municipalityId, login, logout, reload }`.

- [ ] **Step 1: Teste que falha**

`apps/dashboard/src/lib/auth.test.tsx`:
```tsx
import { describe, it, expect, vi, beforeEach } from "vitest";
import { renderHook, act, waitFor } from "@testing-library/react";
import type { ReactNode } from "react";

vi.mock("./api", () => ({
  ApiError: class ApiError extends Error {},
  fetchCurrentSession: vi.fn(),
  login: vi.fn(),
  logout: vi.fn()
}));

import { AuthProvider, useAuth } from "./auth";
import * as api from "./api";

const USER = {
  id: "u1", email_address: "admin@curitiba.demo", operator: false,
  memberships: [{ municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role: "municipal_admin" }]
};

function wrapper({ children }: { children: ReactNode }) {
  return <AuthProvider>{children}</AuthProvider>;
}

beforeEach(() => vi.clearAllMocks());

describe("AuthProvider", () => {
  it("boot sem sessão → anonymous", async () => {
    (api.fetchCurrentSession as ReturnType<typeof vi.fn>).mockResolvedValue(null);
    const { result } = renderHook(() => useAuth(), { wrapper });
    await waitFor(() => expect(result.current.state.kind).toBe("anonymous"));
    expect(result.current.municipalityId).toBe(null);
  });

  it("login ok → authenticated + municipalityId da membership", async () => {
    (api.fetchCurrentSession as ReturnType<typeof vi.fn>).mockResolvedValue(null);
    (api.login as ReturnType<typeof vi.fn>).mockResolvedValue(USER);
    const { result } = renderHook(() => useAuth(), { wrapper });
    await waitFor(() => expect(result.current.state.kind).toBe("anonymous"));
    await act(async () => { await result.current.login("admin@curitiba.demo", "pw"); });
    expect(result.current.state.kind).toBe("authenticated");
    expect(result.current.municipalityId).toBe("m1");
  });

  it("logout → anonymous", async () => {
    (api.fetchCurrentSession as ReturnType<typeof vi.fn>).mockResolvedValue(USER);
    (api.logout as ReturnType<typeof vi.fn>).mockResolvedValue(undefined);
    const { result } = renderHook(() => useAuth(), { wrapper });
    await waitFor(() => expect(result.current.state.kind).toBe("authenticated"));
    await act(async () => { await result.current.logout(); });
    expect(result.current.state.kind).toBe("anonymous");
    expect(result.current.municipalityId).toBe(null);
  });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/dashboard && npx vitest run src/lib/auth.test.tsx`
Expected: FAIL (módulo `./auth` inexistente).

- [ ] **Step 3: Implementar**

`apps/dashboard/src/lib/auth.tsx`:
```tsx
// AuthContext do dashboard. Sessão server-side (cookie HttpOnly). Sem MFA
// (usuário municipal) e sem switching de cidade (single-tenant). O tenant
// vem da primeira membership do usuário.
import {
  createContext, useCallback, useContext, useEffect, useMemo, useState,
  type ReactNode
} from "react";
import {
  fetchCurrentSession, login as apiLogin, logout as apiLogout,
  type SessionUser
} from "./api";

type AuthState =
  | { kind: "loading" }
  | { kind: "anonymous" }
  | { kind: "authenticated"; user: SessionUser };

interface AuthValue {
  state: AuthState;
  user: SessionUser | null;
  municipalityId: string | null;
  login: (email_address: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  reload: () => Promise<void>;
}

const AuthContext = createContext<AuthValue | null>(null);

function pickMunicipality(user: SessionUser): string | null {
  return user.memberships[0]?.municipality_id ?? null;
}

export function AuthProvider({ children }: { children: ReactNode }) {
  const [ state, setState ] = useState<AuthState>({ kind: "loading" });

  const reload = useCallback(async () => {
    const user = await fetchCurrentSession();
    setState(user ? { kind: "authenticated", user } : { kind: "anonymous" });
  }, []);

  useEffect(() => { void reload(); }, [ reload ]);

  const login = useCallback(async (email_address: string, password: string) => {
    const user = await apiLogin(email_address, password);
    setState({ kind: "authenticated", user });
  }, []);

  const logout = useCallback(async () => {
    await apiLogout();
    setState({ kind: "anonymous" });
  }, []);

  const value = useMemo<AuthValue>(() => {
    const user = state.kind === "authenticated" ? state.user : null;
    return {
      state,
      user,
      municipalityId: user ? pickMunicipality(user) : null,
      login,
      logout,
      reload
    };
  }, [ state, login, logout, reload ]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth(): AuthValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth precisa estar dentro de <AuthProvider>");
  return ctx;
}
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/auth.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard
git add src/lib/auth.tsx src/lib/auth.test.tsx
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(dashboard): AuthProvider (sessão, tenant da membership, sem MFA)"
```

---

### Task 3: `lib/scope.ts` — municipalityId nullable (TDD)

**Files:**
- Modify: `apps/dashboard/src/lib/scope.ts`
- Modify (test): `apps/dashboard/src/lib/scope.test.ts`

**Interfaces:**
- Produces: `Scope.municipalityId: string | null`; `scopeParams` omite `municipality_id` quando null.

- [ ] **Step 1: Acrescentar caso de teste (null) — deve falhar a tipagem/comportamento**

Em `apps/dashboard/src/lib/scope.test.ts`, adicionar dentro do `describe("scopeParams", ...)`:
```ts
  it("omite municipality_id quando null", () => {
    expect(scopeParams({ period: "7d", municipalityId: null, setPeriod: () => {} }))
      .toEqual({ period: "7d" });
  });
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/dashboard && npx vitest run src/lib/scope.test.ts`
Expected: FAIL (type error em `municipalityId: null` e/ou objeto inclui `municipality_id`).

- [ ] **Step 3: Implementar — substituir o bloco `Scope`/`scopeParams` em `apps/dashboard/src/lib/scope.ts`:**

Trocar a interface `Scope` para:
```ts
export interface Scope {
  period: PeriodKey;
  municipalityId: string | null;
  setPeriod: (p: PeriodKey) => void;
}
```
E `scopeParams` para:
```ts
export function scopeParams(scope: Scope): Record<string, string> {
  const params: Record<string, string> = { period: scope.period };
  if (scope.municipalityId) params.municipality_id = scope.municipalityId;
  return params;
}
```
(O resto de `scope.ts` — `PeriodKey`, `PERIOD_OPTIONS`, `ScopeContext`, `useScope` — fica igual.)

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/scope.test.ts`
Expected: PASS (3 tests: os 2 antigos + o null).

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard
git add src/lib/scope.ts src/lib/scope.test.ts
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(dashboard): scope.municipalityId nullable; scopeParams omite quando null"
```

---

### Task 4: `modules/Login.tsx` — form de login

**Files:**
- Create: `apps/dashboard/src/modules/Login.tsx`

**Interfaces:**
- Consumes: `useAuth` (Task 2), `ApiError` (Task 1).
- Produces: `<Login>`.

- [ ] **Step 1: Implementar**

`apps/dashboard/src/modules/Login.tsx`:
```tsx
import { useState, type FormEvent } from "react";
import { useAuth } from "../lib/auth";
import { ApiError } from "../lib/api";

export function Login() {
  const auth = useAuth();
  const [ email, setEmail ] = useState("");
  const [ password, setPassword ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);

  async function onSubmit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    setBusy(true);
    try {
      await auth.login(email, password);
    } catch (err) {
      if (err instanceof ApiError && (err.status === 401 || err.status === 422)) {
        setError("E-mail ou senha inválidos.");
      } else {
        setError("Não foi possível entrar. Tente novamente.");
      }
    } finally {
      setBusy(false);
    }
  }

  const inputStyle = {
    width: "100%", padding: 8, marginTop: 4, borderRadius: 6,
    border: "1px solid var(--line, #ccc)"
  } as const;

  return (
    <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <form onSubmit={onSubmit}
        style={{ width: 320, display: "flex", flexDirection: "column", gap: 12, padding: 24,
          border: "1px solid var(--line, #e6e6e6)", borderRadius: 10 }}>
        <strong style={{ fontFamily: "var(--font-mono, monospace)", fontSize: 14 }}>
          Rota Saúde — Dashboard
        </strong>
        <label style={{ fontSize: 12, color: "var(--ink2, #444)" }}>
          E-mail
          <input type="email" value={email} onChange={e => setEmail(e.target.value)} required autoFocus style={inputStyle} />
        </label>
        <label style={{ fontSize: 12, color: "var(--ink2, #444)" }}>
          Senha
          <input type="password" value={password} onChange={e => setPassword(e.target.value)} required style={inputStyle} />
        </label>
        {error && <p role="alert" style={{ color: "var(--danger, #c0341d)", fontSize: 12, margin: 0 }}>{error}</p>}
        <button type="submit" disabled={busy}
          style={{ padding: "8px 12px", borderRadius: 6, border: "none", cursor: busy ? "default" : "pointer",
            background: "var(--accent, #2b59ff)", color: "#fff", fontSize: 13 }}>
          {busy ? "Entrando…" : "Entrar"}
        </button>
      </form>
    </div>
  );
}
```

- [ ] **Step 2: Typecheck**

Run: `cd apps/dashboard && npm run typecheck`
Expected: sem erros.

- [ ] **Step 3: Commit**

```bash
cd apps/dashboard
git add src/modules/Login.tsx
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(dashboard): tela de Login (email+senha)"
```

---

### Task 5: Wire — `main.tsx` (gate) + `App.tsx` (tenant da sessão) + `AppHeader` (logout) + remover `lib/tenant.ts`

**Files:**
- Modify: `apps/dashboard/src/main.tsx`, `apps/dashboard/src/App.tsx`, `apps/dashboard/src/shell/AppHeader.tsx`
- Delete: `apps/dashboard/src/lib/tenant.ts`, `apps/dashboard/src/lib/tenant.test.ts`

**Interfaces:**
- Consumes: `AuthProvider`/`useAuth` (Task 2), `<Login>` (Task 4), `useScope`/`scope.municipalityId` (Task 3).
- Produces: app com gate de auth e logout; `VITE_MUNICIPALITY_ID` aposentado.

- [ ] **Step 1: Substituir `apps/dashboard/src/main.tsx`:**

```tsx
import { StrictMode, useState } from "react";
import { createRoot } from "react-dom/client";
import { QueryCache, QueryClient, QueryClientProvider } from "@tanstack/react-query";
import "@fontsource-variable/geist/index.css";
import "@fontsource-variable/geist-mono/index.css";
import { App } from "./App";
import { Login } from "./modules/Login";
import { AuthProvider, useAuth } from "./lib/auth";
import { ApiError } from "./lib/api";
import "./theme/global.css";

function AppRoot() {
  const auth = useAuth();
  const [ queryClient ] = useState(() => new QueryClient({
    queryCache: new QueryCache({
      onError(err) {
        if (err instanceof ApiError && err.status === 401) void auth.reload();
      }
    }),
    defaultOptions: {
      queries: {
        refetchOnWindowFocus: false,
        retry: (count, err) => {
          if (err instanceof ApiError && (err.status === 401 || err.status === 404)) return false;
          return count < 1;
        },
        staleTime: 30_000
      }
    }
  }));

  if (auth.state.kind === "loading") return <Splash />;
  if (auth.state.kind === "anonymous") return <Login />;
  return (
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  );
}

function Splash() {
  return (
    <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center",
      color: "var(--ink3, #888)", fontFamily: "var(--font-mono, monospace)", fontSize: 11 }}>…</div>
  );
}

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <AuthProvider>
      <AppRoot />
    </AuthProvider>
  </StrictMode>
);
```

- [ ] **Step 2: Substituir `apps/dashboard/src/App.tsx`:**

```tsx
import { useState } from "react";
import { ScopeContext, type PeriodKey } from "./lib/scope";
import { useAuth } from "./lib/auth";
import { AppHeader } from "./shell/AppHeader";
import { labelFor, type ModuleId } from "./shell/modules";
import { Overview } from "./modules/Overview";
import { Placeholder } from "./modules/Placeholder";

export function App() {
  const [ period, setPeriod ] = useState<PeriodKey>("7d");
  const [ active, setActive ] = useState<ModuleId>("overview");
  const { municipalityId } = useAuth();

  return (
    <ScopeContext.Provider value={{ period, municipalityId, setPeriod }}>
      <div style={{ minHeight: "100vh", display: "flex", flexDirection: "column" }}>
        <AppHeader active={active} onSelect={setActive} />
        <main style={{ padding: "22px 24px 48px", flex: 1, width: "100%" }}>
          {active === "overview"
            ? <Overview onNavigate={setActive} />
            : <Placeholder title={labelFor(active)} />}
        </main>
      </div>
    </ScopeContext.Provider>
  );
}
```

- [ ] **Step 3: Substituir `apps/dashboard/src/shell/AppHeader.tsx` (adiciona e-mail + Sair):**

```tsx
// Header do dashboard: nav por módulo + seletor de período + usuário/logout.
// Sem ScopePicker (single-tenant).
import { NAV_GROUPS, type ModuleId } from "./modules";
import { PERIOD_OPTIONS, useScope } from "../lib/scope";
import { useAuth } from "../lib/auth";

interface Props { active: ModuleId; onSelect: (id: ModuleId) => void; }

export function AppHeader({ active, onSelect }: Props) {
  const scope = useScope();
  const auth = useAuth();
  const items = NAV_GROUPS.flatMap(g => g.items);
  return (
    <header style={{ borderBottom: "1px solid var(--line, #e6e6e6)", padding: "10px 24px",
      display: "flex", alignItems: "center", gap: 16, flexWrap: "wrap" }}>
      <strong style={{ fontFamily: "var(--font-mono, monospace)", fontSize: 13 }}>
        Rota Saúde — Dashboard
      </strong>
      <nav style={{ display: "flex", gap: 4, flexWrap: "wrap" }}>
        {items.map(item => (
          <button key={item.id} onClick={() => onSelect(item.id)}
            aria-current={active === item.id ? "page" : undefined}
            style={{ border: "none", cursor: "pointer", padding: "4px 10px", borderRadius: 6,
              fontSize: 12, background: active === item.id ? "var(--accent-soft, #eef)" : "transparent",
              color: active === item.id ? "var(--accent, #2b59ff)" : "var(--ink2, #444)" }}>
            <span style={{ marginRight: 6 }}>{item.icon}</span>{item.label}
          </button>
        ))}
      </nav>
      <div style={{ marginLeft: "auto", display: "flex", gap: 8, alignItems: "center" }}>
        {PERIOD_OPTIONS.map(p => (
          <button key={p.key} onClick={() => scope.setPeriod(p.key)}
            aria-pressed={scope.period === p.key}
            style={{ border: "1px solid var(--line, #e6e6e6)", cursor: "pointer", padding: "4px 10px",
              borderRadius: 6, fontSize: 12,
              background: scope.period === p.key ? "var(--accent-soft, #eef)" : "transparent",
              color: scope.period === p.key ? "var(--accent, #2b59ff)" : "var(--ink2, #444)" }}>
            {p.label}
          </button>
        ))}
        {auth.user && (
          <>
            <span style={{ fontSize: 11, color: "var(--ink3, #888)" }}>{auth.user.email_address}</span>
            <button onClick={() => void auth.logout()}
              style={{ border: "1px solid var(--line, #e6e6e6)", cursor: "pointer", padding: "4px 10px",
                borderRadius: 6, fontSize: 12, background: "transparent", color: "var(--ink2, #444)" }}>
              Sair
            </button>
          </>
        )}
      </div>
    </header>
  );
}
```

- [ ] **Step 4: Remover `lib/tenant.ts` + seu teste**

```bash
cd apps/dashboard
git rm src/lib/tenant.ts src/lib/tenant.test.ts
```

- [ ] **Step 5: Typecheck + build + testes**

Run: `cd apps/dashboard && npm run typecheck && npm run build && npm run test`
Expected: typecheck/build sem erro; `npm run test` PASS (api 4, auth 3, scope 3, format 5 = 15 tests; tenant.test removido).

- [ ] **Step 6: Commit**

```bash
cd apps/dashboard
git add src/main.tsx src/App.tsx src/shell/AppHeader.tsx
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(dashboard): gate de auth no main; tenant da sessão; logout; aposenta VITE_MUNICIPALITY_ID"
```

---

### Task 6: Verificação integrada (happy-path destravado)

**Files:** nenhum (verificação)

- [ ] **Step 1: Suíte + build**

Run: `cd apps/dashboard && npm run test && npm run build`
Expected: 15 tests PASS; build ok.

- [ ] **Step 2: Run + login (manual, contra a API real)**

Pré-req: `api` de pé em `:3030` com o usuário de dev `admin@curitiba.demo` (membership em Curitiba Demo). Descobrir a senha de dev: ver `apps/api/db/seeds.rb` (campo `DEV_USER_PASSWORD` / default). Então:
```bash
cd apps/dashboard
# garantir que .env NÃO precisa mais de VITE_MUNICIPALITY_ID; só o proxy:
printf "VITE_API_PROXY_TARGET=http://localhost:3030\n" > .env
npm run dev -- --port 5188 --strictPort
```
Abrir `http://localhost:5188/dashboard/`. Esperado:
- Tela de **Login** (não o app).
- Logar com `admin@curitiba.demo` + a senha do seed → app monta.
- **Overview renderiza KPIs reais** da Curitiba Demo (não ErrorState/401), com `AsOfStamp`.
- Header mostra o e-mail + botão **Sair**; clicar em Sair → volta pro Login.

Verificação de rede (DevTools): `GET /admin/api/overview` agora retorna **200** (com cookie de sessão), não 401.

Se o login falhar com 401: confirmar a senha do seed (`docker compose exec api bin/rails db:seed` recria `admin@curitiba.demo`). Se o overview 401 mesmo logado: confirmar que o cookie de sessão está sendo enviado (`credentials: "include"` + mesmo host via proxy).

- [ ] **Step 3: Parar o dev server.**

---

## Self-review (cobertura do spec)

- §2 api.ts sessão (login/fetchCurrentSession/logout + tipos) → Task 1. ✓
- §3 lib/auth (AuthProvider sem MFA/switching, municipalityId da membership) → Task 2. ✓
- §4 scope nullable + scopeParams omite null; remover tenant.ts → Tasks 3, 5. ✓
- §5 Login enxuto → Task 4. ✓
- §6 main gate (loading/anonymous/authenticated, onError 401→reload) → Task 5. ✓
- App municipalityId da sessão; AppHeader logout → Task 5. ✓
- Testes vitest (auth + api sessão) → Tasks 1, 2. ✓
- Critério de aceite 1–6 → Task 6 (verificação). ✓
- Out-of-scope respeitado: sem MFA, sem gestão de usuários/convites, sem ScopePicker, sem outros painéis.
