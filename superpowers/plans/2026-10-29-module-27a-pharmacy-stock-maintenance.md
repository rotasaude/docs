# Módulo 27a — Estoque e dispensação (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 27a** na branch `feat/mod-27a-pharmacy-stock` (worktree `apps/api/.claude/mod27a`, servidor de dev na porta **3049**) com `pharmacy` em `Platform::Features::CATALOG` **sem** `requires` e sem `environments` (contrato §1 e §10). Contrato §12: api → dashboard → **maintenance** → catálogo público. As Tasks 0–1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 precisa do api do 27a. A Task 3 (catálogo público) vem **por último**, depois das provas e das entregas de api, dashboard e maintenance.

**Goal:** Lado maintenance de F-27.1: o interruptor `pharmacy` aparece na aba Funcionalidades com nome em português ("Farmácia e almoxarifado", o mesmo do dashboard) e liga pelo mecanismo genérico **sem** pré-requisito; e as F-27.1..F-27.7 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única), como o módulo **27a**.

**Architecture:** O liga/desliga não muda (`setCityFeature`, três estados, `missing` vindo do api): entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts`, e os testes fixam o fluxo genérico da aba para a chave nova — liga "ligada e utilizável" em qualquer modo de prontuário, desliga com dois cliques, sem falar em apagar. Nenhuma tela nova, nenhum SDL novo: o 27a não tem dado de plataforma (locais, equipe, prazos, mínimos e estoque são da cidade, no dashboard). O README do maintenance ganha o parágrafo do módulo. No catálogo, o módulo `"27a"` entra no ciclo 2 depois do `"26a"`, por script que reescreve o JSON byte a byte.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md` (§1 intenção e antiobjetivos, §2 decisões 2, 4, 10 e 12 e as do coordenador, §7 entrega, §8 F-IDs, §9 riscos), `docs/adr/0045.md` e o contrato `docs/superpowers/plans/2026-10-29-module-27a-pharmacy-stock-contracts.md` §1, §10 e §12. Base: Task 1 do plano do maintenance do 25 (`surveillance`, também sem pré-requisito) e Task 3 do do 26a (catálogo). As diferenças estão em "Divergências propostas ao contrato", M1–M4.

## Global Constraints

- Interruptor (spec decisão 12; contrato §1 e §10), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `pharmacy` **não** tem pré-requisito (`requires` vazio): liga "ligada e utilizável" em cidade de modo `off`, `integrated` ou `record`; a coluna "o que falta" diz "nada";
  - existe em todos os ambientes, produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele;
  - desligar não apaga nada (movimentos, lotes e dispensações ficam; a dispensação é retida como o prontuário — spec decisão 12); a aba não diz o contrário.
