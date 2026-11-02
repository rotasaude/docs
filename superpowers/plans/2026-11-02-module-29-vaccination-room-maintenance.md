# Módulo 29 — Sala de vacina (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 29** na branch `feat/mod-29-vaccination-room` (worktree `apps/api/.claude/mod29`, servidor de dev na porta **3051**) com as três entradas em `Platform::Features::CATALOG` (contrato §1): `vaccination` (`requires: %w[record_mode:record]`), `rnds_immunization` (`requires: %w[feature:vaccination credential:rnds]`) e `rnds_mock` (`requires: %w[feature:rnds_immunization]`, `environments: SIMULATION_ENVS`). Contrato §14: api → dashboard → wpda → **maintenance**. As Tasks 1 e 2 não dependem do api rodando (testes com `fetch` falso); a prova da Task 3 precisa dele.

**Goal:** Lado maintenance de F-29.1: os três interruptores do módulo aparecem na aba Funcionalidades com nome em português, os pré-requisitos novos (`vaccination_disabled`, `rnds_immunization_disabled`, `credential_missing:rnds`, `credential_unauthorized:rnds`) aparecem em frase, e a RNDS simulada ligada acende um aviso na aba, como o SNCR e o laboratório simulados.

**Architecture:** Conferido no código (`origin/main` do maintenance, `1fdda62`): o liga/desliga é genérico (`setCityFeature`, três estados, `missing` vindo do api), mas **o catálogo do api não basta** — sem mudança aqui as três chaves aparecem cruas e quatro dos `missing` também (a aba mostra o código, não quebra). Mudam: `FEATURE_LABELS` e `MISSING_LABELS` em `src/lib/features.ts` (Task 1); um `src/lib/rnds.ts` com o aviso da RNDS simulada e uma linha `role="note"` em `src/screens/FeaturesTab.tsx`, no molde exato de `fakeLabOn`/`FAKE_LAB_NOTICE` (Task 2). Nenhuma tela nova, nenhum SDL novo, nenhum quadro de plataforma: a credencial `rnds` é **da cidade** (o administrador municipal grava no dashboard, contrato §3), diferente do SNCR, que é do ambiente.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-02-module-29-vaccination-room-design.md` (§2 decisão 18, §7, §8 F-29.1), `docs/adr/0047.md` e o contrato `docs/superpowers/plans/2026-11-02-module-29-vaccination-room-contracts.md` §1, §12 e §14. Base: plano do maintenance do 28 (`2026-10-30-module-28-oral-health-maintenance.md`) e o aviso do `fake_lab` do 21a (`src/lib/exams.ts`). As diferenças estão em "Divergências propostas ao contrato", M1–M4.

## Global Constraints

- Interruptores (spec decisão 18; contrato §1), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `vaccination` exige `record_mode:record` → `missing` `record_mode_not_record` (frase existente do 19a, não redefinida);
  - `rnds_immunization` exige `vaccination` e a credencial `rnds` → `vaccination_disabled`, `credential_missing:rnds`, `credential_unauthorized:rnds`;
  - `rnds_mock` exige `rnds_immunization` → `rnds_immunization_disabled`; só existe fora de produção (o api não o lista lá);
  - ligar ou desligar uma chave só manda aquela `key`: a aba nunca liga pré-requisito junto.
- Nomes (M1): `vaccination` → "Sala de vacina"; `rnds_immunization` → "Envio das vacinas à RNDS (Ministério da Saúde)"; `rnds_mock` → "RNDS simulada (desenvolvimento)".
- Aviso da RNDS simulada pelo estado **ligado** de `rnds_mock` (não pelo "utilizável"), `role="note"`, `aria-label="modo da RNDS"`, cor `var(--warn)`, como os quatro avisos existentes. Sem `rnds_mock` no catálogo (produção) é o mesmo que desligado.
- Desligar não apaga nada: doses, cadernetas, fila e estoque ficam (a exclusão LGPD retém a dose, ADR 0026). A aba não diz o contrário.
- O maintenance não vê nada de cidade: nenhuma dose, vacina, lote, certificado, senha ou cidadão passa por aqui; a credencial `rnds` não é gravada nem lida aqui.
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte; a tabela rola dentro do `TableScroll`. Achado só visual não vira tarefa.
- O módulo 28 corre em paralelo e mexe nas **mesmas linhas** de `features.ts` (acrescenta `dental_care` depois de `pharmacy`). Âncora: a última entrada de `FEATURE_LABELS` em `origin/main` na hora da execução; conflito no rebase é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-29.1..F-29.8 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-29.1: Done quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod29`, `A=$R/apps/api/.claude/mod29`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-29-vaccination-room`. Caminhos das tasks relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod29 <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177** com o serviço `maintenance` do compose parado; `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local`; o usuário faz login e TOTP. Banco de dev só migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade em modo `integrated` ou `off`** (vacina no PEC). Ligar `vaccination` dá "ligada, falta pré-requisito" com "modo de prontuário diferente de record" — nunca "utilizável". Teste: Task 1, "vaccination fora do modo record: ligada, falta o modo, com a frase do 19a".
2. **Envio à RNDS sem certificado, ou com o certificado recusado.** A linha diz "credencial RNDS não cadastrada" / "credencial RNDS recusada no último teste", nunca o código. Teste: Task 1, "rnds_immunization: sala, credencial ausente e recusada em português".
3. **Produção.** Sem `rnds_mock` no catálogo, nenhum aviso de simulado; `vaccination` e `rnds_immunization` aparecem. Teste: Task 2, "produção: sem rnds_mock, nenhum aviso de RNDS simulada".
4. **RNDS simulada ligada mas não utilizável** (envio desligado). O aviso aparece mesmo assim (pelo estado ligado), porque religar o envio passa a usar o simulado. Teste: Task 2, "rnds_mock ligado sem o envio: o aviso aparece".
5. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `rnds_cnes_missing`). Aparece cru. Teste: Task 1, "módulo 29: nomes; missing desconhecido cru".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nomes e frases dos pré-requisitos; fluxo genérico fixado para `vaccination` e `rnds_immunization` | 1 |
| `src/lib/rnds.ts`, `src/lib/rnds.test.ts`, `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx` | aviso da RNDS simulada | 2 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 3 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`, `rndsMockOn`) e a aba pelo fluxo existente de dois cliques, com `fetch` falso por nome de operação e conferência das variáveis exatas da mutation (Tasks 1 e 2); suíte inteira com as guardas de tema, `codegen:check`, build e a prova contra o api do 29 (Task 3).

