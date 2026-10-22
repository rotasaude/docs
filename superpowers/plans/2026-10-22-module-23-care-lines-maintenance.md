# Módulo 23 — Linhas de cuidado (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 23** na branch `feat/mod-23-care-lines` (worktree `apps/api/.claude/mod23`, servidor de dev na porta **3045**) com `care_lines` em `Platform::Features::CATALOG` (`requires: ["feature:clinical_record"]`) e, no `Maintenance::Schema`, a consulta `careLineTemplates(key)` e a mutation `upsertCareLineTemplate` (contrato §8, na forma da Task 4 do plano do api `2026-10-22-module-23-care-lines-api.md`: elegibilidade e boas práticas em `JSON`, L1/L2), com a auditoria `maintenance.care_line_template.changed` (L13) declarada em `R18_PLATFORM_EVENT_NAMES`. Contrato §10: api → dashboard e wpda → **maintenance** → catálogo público. As Tasks 0–4 não dependem do api rodando (testes com `fetch` falso; o SDL novo entra à mão na Task 2 e é conferido contra o do api na Task 5); a prova da Task 5 precisa do api do 23. A Task 6 (catálogo público) vem **por último**, depois da prova e das entregas de api, dashboard, wpda e maintenance.

**Goal:** Lado maintenance de F-23.1: o interruptor `care_lines` aparece na aba Funcionalidades com nome em português e liga pelo mecanismo genérico, exigindo `clinical_record` (a frase "prontuário da atenção primária (clinical_record) desligado" do módulo 19 quando falta); e a tela nova **"Linhas de cuidado"** no trilho, com o **catálogo-base** das seis linhas (C2–C7) da plataforma: a versão vigente de cada uma, o histórico de versões e a publicação de **versão nova** (elegibilidade por CIAP-2/CID-10 com exclusões, idade em anos ou dias, sexo e DUM; boas práticas com fonte, janela, quantidade, cronograma, âncora, CBO, pontos e o texto para o cidadão; os campos avançados preservados) via GraphQL. E as F-23.1..F-23.8 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única).

