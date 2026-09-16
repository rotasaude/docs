# F-04.5 — Renderização do relatório no wpda — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dar ao cidadão uma página pública no `wpda` que renderiza o relatório a partir do token assinado.

**Architecture:** O `wpda` lê `?token` da URL, busca `GET /r/:token` (JSON já existente) via proxy, e renderiza (loading/relatório/inválido/erro). O backend passa a gerar o link do cidadão apontando pro `wpda` (`WPDA_PUBLIC_BASE`).

**Tech Stack:** Vite + React 18 + TypeScript + vitest (wpda); Rails + RSpec (api).

## Global Constraints

- `wpda` = app **do cidadão**: público, sem login, sem react-router. Token via query param `?token=`.
- Backend `/r/:token` JSON fica **intacto** (`{ tier, priority, summary, completed_at, expires_at }`, 404 se inválido/expirado).
- i18n: PT-BR inline + Intl `America/Sao_Paulo`. Mobile-first.
- **Dois repos git locais (sem remote):** `apps/wpda` (branch `main`) e `apps/api` (branch atual `fix/migrations-owner-ddl-as-admin`). Rodar `git` dentro de cada repo com paths relativos. Specs do backend rodam no container: `docker exec api-dev bundle exec rspec <path>`.
- Commit identity: `git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit ...`. **Mensagens de commit em INGLÊS.**

---

### Task 1: Backend — `ReportSnapshot#url` aponta pro wpda (TDD)

**Files (em `apps/api`):**
- Modify: `app/models/report_snapshot.rb`
- Create: `spec/models/report_snapshot_spec.rb`

**Interfaces:**
- Produces: `ReportSnapshot#url` → `"<WPDA_PUBLIC_BASE sem barra final>/?token=<token>"`.

- [ ] **Step 1: Teste que falha**

`apps/api/spec/models/report_snapshot_spec.rb`:
```ruby
require "rails_helper"

RSpec.describe ReportSnapshot, type: :model do
  describe "#url" do
    around do |ex|
      orig = ENV["WPDA_PUBLIC_BASE"]
      ENV["WPDA_PUBLIC_BASE"] = "https://wpda.example/"
      ex.run
      ENV["WPDA_PUBLIC_BASE"] = orig
    end

    it "aponta pro wpda com o token em query param (sem barra dupla)" do
      snap = ReportSnapshot.new(token: "abc123")
      expect(snap.url).to eq("https://wpda.example/?token=abc123")
    end
  end
end
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `docker exec api-dev bundle exec rspec spec/models/report_snapshot_spec.rb`
Expected: FAIL (url atual usa `report_url`, retorna `http://localhost:.../r/abc123`).

- [ ] **Step 3: Implementar — em `apps/api/app/models/report_snapshot.rb`, trocar o método `url`:**

De:
```ruby
  def url
    Rails.application.routes.url_helpers.report_url(token: token)
  end
```
Para:
```ruby
  def url
    base = ENV.fetch("WPDA_PUBLIC_BASE", "http://localhost:5176/wpda")
    "#{base.chomp('/')}/?token=#{token}"
  end
```

- [ ] **Step 4: Rodar — deve passar**

Run: `docker exec api-dev bundle exec rspec spec/models/report_snapshot_spec.rb`
Expected: PASS (1 example).

- [ ] **Step 5: Commit**

```bash
cd apps/api
git add app/models/report_snapshot.rb spec/models/report_snapshot_spec.rb
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(api): ReportSnapshot#url points to wpda via WPDA_PUBLIC_BASE"
```

---

### Task 2: wpda — deps (vitest) + proxy `/r`

**Files (em `apps/wpda`):**
- Modify: `package.json`, `vite.config.ts`
- Create: `vitest.config.ts`

**Interfaces:**
- Produces: `npm run test` executável; proxy dev `/r` → api.

- [ ] **Step 1: `package.json` — adicionar devDeps + script**

Em `apps/wpda/package.json`, adicionar às `devDependencies`:
```json
"vitest": "^2.1.0",
"jsdom": "^25.0.0"
```
e ao bloco `scripts`:
```json
"test": "vitest run"
```

- [ ] **Step 2: `vitest.config.ts`**

`apps/wpda/vitest.config.ts`:
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

- [ ] **Step 3: `vite.config.ts` — adicionar proxy `/r` e corrigir comentário legado**

