# Módulo 21a — Rastreamento do exame (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21a** rodando na branch dele (`feat/mod-21a-exams`, porta **3041**) com `exam_tracking`, `exam_fhir_inbound`, `exam_pec_bridge` e `fake_lab` em `Platform::Features::CATALOG`. Ele é **mergeado antes** deste (contrato §14: contracts → api → dashboard e wpda → maintenance → catálogo). As Tasks 0–2 não dependem dele (os testes usam `fetch` falso); a prova da Task 3 depende. A Task 4 (catálogo público) vem **por último**, depois de api, dashboard, wpda e maintenance entregues.

**Goal:** Lado maintenance de F-21.1: os quatro interruptores do 21a aparecem e são ligados/desligados na aba **Funcionalidades** pelo mecanismo genérico do módulo 16, com nome em português e as frases dos pré-requisitos que faltam; com `fake_lab` ligado, a aba avisa que a cidade recebe laudos do laboratório simulado. E a entrada do módulo 21 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única).

**Architecture:** Nada muda no liga/desliga nem no schema GraphQL: a `FeaturesTab` já lista o catálogo que o api devolve (`city.features`) e chama `setCityFeature`. Entram só rótulos em `src/lib/features.ts` (`FEATURE_LABELS` e `MISSING_LABELS`) e o aviso do laboratório simulado em `src/lib/exams.ts`, desenhado na aba como os avisos do PSC e do SNCR simulados (pelo estado **ligado**, não pelo "utilizável"). Em produção o api nem põe `fake_lab` no catálogo (`environments: SIMULATION_ENVS`); para a tela é o mesmo que desligado. No catálogo público, o módulo 21 entra no Ciclo 2 com `cycle`, os F-IDs entregues e o status de cada um; o rodapé passa a listar três módulos em validação com vírgula.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite + react-router, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-13-module-21a-exam-tracking-design.md` (§3 "Interruptores", §9 "Maintenance", §10 itens 5 e 6, §11 F-21.1 e a tabela inteira para o catálogo, §12), `docs/adr/0036.md` e o contrato `docs/superpowers/plans/2026-10-13-module-21a-exams-contracts.md` §11 (as diferenças estão em "Divergências propostas ao contrato").

## Global Constraints

- Interruptores (contrato §11; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`, que transforma `feature:<x>` em `<x>_disabled`):
  - `exam_tracking` com `requires: ["feature:clinical_record"]`; faltando, `missing: ["clinical_record_disabled"]` (frase já existente);
  - `exam_fhir_inbound` com `requires: ["feature:exam_tracking"]`; faltando, `missing: ["exam_tracking_disabled"]`;
  - `exam_pec_bridge` com `requires: ["feature:exam_tracking", "feature:pec_external_view"]`; faltando, `exam_tracking_disabled` e/ou `pec_external_view_disabled`;
  - `fake_lab` com `requires: ["feature:exam_fhir_inbound"]`; faltando, `missing: ["exam_fhir_inbound_disabled"]`; **só fora de produção** — em produção não aparece no catálogo e a mutation responde `unknown_feature`.
  - Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados, como no módulo 16). Nenhuma mudança no fluxo de `setCityFeature` da `FeaturesTab`.
- Aviso do laboratório simulado: pelo estado **ligado** de `fake_lab`, na aba (`role="note"`, nome "modo do laboratório"). Texto: `FAKE_LAB_NOTICE`. Desligado ou ausente do catálogo: nenhum aviso e nada diz "SIMULADO".
- Nenhum token FHIR, CPF, CNS, valor de resultado ou PDF passa pelo maintenance; os tokens dos prestadores são do admin municipal no dashboard (contrato §5).
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null` na tela.
- Interface: componentes existentes (`Panel`, `DataTable`, `Tag`, `ConfirmButton`), sem redesign; cores só por papel do tema (`var(--warn)`), a guarda `src/theme/guards.test.ts` roda na suíte. Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única** (mudança de texto e status é no `src/data/funcionalidades.json`); toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); módulo novo com `cycle` e `meta.cycles[].modules` atualizado; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge só com a CI verde e o branch atualizado, e manual.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.

## Ambiente de execução

- Variáveis usadas nos comandos: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod21`, `C=$R/.worktrees/catalog-mod21`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o dev do app): maintenance em `$M`, branch `feat/mod-21a-exams`, com `npm ci` (Task 0); catálogo em `$C`, branch `content/module-21a-catalog`, com `npm ci` (Task 4). Caminhos de arquivo das tasks relativos ao worktree da task.
- maintenance — testes: `cd $M && npx vitest run <arquivos>` (o `vitest.config.ts` exclui `e2e/**`); tipos: `npx tsc --noEmit`; CI: `npx vitest run`, `npx tsc --noEmit`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `cleanup` no `afterEach`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- O api do 21a roda em dev na **3041**, dentro do container `api-dev` (rede `rota-saude_default`, apelido `api`). Login de dev do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **O mantenedor liga `fake_lab` em staging e esquece.** As guias do prestador simulado passam a ganhar laudo de exemplo em pacientes da cidade. O aviso tem de estar onde se liga, com "LABORATÓRIO SIMULADO" e "sem validade clínica", e sumir quando desliga. Testes: Task 2, "laboratório simulado ligado: aviso na aba" e "ligar fake_lab por dois cliques: após o refetch a aba avisa".
2. **Produção: `fake_lab` não existe no catálogo.** A aba não pode quebrar, nem mostrar a linha, nem dizer "simulado". Teste: Task 2, "catálogo sem fake_lab (produção): nada quebra e nada diz simulado".
3. **Ligar `exam_fhir_inbound` ou `exam_pec_bridge` com o pré-requisito desligado.** Fica "ligada, falta pré-requisito", com o motivo em português (não o código cru `exam_tracking_disabled`), e o liga/desliga é o genérico. Testes: Task 1, "pré-requisitos do 21a têm frase"; Task 2, "os pré-requisitos do 21a aparecem em português na lista" e "exam_tracking pelo mecanismo genérico".
4. **Catálogo público promete o que não foi entregue.** A entrada FHIR só foi provada com o laboratório simulado e o botão no PEC depende de prova técnica: nenhuma delas pode aparecer como "Disponível e verificado". Teste: Task 4, "módulo 21: entrada FHIR em validação e o PEC conforme a prova".
5. **Rodapé do catálogo com três módulos em validação.** "o 19 e o 20 e o 21" é português ruim e o teste de hoje fixa dois. Teste: Task 4, "formatValidating: um, dois e três módulos" e "o rodapé resume o estado dos módulos".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card F-21.1, conferência do main e do api do 21a, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/exams.ts`, `src/lib/exams.test.ts` | nomes das quatro chaves e frases dos três pré-requisitos novos; aviso do laboratório simulado | 1 |
| `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md` | o aviso na aba; os quatro interruptores pelo fluxo genérico | 2 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 3 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/pages/ModuleList.tsx`, `src/App.test.tsx` | módulo 21 no Ciclo 2; rodapé com três módulos em validação | 4 |

