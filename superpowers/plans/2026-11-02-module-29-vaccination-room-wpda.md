# Módulo 29 — Sala de vacina (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 29** na branch `feat/mod-29-vaccination-room` (worktree `apps/api/.claude/mod29`, porta **3051**) com `GET /citizen/vaccinations` (contrato §9, na forma das Divergências W1–W3), `vaccination` em `features` da sessão do cidadão e a fonte `vaccination` em `/citizen/history` (contrato §10, W4). Ordem (contrato §14): api → dashboard → **wpda** → maintenance. As Tasks 0–6 não dependem dele (dublês e Playwright com rotas respondidas); a prova da Task 7 depende.

**Goal:** Lado wpda de F-29.7: a tela **"Minhas vacinas"** — doses aplicadas (vacina, dose, data, local, se veio da caderneta, situação do envio "Enviada ao Ministério" / "Em envio"), **próximas doses só com caderneta conferida** e, sem conferência, o texto para levar a caderneta à unidade —, o botão em "Minhas triagens", a entrada `vaccination` em "Meu histórico" e a situação `unchecked` dos marcos de vacina em "Meu acompanhamento".

**Architecture:** O cliente (`src/lib/citizenApi.ts`) ganha `vaccinations(citizenId)` com a normalização na borda: só os campos do §9 sobrevivem (lote, vacinador, fabricante, `rnds` interno e doses canceladas são descartados) e `upcoming` vira `[]` quando `checked` é falso. Regras de tela puras em `src/lib/vaccinations.ts`. Peça nova: `src/modules/citizen/MyVaccinesStep.tsx`, no molde do `MyReferralsStep` (leitura num `useEffect` com `alive`, sem polling). Mudam o mínimo: `HistoryStep` (botão), `Flow` (estado `vaccines`), `src/lib/history.ts` (rótulo e defesa da entrada `vaccination`), `src/lib/care.ts` (`unchecked`), `README.md`.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `src/test/setup.ts` com `cleanup` e `stubDevice`), `vi.spyOn(citizenApi, …)`, Playwright (5 perfis, `e2e/`), Vite 5 (base `/wpda/`).

**Spec:** `docs/superpowers/specs/2026-11-02-module-29-vaccination-room-design.md` (§1 antiobjetivos "dose que fica só no Rota Saúde sem o cidadão saber" e "caderneta que diverge do Meu SUS Digital sem explicação", §2 decisões 3a, 15, 16 e 18, §4 "wpda", §5, §6, §8 F-29.7), `docs/adr/0047.md` e o contrato `docs/superpowers/plans/2026-11-02-module-29-vaccination-room-contracts.md` §9, §10 e §14 (diferenças em W1–W6). Herdados: ADR 0017 (par CPF + celular, validação presencial), ADR 0035 (linha do tempo), plano do wpda do 26a.

## Global Constraints

