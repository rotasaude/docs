# Módulo 28 — Saúde bucal (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 28** na branch `feat/mod-28-oral-health` (worktree `apps/api/.claude/mod28`, servidor de dev na porta **3050**) com `dental_care` em `Platform::Features::CATALOG` e `requires: %w[feature:clinical_record]`, sem `environments` (contrato §1 e §13). Contrato §15: api → dashboard → **maintenance** → catálogo público. A Task 1 não depende do api rodando (testes com `fetch` falso); a prova da Task 2 precisa do api do 28. O catálogo público **não** está neste plano (contrato §15, item 4; ver M3).

**Goal:** Lado maintenance de F-28.1: o interruptor `dental_care` aparece na aba Funcionalidades com nome em português ("Saúde bucal", o mesmo do dashboard) e a descrição do api, e liga pelo mecanismo genérico com o pré-requisito `clinical_record` — sem ele, "ligada, falta pré-requisito" com a frase do módulo 19.

**Architecture:** O liga/desliga não muda (`setCityFeature`, três estados, `missing` vindo do api): entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts`; a frase de `clinical_record_disabled` já existe desde o 19b. Os testes fixam o fluxo genérico da aba para a chave nova, como no 26a (`care_regulation`, o mesmo pré-requisito). Nenhuma tela nova, nenhum SDL novo: o 28 não tem dado de plataforma (unidade CEO, metas, lista do TSB/ASB e laboratório são da cidade, no dashboard). O README ganha o parágrafo do módulo.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-10-30-module-28-oral-health-design.md` (§2 decisão 17, §7 entrega, §8 F-28.1), `docs/adr/0046.md` e o contrato `docs/superpowers/plans/2026-10-30-module-28-oral-health-contracts.md` §1, §13 e §15. Base: plano do maintenance do 26a (`2026-10-28-module-26a-care-regulation-maintenance.md`, Tasks 0–2) e do 27a. As diferenças estão em "Divergências propostas ao contrato", M1–M3.

## Global Constraints

- Interruptor (spec decisão 17; contrato §1 e §13), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `dental_care` exige `clinical_record`: sem ele, a aba mostra "ligada, falta pré-requisito" e "prontuário da atenção primária (clinical_record) desligado" (frase existente, não redefinida); com ele, "ligada e utilizável" e "nada";
  - ligar ou desligar `dental_care` só manda `key: "dental_care"`: a aba nunca liga `clinical_record` junto;
  - existe em todos os ambientes, produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele;
  - desligar não apaga nada (consultas, odontograma, planos e próteses ficam; a exclusão LGPD retém tudo, como o prontuário — ADR 0026); a aba não diz o contrário.
- Nome: `dental_care` → "Saúde bucal" (M1; o mesmo de `src/lib/features.ts` do dashboard).
- O maintenance não vê nada de cidade: nenhum dente, odontograma, plano, CEO, prótese ou paciente passa por aqui.
- Valor desconhecido (api mais novo: código de `missing` ou de erro) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova; `src/screens/FeaturesTab.tsx` **não muda**; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte; a tabela da aba rola dentro do `TableScroll`. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-28.1..F-28.8 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-28.1: Done quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod28`, `A=$R/apps/api/.claude/mod28`.
- Worktree **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): `$M`, branch `feat/mod-28-oral-health`. Caminhos de arquivo das tasks são relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod28 <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `jsdom`, `globals: false` (os testes chamam `cleanup`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- O api do 28 roda em dev na **3050**, na rede `rota-saude_default`. Login: `http://maintenance.localhost:5177`, nunca `localhost:5177` (o proxy de dev injeta o Origin `http://maintenance.localhost:5177`, que o api exige); a prova sobe o Vite do worktree **na 5177** com o serviço `maintenance` do compose parado; conta `dev@local` do `db:seed`; o usuário faz o login e digita o TOTP. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade sem prontuário** (`clinical_record` desligado, modo `off` ou `integrated`). Ligar `dental_care` dá "ligada, falta pré-requisito" com a frase do módulo 19 — nunca "ligada e utilizável". Teste: Task 1, "sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19".
2. **Ligar `dental_care` não liga o prontuário junto.** A mutation leva só `key: "dental_care"`; nenhuma chamada com `clinical_record`. Teste: Task 1, o mesmo caso (variáveis exatas e uma só chamada).
3. **Desligar com consultas e odontogramas gravados.** Dois cliques desligam e a aba não fala em apagar (a exclusão LGPD retém). Teste: Task 1, "com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar".
4. **Ambiente de produção.** A chave aparece e não acende aviso de simulado. Teste: Task 1, "produção: aparece e nenhum aviso de simulado".
5. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `ceo_profile_missing`). Aparece cru, sem `undefined`. Teste: Task 1, "dental_care tem nome em português, o mesmo do dashboard; missing desconhecido cru".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-28.1..F-28.8, conferência do main e do api do 28, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome do interruptor e o fluxo genérico da aba fixado para ele | 1 |
| — | suíte, codegen:check, build, revisão, prova no navegador na 5177 | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation, nos três cenários (sem prontuário; com prontuário; produção) (Task 1); suíte inteira com as guardas de tema, `codegen:check`, build e a prova no navegador contra o api do 28 (Task 2).

