# Módulo 14 — Analytics (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Área "Analytics" do dashboard da cidade. O `analyst` e o `municipal_admin` leem quatro abas (Demanda, Qualidade, Calibração, Epidemiologia) com séries consolidadas até D-1, recortes por bairro, unidade e protocolo, e toda contagem de 1 a 4 aparece como "oculto". O autor marca perguntas `boolean`/`enum` como analíticas no editor de protocolo, e o `municipal_admin` concede o papel "Análise" em Equipe, sem step-up.

**Architecture:** O cliente de `GET /admin/api/analytics/:front` e os tipos do contrato vão para `src/lib/api.ts`. As regras puras (formatação de célula e taxa, carimbo "dados até", intervalos por semana ou mês, opções dos recortes, tradução das recusas) ficam em `src/lib/analytics.ts`, sem React. As telas moram em `src/modules/analytics/`: peças comuns (valor, carimbo, gráfico, bloco de série, moldura de carregamento/erro/vazio, filtros) e uma aba por frente, cada uma testável isolada por props. A raiz `src/modules/Analytics.tsx` guarda o intervalo, compartilhado entre as abas, e troca de aba sem URL. A marcação `analytic` no editor é uma função pura em `src/lib/editor.ts` mais um painel ao lado do JSON.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Recharts 2.13 (já é dependência; nenhuma lib nova), Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.stubGlobal("fetch")` por rota e `vi.mock("../lib/api")` onde a tela já é testada assim), Vite 5.

**Spec:** `docs/superpowers/specs/2026-09-30-module-14-analytics-design.md` (§7 e §8 são deste plano; §6.1 e §6.3 dão o contrato de leitura; §10.3 lista os testes) e `docs/adr/0025.md`. Os formatos HTTP estão fixados em `docs/superpowers/plans/2026-09-30-module-14-analytics-contracts.md` (§0, §1, §4 e §5), o mesmo arquivo que o plano do api usa; os tipos da Task 1 os copiam literalmente. O plano do api (escrito em paralelo) precisa estar **mergeado antes** do merge deste.

## Global Constraints

- Rota única: `GET /admin/api/analytics/:front`, `:front` ∈ `demand`, `quality`, `calibration`, `epidemiology`. Já passa pelo proxy `/admin/api` do Vite; **nenhuma** entrada nova no proxy.
- Parâmetros (contratos §1):
  - `from`, `to` (`YYYY-MM-DD`) sempre;
  - `granularity` (`week` | `month`) só em `demand`, `quality` e `epidemiology`, **nunca** em `calibration`;
  - `neighborhood_id` (uuid ou `none`) em `demand` e `epidemiology`;
  - `health_unit_id` em `demand` e `quality`;
  - `protocol_name` em `demand`, `calibration` e `epidemiology`;
  - `protocol_version` em `calibration` e `epidemiology`, e **só junto com** `protocol_name`.
  - Recorte vazio vai ausente, nunca `""` nem `null`.
- Limites: `week` até **104** períodos, `month` até **60**; fora disso a API responde `422 invalid_range`.
- Unidades do seletor: `demand` e `quality` trazem sempre `data.units: [{ health_unit_id, name, active }]`, com todas as unidades da cidade, independentes do recorte (contratos §1). É a única fonte do seletor; unidade inativa aparece como "(inativa)". O `analyst` não lê `/attendance/units`.
- Totais das triagens: `demand` traz `triages_total: { started, completed, aborted }` (Cells), que são os totais do período do bloco "Triagens".
- Envelope: `{ data, as_of, stale }`. `as_of` é `null` quando nunca houve consolidação, e aí `stale` é `true`.
- Recusas: `403 { error: "forbidden_role" }`; 422 com `invalid_range`, `invalid_neighborhood`, `invalid_unit` ou `invalid_protocol`.
- **Cell** = inteiro ≥ 0 ou `{ suppressed: true }`. **Rate** = número com 1 casa, `{ suppressed: true }` ou `null`.
- Texto das células (spec §8, termos literais):
  - suprimido = "oculto", tanto em contagem quanto em taxa;
  - taxa `null` = "sem dado";
  - taxa com 1 casa decimal e vírgula: `87,5%`, `100,0%`, `0,0%`.
- **O cliente nunca soma, subtrai nem deriva número** de células: total, taxa e ordenação vêm da API. Ponto oculto ou sem dado é lacuna no gráfico, nunca zero.
- Carimbo "dados até DD/MM" (dia da cidade de `as_of`, menos 1); aviso "dados desatualizados" quando `stale`; estado vazio "ainda sem dados consolidados" quando `as_of` é `null`.
- Menu "Analytics": visível para `analyst` e `municipal_admin`; **nunca** para o operador, com ou sem grant (ADR 0025, D12).
- Editor (spec §7, literal): caixa "Usar em Analytics" só em pergunta `boolean`/`enum`, com a dica "respostas desta pergunta aparecerão agregadas por bairro, nunca por pessoa". Pergunta `integer`/`text` nunca sai salva com `analytic`.
- Papel `analyst`: rótulo "Análise"; **fora** de `PRIVILEGED_ROLES`, então conceder, revogar e convidar vão **sem** step-up (contratos §5).
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(NOW)` em `beforeEach`, e `vi.useRealTimers()` em `afterEach`.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. `/.claude/` já está no `.gitignore` do dashboard.

  ```bash
  cd apps/dashboard
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod14 -b feat/mod-14-analytics origin/main
  ln -s ../../node_modules .claude/mod14/node_modules
  ```

- Testes (script `test` = `vitest run`):

  ```bash
  cd apps/dashboard/.claude/mod14 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit (script `typecheck` = `tsc --noEmit`; o `tsconfig` tem `noUnusedLocals` e `noUnusedParameters`):

  ```bash
  cd apps/dashboard/.claude/mod14 && npx tsc --noEmit
  ```

- Base do ambiente de teste:
  - o `vitest.config.ts` já fixa `TZ=America/Sao_Paulo`;
  - não há `setupFiles` nem jest-dom (use `toBeTruthy`, `not.toBeNull()` e `toBeNull()`);
  - `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`;
  - o Recharts dentro de `ResponsiveContainer` precisa de `ResizeObserver`; o `renderWithQuery` da Task 1 o substitui por um no-op. Em jsdom o gráfico tem tamanho zero e não desenha: os testes conferem números nas **tabelas**, nunca no SVG.

## Review Focus

1. **Contagem suprimida tratada como zero ou somada no cliente.** Com um ponto oculto na série, somar os pontos daria um número falso, ou revelaria o oculto por diferença. O total do período é sempre o que a API mandou (`total`, `triages_total`), nunca uma soma feita na tela. Testes:
   - Task 3, "no gráfico, oculto e sem dado viram lacuna" e "sem total na linha, sem tabela de totais";
   - Task 5, "triagens: total do período vem de triages_total, não da soma dos pontos".
2. **"Dados até" na virada do dia.** Um `as_of` às 23h30 em São Paulo já é o dia seguinte em UTC; ler a data em UTC adiantaria o carimbo em um dia. Idem para o fim do intervalo pedido. Testes: Task 2, "as_of às 23h30 em São Paulo" e "às 23h30 em São Paulo, ontem ainda é o dia anterior da cidade".
3. **Seletor de unidade montado a partir das séries.** A tentação é tirar as opções de `attendances_by_unit`/`by_unit`; com uma unidade escolhida, essas listas encolhem para ela, e a unidade sem atendimento no intervalo nunca aparece. A fonte é `data.units`, com as inativas marcadas. Testes: Task 4, "unidade inativa aparece marcada"; Task 5, "unidade: a lista vem de data.units e continua inteira depois de escolher"; Task 6, a mesma regra.
4. **Versão sem protocolo.** `protocol_version` sozinho é `422 invalid_protocol`. Trocar o protocolo zera a versão, a versão fica travada sem protocolo, e o cliente nunca manda a versão sozinha. Testes:
   - Task 1, "protocol_version só vai junto com protocol_name";
   - Task 4, "versão travada sem protocolo; trocar o protocolo zera a versão";
   - Task 7, "trocar o protocolo tira a versão do pedido".
5. **Pergunta marcada que mudou de tipo no JSON.** O autor troca `"boolean"` por `"integer"` numa pergunta que já tinha `analytic: true`. A tela avisa antes, e o salvar tira a marca e mostra o texto salvo; o rascunho nunca chega à API com `analytic` em `integer`/`text`. Testes: Task 10, "pergunta que virou integer perde a marca ao salvar, com aviso antes" e `stripIneligibleAnalytic`.

Também cobertos, fora dos cinco: papel revogado no meio da sessão (403 vira frase, Task 3); `as_of` nulo não mostra carimbo nem zeros (Task 3); link com bairro que não existe (422 vira frase, Task 5).

## Desvios (spec/contratos × código real)

1. **O editor de protocolo é JSON cru.** `src/modules/ProtocolEditor.tsx` é um `<textarea>` com o JSON inteiro, sem formulário por pergunta nem seletor de `answer_type`; as perguntas são `steps[]` (`id`, `prompt`, `answer_type`, `options`), não `questions`. A caixa "Usar em Analytics" vai para um painel novo, "Perguntas para Analytics", ao lado do JSON, derivado do JSON parseado: marcar ou desmarcar reescreve o texto. "Limpar `analytic` quando o tipo muda para `integer`/`text`" vira: aviso imediato no painel e remoção **ao salvar**, com o texto reescrito para mostrar o que foi salvo. Limpar a cada tecla apagaria a marca em estados intermediários (ao trocar `"boolean"` por `"enum"`, o texto passa por `""`), então não é feito.
2. **"Oculto" também em contagem.** Nos painéis do módulo 11, contagem suprimida aparece como "< 5" e só taxa/média como "oculto" (`src/lib/smallCount.ts`). A spec §8 e o ADR 0025 mandam "oculto" para toda célula do Analytics; este plano segue a spec e não mexe nos painéis do módulo 11.
3. **Lista de protocolos.** Vem de `GET /admin/api/protocols` pelo `listAuthorProtocols` que já existe (qualquer membership lê `/admin/api`). A versão chega como string (`"3"`) e vira número; versões em `draft` ficam fora, porque nunca tiveram triagem.

Decisão de leitura, não desvio: o carimbo "dados até" sai do **dia da cidade de `as_of` menos 1**, e não de `data.to`. A API trunca `to` para ontem mesmo quando a última consolidação boa é de dias atrás; `as_of` é o que diz até onde os fatos existem.

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/api.ts` | tipos do contrato e `fetchAnalytics` | 1 |
| `src/test/analyticsFixtures.tsx` | relógio, dados das quatro frentes, stub de fetch por rota, `renderWithQuery`, `rowWith` | 1 |
| `src/lib/analytics.ts` | textos, `fmtCell`, `fmtRate`, `plotValue`, `dataUntil`, `fmtPeriod`, intervalos, chaves de cache, opções dos recortes, rótulos, `analyticsError` | 2 |
| `src/hooks/useAnalytics.ts` | consulta por frente | 3 |
| `src/modules/analytics/values.tsx`, `DataStamp.tsx`, `SeriesChart.tsx`, `SeriesBlock.tsx`, `AnalyticsView.tsx` | peças comuns das abas | 3 |
| `src/modules/analytics/Filters.tsx` | intervalo, bairro, unidade, protocolo e versão | 4 |
| `src/modules/analytics/DemandTab.tsx` | aba Demanda (F-14.3) | 5 |
| `src/modules/analytics/QualityTab.tsx` | aba Qualidade (F-14.4) | 6 |
| `src/modules/analytics/CalibrationTab.tsx` | aba Calibração (F-14.5) | 7 |
| `src/modules/analytics/EpidemiologyTab.tsx` | aba Epidemiologia (F-14.7) | 8 |
| `src/modules/Analytics.tsx`, `src/shell/modules.ts`, `src/App.tsx` | raiz, abas e menu por papel (F-14.2) | 9 |
| `src/lib/editor.ts`, `src/modules/protocolEditor/AnalyticQuestions.tsx`, `src/modules/ProtocolEditor.tsx` | caixa "Usar em Analytics" (F-14.6) | 10 |
| `src/lib/team.ts`, `src/modules/Team.tsx` | papel "Análise" em Equipe (F-14.2) | 11 |
| — | suíte, build, revisão e prova no navegador | 12 |

Mapa de F-IDs: F-14.2 = Tasks 1–4, 9 e 11; F-14.3 = Task 5; F-14.4 = Task 6; F-14.5 = Task 7; F-14.6 = Task 10; F-14.7 = Task 8. F-14.1, F-14.8 e F-14.9 não têm parte no dashboard.

---

### Task 1: Cliente de `/admin/api/analytics`, tipos do contrato e fixtures

**Files:**
- Modify: `src/lib/api.ts` (fim do arquivo)
- Create: `src/test/analyticsFixtures.tsx`
- Test: `src/lib/api.analytics.test.ts`

**Interfaces:**
- Consumes: `adminFetch`, `ApiError` e `AttendanceOutcome`, que já estão em `src/lib/api.ts`.
- Produces (em `src/lib/api.ts`):
  - tipos `Cell`, `Rate`, `AnalyticsFront`, `Granularity`, `AnalyticsFilterEcho`, `AnalyticsBase`, `SeriesRow`, `AnalyticsUnit { health_unit_id; name; active }`, `DemandData`, `WaitBucket`, `AppointmentEndStatus`, `QualityData`, `CalibrationOutcome`, `CalibrationRow`, `CalibrationVersion`, `CalibrationData`, `EpiOption`, `EpiQuestion`, `EpidemiologyData`, `AnalyticsDataMap`, `AnalyticsEnvelope<F>`, `AnalyticsQuery`;
  - `fetchAnalytics<F extends AnalyticsFront>(front: F, query: AnalyticsQuery): Promise<AnalyticsEnvelope<F>>`.
- Produces (em `src/test/analyticsFixtures.tsx`): `NOW`, `AS_OF`, `PERIODS`, `HIDDEN`, `NB1`, `NB2`, `U1`, `U2`, `U3`, `UNITS`, `NEIGHBORHOODS`, `PROTOCOL_ROWS`, `demandData()`, `qualityData()`, `calibrationData()`, `epidemiologyData()` (todas com `overrides?`), `envelope<F>(data, { as_of?, stale? })`, `failWith(status, body)`, `stubAnalyticsApi(routes)`, `paramsOf(fn, path) → URLSearchParams[]`, `renderWithQuery(ui)`, `rowWith(container, text) → string`.

- [ ] **Step 1: Escreva os tipos do contrato (só tipos, sem comportamento)**

No fim de `src/lib/api.ts`, acrescente:

```ts
// ─── Analytics (módulo 14; ADR 0025; contratos mod14 §0–§1) ─────────────────
// Só leitura, sessão municipal, papéis analyst e municipal_admin. O envelope
// tem `stale` além de { data, as_of }, e `as_of` é null quando a cidade nunca
// consolidou. A supressão é da API: o cliente nunca soma nem deriva números.
export type Cell = number | { suppressed: true };
export type Rate = number | { suppressed: true } | null;
export type AnalyticsFront = "demand" | "quality" | "calibration" | "epidemiology";
export type Granularity = "week" | "month";

export interface AnalyticsFilterEcho {
  neighborhood_id: string | null;
  health_unit_id: string | null;
  protocol_name: string | null;
  protocol_version: number | null;
}

export interface AnalyticsBase {
  front: AnalyticsFront;
  granularity?: Granularity;
  from: string;
  to: string;
  filter: AnalyticsFilterEcho;
  periods: string[];
}

export interface SeriesRow { series: Cell[]; total: Cell }

// Todas as unidades da cidade, ativas e inativas, independentes do recorte:
// a fonte do seletor de unidade em demand e quality (contratos §1).
export interface AnalyticsUnit { health_unit_id: string; name: string; active: boolean }

export interface DemandData extends AnalyticsBase {
  units: AnalyticsUnit[];
  triages: { started: Cell[]; completed: Cell[]; aborted: Cell[] };
  triages_total: { started: Cell; completed: Cell; aborted: Cell };
  by_tier: Array<SeriesRow & { tier: string }>;
  by_protocol: Array<SeriesRow & { protocol_name: string }>;
  by_neighborhood: Array<{ neighborhood_id: string | null; name: string; total: Cell }>;
  attendances_by_unit: Array<SeriesRow & { health_unit_id: string; name: string }>;
  requests_opened: Array<SeriesRow & { kind: "return" | "referral" }>;
  requests_closed: Array<SeriesRow & { reason: string }>;
}

export type WaitBucket = "0-15" | "15-30" | "30-60" | "60-120" | "120+";
export type AppointmentEndStatus = "checked_in" | "no_show" | "expired" | "cancelled_by_citizen";

export interface QualityData extends AnalyticsBase {
  units: AnalyticsUnit[];
  wait: { buckets: Array<SeriesRow & { bucket: WaitBucket }>; within_30_pct: Rate[]; within_30_pct_total: Rate };
  appointments: Array<SeriesRow & { status: AppointmentEndStatus }>;
  no_show_pct: Rate[];
  no_show_pct_total: Rate;
  attendance_outcomes: Array<SeriesRow & { outcome: AttendanceOutcome }>;
  left_pct: Rate[];
  left_pct_total: Rate;
  by_unit: Array<{
    health_unit_id: string; name: string; attendances: Cell;
    wait_within_30_pct: Rate; no_show_pct: Rate; left_pct: Rate;
  }>;
}

export type CalibrationOutcome = AttendanceOutcome | "none";
export interface CalibrationRow {
  tier: string;
  total: Cell;
  outcomes: Record<CalibrationOutcome, Cell>;
  shares: Record<CalibrationOutcome, Rate>;
}
export interface CalibrationVersion { protocol_name: string; protocol_version: number; rows: CalibrationRow[] }
export interface CalibrationData extends AnalyticsBase { versions: CalibrationVersion[] }

export interface EpiOption extends SeriesRow { value: string; label: string }
export interface EpiQuestion {
  protocol_name: string;
  question_id: string;
  prompt: string;
  answer_type: "boolean" | "enum";
  options: EpiOption[];
}
export interface EpidemiologyData extends AnalyticsBase { questions: EpiQuestion[] }

export interface AnalyticsDataMap {
  demand: DemandData;
  quality: QualityData;
  calibration: CalibrationData;
  epidemiology: EpidemiologyData;
}

export interface AnalyticsEnvelope<F extends AnalyticsFront> {
  data: AnalyticsDataMap[F];
  as_of: string | null;
  stale: boolean;
}

// Quem chama manda só os recortes da frente (contratos §1); null = sem recorte.
export interface AnalyticsQuery {
  from: string;
  to: string;
  granularity?: Granularity;
  neighborhood_id?: string | null;
  health_unit_id?: string | null;
  protocol_name?: string | null;
  protocol_version?: number | null;
}
```

- [ ] **Step 2: Escreva as fixtures**

