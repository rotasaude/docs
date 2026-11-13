# Módulo 24c — Mapas de campo (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 24c** com `field_maps`, `FieldMaps::BuildJob`, `FieldMaps::RefreshBaseJob` e o GraphQL do contrato §5.4 (`fieldMaps`, `fieldMapBase`, `buildFieldMap`, `refreshFieldMapBase`, com a auditoria `MaintenanceAudit` `maintenance.field_map.build_requested` e `maintenance.field_map.base_refresh_requested`; os jobs gravam `field_map.built` e `field_map.failed` — Q19; Task 15 do plano do api). As Tasks 1–2 rodam com `fetch` falso e um SDL tirado do worktree do api; a prova da Task 3 precisa do api de pé. Ordem do contrato §11: … dashboard → **maintenance** → catálogo; na prática a geração do mapa de uma cidade de dev (Task 3) é pré-requisito da prova do dashboard e do campo — a Task 3 pode rodar antes delas, com autorização do usuário.

**Goal:** Tela de plataforma **"Mapas de campo"** no maintenance: para cada cidade, o estado do mapa (`building` | `ready` | `failed`), a versão (data da base), o tamanho, quando foi gerado e o motivo da falha; **"Gerar de novo"** por cidade; e a base do Brasil com a data, a idade, o aviso "base com mais de 90 dias" e **"Atualizar base"** (F-24.20; contrato §5.4, §9).

**Architecture:** O mapa é dado de **plataforma** (`field_maps` e a base no MinIO), não de cidade: a tela fica no trilho, ao lado de Medicamentos e Laboratório (molde: `src/screens/MedicationCatalog.tsx`, que já tem botões de importação de plataforma), e não abre conexão com banco de cidade. Uma consulta (`FieldMaps`: `cities` + `fieldMaps` + `fieldMapBase`) junta as cidades do catálogo com os mapas por `citySlug` — cidade sem linha em `field_maps` aparece "sem mapa". As duas mutações seguem o padrão `{ ok, errors { path message } }`. Regras puras (rótulos, tamanho legível, idade, quando oferecer os botões) em `src/lib/fieldMaps.ts`. O SDL muda: `schema:pull` do worktree do api + `codegen`.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL codegen (client preset), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-11-13-module-24c-field-app-design.md` (decisões 18, 26; §6 maintenance; §7 runbook: "gerar o mapa da cidade antes de cadastrar aparelhos"). **Contrato:** `docs/superpowers/plans/2026-11-13-module-24c-field-app-contracts.md` §1, §5.1, §5.4, §9, §11. Molde: `docs/superpowers/plans/2026-11-12-module-24b-home-visit-maintenance.md`. As lacunas M-1 a M-6 foram decididas no §13 do contrato (Q19–Q24) e o plano já as aplica; resumo em **"Decisões aplicadas (§13 do contrato)"**, no fim.

## Global Constraints

- **SDL esperado** (contrato §5.4 com Q19 e Q21; é o mesmo bloco da Task 15 do plano do api; a Task 0 confere o real e **para** se for outro). `fieldMaps` traz uma entrada por linha de `field_maps` (cidade sem linha não vem; a tela junta com `cities`):
  ```graphql
  type FieldMap { citySlug: String! state: String! version: ISO8601Date sizeBytes: BigInt builtAt: ISO8601DateTime errorCode: String }
  type FieldMapBase { version: ISO8601Date! ageDays: Int! }
  # Query
  fieldMaps: [FieldMap!]!
  fieldMapBase: FieldMapBase          # null = base nunca copiada
  # Mutation
  buildFieldMap(citySlug: String!): BuildFieldMapPayload           # { ok errors { path message } fieldMap }
  refreshFieldMapBase: RefreshFieldMapBasePayload                  # { ok errors { path message } fieldMapBase }
  ```
- Estados: `building` → "gerando…"; `ready` → "pronto"; `failed` → "falhou"; cidade sem linha → "sem mapa"; outro valor aparece cru. Motivos (`errorCode`, contrato §5.1): `base_missing` → "a base do Brasil ainda não foi copiada"; `extract_failed` → "o recorte falhou — gere de novo"; `boundary_missing` → "a cidade não tem a caixa do município (código IBGE)"; outro, cru.
- Tamanho em MB com uma casa e vírgula (`41234567` → "39,3 MB"; base 1024²). Datas em `pt-BR`. Idade: `ageDays > 90` → "base com mais de 90 dias — atualize antes de gerar".
- "Gerar de novo" desabilitado com o mapa `building` e com a base ausente (`fieldMapBase: null`); "Atualizar base" sempre disponível, com confirmação (é um download grande no servidor). Nenhum dos dois pede TOTP (Q22). Nenhum botão dispara duas vezes (desabilitado enquanto roda).
- Sem atualização automática (spec §6 do maintenance, ruling P5): a tela lê ao abrir, depois de cada mutação e no "atualizar". Sem job recorrente da base: o aviso dos 90 dias na tela atende a decisão 18 do spec (Q23).
- O maintenance não vê dado de cidade: nenhum domicílio, ponto ou aparelho passa por aqui; o arquivo de mapa não é baixado pela tela.
- Interface (subprojeto 5): componentes existentes (`Panel`, `DataTable`, `Tag`, `Button`, `ConfirmButton`, `ErrorState`, `EmptyState`); guardas de tema na suíte.
- Testes: `bin/test maintenance -w maintenance-mod24c <arquivos>`; tipos: `cd $M && npx tsc --noEmit`; CI: `npm run typecheck`, `npm test`, `npm run codegen:check`, `npm run build`.
- Git: `/opt/homebrew/bin/git`; nunca `git add -A`/`git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge e push só com autorização explícita.
- Board #1: F-24.20 (maintenance é superfície). Relato ao fim de cada task à sessão "Próximos módulos".

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `M=$R/.worktrees/maintenance-mod24c`, `A=$R/apps/api/.claude/mod24c` (worktree do api do 24c; nome e porta anotados na Task 0 a partir do plano do api).
- Worktree **fora de `apps/`**: `$M`, branch `feat/mod-24c-field-app` a partir de `origin/main`.
- Prova: Vite do worktree na **5177** (serviço `maintenance` do compose parado), apontando para o api do 24c; abra `http://maintenance.localhost:5177`; conta `dev@local` (o usuário faz login e TOTP).

