# Módulo 22 — Procedimentos e faturamento BPA (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 22** na branch `feat/mod-22-billing` (worktree `apps/api/.claude/mod22`, servidor de dev na porta **3044**) com `procedure_billing` em `Platform::Features::CATALOG`. Contrato §11: contracts → api → dashboard → **maintenance** → catálogo público. As Tasks 0–2 não dependem do api (testes com `fetch` falso; o maintenance não ganha consulta nem schema novo); a prova da Task 3 precisa do api do 22 rodando. A Task 4 (catálogo público) vem **por último**, depois da prova e das entregas de api, contracts, dashboard e maintenance.

**Goal:** Lado maintenance de F-22.1: o interruptor `procedure_billing` aparece na aba Funcionalidades com nome em português e liga pelo mecanismo genérico, sem pré-requisito de modo de prontuário. E as F-22.1..F-22.7 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única), com a F-22.6 "em validação" até a importação real no BPA Magnético.

**Architecture:** Nada muda no liga/desliga (`setCityFeature`, três estados, `missing` vindo do api). Entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts`; a frase de `ibge_code_missing` **já existe** (módulo 16) e só é fixada por teste para a chave nova (Divergência M1). Nenhuma tela, consulta, documento GraphQL, aviso de simulado ou componente novo. Catálogo público: o módulo `"22"` novo, depois do `"21"`, com sete funcionalidades, o rodapé e o ciclo 2 atualizados, por um script que reescreve o JSON no mesmo formato.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-21-module-22-procedures-billing-design.md` (§3 "Interruptor e papel", §6 arquivo, §10 itens 4 e 5, §11 a tabela inteira para o catálogo, §12 riscos e gates), `docs/adr/0039.md` e o contrato `docs/superpowers/plans/2026-10-21-module-22-billing-contracts.md` §1, §8 e §11 (as diferenças estão em "Divergências propostas ao contrato"). Base: plano do maintenance do 21c (`2026-10-18-module-21c-lab-qc-maintenance.md`, Tasks 1–4) e as lições dos contratos do 21a (§15) e do 21b (§14).

## Global Constraints

