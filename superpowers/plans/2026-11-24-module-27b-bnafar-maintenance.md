# Módulo 27b — Envio à BNAFAR (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-24-module-27b-bnafar-contracts.md`, com as decisões Q6–Q18 do §11 (docs `ae1cfbc`). As divergências DM1–DM6 deste plano foram resolvidas por elas (seção final).
> 2. A **Q11** tira do maintenance a importação do saldo inicial: ela passa para o dashboard, feita pela cidade. A spec muda junto, na decisão 6 revista.
> 3. O **api do 27b** na branch `feat/mod-27b-bnafar` (worktree `apps/api/.claude/mod27b`, porta **3062**). Ele precisa ter as cinco chaves no fim de `Platform::Features::CATALOG` e o SDL novo do `Maintenance::Schema`.
> 4. A `main` do maintenance **com o 35b mergeado** (`6644ac8` ou mais novo): a âncora do `FEATURE_LABELS` passa a ser `zoonoses`.
>
> A Task 1 não depende do api. As Tasks 2–4 dependem do SDL do api do 27b. A prova da Task 5 depende do api rodando, com o banco de dev migrado **com autorização do usuário**. Ordem do contrato §10: api → dashboard e **maintenance** → suíte completa → prova contra o simulado.

**Goal:** Lado maintenance do 27b, em três partes:
- os cinco interruptores do envio (Q6), com nome, frases de pré-requisito e o aviso do modo simulado, na aba Funcionalidades;
- uma aba nova da cidade, "BNAFAR", com a situação do envio (a faixa do `Bnafar::Alert`) e o ambiente, que se troca só fora de produção (Q9);
- "Importar OBM" por arquivo, com TOTP, na tela do catálogo de medicamentos (Q12).

Nada de saldo inicial (Q11).

**Architecture:** Conferido no código: `origin/main` do maintenance em `482e53b`; o 35b em `6644ac8`, na branch dele.
- **Aba Funcionalidades.** O liga/desliga é genérico (`FeaturesTab.tsx`, `setCityFeature` sem TOTP, três estados por `featureStatus`). O que muda:
  - entradas novas em `FEATURE_LABELS` e `MISSING_LABELS` (`src/lib/features.ts`);
  - o aviso do simulado, no molde do `RNDS_MOCK_NOTICE` (`src/lib/rnds.ts`).
- **GraphQL.** O maintenance só fala GraphQL em `/graphql` (`src/lib/api.ts`, `gql`). Os documentos são tipados pelo codegen (`src/gql/`, a partir de `schema.graphql`, puxado do api por `scripts/pull-schema.sh`).
  - A recusa de regra chega em `payload.errors: [{ path, message }]`. O `UserError` não tem `code`: o `message` é o código, como em `setFeatureErrorText` e `importErrorText`.
  - A recusa do TOTP vem com `path: "code"` (molde `NotifiableDiseases.tsx:84-96`).
