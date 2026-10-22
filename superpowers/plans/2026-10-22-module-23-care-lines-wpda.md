# Módulo 23 — Linhas de cuidado (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 23** rodando na branch dele (`feat/mod-23-care-lines`, worktree `apps/api/.claude/mod23`, porta **3045**) com `GET/POST/DELETE /citizen/declared_conditions`, `GET /citizen/care`, `overdue` e `last_done_at` nas ofertas do catálogo do cidadão e `care_lines` em `features` da sessão do cidadão (contrato §7 e Divergências W1–W6 deste plano). Ele é **mergeado antes** deste (contrato §10: api → dashboard e wpda → maintenance → catálogo). As Tasks 0–7 não dependem dele (os testes usam dublês e o Playwright responde as rotas); a prova da Task 8 depende.

**Goal:** Lado wpda de F-23.7 e F-23.8: no **perfil** de um par validado, as **condições de saúde declaradas** (diabetes, pressão alta, gravidez, acompanhamento de saúde mental) — opcionais, retiráveis, nunca diagnóstico —; a tela **"Meu acompanhamento"**, só para par validado, com as linhas de cuidado em que a pessoa foi inscrita e o que está para fazer, em linguagem simples ("Medir a pressão — está na hora"), sem código, sem diagnóstico além do nome da linha, sem sugestão e sem motivo de encerramento; e o **aviso de triagem periódica vencida** na tela de triagens ("Está na hora de refazer"), que depende só do catálogo do módulo 15 — e que "Meu acompanhamento" também mostra.

