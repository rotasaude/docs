# Módulo 35b — Centro de zoonoses (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 35b** na branch `feat/mod-35b-zoonoses` (worktree `apps/api/.claude/mod35b`, porta **3061**), com a entrada nova no fim de `Platform::Features::CATALOG`, depois de `vector_control` (contrato §1): `zoonoses`, `requires: %w[feature:territory_registry]`, sem `environments`, em `CITIZEN_HIDDEN`. Ordem (contrato §18): contracts → api → **dashboard, wpda e maintenance** → campo. A Task 1 não depende do api rodando (testes com `fetch` falso); a prova da Task 2 depende do api do 35b com o banco de dev migrado e semeado (`lib/zoonoses_crew.rb`, contrato §17).

**Goal:** Lado maintenance de F-35.11: o interruptor `zoonoses` aparece na aba Funcionalidades com o nome "Centro de zoonoses"; o pré-requisito `territory_registry_disabled` sai com a frase que já existe; o liga/desliga genérico fica fixado em teste para a chave nova, inclusive sem cascata com o `vector_control` do 35a. O README ganha o parágrafo do módulo. Nenhuma mutation, tela ou SDL novo.

**Architecture:** Conferido no código (`origin/main` do maintenance, `482e53b`; api `85dc80ee`).
- O liga/desliga é genérico: `setCityFeature(citySlug, key, enabled)` sem TOTP, três estados (`featureStatus`) e `missing` vindo do api.
- `missing_for` gera `"#{dependency}_disabled"` para cada `feature:<chave>` cuja dependência não está ligada **e** utilizável. O api do 35b devolve `territory_registry_disabled` tanto com o cadastro desligado quanto com ele ligado e incompleto (sem prontuário, IBGE ou UF).
- Esse código já está em `MISSING_LABELS` (`src/lib/features.ts:78`): "cadastro individual e domiciliar (territory_registry) desligado ou incompleto", e não muda (contrato §15). A mudança de produção é **uma** entrada em `FEATURE_LABELS`, depois de `vector_control` (`features.ts:156-157`), no molde do commit `482e53b`.
- `FeaturesTab.tsx` só renderiza o que o api manda; nenhum aviso de simulado pega a chave nova.

**Papéis, unidade e estoque: nada a fazer** (como no 35a):
- `zoonoses_team`, o veterinário (perfil profissional CBO 223305 + CRMV) e a unidade `environmental_surveillance` são da cidade, no dashboard;
- a aba Contas mostra `row.roles` cru, para todos os módulos;
- os produtos `rabies_vaccine` e `collar` e o local da UVZ são do estoque da cidade.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisão 14; Afeta "`apps/maintenance`"; F-35.11); `docs/adr/0054.md`; o contrato `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md` §1, §11 (gate de aparelhos), §15, §17 (roteiro 1), §18, Q1 e a **Q27** do §19 (README do 35a e descrição do `CATALOG`), que prevalece sobre o §1. Molde: `docs/superpowers/plans/2026-11-16-module-35a-vector-control-maintenance.md` e o commit `482e53b` do maintenance.

## Global Constraints

- **Interruptor** (contrato §1): `zoonoses` exige `feature:territory_registry` ligado e utilizável → `missing` = `territory_registry_disabled`. Descrição do api (Q27, sem o prefixo do §1): "Animal com tutor, vacinação antirrábica, demandas da ouvidoria, observação do agressor, leishmaniose canina e canil". A Task 0 confere o texto exato.
- **Nome** (contrato §1 e §15): `FEATURE_LABELS.zoonoses` = "Centro de zoonoses".
- **Frase do pré-requisito** (contrato §15): `MISSING_LABELS.territory_registry_disabled` já existe e não muda: "cadastro individual e domiciliar (territory_registry) desligado ou incompleto".
- **Sem cascata:**
  - ligar ou desligar manda só aquela `key`;
  - a aba nunca liga o cadastro junto com o zoonoses, nem desliga o zoonoses quando o cadastro é desligado;
  - `vector_control` e `zoonoses` são independentes (§1, "Sem cascata para o `vector_control`").
