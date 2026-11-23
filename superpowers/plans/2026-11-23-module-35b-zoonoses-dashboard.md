# Módulo 35b — Centro de zoonoses (dashboard) Implementation Plan

> **Nota do coordenador (Q28, docs d4728f1):** a Task 10 usa `POST /zoonoses/leish_surveys/suggest` para sugerir os quarteirões, a população e a amostra do inquérito. Os anéis do foco vêm dessa rota, e a alternativa "só os vizinhos do quarteirão central" sai do plano.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:** (1) o **contrato** `2026-11-23-module-35b-zoonoses-contracts.md` (fonte única; este plano o segue, as Divergências D-1 a D-18, no fim, foram decididas nas Q10–Q27 do §19 e o plano já as aplica); (2) o **api do 35b** na branch `feat/mod-35b-zoonoses` (`apps/api/.claude/mod35b`): as Tasks 1–14 seguem pelas formas deste plano e pelas fixtures, e só a Task 15 (prova) depende do api rodando; (3) a `main` do dashboard em `2e3a2a9` (35a mergeado). Ordem do contrato §18: contracts → api → **dashboard**, wpda e maintenance → campo.

**Goal:** Lado dashboard do 35b (F-35.11 a F-35.17): o grupo **"Zoonoses"** (animais, campanhas e vacinação, demandas, observação do agressor, leishmaniose, canil e gatil, relatórios e parâmetros), a seção **`rabies_animal`** e a **"Condição final do animal após a observação"** na notificação do 25, o sinal **`animal_outcome_pending`** na fila do 25, as finalidades **`rabies_vaccine`** e **`collar`** nos produtos do 35a (com as telas de insumo do 35a abertas também com `zoonoses`), e o **cadeado** dos assuntos reservados da ouvidoria.

**Architecture:** O cliente ganha a seção "Centro de zoonoses (módulo 35b)" no fim de `src/lib/api.ts`, com as formas do contrato §1–§10 e da Q25 (o api da execução fixa as formas; a Task 0 confere), e estende os tipos do 25 (`SurveillanceNotification`), do 27a (`StockMovementKind`), do 33 (`OmbudsmanSubject`) e do 35a (`VectorProduct`, `VectorShortfall`). Portões, papéis, chaves, regras de tela e recusas ficam em `src/lib/zoonoses.ts`, e as frases numa tabela única `src/lib/zoonosesErrors.ts`. As telas ficam em `src/modules/Zoo*.tsx` (raízes) e `src/modules/zoonoses/` (painéis e diálogos). Rótulos de domínio vêm de `GET /zoonoses/domains` (código desconhecido aparece cru); só a notificação do 25 e a ouvidoria, cujos usuários não leem os domínios, usam rótulos fixos. O dashboard nunca decide o que o api garante (FEFO, falta de estoque, etiqueta, `target_dogs`, `ends_on`, prevalência, cobertura, encaminhamento automático, trilha, supressão): a tela só não oferece nem envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisões 16, 20, 23, 29–32; §7 F-35.11–F-35.17). **Contrato:** `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md` §1–§10, §13, §16, §17. **Molde:** `docs/superpowers/plans/2026-11-16-module-35a-vector-control-dashboard.md`.

## Global Constraints

- **Nomes no código** (contrato §1): interruptor `zoonoses` (rótulo "Centro de zoonoses"); papéis `zoonoses_team` ("Equipe de zoonoses"), `endemic_coordinator` (a coordenação, "Coordenação de vigilância ambiental", Q1) e `municipal_admin` (só lê relatórios); veterinário = CBO `223305` + CRMV + vínculo ativo; unidade `environmental_surveillance` ("Vigilância ambiental e zoonoses"); doença `acidente_animal_raiva`. Telas `zoo-animals`, `zoo-campaigns`, `zoo-demands`, `zoo-observations`, `zoo-leish`, `zoo-shelter`, `zoo-reports`, `zoo-settings`. **Código desconhecido aparece cru, nunca some.**
- **Portões:** `canUseZoonoses(user)` = não operador, `zoonoses` na sessão, `canUseTerritory(user)` (Q10) e um papel de leitura (`zoonoses_team`, `endemic_coordinator` ou `municipal_admin`; §13). `team` = `zoonoses_team` ou `endemic_coordinator`; `coordinator` = `endemic_coordinator`; `reader` = `team` ou `municipal_admin`. O veterinário (Q12) = `team` **e** o perfil de `GET /professionals/me` (`isVeterinarian`): a veterinária precisa do papel `zoonoses_team` para usar o dashboard; o api confere de novo (`403 veterinarian_required`, `403 missing_role`, `403 feature_disabled`).
- **Insumos do 35a com `zoonoses`** (§1 "sem cascata", §3, Q3, Q8, Q11: `VectorControl::Gate.supplies_usable?`): `canUseVectorSupplies(user)` = `canUseVectorControl(user) || canUseZoonoses(user)` abre "Produtos e parâmetros" (só produtos e local de estoque; os parâmetros do 35a seguem `vector_control`), "Quarteirões e imóveis" (só a aba Quarteirões sem `vector_control`), "Faltas de insumo de campo", as telas de estoque do 27a (`canUseStockScreens`), o local `vector_control` e o tipo de unidade "Vigilância ambiental e zoonoses".
- **Sigilo** (§16): tutor, adotante e paciente da notificação nunca em URL, log, storage, `console` nem lista; o tutor só como `{ id, display_name }` no detalhe do animal. **Microchip** só no corpo de `POST /zoonoses/animals/search` e do cadastro: a busca é chamada direta (sem `useQuery`, sem chave com o número), o resultado só no estado local, e nenhuma resposta traz o número (`has_microchip`, Q25). O contato da observação só por `GET …/observations/:id/contact`, no clique, no estado do diálogo, **nunca** no `queryClient`. O extrato (só o texto do cidadão) e o `place_address` da demanda só no detalhe (Q13). Na URL vão só ids, datas, estados, `due=1` e o cursor.
- **Leituras:** detalhe do animal, da demanda e da observação gravam trilha → `NOMINAL_QUERY` (lidos ao abrir e depois de uma ação, nunca sozinhos); filas sem pessoa, domínios, parâmetros, campanhas, inquéritos e relatórios → `STATIC_QUERY`. **Nenhum polling.**
- **Recusar em vez de cortar** (code points depois do `trim`, `textProblem` do 24b): nome do animal 1–60, pelagem 1–40, raça 1–60, registro 1–20, microchip `^\d{15}$`; campanha e inquérito 1–80; `adverse_note`, `cancel_reason`, `outcome_note`, nota do desfecho da LV 10–500; `inspection_note` 10–2000; texto de encerramento da demanda 20–4000; recusa e prorrogação 10–4000 (as do 33); `reason_note` da eutanásia 10–1000; `elisa_request` 1–30; `place_address` 10–300.
- **Frases fixas do contrato** (copiar letra por letra): "Centro de zoonoses"; "Notifique o evento adverso no formulário do Ministério da Saúde (EATAV)."; "Condição final do animal após a observação"; "Desfecho pendente"; "Coleiras a trocar"; os oito subtipos do §4 (Escorpião; Morcego; Rato ou outro roedor; Carrapato; Animal que mordeu ou atacou; Animal com comportamento estranho ou paralisia; Gato com feridas (esporotricose); Outro problema com animal) e o pai "Zoonoses".
- **Step-up** (`SensitiveAction` com `requiresStepUp`): eutanásia (§7) e `PUT /zoonoses/parameters` (Q12: segue o PUT do 35a). Escrita sempre com corpo JSON (`{}` sem campo).
- **Concorrência:** `expected_version` em toda escrita sobre registro mutável; 409 (`animal_changed`, `demand_changed`, `forwarding_changed`, `observation_changed`, `test_changed`, `stay_changed`, `parameters_changed`, `campaign_already_open`, `stay_already_open`, `already_euthanized`, `invalid_transition` e os `*_changed` propostos na Task 2, conferidos na Task 0) → a frase do código, **uma** releitura do que estava na tela, nenhum reenvio (`isZoonosesStale`, §13). Botão de escrita desabilitado enquanto a chamada corre.
- **Eutanásia** (§7, §13): o formulário mostra o veterinário logado (nome e "CRMV-UF número"), motivo, nota e base legal, e pede o step-up; depois do envio, a tela só lê (nenhum botão de editar ou repetir).
- **Interface:** toda tela nova em `NAV_GROUPS` **e** `CATALOG.screens` (`catalog.test.ts` cobra); telas com `DataTable` (`card: "title"`/`"actions"`), 375 px sem rolagem horizontal; cores só por papéis do tema (`Tag` com `tone`); guardas `src/theme/*.test.ts` verdes. Componentes existentes: `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `ActionMessage`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `useFormProblems` (`src/modules/psychosocial/useFormProblems.tsx`), `useDialog`, `saveFile` (`src/modules/lab/saveFile.ts`), `ReportTable` do 35a.
- **CI (lição do `443faca`):** todo `vi.mock("…/lib/api", async (orig) => …)` inclui `fetchCurrentSession: vi.fn()` resolvendo o usuário.
- "Agora" é `new Date()`; telas recebem `today` por prop; testes com `NOW35B = new Date("2026-11-23T10:00:00-03:00")`, `TODAY35B = "2026-11-23"` (`TZ=America/Sao_Paulo`). Datas da cidade por `cityIsoDate` de `src/lib/format.ts`.
- **Testes:** só `$R/bin/test dashboard -w dashboard-mod35b [arquivos do vitest]`, nunca `vitest` direto; tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- **Git:** `/opt/homebrew/bin/git`; nunca `git add -A` nem `git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge, push e tag só com autorização explícita.
- **Board #1:** o dashboard é superfície de F-35.11 a F-35.17. Card vai a Done só quando o **último** app dele termina (F-35.11: api, dashboard e maintenance; F-35.13: api, dashboard e wpda; os outros: api e dashboard) e a Verified depois da prova (Task 15).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`): `dashboard · dashboard-mod35b/feat/mod-35b-zoonoses · Task N · <pronto> · <divergência nova>`.

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod35b`, `G=/opt/homebrew/bin/git`. Worktree **fora de `apps/dashboard`** (o compose monta `./apps/dashboard:/app`; `node_modules` ali duplica o React no Vite), branch `feat/mod-35b-zoonoses` a partir de `origin/main` (`2e3a2a9`), com `npm ci`. O worktree do 35a (`dashboard-mod35a`) fica de pé e não é usado. Caminhos relativos a `$W`.
- Sem jest-dom (`not.toBeNull()`, `toBe()`); `globals: false`; `cleanup` no `afterEach`. `renderWithProviders`/`sessionWith` de `src/test/campaignFixtures.tsx`; as do 35a em `src/test/vectorControlFixtures.ts` (`vcUser`, `LARVICIDE`, `shortfall`, `report`); as novas em `src/test/zoonosesFixtures.ts` (Task 1).

## Review Focus

1. **Eutanásia repetida, por quem não é veterinário ou com o step-up vencido.** Duplo clique, aba velha, usuário da equipe. Espera-se: um envio; `403 veterinarian_required` e `409 already_euthanized` com a frase e uma releitura do histórico; `401 mfa_required` volta ao step-up sem reenviar; depois do 201 a tela mostra a eutanásia só para leitura e o animal "morto". Teste: Task 11, "eutanásia: step-up, um envio, recusas do não veterinário e da duplicidade, só leitura depois".
2. **Microchip, contato ou endereço vazando para URL, cache ou lista.** Espera-se: nenhuma chave do `queryClient` com o microchip, o telefone, o nome do tutor da notificação ou o `place_address`; a lista de demandas sem extrato nem endereço; a fila de observações sem tutor nem paciente. Teste: Task 5, "busca por microchip: corpo, nada no cache"; Task 7, "fila sem extrato nem endereço"; Task 8, "contato: só no diálogo, nada no cache".
3. **A ficha do 25 que apaga a outra metade.** O PATCH da ficha substitui a anterior (`save_form.rb:2`): duas seções separadas apagariam uma à outra, e mandar as chaves do animal noutra doença dá `invalid_form`. Espera-se: uma seção só, com os campos do Sinan e, só em `acidente_animal_raiva`, os do animal, no mesmo PATCH. Teste: Task 9, "ficha: uma seção, um PATCH; animal só em acidente_animal_raiva".
4. **"Sadio" no 10º dia.** A borda do §18. Espera-se: `outcome_on` igual a `ends_on` aceita "sadio"; um dia antes recusa com `observation_not_finished` antes de enviar; os dias que faltam contam pela data da cidade. Teste: Task 2 (`closeObservationProblems`, bordas) e Task 8.
5. **Segunda campanha aberta ou segunda dose na campanha.** Espera-se: `409 campaign_already_open` → frase e uma releitura das campanhas, sem reenviar; `422 already_vaccinated_in_campaign` → a frase no animal, o formulário fica. Teste: Task 6, "abrir com outra aberta: 409 relê uma vez; segunda dose recusada no animal".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.zoonoses.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/devProxy.ts`, `src/devProxy.test.ts`, `vite.config.ts`, `src/test/zoonosesFixtures.ts` | tipos e cliente do 35b; extensões do 25, 27a, 33 e 35a; rótulo do interruptor; proxy; fixtures | 1 |
| `src/lib/zoonosesErrors.ts`, `src/lib/zoonosesErrors.test.ts`, `src/lib/zoonoses.ts`, `src/lib/zoonoses.test.ts` | frases; portões, papéis, chaves, rótulos fixos, regras de tela, recusas | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.zoonoses.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/zoonoses/ZoonosesGate.tsx`, `src/modules/zoonoses/ZoonosesGate.test.tsx`, cascas das 8 telas | grupo "Zoonoses" e portão (F-35.11) | 3 |
| `src/lib/vectorControl.ts`, `src/lib/vectorControl.test.ts`, `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts`, `src/modules/vectorControl/VectorGate.tsx`, `src/modules/VectorSettings.tsx`, `src/modules/vectorControl/ProductsPanel.tsx`, `src/modules/VectorBlocks.tsx`, `src/modules/VectorStockShortfalls.tsx`, `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/Attendance.tsx`, `src/modules/VectorSettings.zoonoses.test.tsx`, `src/modules/VectorStockShortfalls.test.tsx`, `src/modules/attendance/UnitForm.vector.test.tsx` | insumos do 35a com `zoonoses` (F-35.12, F-35.15) | 4 |
| `src/modules/ZooAnimals.tsx`, `src/modules/zoonoses/AnimalSearch.tsx`, `src/modules/zoonoses/AnimalForm.tsx`, `src/modules/zoonoses/AnimalHistory.tsx`, `src/modules/zoonoses/PersonLookup.tsx`, `src/modules/ZooAnimals.test.tsx` | animais (F-35.11) | 5 |
| `src/modules/ZooCampaigns.tsx`, `src/modules/zoonoses/CampaignForm.tsx`, `src/modules/zoonoses/VaccinationForm.tsx`, `src/modules/zoonoses/WeeklyResult.tsx`, `src/modules/vectorControl/ReportTable.tsx`, `src/modules/ZooCampaigns.test.tsx` | campanhas e vacinação (F-35.12) | 6 |
| `src/modules/ZooDemands.tsx`, `src/modules/zoonoses/DemandDetail.tsx`, `src/modules/ZooDemands.test.tsx` | demandas (F-35.13) | 7 |
| `src/modules/ZooObservations.tsx`, `src/modules/zoonoses/ObservationDetail.tsx`, `src/modules/zoonoses/ContactDialog.tsx`, `src/modules/ZooObservations.test.tsx` | observação do agressor (F-35.14) | 8 |
| `src/lib/surveillanceForm.ts`, `src/lib/surveillanceForm.test.ts`, `src/modules/surveillance/NotificationFormSection.tsx`, `src/modules/surveillance/NotificationDetail.tsx`, `src/modules/surveillance/NotificationTable.tsx`, `src/modules/surveillance/NotificationDetail.zoonoses.test.tsx` | seção do animal, condição final e sinal na fila do 25 (F-35.14) | 9 |
| `src/modules/ZooLeish.tsx`, `src/modules/zoonoses/SurveyForm.tsx`, `src/modules/zoonoses/TestsPanel.tsx`, `src/modules/zoonoses/CollarsPanel.tsx`, `src/modules/ZooLeish.test.tsx` | leishmaniose (F-35.15) | 10 |
| `src/modules/ZooShelter.tsx`, `src/modules/zoonoses/StayForm.tsx`, `src/modules/zoonoses/EuthanasiaForm.tsx`, `src/modules/ZooShelter.test.tsx` | canil e gatil (F-35.16) | 11 |
| `src/modules/ZooReports.tsx`, `src/modules/ZooReports.test.tsx` | relatórios (F-35.17) | 12 |
| `src/modules/ZooSettings.tsx`, `src/modules/ZooSettings.test.tsx` | parâmetros (F-35.15, F-35.14) | 13 |
| `src/lib/ombudsman.ts`, `src/modules/ombudsman/SubjectsPanel.tsx`, `src/modules/ombudsman/NewManifestationForm.tsx`, `src/modules/ombudsman/SubjectsPanel.zoonoses.test.tsx` | assunto reservado com cadeado; chaves da demanda no balcão (Q14) (F-35.13) | 14 |
| — | suíte, build, varreduras, revisão, prova no navegador (§17) | 15 |
| — | board, relato, parada antes do merge | 16 |

**Estratégia de teste:** cliente com `fetch` falso (`stub`, `last`, `bodyOf`, como `api.vectorControl.test.ts`) conferindo URL, método e corpo — microchip, CPF, nota e endereço só no corpo (Task 1); regras puras por tabela de casos com as bordas do §18 (10º dia, segunda dose, ELISA sem DPP; Task 2); menu por papel e interruptor (Task 3); cada tela pelo ponto de entrada com `renderWithProviders`, `vi.mock` do api com `fetchCurrentSession`, sessão por papel, cobrindo caminho feliz, cada recusa que muda a tela, duplo envio, 409 com uma releitura e o que **não** pode aparecer (tutor em lista, microchip no cache, contato no cache, paciente da notificação) (Tasks 4–14); `tsc`, build, suíte, varreduras e prova no navegador com o api real na 3061 e o Vite na 5221 (Task 15). O dashboard não testa o que é do api (FEFO, falta, etiqueta, `target_dogs`, encaminhamento automático, jobs, trilha, supressão, sigilo de evento).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`2e3a2a9` ou mais novo); board #1; o api `feat/mod-35b-zoonoses`.
- Produces: cards em In Progress; `$W` na branch; base, âncoras, ícones e formas do api anotados.

- [ ] **Step 1: Cards** (um `gh` simples por vez, sem `$(...)` nem laço) — `gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-35\\.1[1-7] ")) | {id, title, status}'`; cada um de F-35.11 a F-35.17 que não estiver em In Progress ou além vai a In Progress com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <In Progress>`. Card ausente: **pare e pergunte**.

