# Módulo 24b — Visita domiciliar e atividade coletiva (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 24b** na branch `feat/mod-24b-home-visits` (worktree `apps/api/.claude/mod24b`, porta **3057**) com duas entradas novas no fim de `Platform::Features::CATALOG` (contrato §1): `home_visits` (`requires: %w[feature:territory_registry]`) e `collective_activities` (`requires: %w[feature:clinical_record]`), as duas sem `environments`; e `min_gap_days` em `CareLines::Rules::PRACTICE_KEYS` com o catálogo-base novo (contrato §12). Ordem de entrega (contrato §15): api → dashboard → wpda → **maintenance**. As Tasks 0–2 não dependem do api rodando (testes com `fetch` falso); a prova da Task 3 depende (e a publicação do catálogo-base em dev pode ser pedida antes da prova do dashboard e do wpda, M-5).

**Goal:** Lado maintenance de F-24.9 (e do catálogo-base de F-24.14). Os interruptores `home_visits` e `collective_activities` aparecem na aba Funcionalidades com nome em português; o pré-requisito `territory_registry_disabled` sai com frase própria; o liga/desliga genérico fica fixado em teste para as duas chaves (inclusive desligar o cadastro territorial com a visita ligada). Na tela **Linhas de cuidado**, a boa prática ganha o campo **Intervalo mínimo entre eventos (dias)** (`min_gap_days`), sem o qual a versão nova da C5 do catálogo-base não pode ser publicada pela tela, e a fonte "visita do ACS" deixa de dizer "não acompanhada pelo sistema".

