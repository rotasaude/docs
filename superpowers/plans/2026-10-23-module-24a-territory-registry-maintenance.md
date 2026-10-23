# Módulo 24a — Cadastro individual e domiciliar (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 24a** na branch `feat/mod-24a-territory-registry` (worktree `apps/api/.claude/mod24a`, servidor de dev na porta **3046**) com `territory_registry` em `Platform::Features::CATALOG` (`requires: ["feature:clinical_record"]`, sem `environments:`). Contrato §13: api → dashboard → **maintenance** → catálogo público. As Tasks 0–1 não dependem do api rodando (testes com `fetch` falso, pelo fluxo genérico da aba Funcionalidades); a prova da Task 2 precisa do api do 24a. A Task 3 (catálogo público) vem **por último**, depois da prova e das entregas de api, dashboard e maintenance.

**Goal:** Lado maintenance de F-24.1: o interruptor `territory_registry` aparece na aba Funcionalidades com nome em português ("Cadastro individual e domiciliar", o mesmo do dashboard) e liga pelo mecanismo genérico, exigindo `clinical_record` (a frase "prontuário da atenção primária (clinical_record) desligado" do módulo 19 quando falta); e as F-24.1..F-24.8 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única).

**Architecture:** O liga/desliga não muda (`setCityFeature`, três estados, `missing` vindo do api): entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts` e um parágrafo no `README.md`; `src/screens/FeaturesTab.tsx` não muda — os testes fixam o fluxo genérico para a chave nova. O maintenance não tem tela própria do 24a: microáreas, domicílios e cadastros são da cidade (dashboard) e nenhum dado de cidadão passa por aqui. O catálogo ganha o módulo `"24"` por script que reescreve o JSON byte a byte, com os testes de `catalog.test.ts` e `App.test.tsx`.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-23-module-24a-territory-registry-design.md` (§1 intenção, §2 decisões 8, 10 e 12, §3 interruptor, §5 telas, §8 entrega, §9 F-IDs, §10 riscos), `docs/adr/0042.md` e o contrato `docs/superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md` §1, §11 (Maintenance) e §13. Base: plano do maintenance do 23 (`2026-10-22-module-23-care-lines-maintenance.md`, Tasks 1, 5 e 6). As diferenças estão em "Divergências propostas ao contrato", N1–N3.

## Global Constraints

- Interruptor (spec §3; contrato §1 e §11), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `territory_registry` com `requires: ["feature:clinical_record"]`: sem `clinical_record` ligado e utilizável, o `missing` vem `["clinical_record_disabled"]` e a frase é a do módulo 19 — "prontuário da atenção primária (clinical_record) desligado" —, **não** redefinida; o modo `record` chega por tabela (o `clinical_record` já exige `record_mode_not_record`);
  - existe em todos os ambientes, produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele.
