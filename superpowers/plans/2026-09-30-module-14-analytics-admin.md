# Módulo 14 — Analytics (admin) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tela "Analytics das cidades" no console do operador (F-14.8): uma tabela cidade × indicador da semana escolhida, com a tendência das últimas 12 semanas em cada célula, "oculto" e "sem dado" visivelmente distintos, percentuais com 1 casa, a data da última publicação de cada cidade, seletor de semana, estados de erro e de vazio e item de navegação.

**Architecture:** O cliente de `GET /city_analytics` entra em `src/lib/api.ts`, ao lado de `/cities` e `/unknown_channels`, com os tipos copiados do arquivo de contratos. As regras puras (rótulo do indicador, "oculto" × "sem dado" × valor, formato de taxa, pontos da tendência, data da semana sem fuso) ficam em `src/lib/cityAnalytics.ts`, testáveis sem React. A célula (`IndicatorCell`) e a tela (`CityAnalytics`) moram em `src/modules/analytics/`. A tendência reaproveita o `Sparkline` que já existe (Recharts, já é dependência), que passa a aceitar `null` como lacuna. Nenhuma dependência nova.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Recharts 2 (já no `package.json`), Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem user-event, sem msw: `vi.mock("../../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-09-30-module-14-analytics-design.md` (§6.2, §9 e §10.3 são deste plano; §5.2 dá o cálculo dos indicadores) e `docs/adr/0025.md`. O formato de `GET /city_analytics` está fixado em `docs/superpowers/plans/2026-09-30-module-14-analytics-contracts.md` §2, o mesmo arquivo que o plano do api usa; os tipos da Task 1 o copiam literalmente. O plano do api (escrito em paralelo) precisa estar **mergeado antes** do merge deste.

## Global Constraints

- Rota: `GET /city_analytics`, host do console (`PlatformConsoleHost`), sessão de operador. Sem `from`/`to`, o servidor devolve as **12 semanas que terminam na semana anterior à atual**. Esta tela **não manda** `from`/`to`: o seletor escolhe entre as semanas que vieram.
- Resposta (contratos §2, literal):
  - `{ data: { weeks: string[], indicators: string[], cities: [{ id, slug, name, uf, last_published_at, values: { [indicator]: IndicatorValue[] } }] } }`;
  - `IndicatorValue` = `number` (contagem inteira ou percentual com 1 casa) | `{ "suppressed": true }` | `null` (sem linha = sem dado);
  - cada `values[indicator]` é alinhado com `weeks`; cidades ordenadas por nome, todas as ativas.
- Os seis indicadores (spec §5.2): `triages_started`, `triages_completed`, `attendances_closed`, `wait_within_30_pct`, `no_show_pct`, `left_pct`. Os três `_pct` são taxas.
- Termos da interface: **"oculto"** para `{ suppressed: true }` (termo do módulo 11) e **"sem dado"** para `null`/ausente. Os dois nunca se confundem e nenhum dos dois vira `0`.
- Percentual sempre com **1 casa**, vírgula decimal: `12,5%`, `12,0%`, `0,0%`. Contagem inteira com separador de milhar: `1.234`.
- A data da semana (`YYYY-MM-DD`) é formatada **por texto**, nunca por `new Date(...)`: `"2026-07-06"` lido como UTC vira 05/07 em São Paulo.
- O console nunca vê bairro, unidade, protocolo ou pergunta (ADR 0025, D3). Esta tela não pede nem mostra nada além do contrato.
- Nenhuma dependência nova. O Recharts já existe; o `ResponsiveContainer` dele **não roda no jsdom** (não há `ResizeObserver`), então todo teste que renderiza célula ou tela troca o `Sparkline` por um dublê via `vi.mock`.
- O admin não tem `user-event`: interação nos testes é `fireEvent` do `@testing-library/react`.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`, e o checkout principal de `apps/admin` tem mudanças de outra pessoa (`.env.example`, `design_handoff_admin_console 4/`) que não são deste plano. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. O `.gitignore` do admin ignora `node_modules` (sem barra, então também o symlink), mas **não** ignora `.claude/`: o checkout principal vai listar `.claude/` como não rastreado. É esperado; não adicione.

  ```bash
  cd apps/admin
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod14 -b feat/mod-14-analytics origin/main
  ln -s ../../node_modules .claude/mod14/node_modules
  ```

- Todos os caminhos de arquivo abaixo são relativos a `apps/admin/.claude/mod14`.
- Testes (script `test` = `vitest run`):

  ```bash
  cd apps/admin/.claude/mod14 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit (script `typecheck` = `tsc --noEmit`; o `tsconfig` tem `noEmit: true`, então `tsc -b` do build não espalha `.js` em `src/`):

  ```bash
  cd apps/admin/.claude/mod14 && npx tsc --noEmit
  ```

- Base do ambiente de teste: `vitest.config.ts` com `environment: "jsdom"` e `globals: false` (todo teste de componente chama `afterEach(cleanup)`); sem `setupFiles`, sem jest-dom (use `toBeTruthy()`, `toBeNull()`, `.textContent`). O `vitest.config.ts` do admin **não** fixa `TZ`: o `fmtDateTime` de `src/lib/format.ts` passa `timeZone: "America/Sao_Paulo"` explícito, e é por isso que ele pode ser usado nos testes.

## Review Focus

