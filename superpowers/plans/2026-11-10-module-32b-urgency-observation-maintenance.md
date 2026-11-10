# Módulo 32b — Observação, prescrição interna e baixa de estoque na UPA (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 32b** na branch `feat/mod-32b-urgency-observation` (worktree `apps/api/.claude/mod32b`, porta **3055**) com a entrada nova no fim de `Platform::Features::CATALOG` (contrato §1): `urgency_stock_deduction`, `requires: %w[feature:urgency_care feature:pharmacy]`, sem `environments`. Ordem de entrega (contrato §11): api → dashboard → **maintenance**. As Tasks 0 e 1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 depende.

**Goal:** Lado maintenance de F-32.8. O interruptor `urgency_stock_deduction` aparece na aba Funcionalidades com nome em português. Os dois pré-requisitos que o api devolve para ele (`urgency_care_disabled` e `pharmacy_disabled`) saem com frase própria. O liga/desliga genérico fica fixado em teste para a chave nova, incluindo desligar a farmácia e o prontuário com a baixa ligada.

**Architecture:** Conferido no código (`origin/main` do maintenance, `bbff803`; api `69d283c4`). O liga/desliga é genérico: `setCityFeature`, três estados (`featureStatus`) e `missing` vindo do api. Em `app/services/platform/features.rb:329-331`, `missing_for` gera `"#{dependency}_disabled"` para cada `feature:<chave>` cuja dependência não está **utilizável**, na ordem do `requires`. Portanto o api do 32b devolve, para a chave nova, `urgency_care_disabled` e/ou `pharmacy_disabled`, e **só** esses: o pré-requisito do pré-requisito não se propaga (com o prontuário desligado, a baixa mostra `urgency_care_disabled`, nunca `clinical_record_disabled`). Nenhum dos dois códigos existe hoje em `MISSING_LABELS` (`src/lib/features.ts:24-75`), então eles apareceriam crus. A mudança é **uma linha em `FEATURE_LABELS`**, **duas em `MISSING_LABELS`**, um parágrafo no `README.md` e os testes. Nenhuma tela nova, nenhum SDL novo, nenhum aviso de simulado (a chave não é simulada nem tem credencial). A tela de faltas da UPA é do dashboard (grupo "Farmácia"), não daqui.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (Afeta "`apps/maintenance`", decisões 2, 22 e 23, R7, §8 F-32.8); `docs/adr/0050.md`; o contrato `docs/superpowers/plans/2026-11-10-module-32b-urgency-observation-contracts.md` §1, §6 (Estoque e Interruptor desligado), §7 e §11 (Maintenance). Molde: `docs/superpowers/plans/2026-11-05-module-32a-urgency-maintenance.md`. As diferenças estão em "Divergências propostas ao contrato", M1–M4.

## Global Constraints

- Interruptor (spec decisões 2, 22 e 23; contrato §1):
  - `urgency_stock_deduction` exige `feature:urgency_care` **e** `feature:pharmacy`, os dois ligados e utilizáveis. `missing` = `urgency_care_disabled`, `pharmacy_disabled`, ou os dois, nessa ordem;
  - cidade `integrated` ou `off` nunca tem a urgência utilizável, então nunca tem a baixa utilizável;
  - descrição do catálogo do api: "Atendimentos da UPA com prescrição de medicação atualizam o estoque" (contrato §1). A Task 0 confere o texto real.
