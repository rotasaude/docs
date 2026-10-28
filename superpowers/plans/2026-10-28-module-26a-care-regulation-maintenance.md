# Módulo 26a — Regulação municipal (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 26a** na branch `feat/mod-26a-care-regulation` (worktree `apps/api/.claude/mod26a`, servidor de dev na porta **3048**) com `care_regulation` em `Platform::Features::CATALOG` com `requires: %w[feature:clinical_record]` e sem `environments` (contrato §1 e §10). Contrato §12: api → dashboard e wpda → **maintenance** → catálogo público. As Tasks 0–1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 precisa do api do 26a. A Task 3 (catálogo público) vem **por último**, depois das provas e das entregas de api, dashboard, wpda e maintenance.

**Goal:** Lado maintenance de F-26.1: o interruptor `care_regulation` aparece na aba Funcionalidades com nome em português ("Regulação municipal", o mesmo do dashboard) e liga pelo mecanismo genérico **com** o pré-requisito `clinical_record` (sem ele, "ligada, falta pré-requisito" com a frase do módulo 19); e as F-26.1..F-26.7 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única), como o módulo **26a**.

**Architecture:** O liga/desliga não muda (`setCityFeature`, três estados, `missing` vindo do api): entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts`, e os testes fixam o fluxo genérico da aba para a chave nova — sem `clinical_record`, o `missing` `clinical_record_disabled` já traduzido pelo módulo 19 aparece; com ele, "ligada e utilizável". Nenhuma tela nova, nenhum SDL novo: o 26a não tem dado de plataforma (prazos, lista do enfermeiro, prestadores e cotas são da cidade, no dashboard). O README do maintenance ganha o parágrafo do módulo. No catálogo, o módulo `"26a"` entra no ciclo 2 depois do `"25"`, por script que reescreve o JSON byte a byte.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-28-module-26a-care-regulation-design.md` (§1 intenção e antiobjetivos, §2 decisões 1, 8, 9 e 12 e as do coordenador, §7 entrega, §8 F-IDs, §9 riscos), `docs/adr/0044.md` e o contrato `docs/superpowers/plans/2026-10-28-module-26a-care-regulation-contracts.md` §1, §10 e §12. Base: Task 1 do plano do maintenance do 23 (`care_lines`, que também exige `clinical_record`) e Tasks 1, 5 e 6 do do 25. As diferenças estão em "Divergências propostas ao contrato", M1–M3.

## Global Constraints

- Interruptor (spec decisão 12; contrato §1 e §10), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `care_regulation` **requer** `clinical_record` (o pedido nasce na consulta): sem ele ligado e utilizável, a aba mostra "ligada, falta pré-requisito" e o `missing` `clinical_record_disabled` com a frase que já existe ("prontuário da atenção primária (clinical_record) desligado"); a frase **não** é redefinida;
  - existe em todos os ambientes, produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele;
  - desligar não apaga nada (os pedidos ficam, retidos como o prontuário — spec decisão 11); a aba não diz o contrário.
