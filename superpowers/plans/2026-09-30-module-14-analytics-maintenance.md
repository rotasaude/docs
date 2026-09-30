# Módulo 14 — Analytics (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bloco "Analytics" na ficha da cidade do console de manutenção (F-14.9): o estado do pipeline (`analyticsStatus`), com atraso e falha em destaque e o último erro à vista, e os seis indicadores publicados das últimas 12 semanas (`analyticsIndicators`), com "oculto" e "sem dado" distintos.

**Architecture:** O bloco é uma **aba nova, "Analytics"**, no detalhe da cidade, montada por um componente próprio (`AnalyticsTab`, como o `ProtocolsTab`) que só consulta quando a aba é aberta. Ele faz **duas consultas GraphQL separadas**, nenhuma delas junto da consulta de topo (`CityHeader`): `CityAnalyticsStatus` (lê `analytics_runs` no banco da cidade) e `CityAnalyticsIndicators` (lê `city_analytics_indicators` na plataforma). As regras puras (janela das 12 semanas no dia da cidade, texto da célula, rótulos, datas) ficam em `src/lib/analytics.ts`.

**Por que consultas separadas (ordem de deploy):** a validação do GraphQL recusa o **documento inteiro** quando um campo não existe no schema. Um build novo do maintenance contra um api que ainda não tem `analyticsStatus`/`analyticsIndicators` derrubaria qualquer consulta que os pedisse, inclusive a do topo da ficha, se os campos fossem acrescentados a ela. Com consultas próprias, um api antigo só faz a aba Analytics mostrar o aviso; o topo e as outras abas continuam de pé. A separação entre as duas consultas de Analytics tem um segundo motivo: `analyticsStatus` é `AnalyticsStatus!` (não nulo), então um `CITY_UNREACHABLE` nele anula `city` inteira na resposta. Na mesma consulta, isso apagaria também os indicadores, que vêm da plataforma e continuam disponíveis.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL Codegen (client preset, `src/gql/` gerado e commitado), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-09-30-module-14-analytics-design.md` (§6.4, §9, §10.3 e §12 são deste plano) e `docs/adr/0025.md`. O schema GraphQL está fixado em `docs/superpowers/plans/2026-09-30-module-14-analytics-contracts.md` §3, o mesmo arquivo que o plano do api usa. O plano do api (escrito em paralelo) precisa estar **na branch do api e rodando** antes da Task 2 (o `schema.graphql` é extraído do api) e **mergeado antes** do merge deste.

## Global Constraints

- Schema (contratos §3, literal):
  ```graphql
  type AnalyticsIndicator { weekStart: ISO8601Date!  indicator: String!  value: Float  suppressed: Boolean! }
  type AnalyticsStatus {
    lastRunStatus: String        # running | succeeded | failed | null (nunca rodou)
    lastSucceededAt: ISO8601DateTime
    lastPublishedAt: ISO8601DateTime
    lastError: String            # só da última execução se failed, ou erro de publicação
    stale: Boolean!              # lastSucceededAt nulo ou > 36 h
  }
  extend type City {
    analyticsIndicators(from: ISO8601Date!, to: ISO8601Date!): [AnalyticsIndicator!]!   # até 104 semanas
    analyticsStatus: AnalyticsStatus!
  }
  ```
- `analyticsIndicators` devolve só as linhas que existem: semana × indicador sem linha = **"sem dado"**; `suppressed: true` (com `value` nulo) = **"oculto"**. `0` é valor.
- Os seis indicadores (spec §5.2), nesta ordem de coluna: `triages_started`, `triages_completed`, `attendances_closed`, `wait_within_30_pct`, `no_show_pct`, `left_pct`. Os três `_pct` são taxas: **1 casa** decimal (`12,5%`, `0,0%`). Contagens são inteiras (`Float` no fio; `128.0` aparece `128`).
- Janela: as **12 semanas fechadas** que terminam no domingo anterior à semana corrente, no **dia de `America/Sao_Paulo`**, o mesmo padrão do `GET /city_analytics` do console. `from` = segunda-feira 12 semanas antes da segunda-feira corrente; `to` = domingo passado.
- Datas `YYYY-MM-DD` são formatadas **por texto**, nunca com `new Date(...)`.
- Nenhuma das consultas novas entra no `CityHeader`. A aba só consulta quando é aberta, como as outras (`staleTime: Infinity`, botão "atualizar").
- **Ordem de deploy: api → maintenance** (spec §12). O README ganha o aviso, como o da aba Protocolos.
- Nunca escreva à mão em arquivo gerado (`schema.graphql`, `src/gql/*`): eles saem do api em execução e do codegen.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. O `.gitignore` do maintenance ignora `node_modules` (sem barra, então também o symlink), mas **não** ignora `.claude/`: o checkout principal vai listar `.claude/` como não rastreado. É esperado; não adicione.

  ```bash
  cd apps/maintenance
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod14 -b feat/mod-14-analytics origin/main
  ln -s ../../node_modules .claude/mod14/node_modules
  ```

- Todos os caminhos de arquivo abaixo são relativos a `apps/maintenance/.claude/mod14`.
- Testes (script `test` = `vitest run`; o `vitest.config.ts` exclui `e2e/**`):

  ```bash
  cd apps/maintenance/.claude/mod14 && npx vitest run <arquivos>
  ```

- Tipos (script `typecheck` = `tsc --noEmit`) e codegen:

  ```bash
  cd apps/maintenance/.claude/mod14 && npx tsc --noEmit
  cd apps/maintenance/.claude/mod14 && npm run codegen
  ```

- Base do ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `afterEach(cleanup)`), sem jest-dom (use `not.toBeNull()`, `toBeNull()`, `.textContent`). O `vitest.config.ts` **não** fixa `TZ`: toda formatação de data do plano passa `timeZone: "America/Sao_Paulo"` explícito.
- Teste que depende de "hoje" fixa só o `Date`: `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` no `beforeEach`, `vi.useRealTimers()` no `afterEach`. Com só o `Date` falso, `findBy*` e `userEvent` continuam funcionando.
- Login de dev (Task 4): `http://maintenance.localhost:5177`, nunca `localhost:5177` (o 403 de Origin vira a frase genérica). A conta é o `Maintainer` `dev@local` semeado pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **Maintenance novo contra api antigo.** A validação recusa o documento que pede um campo inexistente, com `errors` e sem `data`. A aba Analytics explica ("o api do módulo 14 precisa subir antes do maintenance"); o topo da ficha e as outras abas seguem funcionando, e o `CityHeader` nunca pede campo de Analytics. Testes: Task 2, "api antiga sem os campos: a aba explica, sem derrubar"; Task 3, "api antiga: só a aba Analytics falha".
2. **Banco da cidade inalcançável.** `analyticsStatus` falha com `CITY_UNREACHABLE`, e o erro não nulo anula `city`. O quadro do pipeline mostra o código e a mensagem, e os indicadores (plataforma) continuam na tela. Teste: Task 2, "cidade inalcançável: o estado falha sozinho e os indicadores continuam".
3. **Domingo 23h30 em São Paulo (segunda em UTC).** A janela ainda é a da semana da cidade: `to` = o domingo anterior, e não o de ontem em UTC. Testes: Task 1, "domingo 23h30 em São Paulo ainda é a semana da cidade"; Task 2, "pede as 12 semanas encerradas no domingo passado, no dia da cidade".
4. **Pipeline que nunca rodou** (cidade recém-provisionada, primeiro deploy antes do rebuild). Tudo chega nulo e `stale: true`. A tela diz "nunca rodou" e "nunca publicado", com o atraso em destaque, sem imprimir `null` nem `undefined`. Teste: Task 2, "nunca rodou: sem null nem undefined na tela".
5. **Taxa zero, oculto e semana sem linha.** `0` aparece `0,0%` (ou `0`); `suppressed` aparece "oculto"; semana × indicador sem linha aparece "sem dado". Nenhum dos três vira outro. Testes: Task 1, "zero é valor"; Task 2, "células: valor, oculto, sem dado e 0,0%".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/analytics.ts`, `src/lib/analytics.test.ts` | janela das 12 semanas, grade semana × indicador, texto da célula, rótulos, datas, estado da execução | 1 |
| `codegen.ts` | escalar `ISO8601Date` → `string` | 2 |
| `schema.graphql`, `src/gql/*` (gerados) | tipos novos do api | 2 |
| `src/screens/AnalyticsTab.tsx`, `src/screens/AnalyticsTab.test.tsx` | as duas consultas e o bloco (pipeline + indicadores) | 2 |
| `src/screens/CityDetail.tsx`, `src/screens/CityDetail.test.tsx` | aba "Analytics", só consultada quando aberta | 3 |
| `README.md` | aba nova e ordem de deploy | 3 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 4 |

---

### Task 1: Regras puras do bloco Analytics

**Files:**
- Create: `src/lib/analytics.ts`
- Test: `src/lib/analytics.test.ts`

**Interfaces:**
- Consumes: `Tone` de `src/theme/tokens.ts`.
- Produces (em `src/lib/analytics.ts`):
  ```ts
  export const INDICATORS: readonly ["triages_started", "triages_completed", "attendances_closed",
                                     "wait_within_30_pct", "no_show_pct", "left_pct"];
  export const INDICATOR_LABELS: Record<string, string>;
  export const WEEKS = 12;
  export function cityToday(now: Date): string;                                   // "YYYY-MM-DD" em America/Sao_Paulo
  export type AnalyticsWindow = { from: string; to: string; weeks: string[] };     // weeks = segundas, ascendente
  export function analyticsWindow(now: Date): AnalyticsWindow;
  export type IndicatorRow = { weekStart: string; indicator: string; value?: number | null; suppressed: boolean };
  export type IndicatorGrid = Record<string, Record<string, IndicatorRow>>;       // [weekStart][indicator]
  export function indicatorGrid(rows: IndicatorRow[]): IndicatorGrid;
  export function cellText(indicator: string, row: IndicatorRow | undefined): string;
  export function fmtWeek(weekStart: string): string;                             // "2026-07-06" → "06/07/2026"
  export function fmtWhen(iso: string | null | undefined): string;                // "28/09/2026, 02:02" ou "—"
  export function runStatusView(status: string | null | undefined): { label: string; tone: Tone };
  ```

- [ ] **Step 1: Escreva o teste que falha**

Crie `src/lib/analytics.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  analyticsWindow, cellText, cityToday, fmtWeek, fmtWhen, indicatorGrid, INDICATOR_LABELS, INDICATORS,
  runStatusView, type IndicatorRow
} from "./analytics";

function row(weekStart: string, indicator: string, value: number | null, suppressed = false): IndicatorRow {
  return { weekStart, indicator, value, suppressed };
}

describe("cityToday", () => {
  it("é o dia de São Paulo, não o de UTC", () => {
    expect(cityToday(new Date("2026-10-05T02:30:00Z"))).toBe("2026-10-04");
    expect(cityToday(new Date("2026-10-05T12:00:00Z"))).toBe("2026-10-05");
  });
});

describe("analyticsWindow", () => {
  it("12 semanas fechadas, terminando no domingo passado", () => {
    const w = analyticsWindow(new Date("2026-09-30T15:00:00Z"));
    expect(w.from).toBe("2026-07-06");
    expect(w.to).toBe("2026-09-27");
    expect(w.weeks).toHaveLength(12);
    expect(w.weeks[0]).toBe("2026-07-06");
    expect(w.weeks[11]).toBe("2026-09-21");
  });

  it("domingo 23h30 em São Paulo ainda é a semana da cidade", () => {
    const w = analyticsWindow(new Date("2026-10-05T02:30:00Z"));
    expect(w.from).toBe("2026-07-06");
    expect(w.to).toBe("2026-09-27");
  });

  it("segunda de manhã em São Paulo fecha a semana anterior", () => {
    const w = analyticsWindow(new Date("2026-10-05T12:00:00Z"));
    expect(w.from).toBe("2026-07-13");
    expect(w.to).toBe("2026-10-04");
  });

  it("atravessa a virada de ano", () => {
    const w = analyticsWindow(new Date("2027-01-06T12:00:00Z"));
    expect(w.from).toBe("2026-10-12");
    expect(w.to).toBe("2027-01-03");
    expect(w.weeks[11]).toBe("2026-12-28");
  });
});

describe("indicatorGrid", () => {
  it("indexa por semana e indicador", () => {
    const g = indicatorGrid([ row("2026-09-21", "triages_started", 128), row("2026-09-14", "no_show_pct", 7) ]);
    expect(g["2026-09-21"].triages_started.value).toBe(128);
    expect(g["2026-09-14"].no_show_pct.value).toBe(7);
    expect(g["2026-09-14"].triages_started).toBeUndefined();
  });
});

describe("cellText", () => {
  it("contagem inteira com separador de milhar", () => {
    expect(cellText("triages_started", row("w", "triages_started", 1234))).toBe("1.234");
    expect(cellText("triages_started", row("w", "triages_started", 128.0))).toBe("128");
  });

  it("taxa com exatamente 1 casa", () => {
    expect(cellText("no_show_pct", row("w", "no_show_pct", 12.5))).toBe("12,5%");
    expect(cellText("no_show_pct", row("w", "no_show_pct", 12))).toBe("12,0%");
  });

  it("zero é valor", () => {
    expect(cellText("triages_started", row("w", "triages_started", 0))).toBe("0");
    expect(cellText("left_pct", row("w", "left_pct", 0))).toBe("0,0%");
  });

  it("suprimido é oculto", () => {
    expect(cellText("triages_started", row("w", "triages_started", null, true))).toBe("oculto");
  });

  it("sem linha, ou valor nulo sem supressão, é sem dado", () => {
    expect(cellText("triages_started", undefined)).toBe("sem dado");
    expect(cellText("triages_started", row("w", "triages_started", null))).toBe("sem dado");
  });
});

describe("rótulos", () => {
  it("tem rótulo para os seis indicadores", () => {
    expect(INDICATORS.map((i) => INDICATOR_LABELS[i])).toEqual([
      "Triagens iniciadas", "Triagens concluídas", "Atendimentos encerrados",
      "Espera até 30 min", "Faltas", "Saiu sem atendimento"
    ]);
  });
});

describe("datas", () => {
  it("semana formatada por texto, sem fuso", () => {
    expect(fmtWeek("2026-07-06")).toBe("06/07/2026");
  });

  it("instante em São Paulo; nulo vira travessão", () => {
    expect(fmtWhen("2026-09-28T05:02:11Z")).toBe("28/09/2026, 02:02");
    expect(fmtWhen(null)).toBe("—");
    expect(fmtWhen(undefined)).toBe("—");
  });
});

describe("runStatusView", () => {
  it("traduz o estado da última execução", () => {
    expect(runStatusView("succeeded")).toEqual({ label: "concluída", tone: "ok" });
    expect(runStatusView("running")).toEqual({ label: "em execução", tone: "info" });
    expect(runStatusView("failed")).toEqual({ label: "falhou", tone: "down" });
  });

  it("nulo é nunca rodou; estado desconhecido aparece cru, neutro", () => {
    expect(runStatusView(null)).toEqual({ label: "nunca rodou", tone: "neutral" });
    expect(runStatusView(undefined)).toEqual({ label: "nunca rodou", tone: "neutral" });
    expect(runStatusView("paused")).toEqual({ label: "paused", tone: "neutral" });
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run src/lib/analytics.test.ts`
Expected: FAIL — `Failed to resolve import "./analytics"`.

- [ ] **Step 3: Implemente**

Crie `src/lib/analytics.ts`:

```ts
import type { Tone } from "../theme/tokens";

// Regras do bloco Analytics da ficha da cidade (módulo 14, F-14.9). Puras:
// nenhum React, nenhum fetch. Formatos: contratos do módulo 14, §3.
//
// Três estados por célula, que nunca se confundem:
//   - valor    → número (0 incluído);
//   - oculto   → suppressed: true (1 a 4, suprimido na cidade; o número
//                nunca sai do banco dela — ADR 0025);
//   - sem dado → nenhuma linha para a semana × indicador.

export const INDICATORS = [
  "triages_started", "triages_completed", "attendances_closed",
  "wait_within_30_pct", "no_show_pct", "left_pct"
] as const;

export const INDICATOR_LABELS: Record<string, string> = {
  triages_started: "Triagens iniciadas",
  triages_completed: "Triagens concluídas",
  attendances_closed: "Atendimentos encerrados",
  wait_within_30_pct: "Espera até 30 min",
  no_show_pct: "Faltas",
  left_pct: "Saiu sem atendimento"
};

export const WEEKS = 12;

const TZ = "America/Sao_Paulo";

const dayParts = new Intl.DateTimeFormat("pt-BR", { timeZone: TZ, year: "numeric", month: "2-digit", day: "2-digit" });
const whenFmt = new Intl.DateTimeFormat("pt-BR", {
  timeZone: TZ, day: "2-digit", month: "2-digit", year: "numeric", hour: "2-digit", minute: "2-digit"
});
const countFmt = new Intl.NumberFormat("pt-BR", { maximumFractionDigits: 0 });
const rateFmt = new Intl.NumberFormat("pt-BR", { minimumFractionDigits: 1, maximumFractionDigits: 1 });

// O "hoje" da cidade. Às 23h30 de domingo em São Paulo já é segunda em UTC;
// a semana é a da cidade (fuso fixo, api#27).
export function cityToday(now: Date): string {
  const p = Object.fromEntries(dayParts.formatToParts(now).map((x) => [ x.type, x.value ]));
  return `${p.year}-${p.month}-${p.day}`;
}

// Aritmética em UTC sobre a data pura: meia-noite UTC não tem horário de
// verão, então somar dias nunca pula nem repete um dia.
function addDays(iso: string, n: number): string {
  const d = new Date(`${iso}T00:00:00Z`);
  d.setUTCDate(d.getUTCDate() + n);
  return d.toISOString().slice(0, 10);
}

function mondayOf(iso: string): string {
  const dow = new Date(`${iso}T00:00:00Z`).getUTCDay(); // 0 = domingo
  return addDays(iso, -((dow + 6) % 7));
}

export type AnalyticsWindow = { from: string; to: string; weeks: string[] };

// As 12 semanas fechadas que terminam no domingo anterior à semana corrente —
// o mesmo padrão do GET /city_analytics do console do operador.
export function analyticsWindow(now: Date): AnalyticsWindow {
  const thisMonday = mondayOf(cityToday(now));
  const from = addDays(thisMonday, -7 * WEEKS);
  const to = addDays(thisMonday, -1);
  const weeks = Array.from({ length: WEEKS }, (_, i) => addDays(from, 7 * i));
  return { from, to, weeks };
}

export type IndicatorRow = { weekStart: string; indicator: string; value?: number | null; suppressed: boolean };
export type IndicatorGrid = Record<string, Record<string, IndicatorRow>>;

export function indicatorGrid(rows: IndicatorRow[]): IndicatorGrid {
  const grid: IndicatorGrid = {};
  for (const r of rows) {
    (grid[r.weekStart] ??= {})[r.indicator] = r;
  }
  return grid;
}

export function cellText(indicator: string, row: IndicatorRow | undefined): string {
  if (!row) return "sem dado";
  if (row.suppressed) return "oculto";
  if (typeof row.value !== "number" || Number.isNaN(row.value)) return "sem dado";
  return indicator.endsWith("_pct") ? `${rateFmt.format(row.value)}%` : countFmt.format(row.value);
}

// "YYYY-MM-DD" → "DD/MM/YYYY" por texto: new Date("2026-07-06") é meia-noite
// UTC, que em São Paulo ainda é 05/07.
export function fmtWeek(weekStart: string): string {
  const [ y, m, d ] = weekStart.split("-");
  return `${d}/${m}/${y}`;
}

export function fmtWhen(iso: string | null | undefined): string {
  if (!iso) return "—";
  const d = new Date(iso);
  return Number.isNaN(d.getTime()) ? "—" : whenFmt.format(d);
}

const RUN_STATUS: Record<string, { label: string; tone: Tone }> = {
  running: { label: "em execução", tone: "info" },
  succeeded: { label: "concluída", tone: "ok" },
  failed: { label: "falhou", tone: "down" }
};

export function runStatusView(status: string | null | undefined): { label: string; tone: Tone } {
  if (!status) return { label: "nunca rodou", tone: "neutral" };
  return RUN_STATUS[status] ?? { label: status, tone: "neutral" };
}
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run src/lib/analytics.test.ts`
Expected: PASS.

- [ ] **Step 5: Tipos e commit**

Run: `cd apps/maintenance/.claude/mod14 && npx tsc --noEmit`
Expected: sem saída.

```bash
cd apps/maintenance/.claude/mod14
/opt/homebrew/bin/git add src/lib/analytics.ts src/lib/analytics.test.ts
/opt/homebrew/bin/git commit -m "feat: add display rules for the maintenance analytics block

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Schema, codegen e `AnalyticsTab`

**Files:**
- Modify: `codegen.ts` (escalar `ISO8601Date`)
- Create: `src/screens/AnalyticsTab.tsx`
- Test: `src/screens/AnalyticsTab.test.tsx`
- Gerados: `schema.graphql`, `src/gql/*`

**Interfaces:**
- Consumes: tudo de `src/lib/analytics.ts` (Task 1); `gql` de `src/lib/api.ts`; `graphql` de `src/gql`; `GraphQLRefusal` de `src/lib/errors.ts`; `Panel`, `Button`, `Tag`, `DataTable`, `EmptyState`, `ErrorState` de `src/components/`. Do api: `City.analyticsStatus` e `City.analyticsIndicators(from:, to:)` (contratos §3).
- Produces: `export function AnalyticsTab({ slug }: { slug: string }): JSX.Element`. Operações GraphQL com nomes fixos, que os testes da Task 3 procuram: `query CityAnalyticsStatus` e `query CityAnalyticsIndicators`.
- **Depende do plano do api** ter os dois campos no `Maintenance::Schema` na branch em execução.

- [ ] **Step 1: Extraia o schema do api do módulo 14**

O `npm run schema:pull` escreve no `schema.graphql` do **checkout principal** e roda o api no diretório padrão do container. Aqui o destino é o worktree, e o api pode estar ainda na worktree dele (`apps/api/.claude/mod14`, com o `config/master.key` copiado, como diz o plano do api). Da raiz do monorepo:

```bash
# api do módulo 14 ainda na worktree dele:
docker compose exec -T -w /rails/.claude/mod14 api bin/rails runner 'puts Maintenance::Schema.to_definition' \
  > apps/maintenance/.claude/mod14/schema.graphql
# (se o api do módulo 14 já estiver mergeado em main, tire o `-w /rails/.claude/mod14`)

grep -n "analyticsStatus\|analyticsIndicators\|scalar ISO8601Date \|type AnalyticsStatus\|type AnalyticsIndicator" \
  apps/maintenance/.claude/mod14/schema.graphql
```

Expected: as cinco linhas aparecem. Se não aparecerem, **pare e reporte**: não escreva à mão em arquivo gerado. Se aparecerem com nome ou tipo diferente do contrato §3, pare e reporte também: o contrato é a fonte, e um dos dois planos precisa mudar.

- [ ] **Step 2: Escalar novo no codegen**

Em `codegen.ts`, o `ISO8601Date` é escalar novo; sem mapeamento o codegen o tipa como `any`. Troque a linha do `config`:

```ts
      config: { enumsAsTypes: true, scalars: { ISO8601DateTime: "string", ISO8601Date: "string" } },
```

- [ ] **Step 3: Escreva o teste que falha**

Crie `src/screens/AnalyticsTab.test.tsx`:

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { AnalyticsTab } from "./AnalyticsTab";

afterEach(cleanup);

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string; variables: Record<string, unknown> } {
  return JSON.parse((call[1] as RequestInit).body as string);
}
function calls(fetchMock: ReturnType<typeof vi.fn>, operation: string) {
  return fetchMock.mock.calls.filter((call) => bodyOf(call).query.includes(`query ${operation}`));
}

type Status = {
  lastRunStatus: string | null; lastSucceededAt: string | null; lastPublishedAt: string | null;
  lastError: string | null; stale: boolean;
};
const FRESH: Status = {
  lastRunStatus: "succeeded", lastSucceededAt: "2026-10-04T05:02:11Z",
  lastPublishedAt: "2026-10-04T05:03:00Z", lastError: null, stale: false
};
const ROWS = [
  { weekStart: "2026-09-21", indicator: "triages_started", value: 128, suppressed: false },
  { weekStart: "2026-09-21", indicator: "triages_completed", value: null, suppressed: true },
  { weekStart: "2026-09-21", indicator: "no_show_pct", value: 12.5, suppressed: false },
  { weekStart: "2026-09-21", indicator: "left_pct", value: 0, suppressed: false },
  { weekStart: "2026-09-14", indicator: "triages_started", value: 0, suppressed: false }
];

function statusReply(s: Status) {
  return reply(200, { data: { city: { slug: "sp", analyticsStatus: s } } });
}
function indicatorsReply(rows: unknown[]) {
  return reply(200, { data: { city: { slug: "sp", analyticsIndicators: rows } } });
}
// Resposta da validação do graphql-ruby para campo que o schema não tem:
// `errors` sem `data` — é o que um api antigo devolve.
function undefinedFieldReply(field: string) {
  return reply(200, {
    errors: [ {
      message: `Field '${field}' doesn't exist on type 'City'`,
      extensions: { code: "undefinedField", typeName: "City", fieldName: field }
    } ]
  });
}

function renderTab() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<AnalyticsTab slug="sp" />, { wrapper });
}

describe("AnalyticsTab", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    // Domingo 04/10/2026, 23h30 em São Paulo — já segunda em UTC.
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-05T02:30:00Z"));
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => {
    vi.unstubAllGlobals();
    vi.useRealTimers();
  });

  function route(status: () => Response, indicators: () => Response) {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query CityAnalyticsStatus")) return Promise.resolve(status());
      if (body.query.includes("query CityAnalyticsIndicators")) return Promise.resolve(indicators());
      return Promise.resolve(reply(500, {}));
    });
  }

  it("mostra execução com falha, o último erro e o atraso em destaque", async () => {
    route(
      () => statusReply({
        lastRunStatus: "failed", lastSucceededAt: "2026-09-28T05:02:11Z", lastPublishedAt: "2026-09-28T05:03:00Z",
        lastError: "PG::ConnectionBad: could not connect", stale: true
      }),
      () => indicatorsReply(ROWS)
    );
    renderTab();

    expect(await screen.findByText("falhou")).not.toBeNull();
    expect(screen.getByText("dados desatualizados")).not.toBeNull();
    expect(screen.getByText("Último erro")).not.toBeNull();
    expect(screen.getByText("PG::ConnectionBad: could not connect")).not.toBeNull();
    expect(screen.getByText("28/09/2026, 02:02")).not.toBeNull();
    expect(screen.getByText("28/09/2026, 02:03")).not.toBeNull();
  });

  it("nunca rodou: sem null nem undefined na tela", async () => {
    route(
      () => statusReply({ lastRunStatus: null, lastSucceededAt: null, lastPublishedAt: null, lastError: null, stale: true }),
      () => indicatorsReply([])
    );
    renderTab();

    expect(await screen.findByText("nunca rodou")).not.toBeNull();
    expect(screen.getByText("nunca publicado")).not.toBeNull();
    expect(screen.getByText("dados desatualizados")).not.toBeNull();
    expect(await screen.findByText("nenhum indicador publicado nas últimas 12 semanas")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/null|undefined/);
  });

  it("execução recente: sem destaque de atraso nem erro", async () => {
    route(() => statusReply(FRESH), () => indicatorsReply(ROWS));
    renderTab();

    expect(await screen.findByText("concluída")).not.toBeNull();
    expect(screen.queryByText("dados desatualizados")).toBeNull();
    expect(screen.queryByText("Último erro")).toBeNull();
  });

  it("consolidou mas não publicou: o erro de publicação aparece", async () => {
    route(
      () => statusReply({ ...FRESH, lastPublishedAt: null, lastError: "Analytics::Publish: PG::Error" }),
      () => indicatorsReply(ROWS)
    );
    renderTab();

    expect(await screen.findByText("Analytics::Publish: PG::Error")).not.toBeNull();
    expect(screen.getByText("nunca publicado")).not.toBeNull();
  });

  it("pede as 12 semanas encerradas no domingo passado, no dia da cidade", async () => {
    route(() => statusReply(FRESH), () => indicatorsReply(ROWS));
    renderTab();

    await waitFor(() => expect(calls(fetchMock, "CityAnalyticsIndicators")).toHaveLength(1));
    const { variables } = bodyOf(calls(fetchMock, "CityAnalyticsIndicators")[0]);
    expect(variables).toEqual({ slug: "sp", from: "2026-07-06", to: "2026-09-27" });
    expect(await screen.findByText("Indicadores publicados — 06/07/2026 a 27/09/2026")).not.toBeNull();
  });

  it("células: valor, oculto, sem dado e 0,0%", async () => {
    route(() => statusReply(FRESH), () => indicatorsReply(ROWS));
    renderTab();

    expect(await screen.findByText("128")).not.toBeNull();
    expect(screen.getAllByText("oculto")).toHaveLength(1);
    expect(screen.getByText("12,5%")).not.toBeNull();
    expect(screen.getByText("0,0%")).not.toBeNull();
    expect(screen.getByText("0")).not.toBeNull();
    // 12 semanas × 6 indicadores = 72 células; 5 têm linha.
    expect(screen.getAllByText("sem dado")).toHaveLength(67);
    // Semana mais recente primeiro.
    const text = document.body.textContent ?? "";
    expect(text.indexOf("21/09/2026")).toBeLessThan(text.indexOf("14/09/2026"));
    expect(text).not.toContain("05/10/2026");
  });

  it("cidade inalcançável: o estado falha sozinho e os indicadores continuam", async () => {
    route(
      () => reply(200, {
        data: { city: null },
        errors: [ { message: "banco da cidade inacessível", path: [ "city", "analyticsStatus" ], extensions: { code: "CITY_UNREACHABLE" } } ]
      }),
      () => indicatorsReply(ROWS)
    );
    renderTab();

    expect((await screen.findByRole("alert")).textContent).toBe("CITY_UNREACHABLE — banco da cidade inacessível");
    expect(await screen.findByText("128")).not.toBeNull();
  });

  it("api antiga sem os campos: a aba explica, sem derrubar", async () => {
    route(() => undefinedFieldReply("analyticsStatus"), () => undefinedFieldReply("analyticsIndicators"));
    renderTab();

    await waitFor(() => expect(screen.getAllByRole("alert")).toHaveLength(2));
    for (const alert of screen.getAllByRole("alert")) {
      expect(alert.textContent).toMatch(/o api do módulo 14 precisa subir antes do maintenance/);
    }
  });

  it("atualizar busca as duas consultas de novo", async () => {
    const user = userEvent.setup();
    route(() => statusReply(FRESH), () => indicatorsReply(ROWS));
    renderTab();

    await screen.findByText("128");
    await user.click(screen.getByRole("button", { name: "atualizar" }));

    await waitFor(() => expect(calls(fetchMock, "CityAnalyticsStatus")).toHaveLength(2));
    await waitFor(() => expect(calls(fetchMock, "CityAnalyticsIndicators")).toHaveLength(2));
  });
});
```

- [ ] **Step 4: Rode e confirme que falha**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run src/screens/AnalyticsTab.test.tsx`
Expected: FAIL — `Failed to resolve import "./AnalyticsTab"`.

- [ ] **Step 5: Implemente a aba**

Crie `src/screens/AnalyticsTab.tsx`:

```tsx
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import {
  INDICATORS, INDICATOR_LABELS, analyticsWindow, cellText, fmtWeek, fmtWhen, indicatorGrid, runStatusView
} from "../lib/analytics";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";

// Aba Analytics (módulo 14, F-14.9): estado do pipeline de consolidação da
// cidade e os indicadores que ela publicou na plataforma (ADR 0025).
//
// DUAS consultas, e nenhuma delas é a do topo (CityHeader):
//   - a validação do GraphQL recusa o documento INTEIRO quando um campo não
//     existe; contra um api sem o módulo 14, só esta aba falha, e explica;
//   - analyticsStatus é não nulo e lê o banco da cidade: um CITY_UNREACHABLE
//     anula `city` inteira. Separados, os indicadores (plataforma) seguem.
const CityAnalyticsStatusQuery = graphql(`
  query CityAnalyticsStatus($slug: String!) {
    city(slug: $slug) {
      slug
      analyticsStatus { lastRunStatus lastSucceededAt lastPublishedAt lastError stale }
    }
  }
`);

const CityAnalyticsIndicatorsQuery = graphql(`
  query CityAnalyticsIndicators($slug: String!, $from: ISO8601Date!, $to: ISO8601Date!) {
    city(slug: $slug) {
      slug
      analyticsIndicators(from: $from, to: $to) { weekStart indicator value suppressed }
    }
  }
`);

const dlStyle: CSSProperties = {
  display: "grid",
  gridTemplateColumns: "max-content 1fr",
  columnGap: 12,
  rowGap: 6,
  margin: 0,
  fontSize: 12.5
};

const errorBox: CSSProperties = {
  margin: 0,
  padding: "8px 10px",
  borderRadius: 6,
  background: "var(--down-bg)",
  color: "var(--down)",
  fontSize: 12,
  whiteSpace: "pre-wrap",
  wordBreak: "break-word"
};

// Erro que derrubou a consulta inteira. `undefinedField` é a validação do
// graphql-ruby: o api ainda não tem os campos (ordem de deploy: api antes).
function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && err.code === "undefinedField") {
    return "esta API ainda não tem os campos de Analytics — o api do módulo 14 precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

// Mesmo critério do CityDetail: o refusal do campo, ou da cidade inteira
// quando o campo não nulo anulou `city`.
function fieldError(fieldErrors: GraphQLRefusal[], field: string): GraphQLRefusal | undefined {
  return fieldErrors.find((refusal) => {
    const path = refusal.path ?? [];
    return path[0] === "city" && (path.length === 1 || path[1] === field);
  });
}

export function AnalyticsTab({ slug }: { slug: string }) {
  // A janela é fixada ao abrir a aba; "atualizar" relê os mesmos dias.
  const [ range ] = useState(() => analyticsWindow(new Date()));

  const status = useQuery({
    queryKey: [ "city", slug, "analyticsStatus" ],
    queryFn: () => gql(CityAnalyticsStatusQuery, { slug }),
    staleTime: Infinity
  });
  const indicators = useQuery({
    queryKey: [ "city", slug, "analyticsIndicators", range.from, range.to ],
    queryFn: () => gql(CityAnalyticsIndicatorsQuery, { slug, from: range.from, to: range.to }),
    staleTime: Infinity
  });

  const s = status.data?.data?.city?.analyticsStatus ?? null;
  const statusErr = status.data ? fieldError(status.data.fieldErrors, "analyticsStatus") : undefined;
  const rows = indicators.data?.data?.city?.analyticsIndicators ?? null;
  const indicatorsErr = indicators.data ? fieldError(indicators.data.fieldErrors, "analyticsIndicators") : undefined;
  const grid = indicatorGrid(rows ?? []);
  const run = runStatusView(s?.lastRunStatus);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <div>
        <Button
          onClick={() => { void status.refetch(); void indicators.refetch(); }}
          busy={status.isFetching || indicators.isFetching}
        >
          atualizar
        </Button>
      </div>

      <Panel title="Pipeline de Analytics" actions={s?.stale ? <Tag tone="warn">dados desatualizados</Tag> : undefined}>
        {status.isPending && <p>carregando…</p>}
        {status.isError && <ErrorState message={requestErrorText(status.error)} />}
        {statusErr && <ErrorState message={`${statusErr.code} — ${statusErr.message}`} />}
        {s && (
          <>
            <dl style={dlStyle}>
              <dt>Última execução</dt><dd><Tag tone={run.tone}>{run.label}</Tag></dd>
              <dt>Consolidado com sucesso em</dt><dd>{fmtWhen(s.lastSucceededAt)}</dd>
              <dt>Publicado na plataforma em</dt>
              <dd>{s.lastPublishedAt ? fmtWhen(s.lastPublishedAt) : "nunca publicado"}</dd>
            </dl>
            {s.stale && (
              <p style={{ margin: 0, fontSize: 12, color: "var(--warn)" }}>
                A última consolidação bem-sucedida tem mais de 36 h, ou nunca houve. O painel da cidade avisa que os dados estão desatualizados.
              </p>
            )}
            {s.lastError && (
              <div style={{ display: "flex", flexDirection: "column", gap: 4 }}>
                <span style={{ fontSize: 12, fontWeight: 600, color: "var(--down)" }}>Último erro</span>
                <pre style={errorBox}>{s.lastError}</pre>
              </div>
            )}
          </>
        )}
      </Panel>

      <Panel title={`Indicadores publicados — ${fmtWeek(range.from)} a ${fmtWeek(range.to)}`}>
        {indicators.isPending && <p>carregando…</p>}
        {indicators.isError && <ErrorState message={requestErrorText(indicators.error)} />}
        {indicatorsErr && <ErrorState message={`${indicatorsErr.code} — ${indicatorsErr.message}`} />}
        {rows && rows.length === 0 && <EmptyState message="nenhum indicador publicado nas últimas 12 semanas" />}
        {rows && rows.length > 0 && (
          <>
            <p style={{ margin: 0, fontSize: 11.5, color: "var(--ink3)" }}>
              “oculto”: contagem de 1 a 4, suprimida na cidade. “sem dado”: a cidade não publicou o indicador naquela semana, ou a taxa não tinha denominador.
            </p>
            <DataTable<string>
              columns={[
                { key: "week", label: "Semana", render: (week) => fmtWeek(week) },
                ...INDICATORS.map((indicator) => ({
                  key: indicator,
                  label: INDICATOR_LABELS[indicator],
                  render: (week: string) => cellText(indicator, grid[week]?.[indicator])
                }))
              ]}
              rows={[ ...range.weeks ].reverse()}
              rowKey={(week) => week}
            />
          </>
        )}
      </Panel>
    </div>
  );
}
```

Observações para quem implementa:
- A legenda é **um nó de texto só**; um `<strong>oculto</strong>` faria o `getAllByText("oculto")` contar a legenda.
- A caixa do último erro é um `<pre>` sem `role="alert"`: é dado do pipeline, não falha da requisição. `role="alert"` fica só no `ErrorState`, e os testes contam alertas.

- [ ] **Step 6: Gere os tipos**

Run: `cd apps/maintenance/.claude/mod14 && npm run codegen`
Expected: `src/gql/gql.ts` e `src/gql/graphql.ts` mudam; `grep -n "CityAnalyticsStatus\|CityAnalyticsIndicators\|ISO8601Date: { input: string; output: string; }" src/gql/graphql.ts` acha as três coisas. Se o codegen recusar um documento, o nome de campo diverge do schema extraído no Step 1: pare e reporte.

- [ ] **Step 7: Rode e confirme que passa**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run src/screens/AnalyticsTab.test.tsx && npx tsc --noEmit`
Expected: PASS (9 testes); typecheck sem saída.

- [ ] **Step 8: Commit**

```bash
cd apps/maintenance/.claude/mod14
/opt/homebrew/bin/git add codegen.ts schema.graphql src/gql src/screens/AnalyticsTab.tsx src/screens/AnalyticsTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the analytics tab with pipeline status and published indicators

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: A aba no detalhe da cidade e o README

**Files:**
- Modify: `src/screens/CityDetail.tsx` (comentário do topo, `TabKey`, `TABS`, import, renderização da aba)
- Modify: `README.md` (tabela de telas e seção nova)
- Test: `src/screens/CityDetail.test.tsx`

**Interfaces:**
- Consumes: `AnalyticsTab` (Task 2).
- Produces: aba "Analytics" no detalhe da cidade, com a chave `"analytics"` em `TabKey`.

- [ ] **Step 1: Escreva o teste que falha**

Em `src/screens/CityDetail.test.tsx`:

(a) No primeiro teste ("carrega só o topo…"), acrescente as duas operações novas às **duas** listas de operações que não podem ter saído. A primeira:

```ts
    for (const op of [ "CityProfile", "CityProtocolVersions", "CityRecipients", "CityAccounts", "CityCounts", "CityOperations", "CityAnalyticsStatus", "CityAnalyticsIndicators" ]) {
```

e a segunda (depois do clique em "Contas"):

```ts
    for (const op of [ "CityProfile", "CityProtocolVersions", "CityRecipients", "CityCounts", "CityOperations", "CityAnalyticsStatus", "CityAnalyticsIndicators" ]) {
```

(b) Acrescente, dentro do `describe("CityDetail")`, depois do último teste:

```tsx
  it("api antiga: só a aba Analytics falha; o topo e as outras abas seguem", async () => {
    const user = userEvent.setup();
    const undefinedField = (field: string) => reply(200, {
      errors: [ { message: `Field '${field}' doesn't exist on type 'City'`, extensions: { code: "undefinedField" } } ]
    });
    fetchMock.mockImplementation((_url, init) => {
      const body = bodyOf([ _url, init ]);
      if (body.query.includes("query CityHeader")) return Promise.resolve(HEADER_REPLY.clone());
      if (body.query.includes("query CityAnalyticsStatus")) return Promise.resolve(undefinedField("analyticsStatus"));
      if (body.query.includes("query CityAnalyticsIndicators")) return Promise.resolve(undefinedField("analyticsIndicators"));
      if (body.query.includes("query CityProfile")) {
        return Promise.resolve(
          reply(200, { data: { city: { slug: "sp", consentTermVersion: "v3", profile: { name: "São Paulo", uf: "SP", ibgeCode: "3550308" } } } })
        );
      }
      return Promise.resolve(reply(200, { data: { city: { slug: "sp" } } }));
    });

    renderDetail();
    await screen.findByText("São Paulo");

    // O topo nunca pede campo de Analytics.
    expect(bodyOf(operationCalls(fetchMock, "CityHeader")[0]).query).not.toMatch(/analytics/i);

    await user.click(screen.getByRole("button", { name: "Analytics" }));
    await waitFor(() => expect(screen.getAllByRole("alert")).toHaveLength(2));
    expect(screen.getAllByRole("alert")[0].textContent).toMatch(/o api do módulo 14 precisa subir antes do maintenance/);
    expect(operationCalls(fetchMock, "CityAnalyticsStatus")).toHaveLength(1);
    expect(operationCalls(fetchMock, "CityAnalyticsIndicators")).toHaveLength(1);

    // O topo continua, e a consulta dele não se repetiu.
    expect(screen.getByText("+55 11 90000-0000")).not.toBeNull();
    expect(operationCalls(fetchMock, "CityHeader")).toHaveLength(1);

    await user.click(screen.getByRole("button", { name: "Perfil" }));
    expect(await screen.findByText("3550308")).not.toBeNull();
    expect(screen.queryByRole("alert")).toBeNull();
  });
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run src/screens/CityDetail.test.tsx`
Expected: FAIL no teste novo — `Unable to find an accessible element with the role "button" and name "Analytics"`. Os testes antigos continuam passando (as operações novas ainda não existem, então a lista estendida não quebra nada).

- [ ] **Step 3: Implemente**

Em `src/screens/CityDetail.tsx`:

- import, junto do `ProtocolsTab`:

  ```ts
  import { AnalyticsTab } from "./AnalyticsTab";
  ```

- no comentário do topo, troque a última frase por:

  ```ts
  // exceção: ela consulta pelo próprio componente (`ProtocolsTab`), e também
  // só quando aberta, porque o componente só monta com a aba ativa. A aba
  // Analytics (módulo 14) segue o mesmo molde (`AnalyticsTab`), com consultas
  // PRÓPRIAS: contra um api sem os campos do módulo 14, a validação do
  // GraphQL recusa o documento inteiro, e só aquela aba pode cair — nunca o
  // topo. Por isso nenhum campo de Analytics entra no CityHeader.
  ```

- `TabKey` e `TABS`:

  ```ts
  type TabKey = "profile" | "protocols" | "alertRecipients" | "accounts" | "counts" | "operations" | "analytics";

  const TABS: { key: TabKey; label: string }[] = [
    { key: "profile", label: "Perfil" },
    { key: "protocols", label: "Protocolos" },
    { key: "alertRecipients", label: "Destinatários" },
    { key: "accounts", label: "Contas" },
    { key: "counts", label: "Contagens" },
    { key: "operations", label: "Operação" },
    { key: "analytics", label: "Analytics" }
  ];
  ```

- a aba, logo depois de `{activeTab === "protocols" && <ProtocolsTab slug={slug} />}`:

  ```tsx
            {activeTab === "analytics" && <AnalyticsTab slug={slug} />}
  ```

Em `README.md`:

- na tabela de telas, troque `as abas Perfil, Protocolos, Destinatários, Contas, Contagens e Operação` por `as abas Perfil, Protocolos, Destinatários, Contas, Contagens, Operação e Analytics`;
- depois do bloco de aviso de deploy da seção "Protocolos" (antes de `## Subir em dev`), acrescente:

  ```markdown
  ### Analytics

  A aba **Analytics** do detalhe de cidade (módulo 14, ADR 0025) mostra o
  estado do pipeline de consolidação da cidade (`analyticsStatus`: última
  execução, quando consolidou e publicou com sucesso, último erro) e os seis
  indicadores semanais que ela publicou na plataforma nas últimas 12 semanas
  fechadas (`analyticsIndicators`). "Dados desatualizados" aparece quando a
  última consolidação bem-sucedida tem mais de 36 h, ou nunca houve. Contagem
  de 1 a 4 aparece como "oculto"; semana sem publicação, "sem dado".

  A aba faz duas consultas próprias, nenhuma junto do topo da ficha: o estado
  lê o banco da cidade, e os indicadores, a plataforma. Uma cidade
  inalcançável derruba só o quadro do estado.

  > **Ordem de deploy:** o `api` sobe **antes** do maintenance. Contra um api
  > sem os campos do módulo 14, a validação recusa as consultas da aba
  > Analytics, e só ela mostra o aviso; o topo e as outras abas seguem.
  ```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run src/screens/CityDetail.test.tsx && npx tsc --noEmit`
Expected: PASS (todos, inclusive o novo); typecheck sem saída.

- [ ] **Step 5: Commit**

```bash
cd apps/maintenance/.claude/mod14
/opt/homebrew/bin/git add src/screens/CityDetail.tsx src/screens/CityDetail.test.tsx README.md
/opt/homebrew/bin/git commit -m "feat: add the analytics tab to the maintenance city detail

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Suíte, codegen:check, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd apps/maintenance/.claude/mod14 && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde. A CI (`.github/workflows/ci.yml`) roda os quatro. O número de arquivos de teste não dobra (`noEmit` no `tsconfig`).

```bash
cd apps/maintenance/.claude/mod14 && /opt/homebrew/bin/git status --short
```

Expected: vazio. O symlink `node_modules` não aparece porque é ignorado. Se aparecer, **não** o adicione.

- [ ] **Step 2: O topo não pede Analytics**

```bash
cd apps/maintenance/.claude/mod14 && grep -n "analytics" src/screens/CityDetail.tsx
```

Expected: só o import de `AnalyticsTab`, o comentário, a chave `"analytics"` em `TabKey`/`TABS` e a linha que monta a aba. Nenhuma ocorrência dentro de `CityHeaderQuery`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§6.4, §9, §10.3, §12), o ADR 0025 e os contratos §3. Pontos de atenção:
- nenhum campo de Analytics dentro do `CityHeader` nem de outra consulta existente;
- `analyticsStatus` e `analyticsIndicators` em consultas **separadas**;
- "oculto" × "sem dado" × `0` nunca se confundem; taxas com 1 casa;
- a janela sai do dia de São Paulo, e nenhuma data `YYYY-MM-DD` passa por `new Date` para ser exibida;
- `schema.graphql` e `src/gql/*` iguais ao que o api em execução produz (`codegen:check` verde);
- nenhum `git add -A` no histórico (`git log --stat origin/main..HEAD` sem `node_modules`).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do módulo 14 rodando com a semente e o `city:analytics:rebuild` (spec §11), na porta que o plano do api definir (no módulo 11 foi `:3031`). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`; por isso o Vite do worktree sobe **na 5177** (pare antes o serviço `maintenance` do compose, ou use outra porta e `MAINTENANCE_FRONTEND_ORIGIN` igual ao host aberto e aceito pelo api):

```bash
docker compose stop maintenance   # na raiz do monorepo
cd apps/maintenance/.claude/mod14 && VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5177 --host 0.0.0.0
```

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. Confira com screenshot:
- Cidades → Curitiba → aba **Analytics**;
- o quadro do pipeline com "concluída", as duas datas e, sem erro, nada em vermelho;
- os indicadores das 12 semanas, a mais recente em cima, com ao menos um "oculto" e a legenda;
- as outras abas continuam abrindo.

Depois: `docker compose start maintenance`.

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é api → dashboard → admin → maintenance (spec §12). Na produção, publicar o maintenance novo **antes** do api novo não derruba a ficha, só a aba Analytics, e ela diz o motivo.

---

## Desvios

- **"Bloco" virou aba.** A spec (§9) diz "bloco 'Analytics' na ficha da cidade". O detalhe da cidade já tem uma regra escrita no código (`CityDetail.tsx`, comentário do topo; spec do maintenance §6 e ruling P5): abrir a ficha só consulta a plataforma, e cada leitura do banco da cidade só sai quando a aba dela é aberta. `analyticsStatus` lê `analytics_runs` no banco da cidade, então um quadro sempre visível abriria uma conexão com a cidade a cada ficha aberta. O custo da escolha: o atraso só fica "em destaque" depois de abrir a aba, e não na ficha. Ver as perguntas em aberto.
- **Nenhum desvio de formato.** Os nomes e tipos da Task 2 são os do contrato §3. O Step 1 da Task 2 para a execução se o schema extraído divergir.

## Perguntas em aberto

1. **Atraso visível sem abrir a aba?** Se o usuário quiser o "dados desatualizados" já no topo da ficha, o caminho mais barato é o api expor um campo **de plataforma** (por exemplo `City.analyticsLastPublishedAt`, lido de `city_analytics_indicators`, sem abrir a cidade) e o topo ganhar uma consulta própria, ainda separada do `CityHeader` pela mesma razão de deploy. Isso muda o contrato §3 e o plano do api; fica fora deste plano.
2. **`analyticsStatus` não nulo.** Os campos do `CityType` que abrem o banco da cidade (`counts`, `operations`, `profile`) são `null: true` justamente para que um `CITY_UNREACHABLE` anule só o campo. O contrato §3 fixa `AnalyticsStatus!`, que anula `city` inteira. Este plano absorve isso com consultas separadas, mas o plano do api pode preferir `AnalyticsStatus` anulável por consistência com o resto do tipo. Se mudar, a Task 2 continua valendo sem alteração (o `fieldError` já trata os dois caminhos).

## Self-review

- **Cobertura da spec:** §6.4 `analyticsIndicators(from:, to:)` e `analyticsStatus` → Task 2; §9 "atraso em destaque" → Task 2 (tag "dados desatualizados" + texto) e o pedido do caller "stale/failed destacados, lastError mostrado" → Task 2; "indicadores das últimas 12 semanas" → Tasks 1 e 2; §10.3 "no maintenance o atraso" → Task 2; §12 ordem de deploy → README (Task 3) e consultas separadas (Tasks 2 e 3); nota do caller sobre api antiga → Review Focus 1, Tasks 2 e 3.
- **Placeholders:** nenhum; todo passo de código tem o código.
- **Tipos:** `IndicatorRow` (Task 1) aceita o tipo gerado (`value?: number | null`); `runStatusView` recebe `string | null | undefined`, que é o `lastRunStatus` gerado; `AnalyticsTab({ slug })` é o que a Task 3 monta; os nomes de operação `CityAnalyticsStatus`/`CityAnalyticsIndicators` são os mesmos nos testes das Tasks 2 e 3.
- **Review Focus:** as cinco linhas têm teste citado pelo nome nas Tasks 1, 2 e 3.
- **Código conferido:** o código das Tasks 1–3 foi aplicado numa cópia descartável de `origin/main` do maintenance (`ff7532c`) em 2026-09-30, com o `schema.graphql` editado **só nessa cópia** conforme o contrato §3 (o api ainda não existe): codegen, `tsc --noEmit`, vitest 19 arquivos / 164 testes, `codegen:check` e `npm run build`, tudo verde. Na execução real, o schema vem do api (Task 2, Step 1).
