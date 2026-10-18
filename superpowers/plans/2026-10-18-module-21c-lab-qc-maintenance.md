# Módulo 21c — Qualidade e insumos do laboratório municipal (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21c** na branch `feat/mod-21c-lab-qc` (worktree `apps/api/.claude/mod21c`, servidor de dev na porta **3043**) com `lab_quality_control` e `lab_inventory` em `Platform::Features::CATALOG`, ambos com `requires: %w[feature:municipal_lab]`. Contrato §10: contracts → api → dashboard → **maintenance** → catálogo público. As Tasks 0–2 não dependem do api (testes com `fetch` falso; o maintenance não ganha consulta nem schema novo); a prova da Task 3 precisa do api do 21c rodando. A Task 4 (catálogo público) vem **por último**, depois da prova e das entregas de api, dashboard e maintenance.

**Goal:** Lado maintenance de F-21.25: os interruptores `lab_quality_control` e `lab_inventory` aparecem na aba Funcionalidades com nome e pré-requisito em português e ligam pelo mecanismo genérico; o aviso do middleware simulado passa a dizer que ele também manda corridas de controle. E as F-21.25..F-21.34 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única).

**Architecture:** Nada muda no liga/desliga (`setCityFeature`, três estados, `missing` vindo do api). Entram dois rótulos em `FEATURE_LABELS` de `src/lib/features.ts`; a frase de `municipal_lab_disabled` **já existe** (21b) e só é fixada por teste para as chaves novas. O texto `FAKE_MIDDLEWARE_NOTICE` (`src/lib/lab.ts`) ganha a menção às corridas de controle (Divergência M2). Nenhuma tela, consulta, documento GraphQL ou componente novo. Catálogo público: dez funcionalidades no módulo `"21"` e a descrição do módulo atualizada, por um script que reescreve o JSON no mesmo formato.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-18-module-21c-lab-quality-supplies-design.md` (§3 "Interruptores", §7 último item — `fake-middleware` com controles, §11 itens 4 e 5, §12 a tabela inteira para o catálogo, §13 riscos), `docs/adr/0038.md` e o contrato `docs/superpowers/plans/2026-10-18-module-21c-lab-qc-contracts.md` §8 e §10 (as diferenças estão em "Divergências propostas ao contrato"). Base: plano do maintenance do 21b (`2026-10-15-module-21b-lab-maintenance.md`, Tasks 1, 2 e 7) e lições do contrato do 21a (§15, A25 — aviso do simulado) e do 21b (§14).

## Global Constraints

- Interruptores (contrato §8; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`, que transforma `feature:<x>` em `<x>_disabled`):
  - `lab_quality_control` com `requires: ["feature:municipal_lab"]`; faltando, `missing: ["municipal_lab_disabled"]`;
  - `lab_inventory` com `requires: ["feature:municipal_lab"]`; faltando, `missing: ["municipal_lab_disabled"]`;
  - **independentes entre si**: um nunca aparece como pré-requisito do outro; existem em todos os ambientes (sem `environments`), produção inclusive (Divergência M1).
  - Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados). Nenhuma linha nova no fluxo de `setCityFeature`.