Em `apps/wpda/vite.config.ts`, trocar o comentário do topo (que diz "autoria de protocolo") por:
```ts
// Frontend WPDA — app público do cidadão (relatório por token assinado).
// Em dev, Vite proxa rotas de backend para o Rails (apps/api, :3030).
//   /up         → healthcheck do Rails.
//   /r          → ReportsController (relatório público por token).
```
E no bloco `proxy`, adicionar a entrada `/r` (manter as existentes):
```ts
    proxy: {
      "/up":        proxy(TARGET),
      "/r":         proxy(TARGET),
      "/admin/api": proxy(TARGET),
      "/session":   proxy(TARGET)
    }
```

- [ ] **Step 4: Instalar + sanidade**

Run: `cd apps/wpda && npm install`
Expected: instala sem erro; `node_modules/.bin/vitest` existe.

- [ ] **Step 5: Commit**

```bash
cd apps/wpda
git add package.json package-lock.json vitest.config.ts vite.config.ts
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "chore(wpda): vitest setup + /r dev proxy"
```

---

### Task 3: wpda — `lib/report.ts` (TDD)

**Files (em `apps/wpda`):**
- Create: `src/lib/report.ts`, `src/lib/report.test.ts`

**Interfaces:**
- Produces: `interface Report`, `tokenFromUrl(search?): string|null`, `fetchReport(token): Promise<Report|null>`.

- [ ] **Step 1: Teste que falha**

`apps/wpda/src/lib/report.test.ts`:
```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { tokenFromUrl, fetchReport } from "./report";

describe("tokenFromUrl", () => {
  it("extrai ?token", () => { expect(tokenFromUrl("?token=abc")).toBe("abc"); });
  it("null quando ausente", () => { expect(tokenFromUrl("")).toBe(null); });
  it("null quando vazio", () => { expect(tokenFromUrl("?token=")).toBe(null); });
});

afterEach(() => vi.unstubAllGlobals());

function mockFetch(status: number, body?: unknown) {
  vi.stubGlobal("fetch", vi.fn(async () => new Response(
    body !== undefined ? JSON.stringify(body) : "",
    { status, headers: { "Content-Type": "application/json" } }
  )));
}

describe("fetchReport", () => {
  it("devolve Report em 200", async () => {
    mockFetch(200, { tier: "rotina", priority: "baixa", summary: "ok", completed_at: "2026-06-26T12:00:00Z", expires_at: null });
    const r = await fetchReport("abc");
    expect(r?.summary).toBe("ok");
  });
  it("null em 404", async () => { mockFetch(404); expect(await fetchReport("x")).toBe(null); });
  it("lança em 500", async () => { mockFetch(500); await expect(fetchReport("x")).rejects.toThrow(); });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/wpda && npx vitest run src/lib/report.test.ts`
Expected: FAIL (módulo inexistente).

- [ ] **Step 3: Implementar**

`apps/wpda/src/lib/report.ts`:
```ts
// Relatório público do cidadão. Lê o token da URL e busca o JSON congelado
// de GET /r/:token (sem login). 404 = token inválido/expirado.
export interface Report {
  tier: string | null;
  priority: string | null;
  summary: string | null;
  completed_at: string | null;
  expires_at: string | null;
}

export function tokenFromUrl(search: string = window.location.search): string | null {
  const t = new URLSearchParams(search).get("token");
  return t && t.trim() !== "" ? t : null;
}

export async function fetchReport(token: string): Promise<Report | null> {
  const res = await fetch(`/r/${encodeURIComponent(token)}`, {
    headers: { Accept: "application/json" }
  });
  if (res.status === 404) return null;
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<Report>;
}
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/wpda && npx vitest run src/lib/report.test.ts`
Expected: PASS (6 asserts).

- [ ] **Step 5: Commit**

```bash
cd apps/wpda
git add src/lib/report.ts src/lib/report.test.ts
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(wpda): report lib (tokenFromUrl + fetchReport)"
```

---

### Task 4: wpda — `lib/format.ts` (TDD)

**Files (em `apps/wpda`):**
- Create: `src/lib/format.ts`, `src/lib/format.test.ts`

**Interfaces:**
- Produces: `fmtDateTime(iso): string`.

- [ ] **Step 1: Teste que falha**

`apps/wpda/src/lib/format.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { fmtDateTime } from "./format";

describe("fmtDateTime", () => {
  it("formata ISO em pt-BR (data)", () => {
    expect(fmtDateTime("2026-06-26T15:00:00Z")).toMatch(/26\/06\/2026/);
  });
  it("— para null", () => { expect(fmtDateTime(null)).toBe("—"); });
  it("— para data inválida", () => { expect(fmtDateTime("xxx")).toBe("—"); });
});
```

- [ ] **Step 2: Rodar — deve falhar**