- Nome: `care_regulation` → "Regulação municipal" (o mesmo de `src/lib/features.ts` do dashboard, Task 1 do plano do dashboard).
- O maintenance não vê nada de cidade: nenhum pedido, paciente, prestador, cota ou prazo passa por aqui.
- Valor desconhecido (api mais novo: código de `missing` ou de erro) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): nenhuma tela nova; `FeaturesTab.tsx` **não muda**; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte. Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); o que só foi provado com dado de exemplo fica `validating` com nota; nada promete escrita no SISREG, no e-SUS Regulação ou em sistema do estado, envio automático, teto em dinheiro, APAC/OCI ou RNDS (são 26b/26c ou fora do escopo); o cidadão vê situação, data e local, **nunca a posição na fila**; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-26.1..F-26.7 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-26.1: Done do F-26.1 quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod26a`, `C=$R/.worktrees/catalog-mod26a`, `A=$R/apps/api/.claude/mod26a`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-26a-care-regulation`; catálogo em `$C`, branch `content/module-26a-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task.
- maintenance — testes: `bin/test maintenance -w maintenance-mod26a <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (os testes chamam `cleanup`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- catálogo — testes: `cd $C && npx vitest run` (o catálogo não está no `bin/test`; é estático, sem banco).
- O api do 26a roda em dev na **3048** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177` (o proxy de dev injeta o Origin constante `http://maintenance.localhost:5177`, que o api exige); a prova sobe o Vite do worktree **na 5177** com o serviço `maintenance` do compose parado; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login e digita o TOTP. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade sem prontuário (modo `off` ou `integrated`).** Ligar `care_regulation` ali dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado" — nunca "ligada e utilizável". Teste: Task 1, "sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19".
2. **Desligar com pedidos na fila.** O fluxo de dois cliques desliga e a aba não fala em apagar (os pedidos ficam). Teste: Task 1, "com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar".
3. **Ambiente de produção.** A chave aparece e não acende aviso de simulado. Teste: Task 1, "produção: aparece e nenhum aviso de simulado".
4. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `regulation_settings_missing`). Aparece cru, sem `undefined`. Teste: Task 1, "care_regulation tem nome em português; missing desconhecido cru".
5. **Catálogo prometendo o que não foi entregue.** Nada de escrita no SISREG, envio automático, teto em dinheiro, APAC, OCI ou RNDS; a F-26.7 diz que o cidadão nunca vê a posição. Teste: Task 3, "entra no ciclo 2 depois do 25: sem SISREG automático, sem posição para o cidadão, convivência em validação".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-26.1..F-26.7, conferência do main e do api do 26a, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome do interruptor e o fluxo genérico da aba fixado para ele | 1 |
| — | suíte, codegen:check, build, revisão, prova no navegador | 2 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx` | módulo 26a com F-26.1..F-26.7, rodapé e ciclo 2 | 3 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation, nos três cenários do pré-requisito (sem prontuário, com prontuário, produção) (Task 1); suíte inteira com as guardas de tema, `codegen:check`, build e a prova no navegador contra o api do 26a (Task 2). Catálogo: `validateCatalog`, o teste do módulo 26a, o rodapé e o ciclo (Task 3).

---

### Task 0: Cards, conferência do main e do api do 26a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`83239db` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-26.1..F-26.7 em In Progress (ou além); worktree `$M` na branch `feat/mod-26a-care-regulation`; a base anotada; a entrada do catálogo de interruptores do api anotada.

- [ ] **Step 1: Mova os cards F-26.1..F-26.7 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-26\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified ficam como estão. Para cada um antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os sete itens em In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main do maintenance e o api do 26a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod26a
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n '  surveillance: "Vigilância epidemiológica"\|  clinical_record_disabled: "prontuário da atenção primária (clinical_record) desligado",\|care_regulation'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.test.tsx | grep -n '^describe("FeaturesTab — vigilância epidemiológica (módulo 25)"\|^function featuresReply\|^function renderTab\|^export const FAKE_MIDDLEWARE_NOTICE\|FAKE_MIDDLEWARE_NOTICE'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Vigilância epidemiológica (módulo 25).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n -E 'key: "care_regulation"' -A4 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: a âncora do 25, a frase `clinical_record_disabled` do módulo 19 e **nenhuma** linha com `care_regulation`; em `FeaturesTab.test.tsx` e `README.md`: as âncoras (os auxiliares `featuresReply`, `renderTab`, `reply`, `bodyOf`, `calls`, `Feature` e o `FAKE_MIDDLEWARE_NOTICE` importado no topo). Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 26a: a entrada `care_regulation` com `requires: %w[feature:clinical_record]` e **sem** `environments`. Outro pré-requisito (ou `environments`) → **pare e reporte**: diverge da spec (decisão 12) e do contrato §1. Se o api ainda não tiver a entrada, a Task 1 segue com o contrato e a Task 2 confere.

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod26a -b feat/mod-26a-care-regulation origin/main
cd $R/.worktrees/maintenance-mod26a && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod26a
```