- Nome: `territory_registry` → "Cadastro individual e domiciliar" (o mesmo de `src/lib/features.ts` do dashboard, Task 1 do plano do dashboard).
- Desligar não apaga nada (spec §3: "o que já foi gravado fica"); a aba não diz o contrário.
- O maintenance não vê nada de cidade: nenhum paciente, domicílio, família, cadastro, microárea ou contagem de validade passa por aqui.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); o que só foi provado com dado de exemplo fica `validating` com nota; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): nada novo de tela; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-24.1..F-24.8 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-24.1: Done do F-24.1 quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod24a`, `C=$R/.worktrees/catalog-mod24a`, `A=$R/apps/api/.claude/mod24a`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-24a-territory-registry`; catálogo em `$C`, branch `content/module-24a-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task.
- maintenance — testes: `bin/test maintenance -w maintenance-mod24a <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (os testes chamam `cleanup`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- catálogo — testes: `cd $C && npx vitest run` (o catálogo não está no `bin/test`; é estático, sem banco).
- O api do 24a roda em dev na **3046** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177` (o proxy de dev injeta o Origin constante `http://maintenance.localhost:5177`, que o api exige; por isso a prova usa a 5177 e não a 5201 do brief — N3); conta `dev@local` semeada pelo `db:seed`; o usuário faz o login. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade sem prontuário (modo `off`/`integrated` ou `clinical_record` desligado).** Ligar `territory_registry` dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado" — a frase do módulo 19, sem redefinir; com `clinical_record` ligado e utilizável, "ligada e utilizável". Teste: Task 1, "sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19".
2. **Desligar com cadastros gravados.** Espera-se: dois cliques, a mutation com `enabled: false`, "territory_registry desligada." — e nada na aba sugere que os cadastros somem (a spec guarda o que foi gravado). Teste: Task 1, "desliga com dois cliques; nada fala em apagar".
3. **Api mais novo com um `missing` que o maintenance não conhece** (ex.: um pré-requisito que o plano do api acrescente, como `team_kind_missing`). Espera-se: o código aparece cru na linha, sem `undefined`, e o resto da aba segue. Teste: Task 1, "missing desconhecido aparece cru".
4. **Produção.** Espera-se: a chave aparece e nenhum aviso de simulado. Teste: Task 1, "produção: aparece e nenhum aviso de simulado".
5. **Catálogo prometendo o que não foi provado.** Espera-se: F-24.7 (fichas LEDI de cadastro) em validação até o SIAPS aceitar fichas reais da cidade piloto; nenhuma frase sobre visita domiciliar, atividade coletiva ou app de campo (24b/24c); o módulo inteiro em validação. Teste: Task 3, "módulo 24: fichas em validação; nada de visita nem app de campo".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-24.1..F-24.8, conferência do main e do api do 24a, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome do interruptor e o fluxo genérico da aba fixado para ele | 1 |
| — | suíte, codegen:check, build, revisão, prova no navegador | 2 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx` | módulo 24 com F-24.1..F-24.8, rodapé e ciclo 2 | 3 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`); o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation (Task 1); suíte inteira com as guardas de tema, `codegen:check`, build e a prova no navegador contra o api do 24a (Task 2). Catálogo: `validateCatalog`, o teste do módulo 24, o rodapé e o ciclo (Task 3).

---

### Task 0: Cards, conferência do main e do api do 24a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`2df03d3` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-24.1..F-24.8 em In Progress (ou além); worktree `$M` na branch `feat/mod-24a-territory-registry`; a base anotada.

- [ ] **Step 1: Mova os cards F-24.1..F-24.8 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified ficam como estão. Para cada um antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito itens em In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main do maintenance e o api do 24a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod24a
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n \
  '  clinical_record_disabled: "prontuário da atenção primária (clinical_record) desligado",\|  care_lines: "Linhas de cuidado"\|territory_registry'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.test.tsx | grep -n '^function reply\|^function bodyOf\|^function calls\|^type Feature = {\|^function featuresReply\|^function renderTab'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Linhas de cuidado (módulo 23).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n -E 'key: "territory_registry"' -A4 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: as duas âncoras (a frase de `clinical_record_disabled` já existe, do módulo 19; `care_lines` é o último rótulo) e **nenhuma** linha com `territory_registry`;