**Architecture:** O liga/desliga não muda (`setCityFeature`, três estados, `missing` vindo do api): entra um rótulo em `FEATURE_LABELS` de `src/lib/features.ts`. O catálogo-base segue o molde do catálogo de laboratório do 21b (`LabTemplates`): consulta e mutation **próprias** da tela (contra um api sem o 23, só esta tela cai), regras puras em `src/lib/careLineTemplates.ts` (rótulos, regra de cada boa prática, versão vigente e histórico, rascunho a partir da vigente, conferência de formato — recusar, nunca cortar —, variáveis da mutation, frases das recusas), a tela `src/screens/CareLineTemplates.tsx` e o formulário `src/screens/CareLineTemplateForm.tsx`, reaproveitando `parseDecimal`/`decimalText`/`optionsWith` de `labTemplates.ts` e os estilos exportados de `LabTemplates.tsx`. O trilho (`Shell.tsx`) ganha o item "Linhas de cuidado". O SDL do maintenance ganha o tipo e os campos do plano do api (elegibilidade e práticas como `JSON`). Publicar versão nova não muda nenhuma cidade: cada cidade adota no dashboard.

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-22-module-23-care-lines-design.md` (§1 intenção, §2 decisões 1 e 9 e as do coordenador — metas por contagem, consulta que conta —, §3 interruptor, §4 "Plataforma", §6 "Maintenance", §9 entrega itens 4 e 5, §10 a tabela inteira para o catálogo, §11 riscos), `docs/adr/0040.md` e o contrato `docs/superpowers/plans/2026-10-22-module-23-care-lines-contracts.md` §1, §8, §9 e §10 e o plano do api (L1, L2, L4, L13); as diferenças estão em "Divergências propostas ao contrato", M1–M5. Base: plano do maintenance do 22 (`2026-10-21-module-22-billing-maintenance.md`) e a tela do catálogo de laboratório do 21b.

## Global Constraints

- Interruptor (contrato §8; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `care_lines` com `requires: ["feature:clinical_record"]`: sem `clinical_record` ligado e utilizável, o `missing` vem `["clinical_record_disabled"]` e a frase é a do módulo 19 — "prontuário da atenção primária (clinical_record) desligado" —, **não** redefinida; o modo `record` chega por tabela (o `clinical_record` já exige `record_mode_not_record`);
  - existe em todos os ambientes, produção inclusive; **não** é simulado: nenhum aviso de simulado por causa dele.
- Nome: `care_lines` → "Linhas de cuidado" (o mesmo do dashboard, `src/lib/features.ts` de lá).
- Catálogo-base (contrato §1 e §8; spec §4 "Plataforma"):
  - as seis chaves do contrato §1 — `child_c2`, `pregnancy_c3`, `diabetes_c4`, `hypertension_c5`, `elderly_c6`, `women_c7` — com o indicador de cada uma (C2…C7); a tela não cria chave nova (chave desconhecida vinda do api aparece crua e pode receber versão nova);
  - fonte da boa prática `consultation` | `blood_pressure` | `weight_height` | `exam:<sigtap>` | `procedure:<sigtap>` | `acs_visit` | `vaccine:<código>` | `oral_health`; cronograma `proportional` | `end_of_window`; âncora `enrollment` | `birth` | `lmp` | `rolling`; `acs_visit`, `oral_health` e `vaccine:*` são "não acompanhadas pelo sistema" (nunca contam como atraso; ADR 0040);
  - elegibilidade e boas práticas são **JSON** na forma L1/L2 do plano do api (`snake_case`); a tela edita os campos simples e **preserva sem editar** os avançados (`all_of`, `code_filter`, `applies_to`, identidades de gênero, `declared`), mostrando-os em texto;
  - **versão nova não muda as cidades** até cada uma adotar (spec §4): publicar é sempre criar a próxima versão (`upsertCareLineTemplate` = nova versão), nunca editar a vigente; a tela diz isso antes do botão;
  - o maintenance não vê nada de cidade: nenhum paciente, inscrição, contato, condição declarada ou indicador de equipe passa por aqui.
- Recusar em vez de cortar (lições dos contratos do 21a §15 e 21b §14; limites do plano do api, "Valores fixados" 4): nenhum `maxLength` nos campos novos; nome e rótulo até 120; texto para o cidadão até 80; até 20 boas práticas; CIAP-2 `A00`; CID-10 `A00`/`A000`/`A00X` (por prefixo, sem ponto); exame `02` + 8 dígitos; procedimento 10 dígitos; vacina `[a-z0-9_]{1,30}`; CBO por prefixo de 1 a 6 dígitos; janela de 1 a 7300 dias, início antes do fim; quantidade de 1 a 50; proporcional só com âncora fixa; idade em anos (0–130) ou em dias; **pontos** com **vírgula decimal**, de 0 a 100 e **somando 100** (ponto é recusado, como no catálogo de laboratório) — tudo conferido antes de enviar e de novo no api.
- Valor desconhecido (api mais novo: chave, fonte, cronograma, âncora, código de `missing` ou de erro) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): componentes existentes (`Panel`, `DataTable` — que já embrulha em `<TableScroll>` —, `Tag`, `Button`, `Field`, `EmptyState`, `ErrorState`), sem redesign nem componente comum novo; cores só por papel do tema; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte. Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); o que só foi provado com dado de exemplo fica `validating` com nota; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-23.1..F-23.8 do board #1 em In Progress (Task 0). O maintenance é, com o api, dono do F-23.1: Done do F-23.1 quando api **e** maintenance estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod23`, `C=$R/.worktrees/catalog-mod23`, `A=$R/apps/api/.claude/mod23`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-23-care-lines`; catálogo em `$C`, branch `content/module-23-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task.
- maintenance — testes: `cd $M && npx vitest run <arquivos>`; tipos: `npx tsc --noEmit`; codegen: `npm run codegen` (gera `src/gql/*` a partir de `schema.graphql` e dos documentos `graphql(...)`); CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (os testes chamam `cleanup`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`).
- O api do 23 roda em dev na **3045** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade sem prontuário (modo `off`/`integrated` ou `clinical_record` desligado).** Ligar `care_lines` dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado" — a frase do módulo 19, sem redefinir —; com `clinical_record` ligado e utilizável, "ligada e utilizável". Teste: Task 1, "sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19".
2. **Pontos digitados com ponto, fora da faixa ou sem somar 100** ("84.5", "150,000", "-5", soma 95). Espera-se: recusado com a frase de vírgula decimal, da faixa ou da soma antes de enviar, com o texto mantido no campo; "84,5" vai como `84.5`. Testes: Task 2, "pontos: vírgula decimal, de 0 a 100, somando 100; ponto recusado"; Task 4, "pontos com ponto são recusados e ficam no campo; nada é enviado".
3. **Fonte com código e regras avançadas** (`exam:`, `procedure:`, `vaccine:`; `all_of` dos testes do pré-natal). Espera-se: escolher "exame" pede o SIGTAP do grupo 02 e manda `exam:0202010317`; trocar para "visita do ACS" some o código; fonte desconhecida continua selecionada; a versão nova de C3 manda o `all_of` e a elegibilidade exatamente como vieram. Testes: Task 2, "fonte: monta e desmonta o código; desconhecida continua" e "variáveis: JSON em snake_case, avançados preservados, vazios fora"; Task 4, "exame pede o SIGTAP e manda exam:<código>; visita do ACS some o código; avançados voltam iguais".
4. **Publicar acha que mexe nas cidades.** Espera-se: a tela diz, antes do botão, que a versão nova só vale para a cidade que a adotar e que as inscrições não mudam até lá; publicar sempre manda a chave da linha (versão nova), nunca um id de versão para editar. Teste: Task 4, "publica versão nova a partir da vigente, com o aviso de que as cidades adotam".
5. **Api sem o 23 ou recusa do api.** Contra um api sem `careLineTemplates`, só esta tela cai, com a frase "o api do módulo 23 precisa subir antes do maintenance"; recusa de regra (`invalid_practices` com o campo, código novo) aparece em português ou crua, e o formulário fica. Testes: Task 3, "api sem o 23: só esta tela cai, com a frase"; Task 4, "recusa do api vira frase com o campo; código novo cru; o formulário fica".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards F-23.1..F-23.8, conferência do main e do api do 23, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome do interruptor e o fluxo genérico da aba fixado para ele | 1 |
| `schema.graphql`, `src/lib/careLineTemplates.ts`, `src/lib/careLineTemplates.test.ts` | SDL do catálogo-base (forma do plano do api) e as regras da tela (JSON L1/L2) | 2 |
| `src/screens/CareLineTemplates.tsx`, `src/screens/CareLineTemplates.fixtures.tsx`, `src/screens/CareLineTemplates.test.tsx`, `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx`, `src/gql/*` | a tela no trilho: vigentes e histórico de versões | 3 |
| `src/screens/CareLineTemplateForm.tsx`, `src/screens/CareLineTemplateForm.test.tsx`, `src/screens/CareLineTemplates.tsx`, `src/gql/*` | publicar versão nova | 4 |
| — | suíte, schema igual ao do api, codegen:check, build, revisão, prova no navegador | 5 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx` | módulo 23 com F-23.1..F-23.8, rodapé e ciclo 2 | 6 |

**Estratégia de teste:** funções puras (Task 1: `featureLabel`, `missingText`; Task 2: rótulos, regra de cada boa prática, vigente e histórico, rascunho, conferência de formato com tabela de casos — o que recusa e o que **não** recusa —, variáveis da mutation, frases das recusas); o interruptor pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação e conferência das **variáveis** exatas da mutation (Task 1); a tela e o formulário com `fetch` falso por operação GraphQL, conferindo as variáveis enviadas, a releitura depois de publicar e as recusas (Tasks 3–4); suíte inteira com as guardas de tema, o SDL igual ao do api, `codegen:check`, build e a prova no navegador contra o api do 23 (Task 5). Catálogo: `validateCatalog`, o teste do módulo 23, o rodapé e o ciclo (Task 6).

---

### Task 0: Cards, conferência do main e do api do 23, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`f6d0a8d` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards F-23.1..F-23.8 em In Progress (ou além); worktree `$M` na branch `feat/mod-23-care-lines`; a base anotada.

- [ ] **Step 1: Mova os cards F-23.1..F-23.8 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh` (sem `P=$(...)` nem laço):

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-23\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Os que já estiverem em In Progress, Done ou Verified ficam como estão. Para cada um antes de In Progress:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito itens em In Progress ou além. Se algum card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main do maintenance e o api do 23**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod23
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n \
  '  clinical_record_disabled: "prontuário da atenção primária (clinical_record) desligado",\|  procedure_billing: "Faturamento de procedimentos (BPA)"\|care_lines'
$G -C $R/apps/maintenance show origin/main:src/lib/labTemplates.ts | grep -n '^export function parseDecimal\|^export function decimalText\|^export function optionsWith\|^export const DOT_IN_NUMBER'
$G -C $R/apps/maintenance show origin/main:src/screens/LabTemplates.tsx | grep -n '^export const muted\|^export const row\|^export const grid\|^export const labelStyle\|^export const inputStyle\|^export const checkStyle'
$G -C $R/apps/maintenance show origin/main:src/screens/Shell.tsx | grep -n 'export type Screen = "cities" | "medications" | "lab" | "maintainers" | "tokens" | "audit";\|  { key: "lab", label: "Laboratório" },\|          {screen === "lab" && <LabTemplates />}'
$G -C $R/apps/maintenance show origin/main:schema.graphql | grep -n '^  labExamTemplates: \[LabExamTemplate!\]!\|^  upsertLabExamTemplate(\|^type UpsertLabExamTemplatePayload {'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Faturamento de procedimentos (módulo 22).\*\*\|^\*\*Documentos clínicos (módulo 19c).\*\*'
grep -n -E 'key: "care_lines"' -A4 $A/app/services/platform/features.rb
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod23 api bin/rails runner 'puts Maintenance::Schema.to_definition' 2>/dev/null | grep -n -A14 'type CareLineTemplate \|careLineTemplates\|upsertCareLineTemplate\|input CareLine'
```

Expected:
- em `features.ts`: as duas âncoras (a frase de `clinical_record_disabled` já existe, do módulo 19) e **nenhuma** linha com `care_lines`;
- em `labTemplates.ts` e `LabTemplates.tsx`: as funções e os estilos exportados que este plano reusa; no `Shell.tsx`, no `schema.graphql` e no `README.md`: as âncoras. Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 23: a entrada com `requires: %w[feature:clinical_record]` (ou equivalente) e **sem** `environments:`; e o SDL com `CareLineTemplate` (`eligibility`/`practices` `JSON`), `careLineTemplates(key:)` e `upsertCareLineTemplate`. **Compare** com o SDL da Task 2: diferença de nome ou de tipo de campo vira ajuste no SDL e nos documentos das Tasks 2–4 **antes** de começá-las. Se o api ainda não tiver o schema, as Tasks 1–4 seguem com o SDL da Task 2 e a Task 5 confere. Se o api tiver um `environments` ou outro pré-requisito para `care_lines`, **pare e reporte** (divergência com a spec §3 e o contrato §8).

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod23 -b feat/mod-23-care-lines origin/main
cd $R/.worktrees/maintenance-mod23 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run codegen:check && echo base-ok
```

Expected: suíte verde e `base-ok`. **Anote** a base (arquivos e testes): a Task 5 espera **+3 arquivos de teste** (`careLineTemplates`, `CareLineTemplates`, `CareLineTemplateForm`) e casos novos nos existentes (`features`, `FeaturesTab`, `Shell`, `Shell.rail`).

---

### Task 1: O interruptor `care_lines` — nome e o fluxo genérico da aba

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

`src/screens/FeaturesTab.tsx` **não muda**: a aba já desenha rótulo, estado, o que falta e o botão de dois cliques para qualquer chave do catálogo, e relê a lista depois de cada `setCityFeature`. Esta task dá o nome e fixa esse comportamento para a chave nova.

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes em `features.ts`); `FeaturesTab`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (existentes no topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("care_lines")` → "Linhas de cuidado"; `missingText("clinical_record_disabled")` sem mudança.

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 23", () => {
  it("care_lines tem nome em português, o mesmo do dashboard", () => {
    expect(featureLabel("care_lines")).toBe("Linhas de cuidado");
  });

  it("o pré-requisito é o prontuário: a frase do módulo 19, sem redefinir", () => {
    expect(missingText("clinical_record_disabled")).toBe("prontuário da atenção primária (clinical_record) desligado");
    expect(missingSummary([ "clinical_record_disabled" ])).toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx`:

```tsx
describe("FeaturesTab — linhas de cuidado (módulo 23)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const CL_OFF: Feature = {
    key: "care_lines", description: "Linhas de cuidado: catálogo C2–C7, painel da equipe, busca ativa e prévia", enabled: false,
    usable: false, missing: [], changedAt: null, changedBy: null
  };
  const CL_ON: Feature = { ...CL_OFF, enabled: true, usable: true, changedAt: "2026-10-22T13:00:00Z", changedBy: "dev@local" };

  it("aparece com o nome em português", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ CL_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("care_lines")).not.toBeNull();
    expect(screen.getByText("Linhas de cuidado")).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19", async () => {
    const user = userEvent.setup();
    const CL_BLOCKED: Feature = { ...CL_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ CL_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ CL_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: CL_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar care_lines" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "care_lines", enabled: true });
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();
  });

  it("com o prontuário: liga utilizável; desliga com dois cliques", async () => {
    const user = userEvent.setup();
    let listed = [ CL_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        const on = body.variables.enabled === true;
        listed = [ on ? CL_ON : { ...CL_OFF, changedAt: "2026-10-22T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar care_lines" }));
    expect((await screen.findByRole("status")).textContent).toBe("care_lines ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar care_lines" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "care_lines", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("care_lines desligada.");
  });

  it("produção: aparece e nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ CL_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("care_lines")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("care_lines")` é `null` e a aba não acha "Linhas de cuidado". (Os testes da frase do módulo 19 e do fluxo de dois cliques já passam: fixam o que não pode mudar.)

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `FEATURE_LABELS`, troque

```ts
  procedure_billing: "Faturamento de procedimentos (BPA)"
};
```

por

```ts
  procedure_billing: "Faturamento de procedimentos (BPA)",
  // Módulo 23 (ADR 0040): exige clinical_record (missing:
  // clinical_record_disabled, já traduzido acima); todos os ambientes, sem simulado.
  care_lines: "Linhas de cuidado"
};
```

Em `README.md`, logo antes do parágrafo **Documentos clínicos (módulo 19c).** (depois do parágrafo **Faturamento de procedimentos (módulo 22).**), acrescente:

```markdown
**Linhas de cuidado (módulo 23).** O interruptor `care_lines` ("Linhas de
cuidado") liga o catálogo de linhas da cidade, as sugestões a confirmar, o
painel da equipe com a busca ativa, a prévia de C2–C7 e, no wpda, a condição
declarada e "Meu acompanhamento". Exige `clinical_record` (só cidades em modo
`record`): sem ele, a linha diz "prontuário da atenção primária
(clinical_record) desligado". Existe em todos os ambientes e não tem simulado.
A tela **Linhas de cuidado** do trilho mantém o catálogo-base das seis linhas
das notas metodológicas (C2 a C7) e publica versão nova; a versão nova só vale
para a cidade que a adotar (no dashboard), e nenhuma inscrição muda até lá.
Nenhum dado de cidadão passa por aqui. Ordem de deploy: api → dashboard e wpda
→ maintenance; contra um api sem o 23, a chave não aparece e só a tela do
catálogo-base cai.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais). Atenção às frases que este plano supõe do código atual: "ligada e utilizável", "ligada, falta pré-requisito", "desligada", "<chave> ligada e utilizável." e "<chave> desligada." (`featureStatus` e `doneText`).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the care lines switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: SDL do catálogo-base e as regras da tela (`src/lib/careLineTemplates.ts`)

**Files:**
- Modify: `schema.graphql`
- Create: `src/lib/careLineTemplates.ts`
- Test: `src/lib/careLineTemplates.test.ts`

**Interfaces:**
- Consumes: `parseDecimal`, `decimalText`, `DOT_IN_NUMBER` (`src/lib/labTemplates.ts`).
- Produces:
  - no `schema.graphql`: `CareLineTemplate` (`key`, `version`, `current`, `name`, `indicator`, `declaredCondition`, `eligibility: JSON!`, `practices: JSON!`, `createdAt`), `UpsertCareLineTemplatePayload`, `Query.careLineTemplates(key: String)`, `Mutation.upsertCareLineTemplate(key, name, indicator, declaredCondition, eligibility: JSON!, practices: JSON!)` — a forma do plano do api (Task 4 dele; M1);
  - em `src/lib/careLineTemplates.ts`: tipos `EligibilityJson`, `PracticeJson`, `TemplateView`, `PracticeDraft`, `TemplateDraft`, `TemplateVariables`; constantes `KEYS`, `INDICATORS`, `SOURCE_KINDS`, `SCHEDULES`, `ANCHORS`, `SEXES`, `CONDITIONS`, `LIMITS`; funções `keyText(key)`, `sourceText(source)`, `sourceKindText(kind)`, `scheduleText(code)`, `anchorText(code)`, `sexText(code)`, `ruleText(p)`, `advancedText(p)`, `eligibilityText(e)`, `currentTemplates(list)`, `versionsOf(list, key)`, `pointsTotal(practices)`, `emptyPractice()`, `toDraft(t)`, `firstDraft(key)`, `draftProblem(d)`, `draftToVariables(d)`, `upsertErrorText(errors)`.

As regras de uma boa prática e a elegibilidade são JSON (L1 e L2 do plano do api), com as chaves em `snake_case`. A tela edita os campos simples e **preserva sem editar** os avançados da versão de partida (`all_of`, `code_filter`, `applies_to` na prática; identidades de gênero e `declared` na elegibilidade): eles aparecem em texto e voltam iguais na versão nova.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/careLineTemplates.test.ts
import { describe, expect, it } from "vitest";
import {
  KEYS, advancedText, currentTemplates, draftProblem, draftToVariables, eligibilityText, emptyPractice, firstDraft, keyText, pointsTotal,
  ruleText, sourceText, toDraft, upsertErrorText, versionsOf, type PracticeJson, type TemplateDraft, type TemplateView
} from "./careLineTemplates";

const MD_NURSE = [ "2231", "2251", "2252", "2253", "2235" ];
const ELIG = { problems: { ciap2: [ "T89", "T90" ], cid10: [ "E10", "E11", "E14" ] }, declared: [ "diabetes" ], requires_anchor: null };
const C4_A: PracticeJson = { key: "c4_a", label: "Consulta de médico ou enfermeiro em 6 meses", citizen_label: "Consulta com a equipe",
  source: "consultation", anchor: "rolling", window_days: 183, min_count: 1, schedule: "end_of_window", cbo_filter: MD_NURSE, points: 85 };
const C4_E: PracticeJson = { key: "c4_e", label: "Hemoglobina glicada em 12 meses", citizen_label: "Exame de hemoglobina glicada",
  source: "exam:0202010503", anchor: "rolling", window_days: 365, min_count: 1, schedule: "end_of_window", cbo_filter: null, points: 15 };
const C3_G: PracticeJson = { key: "c3_g", label: "Testes do 1º trimestre", source: "procedure:0214010074", anchor: "lmp",
  window_days: 97, min_count: 1, schedule: "end_of_window", cbo_filter: null, points: 9,
  all_of: [ [ { source: "procedure:0214010074" } ], [ { source: "procedure:0214010040" } ] ] };
const t = (over: Partial<TemplateView> = {}): TemplateView => ({
  key: "diabetes_c4", version: 1, current: false, name: "Pessoa com diabetes", indicator: "C4", declaredCondition: "diabetes",
  eligibility: ELIG, practices: [ C4_A, C4_E ], createdAt: "2026-01-10T12:00:00Z", ...over
});

describe("rótulos", () => {
  it("chaves, fontes, regra, avançados e elegibilidade; desconhecido cru", () => {
    expect(KEYS).toEqual([ "child_c2", "pregnancy_c3", "diabetes_c4", "hypertension_c5", "elderly_c6", "women_c7" ]);
    expect(keyText("diabetes_c4")).toBe("C4 — Pessoa com diabetes");
    expect(keyText("saude_mental")).toBe("saude_mental");
    expect(sourceText("exam:0202010503")).toBe("exame SIGTAP 0202010503 (pedido ou resultado)");
    expect(sourceText("procedure:0301040095")).toBe("procedimento SIGTAP 0301040095");
    expect(sourceText("vaccine:57")).toBe("vacina 57 (não acompanhada pelo sistema)");
    expect(sourceText("acs_visit")).toBe("visita do ACS (não acompanhada pelo sistema)");
    expect(sourceText("oral_health")).toBe("saúde bucal (não acompanhada pelo sistema)");
    expect(sourceText("teleconsulta")).toBe("teleconsulta");
    expect(ruleText(C4_A)).toBe("consulta finalizada · 1× em 183 dias, até o fim da janela · janela móvel · CBO 2231, 2251, 2252, 2253, 2235 · 85 pontos");
    expect(ruleText({ ...C4_E, window_start_days: 196, window_days: 294, anchor: "lmp", min_count: 7, schedule: "proportional", points: 9 }))
      .toBe("exame SIGTAP 0202010503 (pedido ou resultado) · 7× do dia 196 ao 294, proporcional ao tempo · desde a DUM · 9 pontos");
    expect(advancedText(C3_G)).toBe("todas de: procedure:0214010074 | procedure:0214010040");
    expect(advancedText({ ...C4_A, code_filter: { cid10: [ "Z32" ] }, applies_to: { age_min_years: 14, age_max_years: 69 } }))
      .toBe("só com código: Z32 · vale de 14 a 69 anos");
    expect(advancedText(C4_A)).toBeNull();
    expect(eligibilityText(ELIG)).toBe("CIAP-2 T89, T90 · CID-10 E10, E11, E14 · condição declarada: diabetes");
    expect(eligibilityText({ problems: { ciap2: [], cid10: [] }, age_max_days: 730 })).toBe("até 730 dias de vida");
    expect(eligibilityText({ problems: { ciap2: [ "W78" ], cid10: [] }, exclude_problems: { ciap2: [ "W82" ], cid10: [] }, requires_anchor: "lmp" }))
      .toBe("CIAP-2 W78 · exceto CIAP-2 W82 · pede a DUM");
    expect(eligibilityText({ problems: { ciap2: [], cid10: [] }, age_min_years: 9, age_max_years: 69, sex: "female",
      gender_identities_included: [ "trans_man" ], gender_identities_excluded: [ "trans_woman" ] }))
      .toBe("de 9 a 69 anos · sexo feminino · inclui: trans_man · exclui: trans_woman");
  });

  it("vigente por chave na ordem do contrato; histórico da mais nova para a mais antiga", () => {
    const list = [ t(), t({ version: 2, current: true }), t({ key: "pregnancy_c3", indicator: "C3", current: true }),
      t({ key: "saude_mental", indicator: "", current: true }) ];
    expect(currentTemplates(list).map((x) => `${x.key}@${x.version}`)).toEqual([ "pregnancy_c3@1", "diabetes_c4@2", "saude_mental@1" ]);
    expect(versionsOf(list, "diabetes_c4").map((x) => x.version)).toEqual([ 2, 1 ]);
    expect(currentTemplates([ t(), t({ version: 3 }) ]).map((x) => x.version)).toEqual([ 3 ]);
    expect(pointsTotal([ C4_A, C4_E, { ...C4_E, points: null } ])).toBe(100);
  });
});

describe("rascunho e conferência", () => {
  const ok = (): TemplateDraft => toDraft(t({ version: 2, current: true }));

  it("rascunho a partir da vigente; primeira versão usa o nome e o indicador da chave", () => {
    const d = ok();
    expect([ d.key, d.declaredCondition, d.ciap, d.cid10, d.ageMinYears, d.requiresAnchor ]).toEqual([ "diabetes_c4", "diabetes", "T89, T90", "E10, E11, E14", "", "" ]);
    expect(d.practices[1]).toMatchObject({ key: "c4_e", citizenLabel: "Exame de hemoglobina glicada", sourceKind: "exam", code: "0202010503",
      windowStartDays: "", windowDays: "365", minCount: "1", cboFilter: "", points: "15" });
    expect(d.practices[0].cboFilter).toBe("2231, 2251, 2252, 2253, 2235");
    const first = firstDraft("child_c2");
    expect([ first.name, first.indicator, first.practices.length ]).toEqual([ "Cuidado da criança até 2 anos", "C2", 1 ]);
  });

  it("variáveis: JSON em snake_case, avançados preservados, vazios fora", () => {
    expect(draftToVariables(ok())).toEqual({
      key: "diabetes_c4", name: "Pessoa com diabetes", indicator: "C4", declaredCondition: "diabetes",
      eligibility: { problems: { ciap2: [ "T89", "T90" ], cid10: [ "E10", "E11", "E14" ] }, declared: [ "diabetes" ], requires_anchor: null },
      practices: [ C4_A, C4_E ]
    });
    const withAdvanced = toDraft(t({ practices: [ { ...C3_G, points: 100 } ] }));
    expect(draftToVariables(withAdvanced).practices[0]).toEqual({ ...C3_G, points: 100 });
  });

  it("fonte: monta e desmonta o código; desconhecida continua", () => {
    const d = ok();
    const acs = { ...d, practices: [ { ...d.practices[1], sourceKind: "acs_visit", code: "0202010503" } ] };
    expect(draftToVariables(acs).practices[0].source).toBe("acs_visit");
    const vac = { ...d, practices: [ { ...d.practices[1], sourceKind: "vaccine", code: "57" } ] };
    expect(draftToVariables(vac).practices[0].source).toBe("vaccine:57");
    const unknown = toDraft(t({ practices: [ { ...C4_A, source: "teleconsulta", points: 100 } ] }));
    expect(unknown.practices[0].sourceKind).toBe("teleconsulta");
    expect(draftToVariables(unknown).practices[0].source).toBe("teleconsulta");
  });

  it("pontos: vírgula decimal, de 0 a 100, somando 100; ponto recusado", () => {
    const withPoints = (points: string) => { const d = ok(); d.practices[0] = { ...d.practices[0], points }; return d; };
    expect(draftProblem(withPoints("12.5"))).toBe("boa prática 1: pontos — use vírgula para decimais e não use ponto de milhar");
    expect(draftProblem(withPoints("150,000"))).toBe("boa prática 1: pontos de 0 a 100");
    expect(draftProblem(withPoints("-5"))).toBe("boa prática 1: pontos de 0 a 100");
    expect(draftProblem(withPoints(""))).toBe("boa prática 1: informe os pontos (Quadro 01 da nota metodológica)");
    expect(draftProblem(withPoints("80"))).toBe("a soma dos pontos precisa ser 100 (está em 95)");
    const split = ok();
    split.practices = [ { ...split.practices[0], points: "84,5" }, { ...split.practices[1], points: "15,5" } ];
    expect(draftProblem(split)).toBeNull();
    expect(draftToVariables(split).practices.map((p) => p.points)).toEqual([ 84.5, 15.5 ]);
  });

  it("recusa o resto que o api recusaria; nunca corta", () => {
    const d = ok();
    const set = (patch: Partial<TemplateDraft>) => draftProblem({ ...d, ...patch });
    const practice = (patch: object) => draftProblem({ ...d, practices: [ { ...d.practices[0], ...patch, points: "100" } ] });
    expect(set({ name: "" })).toBe("Nome: obrigatório");
    expect(set({ name: "x".repeat(121) })).toBe("Nome: no máximo 120 caracteres");
    expect(set({ ciap: "T90, TX" })).toBe("código CIAP-2 inválido: TX");
    expect(set({ cid10: "E1" })).toBe("código CID-10 inválido: E1");
    expect(set({ ageMinYears: "60", ageMaxYears: "18" })).toBe("a idade mínima passa da máxima");
    expect(set({ ageMaxYears: "131" })).toBe("idade em anos inteiros, de 0 a 130");
    expect(set({ ageMaxDays: "abc" })).toBe("idade em dias inteiros, de 0 a 47450");
    expect(set({ practices: [] })).toBe("inclua pelo menos uma boa prática");
    expect(practice({ key: "Consulta" })).toBe("boa prática 1: chave começa com letra; minúsculas, números e _ (até 40)");
    expect(draftProblem({ ...d, practices: [ d.practices[0], { ...d.practices[1], key: "c4_a" } ] })).toBe("boa prática 2: chave repetida (c4_a)");
    expect(practice({ label: "" })).toBe("boa prática 1: escreva o que conta");
    expect(practice({ citizenLabel: "x".repeat(81) })).toBe("boa prática 1: texto para o cidadão até 80 caracteres");
    expect(practice({ sourceKind: "exam", code: "0301040095" })).toBe("boa prática 1: exame com SIGTAP do grupo 02 (10 dígitos)");
    expect(practice({ sourceKind: "procedure", code: "03010400" })).toBe("boa prática 1: procedimento com SIGTAP de 10 dígitos");
    expect(practice({ sourceKind: "vaccine", code: "" })).toBe("boa prática 1: código da vacina (minúsculas, números e _, até 30)");
    expect(practice({ windowDays: "0" })).toBe("boa prática 1: janela de 1 a 7300 dias");
    expect(practice({ windowStartDays: "200", windowDays: "180" })).toBe("boa prática 1: o início da janela passa do fim");
    expect(practice({ minCount: "51" })).toBe("boa prática 1: quantidade mínima de 1 a 50");
    expect(practice({ schedule: "proportional", anchor: "rolling" })).toBe("boa prática 1: cronograma proporcional só com âncora fixa");
    expect(practice({ cboFilter: "225, 22A" })).toBe("boa prática 1: CBO por prefixo de 1 a 6 dígitos (22A)");
    expect(draftProblem(d)).toBeNull();
    expect(emptyPractice().points).toBe("");
  });

  it("frases das recusas do api, com o campo; desconhecido cru", () => {
    expect(upsertErrorText([ { path: "practices[1].window_days", message: "invalid_practices" } ]))
      .toBe("boas práticas inválidas — confira os campos (practices[1].window_days)");
    expect(upsertErrorText([ { path: "indicator", message: "invalid_indicator" } ])).toBe("indicador desconhecido — use C2 a C7");
    expect(upsertErrorText([ { path: null, message: "brand_new" } ])).toBe("brand_new");
    expect(upsertErrorText([])).toBe("não foi possível concluir — tente de novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/careLineTemplates.test.ts`
Expected: FAIL — `./careLineTemplates` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `schema.graphql`:

1. Logo antes da linha `type LabExamTemplate {` (e do bloco `"""…"""` que a descreve), acrescente

```graphql
"""
Versão do catálogo-base de uma linha de cuidado (C2–C7). Imutável; a cidade adota uma versão.
"""
type CareLineTemplate {
  createdAt: ISO8601DateTime!

  """
  Maior versão da chave
  """
  current: Boolean!
  declaredCondition: String
  eligibility: JSON!
  indicator: String!
  key: String!
  name: String!
  practices: JSON!
  version: Int!
}

```

2. Em `type Mutation`, logo antes da linha que começa com `  upsertLabAnalyteTemplate(`, acrescente

```graphql
  upsertCareLineTemplate(declaredCondition: String, eligibility: JSON!, indicator: String!, key: String!, name: String!, practices: JSON!): UpsertCareLineTemplatePayload
```

3. Em `type Query`, logo antes da linha `  cities(status: CityStatus): [CitySummary!]!`, acrescente

```graphql
  careLineTemplates(key: String): [CareLineTemplate!]!
```

4. Logo antes da linha `type UpsertLabAnalyteTemplatePayload {` (e do bloco `"""…"""` que a descreve), acrescente

```graphql
"""
Autogenerated return type of UpsertCareLineTemplate.
"""
type UpsertCareLineTemplatePayload {
  errors: [UserError!]!
  ok: Boolean!
}

```

(A Task 5 troca este arquivo pelo `to_definition` do api e confere que o codegen não muda — a ordem e as descrições podem mudar; os nomes e tipos dos campos, não.)

```ts
// src/lib/careLineTemplates.ts
// Regras do catálogo-base de linhas de cuidado da plataforma (módulo 23,
// F-23.1; spec §4 "Plataforma"; contrato §1 e §8; L1 e L2 do plano do api).
// Elegibilidade e boas práticas são JSON (snake_case). A tela edita os campos
// simples e PRESERVA os avançados (all_of, code_filter, applies_to; identidades
// de gênero e `declared`) da versão de partida. Quem valida é o api; aqui só se
// recusa o que ele recusaria por formato — nunca cortar. Valor desconhecido
// (api mais novo) aparece cru e continua selecionado.
import { DOT_IN_NUMBER, decimalText, parseDecimal } from "./labTemplates";

type Codes = { ciap2: string[]; cid10: string[] };
export type EligibilityJson = {
  problems: Codes; exclude_problems?: Codes; age_min_days?: number | null; age_max_days?: number | null;
  age_min_years?: number | null; age_max_years?: number | null; sex?: string | null;
  gender_identities_included?: string[]; gender_identities_excluded?: string[]; declared?: string[]; requires_anchor?: string | null;
};
export type PracticeJson = {
  key: string; label: string; citizen_label?: string | null; source: string; anchor: string; window_start_days?: number | null;
  window_days: number; min_count: number; schedule: string; cbo_filter?: string[] | null; points?: number | null;
  code_filter?: { ciap2?: string[]; cid10?: string[] } | null;
  applies_to?: { age_min_years?: number; age_max_years?: number; sex?: string } | null;
  all_of?: { source: string; window_days?: number }[][] | null;
};
export type TemplateView = {
  key: string; version: number; current: boolean; name: string; indicator: string; declaredCondition?: string | null;
  eligibility: EligibilityJson; practices: PracticeJson[]; createdAt: string;
};

export const KEYS = [ "child_c2", "pregnancy_c3", "diabetes_c4", "hypertension_c5", "elderly_c6", "women_c7" ] as const;
const KEY_INFO: Record<string, { indicator: string; name: string }> = {
  child_c2: { indicator: "C2", name: "Cuidado da criança até 2 anos" },
  pregnancy_c3: { indicator: "C3", name: "Gestante e puérpera" },
  diabetes_c4: { indicator: "C4", name: "Pessoa com diabetes" },
  hypertension_c5: { indicator: "C5", name: "Pessoa com hipertensão" },
  elderly_c6: { indicator: "C6", name: "Pessoa idosa" },
  women_c7: { indicator: "C7", name: "Mulher e prevenção do câncer" }
};
export const INDICATORS = [ "C2", "C3", "C4", "C5", "C6", "C7" ] as const;
export function keyText(key: string): string {
  const info = KEY_INFO[key];
  return info ? `${info.indicator} — ${info.name}` : key;
}

export const CONDITIONS = [ "", "diabetes", "hypertension", "pregnancy", "mental_health" ] as const;

export const SOURCE_KINDS = [ "consultation", "blood_pressure", "weight_height", "exam", "procedure", "acs_visit", "vaccine", "oral_health" ] as const;
const SOURCE_KIND_LABELS: Record<string, string> = {
  consultation: "consulta finalizada", blood_pressure: "pressão arterial (escuta ou consulta)", weight_height: "peso e altura (escuta ou consulta)",
  exam: "exame (SIGTAP)", procedure: "procedimento (SIGTAP)", acs_visit: "visita do ACS (não acompanhada pelo sistema)",
  vaccine: "vacina (não acompanhada pelo sistema)", oral_health: "saúde bucal (não acompanhada pelo sistema)"
};
export function sourceKindText(kind: string): string { return SOURCE_KIND_LABELS[kind] ?? kind; }
const CODED = new Set([ "exam", "procedure", "vaccine" ]);

export function sourceText(source: string): string {
  const [ kind, code ] = source.split(":");
  if (kind === "exam" && code) return `exame SIGTAP ${code} (pedido ou resultado)`;
  if (kind === "procedure" && code) return `procedimento SIGTAP ${code}`;
  if (kind === "vaccine" && code) return `vacina ${code} (não acompanhada pelo sistema)`;
  return code === undefined && !CODED.has(kind) && SOURCE_KIND_LABELS[kind] ? SOURCE_KIND_LABELS[kind] : source;
}

export const SCHEDULES = [ "end_of_window", "proportional" ] as const;
const SCHEDULE_LABELS: Record<string, string> = { end_of_window: "até o fim da janela", proportional: "proporcional ao tempo" };
export function scheduleText(code: string): string { return SCHEDULE_LABELS[code] ?? code; }

export const ANCHORS = [ "rolling", "enrollment", "birth", "lmp" ] as const;
const ANCHOR_LABELS: Record<string, string> = { rolling: "janela móvel", enrollment: "desde a inscrição", birth: "desde o nascimento", lmp: "desde a DUM" };
export function anchorText(code: string): string { return ANCHOR_LABELS[code] ?? code; }

export const SEXES = [ "", "female", "male" ] as const;
const SEX_LABELS: Record<string, string> = { "": "qualquer", female: "feminino", male: "masculino" };
export function sexText(code: string | null | undefined): string { return SEX_LABELS[code ?? ""] ?? String(code); }

export function ruleText(p: PracticeJson): string {
  const span = p.window_start_days ? `do dia ${p.window_start_days} ao ${p.window_days}` : `em ${p.window_days} dias`;
  const parts = [ sourceText(p.source), `${p.min_count}× ${span}, ${scheduleText(p.schedule)}`, anchorText(p.anchor) ];
  if (p.cbo_filter && p.cbo_filter.length > 0) parts.push(`CBO ${p.cbo_filter.join(", ")}`);
  if (typeof p.points === "number") parts.push(`${decimalText(p.points)} pontos`);
  return parts.join(" · ");
}

// Os campos que a tela preserva sem editar, em texto (null se não há).
export function advancedText(p: PracticeJson): string | null {
  const parts: string[] = [];
  if (p.all_of && p.all_of.length > 0) parts.push(`todas de: ${p.all_of.map((g) => g.map((a) => a.source).join(" ou ")).join(" | ")}`);
  const codes = [ ...(p.code_filter?.ciap2 ?? []), ...(p.code_filter?.cid10 ?? []) ];
  if (codes.length > 0) parts.push(`só com código: ${codes.join(", ")}`);
  const a = p.applies_to;
  if (a && (a.age_min_years != null || a.age_max_years != null)) parts.push(`vale de ${a.age_min_years ?? 0} a ${a.age_max_years ?? "—"} anos`);
  if (a?.sex) parts.push(`só sexo ${sexText(a.sex)}`);
  return parts.length > 0 ? parts.join(" · ") : null;
}

const range = (min: number | null | undefined, max: number | null | undefined, unit: string, maxOnly: string) =>
  min != null && max != null ? `de ${min} a ${max} ${unit}` : min != null ? `a partir de ${min} ${unit}` : max != null ? maxOnly : null;

export function eligibilityText(e: EligibilityJson): string {
  const parts: string[] = [];
  if (e.problems.ciap2.length > 0) parts.push(`CIAP-2 ${e.problems.ciap2.join(", ")}`);
  if (e.problems.cid10.length > 0) parts.push(`CID-10 ${e.problems.cid10.join(", ")}`);
  const ex = e.exclude_problems;
  if (ex && ex.ciap2.length > 0) parts.push(`exceto CIAP-2 ${ex.ciap2.join(", ")}`);
  if (ex && ex.cid10.length > 0) parts.push(`exceto CID-10 ${ex.cid10.join(", ")}`);
  const days = range(e.age_min_days, e.age_max_days, "dias de vida", `até ${e.age_max_days} dias de vida`);
  if (days) parts.push(days);
  const years = range(e.age_min_years, e.age_max_years, "anos", `até ${e.age_max_years} anos`);
  if (years) parts.push(years);
  if (e.sex) parts.push(`sexo ${sexText(e.sex)}`);
  if (e.gender_identities_included?.length) parts.push(`inclui: ${e.gender_identities_included.join(", ")}`);
  if (e.gender_identities_excluded?.length) parts.push(`exclui: ${e.gender_identities_excluded.join(", ")}`);
  if (e.declared?.length) parts.push(`condição declarada: ${e.declared.join(", ")}`);
  if (e.requires_anchor === "lmp") parts.push("pede a DUM");
  return parts.length > 0 ? parts.join(" · ") : "sem regra (só inscrição manual)";
}

// Vigente de cada chave: a marcada `current`, ou a de maior versão; chaves do
// contrato na ordem dele, depois as desconhecidas.
export function currentTemplates(list: TemplateView[]): TemplateView[] {
  const byKey = new Map<string, TemplateView>();
  for (const t of list) {
    const seen = byKey.get(t.key);
    if (!seen || (t.current && !seen.current) || (t.current === seen.current && t.version > seen.version)) byKey.set(t.key, t);
  }
  return [ ...byKey.values() ].sort((a, b) => keyOrder(a.key) - keyOrder(b.key) || a.key.localeCompare(b.key));
}
export function keyOrder(key: string): number {
  const i = (KEYS as readonly string[]).indexOf(key);
  return i === -1 ? KEYS.length : i;
}

export function versionsOf(list: TemplateView[], key: string): TemplateView[] {
  return list.filter((t) => t.key === key).sort((a, b) => b.version - a.version);
}

export function pointsTotal(practices: { points?: number | null }[]): number {
  return practices.reduce((sum, p) => sum + (typeof p.points === "number" ? p.points : 0), 0);
}

// ─── Rascunho da versão nova ─────────────────────────────────────────────
type Advanced = Pick<PracticeJson, "code_filter" | "applies_to" | "all_of">;
export type PracticeDraft = {
  key: string; label: string; citizenLabel: string; sourceKind: string; code: string; windowStartDays: string; windowDays: string;
  minCount: string; schedule: string; anchor: string; cboFilter: string; points: string; advanced: Advanced;
};
export type TemplateDraft = {
  key: string; name: string; indicator: string; declaredCondition: string; ciap: string; cid10: string; excludeCiap: string;
  excludeCid10: string; ageMinYears: string; ageMaxYears: string; ageMinDays: string; ageMaxDays: string; sex: string;
  requiresAnchor: string; preserved: Pick<EligibilityJson, "gender_identities_included" | "gender_identities_excluded" | "declared">;
  practices: PracticeDraft[];
};
export type TemplateVariables = {
  key: string; name: string; indicator: string; declaredCondition: string | null; eligibility: EligibilityJson; practices: PracticeJson[];
};

export const LIMITS = { name: 120, label: 120, citizenLabel: 80, practices: 20, windowDays: 7300, minCount: 50, ageYears: 130, ageDays: 47450 } as const;

export function emptyPractice(): PracticeDraft {
  return { key: "", label: "", citizenLabel: "", sourceKind: "consultation", code: "", windowStartDays: "", windowDays: "183", minCount: "1",
    schedule: "end_of_window", anchor: "rolling", cboFilter: "", points: "", advanced: {} };
}

const numText = (n: number | null | undefined) => (n == null ? "" : String(n));

function practiceDraft(p: PracticeJson): PracticeDraft {
  const [ kind, code ] = p.source.split(":");
  const advanced: Advanced = {};
  if (p.code_filter) advanced.code_filter = p.code_filter;
  if (p.applies_to) advanced.applies_to = p.applies_to;
  if (p.all_of) advanced.all_of = p.all_of;
  return {
    key: p.key, label: p.label, citizenLabel: p.citizen_label ?? "", sourceKind: kind, code: code ?? "",
    windowStartDays: p.window_start_days ? String(p.window_start_days) : "", windowDays: String(p.window_days), minCount: String(p.min_count),
    schedule: p.schedule, anchor: p.anchor, cboFilter: (p.cbo_filter ?? []).join(", "),
    points: typeof p.points === "number" ? decimalText(p.points) : "", advanced
  };
}

export function toDraft(t: TemplateView): TemplateDraft {
  const e = t.eligibility;
  const preserved: TemplateDraft["preserved"] = {};
  if (e.gender_identities_included) preserved.gender_identities_included = e.gender_identities_included;
  if (e.gender_identities_excluded) preserved.gender_identities_excluded = e.gender_identities_excluded;
  if (e.declared) preserved.declared = e.declared;
  return {
    key: t.key, name: t.name, indicator: t.indicator, declaredCondition: t.declaredCondition ?? "",
    ciap: e.problems.ciap2.join(", "), cid10: e.problems.cid10.join(", "),
    excludeCiap: (e.exclude_problems?.ciap2 ?? []).join(", "), excludeCid10: (e.exclude_problems?.cid10 ?? []).join(", "),
    ageMinYears: numText(e.age_min_years), ageMaxYears: numText(e.age_max_years), ageMinDays: numText(e.age_min_days),
    ageMaxDays: numText(e.age_max_days), sex: e.sex ?? "", requiresAnchor: e.requires_anchor ?? "", preserved,
    practices: t.practices.map(practiceDraft)
  };
}

export function firstDraft(key: string): TemplateDraft {
  const info = KEY_INFO[key] ?? { indicator: "", name: "" };
  return { key, name: info.name, indicator: info.indicator, declaredCondition: "", ciap: "", cid10: "", excludeCiap: "", excludeCid10: "",
    ageMinYears: "", ageMaxYears: "", ageMinDays: "", ageMaxDays: "", sex: "", requiresAnchor: "", preserved: {},
    practices: [ emptyPractice() ] };
}

const codes = (text: string) => [ ...new Set(text.toUpperCase().replace(/\./g, "").split(/[\s,;]+/).filter(Boolean)) ];
const list = (text: string) => [ ...new Set(text.split(/[\s,;]+/).filter(Boolean)) ];
const CIAP_RE = /^[A-Z]\d{2}$/;
const CID_RE = /^[A-Z]\d{2}[0-9X]?$/;
const EXAM_RE = /^02\d{8}$/;
const PROCEDURE_RE = /^\d{10}$/;
const VACCINE_RE = /^[a-z0-9_]{1,30}$/;
const CBO_RE = /^\d{1,6}$/;
const KEY_RE = /^[a-z][a-z0-9_]{0,39}$/;

function intOf(raw: string): number | null {
  const t = raw.trim();
  return /^\d+$/.test(t) ? Number(t) : null;
}
// "" = sem limite; inválido = undefined.
function boundOf(raw: string, max: number): number | null | undefined {
  if (raw.trim() === "") return null;
  const n = intOf(raw);
  return n === null || n > max ? undefined : n;
}

function codesProblem(text: string, re: RegExp, label: string): string | null {
  const bad = codes(text).find((c) => !re.test(c));
  return bad ? `código ${label} inválido: ${bad}` : null;
}

export function draftProblem(d: TemplateDraft): string | null {
  const name = d.name.trim();
  if (!name) return "Nome: obrigatório";
  if (name.length > LIMITS.name) return `Nome: no máximo ${LIMITS.name} caracteres`;
  const codeProblem = codesProblem(d.ciap, CIAP_RE, "CIAP-2") ?? codesProblem(d.cid10, CID_RE, "CID-10")
    ?? codesProblem(d.excludeCiap, CIAP_RE, "CIAP-2") ?? codesProblem(d.excludeCid10, CID_RE, "CID-10");
  if (codeProblem) return codeProblem;
  const minY = boundOf(d.ageMinYears, LIMITS.ageYears);
  const maxY = boundOf(d.ageMaxYears, LIMITS.ageYears);
  if (minY === undefined || maxY === undefined) return `idade em anos inteiros, de 0 a ${LIMITS.ageYears}`;
  if (minY !== null && maxY !== null && minY > maxY) return "a idade mínima passa da máxima";
  const minD = boundOf(d.ageMinDays, LIMITS.ageDays);
  const maxD = boundOf(d.ageMaxDays, LIMITS.ageDays);
  if (minD === undefined || maxD === undefined) return `idade em dias inteiros, de 0 a ${LIMITS.ageDays}`;
  if (minD !== null && maxD !== null && minD > maxD) return "a idade mínima passa da máxima";
  if (d.practices.length === 0) return "inclua pelo menos uma boa prática";
  if (d.practices.length > LIMITS.practices) return `no máximo ${LIMITS.practices} boas práticas`;
  const keys = new Set<string>();
  for (const [ i, p ] of d.practices.entries()) {
    const n = `boa prática ${i + 1}`;
    const key = p.key.trim();
    if (!KEY_RE.test(key)) return `${n}: chave começa com letra; minúsculas, números e _ (até 40)`;
    if (keys.has(key)) return `${n}: chave repetida (${key})`;
    keys.add(key);
    const label = p.label.trim();
    if (!label) return `${n}: escreva o que conta`;
    if (label.length > LIMITS.label) return `${n}: até ${LIMITS.label} caracteres`;
    if (p.citizenLabel.trim().length > LIMITS.citizenLabel) return `${n}: texto para o cidadão até ${LIMITS.citizenLabel} caracteres`;
    const code = p.code.trim();
    if (p.sourceKind === "exam" && !EXAM_RE.test(code)) return `${n}: exame com SIGTAP do grupo 02 (10 dígitos)`;
    if (p.sourceKind === "procedure" && !PROCEDURE_RE.test(code)) return `${n}: procedimento com SIGTAP de 10 dígitos`;
    if (p.sourceKind === "vaccine" && !VACCINE_RE.test(code)) return `${n}: código da vacina (minúsculas, números e _, até 30)`;
    const w = intOf(p.windowDays);
    if (w === null || w < 1 || w > LIMITS.windowDays) return `${n}: janela de 1 a ${LIMITS.windowDays} dias`;
    const start = p.windowStartDays.trim() === "" ? 0 : intOf(p.windowStartDays);
    if (start === null) return `${n}: início da janela em dias inteiros`;
    if (start >= w) return `${n}: o início da janela passa do fim`;
    const c = intOf(p.minCount);
    if (c === null || c < 1 || c > LIMITS.minCount) return `${n}: quantidade mínima de 1 a ${LIMITS.minCount}`;
    if (p.schedule === "proportional" && p.anchor === "rolling") return `${n}: cronograma proporcional só com âncora fixa`;
    const badCbo = list(p.cboFilter).find((x) => !CBO_RE.test(x));
    if (badCbo) return `${n}: CBO por prefixo de 1 a 6 dígitos (${badCbo})`;
    if (p.points.trim() === "") return `${n}: informe os pontos (Quadro 01 da nota metodológica)`;
    if (p.points.includes(".")) return `${n}: pontos — ${DOT_IN_NUMBER}`;
    const points = parseDecimal(p.points);
    if (points === undefined || points === null || points < 0 || points > 100) return `${n}: pontos de 0 a 100`;
  }
  const total = pointsTotal(d.practices.map((p) => ({ points: parseDecimal(p.points) ?? null })));
  if (Math.abs(total - 100) > 1e-9) return `a soma dos pontos precisa ser 100 (está em ${decimalText(total)})`;
  return null;
}

// Só depois de draftProblem(d) === null. Campo vazio não vai; os avançados voltam iguais.
export function draftToVariables(d: TemplateDraft): TemplateVariables {
  const eligibility: EligibilityJson = { problems: { ciap2: codes(d.ciap), cid10: codes(d.cid10) } };
  const exCiap = codes(d.excludeCiap);
  const exCid = codes(d.excludeCid10);
  if (exCiap.length > 0 || exCid.length > 0) eligibility.exclude_problems = { ciap2: exCiap, cid10: exCid };
  const bounds: [ keyof EligibilityJson, string ][] = [
    [ "age_min_years", d.ageMinYears ], [ "age_max_years", d.ageMaxYears ], [ "age_min_days", d.ageMinDays ], [ "age_max_days", d.ageMaxDays ]
  ];
  for (const [ field, raw ] of bounds) if (raw.trim() !== "") (eligibility as unknown as Record<string, unknown>)[field] = Number(raw.trim());
  if (d.sex) eligibility.sex = d.sex;
  Object.assign(eligibility, d.preserved);
  eligibility.requires_anchor = d.requiresAnchor || null;
  return {
    key: d.key, name: d.name.trim(), indicator: d.indicator, declaredCondition: d.declaredCondition || null, eligibility,
    practices: d.practices.map((p) => {
      const out: PracticeJson = {
        key: p.key.trim(), label: p.label.trim(),
        ...(p.citizenLabel.trim() ? { citizen_label: p.citizenLabel.trim() } : {}),
        source: CODED.has(p.sourceKind) ? `${p.sourceKind}:${p.code.trim()}` : p.sourceKind, anchor: p.anchor,
        ...(p.windowStartDays.trim() !== "" && Number(p.windowStartDays) > 0 ? { window_start_days: Number(p.windowStartDays.trim()) } : {}),
        window_days: Number(p.windowDays.trim()), min_count: Number(p.minCount.trim()), schedule: p.schedule,
        cbo_filter: list(p.cboFilter).length > 0 ? list(p.cboFilter) : null, points: parseDecimal(p.points) ?? 0,
        ...p.advanced
      };
      return out;
    })
  };
}

// Códigos de UserError da mutation (Task 4 do plano do api: `message` = código,
// `path` = campo); desconhecido cru.
const UPSERT_ERRORS: Record<string, string> = {
  invalid_key: "linha desconhecida — atualize a tela",
  invalid_indicator: "indicador desconhecido — use C2 a C7",
  invalid_name: "nome inválido — de 1 a 120 caracteres",
  invalid_declared_condition: "condição declarada desconhecida",
  invalid_practices: "boas práticas inválidas — confira os campos",
  invalid_eligibility: "elegibilidade inválida — confira os códigos e as idades"
};
const NAMED_PATHS = new Set([ "key", "indicator", "name", "declaredCondition" ]);

export function upsertErrorText(errors: { path?: string | null; message: string }[]): string {
  if (errors.length === 0) return "não foi possível concluir — tente de novo";
  return errors.map((e) => {
    const phrase = UPSERT_ERRORS[e.message] ?? e.message;
    return e.path && UPSERT_ERRORS[e.message] && !NAMED_PATHS.has(e.path) ? `${phrase} (${e.path})` : phrase;
  }).join("; ");
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/careLineTemplates.test.ts && npx tsc --noEmit && npm run codegen && $G status --short src/gql`
Expected: PASS; o `codegen` roda sem erro (ainda não há documento novo; `src/gql` pode mudar só pelos tipos do schema — se mudar, entra no commit desta task).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add schema.graphql src/lib/careLineTemplates.ts src/lib/careLineTemplates.test.ts
$G status --short src/gql | grep -q . && $G add src/gql/gql.ts src/gql/graphql.ts src/gql/index.ts
$G commit -m "feat: add the care line template schema and screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: A tela no trilho — versões vigentes e histórico

**Files:**
- Create: `src/screens/CareLineTemplates.tsx`, `src/screens/CareLineTemplates.fixtures.tsx`
- Modify: `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx`, `src/gql/*` (codegen)
- Test: `src/screens/CareLineTemplates.test.tsx`

**Interfaces:**
- Consumes: `gql` (`src/lib/api.ts`); `graphql` (`src/gql`); `GraphQLRefusal` (`src/lib/errors.ts`); `KEYS`, `advancedText`, `currentTemplates`, `eligibilityText`, `keyOrder`, `keyText`, `pointsTotal`, `ruleText`, `versionsOf`, `TemplateView` (Task 2); `decimalText` (`labTemplates.ts`); `fmtWhen` (`src/lib/analytics.ts`); `muted`, `row` (`LabTemplates.tsx`); `Panel`, `Button`, `Tag`, `DataTable`, `ErrorState`.
- Produces: `CareLineTemplates()`; `CARE_LINE_TEMPLATES_QUERY_KEY = [ "platform", "careLineTemplates" ]`; `requestErrorText(err)`; os dados de teste `reply`, `renderScreen`, `ELIG`, `C4_A`, `C4_E`, `C4_V1`, `C4_V2`, `C3_V1` em `CareLineTemplates.fixtures.tsx`; o item "Linhas de cuidado" (`key: "careLines"`) no trilho.

- [ ] **Step 1: Write the failing test**

```tsx
// src/screens/CareLineTemplates.fixtures.tsx
// Dados e harness comuns aos testes do catálogo-base de linhas (módulo 23).
// Arquivo sem testes: os dois arquivos de teste importam daqui (importar um
// arquivo .test registraria os testes dele duas vezes). Elegibilidade e boas
// práticas em JSON snake_case (L1, L2 do plano do api).
import { render } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { CareLineTemplates } from "./CareLineTemplates";

export function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
const MD_NURSE = [ "2231", "2251", "2252", "2253", "2235" ];
export const ELIG = { problems: { ciap2: [ "T89", "T90" ], cid10: [ "E10", "E11", "E14" ] }, declared: [ "diabetes" ], requires_anchor: null };
export const C4_A = { key: "c4_a", label: "Consulta de médico ou enfermeiro em 6 meses", citizen_label: "Consulta com a equipe", source: "consultation",
  anchor: "rolling", window_days: 183, min_count: 1, schedule: "end_of_window", cbo_filter: MD_NURSE, points: 85 };
export const C4_E = { key: "c4_e", label: "Hemoglobina glicada em 12 meses", citizen_label: "Exame de hemoglobina glicada", source: "exam:0202010503",
  anchor: "rolling", window_days: 365, min_count: 1, schedule: "end_of_window", cbo_filter: null, points: 15 };
export const C4_V1 = { key: "diabetes_c4", version: 1, current: false, name: "Pessoa com diabetes", indicator: "C4", declaredCondition: "diabetes",
  eligibility: ELIG, practices: [ { ...C4_A, points: 100 } ], createdAt: "2026-01-10T12:00:00Z" };
export const C4_V2 = { ...C4_V1, version: 2, current: true, practices: [ C4_A, C4_E ], createdAt: "2026-10-20T13:00:00Z" };
export const C3_V1 = { key: "pregnancy_c3", version: 1, current: true, name: "Gestante e puérpera", indicator: "C3", declaredCondition: "pregnancy",
  eligibility: { problems: { ciap2: [ "W78" ], cid10: [ "Z34" ] }, exclude_problems: { ciap2: [ "W82" ], cid10: [] }, requires_anchor: "lmp" },
  practices: [ { key: "c3_g", label: "Testes do 1º trimestre", source: "procedure:0214010074", anchor: "lmp", window_days: 97, min_count: 1,
    schedule: "end_of_window", cbo_filter: null, points: 100,
    all_of: [ [ { source: "procedure:0214010074" } ], [ { source: "procedure:0214010040" } ] ] } ],
  createdAt: "2026-01-10T12:00:00Z" };

export function renderScreen() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<CareLineTemplates />, { wrapper });
}
```

```tsx
// src/screens/CareLineTemplates.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { C3_V1, C4_V1, C4_V2, renderScreen, reply } from "./CareLineTemplates.fixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

describe("CareLineTemplates", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });

  const listing = (list: unknown[]) => fetchMock.mockImplementation(() =>
    Promise.resolve(reply(200, { data: { careLineTemplates: list } })));
  const rowOf = (text: string) => screen.getByRole("row", { name: new RegExp(text) });

  it("uma linha por chave, a versão vigente, a soma dos pontos e as chaves sem versão", async () => {
    listing([ C4_V2, C4_V1, C3_V1 ]);
    renderScreen();
    expect(await screen.findByText("C4 — Pessoa com diabetes")).not.toBeNull();
    const c4 = rowOf("C4 — Pessoa com diabetes");
    expect(within(c4).getByText("v2")).not.toBeNull();
    expect(within(c4).getByText("20/10/2026, 10:00")).not.toBeNull();
    expect(within(c4).getByText("100")).not.toBeNull();
    expect(screen.getAllByRole("row", { name: /C4 — Pessoa com diabetes/ })).toHaveLength(1);
    const c2 = rowOf("C2 — Cuidado da criança até 2 anos");
    expect(within(c2).getByText("sem versão")).not.toBeNull();
    expect(within(c2).getByRole("button", { name: "primeira versão" })).not.toBeNull();
    expect(screen.getByText(/só vale para a cidade que a adotar/)).not.toBeNull();
  });

  it("versões: o histórico da chave, da mais nova para a mais antiga, com a regra e os avançados", async () => {
    const user = userEvent.setup();
    listing([ C4_V2, C4_V1, C3_V1 ]);
    renderScreen();
    await user.click(await within(await screen.findByRole("row", { name: /C4 — Pessoa com diabetes/ })).findByRole("button", { name: "versões" }));
    const history = screen.getByRole("region", { name: "Versões de C4 — Pessoa com diabetes" });
    const versions = within(history).getAllByRole("listitem").filter((li) => li.getAttribute("data-version"));
    expect(versions.map((li) => li.getAttribute("data-version"))).toEqual([ "2", "1" ]);
    expect(within(versions[0]).getByText("vigente")).not.toBeNull();
    expect(within(history).getAllByText("CIAP-2 T89, T90 · CID-10 E10, E11, E14 · condição declarada: diabetes")).toHaveLength(2);
    expect(within(versions[0]).getByText(
      "Hemoglobina glicada em 12 meses: exame SIGTAP 0202010503 (pedido ou resultado) · 1× em 365 dias, até o fim da janela · janela móvel · 15 pontos"
    )).not.toBeNull();
    await user.click(within(screen.getByRole("row", { name: /C3 — Gestante e puérpera/ })).getByRole("button", { name: "versões" }));
    const c3 = screen.getByRole("region", { name: "Versões de C3 — Gestante e puérpera" });
    expect(within(c3).getByText("regras avançadas (preservadas na versão nova): todas de: procedure:0214010074 | procedure:0214010040")).not.toBeNull();
  });

  it("catálogo vazio: as seis chaves sem versão", async () => {
    listing([]);
    renderScreen();
    expect(await screen.findByText("C7 — Mulher e prevenção do câncer")).not.toBeNull();
    expect(screen.getAllByText("sem versão")).toHaveLength(6);
  });

  it("api sem o 23: só esta tela cai, com a frase", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      data: null, errors: [ { message: "Field 'careLineTemplates' doesn't exist on type 'Query'", extensions: { code: "undefinedField" } } ]
    })));
    renderScreen();
    expect((await screen.findByRole("alert")).textContent)
      .toBe("esta API ainda não tem o catálogo de linhas de cuidado — o api do módulo 23 precisa subir antes do maintenance");
  });
});
```

Em `src/screens/Shell.test.tsx` e em `src/screens/Shell.rail.test.tsx`, troque (nos dois arquivos)

```ts
    for (const label of [ "Cidades", "Medicamentos", "Laboratório", "Mantenedores", "Tokens", "Auditoria" ]) {
```

por

```ts
    for (const label of [ "Cidades", "Medicamentos", "Laboratório", "Linhas de cuidado", "Mantenedores", "Tokens", "Auditoria" ]) {
```

e, em `Shell.rail.test.tsx`, o nome do teste `"trilho 'Telas' com os 6 botões de nome exato, ícone escondido, Cidades ativo"` por `"trilho 'Telas' com os 7 botões de nome exato, ícone escondido, Cidades ativo"`; em `Shell.test.tsx`, `"mostra o e-mail do mantenedor, a navegação com as seis telas e 'sair' encerra a sessão"` por `"mostra o e-mail do mantenedor, a navegação com as sete telas e 'sair' encerra a sessão"`. No fim do `describe` de `Shell.test.tsx` (depois do teste "Laboratório abre o catálogo-base de exames"), acrescente

```tsx
  it("Linhas de cuidado abre o catálogo-base de linhas", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const path = String(url);
      if (path === "/session") return Promise.resolve(reply(200, { id: "m1", email_address: "mantenedor@rotasaude.app", expires_at: "2026-01-01T01:00:00Z" }));
      if (path === "/graphql") {
        const body = JSON.parse((init as RequestInit).body as string);
        if (String(body.query).includes("query CareLineTemplates")) return Promise.resolve(reply(200, { data: { careLineTemplates: [] } }));
        return Promise.resolve(reply(200, { data: { cities: [] } }));
      }
      return Promise.resolve(reply(204, undefined));
    });
    renderShell();

    await user.click(await screen.findByRole("button", { name: "Linhas de cuidado" }));
    expect(screen.getByRole("button", { name: "Linhas de cuidado" }).getAttribute("aria-current")).toBe("page");
    expect(await screen.findByText("C2 — Cuidado da criança até 2 anos")).not.toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/screens/CareLineTemplates.test.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx`
Expected: FAIL — `./CareLineTemplates` não existe; o trilho não tem "Linhas de cuidado".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/screens/CareLineTemplates.tsx
// Catálogo-base de linhas de cuidado da plataforma (módulo 23, F-23.1; spec
// §4 "Plataforma" e §6 "Maintenance"; contrato §8; plano do api, Task 4). As
// seis linhas das notas metodológicas do componente de qualidade da APS (C2 a
// C7), versionadas e imutáveis. Vale para TODAS as cidades do ambiente, mas
// publicar uma versão nova não muda nenhuma: cada cidade adota no dashboard, e
// as inscrições só mudam de regra quando ela adota. Sem step-up (como o
// catálogo de laboratório, 21b M4); a auditoria é
// `maintenance.care_line_template.changed` (L13). Consulta PRÓPRIA: contra um
// api sem o 23, só esta tela cai.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import { fmtWhen } from "../lib/analytics";
import { decimalText } from "../lib/labTemplates";
import {
  KEYS, advancedText, currentTemplates, eligibilityText, keyOrder, keyText, pointsTotal, ruleText, versionsOf, type TemplateView
} from "../lib/careLineTemplates";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { ErrorState } from "../components/ErrorState";
import { muted, row } from "./LabTemplates";

export const CareLineTemplatesQuery = graphql(`
  query CareLineTemplates {
    careLineTemplates { key version current name indicator declaredCondition eligibility practices createdAt }
  }
`);

const TITLE = "Catálogo-base de linhas de cuidado (todas as cidades do ambiente)";
const INTRO =
  "As seis linhas das notas metodológicas do componente de qualidade da APS (C2 a C7). Publicar uma versão nova não muda nenhuma " +
  "cidade: a versão nova só vale para a cidade que a adotar, e as inscrições só passam a ser avaliadas por ela depois disso. " +
  "Confira a nota técnica vigente antes de publicar (revisão anual).";
export const CARE_LINE_TEMPLATES_QUERY_KEY = [ "platform", "careLineTemplates" ];
const OLD_API_CODES = new Set([ "undefinedField" ]);

export function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem o catálogo de linhas de cuidado — o api do módulo 23 precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

// Linha da tabela: a vigente de cada chave, ou a chave do contrato sem versão.
type Row = { key: string; current: TemplateView | null };

export function CareLineTemplates() {
  const query = useQuery({ queryKey: CARE_LINE_TEMPLATES_QUERY_KEY, queryFn: () => gql(CareLineTemplatesQuery), staleTime: Infinity });
  const [ historyKey, setHistoryKey ] = useState<string | null>(null);
  // eligibility e practices são JSON (o codegen os tipa como `any`): a forma é a de TemplateView (L1, L2).
  const all: TemplateView[] = (query.data?.data?.careLineTemplates ?? []) as TemplateView[];
  const fieldError = query.data?.fieldErrors[0];
  const current = currentTemplates(all);
  const missing = KEYS.filter((k) => !current.some((t) => t.key === k));
  const rows: Row[] = [ ...current.map((t) => ({ key: t.key, current: t })), ...missing.map((key) => ({ key, current: null })) ]
    .sort((a, b) => keyOrder(a.key) - keyOrder(b.key));
  const history = historyKey ? versionsOf(all, historyKey) : [];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <section aria-label={TITLE}>
        <Panel title={TITLE} actions={<Button variant="secondary" onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button>}>
          <p style={muted}>{INTRO}</p>
          {query.isPending && <p>carregando…</p>}
          {query.isError && <ErrorState message={requestErrorText(query.error)} />}
          {fieldError && <ErrorState message={requestErrorText(fieldError)} />}
          {query.isSuccess && !fieldError && (
            <DataTable<Row>
              columns={[
                { key: "line", label: "Linha", render: (r) => keyText(r.key) },
                { key: "version", label: "Versão vigente", render: (r) => (r.current ? `v${r.current.version}` : <Tag tone="warn">sem versão</Tag>) },
                { key: "published", label: "Publicada", render: (r) => (r.current ? fmtWhen(r.current.createdAt) : "—") },
                { key: "practices", label: "Boas práticas", render: (r) => (r.current ? String(r.current.practices.length) : "—") },
                { key: "points", label: "Soma dos pontos", render: (r) => (r.current ? decimalText(pointsTotal(r.current.practices)) : "—") },
                {
                  key: "actions", label: "", render: (r) => (
                    <div style={row}>
                      {r.current && <Button variant="secondary" onClick={() => setHistoryKey(r.key)}>versões</Button>}
                      {!r.current && <Button variant="secondary" onClick={() => setHistoryKey(r.key)}>primeira versão</Button>}
                    </div>
                  )
                }
              ]}
              rows={rows}
              rowKey={(r) => r.key}
            />
          )}
        </Panel>
      </section>

      {historyKey && history.length > 0 && (
        <section aria-label={`Versões de ${keyText(historyKey)}`}>
          <Panel title={`Versões de ${keyText(historyKey)}`} actions={<Button variant="secondary" onClick={() => setHistoryKey(null)}>fechar</Button>}>
            <ul style={{ margin: 0, paddingLeft: 18, display: "flex", flexDirection: "column", gap: 12, fontSize: 12.5 }}>
              {history.map((t) => (
                <li key={`${t.key}-${t.version}`} data-version={String(t.version)}>
                  <div style={row}>
                    <strong>{`v${t.version}`}</strong>
                    {t.current && <Tag tone="ok">vigente</Tag>}
                    <span style={muted}>{fmtWhen(t.createdAt)}</span>
                  </div>
                  <p style={{ ...muted, margin: "4px 0" }}>{eligibilityText(t.eligibility)}</p>
                  <ul style={{ margin: 0, paddingLeft: 18 }}>
                    {t.practices.map((p) => {
                      const advanced = advancedText(p);
                      return (
                        <li key={p.key}>
                          {`${p.label}: ${ruleText(p)}`}
                          {advanced && <div style={muted}>{`regras avançadas (preservadas na versão nova): ${advanced}`}</div>}
                        </li>
                      );
                    })}
                  </ul>
                </li>
              ))}
            </ul>
          </Panel>
        </section>
      )}
    </div>
  );
}
```

Em `src/screens/Shell.tsx`:

1. Troque `import { LabTemplates } from "./LabTemplates";` por

```ts
import { LabTemplates } from "./LabTemplates";
import { CareLineTemplates } from "./CareLineTemplates";
```

2. Troque `export type Screen = "cities" | "medications" | "lab" | "maintainers" | "tokens" | "audit";` por

```ts
// Módulo 23: Linhas de cuidado, o catálogo-base das linhas C2–C7 (plataforma).
export type Screen = "cities" | "medications" | "lab" | "careLines" | "maintainers" | "tokens" | "audit";
```

3. Troque `  { key: "lab", label: "Laboratório" },` por

```ts
  { key: "lab", label: "Laboratório" },
  { key: "careLines", label: "Linhas de cuidado" },
```

4. Em `ICON`, depois da linha `lab: svg(…),`, acrescente

```ts
  careLines: svg("M12 21s-7-4.35-7-10a4 4 0 0 1 7-2.65A4 4 0 0 1 19 11c0 5.65-7 10-7 10zM8 12h2l1-2 2 4 1-2h2"),
```

5. Troque `          {screen === "lab" && <LabTemplates />}` por

```tsx
          {screen === "lab" && <LabTemplates />}
          {screen === "careLines" && <CareLineTemplates />}
```

Rode o codegen (o documento `CareLineTemplates` é novo): `cd $M && npm run codegen`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/screens/CareLineTemplates.test.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx src/theme && npx tsc --noEmit && npm run codegen:check`
Expected: PASS. (O rótulo "Linhas de cuidado" cabe no trilho de 92 px em duas linhas, como "Mantenedores"; achado só visual não vira tarefa.)

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/CareLineTemplates.tsx src/screens/CareLineTemplates.fixtures.tsx src/screens/CareLineTemplates.test.tsx src/screens/Shell.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx src/gql/gql.ts src/gql/graphql.ts
$G commit -m "feat: list the care line templates and their versions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Publicar versão nova

**Files:**
- Create: `src/screens/CareLineTemplateForm.tsx`
- Modify: `src/screens/CareLineTemplates.tsx`, `src/gql/*` (codegen)
- Test: `src/screens/CareLineTemplateForm.test.tsx`

**Interfaces:**
- Consumes: `ANCHORS`, `CONDITIONS`, `INDICATORS`, `SCHEDULES`, `SEXES`, `SOURCE_KINDS`, `advancedText`, `anchorText`, `draftProblem`, `draftToVariables`, `emptyPractice`, `firstDraft`, `keyText`, `pointsTotal`, `scheduleText`, `sexText`, `sourceKindText`, `toDraft`, `upsertErrorText`, `TemplateDraft`, `PracticeDraft` (Task 2); `decimalText`, `optionsWith`, `parseDecimal` (`labTemplates.ts`); `CareLineTemplates`, `CARE_LINE_TEMPLATES_QUERY_KEY`, `requestErrorText` (Task 3); `grid`, `inputStyle`, `labelStyle`, `checkStyle`, `muted`, `row` (`LabTemplates.tsx`); `gql`, `graphql`; `Panel`, `Button`, `Field`, `ErrorState`.
- Produces: `CareLineTemplateForm({ draft, fromVersion, busy, onSave(d), onClose })`; a mutation `UpsertCareLineTemplate`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/screens/CareLineTemplateForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { C3_V1, C4_V1, C4_V2, renderScreen, reply } from "./CareLineTemplates.fixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

type Body = { query: string; variables: Record<string, unknown> };
const bodyOf = (call: unknown[]): Body => JSON.parse((call[1] as RequestInit).body as string);
type SentPractice = { key: string; source: string; points: number; all_of?: unknown };

describe("CareLineTemplateForm", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  let upsertReply: () => Response;
  beforeEach(() => {
    upsertReply = () => reply(200, { data: { upsertCareLineTemplate: { ok: true, errors: [] } } });
    fetchMock = vi.fn((_url: string, init: RequestInit) => {
      const body = JSON.parse(init.body as string) as Body;
      if (body.query.includes("mutation UpsertCareLineTemplate")) return Promise.resolve(upsertReply());
      return Promise.resolve(reply(200, { data: { careLineTemplates: [ C4_V2, C4_V1, C3_V1 ] } }));
    });
    vi.stubGlobal("fetch", fetchMock);
  });

  const upserts = () => fetchMock.mock.calls.filter((c) => bodyOf(c).query.includes("mutation UpsertCareLineTemplate"));
  const lists = () => fetchMock.mock.calls.filter((c) => bodyOf(c).query.includes("query CareLineTemplates"));
  const form = () => screen.getByRole("region", { name: "Versão nova do catálogo-base" });
  const practice = (n: number) => within(form()).getByRole("group", { name: `Boa prática ${n}` });

  async function openNewVersion(user: ReturnType<typeof userEvent.setup>, line = /C4 — Pessoa com diabetes/) {
    renderScreen();
    await user.click(await within(await screen.findByRole("row", { name: line })).findByRole("button", { name: "versões" }));
    await user.click(screen.getByRole("button", { name: "nova versão" }));
  }

  it("publica versão nova a partir da vigente, com o aviso de que as cidades adotam", async () => {
    const user = userEvent.setup();
    await openNewVersion(user);
    expect(within(form()).getByText(/só vale para a cidade que a adotar/)).not.toBeNull();
    expect((within(form()).getByLabelText("Nome da linha") as HTMLInputElement).value).toBe("Pessoa com diabetes");
    await user.clear(within(practice(1)).getByLabelText("Pontos"));
    await user.type(within(practice(1)).getByLabelText("Pontos"), "84,5");
    await user.clear(within(practice(2)).getByLabelText("Pontos"));
    await user.type(within(practice(2)).getByLabelText("Pontos"), "15,5");
    expect(within(form()).getByText("Soma dos pontos: 100")).not.toBeNull();
    await user.click(within(form()).getByRole("button", { name: "publicar versão" }));
    await waitFor(() => expect(upserts()).toHaveLength(1));
    const vars = bodyOf(upserts()[0]).variables;
    expect([ vars.key, vars.name, vars.indicator, vars.declaredCondition ]).toEqual([ "diabetes_c4", "Pessoa com diabetes", "C4", "diabetes" ]);
    expect("version" in vars).toBe(false);
    expect((vars.practices as SentPractice[]).map((p) => [ p.key, p.points ])).toEqual([ [ "c4_a", 84.5 ], [ "c4_e", 15.5 ] ]);
    expect(await screen.findByText("Versão nova publicada. As cidades adotam quando quiserem.")).not.toBeNull();
    await waitFor(() => expect(lists().length).toBe(2));
  });

  it("exame pede o SIGTAP e manda exam:<código>; visita do ACS some o código; avançados voltam iguais", async () => {
    const user = userEvent.setup();
    await openNewVersion(user);
    await user.click(within(form()).getByRole("button", { name: "incluir boa prática" }));
    const third = practice(3);
    await user.type(within(third).getByLabelText("Chave"), "c4_g");
    await user.type(within(third).getByLabelText("O que conta"), "Creatinina em 12 meses");
    await user.selectOptions(within(third).getByLabelText("Fonte"), "exam");
    await user.type(within(third).getByLabelText("Código"), "0202010317");
    await user.type(within(third).getByLabelText("Pontos"), "0");
    await user.click(within(form()).getByRole("button", { name: "publicar versão" }));
    await waitFor(() => expect(upserts()).toHaveLength(1));
    expect((bodyOf(upserts()[0]).variables.practices as SentPractice[])[2].source).toBe("exam:0202010317");
    await user.click(within(screen.getByRole("row", { name: /C4 — Pessoa com diabetes/ })).getByRole("button", { name: "versões" }));
    await user.click(screen.getByRole("button", { name: "nova versão" }));
    await user.selectOptions(within(practice(2)).getByLabelText("Fonte"), "acs_visit");
    expect(within(practice(2)).queryByLabelText("Código")).toBeNull();
    await user.click(within(form()).getByRole("button", { name: "fechar" }));

    await user.click(within(screen.getByRole("row", { name: /C3 — Gestante e puérpera/ })).getByRole("button", { name: "versões" }));
    await user.click(screen.getByRole("button", { name: "nova versão" }));
    expect(within(practice(1)).getByText(/regras avançadas \(preservadas\): todas de/)).not.toBeNull();
    await user.click(within(form()).getByRole("button", { name: "publicar versão" }));
    await waitFor(() => expect(upserts()).toHaveLength(2));
    const c3 = bodyOf(upserts()[1]).variables;
    expect((c3.practices as SentPractice[])[0].all_of).toEqual(C3_V1.practices[0].all_of);
    expect(c3.eligibility).toEqual(C3_V1.eligibility);
  });

  it("pontos com ponto são recusados e ficam no campo; nada é enviado", async () => {
    const user = userEvent.setup();
    await openNewVersion(user);
    await user.clear(within(practice(1)).getByLabelText("Pontos"));
    await user.type(within(practice(1)).getByLabelText("Pontos"), "84.5");
    await user.click(within(form()).getByRole("button", { name: "publicar versão" }));
    expect((await screen.findByRole("alert")).textContent).toBe("boa prática 1: pontos — use vírgula para decimais e não use ponto de milhar");
    expect((within(practice(1)).getByLabelText("Pontos") as HTMLInputElement).value).toBe("84.5");
    expect(upserts()).toHaveLength(0);
  });

  it("primeira versão de uma chave sem versão usa o nome e o indicador do contrato", async () => {
    const user = userEvent.setup();
    renderScreen();
    await user.click(await within(await screen.findByRole("row", { name: /C2 — Cuidado da criança até 2 anos/ })).findByRole("button", { name: "primeira versão" }));
    expect((within(form()).getByLabelText("Nome da linha") as HTMLInputElement).value).toBe("Cuidado da criança até 2 anos");
    expect((within(form()).getByLabelText("Indicador") as HTMLSelectElement).value).toBe("C2");
    expect(within(form()).getAllByRole("group").filter((g) => /Boa prática/.test(g.getAttribute("aria-label") ?? ""))).toHaveLength(1);
  });

  it("recusa do api vira frase com o campo; código novo cru; o formulário fica", async () => {
    const user = userEvent.setup();
    upsertReply = () => reply(200, { data: { upsertCareLineTemplate: { ok: false, errors: [
      { path: "practices[1].source", message: "invalid_practices" }, { path: null, message: "brand_new" } ] } } });
    await openNewVersion(user);
    await user.click(within(form()).getByRole("button", { name: "publicar versão" }));
    expect((await screen.findByRole("alert")).textContent).toBe("boas práticas inválidas — confira os campos (practices[1].source); brand_new");
    expect(form()).not.toBeNull();
    expect(lists()).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/screens/CareLineTemplateForm.test.tsx`
Expected: FAIL — não há o botão "nova versão".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/screens/CareLineTemplateForm.tsx
// Versão nova de uma linha do catálogo-base (módulo 23; contrato §8; plano do
// api, Task 4): parte da vigente (ou do nome e indicador do contrato, na
// primeira), edita os campos simples da elegibilidade e das boas práticas e
// publica. Os campos avançados (all_of, code_filter, applies_to, identidades de
// gênero, `declared`) voltam iguais e aparecem em texto. Publicar SEMPRE cria a
// próxima versão e não muda nenhuma cidade até ela adotar. Formato conferido
// aqui (recusar, nunca cortar) e de novo no api.
import { useState } from "react";
import { decimalText, optionsWith, parseDecimal } from "../lib/labTemplates";
import {
  ANCHORS, CONDITIONS, INDICATORS, SCHEDULES, SEXES, SOURCE_KINDS, advancedText, anchorText, emptyPractice, keyText, pointsTotal, scheduleText,
  sexText, sourceKindText, type PracticeDraft, type TemplateDraft
} from "../lib/careLineTemplates";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Field } from "../components/Field";
import { checkStyle, grid, inputStyle, labelStyle, muted, row } from "./LabTemplates";

const NOTICE = "A versão nova só vale para a cidade que a adotar; as inscrições das cidades não mudam até lá.";
const CODED = new Set([ "exam", "procedure", "vaccine" ]);
const CONDITION_TEXT: Record<string, string> = {
  "": "nenhuma", diabetes: "diabetes", hypertension: "hipertensão", pregnancy: "gestação", mental_health: "saúde mental"
};

type Props = { draft: TemplateDraft; fromVersion: number | null; busy: boolean; onSave(d: TemplateDraft): void; onClose(): void };

export function CareLineTemplateForm({ draft: initial, fromVersion, busy, onSave, onClose }: Props) {
  const [ d, setD ] = useState<TemplateDraft>(initial);
  const set = <K extends keyof TemplateDraft>(field: K) => (value: TemplateDraft[K]) => setD((x) => ({ ...x, [field]: value }));
  const setPractice = (i: number, patch: Partial<PracticeDraft>) =>
    setD((x) => ({ ...x, practices: x.practices.map((p, j) => (j === i ? { ...p, ...patch } : p)) }));
  // Soma só do que já é número; o resto é conferido ao publicar.
  const total = pointsTotal(d.practices.map((p) => ({ points: parseDecimal(p.points) ?? null })));
  const preserved = [
    d.preserved.gender_identities_included?.length ? `inclui: ${d.preserved.gender_identities_included.join(", ")}` : null,
    d.preserved.gender_identities_excluded?.length ? `exclui: ${d.preserved.gender_identities_excluded.join(", ")}` : null,
    d.preserved.declared?.length ? `condição declarada: ${d.preserved.declared.join(", ")}` : null
  ].filter((x): x is string => x !== null);

  const select = (label: string, value: string, options: readonly string[], text: (o: string) => string, onChange: (v: string) => void) => (
    <label style={labelStyle}>
      <span>{label}</span>
      <select value={value} onChange={(event) => onChange(event.target.value)} style={inputStyle}>
        {optionsWith(options, value).map((o) => <option key={o} value={o}>{text(o)}</option>)}
      </select>
    </label>
  );

  return (
    <section aria-label="Versão nova do catálogo-base">
      <Panel title={fromVersion === null ? `Primeira versão de ${keyText(d.key)}` : `Versão nova de ${keyText(d.key)} (a partir da v${fromVersion})`}>
        <p style={muted}>{NOTICE}</p>
        <div style={grid}>
          <Field label="Nome da linha" value={d.name} onChange={set("name")} helpText="até 120 caracteres" />
          {select("Indicador", d.indicator, INDICATORS, (o) => o, set("indicator"))}
          {select("Condição declarada que sugere a linha", d.declaredCondition, CONDITIONS, (o) => CONDITION_TEXT[o] ?? o, set("declaredCondition"))}
        </div>
        <fieldset aria-label="Elegibilidade" style={fieldsetStyle}>
          <legend style={legendStyle}>Quem é elegível</legend>
          <div style={grid}>
            <Field label="Códigos CIAP-2 (problema ativo)" value={d.ciap} onChange={set("ciap")} helpText="por prefixo, separados por vírgula" />
            <Field label="Códigos CID-10 (problema ativo)" value={d.cid10} onChange={set("cid10")} helpText="por prefixo, sem ponto (E11 casa E119)" />
            <Field label="Exceto CIAP-2" value={d.excludeCiap} onChange={set("excludeCiap")} helpText="ex.: aborto W82, W83" />
            <Field label="Exceto CID-10" value={d.excludeCid10} onChange={set("excludeCid10")} />
            <Field label="Idade mínima (anos)" value={d.ageMinYears} onChange={set("ageMinYears")} helpText="vazio = sem limite" />
            <Field label="Idade máxima (anos)" value={d.ageMaxYears} onChange={set("ageMaxYears")} helpText="vazio = sem limite" />
            <Field label="Idade mínima (dias de vida)" value={d.ageMinDays} onChange={set("ageMinDays")} />
            <Field label="Idade máxima (dias de vida)" value={d.ageMaxDays} onChange={set("ageMaxDays")} helpText="até 2 anos = 730" />
            {select("Sexo", d.sex, SEXES, sexText, set("sex"))}
          </div>
          <label style={checkStyle}>
            <input type="checkbox" checked={d.requiresAnchor === "lmp"} onChange={(event) => set("requiresAnchor")(event.target.checked ? "lmp" : "")} />
            pede a DUM na inscrição (gestação)
          </label>
          {preserved.length > 0 && <p style={muted}>{`regras avançadas (preservadas): ${preserved.join(" · ")}`}</p>}
        </fieldset>
        {d.practices.map((p, i) => {
          const advanced = advancedText({ key: p.key, label: p.label, source: p.sourceKind, anchor: p.anchor, window_days: 0, min_count: 0,
            schedule: p.schedule, ...p.advanced });
          return (
            <fieldset key={i} role="group" aria-label={`Boa prática ${i + 1}`} style={fieldsetStyle}>
              <legend style={legendStyle}>Boa prática {i + 1}</legend>
              <div style={grid}>
                <Field label="Chave" value={p.key} onChange={(v) => setPractice(i, { key: v })} helpText="minúsculas, números e _ (ex.: c4_e)" />
                <Field label="O que conta" value={p.label} onChange={(v) => setPractice(i, { label: v })} helpText="até 120 caracteres" />
                <Field label="Texto para o cidadão" value={p.citizenLabel} onChange={(v) => setPractice(i, { citizenLabel: v })}
                  helpText="até 80, em linguagem simples (Meu acompanhamento)" />
                {select("Fonte", p.sourceKind, SOURCE_KINDS, sourceKindText, (v) => setPractice(i, { sourceKind: v, code: CODED.has(v) ? p.code : "" }))}
                {CODED.has(p.sourceKind) && (
                  <Field label="Código" value={p.code} onChange={(v) => setPractice(i, { code: v })}
                    helpText={p.sourceKind === "vaccine" ? "código da vacina" : "SIGTAP, 10 dígitos"} />
                )}
                <Field label="Início da janela (dias)" value={p.windowStartDays} onChange={(v) => setPractice(i, { windowStartDays: v })}
                  helpText="vazio = desde a âncora" />
                <Field label="Fim da janela (dias)" value={p.windowDays} onChange={(v) => setPractice(i, { windowDays: v })} />
                <Field label="Quantidade mínima" value={p.minCount} onChange={(v) => setPractice(i, { minCount: v })} />
                {select("Cronograma", p.schedule, SCHEDULES, scheduleText, (v) => setPractice(i, { schedule: v }))}
                {select("Âncora", p.anchor, ANCHORS, anchorText, (v) => setPractice(i, { anchor: v }))}
                <Field label="CBO que conta (prefixos)" value={p.cboFilter} onChange={(v) => setPractice(i, { cboFilter: v })}
                  helpText="vazio = qualquer; médico e enfermeiro = 2231, 2251, 2252, 2253, 2235" />
                <Field label="Pontos" value={p.points} onChange={(v) => setPractice(i, { points: v })} helpText="Quadro 01; vírgula para decimais" />
              </div>
              {advanced && <p style={muted}>{`regras avançadas (preservadas): ${advanced}`}</p>}
              {d.practices.length > 1 && (
                <div style={row}>
                  <Button variant="secondary" onClick={() => setD((x) => ({ ...x, practices: x.practices.filter((_, j) => j !== i) }))}>
                    tirar boa prática {i + 1}
                  </Button>
                </div>
              )}
            </fieldset>
          );
        })}
        <div style={row}>
          <Button variant="secondary" onClick={() => setD((x) => ({ ...x, practices: [ ...x.practices, emptyPractice() ] }))}>incluir boa prática</Button>
        </div>
        <p style={muted}>{`Soma dos pontos: ${decimalText(total)}${total === 100 ? "" : " (precisa ser 100)"}`}</p>
        <div style={row}>
          <Button onClick={() => onSave(d)} busy={busy}>publicar versão</Button>
          <Button variant="secondary" onClick={onClose}>fechar</Button>
        </div>
      </Panel>
    </section>
  );
}

const fieldsetStyle = { border: "1px solid var(--rule)", borderRadius: 6, padding: 12, margin: "8px 0", display: "flex", flexDirection: "column" as const, gap: 8 };
const legendStyle = { fontSize: 12.5, fontWeight: 600, color: "var(--ink)" };
```

Em `src/screens/CareLineTemplates.tsx`:

1. Troque os imports

```ts
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
```

por

```ts
import { useState } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
```

troque

```ts
import {
  KEYS, advancedText, currentTemplates, eligibilityText, keyOrder, keyText, pointsTotal, ruleText, versionsOf, type TemplateView
} from "../lib/careLineTemplates";
```

por

```ts
import {
  KEYS, advancedText, currentTemplates, draftProblem, draftToVariables, eligibilityText, firstDraft, keyOrder, keyText, pointsTotal, ruleText,
  toDraft, upsertErrorText, versionsOf, type TemplateDraft, type TemplateView
} from "../lib/careLineTemplates";
```

e, depois de `import { muted, row } from "./LabTemplates";`, acrescente

```ts
import { CareLineTemplateForm } from "./CareLineTemplateForm";
```

2. Depois da constante `CareLineTemplatesQuery`, acrescente

```ts
const UpsertCareLineTemplateMutation = graphql(`
  mutation UpsertCareLineTemplate(
    $key: String!, $name: String!, $indicator: String!, $declaredCondition: String, $eligibility: JSON!, $practices: JSON!
  ) {
    upsertCareLineTemplate(key: $key, name: $name, indicator: $indicator, declaredCondition: $declaredCondition,
      eligibility: $eligibility, practices: $practices) {
      ok errors { path message }
    }
  }
`);

type Outcome = { tone: "done" | "failed"; text: string };
type Editing = { draft: TemplateDraft; fromVersion: number | null } | null;
```

3. Em `CareLineTemplates()`, depois de `  const [ historyKey, setHistoryKey ] = useState<string | null>(null);`, acrescente

```ts
  const queryClient = useQueryClient();
  const [ editing, setEditing ] = useState<Editing>(null);
  const [ outcome, setOutcome ] = useState<Outcome | null>(null);

  const save = useMutation({
    mutationFn: (d: TemplateDraft) => gql(UpsertCareLineTemplateMutation, draftToVariables(d)),
    onMutate: () => setOutcome(null),
    onSuccess: async ({ data, fieldErrors }) => {
      const payload = data?.upsertCareLineTemplate ?? null;
      if (!payload) { setOutcome({ tone: "failed", text: fieldErrors[0] ? `${fieldErrors[0].code} — ${fieldErrors[0].message}` : upsertErrorText([]) }); return; }
      if (!payload.ok) { setOutcome({ tone: "failed", text: upsertErrorText(payload.errors) }); return; }
      setEditing(null);
      setOutcome({ tone: "done", text: "Versão nova publicada. As cidades adotam quando quiserem." });
      await queryClient.invalidateQueries({ queryKey: CARE_LINE_TEMPLATES_QUERY_KEY });
    },
    // Rede/timeout: o api pode ter gravado; relê e mantém o formulário.
    onError: (err) => {
      setOutcome({ tone: "failed", text: requestErrorText(err) });
      void queryClient.invalidateQueries({ queryKey: CARE_LINE_TEMPLATES_QUERY_KEY });
    }
  });

  function startNew(key: string) {
    const base = currentTemplates(all).find((t) => t.key === key) ?? null;
    setOutcome(null);
    setEditing({ draft: base ? toDraft(base) : firstDraft(key), fromVersion: base ? base.version : null });
  }
```

(`startNew` usa `all`, declarado logo abaixo no componente; como só roda no clique, a ordem das declarações não importa.)

4. Na coluna `actions`, troque

```tsx
                      {!r.current && <Button variant="secondary" onClick={() => setHistoryKey(r.key)}>primeira versão</Button>}
```

por

```tsx
                      {!r.current && <Button variant="secondary" onClick={() => { setHistoryKey(null); startNew(r.key); }}>primeira versão</Button>}
```

5. No painel de versões, troque

```tsx
          <Panel title={`Versões de ${keyText(historyKey)}`} actions={<Button variant="secondary" onClick={() => setHistoryKey(null)}>fechar</Button>}>
```

por

```tsx
          <Panel title={`Versões de ${keyText(historyKey)}`} actions={
            <div style={row}>
              <Button onClick={() => startNew(historyKey)}>nova versão</Button>
              <Button variant="secondary" onClick={() => setHistoryKey(null)}>fechar</Button>
            </div>
          }>
```

6. Antes do último `</div>` do retorno de `CareLineTemplates()` (depois do bloco `{historyKey && history.length > 0 && (…)}`), acrescente

```tsx
      {editing && (
        <CareLineTemplateForm
          key={`${editing.draft.key}-${editing.fromVersion ?? 0}`}
          draft={editing.draft}
          fromVersion={editing.fromVersion}
          busy={save.isPending}
          onSave={(d) => {
            const problem = draftProblem(d);
            if (problem) setOutcome({ tone: "failed", text: problem });
            else save.mutate(d);
          }}
          onClose={() => { setEditing(null); setOutcome(null); }}
        />
      )}
      {outcome?.tone === "done" && <p role="status" style={{ margin: 0, fontSize: 12.5, color: "var(--ok)" }}>{outcome.text}</p>}
      {outcome?.tone === "failed" && <ErrorState message={outcome.text} />}
```

Rode o codegen (a mutation é nova): `cd $M && npm run codegen`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/screens/CareLineTemplateForm.test.tsx src/screens/CareLineTemplates.test.tsx src/theme && npx tsc --noEmit && npm run codegen:check`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/CareLineTemplateForm.tsx src/screens/CareLineTemplateForm.test.tsx src/screens/CareLineTemplates.tsx src/gql/gql.ts src/gql/graphql.ts
$G commit -m "feat: publish a new version of a care line template

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Suíte, schema igual ao do api, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo (o `schema.graphql` e o `src/gql/*` podem ser ressincronizados).

**Interfaces:**
- Consumes: as Tasks 1–4 na branch `feat/mod-23-care-lines`; o api do 23 na **3045**.
- Produces: branch do maintenance pronta para merge (depois do merge do api, do dashboard e do wpda do 23).

- [ ] **Step 1: O SDL igual ao do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; M=$R/.worktrees/maintenance-mod23; G=/opt/homebrew/bin/git
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod23 api bin/rails runner 'puts Maintenance::Schema.to_definition' > $M/schema.graphql
cd $M && npm run codegen && npx tsc --noEmit && $G status --short
```

Expected: o `tsc` verde (os documentos das Tasks 3–4 batem com o schema do api). Se o `schema.graphql` ou o `src/gql/*` mudaram só por ordem ou descrição, commite como sincronização:

```bash
cd $M
$G add schema.graphql src/gql/gql.ts src/gql/graphql.ts
$G commit -m "chore: sync the maintenance schema with the care lines api

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Se o `tsc` falhar (nome ou tipo de campo diferente da Task 4 do plano do api), **pare e reporte**: a diferença é de contrato; ajuste os documentos e `careLineTemplates.ts` só depois da decisão.

- [ ] **Step 2: Suíte, tipos, codegen e build**

Run: `cd $M && npm run typecheck && npx vitest run && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base anotada na Task 0 mais **3 arquivos de teste novos** (`careLineTemplates.test.ts`, `CareLineTemplates.test.tsx`, `CareLineTemplateForm.test.tsx`) e casos novos em `features.test.ts`, `FeaturesTab.test.tsx`, `Shell.test.tsx` e `Shell.rail.test.tsx`. Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
grep -n "maxLength" src/screens/CareLineTemplates.tsx src/screens/CareLineTemplateForm.tsx
```

Expected: `status` vazio, contagem `0` e nenhum `maxLength`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §4 "Plataforma", §6 "Maintenance", §10 F-23.1), o ADR 0040 e o contrato §1 e §8. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature` nem em `FeaturesTab.tsx`; a frase de `clinical_record_disabled` não foi redefinida; o rótulo igual ao do dashboard;
- publicar manda a chave (versão nova), nunca id de versão; a tela diz que as cidades adotam;
- pontos com vírgula, somando 100; ponto recusado; textos recusados, nunca cortados; fonte desconhecida continua selecionada; avançados (`all_of`, `code_filter`, `applies_to`, identidades de gênero, `declared`) voltam iguais;
- consulta e mutation próprias da tela; contra um api sem o 23, só ela cai;
- `schema.graphql` igual ao `to_definition` do api; `src/gql/*` regenerado;
- nenhum dado de cidadão; cores só por papel do tema; tabelas no `DataTable` (com `TableScroll`).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 23 na **3045** (plano do api: banco de dev migrado — migrações de plataforma a partir de `20261022900001` — e semeado **com autorização do usuário**, com as seis linhas do catálogo-base na versão 1). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, com `node_modules` num volume próprio, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod23 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod23:/app -v maintenance-mod23-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3045 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod23   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 23 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. **Anote** antes de mexer o estado de `care_lines` e de `clinical_record` de Curitiba. Confira com screenshot (salve em `evidencias/2026-10-22-modulo-23/maintenance/`):
- Cidades → Curitiba → **Funcionalidades**: `care_lines` com "Linhas de cuidado"; ligar (dois cliques) → "care_lines ligada e utilizável." (Curitiba em `record` com `clinical_record`); numa cidade sem prontuário (se houver na semente), ligar dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado";
- trilho → **Linhas de cuidado**: as seis linhas com a v1, a soma dos pontos (100) e "Publicada"; **versões** de C4 mostra a regra de cada boa prática (a HbA1c como "exame SIGTAP 0202010503…", a visita do ACS "não acompanhada pelo sistema");
- **nova versão** de C4: mudar os pontos de uma boa prática para "12.5" → recusado; somando 95 → recusado com a soma; de volta a 100 → **publicar versão** → "Versão nova publicada. As cidades adotam quando quiserem." e a C4 em v2; na C3, as regras avançadas (testes do pré-natal) aparecem em texto e voltam iguais na versão nova;
- no dashboard (prova do plano do dashboard), como admin de Curitiba, **Linhas da cidade** oferece "Adotar versão 2" para a C4 — e só muda depois de adotar;
- **Auditoria** mostra as tentativas e os resultados de `setCityFeature` e `maintenance.care_line_template.changed` (L13);
- a 375 px de largura: sem rolagem lateral da página; as tabelas rolam dentro do contêiner com a dica "⇆ role para o lado".

Depois: devolva `care_lines` ao estado anotado (pela própria tela) e, se a prova do api pedir, deixe a C4 em v2 (ou registre na semente de dev); `docker rm -f maintenance-mod23 && docker volume rm maintenance-mod23-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api, do dashboard e do wpda do 23, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-23-care-lines
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod23
/opt/homebrew/bin/git branch -d feat/mod-23-care-lines
```

Se o lado api do F-23.1 já estiver **Done**, mova o card F-23.1 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0; e para **Verified** se a prova do Step 4 passou.

---

### Task 6: O módulo 23 no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`, `src/App.test.tsx`

**Interfaces:**
- Consumes: os F-IDs F-23.1..F-23.8 no board #1 (status real de cada um); o resultado da prova do 23 (planos do api, do dashboard, do wpda e deste); o que foi entregue em api, dashboard, wpda e maintenance.
- Produces: módulo `"23"` no JSON (ciclo 2, depois do `"22"` e antes do `"iu"`), com F-23.1..F-23.8, `status: "validating"`; `meta.cycles[1].modules` = "15 a 23 e interface"; o rodapé "o 19, o 20, o 21, o 22 e o 23 estão em validação".

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-23\\.[1-8] ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod23 -b content/module-23-catalog origin/main
cd $R/.worktrees/catalog-mod23 && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: F-23.1..F-23.8 em Done ou Verified; os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário. A suíte do catálogo verde. **Anote**: o JSON termina o ciclo 2 em `"22"` e `"iu"` (`f3dfe7e` ou mais novo); o rodapé diz "o 19, o 20, o 21 e o 22 estão em validação"; o ciclo 2 diz "15 a 22 e interface".

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("formatValidating")`, troque

```ts
    expect(formatValidating(["19", "20", "21", "22"])).toBe("o 19, o 20, o 21 e o 22 estão");
```

por

```ts
    expect(formatValidating(["19", "20", "21", "22"])).toBe("o 19, o 20, o 21 e o 22 estão");
    expect(formatValidating(["19", "20", "21", "22", "23"])).toBe("o 19, o 20, o 21, o 22 e o 23 estão");
```

e, no fim do arquivo:

```ts
describe("módulo 23", () => {
  it("entra no ciclo 2 depois do 22: a prévia e a busca ativa em validação até a cidade piloto", () => {
    const ids = catalog.modules.map((m) => m.id);
    expect(ids.indexOf("23")).toBe(ids.indexOf("22") + 1);
    const m23 = catalog.modules.find((m) => m.id === "23")!;
    expect(m23.cycle).toBe(2);
    expect(m23.status).toBe("validating");
    expect(m23.name).toBe("Linhas de cuidado");
    expect(m23.description).toMatch(/busca ativa/);
    expect(m23.description).toMatch(/estimativa/);
    expect(m23.features.map((f) => f.id)).toEqual(Array.from({ length: 8 }, (_, i) => `F-23.${i + 1}`));
    for (const id of ["f-23.1", "f-23.2", "f-23.3", "f-23.4", "f-23.5", "f-23.7", "f-23.8"]) {
      expect(findFeature(id)?.feature.status).toBe("available");
    }
    const preview = findFeature("f-23.6")!.feature;
    expect(preview.status).toBe("validating");
    expect(preview.note).toMatch(/SIAPS/);
    expect(findFeature("f-23.1")?.feature.surfaces).toEqual([ "plataforma", "secretaria" ]);
    expect(findFeature("f-23.3")?.feature.description).toMatch(/médico ou enfermeiro/);
    expect(findFeature("f-23.4")?.feature.description).toMatch(/não acompanhado/);
    expect(findFeature("f-23.4")?.feature.surfaces).toEqual([ "bastidores" ]);
    expect(findFeature("f-23.7")?.feature.surfaces).toEqual([ "cidadao" ]);
    expect(findFeature("f-23.8")?.feature.surfaces).toEqual([ "cidadao" ]);
    expect(catalog.meta.cycles.find((c) => c.id === 2)?.modules).toBe("15 a 23 e interface");
  });
});
```

Em `src/App.test.tsx`, troque

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21 e o 22 estão em validação."
```

por

```ts
      "Os módulos 02 a 18 estão disponíveis e verificados; o 19, o 20, o 21, o 22 e o 23 estão em validação."
```

e

```ts
    expect(within(c2).getByText("módulos 15 a 22 e interface")).toBeInTheDocument();
```

por

```ts
    expect(within(c2).getByText("módulos 15 a 23 e interface")).toBeInTheDocument();
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts src/App.test.tsx`
Expected: FAIL — não há módulo 23, o ciclo 2 diz "15 a 22" e o rodapé não fala do 23. (O teste de `formatValidating` com cinco já passa: a função é genérica.)

- [ ] **Step 4: Write minimal implementation**

O JSON é reescrito por script: `json.dumps(…, ensure_ascii=False, indent=2)` reproduz o arquivo publicado byte a byte (o script confere), então o diff fica só no que mudou. **Os textos são o rascunho**: antes de rodar, confira cada frase contra o que foi entregue (cards Verified, telas do dashboard e do wpda e regras do api) e corte o que não existir; nada de promessa.

```bash
cd $C && python3 -I - src/data/funcionalidades.json "$(date +%F)" <<'PY'
import json, sys
path, today = sys.argv[1], sys.argv[2]
raw = open(path, encoding="utf-8").read()
data = json.loads(raw)
assert json.dumps(data, ensure_ascii=False, indent=2) + "\n" == raw, "formato do JSON mudou: pare e reporte"

ids = [m["id"] for m in data["modules"]]
assert "23" not in ids, "módulo 23 já existe"
assert data["meta"]["cycles"][1]["modules"] == "15 a 22 e interface", "ciclo 2 mudou: pare e reporte"

data["meta"]["updatedAt"] = today
data["meta"]["cycles"][1]["modules"] = "15 a 23 e interface"

def feature(n, title, benefit, description, surfaces, status="available", note=None):
    f = {"id": f"F-23.{n}", "slug": f"f-23.{n}", "title": title, "benefit": benefit,
         "description": description, "surfaces": surfaces, "status": status}
    if note:
        f["note"] = note
    return f

m23 = {
    "id": "23",
    "cycle": 2,
    "name": "Linhas de cuidado",
    "status": "validating",
    "description": (
        "Criança, gestante, diabetes, hipertensão, pessoa idosa e prevenção do câncer na mulher ganham acompanhamento pessoa a pessoa, "
        "pelas boas práticas das notas metodológicas do componente de qualidade da APS. A equipe vê quem está em dia ou atrasado e faz a "
        "busca ativa; a gestão vê uma estimativa dos indicadores C2 a C7 por equipe, antes da extração oficial."
    ),
    "features": [
        feature(1, "Interruptor e catálogo-base das linhas",
            "As linhas entram por cidade, a partir de um catálogo mantido pela plataforma.",
            "A equipe da plataforma liga as linhas de cuidado por cidade (só com o prontuário do Rota Saúde) e mantém as seis linhas das "
            "notas metodológicas em versões. Uma versão nova só vale para a cidade que a adotar.",
            ["plataforma", "secretaria"]),
        feature(2, "Linhas da cidade",
            "A secretaria escolhe as linhas e cria as próprias.",
            "A administração adota as linhas do catálogo, adota as versões novas quando quiser e cria linhas próprias — por exemplo, "
            "saúde mental — com as mesmas regras de elegibilidade e boas práticas.",
            ["secretaria"]),
        feature(3, "Sugestão e inscrição confirmada",
            "Ninguém entra numa linha sem um profissional confirmar.",
            "O sistema sugere quem deveria estar em cada linha, pela lista de problemas, pela idade e sexo e pela condição que o próprio "
            "cidadão declarou. Só médico ou enfermeiro confirma; na gestante, com a data da última menstruação. Também dá para inscrever à mão.",
            ["secretaria"]),
        feature(4, "Acompanhamento das boas práticas",
            "A situação de cada pessoa sai do que já foi registrado, sem digitar de novo.",
            "Consultas, pressão, peso e altura, exames e procedimentos já registrados dizem se cada boa prática está em dia, vencendo ou "
            "vencida. O que o sistema ainda não registra, como visita do agente e vacina, aparece como não acompanhado e não conta como atraso.",
            ["bastidores"]),
        feature(5, "Painel da equipe e busca ativa",
            "A equipe sabe quem procurar esta semana e por quê.",
            "O painel mostra, por equipe e linha, quem tem alguma boa prática vencida, registra cada tentativa de contato e abre o "
            "agendamento da unidade para a pessoa. A leitura nominal fica registrada na trilha de auditoria.",
            ["secretaria"]),
        feature(6, "Prévia dos indicadores C2 a C7",
            "A gestão vê onde a equipe vai perder pontos antes da extração oficial.",
            "Por equipe, a prévia estima a pontuação de cada indicador sobre a população elegível, inscrita ou não, sem visita do agente e "
            "vacina. É sempre rotulada como estimativa.",
            ["secretaria"], status="validating",
            note="O valor oficial é calculado pelo SIAPS. A prévia foi conferida com dados de exemplo; falta comparar com uma extração oficial da secretaria piloto."),
        feature(7, "Condição declarada e Meu acompanhamento",
            "O cidadão diz o que tem e vê o que está para fazer.",
            "No celular, o cidadão com cadastro validado pode informar diabetes, pressão alta, gravidez ou acompanhamento de saúde mental, "
            "e tirar quando quiser — não é diagnóstico. Em Meu acompanhamento, vê as linhas em que foi inscrito e o que está para fazer, "
            "em linguagem simples.",
            ["cidadao"]),
        feature(8, "Aviso de triagem periódica vencida",
            "A triagem que precisa ser refeita aparece primeiro.",
            "Na tela de triagens do celular, a triagem com prazo de repetição vencido aparece em destaque, com a data da última vez, para "
            "qualquer pessoa, inscrita ou não numa linha.",
            ["cidadao"]),
    ],
}

data["modules"].insert(ids.index("22") + 1, m23)
open(path, "w", encoding="utf-8").write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
print("ok", [m["id"] for m in data["modules"]][-3:], len(m23["features"]))
PY
```

Expected: `ok ['22', '23', 'iu'] 8`. Se o Step 1 mostrar algum F-23.x (além do F-23.6) que não chegou a Verified, ou a prova tiver ficado só com dado de exemplo em outro ponto, ajuste o `status` para `validating` com `note` que diga o que falta (edite o JSON à mão nesse objeto) e espelhe no teste do Step 2. Se o atalho de agendamento (Divergência D7 do plano do dashboard) não tiver entrado, tire "e abre o agendamento da unidade para a pessoa" da F-23.5.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todas, telas conhecidas, slugs únicos); o módulo 23 com 8 funcionalidades; o rodapé e o ciclo com os textos novos.

```bash
cd $C && $G status --short && $G diff --stat
```

Expected: só os três arquivos da task (nenhum `package-lock.json`); no JSON, só o módulo 23 acrescentado, o ciclo 2 e o `updatedAt`.

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts src/App.test.tsx
$G commit -m "docs: add module 23 care lines to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot: o **Módulo 23 — Linhas de cuidado** no ciclo 2, logo depois do 22, com a descrição e as 8 funcionalidades; a F-23.6 "Em validação" com a nota do SIAPS; o rodapé "o 19, o 20, o 21, o 22 e o 23 estão em validação"; a página `/funcionalidades/f-23.5` com "O que faz", as telas e anterior/próxima dentro do módulo (F-23.4 e F-23.6); a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod23
/opt/homebrew/bin/git push -u origin content/module-23-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-23-catalog --title "docs: add module 23 care lines to the catalog" --body "Adds module 23 (care lines) after module 22 with F-23.1..F-23.8; the C2-C7 preview (F-23.6) stays in validation until it is compared with an official extraction; cycle 2 and the footer updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod23 && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-23-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

Alinhadas ao plano do api (`2026-10-22-module-23-care-lines-api.md`, L1–L14).

1. **M1 — Forma GraphQL do catálogo-base. Adotada (plano do api, Task 4):** `careLineTemplates(key: String)` devolve todas as versões (`key version current name indicator declaredCondition eligibility practices createdAt`, com `eligibility`/`practices` em `JSON`, L1/L2); `upsertCareLineTemplate(key, name, indicator, declaredCondition, eligibility, practices)` grava sempre a versão seguinte e devolve `{ ok, errors { path message } }`. Fica fora o autor da versão (a tela mostra só a data); se o contrato quiser "quem publicou" na tela, proposta: `createdBy: String` (e-mail do mantenedor) no tipo.
2. **M2 — Pontos no catálogo-base. Adotada (L1):** `points` em cada boa prática, somando 100 nos templates; a prévia redistribui (L8).
3. **M3 — Recusas da mutation. Adotada (plano do api):** `invalid_key`, `invalid_indicator`, `invalid_name`, `invalid_declared_condition`, `invalid_practices`, `invalid_eligibility`, com `path` do campo (`practices[2].window_days`). A tela traduz e mostra o campo.
4. **M4 — Sem step-up no catálogo-base. Aberta (só registrar):** como o catálogo de laboratório do 21b; publicar versão nova não muda nenhuma cidade nem toca dado de cidadão; a auditoria é `maintenance.care_line_template.changed` (L13). Proposta: o contrato registra "sem step-up".
5. **M5 — Nome do interruptor e ambientes. Aberta (só registrar):** "Linhas de cuidado" no maintenance e no dashboard; `care_lines` em todos os ambientes, sem simulado (plano do api, "Valores fixados" 1).
6. **M6 — Campos avançados sem edição nesta tela. Nova.** `all_of`, `code_filter`, `applies_to` (L1) e as identidades de gênero e `declared` (L2) vêm da versão de partida e voltam iguais na versão nova; a tela os mostra em texto e não os edita (a revisão anual que mudar um deles passa pelo YAML do api, `config/care_lines/catalog-base.yml`, até uma tela própria). Proposta: o contrato registra essa limitação.

## Self-review

- **Cobertura (spec §3, §4 "Plataforma", §6 "Maintenance", §9 itens 4 e 5, §10, §11; contrato §1, §8, §10):** o interruptor pelo mecanismo genérico, exigindo `clinical_record` com a frase existente — Task 1 (Review Focus 1); todos os ambientes, sem simulado — Task 1 e M5; catálogo-base com versões via GraphQL — Tasks 2 (SDL e regras), 3 (vigentes e histórico) e 4 (versão nova); versão nova não muda as cidades — Tasks 3 e 4 (aviso; Review Focus 4); pontos para a prévia — Task 2 e M2; campos avançados preservados — Tasks 2, 3 e 4 (M6); recusar em vez de cortar, vírgula decimal — Tasks 2 e 4 (Review Focus 2); fontes com código e as não acompanhadas — Tasks 2 e 4 (Review Focus 3); api sem o 23 — Task 3 (Review Focus 5); SDL igual ao do api — Task 5; prova no navegador — Task 5; o catálogo público com F-23.1..F-23.8 como task final, com a F-23.6 em validação até a comparação com a extração oficial (risco da spec §11) — Task 6; cards — Tasks 0 e 5.
- **Placeholders:** nenhum código faltando. Valores lidos na execução: ids dos cards e da opção In Progress (Task 0); o SDL do api (Tasks 0 e 5, com a regra escrita para diferença); status dos cards F-23.x e o texto do catálogo marcado como rascunho a conferir contra o entregue (Task 6).
- **Tipos e nomes:** `TemplateView`, `PracticeJson`, `EligibilityJson`, `TemplateDraft`, `PracticeDraft`, `TemplateVariables` (Task 2) batem com os campos dos documentos `CareLineTemplates` e `UpsertCareLineTemplate` (Tasks 3–4), com o SDL da Task 2 e com a Task 4 do plano do api; `draftToVariables` devolve exatamente as variáveis da mutation (`key`, `name`, `indicator`, `declaredCondition`, `eligibility`, `practices`); `keyOrder` (Task 2) é o usado na Task 3; `parseDecimal`, `decimalText`, `optionsWith`, `DOT_IN_NUMBER` existem em `labTemplates.ts`; `muted`, `row`, `grid`, `labelStyle`, `inputStyle`, `checkStyle` são exportados de `LabTemplates.tsx`; `fmtWhen` existe em `src/lib/analytics.ts` ("05/10/2026, 10:00" para `2026-10-05T13:00:00Z`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem no topo de `FeaturesTab.test.tsx`; `CareLineTemplates.fixtures.tsx` (Task 3, sem testes) exporta `reply`, `renderScreen` e os dados usados pelos dois arquivos de teste.
- **Review Focus:** 1 — Task 1 ("sem clinical_record: ligada, falta o prontuário, com a frase do módulo 19"); 2 — Task 2 ("pontos: vírgula decimal, de 0 a 100, somando 100; ponto recusado") e Task 4 ("pontos com ponto são recusados e ficam no campo; nada é enviado"); 3 — Task 2 ("fonte: monta e desmonta o código; desconhecida continua", "variáveis: JSON em snake_case, avançados preservados, vazios fora") e Task 4 ("exame pede o SIGTAP e manda exam:<código>; visita do ACS some o código; avançados voltam iguais"); 4 — Task 4 ("publica versão nova a partir da vigente, com o aviso de que as cidades adotam"); 5 — Task 3 ("api sem o 23: só esta tela cai, com a frase") e Task 4 ("recusa do api vira frase com o campo; código novo cru; o formulário fica").