```tsx
// src/test/analyticsFixtures.tsx
// Dados e harness dos testes do módulo 14. O fetch é trocado por um stub por
// rota (sem vi.mock do api): o cliente real monta a URL, e os testes leem os
// parâmetros que ela levou.
import type { ReactElement, ReactNode } from "react";
import { vi } from "vitest";
import { render } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type {
  AnalyticsDataMap, AnalyticsFront, CalibrationData, DemandData, EpidemiologyData, Granularity, QualityData
} from "../lib/api";

// 12:00Z = 09:00 de 30/09 em São Paulo (vitest.config.ts fixa o TZ).
export const NOW = new Date("2026-09-30T12:00:00Z");
// 05:02Z = 02:02 de 30/09 em São Paulo: a consolidação cobriu até 29/09.
export const AS_OF = "2026-09-30T05:02:11Z";
export const PERIODS = [ "2026-09-14", "2026-09-21", "2026-09-28" ];
export const HIDDEN = { suppressed: true } as const;

export const NB1 = "11111111-1111-4111-8111-111111111111";
export const NB2 = "22222222-2222-4222-8222-222222222222";
export const U1 = "aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa";
export const U2 = "bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb";
export const U3 = "cccccccc-cccc-4ccc-8ccc-cccccccccccc";

// data.units de demand e quality: todas as unidades, por nome, com a inativa.
// U3 não tem atendimento no intervalo e mesmo assim entra no seletor.
export const UNITS = [
  { health_unit_id: U3, name: "UBS Antiga", active: false },
  { health_unit_id: U1, name: "UBS Centro", active: true },
  { health_unit_id: U2, name: "UPA Boqueirão", active: true }
];

export const NEIGHBORHOODS = [
  { id: NB1, name: "Boqueirão", active: true },
  { id: NB2, name: "Xaxim", active: false }
];

// Formato de GET /admin/api/protocols: `version` é string na API.
export const PROTOCOL_ROWS = [
  { name: "arbovirose", version: "3", status: "draft" },
  { name: "arbovirose", version: "2", status: "active" },
  { name: "arbovirose", version: "1", status: "retired" },
  { name: "respiratorio", version: "4", status: "active" }
];

// Calibração não tem agrupamento (contratos §1.3): chame com `null`.
function base<F extends AnalyticsFront>(front: F, granularity: Granularity | null = "week") {
  return {
    front, ...(granularity ? { granularity } : {}), from: "2026-09-14", to: "2026-09-29",
    filter: { neighborhood_id: null, health_unit_id: null, protocol_name: null, protocol_version: null },
    periods: PERIODS
  };
}

export function demandData(overrides: Partial<DemandData> = {}): DemandData {
  return {
    ...base("demand"),
    units: UNITS,
    triages: { started: [ 12, HIDDEN, 0 ], completed: [ 10, HIDDEN, 0 ], aborted: [ HIDDEN, 0, 0 ] },
    // Somado e suprimido na API: 12 + (1..4) + 0 dá >= 13, nunca um número que a tela calcule.
    triages_total: { started: 15, completed: 13, aborted: HIDDEN },
    by_tier: [
      { tier: "vermelho", series: [ 6, HIDDEN, 0 ], total: 8 },
      { tier: "verde", series: [ HIDDEN, 0, 0 ], total: HIDDEN }
    ],
    by_protocol: [ { protocol_name: "arbovirose", series: [ 10, HIDDEN, 0 ], total: 12 } ],
    by_neighborhood: [
      { neighborhood_id: NB1, name: "Boqueirão", total: 9 },
      { neighborhood_id: null, name: "Sem bairro", total: HIDDEN }
    ],
    attendances_by_unit: [
      { health_unit_id: U1, name: "UBS Centro", series: [ 7, 5, 0 ], total: 12 },
      { health_unit_id: U2, name: "UPA Boqueirão", series: [ HIDDEN, 0, 0 ], total: HIDDEN }
    ],
    requests_opened: [
      { kind: "return", series: [ 5, 0, 0 ], total: 5 },
      { kind: "referral", series: [ 0, 0, 0 ], total: 0 }
    ],
    requests_closed: [ { reason: "fulfilled", series: [ 0, 6, 0 ], total: 6 } ],
    ...overrides
  };
}

export function qualityData(overrides: Partial<QualityData> = {}): QualityData {
  return {
    ...base("quality"),
    units: UNITS,
    wait: {
      buckets: [
        { bucket: "0-15", series: [ 8, 6, 0 ], total: 14 },
        { bucket: "15-30", series: [ HIDDEN, 5, 0 ], total: 7 },
        { bucket: "30-60", series: [ 0, 0, 0 ], total: 0 },
        { bucket: "60-120", series: [ 0, 0, 0 ], total: 0 },
        { bucket: "120+", series: [ HIDDEN, 0, 0 ], total: HIDDEN }
      ],
      within_30_pct: [ 76.9, 100, null ],
      within_30_pct_total: 87.5
    },
    appointments: [
      { status: "checked_in", series: [ 10, 9, 0 ], total: 19 },
      { status: "no_show", series: [ HIDDEN, HIDDEN, 0 ], total: 5 },
      { status: "expired", series: [ 0, 0, 0 ], total: 0 },
      { status: "cancelled_by_citizen", series: [ 0, 0, 0 ], total: 0 }
    ],
    no_show_pct: [ HIDDEN, HIDDEN, null ],
    no_show_pct_total: 20.8,
    attendance_outcomes: [
      { outcome: "discharged", series: [ 9, 8, 0 ], total: 17 },
      { outcome: "referred", series: [ 0, 5, 0 ], total: 5 },
      { outcome: "return", series: [ 0, 0, 0 ], total: 0 },
      { outcome: "left", series: [ HIDDEN, 0, 0 ], total: HIDDEN }
    ],
    left_pct: [ HIDDEN, 0, null ],
    left_pct_total: HIDDEN,
    by_unit: [
      { health_unit_id: U1, name: "UBS Centro", attendances: 22, wait_within_30_pct: 87.5, no_show_pct: 20.8, left_pct: HIDDEN },
      { health_unit_id: U2, name: "UPA Boqueirão", attendances: HIDDEN, wait_within_30_pct: HIDDEN, no_show_pct: null, left_pct: HIDDEN }
    ],
    ...overrides
  };
}

export function calibrationData(overrides: Partial<CalibrationData> = {}): CalibrationData {
  return {
    ...base("calibration", null),
    versions: [
      { protocol_name: "arbovirose", protocol_version: 2, rows: [ {
        tier: "vermelho", total: 20,
        outcomes: { discharged: 10, referred: 6, return: 0, left: HIDDEN, none: HIDDEN },
        shares: { discharged: 50, referred: 30, return: 0, left: HIDDEN, none: HIDDEN }
      } ] },
      { protocol_name: "arbovirose", protocol_version: 1, rows: [ {
        tier: "verde", total: HIDDEN,
        outcomes: { discharged: HIDDEN, referred: 0, return: 0, left: 0, none: 0 },
        shares: { discharged: HIDDEN, referred: HIDDEN, return: HIDDEN, left: HIDDEN, none: HIDDEN }
      } ] }
    ],
    ...overrides
  };
}

export function epidemiologyData(overrides: Partial<EpidemiologyData> = {}): EpidemiologyData {
  return {
    ...base("epidemiology"),
    questions: [
      { protocol_name: "arbovirose", question_id: "febre", prompt: "Teve febre?", answer_type: "boolean", options: [
        { value: "true", label: "Sim", series: [ 7, HIDDEN, 0 ], total: 9 },
        { value: "false", label: "Não", series: [ HIDDEN, 0, 0 ], total: HIDDEN }
      ] },
      { protocol_name: "arbovirose", question_id: "sintoma", prompt: "Qual o sintoma principal?", answer_type: "enum", options: [
        { value: "dor de cabeça", label: "dor de cabeça", series: [ 5, 5, 0 ], total: 10 }
      ] }
    ],
    ...overrides
  };
}

export function envelope<F extends AnalyticsFront>(
  data: AnalyticsDataMap[F], opts: { as_of?: string | null; stale?: boolean } = {}
) {
  return { data, as_of: opts.as_of === undefined ? AS_OF : opts.as_of, stale: opts.stale ?? false };
}

export interface Failure { __status: number; body: unknown }
export const failWith = (status: number, body: unknown): Failure => ({ __status: status, body });

// routes: caminho sem "/admin/api" → corpo inteiro, Failure, ou função da URL
// que devolve um dos dois. /neighborhoods e /protocols já vêm preenchidos.
export function stubAnalyticsApi(routes: Record<string, unknown>) {
  const all: Record<string, unknown> = {
    "/neighborhoods": { neighborhoods: NEIGHBORHOODS },
    "/protocols": { data: { list: PROTOCOL_ROWS }, as_of: AS_OF },
    ...routes
  };
  const fn = vi.fn(async (input: RequestInfo | URL, _init?: RequestInit) => {
    const url = new URL(String(input), "http://x");
    const entry = all[url.pathname.replace("/admin/api", "")];
    const body = typeof entry === "function" ? (entry as (u: URL) => unknown)(url) : entry;
    const headers = { "Content-Type": "application/json" };
    if (body === undefined) return new Response("", { status: 404 });
    if (body && typeof body === "object" && "__status" in body) {
      const failure = body as Failure;
      return new Response(JSON.stringify(failure.body), { status: failure.__status, headers });
    }
    return new Response(JSON.stringify(body), { status: 200, headers });
  });
  vi.stubGlobal("fetch", fn);
  return fn;
}

export function paramsOf(fn: ReturnType<typeof stubAnalyticsApi>, path: string): URLSearchParams[] {
  return fn.mock.calls
    .map(([ input ]) => new URL(String(input), "http://x"))
    .filter((url) => url.pathname === `/admin/api${path}`)
    .map((url) => url.searchParams);
}

class NoopResizeObserver { observe() {} unobserve() {} disconnect() {} }

// O provider vai como `wrapper` para o `rerender` também tê-lo.
export function renderWithQuery(ui: ReactElement) {
  vi.stubGlobal("ResizeObserver", NoopResizeObserver);
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  return { client, ...render(ui, { wrapper }) };
}

// Texto da linha (role="row" do DataTable, ou <tr>) que contém `text`.
export function rowWith(container: HTMLElement, text: string): string {
  const rows = Array.from(container.querySelectorAll('[role="row"], tr'));
  const row = rows.find((r) => r.textContent?.includes(text));
  if (!row) throw new Error(`nenhuma linha com "${text}"`);
  return row.textContent ?? "";
}
```

- [ ] **Step 3: Escreva o teste do cliente**

```ts
// src/lib/api.analytics.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { ApiError, fetchAnalytics } from "./api";
import {
  calibrationData, demandData, envelope, failWith, paramsOf, qualityData, stubAnalyticsApi
} from "../test/analyticsFixtures";

afterEach(() => vi.unstubAllGlobals());

const RANGE = { from: "2026-07-13", to: "2026-09-29" };

describe("fetchAnalytics", () => {
  it("chama /admin/api/analytics/:front só com os parâmetros preenchidos, com o cookie", async () => {
    const fn = stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    await fetchAnalytics("demand", {
      ...RANGE, granularity: "week", neighborhood_id: "none", health_unit_id: null, protocol_name: "arbovirose", protocol_version: null
    });
    const [ params ] = paramsOf(fn, "/analytics/demand");
    expect(Object.fromEntries(params)).toEqual({
      from: "2026-07-13", to: "2026-09-29", granularity: "week", neighborhood_id: "none", protocol_name: "arbovirose"
    });
    expect((fn.mock.calls[0][1] as RequestInit).credentials).toBe("include");
  });

  it("protocol_version só vai junto com protocol_name", async () => {
    const fn = stubAnalyticsApi({ "/analytics/calibration": envelope<"calibration">(calibrationData()) });
    await fetchAnalytics("calibration", { ...RANGE, protocol_version: 2 });
    await fetchAnalytics("calibration", { ...RANGE, protocol_name: "arbovirose", protocol_version: 2 });
    const [ alone, both ] = paramsOf(fn, "/analytics/calibration");
    expect(alone.has("protocol_version")).toBe(false);
    expect(both.get("protocol_version")).toBe("2");
    expect(both.has("granularity")).toBe(false);
  });

  it("devolve o envelope com as_of nulo e stale", async () => {
    stubAnalyticsApi({ "/analytics/quality": envelope<"quality">(qualityData(), { as_of: null, stale: true }) });
    const env = await fetchAnalytics("quality", { ...RANGE, granularity: "month" });
    expect(env.as_of).toBeNull();
    expect(env.stale).toBe(true);
    expect(env.data.wait.buckets).toHaveLength(5);
  });

  it("recusa vira ApiError com o código do corpo", async () => {
    stubAnalyticsApi({ "/analytics/demand": failWith(422, { error: "invalid_unit" }) });
    const err = await fetchAnalytics("demand", RANGE).catch((e: unknown) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).status).toBe(422);
    expect((err as ApiError).body).toEqual({ error: "invalid_unit" });
  });
});
```

- [ ] **Step 4: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/api.analytics.test.ts`
Expected: FAIL, `fetchAnalytics is not a function` (o import não existe ainda).

- [ ] **Step 5: Escreva o cliente**

Logo depois de `AnalyticsQuery`, no fim de `src/lib/api.ts`:

```ts
export async function fetchAnalytics<F extends AnalyticsFront>(
  front: F, query: AnalyticsQuery
): Promise<AnalyticsEnvelope<F>> {
  const params: Record<string, string | undefined> = {
    from: query.from,
    to: query.to,
    granularity: query.granularity,
    neighborhood_id: query.neighborhood_id ?? undefined,
    health_unit_id: query.health_unit_id ?? undefined,
    protocol_name: query.protocol_name ?? undefined,
    // Versão sozinha é 422 invalid_protocol: só vai junto com o nome.
    protocol_version: query.protocol_name && query.protocol_version != null ? String(query.protocol_version) : undefined
  };
  // adminFetch tipa o envelope clássico ({ data, as_of: string }); este tem
  // as_of anulável e `stale`, daí o cast.
  const envelope = await adminFetch<AnalyticsDataMap[F]>(`/analytics/${front}`, params);
  return envelope as unknown as AnalyticsEnvelope<F>;
}
```

- [ ] **Step 6: Rode, cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/api.analytics.test.ts && npx tsc --noEmit`
Expected: PASS, 4 testes; tsc sem erros.

- [ ] **Step 7: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/lib/api.ts src/test/analyticsFixtures.tsx src/lib/api.analytics.test.ts
/opt/homebrew/bin/git commit -m "feat: add the analytics API client and contract types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras do Analytics — célula, taxa, carimbo, intervalo, recortes e recusas

**Files:**
- Create: `src/lib/analytics.ts`
- Test: `src/lib/analytics.test.ts`

**Interfaces:**
- Consumes:
  - da Task 1: `Cell`, `Rate`, `Granularity`, `AnalyticsFront`, `AnalyticsQuery`, `CalibrationOutcome`;
  - já existentes: `ApiError`, `AuthorProtocolRow` (`src/lib/api.ts`), `HIDDEN_LABEL` e `isSuppressed` (`src/lib/smallCount.ts`), `fmtNumber` (`src/lib/format.ts`), `todayInCity` e `addDays` (`src/lib/campaigns.ts`).
- Produces (em `src/lib/analytics.ts`):
  - textos: `HIDDEN_LABEL` (reexportado, `"oculto"`), `NO_DATA_LABEL = "sem dado"`, `HIDDEN_HINT`, `NO_DATA_HINT`, `STALE_LABEL = "dados desatualizados"`, `EMPTY_TITLE = "ainda sem dados consolidados"`, `EMPTY_SUB`, `EPI_EMPTY_TITLE`, `EPI_HOWTO`, `FORBIDDEN_TEXT`;
  - `analyticsKey(front, query) → readonly ["analytics", AnalyticsFront, Record<string, unknown>]` e `ANALYTICS_PROTOCOLS_KEY`;
  - `fmtCell(v: Cell | undefined): string`, `fmtRate(v: Rate | undefined): string`, `plotValue(v: Cell | Rate | undefined): number | null`;
  - `dataUntil(asOf: string | null | undefined): string | null` (`"DD/MM"`), `fmtPeriod(start: string, g: Granularity): string`;
  - `AnalyticsRange { granularity; count }`, `RANGE_OPTIONS`, `DEFAULT_RANGE`, `GRANULARITY_LABEL`, `rangeLabel(g, n)`, `withGranularity(range, g)`, `mondayOf(date)`, `rangeDates(range, today?) → { from; to }`;
  - `ProtocolOption { name; versions: number[] }`, `protocolOptions(rows: AuthorProtocolRow[])`;
  - rótulos `KIND_LABEL`, `CLOSED_REASON_LABEL`, `WAIT_BUCKET_LABEL`, `APPOINTMENT_LABEL`, `OUTCOME_LABEL`, `CALIBRATION_OUTCOMES`, `labelOr(map, key)`;
  - `analyticsError(err: unknown): string`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/analytics.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { ApiError } from "./api";
import {
  DEFAULT_RANGE, FORBIDDEN_TEXT, analyticsError, analyticsKey, dataUntil, fmtCell, fmtPeriod, fmtRate, mondayOf,
  plotValue, protocolOptions, rangeDates, rangeLabel, withGranularity
} from "./analytics";
import { AS_OF, HIDDEN, NOW, PROTOCOL_ROWS } from "../test/analyticsFixtures";

describe("célula e taxa", () => {
  it("contagem: número com milhar, zero e oculto", () => {
    expect(fmtCell(1234)).toBe("1.234");
    expect(fmtCell(0)).toBe("0");
    expect(fmtCell(HIDDEN)).toBe("oculto");
    expect(fmtCell(undefined)).toBe("—");
  });

  it("taxa: uma casa decimal com vírgula, oculto e sem dado", () => {
    expect(fmtRate(87.5)).toBe("87,5%");
    expect(fmtRate(100)).toBe("100,0%");
    expect(fmtRate(0)).toBe("0,0%");
    expect(fmtRate(HIDDEN)).toBe("oculto");
    expect(fmtRate(null)).toBe("sem dado");
  });

  it("no gráfico, oculto e sem dado viram lacuna, nunca zero", () => {
    expect([ 12, HIDDEN, null, 0, undefined ].map(plotValue)).toEqual([ 12, null, null, 0, null ]);
  });
});

describe("carimbo e rótulo de período", () => {
  it("dados até = dia da cidade do as_of, menos 1", () => {
    expect(dataUntil(AS_OF)).toBe("29/09");
  });

  it("as_of às 23h30 em São Paulo (dia seguinte em UTC) não adianta o carimbo", () => {
    // 02:30Z de 30/09 = 23:30 de 29/09 em São Paulo: consolidou até 28/09.
    expect(dataUntil("2026-09-30T02:30:00Z")).toBe("28/09");
  });

  it("sem as_of, ou lixo, não há carimbo", () => {
    expect(dataUntil(null)).toBeNull();
    expect(dataUntil("ontem")).toBeNull();
  });

  it("semana vira DD/MM e mês vira MM/AAAA, sem deslocar o dia", () => {
    expect(fmtPeriod("2026-09-28", "week")).toBe("28/09");
    expect(fmtPeriod("2026-09-01", "month")).toBe("09/2026");
  });
});

describe("intervalo", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(NOW);
  });
  afterEach(() => vi.useRealTimers());

  it("semanas: termina ontem e começa numa segunda-feira", () => {
    expect(rangeDates(DEFAULT_RANGE)).toEqual({ from: "2026-07-13", to: "2026-09-29" });
    expect(rangeDates({ granularity: "week", count: 26 })).toEqual({ from: "2026-04-06", to: "2026-09-29" });
    expect(rangeDates({ granularity: "week", count: 104 })).toEqual({ from: "2024-10-07", to: "2026-09-29" });
  });

  it("meses: começa no dia 1", () => {
    expect(rangeDates({ granularity: "month", count: 6 })).toEqual({ from: "2026-04-01", to: "2026-09-29" });
    expect(rangeDates({ granularity: "month", count: 60 })).toEqual({ from: "2021-10-01", to: "2026-09-29" });
  });

  it("às 23h30 em São Paulo, ontem ainda é o dia anterior da cidade", () => {
    vi.setSystemTime(new Date("2026-10-01T02:30:00Z")); // 23:30 de 30/09 em São Paulo
    expect(rangeDates(DEFAULT_RANGE).to).toBe("2026-09-29");
  });

  it("segunda-feira da semana", () => {
    expect(mondayOf("2026-09-29")).toBe("2026-09-28");
    expect(mondayOf("2026-09-28")).toBe("2026-09-28");
    expect(mondayOf("2026-10-04")).toBe("2026-09-28");
  });

  it("trocar o agrupamento mantém a contagem se ela existe na outra lista", () => {
    expect(withGranularity({ granularity: "week", count: 12 }, "month")).toEqual({ granularity: "month", count: 12 });
    expect(withGranularity({ granularity: "week", count: 26 }, "month")).toEqual({ granularity: "month", count: 6 });
    expect(withGranularity({ granularity: "month", count: 60 }, "week")).toEqual({ granularity: "week", count: 12 });
    expect(rangeLabel("week", 12)).toBe("últimas 12 semanas");
    expect(rangeLabel("month", 6)).toBe("últimos 6 meses");
  });
});

describe("chave e opções", () => {
  it("recorte nulo ou ausente dá a mesma chave de cache", () => {
    expect(analyticsKey("demand", { from: "a", to: "b", health_unit_id: null, neighborhood_id: undefined }))
      .toEqual(analyticsKey("demand", { from: "a", to: "b" }));
  });

  it("protocolos: sem rascunho, versões numéricas da maior para a menor, nomes em ordem", () => {
    expect(protocolOptions(PROTOCOL_ROWS)).toEqual([
      { name: "arbovirose", versions: [ 2, 1 ] },
      { name: "respiratorio", versions: [ 4 ] }
    ]);
  });
});

