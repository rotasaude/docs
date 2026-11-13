# Módulo 24c — Mapa da microárea e aparelhos de campo (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:** (1) o **dashboard do 24b mergeado na `main`** (`feat/mod-24b-home-visits`, hoje em `.worktrees/dashboard-mod24b`): este plano parte da `main` **depois** desse merge e altera `src/modules/FieldDevices.tsx`, `src/modules/FieldDevicesAdmin.tsx`, `src/modules/homeVisits/DeviceQrDialog.tsx` e `registerFieldDevice` de `src/lib/api.ts` que ele cria (Task 13 do plano do 24b); (2) o **api do 24c** com `GET /territory/map`, `GET /territory/map/households` (com `map`, Q14), `location` no `<household>` e no `PATCH /territory/households/:id`, `replaces_device_id` no `POST /field_devices` e o motivo `replaced` (contrato §3.6, §5.2, §5.3; Q14, Q17) — só a prova da Task 7 depende dele rodando; (3) a tag **`map-style-v1.0.0`** no `rotasaude/contracts` (Q11; Task 6 do plano do contracts-repo) — a Task 2 copia dela; (4) uma cidade de dev com mapa `ready` (maintenance, Task 7). Ordem do contrato §11: … campo → **dashboard** → maintenance.

**Goal:** Lado dashboard do 24c: a tela **"Mapa da microárea"** no grupo "Cadastro territorial" (MapLibre GL JS + `pmtiles`, estilo, fontes e ícones locais copiados do `map-style-v1.0.0`, os domicílios com ponto, "corrigir o ponto" para quem grava o domicílio, aviso sem mapa) e, em **Aparelhos de campo**, "Substituir" com step-up (só aparelho ativo), "Cadastrar novo" para o revogado automaticamente, o aviso de perda ao revogar e os rótulos `inactive` → "suspenso enquanto sem vínculo", revogação automática → "revogado automaticamente: cadastre de novo" e `replaced` → "substituído por outro aparelho" (F-24.18, F-24.19, F-24.20 — Q10; contrato §8; Q16, Q17).

**Architecture:** O cliente ganha, na seção do 24a de `src/lib/api.ts`, `location` no `Household`/`HouseholdInput`, `listMapHouseholds` (que devolve também o `map` da cidade, Q14 — sem sonda do arquivo) e `fieldMapUrl`; na seção do 24b, `registerFieldDevice(name, replacesDeviceId?)`. O MapLibre fica atrás de um adaptador fino (`src/modules/territory/map/fieldMap.ts`) — a única peça que importa `maplibre-gl`/`pmtiles`, carregada por `React.lazy` para não pesar no pacote principal — e o estilo é uma função pura (`src/lib/fieldMapStyle.ts`) que troca os marcadores do `style-light.json` do `map-style-v1.0.0` (cópia em `src/lib/vendor/map-style/`, fontes e sprites em `public/map/`, tudo preso por um lock como `src/theme/vendor/`; Q11) por arquivos do próprio dashboard. A tela (`src/modules/TerritoryMap.tsx`) usa o adaptador por interface; os testes simulam o adaptador (tela) e o `maplibre-gl`/`pmtiles` (adaptador). Corrigir o ponto relê o domicílio (`getHousehold`), confere a versão vista no mapa e grava o corpo inteiro do 24a com `location` (o `PATCH` do 24a é corpo inteiro, `save_household.rb:1-4`). Regras de aparelho em `src/lib/fieldDevices.ts` (puras).

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom), Vite 5 (base `/dashboard/`), `maplibre-gl` e `pmtiles` (JS) com versões exatas escolhidas na Task 0 (Q12), estilo do `map-style-v1.0.0` (sem `@protomaps/basemaps` no dashboard), `qrcode` 1.5.

**Spec:** `docs/superpowers/specs/2026-11-13-module-24c-field-app-design.md` (decisões 2A, 6A, 11, 16, 17, 18, 19, 21, 25; §5; §6 dashboard). **Contrato:** `docs/superpowers/plans/2026-11-13-module-24c-field-app-contracts.md` §3.1, §3.3, §3.6, §4, §5.2, §5.3, §8, §10. Base do 24b: `docs/superpowers/plans/2026-11-12-module-24b-home-visit-dashboard.md` (Task 13, Global Constraints). As lacunas D-1 a D-9 foram decididas no §13 do contrato (Q10–Q18) e o plano já as aplica; resumo em **"Decisões aplicadas (§13 do contrato)"**, no fim.

## Global Constraints

- **Nenhum serviço de mapa de terceiros** (2A): estilo do `map-style-v1.0.0` (só `light`, Q18), fontes (glyphs) e ícones (sprites) servidos de `public/map/` do próprio dashboard; a única fonte de dados do mapa é `pmtiles://<origem do dashboard>/territory/map`. Nenhuma URL `http(s)://` de outro host no estilo, no código ou no CSS. Atribuição "© OpenStreetMap" sempre visível.
- **Coordenada:** `{ latitude, longitude, accuracy_m?, source }`; o dashboard só grava `source: "map"`, sem `accuracy_m` (Q3: opcional com `map`), latitude e longitude **arredondadas a 8 casas** (`Number(x.toFixed(8))`) antes de enviar. Coordenada nunca na URL, em `console`, em storage ou em mensagem de erro.
- **Leitura nominal:** `GET /territory/map/households` grava trilha (`clinical_record.viewed`, `access: territory_map`): `NOMINAL_QUERY` de `src/lib/careLines.ts` (nunca relida sozinha); o arquivo de mapa não é nominal. Na URL só `micro_area_id`.
- **Quem vê:** o mesmo portão de "Território da equipe" (`canUseTerritory(user)` e `isProfessional`, `src/shell/modules.ts`); a tela lista **todas** as microáreas do contexto (`getTerritoryContext`), e "Corrigir o ponto" só aparece onde `micro_areas[].can_register` (decisão 21). O api confere de novo.
- **Concorrência:** a correção manda a `version` que o mapa mostrou; `409 stale_version` (ou versão relida diferente) → `STALE_TEXT` de `src/lib/territoryRegistry.ts`, **uma** releitura dos pontos, nada reenviado.
- **Versões fixas (Q12):** `maplibre-gl` e `pmtiles` na versão estável com mais de duas semanas, escolhidas e registradas no contrato na Task 0, sem `^`/`~` no `package.json` e no `package-lock.json`. Nada de `@protomaps/basemaps`: o estilo vem pronto do `map-style-v1.0.0` (Q11).
- **Aparelhos:** token e `qr_payload` seguem as regras do 24b (só no diálogo, QR, sem texto do token, nada no cache). "Substituir" exige step-up (`SensitiveAction`); revogar (próprio ou do administrador) mostra o aviso de perda antes de confirmar.
- **Interface:** tela nova em `NAV_GROUPS` **e** `CATALOG.screens` (ícone e descrição, cobrados pelo `catalog.test.ts`); 375 px: o mapa ocupa a largura, a lista de domicílios com ponto vira cartões; cores só por tokens do tema (as do ponto lidas de `getComputedStyle` — `--accent`, `--warning`, `--text-muted`), sem literal; guardas `src/theme/*.test.ts` verdes.
- Testes: `$R/bin/test dashboard -w dashboard-mod24c <arquivos>`, nunca `vitest` direto; tipos: `cd $W && npx tsc --noEmit`. Todo `vi.mock("…/lib/api", …)` inclui `fetchCurrentSession` (lição do `443faca`).
- Git: `/opt/homebrew/bin/git`; nunca `git add -A`/`git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge e push só com autorização explícita.
- Board #1 (Q10): o dashboard é superfície de F-24.18 (Substituir), F-24.19 (aviso de perda, `device_suspended`, rótulos de revogação) e F-24.20 (mapa). Relato ao fim de cada task à sessão "Próximos módulos": `dashboard · dashboard-mod24c/feat/mod-24c-field-app · Task N · <pronto> · <divergência>`.

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod24c`, `G=/opt/homebrew/bin/git`. Worktree **fora de `apps/dashboard`** (o compose monta `./apps/dashboard:/app`; `node_modules` lá derruba o Vite).
- Caminhos relativos a `$W`. Fixtures: `renderWithProviders`, `sessionWith` (`src/test/campaignFixtures.tsx`); as do 24b (`src/test/homeVisitFixtures.ts`: `device`, `hvUser`, `context`); as novas em `src/test/territoryMapFixtures.ts` (Task 1).