1. **Semana sem publicação no meio de semanas com dado.** A célula diz "sem dado", nunca `0`, e a tendência tem uma **lacuna** naquele ponto, não uma queda a zero (um `0` na série desenharia uma falsa queda de demanda). Testes: Task 2, "suprimido e sem dado viram lacuna na tendência"; Task 4, "tendência leva null nas semanas ocultas e sem dado".
2. **Taxa zero × sem dado.** `0` é um valor legítimo (nenhuma falta na semana) e aparece `0,0%`; só `null` é "sem dado". Um `if (!v)` apagaria o zero. Testes: Task 2, "zero é valor, não sem dado"; Task 4, "0,0% aparece".
3. **Data da semana deslocada pelo fuso.** `"2026-07-06"` aparece como `06/07/2026` no seletor, sem voltar para 05/07 por leitura em UTC. Teste: Task 2, "semana formatada por texto, sem fuso".
4. **Cidade que nunca publicou** (acabou de ser provisionada, ou o job ainda não rodou). A linha aparece com "nunca publicou" e "sem dado" em todas as células, e `values` sem a chave do indicador (ou com array mais curto que `weeks`) não quebra a tela. Testes: Task 2, "array curto ou ausente"; Task 4, "cidade sem publicação".
5. **Nenhuma cidade publicou ainda** (primeiro deploy, antes do `city:analytics:rebuild:all`). A tela explica o motivo em vez de mostrar uma grade inteira de "sem dado". Teste: Task 4, "ninguém publicou: estado vazio explicado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `vite.config.ts`, `README.md` | proxy de `/city_analytics` e documentação | 1 |
| `src/lib/api.ts` | tipos do contrato e `listCityAnalytics()` | 1 |
| `src/lib/cityAnalyticsApi.test.ts` | teste do cliente | 1 |
| `src/lib/cityAnalytics.ts`, `src/lib/cityAnalytics.test.ts` | rótulos, `cellView`, `trendPoints`, `fmtWeek`, `hasAnyPublication` | 2 |
| `src/components/Sparkline.tsx` | aceitar `null` como lacuna | 2 |
| `src/modules/analytics/IndicatorCell.tsx`, `IndicatorCell.test.tsx` | valor da semana + tendência de 12 semanas | 3 |
| `src/modules/analytics/CityAnalytics.tsx`, `CityAnalytics.test.tsx` | tela: seletor, tabela, legenda, estados | 4 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | item de navegação e roteamento | 5 |
| — | suíte, build, revisão e prova no navegador | 6 |

---

### Task 1: Cliente de `/city_analytics`, tipos do contrato e proxy

**Files:**
- Modify: `src/lib/api.ts` (fim do arquivo, depois de `listUnknownChannels`)
- Modify: `vite.config.ts` (comentário do topo e `server.proxy`)
- Modify: `README.md` (parágrafo do proxy, lista do que o console faz, árvore `lib/api.ts`)
- Test: `src/lib/cityAnalyticsApi.test.ts`

**Interfaces:**
- Produces (em `src/lib/api.ts`):
  ```ts
  export type IndicatorValue = number | { suppressed: true } | null;
  export interface CityAnalyticsCity {
    id: string; slug: string; name: string; uf: string | null;
    last_published_at: string | null;
    values: Record<string, IndicatorValue[]>;
  }
  export interface CityAnalyticsData { weeks: string[]; indicators: string[]; cities: CityAnalyticsCity[] }
  export async function listCityAnalytics(): Promise<CityAnalyticsData>;
  ```

- [ ] **Step 1: Escreva o teste que falha**

Crie `src/lib/cityAnalyticsApi.test.ts`:

```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { listCityAnalytics, ApiError, type CityAnalyticsData } from "./api";

const DATA: CityAnalyticsData = {
  weeks: [ "2026-09-14", "2026-09-21" ],
  indicators: [ "triages_started", "no_show_pct" ],
  cities: [ {
    id: "c1", slug: "curitiba", name: "Curitiba", uf: "PR",
    last_published_at: "2026-09-30T05:02:12Z",
    values: { triages_started: [ 128, { suppressed: true } ], no_show_pct: [ null, 12.5 ] }
  } ]
};

describe("listCityAnalytics", () => {
  afterEach(() => vi.unstubAllGlobals());

  it("GETs /city_analytics without params, with the session cookie, and unwraps data", async () => {
    const fetchMock = vi.fn().mockResolvedValue(
      new Response(JSON.stringify({ data: DATA }), { status: 200 })
    );
    vi.stubGlobal("fetch", fetchMock);

    await expect(listCityAnalytics()).resolves.toEqual(DATA);
    const [ url, init ] = fetchMock.mock.calls[0];
    expect(url).toBe("/city_analytics");
    expect(init.credentials).toBe("include");
    expect(init.method ?? "GET").toBe("GET");
  });

  it("raises ApiError with the server code on 422", async () => {
    vi.stubGlobal("fetch", vi.fn().mockResolvedValue(
      new Response(JSON.stringify({ error: "invalid_range" }), { status: 422 })
    ));

    const err = await listCityAnalytics().catch((e) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).status).toBe(422);
    expect((err as ApiError).body).toEqual({ error: "invalid_range" });
  });

  it("raises ApiError 401 when the session is missing", async () => {
    vi.stubGlobal("fetch", vi.fn().mockResolvedValue(new Response("", { status: 401 })));

    const err = await listCityAnalytics().catch((e) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).status).toBe(401);
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/lib/cityAnalyticsApi.test.ts`
Expected: FAIL — `listCityAnalytics` não é exportado por `./api`.

- [ ] **Step 3: Implemente o cliente**

No fim de `src/lib/api.ts`:

```ts
// GET /city_analytics (módulo 14, F-14.8) — indicadores semanais que cada
// cidade publicou na plataforma (PlatformConsoleHost, operador). O api lê só
// city_analytics_indicators e cities: nunca abre banco de cidade (ADR 0025).
// Sem from/to, o servidor devolve as 12 semanas que terminam na semana
// anterior à atual — esta tela não manda período, o seletor escolhe entre
// as semanas que vieram.
//
// values[indicator] é alinhado com weeks: número (contagem inteira ou
// percentual com 1 casa), { suppressed: true } (1 a 4, oculto na origem) ou
// null (a cidade não publicou o indicador naquela semana).
export type IndicatorValue = number | { suppressed: true } | null;

export interface CityAnalyticsCity {
  id: string;
  slug: string;
  name: string;
  uf: string | null;
  last_published_at: string | null;
  values: Record<string, IndicatorValue[]>;
}

export interface CityAnalyticsData {
  weeks: string[];
  indicators: string[];
  cities: CityAnalyticsCity[];
}

export async function listCityAnalytics(): Promise<CityAnalyticsData> {
  const res = await jsonFetch<{ data: CityAnalyticsData }>("/city_analytics");
  return res.data;
}
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/lib/cityAnalyticsApi.test.ts`
Expected: PASS (3 testes).

- [ ] **Step 5: Proxy de dev e README**

Em `vite.config.ts`, acrescente a linha ao comentário do topo, logo abaixo da de `/unknown_channels`:

```ts
//   /city_analytics                 → indicadores publicados pelas cidades (só leitura)
```