- A frase de `municipal_lab_disabled` é a do 21b — "laboratório municipal (municipal_lab) desligado" — e **não** é redefinida.
- Nomes: `lab_quality_control` → "Controle de qualidade do laboratório (CIQ e CEQ)"; `lab_inventory` → "Estoque de reagentes e insumos do laboratório".
- Aviso do middleware simulado: continua pelo estado **ligado** de `fake_middleware`, com o mesmo nome ("modo do middleware"); o texto passa a dizer que o controle de qualidade também recebe corridas de exemplo (Divergência M2). Os interruptores do 21c **não** têm aviso próprio (não são simulados).
- Nenhum token de middleware, mensagem HL7, valor de controle, valor de resultado, CPF, CNS ou PDF passa pelo maintenance.
- Valor desconhecido (api mais novo: chave ou código de `missing`) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): só a `FeaturesTab` existente (`Panel`, `DataTable`, `Tag`, `ConfirmButton`), sem redesign nem componente novo; cores só por papel do tema; a guarda `src/theme/guards.test.ts` roda na suíte; tabelas largas em `<TableScroll>` (o `DataTable` já embrulha). Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); nada provado só com simulador aparece como "Disponível e verificado" sem nota — **F-21.28 fica `validating` até haver middleware real**; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-21.25..F-21.34 do board #1 em In Progress (Task 0). O maintenance só é dono de parte do F-21.25: Done do F-21.25 só quando api, dashboard e maintenance dele estiverem entregues.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod21c`, `C=$R/.worktrees/catalog-mod21c`, `A=$R/apps/api/.claude/mod21c`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-21c-lab-qc`; catálogo em `$C`, branch `content/module-21c-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task. Worktrees antigos (`$R/.worktrees/maintenance-mod21b` etc.) não são tocados por este plano.
- maintenance — testes: `cd $M && npx vitest run <arquivos>`; tipos: `npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (todo teste de componente chama `cleanup` no `afterEach`; o `FeaturesTab.test.tsx` já faz no topo), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- O api do 21c roda em dev na **3043** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **Desligar `municipal_lab` com os dois interruptores do 21c ligados.** Depois do refetch, os dois passam a "ligada, falta pré-requisito" com "laboratório municipal (municipal_lab) desligado" — nunca seguem como "ligada e utilizável" por cache. Teste: Task 2, "desligar municipal_lab: depois do refetch os dois do 21c dizem o que falta".
2. **Estoque ligado sem controle de qualidade (e vice-versa).** São independentes: `lab_inventory` ligado e utilizável com `lab_quality_control` desligado, e nada na linha do estoque fala de controle de qualidade. Teste: Task 2, "independentes: estoque utilizável com o controle de qualidade desligado".
3. **Produção.** Os dois interruptores do 21c aparecem (não são simulados) e nenhum aviso de simulado aparece por causa deles; sem `fake_middleware` no catálogo, nada diz "MIDDLEWARE SIMULADO". Teste: Task 2, "produção: os dois do 21c aparecem e nenhum aviso de simulado".
4. **Api mais novo com um pré-requisito a mais** (por exemplo, um aparelho cadastrado como exigência do CIQ, código que esta tela não conhece). O código aparece cru ao lado da frase conhecida, nunca `undefined` nem a linha some. Teste: Task 2, "código de missing desconhecido aparece cru ao lado do conhecido".
5. **Staging com `fake_middleware` ligado depois do 21c**: o simulado também manda corridas de controle — inclusive reprovadas, que bloqueiam liberação — e quem liga precisa ler isso no aviso. Testes: Task 1, "o aviso do middleware menciona as corridas de controle"; Task 2, "com fake_middleware ligado, o aviso fala do controle de qualidade".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-21.25..F-21.34, conferência do main e do api do 21c, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/lab.ts`, `src/lib/lab.test.ts` | nomes das duas chaves; frase de `municipal_lab_disabled` fixada para elas; aviso do middleware com controles | 1 |
| `src/screens/FeaturesTab.test.tsx`, `README.md` | os dois interruptores pelo fluxo genérico da aba (sem mudar `FeaturesTab.tsx`) | 2 |
| — | suíte, codegen:check, build, revisão, prova no navegador | 3 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts` | F-21.25..F-21.34 no módulo 21 e a descrição do módulo | 4 |

**Estratégia de teste:** funções puras (Task 1: `featureLabel`, `missingText`, `missingSummary`, `FAKE_MIDDLEWARE_NOTICE`); os dois interruptores pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation; o refetch depois de desligar o pré-requisito; independência; produção; código desconhecido (Task 2); suíte inteira com a guarda de tema, `codegen:check` e build, e prova no navegador contra o api do 21c (Task 3). Catálogo: `validateCatalog` e o teste do módulo 21 (Task 4).

---

### Task 0: Cards, conferência do main e do api do 21c, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`c3a534d` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-21.25..F-21.34 em In Progress (ou além); worktree `$M` na branch `feat/mod-21c-lab-qc`; a base anotada.

- [ ] **Step 1: Mova os cards F-21.25..F-21.34 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | test("^F-21\\.(2[5-9]|3[0-4]) ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified (o plano do api move os mesmos cards) ficam como estão. Para cada um que estiver antes de In Progress, com o `id` do item e o id da opção "In Progress":

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os dez itens com `status` In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main do maintenance e o api do 21c**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod21c
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n \
  '  municipal_lab_disabled: "laboratório municipal (municipal_lab) desligado",\|  fake_middleware: "Middleware simulado (desenvolvimento)"\|lab_quality_control\|lab_inventory'
$G -C $R/apps/maintenance show origin/main:src/lib/lab.ts | grep -n 'ganham valores de exemplo pela HL7, sem validade clínica (fora de produção)'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n 'featureLabel(f.key)\|missingSummary(f.missing)\|confirmar: ligar ${f.key}'
grep -n -E 'key: "(lab_quality_control|lab_inventory)"' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: as duas âncoras (a frase de `municipal_lab_disabled` já existe, do 21b) e **nenhuma** linha com `lab_quality_control` ou `lab_inventory`;
- em `lab.ts`: a âncora do texto do aviso; em `FeaturesTab.tsx`: as três âncoras do fluxo genérico. Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 21c: as duas entradas com `requires: %w[feature:municipal_lab]` e **sem** `environments:`. Se o api ainda não tiver as entradas, as Tasks 0–2 seguem; **a Task 3 espera** o api. Se o api tiver outro `requires` ou um `environments`, **pare e reporte** (divergência com o contrato §8).

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod21c -b feat/mod-21c-lab-qc origin/main
cd $R/.worktrees/maintenance-mod21c && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera **o mesmo número de arquivos de teste** (nenhum arquivo novo) e **+12 testes** (3 em `features.test.ts`, 1 em `lab.test.ts`, 8 em `FeaturesTab.test.tsx`).

---

### Task 1: Nomes dos interruptores, a frase do pré-requisito e o aviso do middleware com controles

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/lab.ts`
- Test: `src/lib/features.test.ts`, `src/lib/lab.test.ts`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes em `features.ts`); `FAKE_MIDDLEWARE_NOTICE`, `fakeMiddlewareOn` (existentes em `lab.ts`).
- Produces:
  - `featureLabel("lab_quality_control")` → "Controle de qualidade do laboratório (CIQ e CEQ)"; `featureLabel("lab_inventory")` → "Estoque de reagentes e insumos do laboratório";
  - `missingText("municipal_lab_disabled")` → "laboratório municipal (municipal_lab) desligado" (sem mudança);
  - `FAKE_MIDDLEWARE_NOTICE` com "corridas de controle de qualidade de exemplo"; mesma assinatura, mesmo `fakeMiddlewareOn`.