**Estratégia de teste:** rótulos e frases com tabela de casos (Task 1); os quatro interruptores pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação, sem mudar o código do liga/desliga, e o aviso do laboratório ligado, desligado e ausente (Task 2); prova no navegador contra o api do 21a (Task 3). No catálogo: o teste do conteúdo (`validateCatalog` aceita, módulo 21 com os F-IDs e os status) e o do rodapé (Task 4).

---

### Task 0: Card, conferência do main e do api do 21a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`cc91351` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`).
- Produces: card F-21.1 em In Progress; worktree `$M` na branch `feat/mod-21a-exams`; a base anotada.

- [ ] **Step 1: Mova o card F-21.1 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh`:

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | startswith("F-21.1 ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Se o card já estiver em In Progress (o F-21.1 é também do api), siga. Senão, com o `id` do item e o id da opção "In Progress":

```bash
gh project item-edit --id <PVTI_do_F-21.1> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: o item com `status: "In Progress"`. Se o card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main e o api do 21a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n \
  '  city_unreachable: "banco da cidade inalcançável — não deu para conferir"\|  pec_external_view: "Página para os sistemas externos do PEC"\|exam_tracking\|fake_lab'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n \
  'import { CONTROLLED_KEY, SNCR_MOCK_NOTICE, sncrModeNotice } from "../lib/sncr";\|  const sncrSimulated = city ? sncrModeNotice(city.features).simulated : false;\|{SNCR_MOCK_NOTICE}'
grep -n -E 'key: "(exam_tracking|exam_fhir_inbound|exam_pec_bridge|fake_lab)"' -A3 $R/apps/api/.claude/mod21/app/services/platform/features.rb
```

Expected:
- em `features.ts`: as duas âncoras e **nenhuma** linha com `exam_tracking` ou `fake_lab` (se já existirem, a Task 1 só acrescenta o que faltar e o teste dela continua valendo);
- em `FeaturesTab.tsx`: as três âncoras (o import do SNCR, a linha `const sncrSimulated = …` e o `{SNCR_MOCK_NOTICE}` do aviso). Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 21a: as quatro entradas, com os `requires` do Global Constraints e `environments: SIMULATION_ENVS` no `fake_lab`. Se alguma faltar ou tiver outro `requires`, as Tasks 0–2 seguem com os nomes deste plano, mas **reporte** antes da Task 3 (Divergência M1/M2).

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod21 -b feat/mod-21a-exams origin/main
cd $R/.worktrees/maintenance-mod21 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera a base mais **1 arquivo de teste novo** (`src/lib/exams.test.ts`).

---

### Task 1: Nomes dos interruptores, frases dos pré-requisitos e o aviso do laboratório simulado

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/lib/exams.ts`
- Test: `src/lib/exams.test.ts`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes).
- Produces:
  - `featureLabel("exam_tracking")` → "Rastreamento de exames (pedido, regulação, prazos e resultado)"; `featureLabel("exam_fhir_inbound")` → "Entrada de resultados por FHIR (laboratórios)"; `featureLabel("exam_pec_bridge")` → "Solicitar exames pela tela de sistemas externos do PEC"; `featureLabel("fake_lab")` → "Laboratório simulado (desenvolvimento)";
  - `missingText("exam_tracking_disabled")` → "rastreamento de exames (exam_tracking) desligado"; `missingText("exam_fhir_inbound_disabled")` → "entrada de resultados por FHIR (exam_fhir_inbound) desligada"; `missingText("pec_external_view_disabled")` → "página para os sistemas externos do PEC (pec_external_view) desligada";
  - em `src/lib/exams.ts`:
    ```ts
    export const FAKE_LAB_KEY = "fake_lab";
    export const FAKE_LAB_NOTICE: string; // "Esta cidade recebe laudos do LABORATÓRIO SIMULADO — … sem validade clínica (fora de produção)."
    export function fakeLabOn(features: { key: string; enabled: boolean }[]): boolean;
    ```