---

### Task 0: Cards, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`1fdda62` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards em In Progress; `$M`; a base anotada; as três `description` reais do api anotadas (usadas nos testes).

- [ ] **Step 1: Cards F-29.1..F-29.8 para In Progress** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-29\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito em In Progress ou além. Card ausente → **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod29
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'record_mode_not_record:\|"credential_unauthorized:cadsus"\|city_unreachable:\|pharmacy: "Farmácia e almoxarifado"\|dental_care\|rnds\|vaccin'
$G -C $R/apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n 'fakeMiddlewareOn\|aria-label="modo do middleware"'
grep -n 'key: "vaccination"\|key: "rnds_immunization"\|key: "rnds_mock"' -A3 $A/app/services/platform/features.rb
```

Expected: em `features.ts`, `record_mode_not_record` traduzido, nenhuma linha com `rnds`/`vaccin`; a última entrada de `FEATURE_LABELS` é `pharmacy` (ou `dental_care`, se o 28 já entrou — use-a como âncora); no `FeaturesTab.tsx`, o aviso do middleware. No api, as três entradas com os `requires` da Global Constraints e `environments: SIMULATION_ENVS` só em `rnds_mock`. Outro `requires` ou ambiente → **pare e reporte**. **Anote as três `description`**: substituem os rascunhos das Tasks 1 e 2.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod29 -b feat/mod-29-vaccination-room origin/main
cd $R/.worktrees/maintenance-mod29 && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod29
```