- [ ] **Step 1: Write the failing test**

Em `src/lib/features.test.ts`, no fim do arquivo:

```ts
describe("módulo 21c", () => {
  it("as duas chaves do 21c têm nome em português", () => {
    expect(featureLabel("lab_quality_control")).toBe("Controle de qualidade do laboratório (CIQ e CEQ)");
    expect(featureLabel("lab_inventory")).toBe("Estoque de reagentes e insumos do laboratório");
  });

  it("o pré-requisito das duas é a frase do 21b, sem redefinir", () => {
    // Contrato 21c §8: as duas exigem municipal_lab; o api manda municipal_lab_disabled.
    expect(missingText("municipal_lab_disabled")).toBe("laboratório municipal (municipal_lab) desligado");
    expect(missingSummary([ "municipal_lab_disabled" ])).toBe("laboratório municipal (municipal_lab) desligado");
  });

  it("nenhuma das duas é pré-requisito da outra: não existe frase para elas como missing", () => {
    // São independentes (spec §3). Se um dia o api mandar este código, ele aparece cru.
    expect(missingText("lab_quality_control_disabled")).toBe("lab_quality_control_disabled");
    expect(missingText("lab_inventory_disabled")).toBe("lab_inventory_disabled");
  });
});
```

Em `src/lib/lab.test.ts`, dentro do `describe("fakeMiddlewareOn", …)`, depois do último `it`:

```ts
  it("o aviso do middleware menciona as corridas de controle (21c), sem perder o que já dizia", () => {
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/corridas de controle de qualidade de exemplo/);
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/inclusive reprovadas/);
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/^Esta cidade recebe resultados do MIDDLEWARE SIMULADO — /);
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/sem validade clínica \(fora de produção\)/);
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/nenhum valor sai do laboratório sem o liberador\.$/);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/lib/lab.test.ts`
Expected: FAIL — `featureLabel("lab_quality_control")` é `null` e o aviso não fala de corridas de controle. (Os testes "o pré-requisito das duas…" e "nenhuma das duas…" já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `MISSING_LABELS`, troque

```ts
  // Módulo 21b: municipal_lab exige exam_tracking (frase acima);
  // lab_hl7_inbound exige municipal_lab; fake_middleware exige lab_hl7_inbound.
```

por

```ts
  // Módulo 21b: municipal_lab exige exam_tracking (frase acima);
  // lab_hl7_inbound exige municipal_lab; fake_middleware exige lab_hl7_inbound.
  // Módulo 21c: lab_quality_control e lab_inventory exigem municipal_lab
  // (municipal_lab_disabled, abaixo) e são independentes entre si.
```

e, em `FEATURE_LABELS`, troque

```ts
  fake_middleware: "Middleware simulado (desenvolvimento)"
};
```

por

```ts
  fake_middleware: "Middleware simulado (desenvolvimento)",
  // Módulo 21c (ADR 0038): existem em todos os ambientes; nenhum é simulado.
  lab_quality_control: "Controle de qualidade do laboratório (CIQ e CEQ)",
  lab_inventory: "Estoque de reagentes e insumos do laboratório"
};
```

Em `src/lib/lab.ts`, troque

```ts
export const FAKE_MIDDLEWARE_NOTICE =
  "Esta cidade recebe resultados do MIDDLEWARE SIMULADO — as amostras recebidas no laboratório municipal ganham valores de exemplo pela HL7, sem validade clínica (fora de produção); nenhum valor sai do laboratório sem o liberador.";
```

por

```ts
// Módulo 21c (Divergência M2): o fake-middleware também manda corridas de
// controle — uma reprovada (2-2s) abre bloqueio de liberação no laboratório.
export const FAKE_MIDDLEWARE_NOTICE =
  "Esta cidade recebe resultados do MIDDLEWARE SIMULADO — as amostras recebidas no laboratório municipal ganham valores de exemplo pela HL7, e o controle de qualidade recebe corridas de controle de qualidade de exemplo, inclusive reprovadas, sem validade clínica (fora de produção); nenhum valor sai do laboratório sem o liberador.";
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/lib/lab.test.ts src/screens/FeaturesTab.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos seguem iguais; os da aba comparam com a constante, não com o texto literal).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/lib/lab.ts src/lib/lab.test.ts
$G commit -m "feat: name the lab quality control and inventory switches

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Os dois interruptores pelo fluxo genérico da aba

**Files:**
- Modify: `README.md`
- Test: `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo, e relê a lista depois de cada `setCityFeature`. Esta task fixa esse comportamento para as duas chaves novas.

**Interfaces:**
- Consumes: rótulos e aviso (Task 1); `FeaturesTab`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (existentes no topo de `FeaturesTab.test.tsx`); `FAKE_MIDDLEWARE_NOTICE` (já importado no arquivo).
- Produces: nada novo para outras tasks.

- [ ] **Step 1: Write the failing test**

Em `src/screens/FeaturesTab.test.tsx`, no fim do arquivo:

```tsx
describe("FeaturesTab — qualidade e estoque do laboratório (módulo 21c)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const LAB_ON: Feature = {
    key: "municipal_lab", description: "Laboratório municipal: fluxo analítico", enabled: true, usable: true,
    missing: [], changedAt: "2026-10-18T12:00:00Z", changedBy: "dev@local"
  };
  const QC_OFF: Feature = {
    key: "lab_quality_control", description: "Controle de qualidade do laboratório: CIQ e CEQ", enabled: false, usable: false,
    missing: [ "municipal_lab_disabled" ], changedAt: null, changedBy: null
  };
  const INV_OFF: Feature = {
    key: "lab_inventory", description: "Estoque de reagentes e insumos do laboratório", enabled: false, usable: false,
    missing: [ "municipal_lab_disabled" ], changedAt: null, changedBy: null
  };
  const QC_ON: Feature = { ...QC_OFF, enabled: true, usable: true, missing: [], changedAt: "2026-10-18T13:00:00Z", changedBy: "dev@local" };
  const INV_ON: Feature = { ...INV_OFF, enabled: true, usable: true, missing: [], changedAt: "2026-10-18T13:00:00Z", changedBy: "dev@local" };
  const MW_ON: Feature = {
    key: "fake_middleware", description: "Middleware simulado do compose de dev", enabled: true, usable: true,
    missing: [], changedAt: "2026-10-18T13:00:00Z", changedBy: "dev@local"
  };

  it("os dois aparecem com nome em português e o pré-requisito do laboratório municipal", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ QC_OFF, INV_OFF ])));
    renderTab();

    expect(await screen.findByText("lab_quality_control")).not.toBeNull();
    expect(screen.getByText("lab_inventory")).not.toBeNull();
    expect(screen.getByText("Controle de qualidade do laboratório (CIQ e CEQ)")).not.toBeNull();
    expect(screen.getByText("Estoque de reagentes e insumos do laboratório")).not.toBeNull();
    expect(screen.getAllByText("laboratório municipal (municipal_lab) desligado")).toHaveLength(2);
    expect(document.body.textContent).not.toMatch(/_disabled|undefined|null/);
  });

  it("lab_quality_control liga com dois cliques: sem o laboratório, ligada e com o que falta", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...QC_OFF, enabled: true, changedAt: "2026-10-18T13:00:00Z", changedBy: "dev@local" };
    let listed = [ QC_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("lab_quality_control")).not.toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar lab_quality_control" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "lab_quality_control", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "lab_quality_control ligada, mas ainda não utilizável — falta: laboratório municipal (municipal_lab) desligado."
    );
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
  });

  it("lab_inventory liga com dois cliques: com o laboratório ligado, utilizável", async () => {
    const user = userEvent.setup();
    let listed = [ LAB_ON, INV_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ LAB_ON, INV_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: INV_ON } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("lab_inventory")).not.toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar lab_inventory" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "lab_inventory", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe("lab_inventory ligada e utilizável.");
  });

  it("desligar municipal_lab: depois do refetch os dois do 21c dizem o que falta", async () => {
    const user = userEvent.setup();
    const labOff: Feature = { ...LAB_ON, enabled: false, usable: false, missing: [], changedAt: "2026-10-18T14:00:00Z" };
    let listed = [ LAB_ON, QC_ON, INV_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [
          labOff,
          { ...QC_ON, usable: false, missing: [ "municipal_lab_disabled" ] },
          { ...INV_ON, usable: false, missing: [ "municipal_lab_disabled" ] }
        ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: labOff } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("lab_quality_control")).not.toBeNull();
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(3);
    await user.click(screen.getAllByRole("button", { name: "desligar" })[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar municipal_lab" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "municipal_lab", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("municipal_lab desligada.");
    await waitFor(() => expect(screen.getAllByText("ligada, falta pré-requisito")).toHaveLength(2));
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(screen.getAllByText("laboratório municipal (municipal_lab) desligado")).toHaveLength(2);
  });

  it("independentes: estoque utilizável com o controle de qualidade desligado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      LAB_ON, { ...QC_OFF, missing: [] }, INV_ON
    ])));
    renderTab();

    expect(await screen.findByText("lab_inventory")).not.toBeNull();
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(2);
    expect(screen.getByText("desligada")).not.toBeNull();
    // Nenhuma linha diz que falta o controle de qualidade, nem o laboratório.
    expect(screen.queryByText(/municipal_lab\) desligado/)).toBeNull();
    expect(document.body.textContent).not.toMatch(/lab_quality_control_disabled|\(lab_quality_control\) desligad/);
  });

  it("produção: os dois do 21c aparecem e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LAB_ON, QC_ON, INV_ON ])));
    renderTab();

    expect(await screen.findByText("lab_quality_control")).not.toBeNull();
    expect(screen.getByText("lab_inventory")).not.toBeNull();
    expect(screen.queryByText("fake_middleware")).toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });

  it("código de missing desconhecido aparece cru ao lado do conhecido", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      { ...QC_OFF, enabled: true, missing: [ "municipal_lab_disabled", "lab_instrument_missing" ] }
    ])));
    renderTab();

    expect(await screen.findByText(
      "laboratório municipal (municipal_lab) desligado; lab_instrument_missing"
    )).not.toBeNull();
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("com fake_middleware ligado, o aviso fala do controle de qualidade", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LAB_ON, QC_ON, MW_ON ])));
    renderTab();

    const note = await screen.findByRole("note", { name: "modo do middleware" });
    expect(note.textContent).toBe(FAKE_MIDDLEWARE_NOTICE);
    expect(note.textContent).toMatch(/corridas de controle de qualidade de exemplo/);
  });
});
```

- [ ] **Step 2: Run test to verify it passes já com a Task 1 — e falha sem ela**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx`
Expected: PASS — a aba é genérica; o que estes testes dependem de novo (rótulos, aviso) veio da Task 1. Para ver que eles pegam a regressão, rode uma vez com `features.ts` e `lab.ts` de antes da Task 1 e devolva em seguida:

```bash
cd $M && $G show origin/main:src/lib/features.ts > src/lib/features.ts && $G show origin/main:src/lib/lab.ts > src/lib/lab.ts && npx vitest run src/screens/FeaturesTab.test.tsx 2>&1 | grep -E 'Tests|FAIL' ; $G checkout -- src/lib/features.ts src/lib/lab.ts && $G status --short src/lib
```

Expected: FAIL em "os dois aparecem com nome em português…" (não acha "Controle de qualidade do laboratório (CIQ e CEQ)") e em "com fake_middleware ligado, o aviso fala do controle de qualidade"; depois do `checkout`, o `status` de `src/lib` vazio (a Task 1 commitada de volta) e a suíte da aba volta a passar.

Se algum teste falhar **com** a Task 1 aplicada, o comportamento genérico mudou no main: leia `FeaturesTab.tsx` e ajuste o **teste** à intenção (o maintenance não ganha código novo para isso); se a falha for no refetch depois de desligar, **pare e reporte** — é o Review Focus 1.

- [ ] **Step 3: README**

Em `README.md`, logo antes do parágrafo **Documentos clínicos (módulo 19c).**, acrescente (depois do parágrafo da tela **Laboratório** do 21b):

```markdown
**Qualidade e estoque do laboratório (módulo 21c).** Os interruptores
`lab_quality_control` ("Controle de qualidade do laboratório", CIQ e CEQ) e
`lab_inventory` ("Estoque de reagentes e insumos do laboratório") exigem
`municipal_lab` e são independentes entre si; aparecem na mesma lista, em
todos os ambientes, e ligam como os outros, com o que falta em português.
Aparelhos, materiais e lotes de controle, corridas, bloqueios, exceções,
CEQ e estoque são do laboratório da cidade, no dashboard; nenhum valor de
controle ou de resultado passa por aqui. Com `fake_middleware` ligado, o
aviso diz que o simulado também manda corridas de controle de exemplo,
inclusive reprovadas. Ordem de deploy: api → dashboard → maintenance; contra
um api sem o 21c as chaves só não aparecem.
```

- [ ] **Step 4: Run tests**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx src/theme/guards.test.ts && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/FeaturesTab.test.tsx README.md
$G commit -m "test: pin the lab quality control and inventory switches in the features tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-21c-lab-qc`; o api do 21c na **3043**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 21c).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd $M && npm run typecheck && npx vitest run && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base anotada na Task 0, **mesmo número de arquivos de teste** e **+12 testes**. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e contagem `0`.

