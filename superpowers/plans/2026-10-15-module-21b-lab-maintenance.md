# Módulo 21b — Laboratório municipal (maintenance e catálogo público) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21b** na branch `feat/mod-21b-lab` (worktree `apps/api/.claude/mod21b`, servidor de dev na porta **3042**) com `municipal_lab`, `lab_hl7_inbound` e `fake_middleware` em `Platform::Features::CATALOG`, e com `labExamTemplates`, `upsertLabExamTemplate` e `upsertLabAnalyteTemplate` no `Maintenance::Schema` no formato da Divergência M1. Contrato §13: contracts → api → dashboard → **maintenance** → catálogo público. As Tasks 0–3 não dependem do api (testes com `fetch` falso, sem schema novo); a Task 4 precisa do SDL do api para o `codegen`; a prova da Task 6 precisa do api rodando. A Task 7 (catálogo público) vem **por último**, depois da prova e das entregas de api, dashboard e maintenance.

**Goal:** Lado maintenance de F-21.14: os três interruptores do 21b aparecem e são ligados pelo mecanismo genérico, com nome e pré-requisitos em português e o aviso do middleware simulado; a nova tela **Laboratório** mantém o catálogo-base de exames de laboratório da plataforma (exames, componentes e faixas de referência). E as F-21.14..F-21.24 no **catálogo público** (`apps/catalog`, `src/data/funcionalidades.json` como fonte única).

**Architecture:** Interruptores: nada muda no liga/desliga; entram rótulos em `src/lib/features.ts` e o aviso do middleware simulado em `src/lib/lab.ts`, desenhado na `FeaturesTab` logo abaixo do aviso do laboratório simulado do 21a e no mesmo molde (pelo estado **ligado**). Catálogo-base: tela própria no trilho do Shell (como Medicamentos, porque vale para todas as cidades do ambiente), com consulta própria (`labExamTemplates`) — contra um api sem o 21b só esta tela cai, com a ordem de deploy. Regras de exibição e de conversão (vírgula decimal, valor desconhecido cru) ficam puras em `src/lib/labTemplates.ts`; a lista e o formulário do exame em `src/screens/LabTemplates.tsx`; o formulário do componente com a tabela de faixas (dentro de `<TableScroll>`) em `src/screens/LabAnalyteForm.tsx`. Sem step-up nas duas mutations (Divergência M4).

**Tech Stack:** maintenance — React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset, `schema.graphql` puxado de `Maintenance::Schema.to_definition`), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`). catálogo — React 18 + Vite, Vitest 2 + Testing Library + jest-dom.

**Spec:** `docs/superpowers/specs/2026-10-15-module-21b-municipal-lab-design.md` (§3 "Interruptores", §4 "Plataforma (catálogo-base)", §7 "Maintenance", §10 itens 4 e 5, §11 F-21.14 e a tabela inteira para o catálogo, §12), `docs/adr/0037.md` e o contrato `docs/superpowers/plans/2026-10-15-module-21b-lab-contracts.md` §1, §10 e §13 (as diferenças estão em "Divergências propostas ao contrato"). Lições do 21a: contrato `2026-10-13-module-21a-exams-contracts.md` §15 (A25 — aviso do simulado no padrão do PSC/SNCR).

## Global Constraints

- Interruptores (contrato §10; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`, que transforma `feature:<x>` em `<x>_disabled`):
  - `municipal_lab` com `requires: ["feature:exam_tracking"]`; faltando, `missing: ["exam_tracking_disabled"]` (frase já existente, do 21a);
  - `lab_hl7_inbound` com `requires: ["feature:municipal_lab"]`; faltando, `missing: ["municipal_lab_disabled"]`;
  - `fake_middleware` com `requires: ["feature:lab_hl7_inbound"]`, `environments: SIMULATION_ENVS`; faltando, `missing: ["lab_hl7_inbound_disabled"]`; **em produção não aparece no catálogo** e a mutation responde `unknown_feature`.
  - Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados). Nenhuma mudança no fluxo de `setCityFeature`.
- Aviso do middleware simulado: pelo estado **ligado** de `fake_middleware`, na aba (`role="note"`, nome "modo do middleware"), no molde do aviso do laboratório simulado do 21a. Texto: `FAKE_MIDDLEWARE_NOTICE`. Desligado ou ausente: nenhum aviso e nada diz "MIDDLEWARE SIMULADO".
- Catálogo-base (contrato §10; spec §4): `labExamTemplates` (lista, com componentes e faixas) e as mutations `upsertLabExamTemplate` / `upsertLabAnalyteTemplate` → `{ ok, errors { path message } }`, no SDL da Divergência M1. Sem step-up (Divergência M4). Sem exclusão: exame sai de uso por `active: false`.
- Faixa de referência: `{ sex: male|female|any, ageMinDays, ageMaxDays, low|null, high|null, text|null }`; idade em dias, `43800` (120 anos) como teto de "sem limite" (Divergência M5). Decimal digitado com **vírgula**; "150.000" é ambíguo e é recusado antes de enviar.
- Mudar o catálogo-base **não** muda exame já adotado por cidade nem laudo (spec §4: o laboratório adota uma cópia e cada ajuste vira versão imutável). A tela diz isso.
- Nenhum token de middleware, mensagem HL7, CPF, CNS, valor de resultado ou PDF passa pelo maintenance.
- Valor desconhecido (api mais novo: setor, tipo de valor, sexo, código de erro) aparece cru — nunca `undefined`/`null` — e salvar **não** o troca em silêncio.
- Interface (ciclo de interface, subprojeto 5 — tema rota do maintenance): componentes existentes (`Panel`, `DataTable`, `Field`, `Button`, `Tag`, `EmptyState`, `ErrorState`), sem redesign; cores só por papel do tema (`var(--warn)`, `var(--ok)`, `var(--ink3)`, `var(--rule)`…), a guarda `src/theme/guards.test.ts` roda na suíte; toda tabela larga dentro de `<TableScroll>` (o `DataTable` já embrulha; a tabela de faixas é crua e ganha guarda). Achado só visual não vira tarefa.
- Catálogo público: **o JSON é a fonte única**; toda funcionalidade tem `description` (2–3 frases, linguagem de gestor, só fato confirmado no que foi entregue); nada provado só com simulador aparece como "Disponível e verificado" sem nota; o `package-lock.json` reescrito pelo `npm install` do container `catalog-dev` nunca é commitado; merge manual, com a CI verde e o branch atualizado.
- Nunca `git add -A` nem `git add .`: adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push, PR e tag só com autorização explícita do usuário.
- Primeira ação da execução: o card F-21.14 do board #1 em In Progress (Task 0); Done só quando api e maintenance do F-21.14 estiverem entregues.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod21b`, `C=$R/.worktrees/catalog-mod21b`, `A=$R/apps/api/.claude/mod21b`.
- Worktrees de front **fora de `apps/`** (o compose monta `./apps/<app>:/app` e um segundo React derruba o Vite do container): maintenance em `$M`, branch `feat/mod-21b-lab`; catálogo em `$C`, branch `content/module-21b-catalog`. Caminhos de arquivo das tasks são relativos ao worktree da task.
- maintenance — testes: `cd $M && npx vitest run <arquivos>`; tipos: `npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. Ambiente de teste: `jsdom`, `globals: false` (todo teste de componente chama `cleanup` no `afterEach`), **sem jest-dom** (use `not.toBeNull()`, `toBeNull()`, `.textContent`, `.value`).
- O `graphql()` gerado só conhece documentos que o `codegen` viu: **todo documento novo exige `npm run codegen` antes de rodar o teste da tela** (sem isso o `print` recebe `{}` e o teste quebra por motivo errado).
- Schema: `schema.graphql` vem de `Maintenance::Schema.to_definition` do **worktree do api do 21b** (Task 4, Step 1), nunca escrito à mão.
- O api do 21b roda em dev na **3042** (plano do api), na rede `rota-saude_default`. Login do maintenance: `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local` semeada pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **Plaquetas digitadas "150.000" (milhar no costume brasileiro).** Lido como `150.0`, a faixa de referência vira 150–450000 e todo laudo de toda cidade que adotar sai errado. "150.000" é recusado antes de enviar, com a frase de como digitar; "13,5" vira `13.5`. Testes: Task 3, "ponto de milhar é ambíguo e é recusado"; Task 5, "novo componente: ponto de milhar é recusado antes de enviar".
2. **O api recusa (SIGTAP repetido, faixas inválidas) depois de a pessoa digitar dez faixas.** A frase aparece em português e **nada do que foi digitado some**. Testes: Task 4, "recusa do api: mostra a frase e mantém o que foi digitado"; Task 5, "recusa do api (faixas inválidas): mostra a frase e mantém as faixas digitadas".
3. **Valor que esta tela não conhece (setor, tipo de valor ou sexo novo no api).** Aparece cru e salvar devolve o mesmo valor — um `<select>` sem a opção trocaria em silêncio para a primeira. Testes: Task 3, "optionsWith mantém o valor desconhecido"; Task 4, "setor desconhecido: aparece cru e salvar não o troca"; Task 5, "tipo e sexo desconhecidos aparecem crus e não são trocados".
4. **Produção, e staging com os dois simulados ligados.** Em produção `fake_middleware` não está no catálogo: nada quebra e nada diz "MIDDLEWARE SIMULADO". Com `fake_lab` e `fake_middleware` ligados, os dois avisos aparecem, cada um com seu nome. Testes: Task 2, "catálogo sem fake_middleware (produção)…" e "laboratório e middleware simulados ligados: dois avisos".
5. **Maintenance publicado antes do api do 21b.** Só a tela Laboratório cai, explicando a ordem de deploy; o Shell e as outras telas seguem. Teste: Task 4, "api sem o 21b: explica a ordem de deploy".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card F-21.14, conferência do main e do api do 21b, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/lab.ts`, `src/lib/lab.test.ts` | nomes das três chaves, frases dos dois pré-requisitos novos, aviso do middleware simulado | 1 |
| `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md` | o aviso na aba; os três interruptores pelo fluxo genérico | 2 |
| `src/lib/labTemplates.ts`, `src/lib/labTemplates.test.ts` | rótulos de setor/tipo/sexo, vírgula decimal, idade, resumo das faixas, conversão do rascunho, frases de erro | 3 |
| `schema.graphql`, `src/gql/*`, `src/screens/LabTemplates.tsx`, `src/screens/LabTemplates.test.tsx`, `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx`, `README.md` | tela Laboratório: lista, formulário do exame, componentes (leitura) | 4 |
| `src/screens/LabAnalyteForm.tsx`, `src/screens/LabAnalyteForm.test.tsx`, `src/screens/LabTemplates.tsx`, `src/screens/LabTemplates.test.tsx`, `src/theme/guards.test.ts`, `src/gql/*` | formulário do componente com as faixas de referência | 5 |
| — | suíte, codegen:check, build, revisão, prova no navegador | 6 |
| catálogo: `src/data/funcionalidades.json`, `src/data/catalog.test.ts` | F-21.14..F-21.24 no módulo 21 | 7 |

**Estratégia de teste:** funções puras com tabela de casos (Tasks 1 e 3); os três interruptores pelo fluxo **existente** de dois cliques da `FeaturesTab`, com `fetch` falso que responde por nome de operação, e os avisos ligado, desligado e ausente (Task 2); telas com `fetch` falso roteado por operação — lista, criar, editar, recusa do api, recusa do envelope, api antigo, valor desconhecido — conferindo as **variáveis** exatas enviadas (Tasks 4 e 5); guarda de `<TableScroll>` na tabela de faixas (Task 5); prova no navegador contra o api do 21b (Task 6). Catálogo: `validateCatalog` e o teste do módulo 21 (Task 7).

---

### Task 0: Card, conferência do main e do api do 21b, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`068b4a1` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`); worktree do api `$A`.
- Produces: card F-21.14 em In Progress; worktree `$M` na branch `feat/mod-21b-lab`; a base anotada.

- [ ] **Step 1: Mova o card F-21.14 para In Progress**

