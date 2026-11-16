# Módulo 35a — Vigilância ambiental de endemias (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 35a** na branch `feat/mod-35a-vector-control` (worktree `apps/api/.claude/mod35a`, porta **3059**) com a entrada nova no fim de `Platform::Features::CATALOG` (contrato §1): `vector_control`, `requires: %w[feature:home_visits]`, sem `environments`, em `CITIZEN_HIDDEN`. Ordem de entrega (contrato §19): contracts → api → **dashboard e maintenance** → campo. As Tasks 0–1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 depende do api do 35a com o banco de dev migrado e semeado (`lib/vector_control_crew.rb`, contrato §18).

**Goal:** Lado maintenance de F-35.1: o interruptor `vector_control` aparece na aba Funcionalidades com o nome "Vigilância ambiental (endemias)"; o pré-requisito `home_visits_disabled` sai com frase própria; o liga/desliga genérico fica fixado em teste para a chave nova (inclusive desligar a visita com a vigilância ligada, e o cadastro territorial incompleto que o api não propaga). Nenhuma mutation, tela ou SDL novo (contrato §16).

**Architecture:** Conferido no código (`origin/main` do maintenance, `fea9c29`; api `469e2e68`). O liga/desliga é genérico: `setCityFeature(citySlug, key, enabled)` sem TOTP, três estados (`featureStatus`) e `missing` vindo do api. Em `ro-api/app/services/platform/features.rb:354-356`, `missing_for` gera `"#{dependency}_disabled"` para cada `feature:<chave>` cuja dependência não está **ligada e utilizável**: o api do 35a devolve `home_visits_disabled` tanto com a visita desligada quanto com ela ligada e o cadastro territorial desligado ou incompleto (o api não propaga o pré-requisito do pré-requisito, como no 24b e no 32b). Esse código não existe em `MISSING_LABELS` (`src/lib/features.ts:24-82`) e sairia cru. `FeaturesTab.tsx` só renderiza o que o api manda (a única regra por chave é o painel de `digital_signature`, `FeaturesTab.tsx:240`); nenhum aviso de simulado pega a chave nova. O SDL não muda: `CityFeature` já carrega tudo.

**Papéis e tipo de unidade: nada a fazer** (conferido, pedido do coordenador). `endemic_coordinator`, `vector_lab` e `environmental_surveillance` são da cidade e são administrados no dashboard:
- o maintenance não tem mutation de papel nem de unidade (as 18 mutations do `schema.graphql` em `fea9c29`: protocolos, mantenedores, tokens, `setCityFeature`, PEC, linhas de cuidado, laboratório, lista de doenças, mapa de campo);
- o único lugar com papel é a aba Contas (`CityDetail.tsx:276`), que mostra `row.roles.join(", ")` cru, como faz com todos os papéis de todos os módulos (`CityAccount.roles: [String!]!`); não há mapa de rótulo de papel no app;
- não há tipo de unidade em lugar nenhum de `src/` nem do SDL (o único `*_unit_*` é o código `caps_unit_missing`, testado cru em `features.test.ts:313`);
- o estoque (`Stock::Domain`, locais, lotes) não passa pelo maintenance; `MedicationCatalog` é o CATMAT da plataforma, e o produto do 35a (`vector_products`) é da cidade.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (Afeta "`apps/maintenance`", decisão 1, F-35.1); `docs/adr/0054.md`; o contrato `docs/superpowers/plans/2026-11-16-module-35a-vector-control-contracts.md` §1 (interruptor), §16 (maintenance), §19 (ordem), §20 Q1. Molde: `docs/superpowers/plans/2026-11-12-module-24b-home-visit-maintenance.md` e o commit `55b680b` do maintenance. As divergências MV-1 e MV-2 foram decididas na **Q12** do contrato (docs `aa4f3c5`): a frase de `home_visits_disabled` e a descrição nova do `CATALOG`, ambas aplicadas aqui.

## Global Constraints

