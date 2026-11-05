# Módulo 32a — Urgência (UPA) (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 32a** na branch `feat/mod-32a-urgency` (worktree `apps/api/.claude/mod32a`, porta **3054**) com duas entradas em `Platform::Features::CATALOG` (contrato §1): `urgency_care` (`requires: %w[feature:clinical_record]`) e `waiting_room_display` (sem `requires`). Ordem (contrato §13): contracts → api → dashboard → wpda → **maintenance**. As Tasks 0 e 1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 depende.

**Goal:** Lado maintenance de F-32.1. Os interruptores `urgency_care` e `waiting_room_display` aparecem na aba Funcionalidades com nome em português. O pré-requisito da urgência (`clinical_record_disabled`) sai com a frase que já existe. O painel da TV liga sozinho, sem pré-requisito. O liga/desliga genérico fica fixado em teste para as duas chaves, incluindo desligar o prontuário com a urgência ligada.

**Architecture:** Conferido no código (`origin/main` do maintenance, `ec86846`). O liga/desliga é genérico: `setCityFeature`, três estados e `missing` vindo do api por `Platform::Features#missing_for`, que gera `<chave>_disabled` para `feature:<chave>`. O catálogo é dado do api, mas o nome não é: sem mudança aqui, a chave aparece crua. O pré-requisito já está coberto, porque `clinical_record_disabled` tem frase desde o 19b. A mudança é **duas linhas em `FEATURE_LABELS`** (`src/lib/features.ts`), um parágrafo no `README.md` e os testes. Nenhuma tela nova, nenhum SDL novo, nenhuma frase nova em `MISSING_LABELS` e **nenhum aviso de simulado**: o 32a não tem chave simulada nem credencial. O `urgency_stock_deduction` e a frase `urgency_care_disabled` são do 32b.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (Afeta "`apps/maintenance`", decisões 14 e 23, R6, §8 F-32.1); `docs/adr/0050.md`; o contrato `docs/superpowers/plans/2026-11-05-module-32a-urgency-contracts.md` §1 e §13. Molde: `docs/superpowers/plans/2026-11-04-module-31-psychosocial-care-maintenance.md`. As diferenças estão em "Divergências propostas ao contrato", M1–M3.

## Global Constraints

- Interruptores (spec decisão 23; contrato §1):
  - `urgency_care` exige `feature:clinical_record` ligado e utilizável. `missing` = `clinical_record_disabled`, com a frase do 19b, que **não é redefinida**. Cidade `integrated` ou `off` nunca tem a urgência utilizável;
  - `waiting_room_display` é **independente**: sem `requires`, utilizável em qualquer modo de prontuário, para UPA e UBS.
