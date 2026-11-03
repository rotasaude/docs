# Módulo 30 — Transporte sanitário e TFD (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 30** na branch `feat/mod-30-sanitary-transport` (worktree `apps/api/.claude/mod30`, porta **3052**) com `GET /citizen/trips` (contrato §8, na forma das Divergências W1–W4 e W8), o tipo `trip` em `GET /citizen/notices` (§8, W5), `sanitary_transport` e `tfd` em `features` da sessão do cidadão (W3) e a fonte `transport` em `/citizen/history` (§9, W7). Ordem (contrato §13): api → dashboard → **wpda** → maintenance. As Tasks 0–7 não dependem dele (dublês e Playwright com rotas respondidas); a prova da Task 8 depende.

**Goal:** Lado wpda de F-30.8: a tela **"Minhas viagens"** — próximas viagens e anteriores com data, hora de saída, local de embarque, destino (prestador e cidade), situação e acompanhante, e, no TFD, a situação do processo, o atendimento no destino e a ajuda de custo devida (paga ou a receber) —, o botão em "Minhas triagens", o **aviso de viagem** na caixa de avisos (entrada na viagem, mudança de saída ou de embarque, cancelamento, retirada e véspera), a explicação do SMS de viagem nas preferências e a entrada `transport` em "Meu histórico".

**Architecture:** O cliente (`src/lib/citizenApi.ts`) ganha `trips(citizenId)` e o tipo `TripNotice` em `normalizeNotice`, com a normalização na borda: só os campos do §8 sobrevivem (procedimento, CID, especialidade, veículo, motorista, endereço e o TFD em `draft` são descartados; o aviso perde o destino). Regras de tela puras em `src/lib/trips.ts`. Peça nova: `src/modules/citizen/MyTripsStep.tsx`, no molde do `MyVaccinesStep` (leitura num `useEffect` com `alive`, sem polling). Mudam o mínimo: `HistoryStep` (botão), `Flow` (estado `trips`), `NoticesStep` (item e detalhe do aviso de viagem), `PreferencesStep` (uma frase), `src/lib/history.ts` (rótulo e defesa da entrada `transport`), `README.md`.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `src/test/setup.ts` com `cleanup` e `stubDevice`), `vi.spyOn(citizenApi, …)`, Playwright (5 perfis, `e2e/`), Vite 5 (base `/wpda/`).

**Spec:** `docs/superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md` (§1 sucesso "o cidadão sabe pelo wpda quando e onde embarcar" e antiobjetivo "lista de passageiros com CID ou diagnóstico", §2 decisões 14, 21, 22, 23 e as decisões do coordenador "Avisos" e "Exclusão LGPD", §4 "wpda", §5, §6, §8 F-30.8), `docs/adr/0048.md` e o contrato `docs/superpowers/plans/2026-11-03-module-30-sanitary-transport-contracts.md` §1, §8, §9, §11 e §13 (diferenças em W1–W8). Herdados: ADR 0017 (par CPF + celular, validação presencial), ADR 0024 (caixa de avisos), ADR 0035 (linha do tempo), módulo 17 (lembrete e SMS), plano do wpda do 29.

## Global Constraints

- Rota (§8 com W1–W4, W8): `GET /citizen/trips?citizen_id=` → `{ "items": [ { "id", "date", "departure_at", "boarding": { "label" }, "destination": { "name", "provider_name"|null }, "status", "companion": bool } ], "tfd": [ { "id", "status", "destination_name", "appointment_at"|null, "allowance_total_cents"|null, "paid": bool } ] }`; 403 `not_verified` para par só declarado; 403 `feature_disabled` com `sanitary_transport` desligado; 401 dispara `citizen:unauthenticated`.
- **Nada clínico nem da escala na tela** (spec decisão 14, §5; contrato §8): nunca procedimento, especialidade, CID, justificativa, laudo, veículo, placa ou motorista. O cliente descarta campo fora do contrato; o TFD em `draft` nunca chega à tela.
- Aviso de viagem (W5): só data, hora de saída e local de embarque — **nunca destino nem prestador** (a caixa mostra avisos de todas as pessoas do celular, inclusive pares só declarados; igual ao texto fixo do SMS do §8).
- Situações em palavras do cidadão; código desconhecido vira "Em andamento", nunca o código cru.
- `date` é de calendário (`AAAA-MM-DD`): `fmtCalendarDate`, nunca `new Date`; "hoje" é `cityToday()`. `departure_at` e `appointment_at` são instantes com fuso: `fmtWeekdayDateTime` (fuso da cidade). Dinheiro em centavos inteiros: `R$ 1.234,56`; valor inválido não aparece.
- Na URL só `citizen_id`. Nada em `localStorage`, `sessionStorage` ou `console`. **Sem polling**: lê ao abrir a tela.
- Interruptores: sem `sanitary_transport` em `session.features`, "Minhas triagens" fica como hoje; o bloco do TFD só aparece quando `tfd` vem com itens.
- Interface do canal (subprojetos 4 e 6e): `Screen`, `BigButton`, `ErrorText` de `ui.tsx`; alvos de 48 px, texto de 18 px; cores só por papel do tema (`var(--warn-bg)`, `var(--ok-bg)`, `var(--accent-bg)`, `var(--rule2)`, `var(--ink2)`), sem reserva e sem branco fixo — `src/theme/guards.test.ts` na suíte; fluxo só no celular (`isPhone`) e `e2e/device-gate.spec.ts` verde. Nenhum componente comum muda.
- Testes com "agora" fixam o relógio (`vi.useFakeTimers({ toFake: [ "Date" ] })`, `2026-11-03T10:00:00-03:00`, terça-feira); a suíte roda em `TZ=America/Sao_Paulo`; o Playwright em `timezoneId: "UTC"`.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits por extenso, `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git `/opt/homebrew/bin/git`. Merge, push e tag só com autorização.
- Board: primeira ação, F-30.1..F-30.8 do board #1 em In Progress (Task 0). O wpda é dono de F-30.8 com o api e o dashboard: Done quando o último dos três terminar; Verified depois da prova (Task 8).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod30`, `G=/opt/homebrew/bin/git`. Worktree **fora de `apps/`**, branch `feat/mod-30-sanitary-transport`; caminhos relativos a `$W`.
- Testes: `bin/test wpda -w wpda-mod30 <arquivos>`; tipos `cd $W && npx tsc --noEmit`; e2e `cd $W && WPDA_URL=http://curitiba.localhost:5210 npx playwright test` (Vite da Task 7 no ar).
- O api do 30 roda em dev na **3052** no container `api-dev` (rede `rota-saude_default`, apelido `api`); o Vite do worktree sobe num container dessa rede na **5210**, `node_modules` em volume próprio. `/citizen` já está no proxy. Banco de dev só migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Api que mande dado clínico ou da escala** (procedimento, CID, especialidade, placa, motorista, endereço do prestador, laudo do TFD). Nada disso aparece. Testes: Task 1, "trips: só os campos do contrato"; Task 3, "nada clínico nem da escala na tela".
2. **Aviso de viagem com destino ou procedimento** (api antigo ou bug). A caixa mostra só data, hora e embarque. Testes: Task 5, "notices: aviso de viagem perde destino e prestador"; Task 5, "aviso de viagem: abre com data, hora e embarque, sem destino".
3. **Situação interna ou nova** (TFD `draft`, `foo`; viagem `boarding`). O `draft` some; código desconhecido vira "Em andamento". Testes: Task 1, "trips: TFD em draft some"; Task 2, "tripStatusView e tfdView: tabela e desconhecido".
4. **Viagem de hoje depois da hora de saída, hora inválida e fuso do aparelho.** Viagem de hoje fica em "Próximas" até `done`; sem `departure_at` válido, "Saída em dd/mm/aaaa — a Secretaria informa a hora"; aparelho em UTC mostra a hora da cidade. Testes: Task 2, "splitTrips: hoje fica nas próximas até realizada" e "departureText: hora válida, nula e inválida"; Task 7 (perfil em UTC).
5. **Ajuda de custo ausente, zero, fracionária ou alta.** Nula ou inválida não aparece; zero é "R$ 0,00"; 123456 é "R$ 1.234,56"; `paid` decide "paga"/"a receber". Testes: Task 1, "trips: valores inválidos viram seguros"; Task 2, "allowanceText: nula, zero, alta, paga e a receber".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, rotas do api, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.trips.test.ts` | `trips` e a normalização | 1 |
| `src/lib/trips.ts`, `src/lib/trips.test.ts` | frases e regras de tela | 2 |
| `src/modules/citizen/MyTripsStep.tsx`, `src/modules/citizen/MyTripsStep.test.tsx` | a tela "Minhas viagens" | 3 |
| `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.trips.test.tsx`, `README.md` | botão, estado `trips`, README | 4 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.tripNotices.test.ts`, `src/modules/citizen/NoticesStep.tsx`, `src/modules/citizen/NoticesStep.trips.test.tsx`, `src/modules/citizen/PreferencesStep.tsx`, `src/modules/citizen/PreferencesStep.test.tsx` | aviso de viagem na caixa; frase do SMS de viagem | 5 |
| `src/lib/history.ts`, `src/lib/history.test.ts` | entrada `transport` no histórico | 6 |
| `e2e/trips.spec.ts` | "Minhas viagens" e o aviso nos celulares | 7 |
| — | suíte, build, e2e, revisão, prova | 8 |

