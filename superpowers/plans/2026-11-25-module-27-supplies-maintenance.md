# Módulo 27 — Insumos (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-25-module-27-supplies-contracts.md` §1, §2, §8 e Q5, Q6 e Q13 (docs `7348ae0`). As divergências DM1–DM3 foram resolvidas pela Q6 e pela Q13 (seção final). O plano do api é `2026-11-25-module-27-supplies-api.md` (Valor 1, Task 2); a Task 0 confere contra o api real.
> 2. O **api do 27 (insumos)** na branch `feat/mod-27-supplies` (worktree `apps/api/.claude/mod27s`, porta **3065**), com as duas chaves no fim de `Platform::Features::CATALOG` e a mutation `importSupplyCatalog` no `Maintenance::Schema`.
> 3. A `main` do maintenance **com o 26c mergeado** (o contrato manda executar depois dele). Quando este plano foi escrito, o `origin/main` estava em `14406bd`, com o 35b (`zoonoses` é a última entrada de `FEATURE_LABELS`) e sem o 27b.
>
> A Task 1 não depende do api. A Task 2 depende do SDL do api do 27. A prova da Task 3 depende do api rodando, com o banco de dev migrado **com autorização do usuário**.

**Goal:** Lado maintenance dos insumos, em duas partes:
- os dois interruptores, com nome e frase de pré-requisito, na aba Funcionalidades;
- "Importar catálogo de materiais" (CATMAT, classes 6515 e 6550), fora de produção, na tela de medicamentos da plataforma.

**Architecture:** Conferido no código (`origin/main` do maintenance em `14406bd`).
- **Aba Funcionalidades.** O liga/desliga é genérico (`FeaturesTab.tsx`, `setCityFeature` sem TOTP, três estados por `featureStatus`). Só entram entradas em `FEATURE_LABELS` e `MISSING_LABELS` (`src/lib/features.ts`).
- **Catálogo.** A tela "Medicamentos" (`Shell.tsx:28`, `screen === "medications"`) monta o `MedicationCatalog`, que importa o CATMAT 6505 por `importMedicationCatalog` **sem TOTP**, com `ConfirmButton` (`src/screens/MedicationCatalog.tsx:37-41`, `:133-139`). A mutation do api faz a mesma coisa: síncrona, sem argumento, auditada (`apps/api app/graphql/maintenance/mutations/import_medication_catalog.rb`).
  - O catálogo de materiais entra como um painel novo **com consulta própria**, logo abaixo, no molde da importação do CATMAT. Contra um api sem o 27, só o painel cai.
  - O maintenance só existe fora de produção (`src/env.ts`: `development` | `staging`). Por isso, "fora de produção" (Q5) vale sozinho aqui. Em produção, a carga é pela rake `supplies:import_catmat`.