describe("recusas", () => {
  it("403 diz de quem é o Analytics", () => {
    expect(analyticsError(new ApiError(403, { error: "forbidden_role" }, "x"))).toBe(FORBIDDEN_TEXT);
  });

  it("422 traduz cada código; o resto é a frase genérica", () => {
    const e = (code: string) => analyticsError(new ApiError(422, { error: code }, "x"));
    expect(e("invalid_range")).toBe("Período inválido — escolha outro intervalo.");
    expect(e("invalid_neighborhood")).toBe("Esse bairro não existe nesta cidade — escolha outro.");
    expect(e("invalid_unit")).toBe("Essa unidade não existe nesta cidade — escolha outra.");
    expect(e("invalid_protocol")).toBe("Protocolo ou versão não encontrado — escolha outro.");
    expect(e("http_500")).toBe("não foi possível carregar — tente de novo");
    expect(analyticsError(new Error("rede"))).toBe("não foi possível carregar — tente de novo");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/analytics.test.ts`
Expected: FAIL, `Failed to resolve import "./analytics"`.

- [ ] **Step 3: Escreva `src/lib/analytics.ts`**

```ts
// Regras puras do Analytics (módulo 14; ADR 0025; contratos mod14 §0–§1).
// Sem React: texto de célula e taxa, carimbo "dados até", intervalos, opções
// dos recortes e tradução das recusas. A supressão é da API; aqui nada é
// somado nem derivado — oculto e sem dado nunca viram zero.
import {
  ApiError, type AnalyticsFront, type AnalyticsQuery, type AuthorProtocolRow, type CalibrationOutcome, type Cell,
  type Granularity, type Rate
} from "./api";
import { HIDDEN_LABEL, isSuppressed } from "./smallCount";
import { fmtNumber } from "./format";
import { addDays, todayInCity } from "./campaigns";

// ─── Textos (spec §8, literais) ─────────────────────────────────────────────
export { HIDDEN_LABEL };
export const NO_DATA_LABEL = "sem dado";
export const HIDDEN_HINT =
  "Contagens de 1 a 4, e taxas calculadas sobre elas, aparecem como “oculto” para não identificar ninguém.";
export const NO_DATA_HINT = "Nada no denominador neste período: não há taxa a mostrar.";
export const STALE_LABEL = "dados desatualizados";
export const EMPTY_TITLE = "ainda sem dados consolidados";
export const EMPTY_SUB = "a consolidação roda toda madrugada e cobre até o dia anterior";
export const EPI_EMPTY_TITLE = "nenhuma pergunta marcada para Analytics";
export const EPI_HOWTO =
  "Para uma pergunta aparecer aqui, abra o protocolo no Editor de protocolo, marque “Usar em Analytics” " +
  "numa pergunta de sim/não ou de lista, salve o rascunho e leve a nova versão pelo ciclo de assinaturas " +
  "até a ativação. Só entram as triagens concluídas com essa versão.";
export const FORBIDDEN_TEXT =
  "O Analytics é do papel Análise e do administrador municipal. Peça o acesso a quem administra a equipe.";

// ─── Cache ──────────────────────────────────────────────────────────────────
export const ANALYTICS_PROTOCOLS_KEY = [ "analyticsProtocols" ] as const;

// Recorte nulo e ausente são o mesmo pedido: a chave os iguala, e trocar de
// "Todas" para uma unidade e de volta reaproveita o cache.
export function analyticsKey(front: AnalyticsFront, query: AnalyticsQuery) {
  const compact: Record<string, unknown> = Object.fromEntries(
    Object.entries(query).filter(([ , v ]) => v !== null && v !== undefined && v !== ""));
  return [ "analytics", front, compact ] as const;
}

// ─── Célula e taxa ──────────────────────────────────────────────────────────
const rateFmt = new Intl.NumberFormat("pt-BR", { minimumFractionDigits: 1, maximumFractionDigits: 1 });

export function fmtCell(value: Cell | undefined): string {
  if (isSuppressed(value)) return HIDDEN_LABEL;
  return typeof value === "number" ? fmtNumber(value) : "—";
}

export function fmtRate(value: Rate | undefined): string {
  if (value === null) return NO_DATA_LABEL;
  if (isSuppressed(value)) return HIDDEN_LABEL;
  return typeof value === "number" ? `${rateFmt.format(value)}%` : "—";
}

// Ponto oculto ou sem dado vira lacuna: desenhar 0 seria afirmar um número.
export function plotValue(value: Cell | Rate | undefined): number | null {
  return typeof value === "number" ? value : null;
}

// ─── Carimbo e períodos ─────────────────────────────────────────────────────
// A consolidação que termina no dia D (fuso da cidade) cobre até D-1 (spec §4.1).
export function dataUntil(asOf: string | null | undefined): string | null {
  if (!asOf) return null;
  const at = new Date(asOf);
  if (Number.isNaN(at.getTime())) return null;
  const day = addDays(todayInCity(at), -1);
  return `${day.slice(8, 10)}/${day.slice(5, 7)}`;
}

// Início de período ISO → rótulo. Lido como texto: `new Date("2026-09-28")`
// seria meia-noite UTC e voltaria um dia em São Paulo.
export function fmtPeriod(start: string, granularity: Granularity): string {
  const m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(start);
  if (!m) return start;
  return granularity === "month" ? `${m[2]}/${m[1]}` : `${m[3]}/${m[2]}`;
}

export interface AnalyticsRange { granularity: Granularity; count: number }

// Limites da API: 104 semanas, 60 meses (contratos §1).
export const RANGE_OPTIONS: Record<Granularity, number[]> = { week: [ 12, 26, 52, 104 ], month: [ 6, 12, 24, 60 ] };
export const DEFAULT_RANGE: AnalyticsRange = { granularity: "week", count: 12 };
export const GRANULARITY_LABEL: Record<Granularity, string> = { week: "Semana", month: "Mês" };

export function rangeLabel(granularity: Granularity, count: number): string {
  return granularity === "week" ? `últimas ${count} semanas` : `últimos ${count} meses`;
}

export function withGranularity(range: AnalyticsRange, granularity: Granularity): AnalyticsRange {
  const options = RANGE_OPTIONS[granularity];
  return { granularity, count: options.includes(range.count) ? range.count : options[0] };
}

export function mondayOf(date: string): string {
  const weekday = new Date(`${date}T12:00:00Z`).getUTCDay(); // 0 = domingo
  return addDays(date, -((weekday + 6) % 7));
}

// Termina ontem (o dia corrente nunca é consolidado) e começa no início do
// primeiro período, para a API contar exatamente `count` períodos.
export function rangeDates(range: AnalyticsRange, today: string = todayInCity()): { from: string; to: string } {
  const to = addDays(today, -1);
  if (range.granularity === "week") {
    return { from: addDays(mondayOf(to), -7 * (range.count - 1)), to };
  }
  const [ year, month ] = to.split("-").map(Number);
  const index = year * 12 + (month - 1) - (range.count - 1);
  const from = `${Math.floor(index / 12)}-${String((index % 12) + 1).padStart(2, "0")}-01`;
  return { from, to };
}

// ─── Opções dos recortes ────────────────────────────────────────────────────
export interface ProtocolOption { name: string; versions: number[] }

// GET /admin/api/protocols manda `version` como string; rascunho nunca teve triagem.
export function protocolOptions(rows: AuthorProtocolRow[]): ProtocolOption[] {
  const byName = new Map<string, Set<number>>();
  for (const row of rows) {
    if (row.status === "draft") continue;
    const version = Number(row.version);
    if (!Number.isInteger(version)) continue;
    byName.set(row.name, (byName.get(row.name) ?? new Set<number>()).add(version));
  }
  return [ ...byName.entries() ]
    .map(([ name, versions ]) => ({ name, versions: [ ...versions ].sort((a, b) => b - a) }))
    .sort((a, b) => a.name.localeCompare(b.name, "pt-BR"));
}

// ─── Rótulos ────────────────────────────────────────────────────────────────
export const KIND_LABEL: Record<string, string> = { return: "Retorno", referral: "Encaminhamento" };
export const CLOSED_REASON_LABEL: Record<string, string> = {
  fulfilled: "Atendido", citizen_cancelled: "Cancelado pelo cidadão", dismissed: "Dispensado"
};
export const WAIT_BUCKET_LABEL: Record<string, string> = {
  "0-15": "até 15 min", "15-30": "15 a 30 min", "30-60": "30 a 60 min", "60-120": "1 a 2 h", "120+": "mais de 2 h"
};
export const APPOINTMENT_LABEL: Record<string, string> = {
  checked_in: "Compareceu", no_show: "Faltou", expired: "Expirou", cancelled_by_citizen: "Cancelado pelo cidadão"
};
export const OUTCOME_LABEL: Record<CalibrationOutcome, string> = {
  discharged: "Atendido e liberado", referred: "Encaminhado", return: "Retorno", left: "Saiu sem atendimento",
  none: "Sem atendimento encerrado"
};
export const CALIBRATION_OUTCOMES: CalibrationOutcome[] = [ "discharged", "referred", "return", "left", "none" ];

// Valor novo da API sem rótulo ainda aparece cru, em vez de sumir.
export function labelOr(map: Record<string, string>, key: string): string {
  return map[key] ?? key;
}

// ─── Recusas ────────────────────────────────────────────────────────────────
const GENERIC = "não foi possível carregar — tente de novo";
const REFUSAL_TEXT: Record<string, string> = {
  invalid_range: "Período inválido — escolha outro intervalo.",
  invalid_neighborhood: "Esse bairro não existe nesta cidade — escolha outro.",
  invalid_unit: "Essa unidade não existe nesta cidade — escolha outra.",
  invalid_protocol: "Protocolo ou versão não encontrado — escolha outro."
};

export function analyticsError(err: unknown): string {
  if (!(err instanceof ApiError)) return GENERIC;
  if (err.status === 403) return FORBIDDEN_TEXT;
  if (err.status === 401) return "sessão expirada — entre de novo";
  const code = err.body && typeof err.body === "object" ? (err.body as { error?: unknown }).error : undefined;
  if (err.status === 422 && typeof code === "string" && REFUSAL_TEXT[code]) return REFUSAL_TEXT[code];
  return GENERIC;
}
```

- [ ] **Step 4: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/analytics.test.ts && npx tsc --noEmit`
Expected: PASS (todos os testes do arquivo); tsc sem erros.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/lib/analytics.ts src/lib/analytics.test.ts
/opt/homebrew/bin/git commit -m "feat: add analytics formatting, ranges and filter options

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Peças comuns — valor, carimbo, gráfico, bloco de série e moldura

**Files:**
- Create: `src/hooks/useAnalytics.ts`, `src/modules/analytics/values.tsx`, `src/modules/analytics/DataStamp.tsx`, `src/modules/analytics/SeriesChart.tsx`, `src/modules/analytics/SeriesBlock.tsx`, `src/modules/analytics/AnalyticsView.tsx`
- Test: `src/modules/analytics/components.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `fetchAnalytics`, `AnalyticsFront`, `AnalyticsQuery`, `AnalyticsEnvelope`, `AnalyticsDataMap`, `Cell`, `Rate`, `Granularity`;
  - da Task 2: `analyticsKey`, `fmtCell`, `fmtRate`, `plotValue`, `fmtPeriod`, `dataUntil`, `analyticsError`, `HIDDEN_HINT`, `NO_DATA_HINT`, `STALE_LABEL`, `EMPTY_TITLE`, `EMPTY_SUB`;
  - já existentes: `Panel`, `DataTable`, `EmptyState`, `ErrorState`, `Skeleton`, `Tag`, `fmtDateTime`, `isSuppressed`.
- Produces:
  - `useAnalytics<F>(front: F, query: AnalyticsQuery): UseQueryResult<AnalyticsEnvelope<F>, Error>`;
  - `CellValue({ value })`, `RateValue({ value })`, `Value({ kind: "count" | "rate"; value })`;
  - `DataStamp({ asOf: string | null; stale: boolean })`;
  - `ChartLine { key; label; series: Array<Cell | Rate> }`, `chartRows(periods, granularity, lines)`, `SeriesChart({ label; periods; granularity; lines; kind })`;
  - `SeriesLine extends ChartLine { total?: Cell | Rate }`, `SeriesBlock({ title; sub?; kind; periods; granularity; lines; empty? })` — um `Panel` (região com o nome `title`) com gráfico, grupo "Totais do período" (só se alguma linha tem `total`) e a tabela `"<title> por período"` dentro de `<details>`;
  - `AnalyticsView<F>({ result; children(data: AnalyticsDataMap[F]): ReactNode })`.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/analytics/components.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, within } from "@testing-library/react";
import { useAnalytics } from "../../hooks/useAnalytics";
import { AnalyticsView } from "./AnalyticsView";
import { DataStamp } from "./DataStamp";
import { SeriesBlock } from "./SeriesBlock";
import { chartRows } from "./SeriesChart";
import { CellValue, RateValue } from "./values";
import { FORBIDDEN_TEXT, HIDDEN_HINT } from "../../lib/analytics";
import {
  AS_OF, HIDDEN, PERIODS, demandData, envelope, failWith, renderWithQuery, rowWith, stubAnalyticsApi
} from "../../test/analyticsFixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

const QUERY = { from: "2026-07-13", to: "2026-09-29", granularity: "week" as const };

function Probe() {
  const result = useAnalytics("demand", QUERY);
  return <AnalyticsView result={result}>{(data) => <p>{data.periods.length} períodos</p>}</AnalyticsView>;
}

describe("valores", () => {
  it("contagem oculta mostra 'oculto' com a dica", () => {
    render(<CellValue value={HIDDEN} />);
    expect(screen.getByText("oculto").getAttribute("title")).toBe(HIDDEN_HINT);
  });

  it("taxa sem denominador mostra 'sem dado'; taxa com número tem uma casa", () => {
    render(<p><RateValue value={null} /> · <RateValue value={20.8} /></p>);
    expect(screen.getByText("sem dado")).toBeTruthy();
    expect(screen.getByText(/20,8%/)).toBeTruthy();
  });
});

describe("DataStamp", () => {
  it("mostra 'dados até' sem aviso quando está em dia", () => {
    render(<DataStamp asOf={AS_OF} stale={false} />);
    expect(screen.getByText("dados até 29/09")).toBeTruthy();
    expect(screen.queryByRole("status")).toBeNull();
  });

  it("stale acende 'dados desatualizados'", () => {
    render(<DataStamp asOf={AS_OF} stale />);
    expect(screen.getByRole("status").textContent).toBe("dados desatualizados");
  });

  it("sem as_of não mostra nada", () => {
    const { container } = render(<DataStamp asOf={null} stale />);
    expect(container.textContent).toBe("");
  });
});

describe("AnalyticsView", () => {
  it("com dados: carimbo e conteúdo", async () => {
    stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    renderWithQuery(<Probe />);
    expect(await screen.findByText("3 períodos")).toBeTruthy();
    expect(screen.getByText("dados até 29/09")).toBeTruthy();
  });

  it("as_of nulo: estado vazio, sem carimbo e sem conteúdo (nenhum zero na tela)", async () => {
    stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData(), { as_of: null, stale: true }) });
    renderWithQuery(<Probe />);
    expect(await screen.findByText("ainda sem dados consolidados")).toBeTruthy();
    expect(screen.queryByText("3 períodos")).toBeNull();
    expect(screen.queryByText(/dados até/)).toBeNull();
  });

  it("stale com dados: carimbo e aviso", async () => {
    stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData(), { stale: true }) });
    renderWithQuery(<Probe />);
    expect((await screen.findByRole("status")).textContent).toBe("dados desatualizados");
    expect(screen.getByText("3 períodos")).toBeTruthy();
  });

  it("papel revogado no meio da sessão: 403 vira a frase de acesso", async () => {
    stubAnalyticsApi({ "/analytics/demand": failWith(403, { error: "forbidden_role" }) });
    renderWithQuery(<Probe />);
    expect(await screen.findByText(FORBIDDEN_TEXT)).toBeTruthy();
    expect(screen.getByRole("button", { name: "tentar novamente" })).toBeTruthy();
  });

  it("422 invalid_range vira frase, não código", async () => {
    stubAnalyticsApi({ "/analytics/demand": failWith(422, { error: "invalid_range" }) });
    renderWithQuery(<Probe />);
    expect(await screen.findByText("Período inválido — escolha outro intervalo.")).toBeTruthy();
    expect(screen.queryByText(/invalid_range/)).toBeNull();
  });
});

describe("gráfico e bloco de série", () => {
  it("no gráfico, oculto e sem dado viram lacuna com o rótulo do período", () => {
    const rows = chartRows(PERIODS, "week", [
      { key: "a", label: "A", series: [ 12, HIDDEN, 0 ] },
      { key: "b", label: "B", series: [ 50, null, 100 ] }
    ]);
    expect(rows).toEqual([
      { period: "14/09", s0: 12, s1: 50 },
      { period: "21/09", s0: null, s1: null },
      { period: "28/09", s0: 0, s1: 100 }
    ]);
  });

  it("com total: tabela de totais e tabela por período, 'oculto' nas duas", () => {
    renderWithQuery(
      <SeriesBlock title="Por tier" kind="count" periods={PERIODS} granularity="week" lines={[
        { key: "vermelho", label: "vermelho", series: [ 6, HIDDEN, 0 ], total: 8 },
        { key: "verde", label: "verde", series: [ HIDDEN, 0, 0 ], total: HIDDEN }
      ]} />
    );
    const block = within(screen.getByRole("region", { name: "Por tier" }));
    const totals = block.getByRole("group", { name: "Totais do período" });
    expect(rowWith(totals, "vermelho")).toContain("8");
    expect(rowWith(totals, "verde")).toContain("oculto");
    const byPeriod = block.getByRole("table", { name: "Por tier por período" });
    expect(rowWith(byPeriod, "21/09")).toContain("oculto");
    expect(block.getByRole("list", { name: "legenda" }).textContent).toContain("vermelho");
  });

  it("sem total na linha, sem tabela de totais (o cliente não soma)", () => {
    renderWithQuery(
      <SeriesBlock title="Triagens" kind="count" periods={PERIODS} granularity="week"
        lines={[ { key: "started", label: "Iniciadas", series: [ 12, HIDDEN, 0 ] } ]} />
    );
    const block = within(screen.getByRole("region", { name: "Triagens" }));
    expect(block.queryByRole("group", { name: "Totais do período" })).toBeNull();
  });

  it("taxa: 'sem dado' no período sem denominador", () => {
    renderWithQuery(
      <SeriesBlock title="Faltas" kind="rate" periods={PERIODS} granularity="week"
        lines={[ { key: "no_show", label: "faltas", series: [ HIDDEN, 25, null ], total: 20.8 } ]} />
    );
    const block = within(screen.getByRole("region", { name: "Faltas" }));
    expect(rowWith(block.getByRole("group", { name: "Totais do período" }), "faltas")).toContain("20,8%");
    expect(rowWith(block.getByRole("table", { name: "Faltas por período" }), "28/09")).toContain("sem dado");
  });

  it("sem linhas: estado vazio do bloco", () => {
    renderWithQuery(<SeriesBlock title="Pedidos" kind="count" periods={PERIODS} granularity="week" lines={[]} />);
    expect(within(screen.getByRole("region", { name: "Pedidos" })).getByText("sem registros no período")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/components.test.tsx`
Expected: FAIL, `Failed to resolve import "../../hooks/useAnalytics"`.

- [ ] **Step 3: Escreva o hook**

```ts
// src/hooks/useAnalytics.ts
// Uma consulta por frente e recorte (módulo 14). O cache do TanStack já é por
// usuário (useSessionQueryClient); a chave leva só frente e recorte.
import { useQuery } from "@tanstack/react-query";
import { fetchAnalytics, type AnalyticsFront, type AnalyticsQuery } from "../lib/api";
import { analyticsKey } from "../lib/analytics";

export function useAnalytics<F extends AnalyticsFront>(front: F, query: AnalyticsQuery) {
  return useQuery({
    queryKey: analyticsKey(front, query),
    queryFn: () => fetchAnalytics(front, query),
    // D-1: o dado só muda de madrugada; 5 min evita refazer a leitura a cada aba.
    staleTime: 5 * 60_000
  });
}
```

- [ ] **Step 4: Escreva os valores e o carimbo**

```tsx
// src/modules/analytics/values.tsx
// Célula e taxa do Analytics (spec §8): "oculto" com a dica, "sem dado" para
// taxa sem denominador, percentual com uma casa.
import type { Cell, Rate } from "../../lib/api";
import { HIDDEN_HINT, NO_DATA_HINT, fmtCell, fmtRate } from "../../lib/analytics";
import { isSuppressed } from "../../lib/smallCount";

export function CellValue({ value }: { value: Cell | undefined }) {
  if (isSuppressed(value)) return <span title={HIDDEN_HINT}>{fmtCell(value)}</span>;
  return <>{fmtCell(value)}</>;
}

export function RateValue({ value }: { value: Rate | undefined }) {
  if (isSuppressed(value)) return <span title={HIDDEN_HINT}>{fmtRate(value)}</span>;
  if (value === null) return <span title={NO_DATA_HINT}>{fmtRate(value)}</span>;
  return <>{fmtRate(value)}</>;
}

export function Value({ kind, value }: { kind: "count" | "rate"; value: Cell | Rate | undefined }) {
  return kind === "rate" ? <RateValue value={value as Rate | undefined} /> : <CellValue value={value as Cell | undefined} />;
}
```

```tsx
// src/modules/analytics/DataStamp.tsx
// Carimbo do Analytics (spec §8): "dados até DD/MM" e, com `stale`, o aviso.
import { STALE_LABEL, dataUntil } from "../../lib/analytics";
import { fmtDateTime } from "../../lib/format";
import { Tag } from "../../components/Tag";

export function DataStamp({ asOf, stale }: { asOf: string | null; stale: boolean }) {
  const until = dataUntil(asOf);
  if (!until) return null;
  return (
    <div className="mono" style={{ display: "flex", gap: 8, alignItems: "center", fontSize: 11, color: "var(--ink3)" }}>
      <span title={`consolidado em ${fmtDateTime(asOf)}`}>dados até {until}</span>
      {stale && <span role="status"><Tag tone="warn">{STALE_LABEL}</Tag></span>}
    </div>
  );
}
```

- [ ] **Step 5: Escreva o gráfico e o bloco de série**

```tsx
// src/modules/analytics/SeriesChart.tsx
// Série do Analytics em linhas (Recharts, já dependência). Ponto oculto ou sem
// dado é lacuna (connectNulls={false}); a legenda é HTML, legível sem o SVG.
import { CartesianGrid, Line, LineChart, ResponsiveContainer, Tooltip, XAxis, YAxis } from "recharts";
import type { Cell, Granularity, Rate } from "../../lib/api";
import { fmtPeriod, plotValue } from "../../lib/analytics";

export interface ChartLine { key: string; label: string; series: Array<Cell | Rate> }

export const PALETTE = [ "var(--accent)", "var(--warn)", "var(--ok)", "var(--down)", "var(--info)", "var(--ink3)" ];

export function chartRows(periods: string[], granularity: Granularity, lines: ChartLine[]) {
  return periods.map((period, i) => {
    const row: Record<string, string | number | null> = { period: fmtPeriod(period, granularity) };
    lines.forEach((line, j) => { row[`s${j}`] = plotValue(line.series[i]); });
    return row;
  });
}

interface Props { label: string; periods: string[]; granularity: Granularity; lines: ChartLine[]; kind: "count" | "rate" }

export function SeriesChart({ label, periods, granularity, lines, kind }: Props) {
  const data = chartRows(periods, granularity, lines);
  const hasGaps = lines.some((line) => line.series.some((v) => typeof v !== "number"));
  return (
    <figure aria-label={label} style={{ margin: 0 }}>
      <div style={{ width: "100%", height: 180 }}>
        <ResponsiveContainer width="100%" height="100%">
          <LineChart data={data} margin={{ top: 8, right: 8, bottom: 0, left: 0 }}>
            <CartesianGrid stroke="var(--rule)" vertical={false} />
            <XAxis dataKey="period" tick={{ fontSize: 10 }} />
            <YAxis tick={{ fontSize: 10 }} width={40} allowDecimals={kind === "rate"}
              domain={kind === "rate" ? [ 0, 100 ] : [ 0, "auto" ]} />
            <Tooltip />
            {lines.map((line, j) => (
              <Line key={line.key} dataKey={`s${j}`} name={line.label} stroke={PALETTE[j % PALETTE.length]}
                dot={false} connectNulls={false} isAnimationActive={false} />
            ))}
          </LineChart>
        </ResponsiveContainer>
      </div>
      <figcaption style={{ display: "flex", gap: 12, flexWrap: "wrap", alignItems: "center", fontSize: 11, color: "var(--ink2)" }}>
        <ul aria-label="legenda" style={{ display: "flex", gap: 12, flexWrap: "wrap", listStyle: "none", margin: 0, padding: 0 }}>
          {lines.map((line, j) => (
            <li key={line.key} style={{ display: "inline-flex", alignItems: "center", gap: 4 }}>
              <span aria-hidden="true" style={{ width: 10, height: 2, background: PALETTE[j % PALETTE.length] }} />
              {line.label}
            </li>
          ))}
        </ul>
        {hasGaps && <span style={{ color: "var(--ink3)" }}>lacuna = oculto ou sem dado</span>}
      </figcaption>
    </figure>
  );
}
```

```tsx
// src/modules/analytics/SeriesBlock.tsx
// Bloco de uma métrica (spec §8: "gráfico de série e tabela"): gráfico, totais
// do período inteiro (quando a API manda) e a tabela por período. Nenhum total
// é calculado aqui.
import type { Cell, Granularity, Rate } from "../../lib/api";
import { fmtPeriod } from "../../lib/analytics";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { SeriesChart, type ChartLine } from "./SeriesChart";
import { Value } from "./values";

export interface SeriesLine extends ChartLine { total?: Cell | Rate }

interface Props {
  title: string;
  sub?: string;
  kind: "count" | "rate";
  periods: string[];
  granularity: Granularity;
  lines: SeriesLine[];
  empty?: string;
}

export function SeriesBlock({ title, sub, kind, periods, granularity, lines, empty = "sem registros no período" }: Props) {
  const hasTotals = lines.some((line) => line.total !== undefined);
  return (
    <Panel title={title} sub={sub}>
      {lines.length === 0 ? <EmptyState title={empty} /> : (
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          <SeriesChart label={`${title} ao longo do tempo`} periods={periods} granularity={granularity} lines={lines} kind={kind} />
          {hasTotals && (
            <div role="group" aria-label="Totais do período">
              <DataTable<SeriesLine>
                cols={[
                  { label: "Série", w: "2fr", render: (line) => line.label },
                  { label: "Período inteiro", w: "1fr", align: "right", render: (line) => (
                    <span className="mono"><Value kind={kind} value={line.total} /></span>
                  ) }
                ]}
                rows={lines}
                rowKey={(line) => line.key}
              />
            </div>
          )}
          <details>
            <summary style={{ fontSize: 12, cursor: "pointer", color: "var(--ink2)" }}>ver por período</summary>
            <PeriodTable title={title} kind={kind} periods={periods} granularity={granularity} lines={lines} />
          </details>
        </div>
      )}
    </Panel>
  );
}

function PeriodTable({ title, kind, periods, granularity, lines }: Omit<Props, "sub" | "empty">) {
  return (
    <div style={{ overflowX: "auto" }}>
      <table aria-label={`${title} por período`} className="mono" style={{ width: "100%", borderCollapse: "collapse", fontSize: 11.5 }}>
        <thead>
          <tr>
            <th scope="col" style={headStyle}>Período</th>
            {lines.map((line) => <th key={line.key} scope="col" style={{ ...headStyle, textAlign: "right" }}>{line.label}</th>)}
          </tr>
        </thead>
        <tbody>
          {periods.map((period, i) => (
            <tr key={period}>
              <th scope="row" style={cellStyle}>{fmtPeriod(period, granularity)}</th>
              {lines.map((line) => (
                <td key={line.key} style={{ ...cellStyle, textAlign: "right" }}><Value kind={kind} value={line.series[i]} /></td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

const headStyle = { padding: "4px 8px", borderBottom: "1px solid var(--rule)", color: "var(--ink3)", fontWeight: 500, textAlign: "left" as const };
const cellStyle = { padding: "4px 8px", borderBottom: "1px solid var(--rule)", fontWeight: 400, textAlign: "left" as const };
```

- [ ] **Step 6: Escreva a moldura**

```tsx
// src/modules/analytics/AnalyticsView.tsx
// Moldura de toda aba: carregando, erro traduzido, vazio (nunca consolidou) e,
// com dado, o carimbo antes do conteúdo. `as_of` nulo nunca mostra zeros.
import type { ReactNode } from "react";
import type { UseQueryResult } from "@tanstack/react-query";
import type { AnalyticsDataMap, AnalyticsEnvelope, AnalyticsFront } from "../../lib/api";
import { EMPTY_SUB, EMPTY_TITLE, analyticsError } from "../../lib/analytics";
import { Panel } from "../../components/Panel";
import { EmptyState } from "../../components/EmptyState";
import { ErrorState } from "../../components/ErrorState";
import { Skeleton } from "../../components/Skeleton";
import { DataStamp } from "./DataStamp";

interface Props<F extends AnalyticsFront> {
  result: UseQueryResult<AnalyticsEnvelope<F>, Error>;
  children(data: AnalyticsDataMap[F]): ReactNode;
}

export function AnalyticsView<F extends AnalyticsFront>({ result, children }: Props<F>) {
  if (result.isPending) return <Skeleton rows={6} />;
  if (result.isError) return <ErrorState message={analyticsError(result.error)} onRetry={() => void result.refetch()} />;
  const { as_of: asOf, stale, data } = result.data;
  if (!asOf) {
    return <Panel title="Analytics"><EmptyState title={EMPTY_TITLE} sub={EMPTY_SUB} /></Panel>;
  }
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <DataStamp asOf={asOf} stale={stale} />
      {children(data)}
    </div>
  );
}
```

- [ ] **Step 7: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/components.test.tsx && npx tsc --noEmit`
Expected: PASS (todos os testes do arquivo); tsc sem erros. Um aviso do Recharts sobre largura 0 no console é esperado em jsdom e não falha o teste.

- [ ] **Step 8: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/hooks/useAnalytics.ts src/modules/analytics/values.tsx src/modules/analytics/DataStamp.tsx \
  src/modules/analytics/SeriesChart.tsx src/modules/analytics/SeriesBlock.tsx src/modules/analytics/AnalyticsView.tsx \
  src/modules/analytics/components.test.tsx
/opt/homebrew/bin/git commit -m "feat: add shared analytics building blocks

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Seletores — intervalo, bairro, unidade, protocolo e versão

**Files:**
- Create: `src/modules/analytics/Filters.tsx`
- Test: `src/modules/analytics/Filters.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `Granularity`;
  - da Task 2: `AnalyticsRange`, `RANGE_OPTIONS`, `GRANULARITY_LABEL`, `rangeLabel`, `withGranularity`, `protocolOptions`, `ANALYTICS_PROTOCOLS_KEY`;
  - da Task 1 também: `AnalyticsUnit`;
  - já existentes: `listPanelNeighborhoods`, `listAuthorProtocols` (`src/lib/api.ts`), `NONE` e `PANEL_NEIGHBORHOODS_KEY` (`src/lib/neighborhoodFilter.ts`), `sortByName` (`src/lib/territory.ts`), `inputStyle`.
- Produces:
  - `filterRowStyle` (estilo da linha de recortes);
  - `RangeControls({ value: AnalyticsRange; onChange(next: AnalyticsRange) })` — grupo "Período" com os selects "Agrupar por" e "Intervalo";
  - `NeighborhoodSelect({ value: string | null; onChange(next: string | null) })` — "Bairro": Todos (`null`), Sem bairro (`"none"`), bairros com "(inativo)";
  - `UnitSelect({ value: string | null; units: AnalyticsUnit[]; onChange(next: string | null) })` — "Unidade": Todas (`null`) e as unidades na ordem de `data.units`, com "(inativa)" nas inativas;
  - `ProtocolSelect({ name: string | null; version: number | null; withVersion: boolean; onChange(name: string | null, version: number | null) })` — "Protocolo" e, com `withVersion`, "Versão" (travada sem protocolo; trocar o protocolo manda `version = null`).

Estes seletores **não** usam o `?bairro=` da URL (`useNeighborhoodParam`) dos painéis do módulo 11: o recorte do Analytics é estado da aba e não pode ligar o filtro dos outros painéis.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/analytics/Filters.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, within } from "@testing-library/react";
import { NeighborhoodSelect, ProtocolSelect, RangeControls, UnitSelect } from "./Filters";
import { DEFAULT_RANGE } from "../../lib/analytics";
import { NB1, U1, U2, UNITS, renderWithQuery, stubAnalyticsApi } from "../../test/analyticsFixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

const optionTexts = (select: HTMLElement) => within(select).getAllByRole("option").map((o) => o.textContent);

describe("RangeControls", () => {
  it("lista os intervalos da granularidade e troca de granularidade sem sair dos limites", () => {
    const onChange = vi.fn();
    renderWithQuery(<RangeControls value={DEFAULT_RANGE} onChange={onChange} />);
    expect(optionTexts(screen.getByLabelText("Intervalo"))).toEqual([
      "últimas 12 semanas", "últimas 26 semanas", "últimas 52 semanas", "últimas 104 semanas"
    ]);
    fireEvent.change(screen.getByLabelText("Agrupar por"), { target: { value: "month" } });
    expect(onChange).toHaveBeenCalledWith({ granularity: "month", count: 12 });
    fireEvent.change(screen.getByLabelText("Intervalo"), { target: { value: "52" } });
    expect(onChange).toHaveBeenLastCalledWith({ granularity: "week", count: 52 });
  });
});

describe("NeighborhoodSelect", () => {
  it("Todos, Sem bairro e os bairros da cidade, com inativo marcado", async () => {
    stubAnalyticsApi({});
    const onChange = vi.fn();
    renderWithQuery(<NeighborhoodSelect value={null} onChange={onChange} />);
    expect(await screen.findByRole("option", { name: "Xaxim (inativo)" })).toBeTruthy();
    expect(optionTexts(screen.getByLabelText("Bairro"))).toEqual([ "Todos", "Sem bairro", "Boqueirão", "Xaxim (inativo)" ]);

    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: NB1 } });
    expect(onChange).toHaveBeenLastCalledWith(NB1);
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "none" } });
    expect(onChange).toHaveBeenLastCalledWith("none");
  });

  it("voltar para Todos manda null, não string vazia", async () => {
    stubAnalyticsApi({});
    const onChange = vi.fn();
    renderWithQuery(<NeighborhoodSelect value={NB1} onChange={onChange} />);
    await screen.findByRole("option", { name: "Boqueirão" });
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "" } });
    expect(onChange).toHaveBeenLastCalledWith(null);
  });
});