**Architecture:** O cliente (`src/lib/citizenApi.ts`) ganha `declaredConditions`, `declareCondition`, `withdrawCondition` e `care`, com a normalização na borda como o resto do arquivo, e o `catalog` passa a normalizar `overdue` e `last_done_at` nas ofertas. As regras de tela ficam puras em `src/lib/care.ts` (frase de cada item pela situação, título com maiúscula, triagens vencidas do catálogo, opções e frases). Peças novas: `src/modules/citizen/HealthConditions.tsx` (a seção do perfil, com `Toggle`) e `src/modules/citizen/MyCareStep.tsx` (a tela). Mudam o mínimo: `CatalogStep` (seção "Está na hora de refazer"), `ProfileStep` (a seção, no perfil conferido no posto), `HistoryStep` (o botão "Meu acompanhamento"), `Flow` (o estado `care`). Quem pode ver o quê é do api (par validado, só linhas inscritas, oferta vencida); a tela nunca deduz.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `setupFiles` com `cleanup` e o dublê de celular `stubDevice`), `vi.spyOn(citizenApi, …)`, Playwright (5 perfis de dispositivo, `e2e/`), Vite 5 (base `/wpda/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-22-module-23-care-lines-design.md` (§2 decisões 5 e 8 e as decisões do coordenador — condição declarada e triagem periódica vencida —, §3 interruptor, §4 `citizen_declared_conditions`, §6 "Cidadão (wpda)", §7 segurança e LGPD, §8 testes, §10 F-23.7 e F-23.8), `docs/adr/0040.md` e o contrato `docs/superpowers/plans/2026-10-22-module-23-care-lines-contracts.md` (§1, §7, §9, §10; as diferenças estão em "Divergências propostas ao contrato", W1–W6). Herdados: ADR 0017 (par CPF + celular, validação presencial), ADR 0027 (perfil e catálogo do módulo 15) e os planos do wpda dos módulos 20 e 21a (invariantes do cidadão).

## Global Constraints

- Rotas (contrato §7, na forma do plano do api (L9, L11) e das Divergências W1–W6; sessão do cidadão, cookie `citizen_session`, mesma origem):
  - `GET /citizen/declared_conditions?citizen_id=` → `{ conditions: [ { condition, declared_at } ] }` (só as vigentes); `POST /citizen/declared_conditions` com `{ citizen_id, condition }` no corpo (201) e `DELETE /citizen/declared_conditions?citizen_id=` com `{ condition }` no corpo, os dois devolvendo a lista vigente (L11 do plano do api; W1); `none` retira as outras; 403 `not_verified` para par só declarado; 422 `invalid_condition`;
  - `GET /citizen/care?citizen_id=` → `{ lines: [ { name, items: [ { label, status, due_on|null } ] } ], overdue_triages: [ { protocol_name, title, last_done_at } ] }` (só linhas em que a pessoa está inscrita; L9); 403 `not_verified`;
  - `GET /citizen/people/:id/catalog` → toda oferta com intervalo de repetição ganha `overdue: bool` e `last_done_at`, sempre (o módulo 15 não tem interruptor; L9, W4);
  - `GET /citizen/session` → `features` com `care_lines` quando ligado e utilizável (W3);
  - 403 `feature_disabled` (`feature: "care_lines"`) nas rotas novas; 401 dispara `citizen:unauthenticated`, como hoje.
- **A condição declarada é do cidadão e nunca é diagnóstico** (spec, decisões do coordenador): a tela diz isso ao lado das opções; cada uma é opcional e se retira com o mesmo toque; o nome da condição **nunca** vai na URL (é dado de saúde) — só no corpo; o wpda não manda `none` (ausência de condição ativa já é "nenhuma"; W5).
- **"Meu acompanhamento" só para par validado** (spec §6): par declarado (ou 403 `not_verified`) vê só o convite para validar no posto e nenhuma linha, mesmo que o api mande. A tela mostra **só** os campos do contrato: nome da linha, rótulo do item, situação em frase e a data; campo a mais (código, motivo, sugestão, CID/CIAP) é descartado no cliente. Situação desconhecida não aparece crua para o cidadão (some a linha da situação, o item fica).
- **Triagem periódica vencida depende do catálogo do módulo 15, não de `care_lines`** (spec, decisões do coordenador): o aviso na tela de triagens aparece para qualquer oferta com `overdue: true`, inscrito ou não; "Meu acompanhamento" (que exige `care_lines`) repete o aviso. Sem `overdue` (api antigo), a tela de triagens fica como hoje.
- Interruptor (spec §3): sem `care_lines` em `session.features`, o perfil não oferece condição declarada e "Minhas triagens" não tem "Meu acompanhamento" — exatamente como hoje. Se o interruptor cair com a tela aberta, o 403 `feature_disabled` vira `CARE_DISABLED` ("O acompanhamento ainda não está disponível nesta cidade."), não a frase do histórico.
- Na URL vão só o id da pessoa — nunca CPF, celular, condição, rótulo ou texto. Nada em `localStorage`, `sessionStorage` ou `console`.
- Interface do canal (ciclo de interface, subprojetos 4 e 6e): `Screen`, `BigButton`, `ErrorText`, `Toggle` de `ui.tsx`; alvos de toque de 48 px ou mais, texto de 18 px; cores **só por papel do tema** (`var(--warn-bg)`, `var(--accent-bg)`, `var(--ok-bg)`, `var(--rule2)`, `var(--ink2)`), sem reserva `var(--x, …)` e sem branco fixo — a guarda `src/theme/guards.test.ts` roda na suíte; o fluxo do cidadão só abre no celular (`isPhone`, `src/lib/device.ts`) e o `e2e/device-gate.spec.ts` segue verde nos 5 perfis. Nenhum componente comum muda. Achado só visual não vira tarefa.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` e `vi.useRealTimers()` no `afterEach`; a suíte roda em `TZ=America/Sao_Paulo`.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-23.1..F-23.8** do board #1 para **In Progress** (Task 0); o wpda é o último app de F-23.7 e F-23.8 (com o api): eles vão a **Done** quando o api deles estiver Done e este plano terminar, e a **Verified** depois da prova no navegador (Task 8).

## Ambiente de execução

- Worktree **`/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/wpda-mod23`** (nunca dentro de `apps/wpda`: o compose monta `./apps/wpda:/app` e um segundo React derruba o `wpda-dev`), branch `feat/mod-23-care-lines` a partir de `origin/main`, com `npm ci` (Task 0). Caminhos de arquivo das tasks relativos a ele.
- Variáveis usadas nos comandos: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod23`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos: `npx tsc --noEmit`. e2e: `cd $W && WPDA_URL=http://curitiba.localhost:5199 npx playwright test` (precisa do Vite da Task 7 no ar).
- Ambiente de teste: `vitest.config.ts` com `environment: "jsdom"`, `globals: false`, `setupFiles: ["./src/test/setup.ts"]` (jest-dom, `cleanup`, `sessionStorage.clear()` e o celular padrão `stubDevice()`), `exclude` de `e2e/**` e `TZ=America/Sao_Paulo`. `fmtCalendarDate("2026-11-12")` → `"12/11/2026"`.
- O api do módulo 23 roda em dev na **3045**, dentro do container `api-dev` (rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5199**, com `node_modules` num volume próprio (o `node_modules` do worktree é de macOS). `/citizen` já está no proxy do `vite.config.ts`: nada muda nele. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Par só declarado abre "Meu acompanhamento".** Um api que mande linhas para par declarado, ou responda 403 `not_verified`, não pode fazer a tela mostrar linha nenhuma: só o convite e "Validar no posto". Teste: Task 5, "par declarado (403 not_verified): só o convite, mesmo se o api mandar linhas depois".
2. **Código, diagnóstico ou motivo vazando para o cidadão.** Campo a mais no item (`ciap`, `cid10`, `reason`, `ended_reason`, `suggestion`), situação com código novo ou `untracked`: a tela mostra só o nome da linha, o rótulo em frase e a data; `untracked` vira "Confira com a sua equipe de saúde"; código desconhecido some. Testes: Task 2, "careItemView: tabela das situações; desconhecida não aparece crua"; Task 5, "nada de código, CID ou motivo: só os campos do contrato".
3. **Retirar a condição declarada.** O toque que desliga tem de mandar `DELETE` com a condição **no corpo** (nunca na URL), a tela refletir a lista que o api devolveu e, se o api responder 204 sem corpo, reler a lista em vez de mostrar tudo desligado. Testes: Task 1, "withdrawCondition: DELETE com a condição no corpo; 204 relê a lista"; Task 4, "desligar retira e diz que retirou".
4. **Triagem periódica vencida com api antigo ou campo faltando.** Sem `overdue`, a tela de triagens fica igual (nada de "Está na hora de refazer"); `overdue` sem `last_done_at` não mostra "undefined"; a oferta vencida começa pelo mesmo botão da triagem. Testes: Task 1, "catalog: overdue false e last_done_at null quando faltam"; Task 3, "sem overdue (api antigo): Disponíveis como hoje" e "vencida sem data: sem a linha da data".
5. **Interruptor desligado ou caindo com a tela aberta.** Sem `care_lines` na sessão: nem a seção do perfil nem o botão; com a tela aberta e 403 `feature_disabled`, a frase do acompanhamento (não a do histórico) e a seção do perfil some. Testes: Task 4, "sem care_lines na sessão: o perfil conferido é o de hoje"; Task 5, "interruptor desligado com a tela aberta: a frase do acompanhamento"; Task 6, "sem care_lines na sessão: Minhas triagens como hoje".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-23.1..F-23.8, conferência do main e do api do 23, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.care.test.ts` | condição declarada, `care` e `overdue`/`last_done_at` no catálogo | 1 |
| `src/lib/care.ts`, `src/lib/care.test.ts` | regras de tela do acompanhamento e da triagem vencida | 2 |
| `src/modules/citizen/CatalogStep.tsx`, `src/modules/citizen/CatalogStep.overdue.test.tsx` | "Está na hora de refazer" na tela de triagens (F-23.8) | 3 |
| `src/modules/citizen/HealthConditions.tsx`, `src/modules/citizen/ProfileStep.tsx`, `src/modules/citizen/HealthConditions.test.tsx` | condições de saúde declaradas no perfil (F-23.7) | 4 |
| `src/modules/citizen/MyCareStep.tsx`, `src/modules/citizen/MyCareStep.test.tsx` | a tela "Meu acompanhamento" (F-23.7, F-23.8) | 5 |
| `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.care.test.tsx`, `README.md` | o botão em "Minhas triagens" e o estado `care` no `Flow` | 6 |
| `e2e/care.spec.ts` | perfil, acompanhamento e triagem vencida nos perfis de celular | 7 |
| — | suíte, build, e2e, revisão e prova no navegador | 8 |

**Estratégia de teste:** cliente com `fetch` falso conferindo URL (só o id da pessoa), método, corpo (a condição só no corpo), o 204 do `DELETE` e a normalização (Task 1); regras puras com tabela de casos: cada situação em frase, título, situação desconhecida, triagens vencidas sem repetir (Task 2); a tela de triagens com e sem `overdue` (Task 3); a seção do perfil com `vi.spyOn(citizenApi, …)` — declarar, retirar, recusa, par declarado, interruptor (Task 4); a tela nova com par validado, par declarado, campo a mais, lista vazia, triagem vencida e interruptor caindo (Task 5); a navegação real pelo `Flow` com a sessão com e sem o interruptor (Task 6); o Playwright nos perfis de celular, com as rotas respondidas pelo teste, para as telas reais no navegador e a largura de 375–412 px sem rolagem lateral (Task 7). A prova final é no navegador, contra o api do 23 (Task 8).

---

### Task 0: Cards, conferência do main e do api do 23, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`9748713` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`).
- Produces: cards F-23.1..F-23.8 em In Progress (ou além); worktree `$W` na branch `feat/mod-23-care-lines`; a base anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-23\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card antes de In Progress (os outros planos do 23 movem os mesmos cards), com o `id` do item e o id da opção "In Progress":

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito em In Progress ou além. Se algum card não existir, **pare e pergunte**.

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda fetch -q origin
$G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '^export interface CatalogEntry { protocol_name: string; title: string; summary: string | null }\|      available: (d.available ?? \[\]).map(withSummary),\|^async function call<T>\|  examPdf: (examId: string, citizenId: string): Promise<Blob> =>'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/CatalogStep.tsx | grep -n '      {c && c.available.length > 0 && (\|      {c?.in_progress && ('
$G -C $R/apps/wpda show origin/main:src/modules/citizen/ProfileStep.tsx | grep -n 'VERIFIED_PROFILE_TEXT}</p>'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/HistoryStep.tsx | grep -n '    onExams?: () => void }) {\|{onExams && <BigButton'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n '  | { at: "exams"; consentVersion: string | null; citizenId: string }\|        onExams={hasSessionFeature("exam_tracking")\|    case "verify-code":'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^a entrada \*\*Exame\*\* baixa o PDF pela mesma rota do exame.'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod23 -b feat/mod-23-care-lines origin/main
cd $R/.worktrees/wpda-mod23 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 8 espera a base mais **7 arquivos de teste novos** e o `e2e/care.spec.ts`.

- [ ] **Step 4: Confira o api do 23** (só informativo; as Tasks 1–7 não dependem dele)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-23-care-lines
docker exec -w /rails/.claude/mod23 api-dev bin/rails routes 2>/dev/null | grep -E "/citizen/(declared_conditions|care)" | head
```

Expected: a branch e as rotas. Compare com o contrato §7 e as Divergências W1–W6; diferença de forma (envelope, nome de campo, `DELETE` com corpo) vira ajuste em `src/lib/citizenApi.ts` e nota para a Task 8.

---

### Task 1: Cliente — condição declarada, acompanhamento e `overdue` no catálogo

**Files:**
- Modify: `src/lib/citizenApi.ts`
- Test: `src/lib/citizenApi.care.test.ts`

**Interfaces:**
- Consumes: `call` (privado de `citizenApi.ts`), `withSummary`, `Catalog`, `CatalogEntry`.
- Produces: tipos `DeclaredCondition`, `DeclaredConditionEntry`, `CareItem`, `CareLineView`, `OverdueTriage`, `CitizenCare`; `CatalogEntry` ganha `overdue?: boolean` e `last_done_at?: string | null`; em `citizenApi`: `declaredConditions(citizenId): Promise<DeclaredConditionEntry[]>`, `declareCondition(citizenId, condition): Promise<DeclaredConditionEntry[]>`, `withdrawCondition(citizenId, condition): Promise<DeclaredConditionEntry[]>`, `care(citizenId): Promise<CitizenCare>`; `catalog` normaliza `overdue` (`false` quando falta) e `last_done_at` (`null`) em `available` e `suggested`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.care.test.ts
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { citizenApi, ApiError } from "./citizenApi";

function reply(status: number, body?: unknown) {
  return new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}

describe("cliente do acompanhamento (contrato §7)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => { vi.unstubAllGlobals(); });

  const urlOf = (i = 0) => String(fetchMock.mock.calls[i][0]);
  const initOf = (i = 0) => fetchMock.mock.calls[i][1] as RequestInit;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));

  it("declaredConditions: só o id na URL; normaliza a lista", async () => {
    fetchMock.mockResolvedValueOnce(reply(200, { conditions: [ { condition: "diabetes", declared_at: "2026-10-20T10:00:00-03:00" } ] }));
    expect(await citizenApi.declaredConditions("p1")).toEqual([ { condition: "diabetes", declared_at: "2026-10-20T10:00:00-03:00" } ]);
    expect(urlOf()).toBe("/citizen/declared_conditions?citizen_id=p1");
    expect(initOf().method).toBe("GET");
    fetchMock.mockResolvedValueOnce(reply(200, {}));
    expect(await citizenApi.declaredConditions("p1")).toEqual([]);
  });

  it("declareCondition: POST com a pessoa e a condição só no corpo, em JSON (201)", async () => {
    fetchMock.mockResolvedValueOnce(reply(201, { conditions: [ { condition: "hypertension", declared_at: "2026-10-22T09:00:00-03:00" } ] }));
    expect((await citizenApi.declareCondition("p1", "hypertension")).map((c) => c.condition)).toEqual([ "hypertension" ]);
    expect(urlOf()).toBe("/citizen/declared_conditions");
    expect(initOf().method).toBe("POST");
    expect((initOf().headers as Record<string, string>)["Content-Type"]).toBe("application/json");
    expect(bodyOf()).toEqual({ citizen_id: "p1", condition: "hypertension" });
  });

  it("withdrawCondition: DELETE com a condição no corpo; 204 relê a lista", async () => {
    fetchMock.mockResolvedValueOnce(reply(200, { conditions: [] }));
    expect(await citizenApi.withdrawCondition("p1", "pregnancy")).toEqual([]);
    expect(urlOf(0)).toBe("/citizen/declared_conditions?citizen_id=p1");
    expect(initOf(0).method).toBe("DELETE");
    expect(bodyOf(0)).toEqual({ condition: "pregnancy" });
    fetchMock
      .mockResolvedValueOnce(new Response(null, { status: 204 }))
      .mockResolvedValueOnce(reply(200, { conditions: [ { condition: "diabetes", declared_at: "2026-10-20T10:00:00-03:00" } ] }));
    expect((await citizenApi.withdrawCondition("p1", "pregnancy")).map((c) => c.condition)).toEqual([ "diabetes" ]);
    expect(initOf(2).method).toBe("GET");
  });

  it("care: só o id na URL; descarta campo a mais e normaliza o que falta", async () => {
    fetchMock.mockResolvedValueOnce(reply(200, { lines: [
      { name: "Pessoa com diabetes", reason: "problem", ciap: "T90",
        items: [ { label: "medir a pressão", status: "overdue", due_on: "2026-09-01", cid10: "E11" }, { label: "visita do agente" } ] },
      { items: [] }
    ], overdue_triages: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", last_done_at: "2025-10-02", reason: "x" } ] }));
    expect(await citizenApi.care("p1")).toEqual({ lines: [
      { name: "Pessoa com diabetes", items: [ { label: "medir a pressão", status: "overdue", due_on: "2026-09-01" }, { label: "visita do agente", status: "", due_on: null } ] },
      { name: "", items: [] }
    ], overdue_triages: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", last_done_at: "2025-10-02" } ] });
    fetchMock.mockResolvedValueOnce(reply(200, {}));
    expect(await citizenApi.care("p1")).toEqual({ lines: [], overdue_triages: [] });
    expect(urlOf()).toBe("/citizen/care?citizen_id=p1");
  });

  it("care: 403 not_verified vira ApiError com o código", async () => {
    fetchMock.mockResolvedValueOnce(reply(403, { error: "not_verified" }));
    await expect(citizenApi.care("p1")).rejects.toEqual(new ApiError(403, "not_verified"));
  });

  it("catalog: overdue false e last_done_at null quando faltam; true preservado", async () => {
    fetchMock.mockResolvedValueOnce(reply(200, {
      available: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: null, overdue: true, last_done_at: "2025-10-02" },
        { protocol_name: "respiratoria", title: "Sintomas respiratórios", summary: null } ],
      suggested: [], recent: [], in_progress: null
    }));
    const c = await citizenApi.catalog("p1");
    expect(c.available).toEqual([
      { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: null, overdue: true, last_done_at: "2025-10-02" },
      { protocol_name: "respiratoria", title: "Sintomas respiratórios", summary: null, overdue: false, last_done_at: null }
    ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/citizenApi.care.test.ts`
Expected: FAIL — `citizenApi.declaredConditions` (e as outras) não existem; o catálogo não traz `overdue`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/citizenApi.ts`, troque

```ts
export interface CatalogEntry { protocol_name: string; title: string; summary: string | null }
```

por

```ts
// Módulo 23 (F-23.8; contrato §7, W4): oferta com intervalo de repetição já
// vencido (`overdue`) e a última vez feita. Ausentes numa api anterior.
export interface CatalogEntry {
  protocol_name: string; title: string; summary: string | null; overdue?: boolean; last_done_at?: string | null;
}
```

Em `catalog`, troque

```ts
      suggested: (d.suggested ?? []).map(e => ({ ...withSummary(e), source_title: e.source_title ?? null })),
      available: (d.available ?? []).map(withSummary),
```

por

```ts
      suggested: (d.suggested ?? []).map(e => ({ ...withOverdue(withSummary(e)), source_title: e.source_title ?? null })),
      available: (d.available ?? []).map(e => withOverdue(withSummary(e))),
```

e, logo depois da função `withSummary`, acrescente

```ts
function withOverdue<T extends CatalogEntry>(e: T): T {
  return { ...e, overdue: e.overdue === true, last_done_at: e.last_done_at ?? null };
}
```

Antes de `export interface CitizenSession {`, acrescente

```ts
// ─── Linhas de cuidado (módulo 23, ADR 0040; contrato §7; Divergências W1–W6) ───
// Condição declarada: dado de saúde dito pelo próprio cidadão, nunca
// diagnóstico; só no corpo, nunca na URL. "Meu acompanhamento": só as linhas
// em que a pessoa foi inscrita, em linguagem simples — campo a mais (código,
// motivo, sugestão) é descartado aqui.
export type DeclaredCondition = "diabetes" | "hypertension" | "pregnancy" | "mental_health" | "none";
export interface DeclaredConditionEntry { condition: DeclaredCondition | string; declared_at: string }
export interface CareItem { label: string; status: string; due_on: string | null }
export interface CareLineView { name: string; items: CareItem[] }
// L9 do plano do api: as triagens periódicas vencidas vêm junto.
export interface OverdueTriage { protocol_name: string; title: string; last_done_at: string | null }
export interface CitizenCare { lines: CareLineView[]; overdue_triages: OverdueTriage[] }

type RawConditions = { conditions?: { condition: string; declared_at?: string }[] } | undefined;
type RawCare = {
  lines?: { name?: string; items?: { label?: string; status?: string; due_on?: string | null }[] }[];
  overdue_triages?: { protocol_name: string; title?: string; last_done_at?: string | null }[];
};

function normalizeConditions(d: RawConditions): DeclaredConditionEntry[] {
  return (d?.conditions ?? []).map(c => ({ condition: c.condition, declared_at: c.declared_at ?? "" }));
}

function normalizeCare(d: RawCare): CitizenCare {
  return {
    lines: (d.lines ?? []).map(l => ({
      name: l.name ?? "",
      items: (l.items ?? []).map(i => ({ label: i.label ?? "", status: i.status ?? "", due_on: i.due_on ?? null }))
    })),
    overdue_triages: (d.overdue_triages ?? []).map(t => ({
      protocol_name: t.protocol_name, title: t.title ?? "", last_done_at: t.last_done_at ?? null
    }))
  };
}
```

e, no objeto `citizenApi`, troque

```ts
  examPdf: (examId: string, citizenId: string): Promise<Blob> =>
    pdf(`/exams/${encodeURIComponent(examId)}/pdf?citizen_id=${encodeURIComponent(citizenId)}`)
};
```

por

```ts
  examPdf: (examId: string, citizenId: string): Promise<Blob> =>
    pdf(`/exams/${encodeURIComponent(examId)}/pdf?citizen_id=${encodeURIComponent(citizenId)}`),
  // Condições declaradas ativas (só par validado: 403 not_verified).
  declaredConditions: async (citizenId: string): Promise<DeclaredConditionEntry[]> =>
    normalizeConditions(await call<RawConditions>("GET", `/declared_conditions?citizen_id=${encodeURIComponent(citizenId)}`)),
  // Plano do api (L11): POST com citizen_id e condição no corpo.
  declareCondition: async (citizenId: string, condition: DeclaredCondition): Promise<DeclaredConditionEntry[]> =>
    normalizeConditions(await call<RawConditions>("POST", "/declared_conditions", { citizen_id: citizenId, condition })),
  // Retirar grava withdrawn_at no api. A condição vai no CORPO, nunca na URL
  // (W1: o plano do api lê `condition` de params, que o Rails monta também do
  // corpo JSON). Sem corpo na resposta (204), relê a lista.
  withdrawCondition: async (citizenId: string, condition: DeclaredCondition): Promise<DeclaredConditionEntry[]> => {
    const d = await call<RawConditions>("DELETE", `/declared_conditions?citizen_id=${encodeURIComponent(citizenId)}`, { condition });
    return d?.conditions ? normalizeConditions(d) : citizenApi.declaredConditions(citizenId);
  },
  // "Meu acompanhamento": só par validado (403 not_verified).
  care: async (citizenId: string): Promise<CitizenCare> =>
    normalizeCare(await call<RawCare>("GET", `/care?citizen_id=${encodeURIComponent(citizenId)}`))
};
```

(O `call` já manda `Content-Type: application/json` e o corpo em toda escrita, `DELETE` inclusive, e devolve `undefined` no 204.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/citizenApi.care.test.ts src/lib/citizenApi.test.ts src/modules/citizen/CatalogStep.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos do catálogo seguem: os campos novos são opcionais no tipo).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/citizenApi.ts src/lib/citizenApi.care.test.ts
$G commit -m "feat: add declared conditions, care follow-up and overdue screenings to the citizen client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela do acompanhamento (`src/lib/care.ts`)

**Files:**
- Create: `src/lib/care.ts`
- Test: `src/lib/care.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `CareItem`, `DeclaredCondition` (Task 1); `fmtCalendarDate` (`format.ts`); `messageFor` (`ui.tsx`).
- Produces: constantes `CARE_NOT_VERIFIED`, `CARE_DISABLED`, `CARE_EMPTY`, `CONDITIONS_HELP`, `CONDITION_OPTIONS`; `careItemView(item): { title: string; status: string | null; tone: "ok" | "warn" | "neutral" }`; `lastDoneText(entry: { last_done_at? }): string | null`; `careMessage(error): string`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/care.test.ts
import { describe, it, expect } from "vitest";
import { ApiError } from "./citizenApi";
import { CARE_DISABLED, CONDITION_OPTIONS, careItemView, careMessage, lastDoneText } from "./care";