e a entrada no `server.proxy`, depois de `"/unknown_channels"` (a chave `"/cities"` **não** cobre `/city_analytics`: o proxy do Vite casa por prefixo e `/city_` não começa com `/cities`):

```ts
      "/unknown_channels": proxy(TARGET),
      "/city_analytics": proxy(TARGET)
```

Em `README.md`:
- na lista "O que ele faz hoje", depois do item **Registrar canal**, acrescente:

  ```markdown
  - **Analytics das cidades**: `GET /city_analytics`, só leitura. Seis
    indicadores semanais da cidade inteira que cada cidade publica na
    plataforma (ADR 0025): triagens iniciadas e concluídas, atendimentos
    encerrados, espera de até 30 min, faltas e "saiu sem atendimento" (%).
    Contagem de 1 a 4 chega como "oculto"; semana sem publicação, "sem dado".
    O console nunca vê bairro, unidade, protocolo ou pergunta.
  ```

- no parágrafo do proxy, troque ``O Vite proxa `/admin/api`, `/session`, `/mfa`, `/setup`, `/cities` e `/city_grants` `` por ``O Vite proxa `/admin/api`, `/session`, `/mfa`, `/setup`, `/cities`, `/city_grants`, `/unknown_channels` e `/city_analytics` ``;
- na árvore, troque `api.ts       ← fetch, sessão, MFA, /cities, /city_grants` por `api.ts       ← fetch, sessão, MFA, /cities, /city_grants, /city_analytics`.

- [ ] **Step 6: Tipos e commit**

Run: `cd apps/admin/.claude/mod14 && npx tsc --noEmit`
Expected: sem saída.

```bash
cd apps/admin/.claude/mod14
/opt/homebrew/bin/git add src/lib/api.ts src/lib/cityAnalyticsApi.test.ts vite.config.ts README.md
/opt/homebrew/bin/git commit -m "feat: add the city analytics client to the platform console

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras puras de exibição e `Sparkline` com lacuna

**Files:**
- Create: `src/lib/cityAnalytics.ts`
- Modify: `src/components/Sparkline.tsx`
- Test: `src/lib/cityAnalytics.test.ts`

**Interfaces:**
- Consumes: `IndicatorValue`, `CityAnalyticsData` (Task 1); `fmtNumber` de `src/lib/format.ts`.
- Produces (em `src/lib/cityAnalytics.ts`):
  ```ts
  export const INDICATOR_LABELS: Record<string, string>;
  export function indicatorLabel(indicator: string): string;
  export function isRate(indicator: string): boolean;
  export type CellKind = "value" | "hidden" | "missing";
  export interface CellView { kind: CellKind; text: string }
  export function cellView(indicator: string, v: IndicatorValue | undefined): CellView;
  export function trendPoints(values: IndicatorValue[] | undefined, length: number): (number | null)[];
  export function fmtWeek(weekStart: string): string;          // "2026-07-06" → "06/07/2026"
  export function hasAnyPublication(data: CityAnalyticsData): boolean;
  ```
- Produces (em `src/components/Sparkline.tsx`): `Sparkline({ data: (number | null)[]; color?; h? })`, com `null` desenhado como lacuna.

- [ ] **Step 1: Escreva o teste que falha**

Crie `src/lib/cityAnalytics.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import {
  cellView, fmtWeek, hasAnyPublication, indicatorLabel, isRate, trendPoints
} from "./cityAnalytics";
import type { CityAnalyticsData } from "./api";

describe("indicatorLabel", () => {
  it("names the six indicators of the fixed set", () => {
    expect(indicatorLabel("triages_started")).toBe("Triagens iniciadas");
    expect(indicatorLabel("triages_completed")).toBe("Triagens concluídas");
    expect(indicatorLabel("attendances_closed")).toBe("Atendimentos encerrados");
    expect(indicatorLabel("wait_within_30_pct")).toBe("Espera até 30 min");
    expect(indicatorLabel("no_show_pct")).toBe("Faltas");
    expect(indicatorLabel("left_pct")).toBe("Saiu sem atendimento");
  });

  it("falls back to the raw key for an indicator the console does not know yet", () => {
    expect(indicatorLabel("new_indicator")).toBe("new_indicator");
  });
});

describe("isRate", () => {
  it("treats the _pct indicators as rates", () => {
    expect(isRate("no_show_pct")).toBe(true);
    expect(isRate("triages_started")).toBe(false);
  });
});

describe("cellView", () => {
  it("formats counts as integers with thousands separator", () => {
    expect(cellView("triages_started", 1234)).toEqual({ kind: "value", text: "1.234" });
  });

  it("formats rates with exactly one decimal and a percent sign", () => {
    expect(cellView("no_show_pct", 12.5)).toEqual({ kind: "value", text: "12,5%" });
    expect(cellView("no_show_pct", 12)).toEqual({ kind: "value", text: "12,0%" });
  });

  it("zero is a value, not sem dado", () => {
    expect(cellView("triages_started", 0)).toEqual({ kind: "value", text: "0" });
    expect(cellView("left_pct", 0)).toEqual({ kind: "value", text: "0,0%" });
  });

  it("suppressed is oculto", () => {
    expect(cellView("triages_started", { suppressed: true })).toEqual({ kind: "hidden", text: "oculto" });
    expect(cellView("no_show_pct", { suppressed: true })).toEqual({ kind: "hidden", text: "oculto" });
  });

  it("null and missing are sem dado", () => {
    expect(cellView("triages_started", null)).toEqual({ kind: "missing", text: "sem dado" });
    expect(cellView("triages_started", undefined)).toEqual({ kind: "missing", text: "sem dado" });
  });
});

describe("trendPoints", () => {
  it("suprimido e sem dado viram lacuna na tendência, nunca zero", () => {
    expect(trendPoints([ 128, { suppressed: true }, null, 0 ], 4)).toEqual([ 128, null, null, 0 ]);
  });

  it("array curto ou ausente é completado com lacunas até o número de semanas", () => {
    expect(trendPoints([ 5 ], 3)).toEqual([ 5, null, null ]);
    expect(trendPoints(undefined, 2)).toEqual([ null, null ]);
  });
});

describe("fmtWeek", () => {
  it("semana formatada por texto, sem fuso", () => {
    expect(fmtWeek("2026-07-06")).toBe("06/07/2026");
    expect(fmtWeek("2026-01-05")).toBe("05/01/2026");
  });
});

