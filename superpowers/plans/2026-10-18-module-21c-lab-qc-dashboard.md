# Módulo 21c — Qualidade e insumos do laboratório municipal (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21c** (`feat/mod-21c-lab-qc`, porta de dev **3043**) com as rotas `/lab/admin/instruments`, `/lab/instruments`, `/lab/qc/*`, `/lab/eqa/*` e `/lab/inventory/*` do contrato e as das **Divergências** deste plano (fim do arquivo), os interruptores `lab_quality_control` e `lab_inventory` na sessão, a semente do 21c (aparelho com `hl7_equipment_id`, material de controle com dois níveis, lote com referência da bula, 20 corridas aprovadas para a sugestão, reagente com lote em uso) e o `fake-middleware` mandando controles. Ele é **mergeado antes** deste (contrato §10: contracts → api → dashboard → maintenance). As Tasks 0–16 não dependem dele (os testes usam dublês de `src/lib/api`); a prova da Task 17 depende.

**Goal:** Lado dashboard do subprojeto 21c: no grupo **"Laboratório"**, as telas de aparelhos; CIQ (materiais, lotes com código de barras por nível, referências da bula e do laboratório com aceite do RT, regras por analito com justificativa, corridas do dia por aparelho com digitação em vírgula decimal, gráfico de Levey-Jennings, bloqueios ativos com ação corretiva, exceções e revisões); a liberação do 21b mostrando `qc_blocked` com os itens e o atalho para a exceção do RT; CEQ (rodadas e resumo anual); estoque (itens, entrada de lote, movimentos, lote em uso por aparelho, alertas) e rastreio por lote e consumo estimado, os dois com CSV; e a marca "sem rastreio de reagente" ou o número do lote nos resultados da digitação e da liberação. O contador de bloqueios ativos no menu lê só a rota `/count`, que não grava trilha.

**Architecture:** O cliente HTTP ganha a seção "Qualidade e insumos do laboratório" no fim de `src/lib/api.ts` (tipos do contrato §1–§5 e §7 e as funções; na URL só ids, situação, nível, chave do analito, datas, mês e ano; números decimais como string com ponto). As regras de tela ficam puras em `src/lib/qc.ts` (CIQ e CEQ: rótulos, números com vírgula, validação dos formulários, modelo do Levey-Jennings, frases das recusas) e `src/lib/inventory.ts` (estoque: rótulos, validação, saldo, marca do lote, desvio). Peças comuns: o `LabGate` existente ganha o portão por interruptor (`requires`), `src/modules/lab/LabPick.tsx` escolhe o laboratório e diz se a pessoa é RT/substituto, `src/modules/lab/saveFile.ts` (extraído do `LabReports`) baixa CSV, `src/modules/inventory/ReagentLotTag.tsx` mostra o lote ou "sem rastreio". As telas novas ficam em `src/modules/` (`LabInstruments`, `LabQcSetup`, `LabQcRuns`, `LabQcChart`, `LabQcBlocks`, `LabEqa`, `LabInventory`, `LabTraceability`) com as peças em `src/modules/qc/`, `src/modules/eqa/` e `src/modules/inventory/`, registradas no grupo "Laboratório" (`modules.ts`, `catalog.ts`, `App.tsx`). O gráfico usa **Recharts 2** (`recharts@^2.13.0` no `package.json`, 2.15.4 instalado; já usado em `SeriesChart`, `Sparkline` e `BarMini`). Mudam o mínimo: `ReleaseReview`/`LabRelease` (409 `qc_blocked`, exceção, lote), `ResultEntryForm` (lote), `VersionForm`/`LabExamsPanel`/`labAdmin.ts` (aparelho da versão), `features.ts`. O dashboard nunca decide o que o api garante: a avaliação de Westgard, o bloqueio, quem é RT, o saldo e o lote em uso são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Recharts 2.15, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`, proxy `/lab` já existente), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-18-module-21c-lab-quality-supplies-design.md` (§2 decisões, §3 interruptores, §4 dados, §5 CIQ, §6 estoque, §8 papéis, §9 segurança, §10 testes, §12 F-IDs), `docs/adr/0038.md` e o contrato `docs/superpowers/plans/2026-10-18-module-21c-lab-qc-contracts.md` (§1–§5, §7, §9, §10), com a base do 21b (`2026-10-15-module-21b-lab-contracts.md`, inclusive §14 B1–B64) e do 21a (§15). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim (C1–C17); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): corrida `open` | `approved` | `warning` | `rejected`; referência `package_insert` | `laboratory`; regras `1_2s` (aviso) | `1_3s` | `2_2s` | `R_4s` | `4_1s` | `10x`; desfecho `ok` | `warning` | `reject`; revisão `pending` | `kept` | `rectify`; CEQ `adequate` | `inadequate`; item `reagent` | `calibrator` | `control` | `consumable`; origem `purchase` | `donation` | `loan`; movimento `receipt` | `opening` | `loss` | `expiry` | `adjustment`; lote de controle `in_use` | `closed`. Código desconhecido aparece cru, nunca some.
- Interruptores (contrato §8; spec §3): `lab_quality_control` liga CIQ e CEQ; `lab_inventory` liga o estoque; os dois exigem `municipal_lab` (o api garante) e são independentes. Aparelhos aparecem com qualquer um dos dois (Divergência C13). Desligado, o dashboard é exatamente o de hoje: sem as telas novas no menu, sem a escolha de aparelho na versão do exame, sem marca de lote nos resultados. 403 `feature_disabled` → `QC_DISABLED` / `INVENTORY_DISABLED`. O dashboard nunca escreve interruptor; as duas chaves ganham só rótulo em `features.ts`.
- **Nenhum polling em rota que grava trilha** (A46 do 21a; lição do contrato §14 do 21b). Só a contagem de bloqueios ativos (`GET /lab/qc/blocks/active/count`, sem trilha) relê a cada minuto (`refetchInterval: 60_000`, `gcTime: 0`). Listas e detalhes (`/lab/qc/runs`, `/lab/qc/blocks`, `/lab/qc/reviews`, `/lab/inventory/lots/:id/trace`, `/lab/inventory/consumption`) só leem quando a tela abre ou depois de uma ação; nenhum `refetchInterval` nem `refetchOnWindowFocus` nelas. O rastreio por lote (trilha de leitura) só lê no clique de "Rastrear".
- Números digitados com **vírgula decimal**; ponto é recusado com `DOT_IN_NUMBER` (D11 do 21b; `normalizeNumber`/`hasDot` de `src/lib/lab.ts`). No corpo, decimais vão como string com ponto (`"98.5"`); quantidade de estoque é inteira. Na tela, todo decimal volta com vírgula (`fmtDecimal`).
- Recusar em vez de cortar: nenhum `maxLength` silencioso nos campos novos; texto longo demais é recusado com a frase do limite antes do envio.
- Na URL vão só ids, `status`, `level`, `analyte_key`, `YYYY-MM-DD`, `YYYY-MM`, `YYYY` e `with_balance` — nunca valor de controle, média, dp, justificativa, motivo, ação, código de barras de nível, número de lote ou quantidade.
- Valor de resultado e de controle, nome de paciente e PDF nunca em `console`, `localStorage`, `sessionStorage`, Analytics ou mensagem de erro. O rastreio por lote mostra número da requisição e código do exame, nunca nome de paciente (contrato §5).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; `POST …/close` e afins mandam `{}`). Escrita em `/lab/admin/*` (aparelhos) com step-up pelo `SensitiveAction` (o `Lab::Admin::BaseController` do api exige). As demais escritas do 21c não pedem step-up (o contrato não pede).
- Quem pode (spec §8; contrato §2–§5): `lab_staff` — corridas, ação corretiva, entradas, aberturas, perdas, lote em uso; `lab_releaser` — o de `lab_staff` + Levey-Jennings e CEQ; RT/substituto (`technical_lead_of` do contexto, Divergência C1) — materiais, lotes, referências, aceite da sugestão, regras, exceção e revisão; `municipal_admin` — aparelhos, materiais, itens, ajuste, CEQ, rastreio e consumo. A tela não oferece o que o papel não pode; o api recusa de novo (`missing_role`).
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--accent`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem `outline: "none"`, sem `textOverflow` sem `title`, sem `color: "#fff"` (guarda `src/theme/guards.test.ts`). Use os componentes existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `ActionButton`, `SensitiveAction`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts`); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date(NOW21C))` e `vi.useRealTimers()` no `afterEach`; a sessão de teste é criada **depois** do `setSystemTime`.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-21.25..F-21.34** do board #1 para **In Progress** (Task 0); cada card vai a **Done** quando a parte do dashboard dele termina **e** a do api já está Done (os cards são compartilhados; F-21.28 — controle pela HL7 — não tem parte no dashboard além do que a corrida mostra como "aparelho").

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod21c` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-21c-lab-qc` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod21c`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos antes de cada commit: `npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. O jsdom não tem `URL.createObjectURL` nem `ResizeObserver`: defina-os no teste que precisa (o Recharts mede o pai por `ResizeObserver`).
- **Prova técnica do Levey-Jennings (contrato §10, item 0b) — já feita ao escrever este plano:** com o Recharts 2.15.4 do worktree do 21b, um `LineChart` dentro de `ResponsiveContainer`, sob jsdom e um `ResizeObserver` que informa 600×260, renderiza **um `.recharts-reference-line-line` por `ReferenceLine`** (7 horizontais em −3…+3 dp + 1 vertical = 8) e o `dot` customizado sai como `<circle data-outcome="…">` (4 pontos, 2 `reject`). Os testes da Task 10 usam exatamente esses seletores. Se a execução achar outra versão do Recharts, refaça a prova com o teste da Task 10 antes de seguir.
- A prova (Task 17) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5196**, com proxy para o api do 21c na **3043**.

## Review Focus

1. **Valor de controle com ponto, sem vírgula, negativo ou recusado pelo api no meio de uma série.** O técnico digita cinco analitos de um nível e um deles sai "2.5". Espera-se: nada é enviado enquanto houver valor com ponto (frase `DOT_IN_NUMBER` no campo certo); "2,5" vai como `"2.5"`; negativo é aceito (valor de controle pode ser negativo); um 422 `no_target` no terceiro valor marca aquele campo, mantém os que ainda não foram e apaga só os já gravados. Teste: Task 9, "série de valores: ponto bloqueia tudo; 422 no meio marca o campo e mantém os não enviados".
2. **Liberação presa pelo CIQ com parte dos exames bloqueada.** A requisição tem glicose (bloqueada) e hemograma (livre). Espera-se: 409 `qc_blocked` não libera nada, lista o item bloqueado com analito e aparelho, oferece "Liberar sem os bloqueados" (desmarca só os bloqueados) e, só para o RT/substituto, "Liberar com exceção" com justificativa; depois da exceção, liberar de novo manda os dois itens. Testes: Task 12, "409 qc_blocked: lista, não libera e desmarca só os bloqueados" e "RT: exceção com justificativa e liberação de novo".
3. **Referência trocada no meio do período do gráfico.** Pontos de duas versões da referência no mesmo Levey-Jennings. Espera-se: o eixo é em desvios-padrão (cada ponto já vem com o z da sua versão, nunca reprojetado na média nova), uma linha vertical marca a troca com o número da versão, e a legenda diz qual versão está vigente e que há pontos de outra. Testes: Task 2, "troca de versão vira marca na série e aviso de versões misturadas"; Task 10, "duas versões: linha vertical na troca e aviso na legenda".
4. **Contador de bloqueios no menu.** Espera-se: lê só `/lab/qc/blocks/active/count` a cada minuto, nunca a lista de bloqueios (rota com dado de corrida) nem a de revisões; com o interruptor desligado ou sem o papel, não lê nada; erro do contador não some com o menu (só não mostra número). Testes: Task 4, "relê só /count a cada minuto e nunca a lista" e "sem o interruptor ou sem papel (enabled false): nenhuma leitura".
5. **Saída de estoque maior que o saldo, lote vencido ou ajuste de quem não é admin.** Espera-se: bloqueado antes do envio com a frase (saldo na frase); o ajuste nem aparece para quem não é admin; 422 `insufficient_balance` do api (alguém abriu uma embalagem antes) relê o saldo e mantém o formulário preenchido. Testes: Task 3, "saída maior que o saldo e ajuste sem admin"; Task 14, "422 insufficient_balance: relê os lotes e mantém o formulário".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.qc.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/qcFixtures.ts` | tipos e cliente HTTP do 21c; `reagent_lot` no resultado; `technical_lead_of` no contexto; `id`/`analytes` em `LabExamRef`; `lab_instrument_id` na versão; interruptores; dados de teste | 1 |
| `src/lib/qc.ts`, `src/lib/qc.test.ts` | regras de tela do CIQ e do CEQ (rótulos, números, formulários, Levey-Jennings, recusas) | 2 |
| `src/lib/inventory.ts`, `src/lib/inventory.test.ts` | regras de tela do estoque (rótulos, formulários, saldo, marca do lote, desvio, recusas) | 3 |
| `src/shell/modules.ts`, `src/shell/modules.lab.test.ts`, `src/shell/modules.qc.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/hooks/useQcBlockCount.ts`, `src/hooks/useQcBlockCount.test.tsx` | oito telas no grupo "Laboratório", contador de bloqueios | 4 |
| `src/modules/lab/LabGate.tsx`, `src/modules/lab/LabPick.tsx`, `src/modules/lab/saveFile.ts`, `src/modules/LabReports.tsx`, `src/modules/inventory/ReagentLotTag.tsx`, `src/modules/lab/LabGate.qc.test.tsx` | portão por interruptor, escolha do laboratório e RT, download, marca do lote | 5 |
| `src/modules/LabInstruments.tsx` (+ teste), `src/lib/labAdmin.ts`, `src/modules/labAdmin/VersionForm.tsx`, `src/modules/labAdmin/LabExamsPanel.tsx`, `src/modules/labAdmin/VersionForm.instrument.test.tsx`, `src/App.tsx` | aparelhos; aparelho na versão do exame | 6 |
| `src/modules/LabQcSetup.tsx` (+ teste), `src/modules/qc/MaterialsPanel.tsx`, `src/modules/qc/LotsPanel.tsx`, `src/App.tsx` | materiais e lotes com código de barras por nível | 7 |
| `src/modules/qc/TargetsPanel.tsx` (+ teste), `src/modules/qc/RulesPanel.tsx` (+ teste), `src/modules/LabQcSetup.tsx` | referências (bula, sugestão, aceite) e regras por analito | 8 |
| `src/modules/LabQcRuns.tsx` (+ teste), `src/modules/qc/QcValueForm.tsx`, `src/modules/qc/QcRunCard.tsx`, `src/App.tsx` | corridas do dia por aparelho e digitação | 9 |
| `src/modules/qc/LeveyJennings.tsx` (+ teste), `src/modules/LabQcChart.tsx` (+ teste), `src/App.tsx` | gráfico de Levey-Jennings | 10 |
| `src/modules/LabQcBlocks.tsx` (+ teste), `src/modules/qc/CorrectiveActionForm.tsx`, `src/modules/qc/ReviewsPanel.tsx`, `src/App.tsx` | bloqueios, ação corretiva, exceções concedidas, revisões | 11 |
| `src/modules/qc/QcBlockedPanel.tsx`, `src/modules/qc/QcExceptionForm.tsx`, `src/modules/lab/ReleaseReview.tsx`, `src/modules/LabRelease.tsx`, `src/modules/lab/ResultEntryForm.tsx`, `src/modules/LabRelease.qc.test.tsx`, `src/modules/lab/ResultEntryForm.lot.test.tsx` | `qc_blocked` na liberação, exceção do RT, lote nos resultados | 12 |
| `src/modules/LabEqa.tsx` (+ teste), `src/modules/eqa/RoundForm.tsx`, `src/App.tsx` | CEQ: rodadas, relatório anual e resumo | 13 |
| `src/modules/LabInventory.tsx` (+ teste), `src/modules/inventory/SuppliesPanel.tsx`, `src/modules/inventory/SupplyLotsPanel.tsx`, `src/App.tsx` | itens, entrada de lote, movimentos | 14 |
| `src/modules/inventory/InUsePanel.tsx` (+ teste), `src/modules/inventory/AlertsPanel.tsx`, `src/modules/LabInventory.tsx` | lote em uso por aparelho, alertas | 15 |
| `src/modules/LabTraceability.tsx` (+ teste), `src/App.tsx` | rastreio por lote e consumo estimado, com CSV | 16 |
| — | suíte, build, revisão, prova no navegador | 17 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (valor, média, dp, justificativa, quantidade, código de barras de nível) (Task 1); regras puras com tabela de casos, inclusive o modelo do Levey-Jennings com troca de versão e o saldo (Tasks 2 e 3); o contador por relógio falso, provando que só `/count` relê e que nada lê sem interruptor ou papel (Task 4); cada tela pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel e interruptor, cobrindo o caminho feliz, cada recusa que muda a tela (422 de validação, 409 `qc_blocked`, `feature_disabled`, `missing_role`) e o papel sem acesso (Tasks 5–16); o gráfico com `ResizeObserver` dimensionado e os seletores da prova técnica (`.recharts-reference-line-line`, `[data-outcome]`) mais a tabela acessível dos pontos marcados (Task 10). A prova final é no navegador, contra o api real na 3043 com o `fake-middleware` mandando controles (Task 17).

---

### Task 0: Board, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard; o board #1 (`rotasaude`).
- Produces: cards F-21.25..F-21.34 em In Progress; worktree `$W` na branch `feat/mod-21c-lab-qc`; a base anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json
gh project field-list 1 --owner rotasaude --format json
```

Na saída, ache os itens de título `F-21.25` a `F-21.34` (o `id` `PVTI_…`) e o id da opção **In Progress** do campo Status (`PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`). Para cada card que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id PVTI_xxx --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os dez cards em In Progress. Se algum F-21.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:package.json | grep -n '"recharts"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n "^export interface LabContext\|^export interface LabExamRef\|effective: boolean; notified?: boolean;\|^export interface LabExamVersion {\|^export async function setGalRequisition\|^function readBase64\|^const labPost"
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "municipal_lab" | "lab_hl7_inbound" | "fake_middleware";\|fake_middleware: "Middleware'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '| "lab-admin" | "lab-reports";\|{ id: "lab-release", label: "Liberação de laudos", icon: "✔" },\|const isReleaser = roles.includes("lab_releaser");\|if (item.id === "lab-reports") return\|^export function withLabCriticalCount'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '"lab-release": {\|IconUser, IconUserCircle, IconUsers, IconUsersGroup'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'export function shellGroups\|const labCritical = labCriticalQuery.data ?? 0;\|case "lab-reports": return <LabReports />;'
$G -C $R/apps/dashboard show origin/main:src/modules/lab/LabGate.tsx | grep -n "export function LabGate\|const switchOn = canUseLab(user);"
$G -C $R/apps/dashboard show origin/main:src/modules/lab/ReleaseReview.tsx | grep -n 'if (reason === "invalid_transition") { onConflict(labError(err)); return; }\|export function ReleaseReview\|{error && <p role="alert" style={alert}>{error}</p>}'
$G -C $R/apps/dashboard show origin/main:src/modules/lab/ResultEntryForm.tsx | grep -n "{r.effective && <> <Tag tone=\"ok\">vale</Tag></>}"
$G -C $R/apps/dashboard show origin/main:src/modules/LabReports.tsx | grep -n "^function saveFile\|^const BLOB_TTL_MS"
$G -C $R/apps/dashboard show origin/main:src/lib/labAdmin.ts | grep -n "^export interface VersionDraft\|instrumentTestCode: v?.instrument_test_code\|^export function versionBody"
$G -C $R/apps/dashboard show origin/main:src/modules/labAdmin/VersionForm.tsx | grep -n "export function VersionForm\|Código do exame no aparelho (HL7)\|run={async () => { await createLabExamVersion"
$G -C $R/apps/dashboard show origin/main:src/modules/labAdmin/LabExamsPanel.tsx | grep -n "<VersionForm key={versioning.id}"
$G -C $R/apps/dashboard show origin/main:src/shell/modules.lab.test.ts | grep -n "const LAB_IDS\|com as nove telas"
```

Expected: cada `grep` acha a linha pedida (`recharts` em `^2.13.0`). Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod21c -b feat/mod-21c-lab-qc origin/main
cd $R/.worktrees/dashboard-mod21c && npm ci --silent && node -e "console.log(require('recharts/package.json').version)" && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: a versão do Recharts (2.15.x), tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 17 espera a base mais **21 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 21c** (só informativo; as Tasks 1–16 não dependem dele)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-21c-lab-qc
docker exec -w /rails/.claude/mod21c api-dev bin/rails routes 2>/dev/null | grep -E " /lab/(qc|eqa|inventory|instruments|admin/instruments)" | head -80
```

Expected: a branch e as rotas. Compare com o contrato e com as Divergências deste plano; diferença vira nota para a Task 17.

---

### Task 1: Tipos, cliente HTTP, interruptores e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (`LabContext`, `LabExamRef`, `LabResultRow`, `LabExamVersion`; seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/test/qcFixtures.ts`
- Test: `src/lib/api.qc.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `binaryFetch`, `readBase64`, `labPath`, `labQuery`, `labPost`, `SignatureFile` (existentes em `api.ts`); `labUser`, `labContext`, `LAB_PROVIDER` (`src/test/labFixtures.ts`).
- Produces (em `src/lib/api.ts`): os tipos `QcRunStatus`, `QcTargetSource`, `QcRule`, `QcOutcome`, `QcReviewDecision`, `EqaAdequacy`, `SupplyKind`, `SupplyOrigin`, `SupplyMovementKind`, `QcLotStatus`, `ReagentLotRef`, `LabInstrument`, `LabInstrumentInput`, `QcAnalyteRef`, `QcMaterial`, `QcMaterialInput`, `QcLot`, `QcLotInput`, `QcTarget`, `QcTargetSet`, `QcTargetInput`, `QcTargetKey`, `QcSuggestion`, `QcValue`, `QcRun`, `QcBlock`, `QcValueInput`, `QcValueResult`, `QcChartPoint`, `QcChart`, `QcChartQuery`, `QcRuleSet`, `QcRuleInput`, `QcReview`, `QcException`, `QcBlockedItem`, `EqaItem`, `EqaRound`, `EqaRoundInput`, `EqaSummaryRow`, `LabSupply`, `LabSupplyInput`, `LabSupplyLot`, `LabSupplyLotInput`, `LabSupplyMovement`, `LabSupplyMovementInput`, `LabSupplyInUse`, `LabSupplyInUseInput`, `LabExpiringLot`, `LabLowSupply`, `LabInventoryAlerts`, `LabLotTrace`, `LabConsumptionRow`, `LabConsumptionQuery`; a constante `QC_RULES`; as funções (assinaturas exatas no Step 3). `LabContext` ganha `technical_lead_of?: string[]`; `LabExamRef` ganha `id?: string` e `analytes?: { key: string; name: string }[]`; `LabResultRow` ganha `reagent_lot?: ReagentLotRef | null`; `LabExamVersion` ganha `lab_instrument_id?: string | null`. `FeatureKey` ganha `lab_quality_control` e `lab_inventory`.
- Produces (em `src/test/qcFixtures.ts`): `NOW21C`, `TODAY21C`, `QC_FEATURES`, `qcUser(roles, over)`, `qcContext(over)`, `GLU_EXAM`, `CHOL_EXAM`, `instrument(over)`, `qcMaterial(over)`, `qcLot(over)`, `qcTarget(over)`, `qcSuggestion(over)`, `qcValue(over)`, `qcRun(over)`, `qcBlock(over)`, `qcReview(over)`, `qcException(over)`, `qcRules(over)`, `qcChart(over)`, `eqaRound(over)`, `supply(over)`, `supplyLot(over)`, `supplyMovement(over)`, `inUse(over)`, `SizedResizeObserver`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.qc.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  acceptQcSuggestion, closeQcLot, countActiveQcBlocks, createInstrument, decideQcReview, fetchConsumptionCsv, fetchSupplyLotTraceCsv,
  getQcChart, grantQcException, listLabInstruments, listQcLots, listSupplyLots, recordQcValue, recordSupplyMovement, saveQcRules,
  traceSupplyLot, updateInstrument, uploadEqaReport
} from "./api";

function ok(body: unknown, init: ResponseInit = {}): Promise<Response> {
  return Promise.resolve(new Response(typeof body === "string" ? body : JSON.stringify(body), {
    status: 200, headers: { "Content-Type": "application/json" }, ...init
  }));
}

describe("cliente do 21c (contrato §2–§5)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => {
    fetchMock = vi.fn(() => ok({}));
    vi.stubGlobal("fetch", fetchMock);
  });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => fetchMock.mock.calls[i][1] as RequestInit;
  const headersOf = (i = 0) => initOf(i).headers as Record<string, string>;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));

  it("aparelhos: leitura por laboratório; escrita do admin com JSON", async () => {
    fetchMock.mockImplementationOnce(() => ok({ instruments: [ { id: "in1" } ] }));
    expect(await listLabInstruments("pv9")).toEqual([ { id: "in1" } ]);
    expect(urlOf().pathname).toBe("/lab/instruments");
    expect(urlOf().searchParams.get("provider_id")).toBe("pv9");
    const input = { provider_id: "pv9", name: "BS-200", manufacturer: "Mindray", model: "BS-200", serial_number: "SN1", sector: "biochemistry", hl7_equipment_id: null, active: true };
    await createInstrument(input);
    expect(urlOf(1).pathname).toBe("/lab/admin/instruments");
    expect(initOf(1).method).toBe("POST");
    expect(headersOf(1)["Content-Type"]).toBe("application/json");
    expect(bodyOf(1)).toEqual(input);
    await updateInstrument("in1", { ...input, active: false });
    expect(urlOf(2).pathname).toBe("/lab/admin/instruments/in1");
    expect(initOf(2).method).toBe("PATCH");
  });

  it("lotes de controle: filtro por laboratório e situação; encerrar manda {}", async () => {
    fetchMock.mockImplementationOnce(() => ok({ lots: [] }));
    await listQcLots({ providerId: "pv9", status: "in_use" });
    expect(urlOf().pathname).toBe("/lab/qc/lots");
    expect(urlOf().searchParams.get("provider_id")).toBe("pv9");
    expect(urlOf().searchParams.get("status")).toBe("in_use");
    await closeQcLot("ql1");
    expect(urlOf(1).pathname).toBe("/lab/qc/lots/ql1/close");
    expect(initOf(1).method).toBe("POST");
    expect(bodyOf(1)).toEqual({});
  });

  it("valor de controle e regras: tudo no corpo, nada na URL", async () => {
    fetchMock.mockImplementationOnce(() => ok({ value: {}, run: {}, block: null }));
    await recordQcValue({ run_id: null, instrument_id: "in1", lot_id: "ql1", level: "normal", analyte_key: "glicose", value: "98.5" });
    expect(urlOf().pathname).toBe("/lab/qc/values");
    expect(urlOf().search).toBe("");
    expect(bodyOf()).toEqual({ run_id: null, instrument_id: "in1", lot_id: "ql1", level: "normal", analyte_key: "glicose", value: "98.5" });
    const rules = { "1_2s": true, "1_3s": true, "2_2s": true, "R_4s": true, "4_1s": false, "10x": false };
    await saveQcRules({ instrument_id: "in1", analyte_key: "glicose", rules, justification: "aparelho novo em validação, 4-1s gera falso alarme" });
    expect(urlOf(1).pathname).toBe("/lab/qc/rules");
    expect(initOf(1).method).toBe("PUT");
    expect(headersOf(1)["Content-Type"]).toBe("application/json");
    expect(urlOf(1).search).toBe("");
  });

  it("sugestão aceita, exceção e revisão: chave e texto no corpo", async () => {
    await acceptQcSuggestion({ lot_id: "ql1", level: "normal", analyte_key: "glicose", instrument_id: "in1", reason: "20 corridas aprovadas no lote" });
    expect(urlOf().pathname).toBe("/lab/qc/targets/suggestions/accept");
    expect(bodyOf().reason).toBe("20 corridas aprovadas no lote");
    await grantQcException({ block_id: "qb1", item_id: "it2", justification: "paciente internado, médico ciente" });
    expect(urlOf(1).pathname).toBe("/lab/qc/exceptions");
    expect(urlOf(1).search).toBe("");
    await decideQcReview("rv1", { decision: "kept", note: "valor coerente com o anterior" });
    expect(urlOf(2).pathname).toBe("/lab/qc/reviews/rv1");
    expect(bodyOf(2)).toEqual({ decision: "kept", note: "valor coerente com o anterior" });
  });

  it("Levey-Jennings: só chaves e datas na URL", async () => {
    fetchMock.mockImplementationOnce(() => ok({ target: null, points: [] }));
    await getQcChart({ lot_id: "ql1", level: "patológico", analyte_key: "glicose", instrument_id: "in1", from: "2026-09-18", to: "2026-10-18" });
    const p = urlOf().searchParams;
    expect(urlOf().pathname).toBe("/lab/qc/chart");
    expect([ ...p.keys() ].sort()).toEqual([ "analyte_key", "from", "instrument_id", "level", "lot_id", "to" ]);
    expect(p.get("level")).toBe("patológico");
  });

  it("contador de bloqueios: número; corpo estranho vira 0", async () => {
    fetchMock.mockImplementationOnce(() => ok({ count: 3 }));
    expect(await countActiveQcBlocks()).toBe(3);
    expect(urlOf().pathname).toBe("/lab/qc/blocks/active/count");
    fetchMock.mockImplementationOnce(() => ok({ count: "x" }));
    expect(await countActiveQcBlocks()).toBe(0);
  });

  it("estoque: lotes com saldo; movimento com quantidade no corpo", async () => {
    fetchMock.mockImplementationOnce(() => ok({ lots: [] }));
    await listSupplyLots({ supplyId: "su1" });
    expect(urlOf().pathname).toBe("/lab/inventory/lots");
    expect(urlOf().searchParams.get("supply_id")).toBe("su1");
    expect(urlOf().searchParams.get("with_balance")).toBe("true");
    await recordSupplyMovement({ lot_id: "sl1", kind: "loss", quantity: 1, reason: "frasco quebrado" });
    expect(urlOf(1).pathname).toBe("/lab/inventory/movements");
    expect(urlOf(1).search).toBe("");
    expect(bodyOf(1)).toEqual({ lot_id: "sl1", kind: "loss", quantity: 1, reason: "frasco quebrado" });
  });

  it("rastreio e consumo: JSON e CSV com fetch (não navegação)", async () => {
    fetchMock.mockImplementationOnce(() => ok({ results: [], qc_values: [] }));
    await traceSupplyLot("sl1");
    expect(urlOf().pathname).toBe("/lab/inventory/lots/sl1/trace");
    fetchMock.mockImplementationOnce(() => ok("a,b\n", { headers: { "Content-Type": "text/csv" } }));
    await fetchSupplyLotTraceCsv("sl1");
    expect(urlOf(1).pathname).toBe("/lab/inventory/lots/sl1/trace.csv");
    expect(headersOf(1).Accept).toBe("text/csv");
    fetchMock.mockImplementationOnce(() => ok("a,b\n", { headers: { "Content-Type": "text/csv" } }));
    await fetchConsumptionCsv({ providerId: "pv9", month: "2026-10" });
    expect(urlOf(2).pathname).toBe("/lab/inventory/consumption.csv");
    expect(urlOf(2).searchParams.get("month")).toBe("2026-10");
  });

  it("relatório anual do CEQ: só tipo e conteúdo (o nome do arquivo não vai)", async () => {
    fetchMock.mockImplementationOnce(() => ok({ upload_id: "up1" }));
    const file = new File([ "%PDF-1.4" ], "Relatorio Joana.pdf", { type: "application/pdf" });
    expect(await uploadEqaReport(file)).toBe("up1");
    expect(urlOf().pathname).toBe("/lab/eqa/uploads");
    expect(Object.keys(bodyOf()).sort()).toEqual([ "content_type", "data" ]);
    expect(String(initOf().body)).not.toContain("Joana");
  });
});
```

E no fim do `describe` de `src/lib/features.test.ts`:

```ts
  it("21c: CIQ/CEQ e estoque do laboratório", () => {
    expect(featureLabel("lab_quality_control")).toBe("Controle de qualidade do laboratório (CIQ e CEQ)");
    expect(featureLabel("lab_inventory")).toBe("Estoque do laboratório");
    expect(hasFeature({ features: [ "lab_inventory" ] }, "lab_inventory")).toBe(true);
    expect(hasFeature({ features: [ "lab_inventory" ] }, "lab_quality_control")).toBe(false);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/api.qc.test.ts src/lib/features.test.ts`
Expected: FAIL — `listLabInstruments` (e as demais) não exportadas; `featureLabel("lab_quality_control")` devolve a chave.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, nos tipos do 21b:

```ts
// Divergência D1 (21b) + C1 (21c): `technical_lead_of` = laboratórios em que a sessão é RT ou substituto ativo.
export interface LabContext { labs: LabProviderRef[]; releaser_of: string[]; council: string | null; technical_lead_of?: string[] }
```

```ts
// Divergência D3 (21b) + C5 (21c): `id` e `analytes` para CIQ, CEQ e lote em uso.
export interface LabExamRef { sigtap_code: string; name: string; material: string; delivery_days: number; id?: string; analytes?: { key: string; name: string }[] }
```

Em `LabResultRow`, depois de `effective: boolean; notified?: boolean;`:

```ts
  // 21c (contrato §7): lote de reagente em uso na gravação; null = "sem rastreio"; ausente = api antigo ou quem não é do laboratório.
  reagent_lot?: ReagentLotRef | null;
```

Em `LabExamVersion`, depois de `instrument_test_code: string | null;`:

```ts
  // 21c (contrato §2): aparelho em que o exame roda (opcional).
  lab_instrument_id?: string | null;
```

E no fim do arquivo:

```ts
// ─── Qualidade e insumos do laboratório (módulo 21c; contrato 2026-10-18) ───
// CIQ, CEQ e estoque sobre o 21b. Na URL só ids, situação, nível, chave do
// analito, datas, mês e ano; valor, média, dp, justificativa, motivo, ação,
// código de barras do nível e quantidade vão no corpo. Decimais trafegam como
// string com ponto (Divergência C4), como o valor de resultado do 21b.
export type QcRunStatus = "open" | "approved" | "warning" | "rejected";
export type QcTargetSource = "package_insert" | "laboratory";
export type QcRule = "1_2s" | "1_3s" | "2_2s" | "R_4s" | "4_1s" | "10x";
export const QC_RULES: QcRule[] = [ "1_2s", "1_3s", "2_2s", "R_4s", "4_1s", "10x" ];
export type QcOutcome = "ok" | "warning" | "reject";
export type QcReviewDecision = "pending" | "kept" | "rectify";
export type EqaAdequacy = "adequate" | "inadequate";
export type SupplyKind = "reagent" | "calibrator" | "control" | "consumable";
export type SupplyOrigin = "purchase" | "donation" | "loan";
export type SupplyMovementKind = "receipt" | "opening" | "loss" | "expiry" | "adjustment";
export type QcLotStatus = "in_use" | "closed";
export interface ReagentLotRef { id: string; lot_number: string }

export interface LabInstrument {
  id: string; provider_id: string; name: string; manufacturer: string; model: string; serial_number: string; sector: string;
  hl7_equipment_id: string | null; active: boolean;
}
export type LabInstrumentInput = Omit<LabInstrument, "id">;

export interface QcAnalyteRef { lab_exam_id: string; analyte_key: string }
export interface QcMaterial { id: string; provider_id: string; name: string; manufacturer: string; levels: string[]; analytes: QcAnalyteRef[]; active: boolean }
export type QcMaterialInput = Omit<QcMaterial, "id">;
// Divergência C3: forma do lote de controle.
export interface QcLot {
  id: string; material_id: string; material_name: string; lot_number: string; expires_on: string; level_barcodes: Record<string, string>;
  status: QcLotStatus | string; opened_on: string | null; closed_on: string | null;
}
export interface QcLotInput { material_id: string; lot_number: string; expires_on: string; level_barcodes: Record<string, string> }
export interface QcTargetKey { lot_id: string; level: string; analyte_key: string; instrument_id: string }
export interface QcTarget extends QcTargetKey {
  id: string; version: number; source: QcTargetSource | string; mean: string; sd: string;
  accepted_by: string | null; accepted_at: string; based_on_runs: number | null;
}
// Divergência C3: vigentes e histórico do lote.
export interface QcTargetSet { current: QcTarget[]; history: QcTarget[] }
// Divergência C6: `reason` quando já existe referência vigente para a chave.
export interface QcTargetInput extends QcTargetKey { mean: string; sd: string; reason?: string }
export interface QcSuggestion extends QcTargetKey { mean: string; sd: string; runs: number }
export interface QcValue {
  id: string; lot_id?: string; level: string; analyte_key: string; value: string; z_score: number | null; rules_fired: string[];
  outcome: QcOutcome | string; source: "typed" | "hl7" | string; reagent_lot_id: string | null;
  // Divergência C10: o número do lote, como no resultado do paciente.
  reagent_lot?: ReagentLotRef | null; entered_at: string; entered_by: string | null;
}
export interface QcRun { id: string; instrument_id: string; status: QcRunStatus | string; started_at: string; evaluated_at: string | null; values: QcValue[] }
export interface QcBlock {
  id: string; instrument_id: string; analyte_key: string; opened_at: string; opened_by_run_id: string;
  corrective_action: string | null; closed_at: string | null;
}
export interface QcValueInput { run_id: string | null; instrument_id: string; lot_id: string; level: string; analyte_key: string; value: string }
export interface QcValueResult { value: QcValue; run: QcRun; block: QcBlock | null }
export interface QcChartPoint { at: string; value: string; z_score: number; outcome: QcOutcome | string; rules_fired: string[]; target_version: number }
export interface QcChart { target: { mean: string; sd: string; version: number; source: QcTargetSource | string } | null; points: QcChartPoint[] }
export interface QcChartQuery extends QcTargetKey { from: string; to: string }
// Divergência C16: forma da leitura das regras (ausente = padrão, tudo ligado).
export interface QcRuleSet {
  instrument_id: string; analyte_key: string; rules: Record<QcRule, boolean>; is_default: boolean;
  justification: string | null; changed_by: string | null; changed_at: string | null;
}
export interface QcRuleInput { instrument_id: string; analyte_key: string; rules: Record<QcRule, boolean>; justification: string }
// Divergência C3: item da lista de revisão (sem paciente).
export interface QcReview {
  id: string; block_id: string; item_id: string; requisition_id: string; requisition_number: string; sigtap_code: string; exam_label: string;
  analyte_key: string; instrument_id: string; released_at: string | null; decision: QcReviewDecision | string; note: string | null;
}
// Divergência C7: exceções concedidas.
export interface QcException {
  id: string; block_id: string; item_id: string; requisition_number: string; analyte_key: string; instrument_id: string;
  justification: string; granted_by: string | null; granted_at: string;
}
// Contrato §3 + Divergência C9 (`instrument_name`).
export interface QcBlockedItem { item_id: string; block_id: string; analyte_key: string; instrument_id: string; instrument_name?: string }

export interface EqaItem {
  analyte_key: string; lab_exam_id: string; submitted: string; target: string; acceptable_low: string | null; acceptable_high: string | null;
  adequacy: EqaAdequacy | string; action: string | null;
}
export interface EqaRound {
  id: string; provider_id: string; program: string; round_code: string; round_date: string; items: EqaItem[];
  annual_report_upload_id: string | null; recorded_by?: string | null;
}
export type EqaRoundInput = Omit<EqaRound, "id" | "recorded_by">;
// Divergência C3: resumo anual por analito.
export interface EqaSummaryRow { analyte_key: string; lab_exam_id: string; rounds: number; adequate: number; inadequate: number }

export interface LabSupply {
  id: string; provider_id: string; name: string; kind: SupplyKind | string; unit: string; tests_per_package: number | null;
  minimum_stock: number; expiry_alert_days: number; lab_exam_ids: string[]; active: boolean;
  // Divergência C3: saldo somado dos lotes.
  balance: number;
}
export type LabSupplyInput = Omit<LabSupply, "id" | "balance">;
export interface LabSupplyLot {
  id: string; supply_id: string; supply_name: string; lot_number: string; expires_on: string; supplier: string;
  origin: SupplyOrigin | string; balance: number;
}
export interface LabSupplyLotInput {
  supply_id: string; lot_number: string; expires_on: string; supplier: string; origin: SupplyOrigin; quantity: number; document?: string;
}
export interface LabSupplyMovement {
  id: string; lot_id: string; kind: SupplyMovementKind | string; quantity: number; reason: string | null; document: string | null;
  recorded_by: string | null; recorded_at: string;
}
export interface LabSupplyMovementInput { lot_id: string; kind: Exclude<SupplyMovementKind, "receipt">; quantity: number; reason: string }
export interface LabSupplyInUse {
  id: string; lot_id: string; lot_number: string; supply_name: string; instrument_id: string; lab_exam_id: string; exam_name: string; started_at: string;
}
export interface LabSupplyInUseInput { lot_id: string; instrument_id: string; lab_exam_id: string }
export interface LabExpiringLot { lot_id: string; supply_name: string; lot_number: string; expires_on: string; balance: number }
export interface LabLowSupply { supply_id: string; name: string; unit: string; balance: number; minimum_stock: number }
export interface LabInventoryAlerts { expiring: LabExpiringLot[]; below_minimum: LabLowSupply[]; untraced_results: number }
export interface LabLotTrace {
  results: { requisition_number: string; sigtap_code: string; entered_at: string }[];
  qc_values: { run_id: string; analyte_key: string; entered_at: string }[];
}
export interface LabConsumptionRow {
  supply_id: string; name: string; instrument_id: string | null; tests_performed: number; tests_opened: number; deviation_pct: number | null;
}
export interface LabConsumptionQuery { providerId: string; month: string }

const labPut = (body: unknown): RequestInit => ({ method: "PUT", body: JSON.stringify(body) });
const labPatch = (body: unknown): RequestInit => ({ method: "PATCH", body: JSON.stringify(body) });
const byProvider = (providerId: string, extra: Record<string, string> = {}) => new URLSearchParams({ provider_id: providerId, ...extra });

// Aparelhos (contrato §2; Divergências C2 e C13).
export async function listLabInstruments(providerId: string): Promise<LabInstrument[]> {
  return (await jsonFetch<{ instruments: LabInstrument[] }>(labQuery(labPath("instruments"), byProvider(providerId)))).instruments;
}
export async function listAdminInstruments(providerId: string): Promise<LabInstrument[]> {
  return (await jsonFetch<{ instruments: LabInstrument[] }>(labQuery(labPath("admin", "instruments"), byProvider(providerId)))).instruments;
}
export function createInstrument(input: LabInstrumentInput): Promise<LabInstrument> {
  return jsonFetch(labPath("admin", "instruments"), labPost(input));
}
export function updateInstrument(id: string, input: LabInstrumentInput): Promise<LabInstrument> {
  return jsonFetch(labPath("admin", "instruments", id), labPatch(input));
}

// CIQ (contrato §3).
export async function listQcMaterials(providerId: string): Promise<QcMaterial[]> {
  return (await jsonFetch<{ materials: QcMaterial[] }>(labQuery(labPath("qc", "materials"), byProvider(providerId)))).materials;
}
export function createQcMaterial(input: QcMaterialInput): Promise<QcMaterial> {
  return jsonFetch(labPath("qc", "materials"), labPost(input));
}
export function updateQcMaterial(id: string, input: QcMaterialInput): Promise<QcMaterial> {
  return jsonFetch(labPath("qc", "materials", id), labPatch(input));
}
export async function listQcLots(q: { providerId: string; status?: QcLotStatus }): Promise<QcLot[]> {
  const p = byProvider(q.providerId);
  if (q.status) p.set("status", q.status);
  return (await jsonFetch<{ lots: QcLot[] }>(labQuery(labPath("qc", "lots"), p))).lots;
}
export function createQcLot(input: QcLotInput): Promise<QcLot> {
  return jsonFetch(labPath("qc", "lots"), labPost(input));
}
export function closeQcLot(id: string): Promise<QcLot> {
  return jsonFetch(labPath("qc", "lots", id, "close"), labPost({}));
}
export async function getQcTargets(lotId: string): Promise<QcTargetSet> {
  const body = await jsonFetch<Partial<QcTargetSet>>(labQuery(labPath("qc", "targets"), new URLSearchParams({ lot_id: lotId })));
  return { current: body.current ?? [], history: body.history ?? [] };
}
export function createQcTarget(input: QcTargetInput): Promise<QcTarget> {
  return jsonFetch(labPath("qc", "targets"), labPost(input));
}
export async function listQcSuggestions(providerId: string): Promise<QcSuggestion[]> {
  return (await jsonFetch<{ suggestions: QcSuggestion[] }>(labQuery(labPath("qc", "targets", "suggestions"), byProvider(providerId)))).suggestions;
}
export function acceptQcSuggestion(input: QcTargetKey & { reason: string }): Promise<QcTarget> {
  return jsonFetch(labPath("qc", "targets", "suggestions", "accept"), labPost(input));
}
export async function listQcRuns(instrumentId: string, date: string): Promise<QcRun[]> {
  const p = new URLSearchParams({ instrument_id: instrumentId, date });
  return (await jsonFetch<{ runs: QcRun[] }>(labQuery(labPath("qc", "runs"), p))).runs;
}
export function openQcRun(instrumentId: string): Promise<QcRun> {
  return jsonFetch(labPath("qc", "runs"), labPost({ instrument_id: instrumentId }));
}
export function recordQcValue(input: QcValueInput): Promise<QcValueResult> {
  return jsonFetch(labPath("qc", "values"), labPost(input));
}
// Até 200 pontos; sem trilha (contrato §3).
export function getQcChart(q: QcChartQuery): Promise<QcChart> {
  const p = new URLSearchParams({ lot_id: q.lot_id, level: q.level, analyte_key: q.analyte_key, instrument_id: q.instrument_id, from: q.from, to: q.to });
  return jsonFetch(labQuery(labPath("qc", "chart"), p));
}
export function getQcRules(instrumentId: string, analyteKey: string): Promise<QcRuleSet> {
  return jsonFetch(labQuery(labPath("qc", "rules"), new URLSearchParams({ instrument_id: instrumentId, analyte_key: analyteKey })));
}
export function saveQcRules(input: QcRuleInput): Promise<QcRuleSet> {
  return jsonFetch(labPath("qc", "rules"), labPut(input));
}
export async function listQcBlocks(status: "active" | "closed", providerId: string): Promise<QcBlock[]> {
  return (await jsonFetch<{ blocks: QcBlock[] }>(labQuery(labPath("qc", "blocks"), byProvider(providerId, { status })))).blocks;
}
// Só a contagem, sem dado de corrida e sem trilha (Divergência C11). Única rota do 21c relida sozinha.
export async function countActiveQcBlocks(): Promise<number> {
  const body = await jsonFetch<{ count?: unknown }>(labPath("qc", "blocks", "active", "count"));
  const n = Number(body.count ?? 0);
  return Number.isFinite(n) ? n : 0;
}
export function recordCorrectiveAction(blockId: string, text: string): Promise<QcBlock> {
  return jsonFetch(labPath("qc", "blocks", blockId, "corrective_action"), labPost({ text }));
}
export function grantQcException(input: { block_id: string; item_id: string; justification: string }): Promise<QcException> {
  return jsonFetch(labPath("qc", "exceptions"), labPost(input));
}
export async function listQcExceptions(providerId: string, month: string): Promise<QcException[]> {
  return (await jsonFetch<{ exceptions: QcException[] }>(labQuery(labPath("qc", "exceptions"), byProvider(providerId, { month })))).exceptions;
}
export async function listQcReviews(providerId: string): Promise<QcReview[]> {
  return (await jsonFetch<{ reviews: QcReview[] }>(labQuery(labPath("qc", "reviews"), byProvider(providerId, { status: "pending" })))).reviews;
}
export function decideQcReview(id: string, input: { decision: "kept" | "rectify"; note?: string }): Promise<QcReview> {
  return jsonFetch(labPath("qc", "reviews", id), labPost(input));
}

// CEQ (contrato §4; Divergências C3 e C12).
export async function listEqaRounds(providerId: string, year: string): Promise<EqaRound[]> {
  return (await jsonFetch<{ rounds: EqaRound[] }>(labQuery(labPath("eqa", "rounds"), byProvider(providerId, { year })))).rounds;
}
export function createEqaRound(input: EqaRoundInput): Promise<EqaRound> {
  return jsonFetch(labPath("eqa", "rounds"), labPost(input));
}
// Só o tipo e o conteúdo: o nome do arquivo pode trazer nome de pessoa.
export async function uploadEqaReport(file: File): Promise<string> {
  const data = await readBase64(file);
  return (await jsonFetch<{ upload_id: string }>(labPath("eqa", "uploads"), labPost({ content_type: file.type, data }))).upload_id;
}
export function fetchEqaReport(roundId: string): Promise<SignatureFile> {
  return binaryFetch(labPath("eqa", "rounds", roundId, "annual_report.pdf"), "application/pdf");
}
export async function getEqaSummary(providerId: string, year: string): Promise<EqaSummaryRow[]> {
  return (await jsonFetch<{ rows: EqaSummaryRow[] }>(labQuery(labPath("eqa", "summary"), byProvider(providerId, { year })))).rows;
}

// Estoque (contrato §5; Divergências C2, C3, C8 e C15).
export async function listSupplies(providerId: string): Promise<LabSupply[]> {
  return (await jsonFetch<{ supplies: LabSupply[] }>(labQuery(labPath("inventory", "supplies"), byProvider(providerId)))).supplies;
}
export function createSupply(input: LabSupplyInput): Promise<LabSupply> {
  return jsonFetch(labPath("inventory", "supplies"), labPost(input));
}
export function updateSupply(id: string, input: LabSupplyInput): Promise<LabSupply> {
  return jsonFetch(labPath("inventory", "supplies", id), labPatch(input));
}
export async function listSupplyLots(q: { supplyId?: string; providerId?: string }): Promise<LabSupplyLot[]> {
  const p = new URLSearchParams();
  if (q.supplyId) p.set("supply_id", q.supplyId);
  if (q.providerId) p.set("provider_id", q.providerId);
  p.set("with_balance", "true");
  return (await jsonFetch<{ lots: LabSupplyLot[] }>(labQuery(labPath("inventory", "lots"), p))).lots;
}
export function receiveSupplyLot(input: LabSupplyLotInput): Promise<LabSupplyLot> {
  return jsonFetch(labPath("inventory", "lots"), labPost(input));
}
export async function listSupplyMovements(lotId: string): Promise<LabSupplyMovement[]> {
  return (await jsonFetch<{ movements: LabSupplyMovement[] }>(labQuery(labPath("inventory", "movements"), new URLSearchParams({ lot_id: lotId })))).movements;
}
export function recordSupplyMovement(input: LabSupplyMovementInput): Promise<LabSupplyMovement> {
  return jsonFetch(labPath("inventory", "movements"), labPost(input));
}
export async function listInUse(instrumentId: string): Promise<LabSupplyInUse[]> {
  return (await jsonFetch<{ in_use: LabSupplyInUse[] }>(labQuery(labPath("inventory", "in_use"), new URLSearchParams({ instrument_id: instrumentId })))).in_use;
}
export function setInUse(input: LabSupplyInUseInput): Promise<LabSupplyInUse> {
  return jsonFetch(labPath("inventory", "in_use"), labPost(input));
}
// Sem trilha (contrato §5).
export async function getInventoryAlerts(providerId: string): Promise<LabInventoryAlerts> {
  const body = await jsonFetch<Partial<LabInventoryAlerts>>(labQuery(labPath("inventory", "alerts"), byProvider(providerId)));
  return { expiring: body.expiring ?? [], below_minimum: body.below_minimum ?? [], untraced_results: Number(body.untraced_results ?? 0) || 0 };
}
// Grava trilha de leitura: só no clique.
export async function traceSupplyLot(lotId: string): Promise<LabLotTrace> {
  const body = await jsonFetch<Partial<LabLotTrace>>(labPath("inventory", "lots", lotId, "trace"));
  return { results: body.results ?? [], qc_values: body.qc_values ?? [] };
}
export function fetchSupplyLotTraceCsv(lotId: string): Promise<SignatureFile> {
  return binaryFetch(labPath("inventory", "lots", lotId, "trace.csv"), "text/csv");
}
export async function getConsumption(q: LabConsumptionQuery): Promise<LabConsumptionRow[]> {
  return (await jsonFetch<{ rows: LabConsumptionRow[] }>(labQuery(labPath("inventory", "consumption"), byProvider(q.providerId, { month: q.month })))).rows;
}
export function fetchConsumptionCsv(q: LabConsumptionQuery): Promise<SignatureFile> {
  return binaryFetch(labQuery(labPath("inventory", "consumption.csv"), byProvider(q.providerId, { month: q.month })), "text/csv");
}
```

Em `src/lib/features.ts`:

```ts
  | "municipal_lab" | "lab_hl7_inbound" | "fake_middleware" | "lab_quality_control" | "lab_inventory";
```

```ts
  fake_middleware: "Middleware de aparelhos simulado (fora de produção)",
  lab_quality_control: "Controle de qualidade do laboratório (CIQ e CEQ)",
  lab_inventory: "Estoque do laboratório"
```

E `src/test/qcFixtures.ts`:

```ts
// src/test/qcFixtures.ts
// Dados comuns aos testes do módulo 21c. Cada arquivo de teste faz o próprio
// vi.mock("…/lib/api"); aqui só tipos, a sessão e o ResizeObserver do gráfico.
import type {
  EqaRound, LabContext, LabExamRef, LabInstrument, LabSupply, LabSupplyInUse, LabSupplyLot, LabSupplyMovement, QcBlock, QcChart,
  QcException, QcLot, QcMaterial, QcReview, QcRuleSet, QcRun, QcSuggestion, QcTarget, QcValue, SessionUser
} from "../lib/api";
import { LAB_PROVIDER, labContext, labUser } from "./labFixtures";

export const NOW21C = "2026-10-18T10:00:00-03:00";
export const TODAY21C = "2026-10-18";
export const QC_FEATURES = [ "exam_tracking", "municipal_lab", "lab_quality_control", "lab_inventory" ];

export function qcUser(roles: string[] = [ "lab_staff" ], over: Partial<SessionUser> = {}): SessionUser {
  return labUser(roles, { features: QC_FEATURES, ...over });
}

export function qcContext(over: Partial<LabContext> = {}): LabContext {
  return labContext({ releaser_of: [], technical_lead_of: [], ...over });
}

export const GLU_EXAM: LabExamRef = {
  id: "le1", sigtap_code: "0202010473", name: "Glicose", material: "soro", delivery_days: 2, analytes: [ { key: "glicose", name: "Glicose" } ]
};
export const CHOL_EXAM: LabExamRef = {
  id: "le2", sigtap_code: "0202010295", name: "Colesterol total e frações", material: "soro", delivery_days: 2,
  analytes: [ { key: "ct", name: "Colesterol total" }, { key: "hdl", name: "HDL" } ]
};

export function instrument(over: Partial<LabInstrument> = {}): LabInstrument {
  return {
    id: "in1", provider_id: LAB_PROVIDER.id, name: "BS-200", manufacturer: "Mindray", model: "BS-200", serial_number: "SN-0042",
    sector: "biochemistry", hl7_equipment_id: "BS200-01", active: true, ...over
  };
}

export function qcMaterial(over: Partial<QcMaterial> = {}): QcMaterial {
  return {
    id: "qm1", provider_id: LAB_PROVIDER.id, name: "Controle bioquímico", manufacturer: "Controllab", levels: [ "normal", "patológico" ],
    analytes: [ { lab_exam_id: "le1", analyte_key: "glicose" } ], active: true, ...over
  };
}

export function qcLot(over: Partial<QcLot> = {}): QcLot {
  return {
    id: "ql1", material_id: "qm1", material_name: "Controle bioquímico", lot_number: "L2026A", expires_on: "2027-03-31",
    level_barcodes: { normal: "QC-L2026A-N", "patológico": "QC-L2026A-P" }, status: "in_use", opened_on: "2026-10-01", closed_on: null, ...over
  };
}

export function qcTarget(over: Partial<QcTarget> = {}): QcTarget {
  return {
    id: "qt1", lot_id: "ql1", level: "normal", analyte_key: "glicose", instrument_id: "in1", version: 1, source: "package_insert",
    mean: "98.0", sd: "2.5", accepted_by: "Bia Farmacêutica", accepted_at: "2026-10-01T09:00:00-03:00", based_on_runs: null, ...over
  };
}

export function qcSuggestion(over: Partial<QcSuggestion> = {}): QcSuggestion {
  return { lot_id: "ql1", level: "normal", analyte_key: "glicose", instrument_id: "in1", mean: "97.6", sd: "2.1", runs: 20, ...over };
}

export function qcValue(over: Partial<QcValue> = {}): QcValue {
  return {
    id: "qv1", lot_id: "ql1", level: "normal", analyte_key: "glicose", value: "99.0", z_score: 0.4, rules_fired: [], outcome: "ok",
    source: "typed", reagent_lot_id: "sl1", reagent_lot: { id: "sl1", lot_number: "R-778" }, entered_at: "2026-10-18T07:10:00-03:00",
    entered_by: "Téc. Carla", ...over
  };
}

export function qcRun(over: Partial<QcRun> = {}): QcRun {
  return {
    id: "qr1", instrument_id: "in1", status: "approved", started_at: "2026-10-18T07:00:00-03:00", evaluated_at: "2026-10-18T07:12:00-03:00",
    values: [ qcValue() ], ...over
  };
}

export function qcBlock(over: Partial<QcBlock> = {}): QcBlock {
  return {
    id: "qb1", instrument_id: "in1", analyte_key: "glicose", opened_at: "2026-10-18T08:00:00-03:00", opened_by_run_id: "qr2",
    corrective_action: null, closed_at: null, ...over
  };
}

export function qcReview(over: Partial<QcReview> = {}): QcReview {
  return {
    id: "rv1", block_id: "qb1", item_id: "it2", requisition_id: "rq1", requisition_number: "00000123", sigtap_code: "0202010473",
    exam_label: "Glicose", analyte_key: "glicose", instrument_id: "in1", released_at: "2026-10-18T07:40:00-03:00", decision: "pending",
    note: null, ...over
  };
}

export function qcException(over: Partial<QcException> = {}): QcException {
  return {
    id: "qe1", block_id: "qb1", item_id: "it2", requisition_number: "00000123", analyte_key: "glicose", instrument_id: "in1",
    justification: "paciente internado, médico assistente ciente", granted_by: "Bia Farmacêutica", granted_at: "2026-10-18T09:00:00-03:00", ...over
  };
}

export function qcRules(over: Partial<QcRuleSet> = {}): QcRuleSet {
  return {
    instrument_id: "in1", analyte_key: "glicose",
    rules: { "1_2s": true, "1_3s": true, "2_2s": true, "R_4s": true, "4_1s": true, "10x": true },
    is_default: true, justification: null, changed_by: null, changed_at: null, ...over
  };
}

// Seis pontos: dois avisos/reprovações e a troca da versão 1 para a 2 no quinto.
export function qcChart(over: Partial<QcChart> = {}): QcChart {
  return {
    target: { mean: "98.0", sd: "2.5", version: 2, source: "laboratory" },
    points: [
      { at: "2026-10-13T07:10:00-03:00", value: "99.0", z_score: 0.4, outcome: "ok", rules_fired: [], target_version: 1 },
      { at: "2026-10-14T07:10:00-03:00", value: "103.5", z_score: 2.2, outcome: "warning", rules_fired: [ "1_2s" ], target_version: 1 },
      { at: "2026-10-15T07:10:00-03:00", value: "104.0", z_score: 2.4, outcome: "reject", rules_fired: [ "2_2s" ], target_version: 1 },
      { at: "2026-10-16T07:10:00-03:00", value: "97.0", z_score: -0.4, outcome: "ok", rules_fired: [], target_version: 1 },
      { at: "2026-10-17T07:10:00-03:00", value: "98.5", z_score: 0.2, outcome: "ok", rules_fired: [], target_version: 2 },
      { at: "2026-10-18T07:10:00-03:00", value: "96.0", z_score: -0.8, outcome: "ok", rules_fired: [], target_version: 2 }
    ],
    ...over
  };
}

export function eqaRound(over: Partial<EqaRound> = {}): EqaRound {
  return {
    id: "er1", provider_id: LAB_PROVIDER.id, program: "PNCQ", round_code: "2026-03", round_date: "2026-09-20",
    items: [ { analyte_key: "glicose", lab_exam_id: "le1", submitted: "101.0", target: "100.0", acceptable_low: "94.0", acceptable_high: "106.0", adequacy: "adequate", action: null } ],
    annual_report_upload_id: null, recorded_by: "Bia Farmacêutica", ...over
  };
}

export function supply(over: Partial<LabSupply> = {}): LabSupply {
  return {
    id: "su1", provider_id: LAB_PROVIDER.id, name: "Glicose GOD-PAP 2×100 mL", kind: "reagent", unit: "kit", tests_per_package: 400,
    minimum_stock: 2, expiry_alert_days: 30, lab_exam_ids: [ "le1" ], active: true, balance: 5, ...over
  };
}

export function supplyLot(over: Partial<LabSupplyLot> = {}): LabSupplyLot {
  return {
    id: "sl1", supply_id: "su1", supply_name: "Glicose GOD-PAP 2×100 mL", lot_number: "R-778", expires_on: "2027-01-31",
    supplier: "Distribuidora Sul", origin: "purchase", balance: 5, ...over
  };
}

export function supplyMovement(over: Partial<LabSupplyMovement> = {}): LabSupplyMovement {
  return {
    id: "mv1", lot_id: "sl1", kind: "receipt", quantity: 6, reason: null, document: "NF 1234", recorded_by: "Téc. Carla",
    recorded_at: "2026-10-10T09:00:00-03:00", ...over
  };
}

export function inUse(over: Partial<LabSupplyInUse> = {}): LabSupplyInUse {
  return {
    id: "iu1", lot_id: "sl1", lot_number: "R-778", supply_name: "Glicose GOD-PAP 2×100 mL", instrument_id: "in1", lab_exam_id: "le1",
    exam_name: "Glicose", started_at: "2026-10-12T07:00:00-03:00", ...over
  };
}

// Recharts (ResponsiveContainer) mede o pai por ResizeObserver; o jsdom não tem
// layout. Este informa 600×260 ao observar (prova técnica do Ambiente).
export class SizedResizeObserver {
  constructor(private callback: ResizeObserverCallback) {}
  observe(target: Element) {
    const rect = { width: 600, height: 260, x: 0, y: 0, top: 0, left: 0, right: 600, bottom: 260, toJSON() {} };
    this.callback([ { target, contentRect: rect } as unknown as ResizeObserverEntry ], this as unknown as ResizeObserver);
  }
  unobserve() {}
  disconnect() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/api.qc.test.ts src/lib/features.test.ts src/lib/api.lab.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` limpo (os campos novos dos tipos do 21b são opcionais: nada do 21b quebra).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.qc.test.ts src/lib/features.ts src/lib/features.test.ts src/test/qcFixtures.ts
$G commit -m "feat: add the lab quality control, external quality and inventory client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras do CIQ e do CEQ (`src/lib/qc.ts`)

**Files:**
- Create: `src/lib/qc.ts`
- Test: `src/lib/qc.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `errorCode`, os tipos da Task 1; `featureDisabledKey` (`features.ts`); `DOT_IN_NUMBER`, `SECTOR_LABEL`, `hasDot`, `labError`, `normalizeNumber` (`lab.ts`); `Tone` (`theme/tokens`).
- Produces: chaves `QC_INSTRUMENTS_KEY`, `QC_MATERIALS_KEY`, `QC_LOTS_KEY`, `QC_TARGETS_KEY`, `QC_SUGGESTIONS_KEY`, `QC_RUNS_KEY`, `QC_CHART_KEY`, `QC_RULES_KEY`, `QC_BLOCKS_KEY`, `QC_BLOCK_COUNT_KEY`, `QC_REVIEWS_KEY`, `QC_EXCEPTIONS_KEY`, `QC_EXAMS_KEY`, `EQA_ROUNDS_KEY`, `EQA_SUMMARY_KEY`; frases `QC_DISABLED`, `INSTRUMENTS_DISABLED`, `QC_LEAD_ONLY`, `RT_ONLY`, `REASON_MIN`; rótulos `ruleLabel`, `rulesText`, `runStatusLabel`, `runStatusTone`, `outcomeLabel`, `outcomeTone`, `targetSourceLabel`, `reviewDecisionLabel`, `adequacyLabel`, `adequacyTone`, `lotStatusLabel`, `analyteName(exams, examId, key)`; números `fmtDecimal`, `decimalsOf`, `fmtZ`, `decimalProblem`, `reasonProblem`; formulários `InstrumentDraft`/`EMPTY_INSTRUMENT`/`instrumentDraft`/`instrumentProblem`/`instrumentBody`, `MaterialDraft`/`EMPTY_MATERIAL`/`materialDraft`/`parseLevels`/`analyteToken`/`tokenToAnalyte`/`materialProblem`/`materialBody`, `LotDraft`/`lotProblem`/`lotBody`, `TargetDraft`/`EMPTY_TARGET`/`targetProblem`/`targetBody`, `turnedOff`/`rulesProblem`, `qcValueProblem`, `EqaItemDraft`/`EMPTY_EQA_ITEM`/`EqaRoundDraft`/`EMPTY_EQA_ROUND`/`suggestAdequacy`/`eqaProblem`/`eqaBody`/`adequacyRate`; Levey-Jennings `LjRow`, `LjModel`, `SD_LINES`, `ljModel`, `ljBands`, `sdLabel`; recusas `qcError`, `refreshQc(client)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/qc.test.ts
import { describe, expect, it, vi } from "vitest";
import { ApiError } from "./api";
import {
  EMPTY_EQA_ITEM, EMPTY_EQA_ROUND, EMPTY_INSTRUMENT, EMPTY_TARGET, QC_DISABLED, adequacyRate, analyteName, decimalProblem, eqaBody, eqaProblem,
  fmtDecimal, fmtZ, instrumentBody, instrumentProblem, ljBands, ljModel, lotBody, lotProblem, materialBody, materialProblem, parseLevels,
  qcError, qcValueProblem, refreshQc, ruleLabel, rulesProblem, rulesText, sdLabel, suggestAdequacy, targetBody, targetProblem, turnedOff
} from "./qc";
import { DOT_IN_NUMBER } from "./lab";
import { CHOL_EXAM, GLU_EXAM, qcChart } from "../test/qcFixtures";

const ALL_ON = { "1_2s": true, "1_3s": true, "2_2s": true, "R_4s": true, "4_1s": true, "10x": true } as const;

describe("rótulos", () => {
  it("regras de Westgard com o aviso marcado; desconhecida sai crua", () => {
    expect(ruleLabel("1_2s")).toBe("1-2s (aviso)");
    expect(ruleLabel("R_4s")).toBe("R-4s");
    expect(rulesText([ "2_2s", "10x", "9_9s" ])).toBe("2-2s, 10x, 9_9s");
  });

  it("nome do analito vem do exame; sem exame, a chave", () => {
    expect(analyteName([ GLU_EXAM, CHOL_EXAM ], "le2", "hdl")).toBe("HDL (Colesterol total e frações)");
    expect(analyteName([ GLU_EXAM ], "le9", "tsh")).toBe("tsh");
  });
});

describe("números com vírgula", () => {
  it("decimal: vírgula aceita, ponto recusado, negativo só quando permitido, dp > 0", () => {
    expect(decimalProblem("98,5", "a média")).toBeNull();
    expect(decimalProblem("98.5", "a média")).toBe(DOT_IN_NUMBER);
    expect(decimalProblem("", "a média")).toBe("informe a média");
    expect(decimalProblem("abc", "a média")).toBe("a média: use dígitos e vírgula");
    expect(decimalProblem("-1,2", "a média")).toBe("a média não pode ser negativa");
    expect(decimalProblem("-1,2", "a média", { allowNegative: true })).toBeNull();
    expect(decimalProblem("0", "o desvio-padrão", { positive: true })).toBe("o desvio-padrão precisa ser maior que zero");
    expect(qcValueProblem("-0,3")).toBeNull();
    expect(qcValueProblem("2.5")).toBe(DOT_IN_NUMBER);
  });

  it("formatos de tela", () => {
    expect(fmtDecimal("98.50")).toBe("98,50");
    expect(fmtDecimal(2.5)).toBe("2,5");
    expect(fmtZ(2.34)).toBe("+2,3");
    expect(fmtZ(-0.5)).toBe("-0,5");
    expect(fmtZ(0)).toBe("0,0");
    expect(fmtZ(null)).toBe("—");
  });
});

describe("aparelho", () => {
  const ok = { ...EMPTY_INSTRUMENT, name: "BS-200", sector: "biochemistry", hl7EquipmentId: "BS200-01" };
  it("valida nome, setor, identificador HL7 e tamanhos (recusa, não corta)", () => {
    expect(instrumentProblem(ok)).toBeNull();
    expect(instrumentProblem({ ...ok, name: " " })).toBe("informe o nome do aparelho");
    expect(instrumentProblem({ ...ok, sector: "" })).toBe("escolha o setor");
    expect(instrumentProblem({ ...ok, hl7EquipmentId: "BS 200" })).toBe("identificador HL7 com até 40 letras, números, ponto, hífen ou _");
    expect(instrumentProblem({ ...ok, serialNumber: "x".repeat(121) })).toBe("número de série com até 120 caracteres");
  });
  it("corpo: HL7 vazio vira null", () => {
    expect(instrumentBody({ ...ok, hl7EquipmentId: " " }, "pv9")).toEqual({
      provider_id: "pv9", name: "BS-200", manufacturer: "", model: "", serial_number: "", sector: "biochemistry", hl7_equipment_id: null, active: true
    });
  });
});

describe("material e lote", () => {
  it("níveis: separados por ponto e vírgula, de 1 a 6, sem repetir", () => {
    expect(parseLevels(" normal ; patológico;; ")).toEqual([ "normal", "patológico" ]);
    const base = { name: "Controle bioquímico", manufacturer: "Controllab", levels: "normal; patológico", analytes: [ "le1:glicose" ], active: true };
    expect(materialProblem(base)).toBeNull();
    expect(materialProblem({ ...base, levels: "" })).toBe("informe de 1 a 6 níveis separados por ponto e vírgula");
    expect(materialProblem({ ...base, levels: "normal; Normal" })).toBe("níveis repetidos");
    expect(materialProblem({ ...base, analytes: [] })).toBe("escolha ao menos um analito");
    expect(materialBody(base, "pv9")).toEqual({
      provider_id: "pv9", name: "Controle bioquímico", manufacturer: "Controllab", levels: [ "normal", "patológico" ],
      analytes: [ { lab_exam_id: "le1", analyte_key: "glicose" } ], active: true
    });
  });

  it("lote: validade, um código por nível e códigos distintos", () => {
    const levels = [ "normal", "patológico" ];
    const d = { materialId: "qm1", lotNumber: "L2026A", expiresOn: "2027-03-31", barcodes: { normal: "QC-N1", "patológico": "QC-P1" } };
    expect(lotProblem(d, levels, "2026-10-18")).toBeNull();
    expect(lotProblem({ ...d, expiresOn: "2026-10-17" }, levels, "2026-10-18")).toBe("lote vencido — confira a validade");
    expect(lotProblem({ ...d, barcodes: { normal: "QC-N1" } }, levels, "2026-10-18")).toBe("informe o código de barras do nível patológico");
    expect(lotProblem({ ...d, barcodes: { normal: "QC-N1", "patológico": "qc-n1" } }, levels, "2026-10-18")).toBe("cada nível precisa de um código de barras diferente");
    expect(lotProblem({ ...d, barcodes: { normal: "N 1", "patológico": "QC-P1" } }, levels, "2026-10-18")).toBe("código do nível normal: 4 a 40 letras, números ou hífen");
    expect(lotBody({ ...d, barcodes: { ...d.barcodes, antigo: "QC-X" } }, levels)).toEqual({
      material_id: "qm1", lot_number: "L2026A", expires_on: "2027-03-31", level_barcodes: { normal: "QC-N1", "patológico": "QC-P1" }
    });
  });
});

describe("referência e regras", () => {
  const t = { ...EMPTY_TARGET, level: "normal", analyteKey: "glicose", instrumentId: "in1", mean: "98,0", sd: "2,5" };
  it("referência da bula: média e dp com vírgula; trocar a vigente pede motivo", () => {
    expect(targetProblem(t, false)).toBeNull();
    expect(targetProblem({ ...t, sd: "0" }, false)).toBe("o desvio-padrão precisa ser maior que zero");
    expect(targetProblem(t, true)).toBe("escreva o motivo da troca da referência (ao menos 10 caracteres)");
    expect(targetBody({ ...t, reason: "bula do lote novo" }, "ql1", true)).toEqual({
      lot_id: "ql1", level: "normal", analyte_key: "glicose", instrument_id: "in1", mean: "98.0", sd: "2.5", reason: "bula do lote novo"
    });
    expect(targetBody(t, "ql1", false)).not.toHaveProperty("reason");
  });

  it("regras: desligar pede justificativa; ligar não", () => {
    const off = { ...ALL_ON, "4_1s": false };
    expect(turnedOff(ALL_ON, off)).toEqual([ "4_1s" ]);
    expect(rulesProblem(ALL_ON, off, "curto")).toBe("escreva a justificativa para desligar 4-1s (ao menos 10 caracteres)");
    expect(rulesProblem(ALL_ON, off, "aparelho em validação, 4-1s gera falso alarme")).toBeNull();
    expect(rulesProblem(off, ALL_ON, "")).toBeNull();
  });
});

describe("Levey-Jennings", () => {
  it("troca de versão vira marca na série e aviso de versões misturadas", () => {
    const m = ljModel(qcChart());
    expect(m.rows.map((r) => r.n)).toEqual([ 1, 2, 3, 4, 5, 6 ]);
    expect(m.rows[2]).toMatchObject({ z: 2.4, outcome: "reject", rules: [ "2_2s" ], version: 1 });
    expect(m.versionChanges).toEqual([ { n: 5, version: 2 } ]);
    expect(m.mixedVersions).toBe(true);
    expect(m.flagged.map((r) => r.n)).toEqual([ 2, 3 ]);
    expect(m.domain).toEqual([ -4, 4 ]);
  });

  it("ponto além de 4 dp abre o eixo; uma versão só não avisa", () => {
    const one = qcChart({ points: [ { at: "2026-10-18T07:00:00-03:00", value: "112.0", z_score: -5.6, outcome: "reject", rules_fired: [ "1_3s" ], target_version: 2 } ] });
    const m = ljModel(one);
    expect(m.domain).toEqual([ -7, 7 ]);
    expect(m.mixedVersions).toBe(false);
    expect(m.versionChanges).toEqual([]);
  });

  it("faixas em unidades do analito, com as casas da referência", () => {
    expect(ljBands({ mean: "98.0", sd: "2.5" }).map((b) => `${b.label}=${b.value}`)).toEqual([
      "+3 dp=105,5", "+2 dp=103,0", "+1 dp=100,5", "média=98,0", "-1 dp=95,5", "-2 dp=93,0", "-3 dp=90,5"
    ]);
    expect(sdLabel(-2)).toBe("-2 dp");
  });
});

describe("CEQ", () => {
  const item = { ...EMPTY_EQA_ITEM, token: "le1:glicose", submitted: "101,0", target: "100,0", low: "94,0", high: "106,0", adequacy: "adequate" as const };
  const round = { ...EMPTY_EQA_ROUND, program: "PNCQ" as const, roundCode: "2026-03", roundDate: "2026-09-20", items: [ item ] };

  it("sugere a adequação pelos limites aceitáveis", () => {
    expect(suggestAdequacy(item)).toBe("adequate");
    expect(suggestAdequacy({ ...item, submitted: "107,0" })).toBe("inadequate");
    expect(suggestAdequacy({ ...item, low: "" })).toBe("");
  });

  it("inadequado sem ação, data no futuro e programa 'outro' sem nome são recusados", () => {
    expect(eqaProblem(round, "2026-10-18")).toBeNull();
    expect(eqaProblem({ ...round, items: [ { ...item, adequacy: "inadequate" } ] }, "2026-10-18")).toBe("item 1: escreva a ação para o resultado inadequado (ao menos 10 caracteres)");
    expect(eqaProblem({ ...round, roundDate: "2026-10-19" }, "2026-10-18")).toBe("a data da rodada não pode ser no futuro");
    expect(eqaProblem({ ...round, program: "outro", programOther: "" }, "2026-10-18")).toBe("informe o nome do programa");
    expect(eqaProblem({ ...round, items: [ { ...item, low: "106,0", high: "94,0" } ] }, "2026-10-18")).toBe("item 1: o limite inferior é maior que o superior");
  });

  it("corpo com decimais em ponto e programa escolhido", () => {
    expect(eqaBody({ ...round, uploadId: "up1" }, "pv9")).toEqual({
      provider_id: "pv9", program: "PNCQ", round_code: "2026-03", round_date: "2026-09-20", annual_report_upload_id: "up1",
      items: [ { lab_exam_id: "le1", analyte_key: "glicose", submitted: "101.0", target: "100.0", acceptable_low: "94.0", acceptable_high: "106.0", adequacy: "adequate", action: null } ]
    });
    expect(eqaBody({ ...round, program: "outro", programOther: "Programa estadual" }, "pv9").program).toBe("Programa estadual");
  });

  it("taxa de adequação com uma casa e vírgula", () => {
    expect(adequacyRate({ analyte_key: "glicose", lab_exam_id: "le1", rounds: 3, adequate: 2, inadequate: 1 })).toBe("66,7%");
    expect(adequacyRate({ analyte_key: "glicose", lab_exam_id: "le1", rounds: 0, adequate: 0, inadequate: 0 })).toBe("—");
  });
});

describe("recusas", () => {
  it("interruptor do CIQ, códigos do 21c e o resto pelo 21b", () => {
    expect(qcError(new ApiError(403, { error: "feature_disabled", feature: "lab_quality_control" }, "x"))).toBe(QC_DISABLED);
    expect(qcError(new ApiError(422, { error: "no_target" }, "x"))).toBe("este nível e analito ainda não têm referência neste aparelho — cadastre a da bula em Controles e referências");
    expect(qcError(new ApiError(422, { error: "barcode_taken" }, "x"))).toBe("este código de barras já está em outro lote de controle ou numa amostra");
    expect(qcError(new ApiError(409, { error: "invalid_transition" }, "x"))).toBe("a situação mudou — a tela foi atualizada");
  });

  it("refreshQc invalida todas as chaves do CIQ e do CEQ", async () => {
    const client = { invalidateQueries: vi.fn(() => Promise.resolve()) };
    await refreshQc(client as never);
    const keys = client.invalidateQueries.mock.calls.map(([ arg ]) => (arg as { queryKey: string[] }).queryKey[0]);
    expect(keys).toContain("qcRuns");
    expect(keys).toContain("qcBlockCount");
    expect(keys).toContain("eqaRounds");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/qc.test.ts`
Expected: FAIL — `Cannot find module './qc'`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/qc.ts
// Controle de qualidade do laboratório municipal (módulo 21c, ADR 0038;
// contrato 2026-10-18 §1–§4). Regras de tela, puras: rótulos, números com
// vírgula, validação dos formulários, o modelo do Levey-Jennings e as frases
// das recusas. A avaliação de Westgard, o bloqueio e quem é RT são do api.
import type { QueryClient } from "@tanstack/react-query";
import {
  errorCode,
  type EqaAdequacy, type EqaRoundInput, type EqaSummaryRow, type LabExamRef, type LabInstrument, type LabInstrumentInput, type QcAnalyteRef,
  type QcChart, type QcLotInput, type QcMaterial, type QcMaterialInput, type QcRule, type QcTargetInput
} from "./api";
import { featureDisabledKey } from "./features";
import { DOT_IN_NUMBER, SECTOR_LABEL, hasDot, labError, normalizeNumber } from "./lab";
import type { Tone } from "../theme/tokens";

export const QC_INSTRUMENTS_KEY = "qcInstruments";
export const QC_MATERIALS_KEY = "qcMaterials";
export const QC_LOTS_KEY = "qcLots";
export const QC_TARGETS_KEY = "qcTargets";
export const QC_SUGGESTIONS_KEY = "qcSuggestions";
export const QC_RUNS_KEY = "qcRuns";
export const QC_CHART_KEY = "qcChart";
export const QC_RULES_KEY = "qcRules";
export const QC_BLOCKS_KEY = "qcBlocks";
export const QC_BLOCK_COUNT_KEY = "qcBlockCount";
export const QC_REVIEWS_KEY = "qcReviews";
export const QC_EXCEPTIONS_KEY = "qcExceptions";
export const QC_EXAMS_KEY = "qcExams";
export const EQA_ROUNDS_KEY = "eqaRounds";
export const EQA_SUMMARY_KEY = "eqaSummary";

const QC_KEYS = [
  QC_INSTRUMENTS_KEY, QC_MATERIALS_KEY, QC_LOTS_KEY, QC_TARGETS_KEY, QC_SUGGESTIONS_KEY, QC_RUNS_KEY, QC_CHART_KEY, QC_RULES_KEY,
  QC_BLOCKS_KEY, QC_BLOCK_COUNT_KEY, QC_REVIEWS_KEY, QC_EXCEPTIONS_KEY, EQA_ROUNDS_KEY, EQA_SUMMARY_KEY
];

export const QC_DISABLED = "o controle de qualidade do laboratório está desligado nesta cidade";
export const INSTRUMENTS_DISABLED = "o controle de qualidade e o estoque do laboratório estão desligados nesta cidade";
export const QC_LEAD_ONLY = "só o administrador ou o responsável técnico (ou o substituto) cadastra materiais, lotes, referências e regras";
export const RT_ONLY = "só o responsável técnico ou o substituto faz isto";
export const REASON_MIN = 10;

const RULE_LABEL: Record<string, string> = { "1_2s": "1-2s (aviso)", "1_3s": "1-3s", "2_2s": "2-2s", "R_4s": "R-4s", "4_1s": "4-1s", "10x": "10x" };
export function ruleLabel(r: string): string { return RULE_LABEL[r] ?? r; }
export function rulesText(rules: string[]): string { return rules.map(ruleLabel).join(", "); }

const RUN_STATUS: Record<string, string> = { open: "aberta", approved: "aprovada", warning: "aprovada com aviso", rejected: "reprovada" };
const RUN_TONE: Record<string, Tone> = { open: "info", approved: "ok", warning: "warn", rejected: "down" };
export function runStatusLabel(s: string): string { return RUN_STATUS[s] ?? s; }
export function runStatusTone(s: string): Tone | undefined { return RUN_TONE[s]; }

const OUTCOME: Record<string, string> = { ok: "ok", warning: "aviso", reject: "reprovado" };
const OUTCOME_TONE: Record<string, Tone> = { ok: "ok", warning: "warn", reject: "down" };
export function outcomeLabel(s: string): string { return OUTCOME[s] ?? s; }
export function outcomeTone(s: string): Tone | undefined { return OUTCOME_TONE[s]; }

const SOURCE: Record<string, string> = { package_insert: "bula", laboratory: "laboratório" };
export function targetSourceLabel(s: string): string { return SOURCE[s] ?? s; }

const DECISION: Record<string, string> = { pending: "pendente", kept: "mantido", rectify: "a retificar" };
export function reviewDecisionLabel(s: string): string { return DECISION[s] ?? s; }

const ADEQUACY: Record<string, string> = { adequate: "adequado", inadequate: "inadequado" };
export function adequacyLabel(s: string): string { return ADEQUACY[s] ?? s; }
export function adequacyTone(s: string): Tone | undefined { return s === "adequate" ? "ok" : s === "inadequate" ? "down" : undefined; }

const LOT_STATUS: Record<string, string> = { in_use: "em uso", closed: "encerrado" };
export function lotStatusLabel(s: string): string { return LOT_STATUS[s] ?? s; }

export function analyteName(exams: LabExamRef[], examId: string, key: string): string {
  const exam = exams.find((e) => e.id === examId);
  const analyte = exam?.analytes?.find((a) => a.key === key);
  return exam && analyte ? `${analyte.name} (${exam.name})` : key;
}

// ─── números ───
export function fmtDecimal(raw: string | number): string { return String(raw).replace(".", ","); }
export function decimalsOf(raw: string | number): number { return (String(raw).split(".")[1] ?? "").length; }
export function fmtZ(z: number | null): string {
  if (z === null || !Number.isFinite(z)) return "—";
  const t = Math.abs(z).toFixed(1).replace(".", ",");
  return z > 0.04 ? `+${t}` : z < -0.04 ? `-${t}` : t;
}

export function decimalProblem(raw: string, label: string, opts: { positive?: boolean; allowNegative?: boolean } = {}): string | null {
  const t = raw.trim();
  if (t === "") return `informe ${label}`;
  if (hasDot(t)) return DOT_IN_NUMBER;
  const n = normalizeNumber(t);
  if (n === null) return `${label}: use dígitos e vírgula`;
  if (!opts.allowNegative && n.startsWith("-")) return `${label} não pode ser negativ${label.startsWith("a ") ? "a" : "o"}`;
  if (opts.positive && Number(n) <= 0) return `${label} precisa ser maior que zero`;
  return null;
}

export function reasonProblem(text: string, what: string): string | null {
  return text.trim().length < REASON_MIN ? `escreva ${what} (ao menos ${REASON_MIN} caracteres)` : null;
}

const tooLong = (value: string, max: number, label: string) => (value.trim().length > max ? `${label} com até ${max} caracteres` : null);

// ─── aparelho ───
export interface InstrumentDraft { name: string; manufacturer: string; model: string; serialNumber: string; sector: string; hl7EquipmentId: string; active: boolean }
export const EMPTY_INSTRUMENT: InstrumentDraft = { name: "", manufacturer: "", model: "", serialNumber: "", sector: "", hl7EquipmentId: "", active: true };

export function instrumentDraft(i: LabInstrument): InstrumentDraft {
  return {
    name: i.name, manufacturer: i.manufacturer, model: i.model, serialNumber: i.serial_number, sector: i.sector,
    hl7EquipmentId: i.hl7_equipment_id ?? "", active: i.active
  };
}

export function instrumentProblem(d: InstrumentDraft): string | null {
  if (!d.name.trim()) return "informe o nome do aparelho";
  const long = tooLong(d.name, 120, "nome") ?? tooLong(d.manufacturer, 120, "fabricante") ?? tooLong(d.model, 120, "modelo")
    ?? tooLong(d.serialNumber, 120, "número de série");
  if (long) return long;
  if (!(d.sector in SECTOR_LABEL)) return "escolha o setor";
  const hl7 = d.hl7EquipmentId.trim();
  if (hl7 && !/^[A-Za-z0-9._-]{1,40}$/.test(hl7)) return "identificador HL7 com até 40 letras, números, ponto, hífen ou _";
  return null;
}

export function instrumentBody(d: InstrumentDraft, providerId: string): LabInstrumentInput {
  return {
    provider_id: providerId, name: d.name.trim(), manufacturer: d.manufacturer.trim(), model: d.model.trim(), serial_number: d.serialNumber.trim(),
    sector: d.sector, hl7_equipment_id: d.hl7EquipmentId.trim() || null, active: d.active
  };
}

// ─── material e lote ───
export interface MaterialDraft { name: string; manufacturer: string; levels: string; analytes: string[]; active: boolean }
export const EMPTY_MATERIAL: MaterialDraft = { name: "", manufacturer: "", levels: "", analytes: [], active: true };
export const analyteToken = (a: QcAnalyteRef) => `${a.lab_exam_id}:${a.analyte_key}`;
export function tokenToAnalyte(token: string): QcAnalyteRef {
  const i = token.indexOf(":");
  return { lab_exam_id: token.slice(0, i), analyte_key: token.slice(i + 1) };
}

export function materialDraft(m: QcMaterial): MaterialDraft {
  return { name: m.name, manufacturer: m.manufacturer, levels: m.levels.join("; "), analytes: m.analytes.map(analyteToken), active: m.active };
}

export function parseLevels(raw: string): string[] {
  return raw.split(";").map((l) => l.trim()).filter((l) => l !== "");
}

export function materialProblem(d: MaterialDraft): string | null {
  if (!d.name.trim()) return "informe o nome do material";
  const long = tooLong(d.name, 120, "nome") ?? tooLong(d.manufacturer, 120, "fabricante");
  if (long) return long;
  const levels = parseLevels(d.levels);
  if (levels.length < 1 || levels.length > 6) return "informe de 1 a 6 níveis separados por ponto e vírgula";
  if (new Set(levels.map((l) => l.toLowerCase())).size !== levels.length) return "níveis repetidos";
  if (levels.some((l) => l.length > 40)) return "cada nível com até 40 caracteres";
  if (d.analytes.length === 0) return "escolha ao menos um analito";
  return null;
}

export function materialBody(d: MaterialDraft, providerId: string): QcMaterialInput {
  return {
    provider_id: providerId, name: d.name.trim(), manufacturer: d.manufacturer.trim(), levels: parseLevels(d.levels),
    analytes: d.analytes.map(tokenToAnalyte), active: d.active
  };
}

export interface LotDraft { materialId: string; lotNumber: string; expiresOn: string; barcodes: Record<string, string> }

export function lotProblem(d: LotDraft, levels: string[], today: string): string | null {
  if (!d.materialId) return "escolha o material";
  if (!d.lotNumber.trim()) return "informe o número do lote";
  const long = tooLong(d.lotNumber, 40, "número do lote");
  if (long) return long;
  if (!/^\d{4}-\d{2}-\d{2}$/.test(d.expiresOn)) return "informe a validade";
  if (d.expiresOn < today) return "lote vencido — confira a validade";
  const seen = new Set<string>();
  for (const level of levels) {
    const code = (d.barcodes[level] ?? "").trim();
    if (!code) return `informe o código de barras do nível ${level}`;
    if (!/^[A-Za-z0-9-]{4,40}$/.test(code)) return `código do nível ${level}: 4 a 40 letras, números ou hífen`;
    if (seen.has(code.toUpperCase())) return "cada nível precisa de um código de barras diferente";
    seen.add(code.toUpperCase());
  }
  return null;
}

export function lotBody(d: LotDraft, levels: string[]): QcLotInput {
  return {
    material_id: d.materialId, lot_number: d.lotNumber.trim(), expires_on: d.expiresOn,
    level_barcodes: Object.fromEntries(levels.map((l) => [ l, (d.barcodes[l] ?? "").trim() ]))
  };
}

// ─── referência, regras e valor ───
export interface TargetDraft { level: string; analyteKey: string; instrumentId: string; mean: string; sd: string; reason: string }
export const EMPTY_TARGET: TargetDraft = { level: "", analyteKey: "", instrumentId: "", mean: "", sd: "", reason: "" };

export function targetProblem(d: TargetDraft, replacing: boolean): string | null {
  if (!d.level) return "escolha o nível";
  if (!d.analyteKey) return "escolha o analito";
  if (!d.instrumentId) return "escolha o aparelho";
  const problem = decimalProblem(d.mean, "a média", { allowNegative: true }) ?? decimalProblem(d.sd, "o desvio-padrão", { positive: true });
  if (problem) return problem;
  return replacing ? reasonProblem(d.reason, "o motivo da troca da referência") : null;
}

export function targetBody(d: TargetDraft, lotId: string, replacing: boolean): QcTargetInput {
  return {
    lot_id: lotId, level: d.level, analyte_key: d.analyteKey, instrument_id: d.instrumentId,
    mean: normalizeNumber(d.mean) as string, sd: normalizeNumber(d.sd) as string,
    ...(replacing ? { reason: d.reason.trim() } : {})
  };
}

export function turnedOff(before: Record<QcRule, boolean>, after: Record<QcRule, boolean>): QcRule[] {
  return (Object.keys(before) as QcRule[]).filter((r) => before[r] && !after[r]);
}

export function rulesProblem(before: Record<QcRule, boolean>, after: Record<QcRule, boolean>, justification: string): string | null {
  const off = turnedOff(before, after);
  return off.length > 0 ? reasonProblem(justification, `a justificativa para desligar ${rulesText(off)}`) : null;
}

export function qcValueProblem(raw: string): string | null {
  return decimalProblem(raw, "o valor", { allowNegative: true });
}

// ─── Levey-Jennings ───
// O eixo é em desvios-padrão: cada ponto chega com o z da referência da sua
// versão (o api avaliou na gravação). Reprojetar valores antigos na média nova
// mostraria um gráfico que não foi o avaliado.
export interface LjRow { n: number; z: number; at: string; value: string; outcome: string; rules: string[]; version: number }
export interface LjModel { rows: LjRow[]; versionChanges: { n: number; version: number }[]; domain: [ number, number ]; flagged: LjRow[]; mixedVersions: boolean }
export const SD_LINES = [ -3, -2, -1, 0, 1, 2, 3 ];

export function ljModel(chart: QcChart): LjModel {
  const rows: LjRow[] = chart.points.map((p, i) => ({
    n: i + 1, z: p.z_score, at: p.at, value: p.value, outcome: p.outcome, rules: p.rules_fired, version: p.target_version
  }));
  const versionChanges = rows.filter((r, i) => i > 0 && r.version !== rows[i - 1].version).map((r) => ({ n: r.n, version: r.version }));
  const peak = rows.reduce((m, r) => Math.max(m, Math.abs(r.z)), 0);
  const edge = Math.max(4, Math.ceil(peak + 0.5));
  const current = chart.target?.version;
  return {
    rows, versionChanges, domain: [ -edge, edge ], flagged: rows.filter((r) => r.outcome !== "ok"),
    mixedVersions: versionChanges.length > 0 || (current !== undefined && rows.some((r) => r.version !== current))
  };
}

export function sdLabel(k: number): string {
  return k === 0 ? "média" : `${k > 0 ? "+" : "-"}${Math.abs(k)} dp`;
}

export function ljBands(target: { mean: string; sd: string }): { k: number; label: string; value: string }[] {
  const places = Math.max(decimalsOf(target.mean), decimalsOf(target.sd));
  const mean = Number(target.mean);
  const sd = Number(target.sd);
  return [ 3, 2, 1, 0, -1, -2, -3 ].map((k) => ({ k, label: sdLabel(k), value: fmtDecimal((mean + k * sd).toFixed(places)) }));
}

// ─── CEQ ───
export interface EqaItemDraft { token: string; submitted: string; target: string; low: string; high: string; adequacy: EqaAdequacy | ""; action: string }
export const EMPTY_EQA_ITEM: EqaItemDraft = { token: "", submitted: "", target: "", low: "", high: "", adequacy: "", action: "" };
export interface EqaRoundDraft {
  program: "PNCQ" | "Controllab" | "outro" | ""; programOther: string; roundCode: string; roundDate: string; items: EqaItemDraft[]; uploadId: string | null;
}
export const EMPTY_EQA_ROUND: EqaRoundDraft = { program: "", programOther: "", roundCode: "", roundDate: "", items: [ EMPTY_EQA_ITEM ], uploadId: null };

const dec = (raw: string) => normalizeNumber(raw);

export function suggestAdequacy(i: EqaItemDraft): EqaAdequacy | "" {
  const v = dec(i.submitted), lo = dec(i.low), hi = dec(i.high);
  if (v === null || lo === null || hi === null) return "";
  return Number(v) >= Number(lo) && Number(v) <= Number(hi) ? "adequate" : "inadequate";
}

export function eqaProblem(d: EqaRoundDraft, today: string): string | null {
  if (!d.program) return "escolha o programa";
  if (d.program === "outro" && !d.programOther.trim()) return "informe o nome do programa";
  if (!d.roundCode.trim()) return "informe o código da rodada";
  const long = tooLong(d.roundCode, 40, "código da rodada") ?? tooLong(d.programOther, 80, "nome do programa");
  if (long) return long;
  if (!/^\d{4}-\d{2}-\d{2}$/.test(d.roundDate)) return "informe a data da rodada";
  if (d.roundDate > today) return "a data da rodada não pode ser no futuro";
  if (d.items.length === 0) return "inclua ao menos um analito";
  for (const [ idx, i ] of d.items.entries()) {
    const n = `item ${idx + 1}`;
    if (!i.token) return `${n}: escolha o analito`;
    const num = decimalProblem(i.submitted, "o valor enviado", { allowNegative: true }) ?? decimalProblem(i.target, "o alvo", { allowNegative: true })
      ?? (i.low.trim() ? decimalProblem(i.low, "o limite inferior", { allowNegative: true }) : null)
      ?? (i.high.trim() ? decimalProblem(i.high, "o limite superior", { allowNegative: true }) : null);
    if (num) return `${n}: ${num}`;
    if (i.low.trim() && i.high.trim() && Number(dec(i.low)) > Number(dec(i.high))) return `${n}: o limite inferior é maior que o superior`;
    if (!i.adequacy) return `${n}: diga se foi adequado`;
    if (i.adequacy === "inadequate") {
      const why = reasonProblem(i.action, "a ação para o resultado inadequado");
      if (why) return `${n}: ${why}`;
    }
  }
  return null;
}

export function eqaBody(d: EqaRoundDraft, providerId: string): EqaRoundInput {
  return {
    provider_id: providerId, program: d.program === "outro" ? d.programOther.trim() : d.program, round_code: d.roundCode.trim(),
    round_date: d.roundDate, annual_report_upload_id: d.uploadId,
    items: d.items.map((i) => {
      const ref = tokenToAnalyte(i.token);
      return {
        lab_exam_id: ref.lab_exam_id, analyte_key: ref.analyte_key, submitted: dec(i.submitted) as string, target: dec(i.target) as string,
        acceptable_low: i.low.trim() ? dec(i.low) : null, acceptable_high: i.high.trim() ? dec(i.high) : null,
        adequacy: i.adequacy as EqaAdequacy, action: i.adequacy === "inadequate" ? i.action.trim() : null
      };
    })
  };
}

export function adequacyRate(row: EqaSummaryRow): string {
  if (row.rounds <= 0) return "—";
  return `${fmtDecimal(Math.round((row.adequate * 1000) / row.rounds) / 10)}%`;
}

// ─── recusas ───
const ERRORS: Record<string, string> = {
  invalid_name: "nome inválido — preencha o nome",
  duplicate_hl7_equipment_id: "este identificador HL7 já está em outro aparelho",
  barcode_taken: "este código de barras já está em outro lote de controle ou numa amostra",
  invalid_levels: "níveis inválidos — informe de 1 a 6 nomes distintos",
  expired_lot: "lote vencido — não pode ser usado",
  no_target: "este nível e analito ainda não têm referência neste aparelho — cadastre a da bula em Controles e referências",
  lot_closed: "este lote de controle está encerrado",
  invalid_value: "valor inválido — use dígitos e vírgula",
  justification_required: "escreva a justificativa",
  reason_required: "escreva o motivo da troca da referência",
  corrective_action_required: "a corrida aprovou, mas o bloqueio continua: registre a ação corretiva e rode o controle de novo",
  not_blocked: "este exame não está mais bloqueado — libere normalmente",
  action_required: "escreva a ação de cada item inadequado",
  qc_blocked: "o controle de qualidade reprovou e bloqueou exames desta requisição",
  missing_role: "seu papel não permite esta ação"
};

export function qcError(err: unknown): string {
  const feature = featureDisabledKey(err);
  if (feature === "lab_quality_control") return QC_DISABLED;
  if (feature === "lab_inventory") return "o estoque do laboratório está desligado nesta cidade";
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  return labError(err);
}

export async function refreshQc(client: QueryClient): Promise<void> {
  await Promise.all(QC_KEYS.map((key) => client.invalidateQueries({ queryKey: [ key ] })));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/qc.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/qc.ts src/lib/qc.test.ts
$G commit -m "feat: add the lab quality control screen rules and Levey-Jennings model

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Regras do estoque (`src/lib/inventory.ts`)

**Files:**
- Create: `src/lib/inventory.ts`
- Test: `src/lib/inventory.test.ts`

**Interfaces:**
- Consumes: os tipos da Task 1; `errorCode`; `featureDisabledKey`; `labError` (`lab.ts`); `QC_DISABLED` (`qc.ts`); `Tone`.
- Produces: chaves `INV_SUPPLIES_KEY`, `INV_LOTS_KEY`, `INV_MOVEMENTS_KEY`, `INV_IN_USE_KEY`, `INV_ALERTS_KEY`, `INV_CONSUMPTION_KEY`; `INVENTORY_DISABLED`, `UNTRACED`; rótulos `SUPPLY_KIND_LABEL`, `supplyKindLabel`, `SUPPLY_ORIGIN_LABEL`, `originLabel`, `movementLabel`; `integerProblem`; `SupplyDraft`/`EMPTY_SUPPLY`/`supplyDraft`/`supplyProblem`/`supplyBody`; `LotEntryDraft`/`EMPTY_LOT_ENTRY`/`lotEntryProblem`/`lotEntryBody`; `MovementDraft`/`EMPTY_MOVEMENT`/`movementProblem`/`movementBody`; `reagentLotMark(row, inventoryOn)`; `fmtDeviation`, `deviationTone`; `inventoryError`, `refreshInventory(client)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/inventory.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  EMPTY_LOT_ENTRY, EMPTY_MOVEMENT, EMPTY_SUPPLY, INVENTORY_DISABLED, UNTRACED, deviationTone, fmtDeviation, integerProblem, inventoryError,
  lotEntryBody, lotEntryProblem, movementBody, movementProblem, reagentLotMark, supplyBody, supplyProblem
} from "./inventory";

describe("inteiros", () => {
  it("só dígitos, mínimo e sinal quando permitido", () => {
    expect(integerProblem("3", "a quantidade", { min: 1 })).toBeNull();
    expect(integerProblem("1,5", "a quantidade", { min: 1 })).toBe("a quantidade é um número inteiro");
    expect(integerProblem("0", "a quantidade", { min: 1 })).toBe("a quantidade precisa ser ao menos 1");
    expect(integerProblem("-2", "o ajuste", { allowNegative: true })).toBeNull();
    expect(integerProblem("", "o mínimo", { min: 0 })).toBe("informe o mínimo");
  });
});

describe("item de estoque", () => {
  const reagent = { ...EMPTY_SUPPLY, name: "Glicose GOD-PAP", kind: "reagent" as const, unit: "kit", testsPerPackage: "400", minimumStock: "2", examIds: [ "le1" ] };
  it("reagente exige rendimento e exame ligado; os outros não", () => {
    expect(supplyProblem(reagent)).toBeNull();
    expect(supplyProblem({ ...reagent, testsPerPackage: "" })).toBe("informe quantos testes rende cada embalagem do reagente");
    expect(supplyProblem({ ...reagent, examIds: [] })).toBe("ligue o reagente a ao menos um exame");
    expect(supplyProblem({ ...reagent, kind: "consumable", testsPerPackage: "", examIds: [] })).toBeNull();
    expect(supplyProblem({ ...reagent, expiryAlertDays: "400" })).toBe("aviso de vencimento de 1 a 365 dias");
  });
  it("corpo: rendimento só do reagente; padrão de 30 dias", () => {
    expect(supplyBody(reagent, "pv9")).toEqual({
      provider_id: "pv9", name: "Glicose GOD-PAP", kind: "reagent", unit: "kit", tests_per_package: 400, minimum_stock: 2,
      expiry_alert_days: 30, lab_exam_ids: [ "le1" ], active: true
    });
    expect(supplyBody({ ...reagent, kind: "consumable" }, "pv9").tests_per_package).toBeNull();
  });
});

describe("entrada de lote", () => {
  const d = { ...EMPTY_LOT_ENTRY, supplyId: "su1", lotNumber: "R-778", expiresOn: "2027-01-31", supplier: "Distribuidora Sul", origin: "purchase" as const, quantity: "6" };
  it("lote vencido não entra; quantidade inteira", () => {
    expect(lotEntryProblem(d, "2026-10-18")).toBeNull();
    expect(lotEntryProblem({ ...d, expiresOn: "2026-10-17" }, "2026-10-18")).toBe("lote vencido — não entra no estoque");
    expect(lotEntryProblem({ ...d, quantity: "2,5" }, "2026-10-18")).toBe("a quantidade é um número inteiro");
    expect(lotEntryBody({ ...d, document: " NF 1234 " })).toEqual({
      supply_id: "su1", lot_number: "R-778", expires_on: "2027-01-31", supplier: "Distribuidora Sul", origin: "purchase", quantity: 6, document: "NF 1234"
    });
    expect(lotEntryBody(d)).not.toHaveProperty("document");
  });
});

describe("movimento", () => {
  it("saída maior que o saldo e ajuste sem admin", () => {
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "opening", quantity: "1" }, 5, false)).toBeNull();
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "loss", quantity: "6", reason: "frasco quebrado" }, 5, false)).toBe("saída maior que o saldo do lote (5)");
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "loss", quantity: "1", reason: "" }, 5, false)).toBe("escreva o motivo");
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "adjustment", quantity: "-1", reason: "contagem física" }, 5, false)).toBe("ajuste é só do administrador");
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "adjustment", quantity: "-6", reason: "contagem física" }, 5, true)).toBe("saída maior que o saldo do lote (5)");
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "adjustment", quantity: "0", reason: "contagem física" }, 5, true)).toBe("o ajuste não pode ser zero");
    expect(movementProblem({ ...EMPTY_MOVEMENT, kind: "adjustment", quantity: "2", reason: "contagem física" }, 5, true)).toBeNull();
  });
  it("corpo com quantidade inteira", () => {
    expect(movementBody({ kind: "adjustment", quantity: "-1", reason: " contagem física " }, "sl1")).toEqual({ lot_id: "sl1", kind: "adjustment", quantity: -1, reason: "contagem física" });
  });
});

describe("lote no resultado", () => {
  it("número do lote; null com o estoque ligado = sem rastreio; ausente = nada", () => {
    expect(reagentLotMark({ reagent_lot: { id: "sl1", lot_number: "R-778" } }, true)).toEqual({ text: "lote R-778", untraced: false });
    expect(reagentLotMark({ reagent_lot: null }, true)).toEqual({ text: UNTRACED, untraced: true });
    expect(reagentLotMark({ reagent_lot: null }, false)).toBeNull();
    expect(reagentLotMark({}, true)).toBeNull();
  });
});

describe("consumo e recusas", () => {
  it("desvio com sinal e vírgula; acima de 20% acende", () => {
    expect(fmtDeviation(12.5)).toBe("+12,5%");
    expect(fmtDeviation(-30)).toBe("-30,0%");
    expect(fmtDeviation(null)).toBe("—");
    expect(deviationTone(-30)).toBe("warn");
    expect(deviationTone(5)).toBeUndefined();
  });
  it("frases do estoque; motivo do estoque não é o da rejeição de amostra", () => {
    expect(inventoryError(new ApiError(403, { error: "feature_disabled", feature: "lab_inventory" }, "x"))).toBe(INVENTORY_DISABLED);
    expect(inventoryError(new ApiError(422, { error: "insufficient_balance" }, "x"))).toBe("saída maior que o saldo do lote — o saldo foi relido");
    expect(inventoryError(new ApiError(422, { error: "reason_required" }, "x"))).toBe("escreva o motivo");
    expect(inventoryError(new ApiError(422, { error: "exam_not_linked" }, "x"))).toBe("este exame não está ligado ao item — o administrador liga no cadastro do item");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/inventory.test.ts`
Expected: FAIL — `Cannot find module './inventory'`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/inventory.ts
// Estoque do laboratório municipal (módulo 21c; contrato 2026-10-18 §5 e §7).
// Regras de tela, puras. O saldo de verdade, o lote em uso e a recusa de
// saída maior que o saldo são do api; aqui só não se envia o que já se sabe
// que será recusado. Nada é baixado sozinho (spec §6.8).
import type { QueryClient } from "@tanstack/react-query";
import {
  errorCode,
  type LabSupply, type LabSupplyInput, type LabSupplyLotInput, type LabSupplyMovementInput, type ReagentLotRef, type SupplyKind, type SupplyOrigin
} from "./api";
import { featureDisabledKey } from "./features";
import { labError } from "./lab";
import { QC_DISABLED } from "./qc";
import type { Tone } from "../theme/tokens";

export const INV_SUPPLIES_KEY = "invSupplies";
export const INV_LOTS_KEY = "invLots";
export const INV_MOVEMENTS_KEY = "invMovements";
export const INV_IN_USE_KEY = "invInUse";
export const INV_ALERTS_KEY = "invAlerts";
export const INV_CONSUMPTION_KEY = "invConsumption";
const INV_KEYS = [ INV_SUPPLIES_KEY, INV_LOTS_KEY, INV_MOVEMENTS_KEY, INV_IN_USE_KEY, INV_ALERTS_KEY, INV_CONSUMPTION_KEY ];

export const INVENTORY_DISABLED = "o estoque do laboratório está desligado nesta cidade";
export const UNTRACED = "sem rastreio de reagente";

export const SUPPLY_KIND_LABEL: Record<SupplyKind, string> = { reagent: "Reagente", calibrator: "Calibrador", control: "Controle", consumable: "Consumível" };
export function supplyKindLabel(k: string): string { return (SUPPLY_KIND_LABEL as Record<string, string>)[k] ?? k; }
export const SUPPLY_ORIGIN_LABEL: Record<SupplyOrigin, string> = { purchase: "compra", donation: "doação", loan: "empréstimo" };
export function originLabel(o: string): string { return (SUPPLY_ORIGIN_LABEL as Record<string, string>)[o] ?? o; }
const MOVEMENT: Record<string, string> = { receipt: "entrada", opening: "abertura", loss: "perda", expiry: "vencimento", adjustment: "ajuste" };
export function movementLabel(k: string): string { return MOVEMENT[k] ?? k; }

export function integerProblem(raw: string, label: string, opts: { min?: number; max?: number; allowNegative?: boolean } = {}): string | null {
  const t = raw.trim();
  if (t === "") return `informe ${label}`;
  if (!(opts.allowNegative ? /^-?\d+$/ : /^\d+$/).test(t)) return `${label} é um número inteiro`;
  const n = Number(t);
  if (opts.min !== undefined && n < opts.min) return `${label} precisa ser ao menos ${opts.min}`;
  if (opts.max !== undefined && n > opts.max) return `${label} pode ser no máximo ${opts.max}`;
  return null;
}

const tooLong = (value: string, max: number, label: string) => (value.trim().length > max ? `${label} com até ${max} caracteres` : null);

export interface SupplyDraft {
  name: string; kind: SupplyKind | ""; unit: string; testsPerPackage: string; minimumStock: string; expiryAlertDays: string; examIds: string[]; active: boolean;
}
export const EMPTY_SUPPLY: SupplyDraft = { name: "", kind: "", unit: "", testsPerPackage: "", minimumStock: "0", expiryAlertDays: "30", examIds: [], active: true };

export function supplyDraft(s: LabSupply): SupplyDraft {
  return {
    name: s.name, kind: s.kind as SupplyKind, unit: s.unit, testsPerPackage: s.tests_per_package === null ? "" : String(s.tests_per_package),
    minimumStock: String(s.minimum_stock), expiryAlertDays: String(s.expiry_alert_days), examIds: s.lab_exam_ids, active: s.active
  };
}

export function supplyProblem(d: SupplyDraft): string | null {
  if (!d.name.trim()) return "informe o nome do item";
  if (!d.kind) return "escolha o tipo do item";
  if (!d.unit.trim()) return "informe a unidade (ex.: kit, frasco, caixa)";
  const long = tooLong(d.name, 120, "nome") ?? tooLong(d.unit, 20, "unidade");
  if (long) return long;
  if (d.kind === "reagent" && !d.testsPerPackage.trim()) return "informe quantos testes rende cada embalagem do reagente";
  if (d.testsPerPackage.trim()) {
    const p = integerProblem(d.testsPerPackage, "o rendimento", { min: 1 });
    if (p) return p;
  }
  const min = integerProblem(d.minimumStock, "o estoque mínimo", { min: 0 });
  if (min) return min;
  if (integerProblem(d.expiryAlertDays, "o aviso", { min: 1, max: 365 })) return "aviso de vencimento de 1 a 365 dias";
  if (d.kind === "reagent" && d.examIds.length === 0) return "ligue o reagente a ao menos um exame";
  return null;
}

export function supplyBody(d: SupplyDraft, providerId: string): LabSupplyInput {
  return {
    provider_id: providerId, name: d.name.trim(), kind: d.kind as SupplyKind, unit: d.unit.trim(),
    tests_per_package: d.kind === "reagent" && d.testsPerPackage.trim() ? Number(d.testsPerPackage) : null,
    minimum_stock: Number(d.minimumStock), expiry_alert_days: Number(d.expiryAlertDays), lab_exam_ids: d.examIds, active: d.active
  };
}

export interface LotEntryDraft { supplyId: string; lotNumber: string; expiresOn: string; supplier: string; origin: SupplyOrigin | ""; quantity: string; document: string }
export const EMPTY_LOT_ENTRY: LotEntryDraft = { supplyId: "", lotNumber: "", expiresOn: "", supplier: "", origin: "", quantity: "", document: "" };

export function lotEntryProblem(d: LotEntryDraft, today: string): string | null {
  if (!d.supplyId) return "escolha o item";
  if (!d.lotNumber.trim()) return "informe o número do lote";
  if (!/^\d{4}-\d{2}-\d{2}$/.test(d.expiresOn)) return "informe a validade";
  if (d.expiresOn < today) return "lote vencido — não entra no estoque";
  if (!d.supplier.trim()) return "informe o fornecedor";
  if (!d.origin) return "escolha a origem";
  const long = tooLong(d.lotNumber, 40, "número do lote") ?? tooLong(d.supplier, 120, "fornecedor") ?? tooLong(d.document, 60, "documento");
  if (long) return long;
  return integerProblem(d.quantity, "a quantidade", { min: 1 });
}

export function lotEntryBody(d: LotEntryDraft): LabSupplyLotInput {
  const doc = d.document.trim();
  return {
    supply_id: d.supplyId, lot_number: d.lotNumber.trim(), expires_on: d.expiresOn, supplier: d.supplier.trim(), origin: d.origin as SupplyOrigin,
    quantity: Number(d.quantity), ...(doc ? { document: doc } : {})
  };
}

export interface MovementDraft { kind: LabSupplyMovementInput["kind"] | ""; quantity: string; reason: string }
export const EMPTY_MOVEMENT: MovementDraft = { kind: "", quantity: "", reason: "" };

// Abertura sem motivo; perda, vencimento e ajuste com motivo (spec §6.4).
export function movementProblem(d: MovementDraft, balance: number, isAdmin: boolean): string | null {
  if (!d.kind) return "escolha o movimento";
  if (d.kind === "adjustment" && !isAdmin) return "ajuste é só do administrador";
  const adjustment = d.kind === "adjustment";
  const q = integerProblem(d.quantity, adjustment ? "o ajuste" : "a quantidade", adjustment ? { allowNegative: true } : { min: 1 });
  if (q) return q;
  const n = Number(d.quantity);
  if (adjustment && n === 0) return "o ajuste não pode ser zero";
  const out = adjustment ? -n : n;
  if (out > balance) return `saída maior que o saldo do lote (${balance})`;
  if (d.kind !== "opening" && d.reason.trim().length < 3) return "escreva o motivo";
  return tooLong(d.reason, 200, "motivo");
}

export function movementBody(d: MovementDraft, lotId: string): LabSupplyMovementInput {
  return { lot_id: lotId, kind: d.kind as LabSupplyMovementInput["kind"], quantity: Number(d.quantity), reason: d.reason.trim() };
}

// Contrato §7: `reagent_lot` null = gravado sem lote em uso. Ausente = api
// antigo ou quem não é do laboratório: nada aparece. Com o estoque desligado
// todo resultado viria null — a marca não aparece para não acender tudo.
export function reagentLotMark(row: { reagent_lot?: ReagentLotRef | null }, inventoryOn: boolean): { text: string; untraced: boolean } | null {
  if (row.reagent_lot) return { text: `lote ${row.reagent_lot.lot_number}`, untraced: false };
  if (row.reagent_lot === null && inventoryOn) return { text: UNTRACED, untraced: true };
  return null;
}

export function fmtDeviation(pct: number | null): string {
  if (pct === null || !Number.isFinite(pct)) return "—";
  const t = Math.abs(pct).toFixed(1).replace(".", ",");
  return pct > 0 ? `+${t}%` : pct < 0 ? `-${t}%` : `${t}%`;
}

export function deviationTone(pct: number | null): Tone | undefined {
  return pct !== null && Math.abs(pct) > 20 ? "warn" : undefined;
}

const ERRORS: Record<string, string> = {
  expired_lot: "lote vencido — não pode ser usado",
  insufficient_balance: "saída maior que o saldo do lote — o saldo foi relido",
  reason_required: "escreva o motivo",
  lot_not_reagent: "só lote de reagente fica em uso no aparelho",
  exam_not_linked: "este exame não está ligado ao item — o administrador liga no cadastro do item",
  invalid_quantity: "quantidade inválida",
  missing_role: "seu papel não permite esta ação"
};

export function inventoryError(err: unknown): string {
  const feature = featureDisabledKey(err);
  if (feature === "lab_inventory") return INVENTORY_DISABLED;
  if (feature === "lab_quality_control") return QC_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  return labError(err);
}

export async function refreshInventory(client: QueryClient): Promise<void> {
  await Promise.all(INV_KEYS.map((key) => client.invalidateQueries({ queryKey: [ key ] })));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/inventory.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/inventory.ts src/lib/inventory.test.ts
$G commit -m "feat: add the lab inventory screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Menu, catálogo e contador de bloqueios ativos

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/catalog.ts`, `src/App.tsx` (`shellGroups`, `ShellInner`), `src/lib/qc.ts` (`canUseQc`, `canUseInventory`), `src/shell/modules.lab.test.ts` (`LAB_IDS` e o título do primeiro teste), `src/App.navigation.test.tsx`
- Create: `src/hooks/useQcBlockCount.ts`
- Test: `src/shell/modules.qc.test.ts`, `src/hooks/useQcBlockCount.test.tsx`

**Interfaces:**
- Consumes: `countActiveQcBlocks` (Task 1); `QC_BLOCK_COUNT_KEY` (Task 2); `canUseLab`, `labRoles` (`lab.ts`); `hasFeature`.
- Produces: `ModuleId` ganha `"lab-qc-runs" | "lab-qc-chart" | "lab-qc-blocks" | "lab-qc-setup" | "lab-eqa" | "lab-inventory" | "lab-traceability" | "lab-instruments"`; `withQcBlockCount(groups, count): NavGroupDef[]`; `shellGroups(groups, pending, examPending = 0, labCritical = 0, qcBlocks = 0)`; `useQcBlockCount(enabled, userId)` e `QC_BLOCK_REFRESH_MS = 60_000`; `canUseQc(user)` e `canUseInventory(user)` em `qc.ts`.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.qc.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor, withQcBlockCount } from "./modules";

const QC_IDS = [ "lab-qc-runs", "lab-qc-chart", "lab-qc-blocks", "lab-qc-setup", "lab-eqa", "lab-inventory", "lab-traceability", "lab-instruments" ];
const BOTH = { features: [ "exam_tracking", "municipal_lab", "lab_quality_control", "lab_inventory" ] };
const userWith = (roles: string[], extra: object = BOTH) => ({ operator: false, memberships: roles.map((role) => ({ role })), ...extra });
const qcIds = (user: Parameters<typeof navGroupsFor>[0]) =>
  navGroupsFor(user).flatMap((g) => g.items.map((i) => i.id)).filter((id) => QC_IDS.includes(id));

describe("telas do 21c no grupo Laboratório", () => {
  it("ficam depois da liberação e antes do admin do laboratório", () => {
    const ids = NAV_GROUPS.find((g) => g.label === "Laboratório")!.items.map((i) => i.id);
    expect(ids.slice(ids.indexOf("lab-release") + 1, ids.indexOf("lab-admin"))).toEqual(QC_IDS);
  });

  it("sem os interruptores do 21c ninguém vê; sem municipal_lab também não", () => {
    for (const role of [ "municipal_admin", "lab_staff", "lab_releaser" ]) {
      expect(qcIds(userWith([ role ], { features: [ "exam_tracking", "municipal_lab" ] }))).toEqual([]);
      expect(qcIds(userWith([ role ], { features: [ "exam_tracking", "lab_quality_control", "lab_inventory" ] }))).toEqual([]);
    }
  });

  it("cada papel vê o seu", () => {
    expect(qcIds(userWith([ "lab_staff" ]))).toEqual([ "lab-qc-runs", "lab-qc-blocks", "lab-inventory", "lab-instruments" ]);
    expect(qcIds(userWith([ "lab_releaser" ]))).toEqual(QC_IDS);
    expect(qcIds(userWith([ "municipal_admin" ]))).toEqual([ "lab-qc-setup", "lab-eqa", "lab-inventory", "lab-traceability", "lab-instruments" ]);
    expect(qcIds(userWith([ "health_professional" ]))).toEqual([]);
    expect(qcIds({ operator: true, memberships: [], ...BOTH })).toEqual([]);
  });

  it("interruptores independentes; aparelhos com qualquer um dos dois", () => {
    expect(qcIds(userWith([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab", "lab_quality_control" ] })))
      .toEqual([ "lab-qc-runs", "lab-qc-blocks", "lab-instruments" ]);
    expect(qcIds(userWith([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab", "lab_inventory" ] })))
      .toEqual([ "lab-inventory", "lab-instruments" ]);
  });

  it("contador de bloqueios ativos no rótulo; zero não muda nada", () => {
    const item = (n: number) => withQcBlockCount(NAV_GROUPS, n).flatMap((g) => g.items).find((i) => i.id === "lab-qc-blocks");
    expect(item(0)?.label).toBe("Bloqueios e revisões do CIQ");
    expect(item(1)?.label).toBe("Bloqueios e revisões do CIQ (1 ativo)");
    expect(item(3)?.label).toBe("Bloqueios e revisões do CIQ (3 ativos)");
  });
});
```

```tsx
// src/hooks/useQcBlockCount.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import type { ReactNode } from "react";
import { act, cleanup, renderHook } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { QC_BLOCK_REFRESH_MS, useQcBlockCount } from "./useQcBlockCount";
import { QC_BLOCK_COUNT_KEY } from "../lib/qc";

const fetchMock = vi.fn();
const urls = () => fetchMock.mock.calls.map((c) => String(c[0]));

function setup(enabled: boolean, userId: string | null = "u1") {
  const client = new QueryClient();
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  const hook = renderHook(() => useQcBlockCount(enabled, userId), { wrapper });
  return { client, ...hook };
}
const flush = () => act(async () => { await vi.advanceTimersByTimeAsync(0); });

beforeEach(() => {
  vi.useFakeTimers();
  fetchMock.mockReset();
  fetchMock.mockImplementation(async () => new Response(JSON.stringify({ count: 2 }), { status: 200, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fetchMock);
});
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

describe("useQcBlockCount", () => {
  it("sem o interruptor ou sem papel (enabled false): nenhuma leitura", async () => {
    setup(false);
    await flush();
    await act(async () => { await vi.advanceTimersByTimeAsync(QC_BLOCK_REFRESH_MS * 3); });
    expect(fetchMock).not.toHaveBeenCalled();
  });

  it("relê só /count a cada minuto e nunca a lista", async () => {
    const { result } = setup(true);
    await flush();
    expect(result.current.data).toBe(2);
    await act(async () => { await vi.advanceTimersByTimeAsync(QC_BLOCK_REFRESH_MS * 2); });
    expect(urls()).toHaveLength(3);
    expect(urls().every((u) => /\/lab\/qc\/blocks\/active\/count$/.test(u))).toBe(true);
    expect(urls().some((u) => /\/lab\/qc\/(blocks|reviews)\?/.test(u))).toBe(false);
    expect(QC_BLOCK_REFRESH_MS).toBe(60_000);
  });

  it("erro do contador: sem número, sem exceção", async () => {
    fetchMock.mockImplementation(async () => new Response(JSON.stringify({ error: "feature_disabled" }), { status: 403, headers: { "Content-Type": "application/json" } }));
    const { result } = setup(true);
    await flush();
    expect(result.current.data).toBeUndefined();
    expect(result.current.isError).toBe(true);
  });

  it("a chave é [QC_BLOCK_COUNT_KEY, userId]", async () => {
    const { client } = setup(true, "u7");
    await flush();
    expect(client.getQueryCache().getAll().map((q) => q.queryKey)).toEqual([ [ QC_BLOCK_COUNT_KEY, "u7" ] ]);
  });
});
```

No `describe("grupos do shell")` de `src/App.navigation.test.tsx`, acrescente:

```ts
  it("Bloqueios do CIQ levam a contagem no rótulo e no ponto do trilho", () => {
    const item = (n: number) => shellGroups(NAV_GROUPS, 0, 0, 0, n).flatMap((g) => g.items).find((i) => i.id === "lab-qc-blocks");
    expect(item(2)).toEqual({ id: "lab-qc-blocks", label: "Bloqueios e revisões do CIQ (2 ativos)", count: 2 });
    expect(item(0)).toEqual({ id: "lab-qc-blocks", label: "Bloqueios e revisões do CIQ" });
  });
```

E em `src/shell/modules.lab.test.ts`, troque a constante e o título do primeiro teste (o resto do arquivo fica igual — as telas do 21c precisam dos interruptores que `ON` não tem):

```ts
const LAB_IDS = [
  "lab-requisitions", "lab-collection", "lab-shipments", "lab-reception", "lab-worklist", "lab-entry", "lab-release",
  "lab-qc-runs", "lab-qc-chart", "lab-qc-blocks", "lab-qc-setup", "lab-eqa", "lab-inventory", "lab-traceability", "lab-instruments",
  "lab-admin", "lab-reports"
];
```

```ts
  it("fica logo depois de Exames, com as telas do 21b e do 21c", () => {
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/shell/modules.qc.test.ts src/hooks/useQcBlockCount.test.tsx src/shell/modules.lab.test.ts src/App.navigation.test.tsx`
Expected: FAIL — `withQcBlockCount` não existe, `useQcBlockCount` não existe, a lista do grupo não bate.

- [ ] **Step 3: Write minimal implementation**

`src/shell/modules.ts`:

```ts
  | "lab-requisitions" | "lab-collection" | "lab-shipments" | "lab-reception" | "lab-worklist" | "lab-entry" | "lab-release"
  | "lab-qc-runs" | "lab-qc-chart" | "lab-qc-blocks" | "lab-qc-setup" | "lab-eqa" | "lab-inventory" | "lab-traceability" | "lab-instruments"
  | "lab-admin" | "lab-reports";
```

No grupo "Laboratório", logo depois de `{ id: "lab-release", … }`:

```ts
    { id: "lab-qc-runs", label: "Corridas de controle", icon: "↻" },
    { id: "lab-qc-chart", label: "Levey-Jennings", icon: "∿" },
    { id: "lab-qc-blocks", label: "Bloqueios e revisões do CIQ", icon: "⊘" },
    { id: "lab-qc-setup", label: "Controles e referências", icon: "◎" },
    { id: "lab-eqa", label: "Controle externo (CEQ)", icon: "◈" },
    { id: "lab-inventory", label: "Estoque do laboratório", icon: "▣" },
    { id: "lab-traceability", label: "Rastreio e consumo de reagentes", icon: "⇶" },
    { id: "lab-instruments", label: "Aparelhos", icon: "⌬" },
```

Em `navGroupsFor`, depois de `const isReleaser = roles.includes("lab_releaser");`:

```ts
  // Módulo 21c: CIQ/CEQ com `lab_quality_control`, estoque com `lab_inventory`
  // (independentes; os dois exigem municipal_lab). Aparelhos servem aos dois.
  const qcOn = labOn && hasFeature(user, "lab_quality_control");
  const invOn = labOn && hasFeature(user, "lab_inventory");
```

E no filtro de itens, depois da linha de `"lab-reports"`:

```ts
      if (item.id === "lab-qc-runs" || item.id === "lab-qc-blocks") return qcOn && (isLabStaff || isReleaser);
      if (item.id === "lab-qc-chart") return qcOn && isReleaser;
      // Materiais, lotes, referências e regras: admin ou RT/substituto (a tela confere o RT pelo contexto).
      if (item.id === "lab-qc-setup" || item.id === "lab-eqa") return qcOn && (isAdmin || isReleaser);
      if (item.id === "lab-inventory") return invOn && (isAdmin || isLabStaff || isReleaser);
      if (item.id === "lab-traceability") return invOn && (isAdmin || isReleaser);
      if (item.id === "lab-instruments") return (qcOn || invOn) && (isAdmin || isLabStaff || isReleaser);
```

No fim do arquivo:

```ts
// Módulo 21c: bloqueios ativos do CIQ no item da tela. O número vem da rota
// de contagem (sem trilha, Divergência C11).
export function withQcBlockCount(groups: NavGroupDef[], count: number): NavGroupDef[] {
  if (count <= 0) return groups;
  const shown = count === 1 ? "1 ativo" : `${count} ativos`;
  return groups.map((group) => ({
    ...group,
    items: group.items.map((item) => item.id === "lab-qc-blocks" ? { ...item, label: `${item.label} (${shown})` } : item)
  }));
}
```

`src/shell/catalog.ts` — no import, acrescente `IconAward, IconChartDots, IconClipboardCheck, IconDevices, IconLockExclamation, IconPackages, IconTargetArrow, IconTimeline` (mantendo a ordem alfabética do bloco), e em `screens`, depois de `"lab-release"`:

```ts
    "lab-qc-runs": { icon: IconClipboardCheck, description: "Valores de controle do dia por aparelho" },
    "lab-qc-chart": { icon: IconChartDots, description: "Levey-Jennings por lote, nível e analito" },
    "lab-qc-blocks": { icon: IconLockExclamation, description: "Bloqueios, ação corretiva, exceções e revisões" },
    "lab-qc-setup": { icon: IconTargetArrow, description: "Materiais, lotes, referências e regras" },
    "lab-eqa": { icon: IconAward, description: "Rodadas do controle externo e adequação" },
    "lab-inventory": { icon: IconPackages, description: "Itens, lotes, movimentos e lote em uso" },
    "lab-traceability": { icon: IconTimeline, description: "Rastreio por lote e consumo estimado" },
    "lab-instruments": { icon: IconDevices, description: "Aparelhos, setor e identificação na HL7" },
```

E a descrição do grupo:

```ts
    "Laboratório": { icon: IconFlask, description: "Amostras, laudos, controle de qualidade e estoque" },
```

`src/lib/qc.ts` — acrescente ao import `canUseLab` de `./lab` e `hasFeature` de `./features`, e:

```ts
export function canUseQc(user: { operator: boolean; features?: unknown } | null): boolean {
  return canUseLab(user) && hasFeature(user, "lab_quality_control");
}
export function canUseInventory(user: { operator: boolean; features?: unknown } | null): boolean {
  return canUseLab(user) && hasFeature(user, "lab_inventory");
}
```

`src/hooks/useQcBlockCount.ts`:

```ts
// Contagem de bloqueios ativos do CIQ (módulo 21c; Divergência C11). Só a
// rota /count: a lista de bloqueios traz dado de corrida e a de revisões,
// número de requisição — o contador nunca as lê. Uma requisição por minuto;
// nada fica no cache quando ninguém observa (gcTime 0).
import { useQuery } from "@tanstack/react-query";
import { countActiveQcBlocks } from "../lib/api";
import { QC_BLOCK_COUNT_KEY } from "../lib/qc";

export const QC_BLOCK_REFRESH_MS = 60_000;

export function useQcBlockCount(enabled: boolean, userId: string | null) {
  return useQuery({
    queryKey: [ QC_BLOCK_COUNT_KEY, userId ],
    queryFn: () => countActiveQcBlocks(),
    enabled,
    refetchInterval: enabled ? QC_BLOCK_REFRESH_MS : false,
    gcTime: 0, staleTime: 0, retry: false
  });
}
```

`src/App.tsx` — imports:

```ts
import { NAV_GROUPS, labelFor, isExamProvider, navGroupsFor, withExamPendingCount, withLabCriticalCount, withPendingCount, withQcBlockCount, type ModuleId, type NavGroupDef } from "./shell/modules";
import { useQcBlockCount } from "./hooks/useQcBlockCount";
import { canUseQc } from "./lib/qc";
```

`shellGroups`:

```ts
export function shellGroups(groups: NavGroupDef[], pending: number, examPending = 0, labCritical = 0, qcBlocks = 0): ShellGroup[] {
  return withQcBlockCount(withLabCriticalCount(withExamPendingCount(withPendingCount(groups, pending), examPending), labCritical), qcBlocks).map((g) => ({
    label: g.label,
    slug: slugOf(g.label),
    items: g.items.map((i) => {
      if (i.id === "signature-pending" && pending > 0) return { id: i.id, label: i.label, count: pending };
      if (i.id === "exam-acknowledgements" && examPending > 0) return { id: i.id, label: i.label, count: examPending };
      if (i.id === "lab-release" && labCritical > 0) return { id: i.id, label: i.label, count: labCritical };
      // Módulo 21c: bloqueios ativos acendem a tela de bloqueios.
      if (i.id === "lab-qc-blocks" && qcBlocks > 0) return { id: i.id, label: i.label, count: qcBlocks };
      return { id: i.id, label: i.label };
    })
  }));
}
```

Em `ShellInner`, depois de `const labCritical = labCriticalQuery.data ?? 0;`:

```ts
  const qcRoles = labRoles(auth.user);
  const qcBlockQuery = useQcBlockCount(canUseQc(auth.user) && (qcRoles.staff || qcRoles.releaser), auth.user?.id ?? null);
  const qcBlocks = qcBlockQuery.data ?? 0;
```

e o `useMemo` dos grupos:

```ts
  const groups = useMemo(
    () => shellGroups(navGroupsFor(auth.user), pending.data?.length ?? 0, examCount, labCritical, qcBlocks),
    [ auth.user, pending.data?.length, examCount, labCritical, qcBlocks ]
  );
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/shell src/hooks/useQcBlockCount.test.tsx src/App.navigation.test.tsx && npx tsc --noEmit`
Expected: PASS (inclusive `catalog.test.ts`, que cobra ícone e descrição das oito telas, e `modules.test.ts`, cujas contagens de grupos não mudam). As telas novas ainda caem no `Placeholder` do `renderModule` até as Tasks 6–16.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/catalog.ts src/shell/modules.qc.test.ts src/shell/modules.lab.test.ts src/App.tsx src/App.navigation.test.tsx src/lib/qc.ts src/hooks/useQcBlockCount.ts src/hooks/useQcBlockCount.test.tsx
$G commit -m "feat: add the lab quality and inventory screens to the menu with the active block count

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Portão por interruptor, escolha do laboratório, download e marca do lote

**Files:**
- Modify: `src/modules/lab/LabGate.tsx`, `src/modules/LabReports.tsx` (usa o `saveFile` extraído)
- Create: `src/modules/lab/LabPick.tsx`, `src/modules/lab/saveFile.ts`, `src/modules/inventory/ReagentLotTag.tsx`
- Test: `src/modules/lab/LabGate.qc.test.tsx`

**Interfaces:**
- Consumes: `useLabContext`, `labError`, `LAB_DISABLED` (21b); `hasFeature`, `FeatureKey`; `reagentLotMark` (Task 3); `qcUser`, `qcContext` (Task 1).
- Produces: `LabGate` ganha `requires?: FeatureKey[]` (basta um ligado) e `disabledText?: string`; `LabPick({ children(lab: LabChoice): ReactNode })` com `LabChoice = { id: string; name: string; lead: boolean }`; `saveFile(file: SignatureFile, name: string): void`; `ReagentLotTag({ row })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/lab/LabGate.qc.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn() };
});

import * as api from "../../lib/api";
import { LabGate } from "./LabGate";
import { LabPick } from "./LabPick";
import { saveFile } from "./saveFile";
import { ReagentLotTag } from "../inventory/ReagentLotTag";
import { QC_DISABLED } from "../../lib/qc";
import { renderWithProviders } from "../../test/campaignFixtures";
import { qcContext, qcUser } from "../../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => { mocked(api.fetchCurrentSession).mockReset(); mocked(api.getLabContext).mockReset(); });

const gate = (requires: ("lab_quality_control" | "lab_inventory")[]) => renderWithProviders(
  <LabGate title="CIQ" sub="x" allow={(r) => r.staff} requires={requires} disabledText={QC_DISABLED}>{() => <p>conteúdo</p>}</LabGate>
);

describe("LabGate com interruptor do 21c", () => {
  it("interruptor desligado: a frase do 21c e nenhuma leitura", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab" ] }));
    gate([ "lab_quality_control" ]);
    expect(await screen.findByText(QC_DISABLED)).not.toBeNull();
    expect(api.getLabContext).not.toHaveBeenCalled();
  });

  it("basta um dos interruptores pedidos", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab", "lab_inventory" ] }));
    mocked(api.getLabContext).mockResolvedValue(qcContext());
    gate([ "lab_quality_control", "lab_inventory" ]);
    expect(await screen.findByText("conteúdo")).not.toBeNull();
  });

  it("sem municipal_lab a frase é a do laboratório, mesmo com o 21c ligado", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ], { features: [ "exam_tracking", "lab_quality_control" ] }));
    gate([ "lab_quality_control" ]);
    expect(await screen.findByText("o laboratório municipal está desligado nesta cidade")).not.toBeNull();
  });
});

describe("LabPick", () => {
  it("dois laboratórios: escolhe; RT vem do contexto", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
    mocked(api.getLabContext).mockResolvedValue(qcContext({
      labs: [ { id: "pv9", name: "Laboratório Municipal" }, { id: "pv8", name: "Laboratório Distrital" } ], technical_lead_of: [ "pv8" ]
    }));
    renderWithProviders(<LabPick>{(lab) => <p>{`${lab.name} · ${lab.lead ? "RT" : "equipe"}`}</p>}</LabPick>);
    expect(await screen.findByText("Laboratório Municipal · equipe")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Laboratório"), { target: { value: "pv8" } });
    expect(screen.getByText("Laboratório Distrital · RT")).not.toBeNull();
  });

  it("contexto sem technical_lead_of (api antigo): ninguém é RT", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
    mocked(api.getLabContext).mockResolvedValue({ labs: [ { id: "pv9", name: "Laboratório Municipal" } ], releaser_of: [ "pv9" ], council: "CRF" });
    renderWithProviders(<LabPick>{(lab) => <p>{lab.lead ? "RT" : "equipe"}</p>}</LabPick>);
    expect(await screen.findByText("equipe")).not.toBeNull();
    expect(screen.queryByLabelText("Laboratório")).toBeNull();
  });
});

describe("ReagentLotTag", () => {
  // Estoque desligado (nada aparece) está coberto pela regra pura `reagentLotMark` (Task 3).
  it("lote e sem rastreio com o estoque ligado", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
    renderWithProviders(<p><ReagentLotTag row={{ reagent_lot: { id: "sl1", lot_number: "R-778" } }} /> · <ReagentLotTag row={{ reagent_lot: null }} /></p>);
    expect(await screen.findByText("lote R-778")).not.toBeNull();
    expect(await screen.findByText("sem rastreio de reagente")).not.toBeNull();
  });
});

describe("saveFile", () => {
  it("baixa pelo link temporário e devolve o objeto", () => {
    vi.useFakeTimers();
    const create = vi.fn(() => "blob:x");
    const revoke = vi.fn();
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: create });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: revoke });
    const click = vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(() => undefined);
    saveFile({ blob: new Blob([ "a,b" ]), filename: null }, "rastreio-R-778.csv");
    expect(click).toHaveBeenCalled();
    vi.advanceTimersByTime(60_000);
    expect(revoke).toHaveBeenCalledWith("blob:x");
    vi.useRealTimers();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/lab/LabGate.qc.test.tsx`
Expected: FAIL — `./LabPick`, `./saveFile` e `../inventory/ReagentLotTag` não existem; `LabGate` não conhece `requires`.

- [ ] **Step 3: Write minimal implementation**

`src/modules/lab/LabGate.tsx` — imports e a assinatura:

```tsx
import { hasFeature, type FeatureKey } from "../../lib/features";
```

```tsx
// `requires` (21c): interruptores além de `municipal_lab` — basta um ligado;
// `disabledText` é a frase quando nenhum está.
export function LabGate({ title, sub, allow, needsLab = true, requires, disabledText, children }: {
  title: string; sub: string; allow(roles: LabRoles): boolean; needsLab?: boolean; requires?: FeatureKey[]; disabledText?: string;
  children(roles: LabRoles): ReactNode;
}) {
  const { user } = useAuth();
  const roles = labRoles(user);
  const labOn = canUseLab(user);
  const extraOn = !requires || requires.some((key) => hasFeature(user, key));
  const switchOn = labOn && extraOn;
  const permitted = switchOn && allow(roles);
  const context = useLabContext(permitted && needsLab);
  if (!user) return null;
  let body: ReactNode;
  if (!labOn) body = <EmptyState title={LAB_DISABLED} />;
  else if (!extraOn) body = <EmptyState title={disabledText ?? LAB_DISABLED} />;
  else if (!permitted) body = <EmptyState title={LAB_NO_ROLE} />;
```

(o resto do corpo fica como está).

`src/modules/lab/LabPick.tsx`:

```tsx
// Escolha do laboratório nas telas do 21c (todas as pessoas do laboratório e
// o admin leem o mesmo contexto — o api dá ao admin todos os laboratórios
// ativos) e se a sessão é RT ou substituto dele (Divergência C1). Lista vazia
// já é barrada pelo LabGate (B32). Trocar de laboratório remonta o conteúdo.
import { Fragment, useState, type ReactNode } from "react";
import { labError } from "../../lib/lab";
import { inputStyle } from "../../components/formStyles";
import { useLabContext } from "./useLabContext";
import { alert, fieldLabel, muted, stack } from "../exams/styles";

export interface LabChoice { id: string; name: string; lead: boolean }

export function LabPick({ children }: { children(lab: LabChoice): ReactNode }) {
  const context = useLabContext(true);
  const [ providerId, setProviderId ] = useState("");
  if (context.isPending) return <p className="mono" style={muted}>carregando…</p>;
  if (context.isError && !context.data) return <p role="alert" style={alert}>{labError(context.error)}</p>;
  const labs = context.data!.labs;
  const lab = labs.find((l) => l.id === providerId) ?? labs[0];
  if (!lab) return null;
  const lead = (context.data!.technical_lead_of ?? []).includes(lab.id);
  return (
    <div style={stack}>
      {labs.length > 1 && (
        <label style={fieldLabel}>
          Laboratório
          <select value={lab.id} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
            {labs.map((l) => <option key={l.id} value={l.id}>{l.name}</option>)}
          </select>
        </label>
      )}
      <Fragment key={lab.id}>{children({ id: lab.id, name: lab.name, lead })}</Fragment>
    </div>
  );
}
```

`src/modules/lab/saveFile.ts` (extraído de `LabReports.tsx`, sem mudar o comportamento):

```ts
// Baixa um arquivo lido com fetch (CSV, PDF) por um link temporário; o objeto
// é revogado depois de um minuto. Nome sem dado pessoal: quem chama monta.
import type { SignatureFile } from "../../lib/api";

const BLOB_TTL_MS = 60_000;

export function saveFile(file: SignatureFile, name: string): void {
  const url = URL.createObjectURL(file.blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = name;
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(url), BLOB_TTL_MS);
}
```

Em `src/modules/LabReports.tsx`, apague `const BLOB_TTL_MS = 60_000;` e a função `saveFile` local, tire `type SignatureFile` do import de `../lib/api` e acrescente:

```ts
import { saveFile } from "./lab/saveFile";
```

`src/modules/inventory/ReagentLotTag.tsx`:

```tsx
// Lote de reagente gravado com o resultado ou o valor de controle (contrato
// §7): número do lote, ou "sem rastreio de reagente" quando o estoque está
// ligado e não havia lote em uso no aparelho e exame. Ausente = nada.
import type { ReagentLotRef } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { hasFeature } from "../../lib/features";
import { reagentLotMark } from "../../lib/inventory";
import { Tag } from "../../components/Tag";

export function ReagentLotTag({ row }: { row: { reagent_lot?: ReagentLotRef | null } }) {
  const { user } = useAuth();
  const mark = reagentLotMark(row, hasFeature(user, "lab_inventory"));
  if (!mark) return null;
  return <Tag tone={mark.untraced ? "warn" : "neutral"}>{mark.text}</Tag>;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/lab src/modules/LabReports.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes do `LabGate` e do `LabReports` do 21b continuam verdes).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/lab/LabGate.tsx src/modules/lab/LabPick.tsx src/modules/lab/saveFile.ts src/modules/LabReports.tsx src/modules/inventory/ReagentLotTag.tsx src/modules/lab/LabGate.qc.test.tsx
$G commit -m "feat: gate lab screens by the quality and inventory switches and pick the lab with the technical lead flag

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Aparelhos e o aparelho da versão do exame

**Files:**
- Create: `src/modules/LabInstruments.tsx`
- Modify: `src/lib/labAdmin.ts` (`VersionDraft`, `versionDraft`, `versionBody`), `src/modules/labAdmin/VersionForm.tsx`, `src/modules/labAdmin/LabExamsPanel.tsx`, `src/App.tsx`
- Test: `src/modules/LabInstruments.test.tsx`, `src/modules/labAdmin/VersionForm.instrument.test.tsx`

**Interfaces:**
- Consumes: `listLabInstruments`, `listAdminInstruments`, `createInstrument`, `updateInstrument` (Task 1); `EMPTY_INSTRUMENT`, `instrumentDraft`, `instrumentProblem`, `instrumentBody`, `INSTRUMENTS_DISABLED`, `QC_INSTRUMENTS_KEY`, `qcError` (Task 2); `LabGate` com `requires`, `LabPick` (Task 5); `SECTOR_LABEL`, `sectorLabel` (21b).
- Produces: `LabInstruments({ onGoToSecurity })`; `VersionDraft.instrumentId: string`; `versionBody(d, withInstrument = false)` (com `true`, o corpo leva `lab_instrument_id`); `VersionForm` ganha `instruments?: LabInstrument[]` (sem a prop, a tela e o corpo são os do 21b).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabInstruments.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabInstruments: vi.fn(), listAdminInstruments: vi.fn(),
    createInstrument: vi.fn(), updateInstrument: vi.fn() };
});

import * as api from "../lib/api";
import { LabInstruments } from "./LabInstruments";
import { INSTRUMENTS_DISABLED } from "../lib/qc";
import { renderWithProviders } from "../test/campaignFixtures";
import { instrument, qcContext, qcUser } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listLabInstruments, api.listAdminInstruments, api.createInstrument, api.updateInstrument ];
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "municipal_admin" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext());
  mocked(api.listAdminInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.createInstrument).mockResolvedValue(instrument({ id: "in2", name: "Yumizen H550" }));
});

async function openNew() {
  renderWithProviders(<LabInstruments />);
  fireEvent.click(await screen.findByRole("button", { name: "Novo aparelho" }));
  return screen.getByRole("region", { name: "Novo aparelho" });
}

describe("LabInstruments", () => {
  it("admin cadastra com step-up; a lista relê", async () => {
    const form = await openNew();
    fireEvent.change(within(form).getByLabelText("Nome"), { target: { value: "Yumizen H550" } });
    fireEvent.change(within(form).getByLabelText("Fabricante"), { target: { value: "Horiba" } });
    fireEvent.change(within(form).getByLabelText("Setor"), { target: { value: "hematology" } });
    fireEvent.change(within(form).getByLabelText("Identificador na HL7 (EQU-1)"), { target: { value: "H550-01" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar aparelho" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar aparelho" }));
    await waitFor(() => expect(api.createInstrument).toHaveBeenCalledWith({
      provider_id: "pv9", name: "Yumizen H550", manufacturer: "Horiba", model: "", serial_number: "", sector: "hematology", hl7_equipment_id: "H550-01", active: true
    }));
    expect(await screen.findByText("Aparelho salvo.")).not.toBeNull();
    await waitFor(() => expect(api.listAdminInstruments).toHaveBeenCalledTimes(2));
  });

  it("identificador HL7 com espaço não abre a confirmação", async () => {
    const form = await openNew();
    fireEvent.change(within(form).getByLabelText("Nome"), { target: { value: "Yumizen" } });
    fireEvent.change(within(form).getByLabelText("Setor"), { target: { value: "hematology" } });
    fireEvent.change(within(form).getByLabelText("Identificador na HL7 (EQU-1)"), { target: { value: "H 550" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar aparelho" }));
    expect(within(form).getByRole("alert").textContent).toBe("identificador HL7 com até 40 letras, números, ponto, hífen ou _");
    expect(screen.queryByRole("button", { name: "Salvar aparelho" })).toBeNull();
  });

  it("422 duplicate_hl7_equipment_id vira frase", async () => {
    mocked(api.createInstrument).mockRejectedValueOnce(new api.ApiError(422, { error: "duplicate_hl7_equipment_id" }, "x"));
    const form = await openNew();
    fireEvent.change(within(form).getByLabelText("Nome"), { target: { value: "Yumizen" } });
    fireEvent.change(within(form).getByLabelText("Setor"), { target: { value: "hematology" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar aparelho" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar aparelho" }));
    expect(await screen.findByText("este identificador HL7 já está em outro aparelho")).not.toBeNull();
  });

  it("equipe do laboratório só lê (rota de membros), sem botão de cadastro", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
    renderWithProviders(<LabInstruments />);
    expect(await screen.findByText("BS-200")).not.toBeNull();
    expect(api.listLabInstruments).toHaveBeenCalledWith("pv9");
    expect(api.listAdminInstruments).not.toHaveBeenCalled();
    expect(screen.queryByRole("button", { name: "Novo aparelho" })).toBeNull();
  });

  it("com só o estoque ligado a tela abre; com os dois desligados, a frase", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab", "lab_inventory" ] }));
    const { unmount } = renderWithProviders(<LabInstruments />);
    expect(await screen.findByText("BS-200")).not.toBeNull();
    unmount();
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab" ] }));
    renderWithProviders(<LabInstruments />);
    expect(await screen.findByText(INSTRUMENTS_DISABLED)).not.toBeNull();
  });
});
```

```tsx
// src/modules/labAdmin/VersionForm.instrument.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), createLabExamVersion: vi.fn() };
});

import * as api from "../../lib/api";
import { VersionForm } from "./VersionForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { labExam, labVersion } from "../../test/labFixtures";
import { instrument, qcUser } from "../../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => {
  mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(qcUser([ "municipal_admin" ]));
  mocked(api.createLabExamVersion).mockReset().mockResolvedValue(labVersion({ id: "ev2", version: 2 }));
});

describe("VersionForm com aparelho (contrato §2)", () => {
  it("com aparelhos: escolhe e o corpo leva lab_instrument_id", async () => {
    renderWithProviders(<VersionForm exam={labExam()} instruments={[ instrument(), instrument({ id: "in2", name: "Yumizen H550", active: false }) ]} onSaved={vi.fn()} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Nova versão de Glicose (atual v1)" });
    const select = within(form).getByLabelText("Aparelho") as HTMLSelectElement;
    expect([ ...select.options ].map((o) => o.textContent)).toEqual([ "nenhum", "BS-200" ]);
    fireEvent.change(select, { target: { value: "in1" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar versão" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar versão 2" }));
    await waitFor(() => expect(api.createLabExamVersion).toHaveBeenCalled());
    expect(mocked(api.createLabExamVersion).mock.calls[0][1].lab_instrument_id).toBe("in1");
  });

  it("sem a prop: nenhum campo novo e o corpo do 21b", async () => {
    renderWithProviders(<VersionForm exam={labExam()} onSaved={vi.fn()} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Nova versão de Glicose (atual v1)" });
    expect(within(form).queryByLabelText("Aparelho")).toBeNull();
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar versão" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar versão 2" }));
    await waitFor(() => expect(api.createLabExamVersion).toHaveBeenCalled());
    expect(mocked(api.createLabExamVersion).mock.calls[0][1]).not.toHaveProperty("lab_instrument_id");
  });

  it("a versão atual com aparelho vem escolhida", async () => {
    const exam = labExam({ current_version: labVersion({ lab_instrument_id: "in1" }) });
    renderWithProviders(<VersionForm exam={exam} instruments={[ instrument() ]} onSaved={vi.fn()} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Nova versão de Glicose (atual v1)" });
    expect((within(form).getByLabelText("Aparelho") as HTMLSelectElement).value).toBe("in1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabInstruments.test.tsx src/modules/labAdmin/VersionForm.instrument.test.tsx`
Expected: FAIL — `./LabInstruments` não existe; `VersionForm` não mostra "Aparelho".

- [ ] **Step 3: Write minimal implementation**

`src/modules/LabInstruments.tsx`:

```tsx
// src/modules/LabInstruments.tsx
// Aparelhos do laboratório (contrato §2; spec §4 "Aparelhos"): o admin
// cadastra (step-up: /lab/admin); a equipe e os liberadores leem pela rota de
// membros. Aberta com o CIQ ou com o estoque ligado (Divergência C13).
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createInstrument, listAdminInstruments, listLabInstruments, updateInstrument, type LabInstrument } from "../lib/api";
import { SECTOR_LABEL, sectorLabel } from "../lib/lab";
import {
  EMPTY_INSTRUMENT, INSTRUMENTS_DISABLED, QC_INSTRUMENTS_KEY, instrumentBody, instrumentDraft, instrumentProblem, qcError, type InstrumentDraft
} from "../lib/qc";
import { Panel } from "../components/Panel";
import { DataTable, type Column } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { SensitiveAction } from "../components/SensitiveAction";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { alert, box, checkLabel, fieldLabel, form, okText, row, stack } from "./exams/styles";

export function LabInstruments({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  return (
    <LabGate title="Aparelhos" sub="laboratório municipal · aparelhos, setor e identificação na HL7"
      allow={(r) => r.admin || r.staff || r.releaser} requires={[ "lab_quality_control", "lab_inventory" ]} disabledText={INSTRUMENTS_DISABLED}>
      {(roles) => <LabPick>{(lab) => <InstrumentsBody providerId={lab.id} admin={roles.admin} onGoToSecurity={onGoToSecurity} />}</LabPick>}
    </LabGate>
  );
}

function InstrumentsBody({ providerId, admin, onGoToSecurity }: { providerId: string; admin: boolean; onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const list = useQuery({
    queryKey: [ QC_INSTRUMENTS_KEY, providerId, admin ? "admin" : "lab" ],
    queryFn: () => (admin ? listAdminInstruments(providerId) : listLabInstruments(providerId)), retry: false
  });
  const [ editing, setEditing ] = useState<LabInstrument | "new" | null>(null);
  const [ draft, setDraft ] = useState<InstrumentDraft>(EMPTY_INSTRUMENT);
  const [ confirming, setConfirming ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const set = (patch: Partial<InstrumentDraft>) => setDraft((d) => ({ ...d, ...patch }));

  function open(target: LabInstrument | "new") {
    setEditing(target);
    setDraft(target === "new" ? EMPTY_INSTRUMENT : instrumentDraft(target));
    setConfirming(false); setError(null); setNotice(null);
  }

  function review() {
    const problem = instrumentProblem(draft);
    if (problem) { setError(problem); return; }
    setError(null);
    setConfirming(true);
  }

  const text = (label: string, value: string, onChange: (v: string) => void) => (
    <label style={fieldLabel}>{label}<input value={value} style={inputStyle} onChange={(e) => onChange(e.target.value)} /></label>
  );

  const cols: Column<LabInstrument>[] = [
    { label: "Aparelho", w: "1.4fr", card: "title", render: (i) => (i.model && i.model !== i.name ? `${i.name} · ${i.model}` : i.name) },
    { label: "Setor", w: "1fr", render: (i) => sectorLabel(i.sector) },
    { label: "Fabricante · série", w: "1.4fr", render: (i) => [ i.manufacturer, i.serial_number ].filter(Boolean).join(" · ") || "—" },
    { label: "HL7", w: "1fr", render: (i) => (i.hl7_equipment_id ? <span className="mono">{i.hl7_equipment_id}</span> : "—") },
    { label: "Situação", w: "0.8fr", render: (i) => <Tag tone={i.active ? "ok" : undefined}>{i.active ? "ativo" : "inativo"}</Tag> }
  ];
  if (admin) {
    cols.push({ label: "", w: "auto", card: "actions", render: (i) => <button type="button" style={secondaryButtonStyle} onClick={() => open(i)}>Editar</button> });
  }

  return (
    <Panel title="Aparelhos do laboratório" sub="o identificador HL7 liga o aparelho às mensagens do middleware (EQU-1 ou OBX-18)"
      right={admin ? <button type="button" style={secondaryButtonStyle} onClick={() => open("new")}>Novo aparelho</button> : undefined}>
      <div style={stack}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {list.isError ? <p role="alert" style={alert}>{qcError(list.error)}</p> : (
          <DataTable<LabInstrument> cols={cols} rows={list.data ?? []} rowKey={(i) => i.id} empty={list.isPending ? "carregando…" : "nenhum aparelho cadastrado"} />
        )}
        {admin && editing && (
          <section aria-label={editing === "new" ? "Novo aparelho" : `Editar ${editing.name}`} style={box}>
            <fieldset disabled={confirming} style={{ ...form, border: 0, padding: 0, margin: 0, minWidth: 0 }}>
              {text("Nome", draft.name, (v) => set({ name: v }))}
              {text("Fabricante", draft.manufacturer, (v) => set({ manufacturer: v }))}
              {text("Modelo", draft.model, (v) => set({ model: v }))}
              {text("Número de série", draft.serialNumber, (v) => set({ serialNumber: v }))}
              <label style={fieldLabel}>Setor
                <select value={draft.sector} style={inputStyle} onChange={(e) => set({ sector: e.target.value })}>
                  <option value="">—</option>
                  {Object.entries(SECTOR_LABEL).map(([ k, v ]) => <option key={k} value={k}>{v}</option>)}
                </select>
              </label>
              {text("Identificador na HL7 (EQU-1)", draft.hl7EquipmentId, (v) => set({ hl7EquipmentId: v }))}
              <label style={checkLabel}><input type="checkbox" checked={draft.active} onChange={(e) => set({ active: e.target.checked })} />Ativo</label>
            </fieldset>
            {error && <p role="alert" style={alert}>{error}</p>}
            {!confirming && (
              <div style={row}>
                <button type="button" style={buttonStyle} onClick={review}>Revisar e salvar aparelho</button>
                <button type="button" style={secondaryButtonStyle} onClick={() => setEditing(null)}>Cancelar</button>
              </div>
            )}
            {confirming && (
              <SensitiveAction title="Salvar aparelho" description="O aparelho fica disponível para o CIQ, o lote em uso e a identificação das mensagens HL7."
                requiresStepUp confirmLabel="Salvar aparelho"
                run={async () => {
                  const body = instrumentBody(draft, providerId);
                  if (editing === "new") await createInstrument(body); else await updateInstrument(editing.id, body);
                }}
                onDone={() => { setConfirming(false); setEditing(null); setNotice("Aparelho salvo."); void queryClient.invalidateQueries({ queryKey: [ QC_INSTRUMENTS_KEY ] }); }}
                onCancel={() => setConfirming(false)} onGoToSecurity={onGoToSecurity} translateError={(err) => qcError(err)} />
            )}
          </section>
        )}
      </div>
    </Panel>
  );
}
```

`src/lib/labAdmin.ts`:

```ts
export interface VersionDraft { method: string; material: string; container: string; instrumentTestCode: string; instrumentId: string; analytes: AnalyteDraft[] }
```

Em `versionDraft`, depois de `instrumentTestCode: v?.instrument_test_code ?? "",`:

```ts
    instrumentId: v?.lab_instrument_id ?? "",
```

E `versionBody`:

```ts
// `withInstrument` (21c): só quando a tela ofereceu o aparelho — sem ele o corpo é o do 21b.
export function versionBody(d: VersionDraft, withInstrument = false): LabVersionInput {
  return {
    method: d.method.trim(), material: d.material.trim(), container: d.container.trim(),
    instrument_test_code: d.instrumentTestCode.trim() || null,
    ...(withInstrument ? { lab_instrument_id: d.instrumentId || null } : {}),
    analytes: d.analytes.map((a) => ({
```

(o resto da função fica igual).

`src/modules/labAdmin/VersionForm.tsx` — import e assinatura:

```tsx
import { createLabExamVersion, type LabExam, type LabInstrument, type LabValueType } from "../../lib/api";
```

```tsx
// `instruments` (21c, contrato §2): presente quando o CIQ ou o estoque estão ligados; só os ativos são oferecidos.
export function VersionForm({ exam, instruments, onSaved, onCancel, onGoToSecurity }: {
  exam: LabExam; instruments?: LabInstrument[]; onSaved(): void; onCancel(): void; onGoToSecurity?(): void;
}) {
```

Logo depois do campo "Código do exame no aparelho (HL7)":

```tsx
        {instruments && (
          <label style={fieldLabel}>Aparelho
            <select value={draft.instrumentId} style={inputStyle} onChange={(e) => set({ instrumentId: e.target.value })}>
              <option value="">nenhum</option>
              {instruments.filter((i) => i.active || i.id === draft.instrumentId).map((i) => <option key={i.id} value={i.id}>{i.name}</option>)}
            </select>
          </label>
        )}
```

e no `run` da confirmação:

```tsx
          run={async () => { await createLabExamVersion(exam.id, versionBody(draft, instruments !== undefined)); }}
```

`src/modules/labAdmin/LabExamsPanel.tsx` — imports:

```tsx
import { adoptLabExam, getLabAdminExam, listAdminInstruments, listLabAdminExams, listLabTemplates, updateLabExam, type LabExam, type LabTemplate } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { hasFeature } from "../../lib/features";
import { QC_INSTRUMENTS_KEY } from "../../lib/qc";
```

no começo de `LabExamsPanel`:

```tsx
  const { user } = useAuth();
  // 21c: aparelho da versão só com o CIQ ou o estoque ligados (Divergência C13).
  const instrumentsOn = hasFeature(user, "lab_quality_control") || hasFeature(user, "lab_inventory");
  const instruments = useQuery({
    queryKey: [ QC_INSTRUMENTS_KEY, providerId, "admin" ], queryFn: () => listAdminInstruments(providerId), enabled: instrumentsOn, retry: false
  });
```

e o `VersionForm`:

```tsx
            <VersionForm key={versioning.id} exam={versioning} instruments={instrumentsOn ? instruments.data ?? [] : undefined}
              onGoToSecurity={onGoToSecurity} onCancel={() => setVersioning(null)}
              onSaved={() => { setVersioning(null); setNotice("Versão nova criada."); refresh(); }} />
```

`src/App.tsx`:

```tsx
import { LabInstruments } from "./modules/LabInstruments";
```

```tsx
    case "lab-instruments": return <LabInstruments onGoToSecurity={() => setActive("security")} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabInstruments.test.tsx src/modules/labAdmin src/lib/labAdmin.test.ts src/modules/LabAdmin.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes do admin do 21b continuam verdes: sem o interruptor do 21c, `instruments` é `undefined` e o corpo não muda).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabInstruments.tsx src/modules/LabInstruments.test.tsx src/lib/labAdmin.ts src/modules/labAdmin/VersionForm.tsx src/modules/labAdmin/LabExamsPanel.tsx src/modules/labAdmin/VersionForm.instrument.test.tsx src/App.tsx
$G commit -m "feat: add the lab instruments screen and link exam versions to an instrument

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: CIQ — materiais e lotes com código de barras por nível

**Files:**
- Create: `src/modules/LabQcSetup.tsx`, `src/modules/qc/MaterialsPanel.tsx`, `src/modules/qc/LotsPanel.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabQcSetup.test.tsx`

**Interfaces:**
- Consumes: `listLabExams` (21b, com `id`/`analytes` — Divergência C5), `listQcMaterials`, `createQcMaterial`, `updateQcMaterial`, `listQcLots`, `createQcLot`, `closeQcLot` (Task 1); `EMPTY_MATERIAL`, `materialDraft`, `materialProblem`, `materialBody`, `analyteName`, `lotProblem`, `lotBody`, `lotStatusLabel`, `QC_DISABLED`, `QC_LEAD_ONLY`, `QC_EXAMS_KEY`, `QC_MATERIALS_KEY`, `QC_LOTS_KEY`, `qcError`, `refreshQc` (Task 2); `LabGate`, `LabPick` (Task 5); `todayInCity` (`lib/campaigns`); `fmtDay` (`lib/exams`).
- Produces: `LabQcSetup()`; `SetupBody({ providerId })` (interna; a Task 8 acrescenta os painéis de referência e regras); `MaterialsPanel({ providerId, exams })`; `LotsPanel({ providerId, pickedId, onPick(lotId) })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabQcSetup.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabExams: vi.fn(), listQcMaterials: vi.fn(), createQcMaterial: vi.fn(),
    updateQcMaterial: vi.fn(), listQcLots: vi.fn(), createQcLot: vi.fn(), closeQcLot: vi.fn(), getQcTargets: vi.fn(), listQcSuggestions: vi.fn(),
    listLabInstruments: vi.fn(), getQcRules: vi.fn() };
});

import * as api from "../lib/api";
import { LabQcSetup } from "./LabQcSetup";
import { QC_DISABLED, QC_LEAD_ONLY } from "../lib/qc";
import { renderWithProviders } from "../test/campaignFixtures";
import { CHOL_EXAM, GLU_EXAM, NOW21C, instrument, qcContext, qcLot, qcMaterial, qcRules, qcUser } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listLabExams, api.listQcMaterials, api.createQcMaterial, api.updateQcMaterial,
  api.listQcLots, api.createQcLot, api.closeQcLot, api.getQcTargets, api.listQcSuggestions, api.listLabInstruments, api.getQcRules ];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "municipal_admin" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext());
  mocked(api.listLabExams).mockResolvedValue([ GLU_EXAM, CHOL_EXAM ]);
  mocked(api.listQcMaterials).mockResolvedValue([ qcMaterial() ]);
  mocked(api.listQcLots).mockResolvedValue([ qcLot() ]);
  mocked(api.createQcMaterial).mockResolvedValue(qcMaterial({ id: "qm2" }));
  mocked(api.createQcLot).mockResolvedValue(qcLot({ id: "ql2" }));
  mocked(api.closeQcLot).mockResolvedValue(qcLot({ status: "closed", closed_on: "2026-10-18" }));
  mocked(api.getQcTargets).mockResolvedValue({ current: [], history: [] });
  mocked(api.listQcSuggestions).mockResolvedValue([]);
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.getQcRules).mockResolvedValue(qcRules());
});

describe("LabQcSetup — materiais e lotes", () => {
  it("material novo: níveis por ponto e vírgula e analitos dos exames do laboratório", async () => {
    renderWithProviders(<LabQcSetup />);
    const panel = await screen.findByRole("region", { name: "Materiais de controle" });
    fireEvent.click(within(panel).getByRole("button", { name: "Novo material" }));
    const form = within(panel).getByRole("region", { name: "Novo material" });
    fireEvent.change(within(form).getByLabelText("Nome"), { target: { value: "Controle lipídico" } });
    fireEvent.change(within(form).getByLabelText("Níveis (separados por ;)"), { target: { value: "normal; alto" } });
    fireEvent.click(within(form).getByLabelText("HDL (Colesterol total e frações)"));
    fireEvent.click(within(form).getByRole("button", { name: "Salvar material" }));
    await waitFor(() => expect(api.createQcMaterial).toHaveBeenCalledWith({
      provider_id: "pv9", name: "Controle lipídico", manufacturer: "", levels: [ "normal", "alto" ], analytes: [ { lab_exam_id: "le2", analyte_key: "hdl" } ], active: true
    }));
    expect(await within(panel).findByText("Material salvo.")).not.toBeNull();
  });

  it("lote novo: um código por nível; Enter do leitor não envia; 422 barcode_taken vira frase", async () => {
    mocked(api.createQcLot).mockRejectedValueOnce(new api.ApiError(422, { error: "barcode_taken" }, "x"));
    renderWithProviders(<LabQcSetup />);
    const panel = await screen.findByRole("region", { name: "Lotes de controle" });
    fireEvent.click(within(panel).getByRole("button", { name: "Novo lote" }));
    const form = within(panel).getByRole("region", { name: "Novo lote" });
    fireEvent.change(within(form).getByLabelText("Material"), { target: { value: "qm1" } });
    fireEvent.change(within(form).getByLabelText("Número do lote"), { target: { value: "L2026B" } });
    fireEvent.change(within(form).getByLabelText("Validade"), { target: { value: "2027-06-30" } });
    const normal = within(form).getByLabelText("Código de barras — normal");
    fireEvent.change(normal, { target: { value: "QC-L2026B-N" } });
    fireEvent.keyDown(normal, { key: "Enter" });
    expect(api.createQcLot).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("Código de barras — patológico"), { target: { value: "QC-L2026B-P" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar lote" }));
    await waitFor(() => expect(api.createQcLot).toHaveBeenCalledWith({
      material_id: "qm1", lot_number: "L2026B", expires_on: "2027-06-30", level_barcodes: { normal: "QC-L2026B-N", "patológico": "QC-L2026B-P" }
    }));
    expect((await within(form).findByRole("alert")).textContent).toBe("este código de barras já está em outro lote de controle ou numa amostra");
    expect((within(form).getByLabelText("Número do lote") as HTMLInputElement).value).toBe("L2026B");
  });

  it("lote vencido é recusado antes do envio", async () => {
    renderWithProviders(<LabQcSetup />);
    const panel = await screen.findByRole("region", { name: "Lotes de controle" });
    fireEvent.click(within(panel).getByRole("button", { name: "Novo lote" }));
    const form = within(panel).getByRole("region", { name: "Novo lote" });
    fireEvent.change(within(form).getByLabelText("Material"), { target: { value: "qm1" } });
    fireEvent.change(within(form).getByLabelText("Número do lote"), { target: { value: "L2025Z" } });
    fireEvent.change(within(form).getByLabelText("Validade"), { target: { value: "2026-10-17" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar lote" }));
    expect(within(form).getByRole("alert").textContent).toBe("lote vencido — confira a validade");
    expect(api.createQcLot).not.toHaveBeenCalled();
  });

  it("encerrar lote pede confirmação", async () => {
    renderWithProviders(<LabQcSetup />);
    const panel = await screen.findByRole("region", { name: "Lotes de controle" });
    fireEvent.click(await within(panel).findByRole("button", { name: "Encerrar" }));
    expect(api.closeQcLot).not.toHaveBeenCalled();
    fireEvent.click(within(panel).getByRole("button", { name: "Confirmar encerramento de L2026A" }));
    await waitFor(() => expect(api.closeQcLot).toHaveBeenCalledWith("ql1"));
  });

  it("liberador que não é RT: a frase e nenhuma leitura do CIQ; RT vê", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
    mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ] }));
    const { unmount } = renderWithProviders(<LabQcSetup />);
    expect(await screen.findByText(QC_LEAD_ONLY)).not.toBeNull();
    expect(api.listQcMaterials).not.toHaveBeenCalled();
    unmount();
    mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ], technical_lead_of: [ "pv9" ] }));
    renderWithProviders(<LabQcSetup />);
    expect(await screen.findByRole("region", { name: "Materiais de controle" })).not.toBeNull();
  });

  it("CIQ desligado: a frase", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "municipal_admin" ], { features: [ "exam_tracking", "municipal_lab", "lab_inventory" ] }));
    renderWithProviders(<LabQcSetup />);
    expect(await screen.findByText(QC_DISABLED)).not.toBeNull();
  });
});
```

(Os mocks de referência, regras e aparelhos já entram aqui porque a Task 8 acrescenta esses painéis à mesma tela.)

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabQcSetup.test.tsx`
Expected: FAIL — `./LabQcSetup` não existe.

- [ ] **Step 3: Write minimal implementation**

`src/modules/LabQcSetup.tsx`:

```tsx
// src/modules/LabQcSetup.tsx
// Cadastro do CIQ (spec §5.1; contrato §3): materiais, lotes com código de
// barras por nível, referências e regras. Do admin ou do RT/substituto
// (contexto, Divergência C1); o liberador que não é RT vê a frase e nada é
// lido. Sem polling.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { listLabExams } from "../lib/api";
import { QC_DISABLED, QC_EXAMS_KEY, QC_LEAD_ONLY } from "../lib/qc";
import { EmptyState } from "../components/EmptyState";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { MaterialsPanel } from "./qc/MaterialsPanel";
import { LotsPanel } from "./qc/LotsPanel";
import { stack } from "./exams/styles";

export function LabQcSetup() {
  return (
    <LabGate title="Controles e referências" sub="CIQ · materiais, lotes, referências e regras de Westgard"
      allow={(r) => r.admin || r.releaser} requires={[ "lab_quality_control" ]} disabledText={QC_DISABLED}>
      {(roles) => <LabPick>{(lab) => (roles.admin || lab.lead ? <SetupBody providerId={lab.id} /> : <EmptyState title={QC_LEAD_ONLY} />)}</LabPick>}
    </LabGate>
  );
}

function SetupBody({ providerId }: { providerId: string }) {
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const [ lotId, setLotId ] = useState<string | null>(null);
  return (
    <div style={stack}>
      <MaterialsPanel providerId={providerId} exams={exams.data ?? []} />
      <LotsPanel providerId={providerId} pickedId={lotId} onPick={setLotId} />
    </div>
  );
}
```

`src/modules/qc/MaterialsPanel.tsx`:

```tsx
// Materiais de controle (contrato §3): nome, fabricante, níveis e analitos
// (dos exames do laboratório). Sem step-up (o contrato não pede).
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createQcMaterial, listQcMaterials, updateQcMaterial, type LabExamRef, type QcMaterial } from "../../lib/api";
import {
  EMPTY_MATERIAL, QC_MATERIALS_KEY, analyteName, materialBody, materialDraft, materialProblem, qcError, refreshQc, type MaterialDraft
} from "../../lib/qc";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, checkLabel, fieldLabel, fieldset, form, legend, muted, okText, row, stack } from "../exams/styles";

export function MaterialsPanel({ providerId, exams }: { providerId: string; exams: LabExamRef[] }) {
  const queryClient = useQueryClient();
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const [ editing, setEditing ] = useState<QcMaterial | "new" | null>(null);
  const [ draft, setDraft ] = useState<MaterialDraft>(EMPTY_MATERIAL);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const set = (patch: Partial<MaterialDraft>) => setDraft((d) => ({ ...d, ...patch }));
  const options = exams.filter((e) => e.id).flatMap((e) => (e.analytes ?? []).map((a) => ({ token: `${e.id}:${a.key}`, label: `${a.name} (${e.name})` })));

  function open(target: QcMaterial | "new") {
    setEditing(target);
    setDraft(target === "new" ? EMPTY_MATERIAL : materialDraft(target));
    setError(null); setNotice(null);
  }

  async function save() {
    const problem = materialProblem(draft);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      const body = materialBody(draft, providerId);
      if (editing === "new" || editing === null) await createQcMaterial(body); else await updateQcMaterial(editing.id, body);
      setEditing(null);
      setNotice("Material salvo.");
      void refreshQc(queryClient).catch(() => undefined);
    } catch (err) {
      setError(qcError(err));
    }
  }

  function toggle(token: string, on: boolean) {
    setDraft((d) => ({ ...d, analytes: on ? [ ...d.analytes, token ] : d.analytes.filter((t) => t !== token) }));
  }

  return (
    <Panel title="Materiais de controle" sub="níveis e analitos de cada material"
      right={<button type="button" style={secondaryButtonStyle} onClick={() => open("new")}>Novo material</button>}>
      <div style={stack}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {materials.isError ? <p role="alert" style={alert}>{qcError(materials.error)}</p> : (
          <DataTable<QcMaterial>
            cols={[
              { label: "Material", w: "1.4fr", card: "title", render: (m) => (m.manufacturer ? `${m.name} · ${m.manufacturer}` : m.name) },
              { label: "Níveis", w: "1fr", render: (m) => m.levels.join(", ") },
              { label: "Analitos", w: "1.6fr", render: (m) => m.analytes.map((a) => analyteName(exams, a.lab_exam_id, a.analyte_key)).join(", ") },
              { label: "Situação", w: "0.7fr", render: (m) => <Tag tone={m.active ? "ok" : undefined}>{m.active ? "ativo" : "inativo"}</Tag> },
              { label: "", w: "auto", card: "actions", render: (m) => <button type="button" style={secondaryButtonStyle} onClick={() => open(m)}>Editar</button> }
            ]}
            rows={materials.data ?? []} rowKey={(m) => m.id} empty={materials.isPending ? "carregando…" : "nenhum material cadastrado"}
          />
        )}
        {editing && (
          <section aria-label={editing === "new" ? "Novo material" : `Editar ${editing.name}`} style={box}>
            <div style={form}>
              <label style={fieldLabel}>Nome<input value={draft.name} style={inputStyle} onChange={(e) => set({ name: e.target.value })} /></label>
              <label style={fieldLabel}>Fabricante<input value={draft.manufacturer} style={inputStyle} onChange={(e) => set({ manufacturer: e.target.value })} /></label>
              <label style={fieldLabel}>Níveis (separados por ;)<input value={draft.levels} placeholder="normal; patológico" style={inputStyle} onChange={(e) => set({ levels: e.target.value })} /></label>
              <fieldset style={fieldset}>
                <legend style={legend}>Analitos</legend>
                {options.length === 0 && <p style={muted}>nenhum exame com componentes no laboratório — adote os exames em Exames e liberadores do laboratório</p>}
                {options.map((o) => (
                  <label key={o.token} style={checkLabel}>
                    <input type="checkbox" checked={draft.analytes.includes(o.token)} onChange={(e) => toggle(o.token, e.target.checked)} />{o.label}
                  </label>
                ))}
              </fieldset>
              <label style={checkLabel}><input type="checkbox" checked={draft.active} onChange={(e) => set({ active: e.target.checked })} />Ativo</label>
            </div>
            {error && <p role="alert" style={alert}>{error}</p>}
            <div style={row}>
              <ActionButton onClick={save}>Salvar material</ActionButton>
              <button type="button" style={secondaryButtonStyle} onClick={() => setEditing(null)}>Cancelar</button>
            </div>
          </section>
        )}
      </div>
    </Panel>
  );
}
```

`src/modules/qc/LotsPanel.tsx`:

```tsx
// Lotes de controle (contrato §3): número, validade e um código de barras por
// nível (únicos na cidade e distintos dos de amostra — o api confere,
// `barcode_taken`). O leitor é um teclado: Enter no campo do código não envia
// o formulário. Encerrar pede confirmação.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { closeQcLot, createQcLot, listQcLots, listQcMaterials, type QcLot } from "../../lib/api";
import { QC_LOTS_KEY, QC_MATERIALS_KEY, lotBody, lotProblem, lotStatusLabel, qcError, refreshQc, type LotDraft } from "../../lib/qc";
import { fmtDay } from "../../lib/exams";
import { todayInCity } from "../../lib/campaigns";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, okText, row, stack } from "../exams/styles";

const EMPTY_LOT: LotDraft = { materialId: "", lotNumber: "", expiresOn: "", barcodes: {} };

export function LotsPanel({ providerId, pickedId, onPick }: { providerId: string; pickedId: string | null; onPick(lotId: string): void }) {
  const queryClient = useQueryClient();
  const lots = useQuery({ queryKey: [ QC_LOTS_KEY, providerId ], queryFn: () => listQcLots({ providerId }), retry: false });
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const [ creating, setCreating ] = useState(false);
  const [ draft, setDraft ] = useState<LotDraft>(EMPTY_LOT);
  const [ closing, setClosing ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const material = (materials.data ?? []).find((m) => m.id === draft.materialId);
  const levels = material?.levels ?? [];

  async function save() {
    const problem = lotProblem(draft, levels, todayInCity());
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await createQcLot(lotBody(draft, levels));
      setCreating(false);
      setDraft(EMPTY_LOT);
      setNotice("Lote salvo — cadastre a referência da bula de cada nível.");
      void refreshQc(queryClient).catch(() => undefined);
    } catch (err) {
      setError(qcError(err));
    }
  }

  async function close(lot: QcLot) {
    setError(null);
    try {
      await closeQcLot(lot.id);
      setClosing(null);
      setNotice(`Lote ${lot.lot_number} encerrado.`);
      void refreshQc(queryClient).catch(() => undefined);
    } catch (err) {
      setError(qcError(err));
    }
  }

  return (
    <Panel title="Lotes de controle" sub="um código de barras por nível; o aparelho manda o controle por esse código"
      right={<button type="button" style={secondaryButtonStyle} onClick={() => { setCreating(true); setDraft(EMPTY_LOT); setError(null); setNotice(null); }}>Novo lote</button>}>
      <div style={stack}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {lots.isError ? <p role="alert" style={alert}>{qcError(lots.error)}</p> : (
          <DataTable<QcLot>
            cols={[
              { label: "Lote", w: "1fr", card: "title", render: (l) => <span className="mono">{l.lot_number}</span> },
              { label: "Material", w: "1.3fr", render: (l) => l.material_name },
              { label: "Validade", w: "0.8fr", render: (l) => fmtDay(l.expires_on) },
              { label: "Códigos por nível", w: "1.6fr", render: (l) => Object.entries(l.level_barcodes).map(([ lv, code ]) => `${lv}: ${code}`).join(" · ") },
              { label: "Situação", w: "0.7fr", render: (l) => <Tag tone={l.status === "in_use" ? "ok" : undefined}>{lotStatusLabel(l.status)}</Tag> },
              { label: "", w: "auto", card: "actions", render: (l) => (
                <div style={row}>
                  <button type="button" style={secondaryButtonStyle} aria-pressed={pickedId === l.id} onClick={() => onPick(l.id)}>Referências</button>
                  {l.status === "in_use" && closing !== l.id && (
                    <button type="button" style={secondaryButtonStyle} onClick={() => setClosing(l.id)}>Encerrar</button>
                  )}
                  {closing === l.id && (
                    <>
                      <ActionButton variant="danger" onClick={() => close(l)}>{`Confirmar encerramento de ${l.lot_number}`}</ActionButton>
                      <button type="button" style={secondaryButtonStyle} onClick={() => setClosing(null)}>Manter</button>
                    </>
                  )}
                </div>
              ) }
            ]}
            rows={lots.data ?? []} rowKey={(l) => l.id} empty={lots.isPending ? "carregando…" : "nenhum lote cadastrado"}
          />
        )}
        {creating && (
          <section aria-label="Novo lote" style={box}>
            <div style={form}>
              <label style={fieldLabel}>Material
                <select value={draft.materialId} style={inputStyle} onChange={(e) => setDraft((d) => ({ ...d, materialId: e.target.value, barcodes: {} }))}>
                  <option value="">—</option>
                  {(materials.data ?? []).filter((m) => m.active).map((m) => <option key={m.id} value={m.id}>{m.name}</option>)}
                </select>
              </label>
              <label style={fieldLabel}>Número do lote<input value={draft.lotNumber} style={inputStyle} onChange={(e) => setDraft((d) => ({ ...d, lotNumber: e.target.value }))} /></label>
              <label style={fieldLabel}>Validade<input type="date" value={draft.expiresOn} style={inputStyle} onChange={(e) => setDraft((d) => ({ ...d, expiresOn: e.target.value }))} /></label>
              {levels.map((lv) => (
                <label key={lv} style={fieldLabel}>{`Código de barras — ${lv}`}
                  <input value={draft.barcodes[lv] ?? ""} autoComplete="off" style={inputStyle}
                    onKeyDown={(e) => { if (e.key === "Enter") e.preventDefault(); }}
                    onChange={(e) => setDraft((d) => ({ ...d, barcodes: { ...d.barcodes, [lv]: e.target.value } }))} />
                </label>
              ))}
            </div>
            {error && <p role="alert" style={alert}>{error}</p>}
            <div style={row}>
              <ActionButton onClick={save}>Salvar lote</ActionButton>
              <button type="button" style={secondaryButtonStyle} onClick={() => setCreating(false)}>Cancelar</button>
            </div>
          </section>
        )}
        {!creating && error && <p role="alert" style={alert}>{error}</p>}
      </div>
    </Panel>
  );
}
```

`src/App.tsx`:

```tsx
import { LabQcSetup } from "./modules/LabQcSetup";
```

```tsx
    case "lab-qc-setup": return <LabQcSetup />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabQcSetup.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabQcSetup.tsx src/modules/LabQcSetup.test.tsx src/modules/qc/MaterialsPanel.tsx src/modules/qc/LotsPanel.tsx src/App.tsx
$G commit -m "feat: add quality control materials and lots with level barcodes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: CIQ — referências (bula, sugestão, aceite pelo RT) e regras por analito

**Files:**
- Create: `src/modules/qc/TargetsPanel.tsx`, `src/modules/qc/RulesPanel.tsx`
- Modify: `src/modules/LabQcSetup.tsx`
- Test: `src/modules/qc/TargetsPanel.test.tsx`, `src/modules/qc/RulesPanel.test.tsx`

**Interfaces:**
- Consumes: `getQcTargets`, `createQcTarget`, `listQcSuggestions`, `acceptQcSuggestion`, `getQcRules`, `saveQcRules`, `listLabInstruments`, `listQcLots`, `listQcMaterials` (Task 1); `EMPTY_TARGET`, `targetProblem`, `targetBody`, `reasonProblem`, `rulesProblem`, `ruleLabel`, `targetSourceLabel`, `fmtDecimal`, `analyteName`, `RT_ONLY`, as chaves `QC_*`, `qcError`, `refreshQc` (Task 2); `QC_RULES` (Task 1).
- Produces: `TargetsPanel({ providerId, lotId, lead, exams })`; `RulesPanel({ providerId, lead, exams })`; `SetupBody({ providerId, lead })` mostra o `TargetsPanel` do lote escolhido em `LotsPanel` e o `RulesPanel`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/qc/TargetsPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listQcLots: vi.fn(), listQcMaterials: vi.fn(), listLabInstruments: vi.fn(), getQcTargets: vi.fn(),
    listQcSuggestions: vi.fn(), createQcTarget: vi.fn(), acceptQcSuggestion: vi.fn() };
});

import * as api from "../../lib/api";
import { TargetsPanel } from "./TargetsPanel";
import { DOT_IN_NUMBER } from "../../lib/lab";
import { RT_ONLY } from "../../lib/qc";
import { renderWithProviders } from "../../test/campaignFixtures";
import { GLU_EXAM, instrument, qcLot, qcMaterial, qcSuggestion, qcTarget, qcUser } from "../../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listQcLots, api.listQcMaterials, api.listLabInstruments, api.getQcTargets, api.listQcSuggestions,
  api.createQcTarget, api.acceptQcSuggestion ];
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
  mocked(api.listQcLots).mockResolvedValue([ qcLot() ]);
  mocked(api.listQcMaterials).mockResolvedValue([ qcMaterial() ]);
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.getQcTargets).mockResolvedValue({ current: [ qcTarget() ], history: [ qcTarget() ] });
  mocked(api.listQcSuggestions).mockResolvedValue([ qcSuggestion() ]);
  mocked(api.createQcTarget).mockResolvedValue(qcTarget({ id: "qt2", level: "patológico", mean: "250.0", sd: "6.0" }));
  mocked(api.acceptQcSuggestion).mockResolvedValue(qcTarget({ id: "qt3", version: 2, source: "laboratory", mean: "97.6", sd: "2.1", based_on_runs: 20 }));
});

const render = (lead = true) => renderWithProviders(<TargetsPanel providerId="pv9" lotId="ql1" lead={lead} exams={[ GLU_EXAM ]} />);

async function fill(form: HTMLElement, level: string, mean: string, sd: string) {
  await within(form).findByRole("option", { name: "BS-200" });
  await within(form).findByRole("option", { name: level });
  fireEvent.change(within(form).getByLabelText("Nível"), { target: { value: level } });
  fireEvent.change(within(form).getByLabelText("Analito"), { target: { value: "glicose" } });
  fireEvent.change(within(form).getByLabelText("Aparelho"), { target: { value: "in1" } });
  fireEvent.change(within(form).getByLabelText("Média"), { target: { value: mean } });
  fireEvent.change(within(form).getByLabelText("Desvio-padrão"), { target: { value: sd } });
}

describe("TargetsPanel", () => {
  it("vigentes com versão e origem; referência da bula nova com vírgula", async () => {
    render();
    const panel = await screen.findByRole("region", { name: "Referências do lote L2026A" });
    expect(await within(panel).findByText("v1 · bula")).not.toBeNull();
    const form = within(panel).getByRole("group", { name: "Referência da bula" });
    await fill(form, "patológico", "250,0", "6,0");
    expect(within(form).queryByLabelText("Motivo da troca")).toBeNull();
    fireEvent.click(within(form).getByRole("button", { name: "Salvar referência da bula" }));
    await waitFor(() => expect(api.createQcTarget).toHaveBeenCalledWith({
      lot_id: "ql1", level: "patológico", analyte_key: "glicose", instrument_id: "in1", mean: "250.0", sd: "6.0"
    }));
    expect(await within(panel).findByText("Referência da bula salva.")).not.toBeNull();
  });

  it("trocar a vigente pede o motivo; ponto é recusado", async () => {
    render();
    const panel = await screen.findByRole("region", { name: "Referências do lote L2026A" });
    await within(panel).findByText("v1 · bula");
    const form = within(panel).getByRole("group", { name: "Referência da bula" });
    await fill(form, "normal", "98.5", "2,4");
    fireEvent.click(within(form).getByRole("button", { name: "Salvar referência da bula" }));
    expect(within(form).getByRole("alert").textContent).toBe(DOT_IN_NUMBER);
    fireEvent.change(within(form).getByLabelText("Média"), { target: { value: "98,5" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar referência da bula" }));
    expect(within(form).getByRole("alert").textContent).toBe("escreva o motivo da troca da referência (ao menos 10 caracteres)");
    fireEvent.change(within(form).getByLabelText("Motivo da troca"), { target: { value: "bula revisada pelo fabricante" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar referência da bula" }));
    await waitFor(() => expect(api.createQcTarget).toHaveBeenCalledWith(expect.objectContaining({ mean: "98.5", sd: "2.4", reason: "bula revisada pelo fabricante" })));
  });

  it("RT aceita a sugestão com motivo; vira versão nova do laboratório", async () => {
    render(true);
    const box = await screen.findByRole("group", { name: "Sugestões do laboratório" });
    expect(await within(box).findByText("20 corridas aprovadas")).not.toBeNull();
    fireEvent.click(within(box).getByRole("button", { name: "Aceitar" }));
    fireEvent.click(within(box).getByRole("button", { name: "Confirmar aceite" }));
    expect(within(box).getByRole("alert").textContent).toBe("escreva o motivo do aceite (ao menos 10 caracteres)");
    fireEvent.change(within(box).getByLabelText("Motivo do aceite"), { target: { value: "20 corridas estáveis no lote" } });
    fireEvent.click(within(box).getByRole("button", { name: "Confirmar aceite" }));
    await waitFor(() => expect(api.acceptQcSuggestion).toHaveBeenCalledWith({
      lot_id: "ql1", level: "normal", analyte_key: "glicose", instrument_id: "in1", reason: "20 corridas estáveis no lote"
    }));
    expect(await screen.findByText("Referência do laboratório aceita — vale para as próximas corridas.")).not.toBeNull();
  });

  it("quem não é RT vê a sugestão sem o botão", async () => {
    render(false);
    const box = await screen.findByRole("group", { name: "Sugestões do laboratório" });
    await within(box).findByText("20 corridas aprovadas");
    expect(within(box).queryByRole("button", { name: "Aceitar" })).toBeNull();
    expect(within(box).getByText(RT_ONLY)).not.toBeNull();
  });
});
```

```tsx
// src/modules/qc/RulesPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listQcMaterials: vi.fn(), listLabInstruments: vi.fn(), getQcRules: vi.fn(), saveQcRules: vi.fn() };
});

import * as api from "../../lib/api";
import { RulesPanel } from "./RulesPanel";
import { RT_ONLY } from "../../lib/qc";
import { renderWithProviders } from "../../test/campaignFixtures";
import { GLU_EXAM, instrument, qcMaterial, qcRules, qcUser } from "../../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.listQcMaterials, api.listLabInstruments, api.getQcRules, api.saveQcRules ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
  mocked(api.listQcMaterials).mockResolvedValue([ qcMaterial() ]);
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.getQcRules).mockResolvedValue(qcRules());
  mocked(api.saveQcRules).mockResolvedValue(qcRules({ is_default: false, rules: { ...qcRules().rules, "4_1s": false } }));
});

async function choose(lead: boolean) {
  renderWithProviders(<RulesPanel providerId="pv9" lead={lead} exams={[ GLU_EXAM ]} />);
  const panel = await screen.findByRole("region", { name: "Regras de Westgard" });
  await within(panel).findByRole("option", { name: "BS-200" });
  fireEvent.change(within(panel).getByLabelText("Aparelho"), { target: { value: "in1" } });
  fireEvent.change(within(panel).getByLabelText("Analito"), { target: { value: "glicose" } });
  await within(panel).findByText("padrão: todas as regras ligadas");
  return panel;
}

describe("RulesPanel", () => {
  it("RT desliga 4-1s: sem justificativa recusa; com, salva", async () => {
    const panel = await choose(true);
    expect(api.getQcRules).toHaveBeenCalledWith("in1", "glicose");
    fireEvent.click(within(panel).getByLabelText("4-1s"));
    fireEvent.click(within(panel).getByRole("button", { name: "Salvar regras" }));
    expect(within(panel).getByRole("alert").textContent).toBe("escreva a justificativa para desligar 4-1s (ao menos 10 caracteres)");
    expect(api.saveQcRules).not.toHaveBeenCalled();
    fireEvent.change(within(panel).getByLabelText("Justificativa"), { target: { value: "aparelho em validação, 4-1s gera falso alarme" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Salvar regras" }));
    await waitFor(() => expect(api.saveQcRules).toHaveBeenCalledWith({
      instrument_id: "in1", analyte_key: "glicose", rules: { "1_2s": true, "1_3s": true, "2_2s": true, "R_4s": true, "4_1s": false, "10x": true },
      justification: "aparelho em validação, 4-1s gera falso alarme"
    }));
    expect(await within(panel).findByText("Regras salvas.")).not.toBeNull();
  });

  it("quem não é RT só lê", async () => {
    const panel = await choose(false);
    expect((within(panel).getByLabelText("4-1s") as HTMLInputElement).disabled).toBe(true);
    expect(within(panel).queryByRole("button", { name: "Salvar regras" })).toBeNull();
    expect(within(panel).getByText(RT_ONLY)).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/qc/TargetsPanel.test.tsx src/modules/qc/RulesPanel.test.tsx`
Expected: FAIL — módulos não existem.

- [ ] **Step 3: Write minimal implementation**

`src/modules/qc/TargetsPanel.tsx`:

```tsx
// Referências do lote (spec §5.1 e §5.8; contrato §3): a da bula por nível,
// analito e aparelho; a sugestão do laboratório depois de 20 corridas
// aprovadas, aceita pelo RT/substituto. Cada troca é uma versão nova e pede
// motivo (spec §9; Divergência C6). Decimais com vírgula.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  acceptQcSuggestion, createQcTarget, getQcTargets, listLabInstruments, listQcLots, listQcMaterials, listQcSuggestions,
  type LabExamRef, type QcSuggestion, type QcTarget
} from "../../lib/api";
import {
  EMPTY_TARGET, QC_INSTRUMENTS_KEY, QC_LOTS_KEY, QC_MATERIALS_KEY, QC_SUGGESTIONS_KEY, QC_TARGETS_KEY, RT_ONLY, analyteName, fmtDecimal,
  qcError, reasonProblem, refreshQc, targetBody, targetProblem, targetSourceLabel, type TargetDraft
} from "../../lib/qc";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, fieldset, legend, list, muted, okText, row, stack } from "../exams/styles";

export function TargetsPanel({ providerId, lotId, lead, exams }: { providerId: string; lotId: string; lead: boolean; exams: LabExamRef[] }) {
  const queryClient = useQueryClient();
  const lots = useQuery({ queryKey: [ QC_LOTS_KEY, providerId ], queryFn: () => listQcLots({ providerId }), retry: false });
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const targets = useQuery({ queryKey: [ QC_TARGETS_KEY, lotId ], queryFn: () => getQcTargets(lotId), retry: false });
  const suggestions = useQuery({ queryKey: [ QC_SUGGESTIONS_KEY, providerId ], queryFn: () => listQcSuggestions(providerId), retry: false });
  const [ draft, setDraft ] = useState<TargetDraft>(EMPTY_TARGET);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ accepting, setAccepting ] = useState<QcSuggestion | null>(null);
  const [ reason, setReason ] = useState("");
  const [ acceptError, setAcceptError ] = useState<string | null>(null);

  const lot = (lots.data ?? []).find((l) => l.id === lotId);
  const material = (materials.data ?? []).find((m) => m.id === lot?.material_id);
  const levels = material?.levels ?? [];
  const analytes = material?.analytes ?? [];
  const instName = (id: string) => (instruments.data ?? []).find((i) => i.id === id)?.name ?? id;
  const anName = (key: string) => {
    const a = analytes.find((x) => x.analyte_key === key);
    return a ? analyteName(exams, a.lab_exam_id, key) : key;
  };
  const current = targets.data?.current ?? [];
  const replacing = current.some((t) => t.level === draft.level && t.analyte_key === draft.analyteKey && t.instrument_id === draft.instrumentId);
  const mine = (suggestions.data ?? []).filter((s) => s.lot_id === lotId);
  const set = (patch: Partial<TargetDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function save() {
    const problem = targetProblem(draft, replacing);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await createQcTarget(targetBody(draft, lotId, replacing));
      setDraft(EMPTY_TARGET);
      setNotice("Referência da bula salva.");
      void refreshQc(queryClient).catch(() => undefined);
    } catch (err) {
      setError(qcError(err));
    }
  }

  async function accept() {
    if (!accepting) return;
    const problem = reasonProblem(reason, "o motivo do aceite");
    if (problem) { setAcceptError(problem); return; }
    setAcceptError(null);
    try {
      await acceptQcSuggestion({
        lot_id: accepting.lot_id, level: accepting.level, analyte_key: accepting.analyte_key, instrument_id: accepting.instrument_id, reason: reason.trim()
      });
      setAccepting(null);
      setReason("");
      setNotice("Referência do laboratório aceita — vale para as próximas corridas.");
      void refreshQc(queryClient).catch(() => undefined);
    } catch (err) {
      setAcceptError(qcError(err));
    }
  }

  const targetLine = (t: QcTarget) =>
    `v${t.version} · ${t.level} · ${anName(t.analyte_key)} · ${instName(t.instrument_id)} · ${targetSourceLabel(t.source)} · média ${fmtDecimal(t.mean)} · dp ${fmtDecimal(t.sd)}${t.accepted_by ? ` · ${t.accepted_by}` : ""} · ${fmtDateTime(t.accepted_at)}`;

  return (
    <Panel title={`Referências do lote ${lot?.lot_number ?? ""}`.trim()} sub="média e desvio-padrão por nível, analito e aparelho; cada troca é uma versão nova">
      <div style={stack}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {targets.isError ? <p role="alert" style={alert}>{qcError(targets.error)}</p> : (
          <DataTable<QcTarget>
            cols={[
              { label: "Nível", w: "0.8fr", card: "title", render: (t) => t.level },
              { label: "Analito", w: "1.3fr", render: (t) => anName(t.analyte_key) },
              { label: "Aparelho", w: "1fr", render: (t) => instName(t.instrument_id) },
              { label: "Versão", w: "0.8fr", render: (t) => `v${t.version} · ${targetSourceLabel(t.source)}` },
              { label: "Média", w: "0.7fr", align: "right", render: (t) => fmtDecimal(t.mean) },
              { label: "DP", w: "0.6fr", align: "right", render: (t) => fmtDecimal(t.sd) }
            ]}
            rows={current} rowKey={(t) => t.id} empty={targets.isPending ? "carregando…" : "nenhuma referência — cadastre a da bula"}
          />
        )}
        {(targets.data?.history ?? []).length > 0 && (
          <ul aria-label="histórico das referências" style={list}>
            {(targets.data?.history ?? []).map((t) => <li key={t.id}>{targetLine(t)}</li>)}
          </ul>
        )}
        <fieldset role="group" aria-label="Referência da bula" style={fieldset}>
          <legend style={legend}>Referência da bula</legend>
          <div style={row}>
            <label style={fieldLabel}>Nível
              <select value={draft.level} style={inputStyle} onChange={(e) => set({ level: e.target.value })}>
                <option value="">—</option>
                {levels.map((l) => <option key={l} value={l}>{l}</option>)}
              </select>
            </label>
            <label style={fieldLabel}>Analito
              <select value={draft.analyteKey} style={inputStyle} onChange={(e) => set({ analyteKey: e.target.value })}>
                <option value="">—</option>
                {analytes.map((a) => <option key={a.analyte_key} value={a.analyte_key}>{analyteName(exams, a.lab_exam_id, a.analyte_key)}</option>)}
              </select>
            </label>
            <label style={fieldLabel}>Aparelho
              <select value={draft.instrumentId} style={inputStyle} onChange={(e) => set({ instrumentId: e.target.value })}>
                <option value="">—</option>
                {(instruments.data ?? []).filter((i) => i.active).map((i) => <option key={i.id} value={i.id}>{i.name}</option>)}
              </select>
            </label>
            <label style={fieldLabel}>Média<input value={draft.mean} inputMode="decimal" autoComplete="off" style={inputStyle} onChange={(e) => set({ mean: e.target.value })} /></label>
            <label style={fieldLabel}>Desvio-padrão<input value={draft.sd} inputMode="decimal" autoComplete="off" style={inputStyle} onChange={(e) => set({ sd: e.target.value })} /></label>
          </div>
          {replacing && (
            <label style={fieldLabel}>Motivo da troca
              <textarea value={draft.reason} rows={2} style={inputStyle} onChange={(e) => set({ reason: e.target.value })} />
            </label>
          )}
          {error && <p role="alert" style={alert}>{error}</p>}
          <div><ActionButton onClick={save}>Salvar referência da bula</ActionButton></div>
        </fieldset>
        <fieldset role="group" aria-label="Sugestões do laboratório" style={fieldset}>
          <legend style={legend}>Sugestões do laboratório</legend>
          <p style={muted}>calculadas pelo sistema com 20 corridas aprovadas no lote; o RT aceita e vira a referência vigente</p>
          <DataTable<QcSuggestion>
            cols={[
              { label: "Nível · analito", w: "1.4fr", card: "title", render: (s) => `${s.level} · ${anName(s.analyte_key)}` },
              { label: "Aparelho", w: "1fr", render: (s) => instName(s.instrument_id) },
              { label: "Média · dp", w: "1fr", render: (s) => `${fmtDecimal(s.mean)} · ${fmtDecimal(s.sd)}` },
              { label: "Base", w: "1fr", render: (s) => `${s.runs} corridas aprovadas` },
              { label: "", w: "auto", card: "actions", render: (s) => (lead ? (
                <button type="button" style={secondaryButtonStyle} onClick={() => { setAccepting(s); setReason(""); setAcceptError(null); }}>Aceitar</button>
              ) : null) }
            ]}
            rows={mine} rowKey={(s) => `${s.level}:${s.analyte_key}:${s.instrument_id}`} empty={suggestions.isPending ? "carregando…" : "nenhuma sugestão para este lote"}
          />
          {!lead && mine.length > 0 && <p style={muted}>{RT_ONLY}</p>}
          {accepting && (
            <div style={stack}>
              <label style={fieldLabel}>Motivo do aceite
                <textarea value={reason} rows={2} style={inputStyle} onChange={(e) => setReason(e.target.value)} />
              </label>
              {acceptError && <p role="alert" style={alert}>{acceptError}</p>}
              <div style={row}>
                <ActionButton onClick={accept}>Confirmar aceite</ActionButton>
                <button type="button" style={secondaryButtonStyle} onClick={() => setAccepting(null)}>Cancelar</button>
              </div>
            </div>
          )}
        </fieldset>
      </div>
    </Panel>
  );
}
```

`src/modules/qc/RulesPanel.tsx`:

```tsx
// Regras de Westgard por aparelho e analito (spec §4 "lab_qc_rules"; contrato
// §3): ausente = padrão (todas ligadas). Desligar exige justificativa e o
// RT/substituto; os outros só leem.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { QC_RULES, getQcRules, listLabInstruments, listQcMaterials, saveQcRules, type LabExamRef, type QcRule, type QcRuleSet } from "../../lib/api";
import { QC_INSTRUMENTS_KEY, QC_MATERIALS_KEY, QC_RULES_KEY, RT_ONLY, analyteName, qcError, ruleLabel, rulesProblem } from "../../lib/qc";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle } from "../../components/formStyles";
import { alert, checkLabel, fieldLabel, fieldset, legend, muted, okText, row, stack } from "../exams/styles";

export function RulesPanel({ providerId, lead, exams }: { providerId: string; lead: boolean; exams: LabExamRef[] }) {
  const queryClient = useQueryClient();
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const [ instrumentId, setInstrumentId ] = useState("");
  const [ analyteKey, setAnalyteKey ] = useState("");
  const [ notice, setNotice ] = useState<string | null>(null);
  const analytes = [ ...new Map((materials.data ?? []).flatMap((m) => m.analytes).map((a) => [ a.analyte_key, a ])).values() ];
  const rules = useQuery({
    queryKey: [ QC_RULES_KEY, instrumentId, analyteKey ], queryFn: () => getQcRules(instrumentId, analyteKey),
    enabled: instrumentId !== "" && analyteKey !== "", retry: false
  });

  return (
    <Panel title="Regras de Westgard" sub="por aparelho e analito; 1-2s só avisa, as outras reprovam a corrida e bloqueiam a liberação">
      <div style={stack}>
        <div style={row}>
          <label style={fieldLabel}>Aparelho
            <select value={instrumentId} style={inputStyle} onChange={(e) => { setInstrumentId(e.target.value); setNotice(null); }}>
              <option value="">—</option>
              {(instruments.data ?? []).map((i) => <option key={i.id} value={i.id}>{i.name}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>Analito
            <select value={analyteKey} style={inputStyle} onChange={(e) => { setAnalyteKey(e.target.value); setNotice(null); }}>
              <option value="">—</option>
              {analytes.map((a) => <option key={a.analyte_key} value={a.analyte_key}>{analyteName(exams, a.lab_exam_id, a.analyte_key)}</option>)}
            </select>
          </label>
        </div>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {rules.isError && <p role="alert" style={alert}>{qcError(rules.error)}</p>}
        {rules.data && (
          <RulesForm key={`${instrumentId}:${analyteKey}:${rules.dataUpdatedAt}`} set={rules.data} lead={lead}
            onSaved={() => { setNotice("Regras salvas."); void queryClient.invalidateQueries({ queryKey: [ QC_RULES_KEY ] }); }} />
        )}
      </div>
    </Panel>
  );
}

function RulesForm({ set, lead, onSaved }: { set: QcRuleSet; lead: boolean; onSaved(): void }) {
  const [ after, setAfter ] = useState<Record<QcRule, boolean>>({ ...set.rules });
  const [ justification, setJustification ] = useState("");
  const [ error, setError ] = useState<string | null>(null);

  async function save() {
    const problem = rulesProblem(set.rules, after, justification);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await saveQcRules({ instrument_id: set.instrument_id, analyte_key: set.analyte_key, rules: after, justification: justification.trim() });
      onSaved();
    } catch (err) {
      setError(qcError(err));
    }
  }

  return (
    <fieldset style={fieldset}>
      <legend style={legend}>Regras ligadas</legend>
      <p style={muted}>
        {set.is_default
          ? "padrão: todas as regras ligadas"
          : `alteradas${set.changed_by ? ` por ${set.changed_by}` : ""} em ${fmtDateTime(set.changed_at)}${set.justification ? ` — ${set.justification}` : ""}`}
      </p>
      <div style={row}>
        {QC_RULES.map((r) => (
          <label key={r} style={checkLabel}>
            <input type="checkbox" disabled={!lead} checked={after[r]} onChange={(e) => setAfter((a) => ({ ...a, [r]: e.target.checked }))} />{ruleLabel(r)}
          </label>
        ))}
      </div>
      {lead ? (
        <>
          <label style={fieldLabel}>Justificativa
            <textarea value={justification} rows={2} style={inputStyle} onChange={(e) => setJustification(e.target.value)} />
          </label>
          {error && <p role="alert" style={alert}>{error}</p>}
          <div><ActionButton onClick={save}>Salvar regras</ActionButton></div>
        </>
      ) : <p style={muted}>{RT_ONLY}</p>}
    </fieldset>
  );
}
```

Em `src/modules/LabQcSetup.tsx`, passe `lead` e mostre os painéis:

```tsx
      {(roles) => <LabPick>{(lab) => (roles.admin || lab.lead ? <SetupBody providerId={lab.id} lead={lab.lead} /> : <EmptyState title={QC_LEAD_ONLY} />)}</LabPick>}
```

```tsx
import { TargetsPanel } from "./qc/TargetsPanel";
import { RulesPanel } from "./qc/RulesPanel";
```

```tsx
function SetupBody({ providerId, lead }: { providerId: string; lead: boolean }) {
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const [ lotId, setLotId ] = useState<string | null>(null);
  return (
    <div style={stack}>
      <MaterialsPanel providerId={providerId} exams={exams.data ?? []} />
      <LotsPanel providerId={providerId} pickedId={lotId} onPick={setLotId} />
      {lotId && <TargetsPanel key={lotId} providerId={providerId} lotId={lotId} lead={lead} exams={exams.data ?? []} />}
      <RulesPanel providerId={providerId} lead={lead} exams={exams.data ?? []} />
    </div>
  );
}
```

E acrescente ao `describe` de `src/modules/LabQcSetup.test.tsx`:

```tsx
  it("Referências abre o painel do lote escolhido", async () => {
    renderWithProviders(<LabQcSetup />);
    const panel = await screen.findByRole("region", { name: "Lotes de controle" });
    fireEvent.click(await within(panel).findByRole("button", { name: "Referências" }));
    expect(await screen.findByRole("region", { name: "Referências do lote L2026A" })).not.toBeNull();
    expect(api.getQcTargets).toHaveBeenCalledWith("ql1");
  });
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/qc src/modules/LabQcSetup.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/qc/TargetsPanel.tsx src/modules/qc/TargetsPanel.test.tsx src/modules/qc/RulesPanel.tsx src/modules/qc/RulesPanel.test.tsx src/modules/LabQcSetup.tsx src/modules/LabQcSetup.test.tsx
$G commit -m "feat: add quality control targets with technical lead acceptance and Westgard rules per analyte

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Corridas do dia por aparelho e digitação dos valores

**Files:**
- Create: `src/modules/LabQcRuns.tsx`, `src/modules/qc/QcValueForm.tsx`, `src/modules/qc/QcRunCard.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabQcRuns.test.tsx`

**Interfaces:**
- Consumes: `listLabInstruments`, `listLabExams`, `listQcLots`, `listQcMaterials`, `listQcRuns`, `openQcRun`, `recordQcValue` (Task 1); `qcValueProblem`, `analyteName`, `fmtDecimal`, `fmtZ`, `rulesText`, `runStatusLabel`, `runStatusTone`, `outcomeLabel`, `outcomeTone`, `ruleLabel`, `QC_*` chaves, `QC_DISABLED`, `qcError` (Task 2); `normalizeNumber` (21b); `ReagentLotTag`, `LabGate`, `LabPick` (Task 5); `todayInCity`.
- Produces: `LabQcRuns()`; `QcValueForm({ instrumentId, runId, lots, materials, exams, onRecorded(results: QcValueResult[]) })`; `QcRunCard({ run, analyte(key): string })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabQcRuns.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabInstruments: vi.fn(), listLabExams: vi.fn(), listQcLots: vi.fn(),
    listQcMaterials: vi.fn(), listQcRuns: vi.fn(), openQcRun: vi.fn(), recordQcValue: vi.fn() };
});

import * as api from "../lib/api";
import { LabQcRuns } from "./LabQcRuns";
import { DOT_IN_NUMBER } from "../lib/lab";
import { renderWithProviders } from "../test/campaignFixtures";
import { CHOL_EXAM, GLU_EXAM, NOW21C, instrument, qcBlock, qcContext, qcLot, qcMaterial, qcRun, qcUser, qcValue } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listLabInstruments, api.listLabExams, api.listQcLots, api.listQcMaterials,
  api.listQcRuns, api.openQcRun, api.recordQcValue ];
const MATERIAL = qcMaterial({ analytes: [ { lab_exam_id: "le1", analyte_key: "glicose" }, { lab_exam_id: "le2", analyte_key: "ct" }, { lab_exam_id: "le2", analyte_key: "hdl" } ] });
const ok = (key: string) => ({ value: qcValue({ id: `v-${key}`, analyte_key: key }), run: qcRun(), block: null });

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext());
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.listLabExams).mockResolvedValue([ GLU_EXAM, CHOL_EXAM ]);
  mocked(api.listQcLots).mockResolvedValue([ qcLot() ]);
  mocked(api.listQcMaterials).mockResolvedValue([ MATERIAL ]);
  mocked(api.listQcRuns).mockResolvedValue([ qcRun() ]);
  mocked(api.openQcRun).mockResolvedValue(qcRun({ id: "qr9", status: "open", evaluated_at: null, values: [] }));
  mocked(api.recordQcValue).mockImplementation(async (input: { analyte_key: string }) => ok(input.analyte_key));
});

async function openForm() {
  renderWithProviders(<LabQcRuns />);
  const form = await screen.findByRole("region", { name: "Digitar valores de controle" });
  await within(form).findByRole("option", { name: "L2026A · Controle bioquímico" });
  fireEvent.change(within(form).getByLabelText("Lote"), { target: { value: "ql1" } });
  fireEvent.change(within(form).getByLabelText("Nível"), { target: { value: "normal" } });
  return form;
}

describe("LabQcRuns", () => {
  it("corridas de hoje do aparelho: situação, z, regras e lote do reagente", async () => {
    renderWithProviders(<LabQcRuns />);
    expect(await screen.findByText("aprovada")).not.toBeNull();
    expect(api.listQcRuns).toHaveBeenCalledWith("in1", "2026-10-18");
    expect(screen.getByText("+0,4")).not.toBeNull();
    expect(screen.getByText("lote R-778")).not.toBeNull();
  });

  it("série de valores: ponto bloqueia tudo; 422 no meio marca o campo e mantém os não enviados", async () => {
    mocked(api.recordQcValue)
      .mockImplementationOnce(async () => ok("glicose"))
      .mockRejectedValueOnce(new api.ApiError(422, { error: "no_target" }, "x"));
    const form = await openForm();
    fireEvent.change(within(form).getByLabelText("Glicose (Glicose)"), { target: { value: "99,0" } });
    fireEvent.change(within(form).getByLabelText("Colesterol total (Colesterol total e frações)"), { target: { value: "182.5" } });
    fireEvent.change(within(form).getByLabelText("HDL (Colesterol total e frações)"), { target: { value: "-0,3" } });
    fireEvent.click(within(form).getByRole("button", { name: "Gravar valores" }));
    expect(within(form).getByText(DOT_IN_NUMBER)).not.toBeNull();
    expect(api.recordQcValue).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("Colesterol total (Colesterol total e frações)"), { target: { value: "182,5" } });
    fireEvent.click(within(form).getByRole("button", { name: "Gravar valores" }));
    await waitFor(() => expect(api.recordQcValue).toHaveBeenCalledTimes(2));
    expect(mocked(api.recordQcValue).mock.calls[0][0]).toEqual({ run_id: null, instrument_id: "in1", lot_id: "ql1", level: "normal", analyte_key: "glicose", value: "99.0" });
    expect(mocked(api.recordQcValue).mock.calls[1][0].value).toBe("182.5");
    expect(await within(form).findByText("este nível e analito ainda não têm referência neste aparelho — cadastre a da bula em Controles e referências")).not.toBeNull();
    expect((within(form).getByLabelText("Glicose (Glicose)") as HTMLInputElement).value).toBe("");
    expect((within(form).getByLabelText("Colesterol total (Colesterol total e frações)") as HTMLInputElement).value).toBe("182,5");
    expect((within(form).getByLabelText("HDL (Colesterol total e frações)") as HTMLInputElement).value).toBe("-0,3");
  });

  it("corrida reprovada abre bloqueio: a tela avisa e relê as corridas", async () => {
    mocked(api.recordQcValue).mockResolvedValueOnce({
      value: qcValue({ z_score: 2.4, outcome: "reject", rules_fired: [ "2_2s" ] }), run: qcRun({ id: "qr2", status: "rejected" }), block: qcBlock()
    });
    const form = await openForm();
    fireEvent.change(within(form).getByLabelText("Glicose (Glicose)"), { target: { value: "104,0" } });
    fireEvent.click(within(form).getByRole("button", { name: "Gravar valores" }));
    expect((await screen.findByRole("alert", { name: "corrida reprovada" })).textContent)
      .toBe("Corrida reprovada (2-2s): bloqueio aberto para Glicose (Glicose) no BS-200. A liberação desse analito fica presa até a ação corretiva e uma corrida aprovada com todos os níveis.");
    await waitFor(() => expect(api.listQcRuns).toHaveBeenCalledTimes(2));
  });

  it("abrir corrida nova: os valores seguintes vão nela", async () => {
    const form = await openForm();
    fireEvent.click(screen.getByRole("button", { name: "Abrir corrida nova" }));
    await waitFor(() => expect(api.openQcRun).toHaveBeenCalledWith("in1"));
    fireEvent.change(within(form).getByLabelText("Glicose (Glicose)"), { target: { value: "98,0" } });
    fireEvent.click(within(form).getByRole("button", { name: "Gravar valores" }));
    await waitFor(() => expect(mocked(api.recordQcValue).mock.calls[0][0].run_id).toBe("qr9"));
  });

  it("outro dia: só leitura", async () => {
    renderWithProviders(<LabQcRuns />);
    await screen.findByText("aprovada");
    fireEvent.change(screen.getByLabelText("Dia"), { target: { value: "2026-10-17" } });
    await waitFor(() => expect(api.listQcRuns).toHaveBeenCalledWith("in1", "2026-10-17"));
    expect(screen.queryByRole("region", { name: "Digitar valores de controle" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabQcRuns.test.tsx`
Expected: FAIL — `./LabQcRuns` não existe.

- [ ] **Step 3: Write minimal implementation**

`src/modules/LabQcRuns.tsx`:

```tsx
// src/modules/LabQcRuns.tsx
// Corridas do dia por aparelho (spec §5.2–§5.3; contrato §3): valores
// digitados ou vindos do aparelho (HL7), com z, regras e desfecho que o api
// calculou na gravação. A corrida do dia abre sozinha no primeiro valor; uma
// nova abre a pedido (ou depois de uma reprovada, pelo api). Leitura ao abrir
// e depois de cada gravação — nenhuma releitura automática.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listLabExams, listLabInstruments, listQcLots, listQcMaterials, listQcRuns, openQcRun, type QcValueResult } from "../lib/api";
import {
  QC_BLOCK_COUNT_KEY, QC_DISABLED, QC_EXAMS_KEY, QC_INSTRUMENTS_KEY, QC_LOTS_KEY, QC_MATERIALS_KEY, QC_RUNS_KEY, analyteName, qcError, rulesText
} from "../lib/qc";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { ActionButton } from "../components/ActionButton";
import { inputStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { QcValueForm } from "./qc/QcValueForm";
import { QcRunCard } from "./qc/QcRunCard";
import { alert, fieldLabel, muted, okText, row, stack } from "./exams/styles";

export function LabQcRuns() {
  return (
    <LabGate title="Corridas de controle" sub="CIQ · valores do dia por aparelho, digitados ou vindos do aparelho"
      allow={(r) => r.staff || r.releaser} requires={[ "lab_quality_control" ]} disabledText={QC_DISABLED}>
      {() => <LabPick>{(lab) => <RunsBody providerId={lab.id} />}</LabPick>}
    </LabGate>
  );
}

const READ_ONCE = { retry: false, refetchOnWindowFocus: false, refetchOnReconnect: false } as const;

function RunsBody({ providerId }: { providerId: string }) {
  const queryClient = useQueryClient();
  const today = todayInCity();
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const lots = useQuery({ queryKey: [ QC_LOTS_KEY, providerId, "in_use" ], queryFn: () => listQcLots({ providerId, status: "in_use" }), retry: false });
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const active = (instruments.data ?? []).filter((i) => i.active);
  const [ picked, setPicked ] = useState("");
  const instrumentId = picked || (active.length === 1 ? active[0].id : "");
  const instrument = active.find((i) => i.id === instrumentId);
  const [ date, setDate ] = useState(today);
  const validDate = /^\d{4}-\d{2}-\d{2}$/.test(date);
  const runs = useQuery({
    queryKey: [ QC_RUNS_KEY, instrumentId, date ], queryFn: () => listQcRuns(instrumentId, date), enabled: instrumentId !== "" && validDate,
    gcTime: 0, staleTime: 0, ...READ_ONCE
  });
  const [ runId, setRunId ] = useState<string | null>(null);
  const [ last, setLast ] = useState<QcValueResult[]>([]);
  const [ error, setError ] = useState<string | null>(null);
  const examList = exams.data ?? [];
  const materialList = materials.data ?? [];
  const analyte = (key: string) => {
    const ref = materialList.flatMap((m) => m.analytes).find((a) => a.analyte_key === key);
    return ref ? analyteName(examList, ref.lab_exam_id, key) : key;
  };

  async function openRun() {
    setError(null);
    try {
      const run = await openQcRun(instrumentId);
      setRunId(run.id);
      void runs.refetch();
    } catch (err) {
      setError(qcError(err));
    }
  }

  function recorded(results: QcValueResult[]) {
    setLast(results);
    void runs.refetch();
    void queryClient.invalidateQueries({ queryKey: [ QC_BLOCK_COUNT_KEY ] }).catch(() => undefined);
  }

  const blocked = last.find((r) => r.block);
  const lastRun = last.length > 0 ? last[last.length - 1].run : null;

  return (
    <div style={stack}>
      <div style={{ ...row, alignItems: "flex-end" }}>
        <label style={fieldLabel}>Aparelho
          <select value={instrumentId} style={inputStyle} onChange={(e) => { setPicked(e.target.value); setRunId(null); setLast([]); }}>
            <option value="">—</option>
            {active.map((i) => <option key={i.id} value={i.id}>{i.name}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>Dia<input type="date" value={date} max={today} style={inputStyle} onChange={(e) => { setDate(e.target.value); setRunId(null); setLast([]); }} /></label>
        {instrumentId && date === today && <ActionButton variant="secondary" onClick={openRun}>Abrir corrida nova</ActionButton>}
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      {!instrumentId && <p style={muted}>escolha o aparelho para ver as corridas</p>}
      {blocked && blocked.block && (
        <p role="alert" aria-label="corrida reprovada" style={alert}>
          {`Corrida reprovada (${rulesText(blocked.value.rules_fired)}): bloqueio aberto para ${analyte(blocked.block.analyte_key)} no ${instrument?.name ?? "aparelho"}. A liberação desse analito fica presa até a ação corretiva e uma corrida aprovada com todos os níveis.`}
        </p>
      )}
      {!blocked && lastRun && <p role="status" style={okText}>{`Valores gravados · corrida ${lastRun.status === "warning" ? "aprovada com aviso (1-2s)" : lastRun.status === "rejected" ? "reprovada" : lastRun.status === "approved" ? "aprovada" : "aberta"}.`}</p>}
      {instrumentId && date === today && (
        <QcValueForm instrumentId={instrumentId} runId={runId} lots={lots.data ?? []} materials={materialList} exams={examList} onRecorded={recorded} />
      )}
      {instrumentId && (
        <Panel title="Corridas do dia" sub={runId ? "corrida nova aberta — os próximos valores vão nela" : "a corrida do dia abre sozinha no primeiro valor"}>
          {runs.isError ? <p role="alert" style={alert}>{qcError(runs.error)}</p>
            : runs.isPending ? <p className="mono" style={muted}>carregando…</p>
            : (runs.data ?? []).length === 0 ? <p style={muted}>nenhuma corrida neste dia</p>
            : <div style={stack}>{(runs.data ?? []).map((run) => <QcRunCard key={run.id} run={run} analyte={analyte} />)}</div>}
        </Panel>
      )}
    </div>
  );
}
```

`src/modules/qc/QcValueForm.tsx`:

```tsx
// Digitação dos valores de controle de um lote e nível (contrato §3): um
// campo por analito do material, vírgula decimal (ponto recusado). Nada vai
// enquanto houver campo inválido; depois, um valor por vez, na ordem — o
// primeiro recusado marca o campo e para; os já gravados saem da tela, os
// não enviados ficam. Valor nunca em log nem em storage.
import { useId, useState } from "react";
import { recordQcValue, type LabExamRef, type QcLot, type QcMaterial, type QcValueResult } from "../../lib/api";
import { normalizeNumber } from "../../lib/lab";
import { analyteName, qcError, qcValueProblem } from "../../lib/qc";
import { Panel } from "../../components/Panel";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle } from "../../components/formStyles";
import { alert, fieldLabel, form, muted, row } from "../exams/styles";

export function QcValueForm({ instrumentId, runId, lots, materials, exams, onRecorded }: {
  instrumentId: string; runId: string | null; lots: QcLot[]; materials: QcMaterial[]; exams: LabExamRef[]; onRecorded(results: QcValueResult[]): void;
}) {
  const baseId = useId();
  const [ lotId, setLotId ] = useState("");
  const [ level, setLevel ] = useState("");
  const [ values, setValues ] = useState<Record<string, string>>({});
  const [ fieldErrors, setFieldErrors ] = useState<Record<string, string>>({});
  const [ error, setError ] = useState<string | null>(null);
  const lot = lots.find((l) => l.id === lotId);
  const material = materials.find((m) => m.id === lot?.material_id);

  async function save() {
    if (!lot || !material) { setError("escolha o lote"); return; }
    if (!level) { setError("escolha o nível"); return; }
    const entries = material.analytes.filter((a) => (values[a.analyte_key] ?? "").trim() !== "");
    if (entries.length === 0) { setError("digite ao menos um valor"); return; }
    const problems: Record<string, string> = {};
    for (const a of entries) {
      const p = qcValueProblem(values[a.analyte_key]);
      if (p) problems[a.analyte_key] = p;
    }
    setFieldErrors(problems);
    if (Object.keys(problems).length > 0) { setError("corrija os valores marcados"); return; }
    setError(null);
    const results: QcValueResult[] = [];
    for (const a of entries) {
      try {
        results.push(await recordQcValue({
          run_id: runId, instrument_id: instrumentId, lot_id: lot.id, level, analyte_key: a.analyte_key,
          value: normalizeNumber(values[a.analyte_key]) as string
        }));
        setValues((v) => ({ ...v, [a.analyte_key]: "" }));
      } catch (err) {
        setFieldErrors({ [a.analyte_key]: qcError(err) });
        setError(results.length > 0 ? `${results.length} ${results.length === 1 ? "valor gravado" : "valores gravados"}; o seguinte foi recusado` : "o valor foi recusado");
        break;
      }
    }
    if (results.length > 0) onRecorded(results);
  }

  return (
    <Panel title="Digitar valores de controle" sub="um nível de um lote por vez; o aparelho também manda pela HL7">
      <div style={form}>
        <div style={row}>
          <label style={fieldLabel}>Lote
            <select value={lotId} style={inputStyle} onChange={(e) => { setLotId(e.target.value); setLevel(""); setValues({}); setFieldErrors({}); }}>
              <option value="">—</option>
              {lots.map((l) => <option key={l.id} value={l.id}>{`${l.lot_number} · ${l.material_name}`}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>Nível
            <select value={level} style={inputStyle} onChange={(e) => setLevel(e.target.value)}>
              <option value="">—</option>
              {(material?.levels ?? []).map((lv) => <option key={lv} value={lv}>{lv}</option>)}
            </select>
          </label>
        </div>
        {lots.length === 0 && <p style={muted}>nenhum lote de controle em uso — cadastre em Controles e referências</p>}
        {material && material.analytes.map((a) => {
          const id = `${baseId}-${a.analyte_key}`;
          return (
            <div key={a.analyte_key} style={{ display: "flex", flexDirection: "column", gap: 4 }}>
              <label htmlFor={id} style={fieldLabel}>{analyteName(exams, a.lab_exam_id, a.analyte_key)}</label>
              <input id={id} value={values[a.analyte_key] ?? ""} inputMode="decimal" autoComplete="off" style={inputStyle}
                aria-describedby={fieldErrors[a.analyte_key] ? `${id}-err` : undefined}
                onChange={(e) => setValues((v) => ({ ...v, [a.analyte_key]: e.target.value }))} />
              {fieldErrors[a.analyte_key] && <p id={`${id}-err`} style={alert}>{fieldErrors[a.analyte_key]}</p>}
            </div>
          );
        })}
        {error && <p role="alert" style={alert}>{error}</p>}
        <div><ActionButton onClick={save}>Gravar valores</ActionButton></div>
      </div>
    </Panel>
  );
}
```

`src/modules/qc/QcRunCard.tsx`:

```tsx
// Uma corrida: situação, horários e os valores com z, regras disparadas,
// desfecho, origem e o lote do reagente (contrato §3 e §7).
import type { QcRun, QcValue } from "../../lib/api";
import { fmtDecimal, fmtZ, outcomeLabel, outcomeTone, rulesText, runStatusLabel, runStatusTone } from "../../lib/qc";
import { fmtDateTime } from "../../lib/format";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { ReagentLotTag } from "../inventory/ReagentLotTag";
import { box, muted, row } from "../exams/styles";

export function QcRunCard({ run, analyte }: { run: QcRun; analyte(key: string): string }) {
  return (
    <section aria-label={`Corrida de ${fmtDateTime(run.started_at)}`} style={box}>
      <div style={row}>
        <Tag tone={runStatusTone(run.status)}>{runStatusLabel(run.status)}</Tag>
        <span style={muted}>{`início ${fmtDateTime(run.started_at)}${run.evaluated_at ? ` · avaliada ${fmtDateTime(run.evaluated_at)}` : ""}`}</span>
      </div>
      <DataTable<QcValue>
        cols={[
          { label: "Nível · analito", w: "1.6fr", card: "title", render: (v) => `${v.level} · ${analyte(v.analyte_key)}` },
          { label: "Valor", w: "0.7fr", align: "right", render: (v) => fmtDecimal(v.value) },
          { label: "z", w: "0.5fr", align: "right", render: (v) => fmtZ(v.z_score) },
          { label: "Regras", w: "0.9fr", render: (v) => (v.rules_fired.length > 0 ? rulesText(v.rules_fired) : "—") },
          { label: "Desfecho", w: "0.8fr", render: (v) => <Tag tone={outcomeTone(v.outcome)}>{outcomeLabel(v.outcome)}</Tag> },
          { label: "Origem", w: "1fr", render: (v) => (v.source === "hl7" ? "aparelho" : `digitado${v.entered_by ? ` por ${v.entered_by}` : ""}`) },
          { label: "Reagente", w: "1fr", render: (v) => <ReagentLotTag row={v} /> }
        ]}
        rows={run.values} rowKey={(v) => v.id} empty="nenhum valor nesta corrida"
      />
    </section>
  );
}
```

`src/App.tsx`:

```tsx
import { LabQcRuns } from "./modules/LabQcRuns";
```

```tsx
    case "lab-qc-runs": return <LabQcRuns />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabQcRuns.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabQcRuns.tsx src/modules/LabQcRuns.test.tsx src/modules/qc/QcValueForm.tsx src/modules/qc/QcRunCard.tsx src/App.tsx
$G commit -m "feat: add the daily quality control runs per instrument with comma decimal entry

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Gráfico de Levey-Jennings (Recharts)

**Files:**
- Create: `src/modules/qc/LeveyJennings.tsx`, `src/modules/LabQcChart.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/qc/LeveyJennings.test.tsx`, `src/modules/LabQcChart.test.tsx`

**Interfaces:**
- Consumes: `getQcChart`, `listLabInstruments`, `listLabExams`, `listQcLots`, `listQcMaterials` (Task 1); `ljModel`, `ljBands`, `sdLabel`, `SD_LINES`, `fmtDecimal`, `fmtZ`, `rulesText`, `outcomeLabel`, `outcomeTone`, `targetSourceLabel`, `analyteName`, `QC_CHART_KEY`, `QC_DISABLED`, `qcError` (Task 2); `SizedResizeObserver` (Task 1); `recharts` (`CartesianGrid`, `Line`, `LineChart`, `ReferenceLine`, `ResponsiveContainer`, `Tooltip`, `XAxis`, `YAxis` — os mesmos de `SeriesChart`, mais `ReferenceLine`).
- Produces: `LeveyJennings({ chart: QcChart; label: string })`; `LabQcChart()`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/qc/LeveyJennings.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import { LeveyJennings } from "./LeveyJennings";
import { SizedResizeObserver, qcChart } from "../../test/qcFixtures";

beforeEach(() => { vi.stubGlobal("ResizeObserver", SizedResizeObserver); });
afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

describe("LeveyJennings", () => {
  it("±1/2/3 dp e média, pontos coloridos pelo desfecho", async () => {
    const { container } = render(<LeveyJennings chart={qcChart()} label="Levey-Jennings · glicose" />);
    await waitFor(() => expect(container.querySelectorAll("[data-outcome]").length).toBe(6));
    expect(container.querySelectorAll('[data-outcome="reject"]').length).toBe(1);
    expect(container.querySelectorAll('[data-outcome="warning"]').length).toBe(1);
    expect(screen.getByRole("figure", { name: "Levey-Jennings · glicose" })).not.toBeNull();
  });

  it("duas versões: linha vertical na troca e aviso na legenda", async () => {
    const { container } = render(<LeveyJennings chart={qcChart()} label="LJ" />);
    // 7 horizontais (−3…+3 dp) + 1 vertical na troca da v1 para a v2 (prova técnica do Ambiente).
    await waitFor(() => expect(container.querySelectorAll(".recharts-reference-line-line").length).toBe(8));
    expect(screen.getByText("Referência vigente v2 (laboratório): média 98,0 · dp 2,5")).not.toBeNull();
    expect(screen.getByText("há pontos avaliados contra outra versão da referência — o eixo é em desvios-padrão, cada ponto contra a sua versão")).not.toBeNull();
    const bands = screen.getByRole("list", { name: "faixas da referência vigente" });
    expect(within(bands).getByText("+2 dp: 103,0")).not.toBeNull();
  });

  it("tabela acessível dos pontos com regra disparada", () => {
    render(<LeveyJennings chart={qcChart()} label="LJ" />);
    const table = screen.getByRole("region", { name: "Pontos com regra disparada" });
    expect(within(table).getByText("2-2s")).not.toBeNull();
    expect(within(table).getByText("1-2s (aviso)")).not.toBeNull();
    expect(within(table).getByText("+2,4")).not.toBeNull();
    expect(within(table).getByText("104,0")).not.toBeNull();
  });

  it("uma versão só: sem linha vertical nem aviso; sem pontos: a frase", async () => {
    const one = qcChart({ points: qcChart().points.slice(4) });
    const { container, unmount } = render(<LeveyJennings chart={one} label="LJ" />);
    await waitFor(() => expect(container.querySelectorAll(".recharts-reference-line-line").length).toBe(7));
    expect(screen.queryByText(/outra versão da referência/)).toBeNull();
    unmount();
    render(<LeveyJennings chart={qcChart({ points: [] })} label="LJ" />);
    expect(screen.getByText("nenhum ponto no período")).not.toBeNull();
  });
});
```

```tsx
// src/modules/LabQcChart.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabInstruments: vi.fn(), listLabExams: vi.fn(), listQcLots: vi.fn(),
    listQcMaterials: vi.fn(), getQcChart: vi.fn() };
});

import * as api from "../lib/api";
import { LabQcChart } from "./LabQcChart";
import { renderWithProviders } from "../test/campaignFixtures";
import { GLU_EXAM, NOW21C, SizedResizeObserver, instrument, qcChart, qcContext, qcLot, qcMaterial, qcUser } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listLabInstruments, api.listLabExams, api.listQcLots, api.listQcMaterials, api.getQcChart ];
afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.stubGlobal("ResizeObserver", SizedResizeObserver);
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ] }));
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.listLabExams).mockResolvedValue([ GLU_EXAM ]);
  mocked(api.listQcLots).mockResolvedValue([ qcLot() ]);
  mocked(api.listQcMaterials).mockResolvedValue([ qcMaterial() ]);
  mocked(api.getQcChart).mockResolvedValue(qcChart());
});

describe("LabQcChart", () => {
  it("lote, nível, analito e aparelho; últimos 30 dias; mostra o gráfico", async () => {
    renderWithProviders(<LabQcChart />);
    await screen.findByRole("option", { name: "L2026A · Controle bioquímico" });
    fireEvent.change(screen.getByLabelText("Lote"), { target: { value: "ql1" } });
    fireEvent.change(screen.getByLabelText("Nível"), { target: { value: "normal" } });
    fireEvent.change(screen.getByLabelText("Analito"), { target: { value: "glicose" } });
    await waitFor(() => expect(api.getQcChart).toHaveBeenCalledWith({
      lot_id: "ql1", level: "normal", analyte_key: "glicose", instrument_id: "in1", from: "2026-09-18", to: "2026-10-18"
    }));
    expect(await screen.findByRole("figure", { name: "Levey-Jennings · Glicose (Glicose) · normal · lote L2026A · BS-200" })).not.toBeNull();
  });

  it("período invertido não consulta", async () => {
    renderWithProviders(<LabQcChart />);
    await screen.findByRole("option", { name: "L2026A · Controle bioquímico" });
    fireEvent.change(screen.getByLabelText("Lote"), { target: { value: "ql1" } });
    fireEvent.change(screen.getByLabelText("Nível"), { target: { value: "normal" } });
    fireEvent.change(screen.getByLabelText("Analito"), { target: { value: "glicose" } });
    await waitFor(() => expect(api.getQcChart).toHaveBeenCalledTimes(1));
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-10-19" } });
    expect(screen.getByRole("alert").textContent).toBe("o início do período é depois do fim");
    expect(api.getQcChart).toHaveBeenCalledTimes(1);
  });

  it("equipe sem papel de liberador não vê", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
    renderWithProviders(<LabQcChart />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.getQcChart).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/qc/LeveyJennings.test.tsx src/modules/LabQcChart.test.tsx`
Expected: FAIL — módulos não existem.

- [ ] **Step 3: Write minimal implementation**

`src/modules/qc/LeveyJennings.tsx`:

```tsx
// Gráfico de Levey-Jennings (spec §5.9; contrato §3) com o Recharts já usado
// no Analytics. Eixo em desvios-padrão (cada ponto vem com o z da referência
// da sua versão — Review Focus 3); linhas em ±1/2/3 dp e na média; linha
// vertical onde a versão da referência muda; ponto colorido pelo desfecho.
// A legenda e a tabela dos pontos marcados são HTML: legíveis sem o SVG.
import { CartesianGrid, Line, LineChart, ReferenceLine, ResponsiveContainer, Tooltip, XAxis, YAxis } from "recharts";
import type { QcChart } from "../../lib/api";
import {
  SD_LINES, fmtDecimal, fmtZ, ljBands, ljModel, outcomeLabel, outcomeTone, rulesText, sdLabel, targetSourceLabel, type LjRow
} from "../../lib/qc";
import { fmtDateTime } from "../../lib/format";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { list, muted, row } from "../exams/styles";

const LINE_COLOR: Record<number, string> = { 0: "var(--ink2)", 1: "var(--rule)", 2: "var(--warn)", 3: "var(--down)" };
const DOT_COLOR: Record<string, string> = { ok: "var(--ok)", warning: "var(--warn)", reject: "var(--down)" };

interface DotProps { key?: string; cx?: number; cy?: number; index?: number; payload?: LjRow }

function LjDot({ cx, cy, payload }: DotProps) {
  if (cx === undefined || cy === undefined || !payload) return <g />;
  return (
    <circle cx={cx} cy={cy} r={payload.outcome === "ok" ? 3 : 5} fill={DOT_COLOR[payload.outcome] ?? "var(--ink3)"} stroke="var(--panel)"
      data-outcome={payload.outcome} />
  );
}

function LjTooltip({ active, payload }: { active?: boolean; payload?: { payload?: LjRow }[] }) {
  const point = active ? payload?.[0]?.payload : undefined;
  if (!point) return null;
  return (
    <div style={{ background: "var(--panel)", border: "1px solid var(--rule)", borderRadius: 6, padding: "6px 8px", fontSize: 11, color: "var(--ink)" }}>
      <div>{`#${point.n} · ${fmtDateTime(point.at)}`}</div>
      <div>{`valor ${fmtDecimal(point.value)} · z ${fmtZ(point.z)} · referência v${point.version}`}</div>
      {point.rules.length > 0 && <div>{`regras: ${rulesText(point.rules)}`}</div>}
    </div>
  );
}

export function LeveyJennings({ chart, label }: { chart: QcChart; label: string }) {
  const model = ljModel(chart);
  if (model.rows.length === 0) return <p style={muted}>nenhum ponto no período</p>;
  return (
    <figure aria-label={label} style={{ margin: 0, display: "flex", flexDirection: "column", gap: 8 }}>
      <div style={{ width: "100%", height: 260 }}>
        <ResponsiveContainer width="100%" height="100%">
          <LineChart data={model.rows} margin={{ top: 16, right: 12, bottom: 0, left: 0 }}>
            <CartesianGrid stroke="var(--rule)" vertical={false} />
            <XAxis dataKey="n" tick={{ fontSize: 10 }} />
            <YAxis domain={model.domain} ticks={SD_LINES} tickFormatter={(v: number) => sdLabel(v)} width={52} tick={{ fontSize: 10 }} />
            {SD_LINES.map((k) => (
              <ReferenceLine key={`sd${k}`} y={k} stroke={LINE_COLOR[Math.abs(k)]} strokeDasharray={k === 0 ? undefined : "4 4"} />
            ))}
            {model.versionChanges.map((c) => (
              <ReferenceLine key={`v${c.n}`} x={c.n} stroke="var(--accent)" label={{ value: `v${c.version}`, position: "top", fontSize: 10, fill: "var(--accent)" }} />
            ))}
            <Tooltip content={<LjTooltip />} />
            <Line dataKey="z" stroke="var(--ink3)" strokeWidth={1} isAnimationActive={false} connectNulls={false} activeDot={false}
              dot={(p: DotProps) => <LjDot key={p.key ?? `d${p.index ?? 0}`} cx={p.cx} cy={p.cy} payload={p.payload} />} />
          </LineChart>
        </ResponsiveContainer>
      </div>
      <figcaption style={{ display: "flex", flexDirection: "column", gap: 6, fontSize: 11.5, color: "var(--ink2)" }}>
        <span>{chart.target
          ? `Referência vigente v${chart.target.version} (${targetSourceLabel(chart.target.source)}): média ${fmtDecimal(chart.target.mean)} · dp ${fmtDecimal(chart.target.sd)}`
          : "sem referência vigente para esta chave"}</span>
        {chart.target && (
          <ul aria-label="faixas da referência vigente" style={{ ...list, flexDirection: "row", flexWrap: "wrap", gap: 12, listStyle: "none", paddingLeft: 0 }}>
            {ljBands(chart.target).map((b) => <li key={b.k}>{`${b.label}: ${b.value}`}</li>)}
          </ul>
        )}
        {model.mixedVersions && <span>há pontos avaliados contra outra versão da referência — o eixo é em desvios-padrão, cada ponto contra a sua versão</span>}
        <span style={row}>
          <Tag tone="ok">ok</Tag><Tag tone="warn">aviso (1-2s)</Tag><Tag tone="down">reprovado</Tag>
        </span>
      </figcaption>
      <section aria-label="Pontos com regra disparada">
        <DataTable<LjRow>
          cols={[
            { label: "#", w: "0.4fr", card: "title", render: (r) => String(r.n) },
            { label: "Quando", w: "1fr", render: (r) => fmtDateTime(r.at) },
            { label: "Valor", w: "0.7fr", align: "right", render: (r) => fmtDecimal(r.value) },
            { label: "z", w: "0.5fr", align: "right", render: (r) => fmtZ(r.z) },
            { label: "Regras", w: "1fr", render: (r) => rulesText(r.rules) },
            { label: "Desfecho", w: "0.8fr", render: (r) => <Tag tone={outcomeTone(r.outcome)}>{outcomeLabel(r.outcome)}</Tag> },
            { label: "Referência", w: "0.6fr", render: (r) => `v${r.version}` }
          ]}
          rows={model.flagged} rowKey={(r) => String(r.n)} empty="nenhum ponto com regra disparada no período"
        />
      </section>
    </figure>
  );
}
```

`src/modules/LabQcChart.tsx`:

```tsx
// src/modules/LabQcChart.tsx
// Levey-Jennings por lote, nível, analito e aparelho (spec §5.9; contrato §3),
// do liberador (spec §8). Até 200 pontos; a rota não grava trilha, mas só lê
// quando a escolha muda (sem releitura automática).
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { getQcChart, listLabExams, listLabInstruments, listQcLots, listQcMaterials, type QcChartQuery } from "../lib/api";
import { QC_CHART_KEY, QC_DISABLED, QC_EXAMS_KEY, QC_INSTRUMENTS_KEY, QC_LOTS_KEY, QC_MATERIALS_KEY, analyteName, qcError } from "../lib/qc";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { inputStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { LeveyJennings } from "./qc/LeveyJennings";
import { alert, fieldLabel, muted, row, stack } from "./exams/styles";

function shiftDay(iso: string, days: number): string {
  const d = new Date(`${iso}T12:00:00Z`);
  d.setUTCDate(d.getUTCDate() + days);
  return d.toISOString().slice(0, 10);
}

export function LabQcChart() {
  return (
    <LabGate title="Levey-Jennings" sub="CIQ · controle por lote, nível, analito e aparelho" allow={(r) => r.releaser}
      requires={[ "lab_quality_control" ]} disabledText={QC_DISABLED}>
      {() => <LabPick>{(lab) => <ChartBody providerId={lab.id} />}</LabPick>}
    </LabGate>
  );
}

function ChartBody({ providerId }: { providerId: string }) {
  const today = todayInCity();
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const lots = useQuery({ queryKey: [ QC_LOTS_KEY, providerId ], queryFn: () => listQcLots({ providerId }), retry: false });
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const active = (instruments.data ?? []).filter((i) => i.active);
  const [ pickedInstrument, setPickedInstrument ] = useState("");
  const instrumentId = pickedInstrument || (active.length === 1 ? active[0].id : "");
  const [ lotId, setLotId ] = useState("");
  const [ level, setLevel ] = useState("");
  const [ analyteKey, setAnalyteKey ] = useState("");
  const [ from, setFrom ] = useState(() => shiftDay(today, -30));
  const [ to, setTo ] = useState(today);
  const lot = (lots.data ?? []).find((l) => l.id === lotId);
  const material = (materials.data ?? []).find((m) => m.id === lot?.material_id);
  const ref = material?.analytes.find((a) => a.analyte_key === analyteKey);
  const inverted = from > to;
  const ready = instrumentId !== "" && lotId !== "" && level !== "" && analyteKey !== "" && /^\d{4}-\d{2}-\d{2}$/.test(from) && /^\d{4}-\d{2}-\d{2}$/.test(to) && !inverted;
  const q: QcChartQuery = { lot_id: lotId, level, analyte_key: analyteKey, instrument_id: instrumentId, from, to };
  const chart = useQuery({
    queryKey: [ QC_CHART_KEY, q ], queryFn: () => getQcChart(q), enabled: ready, retry: false, refetchOnWindowFocus: false, refetchOnReconnect: false
  });
  const name = ref ? analyteName(exams.data ?? [], ref.lab_exam_id, analyteKey) : analyteKey;
  const instrumentName = active.find((i) => i.id === instrumentId)?.name ?? "";

  return (
    <div style={stack}>
      <div style={{ ...row, alignItems: "flex-end" }}>
        <label style={fieldLabel}>Aparelho
          <select value={instrumentId} style={inputStyle} onChange={(e) => setPickedInstrument(e.target.value)}>
            <option value="">—</option>
            {active.map((i) => <option key={i.id} value={i.id}>{i.name}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>Lote
          <select value={lotId} style={inputStyle} onChange={(e) => { setLotId(e.target.value); setLevel(""); setAnalyteKey(""); }}>
            <option value="">—</option>
            {(lots.data ?? []).map((l) => <option key={l.id} value={l.id}>{`${l.lot_number} · ${l.material_name}`}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>Nível
          <select value={level} style={inputStyle} onChange={(e) => setLevel(e.target.value)}>
            <option value="">—</option>
            {(material?.levels ?? []).map((lv) => <option key={lv} value={lv}>{lv}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>Analito
          <select value={analyteKey} style={inputStyle} onChange={(e) => setAnalyteKey(e.target.value)}>
            <option value="">—</option>
            {(material?.analytes ?? []).map((a) => <option key={a.analyte_key} value={a.analyte_key}>{analyteName(exams.data ?? [], a.lab_exam_id, a.analyte_key)}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>De<input type="date" value={from} max={today} style={inputStyle} onChange={(e) => setFrom(e.target.value)} /></label>
        <label style={fieldLabel}>Até<input type="date" value={to} max={today} style={inputStyle} onChange={(e) => setTo(e.target.value)} /></label>
      </div>
      {inverted && <p role="alert" style={alert}>o início do período é depois do fim</p>}
      {!ready && !inverted && <p style={muted}>escolha aparelho, lote, nível e analito</p>}
      {ready && (
        <Panel title="Gráfico" sub="até 200 pontos; passe o mouse para ver valor, z e a versão da referência">
          {chart.isError ? <p role="alert" style={alert}>{qcError(chart.error)}</p>
            : chart.isPending ? <p className="mono" style={muted}>carregando…</p>
            : <LeveyJennings chart={chart.data} label={`Levey-Jennings · ${name} · ${level} · lote ${lot?.lot_number ?? ""} · ${instrumentName}`} />}
        </Panel>
      )}
    </div>
  );
}
```

`src/App.tsx`:

```tsx
import { LabQcChart } from "./modules/LabQcChart";
```

```tsx
    case "lab-qc-chart": return <LabQcChart />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/qc/LeveyJennings.test.tsx src/modules/LabQcChart.test.tsx && npx tsc --noEmit`
Expected: PASS. Se a contagem de `.recharts-reference-line-line` não bater, rode a prova técnica do Ambiente com a versão instalada antes de mudar o teste (o seletor é o do Recharts 2.15).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/qc/LeveyJennings.tsx src/modules/qc/LeveyJennings.test.tsx src/modules/LabQcChart.tsx src/modules/LabQcChart.test.tsx src/App.tsx
$G commit -m "feat: add the Levey-Jennings chart with standard deviation bands and target versions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Bloqueios ativos, ação corretiva, exceções concedidas e revisões

**Files:**
- Create: `src/modules/LabQcBlocks.tsx`, `src/modules/qc/CorrectiveActionForm.tsx`, `src/modules/qc/ReviewsPanel.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabQcBlocks.test.tsx`

**Interfaces:**
- Consumes: `listQcBlocks`, `recordCorrectiveAction`, `listQcExceptions`, `listQcReviews`, `decideQcReview`, `listLabInstruments`, `listLabExams`, `listQcMaterials` (Task 1); `reasonProblem`, `reviewDecisionLabel`, `analyteName`, `RT_ONLY`, `QC_BLOCKS_KEY`, `QC_BLOCK_COUNT_KEY`, `QC_EXCEPTIONS_KEY`, `QC_REVIEWS_KEY`, `QC_DISABLED`, `qcError` (Task 2); `ReleaseReview`, `useLabContext`, `refreshLab` (21b); `SegmentedControl`; `LabGate`, `LabPick` (Task 5); `todayInCity`.
- Produces: `LabQcBlocks({ onNavigate })`; `CorrectiveActionForm({ block, label, onDone, onCancel })`; `ReviewsPanel({ providerId, lead, analyte, instrument, onNavigate })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabQcBlocks.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listQcBlocks: vi.fn(), recordCorrectiveAction: vi.fn(), listQcExceptions: vi.fn(),
    listQcReviews: vi.fn(), decideQcReview: vi.fn(), listLabInstruments: vi.fn(), listLabExams: vi.fn(), listQcMaterials: vi.fn(), getReleaseRequisition: vi.fn() };
});

import * as api from "../lib/api";
import { LabQcBlocks } from "./LabQcBlocks";
import { RT_ONLY } from "../lib/qc";
import { renderWithProviders } from "../test/campaignFixtures";
import { labReport, releaseRequisition } from "../test/labFixtures";
import { GLU_EXAM, NOW21C, instrument, qcBlock, qcContext, qcException, qcMaterial, qcReview, qcUser } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listQcBlocks, api.recordCorrectiveAction, api.listQcExceptions, api.listQcReviews,
  api.decideQcReview, api.listLabInstruments, api.listLabExams, api.listQcMaterials, api.getReleaseRequisition ];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ], technical_lead_of: [ "pv9" ], council: "CRF" }));
  mocked(api.listQcBlocks).mockResolvedValue([ qcBlock() ]);
  mocked(api.recordCorrectiveAction).mockResolvedValue(qcBlock({ corrective_action: "calibração refeita e reagente trocado" }));
  mocked(api.listQcExceptions).mockResolvedValue([ qcException() ]);
  mocked(api.listQcReviews).mockResolvedValue([ qcReview() ]);
  mocked(api.decideQcReview).mockResolvedValue(qcReview({ decision: "kept" }));
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.listLabExams).mockResolvedValue([ GLU_EXAM ]);
  mocked(api.listQcMaterials).mockResolvedValue([ qcMaterial() ]);
  mocked(api.getReleaseRequisition).mockResolvedValue(releaseRequisition({ items: releaseRequisition().items.map((i) => ({ ...i, status: "released" })), reports: [ labReport() ] }));
});

describe("LabQcBlocks", () => {
  it("bloqueio ativo com aparelho e analito; ação corretiva registrada relê a lista", async () => {
    renderWithProviders(<LabQcBlocks onNavigate={vi.fn()} />);
    const panel = await screen.findByRole("region", { name: "Bloqueios" });
    expect(await within(panel).findByText("BS-200 · Glicose (Glicose)")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Registrar ação corretiva" }));
    const form = within(panel).getByRole("group", { name: "Ação corretiva — BS-200 · Glicose (Glicose)" });
    fireEvent.change(within(form).getByLabelText("O que foi feito"), { target: { value: "curto" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar" }));
    expect(within(form).getByRole("alert").textContent).toBe("escreva a ação corretiva (ao menos 10 caracteres)");
    expect(api.recordCorrectiveAction).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("O que foi feito"), { target: { value: "calibração refeita e reagente trocado" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar" }));
    await waitFor(() => expect(api.recordCorrectiveAction).toHaveBeenCalledWith("qb1", "calibração refeita e reagente trocado"));
    expect(await screen.findByText("Ação corretiva registrada. O bloqueio fecha sozinho na próxima corrida aprovada com todos os níveis.")).not.toBeNull();
    await waitFor(() => expect(api.listQcBlocks).toHaveBeenCalledTimes(2));
  });

  it("aba de encerrados lê com status=closed", async () => {
    renderWithProviders(<LabQcBlocks onNavigate={vi.fn()} />);
    await screen.findByRole("region", { name: "Bloqueios" });
    fireEvent.click(screen.getByRole("tab", { name: "encerrados" }));
    await waitFor(() => expect(api.listQcBlocks).toHaveBeenCalledWith("closed", "pv9"));
  });

  it("exceções concedidas no mês, com a justificativa", async () => {
    renderWithProviders(<LabQcBlocks onNavigate={vi.fn()} />);
    const panel = await screen.findByRole("region", { name: "Exceções concedidas" });
    expect(await within(panel).findByText("paciente internado, médico assistente ciente")).not.toBeNull();
    expect(api.listQcExceptions).toHaveBeenCalledWith("pv9", "2026-10");
  });

  it("RT mantém o resultado com nota", async () => {
    renderWithProviders(<LabQcBlocks onNavigate={vi.fn()} />);
    const panel = await screen.findByRole("region", { name: "Revisões pendentes" });
    fireEvent.click(await within(panel).findByRole("button", { name: "Manter" }));
    fireEvent.change(within(panel).getByLabelText("Nota (opcional)"), { target: { value: "valor coerente com a série do paciente" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Confirmar: manter" }));
    await waitFor(() => expect(api.decideQcReview).toHaveBeenCalledWith("rv1", { decision: "kept", note: "valor coerente com a série do paciente" }));
    expect(await within(panel).findByText("Resultado mantido.")).not.toBeNull();
  });

  it("RT marca para retificar e a conferência da requisição abre para a retificação do 21b", async () => {
    mocked(api.decideQcReview).mockResolvedValue(qcReview({ decision: "rectify" }));
    renderWithProviders(<LabQcBlocks onNavigate={vi.fn()} />);
    const panel = await screen.findByRole("region", { name: "Revisões pendentes" });
    fireEvent.click(await within(panel).findByRole("button", { name: "Retificar" }));
    fireEvent.click(within(panel).getByRole("button", { name: "Confirmar: retificar" }));
    await waitFor(() => expect(api.decideQcReview).toHaveBeenCalledWith("rv1", { decision: "rectify" }));
    expect(await screen.findByRole("region", { name: "Conferência da requisição 00000123" })).not.toBeNull();
    expect(api.getReleaseRequisition).toHaveBeenCalledWith("rq1");
  });

  it("quem não é RT vê as revisões sem decidir", async () => {
    mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ], council: "CRF" }));
    renderWithProviders(<LabQcBlocks onNavigate={vi.fn()} />);
    const panel = await screen.findByRole("region", { name: "Revisões pendentes" });
    await within(panel).findByText("00000123");
    expect(within(panel).queryByRole("button", { name: "Manter" })).toBeNull();
    expect(within(panel).getByText(RT_ONLY)).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabQcBlocks.test.tsx`
Expected: FAIL — `./LabQcBlocks` não existe.

- [ ] **Step 3: Write minimal implementation**

`src/modules/qc/CorrectiveActionForm.tsx`:

```tsx
// Ação corretiva de um bloqueio (spec §5.7; contrato §3). Registrar não fecha:
// o bloqueio fecha sozinho na corrida aprovada com todos os níveis depois da ação.
import { useState } from "react";
import { recordCorrectiveAction, type QcBlock } from "../../lib/api";
import { qcError, reasonProblem } from "../../lib/qc";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, fieldset, legend, row } from "../exams/styles";

export function CorrectiveActionForm({ block, label, onDone, onCancel }: { block: QcBlock; label: string; onDone(): void; onCancel(): void }) {
  const [ text, setText ] = useState("");
  const [ error, setError ] = useState<string | null>(null);

  async function save() {
    const problem = reasonProblem(text, "a ação corretiva");
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await recordCorrectiveAction(block.id, text.trim());
      onDone();
    } catch (err) {
      setError(qcError(err));
    }
  }

  return (
    <fieldset role="group" aria-label={`Ação corretiva — ${label}`} style={fieldset}>
      <legend style={legend}>{`Ação corretiva — ${label}`}</legend>
      <label style={fieldLabel}>O que foi feito
        <textarea value={text} rows={3} style={inputStyle} onChange={(e) => setText(e.target.value)} />
      </label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton onClick={save}>Registrar</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </fieldset>
  );
}
```

`src/modules/qc/ReviewsPanel.tsx`:

```tsx
// Revisões (spec §5.6; contrato §3): itens liberados desde a última corrida
// aprovada daquele aparelho e analito. O RT/substituto mantém ou marca para
// retificar; "retificar" abre aqui a conferência da requisição do 21b, onde a
// retificação acontece (motivo, contatos e assinatura de sempre). Sem nome de
// paciente na lista; lida ao abrir e depois de cada decisão.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { decideQcReview, listQcReviews, type LabReportSummary, type QcReview } from "../../lib/api";
import { QC_REVIEWS_KEY, RT_ONLY, qcError } from "../../lib/qc";
import { refreshLab } from "../../lib/lab";
import { fmtDateTime } from "../../lib/format";
import type { ModuleId } from "../../shell/modules";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ReleaseReview } from "../lab/ReleaseReview";
import { useLabContext } from "../lab/useLabContext";
import { alert, fieldLabel, muted, okText, row, stack } from "../exams/styles";

export function ReviewsPanel({ providerId, lead, analyte, instrument, onNavigate }: {
  providerId: string; lead: boolean; analyte(key: string): string; instrument(id: string): string; onNavigate(id: ModuleId): void;
}) {
  const queryClient = useQueryClient();
  const context = useLabContext(true);
  const reviews = useQuery({
    queryKey: [ QC_REVIEWS_KEY, providerId ], queryFn: () => listQcReviews(providerId), gcTime: 0, staleTime: 0, retry: false,
    refetchOnWindowFocus: false, refetchOnReconnect: false
  });
  const [ deciding, setDeciding ] = useState<{ review: QcReview; decision: "kept" | "rectify" } | null>(null);
  const [ note, setNote ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ rectifyFor, setRectifyFor ] = useState<string | null>(null);
  const rows = reviews.data ?? [];

  async function decide() {
    if (!deciding) return;
    setError(null);
    try {
      const trimmed = note.trim();
      await decideQcReview(deciding.review.id, { decision: deciding.decision, ...(trimmed ? { note: trimmed } : {}) });
      if (deciding.decision === "rectify") {
        setRectifyFor(deciding.review.requisition_id);
        setNotice(`Marcado para retificar — a conferência da requisição ${deciding.review.requisition_number} abriu abaixo.`);
      } else {
        setNotice("Resultado mantido.");
      }
      setDeciding(null);
      setNote("");
      void reviews.refetch();
    } catch (err) {
      setError(qcError(err));
    }
  }

  function rectified(rep: LabReportSummary) {
    setRectifyFor(null);
    setNotice(rep.status === "pending_signature"
      ? "Laudo retificado enviado para assinatura ICP — assine em Pendentes de assinatura."
      : "Laudo retificado liberado.");
    void refreshLab(queryClient).catch(() => undefined);
  }

  return (
    <Panel title="Revisões pendentes" sub="liberados desde a última corrida aprovada do aparelho e analito reprovados">
      <div style={stack}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {reviews.isError ? <p role="alert" style={alert}>{qcError(reviews.error)}</p> : (
          <DataTable<QcReview>
            cols={[
              { label: "Requisição", w: "1fr", card: "title", render: (r) => <span className="mono">{r.requisition_number}</span> },
              { label: "Exame", w: "1.2fr", render: (r) => r.exam_label },
              { label: "Aparelho · analito", w: "1.4fr", render: (r) => `${instrument(r.instrument_id)} · ${analyte(r.analyte_key)}` },
              { label: "Liberado", w: "1fr", render: (r) => fmtDateTime(r.released_at) },
              { label: "", w: "auto", card: "actions", render: (r) => (lead ? (
                <div style={row}>
                  <button type="button" style={secondaryButtonStyle} onClick={() => { setDeciding({ review: r, decision: "kept" }); setNote(""); setError(null); }}>Manter</button>
                  <button type="button" style={secondaryButtonStyle} onClick={() => { setDeciding({ review: r, decision: "rectify" }); setNote(""); setError(null); }}>Retificar</button>
                </div>
              ) : null) }
            ]}
            rows={rows} rowKey={(r) => r.id} empty={reviews.isPending ? "carregando…" : "nenhuma revisão pendente"}
          />
        )}
        {!lead && rows.length > 0 && <p style={muted}>{RT_ONLY}</p>}
        {deciding && (
          <div style={stack}>
            <label style={fieldLabel}>Nota (opcional)
              <textarea value={note} rows={2} style={inputStyle} onChange={(e) => setNote(e.target.value)} />
            </label>
            {error && <p role="alert" style={alert}>{error}</p>}
            <div style={row}>
              <ActionButton onClick={decide}>{`Confirmar: ${deciding.decision === "kept" ? "manter" : "retificar"}`}</ActionButton>
              <button type="button" style={secondaryButtonStyle} onClick={() => setDeciding(null)}>Cancelar</button>
            </div>
          </div>
        )}
        {!deciding && error && <p role="alert" style={alert}>{error}</p>}
        {rectifyFor && !context.isPending && (
          <ReleaseReview key={rectifyFor} requisitionId={rectifyFor} council={context.data?.council ?? null} onNavigate={onNavigate}
            onClose={() => setRectifyFor(null)} onReleased={rectified} onConflict={(text) => { setRectifyFor(null); setError(text); }} />
        )}
      </div>
    </Panel>
  );
}
```

`src/modules/LabQcBlocks.tsx`:

```tsx
// src/modules/LabQcBlocks.tsx
// Bloqueios do CIQ (spec §5.4–§5.7 e §5.9; contrato §3): ativos e encerrados,
// ação corretiva, exceções concedidas no mês e revisões. O contador do menu é
// a rota /count (Task 4); esta tela lê só quando abre e depois de cada ação.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listLabExams, listLabInstruments, listQcBlocks, listQcExceptions, listQcMaterials, type QcBlock, type QcException } from "../lib/api";
import {
  QC_BLOCKS_KEY, QC_BLOCK_COUNT_KEY, QC_DISABLED, QC_EXAMS_KEY, QC_EXCEPTIONS_KEY, QC_INSTRUMENTS_KEY, QC_MATERIALS_KEY, analyteName, qcError
} from "../lib/qc";
import { todayInCity } from "../lib/campaigns";
import { fmtDateTime } from "../lib/format";
import type { ModuleId } from "../shell/modules";
import { SegmentedControl } from "../shell/SegmentedControl";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { CorrectiveActionForm } from "./qc/CorrectiveActionForm";
import { ReviewsPanel } from "./qc/ReviewsPanel";
import { alert, fieldLabel, okText, stack } from "./exams/styles";

const READ_ONCE = { gcTime: 0, staleTime: 0, retry: false, refetchOnWindowFocus: false, refetchOnReconnect: false } as const;
const STATUS_TABS: { key: "active" | "closed"; label: string }[] = [ { key: "active", label: "ativos" }, { key: "closed", label: "encerrados" } ];

export function LabQcBlocks({ onNavigate }: { onNavigate(id: ModuleId): void }) {
  return (
    <LabGate title="Bloqueios e revisões do CIQ" sub="CIQ · bloqueios, ação corretiva, exceções concedidas e revisões"
      allow={(r) => r.staff || r.releaser} requires={[ "lab_quality_control" ]} disabledText={QC_DISABLED}>
      {() => <LabPick>{(lab) => <BlocksBody providerId={lab.id} lead={lab.lead} onNavigate={onNavigate} />}</LabPick>}
    </LabGate>
  );
}

function BlocksBody({ providerId, lead, onNavigate }: { providerId: string; lead: boolean; onNavigate(id: ModuleId): void }) {
  const queryClient = useQueryClient();
  const [ status, setStatus ] = useState<"active" | "closed">("active");
  const [ month, setMonth ] = useState(() => todayInCity().slice(0, 7));
  const blocks = useQuery({ queryKey: [ QC_BLOCKS_KEY, providerId, status ], queryFn: () => listQcBlocks(status, providerId), ...READ_ONCE });
  const exceptions = useQuery({
    queryKey: [ QC_EXCEPTIONS_KEY, providerId, month ], queryFn: () => listQcExceptions(providerId, month), enabled: /^\d{4}-\d{2}$/.test(month), ...READ_ONCE
  });
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const materials = useQuery({ queryKey: [ QC_MATERIALS_KEY, providerId ], queryFn: () => listQcMaterials(providerId), retry: false });
  const [ acting, setActing ] = useState<QcBlock | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const instrument = (id: string) => (instruments.data ?? []).find((i) => i.id === id)?.name ?? id;
  const analyte = (key: string) => {
    const ref = (materials.data ?? []).flatMap((m) => m.analytes).find((a) => a.analyte_key === key);
    return ref ? analyteName(exams.data ?? [], ref.lab_exam_id, key) : key;
  };
  const label = (b: { instrument_id: string; analyte_key: string }) => `${instrument(b.instrument_id)} · ${analyte(b.analyte_key)}`;

  return (
    <div style={stack}>
      {notice && <p role="status" style={okText}>{notice}</p>}
      <Panel title="Bloqueios" sub="analito bloqueado no aparelho: a liberação desses exames fica presa" right={<SegmentedControl options={STATUS_TABS} value={status} onChange={(s) => { setStatus(s); setActing(null); }} />}>
        <div style={stack}>
          {blocks.isError ? <p role="alert" style={alert}>{qcError(blocks.error)}</p> : (
            <DataTable<QcBlock>
              cols={[
                { label: "Aparelho · analito", w: "1.6fr", card: "title", render: (b) => label(b) },
                { label: "Aberto", w: "1fr", render: (b) => fmtDateTime(b.opened_at) },
                { label: "Ação corretiva", w: "1.8fr", render: (b) => b.corrective_action ?? "—" },
                { label: status === "closed" ? "Encerrado" : "", w: "auto", card: "actions", render: (b) => (status === "closed"
                  ? fmtDateTime(b.closed_at)
                  : !b.corrective_action && (
                    <button type="button" style={secondaryButtonStyle} onClick={() => { setActing(b); setNotice(null); }}>Registrar ação corretiva</button>
                  )) }
              ]}
              rows={blocks.data ?? []} rowKey={(b) => b.id}
              empty={blocks.isPending ? "carregando…" : status === "active" ? "nenhum bloqueio ativo" : "nenhum bloqueio encerrado"}
            />
          )}
          {acting && (
            <CorrectiveActionForm key={acting.id} block={acting} label={label(acting)} onCancel={() => setActing(null)}
              onDone={() => {
                setActing(null);
                setNotice("Ação corretiva registrada. O bloqueio fecha sozinho na próxima corrida aprovada com todos os níveis.");
                void blocks.refetch();
                void queryClient.invalidateQueries({ queryKey: [ QC_BLOCK_COUNT_KEY ] }).catch(() => undefined);
              }} />
          )}
        </div>
      </Panel>
      <Panel title="Exceções concedidas" sub="itens liberados pelo RT ou substituto com o analito bloqueado"
        right={<label style={fieldLabel}>Mês<input type="month" value={month} style={inputStyle} onChange={(e) => setMonth(e.target.value)} /></label>}>
        {exceptions.isError ? <p role="alert" style={alert}>{qcError(exceptions.error)}</p> : (
          <DataTable<QcException>
            cols={[
              { label: "Requisição", w: "0.9fr", card: "title", render: (e) => <span className="mono">{e.requisition_number}</span> },
              { label: "Aparelho · analito", w: "1.4fr", render: (e) => label(e) },
              { label: "Justificativa", w: "2fr", render: (e) => e.justification },
              { label: "Por", w: "1fr", render: (e) => e.granted_by ?? "—" },
              { label: "Quando", w: "1fr", render: (e) => fmtDateTime(e.granted_at) }
            ]}
            rows={exceptions.data ?? []} rowKey={(e) => e.id} empty={exceptions.isPending ? "carregando…" : "nenhuma exceção neste mês"}
          />
        )}
      </Panel>
      <ReviewsPanel providerId={providerId} lead={lead} analyte={analyte} instrument={instrument} onNavigate={onNavigate} />
    </div>
  );
}
```

`src/App.tsx`:

```tsx
import { LabQcBlocks } from "./modules/LabQcBlocks";
```

```tsx
    case "lab-qc-blocks": return <LabQcBlocks onNavigate={setActive} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabQcBlocks.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabQcBlocks.tsx src/modules/LabQcBlocks.test.tsx src/modules/qc/CorrectiveActionForm.tsx src/modules/qc/ReviewsPanel.tsx src/App.tsx
$G commit -m "feat: add quality control blocks with corrective action, granted exceptions and reviews

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Liberação do 21b com `qc_blocked`, exceção do RT e lote nos resultados

**Files:**
- Create: `src/modules/qc/QcBlockedPanel.tsx`, `src/modules/qc/QcExceptionForm.tsx`
- Modify: `src/modules/lab/ReleaseReview.tsx`, `src/modules/LabRelease.tsx`, `src/modules/lab/ResultEntryForm.tsx`
- Test: `src/modules/LabRelease.qc.test.tsx`, `src/modules/lab/ResultEntryForm.lot.test.tsx`

**Interfaces:**
- Consumes: `grantQcException`, `QcBlockedItem`, `ApiError`, `errorCode` (Task 1); `qcError`, `reasonProblem` (Task 2); `ReagentLotTag` (Task 5); `useLabContext` (21b, com `technical_lead_of` — C1).
- Produces: `ReleaseReview` ganha `qcLead?: boolean` (padrão `false`); `blockedItems(err: unknown): QcBlockedItem[]` (em `QcBlockedPanel.tsx`); `QcBlockedPanel({ items, itemLabel, analyteLabel, excepted, lead, onSkip, onExcepted })`; `QcExceptionForm({ blockId, itemId, onDone, onCancel })`. `LabRelease` passa `qcLead`; `ResultEntryForm` e `ReleaseReview` mostram o `ReagentLotTag` em cada valor.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabRelease.qc.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listReleases: vi.fn(), getReleaseRequisition: vi.fn(),
    listUnnotifiedCriticals: vi.fn(), releaseRequisition: vi.fn(), listLabRequisitions: vi.fn(), grantQcException: vi.fn() };
});

import * as api from "../lib/api";
import { LabRelease } from "./LabRelease";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21B, labItem, labReport, labResultRow, releaseRequisition } from "../test/labFixtures";
import { qcContext, qcException, qcUser } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listReleases, api.getReleaseRequisition, api.listUnnotifiedCriticals,
  api.releaseRequisition, api.listLabRequisitions, api.grantQcException ];

const TWO = releaseRequisition({
  items: [
    { ...labItem({ id: "it2", label: "Glicose", sigtap_code: "0202010473", status: "typed" }), restriction_note: null,
      results: [ labResultRow({ reagent_lot: { id: "sl1", lot_number: "R-778" } }) ] },
    { ...labItem({ id: "it3", label: "Hemograma completo", status: "typed" }), restriction_note: null,
      results: [ labResultRow({ id: "res3", item_id: "it3", analyte_key: "hb", name: "Hemoglobina", value: "13.5", unit: "g/dL", reagent_lot: null }) ] }
  ]
});
const BLOCKED = new api.ApiError(409, {
  error: "qc_blocked", items: [ { item_id: "it2", block_id: "qb1", analyte_key: "glicose", instrument_id: "in1", instrument_name: "BS-200" } ]
}, "x");

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21B));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ], council: "CRF" }));
  mocked(api.listReleases).mockResolvedValue([ TWO ]);
  mocked(api.getReleaseRequisition).mockResolvedValue(TWO);
  mocked(api.listUnnotifiedCriticals).mockResolvedValue({ criticals: [], next_after: null });
  mocked(api.listLabRequisitions).mockResolvedValue({ requisitions: [], next_before: null });
  mocked(api.releaseRequisition).mockRejectedValueOnce(BLOCKED).mockResolvedValue(labReport());
  mocked(api.grantQcException).mockResolvedValue(qcException());
});

async function releaseOnce() {
  renderWithProviders(<LabRelease onNavigate={vi.fn()} />);
  const queue = await screen.findByRole("region", { name: "Fila de liberação" });
  fireEvent.click(await within(queue).findByRole("button", { name: "Conferir" }));
  const review = await screen.findByRole("region", { name: "Conferência da requisição 00000123" });
  fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
  fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
  return review;
}

describe("liberação com o CIQ (contrato §3)", () => {
  it("409 qc_blocked: lista, não libera e desmarca só os bloqueados", async () => {
    const review = await releaseOnce();
    const box = await within(review).findByRole("group", { name: "bloqueado pelo controle de qualidade" });
    expect(within(box).getByText("Glicose · Glicose · BS-200")).not.toBeNull();
    expect(screen.queryByText("Laudo liberado com assinatura eletrônica avançada.")).toBeNull();
    fireEvent.click(within(box).getByRole("button", { name: "Liberar sem os bloqueados" }));
    expect((within(review).getByRole("checkbox", { name: /Glicose/ }) as HTMLInputElement).checked).toBe(false);
    expect((within(review).getByRole("checkbox", { name: /Hemograma completo/ }) as HTMLInputElement).checked).toBe(true);
    fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "654321" } });
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo parcial (1 de 2)" }));
    await waitFor(() => expect(api.releaseRequisition).toHaveBeenLastCalledWith("rq1", { item_ids: [ "it3" ], signature: "advanced", totp: "654321" }));
  });

  it("RT: exceção com justificativa e liberação de novo", async () => {
    mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ], technical_lead_of: [ "pv9" ], council: "CRF" }));
    const review = await releaseOnce();
    const box = await within(review).findByRole("group", { name: "bloqueado pelo controle de qualidade" });
    fireEvent.click(within(box).getByRole("button", { name: "Liberar com exceção" }));
    fireEvent.change(within(box).getByLabelText("Justificativa da exceção"), { target: { value: "curta" } });
    fireEvent.click(within(box).getByRole("button", { name: "Registrar exceção" }));
    expect(within(box).getByRole("alert").textContent).toBe("escreva a justificativa da exceção (ao menos 10 caracteres)");
    fireEvent.change(within(box).getByLabelText("Justificativa da exceção"), { target: { value: "paciente internado, médico assistente ciente" } });
    fireEvent.click(within(box).getByRole("button", { name: "Registrar exceção" }));
    await waitFor(() => expect(api.grantQcException).toHaveBeenCalledWith({ block_id: "qb1", item_id: "it2", justification: "paciente internado, médico assistente ciente" }));
    expect(await within(box).findByText("exceção registrada")).not.toBeNull();
    fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "654321" } });
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    await waitFor(() => expect(api.releaseRequisition).toHaveBeenLastCalledWith("rq1", { item_ids: [ "it2", "it3" ], signature: "advanced", totp: "654321" }));
    expect(await screen.findByText("Laudo liberado com assinatura eletrônica avançada.")).not.toBeNull();
  });

  it("quem não é RT não vê o botão da exceção e sabe a quem pedir", async () => {
    const review = await releaseOnce();
    const box = await within(review).findByRole("group", { name: "bloqueado pelo controle de qualidade" });
    expect(within(box).queryByRole("button", { name: "Liberar com exceção" })).toBeNull();
    expect(within(box).getByText("peça a exceção ao responsável técnico ou ao substituto, ou aguarde a corrida aprovada depois da ação corretiva")).not.toBeNull();
  });

  it("lote do reagente em cada valor; null = sem rastreio", async () => {
    renderWithProviders(<LabRelease onNavigate={vi.fn()} />);
    fireEvent.click(await within(await screen.findByRole("region", { name: "Fila de liberação" })).findByRole("button", { name: "Conferir" }));
    const review = await screen.findByRole("region", { name: "Conferência da requisição 00000123" });
    expect(within(review).getByText("lote R-778")).not.toBeNull();
    expect(within(review).getByText("sem rastreio de reagente")).not.toBeNull();
  });
});
```

```tsx
// src/modules/lab/ResultEntryForm.lot.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn() };
});

import * as api from "../../lib/api";
import { ResultEntryForm } from "./ResultEntryForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { glucoseEntry, labResultRow } from "../../test/labFixtures";
import { qcUser } from "../../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => { mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(qcUser([ "lab_staff" ])); });

describe("ResultEntryForm — lote do reagente (contrato §7)", () => {
  it("valor com lote mostra o número; sem lote, sem rastreio", async () => {
    const item = glucoseEntry({ results: [
      labResultRow({ reagent_lot: { id: "sl1", lot_number: "R-778" } }),
      labResultRow({ id: "res2", value: "101", effective: false, chosen: false, reagent_lot: null })
    ] });
    renderWithProviders(<ResultEntryForm item={item} onSaved={vi.fn()} onClose={vi.fn()} />);
    const history = screen.getByRole("list", { name: "valores de Glicose" });
    expect(await within(history).findByText("sem rastreio de reagente")).not.toBeNull();
    expect(within(history).getByText("lote R-778")).not.toBeNull();
  });

  it("api antigo (sem o campo): nada aparece", async () => {
    renderWithProviders(<ResultEntryForm item={glucoseEntry({ results: [ labResultRow() ] })} onSaved={vi.fn()} onClose={vi.fn()} />);
    await vi.waitFor(() => expect(api.fetchCurrentSession).toHaveBeenCalled());
    expect(screen.queryByText(/lote |sem rastreio/)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabRelease.qc.test.tsx src/modules/lab/ResultEntryForm.lot.test.tsx`
Expected: FAIL — o 409 `qc_blocked` cai na frase genérica; a marca do lote não aparece.

- [ ] **Step 3: Write minimal implementation**

`src/modules/qc/QcExceptionForm.tsx`:

```tsx
// Exceção do RT/substituto para um item bloqueado pelo CIQ (spec §5.5;
// contrato §3): justificativa obrigatória; a liberação continua a do 21b
// (mesma assinatura), e o laudo não muda. `not_blocked` = o bloqueio já
// fechou: a tela trata como resolvido.
import { useState } from "react";
import { errorCode, grantQcException } from "../../lib/api";
import { qcError, reasonProblem } from "../../lib/qc";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, row, stack } from "../exams/styles";

export function QcExceptionForm({ blockId, itemId, onDone, onCancel }: { blockId: string; itemId: string; onDone(): void; onCancel(): void }) {
  const [ text, setText ] = useState("");
  const [ error, setError ] = useState<string | null>(null);

  async function save() {
    const problem = reasonProblem(text, "a justificativa da exceção");
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await grantQcException({ block_id: blockId, item_id: itemId, justification: text.trim() });
      onDone();
    } catch (err) {
      if (errorCode(err) === "not_blocked") { onDone(); return; }
      setError(qcError(err));
    }
  }

  return (
    <div style={stack}>
      <label style={fieldLabel}>Justificativa da exceção
        <textarea value={text} rows={2} style={inputStyle} onChange={(e) => setText(e.target.value)} />
      </label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton onClick={save}>Registrar exceção</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </div>
  );
}
```

`src/modules/qc/QcBlockedPanel.tsx`:

```tsx
// 409 qc_blocked na liberação (contrato §3): nada foi liberado. Lista os itens
// presos com analito e aparelho; "Liberar sem os bloqueados" desmarca só eles;
// o RT/substituto registra a exceção item a item e libera de novo.
import { useState } from "react";
import { ApiError, type QcBlockedItem } from "../../lib/api";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { QcExceptionForm } from "./QcExceptionForm";
import { box, list, muted, row } from "../exams/styles";

export function blockedItems(err: unknown): QcBlockedItem[] {
  const items = err instanceof ApiError ? (err.body as { items?: unknown } | null)?.items : undefined;
  if (!Array.isArray(items)) return [];
  return items.filter((i): i is QcBlockedItem =>
    !!i && typeof i === "object" && typeof (i as QcBlockedItem).item_id === "string" && typeof (i as QcBlockedItem).block_id === "string");
}

export function QcBlockedPanel({ items, itemLabel, analyteLabel, excepted, lead, onSkip, onExcepted }: {
  items: QcBlockedItem[]; itemLabel(itemId: string): string; analyteLabel(itemId: string, key: string): string; excepted: string[];
  lead: boolean; onSkip(): void; onExcepted(itemId: string): void;
}) {
  const [ formFor, setFormFor ] = useState<string | null>(null);
  const pending = items.filter((b) => !excepted.includes(b.item_id));
  return (
    <div role="group" aria-label="bloqueado pelo controle de qualidade" style={box}>
      <p style={{ margin: 0 }}>{`O controle de qualidade reprovou e bloqueou ${items.length === 1 ? "1 exame" : `${items.length} exames`} desta requisição. Nada foi liberado.`}</p>
      <ul style={list}>
        {items.map((b) => (
          <li key={b.item_id}>
            <div style={row}>
              <span>{`${itemLabel(b.item_id)} · ${analyteLabel(b.item_id, b.analyte_key)} · ${b.instrument_name ?? b.instrument_id}`}</span>
              {excepted.includes(b.item_id) ? <Tag tone="ok">exceção registrada</Tag> : lead && formFor !== b.item_id && (
                <button type="button" style={secondaryButtonStyle} onClick={() => setFormFor(b.item_id)}>Liberar com exceção</button>
              )}
            </div>
            {formFor === b.item_id && (
              <QcExceptionForm blockId={b.block_id} itemId={b.item_id} onCancel={() => setFormFor(null)}
                onDone={() => { setFormFor(null); onExcepted(b.item_id); }} />
            )}
          </li>
        ))}
      </ul>
      {!lead && <p style={muted}>peça a exceção ao responsável técnico ou ao substituto, ou aguarde a corrida aprovada depois da ação corretiva</p>}
      {pending.length === 0 && <p style={muted}>exceções registradas — confira e libere de novo</p>}
      {pending.length > 0 && <div><button type="button" style={secondaryButtonStyle} onClick={onSkip}>Liberar sem os bloqueados</button></div>}
    </div>
  );
}
```

`src/modules/lab/ReleaseReview.tsx`:

```tsx
import { ReagentLotTag } from "../inventory/ReagentLotTag";
import { QcBlockedPanel, blockedItems } from "../qc/QcBlockedPanel";
import type { QcBlockedItem } from "../../lib/api";
```

Assinatura (o `qcLead` é opcional: as telas do 21b que não passam ficam iguais):

```tsx
export function ReleaseReview({ requisitionId, council, qcLead = false, onReleased, onConflict, onNavigate, onClose }: {
  requisitionId: string; council: string | null; qcLead?: boolean; onReleased(r: LabReportSummary): void; onConflict(text: string): void;
  onNavigate(id: ModuleId): void; onClose(): void;
}) {
```

Estado, junto dos demais `useState`:

```tsx
  // 21c: itens presos pelo CIQ no último envio e os que já têm exceção.
  const [ blocked, setBlocked ] = useState<QcBlockedItem[]>([]);
  const [ excepted, setExcepted ] = useState<string[]>([]);
```

Em `release`, logo depois de `setCertificateLink(false);`:

```tsx
    setBlocked([]);
```

e no `catch`, antes de `if (reason === "invalid_transition")`:

```tsx
      if (reason === "qc_blocked") { setBlocked(blockedItems(err)); setError(null); return; }
```

Em `itemBlock`, na linha do valor, depois da `Tag` da marca:

```tsx
                <ReagentLotTag row={x} />
```

E depois de `{r.items.map(itemBlock)}`:

```tsx
      {blocked.length > 0 && (
        <QcBlockedPanel items={blocked} excepted={excepted} lead={qcLead}
          itemLabel={(id) => r.items.find((i) => i.id === id)?.label ?? id}
          analyteLabel={(id, key) => r.items.find((i) => i.id === id)?.results.find((x) => x.analyte_key === key)?.name ?? key}
          onExcepted={(id) => setExcepted((x) => [ ...x, id ])}
          onSkip={() => {
            const stuck = blocked.filter((b) => !excepted.includes(b.item_id)).map((b) => b.item_id);
            setSelected(chosenIds.filter((id) => !stuck.includes(id)));
            setBlocked([]);
          }} />
      )}
```

`src/modules/LabRelease.tsx` — onde a conferência é montada:

```tsx
        <ReleaseReview key={openId} requisitionId={openId} council={council} qcLead={(context.data?.technical_lead_of ?? []).includes(providerId)}
          onNavigate={onNavigate} onClose={() => setOpenId(null)}
```

(o resto das props fica igual).

`src/modules/lab/ResultEntryForm.tsx` — import e, na lista de valores, depois de `{r.effective && <> <Tag tone="ok">vale</Tag></>}`:

```tsx
import { ReagentLotTag } from "../inventory/ReagentLotTag";
```

```tsx
                      {" "}<ReagentLotTag row={r} />
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabRelease.qc.test.tsx src/modules/lab/ResultEntryForm.lot.test.tsx src/modules/LabRelease.test.tsx src/modules/lab src/modules/LabEntry.test.tsx src/modules/LabWorklist.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes do 21b seguem verdes: sem `reagent_lot` nas fixtures, nada aparece; sem 409 `qc_blocked`, a liberação é a mesma).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/qc/QcBlockedPanel.tsx src/modules/qc/QcExceptionForm.tsx src/modules/lab/ReleaseReview.tsx src/modules/LabRelease.tsx src/modules/lab/ResultEntryForm.tsx src/modules/LabRelease.qc.test.tsx src/modules/lab/ResultEntryForm.lot.test.tsx
$G commit -m "feat: show quality control blocks in lab release with the technical lead exception and reagent lots

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: CEQ — rodadas, relatório anual e resumo do ano

**Files:**
- Create: `src/modules/LabEqa.tsx`, `src/modules/eqa/RoundForm.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabEqa.test.tsx`

**Interfaces:**
- Consumes: `listEqaRounds`, `createEqaRound`, `uploadEqaReport`, `fetchEqaReport`, `getEqaSummary`, `listLabExams` (Task 1); `EMPTY_EQA_ROUND`, `EMPTY_EQA_ITEM`, `suggestAdequacy`, `eqaProblem`, `eqaBody`, `adequacyRate`, `adequacyLabel`, `adequacyTone`, `analyteName`, `fmtDecimal`, `EQA_ROUNDS_KEY`, `EQA_SUMMARY_KEY`, `QC_EXAMS_KEY`, `QC_DISABLED`, `qcError` (Task 2); `openLabPdf` (21b); `LabGate`, `LabPick` (Task 5); `todayInCity`, `fmtDay`.
- Produces: `LabEqa()`; `RoundForm({ providerId, exams, onSaved, onCancel })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabEqa.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabExams: vi.fn(), listEqaRounds: vi.fn(), createEqaRound: vi.fn(),
    uploadEqaReport: vi.fn(), fetchEqaReport: vi.fn(), getEqaSummary: vi.fn() };
});

import * as api from "../lib/api";
import { LabEqa } from "./LabEqa";
import { renderWithProviders } from "../test/campaignFixtures";
import { GLU_EXAM, NOW21C, eqaRound, qcContext, qcUser } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listLabExams, api.listEqaRounds, api.createEqaRound, api.uploadEqaReport, api.fetchEqaReport, api.getEqaSummary ];
afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_releaser" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext({ releaser_of: [ "pv9" ], technical_lead_of: [ "pv9" ] }));
  mocked(api.listLabExams).mockResolvedValue([ GLU_EXAM ]);
  mocked(api.listEqaRounds).mockResolvedValue([ eqaRound({ annual_report_upload_id: "up0" }) ]);
  mocked(api.getEqaSummary).mockResolvedValue([ { analyte_key: "glicose", lab_exam_id: "le1", rounds: 3, adequate: 2, inadequate: 1 } ]);
  mocked(api.createEqaRound).mockResolvedValue(eqaRound({ id: "er2" }));
  mocked(api.uploadEqaReport).mockResolvedValue("up1");
});

async function newRound() {
  renderWithProviders(<LabEqa />);
  fireEvent.click(await screen.findByRole("button", { name: "Nova rodada" }));
  const form = screen.getByRole("region", { name: "Nova rodada do CEQ" });
  await within(form).findByRole("option", { name: "Glicose (Glicose)" });
  fireEvent.change(within(form).getByLabelText("Programa"), { target: { value: "PNCQ" } });
  fireEvent.change(within(form).getByLabelText("Código da rodada"), { target: { value: "2026-04" } });
  fireEvent.change(within(form).getByLabelText("Data da rodada"), { target: { value: "2026-10-10" } });
  const item = within(form).getByRole("group", { name: "Item 1" });
  fireEvent.change(within(item).getByLabelText("Analito"), { target: { value: "le1:glicose" } });
  fireEvent.change(within(item).getByLabelText("Enviado"), { target: { value: "107,0" } });
  fireEvent.change(within(item).getByLabelText("Alvo"), { target: { value: "100,0" } });
  fireEvent.change(within(item).getByLabelText("Limite inferior"), { target: { value: "94,0" } });
  fireEvent.change(within(item).getByLabelText("Limite superior"), { target: { value: "106,0" } });
  return { form, item };
}

describe("LabEqa", () => {
  it("adequação sugerida pelos limites; inadequado exige ação; corpo com decimais em ponto", async () => {
    const { form, item } = await newRound();
    expect((within(item).getByLabelText("Adequação") as HTMLSelectElement).value).toBe("inadequate");
    fireEvent.click(within(form).getByRole("button", { name: "Salvar rodada" }));
    expect(within(form).getByRole("alert").textContent).toBe("item 1: escreva a ação para o resultado inadequado (ao menos 10 caracteres)");
    expect(api.createEqaRound).not.toHaveBeenCalled();
    fireEvent.change(within(item).getByLabelText("Ação para o inadequado"), { target: { value: "recalibração e nova dosagem do controle" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar rodada" }));
    await waitFor(() => expect(api.createEqaRound).toHaveBeenCalledWith({
      provider_id: "pv9", program: "PNCQ", round_code: "2026-04", round_date: "2026-10-10", annual_report_upload_id: null,
      items: [ { lab_exam_id: "le1", analyte_key: "glicose", submitted: "107.0", target: "100.0", acceptable_low: "94.0", acceptable_high: "106.0",
        adequacy: "inadequate", action: "recalibração e nova dosagem do controle" } ]
    }));
    expect(await screen.findByText("Rodada registrada.")).not.toBeNull();
  });

  it("relatório anual: só PDF; o id do envio vai na rodada", async () => {
    const { form, item } = await newRound();
    fireEvent.change(within(item).getByLabelText("Ação para o inadequado"), { target: { value: "recalibração e nova dosagem do controle" } });
    const input = within(form).getByLabelText("Relatório anual (PDF, opcional)");
    fireEvent.change(input, { target: { files: [ new File([ "x" ], "planilha.xlsx", { type: "application/vnd.ms-excel" }) ] } });
    expect(within(form).getByRole("alert").textContent).toBe("o relatório anual precisa ser um PDF");
    expect(api.uploadEqaReport).not.toHaveBeenCalled();
    fireEvent.change(input, { target: { files: [ new File([ "%PDF" ], "relatorio.pdf", { type: "application/pdf" }) ] } });
    expect(await within(form).findByText("relatório anexado")).not.toBeNull();
    fireEvent.click(within(form).getByRole("button", { name: "Salvar rodada" }));
    await waitFor(() => expect(mocked(api.createEqaRound).mock.calls[0][0].annual_report_upload_id).toBe("up1"));
  });

  it("rodadas do ano, taxa de adequação por analito e o relatório numa janela nova", async () => {
    const win = { opener: {}, location: { href: "" }, close: vi.fn() };
    vi.spyOn(window, "open").mockReturnValue(win as unknown as Window);
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") });
    mocked(api.fetchEqaReport).mockResolvedValue({ blob: new Blob([ "%PDF" ]), filename: null });
    renderWithProviders(<LabEqa />);
    const rounds = await screen.findByRole("region", { name: "Rodadas" });
    expect(await within(rounds).findByText("1/1 adequados")).not.toBeNull();
    expect(api.listEqaRounds).toHaveBeenCalledWith("pv9", "2026");
    const summary = screen.getByRole("region", { name: "Adequação no ano" });
    expect(await within(summary).findByText("66,7%")).not.toBeNull();
    fireEvent.click(within(rounds).getByRole("button", { name: "Relatório anual" }));
    await waitFor(() => expect(api.fetchEqaReport).toHaveBeenCalledWith("er1"));
  });

  it("equipe sem papel de liberador não vê", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
    renderWithProviders(<LabEqa />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabEqa.test.tsx`
Expected: FAIL — `./LabEqa` não existe.

- [ ] **Step 3: Write minimal implementation**

`src/modules/eqa/RoundForm.tsx`:

```tsx
// Rodada do CEQ (spec §5.10; contrato §4): programa, código, data, por
// analito o enviado, o alvo, os limites aceitáveis, a adequação (sugerida
// pelos limites, a pessoa confirma) e a ação para o inadequado; o relatório
// anual em PDF é opcional. Decimais com vírgula.
import { useState } from "react";
import { createEqaRound, uploadEqaReport, type LabExamRef } from "../../lib/api";
import { EMPTY_EQA_ITEM, EMPTY_EQA_ROUND, eqaBody, eqaProblem, qcError, suggestAdequacy, type EqaItemDraft, type EqaRoundDraft } from "../../lib/qc";
import { todayInCity } from "../../lib/campaigns";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, fieldset, form, legend, muted, okText, row } from "../exams/styles";

const MAX_PDF_BYTES = 10 * 1024 * 1024;

export function RoundForm({ providerId, exams, onSaved, onCancel }: { providerId: string; exams: LabExamRef[]; onSaved(): void; onCancel(): void }) {
  const today = todayInCity();
  const [ draft, setDraft ] = useState<EqaRoundDraft>(EMPTY_EQA_ROUND);
  const [ error, setError ] = useState<string | null>(null);
  const [ uploading, setUploading ] = useState(false);
  const options = exams.filter((e) => e.id).flatMap((e) => (e.analytes ?? []).map((a) => ({ token: `${e.id}:${a.key}`, label: `${a.name} (${e.name})` })));
  const set = (patch: Partial<EqaRoundDraft>) => setDraft((d) => ({ ...d, ...patch }));

  function setItem(i: number, patch: Partial<EqaItemDraft>) {
    setDraft((d) => ({
      ...d,
      items: d.items.map((item, j) => {
        if (j !== i) return item;
        const next = { ...item, ...patch };
        // Sugestão só enquanto a pessoa não escolheu a adequação.
        if (!("adequacy" in patch) && item.adequacy === "") next.adequacy = suggestAdequacy(next);
        return next;
      })
    }));
  }

  async function attach(file: File | undefined) {
    if (!file) return;
    if (file.type !== "application/pdf") { setError("o relatório anual precisa ser um PDF"); return; }
    if (file.size > MAX_PDF_BYTES) { setError("o relatório anual pode ter no máximo 10 MB"); return; }
    setError(null);
    setUploading(true);
    try {
      set({ uploadId: await uploadEqaReport(file) });
    } catch (err) {
      setError(qcError(err));
    } finally {
      setUploading(false);
    }
  }

  async function save() {
    const problem = eqaProblem(draft, today);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await createEqaRound(eqaBody(draft, providerId));
      onSaved();
    } catch (err) {
      setError(qcError(err));
    }
  }

  return (
    <section aria-label="Nova rodada do CEQ" style={box}>
      <div style={form}>
        <div style={row}>
          <label style={fieldLabel}>Programa
            <select value={draft.program} style={inputStyle} onChange={(e) => set({ program: e.target.value as EqaRoundDraft["program"] })}>
              <option value="">—</option>
              <option value="PNCQ">PNCQ</option>
              <option value="Controllab">Controllab</option>
              <option value="outro">outro</option>
            </select>
          </label>
          {draft.program === "outro" && (
            <label style={fieldLabel}>Nome do programa<input value={draft.programOther} style={inputStyle} onChange={(e) => set({ programOther: e.target.value })} /></label>
          )}
          <label style={fieldLabel}>Código da rodada<input value={draft.roundCode} style={inputStyle} onChange={(e) => set({ roundCode: e.target.value })} /></label>
          <label style={fieldLabel}>Data da rodada<input type="date" value={draft.roundDate} max={today} style={inputStyle} onChange={(e) => set({ roundDate: e.target.value })} /></label>
        </div>
        {draft.items.map((item, i) => (
          <fieldset key={i} role="group" aria-label={`Item ${i + 1}`} style={fieldset}>
            <legend style={legend}>{`Item ${i + 1}`}</legend>
            <div style={row}>
              <label style={fieldLabel}>Analito
                <select value={item.token} style={inputStyle} onChange={(e) => setItem(i, { token: e.target.value })}>
                  <option value="">—</option>
                  {options.map((o) => <option key={o.token} value={o.token}>{o.label}</option>)}
                </select>
              </label>
              <label style={fieldLabel}>Enviado<input value={item.submitted} inputMode="decimal" style={inputStyle} onChange={(e) => setItem(i, { submitted: e.target.value })} /></label>
              <label style={fieldLabel}>Alvo<input value={item.target} inputMode="decimal" style={inputStyle} onChange={(e) => setItem(i, { target: e.target.value })} /></label>
              <label style={fieldLabel}>Limite inferior<input value={item.low} inputMode="decimal" style={inputStyle} onChange={(e) => setItem(i, { low: e.target.value })} /></label>
              <label style={fieldLabel}>Limite superior<input value={item.high} inputMode="decimal" style={inputStyle} onChange={(e) => setItem(i, { high: e.target.value })} /></label>
              <label style={fieldLabel}>Adequação
                <select value={item.adequacy} style={inputStyle} onChange={(e) => setItem(i, { adequacy: e.target.value as EqaItemDraft["adequacy"] })}>
                  <option value="">—</option>
                  <option value="adequate">adequado</option>
                  <option value="inadequate">inadequado</option>
                </select>
              </label>
            </div>
            {item.adequacy === "inadequate" && (
              <label style={fieldLabel}>Ação para o inadequado
                <textarea value={item.action} rows={2} style={inputStyle} onChange={(e) => setItem(i, { action: e.target.value })} />
              </label>
            )}
            {draft.items.length > 1 && (
              <div><button type="button" style={secondaryButtonStyle} onClick={() => setDraft((d) => ({ ...d, items: d.items.filter((_, j) => j !== i) }))}>Tirar item</button></div>
            )}
          </fieldset>
        ))}
        <div><button type="button" style={secondaryButtonStyle} onClick={() => setDraft((d) => ({ ...d, items: [ ...d.items, EMPTY_EQA_ITEM ] }))}>Incluir analito</button></div>
        <label style={fieldLabel}>Relatório anual (PDF, opcional)
          <input type="file" accept="application/pdf" style={inputStyle} onChange={(e) => void attach(e.target.files?.[0])} />
        </label>
        {uploading && <p className="mono" style={muted}>enviando…</p>}
        {draft.uploadId && <p role="status" style={okText}>relatório anexado</p>}
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton busy={uploading} onClick={save}>Salvar rodada</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}
```

`src/modules/LabEqa.tsx`:

```tsx
// src/modules/LabEqa.tsx
// Controle externo da qualidade (spec §5.10; contrato §4): rodadas do ano com
// a adequação por item e o relatório anual; taxa de adequação por analito.
// Do liberador e do admin (spec §8). Sem dado de paciente.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { fetchEqaReport, getEqaSummary, listEqaRounds, listLabExams, type EqaRound, type EqaSummaryRow } from "../lib/api";
import { EQA_ROUNDS_KEY, EQA_SUMMARY_KEY, QC_DISABLED, QC_EXAMS_KEY, adequacyRate, analyteName, qcError } from "../lib/qc";
import { todayInCity } from "../lib/campaigns";
import { fmtDay } from "../lib/exams";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { openLabPdf } from "./lab/openLabPdf";
import { RoundForm } from "./eqa/RoundForm";
import { alert, fieldLabel, okText, stack } from "./exams/styles";

export function LabEqa() {
  return (
    <LabGate title="Controle externo (CEQ)" sub="CEQ · rodadas, adequação por analito e relatório anual"
      allow={(r) => r.admin || r.releaser} requires={[ "lab_quality_control" ]} disabledText={QC_DISABLED}>
      {() => <LabPick>{(lab) => <EqaBody providerId={lab.id} />}</LabPick>}
    </LabGate>
  );
}

function EqaBody({ providerId }: { providerId: string }) {
  const queryClient = useQueryClient();
  const [ year, setYear ] = useState(() => todayInCity().slice(0, 4));
  const valid = /^\d{4}$/.test(year);
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const rounds = useQuery({ queryKey: [ EQA_ROUNDS_KEY, providerId, year ], queryFn: () => listEqaRounds(providerId, year), enabled: valid, retry: false });
  const summary = useQuery({ queryKey: [ EQA_SUMMARY_KEY, providerId, year ], queryFn: () => getEqaSummary(providerId, year), enabled: valid, retry: false });
  const [ creating, setCreating ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const examList = exams.data ?? [];

  async function openReport(id: string) {
    const out = await openLabPdf(() => fetchEqaReport(id));
    setError(out.ok ? null : out.text);
  }

  return (
    <div style={stack}>
      <label style={fieldLabel}>Ano<input value={year} inputMode="numeric" style={{ ...inputStyle, maxWidth: 120 }} onChange={(e) => setYear(e.target.value.trim())} /></label>
      {!valid && <p role="alert" style={alert}>ano com 4 dígitos</p>}
      {notice && <p role="status" style={okText}>{notice}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}
      <Panel title="Rodadas" sub="uma linha por rodada do programa de proficiência"
        right={<button type="button" style={secondaryButtonStyle} onClick={() => { setCreating(true); setNotice(null); }}>Nova rodada</button>}>
        <div style={stack}>
          {rounds.isError ? <p role="alert" style={alert}>{qcError(rounds.error)}</p> : (
            <DataTable<EqaRound>
              cols={[
                { label: "Data", w: "0.8fr", card: "title", render: (r) => fmtDay(r.round_date) },
                { label: "Programa · rodada", w: "1.3fr", render: (r) => `${r.program} · ${r.round_code}` },
                { label: "Adequação", w: "1fr", render: (r) => {
                  const ok = r.items.filter((i) => i.adequacy === "adequate").length;
                  return <>{`${ok}/${r.items.length} adequados`}{ok < r.items.length && <> <Tag tone="down">inadequado</Tag></>}</>;
                } },
                { label: "", w: "auto", card: "actions", render: (r) => (r.annual_report_upload_id
                  ? <button type="button" style={secondaryButtonStyle} onClick={() => void openReport(r.id)}>Relatório anual</button> : null) }
              ]}
              rows={rounds.data ?? []} rowKey={(r) => r.id} empty={rounds.isPending ? "carregando…" : "nenhuma rodada neste ano"}
            />
          )}
          {creating && (
            <RoundForm providerId={providerId} exams={examList} onCancel={() => setCreating(false)}
              onSaved={() => {
                setCreating(false);
                setNotice("Rodada registrada.");
                void queryClient.invalidateQueries({ queryKey: [ EQA_ROUNDS_KEY ] });
                void queryClient.invalidateQueries({ queryKey: [ EQA_SUMMARY_KEY ] });
              }} />
          )}
        </div>
      </Panel>
      <Panel title="Adequação no ano" sub="por analito, somando as rodadas do ano">
        {summary.isError ? <p role="alert" style={alert}>{qcError(summary.error)}</p> : (
          <DataTable<EqaSummaryRow>
            cols={[
              { label: "Analito", w: "1.6fr", card: "title", render: (s) => analyteName(examList, s.lab_exam_id, s.analyte_key) },
              { label: "Rodadas", w: "0.6fr", align: "right", render: (s) => String(s.rounds) },
              { label: "Adequados", w: "0.7fr", align: "right", render: (s) => String(s.adequate) },
              { label: "Inadequados", w: "0.7fr", align: "right", render: (s) => String(s.inadequate) },
              { label: "Taxa", w: "0.6fr", align: "right", render: (s) => adequacyRate(s) }
            ]}
            rows={summary.data ?? []} rowKey={(s) => `${s.lab_exam_id}:${s.analyte_key}`} empty={summary.isPending ? "carregando…" : "nenhuma rodada neste ano"}
          />
        )}
      </Panel>
    </div>
  );
}
```

`src/App.tsx`:

```tsx
import { LabEqa } from "./modules/LabEqa";
```

```tsx
    case "lab-eqa": return <LabEqa />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabEqa.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabEqa.tsx src/modules/LabEqa.test.tsx src/modules/eqa/RoundForm.tsx src/App.tsx
$G commit -m "feat: add external quality assessment rounds with the annual report and adequacy summary

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Estoque — itens, entrada de lote e movimentos

**Files:**
- Create: `src/modules/LabInventory.tsx`, `src/modules/inventory/SuppliesPanel.tsx`, `src/modules/inventory/SupplyLotsPanel.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabInventory.test.tsx`

**Interfaces:**
- Consumes: `listSupplies`, `createSupply`, `updateSupply`, `listSupplyLots`, `receiveSupplyLot`, `listSupplyMovements`, `recordSupplyMovement`, `listLabExams` (Task 1); `EMPTY_SUPPLY`, `supplyDraft`, `supplyProblem`, `supplyBody`, `EMPTY_LOT_ENTRY`, `lotEntryProblem`, `lotEntryBody`, `EMPTY_MOVEMENT`, `movementProblem`, `movementBody`, `SUPPLY_KIND_LABEL`, `supplyKindLabel`, `SUPPLY_ORIGIN_LABEL`, `originLabel`, `movementLabel`, `INV_*` chaves, `INVENTORY_DISABLED`, `inventoryError`, `refreshInventory` (Task 3); `QC_EXAMS_KEY` (Task 2); `errorCode`; `LabGate`, `LabPick` (Task 5); `todayInCity`, `fmtDay`, `fmtDateTime`.
- Produces: `LabInventory()`; `InventoryBody({ providerId, admin })` (interna; a Task 15 acrescenta lote em uso e alertas); `SuppliesPanel({ providerId, admin, exams })`; `SupplyLotsPanel({ providerId, admin })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabInventory.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabExams: vi.fn(), listSupplies: vi.fn(), createSupply: vi.fn(),
    updateSupply: vi.fn(), listSupplyLots: vi.fn(), receiveSupplyLot: vi.fn(), listSupplyMovements: vi.fn(), recordSupplyMovement: vi.fn(),
    listLabInstruments: vi.fn(), listInUse: vi.fn(), setInUse: vi.fn(), getInventoryAlerts: vi.fn() };
});

import * as api from "../lib/api";
import { LabInventory } from "./LabInventory";
import { INVENTORY_DISABLED } from "../lib/inventory";
import { renderWithProviders } from "../test/campaignFixtures";
import { GLU_EXAM, NOW21C, instrument, qcContext, qcUser, supply, supplyLot, supplyMovement } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listLabExams, api.listSupplies, api.createSupply, api.updateSupply, api.listSupplyLots,
  api.receiveSupplyLot, api.listSupplyMovements, api.recordSupplyMovement, api.listLabInstruments, api.listInUse, api.setInUse, api.getInventoryAlerts ];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext());
  mocked(api.listLabExams).mockResolvedValue([ GLU_EXAM ]);
  mocked(api.listSupplies).mockResolvedValue([ supply() ]);
  mocked(api.createSupply).mockResolvedValue(supply({ id: "su2" }));
  mocked(api.listSupplyLots).mockResolvedValue([ supplyLot() ]);
  mocked(api.receiveSupplyLot).mockResolvedValue(supplyLot({ id: "sl2", lot_number: "R-901" }));
  mocked(api.listSupplyMovements).mockResolvedValue([ supplyMovement() ]);
  mocked(api.recordSupplyMovement).mockResolvedValue(supplyMovement({ id: "mv2", kind: "opening", quantity: -1 }));
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.listInUse).mockResolvedValue([]);
  mocked(api.getInventoryAlerts).mockResolvedValue({ expiring: [], below_minimum: [], untraced_results: 0 });
});

// Leituras dos lotes do item escolhido (o painel de lote em uso, da Task 15, lê os do laboratório inteiro).
const lotReads = (supplyId: string) => mocked(api.listSupplyLots).mock.calls.filter(([ q ]) => (q as { supplyId?: string }).supplyId === supplyId).length;

async function lotsPanel() {
  renderWithProviders(<LabInventory />);
  const panel = await screen.findByRole("region", { name: "Lotes e movimentos" });
  await within(panel).findByRole("option", { name: "Glicose GOD-PAP 2×100 mL" });
  fireEvent.change(within(panel).getByLabelText("Item"), { target: { value: "su1" } });
  await within(panel).findByText("R-778");
  return panel;
}

async function movementForm() {
  const panel = await lotsPanel();
  fireEvent.click(within(panel).getByRole("button", { name: "Movimentos" }));
  const box = within(panel).getByRole("group", { name: "Movimentos do lote R-778" });
  await within(box).findByText("NF 1234");
  return box;
}

describe("LabInventory — itens, lotes e movimentos", () => {
  it("admin cadastra reagente: rendimento e exame obrigatórios", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "municipal_admin" ]));
    renderWithProviders(<LabInventory />);
    const panel = await screen.findByRole("region", { name: "Itens de estoque" });
    fireEvent.click(within(panel).getByRole("button", { name: "Novo item" }));
    const form = within(panel).getByRole("group", { name: "Novo item" });
    fireEvent.change(within(form).getByLabelText("Nome"), { target: { value: "Colesterol CHOD-PAP" } });
    fireEvent.change(within(form).getByLabelText("Tipo"), { target: { value: "reagent" } });
    fireEvent.change(within(form).getByLabelText("Unidade"), { target: { value: "kit" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar item" }));
    expect(within(form).getByRole("alert").textContent).toBe("informe quantos testes rende cada embalagem do reagente");
    fireEvent.change(within(form).getByLabelText("Testes por embalagem"), { target: { value: "200" } });
    fireEvent.click(within(form).getByLabelText("Glicose"));
    fireEvent.click(within(form).getByRole("button", { name: "Salvar item" }));
    await waitFor(() => expect(api.createSupply).toHaveBeenCalledWith({
      provider_id: "pv9", name: "Colesterol CHOD-PAP", kind: "reagent", unit: "kit", tests_per_package: 200, minimum_stock: 0,
      expiry_alert_days: 30, lab_exam_ids: [ "le1" ], active: true
    }));
  });

  it("equipe não cadastra item; entrada de lote vencido é recusada; a válida grava e relê", async () => {
    const panel = await lotsPanel();
    expect(screen.queryByRole("button", { name: "Novo item" })).toBeNull();
    const form = within(panel).getByRole("group", { name: "Entrada de lote" });
    fireEvent.change(within(form).getByLabelText("Número do lote"), { target: { value: "R-901" } });
    fireEvent.change(within(form).getByLabelText("Validade"), { target: { value: "2026-10-01" } });
    fireEvent.change(within(form).getByLabelText("Fornecedor"), { target: { value: "Distribuidora Sul" } });
    fireEvent.change(within(form).getByLabelText("Origem"), { target: { value: "donation" } });
    fireEvent.change(within(form).getByLabelText("Quantidade"), { target: { value: "4" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar entrada" }));
    expect(within(form).getByRole("alert").textContent).toBe("lote vencido — não entra no estoque");
    fireEvent.change(within(form).getByLabelText("Validade"), { target: { value: "2027-05-31" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar entrada" }));
    await waitFor(() => expect(api.receiveSupplyLot).toHaveBeenCalledWith({
      supply_id: "su1", lot_number: "R-901", expires_on: "2027-05-31", supplier: "Distribuidora Sul", origin: "donation", quantity: 4
    }));
    expect(await within(panel).findByText("Entrada registrada.")).not.toBeNull();
    await waitFor(() => expect(lotReads("su1")).toBe(2));
  });

  it("movimento: saída maior que o saldo bloqueada; ajuste só para o admin; abertura grava", async () => {
    const box = await movementForm();
    const kinds = [ ...(within(box).getByLabelText("Movimento") as HTMLSelectElement).options ].map((o) => o.value);
    expect(kinds).toEqual([ "", "opening", "loss", "expiry" ]);
    fireEvent.change(within(box).getByLabelText("Movimento"), { target: { value: "loss" } });
    fireEvent.change(within(box).getByLabelText("Quantidade"), { target: { value: "6" } });
    fireEvent.change(within(box).getByLabelText("Motivo"), { target: { value: "frasco quebrado" } });
    fireEvent.click(within(box).getByRole("button", { name: "Registrar movimento" }));
    expect(within(box).getByRole("alert").textContent).toBe("saída maior que o saldo do lote (5)");
    expect(api.recordSupplyMovement).not.toHaveBeenCalled();
    fireEvent.change(within(box).getByLabelText("Movimento"), { target: { value: "opening" } });
    fireEvent.change(within(box).getByLabelText("Quantidade"), { target: { value: "1" } });
    fireEvent.click(within(box).getByRole("button", { name: "Registrar movimento" }));
    await waitFor(() => expect(api.recordSupplyMovement).toHaveBeenCalledWith({ lot_id: "sl1", kind: "opening", quantity: 1, reason: "frasco quebrado" }));
  });

  it("422 insufficient_balance: relê os lotes e mantém o formulário", async () => {
    mocked(api.recordSupplyMovement).mockRejectedValueOnce(new api.ApiError(422, { error: "insufficient_balance" }, "x"));
    const box = await movementForm();
    fireEvent.change(within(box).getByLabelText("Movimento"), { target: { value: "opening" } });
    fireEvent.change(within(box).getByLabelText("Quantidade"), { target: { value: "2" } });
    fireEvent.click(within(box).getByRole("button", { name: "Registrar movimento" }));
    expect((await within(box).findByRole("alert")).textContent).toBe("saída maior que o saldo do lote — o saldo foi relido");
    await waitFor(() => expect(lotReads("su1")).toBe(2));
    expect((within(box).getByLabelText("Quantidade") as HTMLInputElement).value).toBe("2");
  });

  it("admin vê o ajuste (com sinal)", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "municipal_admin" ]));
    const box = await movementForm();
    expect([ ...(within(box).getByLabelText("Movimento") as HTMLSelectElement).options ].map((o) => o.value)).toContain("adjustment");
  });

  it("estoque desligado: a frase", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ], { features: [ "exam_tracking", "municipal_lab", "lab_quality_control" ] }));
    renderWithProviders(<LabInventory />);
    expect(await screen.findByText(INVENTORY_DISABLED)).not.toBeNull();
  });
});
```

(Os mocks de aparelhos, lote em uso e alertas entram aqui porque a Task 15 acrescenta esses painéis à mesma tela.)

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabInventory.test.tsx`
Expected: FAIL — `./LabInventory` não existe.

- [ ] **Step 3: Write minimal implementation**

`src/modules/LabInventory.tsx`:

```tsx
// src/modules/LabInventory.tsx
// Estoque do laboratório (spec §6; contrato §5): itens (admin), entrada de
// lote, movimentos (abertura, perda, vencimento; ajuste só admin), lote em uso
// por aparelho e alertas. Nada é baixado sozinho. Sem polling.
import { useQuery } from "@tanstack/react-query";
import { listLabExams } from "../lib/api";
import { INVENTORY_DISABLED } from "../lib/inventory";
import { QC_EXAMS_KEY } from "../lib/qc";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { SuppliesPanel } from "./inventory/SuppliesPanel";
import { SupplyLotsPanel } from "./inventory/SupplyLotsPanel";
import { stack } from "./exams/styles";

export function LabInventory() {
  return (
    <LabGate title="Estoque do laboratório" sub="insumos · itens, lotes, movimentos e lote em uso"
      allow={(r) => r.admin || r.staff || r.releaser} requires={[ "lab_inventory" ]} disabledText={INVENTORY_DISABLED}>
      {(roles) => <LabPick>{(lab) => <InventoryBody providerId={lab.id} admin={roles.admin} />}</LabPick>}
    </LabGate>
  );
}

function InventoryBody({ providerId, admin }: { providerId: string; admin: boolean }) {
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  return (
    <div style={stack}>
      <SuppliesPanel providerId={providerId} admin={admin} exams={exams.data ?? []} />
      <SupplyLotsPanel providerId={providerId} admin={admin} />
    </div>
  );
}
```

`src/modules/inventory/SuppliesPanel.tsx`:

```tsx
// Itens de estoque (spec §6.1; contrato §5): unidade, mínimo, rendimento em
// testes (reagente), aviso de vencimento e exames ligados. Só o admin
// cadastra; todos do laboratório veem o saldo.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createSupply, listSupplies, updateSupply, type LabExamRef, type LabSupply } from "../../lib/api";
import {
  EMPTY_SUPPLY, INV_SUPPLIES_KEY, SUPPLY_KIND_LABEL, inventoryError, refreshInventory, supplyBody, supplyDraft, supplyKindLabel, supplyProblem,
  type SupplyDraft
} from "../../lib/inventory";
import { Panel } from "../../components/Panel";
import { DataTable, type Column } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, checkLabel, fieldLabel, fieldset, legend, okText, row, stack } from "../exams/styles";

export function SuppliesPanel({ providerId, admin, exams }: { providerId: string; admin: boolean; exams: LabExamRef[] }) {
  const queryClient = useQueryClient();
  const supplies = useQuery({ queryKey: [ INV_SUPPLIES_KEY, providerId ], queryFn: () => listSupplies(providerId), retry: false });
  const [ editing, setEditing ] = useState<LabSupply | "new" | null>(null);
  const [ draft, setDraft ] = useState<SupplyDraft>(EMPTY_SUPPLY);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const set = (patch: Partial<SupplyDraft>) => setDraft((d) => ({ ...d, ...patch }));
  const examName = (id: string) => exams.find((e) => e.id === id)?.name ?? id;

  function open(target: LabSupply | "new") {
    setEditing(target);
    setDraft(target === "new" ? EMPTY_SUPPLY : supplyDraft(target));
    setError(null); setNotice(null);
  }

  async function save() {
    const problem = supplyProblem(draft);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      const body = supplyBody(draft, providerId);
      if (editing === "new" || editing === null) await createSupply(body); else await updateSupply(editing.id, body);
      setEditing(null);
      setNotice("Item salvo.");
      void refreshInventory(queryClient).catch(() => undefined);
    } catch (err) {
      setError(inventoryError(err));
    }
  }

  const cols: Column<LabSupply>[] = [
    { label: "Item", w: "1.6fr", card: "title", render: (s) => s.name },
    { label: "Tipo", w: "0.8fr", render: (s) => supplyKindLabel(s.kind) },
    { label: "Saldo", w: "1fr", render: (s) => <>{`${s.balance} ${s.unit}`}{s.balance < s.minimum_stock && <> <Tag tone="warn">abaixo do mínimo</Tag></>}</> },
    { label: "Mínimo", w: "0.6fr", align: "right", render: (s) => String(s.minimum_stock) },
    { label: "Rendimento", w: "0.8fr", render: (s) => (s.tests_per_package ? `${s.tests_per_package} testes` : "—") },
    { label: "Exames", w: "1.4fr", render: (s) => s.lab_exam_ids.map(examName).join(", ") || "—" }
  ];
  if (admin) cols.push({ label: "", w: "auto", card: "actions", render: (s) => <button type="button" style={secondaryButtonStyle} onClick={() => open(s)}>Editar</button> });

  return (
    <Panel title="Itens de estoque" sub="reagentes, calibradores, controles e consumíveis do laboratório"
      right={admin ? <button type="button" style={secondaryButtonStyle} onClick={() => open("new")}>Novo item</button> : undefined}>
      <div style={stack}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {supplies.isError ? <p role="alert" style={alert}>{inventoryError(supplies.error)}</p> : (
          <DataTable<LabSupply> cols={cols} rows={supplies.data ?? []} rowKey={(s) => s.id} empty={supplies.isPending ? "carregando…" : "nenhum item cadastrado"} />
        )}
        {admin && editing && (
          <fieldset role="group" aria-label={editing === "new" ? "Novo item" : `Editar ${editing.name}`} style={fieldset}>
            <legend style={legend}>{editing === "new" ? "Novo item" : `Editar ${editing.name}`}</legend>
            <div style={row}>
              <label style={fieldLabel}>Nome<input value={draft.name} style={inputStyle} onChange={(e) => set({ name: e.target.value })} /></label>
              <label style={fieldLabel}>Tipo
                <select value={draft.kind} style={inputStyle} onChange={(e) => set({ kind: e.target.value as SupplyDraft["kind"] })}>
                  <option value="">—</option>
                  {Object.entries(SUPPLY_KIND_LABEL).map(([ k, v ]) => <option key={k} value={k}>{v}</option>)}
                </select>
              </label>
              <label style={fieldLabel}>Unidade<input value={draft.unit} placeholder="kit, frasco, caixa" style={inputStyle} onChange={(e) => set({ unit: e.target.value })} /></label>
              {draft.kind === "reagent" && (
                <label style={fieldLabel}>Testes por embalagem<input value={draft.testsPerPackage} inputMode="numeric" style={inputStyle} onChange={(e) => set({ testsPerPackage: e.target.value })} /></label>
              )}
              <label style={fieldLabel}>Estoque mínimo<input value={draft.minimumStock} inputMode="numeric" style={inputStyle} onChange={(e) => set({ minimumStock: e.target.value })} /></label>
              <label style={fieldLabel}>Aviso de vencimento (dias)<input value={draft.expiryAlertDays} inputMode="numeric" style={inputStyle} onChange={(e) => set({ expiryAlertDays: e.target.value })} /></label>
            </div>
            <div style={row}>
              {exams.filter((e) => e.id).map((e) => (
                <label key={e.id} style={checkLabel}>
                  <input type="checkbox" checked={draft.examIds.includes(e.id as string)}
                    onChange={(ev) => set({ examIds: ev.target.checked ? [ ...draft.examIds, e.id as string ] : draft.examIds.filter((x) => x !== e.id) })} />{e.name}
                </label>
              ))}
            </div>
            <label style={checkLabel}><input type="checkbox" checked={draft.active} onChange={(e) => set({ active: e.target.checked })} />Ativo</label>
            {error && <p role="alert" style={alert}>{error}</p>}
            <div style={row}>
              <ActionButton onClick={save}>Salvar item</ActionButton>
              <button type="button" style={secondaryButtonStyle} onClick={() => setEditing(null)}>Cancelar</button>
            </div>
          </fieldset>
        )}
      </div>
    </Panel>
  );
}
```

`src/modules/inventory/SupplyLotsPanel.tsx`:

```tsx
// Lotes e movimentos (spec §6.2–§6.5; contrato §5): entrada (lote vencido
// recusado), saldo por lote, movimentos e o formulário de abertura, perda,
// vencimento e ajuste (só admin, com sinal). Saída maior que o saldo é
// recusada antes; se o api recusar (saldo mudou), relê e mantém o formulário.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, listSupplies, listSupplyLots, listSupplyMovements, receiveSupplyLot, recordSupplyMovement, type LabSupplyLot, type LabSupplyMovement } from "../../lib/api";
import {
  EMPTY_LOT_ENTRY, EMPTY_MOVEMENT, INV_LOTS_KEY, INV_MOVEMENTS_KEY, INV_SUPPLIES_KEY, SUPPLY_ORIGIN_LABEL, inventoryError, lotEntryBody,
  lotEntryProblem, movementBody, movementLabel, movementProblem, originLabel, refreshInventory, type LotEntryDraft, type MovementDraft
} from "../../lib/inventory";
import { todayInCity } from "../../lib/campaigns";
import { fmtDay } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, fieldset, legend, muted, okText, row, stack } from "../exams/styles";

const signed = (q: number) => (q > 0 ? `+${q}` : String(q));

export function SupplyLotsPanel({ providerId, admin }: { providerId: string; admin: boolean }) {
  const queryClient = useQueryClient();
  const today = todayInCity();
  const supplies = useQuery({ queryKey: [ INV_SUPPLIES_KEY, providerId ], queryFn: () => listSupplies(providerId), retry: false });
  const [ supplyId, setSupplyId ] = useState("");
  const lots = useQuery({ queryKey: [ INV_LOTS_KEY, supplyId ], queryFn: () => listSupplyLots({ supplyId }), enabled: supplyId !== "", retry: false });
  const [ entry, setEntry ] = useState<LotEntryDraft>(EMPTY_LOT_ENTRY);
  const [ entryError, setEntryError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ lotId, setLotId ] = useState<string | null>(null);
  const lot = (lots.data ?? []).find((l) => l.id === lotId) ?? null;
  const supply = (supplies.data ?? []).find((s) => s.id === supplyId);

  async function receive() {
    const draft = { ...entry, supplyId };
    const problem = lotEntryProblem(draft, today);
    if (problem) { setEntryError(problem); return; }
    setEntryError(null);
    try {
      await receiveSupplyLot(lotEntryBody(draft));
      setEntry(EMPTY_LOT_ENTRY);
      setNotice("Entrada registrada.");
      void refreshInventory(queryClient).catch(() => undefined);
    } catch (err) {
      setEntryError(inventoryError(err));
    }
  }

  const setE = (patch: Partial<LotEntryDraft>) => setEntry((d) => ({ ...d, ...patch }));

  return (
    <Panel title="Lotes e movimentos" sub="saldo = soma dos movimentos; nada é baixado sozinho">
      <div style={stack}>
        <label style={fieldLabel}>Item
          <select value={supplyId} style={inputStyle} onChange={(e) => { setSupplyId(e.target.value); setLotId(null); setNotice(null); }}>
            <option value="">—</option>
            {(supplies.data ?? []).filter((s) => s.active).map((s) => <option key={s.id} value={s.id}>{s.name}</option>)}
          </select>
        </label>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {supplyId && (lots.isError ? <p role="alert" style={alert}>{inventoryError(lots.error)}</p> : (
          <DataTable<LabSupplyLot>
            cols={[
              { label: "Lote", w: "1fr", card: "title", render: (l) => <span className="mono">{l.lot_number}</span> },
              { label: "Validade", w: "1fr", render: (l) => <>{fmtDay(l.expires_on)}{l.expires_on < today && <> <Tag tone="down">vencido</Tag></>}</> },
              { label: "Fornecedor · origem", w: "1.4fr", render: (l) => `${l.supplier} · ${originLabel(l.origin)}` },
              { label: "Saldo", w: "0.7fr", align: "right", render: (l) => `${l.balance}${supply ? ` ${supply.unit}` : ""}` },
              { label: "", w: "auto", card: "actions", render: (l) => (
                <button type="button" style={secondaryButtonStyle} aria-pressed={lotId === l.id} onClick={() => setLotId(l.id)}>Movimentos</button>
              ) }
            ]}
            rows={lots.data ?? []} rowKey={(l) => l.id} empty={lots.isPending ? "carregando…" : "nenhum lote com saldo"}
          />
        ))}
        {supplyId && (
          <fieldset role="group" aria-label="Entrada de lote" style={fieldset}>
            <legend style={legend}>Entrada de lote</legend>
            <div style={row}>
              <label style={fieldLabel}>Número do lote<input value={entry.lotNumber} style={inputStyle} onChange={(e) => setE({ lotNumber: e.target.value })} /></label>
              <label style={fieldLabel}>Validade<input type="date" value={entry.expiresOn} style={inputStyle} onChange={(e) => setE({ expiresOn: e.target.value })} /></label>
              <label style={fieldLabel}>Fornecedor<input value={entry.supplier} style={inputStyle} onChange={(e) => setE({ supplier: e.target.value })} /></label>
              <label style={fieldLabel}>Origem
                <select value={entry.origin} style={inputStyle} onChange={(e) => setE({ origin: e.target.value as LotEntryDraft["origin"] })}>
                  <option value="">—</option>
                  {Object.entries(SUPPLY_ORIGIN_LABEL).map(([ k, v ]) => <option key={k} value={k}>{v}</option>)}
                </select>
              </label>
              <label style={fieldLabel}>Quantidade<input value={entry.quantity} inputMode="numeric" style={inputStyle} onChange={(e) => setE({ quantity: e.target.value })} /></label>
              <label style={fieldLabel}>Documento (opcional)<input value={entry.document} style={inputStyle} onChange={(e) => setE({ document: e.target.value })} /></label>
            </div>
            {entryError && <p role="alert" style={alert}>{entryError}</p>}
            <div><ActionButton onClick={receive}>Registrar entrada</ActionButton></div>
          </fieldset>
        )}
        {lot && <LotMovements key={lot.id} lot={lot} admin={admin} onChanged={() => void refreshInventory(queryClient).catch(() => undefined)}
          onStale={() => void lots.refetch()} />}
        {!supplyId && <p style={muted}>escolha o item para ver os lotes e registrar entradas</p>}
      </div>
    </Panel>
  );
}

function LotMovements({ lot, admin, onChanged, onStale }: { lot: LabSupplyLot; admin: boolean; onChanged(): void; onStale(): void }) {
  const movements = useQuery({ queryKey: [ INV_MOVEMENTS_KEY, lot.id ], queryFn: () => listSupplyMovements(lot.id), retry: false });
  const [ draft, setDraft ] = useState<MovementDraft>(EMPTY_MOVEMENT);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const set = (patch: Partial<MovementDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function save() {
    const problem = movementProblem(draft, lot.balance, admin);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await recordSupplyMovement(movementBody(draft, lot.id));
      setDraft(EMPTY_MOVEMENT);
      setNotice("Movimento registrado.");
      onChanged();
    } catch (err) {
      setError(inventoryError(err));
      if (errorCode(err) === "insufficient_balance") onStale();
    }
  }

  return (
    <fieldset role="group" aria-label={`Movimentos do lote ${lot.lot_number}`} style={fieldset}>
      <legend style={legend}>{`Movimentos do lote ${lot.lot_number} · saldo ${lot.balance}`}</legend>
      {movements.isError ? <p role="alert" style={alert}>{inventoryError(movements.error)}</p> : (
        <DataTable<LabSupplyMovement>
          cols={[
            { label: "Quando", w: "1fr", card: "title", render: (m) => fmtDateTime(m.recorded_at) },
            { label: "Movimento", w: "0.8fr", render: (m) => movementLabel(m.kind) },
            { label: "Quantidade", w: "0.7fr", align: "right", render: (m) => signed(m.quantity) },
            { label: "Motivo · documento", w: "1.6fr", render: (m) => [ m.reason, m.document ].filter(Boolean).join(" · ") || "—" },
            { label: "Por", w: "1fr", render: (m) => m.recorded_by ?? "—" }
          ]}
          rows={movements.data ?? []} rowKey={(m) => m.id} empty={movements.isPending ? "carregando…" : "nenhum movimento"}
        />
      )}
      {notice && <p role="status" style={okText}>{notice}</p>}
      <div style={row}>
        <label style={fieldLabel}>Movimento
          <select value={draft.kind} style={inputStyle} onChange={(e) => set({ kind: e.target.value as MovementDraft["kind"] })}>
            <option value="">—</option>
            <option value="opening">abertura</option>
            <option value="loss">perda</option>
            <option value="expiry">vencimento</option>
            {admin && <option value="adjustment">ajuste (com sinal)</option>}
          </select>
        </label>
        <label style={fieldLabel}>Quantidade<input value={draft.quantity} inputMode="numeric" style={inputStyle} onChange={(e) => set({ quantity: e.target.value })} /></label>
        <label style={fieldLabel}>Motivo<input value={draft.reason} style={inputStyle} onChange={(e) => set({ reason: e.target.value })} /></label>
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div><ActionButton onClick={save}>Registrar movimento</ActionButton></div>
    </fieldset>
  );
}
```

`src/App.tsx`:

```tsx
import { LabInventory } from "./modules/LabInventory";
```

```tsx
    case "lab-inventory": return <LabInventory />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabInventory.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabInventory.tsx src/modules/LabInventory.test.tsx src/modules/inventory/SuppliesPanel.tsx src/modules/inventory/SupplyLotsPanel.tsx src/App.tsx
$G commit -m "feat: add lab inventory supplies, lot receipts and movements

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Lote em uso por aparelho e alertas do estoque

**Files:**
- Create: `src/modules/inventory/InUsePanel.tsx`, `src/modules/inventory/AlertsPanel.tsx`
- Modify: `src/modules/LabInventory.tsx`
- Test: `src/modules/inventory/InUsePanel.test.tsx`

**Interfaces:**
- Consumes: `listLabInstruments`, `listSupplies`, `listSupplyLots`, `listInUse`, `setInUse`, `getInventoryAlerts`, `listLabExams` (Task 1); `INV_*` chaves, `inventoryError`, `refreshInventory` (Task 3); `QC_INSTRUMENTS_KEY`, `QC_EXAMS_KEY` (Task 2); `todayInCity`, `fmtDay`, `fmtDateTime`.
- Produces: `InUsePanel({ providerId, exams })`; `AlertsPanel({ providerId })`; `InventoryBody` mostra `AlertsPanel` no topo e `InUsePanel` no fim.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/inventory/InUsePanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listLabInstruments: vi.fn(), listSupplies: vi.fn(), listSupplyLots: vi.fn(), listInUse: vi.fn(),
    setInUse: vi.fn(), getInventoryAlerts: vi.fn() };
});

import * as api from "../../lib/api";
import { InUsePanel } from "./InUsePanel";
import { AlertsPanel } from "./AlertsPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CHOL_EXAM, GLU_EXAM, NOW21C, inUse, instrument, qcUser, supply, supplyLot } from "../../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listLabInstruments, api.listSupplies, api.listSupplyLots, api.listInUse, api.setInUse, api.getInventoryAlerts ];
afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
  mocked(api.listSupplies).mockResolvedValue([
    supply(), supply({ id: "su2", name: "Ponteiras", kind: "consumable", tests_per_package: null, lab_exam_ids: [] }),
    supply({ id: "su3", name: "Colesterol CHOD-PAP", lab_exam_ids: [ "le2" ] })
  ]);
  mocked(api.listSupplyLots).mockResolvedValue([
    supplyLot(), supplyLot({ id: "sl9", lot_number: "R-500", expires_on: "2026-10-01" }),
    supplyLot({ id: "sl3", supply_id: "su2", supply_name: "Ponteiras", lot_number: "P-1" }),
    supplyLot({ id: "sl4", supply_id: "su3", supply_name: "Colesterol CHOD-PAP", lot_number: "C-7" })
  ]);
  mocked(api.listInUse).mockResolvedValue([ inUse() ]);
  mocked(api.setInUse).mockResolvedValue(inUse({ id: "iu2", lot_id: "sl1" }));
  mocked(api.getInventoryAlerts).mockResolvedValue({
    expiring: [ { lot_id: "sl1", supply_name: "Glicose GOD-PAP 2×100 mL", lot_number: "R-778", expires_on: "2026-11-05", balance: 5 } ],
    below_minimum: [ { supply_id: "su2", name: "Ponteiras", unit: "caixa", balance: 1, minimum_stock: 3 } ], untraced_results: 3
  });
});

describe("InUsePanel", () => {
  it("lista o lote em uso do aparelho; troca só com lote de reagente do exame, não vencido", async () => {
    renderWithProviders(<InUsePanel providerId="pv9" exams={[ GLU_EXAM, CHOL_EXAM ]} />);
    const panel = await screen.findByRole("region", { name: "Lote em uso por aparelho" });
    expect(await within(panel).findByText("R-778 · Glicose GOD-PAP 2×100 mL")).not.toBeNull();
    expect(api.listInUse).toHaveBeenCalledWith("in1");
    fireEvent.change(within(panel).getByLabelText("Exame"), { target: { value: "le1" } });
    const lots = [ ...(within(panel).getByLabelText("Lote do reagente") as HTMLSelectElement).options ].map((o) => o.value);
    expect(lots).toEqual([ "", "sl1" ]);
    fireEvent.change(within(panel).getByLabelText("Lote do reagente"), { target: { value: "sl1" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Pôr em uso" }));
    await waitFor(() => expect(api.setInUse).toHaveBeenCalledWith({ lot_id: "sl1", instrument_id: "in1", lab_exam_id: "le1" }));
    expect(await within(panel).findByText("Lote em uso trocado — o anterior deste aparelho e exame foi encerrado.")).not.toBeNull();
    await waitFor(() => expect(api.listInUse).toHaveBeenCalledTimes(2));
  });

  it("422 exam_not_linked vira frase", async () => {
    mocked(api.setInUse).mockRejectedValueOnce(new api.ApiError(422, { error: "exam_not_linked" }, "x"));
    renderWithProviders(<InUsePanel providerId="pv9" exams={[ GLU_EXAM ]} />);
    const panel = await screen.findByRole("region", { name: "Lote em uso por aparelho" });
    await within(panel).findByText("R-778 · Glicose GOD-PAP 2×100 mL");
    fireEvent.change(within(panel).getByLabelText("Exame"), { target: { value: "le1" } });
    fireEvent.change(within(panel).getByLabelText("Lote do reagente"), { target: { value: "sl1" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Pôr em uso" }));
    expect((await within(panel).findByRole("alert")).textContent).toBe("este exame não está ligado ao item — o administrador liga no cadastro do item");
  });
});

describe("AlertsPanel", () => {
  it("vencendo, abaixo do mínimo e resultados sem rastreio", async () => {
    renderWithProviders(<AlertsPanel providerId="pv9" />);
    const panel = await screen.findByRole("region", { name: "Alertas do estoque" });
    expect(await within(panel).findByText("R-778 · Glicose GOD-PAP 2×100 mL")).not.toBeNull();
    expect(within(panel).getByText("1 caixa (mínimo 3)")).not.toBeNull();
    expect(within(panel).getByText("3 resultados sem rastreio de reagente")).not.toBeNull();
    expect(api.getInventoryAlerts).toHaveBeenCalledWith("pv9");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/inventory/InUsePanel.test.tsx`
Expected: FAIL — módulos não existem.

- [ ] **Step 3: Write minimal implementation**

`src/modules/inventory/InUsePanel.tsx`:

```tsx
// Lote de reagente em uso por aparelho e exame (spec §6.3; contrato §5 e §7):
// é o lote que o api grava em cada resultado e valor de controle. Pôr um novo
// em uso encerra o anterior do mesmo aparelho e exame (o api faz). Só lote de
// reagente ligado ao exame e dentro da validade é oferecido.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listInUse, listLabInstruments, listSupplies, listSupplyLots, setInUse, type LabExamRef, type LabSupplyInUse } from "../../lib/api";
import { INV_IN_USE_KEY, INV_LOTS_KEY, INV_SUPPLIES_KEY, inventoryError, refreshInventory } from "../../lib/inventory";
import { QC_INSTRUMENTS_KEY } from "../../lib/qc";
import { todayInCity } from "../../lib/campaigns";
import { fmtDay } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle } from "../../components/formStyles";
import { alert, fieldLabel, muted, okText, row, stack } from "../exams/styles";

export function InUsePanel({ providerId, exams }: { providerId: string; exams: LabExamRef[] }) {
  const queryClient = useQueryClient();
  const today = todayInCity();
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const supplies = useQuery({ queryKey: [ INV_SUPPLIES_KEY, providerId ], queryFn: () => listSupplies(providerId), retry: false });
  const lots = useQuery({ queryKey: [ INV_LOTS_KEY, "provider", providerId ], queryFn: () => listSupplyLots({ providerId }), retry: false });
  const active = (instruments.data ?? []).filter((i) => i.active);
  const [ picked, setPicked ] = useState("");
  const instrumentId = picked || (active.length === 1 ? active[0].id : "");
  const inUse = useQuery({ queryKey: [ INV_IN_USE_KEY, instrumentId ], queryFn: () => listInUse(instrumentId), enabled: instrumentId !== "", retry: false });
  const [ examId, setExamId ] = useState("");
  const [ lotId, setLotId ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const reagentIds = (supplies.data ?? []).filter((s) => s.kind === "reagent" && s.active && s.lab_exam_ids.includes(examId)).map((s) => s.id);
  const options = (lots.data ?? []).filter((l) => reagentIds.includes(l.supply_id) && l.expires_on >= today && l.balance > 0);

  async function save() {
    if (!instrumentId) { setError("escolha o aparelho"); return; }
    if (!examId) { setError("escolha o exame"); return; }
    if (!lotId) { setError("escolha o lote do reagente"); return; }
    setError(null);
    try {
      await setInUse({ lot_id: lotId, instrument_id: instrumentId, lab_exam_id: examId });
      setLotId("");
      setNotice("Lote em uso trocado — o anterior deste aparelho e exame foi encerrado.");
      void refreshInventory(queryClient).catch(() => undefined);
    } catch (err) {
      setError(inventoryError(err));
    }
  }

  return (
    <Panel title="Lote em uso por aparelho" sub="o lote que vai em cada resultado e valor de controle; sem lote em uso, o resultado sai sem rastreio">
      <div style={stack}>
        <label style={fieldLabel}>Aparelho
          <select value={instrumentId} style={inputStyle} onChange={(e) => { setPicked(e.target.value); setNotice(null); }}>
            <option value="">—</option>
            {active.map((i) => <option key={i.id} value={i.id}>{i.name}</option>)}
          </select>
        </label>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {instrumentId ? (inUse.isError ? <p role="alert" style={alert}>{inventoryError(inUse.error)}</p> : (
          <DataTable<LabSupplyInUse>
            cols={[
              { label: "Exame", w: "1fr", card: "title", render: (u) => u.exam_name },
              { label: "Lote · item", w: "1.6fr", render: (u) => `${u.lot_number} · ${u.supply_name}` },
              { label: "Desde", w: "1fr", render: (u) => fmtDateTime(u.started_at) }
            ]}
            rows={inUse.data ?? []} rowKey={(u) => u.id} empty={inUse.isPending ? "carregando…" : "nenhum lote em uso neste aparelho"}
          />
        )) : <p style={muted}>escolha o aparelho</p>}
        {instrumentId && (
          <div style={{ ...row, alignItems: "flex-end" }}>
            <label style={fieldLabel}>Exame
              <select value={examId} style={inputStyle} onChange={(e) => { setExamId(e.target.value); setLotId(""); }}>
                <option value="">—</option>
                {exams.filter((e) => e.id).map((e) => <option key={e.id} value={e.id}>{e.name}</option>)}
              </select>
            </label>
            <label style={fieldLabel}>Lote do reagente
              <select value={lotId} style={inputStyle} onChange={(e) => setLotId(e.target.value)}>
                <option value="">—</option>
                {options.map((l) => <option key={l.id} value={l.id}>{`${l.lot_number} · ${l.supply_name} · validade ${fmtDay(l.expires_on)}`}</option>)}
              </select>
            </label>
            <ActionButton onClick={save}>Pôr em uso</ActionButton>
          </div>
        )}
        {examId && options.length === 0 && <p style={muted}>nenhum lote de reagente ligado a este exame, com saldo e dentro da validade</p>}
        {error && <p role="alert" style={alert}>{error}</p>}
      </div>
    </Panel>
  );
}
```

`src/modules/inventory/AlertsPanel.tsx`:

```tsx
// Alertas do estoque (spec §6.6; contrato §5): lotes vencendo dentro do aviso
// do item, itens abaixo do mínimo e quantos resultados saíram sem lote de
// reagente em uso. A rota não grava trilha; lida ao abrir e depois de cada ação.
import { useQuery } from "@tanstack/react-query";
import { getInventoryAlerts, type LabExpiringLot, type LabLowSupply } from "../../lib/api";
import { INV_ALERTS_KEY, inventoryError } from "../../lib/inventory";
import { fmtDay } from "../../lib/exams";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { alert, muted, stack, title } from "../exams/styles";

export function AlertsPanel({ providerId }: { providerId: string }) {
  const alerts = useQuery({ queryKey: [ INV_ALERTS_KEY, providerId ], queryFn: () => getInventoryAlerts(providerId), retry: false });
  const data = alerts.data;
  return (
    <Panel title="Alertas do estoque" sub="vencimento, estoque mínimo e resultados sem rastreio">
      {alerts.isError ? <p role="alert" style={alert}>{inventoryError(alerts.error)}</p> : !data ? <p className="mono" style={muted}>carregando…</p> : (
        <div style={stack}>
          <p style={{ margin: 0 }}>
            {data.untraced_results > 0
              ? <Tag tone="warn">{`${data.untraced_results} ${data.untraced_results === 1 ? "resultado" : "resultados"} sem rastreio de reagente`}</Tag>
              : <span style={muted}>todos os resultados com lote de reagente</span>}
          </p>
          <h3 style={title}>Vencendo</h3>
          <DataTable<LabExpiringLot>
            cols={[
              { label: "Lote · item", w: "1.8fr", card: "title", render: (l) => `${l.lot_number} · ${l.supply_name}` },
              { label: "Validade", w: "0.8fr", render: (l) => fmtDay(l.expires_on) },
              { label: "Saldo", w: "0.6fr", align: "right", render: (l) => String(l.balance) }
            ]}
            rows={data.expiring} rowKey={(l) => l.lot_id} empty="nenhum lote vencendo"
          />
          <h3 style={title}>Abaixo do mínimo</h3>
          <DataTable<LabLowSupply>
            cols={[
              { label: "Item", w: "1.6fr", card: "title", render: (s) => s.name },
              { label: "Saldo", w: "1fr", render: (s) => `${s.balance} ${s.unit} (mínimo ${s.minimum_stock})` }
            ]}
            rows={data.below_minimum} rowKey={(s) => s.supply_id} empty="nenhum item abaixo do mínimo"
          />
        </div>
      )}
    </Panel>
  );
}
```

Em `src/modules/LabInventory.tsx`:

```tsx
import { AlertsPanel } from "./inventory/AlertsPanel";
import { InUsePanel } from "./inventory/InUsePanel";
```

```tsx
function InventoryBody({ providerId, admin }: { providerId: string; admin: boolean }) {
  const exams = useQuery({ queryKey: [ QC_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  return (
    <div style={stack}>
      <AlertsPanel providerId={providerId} />
      <SuppliesPanel providerId={providerId} admin={admin} exams={exams.data ?? []} />
      <SupplyLotsPanel providerId={providerId} admin={admin} />
      <InUsePanel providerId={providerId} exams={exams.data ?? []} />
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/inventory src/modules/LabInventory.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/inventory/InUsePanel.tsx src/modules/inventory/AlertsPanel.tsx src/modules/inventory/InUsePanel.test.tsx src/modules/LabInventory.tsx
$G commit -m "feat: add the reagent lot in use per instrument and inventory alerts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Rastreio por lote e consumo estimado, com CSV

**Files:**
- Create: `src/modules/LabTraceability.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabTraceability.test.tsx`

**Interfaces:**
- Consumes: `listSupplies`, `listSupplyLots`, `traceSupplyLot`, `fetchSupplyLotTraceCsv`, `getConsumption`, `fetchConsumptionCsv`, `listLabInstruments` (Task 1); `fmtDeviation`, `deviationTone`, `INV_SUPPLIES_KEY`, `INV_LOTS_KEY`, `INV_CONSUMPTION_KEY`, `INVENTORY_DISABLED`, `inventoryError` (Task 3); `QC_INSTRUMENTS_KEY` (Task 2); `saveFile`, `LabGate`, `LabPick` (Task 5); `todayInCity`, `fmtDateTime`.
- Produces: `LabTraceability()`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabTraceability.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listSupplies: vi.fn(), listSupplyLots: vi.fn(), traceSupplyLot: vi.fn(),
    fetchSupplyLotTraceCsv: vi.fn(), getConsumption: vi.fn(), fetchConsumptionCsv: vi.fn(), listLabInstruments: vi.fn() };
});

import * as api from "../lib/api";
import { LabTraceability } from "./LabTraceability";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21C, instrument, qcContext, qcUser, supply, supplyLot } from "../test/qcFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listSupplies, api.listSupplyLots, api.traceSupplyLot, api.fetchSupplyLotTraceCsv,
  api.getConsumption, api.fetchConsumptionCsv, api.listLabInstruments ];
const CSV = { blob: new Blob([ "a,b\n" ], { type: "text/csv" }), filename: null };

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21C));
  Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") });
  Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(() => undefined);
  mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "municipal_admin" ]));
  mocked(api.getLabContext).mockResolvedValue(qcContext());
  mocked(api.listSupplies).mockResolvedValue([ supply() ]);
  mocked(api.listSupplyLots).mockResolvedValue([ supplyLot() ]);
  mocked(api.traceSupplyLot).mockResolvedValue({
    results: [ { requisition_number: "00000123", sigtap_code: "0202010473", entered_at: "2026-10-15T08:30:00-03:00" } ],
    qc_values: [ { run_id: "qr1", analyte_key: "glicose", entered_at: "2026-10-15T07:10:00-03:00" } ]
  });
  mocked(api.fetchSupplyLotTraceCsv).mockResolvedValue(CSV);
  mocked(api.getConsumption).mockResolvedValue([
    { supply_id: "su1", name: "Glicose GOD-PAP 2×100 mL", instrument_id: "in1", tests_performed: 450, tests_opened: 400, deviation_pct: 12.5 }
  ]);
  mocked(api.fetchConsumptionCsv).mockResolvedValue(CSV);
  mocked(api.listLabInstruments).mockResolvedValue([ instrument() ]);
});

describe("LabTraceability", () => {
  it("rastreio só no clique (trilha de leitura); requisição e código do exame, sem paciente; CSV", async () => {
    renderWithProviders(<LabTraceability />);
    const panel = await screen.findByRole("region", { name: "Rastreio por lote" });
    await within(panel).findByRole("option", { name: "Glicose GOD-PAP 2×100 mL" });
    fireEvent.change(within(panel).getByLabelText("Item"), { target: { value: "su1" } });
    await within(panel).findByRole("option", { name: "R-778" });
    fireEvent.change(within(panel).getByLabelText("Lote"), { target: { value: "sl1" } });
    expect(api.traceSupplyLot).not.toHaveBeenCalled();
    fireEvent.click(within(panel).getByRole("button", { name: "Rastrear" }));
    await waitFor(() => expect(api.traceSupplyLot).toHaveBeenCalledWith("sl1"));
    expect(await within(panel).findByText("00000123")).not.toBeNull();
    expect(within(panel).getByText("0202010473")).not.toBeNull();
    expect(panel.textContent).not.toContain("Joana");
    fireEvent.click(within(panel).getByRole("button", { name: "Baixar CSV do rastreio" }));
    await waitFor(() => expect(api.fetchSupplyLotTraceCsv).toHaveBeenCalledWith("sl1"));
    expect(URL.createObjectURL).toHaveBeenCalled();
  });

  it("consumo do mês com desvio e CSV", async () => {
    renderWithProviders(<LabTraceability />);
    const panel = await screen.findByRole("region", { name: "Consumo estimado" });
    expect(await within(panel).findByText("+12,5%")).not.toBeNull();
    expect(api.getConsumption).toHaveBeenCalledWith({ providerId: "pv9", month: "2026-10" });
    expect(within(panel).getByText("BS-200")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Baixar CSV do consumo" }));
    await waitFor(() => expect(api.fetchConsumptionCsv).toHaveBeenCalledWith({ providerId: "pv9", month: "2026-10" }));
  });

  it("equipe do laboratório não vê", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(qcUser([ "lab_staff" ]));
    renderWithProviders(<LabTraceability />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.getConsumption).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabTraceability.test.tsx`
Expected: FAIL — `./LabTraceability` não existe.

- [ ] **Step 3: Write minimal implementation**

`src/modules/LabTraceability.tsx`:

```tsx
// src/modules/LabTraceability.tsx
// Rastreio por lote e consumo estimado (spec §6.7–§6.8; contrato §5). O
// rastreio grava trilha de leitura: só lê no clique, nunca sozinho, e mostra
// número da requisição e código do exame (sem paciente). O consumo compara
// testes realizados com testes abertos (embalagens × rendimento); nada é
// baixado. CSV dos dois por fetch.
import { useRef, useState } from "react";
import { useQuery } from "@tanstack/react-query";
import {
  fetchConsumptionCsv, fetchSupplyLotTraceCsv, getConsumption, listLabInstruments, listSupplies, listSupplyLots, traceSupplyLot,
  type LabConsumptionRow, type LabLotTrace
} from "../lib/api";
import { INVENTORY_DISABLED, INV_CONSUMPTION_KEY, INV_LOTS_KEY, INV_SUPPLIES_KEY, deviationTone, fmtDeviation, inventoryError } from "../lib/inventory";
import { QC_INSTRUMENTS_KEY } from "../lib/qc";
import { todayInCity } from "../lib/campaigns";
import { fmtDateTime } from "../lib/format";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { ActionButton } from "../components/ActionButton";
import { inputStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabPick } from "./lab/LabPick";
import { saveFile } from "./lab/saveFile";
import { alert, fieldLabel, muted, row, stack, title } from "./exams/styles";

const safeName = (s: string) => s.replace(/[^A-Za-z0-9-]/g, "-");

export function LabTraceability() {
  return (
    <LabGate title="Rastreio e consumo de reagentes" sub="insumos · rastreio por lote e consumo estimado"
      allow={(r) => r.admin || r.releaser} requires={[ "lab_inventory" ]} disabledText={INVENTORY_DISABLED}>
      {() => <LabPick>{(lab) => (
        <div style={stack}>
          <TracePanel providerId={lab.id} />
          <ConsumptionPanel providerId={lab.id} />
        </div>
      )}</LabPick>}
    </LabGate>
  );
}

function TracePanel({ providerId }: { providerId: string }) {
  const supplies = useQuery({ queryKey: [ INV_SUPPLIES_KEY, providerId ], queryFn: () => listSupplies(providerId), retry: false });
  const [ supplyId, setSupplyId ] = useState("");
  const lots = useQuery({ queryKey: [ INV_LOTS_KEY, supplyId ], queryFn: () => listSupplyLots({ supplyId }), enabled: supplyId !== "", retry: false });
  const [ lotId, setLotId ] = useState("");
  const [ trace, setTrace ] = useState<LabLotTrace | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const busy = useRef(false);
  const lot = (lots.data ?? []).find((l) => l.id === lotId);

  async function run() {
    if (!lotId || busy.current) return;
    busy.current = true;
    setError(null);
    try { setTrace(await traceSupplyLot(lotId)); } catch (err) { setError(inventoryError(err)); } finally { busy.current = false; }
  }

  async function csv() {
    if (!lot) return;
    try { saveFile(await fetchSupplyLotTraceCsv(lot.id), `rastreio-lote-${safeName(lot.lot_number)}.csv`); } catch (err) { setError(inventoryError(err)); }
  }

  return (
    <Panel title="Rastreio por lote" sub="requisições, exames e controles com aquele lote de reagente — sem nome de paciente">
      <div style={stack}>
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>Item
            <select value={supplyId} style={inputStyle} onChange={(e) => { setSupplyId(e.target.value); setLotId(""); setTrace(null); }}>
              <option value="">—</option>
              {(supplies.data ?? []).map((s) => <option key={s.id} value={s.id}>{s.name}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>Lote
            <select value={lotId} style={inputStyle} onChange={(e) => { setLotId(e.target.value); setTrace(null); }}>
              <option value="">—</option>
              {(lots.data ?? []).map((l) => <option key={l.id} value={l.id}>{l.lot_number}</option>)}
            </select>
          </label>
          <ActionButton disabledReason={lotId ? null : "escolha o lote"} onClick={run}>Rastrear</ActionButton>
          {trace && <ActionButton variant="secondary" onClick={csv}>Baixar CSV do rastreio</ActionButton>}
        </div>
        {error && <p role="alert" style={alert}>{error}</p>}
        {trace && (
          <>
            <h3 style={title}>Resultados</h3>
            <DataTable<LabLotTrace["results"][number]>
              cols={[
                { label: "Requisição", w: "1fr", card: "title", render: (r) => <span className="mono">{r.requisition_number}</span> },
                { label: "Exame (SIGTAP)", w: "1fr", render: (r) => <span className="mono">{r.sigtap_code}</span> },
                { label: "Quando", w: "1fr", render: (r) => fmtDateTime(r.entered_at) }
              ]}
              rows={trace.results} rowKey={(r, i) => `${r.requisition_number}:${r.sigtap_code}:${i}`} empty="nenhum resultado com este lote"
            />
            <h3 style={title}>Controles</h3>
            <DataTable<LabLotTrace["qc_values"][number]>
              cols={[
                { label: "Corrida", w: "1fr", card: "title", render: (q) => <span className="mono">{q.run_id}</span> },
                { label: "Analito", w: "1fr", render: (q) => q.analyte_key },
                { label: "Quando", w: "1fr", render: (q) => fmtDateTime(q.entered_at) }
              ]}
              rows={trace.qc_values} rowKey={(q, i) => `${q.run_id}:${q.analyte_key}:${i}`} empty="nenhum controle com este lote"
            />
          </>
        )}
        {!trace && <p style={muted}>escolha o item e o lote e clique em Rastrear (a consulta fica registrada)</p>}
      </div>
    </Panel>
  );
}

function ConsumptionPanel({ providerId }: { providerId: string }) {
  const [ month, setMonth ] = useState(() => todayInCity().slice(0, 7));
  const [ error, setError ] = useState<string | null>(null);
  const valid = /^\d{4}-\d{2}$/.test(month);
  const q = { providerId, month };
  const rows = useQuery({ queryKey: [ INV_CONSUMPTION_KEY, providerId, month ], queryFn: () => getConsumption(q), enabled: valid, retry: false });
  const instruments = useQuery({ queryKey: [ QC_INSTRUMENTS_KEY, providerId, "lab" ], queryFn: () => listLabInstruments(providerId), retry: false });
  const instName = (id: string | null) => (id ? (instruments.data ?? []).find((i) => i.id === id)?.name ?? id : "todos");

  async function csv() {
    setError(null);
    try { saveFile(await fetchConsumptionCsv(q), `consumo-reagentes-${month}.csv`); } catch (err) { setError(inventoryError(err)); }
  }

  return (
    <Panel title="Consumo estimado" sub="testes realizados (resultados, repetições e controles) × testes das embalagens abertas"
      right={<ActionButton variant="secondary" disabledReason={valid ? null : "escolha o mês"} onClick={csv}>Baixar CSV do consumo</ActionButton>}>
      <div style={stack}>
        <label style={fieldLabel}>Mês<input type="month" value={month} style={inputStyle} onChange={(e) => setMonth(e.target.value)} /></label>
        {error && <p role="alert" style={alert}>{error}</p>}
        {rows.isError ? <p role="alert" style={alert}>{inventoryError(rows.error)}</p> : (
          <DataTable<LabConsumptionRow>
            cols={[
              { label: "Item", w: "1.6fr", card: "title", render: (r) => r.name },
              { label: "Aparelho", w: "1fr", render: (r) => instName(r.instrument_id) },
              { label: "Testes realizados", w: "0.9fr", align: "right", render: (r) => String(r.tests_performed) },
              { label: "Testes abertos", w: "0.9fr", align: "right", render: (r) => String(r.tests_opened) },
              { label: "Desvio", w: "0.8fr", align: "right", render: (r) => {
                const tone = deviationTone(r.deviation_pct);
                return tone ? <Tag tone={tone}>{fmtDeviation(r.deviation_pct)}</Tag> : fmtDeviation(r.deviation_pct);
              } }
            ]}
            rows={rows.data ?? []} rowKey={(r) => `${r.supply_id}:${r.instrument_id ?? "-"}`} empty={rows.isPending && valid ? "carregando…" : "nenhum consumo neste mês"}
          />
        )}
      </div>
    </Panel>
  );
}
```

`src/App.tsx`:

```tsx
import { LabTraceability } from "./modules/LabTraceability";
```

```tsx
    case "lab-traceability": return <LabTraceability />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabTraceability.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabTraceability.tsx src/modules/LabTraceability.test.tsx src/App.tsx
$G commit -m "feat: add reagent lot traceability and estimated consumption with CSV

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: Suíte, build, revisão e prova no navegador (api real na 3043 e `fake-middleware` mandando controles)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–16 na branch `feat/mod-21c-lab-qc`; o api do 21c na **3043** com a semente do 21c e o `fake-middleware` mandando controles.
- Produces: branch pronta para merge (depois do merge do api do 21c); evidências em `evidencias/2026-10-18-modulo-21c/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **21 arquivos de teste novos** (`api.qc`, `qc`, `inventory`, `modules.qc`, `useQcBlockCount`, `LabGate.qc`, `LabInstruments`, `VersionForm.instrument`, `LabQcSetup`, `TargetsPanel`, `RulesPanel`, `LabQcRuns`, `LeveyJennings`, `LabQcChart`, `LabQcBlocks`, `LabRelease.qc`, `ResultEntryForm.lot`, `LabEqa`, `LabInventory`, `InUsePanel`, `LabTraceability`) e três arquivos de teste existentes com casos novos (`features.test.ts`, `modules.lab.test.ts`, `App.navigation.test.tsx`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/SegmentedControl.tsx src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css package.json package-lock.json
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css` ou dependência mudados (o Recharts já estava).

- [ ] **Step 2: Nada vaza e nada relê sozinho**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage" src/lib/qc.ts src/lib/inventory.ts src/modules/qc src/modules/eqa src/modules/inventory src/modules/LabQc*.tsx src/modules/LabEqa.tsx src/modules/LabInventory.tsx src/modules/LabTraceability.tsx src/modules/LabInstruments.tsx src/hooks/useQcBlockCount.ts | grep -v "\.test\."
grep -rn "refetchInterval" src/modules src/hooks | grep -v "\.test\."
sed -n '/─── Qualidade e insumos do laboratório (módulo 21c/,$p' src/lib/api.ts | grep -on "p.set(\"[a-z_]*\"\|byProvider([a-zA-Z.]*, { [a-z_]* }\|new URLSearchParams({ [a-z_: .,A-Z]* })"
grep -rn "maxLength" src/modules/qc src/modules/eqa src/modules/inventory src/modules/LabQc*.tsx src/modules/LabEqa.tsx src/modules/LabInventory.tsx src/modules/LabTraceability.tsx src/modules/LabInstruments.tsx
```

Expected: o primeiro e o último sem saída; o segundo só com `useLabCriticalCount.ts` (21b) e `useQcBlockCount.ts` (rotas `/count`, sem trilha); o terceiro só com `provider_id`, `status`, `month`, `year`, `lot_id`, `level`, `analyte_key`, `instrument_id`, `from`, `to`, `date`, `supply_id`, `with_balance` — nenhum `value`, `mean`, `sd`, `justification`, `reason`, `text`, `quantity`, `barcode` ou `lot_number`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3, §5, §6, §8, §9, §10), o ADR 0038, o contrato (§1–§5, §7, §9) e as Divergências deste plano. Pontos de atenção:
- sem `lab_quality_control` e `lab_inventory`, menu, liberação, digitação e versão do exame são exatamente os do 21b (testes antigos verdes; só a lista `LAB_IDS` de `modules.lab.test.ts` mudou);
- nenhuma consulta com `refetchInterval` além das duas contagens; rastreio só no clique;
- vírgula decimal em toda digitação nova; ponto recusado; nada de `maxLength` silencioso;
- 409 `qc_blocked`: nada liberado, itens listados, "Liberar sem os bloqueados", exceção só ao RT; `not_blocked` tratado como resolvido;
- Levey-Jennings em desvios-padrão, linha na troca de versão, legenda com a versão vigente, tabela acessível dos pontos marcados;
- RT/substituto só pelo contexto (`technical_lead_of`); o api recusa de novo;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 21c na **3043** (plano do api: servidor do worktree `apps/api/.claude/mod21c` no `api-dev`, semente do 21c com o "Laboratório Municipal", o aparelho `BS-200` com `hl7_equipment_id`, um material de controle com níveis normal e patológico, lote com códigos de barras e referência da bula, uma chave com 20 corridas aprovadas para a sugestão, um reagente de glicose com lote, um liberador CRF que é RT e a equipe do laboratório; o `fake-middleware` mandando corridas de controle, uma delas reprovada por 2-2s). Ligue em Curitiba, pelo maintenance, `exam_tracking`, `municipal_lab`, `lab_hl7_inbound`, `fake_middleware`, `lab_quality_control` e `lab_inventory`. O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo: o do worktree é do macOS e não roda no Alpine):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod21c --network rota-saude_default -p 5196:5196 \
  -v $R/.worktrees/dashboard-mod21c:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3043 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5196 --host 0.0.0.0"
docker logs -f dashboard-mod21c   # espere o "ready"; Ctrl+C sai do log, o container segue
docker compose up -d fake-middleware
docker exec $(docker compose ps -q fake-middleware) printenv | grep -c FAKE_MIDDLEWARE
```

(Se o plano do api subir o servidor do 21c num container próprio, use o nome dele no lugar de `api`. Variável nova no compose não chega a container já rodando: `printenv` é a verdade; recrie com `docker compose up -d --force-recreate fake-middleware` se faltar. Não gere token novo do middleware para o "Laboratório Municipal": o novo invalida o da semente.)

Abra `http://curitiba.localhost:5196/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot (salve em `evidencias/2026-10-18-modulo-21c/dashboard/`):
1. **Admin** — Laboratório → **Aparelhos**: o `BS-200` com o identificador HL7; cadastrar um segundo aparelho (step-up). Exames e liberadores → nova versão da glicose com o **Aparelho** escolhido.
2. **RT (liberador CRF)** — **Controles e referências**: o material com os dois níveis; **Novo lote** com um código de barras por nível (se houver leitor USB, leia a etiqueta do frasco; o Enter não envia o formulário); **Referências** do lote → referência da bula de cada nível com vírgula ("98,0" e "2,5"); trocar uma vigente pede o motivo.
3. **Equipe** — **Corridas de controle**: escolha o aparelho; digite o nível normal com vírgula → corrida "aprovada"; "98.5" é recusado com a frase do ponto.
4. **HL7** — em até um ciclo do `fake-middleware`, a corrida dele aparece com origem "aparelho"; a reprovada por 2-2s mostra o desfecho "reprovado" e a regra; o menu passa a "Bloqueios e revisões do CIQ (1 ativo)" no minuto seguinte.
5. **Levey-Jennings** (liberador): lote, nível, analito; as linhas de ±1/2/3 dp, os pontos coloridos, o 2-2s na tabela de pontos marcados; passe o mouse num ponto (valor, z, versão).
6. **Liberação bloqueada** — com um resultado de glicose do mesmo aparelho digitado depois da reprovação: **Liberar laudo** → "bloqueado pelo controle de qualidade" com o item, o analito e o aparelho; **Liberar sem os bloqueados** desmarca só a glicose; como RT, **Liberar com exceção** com justificativa → "exceção registrada" → liberar de novo (assinatura avançada, código digitado pelo usuário).
7. **Bloqueios e revisões** — a exceção em "Exceções concedidas"; em "Revisões pendentes", os laudos liberados desde a última corrida aprovada; **Manter** um; **Retificar** outro → a conferência da requisição abre e a retificação do 21b segue (motivo, contato, assinatura).
8. **Ação corretiva** → "Ação corretiva registrada…"; nova corrida com todos os níveis aprovada (digitada ou do `fake-middleware`) → o bloqueio vai para "encerrados" e o contador do menu some no minuto seguinte.
9. **Sugestão de referência** — em Controles e referências, a chave da semente com "20 corridas aprovadas"; como RT, **Aceitar** com motivo → versão nova "laboratório"; no Levey-Jennings, a linha vertical da troca e o aviso de versões misturadas.
10. **Regras** — desligar 4-1s sem justificativa é recusado; com justificativa, salva e mostra quem alterou.
11. **CEQ** (liberador ou admin) — **Nova rodada** PNCQ com um item fora do aceitável → "inadequado" sugerido e a ação exigida; anexar o relatório anual em PDF; **Relatório anual** abre em janela nova; a taxa no "Adequação no ano".
12. **Estoque** (equipe) — entrada de um lote de reagente (lote vencido é recusado); **Movimentos** → abertura de 1; perda maior que o saldo é recusada com o saldo na frase; o ajuste só aparece para o admin; **Lote em uso por aparelho** → o lote novo no `BS-200` para glicose ("o anterior… foi encerrado"); os **Alertas** (vencendo, abaixo do mínimo, resultados sem rastreio).
13. **Resultado com lote** — digite uma glicose (Digitação de resultados) → no histórico do componente aparece "lote R-…"; um exame sem lote em uso aparece "sem rastreio de reagente"; na Liberação, a mesma marca em cada valor; numa corrida de controle, o lote no valor.
14. **Rastreio e consumo** (admin) — escolha o lote e **Rastrear** → requisições e controles com aquele lote, sem nome de paciente; **Baixar CSV do rastreio** (abra e confira que não há paciente); **Consumo estimado** do mês com o desvio; **Baixar CSV do consumo**.
15. **Sem polling** — com a aba de rede aberta em Bloqueios e revisões por 2 minutos, só `/lab/qc/blocks/active/count` (e `/lab/criticals/unnotified/count`, do 21b) se repetem; nenhuma chamada a `/lab/qc/blocks?`, `/lab/qc/reviews`, `/lab/qc/runs` ou `/lab/inventory/lots/…/trace` sem ação da pessoa.
16. **Telefone** (largura 375): Corridas, Bloqueios, Estoque e Rastreio viram cartões, sem rolagem lateral; o gráfico cabe na largura.
17. No maintenance, desligar `lab_quality_control` e recarregar: as telas do CIQ e do CEQ somem do menu, a liberação volta a ser a do 21b; Estoque e Aparelhos seguem. Desligar também `lab_inventory`: Aparelhos e a escolha de aparelho na versão somem; os resultados não mostram marca de lote.

Depois: `docker rm -f dashboard-mod21c`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

Para cada card F-21.25..F-21.34: se a parte do api dele já está Done, mova para **Done** (o mesmo `gh project item-edit` da Task 0, com o id da opção Done `98236657`); senão deixe em In Progress e anote. **Pare.** O merge do dashboard só vem depois do merge do api do 21c, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard → maintenance (contrato §10). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21c-lab-qc
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod21c
/opt/homebrew/bin/git branch -d feat/mod-21c-lab-qc
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (C1–C17), para não colidir com as D do 21b. As que só nomeiam forma (C2, C3, C10, C16, C17) podem ser resolvidas pelo api escolhendo outra forma, desde que o contrato a fixe e este plano ajuste `src/lib/api.ts` e `src/test/qcFixtures.ts`.

1. **C1 — Quem é RT ou substituto.** Exceção, revisão, aceite da sugestão, regras e o cadastro do CIQ são do RT/substituto (contrato §3; spec §8), mas nada diz à tela quem é. Proposta: `GET /lab/context` (D1 do 21b) ganha `"technical_lead_of": [ <provider_id> ]` — laboratórios em que a sessão tem a ligação de liberador ativa com `technical_lead` ou `technical_deputy`. Sem dado clínico, sem trilha. Ausente (api antigo) = ninguém é RT na tela.
2. **C2 — Filtro por laboratório e envelope das listas.** O contrato não diz como as listas se restringem ao laboratório nem o envelope. Proposta: `provider_id` obrigatório em `GET /lab/instruments`, `/lab/admin/instruments`, `/lab/qc/materials`, `/lab/qc/lots` (+ `status` opcional), `/lab/qc/targets/suggestions`, `/lab/qc/blocks` (+ `status`), `/lab/qc/reviews` (+ `status`), `/lab/qc/exceptions` (C7), `/lab/eqa/rounds` (+ `year`), `/lab/eqa/summary`, `/lab/inventory/supplies`, `/lab/inventory/alerts`, `/lab/inventory/consumption`; `/lab/inventory/lots` aceita `supply_id` **ou** `provider_id`. Envelopes: `{ "instruments" }`, `{ "materials" }`, `{ "lots" }`, `{ "suggestions" }`, `{ "runs" }`, `{ "blocks" }`, `{ "reviews" }`, `{ "exceptions" }`, `{ "rounds" }`, `{ "rows" }` (resumo do CEQ e consumo), `{ "supplies" }`, `{ "movements" }`, `{ "in_use" }`. `GET /lab/instruments` também para o `municipal_admin` (o cadastro do CIQ lê os aparelhos por ela).
3. **C3 — Formas que o contrato não fixa.** `<qc_lot>` = `{ "id", "material_id", "material_name", "lot_number", "expires_on", "level_barcodes", "status", "opened_on", "closed_on" }`; `GET /lab/qc/targets?lot_id=` → `{ "current": [<qc_target>], "history": [<qc_target>] }` com `<qc_target>` = `{ "id", "lot_id", "level", "analyte_key", "instrument_id", "version", "source", "mean", "sd", "accepted_by", "accepted_at", "based_on_runs" }` (também a resposta de `POST /lab/qc/targets` e do aceite); `<qc_review>` = `{ "id", "block_id", "item_id", "requisition_id", "requisition_number", "sigtap_code", "exam_label", "analyte_key", "instrument_id", "released_at", "decision", "note" }` (sem paciente; `requisition_id` para abrir a retificação); `<lab_supply>` ganha `"balance"`; `<lab_supply_lot>` = `{ "id", "supply_id", "supply_name", "lot_number", "expires_on", "supplier", "origin", "balance" }`; `<lab_supply_in_use>` = `{ "id", "lot_id", "lot_number", "supply_name", "instrument_id", "lab_exam_id", "exam_name", "started_at" }`; alertas: `expiring: [ { "lot_id", "supply_name", "lot_number", "expires_on", "balance" } ]`, `below_minimum: [ { "supply_id", "name", "unit", "balance", "minimum_stock" } ]`; resumo do CEQ: `{ "rows": [ { "analyte_key", "lab_exam_id", "rounds", "adequate", "inadequate" } ] }`; `<eqa_round>` ganha `"recorded_by"`.
4. **C4 — Decimais como string com ponto.** `value`, `mean`, `sd`, `submitted`, `target`, `acceptable_low`, `acceptable_high` vão e voltam como string com ponto (`"98.5"`), como o valor do resultado no 21b (D11) — nada de float arredondado no caminho. `z_score` é número. Quantidades de estoque são inteiras (C15).
5. **C5 — Exames e componentes fora do admin.** Materiais (RT), CEQ (liberador) e lote em uso (equipe) precisam do id do exame e dos componentes; `/lab/admin/exams` é só do admin. Proposta: `GET /lab/exams?provider_id=` (D3 do 21b) ganha `"id"` e `"analytes": [ { "key", "name" } ]` (da versão vigente) e passa a valer também para `lab_releaser` e `municipal_admin`.
6. **C6 — Motivo na troca da referência.** A spec §9 exige motivo na troca de referência; o contrato não tem o campo. Proposta: `POST /lab/qc/targets` aceita `"reason"`, obrigatório quando já existe referência vigente para a mesma chave; `POST /lab/qc/targets/suggestions/accept` exige `"reason"`; 422 `reason_required`. O motivo fica na trilha.
7. **C7 — Lista das exceções.** A spec §5.9 tem a tela de exceções e §5.5 diz que a exceção fica "no relatório"; o contrato só tem o `POST`. Proposta: `GET /lab/qc/exceptions?provider_id=&month=` → `{ "exceptions": [ { "id", "block_id", "item_id", "requisition_number", "analyte_key", "instrument_id", "justification", "granted_by", "granted_at" } ] }` (sem paciente), para liberador e admin.
8. **C8 — Lista dos movimentos.** A tela de movimentos (spec §6) precisa ler o histórico do lote; o contrato só tem o `POST`. Proposta: `GET /lab/inventory/movements?lot_id=` → `{ "movements": [ { "id", "lot_id", "kind", "quantity" (com sinal), "reason", "document", "recorded_by", "recorded_at" } ] }`, do mais novo ao mais antigo.
9. **C9 — Nome do aparelho no `qc_blocked`.** A tela da liberação não lê aparelhos. Proposta: cada item do 409 `qc_blocked` ganha `"instrument_name"` (a tela cai no id se faltar).
10. **C10 — Lote no valor de controle.** `<qc_value>` traz só `reagent_lot_id`; a corrida mostra o número do lote como o resultado do paciente (§7). Proposta: `<qc_value>` ganha `"reagent_lot": { "id", "lot_number" }|null` e `"lot_id"` (o lote de controle).
11. **C11 — Contador de bloqueios.** `GET /lab/qc/blocks/active/count` → `{ "count" }`, só da sessão `lab_staff`/`lab_releaser`, dos laboratórios dela, sem trilha (molde da D11 do 21b); 403 `feature_disabled` com o CIQ desligado. É a única rota do 21c que o dashboard relê sozinho (a cada minuto).
12. **C12 — Relatório anual do CEQ.** O contrato tem `annual_report_upload_id` mas não a rota de envio nem a de leitura. Proposta: `POST /lab/eqa/uploads { "content_type": "application/pdf", "data": <base64> }` → `{ "upload_id" }` (só PDF, até 10 MB, cifrado como os PDFs do 21a; o nome do arquivo não vai) e `GET /lab/eqa/rounds/:id/annual_report.pdf` (trilha de leitura).
13. **C13 — Interruptor dos aparelhos.** Aparelhos servem ao CIQ e ao lote em uso do estoque; o contrato não diz o interruptor. Proposta: as rotas de aparelhos respondem com **qualquer um** de `lab_quality_control` ou `lab_inventory` ligado (403 `feature_disabled` só com os dois desligados); `<lab_exam_version>` do admin passa a trazer `"lab_instrument_id"` (o dashboard pré-escolhe na versão nova).
14. **C14 — Quem registra a ação corretiva.** O contrato não diz. Proposta: `lab_staff` e `lab_releaser` do laboratório (quem opera o aparelho); o fechamento segue automático.
15. **C15 — Quantidades inteiras e sinal.** Proposta: `quantity` inteira (na unidade do item); em `opening`, `loss` e `expiry` vai positiva e o api grava com sinal negativo; em `adjustment` vai com sinal; a resposta e a lista de movimentos trazem a quantidade com sinal. 422 `invalid_quantity` para zero, fração ou sinal errado.
16. **C16 — Leitura das regras.** `GET /lab/qc/rules?instrument_id=&analyte_key=` → `{ "instrument_id", "analyte_key", "rules": { "<regra>": bool }, "is_default", "justification", "changed_by", "changed_at" }` — sem linha gravada, `is_default: true` e todas ligadas; o `PUT` devolve a mesma forma. `justification_required` só quando alguma regra é desligada em relação ao estado anterior.
17. **C17 — Gráfico sem referência.** `GET /lab/qc/chart` exige `from` e `to` (`YYYY-MM-DD`); `target` é `null` quando a chave não tem referência vigente; `points` do mais antigo ao mais novo, cada um com o `z_score` e a `target_version` com que foi avaliado (o dashboard plota em desvios-padrão, nunca reprojeta).

Alinhamento com o plano do api (escrito em paralelo): C1–C17 precisam entrar no contrato e no plano do api antes da Task 17.

## Self-review

- **Cobertura (spec §3, §5, §6, §8–§10; contrato §2–§5 e §7):** interruptores — Tasks 1 (rótulos), 4 (menu) e 5 (portão); aparelhos e aparelho da versão — Task 6; materiais e lotes com códigos por nível — Task 7; referências da bula, sugestão com 20 corridas e aceite do RT, regras com justificativa — Task 8; corridas do dia por aparelho, digitação em vírgula, corrida nova a pedido, aviso de bloqueio — Task 9; Levey-Jennings (±1/2/3 dp, regras marcadas, versão visível) com o Recharts — Task 10; bloqueios ativos e encerrados, ação corretiva, exceções concedidas, revisões com `kept`/`rectify` e a retificação do 21b — Task 11; liberação com `qc_blocked`, itens e atalho para a exceção, e lote/“sem rastreio” na digitação e na liberação — Task 12; CEQ com rodadas, adequação, ação para inadequado, relatório anual e resumo — Task 13; itens, entrada de lote, movimentos, ajuste só admin, saldo — Task 14; lote em uso por aparelho e alertas — Task 15; rastreio por lote e consumo estimado com CSV — Task 16; contador por `/count` — Task 4; prova no navegador com api na 3043 e `fake-middleware` mandando controles — Task 17. O controle pela HL7 (F-21.28) é do api e do contracts; aqui só aparece como origem "aparelho" (Tasks 9 e 17). Interruptores no maintenance e catálogo público são de outros planos.
- **Placeholders:** nenhum código faltando. Pontos que dependem do outro plano estão marcados: as contas da semente, o nome do container do api (Task 17) e as formas propostas nas Divergências.
- **Tipos:** `LabChoice` (Task 5) é o das Tasks 6–16; `LabInstrument`, `QcLot`, `QcMaterial`, `QcTarget`, `QcRun`, `QcValueResult`, `QcBlock`, `QcReview`, `QcException`, `QcBlockedItem`, `EqaRound`, `LabSupply*` (Task 1) são os das telas; `TargetDraft`/`targetBody(d, lotId, replacing)`, `rulesProblem(before, after, justification)`, `ljModel`, `EqaRoundDraft`/`eqaBody(d, providerId)` (Task 2) e `MovementDraft`/`movementProblem(d, balance, isAdmin)`, `reagentLotMark(row, inventoryOn)` (Task 3) batem com os usos; `versionBody(d, withInstrument)` (Task 6) mantém o 21b igual sem o 2º argumento; `ReleaseReview.qcLead` (Task 12) é opcional e a `ReviewsPanel` (Task 11) não o passa; `shellGroups(…, qcBlocks = 0)` (Task 4) mantém as chamadas antigas.
- **Review Focus:** 1 — Task 9 ("série de valores: ponto bloqueia tudo; 422 no meio marca o campo e mantém os não enviados"); 2 — Task 12 ("409 qc_blocked: lista, não libera e desmarca só os bloqueados" e "RT: exceção com justificativa e liberação de novo"); 3 — Task 2 ("troca de versão vira marca na série e aviso de versões misturadas") e Task 10 ("duas versões: linha vertical na troca e aviso na legenda"); 4 — Task 4 ("relê só /count a cada minuto e nunca a lista" e "sem o interruptor ou sem papel (enabled false): nenhuma leitura"); 5 — Task 3 ("saída maior que o saldo e ajuste sem admin") e Task 14 ("422 insufficient_balance: relê os lotes e mantém o formulário").