describe("careItemView", () => {
  it("tabela das situações; desconhecida não aparece crua", () => {
    expect(careItemView({ label: "consulta com a equipe", status: "ok", due_on: "2027-01-06" }))
      .toEqual({ title: "Consulta com a equipe", status: "Em dia", tone: "ok" });
    expect(careItemView({ label: "medir a pressão", status: "due_soon", due_on: "2026-11-12" }))
      .toEqual({ title: "Medir a pressão", status: "Faça até 12/11/2026", tone: "warn" });
    expect(careItemView({ label: "medir a pressão", status: "due_soon", due_on: null }))
      .toEqual({ title: "Medir a pressão", status: "Está chegando a hora", tone: "warn" });
    expect(careItemView({ label: "exame de hemoglobina glicada", status: "overdue", due_on: "2026-08-20" }))
      .toEqual({ title: "Exame de hemoglobina glicada", status: "Está na hora — procure a sua unidade de saúde", tone: "warn" });
    expect(careItemView({ label: "visita do agente de saúde", status: "untracked", due_on: null }))
      .toEqual({ title: "Visita do agente de saúde", status: "Confira com a sua equipe de saúde", tone: "neutral" });
    expect(careItemView({ label: "consulta", status: "paused", due_on: null })).toEqual({ title: "Consulta", status: null, tone: "neutral" });
    expect(careItemView({ label: "", status: "ok", due_on: null }).title).toBe("Cuidado da linha");
  });
});

describe("triagem periódica vencida", () => {
  it("frase da última vez; sem data, nada", () => {
    expect(lastDoneText({ last_done_at: "2025-10-02" })).toBe("Feita pela última vez em 02/10/2025");
    expect(lastDoneText({ last_done_at: "2025-10-02T09:00:00-03:00" })).toBe("Feita pela última vez em 02/10/2025");
    expect(lastDoneText({ last_done_at: null })).toBeNull();
    expect(lastDoneText({})).toBeNull();
  });
});

