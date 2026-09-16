# Dashboard — Fatia 1 (Fundação + Overview) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir a fundação do app `dashboard` (tenant-scoped) + o painel Overview ligado à API real, portando/adaptando do `admin`.

**Architecture:** Duplicar-e-adaptar do `apps/admin` (decisão C). Componentes e hooks presentacionais são copiados as-is; `lib/api`, `lib/scope`, o shell, `App`/`main` são escritos enxutos (single-tenant, sem auth/setup). Tenant vem de `VITE_MUNICIPALITY_ID`. Testes com vitest cobrem lógica pura.

**Tech Stack:** Vite + React 18 + TypeScript + @tanstack/react-query + recharts + vitest.

## Global Constraints

- Trabalhar em `apps/dashboard/` (repo local; já tem react/react-query/vite, `tsconfig` com `noEmit: true`, `vite.config` com `base: "/dashboard/"` e proxy `/admin/api`→`:3030`).
- Fonte de port: `apps/admin/src/`. Copiar arquivos as-is quando indicado; **não** reescrever o que é cp.
- Tenant **fixo** via `VITE_MUNICIPALITY_ID` — sem `ScopePicker`, sem `municipalityId: "all"`.
- API: `adminFetch` em `/admin/api/*`, envelope `{ data, as_of }`. Read-only.
- i18n: Intl `pt-BR` + texto PT-BR inline. Sem framework i18n, sem react-router.
- Cada chamada manda `municipality_id` (do env) — invariante: nenhum dado fora do tenant.
- Typecheck (`npm run typecheck` = `tsc --noEmit`) limpo ao fim de cada task que mexe em `.ts(x)`.
- Nota: o `ModuleId` inclui `health` (10 ids) porque o Overview navega para `queues`/`health`; o doc do módulo cita "9 painéis" — `health` entra como view operacional (espelha o admin). Divergência registrada.

---

### Task 1: Dependências + setup do vitest

**Files:**
- Modify: `apps/dashboard/package.json`
- Create: `apps/dashboard/vitest.config.ts`
- Create: `apps/dashboard/src/lib/sanity.test.ts` (temporário — removido no fim da task)

**Interfaces:**
- Produces: `npm run test` executável; deps `recharts` + fontes + vitest instaladas.

- [ ] **Step 1: Adicionar deps e script no `package.json`**

Em `apps/dashboard/package.json`, adicionar às `dependencies`:
```json
"recharts": "^2.13.0",
"@fontsource-variable/geist": "^5.2.9",
"@fontsource-variable/geist-mono": "^5.2.8"
```
Adicionar às `devDependencies`:
```json
"vitest": "^2.1.0",
"jsdom": "^25.0.0",
"@testing-library/react": "^16.0.0"
```
Adicionar ao bloco `scripts`:
```json
"test": "vitest run"
```

- [ ] **Step 2: Criar `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: false
  }
});
```

- [ ] **Step 3: Instalar**

Run: `cd apps/dashboard && npm install`
Expected: instala sem erro; `node_modules/.bin/vitest` existe.

- [ ] **Step 4: Teste de sanidade**

Create `apps/dashboard/src/lib/sanity.test.ts`:
```ts
import { describe, it, expect } from "vitest";
describe("sanity", () => { it("roda", () => { expect(1 + 1).toBe(2); }); });
```

- [ ] **Step 5: Rodar e verificar**

Run: `cd apps/dashboard && npm run test`
Expected: PASS (1 test). Depois **remover** `src/lib/sanity.test.ts`.

- [ ] **Step 6: Commit**

```bash
git add apps/dashboard/package.json apps/dashboard/package-lock.json apps/dashboard/vitest.config.ts
git commit -m "chore(dashboard): deps recharts/fontes + setup vitest"
```

---

### Task 2: `lib/tenant.ts` (resolução do tenant) — TDD

**Files:**
- Create: `apps/dashboard/src/lib/tenant.ts`
- Test: `apps/dashboard/src/lib/tenant.test.ts`

**Interfaces:**
- Produces: `municipalityId(): string` — lê `VITE_MUNICIPALITY_ID`, lança se ausente.