- Nome: `pharmacy` → "Farmácia e almoxarifado" (o mesmo de `src/lib/features.ts` do dashboard, Task 1 do plano do dashboard).
- O maintenance não vê nada de cidade: nenhum local, lote, saldo, receita, dispensação ou paciente passa por aqui.
- Valor desconhecido (api mais novo: código de `missing` ou de erro) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): nenhuma tela nova; `FeaturesTab.tsx` **não muda**; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte. Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); o que só foi provado com dado de exemplo fica `validating` com nota; nada promete o que está fora do 27a: envio ao BNAFAR/Hórus (27b), SNGPC, compras, empenho, licitação, componente especializado, Farmácia Popular, materiais e outros itens fora do catálogo de medicamentos, ou consulta de disponibilidade pelo cidadão; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-27.1..F-27.7 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-27.1: Done do F-27.1 quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod27a`, `C=$R/.worktrees/catalog-mod27a`, `A=$R/apps/api/.claude/mod27a`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-27a-pharmacy-stock`; catálogo em `$C`, branch `content/module-27a-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task.
- maintenance — testes: `bin/test maintenance -w maintenance-mod27a <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (os testes chamam `cleanup`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- catálogo — testes: `cd $C && npx vitest run` (o catálogo não está no `bin/test`; é estático, sem banco).
- O api do 27a roda em dev na **3049** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177` (o proxy de dev injeta o Origin constante `http://maintenance.localhost:5177`, que o api exige); a prova sobe o Vite do worktree **na 5177** com o serviço `maintenance` do compose parado; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login e digita o TOTP. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade sem prontuário (modo `off`).** Ligar `pharmacy` ali dá "ligada e utilizável" com "nada" no que falta — nunca "ligada, falta pré-requisito" (a farmácia também dispensa receita externa). Teste: Task 1, "sem prontuário: liga utilizável, nada falta; desliga com dois cliques; nada fala em apagar".
2. **Desligar com estoque e dispensações gravados.** O fluxo de dois cliques desliga e a aba não fala em apagar. Teste: Task 1, o mesmo caso (o texto do corpo nunca casa `/apaga|exclu/`).
3. **Ambiente de produção.** A chave aparece e não acende aviso de simulado. Teste: Task 1, "produção: aparece e nenhum aviso de simulado".
4. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `pharmacy_settings_missing`). Aparece cru, sem `undefined`. Teste: Task 1, "pharmacy tem nome em português, o mesmo do dashboard; missing desconhecido cru".
5. **Catálogo prometendo o que não foi entregue.** Nada de BNAFAR, Hórus, SNGPC, compras, licitação, Farmácia Popular ou materiais; a F-27.4 diz que o saldo nunca fica negativo e a F-27.5 que receita cancelada ou vencida não é dispensada. Teste: Task 3, "entra no ciclo 2 depois do 26a: sem BNAFAR nem SNGPC, saldo nunca negativo, receita vencida não dispensa".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-27.1..F-27.7, conferência do main e do api do 27a, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome do interruptor e o fluxo genérico da aba fixado para ele | 1 |
| — | suíte, codegen:check, build, revisão, prova no navegador | 2 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx` | módulo 27a com F-27.1..F-27.7, rodapé e ciclo 2 | 3 |

**Estratégia de teste:** função pura (`featureLabel`, `missingText`) e o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation, nos dois cenários (cidade sem prontuário; produção) (Task 1); suíte inteira com as guardas de tema, `codegen:check`, build e a prova no navegador contra o api do 27a (Task 2). Catálogo: `validateCatalog`, o teste do módulo 27a, o rodapé, o ciclo e a contagem de módulos (Task 3).

---

### Task 0: Cards, conferência do main e do api do 27a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`eb18725` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-27.1..F-27.7 em In Progress (ou além); worktree `$M` na branch `feat/mod-27a-pharmacy-stock`; a base anotada; a entrada do catálogo de interruptores do api anotada (com a **descrição real**, usada nos testes da Task 1).

- [ ] **Step 1: Mova os cards F-27.1..F-27.7 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-27\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified ficam como estão. Para cada um antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os sete itens em In Progress ou além. Se algum card não existir, **pare e reporte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira o main do maintenance e o api do 27a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod27a
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n '  care_regulation: "Regulação municipal"\|pharmacy'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.test.tsx | grep -n '^describe("FeaturesTab — regulação municipal (módulo 26a)"\|^function featuresReply\|^function renderTab\|FAKE_MIDDLEWARE_NOTICE } from'
$G -C $R/apps/maintenance show origin/main:src/lib/features.test.ts | grep -n '^describe("módulo 26a"'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Regulação municipal (módulo 26a).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n -E 'key: "pharmacy"' -A4 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: a âncora `care_regulation: "Regulação municipal"` (última linha do `FEATURE_LABELS`, sem vírgula) e **nenhuma** linha com `pharmacy`; em `FeaturesTab.test.tsx`, `features.test.ts` e `README.md`: as âncoras (os auxiliares `featuresReply`, `renderTab`, `reply`, `bodyOf`, `calls`, `Feature` e o `FAKE_MIDDLEWARE_NOTICE` importado no topo). Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 27a: a entrada `pharmacy` **sem** `requires` e **sem** `environments`. Com pré-requisito ou `environments` → **pare e reporte**: diverge da spec (decisão 12) e do contrato §1. **Anote a `description` exata**: ela substitui o rascunho `PH_OFF.description` da Task 1. Se o api ainda não tiver a entrada, a Task 1 segue com o rascunho e a Task 2 confere.

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod27a -b feat/mod-27a-pharmacy-stock origin/main
cd $R/.worktrees/maintenance-mod27a && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod27a
```