describe("condições e frases", () => {
  it("quatro opções, sem 'none'", () => {
    expect(CONDITION_OPTIONS.map((o) => o.value)).toEqual([ "diabetes", "hypertension", "pregnancy", "mental_health" ]);
    expect(CONDITION_OPTIONS.map((o) => o.label)).toEqual([
      "Tenho diabetes", "Tenho pressão alta", "Estou grávida", "Faço acompanhamento de saúde mental"
    ]);
  });

  it("feature_disabled tem a frase do acompanhamento; o resto segue o canal", () => {
    expect(careMessage(new ApiError(403, "feature_disabled"))).toBe(CARE_DISABLED);
    expect(careMessage(new ApiError(422, "invalid_condition"))).toBe("Essa opção não está mais disponível. Atualize a tela.");
    expect(careMessage(new ApiError(401, "unauthenticated"))).toBe("Sua sessão expirou. Entre de novo com seu celular.");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/care.test.ts`
Expected: FAIL — `./care` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/care.ts
// Regras de tela do acompanhamento do cidadão (módulo 23, F-23.7 e F-23.8;
// spec §6 "Cidadão"; contrato §7). Linguagem simples, sem código nem
// diagnóstico além do nome da linha. Quem decide o que a pessoa vê (só linhas
// inscritas, só par validado, oferta vencida) é o api. Nada daqui escreve em
// console ou URL.
import { ApiError, type CareItem, type DeclaredCondition } from "./citizenApi";
import { fmtCalendarDate } from "./format";
import { messageFor } from "../modules/citizen/ui";

export const CARE_NOT_VERIFIED =
  "Para ver seu acompanhamento por aqui, valide seu cadastro na unidade de saúde com um documento com foto.";
export const CARE_DISABLED = "O acompanhamento ainda não está disponível nesta cidade.";
export const CARE_EMPTY =
  "Nada para acompanhar por aqui ainda. Quando a equipe da sua unidade incluir você num acompanhamento, o que está para fazer aparece aqui.";
export const CONDITIONS_HELP =
  "Opcional. Ajuda a equipe da sua unidade a cuidar de você. Não é diagnóstico, e você pode tirar quando quiser.";

export const CONDITION_OPTIONS: readonly { value: Exclude<DeclaredCondition, "none">; label: string }[] = [
  { value: "diabetes", label: "Tenho diabetes" },
  { value: "hypertension", label: "Tenho pressão alta" },
  { value: "pregnancy", label: "Estou grávida" },
  { value: "mental_health", label: "Faço acompanhamento de saúde mental" }
];

export type CareTone = "ok" | "warn" | "neutral";

const capitalize = (s: string) => (s ? s.charAt(0).toUpperCase() + s.slice(1) : s);

// Situação em frase; código desconhecido não aparece cru (o item fica, sem a situação).
export function careItemView(item: CareItem): { title: string; status: string | null; tone: CareTone } {
  const title = capitalize(item.label.trim()) || "Cuidado da linha";
  switch (item.status) {
    case "ok": return { title, status: "Em dia", tone: "ok" };
    case "due_soon": {
      const day = fmtCalendarDate(item.due_on);
      return { title, status: day === "—" ? "Está chegando a hora" : `Faça até ${day}`, tone: "warn" };
    }
    case "overdue": return { title, status: "Está na hora — procure a sua unidade de saúde", tone: "warn" };
    case "untracked": return { title, status: "Confira com a sua equipe de saúde", tone: "neutral" };
    default: return { title, status: null, tone: "neutral" };
  }
}

// Oferta do catálogo ou triagem vencida de "Meu acompanhamento" (L9).
export function lastDoneText(entry: { last_done_at?: string | null }): string | null {
  const day = fmtCalendarDate((entry.last_done_at ?? "").slice(0, 10));
  return day === "—" ? null : `Feita pela última vez em ${day}`;
}

export function careMessage(error: unknown): string {
  if (error instanceof ApiError && error.code === "feature_disabled") return CARE_DISABLED;
  if (error instanceof ApiError && error.code === "invalid_condition") return "Essa opção não está mais disponível. Atualize a tela.";
  return messageFor(error);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/care.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/care.ts src/lib/care.test.ts
$G commit -m "feat: add the citizen care follow-up screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: "Está na hora de refazer" na tela de triagens (F-23.8)

**Files:**
- Modify: `src/modules/citizen/CatalogStep.tsx`
- Test: `src/modules/citizen/CatalogStep.overdue.test.tsx`

**Interfaces:**
- Consumes: `CatalogEntry.overdue`, `CatalogEntry.last_done_at` (Task 1); `lastDoneText` (Task 2).
- Produces: a seção `catalog-overdue` ("Está na hora de refazer") antes de "Sugeridas para você"; as ofertas vencidas saem de "Disponíveis" (não aparecem duas vezes).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/CatalogStep.overdue.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { CatalogStep } from "./CatalogStep";
import { citizenApi, type Catalog } from "../../lib/citizenApi";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-22T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const IDOSO = { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: "Avaliação anual de quedas, memória e medicamentos.",
  overdue: true, last_done_at: "2025-10-02" };
const RESP = { protocol_name: "triage-respiratoria", title: "Sintomas respiratórios", summary: null, overdue: false, last_done_at: null };
const catalog = (available: Catalog["available"]): Catalog =>
  ({ in_progress: null, suggested: [], available, recent: [], reference_units: [] });

function setup(c: Catalog) {
  vi.spyOn(citizenApi, "catalog").mockResolvedValue(c);
  const onStart = vi.fn<(name: string) => Promise<string | null>>().mockResolvedValue(null);
  render(<CatalogStep citizenId="p1" onStart={onStart} onProfileRequired={vi.fn()} onProfile={vi.fn()} onHistory={vi.fn()} onBack={vi.fn()} />);
  return { onStart };
}

describe("CatalogStep — triagem periódica vencida (módulo 23)", () => {
  it("vencida aparece em 'Está na hora de refazer', com a última vez, e começa pelo mesmo botão", async () => {
    const { onStart } = setup(catalog([ IDOSO, RESP ]));
    const overdue = await screen.findByRole("region", { name: "Está na hora de refazer" });
    const item = within(overdue).getByText("Saúde do idoso").closest("li") as HTMLElement;
    expect(within(item).getByText("Feita pela última vez em 02/10/2025")).toBeInTheDocument();
    const available = screen.getByRole("region", { name: "Disponíveis" });
    expect(within(available).queryByText("Saúde do idoso")).not.toBeInTheDocument();
    expect(within(available).getByText("Sintomas respiratórios")).toBeInTheDocument();
    await userEvent.click(within(item).getByRole("button", { name: "Começar: Saúde do idoso" }));
    expect(onStart).toHaveBeenCalledWith("saude-do-idoso");
  });

  it("sem overdue (api antigo): Disponíveis como hoje", async () => {
    setup(catalog([ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: null } ]));
    const available = await screen.findByRole("region", { name: "Disponíveis" });
    expect(within(available).getByText("Saúde do idoso")).toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Está na hora de refazer" })).not.toBeInTheDocument();
  });

  it("vencida sem data: sem a linha da data, nada de undefined", async () => {
    setup(catalog([ { ...IDOSO, last_done_at: null } ]));
    const overdue = await screen.findByRole("region", { name: "Está na hora de refazer" });
    expect(within(overdue).queryByText(/Feita pela última vez/)).not.toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
    expect(screen.queryByRole("region", { name: "Disponíveis" })).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/CatalogStep.overdue.test.tsx`
Expected: FAIL — não há a seção "Está na hora de refazer".

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/citizen/CatalogStep.tsx`:

1. Troque `import { ReferenceUnits } from "../ReferenceUnits";` por

```ts
import { ReferenceUnits } from "../ReferenceUnits";
import { lastDoneText } from "../../lib/care";
```

2. Troque

```ts
  const c = catalog;
  const nothingToStart = c !== null && c.in_progress === null && c.suggested.length === 0 && c.available.length === 0;
```

por

```ts
  const c = catalog;
  const nothingToStart = c !== null && c.in_progress === null && c.suggested.length === 0 && c.available.length === 0;
  // Módulo 23 (F-23.8): oferta com o intervalo de repetição vencido vem antes,
  // com a última vez; o resto segue em "Disponíveis". Sem `overdue`, como hoje.
  const due = c ? c.available.filter(a => a.overdue === true) : [];
  const rest = c ? c.available.filter(a => a.overdue !== true) : [];
```

3. Antes de `      {c && c.suggested.length > 0 && (`, acrescente

```tsx
      {due.length > 0 && (
        <Section id="catalog-overdue" title="Está na hora de refazer">
          {due.map(a => {
            const last = lastDoneText(a);
            return <Item key={a.protocol_name} title={a.title} summary={a.summary} lines={last ? [ last ] : []}
              action={startButton(a.protocol_name, a.title)} />;
          })}
        </Section>
      )}
```

4. Troque

```tsx
      {c && c.available.length > 0 && (
        <Section id="catalog-available" title="Disponíveis">
          {c.available.map(a => (
```

por

```tsx
      {rest.length > 0 && (
        <Section id="catalog-available" title="Disponíveis">
          {rest.map(a => (
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen/CatalogStep.overdue.test.tsx src/modules/citizen/CatalogStep.test.tsx src/modules/citizen/Flow.catalog.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos do catálogo não mandam `overdue`: "Disponíveis" igual).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/CatalogStep.tsx src/modules/citizen/CatalogStep.overdue.test.tsx
$G commit -m "feat: show overdue periodic screenings first in the citizen catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Condições de saúde declaradas no perfil (F-23.7)

**Files:**
- Create: `src/modules/citizen/HealthConditions.tsx`
- Modify: `src/modules/citizen/ProfileStep.tsx`
- Test: `src/modules/citizen/HealthConditions.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.declaredConditions`, `declareCondition`, `withdrawCondition`, `ApiError` (Task 1); `CONDITION_OPTIONS`, `CONDITIONS_HELP`, `careMessage` (Task 2); `Toggle`, `ErrorText` (`ui.tsx`); `hasSessionFeature` (`sessionFeatures.ts`).
- Produces: `HealthConditions({ citizenId })`; o `ProfileStep` conferido no posto a mostra com `care_lines` na sessão.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/HealthConditions.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { HealthConditions } from "./HealthConditions";
import { ProfileStep } from "./ProfileStep";
import { CARE_DISABLED, CONDITIONS_HELP } from "../../lib/care";
import { ApiError, citizenApi, type Person } from "../../lib/citizenApi";
import { setSessionFeatures } from "../../lib/sessionFeatures";

afterEach(() => { vi.restoreAllMocks(); setSessionFeatures(null); });

const DIABETES = { condition: "diabetes", declared_at: "2026-10-20T10:00:00-03:00" };
const PRESSURE = { condition: "hypertension", declared_at: "2026-10-22T09:00:00-03:00" };
const VERIFIED: Person = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified", neighborhood: null,
  profile: { birth_date: "1963-04-02", sex: "female", gender_identity: null, profile_source: "verified" } };

describe("HealthConditions", () => {
  it("mostra as ativas ligadas, com a explicação de que não é diagnóstico", async () => {
    vi.spyOn(citizenApi, "declaredConditions").mockResolvedValue([ DIABETES ]);
    render(<HealthConditions citizenId="p1" />);
    expect(await screen.findByRole("switch", { name: "Tenho diabetes" })).toBeChecked();
    expect(screen.getByRole("switch", { name: "Tenho pressão alta" })).not.toBeChecked();
    expect(screen.getByText(CONDITIONS_HELP)).toBeInTheDocument();
    expect(citizenApi.declaredConditions).toHaveBeenCalledWith("p1");
  });

  it("ligar declara e reflete a lista do api", async () => {
    vi.spyOn(citizenApi, "declaredConditions").mockResolvedValue([ DIABETES ]);
    const declare = vi.spyOn(citizenApi, "declareCondition").mockResolvedValue([ DIABETES, PRESSURE ]);
    render(<HealthConditions citizenId="p1" />);
    await userEvent.click(await screen.findByRole("switch", { name: "Tenho pressão alta" }));
    expect(declare).toHaveBeenCalledWith("p1", "hypertension");
    expect(await screen.findByRole("status")).toHaveTextContent("Informação salva.");
    expect(screen.getByRole("switch", { name: "Tenho pressão alta" })).toBeChecked();
  });

  it("desligar retira e diz que retirou", async () => {
    vi.spyOn(citizenApi, "declaredConditions").mockResolvedValue([ DIABETES, PRESSURE ]);
    const withdraw = vi.spyOn(citizenApi, "withdrawCondition").mockResolvedValue([ PRESSURE ]);
    render(<HealthConditions citizenId="p1" />);
    await userEvent.click(await screen.findByRole("switch", { name: "Tenho diabetes" }));
    expect(withdraw).toHaveBeenCalledWith("p1", "diabetes");
    expect(await screen.findByRole("status")).toHaveTextContent("Informação retirada.");
    expect(screen.getByRole("switch", { name: "Tenho diabetes" })).not.toBeChecked();
    expect(screen.getByRole("switch", { name: "Tenho pressão alta" })).toBeChecked();
  });

  it("recusa do api: a frase e o interruptor como estava", async () => {
    vi.spyOn(citizenApi, "declaredConditions").mockResolvedValue([]);
    vi.spyOn(citizenApi, "declareCondition").mockRejectedValue(new ApiError(422, "invalid_condition"));
    render(<HealthConditions citizenId="p1" />);
    await userEvent.click(await screen.findByRole("switch", { name: "Estou grávida" }));
    expect(await screen.findByRole("alert")).toHaveTextContent("Essa opção não está mais disponível. Atualize a tela.");
    expect(screen.getByRole("switch", { name: "Estou grávida" })).not.toBeChecked();
  });

  it("par declarado (403 not_verified) ou interruptor desligado: a seção some", async () => {
    vi.spyOn(citizenApi, "declaredConditions").mockRejectedValueOnce(new ApiError(403, "not_verified"));
    const { unmount } = render(<HealthConditions citizenId="p1" />);
    await waitFor(() => expect(screen.queryByRole("heading", { name: "Condições de saúde" })).not.toBeInTheDocument());
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
    unmount();
    vi.spyOn(citizenApi, "declaredConditions").mockRejectedValueOnce(new ApiError(403, "feature_disabled"));
    render(<HealthConditions citizenId="p1" />);
    await waitFor(() => expect(screen.queryByRole("heading", { name: "Condições de saúde" })).not.toBeInTheDocument());
    expect(screen.queryByText(CARE_DISABLED)).not.toBeInTheDocument();
  });
});

describe("ProfileStep — condições de saúde", () => {
  it("perfil conferido com care_lines: a seção aparece abaixo do texto do posto", async () => {
    setSessionFeatures([ "care_lines" ]);
    vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ VERIFIED ] });
    vi.spyOn(citizenApi, "declaredConditions").mockResolvedValue([]);
    render(<ProfileStep citizenId="p1" required={false} onSaved={vi.fn()} onBack={vi.fn()} />);
    expect(await screen.findByRole("heading", { name: "Condições de saúde" })).toBeInTheDocument();
    expect(await screen.findByRole("switch", { name: "Faço acompanhamento de saúde mental" })).toBeInTheDocument();
  });

  it("sem care_lines na sessão: o perfil conferido é o de hoje", async () => {
    setSessionFeatures([ "exam_tracking" ]);
    vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ VERIFIED ] });
    const load = vi.spyOn(citizenApi, "declaredConditions");
    render(<ProfileStep citizenId="p1" required={false} onSaved={vi.fn()} onBack={vi.fn()} />);
    expect(await screen.findByText("Data de nascimento")).toBeInTheDocument();
    expect(screen.queryByRole("heading", { name: "Condições de saúde" })).not.toBeInTheDocument();
    expect(load).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/HealthConditions.test.tsx`
Expected: FAIL — `./HealthConditions` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/citizen/HealthConditions.tsx
// Condições de saúde declaradas (módulo 23, F-23.7; spec, decisões do
// coordenador; contrato §7): o próprio cidadão diz, opcional e retirável;
// NUNCA é diagnóstico. Só par validado (403 not_verified some com a seção) e
// só com care_lines (quem decide mostrar é o ProfileStep). Cada interruptor
// grava na hora; a lista que vale é a que o api devolve. A condição vai só no
// corpo — nunca na URL.
import { useEffect, useRef, useState } from "react";
import { ApiError, citizenApi, type DeclaredCondition, type DeclaredConditionEntry } from "../../lib/citizenApi";
import { CONDITIONS_HELP, CONDITION_OPTIONS, careMessage } from "../../lib/care";
import { ErrorText, Toggle } from "./ui";

const HIDDEN = new Set([ "not_verified", "feature_disabled" ]);
const activeOf = (list: DeclaredConditionEntry[]) => new Set(list.map((c) => c.condition));

export function HealthConditions({ citizenId }: { citizenId: string }) {
  const [ active, setActive ] = useState<Set<string> | null>(null);
  const [ hidden, setHidden ] = useState(false);
  const [ busy, setBusy ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const inFlight = useRef(false);

  useEffect(() => {
    let alive = true;
    citizenApi.declaredConditions(citizenId)
      .then((list) => { if (alive) setActive(activeOf(list)); })
      .catch((e) => {
        if (!alive) return;
        if (e instanceof ApiError && HIDDEN.has(e.code)) setHidden(true);
        else setError(careMessage(e));
      });
    return () => { alive = false; };
  }, [ citizenId ]);

  async function toggle(condition: Exclude<DeclaredCondition, "none">, on: boolean) {
    if (inFlight.current) return;
    inFlight.current = true;
    setBusy(true); setNotice(null); setError(null);
    try {
      const list = on ? await citizenApi.declareCondition(citizenId, condition) : await citizenApi.withdrawCondition(citizenId, condition);
      setActive(activeOf(list));
      setNotice(on ? "Informação salva." : "Informação retirada.");
    } catch (e) {
      if (e instanceof ApiError && HIDDEN.has(e.code)) setHidden(true);
      else setError(careMessage(e));
    } finally {
      inFlight.current = false;
      setBusy(false);
    }
  }

  if (hidden) return null;
  return (
    <section aria-labelledby="health-conditions" style={{ marginTop: 24 }}>
      <h2 id="health-conditions" style={{ fontSize: 20, margin: "0 0 8px" }}>Condições de saúde</h2>
      <p style={{ margin: "0 0 12px", color: "var(--ink2)" }}>{CONDITIONS_HELP}</p>
      {active === null && !error && <p>Carregando…</p>}
      {active && CONDITION_OPTIONS.map((o) => (
        <Toggle key={o.value} label={o.label} checked={active.has(o.value)} disabled={busy}
          onChange={(on) => void toggle(o.value, on)} />
      ))}
      {notice && <p role="status">{notice}</p>}
      {error && <ErrorText>{error}</ErrorText>}
    </section>
  );
}
```

Em `src/modules/citizen/ProfileStep.tsx`:

1. Troque `import { ProfileForm } from "./ProfileForm";` por

```ts
import { ProfileForm } from "./ProfileForm";
import { HealthConditions } from "./HealthConditions";
import { hasSessionFeature } from "../../lib/sessionFeatures";
```

2. Troque

```tsx
        <p style={{ padding: 12, borderRadius: 12, border: "1px solid var(--line)" }}>{VERIFIED_PROFILE_TEXT}</p>
```

por

```tsx
        <p style={{ padding: 12, borderRadius: 12, border: "1px solid var(--line)" }}>{VERIFIED_PROFILE_TEXT}</p>
        {/* Módulo 23 (F-23.7): condição declarada, só para par validado e com care_lines. */}
        {hasSessionFeature("care_lines") && <HealthConditions citizenId={person.id} />}
```

(O perfil só declarado não ganha a seção: a condição declarada é de par validado — W2.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen/HealthConditions.test.tsx src/modules/citizen/ProfileStep.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/HealthConditions.tsx src/modules/citizen/HealthConditions.test.tsx src/modules/citizen/ProfileStep.tsx
$G commit -m "feat: let a verified citizen declare and withdraw health conditions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: A tela "Meu acompanhamento"

**Files:**
- Create: `src/modules/citizen/MyCareStep.tsx`
- Test: `src/modules/citizen/MyCareStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.care`, `ApiError`, `CitizenCare` (Task 1); `CARE_EMPTY`, `CARE_NOT_VERIFIED`, `careItemView`, `careMessage`, `lastDoneText` (Task 2); `Screen`, `BigButton`, `ErrorText` (`ui.tsx`).
- Produces: `MyCareStep({ citizenId, onBack, onValidate(citizenId) })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyCareStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyCareStep } from "./MyCareStep";
import { ApiError, citizenApi, type CitizenCare } from "../../lib/citizenApi";
import { CARE_DISABLED, CARE_EMPTY, CARE_NOT_VERIFIED } from "../../lib/care";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-22T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const CARE: CitizenCare = { lines: [ { name: "Pessoa com diabetes", items: [
  { label: "consulta com a equipe", status: "ok", due_on: "2027-01-06" },
  { label: "medir a pressão", status: "due_soon", due_on: "2026-11-12" },
  { label: "exame de hemoglobina glicada", status: "overdue", due_on: "2026-08-20" },
  { label: "visita do agente de saúde", status: "untracked", due_on: null }
] } ], overdue_triages: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", last_done_at: "2025-10-02" } ] };

function setup(care: CitizenCare | Error = CARE) {
  const careSpy = vi.spyOn(citizenApi, "care");
  if (care instanceof Error) careSpy.mockRejectedValue(care); else careSpy.mockResolvedValue(care);
  const onValidate = vi.fn();
  render(<MyCareStep citizenId="p1" onBack={vi.fn()} onValidate={onValidate} />);
  return { careSpy, onValidate };
}

describe("MyCareStep", () => {
  it("par validado: cada item em linguagem simples, pela situação", async () => {
    const { careSpy } = setup();
    const line = await screen.findByRole("region", { name: "Pessoa com diabetes" });
    expect(careSpy).toHaveBeenCalledWith("p1");
    const item = (name: string) => within(line).getByRole("listitem", { name });
    expect(within(item("Consulta com a equipe")).getByText("Em dia")).toBeInTheDocument();
    expect(within(item("Medir a pressão")).getByText("Faça até 12/11/2026")).toBeInTheDocument();
    expect(within(item("Exame de hemoglobina glicada")).getByText("Está na hora — procure a sua unidade de saúde")).toBeInTheDocument();
    expect(within(item("Visita do agente de saúde")).getByText("Confira com a sua equipe de saúde")).toBeInTheDocument();
  });

  it("triagem periódica vencida: o aviso com a última vez", async () => {
    setup();
    const screenings = await screen.findByRole("region", { name: "Triagens para refazer" });
    const item = within(screenings).getByRole("listitem", { name: "Saúde do idoso" });
    expect(within(item).getByText("Feita pela última vez em 02/10/2025")).toBeInTheDocument();
  });

  it("sem triagem vencida: sem o aviso, o resto aparece", async () => {
    setup({ ...CARE, overdue_triages: [] });
    expect(await screen.findByRole("region", { name: "Pessoa com diabetes" })).toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Triagens para refazer" })).not.toBeInTheDocument();
    expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  });

  it("nada de código, CID ou motivo: só os campos do contrato", async () => {
    const leaky = { overdue_triages: [], lines: [ { name: "Pessoa com diabetes", reason: "problem", ended_reason: "moved", suggestion: "age_sex",
      items: [ { label: "medir a pressão", status: "brand_new", due_on: null, ciap: "T90", cid10: "E11" } ] } ] };
    setup(leaky as unknown as CitizenCare);
    const line = await screen.findByRole("region", { name: "Pessoa com diabetes" });
    expect(within(line).getByRole("listitem", { name: "Medir a pressão" })).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/T90|E11|problem|moved|age_sex|brand_new/);
  });

  it("par declarado (403 not_verified): só o convite, mesmo se o api mandar linhas depois", async () => {
    const { onValidate } = setup(new ApiError(403, "not_verified"));
    expect((await screen.findByRole("note", { name: "cadastro não validado" })).textContent).toBe(CARE_NOT_VERIFIED);
    expect(screen.queryByRole("listitem")).not.toBeInTheDocument();
    expect(screen.queryByRole("region", { name: "Triagens para refazer" })).not.toBeInTheDocument();
    await userEvent.click(screen.getByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("sem linhas: a frase de vazio, sem sugerir nada", async () => {
    setup({ lines: [], overdue_triages: [] });
    expect(await screen.findByText(CARE_EMPTY)).toBeInTheDocument();
  });

  it("interruptor desligado com a tela aberta: a frase do acompanhamento", async () => {
    setup(new ApiError(403, "feature_disabled"));
    expect(await screen.findByRole("alert")).toHaveTextContent(CARE_DISABLED);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/MyCareStep.test.tsx`
Expected: FAIL — `./MyCareStep` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/citizen/MyCareStep.tsx
// "Meu acompanhamento" (módulo 23, F-23.7 e F-23.8; spec §6 "Cidadão";
// contrato §7). Só par validado: o declarado vê o convite para validar. As
// linhas em que a pessoa foi inscrita e o que está para fazer, em linguagem
// simples — nunca código, sugestão, motivo de encerramento ou diagnóstico
// além do nome da linha (só os campos do contrato são desenhados). O aviso de
// triagem periódica vencida vem na mesma resposta (`overdue_triages`, L9).
import { useEffect, useState, type CSSProperties } from "react";
import { ApiError, citizenApi, type CitizenCare } from "../../lib/citizenApi";
import { CARE_EMPTY, CARE_NOT_VERIFIED, careItemView, careMessage, lastDoneText, type CareTone } from "../../lib/care";
import { BigButton, ErrorText, Screen } from "./ui";

const TONE_BG: Record<CareTone, string> = { ok: "var(--ok-bg)", warn: "var(--warn-bg)", neutral: "var(--sunken)" };

export function MyCareStep({ citizenId, onBack, onValidate }:
  { citizenId: string; onBack: () => void; onValidate: (citizenId: string) => void }) {
  const [ data, setData ] = useState<CitizenCare | null>(null);
  const [ notVerified, setNotVerified ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  useEffect(() => {
    // StrictMode monta duas vezes em dev: a resposta da montagem desfeita não escreve.
    let alive = true;
    citizenApi.care(citizenId)
      .then((d) => { if (alive) setData(d); })
      .catch((e) => {
        if (!alive) return;
        if (e instanceof ApiError && e.code === "not_verified") setNotVerified(true);
        else setError(careMessage(e));
      });
    return () => { alive = false; };
  }, [ citizenId ]);

  return (
    <Screen title="Meu acompanhamento" footer={<BigButton variant="secondary" onClick={onBack}>Voltar</BigButton>}>
      {error && <ErrorText>{error}</ErrorText>}
      {!data && !error && !notVerified && <p role="status">Carregando…</p>}
      {notVerified && (
        <div style={{ display: "grid", gap: 12 }}>
          <p role="note" aria-label="cadastro não validado" style={warnNote}>{CARE_NOT_VERIFIED}</p>
          <BigButton onClick={() => onValidate(citizenId)}>Validar no posto</BigButton>
        </div>
      )}
      {data && !notVerified && (
        <div style={{ display: "grid", gap: 24 }}>
          {data.overdue_triages.length > 0 && (
            <section aria-labelledby="care-screenings">
              <h2 id="care-screenings" style={h2}>Triagens para refazer</h2>
              <ul style={listStyle}>
                {data.overdue_triages.map((e) => {
                  const last = lastDoneText(e);
                  return (
                    <li key={e.protocol_name} aria-label={e.title || "Triagem"} style={{ ...card, background: TONE_BG.warn }}>
                      <strong>{e.title || "Triagem"}</strong>
                      <div>Está na hora de refazer esta triagem.</div>
                      {last && <div style={muted}>{last}</div>}
                    </li>
                  );
                })}
              </ul>
              <p style={muted}>Faça pela tela de triagens desta pessoa.</p>
            </section>
          )}
          {data.lines.length === 0 && <p>{CARE_EMPTY}</p>}
          {data.lines.map((line, i) => (
            <section key={`${line.name}-${i}`} aria-labelledby={`care-line-${i}`}>
              <h2 id={`care-line-${i}`} style={h2}>{line.name || "Acompanhamento"}</h2>
              <ul style={listStyle}>
                {line.items.map((item, j) => {
                  const view = careItemView(item);
                  return (
                    <li key={`${view.title}-${j}`} aria-label={view.title} style={{ ...card, background: TONE_BG[view.tone] }}>
                      <strong>{view.title}</strong>
                      {view.status && <div>{view.status}</div>}
                    </li>
                  );
                })}
              </ul>
            </section>
          ))}
        </div>
      )}
    </Screen>
  );
}

const h2: CSSProperties = { fontSize: 20, margin: "0 0 8px" };
const listStyle: CSSProperties = { listStyle: "none", padding: 0, margin: 0, display: "grid", gap: 12 };
const card: CSSProperties = { border: "1px solid var(--rule2)", borderRadius: 12, padding: 12, display: "grid", gap: 4 };
const muted: CSSProperties = { color: "var(--ink2)", margin: 0 };
const warnNote: CSSProperties = { background: "var(--warn-bg)", padding: 12, borderRadius: 12, margin: 0 };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen/MyCareStep.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS (a guarda de tema aceita: só papéis do tema, sem reserva).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/MyCareStep.tsx src/modules/citizen/MyCareStep.test.tsx
$G commit -m "feat: add the citizen care follow-up screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: O botão em "Minhas triagens" e o estado `care` no `Flow`

**Files:**
- Modify: `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `README.md`
- Test: `src/modules/citizen/Flow.care.test.tsx`

**Interfaces:**
- Consumes: `MyCareStep` (Task 5); `hasSessionFeature` (`sessionFeatures.ts`).
- Produces: `HistoryStep` com a prop opcional `onCare?: () => void` (botão **Meu acompanhamento**); `Flow` com o estado `{ at: "care"; consentVersion: string | null; citizenId: string }`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/Flow.care.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi, ApiError } from "../../lib/citizenApi";
import { setSessionFeatures } from "../../lib/sessionFeatures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-22T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); setSessionFeatures(null); window.history.replaceState(null, "", "/"); });

function signedIn(features?: string[]) {
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
  vi.spyOn(citizenApi, "care").mockRejectedValue(new ApiError(403, "not_verified"));
  vi.spyOn(citizenApi, "catalog").mockRejectedValue(new ApiError(409, "profile_required"));
  vi.spyOn(citizenApi, "issueVerificationCode").mockResolvedValue({ code: "482913", expires_at: "2026-10-22T10:10:00-03:00" });
}

async function toMyTriages() {
  render(<Flow />);
  await userEvent.click(await screen.findByRole("button", { name: "Concordo" }));
  await userEvent.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
  expect(await screen.findByRole("heading", { name: "Minhas triagens" })).toBeInTheDocument();
}

describe("Flow — Meu acompanhamento (módulo 23)", () => {
  it("com care_lines: Minhas triagens → Meu acompanhamento → Voltar → Validar no posto", async () => {
    signedIn([ "care_lines" ]);
    await toMyTriages();
    await userEvent.click(screen.getByRole("button", { name: "Meu acompanhamento" }));
    expect(await screen.findByRole("heading", { name: "Meu acompanhamento" })).toBeInTheDocument();
    expect(citizenApi.care).toHaveBeenCalledWith("p1");
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { name: "Minhas triagens" })).toBeInTheDocument();

    await userEvent.click(screen.getByRole("button", { name: "Meu acompanhamento" }));
    await userEvent.click(await screen.findByRole("button", { name: "Validar no posto" }));
    expect(await screen.findByText("482913")).toBeInTheDocument();
  });

  it("com os três interruptores: os três botões", async () => {
    signedIn([ "patient_timeline", "exam_tracking", "care_lines" ]);
    await toMyTriages();
    for (const name of [ "Meu histórico", "Meus exames", "Meu acompanhamento" ]) expect(screen.getByRole("button", { name })).toBeInTheDocument();
  });

  it("sem care_lines na sessão: Minhas triagens como hoje", async () => {
    signedIn([ "exam_tracking" ]);
    await toMyTriages();
    expect(screen.queryByRole("button", { name: "Meu acompanhamento" })).not.toBeInTheDocument();
    expect(citizenApi.care).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/citizen/Flow.care.test.tsx`
Expected: FAIL — não há o botão "Meu acompanhamento".

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/citizen/HistoryStep.tsx`, troque

```tsx
export function HistoryStep({ citizenId, onBack, onValidate, onCheckIn, onAppointmentCheckIn, onRecord, onExams }:
```

por

```tsx
export function HistoryStep({ citizenId, onBack, onValidate, onCheckIn, onAppointmentCheckIn, onRecord, onExams, onCare }:
```

troque

```tsx
    onExams?: () => void }) {
```

por

```tsx
    onExams?: () => void;
    // Módulo 23: "Meu acompanhamento", só com care_lines ligado (quem decide é o Flow).
    onCare?: () => void }) {
```

e, depois da linha `{onExams && <BigButton variant="secondary" onClick={onExams} style={{ marginBottom: 16 }}>Meus exames</BigButton>}`, acrescente

```tsx
          {onCare && <BigButton variant="secondary" onClick={onCare} style={{ marginBottom: 16 }}>Meu acompanhamento</BigButton>}
```

Em `src/modules/citizen/Flow.tsx`:

1. Troque `import { MyExamsStep } from "./MyExamsStep";` por

```ts
import { MyExamsStep } from "./MyExamsStep";
import { MyCareStep } from "./MyCareStep";
```

2. Troque

```ts
  | { at: "exams"; consentVersion: string | null; citizenId: string }
```

por

```ts
  | { at: "exams"; consentVersion: string | null; citizenId: string }
  | { at: "care"; consentVersion: string | null; citizenId: string }
```

3. No `case "history":`, troque

```tsx
        onExams={hasSessionFeature("exam_tracking")
          ? () => setState({ at: "exams", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined} />;
```

por

```tsx
        onExams={hasSessionFeature("exam_tracking")
          ? () => setState({ at: "exams", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined}
        onCare={hasSessionFeature("care_lines")
          ? () => setState({ at: "care", consentVersion: state.consentVersion, citizenId: state.citizenId })
          : undefined} />;
```

4. Antes de `    case "verify-code":`, acrescente

```tsx
    case "care":
      view = <MyCareStep key={state.citizenId} citizenId={state.citizenId}
        onBack={() => setState({ at: "history", citizenId: state.citizenId, consentVersion: state.consentVersion })}
        onValidate={id => setState({ at: "verify-code", citizenId: id, consentVersion: state.consentVersion })} />;
      break;
```

Em `README.md`, depois da linha `a entrada **Exame** baixa o PDF pela mesma rota do exame.` (fim do parágrafo de "Meus exames"), acrescente uma linha em branco e:

```markdown
**Linhas de cuidado (módulo 23).** Com `care_lines` ligado na cidade (vem em
`features` da sessão), "Minhas triagens" ganha o botão **Meu acompanhamento**
e o perfil conferido no posto ganha **Condições de saúde** — diabetes, pressão
alta, gravidez e acompanhamento de saúde mental —, opcionais, retiráveis e
nunca diagnóstico (a condição vai só no corpo, nunca na URL). Par validado: as
linhas em que a pessoa foi inscrita e o que está para fazer, em linguagem
simples, sem código, sugestão ou motivo. Par só declarado: o convite para
validar no posto. A tela de triagens mostra **Está na hora de refazer** para a
oferta com intervalo de repetição vencido (`overdue`, do catálogo do módulo 15,
inscrito ou não), e "Meu acompanhamento" repete o aviso.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/citizen && npx tsc --noEmit`
Expected: PASS (`Flow.exams.test.tsx` e `Flow.history.test.tsx` seguem iguais: sem `care_lines`, nenhum botão novo).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx src/modules/citizen/Flow.care.test.tsx README.md
$G commit -m "feat: open the care follow-up from the citizen triage history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Perfil, acompanhamento e triagem vencida no Playwright (perfis de celular)

**Files:**
- Create: `e2e/care.spec.ts`

**Interfaces:**
- Consumes: as Tasks 1–6; `playwright.config.ts` (5 perfis; `WPDA_URL`).
- Produces: o e2e do módulo 23, verde nos perfis de celular e pulado nos outros (o fluxo só abre no celular).

- [ ] **Step 1: Escreva o e2e**

```ts
// e2e/care.spec.ts
// Módulo 23 nos perfis de celular. As rotas /citizen/* são respondidas aqui
// (sem api): o que se prova é a tela real no navegador do aparelho — o perfil
// com as condições declaradas (o nome da condição só no corpo), "Meu
// acompanhamento" e a triagem periódica vencida, sem rolagem lateral.
import { expect, test, type Page, type Route } from "@playwright/test";

const PHONES = new Set([ "iphone", "pixel" ]);
const PERSON = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified", neighborhood: null,
  profile: { birth_date: "1963-04-02", sex: "female", gender_identity: null, profile_source: "verified" } };
const CATALOG = { in_progress: null, suggested: [], recent: [], reference_units: [],
  available: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: "Avaliação anual de quedas, memória e medicamentos.",
    overdue: true, last_done_at: "2025-10-02" } ] };
const CARE = { overdue_triages: [ { protocol_name: "saude-do-idoso", title: "Saúde do idoso", last_done_at: "2025-10-02" } ],
  lines: [ { name: "Pessoa com diabetes", items: [
  { label: "medir a pressão", status: "due_soon", due_on: "2026-11-12" },
  { label: "exame de hemoglobina glicada", status: "overdue", due_on: "2026-08-20" },
  { label: "visita do agente de saúde", status: "untracked", due_on: null }
] } ] };

type Sent = { method: string; url: string; body: unknown };

async function mockApi(page: Page, sent: Sent[]) {
  let active = [ { condition: "diabetes", declared_at: "2026-10-20T10:00:00-03:00" } ];
  await page.route((url) => url.pathname.startsWith("/citizen/"), async (route: Route) => {
    const req = route.request();
    const url = new URL(req.url());
    const path = url.pathname.replace(/^\/citizen/, "");
    const json = (body: unknown, status = 200) => route.fulfill({ status, contentType: "application/json", body: JSON.stringify(body) });
    switch (path) {
      case "/session": return json({ phone_masked: "(**) *****-5432", time_zone: "America/Sao_Paulo", features: [ "care_lines" ] });
      case "/consent_term": return json({ version: "1", body: "Termo de teste" });
      case "/people": return json({ people: [ PERSON ] });
      case "/neighborhoods": return json({ neighborhoods: [] });
      case "/notices": return json({ notices: [], unread_count: 0 });
      case "/appointments": return json({ appointments: [] });
      case "/triages": return json({ citizen: { ...PERSON, verified_at: "2026-09-01T10:00:00-03:00" }, triages: [] });
      case "/people/p1/catalog": return json(CATALOG);
      case "/care": return json(CARE);
      case "/declared_conditions": {
        const body = req.postData() ? JSON.parse(req.postData() as string) : null;
        sent.push({ method: req.method(), url: req.url(), body });
        if (req.method() === "POST") active = [ ...active, { condition: body.condition, declared_at: "2026-10-22T10:00:00-03:00" } ];
        if (req.method() === "DELETE") active = active.filter((c) => c.condition !== body.condition);
        return json({ conditions: active });
      }
      default: return json({ error: "not_found" }, 404);
    }
  });
}

const noOverflow = (page: Page) => page.evaluate(() => document.documentElement.scrollWidth - document.documentElement.clientWidth);

test.beforeEach(({}, info) => {
  test.skip(!PHONES.has(info.project.name), "o fluxo do cidadão só abre no celular (device-gate.spec.ts)");
});

test("triagem vencida e condições de saúde no perfil, sem rolagem lateral", async ({ page }) => {
  const sent: Sent[] = [];
  await mockApi(page, sent);
  await page.goto("/wpda/");
  await page.getByRole("button", { name: "Concordo", exact: true }).click();
  await page.getByRole("button", { name: `CPF ${PERSON.cpf_masked}` }).click();

  const overdue = page.getByRole("region", { name: "Está na hora de refazer" });
  await expect(overdue.getByText("Saúde do idoso", { exact: true })).toBeVisible();
  await expect(overdue.getByText("Feita pela última vez em 02/10/2025", { exact: true })).toBeVisible();
  expect(await noOverflow(page)).toBeLessThanOrEqual(0);

  await page.getByRole("button", { name: "Meu perfil" }).click();
  await expect(page.getByRole("heading", { name: "Condições de saúde" })).toBeVisible();
  await expect(page.getByRole("switch", { name: "Tenho diabetes" })).toBeChecked();
  await page.getByRole("switch", { name: "Tenho pressão alta" }).click();
  await expect(page.getByRole("status")).toHaveText("Informação salva.");
  await page.getByRole("switch", { name: "Tenho diabetes" }).click();
  await expect(page.getByRole("status")).toHaveText("Informação retirada.");
  await expect(page.getByRole("switch", { name: "Tenho diabetes" })).not.toBeChecked();

  const writes = sent.filter((s) => s.method !== "GET");
  expect(writes.map((s) => [ s.method, s.body ])).toEqual([
    [ "POST", { condition: "hypertension" } ], [ "DELETE", { condition: "diabetes" } ]
  ]);
  for (const s of sent) expect(s.url).not.toMatch(/diabetes|hypertension/);
  expect(await noOverflow(page)).toBeLessThanOrEqual(0);
});

test("Meu acompanhamento em linguagem simples, sem rolagem lateral", async ({ page }) => {
  await mockApi(page, []);
  await page.goto("/wpda/");
  await page.getByRole("button", { name: "Concordo", exact: true }).click();
  await page.getByRole("button", { name: `Ver triagens de ${PERSON.cpf_masked}` }).click();
  await page.getByRole("button", { name: "Meu acompanhamento" }).click();
  await expect(page.getByRole("heading", { level: 1, name: "Meu acompanhamento" })).toBeVisible();

  const line = page.getByRole("region", { name: "Pessoa com diabetes" });
  await expect(line.getByRole("listitem", { name: "Medir a pressão" }).getByText("Faça até 12/11/2026")).toBeVisible();
  await expect(line.getByRole("listitem", { name: "Exame de hemoglobina glicada" })
    .getByText("Está na hora — procure a sua unidade de saúde")).toBeVisible();
  await expect(line.getByRole("listitem", { name: "Visita do agente de saúde" }).getByText("Confira com a sua equipe de saúde")).toBeVisible();
  await expect(page.getByRole("region", { name: "Triagens para refazer" }).getByRole("listitem", { name: "Saúde do idoso" })).toBeVisible();
  expect(await noOverflow(page)).toBeLessThanOrEqual(0);
});
```

- [ ] **Step 2: Suba o Vite do worktree na 5199 e rode**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod23 --network rota-saude_default -p 5199:5199 \
  -v $R/.worktrees/wpda-mod23:/app -v wpda-mod23-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3045 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5199 --host 0.0.0.0"
docker logs -f wpda-mod23   # espere o "ready"; Ctrl+C sai do log, o container segue
cd $R/.worktrees/wpda-mod23 && WPDA_URL=http://curitiba.localhost:5199 npx playwright test e2e/care.spec.ts e2e/device-gate.spec.ts
```

Expected: `care.spec.ts` verde nos perfis `iphone` e `pixel` (pulado nos outros três); `device-gate.spec.ts` verde nos 5. (O e2e responde as rotas `/citizen/*`: o api do 23 não precisa estar no ar para esta task.) O container fica no ar para a Task 8.

- [ ] **Step 3: Commit**

```bash
cd $W
$G add e2e/care.spec.ts
$G commit -m "test: cover the citizen care follow-up on phone profiles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Suíte, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–7 na branch `feat/mod-23-care-lines`; o api do 23 na **3045** com a semente do 23; o container `wpda-mod23` da Task 7.
- Produces: branch pronta para merge (depois do merge do api do 23); cards F-23.7 e F-23.8 atualizados.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5199 npx playwright test`
Expected: tudo verde: a base anotada na Task 0 mais **7 arquivos de teste novos** (`citizenApi.care`, `care`, `CatalogStep.overdue`, `HealthConditions`, `MyCareStep`, `Flow.care` e — para o vitest — nenhum do `e2e/`, que fica excluído); o e2e nos 5 perfis. Um número de testes que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio (o `dist/` do build está no `.gitignore`) e contagem `0`.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage" src/lib/care.ts src/modules/citizen/HealthConditions.tsx src/modules/citizen/MyCareStep.tsx
sed -n '/─── Linhas de cuidado (módulo 23/,/^export interface CitizenSession/p' src/lib/citizenApi.ts | grep -n 'condition=\|\${condition}\|ciap\|cid10\|reason'
grep -n "ciap\|cid10\|reason\|suggestion" src/modules/citizen/MyCareStep.tsx
```

Expected: os três sem saída.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do wpda contra a spec (§2 decisões 5 e 8 e as do coordenador, §3, §6 "Cidadão", §7, §10 F-23.7 e F-23.8), o ADR 0040 e o contrato §7. Pontos de atenção:
- par declarado: nenhuma linha em "Meu acompanhamento", nenhuma condição no perfil; 403 `not_verified` vira o convite (ou some a seção);
- só os campos do contrato aparecem; situação desconhecida não aparece crua; `untracked` em frase própria;
- a condição só no corpo; `DELETE` com corpo; 204 relê; o wpda nunca manda `none`;
- o aviso de triagem vencida depende só de `overdue` do catálogo (sem `care_lines`, ele continua na tela de triagens);
- sem `care_lines` na sessão, perfil e "Minhas triagens" são os de hoje;
- interface do canal: componentes existentes, alvos de 48 px, cores por papel do tema, guarda verde, e2e de celular verde.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 23 na porta **3045** (plano do api: banco de dev migrado e semeado **com autorização do usuário**; Curitiba em `record`, `clinical_record` e `care_lines` ligados; um celular com uma pessoa **validada** inscrita na linha "Pessoa com diabetes" — consulta em dia, HbA1c vencida — e com uma oferta de triagem com intervalo de repetição vencido; uma pessoa **só declarada** no mesmo celular). O container `wpda-mod23` da Task 7 continua no ar na **5199**, com o proxy para `http://api:3045`.

Abra `http://curitiba.localhost:5199/wpda/` no celular de verdade ou no modo de dispositivo do navegador (375 px, toque ligado: o fluxo só abre no celular). O usuário entra com o celular da semente e o código do SMS de dev, como nas provas do canal web. Confira com screenshot (salve em `evidencias/2026-10-22-modulo-23/wpda/`):
- pessoa **validada** → triagens: **Está na hora de refazer** com a triagem vencida e "Feita pela última vez em …"; tocar **Começar** abre a triagem;
- **Meu perfil** → **Condições de saúde**: ligar "Tenho pressão alta" → "Informação salva."; no dashboard (prova do plano do dashboard), a sugestão por condição declarada aparece para a equipe depois do job; desligar → "Informação retirada." (o api grava `withdrawn_at`);
- **Minhas triagens → Meu acompanhamento**: a linha "Pessoa com diabetes" com a consulta "Em dia", a HbA1c "Está na hora — procure a sua unidade de saúde" e a visita do ACS "Confira com a sua equipe de saúde"; o aviso da triagem vencida; nada de código, CID ou motivo;
- pessoa **só declarada** → Meu acompanhamento: o convite e "Validar no posto" → o código do posto; o perfil dela não tem "Condições de saúde";
- no maintenance (`http://maintenance.localhost:5177`, login do usuário), desligar `care_lines` em Curitiba, sair e entrar de novo no wpda: sem o botão e sem a seção do perfil, e a tela de triagens **continua** com "Está na hora de refazer" (depende do módulo 15); religar;
- a barra de acessibilidade em alto contraste: as telas legíveis, sem cor fixa;
- a 375 px, nenhuma rolagem lateral.

Depois: `docker rm -f wpda-mod23 && docker volume rm wpda-mod23-node-modules`. Deixe os interruptores e as condições de teste como o plano do api pedir (ou registre o que mudou na semente de dev).

- [ ] **Step 5: Cards**

Se o lado api de F-23.7 e de F-23.8 já estiver **Done** no board, mova os dois para **Done** (um `gh project item-edit` por card, como na Task 0, com o id da opção "Done", `98236657`), e para **Verified** se a prova do Step 4 passou:

```bash
gh project item-edit --id <PVTI_do_F-23.7> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id 98236657
```

Senão, deixe em In Progress e avise a sessão do api.

- [ ] **Step 6:** **Pare.** O merge do wpda só vem depois do merge do api do 23, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: api → dashboard e wpda → maintenance → catálogo (contrato §10). Antes do push, confira `origin/main..main` no wpda e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-23-care-lines
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/wpda-mod23
/opt/homebrew/bin/git branch -d feat/mod-23-care-lines
```

Depois do merge, confira o `wpda-dev` (5176): ele lê `apps/wpda` e recompila sozinho.

---

## Divergências propostas ao contrato

Alinhadas ao plano do api (`2026-10-22-module-23-care-lines-api.md`, L1–L14). **Adotada** = a forma do plano do api, já usada aqui; **aberta** = falta ou discordo.

1. **W1 — Condições declaradas. Adotada (L11), com uma discordância.** `GET`/`POST`/`DELETE /citizen/declared_conditions` respondem `{ conditions: [ { condition, declared_at } ] }` (só as vigentes); o `POST` leva `{ citizen_id, condition }` no corpo (201). **Discordo** do `DELETE /citizen/declared_conditions?citizen_id=&condition=`: a condição é dado de saúde e iria para o log de requisições e do proxy. O wpda manda `DELETE /citizen/declared_conditions?citizen_id=` com `{ "condition" }` no **corpo** (com `Content-Type: application/json`); o controller do api lê `params[:condition]`, que o Rails monta também a partir do corpo JSON — então funciona sem mudar o api, mas proponho que o contrato fixe o corpo e o spec do api teste assim. O cliente também aceita 204 e relê a lista.
2. **W2 — Só par validado declara. Adotada (L11):** 403 `not_verified`; o wpda só mostra a seção no perfil conferido no posto e a esconde no 403.
3. **W3 — Interruptor na sessão do cidadão. Aberta (conferir).** O wpda decide pelo `features` de `GET /citizen/session` (como `exam_tracking`, 21a E2). O plano do api não cita a sessão do cidadão; se ela já lista toda chave ligada e utilizável pelo mecanismo genérico, `care_lines` aparece sozinho — a Task 0 (Step 4) e a prova conferem; se não, o api precisa incluí-la.
4. **W4 — Triagem vencida. Adotada (L9):** `overdue` e `last_done_at` em toda oferta com `retake_after_days`, sempre (o módulo 15 não tem interruptor); "Meu acompanhamento" usa `overdue_triages: [ { protocol_name, title, last_done_at } ]` da própria resposta.
5. **W5 — `none`. Adotada (L11), sem tela própria:** `none` retira as outras e uma condição retira o `none`. O wpda não oferece "nenhuma" (desligar cada uma basta); se o api devolver `none` na lista, nenhuma opção aparece ligada.
6. **W6 — Itens do acompanhamento. Adotada (L1):** `label` do item é o `citizen_label` da boa prática (ex.: "Medir a pressão"); `status` com os códigos do §1; `due_on` `AAAA-MM-DD`. Código novo de situação não aparece ao cidadão.

## Self-review

- **Cobertura (spec §2 decisões 5 e 8 e as do coordenador, §3, §6 "Cidadão", §7, §8; contrato §7):** condição declarada no perfil, opcional e retirável, nunca diagnóstico, só par validado — Tasks 1 e 4 (W1, W2); "Meu acompanhamento" só para par validado, linguagem simples, sem código nem motivo — Tasks 1, 2 e 5; o botão em "Minhas triagens" e o interruptor — Task 6 (W3); aviso de triagem periódica vencida na tela de triagens (módulo 15, inscrito ou não) — Tasks 1 e 3 (W4); o aviso repetido em "Meu acompanhamento" — Task 5; regras do wpda (temas, `isPhone`, Playwright) — Global Constraints, guardas nas Tasks 4, 5 e 8, e2e na Task 7; prova no navegador — Task 8. O cidadão nunca vê sugestão nem motivo de encerramento: o cliente descarta e a tela não desenha (Task 5).
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); dados da semente do api (Task 8).
- **Tipos e nomes:** `DeclaredCondition`, `DeclaredConditionEntry`, `CareItem`, `CareLineView`, `OverdueTriage`, `CitizenCare` e os campos opcionais de `CatalogEntry` (Task 1) são os usados nas Tasks 2–5; `careItemView`, `lastDoneText`, `careMessage`, `CONDITION_OPTIONS`, `CONDITIONS_HELP`, `CARE_*` (Task 2) batem com os usos; `HealthConditions({ citizenId })` (Task 4), `MyCareStep({ citizenId, onBack, onValidate })` (Task 5) e `HistoryStep.onCare` (Task 6) são os que o `Flow` usa; `Toggle` de `ui.tsx` tem `label`, `checked`, `disabled` e `onChange(checked)`.
- **Review Focus:** 1 — Task 5 ("par declarado (403 not_verified): só o convite, mesmo se o api mandar linhas depois"); 2 — Task 2 ("careItemView: tabela das situações; desconhecida não aparece crua") e Task 5 ("nada de código, CID ou motivo: só os campos do contrato"); 3 — Task 1 ("withdrawCondition: DELETE com a condição no corpo; 204 relê a lista") e Task 4 ("desligar retira e diz que retirou"); 4 — Task 1 ("catalog: overdue false e last_done_at null quando faltam; true preservado"), Task 3 ("sem overdue (api antigo): Disponíveis como hoje" e "vencida sem data: sem a linha da data, nada de undefined"); 5 — Task 4 ("sem care_lines na sessão: o perfil conferido é o de hoje"), Task 5 ("interruptor desligado com a tela aberta: a frase do acompanhamento") e Task 6 ("sem care_lines na sessão: Minhas triagens como hoje").