**Estratégia de teste:** cliente com `fetch` falso (URL só com `citizen_id`, campos a mais descartados, `draft` some, valores inválidos seguros) — Tasks 1 e 5; regras puras com tabela de casos (situações, datas, hora, dinheiro, divisão próximas/anteriores) — Task 2; a tela com viagens, TFD, par declarado, vazio e interruptor caindo — Task 3; navegação real pelo `Flow` com e sem o interruptor — Task 4; o aviso na caixa (item, detalhe, evento desconhecido, várias pessoas) e a frase das preferências — Task 5; rótulo do histórico — Task 6; Playwright nos celulares em UTC, 375–412 px sem rolagem lateral — Task 7; prova contra o api do 30 — Task 8.

---

### Task 0: Cards, âncoras, rotas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`44b2533` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `apps/api/.claude/mod30`.
- Produces: cards em In Progress; `$W`; a base; a forma real das rotas anotada.

- [ ] **Step 1: Cards F-30.1..F-30.8 para In Progress** — um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-30\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito em In Progress ou além. Card ausente → **pare e reporte**.

- [ ] **Step 2: Âncoras e rotas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '^// ─── Minhas vacinas (módulo 29\|^  vaccinations: async\|^export type Notice = \|^function normalizeNotice\|^export interface CitizenSession'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/HistoryStep.tsx | grep -n 'onVaccines'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n '{ at: "vaccines"\|case "vaccines":\|onVaccines={hasSessionFeature'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/NoticesStep.tsx | grep -n 'REMINDER_HINT =\|current.kind === "appointment_reminder"\|notice.kind === "appointment_reminder"'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/PreferencesStep.tsx | grep -n 'REMINDER_EXPLANATION'
$G -C $R/apps/wpda show origin/main:src/lib/history.ts | grep -n '  vaccination: "Vacina"\|^export function entryPlace\|^export function entryStatusText'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^\*\*Minhas vacinas (módulo 29).\*\*\|^### Relatório público'
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod30 api bin/rails routes 2>/dev/null | grep -E "citizen/(trips|notices|history|session)"
```

Expected: as âncoras e as rotas. Forma diferente da rota (envelope, nomes, `citizen_id`, campos do aviso `trip`) vira ajuste no `normalizeTrips`/`normalizeNotice` e nas fixtures das Tasks 1–7 **antes** de começar, com nota para a Task 8. Se a sessão do cidadão não listar `sanitary_transport`/`tfd` (W3), avise a sessão do api.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod30 -b feat/mod-30-sanitary-transport origin/main
cd $R/.worktrees/wpda-mod30 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod30
```

Expected: verde e `tsc-ok`. **Anote** a base: a Task 8 espera **+6 arquivos de teste** (`citizenApi.trips`, `trips`, `MyTripsStep`, `Flow.trips`, `citizenApi.tripNotices`, `NoticesStep.trips`) e casos novos em `history.test.ts` e `PreferencesStep.test.tsx`.

---

### Task 1: Cliente — `trips`

**Files:**
- Modify: `src/lib/citizenApi.ts` (bloco novo depois do de Minhas vacinas; método depois de `vaccinations`)
- Test: `src/lib/citizenApi.trips.test.ts`