describe("UnitSelect", () => {
  it("unidade inativa aparece marcada, na ordem de data.units", () => {
    const onChange = vi.fn();
    renderWithQuery(<UnitSelect value={null} units={UNITS} onChange={onChange} />);
    expect(optionTexts(screen.getByLabelText("Unidade"))).toEqual([ "Todas", "UBS Antiga (inativa)", "UBS Centro", "UPA Boqueirão" ]);
    fireEvent.change(screen.getByLabelText("Unidade"), { target: { value: U1 } });
    expect(onChange).toHaveBeenLastCalledWith(U1);
    fireEvent.change(screen.getByLabelText("Unidade"), { target: { value: "" } });
    expect(onChange).toHaveBeenLastCalledWith(null);
  });

  it("antes da resposta chegar, a escolhida continua visível", () => {
    renderWithQuery(<UnitSelect value={U2} units={[]} onChange={vi.fn()} />);
    expect(optionTexts(screen.getByLabelText("Unidade"))).toEqual([ "Todas", "unidade selecionada" ]);
  });
});

describe("ProtocolSelect", () => {
  it("protocolos sem rascunho; sem versão quando withVersion é false", async () => {
    stubAnalyticsApi({});
    renderWithQuery(<ProtocolSelect name={null} version={null} withVersion={false} onChange={vi.fn()} />);
    expect(await screen.findByRole("option", { name: "respiratorio" })).toBeTruthy();
    expect(optionTexts(screen.getByLabelText("Protocolo"))).toEqual([ "Todos", "arbovirose", "respiratorio" ]);
    expect(screen.queryByLabelText("Versão")).toBeNull();
  });

  it("versão travada sem protocolo; trocar o protocolo zera a versão", async () => {
    stubAnalyticsApi({});
    const onChange = vi.fn();
    const { rerender } = renderWithQuery(<ProtocolSelect name={null} version={null} withVersion onChange={onChange} />);
    await screen.findByRole("option", { name: "arbovirose" });
    expect((screen.getByLabelText("Versão") as HTMLSelectElement).disabled).toBe(true);

    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "arbovirose" } });
    expect(onChange).toHaveBeenLastCalledWith("arbovirose", null);

    rerender(<ProtocolSelect name="arbovirose" version={2} withVersion onChange={onChange} />);
    expect(optionTexts(screen.getByLabelText("Versão"))).toEqual([ "Todas", "versão 2", "versão 1" ]);
    fireEvent.change(screen.getByLabelText("Versão"), { target: { value: "1" } });
    expect(onChange).toHaveBeenLastCalledWith("arbovirose", 1);

    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "respiratorio" } });
    expect(onChange).toHaveBeenLastCalledWith("respiratorio", null);
    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "" } });
    expect(onChange).toHaveBeenLastCalledWith(null, null);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/Filters.test.tsx`
Expected: FAIL, `Failed to resolve import "./Filters"`.

- [ ] **Step 3: Escreva `Filters.tsx`**

```tsx
// src/modules/analytics/Filters.tsx
// Seletores do Analytics (spec §8). O recorte é estado da aba, não da URL: o
// `?bairro=` dos painéis do módulo 11 não é tocado. Recorte vazio = null.
import { useQuery } from "@tanstack/react-query";
import { listAuthorProtocols, listPanelNeighborhoods, type AnalyticsUnit, type Granularity } from "../../lib/api";
import {
  ANALYTICS_PROTOCOLS_KEY, GRANULARITY_LABEL, RANGE_OPTIONS, protocolOptions, rangeLabel, withGranularity,
  type AnalyticsRange
} from "../../lib/analytics";
import { NONE, PANEL_NEIGHBORHOODS_KEY } from "../../lib/neighborhoodFilter";
import { sortByName } from "../../lib/territory";
import { inputStyle } from "../../components/formStyles";

export const filterRowStyle = { display: "flex", gap: 12, alignItems: "flex-end", flexWrap: "wrap" as const };
const labelStyle = { display: "flex", flexDirection: "column" as const, gap: 2, fontSize: 12, color: "var(--ink2)", minWidth: 180 };
const errorStyle = { fontSize: 11, color: "var(--down)" };

export function RangeControls({ value, onChange }: { value: AnalyticsRange; onChange(next: AnalyticsRange): void }) {
  return (
    <div role="group" aria-label="Período" style={filterRowStyle}>
      <label style={labelStyle}>
        Agrupar por
        <select value={value.granularity} style={inputStyle}
          onChange={(e) => onChange(withGranularity(value, e.target.value as Granularity))}>
          {(Object.keys(GRANULARITY_LABEL) as Granularity[]).map((g) => <option key={g} value={g}>{GRANULARITY_LABEL[g]}</option>)}
        </select>
      </label>
      <label style={labelStyle}>
        Intervalo
        <select value={String(value.count)} style={inputStyle}
          onChange={(e) => onChange({ ...value, count: Number(e.target.value) })}>
          {RANGE_OPTIONS[value.granularity].map((n) => <option key={n} value={n}>{rangeLabel(value.granularity, n)}</option>)}
        </select>
      </label>
    </div>
  );
}

export function NeighborhoodSelect({ value, onChange }: { value: string | null; onChange(next: string | null): void }) {
  const list = useQuery({ queryKey: PANEL_NEIGHBORHOODS_KEY, queryFn: listPanelNeighborhoods });
  return (
    <label style={labelStyle}>
      Bairro
      <select value={value ?? ""} style={inputStyle} onChange={(e) => onChange(e.target.value || null)}>
        <option value="">Todos</option>
        <option value={NONE}>Sem bairro</option>
        {sortByName(list.data ?? []).map((n) => (
          <option key={n.id} value={n.id}>{n.active ? n.name : `${n.name} (inativo)`}</option>
        ))}
      </select>
      {list.isError && <span style={errorStyle}>não foi possível carregar os bairros</span>}
    </label>
  );
}

export function UnitSelect({ value, units, onChange }: { value: string | null; units: AnalyticsUnit[]; onChange(next: string | null): void }) {
  // `units` é o `data.units` da resposta (todas as unidades da cidade,
  // independentes do recorte). Enquanto a resposta não chega, a escolhida
  // continua visível.
  const missing = !!value && !units.some((u) => u.health_unit_id === value);
  return (
    <label style={labelStyle}>
      Unidade
      <select value={value ?? ""} style={inputStyle} onChange={(e) => onChange(e.target.value || null)}>
        <option value="">Todas</option>
        {missing && <option value={value ?? ""}>unidade selecionada</option>}
        {units.map((u) => (
          <option key={u.health_unit_id} value={u.health_unit_id}>{u.active ? u.name : `${u.name} (inativa)`}</option>
        ))}
      </select>
    </label>
  );
}

