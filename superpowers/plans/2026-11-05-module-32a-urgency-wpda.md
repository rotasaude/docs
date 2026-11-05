# Módulo 32a — Urgência (UPA) (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 32a** na branch `feat/mod-32a-urgency` (worktree `apps/api/.claude/mod32a`, porta **3054**) com a fonte `urgency` em `Timeline::KINDS` e a visão do cidadão do contrato §10, na forma das Divergências W1–W4. Ordem (contrato §13): contracts → api → dashboard → **wpda** → maintenance. As Tasks 0–2 não dependem do api rodando (dublês de `fetch`); a prova da Task 3 depende.

**Goal:** Lado wpda de F-32.6: "Meu histórico" mostra o atendimento de urgência com o rótulo próprio, o título que o api manda ("Atendimento de urgência — <UPA>"), a data e a hora, o desfecho em linguagem simples e a orientação de retorno. Nunca mostra o CID, a cor da classificação, a queixa nem o médico.

**Architecture:** **Nenhuma tela nova, nenhuma rota nova, nenhum componente novo** (spec decisão 17; contrato §10). O texto vem pronto do api, como nas outras fontes. O wpda muda três coisas:
- `src/lib/history.ts` ganha o rótulo `urgency: "Urgência"` e a defesa da entrada: o lugar é só a unidade e a situação nunca aparece;
- `normalizeEntry` (`src/lib/citizenApi.ts`) passa a guardar o campo opcional `return_guidance`, no mesmo padrão do `status_label` do 21a (W1). Cor, CID e queixa que um api com defeito mandar já são descartados, porque `normalizeEntry` só copia os campos de `CitizenEntry`;
- o cartão de `EntryList` (`MyHistoryStep.tsx`) desenha `return_guidance` numa linha própria, "Orientação de retorno: …".

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + jest-dom (jsdom, `src/test/setup.ts` com `cleanup` e `stubDevice`), `vi.stubGlobal("fetch")`, Vite 5 (base `/wpda/`).

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md`, nestas partes:
- Afeta "`apps/wpda`";
- decisões 4, 12 e 17;
- R2 (o wpda acha o atendimento pelo CPF do paciente) e R4 (provisório fora; depois da identificação, no paciente real);
- §8 F-32.6.

Também `docs/adr/0050.md` e o contrato `docs/superpowers/plans/2026-11-05-module-32a-urgency-contracts.md` §10 e §13 (diferenças em W1–W4). Herdados: ADR 0035 (linha do tempo do cidadão) e os planos do wpda do 30 (entrada `transport`) e do 31 (molde do teste de tela).

## Global Constraints

- **Nada clínico da urgência no wpda** (contrato §10): nunca CID, cor da classificação, queixa, sinais vitais, justificativa, evolução nem o nome do médico. O cartão mostra só o rótulo, a data e a hora, o título, o desfecho (`summary`), a orientação (`return_guidance`) e a unidade.
- Rótulo: `citizenKindLabel("urgency") === "Urgência"`. `entryPlace` da urgência é só `unit.name` (nunca `professional_name`). `entryStatusText` da urgência é `null`, mesmo que o api mande `status` ou `status_label`.
- `return_guidance`: guardado por `normalizeEntry` só quando é texto não vazio, senão a chave não existe (o mesmo padrão do `status_label`). Na tela, é a linha exata `Orientação de retorno: <texto>`. Sem orientação, a linha não existe.
- O texto do desfecho e da orientação vem pronto do api (W2 e W3). O wpda não traduz código de desfecho.
- Sem filtro novo de `kind`: a urgência aparece para o cidadão. O filtro do CAPS (`isCapsOnlyKind`) continua igual.
- Na URL, só `citizen_id` (e `kinds[]`, `before`, `limit` do histórico). Nada em `localStorage`, `sessionStorage` ou `console`. Sem polling.
- Interface do canal: nenhum componente comum muda. A linha nova usa o estilo `muted` que já existe (`var(--ink2)`). `src/theme/guards.test.ts` e `e2e/device-gate.spec.ts` continuam verdes.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git `/opt/homebrew/bin/git`. Merge, push e tag só com autorização.
- Board: a primeira ação é mover F-32.1..F-32.6 do board #1 para In Progress (Task 0). O wpda é dono de F-32.6 com o api e o dashboard. O card vai para Done quando o último dos três terminar, e para Verified depois da prova (Task 3).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod32a`, `G=/opt/homebrew/bin/git`. O worktree fica **fora de `apps/`**, na branch `feat/mod-32a-urgency`. Os caminhos são relativos a `$W`.
- Testes: `bin/test wpda -w wpda-mod32a <arquivos>`. Tipos: `cd $W && npx tsc --noEmit`.
- O api do 32a roda em dev na **3054**, no container `api-dev` (rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5214**, com `node_modules` em volume próprio. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Api que mande cor, CID ou queixa na entrada de urgência** (bug, ou visão profissional servida por engano). Nada disso chega à tela. Testes: Task 1, "history: entrada urgency guarda a orientação e descarta cor, CID e queixa"; Task 2, "Meu histórico: urgência com título, desfecho e orientação; sem cor, CID, queixa nem médico".
2. **Nome do médico ou situação interna na urgência** (`professional_name` preenchido; `status: "in_care"`/`"closed"`, que hoje viram "em atendimento"/"atendimento encerrado"). O lugar é só a UPA e a situação some. Testes: Task 1, "entrada urgency: rótulo Urgência, só a unidade, sem situação"; Task 2, o mesmo caso de tela.
3. **Orientação ausente, vazia ou de tipo errado** (evasão, transferência, alta comum; `""`, `null`, número). Sem linha "Orientação de retorno". Testes: Task 1, "history: orientação vazia, nula ou que não é texto não vira chave"; Task 2, "evasão: só o desfecho, sem orientação".
4. **Orientação longa ou com quebra de linha** (texto livre do médico). Aparece inteira, numa linha própria do cartão, sem rolagem lateral a 375 px. Testes: Task 2, "orientação longa: inteira, no cartão"; Task 3, prova a 375 px.
5. **Api antigo sem a fonte `urgency`, ou com outro `kind`**. A tela continua como hoje, e um `kind` desconhecido vira "Registro". Teste: Task 1, o caso existente "rótulos do contrato; desconhecido vira Registro" continua verde. A forma real do `kind` é conferida na Task 0.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, forma real da entrada do api, worktree, base | 0 |
| `src/lib/history.ts`, `src/lib/history.test.ts`, `src/lib/citizenApi.ts`, `src/lib/citizenApi.history.test.ts` | rótulo, defesa e `return_guidance` | 1 |
| `src/modules/citizen/MyHistoryStep.tsx`, `src/modules/citizen/MyHistoryStep.urgency.test.tsx`, `README.md` | a linha da orientação no cartão; README | 2 |
| — | suíte, build, revisão, prova | 3 |

**Estratégia de teste:**
- funções puras do histórico (rótulo, lugar, situação) com o helper `e` de `history.test.ts` (Task 1);
- `normalizeEntry` com `fetch` falso, que confere a forma exata da entrada e o descarte de cor, CID e queixa (Task 1);
- a tela inteira com `fetch` falso por URL, para a normalização rodar de verdade, no molde de `MyHistoryStep.caps.test.tsx`: urgência com orientação, sem orientação e com orientação longa (Task 2);
- prova contra o api do 32a com um encontro fechado em `discharged_to_primary_care` e outro em `left_without_care` (Task 3).

---

### Task 0: Cards, âncoras, forma da entrada, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`54b44bf` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod32a`.
- Produces: os cards em In Progress; `$W`; a base anotada; a forma real da entrada `urgency` do cidadão.

- [ ] **Step 1: Cards F-32.1..F-32.6 para In Progress.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-32\\.[1-6] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os seis em In Progress ou além. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e a entrada do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod32a
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/history.ts | grep -n '  transport: "Viagem"\|^export const TRANSPORT_KIND\|^export function entryPlace\|^export function entryStatusText'
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '  status_label?: string | null;\|status_label ? { status_label'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/MyHistoryStep.tsx | grep -n '{status && <div>{status}</div>}'
grep -n 'citizen:' -A3 $A/app/services/timeline/sources/urgency_source.rb
grep -n 'return_guidance' $A/app/services/timeline/json.rb
```

Expected:
- as âncoras do wpda;
- no api, a visão do cidadão com `title`, `summary` (o desfecho), `professional_name: nil`, `status: nil` e `return_guidance` (W1), e `Timeline::Json.citizen` com `return_guidance` só quando presente.

Forma diferente (outro nome do campo, a orientação dentro de `summary`, `kind` diferente de `urgency`): ajuste os nomes nas Tasks 1–2 **antes** de começar e anote para a Task 3. Se a orientação vier dentro de `summary`, a Task 1 fica só com o rótulo e a defesa, e a Task 2 tira a linha nova. Se a visão do cidadão ainda não existir, avise a sessão do api e siga com os dublês.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod32a -b feat/mod-32a-urgency origin/main
cd $R/.worktrees/wpda-mod32a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod32a
```

Expected: verde e `tsc-ok`. **Anote** a base. A Task 3 espera **+1 arquivo de teste** (`MyHistoryStep.urgency`) e casos novos em `history.test.ts` e `citizenApi.history.test.ts`.

---

### Task 1: Rótulo, defesa da entrada e `return_guidance`

**Files:**
- Modify: `src/lib/history.ts` (rótulo depois de `transport`; constante; `entryPlace`; `entryStatusText`), `src/lib/citizenApi.ts` (`CitizenEntry`, `normalizeEntry`)
- Test: `src/lib/history.test.ts`, `src/lib/citizenApi.history.test.ts`

**Interfaces:**
- Consumes: `CitizenEntry`, `normalizeEntry`, `TRANSPORT_KIND`, `VACCINATION_KIND` (existentes).
- Produces:
  - `URGENCY_KIND = "urgency"` (exportado de `history.ts`);
  - `citizenKindLabel("urgency") === "Urgência"`;
  - `entryPlace` e `entryStatusText` com o caso `urgency`;
  - `CitizenEntry.return_guidance?: string | null` (opcional), preenchido por `normalizeEntry` só com texto não vazio.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/history.test.ts` (o helper `e` já existe; importe `URGENCY_KIND` junto dos outros):

```ts
describe("módulo 32a", () => {
  it("entrada urgency: rótulo Urgência, só a unidade, sem situação", () => {
    const u = e("u1", { kind: URGENCY_KIND, title: "Atendimento de urgência — UPA Boqueirão", unit: { name: "UPA Boqueirão" },
      professional_name: "Dr. Paulo Mendes", status: "closed", summary: "Alta, com orientação para procurar a sua UBS" });
    expect(URGENCY_KIND).toBe("urgency");
    expect(citizenKindLabel("urgency")).toBe("Urgência");
    expect(entryPlace(u)).toBe("UPA Boqueirão");
    expect(entryPlace(e("u2", { kind: URGENCY_KIND, professional_name: "Dr. Paulo Mendes" }))).toBeNull();
    expect(entryStatusText(u)).toBeNull();
    expect(entryStatusText({ ...u, status: "in_care", status_label: "amarela" })).toBeNull();
  });
});
```

No fim de `src/lib/citizenApi.history.test.ts` (os helpers `mockJson` e o `afterEach` já existem):

```ts
describe("citizenApi — urgência (módulo 32a)", () => {
  const RAW = { id: "urgency:e1", kind: "urgency", occurred_at: "2026-11-04T22:10:00-03:00", unit: { name: "UPA Boqueirão" },
    title: "Atendimento de urgência — UPA Boqueirão", summary: "Alta, com orientação para procurar a sua UBS" };

  it("history: entrada urgency guarda a orientação e descarta cor, CID e queixa", async () => {
    mockJson(200, { verified: true, partial: false, next_before: null, entries: [ { ...RAW,
      return_guidance: "Procure a sua UBS em até 7 dias para reavaliar a pressão.",
      color: "yellow", current_color: "yellow", cid10: "I10", problems: [ "I10 Hipertensão essencial" ],
      complaint: "dor de cabeça forte", vital_signs: { systolic: 190 } } ] });
    const h = await citizenApi.history("p1");
    expect(h.entries[0]).toEqual({
      id: "urgency:e1", kind: "urgency", occurred_at: "2026-11-04T22:10:00-03:00", unit: { name: "UPA Boqueirão" },
      professional_name: null, title: "Atendimento de urgência — UPA Boqueirão",
      summary: "Alta, com orientação para procurar a sua UBS", status: null,
      return_guidance: "Procure a sua UBS em até 7 dias para reavaliar a pressão.", document: null
    });
  });

  it("history: orientação vazia, nula ou que não é texto não vira chave", async () => {
    mockJson(200, { entries: [ { ...RAW, id: "urgency:a", return_guidance: "" }, { ...RAW, id: "urgency:b", return_guidance: null },
      { ...RAW, id: "urgency:c", return_guidance: 7 }, { ...RAW, id: "urgency:d" } ] });
    const h = await citizenApi.history("p1");
    expect(h.entries.map((x) => "return_guidance" in x)).toEqual([ false, false, false, false ]);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod32a src/lib/history.test.ts src/lib/citizenApi.history.test.ts`. Expected: FAIL (`URGENCY_KIND` não existe; `return_guidance` some na normalização).

- [ ] **Step 3: Implemente**
  - **`history.ts`:** `export const URGENCY_KIND = "urgency";` junto de `TRANSPORT_KIND`. Em `KIND_LABEL`, depois de `transport`, o comentário `// Módulo 32a, contrato §10.` e `urgency: "Urgência"`. `entryPlace` devolve `entry.unit?.name ?? null` para `URGENCY_KIND` (nunca o médico), como já faz na vacina. `entryStatusText` devolve `null` para `URGENCY_KIND`, junto de vacina e viagem.
  - **`citizenApi.ts`:** em `CitizenEntry`, depois de `status_label`, o comentário `// Módulo 32a (contrato §10, W1): orientação de retorno já em palavras do cidadão; presente só quando o api manda.` e `return_guidance?: string | null;`. Em `normalizeEntry`, depois do espalhamento de `status_label`, `...(typeof e.return_guidance === "string" && e.return_guidance.trim() !== "" ? { return_guidance: e.return_guidance } : {})`. O `typeof` é a defesa contra número ou objeto, que chegam pelo JSON apesar do tipo de `RawEntry`.

- [ ] **Step 4: Run** o mesmo comando e `cd $W && npx tsc --noEmit`. Expected: PASS, e os casos antigos de `history.test.ts` (inclusive "desconhecido vira Registro") continuam verdes.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/history.ts src/lib/history.test.ts src/lib/citizenApi.ts src/lib/citizenApi.history.test.ts
$G commit -m "feat: label urgency visits in the citizen history without the doctor or the status

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: A orientação de retorno no cartão e o README

**Files:**
- Modify: `src/modules/citizen/MyHistoryStep.tsx` (em `EntryList`, depois de `{status && <div>{status}</div>}`), `README.md` (parágrafo depois do de Atenção psicossocial)
- Test: `src/modules/citizen/MyHistoryStep.urgency.test.tsx` (novo)

**Interfaces:**
- Consumes: `CitizenEntry.return_guidance`, `URGENCY_KIND`, `entryPlace`, `entryStatusText` (Task 1); `MyHistoryStep` (existente).
- Produces: a linha `Orientação de retorno: <texto>` no cartão de qualquer entrada com `return_guidance`. Hoje só a urgência tem o campo.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyHistoryStep.urgency.test.tsx
// Módulo 32a (contrato §10): atendimento de urgência em "Meu histórico", sem CID, cor, queixa nem médico.
// Fetch falso por URL, para a normalização de citizenApi.history() rodar de verdade.
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import { MyHistoryStep } from "./MyHistoryStep";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-11-05T10:00:00-03:00"));
});
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); vi.unstubAllGlobals(); });