describe("hasAnyPublication", () => {
  const base: CityAnalyticsData = { weeks: [], indicators: [], cities: [] };
  const city = { id: "c", slug: "c", name: "C", uf: null, values: {} };

  it("is false when no city has ever published", () => {
    expect(hasAnyPublication({ ...base, cities: [ { ...city, last_published_at: null } ] })).toBe(false);
    expect(hasAnyPublication(base)).toBe(false);
  });

  it("is true when at least one city published", () => {
    expect(hasAnyPublication({
      ...base,
      cities: [ { ...city, last_published_at: null }, { ...city, id: "d", last_published_at: "2026-09-30T05:02:12Z" } ]
    })).toBe(true);
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/lib/cityAnalytics.test.ts`
Expected: FAIL — `Failed to resolve import "./cityAnalytics"`.

- [ ] **Step 3: Implemente as regras**

Crie `src/lib/cityAnalytics.ts`:

```ts
// Regras de exibição da tela "Analytics das cidades" (módulo 14, F-14.8).
// Puras: nenhum React, nenhum fetch. O formato dos dados é o de
// GET /city_analytics (contratos §2).
//
// Três estados por célula, e eles NUNCA se confundem:
//   - valor  → número (0 incluído: zero faltas é um dado);
//   - oculto → { suppressed: true }: contagem de 1 a 4 que a cidade não deixa
//              sair (ADR 0025); o número real nunca chega aqui;
//   - sem dado → null ou ausente: a cidade não publicou o indicador naquela
//              semana (ou a taxa não tinha denominador).
// Na tendência, oculto e sem dado viram LACUNA (null), nunca zero: um zero
// desenharia uma queda que não aconteceu.

import type { CityAnalyticsData, IndicatorValue } from "./api";
import { fmtNumber } from "./format";

export const INDICATOR_LABELS: Record<string, string> = {
  triages_started: "Triagens iniciadas",
  triages_completed: "Triagens concluídas",
  attendances_closed: "Atendimentos encerrados",
  wait_within_30_pct: "Espera até 30 min",
  no_show_pct: "Faltas",
  left_pct: "Saiu sem atendimento"
};

export function indicatorLabel(indicator: string): string {
  return INDICATOR_LABELS[indicator] ?? indicator;
}

export function isRate(indicator: string): boolean {
  return indicator.endsWith("_pct");
}

export type CellKind = "value" | "hidden" | "missing";

export interface CellView {
  kind: CellKind;
  text: string;
}

// Sempre 1 casa (12,0%), diferente do fmtPercent de format.ts, que corta o
// zero à direita (12%).
const rateFmt = new Intl.NumberFormat("pt-BR", { minimumFractionDigits: 1, maximumFractionDigits: 1 });

function isSuppressed(v: IndicatorValue | undefined): v is { suppressed: true } {
  return typeof v === "object" && v !== null && v.suppressed === true;
}

export function cellView(indicator: string, v: IndicatorValue | undefined): CellView {
  if (isSuppressed(v)) return { kind: "hidden", text: "oculto" };
  if (typeof v !== "number" || Number.isNaN(v)) return { kind: "missing", text: "sem dado" };
  return { kind: "value", text: isRate(indicator) ? `${rateFmt.format(v)}%` : fmtNumber(v) };
}

export function trendPoints(values: IndicatorValue[] | undefined, length: number): (number | null)[] {
  return Array.from({ length }, (_, i) => {
    const v = values?.[i];
    return typeof v === "number" && !Number.isNaN(v) ? v : null;
  });
}

// "YYYY-MM-DD" → "DD/MM/YYYY" por texto. new Date("2026-07-06") é meia-noite
// UTC, que em São Paulo ainda é 05/07.
export function fmtWeek(weekStart: string): string {
  const [ y, m, d ] = weekStart.split("-");
  return `${d}/${m}/${y}`;
}

export function hasAnyPublication(data: CityAnalyticsData): boolean {
  return data.cities.some((c) => c.last_published_at !== null);
}
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/lib/cityAnalytics.test.ts`
Expected: PASS.

- [ ] **Step 5: `Sparkline` aceita lacuna**

Substitua `src/components/Sparkline.tsx` inteiro. A mudança é só de tipo e de guarda: `null` vira lacuna (`connectNulls={false}`, que é o padrão do Recharts, fica explícito), e uma série só de lacunas renderiza o espaço vazio, como uma série vazia já fazia. Os dois usos atuais (`StatTile`, `Overview`) passam `number[]`, que continua aceito.

```tsx
// Sparkline — linha + área. Recharts AreaChart sem eixos.
// `null` é lacuna (semana oculta ou sem dado no Analytics), nunca zero.
import { Area, AreaChart, ResponsiveContainer } from "recharts";

interface Props {
  data: (number | null)[];
  color?: string;
  h?: number;
}

export function Sparkline({ data, color = "var(--accent)", h = 32 }: Props) {
  if (!data || data.every((v) => v === null)) return <div style={{ height: h }} />;
  const series = data.map((v, i) => ({ i, v }));
  return (
    <div style={{ width: "100%", height: h }}>
      <ResponsiveContainer width="100%" height="100%">
        <AreaChart data={series} margin={{ top: 2, right: 0, bottom: 0, left: 0 }}>
          <defs>
            <linearGradient id="spark-fill" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stopColor={color} stopOpacity={0.25} />
              <stop offset="100%" stopColor={color} stopOpacity={0} />
            </linearGradient>
          </defs>
          <Area
            type="monotone"
            dataKey="v"
            stroke={color}
            strokeWidth={1.4}
            fill="url(#spark-fill)"
            connectNulls={false}
            isAnimationActive={false}
          />
        </AreaChart>
      </ResponsiveContainer>
    </div>
  );
}
```

Não há teste de renderização do `Sparkline`: o `ResponsiveContainer` precisa de `ResizeObserver`, que o jsdom não tem. O contrato dele (lacuna = `null`) é provado pelos pontos que a célula passa (Task 3) e pelo typecheck.

- [ ] **Step 6: Tipos, suíte e commit**

Run: `cd apps/admin/.claude/mod14 && npx tsc --noEmit && npx vitest run`
Expected: typecheck sem saída; vitest verde.

```bash
cd apps/admin/.claude/mod14
/opt/homebrew/bin/git add src/lib/cityAnalytics.ts src/lib/cityAnalytics.test.ts src/components/Sparkline.tsx
/opt/homebrew/bin/git commit -m "feat: add display rules for city analytics cells

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: `IndicatorCell` — valor da semana e tendência

**Files:**
- Create: `src/modules/analytics/IndicatorCell.tsx`
- Test: `src/modules/analytics/IndicatorCell.test.tsx`

**Interfaces:**
- Consumes: `IndicatorValue` (Task 1); `cellView`, `trendPoints`, `CellKind` (Task 2); `Sparkline` (Task 2).
- Produces:
  ```ts
  export function IndicatorCell(props: {
    indicator: string;
    values: IndicatorValue[] | undefined;   // alinhado com weeks
    weekIndex: number;                      // semana escolhida; -1 = nenhuma
    weeks: number;                          // weeks.length
  }): JSX.Element;
  ```
  O texto do valor sai num `<span data-kind="value|hidden|missing">`.

- [ ] **Step 1: Escreva o teste que falha**

Crie `src/modules/analytics/IndicatorCell.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen, cleanup } from "@testing-library/react";

// O ResponsiveContainer do Recharts não roda no jsdom (sem ResizeObserver).
// O dublê expõe os pontos que a célula passou.
vi.mock("../../components/Sparkline", () => ({
  Sparkline: ({ data }: { data: (number | null)[] }) => (
    <span data-testid="spark" data-points={JSON.stringify(data)} />
  )
}));

import { IndicatorCell } from "./IndicatorCell";

afterEach(cleanup);

describe("IndicatorCell", () => {
  it("shows the value of the chosen week", () => {
    render(<IndicatorCell indicator="triages_started" values={[ 100, 128 ]} weekIndex={0} weeks={2} />);
    expect(screen.getByText("100").getAttribute("data-kind")).toBe("value");
    expect(screen.queryByText("128")).toBeNull();
  });

  it("oculto e sem dado têm textos e marcas diferentes", () => {
    render(
      <>
        <IndicatorCell indicator="triages_started" values={[ { suppressed: true } ]} weekIndex={0} weeks={1} />
        <IndicatorCell indicator="triages_started" values={[ null ]} weekIndex={0} weeks={1} />
      </>
    );
    expect(screen.getByText("oculto").getAttribute("data-kind")).toBe("hidden");
    expect(screen.getByText("sem dado").getAttribute("data-kind")).toBe("missing");
    expect(screen.getByText("oculto").getAttribute("title")).toMatch(/1 a 4/);
  });

  it("rates show one decimal", () => {
    render(<IndicatorCell indicator="no_show_pct" values={[ 12 ]} weekIndex={0} weeks={1} />);
    expect(screen.getByText("12,0%")).toBeTruthy();
  });

  it("the trend covers every week, with gaps for oculto and sem dado", () => {
    render(
      <IndicatorCell indicator="triages_started" values={[ 90, { suppressed: true }, null, 0 ]} weekIndex={3} weeks={4} />
    );
    expect(screen.getByTestId("spark").getAttribute("data-points")).toBe("[90,null,null,0]");
  });

  it("no chosen week (weekIndex -1) is sem dado, not a crash", () => {
    render(<IndicatorCell indicator="triages_started" values={undefined} weekIndex={-1} weeks={0} />);
    expect(screen.getByText("sem dado")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/modules/analytics/IndicatorCell.test.tsx`
Expected: FAIL — `Failed to resolve import "./IndicatorCell"`.

- [ ] **Step 3: Implemente a célula**

Crie `src/modules/analytics/IndicatorCell.tsx`:

```tsx
// Célula da tabela "Analytics das cidades": o valor da semana escolhida em
// cima e a tendência das semanas que vieram (12 por padrão) embaixo.
// "oculto" e "sem dado" têm texto, cor e título próprios — nunca "0".

import type { IndicatorValue } from "../../lib/api";
import { cellView, trendPoints, type CellKind } from "../../lib/cityAnalytics";
import { Sparkline } from "../../components/Sparkline";

interface Props {
  indicator: string;
  values: IndicatorValue[] | undefined;
  weekIndex: number;
  weeks: number;
}

const TITLES: Record<CellKind, string | undefined> = {
  value: undefined,
  hidden: "contagem de 1 a 4, suprimida na cidade antes de sair dela",
  missing: "a cidade não publicou este indicador nesta semana"
};

const COLORS: Record<CellKind, string> = {
  value: "var(--ink)",
  hidden: "var(--ink3)",
  missing: "var(--ink4)"
};

export function IndicatorCell({ indicator, values, weekIndex, weeks }: Props) {
  const view = cellView(indicator, weekIndex >= 0 ? values?.[weekIndex] : undefined);
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 2, minWidth: 0 }}>
      <span
        data-kind={view.kind}
        title={TITLES[view.kind]}
        className="mono"
        style={{
          fontSize: view.kind === "value" ? 12.5 : 11,
          fontStyle: view.kind === "hidden" ? "italic" : undefined,
          color: COLORS[view.kind]
        }}
      >
        {view.text}
      </span>
      <Sparkline data={trendPoints(values, weeks)} h={22} />
    </div>
  );
}
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/modules/analytics/IndicatorCell.test.tsx`
Expected: PASS (5 testes).

- [ ] **Step 5: Tipos e commit**

Run: `cd apps/admin/.claude/mod14 && npx tsc --noEmit`
Expected: sem saída.

```bash
cd apps/admin/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/IndicatorCell.tsx src/modules/analytics/IndicatorCell.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the indicator cell with weekly value and trend

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tela "Analytics das cidades"

**Files:**
- Create: `src/modules/analytics/CityAnalytics.tsx`
- Test: `src/modules/analytics/CityAnalytics.test.tsx`

**Interfaces:**
- Consumes: `listCityAnalytics`, `CityAnalyticsData`, `CityAnalyticsCity`, `ApiError` (Task 1); `indicatorLabel`, `fmtWeek`, `hasAnyPublication` (Task 2); `IndicatorCell` (Task 3); `PageHeader`, `DataTable`/`Column`, `EmptyState`, `ErrorState`, `Skeleton` de `src/components/`; `fmtDateTime` de `src/lib/format.ts`.
- Produces: `export function CityAnalytics(): JSX.Element` (sem props).

- [ ] **Step 1: Escreva o teste que falha**

Crie `src/modules/analytics/CityAnalytics.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, cleanup, fireEvent } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("../../lib/api")>();
  return { ...actual, listCityAnalytics: vi.fn() };
});

vi.mock("../../components/Sparkline", () => ({
  Sparkline: ({ data }: { data: (number | null)[] }) => (
    <span data-testid="spark" data-points={JSON.stringify(data)} />
  )
}));

import { listCityAnalytics, ApiError, type CityAnalyticsData } from "../../lib/api";
import { CityAnalytics } from "./CityAnalytics";
import { fmtDateTime } from "../../lib/format";

const mocked = vi.mocked(listCityAnalytics);

function renderScreen(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

const INDICATORS = [
  "triages_started", "triages_completed", "attendances_closed",
  "wait_within_30_pct", "no_show_pct", "left_pct"
];

const DATA: CityAnalyticsData = {
  weeks: [ "2026-09-14", "2026-09-21" ],
  indicators: INDICATORS,
  cities: [
    {
      id: "c1", slug: "curitiba", name: "Curitiba", uf: "PR",
      last_published_at: "2026-09-30T05:02:12Z",
      values: {
        triages_started: [ 100, 128 ],
        triages_completed: [ 90, { suppressed: true } ],
        attendances_closed: [ 80, null ],
        wait_within_30_pct: [ 50, 12.5 ],
        no_show_pct: [ 7, 0 ],
        left_pct: [ null, null ]
      }
    },
    { id: "c2", slug: "maringa", name: "Maringá", uf: "PR", last_published_at: null, values: {} }
  ]
};

describe("CityAnalytics", () => {
  beforeEach(() => { mocked.mockReset(); });
  afterEach(() => { cleanup(); });

  it("lists every city with the six indicator columns and the last publication", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    expect(await screen.findByText("Curitiba · PR")).toBeTruthy();
    expect(screen.getByText("Maringá · PR")).toBeTruthy();
    for (const label of [
      "Triagens iniciadas", "Triagens concluídas", "Atendimentos encerrados",
      "Espera até 30 min", "Faltas", "Saiu sem atendimento", "Publicado em"
    ]) {
      expect(screen.getByText(label)).toBeTruthy();
    }
    expect(screen.getByText(fmtDateTime("2026-09-30T05:02:12Z"))).toBeTruthy();
    expect(fmtDateTime("2026-09-30T05:02:12Z")).toBe("30/09/2026, 02:02");
  });

  it("opens on the most recent week", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    expect(await screen.findByText("128")).toBeTruthy();
    expect(screen.queryByText("100")).toBeNull();
    expect((screen.getByLabelText("Semana") as HTMLSelectElement).value).toBe("2026-09-21");
  });

  it("lists the weeks newest first, formatted without time zone shift", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    const select = (await screen.findByLabelText("Semana")) as HTMLSelectElement;
    const labels = Array.from(select.options).map((o) => o.textContent);
    expect(labels).toEqual([ "semana de 21/09/2026", "semana de 14/09/2026" ]);
  });

  it("switching the week shows that week's values", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    const select = await screen.findByLabelText("Semana");
    fireEvent.change(select, { target: { value: "2026-09-14" } });

    expect(screen.getByText("100")).toBeTruthy();
    expect(screen.getByText("90")).toBeTruthy();
    expect(screen.getByText("50,0%")).toBeTruthy();
    expect(screen.getByText("7,0%")).toBeTruthy();
    expect(screen.queryByText("128")).toBeNull();
  });

  it("oculto e sem dado aparecem distintos; 0,0% aparece como valor", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    await screen.findByText("128");
    expect(screen.getAllByText("oculto")).toHaveLength(1);
    // Curitiba: attendances_closed e left_pct; Maringá: as seis.
    expect(screen.getAllByText("sem dado")).toHaveLength(8);
    expect(screen.getByText("12,5%")).toBeTruthy();
    expect(screen.getByText("0,0%")).toBeTruthy();
  });

  it("explains oculto and sem dado in a legend", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    await screen.findByText("128");
    expect(screen.getByText(/contagem de 1 a 4, suprimida na cidade/)).toBeTruthy();
    expect(screen.getByText(/não publicou o indicador naquela semana/)).toBeTruthy();
  });

  it("tendência leva null nas semanas ocultas e sem dado", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    await screen.findByText("128");
    const points = screen.getAllByTestId("spark").map((s) => s.getAttribute("data-points"));
    // Linha de Curitiba, na ordem de data.indicators.
    expect(points.slice(0, 6)).toEqual([
      "[100,128]", "[90,null]", "[80,null]", "[50,12.5]", "[7,0]", "[null,null]"
    ]);
  });

  it("cidade sem publicação: nunca publicou e sem dado, sem quebrar", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen(<CityAnalytics />);

    expect(await screen.findByText("nunca publicou")).toBeTruthy();
    const points = screen.getAllByTestId("spark").map((s) => s.getAttribute("data-points"));
    expect(points.slice(6)).toEqual(Array(6).fill("[null,null]"));
  });

  it("ninguém publicou: estado vazio explicado, sem tabela", async () => {
    mocked.mockResolvedValue({
      ...DATA,
      cities: DATA.cities.map((c) => ({ ...c, last_published_at: null, values: {} }))
    });
    renderScreen(<CityAnalytics />);

    expect(await screen.findByText("Nenhuma cidade publicou indicadores ainda.")).toBeTruthy();
    expect(screen.getByText(/consolidação roda todo dia/)).toBeTruthy();
    expect(screen.queryByRole("table")).toBeNull();
    expect(screen.queryByLabelText("Semana")).toBeNull();
  });

  it("shows the empty state when there is no active city", async () => {
    mocked.mockResolvedValue({ ...DATA, cities: [] });
    renderScreen(<CityAnalytics />);

    expect(await screen.findByText("Nenhuma cidade ativa.")).toBeTruthy();
    expect(screen.queryByRole("table")).toBeNull();
  });

  it("shows the error state when the request fails", async () => {
    mocked.mockRejectedValue(new ApiError(500, "", "500 on /city_analytics"));
    renderScreen(<CityAnalytics />);

    expect(await screen.findByText("Falha ao carregar")).toBeTruthy();
    expect(screen.getByText("500 on /city_analytics")).toBeTruthy();
    expect(screen.queryByRole("table")).toBeNull();
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/modules/analytics/CityAnalytics.test.tsx`
Expected: FAIL — `Failed to resolve import "./CityAnalytics"`.

- [ ] **Step 3: Implemente a tela**

Crie `src/modules/analytics/CityAnalytics.tsx`:

```tsx
// Analytics das cidades (módulo 14, F-14.8) — só leitura. Fala com
// GET /city_analytics (PlatformConsoleHost, operador). Mostra os seis
// indicadores semanais que cada cidade publica na plataforma, da cidade
// inteira, já suprimidos na origem (ADR 0025): o console nunca vê bairro,
// unidade, protocolo ou pergunta, e o número de 1 a 4 nunca chega aqui.
//
// Sem from/to: o servidor devolve as 12 semanas que terminam na semana
// anterior à atual. O seletor escolhe entre elas (a mais recente por
// padrão); cada célula mostra a semana escolhida e a tendência de todas.

import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { listCityAnalytics, type CityAnalyticsCity } from "../../lib/api";
import { fmtWeek, hasAnyPublication, indicatorLabel } from "../../lib/cityAnalytics";
import { fmtDateTime } from "../../lib/format";
import { PageHeader } from "../../components/PageHeader";
import { DataTable, type Column } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { ErrorState } from "../../components/ErrorState";
import { Skeleton } from "../../components/Skeleton";
import { IndicatorCell } from "./IndicatorCell";

export function CityAnalytics() {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: [ "city_analytics" ],
    queryFn: listCityAnalytics,
    staleTime: 60_000
  });
  const [ picked, setPicked ] = useState<string | null>(null);

  const weeks = data?.weeks ?? [];
  const week = picked !== null && weeks.includes(picked) ? picked : weeks[weeks.length - 1];
  const weekIndex = week === undefined ? -1 : weeks.indexOf(week);

  const cols: Column<CityAnalyticsCity>[] = [
    { label: "Cidade", w: "1.2fr", render: (c) => <span>{c.name}{c.uf ? ` · ${c.uf}` : ""}</span> },
    ...(data?.indicators ?? []).map((indicator): Column<CityAnalyticsCity> => ({
      label: indicatorLabel(indicator),
      w: "1fr",
      render: (c) => (
        <IndicatorCell indicator={indicator} values={c.values[indicator]} weekIndex={weekIndex} weeks={weeks.length} />
      )
    })),
    {
      label: "Publicado em",
      w: "1fr",
      render: (c) => (c.last_published_at ? fmtDateTime(c.last_published_at) : "nunca publicou")
    }
  ];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Analytics das cidades" sub="indicadores publicados · semanal" />
      <p style={{ fontSize: 12, color: "var(--ink3)", margin: 0 }}>
        Seis indicadores da cidade inteira, publicados uma vez por dia por cada cidade, com dados até
        o dia anterior. O console não vê bairro, unidade, protocolo nem resposta de triagem.
      </p>

      {isLoading && <Skeleton rows={6} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}

      {data && data.cities.length === 0 && <EmptyState title="Nenhuma cidade ativa." />}

      {data && data.cities.length > 0 && !hasAnyPublication(data) && (
        <EmptyState
          title="Nenhuma cidade publicou indicadores ainda."
          sub="a consolidação roda todo dia às 2h (America/Sao_Paulo); no primeiro deploy, rode city:analytics:rebuild:all"
        />
      )}

      {data && data.cities.length > 0 && hasAnyPublication(data) && (
        <>
          <div style={{ display: "inline-flex", alignItems: "center", gap: 8, fontSize: 12, color: "var(--ink2)" }}>
            <span aria-hidden="true">Semana</span>
            <select
              aria-label="Semana"
              value={week ?? ""}
              onChange={(e) => setPicked(e.target.value)}
              className="mono"
              style={{ fontSize: 12, padding: "4px 8px", borderRadius: 6, border: "1px solid var(--rule2)", background: "var(--panel)", color: "var(--ink)" }}
            >
              {[ ...weeks ].reverse().map((w) => (
                <option key={w} value={w}>{`semana de ${fmtWeek(w)}`}</option>
              ))}
            </select>
          </div>
          <p style={{ fontSize: 11, color: "var(--ink3)", margin: 0 }}>
            “oculto”: contagem de 1 a 4, suprimida na cidade antes de sair dela. “sem dado”: a cidade não publicou o indicador naquela semana, ou a taxa não tinha denominador. Na tendência, os dois aparecem como lacuna.
          </p>
          <DataTable<CityAnalyticsCity> cols={cols} rows={data.cities} rowKey={(c) => c.id} />
        </>
      )}
    </div>
  );
}
```

Observações para quem implementa:
- O seletor tem o nome acessível só pelo `aria-label`. Um `<label>` em volta dele juntaria o texto das `<option>` ao rótulo, e o `getByLabelText("Semana")` deixaria de casar.
- A legenda é **um nó de texto só**: se "oculto" virasse `<strong>`, o `getAllByText("oculto")` do teste contaria a legenda como célula.
- `picked` guarda a escolha; se um refetch trouxer semanas sem ela, a tela volta para a mais recente em vez de mostrar "sem dado" em tudo.
- As colunas saem de `data.indicators` (ordem do servidor), com rótulo local; um indicador novo no api aparece com a chave crua em vez de sumir.

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/modules/analytics/CityAnalytics.test.tsx`
Expected: PASS (11 testes).