- [ ] **Step 1: Teste que falha**

`apps/dashboard/src/lib/tenant.test.ts`:
```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { municipalityId } from "./tenant";

afterEach(() => vi.unstubAllEnvs());

describe("municipalityId", () => {
  it("resolve VITE_MUNICIPALITY_ID", () => {
    vi.stubEnv("VITE_MUNICIPALITY_ID", "muni-123");
    expect(municipalityId()).toBe("muni-123");
  });
  it("lança quando ausente", () => {
    vi.stubEnv("VITE_MUNICIPALITY_ID", "");
    expect(() => municipalityId()).toThrow(/VITE_MUNICIPALITY_ID/);
  });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/dashboard && npx vitest run src/lib/tenant.test.ts`
Expected: FAIL ("Cannot find module ./tenant" ou similar).

- [ ] **Step 3: Implementar**

`apps/dashboard/src/lib/tenant.ts`:
```ts
// Fonte única do tenant na fase 1 (auth deferido — módulo 06). Quando o 06
// entrar, troca-se a origem (env → sessão) SÓ aqui.
export function municipalityId(): string {
  const id = import.meta.env.VITE_MUNICIPALITY_ID;
  if (!id || typeof id !== "string") {
    throw new Error(
      "VITE_MUNICIPALITY_ID não definido — defina o município do tenant no .env (ver .env.example)."
    );
  }
  return id;
}
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/tenant.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add apps/dashboard/src/lib/tenant.ts apps/dashboard/src/lib/tenant.test.ts
git commit -m "feat(dashboard): lib/tenant resolve VITE_MUNICIPALITY_ID"
```

---

### Task 3: `lib/scope.ts` (período + tenant fixo) — TDD

**Files:**
- Create: `apps/dashboard/src/lib/scope.ts`
- Test: `apps/dashboard/src/lib/scope.test.ts`

**Interfaces:**
- Consumes: `municipalityId` (Task 2) — usado pelo App, não pelo `scopeParams`.
- Produces: `PeriodKey`, `PERIOD_OPTIONS`, `Scope`, `ScopeContext`, `useScope()`, `scopeParams(scope): Record<string,string>`.

- [ ] **Step 1: Teste que falha**

`apps/dashboard/src/lib/scope.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { scopeParams, type Scope } from "./scope";

const base: Scope = { period: "7d", municipalityId: "muni-1", setPeriod: () => {} };

describe("scopeParams", () => {
  it("inclui period e municipality_id", () => {
    expect(scopeParams(base)).toEqual({ period: "7d", municipality_id: "muni-1" });
  });
  it("reflete a mudança de period", () => {
    expect(scopeParams({ ...base, period: "30d" }).period).toBe("30d");
  });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/dashboard && npx vitest run src/lib/scope.test.ts`
Expected: FAIL (módulo inexistente).

- [ ] **Step 3: Implementar**

`apps/dashboard/src/lib/scope.ts`:
```ts
// Escopo do painel: período + município (fixo). Propagado aos hooks via
// React Query key. Sem ScopePicker — município vem de lib/tenant.
import { createContext, useContext } from "react";

export type PeriodKey = "today" | "7d" | "30d";

export const PERIOD_OPTIONS: { key: PeriodKey; label: string }[] = [
  { key: "today", label: "Hoje" },
  { key: "7d",    label: "7 dias" },
  { key: "30d",   label: "30 dias" }
];

export interface Scope {
  period: PeriodKey;
  municipalityId: string;
  setPeriod: (p: PeriodKey) => void;
}

export const ScopeContext = createContext<Scope | null>(null);

export function useScope(): Scope {
  const ctx = useContext(ScopeContext);
  if (!ctx) throw new Error("useScope precisa estar dentro de <ScopeContext.Provider>");
  return ctx;
}

export function scopeParams(scope: Scope): Record<string, string> {
  return { period: scope.period, municipality_id: scope.municipalityId };
}
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/scope.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add apps/dashboard/src/lib/scope.ts apps/dashboard/src/lib/scope.test.ts
git commit -m "feat(dashboard): lib/scope com período + tenant fixo"
```

---