## Review Focus

1. **Cidade sem mapa some da lista** (só as que têm linha em `field_maps` aparecem) — justamente as que precisam de "Gerar". Teste: Task 2, "cidade sem linha em field_maps aparece 'sem mapa' com Gerar".
2. **Duplo clique em "Gerar de novo"** enfileira dois jobs (ou gerar com o mapa já `building`). Teste: Task 2, "gerar: uma mutation só; desabilitado em building e sem base".
3. **Base velha passa despercebida** (91 dias). Teste: Task 1, `baseAgeWarning(91)` e `(90)`; Task 2, "base com mais de 90 dias: aviso".
4. **Api sem o 24c** (o GraphQL recusa o documento: campo desconhecido). Espera-se a frase da ordem de deploy, não a tela quebrada. Teste: Task 2, "api sem o 24c: explica a ordem de deploy".
5. **Valor futuro do api** (`state: "queued"`, `errorCode: "disk_full"`): aparece cru, nunca `undefined`. Teste: Task 1, tabela de rótulos.

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, worktree, base, SDL real do api | 0 |
| `schema.graphql`, `codegen.ts`, `src/gql/*` (gerados), `src/lib/fieldMaps.ts`, `src/lib/fieldMaps.test.ts` | SDL novo; rótulos, tamanho, idade, botões | 1 |
| `src/screens/FieldMaps.tsx`, `src/screens/FieldMaps.test.tsx`, `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx` | tela, mutações, trilho | 2 |
| `README.md` | parágrafo "Mapas de campo (módulo 24c)" | 3 |
| — | suíte, codegen:check, build, prova | 3 |

