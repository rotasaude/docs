# Módulo 26b — APAC, OCI, teto em dinheiro e AIH (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-25-module-26b-apac-contracts.md` (fonte única), §1, §11 ("Maintenance") e Q13 (§12; docs `60908e1`). As divergências DM1 e DM2 deste plano foram resolvidas pela Q13 (seção final), e o plano já aplica a decisão.
> 2. A **`main` do maintenance com o 26c mergeado** (o contrato manda executar o 26b depois do 26c). Quando este plano foi escrito, o `origin/main` estava em `14406bd`, com o 35b e sem o 26c.
> 3. O **api do 26b** na branch `feat/mod-26b-apac` (worktree `apps/api/.claude/mod26b`, porta **3064**), com `care_regulation_apac` no `Platform::Features::CATALOG` e `requires: %w[feature:care_regulation sigtap_billing_data]`.
>
> A Task 1 não depende do api: as frases são tabelas, e o liga/desliga é o genérico. A prova da Task 2 depende do api rodando, com o banco de dev migrado e a SIGTAP completa carregada **com autorização do usuário**. Ordem: api → dashboard e **maintenance** → suíte completa do api → prova.

**Goal:** Lado maintenance do 26b, na aba Funcionalidades:
- o nome do interruptor `care_regulation_apac`;
- a frase do pré-requisito `sigtap_billing_data_missing` (e a de `care_regulation_disabled`, se o 26c não a trouxe).

Nada mais passa pelo maintenance: as faixas, o momento, a FPO (Q2: "o maintenance não existe em produção") e os arquivos são da cidade, no dashboard; a carga da SIGTAP é do api.