- Nomes (contrato §11): `urgency_stock_deduction` → "Baixa de estoque na medicação da UPA". Frases novas (M2): `urgency_care_disabled` → "urgência e emergência (urgency_care) desligada"; `pharmacy_disabled` → "farmácia e almoxarifado (pharmacy) desligada". Seguem o molde das frases existentes ("sala de vacina (vaccination) desligada").
- Ligar ou desligar manda só aquela `key`. A aba nunca liga `urgency_care` ou `pharmacy` junto, e nunca desliga a baixa quando um pré-requisito é desligado (sem cascata, como no 32a).
- Desligar não apaga nada: administrações, movimentos `administration` já gravados e faltas da UPA ficam, e as faltas antigas continuam visíveis para a farmácia (contrato §7). Observação e prescrição funcionam igual com a baixa desligada (contrato §1). A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade. Nenhum paciente, medicamento, lote, leito, prescrição ou falta passa por aqui.
- Valor desconhecido (api mais novo) aparece cru, nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova. As guardas de tema (`src/theme/*.test.ts`) rodam na suíte.
- Âncoras: em `FEATURE_LABELS`, a última entrada em `origin/main` na hora da execução (hoje `waiting_room_display`, `features.ts:138`); em `MISSING_LABELS`, logo antes de `city_unreachable` (hoje depois de `sanitary_transport_disabled`, `features.ts:71`). Conflito no rebase com outro módulo é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Board: o maintenance é dono de F-32.8 junto com o api e o dashboard. Os cards F-32.7 e F-32.8 já estão em In Progress pelo plano do api; a Task 0 só confere. O F-32.8 vai para Done quando os três estiverem entregues, e para Verified depois da prova no navegador.
- Execução coordenada: ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod32b`, `A=$R/apps/api/.claude/mod32b`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-32b-urgency-observation`. Caminhos das tasks relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod32b <arquivos>`. Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3055**. Abra `http://maintenance.localhost:5177`, nunca `localhost:5177`. Conta `dev@local`; o usuário faz login e TOTP. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Farmácia desligada com a baixa ligada.** A linha da baixa diz "ligada, falta pré-requisito" com "farmácia e almoxarifado (pharmacy) desligada", e a aba não desliga uma pela outra. Teste: Task 1, "desligar pharmacy com a baixa ligada: só a chave pedida; a baixa fica faltando a farmácia".
2. **Cidade `integrated` (Maringá).** Ligar a baixa dá "ligada, falta pré-requisito" com a frase da urgência, nunca "utilizável", e nunca a frase do prontuário (o api não propaga). Teste: Task 1, "baixa em cidade integrated: ligada, falta a urgência (e a farmácia), sem a frase do prontuário".
3. **Os dois pré-requisitos faltando juntos.** Saem os dois, na ordem do `requires`, separados por "; ". Testes: Task 1, `missingSummary` em `features.test.ts` e o caso da aba de Maringá.
4. **Texto que prometa apagar ao desligar** (administrações, movimentos, faltas). Nenhuma frase da aba nem do README diz isso. Testes: Task 1, os casos da aba (`not.toMatch(/apaga|exclu/i)`); Task 2, grep no README.
5. **Api mais novo que a tela** (um `missing` futuro, como `upa_stock_location_missing`). Aparece cru, e o aviso de simulado não aparece. Testes: Task 1, "módulo 32b: nome e frases; missing desconhecido cru" e "baixa ligada: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entrada do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nome da chave, frases dos dois pré-requisitos, fluxo genérico fixado para `urgency_stock_deduction` | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e a aba pelo fluxo existente de dois cliques. O `fetch` falso responde por nome de operação (helpers `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e o tipo `Feature`, no topo de `FeaturesTab.test.tsx`), e os testes conferem as variáveis exatas da mutation, inclusive o refetch que mostra o dependente faltando (Task 1). A Task 2 roda a suíte inteira com as guardas de tema, o `codegen:check`, o build e a prova contra o api do 32b.

---

### Task 0: Cards, âncoras, entrada do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`bbff803` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: `$M`; a base anotada; a `description` real da chave nova e a ordem real do `requires` anotadas (usadas nos testes).

- [ ] **Step 1: Cards F-32.7 e F-32.8** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-32\\.[78] ")) | {id, title, status}'
```

Expected: os dois em In Progress ou além (o plano do api já moveu). Se algum estiver em Todo, mova com `gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>` (o id da opção sai de `gh project field-list 1 --owner rotasaude --format json`). Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entrada do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod32b
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'urgency\|pharmacy\|sanitary_transport_disabled\|city_unreachable'
grep -n 'key: "urgency_stock_deduction"' -A3 $A/app/services/platform/features.rb
grep -n 'when /\\Afeature:' -A3 $A/app/services/platform/features.rb
```