- [ ] **Step 1: Write the failing test**

Em `src/lib/features.test.ts`, no fim do arquivo, acrescente:

```ts
describe("módulo 21a", () => {
  it("as quatro chaves do 21a têm nome em português", () => {
    expect(featureLabel("exam_tracking")).toBe("Rastreamento de exames (pedido, regulação, prazos e resultado)");
    expect(featureLabel("exam_fhir_inbound")).toBe("Entrada de resultados por FHIR (laboratórios)");
    expect(featureLabel("exam_pec_bridge")).toBe("Solicitar exames pela tela de sistemas externos do PEC");
    expect(featureLabel("fake_lab")).toBe("Laboratório simulado (desenvolvimento)");
  });

  it("pré-requisitos do 21a têm frase", () => {
    expect(missingText("exam_tracking_disabled")).toBe("rastreamento de exames (exam_tracking) desligado");
    expect(missingText("exam_fhir_inbound_disabled")).toBe("entrada de resultados por FHIR (exam_fhir_inbound) desligada");
    expect(missingText("pec_external_view_disabled")).toBe("página para os sistemas externos do PEC (pec_external_view) desligada");
    expect(missingSummary([ "exam_tracking_disabled", "pec_external_view_disabled" ]))
      .toBe("rastreamento de exames (exam_tracking) desligado; página para os sistemas externos do PEC (pec_external_view) desligada");
    // exam_tracking exige o prontuário: a frase do 19b já existe.
    expect(missingText("clinical_record_disabled")).toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

Crie `src/lib/exams.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { FAKE_LAB_KEY, FAKE_LAB_NOTICE, fakeLabOn } from "./exams";