O maintenance não usa SDL novo no 21c, mas o `schema.graphql` do worktree precisa seguir igual ao do api (o api pode ter mexido no `Maintenance::Schema`):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml exec -T -w /app/.claude/mod21c api bin/rails runner 'puts Maintenance::Schema.to_definition' | diff - $M/schema.graphql && echo schema-igual
```

Expected: `schema-igual`. Se diferir, **pare e reporte** (o contrato §8 não prevê mudança no schema do maintenance; não puxe nem edite à mão sem decisão).

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §7, §11 item 4, §12 F-21.25), o ADR 0038 e o contrato §8. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; a frase de `municipal_lab_disabled` não foi redefinida;
- os dois rótulos exatamente como no Global Constraints; nenhuma frase que trate um interruptor do 21c como pré-requisito do outro;
- o aviso do middleware continua pelo estado ligado, com o nome "modo do middleware", e fala das corridas de controle (Divergência M2);
- `schema.graphql` igual ao `to_definition`; `src/gql/*` sem mudança;
- nenhum valor de controle, valor de resultado, token, CPF ou CNS na tela; cores só por papel do tema.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 21c na **3043** (plano do api). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, com `node_modules` num volume próprio, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod21c --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod21c:/app -v maintenance-mod21c-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3043 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod21c   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 21c num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. **Anote** antes de mexer o estado de `municipal_lab`, `lab_quality_control`, `lab_inventory` e `fake_middleware` de Curitiba. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: as duas chaves com o nome em português; com `municipal_lab` desligado, as duas com "laboratório municipal (municipal_lab) desligado";
- ligar `lab_quality_control` (dois cliques) com `municipal_lab` desligado → "lab_quality_control ligada, mas ainda não utilizável — falta: laboratório municipal (municipal_lab) desligado.";
- ligar `municipal_lab` → depois do refetch, `lab_quality_control` "ligada e utilizável"; `lab_inventory` desligado segue "desligada" e sem nada a ver com o controle de qualidade; ligar `lab_inventory` → "lab_inventory ligada e utilizável.";
- desligar `municipal_lab` → as duas do 21c em "ligada, falta pré-requisito";
- com `fake_middleware` ligado, o aviso "Esta cidade recebe resultados do MIDDLEWARE SIMULADO — … corridas de controle de qualidade de exemplo, inclusive reprovadas …";
- **Auditoria** mostra as tentativas e os resultados de `setCityFeature`;
- a 375 px de largura: sem rolagem lateral da página; a tabela rola dentro do contêiner com a dica "⇆ role para o lado";
- nenhum valor de controle, valor de resultado, token, CPF ou CNS na tela.

Depois: devolva os quatro interruptores ao estado anotado (pela própria tela); `docker rm -f maintenance-mod21c && docker volume rm maintenance-mod21c-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 21c, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21c-lab-qc
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod21c
/opt/homebrew/bin/git branch -d feat/mod-21c-lab-qc
```

Se os lados api e dashboard do F-21.25 já estiverem **Done**, mova o card F-21.25 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0.

---

### Task 4: O 21c no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`

**Interfaces:**
- Consumes: os F-IDs F-21.25..F-21.34 no board #1 (status real de cada um); o resultado da prova do 21c (planos do api, dashboard e deste); o que foi entregue em api, dashboard e maintenance.
- Produces: módulo `"21"` do JSON com F-21.1..F-21.34 e a descrição do módulo com controle de qualidade e estoque.

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | test("^F-21\\.(2[5-9]|3[0-4]) ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod21c -b content/module-21c-catalog origin/main
cd $R/.worktrees/catalog-mod21c && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-21.25..F-21.34 em Done/Verified (os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário); a suíte do catálogo verde. **Anote**: o módulo 21 do JSON com F-21.1..F-21.24 (`4ac9981` ou mais novo) e `status: "validating"`; o rodapé não muda (o 21 já está em validação).

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("módulo 21", …)`, troque

```ts
    expect(m21.features.map((f) => f.id)).toEqual(Array.from({ length: 24 }, (_, i) => `F-21.${i + 1}`));
```

por

```ts
    expect(m21.features.map((f) => f.id)).toEqual(Array.from({ length: 34 }, (_, i) => `F-21.${i + 1}`));
```

e, ainda dentro desse `describe`, depois do `it("21b, laboratório municipal: …")`, acrescente:

```ts
  it("21c, qualidade e insumos: só o controle pela HL7 em validação, até haver middleware real", () => {
    const m21 = catalog.modules.find((m) => m.id === "21")!;
    expect(m21.description).toMatch(/controle de qualidade/);
    expect(m21.description).toMatch(/lote do reagente/);
    for (const id of ["f-21.25", "f-21.26", "f-21.27", "f-21.29", "f-21.30", "f-21.31", "f-21.32", "f-21.33", "f-21.34"]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const hl7 = findFeature("f-21.28")!.feature;
    expect(hl7.status).toBe("validating");
    expect(hl7.note).toMatch(/middleware simulado/);
    expect(hl7.note).toMatch(/middleware real/);
    expect(hl7.surfaces).toEqual([ "bastidores" ]);
    expect(findFeature("f-21.25")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    expect(findFeature("f-21.27")?.feature.note).toMatch(/responsável técnico/);
    expect(findFeature("f-21.34")?.feature.description).toMatch(/nada é baixado/);
  });
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts`
Expected: FAIL — o módulo 21 tem 24 funcionalidades e a descrição sem controle de qualidade.

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo publicado byte a byte (conferido em `4ac9981`), então o diff fica só no que mudou. **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard e regras do api) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

m21 = next(m for m in data["modules"] if m["id"] == "21")
assert [f["id"] for f in m21["features"]] == [f"F-21.{i}" for i in range(1, 25)], "módulo 21 não termina no F-21.24"

data["meta"]["updatedAt"] = today
m21["description"] = (
    "O exame pedido é acompanhado até o resultado: autorização, marcação, coleta, laudo e a ciência de quem pediu, "
    "com prazos e aviso de atraso. Laboratórios mandam o resultado por um portal ou por integração, e o cidadão "
    "acompanha pelo celular. O laboratório municipal tem o fluxo inteiro: requisição com etiqueta, coleta na UBS, "
    "remessa, recebimento, mapa de trabalho, resultado dos aparelhos ou digitado, liberação assinada e laudo. "
    "O controle de qualidade reprovado bloqueia a liberação, e cada resultado guarda o lote do reagente usado."
)

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-21.{n}", "slug": f"f-21.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m21["features"] += [
    feature(25, "Interruptores e aparelhos do laboratório",
        "O controle de qualidade e o estoque entram por cidade, e cada exame sabe em que aparelho é feito.",
        "A equipe da plataforma liga, por cidade, o controle de qualidade e o estoque do laboratório municipal, "
        "independentes entre si. A administração cadastra os aparelhos — fabricante, modelo, série e setor — "
        "e liga cada exame ao aparelho que o faz.",
        ["plataforma", "secretaria"]),
    feature(26, "Materiais, lotes e referências de controle",
        "Cada lote de controle tem a sua referência, da bula ou do próprio laboratório.",
        "O laboratório cadastra os materiais de controle com os seus níveis, e cada lote com validade e uma etiqueta "
        "por nível. A referência começa pela bula; depois de 20 corridas aprovadas, o sistema sugere a média e o desvio "
        "do próprio laboratório, e o responsável técnico aceita uma nova versão, sem apagar a anterior.",
        ["secretaria"]),
    feature(27, "Corridas de controle e regras de Westgard",
        "O controle de cada corrida é avaliado na hora, sem conta à mão.",
        "O valor do controle é digitado na corrida do aparelho ou chega pelo middleware, e é avaliado na gravação "
        "pelas regras de Westgard: o aviso não bloqueia, a reprovação reprova a corrida. O responsável técnico pode "
        "desligar uma regra por analito, sempre com justificativa registrada.",
        ["secretaria"],
        note="A validação do controle de qualidade pelo responsável técnico do primeiro laboratório é feita antes do go-live."),
    feature(28, "Controle pela HL7",
        "O resultado do controle chega do aparelho sem digitação e nunca vira resultado de paciente.",
        "O middleware do aparelho manda o resultado do controle pelo mesmo padrão HL7 dos pacientes; o sistema "
        "reconhece o lote de controle pela etiqueta e o aparelho pelo seu código. O valor vai para a corrida e é "
        "avaliado, e lote vencido ou encerrado, controle sem referência ou aparelho desconhecido é recusado.",
        ["bastidores"], status="validating",
        note="Verificado com um middleware simulado, que manda corridas aprovadas e reprovadas. Falta o primeiro middleware real mandando controles, na convenção do fornecedor, antes do go-live."),
    feature(29, "Bloqueio da liberação, exceção e revisão",
        "Com o controle reprovado, nenhum resultado afetado sai sem decisão do responsável técnico.",
        "Quando o controle reprova, nenhum resultado daquele analito naquele aparelho é liberado, nem os que entraram "
        "antes. O responsável técnico ou o substituto libera um exame com justificativa, e os laudos liberados desde a "
        "última corrida aprovada entram numa revisão, mantidos ou retificados. O bloqueio fecha com a ação corretiva e "
        "uma nova corrida aprovada.",
        ["secretaria"]),
    feature(30, "Gráfico de Levey-Jennings",
        "A bancada vê a tendência do controle antes de ele reprovar.",
        "Para cada lote, nível, analito e aparelho, o gráfico mostra os valores do controle com as faixas de 1, 2 e 3 "
        "desvios, as regras que dispararam e a versão da referência usada em cada ponto.",
        ["secretaria"]),
    feature(31, "Registro do controle externo",
        "As rodadas do controle externo ficam registradas, com a ação para cada falha.",
        "O responsável técnico registra cada rodada do controle externo — programa, valor enviado, alvo e adequação —, "
        "com a ação para cada item inadequado, e anexa o relatório anual. Uma tela mostra a taxa de adequação por "
        "analito no ano.",
        ["secretaria"]),
    feature(32, "Estoque de reagentes e insumos com lotes",
        "O laboratório sabe o que tem, o que vence e o que está acabando.",
        "Reagentes, calibradores, controles e consumíveis têm unidade, estoque mínimo e rendimento. A entrada registra "
        "lote, validade, fornecedor e origem — lote vencido é recusado —, e aberturas, perdas e vencimentos pedem "
        "motivo; o saldo nunca fica negativo. Alertas mostram o que vence logo e o que está abaixo do mínimo.",
        ["secretaria"]),
    feature(33, "Lote em uso e rastreio do reagente",
        "Um lote com problema leva direto aos resultados que o usaram.",
        "O laboratório marca o lote de reagente em uso em cada aparelho e exame, e cada resultado e valor de controle "
        "guarda esse lote; o resultado sem lote em uso fica marcado sem rastreio. Por lote, sai a lista das requisições "
        "e corridas que o usaram, também em CSV.",
        ["secretaria"]),
    feature(34, "Relatório de consumo estimado",
        "A coordenação compara o que foi aberto com o que foi de fato usado.",
        "Por mês, item e aparelho, o relatório compara os testes realizados — resultados, repetições e controles — com "
        "os testes das embalagens abertas, e mostra o desvio. Sai na tela e em CSV; nada é baixado do estoque sozinho.",
        ["secretaria"]),
]

open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", len(m21["features"]))
PY
```

Expected: `ok 34`. Se o Step 1 mostrar algum F-21.25..F-21.34 que não chegou a Verified, ou a prova do api/dashboard tiver ficado só com simulador em outro ponto, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2. Se o F-21.34 entregue não tiver CSV, ou a regra de exceção mudar no api, corrija a frase.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 21 com 34 funcionalidades; o rodapé ("o 19, o 20 e o 21 estão em validação") e o ciclo com os textos de antes.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os dois arquivos da task (nenhum `package-lock.json`); no JSON, só linhas acrescentadas no módulo 21, a descrição do módulo e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts
$G commit -m "docs: add module 21c lab quality and supplies to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot: o **Módulo 21 — Exames e laboratório** com a descrição nova e as 34 funcionalidades; a F-21.28 "Em validação" com a nota do middleware real; a F-21.27 com a nota do responsável técnico; a página `/funcionalidades/f-21.29` com "O que faz", as telas e anterior/próxima dentro do módulo (F-21.28 e F-21.30); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod21c
/opt/homebrew/bin/git push -u origin content/module-21c-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-21c-catalog --title "docs: add module 21c lab quality and supplies to the catalog" --body "Adds F-21.25..F-21.34 (lab quality control, external QC and supplies, 21c) to module 21; QC over HL7 in validation until the first real middleware; RT validation note on QC runs; module description updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod21c && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-21c-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — Ambientes e independência dos dois interruptores.** O §8 dá `requires` e o código de `missing`, mas não diz em que ambientes as chaves existem nem que uma não exige a outra (está só na spec §3). Proposta: o contrato registra "sem `environments` (todos os ambientes, produção inclusive)" e "independentes: nenhum é `requires` do outro", para api e maintenance não divergirem. O maintenance testa as duas coisas (Task 2) e a Task 0 para se o api divergir.
2. **M2 — Aviso do middleware simulado fala das corridas de controle.** O §6 faz o `fake-middleware` mandar corridas de controle, uma delas reprovada (2-2s), o que **abre bloqueio de liberação** no laboratório da cidade; o §8 não pede mudança no aviso. Em staging isso bloqueia liberação de amostras de pacientes da cidade por um controle de exemplo, e quem liga `fake_middleware` precisa saber. Proposta (como a A25 do 21a e a M6 do 21b): o texto de `FAKE_MIDDLEWARE_NOTICE` ganha "e o controle de qualidade recebe corridas de controle de qualidade de exemplo, inclusive reprovadas"; mesma chave, mesmo nome do aviso, sem consulta nova (Task 1). Se recusada, sai o Step 3 de `lab.ts` e o teste correspondente das Tasks 1 e 2.
3. **M3 — Frase do pré-requisito já existente.** O §8 cita `municipal_lab_disabled`; a frase é a do 21b ("laboratório municipal (municipal_lab) desligado", `src/lib/features.ts` em `origin/main`). Proposta: o contrato registra que o 21c **reusa** a frase, sem código novo de `missing`. Código a mais que o api venha a mandar (por exemplo, exigir aparelho cadastrado) aparece cru até ganhar frase (Task 2, Review Focus 4).

## Self-review

- **Cobertura (spec §3, §7 último item, §11 itens 4 e 5, §12, §13; contrato §8 e §10):** os dois interruptores pelo mecanismo genérico com os `requires` do contrato — Tasks 1 e 2; frase de `municipal_lab_disabled` conferida (existe, do 21b) e fixada — Task 0 Step 2 e Task 1; independência e todos os ambientes — Task 2 (Divergência M1); `fake-middleware` mandando controles refletido no aviso — Tasks 1 e 2 (Divergência M2); regras de interface do maintenance — Global Constraints, guarda de tema na Task 2 e a suíte na Task 3, 375 px na prova; prova no navegador — Task 3; entrada do 21c no catálogo público como task final, com F-21.25..F-21.34, F-21.28 em validação até middleware real, risco do §13 (validação do CIQ pelo RT) como nota do F-21.27 — Task 4; cards — Tasks 0 e 3.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); status dos cards F-21.25..F-21.34 (Task 4 Step 1, com a regra de ajuste escrita); textos do catálogo marcados como rascunho a conferir contra o entregue, como no 21b.
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary`, `FAKE_MIDDLEWARE_NOTICE`, `fakeMiddlewareOn` existem em `origin/main` (`c3a534d`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem no topo de `FeaturesTab.test.tsx`; os nomes de botão (`ligar`, `desligar`, `confirmar: ligar <key>`) e de estado (`desligada`, `ligada e utilizável`, `ligada, falta pré-requisito`) e as frases de `doneText` são os do código atual; o JSON do catálogo reproduz byte a byte com `indent=2` (conferido).
- **Review Focus:** 1 — Task 2 ("desligar municipal_lab: depois do refetch…"); 2 — Task 2 ("independentes: estoque utilizável…"); 3 — Task 2 ("produção: os dois do 21c aparecem…"); 4 — Task 2 ("código de missing desconhecido aparece cru…"); 5 — Task 1 ("o aviso do middleware menciona as corridas de controle…") e Task 2 ("com fake_middleware ligado, o aviso fala do controle de qualidade").