- **GraphQL.** Documentos tipados pelo codegen (`src/gql/`, a partir de `schema.graphql`, puxado do api). A recusa chega em `payload.errors: [{ path, message }]`, e o `message` é o código (`importErrorText`).

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-25-module-27-supplies-design.md` (decisões 3 e 4; §3.1). **Contrato:** `docs/superpowers/plans/2026-11-25-module-27-supplies-contracts.md` §1, §2, §8 e Q5. **Molde:** `docs/superpowers/plans/2026-11-24-module-27b-bnafar-maintenance.md`; `MedicationCatalog.tsx` e o seu teste.

## Global Constraints

- **Interruptores** (contrato §1), nesta ordem, no fim do `CATALOG`:

  | Chave | Rótulo (letra por letra) | Exige |
  |---|---|---|
  | `pharmacy_supplies` | "Insumos na farmácia" | `feature:pharmacy` |
  | `supply_programs` | "Programas de insumos (diabetes e estomia)" | `feature:pharmacy_supplies` e `feature:clinical_record` |

  Os dois existem em todos os ambientes, sem simulado. A bolsa no faturamento exige também `procedure_billing`, mas isso é regra da entrega no api, não um `requires` do interruptor.
- **Frases de pré-requisito** (`MISSING_LABELS`):
  - entra `pharmacy_supplies_disabled` (código fixado pelo §8) com a frase **proposta** "insumos na farmácia (pharmacy_supplies) desligados", no molde das existentes;
  - o prontuário ausente em `supply_programs` usa `clinical_record_disabled`, que já tem frase (Q13);
  - não mudam as que já existem: `pharmacy_disabled` e `clinical_record_disabled`.
- **Sem cascata:** ligar ou desligar manda só aquela `key`. Desligar `pharmacy_supplies` deixa `supply_programs` "ligada, falta pré-requisito".
- **Catálogo de materiais** (contrato §2, §8, Q5):
  - o botão fixado é **"Importar catálogo de materiais"**;
  - ele chama a mutation `importSupplyCatalog`, sem argumento e sem TOTP, no molde do `importMedicationCatalog` (Q6);
  - o resultado diz o que entrou, mudou e saiu;
  - uma importação por vez: enquanto uma das três corre, as três ficam desabilitadas (`IMPORT_RUNNING`);
  - o painel mostra a versão atual e a contagem por grupo. Os itens em `other` e os com `unit_review` (unidade de fornecimento escolhida sem `UN`) aparecem para revisão, **só leitura** (Q6). A revisão de fato é no YAML de regras do api (`config/supplies/groups.yml`), recarregado pelo operador.
- **Grupos** (contrato §2), com rótulos **propostos**, os mesmos do plano do dashboard:

  | Grupo | Rótulo |
  |---|---|
  | `glucose_strip` | "tira de glicemia" |
  | `lancet` | "lanceta" |
  | `insulin_syringe` | "seringa de insulina" |
  | `pen_needle` | "agulha de caneta" |
  | `ostomy_bag` | "bolsa de estomia" |
  | `ostomy_accessory` | "acessório de estomia" |
  | `other` | "outro material" |

  Grupo desconhecido aparece cru.
- **O que passa aqui:** só o catálogo da plataforma (códigos, descrições, unidades, grupos, contagens). Nenhum dado de paciente, inscrição ou cota: tudo isso é da cidade, no dashboard.
- **Frases da tela:** toda frase que o contrato não fixa é **proposta** e está marcada assim. Código desconhecido aparece cru, nunca `undefined`/`null`.
- **Interface:**
  - componentes: `Panel`, `DataTable`, `Tag`, `Button`, `ConfirmButton`, `ErrorState`, `EmptyState`;
  - cores só por papéis do tema, com `src/theme/*.test.ts` na suíte;
  - 375 px sem rolagem lateral da página.
- **Testes:** só `bin/test maintenance -w maintenance-mod27s [arquivos]` (de `$R`), nunca `vitest` direto. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge e push só com autorização explícita.
- **Board #1:** o maintenance é superfície dos cards do 27 dos interruptores e do catálogo de materiais. O card vai a In Progress pelo plano do api; a Task 0 só confere.
- **Execução coordenada:** ao fim de cada task, uma linha de relato à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod27s`, `A=$R/apps/api/.claude/mod27s`.
- Worktree **fora de `apps/`**: `$M`, na branch `feat/mod-27-supplies` a partir de `origin/main`. Os caminhos das tasks são relativos a `$M`.
- **Prova:**
  - Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3065**;
  - abra `http://maintenance.localhost:5177`, nunca `localhost:5177`;
  - conta `dev@local` / `dev-password`, com TOTP fixo (`lib/dev_maintainer.rb`); o usuário faz o login e o TOTP.

## Review Focus

1. **Desligar `pharmacy_supplies` com `supply_programs` ligado.** Espera-se:
   - só `pharmacy_supplies` vai na mutation;
   - depois do refetch, `supply_programs` diz "ligada, falta pré-requisito", com a frase de `pharmacy_supplies_disabled`, e não é desligado junto.

   Teste: Task 1, "desligar os insumos não desliga os programas".
2. **`supply_programs` com o prontuário fora do modo `record`.** Espera-se a frase de `clinical_record_disabled`, que já existe, sem frase nova nem código cru. Teste: Task 1, "programas sem prontuário".
3. **Importar duas vezes, ou importar enquanto o CATMAT de medicamentos corre.** Espera-se:
   - duplo clique manda uma mutation;
   - com qualquer importação em curso, as três ficam desabilitadas e aparece o `IMPORT_RUNNING`;
   - `import_in_progress` do api mostra a frase.

   Teste: Task 2, "uma importação por vez entre as três".
4. **O Compras.gov fora do ar ou com poucos itens.** Espera-se que `source_unavailable`, `incomplete_source` e `suspicious_drop` mostrem as frases que já existem e digam que nada mudou, e que a versão seja relida mesmo depois de rede ou timeout. Teste: Task 2, "recusas da fonte e rede caída relêem a versão".
5. **Api sem o 27 ou com um grupo futuro.** Espera-se:
   - contra um api sem `supplyCatalog`, só o painel de materiais mostra a frase do api antigo, e o de medicamentos segue;
   - um grupo desconhecido aparece cru na contagem.

   Teste: Task 2, "api antigo derruba só o painel; grupo desconhecido cru".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx` | nomes e frase de pré-requisito | 1 |
| `schema.graphql`, `src/gql/*`, `src/lib/supplies.ts`, `src/lib/supplies.test.ts`, `src/screens/SupplyCatalogPanel.tsx`, `src/screens/SupplyCatalogPanel.test.tsx`, `src/screens/MedicationCatalog.tsx`, `src/screens/MedicationCatalog.test.tsx`, `src/screens/Shell.tsx` | SDL do 27; "Importar catálogo de materiais" | 2 |
| `README.md` | parágrafo do módulo; suíte, codegen:check, build, revisão, prova na 5177, parada | 3 |

**Estratégia de teste:**
- **Funções puras** (`featureLabel`, `missingText`, `groupLabel`, `supplyReleaseText`, `supplyImportText`, `groupCountsText`), por tabela de casos.
- **As telas**, pelo fluxo real, com o `fetch` falso que responde pelo nome da operação. Na aba, os helpers do topo de `FeaturesTab.test.tsx`; no painel, `reply`, `bodyOf`, `calls`, `route` e `renderScreen`, copiados do topo de `MedicationCatalog.test.tsx`. Os testes conferem:
  - as variáveis exatas;
  - o que **não** sai (nenhuma segunda mutation);
  - a releitura depois do ato;
  - a frase de cada recusa.
- **A Task 3:** a suíte inteira com as guardas de tema, o `codegen:check`, o build e a prova contra o api do 27 em Curitiba.

---

### Task 0: Card, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (com o 26c); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); `$A`.
- Produces: `$M`; a base anotada; as `description` reais das duas chaves; o SDL real do 27 comparado com as formas da Task 2.

- [ ] **Step 1: Conferir os cards do 27.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-27\\.")) | select(.title | test("(?i)insumo|materia|interruptor")) | {id, title, status}'
```

Expected: os cards em In Progress ou além. Em Todo, mova com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod27s
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -5 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n '^  zoonoses:\|^  bnafar\|home_visits_disabled:\|pharmacy_disabled:\|clinical_record_disabled:\|supply'
$G -C $R/apps/maintenance show origin/main:src/screens/MedicationCatalog.tsx | grep -n 'label="importar as listas da Anvisa"\|const running ='
$G -C $R/apps/maintenance show origin/main:src/screens/Shell.tsx | grep -n 'screen === "medications"'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*.*(módulo [0-9]*[a-z]*)\.\*\*' | tail -3
grep -n 'key: "pharmacy_supplies"\|key: "supply_programs"' -A4 $A/app/services/platform/features.rb
grep -rn 'importSupplyCatalog\|ImportSupplyCatalog\|supplyCatalog\|SupplyCatalog' $A/app/graphql | head -20
grep -rn 'supply_catalog' $A/app/services/platform/audit*.rb $A/config/initializers 2>/dev/null | head -5
```

Expected:
- em `features.ts`, a última entrada de `FEATURE_LABELS` é a âncora: `zoonoses`, ou a do 27b ou do 26c, se entraram depois. Nenhuma linha tem `supply`;
- no api:
  - as duas chaves no fim do `CATALOG`, nesta ordem, com os `requires` do §1;
  - `importSupplyCatalog` e a consulta do catálogo de materiais no GraphQL;
  - o evento de auditoria `maintenance.supply_catalog.imported`, na lista R18 (Q13);
  - se o SDL pedir `code` (TOTP) na mutation, ou não tiver a consulta do catálogo de materiais, **avise o coordenador** antes da Task 2: a Q6 decidiu a carga sem TOTP e a revisão só leitura, que depende da consulta. O plano do api (Task 2) só cita a mutation, e o teste dele menciona TOTP.

**Anote** a `description` real de cada chave, os campos reais do SDL e o código real da recusa de pré-requisito. **Onde o api diz outra coisa, vale o api**: a task dona ajusta o documento GraphQL e o teste, e o relato ao coordenador cita a diferença.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod27s -b feat/mod-27-supplies origin/main
cd $R/.worktrees/maintenance-mod27s && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod27s
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos).

**Relato ao coordenador:** "maintenance-mod27s Task 0: worktree e base verdes (N arquivos/M casos); chaves do api conferidas (ou ausentes); SDL do 27 <presente|ausente>; diferenças: <…>."

---

### Task 1: Nomes dos interruptores e frase de pré-requisito

**Files:**
- Modify: `src/lib/features.ts`
  - uma entrada em `MISSING_LABELS`, depois da última;
  - duas em `FEATURE_LABELS`, depois da âncora da Task 0.
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary`, `doneText`, `featureStatus` (`features.ts`); os helpers do topo de `FeaturesTab.test.tsx` (`reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`) e o padrão de lista inicial → lista depois da mutation do bloco do 35a.
- Produces: os rótulos e a frase das Global Constraints, com o comentário de uma linha: "Módulo 27 (insumos): pharmacy_supplies exige pharmacy; supply_programs exige pharmacy_supplies e clinical_record (missing: pharmacy_supplies_disabled, clinical_record_disabled). Todos os ambientes, sem simulado; a bolsa no faturamento exige também procedure_billing, conferido na entrega."

- [ ] **Step 1: Write the failing tests**

  No fim de `src/lib/features.test.ts`, um `describe("módulo 27 — insumos")` com:
  - os dois `featureLabel`, letra por letra;
  - `missingText("pharmacy_supplies_disabled")` com a frase;
  - `missingSummary([ "pharmacy_supplies_disabled", "clinical_record_disabled" ])` com as duas frases unidas por "; ";
  - um código futuro (`supply_catalog_missing`), que volta cru;
  - `pharmacy` e `zoonoses` sem mudança.

  No fim de `src/screens/FeaturesTab.test.tsx`, um `describe("FeaturesTab — insumos na farmácia (módulo 27)")`, com as constantes `SUP_OFF`, `SUP_ON`, `SUP_BLOCKED` (`missing: [ "pharmacy_disabled" ]`), `PROG_OFF`, `PROG_ON`, `PROG_BLOCKED` (`missing: [ "pharmacy_supplies_disabled" ]`) e `PROG_NO_RECORD` (`missing: [ "clinical_record_disabled" ]`), com as `description` anotadas na Task 0:

| Caso | Lista inicial → depois da mutation | Clique | Asserções |
|---|---|---|---|
| "as duas chaves com nome e descrição do api" | `[SUP_OFF, PROG_OFF]` | — | os dois rótulos e as descrições; `getAllByText("desligada")` com 2; corpo sem `/undefined\|null/` |
| "ligar os insumos sem a farmácia: ligada, falta a farmácia" | `[SUP_OFF]` → `[SUP_BLOCKED]` | "ligar", "confirmar: ligar pharmacy_supplies" | variáveis `{ citySlug: "sp", key: "pharmacy_supplies", enabled: true }`; o status com a frase de `pharmacy_disabled` |
| "desligar os insumos não desliga os programas" | `[SUP_ON, PROG_ON]` → `[{ ...SUP_ON, enabled: false, usable: false }, PROG_BLOCKED]` | primeiro "desligar", "confirmar: desligar pharmacy_supplies" | 1 mutation, só com `pharmacy_supplies`; `supply_programs` em "ligada, falta pré-requisito", com "insumos na farmácia (pharmacy_supplies) desligados" |
| "programas sem prontuário" | `[SUP_ON, PROG_NO_RECORD]` | — | a frase de `clinical_record_disabled` na linha |
| "missing desconhecido cru" | `[{ ...PROG_ON, usable: false, missing: [ "supply_catalog_missing" ] }]` | — | "supply_catalog_missing" na linha |

- [ ] **Step 2: Run tests to verify they fail** — `bin/test maintenance -w maintenance-mod27s src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos com rótulo e frase; os antigos passam.
- [ ] **Step 3: Implement** pelas Interfaces.
- [ ] **Step 4: Run tests to verify they pass** — o mesmo comando, e `cd $M && npx tsc --noEmit`. Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx
$G commit -m "feat: name the pharmacy supplies and supply programs switches and their prerequisite" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: SDL do 27 e "Importar catálogo de materiais"

**Files:**
- Modify: `schema.graphql`, `src/gql/*` (gerados), `src/screens/MedicationCatalog.tsx`, `src/screens/MedicationCatalog.test.tsx`, `src/screens/Shell.tsx`
- Create: `src/lib/supplies.ts`, `src/lib/supplies.test.ts`, `src/screens/SupplyCatalogPanel.tsx`, `src/screens/SupplyCatalogPanel.test.tsx`

**Interfaces:**
- Consumes: o SDL do api do 27 (Task 0); `IMPORT_RUNNING`, `countText`, `importErrorText`, `fmtWhen` (`src/lib/medications.ts`, `src/lib/analytics.ts`); `GraphQLRefusal`; o padrão das duas importações do `MedicationCatalog`.
- Produces: as formas do SDL que este plano usa (Q6: payload no molde do `ImportMedicationCatalog`; a consulta é a forma deste plano, que a Task 0 confere). O SDL real manda:

```graphql
type Query { supplyCatalog: SupplyCatalog }
type SupplyCatalog {
  currentRelease: SupplyCatalogRelease
  groupCounts: [SupplyGroupCount!]!
  reviewItems(first: Int!): [SupplyCatalogItem!]!          # os de group "other" e os com unitReview
}
type SupplyCatalogRelease { id: ID! importedAt: ISO8601DateTime! itemsCount: Int! hiddenCount: Int! }
type SupplyGroupCount { group: String! count: Int! }
type SupplyCatalogItem { catmatCode: Int! classCode: Int! pdmName: String! description: String! supplyUnit: String! unitReview: Boolean! group: String! sigtapCode: String }
type Mutation { importSupplyCatalog: ImportSupplyCatalogPayload }
type ImportSupplyCatalogPayload { ok: Boolean! errors: [UserError!]! releaseId: ID added: Int changed: Int removed: Int }
```

- `src/lib/supplies.ts` (todas as frases são **proposta**; desconhecido cru; nulo vira "—"):
  - `GROUP_LABELS` e `groupLabel(group)`, a tabela das Global Constraints;
  - `supplyReleaseText(r)`:
    - sem versão: "nenhuma importação ainda — até a primeira, a farmácia não acha material nas entradas";
    - com versão: "importado em <quando> · <n> itens · <n> ocultos para revisão";
  - `groupCountsText(counts)`: "<rótulo>: <n>", unidos por " · ", na ordem da tabela e com os desconhecidos no fim;
  - `supplyImportText(r)`:
    - `ok`: "Catálogo de materiais importado: <n> entraram, <n> mudaram, <n> saíram." Com `removed > 0`, acrescenta "Os que saíram deixam de aparecer nas entradas; lotes já recebidos não mudam.";
    - `!ok`: `importErrorText(errors)`, que já traduz `source_unavailable`, `incomplete_source`, `suspicious_drop`, `import_failed` e `import_in_progress`.
- `SupplyCatalogPanel({ disabled, onRunningChange })`:
  - consulta própria `SupplyCatalog` (chave `[ "platform", "supplyCatalog" ]`, `staleTime: Infinity`, botão "atualizar");
  - `Panel` com o título "Catálogo de materiais — CATMAT, classes 6515 e 6550 (todas as cidades do ambiente)" (proposta);
  - o parágrafo (proposta): "A importação traz os materiais de uso em saúde do CATMAT (Compras.gov) e diz o que entrou, mudou e saiu. Cada material recebe um grupo pela regra do PDM. Os que ficam em "outro material" ou com a unidade a conferir aparecem abaixo; a correção é na regra do api, e a próxima carga aplica. Em produção, a carga é feita pela rake do operador.";
  - `supplyReleaseText`, `groupCountsText`;
  - o `ConfirmButton` **"Importar catálogo de materiais"** (confirmação "confirmar: importar o catálogo de materiais agora", proposta), que chama `importSupplyCatalog`;
  - "Materiais para revisar" (proposta): um `DataTable` só leitura com CATMAT, classe, PDM, descrição, unidade de fornecimento (com a `Tag` "unidade a conferir" quando `unitReview`, proposta), grupo e SIGTAP. Nenhum botão de edição (Q6);
  - contra um api sem o 27 (`undefinedField`), a frase "esta API ainda não tem o catálogo de materiais — o api do módulo 27 precisa subir antes do maintenance".
- `MedicationCatalog`:
  - monta o `SupplyCatalogPanel` depois de "Itens para revisar";
  - `running` passa a contar também a importação de materiais, que sobe por `onRunningChange`;
  - o painel recebe `disabled` quando corre uma das duas do CATMAT de medicamentos;
  - o `IMPORT_RUNNING` aparece uma vez.
- `Shell.tsx`: o rótulo do trilho "Medicamentos" passa a "Medicamentos e materiais" (proposta). A chave `medications` não muda.

- [ ] **Step 1: Puxar o SDL e gerar os tipos** (com o api do 27 de pé no container)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; M=$R/.worktrees/maintenance-mod27s
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod27s api bin/rails runner 'puts Maintenance::Schema.to_definition' > $M/schema.graphql
cd $M && /opt/homebrew/bin/git diff --stat schema.graphql && npm run codegen && npx tsc --noEmit && echo sdl-ok
```

Expected: o diff só acrescenta tipos e campos do 27, sem nada removido, e termina em `sdl-ok`. Remoção ou campo renomeado: **pare e reporte** ao coordenador. Campos diferentes dos propostos: a Task segue o SDL real e o relato cita a diferença.

- [ ] **Step 2: Write the failing tests**
  - `src/lib/supplies.test.ts`:
    - `groupLabel("ostomy_bag")`;
    - `groupLabel("catheter")` = `"catheter"`;
    - `groupCountsText([ { group: "lancet", count: 12 }, { group: "glucose_strip", count: 40 }, { group: "catheter", count: 3 } ])` = `"tira de glicemia: 40 · lanceta: 12 · catheter: 3"`;
    - `supplyReleaseText(null)`;
    - `supplyImportText` com `removed: 0` e com `removed: 2`;
    - `supplyImportText({ ok: false, errors: [ { message: "suspicious_drop" } ] })` com a frase existente.
  - `src/screens/SupplyCatalogPanel.test.tsx` (helpers copiados do topo de `MedicationCatalog.test.tsx`):

| Caso | Asserções |
|---|---|
| "versão, grupos e itens para revisar, sem undefined/null" | o texto da versão, a contagem por grupo, as linhas de revisão com "unidade a conferir" no item com `unitReview`; nenhum botão de edição na tabela; corpo sem `/undefined\|null/` |
| "importar: uma mutation sem variáveis, frase e releitura" | "Importar catálogo de materiais" → "confirmar…" → 1 `calls(fetchMock, "mutation ImportSupplyCatalog")`, com `variables` vazio ou ausente; a frase com as contagens; a consulta relida |
| "recusas da fonte e rede caída relêem a versão" | `ok: false` com `source_unavailable` → a frase, sem releitura obrigatória; `fetch` rejeitado → `ErrorState` e a consulta relida |
| "api antigo derruba só o painel; grupo desconhecido cru" | `errors: [{ extensions: { code: "undefinedField" } }]` na consulta → a frase do api antigo; `groupCounts` com `catheter` → "catheter: 3" |

  - `src/screens/MedicationCatalog.test.tsx`, o caso "uma importação por vez entre as três":
    - com a importação de materiais pendente, "importar o CATMAT" e "importar as listas da Anvisa" ficam desabilitados, e o `IMPORT_RUNNING` aparece uma vez;
    - com a do CATMAT pendente, "Importar catálogo de materiais" fica desabilitado;
    - duplo clique no de materiais manda uma mutation;
    - `ok: false` com `import_in_progress` → a frase.
- [ ] **Step 3: Run tests to verify they fail** — `bin/test maintenance -w maintenance-mod27s src/lib/supplies.test.ts src/screens/SupplyCatalogPanel.test.tsx src/screens/MedicationCatalog.test.tsx src/screens/Shell.test.tsx`. Expected: FAIL nos casos novos.
- [ ] **Step 4: Implement** pelas Interfaces; rode e veja passar; `npx tsc --noEmit && npm run codegen:check`. Se `Shell.test.tsx` ou `Shell.rail.test.tsx` conferirem o rótulo "Medicamentos", ajuste a asserção ao rótulo novo no mesmo commit.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add schema.graphql src/gql src/lib/supplies.ts src/lib/supplies.test.ts src/screens/SupplyCatalogPanel.tsx src/screens/SupplyCatalogPanel.test.tsx src/screens/MedicationCatalog.tsx src/screens/MedicationCatalog.test.tsx src/screens/Shell.tsx
$G commit -m "feat: import the CATMAT supplies catalog (classes 6515 and 6550) from the platform medications screen" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Acrescente ao `add` os testes do `Shell` que o Step 4 tocou.)

---

### Task 3: README, suíte, build, revisão e prova no navegador

**Files:** `README.md`.

- [ ] **Step 1: README.** Um parágrafo **Insumos na farmácia (módulo 27).** depois do último parágrafo de módulo (âncora da Task 0). Ele diz:
  - os dois interruptores e o que cada um exige, com as frases da aba;
  - que a bolsa no faturamento exige também `procedure_billing`, conferido na entrega e não no interruptor;
  - o que "Importar catálogo de materiais" faz (CATMAT 6515 e 6550, sem TOTP, grupo pela regra do PDM no YAML do api, revisão só leitura dos de "outro material" e dos com a unidade a conferir; Q6);
  - que, em produção, a carga é pela rake `supplies:import_catmat` (Q5);
  - que cotas, inscrições, comodato e o ponto de estomia são da cidade, no dashboard, e nada disso passa aqui;
  - a ordem de deploy: api, depois dashboard e maintenance;
  - que, contra um api sem o 27, o painel de materiais mostra a frase do api antigo;
  - que o teste do catálogo público que proíbe "insumo|material" muda no fechamento do módulo, no `apps/catalog` (§8), e não aqui.

  Commit:

```bash
cd $M && $G add README.md
$G commit -m "docs: describe the supplies switches and the supplies catalog import in the maintenance README" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 2: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod27s && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base mais 2 arquivos de teste novos (`src/lib/supplies.test.ts` e `src/screens/SupplyCatalogPanel.test.tsx`) e casos novos em `features.test.ts`, `FeaturesTab.test.tsx` e `MedicationCatalog.test.tsx`;
- `codegen:check` limpo;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 3: Varreduras**

```bash
cd $M
grep -rnE "localStorage|sessionStorage|console\.(log|info|warn|error)" src/screens/SupplyCatalogPanel.tsx src/lib/supplies.ts && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "patient|cpf|cns|enrollment|quota" src/screens/SupplyCatalogPanel.tsx src/lib/supplies.ts && echo "ERRO: dado da cidade no maintenance" || echo ok-city
```

Expected: `ok-storage`, `ok-city`.

- [ ] **Step 4: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 3 e 4) e o contrato (§1, §2, §8, Q5). Ele confere:
  - os rótulos e o botão letra por letra;
  - a carga sem TOTP e a revisão só leitura (Q6);
  - nenhuma cascata no liga/desliga;
  - uma importação por vez entre as três;
  - a consulta própria do painel;
  - nenhum dado de cidade;
  - um README sem promessa de carga em produção pelo maintenance.

- [ ] **Step 5: Prova no navegador na 5177 (com o usuário).** Depende do api do 27 na **3065**, com os bancos de dev migrados **com autorização do usuário** (plano do api). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod27s --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod27s:/app -v maintenance-mod27s-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3065 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod27s   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz o login e o TOTP. Antes de tudo, anote o estado das duas chaves em Curitiba e Maringá. As screenshots vão em `evidencias/2026-11-25-modulo-27-insumos/maintenance/`. Roteiro:
1. **Funcionalidades (Curitiba):**
   - as duas chaves aparecem com nome e descrição;
   - ligar `supply_programs` com `pharmacy_supplies` desligado → "ligada, falta pré-requisito", com a frase dos insumos;
   - ligar `pharmacy_supplies` → `supply_programs` fica "ligada e utilizável" depois do refetch;
   - desligar `pharmacy_supplies` não desliga `supply_programs`.
2. **Medicamentos e materiais:**
   - "Importar catálogo de materiais" → a frase com as contagens e a contagem por grupo;
   - os de "outro material" na revisão;
   - durante a importação, os botões do CATMAT de medicamentos ficam desabilitados.
3. **Combinado com a sessão do dashboard (5226):** depois da carga, a entrada de estoque acha "tira" no tipo "Insumo".
4. **Auditoria:** aparece `maintenance.supply_catalog.imported` (Q13), só com ids e contagens.
5. **A 375 px:** o painel novo não tem rolagem lateral da página.

Depois, volte as chaves ao estado anotado e limpe: `docker rm -f maintenance-mod27s && docker volume rm maintenance-mod27s-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 6: Pare.** O merge só vem depois do api do 27 (a suíte completa do api é da sessão "Testes", pedida pelo plano do api) e só com autorização explícita, uma etapa de cada vez: merge, push e card.
  - Antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas).
  - Quando autorizado: `git fetch`, `merge --ff-only feat/mod-27-supplies`, `push origin main`, `worktree remove $M`, `branch -d`.
  - Os cards vão a **Done** só com as partes do api e do dashboard também entregues, e a **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod27s Task 3: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-25-modulo-27-insumos/maintenance/); aguardando autorização de merge."

---

## Self-review

- **Cobertura** (contrato §8, maintenance):
  - os dois interruptores: Task 1;
  - `MISSING_LABELS` com `pharmacy_supplies_disabled`: Task 1;
  - "Importar catálogo de materiais", fora de produção (Q5): Task 2;
  - o teste do catálogo público: fica com o `apps/catalog` no fechamento (§8), citado no README.
- **Placeholders:** nenhum. Valores lidos na execução: os ids dos cards, as `description` das chaves, o SDL real e o nome do evento de auditoria (Task 0). Se o SDL remover ou renomear algo, a execução para.
- **Tipos e nomes:** existem em `14406bd`: `featureLabel`, `missingText`, `missingSummary`, `doneText`, `featureStatus`, `gql`, `graphql`, `GraphQLRefusal`, `ConfirmButton`, `IMPORT_RUNNING`, `countText`, `importErrorText`, `fmtWhen`, os helpers de teste do `MedicationCatalog.test.tsx` e do `FeaturesTab.test.tsx`. Os campos GraphQL vêm do SDL do 27 (Task 2).
- **Proporção:** as mudanças de produção são três entradas de tabela, um arquivo de rótulos, um painel e duas linhas no `MedicationCatalog` e no `Shell`.

## Divergências propostas ao contrato

Todas foram decididas no §11 do contrato (docs `7348ae0`), e o plano já aplica as decisões. O texto de cada uma fica como registro, resumido.

- **DM1 — Resolvida pela Q6.** A carga não tinha consulta nem forma de resposta, e o grupo "revisável" não dizia onde (`src/screens/MedicationCatalog.tsx:28-35` é a consulta do molde). **Decisão:**
  - as releases ficam em tabela própria;
  - a unidade de fornecimento vem da PDM, com `unit_review`;
  - o grupo sai de regras num YAML, revisado ali e recarregado;
  - no maintenance, a revisão é **só leitura**.

  A Task 2 mostra os de `other` e os com `unitReview`, sem edição. A consulta `supplyCatalog` segue a forma deste plano. O plano do api só cita a mutation, então a Task 0 confere e avisa o coordenador se ela faltar.
- **DM2 — Resolvida pela Q6.** Step-up na carga. **Decisão:** sem TOTP, como a de medicamentos (`src/screens/MedicationCatalog.tsx:37-41`), com `ConfirmButton`. O teste da Task 2 do plano do api menciona TOTP: a Task 0 confere o SDL e avisa o coordenador se ele pedir `code`.
- **DM3 — Resolvida pela Q13.** O pré-requisito do `supply_programs` usa `clinical_record_disabled`, que já tem frase. Entram no R18 `supply_catalog.imported` e `maintenance.supply_catalog.imported`. A Task 1 traz o caso "programas sem prontuário", e a prova confere o evento.