- Nomes (M1): `urgency_care` → "Urgência e emergência (UPA)"; `waiting_room_display` → "Painel de chamada na TV da sala de espera".
- Ligar ou desligar manda só aquela `key`. A aba nunca liga `clinical_record` junto, nunca desliga `urgency_care` quando o prontuário é desligado, e as duas chaves do 32a não se afetam.
- Desligar não apaga nada: encontros, classificações, desfechos, chamadas, TVs e locais de chamada ficam (o encontro com classificação ou consulta é retido, ADR 0026, contrato §13). A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade. Nenhum paciente, queixa, cor, CID, senha, TV ou token passa por aqui. Prefixo da senha, locais de chamada, TVs e o protocolo de urgência são configurados no dashboard.
- Valor desconhecido (api mais novo) aparece cru, nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova. As guardas de tema (`src/theme/*.test.ts`) rodam na suíte.
- Âncora de `FEATURE_LABELS`: a última entrada em `origin/main` na hora da execução (hoje `psychosocial_care`). Conflito no rebase com outro módulo é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover F-32.1..F-32.6 do board #1 para In Progress (Task 0). O maintenance é dono de F-32.1 junto com o api, o dashboard e o contracts. O card vai para Done quando os quatro estiverem entregues, e para Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod32a`, `A=$R/apps/api/.claude/mod32a`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-32a-urgency`. Caminhos das tasks relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod32a <arquivos>`. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado. Abra `http://maintenance.localhost:5177`, nunca `localhost:5177`. Conta `dev@local`; o usuário faz login e TOTP. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade em modo `integrated` ou `off`** (sem prontuário próprio). Ligar `urgency_care` dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado", nunca "utilizável". Teste: Task 1, "urgency_care sem prontuário: ligada, falta o prontuário, com a frase do 19b".
2. **Painel da TV numa cidade sem prontuário** (UBS de cidade `integrated`). `waiting_room_display` liga utilizável, sem pré-requisito. Teste: Task 1, "waiting_room_display em cidade integrated: liga utilizável, sem pré-requisito".
3. **Desligar o prontuário com a urgência ligada.** Só `clinical_record` vai na mutation. Depois do refetch, a linha da urgência diz "ligada, falta pré-requisito" e não é desligada junto. O painel da TV continua utilizável. Teste: Task 1, "desligar clinical_record com urgency_care ligado: só a chave pedida; a urgência fica faltando, a TV não muda".
4. **Texto que prometa apagar ao desligar** (encontros, classificações, TVs). Nenhuma frase da aba nem do README diz isso. Testes: Task 1, os casos da aba (`not.toMatch(/apaga|exclu/i)`); Task 2, grep no README.
5. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `upa_unit_missing`, ou a chave do 32b chegando antes). Aparece cru, e o aviso de simulado não aparece para nenhuma das duas chaves. Testes: Task 1, "módulo 32a: nomes; missing e chave desconhecidos crus" e "urgência e TV ligadas: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nomes das chaves; fluxo genérico fixado para `urgency_care` e `waiting_room_display` | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e a aba pelo fluxo existente de dois cliques. O `fetch` falso responde por nome de operação, e os testes conferem as variáveis exatas da mutation, inclusive o refetch que mostra o dependente faltando e o independente intacto (Task 1). A Task 2 roda a suíte inteira com as guardas de tema, o `codegen:check`, o build e a prova contra o api do 32a.

---

### Task 0: Cards, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`ec86846` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards em In Progress; `$M`; a base anotada; as `description` reais das duas chaves anotadas (usadas nos testes).

- [ ] **Step 1: Cards F-32.1..F-32.6 para In Progress** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-32\\.[1-6] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os seis em In Progress ou além. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod32a
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'clinical_record_disabled:\|psychosocial_care: "Atenção\|urgency\|waiting_room'
grep -n 'key: "urgency_care"\|key: "waiting_room_display"\|key: "urgency_stock_deduction"' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`, `clinical_record_disabled` traduzido e nenhuma linha com `urgency`/`waiting_room`;
- a última entrada de `FEATURE_LABELS` é `psychosocial_care` (ou a de um módulo que entrou depois; use-a como âncora);
- no api, `urgency_care` com `requires: %w[feature:clinical_record]` e `waiting_room_display` sem `requires`, as duas sem `environments`. Nenhum `urgency_stock_deduction` (é do 32b).

Outro `requires`, um ambiente restrito ou uma chave a mais: **pare e reporte**. **Anote as duas `description`**: elas substituem os rascunhos da Task 1.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod32a -b feat/mod-32a-urgency origin/main
cd $R/.worktrees/maintenance-mod32a && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod32a
```

Expected: verde e `base-ok`. **Anote** a base. A Task 2 espera o **mesmo número de arquivos de teste**, com casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: Nomes e pré-requisitos de `urgency_care` e `waiting_room_display`

