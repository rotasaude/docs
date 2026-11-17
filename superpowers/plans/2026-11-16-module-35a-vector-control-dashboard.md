# Módulo 35a — Vigilância ambiental de endemias (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:** (1) o **contrato** `2026-11-16-module-35a-vector-control-contracts.md` (este plano o segue; as Divergências D-1 a D-20, no fim, voltam a ele como Q-n); (2) o **api do 35a** na branch `feat/mod-35a-vector-control` (`apps/api/.claude/mod35a`) — as Tasks 1–18 seguem pelas formas deste plano e pelas fixtures, e só a Task 19 (prova) depende do api rodando; (3) a `main` do dashboard em `677a511` (24c mergeado: `createFieldMap`, `MicroAreaMap`, `fieldMapUrl`). Ordem do contrato §19: contracts → api → **dashboard** e maintenance → campo.

> **Reconciliação (§20 do contrato, Q6–Q12, docs `aa4f3c5`):** D-1..D-20 aceitas na Q8, com estas mudanças já aplicadas no plano: D-3 vira a rota `GET /vector_control/households/:id` (o `<household>` do 24a **não** ganha `vector`); o `status_changed` da D-10 vira `409 invalid_transition` (mesma frase), nota ausente fica `422 invalid_request` (Q9); a tabela de `path` da Task 2 é a fonte única para api e contracts (Q9); Q7 (`sample_label_taken`, nomes dos `limits`, `measures_max`, `consent: false` → `consent_required`, número da ovitrampa `^[0-9A-Za-z-]{1,12}$`); Q10 (semana `"YYYY-WW"`, boletim sem domicílio → `vector_inspection_invalid` em `household_id`, degrau (cidade, qualquer risco) na cadência, `409 unit_has_vector_control_location`, quantidade com teto em 2 casas, estorno `vector_application_reversal`, erro do 24b em motivo derivado volta em `subject`); Q6(4) (semente só depois do APK do 35a no emulador); Q12 (só a descrição do `CATALOG` do maintenance; o rótulo curto do dashboard não muda). **Segunda rodada (docs `c2d755c`, Q13):** o agente de endemias cadastra e edita imóvel **não residencial** do 24a nas microáreas atribuídas (Task 18A); `sample_label_taken` é 422 com uma frase só; `path` da ovitrampa fixados; `GET /vector_control/households/:id` → `{ id, neighborhood_id, vector }`; `cases[].number` é sequência da cidade, imutável; categorias do `map_points` as do plano do api (`transmission` só `area`), e ler pontos grava a trilha `territory_map`; `created_at` da versão 0 dos parâmetros é `null`; mover caso para bloqueio fechado → `invalid_transition`; o api devolve só o primeiro erro, e a lista do dashboard começa pelo mesmo. Cada divergência, no fim, diz a Q que a decidiu.

**Goal:** Lado dashboard do 35a (F-35.1 a F-35.9): o grupo **"Endemias"** (quarteirões e imóveis, ciclos e cadência, ovitrampas, amostras, LIRAa, bloqueio, tarefas, relatórios, produtos e parâmetros), o **boletim** dentro da visita do agente de endemias (formulário e detalhe, com os motivos 34–37 derivados e a etiqueta das amostras), **"Imóveis a visitar"** na Folha do dia, **"Ovitrampas de campo"** do agente, o tipo de unidade **"Vigilância ambiental e zoonoses"**, o domínio **`vector_control`** nas telas de estoque do 27a com a **conciliação da falta de insumo**, os **downloads** CSV e PDF e os **mapas** por risco, contagem e área.

**Architecture:** O cliente ganha a seção "Vigilância ambiental — endemias (módulo 35a)" no fim de `src/lib/api.ts` (formas do contrato §1–§12 e das Divergências) e estende os tipos do 24b (`HomeVisitInput.vector_inspection`, `HomeVisit.vector_inspection`), do 27a (`StockItem.vector_product_id`). As regras do boletim (§3) são portadas para um módulo **puro** `src/lib/vectorInspection.ts` que devolve exatamente `{ error, path }` com os códigos do contrato, e as frases vivem numa tabela única `src/lib/vectorErrors.ts` (uma por código; é a tabela que o campo copia). Portões, rótulos e as demais regras de tela ficam em `src/lib/vectorControl.ts`. As telas ficam em `src/modules/vectorControl/` e nas raízes `src/modules/Vector*.tsx`, `FieldOvitraps.tsx`; o boletim entra no `VisitForm`/`VisitDetail` do 24b como uma seção (`src/modules/homeVisits/InspectionSection.tsx`). O perfil vetorial do imóvel vem de `GET /vector_control/households/:id` (Q8; o `<household>` do 24a não muda). Os mapas reusam o adaptador do 24c (`createFieldMap`), que ganha cores por categoria. O dashboard nunca decide o que o api garante (escopo, cadência, sorteio, agrupamento do bloqueio, baixa de estoque, índices, trilha): a tela só não oferece nem envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), `@tabler/icons-react` 3, `maplibre-gl` e `pmtiles` já fixados pelo 24c.

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisões 1–7 do usuário; 1–13 e 23–30 do coordenador; §4 telas; §5 testes; §7 F-35.1–F-35.9). **Contrato:** `docs/superpowers/plans/2026-11-16-module-35a-vector-control-contracts.md` §1–§12, §15, §17, §18. Modelos de formato: os planos do dashboard do 24b e do 24c.

> **Notas da execução (coordenador, 2026-11-17)** — valem por cima das tasks abaixo:
> - **Q17:** os parâmetros ganham `liraa_half_household_fraction: number` (padrão 0,5, faixa 0,01–1,0, duas casas) ao lado de `liraa_household_fraction` (a do `standard`); entra no tipo da linha dos parâmetros e na tela de parâmetros (Task dos parâmetros). O intervalo é `round(1 ÷ fração)`, não é campo.
> - **Q15:** `charge_size` é texto `"^[0-9]{1,8}\.[0-9]{2}$"` na referência e em `/vector_control/domains` (o plano já tipa como `string`).
> - **Q16(f):** rótulo de `B` = "B — Depósito móvel" (vem dos domínios; nenhum texto fixo no dashboard).
> - **Q18:** `GET /vector_control/blocks?neighborhood_id=` devolve o bairro inteiro, em ordem natural do `number`, sem `next_before`; `before` dá `422`.
> - **Ordem do 1º erro (api Task 6, Q13(10)):** nas listas do boletim, o api confere o **tamanho da lista** (depósitos 7, medidas 20, amostras 99) antes dos itens, e as **faixas** antes das desigualdades. O `checkInspection` devolve as recusas nessa ordem.
> - **Q21 (Task 18):** com `vector_control` utilizável e `pharmacy` desligado, o grupo "Farmácia" mostra só o que é do local `vector_control` (administração do local e da equipe, entrada de nota, posição, faltas de insumo); os testes da linha da `vector-stock-shortfalls` ganham o caso sem `pharmacy`. Causa `no_lot` = nenhum lote do produto no local; com lote sem saldo válido, `insufficient_balance`.
> - **Q21, como o api ficou (Task 7):** com `pharmacy` ligado e `vector_control` desligado, o admin esconde os locais `vector_control` e responde `403` com `feature: "vector_control"` a eles; com `pharmacy` desligado, `GET /pharmacy/locations` devolve `200 []` a quem só tem local de medicamento (a tela mostra o vazio, não erro).
> - **Q23 (Tasks das ovitrampas):** `ovitrap_number_taken` é `422` `path` `number`; corrigir leitura exige contagem diferente da gravada (`countProblems` recusa só a nota nova, `path` `egg_count`).
> - **Q24 (LIRAa):** o N do estrato no editor é o `stratum.households_count` que o api devolve no `draft` (sem PE), não a soma dos quarteirões; o índice de levantamento fechado fica "provisório" até todas as amostras terem resultado.
> - **Q26 (bloqueio):** `confirm` manda `seen_case_ids` (os ids dos casos mostrados) e pode dar `409 proposal_changed` (entrou caso depois da leitura): a tela relê a proposta, avisa "Entrou caso novo nesta proposta; confira a área" e mantém a seleção. `422` `path` `seen_case_ids` (caso movido para fora depois da leitura; teto de 500 ids) também relê a proposta e avisa do mesmo jeito. O endereço do caso só abre em bloqueio proposto ou ativo (`409 invalid_transition` nos outros); o caso movido mostra quem moveu.
> - **Q27 (amostras):** `sampleResultProblems` recusa a correção que só troca a nota (`path` `examined_larvae`), como o `countProblems`; a aba "Com resultado" vem do mais recente para o mais antigo, a "Pendentes" da coleta mais antiga para a mais nova.
> - **Q28 (relatórios e mapa):** o laboratório (`vector_lab`) vê os relatórios **Ovitrampas e LIRAa** (Task 17); na camada do bloqueio o ponto não traz o número do quarteirão; o CSV do LIRAa marca "Provisório". `map_points` sem teto de pontos por bairro, como o 24c.
> - **Q19/Q20:** erros de escopo do agente no cadastro (403 `out_of_scope` ao criar fora da microárea, 404 ao editar/ler, `cbo_not_allowed` com morador ativo e na família); `vacant` no LIRAa é substituído.

## Global Constraints

- **Nomes no código** (contrato §1, §3): interruptor `vector_control` (rótulo curto "Vigilância ambiental (endemias)"); papéis `endemic_coordinator` ("Coordenação de endemias") e `vector_lab` ("Laboratório de entomologia"); CBO do agente `515140`; atividades `li` `li_t` `pe` `ie` `transmission_block` `liraa`; trabalho `routine` | `pending`; situação `worked` `closed` `refused` `vacant`; grupos `A1` `A2` `B` `C` `D1` `D2` `E`; métodos `mechanical` `focal` `perifocal` `spatial`; risco `green` `yellow` `red` `gray` `purple`; unidade `environmental_surveillance` ("Vigilância ambiental e zoonoses"); local `vector_control`; movimento `vector_application`. **Código desconhecido aparece cru, nunca some.**
- **Portões:** `canUseVectorControl(user)` = não operador, `vector_control` na sessão **e** `canUseHomeVisits(user)` (Q1). A sessão não traz o CBO: o agente é reconhecido pela equipe do `GET /home_visits/context` com `cbo === "515140"` e `can_register` (`isEndemicAgentTeam`). Coordenação e laboratório pelos papéis da sessão (valem para a cidade inteira). O `municipal_admin` só lê os relatórios. O api confere de novo (`403 cbo_not_allowed`, `403 missing_role`, `403 feature_disabled`).
- **Sigilo** (§17): nenhuma tela do 35a mostra morador; o caso do bloqueio aparece como "caso N", doença e data; o endereço do caso só pelo `GET …/cases/:case_id/address`, com step-up, lido no clique, guardado só no estado local do diálogo, **nunca** no `queryClient`, em storage, em `console` ou na URL; nenhuma coordenada ligada a caso (o mapa do bloqueio mostra os imóveis da área, sem o caso). Na URL vão só ids, datas, semanas, estados e o cursor.
- **Leituras:** listas de imóveis do 35a (`/vector_control/households`, gravam trilha `territory_registry`) com `NOMINAL_QUERY` (nunca relidas sozinhas); contagens, índices, relatórios, filas sem pessoa, domínios e parâmetros com `STATIC_QUERY`, relidos depois de cada ação. **Nenhum polling.**
- **Recusar em vez de cortar:** notas e motivos 10–500 (code points depois do `trim`, `textProblem` do 24b); número do quarteirão `/^[0-9A-Za-z-]{1,10}$/`; número da ovitrampa `^[0-9A-Za-z-]{1,12}$` (Q7); ovos 0–9999 inteiro; cargas > 0 e ≤ 999 com até duas casas (vírgula aceita); contagens de depósito 0–999; larvas 1–10; nome do levantamento 1–80, do estrato 1–60, do produto 1–80, código 1–10.
- **Step-up** (`SensitiveAction`): `PUT /cadences`, `PUT /parameters`, sorteio e fechamento do LIRAa, confirmação do bloqueio e endereço do caso. Escrita sempre com corpo JSON (`{}` sem campo).
- **Concorrência:** 409 (`survey_not_draft`, `already_reconciled`, `invalid_transition` (Q9), `block_in_use`, `product_in_use`) → a frase do código, **uma** releitura do que estava na tela, nenhum reenvio. Botão de escrita desabilitado enquanto a chamada corre.
- **Boletim:** só para a equipe do agente com o gate utilizável; opcional ("Registrar boletim", ligado por padrão); com ele, o **desfecho sai da situação** (`outcomeForSituation`) e os motivos 34–37 **não** são oferecidos nem enviados (o api os deriva; a tela mostra a prévia). LIRAa e bloqueio só como atividade quando a visita nasce de "Imóveis a visitar" com o vínculo do item (a tela não adivinha o item). Etiqueta das amostras: prévia "provisória" no formulário; a definitiva é a da visita gravada (D-12).
- **Interface:** toda tela nova em `NAV_GROUPS` **e** `CATALOG.screens` (o `catalog.test.ts` cobra); o agente no celular primeiro (375 px: cartões, campos empilhados, nenhuma `<table>` nas telas do agente); telas da coordenação com `DataTable` (`card: "title"`/`"actions"`); cores só por papéis do tema (`Tag` com `tone`; no mapa, `--ok`, `--warn`, `--down`, `--ink3`, `--accent` lidos por `getComputedStyle`); guardas `src/theme/*.test.ts` verdes. Componentes existentes: `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `ActionMessage`, `NeighborhoodPicker`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts`.
- **CI (lição do `443faca`):** todo `vi.mock("…/lib/api", async (orig) => …)` inclui `fetchCurrentSession: vi.fn()` resolvendo o usuário.
- "Agora" é `new Date()`; telas recebem `now`/`today` por prop; os testes usam `NOW35 = new Date("2026-11-16T10:00:00-03:00")`, `TODAY35 = "2026-11-16"` (`TZ=America/Sao_Paulo` no `vitest.config.ts`). Datas da cidade por `cityLocalIso`/`cityIsoDate` de `src/lib/format.ts`.
- **Testes:** só `$R/bin/test dashboard -w dashboard-mod35a [arquivos]`, nunca `vitest` direto; tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- **Git:** `/opt/homebrew/bin/git`; nunca `git add -A` nem `git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge, push e tag só com autorização explícita.
- **Board #1:** o dashboard é superfície de F-35.1 a F-35.9. Um card vai a Done só quando o **último** app dele termina (F-35.1: api, dashboard e maintenance; os outros: api e dashboard) e a Verified depois da prova (Task 19).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`): `dashboard · dashboard-mod35a/feat/mod-35a-vector-control · Task N · <pronto> · <divergência nova>`.

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod35a`, `G=/opt/homebrew/bin/git`. Worktree **fora de `apps/dashboard`** (o compose monta `./apps/dashboard:/app`; `node_modules` ali derruba o Vite com React duplicado), branch `feat/mod-35a-vector-control` a partir de `origin/main` (`677a511`), com `npm ci`. Caminhos relativos a `$W`.
- Sem jest-dom (`not.toBeNull()`, `toBe()`); `globals: false`; `cleanup` no `afterEach`. `renderWithProviders`/`sessionWith` de `src/test/campaignFixtures.tsx`; as do 24b em `src/test/homeVisitFixtures.ts` (`context`, `acsTeam`, `visit`, `VISIT_DOMAINS`, `TODAY24B`); as do 24c em `src/test/territoryMapFixtures.ts` (`MAP_MANIFEST`); as novas em `src/test/vectorControlFixtures.ts` (Task 1).

## Review Focus

1. **Boletim que manda os motivos 34–37 ou estoura o limite depois da derivação.** O agente marca "Imóvel com foco" à mão, ou tem 35 motivos e o boletim acrescenta mais dois. Espera-se: com o boletim, 34–37 somem do `ReasonsField`, a prévia "O boletim acrescenta: …" aparece, o aviso de família (`FAMILY_NOTICE`, Q43) considera os derivados, e o envio para quando motivos + derivados passam de `reasons_max`. Teste: Task 5, "com boletim: 34–37 fora da lista, prévia dos derivados, limite com os derivados e aviso de família".
2. **Etiqueta errada no tubito.** O 24b troca o `client_uuid` depois de qualquer 4xx, e a etiqueta nasce dele (§9): se o agente etiquetou pela prévia e a visita foi recusada, a etiqueta muda. Espera-se: a prévia diz "provisória até gravar", e depois do 201 a tela lista as etiquetas **da visita gravada** ("Etiquete os tubitos: …"). Teste: Task 5, "etiqueta: prévia provisória; 4xx troca; a definitiva vem da visita gravada".
3. **Trocar a situação deixa sobra.** O agente preenche depósitos e amostras e depois marca "fechado". Espera-se: `pruneInspection` tira depósitos, medidas, amostras e ação educativa, o desfecho vira "ausente", e uma linha `role="status"` diz o que saiu (nada some calado). Teste: Task 2 (`pruneInspection`) e Task 5, "fechado depois de trabalhado: poda e diz o quê".
4. **Endereço do caso vaza.** Espera-se: o endereço só aparece depois do step-up, no diálogo, e some ao fechar; nenhuma chave do `queryClient` o guarda; lista, detalhe e mapa do bloqueio mostram só "caso N", doença e data. Teste: Task 15, "endereço do caso: step-up, só no diálogo, nada no cache; lista sem paciente".
5. **Sorteio ou fechamento do LIRAa repetido** (duplo clique, aba velha). Espera-se: um envio; `409 survey_not_draft`/`invalid_transition` → a frase e **uma** releitura do levantamento, sem reenviar. Teste: Task 14, "sorteio: step-up, um envio; 409 relê uma vez".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.vectorControl.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/devProxy.ts`, `src/devProxy.test.ts`, `vite.config.ts`, `src/test/vectorControlFixtures.ts` | tipos e cliente do 35a; extensões do 24a/24b/27a; rótulo do interruptor; proxy; fixtures | 1 |
| `src/lib/vectorErrors.ts`, `src/lib/vectorErrors.test.ts`, `src/lib/vectorInspection.ts`, `src/lib/vectorInspection.test.ts` | frases de todos os códigos; regras puras do boletim (§3) | 2 |
| `src/lib/vectorControl.ts`, `src/lib/vectorControl.test.ts`, `src/lib/homeVisits.ts`, `src/lib/homeVisits.test.ts` | portões, papéis, chaves, rótulos, regras de ovitrampa, contagem, amostra, ciclo, estrato; recusas | 3 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.vectorControl.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/vectorControl/VectorGate.tsx`, `src/modules/vectorControl/VectorGate.test.tsx`, cascas das 11 telas-raiz | grupo "Endemias", itens fora dele, portão | 4 |
| `src/modules/homeVisits/InspectionSection.tsx`, `src/modules/homeVisits/InspectionDetail.tsx`, `src/modules/homeVisits/VisitForm.tsx`, `src/modules/homeVisits/useVisitHandoff.ts`, `src/modules/homeVisits/VisitDetail.tsx`, `src/modules/homeVisits/ReasonsField.tsx`, `src/modules/homeVisits/VisitForm.vector.test.tsx`, `src/modules/homeVisits/VisitDetail.vector.test.tsx` | boletim na visita (F-35.2, F-35.3, F-35.8) | 5 |
| `src/modules/vectorControl/DueList.tsx`, `src/modules/HomeVisitDay.tsx`, `src/modules/HomeVisitDay.due.test.tsx` | Imóveis a visitar (F-35.4) | 6 |
| `src/modules/FieldOvitraps.tsx`, `src/modules/vectorControl/InstallForm.tsx`, `src/modules/vectorControl/CollectForm.tsx`, `src/modules/FieldOvitraps.test.tsx` | ovitrampas do agente (F-35.5) | 7 |
| `src/modules/territory/map/fieldMap.ts`, `src/modules/territory/map/fieldMap.test.ts`, `src/modules/territory/map/points.ts`, `src/modules/territory/map/points.test.ts`, `src/modules/vectorControl/VectorMap.tsx`, `src/modules/vectorControl/VectorMapPanel.tsx`, `src/modules/vectorControl/VectorMapPanel.test.tsx` | cor por categoria no adaptador; mapa do 35a | 8 |
| `src/modules/VectorBlocks.tsx`, `src/modules/vectorControl/BlocksPanel.tsx`, `src/modules/vectorControl/HouseholdsPanel.tsx`, `src/modules/VectorBlocks.test.tsx` | quarteirões, vizinhos, imóveis e atribuição (F-35.1) | 9 |
| `src/modules/VectorSettings.tsx`, `src/modules/vectorControl/ProductsPanel.tsx`, `src/modules/vectorControl/ParametersPanel.tsx`, `src/modules/VectorSettings.test.tsx` | produtos, parâmetros, local de estoque (F-35.3) | 10 |
| `src/modules/VectorCycles.tsx`, `src/modules/vectorControl/CadencesPanel.tsx`, `src/modules/VectorCycles.test.tsx` | ciclos, resumo por bairro, cadência (F-35.4) | 11 |
| `src/modules/VectorOvitraps.tsx`, `src/modules/vectorControl/CountForm.tsx`, `src/modules/VectorOvitraps.test.tsx` | ovitrampas da coordenação e do laboratório (F-35.5) | 12 |
| `src/modules/VectorSamples.tsx`, `src/modules/VectorSamples.test.tsx` | fila e resultado de amostras (F-35.8) | 13 |
| `src/modules/VectorLiraa.tsx`, `src/modules/vectorControl/StrataEditor.tsx`, `src/modules/VectorLiraa.test.tsx` | LIRAa (F-35.6) | 14 |
| `src/modules/VectorTransmission.tsx`, `src/modules/vectorControl/BlockDetail.tsx`, `src/modules/vectorControl/CaseAddressDialog.tsx`, `src/modules/VectorTransmission.test.tsx` | bloqueio de transmissão (F-35.7) | 15 |
| `src/modules/VectorTasks.tsx`, `src/modules/VectorTasks.test.tsx` | tarefas da coordenação (F-35.5, F-35.7) | 16 |
| `src/modules/VectorReports.tsx`, `src/modules/vectorControl/ReportTable.tsx`, `src/modules/VectorReports.test.tsx` | relatórios e downloads (F-35.9) | 17 |
| `src/lib/attendance.ts`, `src/lib/attendance.test.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitForm.vector.test.tsx`, `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts`, `src/modules/pharmacy/{LocationsPanel,ReceiptForm,ItemSearch,TransferForm}.tsx`, `src/modules/PharmacyTransfers.tsx`, `src/modules/PharmacyEntries.test.tsx`, `src/modules/PharmacyTransfers.test.tsx`, `src/modules/PharmacyAdmin.test.tsx`, `src/modules/VectorStockShortfalls.tsx`, `src/modules/VectorStockShortfalls.test.tsx` | tipo de unidade, domínio `vector_control` no 27a, faltas de insumo (F-35.1, F-35.3) | 18 |
| `src/lib/api.ts` (`MicroArea` do contexto do 24a), `src/lib/territoryRegistry.ts`, `src/lib/territoryRegistry.test.ts`, `src/modules/TerritoryTeam.tsx`, `src/modules/territory/HouseholdForm.tsx`, `src/modules/territory/HouseholdDetail.tsx`, `src/modules/TerritoryTeam.vector.test.tsx` | imóvel não residencial pelo agente de endemias no 24a (Q13(1)) | 18A |
| — | suíte, build, varreduras, revisão, prova no navegador (§18) | 19 |
| — | board, relato, parada antes do merge | 20 |

**Estratégia de teste:** cliente com `fetch` falso (`stub`, `last`, `bodyOf` do `api.ombudsman.test.ts`) conferindo URL, método e corpo — nada de pessoa na URL, nota e endereço só no corpo/resposta (Task 1); regras puras por tabela de casos com os códigos e `path` exatos do contrato e as bordas do §19 (Tasks 2–3); menu por papel e interruptor (Task 4); cada tela pelo ponto de entrada com `renderWithProviders`, `vi.mock` do api com `fetchCurrentSession`, sessão por papel e contexto por CBO, cobrindo caminho feliz, cada recusa que muda a tela, duplo envio, 409 com uma releitura e o que **não** pode aparecer (morador, endereço do caso, 34–37 no corpo) (Tasks 5–18); mapa com o adaptador simulado (`vi.mock("../territory/map/fieldMap")`) e o adaptador com `maplibre-gl` simulado (Task 8); `tsc`, build, suíte, varreduras e prova no navegador com o api real na 3059 e o Vite na 5220 (Task 19). O dashboard não testa o que é do api (cadência, sorteio, agrupamento, baixa FEFO, índices, trilha, sigilo do evento, feed do app).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`677a511` ou mais novo); board #1; o api `feat/mod-35a-vector-control`.
- Produces: cards em In Progress; `$W` na branch; base, âncoras, ícones e formas do api anotados.

- [ ] **Step 1: Cards** (um `gh` simples por vez, sem `$(...)` nem laço) — `gh project item-list 1 --owner rotasaude --limit 800 --format json --jq '.items[] | select(.title | test("^F-35\\.[1-9] ")) | {id, title, status}'`; cada um de F-35.1 a F-35.9 que não estiver em In Progress ou além vai a In Progress com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <In Progress>`. Card ausente: **pare e pergunte**.