**Architecture:** Conferido no código (`origin/main` do maintenance, `9269b55`; api `d36aef3`). O liga/desliga é genérico: `setCityFeature`, três estados (`featureStatus`) e `missing` vindo do api. Em `ro-api/app/services/platform/features.rb`, `missing_for` gera `"#{dependency}_disabled"` para cada `feature:<chave>` cuja dependência não está **ligada e utilizável**; `territory_registry` exige `feature:clinical_record ibge_code uf`, então o api do 24b devolve `territory_registry_disabled` para `home_visits` tanto com o cadastro desligado quanto com ele ligado sem IBGE/UF ou sem prontuário (o api não propaga o pré-requisito do pré-requisito, como no 32b). Esse código não existe em `MISSING_LABELS` (`src/lib/features.ts:24-80`) e sairia cru. `collective_activities` devolve `clinical_record_disabled`, que já tem frase (`features.ts:37`). O SDL não muda: `CityFeature` já carrega tudo e `FeaturesTab.tsx` só renderiza o que vem dele. Nenhum aviso de simulado pega as chaves novas. Na tela de linhas de cuidado, `careLineTemplates.ts` preserva chave desconhecida da boa prática em `advanced` (ida e volta) e já edita `citizen_label`, mas **não** tem campo para `min_gap_days`, e o api recusa chave fora de `PRACTICE_KEYS` (`ro-api/app/services/care_lines/rules.rb:72`): a mudança é um campo, uma regra de validação, um texto na regra exibida e um nome de caminho de erro. A mutation `upsertCareLineTemplate` recebe `practices: JSON!`, então o SDL e o `codegen` não mudam.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md` (Afeta "`apps/maintenance`", decisões 12, 14 e 22, §7 runbook, §8 F-24.9 e F-24.14); `docs/adr/0052.md`; o contrato `docs/superpowers/plans/2026-11-12-module-24b-home-visit-contracts.md` §1, §12, §14 (maintenance), §15, §16 (C1, C14, C18). Molde: `docs/superpowers/plans/2026-11-11-module-33-ombudsman-maintenance.md`. As diferenças estão em "Divergências propostas ao contrato", M-1 a M-6.

**Reconciliação (§16 do contrato, 2026-11-12):** M-1 aceita (Q29: a frase com o nome do interruptor, como o plano já faz); M-2 aceita (Q19, decisão do coordenador: o campo `min_gap_days` da Task 2); M-3: rótulo de `acs_visit` aceito (Q29); `vaccine` e `oral_health` ficam fora — card no board #2 a abrir no fechamento do módulo; M-4 aceita (Q19: maintenance em F-24.14); M-5 aceita (Q19: as Tasks 1–2 podem rodar antes da prova do dashboard, e esta sessão publica a versão nova do catálogo-base em dev, na 5177, com autorização do usuário, antes das provas do dashboard e do wpda); M-6 aceita (Q24: a descrição do api passa a "Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo"; a Task 0 confere a `description` real).

## Global Constraints

- Interruptores (contrato §1; spec decisão 12):
  - `home_visits` exige `feature:territory_registry` ligado e utilizável (C1). `missing` = `territory_registry_disabled`. Descrição do api: "Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo" (Q24);
  - `collective_activities` exige `feature:clinical_record`. `missing` = `clinical_record_disabled` (frase do 19b). Descrição do api: "Atividade coletiva com ficha LEDI, sem PSE". Não depende do cadastro territorial.
  - A Task 0 confere os textos reais no api.
- Nomes (contrato §1): `home_visits` → "Visita domiciliar e territorial"; `collective_activities` → "Atividade coletiva". Frase nova: `territory_registry_disabled` → "cadastro individual e domiciliar (territory_registry) desligado ou incompleto" (M-1: o contrato §14 diz "cadastro territorial (…)", mas o nome do interruptor no maintenance é "Cadastro individual e domiciliar", L16 do 24a).
- Ligar ou desligar manda só aquela `key`. A aba nunca liga o cadastro territorial junto com a visita nem desliga a visita quando o cadastro é desligado (sem cascata).
- Desligar não apaga nada: folhas, visitas, atividades, participantes, aparelhos e atribuições ficam; com `home_visits` desligado, o job da madrugada ainda fecha as folhas de dias anteriores e gera a ficha (decisão 22). A aba e o README não dizem o contrário.
- O maintenance não vê nada de cidade: nenhuma visita, motivo, medida, observação, nome, CPF, CNS, token de aparelho ou microárea passa por aqui. Microáreas, atribuições e aparelhos são do dashboard.
- Catálogo-base (contrato §12, C14): `min_gap_days` é opcional, inteiro de 1 a 365; vazio = sem a chave no envio; o valor que veio do api volta igual. A tela não muda nenhuma versão sozinha: publicar continua sendo ato do mantenedor, e cada cidade adota no dashboard.
- Valor desconhecido (api mais novo) aparece cru, nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova; o campo novo usa o `Field` do formulário. As guardas de tema (`src/theme/*.test.ts`) rodam na suíte.
- Âncoras em `origin/main` na execução: em `FEATURE_LABELS`, a última entrada (hoje `ombudsman_anonymous_web`, `features.ts:148`); em `MISSING_LABELS`, logo antes do comentário de `city_unreachable` (hoje depois de `ombudsman_disabled`, `features.ts:76`); no README, depois do parágrafo **Cadastro individual e domiciliar (módulo 24a).** (`README.md:234-246`). Conflito no rebase com outro módulo é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Board: o maintenance é dono de F-24.9 junto com o api e o dashboard (spec §8), e da parte de tela do catálogo-base de F-24.14 (M-4). Os cards já estão em In Progress pelo plano do api; a Task 0 só confere.
- Execução coordenada: ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod24b`, `A=$R/apps/api/.claude/mod24b`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-24b-home-visits`. Caminhos das tasks relativos a `$M`.
- Testes: só `bin/test maintenance -w maintenance-mod24b <arquivos>` (de `$R`). Tipos: `cd $M && npx tsc --noEmit`. CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177**, com o serviço `maintenance` do compose parado, apontando para o api na **3057**. Abra `http://maintenance.localhost:5177`, nunca `localhost:5177`. Conta `dev@local`; o usuário faz login e TOTP. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Desligar o cadastro territorial com a visita ligada.** Só `territory_registry` vai na mutation. Depois do refetch, a linha da visita diz "ligada, falta pré-requisito" com "cadastro individual e domiciliar (territory_registry) desligado ou incompleto" e não é desligada junto. Teste: Task 1, "desligar territory_registry com home_visits ligada: só a chave pedida; a visita fica faltando o cadastro".
2. **Atividade coletiva presa ao cadastro territorial por engano** (copiar a dependência da visita). Em cidade `integrated`, `collective_activities` falta só o prontuário, nunca o cadastro. Teste: Task 1, "collective_activities em cidade integrated: falta o prontuário, não o cadastro territorial".
3. **`min_gap_days` perdido ou inventado na ida e volta.** Versão com 30 abre o campo com 30 e publica 30; versão sem a chave publica sem a chave; 0, 366 e 30,5 são recusados antes de enviar. Testes: Task 2, "rascunho lê min_gap_days para o campo e a ida e volta devolve o número", "campo vazio não manda a chave", "intervalo mínimo fora de 1 a 365 ou não inteiro é recusado", e os dois casos do formulário.
4. **Texto que prometa apagar ao desligar** (visitas, folhas, aparelhos) ou que diga que a visita do ACS "não é acompanhada". Testes: Task 1, os casos da aba (`not.toMatch(/apaga|exclu/i)`); Task 2, `sourceText("acs_visit")`; Task 3, grep no README.
5. **Api mais novo que a tela** (um `missing` futuro, como `field_devices_missing`). Aparece cru, e o aviso de simulado não aparece. Testes: Task 1, "módulo 24b: nomes, frase do cadastro; missing desconhecido cru" e "visita e atividade ligadas: nenhum aviso de simulado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nomes das duas chaves, frase de `territory_registry_disabled`, fluxo genérico fixado | 1 |
| `src/lib/careLineTemplates.ts`, `src/lib/careLineTemplates.test.ts`, `src/screens/CareLineTemplateForm.tsx`, `src/screens/CareLineTemplateForm.test.tsx`, `README.md` | campo `min_gap_days`, regra exibida, caminho de erro, rótulo de `acs_visit`, formas reais do catálogo-base novo | 2 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 3 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`; `toDraft`, `draftToVariables`, `draftProblem`, `ruleText`, `sourceText`, `upsertErrorText`) e as duas telas pelos fluxos existentes. Na aba, o `fetch` falso responde por nome de operação (helpers `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e o tipo `Feature`, no topo de `FeaturesTab.test.tsx:15-50`) e os testes conferem as variáveis exatas da mutation e o refetch que mostra o dependente faltando. No formulário, os helpers `openNewVersion`, `practice`, `publish`, `upserts` e `form` (`CareLineTemplateForm.test.tsx:28-36`) e as fixtures `C4_V2`, `C4_A` (`CareLineTemplates.fixtures.tsx`). As formas reais do catálogo-base (`REAL`, `careLineTemplates.test.ts:97`) passam a ser as do 24b, e o teste de ida e volta que já existe prova que nada se perde. A Task 3 roda a suíte inteira com as guardas de tema, o `codegen:check` (prova de que o SDL não mudou), o build e a prova contra o api do 24b.

---

### Task 0: Cards, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`9269b55` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: `$M`; a base anotada; as `description` reais das duas chaves e as práticas `acs_visit` do catálogo-base novo anotadas (usadas nos testes).

- [ ] **Step 1: Conferir os cards F-24.9 e F-24.14** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.(9|14) ")) | {id, title, status}'
```

Expected: os dois em In Progress ou além (movidos pelo plano do api). Se algum estiver em Todo, mova:

```bash
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod24b
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'ombudsman_disabled:\|city_unreachable:\|ombudsman_anonymous_web:\|home_visits\|collective_activities\|territory_registry_disabled'
$G -C $R/apps/maintenance show origin/main:src/lib/careLineTemplates.ts | grep -n 'min_gap_days\|acs_visit:'
grep -n 'key: "home_visits"\|key: "collective_activities"' -A3 $A/app/services/platform/features.rb
grep -n 'CITIZEN_HIDDEN =' $A/app/services/platform/features.rb
grep -n 'PRACTICE_KEYS' -A2 $A/app/services/care_lines/rules.rb
grep -n 'acs_visit\|min_gap_days\|citizen_label: "Visitas' $A/config/care_lines/catalog-base.yml
```

Expected:
- em `features.ts`, nenhuma linha com `home_visits`, `collective_activities` nem `territory_registry_disabled`;
- a última entrada de `FEATURE_LABELS` é `ombudsman_anonymous_web` (ou a de um módulo que entrou depois; use-a como âncora); `ombudsman_disabled` logo antes de `city_unreachable`;
- em `careLineTemplates.ts`, nenhum `min_gap_days`, e `acs_visit: "visita do ACS (não acompanhada pelo sistema)"`;
- no api, `home_visits` com `requires: %w[feature:territory_registry]` e `collective_activities` com `requires: %w[feature:clinical_record]`, as duas sem `environments` e no fim do `CATALOG`; `CITIZEN_HIDDEN` sem nenhuma das duas;
- `PRACTICE_KEYS` com `min_gap_days`; no `catalog-base.yml`, `min_gap_days: 30` só na `c5_d` e `citizen_label: "Visitas do agente de saúde"` nas seis práticas `acs_visit` (`c2_d`, `c3_e`, `c3_j`, `c4_d`, `c5_d`, `c6_c`).

Outro `requires`, um ambiente restrito, uma chave a mais ou o catálogo-base diferente: **pare e reporte**. **Anote as duas `description`**: elas substituem as dos testes da Task 1. Se uma `description` for **igual** ao nome do contrato, troque o nome antes de escrever os testes (regra do M2 do 33). Se o api ainda não tiver `PRACTICE_KEYS` com `min_gap_days`, as Tasks 1 e 2 seguem (testes com `fetch` falso) e a Task 3 espera.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod24b -b feat/mod-24b-home-visits origin/main
cd $R/.worktrees/maintenance-mod24b && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod24b
```

Expected: verde e `base-ok`. **Anote** a base (arquivos e casos). A Task 3 espera o **mesmo número de arquivos de teste**, com casos novos em `features.test.ts`, `FeaturesTab.test.tsx`, `careLineTemplates.test.ts` e `CareLineTemplateForm.test.tsx`.

**Relato ao coordenador:** "maintenance-mod24b: worktree e base verdes (N arquivos/M casos); entradas `home_visits`/`collective_activities` e `min_gap_days` do api conferidas; F-24.9 e F-24.14 em In Progress."

---

### Task 1: Nomes e pré-requisito de `home_visits` e `collective_activities`

**Files:**
- Modify: `src/lib/features.ts` (uma entrada em `MISSING_LABELS`, depois de `ombudsman_disabled`; duas em `FEATURE_LABELS`, depois de `ombudsman_anonymous_web`), `README.md` (parágrafo depois do de Cadastro individual e domiciliar, módulo 24a)
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("home_visits") === "Visita domiciliar e territorial"`, `featureLabel("collective_activities") === "Atividade coletiva"`, `missingText("territory_registry_disabled") === "cadastro individual e domiciliar (territory_registry) desligado ou incompleto"`.

- [ ] **Step 1: Write the failing tests**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 24b", () => {
  it("módulo 24b: nomes, frase do cadastro; missing desconhecido cru", () => {
    expect(featureLabel("home_visits")).toBe("Visita domiciliar e territorial");
    expect(featureLabel("collective_activities")).toBe("Atividade coletiva");
    expect(missingText("territory_registry_disabled")).toBe("cadastro individual e domiciliar (territory_registry) desligado ou incompleto");
    expect(missingText("field_devices_missing")).toBe("field_devices_missing");
  });

  it("home_visits falta o cadastro; collective_activities reaproveita a frase do prontuário", () => {
    expect(missingSummary([ "territory_registry_disabled" ])).toBe("cadastro individual e domiciliar (territory_registry) desligado ou incompleto");
    expect(missingSummary([ "clinical_record_disabled" ])).toBe("prontuário da atenção primária (clinical_record) desligado");
  });

  it("os nomes do 24a e do 33 não mudam", () => {
    expect(featureLabel("territory_registry")).toBe("Cadastro individual e domiciliar");
    expect(featureLabel("ombudsman_anonymous_web")).toBe("Denúncia anônima sem login (canal do cidadão)");
    expect(missingText("ombudsman_disabled")).toBe("ouvidoria do SUS (ombudsman) desligada");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (as descrições são as do contrato §1; use as anotadas na Task 0; a de `territory_registry` é a do bloco do 24a, `FeaturesTab.test.tsx:1438`):

```tsx
describe("FeaturesTab — visita domiciliar e atividade coletiva (módulo 24b)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const TERRITORY_PHRASE = "cadastro individual e domiciliar (territory_registry) desligado ou incompleto";
  const TR_ON: Feature = {
    key: "territory_registry",
    description: "Cadastro territorial: microáreas, domicílios, famílias e cadastro individual com fichas LEDI de cadastro",
    enabled: true, usable: true, missing: [], changedAt: "2026-11-12T12:00:00Z", changedBy: "dev@local"
  };
  const HV_OFF: Feature = {
    key: "home_visits",
    description: "Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo", // Q24
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const HV_ON: Feature = { ...HV_OFF, enabled: true, usable: true, changedAt: "2026-11-12T13:00:00Z", changedBy: "dev@local" };
  const CA_OFF: Feature = {
    key: "collective_activities",
    description: "Atividade coletiva com ficha LEDI, sem PSE",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const CA_ON: Feature = { ...CA_OFF, enabled: true, usable: true, changedAt: "2026-11-12T13:05:00Z", changedBy: "dev@local" };

  it("aparecem com o nome em português e a descrição do api", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ HV_OFF, CA_OFF ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("home_visits")).not.toBeNull();
    expect(screen.getByText("Visita domiciliar e territorial")).not.toBeNull();
    expect(screen.getByText(HV_OFF.description)).not.toBeNull();
    expect(screen.getByText("collective_activities")).not.toBeNull();
    expect(screen.getByText("Atividade coletiva")).not.toBeNull();
    expect(screen.getByText(CA_OFF.description)).not.toBeNull();
    expect(screen.getAllByText("desligada")).toHaveLength(2);
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("home_visits com o cadastro territorial utilizável: liga utilizável", async () => {
    const user = userEvent.setup();
    let listed = [ TR_ON, HV_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ TR_ON, HV_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: HV_ON } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar home_visits" }));
    expect((await screen.findByRole("status")).textContent).toBe("home_visits ligada e utilizável.");
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "home_visits", enabled: true });
    expect(screen.queryByText("ligada, falta pré-requisito")).toBeNull();
  });

  it("home_visits sem o cadastro territorial: ligada, falta o cadastro", async () => {
    const user = userEvent.setup();
    const HV_BLOCKED: Feature = { ...HV_ON, usable: false, missing: [ "territory_registry_disabled" ] };
    let listed = [ HV_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ HV_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: HV_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar home_visits" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "home_visits", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      `home_visits ligada, mas ainda não utilizável — falta: ${TERRITORY_PHRASE}.`
    );
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|apaga|exclu/i);
  });

  it("desligar territory_registry com home_visits ligada: só a chave pedida; a visita fica faltando o cadastro", async () => {
    const user = userEvent.setup();
    const TR_OFF_NOW: Feature = { ...TR_ON, enabled: false, usable: false, changedAt: "2026-11-12T15:00:00Z" };
    const HV_ORPHAN: Feature = { ...HV_ON, usable: false, missing: [ "territory_registry_disabled" ] };
    let listed = [ TR_ON, HV_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ TR_OFF_NOW, HV_ORPHAN ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: TR_OFF_NOW } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar territory_registry" }));
    await waitFor(() => expect(screen.getByText(TERRITORY_PHRASE)).not.toBeNull());
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "territory_registry", enabled: false });
    expect((await screen.findByRole("status")).textContent).toBe("territory_registry desligada.");
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
    expect(screen.queryByText("ligada e utilizável")).toBeNull();
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("collective_activities em cidade integrated: falta o prontuário, não o cadastro territorial", async () => {
    const user = userEvent.setup();
    const CA_BLOCKED: Feature = { ...CA_ON, usable: false, missing: [ "clinical_record_disabled" ] };
    let listed = [ CA_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ CA_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: CA_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "integrated" }));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar collective_activities" }));
    expect((await screen.findByRole("status")).textContent).toBe(
      "collective_activities ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    );
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "collective_activities", enabled: true });
    expect(screen.queryByText(TERRITORY_PHRASE)).toBeNull();
  });

  it("desligar home_visits não mexe em collective_activities", async () => {
    const user = userEvent.setup();
    const HV_OFF_NOW: Feature = { ...HV_OFF, changedAt: "2026-11-12T16:00:00Z", changedBy: "dev@local" };
    let listed = [ HV_ON, CA_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ HV_OFF_NOW, CA_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: HV_OFF_NOW } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar home_visits" }));
    expect((await screen.findByRole("status")).textContent).toBe("home_visits desligada.");
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "home_visits", enabled: false });
    await waitFor(() => expect(screen.getAllByText("ligada e utilizável")).toHaveLength(1));
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("visita e atividade ligadas: nenhum aviso de simulado", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ HV_ON, CA_ON ], { recordMode: "record" })));
    renderTab();

    expect(await screen.findByText("collective_activities")).not.toBeNull();
    expect(screen.getAllByText("ligada e utilizável")).toHaveLength(2);
    expect(screen.queryAllByRole("note")).toHaveLength(0);
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

- [ ] **Step 2: Run** `bin/test maintenance -w maintenance-mod24b src/lib/features.test.ts src/screens/FeaturesTab.test.tsx` (de `$R`). Expected: FAIL nos casos de nome (`featureLabel` devolve `null`) e de frase (`territory_registry_disabled` sai cru). Os casos de "só a chave pedida", do prontuário e de nenhum simulado já podem passar, porque fixam o mecanismo genérico.

- [ ] **Step 3: Implemente**
  - **`features.ts`, `MISSING_LABELS`:** depois de `ombudsman_disabled`, o comentário `// Módulo 24b (ADR 0052): home_visits exige territory_registry ligado e utilizável; sem IBGE, UF ou prontuário o cadastro fica ligado mas inutilizável, e o api manda só este código (não propaga o pré-requisito do pré-requisito).` e a entrada `territory_registry_disabled: "cadastro individual e domiciliar (territory_registry) desligado ou incompleto"`.
  - **`features.ts`, `FEATURE_LABELS`:** depois de `ombudsman_anonymous_web`, o comentário `// Módulo 24b (ADR 0052): home_visits exige territory_registry (missing: territory_registry_disabled, acima); collective_activities exige clinical_record (frase do 19b) e não depende do cadastro. Todos os ambientes, sem simulado.` e as entradas `home_visits: "Visita domiciliar e territorial"` e `collective_activities: "Atividade coletiva"`.
  - **`README.md`:** parágrafo **Visita domiciliar e atividade coletiva (módulo 24b).** logo depois do de Cadastro individual e domiciliar (módulo 24a):
    - `home_visits` ("Visita domiciliar e territorial") liga, no dashboard, a folha do dia do ACS e dos outros agentes de campo (com a ficha LEDI de visita), as visitas da equipe, a atribuição de microáreas pelo enfermeiro, "quem visitar", os aparelhos do app de campo e a sincronização `/field/v1`; e, no wpda, a visita realizada na linha do tempo e em "Meu acompanhamento";
    - exige `territory_registry` ligado e utilizável: sem ele, ou com ele ligado sem o prontuário, o IBGE ou a UF, a linha diz "cadastro individual e domiciliar (territory_registry) desligado ou incompleto";
    - `collective_activities` ("Atividade coletiva") liga a atividade coletiva com a ficha LEDI, sem PSE; exige `clinical_record` (só cidades em modo `record`) e não depende do cadastro territorial. A aba não liga nem desliga uma pela outra, nem desliga a visita quando o cadastro é desligado;
    - as duas existem em todos os ambientes, sem simulado, e as duas chegam ao cidadão;
    - desligar esconde as telas e bloqueia visita, atividade e sincronização novas; o que foi gravado fica, e as folhas de dias anteriores ainda abertas são fechadas pelo job da madrugada com a ficha gerada (o que foi visitado chega ao SIAPS);
    - microáreas, atribuições e aparelhos são da cidade, no dashboard; nenhum dado de cidadão nem token de aparelho passa por aqui;
    - ordem de deploy: api → dashboard → wpda → maintenance; contra um api sem o 24b, as chaves só não aparecem.

- [ ] **Step 4: Run** o mesmo comando e `cd $M && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the home visit and collective activity switches and their prerequisite

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod24b Task 1: nomes de `home_visits`/`collective_activities` e frase de `territory_registry_disabled` commitados (<sha>); escopo fechado verde."

---

### Task 2: Intervalo mínimo da boa prática e a visita do ACS acompanhada

**Files:**
- Modify: `src/lib/careLineTemplates.ts` (`PracticeJson`, `PracticeDraft`, `LIMITS`, `emptyPractice`, `practiceDraft`, `practiceProblem`, `draftToVariables`, `ruleText`, `PRACTICE_FIELDS`, `SOURCE_KIND_LABELS.acs_visit`), `src/screens/CareLineTemplateForm.tsx` (um `Field` por boa prática), `README.md` (uma frase no parágrafo do 24b)
- Test: `src/lib/careLineTemplates.test.ts`, `src/screens/CareLineTemplateForm.test.tsx`

**Interfaces:**
- Consumes: `toDraft`, `draftToVariables`, `draftProblem`, `ruleText`, `sourceText`, `upsertErrorText`, `TemplateDraft`, `TemplateView`, `PracticeJson` e os helpers `t`, `P`, `C4_A`, `REAL` (`careLineTemplates.test.ts`); `openNewVersion`, `practice`, `publish`, `upserts`, `form`, `bodyOf`, `upsertReply`, `fetchMock` (`CareLineTemplateForm.test.tsx`); `C4_A`, `C4_V1`, `C4_V2`, `reply` (`CareLineTemplates.fixtures.tsx`).
- Produces: `PracticeJson.min_gap_days?: number | null`; `PracticeDraft.minGapDays: string`; `LIMITS.minGapDays = 365`; `sourceText("acs_visit") === "visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)"`; `ruleText` com `intervalo mínimo de N dias`; caminho `practices[i].min_gap_days` → "(intervalo mínimo)".

- [ ] **Step 1: Write the failing tests**

Em `src/lib/careLineTemplates.test.ts`:

(a) Formas reais do catálogo-base do 24b (conferidas na Task 0 contra `$A/config/care_lines/catalog-base.yml`; troque o comentário `copiado de catalog-base.yml @454e502` por `copiado de catalog-base.yml do api do 24b (feat/mod-24b-home-visits)`). Nas seis práticas `acs_visit`, acrescente `citizen_label: "Visitas do agente de saúde"`, e na `c5_d` também `min_gap_days: 30`:

```ts
  P("c2_d", "2 visitas do ACS", "acs_visit", "birth", 180, 2, "end_of_window", 20, { citizen_label: "Visitas do agente de saúde" }),
  P("c3_e", "3 visitas do ACS", "acs_visit", "lmp", 294, 3, "end_of_window", 9, { citizen_label: "Visitas do agente de saúde" }),
  P("c3_j", "Visita do ACS no puerpério", "acs_visit", "lmp", 336, 1, "end_of_window", 9, { window_start_days: 294, citizen_label: "Visitas do agente de saúde" }),
  P("c4_d", "2 visitas do ACS em 12 meses", "acs_visit", "rolling", 365, 2, "end_of_window", 20, { citizen_label: "Visitas do agente de saúde" }),
  P("c5_d", "2 visitas do ACS nos últimos 12 meses", "acs_visit", "rolling", 365, 2, "end_of_window", 25, { citizen_label: "Visitas do agente de saúde", min_gap_days: 30 })
  P("c6_c", "2 visitas do ACS", "acs_visit", "rolling", 365, 2, "end_of_window", 25, { citizen_label: "Visitas do agente de saúde" }),
```

(cada linha substitui a de mesma chave, mantendo a vírgula ou a falta dela que a linha original tinha no fim da lista).

(b) Em `describe("rótulos")`, troque a expectativa de `acs_visit`:

```ts
    expect(sourceText("acs_visit")).toBe("visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)");
```

(c) No fim do arquivo:

```ts
describe("módulo 24b: intervalo mínimo entre eventos", () => {
  const ok = (): TemplateDraft => toDraft(t({ version: 2, current: true }));
  const practice = (patch: object) => {
    const d = ok();
    return draftProblem({ ...d, practices: [ { ...d.practices[0], ...patch, points: "100" } ] });
  };

  it("rascunho lê min_gap_days para o campo e a ida e volta devolve o número", () => {
    const view = t({ version: 2, current: true, practices: [ { ...C4_A, points: 85, min_gap_days: 30 }, { ...C4_E, points: 15 } ] });
    const d = toDraft(view);
    expect(d.practices[0].minGapDays).toBe("30");
    expect("min_gap_days" in d.practices[0].advanced).toBe(false);
    expect(d.practices[1].minGapDays).toBe("");
    const sent = draftToVariables(d).practices;
    expect(sent[0].min_gap_days).toBe(30);
    expect("min_gap_days" in sent[1]).toBe(false);
  });

  it("campo vazio não manda a chave", () => {
    expect(draftToVariables(ok()).practices.every((p) => !("min_gap_days" in p))).toBe(true);
  });

  it("intervalo mínimo fora de 1 a 365 ou não inteiro é recusado", () => {
    expect(practice({ minGapDays: "0" })).toBe("boa prática 1: intervalo mínimo de 1 a 365 dias");
    expect(practice({ minGapDays: "366" })).toBe("boa prática 1: intervalo mínimo de 1 a 365 dias");
    expect(practice({ minGapDays: "30,5" })).toBe("boa prática 1: intervalo mínimo de 1 a 365 dias");
    expect(practice({ minGapDays: "x" })).toBe("boa prática 1: intervalo mínimo de 1 a 365 dias");
    expect(practice({ minGapDays: "" })).toBeNull();
    expect(practice({ minGapDays: "30" })).toBeNull();
    expect(practice({ minGapDays: "365" })).toBeNull();
  });

  it("a regra exibida diz o intervalo; sem a chave, nada muda", () => {
    expect(ruleText({ ...C4_A, min_gap_days: 30 })).toContain("intervalo mínimo de 30 dias");
    expect(ruleText(C4_A)).not.toContain("intervalo");
  });

  it("recusa do api no campo novo cita a boa prática e o campo", () => {
    expect(upsertErrorText([ { path: "practices[0].min_gap_days", message: "invalid_practices" } ]))
      .toBe("boas práticas inválidas — confira a boa prática 1 (intervalo mínimo)");
  });
});
```

Em `src/screens/CareLineTemplateForm.test.tsx`, dentro de `describe("CareLineTemplateForm")`, depois do caso "visita do ACS: o envio sai sem código…" (acrescente `C4_A` ao `import` das fixtures, que já traz `C4_V1` e `C4_V2`):

```tsx
  it("intervalo mínimo (módulo 24b): 0 é recusado antes de enviar; 30 vai como min_gap_days só na prática editada", async () => {
    const user = userEvent.setup();
    await openNewVersion(user);
    const gap = within(practice(1)).getByLabelText("Intervalo mínimo entre eventos (dias)") as HTMLInputElement;
    expect(gap.value).toBe("");
    await user.type(gap, "0");
    await publish(user);
    expect(await within(form()).findByText(/intervalo mínimo de 1 a 365 dias/)).not.toBeNull();
    expect(upserts()).toHaveLength(0);
    await user.clear(gap);
    await user.type(gap, "30");
    await publish(user);
    await waitFor(() => expect(upserts()).toHaveLength(1));
    const sent = bodyOf(upserts()[0]).variables.practices as (SentPractice & { min_gap_days?: number })[];
    expect(sent[0].min_gap_days).toBe(30);
    expect("min_gap_days" in sent[1]).toBe(false);
  });

  it("versão com intervalo abre o campo preenchido, não o lista como avançado e publica igual", async () => {
    const user = userEvent.setup();
    const WITH_GAP = { ...C4_V2, practices: [ { ...C4_A, min_gap_days: 30 }, C4_V2.practices[1] ] };
    fetchMock.mockImplementation((_url: string, init: RequestInit) => {
      const body = JSON.parse(init.body as string) as Body;
      if (body.query.includes("mutation UpsertCareLineTemplate")) return Promise.resolve(upsertReply());
      return Promise.resolve(reply(200, { data: { careLineTemplates: [ WITH_GAP, C4_V1 ] } }));
    });
    await openNewVersion(user);
    expect((within(practice(1)).getByLabelText("Intervalo mínimo entre eventos (dias)") as HTMLInputElement).value).toBe("30");
    expect(within(practice(1)).queryByText(/min_gap_days/)).toBeNull();
    await publish(user);
    await waitFor(() => expect(upserts()).toHaveLength(1));
    expect((bodyOf(upserts()[0]).variables.practices as { min_gap_days?: number }[])[0].min_gap_days).toBe(30);
  });
```

- [ ] **Step 2: Run** `bin/test maintenance -w maintenance-mod24b src/lib/careLineTemplates.test.ts src/screens/CareLineTemplateForm.test.tsx` (de `$R`). Expected: FAIL — `sourceText("acs_visit")` com o texto antigo; `minGapDays` `undefined`; `min_gap_days` preso em `advanced`; "0" aceito; `ruleText` sem o intervalo; o caminho sai cru (`practices[0].min_gap_days`); o formulário sem o campo. A ida e volta com as formas reais (a) já pode passar (a chave desconhecida volta por `advanced`).

- [ ] **Step 3: Implemente**
  - **`careLineTemplates.ts`:**
    - `PracticeJson`: `min_gap_days?: number | null` (depois de `min_count`);
    - `SOURCE_KIND_LABELS.acs_visit`: `"visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)"`, com o comentário `// Módulo 24b (contrato §12): acs_visit passa a ser rastreada com home_visits utilizável na cidade; sem ele, segue "não acompanhado" no dashboard.`;
    - `ruleText`: antes do `if` de `points`, `if (typeof p.min_gap_days === "number") parts.push(\`intervalo mínimo de ${p.min_gap_days} dias\`);`;
    - `PracticeDraft`: `minGapDays: string` (depois de `minCount`); `LIMITS`: `minGapDays: 365`; `emptyPractice`: `minGapDays: ""`;
    - `practiceDraft`: tire `min_gap_days: _g` do resto (`...unknown`) e preencha `minGapDays: numText(p.min_gap_days)`;
    - `practiceProblem`: depois da regra de `minCount`, `const gapRaw = p.minGapDays.trim(); if (gapRaw !== "") { const g = intOf(gapRaw); if (g === null || g < 1 || g > LIMITS.minGapDays) return \`${n}: intervalo mínimo de 1 a ${LIMITS.minGapDays} dias\`; }`;
    - `draftToVariables`: `const gap = numberOrUndefined(p.minGapDays);` e, depois de `min_count`, `...(gap !== undefined ? { min_gap_days: gap } : {}),`;
    - `PRACTICE_FIELDS`: `min_gap_days: "intervalo mínimo"`.
  - **`CareLineTemplateForm.tsx`:** depois do `Field` "Quantidade mínima", `<Field label="Intervalo mínimo entre eventos (dias)" value={p.minGapDays} onChange={(v) => setPractice(i, { minGapDays: v })} helpText="vazio = sem intervalo; de 1 a 365 (C5: 30 dias entre as visitas do ACS)" />`.
  - **`README.md`:** no fim do parágrafo do 24b (Task 1), a frase: "Na tela **Linhas de cuidado**, a boa prática ganha **Intervalo mínimo entre eventos (dias)** (`min_gap_days`, de 1 a 365): dois eventos só contam com esse intervalo. A versão nova do catálogo-base (runbook `operacao/rollout-visita-e-coletiva.md`) põe 30 dias na `c5_d` e o texto para o cidadão "Visitas do agente de saúde" nas seis práticas de visita do ACS; publicar não muda nenhuma cidade até ela adotar no dashboard. A fonte "visita do ACS" passa a ser acompanhada nas cidades com `home_visits` ligado."

- [ ] **Step 4: Run** o mesmo comando e `cd $M && npx tsc --noEmit`. Expected: PASS, inclusive a ida e volta com as formas reais do catálogo-base do 24b.

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/lib/careLineTemplates.ts src/lib/careLineTemplates.test.ts src/screens/CareLineTemplateForm.tsx src/screens/CareLineTemplateForm.test.tsx README.md
$G commit -m "feat: edit the minimum gap between care line events and track the ACS visit source

The base catalog version of module 24b sets min_gap_days on c5_d, which
the practice form could only preserve, not set. The form now edits it
(1 to 365 days, empty sends nothing), the rule text shows it and the api
refusal path names it. The ACS visit source label no longer says it is
untracked.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "maintenance-mod24b Task 2: campo `min_gap_days` e rótulo de `acs_visit` commitados (<sha>); escopo fechado verde."

---

### Task 3: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, codegen e build**

Run (de `$R`): `bin/test maintenance -w maintenance-mod24b && cd $M && npx tsc --noEmit && npm run codegen:check && npm run build && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com o mesmo número de arquivos de teste da base;
- `codegen:check` limpo (o SDL não mudou);
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 12, 14 e 22, F-24.9, F-24.14) e o contrato §1, §12 e §14. Ele confere:
  - só duas entradas novas em `FEATURE_LABELS` e uma em `MISSING_LABELS`;
  - nomes diferentes das descrições do api;
  - nenhum aviso de simulado; nenhum dado de cidade aqui;
  - `min_gap_days` fora de `advanced`, sem chave quando vazio, igual na ida e volta;
  - que o README não promete apagar nada.

  Confira também: `grep -n -i "apaga\|exclu" $M/README.md | grep -i "visita\|atividade\|aparelho\|folha"` e `grep -n "não acompanhada pelo sistema" $M/src/lib/careLineTemplates.ts`. Expected: sem saída no primeiro; no segundo, só as linhas de `vaccine` e `oral_health` (M-3).

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 24b na **3057** (banco de dev migrado **com autorização do usuário**). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod24b --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod24b:/app -v maintenance-mod24b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3057 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod24b   # espere o "ready"
```

Abra `http://maintenance.localhost:5177`. O usuário faz login e TOTP; anote antes o estado de `territory_registry`, `home_visits` e `collective_activities` nas duas cidades. Guarde os screenshots em `evidencias/2026-11-12-modulo-24b/maintenance/`. Roteiro:
1. Curitiba (`record`) → Funcionalidades: `home_visits` e `collective_activities` com nome e descrição.
2. Com `territory_registry` ligado e utilizável: ligar `home_visits` → "ligada e utilizável". No dashboard, o grupo "Visita e atividade coletiva" aparece para o ACS.
3. Desligar `territory_registry` (só se o usuário autorizar; senão pule): `home_visits` continua ligada e passa a "ligada, falta pré-requisito" com "cadastro individual e domiciliar (territory_registry) desligado ou incompleto"; religue o cadastro e a visita volta a utilizável sem novo clique.
4. Ligar `collective_activities` → "ligada e utilizável" (Curitiba tem prontuário). Desligar `home_visits` não mexe nela.
5. Maringá (`integrated`): ligar `collective_activities` → falta "prontuário da atenção primária (clinical_record) desligado"; ligar `home_visits` → falta a frase do cadastro. Desligue as duas de volta.
6. **Linhas de cuidado** → C5 → versões → nova versão: a boa prática `c5_d` mostra **Intervalo mínimo entre eventos (dias)**; a fonte diz "visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)"; "0" é recusado antes de enviar. **Publicar** a versão nova do catálogo-base (30 na `c5_d` e "Visitas do agente de saúde" nas práticas `acs_visit` de C2–C6, contrato §12) só com autorização do usuário: a versão é imutável e vale para todas as cidades do ambiente de dev (M-5). Sem autorização, cancele.
7. **Auditoria** lista as tentativas de `setCityFeature` (só a chave e o estado) e, se houve publicação, `upsertCareLineTemplate`.
8. A 375 px, sem rolagem lateral da página, inclusive o formulário da boa prática.

Depois, volte as chaves ao estado anotado (ou ao que as provas do dashboard e do wpda precisarem) e limpe: `docker rm -f maintenance-mod24b && docker volume rm maintenance-mod24b-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** O merge só vem depois do api, do dashboard e do wpda do 24b (a suíte completa do api é da sessão "Testes", pedida pelo plano do api), e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas). Quando autorizado: `git fetch`, `merge --ff-only feat/mod-24b-home-visits`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api e do dashboard do F-24.9 estiverem **Done**, mova o F-24.9 para **Done** (opção `98236657`), e para **Verified** se as provas passaram; o F-24.14 segue o api, o dashboard e o wpda.

**Relato ao coordenador:** "maintenance-mod24b Task 3: suíte/codegen/build verdes, revisão limpa, prova na 5177 feita (evidências em evidencias/2026-11-12-modulo-24b/maintenance/; catálogo-base <publicado|não publicado>); aguardando autorização de merge."

---

## Divergências propostas ao contrato

> **Decididas na §16 do contrato (2026-11-12):** M-1, M-3 → Q29 (`vaccine`/`oral_health` em card no fechamento); M-2, M-4, M-5 → Q19; M-6 → Q24.

1. **M-1 — Frase do pré-requisito.** O §14 fixa `territory_registry_disabled` → "cadastro territorial (territory_registry) desligado ou incompleto". No maintenance, o interruptor se chama "Cadastro individual e domiciliar" (L16 do 24a, `features.ts:112`), e as outras frases `<chave>_disabled` usam o nome do interruptor (`features.ts:37`, `:65`, `:76`). Proposta: "cadastro individual e domiciliar (territory_registry) desligado ou incompleto". Mantido o "ou incompleto" do contrato (C1): o api manda o mesmo código com o cadastro ligado sem IBGE, UF ou prontuário. Se o coordenador mantiver o texto do §14, a troca é em três lugares (`features.ts`, `features.test.ts`, `TERRITORY_PHRASE` em `FeaturesTab.test.tsx`).
2. **M-2 — `min_gap_days` não cabe na tela que já existe.** O §14 diz que "a versão nova do catálogo-base usa a tela que já existe", e o §12 manda a cidade existente receber a C5 nova "pelo maintenance". O formulário só preserva chave desconhecida (`advanced`); não há como **pôr** `min_gap_days: 30` na `c5_d`. Proposta: o campo da Task 2 (pequeno, e serve para as notas de C2, C3, C4 e C6 quando lidas). Alternativa, se o coordenador preferir: o api publica as versões novas do `catalog-base.yml` por uma tarefa de rollout, e a Task 2 fica só com `ruleText`, o caminho de erro e o rótulo.
3. **M-3 — Rótulo da fonte `acs_visit`.** "visita do ACS (não acompanhada pelo sistema)" (`careLineTemplates.ts:61`) fica falso com o 24b. Proposta: "visita do ACS (acompanhada quando a visita domiciliar está ligada na cidade)". Achado fora do escopo: `vaccine` ("não acompanhada pelo sistema") já é acompanhada com `vaccination` desde o 29, e `oral_health` precisa ser conferida contra o 28; proposta de card no board #2, sem mexer aqui.
4. **M-4 — Superfícies do F-24.14.** A spec §8 lista F-24.14 em api, dashboard e wpda. O campo `min_gap_days` e a publicação da versão nova são do maintenance. Proposta: incluir o maintenance em F-24.14 (o card só vai a Done com a Task 2 deste plano entregue).
5. **M-5 — Quem publica o catálogo-base em dev, e quando.** A ordem do §15 põe o maintenance por último, mas as provas do dashboard e do wpda ("Meu acompanhamento" com "Visitas do agente de saúde" e a C5 com 30 dias) precisam da versão nova publicada e adotada pela cidade. Proposta: as Tasks 1 e 2 deste plano podem rodar antes da prova do dashboard; a publicação em dev é feita por esta sessão, na 5177, com autorização do usuário, antes das provas do dashboard e do wpda; ou o coordenador decide que o api semeia a versão nova em dev.
6. **M-6 — Nome e descrição parecidos.** A descrição de `home_visits` começa pelo próprio nome ("Visita domiciliar e territorial: folha do dia, …"), e a aba mostra os dois lado a lado. Não quebra teste (`getByText` casa o texto inteiro), mas repete o texto na linha. Proposta, opcional: a descrição do §1 começar pelo conteúdo ("Folha do dia, ficha LEDI de visita, quem visitar e sincronização do app de campo").

## Self-review

- **Cobertura (Afeta, decisões 12, 14 e 22, F-24.9, F-24.14; contrato §1, §12, §14, §15, C1, C14, C18):**
  - nomes e frase do pré-requisito: Task 1 (Review Focus 5; M-1);
  - visita presa ao cadastro e sem cascata: Task 1, terceiro e quarto casos (Review Focus 1);
  - atividade independente do cadastro: Task 1, quinto e sexto casos (Review Focus 2);
  - desligar sem apagar e o job da madrugada: README da Task 1 (Review Focus 4);
  - sem simulado: Task 1, sétimo caso;
  - catálogo-base com `min_gap_days` e `citizen_label`: Task 2 (Review Focus 3; M-2, M-5), com as formas reais conferidas na Task 0;
  - rótulo de `acs_visit`: Task 2 (M-3);
  - GraphQL e `FeaturesTab`: conferidos, nada a mudar (`codegen:check` na Task 3);
  - suíte, build e prova (Curitiba, Maringá, catálogo-base, 375 px): Task 3; cards: Tasks 0 e 3.
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards, as `description` (Task 0, com a regra para nome igual) e as práticas do catálogo-base do api (Task 0, que manda parar se diferirem).
- **Tipos e nomes:** existem em `origin/main` (`9269b55`): `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus`, `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature`; `toDraft`, `draftToVariables`, `draftProblem`, `ruleText`, `sourceText`, `upsertErrorText`, `intOf`, `numText`, `numberOrUndefined`, `LIMITS`, `PRACTICE_FIELDS`, `SOURCE_KIND_LABELS`, `openNewVersion`, `practice`, `publish`, `upserts`, `form`, `SentPractice`, `C4_A`, `C4_E`, `C4_V1`, `C4_V2`. No api (`d36aef3`): `Platform::Features::CATALOG`, `missing_for`, `usable?`, `CITIZEN_HIDDEN`, `CareLines::Rules::PRACTICE_KEYS` (o `min_gap_days` vem do api do 24b).
- **Proporção:** os blocos de código são só de testes; a mudança de produção são três linhas em `features.ts`, uns dez pontos pequenos em `careLineTemplates.ts`, um `Field` e dois parágrafos de README.