interface ProtocolSelectProps {
  name: string | null;
  version: number | null;
  withVersion: boolean;
  onChange(name: string | null, version: number | null): void;
}

export function ProtocolSelect({ name, version, withVersion, onChange }: ProtocolSelectProps) {
  const list = useQuery({ queryKey: ANALYTICS_PROTOCOLS_KEY, queryFn: listAuthorProtocols });
  const options = protocolOptions(list.data ?? []);
  const versions = options.find((o) => o.name === name)?.versions ?? [];
  return (
    <>
      <label style={labelStyle}>
        Protocolo
        {/* Trocar o protocolo sempre zera a versão: versão sozinha é 422 invalid_protocol. */}
        <select value={name ?? ""} style={inputStyle} onChange={(e) => onChange(e.target.value || null, null)}>
          <option value="">Todos</option>
          {options.map((o) => <option key={o.name} value={o.name}>{o.name}</option>)}
        </select>
        {list.isError && <span style={errorStyle}>não foi possível carregar os protocolos</span>}
      </label>
      {withVersion && (
        <label style={labelStyle}>
          Versão
          <select value={version === null ? "" : String(version)} disabled={!name} style={inputStyle}
            onChange={(e) => onChange(name, e.target.value ? Number(e.target.value) : null)}>
            <option value="">Todas</option>
            {versions.map((v) => <option key={v} value={v}>versão {v}</option>)}
          </select>
        </label>
      )}
    </>
  );
}
```

- [ ] **Step 4: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics && npx tsc --noEmit`
Expected: PASS (Filters e components); tsc sem erros.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/Filters.tsx src/modules/analytics/Filters.test.tsx
/opt/homebrew/bin/git commit -m "feat: add analytics range and filter selectors

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Aba Demanda (F-14.3)

**Files:**
- Create: `src/modules/analytics/DemandTab.tsx`
- Test: `src/modules/analytics/DemandTab.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `AnalyticsQuery`, `DemandData`;
  - da Task 2: `AnalyticsRange`, `rangeDates`, `KIND_LABEL`, `CLOSED_REASON_LABEL`, `labelOr`;
  - da Task 3: `useAnalytics`, `AnalyticsView`, `SeriesBlock`, `CellValue`;
  - da Task 4: `NeighborhoodSelect`, `ProtocolSelect`, `UnitSelect`, `filterRowStyle`;
  - já existentes: `Panel`, `DataTable`.
- Produces: `DemandTab({ range: AnalyticsRange })`, com as regiões "Triagens", "Triagens concluídas por tier", "Triagens concluídas por protocolo", "Triagens concluídas por bairro", "Atendimentos por unidade", "Pedidos abertos" e "Pedidos encerrados".

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/analytics/DemandTab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { DemandTab } from "./DemandTab";
import { DEFAULT_RANGE } from "../../lib/analytics";
import {
  NB1, NOW, U1, demandData, envelope, failWith, paramsOf, renderWithQuery, rowWith, stubAnalyticsApi
} from "../../test/analyticsFixtures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(NOW);
});
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

const region = (name: string) => within(screen.getByRole("region", { name }));
const lastParams = (fn: ReturnType<typeof stubAnalyticsApi>) => paramsOf(fn, "/analytics/demand").at(-1)!;

describe("DemandTab", () => {
  it("pede as últimas 12 semanas até ontem, sem recorte, e mostra o carimbo", async () => {
    const fn = stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    expect(await screen.findByText("dados até 29/09")).toBeTruthy();
    expect(Object.fromEntries(paramsOf(fn, "/analytics/demand")[0])).toEqual({
      from: "2026-07-13", to: "2026-09-29", granularity: "week"
    });
  });

  it("triagens: total do período vem de triages_total, não da soma dos pontos", async () => {
    stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    await screen.findByText("dados até 29/09");
    const triages = region("Triagens");
    const totals = triages.getByRole("group", { name: "Totais do período" });
    expect(rowWith(totals, "Iniciadas")).toBe("Iniciadas15");
    expect(rowWith(totals, "Concluídas")).toBe("Concluídas13");
    expect(rowWith(totals, "Interrompidas")).toBe("Interrompidasoculto");
    expect(rowWith(triages.getByRole("table", { name: "Triagens por período" }), "21/09")).toContain("oculto");
    expect(triages.getByRole("list", { name: "legenda" }).textContent).toBe("IniciadasConcluídasInterrompidas");
  });

  it("totais por tier, bairro, unidade e pedidos: número ou oculto, como veio", async () => {
    stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    await screen.findByText("dados até 29/09");
    const tierTotals = region("Triagens concluídas por tier").getByRole("group", { name: "Totais do período" });
    expect(rowWith(tierTotals, "vermelho")).toContain("8");
    expect(rowWith(tierTotals, "verde")).toContain("oculto");
    expect(rowWith(screen.getByRole("region", { name: "Triagens concluídas por bairro" }), "Sem bairro")).toContain("oculto");
    expect(rowWith(screen.getByRole("region", { name: "Triagens concluídas por bairro" }), "Boqueirão")).toContain("9");
    expect(rowWith(screen.getByRole("region", { name: "Atendimentos por unidade" }), "UPA Boqueirão")).toContain("oculto");
    expect(rowWith(screen.getByRole("region", { name: "Pedidos abertos" }), "Encaminhamento")).toContain("0");
    expect(rowWith(screen.getByRole("region", { name: "Pedidos encerrados" }), "Atendido")).toContain("6");
  });

  it("bairro recorta com neighborhood_id; 'Sem bairro' manda none", async () => {
    const fn = stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    await screen.findByRole("option", { name: "Boqueirão" });
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: NB1 } });
    await waitFor(() => expect(lastParams(fn).get("neighborhood_id")).toBe(NB1));
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "none" } });
    await waitFor(() => expect(lastParams(fn).get("neighborhood_id")).toBe("none"));
  });

  it("protocolo recorta sem versão (a Demanda não tem seletor de versão)", async () => {
    const fn = stubAnalyticsApi({ "/analytics/demand": envelope<"demand">(demandData()) });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    await screen.findByRole("option", { name: "arbovirose" });
    expect(screen.queryByLabelText("Versão")).toBeNull();
    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "arbovirose" } });
    await waitFor(() => expect(lastParams(fn).get("protocol_name")).toBe("arbovirose"));
    expect(lastParams(fn).has("protocol_version")).toBe(false);
  });

  it("unidade: a lista vem de data.units e continua inteira depois de escolher", async () => {
    // Com health_unit_id, attendances_by_unit encolhe para a escolhida;
    // data.units continua com todas (contratos §1).
    const fn = stubAnalyticsApi({
      "/analytics/demand": (url: URL) => {
        const unit = url.searchParams.get("health_unit_id");
        const data = demandData();
        return envelope<"demand">(unit
          ? { ...data, attendances_by_unit: data.attendances_by_unit.filter((u) => u.health_unit_id === unit) }
          : data);
      }
    });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    // UBS Antiga não tem atendimento no intervalo e mesmo assim é opção.
    expect(await screen.findByRole("option", { name: "UBS Antiga (inativa)" })).toBeTruthy();
    fireEvent.change(screen.getByLabelText("Unidade"), { target: { value: U1 } });
    await waitFor(() => expect(lastParams(fn).get("health_unit_id")).toBe(U1));
    await waitFor(() => expect(screen.getByRole("region", { name: "Atendimentos por unidade" }).textContent).not.toContain("UPA Boqueirão"));
    expect(screen.getByRole("option", { name: "UPA Boqueirão" })).toBeTruthy();
    expect(screen.getByRole("option", { name: "UBS Antiga (inativa)" })).toBeTruthy();
  });

  it("bairro que não existe na cidade: 422 vira frase", async () => {
    stubAnalyticsApi({
      "/analytics/demand": (url: URL) => url.searchParams.has("neighborhood_id")
        ? failWith(422, { error: "invalid_neighborhood" })
        : envelope<"demand">(demandData())
    });
    renderWithQuery(<DemandTab range={DEFAULT_RANGE} />);
    await screen.findByRole("option", { name: "Xaxim (inativo)" });
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "none" } });
    expect(await screen.findByText("Esse bairro não existe nesta cidade — escolha outro.")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/DemandTab.test.tsx`
Expected: FAIL, `Failed to resolve import "./DemandTab"`.

- [ ] **Step 3: Escreva `DemandTab.tsx`**

```tsx
// src/modules/analytics/DemandTab.tsx
// Demanda por território (F-14.3; spec §3.4, contratos §1.1). Bairro e
// protocolo recortam as métricas de triagem; unidade recorta atendimentos e
// pedidos. O total do período das triagens é o `triages_total` da API; o
// cliente nunca soma os pontos da série.
import { useState } from "react";
import type { AnalyticsQuery, DemandData } from "../../lib/api";
import { CLOSED_REASON_LABEL, KIND_LABEL, labelOr, rangeDates, type AnalyticsRange } from "../../lib/analytics";
import { useAnalytics } from "../../hooks/useAnalytics";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { AnalyticsView } from "./AnalyticsView";
import { SeriesBlock } from "./SeriesBlock";
import { CellValue } from "./values";
import { NeighborhoodSelect, ProtocolSelect, UnitSelect, filterRowStyle } from "./Filters";

interface DemandFilter { neighborhood_id: string | null; health_unit_id: string | null; protocol_name: string | null }
const NO_FILTER: DemandFilter = { neighborhood_id: null, health_unit_id: null, protocol_name: null };

export function DemandTab({ range }: { range: AnalyticsRange }) {
  const [ filter, setFilter ] = useState<DemandFilter>(NO_FILTER);
  const query: AnalyticsQuery = { ...rangeDates(range), granularity: range.granularity, ...filter };
  const result = useAnalytics("demand", query);
  // data.units: todas as unidades da cidade, independentes do recorte.
  const units = result.data?.data.units ?? [];
  const set = (patch: Partial<DemandFilter>) => setFilter((current) => ({ ...current, ...patch }));

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div role="group" aria-label="Recortes" style={filterRowStyle}>
        <NeighborhoodSelect value={filter.neighborhood_id} onChange={(v) => set({ neighborhood_id: v })} />
        <ProtocolSelect name={filter.protocol_name} version={null} withVersion={false}
          onChange={(name) => set({ protocol_name: name })} />
        <UnitSelect value={filter.health_unit_id} units={units} onChange={(v) => set({ health_unit_id: v })} />
      </div>
      <p style={noteStyle}>Bairro e protocolo recortam as triagens; unidade recorta atendimentos e pedidos.</p>
      <AnalyticsView result={result}>{(data) => <DemandBody data={data} />}</AnalyticsView>
    </div>
  );
}

function DemandBody({ data }: { data: DemandData }) {
  const common = { periods: data.periods, granularity: data.granularity ?? "week", kind: "count" as const };
  return (
    <>
      <SeriesBlock {...common} title="Triagens" sub="iniciadas · concluídas · interrompidas" lines={[
        { key: "started", label: "Iniciadas", series: data.triages.started, total: data.triages_total.started },
        { key: "completed", label: "Concluídas", series: data.triages.completed, total: data.triages_total.completed },
        { key: "aborted", label: "Interrompidas", series: data.triages.aborted, total: data.triages_total.aborted }
      ]} />
      <SeriesBlock {...common} title="Triagens concluídas por tier"
        lines={data.by_tier.map((r) => ({ key: r.tier, label: r.tier, series: r.series, total: r.total }))} />
      <SeriesBlock {...common} title="Triagens concluídas por protocolo"
        lines={data.by_protocol.map((r) => ({ key: r.protocol_name, label: r.protocol_name, series: r.series, total: r.total }))} />
      <Panel title="Triagens concluídas por bairro" sub="período inteiro">
        <DataTable
          cols={[
            { label: "Bairro", w: "2fr", render: (r: DemandData["by_neighborhood"][number]) => r.name },
            { label: "Triagens", w: "1fr", align: "right", render: (r: DemandData["by_neighborhood"][number]) => (
              <span className="mono"><CellValue value={r.total} /></span>
            ) }
          ]}
          rows={data.by_neighborhood}
          rowKey={(r) => r.neighborhood_id ?? "none"}
          empty="sem triagens concluídas no período"
        />
      </Panel>
      <SeriesBlock {...common} title="Atendimentos por unidade" sub="check-ins"
        lines={data.attendances_by_unit.map((r) => ({ key: r.health_unit_id, label: r.name, series: r.series, total: r.total }))} />
      <SeriesBlock {...common} title="Pedidos abertos" sub="por tipo, na unidade de destino"
        lines={data.requests_opened.map((r) => ({ key: r.kind, label: labelOr(KIND_LABEL, r.kind), series: r.series, total: r.total }))} />
      <SeriesBlock {...common} title="Pedidos encerrados" sub="por motivo"
        lines={data.requests_closed.map((r) => ({ key: r.reason, label: labelOr(CLOSED_REASON_LABEL, r.reason), series: r.series, total: r.total }))} />
    </>
  );
}

const noteStyle = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
```

- [ ] **Step 4: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/DemandTab.test.tsx && npx tsc --noEmit`
Expected: PASS, 7 testes; tsc sem erros.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/DemandTab.tsx src/modules/analytics/DemandTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the analytics demand tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Aba Qualidade (F-14.4)

**Files:**
- Create: `src/modules/analytics/QualityTab.tsx`
- Test: `src/modules/analytics/QualityTab.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `AnalyticsQuery`, `QualityData`;
  - da Task 2: `AnalyticsRange`, `rangeDates`, `WAIT_BUCKET_LABEL`, `APPOINTMENT_LABEL`, `OUTCOME_LABEL`, `labelOr`;
  - da Task 3: `useAnalytics`, `AnalyticsView`, `SeriesBlock`, `CellValue`, `RateValue`;
  - da Task 4: `UnitSelect`, `filterRowStyle`.
- Produces: `QualityTab({ range: AnalyticsRange })`, com as regiões "Espera até a chamada", "Chamados em até 30 min", "Agendamentos encerrados", "Faltas", "Desfechos dos atendimentos", "Saiu sem atendimento" e "Por unidade".

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/analytics/QualityTab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { QualityTab } from "./QualityTab";
import { DEFAULT_RANGE } from "../../lib/analytics";
import {
  NOW, U2, envelope, paramsOf, qualityData, renderWithQuery, rowWith, stubAnalyticsApi
} from "../../test/analyticsFixtures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(NOW);
});
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

const region = (name: string) => screen.getByRole("region", { name });
const totals = (name: string) => within(region(name)).getByRole("group", { name: "Totais do período" });