**Files:**
- Modify: `src/lib/features.ts` (duas entradas em `FEATURE_LABELS`, depois de `psychosocial_care`), `README.md` (parágrafo depois do de Atenção psicossocial)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("urgency_care") === "Urgência e emergência (UPA)"` e `featureLabel("waiting_room_display") === "Painel de chamada na TV da sala de espera"`.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 32a", () => {
  it("módulo 32a: nomes; missing e chave desconhecidos crus", () => {
    expect(featureLabel("urgency_care")).toBe("Urgência e emergência (UPA)");
    expect(featureLabel("waiting_room_display")).toBe("Painel de chamada na TV da sala de espera");
    expect(missingText("upa_unit_missing")).toBe("upa_unit_missing");
  });

  it("urgency_care: o prontuário com a frase do 19b, sem redefinir", () => {
    expect(missingSummary([ "clinical_record_disabled" ]))
      .toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (as descrições são rascunho; use as anotadas na Task 0):

```tsx
describe("FeaturesTab — urgência e painel da TV (módulo 32a)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const URG_OFF: Feature = {
    key: "urgency_care",
    description: "Urgência na UPA: chegada, classificação de risco por cor, atendimento médico, desfecho e contrarreferência",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const URG_ON: Feature = { ...URG_OFF, enabled: true, usable: true, changedAt: "2026-11-05T13:00:00Z", changedBy: "dev@local" };
  const TV_OFF: Feature = {
    key: "waiting_room_display",
    description: "Senha, local da chamada e painel de TV da sala de espera (UPA e UBS)",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const TV_ON: Feature = { ...TV_OFF, enabled: true, usable: true, changedAt: "2026-11-05T13:05:00Z", changedBy: "dev@local" };
  const CR_ON: Feature = {
    key: "clinical_record", description: "Prontuário da atenção primária", enabled: true, usable: true,
    missing: [], changedAt: "2026-10-06T13:00:00Z", changedBy: "dev@local"
  };

  it("aparecem com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ URG_OFF, TV_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("urgency_care")).not.toBeNull();
    expect(screen.getByText("Urgência e emergência (UPA)")).not.toBeNull();
    expect(screen.getByText(URG_OFF.description)).not.toBeNull();
    expect(screen.getByText("waiting_room_display")).not.toBeNull();
    expect(screen.getByText("Painel de chamada na TV da sala de espera")).not.toBeNull();
    expect(screen.getByText(TV_OFF.description)).not.toBeNull();
    expect(screen.getAllByText("desligada")).toHaveLength(2);
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("urgency_care sem prontuário: ligada, falta o prontuário, com a frase do 19b", async () => {
    const user = userEvent.setup();
    const URG_BLOCKED: Feature = { ...URG_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ URG_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ URG_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: URG_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar urgency_care" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "urgency_care", enabled: true });
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect((await screen.findByRole("status")).textContent).toBe(
      "urgency_care ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    );
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|apaga|exclu/i);
  });

  it("waiting_room_display em cidade integrated: liga utilizável, sem pré-requisito", async () => {
    const user = userEvent.setup();
    let listed = [ TV_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ TV_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: TV_ON } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar waiting_room_display" }));
    expect((await screen.findByRole("status")).textContent).toBe("waiting_room_display ligada e utilizável.");
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "waiting_room_display", enabled: true });
    expect(screen.queryByText("ligada, falta pré-requisito")).toBeNull();
  });

  it("com o prontuário, liga a urgência utilizável e desliga com dois cliques; só a chave pedida; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ URG_OFF, TV_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        const urg = body.variables.enabled === true ? URG_ON : { ...URG_OFF, changedAt: "2026-11-05T14:00:00Z", changedBy: "dev@local" };
        listed = [ urg, TV_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: urg } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar urgency_care" }));
    expect((await screen.findByRole("status")).textContent).toBe("urgency_care ligada e utilizável.");
    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar urgency_care" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables)
      .toEqual({ citySlug: "sp", key: "urgency_care", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("urgency_care desligada.");
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(1);
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("desligar clinical_record com urgency_care ligado: só a chave pedida; a urgência fica faltando, a TV não muda", async () => {
    const user = userEvent.setup();
    const CR_OFF: Feature = { ...CR_ON, enabled: false, usable: false, changedAt: "2026-11-05T15:00:00Z" };
    const URG_ORPHAN: Feature = { ...URG_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ CR_ON, URG_ON, TV_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ CR_OFF, URG_ORPHAN, TV_ON ];
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
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(1);
  });

  it("urgência e TV ligadas: nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ CR_ON, URG_ON, TV_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("urgency_care")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run** `bin/test maintenance -w maintenance-mod32a src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos de nome (`featureLabel` devolve `null`, e os nomes em português não aparecem). Os casos de pré-requisito, de independência e de desligar o prontuário já podem passar, porque fixam o mecanismo genérico.

- [ ] **Step 3: Implemente**
  - **`features.ts`:** depois de `psychosocial_care` em `FEATURE_LABELS`, o comentário `// Módulo 32a (ADR 0050): urgency_care exige clinical_record (missing: clinical_record_disabled, já traduzido acima); waiting_room_display é independente (UPA e UBS). Todos os ambientes, sem simulado. urgency_stock_deduction é do 32b.`, e as duas entradas `urgency_care: "Urgência e emergência (UPA)"` e `waiting_room_display: "Painel de chamada na TV da sala de espera"`. Nada muda em `MISSING_LABELS`.
  - **`README.md`:** parágrafo **Urgência e painel da TV (módulo 32a).** depois do de Atenção psicossocial:
    - `urgency_care` ("Urgência e emergência (UPA)") liga no dashboard a recepção, a classificação de risco por cor, a fila, o atendimento médico, o desfecho e o painel do gestor nas unidades `upa`;
    - exige `clinical_record`, então só vale em cidades em modo `record`. Desligar o prontuário deixa a urgência como "ligada, falta pré-requisito", e a aba não desliga uma pela outra;
    - `waiting_room_display` ("Painel de chamada na TV da sala de espera") é independente e vale para UPA e UBS: senha na chegada, local na chamada e a TV pareada pelo dashboard;
    - protocolo de urgência, prefixo da senha, locais de chamada e TVs são configurados no dashboard; as duas chaves existem em todos os ambientes, sem simulado;
    - desligar esconde as telas, mas encontros, classificações, desfechos, chamadas e TVs ficam (a exclusão LGPD retém o encontro com classificação ou consulta);
    - ordem de deploy: api → dashboard → maintenance; contra um api sem o 32a, as chaves só não aparecem.

- [ ] **Step 4: Run** o mesmo comando e `cd $M && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the urgency care and waiting room display switches and pin their prerequisites

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run: `bin/test maintenance -w maintenance-mod32a && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 14 e 23, R6, F-32.1) e o contrato §1. Ele confere:
  - só duas entradas novas em `FEATURE_LABELS`;
  - nenhuma frase redefinida para `clinical_record_disabled`;
  - nenhum aviso de simulado;
  - nenhum dado de cidade aqui;
  - que o README não promete apagar nada.

  Confira também: `grep -n -i "apaga\|exclu" README.md | grep -i "urgência\|TV"`. Expected: sem saída.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 32a na **3054** (banco de dev migrado **com autorização do usuário**). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod32a --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod32a:/app -v maintenance-mod32a-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3054 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod32a   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP; anote antes o estado das duas chaves. Guarde os screenshots em `evidencias/2026-11-05-modulo-32a/maintenance/`. Roteiro:
1. Curitiba (`record`) → Funcionalidades: `urgency_care` e `waiting_room_display` com nome e descrição.
2. Ligar `urgency_care` → "ligada e utilizável". No dashboard, o menu da urgência aparece para quem é da UPA.
3. Ligar `waiting_room_display` → "ligada e utilizável". No dashboard, o cadastro de TV e de locais de chamada aparece para o `municipal_admin`.
4. Desligar `clinical_record` (só se o usuário autorizar; senão pule): a urgência continua ligada e passa a "ligada, falta pré-requisito"; a TV continua utilizável. As rotas `/urgency/*` respondem `feature_disabled`. Religue o prontuário.
5. Maringá (`integrated`): ligar `urgency_care` mostra "prontuário da atenção primária (clinical_record) desligado"; ligar `waiting_room_display` fica utilizável. Desligue as duas de volta.
6. **Auditoria** lista as tentativas de `setCityFeature`.
7. A 375 px, sem rolagem lateral da página.

Depois, volte as chaves ao estado anotado (ou ao que as provas do dashboard e do wpda precisarem) e limpe: `docker rm -f maintenance-mod32a && docker volume rm maintenance-mod32a-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api, do dashboard e do wpda do 32a, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `git fetch`, `merge --ff-only feat/mod-32a-urgency`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api, do dashboard e do contracts do F-32.1 estiverem **Done**, mova o F-32.1 para **Done** (opção `98236657`), e para **Verified** se as provas passaram.

---

## Divergências propostas ao contrato

1. **M1 — Nomes.** O contrato não fixa rótulos. Proposta: "Urgência e emergência (UPA)" e "Painel de chamada na TV da sala de espera", iguais no dashboard. As descrições são as do catálogo do api, e a de `waiting_room_display` deve dizer que vale para UPA e UBS (spec decisão 23).
2. **M2 — "Pelo mecanismo genérico" ainda pede duas linhas aqui.** A spec diz que o maintenance só usa o mecanismo genérico. Conferido: sem `FEATURE_LABELS`, as chaves aparecem cruas. O pré-requisito não pede nada novo (`clinical_record_disabled` já tem frase). Proposta: o §1 registra que o maintenance traduz os nomes, e só isso. A frase `urgency_care_disabled` entra no plano do maintenance do 32b, com o `urgency_stock_deduction`.
3. **M3 — Sem cascata, e o que fica preso.** O §1 fixa o `requires`, mas não diz o que acontece ao desligar. Proposta:
   - nada em cascata: desligar o prontuário deixa `urgency_care` em "ligada, falta pré-requisito", com as rotas em 403 `feature_disabled`;
   - desligar `waiting_room_display` para de gerar senha e de exigir `call_location_id`, e a TV recebe 403 `feature_disabled` (a tela da TV é do dashboard); as chamadas já gravadas ficam;
   - o §7 manda `Attendances::Close` numa UPA exigir o desfecho do 32 (422 `urgency_outcome_required`). Com `urgency_care` desligado, a rota do desfecho dá 403, e o atendimento aberto da UPA ficaria impossível de fechar. Proposta: a exigência vale só quando `urgency_care` está utilizável; desligado, o 13 fecha como antes. Isso fica num request spec no plano do api.

## Self-review

- **Cobertura (Afeta, decisões 14 e 23, R6, F-32.1; contrato §1, §13):**
  - nomes e pré-requisito da urgência: Task 1 (Review Focus 1 e 5; M1, M2);
  - TV independente: Task 1, terceiro caso da aba (Review Focus 2);
  - sem cascata: Task 1, quinto caso da aba (Review Focus 3; M3);
  - desligar sem apagar: Task 1 (Review Focus 4) e o README;
  - sem simulado: Task 1 (Review Focus 5);
  - suíte, build e prova: Task 2; cards: Tasks 0 e 2.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards e as `description` (Task 0, com a regra para diferença).
- **Tipos e nomes:** existem em `origin/main` (`ec86846`): `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus` (pelos três estados), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e `Feature`.
- **Review Focus:** 1 a 4 e a segunda metade do 5 — Task 1 (`FeaturesTab.test.tsx`); a primeira metade do 5 — Task 1 (`features.test.ts`).
- **Proporção:** os blocos de código são só de testes; a mudança de produção são duas linhas.
