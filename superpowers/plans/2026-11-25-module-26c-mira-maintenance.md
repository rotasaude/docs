# Módulo 26c — Envio da regulação à RNDS (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-25-module-26c-mira-contracts.md` (fonte única), §1, §7 e Q14 (§10; docs `5d49ef7`). As divergências DM1 e DM2 deste plano foram resolvidas pela Q14 (seção final), e o plano já aplica a decisão.
> 2. A **`main` do maintenance com o 27b mergeado.** O contrato manda executar o 26c depois do 27b. Quando este plano foi escrito, o `origin/main` estava em `482e53b`, só com o 35a.
> 3. O **api do 26c** na branch `feat/mod-26c-mira` (worktree `apps/api/.claude/mod26c`, porta **3063**), com `rnds_regulation` no fim do `Platform::Features::CATALOG` e o `rnds_mock` exigindo um dos dois envios (requisito `rnds_sending`, motivo `rnds_sending_disabled`, Q14).
>
> A Task 1 não depende do api: as frases são tabelas e o liga/desliga é o genérico. A prova da Task 2 depende do api rodando, com o banco de dev migrado **com autorização do usuário**. Ordem: api → dashboard e **maintenance** → suíte completa do api → prova contra o simulado.

**Goal:** Lado maintenance do 26c, na aba Funcionalidades:
- o nome do interruptor `rnds_regulation`;
- as frases de pré-requisito novas;
- o aviso do `rnds_mock` citando só os envios ligados (Q14).

Nada mais passa pelo maintenance: a credencial é da cidade, no dashboard (contrato §1), e a situação do envio é do dashboard (§6).

**Architecture:** Conferido no código (`origin/main` em `482e53b`):
- O liga/desliga é genérico: `FeaturesTab.tsx`, `setCityFeature` sem TOTP, três estados por `featureStatus`. A chave nova aparece sozinha, com a `description` do api; o maintenance só dá o nome em português.
- `src/lib/features.ts`:
  - entradas novas em `FEATURE_LABELS` e `MISSING_LABELS`;
  - `uf_missing` e `ibge_code_missing` já existem e não mudam;
  - `care_regulation_disabled` **não** existe e entra (o contrato §7 previa o caso).