Expected:
- em `features.ts`: `pharmacy:` e `urgency_care:`/`waiting_room_display:` em `FEATURE_LABELS`; nenhuma linha com `urgency_stock_deduction`, `urgency_care_disabled` ou `pharmacy_disabled`;
- a última entrada de `FEATURE_LABELS` é `waiting_room_display` (ou a de um módulo que entrou depois; use-a como âncora);
- no api: `urgency_stock_deduction` é a **última** entrada do `CATALOG`, com `requires: %w[feature:urgency_care feature:pharmacy]` e sem `environments`; o ramo `feature:` continua gerando `"#{dependency}_disabled"`.

Outro `requires`, outra ordem, ambiente restrito ou outro formato de `missing`: **pare e reporte**. **Anote a `description`**: ela substitui o rascunho da Task 1.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod32b -b feat/mod-32b-urgency-observation origin/main
cd $R/.worktrees/maintenance-mod32b && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod32b
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos). A Task 2 espera o **mesmo número de arquivos de teste**, com casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

**Relato ao coordenador:** "maintenance 32b · Task 0 · worktree maintenance-mod32b na base <sha>, entrada do api conferida (requires urgency_care+pharmacy), suíte base <n> verde."

---

### Task 1: Nome de `urgency_stock_deduction` e frases dos pré-requisitos

