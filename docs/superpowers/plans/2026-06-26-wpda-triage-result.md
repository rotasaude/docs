# F-03.17 — Resultado da triagem no wpda (corrige F-04.5) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Renderizar corretamente o resultado da triagem no wpda — corrigindo o bug do `summary` (que é o trail, não string) do F-04.5.

**Architecture:** Só frontend (`apps/wpda`). O tipo `Report.summary` passa a ser o trail (`TrailEntry[]`), e o `Report.tsx` para de renderizá-lo como string — mostra `tier`/`priority`/data + uma nota genérica não-prescritiva. O endpoint `/r/:token` e a lógica de fetch ficam intactos.

**Tech Stack:** Vite + React 18 + TypeScript + vitest.

## Global Constraints

- Só `apps/wpda` (repo git local, branch `main`). Git de dentro de `apps/wpda`, paths relativos. Commit em **INGLÊS** com a identidade abaixo.
- `tier` é definido pelo protocolo (string arbitrária) — renderizar **como vem**, SEM mapa de recomendação por tier no cliente.
- Não renderizar o trail (step-ids crípticos). Endpoint `/r/:token` JSON intacto.
- Commit identity: `git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit ...`

---

### Task 1: Corrigir tipo + render do resultado

**Files (em `apps/wpda`):**
- Modify: `src/lib/report.ts`, `src/lib/report.test.ts`, `src/modules/Report.tsx`

**Interfaces:**
- Produces: `interface TrailEntry`; `Report.summary: TrailEntry[] | null`. `tokenFromUrl`/`fetchReport` inalterados.

- [ ] **Step 1: Atualizar o teste pro shape real (`summary` = array)**

Em `apps/wpda/src/lib/report.test.ts`, no `describe("fetchReport")`, trocar o caso "devolve Report em 200":
```ts
  it("devolve Report em 200 (summary é o trail)", async () => {
    mockFetch(200, {
      tier: "alta", priority: "1",
      summary: [{ step: "s1", answer: "sim" }],
      completed_at: "2026-06-26T12:00:00Z", expires_at: null
    });
    const r = await fetchReport("abc");
    expect(r?.tier).toBe("alta");
    expect(r?.summary?.[0]?.step).toBe("s1");
  });
```
(Os casos 404/500 e o `describe("tokenFromUrl")` ficam iguais.)

- [ ] **Step 2: Corrigir o tipo em `src/lib/report.ts`**

Trocar a `interface Report` por (adicionar `TrailEntry`; `summary` deixa de ser `string`):
```ts
export interface TrailEntry { step: string; answer: string }

export interface Report {
  tier: string | null;
  priority: string | null;
  summary: TrailEntry[] | null;
  completed_at: string | null;
  expires_at: string | null;
}
```
(`tokenFromUrl` e `fetchReport` ficam exatamente como estão.)

- [ ] **Step 3: Rodar typecheck — deve FALHAR no `Report.tsx`**

Run: `cd apps/wpda && npm run typecheck`
Expected: FAIL em `src/modules/Report.tsx` — `{r.summary}` (agora `TrailEntry[]`) não é um `ReactNode` válido. **Esse erro É o bug do F-04.5.**

- [ ] **Step 4: Corrigir o render em `src/modules/Report.tsx`**

No estado `ok` (o bloco `return (<Centered><article>...`), **remover** a linha:
```tsx
        {r.summary && <p style={{ fontSize: 15, lineHeight: 1.5, margin: "0 0 24px" }}>{r.summary}</p>}
```
e **colocar no lugar** a nota genérica:
```tsx
        <p style={{ fontSize: 15, lineHeight: 1.5, margin: "0 0 24px" }}>
          Este é o resultado da sua triagem. Siga as orientações da sua unidade de saúde.
          Se os sintomas piorarem, procure atendimento.
        </p>
```
(O badge de `tier`, o `priority`, o `completed_at`/`expires_at` e a nota de privacidade ficam como estão. O `tier` já é renderizado como vem.)

- [ ] **Step 5: Rodar typecheck + testes + build — deve PASSAR**

Run: `cd apps/wpda && npm run typecheck && npm run test && npm run build`
Expected: typecheck sem erro; testes PASS (report 6 + format 3 = 9); build ok.

- [ ] **Step 6: Commit**

```bash
cd apps/wpda
git add src/lib/report.ts src/lib/report.test.ts src/modules/Report.tsx
git -c user.name="Eduardo Rocha" -c user.email="eduardo.vinicius.rocha@gmail.com" commit -m "fix(wpda): render triage result correctly (summary is the trail, not a string)"
```

---

### Task 2: Verificação ao vivo

**Files:** nenhum (verificação)

- [ ] **Step 1: Container wpda pega a mudança (HMR) + smoke do 404**

```bash
cd <raiz-do-monorepo>
docker compose up -d wpda 2>/dev/null; sleep 3
curl -s -o /dev/null -w "/r/invalid via wpda proxy → HTTP %{http_code} (esperado 404)\n" http://localhost:5176/r/invalidtoken
curl -s -o /dev/null -w "/wpda/ → HTTP %{http_code}\n" http://localhost:5176/wpda/
```
Expected: `/r/invalid` = 404; `/wpda/` = 200. Abrir `http://localhost:5176/wpda/?token=invalidtoken` → "Este link é inválido ou expirou."

- [ ] **Step 2: (Opcional) happy-path com token real**

Se houver `Triage` em dev, mintar um snapshot com `summary` = trail real e abrir a página:
```bash
docker exec api-dev bin/rails runner '
t = (Triage.respond_to?(:completed) ? Triage.completed.first : nil) || Triage.first
if t && t.protocol_definition
  tok = ReportSnapshot.mint_token
  ReportSnapshot.create!(triage: t, protocol_definition: t.protocol_definition, token: tok,
    signature: ReportSnapshot.sign(tok),
    payload: {"tier"=>"alta","priority"=>"1","summary"=>[{"step"=>"s1","answer"=>"sim"}],"completed_at"=>Time.current.iso8601},
    outcome: {}, expires_at: 30.days.from_now)
  puts "TOKEN=#{tok}"
else
  puts "sem Triage — smoke 404 + testes cobrem"
end' 2>&1 | tail -1
```
Se imprimir `TOKEN=...`, abrir `http://localhost:5176/wpda/?token=<TOKEN>` → resultado renderiza (tier "alta", nota genérica, data) **sem** `[object Object]`.

---

## Self-review (cobertura do spec)

- §Correção de tipo (`summary` → `TrailEntry[]`) → Task 1 Step 2. ✓
- §Render: remover summary-string, nota genérica, tier como vem → Task 1 Step 4. ✓
- §Teste atualizado (summary array) → Task 1 Step 1. ✓
- §Trail omitido → Task 1 Step 4 (não há render do trail). ✓
- Critério de aceite 1–5 → Tasks 1 (typecheck/test/build) + 2 (smoke/visual). ✓
- Out-of-scope respeitado: sem recomendação por tier, sem render de trail, endpoint intacto.