Expected: suíte verde e `base-ok`. **Anote** a base (arquivos e testes): a Task 2 espera **nenhum arquivo de teste novo** e casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: O interruptor `pharmacy` — nome e o fluxo genérico da aba

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo. Esta task dá o nome e fixa esse comportamento para a chave nova, sem pré-requisito.

**Interfaces:**
- Consumes: `featureLabel`, `missingText` (existentes em `features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature`, `FAKE_MIDDLEWARE_NOTICE` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("pharmacy")` → "Farmácia e almoxarifado".

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 27a", () => {
  it("pharmacy tem nome em português, o mesmo do dashboard; missing desconhecido cru", () => {
    expect(featureLabel("pharmacy")).toBe("Farmácia e almoxarifado");
    expect(missingText("pharmacy_settings_missing")).toBe("pharmacy_settings_missing");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (a `description` é a do "Valores fixados" 1 do plano do api; se a anotada na Task 0, Step 2 for outra, use a anotada):

```tsx
describe("FeaturesTab — farmácia e almoxarifado (módulo 27a)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  // Descrição real do catálogo do api (Platform::Features::CATALOG), anotada na Task 0.
  const PH_OFF: Feature = {
    key: "pharmacy",
    description: "Farmácia e almoxarifado: estoque por local com lotes e validade, transferências, inventário e dispensação pela receita",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const PH_ON: Feature = { ...PH_OFF, enabled: true, usable: true, changedAt: "2026-10-29T13:00:00Z", changedBy: "dev@local" };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ PH_OFF ], { recordMode: "off" })));
    renderTab();
    expect(await screen.findByText("pharmacy")).not.toBeNull();
    expect(screen.getByText("Farmácia e almoxarifado")).not.toBeNull();
    expect(screen.getByText(PH_OFF.description)).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("sem prontuário: liga utilizável, nada falta; desliga com dois cliques; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ PH_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? PH_ON : { ...PH_OFF, changedAt: "2026-10-29T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "off" }));
    });
    renderTab();
    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar pharmacy" }));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "pharmacy", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe("pharmacy ligada e utilizável.");
    expect(screen.getByText("nada")).not.toBeNull();
    expect(screen.getByText("ligada e utilizável")).not.toBeNull();
    expect(screen.queryByText("ligada, falta pré-requisito")).toBeNull();
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar pharmacy" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "pharmacy", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("pharmacy desligada.");
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("produção: aparece e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ PH_ON ], { recordMode: "record" })));
    renderTab();
    expect(await screen.findByText("pharmacy")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod27a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("pharmacy")` é `null` e a aba não acha "Farmácia e almoxarifado". (O fluxo de dois cliques e o código cru já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, depois de `care_regulation: "Regulação municipal"`, a vírgula e:

```ts
  // Módulo 27a (ADR 0045): sem pré-requisito (a receita externa não depende
  // do prontuário). Todos os ambientes, sem simulado.
  pharmacy: "Farmácia e almoxarifado"
```

Em `README.md`, logo depois do parágrafo **Regulação municipal (módulo 26a).** (antes de **Documentos clínicos (módulo 19c).**):

```markdown
**Farmácia e almoxarifado (módulo 27a).** O interruptor `pharmacy`
("Farmácia e almoxarifado") liga, no dashboard, o almoxarifado central e a
farmácia de cada unidade que dispensa: locais e equipe por local, entradas
por nota ou doação com lote, validade e conversão da embalagem, inventário de
abertura, transferência com recebimento conferido lote a lote, ajustes e
perdas (com o código do autenticador e, acima do limite da cidade, aprovação
de outra pessoa), posição por local e lote, extrato, alertas de validade e de
estoque mínimo e a dispensação pela receita digital ou externa, com o
controlado só pelo farmacêutico e a retenção da receita. Não tem
pré-requisito: liga em qualquer modo de prontuário. Existe em todos os
ambientes e não tem simulado. Desligar esconde as telas; movimentos, lotes e
dispensações ficam. Nada vai ao BNAFAR nesta entrega (27b). Nenhum dado de
cidadão passa por aqui. Ordem de deploy: api → dashboard → maintenance;
contra um api sem o 27a, a chave só não aparece.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod27a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais). Se as frases "ligada e utilizável", "nada", "<chave> ligada e utilizável." e "<chave> desligada." mudaram no código, use as reais nos testes.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the pharmacy switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: a Task 1 na branch `feat/mod-27a-pharmacy-stock`; o api do 27a na **3049**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 27a).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod27a && cd $M && npm run typecheck && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base da Task 0, sem arquivo de teste novo, e os casos novos em `features` e `FeaturesTab`. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G diff origin/main --stat
```

Expected: `status` vazio; o diff só com os quatro arquivos da Task 1 (nada em `src/screens/FeaturesTab.tsx`, `schema.graphql` ou `src/gql/`). Confira também que a `description` do teste é a do api (Task 0, Step 2): `grep -n 'key: "pharmacy"' -A4 $A/app/services/platform/features.rb`.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§2 decisão 12, §7, §8 F-27.1), o ADR 0045 e o contrato §1, §10 e §12. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; o rótulo igual ao do dashboard; nenhum pré-requisito inventado aqui;
- o README não promete BNAFAR, SNGPC nem disponibilidade ao cidadão;
- nada de local, lote, receita ou cidadão aqui.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 27a na **3049** (plano do api: banco de dev migrado **com autorização do usuário**). O proxy do Vite do maintenance injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod27a --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod27a:/app -v maintenance-mod27a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3049 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod27a   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 27a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor e digita o TOTP quando pedido; não digite senha nem TOTP. **Anote** antes de mexer o estado de `pharmacy` das cidades usadas. Roteiro, com screenshot em `evidencias/2026-10-29-modulo-27a/maintenance/`:
1. Cidades → Curitiba → **Funcionalidades**: `pharmacy` com "Farmácia e almoxarifado" e a descrição do api; ligar (dois cliques) → "pharmacy ligada e utilizável."; o que falta: "nada".
2. Numa cidade da semente em modo `off` (sem prontuário): ligar `pharmacy` → também "ligada e utilizável"; desligar de volta (dois cliques), sem frase de apagar.
3. **Auditoria** mostra as tentativas de `setCityFeature` para `pharmacy`.
4. No dashboard (prova do plano do dashboard), com `pharmacy` ligado em Curitiba, o grupo "Farmácia" aparece; desligado aqui e recarregado lá, some (o estoque e as dispensações ficam); religado, volta com tudo.
5. A 375 px de largura: sem rolagem lateral da página (a tabela rola dentro do `TableScroll`).

Depois: devolva `pharmacy` ao estado anotado, ou ao que a prova do dashboard precisar; `docker rm -f maintenance-mod27a && docker volume rm maintenance-mod27a-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 27a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-27a-pharmacy-stock
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod27a
/opt/homebrew/bin/git branch -d feat/mod-27a-pharmacy-stock
```

Se as partes do api e do dashboard do F-27.1 já estiverem **Done**, mova o card F-27.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 3 e a do dashboard passaram.

---

### Task 3: O módulo 27a no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-27.1..F-27.7 no board #1 (status real de cada um); o resultado das provas do 27a (planos do api, do dashboard e deste); o que foi entregue em api, dashboard e maintenance.
- Produces: módulo `"27a"` no JSON (ciclo 2, depois do `"26a"` e antes do `"iu"`; sem `label`, para contar no rodapé — M2), nome "Farmácia e almoxarifado", `status: "validating"`, com F-27.1..F-27.7; `meta.cycles[1].modules` = "15 a 27a e interface"; o rodapé "o 19, o 20, o 21, o 22, o 23, o 24, o 25, o 26a e o 27a estão em validação"; 28 cabeçalhos de módulo na lista.

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-27\\.[1-7] ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod27a -b content/module-27a-catalog origin/main
cd $R/.worktrees/catalog-mod27a && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-27.1..F-27.7 em Done ou Verified; os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário. A suíte do catálogo verde. **Anote**: o JSON termina o ciclo 2 em `"26a"` e `"iu"` (`4807d7e` ou mais novo); o rodapé diz "o 19, o 20, o 21, o 22, o 23, o 24, o 25 e o 26a estão em validação"; o ciclo 2 diz "15 a 26a e interface"; a lista tem 27 cabeçalhos de módulo.

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("formatValidating")`, depois da linha com `"26a"`:

```ts
    expect(formatValidating(["19", "20", "21", "22", "23", "24", "25", "26a", "27a"])).toBe("o 19, o 20, o 21, o 22, o 23, o 24, o 25, o 26a e o 27a estão");
```

e, no fim do arquivo:

```ts
describe("módulo 27a", () => {
  it("entra no ciclo 2 depois do 26a: sem BNAFAR nem SNGPC, saldo nunca negativo, receita vencida não dispensa", () => {
    const ids = catalog.modules.map((m) => m.id);
    expect(ids.indexOf("27a")).toBe(ids.indexOf("26a") + 1);
    expect(ids.indexOf("iu")).toBe(ids.indexOf("27a") + 1);
    const m27 = catalog.modules.find((m) => m.id === "27a")!;
    expect(m27.cycle).toBe(2);
    expect(m27.status).toBe("validating");
    expect(m27.name).toBe("Farmácia e almoxarifado");
    expect(m27.label).toBeUndefined();
    expect(m27.features.map((f) => f.id)).toEqual(Array.from({ length: 7 }, (_, i) => `F-27.${i + 1}`));
    for (let i = 1; i <= 7; i++) expect(findFeature(`f-27.${i}`)?.feature.status).toBe("available");
    expect(findFeature("f-27.1")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    for (let i = 2; i <= 7; i++) expect(findFeature(`f-27.${i}`)?.feature.surfaces).toEqual([ "secretaria" ]);
    expect(findFeature("f-27.2")?.feature.description).toMatch(/embalagem/);
    expect(findFeature("f-27.3")?.feature.description).toMatch(/em trânsito/);
    expect(findFeature("f-27.4")?.feature.description).toMatch(/nunca fica negativo/);
    expect(findFeature("f-27.4")?.feature.description).toMatch(/10%/);
    expect(findFeature("f-27.5")?.feature.description).toMatch(/cancelada ou vencida/);
    expect(findFeature("f-27.5")?.feature.description).toMatch(/vence primeiro/);
    expect(findFeature("f-27.6")?.feature.description).toMatch(/só o farmacêutico/);
    expect(findFeature("f-27.7")?.feature.description).toMatch(/90 dias/);
    const text = JSON.stringify(m27);
    expect(text).not.toMatch(/BNAFAR|H[óo]rus|SNGPC|Farm[áa]cia Popular|licita|empenho|compras? |insumo|material|automaticamente|disponibilidade para o cidad/i);
    expect(catalog.meta.cycles.find((c) => c.id === 2)?.modules).toBe("15 a 27a e interface");
  });
});
```

