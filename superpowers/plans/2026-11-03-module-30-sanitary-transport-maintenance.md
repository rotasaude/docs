# Módulo 30 — Transporte sanitário e TFD (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 30** na branch `feat/mod-30-sanitary-transport` (worktree `apps/api/.claude/mod30`, servidor de dev na porta **3052**) com as duas entradas em `Platform::Features::CATALOG` (contrato §1): `sanitary_transport` (`requires: []`) e `tfd` (`requires: %w[feature:sanitary_transport feature:clinical_record]`). Contrato §13: api → dashboard → wpda → **maintenance**. As Tasks 0 e 1 não dependem do api rodando (testes com `fetch` falso); a prova da Task 2 precisa dele.

**Goal:** Lado maintenance de F-30.1: os dois interruptores do módulo aparecem na aba Funcionalidades com nome em português, o pré-requisito novo (`sanitary_transport_disabled`) aparece em frase ao lado do que já existe (`clinical_record_disabled`), e o liga/desliga genérico fica fixado em teste para as duas chaves, inclusive o caso de desligar o transporte com o TFD ligado.

**Architecture:** Conferido no código (`origin/main` do maintenance, `6b788fc`): o liga/desliga é genérico (`setCityFeature`, três estados, `missing` vindo do api via `Platform::Features#missing_for`, que gera `<chave>_disabled` para `feature:<chave>`), mas **o catálogo do api não basta** — sem mudança aqui as duas chaves aparecem cruas e `sanitary_transport_disabled` também (a aba mostra o código, não quebra). Muda só `src/lib/features.ts` (`FEATURE_LABELS` e `MISSING_LABELS`) e o `README.md`. Nenhuma tela nova, nenhum SDL novo, nenhum quadro de plataforma e **nenhum aviso de simulado**: o módulo 30 não tem chave simulada nem credencial (diferente do 29).

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md` (§2 decisão 23, §7, §8 F-30.1), `docs/adr/0048.md` e o contrato `docs/superpowers/plans/2026-11-03-module-30-sanitary-transport-contracts.md` §1, §10 e §13. Base: plano do maintenance do 29 (`2026-11-02-module-29-vaccination-room-maintenance.md`) e o do 28. As diferenças estão em "Divergências propostas ao contrato", M1–M4.

## Global Constraints

- Interruptores (spec decisão 23; contrato §1), pelo mecanismo genérico (`City.features`, `setCityFeature`, `Platform::Features#missing_for`):
  - `sanitary_transport` sem pré-requisito (nem o modo de prontuário): ligado é utilizável;
  - `tfd` exige `sanitary_transport` e `clinical_record` ligados e utilizáveis → `missing` `sanitary_transport_disabled` e `clinical_record_disabled` (frase existente do 19b, não redefinida); cidade `integrated` ou `off` nunca tem o TFD utilizável (o laudo nasce da consulta do 19a);
  - ligar ou desligar uma chave só manda aquela `key`: a aba nunca liga pré-requisito junto nem desliga dependente junto.