---

### Task 0: Cards, conferência do main e do api do 28, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`1fdda62` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-28.1..F-28.8 em In Progress (ou além); worktree `$M`; a base anotada; a entrada `dental_care` do api anotada (com a **descrição real**, usada nos testes da Task 1).

- [ ] **Step 1: Mova os cards F-28.1..F-28.8 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-28\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified ficam. Para cada um antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito itens em In Progress ou além. Card ausente → **pare e reporte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira o main do maintenance e o api do 28**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod28
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n '  pharmacy: "Farmácia e almoxarifado"\|clinical_record_disabled:\|dental'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.test.tsx | grep -n '^describe("FeaturesTab — farmácia e almoxarifado (módulo 27a)"\|^function featuresReply\|^function renderTab\|^type Feature'
$G -C $R/apps/maintenance show origin/main:src/lib/features.test.ts | grep -n '^describe("módulo 27a"\|missingSummary'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Farmácia e almoxarifado (módulo 27a).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n 'key: "dental_care"' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: a âncora `pharmacy: "Farmácia e almoxarifado"` (última linha do `FEATURE_LABELS`, sem vírgula), a linha `clinical_record_disabled:` em `MISSING_LABELS` e **nenhuma** linha com `dental`; nos testes e no README, as âncoras (e `missingSummary` já importado em `features.test.ts`). Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 28: `Entry.new(key: "dental_care", description: "...", requires: %w[feature:clinical_record])`, sem `environments`. Outro `requires` ou `environments` → **pare e reporte** (diverge da spec, decisão 17, e do contrato §1). **Anote a `description` exata**: ela substitui o rascunho `DC_OFF.description` da Task 1. Sem a entrada ainda, a Task 1 segue com o rascunho e a Task 2 confere.

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod28 -b feat/mod-28-oral-health origin/main
cd $R/.worktrees/maintenance-mod28 && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod28
```

Expected: suíte verde e `base-ok`. **Anote** a base (arquivos e testes): a Task 2 espera **nenhum arquivo de teste novo**, só casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: O interruptor `dental_care` — nome e o fluxo genérico da aba

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo.

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes em `features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("dental_care")` → "Saúde bucal".

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 28", () => {
  it("dental_care tem nome em português, o mesmo do dashboard; missing desconhecido cru", () => {
    expect(featureLabel("dental_care")).toBe("Saúde bucal");
    expect(missingText("ceo_profile_missing")).toBe("ceo_profile_missing");
  });

  it("o pré-requisito é o prontuário: a frase do módulo 19, sem redefinir", () => {
    expect(missingSummary([ "clinical_record_disabled" ])).toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (a `description` é o rascunho; se a anotada na Task 0, Step 2 for outra, use a anotada):

```tsx
describe("FeaturesTab — saúde bucal (módulo 28)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  // Descrição real do catálogo do api (Platform::Features::CATALOG), anotada na Task 0.
  const DC_OFF: Feature = {
    key: "dental_care",
    description: "Saúde bucal: consulta odontológica com odontograma, plano de tratamento, especialidades do CEO, prótese e ficha FAO",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const DC_ON: Feature = { ...DC_OFF, enabled: true, usable: true, changedAt: "2026-10-30T13:00:00Z", changedBy: "dev@local" };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ DC_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("dental_care")).not.toBeNull();
    expect(screen.getByText("Saúde bucal")).not.toBeNull();
    expect(screen.getByText(DC_OFF.description)).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19", async () => {
    const user = userEvent.setup();
    const DC_BLOCKED: Feature = { ...DC_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ DC_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ DC_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: DC_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar dental_care" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "dental_care", enabled: true });
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect((await screen.findByRole("status")).textContent).toBe(
      "dental_care ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    );
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
  });

  it("com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ DC_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? DC_ON : { ...DC_OFF, changedAt: "2026-10-30T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar dental_care" }));
    expect((await screen.findByRole("status")).textContent).toBe("dental_care ligada e utilizável.");
    expect(screen.getByText("nada")).not.toBeNull();
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar dental_care" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "dental_care", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("dental_care desligada.");
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("produção: aparece e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ DC_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("dental_care")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod28 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("dental_care")` é `null` e a aba não acha "Saúde bucal". (O fluxo de dois cliques, a frase do pré-requisito e o código cru já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, depois de `pharmacy: "Farmácia e almoxarifado"`, a vírgula e:

```ts
  // Módulo 28 (ADR 0046): exige clinical_record (missing:
  // clinical_record_disabled, já traduzido acima); a consulta odontológica é
  // um tipo da consulta do 19a. Todos os ambientes, sem simulado.
  dental_care: "Saúde bucal"
```

Em `README.md`, logo depois do parágrafo **Farmácia e almoxarifado (módulo 27a).** (antes de **Documentos clínicos (módulo 19c).**):

```markdown
**Saúde bucal (módulo 28).** O interruptor `dental_care` ("Saúde bucal")
liga, no dashboard, a consulta odontológica (um tipo da consulta do 19a) com
o odontograma por dente e face, o plano de tratamento, os registros de
especialidade do CEO (periograma, registro periodontal, endodontia), os
pedidos de prótese ao laboratório, a unidade CEO com tipo e especialidades,
as metas do CEO, os indicadores B1/B2 por equipe de saúde bucal e a ficha FAO
pelo exportador LEDI do módulo 16. Exige `clinical_record` (só cidades em modo
`record`): sem ele, a linha diz "prontuário da atenção primária
(clinical_record) desligado". Existe em todos os ambientes e não tem simulado.
Desligar esconde as telas; consultas, odontogramas, planos e próteses ficam
(a exclusão LGPD retém tudo, como o prontuário). Nenhum dado de cidadão passa
por aqui. Ordem de deploy: api → dashboard → maintenance; contra um api sem o
28, a chave só não aparece.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod28 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais). Se as frases "ligada e utilizável", "ligada, falta pré-requisito", "nada" ou as de `doneText` mudaram no código, use as reais nos testes.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the dental care switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: a Task 1 na branch `feat/mod-28-oral-health`; o api do 28 na **3050**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 28).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod28 && cd $M && npm run typecheck && npm run codegen:check && npm run build`
Expected: tudo verde: a base da Task 0, sem arquivo de teste novo, e os casos novos em `features` e `FeaturesTab`. Número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G diff origin/main --stat
grep -n 'key: "dental_care"' -A3 $A/app/services/platform/features.rb
```

Expected: `status` vazio; o diff só com os quatro arquivos da Task 1 (nada em `FeaturesTab.tsx`, `schema.graphql` ou `src/gql/`); a `description` do teste igual à do api.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§2 decisão 17, §7, §8 F-28.1), o ADR 0046 e o contrato §1, §13 e §15. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; nenhuma frase nova para `clinical_record_disabled`; rótulo igual ao do dashboard;
- o README não promete BPA do CEO, escovação supervisionada, radiografia, ortodontia nem implante (fora da entrega, spec);
- nada de dente, plano, CEO ou cidadão aqui.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 28 na **3050** (banco de dev migrado **com autorização do usuário**, plano do api). O Vite do worktree sobe **na 5177**, na rede do compose, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod28 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod28:/app -v maintenance-mod28-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3050 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod28   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 28 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor e digita o TOTP; não digite senha nem TOTP. **Anote** antes de mexer o estado de `dental_care` e de `clinical_record` das cidades usadas. Roteiro, com screenshot em `evidencias/2026-10-30-modulo-28/maintenance/`:
1. Cidades → Curitiba (modo `record`, `clinical_record` ligado) → **Funcionalidades**: `dental_care` com "Saúde bucal" e a descrição do api; ligar (dois cliques) → "dental_care ligada e utilizável."; o que falta: "nada".
2. Numa cidade da semente em modo `off` ou `integrated` (ou com `clinical_record` desligado): ligar `dental_care` → "ligada, falta pré-requisito" e "prontuário da atenção primária (clinical_record) desligado"; `clinical_record` continua como estava; desligar de volta (dois cliques), sem frase de apagar.
3. **Auditoria** mostra as tentativas de `setCityFeature` para `dental_care`.
4. No dashboard (prova do plano do dashboard), com `dental_care` ligado em Curitiba, a consulta odontológica e as telas de saúde bucal aparecem; desligado aqui e recarregado lá, somem (o que foi gravado fica); religado, voltam com tudo.
5. A 375 px de largura: sem rolagem lateral da página (a tabela rola dentro do `TableScroll`).

Depois: devolva `dental_care` (e qualquer `clinical_record` mexido) ao estado anotado, ou ao que a prova do dashboard precisar; `docker rm -f maintenance-mod28 && docker volume rm maintenance-mod28-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 28, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-28-oral-health
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod28
/opt/homebrew/bin/git branch -d feat/mod-28-oral-health
```

Se as partes do api e do dashboard do F-28.1 já estiverem **Done**, mova o card F-28.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 3 e a do dashboard passaram.

---

## Divergências propostas ao contrato

1. **M1 — O rótulo.** O §13 pede "rótulo e descrição" sem fixar o rótulo. Proposta: "Saúde bucal" no maintenance e no dashboard (Task 1 dos dois planos; o nome do módulo na spec); a descrição é a do `Platform::Features::CATALOG` do api (a aba só a mostra). Se o plano do dashboard fixar outro, vale o dele nos dois.
2. **M2 — O `missing` do pré-requisito e os ambientes.** O §1 diz `requires: clinical_record` sem o código que a aba recebe nem os ambientes. Proposta: o §13 registra `clinical_record_disabled` como o `missing` do `dental_care` (o mecanismo genérico do api já traduz `feature:clinical_record` assim, como em `care_lines`, `territory_registry` e `care_regulation`; frase existente desde o 19b, nenhum código novo), e `dental_care` sem `environments` (produção inclusive), sem simulado.
3. **M3 — Catálogo público.** O §15 põe o catálogo público depois do maintenance; este plano não o cobre (escopo dado: interruptor e prova). Proposta: o catálogo ganha plano ou task própria, com `id: "28"` (módulo numerado, conta no rodapé), F-28.1..F-28.8, sem prometer o que a spec deixa fora (ações coletivas, radiografia, ortodontia, implante, BPA do CEO).

## Self-review

- **Cobertura (spec §2 decisão 17, §7, §8 F-28.1; contrato §1, §13, §15):** interruptor pelo mecanismo genérico com o pré-requisito `clinical_record` e a frase do módulo 19 — Task 1 (Review Focus 1, 2 e 5; M2); desligar sem apagar — Task 1 (Review Focus 3); todos os ambientes, sem simulado — Task 1 (Review Focus 4); README — Task 1; suíte, build e prova na 5177 — Task 2; cards — Tasks 0 e 2. Unidade CEO, metas, lista do TSB/ASB e laboratório são da cidade (plano do dashboard): nenhuma tela aqui. Catálogo público fora (M3).
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); a entrada e a `description` do api (Task 0, com a regra escrita para diferença).
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary`, `doneText` (via a frase do status), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem no `origin/main` do maintenance (`1fdda62`); a entrada do api segue a forma real `Entry.new(key:, description:, requires: %w[feature:clinical_record])`, como `care_regulation`.
- **Review Focus:** 1 e 2 — Task 1 ("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19"); 3 — Task 1 ("com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar"); 4 — Task 1 ("produção: aparece e nenhum aviso de simulado"); 5 — Task 1 ("dental_care tem nome em português…; missing desconhecido cru").
- **Proporção:** os blocos de código são testes, o rótulo e o parágrafo do README; nada de tela nova.