Em `src/App.test.tsx`, troque

```ts
    // 26 módulos numerados (com o 26a) + o grupo Interface.
    expect(screen.getAllByRole("heading", { level: 3 })).toHaveLength(27);
```

por

```ts
    // 27 módulos numerados (com o 26a e o 27a) + o grupo Interface.
    expect(screen.getAllByRole("heading", { level: 3 })).toHaveLength(28);
```

troque

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23, o 24, o 25 e o 26a estão em validação."
```

por

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23, o 24, o 25, o 26a e o 27a estão em validação."
```

e `"módulos 15 a 26a e interface"` por `"módulos 15 a 27a e interface"`.

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts src/App.test.tsx`
Expected: FAIL — não há módulo 27a, o ciclo 2 diz "15 a 26a", o rodapé não fala do 27a e a lista tem 27 cabeçalhos. (O teste de `formatValidating` com nove já passa: a função é genérica.)

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script (o mesmo do 26a: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo byte a byte e o script confere). **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard, regras do api — limites de 10%, 90 dias e 25 dias são os padrões editáveis por cidade) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

ids = [m["id"] for m in data["modules"]]
assert "27a" not in ids, "módulo 27a já existe"
assert data["meta"]["cycles"][1]["modules"] == "15 a 26a e interface", "ciclo 2 mudou: pare e reporte"

data["meta"]["updatedAt"] = today
data["meta"]["cycles"][1]["modules"] = "15 a 27a e interface"

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-27.{n}", "slug": f"f-27.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m27 = {
    "id": "27a",
    "cycle": 2,
    "name": "Farmácia e almoxarifado",
    "status": "validating",
    "description": (
        "Todo medicamento que sai da farmácia da unidade sai de um lote conhecido, contra uma receita válida, para um "
        "paciente identificado. A secretaria sabe, por local, o que tem, o que vence e o que falta, sem planilha, e o "
        "prontuário mostra o que o paciente retirou."
    ),
    "features": [
        feature(1, "Interruptor, papéis e locais de estoque",
            "Cada farmácia com a sua equipe.",
            "A equipe da plataforma liga a farmácia por cidade. A secretaria cadastra o almoxarifado central e a farmácia "
            "de cada unidade que dispensa, vincula farmacêuticos, auxiliares e almoxarifes a cada local e define os prazos "
            "de validade da receita e os limites de alerta.",
            ["plataforma", "secretaria"]),
        feature(2, "Entradas, lotes e validade",
            "Cada caixa que chega tem lote e validade.",
            "O almoxarife registra a entrada por nota fiscal ou doação, com lote, validade e fabricante, convertendo a "
            "embalagem do fornecedor na unidade de dispensação. Lote vencido não entra, e o inventário de abertura carrega o "
            "saldo inicial de cada local antes de começar.",
            ["secretaria"]),
        feature(3, "Transferência do almoxarifado para a unidade",
            "A unidade confere o que recebeu, lote a lote.",
            "O almoxarifado envia por lote e a quantidade fica em trânsito até a unidade receber. A unidade confere cada "
            "lote, e a diferença entre o enviado e o recebido exige motivo e fica registrada como perda.",
            ["secretaria"]),
        feature(4, "Inventário, ajustes e perdas",
            "O saldo bate com a prateleira, e cada correção tem dono.",
            "Ajuste de inventário, perda e vencimento pedem motivo e o código do autenticador; o ajuste acima de 10% do "
            "saldo do lote espera a aprovação de outra pessoa do mesmo local. Nenhum movimento é editado ou apagado, e o "
            "saldo nunca fica negativo.",
            ["secretaria"]),
        feature(5, "Dispensação pela receita",
            "O farmacêutico dispensa a receita digital pelo código em segundos.",
            "A receita é encontrada pelo código impresso ou pelo paciente, e o sistema sugere o lote que vence primeiro. "
            "Receita cancelada ou vencida não é dispensada; no uso contínuo, a próxima retirada do mesmo item fica "
            "liberada a partir do 25º dia, e o prontuário mostra a última retirada.",
            ["secretaria"]),
        feature(6, "Receita externa e controlados",
            "Receita de fora da rede também tem registro.",
            "A receita de fora da rede é digitada com o profissional, o conselho e a data, e fica auditada. Medicamento "
            "controlado da Portaria 344 só o farmacêutico dispensa, registrando a retenção da receita.",
            ["secretaria"]),
        feature(7, "Alertas, posição e extrato",
            "O que vence e o que falta, por local.",
            "Cada local vê os lotes que vencem em 90 dias, os já vencidos, os itens abaixo do estoque mínimo, as "
            "transferências em trânsito e os ajustes à espera de aprovação. A posição mostra o saldo por local e lote, "
            "e o extrato, cada movimento do lote.",
            ["secretaria"]),
    ],
}

data["modules"].insert(ids.index("26a") + 1, m27)
open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", [m["id"] for m in data["modules"]][-3:], len(m27["features"]))
PY
```