- Rota (§9 com W1–W3): `GET /citizen/vaccinations?citizen_id=` → `{ "checked": bool, "items": [ { "immunobiological_name", "dose_name", "applied_on", "place_name"|null, "transcription", "rnds": "sent"|"sending" } ], "upcoming": [ { "immunobiological_name", "dose_name", "due_on" } ] }`; 403 `not_verified` para par só declarado; 403 `feature_disabled` com `vaccination` desligado; 401 dispara `citizen:unauthenticated`.
- **Nunca lote, fabricante, vacinador nem dose cancelada na tela** (§9): o cliente descarta campo fora do contrato e item com `status: "cancelled"`.
- **Próximas doses só com caderneta conferida** (spec decisão 3a): `checked: false` → nenhuma próxima dose (mesmo que o api mande) e `VACCINES_UNCHECKED`; a palavra "atrasada" nunca aparece sem conferência.
- Situação do envio (decisão 15): `sent` → "Enviada ao Ministério"; `sending` → "Em envio"; qualquer outro valor (ou ausente) → nada, nunca o código. A tela explica uma vez o que cada uma quer dizer no Meu SUS Digital (`VACCINES_RNDS_HELP`).
- Datas `applied_on` e `due_on` são de calendário (`AAAA-MM-DD`): `fmtCalendarDate`, nunca `new Date`; "hoje" é `cityToday()`.
- Na URL só `citizen_id`. Nada em `localStorage`, `sessionStorage` ou `console`. **Sem polling**: lê ao abrir a tela.
- Interruptor: sem `vaccination` em `session.features`, "Minhas triagens" fica como hoje.
- Interface do canal (subprojetos 4 e 6e): `Screen`, `BigButton`, `ErrorText` de `ui.tsx`; alvos de 48 px, texto de 18 px; cores só por papel do tema (`var(--warn-bg)`, `var(--ok-bg)`, `var(--accent-bg)`, `var(--rule2)`, `var(--ink2)`), sem reserva e sem branco fixo — `src/theme/guards.test.ts` na suíte; fluxo só no celular (`isPhone`) e `e2e/device-gate.spec.ts` verde. Nenhum componente comum muda.
- Testes com "agora" fixam o relógio (`vi.useFakeTimers({ toFake: [ "Date" ] })`); a suíte roda em `TZ=America/Sao_Paulo`.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits por extenso, `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git `/opt/homebrew/bin/git`. Merge, push e tag só com autorização.
- Board: primeira ação, F-29.1..F-29.8 do board #1 em In Progress (Task 0). O wpda é dono de F-29.7 com o api e o dashboard: Done quando o último dos três terminar; Verified depois da prova (Task 7).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod29`, `G=/opt/homebrew/bin/git`. Worktree **fora de `apps/`**, branch `feat/mod-29-vaccination-room`; caminhos relativos a `$W`.
- Testes: `bin/test wpda -w wpda-mod29 <arquivos>`; tipos `cd $W && npx tsc --noEmit`; e2e `cd $W && WPDA_URL=http://curitiba.localhost:5208 npx playwright test` (Vite da Task 6 no ar).
- O api do 29 roda em dev na **3051** no container `api-dev` (rede `rota-saude_default`, apelido `api`); o Vite do worktree sobe num container dessa rede na **5208**, `node_modules` em volume próprio. `/citizen` já está no proxy. Banco de dev só migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Caderneta não conferida com próximas doses vindas do api** (api antigo ou bug). Espera-se só o convite para levar a caderneta; nenhuma próxima dose, nenhum "atrasada". Testes: Task 1, "vaccinations: sem conferência, upcoming vira []"; Task 3, "caderneta não conferida: o texto da caderneta e nenhuma próxima dose".
2. **Lote, vacinador ou dose cancelada vazando.** Api que mande `lot`, `by`, `manufacturer` ou um item `cancelled` não faz a tela mostrar. Testes: Task 1, "vaccinations: só os campos do contrato; cancelada some"; Task 3, "nada de lote, fabricante ou vacinador na tela".
3. **Situação do envio interna** (`blocked`, `rejected`, `failed`, `pending`, ausente). Nunca o código cru nem "recusada": vira nada. Teste: Task 2, "rndsText: só sent e sending; o resto some".
4. **Próxima dose já vencida, sem data ou com data inválida** (conferida). "Já pode tomar desde dd/mm/aaaa" para data ≤ hoje; "A partir de dd/mm/aaaa" no futuro; sem data, "A unidade informa a data"; nunca "Invalid Date" ou "—" solto. Teste: Task 2, "upcomingText: passada, hoje, futura, nula e inválida".
5. **Entrada `vaccination` do histórico com vacinador ou status** (api que mande `professional_name` ou `status: cancelled`). Mostra "Vacina", o título e a unidade; nunca o nome do vacinador nem "cancelado". Teste: Task 5, "entrada vaccination: rótulo Vacina, só a unidade, sem status".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, rota do api, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.vaccinations.test.ts` | `vaccinations` e a normalização | 1 |
| `src/lib/vaccinations.ts`, `src/lib/vaccinations.test.ts` | frases e regras de tela | 2 |
| `src/modules/citizen/MyVaccinesStep.tsx`, `src/modules/citizen/MyVaccinesStep.test.tsx` | a tela "Minhas vacinas" | 3 |
| `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.vaccines.test.tsx`, `README.md` | botão, estado `vaccines`, README | 4 |
| `src/lib/history.ts`, `src/lib/history.test.ts`, `src/lib/care.ts`, `src/lib/care.test.ts` | entrada `vaccination` no histórico; `unchecked` no acompanhamento | 5 |
| `e2e/vaccines.spec.ts` | "Minhas vacinas" nos celulares | 6 |
| — | suíte, build, e2e, revisão, prova | 7 |

**Estratégia de teste:** cliente com `fetch` falso (URL só com `citizen_id`, campos a mais descartados, cancelada some, `upcoming` zerado sem conferência) — Task 1; regras puras com tabela de casos (envio, datas, transcrição) — Task 2; a tela com conferida, não conferida, par declarado, vazio e interruptor caindo — Task 3; navegação real pelo `Flow` com e sem o interruptor — Task 4; rótulos do histórico e do acompanhamento — Task 5; Playwright nos celulares, 375–412 px sem rolagem lateral — Task 6; prova contra o api do 29 — Task 7.

---

### Task 0: Cards, âncoras, rota do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`886ab9a` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `apps/api/.claude/mod29`.
- Produces: cards em In Progress; `$W`; a base; a forma real da rota anotada.

- [ ] **Step 1: Cards F-29.1..F-29.8 para In Progress** — mesmos comandos `gh` do plano do maintenance (Task 0, Step 1), um por chamada, filtro `^F-29\\.[1-8] `. Card ausente → **pare e reporte**.

- [ ] **Step 2: Âncoras e rota do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '^// ─── Encaminhamentos (módulo 26a\|^  regulationRequests: async'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/HistoryStep.tsx | grep -n 'onReferrals'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n '{ at: "referrals"\|case "referrals":\|onReferrals={hasSessionFeature'
$G -C $R/apps/wpda show origin/main:src/lib/history.ts | grep -n '  exam: "Exame"\|^export function entryPlace\|^export function entryStatusText'
$G -C $R/apps/wpda show origin/main:src/lib/care.ts | grep -n 'case "untracked":'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^\*\*Encaminhamentos (módulo 26a).\*\*\|^### Relatório público'
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod29 api bin/rails routes 2>/dev/null | grep -E "citizen/(vaccinations|history|session)"
```