- Interruptor (contrato §1, Q1): `vector_control` exige `feature:home_visits` ligado e utilizável. `missing` = `home_visits_disabled`. Descrição do api (Q12): "Boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio de transmissão" (a Task 0 confere o texto exato).
- Nome (contrato §1 e §16): `vector_control` → "Vigilância ambiental (endemias)".
- Frase do pré-requisito (Q12): `home_visits_disabled` → "visita domiciliar e territorial (home_visits) desligada ou incompleta".
- Ligar ou desligar manda só aquela `key`. A aba nunca liga a visita junto com a vigilância nem desliga a vigilância quando a visita é desligada (sem cascata).
- Desligar não apaga nada (§1): boletins, quarteirões, perfis, ovitrampas, LIRAa, bloqueios e amostras ficam; as rotas respondem `403 feature_disabled`, o app de campo recebe `rejected feature_disabled` e o job de bloqueio não age. A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade: nenhum quarteirão, imóvel, caso, endereço, ovitrampa, amostra, produto ou papel passa por aqui.
- `vector_control` nunca chega ao cidadão (`CITIZEN_HIDDEN`); existe em todos os ambientes, sem simulado.
- Valor desconhecido (api mais novo) aparece cru, nunca `undefined`/`null`.
- Interface: nenhuma tela nova; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte.
- Âncoras em `origin/main` na execução: em `FEATURE_LABELS`, a última entrada (hoje `collective_activities`, `features.ts:153`); em `MISSING_LABELS`, logo antes do comentário de `city_unreachable` (hoje depois de `territory_registry_disabled`, `features.ts:78`); no README, depois do parágrafo **Ouvidoria do SUS (módulo 33).** (`README.md:402-417`), antes de **Mapas de campo (módulo 24c).** Conflito no rebase com outro módulo é só ordem de linhas.
- Testes: só `bin/test maintenance -w maintenance-mod35a [arquivos]` (de `$R`); nunca `vitest` direto. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Git: `/opt/homebrew/bin/git`; nunca `git add -A` nem `git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge e push só com autorização explícita do usuário.
- Board #1: o maintenance é superfície de F-35.1, com o api e o dashboard (spec, tabela de F-IDs). O card vai a In Progress pelo plano do api; a Task 0 só confere.
- Execução coordenada: ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod35a`, `A=$R/apps/api/.claude/mod35a`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-35a-vector-control` a partir de `origin/main`. Caminhos das tasks relativos a `$M`.
- Prova: Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3059**. Abra `http://maintenance.localhost:5177`, nunca `localhost:5177` (o 403 de origem vira a mensagem genérica "não foi possível entrar agora"). Conta `dev@local` / `dev-password`, mantenedor semeado pelo `db:seed` com TOTP fixo (`lib/dev_maintainer.rb`); o usuário faz login e TOTP. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Desligar a visita com a vigilância ligada.** Só `home_visits` vai na mutation. Depois do refetch, a linha de `vector_control` diz "ligada, falta pré-requisito" com a frase da visita e não é desligada junto. Teste: Task 1, "desligar home_visits com vector_control ligada: só a chave pedida; a vigilância fica faltando a visita".
2. **Cadastro territorial incompleto com a visita ligada.** O api manda só `home_visits_disabled` para a vigilância; a aba não inventa a frase do cadastro na linha dela nem diz que a visita está "desligada" sem o "ou incompleta". Teste: Task 1, "visita ligada sem o cadastro: a vigilância mostra só a frase da visita".
3. **Nome igual (ou quase) à descrição do api.** A aba mostra os dois lado a lado; se a `description` real for igual ao nome, `getByText` acha dois nós. Teste: Task 1, "aparece com o nome em português e a descrição do api" (com a `description` da Q12, conferida na Task 0).
4. **Texto que prometa apagar ao desligar** (boletins, ovitrampas, bloqueios). Teste: Task 1, os casos da aba com `not.toMatch(/apaga|exclu/i)`; Task 2, grep no README.
5. **Api mais novo que a tela** (um `missing` futuro, como `stock_location_missing`). Aparece cru, e nenhum aviso de simulado aparece. Testes: Task 1, "módulo 35a: nome, frase da visita; missing desconhecido cru" e "vigilância ligada: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, entrada do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome da chave, frase de `home_visits_disabled`, fluxo genérico fixado, parágrafo do módulo | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177, parada | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) por tabela de casos; a aba pelo fluxo existente, com o `fetch` falso respondendo por nome de operação (helpers `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e o tipo `Feature`, no topo de `FeaturesTab.test.tsx:15-50`), conferindo as variáveis exatas da mutation e o refetch que mostra o dependente faltando. A Task 2 roda a suíte inteira com as guardas de tema, o `codegen:check` (prova de que o SDL não mudou), o build e a prova contra o api do 35a em Curitiba (`record`, com a visita) e Maringá (`integrated`, sem a visita utilizável).