**Estratégia de teste:** funções puras com tabela de casos (Task 1); a tela com `fetch` falso roteado por nome de operação (helpers `reply`, `bodyOf`, `calls`, `route` copiados do topo de `src/screens/MedicationCatalog.test.tsx:10-40`), conferindo as variáveis exatas da mutation, a releitura depois dela, os botões desabilitados e as recusas (envelope com `errors`, campo nulo com `extensions.code`, documento recusado) (Task 2); suíte inteira, `codegen:check`, build e prova no navegador contra o api do 24c, gerando o mapa de uma cidade de dev (Task 3).

---

### Task 0: Card, worktree, base e SDL real

**Files:** nenhum.

- [ ] **Step 1: Card** — `gh project item-list 1 --owner rotasaude --limit 600 --format json --jq '.items[] | select(.title | test("^F-24\\.20 ")) | {id, title, status}'`. Expected: existe. Ausente: pare.

- [ ] **Step 2: SDL do api do 24c**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod24c api bin/rails runner 'puts Maintenance::Schema.to_definition' | grep -n -i -A8 "fieldmap"
```

(Use o diretório do worktree do api que o plano do api publicou; aqui só se lê — o `schema.graphql` do repo é gerado na Task 1.) Expected: as formas do Global Constraints. Diferente (ex.: `cityId` no lugar de `citySlug`, `state` como enum ou anulável, `sizeBytes` como `Int`, TOTP nas mutações): **pare e reporte** ao coordenador com o SDL — o api da Task 15 tem de bater com este bloco (Q19, Q21, Q22).

- [ ] **Step 3: Worktree e base**

```bash
G=/opt/homebrew/bin/git
$G -C $R/apps/maintenance fetch -q origin && $G -C $R/apps/maintenance log --oneline -1 origin/main
$G -C $R/apps/maintenance worktree add $R/.worktrees/maintenance-mod24c -b feat/mod-24c-field-app origin/main
cd $R/.worktrees/maintenance-mod24c && npm ci --silent && npx tsc --noEmit && npm run codegen:check && echo base-ok
$R/bin/test maintenance -w maintenance-mod24c
```

Expected: `9269b55` ou mais novo (com o 24b do maintenance, se já mergeado); `base-ok`; suíte verde; **anote** arquivos/casos e o `Screen` atual de `src/screens/Shell.tsx:22` (âncora da Task 2).

**Relato:** "maintenance-mod24c Task 0: base <sha> verde (N/M); SDL do 24c conferido (ou: divergente, ver relato)."

---

### Task 1: SDL novo e regras puras

**Files:**
- Modify: `schema.graphql`, `src/gql/*` (gerados), `codegen.ts` (`scalars` + `BigInt: "string"` — o SDL de hoje não tem `BigInt`; o graphql-ruby o serializa como texto)
- Create: `src/lib/fieldMaps.ts`, `src/lib/fieldMaps.test.ts`

**Interfaces:**
- Produces (`src/lib/fieldMaps.ts`):
  - `fieldMapStateLabel(state: string | null): string` (`null` = "sem mapa");
  - `fieldMapErrorText(code: string | null): string | null`;
  - `formatMapSize(bytes: number | string | null): string` (aceita `BigInt` serializado como texto; `null` → "—");
  - `baseAgeWarning(ageDays: number | null): string | null` (`> 90` → o aviso; `null` → "a base do Brasil ainda não foi copiada");
  - `canBuild(row: { state: string | null }, base: { version: string } | null): boolean` (falso em `building` e sem base);
  - `type FieldMapRow = { slug: string; name: string; uf: string | null; state: string | null; version: string | null; sizeBytes: string | number | null; builtAt: string | null; errorCode: string | null }`;
  - `joinFieldMaps(cities: { slug; name; uf }[], maps: { citySlug; … }[]): FieldMapRow[]` (todas as cidades, ordem do catálogo; mapa de cidade fora do catálogo também entra, com `name = slug`).

- [ ] **Step 1: SDL e codegen** — `cd $M && sh scripts/pull-schema.sh` com `-w /rails/.claude/mod24c` (o script usa o container `api`; rode o comando do script à mão com o `-w` do worktree, como no Step 2 da Task 0, redirecionando para `$M/schema.graphql`); `npm run codegen`. Expected: `schema.graphql` com os tipos do Global Constraints; `git diff --stat` só em `schema.graphql`, `codegen.ts` e `src/gql/`.

- [ ] **Step 2: Testes que falham** (`src/lib/fieldMaps.test.ts`):

```ts
it.each([
  [ "building", "gerando…" ], [ "ready", "pronto" ], [ "failed", "falhou" ], [ null, "sem mapa" ], [ "queued", "queued" ]
])("estado %s → %s", (s, label) => expect(fieldMapStateLabel(s)).toBe(label));
it("motivos conhecidos têm frase; desconhecido cru; nulo nulo", () => {
  expect(fieldMapErrorText("boundary_missing")).toBe("a cidade não tem a caixa do município (código IBGE)");
  expect(fieldMapErrorText("disk_full")).toBe("disk_full");
  expect(fieldMapErrorText(null)).toBeNull();
});
it("tamanho em MB com vírgula", () => {
  expect(formatMapSize(41234567)).toBe("39,3 MB");
  expect(formatMapSize("41234567")).toBe("39,3 MB");
  expect(formatMapSize(null)).toBe("—");
});
it("idade da base", () => {
  expect(baseAgeWarning(90)).toBeNull();
  expect(baseAgeWarning(91)).toBe("base com mais de 90 dias — atualize antes de gerar");
  expect(baseAgeWarning(null)).toBe("a base do Brasil ainda não foi copiada");
});
it("gerar: não em building nem sem base", () => {
  expect(canBuild({ state: "building" }, { version: "2026-11-01" })).toBe(false);
  expect(canBuild({ state: "ready" }, null)).toBe(false);
  expect(canBuild({ state: null }, { version: "2026-11-01" })).toBe(true);
});
it("junta todas as cidades do catálogo com os mapas pelo slug", () => {
  const rows = joinFieldMaps([ { slug: "curitiba", name: "Curitiba", uf: "PR" }, { slug: "maringa", name: "Maringá", uf: "PR" } ],
    [ { citySlug: "curitiba", state: "ready", version: "2026-11-01", sizeBytes: "41234567", builtAt: "2026-11-02T03:00:00Z", errorCode: null } ]);
  expect(rows.map((r) => [ r.slug, r.state ])).toEqual([ [ "curitiba", "ready" ], [ "maringa", null ] ]);
});
```

- [ ] **Step 3: Rode e veja falhar** — `$R/bin/test maintenance -w maintenance-mod24c src/lib/fieldMaps.test.ts`. **Step 4: Implemente; rode e veja passar; `tsc`; `npm run codegen:check`.**

- [ ] **Step 5: Commit**

```bash
cd $M && $G add schema.graphql codegen.ts src/gql src/lib/fieldMaps.ts src/lib/fieldMaps.test.ts
$G commit -m "feat: add field map labels, size, base age and build rules with the new maintenance schema" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Tela "Mapas de campo"

**Files:**
- Create: `src/screens/FieldMaps.tsx`, `src/screens/FieldMaps.test.tsx`
- Modify: `src/screens/Shell.tsx` (`Screen` + `"fieldMaps"`; `NAV` + `{ key: "fieldMaps", label: "Mapas de campo" }` depois de "Vigilância"; render), `src/screens/Shell.test.tsx`, `src/screens/Shell.rail.test.tsx` (lista de rótulos)

**Interfaces:**
- Consumes: `gql`, `graphql` (`src/lib/api.ts`, `src/gql`); Task 1.
- Produces:
  - Documentos: `query FieldMaps { cities { slug name uf } fieldMaps { citySlug state version sizeBytes builtAt errorCode } fieldMapBase { version ageDays } }`; `mutation BuildFieldMap($citySlug: String!) { buildFieldMap(citySlug: $citySlug) { ok errors { path message } } }`; `mutation RefreshFieldMapBase { refreshFieldMapBase { ok errors { path message } } }`.
  - `FieldMaps()`: painel "Base do Brasil" (versão, "há N dias", `baseAgeWarning`, "Atualizar base" com `ConfirmButton`: "Copiar a base do Protomaps de novo? O download é grande e roda no servidor."); painel "Mapas por cidade" (`DataTable`: Cidade/UF, Situação (`Tag`), Versão, Tamanho, Gerado em, Motivo, ação "Gerar de novo <cidade>" quando `canBuild`); "atualizar" relê. Mutação ok → "pedido enviado — o job gera o arquivo em segundo plano" e releitura; `errors[]` → as mensagens; payload nulo com `errors` do envelope → `<code> — <mensagem>`; documento recusado (validação: campo desconhecido) → "o api ainda não tem os mapas de campo (módulo 24c): atualize o api antes do maintenance".

- [ ] **Step 1: Testes que falham** (`src/screens/FieldMaps.test.tsx`):

```tsx
it("cidade sem linha em field_maps aparece 'sem mapa' com Gerar", async () => {
  route(fetchMock, { "query FieldMaps": () => mapsReply({ fieldMaps: [ READY_CURITIBA ] }) });
  renderScreen();
  const maringa = await screen.findByRole("row", { name: /Maringá/ });
  expect(within(maringa).getByText("sem mapa")).not.toBeNull();
  expect(within(maringa).getByRole("button", { name: "Gerar de novo Maringá" })).not.toBeNull();
  expect(within(screen.getByRole("row", { name: /Curitiba/ })).getByText("39,3 MB")).not.toBeNull();
});
it("gerar: uma mutation só; desabilitado em building e sem base", async () => {
  route(fetchMock, { "query FieldMaps": () => mapsReply(), "mutation BuildFieldMap": () => reply(200, { data: { buildFieldMap: { ok: true, errors: [] } } }) });
  renderScreen();
  const button = await screen.findByRole("button", { name: "Gerar de novo Maringá" });
  await user.dblClick(button);
  await waitFor(() => expect(calls(fetchMock, "mutation BuildFieldMap")).toHaveLength(1));
  expect(bodyOf(calls(fetchMock, "mutation BuildFieldMap")[0]).variables).toEqual({ citySlug: "maringa" });
  await waitFor(() => expect(calls(fetchMock, "query FieldMaps")).toHaveLength(2));
  // building: botão desabilitado; fieldMapBase null: todos desabilitados e a frase da base
});
it("base com mais de 90 dias: aviso; atualizar base pede confirmação e manda a mutation", async () => { /* ageDays 120 */ });
it("falhou: mostra o motivo; boundary_missing com a frase", async () => { /* … */ });
it("errors da mutation aparecem; nada é relido como sucesso", async () => { /* ok false, errors [{ path: "citySlug", message: "cidade arquivada" }] */ });
it("api sem o 24c: explica a ordem de deploy", async () => {
  route(fetchMock, { "query FieldMaps": () => reply(200, { errors: [ { message: "Field 'fieldMaps' doesn't exist on type 'Query'", extensions: { code: "undefinedField" } } ] }) });
  renderScreen();
  expect(await screen.findByText(/o api ainda não tem os mapas de campo/)).not.toBeNull();
});
```

Em `Shell.test.tsx` e `Shell.rail.test.tsx`: a lista de rótulos ganha "Mapas de campo" depois de "Vigilância"; clicar abre a tela.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test maintenance -w maintenance-mod24c src/screens/FieldMaps.test.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx`. **Step 3: Implemente** (o tratamento de "documento recusado" segue o de `MedicationCatalog.tsx` para "api sem o 19c"). **Step 4: Rode e veja passar; `tsc`.**