**Interfaces:**
- Consumes: `call`, `str`, `strOrNull` (existentes no bloco do 29).
- Produces:
  - `type TripStatus = "planned" | "confirmed" | "in_progress" | "done" | "cancelled"`;
  - `interface CitizenTrip { id: string; date: string; departure_at: string | null; boarding_label: string | null; destination_name: string; provider_name: string | null; status: TripStatus | string; companion: boolean }`;
  - `type TfdStatus = "submitted" | "returned" | "authorized" | "denied" | "scheduled" | "travelling" | "completed" | "cancelled"`;
  - `interface CitizenTfdCase { id: string; status: TfdStatus | string; destination_name: string; appointment_at: string | null; allowance_total_cents: number | null; paid: boolean }`;
  - `interface CitizenTrips { items: CitizenTrip[]; tfd: CitizenTfdCase[] }`;
  - `citizenApi.trips(citizenId: string): Promise<CitizenTrips>`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.trips.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function stubFetch(status: number, body: unknown) {
  const fetchMock = vi.fn(async () => new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

const RAW_TRIP = { id: "t1", date: "2026-11-05", departure_at: "2026-11-05T05:30:00-03:00",
  boarding: { label: "Praça da Matriz" }, destination: { name: "Curitiba", provider_name: "Hospital Erasto Gaertner" },
  status: "confirmed", companion: true };
const TRIP = { id: "t1", date: "2026-11-05", departure_at: "2026-11-05T05:30:00-03:00", boarding_label: "Praça da Matriz",
  destination_name: "Curitiba", provider_name: "Hospital Erasto Gaertner", status: "confirmed", companion: true };
const TFD = { id: "c1", status: "authorized", destination_name: "Curitiba", appointment_at: "2026-11-12T09:00:00-03:00",
  allowance_total_cents: 12345, paid: false };

describe("citizenApi.trips (contrato §8)", () => {
  it("trips: GET com só o citizen_id na URL", async () => {
    const fetchMock = stubFetch(200, { items: [], tfd: [] });
    await citizenApi.trips("p 1");
    expect(fetchMock.mock.calls[0][0]).toBe("/citizen/trips?citizen_id=p%201");
    expect((fetchMock.mock.calls[0][1] as RequestInit).method).toBe("GET");
  });

  it("trips: só os campos do contrato", async () => {
    stubFetch(200, {
      items: [ { ...RAW_TRIP, procedure: "Radioterapia", cid10: "C50", specialty: "Oncologia",
        vehicle: { plate: "ABC1D23" }, driver: { name: "João Motorista" },
        destination: { ...RAW_TRIP.destination, address: "Rua Dr. Ovande do Amaral, 201" } } ],
      tfd: [ { ...TFD, report: { cid10: "C50", justification: "sem radioterapia no município" }, sigtap_code: "0304010022" } ]
    });
    expect(await citizenApi.trips("p1")).toEqual({ items: [ TRIP ], tfd: [ TFD ] });
  });

  it("trips: TFD em draft some", async () => {
    stubFetch(200, { items: [], tfd: [ { ...TFD, id: "d1", status: "draft" }, TFD ] });
    expect((await citizenApi.trips("p1")).tfd.map(c => c.id)).toEqual([ "c1" ]);
  });

  it("trips: valores inválidos viram seguros", async () => {
    stubFetch(200, {
      items: [ { id: "t2", date: "2026-11-06" }, { date: "2026-11-07" }, { id: "t3" } ],
      tfd: [ { id: "c2", allowance_total_cents: 10.5, paid: "yes" }, { id: "c3", allowance_total_cents: -100 }, { id: "c4", allowance_total_cents: 0 } ]
    });
    const d = await citizenApi.trips("p1");
    expect(d.items).toEqual([ { id: "t2", date: "2026-11-06", departure_at: null, boarding_label: null,
      destination_name: "", provider_name: null, status: "", companion: false } ]);
    expect(d.tfd.map(c => [ c.id, c.allowance_total_cents, c.paid ])).toEqual([ [ "c2", null, false ], [ "c3", null, false ], [ "c4", 0, false ] ]);
  });

  it("trips: corpo sem listas vira listas vazias", async () => {
    stubFetch(200, {});
    expect(await citizenApi.trips("p1")).toEqual({ items: [], tfd: [] });
  });

  it("trips: 403 not_verified vira ApiError", async () => {
    stubFetch(403, { error: "not_verified" });
    await expect(citizenApi.trips("p1")).rejects.toEqual(new ApiError(403, "not_verified"));
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod30 src/lib/citizenApi.trips.test.ts` — Expected: FAIL (`trips` não existe).

- [ ] **Step 3: Implemente** os tipos da Interfaces e `normalizeTrips(raw)` num bloco `// ─── Minhas viagens (módulo 30, ADR 0048; contrato §8) ───`: item sem `id` ou sem `date` em texto some; achata `boarding.label` e `destination.{name, provider_name}` nos campos da Interfaces; `companion === true`; TFD com `status === "draft"` ou sem `id` some; `allowance_total_cents` só se `Number.isInteger` e `>= 0`, senão `null`; `paid === true`. Método `trips` com `call("GET", \`/trips?citizen_id=${encodeURIComponent(citizenId)}\`)`.

- [ ] **Step 4: Run** o mesmo comando e `cd $W && npx tsc --noEmit` — Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/citizenApi.ts src/lib/citizenApi.trips.test.ts
$G commit -m "feat: read the citizen trips without clinical or crew data

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/trips.ts`)

**Files:**
- Create: `src/lib/trips.ts`
- Test: `src/lib/trips.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `fmtCalendarDate`, `fmtWeekdayDateTime`, `cityToday` (`format.ts`); `messageFor` (`ui.tsx`); `ApiError`.
- Produces: constantes `TRIPS_HELP`, `TRIPS_NOT_VERIFIED`, `TRIPS_DISABLED`, `TRIPS_EMPTY`, `TFD_HELP`; `type TripTone = "ok" | "warn" | "accent" | "neutral"`; `tripTitle(t: CitizenTrip): string`; `tripLabel(t: CitizenTrip): string` (título e data, para o `aria-label`); `departureText(t: CitizenTrip): string`; `boardingText(t: CitizenTrip): string`; `destinationText(t: CitizenTrip): string`; `tripStatusView(status: string): { label: string; tone: TripTone }`; `splitTrips(items: CitizenTrip[], today: string): { upcoming: CitizenTrip[]; past: CitizenTrip[] }`; `moneyText(cents: number | null): string | null`; `tfdView(c: CitizenTfdCase): { title: string; status: string; detail: string | null; tone: TripTone }`; `appointmentText(c: CitizenTfdCase): string | null`; `allowanceText(c: CitizenTfdCase): string | null`; `tripsMessage(error: unknown): string`.

- [ ] **Step 1: Write the failing test** (as frases exatas são decisão deste plano)

```ts
// src/lib/trips.test.ts
import { describe, it, expect } from "vitest";
import { ApiError, type CitizenTfdCase, type CitizenTrip } from "./citizenApi";
import {
  TRIPS_DISABLED, TRIPS_NOT_VERIFIED, allowanceText, appointmentText, boardingText, departureText, destinationText,
  moneyText, splitTrips, tfdView, tripLabel, tripStatusView, tripTitle, tripsMessage
} from "./trips";

const trip = (over: Partial<CitizenTrip> = {}): CitizenTrip => ({
  id: "t1", date: "2026-11-05", departure_at: "2026-11-05T05:30:00-03:00", boarding_label: "Praça da Matriz",
  destination_name: "Curitiba", provider_name: "Hospital Erasto Gaertner", status: "confirmed", companion: false, ...over
});
const tfd = (over: Partial<CitizenTfdCase> = {}): CitizenTfdCase => ({
  id: "c1", status: "authorized", destination_name: "Curitiba", appointment_at: "2026-11-12T09:00:00-03:00",
  allowance_total_cents: 12345, paid: false, ...over
});

describe("trips", () => {
  it("textos fixos", () => {
    expect(TRIPS_NOT_VERIFIED).toBe(
      "Para ver suas viagens por aqui, valide seu cadastro na unidade de saúde com um documento com foto.");
    expect(TRIPS_DISABLED).toBe("As viagens ainda não aparecem por aqui nesta cidade.");
  });

  it("título, rótulo e destino", () => {
    expect(tripTitle(trip())).toBe("Viagem para Curitiba");
    expect(tripTitle(trip({ destination_name: " " }))).toBe("Viagem");
    expect(tripLabel(trip())).toBe("Viagem para Curitiba — 05/11/2026");
    expect(destinationText(trip())).toBe("Hospital Erasto Gaertner — Curitiba");
    expect(destinationText(trip({ provider_name: null }))).toBe("Curitiba");
  });

  it("departureText: hora válida, nula e inválida", () => {
    expect(departureText(trip())).toBe("Saída: qui., 05/11, 05:30");
    expect(departureText(trip({ departure_at: null }))).toBe("Saída em 05/11/2026 — a Secretaria informa a hora");
    expect(departureText(trip({ departure_at: "amanhã" }))).toBe("Saída em 05/11/2026 — a Secretaria informa a hora");
  });

  it("boardingText: com e sem local", () => {
    expect(boardingText(trip())).toBe("Embarque: Praça da Matriz");
    expect(boardingText(trip({ boarding_label: null }))).toBe("A Secretaria informa o local de embarque");
  });

  it("tripStatusView e tfdView: tabela e desconhecido", () => {
    expect(tripStatusView("planned")).toEqual({ label: "Prevista — a Secretaria ainda confirma o veículo e o horário", tone: "warn" });
    expect(tripStatusView("confirmed")).toEqual({ label: "Confirmada", tone: "ok" });
    expect(tripStatusView("in_progress")).toEqual({ label: "Em viagem", tone: "accent" });
    expect(tripStatusView("done")).toEqual({ label: "Realizada", tone: "neutral" });
    expect(tripStatusView("cancelled")).toEqual({ label: "Cancelada", tone: "neutral" });
    expect(tripStatusView("boarding")).toEqual({ label: "Em andamento", tone: "neutral" });
    expect(tfdView(tfd({ status: "submitted" }))).toEqual({ title: "TFD para Curitiba", status: "Em análise pela comissão de TFD",
      detail: "A comissão avalia o laudo do seu médico. A resposta chega por aqui.", tone: "warn" });
    expect(tfdView(tfd({ status: "returned" })).status).toBe("Em revisão pelo seu médico");
    expect(tfdView(tfd({ status: "denied" }))).toMatchObject({ status: "Não autorizado",
      detail: "Procure a sua unidade de saúde para saber os próximos passos." });
    expect(tfdView(tfd({ status: "foo" }))).toMatchObject({ status: "Em andamento", detail: null, tone: "neutral" });
  });

  it("splitTrips: hoje fica nas próximas até realizada", () => {
    const today = "2026-11-03";
    const a = trip({ id: "a", date: "2026-11-03", departure_at: "2026-11-03T05:00:00-03:00" });
    const b = trip({ id: "b", date: "2026-11-10" });
    const c = trip({ id: "c", date: "2026-11-03", status: "done" });
    const d = trip({ id: "d", date: "2026-10-20", status: "done" });
    const e = trip({ id: "e", date: "2026-11-06", status: "cancelled" });
    const { upcoming, past } = splitTrips([ b, d, a, c, e ], today);
    expect(upcoming.map(t => t.id)).toEqual([ "a", "e", "b" ]);
    expect(past.map(t => t.id)).toEqual([ "c", "d" ]);
  });

  it("allowanceText: nula, zero, alta, paga e a receber", () => {
    expect(moneyText(123456)).toBe("R$ 1.234,56");
    expect(moneyText(0)).toBe("R$ 0,00");
    expect(moneyText(null)).toBeNull();
    expect(allowanceText(tfd())).toBe("Ajuda de custo: R$ 123,45 — a receber");
    expect(allowanceText(tfd({ paid: true }))).toBe("Ajuda de custo: R$ 123,45 — paga");
    expect(allowanceText(tfd({ allowance_total_cents: null }))).toBeNull();
  });

  it("appointmentText: com e sem horário", () => {
    expect(appointmentText(tfd())).toBe("Atendimento no destino: qui., 12/11, 09:00");
    expect(appointmentText(tfd({ appointment_at: null }))).toBeNull();
  });

  it("tripsMessage: interruptor desligado tem frase própria", () => {
    expect(tripsMessage(new ApiError(403, "feature_disabled"))).toBe(TRIPS_DISABLED);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod30 src/lib/trips.test.ts` — Expected: FAIL.

- [ ] **Step 3: Implemente** com as frases do teste e mais: `TRIPS_HELP = "Aqui aparecem as viagens marcadas pela Secretaria de Saúde para o seu atendimento em outra cidade. Se algo mudar, você recebe um aviso."`, `TRIPS_EMPTY = "Nenhuma viagem marcada por aqui."`, `TFD_HELP = "O Tratamento Fora de Domicílio (TFD) paga a viagem e a ajuda de custo de quem precisa se tratar em outra cidade. A Secretaria informa quando e como receber."`. Restante da tabela do TFD: `authorized` → "Autorizado", detalhe "A Secretaria vai marcar a viagem. A data chega por aqui.", `ok`; `returned` → detalhe "A comissão pediu mais informações ao seu médico.", `warn`; `denied` → `neutral`; `scheduled` → "Viagem marcada", `ok`; `travelling` → "Em viagem", `accent`; `completed` → "Concluído", `neutral`; `cancelled` → "Cancelado", detalhe "Se ainda precisar, procure a sua unidade de saúde.", `neutral` (todos sem detalhe quando não dito). `departureText` cai na frase de reserva quando `fmtWeekdayDateTime` devolve `"—"`. `splitTrips`: próximas = `date >= today` e `status !== "done"`, em ordem crescente de `date` e `departure_at`; anteriores = o resto, em ordem decrescente. `moneyText` monta `R$ ` + `(cents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 2, maximumFractionDigits: 2 })` (espaço comum depois de `R$`, nunca `Intl` de moeda, que põe espaço inseparável). `tripsMessage`: `feature_disabled` → `TRIPS_DISABLED`; o resto `messageFor`. Nenhuma constante cita procedimento, veículo ou motorista.

- [ ] **Step 4: Run** o mesmo comando — Expected: PASS.

- [ ] **Step 5: Commit** — `$G add src/lib/trips.ts src/lib/trips.test.ts` e `feat: add the citizen trips screen rules`.

---

### Task 3: A tela "Minhas viagens"

**Files:**
- Create: `src/modules/citizen/MyTripsStep.tsx`
- Test: `src/modules/citizen/MyTripsStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.trips` (Task 1); tudo da Task 2; `cityToday`; `Screen`, `BigButton`, `ErrorText`.
- Produces: `MyTripsStep({ citizenId, onBack, onValidate }: { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void })` — `Screen` "Minhas viagens"; `TRIPS_HELP` no topo; `<section aria-label="Próximas viagens">` com `<li aria-label={tripLabel(t)}>` (título, `departureText`, `boardingText`, `destinationText`, pílula de `tripStatusView`, "Com acompanhante" se `companion`) ou `TRIPS_EMPTY`; `<section aria-label="Viagens anteriores">` só com itens; `<section aria-label="Tratamento Fora de Domicílio">` só com `tfd` não vazio, com `TFD_HELP` e `<li aria-label={tfdView(c).title}>` (situação, detalhe, `appointmentText`, `allowanceText`); 403 `not_verified` → `TRIPS_NOT_VERIFIED` em `role="note" aria-label="cadastro não validado"` e "Validar no posto".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyTripsStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyTripsStep } from "./MyTripsStep";
import { citizenApi, ApiError, type CitizenTrips } from "../../lib/citizenApi";
import { TRIPS_DISABLED, TRIPS_EMPTY, TRIPS_NOT_VERIFIED } from "../../lib/trips";

beforeEach(() => { vi.useFakeTimers({ toFake: [ "Date" ] }); vi.setSystemTime(new Date("2026-11-03T10:00:00-03:00")); });
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const DATA: CitizenTrips = {
  items: [
    { id: "t1", date: "2026-11-05", departure_at: "2026-11-05T05:30:00-03:00", boarding_label: "Praça da Matriz",
      destination_name: "Curitiba", provider_name: "Hospital Erasto Gaertner", status: "confirmed", companion: true },
    { id: "t0", date: "2026-10-20", departure_at: "2026-10-20T05:30:00-03:00", boarding_label: "Praça da Matriz",
      destination_name: "Ponta Grossa", provider_name: null, status: "done", companion: false }
  ],
  tfd: [ { id: "c1", status: "authorized", destination_name: "Curitiba", appointment_at: "2026-11-12T09:00:00-03:00",
    allowance_total_cents: 12345, paid: false } ]
};
const renderStep = (onValidate = vi.fn()) => render(<MyTripsStep citizenId="p1" onBack={vi.fn()} onValidate={onValidate} />);

describe("MyTripsStep", () => {
  it("próximas, anteriores e TFD, com uma leitura só", async () => {
    const spy = vi.spyOn(citizenApi, "trips").mockResolvedValue(DATA);
    renderStep();
    const next = within(await screen.findByRole("region", { name: "Próximas viagens" }));
    const t1 = next.getByRole("listitem", { name: "Viagem para Curitiba — 05/11/2026" });
    for (const text of [ "Saída: qui., 05/11, 05:30", "Embarque: Praça da Matriz", "Hospital Erasto Gaertner — Curitiba", "Confirmada", "Com acompanhante" ]) {
      expect(within(t1).getByText(text)).toBeInTheDocument();
    }
    const past = within(screen.getByRole("region", { name: "Viagens anteriores" }));
    expect(within(past.getByRole("listitem", { name: "Viagem para Ponta Grossa — 20/10/2026" })).getByText("Realizada")).toBeInTheDocument();
    const c1 = within(screen.getByRole("region", { name: "Tratamento Fora de Domicílio" })).getByRole("listitem", { name: "TFD para Curitiba" });
    expect(within(c1).getByText("Autorizado")).toBeInTheDocument();
    expect(within(c1).getByText("Atendimento no destino: qui., 12/11, 09:00")).toBeInTheDocument();
    expect(within(c1).getByText("Ajuda de custo: R$ 123,45 — a receber")).toBeInTheDocument();
    expect(spy).toHaveBeenCalledTimes(1);
    expect(spy).toHaveBeenCalledWith("p1");
  });

  it("nada clínico nem da escala na tela", async () => {
    const leaky = { ...DATA, items: [ { ...DATA.items[0], procedure: "Radioterapia", cid10: "C50", vehicle: { plate: "ABC1D23" },
      driver: { name: "João Motorista" } } ] };
    vi.spyOn(citizenApi, "trips").mockResolvedValue(leaky as unknown as CitizenTrips);
    renderStep();
    await screen.findByRole("listitem", { name: "Viagem para Curitiba — 05/11/2026" });
    expect(document.body.textContent).not.toMatch(/Radioterapia|C50|ABC1D23|João Motorista|placa|motorista|undefined|null|NaN/i);
  });

  it("sem TFD e sem anteriores: as seções não aparecem; sem viagens, o vazio", async () => {
    vi.spyOn(citizenApi, "trips").mockResolvedValue({ items: [], tfd: [] });
    renderStep();
    expect(await screen.findByText(TRIPS_EMPTY)).toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Viagens anteriores" })).toBeNull();
    expect(screen.queryByRole("region", { name: "Tratamento Fora de Domicílio" })).toBeNull();
  });

  it("par declarado (403 not_verified): só o convite", async () => {
    const onValidate = vi.fn();
    vi.spyOn(citizenApi, "trips").mockRejectedValue(new ApiError(403, "not_verified"));
    renderStep(onValidate);
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toHaveTextContent(TRIPS_NOT_VERIFIED);
    expect(screen.queryByRole("listitem")).toBeNull();
    await userEvent.setup({ advanceTimers: vi.advanceTimersByTime }).click(screen.getByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("interruptor caindo com a tela aberta", async () => {
    vi.spyOn(citizenApi, "trips").mockRejectedValue(new ApiError(403, "feature_disabled"));
    renderStep();
    expect(await screen.findByText(TRIPS_DISABLED)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod30 src/modules/citizen/MyTripsStep.test.tsx` — Expected: FAIL.

- [ ] **Step 3: Implemente** no molde do `MyVaccinesStep` (mesmos estilos `card`, `pill`, `warnNote`, `muted`, `list`, `h2`; `alive` contra o StrictMode; `cityToday()` lido no render para `splitTrips`). Pílula: `ok` → `var(--ok-bg)`, `warn` → `var(--warn-bg)`, `accent` → `var(--accent-bg)`, `neutral` → borda `var(--rule2)` sem fundo. A tela desenha **só** o que as funções da Task 2 devolvem; ordem das seções: Próximas, TFD, Anteriores.

- [ ] **Step 4: Run** `bin/test wpda -w wpda-mod30 src/modules/citizen/MyTripsStep.test.tsx src/theme && cd $W && npx tsc --noEmit` — Expected: PASS, guarda de tema verde.

- [ ] **Step 5: Commit** — os dois arquivos; `feat: add the citizen trips screen`.

---

### Task 4: Botão em "Minhas triagens", estado `trips` e README

**Files:**
- Modify: `src/modules/citizen/HistoryStep.tsx` (prop `onTrips?: () => void`; `BigButton` "Minhas viagens" depois de "Minhas vacinas"), `src/modules/citizen/Flow.tsx` (estado `{ at: "trips"; consentVersion: string | null; citizenId: string }`, `onTrips={hasSessionFeature("sanitary_transport") ? … : undefined}`, `case "trips"` igual ao `vaccines`, com `key={state.citizenId}`), `README.md` (parágrafo **Minhas viagens (módulo 30).** depois do de Minhas vacinas)
- Test: `src/modules/citizen/Flow.trips.test.tsx`

**Interfaces:**
- Consumes: `MyTripsStep` (Task 3).
- Produces: `HistoryStep.onTrips`.

- [ ] **Step 1: Write the failing test** — copie `signedIn` de `Flow.vaccines.test.tsx` trocando o último spy por `vi.spyOn(citizenApi, "trips").mockResolvedValue({ items: [], tfd: [] })`, relógio em `2026-11-03T10:00:00-03:00`, e:

```tsx
describe("Flow — Minhas viagens (módulo 30)", () => {
  it("com sanitary_transport: Minhas triagens → Minhas viagens → Voltar", async () => {
    const spy = signedIn([ "sanitary_transport" ]);
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
    await user.click(await screen.findByRole("button", { name: "Minhas viagens" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas viagens" })).toBeInTheDocument();
    expect(await screen.findByText("Nenhuma viagem marcada por aqui.")).toBeInTheDocument();
    expect(spy).toHaveBeenCalledWith("p1");
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas triagens" })).toBeInTheDocument();
  });

  it("só tfd na sessão, sem sanitary_transport: Minhas triagens como hoje", async () => {
    const spy = signedIn([ "tfd", "vaccination" ]);
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
    await screen.findByRole("heading", { level: 1, name: "Minhas triagens" });
    expect(screen.queryByRole("button", { name: "Minhas viagens" })).toBeNull();
    expect(spy).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod30 src/modules/citizen/Flow.trips.test.tsx` — Expected: FAIL (sem o botão).

- [ ] **Step 3: Implemente** as três mudanças. README: com `sanitary_transport` ligado (vem em `features` da sessão), "Minhas triagens" tem "Minhas viagens"; só par validado; data, hora de saída, embarque, destino (prestador e cidade), situação e acompanhante; TFD com situação, atendimento no destino e ajuda de custo (paga ou a receber), nunca laudo, CID ou procedimento; nunca veículo nem motorista; sem polling; o aviso de viagem na caixa de avisos (só data, hora e embarque); a entrada "Viagem" em "Meu histórico".

- [ ] **Step 4: Run** `bin/test wpda -w wpda-mod30 src/modules/citizen` — Expected: PASS (os `Flow.*` antigos iguais).

- [ ] **Step 5: Commit** — os quatro arquivos; `feat: open my trips from the citizen triage list`.

---

### Task 5: Aviso de viagem na caixa e frase do SMS de viagem

**Files:**
- Modify: `src/lib/citizenApi.ts` (`TripNotice`, `RawNotice` com `event`, `date`, `departure_at`, `boarding`; ramo `trip` em `normalizeNotice`; `Notice` passa a incluir `TripNotice`), `src/modules/citizen/NoticesStep.tsx` (item e detalhe do aviso de viagem), `src/modules/citizen/PreferencesStep.tsx` (frase `TRIP_SMS_EXPLANATION`)
- Test: `src/lib/citizenApi.tripNotices.test.ts`, `src/modules/citizen/NoticesStep.trips.test.tsx`, `src/modules/citizen/PreferencesStep.test.tsx`

**Interfaces:**
- Consumes: `normalizeNotice`, `NoticesStep`, `PreferencesStep` (existentes); `fmtWeekdayDateTime`, `fmtCalendarDate`; `hasSessionFeature`. O teste da caixa importa `TripNotice` de `citizenApi` e `tripNoticeTitle`, `TRIP_NOTICE_HINT`, `TRIP_NOTICE_CANCELLED_HINT` de `./NoticesStep`.
- Produces:
  - `type TripNoticeEvent = "added" | "departure_changed" | "boarding_changed" | "cancelled" | "removed" | "reminder"`;
  - `interface TripNotice { kind: "trip"; id: string; event: TripNoticeEvent | string; date: string; departure_at: string | null; boarding_label: string | null; read: boolean; cpf_masked: string | null }`;
  - em `NoticesStep.tsx`: `tripNoticeTitle(event: string): string`, `tripNoticeWhen(n: TripNotice): string`, `TRIP_NOTICE_HINT`, `TRIP_NOTICE_CANCELLED_HINT`;
  - em `PreferencesStep.tsx`: `TRIP_SMS_EXPLANATION`.

- [ ] **Step 1: Write the failing test**

`src/lib/citizenApi.tripNotices.test.ts` (mesmo `stubFetch` da Task 1):

```ts
describe("citizenApi.notices — aviso de viagem (módulo 30, W5)", () => {
  it("notices: aviso de viagem perde destino e prestador", async () => {
    stubFetch(200, { unread_count: 1, notices: [ { kind: "trip", id: "n1", event: "departure_changed", date: "2026-11-05",
      departure_at: "2026-11-05T06:00:00-03:00", boarding: { label: "Praça da Matriz" }, destination_name: "Curitiba",
      provider_name: "Hospital Erasto Gaertner", procedure: "Radioterapia", read: false, cpf_masked: null } ] });
    expect((await citizenApi.notices()).notices).toEqual([ { kind: "trip", id: "n1", event: "departure_changed", date: "2026-11-05",
      departure_at: "2026-11-05T06:00:00-03:00", boarding_label: "Praça da Matriz", read: false, cpf_masked: null } ]);
  });

  it("notices: aviso de viagem sem data fica fora; sem embarque e sem hora vira null", async () => {
    stubFetch(200, { unread_count: 0, notices: [
      { kind: "trip", id: "n2", event: "reminder" },
      { kind: "trip", id: "n3", event: "added", date: "2026-11-05", read: true, cpf_masked: "***.982.247-**" }
    ] });
    expect((await citizenApi.notices()).notices).toEqual([ { kind: "trip", id: "n3", event: "added", date: "2026-11-05",
      departure_at: null, boarding_label: null, read: true, cpf_masked: "***.982.247-**" } ]);
  });
});
```

`src/modules/citizen/NoticesStep.trips.test.tsx` (relógio em `2026-11-03T10:00:00-03:00`; `setup(notices)` como em `NoticesStep.test.tsx`):

```tsx
const changed: TripNotice = { kind: "trip", id: "n1", event: "departure_changed", date: "2026-11-05",
  departure_at: "2026-11-05T06:00:00-03:00", boarding_label: "Praça da Matriz", read: false, cpf_masked: null };

describe("NoticesStep — aviso de viagem (módulo 30)", () => {
  it("item: título do evento e a saída; novo", async () => {
    setup([ changed ]);
    const item = await screen.findByRole("button", { name: /Horário da viagem mudou/ });
    expect(within(item).getByText("qui., 05/11, 06:00")).toBeInTheDocument();
    expect(within(item).getByText("novo")).toBeInTheDocument();
  });

  it("aviso de viagem: abre com data, hora e embarque, sem destino", async () => {
    const leaky = { ...changed, destination_name: "Curitiba", provider_name: "Hospital Erasto Gaertner" } as unknown as TripNotice;
    const { markRead } = setup([ leaky ]);
    await userEvent.setup({ advanceTimers: vi.advanceTimersByTime }).click(await screen.findByRole("button", { name: /Horário da viagem mudou/ }));
    expect(await screen.findByRole("heading", { level: 1, name: "Horário da viagem mudou" })).toBeInTheDocument();
    expect(screen.getByText("Saída: qui., 05/11, 06:00")).toBeInTheDocument();
    expect(screen.getByText("Embarque: Praça da Matriz")).toBeInTheDocument();
    expect(screen.getByText(TRIP_NOTICE_HINT)).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/Curitiba|Hospital|undefined|null/);
    expect(markRead).toHaveBeenCalledWith("n1");
  });

  it("títulos por evento; desconhecido vira Aviso de viagem", () => {
    expect([ "added", "departure_changed", "boarding_changed", "cancelled", "removed", "reminder", "foo" ].map(tripNoticeTitle)).toEqual([
      "Viagem marcada", "Horário da viagem mudou", "Local de embarque mudou", "Viagem cancelada",
      "Você saiu da lista desta viagem", "Lembrete de viagem", "Aviso de viagem" ]);
  });

  it("cancelada ou retirada: a dica da central, sem hora; sem hora válida, só a data", async () => {
    setup([ { ...changed, id: "n4", event: "cancelled", departure_at: null } ]);
    await userEvent.setup({ advanceTimers: vi.advanceTimersByTime }).click(await screen.findByRole("button", { name: /Viagem cancelada/ }));
    expect(await screen.findByText("Viagem de 05/11/2026")).toBeInTheDocument();
    expect(screen.getByText(TRIP_NOTICE_CANCELLED_HINT)).toBeInTheDocument();
    expect(TRIP_NOTICE_CANCELLED_HINT).toBe("Se ainda precisar ir, procure a central de transporte da Secretaria de Saúde.");
  });

  it("várias pessoas no telefone: o aviso diz de quem é", async () => {
    setup([ { ...changed, cpf_masked: "***.982.247-**" } ]);
    expect(within(await screen.findByRole("button", { name: /Horário da viagem mudou/ })).getByText("Para o CPF ***.982.247-**")).toBeInTheDocument();
  });
});
```

No fim de `src/modules/citizen/PreferencesStep.test.tsx` (importe `TRIP_SMS_EXPLANATION`, `setSessionFeatures` de `../../lib/sessionFeatures` e `cleanup` de `@testing-library/react`):

```tsx
describe("PreferencesStep — SMS de viagem (módulo 30)", () => {
  afterEach(() => setSessionFeatures(null));

  it("com sanitary_transport e SMS: explica o que o SMS de viagem diz", async () => {
    setSessionFeatures([ "sanitary_transport" ]);
    setup({ sms_available: true, people: [ p1 ] });
    expect(within(await person("***.982.247-**")).getByText(TRIP_SMS_EXPLANATION)).toBeInTheDocument();
    expect(TRIP_SMS_EXPLANATION).toBe(
      "Se tiver viagem marcada pela Secretaria, o SMS da véspera diz a data, a hora e o local de embarque, nunca o destino nem o motivo.");
  });

  it("sem sanitary_transport, ou sem SMS na cidade: nada", async () => {
    setup({ sms_available: true, people: [ p1 ] });
    await person("***.982.247-**");
    expect(screen.queryByText(TRIP_SMS_EXPLANATION)).toBeNull();
    cleanup(); vi.restoreAllMocks();
    setSessionFeatures([ "sanitary_transport" ]);
    setup({ sms_available: false, people: [ p1 ] });
    await person("***.982.247-**");
    expect(screen.queryByText(TRIP_SMS_EXPLANATION)).toBeNull();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod30 src/lib/citizenApi.tripNotices.test.ts src/modules/citizen/NoticesStep.trips.test.tsx src/modules/citizen/PreferencesStep.test.tsx` — Expected: FAIL (o `trip` cai como tipo desconhecido e some; as exportações não existem).

- [ ] **Step 3: Implemente**
  - `normalizeNotice`: ramo `kind === "trip"` antes do de campanha — sem `date` em texto, `null`; monta só os campos de `TripNotice` (`boarding_label = n.boarding?.label ?? null`, `departure_at ?? null`, `event ?? ""`); nenhum outro campo passa.
  - `NoticesStep.tsx`: `tripNoticeTitle` com a tabela do teste; `tripNoticeWhen(n)` = `fmtWeekdayDateTime(departure_at)` e, se der `"—"`, `Viagem de ${fmtCalendarDate(date)}`; `TRIP_NOTICE_HINT = "Os detalhes estão em \"Minhas viagens\", em Minhas triagens."`; `TRIP_NOTICE_CANCELLED_HINT` com a frase do teste. No `ItemContent`, ramo `trip` (título, `tripNoticeWhen`, CPF). Detalhe `TripDetail` no molde do `ReminderDetail`: `Saída: <fmtWeekdayDateTime>` (ou `Viagem de dd/mm/aaaa`), CPF, `Embarque: <label>` quando houver, e a dica (`cancelled`/`removed` → `TRIP_NOTICE_CANCELLED_HINT`; o resto → `TRIP_NOTICE_HINT`). Chave do `li` já inclui o `kind`.
  - `PreferencesStep.tsx`: `TRIP_SMS_EXPLANATION` (frase do teste) num `<p style={{ color: "var(--ink2)" }}>` logo depois do interruptor "Lembrete de horário por SMS", só com `prefs.sms_available && hasSessionFeature("sanitary_transport")`.

- [ ] **Step 4: Run** o mesmo comando mais `src/modules/citizen/NoticesStep.test.tsx src/modules/citizen/Flow.notices.test.tsx src/lib/citizenApi.test.ts src/theme` e `cd $W && npx tsc --noEmit` — Expected: PASS (os testes antigos da caixa iguais).

- [ ] **Step 5: Commit** — os seis arquivos; `feat: show trip notices in the citizen inbox without the destination`.

---

### Task 6: Entrada `transport` no histórico

**Files:**
- Modify: `src/lib/history.ts`
- Test: `src/lib/history.test.ts`

**Interfaces:**
- Consumes: `CitizenEntry`, `VACCINATION_KIND` (existentes).
- Produces: `TRANSPORT_KIND = "transport"` (exportado); `citizenKindLabel("transport") === "Viagem"`; `entryPlace` e `entryStatusText` com o caso `transport`.

- [ ] **Step 1: Write the failing test** — no fim de `src/lib/history.test.ts` (o helper `e` já existe):

```ts
describe("módulo 30", () => {
  it("entrada transport: rótulo Viagem, sem lugar nem motorista, sem status", () => {
    const t = e("t1", { kind: "transport", title: "Viagem para Curitiba", unit: { name: "Hospital Erasto Gaertner" },
      professional_name: "João Motorista", status: "no_show" });
    expect(citizenKindLabel("transport")).toBe("Viagem");
    expect(entryPlace(t)).toBeNull();
    expect(entryStatusText(t)).toBeNull();
    expect(entryPlace(e("t2", { kind: "transport", title: "TFD para Curitiba" }))).toBeNull();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod30 src/lib/history.test.ts` — Expected: FAIL.

- [ ] **Step 3: Implemente**: em `KIND_LABEL`, `transport: "Viagem"` (comentário "Módulo 30, contrato §9"); `entryPlace` devolve `null` quando `kind === TRANSPORT_KIND` (nunca o motorista nem o prestador, W7); `entryStatusText` devolve `null` para essa entrada.

- [ ] **Step 4: Run** o mesmo comando mais `src/modules/citizen/MyHistoryStep.test.tsx` — Expected: PASS.

- [ ] **Step 5: Commit** — os dois arquivos; `feat: show trips in the citizen timeline without the driver`.

---

### Task 7: "Minhas viagens" e o aviso no Playwright (celulares)

**Files:**
- Create: `e2e/trips.spec.ts`

**Interfaces:**
- Consumes: Tasks 1–6; o Vite do worktree na **5210**.

- [ ] **Step 1: Suba o Vite do worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod30 --network rota-saude_default -p 5210:5210 \
  -v $R/.worktrees/wpda-mod30:/app -v wpda-mod30-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3052 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5210 --host 0.0.0.0"
docker logs -f wpda-mod30   # espere o "ready"
```

- [ ] **Step 2: Escreva o teste** no molde de `e2e/vaccines.spec.ts` (mesmo `mockApi`, `PHONES`, `noOverflow`, `timezoneId: "UTC"`), com a sessão `features: [ "sanitary_transport", "tfd" ]`, `/trips` respondendo a viagem confirmada de 05/11 (com `procedure`, `cid10`, `vehicle.plate`, `driver.name` a mais), uma anterior `done` e um TFD `authorized` com `allowance_total_cents: 12345`, mais um TFD `draft`; `/notices` com um aviso `trip` `departure_changed` com `destination_name` e `provider_name` a mais. Dois testes:
  - `"Minhas viagens: próximas, anteriores e TFD, sem dado clínico nem escala, sem rolagem lateral"`: entra, abre "Minhas viagens", confere o `listitem` "Viagem para Curitiba — 05/11/2026" com "Saída: qui., 05/11, 05:30" (hora da cidade com o aparelho em UTC), "Embarque: Praça da Matriz", "Confirmada"; o TFD com "Ajuda de custo: R$ 123,45 — a receber"; `body` sem `Radioterapia|C50|ABC1D23|João Motorista|Rascunho`; toda URL `/citizen/*` com `search` vazio ou `?citizen_id=p1`; `noOverflow ≤ 0`.
  - `"Aviso de viagem: data, hora e embarque, sem destino"`: abre "Avisos" na barra "Conta", toca "Horário da viagem mudou", confere "Embarque: Praça da Matriz" e `body` sem `Curitiba|Hospital`; `noOverflow ≤ 0`.

- [ ] **Step 3: Run** `cd $W && WPDA_URL=http://curitiba.localhost:5210 npx playwright test e2e/trips.spec.ts e2e/device-gate.spec.ts` — Expected: PASS nos perfis `iphone` e `pixel` (os outros pulam com motivo); device-gate verde nos 5.

- [ ] **Step 4: Commit** — `$G add e2e/trips.spec.ts`; `test: cover my trips and trip notices on phone profiles`.

---

### Task 8: Suíte, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run: `bin/test wpda -w wpda-mod30 && cd $W && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5210 npx playwright test && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`
Expected: verde; base + 6 arquivos de teste novos; `status` vazio; contagem `0`. Número de testes que dobra = artefato de build: pare e reporte.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage" src/lib/trips.ts src/modules/citizen/MyTripsStep.tsx src/modules/citizen/NoticesStep.tsx
grep -n "cid\|procedure\|specialty\|plate\|driver\|vehicle\|justification" src/lib/trips.ts src/modules/citizen/MyTripsStep.tsx
grep -n "destination" src/modules/citizen/NoticesStep.tsx
```

Expected: sem saída.

- [ ] **Step 3: Revisão** — subagente revisor lê `origin/main..HEAD` contra a spec (§1, decisões 14, 21, 22, 23, decisões do coordenador "Avisos", §4 "wpda", §5, F-30.8), o ADR 0048 e o contrato §8/§9. Atenção: nada clínico nem da escala; `draft` fora; o aviso sem destino; situações só em palavras; datas de calendário sem `new Date`; dinheiro em centavos inteiros; sem polling; sem `sanitary_transport`, "Minhas triagens" igual; cores por papel do tema.

- [ ] **Step 4: Prova no navegador (com o usuário)** — api do 30 na **3052** com a semente do plano do api (Curitiba com `sanitary_transport` e `tfd` ligados e SMS de dev; um celular com uma pessoa **validada** com uma viagem confirmada amanhã com acompanhante, uma viagem `planned` sem escala, uma anterior `done` e um TFD autorizado com ajuda de custo calculada; uma pessoa **só declarada** no mesmo celular). Container `wpda-mod30` da Task 7 no ar. Abra `http://curitiba.localhost:5210/wpda/` em 375 px com toque; o usuário entra com o celular da semente e o código SMS de dev. Screenshots em `evidencias/2026-11-03-modulo-30/wpda/`:
  - validada: "Minhas viagens" com a próxima (saída, embarque, destino, "Com acompanhante"), a prevista, a anterior e o TFD com a ajuda de custo "a receber";
  - no dashboard (central), mudar a saída da viagem de amanhã em 30 minutos ou mais: sair e voltar à caixa — "Horário da viagem mudou" com a hora nova e o embarque, sem destino; reabrir "Minhas viagens" mostra a hora nova (sem polling);
  - no dashboard (comissão), registrar o pagamento: "Minhas viagens" passa a "paga";
  - no dashboard, cancelar a viagem: "Viagem cancelada" na caixa e "Cancelada" na tela; o SMS de dev (registro do `SmsGateway`, como na prova do 17) só diz data, hora e embarque;
  - "Meu histórico" mostra "Viagem" com "Viagem para <cidade>", sem motorista;
  - pessoa declarada: o convite e "Validar no posto"; o aviso dela na caixa, com o CPF mascarado;
  - "Preferências de avisos": a frase do SMS de viagem;
  - no maintenance, desligar `sanitary_transport` em Curitiba, sair e entrar: sem o botão; religar;
  - alto contraste legível; a 375 px, sem rolagem lateral.
  Depois: `docker rm -f wpda-mod30 && docker volume rm wpda-mod30-node-modules`.

- [ ] **Step 5: Card** — se as partes do api e do dashboard de F-30.8 estiverem **Done**, mova o F-30.8 para **Done** (`98236657`) e para **Verified** se a prova passou; senão, deixe e avise a sessão do api.

- [ ] **Step 6: Pare.** Merge só depois do api e do dashboard do 30, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só com os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-30-sanitary-transport`, `push origin main`, `worktree remove $W`, `branch -d`; o `wpda-dev` (5176) recompila sozinho.

---

## Divergências propostas ao contrato

1. **W1 — `citizen_id` na rota.** O §8 diz `GET /citizen/trips` sem pessoa, mas um celular tem várias pessoas (ADR 0017) e toda rota do cidadão leva `?citizen_id=` (`/vaccinations`, `/regulation_requests`, `/care`, `/history`). Proposta: `GET /citizen/trips?citizen_id=`; pessoa de outro celular → 404.
2. **W2 — Só par validado.** O §8 não diz quem vê. Destino de tratamento e situação do TFD são sensíveis (spec §5). Proposta: par só declarado → 403 `not_verified` (como `/vaccinations`); a tela mostra o convite.
3. **W3 — Interruptores na sessão e recusa.** Proposta: `sanitary_transport` e `tfd` em `features` de `GET /citizen/session` quando utilizáveis; com `sanitary_transport` desligado, a rota responde 403 `feature_disabled` com `"feature": "sanitary_transport"`; com `tfd` desligado (e o transporte ligado), `tfd: []`.
4. **W4 — TFD em `draft` e texto da decisão.** O §8 lista `status` sem dizer quais chegam. Proposta: `draft` nunca chega ao cidadão (o médico ainda não enviou); `returned` chega como `returned` (a tela diz "Em revisão pelo seu médico"); motivo da decisão, laudo e procedimento nunca (o wpda se defende do `draft`, Task 1).
5. **W5 — Forma do aviso `trip` em `/citizen/notices`.** O §8 só diz que a caixa inclui o tipo. Proposta: `{ "kind": "trip", "id", "event": "added"|"departure_changed"|"boarding_changed"|"cancelled"|"removed"|"reminder", "date", "departure_at"|null, "boarding": { "label" }|null, "read", "cpf_masked"|null }` — **sem destino nem prestador**, porque a caixa mostra avisos de todas as pessoas do celular, inclusive pares só declarados (mesma regra do texto fixo do SMS); um aviso por evento e viagem (decisão do coordenador "Avisos"); `read` e `cpf_masked` como no lembrete do 17.
6. **W6 — Qual preferência cala o SMS de viagem.** O §8 diz "regras do 17 (opt-in, janela 8h–20h)". O 17 tem também `appointment_reminders_muted` ("Lembrete de horário por SMS"). Proposta: o SMS de viagem exige `sms_opt_in` e respeita `notices_muted` e `appointment_reminders_muted` (é lembrete); o wpda explica nas preferências o que esse SMS diz (Task 5).
7. **W7 — Entrada `transport` do histórico do cidadão.** O §9 fixa `title` e `summary: null`. Proposta: `occurred_at` = `departure_at`; `unit: null`; `professional_name: null` (nunca o motorista); `status: null`; `document: null`; TFD também com `kind: "transport"` (título "TFD para <cidade>"); viagens canceladas fora da linha do tempo. O wpda se defende de lugar e status (Task 6).
8. **W8 — Ordem, datas e valores.** Proposta: `items` em ordem de `departure_at` crescente (o wpda reordena mesmo assim); `date` como `AAAA-MM-DD` de calendário, `departure_at` e `appointment_at` em ISO 8601 com fuso; `allowance_total_cents` inteiro (soma das `tfd_allowances` do processo, paciente e acompanhante), `null` antes do cálculo; `paid: true` quando a soma dos `tfd_payments` alcança o total.

## Self-review

- **Cobertura (spec decisões 14, 21, 22, 23, decisão do coordenador "Avisos", §4 "wpda", §5, F-30.8; contrato §8, §9):** data, hora, embarque, destino, situação e acompanhante — Tasks 1–3 (Review Focus 4); TFD com situação e ajuda de custo devida — Tasks 1–3 (Review Focus 3, 5; W4, W8); nada clínico nem da escala — Tasks 1, 3, 7 (Review Focus 1); aviso na caixa na entrada, mudança, cancelamento, retirada e véspera — Task 5 (Review Focus 2; W5); SMS com opt-in — Task 5 (W6) e prova; linha do tempo `transport` — Task 6 (W7); só par validado e interruptores — Tasks 3, 4 (W2, W3); sem polling — Task 3 (uma leitura) e prova; padrão gov.br e telefone — Global Constraints, guarda de tema nas Tasks 3, 5 e 8, e2e na Task 7. O painel da decisão 22 é do dashboard.
- **Placeholders:** nenhum. Lidos na execução: ids dos cards, forma real das rotas e do aviso (Task 0, com a regra: ajustar a borda antes de começar), semente (Task 8).
- **Tipos e nomes:** `CitizenTrips`, `CitizenTrip`, `CitizenTfdCase`, `TripStatus`, `TfdStatus`, `citizenApi.trips` (Task 1) usados nas Tasks 2, 3, 7; funções e constantes da Task 2 usadas na 3; `MyTripsStep` (Task 3) e `HistoryStep.onTrips` (Task 4) os do `Flow`; `TripNotice`, `tripNoticeTitle`, `TRIP_NOTICE_HINT`, `TRIP_NOTICE_CANCELLED_HINT`, `TRIP_SMS_EXPLANATION` (Task 5); `TRANSPORT_KIND` (Task 6). Existentes conferidos em `origin/main` (`44b2533`): `call`, `str`, `strOrNull`, `normalizeNotice`, `Notice`, `fmtCalendarDate`, `fmtWeekdayDateTime`, `cityToday`, `messageFor`, `hasSessionFeature`, `setSessionFeatures`, `citizenKindLabel`, `entryPlace`, `entryStatusText`, `REMINDER_EXPLANATION`, `MyVaccinesStep` (molde).
- **Review Focus:** 1 — Tasks 1 e 3; 2 — Task 5; 3 — Tasks 1 e 2; 4 — Task 2 e Task 7; 5 — Tasks 1 e 2.
- **Proporção:** código só em testes e frases exatas; tela e regras descritas por assinatura.