### Task 4: `lib/api.ts` (cliente enxuto) — TDD

**Files:**
- Create: `apps/dashboard/src/lib/api.ts`
- Test: `apps/dashboard/src/lib/api.test.ts`

**Interfaces:**
- Produces: `ApiError`, `Envelope<T>`, `ScopeBlock`, `adminFetch<T>(path, params?): Promise<Envelope<T>>`.

- [ ] **Step 1: Teste que falha**

`apps/dashboard/src/lib/api.test.ts`:
```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { adminFetch, ApiError } from "./api";

afterEach(() => vi.unstubAllGlobals());

function mockFetch(status: number, body: unknown) {
  vi.stubGlobal("fetch", vi.fn(async () => new Response(
    typeof body === "string" ? body : JSON.stringify(body),
    { status, headers: { "Content-Type": "application/json" } }
  )));
}

describe("adminFetch", () => {
  it("devolve o envelope { data, as_of } em 2xx", async () => {
    mockFetch(200, { data: { total: 3 }, as_of: "2026-06-26T12:00:00Z" });
    const env = await adminFetch<{ total: number }>("/overview", { period: "7d", municipality_id: "m1" });
    expect(env.data.total).toBe(3);
    expect(env.as_of).toBe("2026-06-26T12:00:00Z");
  });
  it("lança ApiError em status != 2xx", async () => {
    mockFetch(422, { error: "bad" });
    await expect(adminFetch("/overview")).rejects.toBeInstanceOf(ApiError);
  });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/dashboard && npx vitest run src/lib/api.test.ts`
Expected: FAIL (módulo inexistente).

- [ ] **Step 3: Implementar**

`apps/dashboard/src/lib/api.ts`:
```ts
// Cliente HTTP do dashboard (read-only, tenant-scoped). Consome /admin/api/*
// (namespace Admin::). Envelope universal: { data, as_of }. credentials:
// "include" para quando o módulo 06 (auth por cookie) entrar.
const BASE = import.meta.env.VITE_ADMIN_API_BASE || "/admin/api";

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

async function jsonFetch<T>(input: string): Promise<T> {
  const res = await fetch(input, {
    credentials: "include",
    headers: { Accept: "application/json" }
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
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/api.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add apps/dashboard/src/lib/api.ts apps/dashboard/src/lib/api.test.ts
git commit -m "feat(dashboard): lib/api cliente read-only + envelope"
```

---

### Task 5: `lib/format.ts` + `lib/types.ts` (cp) + teste de format

**Files:**
- Create (cp): `apps/dashboard/src/lib/format.ts`, `apps/dashboard/src/lib/types.ts`
- Test: `apps/dashboard/src/lib/format.test.ts`

**Interfaces:**
- Produces: `fmtNumber`, `fmtTime`, `fmtDateTime`, `fmtPercent`, `fmtDuration`, `fmtRelative`, `TIMEZONE` (de format); todos os tipos de dados (de types) usados pelos hooks/Overview.

- [ ] **Step 1: Copiar format e types do admin**

```bash
cd <raiz-do-monorepo>
cp apps/admin/src/lib/format.ts apps/dashboard/src/lib/format.ts
cp apps/admin/src/lib/types.ts  apps/dashboard/src/lib/types.ts
```

- [ ] **Step 2: Teste de format**

`apps/dashboard/src/lib/format.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { fmtNumber, fmtDuration } from "./format";

describe("fmtNumber", () => {
  it("formata inteiro com separador de milhar pt-BR", () => {
    expect(fmtNumber(1234)).toBe("1.234");
  });
  it("travessão para null e NaN", () => {
    expect(fmtNumber(null)).toBe("—");
    expect(fmtNumber(NaN)).toBe("—");
  });
});

describe("fmtDuration", () => {
  it("segundos", () => { expect(fmtDuration(45)).toBe("45s"); });
  it("minutos arredondados", () => { expect(fmtDuration(90)).toBe("2 min"); });
  it("travessão para null", () => { expect(fmtDuration(null)).toBe("—"); });
});
```

- [ ] **Step 3: Rodar — deve passar**