- [ ] **Step 5: Commit**

```bash
cd $M && $G add src/screens/FieldMaps.tsx src/screens/FieldMaps.test.tsx src/screens/Shell.tsx src/screens/Shell.test.tsx src/screens/Shell.rail.test.tsx
$G commit -m "feat: show field maps per city with rebuild and base refresh in the maintenance rail" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: README, suíte, build e prova

**Files:** Modify `README.md`.

- [ ] **Step 1: README** — parágrafo **Mapas de campo (módulo 24c).** depois do de **Ouvidoria do SUS (módulo 33)** (ou da última seção de módulo em `origin/main`): o que a tela mostra; "Gerar de novo" enfileira `FieldMaps::BuildJob` (recorte da base pela caixa do município + 2 km, zoom até 15); "Atualizar base" enfileira `FieldMaps::RefreshBaseJob`; o arquivo antigo só sai depois do novo `ready`; **gere o mapa da cidade antes de cadastrar aparelhos** (runbook do 24c); base com mais de 90 dias → atualize; o arquivo não tem dado de pessoa.

- [ ] **Step 2: Suíte e CI local**

```bash
cd $R && $R/bin/test maintenance -w maintenance-mod24c
cd $M && npm run typecheck && npm run codegen:check && npm run build
```

Expected: verde, com dois arquivos de teste a mais que a base da Task 0; build ok.

- [ ] **Step 3: Prova no navegador** (com autorização do usuário para disparar os jobs em dev): `http://maintenance.localhost:5177` → "Mapas de campo": (1) sem base → frase e "Gerar" desabilitado; (2) "Atualizar base" → depois de "atualizar", a data e a idade aparecem; (3) "Gerar de novo Curitiba" → "gerando…" e, depois, "pronto" com versão e tamanho; (4) Auditoria mostra `maintenance.field_map.base_refresh_requested` e `maintenance.field_map.build_requested` (Q19). Avise o coordenador que Curitiba tem mapa `ready` (pré-requisito das provas do dashboard e do campo).