## Review Focus

1. **Ponto gravado com mais de 8 casas** (o MapLibre devolve `-25.428471234567`): o api recusaria (`invalid_location`) ou, pior, arredondaria diferente do app. Teste: Task 5, "corrigir: arredonda a 8 casas e manda source map, sem accuracy_m".
2. **Correção por cima do que outro salvou:** o domicílio mudou entre o mapa e o salvar. Espera-se: versão relida diferente da vista → frase de versão nova, pontos relidos uma vez, nada enviado; `409 stale_version` → o mesmo. Teste: Task 5, "versão mudou: não grava; 409 relê uma vez".
3. **Mapa chamando terceiro** (estilo padrão do Protomaps aponta glyphs e sprites para `protomaps.github.io`). Teste: Task 2, "estilo: toda URL é do próprio dashboard; fonte pmtiles; atribuição OSM; nenhum marcador sobra" (e, no contracts, o `map-style/test/style.test.mjs`); Task 3, "fonte pmtiles aponta para /territory/map da origem".
4. **Cidade sem mapa ou microárea sem ponto** (o caso comum no começo). Espera-se: `map: null` na resposta dos pontos (Q14) → aviso "o mapa desta cidade ainda não foi gerado" e a lista de domicílios com ponto segue (se houver); sem ponto → "nenhum domicílio desta microárea tem ponto marcado — o ponto é marcado no app de campo". Teste: Task 4, os dois casos.
5. **Revogar sem saber que perde o pendente** (revogação manual = `device_revoked` = o app apaga a fila; Substituir não perde, Q17). Teste: Task 6, "revogar mostra o aviso com a data do último uso (e 'nunca usado')" no próprio e no do administrador.
6. **Substituir oferecido a quem o api recusa** (aparelho `inactive` ou revogado): o api só substitui aparelho ativo (Q16). Teste: Task 6, "Substituir só em ativo; revogado automaticamente oferece Cadastrar novo".

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree pós-24b, âncoras, base, versões (Q12), tag `map-style-v1.0.0` | 0 |
| `src/lib/api.ts`, `src/lib/api.territoryMap.test.ts`, `src/lib/api.homeVisits.test.ts`, `src/lib/territoryRegistry.ts`, `src/lib/territoryRegistry.test.ts`, `src/lib/homeVisits.ts`, `src/lib/homeVisits.test.ts`, `src/test/territoryMapFixtures.ts` | `location`, pontos com `map`, `replaces_device_id`, rótulo `replaced`, frases novas | 1 |
| `package.json`, `package-lock.json`, `scripts/map-style-sync.mjs`, `src/lib/vendor/map-style/{style-light.json,manifest.json}`, `src/lib/vendor/map-style.lock`, `public/map/**`, `src/lib/fieldMapStyle.ts`, `src/lib/fieldMapStyle.test.ts`, `src/lib/mapStyle.lock.test.ts` | dependências fixas, cópia do `map-style-v1.0.0` presa por lock, estilo local | 2 |
| `src/modules/territory/map/fieldMap.ts`, `src/modules/territory/map/fieldMap.test.ts`, `src/modules/territory/map/points.ts`, `src/modules/territory/map/points.test.ts` | adaptador do MapLibre; GeoJSON dos pontos; arredondamento | 3 |
| `src/modules/TerritoryMap.tsx`, `src/modules/territory/map/MicroAreaMap.tsx`, `src/modules/TerritoryMap.test.tsx`, `src/shell/modules.ts`, `src/shell/modules.territory.test.ts`, `src/shell/catalog.ts`, `src/App.tsx` | tela, menu, aviso sem mapa | 4 |
| `src/modules/territory/map/PointCorrection.tsx`, `src/modules/TerritoryMap.test.tsx` | corrigir o ponto | 5 |
| `src/lib/fieldDevices.ts`, `src/lib/fieldDevices.test.ts`, `src/modules/FieldDevices.tsx`, `src/modules/FieldDevicesAdmin.tsx`, `src/modules/FieldDevices.test.tsx`, `src/modules/FieldDevicesAdmin.test.tsx` | Substituir, aviso de perda, suspenso | 6 |
| — | suíte, build, varreduras, prova no navegador | 7 |

**Estratégia de teste:** cliente com `fetch` falso (URL, método, corpo; nada sensível na URL) (Task 1); estilo (marcadores trocados, nenhuma URL de terceiro) e lock da cópia do `map-style-v1.0.0` por funções puras e leitura de arquivo (Task 2); adaptador com `vi.mock("maplibre-gl")` e `vi.mock("pmtiles")` — confere protocolo, fonte, camadas, eventos e o arredondamento (Task 3); tela pelo ponto de entrada com `vi.mock` do api **e** do adaptador (`./territory/map/fieldMap`), cobrindo sem mapa (`map: null`), sem ponto, leitura nominal sem releitura, correção e recusas (Tasks 4–5); aparelhos pelas telas do 24b, estendendo os testes delas (Task 6); `tsc`, build (o pedaço do MapLibre sai em chunk próprio), suíte inteira, varreduras e prova no navegador com o api real (Task 7). O dashboard não testa a trilha nem o recorte do mapa (api).