- Interruptor (contrato §1 e §8; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `procedure_billing` **sem** `requires` de modo: vale com o modo de prontuário `off`, `integrated` ou `record`; liga, e sem pré-requisito fica "ligada e utilizável";
  - existe em todos os ambientes (sem `environments`), produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele (Divergência M2);
  - se o api passar a exigir o código IBGE da cidade (Divergência M1: o BPA-I leva o IBGE de residência = o da cidade), o `missing` vem `["ibge_code_missing"]` e a frase é a do módulo 16 — "código IBGE não preenchido" —, **não** redefinida. Nenhuma linha nova no fluxo de `setCityFeature`.
- Nome: `procedure_billing` → "Faturamento de procedimentos (BPA)" (o mesmo do dashboard, `src/lib/features.ts` de lá).
- Nenhum CPF, CNS, nome de paciente, arquivo BPA ou CNPJ passa pelo maintenance; a configuração do faturamento (órgão de origem, destino, gestão) é do admin municipal no dashboard, não do mantenedor (spec §2, decisões do coordenador).
- Valor desconhecido (api mais novo: chave ou código de `missing`) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): só a `FeaturesTab` existente (`Panel`, `DataTable`, `Tag`, `ConfirmButton`), sem redesign nem componente novo; cores só por papel do tema; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte; tabelas largas em `<TableScroll>` (o `DataTable` já embrulha). Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); nada provado só com arquivo de teste aparece como "Disponível e verificado" sem nota — **F-22.6 fica `validating` até a importação real no BPA Magnético** (gate de go-live, spec §12); o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-22.1..F-22.7 do board #1 em In Progress (Task 0). O maintenance só é dono de parte do F-22.1: Done do F-22.1 só quando api, maintenance **e** dashboard dele estiverem entregues (o último app a terminar move o card); Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod22`, `C=$R/.worktrees/catalog-mod22`, `A=$R/apps/api/.claude/mod22`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-22-billing`; catálogo em `$C`, branch `content/module-22-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task. Worktrees antigos (`$R/.worktrees/maintenance-mod21c` etc.) não são tocados por este plano.
- maintenance — testes: `cd $M && npx vitest run <arquivos>`; tipos: `npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (o `FeaturesTab.test.tsx` já chama `cleanup` no topo), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- O api do 22 roda em dev na **3044** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **Cidade com o modo de prontuário desligado (`off`) ou integrado.** O faturamento não depende do prontuário (spec §3): ligar `procedure_billing` dá "ligada e utilizável" nos três modos — nunca "falta pré-requisito" por causa do modo. Teste: Task 2, "liga utilizável com o modo off e com o integrated: o modo não é pré-requisito".
2. **Cidade sem código IBGE** (o BPA-I leva o IBGE de residência = o da cidade). Se o api mandar `ibge_code_missing`, a linha diz "ligada, falta pré-requisito" com "código IBGE não preenchido" — a frase do módulo 16, sem redefinir —, e o IBGE só leitura continua "não preenchido" na mesma aba. Teste: Task 2, "sem IBGE: ligada, falta o código IBGE, com a frase do módulo 16".
3. **Produção.** `procedure_billing` aparece em produção e não é simulado: nenhum aviso (`role="note"`) nem "SIMULADO" por causa dele. Teste: Task 2, "produção: o interruptor aparece e nenhum aviso de simulado".
4. **Api mais novo com um pré-requisito a mais** (por exemplo, exigir a configuração do faturamento, código que esta tela não conhece). O código aparece cru ao lado da frase conhecida, nunca `undefined` nem a linha some. Teste: Task 2, "código de missing desconhecido aparece cru ao lado do conhecido".
5. **Desligar com o faturamento em uso.** Desligar é o mesmo fluxo de dois cliques, manda `enabled: false` e diz "procedure_billing desligada."; o maintenance não promete nada sobre competências abertas (o dashboard some com o grupo; os dados ficam). Teste: Task 2, "desliga com dois cliques e diz o estado devolvido".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-22.1..F-22.7, conferência do main e do api do 22, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts` | nome da chave; frase de `ibge_code_missing` fixada para ela | 1 |
| `src/screens/FeaturesTab.test.tsx`, `README.md` | o interruptor pelo fluxo genérico da aba (sem mudar `FeaturesTab.tsx`) | 2 |
| — | suíte, codegen:check, build, revisão, prova no navegador | 3 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx` | módulo 22 com F-22.1..F-22.7, rodapé e ciclo 2 | 4 |

**Estratégia de teste:** funções puras (Task 1: `featureLabel`, `missingText`, `missingSummary`); o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation; os três modos de prontuário; o IBGE ausente; produção; código desconhecido; desligar (Task 2); suíte inteira com as guardas de tema, `codegen:check` e build, e prova no navegador contra o api do 22 (Task 3). Catálogo: `validateCatalog`, o teste do módulo 22, o rodapé e o ciclo (Task 4).

---

### Task 0: Cards, conferência do main e do api do 22, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`5148825` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-22.1..F-22.7 em In Progress (ou além); worktree `$M` na branch `feat/mod-22-billing`; a base anotada.

- [ ] **Step 1: Mova os cards F-22.1..F-22.7 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | test("^F-22\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified (os planos do api e do dashboard movem os mesmos cards) ficam como estão. Para cada um que estiver antes de In Progress, com o `id` do item e o id da opção "In Progress":

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os sete itens com `status` In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main do maintenance e o api do 22**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod22
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n \
  '  ibge_code_missing: "código IBGE não preenchido",\|  lab_inventory: "Estoque do laboratório"\|procedure_billing'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n 'featureLabel(f.key)\|missingSummary(f.missing)\|confirmar: ligar ${f.key}'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Qualidade e estoque do laboratório (módulo 21c).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n -E 'key: "procedure_billing"' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: as duas âncoras (a frase de `ibge_code_missing` já existe, do módulo 16) e **nenhuma** linha com `procedure_billing`;
- em `FeaturesTab.tsx`: as três âncoras do fluxo genérico; no `README.md`: os dois parágrafos. Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 22: a entrada com `requires: []` (ou sem `requires`), ou com `requires: %w[ibge_code]` se a Divergência M1 tiver sido aceita, e **sem** `environments:`. Se o api ainda não tiver a entrada, as Tasks 0–2 seguem; **a Task 3 espera** o api. Se o api tiver um `requires` de modo (`record_mode`, `feature:clinical_record`) ou um `environments`, **pare e reporte** (divergência com a spec §3 e o contrato §8).

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod22 -b feat/mod-22-billing origin/main
cd $R/.worktrees/maintenance-mod22 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera **o mesmo número de arquivos de teste** (nenhum arquivo novo) e **+9 testes** (3 em `features.test.ts`, 6 em `FeaturesTab.test.tsx`).

---

### Task 1: Nome do interruptor e a frase do IBGE fixada para ele

**Files:**
- Modify: `src/lib/features.ts`
- Test: `src/lib/features.test.ts`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes em `features.ts`).
- Produces: `featureLabel("procedure_billing")` → "Faturamento de procedimentos (BPA)"; `missingText("ibge_code_missing")` → "código IBGE não preenchido" (sem mudança).

- [ ] **Step 1: Write the failing test**

Em `src/lib/features.test.ts`, no fim do arquivo:

```ts
describe("módulo 22", () => {
  it("procedure_billing tem nome em português, o mesmo do dashboard", () => {
    expect(featureLabel("procedure_billing")).toBe("Faturamento de procedimentos (BPA)");
  });

  it("se o api exigir o IBGE da cidade (Divergência M1), a frase é a do módulo 16, sem redefinir", () => {
    expect(missingText("ibge_code_missing")).toBe("código IBGE não preenchido");
    expect(missingSummary([ "ibge_code_missing" ])).toBe("código IBGE não preenchido");
  });

  it("não há frase de pré-requisito de modo para o faturamento: código assim apareceria cru", () => {
    // spec §3: sem pré-requisito de modo de prontuário.
    expect(missingText("procedure_billing_disabled")).toBe("procedure_billing_disabled");
    expect(missingText("billing_settings_missing")).toBe("billing_settings_missing");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/features.test.ts`
Expected: FAIL — `featureLabel("procedure_billing")` é `null`. (Os outros dois testes já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, troque

```ts
  lab_inventory: "Estoque do laboratório"
};
```

por

```ts
  lab_inventory: "Estoque do laboratório",
  // Módulo 22 (ADR 0039): sem pré-requisito de modo de prontuário; existe em
  // todos os ambientes e não é simulado. Se o api exigir o IBGE da cidade, a
  // frase é a de ibge_code_missing, acima.
  procedure_billing: "Faturamento de procedimentos (BPA)"
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/screens/FeaturesTab.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts
$G commit -m "feat: name the procedure billing switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: O interruptor pelo fluxo genérico da aba

**Files:**
- Modify: `README.md`
- Test: `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo, e relê a lista depois de cada `setCityFeature`. Esta task fixa esse comportamento para a chave nova.

**Interfaces:**
- Consumes: o rótulo (Task 1); `FeaturesTab`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: nada novo para outras tasks.

- [ ] **Step 1: Write the failing test**

Em `src/screens/FeaturesTab.test.tsx`, no fim do arquivo:

```tsx
describe("FeaturesTab — faturamento de procedimentos (módulo 22)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const PB_OFF: Feature = {
    key: "procedure_billing", description: "Faturamento de procedimentos: críticas SIGTAP e arquivo do BPA Magnético", enabled: false,
    usable: false, missing: [], changedAt: null, changedBy: null
  };
  const PB_ON: Feature = { ...PB_OFF, enabled: true, usable: true, changedAt: "2026-10-21T13:00:00Z", changedBy: "dev@local" };

  async function turnOn(recordMode: string) {
    const user = userEvent.setup();
    let listed = [ PB_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ PB_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: PB_ON } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode }));
    });
    renderTab();
    expect(await screen.findByText("procedure_billing")).not.toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar procedure_billing" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "procedure_billing", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe("procedure_billing ligada e utilizável.");
  }

  it("aparece com o nome em português e sem pré-requisito", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ PB_OFF ])));
    renderTab();

    expect(await screen.findByText("procedure_billing")).not.toBeNull();
    expect(screen.getByText("Faturamento de procedimentos (BPA)")).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(screen.getByText("nada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/_disabled|undefined|null/);
  });

  it("liga utilizável com o modo off e com o integrated: o modo não é pré-requisito", async () => {
    await turnOn("off");
    expect(screen.getByText("desligado (off)")).not.toBeNull();
    cleanup();
    fetchMock.mockReset();
    await turnOn("integrated");
    expect(screen.getByText("integrado ao PEC da cidade (integrated)")).not.toBeNull();
    expect(screen.queryByText("ligada, falta pré-requisito")).toBeNull();
  });

  it("sem IBGE: ligada, falta o código IBGE, com a frase do módulo 16", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply(
      [ { ...PB_ON, usable: false, missing: [ "ibge_code_missing" ] } ], { profile: { ibgeCode: null } }
    )));
    renderTab();

    expect(await screen.findByText("procedure_billing")).not.toBeNull();
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.getByText("código IBGE não preenchido")).not.toBeNull();
    expect(screen.getByText("não preenchido")).not.toBeNull();
  });

  it("desliga com dois cliques e diz o estado devolvido", async () => {
    const user = userEvent.setup();
    let listed = [ PB_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ { ...PB_OFF, changedAt: "2026-10-21T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("ligada e utilizável")).not.toBeNull();
    await user.click(screen.getByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar procedure_billing" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "procedure_billing", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("procedure_billing desligada.");
    expect(await screen.findByText("desligada")).not.toBeNull();
  });

  it("produção: o interruptor aparece e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ PB_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("procedure_billing")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });

  it("código de missing desconhecido aparece cru ao lado do conhecido", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      { ...PB_ON, usable: false, missing: [ "ibge_code_missing", "billing_settings_missing" ] }
    ])));
    renderTab();

    expect(await screen.findByText("código IBGE não preenchido; billing_settings_missing")).not.toBeNull();
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it passes já com a Task 1 — e falha sem ela**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx`
Expected: PASS — a aba é genérica; o que estes testes dependem de novo (o rótulo) veio da Task 1. Para ver que eles pegam a regressão, rode uma vez com `features.ts` de antes da Task 1 e devolva em seguida:

```bash
cd $M && $G show origin/main:src/lib/features.ts > src/lib/features.ts && npx vitest run src/screens/FeaturesTab.test.tsx 2>&1 | grep -E 'Tests|FAIL' ; $G checkout -- src/lib/features.ts && $G status --short src/lib
```

Expected: FAIL em "aparece com o nome em português e sem pré-requisito" (não acha "Faturamento de procedimentos (BPA)"); depois do `checkout`, o `status` de `src/lib` vazio (a Task 1 commitada de volta) e a suíte da aba volta a passar.

Se algum teste falhar **com** a Task 1 aplicada, o comportamento genérico mudou no main: leia `FeaturesTab.tsx` e ajuste o **teste** à intenção (o maintenance não ganha código novo para isso). Atenção às frases que este plano supõe do código atual: "desligado (off)", "integrado ao PEC da cidade (integrated)" e "não preenchido" (`recordModeText` e `ibgeCodeText`) e "nada" (`missingSummary([])`).

- [ ] **Step 3: README**

Em `README.md`, logo antes do parágrafo **Documentos clínicos (módulo 19c).** (depois do parágrafo **Qualidade e estoque do laboratório (módulo 21c).**), acrescente:

```markdown
**Faturamento de procedimentos (módulo 22).** O interruptor
`procedure_billing` ("Faturamento de procedimentos (BPA)") liga o registro do
procedimento realizado com as críticas da SIGTAP, o fechamento da competência
e o arquivo do BPA Magnético. Não depende do modo de prontuário (vale com
`off`, `integrated` ou `record`), existe em todos os ambientes e não tem
simulado. Se o api exigir o código IBGE da cidade (o arquivo leva o município
de residência), a linha diz "código IBGE não preenchido" até o perfil ter o
código. A configuração do arquivo (órgão de origem e de destino, gestão) e o
papel de faturista são da cidade, no dashboard; nenhum dado de paciente nem o
arquivo passam por aqui. Ordem de deploy: api → dashboard → maintenance;
contra um api sem o 22 a chave só não aparece.
```

- [ ] **Step 4: Run tests**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/FeaturesTab.test.tsx README.md
$G commit -m "test: pin the procedure billing switch in the features tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-22-billing`; o api do 22 na **3044**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 22).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd $M && npm run typecheck && npx vitest run && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base anotada na Task 0, **mesmo número de arquivos de teste** e **+9 testes**. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e contagem `0`.