**Files:**
- Modify: `src/lib/features.ts` (uma entrada em `FEATURE_LABELS`, depois de `waiting_room_display`; duas em `MISSING_LABELS`, antes de `city_unreachable`), `README.md` (parágrafo depois de "Urgência e painel da TV (módulo 32a)", hoje na linha 342)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("urgency_stock_deduction") === "Baixa de estoque na medicação da UPA"`; `missingText("urgency_care_disabled") === "urgência e emergência (urgency_care) desligada"`; `missingText("pharmacy_disabled") === "farmácia e almoxarifado (pharmacy) desligada"`.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 32b", () => {
  it("módulo 32b: nome e frases; missing desconhecido cru", () => {
    expect(featureLabel("urgency_stock_deduction")).toBe("Baixa de estoque na medicação da UPA");
    expect(missingText("urgency_care_disabled")).toBe("urgência e emergência (urgency_care) desligada");
    expect(missingText("pharmacy_disabled")).toBe("farmácia e almoxarifado (pharmacy) desligada");
    expect(missingText("upa_stock_location_missing")).toBe("upa_stock_location_missing");
  });

  it("urgency_stock_deduction: os dois pré-requisitos na ordem do requires", () => {
    expect(missingSummary([ "urgency_care_disabled", "pharmacy_disabled" ]))
      .toBe("urgência e emergência (urgency_care) desligada; farmácia e almoxarifado (pharmacy) desligada");
  });

  it("os nomes do 27a e do 32a não mudam", () => {
    expect(featureLabel("pharmacy")).toBe("Farmácia e almoxarifado");
    expect(featureLabel("urgency_care")).toBe("Urgência e emergência (UPA)");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (a descrição é rascunho; use a anotada na Task 0):

```tsx
describe("FeaturesTab — baixa de estoque na medicação da UPA (módulo 32b)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const DED_OFF: Feature = {
    key: "urgency_stock_deduction",
    description: "Atendimentos da UPA com prescrição de medicação atualizam o estoque",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const DED_ON: Feature = { ...DED_OFF, enabled: true, usable: true, changedAt: "2026-11-10T13:00:00Z", changedBy: "dev@local" };
  const URG_ON: Feature = {
    key: "urgency_care", description: "Urgência (UPA)", enabled: true, usable: true,
    missing: [], changedAt: "2026-11-05T13:00:00Z", changedBy: "dev@local"
  };
  const PH_ON: Feature = {
    key: "pharmacy", description: "Farmácia e almoxarifado", enabled: true, usable: true,
    missing: [], changedAt: "2026-10-30T13:00:00Z", changedBy: "dev@local"
  };

  it("aparece com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ DED_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("urgency_stock_deduction")).not.toBeNull();
    expect(screen.getByText("Baixa de estoque na medicação da UPA")).not.toBeNull();
    expect(screen.getByText(DED_OFF.description)).not.toBeNull();
    expect(screen.getByText("desligada")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("com urgência e farmácia, liga utilizável e desliga com dois cliques; só a chave pedida; nada fala em apagar", async () => {
    const user = userEvent.setup();
    let listed = [ URG_ON, PH_ON, DED_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        const ded = body.variables.enabled === true ? DED_ON : { ...DED_OFF, changedAt: "2026-11-10T14:00:00Z", changedBy: "dev@local" };
        listed = [ URG_ON, PH_ON, ded ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: ded } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar urgency_stock_deduction" }));
    expect((await screen.findByRole("status")).textContent).toBe("urgency_stock_deduction ligada e utilizável.");
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "urgency_stock_deduction", enabled: true });
    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[2]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar urgency_stock_deduction" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables)
      .toEqual({ citySlug: "sp", key: "urgency_stock_deduction", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("urgency_stock_deduction desligada.");
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(2);
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("baixa em cidade integrated: ligada, falta a urgência (e a farmácia), sem a frase do prontuário", async () => {
    const user = userEvent.setup();
    const DED_BLOCKED: Feature = { ...DED_ON, usable: false, missing: [ "urgency_care_disabled", "pharmacy_disabled" ] };
    let listed = [ DED_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ DED_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: DED_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar urgency_stock_deduction" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(await screen.findByText("ligada, falta pré-requisito")).not.toBeNull();
    expect((await screen.findByRole("status")).textContent).toBe(
      "urgency_stock_deduction ligada, mas ainda não utilizável — falta: urgência e emergência (urgency_care) desligada; farmácia e almoxarifado (pharmacy) desligada."
    );
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/clinical_record|undefined|apaga|exclu/i);
  });

  it("desligar pharmacy com a baixa ligada: só a chave pedida; a baixa fica faltando a farmácia", async () => {
    const user = userEvent.setup();
    const PH_OFF: Feature = { ...PH_ON, enabled: false, usable: false, changedAt: "2026-11-10T15:00:00Z" };
    const DED_ORPHAN: Feature = { ...DED_ON, usable: false, missing: [ "pharmacy_disabled" ] };
    let listed = [ URG_ON, PH_ON, DED_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ URG_ON, PH_OFF, DED_ORPHAN ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: PH_OFF } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[1]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar pharmacy" }));
    await waitFor(() => expect(screen.getByText("farmácia e almoxarifado (pharmacy) desligada")).not.toBeNull());
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "pharmacy", enabled: false });
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(1);
  });

  it("desligar a urgência com a baixa ligada: a baixa mostra só a urgência, nunca a frase do prontuário", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      { ...URG_ON, usable: false, missing: [ "clinical_record_disabled" ] },
      PH_ON,
      { ...DED_ON, usable: false, missing: [ "urgency_care_disabled" ] }
    ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("urgência e emergência (urgency_care) desligada")).not.toBeNull();
    expect(screen.getAllByText("prontuário da atenção primária (clinical_record) desligado")).toHaveLength(1);
    expect(screen.getAllByText("ligada, falta pré-requisito")).toHaveLength(2);
  });

  it("baixa ligada: nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ URG_ON, PH_ON, DED_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("urgency_stock_deduction")).not.toBeNull();
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

Nota para o executor: a ordem dos botões "desligar" (`[1]`, `[2]`) segue a ordem da lista devolvida pelo `fetch` falso, como nos casos do 32a. Se a aba ordenar de outro jeito, ajuste o índice pelo nome da linha, sem mudar o que o caso confere.

- [ ] **Step 2: Run** `bin/test maintenance -w maintenance-mod32b src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`. Expected: FAIL nos casos de nome e de frase (`featureLabel` devolve `null`; `urgency_care_disabled` e `pharmacy_disabled` saem crus). Os casos de "só a chave pedida" já podem passar nas asserções de variáveis, porque fixam o mecanismo genérico.

- [ ] **Step 3: Implemente**
  - **`features.ts`, `MISSING_LABELS`:** antes do comentário de `city_unreachable`, o comentário `// Módulo 32b (ADR 0050): urgency_stock_deduction exige urgency_care e pharmacy, ligados e utilizáveis; o api não propaga o pré-requisito do pré-requisito (prontuário desligado = urgency_care_disabled).` e as entradas `urgency_care_disabled: "urgência e emergência (urgency_care) desligada"` e `pharmacy_disabled: "farmácia e almoxarifado (pharmacy) desligada"`.
  - **`features.ts`, `FEATURE_LABELS`:** depois de `waiting_room_display`, o comentário `// Módulo 32b (ADR 0050): exige urgency_care e pharmacy (frases acima). Todos os ambientes, sem simulado.` e a entrada `urgency_stock_deduction: "Baixa de estoque na medicação da UPA"`. Corrija o comentário do 32a (linha 136) tirando "urgency_stock_deduction é do 32b."; nada mais muda nas entradas existentes.
  - **`README.md`:** parágrafo **Baixa de estoque na medicação da UPA (módulo 32b).** depois do de Urgência e painel da TV:
    - `urgency_stock_deduction` ("Baixa de estoque na medicação da UPA") faz cada medicação checada pela enfermagem na UPA dar baixa no estoque `unit` da UPA, pelo razão da farmácia (lote por FEFO); sem saldo, a checagem é gravada e nasce uma falta para a farmácia conciliar, e o estoque nunca fica negativo;
    - exige `urgency_care` e `pharmacy` ligados e utilizáveis; com um deles desligado, a baixa fica "ligada, falta pré-requisito", e a aba não desliga uma pela outra. Com o prontuário desligado, a falta mostrada é a da urgência;
    - desligada, observação, prescrição e checagem funcionam igual, só sem lote e sem movimento de estoque;
    - leitos, prescrição e a tela de faltas da UPA ficam no dashboard; a chave existe em todos os ambientes, sem simulado;
    - desligar não desfaz nada: as administrações, os movimentos já gravados e as faltas ficam, e as faltas antigas seguem visíveis para a farmácia;
    - ordem de deploy: api → dashboard → maintenance; contra um api sem o 32b, a chave só não aparece.

- [ ] **Step 4: Run** o mesmo comando e `cd $M && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the urgency stock deduction switch and its urgency and pharmacy prerequisites

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance 32b · Task 1 · <sha> nome de urgency_stock_deduction e frases de urgency_care_disabled/pharmacy_disabled, specs da aba verdes."

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run: `bin/test maintenance -w maintenance-mod32b && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 2, 22 e 23, R7, F-32.8) e o contrato §1, §6, §7 e §11. Ele confere:
  - uma entrada nova em `FEATURE_LABELS` e duas em `MISSING_LABELS`, nenhuma frase existente redefinida;
  - nenhum aviso de simulado;
  - nenhum dado de cidade aqui;
  - que o README não promete apagar nada e não diz que a observação ou a prescrição dependem da baixa.

  Confira também: `grep -n -i "apaga\|exclu" README.md | grep -i "estoque\|baixa\|falta"`. Expected: sem saída.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 32b na **3055** (banco de dev migrado **com autorização do usuário**) e do dashboard do 32b na 5215 (para o passo 3). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod32b --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod32b:/app -v maintenance-mod32b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3055 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod32b   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP; **anote antes** o estado de `urgency_care`, `pharmacy` e `urgency_stock_deduction` nas duas cidades. Guarde os screenshots em `evidencias/2026-11-10-modulo-32b/maintenance/`. Roteiro:
1. Curitiba (`record`) → Funcionalidades: `urgency_stock_deduction` com nome e descrição, "desligada".
2. Com `urgency_care` e `pharmacy` utilizáveis (ligue o que faltar, se o usuário autorizar), ligar a baixa → "ligada e utilizável".
3. No dashboard (5215), uma checagem `administered` na UPA de Curitiba gera o movimento `administration` (a prova detalhada é do plano do dashboard; aqui só se confere que a chave chegou).
4. Desligar `pharmacy` (só se o usuário autorizar; senão pule): a baixa continua ligada e passa a "ligada, falta pré-requisito" com "farmácia e almoxarifado (pharmacy) desligada". Religue a farmácia.
5. Maringá (`integrated`): ligar a baixa mostra "ligada, falta pré-requisito" com "urgência e emergência (urgency_care) desligada" (e a da farmácia, se ela estiver desligada lá), nunca a frase do prontuário. Desligue de volta.
6. **Auditoria** lista as tentativas de `setCityFeature`.
7. A 375 px, sem rolagem lateral da página, com a frase dupla de pré-requisito quebrando dentro da linha.

Depois, volte as chaves ao estado anotado (ou ao que a prova do dashboard precisar) e limpe: `docker rm -f maintenance-mod32b && docker volume rm maintenance-mod32b-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api e do dashboard do 32b, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `git fetch`, `merge --ff-only feat/mod-32b-urgency-observation`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api e do dashboard do F-32.8 estiverem **Done**, mova o F-32.8 para **Done** (opção `98236657`), e para **Verified** se as provas passaram.

**Relato ao coordenador:** "maintenance 32b · Task 2 · suíte <n> verde, build ok, prova na 5177 (Curitiba utilizável/falta farmácia, Maringá falta urgência, 375 px) <passou|pendente>; aguardando autorização de merge."

---

## Divergências propostas ao contrato

1. **M1 — `missing` não é transitivo.** O §11 diz "`MISSING_LABELS` para `urgency_care_disabled` e `pharmacy_disabled`, se o api os devolver". Conferido: devolve, e só esses. `missing_for` gera `"#{dependency}_disabled"` quando a dependência não está utilizável (`ro-api/app/services/platform/features.rb:329-331`), sem descer ao pré-requisito dela. Com o prontuário desligado, a baixa mostra `urgency_care_disabled`, não `clinical_record_disabled`. Proposta: o §11 troca "se o api os devolver" por "o api devolve os dois, na ordem do `requires`, e nunca o pré-requisito do pré-requisito"; o README do maintenance diz isso em uma frase.
2. **M2 — Frases dos pré-requisitos.** O contrato fixa o nome da chave, mas não as frases. Nenhuma das duas existe em `ro-maintenance/src/lib/features.ts:24-75`. Proposta: "urgência e emergência (urgency_care) desligada" e "farmácia e almoxarifado (pharmacy) desligada", no molde de "sala de vacina (vaccination) desligada" (`features.ts:64`). O dashboard, se mostrar `missing`, usa as mesmas.
3. **M3 — "Desligado" ou "não utilizável".** O §6 decide o estoque por `urgency_stock_deduction` **utilizável**, mas a regra "Interruptor desligado: `stock_lot_id`/`stock_quantity` → 422 `stock_deduction_disabled`" e o §1 ("Desligado, a administração é registrada sem lote") falam em **desligado**. Fica ambíguo o estado "ligada, falta pré-requisito" (farmácia desligada com a baixa ligada), que é justamente o que a aba do maintenance passa a mostrar. Proposta: o §1 e o §6 dizem "desligado **ou não utilizável**" nas duas regras, e o plano do api fixa isso num request spec (baixa ligada + `pharmacy` desligada → sem movimento, sem falta, e lote no corpo → 422 `stock_deduction_disabled`).
4. **M4 — Comentário do 32a.** `ro-maintenance/src/lib/features.ts:136` diz "urgency_stock_deduction é do 32b." Proposta: a Task 1 tira essa frase quando a chave entra, para o comentário não ficar falso. Sem efeito em comportamento; registro só para o revisor não estranhar a linha alterada do 32a.

## Self-review

- **Cobertura (Afeta, decisões 2, 22 e 23, R7, F-32.8; contrato §1, §6, §7, §11):**
  - nome e frases: Task 1 (Review Focus 3 e 5; M1, M2);
  - sem cascata, farmácia desligada: Task 1, quarto caso da aba (Review Focus 1);
  - cidade `integrated` e não transitividade: Task 1, terceiro e quinto casos (Review Focus 2; M1);
  - desligar sem apagar: Task 1 (Review Focus 4) e o README;
  - sem simulado: Task 1 (Review Focus 5);
  - suíte, build e prova (Curitiba, Maringá, 375 px): Task 2; cards: Tasks 0 e 2.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards, a `description` e a ordem do `requires` (Task 0, com a regra para diferença).
- **Tipos e nomes:** existem em `origin/main` (`bbff803`): `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus` (pelos três estados), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e `Feature`. No api (`69d283c4`): `Platform::Features::CATALOG`, `missing_for`, `usable?`.
- **Proporção:** os blocos de código são só de testes; a mudança de produção são três linhas e dois comentários.