- no `FeaturesTab.test.tsx`: os ajudantes do topo que a Task 1 reusa; no `README.md`: as âncoras. Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 24a: a entrada com `requires: %w[feature:clinical_record]` (ou equivalente) e **sem** `environments:`. Se o api tiver outro pré-requisito ou `environments` para `territory_registry`, **pare e reporte** (divergência com a spec §3 e o contrato §1). Se o api ainda não tiver a entrada, a Task 1 segue (os testes são por `fetch` falso) e a Task 2 confere.

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod24a -b feat/mod-24a-territory-registry origin/main
cd $R/.worktrees/maintenance-mod24a && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod24a
```

Expected: `base-ok` e a suíte verde. **Anote** a base (arquivos e testes): a Task 2 espera **nenhum arquivo de teste novo** e casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: O interruptor `territory_registry` — nome e o fluxo genérico da aba

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo, e relê a lista depois de cada `setCityFeature`. Esta task dá o nome e fixa esse comportamento para a chave nova.

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes em `features.ts`); `FeaturesTab`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("territory_registry")` → "Cadastro individual e domiciliar"; `missingText("clinical_record_disabled")` sem mudança.

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 24a", () => {
  it("territory_registry tem nome em português, o mesmo do dashboard", () => {
    expect(featureLabel("territory_registry")).toBe("Cadastro individual e domiciliar");
  });

  it("o pré-requisito é o prontuário: a frase do módulo 19, sem redefinir; código desconhecido cru", () => {
    expect(missingText("clinical_record_disabled")).toBe("prontuário da atenção primária (clinical_record) desligado");
    expect(missingSummary([ "clinical_record_disabled" ])).toBe("prontuário da atenção primária (clinical_record) desligado");
    expect(missingText("team_kind_missing")).toBe("team_kind_missing");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx`:

```tsx
describe("FeaturesTab — cadastro individual e domiciliar (módulo 24a)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const TR_OFF: Feature = {
    key: "territory_registry", description: "Cadastro individual e domiciliar: microáreas, domicílios, famílias e fichas de cadastro",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const TR_ON: Feature = { ...TR_OFF, enabled: true, usable: true, changedAt: "2026-10-23T13:00:00Z", changedBy: "dev@local" };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ TR_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("territory_registry")).not.toBeNull();
    expect(screen.getByText("Cadastro individual e domiciliar")).not.toBeNull();
    expect(screen.getByText(TR_OFF.description)).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19", async () => {
    const user = userEvent.setup();
    const TR_BLOCKED: Feature = { ...TR_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ TR_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ TR_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: TR_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar territory_registry" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "territory_registry", enabled: true });
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();
  });

  it("com o prontuário: liga utilizável; desliga com dois cliques; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ TR_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        const on = body.variables.enabled === true;
        listed = [ on ? TR_ON : { ...TR_OFF, changedAt: "2026-10-23T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar territory_registry" }));
    expect((await screen.findByRole("status")).textContent).toBe("territory_registry ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar territory_registry" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "territory_registry", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("territory_registry desligada.");
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("missing desconhecido aparece cru", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ { ...TR_ON, usable: false, missing: [ "team_kind_missing" ] } ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("team_kind_missing")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined/);
  });

  it("produção: aparece e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ TR_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("territory_registry")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod24a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("territory_registry")` é `null` e a aba não acha "Cadastro individual e domiciliar". (Os testes da frase do módulo 19, do código cru e do fluxo de dois cliques já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, troque

```ts
  care_lines: "Linhas de cuidado"
};
```

por

```ts
  care_lines: "Linhas de cuidado",
  // Módulo 24a (ADR 0042): exige clinical_record (missing:
  // clinical_record_disabled, já traduzido acima); todos os ambientes, sem simulado.
  territory_registry: "Cadastro individual e domiciliar"
};
```

Em `README.md`, logo depois do parágrafo **Linhas de cuidado (módulo 23).** (antes de **Documentos clínicos (módulo 19c).**), acrescente:

```markdown
**Cadastro individual e domiciliar (módulo 24a).** O interruptor
`territory_registry` ("Cadastro individual e domiciliar") liga, no dashboard,
as microáreas das equipes, o território da equipe (domicílios, famílias e
moradores), as fichas de cadastro individual e domiciliar enviadas ao e-SUS e
a validade de 24 meses. Exige `clinical_record` (só cidades em modo `record`):
sem ele, a linha diz "prontuário da atenção primária (clinical_record)
desligado". Existe em todos os ambientes e não tem simulado. Desligar esconde
as telas e para fichas novas; o que já foi gravado fica. Nenhum dado de
cidadão passa por aqui. Ordem de deploy: api → dashboard → maintenance; contra
um api sem o 24a, a chave só não aparece.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod24a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais). Atenção às frases que este plano supõe do código atual: "ligada e utilizável", "ligada, falta pré-requisito", "desligada", "<chave> ligada e utilizável." e "<chave> desligada." (`featureStatus` e `doneText`); se alguma mudou, use a real nos testes.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the territory registry switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: a Task 1 na branch `feat/mod-24a-territory-registry`; o api do 24a na **3046**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 24a).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod24a && cd $M && npm run typecheck && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base anotada na Task 0, **nenhum arquivo de teste novo**, e casos novos em `features.test.ts` e `FeaturesTab.test.tsx`. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G diff origin/main --stat
```

Expected: `status` vazio; o diff só com `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx` e `README.md` (nenhum `schema.graphql` nem `src/gql/*`: o 24a não muda o GraphQL do maintenance).

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §9 F-24.1), o ADR 0042 e o contrato §1 e §11. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; a frase de `clinical_record_disabled` não foi redefinida; o rótulo igual ao do dashboard;
- o README não promete o que o 24a não faz (visita domiciliar, atividade coletiva, app de campo, cadastro pelo cidadão);
- nenhum dado de cidadão; nada de tela nova.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 24a na **3046** (plano do api: banco de dev migrado e semeado **com autorização do usuário**). O proxy do Vite do maintenance troca o Host para `maintenance-api.localhost` e injeta o Origin constante `http://maintenance.localhost:5177` (o api exige Origin igual a `MAINTENANCE_FRONTEND_ORIGIN`), então o Vite do worktree sobe **na 5177**, num container da rede do compose, com `node_modules` num volume próprio, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod24a --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod24a:/app -v maintenance-mod24a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3046 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod24a   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 24a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. **Anote** antes de mexer o estado de `territory_registry` e de `clinical_record` de Curitiba. Roteiro, com screenshot em `evidencias/2026-10-23-modulo-24a/maintenance/`:
1. Cidades → Curitiba → **Funcionalidades**: `territory_registry` com "Cadastro individual e domiciliar" e a descrição do api.
2. Ligar (dois cliques) → "territory_registry ligada e utilizável." (Curitiba em `record` com `clinical_record`).
3. Numa cidade sem prontuário (se houver na semente; senão, desligue `clinical_record` de uma cidade de teste e religue depois): ligar dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado".
4. No dashboard (prova do plano do dashboard), o grupo "Cadastro territorial" aparece em Curitiba; desligar aqui e recarregar lá → some; religar → volta, com os cadastros da prova ainda lá.
5. **Auditoria** mostra as tentativas e os resultados de `setCityFeature` para `territory_registry`.
6. A 375 px de largura: sem rolagem lateral da página.