Expected: `ok ['26a', '27a', 'iu'] 7`. Se o Step 1 mostrar algum F-27.x que não chegou a Verified, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 27a com 7 funcionalidades; o rodapé, o ciclo e a contagem com os textos novos.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os três arquivos da task (nenhum `package-lock.json`); no JSON, só o módulo 27a acrescentado, o ciclo 2 e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 27a pharmacy stock to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot (em `evidencias/2026-10-29-modulo-27a/catalogo/`): o **Módulo 27a — Farmácia e almoxarifado** no ciclo 2, logo depois do 26a, com a descrição e as 7 funcionalidades; o rodapé "o 19, o 20, o 21, o 22, o 23, o 24, o 25, o 26a e o 27a estão em validação"; a página `/funcionalidades/f-27.5` com "O que faz", a tela da secretaria e anterior/próxima dentro do módulo (F-27.4 e F-27.6); voltar pelo link abre o módulo (`#m-27a`); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod27a
/opt/homebrew/bin/git push -u origin content/module-27a-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-27a-catalog --title "docs: add module 27a pharmacy stock to the catalog" --body "Adds module 27a (pharmacy stock and dispensing) after module 26a with F-27.1..F-27.7: stock locations and staff, receipts with lot and expiry, transfers with in-transit and checked receipt, adjustments with step-up and second approval, dispensing against the digital or external prescription, controlled substances by the pharmacist, alerts, position and lot statement. No federal reporting (BNAFAR is 27b); cycle 2 and the footer updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod27a && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-27a-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — Nenhum `missing`.** O §1 diz `pharmacy` "sem `requires`" e o §10 só "rótulo e descrição". Proposta: o §10 registra que a aba mostra "nada" no que falta em qualquer modo de prontuário (a receita externa e o estoque não dependem do prontuário; a dispensação da receita digital depende dos documentos clínicos do 19c, mas isso é dado da cidade, não pré-requisito do interruptor). Se o api do 27a puser `requires`, a Task 0 para.
2. **M2 — O módulo no catálogo.** O §12 diz "catálogo público" sem dizer como o 27a aparece. Como no 26a (M2 de lá): `id: "27a"` (cabeçalho "Módulo 27a", âncora `#m-27a`, "o 27a" no rodapé), sem `label`; o 27b entra depois como `"27b"` com as suas F-27.x.
3. **M3 — O rótulo.** O §10 pede "rótulo e descrição" sem fixar o rótulo. Proposta: "Farmácia e almoxarifado" no maintenance e no dashboard (Task 1 dos dois planos); a descrição é a do `Platform::Features::CATALOG` do api (a aba só a mostra).
4. **M4 — Ambientes.** O contrato não diz; a spec não prevê simulado. Proposta: `pharmacy` sem `environments` (produção inclusive) e nenhum aviso de simulado (Task 1, "produção").

