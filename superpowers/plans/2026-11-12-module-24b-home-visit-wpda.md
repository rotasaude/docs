# Módulo 24b — Visita domiciliar e atividade coletiva (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 24b** na branch `feat/mod-24b-home-visits` (worktree `apps/api/.claude/mod24b`, porta **3057**), com `home_visit` em `Timeline::KINDS`, `Sources::HomeVisitSource` e `home_visit` em `CITIZEN_FILTERED_KINDS` (contrato §12), e com `CareLines::Sources.home_visits!` e o `citizen_label` "Visitas do agente de saúde" no catálogo-base novo. Ordem (contrato §15): contracts → api → dashboard → **wpda** → maintenance → catálogo. As Tasks 0–4 não dependem do api rodando (dublês de `fetch` e `vi.spyOn(citizenApi, …)`). A prova da Task 5 depende dele.

**Goal:** Lado wpda de F-24.14 e F-24.15, e nada além disso (spec decisão 3A; contrato §12 "wpda" e §14):
- **"Meu histórico":** a visita **realizada** e não retirada aparece como "Visita", com **só a data** e o título "Visita do agente de saúde". Sem nome do profissional, unidade, motivo, medida, observação, desfecho ou situação. Recusa, ausência e retirada nunca aparecem;
- **"Meu acompanhamento":** a boa prática de visita ("Visitas do agente de saúde") aparece com a situação, como qualquer item rastreado. O api faz a conta; o wpda só desenha.

**Architecture:**
- **Cliente** (`src/lib/citizenApi.ts`): `HOME_VISIT_KIND`, `HOME_VISIT_TITLE` e `normalizeHomeVisitEntry`. A entrada `home_visit` é montada campo a campo, com o título fixo e todos os outros campos nulos. Entrada com `status` não nulo ou `outcome` diferente de 1 é descartada: é a visão do profissional vazando (W5).
- **Regras de tela** (`src/lib/history.ts`): o rótulo "Visita" em `KIND_LABEL`; `entryPlace` e `entryStatusText` nulos; `entryWhen` só com o dia, como a viagem do 30.
- **Tela** (`src/modules/citizen/MyHistoryStep.tsx`): `home_visit` entra em `noDetail`, ao lado de vacina e viagem. Uma linha.
- **"Meu acompanhamento"** (`MyCareStep.tsx`, `care.ts`, `normalizeCare`): **não muda**. O item chega com `label`, `status` e `due_on`, e o `normalizeCare` já descarta o resto. A Task 4 só prova isso com um teste.
- **Não mudam:** `Flow.tsx`, `route.ts`, `sessionFeatures.ts`, `ui.tsx`, `NoticesStep.tsx`. Sem tela nova, botão novo, rota nova, aviso nem SMS. A linha do tempo continua pedindo **todos** os kinds (`kinds[]` ausente; W1).

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `src/test/setup.ts`), `vi.stubGlobal("fetch")` e `vi.spyOn(citizenApi, …)`, Vite 5 (base `/wpda/`), Playwright (5 perfis, só a suíte existente).

**Spec:** `docs/superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md`:
- §1 (antiobjetivos: "Visita recusada ou de ausente visível para o cidadão"; "Dado de cidadão em visita a imóvel não residencial");
- §2, decisões 3A, 16 e 23;
- §4 "wpda", §6 "wpda", §8 F-24.14 e F-24.15.