Depois: devolva `territory_registry` e `clinical_record` ao estado anotado (pela própria tela); `docker rm -f maintenance-mod24a && docker volume rm maintenance-mod24a-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 24a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-24a-territory-registry
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod24a
/opt/homebrew/bin/git branch -d feat/mod-24a-territory-registry
```

Se as partes do api e do dashboard do F-24.1 já estiverem **Done**, mova o card F-24.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 3 e a do dashboard passaram.

---

### Task 3: O módulo 24 (24a) no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-24.1..F-24.8 no board #1 (status real de cada um); o resultado da prova do 24a (planos do api, do dashboard e deste); o que foi entregue em api, dashboard e maintenance.
- Produces: módulo `"24"` no JSON (ciclo 2, depois do `"23"` e antes do `"iu"`), nome "Cadastro individual e domiciliar", `status: "validating"`, com F-24.1..F-24.8; `meta.cycles[1].modules` = "15 a 24 e interface"; o rodapé "o 19, o 20, o 21, o 22, o 23 e o 24 estão em validação".

O nome do módulo no catálogo é o do que foi entregue (24a). Quando o 24b entrar, o plano dele renomeia o módulo para o nome do `docs/modulos/24--domiciliar-e-coletivo.md` ("Atendimento domiciliar e coletivo") e acrescenta as funcionalidades dele (Divergência N2).

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.[1-8] ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod24a -b content/module-24a-catalog origin/main
cd $R/.worktrees/catalog-mod24a && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-24.1..F-24.8 em Done ou Verified; os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário. A suíte do catálogo verde. **Anote**: o JSON termina o ciclo 2 em `"23"` e `"iu"` (`7b9400c` ou mais novo); o rodapé diz "o 19, o 20, o 21, o 22 e o 23 estão em validação"; o ciclo 2 diz "15 a 23 e interface".

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("formatValidating")`, depois da linha com `"23"`:

```ts
    expect(formatValidating(["19", "20", "21", "22", "23", "24"])).toBe("o 19, o 20, o 21, o 22, o 23 e o 24 estão");