describe("QualityTab", () => {
  it("só tem o recorte de unidade, e manda período e agrupamento", async () => {
    const fn = stubAnalyticsApi({ "/analytics/quality": envelope<"quality">(qualityData()) });
    renderWithQuery(<QualityTab range={{ granularity: "month", count: 6 }} />);
    await screen.findByText("dados até 29/09");
    expect(Object.fromEntries(paramsOf(fn, "/analytics/quality")[0])).toEqual({
      from: "2026-04-01", to: "2026-09-29", granularity: "month"
    });
    expect(screen.queryByLabelText("Bairro")).toBeNull();
    expect(screen.queryByLabelText("Protocolo")).toBeNull();
    expect(screen.getByLabelText("Unidade")).toBeTruthy();
  });

  it("faixas de espera nas cinco linhas, na ordem da API, com rótulo", async () => {
    stubAnalyticsApi({ "/analytics/quality": envelope<"quality">(qualityData()) });
    renderWithQuery(<QualityTab range={DEFAULT_RANGE} />);
    await screen.findByText("dados até 29/09");
    const rows = within(totals("Espera até a chamada")).getAllByRole("row").slice(1).map((r) => r.textContent);
    expect(rows).toEqual([ "até 15 min14", "15 a 30 min7", "30 a 60 min0", "1 a 2 h0", "mais de 2 hoculto" ]);
  });

  it("taxas: uma casa, oculto e sem dado", async () => {
    stubAnalyticsApi({ "/analytics/quality": envelope<"quality">(qualityData()) });
    renderWithQuery(<QualityTab range={DEFAULT_RANGE} />);
    await screen.findByText("dados até 29/09");
    expect(rowWith(totals("Chamados em até 30 min"), "até 30 min")).toContain("87,5%");
    expect(rowWith(within(region("Chamados em até 30 min")).getByRole("table", { name: "Chamados em até 30 min por período" }), "28/09"))
      .toContain("sem dado");
    expect(rowWith(totals("Faltas"), "faltas")).toContain("20,8%");
    expect(rowWith(totals("Saiu sem atendimento"), "saiu sem atendimento")).toContain("oculto");
    expect(rowWith(totals("Agendamentos encerrados"), "Faltou")).toContain("5");
    expect(rowWith(totals("Desfechos dos atendimentos"), "Encaminhado")).toContain("5");
  });

  it("por unidade: contagem e as três taxas, cada uma como veio", async () => {
    stubAnalyticsApi({ "/analytics/quality": envelope<"quality">(qualityData()) });
    renderWithQuery(<QualityTab range={DEFAULT_RANGE} />);
    await screen.findByText("dados até 29/09");
    expect(rowWith(region("Por unidade"), "UBS Centro")).toBe("UBS Centro2287,5%20,8%oculto");
    expect(rowWith(region("Por unidade"), "UPA Boqueirão")).toBe("UPA Boqueirãoocultoocultosem dadooculto");
  });

  it("unidade: a lista vem de data.units e continua inteira depois de escolher", async () => {
    const fn = stubAnalyticsApi({
      "/analytics/quality": (url: URL) => {
        const unit = url.searchParams.get("health_unit_id");
        const data = qualityData();
        return envelope<"quality">(unit ? { ...data, by_unit: data.by_unit.filter((u) => u.health_unit_id === unit) } : data);
      }
    });
    renderWithQuery(<QualityTab range={DEFAULT_RANGE} />);
    expect(await screen.findByRole("option", { name: "UBS Antiga (inativa)" })).toBeTruthy();
    fireEvent.change(screen.getByLabelText("Unidade"), { target: { value: U2 } });
    await waitFor(() => expect(paramsOf(fn, "/analytics/quality").at(-1)!.get("health_unit_id")).toBe(U2));
    await waitFor(() => expect(within(region("Por unidade")).queryByText("UBS Centro")).toBeNull());
    expect(screen.getByRole("option", { name: "UBS Centro" })).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/QualityTab.test.tsx`
Expected: FAIL, `Failed to resolve import "./QualityTab"`.

- [ ] **Step 3: Escreva `QualityTab.tsx`**

```tsx
// src/modules/analytics/QualityTab.tsx
// Qualidade operacional (F-14.4; contratos §1.2): espera em faixas (não
// mediana, D11), faltas, desfechos e "saiu sem atendimento", por unidade.
// Taxas vêm prontas da API; nenhuma é recalculada aqui.
import { useState } from "react";
import type { AnalyticsQuery, QualityData } from "../../lib/api";
import {
  APPOINTMENT_LABEL, OUTCOME_LABEL, WAIT_BUCKET_LABEL, labelOr, rangeDates, type AnalyticsRange
} from "../../lib/analytics";
import { useAnalytics } from "../../hooks/useAnalytics";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { AnalyticsView } from "./AnalyticsView";
import { SeriesBlock } from "./SeriesBlock";
import { CellValue, RateValue } from "./values";
import { UnitSelect, filterRowStyle } from "./Filters";

type UnitRow = QualityData["by_unit"][number];

export function QualityTab({ range }: { range: AnalyticsRange }) {
  const [ unit, setUnit ] = useState<string | null>(null);
  const query: AnalyticsQuery = { ...rangeDates(range), granularity: range.granularity, health_unit_id: unit };
  const result = useAnalytics("quality", query);
  // data.units: todas as unidades da cidade, independentes do recorte.
  const units = result.data?.data.units ?? [];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div role="group" aria-label="Recortes" style={filterRowStyle}>
        <UnitSelect value={unit} units={units} onChange={setUnit} />
      </div>
      <AnalyticsView result={result}>{(data) => <QualityBody data={data} />}</AnalyticsView>
    </div>
  );
}

function QualityBody({ data }: { data: QualityData }) {
  const common = { periods: data.periods, granularity: data.granularity ?? "week" };
  return (
    <>
      <SeriesBlock {...common} kind="count" title="Espera até a chamada" sub="atendimentos chamados, por faixa de espera"
        lines={data.wait.buckets.map((b) => ({ key: b.bucket, label: labelOr(WAIT_BUCKET_LABEL, b.bucket), series: b.series, total: b.total }))} />
      <SeriesBlock {...common} kind="rate" title="Chamados em até 30 min" sub="(até 15 + 15 a 30) ÷ todas as faixas"
        lines={[ { key: "within_30", label: "até 30 min", series: data.wait.within_30_pct, total: data.wait.within_30_pct_total } ]} />
      <SeriesBlock {...common} kind="count" title="Agendamentos encerrados" sub="por situação final"
        lines={data.appointments.map((a) => ({ key: a.status, label: labelOr(APPOINTMENT_LABEL, a.status), series: a.series, total: a.total }))} />
      <SeriesBlock {...common} kind="rate" title="Faltas" sub="faltou ÷ (compareceu + faltou)"
        lines={[ { key: "no_show", label: "faltas", series: data.no_show_pct, total: data.no_show_pct_total } ]} />
      <SeriesBlock {...common} kind="count" title="Desfechos dos atendimentos"
        lines={data.attendance_outcomes.map((o) => ({ key: o.outcome, label: OUTCOME_LABEL[o.outcome], series: o.series, total: o.total }))} />
      <SeriesBlock {...common} kind="rate" title="Saiu sem atendimento" sub="saiu ÷ todos os desfechos"
        lines={[ { key: "left", label: "saiu sem atendimento", series: data.left_pct, total: data.left_pct_total } ]} />
      <Panel title="Por unidade" sub="período inteiro">
        <DataTable<UnitRow>
          cols={[
            { label: "Unidade", w: "2fr", render: (r) => r.name },
            { label: "Atendimentos", w: "1fr", align: "right", render: (r) => <span className="mono"><CellValue value={r.attendances} /></span> },
            { label: "Até 30 min", w: "1fr", align: "right", render: (r) => <span className="mono"><RateValue value={r.wait_within_30_pct} /></span> },
            { label: "Faltas", w: "1fr", align: "right", render: (r) => <span className="mono"><RateValue value={r.no_show_pct} /></span> },
            { label: "Saiu sem atendimento", w: "1fr", align: "right", render: (r) => <span className="mono"><RateValue value={r.left_pct} /></span> }
          ]}
          rows={data.by_unit}
          rowKey={(r) => r.health_unit_id}
          empty="nenhum atendimento no período"
        />
      </Panel>
    </>
  );
}
```

- [ ] **Step 4: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/QualityTab.test.tsx && npx tsc --noEmit`
Expected: PASS, 5 testes; tsc sem erros. Se o teste das faixas falhar só por espaço entre rótulo e número, confira se o `DataTable` renderiza as células sem separador (é o que `Team.test.tsx` pressupõe); não troque o texto esperado por regex frouxa.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/QualityTab.tsx src/modules/analytics/QualityTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the analytics quality tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Aba Calibração (F-14.5)

**Files:**
- Create: `src/modules/analytics/CalibrationTab.tsx`
- Test: `src/modules/analytics/CalibrationTab.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `AnalyticsQuery`, `CalibrationData`, `CalibrationRow`, `Cell`, `Rate`;
  - da Task 2: `AnalyticsRange`, `rangeDates`, `CALIBRATION_OUTCOMES`, `OUTCOME_LABEL`;
  - da Task 3: `useAnalytics`, `AnalyticsView`, `CellValue`, `RateValue`;
  - da Task 4: `ProtocolSelect`, `filterRowStyle`;
  - já existentes: `Panel`, `DataTable`, `EmptyState`.
- Produces: `CalibrationTab({ range: AnalyticsRange })`. Uma região por versão, com o nome `"<protocolo> · versão <n>"`; cada linha é um tier, com "Triagens" e uma coluna por desfecho no formato `"<contagem> (<proporção>)"`.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/analytics/CalibrationTab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { CalibrationTab } from "./CalibrationTab";
import { DEFAULT_RANGE } from "../../lib/analytics";
import {
  NOW, calibrationData, envelope, paramsOf, renderWithQuery, rowWith, stubAnalyticsApi
} from "../../test/analyticsFixtures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(NOW);
});
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

const lastParams = (fn: ReturnType<typeof stubAnalyticsApi>) => paramsOf(fn, "/analytics/calibration").at(-1)!;

describe("CalibrationTab", () => {
  it("período inteiro, sem granularity no pedido", async () => {
    const fn = stubAnalyticsApi({ "/analytics/calibration": envelope<"calibration">(calibrationData()) });
    renderWithQuery(<CalibrationTab range={DEFAULT_RANGE} />);
    await screen.findByText("dados até 29/09");
    expect(Object.fromEntries(paramsOf(fn, "/analytics/calibration")[0])).toEqual({ from: "2026-07-13", to: "2026-09-29" });
  });

  it("tier × desfecho com a proporção por linha, uma tabela por versão", async () => {
    stubAnalyticsApi({ "/analytics/calibration": envelope<"calibration">(calibrationData()) });
    renderWithQuery(<CalibrationTab range={DEFAULT_RANGE} />);
    const v2 = await screen.findByRole("region", { name: "arbovirose · versão 2" });
    expect(rowWith(v2, "vermelho")).toBe("vermelho2010 (50,0%)6 (30,0%)0 (0,0%)oculto (oculto)oculto (oculto)");
    const v1 = screen.getByRole("region", { name: "arbovirose · versão 1" });
    expect(rowWith(v1, "verde")).toBe("verdeocultooculto (oculto)0 (oculto)0 (oculto)0 (oculto)0 (oculto)");
  });

  it("cabeçalho com os cinco desfechos, incluindo 'sem atendimento encerrado'", async () => {
    stubAnalyticsApi({ "/analytics/calibration": envelope<"calibration">(calibrationData()) });
    renderWithQuery(<CalibrationTab range={DEFAULT_RANGE} />);
    const v2 = await screen.findByRole("region", { name: "arbovirose · versão 2" });
    expect(Array.from(v2.querySelectorAll('[role="columnheader"]')).map((h) => h.textContent)).toEqual([
      "Tier", "Triagens", "Atendido e liberado", "Encaminhado", "Retorno", "Saiu sem atendimento", "Sem atendimento encerrado"
    ]);
  });

  it("versão vai com o protocolo; trocar o protocolo tira a versão do pedido", async () => {
    const fn = stubAnalyticsApi({ "/analytics/calibration": envelope<"calibration">(calibrationData()) });
    renderWithQuery(<CalibrationTab range={DEFAULT_RANGE} />);
    await screen.findByRole("option", { name: "arbovirose" });
    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "arbovirose" } });
    await screen.findByRole("option", { name: "versão 2" });
    fireEvent.change(screen.getByLabelText("Versão"), { target: { value: "2" } });
    await waitFor(() => expect(lastParams(fn).get("protocol_version")).toBe("2"));
    expect(lastParams(fn).get("protocol_name")).toBe("arbovirose");

    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "respiratorio" } });
    await waitFor(() => expect(lastParams(fn).get("protocol_name")).toBe("respiratorio"));
    expect(lastParams(fn).has("protocol_version")).toBe(false);
  });

  it("sem versões no período: estado vazio", async () => {
    stubAnalyticsApi({ "/analytics/calibration": envelope<"calibration">(calibrationData({ versions: [] })) });
    renderWithQuery(<CalibrationTab range={DEFAULT_RANGE} />);
    expect(await screen.findByText("nenhuma triagem concluída no período")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/CalibrationTab.test.tsx`
Expected: FAIL, `Failed to resolve import "./CalibrationTab"`.

- [ ] **Step 3: Escreva `CalibrationTab.tsx`**

```tsx
// src/modules/analytics/CalibrationTab.tsx
// Calibração de protocolo (F-14.5; contratos §1.3): por versão, tier ×
// desfecho do atendimento ligado, com a proporção por linha que a API manda.
// Período inteiro, sem série: o agrupamento por semana/mês não se aplica.
// É por versão porque `tier` é texto de cada protocolo (spec §14).
import { useState } from "react";
import type { AnalyticsQuery, CalibrationData, CalibrationRow, Cell, Rate } from "../../lib/api";
import { CALIBRATION_OUTCOMES, OUTCOME_LABEL, rangeDates, type AnalyticsRange } from "../../lib/analytics";
import { useAnalytics } from "../../hooks/useAnalytics";
import { Panel } from "../../components/Panel";
import { DataTable, type Column } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { AnalyticsView } from "./AnalyticsView";
import { CellValue, RateValue } from "./values";
import { ProtocolSelect, filterRowStyle } from "./Filters";

interface ProtocolFilter { name: string | null; version: number | null }

export function CalibrationTab({ range }: { range: AnalyticsRange }) {
  const [ protocol, setProtocol ] = useState<ProtocolFilter>({ name: null, version: null });
  const { from, to } = rangeDates(range);
  const query: AnalyticsQuery = { from, to, protocol_name: protocol.name, protocol_version: protocol.version };
  const result = useAnalytics("calibration", query);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div role="group" aria-label="Recortes" style={filterRowStyle}>
        <ProtocolSelect name={protocol.name} version={protocol.version} withVersion
          onChange={(name, version) => setProtocol({ name, version })} />
      </div>
      <p style={noteStyle}>Período inteiro, sem série: o agrupamento por semana ou mês não se aplica aqui.</p>
      <AnalyticsView result={result}>{(data) => <CalibrationBody data={data} />}</AnalyticsView>
    </div>
  );
}

const COLUMNS: Column<CalibrationRow>[] = [
  { label: "Tier", w: "1.2fr", render: (r) => r.tier },
  { label: "Triagens", w: "0.8fr", align: "right", render: (r) => <span className="mono"><CellValue value={r.total} /></span> },
  ...CALIBRATION_OUTCOMES.map((outcome): Column<CalibrationRow> => ({
    label: OUTCOME_LABEL[outcome], w: "1fr", align: "right",
    render: (r) => <OutcomeCell count={r.outcomes[outcome]} share={r.shares[outcome]} />
  }))
];

function CalibrationBody({ data }: { data: CalibrationData }) {
  if (data.versions.length === 0) {
    return <Panel title="Calibração"><EmptyState title="nenhuma triagem concluída no período" /></Panel>;
  }
  return (
    <>
      {data.versions.map((v) => (
        <Panel key={`${v.protocol_name}@${v.protocol_version}`} title={`${v.protocol_name} · versão ${v.protocol_version}`}
          sub="tier × desfecho · proporção por linha">
          <DataTable<CalibrationRow> cols={COLUMNS} rows={v.rows} rowKey={(r) => r.tier} empty="sem triagens nesta versão" />
        </Panel>
      ))}
    </>
  );
}

function OutcomeCell({ count, share }: { count: Cell; share: Rate }) {
  return (
    <span className="mono">
      <CellValue value={count} /> <span style={{ color: "var(--ink3)" }}>(<RateValue value={share} />)</span>
    </span>
  );
}

const noteStyle = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
```

`Column` já é exportado por `src/components/DataTable.tsx` (`export interface Column<T>`).

- [ ] **Step 4: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/CalibrationTab.test.tsx && npx tsc --noEmit`
Expected: PASS, 5 testes; tsc sem erros.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/CalibrationTab.tsx src/modules/analytics/CalibrationTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the analytics calibration tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Aba Epidemiologia (F-14.7)

**Files:**
- Create: `src/modules/analytics/EpidemiologyTab.tsx`
- Test: `src/modules/analytics/EpidemiologyTab.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `AnalyticsQuery`, `EpidemiologyData`;
  - da Task 2: `AnalyticsRange`, `rangeDates`, `EPI_EMPTY_TITLE`, `EPI_HOWTO`;
  - da Task 3: `useAnalytics`, `AnalyticsView`, `SeriesBlock`;
  - da Task 4: `NeighborhoodSelect`, `ProtocolSelect`, `filterRowStyle`;
  - já existentes: `Panel`, `EmptyState`.
- Produces: `EpidemiologyTab({ range: AnalyticsRange })`. Uma região por pergunta, com o nome igual ao `prompt`; uma linha (série) por opção.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/analytics/EpidemiologyTab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { EpidemiologyTab } from "./EpidemiologyTab";
import { DEFAULT_RANGE, EPI_HOWTO } from "../../lib/analytics";
import {
  NB1, NOW, envelope, epidemiologyData, paramsOf, renderWithQuery, rowWith, stubAnalyticsApi
} from "../../test/analyticsFixtures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(NOW);
});
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

const lastParams = (fn: ReturnType<typeof stubAnalyticsApi>) => paramsOf(fn, "/analytics/epidemiology").at(-1)!;

describe("EpidemiologyTab", () => {
  it("uma série por opção de cada pergunta marcada, com o total como veio", async () => {
    stubAnalyticsApi({ "/analytics/epidemiology": envelope<"epidemiology">(epidemiologyData()) });
    renderWithQuery(<EpidemiologyTab range={DEFAULT_RANGE} />);
    const fever = within(await screen.findByRole("region", { name: "Teve febre?" }));
    expect(fever.getByRole("list", { name: "legenda" }).textContent).toBe("SimNão");
    const totals = fever.getByRole("group", { name: "Totais do período" });
    expect(rowWith(totals, "Sim")).toContain("9");
    expect(rowWith(totals, "Não")).toContain("oculto");
    expect(within(screen.getByRole("region", { name: "Qual o sintoma principal?" })).getByText("arbovirose · lista")).toBeTruthy();
    expect(fever.getByText("arbovirose · sim/não")).toBeTruthy();
  });

  it("sem pergunta marcada: explica como marcar no editor", async () => {
    stubAnalyticsApi({ "/analytics/epidemiology": envelope<"epidemiology">(epidemiologyData({ questions: [] })) });
    renderWithQuery(<EpidemiologyTab range={DEFAULT_RANGE} />);
    expect(await screen.findByText("nenhuma pergunta marcada para Analytics")).toBeTruthy();
    expect(screen.getByText(EPI_HOWTO)).toBeTruthy();
  });

  it("recortes: bairro, protocolo e versão, com o agrupamento do intervalo", async () => {
    const fn = stubAnalyticsApi({ "/analytics/epidemiology": envelope<"epidemiology">(epidemiologyData()) });
    renderWithQuery(<EpidemiologyTab range={{ granularity: "month", count: 6 }} />);
    await screen.findByRole("option", { name: "Boqueirão" });
    expect(Object.fromEntries(paramsOf(fn, "/analytics/epidemiology")[0])).toEqual({
      from: "2026-04-01", to: "2026-09-29", granularity: "month"
    });
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: NB1 } });
    await waitFor(() => expect(lastParams(fn).get("neighborhood_id")).toBe(NB1));
    await screen.findByRole("option", { name: "arbovirose" });
    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "arbovirose" } });
    await screen.findByRole("option", { name: "versão 2" });
    fireEvent.change(screen.getByLabelText("Versão"), { target: { value: "2" } });
    await waitFor(() => expect(lastParams(fn).get("protocol_version")).toBe("2"));
    expect(lastParams(fn).get("protocol_name")).toBe("arbovirose");
    expect(lastParams(fn).get("neighborhood_id")).toBe(NB1);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/EpidemiologyTab.test.tsx`
Expected: FAIL, `Failed to resolve import "./EpidemiologyTab"`.

- [ ] **Step 3: Escreva `EpidemiologyTab.tsx`**

```tsx
// src/modules/analytics/EpidemiologyTab.tsx
// Epidemiologia (F-14.7; contratos §1.4): só perguntas `boolean`/`enum` que o
// autor marcou `analytic` (ADR 0025), agregadas por bairro, nunca por pessoa.
// Prompt e opções vêm da versão mais recente que marcou a pergunta.
import { useState } from "react";
import type { AnalyticsQuery, EpidemiologyData } from "../../lib/api";
import { EPI_EMPTY_TITLE, EPI_HOWTO, rangeDates, type AnalyticsRange } from "../../lib/analytics";
import { useAnalytics } from "../../hooks/useAnalytics";
import { Panel } from "../../components/Panel";
import { EmptyState } from "../../components/EmptyState";
import { AnalyticsView } from "./AnalyticsView";
import { SeriesBlock } from "./SeriesBlock";
import { NeighborhoodSelect, ProtocolSelect, filterRowStyle } from "./Filters";

interface EpiFilter { neighborhood_id: string | null; protocol_name: string | null; protocol_version: number | null }
const NO_FILTER: EpiFilter = { neighborhood_id: null, protocol_name: null, protocol_version: null };

export function EpidemiologyTab({ range }: { range: AnalyticsRange }) {
  const [ filter, setFilter ] = useState<EpiFilter>(NO_FILTER);
  const query: AnalyticsQuery = { ...rangeDates(range), granularity: range.granularity, ...filter };
  const result = useAnalytics("epidemiology", query);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div role="group" aria-label="Recortes" style={filterRowStyle}>
        <NeighborhoodSelect value={filter.neighborhood_id}
          onChange={(v) => setFilter((f) => ({ ...f, neighborhood_id: v }))} />
        <ProtocolSelect name={filter.protocol_name} version={filter.protocol_version} withVersion
          onChange={(name, version) => setFilter((f) => ({ ...f, protocol_name: name, protocol_version: version }))} />
      </div>
      <p style={noteStyle}>Respostas agregadas por bairro, nunca por pessoa. Só entram perguntas marcadas pelo autor do protocolo.</p>
      <AnalyticsView result={result}>{(data) => <EpidemiologyBody data={data} />}</AnalyticsView>
    </div>
  );
}

function EpidemiologyBody({ data }: { data: EpidemiologyData }) {
  if (data.questions.length === 0) {
    return (
      <Panel title="Epidemiologia">
        <EmptyState title={EPI_EMPTY_TITLE} />
        <p style={{ ...noteStyle, textAlign: "center", maxWidth: 560, margin: "0 auto" }}>{EPI_HOWTO}</p>
      </Panel>
    );
  }
  return (
    <>
      {data.questions.map((q) => (
        <SeriesBlock key={`${q.protocol_name}/${q.question_id}`} title={q.prompt}
          sub={`${q.protocol_name} · ${q.answer_type === "boolean" ? "sim/não" : "lista"}`}
          kind="count" periods={data.periods} granularity={data.granularity ?? "week"}
          lines={q.options.map((o) => ({ key: o.value, label: o.label, series: o.series, total: o.total }))} />
      ))}
    </>
  );
}

const noteStyle = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
```

- [ ] **Step 4: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/modules/analytics/EpidemiologyTab.test.tsx && npx tsc --noEmit`
Expected: PASS, 3 testes; tsc sem erros.

- [ ] **Step 5: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/modules/analytics/EpidemiologyTab.tsx src/modules/analytics/EpidemiologyTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the analytics epidemiology tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Raiz do Analytics, abas e menu por papel (F-14.2)

**Files:**
- Create: `src/modules/Analytics.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/Analytics.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 2: `AnalyticsRange`, `DEFAULT_RANGE`;
  - da Task 4: `RangeControls`;
  - das Tasks 5 a 8: `DemandTab`, `QualityTab`, `CalibrationTab`, `EpidemiologyTab`;
  - já existentes: `PageHeader`, `SegmentedControl` (`src/shell/SegmentedControl.tsx`, `role="tablist"`/`role="tab"`).
- Produces:
  - `ModuleId` ganha `"analytics"`;
  - grupo novo "Análise" com o item `{ id: "analytics", label: "Analytics", icon: "∿" }`, logo depois de "Triagem";
  - `navGroupsFor` mostra "Análise" só para `analyst` ou `municipal_admin` que não seja operador;
  - `Analytics()` — abas Demanda, Qualidade, Calibração, Epidemiologia; o intervalo é compartilhado entre as abas, e os recortes são de cada aba.

- [ ] **Step 1: Testes da navegação**

Em `src/shell/modules.test.ts`:

1. No teste "operador não vê o grupo Conta…", troque

```ts
      expect(groups.some((g) => g.label === "Comunicação")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 5);
```

por

```ts
      expect(groups.some((g) => g.label === "Comunicação")).toBe(false);
      expect(groups.some((g) => g.label === "Análise")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 6);
```

2. No teste "sem sessão, esconde Equipe, Atendimento e Cidade", troque

```ts
      expect(groups.some((g) => g.label === "Comunicação")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 4);
```

por

```ts
      expect(groups.some((g) => g.label === "Comunicação")).toBe(false);
      expect(groups.some((g) => g.label === "Análise")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 5);