const GUIDANCE = "Procure a sua UBS em até 7 dias para reavaliar a pressão.";
const URGENCY = { id: "urgency:e1", kind: "urgency", occurred_at: "2026-11-04T22:10:00-03:00", unit: { name: "UPA Boqueirão" },
  professional_name: "Dr. Paulo Mendes", title: "Atendimento de urgência — UPA Boqueirão",
  summary: "Alta, com orientação para procurar a sua UBS", status: "closed", return_guidance: GUIDANCE,
  color: "yellow", cid10: "I10", complaint: "dor de cabeça forte" };
const CONSULTATION = { id: "consultation:c1", kind: "consultation", occurred_at: "2026-11-04T23:00:00-03:00",
  unit: { name: "UPA Boqueirão" }, title: "Consulta" };

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

describe("MyHistoryStep — urgência (módulo 32a)", () => {
  it("Meu histórico: urgência com título, desfecho e orientação; sem cor, CID, queixa nem médico", async () => {
    stubCitizenApi([ CONSULTATION, URGENCY ]);
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    const timeline = within(await screen.findByRole("region", { name: "Linha do tempo" }));
    const card = (await timeline.findByText("Atendimento de urgência — UPA Boqueirão")).closest("li") as HTMLElement;
    expect(within(card).getByText("Urgência")).toBeInTheDocument();
    expect(within(card).getByText("04/11/2026, 22:10")).toBeInTheDocument();
    expect(within(card).getByText("Alta, com orientação para procurar a sua UBS")).toBeInTheDocument();
    expect(within(card).getByText(`Orientação de retorno: ${GUIDANCE}`)).toBeInTheDocument();
    expect(within(card).getByText("UPA Boqueirão")).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/yellow|amarel|I10|dor de cabeça|Dr\. Paulo Mendes|atendimento encerrado|Registro/);
  });

  it("evasão: só o desfecho, sem orientação", async () => {
    stubCitizenApi([ { ...URGENCY, summary: "Saiu antes do atendimento", return_guidance: null } ]);
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    expect(await screen.findByText("Saiu antes do atendimento")).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/Orientação de retorno/);
  });

  it("orientação longa: inteira, no cartão", async () => {
    const long = `${"Volte à UBS Vila Hauer para medir a pressão e trazer os exames pedidos na UPA. ".repeat(4)}Se piorar, volte à UPA.`;
    stubCitizenApi([ { ...URGENCY, return_guidance: long } ]);
    render(<MyHistoryStep citizenId="p1" onBack={vi.fn()} onValidate={vi.fn()} />);
    const line = await screen.findByText(`Orientação de retorno: ${long}`);
    expect(line.closest("li")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod32a src/modules/citizen/MyHistoryStep.urgency.test.tsx`. Expected: FAIL (não existe linha "Orientação de retorno").

- [ ] **Step 3: Implemente**
  - **`MyHistoryStep.tsx`:** em `EntryList`, logo depois de `{status && <div>{status}</div>}`, desenhe `{e.return_guidance && <div style={{ ...muted, overflowWrap: "anywhere" }}>{`Orientação de retorno: ${e.return_guidance}`}</div>}`.
  - **`README.md`:** parágrafo **Urgência (módulo 32a).** depois do de Atenção psicossocial, dizendo que:
    - o atendimento na UPA aparece em "Meu histórico" como "Urgência", com a unidade, a data e a hora, o desfecho em palavras simples e, quando o médico deixa, a orientação de retorno;
    - nunca aparecem o CID, a cor da classificação, a queixa nem o médico (contrato §10), e o cliente se defende do lugar e da situação (`history.ts`);
    - quem chegou sem identificação só aparece depois de identificado, e aí no histórico da própria pessoa;
    - não há tela nova nem botão novo: a espera de cada UPA está fora (spec, "Fora desta entrega").

- [ ] **Step 4: Run** o mesmo comando mais `src/modules/citizen/MyHistoryStep.test.tsx src/modules/citizen/MyHistoryStep.caps.test.tsx`, e `cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/MyHistoryStep.tsx src/modules/citizen/MyHistoryStep.urgency.test.tsx README.md
$G commit -m "feat: show the return guidance of urgency visits in the citizen history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test wpda -w wpda-mod32a && cd $W && npx tsc --noEmit && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base mais 1 arquivo de teste;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage" src/lib/history.ts src/modules/citizen/MyHistoryStep.tsx
grep -n -i "color\|cid\|complaint\|queixa" src/lib/history.ts src/modules/citizen/MyHistoryStep.tsx
```

Expected: sem saída.

- [ ] **Step 3: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 4, 12 e 17, R2, R4, F-32.6) e o contrato §10. Ele confere:
  - nenhuma tela nova;
  - só o rótulo, a defesa e a linha da orientação;
  - nada de cor, CID, queixa nem médico;
  - `return_guidance` opcional, como o `status_label`;
  - cores só por papel do tema.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 32a na **3054**, com a semente do plano do api: Curitiba com `urgency_care` ligado e uma UPA. A pessoa **validada** do celular de dev precisa ter três encontros fechados:
- um `discharged_to_primary_care` com orientação;
- um `left_without_care`;
- um de paciente provisório, identificado depois como essa pessoa.

Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod32a --network rota-saude_default -p 5214:5214 \
  -v $R/.worktrees/wpda-mod32a:/app -v wpda-mod32a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3054 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5214 --host 0.0.0.0"
docker logs -f wpda-mod32a   # espere o "ready"
```

Abra `http://curitiba.localhost:5214/wpda/` a 375 px, com toque. O usuário entra com o celular da semente e o código SMS de dev. Guarde os screenshots em `evidencias/2026-11-05-modulo-32a/wpda/`. Roteiro:
- "Meu histórico" mostra "Urgência" com "Atendimento de urgência — <UPA>", a data e a hora, o desfecho e "Orientação de retorno: …". Sem cor, CID, queixa nem médico;
- a evasão mostra só "Saiu antes do atendimento" (ou o texto do api), sem orientação;
- o atendimento do provisório aparece no histórico da pessoa depois da identificação (R4). Antes dela, não aparece em lugar nenhum do wpda;
- na rede (`read_network_requests` ou DevTools), nenhuma resposta `/citizen/history` contém `color`, `cid`, a queixa ou o nome do médico da semente;
- "Documentos" lista o atestado ou a receita da consulta de urgência, se a semente tiver um, como documento comum;
- alto contraste legível, e a 375 px sem rolagem lateral.

Ao terminar: `docker rm -f wpda-mod32a && docker volume rm wpda-mod32a-node-modules`.

- [ ] **Step 5: Card.** Se as partes do api e do dashboard de F-32.6 estiverem **Done**, mova o F-32.6 para **Done** (`98236657`), e para **Verified** se a prova passou. Senão, deixe como está e avise a sessão do api.

- [ ] **Step 6: Pare.** O merge só vem depois do api e do dashboard do 32a, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-32a-urgency`, `push origin main`, `worktree remove $W`, `branch -d`. O `wpda-dev` (5176) recompila sozinho.

---

## Divergências propostas ao contrato

1. **W1 — Onde vai a orientação de retorno.** O §10 diz que o cidadão vê "o desfecho em linguagem simples e a orientação de retorno", mas `<citizen_entry>` só tem `title`, `summary` e `status`. Proposta: um campo opcional `return_guidance` (texto pronto, ou ausente) na visão do cidadão, emitido por `Timeline::Json.citizen` só quando presente, como o `status_label` do 21a (A24). O `summary` fica só com o desfecho. A frase é composta pelo api a partir de `return_guidance: {days, reason}`, por exemplo "Procure a sua UBS em até 7 dias: <motivo>". Alternativa: juntar tudo em `summary`. Não muda a forma, mas o cidadão perde a linha própria, e a Task 2 sai do plano.
2. **W2 — Campos da entrada `urgency` do cidadão.** O §10 fixa o título e o que não entra. Proposta:
   - `occurred_at` é a chegada (`attendance.checked_in_at`);
   - `unit: {name}` é a UPA;
   - `professional_name: null` (a decisão 17 não lista o médico; ele já aparece na entrada "Consulta" do 19a);
   - `status: null` e `document: null` (os documentos saem como `clinical_document`);
   - só encontros **fechados** entram, porque sem desfecho não há texto;
   - a `AttendanceSource` não emite entrada para o `attendance` `walk_in`, para a mesma visita não aparecer duas vezes. O wpda se defende do lugar e da situação (Task 1).
3. **W3 — Desfecho em linguagem simples.** O §10 não traz o texto. Proposta (em `Timeline::Sources::Labels`):

   | Desfecho | Texto para o cidadão |
   |---|---|
   | `discharged` | "Atendimento concluído, com alta" |
   | `discharged_to_primary_care` | "Alta, com orientação para procurar a sua UBS" |
   | `transferred` | "Transferido(a) para outro serviço" |
   | `left_without_care` | "Saiu antes do atendimento" |
   | `left_against_advice` | "Saiu antes da alta médica" |

   O destino da transferência e o pedido de vaga não entram (são cifrados, §13).
4. **W4 — Óbito no histórico do cidadão.** O §10 não trata do `death`. O par do celular pode ser um familiar. Proposta: o encontro com desfecho `death` **não** aparece na visão do cidadão (a visão profissional segue o §10). A alternativa ("Atendimento concluído") esconde o fato e induz a erro, e mostrar "Óbito" no canal web pede uma decisão do usuário. O coordenador decide; o wpda não muda em nenhum caso.

## Self-review

- **Cobertura (decisões 4, 12, 17, R2, R4, F-32.6; contrato §10):**
  - rótulo e título: Task 1 (rótulo) e Task 2 (cartão);
  - desfecho em linguagem simples: Task 2 (`summary`) e W3;
  - orientação de retorno: Tasks 1 e 2 (Review Focus 3 e 4; W1);
  - sem CID nem cor: Tasks 1 e 2 (Review Focus 1);
  - sem médico nem situação: Task 1 (Review Focus 2; W2);
  - provisório e identificação: api, conferido na prova (Task 3);
  - óbito: W4. Nenhuma tela nova, como pede a decisão 17.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards, forma real da entrada (Task 0, com a regra de ajuste) e semente (Task 3).
- **Tipos e nomes:** `URGENCY_KIND` e `return_guidance` (Task 1) são usados na Task 2. Os existentes foram conferidos em `origin/main` (`54b44bf`): `citizenKindLabel`, `entryPlace`, `entryStatusText`, `TRANSPORT_KIND`, `VACCINATION_KIND`, `KIND_LABEL`, `CitizenEntry`, `RawEntry`, `normalizeEntry` (com o espalhamento de `status_label`), `EntryList`, `muted`, `MyHistoryStep`, `mockJson`, o helper `e` e o formato `fmtDateTime` ("04/11/2026, 22:10").
- **Review Focus:** 1 — Tasks 1 e 2; 2 — Tasks 1 e 2; 3 — Tasks 1 e 2; 4 — Task 2 e prova; 5 — Task 1 (caso existente) e Task 0.
- **Proporção:** código só em testes e na frase exata da linha; a mudança de produção são quatro pontos pequenos.