Expected: verde e `base-ok`. **Anote** a base: a Task 3 espera **+1 arquivo de teste** (`src/lib/rnds.test.ts`) e casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: Nomes e pré-requisitos dos três interruptores

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("vaccination" | "rnds_immunization" | "rnds_mock")` com os nomes da Global Constraints; `missingText` para os quatro códigos novos.

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 29", () => {
  it("módulo 29: nomes; missing desconhecido cru", () => {
    expect(featureLabel("vaccination")).toBe("Sala de vacina");
    expect(featureLabel("rnds_immunization")).toBe("Envio das vacinas à RNDS (Ministério da Saúde)");
    expect(featureLabel("rnds_mock")).toBe("RNDS simulada (desenvolvimento)");
    expect(missingText("rnds_cnes_missing")).toBe("rnds_cnes_missing");
  });

  it("rnds_immunization: sala, credencial ausente e recusada em português", () => {
    expect(missingSummary([ "vaccination_disabled", "credential_missing:rnds" ]))
      .toBe("sala de vacina (vaccination) desligada; credencial RNDS não cadastrada");
    expect(missingText("credential_unauthorized:rnds")).toBe("credencial RNDS recusada no último teste");
    expect(missingText("rnds_immunization_disabled")).toBe("envio das vacinas à RNDS (rnds_immunization) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (descrições = rascunho; use as anotadas na Task 0):

```tsx
describe("FeaturesTab — sala de vacina (módulo 29)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const VAC_OFF: Feature = {
    key: "vaccination", description: "Sala de vacina: registro da dose, caderneta, estoque de imunobiológicos e cadeia de frio",
    enabled: false, usable: false, missing: [ "record_mode_not_record" ], changedAt: null, changedBy: null
  };
  const SEND_OFF: Feature = {
    key: "rnds_immunization", description: "Envio das doses à RNDS (RIA-R 2.0)",
    enabled: false, usable: false, missing: [ "vaccination_disabled", "credential_missing:rnds" ], changedAt: null, changedBy: null
  };

  it("vaccination fora do modo record: ligada, falta o modo, com a frase do 19a", async () => {
    const user = userEvent.setup();
    const VAC_BLOCKED: Feature = { ...VAC_OFF, enabled: true, changedAt: "2026-11-02T13:00:00Z", changedBy: "dev@local" };
    let listed = [ VAC_OFF, SEND_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ VAC_BLOCKED, SEND_OFF ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: VAC_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("Sala de vacina")).not.toBeNull();
    expect(screen.getByText("Envio das vacinas à RNDS (Ministério da Saúde)")).not.toBeNull();
    expect(screen.getByText("sala de vacina (vaccination) desligada; credencial RNDS não cadastrada")).not.toBeNull();
    await user.click(screen.getAllByRole("button", { name: "ligar" })[0]);
    await user.click(await screen.findByRole("button", { name: "confirmar: ligar vaccination" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "vaccination", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "vaccination ligada, mas ainda não utilizável — falta: modo de prontuário diferente de record."
    );
    expect(document.body.textContent).not.toMatch(/undefined|apaga|exclu/i);
  });

  it("com o modo record: liga utilizável e desliga com dois cliques", async () => {
    const user = userEvent.setup();
    const VAC_ON: Feature = { ...VAC_OFF, enabled: true, usable: true, missing: [], changedAt: "2026-11-02T13:00:00Z", changedBy: "dev@local" };
    let listed = [ { ...VAC_OFF, missing: [] } ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? VAC_ON : { ...VAC_OFF, missing: [] } ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar vaccination" }));
    expect((await screen.findByRole("status")).textContent).toBe("vaccination ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar vaccination" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "vaccination", enabled: false });
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod29 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("vaccination")` é `null` e `missingText("credential_missing:rnds")` volta cru. (Fluxo de dois cliques e `record_mode_not_record` já passam.)

- [ ] **Step 3: Implemente em `src/lib/features.ts`**

Em `MISSING_LABELS`, antes de `city_unreachable` (com comentário do módulo 29, ADR 0047): `vaccination_disabled: "sala de vacina (vaccination) desligada"`, `rnds_immunization_disabled: "envio das vacinas à RNDS (rnds_immunization) desligado"`, `"credential_missing:rnds": "credencial RNDS não cadastrada"`, `"credential_unauthorized:rnds": "credencial RNDS recusada no último teste"`. Em `FEATURE_LABELS`, depois da última entrada (vírgula na anterior): os três nomes da Global Constraints, com comentário "Módulo 29 (ADR 0047): vaccination exige record_mode:record; rnds_immunization exige vaccination e a credencial rnds da cidade (gravada no dashboard); rnds_mock só fora de produção".

`README.md`, depois do último parágrafo de módulo antes de **Documentos clínicos (módulo 19c).** — parágrafo **Sala de vacina (módulo 29).**: as três chaves e seus nomes; `vaccination` só em cidade `record` (cidade `integrated` vacina no PEC, que já envia à RNDS); `rnds_immunization` exige a credencial `rnds`, gravada pelo administrador municipal no dashboard (certificado A1; nunca passa por aqui); `rnds_mock` só fora de produção, com aviso na aba; desligar não apaga doses, cadernetas nem estoque (exclusão LGPD retém a dose); ordem de deploy api → dashboard → wpda → maintenance; contra um api sem o 29, as chaves só não aparecem.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod29 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (testes antigos da aba iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the vaccination room switches and their prerequisites

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Aviso da RNDS simulada

**Files:**
- Create: `src/lib/rnds.ts`
- Modify: `src/screens/FeaturesTab.tsx` (import, `const rndsSimulated`, um `<p role="note">` depois do aviso do middleware)
- Test: `src/lib/rnds.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: o padrão de `src/lib/exams.ts` (`FAKE_LAB_KEY`, `fakeLabOn`).
- Produces: `RNDS_MOCK_KEY = "rnds_mock"`; `RNDS_MOCK_NOTICE: string`; `rndsMockOn(features: { key: string; enabled: boolean }[]): boolean`.

- [ ] **Step 1: Write the failing test**

`src/lib/rnds.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { RNDS_MOCK_KEY, RNDS_MOCK_NOTICE, rndsMockOn } from "./rnds";

describe("rndsMockOn", () => {
  it("só pelo estado ligado de rnds_mock; ausente (produção) é desligado", () => {
    expect(rndsMockOn([ { key: RNDS_MOCK_KEY, enabled: true } ])).toBe(true);
    expect(rndsMockOn([ { key: RNDS_MOCK_KEY, enabled: false } ])).toBe(false);
    expect(rndsMockOn([ { key: "rnds_immunization", enabled: true } ])).toBe(false);
    expect(rndsMockOn([])).toBe(false);
  });

  it("o aviso diz simulada e que nada chega ao Ministério", () => {
    expect(RNDS_MOCK_NOTICE).toBe(
      "Esta cidade envia as vacinas à RNDS SIMULADA — as doses recebem aceite ou recusa de exemplo e não chegam ao Ministério da Saúde nem ao Meu SUS Digital (fora de produção)."
    );
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (importe `RNDS_MOCK_NOTICE` de `../lib/rnds` no topo):

```tsx
describe("FeaturesTab — RNDS simulada (módulo 29)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const SEND_ON: Feature = {
    key: "rnds_immunization", description: "Envio das doses à RNDS (RIA-R 2.0)",
    enabled: true, usable: true, missing: [], changedAt: "2026-11-02T13:00:00Z", changedBy: "dev@local"
  };
  const MOCK_OFF: Feature = {
    key: "rnds_mock", description: "RNDS simulada (desenvolvimento) — aceite e recusa de exemplo",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const MOCK_ON: Feature = { ...MOCK_OFF, enabled: true, usable: true, changedAt: "2026-11-02T14:00:00Z", changedBy: "dev@local" };

  it("ligar rnds_mock por dois cliques: após o refetch a aba avisa", async () => {
    const user = userEvent.setup();
    let listed = [ SEND_ON, MOCK_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ SEND_ON, MOCK_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: MOCK_ON } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("RNDS simulada (desenvolvimento)")).not.toBeNull();
    expect(screen.queryByRole("note", { name: "modo da RNDS" })).toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(await screen.findByRole("button", { name: "confirmar: ligar rnds_mock" }));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "rnds_mock", enabled: true });
    expect((await screen.findByRole("note", { name: "modo da RNDS" })).textContent).toBe(RNDS_MOCK_NOTICE);
  });

  it("rnds_mock ligado sem o envio: o aviso aparece", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([
      { ...SEND_ON, enabled: false, usable: false },
      { ...MOCK_ON, usable: false, missing: [ "rnds_immunization_disabled" ] }
    ])));
    renderTab();
    expect((await screen.findByRole("note", { name: "modo da RNDS" })).textContent).toBe(RNDS_MOCK_NOTICE);
    expect(screen.getByText("envio das vacinas à RNDS (rnds_immunization) desligado")).not.toBeNull();
  });

  it("produção: sem rnds_mock, nenhum aviso de RNDS simulada", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ SEND_ON ], { recordMode: "record" })));
    renderTab();
    expect(await screen.findByText("rnds_immunization")).not.toBeNull();
    expect(screen.queryByRole("note", { name: "modo da RNDS" })).toBeNull();
    expect(document.body.textContent).not.toMatch(/SIMULAD|undefined|null/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod29 src/lib/rnds.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `./rnds` não existe.

- [ ] **Step 3: Implemente**

`src/lib/rnds.ts` no molde de `src/lib/exams.ts`: comentário do módulo 29 (ADR 0047; nenhum certificado, dose ou CPF passa pelo maintenance), `RNDS_MOCK_KEY`, `RNDS_MOCK_NOTICE` com o texto exato do teste, `rndsMockOn` pelo estado **ligado**. Em `FeaturesTab.tsx`: `const rndsSimulated = city ? rndsMockOn(city.features) : false;` junto dos outros e, depois do aviso do middleware, `<p role="note" aria-label="modo da RNDS" …>` com o mesmo estilo dos quatro avisos existentes.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod29 src/lib src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/rnds.ts src/lib/rnds.test.ts src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx
$G commit -m "feat: warn when a city sends vaccines to the simulated RNDS

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: Tasks 1 e 2; o api do 29 na **3051**.
- Produces: branch pronta para merge (depois do api, do dashboard e do wpda do 29).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod29 && cd $M && npm run typecheck && npm run codegen:check && npm run build && $G status --short && $G diff origin/main --stat`
Expected: verde; a base da Task 0 + `rnds.test.ts`; `status` vazio; o diff só com os arquivos das Tasks 1 e 2 (nada em `schema.graphql` ou `src/gql/`). Número de arquivos que dobra = artefato de build descoberto pelo vitest: pare e reporte.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 18, §7, F-29.1), o ADR 0047 e o contrato §1, §12, §14. Atenção: nenhuma linha nova no fluxo de `setCityFeature`; nenhuma frase redefinida para `record_mode_not_record`; o aviso só pelo estado ligado de `rnds_mock`; o README não promete consulta ao histórico nacional, SIES/SIPNI nem e-SUS Notifica (fora da entrega); nenhum dado de cidade aqui.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 29 na **3051** (banco de dev migrado **com autorização do usuário**). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod29 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod29:/app -v maintenance-mod29-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3051 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod29   # espere o "ready"
```

Em `http://maintenance.localhost:5177` (o usuário faz login e TOTP; anote antes o estado das três chaves), com screenshot em `evidencias/2026-11-02-modulo-29/maintenance/`:
1. Curitiba (`record`) → Funcionalidades: as três chaves com nome e descrição; ligar `vaccination` → "ligada e utilizável"; `rnds_immunization` sem credencial → "credencial RNDS não cadastrada".
2. Depois que o administrador municipal gravar a credencial no dashboard (prova do dashboard; em dev, o certificado de teste do `fake-rnds`), atualizar: `rnds_immunization` utilizável; ligar `rnds_mock` → o aviso "modo da RNDS".
3. Uma cidade em `integrated`: ligar `vaccination` → "modo de prontuário diferente de record"; desligar de volta.
4. **Auditoria** lista as tentativas de `setCityFeature`.
5. A 375 px: sem rolagem lateral da página.

Depois: chaves de volta ao estado anotado (ou ao que as provas do dashboard e do wpda precisarem); `docker rm -f maintenance-mod29 && docker volume rm maintenance-mod29-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** Merge só depois do api, do dashboard e do wpda do 29, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só com os commits desta entrega. Quando autorizado: `git fetch`, `merge --ff-only feat/mod-29-vaccination-room`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api e do dashboard do F-29.1 estiverem **Done**, mova o F-29.1 para **Done** (opção `98236657`) e para **Verified** se as provas passaram.

---

## Divergências propostas ao contrato

1. **M1 — Nomes.** O §12 não fixa rótulos. Proposta: "Sala de vacina", "Envio das vacinas à RNDS (Ministério da Saúde)" e "RNDS simulada (desenvolvimento)", iguais no dashboard; a descrição é a do catálogo do api.
2. **M2 — "Só os interruptores" exige mudança aqui.** O §12 dá a entender que basta o catálogo do api. Conferido: sem os rótulos, a aba mostra as chaves e quatro `missing` crus (`vaccination_disabled`, `rnds_immunization_disabled`, `credential_missing:rnds`, `credential_unauthorized:rnds` — os nomes que `Platform::Features#missing_for` gera para `feature:` e `credential:`). Proposta: o §12 lista esses quatro códigos e diz que o maintenance os traduz.
3. **M3 — Aviso da RNDS simulada.** O §12 diz "nenhuma tela nova"; o aviso não é tela, é uma linha `role="note"` na aba, como `sncr_mock`, `fake_lab` e `fake_middleware`. Proposta: o §12 registra o aviso pelo estado ligado de `rnds_mock`.
4. **M4 — Credencial no caminho do simulado (para o api).** `rnds_mock` exige `rnds_immunization`, que exige `credential:rnds` com o último teste não recusado. Em dev, a prova só fica utilizável com um certificado de teste gravado e o `check` respondido pelo `fake-rnds`. Proposta: o plano do api semeia (ou documenta) esse certificado de teste em dev; sem isso, a prova do Step 3 para no item 2.

## Self-review

- **Cobertura (decisão 18, §7, F-29.1; contrato §1, §12, §14):** nomes e pré-requisitos — Task 1 (Review Focus 1, 2, 5; M1, M2); aviso do simulado — Task 2 (Review Focus 3, 4; M3); desligar sem apagar — Task 1 (segundo caso da aba); README — Task 1; suíte, build e prova — Task 3; cards — Tasks 0 e 3.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards e as três `description` (Task 0, com a regra para diferença).
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem em `origin/main` (`1fdda62`); `rndsMockOn`/`RNDS_MOCK_NOTICE`/`RNDS_MOCK_KEY` (Task 2) seguem `fakeLabOn`/`FAKE_LAB_NOTICE`/`FAKE_LAB_KEY`.
- **Review Focus:** 1 — Task 1 "vaccination fora do modo record…"; 2 e 5 — Task 1 (`features.test.ts`); 3 e 4 — Task 2.
- **Proporção:** blocos de código só de testes e do texto exato do aviso.