O maintenance não usa SDL novo no 22, mas o `schema.graphql` do worktree precisa seguir igual ao do api (o api pode ter mexido no `Maintenance::Schema`):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml exec -T -w /app/.claude/mod22 api bin/rails runner 'puts Maintenance::Schema.to_definition' | diff - $M/schema.graphql && echo schema-igual
```

Expected: `schema-igual`. Se diferir, **pare e reporte** (o contrato §8 não prevê mudança no schema do maintenance; não puxe nem edite à mão sem decisão).

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §10 item 4, §11 F-22.1), o ADR 0039 e o contrato §8. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; a frase de `ibge_code_missing` não foi redefinida;
- o rótulo exatamente como no Global Constraints e igual ao do dashboard;
- nenhum aviso de simulado por causa do `procedure_billing`;
- `schema.graphql` igual ao `to_definition`; `src/gql/*` sem mudança;
- nenhum CPF, CNS, CNPJ, nome de paciente ou conteúdo de arquivo na tela; cores só por papel do tema.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 22 na **3044** (plano do api). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, com `node_modules` num volume próprio, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod22 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod22:/app -v maintenance-mod22-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3044 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod22   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 22 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. **Anote** antes de mexer o estado de `procedure_billing` de Curitiba e o modo de prontuário. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: `procedure_billing` com "Faturamento de procedimentos (BPA)", "desligada" e "nada" no que falta;
- ligar (dois cliques) → "procedure_billing ligada e utilizável." (ou, se a M1 foi aceita e Curitiba não tem IBGE, "… falta: código IBGE não preenchido.");
- numa cidade de modo `integrated` ou `off` (se houver na semente), ligar dá "ligada e utilizável" — o modo não é pré-requisito;
- desligar e religar pelo mesmo fluxo; nenhum aviso de simulado aparece por causa dele;
- **Auditoria** mostra as tentativas e os resultados de `setCityFeature`;
- a 375 px de largura: sem rolagem lateral da página; a tabela rola dentro do contêiner com a dica "⇆ role para o lado";
- no dashboard (prova do plano do dashboard), com a chave ligada, o grupo Faturamento aparece para o faturista; desligada, some.

Depois: devolva `procedure_billing` ao estado anotado (pela própria tela); `docker rm -f maintenance-mod22 && docker volume rm maintenance-mod22-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 22, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-22-billing
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod22
/opt/homebrew/bin/git branch -d feat/mod-22-billing
```

Se os lados api e dashboard do F-22.1 já estiverem **Done**, mova o card F-22.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 3 e a do dashboard passaram.

---

### Task 4: O módulo 22 no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-22.1..F-22.7 no board #1 (status real de cada um); o resultado da prova do 22 (planos do api, do dashboard e deste); o que foi entregue em api, contracts, dashboard e maintenance.
- Produces: módulo `"22"` no JSON (ciclo 2, depois do `"21"` e antes do `"iu"`), com F-22.1..F-22.7, `status: "validating"`; `meta.cycles[1].modules` = "15 a 22 e interface"; o rodapé "o 19, o 20, o 21 e o 22 estão em validação".

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | test("^F-22\\.[1-7] ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod22 -b content/module-22-catalog origin/main
cd $R/.worktrees/catalog-mod22 && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-22.1..F-22.5 e F-22.7 em Done/Verified e F-22.6 em Done (Verified só depois da importação real); os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário. A suíte do catálogo verde. **Anote**: o JSON termina o ciclo 2 em `"21"` e `"iu"` (`b5c6d1e` ou mais novo); o rodapé diz "o 19, o 20 e o 21 estão em validação"; o ciclo 2 diz "15 a 21 e interface".

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("formatValidating")`, troque

```ts
    expect(formatValidating(["19", "20", "21"])).toBe("o 19, o 20 e o 21 estão");
```

por

```ts
    expect(formatValidating(["19", "20", "21"])).toBe("o 19, o 20 e o 21 estão");
    expect(formatValidating(["19", "20", "21", "22"])).toBe("o 19, o 20, o 21 e o 22 estão");
```

e, no fim do arquivo:

```ts
describe("módulo 22", () => {
  it("entra no ciclo 2 depois do 21: só o arquivo BPA em validação, até a importação real no BPA Magnético", () => {
    const ids = catalog.modules.map((m) => m.id);
    expect(ids.indexOf("22")).toBe(ids.indexOf("21") + 1);
    const m22 = catalog.modules.find((m) => m.id === "22")!;
    expect(m22.cycle).toBe(2);
    expect(m22.status).toBe("validating");
    expect(m22.name).toBe("Procedimentos e faturamento");
    expect(m22.description).toMatch(/BPA Magnético/);
    expect(m22.description).toMatch(/UBS/);
    expect(m22.features.map((f) => f.id)).toEqual(Array.from({ length: 7 }, (_, i) => `F-22.${i + 1}`));
    for (const id of ["f-22.1", "f-22.2", "f-22.3", "f-22.4", "f-22.5", "f-22.7"]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const bpa = findFeature("f-22.6")!.feature;
    expect(bpa.status).toBe("validating");
    expect(bpa.note).toMatch(/BPA Magnético/);
    expect(bpa.note).toMatch(/09\/07\/2026/);
    expect(findFeature("f-22.1")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    expect(findFeature("f-22.2")?.feature.description).toMatch(/nunca impede/);
    expect(findFeature("f-22.3")?.feature.surfaces).toEqual([ "bastidores" ]);
    expect(findFeature("f-22.7")?.feature.surfaces).toEqual([ "secretaria", "cidadao" ]);
    expect(catalog.meta.cycles.find((c) => c.id === 2)?.modules).toBe("15 a 22 e interface");
  });
});
```

Em `src/App.test.tsx`, troque

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20 e o 21 estão em validação."
```

por

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21 e o 22 estão em validação."
```

e

```ts
    expect(within(c2).getByText("módulos 15 a 21 e interface")).toBeInTheDocument();
```

por

```ts
    expect(within(c2).getByText("módulos 15 a 22 e interface")).toBeInTheDocument();
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts src/App.test.tsx`
Expected: FAIL — não há módulo 22, o ciclo 2 diz "15 a 21" e o rodapé não fala do 22. (O teste de `formatValidating` com quatro já passa: a função é genérica.)

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo publicado byte a byte (conferido em `b5c6d1e`), então o diff fica só no que mudou. **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard e regras do api) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

ids = [m["id"] for m in data["modules"]]
assert "22" not in ids, "módulo 22 já existe"
assert data["meta"]["cycles"][1]["modules"] == "15 a 21 e interface", "ciclo 2 mudou: pare e reporte"

data["meta"]["updatedAt"] = today
data["meta"]["cycles"][1]["modules"] = "15 a 22 e interface"

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-22.{n}", "slug": f"f-22.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m22 = {
    "id": "22",
    "cycle": 2,
    "name": "Procedimentos e faturamento",
    "status": "validating",
    "description": (
        "O procedimento feito na UPA, no hospital ou no laboratório municipal é registrado e conferido com a tabela SIGTAP "
        "sem travar o atendimento. No fim do mês, o faturista fecha a competência de cada estabelecimento e gera o arquivo "
        "que o BPA Magnético importa, sem redigitar. A UBS fica de fora: a produção da atenção primária segue pelo e-SUS APS."
    ),
    "features": [
        feature(1, "Interruptor e papel de faturista",
            "O faturamento entra por cidade, com uma pessoa responsável por ele.",
            "A equipe da plataforma liga o faturamento de procedimentos por cidade, sem depender do modo de prontuário. "
            "A administração convida o faturista, que fecha as competências e baixa o arquivo; a própria administração "
            "acompanha e configura o cabeçalho do arquivo.",
            ["plataforma", "secretaria"]),
        feature(2, "Registro de procedimento com conferência SIGTAP",
            "O procedimento é registrado na hora, e o que não vai passar no faturamento aparece logo.",
            "No atendimento, na consulta ou avulso pela unidade, o profissional busca o procedimento na tabela SIGTAP da "
            "competência e informa quantidade e CID quando a tabela pede. A conferência de CBO, idade, sexo, CID e instrumento "
            "marca o procedimento como não faturável, com o motivo, e nunca impede o registro.",
            ["secretaria"]),
        feature(3, "Produção do laboratório municipal no faturamento",
            "Os exames do laboratório municipal entram no faturamento sem digitação.",
            "Cada exame liberado no laudo do laboratório municipal vira um procedimento realizado, com o liberador como "
            "profissional e o laboratório como estabelecimento. A retificação do laudo não duplica a produção.",
            ["bastidores"]),
        feature(4, "Lançamento manual pelo faturista",
            "A produção feita fora do sistema também chega ao arquivo.",
            "O faturista lança o procedimento feito fora do Rota Saúde, escolhendo o estabelecimento, o paciente e o "
            "profissional com o CBO do vínculo. O lançamento passa pela mesma conferência com a SIGTAP.",
            ["secretaria"]),
        feature(5, "Fechamento da competência e pendências",
            "O faturista vê antes do fechamento o que não vai passar e o que corrigir.",
            "Por estabelecimento e competência, a tela mostra as linhas de BPA-I e BPA-C, os não faturáveis por motivo e as "
            "pendências agrupadas pelo que corrigir — cadastro do profissional, do paciente ou do estabelecimento. Corrigido "
            "o cadastro, a pendência sai sem relançar. Fechar e reabrir pedem a verificação em duas etapas, e reabrir pede motivo.",
            ["secretaria"]),
        feature(6, "Arquivo BPA-I e BPA-C do BPA Magnético",
            "O arquivo da competência sai pronto para importar no BPA Magnético.",
            "O fechamento gera o arquivo de interface do BPA Magnético, com o BPA individualizado item a item e o consolidado "
            "agregado. Cada fechamento é uma versão guardada; o arquivo nunca muda depois de gerado, e cada download fica "
            "registrado.",
            ["secretaria"], status="validating",
            note="Gerado e conferido contra o layout do DATASUS de 09/07/2026. Falta a importação real no BPA Magnético, pela secretaria piloto, antes do go-live."),
        feature(7, "Procedimentos na linha do tempo",
            "O procedimento realizado aparece no histórico do paciente.",
            "O profissional vê na linha do tempo o procedimento com a data, a unidade e quem fez; o cidadão vê o nome e a "
            "data, sem nada de faturamento.",
            ["secretaria", "cidadao"]),
    ],
}

data["modules"].insert(ids.index("21") + 1, m22)
open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", [m["id"] for m in data["modules"]][-3:], len(m22["features"]))
PY
```

Expected: `ok ['21', '22', 'iu'] 7`. Se o Step 1 mostrar algum F-22.1..F-22.5 ou F-22.7 que não chegou a Verified, ou a prova do api/dashboard tiver ficado só com dado de exemplo em outro ponto, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2. Se a F-22.7 entregue não aparecer para o cidadão no wpda, tire `"cidadao"` das telas e do teste.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 22 com 7 funcionalidades; o rodapé e o ciclo com os textos novos.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os três arquivos da task (nenhum `package-lock.json`); no JSON, só o módulo 22 acrescentado, o ciclo 2 e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 22 procedures and billing to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot: o **Módulo 22 — Procedimentos e faturamento** no ciclo 2, logo depois do 21, com a descrição e as 7 funcionalidades; a F-22.6 "Em validação" com a nota da importação real; o rodapé "o 19, o 20, o 21 e o 22 estão em validação"; a página `/funcionalidades/f-22.5` com "O que faz", as telas e anterior/próxima dentro do módulo (F-22.4 e F-22.6); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod22
/opt/homebrew/bin/git push -u origin content/module-22-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-22-catalog --title "docs: add module 22 procedures and billing to the catalog" --body "Adds module 22 (procedures and BPA billing) after module 21 with F-22.1..F-22.7; the BPA file (F-22.6) stays in validation until the real import in BPA Magnético; cycle 2 and the footer updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod22 && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-22-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — IBGE da cidade como pré-requisito.** O §8 diz "interruptor `procedure_billing` (sem `requires`)", mas o BPA-I leva o IBGE de residência, que a spec §2 (decisão 8) e §6 preenchem com o município da cidade; sem o código no perfil, todo BPA-I sai com pendência. O mecanismo genérico já tem `ibge_code` → `ibge_code_missing` (usado pelo `ledi_export`). Proposta: `procedure_billing` com `requires: %w[ibge_code]` — continua **sem** pré-requisito de modo (spec §3). O maintenance já tem a frase e a testa (Task 1 e Task 2); se a proposta for recusada, os testes seguem válidos (o código só não chega) e o README perde a frase do IBGE.
2. **M2 — Ambientes e simulado.** O §8 não diz em que ambientes a chave existe. Proposta: o contrato registra "sem `environments` (todos os ambientes, produção inclusive)" e "sem simulado e sem aviso" — o arquivo de teste golden é do api, não um serviço simulado ligável. O maintenance testa produção (Task 2) e a Task 0 para se o api divergir.
3. **M3 — Mesmo nome nos dois painéis.** O contrato não fixa o rótulo. Proposta: "Faturamento de procedimentos (BPA)" no maintenance e no dashboard (`features.ts` dos dois), para o mantenedor e a administração falarem da mesma coisa.

## Self-review

- **Cobertura (spec §3, §10 itens 4 e 5, §11, §12; contrato §1, §8, §11):** o interruptor pelo mecanismo genérico, sem pré-requisito de modo — Tasks 1 e 2 (três modos testados; Review Focus 1); IBGE como pré-requisito proposto, com a frase existente — Tasks 1 e 2 (Divergência M1; Review Focus 2); todos os ambientes, sem simulado — Task 2 (Divergência M2; Review Focus 3); regras de interface do maintenance — Global Constraints, guardas de tema nas Tasks 2 e 3, 375 px na prova; prova no navegador com o api na 3044 — Task 3; o módulo 22 no catálogo público como task final, com F-22.1..F-22.7, F-22.6 em validação até a importação real no BPA Magnético (gate da spec §12), rodapé e ciclo 2 — Task 4; cards — Tasks 0 e 3.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); status dos cards F-22.1..F-22.7 (Task 4 Step 1, com a regra de ajuste escrita); textos do catálogo marcados como rascunho a conferir contra o entregue, como no 21b e no 21c.
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary` existem em `origin/main` (`5148825`); `reply`, `bodyOf`, `calls`, `featuresReply` (com `extra` para `recordMode` e `profile`), `renderTab`, `Feature` existem no topo de `FeaturesTab.test.tsx`; os nomes de botão (`ligar`, `desligar`, `confirmar: ligar <key>`, `confirmar: desligar <key>`) e de estado (`desligada`, `ligada e utilizável`, `ligada, falta pré-requisito`) e as frases de `doneText`, `recordModeText` e `ibgeCodeText` são as do código atual; o JSON do catálogo reproduz byte a byte com `indent=2` (conferido em `b5c6d1e`); `formatValidating` é genérica (conferido em `src/pages/ModuleList.tsx`).
- **Review Focus:** 1 — Task 2 ("liga utilizável com o modo off e com o integrated: o modo não é pré-requisito"); 2 — Task 2 ("sem IBGE: ligada, falta o código IBGE, com a frase do módulo 16"); 3 — Task 2 ("produção: o interruptor aparece e nenhum aviso de simulado"); 4 — Task 2 ("código de missing desconhecido aparece cru ao lado do conhecido"); 5 — Task 2 ("desliga com dois cliques e diz o estado devolvido").