```

e, no fim do arquivo:

```ts
describe("módulo 24 (24a)", () => {
  it("entra no ciclo 2 depois do 23: fichas em validação; nada de visita nem app de campo", () => {
    const ids = catalog.modules.map((m) => m.id);
    expect(ids.indexOf("24")).toBe(ids.indexOf("23") + 1);
    const m24 = catalog.modules.find((m) => m.id === "24")!;
    expect(m24.cycle).toBe(2);
    expect(m24.status).toBe("validating");
    expect(m24.name).toBe("Cadastro individual e domiciliar");
    expect(m24.description).toMatch(/24 meses/);
    expect(m24.features.map((f) => f.id)).toEqual(Array.from({ length: 8 }, (_, i) => `F-24.${i + 1}`));
    for (const id of ["f-24.1", "f-24.2", "f-24.3", "f-24.4", "f-24.5", "f-24.6", "f-24.8"]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const fichas = findFeature("f-24.7")!.feature;
    expect(fichas.status).toBe("validating");
    expect(fichas.note).toMatch(/SIAPS/);
    expect(findFeature("f-24.1")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    expect(findFeature("f-24.3")?.feature.description).toMatch(/sem CPF/);
    expect(findFeature("f-24.3")?.feature.description).toMatch(/duplicad/);
    expect(findFeature("f-24.5")?.feature.description).toMatch(/recusa/);
    expect(findFeature("f-24.7")?.feature.surfaces).toEqual([ "bastidores" ]);
    expect(findFeature("f-24.8")?.feature.surfaces).toEqual([ "bastidores" ]);
    const text = JSON.stringify(m24);
    expect(text).not.toMatch(/visita domiciliar|atividade coletiva|tablet|offline|celular/i);
    expect(catalog.meta.cycles.find((c) => c.id === 2)?.modules).toBe("15 a 24 e interface");
  });
});
```

Em `src/App.test.tsx`, troque

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22 e o 23 estão em validação."
```

por

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23 e o 24 estão em validação."
```

e

```ts
    expect(within(c2).getByText("módulos 15 a 23 e interface")).toBeInTheDocument();
```

por

```ts
    expect(within(c2).getByText("módulos 15 a 24 e interface")).toBeInTheDocument();
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts src/App.test.tsx`
Expected: FAIL — não há módulo 24, o ciclo 2 diz "15 a 23" e o rodapé não fala do 24. (O teste de `formatValidating` com seis já passa: a função é genérica.)

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo publicado byte a byte (o script confere), então o diff fica só no que mudou. **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard, regras do api) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

ids = [m["id"] for m in data["modules"]]
assert "24" not in ids, "módulo 24 já existe"
assert data["meta"]["cycles"][1]["modules"] == "15 a 23 e interface", "ciclo 2 mudou: pare e reporte"

data["meta"]["updatedAt"] = today
data["meta"]["cycles"][1]["modules"] = "15 a 24 e interface"

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-24.{n}", "slug": f"f-24.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m24 = {
    "id": "24",
    "cycle": 2,
    "name": "Cadastro individual e domiciliar",
    "status": "validating",
    "description": (
        "A equipe cadastra pela web cada domicílio, família e morador da sua área, no formato das fichas de cadastro do e-SUS, "
        "e sabe por microárea quem tem cadastro válido, quem vence logo e quem falta cadastrar. O cadastro atualizado nos últimos "
        "24 meses é o que conta para o componente de vínculo do financiamento da APS."
    ),
    "features": [
        feature(1, "Interruptor e microáreas",
            "Cada equipe de saúde da família tem suas microáreas e o agente responsável.",
            "A equipe da plataforma liga o cadastro por cidade (só com o prontuário do Rota Saúde). A administração municipal cria as "
            "microáreas de cada equipe, com código de dois dígitos e o agente comunitário responsável, com confirmação em duas etapas.",
            ["plataforma", "secretaria"]),
        feature(2, "Agentes de campo sem conselho profissional",
            "O agente comunitário e o agente de endemias entram no cadastro de profissionais.",
            "O agente comunitário de saúde e o agente de combate às endemias são cadastrados sem registro em conselho, que não existe "
            "para essas ocupações. O CNS continua obrigatório.",
            ["secretaria"]),
        feature(3, "Paciente sem CPF e possíveis duplicados",
            "Recém-nascido e pessoa em situação de rua também são cadastrados.",
            "A pessoa pode ser cadastrada sem CPF, com o motivo da ficha e o CNS. Antes de criar alguém sem CPF, o sistema mostra possíveis "
            "duplicados por nome, nascimento e nome da mãe; um CPF que já pertence a outro paciente é recusado.",
            ["secretaria"]),
        feature(4, "Cadastro domiciliar e família",
            "O imóvel, as condições de moradia e as famílias que moram nele.",
            "O agente ou a equipe registram o endereço, o tipo de imóvel, as condições de moradia e as famílias, com responsável, renda e "
            "número de membros. Cada salvamento guarda uma versão e gera a ficha de cadastro domiciliar e territorial.",
            ["secretaria"]),
        feature(5, "Cadastro individual, recusa e saída",
            "A ficha de cada morador, em seções.",
            "Dados sociodemográficos, condições de saúde e situação de rua, com a mesma validação das fichas do e-SUS. A recusa do cidadão "
            "fica registrada com o termo, e a saída do cadastro tem motivo: mudança de território ou óbito.",
            ["secretaria"]),
        feature(6, "Validade de 24 meses e quem falta cadastrar",
            "A equipe sabe quem precisa ser cadastrado ou atualizado.",
            "Por equipe e microárea, a lista de cadastros vencidos, a vencer em 60 dias e das pessoas da equipe ainda sem cadastro, com "
            "atalho para cadastrar ou atualizar. A leitura nominal fica registrada na trilha de auditoria.",
            ["secretaria"]),
        feature(7, "Fichas de cadastro para o e-SUS",
            "O cadastro feito na web vai para o SIAPS.",
            "Cada versão do cadastro individual ou domiciliar vira a ficha correspondente no formato LEDI, enviada ao PEC da cidade; a "
            "atualização aponta a ficha original.",
            ["bastidores"], status="validating",
            note="As fichas foram conferidas com os exemplos de referência do LEDI; falta o SIAPS aceitar fichas reais da cidade piloto."),
        feature(8, "Equipe e óbito pelo cadastro",
            "A pessoa passa a ser da equipe da microárea onde mora.",
            "Quem tem cadastro individual ativo pertence à equipe da microárea do seu domicílio, inclusive nas linhas de cuidado; a saída "
            "por óbito marca a pessoa como falecida e encerra os acompanhamentos.",
            ["bastidores"]),
    ],
}

data["modules"].insert(ids.index("23") + 1, m24)
open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", [m["id"] for m in data["modules"]][-3:], len(m24["features"]))
PY
```

Expected: `ok ['23', '24', 'iu'] 8`. Se o Step 1 mostrar algum F-24.x (além do F-24.7) que não chegou a Verified, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2. Se a Divergência T7 do plano do dashboard (CNS de quem não tem CPF) tiver decidido CNS obrigatório, troque "com o motivo da ficha e o CNS" por "com o motivo da ficha e o CNS, que passa a ser obrigatório"; se a T8 (instituição de permanência) não tiver entrado, nada muda aqui (a frase não fala dela).

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 24 com 8 funcionalidades; o rodapé e o ciclo com os textos novos.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os três arquivos da task (nenhum `package-lock.json`); no JSON, só o módulo 24 acrescentado, o ciclo 2 e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 24a territory registry to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot (em `evidencias/2026-10-23-modulo-24a/catalogo/`): o **Módulo 24 — Cadastro individual e domiciliar** no ciclo 2, logo depois do 23, com a descrição e as 8 funcionalidades; a F-24.7 "Em validação" com a nota do SIAPS; o rodapé "o 19, o 20, o 21, o 22, o 23 e o 24 estão em validação"; a página `/funcionalidades/f-24.3` com "O que faz", as telas e anterior/próxima dentro do módulo (F-24.2 e F-24.4); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod24a
/opt/homebrew/bin/git push -u origin content/module-24a-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-24a-catalog --title "docs: add module 24a territory registry to the catalog" --body "Adds module 24 (24a, individual and household registration) after module 23 with F-24.1..F-24.8; the LEDI registration records (F-24.7) stay in validation until SIAPS accepts real records from the pilot city; cycle 2 and the footer updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod24a && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-24a-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **N1 — Nome do interruptor e ambientes. Aberta (só registrar):** "Cadastro individual e domiciliar" no maintenance e no dashboard; `territory_registry` em todos os ambientes, sem simulado, com o único pré-requisito `feature:clinical_record` (spec §3). Proposta: o contrato §1/§11 registra o rótulo e "sem `environments`".
2. **N2 — Nome do módulo 24 no catálogo.** O `docs/modulos/24--domiciliar-e-coletivo.md` chama o módulo de "Atendimento domiciliar e coletivo", mas o 24a só entrega o cadastro; o catálogo só afirma o entregue. Proposta: o catálogo usa "Cadastro individual e domiciliar" agora e o plano do 24b renomeia para o nome do módulo, acrescentando F-24.9 em diante.
3. **N3 — Porta do maintenance na prova.** O brief fixa o Vite do maintenance na **5201**, mas o `vite.config.ts` do maintenance injeta o Origin constante `http://maintenance.localhost:5177` e o api exige Origin igual a `MAINTENANCE_FRONTEND_ORIGIN`; na 5201 toda requisição levaria 403. A prova usa a 5177 com o serviço `maintenance` do compose parado (como no 23). Proposta: registrar; derivar o Origin da porta é mudança fora deste módulo.

## Self-review

- **Cobertura (spec §3, §8, §9; contrato §1, §11, §13):** interruptor pelo mecanismo genérico, exigindo `clinical_record` com a frase existente — Task 1 (Review Focus 1); desligar sem apagar — Task 1 (Review Focus 2); código de `missing` desconhecido cru — Task 1 (Review Focus 3); todos os ambientes, sem simulado — Task 1 (Review Focus 4) e N1; README — Task 1; suíte, build e prova no navegador — Task 2; catálogo público com F-24.1..F-24.8, F-24.7 em validação, sem prometer 24b/24c — Task 3 (Review Focus 5); cards — Tasks 0, 2 e 3.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); a entrada do api (Task 0, com a regra escrita para diferença); status dos cards F-24.x e o texto do catálogo marcado como rascunho a conferir contra o entregue (Task 3); a porta da prova é a 5177 (Task 2, N3).
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary` existem em `src/lib/features.ts`; `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem no topo de `FeaturesTab.test.tsx`; `catalog`, `findFeature` e `formatValidating` são os já usados em `catalog.test.ts` (módulo 23); o rótulo "Cadastro individual e domiciliar" é o mesmo da Task 1 do plano do dashboard.
- **Review Focus:** 1 — Task 1 ("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19"); 2 — Task 1 ("com o prontuário: liga utilizável; desliga com dois cliques; nada fala em apagar"); 3 — Task 1 ("missing desconhecido aparece cru"); 4 — Task 1 ("produção: aparece e nenhum aviso de simulado"); 5 — Task 3 ("entra no ciclo 2 depois do 23: fichas em validação; nada de visita nem app de campo").
