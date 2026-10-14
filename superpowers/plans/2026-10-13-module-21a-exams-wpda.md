# Módulo 21a — Rastreamento do exame (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21a** rodando na branch dele (`feat/mod-21a-exams`, porta **3041**) com `GET /citizen/exams`, `GET /citizen/exams/:id/pdf`, a entrada `exam` no `GET /citizen/history` e `exam_tracking` em `features` da sessão do cidadão (Divergências E1, E2 e E5 deste plano). Ele é **mergeado antes** deste (contrato §14: contracts → api → dashboard e wpda → maintenance → catálogo). As Tasks 0–6 não dependem dele (os testes usam dublês e o Playwright responde as rotas); a prova da Task 7 depende.

**Goal:** Lado wpda de F-21.10: a tela **"Meus exames"** de cada pessoa do celular. Par **validado**: a situação de cada exame, a data e o local marcados, o laudo pela política do exame (`immediate` mostra ao registrar, `after_acknowledgement` só depois da ciência, `never_citizen` diz "retire o resultado na sua unidade") e o download do PDF. Par **só declarado**: o aviso e o caminho para validar no posto. E a entrada `exam` em **"Meu histórico"** (contrato §10), com o PDF pela rota do exame. Nunca motivo de negação nem a marca de crítico.

**Architecture:** O cliente (`src/lib/citizenApi.ts`) ganha `exams` e `examPdf` por pessoa, com a normalização na borda como o resto do arquivo, e a entrada da linha do tempo ganha `status_label` opcional. As regras de tela ficam em `src/lib/exams.ts` (puras: o que mostrar do laudo a partir de `result`, frase da marcação, junção das páginas, nome seguro do arquivo, frases). A tela nova `src/modules/citizen/MyExamsStep.tsx` copia a forma de `MyHistoryStep` (`Screen`, `BigButton`, `ErrorText`, cartões com `var(--rule2)`, download por `blob:`). O `Flow` ganha o estado `exams`; "Minhas triagens" (`HistoryStep`) ganha o botão **"Meus exames"** só com `exam_tracking` na sessão. Quem vê o quê é do api (par validado, política do laudo, rótulo da situação); a tela **nunca deduz** o laudo da situação e nunca pede o PDF quando o `result` não o oferece.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `setupFiles` com `cleanup` e o dublê de celular `stubDevice`), `vi.spyOn(citizenApi, …)`, Playwright 1.63 (5 perfis de dispositivo, `e2e/`), Vite 5 (base `/wpda/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-13-module-21a-exam-tracking-design.md` (§3 `exam_tracking`, §6 "Cidadão (wpda)", §6 "Linha do tempo", §7, §8 "dashboard e wpda" e "Prova no navegador", §9 "Cidadão", §11 F-21.10 e F-21.13), `docs/adr/0036.md` e o contrato `docs/superpowers/plans/2026-10-13-module-21a-exams-contracts.md` (§1, §9, §10; as diferenças estão em "Divergências propostas ao contrato"). Herdados: ADR 0017 (par CPF + celular, validação presencial), ADR 0035 e o plano do wpda do módulo 20 (invariantes do cidadão, Divergências W1–W4 já em produção).

## Global Constraints

- Rotas (contrato §9, na forma das Divergências E1–E5; sessão do cidadão, cookie `citizen_session`, mesma origem):
  - `GET /citizen/exams?citizen_id=&before=&limit=20` → `{ verified, exams: [<citizen_exam>], next_before|null }`; `citizen_id` que não é do celular da sessão → 404 `not_found`; par só declarado → `verified: false` e `exams: []` (a tela também aceita 403 `not_verified` como o mesmo caso);
  - `<citizen_exam>` = `{ id, label, status_label, requested_at, scheduled_at|null, scheduled_place|null, result: { available, pickup_at_unit, released_on|null, pdf_available }|null }`;
  - `GET /citizen/exams/:id/pdf?citizen_id=` → `application/pdf`; 404 `not_found` (política não permite, cancelado, de outra pessoa ou inexistente, sem distinguir); 403 `not_verified`; 429 `too_many_requests` (mesmo limite dos documentos);
  - `GET /citizen/history` (módulo 20) passa a trazer `kind: "exam"` com `status_label` e, quando a política permite, `document: { id: <exam_order id>, short_code: null, verification_url: null, downloadable: true }`, cujo PDF sai de `GET /citizen/exams/:id/pdf` (E5);
  - `GET /citizen/session` → `features` com `exam_tracking` quando ligado e utilizável (E2);
  - 403 `feature_disabled` (`feature: "exam_tracking"`) em qualquer uma; 401 dispara `citizen:unauthenticated`, como hoje.
- **A política do laudo é do api** (spec §6 e §7): a tela mostra o botão de PDF só com `result.available && result.pdf_available && !result.pickup_at_unit`; `pickup_at_unit` vence tudo e mostra `PICKUP_AT_UNIT`; `result: null` não mostra nada sobre o laudo. A tela nunca lê a situação (`status_label`) para decidir o laudo.
- **Nunca** motivo de negação, `critical`, valores do resultado, solicitante, CPF ou CNS na tela nova: ela só desenha os campos de `<citizen_exam>` acima (campo a mais que o api mandar é ignorado). Par declarado: nenhum exame aparece, mesmo que o api mande algum.
- Na URL vão só o id da pessoa, o id do exame, o cursor e o `limit` — nunca CPF, celular, nome do exame ou texto. O PDF é lido com `fetch` e salvo por um endereço `blob:` com nome `exame-<id>.pdf` (só letras, números e hífen); nada em `localStorage`, `sessionStorage` ou `console`.
- Interruptor (spec §3): o botão "Meus exames" só aparece com `exam_tracking` em `session.features`; sem a chave (api antigo ou desligado), "Minhas triagens" fica como hoje. Se o interruptor cair com a tela aberta, o 403 `feature_disabled` vira `EXAMS_DISABLED` ("Os exames ainda não aparecem por aqui nesta cidade."), não a frase do histórico.
- Interface do canal (ciclo de interface, subprojetos 4 e 6e): `Screen`, `BigButton`, `ErrorText` de `ui.tsx`; alvos de toque de 48 px ou mais (o `BigButton` tem 56), texto de 18 px; cores **só por papel do tema** (`var(--warn-bg)`, `var(--accent-bg)`, `var(--rule2)`, `var(--ink2)`), sem reserva `var(--x, …)` e sem branco fixo — a guarda `src/theme/guards.test.ts` roda na suíte; o fluxo do cidadão só abre no celular (`isPhone`, `src/lib/device.ts`) e o `e2e/device-gate.spec.ts` segue verde nos 5 perfis. Nenhum componente comum muda. Achado só visual não vira tarefa.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` e `vi.useRealTimers()` no `afterEach`; a suíte roda em `TZ=America/Sao_Paulo`.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree **`/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/wpda-mod21`** (nunca dentro de `apps/wpda`: o compose monta `./apps/wpda:/app` e um segundo React derruba o `wpda-dev`), branch `feat/mod-21a-exams` a partir de `origin/main`, com `npm ci` (Task 0). Caminhos de arquivo das tasks relativos a ele.
- Variáveis usadas nos comandos: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod21`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos: `npx tsc --noEmit`. e2e: `cd $W && WPDA_URL=http://curitiba.localhost:5193 npx playwright test` (precisa do Vite da Task 6 no ar).
- Ambiente de teste: `vitest.config.ts` com `environment: "jsdom"`, `globals: false`, `setupFiles: ["./src/test/setup.ts"]` (jest-dom, `cleanup`, `sessionStorage.clear()` e o celular padrão `stubDevice()`), `exclude` de `e2e/**` e `TZ=America/Sao_Paulo`. `fmtDateTime("2026-10-05T07:30:00-03:00")` → `"05/10/2026, 07:30"`; `fmtDate("2026-10-01T09:00:00-03:00")` → `"01/10/2026"`; `fmtCalendarDate("2026-10-08")` → `"08/10/2026"`.
- O api do módulo 21a roda em dev na **3041**, dentro do container `api-dev` (rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5193**, com `node_modules` num volume próprio (o `node_modules` do worktree é de macOS). `/citizen` já está no proxy do `vite.config.ts`: nada muda nele.

## Review Focus

1. **Par só declarado recebe exames.** Um api que mande `verified: false` com a lista cheia, ou responda 403 `not_verified`, não pode fazer a tela mostrar exame nenhum: só o convite e "Validar no posto". Testes: Task 3, "par declarado: convite e nenhum exame, mesmo se o api mandar" e "403 not_verified: o mesmo convite".
2. **Laudo fora da política.** `never_citizen` com `pdf_available: true` por engano, `after_acknowledgement` antes da ciência (`result: null` ou `available: false`), ou `available` sem PDF: a pessoa não pode ver botão de PDF nem "Resultado disponível" que o api não liberou; "retire na unidade" vence. Testes: Task 2, "examResultView: tabela das políticas"; Task 3, "par validado: situação, marcação e laudo pelas três políticas".
3. **Baixar o PDF dá errado.** Ciência desfeita, exame cancelado ou política trocada entre a lista e o toque (404), sessão vencida (401), limite (429), interruptor desligado (403): a tela diz o que houve em português, sem abrir janela vazia, e o arquivo nunca leva nome do exame, CPF ou nome. Testes: Task 1, "examPdf: blob, recusa com código e 401"; Task 2, "examFilename: só o id"; Task 3, "baixar o resultado", "resultado que deixou de estar disponível (404)", "limite de downloads (429)" e "interruptor desligado".
4. **Exame na linha do tempo baixado pela rota errada.** A entrada `exam` de "Meu histórico" tem `document.id` = id do exame; baixar por `/citizen/documents/:id/pdf` daria 404 para sempre (ou, pior, o documento de mesmo id). O toque tem de ir a `examPdf`, com nome `exame-<id>.pdf`, e a situação vem de `status_label`. Testes: Task 5, "exame na linha do tempo: rótulo, situação e PDF pela rota do exame" e "exame que sumiu na linha do tempo: a frase do exame".
5. **Api antigo, interruptor desligado ou campo faltando.** Sem `exam_tracking` na sessão o botão não aparece e "Minhas triagens" fica igual; `result` incompleto, `scheduled_place` nulo ou `released_on` ausente não viram `undefined`/`null` na tela. Testes: Task 1, "exams: normaliza o que faltar"; Task 3, "campos ausentes: nada de undefined ou null"; Task 4, "sem exam_tracking na sessão: Minhas triagens como hoje".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card F-21.10, conferência do main e do api do 21a, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.exams.test.ts` | tipos do contrato §9; `exams`, `examPdf`; `status_label` opcional na entrada da linha do tempo | 1 |
| `src/lib/exams.ts`, `src/lib/exams.test.ts` | regras de tela de "Meus exames" (laudo pela política, marcação, junção, nome do PDF, frases) | 2 |
| `src/modules/citizen/MyExamsStep.tsx`, `src/modules/citizen/MyExamsStep.test.tsx` | a tela "Meus exames" | 3 |
| `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.exams.test.tsx`, `README.md` | o botão em "Minhas triagens" e o estado `exams` no `Flow` | 4 |
| `src/lib/history.ts`, `src/modules/citizen/MyHistoryStep.tsx`, `src/modules/citizen/MyHistoryStep.exams.test.tsx` | a entrada `exam` em "Meu histórico" (rótulo, situação, PDF pela rota do exame) | 5 |
| `e2e/exams.spec.ts` | "Meus exames" nos perfis de celular do Playwright, com as rotas respondidas pelo teste | 6 |
| — | suíte, build, e2e, revisão e prova no navegador | 7 |

**Estratégia de teste:** cliente com `fetch` falso conferindo URL (por pessoa, cursor, `limit`, nada pessoal), normalização, PDF binário e o 401 (Task 1); regras puras com tabela de casos, em especial as combinações de `result` das três políticas (Task 2); a tela com `vi.spyOn(citizenApi, …)` cobrindo par validado, par declarado (inclusive api que manda exames), as três políticas, "Carregar mais", download (com `URL.createObjectURL` falso) e cada recusa que muda a tela (Task 3); a navegação real pelo `Flow` com a sessão com e sem o interruptor (Task 4); a entrada `exam` na linha do tempo (Task 5); o Playwright nos perfis de celular, com as rotas respondidas pelo teste, para a tela real no navegador, o download e a largura de 375–412 px sem rolagem lateral (Task 6). A prova final é no navegador, contra o api do 21a com o `fake-lab` (Task 7).

---

### Task 0: Card, conferência do main e do api do 21a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`7336dd6` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`).
- Produces: card F-21.10 em In Progress; worktree `$R/.worktrees/wpda-mod21` na branch `feat/mod-21a-exams`; a base anotada.

- [ ] **Step 1: Mova o card F-21.10 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem variável na frente nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | startswith("F-21.10")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Com o `id` do item (`PVTI_…`) e o id da opção "In Progress" lidos acima:

```bash
gh project item-edit --id <PVTI_do_F-21.10> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: o item aparece com `status: "In Progress"` numa nova leitura. Se o card F-21.10 não existir, **pare e reporte** (os cards do 21a são criados pela sessão dona do módulo).

- [ ] **Step 2: Confira as peças que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda fetch origin
$G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n \
  "^export interface CitizenSession\|^async function pdf\|  documentPdf: (documentId: string, citizenId: string)\|    pdf(\`/documents/\|title: e.title ?? \"\", summary: e.summary ?? null, status: e.status ?? null,\|  title: string; summary: string | null; status: string | null;"
$G -C $R/apps/wpda show origin/main:src/modules/citizen/HistoryStep.tsx | grep -n \
  "onRecord?: () => void }) {\|{onRecord && <BigButton"
$G -C $R/apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n \
  '| { at: "record"; consentVersion: string | null; citizenId: string }\|import { MyHistoryStep }\|    case "verify-code":\|          : undefined} />;'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/MyHistoryStep.tsx | grep -n \
  "async function download(doc: CitizenDocumentRef, list: ListKey)\|const status = citizenStatusLabel(e.status);\|onClick={() => onDownload(e.document as CitizenDocumentRef)}"
```

Expected: cada `grep` acha as linhas pedidas (as âncoras das edições). Se alguma mudou, anote: as tasks aplicam a mesma intenção sobre o texto real.

- [ ] **Step 3: Confira o api do 21a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-21a-exams
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod21 api bin/rails routes 2>/dev/null | grep -E "citizen.*exams" | head
grep -rn "exam_tracking" $R/apps/api/.claude/mod21/app/controllers/citizen_api 2>/dev/null | head -5
```

Expected: a branch e as duas rotas (`GET /citizen/exams`, `GET /citizen/exams/:id/pdf`). Confira no controller do cidadão do api que a pessoa vem em `citizen_id` (E1), que a sessão traz `exam_tracking` em `features` (E2) e que a entrada `exam` do histórico traz `status_label` (E5). Se faltar algo, as Tasks 1–6 seguem (os testes usam dublês), mas **reporte** antes da Task 7: as divergências precisam ser decididas para a prova.

- [ ] **Step 4: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add $R/.worktrees/wpda-mod21 -b feat/mod-21a-exams origin/main
cd $R/.worktrees/wpda-mod21 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 7 espera a base mais **5 arquivos de teste novos** (`citizenApi.exams`, `exams`, `MyExamsStep`, `Flow.exams`, `MyHistoryStep.exams`).

---

### Task 1: Cliente de "Meus exames" e `status_label` na linha do tempo

**Files:**
- Modify: `src/lib/citizenApi.ts`
- Test: `src/lib/citizenApi.exams.test.ts`

**Interfaces:**
- Consumes: `call`, `pdf`, `ApiError`, `normalizeEntry` (existentes em `citizenApi.ts`).
- Produces:
  ```ts
  export interface CitizenExamResult { available: boolean; pickup_at_unit: boolean; released_on: string | null; pdf_available: boolean }
  export interface CitizenExam {
    id: string; label: string; status_label: string; requested_at: string;
    scheduled_at: string | null; scheduled_place: string | null; result: CitizenExamResult | null;
  }
  export interface CitizenExams { verified: boolean; exams: CitizenExam[]; next_before: string | null }
  export const EXAMS_PAGE_SIZE = 20;
  // CitizenEntry ganha status_label?: string | null (só presente quando o api manda)
  citizenApi.exams(citizenId: string, opts?: { before?: string | null }): Promise<CitizenExams>;
  citizenApi.examPdf(examId: string, citizenId: string): Promise<Blob>;
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.exams.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError, EXAMS_PAGE_SIZE } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function mockJson(status: number, body: unknown) {
  const fn = vi.fn(async () => new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const urlOf = (fn: ReturnType<typeof vi.fn>, i = 0) => new URL(String(fn.mock.calls[i][0]), "http://curitiba.localhost");

describe("citizenApi — Meus exames (módulo 21a)", () => {
  it("exams: citizen_id, limit 20 e o cursor como veio; nada pessoal na URL", async () => {
    const fn = mockJson(200, { verified: true, exams: [], next_before: null });
    await citizenApi.exams("p1");
    expect(urlOf(fn).pathname).toBe("/citizen/exams");
    expect([ ...urlOf(fn).searchParams.entries() ]).toEqual([ [ "citizen_id", "p1" ], [ "limit", String(EXAMS_PAGE_SIZE) ] ]);
    expect((fn.mock.calls[0][1] as RequestInit).method).toBe("GET");

    await citizenApi.exams("p 1", { before: "2026-10-01T09:00:00-03:00|ex9" });
    const url = urlOf(fn, 1);
    expect(url.searchParams.get("citizen_id")).toBe("p 1");
    expect(url.searchParams.get("before")).toBe("2026-10-01T09:00:00-03:00|ex9");
    expect(String(fn.mock.calls[1][0])).not.toMatch(/cpf|phone|\d{11}/i);
  });

  it("exams: normaliza o que faltar (api mais novo ou incompleto) e ignora campo a mais", async () => {
    mockJson(200, { exams: [
      { id: "ex1", label: "Hemograma completo", status_label: "Pedido", requested_at: "2026-10-01T09:00:00-03:00",
        critical: true, denial_reason: "fora do protocolo" },
      { id: "ex2", label: "Hemoglobina glicada", status_label: "Resultado disponível", requested_at: "2026-10-01T09:00:00-03:00",
        scheduled_at: "2026-10-05T07:30:00-03:00", result: { available: true } }
    ] });
    const r = await citizenApi.exams("p1");
    expect(r.verified).toBe(false);
    expect(r.next_before).toBeNull();
    expect(r.exams[0]).toEqual({
      id: "ex1", label: "Hemograma completo", status_label: "Pedido", requested_at: "2026-10-01T09:00:00-03:00",
      scheduled_at: null, scheduled_place: null, result: null
    });
    expect(r.exams[1].result).toEqual({ available: true, pickup_at_unit: false, released_on: null, pdf_available: false });
    expect(r.exams[1].scheduled_place).toBeNull();
  });

  it("exams: 403 feature_disabled e 404 not_found viram ApiError com o código", async () => {
    mockJson(403, { error: "feature_disabled", feature: "exam_tracking" });
    await expect(citizenApi.exams("p1")).rejects.toEqual(new ApiError(403, "feature_disabled"));
    mockJson(404, { error: "not_found" });
    await expect(citizenApi.exams("p9")).rejects.toEqual(new ApiError(404, "not_found"));
  });

  it("examPdf: blob, recusa com código e 401", async () => {
    const pdf = new Blob([ "%PDF-1.7" ], { type: "application/pdf" });
    const fn = vi.fn(async () => new Response(pdf, { status: 200, headers: { "Content-Type": "application/pdf" } }));
    vi.stubGlobal("fetch", fn);
    const blob = await citizenApi.examPdf("ex 1", "p1");
    expect(blob.type).toBe("application/pdf");
    const [ url, init ] = fn.mock.calls[0] as unknown as [ string, RequestInit ];
    expect(url).toBe("/citizen/exams/ex%201/pdf?citizen_id=p1");
    expect(init.credentials).toBe("same-origin");
    expect((init.headers as Record<string, string>).Accept).toBe("application/pdf");

    mockJson(404, { error: "not_found" });
    await expect(citizenApi.examPdf("ex1", "p1")).rejects.toEqual(new ApiError(404, "not_found"));

    mockJson(401, { error: "unauthenticated" });
    const spy = vi.spyOn(window, "dispatchEvent");
    await expect(citizenApi.examPdf("ex1", "p1")).rejects.toEqual(new ApiError(401, "unauthenticated"));
    expect(spy).toHaveBeenCalledWith(expect.objectContaining({ type: "citizen:unauthenticated" }));
  });

  it("history: status_label do exame passa; sem ele, a entrada fica como no módulo 20", async () => {
    mockJson(200, { verified: true, entries: [
      { id: "exam:ex1", kind: "exam", occurred_at: "2026-10-08T10:00:00-03:00", title: "Hemograma completo",
        status: "resulted", status_label: "Resultado disponível", document: { id: "ex1", downloadable: true } },
      { id: "triage:t1", kind: "triage", occurred_at: "2026-09-09T20:15:00-03:00", title: "Triagem respiratória" }
    ] });
    const h = await citizenApi.history("p1");
    expect(h.entries[0].status_label).toBe("Resultado disponível");
    expect(h.entries[0].document).toEqual({ id: "ex1", short_code: null, verification_url: null, downloadable: true });
    expect("status_label" in h.entries[1]).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/citizenApi.exams.test.ts`
Expected: FAIL — `EXAMS_PAGE_SIZE` não é exportado; `citizenApi.exams` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/citizenApi.ts`, na interface `CitizenEntry`, troque a linha

```ts
  title: string; summary: string | null; status: string | null;
```

por

```ts
  title: string; summary: string | null; status: string | null;
  // Módulo 21a (contrato §10, Divergência E5): situação já em palavras do
  // cidadão (entrada `exam`). Presente só quando o api manda; vence o `status`.
  status_label?: string | null;
```

Em `normalizeEntry`, troque

```ts
    title: e.title ?? "", summary: e.summary ?? null, status: e.status ?? null,
```

por

```ts
    title: e.title ?? "", summary: e.summary ?? null, status: e.status ?? null,
    ...(e.status_label ? { status_label: e.status_label } : {}),
```

Logo **antes** de `async function pdf(path: string)`, acrescente o bloco de tipos:

```ts
// ─── Meus exames (módulo 21a, ADR 0036; contrato §9; Divergências E1–E4) ───
// Por pessoa do celular (citizen_id, como o histórico). Só o que o contrato
// traz: nunca motivo de negação, marca de crítico ou valores do resultado —
// campo a mais que o api mande é descartado aqui. A política do laudo é do
// api: `result` diz o que a pessoa pode ver.
export interface CitizenExamResult { available: boolean; pickup_at_unit: boolean; released_on: string | null; pdf_available: boolean }
export interface CitizenExam {
  id: string; label: string; status_label: string; requested_at: string;
  scheduled_at: string | null; scheduled_place: string | null; result: CitizenExamResult | null;
}
export interface CitizenExams { verified: boolean; exams: CitizenExam[]; next_before: string | null }
export const EXAMS_PAGE_SIZE = 20;

type RawExam = Partial<Omit<CitizenExam, "result">> & { id: string; result?: Partial<CitizenExamResult> | null };

function normalizeExam(e: RawExam): CitizenExam {
  const r = e.result;
  return {
    id: e.id, label: e.label ?? "", status_label: e.status_label ?? "", requested_at: e.requested_at ?? "",
    scheduled_at: e.scheduled_at ?? null, scheduled_place: e.scheduled_place ?? null,
    result: r
      ? { available: r.available === true, pickup_at_unit: r.pickup_at_unit === true,
          released_on: r.released_on ?? null, pdf_available: r.pdf_available === true }
      : null
  };
}

```

No objeto `citizenApi`, troque

```ts
  documentPdf: (documentId: string, citizenId: string): Promise<Blob> =>
    pdf(`/documents/${encodeURIComponent(documentId)}/pdf?citizen_id=${encodeURIComponent(citizenId)}`)
};
```

por

```ts
  documentPdf: (documentId: string, citizenId: string): Promise<Blob> =>
    pdf(`/documents/${encodeURIComponent(documentId)}/pdf?citizen_id=${encodeURIComponent(citizenId)}`),
  // Meus exames (módulo 21a): mais novo primeiro; par declarado = verified false.
  exams: async (citizenId: string, opts: { before?: string | null } = {}): Promise<CitizenExams> => {
    const params = new URLSearchParams({ citizen_id: citizenId });
    if (opts.before) params.set("before", opts.before);
    params.set("limit", String(EXAMS_PAGE_SIZE));
    const d = await call<{ verified?: boolean; exams?: RawExam[]; next_before?: string | null }>(
      "GET", `/exams?${params.toString()}`);
    return { verified: d.verified === true, exams: (d.exams ?? []).map(normalizeExam), next_before: d.next_before ?? null };
  },
  // PDF do laudo, só quando a política permite (senão 404 not_found).
  examPdf: (examId: string, citizenId: string): Promise<Blob> =>
    pdf(`/exams/${encodeURIComponent(examId)}/pdf?citizen_id=${encodeURIComponent(citizenId)}`)
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/citizenApi.exams.test.ts src/lib/citizenApi.history.test.ts src/lib/citizenApi.test.ts && npx tsc --noEmit`
Expected: PASS (o teste do módulo 20 segue igual, sem asserção editada) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/citizenApi.ts src/lib/citizenApi.exams.test.ts
$G commit -m "feat: add citizen exams client and exam pdf download

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela de "Meus exames"

**Files:**
- Create: `src/lib/exams.ts`
- Test: `src/lib/exams.test.ts`

**Interfaces:**
- Consumes: `CitizenExam`, `CitizenExamResult` (Task 1); `fmtCalendarDate`, `fmtDate`, `fmtDateTime` (`src/lib/format.ts`).
- Produces:
  ```ts
  export const EXAM_KIND = "exam";
  export const EXAMS_NOT_VERIFIED: string; export const EXAMS_DISABLED: string; export const EXAMS_EMPTY: string;
  export const PICKUP_AT_UNIT: string; export const EXAM_PDF_GONE: string;
  export type ExamResultView = { kind: "none" } | { kind: "pickup"; text: string } | { kind: "available"; text: string; downloadable: boolean };
  export function examResultView(result: CitizenExamResult | null): ExamResultView;
  export function examScheduleText(exam: CitizenExam): string | null;
  export function examRequestedText(exam: CitizenExam): string | null;
  export function mergeExams(current: CitizenExam[], next: CitizenExam[]): CitizenExam[];
  export function examFilename(examId: string): string;
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/exams.test.ts
import { describe, it, expect } from "vitest";
import type { CitizenExam } from "./citizenApi";
import {
  PICKUP_AT_UNIT, examFilename, examRequestedText, examResultView, examScheduleText, mergeExams
} from "./exams";

const exam = (over: Partial<CitizenExam> = {}): CitizenExam => ({
  id: "ex1", label: "Hemograma completo", status_label: "Pedido", requested_at: "2026-10-01T09:00:00-03:00",
  scheduled_at: null, scheduled_place: null, result: null, ...over
});

describe("examResultView: tabela das políticas", () => {
  const cases: [ string, CitizenExam["result"], ReturnType<typeof examResultView> ][] = [
    [ "sem resultado (ou after_acknowledgement antes da ciência)", null, { kind: "none" } ],
    [ "immediate com PDF", { available: true, pickup_at_unit: false, released_on: "2026-10-08", pdf_available: true },
      { kind: "available", text: "Resultado disponível — liberado em 08/10/2026", downloadable: true } ],
    [ "disponível sem PDF", { available: true, pickup_at_unit: false, released_on: "2026-10-08", pdf_available: false },
      { kind: "available", text: "Resultado disponível — liberado em 08/10/2026", downloadable: false } ],
    [ "disponível sem data", { available: true, pickup_at_unit: false, released_on: null, pdf_available: true },
      { kind: "available", text: "Resultado disponível", downloadable: true } ],
    [ "never_citizen", { available: false, pickup_at_unit: true, released_on: "2026-10-08", pdf_available: false },
      { kind: "pickup", text: PICKUP_AT_UNIT } ],
    [ "never_citizen com PDF por engano: retirar vence", { available: true, pickup_at_unit: true, released_on: "2026-10-08", pdf_available: true },
      { kind: "pickup", text: PICKUP_AT_UNIT } ],
    [ "PDF sem available: nada", { available: false, pickup_at_unit: false, released_on: "2026-10-08", pdf_available: true },
      { kind: "none" } ]
  ];
  it.each(cases)("%s", (_name, result, expected) => {
    expect(examResultView(result)).toEqual(expected);
  });
});

describe("examScheduleText e examRequestedText", () => {
  it("marcação com e sem local; sem data, nada", () => {
    expect(examScheduleText(exam())).toBeNull();
    expect(examScheduleText(exam({ scheduled_at: "2026-10-05T07:30:00-03:00" }))).toBe("Marcado para 05/10/2026, 07:30");
    expect(examScheduleText(exam({ scheduled_at: "2026-10-05T07:30:00-03:00", scheduled_place: "Laboratório Simulado — Rua XV, 100" })))
      .toBe("Marcado para 05/10/2026, 07:30 — Laboratório Simulado — Rua XV, 100");
  });

  it("data do pedido; vazia ou inválida, nada", () => {
    expect(examRequestedText(exam())).toBe("Pedido em 01/10/2026");
    expect(examRequestedText(exam({ requested_at: "" }))).toBeNull();
    expect(examRequestedText(exam({ requested_at: "ontem" }))).toBeNull();
  });
});

describe("mergeExams", () => {
  it("junta sem repetir, na ordem", () => {
    const a = exam({ id: "a" }); const b = exam({ id: "b" }); const c = exam({ id: "c" });
    expect(mergeExams([ a, b ], [ b, c ]).map((x) => x.id)).toEqual([ "a", "b", "c" ]);
    expect(mergeExams([], [ a ]).map((x) => x.id)).toEqual([ "a" ]);
  });
});

describe("examFilename: só o id", () => {
  it("letras, números e hífen; nunca nome ou separador de caminho", () => {
    expect(examFilename("0b6f-41a2")).toBe("exame-0b6f-41a2.pdf");
    expect(examFilename("../Hemograma de José/1")).toBe("exame-HemogramadeJos1.pdf");
    expect(examFilename("")).toBe("exame-resultado.pdf");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/exams.test.ts`
Expected: FAIL — `./exams` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/exams.ts
// Regras de tela de "Meus exames" (módulo 21a, F-21.10; spec §6 "Cidadão";
// contrato §9). A política do laudo (immediate, after_acknowledgement,
// never_citizen) é aplicada no api: aqui só se traduz o `result` que ele
// devolveu. A tela nunca deduz o laudo da situação (`status_label`).
import type { CitizenExam, CitizenExamResult } from "./citizenApi";
import { fmtCalendarDate, fmtDate, fmtDateTime } from "./format";

// `kind` da entrada de exame na linha do tempo (contrato §10).
export const EXAM_KIND = "exam";

export const EXAMS_NOT_VERIFIED =
  "Para ver seus exames por aqui, valide seu cadastro na unidade de saúde com um documento com foto.";
export const EXAMS_DISABLED = "Os exames ainda não aparecem por aqui nesta cidade.";
export const EXAMS_EMPTY = "Nenhum exame pedido ainda.";
export const PICKUP_AT_UNIT = "Retire o resultado na sua unidade de saúde.";
export const EXAM_PDF_GONE = "Este resultado não está disponível por aqui. Procure a sua unidade de saúde.";

export type ExamResultView =
  | { kind: "none" }
  | { kind: "pickup"; text: string }
  | { kind: "available"; text: string; downloadable: boolean };

// "Retire na unidade" vence tudo (never_citizen); PDF só com available e
// pdf_available. Sem result, ou sem available, nada sobre o laudo.
export function examResultView(result: CitizenExamResult | null): ExamResultView {
  if (!result) return { kind: "none" };
  if (result.pickup_at_unit) return { kind: "pickup", text: PICKUP_AT_UNIT };
  if (!result.available) return { kind: "none" };
  const day = fmtCalendarDate(result.released_on);
  return {
    kind: "available",
    text: day === "—" ? "Resultado disponível" : `Resultado disponível — liberado em ${day}`,
    downloadable: result.pdf_available
  };
}

export function examScheduleText(exam: CitizenExam): string | null {
  if (!exam.scheduled_at) return null;
  const when = fmtDateTime(exam.scheduled_at);
  if (when === "—") return null;
  return exam.scheduled_place ? `Marcado para ${when} — ${exam.scheduled_place}` : `Marcado para ${when}`;
}

export function examRequestedText(exam: CitizenExam): string | null {
  const day = fmtDate(exam.requested_at);
  return day === "—" ? null : `Pedido em ${day}`;
}

export function mergeExams(current: CitizenExam[], next: CitizenExam[]): CitizenExam[] {
  const seen = new Set(current.map((x) => x.id));
  return [ ...current, ...next.filter((x) => !seen.has(x.id)) ];
}

// Só letras, números e hífen: nunca o nome do exame, CPF ou separador de caminho.
export function examFilename(examId: string): string {
  const raw = examId.replace(/[^A-Za-z0-9-]/g, "");
  return `exame-${raw || "resultado"}.pdf`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/exams.test.ts && npx tsc --noEmit`
Expected: PASS. (`"../Hemograma de José/1"` perde `.`, `/`, espaços e o `é`: `HemogramadeJos1`.)

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/exams.ts src/lib/exams.test.ts
$G commit -m "feat: add citizen exam display rules by release policy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: A tela "Meus exames"

**Files:**
- Create: `src/modules/citizen/MyExamsStep.tsx`
- Test: `src/modules/citizen/MyExamsStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.exams`, `citizenApi.examPdf`, `ApiError`, `CitizenExam`, `CitizenExams` (Task 1); tudo de `src/lib/exams.ts` (Task 2); `BigButton`, `ErrorText`, `Screen`, `messageFor`, `pdfMessageFor`, `PDF_RATE_LIMITED` (`ui.tsx`).
- Produces: `export function MyExamsStep(props: { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void }): JSX.Element` — título "Meus exames"; cada exame é um `listitem` com `aria-label` = `label`; botão "Baixar resultado de <label>".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyExamsStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyExamsStep } from "./MyExamsStep";
import { citizenApi, ApiError, type CitizenExam, type CitizenExams } from "../../lib/citizenApi";
import { EXAMS_DISABLED, EXAMS_EMPTY, EXAMS_NOT_VERIFIED, EXAM_PDF_GONE, PICKUP_AT_UNIT } from "../../lib/exams";
import { PDF_RATE_LIMITED } from "./ui";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-13T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); vi.unstubAllGlobals(); });

const exam = (over: Partial<CitizenExam>): CitizenExam => ({
  id: "x", label: "", status_label: "", requested_at: "2026-10-01T09:00:00-03:00",
  scheduled_at: null, scheduled_place: null, result: null, ...over
});
const IMMEDIATE = exam({ id: "ex-imm", label: "Hemograma completo", status_label: "Resultado disponível",
  scheduled_at: "2026-10-05T07:30:00-03:00", scheduled_place: "Laboratório Simulado — Rua XV, 100",
  result: { available: true, pickup_at_unit: false, released_on: "2026-10-08", pdf_available: true } });
const AFTER_ACK = exam({ id: "ex-ack", label: "Hemoglobina glicada", status_label: "Coletado",
  scheduled_at: "2026-10-05T07:30:00-03:00", scheduled_place: "Laboratório Simulado — Rua XV, 100" });
const NEVER = exam({ id: "ex-unit", label: "Sorologia para HIV", status_label: "Resultado pronto",
  result: { available: false, pickup_at_unit: true, released_on: "2026-10-08", pdf_available: false } });
const DENIED = exam({ id: "ex-den", label: "Ressonância magnética", status_label: "Não autorizado — procure a sua unidade" });

const page = (over: Partial<CitizenExams> = {}): CitizenExams =>
  ({ verified: true, exams: [ IMMEDIATE, AFTER_ACK, NEVER, DENIED ], next_before: null, ...over });

const renderStep = (onValidate = vi.fn(), onBack = vi.fn()) =>
  render(<MyExamsStep citizenId="p1" onBack={onBack} onValidate={onValidate} />);

function fakeBlobUrl(href = "blob:http://curitiba.localhost/abc") {
  const createObjectURL = vi.fn(() => href);
  const revokeObjectURL = vi.fn();
  // O jsdom não tem createObjectURL/revokeObjectURL: um URL dublê no global, desfeito pelo unstubAllGlobals.
  class FakeURL extends URL {}
  Object.assign(FakeURL, { createObjectURL, revokeObjectURL });
  vi.stubGlobal("URL", FakeURL);
  return { createObjectURL, revokeObjectURL };
}

describe("MyExamsStep", () => {
  it("par validado: situação, marcação e laudo pelas três políticas", async () => {
    const spy = vi.spyOn(citizenApi, "exams").mockResolvedValue(page());
    renderStep();

    const imm = await screen.findByRole("listitem", { name: "Hemograma completo" });
    expect(spy).toHaveBeenCalledWith("p1");
    expect(within(imm).getByText("Pedido em 01/10/2026")).toBeInTheDocument();
    expect(within(imm).getByText("Marcado para 05/10/2026, 07:30 — Laboratório Simulado — Rua XV, 100")).toBeInTheDocument();
    expect(within(imm).getByText("Resultado disponível — liberado em 08/10/2026")).toBeInTheDocument();
    expect(within(imm).getByRole("button", { name: "Baixar resultado de Hemograma completo" })).toBeInTheDocument();

    const ack = screen.getByRole("listitem", { name: "Hemoglobina glicada" });
    expect(within(ack).getByText("Coletado")).toBeInTheDocument();
    expect(within(ack).queryByRole("button")).not.toBeInTheDocument();
    expect(within(ack).queryByText(/Resultado disponível/)).not.toBeInTheDocument();

    const never = screen.getByRole("listitem", { name: "Sorologia para HIV" });
    expect(within(never).getByText(PICKUP_AT_UNIT)).toBeInTheDocument();
    expect(within(never).queryByRole("button")).not.toBeInTheDocument();

    const denied = screen.getByRole("listitem", { name: "Ressonância magnética" });
    expect(within(denied).getByText("Não autorizado — procure a sua unidade")).toBeInTheDocument();
    expect(screen.queryByRole("note", { name: "cadastro não validado" })).not.toBeInTheDocument();
  });

  it("nada de crítico, motivo ou valor: só os campos do contrato aparecem", async () => {
    const leaky = { ...IMMEDIATE, critical: true, denial_reason: "fora do protocolo municipal",
      values: [ { label: "Hemoglobina", value: "6,1" } ], requester_name: "Dr. Fulano" };
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page({ exams: [ leaky as unknown as CitizenExam ] }));
    renderStep();
    await screen.findByRole("listitem", { name: "Hemograma completo" });
    expect(document.body.textContent).not.toMatch(/crític|fora do protocolo|6,1|Fulano/i);
  });

  it("par declarado: convite e nenhum exame, mesmo se o api mandar", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page({ verified: false }));
    const onValidate = vi.fn();
    renderStep(onValidate);
    expect((await screen.findByRole("note", { name: "cadastro não validado" })).textContent).toBe(EXAMS_NOT_VERIFIED);
    expect(screen.queryByRole("listitem")).not.toBeInTheDocument();
    expect(screen.queryByText("Hemograma completo")).not.toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("403 not_verified: o mesmo convite, sem erro", async () => {
    vi.spyOn(citizenApi, "exams").mockRejectedValue(new ApiError(403, "not_verified"));
    renderStep();
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toBeInTheDocument();
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });

  it("sem exames: a frase de vazio", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page({ exams: [] }));
    renderStep();
    expect(await screen.findByText(EXAMS_EMPTY)).toBeInTheDocument();
  });

  it("campos ausentes: nada de undefined ou null", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page({ exams: [
      exam({ id: "e1", label: "Glicemia de jejum", status_label: "", requested_at: "",
        result: { available: true, pickup_at_unit: false, released_on: null, pdf_available: false } })
    ] }));
    renderStep();
    const item = await screen.findByRole("listitem", { name: "Glicemia de jejum" });
    expect(within(item).getByText("Resultado disponível")).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
    expect(within(item).queryByText(/Pedido em/)).not.toBeInTheDocument();
  });

  it("interruptor desligado (403 feature_disabled): a frase dos exames", async () => {
    vi.spyOn(citizenApi, "exams").mockRejectedValue(new ApiError(403, "feature_disabled"));
    renderStep();
    expect((await screen.findByRole("alert")).textContent).toBe(EXAMS_DISABLED);
  });

  it("baixar o resultado: arquivo exame-<id>.pdf e o endereço revogado depois", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page());
    const blob = new Blob([ "%PDF-1.7" ], { type: "application/pdf" });
    const pdf = vi.spyOn(citizenApi, "examPdf").mockResolvedValue(blob);
    const { createObjectURL, revokeObjectURL } = fakeBlobUrl();
    // O mock só guarda o que viu: um expect aqui dentro seria engolido pelo try do download.
    const seen: { download: string; href: string }[] = [];
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) {
      seen.push({ download: this.download, href: this.href });
    });
    renderStep();
    const imm = await screen.findByRole("listitem", { name: "Hemograma completo" });
    vi.useRealTimers();
    vi.useFakeTimers({ toFake: [ "Date", "setTimeout", "clearTimeout" ], shouldAdvanceTime: true });
    vi.setSystemTime(new Date("2026-10-13T10:00:00-03:00"));
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    await user.click(within(imm).getByRole("button", { name: "Baixar resultado de Hemograma completo" }));
    await vi.waitFor(() => expect(click).toHaveBeenCalledTimes(1));
    expect(pdf).toHaveBeenCalledWith("ex-imm", "p1");
    expect(createObjectURL).toHaveBeenCalledWith(blob);
    expect(seen).toEqual([ { download: "exame-ex-imm.pdf", href: "blob:http://curitiba.localhost/abc" } ]);
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
    expect(revokeObjectURL).not.toHaveBeenCalled();
    vi.advanceTimersByTime(60_000);
    expect(revokeObjectURL).toHaveBeenCalledWith("blob:http://curitiba.localhost/abc");
  });

  it("resultado que deixou de estar disponível (404): diz no cartão e não baixa", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page());
    vi.spyOn(citizenApi, "examPdf").mockRejectedValue(new ApiError(404, "not_found"));
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click");
    renderStep();
    const imm = await screen.findByRole("listitem", { name: "Hemograma completo" });
    await userEvent.click(within(imm).getByRole("button", { name: "Baixar resultado de Hemograma completo" }));
    expect((await within(imm).findByRole("alert")).textContent).toBe(EXAM_PDF_GONE);
    expect(click).not.toHaveBeenCalled();
  });

  it("limite de downloads (429): a frase do download, não a do SMS", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page());
    vi.spyOn(citizenApi, "examPdf").mockRejectedValue(new ApiError(429, "too_many_requests"));
    renderStep();
    const imm = await screen.findByRole("listitem", { name: "Hemograma completo" });
    await userEvent.click(within(imm).getByRole("button", { name: "Baixar resultado de Hemograma completo" }));
    expect((await within(imm).findByRole("alert")).textContent).toBe(PDF_RATE_LIMITED);
  });

  it("sair da tela durante o download: não cria o endereço nem clica", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page());
    let resolve: (b: Blob) => void = () => {};
    vi.spyOn(citizenApi, "examPdf").mockImplementation(() => new Promise<Blob>((r) => { resolve = r; }));
    const { createObjectURL } = fakeBlobUrl("blob:http://curitiba.localhost/late");
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(() => {});
    const { unmount } = renderStep();
    const imm = await screen.findByRole("listitem", { name: "Hemograma completo" });
    await userEvent.click(within(imm).getByRole("button", { name: "Baixar resultado de Hemograma completo" }));
    unmount();
    resolve(new Blob([ "%PDF" ], { type: "application/pdf" }));
    await Promise.resolve();
    expect(createObjectURL).not.toHaveBeenCalled();
    expect(click).not.toHaveBeenCalled();
  });

  it("carregar mais: cursor como veio, sem repetir; a falha aparece junto do botão", async () => {
    const spy = vi.spyOn(citizenApi, "exams")
      .mockResolvedValueOnce(page({ exams: [ IMMEDIATE, AFTER_ACK ], next_before: "2026-10-01T09:00:00-03:00|ex-ack" }))
      .mockRejectedValueOnce(new ApiError(500, "http_500"))
      .mockResolvedValueOnce(page({ exams: [ AFTER_ACK, NEVER ], next_before: null }));
    renderStep();
    await screen.findByRole("listitem", { name: "Hemoglobina glicada" });

    await userEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
    expect(await screen.findByRole("alert")).toBeInTheDocument();
    expect(spy).toHaveBeenLastCalledWith("p1", { before: "2026-10-01T09:00:00-03:00|ex-ack" });

    await userEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
    expect(await screen.findByRole("listitem", { name: "Sorologia para HIV" })).toBeInTheDocument();
    expect(screen.getAllByRole("listitem").map((li) => li.getAttribute("aria-label")))
      .toEqual([ "Hemograma completo", "Hemoglobina glicada", "Sorologia para HIV" ]);
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Carregar mais" })).not.toBeInTheDocument();
  });

  it("Voltar", async () => {
    vi.spyOn(citizenApi, "exams").mockResolvedValue(page());
    const onBack = vi.fn();
    renderStep(vi.fn(), onBack);
    await screen.findByRole("listitem", { name: "Hemograma completo" });
    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(onBack).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/MyExamsStep.test.tsx`
Expected: FAIL — `./MyExamsStep` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/citizen/MyExamsStep.tsx
// "Meus exames" de uma pessoa do celular (módulo 21a, F-21.10; spec §6
// "Cidadão"; contrato §9). Só par validado: o par declarado vê o convite para
// validar e nenhum exame. A política do laudo é do api; a tela mostra a
// situação, a marcação e o que `result` oferece. Nunca motivo de negação,
// marca de crítico ou valores: só os campos do contrato são desenhados.
import { useEffect, useRef, useState, type CSSProperties } from "react";
import { citizenApi, ApiError, type CitizenExam, type CitizenExams } from "../../lib/citizenApi";
import {
  EXAMS_DISABLED, EXAMS_EMPTY, EXAMS_NOT_VERIFIED, EXAM_PDF_GONE, examFilename, examRequestedText, examResultView,
  examScheduleText, mergeExams
} from "../../lib/exams";
import { BigButton, ErrorText, Screen, messageFor, pdfMessageFor } from "./ui";

const PDF_URL_TTL_MS = 60_000;
const NOT_VERIFIED: CitizenExams = { verified: false, exams: [], next_before: null };

function listMessage(e: unknown): string {
  return e instanceof ApiError && e.code === "feature_disabled" ? EXAMS_DISABLED : messageFor(e);
}

function pdfMessage(e: unknown): string {
  if (e instanceof ApiError && e.code === "not_found") return EXAM_PDF_GONE;
  if (e instanceof ApiError && e.code === "feature_disabled") return EXAMS_DISABLED;
  // 429 do PDF tem frase própria (pdfMessageFor), não a do SMS.
  return pdfMessageFor(e);
}

export function MyExamsStep({ citizenId, onBack, onValidate }:
  { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void }) {
  const [data, setData] = useState<CitizenExams | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [more, setMore] = useState(false);
  const [moreError, setMoreError] = useState<string | null>(null);
  const [downloading, setDownloading] = useState<string | null>(null);
  const [downloadError, setDownloadError] = useState<{ examId: string; message: string } | null>(null);
  // A tela pode sair enquanto o PDF baixa: sem tela, nada de endereço blob nem clique.
  const mounted = useRef(true);
  const pendingUrls = useRef(new Map<string, ReturnType<typeof setTimeout>>());

  useEffect(() => {
    mounted.current = true;
    const pending = pendingUrls.current;
    return () => {
      mounted.current = false;
      pending.forEach((timer, url) => { clearTimeout(timer); URL.revokeObjectURL(url); });
      pending.clear();
    };
  }, []);

  useEffect(() => {
    // StrictMode monta duas vezes em dev: a resposta da montagem desfeita não escreve.
    let alive = true;
    citizenApi.exams(citizenId)
      .then((d) => { if (alive) setData(d); })
      .catch((e) => {
        if (!alive) return;
        // Api que recusa o par declarado com 403 em vez de verified:false: o mesmo convite.
        if (e instanceof ApiError && e.code === "not_verified") setData(NOT_VERIFIED);
        else setError(listMessage(e));
      });
    return () => { alive = false; };
  }, [citizenId]);

  async function loadMore() {
    if (!data?.next_before || more) return;
    setMore(true); setMoreError(null);
    try {
      const next = await citizenApi.exams(citizenId, { before: data.next_before });
      if (!mounted.current) return;
      setData({ ...data, exams: mergeExams(data.exams, next.exams), next_before: next.next_before });
    } catch (e) {
      if (mounted.current) setMoreError(listMessage(e));
    } finally {
      if (mounted.current) setMore(false);
    }
  }

  async function download(exam: CitizenExam) {
    if (downloading) return;
    setDownloading(exam.id); setDownloadError(null);
    try {
      const blob = await citizenApi.examPdf(exam.id, citizenId);
      if (!mounted.current) return;
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = examFilename(exam.id);
      a.click();
      const pending = pendingUrls.current;
      pending.set(url, setTimeout(() => { pending.delete(url); URL.revokeObjectURL(url); }, PDF_URL_TTL_MS));
    } catch (e) {
      if (mounted.current) setDownloadError({ examId: exam.id, message: pdfMessage(e) });
    } finally {
      if (mounted.current) setDownloading(null);
    }
  }

  return (
    <Screen title="Meus exames" footer={<BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>}>
      {error && <ErrorText>{error}</ErrorText>}
      {!data && !error && <p role="status">Carregando…</p>}
      {data && !data.verified && (
        <div style={{ display: "grid", gap: 12 }}>
          <p role="note" aria-label="cadastro não validado" style={warnNote}>{EXAMS_NOT_VERIFIED}</p>
          <BigButton onClick={() => onValidate(citizenId)}>Validar no posto</BigButton>
        </div>
      )}
      {data && data.verified && (
        <section aria-label="Exames" style={{ display: "grid", gap: 12 }}>
          {data.exams.length === 0 && <p>{EXAMS_EMPTY}</p>}
          <ul style={{ listStyle: "none", padding: 0, margin: 0, display: "grid", gap: 12 }}>
            {data.exams.map((x) => (
              <ExamCard key={x.id} exam={x} downloading={downloading}
                error={downloadError?.examId === x.id ? downloadError.message : null}
                onDownload={() => void download(x)} />
            ))}
          </ul>
          {data.next_before && (
            <div style={{ display: "grid", gap: 8 }}>
              {moreError && <ErrorText>{moreError}</ErrorText>}
              <BigButton variant="secondary" disabled={more} onClick={() => void loadMore()}>Carregar mais</BigButton>
            </div>
          )}
        </section>
      )}
    </Screen>
  );
}

function ExamCard({ exam, downloading, error, onDownload }:
  { exam: CitizenExam; downloading: string | null; error: string | null; onDownload: () => void }) {
  const requested = examRequestedText(exam);
  const schedule = examScheduleText(exam);
  const result = examResultView(exam.result);
  const busy = downloading === exam.id;
  return (
    <li aria-label={exam.label} style={card}>
      <strong>{exam.label}</strong>
      {exam.status_label && <div>{exam.status_label}</div>}
      {requested && <div style={muted}>{requested}</div>}
      {schedule && <div>{schedule}</div>}
      {result.kind === "pickup" && <p style={infoNote}>{result.text}</p>}
      {result.kind === "available" && (
        <div style={{ display: "grid", gap: 8, marginTop: 8 }}>
          <div>{result.text}</div>
          {result.downloadable && (
            <BigButton variant="secondary" disabled={downloading !== null} aria-busy={busy}
              aria-label={busy ? `Baixando resultado de ${exam.label}` : `Baixar resultado de ${exam.label}`}
              onClick={onDownload}>
              {busy ? "Baixando…" : "Baixar resultado (PDF)"}
            </BigButton>
          )}
        </div>
      )}
      {error && <ErrorText>{error}</ErrorText>}
    </li>
  );
}

const warnNote: CSSProperties = { background: "var(--warn-bg)", padding: 12, borderRadius: 12, margin: 0 };
const infoNote: CSSProperties = { background: "var(--accent-bg)", padding: 12, borderRadius: 12, margin: "8px 0 0" };
const card: CSSProperties = { border: "1px solid var(--rule2)", borderRadius: 12, padding: 12, display: "grid", gap: 4 };
const muted: CSSProperties = { color: "var(--ink2)" };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen/MyExamsStep.test.tsx src/theme/guards.test.ts && npx tsc --noEmit`
Expected: PASS (a guarda de cor segue verde com a tela nova).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/MyExamsStep.tsx src/modules/citizen/MyExamsStep.test.tsx
$G commit -m "feat: add my exams screen with release policy and pdf download

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: O botão em "Minhas triagens" e o estado no `Flow`

**Files:**
- Modify: `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `README.md`
- Test: `src/modules/citizen/Flow.exams.test.tsx`

**Interfaces:**
- Consumes: `MyExamsStep` (Task 3); `hasSessionFeature`, `setSessionFeatures` (`src/lib/sessionFeatures.ts`).
- Produces: `HistoryStep` aceita `onExams?: () => void` (botão "Meus exames"); estado `{ at: "exams"; consentVersion: string | null; citizenId: string }` no `Flow`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/Flow.exams.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi } from "../../lib/citizenApi";
import { setSessionFeatures } from "../../lib/sessionFeatures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-13T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); setSessionFeatures(null); window.history.replaceState(null, "", "/"); });

function signedIn(features?: string[]) {
  // O spy não passa pelo withCityTimeZone do cliente: instala os interruptores como ele faria.
  vi.spyOn(citizenApi, "currentSession").mockImplementation(async () => {
    setSessionFeatures(features);
    return { phone_masked: "(**) *****-5432", ...(features ? { features } : {}) };
  });
  vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({
    people: [ { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null, profile: null } ]
  });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
  vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
  vi.spyOn(citizenApi, "appointments").mockResolvedValue({ appointments: [] });
  vi.spyOn(citizenApi, "triages").mockResolvedValue({
    citizen: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" }, triages: []
  });
  vi.spyOn(citizenApi, "exams").mockResolvedValue({ verified: false, exams: [], next_before: null });
  vi.spyOn(citizenApi, "issueVerificationCode").mockResolvedValue({ code: "482913", expires_at: "2026-10-13T10:10:00-03:00" });
}

async function toMyTriages() {
  render(<Flow />);
  await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
  await userEvent.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
  expect(await screen.findByRole("heading", { name: "Minhas triagens" })).toBeInTheDocument();
}

describe("Flow — Meus exames (módulo 21a)", () => {
  it("com exam_tracking: Minhas triagens → Meus exames → Voltar → Validar no posto", async () => {
    signedIn([ "exam_tracking" ]);
    await toMyTriages();
    expect(screen.queryByRole("button", { name: "Meu histórico" })).not.toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Meus exames" }));
    expect(await screen.findByRole("heading", { name: "Meus exames" })).toBeInTheDocument();
    expect(citizenApi.exams).toHaveBeenCalledWith("p1");
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { name: "Minhas triagens" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Meus exames" }));
    await userEvent.click(await screen.findByRole("button", { name: "Validar no posto" }));
    expect(await screen.findByText("482913")).toBeInTheDocument();
    expect(citizenApi.issueVerificationCode).toHaveBeenCalledWith("p1");
  });

  it("com os dois interruptores: os dois botões", async () => {
    signedIn([ "patient_timeline", "exam_tracking" ]);
    await toMyTriages();
    expect(screen.getByRole("button", { name: "Meu histórico" })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Meus exames" })).toBeInTheDocument();
  });

  it("sem exam_tracking na sessão: Minhas triagens como hoje", async () => {
    signedIn([ "patient_timeline" ]);
    await toMyTriages();
    expect(screen.queryByRole("button", { name: "Meus exames" })).not.toBeInTheDocument();
    expect(citizenApi.exams).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/Flow.exams.test.tsx`
Expected: FAIL — o botão "Meus exames" não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/citizen/HistoryStep.tsx`, troque

```tsx
    // Módulo 20: "Meu histórico", só com patient_timeline ligado (quem decide é o Flow).
    onRecord?: () => void }) {
```

por

```tsx
    // Módulo 20: "Meu histórico", só com patient_timeline ligado (quem decide é o Flow).
    onRecord?: () => void;
    // Módulo 21a: "Meus exames", só com exam_tracking ligado (quem decide é o Flow).
    onExams?: () => void }) {
```

e a assinatura `export function HistoryStep({ citizenId, onBack, onValidate, onCheckIn, onAppointmentCheckIn, onRecord }:` por

```tsx
export function HistoryStep({ citizenId, onBack, onValidate, onCheckIn, onAppointmentCheckIn, onRecord, onExams }:
```

e, logo depois da linha `{onRecord && <BigButton variant="secondary" onClick={onRecord} style={{ marginBottom: 16 }}>Meu histórico</BigButton>}`, acrescente

```tsx
          {onExams && <BigButton variant="secondary" onClick={onExams} style={{ marginBottom: 16 }}>Meus exames</BigButton>}
```

Em `src/modules/citizen/Flow.tsx`:

1. Depois de `import { MyHistoryStep } from "./MyHistoryStep";`, acrescente `import { MyExamsStep } from "./MyExamsStep";`.
2. Depois de `  | { at: "record"; consentVersion: string | null; citizenId: string }`, acrescente `  | { at: "exams"; consentVersion: string | null; citizenId: string }`.
3. No `case "history":`, troque o final

```tsx
        onRecord={hasSessionFeature("patient_timeline")
          ? () => setState({ at: "record", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined} />;
      break;
```

por

```tsx
        onRecord={hasSessionFeature("patient_timeline")
          ? () => setState({ at: "record", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined}
        onExams={hasSessionFeature("exam_tracking")
          ? () => setState({ at: "exams", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined} />;
      break;
```

4. Logo antes de `    case "verify-code":`, acrescente

```tsx
    case "exams":
      view = <MyExamsStep key={state.citizenId} citizenId={state.citizenId}
        onBack={() => setState({ at: "history", citizenId: state.citizenId, consentVersion: state.consentVersion })}
        onValidate={id => setState({ at: "verify-code", citizenId: id, consentVersion: state.consentVersion })} />;
      break;
```

Em `README.md`, logo depois do parágrafo **Meu histórico (módulo 20).** (o que termina em "`documento-<código>.pdf`."), acrescente:

```markdown
**Meus exames (módulo 21a).** Com `exam_tracking` ligado na cidade (vem em
`features` da sessão), "Minhas triagens" ganha o botão **Meus exames**. Par
validado: cada exame com a situação, a data e o local marcados e o laudo pela
política do exame — `immediate` mostra ao registrar, `after_acknowledgement` só
depois da ciência do profissional, `never_citizen` diz "Retire o resultado na
sua unidade de saúde." — com o **Baixar resultado (PDF)** (`exame-<id>.pdf`)
quando o api oferece. Par só declarado: o convite para validar no posto. Quem
decide o laudo é o api (`result`); a tela nunca o deduz da situação e nunca
mostra motivo de negação, a marca de crítico ou valores. Em "Meu histórico",
a entrada **Exame** baixa o PDF pela mesma rota do exame.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen && npx tsc --noEmit`
Expected: PASS — inclusive `Flow.history.test.tsx` e os testes antigos do `HistoryStep`, sem asserção editada.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx src/modules/citizen/Flow.exams.test.tsx README.md
$G commit -m "feat: open my exams from my triages when exam tracking is on

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: A entrada `exam` em "Meu histórico"

**Files:**
- Modify: `src/lib/history.ts`, `src/modules/citizen/MyHistoryStep.tsx`
- Test: `src/modules/citizen/MyHistoryStep.exams.test.tsx`

**Interfaces:**
- Consumes: `CitizenEntry.status_label`, `citizenApi.examPdf` (Task 1); `EXAM_KIND`, `EXAM_PDF_GONE`, `examFilename` (Task 2); `citizenStatusLabel`, `documentFilename` (existentes).
- Produces:
  ```ts
  // src/lib/history.ts
  export function entryStatusText(entry: CitizenEntry): string | null; // status_label vence citizenStatusLabel(status)
  export function entryFilename(entry: CitizenEntry): string;          // exam → exame-<id>.pdf; senão documentFilename
  // KIND_LABEL ganha exam: "Exame"
  ```

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyHistoryStep.exams.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyHistoryStep } from "./MyHistoryStep";
import { citizenApi, ApiError, type CitizenEntry, type CitizenHistory } from "../../lib/citizenApi";
import { EXAM_PDF_GONE } from "../../lib/exams";
import { DOCUMENT_GONE } from "../../lib/history";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-13T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); vi.unstubAllGlobals(); });

const EXAM: CitizenEntry = {
  id: "exam:ex1", kind: "exam", occurred_at: "2026-10-08T10:00:00-03:00", unit: { name: "UBS Centro" },
  professional_name: null, title: "Hemograma completo", summary: null, status: "scheduled",
  status_label: "Resultado disponível",
  document: { id: "ex1", short_code: null, verification_url: null, downloadable: true }
};
const EXAM_NO_PDF: CitizenEntry = { ...EXAM, id: "exam:ex2", title: "Hemoglobina glicada", status_label: "Coletado", document: null };
const DOC: CitizenEntry = {
  id: "clinical_document:doc2", kind: "clinical_document", occurred_at: "2026-09-10T14:40:00-03:00", unit: null,
  professional_name: null, title: "Receita", summary: null, status: "issued",
  document: { id: "doc2", short_code: "P3H8T-2LQ6R", verification_url: null, downloadable: true }
};

function api(entries: CitizenEntry[]) {
  const timeline: CitizenHistory = { verified: true, partial: false, entries, next_before: null };
  vi.spyOn(citizenApi, "history").mockImplementation(async (_id, o = {}) =>
    (o.kinds?.length ? { ...timeline, entries: [ DOC ] } : timeline));
  vi.spyOn(citizenApi, "healthSummary").mockRejectedValue(new ApiError(404, "not_available"));
}

const renderStep = () => render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);

describe("MyHistoryStep — exames (módulo 21a)", () => {
  it("exame na linha do tempo: rótulo, situação e PDF pela rota do exame", async () => {
    api([ EXAM, EXAM_NO_PDF, DOC ]);
    const examPdf = vi.spyOn(citizenApi, "examPdf").mockResolvedValue(new Blob([ "%PDF" ], { type: "application/pdf" }));
    const documentPdf = vi.spyOn(citizenApi, "documentPdf");
    class FakeURL extends URL {}
    Object.assign(FakeURL, { createObjectURL: vi.fn(() => "blob:http://curitiba.localhost/ex"), revokeObjectURL: vi.fn() });
    vi.stubGlobal("URL", FakeURL);
    const seen: string[] = [];
    vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) { seen.push(this.download); });
    renderStep();

    const timeline = await screen.findByRole("region", { name: "Linha do tempo" });
    expect(within(timeline).getAllByText("Exame")).toHaveLength(2);
    expect(within(timeline).getByText("Resultado disponível")).toBeInTheDocument();
    expect(within(timeline).getByText("Coletado")).toBeInTheDocument();
    // status_label vence o código: "scheduled" não vira "marcado".
    expect(within(timeline).queryByText("marcado")).not.toBeInTheDocument();
    // Só o documento tem código de conferência; o exame não.
    expect(within(timeline).getAllByText(/Código de conferência/)).toHaveLength(1);

    await userEvent.click(within(timeline).getByRole("button", { name: "Baixar PDF de Hemograma completo" }));
    await vi.waitFor(() => expect(seen).toEqual([ "exame-ex1.pdf" ]));
    expect(examPdf).toHaveBeenCalledWith("ex1", "p1");
    expect(documentPdf).not.toHaveBeenCalled();
    expect(within(timeline).queryByRole("button", { name: "Baixar PDF de Hemoglobina glicada" })).not.toBeInTheDocument();
  });

  it("exame que sumiu na linha do tempo: a frase do exame, não a do documento", async () => {
    api([ EXAM ]);
    vi.spyOn(citizenApi, "examPdf").mockRejectedValue(new ApiError(404, "not_found"));
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click");
    renderStep();
    const timeline = await screen.findByRole("region", { name: "Linha do tempo" });
    await userEvent.click(within(timeline).getByRole("button", { name: "Baixar PDF de Hemograma completo" }));
    expect((await within(timeline).findByRole("alert")).textContent).toBe(EXAM_PDF_GONE);
    expect(within(timeline).queryByText(DOCUMENT_GONE)).not.toBeInTheDocument();
    expect(click).not.toHaveBeenCalled();
  });

  it("documento segue pela rota de documentos", async () => {
    api([ DOC ]);
    const documentPdf = vi.spyOn(citizenApi, "documentPdf").mockRejectedValue(new ApiError(404, "not_found"));
    const examPdf = vi.spyOn(citizenApi, "examPdf");
    renderStep();
    const timeline = await screen.findByRole("region", { name: "Linha do tempo" });
    await userEvent.click(within(timeline).getByRole("button", { name: "Baixar PDF de Receita" }));
    expect((await within(timeline).findByRole("alert")).textContent).toBe(DOCUMENT_GONE);
    expect(documentPdf).toHaveBeenCalledWith("doc2", "p1");
    expect(examPdf).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/MyHistoryStep.exams.test.tsx`
Expected: FAIL — "Exame" aparece como "Registro" e o toque chama `documentPdf`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/history.ts`:

1. Troque `import { fmtCalendarDate, fmtDate } from "./format";` por

```ts
import { fmtCalendarDate, fmtDate } from "./format";
import { EXAM_KIND, examFilename } from "./exams";
```

2. Em `KIND_LABEL`, depois de `  campaign: "Campanha"`, troque por

```ts
  campaign: "Campanha",
  // Módulo 21a (contrato §10).
  exam: "Exame"
```

3. No fim do arquivo, acrescente

```ts
// Situação da entrada: o rótulo pronto do api (exame, módulo 21a) vence a
// tradução do código da origem.
export function entryStatusText(entry: CitizenEntry): string | null {
  return entry.status_label || citizenStatusLabel(entry.status);
}

// O PDF do exame tem nome próprio (o id é o do exame, sem código de conferência).
export function entryFilename(entry: CitizenEntry): string {
  if (entry.kind === EXAM_KIND && entry.document) return examFilename(entry.document.id);
  return entry.document ? documentFilename(entry.document) : "documento-pdf.pdf";
}
```

Em `src/modules/citizen/MyHistoryStep.tsx`:

1. Troque o bloco de import de `../../lib/citizenApi` por

```tsx
import {
  citizenApi, ApiError, DOCUMENT_KINDS, type CitizenEntry, type CitizenHistory, type HealthSummary
} from "../../lib/citizenApi";
```

(sai `type CitizenDocumentRef`, que deixa de ser usado; o `tsc` tem `noUnusedLocals`.)

2. Troque o import de `../../lib/history` por

```tsx
import {
  DOCUMENT_GONE, NOT_VERIFIED_INVITE, PARTIAL_NOTICE, SUMMARY_EMPTY, citizenKindLabel, entryFilename, entryPlace,
  entryStatusText, mergeEntries, sinceText, vitalsLines
} from "../../lib/history";
import { EXAM_KIND, EXAM_PDF_GONE } from "../../lib/exams";
```

3. Troque a função `download` inteira por

```tsx
  async function download(entry: CitizenEntry, list: ListKey) {
    const doc = entry.document;
    if (!doc || downloading) return;
    const isExam = entry.kind === EXAM_KIND;
    setDownloading(doc.id); setDownloadError(null);
    try {
      // Exame (módulo 21a): a rota do exame aplica a política do laudo.
      const blob = isExam ? await citizenApi.examPdf(doc.id, citizenId) : await citizenApi.documentPdf(doc.id, citizenId);
      if (!mounted.current) return;
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = entryFilename(entry);
      a.click();
      const pending = pendingUrls.current;
      pending.set(url, setTimeout(() => { pending.delete(url); URL.revokeObjectURL(url); }, PDF_URL_TTL_MS));
    } catch (e) {
      if (!mounted.current) return;
      // 429 do PDF tem frase própria (pdfMessageFor), não a do SMS.
      const gone = isExam ? EXAM_PDF_GONE : DOCUMENT_GONE;
      setDownloadError({ list, docId: doc.id,
        message: e instanceof ApiError && e.code === "not_found" ? gone : pdfMessageFor(e) });
    } finally {
      if (mounted.current) setDownloading(null);
    }
  }
```

4. Nas duas chamadas de `EntryList`, troque `onDownload={(doc) => void download(doc, "documents")}` por `onDownload={(entry) => void download(entry, "documents")}` e `onDownload={(doc) => void download(doc, "timeline")}` por `onDownload={(entry) => void download(entry, "timeline")}`.
5. Em `EntryList`, troque o tipo `onDownload: (doc: CitizenDocumentRef) => void }) {` por `onDownload: (entry: CitizenEntry) => void }) {`, a linha `const status = citizenStatusLabel(e.status);` por `const status = entryStatusText(e);` e `onClick={() => onDownload(e.document as CitizenDocumentRef)}>` por `onClick={() => onDownload(e)}>`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen/MyHistoryStep.exams.test.tsx src/modules/citizen/MyHistoryStep.test.tsx src/lib/history.test.ts && npx tsc --noEmit`
Expected: PASS — os testes do módulo 20 seguem iguais, sem asserção editada.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/history.ts src/modules/citizen/MyHistoryStep.tsx src/modules/citizen/MyHistoryStep.exams.test.tsx
$G commit -m "feat: show exam entries in my history and download their pdf from the exam route

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: "Meus exames" no Playwright (perfis de celular)

**Files:**
- Create: `e2e/exams.spec.ts`

**Interfaces:**
- Consumes: as Tasks 1–5; `playwright.config.ts` (5 perfis, `WPDA_URL`); o Vite do worktree na **5193**.
- Produces: e2e que abre "Meus exames" pela navegação real, nos perfis `iphone` e `pixel`, com as rotas `/citizen/*` respondidas pelo teste.

- [ ] **Step 1: Suba o Vite do worktree num container da rede do compose**

O `node_modules` do worktree é de macOS: o container usa um volume próprio.

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod21 --network rota-saude_default -p 5193:5193 \
  -v $R/.worktrees/wpda-mod21:/app -v wpda-mod21-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3041 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5193 --host 0.0.0.0"
docker logs -f wpda-mod21   # espere o "ready"; Ctrl+C sai do log, o container segue
```

Expected: `http://curitiba.localhost:5193/wpda/` responde. O container fica no ar para a Task 7.

- [ ] **Step 2: Write the failing test**

```ts
// e2e/exams.spec.ts
// "Meus exames" (módulo 21a) nos perfis de celular. As rotas /citizen/* são
// respondidas aqui (sem api): o que se prova é a tela real no navegador do
// aparelho — as três políticas, o download e a largura sem rolagem lateral.
import { expect, test, type Page, type Route } from "@playwright/test";

const PHONES = new Set([ "iphone", "pixel" ]);
const PERSON = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified", neighborhood: null, profile: null };

const EXAMS = { verified: true, next_before: null, exams: [
  { id: "ex-imm", label: "Hemograma completo", status_label: "Resultado disponível",
    requested_at: "2026-10-01T09:00:00-03:00", scheduled_at: "2026-10-05T07:30:00-03:00",
    scheduled_place: "Laboratório Simulado — Rua XV, 100",
    result: { available: true, pickup_at_unit: false, released_on: "2026-10-08", pdf_available: true } },
  { id: "ex-ack", label: "Hemoglobina glicada", status_label: "Coletado",
    requested_at: "2026-10-01T09:00:00-03:00", scheduled_at: "2026-10-05T07:30:00-03:00",
    scheduled_place: "Laboratório Simulado — Rua XV, 100", result: null },
  { id: "ex-unit", label: "Sorologia para HIV", status_label: "Resultado pronto",
    requested_at: "2026-10-01T09:00:00-03:00", scheduled_at: null, scheduled_place: null,
    result: { available: false, pickup_at_unit: true, released_on: "2026-10-08", pdf_available: false } }
] };

// Só o caminho /citizen/...: os módulos do Vite (/wpda/src/modules/citizen/…) passam direto.
async function mockApi(page: Page, exams: unknown = EXAMS) {
  await page.route((url) => url.pathname.startsWith("/citizen/"), async (route: Route) => {
    const path = new URL(route.request().url()).pathname.replace(/^\/citizen/, "");
    const json = (body: unknown, status = 200) =>
      route.fulfill({ status, contentType: "application/json", body: JSON.stringify(body) });
    switch (path) {
      case "/session": return json({ phone_masked: "(**) *****-5432", time_zone: "America/Sao_Paulo", features: [ "exam_tracking" ] });
      case "/consent_term": return json({ version: "1", body: "Termo de teste" });
      case "/people": return json({ people: [ PERSON ] });
      case "/neighborhoods": return json({ neighborhoods: [] });
      case "/notices": return json({ notices: [], unread_count: 0 });
      case "/appointments": return json({ appointments: [] });
      case "/triages": return json({ citizen: { ...PERSON, verified_at: "2026-09-01T10:00:00-03:00" }, triages: [] });
      case "/exams": return json(exams);
      case "/exams/ex-imm/pdf": return route.fulfill({ status: 200, contentType: "application/pdf", body: "%PDF-1.7\n%%EOF\n" });
      default: return json({ error: "not_found" }, 404);
    }
  });
}

async function openMyExams(page: Page) {
  await page.goto("/wpda/");
  await page.getByRole("button", { name: "Concordo" }).click();
  await page.getByRole("button", { name: `Ver triagens de ${PERSON.cpf_masked}` }).click();
  await page.getByRole("button", { name: "Meus exames" }).click();
  await expect(page.getByRole("heading", { level: 1, name: "Meus exames" })).toBeVisible();
}

test.beforeEach(({}, info) => {
  test.skip(!PHONES.has(info.project.name), "o fluxo do cidadão só abre no celular (device-gate.spec.ts)");
});

test("as três políticas do laudo, sem rolagem lateral", async ({ page }) => {
  await mockApi(page);
  await openMyExams(page);

  const imm = page.getByRole("listitem", { name: "Hemograma completo" });
  await expect(imm.getByText("Resultado disponível — liberado em 08/10/2026", { exact: true })).toBeVisible();
  await expect(imm.getByRole("button", { name: "Baixar resultado de Hemograma completo" })).toBeVisible();

  const ack = page.getByRole("listitem", { name: "Hemoglobina glicada" });
  await expect(ack.getByText("Coletado", { exact: true })).toBeVisible();
  await expect(ack.getByRole("button")).toHaveCount(0);

  const unit = page.getByRole("listitem", { name: "Sorologia para HIV" });
  await expect(unit.getByText("Retire o resultado na sua unidade de saúde.", { exact: true })).toBeVisible();
  await expect(unit.getByRole("button")).toHaveCount(0);

  const overflow = await page.evaluate(() => document.documentElement.scrollWidth - document.documentElement.clientWidth);
  expect(overflow).toBeLessThanOrEqual(0);
});

test("par declarado: o convite e nenhum exame", async ({ page }) => {
  await mockApi(page, { verified: false, exams: EXAMS.exams, next_before: null });
  await openMyExams(page);
  await expect(page.getByRole("note", { name: "cadastro não validado" })).toBeVisible();
  await expect(page.getByRole("listitem")).toHaveCount(0);
  await expect(page.getByRole("button", { name: "Validar no posto" })).toBeVisible();
});

test("baixar o resultado: arquivo sem dado pessoal", async ({ page }, info) => {
  test.skip(info.project.name !== "pixel", "download por endereço blob: conferido no Chromium");
  await mockApi(page);
  await openMyExams(page);
  const [ download ] = await Promise.all([
    page.waitForEvent("download"),
    page.getByRole("button", { name: "Baixar resultado de Hemograma completo" }).click()
  ]);
  expect(download.suggestedFilename()).toBe("exame-ex-imm.pdf");
});
```

- [ ] **Step 3: Run the e2e on main to see it fail, then on the branch**

Para ver falhar, aponte para o `wpda-dev` do `main` (5176), que não tem a tela:

Run: `cd $W && WPDA_URL=http://curitiba.localhost:5176 npx playwright test e2e/exams.spec.ts --project=pixel`
Expected: FAIL — o botão "Meus exames" não existe no `main`.

Run: `cd $W && WPDA_URL=http://curitiba.localhost:5193 npx playwright test`
Expected: PASS nos 5 perfis — `exams.spec.ts` roda em `iphone` e `pixel` (o download só em `pixel`) e é pulado em `ipad` e nos desktops; `device-gate.spec.ts` segue verde. Se faltar navegador do Playwright na máquina, **pare e peça ao usuário** (`npx playwright install` baixa arquivos).

- [ ] **Step 4: Commit**

```bash
cd $W
$G add e2e/exams.spec.ts
$G commit -m "test: cover my exams on phone profiles with playwright

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–6 na branch `feat/mod-21a-exams`; o api do 21a na **3041** com a semente do 21a e o `fake-lab` do compose; o container `wpda-mod21` da Task 6.
- Produces: branch pronta para merge (depois do merge do api do 21a); card F-21.10 atualizado.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5193 npx playwright test`
Expected: tudo verde: a base anotada na Task 0 mais **5 arquivos de teste novos**; o e2e nos 5 perfis. Um número de testes que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio (o `dist/` do build está no `.gitignore`) e contagem `0`.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage" src/lib/exams.ts src/modules/citizen/MyExamsStep.tsx
sed -n '/─── Meus exames (módulo 21a/,/^async function pdf/p' src/lib/citizenApi.ts | grep -n -i "cpf\|phone\|critical\|reason\|values"
grep -n "critical\|denial\|reason\|values" src/modules/citizen/MyExamsStep.tsx
```

Expected: os três sem saída (o comentário do bloco do cliente fala em "motivo de negação" e "crítico" em português, não com esses nomes).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do wpda contra a spec (§3, §6 "Cidadão" e "Linha do tempo", §7, §11 F-21.10 e F-21.13), o ADR 0036 e o contrato (§9, §10). Pontos de atenção:
- par declarado: nenhum exame na tela, mesmo que o api mande; 403 `not_verified` vira o convite;
- o laudo sai só de `result` (pickup vence; PDF só com `available && pdf_available`); a situação nunca decide o laudo;
- nada de motivo de negação, crítico, valores, solicitante, CPF ou CNS; campo a mais é descartado no cliente;
- PDF com `fetch` e `blob:`, nome `exame-<id>.pdf`; 404 diz que o resultado não está disponível; 401 volta ao login; 429 com a frase do download;
- a entrada `exam` de "Meu histórico" baixa por `examPdf`; documentos seguem por `documentPdf`;
- sem `exam_tracking` na sessão, "Minhas triagens" é a de hoje; URL só com ids, cursor e `limit`;
- interface do canal: componentes existentes, alvos de 48 px, cores por papel do tema, guarda verde, e2e de celular verde.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 21a na porta **3041** (plano do api: semente com Curitiba em `record`, `clinical_record`, `patient_timeline`, `exam_tracking`, `exam_fhir_inbound` e `fake_lab` ligados; catálogo com um exame `immediate` (hemograma), um `after_acknowledgement` (HbA1c) e um `never_citizen`; o prestador "Laboratório Simulado" com o `fake-lab` no ar; um celular com uma pessoa **validada** com exames nas três políticas e uma **só declarada**). Se a semente não tiver algum desses exames, crie pelo dashboard do 21a (pedido na consulta, agendamento pelo prestador e o resultado pelo `fake-lab`). O container `wpda-mod21` da Task 6 continua no ar na **5193**, com o proxy para `http://api:3041`.

Abra `http://curitiba.localhost:5193/wpda/` no celular de verdade ou no modo de dispositivo do navegador (375 px, toque ligado: o fluxo só abre no celular). O usuário entra com o celular da semente e o código do SMS de dev, como nas provas do canal web. Confira com screenshot:
- pessoa **validada** → "Ver triagens de …" → **Meus exames**: cada exame com a situação, "Pedido em …" e "Marcado para … — <local>";
  - o hemograma (`immediate`), com o resultado enviado pelo `fake-lab`: "Resultado disponível — liberado em …" e **Baixar resultado (PDF)** → o arquivo `exame-<id>.pdf` com o PDF de exemplo do `fake-lab`;
  - a HbA1c (`after_acknowledgement`) com resultado e **sem** ciência: nada de laudo nem botão; dar a ciência no dashboard (como o solicitante) e voltar a "Meus exames": agora o laudo e o botão;
  - o exame `never_citizen` com resultado: "Retire o resultado na sua unidade de saúde.", sem botão;
  - um exame negado pela regulação (se a semente tiver, ou negar um pelo dashboard): a situação sem o motivo; um resultado marcado como crítico: nada diz "crítico";
- **Meu histórico** da mesma pessoa: as entradas **Exame** com a situação; baixar o hemograma pela linha do tempo → `exame-<id>.pdf`;
- pessoa **só declarada** → Meus exames: o convite e "Validar no posto" → o código do posto;
- no maintenance (`http://maintenance.localhost:5177`, login do usuário), desligar `exam_tracking` em Curitiba, sair e entrar de novo no wpda: "Minhas triagens" sem o botão; religar;
- a barra de acessibilidade em alto contraste: a tela legível, sem cor fixa;
- a 375 px, nenhuma rolagem lateral.

Depois: `docker rm -f wpda-mod21 && docker volume rm wpda-mod21-node-modules`. Deixe os interruptores e a ciência de teste como o plano do api pedir (ou registre o que mudou na semente de dev).

- [ ] **Step 5: Card**

Se o lado api do F-21.10 já estiver **Done** no board, mova o F-21.10 para **Done** (um `gh project item-edit` como na Task 0, com o id da opção "Done", `98236657`):

```bash
gh project item-edit --id <PVTI_do_F-21.10> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id 98236657
```

Senão, deixe em In Progress e avise a sessão do api.

- [ ] **Step 6:** **Pare.** O merge do wpda só vem depois do merge do api do 21a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard e wpda → maintenance → catálogo (contrato §14). Antes do push, confira `origin/main..main` no wpda e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21a-exams
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/wpda-mod21
/opt/homebrew/bin/git branch -d feat/mod-21a-exams
```

Depois do merge, confira o `wpda-dev` (5176): ele lê `apps/wpda` e recompila sozinho.

---

## Divergências propostas ao contrato

1. **E1 — A pessoa nas rotas do cidadão.** O contrato §9 tem `GET /citizen/exams?before=&limit=` e `GET /citizen/exams/:id/pdf` sem dizer de quem: a sessão do wpda é do **celular**, que tem até 10 pessoas, cada uma validada ou não (ADR 0017). Proposta, no molde já em produção do módulo 20 (W1; `history`, `health_summary` e `documents/:id/pdf` levam `citizen_id`): `GET /citizen/exams?citizen_id=&before=&limit=` (404 `not_found` se a pessoa não é do celular da sessão) e `GET /citizen/exams/:id/pdf?citizen_id=` (o api confere que o exame é daquela pessoa e que o par é validado).
2. **E2 — `exam_tracking` na sessão do cidadão.** A sessão traz `features` desde o módulo 20 (W2), hoje só com `patient_timeline`. Proposta: incluir `exam_tracking` quando ligado e utilizável; ausente = botão escondido. Sem isso o botão apareceria em toda cidade e levaria a um 403.
3. **E3 — Semântica de `result` e formato das datas.** O contrato dá os campos, não quando cada um vale. Proposta, para a política ficar só no servidor:
   - `result: null` enquanto a pessoa não pode saber do laudo (sem resultado; ou `after_acknowledgement` antes da ciência);
   - `immediate` depois do registro, e `after_acknowledgement` depois da ciência: `available: true`, `pickup_at_unit: false`, `pdf_available: true` quando há PDF;
   - `never_citizen` depois do registro: `available: false`, `pickup_at_unit: true`, `pdf_available: false` (e o PDF responde 404);
   - `requested_at` e `scheduled_at` em ISO 8601 com fuso; `released_on` como data `AAAA-MM-DD`.
   A tela já se defende de combinações erradas (Task 2: "retirar" vence; PDF só com `available && pdf_available`).
4. **E4 — Par declarado e `status_label`.** Proposta: par declarado recebe 200 `{ verified: false, exams: [], next_before: null }` (a tela aceita também 403 `not_verified`). `status_label` vem em português, pronto, sem o motivo: para `denied`, algo como "Não autorizado — procure a sua unidade"; para `resulted` sob `after_acknowledgement` antes da ciência, o rótulo da etapa anterior ("Coletado"), para não anunciar o que a política ainda não libera.
5. **E5 — Entrada `exam` do cidadão na linha do tempo.** O contrato §10 diz "entrada com `status_label`" e "`document`-like `{ id, downloadable: true }`". Proposta: `status_label` vira campo **opcional** de `<citizen_entry>` (presente só no `exam`; vence o `status`); `document` = `{ id: <exam_order id>, short_code: null, verification_url: null, downloadable: true }` só quando a política permite (nunca no `never_citizen`); o PDF sai de `GET /citizen/exams/:id/pdf?citizen_id=` (não de `/citizen/documents/:id/pdf`), com o mesmo 429 `too_many_requests` dos documentos.

## Self-review

- **Cobertura (spec §3, §6 "Cidadão" e "Linha do tempo", §7, §8, §9 "Cidadão"; contrato §9, §10; F-21.10 e o lado cidadão de F-21.13):** "Meus exames" só para par validado, com aviso para o declarado — Tasks 3 e 4; situação, data e local — Tasks 2 e 3; laudo pelas três políticas — Tasks 2, 3 e 6; download do PDF — Tasks 1, 3 e 6; entrada `exam` na linha do tempo — Tasks 1 e 5; nunca motivo nem crítico — Tasks 1, 3 e 7; interruptor — Task 4; regras do ciclo de interface (tema, `isPhone`, Playwright) — Global Constraints, Tasks 3 (guarda), 6 (e2e) e 7; prova no navegador — Task 7; card — Tasks 0 e 7.
- **Placeholders:** nenhum código faltando. As únicas variáveis são de ambiente (ids do card e da opção In Progress, lidos na Task 0; a semente do api do 21a; o nome do container do api se o plano do api usar outro).
- **Tipos:** `CitizenExam`, `CitizenExamResult`, `CitizenExams`, `EXAMS_PAGE_SIZE` (Task 1) são os usados nas Tasks 2–6; `citizenApi.exams(id, { before? })` e `citizenApi.examPdf(examId, citizenId)` têm a mesma forma no cliente, na tela e nos testes; `EXAM_KIND`, `EXAM_PDF_GONE`, `examFilename` (Task 2) são os da Task 5; `entryStatusText`/`entryFilename` (Task 5) são os do `MyHistoryStep`; `onExams` é opcional no `HistoryStep` para os testes antigos seguirem iguais; `status_label?` opcional no `CitizenEntry` mantém o teste do módulo 20 sem edição.
- **Review Focus:** 1 — Task 3 ("par declarado: …", "403 not_verified: …") e Task 6 ("par declarado: …"); 2 — Task 2 ("tabela das políticas") e Task 3 ("três políticas"); 3 — Task 1 ("examPdf: …"), Task 2 ("examFilename: …"), Task 3 ("baixar o resultado", "404", "429", "interruptor desligado"); 4 — Task 5 ("exame na linha do tempo…", "exame que sumiu…"); 5 — Task 1 ("normaliza o que faltar"), Task 3 ("campos ausentes…") e Task 4 ("sem exam_tracking…").