---

### Task 0: Card, âncoras, entrada do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`fea9c29` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: `$M`; a base anotada (arquivos/casos); a `description` de `vector_control` conferida igual à da Q12.

- [ ] **Step 1: Conferir o card F-35.1** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 600 --format json --jq '.items[] | select(.title | test("^F-35\\.1 ")) | {id, title, status}'
```

Expected: In Progress ou além (movido pelo plano do api). Em Not Started/Todo, mova com `gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'` e `gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entrada do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod35a
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'territory_registry_disabled:\|city_unreachable:\|collective_activities:\|vector_control\|home_visits_disabled'
grep -n 'key: "vector_control"' -A3 $A/app/services/platform/features.rb
grep -c 'description: "Boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio de transmissão"' $A/app/services/platform/features.rb
grep -n 'CITIZEN_HIDDEN =' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`, nenhuma linha com `vector_control` nem `home_visits_disabled`; a última entrada de `FEATURE_LABELS` é `collective_activities` (ou a de um módulo que entrou depois; use-a como âncora); `territory_registry_disabled` logo antes do comentário de `city_unreachable`;
- no api, `vector_control` no fim do `CATALOG`, com `requires: %w[feature:home_visits]`, sem `environments` e com a `description` exata da Q12 (o `grep -c` devolve `1`); `CITIZEN_HIDDEN` com `vector_control`.

Outro `requires`, um ambiente restrito, uma chave a mais ou outra `description` (por exemplo, a antiga "Vigilância ambiental de endemias: …"): **pare e reporte** ao coordenador, porque o api tem de bater com a Q12. Se o api ainda não tiver a entrada, a Task 1 segue com a `description` da Q12 e a Task 2 espera.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod35a -b feat/mod-35a-vector-control origin/main
cd $R/.worktrees/maintenance-mod35a && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod35a
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos). A Task 2 espera o **mesmo número de arquivos de teste**, com casos novos só em `features.test.ts` e `FeaturesTab.test.tsx`.

**Relato ao coordenador:** "maintenance-mod35a Task 0: worktree e base verdes (N arquivos/M casos); entrada `vector_control` do api conferida (ou: ausente, Task 2 espera); F-35.1 em In Progress."

---

### Task 1: Nome e pré-requisito de `vector_control`

**Files:**
- Modify: `src/lib/features.ts` (uma entrada em `MISSING_LABELS`, depois de `territory_registry_disabled`; uma em `FEATURE_LABELS`, depois de `collective_activities`; cada uma com o comentário de uma linha no molde do 24b: "Módulo 35a (ADR 0054): …"), `README.md` (parágrafo depois do de Ouvidoria do SUS, módulo 33)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("vector_control") === "Vigilância ambiental (endemias)"`; `missingText("home_visits_disabled") === "visita domiciliar e territorial (home_visits) desligada ou incompleta"`.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 35a", () => {
  it("módulo 35a: nome, frase da visita; missing desconhecido cru", () => {
    expect(featureLabel("vector_control")).toBe("Vigilância ambiental (endemias)");
    expect(missingText("home_visits_disabled")).toBe("visita domiciliar e territorial (home_visits) desligada ou incompleta");
    expect(missingText("stock_location_missing")).toBe("stock_location_missing");
    expect(missingSummary([ "home_visits_disabled" ])).toBe("visita domiciliar e territorial (home_visits) desligada ou incompleta");
  });

  it("os nomes do 24b não mudam", () => {
    expect(featureLabel("home_visits")).toBe("Visita domiciliar e territorial");
    expect(missingText("territory_registry_disabled")).toBe("cadastro individual e domiciliar (territory_registry) desligado ou incompleto");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx`, um `describe("FeaturesTab — vigilância ambiental de endemias (módulo 35a)")` com o `beforeEach`/`afterEach` do bloco do 24b, as constantes abaixo e seis casos. `HV_ON` é o do bloco do 24b (descrição real do api em `469e2e68`); `VC_OFF.description` é a da Q12, conferida na Task 0.

```tsx
const HOME_VISITS_PHRASE = "visita domiciliar e territorial (home_visits) desligada ou incompleta";
const HV_ON: Feature = { key: "home_visits", description: "Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo",
  enabled: true, usable: true, missing: [], changedAt: "2026-11-16T12:00:00Z", changedBy: "dev@local" };
const VC_OFF: Feature = { key: "vector_control",
  description: "Boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio de transmissão",  // Q12
  enabled: false, usable: false, missing: [], changedAt: null, changedBy: null };
const VC_ON: Feature = { ...VC_OFF, enabled: true, usable: true, changedAt: "2026-11-16T13:00:00Z", changedBy: "dev@local" };
const VC_BLOCKED: Feature = { ...VC_ON, usable: false, missing: [ "home_visits_disabled" ] };
```

| Caso | Lista inicial → depois da mutation | Clique | Asserções |
|---|---|---|---|
| "aparece com o nome em português e a descrição do api" | `[VC_OFF]` | — | `findByText("vector_control")`; `getByText("Vigilância ambiental (endemias)")`; `getByText(VC_OFF.description)`; `getAllByText("desligada")` com 1; corpo sem `/undefined\|null/` |
| "com a visita utilizável: liga utilizável" | `[HV_ON, VC_OFF]` → `[HV_ON, VC_ON]` | "ligar", "confirmar: ligar vector_control" | status = "vector_control ligada e utilizável."; variáveis `{ citySlug: "sp", key: "vector_control", enabled: true }`; sem "ligada, falta pré-requisito" |
| "sem a visita: ligada, falta a visita" | `[VC_OFF]` → `[VC_BLOCKED]` | idem | 1 mutation com as mesmas variáveis; status = `` `vector_control ligada, mas ainda não utilizável — falta: ${HOME_VISITS_PHRASE}.` ``; "ligada, falta pré-requisito" presente; "ligada e utilizável" ausente; corpo sem `/undefined\|apaga\|exclu/i` |
| "desligar home_visits com vector_control ligada: só a chave pedida; a vigilância fica faltando a visita" | `[HV_ON, VC_ON]` → `[{ ...HV_ON, enabled: false, usable: false, changedAt: "2026-11-16T15:00:00Z" }, VC_BLOCKED]` | primeiro "desligar", "confirmar: desligar home_visits" | `waitFor` o `HOME_VISITS_PHRASE` na tabela; 1 mutation, variáveis `{ citySlug: "sp", key: "home_visits", enabled: false }`; status = "home_visits desligada."; "ligada, falta pré-requisito" presente; "ligada e utilizável" ausente; corpo sem `/apaga\|exclu/i` |
| "visita ligada sem o cadastro: a vigilância mostra só a frase da visita" | `[{ ...HV_ON, usable: false, missing: [ "territory_registry_disabled" ] }, VC_BLOCKED]` | — | `getAllByText("ligada, falta pré-requisito")` com 2; `getByText(HOME_VISITS_PHRASE)` (uma vez, na linha da vigilância); `getByText("cadastro individual e domiciliar (territory_registry) desligado ou incompleto")` (uma vez, na linha da visita); nenhuma mutation |
| "vigilância ligada: nenhum aviso de simulado" | `[HV_ON, VC_ON]` | — | `getAllByText("ligada e utilizável")` com 2; `queryAllByRole("note")` com 0; corpo sem `/SIMULADO\|undefined\|null/` |

Os casos com mutation usam o `fetchMock.mockImplementation` com `let listed` do bloco do 24b (`mutation SetCityFeature` troca `listed` e responde `{ ok: true, errors: [], feature: <a chave pedida, no estado novo> }`; o resto responde `featuresReply(listed, { recordMode: "record" })`).

- [ ] **Step 2: Run tests to verify they fail**

Run (de `$R`): `bin/test maintenance -w maintenance-mod35a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL nos casos novos (`featureLabel("vector_control")` = `null`; a frase sai crua `home_visits_disabled`); os casos antigos passam.

- [ ] **Step 3: Implement**

Duas entradas em `src/lib/features.ts`, com os valores do Produces e o comentário: no `MISSING_LABELS`, "vector_control exige home_visits ligada e utilizável; com o cadastro territorial desligado ou incompleto a visita fica ligada mas inutilizável, e o api manda só este código"; no `FEATURE_LABELS`, "vector_control exige home_visits (missing: home_visits_disabled, acima). Todos os ambientes, sem simulado; nunca chega ao cidadão".

README, parágrafo **Vigilância ambiental de endemias (módulo 35a).** depois do de Ouvidoria do SUS (módulo 33), com:
- o interruptor `vector_control` ("Vigilância ambiental (endemias)") liga, no dashboard, o grupo "Endemias" (quarteirões e risco do imóvel, ciclos e cadência, ovitrampas, LIRAa, bloqueio de transmissão, amostras, tarefas, produtos e parâmetros, relatórios), o boletim do agente de endemias (CBO 515140) na visita do 24b, e o boletim, a ovitrampa, o LIRAa e o bloqueio no app de campo;
- exige `home_visits` ligada e utilizável; sem ela, ou com ela ligada sem o cadastro territorial, a linha diz "visita domiciliar e territorial (home_visits) desligada ou incompleta"; a aba não liga nem desliga uma pela outra;
- existe em todos os ambientes, sem simulado, e nunca chega ao cidadão;
- desligar esconde as telas e recusa boletim, ovitrampa, LIRAa e bloqueio novos (inclusive os que chegam do app); o que foi gravado fica, e o bloqueio automático a partir da notificação para;
- os papéis "Coordenação de endemias" e "Laboratório de entomologia", a unidade "Vigilância ambiental e zoonoses", o local de estoque e os produtos são da cidade, no dashboard; nenhum dado de cidadão nem endereço de caso passa por aqui;
- ordem de deploy: api → dashboard → maintenance; contra um api sem o 35a, a chave só não aparece.

- [ ] **Step 4: Run tests to verify they pass**

Run (de `$R`): `bin/test maintenance -w maintenance-mod35a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx && cd $M && npx tsc --noEmit`
Expected: PASS; `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the vector control switch and its home visit prerequisite" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod35a Task 1: `vector_control` e `home_visits_disabled` com nome e frase, 8 casos novos verdes, README; commit <sha>."

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod35a && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base e 8 casos a mais;
- `codegen:check` limpo (o SDL não mudou);
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 1, F-35.1) e o contrato §1, §16 e Q1. Ele confere:
  - só uma entrada nova em `FEATURE_LABELS` e uma em `MISSING_LABELS`;
  - nome diferente da descrição do api;
  - nenhum aviso de simulado; nenhum dado de cidade, papel ou unidade aqui;
  - que o README não promete apagar nada.

  Confira também: `grep -n -i "apaga\|exclu" $M/README.md | grep -i "boletim\|ovitrampa\|bloqueio\|endemia"`. Expected: sem saída.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 35a na **3059**, com o banco de dev migrado e semeado **com autorização do usuário** (a semente liga `vector_control` em Curitiba, contrato §18). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod35a --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod35a:/app -v maintenance-mod35a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3059 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod35a   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP; anote antes o estado de `home_visits` e `vector_control` nas duas cidades. Screenshots em `evidencias/2026-11-16-modulo-35a/maintenance/`. Roteiro:
1. Curitiba (`record`) → Funcionalidades: `vector_control` com o nome, a descrição e "ligada e utilizável" (a semente a ligou).
2. Contas de Curitiba: `endemias@curitiba.demo` com `endemic_coordinator` e `entomologia@curitiba.demo` com `vector_lab`, crus como os outros papéis.
3. Maringá (`integrated`, sem a visita utilizável): ligar `vector_control` → "ligada, falta pré-requisito" com a frase da visita; desligue de volta.
4. Curitiba, **só se o usuário autorizar** (as provas do dashboard e do campo dependem da chave ligada; combine com o coordenador): desligar `home_visits` → `vector_control` continua ligada e passa a "falta pré-requisito" com a frase da visita; religue a visita e a vigilância volta a utilizável sem novo clique. Sem autorização, pule.
5. **Auditoria** lista as tentativas de `setCityFeature` (só a chave e o estado).
6. A 375 px, sem rolagem lateral da página na aba Funcionalidades.

Depois, volte as chaves ao estado anotado e limpe: `docker rm -f maintenance-mod35a && docker volume rm maintenance-mod35a-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api do 35a (a suíte completa do api é da sessão "Testes", pedida pelo plano do api) e só com autorização explícita (merge, push e card, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter o commit desta entrega (sessões paralelas). Quando autorizado: `git fetch`, `merge --ff-only feat/mod-35a-vector-control`, `push origin main`, `worktree remove $M`, `branch -d`. O F-35.1 vai a **Done** só com as partes do api e do dashboard também entregues, e a **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod35a Task 2: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-16-modulo-35a/maintenance/); aguardando autorização de merge."

---

## Divergências propostas ao contrato

> **Decididas na Q12 do contrato (docs `aa4f3c5`):** MV-1 aceita (a frase abaixo); MV-2 aceita (a descrição nova no `CATALOG` do api). O plano já aplica as duas.

1. **MV-1 — Frase de `home_visits_disabled`.** O §16 fixa "precisa da visita domiciliar ligada". Dois problemas:
   - a aba compõe "… ligada, mas ainda não utilizável — falta: precisa da visita domiciliar ligada." (`doneText`, `features.ts:193-197`) e "O que falta: precisa da …" na tabela (`missingSummary`); todas as outras frases `<chave>_disabled` são "<nome do interruptor> (<chave>) desligada" (`features.ts:37-78`);
   - o api manda o mesmo código com a visita **ligada** e o cadastro territorial desligado ou incompleto (`missing_for` não propaga, `features.rb:354-356`), e aí "ligada" fica falso.

   Proposta, no molde do `territory_registry_disabled` do 24b (Q29): "visita domiciliar e territorial (home_visits) desligada ou incompleta". A Task 1 já usa essa frase. Se o coordenador mantiver a do §16, a troca é em três pontos: `features.ts`, `features.test.ts` e a constante `HOME_VISITS_PHRASE` em `FeaturesTab.test.tsx`.
2. **MV-2 — Descrição repete o nome.** A descrição do §1 começa por "Vigilância ambiental de endemias: …", e a aba mostra o nome "Vigilância ambiental (endemias)" logo ao lado. Não quebra teste (`getByText` casa o texto inteiro). Mas é o caso que a Q24 do 24b corrigiu para `home_visits` (a descrição passou a começar pelo conteúdo). Proposta, opcional: "Boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio de transmissão". Isso toca só o `CATALOG` do api e o spec que fixa a descrição; a Task 0 anota a descrição real de qualquer jeito.

**Sem divergência, para registro no §16:** papéis (`endemic_coordinator`, `vector_lab`) e tipo de unidade (`environmental_surveillance`) não pedem nada no maintenance (evidência em "Architecture"). A aba Contas mostra papéis crus para todos os módulos, então um rótulo só para os do 35a seria inconsistente. Se o coordenador quiser rótulos, é achado de interface, com card [CICLO-02] só se o usuário pedir.

## Self-review

- **Cobertura (Afeta "`apps/maintenance`", decisão 1, F-35.1; contrato §1, §16, §19, Q1):**
  - nome e frase do pré-requisito: Task 1 (Review Focus 3 e 5; Q12);
  - vigilância presa à visita e sem cascata: Task 1, quarto caso (Review Focus 1);
  - cadastro incompleto não propagado: Task 1, quinto caso (Review Focus 2);
  - desligar sem apagar: README da Task 1 e grep da Task 2 (Review Focus 4);
  - sem simulado, sem cidadão: Task 1, sexto caso, e README;
  - "nenhuma mutation nova": `codegen:check` na Task 2;
  - papéis e tipo de unidade: conferidos, nada a mudar (Architecture; prova da Task 2, passo 2);
  - suíte, build e prova (Curitiba, Maringá, 375 px): Task 2; card: Tasks 0 e 2.
- **Placeholders:** nenhum. Valores lidos na execução: o id do card; a `description` é a da Q12, conferida na Task 0, com parada se for outra.
- **Tipos e nomes:** existem em `origin/main` (`fea9c29`): `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature`. No api (`469e2e68`): `Platform::Features::CATALOG`, `missing_for`, `usable?`, `CITIZEN_HIDDEN`; a entrada `vector_control` vem do api do 35a.
- **Proporção:** a mudança de produção são duas linhas em `features.ts` e um parágrafo de README. O plano traz os testes numa tabela, não em código.
