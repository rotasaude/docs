# Módulo 33 — Ouvidoria do SUS municipal (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 33** na branch `feat/mod-33-ombudsman` (worktree `apps/api/.claude/mod33`, porta **3056**) com duas entradas novas no fim de `Platform::Features::CATALOG` (contrato §1): `ombudsman` (sem `requires`) e `ombudsman_anonymous_web` (`requires: %w[feature:ombudsman]`), as duas sem `environments`. Ordem de entrega (contrato §15): api → dashboard → wpda → **maintenance**. As Tasks 0 e 1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 depende.

**Goal:** Lado maintenance de F-33.1. Os interruptores `ombudsman` e `ombudsman_anonymous_web` aparecem na aba Funcionalidades com nome em português. O pré-requisito da denúncia anônima (`ombudsman_disabled`) sai com frase própria. O liga/desliga genérico fica fixado em teste para as duas chaves, incluindo desligar a ouvidoria com a denúncia anônima ligada.

**Architecture:** Conferido no código (`origin/main` do maintenance, `f2086a1`; api `3e96796d`). O liga/desliga é genérico: `setCityFeature`, três estados (`featureStatus`) e `missing` vindo do api. Em `ro-api/app/services/platform/features.rb:335-337`, `missing_for` gera `"#{dependency}_disabled"` para cada `feature:<chave>` cuja dependência não está utilizável; portanto o api do 33 devolve `ombudsman_disabled` para `ombudsman_anonymous_web` com a ouvidoria desligada, e esse código não existe em `MISSING_LABELS` (`src/lib/features.ts:24-78`), então sairia cru. O SDL não muda: `CityFeature` (`schema.graphql:145-153`) já carrega `key`, `description`, `enabled`, `usable`, `missing`, `changedAt`, `changedBy`, e `FeaturesTab.tsx` só renderiza o que vem dele, com `featureLabel` na coluna "Chave" (`FeaturesTab.tsx:193-200`). Os avisos de simulado (`role="note"`, `FeaturesTab.tsx:162-186`) são por chave própria (`signature_psc_mock`, `sncr_mock`, `fake_lab`, `fake_middleware`, `rnds_mock`) e não pegam as chaves novas. Nenhum teste do maintenance fixa a lista ou a ordem das chaves (a ordem é fixada só no api, `ro-api/spec/services/platform/features_spec.rb:119-128`, e é do plano do api atualizá-la). A mudança é **duas linhas em `FEATURE_LABELS`**, **uma em `MISSING_LABELS`**, um parágrafo no `README.md` e os testes. Nenhuma tela nova, nenhum SDL novo, nenhum `codegen` novo, nenhum aviso de simulado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-11-module-33-ombudsman-design.md` (Afeta "`apps/maintenance`", decisão 19, Exclusão LGPD, §F-33.1); `docs/adr/0051.md`; o contrato `docs/superpowers/plans/2026-11-11-module-33-ombudsman-contracts.md` §1, §14 (maintenance) e §15. Molde: `docs/superpowers/plans/2026-11-05-module-32a-urgency-maintenance.md` (e o 31, `2026-11-04-module-31-psychosocial-care-maintenance.md`). As diferenças estão em "Divergências propostas ao contrato", M1–M4.

## Global Constraints

- Interruptores (spec decisão 19; contrato §1):
  - `ombudsman` é independente: sem `requires`, utilizável em qualquer modo de prontuário (`off`, `integrated`, `record`). Descrição do api: "Ouvidoria do SUS municipal";
  - `ombudsman_anonymous_web` exige `feature:ombudsman` ligado e utilizável. `missing` = `ombudsman_disabled`. Descrição do api: "Denúncia anônima sem login no canal do cidadão". A Task 0 confere os textos reais.
- Nomes (M1): `ombudsman` → "Ouvidoria do SUS"; `ombudsman_anonymous_web` → "Denúncia anônima sem login (canal do cidadão)". Frase nova (contrato §14): `ombudsman_disabled` → "ouvidoria do SUS (ombudsman) desligada", no molde de "sala de vacina (vaccination) desligada" (`features.ts:64`). Os nomes **diferem** das descrições do api de propósito (M2).
- Ligar ou desligar manda só aquela `key`. A aba nunca liga `ombudsman` junto com a denúncia anônima, e nunca desliga a denúncia anônima quando a ouvidoria é desligada (sem cascata, como no 32a).
- Desligar não apaga nada: manifestações, eventos, encaminhamentos, anexos, trilha de identidade e pesquisas ficam (a manifestação é registro de obrigação legal e é **retida**, spec "Exclusão LGPD", ADR 0026). A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade. Nenhum protocolo, texto de manifestação, nome, CPF, contato, setor, respondente ou prazo passa por aqui. Setores, respondentes, assuntos, prazos e o teto diário da denúncia anônima (`anonymous_daily_cap`) são configurados no dashboard.
- Valor desconhecido (api mais novo) aparece cru, nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova. As guardas de tema (`src/theme/*.test.ts`) rodam na suíte.
- Âncoras: em `FEATURE_LABELS`, a última entrada em `origin/main` na hora da execução (hoje `urgency_stock_deduction`, `features.ts:143`); em `MISSING_LABELS`, logo antes do comentário de `city_unreachable` (hoje depois de `pharmacy_disabled`, `features.ts:74`). Conflito no rebase com outro módulo é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Board: o maintenance é dono de F-33.1 junto com o api e o dashboard (spec §F-33). Os cards F-33.x já estão em In Progress pelo plano do api; a Task 0 só confere. O F-33.1 vai para Done quando os três estiverem entregues, e para Verified depois da prova no navegador.
- Execução coordenada: ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod33`, `A=$R/apps/api/.claude/mod33`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-33-ombudsman`. Caminhos das tasks relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod33 <arquivos>`. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3056**. Abra `http://maintenance.localhost:5177`, nunca `localhost:5177`. Conta `dev@local`; o usuário faz login e TOTP. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Desligar a ouvidoria com a denúncia anônima ligada.** Só `ombudsman` vai na mutation. Depois do refetch, a linha da denúncia anônima diz "ligada, falta pré-requisito" com "ouvidoria do SUS (ombudsman) desligada" e não é desligada junto. Teste: Task 1, "desligar ombudsman com a denúncia anônima ligada: só a chave pedida; a denúncia fica faltando a ouvidoria".
2. **Ligar a denúncia anônima antes da ouvidoria.** Dá "ligada, falta pré-requisito", nunca "utilizável", e a aba não liga a ouvidoria junto. Teste: Task 1, "denúncia anônima sem a ouvidoria: ligada, falta a ouvidoria".
3. **Ouvidoria em cidade sem prontuário próprio** (`integrated`/`off`). `ombudsman` liga utilizável, sem pré-requisito. Teste: Task 1, "ombudsman em cidade integrated: liga utilizável, sem pré-requisito".
4. **Texto que prometa apagar ao desligar** (manifestações, anexos, trilha). Nenhuma frase da aba nem do README diz isso. Testes: Task 1, os casos da aba (`not.toMatch(/apaga|exclu/i)`); Task 2, grep no README.
5. **Api mais novo que a tela** (um `missing` futuro, como `ombudsman_settings_missing`). Aparece cru, e o aviso de simulado não aparece. Testes: Task 1, "módulo 33: nomes e frase; missing desconhecido cru" e "ouvidoria e denúncia ligadas: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nomes das duas chaves, frase de `ombudsman_disabled`, fluxo genérico fixado | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e a aba pelo fluxo existente de dois cliques. O `fetch` falso responde por nome de operação (helpers `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e o tipo `Feature`, no topo de `FeaturesTab.test.tsx:15-50`), e os testes conferem as variáveis exatas da mutation, inclusive o refetch que mostra o dependente faltando (Task 1). A Task 2 roda a suíte inteira com as guardas de tema, o `codegen:check` (prova de que o SDL não mudou), o build e a prova contra o api do 33.

---

### Task 0: Cards, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`f2086a1` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: `$M`; a base anotada; as `description` reais das duas chaves anotadas (usadas nos testes).

- [ ] **Step 1: Conferir o card F-33.1** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-33\\.1 ")) | {id, title, status}'
```

Expected: F-33.1 em In Progress ou além (movido pelo plano do api). Se estiver em Todo, mova:

```bash
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod33
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'pharmacy_disabled:\|city_unreachable:\|urgency_stock_deduction:\|ombudsman'
grep -n 'key: "ombudsman' -A3 $A/app/services/platform/features.rb
grep -n 'CITIZEN_HIDDEN =' $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`, nenhuma linha com `ombudsman`;
- a última entrada de `FEATURE_LABELS` é `urgency_stock_deduction` (ou a de um módulo que entrou depois; use-a como âncora); `pharmacy_disabled` logo antes de `city_unreachable`;
- no api, `ombudsman` sem `requires` e `ombudsman_anonymous_web` com `requires: %w[feature:ombudsman]`, as duas sem `environments` e no fim do `CATALOG`;
- `CITIZEN_HIDDEN` sem nenhuma das duas (contrato §1: as duas vão ao cidadão).

Outro `requires`, um ambiente restrito ou uma chave a mais: **pare e reporte**. **Anote as duas `description`**: elas substituem os rascunhos da Task 1. Se uma `description` do api for **igual** ao nome escolhido em M1, troque o nome (M2) antes de escrever os testes.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod33 -b feat/mod-33-ombudsman origin/main
cd $R/.worktrees/maintenance-mod33 && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod33
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos). A Task 2 espera o **mesmo número de arquivos de teste**, com casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

**Relato ao coordenador:** "maintenance-mod33: worktree e base verdes (N arquivos/M casos); entradas `ombudsman`/`ombudsman_anonymous_web` do api conferidas; F-33.1 em In Progress."

---

### Task 1: Nomes e pré-requisito de `ombudsman` e `ombudsman_anonymous_web`

**Files:**
- Modify: `src/lib/features.ts` (uma entrada em `MISSING_LABELS`, depois de `pharmacy_disabled`; duas em `FEATURE_LABELS`, depois de `urgency_stock_deduction`), `README.md` (parágrafo depois do de Baixa de estoque, `README.md:357-370`)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("ombudsman") === "Ouvidoria do SUS"`, `featureLabel("ombudsman_anonymous_web") === "Denúncia anônima sem login (canal do cidadão)"`, `missingText("ombudsman_disabled") === "ouvidoria do SUS (ombudsman) desligada"`.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 33", () => {
  it("módulo 33: nomes e frase; missing desconhecido cru", () => {
    expect(featureLabel("ombudsman")).toBe("Ouvidoria do SUS");
    expect(featureLabel("ombudsman_anonymous_web")).toBe("Denúncia anônima sem login (canal do cidadão)");
    expect(missingText("ombudsman_disabled")).toBe("ouvidoria do SUS (ombudsman) desligada");
    expect(missingText("ombudsman_settings_missing")).toBe("ombudsman_settings_missing");
  });

  it("ombudsman_anonymous_web: a ouvidoria como único pré-requisito", () => {
    expect(missingSummary([ "ombudsman_disabled" ])).toBe("ouvidoria do SUS (ombudsman) desligada");
  });

  it("os nomes do 32b não mudam", () => {
    expect(featureLabel("urgency_stock_deduction")).toBe("Baixa de estoque na medicação da UPA");
    expect(missingText("pharmacy_disabled")).toBe("farmácia e almoxarifado (pharmacy) desligada");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (as descrições são as do contrato §1; use as anotadas na Task 0):

```tsx
describe("FeaturesTab — ouvidoria do SUS (módulo 33)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const OMB_OFF: Feature = {
    key: "ombudsman",
    description: "Ouvidoria do SUS municipal",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const OMB_ON: Feature = { ...OMB_OFF, enabled: true, usable: true, changedAt: "2026-11-11T13:00:00Z", changedBy: "dev@local" };
  const ANON_OFF: Feature = {
    key: "ombudsman_anonymous_web",
    description: "Denúncia anônima sem login no canal do cidadão",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const ANON_ON: Feature = { ...ANON_OFF, enabled: true, usable: true, changedAt: "2026-11-11T13:05:00Z", changedBy: "dev@local" };

  it("aparecem com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ OMB_OFF, ANON_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("ombudsman")).not.toBeNull();
    expect(screen.getByText("Ouvidoria do SUS")).not.toBeNull();
    expect(screen.getByText(OMB_OFF.description)).not.toBeNull();
    expect(screen.getByText("ombudsman_anonymous_web")).not.toBeNull();
    expect(screen.getByText("Denúncia anônima sem login (canal do cidadão)")).not.toBeNull();
    expect(screen.getByText(ANON_OFF.description)).not.toBeNull();
    expect(screen.getAllByText("desligada")).toHaveLength(2);
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("ombudsman em cidade integrated: liga utilizável, sem pré-requisito", async () => {
    const user = userEvent.setup();
    let listed = [ OMB_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ OMB_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: OMB_ON } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar ombudsman" }));
    expect((await screen.findByRole("status")).textContent).toBe("ombudsman ligada e utilizável.");
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "ombudsman", enabled: true });
    expect(screen.queryByText("ligada, falta pré-requisito")).toBeNull();
  });

  it("denúncia anônima sem a ouvidoria: ligada, falta a ouvidoria", async () => {
    const user = userEvent.setup();
    const ANON_BLOCKED: Feature = { ...ANON_ON, usable: false, missing: [ "ombudsman_disabled" ] };
    let listed = [ OMB_OFF, ANON_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ OMB_OFF, ANON_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: ANON_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "ligar" }))[1]);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar ombudsman_anonymous_web" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "ombudsman_anonymous_web", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "ombudsman_anonymous_web ligada, mas ainda não utilizável — falta: ouvidoria do SUS (ombudsman) desligada."
    );
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|apaga|exclu/i);
  });

  it("desligar ombudsman com a denúncia anônima ligada: só a chave pedida; a denúncia fica faltando a ouvidoria", async () => {
    const user = userEvent.setup();
    const OMB_OFF_NOW: Feature = { ...OMB_OFF, changedAt: "2026-11-11T15:00:00Z", changedBy: "dev@local" };
    const ANON_ORPHAN: Feature = { ...ANON_ON, usable: false, missing: [ "ombudsman_disabled" ] };
    let listed = [ OMB_ON, ANON_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ OMB_OFF_NOW, ANON_ORPHAN ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: OMB_OFF_NOW } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar ombudsman" }));
    await waitFor(() => expect(screen.getByText("ouvidoria do SUS (ombudsman) desligada")).not.toBeNull());
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "ombudsman", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("ombudsman desligada.");
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("ouvidoria e denúncia ligadas: nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ OMB_ON, ANON_ON ], { recordMode: "off" })));
    renderTab();

    expect(await screen.findByText("ombudsman_anonymous_web")).not.toBeNull();
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(2);
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run** `bin/test maintenance -w maintenance-mod33 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos de nome (`featureLabel` devolve `null`) e de frase (`ombudsman_disabled` sai cru). Os casos de independência e de "só a chave pedida" já podem passar, porque fixam o mecanismo genérico.

- [ ] **Step 3: Implemente**
  - **`features.ts`, `MISSING_LABELS`:** depois de `pharmacy_disabled`, o comentário `// Módulo 33 (ADR 0051): ombudsman_anonymous_web exige ombudsman ligado e utilizável.` e a entrada `ombudsman_disabled: "ouvidoria do SUS (ombudsman) desligada"`.
  - **`features.ts`, `FEATURE_LABELS`:** depois de `urgency_stock_deduction`, o comentário `// Módulo 33 (ADR 0051): ombudsman sem pré-requisito (nem o modo prontuário); ombudsman_anonymous_web exige ombudsman (missing: ombudsman_disabled, acima). Todos os ambientes, sem simulado.` e as entradas `ombudsman: "Ouvidoria do SUS"` e `ombudsman_anonymous_web: "Denúncia anônima sem login (canal do cidadão)"`.
  - **`README.md`:** parágrafo **Ouvidoria do SUS (módulo 33).** depois do de Baixa de estoque:
    - `ombudsman` ("Ouvidoria do SUS") liga no dashboard a caixa da ouvidoria, o registro pela equipe, o encaminhamento a setores e unidades, os prazos, o painel e o relatório anual, e no wpda a manifestação identificada ou com identidade preservada e "Minhas manifestações";
    - não tem pré-requisito: vale em qualquer modo de prontuário;
    - `ombudsman_anonymous_web` ("Denúncia anônima sem login (canal do cidadão)") liga o formulário público sem login do wpda; exige `ombudsman`. Desligar a ouvidoria deixa a denúncia como "ligada, falta pré-requisito", e a aba não desliga uma pela outra. Desligar só a denúncia anônima tira o formulário público sem mexer no resto da ouvidoria;
    - setores, respondentes, assuntos, prazos e o teto diário da denúncia anônima são configurados no dashboard; as duas chaves existem em todos os ambientes, sem simulado, e as duas chegam ao cidadão;
    - desligar esconde as telas, mas manifestações, encaminhamentos, anexos e a trilha de acesso à identidade ficam (a manifestação é retida por obrigação legal, ADR 0026);
    - ordem de deploy: api → dashboard → wpda → maintenance; contra um api sem o 33, as chaves só não aparecem.

- [ ] **Step 4: Run** o mesmo comando e `cd $M && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the ombudsman and anonymous web report switches and their prerequisite

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod33 Task 1: nomes de `ombudsman`/`ombudsman_anonymous_web` e frase de `ombudsman_disabled` commitados (<sha>); escopo fechado verde."

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run: `bin/test maintenance -w maintenance-mod33 && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base;
- `codegen:check` limpo (o SDL não mudou);
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 19, Exclusão LGPD, F-33.1) e o contrato §1 e §14. Ele confere:
  - só duas entradas novas em `FEATURE_LABELS` e uma em `MISSING_LABELS`;
  - nomes diferentes das descrições do api (M2);
  - nenhum aviso de simulado;
  - nenhum dado de cidade aqui;
  - que o README não promete apagar nada.

  Confira também: `grep -n -i "apaga\|exclu" README.md | grep -i "ouvidoria\|manifesta\|denúncia"`. Expected: sem saída.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 33 na **3056** (banco de dev migrado **com autorização do usuário**). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod33 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod33:/app -v maintenance-mod33-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3056 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod33   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP; anote antes o estado das duas chaves. Guarde os screenshots em `evidencias/2026-11-11-modulo-33/maintenance/`. Roteiro:
1. Curitiba → Funcionalidades: `ombudsman` e `ombudsman_anonymous_web` com nome e descrição.
2. Ligar `ombudsman_anonymous_web` com a ouvidoria desligada → "ligada, falta pré-requisito" com "ouvidoria do SUS (ombudsman) desligada".
3. Ligar `ombudsman` → as duas "ligada e utilizável". No dashboard, a área "Ouvidoria" aparece para quem tem o papel; no wpda, a entrada "Denúncia anônima" aparece.
4. Desligar `ombudsman_anonymous_web` → no wpda, "Denúncia anônima" some e "Fazer uma manifestação" continua. Religue.
5. Desligar `ombudsman` (só se o usuário autorizar; senão pule): a denúncia continua ligada e passa a "ligada, falta pré-requisito"; as rotas do módulo respondem `feature_disabled`; as manifestações já gravadas continuam no banco (o dashboard volta a mostrá-las ao religar). Religue.
6. Maringá (`integrated`): ligar `ombudsman` fica utilizável, sem pré-requisito. Desligue de volta.
7. **Auditoria** lista as tentativas de `setCityFeature`, só com a chave e o estado.
8. A 375 px, sem rolagem lateral da página.

Depois, volte as chaves ao estado anotado (ou ao que as provas do dashboard e do wpda precisarem) e limpe: `docker rm -f maintenance-mod33 && docker volume rm maintenance-mod33-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api, do dashboard e do wpda do 33 (a suíte completa do api é da sessão "Testes", pedida pelo plano do api), e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas). Quando autorizado: `git fetch`, `merge --ff-only feat/mod-33-ombudsman`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api e do dashboard do F-33.1 estiverem **Done**, mova o F-33.1 para **Done** (opção `98236657`), e para **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance-mod33 Task 2: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-11-modulo-33/maintenance/); aguardando autorização de merge."

---

## Divergências propostas ao contrato

1. **M1 — Nomes.** O §14 fixa onde ficam os rótulos (`src/lib/features.ts`), mas não o texto. Proposta: `ombudsman` → "Ouvidoria do SUS"; `ombudsman_anonymous_web` → "Denúncia anônima sem login (canal do cidadão)"; `ombudsman_disabled` → "ouvidoria do SUS (ombudsman) desligada" (molde de `features.ts:64`, `:73-74`). O dashboard, que também tem `FEATURE_LABELS` (`ro-dashboard/src/lib/features.ts:58` tem `psychosocial_care` com o mesmo nome do maintenance), usa os mesmos nomes se rotular as chaves.
2. **M2 — Nome igual à descrição.** As descrições do §1 ("Ouvidoria do SUS municipal", "Denúncia anônima sem login no canal do cidadão") são frases curtas, do tamanho de um nome. A aba mostra a chave, o nome (`FeaturesTab.tsx:193-200`) e a descrição em colunas; se o nome for igual à descrição, o texto aparece duas vezes na linha e os testes por `getByText` quebram. Proposta: o nome é sempre diferente da descrição (como em M1); se o api mudar a descrição para coincidir com o nome, a Task 0 troca o nome. Alternativa, se o coordenador preferir: o §1 alonga as descrições (por exemplo, "Ouvidoria do SUS municipal: manifestações, encaminhamento, prazos, pesquisa e relatório anual").
3. **M3 — 404 ou 403 em `feature_disabled`.** O §1 diz "Sem `ombudsman`, toda rota do módulo responde 404 `feature_disabled`, **como os outros gates**". Os outros gates respondem **403**: `ro-api/app/controllers/attendances_controller.rb:14`, `consultations_controller.rb:214`, `campaigns_controller.rb:17`, e no cidadão `citizen_api/exams_controller.rb:46` e `citizen_api/trips_controller.rb:28`. O maintenance não depende do código (o README não cita status), mas o roteiro da Task 2 (passo 5) confere `feature_disabled`. Proposta: o §1 decide entre 403 (como os outros) e 404 (para não revelar o módulo ao público, o que faria sentido só no `public_config` e no `anonymous_reports`), e tira "como os outros gates" se ficar 404. A decisão é do plano do api; o maintenance só segue.
4. **M4 — Sem cascata e o que fica preso.** O §1 fixa o `requires`, mas não diz o que acontece ao desligar a ouvidoria com a denúncia anônima ligada. Proposta, registrada no §1: nada em cascata. A denúncia fica "ligada, falta pré-requisito" (`ombudsman_disabled`), `public_config` e `anonymous_reports` dão `feature_disabled`, e religar a ouvidoria devolve a denúncia ao estado utilizável sem novo clique. Manifestações e trilhas ficam (retenção, spec "Exclusão LGPD"). Isso fica fixado no maintenance pela Task 1 (quarto caso da aba) e deve ter um spec de serviço no plano do api.

## Self-review

- **Cobertura (Afeta, decisão 19, Exclusão LGPD, F-33.1; contrato §1, §14, §15):**
  - nomes e frase do pré-requisito: Task 1 (Review Focus 5; M1, M2);
  - ouvidoria independente: Task 1, segundo caso da aba (Review Focus 3);
  - denúncia antes da ouvidoria: Task 1, terceiro caso (Review Focus 2);
  - sem cascata: Task 1, quarto caso (Review Focus 1; M4);
  - desligar sem apagar: Task 1 (Review Focus 4) e o README;
  - sem simulado: Task 1 (Review Focus 5);
  - GraphQL e `FeaturesTab`: conferidos, nada a mudar (`schema.graphql:145-153`; `codegen:check` na Task 2);
  - teste que fixa lista ou ordem de chaves: não existe no maintenance; a ordem é do api;
  - suíte, build e prova (Curitiba, Maringá, 375 px): Task 2; card: Tasks 0 e 2.
- **Placeholders:** nenhum. Valores lidos na execução: ids do card e as `description` (Task 0, com a regra para diferença).
- **Tipos e nomes:** existem em `origin/main` (`f2086a1`): `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus` (pelos três estados), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e `Feature`. No api (`3e96796d`): `Platform::Features::CATALOG`, `missing_for`, `usable?`, `CITIZEN_HIDDEN`.
- **Proporção:** os blocos de código são só de testes; a mudança de produção são três linhas e dois comentários.