Expected: as âncoras e as rotas. Forma diferente da rota (envelope, nomes, `citizen_id`) vira ajuste no `normalizeVaccinations` e nas fixtures das Tasks 1–6 **antes** de começar, com nota para a Task 7. Se a sessão do cidadão não listar `vaccination` (W3), avise a sessão do api.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod29 -b feat/mod-29-vaccination-room origin/main
cd $R/.worktrees/wpda-mod29 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod29
```

Expected: verde e `tsc-ok`. **Anote** a base: a Task 7 espera **+4 arquivos de teste** (`citizenApi.vaccinations`, `vaccinations`, `MyVaccinesStep`, `Flow.vaccines`) e casos novos em `history.test.ts` e `care.test.ts`.

---

### Task 1: Cliente — `vaccinations`

**Files:**
- Modify: `src/lib/citizenApi.ts` (bloco novo depois do de Encaminhamentos; método depois de `regulationRequests`)
- Test: `src/lib/citizenApi.vaccinations.test.ts`

**Interfaces:**
- Consumes: `call` (existente).
- Produces:
  - `type VaccineRnds = "sent" | "sending"`;
  - `interface CitizenVaccineDose { immunobiological_name: string; dose_name: string; applied_on: string; place_name: string | null; transcription: boolean; rnds: VaccineRnds | null }`;
  - `interface CitizenUpcomingDose { immunobiological_name: string; dose_name: string; due_on: string | null }`;
  - `interface CitizenVaccinations { checked: boolean; items: CitizenVaccineDose[]; upcoming: CitizenUpcomingDose[] }`;
  - `citizenApi.vaccinations(citizenId: string): Promise<CitizenVaccinations>`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.vaccinations.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function stubFetch(status: number, body: unknown) {
  const fetchMock = vi.fn(async () => new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

const DOSE = { immunobiological_name: "Pentavalente", dose_name: "2ª dose", applied_on: "2026-09-10",
  place_name: "UBS Vila Verde", transcription: false, rnds: "sent" };

describe("citizenApi.vaccinations (contrato §9)", () => {
  it("vaccinations: GET com só o citizen_id na URL", async () => {
    const fetchMock = stubFetch(200, { checked: true, items: [], upcoming: [] });
    await citizenApi.vaccinations("p 1");
    expect(fetchMock.mock.calls[0][0]).toBe("/citizen/vaccinations?citizen_id=p%201");
    expect((fetchMock.mock.calls[0][1] as RequestInit).method).toBe("GET");
  });

  it("vaccinations: só os campos do contrato; cancelada some", async () => {
    stubFetch(200, { checked: true, upcoming: [], items: [
      { ...DOSE, lot: { number: "AB123" }, manufacturer: "Serum", by: { name: "Ana", cbo: "322205" }, id: "im1" },
      { ...DOSE, dose_name: "1ª dose", status: "cancelled" },
      { ...DOSE, dose_name: "3ª dose", rnds: "blocked" }
    ] });
    const d = await citizenApi.vaccinations("p1");
    expect(d.items).toEqual([ DOSE, { ...DOSE, dose_name: "3ª dose", rnds: null } ]);
  });

  it("vaccinations: sem conferência, upcoming vira []", async () => {
    stubFetch(200, { checked: false, items: [], upcoming: [ { immunobiological_name: "Tríplice viral", dose_name: "1ª dose", due_on: "2026-08-01" } ] });
    expect(await citizenApi.vaccinations("p1")).toEqual({ checked: false, items: [], upcoming: [] });
  });

  it("vaccinations: campos ausentes viram valores seguros", async () => {
    stubFetch(200, { checked: true, items: [ { immunobiological_name: "BCG" } ], upcoming: [ { dose_name: "Reforço" } ] });
    expect(await citizenApi.vaccinations("p1")).toEqual({
      checked: true,
      items: [ { immunobiological_name: "BCG", dose_name: "", applied_on: "", place_name: null, transcription: false, rnds: null } ],
      upcoming: [ { immunobiological_name: "", dose_name: "Reforço", due_on: null } ]
    });
  });

  it("vaccinations: 403 not_verified vira ApiError", async () => {
    stubFetch(403, { error: "not_verified" });
    await expect(citizenApi.vaccinations("p1")).rejects.toEqual(new ApiError(403, "not_verified"));
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod29 src/lib/citizenApi.vaccinations.test.ts` — Expected: FAIL (`vaccinations` não existe).

- [ ] **Step 3: Implemente** os tipos da Interfaces e `normalizeVaccinations(raw)` num bloco `// ─── Minhas vacinas (módulo 29, ADR 0047; contrato §9) ───`: monta só os seis campos por item, descarta item com `status === "cancelled"`, `rnds` fora de `sent|sending` → `null`, `checked = raw.checked === true`, `upcoming = checked ? … : []`. Método `vaccinations` com `call("GET", \`/vaccinations?citizen_id=${encodeURIComponent(citizenId)}\`)`.

- [ ] **Step 4: Run** o mesmo comando e `cd $W && npx tsc --noEmit` — Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/citizenApi.ts src/lib/citizenApi.vaccinations.test.ts
$G commit -m "feat: read the citizen vaccination card without lot or vaccinator

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/vaccinations.ts`)

**Files:**
- Create: `src/lib/vaccinations.ts`
- Test: `src/lib/vaccinations.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `fmtCalendarDate`, `cityToday` (`format.ts`); `messageFor` (`ui.tsx`); `ApiError`.
- Produces: constantes `VACCINES_HELP`, `VACCINES_RNDS_HELP`, `VACCINES_UNCHECKED`, `VACCINES_NOT_VERIFIED`, `VACCINES_DISABLED`, `VACCINES_EMPTY`, `UPCOMING_EMPTY`; `doseTitle(d: { immunobiological_name: string; dose_name: string }): string`; `appliedText(d: CitizenVaccineDose): string | null`; `rndsText(rnds: VaccineRnds | null): { label: string; tone: "ok" | "warn" } | null`; `upcomingText(u: CitizenUpcomingDose, today: string): string`; `vaccinationsMessage(error: unknown): string`.

