# Módulo 31 — Atenção psicossocial (CAPS) (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 31** na branch `feat/mod-31-psychosocial-care` (worktree `apps/api/.claude/mod31`, porta **3053**), com os filtros do contrato §9 e §10 e as Divergências W1–W3 resolvidas. Ordem (contrato §14): api → dashboard → maintenance → **wpda** (só o spec de ausência). As Tasks 0–2 não dependem do api rodando (dublês de `fetch` e de `citizenApi`); a prova da Task 3 depende.

**Goal:** Lado wpda de F-31.8. O plano fixa em teste que nada do CAPS aparece ao cidadão (spec decisão 4) nestas telas: "Seus agendamentos", "Meu histórico" (linha do tempo, documentos e resumo), a caixa de avisos e os botões de "Minhas triagens". Isso vale com o interruptor ligado e o api devolvendo dados do CAPS. A prova confere o mesmo contra o api real.

**Architecture:** **Nenhuma tela nova, nenhuma rota nova, nenhum componente novo** (spec decisão 4; contrato §10). Quem garante a ausência é o api: as rotas do `citizen_api` filtram o que tem origem CAPS (contrato §9 e §10; Divergência W4). O wpda tem uma única mudança de código, **um filtro de borda** em `citizenApi.history()`: entrada com `kind` da fonte do CAPS (prefixo `psychosocial`, Divergência W2) nunca chega à tela. Sem esse filtro, a tela mostraria a entrada como "Registro", com o título vindo do api (`citizenKindLabel`). O resto são testes de ausência que fixam o comportamento atual:
- a caixa já descarta `kind` desconhecido (`normalizeNotice`);
- nenhum botão depende de `psychosocial_care`.

Sobre o pedido de "Minhas consultas": a tela não existe com esse nome no wpda. Ela corresponde a "Seus agendamentos" (`AppointmentsSection`), que lê a agenda do 17, que o CAPS não usa (decisão 18), e às consultas dentro de "Meu histórico". Documento e agendamento do CAPS chegariam com `kind: "clinical_document"` ou como agendamento comum, e o wpda não tem como distingui-los. Para esses, a garantia é o spec do api (W4) e a prova (Task 3).

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `src/test/setup.ts` com `cleanup` e `stubDevice`), `vi.stubGlobal("fetch")` e `vi.spyOn(citizenApi, …)`, Vite 5 (base `/wpda/`).

**Spec:** `docs/superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md` (Afeta, §1 antiobjetivo "CID ou evolução do CAPS visível … no wpda", §2 decisões 4, 18, 19, 21, §5 "Fora do sistema", §7, §8 F-31.8), `docs/adr/0049.md` e o contrato `docs/superpowers/plans/2026-11-04-module-31-psychosocial-care-contracts.md` §1, §9, §10 e §14 (diferenças em W1–W4). Herdados: ADR 0035 (linha do tempo do cidadão), ADR 0024 (caixa de avisos), plano do wpda do 30 (molde do ambiente e da prova).

## Global Constraints

