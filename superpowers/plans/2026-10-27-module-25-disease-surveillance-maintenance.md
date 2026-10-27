# Módulo 25 — Vigilância epidemiológica (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 25** na branch `feat/mod-25-disease-surveillance` (worktree `apps/api/.claude/mod25`, servidor de dev na porta **3047**) com `surveillance` em `Platform::Features::CATALOG` (sem `requires`, sem `environments`) e, no `Maintenance::Schema`, a consulta `notifiableDiseases` e a mutation `publishNotifiableDiseaseList` (contrato §3 e §11), com o TOTP do momento e a auditoria de plataforma `notifiable_disease_list.published` declarada em `R18_PLATFORM_EVENT_NAMES`. Contrato §13: api → dashboard → **maintenance** → catálogo público. As Tasks 0–4 não dependem do api rodando (testes com `fetch` falso; o SDL novo entra à mão na Task 2 e é conferido contra o do api na Task 5); a prova da Task 5 precisa do api do 25. A Task 6 (catálogo público) vem **por último**, depois da prova e das entregas de api, dashboard e maintenance.

**Goal:** Lado maintenance de F-25.1: o interruptor `surveillance` aparece na aba Funcionalidades com nome em português ("Vigilância epidemiológica", o mesmo do dashboard) e liga pelo mecanismo genérico **sem** pré-requisito de prontuário; e a tela nova **"Vigilância"** no trilho, com a **lista nacional de doenças e agravos de notificação compulsória** da plataforma: a versão vigente (portaria, vigência, doenças com prazo, CID-10, CIAP-2, janela de repetição, ficha do Sinan e fonte), o histórico de versões e a publicação de **versão nova** a partir da vigente (editar, acrescentar e retirar doenças; portaria e vigência), com o TOTP do momento (step-up), no padrão do catálogo-base do 23. E as F-25.1..F-25.8 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única).

**Architecture:** O liga/desliga não muda (`setCityFeature`, três estados, `missing` vindo do api): entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts`. A lista de doenças segue o molde do catálogo-base de linhas de cuidado do 23 (`CareLineTemplates`): consulta e mutation **próprias** da tela (contra um api sem o 25, só esta tela cai), regras puras em `src/lib/notifiableDiseases.ts` (rótulos, vigente, agendada e histórico, rascunho a partir da vigente, conferência de formato — recusar, nunca cortar —, diferença entre a vigente e o rascunho, variáveis da mutation, frases das recusas), a tela `src/screens/NotifiableDiseases.tsx` e o formulário `src/screens/NotifiableDiseaseListForm.tsx`, reaproveitando os estilos exportados de `LabTemplates.tsx` e o `StepUpCode` dos Mantenedores. O trilho (`Shell.tsx`) ganha o item "Vigilância". O SDL do maintenance ganha os tipos e os campos propostos aqui (M1), conferidos com o do api. Publicar versão nova vale para todas as cidades do ambiente a partir da vigência; nada de dado de cidade passa por aqui.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md` (§1 intenção e antiobjetivos, §2 decisões 1, 11 e as do coordenador sobre a lista e os CIDs, §3 interruptor, §4 "Plataforma", §8 entrega, §9 F-IDs, §10 riscos), `docs/adr/0043.md` e o contrato `docs/superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md` §1, §2 (`<disease>`), §3, §11, §12 e §13. Base: plano do maintenance do 23 (`2026-10-22-module-23-care-lines-maintenance.md`, Tasks 1–6) e o do 24a (Tasks 1–3). As diferenças estão em "Divergências propostas ao contrato", M1–M6.

## Global Constraints

- Interruptor (spec §3, decisão 11; contrato §1 e §11), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `surveillance` **sem** `requires`: liga "ligada e utilizável" em qualquer modo de prontuário (a sugestão na consulta, que exige `clinical_record`, é regra do dashboard e do api, não um pré-requisito do interruptor);
  - existe em todos os ambientes, produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele;
  - desligar não apaga nada (spec §3: "o que foi gravado fica"); a aba não diz o contrário.
- Nome: `surveillance` → "Vigilância epidemiológica" (o mesmo de `src/lib/features.ts` do dashboard, Task 1 do plano do dashboard).
- Lista nacional (spec decisão 1 e decisões do coordenador; contrato §2 e §3):
  - versionada e imutável: publicar é sempre criar a **próxima versão** inteira (portaria, vigência e todas as doenças), nunca editar a vigente; a mutation leva a versão de partida (`expectedVersion`, M2) e o TOTP do momento (`code`, step-up);
  - cada doença: código, nome, prazo `immediate` (24 h) | `weekly` (7 dias), CID-10 e CIAP-2 que geram a sugestão, janela de repetição em dias (padrão 30), ficha do Sinan correspondente (opcional), uma nota (opcional) e as **fontes** da linha (`sources`: ao menos a do prazo e a dos códigos — a portaria ou a ficha do Sinan; as outras fontes que a carga do api trouxer voltam como vieram);
  - a tela diz, antes do botão, que a versão nova vale para todas as cidades a partir da vigência e que as notificações já feitas ficam com a versão em que nasceram;
  - **as doenças locais são da cidade** (dashboard, plano do dashboard Task 12); o maintenance não vê nem edita doença local.