---

### Task 0: Board, worktree pós-24b, âncoras e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard com o 24b; board #1.
- Produces: `$W` em `feat/mod-24c-field-app`; âncoras e base anotadas.

- [ ] **Step 1: Cards** — `gh project item-list 1 --owner rotasaude --limit 600 --format json --jq '.items[] | select(.title | test("^F-24\\.(18|19|20) ")) | {id, title, status}'`. Expected: os três existem (Q10; o api e o campo os movem para In Progress). Ausente: pare.

- [ ] **Step 2: O 24b está na `main`?**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export function registerFieldDevice(name: string)\|^export interface FieldDevice {\|^export function updateHousehold(\|^export interface Household {\|^export interface HouseholdInput {\|^export function getHousehold('
$G -C $R/apps/dashboard show origin/main:src/modules/FieldDevices.tsx | grep -c 'SensitiveAction'
$G -C $R/apps/dashboard show origin/main:src/modules/homeVisits/DeviceQrDialog.tsx | grep -n 'export function DeviceQrDialog'
$G -C $R/apps/dashboard show origin/main:src/modules/territory/HouseholdForm.tsx | grep -n 'export function householdAsInput'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '"territory-team" | "registry-status" | "territory-micro-areas"\|{ id: "territory-micro-areas", label: "Microáreas"\|item.id === "territory-team" || item.id === "registry-status"'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"/territory": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/homeVisits.ts | grep -n 'export function revokedReasonLabel'
$G -C $R/contracts fetch -q --tags origin && $G -C $R/contracts rev-parse map-style-v1.0.0^{commit}
```

Expected: todas as linhas; `SensitiveAction` ≥ 1 em `FieldDevices.tsx` (não é mais a casca "Em construção"); a tag `map-style-v1.0.0` existe. Faltou alguma linha do 24b: **pare** (o 24b não foi mergeado; este plano espera). Sem a tag: as Tasks 0–1 seguem, a Task 2 espera a Task 6 do plano do contracts-repo.

- [ ] **Step 3: Worktree e base**

```bash
$G -C $R/apps/dashboard worktree add $R/.worktrees/dashboard-mod24c -b feat/mod-24c-field-app origin/main
cd $R/.worktrees/dashboard-mod24c && npm ci --silent && npx tsc --noEmit && echo base-ok
$R/bin/test dashboard -w dashboard-mod24c
```

Expected: `base-ok` e suíte verde; **anote** arquivos/casos.

- [ ] **Step 4: Versões do mapa (Q12)** — `npm view maplibre-gl time --json` e `npm view pmtiles time --json`: a versão estável mais nova publicada há mais de duas semanas de cada (sem `-beta`/`-rc`), compatível com PMTiles v3 e com o `style-light.json` do `map-style-v1.0.0` (estilo MapLibre `version: 8`). Registre no contrato, numa subseção "Versões do dashboard (Task 0 do dashboard, Q12)" logo abaixo da seção "Prova técnica do campo" (só esta task escreve ali), e commite no worktree do docs:

```bash
D=/private/tmp/claude-501/-Users-eduardovrocha-Development-ioit-solutions-rota-saude/fd670630-7d57-4330-811c-0f40fb741325/scratchpad/docs-wt
$G -C $D add superpowers/plans/2026-11-13-module-24c-field-app-contracts.md
$G -C $D commit -m "docs: record the dashboard map library versions" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** `dashboard · dashboard-mod24c/feat/mod-24c-field-app · Task 0 · base <sha> verde (N/M); maplibre-gl <v>, pmtiles <v> no contrato · —`.

---

### Task 1: Tipos, cliente e frases

**Files:**
- Modify: `src/lib/api.ts` (seção do 24a: tipos e funções do mapa; seção do 24b: `registerFieldDevice`, `FieldDevice.revoked_reason` com `replaced`), `src/lib/territoryRegistry.ts` (`ERRORS`), `src/lib/homeVisits.ts` (`homeVisitError`, `revokedReasonLabel`)
- Create: `src/lib/api.territoryMap.test.ts`, `src/test/territoryMapFixtures.ts`
- Test: `src/lib/api.homeVisits.test.ts`, `src/lib/territoryRegistry.test.ts`, `src/lib/homeVisits.test.ts`

**Interfaces:**
- Produces:
  - `export interface HouseholdLocation { latitude: number; longitude: number; accuracy_m?: number; source: "gps" | "map" | (string & {}) }`
  - `Household.location: HouseholdLocation | null`; `HouseholdInput.location?: HouseholdLocation | null` (ausente = mantém; `householdAsInput` **não** muda: o formulário do 24a continua sem mexer no ponto).
  - `export interface MapHousehold { id: string; location: HouseholdLocation; address: { street: string | null; number: string | null }; families_count: number; version: number }`
  - `export interface FieldMapManifest { version: string; size_bytes: number; sha256: string }` (a mesma forma do `map` da sessão do app).
  - `export interface MapHouseholdsResponse { map: FieldMapManifest | null; items: MapHousehold[] }`.
  - `listMapHouseholds(microAreaId: string): Promise<MapHouseholdsResponse>` → `GET /territory/map/households?micro_area_id=<id>`, envelope `{ map, items }` (Q14; ausentes = `{ map: null, items: [] }`). O dashboard não sonda o arquivo de mapa.
  - `fieldMapUrl(): string` → `new URL(\`${TERRITORY_BASE}/map\`, window.location.origin).href`.
  - `registerFieldDevice(name: string, replacesDeviceId?: string): Promise<FieldDeviceCreated>` → corpo `{ name }` ou `{ name, replaces_device_id }`.
  - `FieldDevice.revoked_reason`: `"user" | "admin" | "replaced" | "membership_ended" | "user_disabled" | (string & {}) | null` (Q17); `revokedReasonLabel` (`src/lib/homeVisits.ts`, do 24b) ganha `replaced: "substituído por outro aparelho"`.
  - `src/lib/territoryRegistry.ts` `ERRORS` + `invalid_location: "ponto inválido — marque de novo no mapa"`, `map_not_found: "o mapa desta cidade ainda não foi gerado — peça à equipe de manutenção"`; `homeVisitError` (`src/lib/homeVisits.ts`) trata o `422 invalid_request` com `path: "replaces_device_id"` (sem código novo, contrato §3.6; o api só substitui aparelho ativo, Q16) como `"o aparelho a substituir não está mais ativo — recarregue a lista"`.
  - Fixtures: `mapHousehold(o?)`, `MAP_MANIFEST = { version: "2026-11-01", size_bytes: 41234567, sha256: "ab".repeat(32) }`, `mapHouseholds(o?)` (`{ map: MAP_MANIFEST, items: [mapHousehold()] }`), `LOC_MAP = { latitude: -25.42851, longitude: -49.2733, source: "map" }`, `householdWithLocation(o?)`.

- [ ] **Step 1: Testes que falham**

`src/lib/api.territoryMap.test.ts` (molde `src/lib/api.ombudsman.test.ts`: `stub`, `last`, `bodyOf`):

```ts
it("pontos: só o micro_area_id na URL; envelope ausente = { map: null, items: [] }", async () => {
  stub(200, {});
  expect(await listMapHouseholds("ma-1")).toEqual({ map: null, items: [] });
  expect(last().url).toBe("/territory/map/households?micro_area_id=ma-1");
});
it("pontos: o map da cidade vem junto (Q14)", async () => {
  stub(200, { map: MAP_MANIFEST, items: [ mapHousehold() ] });
  expect(await listMapHouseholds("ma-1")).toEqual({ map: MAP_MANIFEST, items: [ mapHousehold() ] });
});
it("fieldMapUrl: /territory/map da origem", () => {
  expect(fieldMapUrl()).toBe(`${window.location.origin}/territory/map`);
});
it("PATCH do domicílio leva location e version no corpo, nunca na URL", async () => {
  stub(200, household());
  await updateHousehold("hh-1", { ...input(), location: LOC_MAP }, 4);
  expect(last().url).toBe("/territory/households/hh-1");
  expect(bodyOf()).toMatchObject({ location: LOC_MAP, version: 4 });
});
```

Em `src/lib/api.homeVisits.test.ts`: `registerFieldDevice("Tablet novo", "fd1")` manda `{ name: "Tablet novo", replaces_device_id: "fd1" }`; `registerFieldDevice("Tablet")` manda só `{ name }`. Em `src/lib/homeVisits.test.ts`: `revokedReasonLabel("replaced")` = `"substituído por outro aparelho"`; os do 24b (`user`, `admin`) não mudam. Em `src/lib/territoryRegistry.test.ts`: `territoryError(new ApiError(422, { error: "invalid_location", path: "location.latitude" }, "x"))` e o `map_not_found` dão as frases acima.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod24c src/lib/api.territoryMap.test.ts src/lib/api.homeVisits.test.ts src/lib/territoryRegistry.test.ts src/lib/homeVisits.test.ts`. Expected: FAIL (funções inexistentes).

- [ ] **Step 3: Implemente** pelas Interfaces (`withQuery`/`territoryPath` existentes).

- [ ] **Step 4: Rode e veja passar; tipos.** Expected: PASS; `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.territoryMap.test.ts src/lib/api.homeVisits.test.ts src/lib/territoryRegistry.ts src/lib/territoryRegistry.test.ts src/lib/homeVisits.ts src/lib/homeVisits.test.ts src/test/territoryMapFixtures.ts
$G commit -m "feat: add household coordinates, map points with the city map manifest and device replacement to the API client" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Dependências fixas, cópia do `map-style-v1.0.0` presa por lock e estilo local (Q11, Q12, Q18)

Depende da tag `map-style-v1.0.0` (Task 6 do plano do contracts-repo). Sem ela: pare e avise o coordenador.

**Files:**
- Modify: `package.json`, `package-lock.json`
- Create: `scripts/map-style-sync.mjs`, `src/lib/vendor/map-style/style-light.json`, `src/lib/vendor/map-style/manifest.json`, `src/lib/vendor/map-style.lock`, `public/map/fonts/**`, `public/map/sprites/light*`, `public/map/fonts/OFL.txt`, `src/lib/fieldMapStyle.ts`, `src/lib/fieldMapStyle.test.ts`, `src/lib/mapStyle.lock.test.ts`

**Interfaces:**
- Consumes: `map-style/dist` da tag `map-style-v1.0.0` (`style-light.json` com os marcadores `__PMTILES__`, `__GLYPHS__`, `__SPRITE__`; `manifest.json`; `fonts/`; `sprites/light*`); as versões de `maplibre-gl` e `pmtiles` da Task 0.
- Produces:
  - `node scripts/map-style-sync.mjs map-style-v1.0.0 --contracts <clone>` (molde: `scripts/tokens-sync.mjs`): copia da tag `style-light.json` e `manifest.json` para `src/lib/vendor/map-style/`, as fontes do manifesto e a `OFL.txt` para `public/map/fonts/`, os sprites `light` para `public/map/sprites/` (só `light`, Q18), e grava `src/lib/vendor/map-style.lock` = `{ "ref": "map-style-v1.0.0", "files": { "<caminho a partir da raiz do repo>": "<sha256>" } }` (formato de `src/theme/vendor/tokens.lock`).
  - `fieldMapStyle(opts: { base: string; mapUrl: string }): StyleSpecification` (tipo de `maplibre-gl`): cópia do `style-light.json` vendorizado com `sources.protomaps.url = \`pmtiles://${mapUrl}\``, `glyphs = \`${base}map/fonts/{fontstack}/{range}.pbf\``, `sprite = \`${base}map/sprites/light\`` e o resto intacto (camadas em português, atribuição `© OpenStreetMap`). `base` = `import.meta.env.BASE_URL` (`/dashboard/`).

- [ ] **Step 1: Fixe as versões** da Task 0: `npm install --save-exact maplibre-gl@<v> pmtiles@<v>`. Expected: `package.json` sem `^`/`~` nos dois; nenhum `@protomaps/basemaps`.

- [ ] **Step 2: Testes que falham**

`src/lib/fieldMapStyle.test.ts`:

```ts
it("estilo: toda URL é do próprio dashboard; fonte pmtiles; atribuição OSM; nenhum marcador sobra", () => {
  const s = fieldMapStyle({ base: "/dashboard/", mapUrl: "http://curitiba.localhost:5218/territory/map" });
  expect(s.glyphs).toBe("/dashboard/map/fonts/{fontstack}/{range}.pbf");
  expect(s.sprite).toBe("/dashboard/map/sprites/light");
  expect(s.sources.protomaps).toMatchObject({ type: "vector", url: "pmtiles://http://curitiba.localhost:5218/territory/map", attribution: "© OpenStreetMap" });
  const text = JSON.stringify(s);
  expect(text).not.toMatch(/__(PMTILES|GLYPHS|SPRITE)__/);
  const urls = text.match(/https?:\/\/[^"\s]+/g) ?? [];
  expect(urls.every((u) => u.startsWith("http://curitiba.localhost:5218/territory/map"))).toBe(true);
});
it("toda fonte que as camadas pedem existe na cópia", () => {
  const stacks = new Set(fieldMapStyle({ base: "/", mapUrl: "x" }).layers.flatMap((l) => (l.layout as { "text-font"?: string[] })?.["text-font"] ?? []));
  for (const font of stacks) expect(existsSync(join(publicDir, "map/fonts", font, "0-255.pbf")), font).toBe(true);
});
it("não altera a cópia vendorizada (função pura)", () => { /* duas chamadas com bases diferentes; o JSON importado continua com os marcadores */ });
```

`src/lib/mapStyle.lock.test.ts`: os três testes de `src/theme/vendor.lock.test.ts` aplicados ao `map-style.lock` (cada arquivo listado bate com o hash; nada em `src/lib/vendor/map-style/` nem em `public/map/` fora do lock; `ref` = `map-style-v1.0.0`).

- [ ] **Step 3: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod24c src/lib/fieldMapStyle.test.ts src/lib/mapStyle.lock.test.ts`. Expected: FAIL (cópia e função ausentes).

- [ ] **Step 4: Copie e implemente** — `node scripts/map-style-sync.mjs map-style-v1.0.0 --contracts $R/contracts`; `fieldMapStyle` (clona com `structuredClone` e troca os três campos). Rode e veja passar; `tsc`.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add package.json package-lock.json scripts/map-style-sync.mjs src/lib/vendor/map-style src/lib/vendor/map-style.lock public/map src/lib/fieldMapStyle.ts src/lib/fieldMapStyle.test.ts src/lib/mapStyle.lock.test.ts
$G commit -m "build: pin maplibre-gl and pmtiles and copy the map-style-v1.0.0 light style with local fonts and sprites" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Adaptador do MapLibre e pontos

**Files:**
- Create: `src/modules/territory/map/fieldMap.ts`, `src/modules/territory/map/fieldMap.test.ts`, `src/modules/territory/map/points.ts`, `src/modules/territory/map/points.test.ts`

**Interfaces:**
- Consumes: `fieldMapStyle`, `fieldMapUrl` (Tasks 1–2).
- Produces:
  - `points.ts`: `roundCoord(n: number): number` (8 casas); `toFeatureCollection(items: MapHousehold[], selectedId: string | null): GeoJSON.FeatureCollection<GeoJSON.Point, { id: string; selected: boolean; label: string }>` (`label` = `"<rua>, <número>"`, ou `"S/N"`, nunca nome de pessoa); `boundsOf(items): [[number, number], [number, number]] | null`.
  - `fieldMap.ts`: `createFieldMap(container: HTMLElement, opts: { mapUrl: string; base: string; colors: { point: string; selected: string } }): FieldMapHandle`, com
    `interface FieldMapHandle { setPoints(fc): void; fitTo(bounds): void; onPointClick(cb: (id: string) => void): void; startCorrection(from: { latitude: number; longitude: number }, onMove: (to: { latitude: number; longitude: number }) => void): void; stopCorrection(): void; destroy(): void }`.
    Registra o protocolo uma vez por página (`maplibregl.addProtocol("pmtiles", new Protocol().tile)`); fonte GeoJSON `households` com `cluster: true`, `clusterRadius: 40`; camadas `households-clusters`, `households-points`; `AttributionControl({ compact: false })`; correção = `Marker({ draggable: true })` + clique no mapa move o marcador; `onMove` recebe valores **já** arredondados por `roundCoord`.

- [ ] **Step 1: Testes que falham**

`points.test.ts`: `roundCoord(-25.428471234567) === -25.42847123`; `roundCoord(-49.2733) === -49.2733`; `toFeatureCollection` com dois itens dá coordenadas `[longitude, latitude]`, `selected` só no escolhido e `label` `"das Flores, 120"` / `"S/N"`; nenhum campo além de `id`, `selected`, `label` nas propriedades; `boundsOf([])` = `null`.

`fieldMap.test.ts` (com `vi.mock("maplibre-gl")` — `Map` falso que guarda `addSource`, `addLayer`, `on` e expõe `fire(evento, dado)`; `Marker` falso com `setLngLat`/`on("dragend")`; `vi.mock("pmtiles", () => ({ Protocol: class { tile = vi.fn() } }))`):

```ts
it("protocolo pmtiles uma vez; estilo local; atribuição visível", () => {
  createFieldMap(div(), OPTS); createFieldMap(div(), OPTS);
  expect(maplibregl.addProtocol).toHaveBeenCalledTimes(1);
  expect(lastMap().options.style.sources.protomaps.url).toBe(`pmtiles://${OPTS.mapUrl}`);
  expect(lastMap().controls.some((c) => c instanceof maplibregl.AttributionControl)).toBe(true);
});
it("correção: clique e arrasto devolvem coordenadas com 8 casas", () => {
  const h = createFieldMap(div(), OPTS); const moved = vi.fn();
  h.startCorrection({ latitude: -25.42851, longitude: -49.2733 }, moved);
  lastMap().fire("click", { lngLat: { lat: -25.428471234567, lng: -49.273254119999 } });
  expect(moved).toHaveBeenLastCalledWith({ latitude: -25.42847123, longitude: -49.27325412 });
});
it("clique num ponto devolve o id; destroy remove o mapa", () => { /* fire("click", "households-points", { features: [{ properties: { id: "hh-1" } }] }) → cb("hh-1"); destroy() → map.remove() */ });
```

- [ ] **Step 2: Rode e veja falhar.** **Step 3: Implemente.** **Step 4: Rode e veja passar; `tsc`.**

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/territory/map/fieldMap.ts src/modules/territory/map/fieldMap.test.ts src/modules/territory/map/points.ts src/modules/territory/map/points.test.ts
$G commit -m "feat: wrap MapLibre with a field map adapter for household points and point correction" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tela "Mapa da microárea"

**Files:**
- Create: `src/modules/TerritoryMap.tsx`, `src/modules/territory/map/MicroAreaMap.tsx`, `src/modules/TerritoryMap.test.tsx`
- Modify: `src/shell/modules.ts` (`ModuleId` + `"territory-map"`; item `{ id: "territory-map", label: "Mapa da microárea", icon: "⌖" }` depois de `territory-team`; filtro junto de `territory-team`), `src/shell/modules.territory.test.ts`, `src/shell/catalog.ts` (`"territory-map": { icon: IconMapPins, description: "Domicílios com ponto no mapa e correção do ponto" }` — confira o ícone em `node_modules/@tabler/icons-react`), `src/App.tsx` (`case "territory-map": return <TerritoryMap />;`)

**Interfaces:**
- Consumes: `getTerritoryContext`, `listMapHouseholds`, `fieldMapUrl` (Task 1); `createFieldMap`, `toFeatureCollection`, `boundsOf` (Task 3); `TerritoryGate`, `STATIC_QUERY`, `NOMINAL_QUERY`, `territoryError`, `teamName`.
- Produces:
  - `TerritoryMap()`: `TerritoryGate` (`allow={(r) => r.professional}`, título "Mapa da microárea"); seletor de equipe e microárea com **todas** as microáreas do contexto; uma consulta só, `["territoryMapHouseholds", microAreaId]` (`NOMINAL_QUERY`), que traz `map` e `items` (Q14); `map: null` → `EmptyState` com `map_not_found` e, abaixo, a lista; `map` presente → `MicroAreaMap` por `React.lazy`; lista de cartões "Domicílios com ponto" (endereço curto, famílias); sem ponto → `"nenhum domicílio desta microárea tem ponto marcado — o ponto é marcado no app de campo"`.
  - `MicroAreaMap({ items, selectedId, onSelect, correction })` (default export, para o `lazy`): monta `createFieldMap` num `div` com `aria-label="Mapa da microárea"` e altura `min(60vh, 480px)`; `setPoints`/`fitTo` quando `items` muda; `destroy` ao desmontar.

- [ ] **Step 1: Testes que falham** (`vi.mock("../lib/api", …)` com `fetchCurrentSession`; `vi.mock("./territory/map/fieldMap", () => ({ createFieldMap: vi.fn(() => handle) }))`):

```tsx
it("sem mapa da cidade: aviso; a lista segue; nada do MapLibre", async () => {
  vi.mocked(api.listMapHouseholds).mockResolvedValue({ map: null, items: [ mapHousehold() ] });
  mount();
  expect(await screen.findByText(/o mapa desta cidade ainda não foi gerado/)).not.toBeNull();
  expect(await screen.findByText("das Flores, 120")).not.toBeNull();
  expect(createFieldMap).not.toHaveBeenCalled();
});
it("sem ponto na microárea: frase do app de campo", async () => { /* listMapHouseholds → { map: MAP_MANIFEST, items: [] } */ });
it("mapa pronto: pontos no adaptador; leitura nominal uma vez, sem releitura sozinha", async () => {
  vi.mocked(api.listMapHouseholds).mockResolvedValue(mapHouseholds());
  mount();
  await waitFor(() => expect(handle.setPoints).toHaveBeenCalled());
  expect(api.listMapHouseholds).toHaveBeenCalledTimes(1);
  expect(api.listMapHouseholds).toHaveBeenCalledWith("ma-1");
  act(() => { window.dispatchEvent(new Event("focus")); });
  expect(api.listMapHouseholds).toHaveBeenCalledTimes(1);
});
it("enfermeiro vê a microárea sem can_register, sem 'Corrigir o ponto'", async () => { /* contexto com can_register false */ });
it("nenhuma requisição ao arquivo de mapa fora do MapLibre (sem sonda, Q14)", async () => { /* fetch espiado: só /territory/map/households e o contexto */ });
```

Em `src/shell/modules.territory.test.ts`: `territory-map` aparece para profissional com o cadastro territorial utilizável e some sem ele ou para o operador; o `catalog.test.ts` existente cobra ícone e descrição.

- [ ] **Step 2: Rode e veja falhar.** **Step 3: Implemente.** **Step 4: Rode e veja passar; `tsc`.**

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TerritoryMap.tsx src/modules/territory/map/MicroAreaMap.tsx src/modules/TerritoryMap.test.tsx src/shell/modules.ts src/shell/modules.territory.test.ts src/shell/catalog.ts src/App.tsx
$G commit -m "feat: show the micro-area map with household points in the territory group" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Corrigir o ponto

**Files:**
- Create: `src/modules/territory/map/PointCorrection.tsx`
- Modify: `src/modules/TerritoryMap.tsx`, `src/modules/territory/map/MicroAreaMap.tsx`
- Test: `src/modules/TerritoryMap.test.tsx`

**Interfaces:**
- Consumes: `getHousehold`, `updateHousehold`, `householdAsInput` (`src/modules/territory/HouseholdForm.tsx`), `isStale`, `territoryError`, `STALE_TEXT`; `FieldMapHandle.startCorrection/stopCorrection`.
- Produces: `PointCorrection({ item, onDone, onStale })` (Q13: sem `PATCH` parcial; Q15: só corrige ponto existente — o primeiro ponto é marcado no app): botão "Corrigir o ponto" (só com `can_register` da microárea) → "Toque no mapa ou arraste o marcador" + "Salvar ponto" (desabilitado até mover e enquanto grava) + "Cancelar". Salvar: `getHousehold(item.id)`; `household.version !== item.version` → `STALE_TEXT` + `onStale()` (uma releitura dos pontos), sem `PATCH`; senão `updateHousehold(item.id, { ...householdAsInput(household), location: { latitude, longitude, source: "map" } }, item.version)`; sucesso → "ponto salvo" + `onDone()` (relê os pontos); `409 stale_version` → `STALE_TEXT` + `onStale()`; `422 invalid_location` e demais → `territoryError`.

- [ ] **Step 1: Testes que falham**

(`mountReady(items)` = `listMapHouseholds` resolvendo `{ map: MAP_MANIFEST, items }`.)

```tsx
it("corrigir: arredonda a 8 casas e manda source map, sem accuracy_m", async () => {
  vi.mocked(api.getHousehold).mockResolvedValue(householdWithLocation({ version: 4 }));
  vi.mocked(api.updateHousehold).mockResolvedValue(householdWithLocation({ version: 5 }));
  mountReady([ mapHousehold({ version: 4 }) ]);
  fireEvent.click(await screen.findByRole("button", { name: "Corrigir o ponto" }));
  act(() => handle.startCorrection.mock.calls[0][1]({ latitude: -25.42847123, longitude: -49.27325412 }));
  fireEvent.click(screen.getByRole("button", { name: "Salvar ponto" }));
  fireEvent.click(screen.getByRole("button", { name: "Salvar ponto" }));
  await waitFor(() => expect(api.updateHousehold).toHaveBeenCalledTimes(1));
  const [ id, input, version ] = vi.mocked(api.updateHousehold).mock.calls[0];
  expect([ id, version ]).toEqual([ "hh-1", 4 ]);
  expect(input.location).toEqual({ latitude: -25.42847123, longitude: -49.27325412, source: "map" });
  expect(input.micro_area_id).toBe("ma-1"); // corpo inteiro do 24a
  expect(api.listMapHouseholds).toHaveBeenCalledTimes(2);
});
it("versão mudou: não grava; 409 relê uma vez", async () => {
  // getHousehold com version 5 e item com 4 → sem updateHousehold, STALE_TEXT, listMapHouseholds 2×
  // depois: versões iguais, updateHousehold rejeita ApiError(409, { error: "stale_version" }) → STALE_TEXT, 1 releitura, sem reenvio
});
it("422 invalid_location: frase; o marcador fica para tentar de novo", async () => { /* … */ });
it("sem can_register: não há 'Corrigir o ponto'", async () => { /* … */ });
```

- [ ] **Step 2: Rode e veja falhar.** **Step 3: Implemente.** **Step 4: Rode e veja passar; `tsc`.**

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/territory/map/PointCorrection.tsx src/modules/territory/map/MicroAreaMap.tsx src/modules/TerritoryMap.tsx src/modules/TerritoryMap.test.tsx
$G commit -m "feat: correct a household point on the micro-area map with the expected version" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Aparelhos de campo — Substituir, Cadastrar novo, aviso de perda e rótulos de revogação (Q16, Q17)

**Files:**
- Create: `src/lib/fieldDevices.ts`, `src/lib/fieldDevices.test.ts`
- Modify: `src/modules/FieldDevices.tsx`, `src/modules/FieldDevicesAdmin.tsx`
- Test: `src/modules/FieldDevices.test.tsx`, `src/modules/FieldDevicesAdmin.test.tsx`

**Interfaces:**
- Consumes: `registerFieldDevice(name, replacesDeviceId?)` (Task 1); `DeviceQrDialog`, `SensitiveAction`, `ConfirmButton`, `homeVisitError`, `revokedReasonLabel` (24b, com `replaced` da Task 1); `fmtDateTime` de `src/lib/format.ts` (fuso da cidade).
- Produces (`src/lib/fieldDevices.ts`):
  - `AUTO_REVOKED_REASONS = ["membership_ended", "user_disabled"] as const`; `isAutoRevoked(d: FieldDevice): boolean` → `d.status === "revoked"` e motivo em `AUTO_REVOKED_REASONS`.
  - `deviceStatusLabel(d: FieldDevice): string` (Q16) → `active` "ativo"; `inactive` **"suspenso enquanto sem vínculo"**; revogado automaticamente → **"revogado automaticamente: cadastre de novo"**; outro `revoked` → `"revogado: " + revokedReasonLabel(reason)` (`user` "por você", `admin` "pelo administrador", `replaced` "substituído por outro aparelho"); outro valor de `status` cru.
  - `revokeLossWarning(lastUsedAt: string | null): string` → `"O que não foi sincronizado neste aparelho será perdido. Último uso: <data e hora da cidade>."`; `null` → `"… Último uso: nunca usado."`.
  - `canReplace(d: FieldDevice): boolean` → `d.status === "active"` (Q16: o api só substitui aparelho ativo; `inactive` e revogado não).
  - `REPLACE_NOTICE = "O aparelho antigo para de sincronizar agora e guarda o que não foi enviado. Leia o QR novo no tablet: no mesmo tablet, o app envia o que ficou guardado."` (Q17: o substituído recebe `device_suspended` e guarda a fila).
  - `REREGISTER_NOTICE = "Este aparelho foi revogado automaticamente. Cadastre um aparelho novo e leia o QR no mesmo tablet: o app envia o que não foi enviado."` (Q16).
- `FieldDevices`: em cada aparelho com `canReplace`, "Substituir <nome>" (`SensitiveAction`, step-up, campo "Nome do aparelho" 1–60, `description` = `REPLACE_NOTICE`; `run` → `registerFieldDevice(nome, device.id)` → o mesmo `DeviceQrDialog` do cadastro; vale com 3 aparelhos — o botão não depende do limite); em cada aparelho com `isAutoRevoked`, "Cadastrar novo <nome>" (o mesmo fluxo do cadastro do 24b, `registerFieldDevice(nome)` sem `replaces_device_id`, nome pré-preenchido, `description` = `REREGISTER_NOTICE`; o revogado não conta no limite de 3); "Revogar <nome>" passa a mostrar `revokeLossWarning(device.last_used_at)` no `ConfirmButton`; status por `deviceStatusLabel`. `422 invalid_request` com `path: "replaces_device_id"` → a frase da Task 1 e a lista relida.
- `FieldDevicesAdmin`: o aviso de perda na confirmação da revogação; status por `deviceStatusLabel`; sem "Substituir" nem "Cadastrar novo" (só o dono cadastra).

- [ ] **Step 1: Testes que falham**

`src/lib/fieldDevices.test.ts`: tabela de `deviceStatusLabel` (`active` → "ativo"; `inactive` → "suspenso enquanto sem vínculo"; `revoked` + `membership_ended` e + `user_disabled` → "revogado automaticamente: cadastre de novo"; `revoked` + `replaced` → "revogado: substituído por outro aparelho"; `revoked` + `user` → "revogado: por você"; `"paused"` → `"paused"`); `revokeLossWarning("2026-11-12T10:00:00-03:00")` contém `"12/11/2026"` e `"será perdido"`; `revokeLossWarning(null)` contém `"nunca usado"`; `canReplace` verdadeiro só para `active`; `isAutoRevoked` só para os dois motivos automáticos.

`src/modules/FieldDevices.test.tsx` (estende o do 24b):

```tsx
it("substituir: step-up, manda replaces_device_id, QR uma vez, vale com 3 aparelhos", async () => {
  vi.mocked(api.listFieldDevices).mockResolvedValue([ device(), device({ id: "fd2" }), device({ id: "fd3" }) ]);
  vi.mocked(api.registerFieldDevice).mockResolvedValue({ device: device({ id: "fd4", name: "Tablet novo" }), token: "rsf_SEGREDO", qr_payload: "rotasaude-field:v1:eyJ9" });
  const { client, container } = mount();
  fireEvent.click(await screen.findByRole("button", { name: "Substituir Celular da Joana" }));
  expect(screen.getByText(REPLACE_NOTICE)).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Nome do aparelho"), { target: { value: "Tablet novo" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByAltText("QR do aparelho de campo")).not.toBeNull();
  expect(api.registerFieldDevice).toHaveBeenCalledWith("Tablet novo", "fd1");
  expect(container.textContent).not.toMatch(/rsf_SEGREDO/);
  expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state.data))).not.toMatch(/rsf_|rotasaude-field/);
});
it("revogar mostra o aviso com a data do último uso (e 'nunca usado')", async () => { /* device({ last_used_at: "2026-11-12T10:00:00-03:00" }) e device({ id: "fd2", last_used_at: null }) */ });
it("Substituir só em ativo; revogado automaticamente oferece Cadastrar novo", async () => {
  // device({ status: "inactive" }) → "suspenso enquanto sem vínculo", sem Substituir nem Cadastrar novo
  // device({ id: "fd2", status: "revoked", revoked_reason: "membership_ended" }) → "revogado automaticamente: cadastre de novo" e "Cadastrar novo …" → registerFieldDevice(nome) sem segundo argumento
  // device({ id: "fd3", status: "revoked", revoked_reason: "replaced" }) → "revogado: substituído por outro aparelho", sem botões
});
it("422 replaces_device_id: frase e lista relida", async () => { /* … */ });
```

`src/modules/FieldDevicesAdmin.test.tsx`: a confirmação da revogação mostra `revokeLossWarning`; nenhum botão "Substituir" nem "Cadastrar novo"; `inactive` → "suspenso enquanto sem vínculo"; `replaced` → "revogado: substituído por outro aparelho".

- [ ] **Step 2: Rode e veja falhar.** **Step 3: Implemente** (troque o rótulo local do 24b por `deviceStatusLabel`; o teste do 24b que espera "inativo: vínculo encerrado" passa a esperar "suspenso enquanto sem vínculo"). **Step 4: Rode e veja passar; `tsc`.**

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/fieldDevices.ts src/lib/fieldDevices.test.ts src/modules/FieldDevices.tsx src/modules/FieldDevicesAdmin.tsx src/modules/FieldDevices.test.tsx src/modules/FieldDevicesAdmin.test.tsx
$G commit -m "feat: replace or re-register field devices, warn about unsynced loss on revoke and label revocations" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Suíte, build, varreduras e prova no navegador

**Files:** nenhum (correções voltam à task dona).

- [ ] **Step 1: Suíte e build**

```bash
cd $R && $R/bin/test dashboard -w dashboard-mod24c
cd $W && npx tsc --noEmit && npm run build && ls dist/assets | grep -i maplibre
```

Expected: verde, com os arquivos novos além da base da Task 0; build ok; o MapLibre num chunk próprio (o `lazy` da Task 4).

- [ ] **Step 2: Varreduras**

```bash
cd $W && grep -rnE "https?://" src/lib/fieldMapStyle.ts src/modules/territory/map src/modules/TerritoryMap.tsx | grep -v test
grep -rn "console\.\|localStorage\|sessionStorage" src/modules/territory/map src/modules/TerritoryMap.tsx src/lib/fieldDevices.ts
grep -rn 'vi.mock("../lib/api"' src/modules/TerritoryMap.test.tsx src/modules/FieldDevices*.test.tsx | wc -l; grep -rn "fetchCurrentSession" src/modules/TerritoryMap.test.tsx src/modules/FieldDevices*.test.tsx | wc -l
```

Expected: as duas primeiras vazias; as duas contagens iguais.

- [ ] **Step 3: Prova no navegador** (api do 24c de pé, cidade de dev com mapa `ready` gerado pelo maintenance; o usuário faz o login): `curitiba.localhost` no Vite do worktree. Conferir: (1) a microárea abre o mapa, sem requisição a outro host (aba Rede: só `/territory/map` com `206` e `/dashboard/map/**`); (2) atribuição "© OpenStreetMap" visível; (3) corrigir um ponto, recarregar, ver o ponto novo; (4) um domicílio editado em outra aba antes do salvar dá a frase de versão nova; (5) cidade sem mapa mostra o aviso; (6) Substituir um aparelho mostra o QR e o antigo aparece "revogado: substituído por outro aparelho" (Q17); (7) revogar mostra o aviso com a data; (8) um aparelho com o vínculo encerrado na semente aparece "suspenso enquanto sem vínculo" e, depois que o app bate na borda, "revogado automaticamente: cadastre de novo" com "Cadastrar novo" (Q16). Anote o resultado no relato.

- [ ] **Step 4: Pare e peça autorização** para merge/push; cards (Q10): F-24.18 a Done quando campo e dashboard estiverem entregues; F-24.19 quando contracts, api, campo e dashboard; F-24.20 quando contracts, api, campo, dashboard e maintenance.

**Relato:** `dashboard · dashboard-mod24c/feat/mod-24c-field-app · Task 7 · suíte N/M, build, prova ok · <divergências>`.

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do §8:** rota nova no grupo do território (Task 4); `maplibre-gl` + `pmtiles` com versões fixas (Task 2); estilo local (Task 2); `GET /territory/map` (Tasks 1, 3) e `/territory/map/households` (Tasks 1, 4); corrigir o ponto para quem edita, com `PATCH` e `version`, `stale_version` relê (Task 5); sem mapa → aviso (Task 4); Substituir com step-up e `replaces_device_id` (Task 6); aviso de perda com `last_used_at` (Task 6); `inactive` → "suspenso" (Task 6); testes de componente com MapLibre simulado (Tasks 3–6).
2. **Placeholders:** versões exatas e a origem das fontes são decididas na Task 2 (comando e regra dados); os corpos de teste resumidos em comentário listam as asserções.
3. **Consistência:** `MapHousehold`, `MapHouseholdsResponse`, `FieldMapManifest`, `HouseholdLocation`, `listMapHouseholds`, `fieldMapUrl` (Task 1) usados com os mesmos nomes nas Tasks 3–5; `FieldMapHandle` (Task 3) nas Tasks 4–5; `deviceStatusLabel`, `isAutoRevoked`, `revokeLossWarning`, `canReplace`, `REPLACE_NOTICE`, `REREGISTER_NOTICE` (Task 6). Entre planos: `map` = `{ version, size_bytes, sha256 }` como na Task 14 do api; `replaced` como no `REVOKED_REASONS` do api (Q17); marcadores `__PMTILES__`/`__GLYPHS__`/`__SPRITE__` como no `map-style/` do contracts e no `buildStyle` do campo.
4. **Review Focus:** as seis linhas têm teste (Tasks 2–6).

## Decisões aplicadas (§13 do contrato)

| Código | Q | Onde o plano aplica |
|---|---|---|
| D-1 | Q11 | Pré-requisito (3) e Task 2: estilo, fontes e sprites copiados do `map-style-v1.0.0` por lock (`src/lib/vendor/map-style.lock`) |
| D-2 | Q12 | Task 0 (Step 4): `maplibre-gl` e `pmtiles` estáveis com mais de duas semanas, registrados no contrato; Task 2 instala |
| D-3 | Q13 | Task 5: relê o domicílio, confere a versão e manda o corpo inteiro do 24a com `location` |
| D-4 | Q14 | Tasks 1 e 4: `map` na resposta de `GET /territory/map/households`; sem sonda do arquivo |
| D-5 | Q15 | Task 5: só corrige ponto existente |
| D-6 | Q16 | Task 6: Substituir só em ativo; "Cadastrar novo" para o revogado automaticamente; rótulos "suspenso enquanto sem vínculo" e "revogado automaticamente: cadastre de novo" |
| D-7 | Q17 | Tasks 1 e 6: motivo `replaced` ("substituído por outro aparelho"); `REPLACE_NOTICE` diz que o antigo guarda a fila |
| D-8 | Q18 | Task 2: só o estilo `light` |
| D-9 | Q10 | Global Constraints, Tasks 0 e 7: dashboard em F-24.18, F-24.19 e F-24.20 |
| — | Q3 | Global Constraints e Task 5: o dashboard manda `source: "map"` sem `accuracy_m` (opcional com `map`) |