- **Nada do CAPS no wpda** (spec decisão 4; contrato §10). Isso vale para consultas, agendamentos, avisos, linha do tempo, documentos, resumo de saúde, SMS e botões. Também não existe tela, frase nem rótulo que cite CAPS, "psicossocial" ou `psychosocial`.
- Filtro de borda (W2): `isCapsOnlyKind(kind)` é verdadeiro para `"psychosocial"` e para todo `kind` que começa com `"psychosocial_"`. Entradas assim saem de `history()` antes de qualquer tela, inclusive na consulta de documentos (`kinds: DOCUMENT_KINDS`). Os outros `kind` continuam como hoje.
- `psychosocial_care` em `session.features` (se um api antigo mandar, W1) não muda nada na tela. Nenhum código do wpda consulta essa chave.
- A condição declarada `mental_health` do módulo 23 ("Meu acompanhamento") é do próprio cidadão e **não é do CAPS**: continua como está.
- Na URL, só `citizen_id` (e `kinds[]`, `before`, `limit` do histórico). Nada em `localStorage`, `sessionStorage` ou `console`.
- Interface do canal: nenhum componente muda; `src/theme/guards.test.ts` e `e2e/device-gate.spec.ts` continuam verdes.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits por extenso, com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git `/opt/homebrew/bin/git`. Merge, push e tag só com autorização.
- Board: a primeira ação é mover F-31.1..F-31.8 do board #1 para In Progress (Task 0). O wpda é dono de F-31.8 com o api e o dashboard. O card vai para Done quando o último dos três terminar, e para Verified depois da prova (Task 3).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod31`, `G=/opt/homebrew/bin/git`. Worktree **fora de `apps/`**, branch `feat/mod-31-psychosocial-care`; caminhos relativos a `$W`.
- Testes: `bin/test wpda -w wpda-mod31 <arquivos>`. Tipos: `cd $W && npx tsc --noEmit`.
- O api do 31 roda em dev na **3053**, no container `api-dev` (rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5212**, com `node_modules` em volume próprio. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Api que mande a entrada do CAPS na linha do tempo do cidadão** (fonte `psychosocial`, ou o marcador "Acompanhamento no CAPS …" da visão geral). Ela não aparece, nem como "Registro". Testes: Task 1, "history: entradas do CAPS saem antes da tela"; Task 2, "Meu histórico: nada do CAPS na linha do tempo nem nos documentos".
2. **Aviso com tipo do CAPS na caixa** (api com bug ou mais novo). Fica fora da lista e não abre detalhe. Testes: Task 1, "notices: tipo do CAPS fica fora"; Task 2, "Avisos: nada do CAPS na caixa".
3. **`psychosocial_care` vindo em `session.features`** (api sem W1). "Minhas triagens" fica com os mesmos botões que teria sem a chave. Teste: Task 2, "psychosocial_care na sessão: Minhas triagens igual".
4. **Filtro largo demais** (`kind` parecido que não é do CAPS, como `procedure`, `consultation` ou `clinical_document`, ou a condição `mental_health`). Continua aparecendo. Testes: Task 1, "isCapsOnlyKind: só o prefixo do CAPS" e "history: entradas do CAPS saem antes da tela" (as outras ficam, na ordem).
5. **Página do histórico só com entradas do CAPS** (api com bug). A tela mostra "Nenhum registro ainda." e mantém "Carregar mais", sem quebrar. Teste: Task 2, "página só com entradas do CAPS: vazio e 'Carregar mais'".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, rotas do api, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.caps.test.ts` | `isCapsOnlyKind` e o filtro em `history()`; ausência na caixa | 1 |
| `src/modules/citizen/Flow.caps.test.tsx`, `src/modules/citizen/MyHistoryStep.caps.test.tsx`, `src/modules/citizen/NoticesStep.caps.test.tsx`, `README.md` | specs de ausência nas telas; README | 2 |
| — | suíte, build, revisão, prova | 3 |

**Estratégia de teste:**
- Task 1: cliente com `fetch` falso, com os dados do CAPS no corpo cru (o caminho real de normalização).
- Task 2: telas com `fetch` falso (histórico e caixa) ou com dublês de `citizenApi` (o `Flow`, como em `Flow.trips.test.tsx`), conferindo que o texto da página não contém nenhum marcador do CAPS (`CAPS|psicossocial|psychosocial|F19|Sertralina|Registro`).
- Task 3: suíte inteira, e a prova contra o api do 31 com a semente do CAPS, inspecionando as respostas `/citizen/*`.

---

### Task 0: Cards, âncoras, rotas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`f923cdc` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `apps/api/.claude/mod31`.
- Produces: cards em In Progress; `$W`; a base; os `kind` reais da fonte do CAPS anotados.

- [ ] **Step 1: Cards F-31.1..F-31.8 para In Progress** — um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-31\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito em In Progress ou além. Card ausente → **pare e reporte**.

- [ ] **Step 2: Âncoras e o que o api do 31 fixou**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod31
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '^function normalizeEntry\|entries: (d.entries ?? \[\]).map(normalizeEntry)\|^export const DOCUMENT_KINDS\|if (n.kind !== undefined && n.kind !== "campaign") return null'
$G -C $R/apps/wpda grep -n -i "psychosocial\|psicossocial\|caps" origin/main -- src README.md
grep -rn "CAPS_ONLY_KINDS\|kind: \"psychosocial" $A/app/services/timeline
grep -n "psychosocial_care\|CITIZEN" $A/app/controllers/citizen_api/sessions_controller.rb $A/app/services/platform/features.rb
```

Expected:
- as quatro âncoras do `citizenApi.ts`;
- nenhuma ocorrência de `psychosocial`/`psicossocial`/`caps` no wpda;
- no api, os `kind` de `CAPS_ONLY_KINDS` e do marcador, e a sessão excluindo `psychosocial_care` (W1).

Se algum `kind` do CAPS **não** começar com `psychosocial`, acrescente-o como nome exato em `isCapsOnlyKind` (Task 1) e avise a sessão do api (W2). Se a sessão ainda listar `psychosocial_care`, avise a sessão do api; o wpda segue, porque a Task 2 cobre o caso.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod31 -b feat/mod-31-psychosocial-care origin/main
cd $R/.worktrees/wpda-mod31 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod31
```

Expected: verde e `tsc-ok`. **Anote** a base: a Task 3 espera **+4 arquivos de teste** (`citizenApi.caps`, `Flow.caps`, `MyHistoryStep.caps`, `NoticesStep.caps`).

---

### Task 1: Borda — entradas do CAPS fora do histórico; aviso do CAPS fora da caixa

**Files:**
- Modify: `src/lib/citizenApi.ts` (função nova logo antes de `normalizeEntry`; filtro no `return` de `history`)
- Test: `src/lib/citizenApi.caps.test.ts`

**Interfaces:**
- Consumes: `citizenApi.history`, `citizenApi.notices`, `DOCUMENT_KINDS` (existentes).
- Produces: `export function isCapsOnlyKind(kind: string): boolean`; `history()` nunca devolve entrada com `isCapsOnlyKind(entry.kind)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.caps.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, isCapsOnlyKind, DOCUMENT_KINDS } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function stubFetch(body: unknown) {
  const fetchMock = vi.fn(async () => new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

// Dados do CAPS que o api nunca deveria mandar (contrato §10): fonte psychosocial e o marcador da visão geral.
const CAPS_ACTION = { id: "psychosocial_action:a1", kind: "psychosocial_action", occurred_at: "2026-11-03T10:00:00-03:00",
  unit: { name: "CAPS AD Centro" }, professional_name: "Psic. Ana Lima", title: "Atendimento individual — F19.2", summary: "Sertralina 50 mg" };
const CAPS_MARKER = { id: "psychosocial_link:e1", kind: "psychosocial_link", occurred_at: "2026-10-01T00:00:00-03:00",
  unit: { name: "CAPS AD Centro" }, title: "Acompanhamento no CAPS AD Centro" };
const CAPS_BARE = { id: "psychosocial:x", kind: "psychosocial", occurred_at: "2026-10-02T00:00:00-03:00", title: "CAPS" };
const CONSULTATION = { id: "consultation:c1", kind: "consultation", occurred_at: "2026-10-20T09:00:00-03:00",
  unit: { name: "UBS Centro" }, title: "Consulta com Dra. Helena Prado" };
const PROCEDURE = { id: "procedure:p1", kind: "procedure", occurred_at: "2026-10-19T09:00:00-03:00", title: "Curativo" };

describe("citizenApi — nada do CAPS (módulo 31, decisão 4)", () => {
  it("isCapsOnlyKind: só o prefixo do CAPS", () => {
    expect([ "psychosocial", "psychosocial_action", "psychosocial_link", "psychosocial_enrollment" ].map(isCapsOnlyKind))
      .toEqual([ true, true, true, true ]);
    expect([ "procedure", "consultation", "clinical_document", "controlled_notification", "screening", "mental_health", "psychosocialx", "" ]
      .map(isCapsOnlyKind)).toEqual([ false, false, false, false, false, false, false, false ]);
  });

  it("history: entradas do CAPS saem antes da tela", async () => {
    stubFetch({ verified: true, partial: false, entries: [ CAPS_ACTION, CONSULTATION, CAPS_MARKER, PROCEDURE, CAPS_BARE ], next_before: "2026-10-01T00:00:00-03:00" });
    const h = await citizenApi.history("p1");
    expect(h.entries.map(e => e.id)).toEqual([ "consultation:c1", "procedure:p1" ]);
    expect(h.next_before).toBe("2026-10-01T00:00:00-03:00");
    expect(JSON.stringify(h)).not.toMatch(/CAPS|psychosocial|F19|Sertralina/);
  });

  it("history com kinds de documentos: o CAPS também sai", async () => {
    const fetchMock = stubFetch({ verified: true, partial: false,
      entries: [ { ...CAPS_ACTION, kind: "psychosocial_document" } ], next_before: null });
    const h = await citizenApi.history("p1", { kinds: DOCUMENT_KINDS });
    expect(h.entries).toEqual([]);
    expect(fetchMock.mock.calls[0][0]).toBe("/citizen/history?citizen_id=p1&kinds%5B%5D=clinical_document&limit=20");
  });

  it("notices: tipo do CAPS fica fora", async () => {
    stubFetch({ unread_count: 1, notices: [
      { kind: "psychosocial", id: "n1", title: "CAPS AD Centro", body: "Seu atendimento no CAPS", read: false },
      { kind: "psychosocial_appointment", id: "n2", scheduled_at: "2026-11-05T09:00:00-03:00", read: false },
      { kind: "campaign", id: "n3", title: "Vacinação", body: "Campanha", dispatched_at: "2026-11-01T09:00:00-03:00", read: true }
    ] });
    expect((await citizenApi.notices()).notices.map(n => n.id)).toEqual([ "n3" ]);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod31 src/lib/citizenApi.caps.test.ts`. Expected: FAIL, porque `isCapsOnlyKind` não existe. O caso `notices` já passaria sozinho: ele fixa a regra atual de `normalizeNotice`.

- [ ] **Step 3: Implemente** `isCapsOnlyKind(kind: string): boolean` (`kind === "psychosocial" || kind.startsWith("psychosocial_")`). Acima dela, um comentário de uma linha: `// Módulo 31 (ADR 0049, decisão 4): nada do CAPS chega ao cidadão; o api já filtra (contrato §10), isto é a defesa da borda.` Em `history`, troque `entries: (d.entries ?? []).map(normalizeEntry)` por `.map(normalizeEntry).filter((e) => !isCapsOnlyKind(e.kind))`. Nada mais muda: `next_before` e `partial` vêm como o api mandou, e `normalizeNotice` fica igual.

- [ ] **Step 4: Run** o mesmo comando, `bin/test wpda -w wpda-mod31 src/lib` e `cd $W && npx tsc --noEmit`. Expected: PASS (os `citizenApi.*` antigos iguais).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/citizenApi.ts src/lib/citizenApi.caps.test.ts
$G commit -m "fix: drop psychosocial care entries from the citizen history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Specs de ausência nas telas e README

**Files:**
- Test: `src/modules/citizen/Flow.caps.test.tsx`, `src/modules/citizen/MyHistoryStep.caps.test.tsx`, `src/modules/citizen/NoticesStep.caps.test.tsx`
- Modify: `README.md` (parágrafo **Atenção psicossocial (módulo 31).** depois do de Minhas viagens)

**Interfaces:**
- Consumes: `isCapsOnlyKind` e o filtro de `history()` (Task 1); `Flow`, `MyHistoryStep`, `NoticesStep`, `setSessionFeatures` (existentes); o `signedIn` de `Flow.trips.test.tsx` (copiado, sem o spy de `trips`).
- Produces: nada além dos testes.

- [ ] **Step 1: Write the tests**

`src/modules/citizen/Flow.caps.test.tsx`. Copie `beforeEach`, `afterEach` e `signedIn(features)` de `Flow.trips.test.tsx`; `signedIn` devolve o spy de `history` e não espiona `trips`. No `appointments`, use um agendamento comum da UBS:

```tsx
async function triageButtons(features: string[]) {
  signedIn(features);
  const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
  const view = render(<Flow />);
  await user.click(await screen.findByRole("button", { name: "Concordo" }));
  await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
  await screen.findByRole("heading", { level: 1, name: "Minhas triagens" });
  const names = screen.getAllByRole("button").map(b => b.textContent);
  const text = document.body.textContent;
  view.unmount(); vi.restoreAllMocks(); setSessionFeatures(null);
  return { names, text };
}

describe("Flow — nada do CAPS (módulo 31)", () => {
  it("psychosocial_care na sessão: Minhas triagens igual", async () => {
    const base = [ "patient_timeline", "clinical_record" ];
    const without = await triageButtons(base);
    const withCaps = await triageButtons([ ...base, "psychosocial_care" ]);
    expect(withCaps.names).toEqual(without.names);
    expect(withCaps.text).not.toMatch(/CAPS|psicossocial|psychosocial/i);
  });
});
```

`src/modules/citizen/MyHistoryStep.caps.test.tsx`. Use `fetch` falso por URL, para que a normalização da Task 1 rode, com relógio em `2026-11-04T10:00:00-03:00`. Dados: `CAPS_ACTION`, `CAPS_MARKER` e `CONSULTATION`, iguais aos da Task 1, e um `DOCUMENT` comum da UBS no formato de `MyHistoryStep.test.tsx`:

```tsx
function stubCitizenApi(timeline: unknown[], documents: unknown[], next_before: string | null = null) {
  const fetchMock = vi.fn(async (url: string) => {
    const json = (body: unknown) => new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } });
    if (url.startsWith("/citizen/health_summary")) return json({ problems: [], medications: [], exam_requests: [], vitals: {} });
    if (url.includes("kinds%5B%5D=")) return json({ verified: true, partial: false, entries: documents, next_before: null });
    return json({ verified: true, partial: false, entries: timeline, next_before });
  });
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

describe("MyHistoryStep — nada do CAPS (módulo 31)", () => {
  it("Meu histórico: nada do CAPS na linha do tempo nem nos documentos", async () => {
    stubCitizenApi([ CAPS_ACTION, CONSULTATION, CAPS_MARKER ], [ { ...CAPS_ACTION, kind: "psychosocial_document" }, DOCUMENT ]);
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    const timeline = within(await screen.findByRole("region", { name: "Linha do tempo" }));
    expect(await timeline.findByText("Consulta com Dra. Helena Prado")).toBeInTheDocument();
    const docs = within(await screen.findByRole("region", { name: "Documentos" }));
    expect(docs.getByText("Receita")).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/CAPS|psicossocial|psychosocial|F19|Sertralina|Psic\. Ana Lima|Registro/);
  });

  it("página só com entradas do CAPS: vazio e 'Carregar mais'", async () => {
    stubCitizenApi([ CAPS_ACTION, CAPS_MARKER ], [], "2026-10-01T00:00:00-03:00");
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    expect(await screen.findByText("Nenhum registro ainda.")).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Carregar mais" })).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/CAPS|Registro/);
  });
});
```


`src/modules/citizen/NoticesStep.caps.test.tsx`. Use `fetch` falso para `/citizen/notices` com os três avisos do teste `notices` da Task 1 (`readNotice` espiado):

```tsx
describe("NoticesStep — nada do CAPS (módulo 31)", () => {
  it("Avisos: nada do CAPS na caixa", async () => {
    render(<NoticesStep onBack={vi.fn()} onPreferences={vi.fn()} />);
    expect(await screen.findByRole("button", { name: /Vacinação/ })).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: /CAPS|atendimento/i })).toBeNull();
    expect(document.body.textContent).not.toMatch(/CAPS|psicossocial|psychosocial|undefined/);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod31 src/modules/citizen/Flow.caps.test.tsx src/modules/citizen/MyHistoryStep.caps.test.tsx src/modules/citizen/NoticesStep.caps.test.tsx`. Expected: PASS, porque a Task 1 já está aplicada. Para conferir que os testes mordem, reverta só o `.filter` de `history()` (`$G stash push src/lib/citizenApi.ts`). Os dois casos do `MyHistoryStep.caps` devem falhar com "Registro"/"CAPS" na página. Depois `$G stash pop`.

- [ ] **Step 3: README** — o parágrafo **Atenção psicossocial (módulo 31).** diz:
  - o CAPS não aparece no canal web: não há tela, aviso nem SMS;
  - "Seus agendamentos" é a agenda do 17, que o CAPS não usa;
  - a linha do tempo, os documentos e o resumo não trazem nada do CAPS, porque o api filtra (contrato §10);
  - o cliente descarta, por defesa, entrada de `kind` `psychosocial*`;
  - `psychosocial_care` não muda nenhum botão;
  - a condição declarada "saúde mental" do módulo 23 é do cidadão e continua.

- [ ] **Step 4: Run** `bin/test wpda -w wpda-mod31 src/modules/citizen src/theme && cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/Flow.caps.test.tsx src/modules/citizen/MyHistoryStep.caps.test.tsx src/modules/citizen/NoticesStep.caps.test.tsx README.md
$G commit -m "test: pin that nothing from psychosocial care reaches the citizen screens

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test wpda -w wpda-mod31 && cd $W && npx tsc --noEmit && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base mais 4 arquivos de teste;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Nada do CAPS no código de tela**

```bash
cd $W && $G diff --stat origin/main..HEAD -- src | grep -v "\.test\.ts" ; grep -rn -i "caps\|psicossocial\|psychosocial" src --include=*.tsx | grep -v "\.test\.tsx"
```

Expected:
- o primeiro comando lista só `src/lib/citizenApi.ts`;
- o segundo, sem saída: nenhuma tela cita o CAPS.

- [ ] **Step 3: Revisão** — subagente revisor lê `origin/main..HEAD` contra a spec (decisões 4, 18, 19, 21, §5, F-31.8) e o contrato §9/§10. Atenção:
  - única mudança de produção é o filtro;
  - o filtro não pega `kind` alheio nem `mental_health`;
  - os testes mordem (Task 2, Step 2);
  - nenhum texto novo de tela.

- [ ] **Step 4: Prova no navegador (com o usuário)**

O api do 31 deve estar na **3053**, com a semente do plano do api: Curitiba (`record`) com `psychosocial_care` ligado; um celular com uma pessoa **validada** que também é paciente matriculado num CAPS AD, com:
- ações individuais e coletivas;
- um PTS;
- uma receita de controle especial emitida a partir de uma ação do CAPS (19d, `psychosocial_action_id`);
- um caso de matriciamento;
- uma consulta comum na UBS.

Sem essa pessoa na semente, **pare** e peça à sessão do api.

Suba o Vite:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod31 --network rota-saude_default -p 5212:5212 \
  -v $R/.worktrees/wpda-mod31:/app -v wpda-mod31-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3053 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5212 --host 0.0.0.0"
docker logs -f wpda-mod31   # espere o "ready"
```

Abra `http://curitiba.localhost:5212/wpda/` em 375 px, com toque. O usuário entra com o celular da semente e o código SMS de dev. Guarde os screenshots em `evidencias/2026-11-04-modulo-31/wpda/`. Confira:
  - "Minhas triagens": os mesmos botões de antes do 31, e nenhum cita o CAPS;
  - "Seus agendamentos": só o que é da agenda do 17, sem horário do CAPS;
  - "Meu histórico": a consulta da UBS aparece; nenhuma entrada do CAPS, nem o marcador; em "Documentos", a receita do CAPS não aparece; o resumo não traz CID nem medicamento do CAPS (W3);
  - "Avisos": nada do CAPS; com o celular de dev, nenhum SMS do CAPS no registro do `SmsGateway`;
  - nas respostas (`read_network_requests` ou DevTools): `GET /citizen/session` sem `psychosocial_care` em `features` (W1); nenhuma resposta `/citizen/*` contém `psychosocial`, o nome do CAPS, o CID nem a substância da semente;
  - `GET /citizen/documents/<id da receita do CAPS>/pdf?citizen_id=<pessoa>`, aberto no navegador, responde 404 (o id vem do dashboard; ele não aparece no wpda);
  - no maintenance, desligar `psychosocial_care`, sair e entrar: nada muda no wpda. Depois religue.

Ao terminar: `docker rm -f wpda-mod31 && docker volume rm wpda-mod31-node-modules`.

- [ ] **Step 5: Card** — se as partes do api e do dashboard de F-31.8 estiverem **Done**, mova o F-31.8 para **Done** (`98236657`), e para **Verified** se a prova passou. Senão, deixe como está e avise a sessão do api.

- [ ] **Step 6: Pare.** O merge só vem depois do api, do dashboard e do maintenance do 31, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-31-psychosocial-care`, `push origin main`, `worktree remove $W`, `branch -d`. O `wpda-dev` (5176) recompila sozinho.

---

## Divergências propostas ao contrato

1. **W1 — `psychosocial_care` vaza em `features` da sessão do cidadão.** Hoje `session_json` (`app/controllers/citizen_api/sessions_controller.rb`) devolve **todo** interruptor utilizável da cidade. Com o 31 ligado, `GET /citizen/session` passaria a dizer ao celular que a cidade tem CAPS, e o §10 não trata disso.
   - Proposta: o api exclui a chave. A forma mais simples é uma lista `Platform::Features::CITIZEN_HIDDEN = %w[psychosocial_care]` aplicada no `select`, com request spec no plano do api.
   - Alternativa: uma lista de permissão das chaves que o wpda usa. É mais segura para chaves futuras, mas muda o que já é publicado e precisa ser mantida a cada módulo.
   - Recomendação: a lista de exclusão, pela simplicidade. O wpda não depende disso: a Task 2 cobre o caso.
2. **W2 — Nomes dos `kind` da fonte do CAPS e do marcador.** O §10 cria `CAPS_ONLY_KINDS` e o marcador "Acompanhamento no CAPS <nome>", mas não fixa os `kind`.
   - Proposta: todos começam com `psychosocial` (ex.: `psychosocial_action`, `psychosocial_enrollment`, marcador `psychosocial_link`), para o filtro de borda do wpda (Task 1).
   - O §10 também deve dizer explicitamente que o marcador fica **fora da visão do cidadão**. Ele vai só para a visão profissional geral (spec decisão 21).
3. **W3 — Resumo de saúde do cidadão.** `GET /citizen/health_summary` (`Timeline::CitizenSummary`) lista `patient.problems` ativos e `PatientMedication` ativos. O §9/§10 cobre documentos, linha do tempo e LEDI, mas não o resumo.
   - Proposta: o CAPS não grava CID em `problems` nem medicamento em `patient_medications` (os CIDs ficam cifrados nas tabelas `psychosocial_*`, §14).
   - Se a receita do CAPS alimentar `patient_medications`, o resumo filtra pela origem `psychosocial_action_id`.
   - O spec de request fica no plano do api.
4. **W4 — A ausência é provada no api.** O wpda não distingue um documento do CAPS (`kind: "clinical_document"`) nem um agendamento do CAPS de um comum. Proposta: o plano do api tem request specs, com a semente do CAPS (matrícula, ação, receita com `psychosocial_action_id`, matriciamento), para:
   - `GET /citizen/history`, com e sem `kinds[]=clinical_document`: sem entrada do CAPS nem marcador;
   - `GET /citizen/documents/:id/pdf` do documento do CAPS: 404;
   - `GET /citizen/health_summary`: W3;
   - `GET /citizen/notices`, `GET /citizen/appointments` e `GET /citizen/session`: W1;
   - nenhum SMS enfileirado.

   O spec do wpda (Tasks 1 e 2) cobre só a defesa da borda e as telas.

## Self-review

- **Cobertura (decisões 4, 18, 19, 21, §5, F-31.8; contrato §9, §10):**
  - linha do tempo e documentos: Task 1 (filtro) e Task 2 (`MyHistoryStep.caps`) (Review Focus 1, 4, 5; W2);
  - avisos: Tasks 1 e 2 (Review Focus 2);
  - interruptor e catálogo do cidadão: Task 2 (`Flow.caps`) (Review Focus 3; W1);
  - agendamentos ("Minhas consultas"), resumo e PDF: garantidos no api (W3, W4) e conferidos na prova (Task 3);
  - SMS: prova.
  - Nenhuma tela nova, como pede a decisão 4. A única mudança de produção é o filtro de borda; o resto são testes e README.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards, `kind` reais do api (Task 0, com a regra de ajuste), semente (Task 3).
- **Tipos e nomes:** `isCapsOnlyKind` (Task 1) é usado só em `history()`. Existentes conferidos em `origin/main` (`f923cdc`): `citizenApi.history`, `citizenApi.notices`, `normalizeEntry`, `normalizeNotice`, `DOCUMENT_KINDS`, `HISTORY_PAGE_SIZE` (20, na URL do teste), `citizenKindLabel` ("Registro"), `setSessionFeatures`, `hasSessionFeature`, `MyHistoryStep`, `NoticesStep`, `Flow`, e o `signedIn` de `Flow.trips.test.tsx`.
- **Review Focus:** 1 — Tasks 1 e 2; 2 — Tasks 1 e 2; 3 — Task 2; 4 — Task 1; 5 — Task 2.
- **Proporção:** código só em testes; a mudança de produção é uma função de uma linha e um `.filter`.