Run: `cd apps/dashboard && npx vitest run src/lib/format.test.ts`
Expected: PASS (5 asserts).

- [ ] **Step 4: Commit**

```bash
git add apps/dashboard/src/lib/format.ts apps/dashboard/src/lib/types.ts apps/dashboard/src/lib/format.test.ts
git commit -m "feat(dashboard): porta lib/format + lib/types"
```

---

### Task 6: theme + componentes (cp)

**Files:**
- Create (cp): `apps/dashboard/src/theme/tokens.ts`, `apps/dashboard/src/theme/global.css` (substituem o scaffold)
- Create (cp): todos os `apps/dashboard/src/components/*` **exceto** `QrCode.tsx`

**Interfaces:**
- Consumes: `lib/format`, `lib/types`, `theme/tokens` (Task 5 + este).
- Produces: biblioteca de componentes presentacionais (Panel, StatTile, AsOfStamp, SourceBadge, Sparkline, Funnel, Skeleton, EmptyState, ErrorState, PageHeader, KeyValue, Tag, StatusDot, Divider, etc.).

- [ ] **Step 1: Copiar theme (substitui o scaffold)**

```bash
cd <raiz-do-monorepo>
cp apps/admin/src/theme/tokens.ts  apps/dashboard/src/theme/tokens.ts
cp apps/admin/src/theme/global.css apps/dashboard/src/theme/global.css
```

- [ ] **Step 2: Copiar componentes exceto QrCode**

```bash
cd <raiz-do-monorepo>
mkdir -p apps/dashboard/src/components
cp apps/admin/src/components/*.tsx apps/dashboard/src/components/
rm -f apps/dashboard/src/components/QrCode.tsx
```

(QrCode é MFA-only, usa a dep `qrcode` que o dashboard não instala.)

- [ ] **Step 3: Typecheck**

Run: `cd apps/dashboard && npm run typecheck`
Expected: sem erros. Se algum componente importar `./QrCode`, removê-lo do consumidor (não há — QrCode só é usado por módulos de MFA, não portados).

- [ ] **Step 4: Commit**

```bash
git add apps/dashboard/src/theme apps/dashboard/src/components
git commit -m "feat(dashboard): porta theme + biblioteca de componentes"
```

---

### Task 7: hooks do Overview (cp)

**Files:**
- Create (cp): `apps/dashboard/src/hooks/{useOverview,useIngestion,useConversations,useQueues,useHealth,useEvents}.ts`

**Interfaces:**
- Consumes: `adminFetch` (Task 4), `scopeParams`/`useScope` (Task 3), tipos (Task 5).
- Produces: 6 hooks React Query, um por endpoint.

- [ ] **Step 1: Copiar os 6 hooks**

```bash
cd <raiz-do-monorepo>
mkdir -p apps/dashboard/src/hooks
for h in useOverview useIngestion useConversations useQueues useHealth useEvents; do
  cp "apps/admin/src/hooks/$h.ts" "apps/dashboard/src/hooks/$h.ts"
done
```

- [ ] **Step 2: Typecheck**

Run: `cd apps/dashboard && npm run typecheck`
Expected: sem erros (os hooks usam exatamente `adminFetch`/`scopeParams`/`useScope`/tipos já presentes).

- [ ] **Step 3: Commit**

```bash
git add apps/dashboard/src/hooks
git commit -m "feat(dashboard): porta hooks do Overview"
```

---

### Task 8: shell — `modules.ts` + `AppHeader.tsx` (novos)

**Files:**
- Create: `apps/dashboard/src/shell/modules.ts`
- Create: `apps/dashboard/src/shell/AppHeader.tsx`

**Interfaces:**
- Consumes: `PERIOD_OPTIONS`/`useScope` (Task 3).
- Produces: `ModuleId`, `NAV_GROUPS`, `<AppHeader active onSelect>`.

- [ ] **Step 1: `shell/modules.ts`**