- Nomes (M1): `sanitary_transport` → "Transporte sanitário"; `tfd` → "Tratamento Fora de Domicílio (TFD)".
- Desligar não apaga nada: frota, viagens, presença, processos de TFD, pagamentos e linhas de BPA ficam (a exclusão LGPD retém o TFD, ADR 0026). A aba não diz o contrário.
- O maintenance não vê nada de cidade: nenhuma viagem, passageiro, CNH, laudo, CID, valor ou cidadão passa por aqui; papéis (`transport_dispatcher`, `tfd_committee`, `driver`), destinos, CNES do TFD e complementos são do dashboard.
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null`.
- Interface (ciclo de interface, subprojeto 5): nenhuma tela nova; as guardas de tema (`src/theme/*.test.ts`) rodam na suíte; a tabela rola dentro do `TableScroll`. Achado só visual não vira tarefa.
- Âncora de `FEATURE_LABELS`: a última entrada em `origin/main` na hora da execução (hoje `rnds_mock`); conflito no rebase com outro módulo é só ordem de linhas.
- Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.
- Primeira ação da execução: os cards F-30.1..F-30.8 do board #1 em In Progress (Task 0). O maintenance é, com o api e o dashboard, dono do F-30.1: Done quando os três estiverem entregues; Verified depois da prova no navegador.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod30`, `A=$R/apps/api/.claude/mod30`.
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-30-sanitary-transport`. Caminhos das tasks relativos a `$M`.
- Testes: `bin/test maintenance -w maintenance-mod30 <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`. `globals: false`, **sem jest-dom**.
- Prova: Vite do worktree **na 5177** com o serviço `maintenance` do compose parado; `http://maintenance.localhost:5177`, nunca `localhost:5177`; conta `dev@local`; o usuário faz login e TOTP. Banco de dev só migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Cidade em modo `integrated` ou `off`** (sem prontuário próprio). Ligar `tfd` dá "ligada, falta pré-requisito" com "prontuário da atenção primária (clinical_record) desligado" — nunca "utilizável". Teste: Task 1, "tfd sem transporte e sem prontuário: ligada, falta os dois, em português".
2. **Desligar o transporte com o TFD ligado.** Só `sanitary_transport` vai na mutation; depois do refetch a linha do TFD diz "ligada, falta pré-requisito" com "transporte sanitário (sanitary_transport) desligado" — o TFD não é desligado junto. Teste: Task 1, "desligar sanitary_transport com tfd ligado: só a chave pedida; o tfd fica faltando".
3. **Transporte sem nenhum pré-requisito.** Em cidade `integrated`, `sanitary_transport` liga utilizável (a frota não depende do prontuário). Teste: Task 1, "sanitary_transport em cidade integrated: liga utilizável e desliga com dois cliques".
4. **Api mais novo que a tela** (`missing` desconhecido, como um futuro `tfd_billing_unit_missing`). Aparece cru. Teste: Task 1, "módulo 30: nomes; missing desconhecido cru".
5. **Texto que prometa apagar ao desligar.** Nenhuma frase da aba nem do README diz que desligar apaga viagens, TFD ou pagamentos. Testes: Task 1, os dois casos da aba (`not.toMatch(/apaga|exclu/i)`).

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, entradas do api, worktree, base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`, `README.md` | nomes e frase do pré-requisito; fluxo genérico fixado para `sanitary_transport` e `tfd` | 1 |
| — | suíte, codegen:check, build, revisão, prova na 5177 | 2 |

**Estratégia de teste:** funções puras (`featureLabel`, `missingText`, `missingSummary`) e a aba pelo fluxo existente de dois cliques, com `fetch` falso por nome de operação e conferência das variáveis exatas da mutation, incluindo o refetch que mostra o dependente faltando (Task 1); suíte inteira com as guardas de tema, `codegen:check`, build e a prova contra o api do 30 (Task 2).

---

### Task 0: Cards, âncoras, entradas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`6b788fc` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); worktree do api `$A`.
- Produces: cards em In Progress; `$M`; a base anotada; as duas `description` reais do api anotadas (usadas nos testes).

- [ ] **Step 1: Cards F-30.1..F-30.8 para In Progress** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-30\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os oito em In Progress ou além. Card ausente → **pare e reporte**.

- [ ] **Step 2: Âncoras e entradas do api**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod30
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -3 origin/main
$G -C $R/apps/maintenance show origin/main:src/lib/features.ts | grep -n 'clinical_record_disabled:\|vaccination_catalog_under_review:\|city_unreachable:\|rnds_mock: "RNDS simulada\|sanitary\|tfd'
grep -n 'key: "sanitary_transport"\|key: "tfd"' -A3 $A/app/services/platform/features.rb
```

Expected: em `features.ts`, `clinical_record_disabled` traduzido, nenhuma linha com `sanitary`/`tfd`; a última entrada de `FEATURE_LABELS` é `rnds_mock` (ou a de um módulo que entrou depois — use-a como âncora). No api, as duas entradas com os `requires` da Global Constraints e sem `environments` (existem em todos os ambientes). Outro `requires`, ambiente ou uma terceira chave do 30 → **pare e reporte**. **Anote as duas `description`**: substituem os rascunhos da Task 1.

- [ ] **Step 3: Worktree fora de `apps/` e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod30 -b feat/mod-30-sanitary-transport origin/main
cd $R/.worktrees/maintenance-mod30 && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod30
```

Expected: verde e `base-ok`. **Anote** a base: a Task 2 espera o **mesmo número de arquivos de teste** e casos novos em `features.test.ts` e `FeaturesTab.test.tsx`.

---

### Task 1: Nomes e pré-requisitos dos dois interruptores

**Files:**
- Modify: `src/lib/features.ts`, `README.md`
- Test: `src/lib/features.test.ts`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `featureLabel`, `missingText`, `missingSummary` (`features.ts`); `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` (topo de `FeaturesTab.test.tsx`).
- Produces: `featureLabel("sanitary_transport" | "tfd")` com os nomes da Global Constraints; `missingText("sanitary_transport_disabled")`.

- [ ] **Step 1: Write the failing test**

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 30", () => {
  it("módulo 30: nomes; missing desconhecido cru", () => {
    expect(featureLabel("sanitary_transport")).toBe("Transporte sanitário");
    expect(featureLabel("tfd")).toBe("Tratamento Fora de Domicílio (TFD)");
    expect(missingText("tfd_billing_unit_missing")).toBe("tfd_billing_unit_missing");
  });

  it("tfd: transporte e prontuário em português, sem redefinir a frase do 19b", () => {
    expect(missingSummary([ "sanitary_transport_disabled", "clinical_record_disabled" ]))
      .toBe("transporte sanitário (sanitary_transport) desligado; prontuário da atenção primária (clinical_record) desligado");
  });
});
```

No fim de `src/screens/FeaturesTab.test.tsx` (descrições = rascunho; use as anotadas na Task 0):

```tsx
describe("FeaturesTab — transporte sanitário e TFD (módulo 30)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const SAN_OFF: Feature = {
    key: "sanitary_transport", description: "Transporte sanitário: frota, motoristas, destinos, viagens com lista de passageiros, séries recorrentes e tela do motorista",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const SAN_ON: Feature = { ...SAN_OFF, enabled: true, usable: true, changedAt: "2026-11-03T13:00:00Z", changedBy: "dev@local" };
  const TFD_OFF: Feature = {
    key: "tfd", description: "Tratamento Fora de Domicílio: laudo, comissão, trechos, ajuda de custo, pagamento registrado, prestação de contas e BPA",
    enabled: false, usable: false, missing: [ "sanitary_transport_disabled", "clinical_record_disabled" ], changedAt: null, changedBy: null
  };
  const TFD_ON: Feature = { ...TFD_OFF, enabled: true, usable: true, missing: [], changedAt: "2026-11-03T13:05:00Z", changedBy: "dev@local" };

  it("tfd sem transporte e sem prontuário: ligada, falta os dois, em português", async () => {
    const user = userEvent.setup();
    const TFD_BLOCKED: Feature = { ...TFD_OFF, enabled: true, changedAt: "2026-11-03T13:00:00Z", changedBy: "dev@local" };
    let listed = [ SAN_OFF, TFD_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ SAN_OFF, TFD_BLOCKED ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: TFD_BLOCKED } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("Transporte sanitário")).not.toBeNull();
    expect(screen.getByText("Tratamento Fora de Domicílio (TFD)")).not.toBeNull();
    await user.click(screen.getAllByRole("button", { name: "ligar" })[1]);
    await user.click(await screen.findByRole("button", { name: "confirmar: ligar tfd" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "tfd", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "tfd ligada, mas ainda não utilizável — falta: transporte sanitário (sanitary_transport) desligado; prontuário da atenção primária (clinical_record) desligado."
    );
    expect(document.body.textContent).not.toMatch(/undefined|apaga|exclu/i);
  });

  it("sanitary_transport em cidade integrated: liga utilizável e desliga com dois cliques", async () => {
    const user = userEvent.setup();
    let listed = [ SAN_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ body.variables.enabled === true ? SAN_ON : SAN_OFF ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: listed[0] } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar sanitary_transport" }));
    expect((await screen.findByRole("status")).textContent).toBe("sanitary_transport ligada e utilizável.");
    await user.click(await screen.findByRole("button", { name: "desligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: desligar sanitary_transport" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(2));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[1]).variables).toEqual({ citySlug: "sp", key: "sanitary_transport", enabled: false });
    expect(document.body.textContent).not.toMatch(/apaga|exclu/i);
  });

  it("desligar sanitary_transport com tfd ligado: só a chave pedida; o tfd fica faltando", async () => {
    const user = userEvent.setup();
    const TFD_ORPHAN: Feature = { ...TFD_ON, usable: false, missing: [ "sanitary_transport_disabled" ] };
    let listed = [ SAN_ON, TFD_ON ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ SAN_OFF, TFD_ORPHAN ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: SAN_OFF } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    await user.click((await screen.findAllByRole("button", { name: "desligar" }))[0]);
    await user.click(screen.getByRole("button", { name: "confirmar: desligar sanitary_transport" }));
    await waitFor(() => expect(screen.getByText("transporte sanitário (sanitary_transport) desligado")).not.toBeNull());
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1);
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables).toEqual({ citySlug: "sp", key: "sanitary_transport", enabled: false });
    expect(screen.getByText("ligada, falta pré-requisito")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test maintenance -w maintenance-mod30 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `featureLabel("sanitary_transport")` é `null` e `missingText("sanitary_transport_disabled")` volta cru. (Fluxo de dois cliques e `clinical_record_disabled` já passam.)

- [ ] **Step 3: Implemente em `src/lib/features.ts`**

Em `MISSING_LABELS`, antes de `city_unreachable` (comentário "Módulo 30 (ADR 0048): tfd exige sanitary_transport e clinical_record (frase do 19b, acima)"): `sanitary_transport_disabled: "transporte sanitário (sanitary_transport) desligado"`. Em `FEATURE_LABELS`, depois da última entrada (vírgula na anterior), com comentário "Módulo 30 (ADR 0048): sanitary_transport sem pré-requisito; tfd exige sanitary_transport e clinical_record (o laudo nasce da consulta). Todos os ambientes, sem simulado.": `sanitary_transport: "Transporte sanitário"`, `tfd: "Tratamento Fora de Domicílio (TFD)"`.

`README.md`, depois do parágrafo **Sala de vacina (módulo 29).** e antes de **Documentos clínicos (módulo 19c).** — parágrafo **Transporte sanitário e TFD (módulo 30).**: as duas chaves e seus nomes; `sanitary_transport` liga no dashboard a central de transporte (frota, motoristas, destinos, viagens, séries, lista de passageiros) e a tela do motorista, sem pré-requisito; `tfd` liga o laudo, a comissão, os trechos, a ajuda de custo, o pagamento registrado, a prestação de contas e a produção no BPA, e exige `sanitary_transport` e `clinical_record` (só cidades em modo `record`); a aba nunca liga um pelo outro nem desliga o TFD junto com o transporte; papéis, destinos e o CNES do TFD são configurados no dashboard; existem em todos os ambientes e não têm simulado; desligar esconde as telas, viagens, processos e pagamentos ficam (a exclusão LGPD retém o TFD); nenhum dado de cidadão passa por aqui; ordem de deploy api → dashboard → wpda → maintenance; contra um api sem o 30, as chaves só não aparecem.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test maintenance -w maintenance-mod30 src/lib/features.test.ts src/screens/FeaturesTab.test.tsx src/theme && cd $M && npx tsc --noEmit`
Expected: PASS (testes antigos da aba iguais).

- [ ] **Step 5: Commit**

```bash
cd $M
$G add src/lib/features.ts src/lib/features.test.ts src/screens/FeaturesTab.test.tsx README.md
$G commit -m "feat: name the sanitary transport and TFD switches and their prerequisites

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: Task 1; o api do 30 na **3052**.
- Produces: branch pronta para merge (depois do api, do dashboard e do wpda do 30).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `bin/test maintenance -w maintenance-mod30 && cd $M && npm run typecheck && npm run codegen:check && npm run build && $G status --short && $G diff origin/main --stat`
Expected: verde; o mesmo número de arquivos de teste da base; `status` vazio; o diff só com os quatro arquivos da Task 1 (nada em `schema.graphql`, `src/gql/` nem `FeaturesTab.tsx`). Número de arquivos que dobra = artefato de build descoberto pelo vitest: pare e reporte.

- [ ] **Step 2: Revisão**

Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 23, §7, F-30.1), o ADR 0048 e o contrato §1, §10, §13. Atenção: nenhuma linha nova no fluxo de `setCityFeature`; nenhuma frase redefinida para `clinical_record_disabled`; nenhum aviso de simulado; o README não promete GPS, aplicativo do motorista, pagamento na tesouraria nem CNRAC (fora da entrega); nenhum dado de cidade aqui.

- [ ] **Step 3: Prova no navegador na 5177 (com o usuário)**

Depende do api do 30 na **3052** (banco de dev migrado **com autorização do usuário**). Suba o Vite do worktree:

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml stop maintenance
docker run -d --rm --name maintenance-mod30 --network rota-saude_default -p 5177:5177 \
  -v $R/.worktrees/maintenance-mod30:/app -v maintenance-mod30-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3052 -e VITE_MAINTENANCE_ENV=development node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5177 --host 0.0.0.0"
docker logs -f maintenance-mod30   # espere o "ready"
```

Em `http://maintenance.localhost:5177` (o usuário faz login e TOTP; anote antes o estado das duas chaves), com screenshot em `evidencias/2026-11-03-modulo-30/maintenance/`:
1. Curitiba (`record`) → Funcionalidades: as duas chaves com nome e descrição; com o transporte desligado, `tfd` diz "transporte sanitário (sanitary_transport) desligado".
2. Ligar `sanitary_transport` → "ligada e utilizável"; ligar `tfd` → "ligada e utilizável" (com `clinical_record` ligado).
3. Desligar `sanitary_transport`: o `tfd` continua ligado e passa a "ligada, falta pré-requisito"; no dashboard, as telas do TFD respondem `feature_disabled`; religar o transporte devolve o TFD a utilizável.
4. Maringá (`integrated`): ligar `tfd` → "prontuário da atenção primária (clinical_record) desligado"; `sanitary_transport` liga utilizável; desligar os dois de volta.
5. **Auditoria** lista as tentativas de `setCityFeature`.
6. A 375 px: sem rolagem lateral da página.

Depois: chaves de volta ao estado anotado (ou ao que as provas do dashboard e do wpda precisarem); `docker rm -f maintenance-mod30 && docker volume rm maintenance-mod30-node-modules && docker compose -f $R/docker-compose.yml start maintenance`.

- [ ] **Step 4: Pare.** Merge só depois do api, do dashboard e do wpda do 30, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só com os commits desta entrega. Quando autorizado: `git fetch`, `merge --ff-only feat/mod-30-sanitary-transport`, `push origin main`, `worktree remove $M`, `branch -d`. Se as partes do api e do dashboard do F-30.1 estiverem **Done**, mova o F-30.1 para **Done** (opção `98236657`) e para **Verified** se as provas passaram.

---

## Divergências propostas ao contrato

1. **M1 — Nomes.** O §10 não fixa rótulos. Proposta: "Transporte sanitário" e "Tratamento Fora de Domicílio (TFD)", iguais no dashboard; a descrição é a do catálogo do api.
2. **M2 — "Só os dois interruptores" exige mudança aqui.** O §10 dá a entender que basta o catálogo do api. Conferido: sem os rótulos, a aba mostra as duas chaves e o `missing` `sanitary_transport_disabled` crus (é o nome que `Platform::Features#missing_for` gera para `feature:sanitary_transport`); `clinical_record_disabled` já tem frase. Proposta: o §10 lista esse código e diz que o maintenance o traduz.
3. **M3 — Sem cascata.** O §1 fixa `tfd` exigindo `feature:sanitary_transport`, mas não diz o que acontece ao desligar o transporte com o TFD ligado. Proposta: nada em cascata — o TFD continua ligado e fica "ligado, falta pré-requisito" (o api já calcula assim, ADR 0032); suas rotas respondem 403 `feature_disabled` até o transporte voltar. O §10 registra isso.
4. **M4 — CNES do TFD não é pré-requisito (para o api).** `tfd_billing_health_unit_id` (§3) é configuração do dashboard; sem ela o TFD funciona e só o BPA fica pendente. Proposta: não vira `requires` do catálogo; se o api preferir torná-lo pré-requisito (`tfd_billing_unit_missing`), a aba já o mostra cru (Review Focus 4) e a frase entra numa correção de uma linha.

## Self-review

- **Cobertura (decisão 23, §7, F-30.1; contrato §1, §10, §13):** nomes e pré-requisitos — Task 1 (Review Focus 1, 3, 4; M1, M2); sem cascata — Task 1, terceiro caso da aba (Review Focus 2; M3); desligar sem apagar — Task 1 (Review Focus 5) e README; suíte, build e prova — Task 2; cards — Tasks 0 e 2. Não há chave simulada no 30, então não há aviso (diferente do 29).
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards e as duas `description` (Task 0, com a regra para diferença).
- **Tipos e nomes:** `featureLabel`, `missingText`, `missingSummary`, `doneText` (pela frase do status), `featureStatus` (pelo "ligada, falta pré-requisito"), `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` existem em `origin/main` (`6b788fc`).
- **Review Focus:** 1, 2 e 3 — Task 1 (`FeaturesTab.test.tsx`); 4 — Task 1 (`features.test.ts`); 5 — Task 1 (os casos da aba).
- **Proporção:** blocos de código só de testes.
