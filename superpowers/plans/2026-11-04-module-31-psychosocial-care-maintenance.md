# Módulo 31 — Atenção psicossocial (CAPS) (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 31** na branch `feat/mod-31-psychosocial-care` (worktree `apps/api/.claude/mod31`, servidor de dev na porta **3053**) com a entrada `psychosocial_care` em `Platform::Features::CATALOG` (`requires: %w[feature:clinical_record]`, contrato §1). Ordem (contrato §14): api → dashboard → **maintenance** → wpda. As Tasks 0 e 1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 depende.

**Goal:** Lado maintenance de F-31.1: o interruptor `psychosocial_care` aparece na aba Funcionalidades com nome em português, o pré-requisito (`clinical_record_disabled`) sai na frase que já existe, e o liga/desliga genérico fica fixado em teste para a chave nova, incluindo o caso de desligar o prontuário com o CAPS ligado.

**Architecture:** Conferido no código (`origin/main` do maintenance, `90237b1`). O liga/desliga é genérico: `setCityFeature`, três estados e `missing` vindo do api por `Platform::Features#missing_for`, que gera `<chave>_disabled` para `feature:<chave>`. **O catálogo é dado do api, mas o nome não é:** sem mudança aqui a chave aparece crua (a aba mostra o código, não quebra). O pré-requisito já está coberto, porque `clinical_record_disabled` tem frase desde o 19b. A mudança é **uma linha em `FEATURE_LABELS`** (`src/lib/features.ts`), um parágrafo no `README.md` e os testes. Nenhuma tela nova, nenhum SDL novo, nenhuma frase nova em `MISSING_LABELS` e **nenhum aviso de simulado**: o 31 não tem chave simulada nem credencial.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md` (Afeta, §2 decisão 24, §5, §7, §8 F-31.1), `docs/adr/0049.md` e o contrato `docs/superpowers/plans/2026-11-04-module-31-psychosocial-care-contracts.md` §1 e §14. Base: plano do maintenance do 30 (`2026-11-03-module-30-sanitary-transport-maintenance.md`). As diferenças estão em "Divergências propostas ao contrato", M1–M3.

## Global Constraints

- Interruptor (spec decisão 24; contrato §1): `psychosocial_care` exige `feature:clinical_record` ligado e utilizável. `missing` = `clinical_record_disabled`, com a frase do 19b, que **não é redefinida**. Cidade `integrated` ou `off` nunca tem o CAPS utilizável.
- Ligar ou desligar manda só aquela `key`. A aba nunca liga `clinical_record` junto, nem desliga `psychosocial_care` quando o prontuário é desligado.
- Nome (M1): `psychosocial_care` → "Atenção psicossocial (CAPS)".
- Desligar não apaga nada: matrículas, PTS, ações, RAAS, permanências e leitos ficam (a exclusão LGPD retém o prontuário do CAPS, ADR 0026). A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade. Nenhum paciente, CID, substância, PTS, CAPS, leito ou RAAS passa por aqui. Papéis (`psychosocial_staff`, `psychosocial_lead`), modalidade do CAPS, equipe e leitos são configurados no dashboard.
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova. As guardas de tema (`src/theme/*.test.ts`) rodam na suíte. Achado só visual não vira tarefa.
- Âncora de `FEATURE_LABELS`: a última entrada em `origin/main` na hora da execução (hoje `tfd`). Conflito no rebase com outro módulo é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-31.1..F-31.8 do board #1 em In Progress (Task 0). O maintenance é dono do F-31.1 junto com o api e o dashboard. O card vai para Done quando os três estiverem entregues, e para Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod31`, `A=$R/apps/api/.claude/mod31`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-31-psychosocial-care`. Caminhos das tasks relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod31 <arquivos>`. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado. Abra `http://maintenance.localhost:5177`, nunca `localhost:5177`. Conta `dev@local`; o usuário faz login e TOTP. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade em modo `integrated` ou `off`** (sem prontuário próprio). Ligar `psychosocial_care` dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado", nunca "utilizável". Teste: Task 1, "psychosocial_care sem prontuário: ligada, falta o prontuário, com a frase do 19b".
2. **Desligar o prontuário com o CAPS ligado.** Só `clinical_record` vai na mutation. Depois do refetch, a linha do CAPS diz "ligada, falta pré-requisito"; ela não é desligada junto. Teste: Task 1, "desligar clinical_record com psychosocial_care ligado: só a chave pedida; o CAPS fica faltando".
3. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `caps_unit_missing`). Aparece cru. Teste: Task 1, "módulo 31: nome; missing desconhecido cru".
4. **Texto que prometa apagar ao desligar.** Nenhuma frase da aba nem do README diz que desligar apaga matrículas, PTS ou RAAS. Testes: Task 1, os casos da aba (`not.toMatch(/apaga|exclu/i)`); Task 2, grep no README.
5. **Aviso de simulado indevido.** O 31 não tem chave simulada; com o CAPS ligado, a aba não mostra nota de simulado. Teste: Task 1, "psychosocial_care ligado: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entrada do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome da chave; fluxo genérico fixado para `psychosocial_care` | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e a aba pelo fluxo existente de dois cliques. O `fetch` falso responde por nome de operação, e os testes conferem as variáveis exatas da mutation, incluindo o refetch que mostra o dependente faltando (Task 1). A Task 2 roda a suíte inteira com as guardas de tema, o `codegen:check`, o build e a prova contra o api do 31.

---

### Task 0: Cards, âncoras, entrada do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`90237b1` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards em In Progress; `$M`; a base anotada; a `description` real de `psychosocial_care` anotada (usada nos testes).

- [ ] **Step 1: Cards F-31.1..F-31.8 para In Progress** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-31\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito em In Progress ou além. Card ausente → **pare e reporte**.

- [ ] **Step 2: Âncoras e entrada do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod31
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'clinical_record_disabled:\|tfd: "Tratamento\|psychosocial\|caps'
grep -n 'key: "psychosocial_care"' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`, `clinical_record_disabled` traduzido e nenhuma linha com `psychosocial`/`caps`;
- a última entrada de `FEATURE_LABELS` é `tfd` (ou a de um módulo que entrou depois — use-a como âncora);
- no api, a entrada com `requires: %w[feature:clinical_record]` e sem `environments` (existe em todos os ambientes).

Outro `requires`, um ambiente restrito ou uma segunda chave do 31 → **pare e reporte**. **Anote a `description`**: ela substitui o rascunho da Task 1.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod31 -b feat/mod-31-psychosocial-care origin/main
cd $R/.worktrees/maintenance-mod31 && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod31
```

Expected: verde e `base-ok`. **Anote** a base. A Task 2 espera o **mesmo número de arquivos de teste**, com casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: Nome e pré-requisito do interruptor `psychosocial_care`

**Files:**
- Modify: `src/lib/features.ts` (uma entrada em `FEATURE_LABELS`, depois de `tfd`), `README.md` (parágrafo depois do de Transporte sanitário e TFD)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("psychosocial_care") === "Atenção psicossocial (CAPS)"`.

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 31", () => {
  it("módulo 31: nome; missing desconhecido cru", () => {
    expect(featureLabel("psychosocial_care")).toBe("Atenção psicossocial (CAPS)");
    expect(missingText("caps_unit_missing")).toBe("caps_unit_missing");
  });

  it("psychosocial_care: o prontuário com a frase do 19b, sem redefinir", () => {
    expect(missingSummary([ "clinical_record_disabled" ]))
      .toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (a descrição é rascunho; use a anotada na Task 0):

```tsx
describe("FeaturesTab — atenção psicossocial (módulo 31)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const PSY_OFF: Feature = {
    key: "psychosocial_care",
    description: "Atenção psicossocial: CAPS, matrícula, PTS, ações, acolhimento noturno, matriciamento e RAAS do mês",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const PSY_ON: Feature = { ...PSY_OFF, enabled: true, usable: true, changedAt: "2026-11-04T13:00:00Z", changedBy: "dev@local" };
  const CR_ON: Feature = {
    key: "clinical_record", description: "Prontuário da atenção primária", enabled: true, usable: true,
    missing: [], changedAt: "2026-10-06T13:00:00Z", changedBy: "dev@local"
  };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ PSY_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("psychosocial_care")).not.toBeNull();
    expect(screen.getByText("Atenção psicossocial (CAPS)")).not.toBeNull();
    expect(screen.getByText(PSY_OFF.description)).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("psychosocial_care sem prontuário: ligada, falta o prontuário, com a frase do 19b", async () => {
    const user = userEvent.setup();
    const PSY_BLOCKED: Feature = { ...PSY_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ PSY_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ PSY_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: PSY_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar psychosocial_care" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "psychosocial_care", enabled: true });
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect((await screen.findByRole("status")).textContent).toBe(
      "psychosocial_care ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    );
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|apaga|exclu/i);
  });

  it("com o prontuário, liga utilizável e desliga com dois cliques; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ PSY_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? PSY_ON : { ...PSY_OFF, changedAt: "2026-11-04T14:00:00Z", changedBy: "dev@local" } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar psychosocial_care" }));
    expect((await screen.findByRole("status")).textContent).toBe("psychosocial_care ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar psychosocial_care" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables)
      .toEqual({ citySlug: "sp", key: "psychosocial_care", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("psychosocial_care desligada.");
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("desligar clinical_record com psychosocial_care ligado: só a chave pedida; o CAPS fica faltando", async () => {
    const user = userEvent.setup();
    const CR_OFF: Feature = { ...CR_ON, enabled: false, usable: false, changedAt: "2026-11-04T15:00:00Z" };
    const PSY_ORPHAN: Feature = { ...PSY_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ CR_ON, PSY_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ CR_OFF, PSY_ORPHAN ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: CR_OFF } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar clinical_record" }));
    await waitFor(() => expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull());
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "clinical_record", enabled: false });
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
  });

  it("psychosocial_care ligado: nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ CR_ON, PSY_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("psychosocial_care")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run** `bin/test maintenance -w maintenance-mod31 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos de nome (`featureLabel` devolve `null`; "Atenção psicossocial (CAPS)" não aparece). Os casos de pré-requisito e de desligar o prontuário já podem passar, porque fixam o mecanismo genérico.

- [ ] **Step 3: Implemente**
  - **`features.ts`:** depois de `tfd` em `FEATURE_LABELS`, o comentário `// Módulo 31 (ADR 0049): exige clinical_record (missing: clinical_record_disabled, já traduzido acima). Todos os ambientes, sem simulado.` e `psychosocial_care: "Atenção psicossocial (CAPS)"`. Nada muda em `MISSING_LABELS`.
  - **`README.md`:** parágrafo **Atenção psicossocial (módulo 31).** depois do de Transporte sanitário e TFD:
    - `psychosocial_care` ("Atenção psicossocial (CAPS)") liga no dashboard o CAPS: matrícula, PTS, ações, grupos, acolhimento noturno, matriciamento, RAAS do mês e o vínculo na tela da UBS;
    - exige `clinical_record`, então só vale em cidades em modo `record`;
    - desligar o prontuário deixa o CAPS como "ligada, falta pré-requisito", e a aba não desliga um pelo outro;
    - papéis, modalidade, equipe e leitos são configurados no dashboard; existe em todos os ambientes, sem simulado;
    - desligar esconde as telas, mas matrículas, PTS, ações e RAAS ficam (a exclusão LGPD retém o prontuário do CAPS);
    - nada do CAPS chega ao cidadão nem passa por aqui;
    - ordem de deploy: api → dashboard → maintenance; contra um api sem o 31, a chave só não aparece.

- [ ] **Step 4: Run** o mesmo comando e `cd $M && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the psychosocial care switch and pin its prerequisite

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run: `bin/test maintenance -w maintenance-mod31 && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão** — subagente revisor lê `origin/main..HEAD` contra a spec (decisão 24, §5, F-31.1) e o contrato §1. Ele confere:
  - só uma entrada nova em `FEATURE_LABELS`;
  - nenhuma frase redefinida para `clinical_record_disabled`;
  - nenhum aviso de simulado;
  - nenhum dado de cidade aqui;
  - que o README não promete apagar nada, nem remessa RAAS validada (gate do layout, spec §9).

  Confira também: `grep -n -i "apaga\|exclu" README.md | grep -i "psicossocial\|CAPS"`. Expected: sem saída.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 31 na **3053** (banco de dev migrado **com autorização do usuário**). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod31 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod31:/app -v maintenance-mod31-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3053 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod31   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP; anote antes o estado da chave. Guarde os screenshots em `evidencias/2026-11-04-modulo-31/maintenance/`. Roteiro:
1. Curitiba (`record`) → Funcionalidades: `psychosocial_care` com nome e descrição.
2. Ligar → "ligada e utilizável". No dashboard, o menu do CAPS aparece para quem tem `psychosocial_staff`.
3. Desligar `clinical_record` (só se o usuário autorizar; senão pule): o CAPS continua ligado e passa a "ligada, falta pré-requisito". As rotas `/psychosocial/*` respondem `feature_disabled`. Religue o prontuário.
4. Maringá (`integrated`): ligar `psychosocial_care` mostra "prontuário da atenção primária (clinical_record) desligado". Desligue de volta.
5. **Auditoria** lista as tentativas de `setCityFeature`.
6. A 375 px, sem rolagem lateral da página.

Depois, volte a chave ao estado anotado (ou ao que as provas do dashboard e do wpda precisarem) e limpe: `docker rm -f maintenance-mod31 && docker volume rm maintenance-mod31-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api e do dashboard do 31, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `git fetch`, `merge --ff-only feat/mod-31-psychosocial-care`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api e do dashboard do F-31.1 estiverem **Done**, mova o F-31.1 para **Done** (opção `98236657`), e para **Verified** se as provas passaram.

---

## Divergências propostas ao contrato

1. **M1 — Nome.** O contrato não fixa rótulo. Proposta: "Atenção psicossocial (CAPS)", igual no dashboard; a descrição é a do catálogo do api.
2. **M2 — "Pelo mecanismo genérico" ainda pede uma linha aqui.** A spec diz que o maintenance só usa o mecanismo genérico. Conferido: sem `FEATURE_LABELS`, a chave aparece crua. O pré-requisito não pede nada novo (`clinical_record_disabled` já tem frase). Proposta: o §1 registra que o maintenance traduz o nome, e só isso.
3. **M3 — Sem cascata.** O §1 fixa `requires: feature:clinical_record`, mas não diz o que acontece ao desligar o prontuário com o CAPS ligado. Proposta: nada em cascata. O CAPS continua ligado, fica "ligado, falta pré-requisito" (ADR 0032) e suas rotas respondem 403 `feature_disabled` até o prontuário voltar. O que já foi gravado fica. O §1 registra isso.

## Self-review

- **Cobertura (Afeta, decisão 24, §5, §7, F-31.1; contrato §1, §14):**
  - nome e pré-requisito: Task 1 (Review Focus 1 e 3; M1, M2);
  - sem cascata: Task 1, quarto caso da aba (Review Focus 2; M3);
  - desligar sem apagar: Task 1 (Review Focus 4) e o README;
  - sem simulado: Task 1 (Review Focus 5);
  - suíte, build e prova: Task 2;
  - cards: Tasks 0 e 2.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards e a `description` (Task 0, com a regra para diferença).
- **Tipos e nomes:** existem em `origin/main` (`90237b1`): `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus` (pelo "ligada, falta pré-requisito"), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e `Feature`.
- **Review Focus:** 1, 2, 4 e 5 — Task 1 (`FeaturesTab.test.tsx`); 3 — Task 1 (`features.test.ts`).
- **Proporção:** os blocos de código são só de testes; a mudança de produção é uma linha.