Um comando `gh` por chamada, cada um começando com `gh`:

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | startswith("F-21.14 ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Se já estiver em In Progress (o F-21.14 é também do api), siga. Senão, com o `id` do item e o id da opção "In Progress":

```bash
gh project item-edit --id <PVTI_do_F-21.14> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: o item com `status: "In Progress"`. Se o card não existir, **pare e reporte**.

- [ ] **Step 2: Confira o main e o api do 21b**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod21b
$G -C $R/apps/maintenance fetch origin
$G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n \
  '  city_unreachable: "banco da cidade inalcançável — não deu para conferir"\|  fake_lab: "Laboratório simulado (desenvolvimento)"\|municipal_lab\|fake_middleware'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n \
  'import { FAKE_LAB_NOTICE, fakeLabOn } from "../lib/exams";\|  const labSimulated = city ? fakeLabOn(city.features) : false;\|{FAKE_LAB_NOTICE}'
$G -C $R/apps/maintenance show origin/main:src/screens/Shell.tsx | grep -n 'export type Screen\|{ key: "medications", label: "Medicamentos" },\|{screen === "medications" && <MedicationCatalog />}'
grep -n -E 'key: "(municipal_lab|lab_hl7_inbound|fake_middleware)"' -A3 $A/app/services/platform/features.rb
grep -rln -E 'upsert_lab_exam_template|upsert_lab_analyte_template|lab_exam_templates' $A/app/graphql/maintenance
```

Expected:
- em `features.ts`: as duas âncoras e **nenhuma** linha com `municipal_lab` ou `fake_middleware`;
- em `FeaturesTab.tsx`: as três âncoras; em `Shell.tsx`: as três. Se mudaram, aplique a mesma intenção sobre o texto real;
- no api do 21b: as três entradas com os `requires` do Global Constraints e `environments: SIMULATION_ENVS` no `fake_middleware`; e os arquivos das duas mutations e da consulta. Faltando o GraphQL, as Tasks 0–3 seguem; **a Task 4 espera** o api (não escreva o SDL à mão).

- [ ] **Step 3: Crie o worktree fora de `apps/` e instale**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod21b -b feat/mod-21b-lab origin/main
cd $R/.worktrees/maintenance-mod21b && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 6 espera a base mais **4 arquivos de teste novos** (`src/lib/lab.test.ts`, `src/lib/labTemplates.test.ts`, `src/screens/LabTemplates.test.tsx`, `src/screens/LabAnalyteForm.test.tsx`).

---

### Task 1: Nomes dos interruptores, frases dos pré-requisitos e o aviso do middleware simulado

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/lib/lab.ts`
- Test: `src/lib/lab.test.ts`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (existentes).
- Produces:
  - `featureLabel("municipal_lab")` → "Laboratório municipal (requisição, amostra, análise e laudo)"; `featureLabel("lab_hl7_inbound")` → "Entrada de resultados de aparelhos por HL7 (middleware)"; `featureLabel("fake_middleware")` → "Middleware simulado (desenvolvimento)";
  - `missingText("municipal_lab_disabled")` → "laboratório municipal (municipal_lab) desligado"; `missingText("lab_hl7_inbound_disabled")` → "entrada de resultados por HL7 (lab_hl7_inbound) desligada";
  - em `src/lib/lab.ts`:
    ```ts
    export const FAKE_MIDDLEWARE_KEY = "fake_middleware";
    export const FAKE_MIDDLEWARE_NOTICE: string; // "Esta cidade recebe resultados do MIDDLEWARE SIMULADO — … sem validade clínica (fora de produção) …"
    export function fakeMiddlewareOn(features: { key: string; enabled: boolean }[]): boolean;
    ```

- [ ] **Step 1: Write the failing test**

Em `src/lib/features.test.ts`, no fim do arquivo:

```ts
describe("módulo 21b", () => {
  it("as três chaves do 21b têm nome em português", () => {
    expect(featureLabel("municipal_lab")).toBe("Laboratório municipal (requisição, amostra, análise e laudo)");
    expect(featureLabel("lab_hl7_inbound")).toBe("Entrada de resultados de aparelhos por HL7 (middleware)");
    expect(featureLabel("fake_middleware")).toBe("Middleware simulado (desenvolvimento)");
  });

  it("pré-requisitos do 21b têm frase", () => {
    expect(missingText("municipal_lab_disabled")).toBe("laboratório municipal (municipal_lab) desligado");
    expect(missingText("lab_hl7_inbound_disabled")).toBe("entrada de resultados por HL7 (lab_hl7_inbound) desligada");
    // municipal_lab exige exam_tracking: a frase do 21a já existe.
    expect(missingText("exam_tracking_disabled")).toBe("rastreamento de exames (exam_tracking) desligado");
    expect(missingSummary([ "municipal_lab_disabled", "lab_hl7_inbound_disabled" ]))
      .toBe("laboratório municipal (municipal_lab) desligado; entrada de resultados por HL7 (lab_hl7_inbound) desligada");
  });
});
```

Crie `src/lib/lab.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { FAKE_MIDDLEWARE_KEY, FAKE_MIDDLEWARE_NOTICE, fakeMiddlewareOn } from "./lab";
import { FAKE_LAB_NOTICE } from "./exams";

describe("fakeMiddlewareOn", () => {
  it("só pelo estado ligado de fake_middleware; ausente (produção) é desligado", () => {
    expect(fakeMiddlewareOn([ { key: FAKE_MIDDLEWARE_KEY, enabled: true } ])).toBe(true);
    expect(fakeMiddlewareOn([ { key: FAKE_MIDDLEWARE_KEY, enabled: false } ])).toBe(false);
    expect(fakeMiddlewareOn([ { key: "lab_hl7_inbound", enabled: true }, { key: "fake_lab", enabled: true } ])).toBe(false);
    expect(fakeMiddlewareOn([])).toBe(false);
  });

  it("o aviso diz simulado, sem validade e que nada sai sem o liberador; e não é o do laboratório do 21a", () => {
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/MIDDLEWARE SIMULADO/);
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/sem validade clínica/);
    expect(FAKE_MIDDLEWARE_NOTICE).toMatch(/liberador/);
    expect(FAKE_MIDDLEWARE_NOTICE).not.toBe(FAKE_LAB_NOTICE);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/lib/lab.test.ts`
Expected: FAIL — `featureLabel("municipal_lab")` é `null`, `missingText("municipal_lab_disabled")` volta cru e `./lab` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, em `MISSING_LABELS`, troque

```ts
  // O api não conseguiu ler o banco da cidade: usable vem false sem que se
```

por

```ts
  // Módulo 21b: municipal_lab exige exam_tracking (frase acima);
  // lab_hl7_inbound exige municipal_lab; fake_middleware exige lab_hl7_inbound.
  municipal_lab_disabled: "laboratório municipal (municipal_lab) desligado",
  lab_hl7_inbound_disabled: "entrada de resultados por HL7 (lab_hl7_inbound) desligada",
  // O api não conseguiu ler o banco da cidade: usable vem false sem que se
```

e, em `FEATURE_LABELS`, troque

```ts
  fake_lab: "Laboratório simulado (desenvolvimento)"
};
```

por

```ts
  fake_lab: "Laboratório simulado (desenvolvimento)",
  // Módulo 21b (ADR 0037): fake_middleware só existe fora de produção.
  municipal_lab: "Laboratório municipal (requisição, amostra, análise e laudo)",
  lab_hl7_inbound: "Entrada de resultados de aparelhos por HL7 (middleware)",
  fake_middleware: "Middleware simulado (desenvolvimento)"
};
```

Crie `src/lib/lab.ts`:

```ts
// Interruptores do módulo 21b no maintenance (ADR 0037; contrato §10). O
// liga/desliga e os pré-requisitos são o mecanismo genérico (features.ts);
// aqui só o aviso do middleware simulado, no molde do laboratório simulado
// do 21a (exams.ts). Nenhum token de middleware, mensagem HL7 ou valor de
// resultado passa pelo maintenance.
export const FAKE_MIDDLEWARE_KEY = "fake_middleware";
export const FAKE_MIDDLEWARE_NOTICE =
  "Esta cidade recebe resultados do MIDDLEWARE SIMULADO — as amostras recebidas no laboratório municipal ganham valores de exemplo pela HL7, sem validade clínica (fora de produção); nenhum valor sai do laboratório sem o liberador.";

// Pelo estado LIGADO (não pelo "utilizável"): ligado, o fake-middleware passa
// a enviar assim que a entrada HL7 estiver utilizável. Sem o interruptor no
// catálogo (produção) é o mesmo que desligado.
export function fakeMiddlewareOn(features: { key: string; enabled: boolean }[]): boolean {
  return features.some((f) => f.key === FAKE_MIDDLEWARE_KEY && f.enabled);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/features.test.ts src/lib/lab.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos de `features.test.ts` seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/lib/lab.ts src/lib/lab.test.ts
$G commit -m "feat: name the municipal lab switches and their prerequisites

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: O aviso do middleware na aba e os três interruptores pelo fluxo genérico

**Files:**
- Modify: `src/screens/FeaturesTab.tsx`, `README.md`
- Test: `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `FAKE_MIDDLEWARE_NOTICE`, `fakeMiddlewareOn` (Task 1); `FAKE_LAB_NOTICE` (existente); rótulos e frases (Task 1); `FeaturesTab`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature`, `LEDI_OFF` (existentes em `FeaturesTab.test.tsx`).
- Produces: na aba, `<p role="note" aria-label="modo do middleware">{FAKE_MIDDLEWARE_NOTICE}</p>` quando `fake_middleware` está ligado.

- [ ] **Step 1: Write the failing test**

Em `src/screens/FeaturesTab.test.tsx`, acrescente ao bloco de imports `import { FAKE_MIDDLEWARE_NOTICE } from "../lib/lab";` e, no fim do arquivo:

```tsx
describe("FeaturesTab — laboratório municipal (módulo 21b)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const LAB_OFF: Feature = {
    key: "municipal_lab", description: "Laboratório municipal: fluxo analítico", enabled: false, usable: false,
    missing: [ "exam_tracking_disabled" ], changedAt: null, changedBy: null
  };
  const HL7_OFF: Feature = {
    key: "lab_hl7_inbound", description: "Entrada HL7 do middleware", enabled: false, usable: false,
    missing: [ "municipal_lab_disabled" ], changedAt: null, changedBy: null
  };
  const MW_OFF: Feature = {
    key: "fake_middleware", description: "Middleware simulado do compose de dev", enabled: false, usable: false,
    missing: [ "lab_hl7_inbound_disabled" ], changedAt: null, changedBy: null
  };
  const MW_ON: Feature = { ...MW_OFF, enabled: true, changedAt: "2026-10-15T13:00:00Z", changedBy: "dev@local" };
  const FAKE_LAB_ON: Feature = {
    key: "fake_lab", description: "Laboratório simulado do compose de dev", enabled: true, usable: true,
    missing: [], changedAt: "2026-10-15T12:00:00Z", changedBy: "dev@local"
  };

  it("municipal_lab pelo mecanismo genérico: nome em português, falta o rastreamento e liga com dois cliques", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...LAB_OFF, enabled: true, changedAt: "2026-10-15T13:00:00Z", changedBy: "dev@local" };
    let listed = [ LAB_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("municipal_lab")).not.toBeNull();
    expect(screen.getByText("Laboratório municipal (requisição, amostra, análise e laudo)")).not.toBeNull();
    expect(screen.getByText("rastreamento de exames (exam_tracking) desligado")).not.toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar municipal_lab" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "municipal_lab", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "municipal_lab ligada, mas ainda não utilizável — falta: rastreamento de exames (exam_tracking) desligado."
    );
  });

  it("os pré-requisitos do 21b aparecem em português na lista", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ HL7_OFF, MW_OFF ])));
    renderTab();
    expect(await screen.findByText("lab_hl7_inbound")).not.toBeNull();
    expect(screen.getByText("Entrada de resultados de aparelhos por HL7 (middleware)")).not.toBeNull();
    expect(screen.getByText("Middleware simulado (desenvolvimento)")).not.toBeNull();
    expect(screen.getByText("laboratório municipal (municipal_lab) desligado")).not.toBeNull();
    expect(screen.getByText("entrada de resultados por HL7 (lab_hl7_inbound) desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/_disabled|undefined|null/);
    expect(screen.queryByRole("note", { name: "modo do middleware" })).toBeNull();
  });

  it("ligar fake_middleware por dois cliques: após o refetch a aba avisa", async () => {
    const user = userEvent.setup();
    let listed = [ MW_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ MW_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: MW_ON } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("fake_middleware")).not.toBeNull();
    expect(screen.queryByRole("note", { name: "modo do middleware" })).toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(await screen.findByRole("button", { name: "confirmar: ligar fake_middleware" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "fake_middleware", enabled: true });
    expect((await screen.findByRole("note", { name: "modo do middleware" })).textContent).toBe(FAKE_MIDDLEWARE_NOTICE);
  });

  it("laboratório e middleware simulados ligados: dois avisos, cada um com seu nome", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ FAKE_LAB_ON, MW_ON ])));
    renderTab();
    expect((await screen.findByRole("note", { name: "modo do middleware" })).textContent).toBe(FAKE_MIDDLEWARE_NOTICE);
    expect(screen.getByRole("note", { name: "modo do laboratório" }).textContent).toBe(FAKE_LAB_NOTICE);
  });

  it("catálogo sem fake_middleware (produção): nada quebra e nada diz middleware simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      { ...LAB_OFF, enabled: true, usable: true, missing: [] },
      { ...HL7_OFF, enabled: true, usable: true, missing: [] },
      LEDI_OFF
    ])));
    renderTab();
    expect(await screen.findByText("lab_hl7_inbound")).not.toBeNull();
    expect(screen.queryByText("fake_middleware")).toBeNull();
    expect(screen.queryByRole("note", { name: "modo do middleware" })).toBeNull();
    expect(document.body.textContent).not.toMatch(/MIDDLEWARE SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx`
Expected: FAIL — os testes "ligar fake_middleware…" e "laboratório e middleware simulados ligados…" não acham `role="note"` com o nome "modo do middleware" (os outros três já passam com a Task 1).

- [ ] **Step 3: Write minimal implementation**

Em `src/screens/FeaturesTab.tsx`:

1. Depois de `import { FAKE_LAB_NOTICE, fakeLabOn } from "../lib/exams";`, acrescente

```tsx
import { FAKE_MIDDLEWARE_NOTICE, fakeMiddlewareOn } from "../lib/lab";
```

2. Depois de `  const labSimulated = city ? fakeLabOn(city.features) : false;`, acrescente

```tsx
  // Módulo 21b: aviso pelo estado LIGADO de fake_middleware (ausente em produção).
  const middlewareSimulated = city ? fakeMiddlewareOn(city.features) : false;
```

3. Logo depois do bloco do aviso do laboratório (o `{labSimulated && ( … {FAKE_LAB_NOTICE} … )}`), acrescente

```tsx
            {middlewareSimulated && (
              <p role="note" aria-label="modo do middleware" style={{ margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" }}>
                {FAKE_MIDDLEWARE_NOTICE}
              </p>
            )}
```

Em `README.md`, logo antes do parágrafo **Documentos clínicos (módulo 19c).**, acrescente:

```markdown
**Laboratório municipal (módulo 21b).** Os interruptores `municipal_lab`
("Laboratório municipal", exige `exam_tracking`), `lab_hl7_inbound` ("Entrada
de resultados de aparelhos por HL7", exige `municipal_lab`) e
`fake_middleware` ("Middleware simulado", exige `lab_hl7_inbound`) aparecem na
mesma lista e ligam como os outros, com o que falta em português.
`fake_middleware` só existe fora de produção; ligado, a aba avisa que as
amostras recebidas ganham valores de exemplo pela HL7, sem validade clínica.
Tokens de middleware, exames adotados, liberadores e motivos de rejeição são
do admin municipal, no dashboard; nada disso passa por aqui. Ordem de deploy:
api → dashboard → maintenance; contra um api sem o 21b as chaves só não
aparecem.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/screens/FeaturesTab.test.tsx src/theme/guards.test.ts && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da aba seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: warn in the features tab when the simulated middleware is on

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Regras do catálogo-base (rótulos, vírgula decimal, faixas e erros)

**Files:**
- Create: `src/lib/labTemplates.ts`
- Test: `src/lib/labTemplates.test.ts`

**Interfaces:**
- Consumes: nada de tasks anteriores.
- Produces (usado pelas Tasks 4 e 5):
  ```ts
  export type RangeView = { sex: string; ageMinDays: number; ageMaxDays: number; low?: number | null; high?: number | null; text?: string | null };
  export type AnalyteView = { id: string; key: string; name: string; loinc?: string | null; unit?: string | null; valueType: string; choices: string[]; decimals: number; position: number; referenceRanges: RangeView[] };
  export type TemplateView = { id: string; sigtapCode: string; name: string; sector: string; material: string; container: string; suggestedMethod?: string | null; preparation?: string | null; active: boolean; analytes: AnalyteView[] };
  export type RangeInput = { sex: string; ageMinDays: number; ageMaxDays: number; low: number | null; high: number | null; text: string | null };
  export type RangeDraft = { sex: string; ageMinDays: string; ageMaxDays: string; low: string; high: string; text: string };
  export const SECTORS: readonly ["biochemistry", "hematology", "immunology", "urinalysis", "parasitology", "microbiology"];
  export const VALUE_TYPES: readonly ["numeric", "text", "choice"];
  export const SEXES: readonly ["any", "female", "male"];
  export const UNBOUNDED_AGE_DAYS = 43800;
  export const EMPTY_RANGE: RangeDraft;
  export function sectorText(sector: string): string;
  export function valueTypeText(valueType: string): string;
  export function sexText(sex: string): string;
  export function optionsWith(known: readonly string[], current: string): string[];
  export function orNull(raw: string): string | null;
  export function decimalText(n: number | null | undefined): string;
  export function parseDecimal(raw: string): number | null | undefined; // undefined = inválido
  export function ageText(days: number): string;
  export function rangeText(r: RangeView): string;
  export function rangesSummary(ranges: RangeView[]): string;
  export function rangeToDraft(r: RangeView): RangeDraft;
  export function draftsToRanges(drafts: RangeDraft[]): { ranges: RangeInput[] } | { error: string };
  export function upsertErrorText(errors: { path?: string | null; message: string }[]): string;
  ```

- [ ] **Step 1: Write the failing test**

Crie `src/lib/labTemplates.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  EMPTY_RANGE, SECTORS, ageText, decimalText, draftsToRanges, optionsWith, orNull, parseDecimal, rangeText, rangeToDraft,
  rangesSummary, sectorText, sexText, upsertErrorText, valueTypeText
} from "./labTemplates";

describe("rótulos", () => {
  it("setor, tipo de valor e sexo em português; desconhecido aparece cru", () => {
    expect(SECTORS.map(sectorText)).toEqual([ "Bioquímica", "Hematologia", "Imunologia", "Urinálise", "Parasitologia", "Microbiologia" ]);
    expect(sectorText("cytology")).toBe("cytology");
    expect(valueTypeText("numeric")).toBe("numérico");
    expect(valueTypeText("text")).toBe("texto");
    expect(valueTypeText("choice")).toBe("lista de opções");
    expect(valueTypeText("ratio")).toBe("ratio");
    expect(sexText("any")).toBe("qualquer");
    expect(sexText("female")).toBe("feminino");
    expect(sexText("male")).toBe("masculino");
    expect(sexText("intersex")).toBe("intersex");
  });

  it("optionsWith mantém o valor desconhecido (salvar não o troca)", () => {
    expect(optionsWith([ "a", "b" ], "b")).toEqual([ "a", "b" ]);
    expect(optionsWith([ "a", "b" ], "z")).toEqual([ "a", "b", "z" ]);
  });

  it("orNull apara e transforma vazio em null", () => {
    expect(orNull("  g/dL ")).toBe("g/dL");
    expect(orNull("   ")).toBeNull();
  });
});

describe("números", () => {
  it("vírgula ou ponto decimal viram número; vazio é null", () => {
    expect(parseDecimal("13,5")).toBe(13.5);
    expect(parseDecimal(" 0,6 ")).toBe(0.6);
    expect(parseDecimal("13.5")).toBe(13.5);
    expect(parseDecimal("150000")).toBe(150000);
    expect(parseDecimal("-2,25")).toBe(-2.25);
    expect(parseDecimal("")).toBeNull();
    expect(parseDecimal("  ")).toBeNull();
  });

  it("ponto de milhar é ambíguo e é recusado", () => {
    expect(parseDecimal("150.000")).toBeUndefined();
    expect(parseDecimal("1.000")).toBeUndefined();
    expect(parseDecimal("150.000,5")).toBeUndefined();
    expect(parseDecimal("abc")).toBeUndefined();
    expect(parseDecimal("13,5 g/dL")).toBeUndefined();
  });

  it("decimalText com vírgula e sem separador de milhar; nulo vira —", () => {
    expect(decimalText(13.5)).toBe("13,5");
    expect(decimalText(150000)).toBe("150000");
    expect(decimalText(12)).toBe("12");
    expect(decimalText(null)).toBe("—");
    expect(decimalText(undefined)).toBe("—");
  });

  it("idade em anos quando exata, senão em dias", () => {
    expect(ageText(0)).toBe("0");
    expect(ageText(6570)).toBe("18 a");
    expect(ageText(43800)).toBe("120 a");
    expect(ageText(28)).toBe("28 d");
    expect(ageText(400)).toBe("400 d");
  });
});

describe("faixas de referência", () => {
  const MALE = { sex: "male", ageMinDays: 6570, ageMaxDays: 43800, low: 13.5, high: 17.5, text: null };

  it("rangeText: sexo, idades e valores; só um limite; só texto; valor e texto", () => {
    expect(rangeText(MALE)).toBe("masculino, 18 a–120 a: 13,5–17,5");
    expect(rangeText({ ...MALE, sex: "any", ageMinDays: 0, low: null, high: 200 })).toBe("qualquer, 0–120 a: ≤ 200");
    expect(rangeText({ ...MALE, high: null })).toBe("masculino, 18 a–120 a: ≥ 13,5");
    expect(rangeText({ ...MALE, low: null, high: null, text: "negativo" })).toBe("masculino, 18 a–120 a: negativo");
    expect(rangeText({ ...MALE, text: "jejum" })).toBe("masculino, 18 a–120 a: 13,5–17,5 (jejum)");
    expect(rangeText({ ...MALE, low: null, high: null, text: null })).toBe("masculino, 18 a–120 a: —");
  });

  it("rangesSummary junta com ponto e vírgula; lista vazia diz sem faixa", () => {
    expect(rangesSummary([ MALE, { ...MALE, sex: "female", low: 12, high: 15.5 } ]))
      .toBe("masculino, 18 a–120 a: 13,5–17,5; feminino, 18 a–120 a: 12–15,5");
    expect(rangesSummary([])).toBe("sem faixa");
  });

  it("rangeToDraft escreve com vírgula; draftsToRanges volta ao mesmo número", () => {
    const draft = rangeToDraft(MALE);
    expect(draft).toEqual({ sex: "male", ageMinDays: "6570", ageMaxDays: "43800", low: "13,5", high: "17,5", text: "" });
    expect(draftsToRanges([ draft ])).toEqual({ ranges: [ MALE ] });
  });

  it("faixa nova vazia: qualquer sexo, 0 a 120 anos; texto aparado", () => {
    expect(EMPTY_RANGE).toEqual({ sex: "any", ageMinDays: "0", ageMaxDays: "43800", low: "", high: "", text: "" });
    expect(draftsToRanges([ { ...EMPTY_RANGE, text: "  negativo " } ])).toEqual({
      ranges: [ { sex: "any", ageMinDays: 0, ageMaxDays: 43800, low: null, high: null, text: "negativo" } ]
    });
  });

  it("número inválido ou idade não inteira: erro que diz a faixa e o campo, sem enviar nada", () => {
    expect(draftsToRanges([ rangeToDraft(MALE), { ...EMPTY_RANGE, low: "150.000" } ])).toEqual({
      error: "faixa 2: \"baixo\" não é um número — use vírgula para decimais, sem ponto de milhar"
    });
    expect(draftsToRanges([ { ...EMPTY_RANGE, high: "alto" } ])).toEqual({
      error: "faixa 1: \"alto\" não é um número — use vírgula para decimais, sem ponto de milhar"
    });
    expect(draftsToRanges([ { ...EMPTY_RANGE, ageMaxDays: "1,5" } ])).toEqual({
      error: "faixa 1: idade em dias deve ser um número inteiro"
    });
  });
});

describe("erros das mutations", () => {
  it("códigos conhecidos em português; blank com o campo; desconhecido cru; vazio genérico", () => {
    expect(upsertErrorText([ { path: "sigtapCode", message: "duplicate_sigtap" } ]))
      .toBe("já existe um exame do catálogo-base com este código SIGTAP");
    expect(upsertErrorText([ { path: "sigtapCode", message: "sigtap_unknown" } ]))
      .toBe("código SIGTAP fora da competência vigente da plataforma");
    expect(upsertErrorText([ { path: "referenceRanges", message: "invalid_reference_ranges" } ]))
      .toBe("faixas de referência inválidas — confira idade mínima ≤ máxima e baixo ≤ alto");
    expect(upsertErrorText([ { path: "name", message: "blank" }, { path: "key", message: "duplicate_analyte_key" } ]))
      .toBe("campo obrigatório vazio: name; já existe um componente com esta chave neste exame");
    expect(upsertErrorText([ { path: null, message: "lab_template_locked" } ])).toBe("lab_template_locked");
    expect(upsertErrorText([])).toBe("não foi possível concluir — tente de novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/lib/labTemplates.test.ts`
Expected: FAIL — `./labTemplates` não existe.

- [ ] **Step 3: Write minimal implementation**

Crie `src/lib/labTemplates.ts`:

```ts
// Regras de exibição e de entrada do catálogo-base de laboratório da
// plataforma (módulo 21b, F-21.14; spec §4 "Plataforma"; contrato §10).
// Quem valida de verdade é o api; aqui só se traduz o que ele devolveu e se
// converte o que a pessoa digitou. Valor desconhecido (api mais novo) aparece
// cru e continua selecionado; nulo vira "—".
export type RangeView = {
  sex: string; ageMinDays: number; ageMaxDays: number; low?: number | null; high?: number | null; text?: string | null;
};
export type AnalyteView = {
  id: string; key: string; name: string; loinc?: string | null; unit?: string | null; valueType: string;
  choices: string[]; decimals: number; position: number; referenceRanges: RangeView[];
};
export type TemplateView = {
  id: string; sigtapCode: string; name: string; sector: string; material: string; container: string;
  suggestedMethod?: string | null; preparation?: string | null; active: boolean; analytes: AnalyteView[];
};
export type RangeInput = { sex: string; ageMinDays: number; ageMaxDays: number; low: number | null; high: number | null; text: string | null };
export type RangeDraft = { sex: string; ageMinDays: string; ageMaxDays: string; low: string; high: string; text: string };

// Divergência M3: códigos de setor do contrato.
export const SECTORS = [ "biochemistry", "hematology", "immunology", "urinalysis", "parasitology", "microbiology" ] as const;
const SECTOR_LABELS: Record<string, string> = {
  biochemistry: "Bioquímica", hematology: "Hematologia", immunology: "Imunologia",
  urinalysis: "Urinálise", parasitology: "Parasitologia", microbiology: "Microbiologia"
};
export function sectorText(sector: string): string {
  return SECTOR_LABELS[sector] ?? sector;
}

export const VALUE_TYPES = [ "numeric", "text", "choice" ] as const;
const VALUE_TYPE_LABELS: Record<string, string> = { numeric: "numérico", text: "texto", choice: "lista de opções" };
export function valueTypeText(valueType: string): string {
  return VALUE_TYPE_LABELS[valueType] ?? valueType;
}

export const SEXES = [ "any", "female", "male" ] as const;
const SEX_LABELS: Record<string, string> = { any: "qualquer", female: "feminino", male: "masculino" };
export function sexText(sex: string): string {
  return SEX_LABELS[sex] ?? sex;
}

// Opções de um <select>: as conhecidas e, se o valor atual for desconhecido,
// ele também — sem isso o <select> mostraria a primeira opção e salvar
// trocaria o valor em silêncio.
export function optionsWith(known: readonly string[], current: string): string[] {
  return known.includes(current) ? [ ...known ] : [ ...known, current ];
}

export function orNull(raw: string): string | null {
  const trimmed = raw.trim();
  return trimmed === "" ? null : trimmed;
}

// Divergência M5: idade em dias; 43800 (120 anos) é o "sem limite".
export const UNBOUNDED_AGE_DAYS = 43800;
export const EMPTY_RANGE: RangeDraft = {
  sex: "any", ageMinDays: "0", ageMaxDays: String(UNBOUNDED_AGE_DAYS), low: "", high: "", text: ""
};

// Sem separador de milhar, de propósito: é o mesmo formato que a entrada aceita.
const decimal = new Intl.NumberFormat("pt-BR", { maximumFractionDigits: 6, useGrouping: false });
export function decimalText(n: number | null | undefined): string {
  return typeof n === "number" ? decimal.format(n) : "—";
}

// "13,5" e "13.5" → 13.5; "" → null; inválido → undefined. "150.000" é
// ambíguo (milhar no costume brasileiro, decimal no americano) e é recusado:
// lido como 150, a faixa das plaquetas sairia errada para toda cidade.
export function parseDecimal(raw: string): number | null | undefined {
  const t = raw.trim();
  if (t === "") return null;
  if (/^-?\d{1,3}\.\d{3}$/.test(t)) return undefined;
  if (!/^-?\d+([.,]\d+)?$/.test(t)) return undefined;
  return Number(t.replace(",", "."));
}

export function ageText(days: number): string {
  if (days === 0) return "0";
  return days % 365 === 0 ? `${days / 365} a` : `${days} d`;
}

export function rangeText(r: RangeView): string {
  const hasLow = typeof r.low === "number";
  const hasHigh = typeof r.high === "number";
  const values = hasLow && hasHigh ? `${decimalText(r.low)}–${decimalText(r.high)}`
    : hasLow ? `≥ ${decimalText(r.low)}`
    : hasHigh ? `≤ ${decimalText(r.high)}`
    : null;
  const text = r.text?.trim() ? r.text.trim() : null;
  const shown = values && text ? `${values} (${text})` : values ?? text ?? "—";
  return `${sexText(r.sex)}, ${ageText(r.ageMinDays)}–${ageText(r.ageMaxDays)}: ${shown}`;
}

export function rangesSummary(ranges: RangeView[]): string {
  return ranges.length === 0 ? "sem faixa" : ranges.map(rangeText).join("; ");
}

const draftNumber = (n: number | null | undefined) => (typeof n === "number" ? decimalText(n) : "");

export function rangeToDraft(r: RangeView): RangeDraft {
  return {
    sex: r.sex, ageMinDays: String(r.ageMinDays), ageMaxDays: String(r.ageMaxDays),
    low: draftNumber(r.low), high: draftNumber(r.high), text: r.text ?? ""
  };
}

const RANGE_FIELD_LABELS = { low: "baixo", high: "alto" } as const;

// Converte o que foi digitado. Para na primeira faixa inválida, com o número
// da faixa e o campo; nada é enviado. Regras de conjunto (idade mínima ≤
// máxima, baixo ≤ alto) são do api (invalid_reference_ranges).
export function draftsToRanges(drafts: RangeDraft[]): { ranges: RangeInput[] } | { error: string } {
  const ranges: RangeInput[] = [];
  for (const [ index, d ] of drafts.entries()) {
    const n = index + 1;
    const min = d.ageMinDays.trim();
    const max = d.ageMaxDays.trim();
    if (!/^\d+$/.test(min) || !/^\d+$/.test(max)) return { error: `faixa ${n}: idade em dias deve ser um número inteiro` };
    const low = parseDecimal(d.low);
    const high = parseDecimal(d.high);
    for (const [ field, value ] of [ [ "low", low ], [ "high", high ] ] as const) {
      if (value === undefined) {
        return { error: `faixa ${n}: "${RANGE_FIELD_LABELS[field]}" não é um número — use vírgula para decimais, sem ponto de milhar` };
      }
    }
    ranges.push({
      sex: d.sex, ageMinDays: Number(min), ageMaxDays: Number(max),
      low: low ?? null, high: high ?? null, text: orNull(d.text)
    });
  }
  return { ranges };
}

// Divergência M3: códigos de UserError das duas mutations; desconhecido cru.
const UPSERT_ERRORS: Record<string, string> = {
  sigtap_unknown: "código SIGTAP fora da competência vigente da plataforma",
  duplicate_sigtap: "já existe um exame do catálogo-base com este código SIGTAP",
  invalid_sector: "setor desconhecido pelo api",
  duplicate_analyte_key: "já existe um componente com esta chave neste exame",
  invalid_analyte_key: "chave do componente inválida — use letras minúsculas, números e _",
  invalid_value_type: "tipo de valor desconhecido pelo api",
  choices_required: "componente do tipo lista precisa de ao menos uma opção",
  invalid_reference_ranges: "faixas de referência inválidas — confira idade mínima ≤ máxima e baixo ≤ alto",
  invalid_loinc: "código LOINC inválido",
  not_found: "registro não encontrado — atualize a tela"
};

export function upsertErrorText(errors: { path?: string | null; message: string }[]): string {
  if (errors.length === 0) return "não foi possível concluir — tente de novo";
  return errors
    .map((e) => (e.message === "blank" && e.path ? `campo obrigatório vazio: ${e.path}` : UPSERT_ERRORS[e.message] ?? e.message))
    .join("; ");
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/lib/labTemplates.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/labTemplates.ts src/lib/labTemplates.test.ts
$G commit -m "feat: add display and input rules for the lab exam base catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tela Laboratório — schema, lista, formulário do exame e componentes

**Files:**
- Modify: `schema.graphql`, `src/gql/gql.ts`, `src/gql/graphql.ts` (gerados), `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx`, `README.md`
- Create: `src/screens/LabTemplates.tsx`
- Test: `src/screens/LabTemplates.test.tsx`

**Interfaces:**
- Consumes: Task 3 (`TemplateView`, `AnalyteView`, `SECTORS`, `optionsWith`, `orNull`, `sectorText`, `valueTypeText`, `rangesSummary`, `upsertErrorText`); `gql`, `graphql`, `GraphQLRefusal` (existentes); do api, o SDL da Divergência M1.
- Produces:
  - `export function LabTemplates(): JSX.Element` e `export const LAB_TEMPLATES_QUERY_KEY = [ "platform", "labExamTemplates" ]` em `src/screens/LabTemplates.tsx`;
  - regiões acessíveis: "Catálogo-base de exames de laboratório (todas as cidades do ambiente)", "Exame do catálogo-base" (formulário), `Componentes de <nome>`;
  - Shell: `Screen` ganha `"lab"`, botão do trilho "Laboratório".

- [ ] **Step 1: Puxe o schema do api do 21b e confira**

O script `schema:pull` lê do container `api`, que monta `apps/api`, não o worktree; rode o mesmo comando com `-w` no worktree do api:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml exec -T -w /app/.claude/mod21b api bin/rails runner 'puts Maintenance::Schema.to_definition' > $R/.worktrees/maintenance-mod21b/schema.graphql
cd $R/.worktrees/maintenance-mod21b && /opt/homebrew/bin/git diff --stat schema.graphql
grep -n -E 'labExamTemplates|upsertLabExamTemplate\(|upsertLabAnalyteTemplate\(|^type LabExamTemplate |^type LabAnalyteTemplate |^type LabReferenceRange |^input LabReferenceRangeInput' schema.graphql
```

Expected: o diff só **acrescenta** os tipos do 21b; as seis linhas do grep presentes. Compare o bloco com o SDL da Divergência M1 (nomes, nulidade e argumentos): se diferir, **pare e reporte** — as Tasks 4 e 5 usam esses nomes. Se o comando falhar por `master.key` ou bundle, siga a receita do plano do api para o worktree no container (copiar `config/master.key`).

- [ ] **Step 2: Write the failing test**

Crie `src/screens/LabTemplates.test.tsx`:

```tsx
// src/screens/LabTemplates.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { LabTemplates } from "./LabTemplates";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string; variables: Record<string, unknown> } {
  return JSON.parse((call[1] as RequestInit).body as string);
}
function calls(fetchMock: ReturnType<typeof vi.fn>, operation: string) {
  return fetchMock.mock.calls.filter((call) => bodyOf(call).query.includes(operation));
}
type Route = (body: { query: string; variables: Record<string, unknown> }) => Promise<Response> | Response;
function route(fetchMock: ReturnType<typeof vi.fn>, handlers: Record<string, Route>) {
  fetchMock.mockImplementation((_url: string, init: RequestInit) => {
    const body = JSON.parse(init.body as string);
    const name = Object.keys(handlers).find((op) => body.query.includes(op));
    return Promise.resolve(name ? handlers[name](body) : reply(500, {}));
  });
}

const HB = {
  id: "a1", key: "hemoglobin", name: "Hemoglobina", loinc: "718-7", unit: "g/dL", valueType: "numeric", choices: [],
  decimals: 1, position: 1,
  referenceRanges: [
    { sex: "male", ageMinDays: 6570, ageMaxDays: 43800, low: 13.5, high: 17.5, text: null },
    { sex: "female", ageMinDays: 6570, ageMaxDays: 43800, low: 12, high: 15.5, text: null }
  ]
};
const HCT = {
  id: "a2", key: "hematocrit", name: "Hematócrito", loinc: "4544-3", unit: "%", valueType: "numeric", choices: [],
  decimals: 1, position: 2, referenceRanges: []
};
const HEMOGRAMA = {
  id: "t1", sigtapCode: "0202020380", name: "Hemograma completo", sector: "hematology", material: "sangue total",
  container: "tubo roxo (EDTA)", suggestedMethod: "automação hematológica", preparation: null, active: true,
  analytes: [ HCT, HB ]
};
const GLICOSE = {
  id: "t2", sigtapCode: "0202010473", name: "Glicose", sector: "biochemistry", material: "soro",
  container: "tubo cinza (fluoreto)", suggestedMethod: null, preparation: "jejum de 8 horas", active: false, analytes: []
};
const CREATININA = {
  id: "t3", sigtapCode: "0202010317", name: "Creatinina", sector: "biochemistry", material: "soro",
  container: "tubo amarelo (gel)", suggestedMethod: null, preparation: null, active: true, analytes: []
};

function templatesReply(list: unknown[] = [ HEMOGRAMA, GLICOSE ]) {
  return reply(200, { data: { labExamTemplates: list } });
}
const SAVED = () => reply(200, { data: { upsertLabExamTemplate: { ok: true, errors: [] } } });

function renderScreen() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<LabTemplates />, { wrapper });
}
const rowOf = (text: string) => screen.getByRole("row", { name: new RegExp(text) });
const form = () => screen.getByRole("region", { name: "Exame do catálogo-base" });
const input = (label: string) => within(form()).getByLabelText(label) as HTMLInputElement;

describe("LabTemplates", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  it("lista os exames com setor, material, componentes e situação, sem undefined/null", async () => {
    route(fetchMock, { "query LabExamTemplates": () => templatesReply() });
    renderScreen();

    expect(await screen.findByText("Hemograma completo")).not.toBeNull();
    const hb = rowOf("0202020380");
    expect(within(hb).getByText("Hematologia")).not.toBeNull();
    expect(within(hb).getByText("sangue total · tubo roxo (EDTA)")).not.toBeNull();
    expect(within(hb).getByText("2")).not.toBeNull();
    expect(within(hb).getByText("ativo")).not.toBeNull();
    const gl = rowOf("0202010473");
    expect(within(gl).getByText("Bioquímica")).not.toBeNull();
    expect(within(gl).getByText("0")).not.toBeNull();
    expect(within(gl).getByText("inativo")).not.toBeNull();
    expect(screen.getByText(/os exames já adotados pelas cidades e os laudos não mudam/)).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("nenhum exame no catálogo-base", async () => {
    route(fetchMock, { "query LabExamTemplates": () => templatesReply([]) });
    renderScreen();
    expect(await screen.findByText("nenhum exame no catálogo-base")).not.toBeNull();
  });

  it("editar um exame: o formulário vem preenchido, salva com as mudanças e relê o catálogo", async () => {
    const user = userEvent.setup();
    route(fetchMock, { "query LabExamTemplates": () => templatesReply(), "mutation UpsertLabExamTemplate": SAVED });
    renderScreen();

    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));
    expect(input("Nome").value).toBe("Hemograma completo");
    expect((within(form()).getByLabelText("Setor") as HTMLSelectElement).value).toBe("hematology");
    expect(input("Preparo do paciente").value).toBe("");
    await user.type(input("Preparo do paciente"), "sem jejum");
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));

    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabExamTemplate")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation UpsertLabExamTemplate")[0]).variables).toEqual({
      id: "t1", sigtapCode: "0202020380", name: "Hemograma completo", sector: "hematology", material: "sangue total",
      container: "tubo roxo (EDTA)", suggestedMethod: "automação hematológica", preparation: "sem jejum", active: true
    });
    expect((await screen.findByRole("status")).textContent).toBe("Exame salvo.");
    await waitFor(() => expect(calls(fetchMock, "query LabExamTemplates").length).toBeGreaterThanOrEqual(2));
  });

  it("desativar: o exame sai de uso sem ser apagado", async () => {
    const user = userEvent.setup();
    route(fetchMock, { "query LabExamTemplates": () => templatesReply(), "mutation UpsertLabExamTemplate": SAVED });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));
    await user.click(within(form()).getByLabelText("ativo (aparece para adoção)"));
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));
    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabExamTemplate")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation UpsertLabExamTemplate")[0]).variables.active).toBe(false);
  });

  it("novo exame: salva sem id, com o texto aparado, e depois de reler abre os componentes do exame criado", async () => {
    const user = userEvent.setup();
    let listed: unknown[] = [ HEMOGRAMA ];
    route(fetchMock, {
      "query LabExamTemplates": () => templatesReply(listed),
      "mutation UpsertLabExamTemplate": () => { listed = [ HEMOGRAMA, CREATININA ]; return SAVED(); }
    });
    renderScreen();

    await screen.findByText("Hemograma completo");
    await user.click(screen.getByRole("button", { name: "novo exame" }));
    await user.type(input("Código SIGTAP"), " 0202010317 ");
    await user.type(input("Nome"), "Creatinina");
    await user.selectOptions(within(form()).getByLabelText("Setor"), "biochemistry");
    await user.type(input("Material"), "soro");
    await user.type(input("Recipiente"), "tubo amarelo (gel)");
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));

    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabExamTemplate")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation UpsertLabExamTemplate")[0]).variables).toEqual({
      id: null, sigtapCode: "0202010317", name: "Creatinina", sector: "biochemistry", material: "soro",
      container: "tubo amarelo (gel)", suggestedMethod: null, preparation: null, active: true
    });
    const components = await screen.findByRole("region", { name: "Componentes de Creatinina" });
    expect(within(components).getByText("nenhum componente")).not.toBeNull();
    expect(screen.getByRole("status").textContent).toBe("Exame salvo.");
    expect(input("Nome").value).toBe("Creatinina");
  });

  it("recusa do api: mostra a frase e mantém o que foi digitado", async () => {
    const user = userEvent.setup();
    route(fetchMock, {
      "query LabExamTemplates": () => templatesReply(),
      "mutation UpsertLabExamTemplate": () => reply(200, { data: { upsertLabExamTemplate: {
        ok: false, errors: [ { path: "sigtapCode", message: "duplicate_sigtap" } ]
      } } })
    });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(screen.getByRole("button", { name: "novo exame" }));
    await user.type(input("Código SIGTAP"), "0202010473");
    await user.type(input("Nome"), "Glicemia de jejum");
    await user.type(input("Material"), "plasma");
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));

    expect((await screen.findByRole("alert")).textContent).toBe("já existe um exame do catálogo-base com este código SIGTAP");
    expect(input("Código SIGTAP").value).toBe("0202010473");
    expect(input("Nome").value).toBe("Glicemia de jejum");
    expect(input("Material").value).toBe("plasma");
    expect(screen.queryByRole("status")).toBeNull();
  });

  it("setor desconhecido (api mais novo): aparece cru e salvar não o troca", async () => {
    const user = userEvent.setup();
    const CITO = { ...GLICOSE, id: "t9", sigtapCode: "0203010019", name: "Citopatológico", sector: "cytology" };
    route(fetchMock, { "query LabExamTemplates": () => templatesReply([ CITO ]), "mutation UpsertLabExamTemplate": SAVED });
    renderScreen();
    await screen.findByText("Citopatológico");
    expect(within(rowOf("0203010019")).getByText("cytology")).not.toBeNull();
    await user.click(within(rowOf("0203010019")).getByRole("button", { name: "editar" }));
    expect((within(form()).getByLabelText("Setor") as HTMLSelectElement).value).toBe("cytology");
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));
    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabExamTemplate")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation UpsertLabExamTemplate")[0]).variables.sector).toBe("cytology");
  });

  it("componentes do exame em ordem, com as faixas de referência em português", async () => {
    const user = userEvent.setup();
    route(fetchMock, { "query LabExamTemplates": () => templatesReply() });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));

    const components = screen.getByRole("region", { name: "Componentes de Hemograma completo" });
    const rows = within(components).getAllByRole("row");
    expect(within(rows[1]).getByText("hemoglobin")).not.toBeNull();
    expect(within(rows[1]).getByText("718-7")).not.toBeNull();
    expect(within(rows[1]).getByText("g/dL")).not.toBeNull();
    expect(within(rows[1]).getByText("numérico")).not.toBeNull();
    expect(within(rows[1]).getByText("masculino, 18 a–120 a: 13,5–17,5; feminino, 18 a–120 a: 12–15,5")).not.toBeNull();
    expect(within(rows[2]).getByText("hematocrit")).not.toBeNull();
    expect(within(rows[2]).getByText("sem faixa")).not.toBeNull();
  });

  it("recusa do envelope (payload nulo): mostra o código e a mensagem do api", async () => {
    const user = userEvent.setup();
    route(fetchMock, {
      "query LabExamTemplates": () => templatesReply(),
      "mutation UpsertLabExamTemplate": () => reply(200, {
        data: { upsertLabExamTemplate: null },
        errors: [ { message: "sessão sem permissão de escrita", path: [ "upsertLabExamTemplate" ], extensions: { code: "FORBIDDEN" } } ]
      })
    });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));
    expect((await screen.findByRole("alert")).textContent).toBe("FORBIDDEN — sessão sem permissão de escrita");
  });

  it("api sem o 21b: explica a ordem de deploy", async () => {
    route(fetchMock, { "query LabExamTemplates": () => reply(200, {
      errors: [ {
        message: "Field 'labExamTemplates' doesn't exist on type 'Query'",
        extensions: { code: "undefinedField", typeName: "Query", fieldName: "labExamTemplates" }
      } ]
    }) });
    renderScreen();
    expect((await screen.findByRole("alert")).textContent).toMatch(/o api do módulo 21b precisa subir antes do maintenance/);
  });

  it("falha de rede ao salvar: mostra o alerta, mantém o formulário e relê o catálogo", async () => {
    const user = userEvent.setup();
    route(fetchMock, { "query LabExamTemplates": () => templatesReply(), "mutation UpsertLabExamTemplate": () => reply(504, {}) });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));
    await user.click(within(form()).getByRole("button", { name: "salvar exame" }));
    expect(await screen.findByRole("alert")).not.toBeNull();
    expect(input("Nome").value).toBe("Hemograma completo");
    await waitFor(() => expect(calls(fetchMock, "query LabExamTemplates").length).toBeGreaterThanOrEqual(2));
  });
});
```

Em `src/screens/Shell.test.tsx`, no primeiro teste, troque o nome `"mostra o e-mail do mantenedor, a navegação com as cinco telas e 'sair' encerra a sessão"` por `"mostra o e-mail do mantenedor, a navegação com as seis telas e 'sair' encerra a sessão"` e a lista `[ "Cidades", "Medicamentos", "Mantenedores", "Tokens", "Auditoria" ]` por `[ "Cidades", "Medicamentos", "Laboratório", "Mantenedores", "Tokens", "Auditoria" ]`. No fim do `describe("Shell", …)`, acrescente:

```tsx
  it("Laboratório abre o catálogo-base de exames", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const path = String(url);
      if (path === "/session") return Promise.resolve(reply(200, { id: "m1", email_address: "mantenedor@rotasaude.app", expires_at: "2026-01-01T01:00:00Z" }));
      if (path === "/graphql") {
        const body = JSON.parse((init as RequestInit).body as string);
        if (String(body.query).includes("query LabExamTemplates")) return Promise.resolve(reply(200, { data: { labExamTemplates: [] } }));
        return Promise.resolve(reply(200, { data: { cities: [] } }));
      }
      return Promise.resolve(reply(204, undefined));
    });
    renderShell();

    await user.click(await screen.findByRole("button", { name: "Laboratório" }));
    expect(screen.getByRole("button", { name: "Laboratório" }).getAttribute("aria-current")).toBe("page");
    expect(await screen.findByText("nenhum exame no catálogo-base")).not.toBeNull();
  });
```

Em `src/screens/Shell.rail.test.tsx`, troque o nome `"trilho 'Telas' com os 5 botões de nome exato, ícone escondido, Cidades ativo"` por `"trilho 'Telas' com os 6 botões de nome exato, ícone escondido, Cidades ativo"` e a lista `[ "Cidades", "Medicamentos", "Mantenedores", "Tokens", "Auditoria" ]` por `[ "Cidades", "Medicamentos", "Laboratório", "Mantenedores", "Tokens", "Auditoria" ]` (o ícone do botão novo também precisa ser `aria-hidden`).

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $M && npx vitest run src/screens/LabTemplates.test.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx`
Expected: FAIL — `./LabTemplates` não existe; o Shell e o trilho não têm "Laboratório".

- [ ] **Step 4: Write minimal implementation**

Crie `src/screens/LabTemplates.tsx`:

```tsx
// src/screens/LabTemplates.tsx
// Catálogo-base de exames de laboratório da plataforma (módulo 21b, F-21.14;
// spec §4 "Plataforma" e §7; contrato §10). Vale para TODAS as cidades do
// ambiente: é o ponto de partida que o laboratório municipal adota e ajusta
// ao seu método. Adotar copia, e cada ajuste da cidade vira versão imutável:
// mudar aqui não mexe em exame adotado nem em laudo. Sem step-up (Divergência
// M4): nada aqui emite credencial nem muda o que uma cidade usa hoje.
//
// Consulta PRÓPRIA, como Medicamentos: contra um api sem o 21b a validação
// recusa o documento inteiro, e só esta tela cai.
import { useState, type CSSProperties } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import {
  SECTORS, optionsWith, orNull, rangesSummary, sectorText, upsertErrorText, valueTypeText,
  type AnalyteView, type TemplateView
} from "../lib/labTemplates";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Field } from "../components/Field";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";

const LabExamTemplatesQuery = graphql(`
  query LabExamTemplates {
    labExamTemplates {
      id sigtapCode name sector material container suggestedMethod preparation active
      analytes {
        id key name loinc unit valueType choices decimals position
        referenceRanges { sex ageMinDays ageMaxDays low high text }
      }
    }
  }
`);

const UpsertLabExamTemplateMutation = graphql(`
  mutation UpsertLabExamTemplate(
    $id: ID, $sigtapCode: String!, $name: String!, $sector: String!, $material: String!, $container: String!,
    $suggestedMethod: String, $preparation: String, $active: Boolean!
  ) {
    upsertLabExamTemplate(
      id: $id, sigtapCode: $sigtapCode, name: $name, sector: $sector, material: $material, container: $container,
      suggestedMethod: $suggestedMethod, preparation: $preparation, active: $active
    ) { ok errors { path message } }
  }
`);

const TITLE = "Catálogo-base de exames de laboratório (todas as cidades do ambiente)";
const INTRO =
  "Ponto de partida que o laboratório municipal adota e ajusta ao seu método. Mudar aqui vale para as próximas " +
  "adoções: os exames já adotados pelas cidades e os laudos não mudam. Para tirar um exame de uso, desmarque \"ativo\".";
export const LAB_TEMPLATES_QUERY_KEY = [ "platform", "labExamTemplates" ];
const OLD_API_CODES = new Set([ "undefinedField" ]);

function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem o catálogo de laboratório — o api do módulo 21b precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

function refused(fieldErrors: GraphQLRefusal[]): string {
  const refusal = fieldErrors[0];
  return refusal ? `${refusal.code} — ${refusal.message}` : upsertErrorText([]);
}

type Selection = { kind: "new" } | { kind: "edit"; id: string } | null;
type Outcome = { tone: "done" | "failed"; text: string };
type TemplateDraft = {
  sigtapCode: string; name: string; sector: string; material: string; container: string;
  suggestedMethod: string; preparation: string; active: boolean;
};

function toDraft(t: TemplateView | null): TemplateDraft {
  return t
    ? {
      sigtapCode: t.sigtapCode, name: t.name, sector: t.sector, material: t.material, container: t.container,
      suggestedMethod: t.suggestedMethod ?? "", preparation: t.preparation ?? "", active: t.active
    }
    : { sigtapCode: "", name: "", sector: SECTORS[0], material: "", container: "", suggestedMethod: "", preparation: "", active: true };
}

export function LabTemplates() {
  const queryClient = useQueryClient();
  const query = useQuery({
    queryKey: LAB_TEMPLATES_QUERY_KEY,
    queryFn: () => gql(LabExamTemplatesQuery),
    staleTime: Infinity
  });
  const [ selection, setSelection ] = useState<Selection>(null);
  const [ outcome, setOutcome ] = useState<Outcome | null>(null);

  const templates: TemplateView[] = query.data?.data?.labExamTemplates ?? [];
  const fieldError = query.data?.fieldErrors[0];
  const current = selection?.kind === "edit" ? templates.find((t) => t.id === selection.id) ?? null : null;

  function open(next: Selection) {
    setSelection(next);
    setOutcome(null);
  }

  const save = useMutation({
    mutationFn: (vars: { id: string | null } & TemplateDraft) => gql(UpsertLabExamTemplateMutation, {
      id: vars.id, sigtapCode: vars.sigtapCode.trim(), name: vars.name.trim(), sector: vars.sector,
      material: vars.material.trim(), container: vars.container.trim(),
      suggestedMethod: orNull(vars.suggestedMethod), preparation: orNull(vars.preparation), active: vars.active
    }),
    onMutate: () => setOutcome(null),
    onSuccess: async ({ data, fieldErrors }, vars) => {
      const payload = data?.upsertLabExamTemplate ?? null;
      if (!payload) { setOutcome({ tone: "failed", text: refused(fieldErrors) }); return; }
      if (!payload.ok) { setOutcome({ tone: "failed", text: upsertErrorText(payload.errors) }); return; }
      await queryClient.invalidateQueries({ queryKey: LAB_TEMPLATES_QUERY_KEY });
      // O contrato devolve só { ok, errors }: o exame criado é achado pelo SIGTAP, que é único.
      if (vars.id === null) {
        const fresh = queryClient.getQueryData<typeof query.data>(LAB_TEMPLATES_QUERY_KEY)?.data?.labExamTemplates ?? [];
        const created = fresh.find((t) => t.sigtapCode === vars.sigtapCode.trim());
        if (created) setSelection({ kind: "edit", id: created.id });
      }
      setOutcome({ tone: "done", text: "Exame salvo." });
    },
    // Rede/timeout: o api pode ter gravado; relê o catálogo e mantém o formulário.
    onError: (err) => {
      setOutcome({ tone: "failed", text: requestErrorText(err) });
      void queryClient.invalidateQueries({ queryKey: LAB_TEMPLATES_QUERY_KEY });
    }
  });

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <section aria-label={TITLE}>
        <Panel
          title={TITLE}
          actions={
            <div style={row}>
              <Button variant="secondary" onClick={() => { setOutcome(null); void query.refetch(); }} busy={query.isFetching}>atualizar</Button>
              <Button onClick={() => open({ kind: "new" })}>novo exame</Button>
            </div>
          }
        >
          <p style={muted}>{INTRO}</p>
          {query.isPending && <p>carregando…</p>}
          {query.isError && <ErrorState message={requestErrorText(query.error)} />}
          {fieldError && <ErrorState message={`${fieldError.code} — ${fieldError.message}`} />}
          {query.isSuccess && !fieldError && (templates.length === 0 ? <EmptyState message="nenhum exame no catálogo-base" /> : (
            <DataTable<TemplateView>
              columns={[
                { key: "sigtapCode", label: "SIGTAP", render: (t) => t.sigtapCode },
                { key: "name", label: "Exame", render: (t) => t.name },
                { key: "sector", label: "Setor", render: (t) => sectorText(t.sector) },
                { key: "material", label: "Material e recipiente", render: (t) => `${t.material} · ${t.container}` },
                { key: "analytes", label: "Componentes", render: (t) => String(t.analytes.length) },
                { key: "active", label: "Situação", render: (t) => <Tag tone={t.active ? "ok" : "neutral"}>{t.active ? "ativo" : "inativo"}</Tag> },
                { key: "edit", label: "", render: (t) => <Button variant="secondary" onClick={() => open({ kind: "edit", id: t.id })}>editar</Button> }
              ]}
              rows={templates}
              rowKey={(t) => t.id}
            />
          ))}
        </Panel>
      </section>

      {(selection?.kind === "new" || current) && (
        <section aria-label="Exame do catálogo-base">
          <TemplateForm
            key={current ? current.id : "new"}
            template={current}
            busy={save.isPending}
            onSave={(draft) => save.mutate({ id: current?.id ?? null, ...draft })}
            onClose={() => open(null)}
          />
        </section>
      )}

      {outcome?.tone === "done" && <p role="status" style={{ margin: 0, fontSize: 12.5, color: "var(--ok)" }}>{outcome.text}</p>}
      {outcome?.tone === "failed" && <ErrorState message={outcome.text} />}

      {current && (
        <section aria-label={`Componentes de ${current.name}`}>
          <Panel title={`Componentes de ${current.name}`}>
            {current.analytes.length === 0 ? <EmptyState message="nenhum componente" /> : (
              <DataTable<AnalyteView>
                columns={[
                  { key: "position", label: "Ordem", render: (a) => String(a.position) },
                  { key: "key", label: "Chave", render: (a) => a.key },
                  { key: "name", label: "Nome", render: (a) => a.name },
                  { key: "loinc", label: "LOINC", render: (a) => a.loinc ?? "—" },
                  { key: "unit", label: "Unidade", render: (a) => a.unit ?? "—" },
                  { key: "valueType", label: "Tipo", render: (a) => valueTypeText(a.valueType) },
                  { key: "ranges", label: "Faixas de referência", render: (a) => rangesSummary(a.referenceRanges) }
                ]}
                rows={[ ...current.analytes ].sort((a, b) => a.position - b.position)}
                rowKey={(a) => a.id}
              />
            )}
          </Panel>
        </section>
      )}
    </div>
  );
}

type FormProps = { template: TemplateView | null; busy: boolean; onSave(draft: TemplateDraft): void; onClose(): void };

function TemplateForm({ template, busy, onSave, onClose }: FormProps) {
  const [ draft, setDraft ] = useState<TemplateDraft>(() => toDraft(template));
  const set = <K extends keyof TemplateDraft>(field: K) => (value: TemplateDraft[K]) => setDraft((d) => ({ ...d, [field]: value }));

  return (
    <Panel title={template ? `Editar ${template.name} (${template.sigtapCode})` : "Novo exame do catálogo-base"}>
      <div style={grid}>
        <Field label="Código SIGTAP" value={draft.sigtapCode} onChange={set("sigtapCode")} helpText="10 dígitos, da competência vigente" />
        <Field label="Nome" value={draft.name} onChange={set("name")} />
        <label style={labelStyle}>
          <span>Setor</span>
          <select value={draft.sector} onChange={(event) => set("sector")(event.target.value)} style={inputStyle}>
            {optionsWith(SECTORS, draft.sector).map((s) => <option key={s} value={s}>{sectorText(s)}</option>)}
          </select>
        </label>
        <Field label="Material" value={draft.material} onChange={set("material")} />
        <Field label="Recipiente" value={draft.container} onChange={set("container")} />
        <Field label="Método sugerido" value={draft.suggestedMethod} onChange={set("suggestedMethod")} />
        <Field label="Preparo do paciente" multiline value={draft.preparation} onChange={set("preparation")} />
        <label style={checkStyle}>
          <input type="checkbox" checked={draft.active} onChange={(event) => set("active")(event.target.checked)} />
          ativo (aparece para adoção)
        </label>
      </div>
      <div style={row}>
        <Button onClick={() => onSave(draft)} busy={busy}>salvar exame</Button>
        <Button variant="secondary" onClick={onClose}>fechar</Button>
      </div>
    </Panel>
  );
}

const muted: CSSProperties = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap" };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(200px, 1fr))", gap: 12 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const inputStyle: CSSProperties = {
  padding: "8px 10px", border: "1px solid var(--rule2)", borderRadius: 6, background: "var(--panel)", color: "var(--ink)", fontSize: 13
};
const checkStyle: CSSProperties = { display: "flex", alignItems: "center", gap: 8, fontSize: 12.5, color: "var(--ink)" };
```

Gere os tipos (o `graphql()` só conhece documentos que o codegen viu):

```bash
cd $M && npm run codegen && /opt/homebrew/bin/git status --short src/gql schema.graphql
```

Expected: `schema.graphql`, `src/gql/gql.ts` e `src/gql/graphql.ts` modificados.

Em `src/screens/Shell.tsx`:

1. Depois de `import { MedicationCatalog } from "./MedicationCatalog";`, acrescente `import { LabTemplates } from "./LabTemplates";`.
2. Troque `export type Screen = "cities" | "medications" | "maintainers" | "tokens" | "audit";` por `export type Screen = "cities" | "medications" | "lab" | "maintainers" | "tokens" | "audit";` e, no comentário acima, acrescente a frase `Módulo 21b: Laboratório, o catálogo-base de exames (também da plataforma).`
3. Depois de `  { key: "medications", label: "Medicamentos" },`, acrescente `  { key: "lab", label: "Laboratório" },`.
4. Em `ICON`, depois da linha `medications: …`, acrescente `  lab: svg("M9 3h6M10 3v7l-5 9a2 2 0 0 0 1.7 3h10.6a2 2 0 0 0 1.7-3l-5-9V3"),`.
5. Depois de `          {screen === "medications" && <MedicationCatalog />}`, acrescente `          {screen === "lab" && <LabTemplates />}`.

Em `README.md`, no fim do parágrafo **Laboratório municipal (módulo 21b).** (Task 2), acrescente:

```markdown
A tela **Laboratório** do trilho mantém o catálogo-base de exames de
laboratório da plataforma (vale para todas as cidades do ambiente): exames
(SIGTAP, setor, material, recipiente, preparo), componentes (chave, LOINC,
unidade, tipo) e faixas de referência por sexo e idade em dias. Decimal com
vírgula e sem ponto de milhar. É o ponto de partida que cada laboratório adota
e ajusta: mudar aqui não altera exame já adotado nem laudo. O SDL vem do api
(`npm run schema:pull`, depois `npm run codegen`).
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $M && npx vitest run src/screens/LabTemplates.test.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx src/theme/guards.test.ts && npx tsc --noEmit && npm run codegen:check`
Expected: PASS; `codegen:check` sem diferença.

- [ ] **Step 6: Commit**

```bash
cd $M
$G add schema.graphql src/gql/gql.ts src/gql/graphql.ts src/screens/LabTemplates.tsx src/screens/LabTemplates.test.tsx src/screens/Shell.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx README.md
$G commit -m "feat: add the lab exam base catalog screen to maintenance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Componentes e faixas de referência

**Files:**
- Create: `src/screens/LabAnalyteForm.tsx`
- Modify: `src/screens/LabTemplates.tsx`, `src/screens/LabTemplates.test.tsx`, `src/theme/guards.test.ts`, `src/gql/gql.ts`, `src/gql/graphql.ts`
- Test: `src/screens/LabAnalyteForm.test.tsx`

**Interfaces:**
- Consumes: Task 3 (`AnalyteView`, `RangeDraft`, `EMPTY_RANGE`, `SEXES`, `VALUE_TYPES`, `draftsToRanges`, `rangeToDraft`, `optionsWith`, `orNull`, `sexText`, `valueTypeText`, `upsertErrorText`); Task 4 (`LabTemplates`, `LAB_TEMPLATES_QUERY_KEY`, a região `Componentes de <nome>`); `TableScroll` (existente); `UpsertLabAnalyteTemplateMutationVariables` (gerado).
- Produces: `export function LabAnalyteForm(props: { templateId: string; analyte: AnalyteView | null; nextPosition: number; onSaved(text: string): void; onClose(): void }): JSX.Element` — região "Componente", tabela "Faixas de referência", botões "acrescentar faixa", "remover", "salvar componente", "cancelar".

- [ ] **Step 1: Write the failing test**

Crie `src/screens/LabAnalyteForm.test.tsx`:

```tsx
// src/screens/LabAnalyteForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { LabAnalyteForm } from "./LabAnalyteForm";
import type { AnalyteView } from "../lib/labTemplates";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string; variables: Record<string, unknown> } {
  return JSON.parse((call[1] as RequestInit).body as string);
}
function calls(fetchMock: ReturnType<typeof vi.fn>, operation: string) {
  return fetchMock.mock.calls.filter((call) => bodyOf(call).query.includes(operation));
}

const HB: AnalyteView = {
  id: "a1", key: "hemoglobin", name: "Hemoglobina", loinc: "718-7", unit: "g/dL", valueType: "numeric", choices: [],
  decimals: 1, position: 1,
  referenceRanges: [
    { sex: "male", ageMinDays: 6570, ageMaxDays: 43800, low: 13.5, high: 17.5, text: null },
    { sex: "female", ageMinDays: 6570, ageMaxDays: 43800, low: 12, high: 15.5, text: null }
  ]
};
const MALE_IN = { sex: "male", ageMinDays: 6570, ageMaxDays: 43800, low: 13.5, high: 17.5, text: null };
const FEMALE_IN = { sex: "female", ageMinDays: 6570, ageMaxDays: 43800, low: 12, high: 15.5, text: null };
const SAVED = () => reply(200, { data: { upsertLabAnalyteTemplate: { ok: true, errors: [] } } });

function renderForm(analyte: AnalyteView | null = HB) {
  const onSaved = vi.fn();
  const onClose = vi.fn();
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  render(<LabAnalyteForm templateId="t1" analyte={analyte} nextPosition={3} onSaved={onSaved} onClose={onClose} />, { wrapper });
  return { onSaved, onClose };
}
const field = (label: string) => screen.getByLabelText(label) as HTMLInputElement;
const save = () => screen.getByRole("button", { name: "salvar componente" });
const rangeRows = () => within(screen.getByRole("table", { name: "Faixas de referência" })).getAllByRole("row");
const sent = (fetchMock: ReturnType<typeof vi.fn>) => bodyOf(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")[0]).variables;

describe("LabAnalyteForm", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  it("editar: faixas vêm com vírgula e voltam como número", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(SAVED()));
    const { onSaved } = renderForm();

    expect(field("Nome do componente").value).toBe("Hemoglobina");
    expect(field("faixa 1: baixo").value).toBe("13,5");
    expect((screen.getByLabelText("faixa 2: sexo") as HTMLSelectElement).value).toBe("female");
    await user.clear(field("faixa 1: alto"));
    await user.type(field("faixa 1: alto"), "17,8");
    await user.click(save());

    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(1));
    expect(sent(fetchMock)).toEqual({
      id: "a1", labExamTemplateId: "t1", key: "hemoglobin", name: "Hemoglobina", loinc: "718-7", unit: "g/dL",
      valueType: "numeric", choices: [], decimals: 1, position: 1,
      referenceRanges: [ { ...MALE_IN, high: 17.8 }, FEMALE_IN ]
    });
    await waitFor(() => expect(onSaved).toHaveBeenCalledWith("Componente salvo."));
  });

  it("novo componente: ponto de milhar é recusado antes de enviar", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(SAVED()));
    renderForm(null);

    expect(field("Ordem").value).toBe("3");
    await user.type(field("Chave"), "platelets");
    await user.type(field("Nome do componente"), "Plaquetas");
    await user.type(field("Unidade"), "/mm³");
    await user.type(field("faixa 1: baixo"), "150.000");
    await user.type(field("faixa 1: alto"), "450000");
    await user.click(save());

    expect((await screen.findByRole("alert")).textContent)
      .toBe("faixa 1: \"baixo\" não é um número — use vírgula para decimais, sem ponto de milhar");
    expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(0);

    await user.clear(field("faixa 1: baixo"));
    await user.type(field("faixa 1: baixo"), "150000");
    await user.click(save());
    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(1));
    expect(sent(fetchMock)).toEqual({
      id: null, labExamTemplateId: "t1", key: "platelets", name: "Plaquetas", loinc: null, unit: "/mm³",
      valueType: "numeric", choices: [], decimals: 0, position: 3,
      referenceRanges: [ { sex: "any", ageMinDays: 0, ageMaxDays: 43800, low: 150000, high: 450000, text: null } ]
    });
  });

  it("acrescentar e remover faixas", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(SAVED()));
    renderForm();

    expect(rangeRows()).toHaveLength(3);
    await user.click(screen.getByRole("button", { name: "acrescentar faixa" }));
    expect(rangeRows()).toHaveLength(4);
    expect((screen.getByLabelText("faixa 3: sexo") as HTMLSelectElement).value).toBe("any");
    expect(field("faixa 3: idade máxima em dias").value).toBe("43800");
    await user.type(field("faixa 3: texto"), "ver laudo");
    await user.click(within(rangeRows()[1]).getByRole("button", { name: "remover" }));
    expect(rangeRows()).toHaveLength(3);
    await user.click(save());

    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(1));
    expect(sent(fetchMock).referenceRanges).toEqual([
      FEMALE_IN,
      { sex: "any", ageMinDays: 0, ageMaxDays: 43800, low: null, high: null, text: "ver laudo" }
    ]);
  });

  it("recusa do api (faixas inválidas): mostra a frase e mantém as faixas digitadas", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, { data: { upsertLabAnalyteTemplate: {
      ok: false, errors: [ { path: "referenceRanges", message: "invalid_reference_ranges" } ]
    } } })));
    const { onSaved } = renderForm();

    await user.clear(field("faixa 1: baixo"));
    await user.type(field("faixa 1: baixo"), "18");
    await user.click(save());

    expect((await screen.findByRole("alert")).textContent)
      .toBe("faixas de referência inválidas — confira idade mínima ≤ máxima e baixo ≤ alto");
    expect(field("faixa 1: baixo").value).toBe("18");
    expect(field("faixa 2: alto").value).toBe("15,5");
    expect(onSaved).not.toHaveBeenCalled();
  });

  it("tipo lista: opções uma por linha, aparadas e sem linha vazia", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(SAVED()));
    renderForm(null);

    expect(screen.queryByLabelText("Opções (uma por linha)")).toBeNull();
    await user.type(field("Chave"), "color");
    await user.type(field("Nome do componente"), "Cor");
    await user.selectOptions(screen.getByLabelText("Tipo de valor"), "choice");
    await user.type(field("Opções (uma por linha)"), " amarelo-claro {enter}amarelo{enter}{enter}âmbar");
    await user.type(field("faixa 1: texto"), "amarelo-claro");
    await user.click(save());

    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(1));
    expect(sent(fetchMock).valueType).toBe("choice");
    expect(sent(fetchMock).choices).toEqual([ "amarelo-claro", "amarelo", "âmbar" ]);
  });

  it("tipo e sexo desconhecidos (api mais novo) aparecem crus e não são trocados", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(SAVED()));
    renderForm({ ...HB, valueType: "ratio", referenceRanges: [ { ...MALE_IN, sex: "intersex" } ] });

    expect((screen.getByLabelText("Tipo de valor") as HTMLSelectElement).value).toBe("ratio");
    expect((screen.getByLabelText("faixa 1: sexo") as HTMLSelectElement).value).toBe("intersex");
    await user.click(save());
    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(1));
    expect(sent(fetchMock).valueType).toBe("ratio");
    expect(sent(fetchMock).referenceRanges).toEqual([ { ...MALE_IN, sex: "intersex" } ]);
  });

  it("casas decimais não inteiras são recusadas antes de enviar; cancelar não envia", async () => {
    const user = userEvent.setup();
    const { onClose } = renderForm();
    await user.clear(field("Casas decimais"));
    await user.type(field("Casas decimais"), "1,5");
    await user.click(save());
    expect((await screen.findByRole("alert")).textContent).toBe("casas decimais: use um número inteiro");
    await user.click(screen.getByRole("button", { name: "cancelar" }));
    expect(onClose).toHaveBeenCalled();
    expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(0);
  });
});
```

Em `src/screens/LabTemplates.test.tsx`, dentro do `describe("LabTemplates", …)`, acrescente:

```tsx
  it("editar um componente pela tela: salva, fecha o formulário e relê o catálogo", async () => {
    const user = userEvent.setup();
    route(fetchMock, {
      "query LabExamTemplates": () => templatesReply(),
      "mutation UpsertLabAnalyteTemplate": () => reply(200, { data: { upsertLabAnalyteTemplate: { ok: true, errors: [] } } })
    });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));
    const components = screen.getByRole("region", { name: "Componentes de Hemograma completo" });
    await user.click(within(within(components).getAllByRole("row")[1]).getByRole("button", { name: "editar" }));

    const analyte = screen.getByRole("region", { name: "Componente" });
    expect((within(analyte).getByLabelText("Nome do componente") as HTMLInputElement).value).toBe("Hemoglobina");
    await user.click(within(analyte).getByRole("button", { name: "salvar componente" }));

    await waitFor(() => expect(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation UpsertLabAnalyteTemplate")[0]).variables.labExamTemplateId).toBe("t1");
    expect((await screen.findByRole("status")).textContent).toBe("Componente salvo.");
    expect(screen.queryByRole("region", { name: "Componente" })).toBeNull();
    await waitFor(() => expect(calls(fetchMock, "query LabExamTemplates").length).toBeGreaterThanOrEqual(2));
  });

  it("novo componente pela tela: a ordem sugerida é a próxima", async () => {
    const user = userEvent.setup();
    route(fetchMock, { "query LabExamTemplates": () => templatesReply() });
    renderScreen();
    await screen.findByText("Hemograma completo");
    await user.click(within(rowOf("0202020380")).getByRole("button", { name: "editar" }));
    await user.click(screen.getByRole("button", { name: "novo componente" }));
    const analyte = screen.getByRole("region", { name: "Componente" });
    expect((within(analyte).getByLabelText("Ordem") as HTMLInputElement).value).toBe("3");
  });
```

Em `src/theme/guards.test.ts`, no `describe("tabelas do maintenance", …)`, acrescente:

```ts
  it("a tabela de faixas de referência fica dentro de TableScroll", () => {
    const code = readFileSync(join(src, "screens/LabAnalyteForm.tsx"), "utf8");
    expect(code).toMatch(/<TableScroll>\s*<table/);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $M && npx vitest run src/screens/LabAnalyteForm.test.tsx src/screens/LabTemplates.test.tsx src/theme/guards.test.ts`
Expected: FAIL — `./LabAnalyteForm` não existe; a tela não tem "novo componente" nem "editar" nos componentes; a guarda não acha o arquivo.

- [ ] **Step 3: Write minimal implementation**

Crie `src/screens/LabAnalyteForm.tsx`:

```tsx
// src/screens/LabAnalyteForm.tsx
// Componente (analito) de um exame do catálogo-base, com as faixas de
// referência por sexo e idade (módulo 21b, F-21.14; spec §4; contrato §10).
// A conversão do que foi digitado (vírgula decimal, ponto de milhar recusado)
// é de labTemplates.ts; regras de conjunto das faixas são do api. Uma recusa
// nunca apaga o que foi digitado.
import { useState, type CSSProperties } from "react";
import { useMutation } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import type { UpsertLabAnalyteTemplateMutationVariables } from "../gql/graphql";
import {
  EMPTY_RANGE, SEXES, VALUE_TYPES, draftsToRanges, optionsWith, orNull, rangeToDraft, sexText, upsertErrorText, valueTypeText,
  type AnalyteView, type RangeDraft
} from "../lib/labTemplates";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Field } from "../components/Field";
import { ErrorState } from "../components/ErrorState";
import { TableScroll } from "../components/TableScroll";

const UpsertLabAnalyteTemplateMutation = graphql(`
  mutation UpsertLabAnalyteTemplate(
    $id: ID, $labExamTemplateId: ID!, $key: String!, $name: String!, $loinc: String, $unit: String,
    $valueType: String!, $choices: [String!], $decimals: Int!, $position: Int!, $referenceRanges: [LabReferenceRangeInput!]!
  ) {
    upsertLabAnalyteTemplate(
      id: $id, labExamTemplateId: $labExamTemplateId, key: $key, name: $name, loinc: $loinc, unit: $unit,
      valueType: $valueType, choices: $choices, decimals: $decimals, position: $position, referenceRanges: $referenceRanges
    ) { ok errors { path message } }
  }
`);

type Props = { templateId: string; analyte: AnalyteView | null; nextPosition: number; onSaved(text: string): void; onClose(): void };
type RangeField = keyof RangeDraft;

const INTEGER = /^\d+$/;
const RANGE_COLUMNS: { field: RangeField; header: string; label: string }[] = [
  { field: "ageMinDays", header: "Idade mín. (dias)", label: "idade mínima em dias" },
  { field: "ageMaxDays", header: "Idade máx. (dias)", label: "idade máxima em dias" },
  { field: "low", header: "Baixo", label: "baixo" },
  { field: "high", header: "Alto", label: "alto" },
  { field: "text", header: "Texto", label: "texto" }
];

export function LabAnalyteForm({ templateId, analyte, nextPosition, onSaved, onClose }: Props) {
  const [ key, setKey ] = useState(analyte?.key ?? "");
  const [ name, setName ] = useState(analyte?.name ?? "");
  const [ loinc, setLoinc ] = useState(analyte?.loinc ?? "");
  const [ unit, setUnit ] = useState(analyte?.unit ?? "");
  const [ valueType, setValueType ] = useState(analyte?.valueType ?? "numeric");
  const [ choices, setChoices ] = useState((analyte?.choices ?? []).join("\n"));
  const [ decimals, setDecimals ] = useState(String(analyte?.decimals ?? 0));
  const [ position, setPosition ] = useState(String(analyte?.position ?? nextPosition));
  const [ ranges, setRanges ] = useState<RangeDraft[]>(analyte ? analyte.referenceRanges.map(rangeToDraft) : [ { ...EMPTY_RANGE } ]);
  const [ error, setError ] = useState<string | null>(null);

  const save = useMutation({
    mutationFn: (vars: UpsertLabAnalyteTemplateMutationVariables) => gql(UpsertLabAnalyteTemplateMutation, vars),
    onMutate: () => setError(null),
    onSuccess: ({ data, fieldErrors }) => {
      const payload = data?.upsertLabAnalyteTemplate ?? null;
      if (!payload) {
        const refusal = fieldErrors[0];
        setError(refusal ? `${refusal.code} — ${refusal.message}` : upsertErrorText([]));
        return;
      }
      if (!payload.ok) { setError(upsertErrorText(payload.errors)); return; }
      onSaved("Componente salvo.");
    },
    onError: (err) => setError(err instanceof Error ? err.message : "erro inesperado")
  });

  function change(index: number, field: RangeField, value: string) {
    setRanges((rs) => rs.map((r, i) => (i === index ? { ...r, [field]: value } : r)));
  }

  function submit() {
    if (!INTEGER.test(decimals.trim())) { setError("casas decimais: use um número inteiro"); return; }
    if (!INTEGER.test(position.trim())) { setError("ordem: use um número inteiro"); return; }
    const parsed = draftsToRanges(ranges);
    if ("error" in parsed) { setError(parsed.error); return; }
    save.mutate({
      id: analyte?.id ?? null, labExamTemplateId: templateId, key: key.trim(), name: name.trim(),
      loinc: orNull(loinc), unit: orNull(unit), valueType,
      choices: valueType === "choice" ? choices.split("\n").map((c) => c.trim()).filter((c) => c !== "") : [],
      decimals: Number(decimals.trim()), position: Number(position.trim()), referenceRanges: parsed.ranges
    });
  }

  return (
    <section aria-label="Componente">
      <Panel title={analyte ? `Editar componente ${analyte.name}` : "Novo componente"}>
        <div style={grid}>
          <Field label="Chave" value={key} onChange={setKey} helpText="minúsculas, números e _ (ex.: hemoglobin)" />
          <Field label="Nome do componente" value={name} onChange={setName} />
          <Field label="LOINC" value={loinc} onChange={setLoinc} />
          <Field label="Unidade" value={unit} onChange={setUnit} />
          <label style={labelStyle}>
            <span>Tipo de valor</span>
            <select value={valueType} onChange={(event) => setValueType(event.target.value)} style={inputStyle}>
              {optionsWith(VALUE_TYPES, valueType).map((t) => <option key={t} value={t}>{valueTypeText(t)}</option>)}
            </select>
          </label>
          {valueType === "choice" && <Field label="Opções (uma por linha)" multiline value={choices} onChange={setChoices} />}
          <Field label="Casas decimais" value={decimals} onChange={setDecimals} />
          <Field label="Ordem" value={position} onChange={setPosition} />
        </div>

        <p style={muted}>
          Idade em dias na data da coleta (365 = 1 ano; 43800 = 120 anos, sem limite). Decimal com vírgula e sem ponto de
          milhar: 150000, não 150.000. Em branco, "baixo" ou "alto" fica sem limite.
        </p>
        <TableScroll>
          <table aria-label="Faixas de referência" style={{ width: "100%", borderCollapse: "collapse", fontSize: 12 }}>
            <thead>
              <tr>
                <th style={th}>Sexo</th>
                {RANGE_COLUMNS.map((c) => <th key={c.field} style={th}>{c.header}</th>)}
                <th style={th} />
              </tr>
            </thead>
            <tbody>
              {ranges.map((r, i) => (
                <tr key={i} style={{ borderBottom: "1px solid var(--rule)" }}>
                  <td style={td}>
                    <select aria-label={`faixa ${i + 1}: sexo`} value={r.sex} onChange={(event) => change(i, "sex", event.target.value)} style={inputStyle}>
                      {optionsWith(SEXES, r.sex).map((s) => <option key={s} value={s}>{sexText(s)}</option>)}
                    </select>
                  </td>
                  {RANGE_COLUMNS.map((c) => (
                    <td key={c.field} style={td}>
                      <input
                        aria-label={`faixa ${i + 1}: ${c.label}`}
                        value={r[c.field]}
                        inputMode={c.field === "text" ? undefined : "decimal"}
                        onChange={(event) => change(i, c.field, event.target.value)}
                        style={{ ...inputStyle, width: c.field === "text" ? 140 : 90 }}
                      />
                    </td>
                  ))}
                  <td style={td}>
                    <Button variant="secondary" onClick={() => setRanges((rs) => rs.filter((_, j) => j !== i))}>remover</Button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </TableScroll>

        {error && <ErrorState message={error} />}
        <div style={row}>
          <Button variant="secondary" onClick={() => setRanges((rs) => [ ...rs, { ...EMPTY_RANGE } ])}>acrescentar faixa</Button>
          <Button onClick={submit} busy={save.isPending}>salvar componente</Button>
          <Button variant="secondary" onClick={onClose}>cancelar</Button>
        </div>
      </Panel>
    </section>
  );
}

const muted: CSSProperties = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap" };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(200px, 1fr))", gap: 12 };
const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const inputStyle: CSSProperties = {
  padding: "8px 10px", border: "1px solid var(--rule2)", borderRadius: 6, background: "var(--panel)", color: "var(--ink)", fontSize: 13
};
const th: CSSProperties = { textAlign: "left", padding: "8px 10px", borderBottom: "1px solid var(--rule)", color: "var(--ink3)", fontWeight: 600 };
const td: CSSProperties = { padding: "6px 10px" };
```

Em `src/screens/LabTemplates.tsx`:

1. Depois de `import { ErrorState } from "../components/ErrorState";`, acrescente `import { LabAnalyteForm } from "./LabAnalyteForm";`.
2. Depois de `type Selection = { kind: "new" } | { kind: "edit"; id: string } | null;`, acrescente `type AnalyteSelection = { kind: "new" } | { kind: "edit"; id: string } | null;`.
3. Depois de `  const [ outcome, setOutcome ] = useState<Outcome | null>(null);`, acrescente `  const [ analyteSelection, setAnalyteSelection ] = useState<AnalyteSelection>(null);`.
4. Em `open`, depois de `    setSelection(next);`, acrescente `    setAnalyteSelection(null);`.
5. Troque `          <Panel title={`Componentes de ${current.name}`}>` por

```tsx
          <Panel
            title={`Componentes de ${current.name}`}
            actions={<Button onClick={() => { setOutcome(null); setAnalyteSelection({ kind: "new" }); }}>novo componente</Button>}
          >
```

6. Na lista de colunas dos componentes, troque

```tsx
                  { key: "ranges", label: "Faixas de referência", render: (a) => rangesSummary(a.referenceRanges) }
```

por

```tsx
                  { key: "ranges", label: "Faixas de referência", render: (a) => rangesSummary(a.referenceRanges) },
                  {
                    key: "edit", label: "",
                    render: (a) => (
                      <Button variant="secondary" onClick={() => { setOutcome(null); setAnalyteSelection({ kind: "edit", id: a.id }); }}>editar</Button>
                    )
                  }
```

7. Logo antes do `</Panel>` que fecha os componentes (depois do `)}` da `DataTable`/`EmptyState`), acrescente

```tsx
            {analyteSelection && (
              <LabAnalyteForm
                key={analyteSelection.kind === "edit" ? analyteSelection.id : "new"}
                templateId={current.id}
                analyte={analyteSelection.kind === "edit" ? current.analytes.find((a) => a.id === analyteSelection.id) ?? null : null}
                nextPosition={current.analytes.reduce((max, a) => Math.max(max, a.position), 0) + 1}
                onSaved={(text) => {
                  setAnalyteSelection(null);
                  setOutcome({ tone: "done", text });
                  void queryClient.invalidateQueries({ queryKey: LAB_TEMPLATES_QUERY_KEY });
                }}
                onClose={() => setAnalyteSelection(null)}
              />
            )}
```

Gere os tipos do documento novo:

```bash
cd $M && npm run codegen
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $M && npx vitest run src/screens/LabAnalyteForm.test.tsx src/screens/LabTemplates.test.tsx src/theme/guards.test.ts && npx tsc --noEmit && npm run codegen:check`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/screens/LabAnalyteForm.tsx src/screens/LabAnalyteForm.test.tsx src/screens/LabTemplates.tsx src/screens/LabTemplates.test.tsx src/theme/guards.test.ts src/gql/gql.ts src/gql/graphql.ts
$G commit -m "feat: edit lab base catalog analytes and their reference ranges

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–5 na branch `feat/mod-21b-lab`; o api do 21b na **3042**.
- Produces: branch do maintenance pronta para merge (depois do merge do api e do dashboard do 21b).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd $M && npm run typecheck && npx vitest run && npm run codegen:check && npm run build`
Expected: tudo verde (os quatro passos da CI): a base anotada na Task 0 mais **4 arquivos de teste novos**. Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $M && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e contagem `0`.

Confira também que o schema do worktree é o do api, sem edição à mão:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml exec -T -w /app/.claude/mod21b api bin/rails runner 'puts Maintenance::Schema.to_definition' | diff - $M/schema.graphql && echo schema-igual
```

Expected: `schema-igual`.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §4 "Plataforma", §7, §11 F-21.14), o ADR 0037 e o contrato §10. Pontos de atenção:
- nenhuma linha nova no fluxo de `setCityFeature`; o aviso sai do estado **ligado** de `fake_middleware`, some quando desliga e nada diz "MIDDLEWARE SIMULADO" sem a chave;
- `schema.graphql` igual ao `to_definition` do api; `src/gql/*` só do `codegen`;
- "150.000" nunca vira número; vírgula decimal; recusa do api nunca apaga o digitado; valor desconhecido nunca trocado em silêncio;
- nenhum token, mensagem HL7, CPF, CNS ou valor de resultado na tela; cores só por papel do tema; a tabela de faixas dentro de `<TableScroll>`.

- [ ] **Step 3: Prova no navegador (com o usuário)**

Depende do api do 21b na **3042** (plano do api). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, com `node_modules` num volume próprio, depois de parar o serviço `maintenance`:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod21b --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod21b:/app -v maintenance-mod21b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3042 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod21b   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 21b num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. **Anote** antes de mexer: o estado dos três interruptores de Curitiba e, no catálogo-base, o preparo da Glicose e as faixas da Hemoglobina. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: as três chaves com o nome em português; com `exam_tracking` desligado, `municipal_lab` com "rastreamento de exames (exam_tracking) desligado"; `lab_hl7_inbound` com "laboratório municipal (municipal_lab) desligado"; `fake_middleware` com "entrada de resultados por HL7 (lab_hl7_inbound) desligada";
- ligar `municipal_lab`, `lab_hl7_inbound` e `fake_middleware` (dois cliques cada) → a frase do estado devolvido; com `fake_middleware` ligado, o aviso "Esta cidade recebe resultados do MIDDLEWARE SIMULADO — …"; com `fake_lab` também ligado, os dois avisos; desligar `fake_middleware` → o aviso some;
- **Laboratório** no trilho: a semente da APS (hemograma, glicemia, HbA1c, lipídios, creatinina, ureia, TGO, TGP, TSH, T4 livre, urina tipo 1, parasitológico, beta-hCG) com setor, material, número de componentes e situação;
- editar Glicose → mudar o preparo → "Exame salvo."; **Auditoria** mostra a tentativa e o resultado;
- Hemograma → editar Hemoglobina → digitar "150.000" num "baixo" → a frase do ponto de milhar e nada enviado; acrescentar uma faixa, salvar → "Componente salvo." e o resumo novo na lista;
- "novo exame" com o SIGTAP da Glicose → "já existe um exame do catálogo-base com este código SIGTAP", com o digitado preservado;
- a 375 px de largura: sem rolagem lateral da página; as tabelas rolam dentro do contêiner com a dica "⇆ role para o lado";
- nenhum token, CPF, CNS ou valor de resultado na tela.

Depois: devolva o preparo da Glicose e as faixas da Hemoglobina ao anotado (pela própria tela) e os três interruptores ao estado anotado; `docker rm -f maintenance-mod21b && docker volume rm maintenance-mod21b-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 21b, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21b-lab
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/maintenance-mod21b
/opt/homebrew/bin/git branch -d feat/mod-21b-lab
```

Se o lado api do F-21.14 já estiver **Done**, mova o card F-21.14 para **Done** (opção `98236657`), com um `gh project item-edit` como na Task 0.

---

### Task 7: O 21b no catálogo público

**Files (repo `apps/catalog`, worktree `$C`):**
- Modify: `src/data/funcionalidades.json`, `src/data/catalog.test.ts`

**Interfaces:**
- Consumes: os F-IDs F-21.14..F-21.24 no board #1 (status real de cada um); o resultado da prova do 21b (planos do api, dashboard e deste); o que foi entregue em api, dashboard e maintenance.
- Produces: módulo `"21"` do JSON com F-21.1..F-21.24 e a descrição sem "etapa seguinte" para o laboratório municipal.

- [ ] **Step 1: Confira o que foi entregue e crie o worktree**

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | test("^F-21\\.(1[4-9]|2[0-4]) ")) | {title, status}'
```

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/catalog fetch origin
$G -C $R/apps/catalog log --oneline -3 origin/main
$G -C $R/apps/catalog worktree add $R/.worktrees/catalog-mod21b -b content/module-21b-catalog origin/main
cd $R/.worktrees/catalog-mod21b && npm ci --silent && npx vitest run 2>&1 | tail -3
```

Expected: os F-21.14..F-21.24 em Done/Verified (os que não estiverem **não entram como disponíveis**: pare e pergunte ao usuário); a suíte do catálogo verde. **Anote**: o módulo 21 do JSON com F-21.1..F-21.13 (do 21a); o status dos módulos 19, 20 e 21 (o rodapé não muda: o 21 já está em validação).

- [ ] **Step 2: Write the failing test**

Em `src/data/catalog.test.ts`, no `describe("módulo 21", …)`, troque `Array.from({ length: 13 }, (_, i) => `F-21.${i + 1}`)` por `Array.from({ length: 24 }, (_, i) => `F-21.${i + 1}`)` e o laço dos disponíveis

```ts
    for (const id of ["f-21.1", "f-21.2", "f-21.3", "f-21.4", "f-21.5", "f-21.6", "f-21.8", "f-21.9", "f-21.10", "f-21.12", "f-21.13"]) {
```

por

```ts
    for (const id of ["f-21.1", "f-21.2", "f-21.3", "f-21.4", "f-21.5", "f-21.6", "f-21.8", "f-21.9", "f-21.10", "f-21.12", "f-21.13",
      "f-21.14", "f-21.15", "f-21.16", "f-21.17", "f-21.18", "f-21.20", "f-21.21", "f-21.22", "f-21.23", "f-21.24"]) {
```

e, no fim do arquivo, acrescente:

```ts
describe("módulo 21 — laboratório municipal (21b)", () => {
  it("HL7 em validação com o middleware simulado; simulados e ICP com nota; descrição atualizada", () => {
    const m21 = catalog.modules.find((m) => m.id === "21")!;
    expect(m21.description).toMatch(/laboratório municipal/);
    expect(m21.description).not.toMatch(/vem numa etapa seguinte/);
    const hl7 = findFeature("f-21.19")!.feature;
    expect(hl7.status).toBe("validating");
    expect(hl7.note).toMatch(/middleware simulado/);
    expect(findFeature("f-21.14")?.feature.note).toMatch(/fora de produção/);
    expect(findFeature("f-21.20")?.feature.note).toMatch(/prestador simulado/);
    expect(findFeature("f-21.14")?.feature.surfaces).toEqual([ "plataforma", "bastidores" ]);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd $C && npx vitest run src/data/catalog.test.ts`
Expected: FAIL — o módulo 21 tem 13 funcionalidades e a descrição antiga.

- [ ] **Step 4: Write minimal implementation**

Em `src/data/funcionalidades.json`:

1. `meta.updatedAt` → a data do dia (`AAAA-MM-DD`).
2. No módulo `"21"`, troque a `description` por:

```json
"description": "O exame pedido é acompanhado até o resultado: autorização, marcação, coleta, laudo e a ciência de quem pediu, com prazos e aviso de atraso. Laboratórios mandam o resultado por um portal ou por integração, e o cidadão acompanha pelo celular. O laboratório municipal trabalha no próprio Rota Saúde, da etiqueta do tubo ao laudo assinado; controle de qualidade e estoque de reagentes vêm numa etapa seguinte."
```

3. Em `features` do módulo 21, depois do objeto do F-21.13, acrescente os onze abaixo. **Os textos são o rascunho**: confira cada frase contra o que foi entregue (cards Verified, telas e regras do api) e corte o que não existir; nada de promessa.

```json
    {
      "id": "F-21.14", "slug": "f-21.14",
      "title": "Interruptores e catálogo-base do laboratório",
      "benefit": "A plataforma liga o laboratório municipal por cidade e mantém o ponto de partida dos exames.",
      "description": "O laboratório municipal, a entrada de resultados de aparelhos e o middleware simulado são ligados por cidade, cada um com os seus pré-requisitos. A equipe da plataforma mantém o catálogo-base dos exames da atenção primária — componentes, unidade, LOINC e faixas de referência de partida —, que cada laboratório adota e ajusta ao seu método.",
      "surfaces": [ "plataforma", "bastidores" ],
      "status": "available",
      "note": "O middleware simulado, usado para demonstração e testes, só existe fora de produção."
    },
    {
      "id": "F-21.15", "slug": "f-21.15",
      "title": "Catálogo do laboratório com versões",
      "benefit": "O laboratório ajusta cada exame ao seu método sem perder o histórico.",
      "description": "O laboratório adota os exames do catálogo-base e ajusta método, material, faixas de referência, limites de digitação e valores críticos. Cada ajuste vira uma versão que não muda mais, e o laudo guarda a versão usada.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.16", "slug": "f-21.16",
      "title": "Requisição, etiqueta e coleta",
      "benefit": "Cada tubo sai identificado com código de barras, na UBS ou no posto do laboratório.",
      "description": "A UBS que coleta ou a recepção do laboratório abre a requisição com os exames pedidos do paciente, ou com a guia em papel. Sai o comprovante com a data prevista e uma etiqueta com código de barras por tubo; a leitura da etiqueta registra quem coletou e quando.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.17", "slug": "f-21.17",
      "title": "Remessa da UBS e recebimento com rejeição",
      "benefit": "O laboratório confere o que chegou, e a UBS sabe na hora o que precisa recoletar.",
      "description": "A UBS monta a remessa lendo as etiquetas e informa a temperatura; o laboratório confere o que chegou e o que faltou. Cada amostra é aceita, rejeitada com motivo — o que gera a recoleta na UBS de origem — ou aceita com restrição, que vai ao laudo.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.18", "slug": "f-21.18",
      "title": "Mapa de trabalho e digitação",
      "benefit": "A bancada sabe o que analisar e digita o resultado com a referência à vista.",
      "description": "O mapa de trabalho mostra, por setor e bancada, o que está para analisar, com filtros de urgência e data, na tela e em PDF. Na digitação, o valor fora da referência para o sexo e a idade fica em destaque, o valor impossível é bloqueado, os calculados se preenchem, e a repetição não apaga o anterior.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.19", "slug": "f-21.19",
      "title": "Resultados de aparelhos por HL7",
      "benefit": "O resultado do aparelho chega sem digitação, e ninguém libera sem conferir.",
      "description": "O sistema do fornecedor do aparelho consulta os exames de cada amostra pelo código de barras e devolve os resultados pelo padrão HL7, com um acesso próprio que a cidade pode revogar. Os resultados entram aguardando conferência, pelas mesmas regras da digitação; nenhum é liberado sozinho.",
      "surfaces": [ "bastidores" ],
      "status": "validating",
      "note": "Provada com um middleware simulado. Falta o primeiro middleware real, do fornecedor do reagente, antes de um laboratório usar."
    },
    {
      "id": "F-21.20", "slug": "f-21.20",
      "title": "Liberação com assinatura e laudo",
      "benefit": "Só o liberador libera, e o laudo traz o que a Anvisa exige.",
      "description": "O liberador do laboratório libera a requisição inteira ou por exame. O laudo traz os itens exigidos pela Anvisa (RDC 978) e um QR de conferência, assinado com certificado ICP-Brasil ou com assinatura eletrônica avançada; médico só com ICP. Liberado, o resultado chega ao exame, a quem pediu e ao cidadão conforme a regra da cidade.",
      "surfaces": [ "secretaria", "cidadao" ],
      "status": "available",
      "note": "A assinatura ICP-Brasil segue em validação com um prestador simulado (veja a assinatura digital). A aceitação da assinatura eletrônica avançada pela vigilância sanitária local é conferida antes do go-live."
    },
    {
      "id": "F-21.21", "slug": "f-21.21",
      "title": "Valores críticos",
      "benefit": "Valor crítico nunca fica sem aviso registrado.",
      "description": "O valor crítico exige registrar quem foi avisado, como e quando. Se o laudo for liberado antes, o exame entra na lista de críticos sem aviso até a comunicação ser registrada.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.22", "slug": "f-21.22",
      "title": "Retificação do laudo",
      "benefit": "Laudo errado é corrigido por um novo, sem apagar o anterior.",
      "description": "O laudo liberado só muda por um novo laudo, com motivo e registro de quem foi avisado. O anterior fica guardado e marcado como retificado; quem pediu volta a dar ciência, e o cidadão vê só o vigente.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.23", "slug": "f-21.23",
      "title": "Relatórios de produção, rejeição e prazo",
      "benefit": "A secretaria sabe quanto o laboratório produziu, o que rejeitou e o que atrasou.",
      "description": "Por mês: produção por unidade de origem, exame, setor e liberador, com quantidade e valor da tabela SIGTAP, na tela e em planilha; rejeições por motivo e unidade; laudos entregues no prazo e atrasados.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    },
    {
      "id": "F-21.24", "slug": "f-21.24",
      "title": "Registro do GAL",
      "benefit": "O exame enviado ao laboratório central do estado não se perde do histórico.",
      "description": "O exame encaminhado ao LACEN guarda o número da requisição do GAL, e o laudo que volta entra como resultado externo do paciente.",
      "surfaces": [ "secretaria" ],
      "status": "available"
    }
```

Se o Step 1 mostrar algum F-21.15..F-21.24 que não chegou a Verified, ou a prova da Task 6 / do dashboard tiver ficado só com simulador em outro ponto, ajuste o `status` para `validating` com `note` que diga o que falta, e espelhe no teste do Step 2.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd $C && npx vitest run && npx tsc --noEmit && npm run build`
Expected: PASS — `validateCatalog` aceita (descrição em todos, telas conhecidas, slugs únicos); o módulo 21 com 24 funcionalidades; o rodapé e o ciclo com os textos de antes.

```bash
cd $C && $G status --short
```

Expected: só os dois arquivos da task (nenhum `package-lock.json`).

- [ ] **Step 6: Commit**

```bash
cd $C
$G add src/data/funcionalidades.json src/data/catalog.test.ts
$G commit -m "docs: add module 21b municipal lab to the catalog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Veja no navegador (com o usuário)**

O `catalog-dev` (5178) lê `apps/catalog`, não o worktree. Suba o Vite do worktree na **5998**:

```bash
cd $C && npx vite --port 5998 --host 127.0.0.1
```

Abra `http://127.0.0.1:5998/` e confira com screenshot: o **Módulo 21 — Exames e laboratório** com a descrição nova e as 24 funcionalidades; a F-21.19 "Em validação" com a nota do middleware simulado; a F-21.20 com a nota da ICP; a página `/funcionalidades/f-21.16` com "O que faz", as telas e anterior/próxima dentro do módulo; a 375 px, sem rolagem lateral. Pare o Vite (Ctrl+C).

- [ ] **Step 8:** **Pare.** Push do branch, PR e merge só com autorização explícita do usuário (uma etapa de cada vez). O catálogo não tem auto-merge: merge manual, com a CI verde e o branch atualizado com `origin/main`. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/catalog-mod21b
/opt/homebrew/bin/git push -u origin content/module-21b-catalog
gh pr create --repo rotasaude/catalog --base main --head content/module-21b-catalog --title "docs: add module 21b municipal lab to the catalog" --body "Adds F-21.14..F-21.24 (municipal lab, 21b) to module 21; HL7 inbound in validation until the first real middleware; ICP signature note; module description updated.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Depois do merge: `/opt/homebrew/bin/git -C $R/apps/catalog worktree remove $R/.worktrees/catalog-mod21b && /opt/homebrew/bin/git -C $R/apps/catalog branch -d content/module-21b-catalog`, e confira o `catalog-dev` (5178).

---

## Divergências propostas ao contrato

1. **M1 — SDL do catálogo-base.** O §10 nomeia `labExamTemplates`, `upsertLabExamTemplate` e `upsertLabAnalyteTemplate` → `{ ok, errors }`, sem tipos nem argumentos. Proposta (o api implementa; o maintenance puxa por `to_definition` e a Task 4 para se divergir):

```graphql
type LabExamTemplate {
  active: Boolean!
  analytes: [LabAnalyteTemplate!]!   # em ordem de position
  container: String!
  id: ID!
  material: String!
  name: String!
  preparation: String
  sector: String!
  sigtapCode: String!
  suggestedMethod: String
}
type LabAnalyteTemplate {
  choices: [String!]!
  decimals: Int!
  id: ID!
  key: String!
  loinc: String
  name: String!
  position: Int!
  referenceRanges: [LabReferenceRange!]!
  unit: String
  valueType: String!
}
type LabReferenceRange { ageMaxDays: Int! ageMinDays: Int! high: Float low: Float sex: String! text: String }
input LabReferenceRangeInput { ageMaxDays: Int! ageMinDays: Int! high: Float low: Float sex: String! text: String }
type UpsertLabExamTemplatePayload { errors: [UserError!]! ok: Boolean! }
type UpsertLabAnalyteTemplatePayload { errors: [UserError!]! ok: Boolean! }
# Query
labExamTemplates: [LabExamTemplate!]!   # ativos e inativos, por nome
# Mutation (id nulo = cria)
upsertLabExamTemplate(active: Boolean!, container: String!, id: ID, material: String!, name: String!, preparation: String, sector: String!, sigtapCode: String!, suggestedMethod: String): UpsertLabExamTemplatePayload
upsertLabAnalyteTemplate(choices: [String!] = [], decimals: Int!, id: ID, key: String!, labExamTemplateId: ID!, loinc: String, name: String!, position: Int!, referenceRanges: [LabReferenceRangeInput!]!, unit: String, valueType: String!): UpsertLabAnalyteTemplatePayload
```

   O payload fica `{ ok, errors }` como no contrato: o exame recém-criado é achado pelo `sigtapCode` (único) depois de reler a lista.
2. **M2 — Códigos de `missing`.** Pelo `Platform::Features#missing_for`, os códigos novos são `municipal_lab_disabled` e `lab_hl7_inbound_disabled` (`exam_tracking_disabled` já existe do 21a). Proposta: o contrato registra os dois, para api e maintenance não divergirem num nome.
3. **M3 — Códigos de setor e de erro.** O §10 e a spec §4 descrevem os setores em português ("bioquímica, hematologia…"). Proposta: `sector` ∈ `biochemistry | hematology | immunology | urinalysis | parasitology | microbiology` (o mesmo vale para o `sector` de `lab_exams` no dashboard, contrato §3). Erros (`UserError.message`, `path` em camelCase): `sigtap_unknown` e `duplicate_sigtap` (os nomes do 21a, A30), `invalid_sector`, `blank` (com o campo em `path`), `duplicate_analyte_key`, `invalid_analyte_key`, `invalid_value_type`, `choices_required`, `invalid_reference_ranges` (idade mínima > máxima, baixo > alto, sexo fora de `male|female|any`), `invalid_loinc`, `not_found`. Código fora da lista aparece cru na tela.
4. **M4 — Sem step-up nas duas mutations.** No maintenance o TOTP do momento (`code`) é exigido onde a escrita emite credencial (`createMaintenanceToken`, `inviteMaintainer`, `rotateCityPecHmacKey`) ou muda o que uma cidade usa agora (`publishProtocol`, `activateProtocol`, `retireProtocol`, `revertProtocolActivation`); o rascunho de protocolo e a importação do catálogo de medicamentos — os análogos mais próximos — não pedem. O catálogo-base só vale na **próxima** adoção (a cidade copia e versiona), então não muda laudo nem exame em uso. Trade-off: sem step-up, uma sessão sequestrada pode alterar a referência de partida de novas adoções; a auditoria da base (tentativa/resultado) e o evento `lab.template_changed` registram, e o laboratório revisa ao adotar ("ajuste ao seu método"). Proposta: o contrato diz "sem `code`". Se o api optar por step-up, o maintenance acrescenta o `code` com `StepUpCode`, como em `PecConfigPanel`.
5. **M5 — Faixa sem limite de idade.** O contrato §3 traz `age_min_days` e `age_max_days` sem `null`. Proposta: os dois obrigatórios, e `43800` (120 anos) como teto convencional de "sem limite"; `low`/`high` nulos = sem limite naquele lado; `text` para referência não numérica ("negativo").
6. **M6 — Aviso do middleware simulado.** O §10 não pede aviso para `fake_middleware`. Proposta (como a A25 do 21a): o maintenance avisa pelo estado **ligado**, porque em staging o `fake-middleware` escreve valores de exemplo em amostras de pacientes da cidade. Só texto na aba, sem consulta nova.
7. **M7 — Sem exclusão no catálogo-base.** O §10 não tem `delete`. Proposta: exame sai de uso por `active: false`; componente não é apagado (é corrigido por edição). Se a curadoria precisar remover um componente antes da primeira adoção, vira mutation nova em outro ciclo.

## Self-review

- **Cobertura (spec §3, §4 "Plataforma", §7 "Maintenance", §10 itens 4 e 5, §11 F-21.14; contrato §10):** os três interruptores pelo mecanismo genérico com os `requires` do contrato — Tasks 1 e 2; frases de `missing` — Task 1 (e Task 2 na lista); aviso do middleware simulado no padrão do laboratório simulado — Tasks 1 e 2 (Divergência M6); `fake_middleware` só fora de produção — Task 2 ("catálogo sem fake_middleware"); catálogo-base com `labExamTemplates`, componentes e faixas — Tasks 3, 4 e 5; `upsertLabExamTemplate` e `upsertLabAnalyteTemplate` seguindo o padrão GraphQL (`graphql()` + `gql()` + `{ ok, errors }`, consulta própria, recusa do envelope, api antigo) e o `codegen:check` — Tasks 4, 5 e 6; step-up decidido com trade-off — Divergência M4; tema rota, `<TableScroll>` e cores por papel — Global Constraints, guarda nova na Task 5 e a suíte de guardas nas Tasks 2, 4 e 5; prova no navegador — Task 6; entrada do 21b no catálogo público como task final, JSON como fonte única — Task 7; card — Tasks 0 e 6.
- **Placeholders:** nenhum código faltando. Variáveis de ambiente lidas na execução: ids do card e da opção In Progress (Task 0); status dos cards F-21.14..F-21.24 (Task 7, Step 1, com a regra de ajuste escrita).
- **Tipos:** `TemplateView`, `AnalyteView`, `RangeDraft`, `EMPTY_RANGE`, `draftsToRanges`, `rangeToDraft`, `optionsWith`, `orNull`, `upsertErrorText` (Task 3) são os usados nas Tasks 4 e 5; `LAB_TEMPLATES_QUERY_KEY` (Task 4) é o da invalidação na Task 5; os nomes de operação (`LabExamTemplates`, `UpsertLabExamTemplate`, `UpsertLabAnalyteTemplate`) são os que os testes roteiam; `UpsertLabAnalyteTemplateMutationVariables` sai do codegen do documento da Task 5; os campos e argumentos batem com o SDL da Divergência M1; `FAKE_MIDDLEWARE_NOTICE`/`fakeMiddlewareOn` (Task 1) são os da Task 2.
- **Review Focus:** 1 — Task 3 ("ponto de milhar é ambíguo e é recusado") e Task 5 ("novo componente: ponto de milhar é recusado antes de enviar"); 2 — Task 4 ("recusa do api: mostra a frase e mantém o que foi digitado") e Task 5 ("recusa do api (faixas inválidas)…"); 3 — Task 3 ("optionsWith mantém o valor desconhecido"), Task 4 ("setor desconhecido…") e Task 5 ("tipo e sexo desconhecidos…"); 4 — Task 2 ("catálogo sem fake_middleware (produção)…", "laboratório e middleware simulados ligados: dois avisos…"); 5 — Task 4 ("api sem o 21b: explica a ordem de deploy").