```ts
// Catálogo de módulos do dashboard (tenant-scoped). Subconjunto operacional do
// admin — sem o grupo Setup (cross-tenant) nem ScopePicker. Inclui `health`
// porque o Overview navega para queues/health.
export type ModuleId =
  | "overview" | "ingestion" | "conversations" | "consent"
  | "triages" | "classification" | "protocols" | "events"
  | "queues" | "health";

export interface NavItem { id: ModuleId; label: string; icon: string; }
export interface NavGroupDef { label: string; items: NavItem[]; }

export const NAV_GROUPS: NavGroupDef[] = [
  { label: "Visão geral", items: [{ id: "overview", label: "Visão geral", icon: "▦" }] },
  { label: "Aquisição", items: [
    { id: "ingestion", label: "Ingestão", icon: "↘" },
    { id: "conversations", label: "Conversas", icon: "⇄" },
    { id: "consent", label: "Consentimento", icon: "✓" }
  ]},
  { label: "Triagem", items: [
    { id: "triages", label: "Triagens", icon: "≣" },
    { id: "classification", label: "Classificação", icon: "◔" }
  ]},
  { label: "Governança", items: [
    { id: "protocols", label: "Protocolos", icon: "❏" },
    { id: "events", label: "Eventos & auditoria", icon: "❖" }
  ]},
  { label: "Operação", items: [
    { id: "queues", label: "Filas & jobs", icon: "≋" },
    { id: "health", label: "Saúde", icon: "◍" }
  ]}
];

export function labelFor(id: ModuleId): string {
  return NAV_GROUPS.flatMap(g => g.items).find(i => i.id === id)?.label ?? id;
}
```

- [ ] **Step 2: `shell/AppHeader.tsx` (mínimo, self-contained)**

```tsx
// Header enxuto do dashboard: nav por módulo + seletor de período. SEM
// ScopePicker (single-tenant) e SEM NotificationCenter (fora da Fatia 1).
import { NAV_GROUPS, type ModuleId } from "./modules";
import { PERIOD_OPTIONS, useScope } from "../lib/scope";

interface Props { active: ModuleId; onSelect: (id: ModuleId) => void; }

export function AppHeader({ active, onSelect }: Props) {
  const scope = useScope();
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
      <div style={{ marginLeft: "auto", display: "flex", gap: 4 }}>
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
      </div>
    </header>
  );
}
```

- [ ] **Step 3: Typecheck**

Run: `cd apps/dashboard && npm run typecheck`
Expected: sem erros.

- [ ] **Step 4: Commit**

```bash
git add apps/dashboard/src/shell
git commit -m "feat(dashboard): shell modules + AppHeader (nav + período, sem ScopePicker)"
```

---

### Task 9: `modules/Overview.tsx` (cp) + `modules/Placeholder.tsx` (novo)

**Files:**
- Create (cp): `apps/dashboard/src/modules/Overview.tsx`
- Create: `apps/dashboard/src/modules/Placeholder.tsx`

**Interfaces:**
- Consumes: hooks (Task 7), componentes (Task 6), `fmtNumber`/`fmtTime` (Task 5), `ModuleId` (Task 8).
- Produces: `<Overview onNavigate?>`, `<Placeholder title>`.

- [ ] **Step 1: Copiar Overview**

```bash
cp <raiz-do-monorepo>/apps/admin/src/modules/Overview.tsx \
   <raiz-do-monorepo>/apps/dashboard/src/modules/Overview.tsx
```

- [ ] **Step 2: `modules/Placeholder.tsx`**

```tsx
// Placeholder honesto para os módulos ainda não ligados nesta fatia.
export function Placeholder({ title }: { title: string }) {
  return (
    <div>
      <h1 style={{ fontSize: 18, margin: "0 0 8px" }}>{title}</h1>
      <p style={{ color: "var(--ink3, #888)", fontSize: 13 }}>
        Painel ainda não ligado nesta fatia. Em breve.
      </p>
    </div>
  );
}
```

- [ ] **Step 3: Typecheck**

Run: `cd apps/dashboard && npm run typecheck`
Expected: sem erros. (Overview importa `ModuleId` de `../shell/modules` — presente; componentes e hooks — presentes.)

- [ ] **Step 4: Commit**

```bash
git add apps/dashboard/src/modules
git commit -m "feat(dashboard): porta Overview + Placeholder"
```

---