- [ ] **Step 5: Tipos e commit**

Run: `cd apps/admin/.claude/mod14 && npx tsc --noEmit`
Expected: sem saída.

```bash
cd apps/admin/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/CityAnalytics.tsx src/modules/analytics/CityAnalytics.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the city analytics screen to the platform console

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Navegação

**Files:**
- Modify: `src/shell/modules.ts` (tipo `ModuleId` e `NAV_GROUPS`)
- Modify: `src/App.tsx` (import e `renderModule`)
- Test: `src/shell/modules.test.ts`

**Interfaces:**
- Consumes: `CityAnalytics` (Task 4).
- Produces: `ModuleId` ganha `"city_analytics"`; `NAV_GROUPS` ganha o grupo `"Analytics"` com o item `{ id: "city_analytics", label: "Analytics das cidades" }`, só para operador.

- [ ] **Step 1: Escreva o teste que falha**

Acrescente ao fim de `src/shell/modules.test.ts`:

```ts
describe("NAV_GROUPS analytics", () => {
  const group = NAV_GROUPS.find((g) => g.label === "Analytics");
  const item = group?.items.find((i) => i.id === "city_analytics");

  it("lists Analytics das cidades in its own group", () => {
    expect(item?.label).toBe("Analytics das cidades");
  });

  it("shows it to operators only", () => {
    expect(item?.visible?.({ operator: true, memberships: [] })).toBe(true);
    expect(item?.visible?.({ operator: false, memberships: [ { role: "municipal_admin" } ] })).toBe(false);
  });
});
```

- [ ] **Step 2: Rode e confirme que falha**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/shell/modules.test.ts`
Expected: FAIL — `expected undefined to be 'Analytics das cidades'`.