- [ ] **Step 1: Write the failing test** (as frases exatas são decisão deste plano)

```ts
// src/lib/vaccinations.test.ts
import { describe, it, expect } from "vitest";
import { ApiError, type CitizenVaccineDose } from "./citizenApi";
import {
  VACCINES_DISABLED, VACCINES_RNDS_HELP, VACCINES_UNCHECKED, appliedText, doseTitle, rndsText, upcomingText, vaccinationsMessage
} from "./vaccinations";

const dose = (over: Partial<CitizenVaccineDose> = {}): CitizenVaccineDose => ({
  immunobiological_name: "Pentavalente", dose_name: "2ª dose", applied_on: "2026-09-10",
  place_name: "UBS Vila Verde", transcription: false, rnds: "sent", ...over
});

describe("vaccinations", () => {
  it("textos fixos", () => {
    expect(VACCINES_UNCHECKED).toBe(
      "Leve a caderneta de vacinação à sala de vacina da sua unidade de saúde. Depois que a equipe conferir, as próximas doses aparecem aqui.");
    expect(VACCINES_RNDS_HELP).toBe(
      "“Enviada ao Ministério” quer dizer que a dose também aparece no Meu SUS Digital. “Em envio” quer dizer que ela ainda está a caminho.");
    expect(VACCINES_DISABLED).toBe("As vacinas ainda não aparecem por aqui nesta cidade.");
  });

  it("doseTitle: vacina e dose; faltando, só o que veio ou Vacina", () => {
    expect(doseTitle(dose())).toBe("Pentavalente — 2ª dose");
    expect(doseTitle(dose({ dose_name: "" }))).toBe("Pentavalente");
    expect(doseTitle(dose({ immunobiological_name: " ", dose_name: "" }))).toBe("Vacina");
  });

  it("appliedText: data e local; transcrição; data inválida some", () => {
    expect(appliedText(dose())).toBe("Aplicada em 10/09/2026 — UBS Vila Verde");
    expect(appliedText(dose({ place_name: null }))).toBe("Aplicada em 10/09/2026");
    expect(appliedText(dose({ transcription: true, place_name: null }))).toBe("Aplicada em 10/09/2026 — registrada a partir da caderneta");
    expect(appliedText(dose({ applied_on: "x" }))).toBeNull();
  });

  it("rndsText: só sent e sending; o resto some", () => {
    expect(rndsText("sent")).toEqual({ label: "Enviada ao Ministério", tone: "ok" });
    expect(rndsText("sending")).toEqual({ label: "Em envio", tone: "warn" });
    expect(rndsText(null)).toBeNull();
    expect(rndsText("blocked" as never)).toBeNull();
  });

  it("upcomingText: passada, hoje, futura, nula e inválida", () => {
    const u = (due_on: string | null) => ({ immunobiological_name: "Tríplice viral", dose_name: "1ª dose", due_on });
    expect(upcomingText(u("2026-10-01"), "2026-11-02")).toBe("Já pode tomar desde 01/10/2026");
    expect(upcomingText(u("2026-11-02"), "2026-11-02")).toBe("Já pode tomar desde 02/11/2026");
    expect(upcomingText(u("2026-12-15"), "2026-11-02")).toBe("A partir de 15/12/2026");
    expect(upcomingText(u(null), "2026-11-02")).toBe("A unidade informa a data");
    expect(upcomingText(u("15/12/2026"), "2026-11-02")).toBe("A unidade informa a data");
  });

  it("vaccinationsMessage: interruptor desligado tem frase própria", () => {
    expect(vaccinationsMessage(new ApiError(403, "feature_disabled"))).toBe(VACCINES_DISABLED);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod29 src/lib/vaccinations.test.ts` — Expected: FAIL.

- [ ] **Step 3: Implemente** com as frases do teste e mais: `VACCINES_HELP = "Aqui aparecem as vacinas registradas nas salas de vacina da prefeitura e as que a equipe transcreveu da sua caderneta."`, `VACCINES_NOT_VERIFIED = "Para ver suas vacinas por aqui, valide seu cadastro na unidade de saúde com um documento com foto."`, `VACCINES_EMPTY = "Nenhuma vacina registrada por aqui ainda."`, `UPCOMING_EMPTY = "Nenhuma dose prevista no calendário por enquanto."`. `upcomingText` compara `due_on` com `today` por texto (formato `AAAA-MM-DD` validado por `fmtCalendarDate`). `vaccinationsMessage`: `feature_disabled` → `VACCINES_DISABLED`; o resto `messageFor`. Nenhuma palavra "atrasad" em constante.

- [ ] **Step 4: Run** o mesmo comando — Expected: PASS.

- [ ] **Step 5: Commit** — `$G add src/lib/vaccinations.ts src/lib/vaccinations.test.ts` e `feat: add the citizen vaccination screen rules`.