## Self-review

- **Cobertura (spec §2 decisões 2, 4, 10 e 12, §7, §8 F-27.1 e a entrega do catálogo; contrato §1, §10, §12):** interruptor pelo mecanismo genérico, sem pré-requisito — Task 1 (Review Focus 1 e 4; M1); desligar sem apagar — Task 1 (Review Focus 2); todos os ambientes, sem simulado — Task 1 (Review Focus 3; M4); README — Task 1; suíte, build e prova no navegador — Task 2; catálogo público com F-27.1..F-27.7, sem prometer BNAFAR, SNGPC, compras ou materiais — Task 3 (Review Focus 5; M2); cards — Tasks 0, 2 e 3. Locais, equipe, prazos, mínimos e estoque são da cidade (plano do dashboard): nenhuma tela aqui.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); a entrada e a `description` do api (Task 0, com a regra escrita para diferença); status dos cards F-27.x e o texto do catálogo marcado como rascunho a conferir contra o entregue (Task 3).
- **Tipos e nomes:** `featureLabel`, `missingText`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem no maintenance (conferidos no Step 2 da Task 0); `catalog`, `findFeature`, `formatValidating` são os de `catalog.test.ts`; o rótulo "Farmácia e almoxarifado" é o mesmo da Task 1 do plano do dashboard; o id `"27a"` é o mesmo no script, no teste e no rodapé.
- **Review Focus:** 1 e 2 — Task 1 ("sem prontuário: liga utilizável, nada falta; desliga com dois cliques; nada fala em apagar"); 3 — Task 1 ("produção: aparece e nenhum aviso de simulado"); 4 — Task 1 ("pharmacy tem nome em português…; missing desconhecido cru"); 5 — Task 3 ("entra no ciclo 2 depois do 26a…").
- **Proporção:** os blocos de código são testes, o rótulo, o parágrafo do README e o script do catálogo (o texto do catálogo é copy que a spec fixa); nada de tela nova.