- [ ] **Step 3: Implemente**

Em `src/shell/modules.ts`, acrescente o id ao fim do tipo:

```ts
  | "register_channel"
  | "unknown_channels"
  // Módulo 14 (ADR 0025)
  | "city_analytics";
```

e um grupo novo depois de `"Setup"` em `NAV_GROUPS` (Analytics não é setup; grupo próprio deixa o menu Setup só com provisionamento):

```ts
  },
  {
    label: "Analytics",
    items: [
      {
        id: "city_analytics",
        label: "Analytics das cidades",
        icon: "◔",
        visible: (u) => u.operator
      }
    ]
  }
];
```

Em `src/App.tsx`, o import junto dos outros módulos:

```ts
import { CityAnalytics } from "./modules/analytics/CityAnalytics";
```

e o caso no fim do `switch` de `renderModule`:

```tsx
    case "unknown_channels": return <UnknownChannels />;
    case "city_analytics": return <CityAnalytics />;
```

- [ ] **Step 4: Rode e confirme que passa**

Run: `cd apps/admin/.claude/mod14 && npx vitest run src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS; typecheck sem saída.

- [ ] **Step 5: Commit**

```bash
cd apps/admin/.claude/mod14
/opt/homebrew/bin/git add src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add city analytics to the platform console navigation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/admin/.claude/mod14 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI (`.github/workflows/ci.yml`) roda os mesmos três. O número de arquivos de teste **não** dobra (se dobrar, um build antigo deixou `.js` em `src/`; ver `noEmit` no `tsconfig.json`).