```

3. Antes do `});` final do arquivo, acrescente:

```ts
  describe("módulo 14 na navegação", () => {
    const user = (roles: string[], operator = false) => ({ operator, memberships: roles.map((role) => ({ role })) });
    const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));

    it("Analytics para analyst e municipal_admin, e para ninguém mais", () => {
      expect(ids(user([ "analyst" ]))).toContain("analytics");
      expect(ids(user([ "municipal_admin" ]))).toContain("analytics");
      for (const role of [ "viewer", "citizen_verifier", "health_professional", "protocol_reviewer", "campaign_manager" ]) {
        expect(ids(user([ role ]))).not.toContain("analytics");
      }
      expect(ids(null)).not.toContain("analytics");
      expect(labelFor("analytics")).toBe("Analytics");
    });

    it("operador nunca vê Analytics, nem com papel na lista (D12)", () => {
      expect(ids(user([], true))).not.toContain("analytics");
      expect(ids(user([ "municipal_admin" ], true))).not.toContain("analytics");
    });

    it("analista não ganha Equipe, Território nem Campanhas", () => {
      expect(ids(user([ "analyst" ]))).not.toContain("team");
      expect(ids(user([ "analyst" ]))).not.toContain("territory");
      expect(ids(user([ "analyst" ]))).not.toContain("campaigns");
    });
  });
```

- [ ] **Step 2: Testes da raiz**

```tsx
// src/modules/Analytics.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { Analytics } from "./Analytics";
import {
  NOW, calibrationData, demandData, envelope, epidemiologyData, paramsOf, qualityData, renderWithQuery, stubAnalyticsApi
} from "../test/analyticsFixtures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(NOW);
});
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

function stubAll() {
  return stubAnalyticsApi({
    "/analytics/demand": envelope<"demand">(demandData()),
    "/analytics/quality": envelope<"quality">(qualityData()),
    "/analytics/calibration": envelope<"calibration">(calibrationData()),
    "/analytics/epidemiology": envelope<"epidemiology">(epidemiologyData())
  });
}

describe("Analytics", () => {
  it("abre na Demanda, com as quatro abas", async () => {
    const fn = stubAll();
    renderWithQuery(<Analytics />);
    expect(screen.getAllByRole("tab").map((t) => t.textContent)).toEqual([ "Demanda", "Qualidade", "Calibração", "Epidemiologia" ]);
    expect(screen.getByRole("tab", { name: "Demanda" }).getAttribute("aria-selected")).toBe("true");
    expect(await screen.findByRole("region", { name: "Triagens" })).toBeTruthy();
    expect(paramsOf(fn, "/analytics/quality")).toHaveLength(0);
  });

  it("cada aba pede a sua frente", async () => {
    const fn = stubAll();
    renderWithQuery(<Analytics />);
    fireEvent.click(screen.getByRole("tab", { name: "Qualidade" }));
    expect(await screen.findByRole("region", { name: "Espera até a chamada" })).toBeTruthy();
    fireEvent.click(screen.getByRole("tab", { name: "Calibração" }));
    expect(await screen.findByRole("region", { name: "arbovirose · versão 2" })).toBeTruthy();
    fireEvent.click(screen.getByRole("tab", { name: "Epidemiologia" }));
    expect(await screen.findByRole("region", { name: "Teve febre?" })).toBeTruthy();
    expect(paramsOf(fn, "/analytics/calibration")[0].has("granularity")).toBe(false);
  });

  it("o intervalo vale para todas as abas", async () => {
    const fn = stubAll();
    renderWithQuery(<Analytics />);
    fireEvent.change(screen.getByLabelText("Agrupar por"), { target: { value: "month" } });
    await waitFor(() => expect(paramsOf(fn, "/analytics/demand").at(-1)!.get("from")).toBe("2025-10-01"));
    fireEvent.click(screen.getByRole("tab", { name: "Qualidade" }));
    await waitFor(() => expect(paramsOf(fn, "/analytics/quality").at(-1)!.get("granularity")).toBe("month"));
    expect(paramsOf(fn, "/analytics/quality").at(-1)!.get("from")).toBe("2025-10-01");
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/shell/modules.test.ts src/modules/Analytics.test.tsx`
Expected: FAIL. `modules.test.ts` falha nas contagens e em "analytics" ausente; `Analytics.test.tsx` falha com `Failed to resolve import "./Analytics"`.

- [ ] **Step 4: Menu por papel**

Em `src/shell/modules.ts`:

1. Troque `  | "professionals" | "my-profile" | "territory" | "campaigns";` por `  | "professionals" | "my-profile" | "territory" | "campaigns" | "analytics";`.
2. Troque

```ts
    { id: "reports", label: "Relatórios", icon: "▤" }
  ]},
```

por

```ts
    { id: "reports", label: "Relatórios", icon: "▤" }
  ]},
  { label: "Análise", items: [
    { id: "analytics", label: "Analytics", icon: "∿" }
  ]},
```

3. Depois da linha `  const canCampaigns = isAdmin || roles.includes("campaign_manager");`, acrescente:

```ts
  // Módulo 14 (ADR 0025, D6/D12): Analytics é do analyst e do municipal_admin
  // da cidade; o operador, com ou sem grant, nunca (a API responde 403).
  const canAnalytics = !user?.operator && (isAdmin || roles.includes("analyst"));
```

4. Depois da linha `    if (group.label === "Comunicação") return canCampaigns;`, acrescente:

```ts
    if (group.label === "Análise") return canAnalytics;
```

- [ ] **Step 5: Raiz do módulo**

```tsx
// src/modules/Analytics.tsx
// Analytics da cidade (módulo 14; ADR 0025; spec §8): séries consolidadas até
// D-1, só para analyst e municipal_admin (o menu já filtra; a API responde 403
// para o resto). O intervalo é um só para as quatro abas; os recortes são de
// cada aba e zeram ao trocar de aba.
import { useState } from "react";
import type { AnalyticsFront } from "../lib/api";
import { DEFAULT_RANGE, type AnalyticsRange } from "../lib/analytics";
import { PageHeader } from "../components/PageHeader";
import { SegmentedControl } from "../shell/SegmentedControl";
import { RangeControls } from "./analytics/Filters";
import { DemandTab } from "./analytics/DemandTab";
import { QualityTab } from "./analytics/QualityTab";
import { CalibrationTab } from "./analytics/CalibrationTab";
import { EpidemiologyTab } from "./analytics/EpidemiologyTab";

const TABS: { key: AnalyticsFront; label: string }[] = [
  { key: "demand", label: "Demanda" },
  { key: "quality", label: "Qualidade" },
  { key: "calibration", label: "Calibração" },
  { key: "epidemiology", label: "Epidemiologia" }
];

export function Analytics() {
  const [ tab, setTab ] = useState<AnalyticsFront>("demand");
  const [ range, setRange ] = useState<AnalyticsRange>(DEFAULT_RANGE);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Analytics" sub="séries consolidadas · até o dia anterior" />
      <div style={{ display: "flex", gap: 16, alignItems: "flex-end", flexWrap: "wrap" }}>
        <SegmentedControl options={TABS} value={tab} onChange={setTab} />
        <RangeControls value={range} onChange={setRange} />
      </div>
      {tab === "demand" && <DemandTab range={range} />}
      {tab === "quality" && <QualityTab range={range} />}
      {tab === "calibration" && <CalibrationTab range={range} />}
      {tab === "epidemiology" && <EpidemiologyTab range={range} />}
    </div>
  );
}
```

- [ ] **Step 6: Rota no `App.tsx`**

1. Depois de `import { Campaigns } from "./modules/Campaigns";`, acrescente `import { Analytics } from "./modules/Analytics";`.
2. Depois de `    case "campaigns":      return <Campaigns onNavigate={setActive} />;`, acrescente `    case "analytics":      return <Analytics />;`.

- [ ] **Step 7: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/shell/modules.test.ts src/modules/Analytics.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos de navegação continuam verdes com as contagens novas); tsc sem erros.

- [ ] **Step 8: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/modules/Analytics.tsx src/modules/Analytics.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the analytics area to the city dashboard menu

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: "Usar em Analytics" no editor de protocolo (F-14.6)

**Files:**
- Modify: `src/lib/editor.ts`, `src/lib/editor.test.ts`, `src/modules/ProtocolEditor.tsx`
- Create: `src/modules/protocolEditor/AnalyticQuestions.tsx`
- Test: `src/modules/protocolEditor/AnalyticQuestions.test.tsx`, `src/modules/ProtocolEditor.test.tsx`

**Interfaces:**
- Consumes: `parseDefinition`, `TEMPLATE` (`src/lib/editor.ts`); `listAuthorProtocols`, `gateProtocol`, `saveProtocolDraft` (`src/lib/api.ts`).
- Produces (em `src/lib/editor.ts`):
  - `ANALYTIC_HINT = "respostas desta pergunta aparecerão agregadas por bairro, nunca por pessoa"`;
  - `AnalyticStep { id: string; prompt: string; answerType: string; analytic: boolean; eligible: boolean }`;
  - `analyticSteps(definition: unknown): AnalyticStep[]`;
  - `setAnalytic(definition: unknown, stepId: string, on: boolean): unknown` — `on` grava `analytic: true` só em `boolean`/`enum`; `off` apaga a chave;
  - `stripIneligibleAnalytic(definition: unknown): { definition: unknown; removed: string[] }`.
- Produces (tela): `AnalyticQuestions({ definition: unknown; onChange(next: unknown) })` — região "Perguntas para Analytics"; um `fieldset` por pergunta elegível, com a legenda igual ao `prompt`, a caixa "Usar em Analytics" e a dica.

Ver Desvio 1: o editor é JSON cru. A caixa reescreve o texto; a marca em pergunta que virou `integer`/`text` gera aviso na hora e sai ao salvar.

- [ ] **Step 1: Testes da lib**

No fim de `src/lib/editor.test.ts`, acrescente (e troque o import do topo por `import { analyticSteps, parseDefinition, setAnalytic, stripIneligibleAnalytic, TEMPLATE } from "./editor";`):

```ts
describe("perguntas analíticas (módulo 14)", () => {
  const def = {
    name: "arbovirose", version: 3, start_step_id: "febre",
    steps: [
      { id: "febre", prompt: "Teve febre?", answer_type: "boolean" },
      { id: "sintoma", prompt: "Qual o sintoma?", answer_type: "enum", options: [ "dor", "tosse" ], analytic: true },
      { id: "dias", prompt: "Há quantos dias?", answer_type: "integer", analytic: true },
      { id: "obs", prompt: "Observações", answer_type: "text" }
    ]
  };

  it("analyticSteps diz quem está marcado e quem pode ser", () => {
    expect(analyticSteps(def).map((s) => [ s.id, s.analytic, s.eligible ])).toEqual([
      [ "febre", false, true ], [ "sintoma", true, true ], [ "dias", true, false ], [ "obs", false, false ]
    ]);
    expect(analyticSteps(null)).toEqual([]);
    expect(analyticSteps({ steps: "x" })).toEqual([]);
  });

  // As perguntas de `def` têm formatos diferentes; lidas como registro soltas.
  const stepsOf = (d: unknown) => (d as { steps: Array<Record<string, unknown>> }).steps;

  it("setAnalytic marca boolean/enum, desmarca apagando a chave, e ignora integer/text", () => {
    expect(stepsOf(setAnalytic(def, "febre", true))[0])
      .toEqual({ id: "febre", prompt: "Teve febre?", answer_type: "boolean", analytic: true });
    expect("analytic" in stepsOf(setAnalytic(def, "sintoma", false))[1]).toBe(false);
    expect("analytic" in stepsOf(setAnalytic(def, "obs", true))[3]).toBe(false);
    expect(def.steps[0]).toEqual({ id: "febre", prompt: "Teve febre?", answer_type: "boolean" }); // não muta
  });

  it("stripIneligibleAnalytic tira a marca de integer/text e diz de quais", () => {
    const { definition, removed } = stripIneligibleAnalytic(def);
    expect(removed).toEqual([ "dias" ]);
    expect("analytic" in stepsOf(definition)[2]).toBe(false);
    expect(stepsOf(definition)[1].analytic).toBe(true);
  });

  it("sem nada a tirar, devolve a mesma definição", () => {
    const clean = { steps: [ { id: "febre", prompt: "Teve febre?", answer_type: "boolean", analytic: true } ] };
    expect(stripIneligibleAnalytic(clean)).toEqual({ definition: clean, removed: [] });
  });
});
```

- [ ] **Step 2: Testes do painel**

```tsx
// src/modules/protocolEditor/AnalyticQuestions.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { AnalyticQuestions } from "./AnalyticQuestions";
import { ANALYTIC_HINT } from "../../lib/editor";

afterEach(cleanup);

const def = {
  steps: [
    { id: "febre", prompt: "Teve febre?", answer_type: "boolean" },
    { id: "sintoma", prompt: "Qual o sintoma?", answer_type: "enum", options: [ "dor" ], analytic: true },
    { id: "dias", prompt: "Há quantos dias?", answer_type: "integer" },
    { id: "obs", prompt: "Observações", answer_type: "text" }
  ]
};

describe("AnalyticQuestions", () => {
  it("caixa só em boolean/enum, com a dica", () => {
    render(<AnalyticQuestions definition={def} onChange={vi.fn()} />);
    const fever = within(screen.getByRole("group", { name: "Teve febre?" }));
    expect((fever.getByLabelText("Usar em Analytics") as HTMLInputElement).checked).toBe(false);
    expect(fever.getByText(ANALYTIC_HINT)).toBeTruthy();
    const symptom = within(screen.getByRole("group", { name: "Qual o sintoma?" }));
    expect((symptom.getByLabelText("Usar em Analytics") as HTMLInputElement).checked).toBe(true);
    expect(screen.queryByRole("group", { name: "Há quantos dias?" })).toBeNull();
    expect(screen.queryByRole("group", { name: "Observações" })).toBeNull();
    expect(screen.getAllByLabelText("Usar em Analytics")).toHaveLength(2);
  });

  it("marcar devolve a definição com analytic: true na pergunta", () => {
    const onChange = vi.fn();
    render(<AnalyticQuestions definition={def} onChange={onChange} />);
    fireEvent.click(within(screen.getByRole("group", { name: "Teve febre?" })).getByLabelText("Usar em Analytics"));
    expect((onChange.mock.calls[0][0] as { steps: unknown[] }).steps[0]).toEqual(
      { id: "febre", prompt: "Teve febre?", answer_type: "boolean", analytic: true });
  });

  it("marca que sobrou em integer/text: aviso de que sai ao salvar", () => {
    const stranded = { steps: [ { id: "dias", prompt: "Há quantos dias?", answer_type: "integer", analytic: true } ] };
    render(<AnalyticQuestions definition={stranded} onChange={vi.fn()} />);
    expect(screen.getByRole("alert").textContent)
      .toBe("“Há quantos dias?” não é de sim/não nem de lista: a marca de Analytics sai ao salvar.");
  });

  it("JSON inválido ou sem perguntas: nada aparece", () => {
    const { container } = render(<AnalyticQuestions definition={null} onChange={vi.fn()} />);
    expect(container.textContent).toBe("");
  });
});
```

- [ ] **Step 3: Teste do editor**

```tsx
// src/modules/ProtocolEditor.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, listAuthorProtocols: vi.fn(), loadProtocolDefinition: vi.fn(), gateProtocol: vi.fn(),
    previewProtocol: vi.fn(), saveProtocolDraft: vi.fn()
  };
});

import * as api from "../lib/api";
import { ProtocolEditor } from "./ProtocolEditor";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

const DEF = {
  name: "arbovirose", version: 3, start_step_id: "febre",
  steps: [
    { id: "febre", prompt: "Teve febre?", answer_type: "boolean", branches: { true: "dias", false: null } },
    { id: "dias", prompt: "Há quantos dias?", answer_type: "integer", branches: {} }
  ]
};

const definitionBox = () => screen.getAllByRole("textbox")[0] as HTMLTextAreaElement;
const typeDefinition = (value: unknown) =>
  fireEvent.change(definitionBox(), { target: { value: JSON.stringify(value, null, 2) } });

describe("ProtocolEditor — Usar em Analytics", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    mocked(api.listAuthorProtocols).mockResolvedValue([]);
    mocked(api.gateProtocol).mockResolvedValue({ valid: true });
    mocked(api.saveProtocolDraft).mockResolvedValue({ name: "arbovirose", version: 3, status: "draft" });
  });

  it("marcar a caixa grava analytic: true no JSON da pergunta", () => {
    render(<ProtocolEditor />);
    typeDefinition(DEF);
    fireEvent.click(within(screen.getByRole("group", { name: "Teve febre?" })).getByLabelText("Usar em Analytics"));
    expect(JSON.parse(definitionBox().value).steps[0].analytic).toBe(true);
    expect(screen.queryByRole("group", { name: "Há quantos dias?" })).toBeNull();
  });

  it("desmarcar tira a chave do JSON", () => {
    render(<ProtocolEditor />);
    typeDefinition({ ...DEF, steps: [ { ...DEF.steps[0], analytic: true }, DEF.steps[1] ] });
    fireEvent.click(within(screen.getByRole("group", { name: "Teve febre?" })).getByLabelText("Usar em Analytics"));
    expect("analytic" in JSON.parse(definitionBox().value).steps[0]).toBe(false);
  });

  it("pergunta que virou integer perde a marca ao salvar, com aviso antes", async () => {
    render(<ProtocolEditor />);
    typeDefinition({ ...DEF, steps: [ { ...DEF.steps[0], answer_type: "integer", analytic: true }, DEF.steps[1] ] });
    expect(screen.getByRole("alert").textContent).toContain("“Teve febre?” não é de sim/não nem de lista");
    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    await waitFor(() => expect(api.saveProtocolDraft).toHaveBeenCalledTimes(1));
    const sent = mocked(api.saveProtocolDraft).mock.calls[0][0] as { steps: Array<Record<string, unknown>> };
    expect("analytic" in sent.steps[0]).toBe(false);
    expect(definitionBox().value).not.toContain("analytic");
    expect(await screen.findByText("Salvo: arbovirose@3 (draft)")).toBeTruthy();
  });

  it("marca válida vai inteira no rascunho", async () => {
    render(<ProtocolEditor />);
    typeDefinition({ ...DEF, steps: [ { ...DEF.steps[0], analytic: true }, DEF.steps[1] ] });
    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    await waitFor(() => expect(api.saveProtocolDraft).toHaveBeenCalledTimes(1));
    expect((mocked(api.saveProtocolDraft).mock.calls[0][0] as { steps: Array<{ analytic?: boolean }> }).steps[0].analytic).toBe(true);
  });
});
```

- [ ] **Step 4: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/editor.test.ts src/modules/protocolEditor src/modules/ProtocolEditor.test.tsx`
Expected: FAIL. `analyticSteps is not a function` na lib; `Failed to resolve import "./AnalyticQuestions"` no painel; no editor, nenhum grupo "Teve febre?".

- [ ] **Step 5: Lib do editor**

No fim de `src/lib/editor.ts`, acrescente:

```ts
// ─── Perguntas analíticas (módulo 14; ADR 0025; spec §7) ────────────────────
// `analytic: true` só vale em pergunta boolean/enum; a API recusa em
// integer/text. A marca é parte da versão e passa pelo ciclo assinado.
export const ANALYTIC_HINT = "respostas desta pergunta aparecerão agregadas por bairro, nunca por pessoa";
const ANALYTIC_TYPES = new Set([ "boolean", "enum" ]);

type Step = Record<string, unknown>;

export interface AnalyticStep { id: string; prompt: string; answerType: string; analytic: boolean; eligible: boolean }

function stepsOf(definition: unknown): Step[] | null {
  if (!definition || typeof definition !== "object") return null;
  const steps = (definition as { steps?: unknown }).steps;
  if (!Array.isArray(steps)) return null;
  return steps.filter((s): s is Step => !!s && typeof s === "object");
}

export function analyticSteps(definition: unknown): AnalyticStep[] {
  return (stepsOf(definition) ?? []).map((s) => {
    const answerType = typeof s.answer_type === "string" ? s.answer_type : "";
    const id = String(s.id ?? "");
    return {
      id, prompt: typeof s.prompt === "string" && s.prompt ? s.prompt : id, answerType,
      analytic: s.analytic === true, eligible: ANALYTIC_TYPES.has(answerType)
    };
  });
}

// Cópia rasa com a pergunta trocada; o resto da definição fica como está.
function mapSteps(definition: unknown, fn: (step: Step) => Step): unknown {
  const d = definition as Record<string, unknown>;
  return { ...d, steps: (d.steps as unknown[]).map((s) => (s && typeof s === "object" ? fn(s as Step) : s)) };
}

function withoutAnalytic(step: Step): Step {
  const next = { ...step };
  delete next.analytic;
  return next;
}

export function setAnalytic(definition: unknown, stepId: string, on: boolean): unknown {
  if (!stepsOf(definition)) return definition;
  return mapSteps(definition, (s) => {
    if (s.id !== stepId) return s;
    const next = withoutAnalytic(s);
    if (on && ANALYTIC_TYPES.has(String(s.answer_type))) next.analytic = true;
    return next;
  });
}

export function stripIneligibleAnalytic(definition: unknown): { definition: unknown; removed: string[] } {
  const removed = analyticSteps(definition).filter((s) => s.analytic && !s.eligible).map((s) => s.id);
  if (removed.length === 0) return { definition, removed };
  return {
    definition: mapSteps(definition, (s) => (removed.includes(String(s.id ?? "")) ? withoutAnalytic(s) : s)),
    removed
  };
}
```