- [ ] **Step 4: Commit e parada**

```bash
cd $M && $G add README.md
$G commit -m "docs: describe the field maps screen in the maintenance README" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Pare e peça autorização para merge/push; F-24.20 vai a Done só com api, campo, dashboard e maintenance entregues.

**Relato:** "maintenance-mod24c Task 3: suíte N/M, codegen:check, build; mapa de Curitiba ready em dev."

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do §5.4 e §9:** estado, versão, tamanho, data (Task 2), "Gerar de novo" por cidade (Task 2), "Atualizar base" com a idade e o aviso de 90 dias (Tasks 1–2), `fieldMaps`/`fieldMapBase`/`buildFieldMap`/`refreshFieldMapBase` (Tasks 1–2), auditoria conferida na prova (Task 3), testes de componente (Task 2).
2. **Placeholders:** o SDL real é conferido na Task 0 com parada se divergir; nomes do worktree e porta do api vêm do plano do api.
3. **Consistência:** `FieldMapRow`, `joinFieldMaps`, `canBuild`, `baseAgeWarning`, `formatMapSize`, `fieldMapStateLabel`, `fieldMapErrorText` com os mesmos nomes nas Tasks 1–2; operações `FieldMaps`, `BuildFieldMap`, `RefreshFieldMapBase` iguais nos documentos e nos testes.
4. **Review Focus:** as cinco linhas têm teste nas Tasks 1–2.

## Decisões aplicadas (§13 do contrato)

| Código | Q | Onde o plano aplica |
|---|---|---|
| M-1 | Q19 | Global Constraints, Tasks 1–2: cidade por `citySlug`; auditoria `maintenance.field_map.*` (prova da Task 3) |
| M-2 | Q20 | Task 2: tela própria "Mapas de campo" no trilho, no molde de Medicamentos |
| M-3 | Q21 | Global Constraints, Task 1: `state`/`errorCode` `String`, `sizeBytes` `BigInt` mapeado no `codegen.ts`, `fieldMapBase` anulável |
| M-4 | Q22 | Global Constraints: as duas mutações sem TOTP |
| M-5 | Q23 | Global Constraints, Task 1: sem job recorrente; aviso "base com mais de 90 dias" |
| M-6 | Q24 | Task 1 (`canBuild`), Task 2: sem base, "Gerar de novo" desabilitado; o api responde `base_missing` se chamado |
| — | Q10 | Global Constraints: maintenance só em F-24.20 |