- **A aba nova** tem consulta própria, no molde de `AnalyticsTab` e `FeaturesTab`: contra um api sem o 27b, só ela cai, nunca o topo da cidade.
- **Envio de arquivo.** Não há envio de arquivo no maintenance hoje (nenhum `FileReader` em `src/`). A leitura em base64 do arquivo OBM é nova, no molde do `readBase64` do dashboard (`apps/dashboard/src/lib/api.ts:2652-2662`).

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-24-module-27b-bnafar-design.md` (decisões 6 revista, 8 e 13; §4.4, §4.6). **Contrato:** `docs/superpowers/plans/2026-11-24-module-27b-bnafar-contracts.md` §1, §2 (catálogo OBM), §3 (alerta e homologação), §8, §9 e Q6, Q9, Q11, Q12, Q17, Q18. **Plano do api:** `2026-11-24-module-27b-bnafar-api.md`, Tasks 2, 4 e 15. **Molde:** `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-maintenance.md`; `MedicationCatalog.tsx` (importação) e `NotifiableDiseaseListForm.tsx` (TOTP).

## Global Constraints

- **Interruptores** (contrato §1 e Q6), nesta ordem, no fim do `CATALOG`:

  | Chave | Exige |
  |---|---|
  | `bnafar` | `credential:bnafar` e `ibge_code` |
  | `bnafar_medications` | `feature:bnafar` e `feature:pharmacy` |
  | `bnafar_vector_supplies` | `feature:bnafar` e o domínio vetorial (`vector_control` **ou** `zoonoses`; motivo `vector_supplies_disabled`) |
  | `bnafar_vaccines` | `feature:bnafar` e `feature:vaccination` |
  | `bnafar_mock` | só fora de produção (`environments: development, test, staging`) |

  Desligar não apaga o que foi enviado. Ligar um domínio vale a partir do dia seguinte.
- **Nomes** (contrato §1, §8, Q6), letra por letra:
  - `FEATURE_LABELS`: `bnafar` "Envio à BNAFAR", `bnafar_medications` "BNAFAR: medicamentos", `bnafar_vector_supplies` "BNAFAR: endemias e zoonoses", `bnafar_vaccines` "BNAFAR: vacinas", `bnafar_mock` "BNAFAR simulado".
  - Rótulos que o contrato fixa nesta tela: "Importar OBM" (§8).
- **Frases de pré-requisito** (`MISSING_LABELS`). Os códigos são os do §8; as frases são **propostas**, no molde das da RNDS:

  | Código | Frase |
  |---|---|
  | `credential_missing:bnafar` | "credencial BNAFAR não cadastrada para o ambiente em uso (o administrador da cidade grava no dashboard)" |
  | `credential_unauthorized:bnafar` | "credencial BNAFAR recusada no último teste" |
  | `bnafar_disabled` | "envio à BNAFAR (bnafar) desligado" |
  | `vector_supplies_disabled` | "vigilância ambiental (vector_control) e centro de zoonoses (zoonoses) desligados ou incompletos" |

  As que já existem não mudam: `ibge_code_missing`, `pharmacy_disabled`, `vaccination_disabled`.
- **Sem cascata:** ligar ou desligar manda só aquela `key`. Desligar `bnafar` não desliga os três domínios: eles ficam "ligada, falta pré-requisito".
- **Ambiente** (Q9): em produção, é sempre `production`. A troca para `homologation` existe só fora de produção, e o maintenance só existe fora de produção (`apps/api lib/maintenance_api.rb:10`). A tela oferece a troca e trata a recusa `environment_not_allowed` (plano do api, Valor 8).
- **OBM** (Q12):
  - a carga é só por arquivo, com o mapa de colunas do api;
  - em produção, ela é feita pela rake do operador, fora do maintenance;
  - aqui, pela mutation `importObmCatalog(fileBase64, code)`, com TOTP;
  - o arquivo, o base64 e o TOTP só vivem no estado do formulário: nunca em URL, cache do `queryClient`, `console`, storage ou log;
  - o nome do arquivo não vai ao api.
- **Step-up:** "Importar OBM" pede o TOTP do momento, no molde da publicação da lista de agravos:
  - o campo "Código" com `STEP_UP_CODE_HELP_TEXT`;
  - a recusa no campo, com `StepUpCodeError`;
  - `/^\d{6}$/` conferido antes de enviar (`CODE_MISSING_TEXT`);
  - o código limpo no instante em que o pedido sai.

  Trocar o ambiente usa `ConfirmButton`: o contrato não pede TOTP.
- **Convenção do SDL** (Q17): o TOTP é `code`, com `expectedVersion` onde houver versão. Nenhuma das três operações deste plano tem versão.
- **Frases da tela:** toda frase que o contrato não fixa é **proposta** e está marcada assim. Código desconhecido (situação, domínio, recusa) aparece cru, nunca `undefined`/`null`.
- **O que passa aqui:** a situação do envio da cidade (contagens, datas, ambiente, domínios). Nenhum usuário, senha, token, registro enviado nem dado de paciente passa pelo maintenance: a credencial é da cidade, no dashboard (decisão 12).
- **Interface:**
  - a aba nova segue o `CityDetail`: botão de aba, consulta só quando a aba está aberta;
  - componentes: `Panel`, `DataTable`, `Tag`, `Button`, `ConfirmButton`, `Field`, `ErrorState`, `EmptyState`, `StepUpCode`;
  - cores só por papéis do tema, com `src/theme/*.test.ts` na suíte;
  - 375 px sem rolagem lateral da página.
- **Testes:** só `bin/test maintenance -w maintenance-mod27b [arquivos]` (de `$R`), nunca `vitest` direto. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge e push só com autorização explícita.
- **Board #1:** o maintenance é superfície dos cards do 27b dos interruptores e do catálogo OBM. O card vai a In Progress pelo plano do api; a Task 0 só confere.
- **Execução coordenada:** ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod27b`, `A=$R/apps/api/.claude/mod27b`.
- Worktree **fora de `apps/`**: `$M`, na branch `feat/mod-27b-bnafar` a partir de `origin/main`. Os caminhos das tasks são relativos a `$M`.
- **Prova:**
  - Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3062**;
  - abra `http://maintenance.localhost:5177`, nunca `localhost:5177`;
  - conta `dev@local` / `dev-password`, com TOTP fixo (`lib/dev_maintainer.rb`); o usuário faz login e TOTP.

## Review Focus

1. **Desligar `bnafar` com os domínios ligados.** Espera-se que só `bnafar` vá na mutation. Depois do refetch, `bnafar_medications` diz "ligada, falta pré-requisito", com a frase de `bnafar_disabled`, e não é desligada junto. Teste: Task 1, "desligar bnafar não desliga os domínios".
2. **O simulado ligado sem que se perceba.** Espera-se que, com `bnafar_mock` ligado, o aviso da BNAFAR simulada apareça na aba, e que, desligado ou ausente (produção), não apareça. Teste: Task 1, "simulado ligado: o aviso da BNAFAR" e "simulado desligado: nenhum aviso".
3. **Importar o OBM com o código vencido, duas vezes ou com arquivo grande.** Espera-se:
   - código curto recusado antes de enviar, sem gastar tentativa;
   - duplo clique manda uma mutation só;
   - recusa do api no campo do código, com o aviso de bloqueio;
   - código limpo depois do envio;
   - arquivo acima do teto recusado antes de enviar, sem chamar o api.

   Teste: Task 4, "OBM: TOTP conferido antes, um pedido, recusa no campo, código limpo, arquivo grande recusado".
4. **Trocar o ambiente e a recusa do api.** Espera-se que a troca mande só `citySlug` e `environment` e releia a situação, e que `environment_not_allowed` mostre a frase e deixe o ambiente como estava. Teste: Task 3, "ambiente: troca relê; recusa mantém o ambiente".
5. **Api mais novo que a tela** (um `missing` ou um domínio futuro). Espera-se que apareça cru e que nenhum aviso de simulado indevido apareça. Teste: Task 1, "chave e missing desconhecidos crus"; Task 3, "domínio desconhecido cru".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/bnafar.ts`, `src/lib/bnafar.test.ts`, `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx` | nomes, frases de pré-requisito, aviso do simulado | 1 |
| `schema.graphql`, `src/gql/*`, `src/lib/bnafar.ts`, `src/lib/bnafar.test.ts`, `src/lib/files.ts`, `src/lib/files.test.ts` | SDL do 27b; rótulos do envio; leitura de arquivo em base64 | 2 |
| `src/screens/CityDetail.tsx`, `src/screens/CityDetail.test.tsx`, `src/screens/BnafarStatusPanel.tsx`, `src/screens/BnafarStatusPanel.test.tsx` | aba "BNAFAR": situação e ambiente | 3 |
| `src/screens/MedicationCatalog.tsx`, `src/screens/MedicationCatalog.test.tsx`, `src/lib/medications.ts`, `src/lib/medications.test.ts` | "Importar OBM" | 4 |
| `README.md` | parágrafo do módulo; suíte, codegen:check, build, revisão, prova na 5177, parada | 5 |

**Estratégia de teste:**
- **Funções puras** (`featureLabel`, `missingText`, `bnafarMockOn`, `alertLines`, `domainLabel`, `environmentLabel`, `readFileBase64`, `obmImportText`), por tabela de casos.
- **As telas**, pelo fluxo real, com o `fetch` falso que responde por nome de operação:
  - os helpers do topo de `FeaturesTab.test.tsx`, e o `mockWithMutation` do bloco do 35b, na aba;
  - `reply`, `bodyOf`, `calls`, `route` e `renderScreen`, copiados do topo de `MedicationCatalog.test.tsx`, nos painéis novos;
  - o teste confere as variáveis exatas de cada mutation, o que **não** sai (nenhuma chamada com código curto ou arquivo grande, o TOTP fora de qualquer query), a releitura depois de cada ato e a frase de cada recusa.
- **A Task 5:** a suíte inteira com as guardas de tema, o `codegen:check` (os tipos gerados batem com o SDL puxado), o build e a prova contra o api do 27b em Curitiba.

---

### Task 0: Card, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (com o 35b); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); `$A`.
- Produces: `$M`; a base anotada; as `description` reais das cinco chaves; o SDL real do 27b comparado com as formas da Task 2.

- [ ] **Step 1: Conferir os cards do 27b.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-27\\.([89]|[1-9][0-9]) ")) | {id, title, status}'
```

Expected: os cards em In Progress ou além. Em Todo, mova com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod27b
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'zoonoses:\|"credential_unauthorized:rnds"\|home_visits_disabled:\|bnafar'
$G -C $R/apps/maintenance show origin/main:src/screens/CityDetail.tsx | grep -n 'type TabKey\|{ key: "features", label: "Funcionalidades" }\|activeTab === "features" &&'
$G -C $R/apps/maintenance show origin/main:src/screens/MedicationCatalog.tsx | grep -n 'label="importar as listas da Anvisa"'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Centro de zoonoses (módulo 35b).\*\*'
grep -n 'key: "bnafar' -A4 $A/app/services/platform/features.rb
grep -n 'CITIZEN_HIDDEN =' -A3 $A/app/services/platform/features.rb
grep -rn 'setBnafarEnvironment\|cityBnafarStatus\|importObmCatalog\|OpeningImport' $A/app/graphql | head -20
```

Expected:
- em `features.ts`: `zoonoses` é a última entrada de `FEATURE_LABELS` (ou a de um módulo que entrou depois: use-a como âncora), e nenhuma linha tem `bnafar`;
- no api:
  - as cinco chaves no fim do `CATALOG`, nesta ordem, com os `requires` do §1;
  - o `bnafar_mock` só fora de produção;
  - `setBnafarEnvironment`, `cityBnafarStatus` e `importObmCatalog` no GraphQL;
  - **nenhum** `OpeningImport` no GraphQL (Q11).

**Anote** a `description` real de cada chave e os argumentos reais de cada campo GraphQL. **Onde o api diz outra coisa, vale o api**: a task dona ajusta o documento GraphQL e o teste, e o relato ao coordenador cita a diferença. Se o SDL ainda tiver as mutations da importação (o plano do api antes da Q11), elas ficam sem uso aqui; avise o coordenador.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod27b -b feat/mod-27b-bnafar origin/main
cd $R/.worktrees/maintenance-mod27b && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod27b
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos).

**Relato ao coordenador:** "maintenance-mod27b Task 0: worktree e base verdes (N arquivos/M casos); chaves do api conferidas (ou ausentes); SDL do 27b <presente|ausente>; diferenças: <…>."

---

### Task 1: Nomes dos interruptores, frases de pré-requisito e aviso do simulado

**Files:**
- Create: `src/lib/bnafar.ts`, `src/lib/bnafar.test.ts`
- Modify:
  - `src/lib/features.ts`: quatro entradas em `MISSING_LABELS` e cinco em `FEATURE_LABELS`, depois de `zoonoses`;
  - `src/screens/FeaturesTab.tsx`: o aviso.
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary`, `doneText` (`features.ts`); os helpers do topo de `FeaturesTab.test.tsx` e o `mockWithMutation` do bloco do 35b.
- Produces:
  - os rótulos e as frases das Global Constraints;
  - em `src/lib/bnafar.ts`:
    - `BNAFAR_MOCK_KEY = "bnafar_mock"`;
    - `BNAFAR_MOCK_NOTICE = "Esta cidade envia à BNAFAR SIMULADA — as remessas recebem aceite ou recusa de exemplo e não chegam ao Ministério da Saúde (fora de produção)."` (proposta; Q6: o mesmo aviso do `rnds_mock`);
    - `bnafarMockOn(features: { key: string; enabled: boolean }[]): boolean`, pelo estado **ligado**;
  - no `FeaturesTab`, um `<p role="note" aria-label="modo da BNAFAR">` com o aviso, junto dos outros.

- [ ] **Step 1: Write the failing tests**

  No fim de `src/lib/features.test.ts`, um `describe("módulo 27b")` com:
  - os cinco `featureLabel`, com os rótulos acima;
  - as quatro `missingText`, com as frases;
  - `missingSummary([ "credential_missing:bnafar", "ibge_code_missing" ])`, com as duas frases unidas por "; ";
  - um código futuro, que volta cru;
  - os do 35a e do 35b sem mudança.

  Em `src/lib/bnafar.test.ts`: `bnafarMockOn` é verdadeiro só com `bnafar_mock` ligado, e falso sem a chave.

  No fim de `src/screens/FeaturesTab.test.tsx`, um `describe("FeaturesTab — envio à BNAFAR (módulo 27b)")`, com as constantes `BN_OFF`, `BN_BLOCKED` (`missing: [ "credential_missing:bnafar" ]`), `BN_ON`, `MED_ON`, `MED_BLOCKED` (`missing: [ "bnafar_disabled" ]`), `VEC_BLOCKED` (`missing: [ "vector_supplies_disabled" ]`) e `MOCK_ON` (as `description` anotadas na Task 0):

| Caso | Lista inicial → depois da mutation | Clique | Asserções |
|---|---|---|---|
| "as cinco chaves com nome e descrição do api" | `[BN_OFF, MED_OFF, VEC_OFF, VAC_OFF, MOCK_OFF]` | — | os cinco rótulos e as descrições; `getAllByText("desligada")` com 5; corpo sem `/undefined\|null/` |
| "ligar bnafar sem credencial: ligada, falta a credencial" | `[BN_OFF]` → `[BN_BLOCKED]` | "ligar", "confirmar: ligar bnafar" | variáveis `{ citySlug: "sp", key: "bnafar", enabled: true }`; status com a frase de `credential_missing:bnafar` |
| "desligar bnafar não desliga os domínios" | `[BN_ON, MED_ON]` → `[{ ...BN_ON, enabled: false, usable: false }, MED_BLOCKED]` | primeiro "desligar", "confirmar: desligar bnafar" | 1 mutation, só com `bnafar`; `bnafar_medications` em "ligada, falta pré-requisito", com "envio à BNAFAR (bnafar) desligado" |
| "endemias e zoonoses sem nenhum dos dois domínios" | `[BN_ON, VEC_BLOCKED]` | — | a frase de `vector_supplies_disabled` na linha |
| "simulado ligado: o aviso da BNAFAR" | `[BN_ON, MOCK_ON]` | — | `getByRole("note", { name: "modo da BNAFAR" })` com `BNAFAR_MOCK_NOTICE` |
| "simulado desligado: nenhum aviso" | `[BN_ON, { ...MOCK_ON, enabled: false }]` | — | `queryByRole("note", { name: "modo da BNAFAR" })` nulo |
| "chave e missing desconhecidos crus" | `[{ ...BN_ON, usable: false, missing: [ "bnafar_catalog_missing" ] }]` | — | "bnafar_catalog_missing" na linha; nenhum aviso da BNAFAR |

- [ ] **Step 2: Run tests to verify they fail** — `bin/test maintenance -w maintenance-mod27b src/lib/features.test.ts src/lib/bnafar.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos com rótulo, frase e aviso; os casos antigos passam.
- [ ] **Step 3: Implement.** As entradas levam um comentário de uma linha: "Módulo 27b: bnafar exige credential:bnafar (do ambiente em uso) e ibge_code; os domínios exigem bnafar e o interruptor do domínio (vector_supplies: vector_control ou zoonoses). bnafar_mock só fora de produção (Q6)." O aviso no `FeaturesTab` segue o molde do aviso da RNDS.
- [ ] **Step 4: Run tests to verify they pass** — o mesmo comando, e `cd $M && npx tsc --noEmit`. Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/lib/bnafar.ts src/lib/bnafar.test.ts src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx
$G commit -m "feat: name the BNAFAR switches, their prerequisites and the simulated mode notice" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: SDL do 27b, rótulos do envio e leitura de arquivo

**Files:**
- Modify: `schema.graphql`, `src/gql/*` (gerados), `src/lib/bnafar.ts`, `src/lib/bnafar.test.ts`
- Create: `src/lib/files.ts`, `src/lib/files.test.ts`

**Interfaces:**
- Consumes: o SDL do api do 27b (Task 0).
- Produces: as formas do SDL que este plano usa, pelo plano do api (Tasks 4 e 15) e pela Q12 e Q17. O SDL real manda, e os documentos das Tasks 3 e 4 seguem ele:

```graphql
type Query { cityBnafarStatus(citySlug: String!): CityBnafarStatus }
type CityBnafarStatus { environment: String! enabledDomains: [String!]! startedOn: ISO8601Date alert: BnafarAlert! }
type BnafarAlert { late: Int! lost: Int! rejected: Int! lastFinishedOn: ISO8601Date }
type Mutation {
  setBnafarEnvironment(citySlug: String!, environment: String!): SetBnafarEnvironmentPayload
  importObmCatalog(fileBase64: String!, code: String!): ImportObmCatalogPayload
}
type SetBnafarEnvironmentPayload { ok: Boolean! errors: [UserError!]! }
type ImportObmCatalogPayload { ok: Boolean! errors: [UserError!]! added: Int changed: Int deactivated: Int vmpLinked: Int }
```

- `src/lib/files.ts`:
  - `readFileBase64(file: File): Promise<string>`: `FileReader`, sem o prefixo `data:`; molde do dashboard;
  - `fileTooLarge(file: File, maxBytes: number): boolean`.
- `src/lib/bnafar.ts`, somado ao da Task 1 (rótulos propostos; código desconhecido cru):
  - `environmentLabel`: `production` "produção", `homologation` "homologação";
  - `domainLabel`: `medication` "medicamentos", `vector_control` "endemias e zoonoses", `immunobiological` "vacinas";
  - `alertLines(alert)`, as mesmas frases do plano do dashboard (Task 2, propostas):
    - `late` → "A remessa de ontem ainda não foi aceita pela BNAFAR. Depois da meia-noite de hoje, o dia só volta por exceção.";
    - `rejected` e `lost` → as frases com a contagem;
  - `BNAFAR_ERRORS`:
    - `environment_not_allowed` → "o ambiente de homologação só existe fora de produção";
    - `invalid_environment` → "ambiente desconhecido";
  - `bnafarErrorText(errors)`.

- [ ] **Step 1: Puxar o SDL e gerar os tipos** (com o api do 27b de pé no container)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; M=$R/.worktrees/maintenance-mod27b
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod27b api bin/rails runner 'puts Maintenance::Schema.to_definition' > $M/schema.graphql
cd $M && /opt/homebrew/bin/git diff --stat schema.graphql && npm run codegen && npx tsc --noEmit && echo sdl-ok
```

Expected: o diff só acrescenta tipos e campos do 27b, sem nada removido, e termina em `sdl-ok`. Remoção ou campo renomeado: **pare e reporte** ao coordenador.

- [ ] **Step 2: Write the failing tests**
  - `src/lib/files.test.ts`: `readFileBase64(new File([ "abc" ], "a.obm"))` = `"YWJj"`; `fileTooLarge` na borda (igual ao teto passa; um byte acima, não).
  - `src/lib/bnafar.test.ts`:
    - `alertLines({ late: 1, rejected: 2, lost: 0, lastFinishedOn: null })` com as duas frases;
    - `domainLabel("vector_control")`;
    - `domainLabel("insumos")` = `"insumos"`;
    - `environmentLabel("homologation")`;
    - `bnafarErrorText([{ message: "environment_not_allowed" }])`.
- [ ] **Step 3: Run tests to verify they fail** — `bin/test maintenance -w maintenance-mod27b src/lib/files.test.ts src/lib/bnafar.test.ts`. Expected: FAIL.
- [ ] **Step 4: Implement** pelas Interfaces; rode e veja passar; `npx tsc --noEmit && npm run codegen:check`.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add schema.graphql src/gql src/lib/files.ts src/lib/files.test.ts src/lib/bnafar.ts src/lib/bnafar.test.ts
$G commit -m "feat: pull the BNAFAR maintenance schema and add the submission labels and file reader" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Aba "BNAFAR": situação do envio e ambiente

**Files:**
- Create: `src/screens/BnafarStatusPanel.tsx`, `src/screens/BnafarStatusPanel.test.tsx`
- Modify: `src/screens/CityDetail.tsx`, `src/screens/CityDetail.test.tsx`
  - `TabKey` + `"bnafar"`;
  - `TABS` + `{ key: "bnafar", label: "BNAFAR" }` (proposta), depois de "Funcionalidades";
  - `{activeTab === "bnafar" && <BnafarStatusPanel slug={slug} />}`.

**Interfaces:**
- Consumes: `cityBnafarStatus` e `setBnafarEnvironment` (SDL, Task 2); `alertLines`, `domainLabel`, `environmentLabel`, `bnafarErrorText` (Task 2).
- Produces: `BnafarStatusPanel({ slug })`, com consulta própria `CityBnafarStatus` (chave `[ "city", slug, "bnafar" ]`, `staleTime: Infinity`, botão "atualizar"). Mostra:
  - a faixa (`role="alert"`) com `alertLines`, só quando há alerta;
  - "Ambiente: <produção|homologação>";
  - "Envio desde: <data>" ou "ainda não começou";
  - "Última remessa concluída: <data>" ou "nenhuma ainda";
  - "Tipos no envio: <domínios>", os `enabledDomains` por `domainLabel`, ou "nenhum".

  O `ConfirmButton` "trocar para <outro ambiente>" (confirmação "confirmar: trocar o envio desta cidade para <outro>") vem com a nota proposta "A troca muda a credencial em uso; o envio do dia seguinte já vai para o ambiente novo." A recusa (`environment_not_allowed`, Q9) mostra a frase e mantém o ambiente. Contra um api sem o 27b (`undefinedField`), a frase "esta API ainda não tem o envio à BNAFAR — o api do módulo 27b precisa subir antes do maintenance".

- [ ] **Step 1: Write the failing tests** (`BnafarStatusPanel.test.tsx`, com os helpers copiados do topo de `MedicationCatalog.test.tsx`):
  - a faixa com as duas frases, os domínios e as datas;
  - sem alerta, nenhum `role="alert"`;
  - "domínio desconhecido cru";
  - o caso do Review Focus 4, "ambiente: troca relê; recusa mantém o ambiente": a mutation leva exatamente `{ citySlug: "sp", environment: "homologation" }` e a consulta é relida; `ok: false` com `environment_not_allowed` mostra a frase, e o ambiente continua "produção";
  - api antigo → a frase.

  No `CityDetail.test.tsx`: abrir o detalhe não consulta `cityBnafarStatus`; abrir a aba consulta.
- [ ] **Step 2–4:** rode e veja falhar (`src/screens/BnafarStatusPanel.test.tsx src/screens/CityDetail.test.tsx`); implemente; rode e veja passar; `tsc`, `codegen:check`.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/screens/CityDetail.tsx src/screens/CityDetail.test.tsx src/screens/BnafarStatusPanel.tsx src/screens/BnafarStatusPanel.test.tsx src/gql
$G commit -m "feat: show the city BNAFAR submission status and environment in a new city tab" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: "Importar OBM" por arquivo

**Files:**
- Modify: `src/screens/MedicationCatalog.tsx`, `src/screens/MedicationCatalog.test.tsx`, `src/lib/medications.ts`, `src/lib/medications.test.ts`

**Interfaces:**
- Consumes: `importObmCatalog(fileBase64, code)` (SDL, Task 2); `readFileBase64`, `fileTooLarge` (Task 2); `STEP_UP_CODE_HELP_TEXT`, `StepUpCodeError`, `CODE_MISSING_TEXT`; o padrão das duas importações que já existem na tela.
- Produces:
  - Uma seção "OBM — VMP e apresentações (AMPP)" (proposta) na tela do catálogo, com:
    - `<input type="file" accept=".csv,text/csv" aria-label="Arquivo oficial do OBM">`;
    - o campo "Código" (TOTP);
    - o botão **"Importar OBM"** (contrato §8).
  - Arquivo acima de `OBM_MAX_BYTES = 5 * 1024 * 1024` (o teto do plano do api) é recusado antes de enviar, com "o arquivo passa de 5 MB" (proposta), sem chamar o api.
  - `obmImportText(r)` em `medications.ts` (proposta): "OBM importado: <n> entraram, <n> mudaram, <n> desativadas, <n> medicamentos ligados ao VMP. O AMPP passa a ser escolhido entre as apresentações de cada VMP. Os sem correspondência vão para a lista de revisão." (Q12).
  - As recusas, por `importErrorText`: `invalid_file`, `import_in_progress` e `import_failed` já têm frase; entram `file_too_large` e `suspicious_drop` (o do CATMAT, já existente). O TOTP recusado vai ao campo do código.
  - Uma frase no parágrafo da tela (proposta): "O OBM traz o VMP de cada medicamento e as apresentações (AMPP) de cada fabricante, usados no envio à BNAFAR. O portal não tem download automático: importe o arquivo oficial."

- [ ] **Step 1: Write the failing tests** (o caso do Review Focus 3 tem o nome "OBM: TOTP conferido antes, um pedido, recusa no campo, código limpo, arquivo grande recusado"):
  - sem arquivo, ou com o código curto, nada sai;
  - com arquivo e código, `importObmCatalog` leva exatamente `{ fileBase64, code }` e o código é limpo;
  - o duplo clique manda uma mutation;
  - a recusa com `path: "code"` vai ao campo;
  - arquivo acima do teto: nenhuma chamada;
  - durante a importação, os botões das três importações ficam desabilitados (`IMPORT_RUNNING`);
  - `ok: false` com `suspicious_drop` → a frase;
  - nenhuma query leva `code` ou `fileBase64` nas variáveis.

  Em `medications.test.ts`: `obmImportText`.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`, `codegen:check`.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/screens/MedicationCatalog.tsx src/screens/MedicationCatalog.test.tsx src/lib/medications.ts src/lib/medications.test.ts src/gql
$G commit -m "feat: import the OBM catalog from the official file with step-up" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: README, suíte, build, revisão e prova no navegador

**Files:** `README.md`.

- [ ] **Step 1: README.** Um parágrafo **Envio à BNAFAR (módulo 27b).** depois do de **Centro de zoonoses (módulo 35b).**, que diz:
  - os cinco interruptores e o que cada um exige, com as frases da aba;
  - que o `bnafar_mock` existe só fora de produção;
  - que desligar não apaga o que foi enviado, e que ligar um tipo vale a partir do dia seguinte;
  - que a credencial é da cidade, gravada no dashboard pelo `municipal_admin`, e que nada dela passa aqui;
  - o que a aba "BNAFAR" mostra (situação e ambiente) e que o ambiente se troca só fora de produção;
  - como funciona "Importar OBM": por arquivo, com TOTP; em produção, pela rake do operador (Q12);
  - que o saldo inicial é importado pela cidade, no dashboard ("Saldo inicial", Q11), e não aqui;
  - a ordem de deploy: api, depois dashboard e maintenance;
  - que, contra um api sem o 27b, a aba e a seção do OBM mostram a frase do api antigo;
  - que os interruptores em produção dependem do maintenance em produção, uma pendência da plataforma (Q18).

  Commit:

```bash
cd $M && $G add README.md
$G commit -m "docs: describe the BNAFAR switches, status tab and OBM import in the maintenance README" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 2: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod27b && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base + 3 arquivos de teste novos (`src/lib/bnafar.test.ts`, `src/lib/files.test.ts`, `src/screens/BnafarStatusPanel.test.tsx`) e casos novos em `features.test.ts`, `FeaturesTab.test.tsx`, `CityDetail.test.tsx`, `MedicationCatalog.test.tsx` e `medications.test.ts`;
- `codegen:check` limpo;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 3: Varreduras**

```bash
cd $M
grep -rnE "localStorage|sessionStorage|console\.(log|info|warn|error)" src/screens/BnafarStatusPanel.tsx src/lib/bnafar.ts src/lib/files.ts && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "queryKey:.*(fileBase64|code)" src/screens/MedicationCatalog.tsx src/screens/BnafarStatusPanel.tsx && echo "ERRO: arquivo ou TOTP no cache" || echo ok-cache
grep -rn "OpeningImport\|opening_import" src && echo "ERRO: importação no maintenance (Q11)" || echo ok-q11
```

Expected: `ok-storage`, `ok-cache`, `ok-q11`.

- [ ] **Step 4: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 6 revista e 13) e o contrato (§1, §8, Q6, Q9, Q11, Q12, Q17). Ele confere:
  - os rótulos letra por letra;
  - nenhuma cascata no liga/desliga;
  - o aviso do simulado;
  - o TOTP só no estado e limpo no envio;
  - nada de usuário, senha ou dado de pessoa;
  - a aba com consulta própria;
  - nenhuma importação de saldo inicial;
  - um README sem promessa de apagar dado enviado.

- [ ] **Step 5: Prova no navegador na 5177 (com o usuário).** Depende do api do 27b na **3062**, com o banco de dev migrado e o simulado `fake-bnafar-dev` de pé, **com autorização do usuário** (plano do api). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod27b --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod27b:/app -v maintenance-mod27b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3062 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod27b   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP. Antes de tudo, anote o estado das cinco chaves em Curitiba e Maringá. As screenshots vão em `evidencias/2026-11-24-modulo-27b/maintenance/`. Roteiro:
1. **Funcionalidades (Curitiba):**
   - as cinco chaves aparecem com nome e descrição;
   - ligar `bnafar` sem credencial → "ligada, falta pré-requisito", com a frase da credencial;
   - ligar `bnafar_mock` → o aviso da BNAFAR simulada;
   - `bnafar_medications` ligado com `bnafar` desligado → "envio à BNAFAR (bnafar) desligado".
2. **Combinado com a sessão do dashboard (5223):** o `municipal_admin` grava a credencial; ao recarregar a aba, `bnafar` aparece "ligada e utilizável".
3. **Aba "BNAFAR":**
   - a situação (antes do primeiro envio: "ainda não começou") e os tipos no envio;
   - trocar para homologação e voltar;
   - depois de o api rodar o envio contra o simulado, a faixa e "Última remessa concluída".
4. **Catálogo de medicamentos:**
   - "Importar OBM" com o arquivo da semente e o TOTP → a frase com as contagens;
   - um código curto é recusado sem chamar o api;
   - no dashboard, a entrada de medicamento oferece o AMPP.
5. **Auditoria:** aparecem `maintenance.obm_catalog.imported` e `maintenance.bnafar.environment_changed`, só com ids e contagens.
6. **A 375 px:** a aba nova não tem rolagem lateral da página.

Depois, volte as chaves e o ambiente ao estado anotado e limpe: `docker rm -f maintenance-mod27b && docker volume rm maintenance-mod27b-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 6: Pare.** O merge só vem depois do api do 27b (a suíte completa do api é da sessão "Testes", pedida pelo plano do api) e só com autorização explícita, uma etapa de cada vez: merge, push e card.
  - Antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas).
  - Quando autorizado: `git fetch`, `merge --ff-only feat/mod-27b-bnafar`, `push origin main`, `worktree remove $M`, `branch -d`.
  - Os cards vão a **Done** só com as partes do api e do dashboard também entregues, e a **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod27b Task 5: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-24-modulo-27b/maintenance/); aguardando autorização de merge."

---

## Self-review

- **Cobertura** (contrato §8 depois da Q11 e da Q17; Q6, Q9, Q12):
  - interruptores com `FEATURE_LABELS`, as quatro `MISSING_LABELS` e o aviso do `bnafar_mock`: Task 1;
  - situação do envio com a faixa do `Bnafar::Alert` e o ambiente fora de produção: Task 3;
  - "Importar OBM" por arquivo, com TOTP: Task 4;
  - os nomes de auditoria do §9 que passam por aqui: conferidos na prova (Task 5, passo 5).
  - A importação do saldo inicial saiu (Q11) e é do plano do dashboard (Task 11 dele).
- **Placeholders:** nenhum. Valores lidos na execução: os ids dos cards, as `description` das chaves e o SDL real (Task 0), com parada se o SDL remover ou renomear algo.
- **Tipos e nomes:** existem em `482e53b`/`6644ac8`: `featureLabel`, `missingText`, `missingSummary`, `doneText`, `featureStatus`, `gql`, `graphql`, `GraphQLRefusal`, `ConfirmButton`, `Field`, `StepUpCodeError`, `STEP_UP_CODE_HELP_TEXT`, `CODE_MISSING_TEXT`, `importErrorText`, `IMPORT_RUNNING`, os helpers de teste do `MedicationCatalog.test.tsx` e o `mockWithMutation` do `FeaturesTab.test.tsx`. Os campos GraphQL vêm do SDL do 27b (Task 2).
- **Proporção:** as mudanças de produção são:
  - entradas de tabela;
  - duas funções puras;
  - um painel;
  - uma seção na tela do catálogo.

  O plano traz os testes em tabela, só com as assinaturas.

## Divergências propostas ao contrato

Todas foram decididas no §11 do contrato (docs `ae1cfbc`), e o plano já aplica as decisões. O texto de cada uma fica como registro.

1. **DM1 — Resolvida pela Q11 e pela Q18.** O modelo da planilha num caminho que o maintenance não alcança, e a importação sem produção. A API de manutenção não existe em produção (`apps/api lib/maintenance_api.rb:10`), então a importação feita pela plataforma não chegaria à cidade real. **Decisão:**
   - a importação do saldo inicial passa para o dashboard, feita pela cidade, com o modelo em `GET /pharmacy/opening_imports/template.csv`, e sai do maintenance;
   - os interruptores em produção herdam o gate do maintenance em produção (Q18).
2. **DM2 — Resolvida pela Q17.** O argumento do TOTP é `code`, com `expectedVersion` onde houver versão.
3. **DM3 — Resolvida pela Q9.** Em produção, o ambiente é sempre `production`. A credencial e o ambiente `homologation` só existem fora de produção, onde o maintenance troca o ambiente. A Task 3 oferece a troca e trata `environment_not_allowed`.
4. **DM4 — Resolvida pela Q11 e pela Q12.** As formas que faltavam eram da importação (relatório, linhas, lista e versão), que saiu do maintenance. A carga do OBM é só por arquivo. A consulta do estado do OBM, que a DM4 propunha, sai deste plano: a tela mostra o resultado da importação, e nada mais.
5. **DM5 — Resolvida pela Q17.** Os códigos de erro de linha da planilha e o BOM do Excel são os do plano do api. Com a Q11, eles passam para o plano do dashboard.
6. **DM6 — Resolvida pela Q6.** O `bnafar_mock` é a quinta chave do `CATALOG`, só fora de produção, com o rótulo "BNAFAR simulado" e o mesmo aviso de modo simulado do `rnds_mock` (Task 1).

**Sem divergência, para registro:** a credencial `bnafar` (usuário, senha e ambiente) não passa pelo maintenance (decisão 12). Os códigos `ibge_code_missing`, `pharmacy_disabled` e `vaccination_disabled` já têm frase em `MISSING_LABELS` e não mudam.