- `src/lib/rnds.ts`: o aviso passa a sair de `rndsMockNotice(features)`, que escolhe o texto pelos envios **ligados** (Q14). O `RNDS_MOCK_NOTICE` de hoje (só vacinas) não muda, e o `rndsMockOn` continua pelo estado ligado.
- **Sem GraphQL novo:** a lista de interruptores vem do `city.features` que já existe. O `schema.graphql` não muda, e o `codegen:check` fica limpo sem gerar nada.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-25-module-26c-mira-design.md` (decisão 11; §3.5, "Maintenance"). **Contrato:** `docs/superpowers/plans/2026-11-25-module-26c-mira-contracts.md` §1, §7 e Q14. **Plano do api:** `2026-11-25-module-26c-mira-api.md`, Valor 1 e Task 2. **Molde:** `docs/superpowers/plans/2026-11-24-module-27b-bnafar-maintenance.md` (Task 1); `src/lib/rnds.ts` e o bloco "FeaturesTab — RNDS simulada (módulo 29)" de `src/screens/FeaturesTab.test.tsx`.

## Global Constraints

- **Interruptor** (contrato §1): `rnds_regulation`, no fim do `CATALOG`, com `requires: feature:care_regulation credential:rnds_regulation ibge_code uf`. O `rnds_mock` passa a exigir `rnds_immunization` **ou** `rnds_regulation` utilizável, com o motivo `rnds_sending_disabled` (Q14). Com ele ligado, os envios ligados vão ao simulado.
- **Nomes**, letra por letra: `FEATURE_LABELS.rnds_regulation` = "Envio da regulação à RNDS" (contrato §7). Os de hoje não mudam: `rnds_immunization` "Envio das vacinas à RNDS (Ministério da Saúde)" e `rnds_mock` "RNDS simulada (desenvolvimento)".
- **Frases de pré-requisito** (`MISSING_LABELS`). Os códigos são os do §7 e da Q14; a de `rnds_sending_disabled` é fixa (Q14, letra por letra), e as outras são **propostas**, no molde das da RNDS da vacina:

  | Código | Frase |
  |---|---|
  | `credential_missing:rnds_regulation` | "credencial RNDS da regulação não cadastrada (o administrador da cidade grava no dashboard)" |
  | `credential_unauthorized:rnds_regulation` | "credencial RNDS da regulação recusada no último teste" |
  | `care_regulation_disabled` | "regulação municipal (care_regulation) desligada ou incompleta" |
  | `rnds_sending_disabled` | "ligue antes um dos envios à RNDS (vacina ou regulação)" |

  As que já existem não mudam: `ibge_code_missing`, `uf_missing`, `rnds_immunization_disabled` (ainda chega de um api sem o 26c), `credential_missing:rnds`, `credential_unauthorized:rnds`.
- **Aviso do simulado** (Q14: cita só os envios **ligados**; textos novos são proposta). `rndsMockNotice(features)` devolve `null` sem `rnds_mock` ligado; com ele, pelo estado ligado de `rnds_immunization` e `rnds_regulation`:

  | Ligados | Texto |
  |---|---|
  | só vacinas | `RNDS_MOCK_NOTICE`, o de hoje, sem mudança: "Esta cidade envia as vacinas à RNDS SIMULADA — as doses recebem aceite ou recusa de exemplo e não chegam ao Ministério da Saúde nem ao Meu SUS Digital (fora de produção)." |
  | só regulação | `RNDS_MOCK_NOTICE_REGULATION` = "Esta cidade envia a regulação à RNDS SIMULADA — os pedidos recebem aceite ou recusa de exemplo e não chegam ao Ministério da Saúde (fora de produção)." |
  | os dois | `RNDS_MOCK_NOTICE_BOTH` = "Esta cidade envia as vacinas e a regulação à RNDS SIMULADA — as doses e os pedidos recebem aceite ou recusa de exemplo e não chegam ao Ministério da Saúde nem ao Meu SUS Digital (fora de produção)." |
  | nenhum | `RNDS_MOCK_NOTICE_IDLE` = "A RNDS SIMULADA está ligada, mas nenhum envio à RNDS está ligado — nada é enviado (fora de produção)." |

  O `aria-label` "modo da RNDS" não muda. O estado é o **ligado**, não o utilizável, como hoje: o aviso existe para que o simulado nunca passe despercebido.
- **Sem cascata:** ligar ou desligar manda só aquela `key`. Desligar `rnds_regulation` não desliga `rnds_mock`: ele fica "ligada, falta pré-requisito" se o envio das vacinas também estiver desligado, e o aviso muda para o da linha "nenhum".
- **O que passa aqui:** só o estado dos interruptores. Nenhum certificado, senha, identificador do solicitante, CNS, pedido ou dado de paciente passa pelo maintenance.
- **Frases da tela:** toda frase que o contrato não fixa é **proposta** e está marcada assim. Código desconhecido aparece cru, nunca `undefined`/`null`.
- **Interface:** nada de componente novo; cores só por papéis do tema, com `src/theme/*.test.ts` na suíte; 375 px sem rolagem lateral da página.
- **Testes:** só `bin/test maintenance -w maintenance-mod26c [arquivos]` (de `$R`), nunca `vitest` direto. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge e push só com autorização explícita.
- **Board #1:** o maintenance é superfície do card do 26c do interruptor (criado pelo plano do api). O card vai a In Progress pelo plano do api; a Task 0 só confere.
- **Execução coordenada:** ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod26c`, `A=$R/apps/api/.claude/mod26c`.
- Worktree **fora de `apps/`**: `$M`, na branch `feat/mod-26c-mira` a partir de `origin/main`. Os caminhos das tasks são relativos a `$M`.
- **Prova:**
  - Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3063**;
  - abra `http://maintenance.localhost:5177`, nunca `localhost:5177`;
  - conta `dev@local` / `dev-password`, com TOTP fixo (`lib/dev_maintainer.rb`); o usuário faz login e TOTP.

## Review Focus

1. **Desligar `rnds_regulation` com o `rnds_mock` ligado e o envio das vacinas desligado.** Espera-se que só `rnds_regulation` vá na mutation. Depois do refetch, `rnds_mock` diz "ligada, falta pré-requisito", com a frase de `rnds_sending_disabled`, e o aviso passa ao `RNDS_MOCK_NOTICE_IDLE`. Teste: Task 1, "desligar a regulação deixa o simulado ligado, com o motivo e o aviso de nada enviado".
2. **O simulado ligado só com a regulação, ou com os dois.** Espera-se o aviso que cita só o que está ligado: a regulação sozinha não fala em vacinas nem no Meu SUS Digital; os dois juntos citam os dois. Teste: Task 1, "simulado: o aviso cita só os envios ligados".
3. **Ligar `rnds_regulation` sem credencial e sem UF.** Espera-se "ligada, falta pré-requisito", com as duas frases unidas por "; ", na ordem do api, e nada de "ligada e utilizável". Teste: Task 1, "ligar a regulação sem credencial e sem UF: as duas frases".
4. **Api sem o 26c** (o `rnds_mock` ainda com `rnds_immunization_disabled`, sem a chave nova). Espera-se a frase antiga, nenhuma linha `rnds_regulation`, e o aviso pelos envios ligados (o das vacinas, se o `rnds_immunization` estiver ligado). Teste: Task 1, "api sem o 26c: a frase antiga e nenhuma chave nova".
5. **Api mais novo que a tela** (um `missing` futuro na linha da regulação). Espera-se o código cru. Teste: Task 1, "missing desconhecido da regulação aparece cru".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/rnds.ts`, `src/lib/rnds.test.ts`, `src/screens/FeaturesTab.tsx` (o aviso), `src/screens/FeaturesTab.test.tsx` | nome, frases de pré-requisito, aviso do simulado pelos envios ligados | 1 |
| `README.md` | parágrafo do módulo e o do 29 ajustado; suíte, codegen:check, build, revisão, prova na 5177, parada | 2 |

**Estratégia de teste:**
- **Funções puras** (`featureLabel`, `missingText`, `missingSummary`, `rndsMockOn`, `rndsMockNotice`), por tabela de casos.
- **A aba**, pelo fluxo real, com o `fetch` falso que responde por nome de operação: os helpers do topo de `FeaturesTab.test.tsx` (`reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`) e um `mockWithMutation` local, copiado do bloco do 35a. O teste confere as variáveis exatas da mutation, o estado depois do refetch, a frase de cada motivo e o aviso.
- **A Task 2:** a suíte inteira com as guardas de tema, o `codegen:check`, o build e a prova contra o api do 26c em Curitiba.

---

### Task 0: Card, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (com o 27b); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); `$A`.
- Produces: `$M`; a base anotada; a `description` real de `rnds_regulation` e o `missing` real do `rnds_mock`.

- [ ] **Step 1: Conferir o card do 26c.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-26\\.([89]|[1-9][0-9]) ")) | {id, title, status}'
```

Expected: o card do interruptor em In Progress ou além. Em Todo, mova com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod26c
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance log --oneline origin/main | grep -ci bnafar
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'rnds_mock:\|"credential_unauthorized:rnds"\|care_regulation_disabled\|rnds_regulation\|uf_missing'
$G -C $R/apps/maintenance show origin/main:src/lib/rnds.ts | grep -n 'RNDS_MOCK_NOTICE ='
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n 'Módulo 29: aviso pelo estado LIGADO de rnds_mock'
$G -C $R/apps/maintenance show origin/main:README.md | grep -n '^\*\*Sala de vacina (módulo 29).\*\*\|^\*\*Envio à BNAFAR (módulo 27b).\*\*'
grep -n 'key: "rnds_regulation"\|key: "rnds_mock"' -A4 $A/app/services/platform/features.rb
grep -n 'rnds_sending' $A/app/services/platform/features.rb
```

Expected:
- o 27b na `main` (contagem > 0). Se for 0, **pare**: o contrato manda começar depois dele;
- em `features.ts`: `rnds_mock` e `credential_unauthorized:rnds` presentes; **nenhuma** linha com `care_regulation_disabled` nem `rnds_regulation`;
- no api: `rnds_regulation` no fim do `CATALOG`, com os `requires` do §1; o `rnds_mock` com `requires: %w[rnds_sending]` e o motivo `rnds_sending_disabled` (Q14).

**Anote** a `description` real de `rnds_regulation` e de `rnds_mock`. **Onde o api diz outra coisa, vale o api**: a Task 1 ajusta as constantes dos testes, e o relato ao coordenador cita a diferença. Sem a branch do api ainda, siga pelo contrato e repita este passo antes da prova.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod26c -b feat/mod-26c-mira origin/main
cd $R/.worktrees/maintenance-mod26c && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod26c
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos).

**Relato ao coordenador:** "maintenance-mod26c Task 0: worktree e base verdes (N arquivos/M casos); rnds_regulation e rnds_sending_disabled no api <conferidos|ausentes>; diferenças: <…>."

---

### Task 1: Nome do interruptor, frases de pré-requisito e aviso do simulado

**Files:**
- Modify:
  - `src/lib/features.ts`: uma entrada em `FEATURE_LABELS`, depois da última; quatro em `MISSING_LABELS`, junto das da RNDS do 29;
  - `src/lib/rnds.ts`: os três textos novos, `rndsMockNotice` e o comentário do topo;
  - `src/screens/FeaturesTab.tsx`: o aviso sai de `rndsMockNotice(city.features)`, no lugar de `rndsMockOn` + `RNDS_MOCK_NOTICE`, com o comentário "Módulos 29 e 26c (Q14)".
- Test: `src/lib/features.test.ts`, `src/lib/rnds.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary`, `featureStatus`, `doneText` (`features.ts`); `RNDS_MOCK_KEY`, `rndsMockOn` (`rnds.ts`); os helpers do topo de `FeaturesTab.test.tsx`.
- Produces:
  - o rótulo e as quatro frases das Global Constraints;
  - `rndsMockNotice(features: { key: string; enabled: boolean }[]): string | null` e os textos `RNDS_MOCK_NOTICE_REGULATION`, `RNDS_MOCK_NOTICE_BOTH` e `RNDS_MOCK_NOTICE_IDLE`; `RNDS_MOCK_NOTICE`, `RNDS_MOCK_KEY` e `rndsMockOn` sem mudança;
  - no `FeaturesTab`, o mesmo `<p role="note" aria-label="modo da RNDS">`, com o texto de `rndsMockNotice`, e nada quando ela devolve `null`.

- [ ] **Step 1: Write the failing tests**

  No fim de `src/lib/features.test.ts`, um `describe("módulo 26c")` com:
  - `featureLabel("rnds_regulation")` = "Envio da regulação à RNDS";
  - as quatro `missingText`, com as frases;
  - `missingSummary([ "credential_missing:rnds_regulation", "uf_missing" ])` = as duas frases unidas por "; ";
  - um código futuro (`regulation_settings_missing`), que volta cru;
  - os do 29 sem mudança: `featureLabel("rnds_mock")`, `missingText("rnds_immunization_disabled")` e `missingText("credential_missing:rnds")`.

  Em `src/lib/rnds.test.ts`, os casos de hoje não mudam (o `RNDS_MOCK_NOTICE` é o mesmo), e um `it.each` de `rndsMockNotice` com as quatro linhas da tabela das Global Constraints, mais `null` sem `rnds_mock` ligado e `null` com `rnds_mock` ausente (produção). O texto só da regulação não contém "vacina" nem "Meu SUS".

  No fim de `src/screens/FeaturesTab.test.tsx`, um `describe("FeaturesTab — envio da regulação à RNDS (módulo 26c)")`, com `mockWithMutation(initial, after, requested)` copiado do bloco do 35a e as constantes (as `description` anotadas na Task 0):
  - `RG_ON` (`care_regulation`, ligada e utilizável);
  - `RR_OFF`, `RR_ON` e `RR_BLOCKED` (`missing: [ "credential_missing:rnds_regulation", "uf_missing" ]`), da chave `rnds_regulation`;
  - `VAC_OFF` (`rnds_immunization` desligada);
  - `VAC_ON` (`rnds_immunization` ligada);
  - `MOCK_ON` e `MOCK_BLOCKED` (`missing: [ "rnds_sending_disabled" ]`).

| Caso | Lista inicial → depois da mutation | Clique | Asserções |
|---|---|---|---|
| "aparece com o nome em português e a descrição do api" | `[RG_ON, RR_OFF]` | — | "rnds_regulation", "Envio da regulação à RNDS" e a descrição; corpo sem `/undefined\|null/` |
| "ligar a regulação sem credencial e sem UF: as duas frases" | `[RG_ON, RR_OFF]` → `[RG_ON, RR_BLOCKED]` | "ligar" da linha, "confirmar: ligar rnds_regulation" | variáveis `{ citySlug: "sp", key: "rnds_regulation", enabled: true }`; "ligada, falta pré-requisito"; o texto "credencial RNDS da regulação não cadastrada (o administrador da cidade grava no dashboard); UF não preenchida"; nada de "ligada e utilizável" |
| "sem a regulação municipal: a frase do 26a" | `[{ ...RR_ON, usable: false, missing: [ "care_regulation_disabled" ] }]` | — | "regulação municipal (care_regulation) desligada ou incompleta" |
| "desligar a regulação deixa o simulado ligado, com o motivo e o aviso de nada enviado" | `[VAC_OFF, RR_ON, MOCK_ON]` → `[VAC_OFF, { ...RR_OFF, changedAt, changedBy }, MOCK_BLOCKED]` | "desligar" da linha `rnds_regulation`, "confirmar: desligar rnds_regulation" | 1 mutation, só com `rnds_regulation`; `rnds_mock` em "ligada, falta pré-requisito", com "ligue antes um dos envios à RNDS (vacina ou regulação)"; o `note` "modo da RNDS" com `RNDS_MOCK_NOTICE_IDLE`; nada fala em apagar (`/apaga\|exclu/i`) |
| "simulado: o aviso cita só os envios ligados" | `[VAC_OFF, RR_ON, MOCK_ON]`; depois, num segundo render, `[VAC_ON, RR_ON, MOCK_ON]` | — | primeiro `RNDS_MOCK_NOTICE_REGULATION`, sem "vacina"; depois `RNDS_MOCK_NOTICE_BOTH` |
| "api sem o 26c: a frase antiga e nenhuma chave nova" | `[VAC_OFF, { ...MOCK_ON, usable: false, missing: [ "rnds_immunization_disabled" ] }]` | — | "envio das vacinas à RNDS (rnds_immunization) desligado"; `queryByText("rnds_regulation")` nulo; o aviso `RNDS_MOCK_NOTICE_IDLE` |
| "missing desconhecido da regulação aparece cru" | `[{ ...RR_ON, usable: false, missing: [ "rira_profile_missing" ] }]` | — | "rira_profile_missing" na linha |
| "produção: sem rnds_mock, a regulação aparece e nenhum aviso" | `[RG_ON, RR_ON]` | — | "rnds_regulation" presente; `queryByRole("note", { name: "modo da RNDS" })` nulo; corpo sem `/SIMULAD/` |

  No bloco do 29 ("FeaturesTab — RNDS simulada (módulo 29)"):
  - "ligar rnds_mock por dois cliques" (com `SEND_ON`) continua esperando `RNDS_MOCK_NOTICE`, sem mudança;
  - "rnds_mock ligado sem o envio: o aviso aparece" passa a esperar `RNDS_MOCK_NOTICE_IDLE`, porque nenhum envio está ligado (Q14). Mude só a constante esperada e anote no commit.

- [ ] **Step 2: Run tests to verify they fail** — `bin/test maintenance -w maintenance-mod26c src/lib/features.test.ts src/lib/rnds.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos com o rótulo, as frases e o texto novo do aviso; os casos antigos passam.
- [ ] **Step 3: Implement.** As entradas levam um comentário de uma linha: "Módulo 26c: rnds_regulation exige care_regulation, a credencial rnds_regulation (gravada no dashboard), IBGE e UF; rnds_mock exige um dos dois envios (rnds_sending_disabled, Q14), e o aviso cita só os envios ligados."
- [ ] **Step 4: Run tests to verify they pass** — o mesmo comando, e `cd $M && npx tsc --noEmit`. Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/lib/rnds.ts src/lib/rnds.test.ts src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx
$G commit -m "feat: name the regulation RNDS switch, its prerequisites and the simulated mode for both sends" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod26c Task 1: rótulo, 4 frases e aviso pelos envios ligados; FeaturesTab verde; diferenças do api: <…>."

---

### Task 2: README, suíte, build, revisão e prova no navegador

**Files:** `README.md`; `$R/.claude/launch.json` (entrada nova; fora de git).

- [ ] **Step 1: README.**
  - No parágrafo **Sala de vacina (módulo 29).**: a frase do `rnds_mock` passa a dizer que ele vale para os dois envios e que exige um deles ligado.
  - Um parágrafo novo, **Envio da regulação à RNDS (módulo 26c).**, depois do último parágrafo de módulo (o do 27b, se for o último). Ele diz:
    - o interruptor e o que ele exige, com as frases da aba;
    - que a credencial `rnds_regulation` é da cidade, gravada no dashboard pelo `municipal_admin` (certificado, identificador do solicitante e CNS do responsável), e que nada dela passa aqui;
    - que, ao ligar, a carga inicial manda os pedidos abertos e os encerrados nos últimos 30 dias, no ritmo de 60 por minuto;
    - que desligar não apaga nada do que foi enviado; as linhas ficam calculadas, mas nada sai;
    - que o `rnds_mock` manda ao simulado os envios ligados, só fora de produção, e o aviso diz quais;
    - a ordem de deploy: api, depois dashboard e maintenance; contra um api sem o 26c, a chave só não aparece.

  Commit:

```bash
cd $M && $G add README.md
$G commit -m "docs: describe the regulation RNDS switch and the shared simulated mode in the maintenance README" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 2: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod26c && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base e **nenhum** arquivo de teste novo; casos novos em `features.test.ts`, `rnds.test.ts` e `FeaturesTab.test.tsx`;
- `codegen:check` limpo, e `schema.graphql` fora do diff;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 3: Varreduras**

```bash
cd $M
$G diff --stat origin/main..HEAD -- schema.graphql src/gql | grep -c . | grep -qx 0 && echo ok-sdl || echo "ERRO: SDL mudou sem motivo"
grep -rnE "pfx|requester_id|responsible_cns|password" src/lib/rnds.ts src/lib/features.ts && echo "ERRO: segredo no maintenance" || echo ok-segredo
grep -n 'rnds_regulation' src/lib/features.ts | grep -c . | grep -qx 1 && echo ok-rotulo || echo "revisar: rnds_regulation fora de FEATURE_LABELS"
```

Expected: `ok-sdl`, `ok-segredo`, `ok-rotulo`. (O comentário da Task 1 cita a chave; se a contagem der 2 por ele, confira à mão que a única entrada de tabela é a de `FEATURE_LABELS`.)

- [ ] **Step 4: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 11, §3.5) e o contrato (§1, §7). Ele confere:
  - o rótulo letra por letra;
  - as quatro frases e nenhuma frase antiga mudada;
  - nenhuma cascata no liga/desliga;
  - o aviso pelo estado ligado, citando só os envios ligados (Q14), e a frase de `rnds_sending_disabled` letra por letra;
  - nada de certificado, identificador, CNS ou dado de paciente;
  - um README sem promessa de apagar dado enviado.

- [ ] **Step 5: Prova no navegador na 5177 (com o usuário).** Depende do api do 26c na **3063**, com o banco de dev migrado e o `fake_rnds` na 8094, **com autorização do usuário** (plano do api). Acrescente ao `$R/.claude/launch.json`, depois da última entrada `maintenance-mod*`:

```json
{
  "name": "maintenance-mod26c",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/maintenance-mod26c && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3063 VITE_MAINTENANCE_ENV=development npx vite --port 5177 --host 0.0.0.0" ],
  "port": 5177
}
```

  Pare o serviço do compose (`docker compose -f $R/docker-compose.yml stop maintenance`) e suba com `preview_start` `maintenance-mod26c`. Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP. Antes de tudo, **anote** o estado de `care_regulation`, `rnds_immunization`, `rnds_regulation` e `rnds_mock` em Curitiba e Maringá. As screenshots vão em `evidencias/2026-11-25-modulo-26c/maintenance/`. Roteiro:
  1. **Funcionalidades (Curitiba):**
     - `rnds_regulation` aparece com nome e descrição;
     - ligar sem credencial → "ligada, falta pré-requisito", com a frase da credencial;
     - com `rnds_immunization` desligado, ligar `rnds_mock` → o aviso só da regulação; ligar `rnds_immunization` → o aviso dos dois.
  2. **Combinado com a sessão do dashboard (5224):** o `municipal_admin` grava a credencial `rnds_regulation`; ao recarregar a aba, `rnds_regulation` aparece "ligada e utilizável".
  3. **Sem cascata:** desligar `rnds_regulation` → `rnds_mock` fica "ligada, falta pré-requisito", com a frase de `rnds_sending_disabled`, e o aviso diz que nada é enviado. Religue.
  4. **A 375 px:** a aba não tem rolagem lateral da página, e o aviso quebra linha.

  Depois, volte as chaves ao estado anotado e limpe: `preview_stop` do `maintenance-mod26c` e `docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 6: Pare.** O merge só vem depois do api do 26c (a suíte completa do api é da sessão "Testes", pedida pelo plano do api) e só com autorização explícita, uma etapa de cada vez: merge, push e card.
  - Antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas).
  - Quando autorizado: `git fetch`, `merge --ff-only feat/mod-26c-mira`, `push origin main`, `worktree remove $M`, `branch -d`.
  - O card vai a **Done** só com as partes do api e do dashboard também entregues, e a **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod26c Task 2: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-25-modulo-26c/maintenance/); aguardando autorização de merge."

---

## Self-review

- **Cobertura** (contrato §7; spec decisão 11 e §3.5):
  - `FEATURE_LABELS.rnds_regulation`: Task 1;
  - `MISSING_LABELS` com `credential_missing:rnds_regulation`, `credential_unauthorized:rnds_regulation`, `care_regulation_disabled` (não existia) e `rnds_sending_disabled` (Q14): Task 1;
  - o aviso do `rnds_mock` pelos envios ligados (Q14): Task 1;
  - a documentação e a prova: Task 2.
- **Placeholders:** nenhum. Valores lidos na execução: os ids dos cards e as `description` reais (Task 0).
- **Tipos e nomes:** existem em `482e53b`: `featureLabel`, `missingText`, `missingSummary`, `featureStatus`, `doneText`, `RNDS_MOCK_KEY`, `RNDS_MOCK_NOTICE`, `rndsMockOn`, o `aria-label` "modo da RNDS" e os helpers de teste do `FeaturesTab.test.tsx` (`reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`; `mockWithMutation` no bloco do 35a).
- **Proporção:** a mudança de produção é uma entrada de rótulo, quatro frases, três textos de aviso, uma função pura e uma linha no `FeaturesTab`. O plano traz os testes em tabela.

## Divergências propostas ao contrato

As duas foram decididas no §10 do contrato (docs `5d49ef7`), e o plano já aplica a decisão. O texto de cada uma fica como registro.

- **DM1 — Resolvida pela Q14.** `rnds_disabled` não cabia na gramática de pré-requisitos do api, que só conhece `feature:X` → `X_disabled` (`apps/api/app/services/platform/features.rb:365-367`; o `rnds_mock` de hoje em `features.rb:167-170`). **Decisão:** o motivo é `rnds_sending_disabled`, com a frase "ligue antes um dos envios à RNDS (vacina ou regulação)"; o api cria o requisito alternativo `rnds_sending` no `missing_for` (plano do api, Valor 1). A frase antiga `rnds_immunization_disabled` fica, porque um api sem o 26c ainda a manda.
- **DM2 — Resolvida pela Q14.** O aviso de texto único citaria também o envio desligado. **Decisão:** o aviso cita só os envios ligados, por `rndsMockNotice` (Task 1), com o texto de hoje quando só as vacinas estão ligadas.

**Sem divergência, para registro:**
- `uf_missing` e `ibge_code_missing` já têm frase e não mudam.
- `care_regulation_disabled` não existia; entra pela Task 1, como o §7 previa ("se não existir").
- A credencial `rnds_regulation` não passa pelo maintenance (contrato §1).
- O SDL não muda.