---

### Task 3: A tela "Minhas vacinas"

**Files:**
- Create: `src/modules/citizen/MyVaccinesStep.tsx`
- Test: `src/modules/citizen/MyVaccinesStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.vaccinations` (Task 1); tudo da Task 2; `cityToday`; `Screen`, `BigButton`, `ErrorText`.
- Produces: `MyVaccinesStep({ citizenId, onBack, onValidate }: { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void })` — `Screen` "Minhas vacinas"; `VACCINES_HELP` no topo; `<section aria-label="Próximas doses">` (com conferência: `<li aria-label={doseTitle(u)}>` com `upcomingText`, ou `UPCOMING_EMPTY`; sem conferência: `VACCINES_UNCHECKED` em `role="note" aria-label="caderneta não conferida"`); `<section aria-label="Vacinas aplicadas">` com `<li aria-label={doseTitle(d)}>`, `appliedText` e a pílula de `rndsText` (fundo `var(--ok-bg)` / `var(--warn-bg)`), `VACCINES_RNDS_HELP` uma vez abaixo da lista, `VACCINES_EMPTY` se vazia; 403 `not_verified` → `VACCINES_NOT_VERIFIED` em `role="note" aria-label="cadastro não validado"` e "Validar no posto".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyVaccinesStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyVaccinesStep } from "./MyVaccinesStep";
import { citizenApi, ApiError, type CitizenVaccinations } from "../../lib/citizenApi";
import { VACCINES_DISABLED, VACCINES_EMPTY, VACCINES_NOT_VERIFIED, VACCINES_UNCHECKED } from "../../lib/vaccinations";

beforeEach(() => { vi.useFakeTimers({ toFake: [ "Date" ] }); vi.setSystemTime(new Date("2026-11-02T10:00:00-03:00")); });
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const CHECKED: CitizenVaccinations = {
  checked: true,
  items: [
    { immunobiological_name: "Pentavalente", dose_name: "2ª dose", applied_on: "2026-09-10", place_name: "UBS Vila Verde", transcription: false, rnds: "sent" },
    { immunobiological_name: "BCG", dose_name: "Dose única", applied_on: "2026-03-01", place_name: null, transcription: true, rnds: "sending" }
  ],
  upcoming: [ { immunobiological_name: "Pentavalente", dose_name: "3ª dose", due_on: "2026-11-10" } ]
};
const renderStep = (onValidate = vi.fn()) => render(<MyVaccinesStep citizenId="p1" onBack={vi.fn()} onValidate={onValidate} />);