Também o contrato `docs/superpowers/plans/2026-11-12-module-24b-home-visit-contracts.md` §1 (interruptores na sessão do cidadão), §12 (linha do tempo do cidadão; linhas de cuidado; "wpda"), §14 e §15 (diferenças em W1–W9). Herdados: módulo 20 (Meu histórico; `kinds[]` ausente = todos, W3 do 20; par declarado só vê triagem e agendamento), módulo 23 (Meu acompanhamento; P5a: o api nunca manda `untracked`), módulo 30 (viagem só com a data; registro retido com o interruptor desligado), módulo 32a (molde do teste de tela com `fetch` falso), ciclo de interface (decisão 9; wpda#2) e a lição do polling (82dddf1).

**Reconciliação (§16 do contrato, Q22–Q25, 2026-11-12):** W1, W3, W5 aceitas (Q22); W4 pela **alternativa** (Q22): o api manda `occurred_at` ao meio-dia local do dia da visita e o wpda continua mostrando só o dia; W2, W8, W9 aceitas (Q23); W6 aceita, `collective_activities` em `CITIZEN_HIDDEN` (Q24); **W7 pendente do usuário** (Q25): até a decisão vale o plano (frases de sempre, Task 4 como caracterização); se o usuário escolher a alternativa, a Task 4 ganha `by_team` em `normalizeCare` e as duas frases em `careItemView`, com teste.

## Global Constraints

- **Leitura campo a campo.** A entrada `home_visit` sai do cliente com exatamente `{ id, kind: "home_visit", occurred_at, unit: null, professional_name: null, title: "Visita do agente de saúde", summary: null, status: null, document: null }`. Sem `status_label` e sem `return_guidance`. O título é a constante do wpda, nunca o `title` do api: o título do profissional ("Visita domiciliar", "Visita ao imóvel") e o `summary` dele (desfecho, motivos, medidas) nunca passam da borda.
- **O cidadão nunca vê:** nome do profissional, unidade, motivo, medida, observação, desfecho, retirada, recusa, ausência, folha, microárea ou domicílio. Defesa no cliente, além do filtro do api:
  - `status` não nulo → a entrada some (o contrato manda `status` nulo para o cidadão; `recorded`/`withdrawn` é da visão do profissional);
  - `outcome` presente e diferente de `1` → some;
  - `occurred_at` ausente ou vazio → some (a data é todo o conteúdo da entrada).
- **Só a data:** `fmtDate(occurred_at)` no fuso da cidade, nunca a hora. A hora da visita diz a rotina da casa e não é para o histórico (W4). O api já manda o meio-dia local (Q22); o wpda não depende disso.
- **Sem kinds novos na URL.** `citizenApi.history()` continua sem `kinds[]` na linha do tempo; a seção "Documentos" continua com `DOCUMENT_KINDS`. Nada pessoal na URL: só `citizen_id`, `limit` e `before`, como hoje.
- **"Meu acompanhamento"** desenha o item de visita como os outros (`careItemView`): `ok` → "Em dia", `due_soon` → "Faça até <dia>", `overdue` → "Está na hora — procure a sua unidade de saúde". Nenhum campo novo é lido. A frase específica de visita é a W7.
- **Sem polling, sem timer novo, sem console, sem storage.** Nenhuma leitura nova: as telas leem como hoje (`alive`, `mounted`).
- **Interruptores:** o wpda não lê `home_visits` nem `collective_activities` da sessão. O histórico só existe com `patient_timeline`, e o acompanhamento com `care_lines`, como hoje (W6, W8).
- **Interface do canal:** nenhum componente comum muda; cores só por papel do tema; alvos de 48 px e texto de 18 px já vêm do `EntryList` e do `MyCareStep`. `src/theme/guards.test.ts` e `e2e/device-gate.spec.ts` continuam verdes.
- **Relógio:** os testes com data fixam o relógio (`vi.useFakeTimers({ toFake: [ "Date" ] })`, `2026-11-12T10:00:00-03:00`); a suíte roda em `TZ=America/Sao_Paulo`.
- **Testes:** só `bin/test wpda -w wpda-mod24b <args>`. Nunca `vitest` direto.
- **Git e board:**
  - nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - git `/opt/homebrew/bin/git`. Merge, push e tag só com autorização;
  - board #1: o wpda é dono de F-24.14 e F-24.15 junto com o api e o dashboard. Cada card vai para Done quando a última parte terminar, e para Verified depois da prova (Task 5).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod24b`, `G=/opt/homebrew/bin/git`. O worktree fica **fora de `apps/`** (React duplicado no Vite do container), na branch `feat/mod-24b-home-visits`. Os caminhos são relativos a `$W`.
- Testes: `$R/bin/test wpda -w wpda-mod24b <arquivos>`. Tipos: `cd $W && npx tsc --noEmit`. Build: `cd $W && npm run build`.
- O api do 24b roda em dev na **3057**, no container `api-dev` (rede `rota-saude_default`, apelido `api`). O dashboard do 24b fica na **5218**. O Vite do worktree sobe num container dessa rede na **5219**, com `node_modules` em volume próprio (Task 5). O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Visão do profissional chegando ao cidadão** (título "Visita domiciliar", `summary` com motivos e medidas, `professional_name`, `unit`, `status` `withdrawn`, um `outcome` 2 ou 3, a `note`). Some na borda. Testes:
   - Task 1, "visita: só os campos do contrato, com o título fixo";
   - Task 1, "visita com situação ou desfecho que não é realizada: some";
   - Task 3, "Meu histórico: visita com o rótulo, o dia e o título; nada do profissional".
2. **A hora da visita** aparecendo no cartão. Testes: Task 2, "home_visit: só o dia, no fuso da cidade"; Task 3 confere que `09:40` não está na tela.
3. **Outros kinds mudando sem querer** (urgência, vacina, viagem, exame). Os testes de 20, 21a, 29, 30, 31 e 32a continuam verdes sem edição; Task 1, "os outros kinds não mudam".
4. **"Meu acompanhamento" lendo campo a mais** (`source`, `last_visit_on`, `min_gap_days`, nome do ACS). Teste: Task 4, "a prática de visita aparece com o rótulo e a situação; nada além".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, forma real do api, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.homeVisit.test.ts` | `HOME_VISIT_KIND`, `HOME_VISIT_TITLE`, `normalizeHomeVisitEntry`, uso em `history()` | 1 |
| `src/lib/history.ts`, `src/lib/history.test.ts` | rótulo "Visita", lugar e situação nulos, só o dia | 2 |
| `src/modules/citizen/MyHistoryStep.tsx`, `src/modules/citizen/MyHistoryStep.homeVisit.test.tsx` | `home_visit` em `noDetail`; a entrada na tela | 3 |
| `src/modules/citizen/MyCareStep.homeVisit.test.tsx`, `README.md` | a prática de visita em "Meu acompanhamento" (só teste); README | 4 |
| — | suíte, tipos, build, e2e existente, revisão, prova | 5 |

**Estratégia de teste:**
- **Cliente** (Task 1), com `fetch` falso: a forma exata da entrada, o título fixo, o descarte da visão do profissional e a URL sem `kinds[]`.
- **Regras puras** (Task 2), com o helper `e()` de `history.test.ts`: rótulo, lugar, situação e data só com o dia, com uma visita às 23h30 de Brasília (02h30 UTC do dia seguinte).
- **Tela** (Task 3), com `fetch` falso por URL (molde de `MyHistoryStep.urgency.test.tsx`), para a normalização rodar de verdade.
- **"Meu acompanhamento"** (Task 4), com `fetch` falso, para o `normalizeCare` rodar: o item de visita nas três situações e nada além dos campos do contrato.
- **Regressão:** a suíte inteira, o `tsc`, o build e o Playwright existente (Task 5). Sem e2e novo: a mudança é de dado num cartão que os e2e de 23 e 32a já cobrem no celular.
- **Prova contra o api do 24b** (Task 5): visita realizada, recusada, ausente, retirada, à família e a imóvel não residencial, registradas no dashboard; o que o cidadão vê no wpda.

---

### Task 0: Cards, âncoras, forma real do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`7482397` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod24b`.
- Produces: os cards conferidos; `$W`; a base anotada; a forma real da entrada `home_visit` do cidadão.

- [ ] **Step 1: Cards F-24.14 e F-24.15.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.1[45] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os dois em In Progress ou além (o `item-edit` só se algum estiver em Not Started). Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras do wpda e o que o api do 24b fixou**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod24b
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '^export function isCapsOnlyKind\|^function normalizeEntry\|^type RawEntry\|entries: (d.entries ?? \[\]).map(normalizeEntry)\|^function normalizeCare'
$G -C $R/apps/wpda show origin/main:src/lib/history.ts | grep -n '^export const URGENCY_KIND\|^const KIND_LABEL\|^export function entryPlace\|^export function entryStatusText\|^export function entryWhen'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/MyHistoryStep.tsx | grep -n 'const noDetail = \|VACCINATION_KIND, TRANSPORT_KIND, entryPlace'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^\*\*Urgência (módulo 32a).\*\*\|^\*\*Ouvidoria (módulo 33).\*\*'
grep -n "home_visit" $A/app/services/timeline.rb $A/app/services/timeline/query.rb
grep -rn "Visita do agente de saúde" $A/app/services/timeline/sources/
grep -n "home_visit" $A/app/controllers/citizen_api/history_controller.rb
grep -n "citizen_label" $A/config/care_lines/catalog-base.yml | head -3
grep -n "home_visits\|collective_activities" $A/app/services/platform/features.rb | grep -i "hidden"
```

Expected:
- as âncoras do wpda (em `7482397`: `isCapsOnlyKind` em `citizenApi.ts:510`, `normalizeEntry` em `:514`, `history()` em `:1002`; `URGENCY_KIND` em `history.ts:22`, `entryWhen` em `:137`; `noDetail` em `MyHistoryStep.tsx:216`);
- no api: `home_visit` em `Timeline::KINDS` e em `CITIZEN_FILTERED_KINDS`, fora de `DECLARED_KINDS` (W2); o título "Visita do agente de saúde" na fonte; `citizen_label` no catálogo-base.

Ajuste **antes** de começar, com nota para a Task 5, se a forma real for diferente: o `kind` (`home_visit`), o título, um `status` não nulo na visão do cidadão (conflita com a defesa W5: **pare e avise a sessão do api**), ou `home_visit` em `DECLARED_KINDS` (W2). O ajuste vai em `normalizeHomeVisitEntry` (Task 1) e nas fixtures.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod24b -b feat/mod-24b-home-visits origin/main
cd $R/.worktrees/wpda-mod24b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod24b
```

Expected: verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 5 espera **+3 arquivos de teste** (`citizenApi.homeVisit`, `MyHistoryStep.homeVisit`, `MyCareStep.homeVisit`) e os casos novos em `history.test.ts`.

**Relato ao coordenador:** "wpda-mod24b Task 0: worktree e base verdes (<N> testes); cards F-24.14/.15 em <estado>; forma da entrada `home_visit` do api conferida; ajustes: <lista ou 'nenhum'>".

---

### Task 1: Cliente — a entrada `home_visit` em `citizenApi.history()`

**Files:**
- Modify: `src/lib/citizenApi.ts`:
  - `HOME_VISIT_KIND`, `HOME_VISIT_TITLE` e `normalizeHomeVisitEntry` logo depois de `isCapsOnlyKind`;
  - o `map` de `history()`.
- Test: `src/lib/citizenApi.homeVisit.test.ts`

**Interfaces:**
- Consumes: `RawEntry`, `CitizenEntry`, `normalizeEntry`, `isCapsOnlyKind` (existentes).
- Produces:
  - `export const HOME_VISIT_KIND = "home_visit"`;
  - `export const HOME_VISIT_TITLE = "Visita do agente de saúde"`;
  - `function normalizeHomeVisitEntry(e: RawEntry): CitizenEntry | null` (não exportada).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.homeVisit.test.ts
// Módulo 24b (contrato §12; W1, W5): a visita do agente de saúde no "Meu histórico". Só a realizada e não
// retirada, com a data e o título fixo. Nada da visão do profissional passa da borda.
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, HOME_VISIT_KIND, HOME_VISIT_TITLE } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function mockJson(body: unknown) {
  const fn = vi.fn(async (..._args: unknown[]) => new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const page = (entries: unknown[]) => ({ verified: true, partial: false, entries, next_before: null });

const VISIT = { id: "home_visit:v1", kind: "home_visit", occurred_at: "2026-11-10T09:40:00-03:00", unit: null,
  professional_name: null, title: "Visita do agente de saúde", summary: null, status: null, document: null };
const TRIAGE = { id: "triage:t1", kind: "triage", occurred_at: "2026-11-09T20:15:00-03:00", title: "Triagem respiratória",
  professional_name: "Enf. Carla", status: "completed" };

describe("citizenApi.history — visita do agente de saúde (módulo 24b)", () => {
  it("constantes do contrato", () => {
    expect(HOME_VISIT_KIND).toBe("home_visit");
    expect(HOME_VISIT_TITLE).toBe("Visita do agente de saúde");
  });

  it("visita: só os campos do contrato, com o título fixo", async () => {
    mockJson(page([ { ...VISIT, title: "Visita domiciliar", summary: "Realizada — Acompanhamento de gestante; PA 120/80",
      unit: { name: "UBS Centro" }, professional_name: "Maria Souza", status_label: "realizada", return_guidance: "Volte em 30 dias",
      document: { id: "d1", short_code: "X1" }, outcome: 1, reasons: [ 5 ], measurements: { systolic: 120, diastolic: 80 },
      note: "cachorro bravo no portão", micro_area: { code: "03" }, household: { address_short: "Rua A, 10" } } ]));
    const h = await citizenApi.history("p1");
    expect(h.entries).toEqual([ { id: "home_visit:v1", kind: "home_visit", occurred_at: "2026-11-10T09:40:00-03:00", unit: null,
      professional_name: null, title: "Visita do agente de saúde", summary: null, status: null, document: null } ]);
    expect(JSON.stringify(h)).not.toMatch(/Maria|UBS Centro|gestante|120|cachorro|realizada|Volte|Rua A|reasons|measurements|note|outcome|micro_area/);
  });

  it("visita com situação ou desfecho que não é realizada: some", async () => {
    mockJson(page([
      { ...VISIT, id: "home_visit:withdrawn", status: "withdrawn" },
      { ...VISIT, id: "home_visit:recorded", status: "recorded" },
      { ...VISIT, id: "home_visit:refused", outcome: 2 },
      { ...VISIT, id: "home_visit:absent", outcome: 3 },
      { ...VISIT, id: "home_visit:no-date", occurred_at: undefined },
      { ...VISIT, id: "home_visit:empty-date", occurred_at: "" },
      { ...VISIT, id: "home_visit:done", outcome: 1 },
      { ...VISIT, id: "home_visit:plain" }
    ]));
    const h = await citizenApi.history("p1");
    expect(h.entries.map((x) => x.id)).toEqual([ "home_visit:done", "home_visit:plain" ]);
  });

  it("os outros kinds não mudam", async () => {
    mockJson(page([ VISIT, TRIAGE, { id: "psychosocial:x", kind: "psychosocial", occurred_at: "2026-11-08T10:00:00-03:00", title: "CAPS" } ]));
    const h = await citizenApi.history("p1");
    expect(h.entries.map((x) => x.id)).toEqual([ "home_visit:v1", "triage:t1" ]);
    expect(h.entries[1]).toEqual({ id: "triage:t1", kind: "triage", occurred_at: "2026-11-09T20:15:00-03:00", unit: null,
      professional_name: "Enf. Carla", title: "Triagem respiratória", summary: null, status: "completed", document: null });
  });

  it("a linha do tempo não pede kinds: a visita chega na lista de sempre (W1)", async () => {
    const fn = mockJson(page([ VISIT ]));
    await citizenApi.history("p1");
    const url = new URL(String(fn.mock.calls[0][0]), "http://curitiba.localhost");
    expect(url.searchParams.getAll("kinds[]")).toEqual([]);
    expect(url.search).not.toMatch(/home_visit/);
  });
});
```

- [ ] **Step 2: Run** `$R/bin/test wpda -w wpda-mod24b src/lib/citizenApi.homeVisit.test.ts`. Expected: FAIL (`HOME_VISIT_KIND` não existe; a visita sai com o título, o resumo e o profissional do api).

- [ ] **Step 3: Implemente**, logo depois de `isCapsOnlyKind`:

```ts
// Módulo 24b (contrato §12; W2–W5): a visita do agente de saúde chega ao cidadão só realizada e não retirada,
// com a data e o título. O título é deste canal, nunca o do api: o do profissional ("Visita domiciliar") e o
// resumo dele (desfecho, motivos, medidas) não passam. Situação não nula ou desfecho que não é "realizada" é a
// visão do profissional vazando: a entrada some.
export const HOME_VISIT_KIND = "home_visit";
export const HOME_VISIT_TITLE = "Visita do agente de saúde";

function normalizeHomeVisitEntry(e: RawEntry): CitizenEntry | null {
  const outcome = (e as { outcome?: unknown }).outcome;
  if (e.status != null || (outcome !== undefined && outcome !== 1)) return null;
  if (typeof e.occurred_at !== "string" || e.occurred_at === "") return null;
  return { id: e.id, kind: HOME_VISIT_KIND, occurred_at: e.occurred_at, unit: null, professional_name: null,
    title: HOME_VISIT_TITLE, summary: null, status: null, document: null };
}
```

Em `history()`, troque o `map`:

```ts
      entries: (d.entries ?? [])
        .map((e) => (e.kind === HOME_VISIT_KIND ? normalizeHomeVisitEntry(e) : normalizeEntry(e)))
        .filter((e): e is CitizenEntry => e !== null && !isCapsOnlyKind(e.kind)),
      next_before: d.next_before ?? null
```

- [ ] **Step 4: Run** o mesmo comando, `$R/bin/test wpda -w wpda-mod24b src/lib` e `cd $W && npx tsc --noEmit`. Expected: PASS. `citizenApi.history.test.ts`, `citizenApi.caps.test.ts` e os demais continuam iguais.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/citizenApi.ts src/lib/citizenApi.homeVisit.test.ts
$G commit -m "feat: read the health agent visit in the citizen history field by field

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod24b Task 1: entrada `home_visit` normalizada com título fixo; visão do profissional descartada na borda".

---

### Task 2: Regras de tela — "Visita" em `history.ts`

**Files:**
- Modify: `src/lib/history.ts`
- Test: `src/lib/history.test.ts` (bloco novo "módulo 24b" no fim)

**Interfaces:**
- Consumes: `HOME_VISIT_KIND` (Task 1), `fmtDate` (existente).
- Produces: `HOME_VISIT_KIND` reexportado por `history.ts` (como `URGENCY_KIND`); `citizenKindLabel("home_visit") === "Visita"`; `entryPlace`, `entryStatusText` nulos e `entryWhen` só com o dia para `home_visit`.

- [ ] **Step 1: Write the failing test.** No import do topo de `src/lib/history.test.ts`, acrescente `HOME_VISIT_KIND` à lista de `./history`. No fim do arquivo:

```ts
describe("módulo 24b", () => {
  it("entrada home_visit: rótulo Visita, sem lugar, sem profissional, sem situação", () => {
    const v = e("home_visit:v1", { kind: HOME_VISIT_KIND, title: "Visita do agente de saúde", unit: { name: "UBS Centro" },
      professional_name: "Maria Souza", status: "recorded", status_label: "realizada", summary: "Realizada" });
    expect(HOME_VISIT_KIND).toBe("home_visit");
    expect(citizenKindLabel("home_visit")).toBe("Visita");
    expect(entryPlace(v)).toBeNull();
    expect(entryStatusText(v)).toBeNull();
  });

  it("home_visit: só o dia, no fuso da cidade", () => {
    // 23h30 de Brasília = 02h30 UTC do dia seguinte: o dia é o da cidade.
    expect(entryWhen(e("home_visit:v2", { kind: HOME_VISIT_KIND, occurred_at: "2026-11-11T02:30:00Z" }))).toBe("10/11/2026");
    expect(entryWhen(e("home_visit:v3", { kind: HOME_VISIT_KIND, occurred_at: "2026-11-10T09:40:00-03:00" }))).not.toMatch(/09:40/);
  });
});
```

- [ ] **Step 2: Run** `$R/bin/test wpda -w wpda-mod24b src/lib/history.test.ts`. Expected: FAIL (`HOME_VISIT_KIND` não é exportado por `./history`; o rótulo é "Registro").

- [ ] **Step 3: Implemente** em `src/lib/history.ts`:
  - import: `import { HOME_VISIT_KIND, type CitizenDocumentRef, type CitizenEntry, type HealthVitals } from "./citizenApi";` e, depois de `URGENCY_KIND`, `export { HOME_VISIT_KIND };`;
  - em `KIND_LABEL`, depois de `urgency`: `// Módulo 24b, contrato §12.` e `home_visit: "Visita"`;
  - em `entryPlace`, antes da linha `const parts = …`: `// Nem o agente nem a unidade da visita (módulo 24b, contrato §12).` e `if (entry.kind === HOME_VISIT_KIND) return null;`;
  - em `entryStatusText`, acrescente `|| entry.kind === HOME_VISIT_KIND` à primeira condição;
  - em `entryWhen`, troque a condição por `entry.kind === TRANSPORT_KIND || entry.kind === HOME_VISIT_KIND` e acrescente ao comentário: `Visita do agente de saúde (módulo 24b, W4): só o dia; a hora diz a rotina da casa.`.

- [ ] **Step 4: Run** o mesmo comando, `$R/bin/test wpda -w wpda-mod24b src/lib` e `cd $W && npx tsc --noEmit`. Expected: PASS; os blocos de 29, 30 e 32a sem mudança.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/history.ts src/lib/history.test.ts
$G commit -m "feat: label the health agent visit and show only its day in the history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod24b Task 2: rótulo 'Visita', sem lugar nem situação, só o dia".

---

### Task 3: Tela — a visita em "Meu histórico"

**Files:**
- Modify: `src/modules/citizen/MyHistoryStep.tsx` (import e `noDetail`)
- Test: `src/modules/citizen/MyHistoryStep.homeVisit.test.tsx`

**Interfaces:**
- Consumes: `HOME_VISIT_KIND` (Task 2, via `../../lib/history`), `MyHistoryStep` (existente).
- Produces: o cartão da visita com o rótulo, o dia e o título, sem resumo, documento, lugar ou situação.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyHistoryStep.homeVisit.test.tsx
// Módulo 24b (spec decisão 3A; contrato §12): a visita do agente de saúde em "Meu histórico". Só "Visita", o dia e
// "Visita do agente de saúde". Fetch falso por URL, para a normalização de citizenApi.history() rodar de verdade.
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import { MyHistoryStep } from "./MyHistoryStep";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-11-12T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); vi.unstubAllGlobals(); });