```bash
cd apps/admin/.claude/mod14 && /opt/homebrew/bin/git status --short
```

Expected: vazio. O symlink `node_modules` não aparece porque é ignorado. Se aparecer, **não** o adicione.

- [ ] **Step 2: Nenhuma dependência nova**

```bash
cd apps/admin/.claude/mod14 && /opt/homebrew/bin/git diff origin/main..HEAD -- package.json package-lock.json pnpm-lock.yaml
```

Expected: nenhuma linha.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do admin contra a spec (§6.2, §9, §10.3), o ADR 0025 e os contratos §2. Pontos de atenção:
- "oculto" × "sem dado" × `0` nunca se confundem, nem na célula nem na tendência (nada de `if (!v)` / `v || 0`);
- percentuais sempre com 1 casa;
- nenhuma data `YYYY-MM-DD` passa por `new Date`;
- a tela não manda `from`/`to` nem nenhum parâmetro;
- nenhum `git add -A` no histórico (`git log --stat origin/main..HEAD` sem `node_modules` e sem `design_handoff_admin_console 4/`).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do módulo 14 rodando com a semente e o `city:analytics:rebuild` (spec §11), na porta que o plano do api definir (no módulo 11 foi `:3031`). Rode o Vite do worktree apontando para ele:

```bash
cd apps/admin/.claude/mod14 && VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5180 --host 0.0.0.0
```