describe("MyVaccinesStep", () => {
  it("conferida: doses com envio e próximas doses", async () => {
    const spy = vi.spyOn(citizenApi, "vaccinations").mockResolvedValue(CHECKED);
    renderStep();
    const penta = await screen.findByRole("listitem", { name: "Pentavalente — 2ª dose" });
    expect(within(penta).getByText("Aplicada em 10/09/2026 — UBS Vila Verde")).toBeInTheDocument();
    expect(within(penta).getByText("Enviada ao Ministério")).toBeInTheDocument();
    const bcg = screen.getByRole("listitem", { name: "BCG — Dose única" });
    expect(within(bcg).getByText("Aplicada em 01/03/2026 — registrada a partir da caderneta")).toBeInTheDocument();
    expect(within(bcg).getByText("Em envio")).toBeInTheDocument();
    const next = within(screen.getByRole("region", { name: "Próximas doses" }));
    expect(next.getByRole("listitem", { name: "Pentavalente — 3ª dose" })).toHaveTextContent("A partir de 10/11/2026");
    expect(spy).toHaveBeenCalledTimes(1);
    expect(spy).toHaveBeenCalledWith("p1");
  });

  it("caderneta não conferida: o texto da caderneta e nenhuma próxima dose", async () => {
    vi.spyOn(citizenApi, "vaccinations").mockResolvedValue({ ...CHECKED, checked: false, upcoming: [] });
    renderStep();
    expect(await screen.findByRole("note", { name: "caderneta não conferida" })).toHaveTextContent(VACCINES_UNCHECKED);
    expect(within(screen.getByRole("region", { name: "Próximas doses" })).queryByRole("listitem")).toBeNull();
    expect(screen.getByRole("listitem", { name: "Pentavalente — 2ª dose" })).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/atrasad/i);
  });

  it("nada de lote, fabricante ou vacinador na tela", async () => {
    const leaky = { ...CHECKED, items: [ { ...CHECKED.items[0], lot: { number: "AB123" }, manufacturer: "Serum", by: { name: "Ana Souza" } } ] };
    vi.spyOn(citizenApi, "vaccinations").mockResolvedValue(leaky as unknown as CitizenVaccinations);
    renderStep();
    await screen.findByRole("listitem", { name: "Pentavalente — 2ª dose" });
    expect(document.body.textContent).not.toMatch(/AB123|Serum|Ana Souza|lote|fabricante|undefined|null/i);
  });

  it("par declarado (403 not_verified): só o convite", async () => {
    const onValidate = vi.fn();
    vi.spyOn(citizenApi, "vaccinations").mockRejectedValue(new ApiError(403, "not_verified"));
    renderStep(onValidate);
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toHaveTextContent(VACCINES_NOT_VERIFIED);
    expect(screen.queryByRole("listitem")).toBeNull();
    await userEvent.setup({ advanceTimers: vi.advanceTimersByTime }).click(screen.getByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("vazio e interruptor caindo com a tela aberta", async () => {
    vi.spyOn(citizenApi, "vaccinations").mockResolvedValueOnce({ checked: true, items: [], upcoming: [] });
    const { unmount } = renderStep();
    expect(await screen.findByText(VACCINES_EMPTY)).toBeInTheDocument();
    unmount();
    vi.spyOn(citizenApi, "vaccinations").mockRejectedValueOnce(new ApiError(403, "feature_disabled"));
    renderStep();
    expect(await screen.findByText(VACCINES_DISABLED)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod29 src/modules/citizen/MyVaccinesStep.test.tsx` — Expected: FAIL.

- [ ] **Step 3: Implemente** no molde do `MyReferralsStep` (mesmos estilos `card`, `statusPill`, `warnNote`, `muted`; `alive` contra o StrictMode; `cityToday()` lido no render para `upcomingText`). A tela desenha **só** o que as funções da Task 2 devolvem; a seção "Próximas doses" vem antes de "Vacinas aplicadas".

- [ ] **Step 4: Run** `bin/test wpda -w wpda-mod29 src/modules/citizen/MyVaccinesStep.test.tsx src/theme && cd $W && npx tsc --noEmit` — Expected: PASS, guarda de tema verde.

- [ ] **Step 5: Commit** — os dois arquivos; `feat: add the citizen vaccination card screen`.

---

### Task 4: Botão em "Minhas triagens", estado `vaccines` e README

**Files:**
- Modify: `src/modules/citizen/HistoryStep.tsx` (prop `onVaccines?: () => void`; `BigButton` "Minhas vacinas" depois de "Meus encaminhamentos"), `src/modules/citizen/Flow.tsx` (estado `{ at: "vaccines"; consentVersion: string | null; citizenId: string }`, `onVaccines={hasSessionFeature("vaccination") ? … : undefined}`, `case "vaccines"` igual ao `referrals`), `README.md` (parágrafo **Minhas vacinas (módulo 29).** depois do de Encaminhamentos)
- Test: `src/modules/citizen/Flow.vaccines.test.tsx`

**Interfaces:**
- Consumes: `MyVaccinesStep` (Task 3).
- Produces: `HistoryStep.onVaccines`.

- [ ] **Step 1: Write the failing test** — copie `signedIn` de `Flow.referrals.test.tsx` trocando o último spy por `vi.spyOn(citizenApi, "vaccinations").mockResolvedValue({ checked: false, items: [], upcoming: [] })`, relógio em `2026-11-02T10:00:00-03:00`, e:

```tsx
describe("Flow — Minhas vacinas (módulo 29)", () => {
  it("com vaccination: Minhas triagens → Minhas vacinas → Voltar", async () => {
    const spy = signedIn([ "vaccination" ]);
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
    await user.click(await screen.findByRole("button", { name: "Minhas vacinas" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas vacinas" })).toBeInTheDocument();
    expect(await screen.findByRole("note", { name: "caderneta não conferida" })).toBeInTheDocument();
    expect(spy).toHaveBeenCalledWith("p1");
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas triagens" })).toBeInTheDocument();
  });

  it("sem vaccination na sessão: Minhas triagens como hoje", async () => {
    const spy = signedIn([ "care_regulation" ]);
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
    await screen.findByRole("heading", { level: 1, name: "Minhas triagens" });
    expect(screen.queryByRole("button", { name: "Minhas vacinas" })).toBeNull();
    expect(spy).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod29 src/modules/citizen/Flow.vaccines.test.tsx` — Expected: FAIL (sem o botão).

- [ ] **Step 3: Implemente** as três mudanças. README: com `vaccination` ligado (vem em `features` da sessão), "Minhas triagens" tem "Minhas vacinas"; só par validado; doses (sem lote nem vacinador), situação do envio, próximas doses só com caderneta conferida; sem polling; a entrada "Vacina" em "Meu histórico".

- [ ] **Step 4: Run** `bin/test wpda -w wpda-mod29 src/modules/citizen` — Expected: PASS (os `Flow.*` antigos iguais).

- [ ] **Step 5: Commit** — os quatro arquivos; `feat: open my vaccines from the citizen triage list`.

---

### Task 5: Entrada `vaccination` no histórico e `unchecked` no acompanhamento

**Files:**
- Modify: `src/lib/history.ts`, `src/lib/care.ts`
- Test: `src/lib/history.test.ts`, `src/lib/care.test.ts`

**Interfaces:**
- Consumes: `CitizenEntry`, `CareItem` (existentes).
- Produces: `VACCINATION_KIND = "vaccination"` (exportado de `history.ts`); `citizenKindLabel("vaccination") === "Vacina"`; `entryPlace` e `entryStatusText` com o caso `vaccination`; `careItemView` com o caso `unchecked`.

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/history.test.ts` (o helper `e` já existe; importe `entryStatusText`):

```ts
describe("módulo 29", () => {
  it("entrada vaccination: rótulo Vacina, só a unidade, sem status", () => {
    const v = e("v1", { kind: "vaccination", title: "Vacina Pentavalente — 2ª dose", unit: { name: "UBS Vila Verde" },
      professional_name: "Ana Souza", status: "cancelled" });
    expect(citizenKindLabel("vaccination")).toBe("Vacina");
    expect(entryPlace(v)).toBe("UBS Vila Verde");
    expect(entryStatusText(v)).toBeNull();
  });
});
```

No fim de `src/lib/care.test.ts`:

```ts
describe("módulo 29", () => {
  it("marco de vacina com caderneta não conferida: leve a caderneta, sem pendência", () => {
    expect(careItemView({ label: "pentavalente — 3ª dose", status: "unchecked", due_on: null }))
      .toEqual({ title: "Pentavalente — 3ª dose", status: "Leve a caderneta de vacinação à unidade para conferir", tone: "neutral" });
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod29 src/lib/history.test.ts src/lib/care.test.ts` — Expected: FAIL.

- [ ] **Step 3: Implemente**: em `KIND_LABEL`, `vaccination: "Vacina"` (comentário "Módulo 29, contrato §10"); `entryPlace` devolve só `entry.unit?.name ?? null` quando `kind === VACCINATION_KIND` (o cidadão não vê o vacinador, §9); `entryStatusText` devolve `null` para essa entrada; em `careItemView`, `case "unchecked"` com a frase do teste e `tone: "neutral"`.

- [ ] **Step 4: Run** o mesmo comando mais `src/modules/citizen/MyHistoryStep.test.tsx src/modules/citizen/MyCareStep.test.tsx` — Expected: PASS.

- [ ] **Step 5: Commit** — os quatro arquivos; `feat: show vaccinations in the citizen timeline and care lines`.

---

### Task 6: "Minhas vacinas" no Playwright (celulares)

**Files:**
- Create: `e2e/vaccines.spec.ts`

**Interfaces:**
- Consumes: Tasks 1–5; o Vite do worktree na **5208**.

- [ ] **Step 1: Suba o Vite do worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod29 --network rota-saude_default -p 5208:5208 \
  -v $R/.worktrees/wpda-mod29:/app -v wpda-mod29-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3051 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5208 --host 0.0.0.0"
docker logs -f wpda-mod29   # espere o "ready"
```

- [ ] **Step 2: Escreva o teste** no molde de `e2e/referrals.spec.ts` (mesmo `mockApi`, `PHONES`, `noOverflow`, `timezoneId: "UTC"`), com a sessão `features: [ "vaccination" ]` e `/vaccinations` respondendo `{ checked: true, items: [ Pentavalente 2ª dose com lot/by/manufacturer a mais, rnds "sent"; BCG transcrita rnds "sending"; uma cancelada ], upcoming: [ Pentavalente 3ª dose due_on "2026-11-10" ] }`. Teste `"Minhas vacinas: doses, envio e próximas, sem lote nem vacinador, sem rolagem lateral"`: entra, abre "Minhas vacinas", confere `listitem` "Pentavalente — 2ª dose" com "Enviada ao Ministério", "BCG — Dose única" com "Em envio" e "registrada a partir da caderneta", a próxima com "10/11/2026"; `body` sem `AB123|Serum|Ana Souza|cancelad|lote`; toda URL `/citizen/*` com `search` vazio ou `?citizen_id=p1`; `noOverflow ≤ 0`.

- [ ] **Step 3: Run** `cd $W && WPDA_URL=http://curitiba.localhost:5208 npx playwright test e2e/vaccines.spec.ts e2e/device-gate.spec.ts` — Expected: PASS nos perfis `iphone` e `pixel` (os outros pulam com motivo); device-gate verde nos 5.

- [ ] **Step 4: Commit** — `$G add e2e/vaccines.spec.ts`; `test: cover my vaccines on phone profiles`.

---

### Task 7: Suíte, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run: `bin/test wpda -w wpda-mod29 && cd $W && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5208 npx playwright test && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`
Expected: verde; base + 4 arquivos de teste novos; `status` vazio; contagem `0`. Número de testes que dobra = artefato de build: pare e reporte.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage\|atrasad" src/lib/vaccinations.ts src/modules/citizen/MyVaccinesStep.tsx
grep -n "lot\b\|manufacturer\|\.by\b\|professional" src/lib/vaccinations.ts src/modules/citizen/MyVaccinesStep.tsx
```

Expected: sem saída.

- [ ] **Step 3: Revisão** — subagente revisor lê `origin/main..HEAD` contra a spec (§1 antiobjetivos, decisões 3a, 15, 16, §4 "wpda", §5, F-29.7), o ADR 0047 e o contrato §9/§10. Atenção: próximas doses só com conferência; nunca lote, vacinador ou cancelada; envio só em duas frases; sem polling; sem `vaccination`, "Minhas triagens" igual; cores por papel do tema.

- [ ] **Step 4: Prova no navegador (com o usuário)** — api do 29 na **3051** com a semente do plano do api (Curitiba com `vaccination` ligado e `rnds_mock`; um celular com uma criança **validada** e **conferida** com doses enviadas, uma em envio e uma transcrita; um adulto **validado não conferido**; uma pessoa **só declarada**). Container `wpda-mod29` da Task 6 no ar. Abra `http://curitiba.localhost:5208/wpda/` em 375 px com toque; o usuário entra com o celular da semente e o código SMS de dev. Screenshots em `evidencias/2026-11-02-modulo-29/wpda/`:
  - criança: "Minhas vacinas" com doses, "Enviada ao Ministério"/"Em envio", a transcrita, próximas doses com data;
  - no dashboard, registrar uma dose nova para a criança; sair e voltar à tela: a dose aparece "Em envio" e, depois do aceite do `rnds_mock`, "Enviada ao Ministério" (sem polling: relê ao abrir);
  - "Meu histórico" mostra "Vacina" com o título e a unidade, sem vacinador;
  - adulto não conferido: o texto da caderneta, nenhuma próxima dose; "Meu acompanhamento" (se inscrito numa linha com marco de vacina) diz "Leve a caderneta…";
  - pessoa declarada: o convite e "Validar no posto";
  - no maintenance, desligar `vaccination` em Curitiba, sair e entrar: sem o botão; religar;
  - alto contraste legível; a 375 px, sem rolagem lateral.
  Depois: `docker rm -f wpda-mod29 && docker volume rm wpda-mod29-node-modules`.

- [ ] **Step 5: Card** — se as partes do api e do dashboard de F-29.7 estiverem **Done**, mova o F-29.7 para **Done** (`98236657`) e para **Verified** se a prova passou; senão, deixe e avise a sessão do api.

- [ ] **Step 6: Pare.** Merge só depois do api e do dashboard do 29, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só com os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-29-vaccination-room`, `push origin main`, `worktree remove $W`, `branch -d`; o `wpda-dev` (5176) recompila sozinho.

---

## Divergências propostas ao contrato

1. **W1 — `citizen_id` na rota.** O §9 diz `GET /citizen/vaccinations` sem pessoa, mas um celular tem várias pessoas (ADR 0017) e toda rota do cidadão leva `?citizen_id=` (`/history`, `/exams`, `/care`, `/regulation_requests`). Proposta: `GET /citizen/vaccinations?citizen_id=`; pessoa de outro celular → 404.
2. **W2 — Só par validado.** O §9 não diz quem vê. A caderneta é dado de saúde (spec §5). Proposta: par só declarado → 403 `not_verified` (como `/care` e `/regulation_requests`); a tela mostra o convite.
3. **W3 — Interruptor na sessão e recusa.** Proposta: `vaccination` em `features` de `GET /citizen/session` quando utilizável; com ele desligado, a rota responde 403 `feature_disabled`.
4. **W4 — Entrada `vaccination` do histórico do cidadão.** O §10 fixa o `title` mas não os outros campos de `CitizenEntry`. Proposta: `occurred_at` = `applied_at`; `unit` = unidade da sala (ou nula na transcrição); `professional_name: null` (§9, sem vacinador); `status: null`; `summary: null`; `document: null`; doses canceladas fora da linha do tempo do cidadão (§9). O wpda se defende dos dois primeiros (Task 5).
5. **W5 — `unchecked` em `/citizen/care`.** O §10 diz que o marco de vacina sem conferência é `unchecked` ("não conferido"), mas não diz se chega ao cidadão. Proposta: chega como `status: "unchecked"` no item da linha; o wpda mostra "Leve a caderneta de vacinação à unidade para conferir" (Task 5), nunca como pendência.
6. **W6 — Ordem e datas.** Proposta: `items` por `applied_on` mais recente primeiro; `upcoming` por `due_on` mais próximo primeiro (nulas no fim); `applied_on` e `due_on` como data de calendário `AAAA-MM-DD`. Item `upcoming` com `due_on` no passado é permitido (dose devida) e a tela diz "Já pode tomar desde …", sem a palavra "atrasada".

## Self-review

- **Cobertura (spec decisões 3a, 15, 16, §4 "wpda", §5, F-29.7; contrato §9, §10):** doses sem lote/vacinador/cancelada — Tasks 1, 3, 6 (Review Focus 2); envio em duas frases e a explicação do Meu SUS Digital — Tasks 2, 3 (Review Focus 3); próximas doses só conferidas e o texto da caderneta — Tasks 1, 2, 3 (Review Focus 1, 4); só par validado e interruptor — Tasks 3, 4 (W2, W3); sem polling — Task 3 (uma leitura) e prova; histórico — Task 5 (Review Focus 5; W4); acompanhamento `unchecked` — Task 5 (W5); padrão gov.br e telefone — Global Constraints, guarda de tema nas Tasks 3 e 7, e2e na Task 6.
- **Placeholders:** nenhum. Lidos na execução: ids dos cards, forma real da rota (Task 0, com a regra: ajustar a borda antes de começar), semente (Task 7).
- **Tipos e nomes:** `CitizenVaccinations`, `CitizenVaccineDose`, `CitizenUpcomingDose`, `VaccineRnds`, `citizenApi.vaccinations` (Task 1) usados nas Tasks 2, 3, 6; funções e constantes da Task 2 usadas na 3; `MyVaccinesStep` (Task 3) e `HistoryStep.onVaccines` (Task 4) os do `Flow`; `VACCINATION_KIND` (Task 5). Existentes conferidos em `origin/main` (`886ab9a`): `call`, `fmtCalendarDate`, `cityToday`, `messageFor`, `hasSessionFeature`, `citizenKindLabel`, `entryPlace`, `entryStatusText`, `careItemView`.
- **Review Focus:** 1 — Tasks 1 e 3; 2 — Tasks 1 e 3; 3 — Task 2; 4 — Task 2; 5 — Task 5.
- **Proporção:** código só em testes e frases exatas; tela e regras descritas por assinatura.