- O maintenance não vê nada de cidade: nenhum paciente, notificação, ficha, contagem, bairro ou unidade passa por aqui.
- Recusar em vez de cortar (lições dos contratos do 21a §15 e 21b §14): nenhum `maxLength` nos campos novos; portaria 3–200 caracteres; nome da doença 3–120; código `^[a-z][a-z0-9_]{1,47}$` sem o prefixo `local_` (o das doenças da cidade; plano do api, `PLATFORM_CODE`); CID-10 `A00`/`A000`/`A00X` (sem ponto; o ponto digitado é tirado, como no 23); CIAP-2 `A00`; janela de repetição inteira de 1 a 36500 (vírgula ou ponto recusados: é inteiro; HIV e hepatites usam 36500 na carga do api); ficha do Sinan até 60; nota até 300; fonte do prazo e dos códigos 3–300 cada; ao menos uma doença; códigos únicos na lista; vigência em `AAAA-MM-DD`, não antes de hoje nem antes da vigência da versão vigente (M4) — tudo conferido antes de enviar e de novo no api.
- Valor desconhecido (api mais novo: prazo, código de `missing` ou de erro) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): componentes existentes (`Panel`, `DataTable` — que já embrulha em `<TableScroll>` —, `Tag`, `Button`, `Field`, `EmptyState`, `ErrorState`, `StepUpCodeError`/`STEP_UP_CODE_HELP_TEXT`), sem redesign nem componente comum novo; cores só por papel do tema; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte. Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); o que só foi provado com dado de exemplo fica `validating` com nota; nada promete envio automático ao Sinan nem apresenta o sinal como surto; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-25.1..F-25.8 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-25.1: Done do F-25.1 quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod25`, `C=$R/.worktrees/catalog-mod25`, `A=$R/apps/api/.claude/mod25`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-25-disease-surveillance`; catálogo em `$C`, branch `content/module-25-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task.
- maintenance — testes: `bin/test maintenance -w maintenance-mod25 <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; codegen: `npm run codegen` (gera `src/gql/*` a partir de `schema.graphql` e dos documentos `graphql(...)`); CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (os testes chamam `cleanup`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- catálogo — testes: `cd $C && npx vitest run` (o catálogo não está no `bin/test`; é estático, sem banco).
- O api do 25 roda em dev na **3047** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177` (o proxy de dev injeta o Origin constante `http://maintenance.localhost:5177`, que o api exige); a prova sobe o Vite do worktree **na 5177** com o serviço `maintenance` do compose parado; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login e digita o TOTP. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade sem prontuário.** Ligar `surveillance` numa cidade em modo `off` ou `integrated` dá "ligada e utilizável", sem pedir `clinical_record` (decisão 11); o nome em português aparece. Teste: Task 1, "liga utilizável sem o prontuário; desliga com dois cliques; nada fala em apagar".
2. **Duas pessoas publicam ao mesmo tempo.** A segunda parte da v3, mas a v4 já saiu. Espera-se: a recusa (`expectedVersion`) vira "outra pessoa publicou uma versão enquanto você editava — recarregue", o rascunho fica, nada é reenviado sozinho. Teste: Task 4, "versão de partida desatualizada: frase, o rascunho fica".
3. **Código errado ou TOTP velho.** Espera-se: a recusa do `code` aparece no campo do código com o aviso de bloqueio ("Tentativas erradas contam para o bloqueio da conta."), o código é limpo e o rascunho fica. Teste: Task 4, "TOTP recusado: no campo, com o aviso de bloqueio; o rascunho fica".
4. **CID digitado com ponto, CIAP trocado por CID, janela "30,5" ou código repetido.** Espera-se: "A90.0" vira `A900`; "A9" é recusado com a doença e o campo; "30,5" é recusado (inteiro); dois "A90" na lista são recusados — antes de enviar, com o texto mantido. Testes: Task 2, "doença: formato, recusar sem cortar" e "lista: códigos únicos, vigência e portaria"; Task 4, "CID inválido é recusado e fica no campo; nada é enviado".
5. **Api sem o 25.** Contra um api sem `notifiableDiseases`, só esta tela cai, com a frase "o api do módulo 25 precisa subir antes do maintenance"; o resto do maintenance segue. Teste: Task 3, "api sem o 25: só esta tela cai, com a frase".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-25.1..F-25.8, conferência do main e do api do 25, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome do interruptor e o fluxo genérico da aba fixado para ele | 1 |
| `schema.graphql`, `src/lib/notifiableDiseases.ts`, `src/lib/notifiableDiseases.test.ts` | SDL da lista (M1) e as regras da tela | 2 |
| `src/screens/NotifiableDiseases.tsx`, `src/screens/NotifiableDiseases.fixtures.tsx`, `src/screens/NotifiableDiseases.test.tsx`, `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx`, `src/gql/*` | a tela no trilho: vigente, agendada e histórico | 3 |
| `src/screens/NotifiableDiseaseListForm.tsx`, `src/screens/NotifiableDiseaseListForm.test.tsx`, `src/screens/NotifiableDiseases.tsx`, `src/gql/*` | editar e publicar versão nova com step-up | 4 |
| — | suíte, schema igual ao do api, codegen:check, build, revisão, prova no navegador | 5 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx` | módulo 25 com F-25.1..F-25.8, rodapé e ciclo 2 | 6 |

**Estratégia de teste:** funções puras (Task 1: `featureLabel`, `missingText`; Task 2: rótulos, vigente/agendada/histórico, rascunho, conferência de formato com tabela de casos — o que recusa e o que **não** recusa —, diferença, variáveis da mutation, frases das recusas); o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation (Task 1); a tela e o formulário com `fetch` falso por operação GraphQL, conferindo as variáveis enviadas, a releitura depois de publicar, o step-up e as recusas (Tasks 3–4); suíte inteira com as guardas de tema, o SDL igual ao do api, `codegen:check`, build e a prova no navegador contra o api do 25 (Task 5). Catálogo: `validateCatalog`, o teste do módulo 25, o rodapé e o ciclo (Task 6).

---

### Task 0: Cards, conferência do main e do api do 25, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`eb0eeb0` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-25.1..F-25.8 em In Progress (ou além); worktree `$M` na branch `feat/mod-25-disease-surveillance`; a base anotada; o SDL do api anotado.

- [ ] **Step 1: Mova os cards F-25.1..F-25.8 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-25\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified ficam como estão. Para cada um antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito itens em In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main do maintenance e o api do 25**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod25
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n '  territory_registry: "Cadastro individual e domiciliar"\|surveillance'
$G -C $R/apps/maintenance show origin/main:src/screens/LabTemplates.tsx | grep -n '^export const muted\|^export const row\|^export const grid\|^export const labelStyle\|^export const inputStyle'
$G -C $R/apps/maintenance show origin/main:src/components/StepUpCode.tsx | grep -n '^export const STEP_UP_CODE_HELP_TEXT\|^export function StepUpCodeError'
$G -C $R/apps/maintenance show origin/main:src/screens/Shell.tsx | grep -n 'export type Screen = "cities" | "medications" | "lab" | "careLines" | "maintainers" | "tokens" | "audit";\|  { key: "careLines", label: "Linhas de cuidado" },\|          {screen === "careLines" && <CareLineTemplates />}'
$G -C $R/apps/maintenance show origin/main:src/screens/Shell.rail.test.tsx | grep -n '"Cidades", "Medicamentos", "Laboratório", "Linhas de cuidado", "Mantenedores", "Tokens", "Auditoria"'
$G -C $R/apps/maintenance show origin/main:schema.graphql | grep -n '^  careLineTemplates(key: String): \[CareLineTemplate!\]!\|^  upsertCareLineTemplate(\|^scalar ISO8601Date \|^type UserError {'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Cadastro individual e domiciliar (módulo 24a).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n -E 'key: "surveillance"' -A4 $A/app/services/platform/features.rb
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod25 api bin/rails runner 'puts Maintenance::Schema.to_definition' 2>/dev/null | grep -n -A16 'type NotifiableDisease\|notifiableDiseases\|publishNotifiableDiseaseList\|input NotifiableDisease'
```

Expected:
- em `features.ts`: a âncora do 24a e **nenhuma** linha com `surveillance`; em `LabTemplates.tsx`, `StepUpCode.tsx`, `Shell.tsx`, `Shell.rail.test.tsx`, `schema.graphql` e `README.md`: as âncoras. Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 25: a entrada `surveillance` **sem** `requires` e **sem** `environments` (outro pré-requisito → **pare e reporte**: diverge da spec §3/decisão 11); e o SDL com `NotifiableDiseaseVersion`, `NotifiableDisease`, `NotifiableDiseaseInput`, `notifiableDiseases` e `publishNotifiableDiseaseList` (plano do api, Task 4). **Compare** com o SDL da Task 2 (M1): diferença de nome, de tipo ou de argumento vira ajuste no SDL, nas regras e nos documentos das Tasks 2–4 **antes** de começá-las, e a regra do api para o código, a janela e a vigência (M3, M4) passa a valer aqui. **Sem `expectedVersion`** no api (M2): siga a instrução do fim do cabeçalho da Task 2. Se o api ainda não tiver o schema, as Tasks 1–4 seguem com o SDL da Task 2 e a Task 5 confere.

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod25 -b feat/mod-25-disease-surveillance origin/main
cd $R/.worktrees/maintenance-mod25 && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod25
```

Expected: suíte verde e `base-ok`. **Anote** a base (arquivos e testes): a Task 5 espera **+3 arquivos de teste** (`notifiableDiseases`, `NotifiableDiseases`, `NotifiableDiseaseListForm`) e casos novos nos existentes (`features`, `FeaturesTab`, `Shell`, `Shell.rail`).

---

### Task 1: O interruptor `surveillance` — nome e o fluxo genérico da aba

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo. Esta task dá o nome e fixa esse comportamento para a chave nova.

**Interfaces:**
- Consumes: `featureLabel`, `missingText` (existentes em `features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("surveillance")` → "Vigilância epidemiológica".

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 25", () => {
  it("surveillance tem nome em português, o mesmo do dashboard; missing desconhecido cru", () => {
    expect(featureLabel("surveillance")).toBe("Vigilância epidemiológica");
    expect(missingText("notifiable_list_missing")).toBe("notifiable_list_missing");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx`:

```tsx
describe("FeaturesTab — vigilância epidemiológica (módulo 25)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const SV_OFF: Feature = {
    key: "surveillance", description: "Vigilância epidemiológica: notificação compulsória, fila, negativa semanal e painel",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const SV_ON: Feature = { ...SV_OFF, enabled: true, usable: true, changedAt: "2026-10-27T13:00:00Z", changedBy: "dev@local" };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ SV_OFF ], { recordMode: "off" })));
    renderTab();
    expect(await screen.findByText("surveillance")).not.toBeNull();
    expect(screen.getByText("Vigilância epidemiológica")).not.toBeNull();
    expect(screen.getByText(SV_OFF.description)).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("liga utilizável sem o prontuário; desliga com dois cliques; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ SV_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? SV_ON : { ...SV_OFF, changedAt: "2026-10-27T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "off" }));
    });
    renderTab();
    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar surveillance" }));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "surveillance", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe("surveillance ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar surveillance" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect((await screen.findByRole("status")).textContent).toBe("surveillance desligada.");
    expect(document.body.textContent).not.toMatch(/apaga|exclu|prontuário/i);
  });

  it("produção: aparece e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ SV_ON ], { recordMode: "record" })));
    renderTab();
    expect(await screen.findByText("surveillance")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod25 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("surveillance")` é `null` e a aba não acha "Vigilância epidemiológica". (O fluxo de dois cliques e o código cru já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, depois de `territory_registry: "Cadastro individual e domiciliar"`, a vírgula e:

```ts
  // Módulo 25 (ADR 0043): sem pré-requisito (nem o modo prontuário); a
  // sugestão na consulta exige clinical_record no dashboard e no api, não aqui.
  // Todos os ambientes, sem simulado.
  surveillance: "Vigilância epidemiológica"
```

Em `README.md`, logo depois do parágrafo **Cadastro individual e domiciliar (módulo 24a).** (antes de **Documentos clínicos (módulo 19c).**):

```markdown
**Vigilância epidemiológica (módulo 25).** O interruptor `surveillance`
("Vigilância epidemiológica") liga, no dashboard, a fila da vigilância
municipal, a notificação manual, a notificação negativa semanal, os relatórios
e o painel por bairro e semana epidemiológica; com `clinical_record` também
ligado, a consulta sugere notificar quando um CID ou CIAP-2 da lista é
registrado. Não exige o prontuário. Existe em todos os ambientes e não tem
simulado. Desligar esconde as telas e as sugestões; o que já foi gravado fica.
A tela **Vigilância** do trilho mantém a lista nacional de doenças e agravos de
notificação compulsória (Portaria GM/MS 11.211/2026): cada publicação é uma
versão nova, com portaria e vigência, pedida com o código do autenticador, e
vale para todas as cidades do ambiente a partir da vigência. As doenças locais
são da cidade (dashboard). Nenhum dado de cidadão passa por aqui. Ordem de
deploy: api → dashboard → maintenance.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod25 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais). Se as frases "ligada e utilizável", "<chave> ligada e utilizável." e "<chave> desligada." mudaram no código, use as reais nos testes.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the disease surveillance switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: SDL da lista e as regras da tela (`src/lib/notifiableDiseases.ts`)

**Files:**
- Modify: `schema.graphql`
- Create: `src/lib/notifiableDiseases.ts`
- Test: `src/lib/notifiableDiseases.test.ts`

**Interfaces:**
- Consumes: nada de tasks anteriores.
- Produces: em `schema.graphql`, o SDL abaixo (o do plano do api, Task 4, mais `expectedVersion` — M1, M2); em `notifiableDiseases.ts`: `DiseaseView`, `ListView`, `DiseaseDraft`, `ListDraft`, `PublishVariables`; `DEADLINES`, `deadlineText(code)`, `LIMITS`; `currentList(all): ListView | null`, `scheduledList(all, today): ListView | null`, `historyOf(all): ListView[]`, `listState(l, all, today): "vigente" | "agendada" | "anterior"`; `toDraft(base: ListView | null): ListDraft`, `emptyDisease(): DiseaseDraft`, `codesOf(text): string[]`; `diseaseProblem(d): string | null`, `draftProblem(d, current, today): string | null`; `draftToVariables(d, code, expectedVersion): PublishVariables`; `listDiff(base, d): { added: string[]; removed: string[]; changed: string[] }`, `diffText(diff): string`; `publishErrorText(errors): string`, `STALE_LIST_TEXT`, `NOTIFIABLE_QUERY_KEY = [ "platform", "notifiableDiseases" ]`.

SDL (no `schema.graphql`, em ordem alfabética como o resto do arquivo; o `codegen` usa este arquivo). Nomes do plano do api (Task 4): versão `NotifiableDiseaseVersion` com `effectiveFrom` e `publishedBy`; doença com `notes` e `sources` (JSON: a fonte de cada campo, ao menos `deadline` e `cid10` — Task 0 do api):

```graphql
"""
Versão da lista nacional de doenças e agravos de notificação compulsória. Imutável; vale para todas as cidades a partir de effectiveFrom.
"""
type NotifiableDiseaseVersion {
  createdAt: ISO8601DateTime!
  """
  A versão em vigor hoje (maior versão com effectiveFrom até hoje)
  """
  current: Boolean!
  diseases: [NotifiableDisease!]!
  effectiveFrom: ISO8601Date!
  """
  Portaria de origem (ex.: Portaria GM/MS nº 11.211/2026)
  """
  ordinance: String!
  """
  E-mail de quem publicou (nulo na carga inicial)
  """
  publishedBy: String
  version: Int!
}

type NotifiableDisease {
  ciap2: [String!]!
  cid10: [String!]!
  """
  Código da doença na plataforma (minúsculas; nunca local_)
  """
  code: String!
  """
  immediate (24 h) | weekly (7 dias)
  """
  deadline: String!
  name: String!
  notes: String
  repeatAfterDays: Int!
  sinanForm: String
  """
  Fonte de cada campo (ao menos deadline e cid10)
  """
  sources: JSON!
}

input NotifiableDiseaseInput {
  ciap2: [String!]!
  cid10: [String!]!
  code: String!
  deadline: String!
  name: String!
  notes: String
  repeatAfterDays: Int!
  sinanForm: String
  sources: JSON!
}

"""
Autogenerated return type of PublishNotifiableDiseaseList.
"""
type PublishNotifiableDiseaseListPayload {
  errors: [UserError!]!
  ok: Boolean!
  version: Int
}
```

Em `type Query`: `notifiableDiseases: [NotifiableDiseaseVersion!]!`. Em `type Mutation`, com a descrição "Publica uma versão nova da lista nacional de doenças de notificação compulsória. Exige o TOTP do momento.":

```graphql
  publishNotifiableDiseaseList(
    """
    TOTP do momento
    """
    code: String!
    diseases: [NotifiableDiseaseInput!]!
    effectiveFrom: ISO8601Date!
    """
    A versão vigente de onde o rascunho partiu (nula só para a primeira). Divergente, a publicação é recusada.
    """
    expectedVersion: Int
    ordinance: String!
  ): PublishNotifiableDiseaseListPayload
```

Se a Task 0 achou o api **sem** `expectedVersion` (M2 recusada pelo coordenador), tire o argumento do SDL, do documento da Task 4 e de `draftToVariables`, e tire o teste "versão de partida desatualizada" da Task 4 (anote que o Review Focus 2 ficou descoberto).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/notifiableDiseases.test.ts
import { describe, expect, it } from "vitest";
import {
  STALE_LIST_TEXT, codesOf, currentList, deadlineText, diffText, diseaseProblem, draftProblem, draftToVariables, emptyDisease, historyOf, listDiff, listState,
  publishErrorText, scheduledList, toDraft, type DiseaseView, type ListView
} from "./notifiableDiseases";

const DENGUE: DiseaseView = { code: "dengue", name: "Dengue", deadline: "weekly", cid10: [ "A90", "A91" ], ciap2: [], repeatAfterDays: 30,
  sinanForm: "dengue", notes: "óbito por dengue: imediata, notificar à mão",
  sources: { deadline: "Portaria GM/MS 11.211/2026, anexo 1", cid10: "ficha de investigação de dengue", repeat_after_days: "proposta do plano 25" } };
const MEASLES: DiseaseView = { code: "sarampo", name: "Sarampo", deadline: "immediate", cid10: [ "B059" ], ciap2: [ "A71" ], repeatAfterDays: 30,
  sinanForm: null, notes: null, sources: { deadline: "Portaria GM/MS 11.211/2026, anexo 1", cid10: "ficha de investigação de sarampo" } };
const V1: ListView = { version: 1, ordinance: "Portaria GM/MS nº 11.211/2026", effectiveFrom: "2026-03-01", current: false,
  createdAt: "2026-10-27T12:00:00Z", publishedBy: null, diseases: [ DENGUE ] };
const V2: ListView = { ...V1, version: 2, effectiveFrom: "2026-10-01", current: true, publishedBy: "mantenedor@rotasaude.app", diseases: [ DENGUE, MEASLES ] };
const V3: ListView = { ...V2, version: 3, effectiveFrom: "2026-12-01", current: false };
const TODAY = "2026-10-27";

describe("rótulos e versões", () => {
  it("prazo com rótulo; desconhecido cru", () => {
    expect(deadlineText("immediate")).toBe("imediata (24 h)");
    expect(deadlineText("weekly")).toBe("semanal (7 dias)");
    expect(deadlineText("monthly")).toBe("monthly");
  });
  it("vigente, agendada e histórico do mais novo para o mais antigo", () => {
    expect(currentList([ V1, V3, V2 ])?.version).toBe(2);
    expect(scheduledList([ V1, V3, V2 ], TODAY)?.version).toBe(3);
    expect(scheduledList([ V1, V2 ], TODAY)).toBeNull();
    expect(historyOf([ V1, V3, V2 ]).map((l) => l.version)).toEqual([ 3, 2, 1 ]);
    expect([ V1, V2, V3 ].map((l) => listState(l, [ V1, V2, V3 ], TODAY))).toEqual([ "anterior", "vigente", "agendada" ]);
  });
});

describe("rascunho", () => {
  it("parte da vigente, sem vigência; códigos em texto; as fontes de prazo e códigos editáveis, as outras guardadas", () => {
    const d = toDraft(V2);
    expect(d.ordinance).toBe("Portaria GM/MS nº 11.211/2026");
    expect(d.effectiveFrom).toBe("");
    expect(d.diseases[0]).toEqual({ code: "dengue", name: "Dengue", deadline: "weekly", cid10: "A90, A91", ciap2: "", repeatAfterDays: "30",
      sinanForm: "dengue", notes: "óbito por dengue: imediata, notificar à mão", deadlineSource: "Portaria GM/MS 11.211/2026, anexo 1",
      cid10Source: "ficha de investigação de dengue", otherSources: { repeat_after_days: "proposta do plano 25" } });
    expect(toDraft(null).diseases).toEqual([]);
    expect(emptyDisease()).toEqual({ code: "", name: "", deadline: "weekly", cid10: "", ciap2: "", repeatAfterDays: "30", sinanForm: "", notes: "",
      deadlineSource: "", cid10Source: "", otherSources: {} });
  });
  it("códigos: maiúsculas, sem ponto, sem repetir", () => {
    expect(codesOf(" a90.0, A91;a90.0  b05 ")).toEqual([ "A900", "A91", "B05" ]);
  });
});

describe("formato: recusar, nunca cortar", () => {
  const ok = toDraft(V2).diseases[0];
  it("doença: formato, recusar sem cortar", () => {
    expect(diseaseProblem(ok)).toBeNull();
    expect(diseaseProblem({ ...ok, code: "Dengue" })).toBe("código de 2 a 48 caracteres: minúscula no início, depois minúsculas, números ou _");
    expect(diseaseProblem({ ...ok, code: "local_dengue" })).toBe("código local_ é das doenças da cidade, não da lista nacional");
    expect(diseaseProblem({ ...ok, name: "De" })).toBe("nome de 3 a 120 caracteres");
    expect(diseaseProblem({ ...ok, deadline: "" })).toBe("escolha o prazo");
    expect(diseaseProblem({ ...ok, cid10: "A9" })).toBe("CID-10 inválido: A9");
    expect(diseaseProblem({ ...ok, cid10: "" })).toBeNull();
    expect(diseaseProblem({ ...ok, ciap2: "A770" })).toBe("CIAP-2 inválido: A770");
    expect(diseaseProblem({ ...ok, repeatAfterDays: "30,5" })).toBe("janela de repetição: inteiro de 1 a 36500 dias");
    expect(diseaseProblem({ ...ok, repeatAfterDays: "0" })).toBe("janela de repetição: inteiro de 1 a 36500 dias");
    expect(diseaseProblem({ ...ok, repeatAfterDays: "36500" })).toBeNull();
    expect(diseaseProblem({ ...ok, sinanForm: "x".repeat(61) })).toBe("ficha do Sinan com mais de 60 caracteres");
    expect(diseaseProblem({ ...ok, notes: "x".repeat(301) })).toBe("nota com mais de 300 caracteres");
    expect(diseaseProblem({ ...ok, deadlineSource: "" })).toBe("informe a fonte do prazo");
    expect(diseaseProblem({ ...ok, cid10Source: "" })).toBe("informe a fonte dos códigos");
    expect(diseaseProblem({ ...ok, cid10Source: "pendente" })).toBe("informe a fonte dos códigos");
    expect(diseaseProblem({ ...ok, name: "Ç".repeat(120) })).toBeNull();
  });
  it("lista: códigos únicos, vigência e portaria", () => {
    const d = { ...toDraft(V2), effectiveFrom: "2026-11-01" };
    expect(draftProblem(d, V2, TODAY)).toBeNull();
    expect(draftProblem({ ...d, ordinance: "P" }, V2, TODAY)).toBe("portaria de 3 a 200 caracteres");
    expect(draftProblem({ ...d, effectiveFrom: "" }, V2, TODAY)).toBe("informe o início da vigência");
    expect(draftProblem({ ...d, effectiveFrom: "2026-10-26" }, V2, TODAY)).toBe("a vigência não pode começar antes de hoje");
    expect(draftProblem({ ...d, diseases: [] }, V2, TODAY)).toBe("a lista precisa de ao menos uma doença");
    expect(draftProblem({ ...d, diseases: [ ...d.diseases, { ...d.diseases[0], name: "Dengue de novo" } ] }, V2, TODAY)).toBe("código repetido na lista: dengue");
    expect(draftProblem({ ...d, diseases: [ d.diseases[0], { ...d.diseases[1], cid10: "B0" } ] }, V2, TODAY)).toBe("doença 2 (sarampo): CID-10 inválido: B0");
  });
});

describe("envio e diferença", () => {
  it("variáveis: códigos normalizados, números, nulos, fontes juntas; o TOTP e a versão de partida", () => {
    const d = { ...toDraft(V2), effectiveFrom: "2026-11-01" };
    d.diseases[1] = { ...d.diseases[1], cid10: "b05.9", sinanForm: "", notes: "" };
    expect(draftToVariables(d, "123456", 2)).toEqual({
      code: "123456", expectedVersion: 2, ordinance: "Portaria GM/MS nº 11.211/2026", effectiveFrom: "2026-11-01",
      diseases: [
        { code: "dengue", name: "Dengue", deadline: "weekly", cid10: [ "A90", "A91" ], ciap2: [], repeatAfterDays: 30, sinanForm: "dengue",
          notes: "óbito por dengue: imediata, notificar à mão",
          sources: { repeat_after_days: "proposta do plano 25", deadline: "Portaria GM/MS 11.211/2026, anexo 1", cid10: "ficha de investigação de dengue" } },
        { code: "sarampo", name: "Sarampo", deadline: "immediate", cid10: [ "B059" ], ciap2: [ "A71" ], repeatAfterDays: 30, sinanForm: null, notes: null,
          sources: { deadline: "Portaria GM/MS 11.211/2026, anexo 1", cid10: "ficha de investigação de sarampo" } } ] });
  });
  it("diferença entre a vigente e o rascunho", () => {
    const d = toDraft(V2);
    d.diseases = [ { ...d.diseases[0], deadline: "immediate" },
      { ...emptyDisease(), code: "mpox", name: "Mpox", cid10: "B04", deadlineSource: "Portaria", cid10Source: "ficha de mpox" } ];
    expect(listDiff(V2, d)).toEqual({ added: [ "mpox" ], removed: [ "sarampo" ], changed: [ "dengue" ] });
    expect(diffText(listDiff(V2, d))).toBe("1 nova, 1 retirada, 1 alterada");
    expect(diffText({ added: [ "a", "b" ], removed: [], changed: [] })).toBe("2 novas, 0 retiradas, 0 alteradas");
    expect(diffText({ added: [], removed: [], changed: [] })).toBe("nenhuma mudança nas doenças");
  });
  it("recusas do api viram frase; versão de partida velha", () => {
    expect(publishErrorText([ { path: "expectedVersion", message: "stale" } ])).toBe(STALE_LIST_TEXT);
    expect(publishErrorText([ { path: "diseases.1.cid10.0", message: "invalid" } ])).toBe("doença 2: CID-10 — invalid");
    expect(publishErrorText([ { path: "effectiveFrom", message: "before_current" } ])).toBe("vigência — before_current");
    expect(publishErrorText([ { path: null, message: "algo novo" } ])).toBe("algo novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod25 src/lib/notifiableDiseases.test.ts`
Expected: FAIL — o módulo não existe.

- [ ] **Step 3: Implemente o SDL e `src/lib/notifiableDiseases.ts`**

Cabeçalho: "Regras da lista nacional de doenças e agravos de notificação compulsória (módulo 25, F-25.1; spec §2 decisão 1; contrato §2, §3 e §11; plano do api, Tasks 0, 3 e 4). Versionada e imutável: publicar é criar a versão seguinte inteira. Quem valida é o api; aqui se recusa o que ele recusaria por formato — nunca cortar. Valor desconhecido aparece cru; fonte de outro campo que a tela não edita volta como veio."

Tipos: `DiseaseView = { code; name; deadline; cid10: string[]; ciap2: string[]; repeatAfterDays: number; sinanForm: string | null; notes: string | null; sources: Record<string, string> }`; `ListView = { version; ordinance; effectiveFrom; current: boolean; createdAt; publishedBy: string | null; diseases: DiseaseView[] }`; `DiseaseDraft` = os campos de texto do teste (`deadlineSource` = `sources.deadline`, `cid10Source` = `sources.cid10`, `otherSources` = o resto de `sources`, intocado); `ListDraft = { ordinance; effectiveFrom; diseases: DiseaseDraft[] }`.

Valores fixados (os do plano do api, "Valores fixados" 2): `DEADLINES = [ "immediate", "weekly" ]`; `LIMITS = { ordinance: [3, 200], name: [3, 120], sinanForm: 60, notes: 300, source: [3, 300], repeat: [1, 36500] }`; contagem por `[...texto].length`; `PLATFORM_CODE = /^[a-z][a-z0-9_]{1,47}$/` sem o prefixo `local_` (frase própria quando começa com `local_`), `CID_RE = /^[A-Z]\d{2}[0-9X]?$/`, `CIAP_RE = /^[A-Z]\d{2}$/`, janela `^\d+$` e faixa; a fonte "pendente" (marca da carga do api) é tratada como vazia; `STALE_LIST_TEXT = "outra pessoa publicou uma versão enquanto você editava — recarregue"`. `currentList` = a com `current`; `scheduledList` = a de maior versão com `effectiveFrom > today` e versão maior que a vigente (ou `null`); `listState` por essas duas. `diseaseProblem` confere, nessa ordem: código, nome, prazo, cada CID-10, cada CIAP-2, janela, ficha do Sinan, nota, fonte do prazo, fonte dos códigos. `draftProblem` confere portaria, vigência (vazia; formato; antes de hoje; antes da vigência da vigente → "a vigência não pode começar antes da versão vigente"), lista vazia, código repetido (comparação exata: o formato já é minúsculo) e, por doença, "doença N (código): <frase>". `draftToVariables`: `codesOf` nos códigos, `Number` na janela, `sinanForm` e `notes` vazios → `null`, textos com `trim`, `sources = { ...otherSources, deadline, cid10 }`. `listDiff` por código: `changed` quando nome, prazo, códigos (como conjunto), janela, ficha, nota ou fontes mudam. `diffText`: "N nova(s), N retirada(s), N alterada(s)", no singular só com 1; tudo zero → "nenhuma mudança nas doenças". `publishErrorText`: `expectedVersion` → `STALE_LIST_TEXT`; `diseases.N.campo[.i]` (camelCase, como o api devolve) → "doença N+1: <rótulo> — <mensagem>" (rótulos `code` "código", `name` "nome", `deadline` "prazo", `cid10` "CID-10", `ciap2` "CIAP-2", `repeatAfterDays` "janela de repetição", `sinanForm` "ficha do Sinan", `notes` "nota", `sources` "fonte"); `ordinance` "portaria", `effectiveFrom` "vigência"; sem caminho, a mensagem crua; várias, unidas por "; ". O `code` (TOTP) não passa por aqui: o formulário mostra no campo.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod25 src/lib/notifiableDiseases.test.ts && cd $M && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add schema.graphql src/lib/notifiableDiseases.ts src/lib/notifiableDiseases.test.ts
$G commit -m "feat: add notifiable disease list rules and schema

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: A tela no trilho — vigente, agendada e histórico

**Files:**
- Create: `src/screens/NotifiableDiseases.tsx`, `src/screens/NotifiableDiseases.fixtures.tsx`
- Modify: `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx`, `src/gql/*` (gerado)
- Test: `src/screens/NotifiableDiseases.test.tsx`

**Interfaces:**
- Consumes: `currentList`, `scheduledList`, `historyOf`, `listState`, `deadlineText`, `NOTIFIABLE_QUERY_KEY` (Task 2); `gql` (`src/lib/api.ts`); `GraphQLRefusal` (`src/lib/errors.ts`); `fmtWhen` (`src/lib/analytics.ts`); `muted`, `row` (`LabTemplates.tsx`).
- Produces: `NotifiableDiseases()`; o documento `query NotifiableDiseases { notifiableDiseases { version ordinance effectiveFrom current createdAt publishedBy diseases { code name deadline cid10 ciap2 repeatAfterDays sinanForm notes sources } } }`; `Screen` ganha `"surveillance"` e o `NAV` ganha `{ key: "surveillance", label: "Vigilância" }` depois de "Linhas de cuidado" (ícone de escudo com cruz: `svg("M12 3l8 3v6c0 5-3.5 8-8 9-4.5-1-8-4-8-9V6l8-3zM12 9v6M9 12h6")`); em `NotifiableDiseases.fixtures.tsx`: `reply`, `DENGUE`, `MEASLES`, `V1`, `V2`, `V3` (as formas do teste da Task 2, com `__typename` ausente) e `renderScreen()`.

Comportamento: `section` com `aria-label` = `TITLE` = "Lista nacional de doenças e agravos de notificação compulsória (todas as cidades do ambiente)"; `INTRO` = "Publicar uma versão nova vale para todas as cidades a partir da vigência: as consultas passam a sugerir pelos códigos novos. As notificações já feitas ficam com a versão em que nasceram. As doenças locais são de cada cidade, no painel dela." Resumo da vigente: "v2 · Portaria GM/MS nº 11.211/2026 · vigente desde 01/10/2026 · 2 doenças · publicada por mantenedor@rotasaude.app" ("carga inicial" quando `publishedBy` é nulo); a agendada, se houver, numa linha com `Tag tone="warn"` "agendada para 01/12/2026". `DataTable` da vigente (colunas Doença, Prazo, CID-10, CIAP-2, Repetição "30 dias", Ficha do Sinan "—" quando nula, Fonte dos códigos = `sources.cid10`; a nota, quando houver, embaixo do nome) — a imediata com `Tag` do prazo. "versões" mostra a região `Versões da lista` com um `li` por versão (`data-version`), do mais novo ao mais antigo, com o estado (`vigente`/`agendada`/`anterior`), a portaria, a vigência, o número de doenças e quem publicou. Sem nenhuma versão: `EmptyState` "nenhuma versão da lista" e o botão "primeira versão" (Task 4). Erro de api sem o 25 (`GraphQLRefusal` `undefinedField`): "esta API ainda não tem a lista de doenças de notificação — o api do módulo 25 precisa subir antes do maintenance".

- [ ] **Step 1: Write the failing test**

```tsx
// src/screens/NotifiableDiseases.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { V1, V2, V3, renderScreen, reply } from "./NotifiableDiseases.fixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

describe("NotifiableDiseases", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  const listing = (list: unknown[]) => fetchMock.mockImplementation(() => Promise.resolve(reply(200, { data: { notifiableDiseases: list } })));

  it("a vigente com portaria, vigência, doenças e a agendada", async () => {
    listing([ V1, V2, V3 ]);
    renderScreen();
    expect(await screen.findByText("v2 · Portaria GM/MS nº 11.211/2026 · vigente desde 01/10/2026 · 2 doenças · publicada por mantenedor@rotasaude.app"))
      .not.toBeNull();
    expect(screen.getByText("agendada para 01/12/2026")).not.toBeNull();
    const sarampo = screen.getByRole("row", { name: /Sarampo/ });
    expect(within(sarampo).getByText("imediata (24 h)")).not.toBeNull();
    expect(within(sarampo).getByText("B059")).not.toBeNull();
    expect(within(screen.getByRole("row", { name: /Dengue/ })).getByText("A90, A91")).not.toBeNull();
    expect(screen.getByText(/As notificações já feitas ficam com a versão em que nasceram/)).not.toBeNull();
  });

  it("versões: do mais novo ao mais antigo, com o estado", async () => {
    const user = userEvent.setup();
    listing([ V1, V2, V3 ]);
    renderScreen();
    await user.click(await screen.findByRole("button", { name: "versões" }));
    const history = screen.getByRole("region", { name: "Versões da lista" });
    const items = within(history).getAllByRole("listitem").filter((li) => li.getAttribute("data-version"));
    expect(items.map((li) => li.getAttribute("data-version"))).toEqual([ "3", "2", "1" ]);
    expect(within(items[0]).getByText("agendada")).not.toBeNull();
    expect(within(items[1]).getByText("vigente")).not.toBeNull();
    expect(within(items[2]).getByText(/carga inicial/)).not.toBeNull();
  });

  it("sem nenhuma versão: estado vazio e primeira versão", async () => {
    listing([]);
    renderScreen();
    expect(await screen.findByText("nenhuma versão da lista")).not.toBeNull();
    expect(screen.getByRole("button", { name: "primeira versão" })).not.toBeNull();
  });

  it("api sem o 25: só esta tela cai, com a frase", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      errors: [ { message: "Field 'notifiableDiseases' doesn't exist on type 'Query'", extensions: { code: "undefinedField" } } ] })));
    renderScreen();
    expect(await screen.findByText(/o api do módulo 25 precisa subir antes do maintenance/)).not.toBeNull();
  });
});
```

Em `src/screens/Shell.rail.test.tsx`, nas duas listas de rótulos, `"Vigilância"` depois de `"Linhas de cuidado"`. Em `src/screens/Shell.test.tsx`, no fim do `describe` do trilho:

```tsx
  it("Vigilância abre a lista de doenças de notificação", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const path = String(url);
      if (path === "/session") return Promise.resolve(reply(200, { id: "m1", email_address: "mantenedor@rotasaude.app", expires_at: "2026-01-01T01:00:00Z" }));
      if (path === "/graphql") {
        const body = JSON.parse((init as RequestInit).body as string);
        if (String(body.query).includes("query NotifiableDiseases")) return Promise.resolve(reply(200, { data: { notifiableDiseases: [] } }));
        return Promise.resolve(reply(200, { data: { cities: [] } }));
      }
      return Promise.resolve(reply(204, undefined));
    });
    renderShell();
    await user.click(await screen.findByRole("button", { name: "Vigilância" }));
    expect(screen.getByRole("button", { name: "Vigilância" }).getAttribute("aria-current")).toBe("page");
    expect(await screen.findByText("nenhuma versão da lista")).not.toBeNull();
  });
```

(Se o formato de erro de campo inexistente que o `gql` transforma em `GraphQLRefusal` for outro, copie o do teste equivalente de `CareLineTemplates.test.tsx`.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod25 src/screens/NotifiableDiseases.test.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx`
Expected: FAIL — a tela e o item do trilho não existem.

- [ ] **Step 3: Implemente** a tela (molde de `CareLineTemplates.tsx`: consulta própria com `staleTime: Infinity`, `atualizar` no `Panel`, `DataTable` com `columns`), as fixtures e o item do trilho; rode `cd $M && npm run codegen`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod25 src/screens src/theme && cd $M && npx tsc --noEmit && npm run codegen:check`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/NotifiableDiseases.tsx src/screens/NotifiableDiseases.fixtures.tsx src/screens/NotifiableDiseases.test.tsx src/screens/Shell.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx src/gql/gql.ts src/gql/graphql.ts
$G commit -m "feat: show the notifiable disease list and its versions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Editar e publicar versão nova, com step-up

**Files:**
- Create: `src/screens/NotifiableDiseaseListForm.tsx`
- Modify: `src/screens/NotifiableDiseases.tsx`, `src/gql/*` (gerado)
- Test: `src/screens/NotifiableDiseaseListForm.test.tsx`

**Interfaces:**
- Consumes: `toDraft`, `emptyDisease`, `diseaseProblem`, `draftProblem`, `draftToVariables`, `listDiff`, `diffText`, `publishErrorText`, `DEADLINES`, `deadlineText`, `NOTIFIABLE_QUERY_KEY` (Task 2); `NotifiableDiseases`, as fixtures (Task 3); `Field`, `Button`, `ErrorState`, `StepUpCodeError`, `STEP_UP_CODE_HELP_TEXT`.
- Produces: o documento `mutation PublishNotifiableDiseaseList($code: String!, $expectedVersion: Int, $ordinance: String!, $effectiveFrom: ISO8601Date!, $diseases: [NotifiableDiseaseInput!]!) { publishNotifiableDiseaseList(code: $code, expectedVersion: $expectedVersion, ordinance: $ordinance, effectiveFrom: $effectiveFrom, diseases: $diseases) { ok version errors { path message } } }`; `NotifiableDiseaseListForm({ base: ListView | null, today, busy, outcome, onPublish(draft, code), onClose })`.

Comportamento: "editar e publicar versão nova" (ou "primeira versão") abre o formulário a partir da vigente (`toDraft`). Campos "Portaria" e "Início da vigência" (`type="date"`). A lista de doenças em tabela (Doença, Prazo, CID-10, CIAP-2, ações "editar"/"retirar"); "editar" e "acrescentar doença" abrem um subformulário com Código (só editável em doença nova), Nome, Prazo (`<select>` com `DEADLINES`; valor desconhecido continua selecionado e aparece cru), CID-10 e CIAP-2 (texto separado por vírgula), Janela de repetição (dias), Ficha do Sinan, Nota, Fonte do prazo e Fonte dos códigos (as outras fontes da linha não aparecem e voltam como vieram); "aplicar" confere `diseaseProblem` e fecha; "retirar" pede confirmação na própria linha ("retirar Sarampo da versão nova?"). Antes do botão: a diferença (`diffText`) e o aviso "Vale para todas as cidades a partir da vigência; as notificações já feitas não mudam." O campo "Código" (TOTP, `autoComplete="one-time-code"`, `helpText={STEP_UP_CODE_HELP_TEXT}`) e "publicar versão N" (N = vigente + 1; 1 sem vigente). Publicar: `draftProblem` antes (frase no topo, nada enviado); depois a mutation com `draftToVariables(draft, code, base?.version ?? null)`. Recusas: `path: "code"` → `StepUpCodeError` no campo, o código é limpo, o rascunho fica; as outras → `publishErrorText` num `ErrorState`, o rascunho fica; sucesso → "Versão N publicada." e relê `NOTIFIABLE_QUERY_KEY`, fechando o formulário. Rede ou timeout: relê a lista e mantém o formulário (o api pode ter gravado).

- [ ] **Step 1: Write the failing test**

```tsx
// src/screens/NotifiableDiseaseListForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { V2, renderScreen, reply } from "./NotifiableDiseases.fixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });
type Body = { query: string; variables: Record<string, unknown> };
const bodyOf = (call: unknown[]): Body => JSON.parse((call[1] as RequestInit).body as string);

describe("NotifiableDiseaseListForm", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  let publishReply: unknown;
  beforeEach(() => {
    publishReply = { data: { publishNotifiableDiseaseList: { ok: true, version: 3, errors: [] } } };
    fetchMock = vi.fn((_: string, init?: RequestInit) => {
      const body = JSON.parse(String(init?.body)) as Body;
      if (body.query.includes("mutation PublishNotifiableDiseaseList")) return Promise.resolve(reply(200, publishReply));
      return Promise.resolve(reply(200, { data: { notifiableDiseases: [ V2 ] } }));
    });
    vi.stubGlobal("fetch", fetchMock);
  });
  const publishes = () => fetchMock.mock.calls.filter((c) => bodyOf(c).query.includes("mutation PublishNotifiableDiseaseList"));
  const lists = () => fetchMock.mock.calls.filter((c) => bodyOf(c).query.includes("query NotifiableDiseases"));

  // Data longe no futuro: o teste não depende do dia em que roda.
  const setValidFrom = (value: string) => fireEvent.change(screen.getByLabelText("Início da vigência"), { target: { value } });
  async function openForm(user: ReturnType<typeof userEvent.setup>) {
    renderScreen();
    await user.click(await screen.findByRole("button", { name: "editar e publicar versão nova" }));
  }

  it("publica a versão 3 a partir da vigente: muda o prazo, acrescenta uma doença, com a vigência e o TOTP", async () => {
    const user = userEvent.setup();
    await openForm(user);
    setValidFrom("2099-11-01");
    await user.click(within(screen.getByRole("row", { name: /Dengue/ })).getByRole("button", { name: "editar" }));
    await user.selectOptions(screen.getByLabelText("Prazo"), "immediate");
    await user.click(screen.getByRole("button", { name: "aplicar" }));
    await user.click(screen.getByRole("button", { name: "acrescentar doença" }));
    await user.type(screen.getByLabelText("Código da doença"), "mpox");
    await user.type(screen.getByLabelText("Nome"), "Mpox");
    await user.type(screen.getByLabelText("CID-10"), "B04");
    await user.type(screen.getByLabelText("Fonte do prazo"), "Portaria GM/MS 11.211/2026, anexo 1");
    await user.type(screen.getByLabelText("Fonte dos códigos"), "ficha de notificação de mpox");
    await user.click(screen.getByRole("button", { name: "aplicar" }));
    expect(screen.getByText("1 nova, 0 retiradas, 1 alterada")).not.toBeNull();
    expect(screen.getByText(/Vale para todas as cidades a partir da vigência/)).not.toBeNull();
    await user.type(screen.getByLabelText("Código"), "123456");
    await user.click(screen.getByRole("button", { name: "publicar versão 3" }));
    await waitFor(() => expect(publishes()).toHaveLength(1));
    const v = bodyOf(publishes()[0]).variables;
    expect(v).toMatchObject({ code: "123456", expectedVersion: 2, effectiveFrom: "2099-11-01", ordinance: "Portaria GM/MS nº 11.211/2026" });
    expect((v.diseases as { code: string; deadline: string }[]).map((d) => [ d.code, d.deadline ]))
      .toEqual([ [ "dengue", "immediate" ], [ "sarampo", "immediate" ], [ "mpox", "weekly" ] ]);
    expect(await screen.findByText("Versão 3 publicada.")).not.toBeNull();
    await waitFor(() => expect(lists().length).toBeGreaterThanOrEqual(2));
  });

  it("CID inválido é recusado e fica no campo; nada é enviado", async () => {
    const user = userEvent.setup();
    await openForm(user);
    await user.click(within(screen.getByRole("row", { name: /Sarampo/ })).getByRole("button", { name: "editar" }));
    await user.clear(screen.getByLabelText("CID-10"));
    await user.type(screen.getByLabelText("CID-10"), "B0");
    await user.click(screen.getByRole("button", { name: "aplicar" }));
    expect(screen.getByText("CID-10 inválido: B0")).not.toBeNull();
    expect((screen.getByLabelText("CID-10") as HTMLInputElement).value).toBe("B0");
    expect(publishes()).toHaveLength(0);
  });

  it("vigência antes de hoje é recusada antes de enviar", async () => {
    const user = userEvent.setup();
    await openForm(user);
    setValidFrom("2020-01-01");
    await user.type(screen.getByLabelText("Código"), "123456");
    await user.click(screen.getByRole("button", { name: "publicar versão 3" }));
    expect(screen.getByText("a vigência não pode começar antes de hoje")).not.toBeNull();
    expect(publishes()).toHaveLength(0);
  });

  it("TOTP recusado: no campo, com o aviso de bloqueio; o rascunho fica", async () => {
    const user = userEvent.setup();
    publishReply = { data: { publishNotifiableDiseaseList: { ok: false, version: null, errors: [ { path: "code", message: "código inválido" } ] } } };
    await openForm(user);
    setValidFrom("2099-11-01");
    await user.type(screen.getByLabelText("Código"), "000000");
    await user.click(screen.getByRole("button", { name: "publicar versão 3" }));
    expect(await screen.findByText("código inválido")).not.toBeNull();
    expect(screen.getByText("Tentativas erradas contam para o bloqueio da conta.")).not.toBeNull();
    expect((screen.getByLabelText("Código") as HTMLInputElement).value).toBe("");
    expect((screen.getByLabelText("Início da vigência") as HTMLInputElement).value).toBe("2099-11-01");
  });

  it("versão de partida desatualizada: frase, o rascunho fica", async () => {
    const user = userEvent.setup();
    publishReply = { data: { publishNotifiableDiseaseList: { ok: false, version: null, errors: [ { path: "expectedVersion", message: "stale" } ] } } };
    await openForm(user);
    setValidFrom("2099-11-01");
    await user.type(screen.getByLabelText("Código"), "123456");
    await user.click(screen.getByRole("button", { name: "publicar versão 3" }));
    expect(await screen.findByText("outra pessoa publicou uma versão enquanto você editava — recarregue")).not.toBeNull();
    expect(screen.getByRole("button", { name: "publicar versão 3" })).not.toBeNull();
    expect(publishes()).toHaveLength(1);
  });
});
```

(O formulário recebe `today` da tela — a data local de hoje em `AAAA-MM-DD`; os testes usam 2099 e 2020 para não depender do dia da execução. O rótulo "Código da doença" separa o código da doença do "Código" do TOTP.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod25 src/screens/NotifiableDiseaseListForm.test.tsx`
Expected: FAIL — o formulário e a mutation não existem.

- [ ] **Step 3: Implemente** o formulário como descrito, a mutation na tela (molde da `save` de `CareLineTemplates.tsx`, com o `code` separado como no `Maintainers.tsx`) e rode `cd $M && npm run codegen`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod25 src/screens src/lib src/theme && cd $M && npx tsc --noEmit && npm run codegen:check`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/NotifiableDiseaseListForm.tsx src/screens/NotifiableDiseaseListForm.test.tsx src/screens/NotifiableDiseases.tsx src/gql/gql.ts src/gql/graphql.ts
$G commit -m "feat: publish new notifiable disease list versions with step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Suíte, schema igual ao do api, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo (o `schema.graphql` só muda se o do api divergir).

**Interfaces:**
- Consumes: as Tasks 1–4 na branch `feat/mod-25-disease-surveillance`; o api do 25 na **3047**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 25).

- [ ] **Step 1: Schema igual ao do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; M=$R/.worktrees/maintenance-mod25
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod25 api bin/rails runner 'puts Maintenance::Schema.to_definition' > /tmp/mod25-schema.graphql
diff <(grep -v '^\s*$' /tmp/mod25-schema.graphql) <(grep -v '^\s*$' $M/schema.graphql) && echo schema-igual
```

Expected: `schema-igual`. Diferença: copie o do api (`npm run schema:pull`, se o script servir; senão o arquivo acima), rode `npm run codegen` e ajuste as Tasks 2–4 ao que mudou (com teste), anotando a diferença no relatório da task (M1).

- [ ] **Step 2: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod25 && cd $M && npm run typecheck && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base da Task 0 **+3 arquivos de teste** e os casos novos em `features`, `FeaturesTab`, `Shell` e `Shell.rail`. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G diff origin/main --stat
grep -rn "maxLength\|console\.\|localStorage" src/lib/notifiableDiseases.ts src/screens/NotifiableDisease*.tsx | grep -v "\.test\."
```

Expected: `status` vazio; o diff só com os arquivos do mapa; o `grep` sem saída.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§2 decisão 1 e as do coordenador, §3, §9 F-25.1), o ADR 0043 e o contrato §2, §3, §11 e §12. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; o rótulo igual ao do dashboard; nenhum pré-requisito inventado;
- publicar é sempre versão nova, com `expectedVersion` e o TOTP; o código do TOTP nunca fica na tela depois da recusa;
- recusar, nunca cortar; valor desconhecido cru;
- nada de doença local, notificação, cidade ou cidadão aqui;
- o README não promete envio ao Sinan nem fala em surto.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 25 na **3047** (plano do api: banco de dev migrado e semeado **com autorização do usuário**, com a carga inicial da lista). O proxy do Vite do maintenance injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod25 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod25:/app -v maintenance-mod25-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3047 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod25   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 25 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor e digita o TOTP quando pedido; não digite senha nem TOTP. **Anote** antes de mexer o estado de `surveillance` de Curitiba e a versão vigente da lista. Roteiro, com screenshot em `evidencias/2026-10-27-modulo-25/maintenance/`:
1. Cidades → Curitiba → **Funcionalidades**: `surveillance` com "Vigilância epidemiológica" e a descrição do api; ligar (dois cliques) → "surveillance ligada e utilizável." Numa cidade em modo `off` ou `integrated` (se houver na semente), ligar também dá "ligada e utilizável".
2. **Vigilância** no trilho: a vigente da carga inicial (portaria, vigência, número de doenças, "carga inicial"), as doenças com prazo, CID-10, CIAP-2 e fonte; "versões".
3. **Publicar versão nova**: vigência de amanhã; mudar a janela de uma doença; acrescentar uma doença de teste com um CID; a diferença "1 nova, 0 retiradas, 1 alterada"; um CID "A9" é recusado no subformulário; TOTP errado → no campo, com o aviso de bloqueio; TOTP certo → "Versão N publicada.", a nova aparece "agendada para <amanhã>".
4. **Concorrência**: o formulário aberto em duas abas; publicar na primeira e tentar na segunda → "outra pessoa publicou…", o rascunho fica.
5. **Auditoria** mostra `notifiable_disease_list.published` (sem dado de cidade) e as tentativas de `setCityFeature` para `surveillance`.
6. No dashboard (prova do plano do dashboard), com a versão agendada vencida (ou uma de vigência hoje, se o api aceitar), o CID da doença de teste gera a sugestão na consulta.
7. A 375 px de largura: sem rolagem lateral da página (a tabela rola dentro do `TableScroll`).

Depois: devolva `surveillance` ao estado anotado; a versão de teste publicada fica (é imutável) — publique, com o usuário, uma versão que retire a doença de teste se ele pedir; `docker rm -f maintenance-mod25 && docker volume rm maintenance-mod25-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 25, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-25-disease-surveillance
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod25
/opt/homebrew/bin/git branch -d feat/mod-25-disease-surveillance
```

Se as partes do api e do dashboard do F-25.1 já estiverem **Done**, mova o card F-25.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 4 e a do dashboard passaram.

---

### Task 6: O módulo 25 no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-25.1..F-25.8 no board #1 (status real de cada um); o resultado das provas do 25 (planos do api, do dashboard e deste); o que foi entregue em api, dashboard e maintenance.
- Produces: módulo `"25"` no JSON (ciclo 2, depois do `"24"` e antes do `"iu"`), nome "Vigilância epidemiológica", `status: "validating"`, com F-25.1..F-25.8; `meta.cycles[1].modules` = "15 a 25 e interface"; o rodapé "o 19, o 20, o 21, o 22, o 23, o 24 e o 25 estão em validação".

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-25\\.[1-8] ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod25 -b content/module-25-catalog origin/main
cd $R/.worktrees/catalog-mod25 && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-25.1..F-25.8 em Done ou Verified; os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário. A suíte do catálogo verde. **Anote**: o JSON termina o ciclo 2 em `"24"` e `"iu"` (`a3af8f7` ou mais novo); o rodapé diz "o 19, o 20, o 21, o 22, o 23 e o 24 estão em validação"; o ciclo 2 diz "15 a 24 e interface".

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("formatValidating")`, depois da linha com `"24"`:

```ts
    expect(formatValidating(["19", "20", "21", "22", "23", "24", "25"])).toBe("o 19, o 20, o 21, o 22, o 23, o 24 e o 25 estão");
```

e, no fim do arquivo:

```ts
describe("módulo 25", () => {
  it("entra no ciclo 2 depois do 24: digitação manual no Sinan, sinal sem surto, ficha em validação", () => {
    const ids = catalog.modules.map((m) => m.id);
    expect(ids.indexOf("25")).toBe(ids.indexOf("24") + 1);
    const m25 = catalog.modules.find((m) => m.id === "25")!;
    expect(m25.cycle).toBe(2);
    expect(m25.status).toBe("validating");
    expect(m25.name).toBe("Vigilância epidemiológica");
    expect(m25.features.map((f) => f.id)).toEqual(Array.from({ length: 8 }, (_, i) => `F-25.${i + 1}`));
    for (const id of [ "f-25.1", "f-25.2", "f-25.3", "f-25.4", "f-25.5", "f-25.6", "f-25.8" ]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const outputs = findFeature("f-25.7")!.feature;
    expect(outputs.status).toBe("validating");
    expect(outputs.note).toMatch(/Sinan/);
    expect(findFeature("f-25.1")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    expect(findFeature("f-25.3")?.feature.surfaces).toEqual([ "bastidores" ]);
    expect(findFeature("f-25.2")?.feature.description).toMatch(/não impede/);
    expect(findFeature("f-25.5")?.feature.description).toMatch(/à mão/);
    expect(findFeature("f-25.8")?.feature.description).toMatch(/sinal/);
    const text = JSON.stringify(m25);
    expect(text).not.toMatch(/surto|vacina|automaticamente|envia(m)? ao Sinan/i);
    expect(catalog.meta.cycles.find((c) => c.id === 2)?.modules).toBe("15 a 25 e interface");
  });
});
```

Em `src/App.test.tsx`, troque

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23 e o 24 estão em validação."
```

por

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22, o 23, o 24 e o 25 estão em validação."
```

e `"módulos 15 a 24 e interface"` por `"módulos 15 a 25 e interface"`.

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts src/App.test.tsx`
Expected: FAIL — não há módulo 25, o ciclo 2 diz "15 a 24" e o rodapé não fala do 25. (O teste de `formatValidating` com sete já passa: a função é genérica.)

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script (o mesmo do 24a: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo byte a byte e o script confere). **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard, regras do api) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

ids = [m["id"] for m in data["modules"]]
assert "25" not in ids, "módulo 25 já existe"
assert data["meta"]["cycles"][1]["modules"] == "15 a 24 e interface", "ciclo 2 mudou: pare e reporte"

data["meta"]["updatedAt"] = today
data["meta"]["cycles"][1]["modules"] = "15 a 25 e interface"

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-25.{n}", "slug": f"f-25.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m25 = {
    "id": "25",
    "cycle": 2,
    "name": "Vigilância epidemiológica",
    "status": "validating",
    "description": (
        "Nenhum caso de doença de notificação obrigatória atendido no Rota Saúde passa sem a vigilância municipal saber. "
        "O médico ou o enfermeiro confirma a suspeita na consulta, a vigilância acompanha cada caso com prazo até a digitação "
        "no Sinan, que continua feita à mão, e vê por bairro e semana epidemiológica onde os casos estão aumentando."
    ),
    "features": [
        feature(1, "Interruptor, papel de vigilância e lista de doenças",
            "A lista nacional de notificação compulsória, sempre na versão vigente.",
            "A equipe da plataforma liga a vigilância por cidade e mantém a lista nacional de doenças e agravos, com prazo e "
            "códigos, publicando versões novas com confirmação em duas etapas. A cidade acrescenta suas doenças locais e dá o "
            "papel de vigilância a quem acompanha os casos.",
            ["plataforma", "secretaria"]),
        feature(2, "Sugestão de notificação na consulta",
            "O médico notifica a suspeita sem sair do atendimento.",
            "Quando a consulta registra um código de doença da lista, aparece o aviso para notificar a suspeita, com a data "
            "dos primeiros sintomas. Médico e enfermeiro também podem notificar à mão a qualquer momento, e o aviso não impede "
            "finalizar a consulta.",
            ["secretaria"]),
        feature(3, "Rede de segurança na finalização",
            "Caso esquecido na consulta chega à vigilância.",
            "A consulta finalizada com código da lista e sem notificação vira uma possível notificação não feita na fila da "
            "vigilância, que confirma ou descarta com motivo.",
            ["bastidores"]),
        feature(4, "Fila, notificação manual e ficha",
            "A vigilância vê todos os casos num só lugar.",
            "A fila mostra primeiro os casos imediatos e os vencidos. A vigilância registra casos que chegaram de hospital, "
            "laboratório ou outra cidade, com busca da pessoa e de possíveis duplicados, e completa a ficha de notificação.",
            ["secretaria"]),
        feature(5, "Andamento, encerramento e número oficial",
            "Cada caso tem início, investigação e fim registrados.",
            "O caso passa de notificado a em investigação e a encerrado, com a classificação final. A digitação no Sinan é "
            "feita à mão, e o número da notificação no sistema oficial fica registrado no caso.",
            ["secretaria"]),
        feature(6, "Prazos, alertas e notificação negativa",
            "Nenhum caso imediato passa das 24 horas sem alguém ver.",
            "O prazo conta da data da suspeita: 24 horas para as imediatas e 7 dias para as semanais. Caso imediato gera "
            "alerta no painel e e-mail sem dado do paciente, e a unidade sem casos na semana registra a notificação negativa.",
            ["secretaria"]),
        feature(7, "Ficha em PDF, CSV por semana e relatório sem número",
            "O que precisa ser digitado no Sinan, pronto para digitar.",
            "A ficha de notificação individual sai em PDF no formato do Sinan, os casos de cada semana epidemiológica saem "
            "em planilha sem CPF, e um relatório mostra os casos encerrados ainda sem número oficial.",
            ["secretaria"], status="validating",
            note="A ficha foi conferida com o modelo do Sinan; falta a vigilância da cidade piloto digitar casos reais a partir dela."),
        feature(8, "Painel por bairro e semana epidemiológica",
            "Onde os casos estão aumentando.",
            "Casos por bairro do domicílio e semana epidemiológica, com contagens pequenas escondidas para não identificar "
            "ninguém, e um sinal quando a semana passa do dobro da média das quatro anteriores — um aviso para a vigilância avaliar.",
            ["secretaria"]),
    ],
}

data["modules"].insert(ids.index("24") + 1, m25)
open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", [m["id"] for m in data["modules"]][-3:], len(m25["features"]))
PY
```

Expected: `ok ['24', '25', 'iu'] 8`. Se o Step 1 mostrar algum F-25.x (além do F-25.7) que não chegou a Verified, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2. Se a ficha complementar (Task 8 do dashboard) tiver ficado para depois, F-25.4 fica `validating` com a nota "a ficha complementar entra depois da publicação dos campos" e a frase "e completa a ficha de notificação" sai.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 25 com 8 funcionalidades; o rodapé e o ciclo com os textos novos.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os três arquivos da task (nenhum `package-lock.json`); no JSON, só o módulo 25 acrescentado, o ciclo 2 e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 25 disease surveillance to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot (em `evidencias/2026-10-27-modulo-25/catalogo/`): o **Módulo 25 — Vigilância epidemiológica** no ciclo 2, logo depois do 24, com a descrição e as 8 funcionalidades; a F-25.7 "Em validação" com a nota; o rodapé "o 19, o 20, o 21, o 22, o 23, o 24 e o 25 estão em validação"; a página `/funcionalidades/f-25.2` com "O que faz", as telas e anterior/próxima dentro do módulo (F-25.1 e F-25.3); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod25
/opt/homebrew/bin/git push -u origin content/module-25-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-25-catalog --title "docs: add module 25 disease surveillance to the catalog" --body "Adds module 25 (disease surveillance) after module 24 with F-25.1..F-25.8; the Sinan-format outputs (F-25.7) stay in validation until the pilot city enters real cases from them; entry into Sinan stays manual; cycle 2 and the footer updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod25 && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-25-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — SDL da lista.** O §3 diz `notifiableDiseases` (vigente e histórico) e `publishNotifiableDiseaseList(input)` → `{ ok, errors }`, sem os tipos. Este plano usa o SDL do plano do api (Task 4, V13): `NotifiableDiseaseVersion { version, ordinance, effectiveFrom, current, createdAt, publishedBy, diseases }`, `NotifiableDisease { code, name, deadline, cid10, ciap2, repeatAfterDays, sinanForm, notes, sources }`, `NotifiableDiseaseInput` e a mutation com argumentos soltos (`ordinance`, `effectiveFrom`, `diseases`, `code`) devolvendo `{ ok, errors, version }`. Proposta: o contrato §3 registra esses nomes; `sources` é JSON com ao menos `deadline` e `cid10` (a "fonte por linha"); a Task 0 compara com o api real e o api vence.
2. **M2 — Versão de partida.** O plano do api não tem como saber de que versão o rascunho partiu: duas publicações simultâneas gravam duas versões, e a segunda desfaz a primeira sem ninguém ver. Proposta: argumento `expectedVersion: Int` (a vigente de onde o rascunho partiu; divergente → `errors: [{ path: "expectedVersion" }]`, nada gravado), no molde do `revertProtocolActivation(expectedVersion:)`. Recusada, o plano tira o argumento e o teste correspondente (Task 2, cabeçalho).
3. **M3 — Código da doença.** O contrato não fixa. Alinhado com o plano do api: plataforma `^[a-z][a-z0-9_]{1,47}$` sem `local_`; doença local `local_[a-z0-9_]{1,40}` (V2 do api; D4 do plano do dashboard). Proposta: o contrato §2/§3 registra os dois formatos.
4. **M4 — Vigência.** O contrato fala em "vigência" sem regra; o plano do api calcula `current` (maior versão com `effectiveFrom` até hoje), mas não diz se aceita `effectiveFrom` no passado. Proposta: `effectiveFrom` não antes de hoje (no fuso da plataforma) nem antes da vigência da versão vigente — mudar a lista no passado reclassificaria sugestões já feitas; a versão com `effectiveFrom` no futuro fica **agendada**; notificações guardam a versão em que nasceram (`list_version`). E a janela de repetição vai até 36500 (o CHECK do api), não 3650.
5. **M5 — Auditoria.** O §12 nomeia `notifiable_disease_list.published`; o plano do api grava esse (plataforma) e também `maintenance.notifiable_disease_list.published` (auditoria da mutation), os dois em `R18_PLATFORM_EVENT_NAMES`. Proposta: o contrato §12 registra os dois nomes; nenhum com diff textual da lista (só versão, vigência e número de doenças).
6. **M6 — Retirar doença numa versão nova.** O contrato não diz se uma versão pode tirar doença. Proposta: pode (a portaria muda), e a tela mostra a retirada na diferença antes de publicar; notificações abertas da doença retirada seguem até o fim, e a doença some das sugestões a partir da vigência.

## Self-review

- **Cobertura (spec §2 decisão 1 e as do coordenador, §3, §8, §9; contrato §1, §3, §11–§13):** interruptor pelo mecanismo genérico, sem pré-requisito — Task 1 (Review Focus 1); desligar sem apagar — Task 1; todos os ambientes, sem simulado — Task 1; lista versionada, vigente, agendada e histórico — Tasks 2 e 3; editar, acrescentar, retirar e publicar versão nova com step-up e auditoria — Tasks 2, 4 e 5 (Review Focus 2 e 3); CID-10 e CIAP-2 por doença, janela de repetição, ficha do Sinan e fonte por linha — Tasks 2 e 4 (Review Focus 4); api sem o 25 — Task 3 (Review Focus 5); README — Task 1; schema igual ao do api, suíte, build e prova no navegador — Task 5; catálogo público com F-25.1..F-25.8, F-25.7 em validação, sem prometer envio ao Sinan nem surto — Task 6; cards — Tasks 0, 5 e 6. As doenças locais são do dashboard (Task 12 do plano do dashboard).
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); o SDL e as regras do api (Task 0 e Task 5, com a regra escrita para diferença); status dos cards F-25.x e o texto do catálogo marcado como rascunho a conferir contra o entregue (Task 6).
- **Tipos e nomes:** `DiseaseView`, `ListView`, `DiseaseDraft`, `ListDraft` (Task 2) são os das Tasks 3–4; `draftToVariables(draft, code, expectedVersion)` é a assinatura usada no teste da Task 2 e na Task 4; `NOTIFIABLE_QUERY_KEY` igual nas Tasks 2–4; `V1`, `V2`, `V3`, `reply`, `renderScreen` vêm de `NotifiableDiseases.fixtures.tsx` (Task 3) nas Tasks 3–4; `featureLabel`, `missingText`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem no maintenance; `catalog`, `findFeature`, `formatValidating` são os de `catalog.test.ts`; o rótulo "Vigilância epidemiológica" é o mesmo da Task 1 do plano do dashboard.
- **Review Focus:** 1 — Task 1 ("liga utilizável sem o prontuário; desliga com dois cliques; nada fala em apagar"); 2 — Task 4 ("versão de partida desatualizada: frase, o rascunho fica"); 3 — Task 4 ("TOTP recusado: no campo, com o aviso de bloqueio; o rascunho fica"); 4 — Task 2 ("doença: formato, recusar sem cortar" e "lista: códigos únicos, vigência e portaria") e Task 4 ("CID inválido é recusado e fica no campo; nada é enviado"); 5 — Task 3 ("api sem o 25: só esta tela cai, com a frase").
- **Proporção:** os blocos de código são testes, o SDL (o do plano do api, mais M2) e o script do catálogo (o texto exato do catálogo é copy que a spec fixa); a tela e o formulário ficam com o comportamento e as frases.
