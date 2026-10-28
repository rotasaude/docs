# Módulo 26a — Regulação municipal (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 26a** rodando na branch dele (`feat/mod-26a-care-regulation`, worktree `apps/api/.claude/mod26a`, porta **3048**) com `GET /citizen/regulation_requests` (contrato §9), `care_regulation` em `features` da sessão do cidadão e o `kind: regulated` em `/citizen/appointments` (Divergências W1–W7 deste plano). Ele é **mergeado antes** deste (contrato §12: api → dashboard e wpda → maintenance → catálogo). As Tasks 0–5 não dependem dele (os testes usam dublês e o Playwright responde as rotas); a prova da Task 6 depende.

**Goal:** Lado wpda de F-26.7: a tela **"Meus encaminhamentos"**, só para par validado, com cada pedido de regulação da pessoa — o que foi pedido (`label`), a **situação** em linguagem simples (aguardando, agendado com **data e local**, devolvido à unidade, não autorizado, encerrado, realizado) e a data da última mudança —, **nunca** a cor de risco, a posição na fila, o CID ou a justificativa; o botão em "Minhas triagens"; e o texto do pedido de agendamento `regulated` do módulo 17 em "Seus agendamentos".

**Architecture:** O cliente (`src/lib/citizenApi.ts`) ganha `regulationRequests(citizenId)`, com a normalização na borda (só os campos do contrato §9 sobrevivem; o resto é descartado), e o `kind` do pedido de agendamento ganha `regulated`. As regras de tela ficam puras em `src/lib/referrals.ts` (frase de cada situação, data e local, frases de recusa). Peça nova: `src/modules/citizen/MyReferralsStep.tsx`. Mudam o mínimo: `HistoryStep` (o botão "Meus encaminhamentos"), `Flow` (o estado `referrals`), `AppointmentsSection` (o texto do `regulated`), `README.md`. Quem vê o quê é do api (par validado, só os pedidos da pessoa, sem rascunho); a tela nunca deduz nem calcula posição.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `setupFiles` com `cleanup` e o dublê de celular `stubDevice`), `vi.spyOn(citizenApi, …)`, Playwright (5 perfis de dispositivo, `e2e/`), Vite 5 (base `/wpda/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-28-module-26a-care-regulation-design.md` (§1 antiobjetivos — "mostrar ao cidadão uma posição que muda com o risco dos outros" —, §2 decisão 9 (opção A do usuário), §4 "wpda", §5, §6, §8 F-26.7), `docs/adr/0044.md` (invariante "o cidadão nunca vê a posição na fila") e o contrato `docs/superpowers/plans/2026-10-28-module-26a-care-regulation-contracts.md` (§1, §9, §11, §12; as diferenças estão em "Divergências propostas ao contrato", W1–W7). Herdados: ADR 0017 (par CPF + celular, validação presencial) e os planos do wpda dos módulos 21a e 23 (invariantes do cidadão).

## Global Constraints

- Rota (contrato §9, na forma das Divergências W1–W4): `GET /citizen/regulation_requests?citizen_id=` → `{ "requests": [ { "id", "label", "status": "waiting"|"scheduled"|"returned"|"denied"|"closed"|"completed", "scheduled_at"|null, "place"|null, "updated_at" } ] }`; 403 `not_verified` para par só declarado; 403 `feature_disabled` (`feature: "care_regulation"`) com o interruptor desligado; 401 dispara `citizen:unauthenticated`, como hoje.
- **O cidadão nunca vê a posição na fila, a cor de risco, o CID, a justificativa nem motivo** (ADR 0044, invariante; spec decisão 9): o cliente descarta todo campo fora do contrato (`color`, `suggested_color`, `position`, `cid10`, `justification`, `reason`, `due_on`, `overdue`, `events`…) e a tela só desenha `label`, a frase da situação, a data e o local e "Atualizado em". Nada de "prazo", "fila", "posição", "vermelho/amarelo/verde/azul" ou "urgente" em texto da tela.
- **Situação desconhecida nunca aparece crua** (como `AppointmentsSection`): vira "Em andamento".
- **Só par validado**: par declarado (ou 403 `not_verified`) vê só o convite para validar no posto e "Validar no posto", nenhum pedido, mesmo que o api mande algum.
- Interruptor (spec decisão 12): sem `care_regulation` em `session.features`, "Minhas triagens" não tem "Meus encaminhamentos" — exatamente como hoje. Se o interruptor cair com a tela aberta, o 403 `feature_disabled` vira `REFERRALS_DISABLED` ("Os encaminhamentos ainda não aparecem por aqui nesta cidade.").
- Na URL vai só o id da pessoa (`citizen_id`) — nunca CPF, celular, rótulo ou texto. Nada em `localStorage`, `sessionStorage` ou `console`. Sem polling: a lista lê ao abrir a tela.
- Interface do canal (ciclo de interface, subprojetos 4 e 6e): `Screen`, `BigButton`, `ErrorText` de `ui.tsx`; alvos de toque de 48 px ou mais, texto de 18 px; cores **só por papel do tema** (`var(--warn-bg)`, `var(--accent-bg)`, `var(--ok-bg)`, `var(--rule2)`, `var(--ink2)`), sem reserva `var(--x, …)` e sem branco fixo — a guarda `src/theme/guards.test.ts` roda na suíte; o fluxo do cidadão só abre no celular (`isPhone`, `src/lib/device.ts`) e o `e2e/device-gate.spec.ts` segue verde nos 5 perfis. Nenhum componente comum muda. Achado só visual não vira tarefa.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` e `vi.useRealTimers()` no `afterEach`; a suíte roda em `TZ=America/Sao_Paulo`.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-26.1..F-26.7** do board #1 para **In Progress** (Task 0); o wpda é o último app de F-26.7 (com o api): ele vai a **Done** quando o api dele estiver Done e este plano terminar, e a **Verified** depois da prova no navegador (Task 6).

## Ambiente de execução

- Worktree **`/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/wpda-mod26a`** (nunca dentro de `apps/wpda`: o compose monta `./apps/wpda:/app` e um segundo React derruba o `wpda-dev`), branch `feat/mod-26a-care-regulation` a partir de `origin/main`, com `npm ci` (Task 0). Caminhos de arquivo das tasks relativos a ele.
- Variáveis usadas nos comandos: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod26a`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test wpda -w wpda-mod26a <arquivos>` (o runner único; o log fica em `.test-runs/`). Tipos: `cd $W && npx tsc --noEmit`. e2e: `cd $W && WPDA_URL=http://curitiba.localhost:5204 npx playwright test` (precisa do Vite da Task 5 no ar).
- Ambiente de teste: `vitest.config.ts` com `environment: "jsdom"`, `globals: false`, `setupFiles: ["./src/test/setup.ts"]` (jest-dom, `cleanup`, `sessionStorage.clear()` e o celular padrão `stubDevice()`), `exclude` de `e2e/**` e `TZ=America/Sao_Paulo`.
- O api do 26a roda em dev na **3048**, dentro do container `api-dev` (rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5204**, com `node_modules` num volume próprio (o `node_modules` do worktree é de macOS). `/citizen` já está no proxy do `vite.config.ts`: nada muda nele. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cor, posição, CID ou justificativa vazando para o cidadão.** Um api que mande `color`, `position`, `cid10`, `justification`, `due_on` ou `events` não pode fazer a tela mostrar nada disso: o cliente descarta e a tela não desenha; nenhuma palavra de fila ou cor aparece. Testes: Task 1, "regulationRequests: só os campos do contrato sobrevivem"; Task 3, "nada de cor, posição, CID ou justificativa na tela".
2. **Par só declarado abre "Meus encaminhamentos".** 403 `not_verified` (ou api que mande pedidos para par declarado) → só o convite e "Validar no posto". Teste: Task 3, "par declarado (403 not_verified): só o convite".
3. **Situação nova ou interna do api** (`cancelled`, `external`, `pending`, código desconhecido). Espera-se "Em andamento", nunca o código cru nem "undefined". Testes: Task 2, "referralView: tabela das situações; desconhecida vira Em andamento".
4. **Agendado sem local, ou com data inválida.** Espera-se "Agendado para <data> — a unidade informa o local" sem local; sem data válida, "Agendado — a unidade informa a data e o local"; nunca "null", "undefined" ou "Invalid Date". Teste: Task 2, "agendado: com e sem local, data inválida".
5. **Interruptor desligado ou caindo com a tela aberta.** Sem `care_regulation` na sessão, nem o botão; com a tela aberta e 403 `feature_disabled`, a frase dos encaminhamentos. Testes: Task 3, "interruptor desligado com a tela aberta: a frase dos encaminhamentos"; Task 4, "sem care_regulation na sessão: Minhas triagens como hoje".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-26.1..F-26.7, conferência do main e do api do 26a, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.referrals.test.ts` | `regulationRequests` e o `kind: regulated` | 1 |
| `src/lib/referrals.ts`, `src/lib/referrals.test.ts` | regras de tela dos encaminhamentos | 2 |
| `src/modules/citizen/MyReferralsStep.tsx`, `src/modules/citizen/MyReferralsStep.test.tsx` | a tela "Meus encaminhamentos" (F-26.7) | 3 |
| `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/Flow.referrals.test.tsx`, `src/modules/citizen/AppointmentsSection.tsx`, `src/modules/citizen/AppointmentsSection.test.tsx`, `README.md` | o botão, o estado `referrals` e o `regulated` em "Seus agendamentos" | 4 |
| `e2e/referrals.spec.ts` | "Meus encaminhamentos" nos perfis de celular | 5 |
| — | suíte, build, e2e, revisão e prova no navegador | 6 |

**Estratégia de teste:** cliente com `fetch` falso conferindo URL (só o id da pessoa), o envelope e a normalização que descarta campo fora do contrato (Task 1); regras puras com tabela de casos: cada situação em frase, data e local presentes ou faltando, situação desconhecida (Task 2); a tela com par validado, par declarado, campo a mais, lista vazia e interruptor caindo (Task 3); a navegação real pelo `Flow` com a sessão com e sem o interruptor, e o texto do `regulated` em "Seus agendamentos" (Task 4); o Playwright nos perfis de celular, com as rotas respondidas pelo teste, para a tela real no navegador, a largura de 375–412 px sem rolagem lateral e nenhuma palavra de fila ou cor (Task 5). A prova final é no navegador, contra o api do 26a (Task 6).

---

### Task 0: Cards, conferência do main e do api do 26a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`7c46249` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `apps/api/.claude/mod26a`.
- Produces: cards F-26.1..F-26.7 em In Progress (ou além); worktree `$W` na branch `feat/mod-26a-care-regulation`; a base anotada; a forma da rota do cidadão anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-26\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os sete itens em In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira as âncoras e o api do 26a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda fetch -q origin
$G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '  kind: "return" | "referral" | "triage" | "care_line" | "screening";\|^  care: async (citizenId: string): Promise<CitizenCare> =>\|^async function call<T>'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/HistoryStep.tsx | grep -n '    onCare?: () => void }) {\|          {onCare && <BigButton variant="secondary" onClick={onCare}'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n '  | { at: "care"; consentVersion: string | null; citizenId: string }\|        onCare={hasSessionFeature("care_lines")\|    case "care":'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/AppointmentsSection.tsx | grep -n '    case "care_line":\|    // Tipo que esta versão não conhece: nunca o código cru.'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^\*\*Linhas de cuidado (módulo 23).\*\*\|^### Relatório público'
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod26a api bin/rails routes 2>/dev/null | grep -E "citizen/(regulation_requests|appointments|session)"
```

Expected: as âncoras e as rotas. Diferença de forma da rota do cidadão (envelope, nomes, situações) vira ajuste no `normalizeReferral` e nas fixtures das Tasks 1–5 **antes** de começá-las, com nota para a Task 6. Se a sessão do cidadão não listar `care_regulation` (W5), avise a sessão do api.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod26a -b feat/mod-26a-care-regulation origin/main
cd $R/.worktrees/wpda-mod26a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod26a
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 6 espera **+4 arquivos de teste** (`citizenApi.referrals`, `referrals`, `MyReferralsStep`, `Flow.referrals`) e casos novos em `AppointmentsSection.test.tsx`.

---

### Task 1: Cliente — `regulationRequests` e o `kind: regulated`

**Files:**
- Modify: `src/lib/citizenApi.ts`
- Test: `src/lib/citizenApi.referrals.test.ts`

**Interfaces:**
- Consumes: `call` (existente em `citizenApi.ts`).
- Produces: tipos `ReferralStatus = "waiting" | "scheduled" | "returned" | "denied" | "closed" | "completed"`, `CitizenReferral { id: string; label: string; status: ReferralStatus | string; scheduled_at: string | null; place: string | null; updated_at: string }`; `citizenApi.regulationRequests(citizenId: string): Promise<CitizenReferral[]>`; `AppointmentRequest.kind` com `"regulated"`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.referrals.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function stubFetch(status: number, body: unknown) {
  const fetchMock = vi.fn(async () => new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}

describe("citizenApi.regulationRequests (contrato §9)", () => {
  it("só o id da pessoa na URL; GET sem corpo", async () => {
    const fetchMock = stubFetch(200, { requests: [] });
    expect(await citizenApi.regulationRequests("p1")).toEqual([]);
    const [ url, init ] = fetchMock.mock.calls[0] as unknown as [ string, RequestInit ];
    expect(url).toBe("/citizen/regulation_requests?citizen_id=p1");
    expect(init.method).toBe("GET");
    expect(init.body).toBeUndefined();
  });

  it("regulationRequests: só os campos do contrato sobrevivem", async () => {
    stubFetch(200, { requests: [ {
      id: "rr1", label: "Consulta em Cardiologia", status: "scheduled", scheduled_at: "2026-11-10T14:30:00-03:00",
      place: "Ambulatório de Especialidades — Rua XV, 100", updated_at: "2026-10-27T10:00:00-03:00",
      color: "red", suggested_color: "yellow", position: 3, cid10: "I209", justification: "Dor torácica", due_on: "2026-11-04",
      overdue: true, events: [ { status: "pending", reason: "x" } ], reason: "x"
    } ] });
    expect(await citizenApi.regulationRequests("p1")).toEqual([ {
      id: "rr1", label: "Consulta em Cardiologia", status: "scheduled", scheduled_at: "2026-11-10T14:30:00-03:00",
      place: "Ambulatório de Especialidades — Rua XV, 100", updated_at: "2026-10-27T10:00:00-03:00"
    } ]);
  });

  it("campos faltando viram nulo ou vazio; envelope ausente é lista vazia", async () => {
    stubFetch(200, { requests: [ { id: "rr2", status: "waiting" } ] });
    expect(await citizenApi.regulationRequests("p1")).toEqual([
      { id: "rr2", label: "", status: "waiting", scheduled_at: null, place: null, updated_at: "" } ]);
    stubFetch(200, {});
    expect(await citizenApi.regulationRequests("p1")).toEqual([]);
  });

  it("403 not_verified e feature_disabled chegam como ApiError com o código", async () => {
    stubFetch(403, { error: "not_verified" });
    await expect(citizenApi.regulationRequests("p1")).rejects.toEqual(new ApiError(403, "not_verified"));
    stubFetch(403, { error: "feature_disabled", feature: "care_regulation" });
    await expect(citizenApi.regulationRequests("p1")).rejects.toEqual(new ApiError(403, "feature_disabled"));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test wpda -w wpda-mod26a src/lib/citizenApi.referrals.test.ts`
Expected: FAIL — `regulationRequests` não existe.

- [ ] **Step 3: Implemente**

Depois da seção do módulo 23 em `citizenApi.ts`, a seção "Encaminhamentos (módulo 26a, ADR 0044; contrato §9)" com os tipos, `type RawReferral = { id: string } & Partial<Record<"label" | "status" | "scheduled_at" | "place" | "updated_at", string | null>>` e `normalizeReferral(r: RawReferral): CitizenReferral` que monta **só** os seis campos (`label ?? ""`, `status ?? ""`, `scheduled_at ?? null`, `place ?? null`, `updated_at ?? ""`) — o comentário diz que cor, posição, CID e justificativa nunca passam da borda. Em `citizenApi`: `regulationRequests: async (citizenId) => ((await call<{ requests?: RawReferral[] }>("GET", \`/regulation_requests?citizen_id=${encodeURIComponent(citizenId)}\`)).requests ?? []).map(normalizeReferral)`. Em `AppointmentRequest.kind`, acrescente `| "regulated"` com o comentário "Módulo 26a: pedido aberto pela autorização da regulação na rede própria".

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test wpda -w wpda-mod26a src/lib/citizenApi.referrals.test.ts src/lib && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/citizenApi.ts src/lib/citizenApi.referrals.test.ts
$G commit -m "feat: add the citizen referrals client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela dos encaminhamentos (`src/lib/referrals.ts`)

**Files:**
- Create: `src/lib/referrals.ts`
- Test: `src/lib/referrals.test.ts`

**Interfaces:**
- Consumes: `CitizenReferral`, `ApiError` (Task 1); `fmtWeekdayDateTime`, `fmtDate` (`src/lib/format.ts`); `messageFor` (`ui.tsx`).
- Produces: frases `REFERRALS_NOT_VERIFIED = "Para ver seus encaminhamentos por aqui, valide seu cadastro na unidade de saúde com um documento com foto."`, `REFERRALS_DISABLED = "Os encaminhamentos ainda não aparecem por aqui nesta cidade."`, `REFERRALS_EMPTY = "Nenhum encaminhamento a especialista ou procedimento por aqui ainda."`, `REFERRALS_HELP = "Aqui aparece a situação de cada encaminhamento. A data e o local chegam por aqui quando a vaga for marcada."`; `referralView(r: CitizenReferral): { title: string; status: string; detail: string | null; tone: "warn" | "accent" | "ok" | "neutral" }`; `updatedText(r): string | null`; `referralsMessage(e: unknown): string`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/referrals.test.ts
import { describe, it, expect } from "vitest";
import { ApiError, type CitizenReferral } from "./citizenApi";
import { REFERRALS_DISABLED, referralView, referralsMessage, updatedText } from "./referrals";

const ref = (over: Partial<CitizenReferral>): CitizenReferral => ({ id: "rr1", label: "Consulta em Cardiologia", status: "waiting",
  scheduled_at: null, place: null, updated_at: "2026-10-27T10:00:00-03:00", ...over });
const AT = "2026-11-10T14:30:00-03:00";
const when = new Intl.DateTimeFormat("pt-BR", { timeZone: "America/Sao_Paulo", weekday: "short", day: "2-digit", month: "2-digit",
  hour: "2-digit", minute: "2-digit" }).format(new Date(AT));

describe("referralView: tabela das situações; desconhecida vira Em andamento", () => {
  it.each([
    [ "waiting", "Aguardando a regulação", "A Secretaria de Saúde está organizando a vaga. A data e o local chegam por aqui.", "warn" ],
    [ "returned", "Em revisão pela sua unidade", "A sua unidade de saúde está completando o pedido.", "warn" ],
    [ "denied", "Não autorizado", "Procure a sua unidade de saúde para saber os próximos passos.", "neutral" ],
    [ "closed", "Encerrado", "Se ainda precisar, procure a sua unidade de saúde.", "neutral" ],
    [ "completed", "Realizado", null, "ok" ],
    [ "cancelled", "Em andamento", null, "neutral" ],
    [ "external", "Em andamento", null, "neutral" ],
    [ "", "Em andamento", null, "neutral" ]
  ])("%s", (status, text, detail, tone) => {
    const v = referralView(ref({ status }));
    expect(v).toEqual({ title: "Consulta em Cardiologia", status: text, detail, tone });
  });
  it("sem rótulo: Encaminhamento", () => {
    expect(referralView(ref({ label: "" })).title).toBe("Encaminhamento");
  });
});

describe("agendado: com e sem local, data inválida", () => {
  it("data e local", () => {
    expect(referralView(ref({ status: "scheduled", scheduled_at: AT, place: "Ambulatório de Especialidades — Rua XV, 100" })))
      .toEqual({ title: "Consulta em Cardiologia", status: "Agendado", detail: `${when} — Ambulatório de Especialidades — Rua XV, 100`, tone: "accent" });
  });
  it("sem local", () => {
    expect(referralView(ref({ status: "scheduled", scheduled_at: AT })).detail).toBe(`${when} — a unidade informa o local`);
  });
  it("sem data válida", () => {
    for (const at of [ null, "não é data" ]) {
      const v = referralView(ref({ status: "scheduled", scheduled_at: at, place: "Clínica" }));
      expect(v.detail).toBe("A unidade informa a data e o local.");
      expect(JSON.stringify(v)).not.toMatch(/null|undefined|Invalid/);
    }
  });
});

describe("frases e datas", () => {
  it("atualizado em", () => {
    expect(updatedText(ref({}))).toBe("Atualizado em 27/10/2026");
    expect(updatedText(ref({ updated_at: "" }))).toBeNull();
  });
  it("recusas", () => {
    expect(referralsMessage(new ApiError(403, "feature_disabled"))).toBe(REFERRALS_DISABLED);
    expect(referralsMessage(new ApiError(500, "boom"))).not.toMatch(/boom/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test wpda -w wpda-mod26a src/lib/referrals.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente** a tabela do teste. `scheduled`: `fmtWeekdayDateTime(scheduled_at)` (se devolver "—" ou o ISO for inválido, "A unidade informa a data e o local."); `updatedText` por `fmtDate` ("—" → `null`); `referralsMessage`: `feature_disabled` → `REFERRALS_DISABLED`, o resto pelo `messageFor`. Nada aqui menciona cor, prazo, fila ou posição.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test wpda -w wpda-mod26a src/lib/referrals.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/referrals.ts src/lib/referrals.test.ts
$G commit -m "feat: add citizen referral status rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: A tela "Meus encaminhamentos" (F-26.7)

**Files:**
- Create: `src/modules/citizen/MyReferralsStep.tsx`
- Test: `src/modules/citizen/MyReferralsStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.regulationRequests` (Task 1); `referralView`, `updatedText`, `referralsMessage`, `REFERRALS_*` (Task 2); `Screen`, `BigButton`, `ErrorText` (`ui.tsx`).
- Produces: `MyReferralsStep({ citizenId, onBack, onValidate })` — `Screen` "Meus encaminhamentos"; `REFERRALS_HELP` no topo; cada pedido como `<li aria-label={title}>` com o título, a situação, o detalhe (se houver) e "Atualizado em …"; par declarado → `REFERRALS_NOT_VERIFIED` em `role="note"` e "Validar no posto"; vazio → `REFERRALS_EMPTY`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyReferralsStep.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyReferralsStep } from "./MyReferralsStep";
import { citizenApi, ApiError, type CitizenReferral } from "../../lib/citizenApi";
import { REFERRALS_DISABLED, REFERRALS_EMPTY, REFERRALS_NOT_VERIFIED } from "../../lib/referrals";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-28T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const ref = (over: Partial<CitizenReferral>): CitizenReferral => ({ id: "rr", label: "", status: "waiting", scheduled_at: null, place: null,
  updated_at: "2026-10-27T10:00:00-03:00", ...over });
const LIST = [
  ref({ id: "rr1", label: "Consulta em Cardiologia", status: "scheduled", scheduled_at: "2026-11-10T14:30:00-03:00",
    place: "Ambulatório de Especialidades — Rua XV, 100" }),
  ref({ id: "rr2", label: "Ecocardiografia transtorácica", status: "waiting" }),
  ref({ id: "rr3", label: "Consulta em Ortopedia", status: "denied" })
];
const renderStep = (onValidate = vi.fn()) => render(<MyReferralsStep citizenId="p1" onBack={vi.fn()} onValidate={onValidate} />);

describe("MyReferralsStep", () => {
  it("par validado: situação, data e local de cada encaminhamento", async () => {
    const spy = vi.spyOn(citizenApi, "regulationRequests").mockResolvedValue(LIST);
    renderStep();
    const card = await screen.findByRole("listitem", { name: "Consulta em Cardiologia" });
    expect(within(card).getByText("Agendado")).toBeInTheDocument();
    expect(within(card).getByText(/Ambulatório de Especialidades — Rua XV, 100/)).toBeInTheDocument();
    expect(within(screen.getByRole("listitem", { name: "Ecocardiografia transtorácica" })).getByText("Aguardando a regulação")).toBeInTheDocument();
    expect(within(screen.getByRole("listitem", { name: "Consulta em Ortopedia" })).getByText("Não autorizado")).toBeInTheDocument();
    expect(screen.getAllByText("Atualizado em 27/10/2026")).toHaveLength(3);
    expect(spy).toHaveBeenCalledWith("p1");
    expect(spy).toHaveBeenCalledTimes(1);
  });

  it("nada de cor, posição, CID ou justificativa na tela", async () => {
    // Mesmo que o cliente deixasse passar (ele não deixa: Task 1), a tela só desenha os campos do contrato.
    const leaky = { ...LIST[1], color: "red", position: 37, cid10: "I209", justification: "Dor aos esforços há três meses" } as unknown as CitizenReferral;
    vi.spyOn(citizenApi, "regulationRequests").mockResolvedValue([ leaky ]);
    renderStep();
    await screen.findByRole("listitem", { name: "Ecocardiografia transtorácica" });
    expect(document.body.textContent).not.toMatch(/vermelh|amarel|verde|azul|\bred\b|posição|position|fila|I209|esforços|prazo|urgente|\b37\b/i);
  });

  it("par declarado (403 not_verified): só o convite", async () => {
    const onValidate = vi.fn();
    vi.spyOn(citizenApi, "regulationRequests").mockRejectedValue(new ApiError(403, "not_verified"));
    renderStep(onValidate);
    expect(await screen.findByRole("note", { name: "cadastro não validado" })).toHaveTextContent(REFERRALS_NOT_VERIFIED);
    expect(screen.queryByRole("listitem")).toBeNull();
    await userEvent.setup({ advanceTimers: vi.advanceTimersByTime }).click(screen.getByRole("button", { name: "Validar no posto" }));
    expect(onValidate).toHaveBeenCalledWith("p1");
  });

  it("vazio", async () => {
    vi.spyOn(citizenApi, "regulationRequests").mockResolvedValue([]);
    renderStep();
    expect(await screen.findByText(REFERRALS_EMPTY)).toBeInTheDocument();
  });

  it("interruptor desligado com a tela aberta: a frase dos encaminhamentos", async () => {
    vi.spyOn(citizenApi, "regulationRequests").mockRejectedValue(new ApiError(403, "feature_disabled"));
    renderStep();
    expect(await screen.findByText(REFERRALS_DISABLED)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test wpda -w wpda-mod26a src/modules/citizen/MyReferralsStep.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** no molde do `MyExamsStep` (leitura num `useEffect` com `alive`, sem polling; 403 `not_verified` → estado "não validado"; cartão com `border: 1px solid var(--rule2)`; a situação com fundo pelo tom: `warn` → `var(--warn-bg)`, `accent` → `var(--accent-bg)`, `ok` → `var(--ok-bg)`, `neutral` sem fundo). Cada cartão desenha **só** `referralView(r)` e `updatedText(r)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test wpda -w wpda-mod26a src/modules/citizen/MyReferralsStep.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (a guarda de tema verde).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/MyReferralsStep.tsx src/modules/citizen/MyReferralsStep.test.tsx
$G commit -m "feat: add the citizen referrals screen without queue position

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: O botão em "Minhas triagens", o estado `referrals` e o `regulated` em "Seus agendamentos"

**Files:**
- Modify: `src/modules/citizen/HistoryStep.tsx`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/AppointmentsSection.tsx`, `src/modules/citizen/AppointmentsSection.test.tsx`, `README.md`
- Test: `src/modules/citizen/Flow.referrals.test.tsx`

**Interfaces:**
- Consumes: `MyReferralsStep` (Task 3); `hasSessionFeature` (`sessionFeatures.ts`).
- Produces: `HistoryStep` com a prop opcional `onReferrals?: () => void` (botão **Meus encaminhamentos**, depois de "Meu acompanhamento"); `Flow` com o estado `{ at: "referrals"; consentVersion: string | null; citizenId: string }`; `openRequestText` com `case "regulated"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/Flow.referrals.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi } from "../../lib/citizenApi";
import { setSessionFeatures } from "../../lib/sessionFeatures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-10-28T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); setSessionFeatures(null); window.history.replaceState(null, "", "/"); });

function signedIn(features?: string[]) {
  vi.spyOn(citizenApi, "currentSession").mockImplementation(async () => {
    setSessionFeatures(features);
    return { phone_masked: "(**) *****-5432", ...(features ? { features } : {}) };
  });
  vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({
    people: [ { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified", neighborhood: null, profile: null } ]
  });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
  vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
  vi.spyOn(citizenApi, "appointments").mockResolvedValue({ appointments: [] });
  vi.spyOn(citizenApi, "triages").mockResolvedValue({
    citizen: { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified" }, triages: []
  });
  return vi.spyOn(citizenApi, "regulationRequests").mockResolvedValue([
    { id: "rr1", label: "Consulta em Cardiologia", status: "waiting", scheduled_at: null, place: null, updated_at: "2026-10-27T10:00:00-03:00" } ]);
}

describe("Flow — Meus encaminhamentos (módulo 26a)", () => {
  it("com care_regulation: Minhas triagens → Meus encaminhamentos → Voltar", async () => {
    const spy = signedIn([ "care_regulation" ]);
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
    await user.click(await screen.findByRole("button", { name: "Meus encaminhamentos" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Meus encaminhamentos" })).toBeInTheDocument();
    expect(await screen.findByText("Aguardando a regulação")).toBeInTheDocument();
    expect(spy).toHaveBeenCalledWith("p1");
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas triagens" })).toBeInTheDocument();
  });

  it("sem care_regulation na sessão: Minhas triagens como hoje", async () => {
    const spy = signedIn([ "care_lines" ]);
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ver triagens de ***.982.247-**" }));
    await screen.findByRole("heading", { level: 1, name: "Minhas triagens" });
    expect(screen.queryByRole("button", { name: "Meus encaminhamentos" })).toBeNull();
    expect(spy).not.toHaveBeenCalled();
  });
});
```

(Os nomes dos botões "Concordo" e "Ver triagens de …" são os do `Flow.care.test.tsx`; se mudaram no main, use os de lá.)

Em `src/modules/citizen/AppointmentsSection.test.tsx`, na tabela `cases` do describe "texto do pedido aberto por tipo":

```ts
    [ "regulated", "Ambulatório de Especialidades", "Encaminhamento autorizado para Ambulatório de Especialidades — a unidade vai marcar o horário" ],
    [ "regulated", null, "Encaminhamento autorizado. A unidade vai marcar o horário." ],
```

e acrescente `regulated` à expressão do `not.toMatch` (`/null|undefined|care_line|screening|algo_novo|regulated/`).

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test wpda -w wpda-mod26a src/modules/citizen/Flow.referrals.test.tsx src/modules/citizen/AppointmentsSection.test.tsx`
Expected: FAIL — o botão, o estado e o texto não existem.

- [ ] **Step 3: Implemente**

- `HistoryStep`: `onReferrals?: () => void` (comentário "Módulo 26a: 'Meus encaminhamentos', só com care_regulation ligado (quem decide é o Flow)") e o `BigButton` secundário depois do de "Meu acompanhamento".
- `Flow`: o estado `referrals`; no `history`, `onReferrals={hasSessionFeature("care_regulation") ? () => setState({ at: "referrals", … }) : undefined}`; o `case "referrals"` com `MyReferralsStep` (`key={state.citizenId}`, `onBack` volta ao `history`, `onValidate` vai ao `verify-code`), como o `care`.
- `AppointmentsSection.openRequestText`: `case "regulated"` com as duas frases do teste, antes do `default`.
- `README.md`, depois do parágrafo **Linhas de cuidado (módulo 23).**:

```markdown
**Encaminhamentos (módulo 26a).** Com `care_regulation` ligado na cidade (vem
em `features` da sessão), "Minhas triagens" ganha o botão **Meus
encaminhamentos**. Par validado: cada encaminhamento a especialista ou
procedimento pedido na consulta, com a situação em linguagem simples
(aguardando a regulação, agendado com data e local, em revisão pela unidade,
não autorizado, encerrado, realizado). Nunca a cor de risco, a posição na
fila, o CID ou a justificativa: a posição muda com o risco dos outros pedidos
e não é mostrada (ADR 0044). Par só declarado: o convite para validar no
posto. O encaminhamento autorizado na rede própria aparece também em "Seus
agendamentos", como pedido da unidade de destino.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test wpda -w wpda-mod26a src/modules/citizen && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos do `Flow`, do `HistoryStep` e da seção seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/citizen/HistoryStep.tsx src/modules/citizen/Flow.tsx src/modules/citizen/Flow.referrals.test.tsx \
  src/modules/citizen/AppointmentsSection.tsx src/modules/citizen/AppointmentsSection.test.tsx README.md
$G commit -m "feat: link the referrals screen and label regulated appointment requests

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: "Meus encaminhamentos" no Playwright (perfis de celular)

**Files:**
- Create: `e2e/referrals.spec.ts`

**Interfaces:**
- Consumes: as Tasks 1–4; o Vite do worktree na **5204**.
- Produces: a prova nos perfis `iphone` e `pixel` (os outros três pulam com o motivo).

- [ ] **Step 1: Suba o Vite do worktree num container da rede do compose**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod26a --network rota-saude_default -p 5204:5204 \
  -v $R/.worktrees/wpda-mod26a:/app -v wpda-mod26a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3048 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5204 --host 0.0.0.0"
docker logs -f wpda-mod26a   # espere o "ready"; Ctrl+C sai do log
```

- [ ] **Step 2: Write the failing test**

```ts
// e2e/referrals.spec.ts
// Módulo 26a nos perfis de celular. As rotas /citizen/* são respondidas aqui
// (sem api): o que se prova é a tela real no navegador do aparelho — a
// situação, a data e o local, e nada de cor, posição, CID ou justificativa
// (mesmo que a resposta traga), sem rolagem lateral.
import { expect, test, type Page, type Route } from "@playwright/test";

const PHONES = new Set([ "iphone", "pixel" ]);
const PERSON = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "verified", neighborhood: null, profile: null };
// A resposta traz campos que o contrato proíbe mostrar: o cliente descarta, a tela não desenha.
const REQUESTS = { requests: [
  { id: "rr1", label: "Consulta em Cardiologia", status: "scheduled", scheduled_at: "2026-11-10T14:30:00-03:00",
    place: "Ambulatório de Especialidades — Rua XV, 100", updated_at: "2026-10-27T10:00:00-03:00",
    color: "red", position: 2, cid10: "I209", justification: "Dor aos esforços há três meses" },
  { id: "rr2", label: "Ecocardiografia transtorácica", status: "waiting", scheduled_at: null, place: null,
    updated_at: "2026-10-20T10:00:00-03:00", color: "yellow", position: 37 },
  { id: "rr3", label: "Consulta em Ortopedia", status: "returned", scheduled_at: null, place: null, updated_at: "2026-10-22T10:00:00-03:00" }
] };
const APPOINTMENTS = { appointments: [ { request: { id: "ar1", kind: "regulated", target_unit_name: "Ambulatório de Especialidades",
  status: "open", closed_reason: null, reopened_reason: null }, appointment: null } ] };

async function mockApi(page: Page, urls: string[]) {
  await page.route((url) => url.pathname.startsWith("/citizen/"), async (route: Route) => {
    urls.push(route.request().url());
    const path = new URL(route.request().url()).pathname.replace(/^\/citizen/, "");
    const json = (body: unknown, status = 200) => route.fulfill({ status, contentType: "application/json", body: JSON.stringify(body) });
    switch (path) {
      case "/session": return json({ phone_masked: "(**) *****-5432", time_zone: "America/Sao_Paulo", features: [ "care_regulation" ] });
      case "/consent_term": return json({ version: "1", body: "Termo de teste" });
      case "/people": return json({ people: [ PERSON ] });
      case "/neighborhoods": return json({ neighborhoods: [] });
      case "/notices": return json({ notices: [], unread_count: 0 });
      case "/appointments": return json(APPOINTMENTS);
      case "/triages": return json({ citizen: { ...PERSON, verified_at: "2026-09-01T10:00:00-03:00" }, triages: [] });
      case "/regulation_requests": return json(REQUESTS);
      default: return json({ error: "not_found" }, 404);
    }
  });
}

const noOverflow = (page: Page) => page.evaluate(() => document.documentElement.scrollWidth - document.documentElement.clientWidth);

test.beforeEach(({}, info) => {
  test.skip(!PHONES.has(info.project.name), "esta prova roda nos dois celulares (iphone e pixel)");
});

test("Meus encaminhamentos: situação, data e local, sem fila nem cor, sem rolagem lateral", async ({ page }) => {
  const urls: string[] = [];
  await mockApi(page, urls);
  await page.goto("/wpda/");
  await page.getByRole("button", { name: "Concordo", exact: true }).click();
  await page.getByRole("button", { name: `Ver triagens de ${PERSON.cpf_masked}` }).click();
  await expect(page.getByText("Encaminhamento autorizado para Ambulatório de Especialidades — a unidade vai marcar o horário")).toBeVisible();
  await page.getByRole("button", { name: "Meus encaminhamentos" }).click();
  await expect(page.getByRole("heading", { level: 1, name: "Meus encaminhamentos" })).toBeVisible();

  const cardio = page.getByRole("listitem", { name: "Consulta em Cardiologia" });
  await expect(cardio.getByText("Agendado", { exact: true })).toBeVisible();
  await expect(cardio.getByText(/Ambulatório de Especialidades — Rua XV, 100/)).toBeVisible();
  await expect(page.getByRole("listitem", { name: "Ecocardiografia transtorácica" }).getByText("Aguardando a regulação")).toBeVisible();
  await expect(page.getByRole("listitem", { name: "Consulta em Ortopedia" }).getByText("Em revisão pela sua unidade")).toBeVisible();

  const body = await page.locator("body").innerText();
  expect(body).not.toMatch(/vermelh|amarel|posição|fila|I209|esforços|prazo|urgente|\b37\b/i);
  for (const u of urls) {
    const q = new URL(u).search;
    expect(q === "" || /^\?citizen_id=p1$/.test(q)).toBe(true);
  }
  expect(await noOverflow(page)).toBeLessThanOrEqual(0);
});
```

- [ ] **Step 3: Run test to verify it fails, then passes**

Run (antes das Tasks 1–4 na branch, o teste falha; com elas, passa): `cd $W && WPDA_URL=http://curitiba.localhost:5204 npx playwright test e2e/referrals.spec.ts e2e/device-gate.spec.ts`
Expected: PASS nos dois celulares (os outros três perfis pulam o `referrals`), e o `device-gate` verde nos 5. Se o texto "Ver triagens de …" ou "Concordo" mudou, use os do `e2e/care.spec.ts`.

- [ ] **Step 4: Commit**

```bash
cd $W
$G add e2e/referrals.spec.ts
$G commit -m "test: cover the citizen referrals screen on phones

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Suíte, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–5 na branch `feat/mod-26a-care-regulation`; o api do 26a na **3048** com a semente do 26a; o container `wpda-mod26a` da Task 5.
- Produces: branch pronta para merge (depois do merge do api do 26a); card F-26.7 atualizado.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run: `bin/test wpda -w wpda-mod26a && cd $W && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5204 npx playwright test`
Expected: tudo verde: a base anotada na Task 0 mais **4 arquivos de teste novos** (`citizenApi.referrals`, `referrals`, `MyReferralsStep`, `Flow.referrals`; nenhum do `e2e/`, que o vitest exclui) e os casos novos em `AppointmentsSection.test.tsx`; o e2e nos 5 perfis. Um número de testes que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e contagem `0`.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage" src/lib/referrals.ts src/modules/citizen/MyReferralsStep.tsx
grep -n "color\|position\|cid10\|justification\|due_on\|overdue" src/lib/referrals.ts src/modules/citizen/MyReferralsStep.tsx
sed -n '/─── Encaminhamentos (módulo 26a/,/^export interface CitizenSession/p' src/lib/citizenApi.ts | grep -n 'color\|position\|cid10\|justification'
```

Expected: os três sem saída (o comentário da borda no `citizenApi` fica fora do trecho conferido ou é ajustado para não repetir os nomes).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do wpda contra a spec (§1 antiobjetivos, §2 decisão 9, §4 "wpda", §5, §8 F-26.7), o ADR 0044 e o contrato §9. Pontos de atenção:
- nenhuma posição, cor, CID, justificativa, prazo ou motivo chega à tela; o cliente descarta campo fora do contrato;
- par declarado: só o convite; 403 `not_verified` vira o convite;
- situação desconhecida vira "Em andamento", nunca o código;
- sem `care_regulation` na sessão, "Minhas triagens" é a de hoje;
- o `regulated` em "Seus agendamentos" tem frase própria;
- interface do canal: componentes existentes, alvos de 48 px, cores por papel do tema, guarda verde, e2e de celular verde.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 26a na porta **3048** (plano do api: banco de dev migrado e semeado **com autorização do usuário**; Curitiba com `clinical_record` e `care_regulation` ligados; um celular com uma pessoa **validada** com três pedidos — um agendado num prestador externo com data e local, um aguardando, um devolvido — e um autorizado na rede própria com pedido de agendamento aberto; uma pessoa **só declarada** no mesmo celular). O container `wpda-mod26a` da Task 5 continua no ar na **5204**, com o proxy para `http://api:3048`.

Abra `http://curitiba.localhost:5204/wpda/` no celular de verdade ou no modo de dispositivo do navegador (375 px, toque ligado: o fluxo só abre no celular). O usuário entra com o celular da semente e o código do SMS de dev, como nas provas do canal web. Confira com screenshot (salve em `evidencias/2026-10-28-modulo-26a/wpda/`):
- pessoa **validada** → Minhas triagens: "Seus agendamentos" mostra "Encaminhamento autorizado para <unidade> — a unidade vai marcar o horário";
- **Meus encaminhamentos**: o agendado com dia, hora e local; o aguardando; o devolvido "Em revisão pela sua unidade"; nenhuma cor, posição, CID ou justificativa;
- no dashboard (prova do plano do dashboard), o regulador registra o agendamento de um pedido aguardando; sair e voltar à tela no wpda: ele aparece agendado com data e local (sem polling: a tela relê ao abrir);
- pessoa **só declarada** → Meus encaminhamentos: o convite e "Validar no posto";
- no maintenance (`http://maintenance.localhost:5177`, login do usuário), desligar `care_regulation` em Curitiba, sair e entrar de novo no wpda: sem o botão; religar;
- a barra de acessibilidade em alto contraste: a tela legível, sem cor fixa;
- a 375 px, nenhuma rolagem lateral.

Depois: `docker rm -f wpda-mod26a && docker volume rm wpda-mod26a-node-modules`. Deixe os interruptores como o plano do api pedir (ou registre o que mudou na semente de dev).

- [ ] **Step 5: Card**

Se o lado api de F-26.7 já estiver **Done** no board, mova o F-26.7 para **Done** (um `gh project item-edit`, como na Task 0, com o id da opção "Done", `98236657`), e para **Verified** se a prova do Step 4 passou. Senão, deixe em In Progress e avise a sessão do api.

- [ ] **Step 6:** **Pare.** O merge do wpda só vem depois do merge do api do 26a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: api → dashboard e wpda → maintenance → catálogo (contrato §12). Antes do push, confira `origin/main..main` no wpda e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/wpda
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-26a-care-regulation
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/wpda-mod26a
/opt/homebrew/bin/git branch -d feat/mod-26a-care-regulation
```

Depois do merge, confira o `wpda-dev` (5176): ele lê `apps/wpda` e recompila sozinho.

---

## Divergências propostas ao contrato

O contrato §9 é a base; conferidas com o plano do api (`2026-10-28-module-26a-care-regulation-api.md`, "Valores fixados" 2 e Task 13). **Adotada** = a forma do plano do api, já usada aqui; **aberta** = falta ou discordo.

1. **W1 — Situações internas → situação do cidadão. Adotada (`CITIZEN_STATUS`).** `pending`, `authorized` e `external` → `waiting`; `scheduled` → `scheduled`; `returned` → `returned`; `denied` → `denied`; `cancelled` e `closed` → `closed`; `completed` → `completed`; `draft` nunca aparece; `missed` nunca é situação atual (R7). A tela trata qualquer outro valor como "Em andamento".
2. **W2 — Par declarado. Adotada.** 403 `not_verified` (a tela também aceita `{ "verified": false }`, como o 21a, e mostra o convite); `citizen_id` de outro celular → 404 (frase genérica).
3. **W3 — `label`. Aberta.** O plano do api manda o `target_name` cru ("Cardiologia", "Ecocardiografia transtorácica"). Proposta: "Consulta em <especialidade>" para `specialty` e o nome do procedimento para `procedure` — o cidadão entende "Consulta em Cardiologia" melhor que "Cardiologia". A tela desenha o que vier; os testes usam o rótulo proposto e passam com o outro.
4. **W4 — Ordem e tamanho. Adotada.** `updated_at` mais recente primeiro, sem paginação.
5. **W5 — Interruptor na sessão do cidadão. Aberta (conferir).** O wpda decide pelo `features` de `GET /citizen/session` (como `exam_tracking` e `care_lines`); o plano do api não cita a sessão do cidadão. Se ela já lista toda chave ligada e utilizável pelo mecanismo genérico, `care_regulation` aparece sozinho — a Task 0 confere; senão, o api precisa incluí-la. Com o prontuário fora, o 403 vem com `feature: "clinical_record"`; a tela usa a mesma frase.
6. **W6 — `regulated` em `/citizen/appointments`. Adotada (R13).** O pedido de agendamento criado pela autorização na rede própria vem com `kind: "regulated"` e `target_unit_name` = unidade de destino; o wpda tem frase própria para ele.
7. **W7 — `place` e `scheduled_at`. Adotada (Desvio 7 e Task 13 do api).** Só com `scheduled`: na rede própria, o horário do 17 e `place` = nome e endereço da unidade; no prestador externo, o local registrado pelo regulador; fora de `scheduled`, os dois nulos.

## Self-review

- **Cobertura (spec §1 antiobjetivo da posição, §2 decisão 9, §4 "wpda", §5, §8 F-26.7; contrato §9; tarefa "rótulo `regulated` em compromissos do 17 se aparecer ao cidadão"):** situação, data e local sem cor nem posição nem CID — Tasks 1, 2, 3 e 5 (Review Focus 1, 3, 4); só par validado — Task 3 (Review Focus 2; W2); o botão e o interruptor — Task 4 (Review Focus 5; W5); o `regulated` em "Seus agendamentos" — Tasks 1 e 4 (W6); testes de componente e Playwright de celular — Tasks 1–5; regras do wpda (temas, `isPhone`, Playwright) — Global Constraints, guarda nas Tasks 3 e 6, e2e na Task 5; prova no navegador — Task 6.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards (Task 0), a forma real da rota do api (Task 0, com a regra: ajustar a borda antes de começar) e a semente (Task 6).
- **Tipos e nomes:** `CitizenReferral`, `ReferralStatus` e `citizenApi.regulationRequests` (Task 1) são os das Tasks 2–5; `referralView`, `updatedText`, `referralsMessage`, `REFERRALS_*` (Task 2) batem com a Task 3; `MyReferralsStep({ citizenId, onBack, onValidate })` (Task 3) e `HistoryStep.onReferrals` (Task 4) são os que o `Flow` usa; o `kind` `"regulated"` (Task 1) é o da Task 4 e do e2e.
- **Review Focus:** 1 — Task 1 ("regulationRequests: só os campos do contrato sobrevivem") e Task 3 ("nada de cor, posição, CID ou justificativa na tela"); 2 — Task 3 ("par declarado (403 not_verified): só o convite"); 3 — Task 2 ("referralView: tabela das situações; desconhecida vira Em andamento"); 4 — Task 2 ("agendado: com e sem local, data inválida"); 5 — Task 3 ("interruptor desligado com a tela aberta…") e Task 4 ("sem care_regulation na sessão: Minhas triagens como hoje").
- **Proporção:** os blocos de código são testes (rótulos, frases e valores que o implementador não decide sozinho) e o parágrafo do README; a tela e as regras ficam com o comportamento.