describe("fakeLabOn", () => {
  it("só pelo estado ligado de fake_lab; ausente (produção) é desligado", () => {
    expect(fakeLabOn([ { key: FAKE_LAB_KEY, enabled: true } ])).toBe(true);
    expect(fakeLabOn([ { key: FAKE_LAB_KEY, enabled: false } ])).toBe(false);
    expect(fakeLabOn([ { key: "exam_fhir_inbound", enabled: true } ])).toBe(false);
    expect(fakeLabOn([])).toBe(false);
  });

  it("o aviso diz simulado e sem validade", () => {
    expect(FAKE_LAB_NOTICE).toMatch(/LABORATÓRIO SIMULADO/);
    expect(FAKE_LAB_NOTICE).toMatch(/sem validade clínica/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/lib/exams.test.ts`
Expected: FAIL — `featureLabel("exam_tracking")` é `null`, `missingText("exam_tracking_disabled")` volta cru e `./exams` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `MISSING_LABELS`, troque

```ts
  // O api não conseguiu ler o banco da cidade: usable vem false sem que se
```

por

```ts
  // Módulo 21a: exam_fhir_inbound e exam_pec_bridge exigem exam_tracking;
  // exam_pec_bridge também pec_external_view; fake_lab exige exam_fhir_inbound.
  exam_tracking_disabled: "rastreamento de exames (exam_tracking) desligado",
  exam_fhir_inbound_disabled: "entrada de resultados por FHIR (exam_fhir_inbound) desligada",
  pec_external_view_disabled: "página para os sistemas externos do PEC (pec_external_view) desligada",
  // O api não conseguiu ler o banco da cidade: usable vem false sem que se
```

e, em `FEATURE_LABELS`, troque

```ts
  pec_external_view: "Página para os sistemas externos do PEC"
};
```

por

```ts
  pec_external_view: "Página para os sistemas externos do PEC",
  // Módulo 21a (ADR 0036): fake_lab só existe fora de produção.
  exam_tracking: "Rastreamento de exames (pedido, regulação, prazos e resultado)",
  exam_fhir_inbound: "Entrada de resultados por FHIR (laboratórios)",
  exam_pec_bridge: "Solicitar exames pela tela de sistemas externos do PEC",
  fake_lab: "Laboratório simulado (desenvolvimento)"
};
```

Crie `src/lib/exams.ts`:

```ts
// Interruptores do módulo 21a no maintenance (ADR 0036; contrato §11). O
// liga/desliga e os pré-requisitos são o mecanismo genérico (features.ts);
// aqui só o aviso do laboratório simulado. Nenhum token FHIR, CPF, CNS ou
// valor de resultado passa pelo maintenance.
export const FAKE_LAB_KEY = "fake_lab";
export const FAKE_LAB_NOTICE =
  "Esta cidade recebe laudos do LABORATÓRIO SIMULADO — as guias agendadas do prestador simulado ganham resultado de exemplo, sem validade clínica (fora de produção).";

// Pelo estado LIGADO (não pelo "utilizável"): ligado, o fake-lab passa a
// enviar assim que a entrada FHIR estiver utilizável. Sem o interruptor no
// catálogo (produção) é o mesmo que desligado.
export function fakeLabOn(features: { key: string; enabled: boolean }[]): boolean {
  return features.some((f) => f.key === FAKE_LAB_KEY && f.enabled);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/lib/exams.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos de `features.test.ts` seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/lib/exams.ts src/lib/exams.test.ts
$G commit -m "feat: name the exam tracking switches and their prerequisites

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: O aviso do laboratório na aba e os quatro interruptores pelo fluxo genérico

**Files:**
- Modify: `src/screens/FeaturesTab.tsx`, `README.md`
- Test: `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `FAKE_LAB_NOTICE`, `fakeLabOn` (Task 1); rótulos e frases (Task 1); `FeaturesTab`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature`, `LEDI_OFF` (existentes em `FeaturesTab.test.tsx`).
- Produces: na aba, `<p role="note" aria-label="modo do laboratório">{FAKE_LAB_NOTICE}</p>` quando `fake_lab` está ligado.

- [ ] **Step 1: Write the failing test**

Em `src/screens/FeaturesTab.test.tsx`, acrescente ao bloco de imports `import { FAKE_LAB_NOTICE } from "../lib/exams";` e, no fim do arquivo:

```tsx
describe("FeaturesTab — rastreamento de exames (módulo 21a)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const TRACKING_OFF: Feature = {
    key: "exam_tracking", description: "Rastreamento do exame do pedido ao resultado",
    enabled: false, usable: false, missing: [ "clinical_record_disabled" ], changedAt: null, changedBy: null
  };
  const FHIR_OFF: Feature = {
    key: "exam_fhir_inbound", description: "Entrada FHIR de resultados", enabled: false, usable: false,
    missing: [ "exam_tracking_disabled" ], changedAt: null, changedBy: null
  };
  const BRIDGE_OFF: Feature = {
    key: "exam_pec_bridge", description: "Solicitar exames pela ponte do PEC", enabled: false, usable: false,
    missing: [ "exam_tracking_disabled", "pec_external_view_disabled" ], changedAt: null, changedBy: null
  };
  const LAB_OFF: Feature = {
    key: "fake_lab", description: "Laboratório simulado do compose de dev", enabled: false, usable: false,
    missing: [ "exam_fhir_inbound_disabled" ], changedAt: null, changedBy: null
  };
  const LAB_ON: Feature = { ...LAB_OFF, enabled: true, changedAt: "2026-10-13T13:00:00Z", changedBy: "dev@local" };

  it("exam_tracking pelo mecanismo genérico: nome em português, falta o prontuário e liga com dois cliques", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...TRACKING_OFF, enabled: true, changedAt: "2026-10-13T13:00:00Z", changedBy: "dev@local" };
    let listed = [ TRACKING_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    expect(await screen.findByText("exam_tracking")).not.toBeNull();
    expect(screen.getByText("Rastreamento de exames (pedido, regulação, prazos e resultado)")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar exam_tracking" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "exam_tracking", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "exam_tracking ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    );
  });

  it("os pré-requisitos do 21a aparecem em português na lista", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ FHIR_OFF, BRIDGE_OFF, LAB_OFF ])));
    renderTab();
    expect(await screen.findByText("exam_fhir_inbound")).not.toBeNull();
    expect(screen.getByText("Entrada de resultados por FHIR (laboratórios)")).not.toBeNull();
    expect(screen.getByText("Solicitar exames pela tela de sistemas externos do PEC")).not.toBeNull();
    expect(screen.getByText("Laboratório simulado (desenvolvimento)")).not.toBeNull();
    expect(screen.getByText("rastreamento de exames (exam_tracking) desligado")).not.toBeNull();
    expect(screen.getByText(
      "rastreamento de exames (exam_tracking) desligado; página para os sistemas externos do PEC (pec_external_view) desligada"
    )).not.toBeNull();
    expect(screen.getByText("entrada de resultados por FHIR (exam_fhir_inbound) desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/_disabled|undefined|null/);
    // Desligado: nenhum aviso de simulado.
    expect(screen.queryByRole("note", { name: "modo do laboratório" })).toBeNull();
  });

  it("laboratório simulado ligado: aviso na aba", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LAB_ON ])));
    renderTab();
    expect((await screen.findByRole("note", { name: "modo do laboratório" })).textContent).toBe(FAKE_LAB_NOTICE);
  });

  it("ligar fake_lab por dois cliques: após o refetch a aba avisa", async () => {
    const user = userEvent.setup();
    let listed = [ LAB_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ LAB_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: LAB_ON } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("fake_lab")).not.toBeNull();
    expect(screen.queryByRole("note", { name: "modo do laboratório" })).toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(await screen.findByRole("button", { name: "confirmar: ligar fake_lab" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "fake_lab", enabled: true });
    expect((await screen.findByRole("note", { name: "modo do laboratório" })).textContent).toBe(FAKE_LAB_NOTICE);
  });

  it("catálogo sem fake_lab (produção): nada quebra e nada diz simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      { ...TRACKING_OFF, enabled: true, usable: true, missing: [] },
      { ...FHIR_OFF, enabled: true, usable: true, missing: [] },
      LEDI_OFF
    ])));
    renderTab();
    expect(await screen.findByText("exam_fhir_inbound")).not.toBeNull();
    expect(screen.queryByText("fake_lab")).toBeNull();
    expect(screen.queryByRole("note", { name: "modo do laboratório" })).toBeNull();
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx`
Expected: FAIL — os dois testes do aviso não acham `role="note"` com o nome "modo do laboratório" (os outros três já passam com a Task 1: o fluxo genérico não muda).

- [ ] **Step 3: Write minimal implementation**

Em `src/screens/FeaturesTab.tsx`:

1. Depois de `import { CONTROLLED_KEY, SNCR_MOCK_NOTICE, sncrModeNotice } from "../lib/sncr";`, acrescente

```tsx
import { FAKE_LAB_NOTICE, fakeLabOn } from "../lib/exams";
```

2. Depois de `  const sncrSimulated = city ? sncrModeNotice(city.features).simulated : false;`, acrescente

```tsx
  // Módulo 21a: aviso pelo estado LIGADO de fake_lab (ausente em produção).
  const labSimulated = city ? fakeLabOn(city.features) : false;
```

3. Logo depois do bloco do aviso do SNCR (o `{sncrSimulated && ( … {SNCR_MOCK_NOTICE} … )}`), acrescente

```tsx
            {labSimulated && (
              <p role="note" aria-label="modo do laboratório" style={{ margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" }}>
                {FAKE_LAB_NOTICE}
              </p>
            )}
```

Em `README.md`, logo antes do parágrafo **Documentos clínicos (módulo 19c).**, acrescente:

```markdown
**Rastreamento de exames (módulo 21a).** Os interruptores `exam_tracking`
("Rastreamento de exames", exige `clinical_record`), `exam_fhir_inbound`
("Entrada de resultados por FHIR", exige `exam_tracking`), `exam_pec_bridge`
("Solicitar exames pela tela de sistemas externos do PEC", exige
`exam_tracking` e `pec_external_view`) e `fake_lab` ("Laboratório simulado",
exige `exam_fhir_inbound`) aparecem na mesma lista e ligam como os outros, com
o que falta em português. `fake_lab` só existe fora de produção; ligado, a aba
avisa que as guias do prestador simulado ganham laudo de exemplo, sem validade
clínica. Tokens FHIR dos prestadores, catálogo de exames e prestadores são do
admin municipal, no dashboard; nada disso passa por aqui. Ordem de deploy:
api → dashboard e wpda → maintenance; contra um api sem o 21a as chaves só
não aparecem.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx src/theme/guards.test.ts && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: warn in the features tab when the simulated lab is on

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-21a-exams`; o api do 21a na **3041**.
- Produces: branch do maintenance pronta para merge (depois do merge do api, do dashboard e do wpda do 21a).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd $M && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde (a CI roda os quatro): a base anotada na Task 0 mais **1 arquivo de teste novo** (`src/lib/exams.test.ts`). O `codegen:check` fica verde porque o schema não muda. Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e contagem `0`.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §9, §11 F-21.1), o ADR 0036 e o contrato §11. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature`, nenhuma consulta GraphQL nova, `schema.graphql` e `src/gql/*` intocados;
- as quatro chaves com nome em português e os três pré-requisitos novos com frase;
- o aviso sai do estado **ligado** de `fake_lab` e some quando desliga; em produção (chave ausente) nada diz "simulado";
- nenhum token, CPF, CNS ou resultado na tela; cores só por papel do tema.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 21a na **3041** (plano do api). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose (para alcançar `http://api:3041`), com `node_modules` num volume próprio (o do worktree é de macOS), depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod21 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod21:/app -v maintenance-mod21-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3041 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod21   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 21a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. **Anote** o estado dos quatro interruptores de Curitiba antes de mexer. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: as quatro chaves na lista, com o nome em português e a descrição do catálogo do api;
- com `exam_tracking` desligado: `exam_fhir_inbound` e `exam_pec_bridge` com "rastreamento de exames (exam_tracking) desligado" (e, no `exam_pec_bridge`, também "página para os sistemas externos do PEC (pec_external_view) desligada" se ela estiver desligada); `fake_lab` com "entrada de resultados por FHIR (exam_fhir_inbound) desligada";
- "ligar" → "confirmar: ligar exam_tracking" → a frase do estado devolvido ("ligada e utilizável" com o prontuário ligado); depois `exam_fhir_inbound` passa a utilizável ao ser ligado;
- ligar `fake_lab` → o aviso em destaque "Esta cidade recebe laudos do LABORATÓRIO SIMULADO — …"; "Última mudança" com a hora e `dev@local`; a tela **Auditoria** mostra as mudanças;
- desligar `fake_lab` → o aviso some;
- nenhum token, CPF, CNS ou valor de resultado na tela.

Depois: devolva os quatro interruptores ao estado anotado, `docker rm -f maintenance-mod21 && docker volume rm maintenance-mod21-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api, do dashboard e do wpda do 21a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21a-exams
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod21
/opt/homebrew/bin/git branch -d feat/mod-21a-exams
```

Se o lado api do F-21.1 já estiver **Done**, mova o card F-21.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0.

---

### Task 4: O módulo 21 no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/pages/ModuleList.tsx`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-21.1..F-21.13 no board #1 (status real de cada um), o resultado da prova técnica do PEC (plano do api, Task 0), o que foi entregue em api, dashboard, wpda e maintenance.
- Produces: módulo `"21"` no Ciclo 2 do JSON; `export function formatValidating(ids: string[]): string` em `src/pages/ModuleList.tsx` ("o 19 está" / "o 19 e o 20 estão" / "o 19, o 20 e o 21 estão").

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | startswith("F-21.")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod21 -b content/module-21a-catalog origin/main
cd $R/.worktrees/catalog-mod21 && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: os F-21.x em Done/Verified (os que não estiverem, **não entram como disponíveis**: pare e pergunte ao usuário); a suíte do catálogo verde. **Anote**: o resultado da prova técnica do PEC (F-21.11 entregue ou fora do escopo), o status atual dos módulos 19 e 20 no JSON (o rodapé depende deles) e o `meta.cycles` do Ciclo 2.

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, troque `import { formatRanges } from "../pages/ModuleList";` por `import { formatRanges, formatValidating } from "../pages/ModuleList";` e acrescente no fim:

```ts
describe("formatValidating", () => {
  it("um, dois e três módulos", () => {
    expect(formatValidating(["19"])).toBe("o 19 está");
    expect(formatValidating(["19", "20"])).toBe("o 19 e o 20 estão");
    expect(formatValidating(["19", "20", "21"])).toBe("o 19, o 20 e o 21 estão");
  });
});

describe("módulo 21", () => {
  it("entra no ciclo 2: entrada FHIR em validação e o PEC conforme a prova", () => {
    const m21 = catalog.modules.find((m) => m.id === "21")!;
    expect(m21.cycle).toBe(2);
    expect(m21.status).toBe("validating");
    expect(m21.features.map((f) => f.id)).toEqual(Array.from({ length: 13 }, (_, i) => `F-21.${i + 1}`));
    for (const id of [ "f-21.1", "f-21.2", "f-21.3", "f-21.4", "f-21.5", "f-21.6", "f-21.8", "f-21.9", "f-21.10", "f-21.12", "f-21.13" ]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const fhir = findFeature("f-21.7")!.feature;
    expect(fhir.status).toBe("validating");
    expect(fhir.note).toMatch(/laboratório simulado/);
    expect(findFeature("f-21.1")?.feature.note).toMatch(/fora de produção/);
    const pec = findFeature("f-21.11")!.feature;
    // Prova técnica do PEC: mantenha só a linha do resultado real (plano do api, Task 0).
    expect(pec.status).toBe("validating"); expect(pec.note).toMatch(/e-SUS PEC real/);   // prova passou
    // expect(pec.status).toBe("planned"); expect(pec.note).toMatch(/recepção/);       // prova falhou
  });
});
```

Em `src/App.test.tsx`, no teste "o rodapé resume o estado dos módulos", troque a frase esperada pela que o JSON vai produzir com o 21 em validação — com os módulos 19 e 20 ainda em validação, é

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20 e o 21 estão em validação."
```

(se o 19 ou o 20 já tiverem virado `available` no `origin/main`, ajuste a faixa e a lista ao que o Step 1 anotou), e, em "agrupa os módulos pelo ciclo", troque `"módulos 15 a 20 e interface"` por `"módulos 15 a 21 e interface"`.

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run`
Expected: FAIL — `formatValidating` não existe; o módulo 21 não está no JSON; o rodapé e o ciclo com o texto antigo.

- [ ] **Step 4: Write minimal implementation**

Em `src/pages/ModuleList.tsx`, depois de `formatRanges`, acrescente

```ts
// "o 19 está", "o 19 e o 20 estão", "o 19, o 20 e o 21 estão".
export function formatValidating(ids: string[]): string {
  const items = ids.map((id) => `o ${id}`);
  const list = items.length > 1 ? `${items.slice(0, -1).join(", ")} e ${items[items.length - 1]}` : items[0] ?? "";
  return `${list} ${items.length > 1 ? "estão" : "está"}`;
}
```

e, no rodapé, troque

```tsx
        estão disponíveis e verificados{validating.length > 0 && `; ${validating.length > 1 ? `${validating.map((id) => `o ${id}`).join(" e ")} estão` : `o ${validating[0]} está`} em validação`}
```

por

```tsx
        estão disponíveis e verificados{validating.length > 0 && `; ${formatValidating(validating)} em validação`}
```

Em `src/data/funcionalidades.json`:

1. `meta.updatedAt` → a data do dia (`AAAA-MM-DD`); no Ciclo 2 de `meta.cycles`, `"modules": "15 a 21 e interface"`.
2. Em `modules`, logo depois do objeto do módulo `"20"` (e antes do `"iu"`), acrescente o módulo 21. **Os textos abaixo são o rascunho**: confira cada frase contra o que foi entregue (cards Verified, telas e regras do api) e corte o que não existir; nada de promessa.

```json
{
  "id": "21",
  "cycle": 2,
  "name": "Exames e laboratório",
  "status": "validating",
  "description": "O exame pedido é acompanhado até o resultado: autorização, marcação, coleta, laudo e a ciência de quem pediu, com prazos e aviso de atraso. Laboratórios mandam o resultado por um portal ou por integração, e o cidadão acompanha pelo celular. O laboratório municipal (coleta com etiqueta e laudo digitado) vem numa etapa seguinte.",
  "features": [
    {
      "id": "F-21.1", "slug": "f-21.1",
      "title": "Interruptores do rastreamento de exames",
      "benefit": "A cidade liga o rastreamento de exames quando quiser, e cada integração à parte.",
      "description": "O rastreamento, a entrada de resultados por integração com laboratórios e o pedido pelo PEC são ligados por cidade, pela equipe da plataforma, cada um com os seus pré-requisitos. A tela diz o que falta para cada um funcionar.",
      "surfaces": [ "plataforma", "bastidores" ],
      "status": "available",
      "note": "O laboratório simulado, usado para demonstração e testes, só existe fora de produção."
    },
    {
      "id": "F-21.2", "slug": "f-21.2",
      "title": "Catálogo de exames e prestadores da cidade",
      "benefit": "A cidade diz quais exames acompanha, quem os faz e em quanto tempo.",
      "description": "A administração municipal monta o catálogo a partir da tabela SIGTAP: quais exames exigem autorização, o prazo de cada etapa e o que o cidadão pode ver do resultado. Cadastra os prestadores — laboratório municipal, credenciado, consórcio, LACEN ou imagem — com prazos próprios por exame.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.3", "slug": "f-21.3",
      "title": "Exame acompanhado do pedido ao resultado",
      "benefit": "Cada exame pedido vira um acompanhamento com situação e histórico.",
      "description": "O pedido feito na consulta já nasce acompanhado, e a recepção cadastra o que chega pela guia em papel, com a foto da guia. Cada passo fica registrado com quem fez e quando; antes do resultado, o exame pode ser cancelado com motivo.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.4", "slug": "f-21.4",
      "title": "Fila de regulação",
      "benefit": "Exames que exigem autorização entram numa fila única da cidade.",
      "description": "O regulador decide na ordem: urgentes primeiro, depois os mais antigos. Ao autorizar escolhe o prestador; ao negar registra o motivo, e quem pediu fica sabendo.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.5", "slug": "f-21.5",
      "title": "Prazos e exames atrasados",
      "benefit": "A unidade e a coordenação veem o que está atrasado e em que etapa.",
      "description": "Cada etapa tem prazo — do prestador, do catálogo ou o padrão da plataforma — e o urgente conta a metade. Uma conferência diária marca os atrasos, mostrados num painel com filtros por etapa, prestador e exame.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.6", "slug": "f-21.6",
      "title": "Portal do prestador",
      "benefit": "O laboratório agenda, registra a coleta e envia o resultado pelo próprio Rota Saúde.",
      "description": "Cada prestador tem usuário próprio e vê só as suas guias, com o mínimo para identificar o paciente. Não acessa prontuário nem guias de outro prestador, e vê as coletas e resultados em atraso.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.7", "slug": "f-21.7",
      "title": "Resultado por integração com o laboratório",
      "benefit": "O laboratório com sistema próprio manda o resultado direto, sem digitação.",
      "description": "O laboratório envia o laudo em PDF e os valores no padrão FHIR, com um acesso próprio que a cidade pode revogar. O resultado só entra se o paciente bater com o do exame, e o mesmo envio repetido não duplica.",
      "surfaces": [ "bastidores" ],
      "status": "validating",
      "note": "Provada com um laboratório simulado. Falta o primeiro laboratório real antes de uma cidade usar."
    },
    {
      "id": "F-21.8", "slug": "f-21.8",
      "title": "Resultado, ciência e fila da equipe",
      "benefit": "Todo resultado é visto por um profissional, e nenhum fica esquecido.",
      "description": "Quem pediu o exame dá ciência do resultado. Se o prazo vence, ou se o resultado vem marcado como crítico, ele vai para a fila da equipe de referência do paciente, onde médico ou enfermeiro dá ciência com motivo.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.9", "slug": "f-21.9",
      "title": "Resultado externo",
      "benefit": "Exame feito fora da rede também entra no histórico do paciente.",
      "description": "O profissional ou a unidade registra um resultado que chegou sem pedido no Rota Saúde, com laboratório, data, PDF e valores. Quem registra já dá a ciência.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.10", "slug": "f-21.10",
      "title": "Meus exames, para o cidadão",
      "benefit": "A pessoa acompanha os próprios exames pelo celular.",
      "description": "No canal web, quem tem cadastro validado vê a situação de cada exame, a data e o local marcados e baixa o PDF do resultado quando a cidade libera: na hora, depois que o profissional viu, ou nunca pelo celular, com o aviso para retirar na unidade. Quem tem só cadastro declarado vê o convite para validar no posto.",
      "surfaces": [ "cidadao" ],
      "status": "available"
    },
    {
      "id": "F-21.11", "slug": "f-21.11",
      "title": "Pedido de exame pelo e-SUS PEC",
      "benefit": "Em cidade com e-SUS PEC, o profissional pede o exame sem sair do PEC.",
      "description": "Dentro do atendimento no PEC, o profissional abre o pedido de exames do Rota Saúde, com profissional e paciente já preenchidos pela chamada assinada do PEC. O exame nasce acompanhado, como os pedidos da consulta.",
      "surfaces": [ "secretaria" ],
      "status": "validating",
      "note": "Provado com o e-SUS PEC local. A prova num e-SUS PEC real é o que falta antes de a primeira cidade usar."
    },
    {
      "id": "F-21.12", "slug": "f-21.12",
      "title": "Relatório de conferência por prestador",
      "benefit": "A cidade confere mês a mês o que cada prestador entregou, e no prazo.",
      "description": "Por prestador e mês: guias autorizadas, coletadas e com resultado no prazo ou em atraso, na tela e em planilha. A planilha detalhada leva só o código da guia, o código SIGTAP e as datas.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.13", "slug": "f-21.13",
      "title": "Exames na linha do tempo",
      "benefit": "Os exames aparecem no histórico do paciente, para o profissional e para o cidadão.",
      "description": "O profissional vê o pedido, as etapas e o resultado na linha do tempo do paciente. O cidadão vê o andamento e o resultado conforme o que a cidade libera para cada exame.",
      "surfaces": [ "secretaria", "cidadao" ],
      "status": "available"
    }
  ]
}
```

Se a prova técnica do PEC **falhou** (F-21.11 fora do escopo), troque só o objeto do F-21.11 por

```json
    {
      "id": "F-21.11", "slug": "f-21.11",
      "title": "Pedido de exame pelo e-SUS PEC",
      "benefit": "Em cidade com e-SUS PEC, o profissional pediria o exame sem sair do PEC.",
      "description": "Planejado para quando o e-SUS PEC aceitar formulário na tela de sistemas externos. Até lá, a recepção cadastra o exame pela guia em papel.",
      "surfaces": [ "secretaria" ],
      "status": "planned",
      "note": "A prova no e-SUS PEC local mostrou que a tela de sistemas externos não aceita o envio; a recepção segue cadastrando pela guia."
    }
```

e, no teste do Step 2, deixe a linha "prova falhou" e apague a "prova passou".

3. **Textos que ficaram desatualizados:** na F-20.2, troque "Reúne triagens, atendimentos, acolhimento, agendamentos, consultas, documentos, Notificações de receita e campanhas" por "Reúne triagens, atendimentos, acolhimento, agendamentos, consultas, documentos, Notificações de receita, campanhas e exames". Confira as demais descrições dos módulos 19 e 20 que falam de exame (pedido na consulta, requisição em PDF) e acrescente só o que o 21a mudou de fato.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita o conteúdo (descrição em todos, telas conhecidas, slugs únicos, ciclo existente); o módulo 21, o `formatValidating`, o rodapé e o ciclo com os textos novos.

```bash
cd $C && $G status --short
```

Expected: só os quatro arquivos da task (nenhum `package-lock.json`).

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/pages/ModuleList.tsx src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 21a exam tracking to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot: o Ciclo 2 com "módulos 15 a 21 e interface"; o **Módulo 21 — Exames e laboratório** "Em validação", com as 13 funcionalidades e a contagem; a F-21.7 com "Em validação" e a nota do laboratório simulado; a F-21.11 conforme a prova; a página de uma funcionalidade (`/funcionalidades/f-21.10`) com "O que faz", as telas e anterior/próxima dentro do módulo; o rodapé "…; o 19, o 20 e o 21 estão em validação." (ou o que o Step 1 anotou); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: o merge é manual, com a CI (`vitest`) verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod21
/opt/homebrew/bin/git push -u origin content/module-21a-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-21a-catalog --title "docs: add module 21a exam tracking to the catalog" --body "Adds module 21 (exam tracking, 21a) to Cycle 2 with F-21.1..F-21.13; FHIR inbound in validation until the first real lab; footer lists three modules in validation with commas.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod21 && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-21a-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — `non_production: true` × `environments:`.** O contrato §11 marca `fake_lab` com `non_production: true`; o catálogo de interruptores do api não tem esse campo — usa `environments: SIMULATION_ENVS` (`development test staging`), como `signature_psc_mock` e `sncr_mock`. Proposta: o contrato passa a dizer `environments: SIMULATION_ENVS`. O efeito é o mesmo que o maintenance espera: em produção a chave não aparece no catálogo e `setCityFeature` responde `unknown_feature`.
2. **M2 — Os códigos de `missing`.** O contrato diz "mensagens de `missing` pelo mecanismo genérico" sem nomeá-los. Pelo `Platform::Features#missing_for`, `feature:<x>` vira `<x>_disabled`: os códigos novos são `exam_tracking_disabled`, `exam_fhir_inbound_disabled` e `pec_external_view_disabled` (o `clinical_record_disabled` já existia). Proposta: o contrato registra os três, para o maintenance e o api não divergirem num nome.
3. **M3 — Aviso do laboratório simulado.** O contrato §11 não pede aviso para `fake_lab`. Proposta: o maintenance avisa pelo estado **ligado**, como os simulados do PSC (19b) e do SNCR (19d), porque em staging o `fake-lab` escreve laudos de exemplo em pacientes da cidade. Só texto na aba, sem consulta nova.

## Self-review

- **Cobertura (spec §3, §9 "Maintenance", §10 itens 5 e 6, §11 F-21.1; contrato §11):** os quatro interruptores pelo mecanismo genérico, com os `requires` do contrato — Tasks 1 e 2 (fluxo de dois cliques sem mudar o código do liga/desliga); mensagens de `missing` — Task 1 (e Task 2 na lista); `fake_lab` só fora de produção — Task 2 ("catálogo sem fake_lab") e Divergência M1; prova no navegador — Task 3; entrada do módulo 21a no catálogo público como task final, JSON como fonte única — Task 4; cards — Tasks 0 e 3.
- **Placeholders:** nenhum código faltando. As variáveis são de ambiente (ids do card e da opção In Progress, lidos na Task 0; o resultado da prova técnica do PEC e o status dos módulos 19 e 20 no catálogo, lidos na Task 4, Step 1, com as duas formas do F-21.11 escritas por inteiro).
- **Tipos:** `FAKE_LAB_KEY`, `FAKE_LAB_NOTICE`, `fakeLabOn` (Task 1) são os da `FeaturesTab` e dos testes (Task 2); os rótulos e frases da Task 1 são os textos que os testes da Task 2 procuram; `formatValidating(ids)` (Task 4) é o do rodapé e do teste.
- **Review Focus:** 1 — Task 2 ("laboratório simulado ligado…", "ligar fake_lab por dois cliques…"); 2 — Task 2 ("catálogo sem fake_lab (produção)…"); 3 — Task 1 ("pré-requisitos do 21a têm frase") e Task 2 ("os pré-requisitos do 21a aparecem em português…", "exam_tracking pelo mecanismo genérico…"); 4 — Task 4 ("módulo 21: entrada FHIR em validação e o PEC conforme a prova"); 5 — Task 4 ("formatValidating…", rodapé no `App.test.tsx`).