- **Desligar não apaga nada** (§1): as rotas respondem `403 feature_disabled`, o app recebe `rejected feature_disabled`, os jobs não agem e o que foi gravado fica. A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade: nenhum animal, tutor, microchip, demanda, endereço, observação, eutanásia, produto ou papel passa por aqui.
- Existe em todos os ambientes, sem simulado. Valor desconhecido (api mais novo) aparece cru, nunca `undefined`/`null`.
- **Interface:** nenhuma tela nova; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte.
- **Âncoras** em `origin/main` na execução:
  - em `FEATURE_LABELS`, a última entrada (hoje `vector_control`, `features.ts:157`);
  - no README, depois do parágrafo **Vigilância ambiental de endemias (módulo 35a).** (`README.md:419-435`), antes de **Mapas de campo (módulo 24c).** (`README.md:437`).
  Conflito no rebase com outro módulo é só ordem de linhas.
- **Testes:** só `bin/test maintenance -w maintenance-mod35b [arquivos]` (de `$R`), nunca `vitest` direto. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- **Git:** `/opt/homebrew/bin/git`; nunca `git add -A` nem `git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge e push só com autorização explícita.
- **Board #1:** o maintenance é superfície de F-35.11, com o api e o dashboard. O card vai a In Progress pelo plano do api; a Task 0 só confere.
- **Execução coordenada:** ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod35b`, `A=$R/apps/api/.claude/mod35b`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-35b-zoonoses` a partir de `origin/main`. Os caminhos das tasks são relativos a `$M`.
- **Prova:** Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3061**.
  - Abra `http://maintenance.localhost:5177`, nunca `localhost:5177`.
  - Conta `dev@local` / `dev-password`, com TOTP fixo (`lib/dev_maintainer.rb`); o usuário faz login e TOTP.
  - O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Desligar o cadastro territorial com o zoonoses ligado.** Só `territory_registry` vai na mutation. Depois do refetch, a linha de `zoonoses` diz "ligada, falta pré-requisito", com a frase do cadastro, e não é desligada junto. Teste: Task 1, "desligar territory_registry com zoonoses ligada: só a chave pedida; o zoonoses fica faltando o cadastro".
2. **Cascata com o `vector_control`.** Com a vigilância desligada e o cadastro utilizável, o zoonoses é utilizável. Desligar o zoonoses manda só `zoonoses` e não mexe na vigilância. Teste: Task 1, "sem cascata com a vigilância: vector_control desligada não bloqueia; desligar zoonoses manda só zoonoses".
3. **Api com a descrição antiga do §1** ("Centro de zoonoses: …"), que repete o nome ao lado. A Q27 tirou o prefixo; o teste fixa o nome e a descrição nova como nós distintos. Teste: Task 1, "aparece com o nome em português e a descrição do api", com a descrição da Q27, conferida na Task 0.
4. **Texto que prometa apagar ao desligar** (animais, vacinas, eutanásias). Testes: Task 1, casos da aba com `not.toMatch(/apaga|exclu/i)`; Task 2, grep no README.
5. **Api mais novo que a tela** (um `missing` futuro, como `environmental_unit_missing`). Ele aparece cru, e nenhum aviso de simulado aparece. Testes: Task 1, "módulo 35b: nome; frase do cadastro; missing desconhecido cru" e "zoonoses ligada: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, entrada do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome da chave, fluxo genérico fixado, parágrafo do módulo | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177, parada | 2 |