Expected: suíte verde e `base-ok`. **Anote** a base (arquivos e testes): a Task 2 espera **nenhum arquivo de teste novo** e casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: O interruptor `care_regulation` — nome e o fluxo genérico da aba

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo. Esta task dá o nome e fixa esse comportamento para a chave nova, com o pré-requisito.

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes em `features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature`, `FAKE_MIDDLEWARE_NOTICE` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("care_regulation")` → "Regulação municipal".

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 26a", () => {
  it("care_regulation tem nome em português, o mesmo do dashboard; missing desconhecido cru", () => {
    expect(featureLabel("care_regulation")).toBe("Regulação municipal");
    expect(missingText("regulation_settings_missing")).toBe("regulation_settings_missing");
  });

  it("o pré-requisito é o prontuário: a frase do módulo 19, sem redefinir", () => {
    expect(missingText("clinical_record_disabled")).toBe("prontuário da atenção primária (clinical_record) desligado");
    expect(missingSummary([ "clinical_record_disabled" ])).toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx`:

```tsx
describe("FeaturesTab — regulação municipal (módulo 26a)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const RG_OFF: Feature = {
    key: "care_regulation", description: "Regulação municipal: pedido de regulação na consulta, fila por cor e espera, ofertas e cotas por prestador e convivência com o sistema do estado",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const RG_ON: Feature = { ...RG_OFF, enabled: true, usable: true, changedAt: "2026-10-28T13:00:00Z", changedBy: "dev@local" };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ RG_OFF ], { recordMode: "record" })));
    renderTab();
    expect(await screen.findByText("care_regulation")).not.toBeNull();
    expect(screen.getByText("Regulação municipal")).not.toBeNull();
    expect(screen.getByText(RG_OFF.description)).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19", async () => {
    const user = userEvent.setup();
    const RG_BLOCKED: Feature = { ...RG_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ RG_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ RG_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: RG_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();
    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar care_regulation" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "care_regulation", enabled: true });
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
  });

  it("com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ RG_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? RG_ON : { ...RG_OFF, changedAt: "2026-10-28T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();
    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar care_regulation" }));
    expect((await screen.findByRole("status")).textContent).toBe("care_regulation ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar care_regulation" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "care_regulation", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("care_regulation desligada.");
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("produção: aparece e nenhum aviso de simulado", async () => {
    const MW_ON_HERE: Feature = {
      key: "fake_middleware", description: "Middleware simulado (desenvolvimento)", enabled: true, usable: true,
      missing: [], changedAt: "2026-10-28T13:00:00Z", changedBy: "dev@local"
    };
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ RG_ON, MW_ON_HERE ], { recordMode: "record" })));
    renderTab();
    expect(await screen.findByText("care_regulation")).not.toBeNull();
    const notes = screen.getAllByRole("note");
    expect(notes).toHaveLength(1);
    expect(notes[0].textContent).toBe(FAKE_MIDDLEWARE_NOTICE);
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod26a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("care_regulation")` é `null` e a aba não acha "Regulação municipal". (O fluxo de dois cliques, o `missing` traduzido e o código cru já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, depois de `surveillance: "Vigilância epidemiológica"`, a vírgula e:

```ts
  // Módulo 26a (ADR 0044): exige clinical_record (missing:
  // clinical_record_disabled, já traduzido acima); o pedido nasce na consulta.
  // Todos os ambientes, sem simulado.
  care_regulation: "Regulação municipal"
```

Em `README.md`, logo depois do parágrafo **Vigilância epidemiológica (módulo 25).** (antes de **Documentos clínicos (módulo 19c).**):

```markdown
**Regulação municipal (módulo 26a).** O interruptor `care_regulation`
("Regulação municipal") liga, no dashboard, o "Pedir regulação" na consulta
(especialidade ou procedimento, CID, justificativa, cor de risco e guia em
PDF), a fila regulada por cor e tempo de espera, a autorização numa oferta
com saldo, os prestadores com cota mensal, os prazos por cor, a lista do que o
enfermeiro pode pedir, o agendamento externo e o desfecho, o número do sistema
do estado e a exportação para digitar nele; no wpda, "Meus encaminhamentos",
com a situação e nunca a posição na fila. Exige `clinical_record` (só cidades
em modo `record`): sem ele, a linha diz "prontuário da atenção primária
(clinical_record) desligado". Existe em todos os ambientes e não tem simulado.
Desligar esconde as telas; os pedidos ficam. O Rota Saúde não escreve no
SISREG, no e-SUS Regulação nem em sistema do estado. Nenhum dado de cidadão
passa por aqui. Ordem de deploy: api → dashboard e wpda → maintenance; contra
um api sem o 26a, a chave só não aparece.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod26a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais). Se as frases "ligada e utilizável", "ligada, falta pré-requisito", "<chave> ligada e utilizável." e "<chave> desligada." mudaram no código, use as reais nos testes.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the care regulation switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: a Task 1 na branch `feat/mod-26a-care-regulation`; o api do 26a na **3048**.
- Produces: branch do maintenance pronta para merge (depois do merge do api, do dashboard e do wpda do 26a).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod26a && cd $M && npm run typecheck && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base da Task 0, sem arquivo de teste novo, e os casos novos em `features` e `FeaturesTab`. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G diff origin/main --stat
```

Expected: `status` vazio; o diff só com os quatro arquivos da Task 1 (nada em `src/screens/FeaturesTab.tsx`, `schema.graphql` ou `src/gql/`).

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§2 decisões 11 e 12, §7, §8 F-26.1), o ADR 0044 e o contrato §1, §10 e §12. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; o rótulo igual ao do dashboard; o pré-requisito vem do api (nenhum inventado aqui) e a frase do `clinical_record_disabled` não foi redefinida;
- o README não promete escrita no SISREG nem posição para o cidadão;
- nada de pedido, prestador, cota ou cidadão aqui.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 26a na **3048** (plano do api: banco de dev migrado **com autorização do usuário**). O proxy do Vite do maintenance injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod26a --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod26a:/app -v maintenance-mod26a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3048 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod26a   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 26a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor e digita o TOTP quando pedido; não digite senha nem TOTP. **Anote** antes de mexer o estado de `care_regulation` e de `clinical_record` das cidades usadas. Roteiro, com screenshot em `evidencias/2026-10-28-modulo-26a/maintenance/`:
1. Cidades → Curitiba (modo `record`, `clinical_record` ligado) → **Funcionalidades**: `care_regulation` com "Regulação municipal" e a descrição do api; ligar (dois cliques) → "care_regulation ligada e utilizável.".
2. Numa cidade da semente em modo `off` ou `integrated` (ou com `clinical_record` desligado): ligar `care_regulation` → "ligada, falta pré-requisito" e "prontuário da atenção primária (clinical_record) desligado"; desligar de volta.
3. **Auditoria** mostra as tentativas de `setCityFeature` para `care_regulation`.
4. No dashboard (prova do plano do dashboard), com `care_regulation` ligado em Curitiba, o grupo "Regulação" aparece; desligado aqui e recarregado lá, some (os pedidos ficam).
5. A 375 px de largura: sem rolagem lateral da página (a tabela rola dentro do `TableScroll`).