- [ ] **Step 2: Âncoras**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin && $G -C $R/apps/dashboard log --oneline -1 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Vigilância ambiental — endemias\|^export interface SurveillanceNotification {\|^export type StockMovementKind =\|^export interface OmbudsmanSubject {\|^export interface NewManifestationInput {\|^export interface VectorProduct extends\|^export interface VectorShortfall {\|^export async function getMyProfessional\|^type VcPage\|^async function binaryFetch'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "vector_control";'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '{ label: "Endemias", items: \[\|const vcOn = canUseVectorControl(user);\|if (item.id.startsWith("vector-"))\|"vector-stock-shortfalls") return'
$G -C $R/apps/dashboard show origin/main:src/lib/pharmacy.ts | grep -n '^export function canUseStockScreens\|^const MOVEMENT_LABEL\|^const DOCUMENT_LABEL'
$G -C $R/apps/dashboard show origin/main:src/modules/surveillance/NotificationDetail.tsx | grep -n 'const defaultFormSlot\|{formSlot(n, formSaved)}'
$G -C $R/apps/dashboard show origin/main:src/modules/surveillance/NotificationFormSection.tsx | grep -n '^const FIELDS = SINAN_FORM_FIELDS;'
$G -C $R/apps/dashboard show origin/main:src/modules/vectorControl/ProductsPanel.tsx | grep -n '^const USES: Use\[\]'
$G -C $R/apps/dashboard show origin/main:src/modules/ombudsman/SubjectsPanel.tsx | grep -n 'ConfirmButton label='
$G -C $R/apps/dashboard show origin/main:src/devProxy.ts | grep -n 'MODULE_35A_PROXY_KEYS'
```

Expected: cada linha achada. Linha mudada: a task dona aplica a mesma intenção sobre o texto real e anota no commit.

- [ ] **Step 3: Worktree, base e ícones**

```bash
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod35b -b feat/mod-35b-zoonoses origin/main
cd $R/.worktrees/dashboard-mod35b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod35b
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Paw|Dog|Cat|Vaccine|MessageReport|Eye|Microscope|Home|ReportAnalytics|Adjustments|Lock)\.mjs$'
```

Expected: `tsc-ok`, suíte verde (**anote** arquivos e testes) e os ícones (faltou algum → outro que exista, o mesmo na Task 3).

- [ ] **Step 4: Formas do api** — `$G -C $R/apps/api show feat/mod-35b-zoonoses:config/routes.rb | grep -n zoonoses`, os serializadores `app/services/zoonoses/json.rb` e os specs de request. Confira as formas do plano contra o api real (Q25: os tipos da Task 1 são proposta e o api da execução as fixa) e as decisões Q10–Q27; **onde o api diz outra coisa, vale o api**: a task dona ajusta tipo, fixture e teste (registre no commit). Sem branch ainda: siga pelas formas deste plano e repita este passo antes da Task 15. Confira também os rótulos de `config/surveillance/sinan-form.yml` (seção `rabies_animal`) e de `config/zoonoses/domains.yml` contra as fixtures.

---

### Task 1: Tipos, cliente HTTP, rótulo do interruptor, proxy e fixtures

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; extensões nas seções do 25, 27a, 33 e 35a), `src/lib/features.ts`, `src/devProxy.ts`, `vite.config.ts`
- Create: `src/lib/api.zoonoses.test.ts`, `src/test/zoonosesFixtures.ts`
- Test: `src/lib/features.test.ts`, `src/devProxy.test.ts`

**Interfaces:**
- Produces (em `src/lib/api.ts`, seção "Centro de zoonoses (módulo 35b; ADR 0054; contrato 2026-11-23 §1–§10)"; `ZOONOSES_BASE = import.meta.env.VITE_ZOONOSES_BASE || "/zoonoses"`, `zooPath(...parts)` = `hvPath(ZOONOSES_BASE, ...)`, `zooWrite(method, body = {})`, query por `hvQuery`/`withQuery`, listas `VcPage<T>`):

```ts
export type AnimalSpecies = "dog" | "cat" | "other";
export type AnimalSex = "male" | "female" | "unknown";
export type AnimalSize = "small" | "medium" | "large";
export type AnimalStatus = "active" | "deceased" | "missing" | "transferred";
export interface ZooRef { id: string; name: string }
export interface TutorRef { id: string; display_name: string | null }          // §2: só { id, display_name }
// Q25: nenhuma resposta traz o número do microchip (§16, Q9).
export interface Animal {
  id: string; version: number; species: AnimalSpecies | (string & {}); name: string | null; sex: AnimalSex | (string & {});
  size: AnimalSize | null; coat: string | null; breed: string | null; birth_estimate_on: string | null;
  has_microchip: boolean; registry_number: string | null; household: { id: string; address_short: string } | null;
  tutor: TutorRef | null; community: boolean; status: AnimalStatus | (string & {}); status_on: string | null; neutered: boolean;
}
export interface AnimalInput {
  species: AnimalSpecies; name?: string; sex: AnimalSex; size?: AnimalSize | null; coat?: string; breed?: string; birth_estimate_on?: string;
  microchip?: string; registry_number?: string; household_id?: string | null; tutor_patient_id?: string | null; community: boolean;
}
// Q20/Q25: o microchip e o registro só na criação; `status` não volta a `active`; a espécie não muda.
export type AnimalPatch = Partial<Omit<AnimalInput, "microchip" | "species">> & { status?: AnimalStatus; status_on?: string; expected_version: number };
export interface AnimalSearch { microchip?: string; registry_number?: string; tutor_patient_id?: string; household_id?: string }
export type CampaignStage = "planned" | "open" | "closed";
export interface RabiesCampaign { id: string; version: number; name: string; year: number; starts_on: string; ends_on: string; goal_dogs: number; goal_cats: number; stage: CampaignStage | (string & {}) }
export type CampaignInput = Omit<RabiesCampaign, "id" | "version" | "stage">;
export type VaccinationModality = "house" | "fixed_post" | "routine";
export interface RabiesVaccination {
  id: string; animal: { id: string; name: string | null } | null; species: "dog" | "cat"; campaign_id: string | null;
  modality: VaccinationModality | (string & {}); vaccinated_on: string; health_unit: ZooRef | null; product: VectorProductRef;
  stock_shortfall: boolean; adverse_event: boolean; cancelled_at: string | null; recorded_by: { name: string | null } | null;
}
export interface VaccinationInput {
  modality: VaccinationModality; animal_id?: string; species?: "dog" | "cat"; campaign_id?: string | null; vaccinated_on: string;
  health_unit_id?: string; household_id?: string; vector_product_id: string; adverse_event: boolean; adverse_note?: string;
}
export type DemandSubtype = "scorpion" | "bat" | "rodent" | "tick" | "aggressive_animal" | "neurological_animal" | "sporotrichosis" | "other";
export type DemandState = "open" | "scheduled" | "inspected" | "closed";
export type DemandOutcome = "resolved" | "guidance" | "not_found" | "referred";
// Q13: a lista traz subtipo, prazo, cor e estado; o extrato (só o texto do cidadão) e o endereço só no detalhe.
export interface ZoonosesDemand {
  id: string; version: number; subtype: DemandSubtype | (string & {}); state: DemandState | (string & {}); outcome: DemandOutcome | null;
  scheduled_on: string | null; inspected_on: string | null; created_at: string;
  forwarding: { id: string; state: ForwardingState | (string & {}); due_on: string; color: DeadlineColor | null; extended: boolean };
}
export interface ZoonosesDemandDetail extends ZoonosesDemand { extract: string; place_address: string | null; inspection_note: string | null }
export type ObservationState = "open" | "closed";   // Q16: sem `cancelled`
export type ObservationOutcome = "healthy" | "died" | "missing" | "rabies_signs" | "sent_to_exam" | "euthanized";
// §5: só o bloco do animal; nunca o paciente.
export interface BiteObservation {
  id: string; version: number; species: "dog" | "cat"; animal_condition: string | null; exposure_on: string; ends_on: string;
  place: "home" | "shelter"; animal: { id: string; name: string | null } | null; state: ObservationState | (string & {});
  outcome: ObservationOutcome | null; outcome_on: string | null;
}
export interface ObservationContact { tutor_name: string | null; tutor_phone: string | null; animal_address: string | null }
export type SurveyKind = "sample" | "census" | "focus";
export interface LeishSurvey {
  id: string; version: number; name: string; kind: SurveyKind | (string & {}); neighborhood: ZooRef; city_block_ids: string[];
  dog_population: number; target_dogs: number; dog_population_suggestion: number | null;   // Q18 starts_on: string; ends_on: string;
  stage: "draft" | "open" | "closed" | (string & {}); parameters_version: number;
}
export interface LeishSurveyInput { name: string; kind: SurveyKind; neighborhood_id: string; city_block_ids: string[]; dog_population: number; starts_on: string; ends_on: string }
export type DppResult = "reactive" | "non_reactive" | "invalid";
export type ElisaResult = "reactive" | "non_reactive" | "inconclusive";
export interface LeishTest {
  id: string; version: number; survey_id: string | null; animal: { id: string; name: string | null; tutor: boolean }; label: string; collected_on: string;
  dpp_result: DppResult | null; dpp_on: string | null; elisa_result: ElisaResult | null; elisa_on: string | null; elisa_request: string | null;
  outcome: { decision: string; decided_on: string } | null;
}
export interface Collar { id: string; version: number; animal: { id: string; name: string | null }; product: VectorProductRef; placed_on: string; due_on: string; removed_on: string | null; removal_reason: string | null; stock_shortfall: boolean }
export interface ShelterStay {
  id: string; version: number; animal: { id: string; name: string | null; species: string; tutor: boolean }; health_unit: ZooRef;
  entry_kind: string; entered_on: string; exit_kind: string | null; exited_on: string | null; bite_observation_id: string | null;
}
export interface ShelterEntryInput {
  animal_id?: string; new_animal?: Pick<AnimalInput, "species" | "sex" | "size" | "coat" | "breed">; health_unit_id: string;
  entry_kind: "collection" | "voluntary_surrender" | "observation" | "other"; entered_on: string; bite_observation_id?: string;
}
export interface ShelterExitInput { exit_kind: "returned" | "adopted" | "death" | "euthanasia" | "transferred"; exited_on: string; adopter_patient_id?: string; expected_version: number }
export interface Neutering { id: string; performed_on: string; professional: { name: string } }
export interface Euthanasia {
  id: string; performed_on: string; professional: { name: string; council: string; council_state: string; registration_number: string };
  reason: string; legal_basis: string[];
}
export interface EuthanasiaInput {
  animal_id: string; performed_on: string; reason: string; reason_note: string; legal_basis: string[];
  shelter_stay_id?: string; bite_observation_id?: string;   // Q18: a ligação vai do desfecho à eutanásia
}
export interface AnimalHistory extends Animal {
  vaccinations: RabiesVaccination[]; observations: BiteObservation[]; tests: LeishTest[]; collars: Collar[];
  stays: ShelterStay[]; neuterings: Neutering[]; euthanasia: Euthanasia | null;
}
export interface ZoonosesParameters {
  observation_days: number; collar_months: number; leish_sample_table: { population: number; sample: number }[];
  leish_focus_min_dogs: number; dogs_per_block: number; rabies_coverage_target: number;
  dog_population_estimate: number | null; cat_population_estimate: number | null;
}
export interface ZoonosesParameterVersion { version: number; values: ZoonosesParameters; created_at: string | null }
type Coded = { code: string; label: string }[];
// Q25: proposta; um serializador que estende o `zoonoses_domains` do field-sync (§11) com o que as telas precisam.
export interface ZoonosesDomains {
  species: Coded; sexes: Coded; sizes: Coded; animal_statuses: Coded; dpp_results: Coded; elisa_results: Coded; modalities: Coded;
  demand_subtypes: Coded; demand_states: Coded; demand_outcomes: Coded; observation_outcomes: Coded; survey_kinds: Coded;
  leish_decisions: Coded; collar_removal_reasons: Coded; entry_kinds: Coded; exit_kinds: Coded; euthanasia_reasons: Coded; legal_bases: Coded;
  products: (VectorProductRef & { use: "rabies_vaccine" | "collar" })[];
  open_campaign: { id: string; name: string; starts_on: string; ends_on: string } | null;
  parameters: { collar_months: number; observation_days: number };
  units: ZooRef[]; shelter_units: ZooRef[];
}
export type ZoonosesReportName = "rabies_weekly" | "rabies_modalities" | "demands" | "observations" | "leish" | "shelter";
export type ZoonosesReportFilters = Partial<{ from: string; to: string; campaign_id: string; survey_id: string }>;   // Q25
export type ZoonosesReport = Omit<VectorReport, "name" | "provisional"> & { name: ZoonosesReportName };              // Q25 (forma da D-13 do 35a); o rabies_weekly traz as duas coberturas (Q19)
// Q25: o domicílio do animal sai daqui (o do cadastro individual ativo da pessoa); não há lista de domicílios.
export interface ZooPerson { patient_id: string | null; display_name: string | null; household: { id: string; address_short: string } | null }
```

- Funções (todas `Promise`; listas `{ items, next_before }` com `before` opcional):
  - `getZoonosesDomains()` → `GET /zoonoses/domains`; `lookupZooPerson(i: { cpf: string } | { cns: string }): ZooPerson | null` → `POST /zoonoses/person_lookup` (Q25, com trilha; `404`/corpo nulo → `null`).
  - Animal: `searchAnimals(i: AnimalSearch): Animal[]` → `POST /zoonoses/animals/search` (envelope `{ items }`); `createAnimal(i: AnimalInput): Animal`; `getAnimal(id): AnimalHistory`; `updateAnimal(id, i: AnimalPatch): Animal` (PATCH).
  - Campanha: `listCampaigns(): RabiesCampaign[]`; `createCampaign(i)`; `updateCampaign(id, i: Partial<CampaignInput> & { expected_version })`; `openCampaign(id, expectedVersion)`, `closeCampaign(id, expectedVersion)` → `{ expected_version }`.
  - Vacinação: `recordVaccination(i: VaccinationInput): RabiesVaccination`; `cancelVaccination(id, reason): RabiesVaccination` → `{ reason }`; `listVaccinations(q: { campaignId?: string; from?: string; to?: string; modality?: VaccinationModality; before? })`.
  - Demanda: `listDemands(q: { state?: DemandState[]; before? })`; `getDemand(id): ZoonosesDemandDetail`; `scheduleDemand(id, { scheduled_on, expected_version })`; `inspectDemand(id, { inspected_on, inspection_note, expected_version })`; `closeDemand(id, { outcome, text, expected_version })`; `declineDemand(id, { text, expected_version })`; `extendDemand(id, { reason, expected_version })` (todas `ZoonosesDemandDetail`).
  - Observação: `listObservations(q: { state?: ObservationState[]; before? })`; `getObservation(id): BiteObservation`; `getObservationContact(id): ObservationContact`; `closeObservation(id, { outcome, outcome_on, outcome_note?, expected_version })`; `linkObservationAnimal(id, { animal_id, expected_version })`.
  - LV: `listLeishSurveys(): LeishSurvey[]`; `createLeishSurvey(i)`; `updateLeishSurvey(id, Partial<LeishSurveyInput> & { expected_version })`; `openLeishSurvey(id, v)`, `closeLeishSurvey(id, v)`; `listLeishTests(q: { surveyId?: string; pending?: "elisa" | "outcome"; before? })`; `collectLeishTest(i: { leish_survey_id?: string; animal_id: string; collected_on: string; dpp_result?: DppResult; dpp_on?: string }): LeishTest`; `updateLeishTest(id, i: Partial<Pick<LeishTest, "dpp_result" | "dpp_on" | "elisa_result" | "elisa_on" | "elisa_request">> & { expected_version })`; `recordLeishOutcome(i: { animal_id; leish_test_id; decision; decided_on; note?; euthanasia_id? })` (Q18: `euthanasia_id` obrigatório com `euthanasia`).
  - Coleira: `placeCollar(i: { animal_id; vector_product_id; placed_on }): Collar`; `removeCollar(id, { removed_on, removal_reason, expected_version })`; `listCollars(q: { due?: boolean; before? })` → `?due=1`.
  - Canil: `listShelterStays(q: { state: "open" | "closed"; healthUnitId?: string; before? })`; `enterShelter(i: ShelterEntryInput): ShelterStay`; `exitShelter(id, i: ShelterExitInput): ShelterStay`; `recordNeutering(i: { animal_id; performed_on; shelter_stay_id? }): Neutering`; `recordEuthanasia(i: EuthanasiaInput): Euthanasia`.
  - `getZooParameters(): ZoonosesParameterVersion`; `putZooParameters(values, expectedVersion)` → `{ values, expected_version }`.
  - `getZoonosesReport(name, f): ZoonosesReport`; `fetchZoonosesReportFile(name, f): SignatureFile` (`binaryFetch` em `/zoonoses/reports/<name>.csv`, `text/csv`).
- Extensões:
  - 25: `SurveillanceNotification` + `animal_observation?: { state: string; ends_on: string; outcome: string | null; outcome_on: string | null } | null` e `animal_outcome_pending?: boolean` (Q16).
  - 27a: `StockMovementKind` + `"animal_vaccination" | "animal_vaccination_reversal" | "collar_placement"`.
  - 33: `OmbudsmanSubject.code?: string | null` e o mesmo nos `children` (Q14); `NewManifestationInput` + `zoonoses_subtype?: DemandSubtype`, `place_address?: string` (Q14; `ombCompact` inclui as duas).
  - 35a: `type VectorProductUse = Exclude<MeasureMethod, "mechanical"> | "rabies_vaccine" | "collar"` em `VectorProduct.uses` e `VectorProductInput.uses`; `VectorShortfall.visit_on: string | null`, `source?: "inspection" | "vaccination" | "collar" | (string & {})`, `occurred_on?: string` (Q17).
- `src/lib/features.ts`: `FeatureKey` + `"zoonoses"`; `FEATURE_LABEL.zoonoses = "Centro de zoonoses"`.
- `src/devProxy.ts`: `MODULE_35B_PROXY_KEYS = [ "^/zoonoses(/|\\?|$)" ]`; `vite.config.ts` espalha como o 35a.
- Fixtures (`src/test/zoonosesFixtures.ts`): `NOW35B`, `TODAY35B`, `ZOO_FEATURES = [ "clinical_record", "territory_registry", "zoonoses" ]`, `zooUser(roles = [ "zoonoses_team" ], features = ZOO_FEATURES)`, `VACCINE` (`{ id: "pz1", code: "VAR", name: "Vacina antirrábica animal", unit: "un", use: "rabies_vaccine" }`), `COLLAR` (`pz2`, "Coleira deltametrina 4%", `collar`), `ZOO_DOMAINS` (rótulos de `config/zoonoses/domains.yml` do api, conferidos na Task 0; os subtipos com os rótulos do §4), `ZOO_PARAMS` (o padrão do §9 sem `leish_expected_prevalence`, Q18; `dog_population_estimate: null`), `animal(o?)` (cão "Thor", tutor `{ id: "pt1", display_name: "M. S." }`, `has_microchip: true`), `PERSON` (`{ patient_id: "pt1", display_name: "M. S.", household: { id: "h1", address_short: "R. das Flores, 10" } }`), `history(o?)`, `campaign(o?)` ("Campanha antirrábica 2026", `open`, metas 800/300), `vaccination(o?)`, `demand(o?)` (escorpião), `demandDetail(o?)`, `observation(o?)` (`exposure_on: "2026-11-15"`, `ends_on: "2026-11-25"`), `CONTACT`, `survey(o?)`, `leishTest(o?)`, `collar(o?)`, `stay(o?)`, `euthanasia(o?)`, `vetProfessional(o?)` (`MyProfessional` com CBO `223305`, `council: "CRMV"`, `council_state: "PR"`, `registration_number: "12345"`), `zooReport(o?)`.

- [ ] **Step 1: Testes que falham** — `src/lib/api.zoonoses.test.ts` (molde `src/lib/api.vectorControl.test.ts`):

```ts
it("busca do animal: microchip só no corpo, nunca na URL", async () => {
  stub({ items: [ animal() ] });
  expect(await searchAnimals({ microchip: "985112003456789" })).toEqual([ animal() ]);
  expect([ last().url, last().init.method ]).toEqual([ "/zoonoses/animals/search", "POST" ]);
  expect(bodyOf()).toEqual({ microchip: "985112003456789" });
  expect(last().url).not.toContain("985");
});
it("person_lookup: documento no corpo; traz o domicílio da pessoa; nada achado = null", async () => {
  stub(PERSON); expect((await lookupZooPerson({ cpf: "52998224725" }))?.household?.id).toBe("h1");
  stub(null); expect(await lookupZooPerson({ cns: "898001160651234" })).toBeNull();
  expect([ last().url, bodyOf() ]).toEqual([ "/zoonoses/person_lookup", { cns: "898001160651234" } ]);
});
it("filtros: só ids, datas, estados, due=1 e cursor; listas como chave[]", async () => {
  stub({}); expect(await listCollars({ due: true })).toEqual({ items: [], next_before: null });
  expect(last().url).toBe("/zoonoses/collars?due=1");
  stub({}); await listDemands({ state: [ "open", "scheduled" ], before: "c1" });
  expect(last().url).toBe("/zoonoses/demands?state%5B%5D=open&state%5B%5D=scheduled&before=c1");
  stub({ columns: [], rows: [] }); await getZoonosesReport("rabies_weekly", { campaign_id: "cp1" });
  expect(last().url).toBe("/zoonoses/reports/rabies_weekly?campaign_id=cp1");
});
it("notas, motivos, contato e texto só no corpo ou na resposta", async () => {
  stub(vaccination()); await cancelVaccination("v1", "dose registrada no animal errado");
  expect([ last().url, bodyOf() ]).toEqual([ "/zoonoses/vaccinations/v1/cancel", { reason: "dose registrada no animal errado" } ]);
  stub(CONTACT); expect(await getObservationContact("ob1")).toEqual(CONTACT);
  expect([ last().url, last().init.method ?? "GET" ]).toEqual([ "/zoonoses/observations/ob1/contact", "GET" ]);
});
it("transições levam só expected_version", async () => {
  stub(campaign()); await openCampaign("cp1", 3);
  expect([ last().url, bodyOf() ]).toEqual([ "/zoonoses/campaigns/cp1/open", { expected_version: 3 } ]);
});
it("CSV do relatório pelo binaryFetch, com o nome do api", async () => {
  stubBlob(200, "a;b\r\n", 'attachment; filename="rabies_weekly-2026-11-23.csv"');
  expect((await fetchZoonosesReportFile("rabies_weekly", { campaign_id: "cp1" })).filename).toBe("rabies_weekly-2026-11-23.csv");
  expect(last().url).toBe("/zoonoses/reports/rabies_weekly.csv?campaign_id=cp1");
});
it("manifestação da equipe leva zoonoses_subtype e place_address só quando preenchidos", async () => { /* createOmbudsmanManifestation com e sem as chaves */ });
```

Em `src/lib/features.test.ts`: `featureLabel("zoonoses")` = `"Centro de zoonoses"`. Em `src/devProxy.test.ts`: as chaves do 35b casam `/zoonoses`, `/zoonoses?x=1`, `/zoonoses/reports/leish.csv`, e não `/zoonosesX` nem `/dashboard/zoonoses`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod35b src/lib/api.zoonoses.test.ts src/lib/features.test.ts src/devProxy.test.ts`. Expected: FAIL (funções inexistentes).
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS; `tsc` limpo (a troca de `VectorShortfall.visit_on` para nulável pode pedir `?? "—"` em `VectorStockShortfalls.tsx`: faça aqui, sem mudar comportamento).
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.zoonoses.test.ts src/lib/features.ts src/lib/features.test.ts src/devProxy.ts src/devProxy.test.ts vite.config.ts src/test/zoonosesFixtures.ts src/modules/VectorStockShortfalls.tsx
$G commit -m "feat: add the zoonoses API client, notification, stock and ombudsman type extensions and dev proxy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Frases, portões, papéis, rótulos fixos e regras de tela

> **Q24:** `src/lib/zoonosesErrors.ts` (`ZOONOSES_ERRORS`) é a fonte das frases que o campo copia, e a Task 5 do campo espera este commit. Por isso a tabela vai num commit próprio (Step 2a), antes do resto da task, e o relato ao coordenador sai logo depois dele, com o SHA.

**Files:**
- Create: `src/lib/zoonosesErrors.ts`, `src/lib/zoonosesErrors.test.ts`, `src/lib/zoonoses.ts`, `src/lib/zoonoses.test.ts`

**Interfaces:**
- Consumes: `canUseTerritory` (24a); `textProblem`, `homeVisitRefusals`, `HOME_VISIT_ERRORS`, `STATIC_QUERY`, `NOMINAL_QUERY` (24b); `OMBUDSMAN_ERRORS.forwarding_changed` (33); `hasFeature`; `MyProfessional`.
- Produces (`src/lib/zoonosesErrors.ts`):
  - `ZOONOSES_DISABLED = "o centro de zoonoses está desligado nesta cidade"`.
  - `ZOONOSES_ERROR_CODES` e `ZOONOSES_ERRORS: Record<code, string>` — uma frase por código, todas distintas, sem `_`, sem dado de pessoa:

| Código | Frase |
|---|---|
| `feature_disabled` | `ZOONOSES_DISABLED` |
| `animal_changed` | "o animal mudou enquanto a tela estava aberta — os dados foram relidos; confira antes de repetir" |
| `microchip_taken` | "este microchip já está em outro animal da cidade" |
| `registry_number_taken` | "este número de registro já está em outro animal da cidade" |
| `campaign_already_open` | "já há uma campanha aberta — encerre-a antes de abrir outra" |
| `already_vaccinated_in_campaign` | "este animal já foi vacinado nesta campanha" |
| `campaign_not_open` | "a campanha não está aberta" |
| `survey_not_open` | "o inquérito não está aberto" |
| `animal_not_dog` | "coleta de sorologia e coleira são só para cão" |
| `animal_not_active` | "o animal não está ativo" |
| `reserved_subject` | "este assunto é reservado ao centro de zoonoses e não pode ser desativado com o centro ligado" |
| `demand_changed` | "a demanda mudou enquanto a tela estava aberta — os dados foram relidos" |
| `forwarding_changed` | a frase de `OMBUDSMAN_ERRORS.forwarding_changed` |
| `date_out_of_range` (Q20) | "a data está fora do período permitido" |
| `household_not_in_survey` (Q20) | "o domicílio não está nos quarteirões deste inquérito" |
| `observation_changed` | "a observação mudou enquanto a tela estava aberta — os dados foram relidos" |
| `observation_not_finished` | "o desfecho sadio só vale a partir do último dia da observação" |
| `elisa_without_reactive_dpp` | "o ELISA só é lançado depois de um TR-DPP reagente" |
| `test_changed` | "o exame mudou enquanto a tela estava aberta — os dados foram relidos" |
| `stay_already_open` | "o animal já tem uma estada aberta no canil" |
| `stay_changed` | "a estada mudou enquanto a tela estava aberta — os dados foram relidos" |
| `veterinarian_required` | "só o médico veterinário (CBO 223305, com CRMV) registra esta ação" |
| `already_euthanized` | "este animal já tem eutanásia registrada" |
| `parameters_changed` | "os parâmetros foram salvos por outra pessoa — os dados foram relidos; confira antes de salvar" |
| `expected_version_required` | "a tela não mandou a versão lida — recarregue e tente de novo" |
| `invalid_parameter` (proposto; Task 0 confere) | "valor fora da faixa permitida para este parâmetro" |
| `invalid_transition` (proposto; Task 0 confere) | "a situação mudou enquanto a tela estava aberta — confira e tente de novo" |
| `euthanasia_required` (proposto; Task 0 confere) | "registre antes a eutanásia do animal" |
| `tutor_required` (proposto; Task 0 confere) | "o animal precisa de tutor para esta ação" |
| `campaign_changed`, `survey_changed`, `collar_changed` (propostos; Task 0 confere) | "a campanha mudou…", "o inquérito mudou…", "a coleira mudou…" + " enquanto a tela estava aberta — os dados foram relidos" |

- Produces (`src/lib/zoonoses.ts`):
  - Portões: `zoonosesUsable(user)` (não operador, `zoonoses` e `canUseTerritory`, sem papel: o que a ouvidoria usa na Task 14); `canUseZoonoses(user)` (`zoonosesUsable` e um papel de leitura, Q10); `zoonosesRoles(user): { team; coordinator; admin; reader }`; `VET_CBO = "223305"`; `isVeterinarian(me: MyProfessional | null, roles: ZoonosesRoles): boolean` (Q12: `roles.team` **e** `council === "CRMV"`, `council_state` e `registration_number` preenchidos, e um vínculo ativo `cbo_code === VET_CBO` e `ended_at === null`); `vetIdentity(me): string` → `"<nome> · CRMV-<UF> <número>"`; `ZOO_NO_ROLE = "seu papel não permite usar o centro de zoonoses"`; `ZOO_NOT_VET = "só o médico veterinário (CBO 223305, com CRMV e vínculo na unidade de vigilância ambiental) registra castração e eutanásia"`.
  - Chaves: `ZOO_DOMAINS_KEY`, `ZOO_ANIMAL_KEY`, `ZOO_CAMPAIGNS_KEY`, `ZOO_VACCINATIONS_KEY`, `ZOO_DEMANDS_KEY`, `ZOO_DEMAND_KEY`, `ZOO_OBSERVATIONS_KEY`, `ZOO_OBSERVATION_KEY`, `ZOO_SURVEYS_KEY`, `ZOO_TESTS_KEY`, `ZOO_COLLARS_KEY`, `ZOO_STAYS_KEY`, `ZOO_PARAMS_KEY`, `ZOO_REPORT_KEY`, `ZOO_ME_KEY`; reexporta `STATIC_QUERY`, `NOMINAL_QUERY`.
  - Rótulos: `zooLabel(list: Coded | undefined, code)` (código cru quando falta); fixos, para quem não lê os domínios: `ZOONOSES_SUBTYPES` (os oito do §4, na ordem, com os rótulos do contrato), `observationStateLabel` (`open` "em observação", `closed` "encerrada"; Q16: sem `cancelled`), `observationOutcomeLabel` (`healthy` "sadio", `died` "morreu", `missing` "desapareceu", `rabies_signs` "sinais de raiva", `sent_to_exam` "enviado a exame", `euthanized` "eutanásia") — rótulos propostos, conferir com o `domains.yml` na Task 0.
  - Regras (`ZooProblem = { code: string | null; path: string; text: string }`):
    - `animalProblems(d: AnimalDraft): ZooProblem[]` (tamanhos da Global Constraints; microchip `^\d{15}$` em `microchip`; tutor e `community` juntos → `community`; sem tutor e sem `community` vale, "tutor não identificado", Q22).
    - `nextAnimalStatuses(status): AnimalStatus[]` (`active` → os três; os outros → `[]`).
    - `vaccinationProblems(d: { modality; animalId; species; vaccinatedOn; adverse; adverseNote; householdId }, c: { campaign: RabiesCampaign | null; fieldUsable: boolean }, today)`: `house` exige campanha aberta, animal, domicílio e `fieldUsable`; `fixed_post` exige campanha aberta e, sem animal, `species` dog/cat; `routine` exige animal e nenhuma campanha; `vaccinated_on` entre `campaign.starts_on` (rotina: hoje − 30) e hoje; nota adversa 10–500 só com `adverse`.
    - `suggestedGoal(estimate: number | null, target: number): number | null` → `Math.round(estimate × target)`.
    - `closeObservationProblems(d: { outcome; outcomeOn; note }, o: Pick<BiteObservation, "exposure_on" | "ends_on">, today)`: desfecho obrigatório; `outcome_on` ≥ `exposure_on` e ≤ hoje; `healthy` com `outcome_on < ends_on` → `{ code: "observation_not_finished", path: "outcome_on" }`; nota opcional 10–500.
    - `observationDaysLeft(endsOn, today): number` (dias corridos; 0 no último dia; negativo depois).
    - `elisaAllowed(t: LeishTest): boolean` (`dpp_result === "reactive"`); `isReactiveDog(t)` (DPP e ELISA reagentes); `leishOutcomeProblems(d: { decision; euthanasiaId }, a: { tutor: boolean })` (Q18: `euthanasia` sem `euthanasia_id` → `euthanasia_required`; `tutor_treatment`/`refused` sem tutor → `tutor_required`).
    - `stayExitProblems(d: { exitKind; adopterId }, a: { tutor: boolean; euthanized: boolean })` (`returned` sem tutor → `tutor_required`; `adopted` sem adotante → `adopter_patient_id`; `euthanasia` sem eutanásia → `euthanasia_required`).
    - `euthanasiaProblems(d: { reason; note; legalBasis: string[]; performedOn }, today)` (motivo; nota 10–1000 em `reason_note`; ao menos uma base em `legal_basis`; data ≤ hoje).
    - `demandProblems(action: "schedule" | "inspect" | "close" | "decline" | "extend", d: { date?; note?; outcome?; text? }, today)` (agenda: data ≥ hoje; vistoria: data ≤ hoje e nota 10–2000; encerramento: desfecho e texto 20–4000; recusa e prorrogação: 10–4000).
    - `demandActions(state): ("schedule" | "inspect" | "close" | "decline" | "extend")[]` (`open`: agendar, vistoriar, encerrar, recusar, prorrogar; `scheduled`: vistoriar, encerrar, recusar, prorrogar; `inspected`: encerrar, recusar, prorrogar; `closed`: nada; prorrogar só sem `forwarding.extended`).
    - `sampleTableProblems(rows)` (população e amostra inteiras > 0, populações estritamente crescentes) e `parameterProblems(v: ZoonosesParameters)` com as faixas do §9, sem `leish_expected_prevalence` (Q18) (`path` `values.<chave>`).
  - Recusas: `zoonosesRefusals(err)` (o parser do 24b com `{ ...HOME_VISIT_ERRORS, ...ZOONOSES_ERRORS }`; `401 mfa_required` diz o step-up); `zoonosesError(err)`; `isZoonosesStale(err)` (404, os 409 da Global Constraints).

- [ ] **Step 1: Testes que falham**

`src/lib/zoonosesErrors.test.ts`: cada código de `ZOONOSES_ERROR_CODES` tem uma frase e só uma; frases distintas, sem `_`, mais de 10 caracteres; `ZOONOSES_ERRORS.feature_disabled` = `ZOONOSES_DISABLED`; `ZOONOSES_ERRORS.forwarding_changed` = `OMBUDSMAN_ERRORS.forwarding_changed`.

`src/lib/zoonoses.test.ts`:

```ts
it("portão: zoonoses, cadastro territorial utilizável e um papel de leitura (Q10); operador nunca", () => {
  expect(canUseZoonoses(zooUser())).toBe(true);
  expect(canUseZoonoses(zooUser([ "municipal_admin" ]))).toBe(true);
  expect(canUseZoonoses(zooUser([ "health_professional" ]))).toBe(false);
  expect(canUseZoonoses(zooUser(undefined, [ "clinical_record", "zoonoses" ]))).toBe(false);
  expect(canUseZoonoses({ ...zooUser(), operator: true })).toBe(false);
});
it("papéis: a coordenação de endemias é a do centro (Q1); admin só lê", () => {
  expect(zoonosesRoles(zooUser([ "endemic_coordinator" ]))).toEqual({ team: true, coordinator: true, admin: false, reader: true });
  expect(zoonosesRoles(zooUser([ "municipal_admin" ]))).toEqual({ team: false, coordinator: false, admin: true, reader: true });
});
it("veterinário: papel da equipe, CBO 223305, CRMV completo e vínculo ativo (Q12)", () => {
  const team = zoonosesRoles(zooUser());
  expect(isVeterinarian(vetProfessional(), team)).toBe(true);
  expect(isVeterinarian(vetProfessional(), zoonosesRoles(zooUser([ "health_professional" ])))).toBe(false);
  expect(vetIdentity(vetProfessional())).toBe("Ana Veterinária · CRMV-PR 12345");
  expect(isVeterinarian(vetProfessional({ council: "CRM" }), team)).toBe(false);
  expect(isVeterinarian(vetProfessional({ linkEnded: true }), team)).toBe(false);
  expect(isVeterinarian(null, team)).toBe(false);
});
it.each([ [ "2026-11-24", true ], [ "2026-11-25", false ], [ "2026-11-23", true ] ] as const)(
  "sadio em %s com ends_on 2026-11-25 → observation_not_finished = %s (10º dia exato vale)", (outcomeOn, refused) => {
  const today = "2026-11-26";
  const p = closeObservationProblems({ outcome: "healthy", outcomeOn, note: "" }, observation(), today);
  expect(p.some((x) => x.code === "observation_not_finished" && x.path === "outcome_on")).toBe(refused);
});
it("desfecho antes da exposição ou no futuro recusa em outcome_on", () => { /* "2026-11-14" e "2026-11-27" com today 2026-11-26 */ });
it("dias que faltam: 2 no dia 23, 0 no dia 25, -1 no dia 26", () => {
  expect([ "2026-11-23", "2026-11-25", "2026-11-26" ].map((d) => observationDaysLeft("2026-11-25", d))).toEqual([ 2, 0, -1 ]);
});
it("vacinação: posto fixo sem animal exige espécie; rotina sem campanha; casa a casa exige domicílio e campo", () => {
  const c = { campaign: campaign(), fieldUsable: true };
  expect(vaccinationProblems({ modality: "fixed_post", animalId: null, species: null, vaccinatedOn: TODAY35B, adverse: false, adverseNote: "", householdId: null }, c, TODAY35B).map((p) => p.path)).toEqual([ "species" ]);
  expect(vaccinationProblems({ modality: "house", animalId: "a1", species: null, vaccinatedOn: TODAY35B, adverse: false, adverseNote: "", householdId: null }, { ...c, fieldUsable: false }, TODAY35B).map((p) => p.path)).toEqual([ "household_id", "modality" ]);
});
it("vacinação: data entre o início da campanha (rotina: 30 dias antes) e hoje", () => { /* 2026-10-31 recusa na campanha que começa 2026-11-01; rotina 2026-10-24 vale, 2026-10-23 recusa; amanhã recusa */ });
it("meta sugerida = round(estimativa × cobertura); sem estimativa, null", () => {
  expect(suggestedGoal(1001, 0.8)).toBe(801);
  expect(suggestedGoal(null, 0.8)).toBeNull();
});
it("ELISA só depois de DPP reagente; cão reagente com os dois", () => {
  expect(elisaAllowed(leishTest({ dpp_result: "non_reactive" }))).toBe(false);
  expect(isReactiveDog(leishTest({ dpp_result: "reactive", elisa_result: "reactive" }))).toBe(true);
});
it("desfecho da LV e saída do canil: eutanásia exige a eutanásia; tratamento, recusa e devolução exigem tutor", () => {
  expect(leishOutcomeProblems({ decision: "euthanasia", euthanasiaId: null }, { tutor: true })[0].code).toBe("euthanasia_required");
  expect(leishOutcomeProblems({ decision: "refused", euthanasiaId: null }, { tutor: false })[0].code).toBe("tutor_required");
  expect(stayExitProblems({ exitKind: "adopted", adopterId: null }, { tutor: false, euthanized: false }).map((p) => p.path)).toEqual([ "adopter_patient_id" ]);
});
it("eutanásia: nota 10–1000 e ao menos uma base legal", () => {
  expect(euthanasiaProblems({ reason: "leishmaniasis", note: "curta", legalBasis: [], performedOn: TODAY35B }, TODAY35B).map((p) => p.path)).toEqual([ "reason_note", "legal_basis" ]);
});
it("demanda: ações por estado; prorrogar só uma vez", () => {
  expect(demandActions(demand({ state: "inspected" }))).toEqual([ "close", "decline", "extend" ]);
  expect(demandActions(demand({ state: "open", forwarding: { ...demand().forwarding, extended: true } }))).not.toContain("extend");
  expect(demandActions(demand({ state: "closed" }))).toEqual([]);
});
it("tabela da amostra: populações crescentes, inteiros > 0", () => { /* [{1000,100},{900,90}] → path values.leish_sample_table[1].population */ });
it("recusas: código do 35b com a frase; 403 zoonoses diz o interruptor; código novo cru; 409 é velho", () => {
  expect(zoonosesError(new ApiError(422, { error: "microchip_taken", path: "microchip" }, "x"))).toBe(ZOONOSES_ERRORS.microchip_taken);
  expect(zoonosesError(new ApiError(403, { error: "feature_disabled", feature: "zoonoses" }, "x"))).toBe(ZOONOSES_DISABLED);
  expect(zoonosesError(new ApiError(422, { error: "brand_new" }, "x"))).toBe("recusado pelo servidor (brand_new)");
  expect(isZoonosesStale(new ApiError(409, { error: "campaign_already_open" }, "x"))).toBe(true);
  expect(isZoonosesStale(new ApiError(422, { error: "already_vaccinated_in_campaign" }, "x"))).toBe(false);
});
```

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod35b src/lib/zoonosesErrors.test.ts src/lib/zoonoses.test.ts`.
- [ ] **Step 2a: As frases primeiro (Q24)** — implemente `src/lib/zoonosesErrors.ts`; `$R/bin/test dashboard -w dashboard-mod35b src/lib/zoonosesErrors.test.ts` passa; `tsc`; commit e relato com o SHA à sessão "Próximos módulos" (a Task 5 do campo espera):

```bash
cd $W && $G add src/lib/zoonosesErrors.ts src/lib/zoonosesErrors.test.ts
$G commit -m "feat: add the zoonoses refusal phrases shared with the field app" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 3: Implemente `src/lib/zoonoses.ts`.** **Step 4: Rode e veja passar (os dois arquivos); `tsc`.**
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/zoonoses.ts src/lib/zoonoses.test.ts
$G commit -m "feat: add zoonoses gates, roles, veterinarian check and screen rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Zoonoses" e o portão (F-35.11)

**Files:**
- Create: `src/shell/modules.zoonoses.test.ts`, `src/modules/zoonoses/ZoonosesGate.tsx`, `src/modules/zoonoses/ZoonosesGate.test.tsx`, cascas (`PageHeader` + `EmptyState` "Em construção") de `src/modules/{ZooAnimals,ZooCampaigns,ZooDemands,ZooObservations,ZooLeish,ZooShelter,ZooReports,ZooSettings}.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts` (contagens `NAV_GROUPS.length - N` sobem 1 onde o grupo some), `src/shell/catalog.ts`, `src/App.tsx`

**Interfaces:**
- Consumes: `canUseZoonoses`, `zoonosesRoles`, `ZOO_NO_ROLE`, `ZOONOSES_DISABLED` (Task 2).
- Produces:
  - `ModuleId` + `"zoo-animals" | "zoo-campaigns" | "zoo-demands" | "zoo-observations" | "zoo-leish" | "zoo-shelter" | "zoo-reports" | "zoo-settings"`.
  - Grupo `{ label: "Zoonoses", items: [ zoo-animals "Animais" ◉, zoo-campaigns "Campanhas e vacinação" ✚, zoo-demands "Demandas" ⚑, zoo-observations "Observação do agressor" ◎, zoo-leish "Leishmaniose" ⚗, zoo-shelter "Canil e gatil" ⌂, zoo-reports "Relatórios de zoonoses" ▤, zoo-settings "Parâmetros de zoonoses" ⚙ ] }` logo **depois de "Endemias"** (Q25).
  - Filtro em `navGroupsFor` (`zooOn = canUseZoonoses(user)`, `zoo = zoonosesRoles(user)`), **antes** do `item.id.startsWith("vector-")`: `zoo-campaigns`, `zoo-reports` → `zooOn && zoo.reader`; `zoo-settings` → `zooOn && zoo.coordinator`; os outros → `zooOn && zoo.team`.
  - `CATALOG.groups.Zoonoses = { icon: IconPaw, description: "Animais com tutor, vacinação antirrábica, demandas, observação do agressor, leishmaniose canina e canil" }` e as 8 telas com ícone e descrição.
  - `renderModule`: um `case` por tela (`ZooShelter` e `ZooSettings` com `onGoToSecurity`).
  - `ZoonosesGate({ title, sub, allow(roles), children(roles) })`: operador → `ZOO_NO_ROLE`; sem `canUseZoonoses` → `ZOONOSES_DISABLED` (sem ler nada); `allow` falso → `ZOO_NO_ROLE`; senão `children`.

- [ ] **Step 1: Testes que falham** — `src/shell/modules.zoonoses.test.ts`:

```ts
const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).find((g) => g.label === "Zoonoses")?.items.map((i) => i.id) ?? [];
const TEAM = [ "zoo-animals", "zoo-campaigns", "zoo-demands", "zoo-observations", "zoo-leish", "zoo-shelter", "zoo-reports" ];
it("equipe vê tudo menos parâmetros; coordenação, tudo; admin, campanhas e relatórios", () => {
  expect(ids(zooUser())).toEqual(TEAM);
  expect(ids(zooUser([ "endemic_coordinator" ]))).toEqual([ ...TEAM, "zoo-settings" ]);
  expect(ids(zooUser([ "municipal_admin" ]))).toEqual([ "zoo-campaigns", "zoo-reports" ]);
});
it("sem zoonoses, sem cadastro territorial ou só health_professional: o grupo some", () => {
  expect(ids(zooUser(undefined, [ "clinical_record", "territory_registry" ]))).toEqual([]);
  expect(ids(zooUser(undefined, [ "clinical_record", "zoonoses" ]))).toEqual([]);
  expect(ids(zooUser([ "health_professional" ]))).toEqual([]);
});
it("Zoonoses vem logo depois de Endemias, com ícone e descrição", () => {
  const labels = NAV_GROUPS.map((g) => g.label);
  expect(labels.indexOf("Zoonoses")).toBe(labels.indexOf("Endemias") + 1);
  expect(CATALOG.groups.Zoonoses.description).toBe("Animais com tutor, vacinação antirrábica, demandas, observação do agressor, leishmaniose canina e canil");
});
```

`ZoonosesGate.test.tsx`: sem o interruptor mostra `ZOONOSES_DISABLED` e não chama `children`; papel errado → `ZOO_NO_ROLE`; equipe → `children` com os papéis.

- [ ] **Step 2: Rode e veja falhar.** **Step 3: Implemente.** **Step 4: Rode e veja passar** (`src/shell/modules.zoonoses.test.ts src/shell/modules.test.ts src/shell/modules.vectorControl.test.ts src/shell/catalog.test.ts src/modules/zoonoses/ZoonosesGate.test.tsx src/App.navigation.test.tsx`); `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.zoonoses.test.ts src/shell/catalog.ts src/App.tsx src/modules/zoonoses/ZoonosesGate.tsx src/modules/zoonoses/ZoonosesGate.test.tsx src/modules/ZooAnimals.tsx src/modules/ZooCampaigns.tsx src/modules/ZooDemands.tsx src/modules/ZooObservations.tsx src/modules/ZooLeish.tsx src/modules/ZooShelter.tsx src/modules/ZooReports.tsx src/modules/ZooSettings.tsx
$G commit -m "feat: add the zoonoses menu group and screen gate" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Insumos do 35a com `zoonoses` — produtos, local, quarteirões, faltas, estoque e unidade (F-35.12, F-35.15)

O 35b funciona sem o 35a (§1): a vacina e a coleira são produtos do 35a (Q3), a baixa é do local da UVZ (§3), o inquérito usa os quarteirões (Q8). Sem esta task, uma cidade só com `zoonoses` não cadastra a vacina.

**Files:**
- Create: `src/modules/VectorSettings.zoonoses.test.tsx`
- Modify: `src/lib/vectorControl.ts`, `src/lib/vectorControl.test.ts`, `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts`, `src/shell/modules.ts`, `src/shell/modules.zoonoses.test.ts`, `src/modules/vectorControl/VectorGate.tsx`, `src/modules/VectorSettings.tsx`, `src/modules/vectorControl/ProductsPanel.tsx`, `src/modules/VectorBlocks.tsx`, `src/modules/VectorStockShortfalls.tsx`, `src/modules/VectorStockShortfalls.test.tsx`, `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitForm.vector.test.tsx`

**Interfaces:**
- Consumes: `canUseZoonoses` (Task 2); `VectorProductUse`, `VectorShortfall.source/occurred_on` (Task 1).
- Produces:
  - `src/lib/vectorControl.ts`: `canUseVectorSupplies(user) = canUseVectorControl(user) || canUseZoonoses(user)`; `vectorProductUses(user): VectorProductUse[]` (`focal`, `perifocal`, `spatial` com `canUseVectorControl`; `rabies_vaccine`, `collar` com `canUseZoonoses`); `ZOONOSES_USES = [ "rabies_vaccine", "collar" ]`.
  - `VectorGate` + prop `supplies?: boolean`: com ela, o interruptor confere `canUseVectorSupplies`; sem ela, como hoje.
  - `modules.ts`: `vector-settings` e `vector-blocks` → `(vcOn || zooOn) && vc.coordinator`; `vector-stock-shortfalls` → `(vcOn || zooOn) && (vc.coordinator || vc.stockKeeper)`; os outros `vector-*` seguem `vcOn`.
  - `pharmacy.ts`: `canUseStockScreens` = `canUsePharmacy || canUseVectorSupplies`; `MOVEMENT_LABEL.animal_vaccination = "vacinação antirrábica animal"`, `animal_vaccination_reversal = "estorno de vacinação antirrábica (anulada)"`, `collar_placement = "coleira colocada"`; `DOCUMENT_LABEL.rabies_vaccination = "vacinação antirrábica"`, `DOCUMENT_LABEL.collar = "coleira"`.
  - `VectorSettings`: `VectorGate supplies`; `ParametersPanel` só com `canUseVectorControl`; produtos e local sempre.
  - `ProductsPanel`: os usos oferecidos são `vectorProductUses(user)` mais os que o produto já tem (um uso fora do portão aparece marcado e desabilitado, nunca some); rótulos `rabies_vaccine` "vacina antirrábica animal", `collar` "coleira"; com um uso de zoonoses marcado, unidade "un" e carga "1,00" fixas e os usos do 35a desabilitados, e vice-versa (Q11: uma finalidade só — `rabies_vaccine` **ou** `collar` —, `un` e `1.00`, sem misturar com as vetoriais); `productProblems` recusa a mistura e as duas finalidades de zoonoses juntas em `uses`.
  - `VectorBlocks`: `VectorGate supplies`; sem `canUseVectorControl`, só a aba "Quarteirões" (imóveis e mapa são do 35a; Q11: o resto do `/vector_control` continua só com `vector_control`).
  - `VectorStockShortfalls`: `supplies`; coluna "Origem" (Q17, `source`: `inspection` "boletim", `vaccination` "vacinação antirrábica", `collar` "coleira", outro cru) e data `occurred_on ?? visit_on`.
  - `LocationsPanel`: `VECTOR_KIND` com `canUseVectorSupplies`. `Attendance`: `vectorOn={canUseVectorSupplies(user)}`.

- [ ] **Step 1: Testes que falham**
  - `vectorControl.test.ts`: `canUseVectorSupplies(zooUser([ "endemic_coordinator" ]))` = `true` sem `vector_control`; `vectorProductUses` só com `zoonoses` = `[ "rabies_vaccine", "collar" ]`; com os dois, os cinco.
  - `modules.zoonoses.test.ts`: coordenação só com `zoonoses` vê em "Endemias" exatamente `[ "vector-blocks", "vector-settings" ]` e em "Farmácia" `[ "vector-stock-shortfalls" ]`; a equipe não vê nenhum.
  - `VectorSettings.zoonoses.test.tsx`: só com `zoonoses` não há o painel de parâmetros do 35a e não lê `getVectorParameters`; produto novo "Vacina antirrábica animal" com uso `rabies_vaccine` manda `createVectorProduct({ code: "VAR", name: "Vacina antirrábica animal", unit: "un", charge_size: "1.00", uses: [ "rabies_vaccine" ], active: true })`; marcar `collar` desabilita focal/perifocal/espacial e `rabies_vaccine`; um produto antigo `focal` aparece com o uso marcado e desabilitado.
  - `pharmacy.test.ts`: `movementLabel("animal_vaccination")`, `movementLabel("collar_placement")`, `movementDocument({ type: "collar", number: null })` = "coleira".
  - `VectorStockShortfalls.test.tsx`: falta de vacinação mostra "vacinação antirrábica" e a data de `occurred_on`; a de boletim antiga (sem `source`) mostra "boletim".
  - `UnitForm.vector.test.tsx`: só com `zoonoses`, "Vigilância ambiental e zoonoses" é oferecido.
- [ ] **Step 2–4:** rode e veja falhar (os arquivos acima e `src/modules/VectorSettings.test.tsx src/modules/VectorBlocks.test.tsx src/modules/PharmacyAdmin.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/vectorControl.ts src/lib/vectorControl.test.ts src/lib/pharmacy.ts src/lib/pharmacy.test.ts src/shell/modules.ts src/shell/modules.zoonoses.test.ts src/modules/vectorControl/VectorGate.tsx src/modules/VectorSettings.tsx src/modules/vectorControl/ProductsPanel.tsx src/modules/VectorBlocks.tsx src/modules/VectorStockShortfalls.tsx src/modules/VectorStockShortfalls.test.tsx src/modules/pharmacy/LocationsPanel.tsx src/modules/Attendance.tsx src/modules/attendance/UnitForm.vector.test.tsx src/modules/VectorSettings.zoonoses.test.tsx
$G commit -m "feat: open vector supplies, city blocks and the environmental surveillance unit to zoonoses-only cities" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Animais — busca, cadastro, histórico e edição (F-35.11)

**Files:**
- Create: `src/modules/zoonoses/AnimalSearch.tsx`, `src/modules/zoonoses/AnimalForm.tsx`, `src/modules/zoonoses/AnimalHistory.tsx`, `src/modules/zoonoses/PersonLookup.tsx`, `src/modules/ZooAnimals.test.tsx`
- Modify: `src/modules/ZooAnimals.tsx`

**Interfaces:**
- Consumes: `searchAnimals`, `createAnimal`, `getAnimal`, `updateAnimal`, `lookupZooPerson`, `getZoonosesDomains` (Task 1); `animalProblems`, `nextAnimalStatuses`, `zooLabel`, `zoonosesRefusals`, `isZoonosesStale`, chaves (Task 2); `ZoonosesGate`.
- Produces:
  - `ZooAnimals()`: `ZoonosesGate allow={(r) => r.team}`; "Buscar animal" e "Cadastrar animal"; o animal aberto vira `AnimalHistory`.
  - `AnimalSearch({ onOpen })`: modo `SegmentedControl` (microchip | registro | tutor | domicílio); microchip e registro chamam `searchAnimals` **direto** (estado local, sem `useQuery`), no máximo 1 resultado; tutor pelo `PersonLookup` (`tutor_patient_id`) e domicílio pelo domicílio da pessoa buscada (`household.id`, Q25) → lista (nome, espécie, situação; **nunca** o tutor na lista); nada achado → "nenhum animal com este dado".
  - `PersonLookup({ value, onChange, label })`: CPF (11 dígitos, `isValidCpf`) ou CNS (15 dígitos) → `lookupZooPerson`; achado mostra só `display_name` e, quando houver, o domicílio do cadastro individual ativo (`household.address_short`, Q25); nada → "ninguém com este documento no cadastro"; "Trocar" limpa; o nome digitado nunca vale como tutor (§2).
  - `AnimalForm({ animal?, onSaved })`: campos do §1 com rótulos dos domínios; mês de nascimento estimado (`<input type="month">` → `AAAA-MM-01`); microchip só na criação; tutor (`PersonLookup`), "Comunitário (ninguém responde pelo animal)" ou nenhum dos dois (tutor não identificado, Q22) — tutor e comunitário não vão juntos; domicílio opcional e só pela busca de pessoa (Q25): "Usar o domicílio do tutor (<endereço curto>)" ou "Buscar o domicílio por outro morador" (outro `PersonLookup`, cujo resultado não vira tutor); sem domicílio no cadastro da pessoa, a linha `"esta pessoa não tem domicílio no cadastro individual"`; `animalProblems` antes de enviar; `422 microchip_taken`/`registry_number_taken` no campo; edição com `expected_version` e só o que mudou; situação só pelos `nextAnimalStatuses` com a data, com `ConfirmButton` ("a situação não volta a ativo").
  - `AnimalHistory({ id, onEdit })`: `getAnimal` com `NOMINAL_QUERY` (trilha `animal.viewed`); dados, tutor como `display_name`, microchip como "com microchip"/"sem microchip"; seções vacinações (anuladas marcadas), observações, exames de LV (etiqueta e resultados), coleiras, estadas, castrações e a eutanásia.

- [ ] **Step 1: Testes que falham** (`ZooAnimals.test.tsx`):

```tsx
it("busca por microchip: corpo, um resultado, nada no cache", async () => {
  // digita 985112003456789 → searchAnimals({ microchip: "985112003456789" }); abre o histórico
  // client.getQueryCache().getAll() — nenhuma queryKey contém "985112003456789"
});
it("busca por tutor: CPF no corpo do person_lookup; a lista não mostra o tutor", async () => { /* … */ });
it("domicílio vem da pessoa buscada: o do tutor ou o de outro morador, que não vira tutor (Q25)", async () => {
  // createAnimal({ …, tutor_patient_id: "pt1", household_id: "h1", community: false }); com outro morador: household_id dele, tutor_patient_id do tutor
});
it("sem tutor e sem comunitário vale (tutor não identificado, Q22)", async () => { /* … */ });
it("cadastro com tutor: corpo com tutor_patient_id e community false; o nome digitado não vai", async () => {
  // createAnimal({ species: "dog", name: "Thor", sex: "male", size: "medium", microchip: "985112003456789", tutor_patient_id: "pt1", community: false })
});
it("microchip repetido: 422 microchip_taken no campo, formulário fica", async () => { /* … */ });
it("comunitário limpa o tutor e vice-versa", async () => { /* … */ });
it("histórico: lê uma vez, tutor só pelo display_name, sem o número do microchip", async () => { /* getAnimal 1x; "M. S."; "com microchip" */ });
it("edição: expected_version; 409 animal_changed diz a frase e relê uma vez, sem reenviar", async () => { /* updateAnimal 1x; getAnimal 2x */ });
it("situação: ativo → morto com data; morto não oferece volta", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooAnimals.tsx src/modules/zoonoses/AnimalSearch.tsx src/modules/zoonoses/AnimalForm.tsx src/modules/zoonoses/AnimalHistory.tsx src/modules/zoonoses/PersonLookup.tsx src/modules/ZooAnimals.test.tsx
$G commit -m "feat: search, register and edit animals with their tutor and show the animal history" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Campanhas e vacinação — campanha, posto fixo, rotina, anulação e resultado semanal (F-35.12)

**Files:**
- Create: `src/modules/zoonoses/CampaignForm.tsx`, `src/modules/zoonoses/VaccinationForm.tsx`, `src/modules/zoonoses/WeeklyResult.tsx`, `src/modules/ZooCampaigns.test.tsx`
- Modify: `src/modules/ZooCampaigns.tsx`, `src/modules/vectorControl/ReportTable.tsx` (tipo da prop)

**Interfaces:**
- Consumes: campanhas, vacinações, `getZooParameters`, `getZoonosesReport`, `fetchZoonosesReportFile`, `getZoonosesDomains` (Task 1); `vaccinationProblems`, `suggestedGoal`, `zoonosesRefusals`, `isZoonosesStale`, `noteProblem` (Task 2); `AnimalSearch` (Task 5); `ReportTable` (35a; esta task troca a prop para `Omit<VectorReport, "name"> & { name: string }`, sem mudar o 35a); `saveFile`; `canUseHomeVisits`.
- Produces:
  - `ZooCampaigns({ today? })`: `ZoonosesGate allow={(r) => r.reader}`; campanhas (`DataTable`: nome, ano, período, metas, etapa); a coordenação cria/edita (`planned`), abre e encerra (`ConfirmButton`, `expected_version`); `409 campaign_already_open` → frase e uma releitura.
  - `CampaignForm`: nome 1–80, ano, início ≤ fim, metas inteiras ≥ 0; metas sugeridas por `suggestedGoal(dog_population_estimate, rabies_coverage_target)` (e de gatos) com a linha `"sugestão: N cães e M gatos (estimativa × cobertura de X%)"`; sem estimativa, sem sugestão.
  - `VaccinationForm({ campaign, today })` (equipe): modalidade — "Posto fixo" e "Casa a casa" só com campanha aberta (levam o `campaign_id` dela); "Rotina" sempre, com `campaign_id: null` e animal cadastrado (§3); casa a casa só com `canUseHomeVisits(user)` e animal com domicílio; animal por `AnimalSearch` (opcional no posto fixo: sem ele, só a espécie cão/gato e nenhum campo de nome ou tutor, Q4); unidade (`domains.units`) no posto fixo e na rotina; produto (`domains.products` com `use: "rabies_vaccine"`); "Evento adverso" marcado mostra **"Notifique o evento adverso no formulário do Ministério da Saúde (EATAV)."** e a nota 10–500; `422 already_vaccinated_in_campaign` na linha do animal (só o api barra a segunda dose, Q19); resposta com `stock_shortfall` mostra "dose registrada; faltou saldo no local — a coordenação concilia em Faltas de insumo de campo".
  - Lista das doses da campanha (`listVaccinations`): data, modalidade, espécie, animal (nome ou "sem cadastro"), unidade; "Anular" (quem registrou ou a coordenação; motivo 10–500) → `cancelVaccination`; anulada some das contas e aparece marcada.
  - `WeeklyResult({ campaignId })`: "Ver resultado semanal" lê `getZoonosesReport("rabies_weekly", { campaign_id })` (`STATIC_QUERY`, só no clique) → `ReportTable`, com as duas coberturas que o api manda (sobre a meta e, quando há estimativa, sobre a população; Q19); "Baixar CSV" → `saveFile(file, file.filename ?? "rabies_weekly-<data>.csv")`.

- [ ] **Step 1: Testes que falham** (`ZooCampaigns.test.tsx`):

```tsx
it("posto fixo sem cadastro: só a espécie; nenhum campo de nome ou tutor", async () => {
  // recordVaccination({ modality: "fixed_post", species: "cat", campaign_id: "cp1", vaccinated_on: TODAY35B, health_unit_id: "u1", vector_product_id: "pz1", adverse_event: false })
});
it("rotina: campaign_id null e animal cadastrado, mesmo com campanha aberta", async () => { /* … */ });
it("evento adverso mostra o lembrete do EATAV e pede a nota", async () => {
  // screen.getByText("Notifique o evento adverso no formulário do Ministério da Saúde (EATAV).")
});
it("segunda dose: 422 already_vaccinated_in_campaign no animal; o formulário fica", async () => { /* … */ });
it("abrir com outra aberta: 409 campaign_already_open relê uma vez e não reenvia", async () => { /* openCampaign 1x; listCampaigns 2x */ });
it("anular: motivo 10–500; estorna e marca a dose", async () => { /* cancelVaccination("v1", "dose registrada no animal errado") */ });
it("meta sugerida pela estimativa e pela cobertura; sem estimativa, nada", async () => { /* … */ });
it("resultado semanal só no clique; as duas coberturas; CSV com o nome do api", async () => { /* getZoonosesReport 0x antes; 1x depois; as colunas das duas coberturas aparecem com uma casa (Q19) */ });
it("admin lê campanhas e resultado, sem botões de escrita", async () => { /* … */ });
it("falta de saldo não recusa: a dose fica e a frase da conciliação aparece", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooCampaigns.tsx src/modules/vectorControl/ReportTable.tsx src/modules/zoonoses/CampaignForm.tsx src/modules/zoonoses/VaccinationForm.tsx src/modules/zoonoses/WeeklyResult.tsx src/modules/ZooCampaigns.test.tsx
$G commit -m "feat: run rabies campaigns, record fixed post and routine doses and show the weekly result" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Demandas — fila do centro e andamento (F-35.13)

**Files:**
- Create: `src/modules/zoonoses/DemandDetail.tsx`, `src/modules/ZooDemands.test.tsx`
- Modify: `src/modules/ZooDemands.tsx`

**Interfaces:**
- Consumes: `listDemands`, `getDemand`, `scheduleDemand`, `inspectDemand`, `closeDemand`, `declineDemand`, `extendDemand`, `getZoonosesDomains` (Task 1); `demandActions`, `demandProblems`, `zoonosesRefusals`, `isZoonosesStale` (Task 2); `DeadlineTag` (33).
- Produces:
  - `ZooDemands({ today? })`: `ZoonosesGate allow={(r) => r.team}`; abas "Abertas" (`open`, `scheduled`, `inspected`) | "Encerradas"; `DataTable`: subtipo (rótulo do §4), prazo e cor (`DeadlineTag`) e estado (Q13); **sem extrato, endereço, identidade, contato, protocolo ou anexo**; "Carregar mais" pelo cursor.
  - `DemandDetail({ id, onChanged })`: `getDemand` com `NOMINAL_QUERY` (trilha `zoonoses_demand.viewed`); subtipo, extrato (só o texto do cidadão, Q13), "Endereço do local" (`place_address`), prazo, estado, vistoria; uma ação aberta por vez, só as de `demandActions`: "Agendar vistoria" (data), "Registrar vistoria" (data e nota 10–2000), "Encerrar" (desfecho dos domínios e texto 20–4000, com a linha `"este texto vai à ouvidoria como a resposta do encaminhamento"`), "Não é do centro" (texto 10–4000), "Prorrogar prazo" (motivo 10–4000, uma vez); `expected_version`; `409 demand_changed`/`forwarding_changed` → frase e uma releitura do detalhe e da fila.

- [ ] **Step 1: Testes que falham** (`ZooDemands.test.tsx`):

```tsx
it("fila sem extrato nem endereço; detalhe lê uma vez e mostra os dois", async () => {
  // a linha não contém demandDetail().extract nem demandDetail().place_address; getDemand 1x ao abrir
});
it("agendar → vistoriar → encerrar com desfecho e texto 20–4000; cada um com expected_version", async () => {
  // closeDemand("d1", { outcome: "resolved", text: "<20+ caracteres>", expected_version: 3 })
});
it("encerramento com 19 caracteres não envia", async () => { /* … */ });
it("recusa: closed sem desfecho; prorrogar some depois de usado", async () => { /* … */ });
it("409 forwarding_changed: frase do 33 e uma releitura, sem reenviar", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooDemands.tsx src/modules/zoonoses/DemandDetail.tsx src/modules/ZooDemands.test.tsx
$G commit -m "feat: work the zoonoses demand queue from scheduling to the forwarding reply" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Observação do agressor — fila, contato, desfecho e animal (F-35.14)

**Files:**
- Create: `src/modules/zoonoses/ObservationDetail.tsx`, `src/modules/zoonoses/ContactDialog.tsx`, `src/modules/ZooObservations.test.tsx`
- Modify: `src/modules/ZooObservations.tsx`

**Interfaces:**
- Consumes: `listObservations`, `getObservation`, `getObservationContact`, `closeObservation`, `linkObservationAnimal` (Task 1); `closeObservationProblems`, `observationDaysLeft`, `observationOutcomeLabel`, `zoonosesRefusals`, `isZoonosesStale` (Task 2); `AnimalSearch`, `AnimalForm` (Task 5); `useDialog`.
- Produces:
  - `ZooObservations({ today? })`: `ZoonosesGate allow={(r) => r.team}`; abas "Em observação" | "Encerradas"; `DataTable`: espécie, condição, exposição, fim (`ends_on`), "faltam N dias" / "último dia" / "terminou há N dias" (`observationDaysLeft`), local (casa ou canil); **nunca** tutor, paciente, idade ou unidade (decisão 32).
  - `ObservationDetail({ id, today, onChanged })`: `getObservation` com `NOMINAL_QUERY`; o bloco do animal; "Ver contato do tutor" abre o `ContactDialog`; "Ligar animal" (`AnimalSearch` ou cadastro sem tutor) → `linkObservationAnimal`; "Registrar desfecho": desfecho dos domínios, data e nota opcional, `closeObservationProblems` antes; `euthanized` só aparece quando o animal ligado tem eutanásia (senão a linha `"registre antes a eutanásia em Canil e gatil"`); `expected_version`; `409 observation_changed` → frase e uma releitura.
  - `ContactDialog({ observationId, onClose })` (molde do `CaseAddressDialog`, sem step-up a mais, Q12): `getObservationContact` **direto**, no clique; nome, telefone e endereço só no estado do diálogo; `"A leitura fica registrada na trilha. Use o contato só para acompanhar o animal."`; Escape e Fechar limpam; nada no `queryClient`.

- [ ] **Step 1: Testes que falham** (`ZooObservations.test.tsx`):

```tsx
it("fila: bloco do animal e dias que faltam; nenhum dado do tutor ou do paciente", async () => { /* "faltam 2 dias" em TODAY35B */ });
it("contato: só no clique, só no diálogo, nada no cache; fechar limpa", async () => {
  // getObservationContact 0x antes; 1x no clique; nenhuma query com CONTACT.tutor_phone; depois de Escape o telefone some
});
it("sadio antes do último dia: observation_not_finished antes de enviar; no último dia envia", async () => {
  // closeObservation("ob1", { outcome: "healthy", outcome_on: "2026-11-25", expected_version: 1 }) com today 2026-11-25
});
it("eutanásia como desfecho só com a eutanásia do animal registrada", async () => { /* … */ });
it("ligar animal: expected_version; 409 observation_changed relê uma vez", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooObservations.tsx src/modules/zoonoses/ObservationDetail.tsx src/modules/zoonoses/ContactDialog.tsx src/modules/ZooObservations.test.tsx
$G commit -m "feat: follow the biting animal observation with an audited contact read and the outcome" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: A notificação do 25 — bloco do animal, condição final e sinal na fila (F-35.14)

**Files:**
- Create: `src/modules/surveillance/NotificationDetail.zoonoses.test.tsx`
- Modify: `src/lib/surveillanceForm.ts`, `src/lib/surveillanceForm.test.ts`, `src/modules/surveillance/NotificationFormSection.tsx`, `src/modules/surveillance/NotificationDetail.tsx`, `src/modules/surveillance/NotificationTable.tsx`

**Interfaces:**
- Consumes: `SurveillanceNotification.animal_observation`, `animal_outcome_pending` (Task 1); `observationStateLabel`, `observationOutcomeLabel` (Task 2).
- Produces:
  - `src/lib/surveillanceForm.ts`: `FormFieldKind` + `"date" | "bool"` (Q16: valores em texto, `"YYYY-MM-DD"` e `"true"`/`"false"`); `RABIES_DISEASE_CODE = "acidente_animal_raiva"`; `RABIES_ANIMAL_FIELDS: FormFieldSpec[]` (rótulos propostos; conferir com o yml do api na Task 0):

| Chave | Rótulo | Tipo |
|---|---|---|
| `animal_species` | Espécie do animal | `code`: `dog` Cão, `cat` Gato, `bat` Morcego, `primate` Primata, `fox` Raposa, `livestock` Animal de produção, `other` Outro |
| `animal_condition` | Condição do animal | `code`: `healthy` Sadio, `suspect` Suspeito, `rabid` Raivoso, `dead_or_missing` Morto ou desaparecido |
| `animal_observable` | Animal passível de observação | `bool` |
| `exposure_on` | Data da exposição | `date` |
| `animal_tutor_name` | Nome do tutor do animal | `text`, 1–120 |
| `animal_tutor_phone` | Telefone do tutor | `digits`, 10–11 |
| `animal_address` | Endereço do animal | `text`, 10–300 (`min: 10`) |

  - `formFieldsFor(diseaseCode: string): FormFieldSpec[]` → `SINAN_FORM_FIELDS` mais `RABIES_ANIMAL_FIELDS` só em `acidente_animal_raiva`; `formProblem` confere `date` (data válida, ≤ hoje) e `bool`, o mínimo de `text`, e `animal_observable` `"true"` com espécie fora de `dog`/`cat` → `form.animal_observable`.
  - `NotificationFormSection` + prop `fields?: FormFieldSpec[]` (padrão `SINAN_FORM_FIELDS`): os campos do animal num `fieldset` "Animal agressor"; `bool` como caixa de marcar; `date` como `<input type="date">`; **um** PATCH com todos os campos da lista.
  - `NotificationDetail`: `defaultFormSlot` passa `fields={formFieldsFor(n.disease.code)}` (a ficha continua sendo o `formSlot`, decisão 32); com `n.animal_observation`, o bloco **"Condição final do animal após a observação"**: situação, fim da observação, desfecho e data (sem desfecho: `"observação em curso até <data>"`).
  - `NotificationTable`: `animal_outcome_pending` → `Tag tone="warn"` "desfecho do animal".

- [ ] **Step 1: Testes que falham**
  - `surveillanceForm.test.ts`: `formFieldsFor("dengue")` = `SINAN_FORM_FIELDS`; `formFieldsFor("acidente_animal_raiva")` termina com as 7 chaves; `formProblem` recusa `exposure_on` "2026-02-30" e amanhã, `animal_observable` "true" com `animal_species` "bat" em `form.animal_observable`, `animal_address` com 9 caracteres.
  - `NotificationDetail.zoonoses.test.tsx`:

```tsx
it("ficha: uma seção, um PATCH; animal só em acidente_animal_raiva", async () => {
  // dengue: nenhum campo "Espécie do animal"; acidente: salva → updateNotificationForm 1x com as chaves do Sinan E as do animal
  // ("animal_observable": "true", "exposure_on": "2026-11-15")
});
it("condição final do animal: título fixo, desfecho e data; em curso mostra o fim", async () => {
  // screen.getByText("Condição final do animal após a observação")
});
it("fila: animal_outcome_pending vira o sinal da linha", async () => { /* NotificationTable */ });
```

- [ ] **Step 2–4:** rode e veja falhar (os arquivos acima, `src/modules/surveillance/NotificationDetail.test.tsx` e `src/modules/surveillance/NotificationFormSection.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/surveillanceForm.ts src/lib/surveillanceForm.test.ts src/modules/surveillance/NotificationFormSection.tsx src/modules/surveillance/NotificationDetail.tsx src/modules/surveillance/NotificationTable.tsx src/modules/surveillance/NotificationDetail.zoonoses.test.tsx
$G commit -m "feat: add the biting animal block to the rabies notification form and show the observation outcome" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Leishmaniose — inquéritos, coleta, resultados, desfecho e coleiras (F-35.15)

**Files:**
- Create: `src/modules/zoonoses/SurveyForm.tsx`, `src/modules/zoonoses/TestsPanel.tsx`, `src/modules/zoonoses/CollarsPanel.tsx`, `src/modules/ZooLeish.test.tsx`
- Modify: `src/modules/ZooLeish.tsx`

**Interfaces:**
- Consumes: inquéritos, exames, desfecho, coleiras, `getZoonosesDomains`, `listCityBlocks`, `listPanelNeighborhoods` (Task 1); `elisaAllowed`, `isReactiveDog`, `leishOutcomeProblems`, `zoonosesRefusals`, `isZoonosesStale` (Task 2); `AnimalSearch` (Task 5).
- Produces:
  - `ZooLeish({ today? })`: `ZoonosesGate allow={(r) => r.team}`; abas "Inquéritos" | "Exames" | "Desfecho pendente" | "Coleiras a trocar".
  - `SurveyForm` (coordenação): nome 1–80, tipo (domínios), bairro, quarteirões do bairro (`listCityBlocks`, marcar; não vazio), população de cães (inteiro > 0, com a sugestão `dog_population_suggestion` que o api devolve: os cães ativos dos quarteirões ou, com zero, quarteirões × `dogs_per_block`, Q18), período; no `focus`, um quarteirão central e a sugestão de anéis do `VectorControl::Rings` que o api devolve (Q18; a forma — campo do rascunho ou rota dos quarteirões — a Task 0 confere; sem ela, a tela marca os `neighbor_ids` do central), e o coordenador ajusta; o tamanho é o `target_dogs` **que o api devolve** (`"cães a examinar: N"`); a tela não calcula; `draft` → `open` → `closed` com `ConfirmButton` e `expected_version`.
  - `TestsPanel`: "Nova coleta" (inquérito aberto ou avulsa; cão ativo por `AnimalSearch`; DPP opcional) → a etiqueta que o api devolve, em destaque (`"Etiqueta: LV-2026-…"`); `422 animal_not_dog`/`animal_not_active`/`survey_not_open` no campo; por exame: DPP e data; ELISA, data e requisição do GAL só com `elisaAllowed` (senão a linha de `elisa_without_reactive_dpp`); correção com `expected_version`, `409 test_changed` relê uma vez.
  - "Desfecho pendente" (`listLeishTests({ pending: "outcome" })`): os cães reagentes; decisão (domínios) com as regras de `leishOutcomeProblems`; `euthanasia` manda o `euthanasia_id` da eutanásia do animal (lida do histórico, Q18) e, sem ela, leva a Canil e gatil.
  - `CollarsPanel` ("Coleiras a trocar", `listCollars({ due: true })`): animal, colocada, vence; "Colocar coleira" (cão; produto `collar`; a nova troca a anterior — a linha `"a coleira ativa sai como trocada"`); "Retirar" (data e motivo, `expected_version`); `stock_shortfall` → a frase da Task 6.
  - Prevalência por inquérito: vem do relatório `leish` (Task 12); esta tela não calcula.

- [ ] **Step 1: Testes que falham** (`ZooLeish.test.tsx`): inquérito por amostra mostra o `target_dogs` e a `dog_population_suggestion` do api depois de salvar o rascunho; `focus` marca os quarteirões sugeridos; desfecho `euthanasia` leva o `euthanasia_id`; coleta devolve e mostra a etiqueta; ELISA sem DPP reagente não aparece e o 422 `elisa_without_reactive_dpp` cai na linha; "Desfecho pendente" lista só o reagente; `refused` sem tutor recusa antes; coleira nova avisa a troca; equipe não vê "Novo inquérito".
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooLeish.tsx src/modules/zoonoses/SurveyForm.tsx src/modules/zoonoses/TestsPanel.tsx src/modules/zoonoses/CollarsPanel.tsx src/modules/ZooLeish.test.tsx
$G commit -m "feat: run canine leishmaniasis surveys, serology results, reactive dog outcomes and collar replacement" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Canil e gatil — estadas, castração e eutanásia (F-35.16)

**Files:**
- Create: `src/modules/zoonoses/StayForm.tsx`, `src/modules/zoonoses/EuthanasiaForm.tsx`, `src/modules/ZooShelter.test.tsx`
- Modify: `src/modules/ZooShelter.tsx`

**Interfaces:**
- Consumes: estadas, `recordNeutering`, `recordEuthanasia`, `getAnimal`, `getMyProfessional`, `getZoonosesDomains` (Task 1); `isVeterinarian`, `vetIdentity`, `stayExitProblems`, `euthanasiaProblems`, `ZOO_NOT_VET`, `zoonosesRefusals`, `isZoonosesStale`, `ZOO_ME_KEY` (Task 2); `AnimalSearch`, `PersonLookup` (Task 5); `SensitiveAction`.
- Produces:
  - `ZooShelter({ today?, onGoToSecurity? })`: `ZoonosesGate allow={(r) => r.team}`; unidade (`domains.shelter_units`); abas "No canil" (estadas abertas, contagem por unidade sem limite) | "Saídas".
  - `StayForm` (entrada): animal cadastrado ou "animal sem cadastro" (espécie, sexo, porte, pelagem, raça; criado sem tutor e sem domicílio); tipo de entrada; `observation` exige escolher uma observação aberta, que passa a `place` `shelter` (Q16); `409 stay_already_open` → frase e uma releitura.
  - Saída: tipo, data; "Adotado" pede o adotante (`PersonLookup`, que vira o tutor); `stayExitProblems` antes; `expected_version`; `409 stay_changed` relê uma vez.
  - Veterinário: `getMyProfessional` (`ZOO_ME_KEY`, `STATIC_QUERY`) → com `isVeterinarian(me, roles)` (Q12: o papel da equipe e o perfil), "Registrar castração" (data; `neutered` não volta) e "Registrar eutanásia"; sem ela, a linha `ZOO_NOT_VET` e nenhum botão.
  - `EuthanasiaForm({ animal, me, onDone })`: mostra **o veterinário logado** (`vetIdentity(me)`), motivo, nota (10–1000) e base legal (domínios, ao menos uma), `euthanasiaProblems` antes; "Registrar eutanásia" por `SensitiveAction` com `requiresStepUp` e `description` "A eutanásia não se edita nem se apaga. O animal passa a morto na data informada."; um envio; `403 veterinarian_required`, `409 already_euthanized` → frase e uma releitura do histórico; depois do 201, o histórico mostra a eutanásia **só para leitura** e o formulário some.

- [ ] **Step 1: Testes que falham** (`ZooShelter.test.tsx`):

```tsx
it("eutanásia: step-up, um envio, recusas do não veterinário e da duplicidade, só leitura depois", async () => {
  // mostra "Ana Veterinária · CRMV-PR 12345"; SensitiveAction → recordEuthanasia 1x com legal_basis ["cfmv_1000_2012"]
  // duplo clique → 1 chamada; 403 veterinarian_required → ZOONOSES_ERRORS.veterinarian_required, getAnimal relido 1x
  // depois do 201: nenhum botão "Registrar eutanásia" e o resumo da eutanásia visível
});
it("usuário da equipe sem perfil de veterinário: sem castração nem eutanásia e a frase", async () => { /* getMyProfessional → null */ });
it("entrada sem cadastro cria o animal sem tutor; segunda entrada → 409 stay_already_open relê uma vez", async () => { /* … */ });
it("saída adotado exige o adotante; devolvido sem tutor recusa antes", async () => { /* … */ });
it("castração marca castrado e não oferece desfazer", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooShelter.tsx src/modules/zoonoses/StayForm.tsx src/modules/zoonoses/EuthanasiaForm.tsx src/modules/ZooShelter.test.tsx
$G commit -m "feat: manage shelter stays, neutering and step-up euthanasia by the logged veterinarian" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Relatórios de zoonoses (F-35.17)

**Files:**
- Create: `src/modules/ZooReports.test.tsx`
- Modify: `src/modules/ZooReports.tsx`

**Interfaces:**
- Consumes: `getZoonosesReport`, `fetchZoonosesReportFile`, `listCampaigns`, `listLeishSurveys` (Task 1); `zoonosesError`, `zoonosesRefusals`, `ZOO_REPORT_KEY` (Task 2); `ReportTable`, `saveFile`.
- Produces:
  - `ZOO_REPORT_TITLES`: `rabies_weekly` "Vacinação antirrábica por semana", `rabies_modalities` "Doses por modalidade e espécie", `demands` "Demandas por subtipo", `observations` "Observações por desfecho", `leish` "Leishmaniose por inquérito", `shelter` "Canil e gatil".
  - `ZooReports({ today? })`: `ZoonosesGate allow={(r) => r.reader}`; escolha do relatório; filtros (Q25): `rabies_weekly` a campanha; os outros o período (`from`, `to`, até 366 dias, padrão os últimos 30) e, no `leish`, o inquérito opcional; "Ver" (`STATIC_QUERY`, só no clique); "Baixar CSV"; `422 invalid_request` com `path` no filtro.
  - A supressão é do api: `{ suppressed: true }` aparece "< 5" pelo `ReportTable`; a tela não soma linhas.

- [ ] **Step 1: Testes que falham** (`ZooReports.test.tsx`): os seis títulos para a equipe e para o `municipal_admin`; `demands` com período chama `getZoonosesReport("demands", { from: "2026-10-25", to: "2026-11-23" })`; contagem suprimida aparece "< 5"; CSV com o nome do api; nada lido antes do "Ver"; período de 367 dias recusa antes.
- [ ] **Step 2–4:** rode e veja falhar (`src/modules/ZooReports.test.tsx src/modules/VectorReports.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooReports.tsx src/modules/ZooReports.test.tsx
$G commit -m "feat: show the six zoonoses reports with suppressed small counts and CSV download" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Parâmetros de zoonoses (F-35.14, F-35.15)

**Files:**
- Create: `src/modules/ZooSettings.test.tsx`
- Modify: `src/modules/ZooSettings.tsx`

**Interfaces:**
- Consumes: `getZooParameters`, `putZooParameters` (Task 1); `parameterProblems`, `sampleTableProblems`, `zoonosesRefusals`, `isZoonosesStale`, `ZOO_PARAMS_KEY`, `ZOO_DOMAINS_KEY` (Task 2); `SensitiveAction`.
- Produces:
  - `ZooSettings({ onGoToSecurity? })`: `ZoonosesGate allow={(r) => r.coordinator}`; os 8 campos do §9 sem `leish_expected_prevalence` (Q18) agrupados (observação; coleira; leishmaniose — a tabela da amostra do manual (2%) com linhas "população até" e "amostra", mínimo do foco, cães por quarteirão; raiva — cobertura alvo, estimativas de cães e de gatos, vazias = nulo); `"versão N, de DD/MM/AAAA"` (versão 0 com `created_at: null` → `"padrão de fábrica (versão 0)"`); decimais com vírgula; "Salvar parâmetros" por `SensitiveAction` (`requiresStepUp`, Q12: segue o PUT do 35a; `description` "Cria a versão N+1. O inquérito já aberto continua com a versão dele.") → `putZooParameters(values, version)`; `422 invalid_parameter` com `path` `values.<chave>` no campo; `409 parameters_changed` → frase, relê, mantém o digitado; depois de gravar, invalida também `ZOO_DOMAINS_KEY`.

- [ ] **Step 1: Testes que falham** (`ZooSettings.test.tsx`): `observation_days` 9 e 31 recusados antes, 10 e 30 valem; tabela com população decrescente recusada na linha; step-up e corpo inteiro com `expected_version`; `invalid_parameter` no campo; `409 parameters_changed` relê uma vez e mantém o digitado; equipe vê `ZOO_NO_ROLE`.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ZooSettings.tsx src/modules/ZooSettings.test.tsx
$G commit -m "feat: edit versioned zoonoses parameters with step-up" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Ouvidoria — assunto reservado com cadeado e as chaves da demanda no balcão (F-35.13)

**Files:**
- Create: `src/modules/ombudsman/SubjectsPanel.zoonoses.test.tsx`
- Modify: `src/lib/ombudsman.ts`, `src/modules/ombudsman/SubjectsPanel.tsx`, `src/modules/ombudsman/NewManifestationForm.tsx` (Q14)

**Interfaces:**
- Consumes: `OmbudsmanSubject.code` (Task 1, Q14); `zoonosesUsable`, `ZOONOSES_SUBTYPES`, `ZOONOSES_ERRORS.reserved_subject`, `hasFeature` (Task 2).
- Produces:
  - `OMBUDSMAN_ERRORS.reserved_subject = ZOONOSES_ERRORS.reserved_subject`.
  - `SubjectRow`: assunto com `code` mostra `IconLock` com `aria-label="assunto reservado do centro de zoonoses"`; com `zoonoses` na sessão, o botão "Desativar" não aparece nele (renomear continua); `409 reserved_subject` (corrida) → a frase.
  - `NewManifestationForm` (Q14): só nos tipos Solicitação e Denúncia, com `zoonosesUsable(user)` e a ouvidoria (Q10), "É sobre animais, insetos ou bichos?" (Sim/Não); com Sim, o subtipo (`ZOONOSES_SUBTYPES`) e "Onde fica o problema? (endereço do local)" obrigatório (10–300); o corpo leva `zoonoses_subtype` e `place_address` (nos três modos de identificação); sem o portão ou noutro tipo, nada muda.

- [ ] **Step 1: Testes que falham** (`SubjectsPanel.zoonoses.test.tsx`): o pai "Zoonoses" e o filho "Escorpião" com `code` mostram o cadeado e não têm "Desativar" com `zoonoses` ligado; sem `zoonoses`, "Desativar" volta; assunto sem `code` não muda; `409 reserved_subject` mostra a frase. `NewManifestationForm` (Q14): com Sim, sem endereço não envia; com endereço, `createOmbudsmanManifestation` leva `zoonoses_subtype: "scorpion"` e `place_address`; Elogio, Reclamação e Sugestão não mostram a pergunta.
- [ ] **Step 2–4:** rode e veja falhar (o arquivo acima e `src/modules/ombudsman/NewManifestation.test.tsx src/modules/OmbudsmanSettings.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/ombudsman.ts src/modules/ombudsman/SubjectsPanel.tsx src/modules/ombudsman/SubjectsPanel.zoonoses.test.tsx src/modules/ombudsman/NewManifestationForm.tsx
$G commit -m "feat: lock reserved zoonoses subjects and offer the zoonoses demand fields at the ombudsman desk" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Suíte, build, varreduras, revisão e prova no navegador (api na 3061, Vite na 5221)

**Files:** `$R/.claude/launch.json` (entrada nova; fora de git). Correção que a prova pedir entra na task dona, com teste.

- [ ] **Step 1: Suíte, tipos e build**

```bash
$R/bin/test dashboard -w dashboard-mod35b
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **16 arquivos de teste novos**: `src/lib/api.zoonoses.test.ts`, `src/lib/zoonosesErrors.test.ts`, `src/lib/zoonoses.test.ts`, `src/shell/modules.zoonoses.test.ts`, `src/modules/zoonoses/ZoonosesGate.test.tsx`, `src/modules/VectorSettings.zoonoses.test.tsx`, `src/modules/ZooAnimals.test.tsx`, `src/modules/ZooCampaigns.test.tsx`, `src/modules/ZooDemands.test.tsx`, `src/modules/ZooObservations.test.tsx`, `src/modules/surveillance/NotificationDetail.zoonoses.test.tsx`, `src/modules/ZooLeish.test.tsx`, `src/modules/ZooShelter.test.tsx`, `src/modules/ZooReports.test.tsx`, `src/modules/ZooSettings.test.tsx`, `src/modules/ombudsman/SubjectsPanel.zoonoses.test.tsx` (os do 25, 27a, 33 e 35a ganham casos, não arquivo). Contagem que dobra = artefato de build achado pelo vitest; erro não tratado com tudo verde = mock sem `fetchCurrentSession`.

- [ ] **Step 2: Varreduras**

```bash
cd $W
O="src/modules/zoonoses src/modules/Zoo*.tsx src/lib/zoonoses.ts src/lib/zoonosesErrors.ts"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
grep -rnE "useQuery\(.*(getObservationContact|searchAnimals|lookupZooPerson)|setQueryData\(.*(contact|microchip)" $O && echo "ERRO: dado sensível no cache" || echo ok-cache
sed -n '/Centro de zoonoses (módulo 35b/,$p' src/lib/api.ts | grep -nE "microchip|cpf|cns|reason|note|text|place_address" | grep -E "withQuery|hvQuery|\?\$\{" && echo "revisar: URL" || echo ok-url
for f in $(grep -rlE 'vi\.mock\("\.\./(\.\./)?lib/api", async' src/modules/Zoo*.test.tsx src/modules/zoonoses src/modules/surveillance/*.zoonoses.test.tsx src/modules/ombudsman/*.zoonoses.test.tsx src/modules/VectorSettings.zoonoses.test.tsx 2>/dev/null); do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-cache`, `ok-url`, `ok-mocks`, sem `ERRO`.

- [ ] **Step 3: Conferência do api** — repita a Task 0 Step 4 contra o api final; forma diferente vira ajuste na task dona antes da prova.

- [ ] **Step 4: Revisão do ramo** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, na tabela de frases, no microchip e no contato, na ficha do 25 e nas telas do 35a abertas com `zoonoses`.

- [ ] **Step 5: Entrada do Vite** — acrescente ao `$R/.claude/launch.json`, depois da `dashboard-mod35a`:

```json
{
  "name": "dashboard-mod35b",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/dashboard-mod35b && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3061 npx vite --port 5221 --host 0.0.0.0" ],
  "port": 5221
}
```

- [ ] **Step 6: Prova no navegador** — **só com autorização do usuário** para migrar os bancos de dev das cidades (migrações a partir de `20261123900001`) e rodar a semente (`lib/zoonoses_crew.rb`, só Curitiba). Peça antes de qualquer `db:migrate` ou `db:seed`. A semente liga `zoonoses`: combine pela sessão "Próximos módulos" que nenhum aparelho com app anterior ao 35a está pareado em Curitiba (gate do §11). O api do worktree `apps/api/.claude/mod35b` na **3061** é da sessão do api; o maintenance na 5177 e o wpda na 5222 são das sessões deles. Abra `curitiba.localhost:5221/dashboard/` (`preview_start` com `dashboard-mod35b`). Logins da semente (§17; senha e TOTP de dev podem ser mostrados, nunca os de staging). Roteiro do §17, um passo por linha:

1. **Interruptor** (maintenance 5177): desligado, o grupo "Zoonoses" some do menu de `zoonoses@curitiba.demo` depois de recarregar; religado, volta.
2. **Animal** (`zoonoses@curitiba.demo`): busca pelo microchip de um animal da semente; cadastro com tutor (CPF de um paciente da semente); histórico. Confira na aba de rede que o microchip só vai no corpo.
3. **Campanha** ("Campanha antirrábica 2026"): dose no posto fixo sem cadastro (gato), dose de rotina num animal cadastrado (sem campanha), anulação com estorno (em "Posição e extrato" do local da UVZ, o movimento "estorno de vacinação antirrábica (anulada)"), "Ver resultado semanal" com as duas coberturas (Q19) e o CSV.
4. **Demandas**: a solicitação de escorpião com endereço e a denúncia anônima de morcego feitas no wpda 5222 aparecem na fila só com subtipo, prazo, cor e estado; no detalhe, o extrato e o endereço (Q13); agendar, vistoriar e encerrar com texto; a resposta aparece no encaminhamento em "Caixa da ouvidoria".
5. **Notificação** (usuário de vigilância da semente): a notificação de `acidente_animal_raiva` mostra a seção do animal; em "Observação do agressor", a observação aberta; "Ver contato do tutor" (trilha); desfecho "sadio" com data no último dia; na notificação, "Condição final do animal após a observação" e, na fila do 25, o sinal.
6. **Leishmaniose** (`endemias@curitiba.demo`): inquérito por amostra com o tamanho do api; coleta com etiqueta; DPP; ELISA; o reagente em "Desfecho pendente"; coleira colocada e a lista "Coleiras a trocar".
7. **Canil** (`veterinaria@curitiba.demo`, com `zoonoses_team`, Q12): entrada, castração, eutanásia com step-up e a tela só lendo depois; com `zoonoses@curitiba.demo`, nenhum botão de eutanásia e a frase do veterinário.
8. **Relatórios**: os seis em tela e em CSV; "< 5" em demandas e em observações; o `municipal_admin` só vê "Campanhas e vacinação" e "Relatórios".
9. **Campo** é da prova do emulador, não desta.

Também: só com `zoonoses` (desligue `vector_control` no maintenance, se a semente o liga), a coordenação cadastra a vacina em "Produtos e parâmetros" e escolhe o local; o assunto "Zoonoses" com cadeado em "Configuração da ouvidoria"; preset mobile 375×812 nos passos 2, 3 e 7; só teclado nos passos 5 e 7; `resize_window` preset desktop ao terminar. Anote cada passo; achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai ao usuário (não vira tarefa, memória "Interface em ciclo próprio").

---

### Task 16: Entrega — board, relato e parada antes do merge

**Files:** nenhum.

- [ ] **Step 1: Board** — F-35.12, F-35.14, F-35.15, F-35.16 e F-35.17 → **Done** se o api já terminou o deles; F-35.11 espera o maintenance e F-35.13 o wpda (o último app move). **Verified** só depois da prova da Task 15, com o usuário.
- [ ] **Step 2: Suíte completa do api** — não é desta sessão: o pedido vai à sessão "Testes" pela sessão dona do api (`api · mod35b/feat/mod-35b-zoonoses · --full · mudança transversal do 35b`). A suíte inteira do dashboard neste worktree (Task 15 Step 1) basta para o dashboard.
- [ ] **Step 3: Pare e peça autorização** para merge e push (ordem do §18: depois do api). Antes do push, `$G -C $W log --oneline origin/main..HEAD` e `origin/main..main` (sessões paralelas). Worktree: `$G -C $R/apps/dashboard worktree remove $W` só depois do merge.

**Relato:** `dashboard · dashboard-mod35b/feat/mod-35b-zoonoses · Task 16 · suíte <n>/<n>, build ok, varreduras ok, prova <passos> · cards movidos · aguardando autorização de merge`.

---

## Self-review

1. **Cobertura do §13 e de F-35.11–F-35.17:** grupo e portão (Task 3); `zoo-animals` (5, F-35.11); `zoo-campaigns` com posto fixo, rotina, anulação e resultado semanal (6, F-35.12); `zoo-demands` (7, F-35.13); `zoo-observations` (8, F-35.14); ficha, condição final e sinal do 25 (9, F-35.14); `zoo-leish` com "Desfecho pendente" e "Coleiras a trocar" (10, F-35.15); `zoo-shelter` com a eutanásia do §13 (11, F-35.16); `zoo-reports` (12, F-35.17); `zoo-settings` (13); produtos do 35a com as duas finalidades (4); cadeado dos assuntos (14); `isZoonosesStale` (2, em todas as telas); roteiro do §17 (15). O passo 9 do roteiro (campo) e o maintenance (§15) não são deste repo.
2. **Bordas do §18 que tocam a tela:** 10º dia exato (Tasks 2 e 8), segunda dose na campanha (6), ELISA sem DPP (2 e 10), coleira trocada (10), eutanásia por quem não é veterinário e em duplicidade (11). A população acima da última linha da tabela é do api: a tela mostra o `target_dogs` dele (10).
3. **Tipos:** `Animal`, `AnimalHistory`, `ZoonosesDomains`, `ZooPerson` (Task 1) são os mesmos nas Tasks 5–11; `AnimalSearch`/`PersonLookup` (Task 5) servem às Tasks 6, 8, 10 e 11; `ReportTable` genérica (6) serve à 12; `canUseZoonoses`/`zoonosesRoles`/`isVeterinarian` (2) governam as Tasks 3, 4 e 11.
4. **Review Focus:** as cinco linhas têm teste na task dona (11; 5, 7 e 8; 9; 2 e 8; 6).
5. **Proporção:** o código do plano é tipo, frase fixa e teste; as telas vão por assinatura, props e comportamento.

## Divergências propostas ao contrato

> **Todas resolvidas** no §19 do contrato (Q10–Q27, 2026-11-23); a Q de cada uma está no início da linha e, onde a decisão difere da proposta, a diferença vem logo depois. O texto original fica como registro; vale a decisão, e o plano já a aplica (as tasks citam a Q). Onde o api da execução fixar outra forma, vale a do api (Q25; Task 0 Step 4).

Cada uma diz o que a tela precisa, a evidência e a proposta; se o api adotar outra forma, vale a do api (Task 0 Step 4) e a task dona ajusta tipo, fixture e teste.

- **Resolvida pela Q25** — os tipos da Task 1 valem como proposta; o api da execução fixa as formas e a Task 0 confere. **D-1 — Formas das respostas e dos corpos não fixadas.** O §10 lista as rotas e o §1 as colunas, mas não as formas de leitura nem os corpos (a D-1 do 35a, no mesmo ponto). **Proposta:** os tipos e corpos da Task 1: tutor só `{ id, display_name }`; **nenhuma resposta traz o número do microchip** (`has_microchip`, §16) e o microchip só na criação (como o app, §11); listas `{ items, next_before }`; transições com `{ expected_version }`; encerramento da demanda `{ outcome, text }`, recusa `{ text }`, prorrogação `{ reason }`; relatórios na forma da D-13 do 35a (`{ name, columns, rows, notes? }`, `SmallCount` nas células suprimidas); a vacinação e a coleira com `stock_shortfall: boolean`; filtros de `leish_tests` como `leish_survey_id` e `pending=elisa|outcome`. **`GET /zoonoses/domains`** estende o `zoonoses_domains` do field-sync com as listas `{ code, label }` que as telas rotulam (as do `ZoonosesDomains`), `units` (unidades ativas, para o `health_unit_id` da vacinação) e `shelter_units` (as `environmental_surveillance` ativas).
- **Resolvida pela Q10** — aceita: `canUseZoonoses` exige `canUseTerritory`; o `/zoonoses/domains` responde 403 `feature_disabled` com o gate desligado. **D-2 — Portão do dashboard sem o cadastro territorial.** O §13 define `canUseZoonoses` como interruptor e papel, mas o `Zoonoses::Gate.usable?` exige o `TerritoryRegistry::Gate.usable?` (§1), e a sessão traz as chaves **ligadas**, não as utilizáveis (`src/lib/features.ts:1-4`). O `canUseTerritory` do dashboard é `territory_registry` e `clinical_record` (`src/lib/territoryRegistry.ts:50-52`). Sem isso, o menu aparece e toda rota responde `403 feature_disabled`. **Proposta:** `canUseZoonoses` = não operador, `zoonoses`, `canUseTerritory` e um papel de leitura (como o 35a com a visita, Q1 do 35a).
- **Resolvida pela Q12** — aceita: a veterinária precisa do papel `zoonoses_team`; `veterinarian?` = `team?` e o perfil 223305 com CRMV e vínculo na UVZ; a tela a reconhece por `/professionals/me`. O `zoonoses_team` é privilegiado. **D-3 — O veterinário sem papel não vê o menu.** O §1 diz que o veterinário faz tudo da equipe "sem precisar do papel", mas a sessão não traz CBO nem perfil (`src/lib/api.ts:44-56`) e o §13 exige um papel; a semente (§17) cria a veterinária sem papel. **Proposta:** a semente dá `zoonoses_team` à `veterinaria@curitiba.demo`, e o contrato registra que, no dashboard, o veterinário usa um papel do 35b para ver o grupo; a tela reconhece o veterinário por `GET /professionals/me` (CBO, CRMV e vínculo ativo; `src/lib/api.ts:816`) e o api confere a unidade (`403 veterinarian_required`). Alternativa pior: o menu para todo `health_professional` com uma leitura extra no shell.
- **Resolvida pela Q25** — aceita: `POST /zoonoses/person_lookup` (CPF ou CNS no corpo, com trilha), para tutor e adotante. **D-4 — Busca do tutor e do adotante.** O §2 manda usar "a busca que o dashboard já tem (CPF ou CNS no corpo, com trilha)", mas cada módulo tem a sua `person_lookup` com o próprio portão (`apps/api config/routes.rb:475, 588, 631, 687, 771, 815, 933, 984`), e a do 24a é só CPF, para quem tem o cadastro territorial (`src/lib/api.ts:4343`). O `zoonoses_team` não passa nesses portões. **Proposta:** `POST /zoonoses/person_lookup` com `{ cpf }` ou `{ cns }` → `{ patient_id, display_name } | null`, para `team?`, com trilha, como as dos outros módulos.
- **Resolvida pela Q25** — diferente da proposta: **não há** `GET /zoonoses/households`; o domicílio do animal sai da resposta do `person_lookup` (o do cadastro individual ativo da pessoa, com resumo do endereço). Aplicado nas Tasks 1 e 5. **D-5 — Domicílio do animal pelo dashboard.** O §2 aceita domicílio ativo do 24a e a busca por `household_id`, e o §3 aceita a vacinação casa a casa "pelo dashboard com domicílio", mas nenhuma rota lista domicílios para a equipe: `/vector_control/households` é da coordenação e passa pelo `require_vector_control!` (`apps/api app/controllers/vector_control/base_controller.rb:39`). **Proposta:** `GET /zoonoses/households?neighborhood_id=&city_block_id=&before=` → `{ items: [{ id, address_short, neighborhood_id, city_block }], next_before }`, para `team?`, com a trilha `territory_registry`. Se recusada, o domicílio e o casa a casa ficam só pelo app no 35b, e a Task 5 e a Task 6 tiram os campos.
- **Resolvida pela Q11** — aceita, pelo gate novo `VectorControl::Gate.supplies_usable?` (produtos, quarteirões com os anéis, local de estoque, faltas, busca de item e `/vector_control/domains`); o resto do `/vector_control` continua só com `vector_control`. **D-6 — Rotas do 35a que precisam abrir com `zoonoses`.** O §3 abre as rotas de produto e o domínio de estoque, a Q8 abre o cadastro de quarteirões, e o §1 põe o local de estoque da UVZ "fora dos dois gates". Mas todo `/vector_control/*` passa pelo `require_vector_control!` (`base_controller.rb:39`), e a busca de item do 27a confere o `VectorControl::Gate` direto (`apps/api app/controllers/pharmacy/items_controller.rb:36-43`), não o `Stock::Domain`. **Proposta:** com `vector_control` **ou** `zoonoses` utilizável: `GET/POST/PATCH /vector_control/products`, `GET/POST/PATCH /vector_control/blocks` e os vizinhos, `GET/PUT /vector_control/settings`, `GET /vector_control/stock_shortfalls` e o `reconcile`, `GET /vector_control/domains` e `POST /pharmacy/items/search` com `domain: "vector_control"`. Com só `zoonoses`, os imóveis, o perfil, o mapa e os parâmetros do 35a continuam fechados (a Task 4 mostra só a aba "Quarteirões").
- **Resolvida pela Q11** — aceita, e mais estrita: o produto de zoonoses tem **uma** finalidade só (`rabies_vaccine` ou `collar`), `un` e 1.00, sem misturar com as vetoriais. Aplicado na Task 4. **D-7 — Produto com finalidade de zoonoses.** O §3 diz "`unit` `un` (`charge_size` 1)" sem dizer se é regra, nem se a vacina pode ter também `focal`. O formulário de hoje só conhece os três usos do 35a (`src/modules/vectorControl/ProductsPanel.tsx:22-23, 53`). **Proposta:** o api recusa `rabies_vaccine` ou `collar` com `unit` diferente de `un` ou `charge_size` diferente de `"1.00"` (`invalid_request`, `path` `unit`/`charge_size`) e a mistura com os usos do 35a (`path` `uses`); os usos do 35a só com `vector_control` utilizável e os do 35b só com `zoonoses` (o produto já gravado mantém o que tem).
- **Resolvida pela Q17** — aceita com outros valores: `source` é `inspection` | `vaccination` | `collar`, mais `occurred_on` e `visit_on` nulo fora do `inspection`; a tela de faltas do 35a mostra a origem. Aplicado nas Tasks 1 e 4. **D-8 — Falta de insumo sem visita.** A falta do 35a sempre tem `visit_on` (`src/lib/api.ts:8007-8010`), e a do 35b nasce da vacinação ou da coleira (§3). **Proposta:** `visit_on: string | null`, mais `source` (`home_visit` | `rabies_vaccination` | `collar`) e `occurred_on` (a data da visita, da dose ou da coleira) no JSON da falta.
- **Resolvida pela Q16** — aceita: tipos de texto (`"AAAA-MM-DD"`, `"true"`/`"false"`), erro `invalid_form` com `path` `form.<campo>`, PATCH substitui a ficha inteira. **D-9 — A ficha do 25 não aceita booleano nem data, e o erro tem outro nome.** O §5 tipa `animal_observable` como bool e `exposure_on` como data, mas o `Surveillance::Form` só aceita `String` (`apps/api app/services/surveillance/form.rb:59-63`) e os tipos `code`, `text`, `digits`, `uf`, `cbo` (`form.rb:69-75`). A recusa é `invalid_form`, não `invalid_request` (`apps/api app/commands/surveillance/save_form.rb:5`; `src/lib/surveillance.ts:307`), e o PATCH **substitui** a ficha inteira (`save_form.rb:2-3`). **Proposta:** tipos novos no yml, ainda em texto: `date` (`"YYYY-MM-DD"`, até hoje) e `boolean` (`"true"`/`"false"`); o `text` ganha `min`; a recusa continua `invalid_form` com `path` `form.<campo>` (inclusive campo da seção noutra doença e `animal_observable` `"true"` fora de cão e gato); os rótulos dos campos e dos valores ficam no yml do api e o dashboard os copia (propostos na Task 9). O dashboard manda as duas partes num PATCH só.
- **Resolvida pela Q16** — aceita: `animal_outcome_pending: boolean`. A Q16 também tira o estado `cancelled` da observação (só `open` e `closed`) e faz a observação nascer `home` e virar `shelter` na entrada `observation` do canil (Tasks 1, 2, 8 e 11). **D-10 — Forma do sinal `animal_outcome_pending`.** O §5 cria o sinal na fila do 25 sem dizer onde ele vem. **Proposta:** a lista e o detalhe da notificação trazem `animal_outcome_pending: boolean` ao lado do `animal_observation`; sem filtro nem contagem nova.
- **Resolvida pela Q13** — aceita: a fila mostra subtipo, prazo, cor e estado; o extrato e o endereço só no detalhe, com trilha. E o extrato passa a ser **só o texto do cidadão** (subtipo e `place_address` em campos próprios). **D-11 — O extrato da demanda leva o endereço para a fila.** O §4 monta o extrato com "o rótulo do subtipo, o texto do cidadão e o `place_address`", diz que a fila mostra o extrato e que o `place_address` sai **só no detalhe, com trilha**. As duas regras se contradizem. **Proposta:** a fila (`GET /zoonoses/demands`) traz subtipo, prazo, cor, estado e datas, sem extrato nem endereço; o extrato e o `place_address` só no `GET /zoonoses/demands/:id`, que grava `zoonoses_demand.viewed`.
- **Resolvida pela Q12** — aceita: o contato grava trilha e não pede step-up a mais. **D-12 — Step-up no contato da observação.** O §5 manda gravar a trilha `bite_observation.contact_viewed` "como o `address_viewed` do 35a", e o endereço do caso do 35a pedia step-up (`src/modules/vectorControl/CaseAddressDialog.tsx:3`). **Proposta:** sem step-up (a equipe não é privilegiada e o 35b só pede step-up na eutanásia, §7), com a trilha e a leitura só por clique, fora do cache.
- **Resolvida pela Q12 e pela Q25** — o PUT dos parâmetros segue o PUT do 35a (step-up); os códigos propostos ficam na Task 2 como proposta e o api da execução os fixa (a Task 0 confere). **D-13 — Códigos e step-up que o contrato não nomeia.** **Proposta:** transição fora do estado (campanha, inquérito, demanda, observação, estada) → `409 invalid_transition`, `path` `status` (a Q9 do 35a); versões sem nome: `campaign_changed`, `survey_changed` (inquérito de LV) e `collar_changed`; "exige a eutanásia registrada" (§5, §6, §7) → `422 euthanasia_required`; "exige tutor" (§6, §7) → `422 tutor_required`; adoção sem adotante, entrada `observation` sem observação aberta no canil, base legal vazia e data fora da faixa → `422 invalid_request` com o `path` do campo; parâmetro fora da faixa → `422 invalid_parameter`, `path` `values.<chave>` (o do 35a); `PUT /zoonoses/parameters` com step-up, como o do 35a.
- **Resolvida pela Q25** — aceita: `from`/`to`, `campaign_id` e `survey_id`. A Q19 acrescenta as duas coberturas do `rabies_weekly` (sobre a meta e sobre a estimativa, esta só com estimativa; Task 6). **D-14 — Filtros dos relatórios.** O §8 diz "filtro de período" sem as chaves. **Proposta:** `from` e `to` (até 366 dias, o `Reports::MAX_DAYS` do 35a); no `rabies_weekly`, `campaign_id` obrigatório e o período é o da campanha; no `leish`, `survey_id` opcional; o CSV no mesmo caminho com `.csv`.
- **Resolvida pela Q18** — diferente da proposta: sai o parâmetro `leish_expected_prevalence` (a tabela do manual é a de 2% e é ela que se edita); população acima da última linha usa a última amostra; a sugestão de população é a dos cães ativos ou, com zero, quarteirões × `dogs_per_block`; o foco sugere os anéis do `VectorControl::Rings` (não um anel fixo); o desfecho `euthanasia` liga a eutanásia por `euthanasia_id`. Aplicado nas Tasks 1, 2, 10, 11 e 13. **D-15 — Sugestões do inquérito sem fonte.** O §6 diz que a tela sugere a soma dos cães ativos nos quarteirões e que o `focus` sugere "os anéis do 35a", mas nenhuma rota dá a soma, e o número de anéis é parâmetro do 35a (`block_radius_rings`), que só se lê com `vector_control`. A regra do `sample` ("população × prevalência esperada" e "a primeira linha cuja população seja ≥") também não fecha a conta sozinha. **Proposta:** o inquérito traz `registered_dogs` (a soma) e o `target_dogs` já no rascunho; o `focus` sugere um anel (os vizinhos diretos, `CityBlock.neighbor_ids`) e o coordenador ajusta; o plano do api fixa a fórmula do `sample` com exemplo numérico.
- **Resolvida pela Q25** — aceita: "Relatórios de zoonoses" e "Parâmetros de zoonoses" (Task 3). **D-16 — Rótulos curtos que repetem outros itens.** "Relatórios" e "Parâmetros" (§13) repetem itens de outros grupos (`src/shell/modules.ts:65`, "Relatórios" da Triagem). **Proposta:** "Relatórios de zoonoses" e "Parâmetros de zoonoses", como o 35a fez com "Relatórios de endemias".
- **Resolvida pela Q14** — aceita: o registro do balcão aceita as duas chaves só com `kind` `request` ou `report`, e entra na Task 14. **D-17 — Registro pela equipe da ouvidoria.** O §4 diz que a equipe da ouvidoria, no balcão, no telefone ou pelo OuvidorSUS, aceita `zoonoses_subtype` e `place_address`, mas o §13 não lista a tela de "Nova manifestação" do dashboard nem diz se o detalhe da manifestação mostra o `place_address` à equipe. **Proposta:** o dashboard oferece as duas perguntas na "Nova manifestação" (Task 14), com as frases do wpda (§14), e o detalhe da manifestação mostra o endereço à equipe da ouvidoria, que já vê a identidade. Até a decisão, a Task 14 faz só o cadeado.
- **Resolvida pela Q14** — aceita: o JSON do assunto traz `code`. **D-18 — O `code` do assunto não está no JSON.** O §1 cria `ombudsman_subjects.code` e o §13 pede o cadeado, mas `GET /ombudsman/subjects` não traz o campo (`src/lib/api.ts:7206-7211`). **Proposta:** `code: string | null` no pai e nos filhos.