Run: `cd apps/wpda && npx vitest run src/lib/format.test.ts`
Expected: FAIL (módulo inexistente).

- [ ] **Step 3: Implementar**

`apps/wpda/src/lib/format.ts`:
```ts
const dateTimeFmt = new Intl.DateTimeFormat("pt-BR", {
  timeZone: "America/Sao_Paulo",
  day: "2-digit", month: "2-digit", year: "numeric",
  hour: "2-digit", minute: "2-digit"
});

export function fmtDateTime(iso: string | null | undefined): string {
  if (!iso) return "—";
  const d = new Date(iso);
  if (Number.isNaN(d.getTime())) return "—";
  return dateTimeFmt.format(d);
}
```

- [ ] **Step 4: Rodar — deve passar**

Run: `cd apps/wpda && npx vitest run src/lib/format.test.ts`
Expected: PASS (3 asserts).

- [ ] **Step 5: Commit**

```bash
cd apps/wpda
git add src/lib/format.ts src/lib/format.test.ts
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(wpda): pt-BR date format helper"
```

---

### Task 5: wpda — `modules/Report.tsx` + `App.tsx` (UI)

**Files (em `apps/wpda`):**
- Create: `src/modules/Report.tsx`
- Modify: `src/App.tsx` (substitui o stub)

**Interfaces:**
- Consumes: `fetchReport`/`tokenFromUrl`/`Report` (Task 3), `fmtDateTime` (Task 4).

- [ ] **Step 1: `src/modules/Report.tsx`**

```tsx
import { useEffect, useState, type ReactNode } from "react";
import { fetchReport, type Report as ReportData } from "../lib/report";
import { fmtDateTime } from "../lib/format";

type State =
  | { kind: "loading" }
  | { kind: "ok"; data: ReportData }
  | { kind: "invalid" }
  | { kind: "error" };

export function Report({ token }: { token: string }) {
  const [ state, setState ] = useState<State>({ kind: "loading" });

  useEffect(() => {
    let alive = true;
    fetchReport(token)
      .then(data => { if (alive) setState(data ? { kind: "ok", data } : { kind: "invalid" }); })
      .catch(() => { if (alive) setState({ kind: "error" }); });
    return () => { alive = false; };
  }, [ token ]);

  if (state.kind === "loading") {
    return <Centered><p style={{ color: "var(--ink3, #888)" }}>Carregando…</p></Centered>;
  }
  if (state.kind === "invalid") {
    return <Centered><Message title="Link inválido" body="Este link é inválido ou expirou." /></Centered>;
  }
  if (state.kind === "error") {
    return <Centered><Message title="Ops" body="Não foi possível carregar. Tente novamente." /></Centered>;
  }

  const r = state.data;
  return (
    <Centered>
      <article style={{ maxWidth: 520, width: "100%" }}>
        <header style={{ marginBottom: 16 }}>
          <strong style={{ fontFamily: "var(--font-mono, monospace)", fontSize: 13 }}>Rota Saúde</strong>
          <h1 style={{ fontSize: 20, margin: "8px 0 0" }}>Seu resultado</h1>
        </header>
        <div style={{ marginBottom: 12 }}>
          {r.tier && (
            <span style={{ display: "inline-block", padding: "2px 10px", borderRadius: 999,
              background: "var(--accent-soft, #eef)", color: "var(--accent, #2b59ff)", fontSize: 12, marginRight: 8 }}>
              {r.tier}
            </span>
          )}
          {r.priority && <span style={{ fontSize: 12, color: "var(--ink2, #555)" }}>prioridade: {r.priority}</span>}
        </div>
        {r.summary && <p style={{ fontSize: 15, lineHeight: 1.5, margin: "0 0 24px" }}>{r.summary}</p>}
        <footer style={{ fontSize: 12, color: "var(--ink3, #888)", borderTop: "1px solid var(--line, #eee)", paddingTop: 12 }}>
          {r.completed_at && <div>Realizado em {fmtDateTime(r.completed_at)}.</div>}
          {r.expires_at && <div>Válido até {fmtDateTime(r.expires_at)}.</div>}
          <p style={{ marginTop: 12 }}>Estas informações são pessoais — não compartilhe este link.</p>
        </footer>
      </article>
    </Centered>
  );
}

function Centered({ children }: { children: ReactNode }) {
  return (
    <main style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center",
      padding: 24, fontFamily: "system-ui, sans-serif" }}>
      {children}
    </main>
  );
}

function Message({ title, body }: { title: string; body: string }) {
  return (
    <div style={{ textAlign: "center", maxWidth: 360 }}>
      <h1 style={{ fontSize: 18, margin: "0 0 8px" }}>{title}</h1>
      <p style={{ color: "var(--ink3, #888)", fontSize: 14 }}>{body}</p>
    </div>
  );
}
```