Depois: devolva `care_regulation` (e qualquer `clinical_record` mexido) ao estado anotado, ou ao que a prova do dashboard e do wpda precisar; `docker rm -f maintenance-mod26a && docker volume rm maintenance-mod26a-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api, do dashboard e do wpda do 26a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-26a-care-regulation
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod26a
/opt/homebrew/bin/git branch -d feat/mod-26a-care-regulation
```

Se as partes do api e do dashboard do F-26.1 já estiverem **Done**, mova o card F-26.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 3 e a do dashboard passaram.

---

### Task 3: O módulo 26a no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-26.1..F-26.7 no board #1 (status real de cada um); o resultado das provas do 26a (planos do api, do dashboard, do wpda e deste); o que foi entregue em api, dashboard, wpda e maintenance.
- Produces: módulo `"26a"` no JSON (ciclo 2, depois do `"25"` e antes do `"iu"`; sem `label`, para contar no rodapé — M2), nome "Regulação municipal", `status: "validating"`, com F-26.1..F-26.7; `meta.cycles[1].modules` = "15 a 26a e interface"; o rodapé "o 19, o 20, o 21, o 22, o 23, o 24, o 25 e o 26a estão em validação".

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-26\\.[1-7] ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod26a -b content/module-26a-catalog origin/main
cd $R/.worktrees/catalog-mod26a && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-26.1..F-26.7 em Done ou Verified; os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário. A suíte do catálogo verde. **Anote**: o JSON termina o ciclo 2 em `"25"` e `"iu"` (`2634b21` ou mais novo); o rodapé diz "o 19, o 20, o 21, o 22, o 23, o 24 e o 25 estão em validação"; o ciclo 2 diz "15 a 25 e interface".

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("formatValidating")`, depois da linha com `"25"`:

```ts
    expect(formatValidating(["19", "20", "21", "22", "23", "24", "25", "26a"])).toBe("o 19, o 20, o 21, o 22, o 23, o 24, o 25 e o 26a estão");