// Uma visita "suja", como a de um api com defeito: o que for do profissional não pode aparecer.
const VISIT = { id: "home_visit:v1", kind: "home_visit", occurred_at: "2026-11-10T09:40:00-03:00",
  unit: { name: "UBS Vila Verde" }, professional_name: "Maria Souza", title: "Visita domiciliar",
  summary: "Realizada — Acompanhamento de gestante; PA 120/80", status: null, return_guidance: "Volte em 30 dias",
  reasons: [ 5 ], measurements: { systolic: 120, diastolic: 80 }, note: "cachorro bravo no portão" };
const CONSULTATION = { id: "consultation:c1", kind: "consultation", occurred_at: "2026-11-09T15:00:00-03:00",
  unit: { name: "UBS Vila Verde" }, title: "Consulta" };

function stubCitizenApi(timeline: unknown[]) {
  const fetchMock = vi.fn(async (url: string) => {
    const json = (body: unknown) => new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } });
    if (url.startsWith("/citizen/health_summary")) return json({ problems: [], medications: [], exam_requests: [], vitals: {} });
    if (url.includes("kinds%5B%5D=")) return json({ verified: true, partial: false, entries: [], next_before: null });
    return json({ verified: true, partial: false, entries: timeline, next_before: null });
  });
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

describe("MyHistoryStep — visita do agente de saúde (módulo 24b)", () => {
  it("Meu histórico: visita com o rótulo, o dia e o título; nada do profissional", async () => {
    stubCitizenApi([ VISIT, CONSULTATION ]);
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    const timeline = within(await screen.findByRole("region", { name: "Linha do tempo" }));
    const card = (await timeline.findByText("Visita do agente de saúde")).closest("li") as HTMLElement;
    expect(within(card).getByText("Visita")).toBeInTheDocument();
    expect(within(card).getByText("10/11/2026")).toBeInTheDocument();
    expect(card.textContent).toBe("Visita10/11/2026Visita do agente de saúde");
    expect(document.body.textContent).not.toMatch(/Maria Souza|Visita domiciliar|gestante|120\/80|cachorro|Volte em 30|09:40|Registro/);
  });

  it("recusa, ausência e retirada que vazem: nenhum cartão de visita", async () => {
    stubCitizenApi([ { ...VISIT, id: "home_visit:r", outcome: 2 }, { ...VISIT, id: "home_visit:a", outcome: 3 },
      { ...VISIT, id: "home_visit:w", status: "withdrawn" }, CONSULTATION ]);
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    expect(await screen.findByText("Consulta")).toBeInTheDocument();
    expect(screen.queryByText("Visita do agente de saúde")).not.toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/Recusad|Ausente|retirad/i);
  });
});
```

- [ ] **Step 2: Run** `$R/bin/test wpda -w wpda-mod24b src/modules/citizen/MyHistoryStep.homeVisit.test.tsx`. Expected: o primeiro FAIL. Sem `home_visit` em `noDetail`, a tela ainda desenharia um `summary`. Depois das Tasks 1 e 2 ele já sai nulo, então o teste pode passar já aqui: anote isso e siga para o Step 3, que é a defesa da tela.

- [ ] **Step 3: Implemente** em `MyHistoryStep.tsx`:
  - no import de `../../lib/history`, acrescente `HOME_VISIT_KIND`;
  - `const noDetail = e.kind === VACCINATION_KIND || e.kind === TRANSPORT_KIND || e.kind === HOME_VISIT_KIND;`
  - no comentário da linha de cima: `Vacina (módulo 29, RF5), viagem (módulo 30) e visita do agente de saúde (módulo 24b): só título e data (a vacina com a unidade); resumo e documento do api não se desenham.`

- [ ] **Step 4: Run** o mesmo comando e `$R/bin/test wpda -w wpda-mod24b src/modules/citizen/MyHistoryStep` (os arquivos `MyHistoryStep.*`). Expected: PASS em todos.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/MyHistoryStep.tsx src/modules/citizen/MyHistoryStep.homeVisit.test.tsx
$G commit -m "feat: show the health agent visit in my history with only its day

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod24b Task 3: cartão 'Visita' em Meu histórico, só o dia e o título; recusa, ausência e retirada não aparecem".

---

### Task 4: "Meu acompanhamento" com a prática de visita, e README

**Files:**
- Test: `src/modules/citizen/MyCareStep.homeVisit.test.tsx`
- Modify: `README.md` (parágrafo novo depois de **Urgência (módulo 32a)**)

**Interfaces:**
- Consumes: `MyCareStep`, `citizenApi.care`, `normalizeCare`, `careItemView` (existentes, sem mudança).
- Produces: a prova de que o item "Visitas do agente de saúde" é desenhado com a situação e nada mais.

- [ ] **Step 1: Write the test**

```tsx
// src/modules/citizen/MyCareStep.homeVisit.test.tsx
// Módulo 24b (spec decisões 3A e 23; contrato §12): com home_visits na cidade, a prática de visita entra em
// "Meu acompanhamento" como item rastreado, com o citizen_label do catálogo. O api faz a conta; aqui só se desenha
// o rótulo e a situação. Fetch falso, para o normalizeCare rodar de verdade.
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import { MyCareStep } from "./MyCareStep";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-11-12T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); vi.unstubAllGlobals(); });