Abra `http://admin.localhost:5180/admin/`. O usuário faz o login de operador; não digite senha nem TOTP. Confira com screenshot:
- menu **Analytics → Analytics das cidades**;
- Curitiba e Maringá com os seis indicadores, a tendência de 12 semanas e "Publicado em";
- percentuais com 1 casa;
- trocar a semana muda os valores e não a tendência;
- ao menos um "oculto" (semana de volume baixo na semente) e a legenda.

- [ ] **Step 5:** **Pare.** O merge do admin só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é api → dashboard → admin → maintenance (spec §12).

---

## Desvios

- **Seletor de semana limitado às 12 semanas padrão.** A spec (§9) diz "semana escolhida" e "tendência das últimas 12 semanas"; o contrato §2 aceita `from`/`to` até 104 semanas. Esta tela não manda período: o seletor escolhe entre as 12 semanas que o padrão do servidor devolve. Escolher uma semana mais antiga exigiria outra janela para a tendência (as 12 anteriores a ela) e um segundo fetch; fica de fora até alguém pedir.
- **Grupo de menu novo "Analytics"** em vez de pôr o item em "Setup". A spec não diz onde; Setup é provisionamento.

## Self-review

- **Cobertura da spec:** §6.2 (cliente, sem abrir cidade) → Task 1; §9 "tabela cidade × indicador da semana escolhida" → Task 4; "tendência das últimas 12 semanas por célula" → Tasks 2–4; "oculto e sem dado distintos" → Tasks 2–4; §10.3 "no admin oculto × sem dado" → Tasks 2, 3 e 4; pedidos do caller: 1 casa decimal (Task 2), `last_published_at` (Task 4), seletor (Task 4), erro e vazio (Task 4), navegação (Task 5), sem dependência nova (Task 6, Step 2).
- **Placeholders:** nenhum; todo passo de código tem o código.
- **Tipos:** `IndicatorValue`, `CityAnalyticsData`, `CityAnalyticsCity` (Task 1) são os usados nas Tasks 2–4; `cellView`/`trendPoints`/`CellKind` (Task 2) batem com a Task 3; `IndicatorCell` recebe `{ indicator, values, weekIndex, weeks }` na Task 3 e na Task 4.
- **Review Focus:** as cinco linhas têm teste nas Tasks 2 e 4, citados pelo nome.
- **Código conferido:** o código das Tasks 1–5 foi aplicado numa cópia descartável de `origin/main` do admin (`9a8c0f6`) em 2026-09-30: `tsc --noEmit` limpo, vitest 13 arquivos / 80 testes verdes, `npm run build` ok.