- [ ] **Step 6: Painel "Perguntas para Analytics"**

```tsx
// src/modules/protocolEditor/AnalyticQuestions.tsx
// Caixa "Usar em Analytics" (módulo 14; spec §7), derivada do JSON do editor:
// só pergunta boolean/enum tem a caixa. Marca que sobrou numa pergunta que
// virou integer/text ganha aviso e sai ao salvar (ProtocolEditor.save).
import { ANALYTIC_HINT, analyticSteps, setAnalytic } from "../../lib/editor";

export function AnalyticQuestions({ definition, onChange }: { definition: unknown; onChange(next: unknown): void }) {
  const steps = analyticSteps(definition);
  if (steps.length === 0) return null;
  const eligible = steps.filter((s) => s.eligible);
  const stranded = steps.filter((s) => s.analytic && !s.eligible);

  return (
    <section aria-label="Perguntas para Analytics" style={{ marginTop: 12, display: "flex", flexDirection: "column", gap: 8 }}>
      <h3 style={{ fontSize: 14, margin: 0 }}>Perguntas para Analytics</h3>
      {eligible.length === 0 && (
        <p style={hintStyle}>Só perguntas de sim/não ou de lista podem ir para o Analytics.</p>
      )}
      {eligible.map((s) => (
        <fieldset key={s.id} style={{ border: "1px solid var(--rule)", borderRadius: 6, padding: "6px 10px", margin: 0 }}>
          <legend style={{ fontSize: 13 }}>{s.prompt}</legend>
          <label style={{ fontSize: 13, display: "inline-flex", gap: 6, alignItems: "center" }}>
            <input type="checkbox" checked={s.analytic}
              onChange={(e) => onChange(setAnalytic(definition, s.id, e.target.checked))} />
            Usar em Analytics
          </label>
          <p style={hintStyle}>{ANALYTIC_HINT}</p>
        </fieldset>
      ))}
      {stranded.map((s) => (
        <p key={s.id} role="alert" style={{ margin: 0, fontSize: 12.5, color: "var(--down)" }}>
          “{s.prompt}” não é de sim/não nem de lista: a marca de Analytics sai ao salvar.
        </p>
      ))}
      <p style={hintStyle}>A marca vale para a versão salva e passa pelas assinaturas como o resto do protocolo.</p>
    </section>
  );
}

const hintStyle = { margin: 0, fontSize: 12, color: "var(--ink3)" };
```

Nota: o `fieldset` com `legend` tem papel `group` e nome acessível igual à legenda, e é por ele que os testes acham cada pergunta. O texto do aviso é montado com template literal no JSX (`“{s.prompt}” …`), o que gera três nós de texto; o teste lê `textContent`, que os junta.

- [ ] **Step 7: Editor**

Em `src/modules/ProtocolEditor.tsx`:

1. Troque `import { parseDefinition, TEMPLATE } from "../lib/editor";` por:

```ts
import { parseDefinition, stripIneligibleAnalytic, TEMPLATE } from "../lib/editor";
import { AnalyticQuestions } from "./protocolEditor/AnalyticQuestions";
```

2. Troque a função `save` inteira

```ts
  function save() {
    const parsed = parseDefinition(text);
    if (!parsed.ok) return;
    saveProtocolDraft(parsed.value).then(setSaved);
  }
```

por

```ts
  function save() {
    const parsed = parseDefinition(text);
    if (!parsed.ok) return;
    // Módulo 14 (spec §7): `analytic` só vale em boolean/enum. Pergunta que
    // virou integer/text perde a marca aqui, e o texto passa a mostrar o que
    // foi salvo (Desvio 1 do plano: o editor é JSON, sem seletor de tipo).
    const { definition, removed } = stripIneligibleAnalytic(parsed.value);
    if (removed.length > 0) setText(JSON.stringify(definition, null, 2));
    saveProtocolDraft(definition).then(setSaved);
  }

  const current = parseDefinition(text);
```

3. Troque

```tsx
        <button onClick={save} style={{ marginTop: 12 }}>Salvar rascunho</button>
```

por

```tsx
        <AnalyticQuestions
          definition={current.ok ? current.value : null}
          onChange={(next) => setText(JSON.stringify(next, null, 2))}
        />
        <button onClick={save} style={{ marginTop: 12 }}>Salvar rascunho</button>
```

- [ ] **Step 8: Rode e cheque tipos**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/editor.test.ts src/modules/protocolEditor src/modules/ProtocolEditor.test.tsx src/modules/Protocols.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos do editor e de Protocolos continuam verdes); tsc sem erros.

- [ ] **Step 9: Commit**

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/lib/editor.ts src/lib/editor.test.ts src/modules/protocolEditor/AnalyticQuestions.tsx \
  src/modules/protocolEditor/AnalyticQuestions.test.tsx src/modules/ProtocolEditor.tsx src/modules/ProtocolEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: mark boolean and enum protocol questions for analytics in the editor

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Papel "Análise" em Equipe, sem step-up (F-14.2)

**Files:**
- Modify: `src/lib/team.ts`, `src/lib/team.test.ts`, `src/modules/Team.tsx`, `src/modules/Team.test.tsx`

**Interfaces:**
- Consumes: `grantRole`, `revokeMembership`, `inviteMember` e o `SensitiveAction`, que já existem.
- Produces:
  - `ANALYST_ROLE = "analyst"` em `src/lib/team.ts`;
  - `TeamMember` ganha `isAnalyst: boolean` e `analystMembershipId: string | null`;
  - `INVITE_ROLES` passa a 9 papéis, com `{ role: "analyst", label: "Análise" }`;
  - `isPrivilegedRole("analyst") === false`;
  - em Equipe, a coluna "Análise" com "Tornar analista" e "Remover analista", ambos com `requiresStepUp={false}`.

- [ ] **Step 1: Testes da lib**

Em `src/lib/team.test.ts`:

1. Troque o teste "oferece os 8 papéis da cidade" inteiro por:

```ts
  it("oferece os 9 papéis da cidade", () => {
    expect(INVITE_ROLES.map((r) => r.role).sort()).toEqual([
      "analyst", "campaign_manager", "citizen_verifier", "health_professional", "municipal_admin", "protocol_author",
      "protocol_publisher", "protocol_reviewer", "viewer"
    ]);
    expect(INVITE_ROLES.find((r) => r.role === "campaign_manager")?.label).toBe("Gestor de campanhas");
    expect(INVITE_ROLES.find((r) => r.role === "analyst")?.label).toBe("Análise");
  });
```

2. No teste "papéis privilegiados são os mesmos que a API protege com step-up", troque `    for (const role of [ "viewer", "protocol_author", "protocol_publisher" ]) {` por `    for (const role of [ "viewer", "protocol_author", "protocol_publisher", "analyst" ]) {`.

3. No fim do arquivo, acrescente:

```ts
describe("analista", () => {
  it("teamMembers marca o papel e guarda o id da membership", () => {
    const [ ana ] = teamMembers([ row("ana@cidade.gov.br", "viewer"), row("ana@cidade.gov.br", "analyst", "m-an") ]);
    expect(ana.isAnalyst).toBe(true);
    expect(ana.analystMembershipId).toBe("m-an");
  });
});
```

- [ ] **Step 2: Testes da tela**

Em `src/modules/Team.test.tsx`, antes do `});` que fecha `describe("Team")` (a última linha do arquivo), acrescente:

```tsx
  describe("analista (módulo 14)", () => {
    it("tornar e remover analista sem código, mesmo com a janela de step-up fechada", async () => {
      mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_verified_at: null }));
      mocked(api.listMemberships).mockResolvedValue([
        membership("ana@cidade.gov.br", "viewer"),
        membership("bia@cidade.gov.br", "analyst", "m-bia-an")
      ]);
      mocked(api.grantRole).mockResolvedValue(undefined);
      mocked(api.revokeMembership).mockResolvedValue(undefined);
      renderTeam();

      fireEvent.click(await screen.findByRole("button", { name: "Tornar analista" }));
      expect(screen.queryByLabelText("Código do autenticador")).toBeNull();
      fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
      await waitFor(() => expect(api.grantRole).toHaveBeenCalledWith("u-ana@cidade.gov.br", "analyst"));
      expect((await screen.findByRole("status")).textContent).toBe("ana@cidade.gov.br agora é analista");

      fireEvent.click(await screen.findByRole("button", { name: "Remover analista" }));
      expect(screen.queryByLabelText("Código do autenticador")).toBeNull();
      fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
      await waitFor(() => expect(api.revokeMembership).toHaveBeenCalledWith("m-bia-an"));
      expect((await screen.findByRole("status")).textContent).toBe("bia@cidade.gov.br não é mais analista");
      expect(api.stepUpMfa).not.toHaveBeenCalled();
    });

    it("convite como Análise não pede código", async () => {
      mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_verified_at: null }));
      mocked(api.listMemberships).mockResolvedValue([]);
      renderTeam();

      fireEvent.change(await screen.findByLabelText("E-mail da pessoa"), { target: { value: "dados@cidade.gov.br" } });
      fireEvent.change(screen.getByLabelText("Papel"), { target: { value: "analyst" } });
      fireEvent.click(screen.getByRole("button", { name: "Convidar" }));
      expect(await screen.findByText("Convidar dados@cidade.gov.br como Análise.")).toBeTruthy();
      expect(screen.queryByLabelText("Código do autenticador")).toBeNull();
    });
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/team.test.ts src/modules/Team.test.tsx`
Expected: FAIL. `analyst` não está em `INVITE_ROLES`, e o botão "Tornar analista" não existe.

- [ ] **Step 4: `team.ts`**

1. Depois de `export const PROFESSIONAL_ROLE = "health_professional";`, acrescente:

```ts
// Módulo 14 (ADR 0025): só leitura do Analytics, fora de PRIVILEGED_ROLES.
export const ANALYST_ROLE = "analyst";
```

2. Em `TeamMember`, depois de `  campaignManagerMembershipId: string | null;`, acrescente:

```ts
  isAnalyst: boolean;
  analystMembershipId: string | null;
```

3. No objeto inicial de `teamMembers`, troque `isCampaignManager: false, campaignManagerMembershipId: null` por `isCampaignManager: false, campaignManagerMembershipId: null, isAnalyst: false, analystMembershipId: null`.
4. Depois do bloco `if (row.role === CAMPAIGN_MANAGER_ROLE) { … }`, acrescente:

```ts
    if (row.role === ANALYST_ROLE) {
      current.isAnalyst = true;
      current.analystMembershipId = row.id;
    }
```

5. Troque `// Os 8 papéis da cidade (Membership::ROLES na API), na ordem do seletor do` por `// Os 9 papéis da cidade (Membership::ROLES na API), na ordem do seletor do`. Em `INVITE_ROLES`, depois de `  { role: "viewer", label: "Leitura (viewer)" },`, acrescente:

```ts
  { role: ANALYST_ROLE, label: "Análise" },
```

`PRIVILEGED_ROLES` **não** muda.

- [ ] **Step 5: `Team.tsx`**

1. No import de `../lib/team`, troque `INVITE_ROLES, PROFESSIONAL_ROLE,` por `ANALYST_ROLE, INVITE_ROLES, PROFESSIONAL_ROLE,`.
2. Troque `type Pending = { member: TeamMember; kind: "grant" | "revoke"; role: "reviewer" | "verifier" | "professional" | "campaign" };` por:

```ts
type Pending = { member: TeamMember; kind: "grant" | "revoke"; role: "reviewer" | "verifier" | "professional" | "campaign" | "analyst" };
```

3. Troque `<PageHeader title="Equipe" sub="papéis · revisores de protocolo · atendentes · campanhas" />` por `<PageHeader title="Equipe" sub="papéis · revisores de protocolo · atendentes · campanhas · análise" />`.
4. Nas `cols` do `DataTable`, depois da coluna "Campanhas", acrescente:

```tsx
                  { label: "Análise", w: "auto", align: "right", render: (m) => (
                    m.isAnalyst
                      ? <button type="button" style={buttonStyle} onClick={() => open(m, "revoke", "analyst")}>Remover analista</button>
                      : <button type="button" style={buttonStyle} onClick={() => open(m, "grant", "analyst")}>Tornar analista</button>
                  ) },
```

5. Depois do bloco `{pending && pending.role === "campaign" && ( … )}`, acrescente:

```tsx
      {pending && pending.role === "analyst" && (
        pending.kind === "grant" ? (
          // Sem step-up: analyst não é papel privilegiado (contratos §5).
          <SensitiveAction
            title="Tornar analista"
            description={`${pending.member.email} poderá ler o Analytics da cidade: séries agregadas, sem dado de pessoa.`}
            requiresStepUp={false}
            run={async () => { await grantRole(pending.member.userId, ANALYST_ROLE); }}
            onDone={() => finish(`${pending.member.email} agora é analista`)}
            onCancel={() => setPending(null)}
            onGoToSecurity={() => onNavigate("security")}
          />
        ) : (
          // O `!` é seguro: "Remover analista" só existe quando isAnalyst é
          // true, e teamMembers preenche os dois juntos.
          <SensitiveAction
            title="Remover analista"
            description={`${pending.member.email} deixa de ler o Analytics da cidade.`}
            requiresStepUp={false}
            run={async () => { await revokeMembership(pending.member.analystMembershipId!); }}
            onDone={() => finish(`${pending.member.email} não é mais analista`)}
            onCancel={() => setPending(null)}
            onGoToSecurity={() => onNavigate("security")}
          />
        )
      )}
```

6. No comentário do topo, troque `// Escopo: conceder e revogar revisor, atendente, profissional de saúde e gestor de campanhas;` por `// Escopo: conceder e revogar revisor, atendente, profissional de saúde, gestor de campanhas e analista;` e `// convidar pessoa com qualquer um dos 8 papéis` por `// convidar pessoa com qualquer um dos 9 papéis`.

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run src/lib/team.test.ts src/modules/Team.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos de Equipe continuam verdes); tsc sem erros.

```bash
cd apps/dashboard/.claude/mod14
/opt/homebrew/bin/git add src/lib/team.ts src/lib/team.test.ts src/modules/Team.tsx src/modules/Team.test.tsx
/opt/homebrew/bin/git commit -m "feat: grant and invite the analyst role without step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod14 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI (`.github/workflows/ci.yml`) roda os mesmos três. `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git status --short
```

Expected: vazio. O symlink `node_modules` não aparece porque é ignorado. Se aparecer, **não** o adicione.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§7, §8 e §10.3), o ADR 0025 e o arquivo de contratos. Pontos de atenção:
- nenhuma soma, subtração, média ou taxa calculada no cliente sobre `Cell`/`Rate` (procure `reduce`, `+`, `/` em `src/modules/analytics/`);
- `{ suppressed: true }` sempre como "oculto", taxa `null` sempre como "sem dado", nunca `0` nem `—`;
- `granularity` nunca vai em `calibration`; `protocol_version` nunca vai sem `protocol_name`; recorte vazio nunca vai como `""`;
- "dados até" e o intervalo saem do dia da cidade, nunca de `toISOString()` direto;
- o menu esconde "Análise" do operador e de quem não é `analyst`/`municipal_admin`;
- o Analytics não lê nem escreve o `?bairro=` da URL (o filtro dos painéis do módulo 11 continua separado);
- o editor nunca manda `analytic` em `integer`/`text`;
- conceder, revogar e convidar `analyst` vão sem step-up; os papéis privilegiados continuam com;
- nenhum `git add -A` no histórico (`git log --stat origin/main..HEAD` sem `node_modules`).

- [ ] **Step 3: Prova no navegador (com o usuário; spec §10.4)**

Rode o api da branch do módulo 14 a partir do worktree dele, na porta `:3031`, com a semente do Analytics aplicada e o `city:analytics:rebuild` rodado. Depois rode o Vite deste worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod14 && VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5179 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5179/dashboard/`. O usuário faz o login. Não digite senha nem TOTP. Confira com screenshot:
- como `analise@curitiba.demo`:
  - o menu tem "Análise → Analytics" e não tem Equipe nem Território;
  - as quatro abas com a semente, o carimbo "dados até" de ontem e nenhum aviso de desatualizado;
  - filtrar por um bairro pequeno até aparecer "oculto" numa tabela e lacuna no gráfico;
  - Calibração com uma versão do protocolo de dev e as proporções por linha;
  - Epidemiologia com as duas perguntas marcadas da semente (`boolean` e `enum`);
- como `admin@curitiba.demo`:
  - o mesmo Analytics;
  - em Equipe, a coluna "Análise": conceder e remover sem pedir código; o convite com "Análise";
  - no Editor de protocolo, abrir um rascunho, marcar "Usar em Analytics" numa pergunta de sim/não e ver a dica; trocar o tipo de uma pergunta marcada para `integer` no JSON e ver o aviso; salvar e ver a marca sumir do texto.

- [ ] **Step 4:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é api → dashboard → admin → maintenance (spec §12).

---

## Self-review

- **Cobertura da spec:**
  - §7, protocolo no editor: caixa só em `boolean`/`enum`, dica literal, marca fora em `integer`/`text` ao salvar — Task 10 (Desvio 1).
  - §8, dashboard:
    - item de menu para `analyst`/`municipal_admin`: Task 9;
    - quatro abas: Tasks 5 a 9;
    - seletores de período e granularidade: Tasks 2, 4 e 9;
    - recortes por frente (bairro, unidade, protocolo, versão): Tasks 4 a 8, com o seletor de unidade vindo de `data.units`;
    - gráfico de série e tabela: Task 3 (`SeriesBlock`), usado nas Tasks 5, 6 e 8;
    - "oculto", "dados até DD/MM", "dados desatualizados", "ainda sem dados consolidados": Tasks 2 e 3;
    - calibração tier × desfecho por versão, com proporção por linha: Task 7;
    - epidemiologia, uma série por opção, e texto de como marcar: Task 8;
    - Equipe, papel `analyst` sem step-up: Task 11.
  - §6.1/contratos §1: parâmetros, limites (104 semanas, 60 meses), envelope com `stale`, recusas 403/422 — Tasks 1, 2 e 3.
  - §10.3: "oculto" (Tasks 3, 5, 6, 7, 8), carimbo e `stale` (Task 3), estado vazio (Task 3), recortes (Tasks 4 a 8), caixa do editor só em boolean/enum (Task 10), papel em Equipe (Task 11).
  - §10.4, prova no navegador: Task 12.
- **Placeholders:** nenhum. As Tasks 9, 10 e 11 alteram arquivos existentes por trechos exatos (de/para), com o código inteiro do trecho novo.
- **Consistência de nomes:**
  - `Cell`, `Rate`, `AnalyticsFront`, `Granularity`, `AnalyticsQuery`, `AnalyticsEnvelope`, `AnalyticsDataMap`, `AnalyticsUnit` e os `*Data` saem da Task 1 e são usados nas Tasks 2 a 8.
  - `rangeDates`, `AnalyticsRange`, `DEFAULT_RANGE`, `withGranularity`, `protocolOptions`, `analyticsKey`, `analyticsError` e os rótulos saem da Task 2.
  - `useAnalytics`, `AnalyticsView`, `SeriesBlock` (props `title`, `sub`, `kind`, `periods`, `granularity`, `lines`, `empty`), `CellValue`, `RateValue` e `Value` saem da Task 3.
  - `RangeControls`, `NeighborhoodSelect`, `UnitSelect`, `ProtocolSelect` (`onChange(name, version)`) e `filterRowStyle` saem da Task 4.
  - `DemandTab`, `QualityTab`, `CalibrationTab` e `EpidemiologyTab` recebem `{ range: AnalyticsRange }` e são montados na Task 9.
  - `ANALYST_ROLE` sai da Task 11; `navGroupsFor` usa o literal `"analyst"`, como já faz com os outros papéis.
  - Os nomes de região que os testes usam são os `title` passados aos `Panel`/`SeriesBlock` na mesma task.
- **Review Focus:** cada um dos cinco itens tem teste na task dona:
  - 1: Tasks 3 e 5;
  - 2: Task 2;
  - 3: Tasks 4, 5 e 6;
  - 4: Tasks 1, 4 e 7;
  - 5: Task 10.