- [ ] **Step 2: `src/App.tsx` (substituir o stub)**

```tsx
import { tokenFromUrl } from "./lib/report";
import { Report } from "./modules/Report";

export function App() {
  const token = tokenFromUrl();
  if (!token) {
    return (
      <main style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center",
        padding: 24, fontFamily: "system-ui, sans-serif" }}>
        <div style={{ textAlign: "center", maxWidth: 360 }}>
          <strong style={{ fontFamily: "var(--font-mono, monospace)", fontSize: 13 }}>Rota Saúde</strong>
          <h1 style={{ fontSize: 18, margin: "8px 0" }}>Link inválido</h1>
          <p style={{ color: "var(--ink3, #888)", fontSize: 14 }}>
            Use o link enviado por WhatsApp para ver seu resultado.
          </p>
        </div>
      </main>
    );
  }
  return <Report token={token} />;
}
```

- [ ] **Step 3: Typecheck**

Run: `cd apps/wpda && npm run typecheck`
Expected: sem erros.

- [ ] **Step 4: Commit**

```bash
cd apps/wpda
git add src/modules/Report.tsx src/App.tsx
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "feat(wpda): public report page (F-04.5)"
```

---

### Task 6: Verificação integrada

**Files:** nenhum (verificação)

- [ ] **Step 1: Suíte + build do wpda**

Run: `cd apps/wpda && npm run test && npm run typecheck && npm run build`
Expected: testes PASS (report 6 + format 3 = 9); typecheck/build sem erro.

- [ ] **Step 2: Spec do backend**

Run: `docker exec api-dev bundle exec rspec spec/models/report_snapshot_spec.rb`
Expected: PASS (1 example).

- [ ] **Step 3: Smoke do 404 (garantido, sem seed)**

Subir o wpda (container ou dev) e bater com um token inválido:
```bash
cd apps/wpda
docker compose -f ../../docker-compose.yml up -d wpda 2>/dev/null || (printf "VITE_API_PROXY_TARGET=http://localhost:3030\n" > .env && npm run dev -- --port 5176 --strictPort &)
sleep 4
# via proxy do wpda, /r/<token-inválido> deve dar 404 na API:
curl -s -o /dev/null -w "/r/invalid via wpda proxy → HTTP %{http_code} (esperado 404)\n" http://localhost:5176/r/invalidtoken
```
Abrir `http://localhost:5176/wpda/?token=invalidtoken` no browser → deve mostrar **"Este link é inválido ou expirou."** (não JSON, não erro cru).

- [ ] **Step 4: (Opcional) happy-path com token real**

Se houver uma `Triage` em dev, mintar um snapshot e abrir a página:
```bash
docker exec api-dev bin/rails runner '
t = (Triage.respond_to?(:completed) ? Triage.completed.first : nil) || Triage.first
if t && t.protocol_definition
  tok = ReportSnapshot.mint_token
  ReportSnapshot.create!(triage: t, protocol_definition: t.protocol_definition, token: tok,
    signature: ReportSnapshot.sign(tok),
    payload: {"tier"=>"rotina","priority"=>"baixa","summary"=>"Resultado de demonstração.","completed_at"=>Time.current.iso8601},
    outcome: {}, expires_at: 30.days.from_now)
  puts "TOKEN=#{tok}"
else
  puts "sem Triage em dev — pular (o 404 smoke + os testes mockados cobrem a renderização)"
end'
```
Se imprimir `TOKEN=...`, abrir `http://localhost:5176/wpda/?token=<TOKEN>` → relatório renderizado (summary, tier, prioridade, data).

---

## Self-review (cobertura do spec)

- §Backend (ReportSnapshot#url + WPDA_PUBLIC_BASE + spec) → Task 1. ✓
- §wpda vite proxy `/r` + vitest → Task 2. ✓
- §lib/report (tokenFromUrl, fetchReport, Report) → Task 3. ✓
- §lib/format → Task 4. ✓
- §Report.tsx (loading/relatório/inválido/erro) + App.tsx (sem token) → Task 5. ✓
- §Renderização (tier/priority/summary/completed_at/expires_at, PT-BR, nota privacidade) → Task 5. ✓
- §Testes (backend spec; wpda report+format) → Tasks 1,3,4. ✓
- Critério de aceite 1–6 → Task 6. ✓
- Out-of-scope respeitado: sem F-03.17, sem react-router, endpoint /r/:token intacto.