```

e, no fim do arquivo:

```ts
describe("módulo 26a", () => {
  it("entra no ciclo 2 depois do 25: sem SISREG automático, sem posição para o cidadão, convivência em validação", () => {
    const ids = catalog.modules.map((m) => m.id);
    expect(ids.indexOf("26a")).toBe(ids.indexOf("25") + 1);
    expect(ids.indexOf("iu")).toBe(ids.indexOf("26a") + 1);
    const m26 = catalog.modules.find((m) => m.id === "26a")!;
    expect(m26.cycle).toBe(2);
    expect(m26.status).toBe("validating");
    expect(m26.name).toBe("Regulação municipal");
    expect(m26.label).toBeUndefined();
    expect(m26.features.map((f) => f.id)).toEqual(Array.from({ length: 7 }, (_, i) => `F-26.${i + 1}`));
    for (const id of [ "f-26.1", "f-26.2", "f-26.3", "f-26.4", "f-26.5", "f-26.7" ]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const state = findFeature("f-26.6")!.feature;
    expect(state.status).toBe("validating");
    expect(state.note).toMatch(/sistema do estado/);
    expect(findFeature("f-26.1")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    expect(findFeature("f-26.7")?.feature.surfaces).toEqual([ "cidadao" ]);
    expect(findFeature("f-26.2")?.feature.description).toMatch(/não impede/);
    expect(findFeature("f-26.3")?.feature.description).toMatch(/motivo/);
    expect(findFeature("f-26.4")?.feature.description).toMatch(/saldo/);
    expect(findFeature("f-26.6")?.feature.description).toMatch(/à mão/);
    expect(findFeature("f-26.7")?.feature.description).toMatch(/nunca a posição/);
    const text = JSON.stringify(m26);
    expect(text).not.toMatch(/automaticamente|envia(m)? ao (SISREG|estado)|integra(do|ção) com o SISREG|teto|APAC|OCI|RNDS|MIRA/i);
    expect(catalog.meta.cycles.find((c) => c.id === 2)?.modules).toBe("15 a 26a e interface");
  });
});
```

Em `src/App.test.tsx`, troque

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23, o 24 e o 25 estão em validação."
```

por

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23, o 24, o 25 e o 26a estão em validação."
```

e `"módulos 15 a 25 e interface"` por `"módulos 15 a 26a e interface"`.

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts src/App.test.tsx`
Expected: FAIL — não há módulo 26a, o ciclo 2 diz "15 a 25" e o rodapé não fala do 26a. (O teste de `formatValidating` com oito já passa: a função é genérica.)

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script (o mesmo do 25: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo byte a byte e o script confere). **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard e do wpda, regras do api) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

ids = [m["id"] for m in data["modules"]]
assert "26a" not in ids, "módulo 26a já existe"
assert data["meta"]["cycles"][1]["modules"] == "15 a 25 e interface", "ciclo 2 mudou: pare e reporte"

data["meta"]["updatedAt"] = today
data["meta"]["cycles"][1]["modules"] = "15 a 26a e interface"

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-26.{n}", "slug": f"f-26.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m26 = {
    "id": "26a",
    "cycle": 2,
    "name": "Regulação municipal",
    "status": "validating",
    "description": (
        "O encaminhamento a especialista ou procedimento sai da consulta com o que o regulador precisa para decidir e entra "
        "numa fila única que respeita a cor de risco e o tempo de espera. A secretaria só autoriza quando há vaga numa "
        "oferta real, acompanha quanto da cota de cada prestador já foi usado e convive com o sistema de regulação do estado."
    ),
    "features": [
        feature(1, "Interruptor, papel de regulador e prazos por cor",
            "Cada cor de risco com o seu prazo, definido pela cidade.",
            "A equipe da plataforma liga a regulação por cidade. A cidade dá o papel de regulador à equipe central, define o "
            "prazo de cada cor (vermelho, amarelo, verde e azul) e a lista do que o enfermeiro pode pedir.",
            ["plataforma", "secretaria"]),
        feature(2, "Pedido de regulação na consulta e guia",
            "O médico pede o encaminhamento sem sair do atendimento.",
            "Na consulta, o profissional escolhe a especialidade ou o procedimento, informa o CID, a justificativa e a cor de "
            "risco sugerida, e a guia de encaminhamento sai em PDF. O pedido vai para a fila ao finalizar e não impede "
            "finalizar a consulta.",
            ["secretaria"]),
        feature(3, "Fila regulada e decisões do regulador",
            "O regulador decide a fila do dia sem telefone.",
            "A fila mostra cada pedido com a cor, o tempo de espera e o prazo, com os vencidos primeiro. O regulador "
            "confirma ou muda a cor, autoriza, devolve para complementar, nega ou cancela, e passar um pedido à frente "
            "de outro mais antigo exige motivo registrado.",
            ["secretaria"]),
        feature(4, "Prestadores, ofertas e cotas",
            "Nenhuma autorização sem vaga.",
            "A secretaria cadastra a rede própria e os prestadores contratados, de consórcio e da PPI, com a cota mensal de "
            "cada especialidade ou procedimento e o saldo à vista. Autorizar consome a cota e cancelar antes do "
            "atendimento devolve.",
            ["secretaria"]),
        feature(5, "Agendamento e desfecho",
            "O paciente recebe data e local.",
            "Na rede própria, a autorização vira pedido de agendamento para a unidade de destino marcar o horário; no "
            "prestador externo, o regulador registra data, hora e local. Depois, registra realizado ou falta: a primeira "
            "falta devolve o pedido à fila e a segunda encerra.",
            ["secretaria"]),
        feature(6, "Convivência com o sistema do estado",
            "Pedidos regulados pelo estado também ficam registrados.",
            "O pedido marcado como regulado pelo estado fica fora da fila municipal e não consome cota, com o número do "
            "SISREG, CARE ou CROSS. Uma planilha reúne o necessário para digitar à mão no sistema do estado.",
            ["secretaria"], status="validating",
            note="Falta a secretaria da cidade piloto digitar pedidos reais no sistema do estado a partir da planilha."),
        feature(7, "Situação do pedido para o cidadão",
            "O cidadão acompanha o encaminhamento pelo celular.",
            "Em Meus encaminhamentos, a pessoa vê se o pedido está aguardando, agendado com data e local, em revisão pela "
            "unidade, não autorizado ou encerrado — nunca a posição na fila, que muda com o risco dos outros pedidos.",
            ["cidadao"]),
    ],
}

data["modules"].insert(ids.index("25") + 1, m26)
open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", [m["id"] for m in data["modules"]][-3:], len(m26["features"]))
PY
```

Expected: `ok ['25', '26a', 'iu'] 7`. Se o Step 1 mostrar algum F-26.x (além do F-26.6) que não chegou a Verified, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 26a com 7 funcionalidades; o rodapé e o ciclo com os textos novos.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os três arquivos da task (nenhum `package-lock.json`); no JSON, só o módulo 26a acrescentado, o ciclo 2 e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 26a care regulation to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot (em `evidencias/2026-10-28-modulo-26a/catalogo/`): o **Módulo 26a — Regulação municipal** no ciclo 2, logo depois do 25, com a descrição e as 7 funcionalidades; a F-26.6 "Em validação" com a nota; o rodapé "o 19, o 20, o 21, o 22, o 23, o 24, o 25 e o 26a estão em validação"; a página `/funcionalidades/f-26.7` com "O que faz", a tela do cidadão e anterior/próxima dentro do módulo (F-26.6); voltar pelo link abre o módulo (`#m-26a`); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod26a
/opt/homebrew/bin/git push -u origin content/module-26a-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-26a-catalog --title "docs: add module 26a care regulation to the catalog" --body "Adds module 26a (municipal care regulation) after module 25 with F-26.1..F-26.7; coexistence with the state system (F-26.6) stays in validation until the pilot city enters real requests from the export; the citizen never sees the queue position; cycle 2 and the footer updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod26a && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-26a-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — O `missing` do pré-requisito. Adotada (plano do api, "Valores fixados" 1: `requires: %w[feature:clinical_record]`).** O §1 diz `requires: ["feature:clinical_record"]`, sem dizer o código que a aba recebe. O mecanismo genérico do api já traduz `feature:clinical_record` para `clinical_record_disabled` (como `care_lines` e `territory_registry`), e o maintenance já tem a frase desde o módulo 19. Proposta: o contrato §10 registra `clinical_record_disabled` como o `missing` do `care_regulation`; nenhum código novo.
2. **M2 — O módulo no catálogo.** O §12 diz "catálogo público" sem dizer como o 26a aparece. O campo `label` do catálogo **tira** o módulo do rodapé (só módulos numerados contam), então "Módulo 26a" por `label` sumiria do resumo. Proposta: `id: "26a"` (o cabeçalho vira "Módulo 26a" sozinho, a âncora `#m-26a` passa no `[\w-]+` e o rodapé diz "o 26a"); o 26b e o 26c entram depois como `"26b"` e `"26c"`, cada um com as suas F-26.x.
3. **M3 — Ordem de entrega com o wpda.** O §12 põe "dashboard e wpda" antes do maintenance; o F-26.1 é de api, dashboard e maintenance, e o F-26.7 é de api e wpda. Proposta: o maintenance espera os três (api, dashboard e wpda) mergeados antes do merge dele (Task 2, Step 4), para a prova da Task 2 desligar e religar o interruptor com as telas de todos os apps no ar.

## Self-review

- **Cobertura (spec §2 decisões 11 e 12, §7, §8 F-26.1 e a entrega do catálogo; contrato §1, §10, §12):** interruptor pelo mecanismo genérico, com o pré-requisito `clinical_record` e a frase do módulo 19 — Task 1 (Review Focus 1 e 4; M1); desligar sem apagar — Task 1 (Review Focus 2); todos os ambientes, sem simulado — Task 1 (Review Focus 3); README — Task 1; suíte, build e prova no navegador — Task 2; catálogo público com F-26.1..F-26.7, F-26.6 em validação, sem prometer escrita no SISREG nem posição para o cidadão — Task 3 (Review Focus 5; M2); cards — Tasks 0, 2 e 3. Prazos, lista do enfermeiro, prestadores e cotas são da cidade (plano do dashboard), não da plataforma: nenhuma tela aqui.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); a entrada do api (Task 0, com a regra escrita para diferença); status dos cards F-26.x e o texto do catálogo marcado como rascunho a conferir contra o entregue (Task 3).
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` e `FAKE_MIDDLEWARE_NOTICE` existem no maintenance (conferidos no Step 2 da Task 0); `catalog`, `findFeature`, `formatValidating` são os de `catalog.test.ts`; o rótulo "Regulação municipal" é o mesmo da Task 1 do plano do dashboard; o id `"26a"` é o mesmo no script, no teste e no rodapé.
- **Review Focus:** 1 — Task 1 ("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19"); 2 — Task 1 ("com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar"); 3 — Task 1 ("produção: aparece e nenhum aviso de simulado"); 4 — Task 1 ("care_regulation tem nome em português…; missing desconhecido cru"); 5 — Task 3 ("entra no ciclo 2 depois do 25: sem SISREG automático, sem posição para o cidadão, convivência em validação").
- **Proporção:** os blocos de código são testes, o rótulo, o parágrafo do README e o script do catálogo (o texto exato do catálogo é copy que a spec fixa); nada de tela nova.