const LABEL = "Visitas do agente de saúde";
// Campos fora do contrato §7 do 23, como os de um api com defeito: nunca aparecem.
const EXTRA = { source: "acs_visit", practice_key: "c5_d", min_gap_days: 30, last_visit_on: "2026-11-10",
  professional: { name: "Maria Souza" }, reasons: [ 13 ] };

function stubCare(status: string, due_on: string | null) {
  const fetchMock = vi.fn(async (url: string) => {
    if (!url.startsWith("/citizen/care?")) throw new Error(`rota inesperada: ${url}`);
    return new Response(JSON.stringify({ overdue_triages: [], lines: [ { name: "Pessoa com hipertensão", items: [
      { label: "medir a pressão", status: "ok", due_on: "2027-01-06" },
      { label: LABEL, status, due_on, ...EXTRA }
    ] } ] }), { status: 200, headers: { "Content-Type": "application/json" } });
  });
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

function renderCare() {
  render(<MyCareStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} onStartCatalog={vi.fn()} />);
}

describe("MyCareStep — prática de visita (módulo 24b)", () => {
  it("a prática de visita aparece com o rótulo e a situação; nada além", async () => {
    stubCare("ok", "2026-12-10");
    renderCare();
    const item = within(await screen.findByRole("listitem", { name: LABEL }));
    expect(item.getByText("Em dia")).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/acs_visit|c5_d|min_gap|Maria Souza|2026-11-10|10\/11\/2026/);
  });

  it("perto do prazo e atrasada: as frases de sempre", async () => {
    stubCare("due_soon", "2026-12-10");
    renderCare();
    expect(within(await screen.findByRole("listitem", { name: LABEL })).getByText("Faça até 10/12/2026")).toBeInTheDocument();
  });

  it("atrasada: procure a unidade", async () => {
    stubCare("overdue", "2026-10-01");
    renderCare();
    expect(within(await screen.findByRole("listitem", { name: LABEL }))
      .getByText("Está na hora — procure a sua unidade de saúde")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run** `$R/bin/test wpda -w wpda-mod24b src/modules/citizen/MyCareStep.homeVisit.test.tsx`. Expected: **PASS sem mudar código**. É um teste de caracterização: o contrato §12 não pede rota nem campo novo. Se falhar, **pare**: o `normalizeCare` ou o `careItemView` mudou desde `7482397`, e o ajuste é conversa com o coordenador (W7), não remendo aqui.

- [ ] **Step 3: Commit do teste**

```bash
cd $W && $G add src/modules/citizen/MyCareStep.homeVisit.test.tsx
$G commit -m "test: cover the health agent visit practice in my care

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 4: README.** Depois do parágrafo **Urgência (módulo 32a)**, acrescente:

```markdown
**Visita do agente de saúde (módulo 24b).** A visita que o agente de saúde fez
e registrou como realizada aparece em "Meu histórico" como "Visita", só com o
dia e "Visita do agente de saúde". Nunca aparecem o nome do agente, a unidade,
os motivos, as medidas, a observação nem a hora. A visita recusada, a de quem
não estava em casa e a retirada pela equipe não aparecem; a visita a imóvel sem
morador também não. O cliente se defende: monta a entrada campo a campo, com o
título fixo, e descarta a que chegar com situação ou desfecho (`citizenApi.ts`,
`normalizeHomeVisitEntry`). Com `home_visits` ligado na cidade, "Meu
acompanhamento" mostra a prática "Visitas do agente de saúde" com a situação,
contada assim que a visita é gravada. Não há tela, botão, aviso nem SMS novo.
A atividade coletiva não aparece no canal web.
```

- [ ] **Step 5: Commit do README**

```bash
cd $W && $G add README.md
$G commit -m "docs: describe the health agent visit in the citizen channel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod24b Task 4: 'Meu acompanhamento' desenha a prática de visita sem mudar código (teste de caracterização); README atualizado".

---

**Q25 (decisão do usuário, substitui a caracterização pura desta task):** o cliente de "Meu acompanhamento" lê `by_team` (booleano; ausente = `false`). Com `by_team: true`, o `careItemView` usa "O agente de saúde deve passar até <dia>" no `due_soon` e "A visita está atrasada — fale com a sua unidade de saúde" no `overdue`; nos outros estados e itens, as frases de sempre. Testes: "item by_team due_soon mostra a frase do agente", "item by_team overdue manda falar com a unidade", "item sem by_team segue Faça até". Commit `feat: say the health agent visits in my care when the practice is the agent's`.

### Task 5: Suíte, tipos, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos, build e e2e existente**

Run: `$R/bin/test wpda -w wpda-mod24b && cd $W && npx tsc --noEmit && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base mais 3 arquivos de teste e os 2 casos novos de `history.test.ts`;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

Com o Vite do Step 4 no ar: `cd $W && WPDA_URL=http://curitiba.localhost:5219 npx playwright test`. Expected: verde (os e2e de `care`, `device-gate` e os demais, sem edição).

- [ ] **Step 2: Nada vaza**

```bash
cd $W
$G diff origin/main..HEAD -- src | grep -n "^+.*\(console\.\|localStorage\|sessionStorage\|setTimeout\|setInterval\)"
$G diff origin/main..HEAD -- src/lib/citizenApi.ts | grep -n "^+.*\(e\.title\|e\.summary\|e\.professional_name\|e\.unit\|reasons\|measurements\|note\)"
$G diff origin/main..HEAD --stat -- src/modules/citizen/Flow.tsx src/lib/route.ts src/lib/sessionFeatures.ts src/modules/citizen/ui.tsx src/modules/citizen/MyCareStep.tsx src/lib/care.ts
```

Expected: sem saída nos três. Sem console, storage ou timer novo; a entrada de visita não lê título, resumo, profissional, unidade, motivos, medidas nem observação do api; os arquivos que não deviam mudar não mudaram.

- [ ] **Step 3: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 3A, 16 e 23; §1; F-24.14, F-24.15) e o contrato (§12, §14). Ele confere:
  - a forma exata da entrada `home_visit` e o título fixo;
  - o descarte por `status`, `outcome` e data;
  - só o dia no cartão, sem a hora;
  - nenhum kind novo na URL;
  - "Meu acompanhamento" sem campo novo lido;
  - os outros kinds intactos (20, 21a, 29, 30, 31, 32a);
  - o README sem promessa que o código não cumpre.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende de:
- o api do 24b na **3057** e o banco de dev migrado, **com autorização do usuário** (plano do api);
- Curitiba com `territory_registry`, `home_visits`, `care_lines` e `patient_timeline` ligados;
- a versão nova do catálogo-base publicada no maintenance (`citizen_label` "Visitas do agente de saúde"; plano do maintenance);
- a semente do plano do api: um ACS (`515105`) com CNS, responsável por uma microárea; nela, um domicílio com uma família de duas pessoas e um imóvel não residencial (comércio);
- no celular de dev, **duas pessoas validadas**: o responsável da família, inscrito na linha de hipertensão (`hypertension_c5`), e outro morador do mesmo domicílio;
- o dashboard do 24b na **5218**, para o ACS registrar.

Suba o Vite do worktree (fica no ar até o fim da prova):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod24b --network rota-saude_default -p 5219:5219 \
  -v $R/.worktrees/wpda-mod24b:/app -v wpda-mod24b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3057 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5219 --host 0.0.0.0"
docker logs -f wpda-mod24b   # espere o "ready"
```

Abra `http://curitiba.localhost:5219/wpda/` a 375 px, com toque. Guarde os screenshots em `evidencias/2026-11-12-modulo-24b/wpda/`. Roteiro:
1. **Realizada ao morador:** no dashboard, o ACS registra uma visita **realizada** ao responsável, com motivos (acompanhamento de hipertenso), pressão e uma observação. No wpda, com o responsável, "Meu histórico" mostra um cartão "Visita", com o dia e "Visita do agente de saúde", e nada mais. Na rede (`read_network_requests`), a resposta de `/citizen/history` não traz o nome do ACS, motivo, medida, observação, `outcome` nem `status` não nulo. A URL não tem `kinds[]`.
2. **"Meu acompanhamento":** a linha de hipertensão mostra "Visitas do agente de saúde" com a situação ("Em dia", se a regra da C5 foi cumprida pela visita, contada com a folha ainda aberta: decisão 23).
3. **Recusada e ausente:** o ACS registra uma visita **recusada** e uma **ausente** ao outro morador. No wpda, com o outro morador, nenhum cartão de visita aparece, e o acompanhamento não muda.
4. **Retirada:** o ACS retira a visita do passo 1, com motivo. Ao abrir de novo "Meu histórico" (sem polling: sair e entrar), o cartão sumiu; "Meu acompanhamento" recalcula.
5. **À família:** o ACS registra uma visita realizada à família (`family_responsible`). O cartão aparece só para o responsável, não para o outro morador (W9).
6. **Não residencial:** uma visita realizada ao comércio não aparece para ninguém do celular.
7. **Interruptor:** no maintenance (5177, com o serviço do compose parado), desligue `home_visits`. A prática de visita sai de "Meu acompanhamento" (fonte não rastreada). Em "Meu histórico", confira o que foi decidido na W8 (proposta: o cartão da visita realizada fica). Religue.
8. **Par declarado:** com uma pessoa só declarada no celular, "Meu histórico" mostra o convite para validar e nenhum cartão de visita (W2).
9. Alto contraste legível; a 375 px, nenhuma tela com rolagem lateral.

Ao terminar: `docker rm -f wpda-mod24b && docker volume rm wpda-mod24b-node-modules`.

- [ ] **Step 5: Cards.** Se as partes do api e do dashboard de F-24.14 e F-24.15 estiverem **Done**, mova os dois para **Done** (`98236657`), e para **Verified** se a prova passou. Senão, deixe como está e avise a sessão do api.

- [ ] **Step 6: Pare.** O merge só vem depois do api e do dashboard do 24b, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-24b-home-visits`, `push origin main`, `worktree remove $W`, `branch -d`. O `wpda-dev` (5176) recompila sozinho.

**Relato ao coordenador:** "wpda-mod24b Task 5: suíte <N>/0, tsc, build e e2e verdes; prova <passou/pendente: …>; cards F-24.14/.15 em <estado>; pronto para merge depois do api e do dashboard".

---

## Divergências propostas ao contrato

> **Decididas na §16 do contrato (2026-11-12):** W1, W3, W4 (alternativa: meio-dia local), W5 → Q22; W2, W8, W9 → Q23; W6 → Q24; W7 → Q25, **pendente do usuário**.

1. **W1 — "O wpda acrescenta `home_visit` à lista de kinds que pede."** O contrato §12 diz isso, mas a linha do tempo do wpda não manda lista: `citizenApi.history(citizenId)` vai sem `kinds[]`, e o api lê `kinds[]` ausente como todos (`history_controller.rb:70-75`, W3 do módulo 20). A única lista do wpda é a da seção "Documentos" (`DOCUMENT_KINDS = [ "clinical_document" ]`), onde a visita não entra.
   - Proposta: o §12 diz "o wpda não muda o pedido; a visita chega porque `home_visit` entra em `Timeline::KINDS` e passa por `Timeline::Query.citizen_kinds`".
   - O plano do api confere, num request spec, que `GET /citizen/history` sem `kinds[]` devolve a visita para o par validado. O wpda prova a URL sem `kinds[]` (Task 1).
2. **W2 — Par só declarado.** O contrato não diz se o par declarado vê a visita. Hoje ele só vê triagem e agendamento (`Timeline::Query::DECLARED_KINDS`, `query.rb:10`), e o "Meu acompanhamento" é só do par validado.
   - Proposta: `home_visit` **não** entra em `DECLARED_KINDS`. O §12 registra "só par validado", como vacina, viagem e urgência.
   - Prova na Task 5, passo 8.
3. **W3 — "Ícone de visita".** O §12 diz "mostra a entrada com o ícone de visita", mas a linha do tempo do wpda não tem ícones. Cada cartão diz o tipo em texto (`citizenKindLabel`: "Vacina", "Viagem", "Urgência").
   - Proposta: o §12 troca "ícone" por "rótulo `Visita`". O wpda usa "Visita" (Task 2), e o título "Visita do agente de saúde" fica logo abaixo.
4. **W4 — Só o dia, sem a hora.** A spec (3A) e o §14 dizem "só com a data", mas o §12 não diz o que o api manda em `occurred_at` (a hora real da visita). O wpda mostra data e hora em todos os kinds, menos a viagem.
   - Proposta: o api manda `occurred_at` como está, e o §12 registra que o wpda mostra só o dia, no fuso da cidade (`fmtDate`; Task 2). A hora da visita diz a rotina da casa.
   - Alternativa: o api manda o meio-dia do dia local, como a passagem de TFD do 30. É mais forte (a hora nem sai do servidor), mas cria um caso especial na fonte.
5. **W5 — Forma exata da entrada do cidadão e a defesa do cliente.** O §12 diz `summary`, `professional_name`, `unit` e `status` nulos, mas não fixa a forma inteira, e nada impede o `title` do profissional de chegar ao cidadão num api com defeito.
   - Proposta: o §12 fixa a entrada do cidadão em `{ id, kind: "home_visit", occurred_at, unit: null, professional_name: null, title: "Visita do agente de saúde", summary: null, status: null, document: null }`, sem `status_label` nem `return_guidance`. O plano do api compara a resposta com essa forma, chave por chave.
   - No wpda (Task 1): o título é a constante do canal, não o do api. A entrada some se vier com `status` não nulo, com `outcome` diferente de 1 ou sem data. Assim, se o filtro do api falhar, quem cai é a entrada, não o sigilo.
6. **W6 — Interruptores na sessão do cidadão.** O §1 manda `home_visits` e `collective_activities` para `GET /citizen/session`. O wpda não lê nenhum dos dois: a linha do tempo depende de `patient_timeline`, e o acompanhamento de `care_lines`. A atividade coletiva nunca chega ao cidadão (`PROFESSIONAL_ONLY_KINDS`).
   - Proposta: `collective_activities` entra em `CITIZEN_HIDDEN`. `home_visits` pode ficar visível (é inofensivo), mas o §1 registra que o wpda não depende dele.
   - O wpda não muda com nenhuma das duas escolhas.
7. **W7 — Frase da prática de visita em "Meu acompanhamento".** O item chega como os outros (`label`, `status`, `due_on`), e o `careItemView` diz "Faça até <dia>" no `due_soon`. Só que quem faz a visita é o agente, não o cidadão. "Faça até 10/12/2026 — Visitas do agente de saúde" pode levar a pessoa a achar que precisa agir.
   - O plano mantém as frases de sempre, sem campo novo (Task 4, teste de caracterização), como pede o §12 ("sem rota nova").
   - Alternativa: o item ganha `by_team: true` (sem código de fonte nem nome) e o wpda diz "O agente de saúde deve passar até <dia>" no `due_soon` e "A visita está atrasada — fale com a sua unidade de saúde" no `overdue`. São uma mudança de `normalizeCare` e outra de `careItemView`, com teste. O coordenador decide.
8. **W8 — Linha do tempo do cidadão com `home_visits` desligado.** O §1 diz que, desligado, "o que foi gravado fica", mas não diz se a visita continua no "Meu histórico". No acompanhamento, a fonte vira não rastreada e a prática sai (§12).
   - Proposta: a visita realizada continua no histórico, como a viagem do 30 com o transporte desligado (registro retido). O §12 registra isso, e o plano do api testa com o interruptor desligado.
   - O wpda não muda. A prova (Task 5, passo 7) confere o que for decidido.
9. **W9 — Visita à família.** O §12 diz que a pessoa da entrada é o `patient_id`, que é o responsável na visita à família. Os outros moradores não veem a visita no histórico, e a spec §9 deixa aberto se ela conta no Acompanhamento só para o responsável.
   - Proposta: manter. O wpda mostra o que o api manda por `citizen_id`, e a prova confere (Task 5, passo 5). Se o SIAPS contar para a família toda, a mudança é do api (fonte e `CareLines::Sources.home_visits!`), sem tocar no wpda.

## Self-review

- **Cobertura (spec 3A, 16, 23; §1; F-24.14, F-24.15; contrato §12 "wpda", §14):**
  - visita realizada no histórico, com a data e "Visita do agente de saúde": Tasks 1, 2 e 3 (Review Focus 1 e 2; W3, W4, W5);
  - recusa, ausência e retirada nunca: Task 1 (defesa) e Task 3; prova nos passos 3 e 4 (decisão 23);
  - sem nome do profissional, motivo, medida, observação: Tasks 1 e 3 (Review Focus 1);
  - imóvel não residencial sem dado de cidadão: é do api (visita sem pessoa não tem `patient_id`); prova no passo 6;
  - "Meu acompanhamento" contando a visita: Task 4, sem mudança de código (Review Focus 4; W7);
  - testes de componente da entrada e de "Meu acompanhamento" (spec §6 "wpda"): Tasks 3 e 4;
  - atividade coletiva fora do canal: nada a fazer no wpda (W6); o README diz isso.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards, a forma real da entrada no api (Task 0, com a regra de ajuste) e a semente (Task 5).
- **Tipos e nomes:** os novos (`HOME_VISIT_KIND`, `HOME_VISIT_TITLE`, `normalizeHomeVisitEntry`) são usados nas Tasks 2 e 3. Os existentes foram conferidos em `origin/main` (`7482397`):
  - `citizenApi.ts`: `CitizenEntry` (`:476`), `RawEntry` (`:505`), `isCapsOnlyKind` (`:510`), `normalizeEntry` (`:514`), `normalizeCare` (`:603`), `history` (`:1002`), `care` (`:1045`);
  - `history.ts`: `URGENCY_KIND` (`:22`), `KIND_LABEL` (`:24`), `entryPlace` (`:81`), `entryStatusText` (`:123`), `entryWhen` (`:137`), `fmtDate`;
  - `MyHistoryStep.tsx`: `noDetail` (`:216`), os imports de `history` (`:13-14`);
  - `MyCareStep.tsx` e `care.ts`: `careItemView` (`:33`), as frases de `ok`, `due_soon` e `overdue`;
  - `history.test.ts`: helper `e()` (`:10`); `MyHistoryStep.urgency.test.tsx` (molde do `fetch` por URL);
  - api: `Timeline::KINDS` (`timeline.rb:13`), `DECLARED_KINDS` e `citizen_kinds` (`query.rb:10`, `:73`), `requested_kinds` (`history_controller.rb:71`).
- **Proporção:** três linhas de produção em `history.ts`, uma em `MyHistoryStep.tsx` e uma função em `citizenApi.ts`. Nenhuma tela nova, porque o §14 não pede. "Meu acompanhamento" só ganha teste.