- [ ] **Step 2: Âncoras**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin && $G -C $R/apps/dashboard log --oneline -1 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const HOME_VISITS_BASE\|^export interface HomeVisitInput {\|^export interface HomeVisit {\|^export interface HomeVisitTeam {\|^export interface VisitHouseholdDetail\|^export async function getVisitHousehold\|^export interface Household {\|^export interface StockItem {\|^export interface ReceiptItemInput {\|^async function binaryFetch\|^function hvQuery\|^export interface FieldMapManifest'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '"home_visits" | "collective_activities";'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '{ label: "Vigilância", items: \[\|"pharmacy-urgency-shortfalls", label\|const hvOn = canUseHomeVisits(user);\|NAV_GROUPS.filter((group) => {'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - '
$G -C $R/apps/dashboard show origin/main:src/lib/homeVisits.ts | grep -n '^export const HOME_VISIT_ERRORS\|^export function homeVisitRefusals\|^function disabledText\|^export function familyVisitNotice'
$G -C $R/apps/dashboard show origin/main:src/modules/homeVisits/useIdempotentSubmit.ts | grep -n 'uuid: id.uuid'
$G -C $R/apps/dashboard show origin/main:src/modules/territory/map/fieldMap.ts | grep -n '^export type FieldMapOptions\|cluster: true'
$G -C $R/apps/dashboard show origin/main:src/lib/attendance.ts | grep -n '^export function unitKindOptions'
$G -C $R/apps/dashboard show origin/main:src/lib/pharmacy.ts | grep -n '^const KIND_LABEL\|^export const itemKey\|^const MOVEMENT_LABEL'
$G -C $R/apps/dashboard show origin/main:src/devProxy.ts | grep -n 'MODULE_24B_PROXY_KEYS'
```

Expected: cada linha achada. Linha mudada: a task aplica a mesma intenção sobre o texto real e anota no commit.

- [ ] **Step 3: Worktree, base e ícones**

```bash
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod35a -b feat/mod-35a-vector-control origin/main
cd $R/.worktrees/dashboard-mod35a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod35a
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Bug|GridDots|LayoutGrid|CalendarRepeat|Egg|TestPipe|Dice|ShieldHalf|ListCheck|ReportAnalytics|Adjustments|MapPinCheck|PackageOff)\.mjs$'
```

Expected: `tsc-ok`, suíte verde (**anote** arquivos e testes) e os ícones (faltou algum → outro que exista, o mesmo na Task 4).

- [ ] **Step 4: Formas do api** — `$G -C $R/apps/api show feat/mod-35a-vector-control:config/routes.rb | grep -n vector_control` e os serializadores de `app/**/vector_control/*json*.rb`. Para cada Divergência (D-1 a D-20) anote se o api adotou a forma proposta; **onde o api diz outra coisa, vale o api**: a task dona ajusta tipo, fixture e teste (registre no commit). Sem branch ainda: siga pelas formas deste plano e repita este passo antes da Task 19.

---

### Task 1: Tipos, cliente HTTP, rótulo do interruptor, proxy e fixtures

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; extensões nas seções do 24a, 24b e 27a), `src/lib/features.ts`, `src/devProxy.ts`, `vite.config.ts`
- Create: `src/lib/api.vectorControl.test.ts`, `src/test/vectorControlFixtures.ts`
- Test: `src/lib/features.test.ts`, `src/devProxy.test.ts`

**Interfaces:**
- Produces (em `src/lib/api.ts`, seção "Vigilância ambiental — endemias (módulo 35a; ADR 0054; contrato 2026-11-16 §1–§12)"; `VECTOR_BASE = import.meta.env.VITE_VECTOR_CONTROL_BASE || "/vector_control"`, `vcPath(...parts)` como `hvPath`, `vcWrite(method, body = {})`, query por `hvQuery`, listas por `hvPage`; decimais em texto como no 27a):

```ts
export type VectorActivity = "li" | "li_t" | "pe" | "ie" | "transmission_block" | "liraa";
export type VectorWorkType = "routine" | "pending";
export type VectorSituation = "worked" | "closed" | "refused" | "vacant";
export type DepositGroup = "A1" | "A2" | "B" | "C" | "D1" | "D2" | "E";
export type MeasureMethod = "mechanical" | "focal" | "perifocal" | "spatial";
export type RiskClass = "green" | "yellow" | "red" | "gray" | "purple";
export interface DepositCounts { group: DepositGroup; existing: number; with_water: number; with_larvae: number; eliminated: number; treated: number }
export interface VectorMeasureInput { method: MeasureMethod; vector_product_id?: string; charges?: number }
export interface VectorSampleInput { number: number; deposit_group: DepositGroup; larvae_collected: number }
export interface VectorInspectionInput {
  activity: VectorActivity; work_type: VectorWorkType; situation: VectorSituation; educational_action: boolean;
  entered_at?: string; left_at?: string; city_block_id?: string; risk_class?: RiskClass;
  liraa_draw_item_id?: string; transmission_block_id?: string;
  deposits: DepositCounts[]; measures: VectorMeasureInput[]; samples: VectorSampleInput[];
}
export interface SampleResult { examined_larvae: number; aegypti_larvae: number; albopictus_larvae: number; result_at: string; corrected: boolean }
export interface VectorProductRef { id: string; code: string; name: string; unit: "g" | "ml" | "un" | (string & {}) }
export interface VectorInspection extends Omit<VectorInspectionInput, "measures" | "samples" | "city_block_id"> {
  id: string; recovered: boolean; city_block: { id: string; number: string } | null;
  measures: (VectorMeasureInput & { quantity: string | null; product: VectorProductRef | null })[];
  samples: (VectorSampleInput & { id: string; label: string; result: SampleResult | null })[];
}
// Mesma forma do `vector_control_domains` da referência do field-sync (§13; D-2).
export interface VectorControlDomains {
  activities: { code: VectorActivity; label: string }[]; work_types: { code: VectorWorkType; label: string }[];
  situations: { code: VectorSituation; label: string }[]; deposit_groups: { code: DepositGroup; label: string }[];
  measure_methods: { code: MeasureMethod; label: string }[]; risk_classes: { code: RiskClass; label: string }[];
  products: (VectorProductRef & { charge_size: string; uses: Exclude<MeasureMethod, "mechanical">[] })[];
  cadences: VectorCadence[];
  parameters: { default_days: number; strategic_point_days: number; ovitrap_collection_days_min: number; ovitrap_collection_days_max: number };
  // Q7(3): 999, 999, 99, 10, 20; `deposits` tem no máximo 7 itens (um por grupo).
  limits: { deposit_count_max: number; charges_max: number; samples_max: number; larvae_collected_max: number; measures_max: number };
}
// O mesmo `vector` do `<household>` do field-sync (§13), servido por `GET /vector_control/households/:id` (Q8).
export interface HouseholdVectorProfile { id: string; neighborhood_id: string | null; vector: HouseholdVector | null }   // Q13(5)
export interface HouseholdVector {
  city_block_id: string | null; risk_class: RiskClass | null; risk_assessed_on: string | null; last_worked_on: string | null; last_situation: VectorSituation | null;
}
export interface CityBlock { id: string; neighborhood: { id: string; name: string }; number: string; active: boolean; neighbor_ids: string[]; households_count: number; version: number }
export interface VectorHousehold {
  id: string; address_short: string; property_type: string; neighborhood_id: string | null; micro_area: { id: string; code: string };
  block: { id: string; number: string; neighborhood_id: string } | null; risk_class: RiskClass | null; risk_assessed_on: string | null;
  last_worked_on: string | null; last_situation: VectorSituation | null;
}
export type DueReason = "transmission_block" | "liraa" | "pending" | "due" | (string & {});
export interface DueItem {
  household_id: string; address_short: string; property_type: string; block: { id: string; number: string } | null; risk_class: RiskClass | null;
  last_worked_on: string | null; due_since: string | null; reason: DueReason; liraa_draw_item_id: string | null; transmission_block_id: string | null;
}
export interface VectorCadence { neighborhood_id: string | null; risk_class: RiskClass | null; strategic_point: boolean; days: number }
export interface VectorCycle { id: string; year: number; number: number; starts_on: string; ends_on: string }
export interface CycleSummary {
  cycle: VectorCycle;
  neighborhoods: { neighborhood: { id: string; name: string }; to_visit: number; visited: number; worked: number; closed: number; refused: number;
    vacant: number; recovered: number; pending_percent: number | null; registered_households: number }[];
}
export interface VectorParameters {
  default_days: number; strategic_point_days: number; block_radius_rings: number; block_grouping_days: number; block_diseases: string[];
  ubv_cycles_max: number; ubv_cycle_days_min: number; ubv_cycle_days_max: number; ubv_cycle_interval_days: number;
  ovitrap_collection_days_min: number; ovitrap_collection_days_max: number; ovitrap_alert_eggs: number; ovitrap_radius_rings: number;
  liraa_standard_min: number; liraa_standard_max: number; liraa_half_min: number; liraa_base_sample: number; liraa_household_fraction: number;
  iip_bands: [ number, number ];
}
// Q13(8): a versão 0 (padrão de fábrica, sem gravação) tem `created_at: null`.
export interface VectorParameterVersion { version: number; values: VectorParameters; created_at: string | null }
export interface VectorProduct extends VectorProductRef { charge_size: string; uses: Exclude<MeasureMethod, "mechanical">[]; active: boolean; in_use: boolean }
export interface VectorProductInput { code: string; name: string; unit: "g" | "ml" | "un"; charge_size: string; uses: Exclude<MeasureMethod, "mechanical">[]; active: boolean }
export interface VectorSettings { stock_location: { id: string; name: string } | null; options: { id: string; name: string }[] }
export type OvitrapStatus = "active" | "removed" | "lost" | (string & {});
export type ReadingCondition = "ok" | "damaged" | "lost" | "dry";
export interface Ovitrap {
  id: string; number: string; household: { id: string; address_short: string }; city_block: { id: string; number: string }; status: OvitrapStatus;
  installed_on: string; removed_on: string | null; open_reading: { id: string; installed_on: string } | null;
  last_reading: { id: string; collected_on: string; condition: ReadingCondition | (string & {}); egg_count: number | null; epi_week: string } | null; version: number;
}
export interface OvitrapReading {
  id: string; ovitrap: { id: string; number: string }; city_block: { id: string; number: string }; neighborhood: { id: string; name: string };
  installed_on: string; collected_on: string | null; condition: ReadingCondition | (string & {}) | null; epi_week: string;
  egg_count: number | null; counted_at: string | null; corrected: boolean;
}
export type VectorTaskKind = "ovitrap_intensification" | "case_resident_elsewhere" | "block_case_without_place" | (string & {});
export interface VectorTask {
  id: string; kind: VectorTaskKind; state: "open" | "done" | "dismissed" | (string & {}); created_at: string;
  ovitrap: { id: string; number: string } | null; suggested_blocks: { id: string; number: string }[];
  disease: string | null; case_on: string | null; transmission_block_id: string | null; closed_at: string | null;
}
export type StratumMode = "standard" | "half" | "census";
export interface LiraaStratum {
  id: string; name: string; mode: StratumMode; block_ids: string[]; households_count: number | null; sample_target: number | null;
  block_interval: number | null; random_start: number | null;
}
export interface LiraaSurvey {
  id: string; name: string; starts_on: string; ends_on: string; status: "draft" | "drawn" | "closed" | (string & {});
  parameters_version: number | null; drawn_at: string | null; closed_at: string | null; strata: LiraaStratum[];
}
export interface LiraaSurveyInput { name: string; starts_on: string; ends_on: string }
export interface LiraaStratumInput { name: string; mode: StratumMode; block_ids: string[] }
export interface LiraaItem {
  id: string; stratum_id: string; city_block: { id: string; number: string }; household: { id: string; address_short: string }; position: number;
  status: "pending" | "inspected" | "substituted" | (string & {}); substitute_of_id: string | null;
}
export interface TransmissionCase { id: string; number: number; disease: string; case_on: string; has_place: boolean }
export interface TransmissionBlock {
  id: string; status: "proposed" | "active" | "closed" | "dismissed" | (string & {}); disease_group: string;
  center_block: { id: string; number: string } | null; area: { id: string; number: string; suggested: boolean }[];
  cases: TransmissionCase[]; cycles: { number: number; starts_on: string; ends_on: string }[];
  created_at: string; confirmed_at: string | null; closed_at: string | null;
}
export interface CaseAddress { patient_address: string | null; household_address: string | null }
export interface VectorSample {
  id: string; label: string; visit_on: string; deposit_group: DepositGroup; larvae_collected: number;
  city_block: { id: string; number: string } | null; result: SampleResult | null;
}
export type VectorShortfallCause = "no_location" | "insufficient_balance" | "no_lot" | (string & {});
export interface VectorShortfall {
  id: string; created_at: string; stock_location: { id: string; name: string } | null; product: VectorProductRef; quantity: string;
  cause: VectorShortfallCause; visit_on: string; reconciled_at: string | null; reconciled_by: { name: string | null } | null;
}
export type VectorReportName = "situations" | "strategic_points" | "production" | "consumption" | "ovitraps" | "liraa" | "blocks";
export type VectorReportFilters = Partial<{ cycle_id: string; neighborhood_ids: string[]; from: string; to: string; professional_ids: string[];
  epi_weeks: string[]; survey_id: string }>;
// D-13: uma forma para os sete relatórios; contagem pode vir suprimida (SmallCount).
export interface VectorReport {
  name: VectorReportName; provisional?: boolean;
  columns: { key: string; label: string; kind: "text" | "count" | "percent" | "decimal" }[];
  rows: Record<string, SmallCount | string | null>[]; notes?: string[];
}
export type VectorMapQuery =
  | { layer: "risk"; neighborhoodId: string } | { layer: "ovitraps"; neighborhoodId: string; epiWeek: string } | { layer: "transmission"; transmissionBlockId: string };
export interface VectorMapPoint { id: string; latitude: number; longitude: number; category: string; label: string }
export interface VectorMapResponse { map: FieldMapManifest | null; points: VectorMapPoint[] }
```

- Funções (todas `Promise`, listas `{ items, next_before }` com `before` opcional):
  - `getVectorDomains(): VectorControlDomains` → `GET /vector_control/domains` (D-2).
  - Quarteirões: `listCityBlocks(neighborhoodId: string): CityBlock[]` (`?neighborhood_id=`, envelope `{ items }`); `createCityBlock({ neighborhood_id, number })`; `updateCityBlock(id, { number?, active? })` (PATCH); `addBlockNeighbor(id, neighborId)` → `POST /blocks/:id/neighbors { neighbor_block_id }`; `removeBlockNeighbor(id, neighborId)` → `DELETE /blocks/:id/neighbors/:neighbor_id` com corpo `{}`; todas devolvem `CityBlock`.
  - Imóveis: `listVectorHouseholds(q: { neighborhoodId?: string; cityBlockId?: string; withoutBlock?: boolean; before?: string | null })` → `?neighborhood_id=&city_block_id=&without_block=true`; `assignHouseholdsBlock(householdIds: string[], cityBlockId: string): { assigned: number }` (corpo, nunca URL; até 200); `updateHouseholdProfile(id, { city_block_id?: string | null; risk_class?: RiskClass | null }): VectorHousehold`.
  - `getHouseholdVectorProfile(householdId: string): HouseholdVectorProfile` → `GET /vector_control/households/:id` (Q8, Q13(5): `{ id, neighborhood_id, vector }`, `vector: null` sem perfil; agente nas microáreas dele e coordenação).
  - `listDue(microAreaId: string, before?: string | null)` → `GET /vector_control/due?micro_area_id=`.
  - `listCadences(): VectorCadence[]`; `putCadences(items: VectorCadence[]): VectorCadence[]` (`{ items }`); `listCycles(year?: number): VectorCycle[]`; `createCycle(i: Omit<VectorCycle, "id">)`; `updateCycle(id, i)`; `getCycleSummary(id): CycleSummary` → `GET /cycles/:id` (D-6).
  - `getVectorParameters(): VectorParameterVersion`; `putVectorParameters(values: VectorParameters): VectorParameterVersion` (`{ values }`).
  - `listVectorProducts(): VectorProduct[]`; `createVectorProduct(i)`; `updateVectorProduct(id, Partial<VectorProductInput>)`; `getVectorSettings(): VectorSettings`; `putVectorSettings(stockLocationId: string | null): VectorSettings`.
  - Ovitrampas: `listOvitraps(q: { microAreaId?: string; neighborhoodId?: string; status?: OvitrapStatus; before? })` (D-7); `installOvitrap(i: { client_uuid: string; household_id: string; number: string; consent: boolean; installed_on: string }): Ovitrap` (a tela só manda `true`; `false` → `consent_required`, Q7(5)); `collectOvitrap(id, i: { collected_on: string; condition: ReadingCondition; reinstall: boolean; remove: boolean }): Ovitrap` (manda `reinstall: !remove`, Q7(5)); `listOvitrapReadings(q: { state: "to_count" | "counted"; before? })`; `countOvitrapReading(id, i: { egg_count: number; count_note?: string }): OvitrapReading`.
  - `listVectorTasks(q: { state: "open" | "closed"; before? })`; `closeVectorTask(id, i: { state: "done" | "dismissed"; note: string }): VectorTask`.
  - LIRAa: `listLiraaSurveys(): LiraaSurvey[]`; `createLiraaSurvey(i)`; `updateLiraaSurvey(id, i: Partial<LiraaSurveyInput> & { strata?: LiraaStratumInput[] })`; `drawLiraaSurvey(id)`; `closeLiraaSurvey(id)`; `listLiraaItems(id, before?)`.
  - Bloqueio: `listTransmissionBlocks(q: { status?: string[]; before? })`; `getTransmissionBlock(id)`; `confirmTransmissionBlock(id, i: { center_block_id: string; block_ids: string[] })`; `dismissTransmissionBlock(id, reason)`; `closeTransmissionBlock(id)`; `addTransmissionCycle(id, i: { starts_on: string; ends_on: string })`; `moveTransmissionCase(id, caseId, toBlockId)` (`{ to_transmission_block_id }`); `getTransmissionCaseAddress(id, caseId): CaseAddress` (todas devolvem `TransmissionBlock`, salvo o endereço).
  - `listVectorSamples(q: { state: "pending" | "done"; before? })`; `recordSampleResult(id, i: { examined_larvae: number; aegypti_larvae: number; albopictus_larvae: number; result_note?: string }): VectorSample`.
  - `listVectorShortfalls(q: { state: "open" | "reconciled"; locationId?: string | null; before? })`; `reconcileVectorShortfall(id, note): VectorShortfall`.
  - `getVectorReport(name, f: VectorReportFilters): VectorReport`; `fetchVectorReportFile(name, f, format: "csv" | "pdf"): SignatureFile` (`binaryFetch` em `/vector_control/reports/<name>.<format>`; `pdf` só com `liraa`, senão lança antes de chamar).
  - `listVectorMapPoints(q: VectorMapQuery): VectorMapResponse` → `GET /vector_control/map_points?layer=…` (D-5; ausentes = `{ map: null, points: [] }`).
- Extensões: `HomeVisitInput.vector_inspection?: VectorInspectionInput`; `HomeVisit.vector_inspection?: VectorInspection`; `StockItem` + `vector_product_id?: string | null`, `item_kind` + `"vector_control"`; `ReceiptItemInput` vira `({ catalog_item_id: string } | { vector_product_id: string }) & { lot_number; expires_on; manufacturer?; quantity; package? }`; `StockMovementKind` + `"vector_application" | "vector_application_reversal"` (Q10); `searchPharmacyItems(q: string, domain?: "vector_control")` manda `{ q, domain }` só com o domínio (D-8).
- `src/lib/features.ts`: `FeatureKey` + `"vector_control"`; `FEATURE_LABEL.vector_control = "Vigilância ambiental (endemias)"`.
- `src/devProxy.ts`: `MODULE_35A_PROXY_KEYS = [ "^/vector_control(/|\\?|$)" ]`; `vite.config.ts` espalha como o 24b.
- Fixtures (`src/test/vectorControlFixtures.ts`): `NOW35`, `TODAY35`, `VC_FEATURES = [ "clinical_record", "territory_registry", "home_visits", "vector_control" ]`, `vcUser(roles = [ "health_professional" ], features = VC_FEATURES)`, `agentTeam(o?)` (`acsTeam({ cbo: "515140", out_of_area_allowed: false, micro_areas: [ { id: "ma1", code: "01", name: "Vila Verde", scope: "assigned" } ] })`), `vcContext(o?)`, `LARVICIDE` (`{ id: "pr1", code: "PYR", name: "Pyriproxyfen 0,5% G", unit: "g", charge_size: "2.00", uses: [ "focal" ] }`), `ADULTICIDE` (`pr2`, `"ml"`, `[ "perifocal", "spatial" ]`), `INACTIVE_PRODUCT`, `VC_DOMAINS` (com `limits` da Q7(3)), `PARAMS` (o padrão do §11), `householdVector(o?)`, `householdProfile(o?)` (`{ id: "h1", neighborhood_id: "nb1", vector: householdVector() }`), `cityBlock(o?)`, `vectorHousehold(o?)`, `dueItem(o?)`, `inspectionInput(o?)` (o boletim completo: `li_t`, `routine`, `worked`, A2 `{ existing: 3, with_water: 2, with_larvae: 1, eliminated: 1, treated: 1 }`, focal `pr1` 2 cargas, amostra 1 A2 8 larvas, ação educativa, 09:40–10:20), `inspection(o?)`, `ovitrap(o?)`, `reading(o?)`, `sample(o?)`, `liraaSurvey(o?)`, `liraaItem(o?)`, `transmissionBlock(o?)`, `vectorTask(o?)`, `shortfall(o?)`, `report(o?)`, `mapPoints(o?)`.

- [ ] **Step 1: Testes que falham** — `src/lib/api.vectorControl.test.ts` (molde `src/lib/api.homeVisits.test.ts`: `stub(body, status?, headers?)`, `last()` = `{ url, init }`, `bodyOf()`; `HV_INPUT` = uma visita mínima do 24b, local ao teste):

```ts
it("visita leva vector_inspection no corpo; nada do boletim na URL", async () => {
  stub({ visit: visit(), day: day() }, 201);
  await recordHomeVisit({ ...HV_INPUT, vector_inspection: inspectionInput() });
  expect(last().url).toBe("/home_visits");
  expect(bodyOf().vector_inspection).toEqual(inspectionInput());
});
it("filtros: só ids, estado e cursor; listas como chave[]", async () => {
  stub({}); expect(await listDue("ma-1", "c1")).toEqual({ items: [], next_before: null });
  expect(last().url).toBe("/vector_control/due?micro_area_id=ma-1&before=c1");
  stub({}); await listVectorHouseholds({ neighborhoodId: "nb1", withoutBlock: true });
  expect(last().url).toBe("/vector_control/households?neighborhood_id=nb1&without_block=true");
  stub({ columns: [], rows: [] }); await getVectorReport("ovitraps", { epi_weeks: [ "2026-45", "2026-46" ], neighborhood_ids: [ "nb1" ] });
  expect(last().url).toBe("/vector_control/reports/ovitraps?epi_weeks%5B%5D=2026-45&epi_weeks%5B%5D=2026-46&neighborhood_ids%5B%5D=nb1");
});
it("atribuição em lote, nota, motivo e contagem só no corpo", async () => {
  stub({ assigned: 2 }); await assignHouseholdsBlock([ "h1", "h2" ], "cb1");
  expect([ last().url, last().init.method, bodyOf() ]).toEqual([ "/vector_control/households/assign_block", "POST", { household_ids: [ "h1", "h2" ], city_block_id: "cb1" } ]);
  stub(shortfall()); await reconcileVectorShortfall("sf1", "lote achado no depósito");
  expect(last().url).toBe("/vector_control/stock_shortfalls/sf1/reconcile");
  stub(transmissionBlock()); await dismissTransmissionBlock("tb1", "caso descartado pela vigilância");
  expect(bodyOf()).toEqual({ reason: "caso descartado pela vigilância" });
});
it("escrita sem campo manda {} (draw, close, remove neighbor)", async () => { /* drawLiraaSurvey, closeLiraaSurvey, closeTransmissionBlock, removeBlockNeighbor → corpo "{}" e Content-Type json */ });
it("arquivos: csv de qualquer relatório; pdf só do liraa, sem chamar o api nos outros", async () => {
  stubBlob(200, "a;b\r\n", 'attachment; filename="liraa-2026.csv"');
  expect((await fetchVectorReportFile("liraa", { survey_id: "s1" }, "csv")).filename).toBe("liraa-2026.csv");
  expect(last().url).toBe("/vector_control/reports/liraa.csv?survey_id=s1");
  await expect(fetchVectorReportFile("situations", {}, "pdf")).rejects.toThrow();
  expect(fetchCalls()).toBe(1);
});
it("mapa: layer e ids na URL; ausentes = { map: null, points: [] }", async () => { /* listVectorMapPoints({ layer: "ovitraps", neighborhoodId: "nb1", epiWeek: "2026-46" }) */ });
it("perfil do imóvel pela rota do 35a: { id, neighborhood_id, vector } (Q8, Q13(5))", async () => {
  stub(householdProfile()); expect(await getHouseholdVectorProfile("h1")).toEqual(householdProfile());
  expect(last().url).toBe("/vector_control/households/h1");
  stub(householdProfile({ vector: null })); expect((await getHouseholdVectorProfile("h2")).vector).toBeNull();
});
it("estoque: busca do domínio vector_control e entrada com vector_product_id (D-8)", async () => {
  stub({ items: [] }); await searchPharmacyItems("pyr", "vector_control");
  expect(bodyOf()).toEqual({ q: "pyr", domain: "vector_control" });
  stub({}); await searchPharmacyItems("dipirona");
  expect(bodyOf()).toEqual({ q: "dipirona" });
});
```

Em `src/lib/features.test.ts`: `featureLabel("vector_control")` = `"Vigilância ambiental (endemias)"`. Em `src/devProxy.test.ts`: as chaves do 35a casam `/vector_control`, `/vector_control?x=1`, `/vector_control/reports/liraa.pdf`, e não `/vector_controlX` nem `/dashboard/vector_control`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod35a src/lib/api.vectorControl.test.ts src/lib/features.test.ts src/devProxy.test.ts`. Expected: FAIL (funções inexistentes).
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS; `tsc` limpo.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.vectorControl.test.ts src/lib/features.ts src/lib/features.test.ts src/devProxy.ts src/devProxy.test.ts vite.config.ts src/test/vectorControlFixtures.ts
$G commit -m "feat: add the vector control API client, visit inspection types and dev proxy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Frases de todos os códigos e regras puras do boletim (§3)

O campo copia as duas coisas (contrato §14: "as frases de recusa são as do dashboard"). Nada de React aqui; nenhum import de tela.

**Files:**
- Create: `src/lib/vectorErrors.ts`, `src/lib/vectorErrors.test.ts`, `src/lib/vectorInspection.ts`, `src/lib/vectorInspection.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `VisitOutcome` (24b); `cityIsoDate`, `parseCityLocal`, `cityLocalIso` (`src/lib/format.ts`).
- Produces (`src/lib/vectorErrors.ts`):
  - `VECTOR_CONTROL_DISABLED = "a vigilância ambiental (endemias) está desligada nesta cidade"`.
  - `VECTOR_ERROR_CODES` (readonly, 38: os 33 do §17 + `sample_label_taken` (Q7), `unit_has_vector_control_location` (Q10), `cbo_not_allowed`, `feature_disabled` e `invalid_transition` (Q9)) e `VECTOR_ERRORS: Record<VectorErrorCode, string>` — **exatamente** esta tabela:

| Código | Frase |
|---|---|
| `vector_inspection_invalid` | confira o campo destacado do boletim |
| `situation_outcome_mismatch` | a situação do imóvel não combina com o desfecho: trabalhado é visita realizada, recusado é recusada, fechado e desocupado são ausente |
| `inspection_not_worked` | imóvel fechado, recusado ou desocupado não leva depósito, medida, amostra nem ação educativa |
| `activity_property_mismatch` | esta atividade não vale para este tipo de imóvel: PE só em ponto estratégico e IE só em imóvel especial |
| `liraa_item_not_found` | este imóvel não tem item do LIRAa pendente em levantamento sorteado |
| `transmission_block_not_active` | o bloqueio não está ativo ou o quarteirão do imóvel não está na área dele |
| `deposit_counts_inconsistent` | contagem de depósitos incoerente: com água até os existentes, com larva até os com água, eliminados até os existentes e tratados até os com água |
| `product_not_allowed` | este produto está inativo ou não serve para este método |
| `samples_exceed_larvae` | amostras incoerentes: numere de 1 em diante sem pular, só em grupo com larva e no máximo um tubito por depósito com larva |
| `inspection_times_invalid` | horários inválidos: entrada e saída no dia da visita, a entrada até a hora da visita, a saída depois dela e no máximo 12 horas entre as duas |
| `reason_derived` | os motivos de controle vetorial (34 a 37) saem do boletim — desmarque-os |
| `block_already_set` | o imóvel já tem quarteirão — só a coordenação de endemias troca |
| `household_without_neighborhood` | o domicílio está sem bairro no cadastro — o ACS corrige o endereço no cadastro territorial |
| `block_not_in_household_neighborhood` | o quarteirão é de outro bairro — escolha um do bairro do domicílio |
| `household_without_block` | o imóvel ainda não tem quarteirão — a ovitrampa só é instalada em imóvel com quarteirão |
| `consent_required` | a instalação exige o termo de consentimento assinado pelo morador |
| `ovitrap_number_taken` | já existe ovitrampa ativa com este número |
| `collection_out_of_range` | a coleta está fora do prazo da palheta — confira as datas de instalação e de coleta |
| `reading_not_open` | esta ovitrampa não tem palheta instalada para coletar |
| `reading_not_countable` | só se contam ovos de palheta coletada em bom estado |
| `ovitrap_not_active` | esta ovitrampa não está mais ativa |
| `invalid_neighbor` | vizinho inválido — um quarteirão não é vizinho dele mesmo |
| `block_in_use` | o quarteirão tem imóvel ativo — atribua os imóveis a outro quarteirão antes de desativar |
| `product_in_use` | unidade e tamanho da carga não mudam depois do primeiro lote ou aplicação — cadastre outro produto |
| `already_reconciled` | esta falta já foi conciliada |
| `stratum_overlap` | este quarteirão já está em outro estrato deste levantamento |
| `stratum_out_of_range` | o número de imóveis do estrato está fora da faixa do tipo escolhido |
| `survey_not_draft` | o levantamento já foi sorteado — os estratos não mudam mais |
| `cycle_invalid` | ciclo inválido — confira as datas, a duração e o intervalo desde o ciclo anterior |
| `cycle_overlap` | este período se sobrepõe a outro ciclo |
| `invalid_parameter` | valor fora da faixa aceita para este parâmetro |
| `sample_result_invalid` | resultado incoerente: examinadas até as coletadas, e Ae. aegypti mais Ae. albopictus até as examinadas |
| `block_number_taken` | já existe quarteirão com este número neste bairro |
| `cbo_not_allowed` | só o agente de combate às endemias registra boletim e ovitrampa |
| `feature_disabled` | (= `VECTOR_CONTROL_DISABLED`) |
| `sample_label_taken` | esta etiqueta já foi usada — gere etiquetas novas para este boletim |
| `unit_has_vector_control_location` | esta unidade tem local de estoque do controle vetorial e não troca de tipo |
| `invalid_transition` | a situação mudou enquanto a tela estava aberta — os dados foram relidos |

- Produces (`src/lib/vectorInspection.ts`):
  - `ENDEMIC_AGENT_CBO = "515140"`; `DERIVED_REASONS = [ 34, 35, 36, 37 ] as const`; `PE_PROPERTY = "4"`; `IE_PROPERTIES = [ "5", …, "13" ]`.
  - `type InspectionIssue = { error: VectorErrorCode; path: string }`.
  - `type InspectionCtx = { visit: { outcome: VisitOutcome; occurred_at: string; reasons: number[] }; household: { property_type: string; neighborhood_id: string | null } | null; property_type: string; profile: { city_block_id: string | null } | null; agent: boolean; gateUsable: boolean; products: VectorControlDomains["products"]; limits: VectorControlDomains["limits"]; blocks?: { id: string; neighborhood_id: string; active: boolean }[]; liraaItems?: { id: string; household_id: string; status: string }[]; transmissionBlocks?: { id: string; status: string; block_ids: string[] }[] }` (opcionais ausentes = a regra só confere a presença do vínculo; o api confere o resto).
  - `checkInspection(input: VectorInspectionInput, ctx: InspectionCtx): InspectionIssue[]` — **todas** as recusas de uma vez, nesta ordem e com estes `path` (tabela de `path` abaixo, fonte única, Q9): 1 `cbo_not_allowed`/`feature_disabled` em `vector_inspection` (e para aí); sem domicílio → `vector_inspection_invalid` em `household_id` (e para aí, Q10); 2 `situation_outcome_mismatch` em `vector_inspection.situation`; `inspection_not_worked` em `vector_inspection.<deposits|measures|samples|educational_action>` (um por campo não vazio); 3 `activity_property_mismatch` em `vector_inspection.activity`; 4 `liraa_item_not_found` em `vector_inspection.liraa_draw_item_id`, `transmission_block_not_active` em `vector_inspection.transmission_block_id`, vínculo em atividade sem vínculo → `vector_inspection_invalid` no campo; 5 `deposit_counts_inconsistent` em `vector_inspection.deposits[i].<with_water|with_larvae|eliminated|treated|group>`, contagem fora de 0–`deposit_count_max` ou não inteira → `vector_inspection_invalid` no campo; mais de 7 grupos → `vector_inspection_invalid` em `vector_inspection.deposits`; 6 `product_not_allowed` em `vector_inspection.measures[i].vector_product_id`, mecânico com produto ou cargas e cargas fora de (0, `charges_max`] ou com mais de 2 casas → `vector_inspection_invalid` em `…measures[i].<vector_product_id|charges>`; medida repetida (método, produto) → `vector_inspection_invalid` em `…measures[i]`; mais de `measures_max` → `vector_inspection_invalid` em `vector_inspection.measures`; 7 `samples_exceed_larvae` em `vector_inspection.samples[i]`, larvas fora de 1–`larvae_collected_max` → `vector_inspection_invalid` em `…samples[i].larvae_collected`; mais de `samples_max` → `vector_inspection_invalid` em `vector_inspection.samples`; 8 `inspection_times_invalid` em `vector_inspection.entered_at` (só um dos dois, ordem, 12 h, outro dia — o dia é `cityIsoDate` da visita); 9 `reason_derived` em `reasons`; 10 `block_already_set` / `household_without_neighborhood` / `block_not_in_household_neighborhood` em `vector_inspection.city_block_id`. `sample_label_taken` é do api (índice único), em `vector_inspection.samples[i]`.
  - `derivedReasons(input): number[]` (34 se `educational_action`; 35 se algum `with_larvae > 0`; 36 se algum `eliminated > 0` ou medida `mechanical`; 37 se medida `focal`; crescente, sem repetir).
  - `outcomeForSituation(s: VectorSituation): VisitOutcome` (`worked` 1, `refused` 2, `closed`/`vacant` 3).
  - `sampleLabel(clientUuid: string, n: number): string` → `clientUuid.slice(0, 8).toUpperCase() + "-" + String(n).padStart(2, "0")`.
  - `measureQuantity(charges: number, chargeSize: string): string` (`charges × charge_size` com **teto** em 2 casas, mínimo `0.01`, texto com ponto — a mesma conta do api, Q10).
  - `InspectionDraft = { activity: VectorActivity | null; workType: VectorWorkType; situation: VectorSituation | null; educationalAction: boolean; enteredAt: string; leftAt: string; cityBlockId: string | null; riskClass: RiskClass | null; liraaDrawItemId: string | null; transmissionBlockId: string | null; deposits: ({ group: DepositGroup } & Record<"existing" | "with_water" | "with_larvae" | "eliminated" | "treated", string>)[]; measures: { method: MeasureMethod; productId: string | null; charges: string }[]; samples: { group: DepositGroup; larvae: string }[] }` (números em texto, horários `"HH:MM"`, a numeração das amostras é a posição), `emptyInspectionDraft(start?: VectorStart)`, `inspectionInput(draft, day: string): VectorInspectionInput` (texto → número com vírgula aceita; horário vazio não vai; vínculo só na atividade dele), `pruneInspection(draft): { draft; cleared: string[] }` (situação ≠ `worked` tira depósitos, medidas, amostras e ação educativa; grupo sem larva perde as amostras dele; método mecânico perde produto e cargas; `cleared` com os rótulos em português).
  - `export type VectorStart = { activity?: VectorActivity; workType?: VectorWorkType; liraaDrawItemId?: string | null; transmissionBlockId?: string | null }`.

**Tabela de `path` (fonte única para api, contracts e campo — Q9).** `i` é o índice na lista enviada; "—" = sem `path`.

| Código | Onde | `path` |
|---|---|---|
| `cbo_not_allowed`, `feature_disabled` | boletim | `vector_inspection` |
| `cbo_not_allowed`, `feature_disabled` | ovitrampa (dashboard e item do app) | — |
| `vector_inspection_invalid` | boletim sem domicílio (Q10) | `household_id` |
| `vector_inspection_invalid` | forma do boletim (Q7(7)), vínculo fora da atividade, contagem, carga, larvas, medida repetida, listas acima do limite | o campo: `vector_inspection.<campo>`, `…deposits[i].<campo>`, `…measures[i].<vector_product_id\|charges>`, `…measures[i]`, `…samples[i].larvae_collected`, `…deposits`/`…measures`/`…samples` |
| `situation_outcome_mismatch` | §3.2 | `vector_inspection.situation` |
| `inspection_not_worked` | §3.2 | `vector_inspection.<deposits\|measures\|samples\|educational_action>`, um por campo não vazio |
| `activity_property_mismatch` | §3.3 | `vector_inspection.activity` |
| `liraa_item_not_found` | §3.4 | `vector_inspection.liraa_draw_item_id` |
| `transmission_block_not_active` | §3.4 | `vector_inspection.transmission_block_id` |
| `deposit_counts_inconsistent` | §3.5 | `vector_inspection.deposits[i].<with_water\|with_larvae\|eliminated\|treated\|group>` |
| `product_not_allowed` | §3.6 | `vector_inspection.measures[i].vector_product_id` |
| `samples_exceed_larvae` | §3.7 | `vector_inspection.samples[i]` |
| `sample_label_taken` | §9 (Q7(1)) | `vector_inspection.samples[i]` |
| `inspection_times_invalid` | §3.8 | `vector_inspection.entered_at` |
| `reason_derived` | §3.9 | `reasons` |
| erro do 24b sobre motivo derivado (Q10) | §3.9 | `subject` |
| `block_already_set`, `household_without_neighborhood`, `block_not_in_household_neighborhood` | §3.10 (boletim) | `vector_inspection.city_block_id` |
| `household_without_neighborhood`, `block_not_in_household_neighborhood` | §2 (lote da coordenação) | `household_ids[i]` |
| `household_without_neighborhood`, `block_not_in_household_neighborhood` | §2 (perfil de um imóvel) | `city_block_id` |
| `household_without_block` | §6 instalação | `household_id` (Q13(2)) |
| `consent_required` | §6 instalação | `consent` |
| `ovitrap_number_taken` | §6 instalação | `number` |
| `collection_out_of_range` | §6 coleta | `collected_on` |
| `reading_not_open`, `ovitrap_not_active` | §6 coleta | `ovitrap_id` (Q13(2)) |
| `reading_not_countable` | §6 contagem | — |
| nota ausente (Q9) | correção de contagem, de resultado, tarefa | `count_note`, `result_note`, `note` (`422 invalid_request`) |
| `invalid_transition` | toda transição fora do estado (Q9), inclusive mover caso para bloqueio fechado (Q13(9)) | `status` |
| `sample_label_taken` é 422 (Q13(3)) | | |

**Ordem (Q13(10)):** o api devolve só o primeiro erro, como no 24b; `checkInspection` devolve todos, mas o primeiro da lista é o mesmo que o api daria (a ordem é a das regras 1–10 acima). Teste: `issues(...)[0]` nos casos com mais de uma recusa.

- [ ] **Step 1: Testes que falham**

`src/lib/vectorErrors.test.ts`:

```ts
it("cada código do contrato tem uma frase, e só uma por código", () => {
  expect(Object.keys(VECTOR_ERRORS).sort()).toEqual([ ...VECTOR_ERROR_CODES ].sort());
  const phrases = Object.values(VECTOR_ERRORS);
  expect(new Set(phrases).size).toBe(phrases.length);
  for (const p of phrases) expect(p.length).toBeGreaterThan(10);
});
it("frases que o campo copia, sem dado de pessoa nem código cru", () => {
  expect(VECTOR_ERRORS.reason_derived).toBe("os motivos de controle vetorial (34 a 37) saem do boletim — desmarque-os");
  expect(VECTOR_ERRORS.feature_disabled).toBe(VECTOR_CONTROL_DISABLED);
  expect(VECTOR_ERRORS.sample_label_taken).toBe("esta etiqueta já foi usada — gere etiquetas novas para este boletim");
  expect(VECTOR_ERRORS.invalid_transition).toBe("a situação mudou enquanto a tela estava aberta — os dados foram relidos");
  for (const p of Object.values(VECTOR_ERRORS)) expect(p).not.toMatch(/_/);
});
```

`src/lib/vectorInspection.test.ts` (`ctx(o)` = visita realizada às 10:00 de `TODAY35`, imóvel `"1"` no bairro `nb1`, perfil com `cb1`, agente, gate ligado, produtos de `VC_DOMAINS` mais `INACTIVE_PRODUCT` com `active: false`, `limits` de `VC_DOMAINS`; `ZERO_DEPOSIT`, `GROUPS` locais ao teste; `base = ctx()`; `issues(i, c)` = `checkInspection(i, ctx(c))`; `fullDraft()` = o rascunho do boletim completo, local ao teste):

```ts
it("boletim completo do agente: nada a recusar", () => expect(issues(inspectionInput())).toEqual([]));
it("1. outro CBO ou gate desligado param tudo", () => {
  expect(issues(inspectionInput(), { agent: false })).toEqual([ { error: "cbo_not_allowed", path: "vector_inspection" } ]);
  expect(issues(inspectionInput(), { gateUsable: false })).toEqual([ { error: "feature_disabled", path: "vector_inspection" } ]);
});
it.each([ [ "worked", 2 ], [ "worked", 3 ], [ "refused", 1 ], [ "closed", 1 ], [ "vacant", 2 ] ] as const)(
  "2. situação %s com desfecho %i → situation_outcome_mismatch", (situation, outcome) => {
  expect(issues(inspectionInput({ situation }), { visit: { ...base.visit, outcome } })).toContainEqual({ error: "situation_outcome_mismatch", path: "vector_inspection.situation" });
});
it("2. fechado com depósito, medida, amostra e ação educativa: uma recusa por campo", () => {
  expect(issues(inspectionInput({ situation: "closed" }), { visit: { ...base.visit, outcome: 3 } }).filter((p) => p.error === "inspection_not_worked").map((p) => p.path))
    .toEqual([ "vector_inspection.deposits", "vector_inspection.measures", "vector_inspection.samples", "vector_inspection.educational_action" ]);
});
it.each([ [ "pe", "1", true ], [ "pe", "4", false ], [ "ie", "4", true ], [ "ie", "5", false ], [ "ie", "13", false ], [ "li", "99", false ], [ "li_t", "4", false ] ] as const)(
  "3. atividade %s em imóvel %s recusa = %s", (activity, property_type, refused) => { /* toContainEqual / not.toContainEqual activity_property_mismatch */ });
it("4. vínculos: liraa sem item, item de outro imóvel, bloqueio inativo ou fora da área, vínculo em li", () => {
  expect(issues(inspectionInput({ activity: "liraa" }))).toContainEqual({ error: "liraa_item_not_found", path: "vector_inspection.liraa_draw_item_id" });
  expect(issues(inspectionInput({ activity: "liraa", liraa_draw_item_id: "li1" }), { liraaItems: [ { id: "li1", household_id: "h2", status: "pending" } ] }))
    .toContainEqual({ error: "liraa_item_not_found", path: "vector_inspection.liraa_draw_item_id" });
  expect(issues(inspectionInput({ activity: "transmission_block", transmission_block_id: "tb1" }), { transmissionBlocks: [ { id: "tb1", status: "active", block_ids: [ "cb9" ] } ] }))
    .toContainEqual({ error: "transmission_block_not_active", path: "vector_inspection.transmission_block_id" });
  expect(issues(inspectionInput({ activity: "li", liraa_draw_item_id: "li1" }))).toContainEqual({ error: "vector_inspection_invalid", path: "vector_inspection.liraa_draw_item_id" });
});
it("5. depósitos: grupo repetido e cada desigualdade no próprio campo", () => {
  const bad = inspectionInput({ deposits: [ { group: "A1", existing: 2, with_water: 3, with_larvae: 4, eliminated: 3, treated: 4 }, { group: "A1", existing: 0, with_water: 0, with_larvae: 0, eliminated: 0, treated: 0 } ], samples: [] });
  const found = issues(bad).filter((p) => p.error === "deposit_counts_inconsistent").map((p) => p.path);
  expect(found).toEqual([ "vector_inspection.deposits[0].with_water", "vector_inspection.deposits[0].with_larvae", "vector_inspection.deposits[0].eliminated",
    "vector_inspection.deposits[0].treated", "vector_inspection.deposits[1].group" ]);
  expect(issues(inspectionInput({ deposits: [ { group: "B", existing: 1000, with_water: 0, with_larvae: 0, eliminated: 0, treated: 0 } ], samples: [] })))
    .toContainEqual({ error: "vector_inspection_invalid", path: "vector_inspection.deposits[0].existing" });
});
it("6. medidas: mecânico sem produto; inativo; uso errado; cargas 0, 999,01 e 3 casas", () => {
  expect(issues(inspectionInput({ measures: [ { method: "mechanical", vector_product_id: "pr1", charges: 1 } ] })).map((p) => p.path))
    .toEqual(expect.arrayContaining([ "vector_inspection.measures[0].vector_product_id", "vector_inspection.measures[0].charges" ]));
  expect(issues(inspectionInput({ measures: [ { method: "spatial", vector_product_id: "pr1", charges: 1 } ] })))
    .toContainEqual({ error: "product_not_allowed", path: "vector_inspection.measures[0].vector_product_id" });
  expect(issues(inspectionInput({ measures: [ { method: "focal", vector_product_id: INACTIVE_PRODUCT.id, charges: 1 } ] })))
    .toContainEqual({ error: "product_not_allowed", path: "vector_inspection.measures[0].vector_product_id" });
  for (const charges of [ 0, 999.01, 1.005 ]) expect(issues(inspectionInput({ measures: [ { method: "focal", vector_product_id: "pr1", charges } ] })))
    .toContainEqual({ error: "vector_inspection_invalid", path: "vector_inspection.measures[0].charges" });
  expect(issues(inspectionInput({ measures: [ { method: "focal", vector_product_id: "pr1", charges: 999 } ] }))).toEqual([]);
});
it("7. amostras: buraco na numeração, grupo sem larva, mais tubitos que depósitos com larva", () => {
  const one = { number: 1, deposit_group: "A2" as const, larvae_collected: 8 };
  expect(issues(inspectionInput({ samples: [ { ...one, number: 2 } ] }))).toContainEqual({ error: "samples_exceed_larvae", path: "vector_inspection.samples[0]" });
  expect(issues(inspectionInput({ samples: [ { ...one, deposit_group: "B" } ] }))).toContainEqual({ error: "samples_exceed_larvae", path: "vector_inspection.samples[0]" });
  expect(issues(inspectionInput({ samples: [ one, { ...one, number: 2 } ] }))).toContainEqual({ error: "samples_exceed_larvae", path: "vector_inspection.samples[1]" });
  expect(issues(inspectionInput({ samples: [ { ...one, larvae_collected: 11 } ] }))).toContainEqual({ error: "vector_inspection_invalid", path: "vector_inspection.samples[0].larvae_collected" });
});
it("8. horários: um sem o outro; entrada depois da visita; 12 h 01; outro dia; os dois omitidos valem", () => {
  const t = (entered_at?: string, left_at?: string) => issues(inspectionInput({ entered_at, left_at }));
  const bad = { error: "inspection_times_invalid", path: "vector_inspection.entered_at" };
  expect(t("2026-11-16T09:40:00-03:00", undefined)).toContainEqual(bad);
  expect(t("2026-11-16T10:05:00-03:00", "2026-11-16T10:20:00-03:00")).toContainEqual(bad);
  expect(t("2026-11-16T00:00:00-03:00", "2026-11-16T12:01:00-03:00")).toContainEqual(bad);
  expect(t("2026-11-15T23:50:00-03:00", "2026-11-16T10:20:00-03:00")).toContainEqual(bad);
  expect(t(undefined, undefined)).toEqual([]);
  expect(t("2026-11-16T00:00:00-03:00", "2026-11-16T12:00:00-03:00")).toEqual([]);
});
it("9. 34–37 no corpo com boletim → reason_derived; derivados em ordem crescente", () => {
  expect(issues(inspectionInput(), { visit: { ...base.visit, reasons: [ 1, 35 ] } })).toContainEqual({ error: "reason_derived", path: "reasons" });
  expect(derivedReasons(inspectionInput())).toEqual([ 34, 35, 36, 37 ]);
  expect(derivedReasons(inspectionInput({ educational_action: false, deposits: [], samples: [], measures: [ { method: "mechanical" } ] }))).toEqual([ 36 ]);
  expect(derivedReasons(inspectionInput({ situation: "closed", educational_action: false, deposits: [], measures: [], samples: [] }))).toEqual([]);
});
it("10. quarteirão: perfil já tem → block_already_set; domicílio sem bairro; quarteirão de outro bairro", () => {
  const p = { error: "block_already_set", path: "vector_inspection.city_block_id" };
  expect(issues(inspectionInput({ city_block_id: "cb2" }))).toContainEqual(p);
  const noBlock = { profile: { city_block_id: null }, blocks: [ { id: "cb2", neighborhood_id: "nb2", active: true } ] };
  expect(issues(inspectionInput({ city_block_id: "cb2" }), noBlock)).toContainEqual({ error: "block_not_in_household_neighborhood", path: "vector_inspection.city_block_id" });
  expect(issues(inspectionInput({ city_block_id: "cb2" }), { ...noBlock, household: { property_type: "1", neighborhood_id: null } }))
    .toContainEqual({ error: "household_without_neighborhood", path: "vector_inspection.city_block_id" });
});
it("primeiro erro = o do api (Q13(10)): situação errada e depósito incoerente → situation_outcome_mismatch primeiro", () => {
  const two = issues(inspectionInput({ deposits: [ { group: "A2", existing: 1, with_water: 2, with_larvae: 1, eliminated: 0, treated: 0 } ] }), { visit: { ...base.visit, outcome: 2 } });
  expect(two[0]).toEqual({ error: "situation_outcome_mismatch", path: "vector_inspection.situation" });
});
it("sem domicílio: vector_inspection_invalid em household_id, e mais nada (Q10)", () => {
  expect(issues(inspectionInput(), { household: null })).toEqual([ { error: "vector_inspection_invalid", path: "household_id" } ]);
});
it("limites da Q7(3): oito grupos, 21 medidas e 100 amostras recusam a lista", () => {
  expect(issues(inspectionInput({ deposits: Array.from({ length: 8 }, (_, i) => ({ ...ZERO_DEPOSIT, group: GROUPS[i % 7] })), samples: [] })))
    .toContainEqual({ error: "vector_inspection_invalid", path: "vector_inspection.deposits" });
  expect(issues(inspectionInput({ measures: Array.from({ length: 21 }, () => ({ method: "mechanical" as const })) })))
    .toContainEqual({ error: "vector_inspection_invalid", path: "vector_inspection.measures" });
});
it("etiqueta, desfecho e quantidade", () => {
  expect(sampleLabel("3f2a9c1e-0b4d-4c1a-9e7f-1a2b3c4d5e6f", 1)).toBe("3F2A9C1E-01");
  expect(sampleLabel("3f2a9c1e-0b4d-4c1a-9e7f-1a2b3c4d5e6f", 12)).toBe("3F2A9C1E-12");
  expect([ outcomeForSituation("worked"), outcomeForSituation("refused"), outcomeForSituation("closed"), outcomeForSituation("vacant") ]).toEqual([ 1, 2, 3, 3 ]);
  expect(measureQuantity(2.5, "2.00")).toBe("5.00");
  expect(measureQuantity(1.25, "0.33")).toBe("0.42");   // 0,4125 com teto em 2 casas (Q10)
  expect(measureQuantity(0.01, "0.10")).toBe("0.01");   // mínimo 0,01
});
it("rascunho: vírgula vira número; horário vazio não vai; vínculo só na atividade dele; poda ao fechar", () => {
  const d = { ...emptyInspectionDraft(), activity: "li_t", situation: "worked", enteredAt: "", leftAt: "",
    measures: [ { method: "focal", productId: "pr1", charges: "1,5" } ], liraaDrawItemId: "li1" };
  expect(inspectionInput(d, TODAY35)).toMatchObject({ measures: [ { method: "focal", vector_product_id: "pr1", charges: 1.5 } ] });
  expect(inspectionInput(d, TODAY35)).not.toHaveProperty("entered_at");
  expect(inspectionInput(d, TODAY35)).not.toHaveProperty("liraa_draw_item_id");
  const out = pruneInspection({ ...fullDraft(), situation: "closed" });
  expect(out.draft.deposits).toEqual([]); expect(out.draft.samples).toEqual([]); expect(out.draft.educationalAction).toBe(false);
  expect(out.cleared).toEqual([ "depósitos", "medidas", "amostras", "ação educativa" ]);
});
```

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod35a src/lib/vectorErrors.test.ts src/lib/vectorInspection.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces. "Dia da visita" = `cityIsoDate(new Date(occurred_at))`; 12 h = 43 200 000 ms; duas casas = `Math.round(x * 100) === x * 100` com tolerância de ponto flutuante (`Math.abs(x * 100 - Math.round(x * 100)) < 1e-9`).
- [ ] **Step 4: Rode e veja passar; `tsc`.**
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/vectorErrors.ts src/lib/vectorErrors.test.ts src/lib/vectorInspection.ts src/lib/vectorInspection.test.ts
$G commit -m "feat: port the field inspection rules and the vector control error phrases as pure modules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Portões, papéis, rótulos e as demais regras de tela

**Files:**
- Create: `src/lib/vectorControl.ts`, `src/lib/vectorControl.test.ts`
- Modify: `src/lib/homeVisits.ts` (`disabledText` + `vector_control`; `HOME_VISIT_ERRORS` espalha as frases do boletim; `cbo_not_allowed` com `path` `vector_inspection…` usa a frase do 35a), `src/lib/homeVisits.test.ts`

**Interfaces:**
- Consumes: `VECTOR_ERRORS`, `VECTOR_CONTROL_DISABLED`, `ENDEMIC_AGENT_CBO` (Task 2); `canUseHomeVisits`, `homeVisitRefusals`, `textProblem`, `STATIC_QUERY`, `NOMINAL_QUERY` (24b); `hasFeature`.
- Produces (`src/lib/vectorControl.ts`):
  - `VC_NO_ROLE = "seu papel não permite usar a vigilância ambiental"`, `VC_NOT_AGENT = "só o agente de combate às endemias (CBO 515140) usa esta tela"`.
  - Portões: `canUseVectorControl(user)`; `vectorRoles(user): { coordinator; lab; reader; admin; professional; stockKeeper }` (`reader` = coordenação, laboratório ou `municipal_admin`); `isEndemicAgentTeam(team: HomeVisitTeam): boolean` (`cbo === ENDEMIC_AGENT_CBO && can_register`).
  - Chaves: `VC_DOMAINS_KEY`, `VC_BLOCKS_KEY`, `VC_HOUSEHOLDS_KEY`, `VC_PROFILE_KEY`, `VC_DUE_KEY`, `VC_CYCLES_KEY`, `VC_CYCLE_KEY`, `VC_CADENCES_KEY`, `VC_PARAMS_KEY`, `VC_PRODUCTS_KEY`, `VC_SETTINGS_KEY`, `VC_OVITRAPS_KEY`, `VC_READINGS_KEY`, `VC_TASKS_KEY`, `VC_SURVEYS_KEY`, `VC_ITEMS_KEY`, `VC_BLOCKS_TB_KEY`, `VC_BLOCK_TB_KEY`, `VC_SAMPLES_KEY`, `VC_SHORTFALLS_KEY`, `VC_REPORT_KEY`, `VC_MAP_KEY`; reexporta `STATIC_QUERY`, `NOMINAL_QUERY`.
  - Rótulos (código novo cru): `domainLabel(list, code)`; `riskTone(r)` (`green` ok, `yellow` warn, `red` down, `gray` neutral, `purple` accent); `dueReasonLabel` (`transmission_block` "bloqueio", `liraa` "LIRAa", `pending` "pendência", `due` "vencido"); `ovitrapStatusLabel` (ativa, retirada, perdida); `conditionLabel` (`ok` "em bom estado", `damaged` "avariada", `lost` "perdida", `dry` "seca"); `taskKindLabel` (`ovitrap_intensification` "Intensificar em torno da ovitrampa", `case_resident_elsewhere` "Caso de residente de outro município", `block_case_without_place` "Caso de bloqueio sem quarteirão"); `taskStateLabel`; `surveyStatusLabel` (rascunho, sorteado, fechado); `stratumModeLabel` (`standard` "Padrão (8.100 a 12.000 imóveis)", `half` "Reduzido (2.000 a 8.099)", `census` "Censitário (até 1.999)"); `blockStatusLabel` (proposto, ativo, fechado, descartado); `shortfallCauseLabel` (`no_location` "sem local de estoque configurado", `insufficient_balance` "saldo insuficiente", `no_lot` "sem lote do produto"); `itemStatusLabel` (pendente, inspecionado, substituído); `caseLabel(c)` → `"caso " + c.number`.
  - Regras: `blockNumberProblem(n): Problem | null` (`/^[0-9A-Za-z-]{1,10}$/`); `installProblems(d: { number; consent; installedOn; householdHasBlock: boolean | null }, today): { code: string | null; path; text }[]` (`consent_required` em `consent` com `false`; `household_without_block` em `household_id` quando `false`; número `^[0-9A-Za-z-]{1,12}$` em `number` (Q7(6)); data ≤ hoje em `installed_on`); `collectionProblems(d: { installedOn; collectedOn; condition }, p: { ovitrap_collection_days_min; ovitrap_collection_days_max }, today)` (`collection_out_of_range` em `collected_on` quando `collected_on − installed_on` sai de [min, max]; data futura; condição); `countProblems(d: { eggs: string; note: string }, corrected: boolean)` (0–9999 inteiro; nota 10–500 só na correção, em `count_note`); `sampleResultProblems(d: { examined; aegypti; albopictus; note }, s: { larvae_collected }, corrected)` (`sample_result_invalid`; nota 10–500 na correção, em `result_note`); `ubvCycleProblems(prev: { ends_on }[], d: { starts_on; ends_on }, p)` (`cycle_invalid` em `ends_on` quando a duração inclusiva sai de [`ubv_cycle_days_min`, `_max`], em `starts_on` quando `starts_on − prev.ends_on < ubv_cycle_interval_days`, em `cycles` quando já há `ubv_cycles_max`); `cycleProblems(d: { year; number; starts_on; ends_on })` (número 1–12, fim ≥ início, ano da data de início); `stratumRangeProblem(mode, n, p): string | null` (`stratum_out_of_range`; faixas do §7 com os limites dos parâmetros); `noteProblem(text, path)` (envolve `textProblem` 10–500).
  - Recusas: `vectorRefusals(err): HomeVisitRefusal[]` (o parser do 24b com a tabela `{ ...HOME_VISIT_ERRORS, ...VECTOR_ERRORS }`); `vectorError(err): string`; `isVectorStale(err)` (`invalid_transition`, `survey_not_draft`, `already_reconciled`, `not_found`).
- Em `src/lib/homeVisits.ts`: `homeVisitRefusals(err, table = HOME_VISIT_ERRORS)` (parâmetro novo, padrão o de hoje); `disabledText` devolve `VECTOR_CONTROL_DISABLED` para `feature: "vector_control"`; `HOME_VISIT_ERRORS` ganha as 15 frases do boletim (os 12 códigos de §3, os dois do §2 e `sample_label_taken`) e, no `one()`, `cbo_not_allowed` com `path` começando por `vector_inspection` usa `VECTOR_ERRORS.cbo_not_allowed` (o mesmo molde do `replaces_device_id`).

- [ ] **Step 1: Testes que falham** — `src/lib/vectorControl.test.ts`:

```ts
it("portão: exige vector_control e a visita domiciliar utilizável (Q1); operador nunca", () => {
  expect(canUseVectorControl(vcUser())).toBe(true);
  expect(canUseVectorControl(vcUser(undefined, [ "clinical_record", "territory_registry", "vector_control" ]))).toBe(false);
  expect(canUseVectorControl({ ...vcUser(), operator: true })).toBe(false);
});
it("papéis da cidade e o agente pela equipe do contexto", () => {
  expect(vectorRoles(vcUser([ "endemic_coordinator" ]))).toMatchObject({ coordinator: true, lab: false, reader: true });
  expect(vectorRoles(vcUser([ "municipal_admin" ]))).toMatchObject({ coordinator: false, reader: true, admin: true });
  expect(isEndemicAgentTeam(agentTeam())).toBe(true);
  expect(isEndemicAgentTeam(acsTeam())).toBe(false);
  expect(isEndemicAgentTeam(agentTeam({ can_register: false }))).toBe(false);
});
it.each([ [ "2026-11-02", "2026-11-04", true ], [ "2026-11-02", "2026-11-05", false ], [ "2026-11-02", "2026-11-12", false ], [ "2026-11-02", "2026-11-13", true ] ])(
  "coleta: instalada %s, coletada %s → fora do prazo = %s (bordas de 2 e 11 dias)", (installedOn, collectedOn, out) => {
  const p = collectionProblems({ installedOn, collectedOn, condition: "ok" }, PARAMS, TODAY35);
  expect(p.some((x) => x.code === "collection_out_of_range" && x.path === "collected_on")).toBe(out);
});
it("contagem: 0 vale; 10000, -1 e 3,5 não; correção exige nota", () => {
  expect(countProblems({ eggs: "0", note: "" }, false)).toEqual([]);
  for (const eggs of [ "10000", "-1", "3,5", "" ]) expect(countProblems({ eggs, note: "" }, false).map((p) => p.path)).toEqual([ "egg_count" ]);
  expect(countProblems({ eggs: "12", note: "curta" }, true).map((p) => p.path)).toEqual([ "count_note" ]);
});
it("resultado da amostra: examinadas ≤ coletadas; aegypti + albopictus ≤ examinadas", () => {
  expect(sampleResultProblems({ examined: "9", aegypti: "0", albopictus: "0", note: "" }, { larvae_collected: 8 }, false)[0].code).toBe("sample_result_invalid");
  expect(sampleResultProblems({ examined: "8", aegypti: "5", albopictus: "4", note: "" }, { larvae_collected: 8 }, false)[0].code).toBe("sample_result_invalid");
  expect(sampleResultProblems({ examined: "8", aegypti: "5", albopictus: "3", note: "" }, { larvae_collected: 8 }, false)).toEqual([]);
});
it("ciclo de UBV: intervalo de 4 dias recusa, 5 vale; duração 3 a 5; máximo 3", () => {
  const prev = [ { ends_on: "2026-11-10" } ];
  expect(ubvCycleProblems(prev, { starts_on: "2026-11-14", ends_on: "2026-11-16" }, PARAMS).map((p) => p.path)).toEqual([ "starts_on" ]);
  expect(ubvCycleProblems(prev, { starts_on: "2026-11-15", ends_on: "2026-11-17" }, PARAMS)).toEqual([]);
  expect(ubvCycleProblems(prev, { starts_on: "2026-11-15", ends_on: "2026-11-20" }, PARAMS).map((p) => p.path)).toEqual([ "ends_on" ]);
  expect(ubvCycleProblems([ ...prev, ...prev, ...prev ], { starts_on: "2026-12-01", ends_on: "2026-12-03" }, PARAMS).map((p) => p.path)).toContain("cycles");
});
it.each([ [ "half", 8099, null ], [ "standard", 8099, "x" ], [ "standard", 8100, null ], [ "standard", 12000, null ], [ "standard", 12001, "x" ],
  [ "census", 1999, null ], [ "census", 2000, "x" ], [ "half", 2000, null ], [ "half", 1999, "x" ] ] as const)(
  "estrato %s com N = %i", (mode, n, problem) => {
  expect(stratumRangeProblem(mode, n, PARAMS) === null).toBe(problem === null);
});
it("quarteirão: número de 1 a 10 letras, dígitos ou hífen", () => { /* "12", "12-A" ok; "", "12 A", "12345678901" → problema */ });
it("recusas: código do 35a tem a frase da tabela; 403 vector_control diz o interruptor; código novo cru", () => {
  expect(vectorError(new ApiError(409, { error: "survey_not_draft" }, "x"))).toBe(VECTOR_ERRORS.survey_not_draft);
  expect(isVectorStale(new ApiError(409, { error: "invalid_transition", path: "status" }, "x"))).toBe(true);
  expect(vectorError(new ApiError(403, { error: "feature_disabled", feature: "vector_control" }, "x"))).toBe(VECTOR_CONTROL_DISABLED);
  expect(vectorError(new ApiError(422, { error: "brand_new" }, "x"))).toBe("recusado pelo servidor (brand_new)");
  expect(vectorRefusals(new ApiError(422, { error: "deposit_counts_inconsistent", path: "vector_inspection.deposits[0].treated" }, "x"))[0].path)
    .toBe("vector_inspection.deposits[0].treated");
});
```

Em `src/lib/homeVisits.test.ts`: `homeVisitError(new ApiError(422, { error: "reason_derived", path: "reasons" }, "x"))` = `VECTOR_ERRORS.reason_derived`; `cbo_not_allowed` com `path: "vector_inspection"` = `VECTOR_ERRORS.cbo_not_allowed` e sem `path` continua "seu CBO não permite esta ação"; os testes do 24b seguem verdes.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod35a src/lib/vectorControl.test.ts src/lib/homeVisits.test.ts`. **Step 3: Implemente.** **Step 4: Rode e veja passar; `tsc`.**
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/vectorControl.ts src/lib/vectorControl.test.ts src/lib/homeVisits.ts src/lib/homeVisits.test.ts
$G commit -m "feat: add vector control gates, roles, labels and ovitrap, sample, cycle and stratum screen rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Grupo "Endemias", itens fora dele e o portão (F-35.1)

**Files:**
- Create: `src/shell/modules.vectorControl.test.ts`, `src/modules/vectorControl/VectorGate.tsx`, `src/modules/vectorControl/VectorGate.test.tsx`, cascas (`PageHeader` + `EmptyState` "Em construção") de `src/modules/{VectorBlocks,VectorCycles,VectorOvitraps,VectorSamples,VectorLiraa,VectorTransmission,VectorTasks,VectorReports,VectorSettings,FieldOvitraps,VectorStockShortfalls}.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts` (as contagens `NAV_GROUPS.length - N` sobem 1 onde o grupo some), `src/shell/catalog.ts`, `src/App.tsx`

**Interfaces:**
- Consumes: `canUseVectorControl`, `vectorRoles`, `VC_NO_ROLE`, `VECTOR_CONTROL_DISABLED` (Tasks 2–3).
- Produces:
  - `ModuleId` + `"vector-blocks" | "vector-cycles" | "vector-ovitraps" | "vector-samples" | "vector-liraa" | "vector-transmission" | "vector-tasks" | "vector-reports" | "vector-settings" | "field-ovitraps" | "vector-stock-shortfalls"`.
  - Grupo `{ label: "Endemias", items: [ vector-blocks "Quarteirões e imóveis" ▦, vector-cycles "Ciclos e cadência" ↻, vector-ovitraps "Ovitrampas" ◎, vector-samples "Amostras de larvas" ⚗, vector-liraa "LIRAa" ⚄, vector-transmission "Bloqueio de transmissão" ⛨, vector-tasks "Tarefas de endemias" ☑, vector-reports "Relatórios de endemias" ▤, vector-settings "Produtos e parâmetros" ⚙ ] }` logo **depois de "Vigilância"**; `field-ovitraps` "Ovitrampas de campo" ◎ no fim de "Visita e atividade coletiva"; `vector-stock-shortfalls` "Faltas de insumo de campo" ⚠ logo depois de `pharmacy-urgency-shortfalls` em "Farmácia".
  - Filtro em `navGroupsFor` (`vcOn = canUseVectorControl(user)`, `vc = vectorRoles(user)`): `vector-ovitraps`, `vector-samples` → `vcOn && (vc.coordinator || vc.lab)`; `vector-reports` → `vcOn && vc.reader`; os outros do grupo → `vcOn && vc.coordinator`; `field-ovitraps` → `vcOn && isProfessional` (a tela confere a equipe do agente); `vector-stock-shortfalls` → `vcOn && (vc.coordinator || (phOn && isStockKeeper))`.
  - `CATALOG.groups.Endemias = { icon: IconBug, description: "Quarteirões, ciclos, ovitrampas, LIRAa, bloqueio, amostras e relatórios do controle vetorial" }` e as 11 telas com ícone e descrição (ícones confirmados na Task 0).
  - `renderModule`: um `case` por tela.
  - `VectorGate({ title, sub, allow(roles), children(roles) })`: sem `canUseVectorControl` → `EmptyState` `VECTOR_CONTROL_DISABLED`, sem ler nada; operador ou `allow` falso → `VC_NO_ROLE`; senão `children`.

- [ ] **Step 1: Testes que falham** — `src/shell/modules.vectorControl.test.ts`:

```ts
const ids = (u: Parameters<typeof navGroupsFor>[0], label = "Endemias") => navGroupsFor(u).find((g) => g.label === label)?.items.map((i) => i.id) ?? [];
const ALL = [ "vector-blocks", "vector-cycles", "vector-ovitraps", "vector-samples", "vector-liraa", "vector-transmission", "vector-tasks", "vector-reports", "vector-settings" ];
it("coordenação vê o grupo inteiro; laboratório, ovitrampas, amostras e relatórios; admin, só relatórios", () => {
  expect(ids(vcUser([ "endemic_coordinator" ]))).toEqual(ALL);
  expect(ids(vcUser([ "vector_lab" ]))).toEqual([ "vector-ovitraps", "vector-samples", "vector-reports" ]);
  expect(ids(vcUser([ "municipal_admin" ]))).toEqual([ "vector-reports" ]);
});
it("sem vector_control ou sem a visita domiciliar o grupo some; operador nunca", () => {
  expect(ids(vcUser([ "endemic_coordinator" ], [ "clinical_record", "territory_registry", "home_visits" ]))).toEqual([]);
  expect(ids(vcUser([ "endemic_coordinator" ], [ "clinical_record", "territory_registry", "vector_control" ]))).toEqual([]);
});
it("profissional: Ovitrampas de campo na visita; almoxarife e coordenação: faltas de insumo na Farmácia", () => {
  expect(ids(vcUser([ "health_professional" ]), "Visita e atividade coletiva")).toContain("field-ovitraps");
  expect(ids(vcUser([ "stock_keeper" ], [ ...VC_FEATURES, "pharmacy" ]), "Farmácia")).toContain("vector-stock-shortfalls");
  expect(ids(vcUser([ "endemic_coordinator" ]), "Farmácia")).toEqual([ "vector-stock-shortfalls" ]);
  expect(ids(vcUser([ "pharmacist" ], [ ...VC_FEATURES, "pharmacy" ]), "Farmácia")).not.toContain("vector-stock-shortfalls");
});
it("Endemias vem logo depois de Vigilância, com ícone e descrição", () => {
  const labels = NAV_GROUPS.map((g) => g.label);
  expect(labels.indexOf("Endemias")).toBe(labels.indexOf("Vigilância") + 1);
  expect(CATALOG.groups.Endemias.description).toBe("Quarteirões, ciclos, ovitrampas, LIRAa, bloqueio, amostras e relatórios do controle vetorial");
});
```

`VectorGate.test.tsx`: sem o interruptor mostra `VECTOR_CONTROL_DISABLED` e não chama `children`; papel errado → `VC_NO_ROLE`; coordenação → `children` com os papéis.

- [ ] **Step 2: Rode e veja falhar.** **Step 3: Implemente** (ajuste as contagens de `modules.test.ts`: o grupo novo some para quem não tem o papel). **Step 4: Rode e veja passar** (`src/shell/modules.vectorControl.test.ts src/shell/modules.test.ts src/shell/catalog.test.ts src/modules/vectorControl/VectorGate.test.tsx src/App.navigation.test.tsx`); `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.vectorControl.test.ts src/shell/catalog.ts src/App.tsx src/modules/vectorControl/VectorGate.tsx src/modules/vectorControl/VectorGate.test.tsx src/modules/VectorBlocks.tsx src/modules/VectorCycles.tsx src/modules/VectorOvitraps.tsx src/modules/VectorSamples.tsx src/modules/VectorLiraa.tsx src/modules/VectorTransmission.tsx src/modules/VectorTasks.tsx src/modules/VectorReports.tsx src/modules/VectorSettings.tsx src/modules/FieldOvitraps.tsx src/modules/VectorStockShortfalls.tsx
$G commit -m "feat: add the endemics menu group, field ovitraps and field supply shortfalls entries behind the vector control gate" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Boletim na visita do agente — formulário e detalhe (F-35.2, F-35.3, F-35.8)

**Files:**
- Create: `src/modules/homeVisits/InspectionSection.tsx`, `src/modules/homeVisits/InspectionDetail.tsx`, `src/modules/homeVisits/VisitForm.vector.test.tsx`, `src/modules/homeVisits/VisitDetail.vector.test.tsx`
- Modify: `src/modules/homeVisits/VisitForm.tsx`, `src/modules/homeVisits/VisitDetail.tsx`, `src/modules/homeVisits/ReasonsField.tsx` (prop `hidden?: number[]`), `src/modules/homeVisits/useVisitHandoff.ts` (`VisitStart.vector?: VectorStart`, só em memória como o resto do atalho), `src/modules/HomeVisitDay.tsx` e `src/modules/HomeVisitsTeam.tsx` (passam `vectorDomains`)

**Interfaces:**
- Consumes: `checkInspection`, `derivedReasons`, `outcomeForSituation`, `sampleLabel`, `measureQuantity`, `InspectionDraft`, `emptyInspectionDraft`, `inspectionInput`, `pruneInspection`, `VectorStart`, `DERIVED_REASONS` (Task 2); `canUseVectorControl`, `isEndemicAgentTeam`, `riskTone`, `domainLabel`, `VC_DOMAINS_KEY`, `VC_BLOCKS_KEY`, `vectorRefusals` (Task 3); `getVectorDomains`, `listCityBlocks`, `getHouseholdVectorProfile` (Task 1); `useIdempotentSubmit().uuid`, `familyVisitNotice`, `REASONS_MAX_TEXT`, `prunedText` (24b).
- Produces:
  - `VisitForm` ganha `vectorDomains?: VectorControlDomains | null` (nulo = sem boletim) e lê `start?.vector` (`VisitStart.vector?: VectorStart`, desta task; a Task 6 o preenche). Com `vectorDomains`: caixa "Registrar boletim" (ligada); com ela, a `InspectionSection` aparece **depois de "Quem foi visitado"**, o fieldset "Desfecho" vira a linha `"Desfecho: <rótulo> (da situação do imóvel)"`, `ReasonsField` recebe `hidden={DERIVED_REASONS}` e abaixo dele a prévia `"O boletim acrescenta: 34 Ação educativa, 35 Imóvel com foco"` (`reasonLabel` do 24b); `familyVisitNotice` recebe o rascunho com os motivos + derivados; o envio confere `visitProblems` **e** `checkInspection` (convertidos em `Problem` com a frase de `VECTOR_ERRORS`), mais motivos + derivados ≤ `reasons_max` (`REASONS_MAX_TEXT`, `path` `reasons`); o corpo é `{ team_id, ...visitBody(draft), vector_inspection: inspectionInput(inspection, draft.day) }`; recusas com `path` `vector_inspection.*` caem no campo da seção (`isOwn` aceita o prefixo).
  - `InspectionSection({ draft, onChange, domains, household, blocks, uuid, disabled, aria, errNode })`, cartões empilhados: atividade (sem `liraa`/`transmission_block`, salvo com o vínculo do `start`; `pe` só em imóvel `"4"`, `ie` só em 5–13 — a lista oferecida já filtra); tipo de trabalho; situação (trocar passa por `pruneInspection` e a linha `role="status"` diz o que saiu); entrada e saída (`type="time"`, opcionais); depósitos ("Adicionar grupo" → cartão do grupo com os cinco contadores `inputMode="numeric"`; "Retirar grupo"); medidas (método, produto filtrado por `uses`, cargas, e ao lado `"= 5,00 g"` por `measureQuantity`); amostras (grupo só entre os com larva; nº automático 1..N; larvas; etiqueta `sampleLabel(uuid, n)` com `"(provisória até gravar)"`); ação educativa; quarteirão (só quando o perfil — `getHouseholdVectorProfile(householdId)`, chave `[VC_PROFILE_KEY, householdId]`, `STATIC_QUERY`, lido quando o domicílio é escolhido — não tem `vector?.city_block_id`: `<select>` dos ativos de `listCityBlocks(profile.neighborhood_id)`; o `neighborhood_id` do perfil também vai ao `checkInspection`; sem bairro, a frase `household_without_neighborhood` e nada de seletor); risco (`risk_classes` com `Tag tone={riskTone}`, opcional). Imóvel sem cadastro: o boletim não aparece e a caixa "Registrar boletim" diz `"o boletim exige um domicílio do cadastro territorial — cadastre o imóvel não residencial em Território da equipe"` (Q10, Q13(1)), com o botão "Cadastrar o imóvel no território" (`onGoToTerritory`, prop nova do `VisitForm`, passada pelo `HomeVisitDay` → `territory-team`). Erro do 24b sobre motivo derivado volta com `path` `subject` (Q10) e cai em "Quem foi visitado".
  - Depois do 201 com amostras: `VISIT_SAVED` + `"Etiquete os tubitos: 3F2A9C1E-01, 3F2A9C1E-02"` com os `label` de `visit.vector_inspection.samples` (a resposta), nunca os da prévia.
  - `InspectionDetail({ inspection, domains })`: atividade, trabalho, situação (+ `Tag` "recuperado" quando `recovered`), horários, quarteirão, risco, depósitos por grupo, medidas com produto e quantidade, amostras com etiqueta e resultado (`"aguardando resultado"` sem ele; `"corrigido"` quando `corrected`), ação educativa. `VisitDetail` a mostra quando `v.vector_inspection` existe e lê os rótulos de `getVectorDomains` (`STATIC_QUERY`); sem os domínios, os códigos crus.

- [ ] **Step 1: Testes que falham** — `src/modules/homeVisits/VisitForm.vector.test.tsx` (monta o `VisitForm` com `team={agentTeam()}`, `context={vcContext()}`, `vectorDomains={VC_DOMAINS}`, `now={NOW35}`; `vi.mock("../../lib/api", …)` com `fetchCurrentSession`, `getVisitHousehold` → `householdDetail()` do 24b, `getHouseholdVectorProfile` → `householdProfile()`, `listCityBlocks` → `[ cityBlock() ]`, `recordHomeVisit`):

```tsx
it("boletim completo: corpo com vector_inspection, desfecho da situação, sem 34–37 nos motivos", async () => {
  vi.mocked(api.recordHomeVisit).mockResolvedValue({ visit: visit({ vector_inspection: inspection() }), day: day(), replayed: false });
  mountWithHousehold();
  fillCompleteInspection(); // atividade LI+T, trabalhado, A2 3/2/1/1/1, focal Pyriproxyfen 2, amostra A2 8, ação educativa
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  await waitFor(() => expect(api.recordHomeVisit).toHaveBeenCalledTimes(1));
  const body = vi.mocked(api.recordHomeVisit).mock.calls[0][0];
  expect(body.outcome).toBe(1);
  expect(body.reasons.some((r) => [ 34, 35, 36, 37 ].includes(r))).toBe(false);
  expect(body.vector_inspection).toMatchObject({ activity: "li_t", situation: "worked", deposits: [ { group: "A2", with_larvae: 1 } ], samples: [ { number: 1, deposit_group: "A2" } ] });
});
it("com boletim: 34–37 fora da lista, prévia dos derivados, limite com os derivados e aviso de família", async () => {
  mountWithHousehold({ domains: { ...VISIT_DOMAINS, reasons_max: 3 } });
  expect(screen.queryByLabelText("Imóvel com foco")).toBeNull();
  fillCompleteInspection();
  expect(screen.getByText(/O boletim acrescenta: 34 .*35 .*36 .*37/)).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  expect(await screen.findByText(REASONS_MAX_TEXT(3))).not.toBeNull();
  expect(api.recordHomeVisit).not.toHaveBeenCalled();
});
it("fechado depois de trabalhado: poda e diz o quê; desfecho vira ausente", async () => {
  mountWithHousehold(); fillCompleteInspection();
  fireEvent.click(screen.getByLabelText("Fechado"));
  expect(screen.getByRole("status").textContent).toMatch(/depósitos, medidas, amostras, ação educativa/);
  expect(screen.getByText(/Desfecho: Ausente/)).not.toBeNull();
});
it("etiqueta: prévia provisória; 4xx troca; a definitiva vem da visita gravada", async () => {
  mountWithHousehold(); fillCompleteInspection();
  const first = screen.getByText(/-01 \(provisória até gravar\)/).textContent;
  vi.mocked(api.recordHomeVisit).mockRejectedValueOnce(new ApiError(422, { error: "product_not_allowed", path: "vector_inspection.measures[0].vector_product_id" }, "x"));
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  expect(await screen.findByText(VECTOR_ERRORS.product_not_allowed)).not.toBeNull();
  expect(screen.getByText(/-01 \(provisória até gravar\)/).textContent).not.toBe(first);
  vi.mocked(api.recordHomeVisit).mockResolvedValueOnce({ visit: visit({ vector_inspection: inspection({ samples: [ { ...SAMPLE, label: "ABCDEF12-01" } ] }) }), day: day(), replayed: false });
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  expect(await screen.findByText("Etiquete os tubitos: ABCDEF12-01")).not.toBeNull();
});
it("sample_label_taken: frase na amostra e etiquetas novas na prévia (Q7(1))", async () => {
  // recordHomeVisit rejeita 422 { error: "sample_label_taken", path: "vector_inspection.samples[0]" } → a frase no cartão da amostra 1; a prévia já mostra outra etiqueta (uuid renovado)
});
it("imóvel sem cadastro: sem boletim, com a frase do domicílio (Q10) e o atalho para cadastrar o não residencial (Q13(1))", async () => { /* modo "imóvel sem cadastro" → sem InspectionSection, corpo sem vector_inspection; link "Cadastrar o imóvel no território" → onGoToTerritory */ });
it("recusa do api no campo do boletim (path vector_inspection.deposits[0].treated)", async () => { /* o input "Tratados" do grupo A2 fica aria-invalid com a frase deposit_counts_inconsistent */ });
it("quarteirão só quando o perfil não tem; sem bairro, a frase e nenhum seletor", async () => {
  // getHouseholdVectorProfile → householdProfile({ vector: householdVector({ city_block_id: null }) }) → select "Quarteirão" com os ativos de nb1
  // householdProfile({ neighborhood_id: null, vector: null }) → VECTOR_ERRORS.household_without_neighborhood, sem select
});
it("ACS (515105), gate desligado ou 'Registrar boletim' desmarcado: sem seção, nada de vector_inspection no corpo", async () => { /* … */ });
it("LIRAa e bloqueio só com o vínculo do start; PE só em ponto estratégico", async () => { /* start.vector { activity: "liraa", liraaDrawItemId: "li1" } → corpo com liraa_draw_item_id; sem start, sem a opção LIRAa; imóvel "1" sem PE */ });
```

`src/modules/homeVisits/VisitDetail.vector.test.tsx`: visita com `inspection({ recovered: true })` mostra "recuperado", os depósitos, `"Pyriproxyfen 0,5% G · 2 cargas · 4,00 g"`, a etiqueta `"3F2A9C1E-01"` e "aguardando resultado"; com `result` mostra examinadas/aegypti/albopictus; visita sem `vector_inspection` não mostra a seção nem chama `getVectorDomains`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod35a src/modules/homeVisits/VisitForm.vector.test.tsx src/modules/homeVisits/VisitDetail.vector.test.tsx src/modules/homeVisits/VisitForm.test.tsx`. **Step 3: Implemente.** **Step 4: Rode e veja passar** (os testes do 24b continuam verdes: sem `vectorDomains` nada muda); `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/homeVisits/InspectionSection.tsx src/modules/homeVisits/InspectionDetail.tsx src/modules/homeVisits/VisitForm.tsx src/modules/homeVisits/useVisitHandoff.ts src/modules/homeVisits/VisitDetail.tsx src/modules/homeVisits/ReasonsField.tsx src/modules/HomeVisitDay.tsx src/modules/HomeVisitsTeam.tsx src/modules/homeVisits/VisitForm.vector.test.tsx src/modules/homeVisits/VisitDetail.vector.test.tsx
$G commit -m "feat: record the endemic agent field inspection inside the home visit with derived reasons and sample labels" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: "Imóveis a visitar" na Folha do dia (F-35.4)

**Files:**
- Create: `src/modules/vectorControl/DueList.tsx`, `src/modules/HomeVisitDay.due.test.tsx`
- Modify: `src/modules/HomeVisitDay.tsx`

**Interfaces:**
- Consumes: `listDue`, `getVectorDomains` (Task 1); `dueReasonLabel`, `riskTone`, `domainLabel`, `isEndemicAgentTeam`, `canUseVectorControl`, `VC_DUE_KEY`, `STATIC_QUERY`, `vectorError` (Task 3); `usePaged`, `PagedBody` (24b); `propertyLabel` (24b).
- Produces:
  - `DueList({ team, domains, visitDomains, onStart(s: VisitStart) })`: seletor de microárea (as de `team.micro_areas`; uma só → sem seletor), lista paginada `usePaged([VC_DUE_KEY, microAreaId], (b) => listDue(microAreaId, b), false, true)` (sem pessoa, sem trilha), cartões na ordem do api: endereço curto, tipo de imóvel, `"Quarteirão 12"` ou `"sem quarteirão"`, `Tag` de risco, `Tag` do motivo, `"trabalhado em 01/10/2026"` ou `"nunca trabalhado"`, `"vencido desde 10/11/2026"`; botão "Registrar visita" → `onStart({ teamId, microAreaId, householdId, familyId: null, patientId: null, displayName: null, vector: startFor(item) })`, com `startFor`: `transmission_block` → `{ activity: "transmission_block", transmissionBlockId }`; `liraa` → `{ activity: "liraa", liraaDrawItemId }`; `pending` → `{ workType: "pending" }`; imóvel `"4"` → `{ activity: "pe" }`; senão `{}`. Vazio: `"nenhum imóvel vencido nesta microárea"`.
  - `HomeVisitDay`/`TodaySheets`: com `canUseVectorControl(user)` e `isEndemicAgentTeam(team)`, lê `getVectorDomains` (`VC_DOMAINS_KEY`, `STATIC_QUERY`), passa `vectorDomains` ao `VisitForm` e mostra `Panel "Imóveis a visitar"` abaixo do "Nova visita"; `onStart` abre o formulário com o `start`; depois de `onSaved`, invalida `[VC_DUE_KEY, …]` uma vez.

- [ ] **Step 1: Testes que falham** (`HomeVisitDay.due.test.tsx`, contexto com `agentTeam()`):

```tsx
it("agente: lista sem pessoa, na ordem do api, com motivo, risco e quarteirão", async () => {
  vi.mocked(api.listDue).mockResolvedValue({ items: [ dueItem({ reason: "transmission_block", transmission_block_id: "tb1" }), dueItem({ household_id: "h2", reason: "due", block: null, last_worked_on: null }) ], next_before: null });
  mount();
  const cards = await screen.findAllByRole("article");
  expect(cards.map((c) => c.textContent)).toEqual([ expect.stringMatching(/bloqueio.*Quarteirão 12/), expect.stringMatching(/vencido.*sem quarteirão.*nunca trabalhado/) ]);
  expect(api.listDue).toHaveBeenCalledWith("ma1", null);
});
it("Registrar visita do bloqueio abre o formulário com a atividade e o vínculo", async () => {
  // clique → VisitForm com a atividade "Bloqueio" marcada; o corpo enviado leva transmission_block_id "tb1" e household_id
});
it("ACS ou gate desligado: sem a lista e sem chamar listDue", async () => { /* acsTeam(); vcUser sem vector_control */ });
it("depois de gravar a visita, a lista é relida uma vez", async () => { /* listDue 2× */ });
```

- [ ] **Step 2–4:** rode e veja falhar (`src/modules/HomeVisitDay.due.test.tsx src/modules/homeVisits/DaySheet.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/vectorControl/DueList.tsx src/modules/HomeVisitDay.tsx src/modules/HomeVisitDay.due.test.tsx
$G commit -m "feat: list due properties for the endemic agent on the day sheet and start the visit from them" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Ovitrampas de campo do agente — instalar e coletar (F-35.5)

**Files:**
- Create: `src/modules/vectorControl/InstallForm.tsx`, `src/modules/vectorControl/CollectForm.tsx`, `src/modules/FieldOvitraps.test.tsx`
- Modify: `src/modules/FieldOvitraps.tsx`

**Interfaces:**
- Consumes: `listOvitraps`, `installOvitrap`, `collectOvitrap`, `getVectorDomains`, `getHomeVisitContext`, `listVisitHouseholds`, `getHouseholdVectorProfile` (Task 1 / 24b); `installProblems`, `collectionProblems`, `ovitrapStatusLabel`, `conditionLabel`, `isEndemicAgentTeam`, `vectorError`, `vectorRefusals`, `VC_OVITRAPS_KEY` (Task 3); `useClientUuid` (24b); `VectorGate` (Task 4).
- Produces:
  - `FieldOvitraps({ today? })`: `VectorGate allow={(r) => r.professional}` → contexto do 24b; sem equipe do agente → `EmptyState VC_NOT_AGENT`; seletor de equipe e microárea; cartões das ativas (`listOvitraps({ microAreaId, status: "active" })`, `STATIC_QUERY`): número, endereço curto, quarteirão, `"palheta desde 10/11/2026 — colete entre 13/11 e 20/11 (ideal 15/11)"` (min, max e o alvo de 5 dias), última leitura; "Coletar palheta" abre `CollectForm`; "Instalar ovitrampa" abre `InstallForm`. Nenhuma `<table>`.
  - `InstallForm({ microAreaId, today, onDone })`: domicílio pela lista da microárea (`listVisitHouseholds`) e, escolhido, `getHouseholdVectorProfile` (`STATIC_QUERY`) para saber o quarteirão (`vector` nulo ou `vector.city_block_id` nulo → a frase `household_without_block` e o botão desabilitado); número; data (≤ hoje); caixa "O morador assinou o termo de consentimento (TCLE)"; `installOvitrap({ client_uuid, household_id, number, consent: true, installed_on })` com `useClientUuid` (o mesmo uuid no reenvio por rede/5xx, como o 24b); `409 ovitrap_number_taken` no campo "Número".
  - `CollectForm({ ovitrap, params, today, onDone })`: data da coleta (padrão hoje), condição (rádios), "Reinstalar palheta" (marcado), "Retirar a ovitrampa" (desmarca reinstalar; com "perdida" a frase "a ovitrampa sai como perdida"); `collectionProblems` antes de enviar; `collectOvitrap(id, { collected_on, condition, reinstall, remove })`.

- [ ] **Step 1: Testes que falham** (`FieldOvitraps.test.tsx`):

```tsx
it("só o agente: ACS vê a frase e nada é listado", async () => { /* context com acsTeam → VC_NOT_AGENT; listOvitraps não chamado */ });
it("instalar: exige quarteirão e o TCLE; manda client_uuid, consent true e a data", async () => {
  vi.mocked(api.getHouseholdVectorProfile).mockResolvedValueOnce(householdProfile({ vector: householdVector({ city_block_id: null }) }));
  // escolher h1 → VECTOR_ERRORS.household_without_block e "Instalar" desabilitado
  // escolher h2 (com quarteirão), número "OV-12", sem TCLE → VECTOR_ERRORS.consent_required, sem chamada
  // marcar TCLE → installOvitrap({ client_uuid: expect.any(String), household_id: "h2", number: "OV-12", consent: true, installed_on: TODAY35 }); número "OV 12" recusado antes (Q7(6))
});
it("coletar: 2 dias recusa antes de enviar; 3 dias envia com reinstalar", async () => {
  // ovitrap({ open_reading: { id: "r1", installed_on: "2026-11-14" } }) e hoje 2026-11-16 → VECTOR_ERRORS.collection_out_of_range, sem chamada
  // installed_on "2026-11-13" → collectOvitrap("ov1", { collected_on: "2026-11-16", condition: "ok", reinstall: true, remove: false })
});
it("retirar perdida: reinstalar desmarca; o corpo leva remove true e condition lost", async () => { /* … */ });
it("422 ovitrap_number_taken (path number, Q23) no campo; rede guarda o uuid e 'Tentar de novo' manda o mesmo", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar (`src/modules/FieldOvitraps.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/FieldOvitraps.tsx src/modules/vectorControl/InstallForm.tsx src/modules/vectorControl/CollectForm.tsx src/modules/FieldOvitraps.test.tsx
$G commit -m "feat: let the endemic agent install ovitraps and collect paddles from the dashboard" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Mapa do 35a — cor por categoria no adaptador do 24c

**Files:**
- Create: `src/modules/vectorControl/VectorMap.tsx`, `src/modules/vectorControl/VectorMapPanel.tsx`, `src/modules/vectorControl/VectorMapPanel.test.tsx`
- Modify: `src/modules/territory/map/fieldMap.ts`, `src/modules/territory/map/fieldMap.test.ts`, `src/modules/territory/map/points.ts`, `src/modules/territory/map/points.test.ts`

**Interfaces:**
- Consumes: `createFieldMap`, `FieldMapHandle`, `roundCoord`, `boundsOf` (24c); `listVectorMapPoints`, `fieldMapUrl` (Task 1); `VC_MAP_KEY`, `STATIC_QUERY` (Task 3).
- Produces:
  - `PointProperties` + `category?: string`; `FieldMapOptions` + `cluster?: boolean` (padrão `true`, o 24c não muda) e `colors.categories?: Record<string, string>`; com `categories`, `households-points` pinta por `["match", ["get", "category"], <cat>, <cor>, …, colors.point]`.
  - `toCategoryCollection(points: VectorMapPoint[]): PointCollection` (`[longitude, latitude]`; propriedades só `id`, `selected: false`, `label`, `category`); `boundsOfPoints(points)`.
  - `CATEGORY_COLORS` por camada: `risk` → `green` `--ok`, `yellow` `--warn`, `red` `--down`, `gray` `--ink3`, `purple` `--accent`, `none` `--ink2`; `ovitraps` → `negative` `--ok`, `positive` `--warn`, `alert` `--down`, `not_counted` `--ink3`; `transmission` → `area` `--warn`. São exatamente as categorias do plano do api (Q13(7); `risk` = `risk_class` ou `none`, `ovitraps` = `negative`/`positive`/`alert`/`not_counted`, `transmission` = `area`); categoria desconhecida pinta com `colors.point` e aparece crua na legenda. A legenda: risco pelos domínios, `"sem risco avaliado"`, `"negativa"`, `"positiva"`, `"acima do alerta"`, `"não contada"`, `"área do bloqueio"`.
  - `VectorMap({ points, layer })` (default export, `React.lazy`): `createFieldMap(el, { mapUrl: fieldMapUrl(), base, cluster: false, colors: { point, selected, outline, categories } })`, cores lidas por `getComputedStyle`; `setPoints`/`fitTo` quando muda; `destroy` ao sair.
  - `VectorMapPanel({ query: VectorMapQuery, title })`: `useQuery([VC_MAP_KEY, query], …, NOMINAL_QUERY)` (ler pontos grava a trilha `territory_map` do 24c, Q13(7): lido ao abrir e por ação, nunca sozinho); `map: null` → `EmptyState "o mapa desta cidade ainda não foi gerado"` (frase do 24c) e nada de MapLibre; sem pontos → `"nenhum imóvel desta seleção tem ponto marcado — o ponto é marcado no app de campo"`; com pontos → `VectorMap` + legenda (contagem por categoria) + `aria-label` do mapa = título. Nenhum rótulo com pessoa: o `label` vem do api (D-5) e só vai à legenda do ponto clicado.

- [ ] **Step 1: Testes que falham**

`points.test.ts`: `toCategoryCollection([ { id: "h1", latitude: -25.4, longitude: -49.27, category: "red", label: "Quarteirão 12" } ])` → coordenadas `[ -49.27, -25.4 ]`, propriedades exatamente `{ id, selected, label, category }`. `fieldMap.test.ts`: com `cluster: false` a fonte não tem `cluster`; com `categories` o `circle-color` de `households-points` é `[ "match", [ "get", "category" ], "red", "#c00", "green", "#0a0", "#888" ]`; sem as opções novas, o 24c segue igual (os testes dele passam sem mudança).

`VectorMapPanel.test.tsx` (`vi.mock("../territory/map/fieldMap", () => ({ createFieldMap: vi.fn(() => handle) }))`):

```tsx
it("sem mapa da cidade: aviso, nada do MapLibre", async () => { /* { map: null, points: [] } → texto e createFieldMap não chamado */ });
it("pontos por categoria, sem cluster, com a legenda contada", async () => {
  vi.mocked(api.listVectorMapPoints).mockResolvedValue({ map: MAP_MANIFEST, points: mapPoints() }); // 2 red, 1 green
  mount({ layer: "risk", neighborhoodId: "nb1" });
  await waitFor(() => expect(handle.setPoints).toHaveBeenCalled());
  expect(vi.mocked(createFieldMap).mock.calls[0][1]).toMatchObject({ cluster: false, colors: { categories: expect.objectContaining({ red: expect.any(String) }) } });
  expect(screen.getByText("Vermelho (alto): 2")).not.toBeNull();
});
it("camada do bloqueio: só o id do bloqueio na consulta; uma leitura, sem releitura no foco (trilha)", async () => { /* listVectorMapPoints({ layer: "transmission", transmissionBlockId: "tb1" }) 1×, também depois de focus */ });
it("CATEGORY_COLORS cobre as categorias do api e nada mais (Q13(7))", () => {
  expect(Object.keys(CATEGORY_COLORS.risk).sort()).toEqual([ "gray", "green", "none", "purple", "red", "yellow" ]);
  expect(Object.keys(CATEGORY_COLORS.ovitraps).sort()).toEqual([ "alert", "negative", "not_counted", "positive" ]);
  expect(Object.keys(CATEGORY_COLORS.transmission)).toEqual([ "area" ]);
});
```

- [ ] **Step 2–4:** rode e veja falhar (`src/modules/territory/map/points.test.ts src/modules/territory/map/fieldMap.test.ts src/modules/vectorControl/VectorMapPanel.test.tsx src/modules/TerritoryMap.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/territory/map/fieldMap.ts src/modules/territory/map/fieldMap.test.ts src/modules/territory/map/points.ts src/modules/territory/map/points.test.ts src/modules/vectorControl/VectorMap.tsx src/modules/vectorControl/VectorMapPanel.tsx src/modules/vectorControl/VectorMapPanel.test.tsx
$G commit -m "feat: color field map points by category for the risk, ovitrap and transmission block maps" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Quarteirões e imóveis (F-35.1)

**Files:**
- Create: `src/modules/vectorControl/BlocksPanel.tsx`, `src/modules/vectorControl/HouseholdsPanel.tsx`, `src/modules/VectorBlocks.test.tsx`
- Modify: `src/modules/VectorBlocks.tsx`

**Interfaces:**
- Consumes: `listPanelNeighborhoods`, `listCityBlocks`, `createCityBlock`, `updateCityBlock`, `addBlockNeighbor`, `removeBlockNeighbor`, `listVectorHouseholds`, `assignHouseholdsBlock`, `updateHouseholdProfile`, `getVectorDomains` (Task 1); `blockNumberProblem`, `riskTone`, `vectorError`, `vectorRefusals`, `isVectorStale`, `VC_BLOCKS_KEY`, `VC_HOUSEHOLDS_KEY`, `NOMINAL_QUERY` (Task 3); `VectorMapPanel` (Task 8); `NeighborhoodPicker`, `DataTable`, `ConfirmButton`.
- Produces:
  - `VectorBlocks()`: `VectorGate allow={(r) => r.coordinator}`; bairro (`NeighborhoodPicker`, um); `SegmentedControl` "Quarteirões" | "Imóveis" | "Mapa".
  - `BlocksPanel({ neighborhoodId })`: `DataTable` (número, imóveis ativos, vizinhos por número, ativo); "Novo quarteirão" (número, `blockNumberProblem`; `422 block_number_taken` no campo); por quarteirão, "Vizinhos" (adicionar por `<select>` dos quarteirões **de qualquer bairro** — busca por bairro + número; remover com `ConfirmButton` "o par fica no histórico"); "Desativar" (`ConfirmButton`; `409 block_in_use` → frase e uma releitura).
  - `HouseholdsPanel({ neighborhoodId, domains })`: filtro "sem quarteirão" (padrão ligado) ou por quarteirão; lista nominal (`NOMINAL_QUERY`, relida só por ação; trilha `territory_registry`): endereço curto, tipo, microárea, quarteirão (com `Tag tone="warn"` "quarteirão de outro bairro" quando `block.neighborhood_id !== neighborhood_id`), risco, última visita trabalhada; caixas de seleção (até 200; "Selecionar a página"); "Atribuir ao quarteirão" (`<select>` dos ativos do bairro) → `assignHouseholdsBlock(ids, blockId)`; recusa com `path` `household_ids[i]` marca **aquele** imóvel e mantém a seleção; "Editar perfil" por imóvel (quarteirão e risco, `updateHouseholdProfile`).
  - "Mapa": `VectorMapPanel query={{ layer: "risk", neighborhoodId }} title="Imóveis por risco"`.

- [ ] **Step 1: Testes que falham** (`VectorBlocks.test.tsx`, sessão `endemic_coordinator`):

```tsx
it("novo quarteirão: número validado; block_number_taken no campo", async () => { /* "12 A" → frase, sem chamada; "12-A" → createCityBlock({ neighborhood_id: "nb1", number: "12-A" }); 422 → VECTOR_ERRORS.block_number_taken */ });
it("vizinho: de outro bairro vale; remover pede confirmação; invalid_neighbor tem frase", async () => { /* … */ });
it("desativar com imóvel: 409 block_in_use → frase e uma releitura", async () => { /* listCityBlocks 2× */ });
it("imóveis sem quarteirão: lista nominal lida uma vez; atribuir em lote manda os ids no corpo", async () => {
  vi.mocked(api.listVectorHouseholds).mockResolvedValue({ items: [ vectorHousehold({ id: "h1", block: null }), vectorHousehold({ id: "h2", block: null }) ], next_before: null });
  mount(); await selectNeighborhood("nb1"); fireEvent.click(await screen.findByRole("tab", { name: "Imóveis" }));
  fireEvent.click(await screen.findByLabelText("Selecionar a página"));
  pickBlock("12"); fireEvent.click(screen.getByRole("button", { name: "Atribuir ao quarteirão" }));
  await waitFor(() => expect(api.assignHouseholdsBlock).toHaveBeenCalledWith([ "h1", "h2" ], "cb1"));
  act(() => { window.dispatchEvent(new Event("focus")); });
  expect(api.listVectorHouseholds).toHaveBeenCalledTimes(2); // a de abrir e a releitura depois da ação
});
it("lote com um imóvel sem bairro: a recusa marca aquele imóvel e a seleção fica", async () => {
  // assignHouseholdsBlock rejeita ApiError(422, { error: "household_without_neighborhood", path: "household_ids[1]" }) → linha h2 com a frase; as duas caixas seguem marcadas
});
it("quarteirão de outro bairro aparece marcado", async () => { /* vectorHousehold({ neighborhood_id: "nb2", block: { id: "cb1", number: "12", neighborhood_id: "nb1" } }) */ });
it("nenhum morador na lista", async () => { /* container.textContent sem nome de pessoa das fixtures */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorBlocks.tsx src/modules/vectorControl/BlocksPanel.tsx src/modules/vectorControl/HouseholdsPanel.tsx src/modules/VectorBlocks.test.tsx
$G commit -m "feat: manage city blocks, their neighbors and household block and risk assignment for endemics" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Produtos, parâmetros e local de estoque (F-35.3)

**Files:**
- Create: `src/modules/vectorControl/ProductsPanel.tsx`, `src/modules/vectorControl/ParametersPanel.tsx`, `src/modules/VectorSettings.test.tsx`
- Modify: `src/modules/VectorSettings.tsx`

**Interfaces:**
- Consumes: `listVectorProducts`, `createVectorProduct`, `updateVectorProduct`, `getVectorParameters`, `putVectorParameters`, `getVectorSettings`, `putVectorSettings` (Task 1); `vectorError`, `vectorRefusals`, `VC_PRODUCTS_KEY`, `VC_PARAMS_KEY`, `VC_SETTINGS_KEY` (Task 3); `SensitiveAction`.
- Produces:
  - `VectorSettings({ onGoToSecurity? })`: `VectorGate allow={(r) => r.coordinator}`; três `Panel`s.
  - `ProductsPanel`: `DataTable` (código, nome, unidade, carga, usos, ativo); novo/editar (código 1–10, nome 1–80, unidade `g`/`ml`/`un`, carga > 0 com vírgula, usos ≥ 1 entre focal/perifocal/espacial, ativo); produto com `in_use` mostra unidade e carga **desabilitadas** com a frase de `product_in_use`; `409 product_in_use` → frase e uma releitura. Depois de gravar, invalida também `VC_DOMAINS_KEY`.
  - `ParametersPanel`: os 19 campos do §11 agrupados (cadência, bloqueio, UBV, ovitrampa, LIRAa, IIP), `"versão N, de 16/11/2026"` (versão 0 com `created_at: null` → `"padrão de fábrica (versão 0)"`, Q13(8)); `block_diseases` como lista de códigos separados por vírgula; `iip_bands` dois números; "Salvar parâmetros" por `SensitiveAction` (step-up, `description` "Cria a versão N+1. O levantamento já sorteado continua com a versão do sorteio."); `422 invalid_parameter` com `path` cai no campo da chave.
  - Local de estoque: `<select>` de `settings.options` + "sem baixa de estoque" (nulo) → `putVectorSettings`; a frase `"sem local, cada aplicação vira falta de insumo (sem local)"` quando nulo.

- [ ] **Step 1: Testes que falham** (`VectorSettings.test.tsx`):

```tsx
it("produto novo: corpo com charge_size em texto e usos; produto em uso trava unidade e carga", async () => { /* createVectorProduct({ code: "BTI", name: "Bti WG", unit: "g", charge_size: "0.5", uses: [ "focal" ], active: true }) */ });
it("parâmetros: step-up, corpo inteiro, invalid_parameter no campo da chave", async () => {
  // SensitiveAction → putVectorParameters({ ...PARAMS, ovitrap_alert_eggs: 120 }); rejeita 422 { error: "invalid_parameter", path: "values.ovitrap_alert_eggs" } → o input "Alerta de ovos" com a frase
});
it("local de estoque: escolhe entre as opções; nulo diz que vira falta", async () => { /* putVectorSettings("loc-uvz"); putVectorSettings(null) */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorSettings.tsx src/modules/vectorControl/ProductsPanel.tsx src/modules/vectorControl/ParametersPanel.tsx src/modules/VectorSettings.test.tsx
$G commit -m "feat: manage vector control products, versioned technical parameters and the stock location" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Ciclos, resumo por bairro e cadência (F-35.4)

**Files:**
- Create: `src/modules/vectorControl/CadencesPanel.tsx`, `src/modules/VectorCycles.test.tsx`
- Modify: `src/modules/VectorCycles.tsx`

**Interfaces:**
- Consumes: `listCycles`, `createCycle`, `updateCycle`, `getCycleSummary`, `listCadences`, `putCadences`, `listPanelNeighborhoods`, `getVectorDomains` (Task 1); `cycleProblems`, `vectorRefusals`, `VC_CYCLES_KEY`, `VC_CYCLE_KEY`, `VC_CADENCES_KEY` (Task 3); `SensitiveAction`, `StatTile`.
- Produces:
  - `VectorCycles({ today? })`: `VectorGate allow={(r) => r.coordinator}`; ano (padrão o de hoje); `DataTable` dos ciclos (número, período); novo/editar (`cycleProblems`; `422 cycle_overlap`/`cycle_invalid` no campo); ciclo escolhido → `getCycleSummary` (`STATIC_QUERY`): por bairro, a visitar, visitados, trabalhados, fechados, recusados, desocupados, recuperados, pendência (`"12,5%"`, `null` = "—") e "imóveis cadastrados" ao lado (risco de cobertura, decisão 7), com o aviso `"os números contam só os imóveis do cadastro territorial"`.
  - `CadencesPanel`: linhas (bairro ou "cidade", risco ou "qualquer", "ponto estratégico", dias 7–365) editadas em memória; a ordem de aplicação explicada em uma linha (§5 com a Q10): PE — (bairro, PE) → (cidade, PE) → `strategic_point_days`; demais — (bairro, risco) → (bairro, qualquer) → (cidade, risco) → (cidade, qualquer) → `default_days`; "Salvar cadência" por `SensitiveAction` → `putCadences(lista inteira)`; linha repetida (bairro, risco, PE) recusada antes de enviar.

- [ ] **Step 1: Testes que falham** (`VectorCycles.test.tsx`): ciclo novo com fim antes do início não envia; `cycle_overlap` no campo; resumo mostra `"12,5%"` e `"—"` para `null`; cadência: step-up, corpo `[ { neighborhood_id: null, risk_class: "red", strategic_point: false, days: 30 } ]` com a lista inteira; linha repetida recusada antes; dias 6 e 366 recusados.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorCycles.tsx src/modules/vectorControl/CadencesPanel.tsx src/modules/VectorCycles.test.tsx
$G commit -m "feat: plan vector control cycles with per-neighborhood progress and edit the visit cadence" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Ovitrampas da coordenação e do laboratório — lista, mapa e contagem (F-35.5)

**Files:**
- Create: `src/modules/vectorControl/CountForm.tsx`, `src/modules/VectorOvitraps.test.tsx`
- Modify: `src/modules/VectorOvitraps.tsx`

**Interfaces:**
- Consumes: `listOvitraps`, `listOvitrapReadings`, `countOvitrapReading`, `getVectorReport`, `listPanelNeighborhoods` (Task 1); `countProblems`, `conditionLabel`, `ovitrapStatusLabel`, `vectorRefusals`, `VC_OVITRAPS_KEY`, `VC_READINGS_KEY`, `VC_REPORT_KEY` (Task 3); `epiWeekOf`, `previousEpiWeek`, `weekLabel` (`src/lib/surveillance.ts`); `VectorMapPanel` (Task 8).
- Produces:
  - `VectorOvitraps({ today? })`: `VectorGate allow={(r) => r.coordinator || r.lab}`; abas "A contar" | "Ovitrampas" | "Índices e mapa".
  - "A contar": `listOvitrapReadings({ state: "to_count" })` (`DataTable`: ovitrampa, quarteirão, bairro, instalada, coletada, semana) e "Contadas" (`state: "counted"`); "Contar" e "Corrigir contagem" abrem `CountForm({ reading, corrected })` (ovos, nota obrigatória só na correção, `FrozenTextNotice` na nota); `422 reading_not_countable` → frase e uma releitura.
  - "Ovitrampas": por bairro e status (`listOvitraps({ neighborhoodId, status })`), com número, quarteirão, status e última leitura.
  - "Índices e mapa": semana (padrão `previousEpiWeek(today)`), bairro; tabela do relatório `ovitraps` (`getVectorReport("ovitraps", { epi_weeks: [w], neighborhood_ids: [nb] })`, `ReportTable` da Task 17 — até lá, a tabela simples desta task, trocada na Task 17); `VectorMapPanel query={{ layer: "ovitraps", neighborhoodId, epiWeek }}`.

- [ ] **Step 1: Testes que falham** (`VectorOvitraps.test.tsx`): laboratório vê as três abas e conta: `countOvitrapReading("rd1", { egg_count: 0 })` (zero vale, sem `count_note`); correção sem nota não envia, com nota manda `count_note`; `reading_not_countable` → frase e `listOvitrapReadings` 2×; a aba "Índices e mapa" chama o relatório com a semana anterior a `TODAY35` (`"2026-45"`) e o mapa com a mesma semana; a lista não mostra morador.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorOvitraps.tsx src/modules/vectorControl/CountForm.tsx src/modules/VectorOvitraps.test.tsx
$G commit -m "feat: count ovitrap eggs, list ovitraps and show weekly indexes and the egg count map" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Amostras de larvas — fila e resultado (F-35.8)

**Files:**
- Create: `src/modules/VectorSamples.test.tsx`
- Modify: `src/modules/VectorSamples.tsx`

**Interfaces:**
- Consumes: `listVectorSamples`, `recordSampleResult` (Task 1); `sampleResultProblems`, `vectorRefusals`, `VC_SAMPLES_KEY` (Task 3); `usePaged`.
- Produces: `VectorSamples()`: `VectorGate allow={(r) => r.coordinator || r.lab}`; abas "Sem resultado" (por data da visita, a mais antiga primeiro, como o api manda) e "Com resultado"; colunas: etiqueta (mono), data da visita, grupo, larvas coletadas, quarteirão; "Lançar resultado" / "Corrigir resultado" → formulário (examinadas, *Ae. aegypti*, *Ae. albopictus*, nota só na correção) com `sampleResultProblems`; depois de gravar, a fila é relida uma vez e a frase `"Resultado lançado. O índice do LIRAa usa o resultado novo."`.

- [ ] **Step 1: Testes que falham**: fila mostra a etiqueta `"3F2A9C1E-01"` e nenhum morador; resultado com `examined 9 > 8` não envia (`VECTOR_ERRORS.sample_result_invalid`); `recordSampleResult("sa1", { examined_larvae: 8, aegypti_larvae: 5, albopictus_larvae: 0 })`; correção exige `result_note`; duplo clique envia uma vez.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorSamples.tsx src/modules/VectorSamples.test.tsx
$G commit -m "feat: record and correct larvae sample results from the sample queue" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: LIRAa — estratos, sorteio, itens, fechamento e índices (F-35.6)

**Files:**
- Create: `src/modules/vectorControl/StrataEditor.tsx`, `src/modules/VectorLiraa.test.tsx`
- Modify: `src/modules/VectorLiraa.tsx`

**Interfaces:**
- Consumes: `listLiraaSurveys`, `createLiraaSurvey`, `updateLiraaSurvey`, `drawLiraaSurvey`, `closeLiraaSurvey`, `listLiraaItems`, `getVectorReport`, `fetchVectorReportFile`, `getVectorParameters`, `listCityBlocks`, `listPanelNeighborhoods` (Task 1); `stratumRangeProblem`, `stratumModeLabel`, `surveyStatusLabel`, `itemStatusLabel`, `isVectorStale`, `vectorRefusals` (Task 3); `SensitiveAction`, `saveFile`.
- Produces:
  - `VectorLiraa({ today? })`: `VectorGate allow={(r) => r.coordinator}`; lista de levantamentos (nome, período, status); "Novo levantamento" (nome 1–80, início ≤ fim).
  - Em `draft`: `StrataEditor({ survey, params, onSaved })`: estratos (nome 1–60, tipo) e, por estrato, quarteirões escolhidos por bairro (caixa por quarteirão, `households_count` ao lado); soma N por estrato e `stratumRangeProblem(mode, N, params)` ao vivo (bloqueia "Salvar estratos" e "Sortear"); quarteirão já em outro estrato aparece desabilitado (`stratum_overlap`); "Salvar estratos" → `updateLiraaSurvey(id, { strata })`; `422 stratum_out_of_range` com `path` `strata[i]` cai no estrato.
  - "Sortear" por `SensitiveAction` (step-up; `description` "O sorteio é feito uma vez, com início aleatório gravado. Depois dele os estratos não mudam."); botão some enquanto envia; `409 survey_not_draft` → frase e **uma** releitura.
  - Em `drawn`/`closed`: por estrato, N, n, intervalo, início; itens (`listLiraaItems`, `DataTable`: quarteirão, endereço curto, posição, status, "substituto de …"); índices (`getVectorReport("liraa", { survey_id })`, com `"provisório: há amostra sem resultado"` quando `provisional`); "Fechar levantamento" por `SensitiveAction` (só `drawn`); downloads "CSV" e "PDF" (`fetchVectorReportFile("liraa", { survey_id }, fmt)` + `saveFile(file, file.filename ?? "liraa-<id>.<fmt>")`).

- [ ] **Step 1: Testes que falham** (`VectorLiraa.test.tsx`):

```tsx
it("estrato fora da faixa trava salvar e sortear (N 8.099 em padrão)", async () => { /* quarteirões somando 8099 em "Padrão" → frase stratum_out_of_range; trocar para "Reduzido" libera */ });
it("quarteirão de outro estrato fica desabilitado", async () => { /* … */ });
it("sorteio: step-up, um envio; 409 relê uma vez", async () => {
  vi.mocked(api.drawLiraaSurvey).mockRejectedValueOnce(new ApiError(409, { error: "survey_not_draft" }, "x"));
  mountDraft(); confirmStepUp("Sortear");
  expect(await screen.findByText(VECTOR_ERRORS.survey_not_draft)).not.toBeNull();
  expect(api.drawLiraaSurvey).toHaveBeenCalledTimes(1);
  expect(api.listLiraaSurveys).toHaveBeenCalledTimes(2);
});
it("sorteado: itens com substituição e índice provisório; PDF e CSV baixam pelo saveFile", async () => {
  // liraaItem({ status: "substituted" }) e outro com substitute_of_id → "substituto de"; report({ provisional: true }) → aviso; PDF → saveFile(file, "liraa-2026.pdf")
});
it("fechar: só de sorteado, com step-up", async () => { /* … */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorLiraa.tsx src/modules/vectorControl/StrataEditor.tsx src/modules/VectorLiraa.test.tsx
$G commit -m "feat: build LIRAa strata, draw and close surveys and show items, substitutions and indexes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Bloqueio de transmissão — propostas, área, ciclos e casos sem paciente (F-35.7)

**Files:**
- Create: `src/modules/vectorControl/BlockDetail.tsx`, `src/modules/vectorControl/CaseAddressDialog.tsx`, `src/modules/VectorTransmission.test.tsx`
- Modify: `src/modules/VectorTransmission.tsx`

**Interfaces:**
- Consumes: `listTransmissionBlocks`, `getTransmissionBlock`, `confirmTransmissionBlock`, `dismissTransmissionBlock`, `closeTransmissionBlock`, `addTransmissionCycle`, `moveTransmissionCase`, `getTransmissionCaseAddress`, `listCityBlocks`, `getVectorParameters` (Task 1); `ubvCycleProblems`, `blockStatusLabel`, `caseLabel`, `noteProblem`, `isVectorStale`, `vectorRefusals`, `VC_BLOCKS_TB_KEY`, `VC_BLOCK_TB_KEY` (Task 3); `VectorMapPanel` (Task 8); `SensitiveAction`.
- Produces:
  - `VectorTransmission()`: `VectorGate allow={(r) => r.coordinator}`; lista por status (propostos primeiro): doença, casos (`"2 casos"`), quarteirão do caso, criado em; nunca paciente.
  - `BlockDetail({ id })`: casos como `"caso 1 · dengue · 12/11/2026"` (`"sem quarteirão"` quando `!has_place`); área (quarteirões, os sugeridos marcados e pré-marcados na confirmação); quarteirão do caso (`<select>`); "Confirmar bloqueio" por `SensitiveAction` (só `proposed`, ≥ 1 quarteirão e o do caso) → `confirmTransmissionBlock(id, { center_block_id, block_ids })`; "Descartar" (motivo 10–500, `FrozenTextNotice`); "Fechar" (`ConfirmButton`, só `active`); ciclos de UBV (lista e "Novo ciclo" com `ubvCycleProblems`; `422 cycle_invalid` no campo); "Juntar a outro bloqueio" por caso (`<select>` dos `proposed`/`active`) → `moveTransmissionCase`; `VectorMapPanel query={{ layer: "transmission", transmissionBlockId: id }}`. `409 invalid_transition` (Q9) → frase e uma releitura.
  - `CaseAddressDialog({ blockId, caseId, onClose })`: botão "Ver endereço do caso" (só para caso sem quarteirão) → `SensitiveAction` (step-up, `description` "A leitura fica registrada na trilha. Use o endereço só para escolher o quarteirão.") → `getTransmissionCaseAddress` chamado **direto** (sem `useQuery`); o resultado fica em `useState` do diálogo e é apagado ao fechar.

- [ ] **Step 1: Testes que falham** (`VectorTransmission.test.tsx`):

```tsx
it("endereço do caso: step-up, só no diálogo, nada no cache; lista sem paciente", async () => {
  vi.mocked(api.getTransmissionCaseAddress).mockResolvedValue({ patient_address: "Rua Secreta, 99", household_address: null });
  const { client, container } = mountDetail(transmissionBlock({ cases: [ { id: "c1", number: 1, disease: "dengue", case_on: "2026-11-12", has_place: false } ] }));
  expect(container.textContent).toMatch(/caso 1 · dengue · 12\/11\/2026/);
  confirmStepUp("Ver endereço do caso");
  expect(await screen.findByText("Rua Secreta, 99")).not.toBeNull();
  expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state.data))).not.toMatch(/Rua Secreta/);
  fireEvent.click(screen.getByRole("button", { name: "Fechar" }));
  expect(container.textContent).not.toMatch(/Rua Secreta/);
});
it("confirmar: área sugerida pré-marcada, quarteirão do caso obrigatório, step-up, corpo com os ids", async () => { /* confirmTransmissionBlock("tb1", { center_block_id: "cb1", block_ids: [ "cb1", "cb2" ] }) */ });
it("ciclo de UBV com 4 dias de intervalo não envia; com 5 envia", async () => { /* ubvCycleProblems; addTransmissionCycle("tb1", { starts_on: "2026-11-15", ends_on: "2026-11-17" }) */ });
it("descartar exige motivo de 10 a 500; 409 invalid_transition relê uma vez", async () => { /* … */ });
it("juntar caso a outro bloqueio manda o destino no corpo", async () => { /* moveTransmissionCase("tb1", "c1", "tb2") */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorTransmission.tsx src/modules/vectorControl/BlockDetail.tsx src/modules/vectorControl/CaseAddressDialog.tsx src/modules/VectorTransmission.test.tsx
$G commit -m "feat: review, confirm, dismiss and close transmission blocks with UBV cycles and cases without patient data" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Tarefas de endemias

**Files:**
- Create: `src/modules/VectorTasks.test.tsx`
- Modify: `src/modules/VectorTasks.tsx`

**Interfaces:**
- Consumes: `listVectorTasks`, `closeVectorTask` (Task 1); `taskKindLabel`, `taskStateLabel`, `noteProblem`, `isVectorStale`, `vectorRefusals`, `VC_TASKS_KEY` (Task 3).
- Produces: `VectorTasks({ onOpenTransmission? })`: `VectorGate allow={(r) => r.coordinator}`; abas "Abertas" | "Encerradas"; cartão por tarefa: tipo, criada em, ovitrampa (`"OV-12"`) e quarteirões sugeridos (`"Quarteirões 12, 13 e 14"`) na intensificação; doença e data no caso de fora; "Abrir o bloqueio" (navega para `vector-transmission`) no caso sem quarteirão; "Concluir" e "Dispensar" com nota 10–500 → `closeVectorTask(id, { state, note })`; 409 → frase e uma releitura.
- [ ] **Step 1: Testes que falham**: intensificação mostra a ovitrampa e os quarteirões sugeridos; concluir sem nota não envia; com nota manda `{ state: "done", note }`; nenhuma tarefa mostra paciente; `invalid_transition` relê uma vez.
- [ ] **Step 2–4:** rode e veja falhar; implemente (o `App.tsx` passa `onOpenTransmission={() => setActive("vector-transmission")}`); rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorTasks.tsx src/modules/VectorTasks.test.tsx src/App.tsx
$G commit -m "feat: list and close endemics coordination tasks" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: Relatórios de endemias e downloads (F-35.9)

**Files:**
- Create: `src/modules/vectorControl/ReportTable.tsx`, `src/modules/VectorReports.test.tsx`
- Modify: `src/modules/VectorReports.tsx`, `src/modules/VectorOvitraps.tsx` (troca a tabela simples da Task 12 pela `ReportTable`)

**Interfaces:**
- Consumes: `getVectorReport`, `fetchVectorReportFile`, `listCycles`, `listLiraaSurveys`, `listPanelNeighborhoods` (Task 1); `vectorRoles`, `vectorError`, `VC_REPORT_KEY` (Task 3); `fmtCount` (`src/lib/smallCount.ts`); `saveFile`; `weekLabel`, `previousEpiWeek`.
- Produces:
  - `REPORTS_FOR(roles)`: coordenação e admin, os sete; laboratório, só `ovitraps` (D-19). Nomes: `situations` "Situação dos imóveis por ciclo", `strategic_points` "Pontos estratégicos", `production` "Produção dos agentes", `consumption` "Consumo de produtos", `ovitraps` "Ovitrampas (IPO, IDO, IDV)", `liraa` "LIRAa", `blocks` "Bloqueios".
  - `VectorReports({ today? })`: `VectorGate allow={(r) => r.reader}`; escolha do relatório e os filtros dele (§10: ciclo e bairros; período e profissionais; semanas e bairros; levantamento); "Ver" lê o JSON (`STATIC_QUERY`, só no clique); "Baixar CSV" (todos) e "Baixar PDF" (só `liraa`) → `saveFile(file, file.filename ?? "<nome>-<data>.<fmt>")`; o nome nunca leva pessoa.
  - `ReportTable({ report })`: colunas do api, contagem suprimida por `fmtCount` (`"< 5"`), percentual com uma casa, `"provisório"` quando `provisional`, as `notes` abaixo; `DataTable` com `card: "title"` na primeira coluna.

- [ ] **Step 1: Testes que falham** (`VectorReports.test.tsx`, `vi.mock("./lab/saveFile")`): coordenação vê os sete; laboratório só "Ovitrampas"; `situations` com ciclo e dois bairros chama `getVectorReport("situations", { cycle_id: "cy1", neighborhood_ids: [ "nb1", "nb2" ] })`; contagem `{ suppressed: true }` aparece `"< 5"`; CSV baixa com o nome do api; PDF só no LIRAa; nenhuma leitura antes do "Ver".
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar (`src/modules/VectorReports.test.tsx src/modules/VectorOvitraps.test.tsx`); `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/VectorReports.tsx src/modules/vectorControl/ReportTable.tsx src/modules/VectorReports.test.tsx src/modules/VectorOvitraps.tsx
$G commit -m "feat: show endemics reports with suppressed small counts and download them as CSV and PDF" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 18: Unidade de vigilância ambiental, domínio `vector_control` no estoque do 27a e faltas de insumo (F-35.1, F-35.3)

**Files:**
- Create: `src/modules/attendance/UnitForm.vector.test.tsx`, `src/modules/VectorStockShortfalls.test.tsx`
- Modify: `src/lib/attendance.ts`, `src/lib/attendance.test.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`, `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts`, `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/pharmacy/ReceiptForm.tsx`, `src/modules/pharmacy/ItemSearch.tsx`, `src/modules/pharmacy/TransferForm.tsx`, `src/modules/PharmacyTransfers.tsx`, `src/modules/PharmacyEntries.test.tsx`, `src/modules/PharmacyTransfers.test.tsx`, `src/modules/PharmacyAdmin.test.tsx`, `src/modules/VectorStockShortfalls.tsx`

**Interfaces:**
- Consumes: `canUseVectorControl`, `vectorRoles`, `shortfallCauseLabel`, `noteProblem`, `isVectorStale`, `vectorRefusals`, `VC_SHORTFALLS_KEY` (Task 3); `listVectorShortfalls`, `reconcileVectorShortfall`, `searchPharmacyItems(q, domain)` (Task 1).
- Produces:
  - `src/lib/attendance.ts`: `ENVIRONMENTAL_SURVEILLANCE_UNIT_KIND = { value: "environmental_surveillance", label: "Vigilância ambiental e zoonoses" }`; `unitKindOptions(dentalOn, tfdOn = false, capsOn = false, vectorOn = false)`; `UnitForm` ganha `vectorOn` (oferece o tipo com `vectorOn || value.kind === "environmental_surveillance"`); `Units` e `Attendance` passam `vectorOn={canUseVectorControl(user)}`; o `KIND_LABEL` de `Units.tsx` inclui o tipo; trocar o tipo de unidade com local `vector_control` → `409 unit_has_vector_control_location` com a frase de `VECTOR_ERRORS` (o `MESSAGES` do `attendance.ts` a importa), sem reenviar (Q10).
  - `src/lib/pharmacy.ts`: `KIND_LABEL.vector_control = "Controle vetorial (vigilância ambiental)"`; `MOVEMENT_LABEL.vector_application = "aplicação no campo (boletim)"`, `MOVEMENT_LABEL.vector_application_reversal = "estorno de aplicação (visita retirada)"` (Q10); `DOCUMENT_LABEL.vector_inspection_measure = "boletim de campo"`; `itemKey` → `vec:<vector_product_id>` para o domínio novo; `isVectorLocation(loc)`.
  - `LocationsPanel`: `KINDS` + `vector_control` (com `vectorOn`), exigindo unidade do tipo `environmental_surveillance` (o `<select>` só lista essas; sem nenhuma, a frase `"cadastre antes uma unidade do tipo Vigilância ambiental e zoonoses"`).
  - `ReceiptForm`/`ItemSearch` em local `vector_control`: rótulo "Buscar produto de controle vetorial", busca com `domain: "vector_control"`, linha manda `vector_product_id` (nunca `catalog_item_id`), sem embalagem; `422 item_not_allowed_here` tem a frase do 27a.
  - `TransfersBody`: em local `vector_control`, `canSend` = almoxarife do local e os destinos = outros locais `vector_control` ativos.
  - `VectorStockShortfalls()`: portão próprio (`canUseVectorControl` e coordenação ou almoxarife); almoxarife escolhe o local (`LocationPick` só com locais `vector_control`), a coordenação lê todos e as `no_location`; abas "Abertas" | "Conciliadas" (`SegmentedControl`); `DataTable`: data da visita, produto, quantidade com unidade, causa, local; "Conciliar" com nota 10–500 → `reconcileVectorShortfall`; `409 already_reconciled` → frase e uma releitura; o texto `"a conciliação só registra a conferência — o ajuste de saldo é feito em Inventário e ajustes"` (como o 32b).

- [ ] **Step 1: Testes que falham**
  - `UnitForm.vector.test.tsx`: com `vectorOn` o tipo "Vigilância ambiental e zoonoses" é oferecido; sem ele não; a unidade existente do tipo continua legível com o interruptor desligado.
  - `PharmacyAdmin.test.tsx`: local "Controle vetorial" só com unidade `environmental_surveillance`; `createAdminLocation({ kind: "vector_control", health_unit_id: "u-uvz", name: "UVZ — insumos", active: true })`.
  - `PharmacyEntries.test.tsx`: em local `vector_control`, a busca manda `domain` e a entrada manda `items: [ { vector_product_id: "pr1", lot_number: "L1", expires_on: "2027-06-30", quantity: "500" } ]`, sem `catalog_item_id` nem `package`.
  - `PharmacyTransfers.test.tsx`: em local `vector_control`, os destinos são só os outros `vector_control`.
  - `pharmacy.test.ts`: `movementLabel("vector_application")`, `movementLabel("vector_application_reversal")` e `locationKindLabel("vector_control")`.
  - `UnitForm.vector.test.tsx` (também): salvar a troca de tipo recusada com `409 unit_has_vector_control_location` mostra `VECTOR_ERRORS.unit_has_vector_control_location`.
  - `VectorStockShortfalls.test.tsx`: coordenação vê a `no_location` sem local; almoxarife só do seu local; conciliar sem nota não envia; `already_reconciled` relê uma vez; nenhuma linha com pessoa.
- [ ] **Step 2–4:** rode e veja falhar (os arquivos acima e `src/modules/attendance/UnitForm.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/attendance.ts src/lib/attendance.test.ts src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/Attendance.tsx src/modules/attendance/UnitForm.vector.test.tsx src/lib/pharmacy.ts src/lib/pharmacy.test.ts src/modules/pharmacy/LocationsPanel.tsx src/modules/pharmacy/ReceiptForm.tsx src/modules/pharmacy/ItemSearch.tsx src/modules/pharmacy/TransferForm.tsx src/modules/PharmacyTransfers.tsx src/modules/PharmacyEntries.test.tsx src/modules/PharmacyTransfers.test.tsx src/modules/PharmacyAdmin.test.tsx src/modules/VectorStockShortfalls.tsx src/modules/VectorStockShortfalls.test.tsx
$G commit -m "feat: add the environmental surveillance unit kind, the vector control stock domain and field supply shortfall reconciliation" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 18A: Imóvel não residencial pelo agente de endemias no cadastro do 24a (Q13(1))

Com `vector_control` utilizável, o agente (`515140`) cadastra e edita domicílio do 24a **só** de tipo não residencial (`2 3 4 5 6 12 13`), sem família nem morador, nas microáreas atribuídas a ele; o resto continua `403 cbo_not_allowed`. É o caminho para o boletim de PE e IE, que exige domicílio (Q10).

**Files:**
- Modify: `src/lib/api.ts` (`TerritoryTeam.micro_areas[]` + `property_types?: string[] | null`, D-21), `src/lib/territoryRegistry.ts` (`ERRORS.cbo_not_allowed`; `householdTypesFor(microArea)`), `src/lib/territoryRegistry.test.ts`, `src/modules/TerritoryTeam.tsx`, `src/modules/territory/HouseholdForm.tsx` (prop `propertyTypes?: string[] | null`), `src/modules/territory/HouseholdDetail.tsx` (prop `householdOnly?: boolean`)
- Create: `src/modules/TerritoryTeam.vector.test.tsx`

**Interfaces:**
- Consumes: `getTerritoryContext` (24a) com o campo novo da D-21; `HouseholdForm`, `HouseholdDetail`, `TerritoryGate` (24a).
- Produces:
  - `householdTypesFor(m: { can_register: boolean; property_types?: string[] | null }): string[] | null` → `null` = todos os tipos (ACS, como hoje); lista = só esses (o agente).
  - `HouseholdForm({ …, propertyTypes })`: com lista, o `<select>` de tipo só oferece esses códigos (rótulos do `DOMAINS.tipo_de_imovel`) e o tipo residencial nunca aparece, nem na edição.
  - `HouseholdDetail({ …, householdOnly })`: sem "Nova família", sem `PersonStart`/`RegistrationForm`, sem editar família; "Editar domicílio" só quando o tipo do imóvel está na lista (residencial fica só leitura).
  - `TerritoryTeam`: a microárea com `property_types` mostra `"Nesta microárea você cadastra só imóvel não residencial (ponto estratégico, comércio, terreno baldio, escola, creche, estabelecimento religioso e outros)."` e passa a lista às duas telas.
  - `ERRORS.cbo_not_allowed` (24a) passa a `"só o ACS da microárea, enfermeiro ou médico da equipe grava o cadastro — o agente de endemias, só imóvel não residencial"`.

- [ ] **Step 1: Testes que falham** — `src/modules/TerritoryTeam.vector.test.tsx` (contexto do 24a com a microárea `ma1` atribuída, `can_register: true`, `property_types: [ "2", "3", "4", "5", "6", "12", "13" ]`):

```tsx
it("agente: novo domicílio só com os tipos não residenciais; o corpo leva o tipo escolhido", async () => {
  // "Novo domicílio" → options de Tipo = os 7 rótulos, sem "Domicílio" ("1") nem os residenciais; createHousehold com property_type "4"
});
it("agente: sem família nem cadastro individual; residencial só leitura", async () => {
  // HouseholdDetail de um imóvel "4": sem "Nova família" nem a busca de pessoa; de um imóvel "1": sem "Editar domicílio"
});
it("ACS (property_types ausente ou null): nada muda", async () => { /* os 14 tipos, "Nova família" presente */ });
it("403 cbo_not_allowed mostra a frase nova", async () => { /* … */ });
```

Em `territoryRegistry.test.ts`: `householdTypesFor({ can_register: true })` = `null`; com a lista, a lista.

- [ ] **Step 2–4:** rode e veja falhar (`src/modules/TerritoryTeam.vector.test.tsx src/modules/TerritoryTeam.test.tsx src/modules/territory/HouseholdForm.test.tsx src/lib/territoryRegistry.test.ts`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/territoryRegistry.ts src/lib/territoryRegistry.test.ts src/modules/TerritoryTeam.tsx src/modules/territory/HouseholdForm.tsx src/modules/territory/HouseholdDetail.tsx src/modules/TerritoryTeam.vector.test.tsx
$G commit -m "feat: let the endemic agent register non-residential households in assigned micro-areas" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: Suíte, build, varreduras, revisão e prova no navegador (api na 3059, Vite na 5220)

**Files:** nenhum novo (correção que a prova pedir entra na task dona, com teste).

- [ ] **Step 1: Suíte, tipos e build**

```bash
$R/bin/test dashboard -w dashboard-mod35a
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **23 arquivos de teste novos** (os 22 abaixo e `src/modules/TerritoryTeam.vector.test.tsx`): `src/lib/api.vectorControl.test.ts`, `src/lib/vectorErrors.test.ts`, `src/lib/vectorInspection.test.ts`, `src/lib/vectorControl.test.ts`, `src/shell/modules.vectorControl.test.ts`, `src/modules/vectorControl/VectorGate.test.tsx`, `src/modules/homeVisits/VisitForm.vector.test.tsx`, `src/modules/homeVisits/VisitDetail.vector.test.tsx`, `src/modules/HomeVisitDay.due.test.tsx`, `src/modules/FieldOvitraps.test.tsx`, `src/modules/vectorControl/VectorMapPanel.test.tsx`, `src/modules/VectorBlocks.test.tsx`, `src/modules/VectorSettings.test.tsx`, `src/modules/VectorCycles.test.tsx`, `src/modules/VectorOvitraps.test.tsx`, `src/modules/VectorSamples.test.tsx`, `src/modules/VectorLiraa.test.tsx`, `src/modules/VectorTransmission.test.tsx`, `src/modules/VectorTasks.test.tsx`, `src/modules/VectorReports.test.tsx`, `src/modules/attendance/UnitForm.vector.test.tsx`, `src/modules/VectorStockShortfalls.test.tsx` (os do 24a/24b/24c/27a ganham casos, não arquivo). Contagem que dobra = artefato de build descoberto pelo vitest; erro não tratado com tudo verde = mock sem `fetchCurrentSession`.

- [ ] **Step 2: Varreduras**

```bash
cd $W
O="src/modules/vectorControl src/modules/Vector*.tsx src/modules/FieldOvitraps.tsx src/modules/homeVisits/Inspection*.tsx src/lib/vectorControl.ts src/lib/vectorInspection.ts src/lib/vectorErrors.ts"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
grep -rnE "useQuery\(.*getTransmissionCaseAddress|setQueryData\(.*address" src/modules/vectorControl && echo "ERRO: endereço no cache" || echo ok-address
sed -n '/Vigilância ambiental — endemias (módulo 35a/,$p' src/lib/api.ts | grep -nE "note|reason|address|egg_count|household_ids" | grep -E "withQuery|hvQuery|\?\$\{" && echo "revisar: URL" || echo ok-url
for f in $(grep -rlE 'vi\.mock\("\.\./(\.\./)?lib/api", async' src/modules/vectorControl src/modules/Vector*.test.tsx src/modules/FieldOvitraps.test.tsx src/modules/HomeVisitDay.due.test.tsx src/modules/homeVisits/*.vector.test.tsx 2>/dev/null); do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-address`, `ok-url`, `ok-mocks`, sem `ERRO`.

- [ ] **Step 3: Conferência do api** — repita a Task 0 Step 4 contra o api final; forma diferente vira ajuste na task dona antes da prova.

- [ ] **Step 4: Revisão do ramo** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, nas regras portadas (Task 2, contra o `VectorControl::InspectionRules` do api), na tabela de frases, no endereço do caso e no domínio novo do 27a.

- [ ] **Step 5: Prova no navegador** — **só com autorização do usuário** para migrar os bancos de dev das cidades (migrações `20261116900001`–`20261116900003`) e rodar a semente (`lib/vector_control_crew.rb`, só Curitiba). Peça antes de qualquer `db:migrate` ou `db:seed`. **Ordem da Q6(4):** a semente liga `vector_control`; ela só roda depois que a sessão do campo confirmar o APK do 35a no emulador (o app da v1.2.0 quebra com as entidades novas) — combine pela sessão "Próximos módulos".

Com a autorização: api do worktree `apps/api/.claude/mod35a` na **3059**; Vite do worktree na rede do compose na **5220**, com proxy para `http://api:3059` (o comando da prova do 24b, trocando porta e worktree); abra `curitiba.localhost:5220/dashboard/`. Logins da semente (§18; senha e TOTP das outras contas de dev podem ser mostrados, nunca os de staging). Um caminho por requisito do edital:

1. **Boletim completo** (`ace@curitiba.demo`, preset mobile 375×812): Folha do dia → "Imóveis a visitar" → "Registrar visita" → boletim LI+T trabalhado, A2 com larva, focal com larvicida, uma amostra, ação educativa → "Etiquete os tubitos: …"; no detalhe da visita, motivos 34, 35, 36/37 (a ficha 8 derivada) e a etiqueta.
2. **Baixa e falta conciliada:** na posição do local da UVZ (`stock_keeper` da semente), o movimento "aplicação no campo (boletim)"; um boletim com produto sem lote → em "Faltas de insumo de campo" (`endemias@curitiba.demo`), a falta "sem lote do produto" conciliada com nota.
3. **Vencidos por risco:** a coordenação põe a cadência `red` em 30 dias (step-up) e marca um imóvel trabalhado há 40 dias como `red`; o imóvel aparece em "Imóveis a visitar" do agente como "vencido".
4. **Ovitrampa:** o agente instala (TCLE) e coleta uma palheta em "Ovitrampas de campo"; `entomologia@curitiba.demo` conta os ovos; em "Índices e mapa", o IPO da semana e o ponto no mapa; a contagem acima do alerta abre a tarefa de intensificação em "Tarefas de endemias".
5. **LIRAa:** o levantamento censitário da semente: itens com a substituição de um imóvel fechado; o PDF e o CSV baixam.
6. **Bloqueio:** uma notificação de dengue confirmada na fila do 25 (usuário da vigilância da semente) propõe o bloqueio; a coordenação confirma (step-up) com a área sugerida, cria um ciclo de UBV e vê o mapa da área sem o caso; no caso sem quarteirão, "Ver endereço do caso" pede step-up e o endereço some ao fechar.
7. **Amostra muda o IIP:** o resultado de uma amostra pendente do LIRAa (`entomologia@curitiba.demo`) tira o "provisório" ou muda o IIP.
8. **Relatórios em CSV:** os sete baixam; `municipal_admin` só vê "Relatórios de endemias".
9. **Boletim offline** é da prova do campo (emulador), não desta.

Também: o agente cadastra um ponto estratégico (tipo 4) em "Território da equipe" e registra nele um boletim de atividade PE (Q13(1)); unidade "UVZ Curitiba" com o tipo novo no cadastro de unidades; o local "Controle vetorial" na tela de locais do 27a; só teclado nos passos 1 e 6; `resize_window` preset desktop ao terminar. Anote cada passo; achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai ao usuário (não vira tarefa).

---

### Task 20: Entrega — board, relato e parada antes do merge

**Files:** nenhum.

- [ ] **Step 1: Board** — F-35.2 a F-35.9 → **Done** se o api já terminou o deles; F-35.1 espera também o maintenance (o último app move). **Verified** só depois da prova da Task 19, com o usuário.
- [ ] **Step 2: Suíte completa do api** — não é desta sessão: o pedido vai à sessão "Testes" no formato `api · mod35a/feat/mod-35a-vector-control · --full · mudança transversal do 35a` pela sessão dona do api. A suíte inteira do dashboard neste worktree (Task 19 Step 1) basta para o dashboard.
- [ ] **Step 3: Pare e peça autorização** para merge e push (ordem do §19: depois do api). Antes do push, `$G -C $W log --oneline origin/main..HEAD` e `origin/main..main` (sessões paralelas). Worktree: `$G -C $R/apps/dashboard worktree remove $W` só depois do merge.

**Relato:** `dashboard · dashboard-mod35a/feat/mod-35a-vector-control · Task 20 · suíte <n>/<n>, build ok, varreduras ok, prova <passos> · cards movidos · aguardando autorização de merge`.

---

## Divergências propostas ao contrato

> **Todas decididas** no §20 do contrato (docs `aa4f3c5`); a Q de cada uma está no fim da linha. O texto original fica como registro; onde a decisão mudou a proposta, vale a decisão (e o plano já a aplica).

Cada uma diz o que a tela precisa, a evidência e a resolução recomendada. Se o api adotar outra forma, vale a do api (Task 0 Step 4), e o plano ajusta tipo, fixture e teste.

- **D-1 — Formas das respostas e dos corpos não fixadas.** O §12 lista as rotas, mas não as formas de leitura nem os corpos de escrita (só a visita, §3, e os itens do app, §13, têm forma). **Recomendação:** adotar os tipos e corpos da Task 1 (`CityBlock`, `VectorHousehold`, `DueItem`, `CycleSummary`, `VectorParameterVersion`, `VectorProduct`, `VectorSettings`, `Ovitrap`, `OvitrapReading`, `VectorTask`, `LiraaSurvey`/`LiraaStratum`/`LiraaItem`, `TransmissionBlock`/`TransmissionCase`, `VectorSample`, `VectorShortfall`; corpos `{ household_ids, city_block_id }`, `{ items }` na cadência, `{ values }` nos parâmetros, `{ stock_location_id }`, `{ strata: [{ name, mode, block_ids }] }`, `{ center_block_id, block_ids }`, `{ reason }`, `{ starts_on, ends_on }` (o número do ciclo é o próximo, dado pelo api), `{ to_transmission_block_id }`, `{ state, note }`, `{ egg_count, count_note? }`, `{ examined_larvae, aegypti_larvae, albopictus_larvae, result_note? }`, `{ note }`); decimais em texto como no 27a; listas `{ items, next_before }`; `GET /due` ordenado pelo motivo do §5 e depois por `due_since`. **Decidida (Q8 — aceita; o `/due` põe os nunca trabalhados (`due_since` nulo) primeiro).**
- **D-2 — Domínios do boletim para o dashboard.** O formulário precisa dos rótulos de atividade, situação, grupos, métodos e risco, dos produtos ativos com `uses` e `charge_size`, e de `ovitrap_collection_days_min`/`_max`; hoje só a referência do field-sync os traz (§13), e `GET /products` é da coordenação. **Recomendação:** `GET /vector_control/domains` → exatamente o `vector_control_domains` da referência (o mesmo serializador), para o agente, a coordenação e o laboratório. **Decidida (Q8 — aceita).**
- **D-3 — Perfil vetorial do imóvel para o agente.** A regra 10 (quarteirão só se faltar) e a instalação de ovitrampa (`household_without_block`) dependem do perfil, que nenhuma rota do agente devolve. **Recomendação:** o `<household>` de `GET /territory/households/:id` ganha `vector` com a forma do `<household>.vector` do field-sync (só com o gate utilizável; `null` sem perfil). O bairro vem de `address.neighborhood_id` da versão atual (o mesmo `household_versions.neighborhood_id` do §2). **Decidida (Q8 — **substituída:** rota nova `GET /vector_control/households/:id` com o perfil na forma do `<household>.vector`, para o agente nas microáreas dele e a coordenação; o `TerritoryRegistry::Json.household` do 24a **não** muda (aplicado nas Tasks 1, 5 e 7: `getHouseholdVectorProfile`, resposta `{ id, neighborhood_id, vector }` pela Q13(5))).**
- **D-4 — `households_count` no quarteirão.** O editor de estratos precisa de N antes do sorteio (faixas do §7), e a desativação avisa `block_in_use`. **Recomendação:** `CityBlock.households_count` = imóveis ativos com o quarteirão no perfil. **Decidida (Q8 — aceita).**
- **D-5 — Nenhuma rota dá coordenada para os mapas.** §6, §8 e §15 prometem mapas por risco, contagem e área, mas as rotas do §12 não trazem `location`, nem o `map` da cidade (Q14 do 24c). **Recomendação:** `GET /vector_control/map_points?layer=risk&neighborhood_id=` | `layer=ovitraps&neighborhood_id=&epi_week=` | `layer=transmission&transmission_block_id=` → `{ map: <manifesto> | null, points: [{ id, latitude, longitude, category, label }] }`; a categoria é do api (a faixa de ovos sai dos parâmetros, não do código da tela); `label` sem pessoa (número do quarteirão, da ovitrampa, contagem e semana); a camada do bloqueio é a dos imóveis da área, sem o caso e sem trilha. **Decidida (Q8 e Q13(7) — categorias as do plano do api: `risk` `risk_class`/`none`, `ovitraps` `negative`/`positive`/`alert`/`not_counted`, `transmission` só `area`; ler pontos grava a trilha `territory_map`, por isso `NOMINAL_QUERY`).**
- **D-6 — Resumo do ciclo por bairro sem rota.** O §5 descreve os números por ciclo e bairro, mas o relatório `situations` é por classe de imóvel. **Recomendação:** `GET /vector_control/cycles/:id` → `CycleSummary` (ciclo + `neighborhoods[]` com a visitar, visitados, trabalhados, fechados, recusados, desocupados, recuperados, `pending_percent` e imóveis cadastrados). **Decidida (Q8 — aceita).**
- **D-7 — Filtros e corpo das ovitrampas.** `GET /ovitraps` não diz o escopo: o agente lê por microárea, a coordenação e o laboratório por bairro. **Recomendação:** `?micro_area_id=` (agente, escopo do 24b) ou `?neighborhood_id=` (coordenação e laboratório), e `status`; a instalação pelo dashboard manda o corpo do item do app com `household_id` e `client_uuid` (idempotente como a visita). **Decidida (Q8 — aceita).**
- **D-8 — O almoxarife não acha o produto nas telas do 27a.** A entrada do 27a busca item por `POST /pharmacy/items/search { q }` (só CATMAT), e `GET /vector_control/products` é da coordenação. **Recomendação:** a busca aceita `domain: "vector_control"` e devolve os produtos ativos como `StockItem` (`item_kind: "vector_control"`, `vector_product_id`, `catalog_item_id: null`, `dispensing_unit` = `unit`); a entrada manda `vector_product_id` no lugar de `catalog_item_id`, sem `package`. Os mínimos por produto ficam fora do dashboard no 35a. **Decidida (Q8 — aceita).**
- **D-9 — Quem lê as faltas e onde a tela fica.** O §4 diz quem concilia, não quem lê nem com que filtro. **Recomendação:** `GET /vector_control/stock_shortfalls?state=open|reconciled&location_id=`; a coordenação lê todas (inclusive `no_location`, sem local); o `stock_keeper` só as do seu local; a tela fica no grupo Farmácia, ao lado das faltas da UPA. **Decidida (Q8 — aceita).**
- **D-10 — Transições sem código.** Confirmar, descartar ou fechar bloqueio fora do status, fechar levantamento que não está `drawn`, fechar tarefa que não está `open`, novo ciclo de UBV em bloqueio não `active` e mover caso para bloqueio fechado não têm código; nem a falta de `count_note`/`result_note` na correção. **Recomendação:** `409 status_changed` (com `path: "status"`) para toda transição fora do status (a tela relê uma vez; `survey_not_draft` continua para o sorteio e o PATCH), e `422 invalid_request` com `path` `count_note`/`result_note`/`note` para nota ausente. A frase de `status_changed` já está na tabela (Task 2). **Decidida (Q9 — aceita com outro código: `409 invalid_transition` (o do 25), `path` `status`, com a frase que era do `status_changed`; nota ausente `422 invalid_request` com `path` `count_note`/`result_note`/`note`).**
- **D-11 — `path` das regras do §3 e contagem de dias.** O contrato fixa o `path` de algumas regras (1, 5, 7, 9) e não das outras. **Recomendação:** adotar os `path` da Task 2 (`vector_inspection.situation`, `.activity`, `.liraa_draw_item_id`, `.transmission_block_id`, `.measures[i].<campo>`, `.entered_at`, `.city_block_id`; `inspection_not_worked` em cada lista não vazia), para o campo e o dashboard destacarem o mesmo campo. Também: visita sem domicílio com `city_block_id` ou `risk_class` → `vector_inspection_invalid` no campo; duração do ciclo de UBV contada com os dois dias (`ends_on − starts_on + 1`) e intervalo como `starts_on − ends_on` anterior; janela da palheta como `collected_on − installed_on` (2 e 11 dias recusam com o padrão). **Decidida (Q9 e Q10 — aceita; a tabela de `path` da Task 2 é a fonte única para api e contracts; boletim sem domicílio → `vector_inspection_invalid` em `household_id` (não mais no campo do quarteirão ou do risco); `path` da ovitrampa fixados na Q13(2); o primeiro erro da lista é o do api, Q13(10)).**
- **D-12 — Etiqueta definitiva só depois do 201 no dashboard.** O 24b troca o `client_uuid` depois de toda recusa 4xx (`useIdempotentSubmit`), e a etiqueta nasce dele (§9). **Recomendação:** o contrato registra que, no dashboard, a etiqueta mostrada antes de gravar é provisória e que a definitiva é a da visita gravada (`samples[].label`); o api não muda. Manter o uuid depois de 4xx foi descartado (quebra a regra do 24b). **Decidida (Q7(1) — aceita; e a colisão de etiqueta ganha o código `sample_label_taken` em `vector_inspection.samples[i]` (não `client_uuid_conflict`); 422 com uma frase só para app e dashboard, Q13(3)).**
- **D-13 — Formato do JSON dos relatórios.** O §10 dá as linhas de cada relatório, não a forma. **Recomendação:** uma forma para os sete: `{ name, provisional?, columns: [{ key, label, kind }], rows, notes? }`, com contagens como `SmallCount` (supressão do `Admin::SmallCount` nos `blocks`), para uma tabela só na tela e o CSV com as mesmas colunas. **Decidida (Q8 — aceita).**
- **D-14 — Semana epidemiológica.** O 35a usa `YYYYWW` (`ovitrap_readings.epi_week`, filtro `epi_weeks[]`), e o 25 usa `AAAA-SS` (`epiWeekOf`, `weekLabel`, `week.csv`). **Recomendação:** o 35a adota o `AAAA-SS` do 25 nas rotas e no field-sync (os helpers do dashboard e do api já existem); se o contrato mantiver `YYYYWW`, o cliente converte na borda (`toVectorWeek`) e nada mais muda. **Decidida (Q10 — aceita: semana exposta como `"YYYY-WW"` (gravada como inteiro `YYYYWW`, pelo `Surveillance::EpiWeek`); o `toVectorWeek` não existe).**
- **D-15 — Unidade `environmental_surveillance` nos seletores.** O §1 diz que a unidade "não dispensa, não tem fila e não tem leito", mas não diz se aparece nos seletores de check-in, agenda e encaminhamento. **Recomendação:** o cadastro só oferece o tipo com `vector_control` utilizável (como CEO e CAPS); os seletores não a filtram; se o api for recusar check-in nela, o contrato nomeia o código. **Decidida (Q8 — aceita).**
- **D-16 — Interruptor na sessão.** O §1 define o gate como habilitado + `HomeVisits::Gate.usable?`, mas não diz se a sessão lista `vector_control` sem a visita utilizável. **Recomendação:** a sessão lista as chaves ligadas (como hoje); o dashboard exige também `canUseHomeVisits` e trata o `403 feature_disabled` com a frase do 35a. **Decidida (Q8 — aceita).**
- **D-17 — Agente sem boletim.** O §3 diz que `vector_inspection` é opcional, sem dizer se o agente pode gravar visita sem ele. **Recomendação:** pode (a visita do 24b segue valendo); o dashboard oferece "Registrar boletim" ligado por padrão e o desfecho normal quando desligado. **Decidida (Q8 — aceita).**
- **D-18 — LIRAa e bloqueio escolhidos à mão.** A regra 4 exige o item do LIRAa ou o bloqueio, que o dashboard só conhece pela folha (`GET /due`). **Recomendação:** o contrato registra que, no dashboard, as atividades `liraa` e `transmission_block` só nascem de "Imóveis a visitar" com o vínculo do item; a visita aberta à mão não as oferece. **Decidida (Q8 — aceita).**
- **D-19 — Relatório de amostras do laboratório.** O §10 diz que o `vector_lab` lê "só ovitrampas e amostras", mas não há relatório de amostras na tabela. **Recomendação:** o laboratório lê o relatório `ovitraps`; "amostras" é a fila do §9 (`GET /samples`), que ele já lê; nenhum relatório novo. **Decidida (Q8 — aceita).**
- **D-20 — Número do caso no bloqueio.** O §8 manda mostrar "caso 1, 2…", sem dizer de onde vem o número. **Recomendação:** `cases[].number` dado pelo api (ordem de entrada no bloqueio, estável depois de mover caso), para o dashboard e o relatório numerarem igual. **Decidida (Q8 e Q13(6) — `cases[].number` é sequência da cidade, gravada e imutável).**
- **D-21 (decidida pela Q14, docs 792a78f: o campo `property_types` vem em `GET /territory/teams`, que o `getTerritoryContext` já lê) — Como a tela do 24a sabe que o agente só cadastra não residencial (nova, da Q13(1)).** A Q13(1) libera o cadastro, mas o `GET /territory/context` hoje só diz `can_register` por microárea ("o ACS só na dele"), e a sessão não traz CBO. Sem dado, a tela ofereceria tipo residencial e família ao agente (o api recusaria com 403). **Recomendação:** o contexto lista as microáreas atribuídas ao agente com `can_register: true` e `property_types: ["2","3","4","5","6","12","13"]` (ausente ou `null` = todos os tipos, como hoje); a lista vem do api, não do código da tela. Alternativa pior: cruzar o `id` da equipe com o `cbo` do `GET /home_visits/context`. **Aberta** — volta ao coordenador.

## Self-review

1. **Cobertura (dashboard do §15 e F-35.1–F-35.9):** grupo "Endemias" e itens fora dele (Task 4); `vector-blocks` com mapa por risco (Tasks 8–9, F-35.1); boletim na `VisitForm` e no `VisitDetail` com motivos derivados, medidas e amostras (Task 5, F-35.2, F-35.3, F-35.8); "Imóveis a visitar" (Task 6, F-35.4); ovitrampas do agente (Task 7) e da coordenação/laboratório com mapa por contagem (Task 12, F-35.5); `vector-settings` (Task 10, F-35.3); `vector-cycles` (Task 11, F-35.4); `vector-samples` (Task 13, F-35.8); `vector-liraa` (Task 14, F-35.6); `vector-transmission` com mapa por área e casos sem paciente (Task 15, F-35.7); `vector-tasks` (Task 16); `vector-reports` com CSV e PDF (Task 17, F-35.9); unidade, domínio do estoque e falta conciliada (Task 18, F-35.1, F-35.3); `FeatureKey`/`FEATURE_LABEL` (Task 1); roteiro do §18 (Task 19). Imóvel não residencial pelo agente no 24a (Task 18A, Q13(1)). O passo 9 do roteiro (app offline) e o maintenance (§16) não são deste repo.
2. **Regras do §3:** os onze números têm caso na Task 2 (1–10 em `checkInspection`, 11 é do api e aparece no detalhe, Task 5); as bordas do §19 que tocam a tela (N = 8.099/8.100/12.000/12.001/1.999/2.000, leitura com 2 e 11 dias, contagem 0, UBV com intervalo de 4) estão na Task 3.
3. **Tipos:** `VectorInspectionInput`, `VectorControlDomains`, `HouseholdVector`, `VectorStart` (Tasks 1–2) são os mesmos nas Tasks 5–7; `checkInspection`/`derivedReasons`/`sampleLabel`/`pruneInspection` (Task 2) governam a Task 5; `VectorMapPanel` (Task 8) serve às Tasks 9, 12 e 15; `ReportTable` (Task 17) substitui a tabela da Task 12; `vectorRefusals`/`isVectorStale` (Task 3) em todas as telas.
4. **Review Focus:** as cinco linhas têm teste na task dona (5; 5; 2 e 5; 15; 14).
5. **Proporção:** o código do plano é tipo, frase fixa e teste; os componentes vão por assinatura, props e comportamento.