**Architecture:** Conferido no código (`origin/main` em `14406bd`):
- O liga/desliga é genérico: `FeaturesTab.tsx`, `setCityFeature` sem TOTP, três estados por `featureStatus` (`src/lib/features.ts`). A chave nova aparece sozinha, com a `description` do api; o maintenance só dá o nome em português.
- `src/lib/features.ts`: uma entrada em `FEATURE_LABELS` e uma (ou duas) em `MISSING_LABELS`. `care_regulation_disabled` **não** existe em `14406bd`; o plano do 26c o acrescenta ("regulação municipal (care_regulation) desligada ou incompleta"). A Task 0 confere.
- **Sem GraphQL novo:** a lista vem do `city.features` que já existe. O `schema.graphql` não muda, e o `codegen:check` fica limpo sem gerar nada.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-25-module-26b-apac-oci-aih-design.md` ("Afeta: apps/maintenance: interruptores"; decisão 14). **Contrato:** `docs/superpowers/plans/2026-11-25-module-26b-apac-contracts.md` §1 e §11. **Molde:** `docs/superpowers/plans/2026-11-25-module-26c-mira-maintenance.md` (Tasks 0–2); no código, o bloco `describe("módulo 35b")` de `src/lib/features.test.ts` e o `describe("FeaturesTab — centro de zoonoses (módulo 35b)")` de `src/screens/FeaturesTab.test.tsx`.

## Global Constraints

- **Interruptor** (contrato §1): `care_regulation_apac`, com `requires: feature:care_regulation sigtap_billing_data`. `sigtap_billing_data` = existe release SIGTAP ativa com financiamento carregado; sem ela, o api manda `sigtap_billing_data_missing`.
- **Nome**, letra por letra: `FEATURE_LABELS.care_regulation_apac` = "Regulação por laudo (APAC, OCI e AIH)" (contrato §1). Os de hoje não mudam (`care_regulation` "Regulação municipal").
- **Frases de pré-requisito** (`MISSING_LABELS`). Os códigos são do contrato. A de `care_regulation_disabled` vem do 26c, ou é acrescentada igual (Q13); a de `sigtap_billing_data_missing` é **proposta**, no molde das que existem (o nome da chave entre parênteses):

  | Código | Frase |
  |---|---|
  | `sigtap_billing_data_missing` | "SIGTAP da competência incompleta: sem o financiamento e os valores (a plataforma carrega a tabela completa do DATASUS)" |
  | `care_regulation_disabled` (só se a Task 0 não a achar) | "regulação municipal (care_regulation) desligada ou incompleta" (a mesma do plano do 26c, letra por letra; Q13) |

  As que já existem não mudam.
- **Sem cascata:** ligar ou desligar manda só aquela `key`. Desligar `care_regulation` não desliga `care_regulation_apac`: ele fica "ligada, falta pré-requisito", com a frase de `care_regulation_disabled`.
- **O que passa aqui:** só o estado dos interruptores. Nenhum laudo, faixa, FPO, número, arquivo ou dado de paciente passa pelo maintenance.
- **Frases da tela:** código desconhecido aparece cru, nunca `undefined`/`null`.
- **Interface:** nada de componente novo; cores só por papéis do tema, com `src/theme/*.test.ts` na suíte; 375 px sem rolagem lateral da página.
- **Testes:** só `bin/test maintenance -w maintenance-mod26b [arquivos]` (de `$R`), nunca `vitest` direto. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge e push só com autorização explícita.
- **Board #1:** o maintenance é superfície do card do interruptor do 26b (criado pelo plano do api). O card vai a In Progress pelo plano do api; a Task 0 só confere.
- **Execução coordenada:** ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod26b`, `A=$R/apps/api/.claude/mod26b`.
- Worktree **fora de `apps/`**: `$M`, na branch `feat/mod-26b-apac` a partir de `origin/main`. Os caminhos das tasks são relativos a `$M`.
- **Prova:**
  - Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3064**;
  - abra `http://maintenance.localhost:5177`, nunca `localhost:5177`;
  - conta `dev@local` / `dev-password`, com TOTP fixo (`lib/dev_maintainer.rb`); o usuário faz login e TOTP.

## Review Focus

1. **Ligar `care_regulation_apac` antes da carga completa da SIGTAP.** Espera-se "ligada, falta pré-requisito", com a frase de `sigtap_billing_data_missing`, e nada de "ligada e utilizável". Teste: Task 1, "ligar sem a SIGTAP completa: a frase do financiamento".
2. **Desligar `care_regulation` com o 26b ligado.** Espera-se que só `care_regulation` vá na mutation e que, depois do refetch, `care_regulation_apac` fique "ligada, falta pré-requisito", com a frase de `care_regulation_disabled`, sem nada falar em apagar. Teste: Task 1, "desligar a regulação deixa o 26b ligado, com o motivo".
3. **Os dois pré-requisitos faltando juntos.** Espera-se as duas frases unidas por "; ", na ordem do api. Teste: Task 1, "sem regulação e sem SIGTAP: as duas frases".
4. **Api sem o 26b.** Espera-se nenhuma linha `care_regulation_apac` e as linhas de hoje iguais. Teste: Task 1, "api sem o 26b: nenhuma chave nova".
5. **Api mais novo que a tela** (um `missing` futuro na linha do 26b, como `sigtap_compatibility_missing`). Espera-se o código cru. Teste: Task 1, "missing desconhecido do 26b aparece cru".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx` | nome e frases de pré-requisito | 1 |
| `README.md` | parágrafo do módulo; suíte, codegen:check, build, revisão, prova na 5177, parada | 2 |

**Estratégia de teste:**
- **Funções puras** (`featureLabel`, `missingText`, `missingSummary`), por tabela de casos.
- **A aba**, pelo fluxo real, com o `fetch` falso que responde por nome de operação: os helpers do topo de `FeaturesTab.test.tsx` (`reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`) e o `mockWithMutation` do bloco do 35a. O teste confere as variáveis exatas da mutation, o estado depois do refetch e a frase de cada motivo.
- **A Task 2:** a suíte inteira com as guardas de tema, o `codegen:check`, o build e a prova contra o api do 26b em Curitiba.

---

### Task 0: Card, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (com o 26c); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); `$A`.
- Produces: `$M`; a base anotada; a `description` real de `care_regulation_apac` e o `missing` real; se `care_regulation_disabled` já existe.

- [ ] **Step 1: Conferir o card do 26b.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-26\\.")) | {id, title, status}'
```

Expected: o card do interruptor do 26b em In Progress ou além. Em Todo, mova com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod26b
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'rnds_regulation\|care_regulation_disabled\|care_regulation_apac\|sigtap_billing_data\|  care_regulation: "Regulação municipal"'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Envio da regulação à RNDS (módulo 26c).\*\*\|^\*\*Centro de zoonoses (módulo 35b).\*\*'
grep -n 'key: "care_regulation_apac"' -A4 $A/app/services/platform/features.rb
grep -n 'sigtap_billing_data' $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: `rnds_regulation` presente (o 26c na `main`). Se ausente, **pare**: o contrato manda começar depois dele;
- `care_regulation_disabled`: anote se existe (a Task 1 só o acrescenta se faltar); **nenhuma** linha com `care_regulation_apac` nem `sigtap_billing_data`;
- no api: `care_regulation_apac` com os `requires` do §1 e o motivo `sigtap_billing_data_missing`.

**Anote** a `description` real de `care_regulation_apac`. **Onde o api diz outra coisa, vale o api**: a Task 1 ajusta as constantes dos testes, e o relato ao coordenador cita a diferença. Sem a branch do api ainda, siga pelo contrato e repita este passo antes da prova.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod26b -b feat/mod-26b-apac origin/main
cd $R/.worktrees/maintenance-mod26b && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod26b
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos).

**Relato ao coordenador:** "maintenance-mod26b Task 0: worktree e base verdes (N arquivos/M casos); care_regulation_apac e sigtap_billing_data_missing no api <conferidos|ausentes>; care_regulation_disabled <já existe|entra na Task 1>; diferenças: <…>."

---

### Task 1: Nome do interruptor e frases de pré-requisito

**Files:**
- Modify: `src/lib/features.ts`: uma entrada em `FEATURE_LABELS`, depois da última; em `MISSING_LABELS`, `sigtap_billing_data_missing` (e `care_regulation_disabled`, se a Task 0 não o achou), junto das do 26a/26c.
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary`, `featureStatus`, `doneText` (`features.ts`); os helpers do topo de `FeaturesTab.test.tsx` e o `mockWithMutation` do bloco do 35a.
- Produces: o rótulo e as frases das Global Constraints. Nenhuma função nova.

- [ ] **Step 1: Write the failing tests**

  No fim de `src/lib/features.test.ts`, um `describe("módulo 26b")` com:
  - `featureLabel("care_regulation_apac")` = "Regulação por laudo (APAC, OCI e AIH)";
  - `missingText("sigtap_billing_data_missing")` com a frase;
  - `missingText("care_regulation_disabled")` com a frase (a do 26c, ou a desta task);
  - `missingSummary([ "care_regulation_disabled", "sigtap_billing_data_missing" ])` = as duas frases unidas por "; ";
  - um código futuro (`sigtap_compatibility_missing`), que volta cru;
  - `featureLabel("care_regulation")` = "Regulação municipal", sem mudança.

  No fim de `src/screens/FeaturesTab.test.tsx`, um `describe("FeaturesTab — regulação por laudo (módulo 26b)")`, com o `mockWithMutation(initial, after, requested)` e as constantes (a `description` anotada na Task 0):
  - `RG_ON` e `RG_OFF` (`care_regulation`);
  - `APAC_OFF`, `APAC_ON` e `APAC_NO_SIGTAP` (`missing: [ "sigtap_billing_data_missing" ]`), da chave `care_regulation_apac`.

| Caso | Lista inicial → depois da mutation | Clique | Asserções |
|---|---|---|---|
| "aparece com o nome em português e a descrição do api" | `[RG_ON, APAC_OFF]` | — | "care_regulation_apac", "Regulação por laudo (APAC, OCI e AIH)" e a descrição; corpo sem `/undefined\|null/` |
| "ligar sem a SIGTAP completa: a frase do financiamento" | `[RG_ON, APAC_OFF]` → `[RG_ON, APAC_NO_SIGTAP]` | "ligar" da linha, "confirmar: ligar care_regulation_apac" | variáveis `{ citySlug: "sp", key: "care_regulation_apac", enabled: true }`; "ligada, falta pré-requisito"; a frase de `sigtap_billing_data_missing`; nada de "ligada e utilizável" |
| "desligar a regulação deixa o 26b ligado, com o motivo" | `[RG_ON, APAC_ON]` → `[{ ...RG_OFF, changedAt, changedBy }, { ...APAC_ON, usable: false, missing: [ "care_regulation_disabled" ] }]` | "desligar" da linha `care_regulation`, "confirmar: desligar care_regulation" | 1 mutation, só com `care_regulation`; `care_regulation_apac` em "ligada, falta pré-requisito", com a frase de `care_regulation_disabled`; nada fala em apagar (`/apaga\|exclu/i`) |
| "sem regulação e sem SIGTAP: as duas frases" | `[RG_OFF, { ...APAC_ON, usable: false, missing: [ "care_regulation_disabled", "sigtap_billing_data_missing" ] }]` | — | as duas frases unidas por "; ", nessa ordem |
| "api sem o 26b: nenhuma chave nova" | `[RG_ON]` | — | `queryByText("care_regulation_apac")` nulo; a linha `care_regulation` igual |
| "missing desconhecido do 26b aparece cru" | `[{ ...APAC_ON, usable: false, missing: [ "sigtap_compatibility_missing" ] }]` | — | "sigtap_compatibility_missing" na linha |

- [ ] **Step 2: Run tests to verify they fail** — `bin/test maintenance -w maintenance-mod26b src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos com o rótulo e a frase nova; os casos antigos passam.
- [ ] **Step 3: Implement.** As entradas levam um comentário de uma linha: "Módulo 26b: care_regulation_apac exige care_regulation e a SIGTAP com financiamento (sigtap_billing_data_missing); a numeração, a FPO e os arquivos são da cidade, no dashboard."
- [ ] **Step 4: Run tests to verify they pass** — o mesmo comando, e `cd $M && npx tsc --noEmit`. Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx
$G commit -m "feat: name the laudo regulation switch and its SIGTAP billing data prerequisite" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod26b Task 1: rótulo e frases; FeaturesTab verde; diferenças do api: <…>."

---

### Task 2: README, suíte, build, revisão e prova no navegador

**Files:** `README.md`; `$R/.claude/launch.json` (entrada nova; fora de git).

- [ ] **Step 1: README.** Um parágrafo novo, **Regulação por laudo (módulo 26b).**, depois do último parágrafo de módulo (o do 26c, se for o último). Ele diz:
  - o interruptor e o que ele exige, com as frases da aba;
  - que, sem a carga completa da SIGTAP (financiamento, valores SH e SP, compatibilidades), ele fica "ligada, falta pré-requisito", e quem carrega é o api;
  - que as faixas de numeração, o momento da autorização, a FPO e os arquivos (remessa APAC e fila nominal da OCI) são da cidade, no dashboard, e nada disso passa aqui;
  - que desligar não apaga autorização, número nem reserva: o pedido por laudo só deixa de ser aceito;
  - a ordem de deploy: api, depois dashboard e maintenance; contra um api sem o 26b, a chave só não aparece.

  Commit:

```bash
cd $M && $G add README.md
$G commit -m "docs: describe the laudo regulation switch in the maintenance README" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 2: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod26b && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base e **nenhum** arquivo de teste novo; casos novos em `features.test.ts` e `FeaturesTab.test.tsx`;
- `codegen:check` limpo, e `schema.graphql` fora do diff;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 3: Varreduras**

```bash
cd $M
$G diff --stat origin/main..HEAD -- schema.graphql src/gql | grep -c . | grep -qx 0 && echo ok-sdl || echo "ERRO: SDL mudou sem motivo"
grep -rnE "laudo|fpo|number_range|apac_export" src/lib src/screens && echo "ERRO: dado da cidade no maintenance" || echo ok-escopo
grep -n '  care_regulation_apac:' src/lib/features.ts | grep -c . | grep -qx 1 && echo ok-rotulo || echo "revisar: care_regulation_apac fora de FEATURE_LABELS"
```

Expected: `ok-sdl`, `ok-escopo`, `ok-rotulo`.

- [ ] **Step 4: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec e o contrato (§1, §11). Ele confere:
  - o rótulo letra por letra;
  - as frases e nenhuma frase antiga mudada;
  - nenhuma cascata no liga/desliga;
  - nada de laudo, faixa, FPO ou dado de paciente;
  - um README sem promessa de apagar autorização ou reserva.

- [ ] **Step 5: Prova no navegador na 5177 (com o usuário).** Depende do api do 26b na **3064**, com o banco de dev migrado e a SIGTAP completa carregada, **com autorização do usuário** (plano do api). Acrescente ao `$R/.claude/launch.json`, depois da última entrada `maintenance-mod*`:

```json
{
  "name": "maintenance-mod26b",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/maintenance-mod26b && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3064 VITE_MAINTENANCE_ENV=development npx vite --port 5177 --host 0.0.0.0" ],
  "port": 5177
}
```

  Pare o serviço do compose (`docker compose -f $R/docker-compose.yml stop maintenance`) e suba com `preview_start` `maintenance-mod26b`. Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP. Antes de tudo, **anote** o estado de `care_regulation` e `care_regulation_apac` em Curitiba e Maringá. As screenshots vão em `evidencias/2026-11-25-modulo-26b/maintenance/`. Roteiro:
  1. **Funcionalidades (Curitiba):** `care_regulation_apac` aparece com nome e descrição; antes da carga da SIGTAP (se a sessão do api ainda não carregou), ligar → "ligada, falta pré-requisito", com a frase do financiamento; depois da carga, ao recarregar a aba, "ligada e utilizável".
  2. **Sem cascata:** desligar `care_regulation` → `care_regulation_apac` fica "ligada, falta pré-requisito", com a frase de `care_regulation_disabled`. Religue.
  3. **Combinado com a sessão do dashboard (5225):** com o 26b utilizável, o grupo Regulação do dashboard mostra as telas novas; desligado, some.
  4. **A 375 px:** a aba não tem rolagem lateral da página.

  Depois, volte as chaves ao estado anotado e limpe: `preview_stop` do `maintenance-mod26b` e `docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 6: Pare.** O merge só vem depois do api do 26b (a suíte completa do api é da sessão "Testes", pedida pelo plano do api) e só com autorização explícita, uma etapa de cada vez: merge, push e card.
  - Antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas).
  - Quando autorizado: `git fetch`, `merge --ff-only feat/mod-26b-apac`, `push origin main`, `worktree remove $M`, `branch -d`.
  - O card vai a **Done** só com as partes do api e do dashboard também entregues, e a **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod26b Task 2: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-25-modulo-26b/maintenance/); aguardando autorização de merge."

---

## Self-review

- **Cobertura** (contrato §1 e §11, "Maintenance"):
  - `FEATURE_LABELS.care_regulation_apac`: Task 1;
  - `MISSING_LABELS.sigtap_billing_data_missing` e, se faltar, `care_regulation_disabled`: Task 1;
  - a documentação e a prova: Task 2.
- **Placeholders:** nenhum. Valores lidos na execução: os ids dos cards, a `description` real e a presença de `care_regulation_disabled` (Task 0).
- **Tipos e nomes:** existem em `14406bd`: `featureLabel`, `missingText`, `missingSummary`, `featureStatus`, `doneText` e os helpers de teste do `FeaturesTab.test.tsx` (`mockWithMutation` no bloco do 35a).
- **Proporção:** a mudança de produção é uma entrada de rótulo e uma ou duas frases. O plano traz os testes em tabela.

## Divergências propostas ao contrato

As duas foram decididas pela Q13 do contrato (docs `60908e1`), e o plano já aplica a decisão. O texto de cada uma fica como registro.

- **DM1 — Resolvida pela Q13. `care_regulation_disabled` depende do 26c.** O `requires` do §1 gera `care_regulation_disabled` (a gramática `feature:X` → `X_disabled` do api, a mesma do 26c), mas a frase não existe em `14406bd` (`apps/maintenance/src/lib/features.ts:24-84`, `MISSING_LABELS`), e só o plano do 26c a acrescenta. **Decisão:** a frase vem do 26c, ou é acrescentada igual. A Task 0 confere; se o 26c não a trouxe, a Task 1 a acrescenta com a frase do plano do 26c, letra por letra.
- **DM2 — Resolvida pela Q13. Pré-requisito do pré-requisito.** `care_regulation` exige `clinical_record`; com o prontuário desligado, o api não propaga a falta e manda só `care_regulation_disabled` (o mesmo caso de `territory_registry_disabled`, `features.ts:78`, "desligado ou incompleto"). **Decisão:** a frase diz "desligada ou incompleta". **Ressalva para o coordenador:** a Q13 escreve que é a frase do `sigtap_billing_data_missing` que diz "desligada ou incompleta", mas a DM2 tratava do `care_regulation_disabled` (a SIGTAP não se desliga). O plano lê assim: `care_regulation_disabled` fica "desligada ou incompleta" (a frase do 26c), e a do `sigtap_billing_data_missing` diz "incompleta". Se a intenção era outra, só muda a constante da Task 1.

**Sem divergência, para registro:**
- O SDL não muda.
- A carga da SIGTAP não tem tela no maintenance (o api carrega; contrato §2).