### Task 10: `App.tsx` + `main.tsx` (novos) + verificação integrada

**Files:**
- Create (substitui scaffold): `apps/dashboard/src/App.tsx`, `apps/dashboard/src/main.tsx`

**Interfaces:**
- Consumes: tudo das tasks anteriores.
- Produces: app montado — Overview ligado, outros 8 módulos via Placeholder.

- [ ] **Step 1: `App.tsx`**

```tsx
import { useState } from "react";
import { ScopeContext, type PeriodKey } from "./lib/scope";
import { municipalityId } from "./lib/tenant";
import { AppHeader } from "./shell/AppHeader";
import { labelFor, type ModuleId } from "./shell/modules";
import { Overview } from "./modules/Overview";
import { Placeholder } from "./modules/Placeholder";

export function App() {
  const [ period, setPeriod ] = useState<PeriodKey>("7d");
  const [ active, setActive ] = useState<ModuleId>("overview");
  const muni = municipalityId();

  return (
    <ScopeContext.Provider value={{ period, municipalityId: muni, setPeriod }}>
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

- [ ] **Step 2: `main.tsx`**

```tsx
import { StrictMode, useState } from "react";
import { createRoot } from "react-dom/client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import "@fontsource-variable/geist/index.css";
import "@fontsource-variable/geist-mono/index.css";
import { App } from "./App";
import { ApiError } from "./lib/api";
import "./theme/global.css";

function Root() {
  const [ queryClient ] = useState(() => new QueryClient({
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
  return (
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  );
}

createRoot(document.getElementById("root")!).render(
  <StrictMode><Root /></StrictMode>
);
```

- [ ] **Step 3: Typecheck + build**

Run: `cd apps/dashboard && npm run typecheck && npm run build`
Expected: ambos sem erro (`tsc -b && vite build` gera `dist/`).

- [ ] **Step 4: Suíte de testes completa**

Run: `cd apps/dashboard && npm run test`
Expected: PASS — tenant (2), scope (2), api (2), format (5).

- [ ] **Step 5: Verificação rodando (manual)**

Pré-req: `api` de pé em `:3030` (repo `rotasaude/api`), e um município no banco de dev. Então:
```bash
cd apps/dashboard
echo "VITE_MUNICIPALITY_ID=<id-de-um-municipio-de-dev>" >> .env   # ou .env.local
echo "VITE_API_PROXY_TARGET=http://localhost:3030" >> .env
npm run dev
```
Abrir `http://localhost:5173/dashboard/`. Esperado:
- Header com os 10 itens de nav + seletor de período.
- **Overview** renderiza KPIs + painéis-resumo (zerados em dev limpo), com `AsOfStamp`, estados loading/erro/vazio funcionando.
- Clicar em outro módulo → `Placeholder` honesto.
- Sem seletor de cidade em lugar nenhum.

Se o Overview der erro de rede: confirmar que `municipality_id` está indo na request (DevTools → Network) e que o `api` responde em `/admin/api/overview`.

- [ ] **Step 6: Commit**

```bash
git add apps/dashboard/src/App.tsx apps/dashboard/src/main.tsx
git commit -m "feat(dashboard): App + main — Overview ligado, demais via Placeholder"
```

---

## Self-review (cobertura do spec)

- §1 Reuso (C): cp components/hooks/Overview; api/scope/shell/App/main escritos enxutos. → Tasks 4–10. ✓
- §2 Tenant fixo `VITE_MUNICIPALITY_ID`, sem ScopePicker, mantém período. → Tasks 2, 3, 8. ✓
- §3 Fundação (main, App, nav 9+ módulos, AsOfStamp via Panel). → Tasks 8, 9, 10. ✓
- §4 Overview com 6 hooks + estados. → Tasks 7, 9. ✓
- §5 Data flow (React Query, envelope, scopeParams). → Tasks 3, 4, 7. ✓
- §6 Testes vitest (lógica pura: tenant, scope, api, format). → Tasks 1–5. ✓
- Critério de aceite 1–6 → Task 10 (verificação). ✓
- Out-of-scope respeitado: sem auth/login, sem outros 8 painéis ligados, sem react-router, sem testes visuais.