**Estratégia de teste:**
- **Funções puras** (`featureLabel`, `missingText`, `missingSummary`), por tabela de casos.
- **A aba**, pelo fluxo existente: o `fetch` falso responde por nome de operação, com os helpers `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e o tipo `Feature` do topo de `FeaturesTab.test.tsx`, e o `mockWithMutation` do bloco do 35a. O teste confere as variáveis exatas da mutation e o refetch que mostra o dependente faltando.
- **A Task 2:** a suíte inteira com as guardas de tema, o `codegen:check` (prova de que o SDL não mudou), o build e a prova contra o api do 35b em Curitiba (`record`, com o cadastro) e Maringá (`integrated`, sem o cadastro utilizável).

---

### Task 0: Card, âncoras, entrada do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`482e53b` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); `$A`.
- Produces: `$M`; a base anotada (arquivos e casos); a `description` real de `zoonoses`, usada como `ZO_OFF.description` na Task 1.

- [ ] **Step 1: Conferir o card F-35.11** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 600 --format json --jq '.items[] | select(.title | test("^F-35\\.11 ")) | {id, title, status}'
```

Expected: In Progress ou além (movido pelo plano do api). Em Todo, mova com `gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'` e `gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entrada do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod35b
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'territory_registry_disabled:\|vector_control:\|zoonoses'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Vigilância ambiental de endemias (módulo 35a).\*\*\|^\*\*Mapas de campo (módulo 24c).\*\*\|Coordenação de endemias'
grep -c 'description: "Animal com tutor, vacinação antirrábica, demandas da ouvidoria, observação do agressor, leishmaniose canina e canil"' $A/app/services/platform/features.rb
grep -n 'key: "zoonoses"' -A3 $A/app/services/platform/features.rb
grep -n 'CITIZEN_HIDDEN =' $A/app/services/platform/features.rb
grep -n 'key: "territory_registry"' -A1 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: nenhuma linha com `zoonoses`; `territory_registry_disabled` presente com a frase das Global Constraints; a última entrada de `FEATURE_LABELS` é `vector_control` (ou a de um módulo que entrou depois: use-a como âncora);
- no api: `zoonoses` no fim do `CATALOG`, depois de `vector_control`, com `requires: %w[feature:territory_registry]`, sem `environments` e com a `description` da Q27 (o `grep -c` devolve `1`); `CITIZEN_HIDDEN` com `zoonoses`;
- no README, "Coordenação de endemias" uma vez, no parágrafo do 35a (hoje `README.md:430-431`).

**Anote** a `description` real de `zoonoses` e a de `territory_registry` (hoje "Cadastro territorial: microáreas, domicílios, famílias e cadastro individual com fichas LEDI de cadastro").

Se aparecer outro `requires`, um ambiente restrito, uma chave a mais ou outra `description` (por exemplo, a antiga "Centro de zoonoses: …"): **pare e reporte** ao coordenador, porque o api tem de bater com a Q27. Se o api ainda não tiver a entrada, a Task 1 segue com a `description` da Q27 e a Task 2 espera.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod35b -b feat/mod-35b-zoonoses origin/main
cd $R/.worktrees/maintenance-mod35b && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod35b
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos). A Task 2 espera o **mesmo número de arquivos de teste**, com casos novos só em `features.test.ts` e `FeaturesTab.test.tsx`.

**Relato ao coordenador:** "maintenance-mod35b Task 0: worktree e base verdes (N arquivos/M casos); entrada `zoonoses` do api conferida (descrição da Q27) ou ausente (Task 2 espera); F-35.11 em <estado>."

---

### Task 1: Nome de `zoonoses`, fluxo fixado e README

**Files:**
- Modify: `src/lib/features.ts` (uma entrada em `FEATURE_LABELS`, depois de `vector_control`, com o comentário de uma linha no molde do 35a), `README.md` (parágrafo depois do de Vigilância ambiental de endemias, módulo 35a)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`); o padrão `mockWithMutation(initial, after, requested)` do bloco do 35a (copiado no bloco novo, como o 35a copiou o do 24b).
- Produces: `featureLabel("zoonoses") === "Centro de zoonoses"`. `MISSING_LABELS` não muda.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 35b", () => {
  it("módulo 35b: nome; frase do cadastro; missing desconhecido cru", () => {
    expect(featureLabel("zoonoses")).toBe("Centro de zoonoses");
    expect(missingText("territory_registry_disabled")).toBe("cadastro individual e domiciliar (territory_registry) desligado ou incompleto");
    expect(missingSummary([ "territory_registry_disabled" ])).toBe("cadastro individual e domiciliar (territory_registry) desligado ou incompleto");
    expect(missingText("environmental_unit_missing")).toBe("environmental_unit_missing");
  });

  it("os nomes do 35a não mudam", () => {
    expect(featureLabel("vector_control")).toBe("Vigilância ambiental (endemias)");
    expect(missingText("home_visits_disabled")).toBe("visita domiciliar e territorial (home_visits) desligada ou incompleta");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx`, um `describe("FeaturesTab — centro de zoonoses (módulo 35b)")` com o `beforeEach`/`afterEach` e o `mockWithMutation` do bloco do 35a, as constantes abaixo e seis casos. `TR_ON.description` é a real do api (`85dc80ee`). `ZO_OFF.description` é a da Q27, conferida na Task 0.

```tsx
const REGISTRY_PHRASE = "cadastro individual e domiciliar (territory_registry) desligado ou incompleto";
const TR_ON: Feature = { key: "territory_registry",
  description: "Cadastro territorial: microáreas, domicílios, famílias e cadastro individual com fichas LEDI de cadastro",
  enabled: true, usable: true, missing: [], changedAt: "2026-11-23T12:00:00Z", changedBy: "dev@local" };
const VC_OFF: Feature = { key: "vector_control",
  description: "Boletim do agente na visita, quarteirões, ciclos, ovitrampas, LIRAa e bloqueio de transmissão",
  enabled: false, usable: false, missing: [], changedAt: null, changedBy: null };
const ZO_OFF: Feature = { key: "zoonoses",
  description: "Animal com tutor, vacinação antirrábica, demandas da ouvidoria, observação do agressor, leishmaniose canina e canil", // Q27
  enabled: false, usable: false, missing: [], changedAt: null, changedBy: null };
const ZO_ON: Feature = { ...ZO_OFF, enabled: true, usable: true, changedAt: "2026-11-23T13:00:00Z", changedBy: "dev@local" };
const ZO_BLOCKED: Feature = { ...ZO_ON, usable: false, missing: [ "territory_registry_disabled" ] };
```

| Caso | Lista inicial → depois da mutation | Clique | Asserções |
|---|---|---|---|
| "aparece com o nome em português e a descrição do api" | `[ZO_OFF]` | — | `findByText("zoonoses")`; `getByText("Centro de zoonoses")`; `getByText(ZO_OFF.description)`; `getAllByText("desligada")` com 1; corpo sem `/undefined\|null/` |
| "com o cadastro utilizável: liga utilizável" | `[TR_ON, ZO_OFF]` → `[TR_ON, ZO_ON]` | "ligar", "confirmar: ligar zoonoses" | status = "zoonoses ligada e utilizável."; variáveis `{ citySlug: "sp", key: "zoonoses", enabled: true }`; sem "ligada, falta pré-requisito" |
| "sem o cadastro: ligada, falta o cadastro" | `[ZO_OFF]` → `[ZO_BLOCKED]` | idem | 1 mutation com as mesmas variáveis; status = `` `zoonoses ligada, mas ainda não utilizável — falta: ${REGISTRY_PHRASE}.` ``; "ligada, falta pré-requisito" presente; "ligada e utilizável" ausente; corpo sem `/undefined\|apaga\|exclu/i` |
| "desligar territory_registry com zoonoses ligada: só a chave pedida; o zoonoses fica faltando o cadastro" | `[TR_ON, ZO_ON]` → `[{ ...TR_ON, enabled: false, usable: false, changedAt: "2026-11-23T15:00:00Z" }, ZO_BLOCKED]` | primeiro "desligar", "confirmar: desligar territory_registry" | `waitFor` o `REGISTRY_PHRASE` na tabela; 1 mutation, variáveis `{ citySlug: "sp", key: "territory_registry", enabled: false }`; status = "territory_registry desligada."; "ligada, falta pré-requisito" presente; "ligada e utilizável" ausente; corpo sem `/apaga\|exclu/i` |
| "sem cascata com a vigilância: vector_control desligada não bloqueia; desligar zoonoses manda só zoonoses" | `[TR_ON, VC_OFF, ZO_ON]` → `[TR_ON, VC_OFF, { ...ZO_OFF, changedAt: "2026-11-23T16:00:00Z", changedBy: "dev@local" }]` | segundo "desligar" (o primeiro é o do cadastro), "confirmar: desligar zoonoses" | antes do clique: `getAllByText("ligada e utilizável")` com 2 e `getAllByText("desligada")` com 1; depois: 1 mutation, variáveis `{ citySlug: "sp", key: "zoonoses", enabled: false }`; status = "zoonoses desligada."; `getAllByText("desligada")` com 2; `getByText("Vigilância ambiental (endemias)")` presente |
| "zoonoses ligada: nenhum aviso de simulado" | `[TR_ON, ZO_ON]` | — | `getAllByText("ligada e utilizável")` com 2; `queryAllByRole("note")` com 0; corpo sem `/SIMULADO\|undefined\|null/` |

- [ ] **Step 2: Run tests to verify they fail**

Run (de `$R`): `bin/test maintenance -w maintenance-mod35b src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL nos casos com o nome (`featureLabel("zoonoses")` = `null`; "Centro de zoonoses" não aparece). Os casos que só usam a frase do cadastro podem passar já (a frase existe): eles fixam o fluxo, não guiam a mudança. Os casos antigos passam.

- [ ] **Step 3: Implement**

Uma entrada em `FEATURE_LABELS`, com o valor do Produces e o comentário: "Módulo 35b (ADR 0054): zoonoses exige territory_registry (missing: territory_registry_disabled, acima) e não depende de vector_control. Todos os ambientes, sem simulado."

README, parágrafo **Centro de zoonoses (módulo 35b).** depois do de Vigilância ambiental de endemias (módulo 35a), com:
- o interruptor `zoonoses` ("Centro de zoonoses") liga:
  - no dashboard, o grupo "Zoonoses": animais, campanhas e vacinação antirrábica, demandas, observação do animal agressor, leishmaniose, canil e gatil, relatórios e parâmetros;
  - no wpda, a pergunta de zoonoses na ouvidoria;
  - no app de campo, animal, vacinação, coleta de leishmaniose e coleira;
- exige `territory_registry` ligado e utilizável. Sem ele, ou com ele ligado e incompleto, a linha diz "cadastro individual e domiciliar (territory_registry) desligado ou incompleto";
- não depende de `vector_control` e não o liga nem desliga; a aba não liga nem desliga uma chave pela outra;
- existe em todos os ambientes, sem simulado;
- desligar esconde as telas e recusa registros novos, inclusive os que chegam do app; o que foi gravado fica;
- ligar ou desligar chega aos aparelhos como `reset: true`. Gate de go-live: nenhum aparelho com app anterior ao 35a numa cidade com `zoonoses` ligado (contrato §11);
- a "Equipe de zoonoses", o médico veterinário (perfil profissional CBO 223305 com CRMV), a unidade "Vigilância ambiental e zoonoses", a vacina e a coleira são da cidade, no dashboard; nenhum dado de animal, tutor, microchip ou endereço passa por aqui;
- ordem de deploy: contracts → api → dashboard, wpda e maintenance → campo; contra um api sem o 35b, a chave só não aparece.

No mesmo commit (Q27), troque "Coordenação de endemias" por "Coordenação de vigilância ambiental" no parágrafo do 35a (`README.md:430-431`, Q1). O resto daquele parágrafo não muda.

- [ ] **Step 4: Run tests to verify they pass**

Run (de `$R`): `bin/test maintenance -w maintenance-mod35b src/lib/features.test.ts src/screens/FeaturesTab.test.tsx && cd $M && npx tsc --noEmit`
Expected: PASS; `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the zoonoses center switch in the features tab" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod35b Task 1: `zoonoses` com nome, 8 casos novos verdes (sem cascata com vector_control fixada), README; commit <sha>."

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod35b && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base e 8 casos a mais;
- `codegen:check` limpo (o SDL não mudou);
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 14, F-35.11) e o contrato §1, §15, Q1 e Q27. Ele confere:
  - uma entrada nova em `FEATURE_LABELS` e nenhuma em `MISSING_LABELS`;
  - nenhum aviso de simulado; nenhum dado de cidade, papel ou unidade aqui;
  - que o README tem a ordem de deploy e o gate dos aparelhos do §15 e não promete apagar nada;
  - que o parágrafo do 35a diz "Coordenação de vigilância ambiental" e não diz mais "Coordenação de endemias" (Q27): `grep -c "Coordenação de endemias" $M/README.md` devolve `0`.

  Confira também: `grep -n -i "apaga\|exclu" $M/README.md | grep -i "animal\|zoonoses\|eutanásia\|vacina"`. Expected: sem saída.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 35b na **3061**, com o banco de dev migrado e semeado **com autorização do usuário** (a semente liga `zoonoses` em Curitiba, contrato §17). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod35b --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod35b:/app -v maintenance-mod35b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3061 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod35b   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP. Antes de tudo, anote o estado de `territory_registry`, `vector_control` e `zoonoses` nas duas cidades. Screenshots em `evidencias/2026-11-23-modulo-35b/maintenance/`. Roteiro:
1. **Curitiba (`record`) → Funcionalidades:** `zoonoses` aparece com o nome, a descrição e "ligada e utilizável" (a semente a ligou).
2. **Contas de Curitiba:** `zoonoses@curitiba.demo` tem `zoonoses_team`, cru como os outros papéis; `endemias@curitiba.demo` continua com `endemic_coordinator`.
3. **Maringá (`integrated`, sem o cadastro utilizável):** ligar `zoonoses` → "ligada, falta pré-requisito", com a frase do cadastro; desligue de volta.
4. **Roteiro 1 do contrato (§17), combinado com as sessões do dashboard e do wpda:** desligar `zoonoses` em Curitiba → o dashboard (5221) esconde o grupo "Zoonoses" e o wpda (5222) esconde a pergunta; religue.
5. **Curitiba, só se o usuário autorizar** (as provas do dashboard e do campo dependem das chaves ligadas):
   - desligar `vector_control` → `zoonoses` continua "ligada e utilizável"; religue;
   - desligar `territory_registry` → `zoonoses` continua ligada e passa a "falta pré-requisito"; religue, e o zoonoses volta a utilizável sem novo clique.
   Sem autorização, pule o passo.
6. **Auditoria:** lista as tentativas de `setCityFeature`, só com a chave e o estado.
7. **A 375 px:** sem rolagem lateral da página na aba Funcionalidades.

Depois, volte as chaves ao estado anotado e limpe: `docker rm -f maintenance-mod35b && docker volume rm maintenance-mod35b-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api do 35b (a suíte completa do api é da sessão "Testes", pedida pelo plano do api) e só com autorização explícita (merge, push e card, uma etapa de cada vez).
  - Antes do push, `origin/main..main` só pode ter o commit desta entrega (sessões paralelas).
  - Quando autorizado: `git fetch`, `merge --ff-only feat/mod-35b-zoonoses`, `push origin main`, `worktree remove $M`, `branch -d`.
  - O F-35.11 vai a **Done** só com as partes do api e do dashboard também entregues, e a **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod35b Task 2: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-23-modulo-35b/maintenance/); aguardando autorização de merge."

---

## Divergências propostas ao contrato

> Todas decididas pela **Q27** do §19 do contrato. O plano acima já aplica as decisões.

1. **DM1 — Rótulo da coordenação no README do 35a.** A Q1 renomeia o `endemic_coordinator` para "Coordenação de vigilância ambiental". O parágrafo do 35a no README do maintenance (commit `482e53b`, `apps/maintenance/README.md:430-431`) ainda diz "Coordenação de endemias". O §15 não fala disso, e o maintenance não tem mapa de rótulo de papel (a aba Contas mostra o papel cru), então nenhum código muda.
   - **Proposta:** o commit da Task 1 troca essas duas palavras no parágrafo do 35a.
   - **Resolvida pela Q27:** o parágrafo do 35a passa a dizer "Coordenação de vigilância ambiental". A Task 1 faz a troca, e a Task 2 confere.
2. **DM2 — A descrição repete o nome.** A descrição do §1 começa por "Centro de zoonoses: …", e a aba mostra o nome "Centro de zoonoses" logo ao lado. Não quebra teste, porque `getByText` casa o texto inteiro. Mas é o caso que a Q12 corrigiu para o `vector_control` (MV-2 do 35a) e a Q24 do 24b corrigiu para o `home_visits`.
   - **Proposta:** a descrição passa a ser "Animal com tutor, vacinação antirrábica, demandas, observação do agressor, leishmaniose canina e canil".
   - Isso toca só o `CATALOG` do api e o spec que fixa a descrição.
   - **Resolvida pela Q27:** a descrição fica "Animal com tutor, vacinação antirrábica, demandas da ouvidoria, observação do agressor, leishmaniose canina e canil". A Task 0 confere no api, e a Task 1 a usa nos testes.
3. **DM3 — A frase do README sobre o cidadão depende da D1 do wpda.** O 35a escreveu "nunca chega ao cidadão" para o `vector_control`. O `zoonoses` também entra em `CITIZEN_HIDDEN`, mas o wpda precisa saber dele para mostrar a pergunta da ouvidoria (contrato §1; D1 do plano do wpda: `public_config.zoonoses`).
   - **Proposta:** o README diz só "no wpda, a pergunta de zoonoses na ouvidoria", que vale qualquer que seja a decisão da D1. Não repete o "nunca chega ao cidadão".
   - **Resolvida pela Q27** (com a Q10, que pôs o interruptor no `public_config` do wpda): a redação neutra da Task 1 fica.

**Sem divergência, para registro:** o `zoonoses_team`, o veterinário, a unidade `environmental_surveillance`, os produtos e o local de estoque não pedem nada no maintenance (veja "Architecture"). `MISSING_LABELS.territory_registry_disabled` já cobre o "ligado e incompleto", e não muda (contrato §15).

## Self-review

- **Cobertura** (contrato §1, §15, §17 roteiro 1, Q1, Q27; F-35.11):
  - nome da chave: Task 1 (Review Focus 3 e 5);
  - frase do pré-requisito sem mudança: Task 1, casos puros e da aba;
  - sem cascata com o cadastro e com a vigilância: Task 1, quarto e quinto casos (Review Focus 1 e 2);
  - desligar sem apagar: README da Task 1 e grep da Task 2 (Review Focus 4);
  - README com a ordem de deploy e o gate dos aparelhos (§15): Task 1, conferido na Task 2;
  - "nenhuma mutation nova": `codegen:check` na Task 2;
  - roteiro 1 do §17: Task 2, passo 4.
- **Placeholders:** nenhum. Valor lido na execução: o id do card. A `description` é a da Q27, conferida na Task 0, com parada se for outra.
- **Tipos e nomes:** existem em `482e53b`: `featureLabel`, `missingText`, `missingSummary`, `doneText` (a frase do status), `featureStatus`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` e o `mockWithMutation` do bloco do 35a. A entrada `zoonoses` vem do api do 35b.
- **Proporção:** a mudança de produção é uma linha em `features.ts` e um parágrafo de README. O plano traz os testes numa tabela, não em código.
