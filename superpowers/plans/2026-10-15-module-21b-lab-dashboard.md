# Módulo 21b — Laboratório municipal (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21b** (`feat/mod-21b-lab`, porta de dev **3042**) com as rotas `/lab/*` do contrato e as rotas das Divergências deste plano (fim do arquivo), os interruptores `municipal_lab`, `lab_hl7_inbound` e `fake_middleware` na sessão, a semente do laboratório (prestador municipal `uses_municipal_lab`, UBS `collects_samples`, exames adotados, contas `lab_staff` e `lab_releaser`) e o serviço `fake-middleware` no compose. Ele é **mergeado antes** deste (contrato §13: contracts → api → dashboard → maintenance). As Tasks 0–19 não dependem dele (os testes usam dublês de `src/lib/api`); a prova da Task 20 depende.

**Goal:** Lado dashboard do subprojeto 21b: grupo novo **"Laboratório"** com requisição (exames do 21a candidatos e pedido em papel), comprovante e etiquetas; coleta por leitura de código de barras; remessa da UBS (montar, temperatura, enviar), recoletas e amostras que não chegaram; conferência da remessa e recebimento no laboratório (aceitar, rejeitar, aceitar com restrição); mapa de trabalho com filtros e PDF; digitação por mapa e por leitura (destaque fora da referência, bloqueio fora do limite, calculados, repetições); fila de liberação com assinatura ICP ou avançada (TOTP), críticos sem aviso e registro da comunicação; retificação; entrega e situação da requisição; admin do laboratório (adotar exame do catálogo-base, versões com componentes e faixas por sexo e idade, motivos de rejeição, liberadores com RT e substituto, token do middleware mostrado uma vez); as marcas `uses_municipal_lab` (prestador) e `collects_samples` (unidade); relatórios de produção, rejeição e prazo com CSV; e, no 21a, o resultado com `source: lab`, o laudo retificado (substituído, visível só ao profissional) e o número do GAL na tela do exame.

**Architecture:** O cliente HTTP ganha a seção "Laboratório" no fim de `src/lib/api.ts` (tipos do contrato §1–§6 e §9 e as funções; na URL só ids, situações, setor, bancada, data, mês, número da requisição, código de barras e o cursor). As regras de tela ficam puras em `src/lib/lab.ts` (rótulos, papéis, leitura do código de barras, número com vírgula, limite de digitação, prévia da marca, validação dos formulários, frases das recusas) e `src/lib/labAdmin.ts` (formulário de versão: componentes, faixas por sexo e idade). As peças comuns ficam em `src/modules/lab/` (`ScanField`, `openLabPdf`, `useLabContext`, `LabRequisitionDetail`, `ResultEntryForm`, `SignatureChoice`, `CriticalNoticeForm`, `RectifyForm`) e as telas em `src/modules/` (`LabRequisitions`, `LabCollection`, `LabShipments`, `LabReception`, `LabWorklist`, `LabEntry`, `LabRelease`, `LabAdmin`, `LabReports`), registradas no grupo novo **"Laboratório"** do shell (`modules.ts`, `catalog.ts`, `App.tsx`). As telas existentes ganham o mínimo: `ProviderForm` (marca `uses_municipal_lab`), `UnitForm`/`Units` (marca `collects_samples`), `team.ts` (papéis `lab_staff` e `lab_releaser` no convite), `signature.ts`/`SignaturePending` (laudo laboratorial nas pendências de assinatura ICP), `ExamResultBlock`/`ExamOrderDetail` (fonte `lab`, resultados substituídos, GAL). O dashboard nunca decide o que o api garante: quem lê, a situação, a faixa de referência aplicável, a fórmula, a marca final, quem libera e a política do cidadão são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5 (`useQuery`, `useInfiniteQuery`), Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`, proxy de dev), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-15-module-21b-municipal-lab-design.md` (§2 decisões, §4 dados, §5 fluxo, §7 telas, §8 segurança, §9 testes, §11 F-IDs), `docs/adr/0037.md` e o contrato `docs/superpowers/plans/2026-10-15-module-21b-lab-contracts.md` (§1–§6, §8 por contraste, §9, §11) com a base do 21a (`2026-10-13-module-21a-exams-contracts.md`, inclusive §15 A1–A49). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim; nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): papéis `lab_staff` e `lab_releaser` (os dois em `PRIVILEGED_ROLES`); RT/substituto `technical_lead`/`technical_deputy`; `origin` nova `lab_reception`; `source` nova `lab`; `collection_place` `ubs` | `lab`; requisição `open` | `in_analysis` | `typed` | `released` | `delivered`; amostra `awaiting` | `in_transit` | `received` | `rejected` | `accepted_with_restriction` | `not_received`; remessa `open` | `sent` | `received`; laudo `pending_signature` | `released` | `rectified`; marca `normal` | `low` | `high` | `critical_low` | `critical_high`; assinatura `icp` | `advanced`; conselho `CRF` | `CRBM` | `CRM`. Código desconhecido aparece cru, nunca some.
- Interruptor: `municipal_lab` na sessão liga tudo deste plano (e exige `exam_tracking`, que o api garante). Desligado, o dashboard é exatamente o de hoje: sem grupo "Laboratório", sem as marcas no prestador e na unidade (os papéis `lab_staff` e `lab_releaser` entram na lista do convite como o `exam_regulator` do 21a: a lista é fixa e o api valida). 403 `feature_disabled` → `LAB_DISABLED` ("o laboratório municipal está desligado nesta cidade"). O dashboard nunca escreve interruptor; `lab_hl7_inbound` e `fake_middleware` ganham só rótulo em `features.ts`.
- **Nenhum polling em rota que grava trilha** (A46 do 21a). Só a contagem de críticos sem aviso (`GET /lab/criticals/unnotified/count`, sem dado clínico e sem trilha — Divergência D11) relê a cada minuto. Listas e detalhes com valor (`/lab/entry`, `/lab/releases`, `/lab/criticals/unnotified`, `/lab/requisitions/:id`) só leem quando a tela abre ou depois de uma ação; nenhum `refetchInterval`, nenhum `refetchOnWindowFocus` nelas (`retry: false`, `gcTime: 0`, `staleTime: 0`).
- O leitor de código de barras é um teclado: o campo de leitura (`ScanField`) recebe os caracteres e o **Enter** encerra a leitura. Nada de agente local, porta serial ou API do navegador além do teclado. O campo nunca rouba o foco de outro controle; sem foco, a tela diz "clique no campo para ler".
- Na URL vão só ids, situações, setor, bancada, `YYYY-MM-DD`, `YYYY-MM`, número da requisição, código de barras, `urgent`, `pending` e o cursor — nunca CPF, CNS, nome, nome da mãe, valor, motivo, nota de restrição, contato ou TOTP. O código de barras da coleta, do recebimento e da remessa vai no corpo do POST.
- Valor de resultado, CPF, nome da mãe, PDF, token do middleware e TOTP nunca em `console`, `localStorage`, `sessionStorage`, Analytics, mensagem de erro ou cache do React Query que sobreviva à tela (`gcTime: 0`). O token do middleware vive só no estado do componente que o pediu e some com "Já guardei" ou ao sair (Review Focus 5). O TOTP vive só no estado do formulário de liberação e é limpo antes de cada envio.
- Toda escrita por cookie com `Content-Type: application/json`, inclusive os `DELETE` (`body: "{}"`). Escrita do admin (`/lab/admin/*`, marcas do prestador e da unidade) com step-up pelo `SensitiveAction`. A liberação avançada manda o TOTP no corpo (`totp`), não abre janela de step-up.
- Médico (CRM) só libera com ICP (o api recusa com 422 `icp_required`); a tela já oferece só ICP quando o conselho do liberador é CRM (`GET /lab/context`, D1), mas não decide sozinha. Laudo ICP nasce `pending_signature` e é assinado em **Pendentes de assinatura** (19b); o 21a só recebe o resultado depois da assinatura.
- Resultado não liberado nunca aparece fora das telas do laboratório: o 21a só mostra `source: lab` depois do laudo `released`, e o laudo substituído só aparece na visão clínica (o api decide; a tela mostra o que vier).
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova registrada em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--ok`, `--rule*`, `--sunken`, `--panel`, `--accent`; `Tag` com `tone`), sem `outline: "none"`, sem `textOverflow` sem `title`, sem `color: "#fff"`. Use os componentes existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `ActionButton`, `ConfirmButton`, `ActionMessage`, `SensitiveAction`, `SegmentedControl`, `UnitPicker`, `PatientLookup`, `formStyles`, os estilos de `src/modules/exams/styles.ts`); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date(NOW21B))` e `vi.useRealTimers()` no `afterEach`; a sessão de teste é criada **depois** do `setSystemTime`.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-21.14..F-21.24** do board #1 para **In Progress** (Task 0); cada card vai a **Done** quando a parte do dashboard dele termina **e** a do api já está Done (os cards são compartilhados; F-21.14 e F-21.19 não têm parte no dashboard além do rótulo dos interruptores).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod21b` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-21b-lab` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod21b`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos antes de cada commit: `npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. O jsdom não tem `URL.createObjectURL`: defina com `Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") })`.
- A prova (Task 20) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5194**, com proxy para o api do 21b na **3042**.

## Review Focus

1. **Leitura com o foco fora do campo, leitura dupla e leitura enquanto a anterior ainda grava.** O técnico lê três tubos seguidos; o leitor manda o mesmo código duas vezes num disparo; alguém clicou num filtro e o foco saiu do campo. Espera-se: nenhuma leitura perdida (fila na ordem), a repetição imediata do mesmo código ignorada com aviso, e "clique no campo para ler" quando o campo não tem foco — sem roubar o foco de volta. Testes: Task 4, "três leituras seguidas: grava as três, na ordem, uma por vez", "o mesmo código duas vezes em 2 s: a segunda é ignorada com aviso" e "sem foco: avisa e não rouba o foco".
2. **Dois liberadores sobre a mesma requisição, ou o item já liberado por outro.** O segundo recebe 409 `invalid_transition`: a tela explica, fecha a assinatura, limpa o TOTP e relê a fila — sem segundo laudo e sem botão velho. Teste: Task 12, "409 invalid_transition na liberação: explica, limpa o código e relê a fila".
3. **TOTP errado ou vencido na assinatura avançada.** O api responde 401 `mfa_required`: a tela diz que o código não valeu, apaga o código digitado e pede o atual; nada é liberado e nada do código vai a console ou cache. Testes: Task 12, "assinatura avançada com código recusado: apaga o código e pede de novo"; Task 13, "código recusado: apaga e pede de novo; 409 devolve a quem chamou".
4. **Valor com vírgula decimal, fora do limite de digitação, ou recusado pelo api.** "13,5" vira "13.5"; "1350" num campo com limite 0–30 é bloqueado antes do envio com o limite na frase; 422 `out_of_entry_range` do api (limite mudou numa versão nova) marca o componente certo e não perde o que foi digitado. Testes: Task 2, "número com vírgula e limite de digitação"; Task 10, "bloqueia fora do limite sem chamar o api" e "422 out_of_entry_range: marca o componente e mantém os valores".
5. **Token do middleware mostrado uma vez.** Depois de "Já guardei" (ou ao sair da tela) o token não está na tela nem no cache do React Query; "Trocar" gera outro e o anterior deixa de valer (o api garante; a tela diz). Teste: Task 16, "token do middleware: aparece uma vez, some com Já guardei e nunca vai ao cache".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.lab.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/labFixtures.ts`, `vite.config.ts` | tipos e cliente HTTP do laboratório; campos novos do 21a, prestador e unidade; interruptores; dados de teste; proxy `/lab` | 1 |
| `src/lib/lab.ts`, `src/lib/lab.test.ts` | regras de tela (rótulos, papéis, leitura, número, limite, prévia da marca, formulários, recusas) | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.lab.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/hooks/useLabCriticalCount.ts`, `src/modules/lab/LabCriticalNotice.tsx` (+ teste), `src/lib/team.ts`, `src/lib/team.test.ts` | grupo "Laboratório", contador e aviso de críticos sem aviso, papéis no convite | 3 |
| `src/modules/lab/ScanField.tsx` (+ teste), `src/modules/lab/openLabPdf.ts` (+ teste), `src/modules/lab/useLabContext.ts`, `src/modules/lab/LabGate.tsx` | leitura por teclado, PDF em janela nova, contexto do laboratório, portão de papel/interruptor | 4 |
| `src/modules/lab/RequisitionForm.tsx` (+ teste) | requisição: paciente, exames candidatos, pedido em papel | 5 |
| `src/modules/lab/LabScope.tsx`, `src/modules/LabRequisitions.tsx` (+ teste), `src/modules/lab/LabRequisitionDetail.tsx` (+ teste), `src/App.tsx` | escopo laboratório/UBS; situação por número ou etiqueta; detalhe, comprovante, etiquetas, entrega | 6 |
| `src/modules/LabCollection.tsx` (+ teste), `src/App.tsx` | coleta por leitura | 7 |
| `src/modules/LabShipments.tsx` (+ teste), `src/modules/lab/ShipmentBuilder.tsx`, `src/modules/lab/ShipmentCheck.tsx`, `src/modules/lab/UnitPendingSamples.tsx`, `src/App.tsx` | remessa da UBS, recoletas e não recebidas; conferência no laboratório | 8 |
| `src/modules/LabReception.tsx` (+ teste), `src/App.tsx` | recebimento: aceitar, rejeitar, aceitar com restrição | 9 |
| `src/modules/lab/ResultEntryForm.tsx` (+ teste), `src/modules/LabEntry.tsx` (+ teste), `src/App.tsx` | digitação: formulário e leitura | 10 |
| `src/modules/LabWorklist.tsx` (+ teste), `src/App.tsx` | mapa de trabalho com filtros, PDF e digitação pelo mapa | 11 |
| `src/modules/LabRelease.tsx` (+ teste), `src/modules/lab/ReleaseReview.tsx`, `src/modules/lab/SignatureChoice.tsx`, `src/modules/lab/CriticalNoticeForm.tsx`, `src/App.tsx` | fila de liberação, repetições, assinatura, críticos sem aviso | 12 |
| `src/modules/lab/LabReportList.tsx`, `src/modules/lab/RectifyForm.tsx` (+ teste), `src/modules/lab/ReleaseReview.tsx`, `src/modules/LabRelease.tsx` | laudos da requisição, PDF e retificação | 13 |
| `src/lib/api.ts`, `src/lib/signature.ts`, `src/lib/signature.lab.test.ts`, `src/modules/SignaturePending.tsx`, `src/modules/SignaturePending.lab.test.tsx` | laudo ICP nas pendências de assinatura | 14 |
| `src/lib/labAdmin.ts` (+ teste), `src/modules/LabAdmin.tsx`, `src/modules/labAdmin/LabExamsPanel.tsx`, `src/modules/labAdmin/VersionForm.tsx` (+ teste), `src/App.tsx` | adotar exame, versões com componentes e faixas | 15 |
| `src/modules/labAdmin/RejectionReasonsPanel.tsx`, `ReleasersPanel.tsx`, `MiddlewareTokenPanel.tsx`, `LabAccessPanels.test.tsx`, `src/modules/LabAdmin.tsx` | motivos de rejeição, liberadores (RT/substituto), token do middleware | 16 |
| `src/lib/examsAdmin.ts`, `src/modules/examsAdmin/ProviderForm.tsx`, `src/modules/examsAdmin/ProvidersPanel.tsx`, `src/modules/examsAdmin/ProviderForm.lab.test.tsx`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitForm.lab.test.tsx` | marcas `uses_municipal_lab` e `collects_samples` | 17 |
| `src/modules/LabReports.tsx` (+ teste), `src/App.tsx` | produção, rejeição e prazo, com CSV | 18 |
| `src/lib/exams.ts`, `src/modules/exams/ExamResultBlock.tsx`, `src/modules/exams/ExamOrderDetail.tsx`, `src/modules/exams/GalRequisitionForm.tsx`, `src/modules/exams/ExamResultBlock.lab.test.tsx`, `src/modules/exams/ExamOrderDetail.lab.test.tsx` | 21a: `source: lab`, laudo substituído, origem `lab_reception`, GAL | 19 |
| — | suíte, build, revisão, prova no navegador | 20 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir (código de barras na URL de escrita, TOTP fora do corpo, nome da mãe ou valor na URL) (Task 1); regras puras com tabela de casos (Tasks 2 e 15); o campo de leitura com eventos de teclado reais (`fireEvent.change` + `fireEvent.keyDown` com `Enter`) e promessas controladas para provar a fila (Task 4); cada tela pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel, cobrindo o caminho feliz, cada recusa que muda a tela (409 `invalid_transition`, 422 de validação, 404 do código, 401 `mfa_required`, `feature_disabled`) e o papel sem acesso (Tasks 5–19); o menu por papel e por interruptor como função pura (Task 3); e a regra "sem polling em rota com trilha" conferida pela ausência de nova chamada depois de avançar o relógio (Tasks 3 e 12). A prova final é no navegador, contra o api real na 3042 com o `fake-middleware` ligado (Task 20).

---

### Task 0: Board, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard; o board #1 (`rotasaude`).
- Produces: cards F-21.14..F-21.24 em In Progress; worktree `$W` na branch `feat/mod-21b-lab`; a base anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json
gh project field-list 1 --owner rotasaude --format json
```

Na saída, ache os itens de título `F-21.14` a `F-21.24` (o `id` `PVTI_…`) e o id da opção **In Progress** do campo Status (`PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`). Para cada card que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id PVTI_xxx --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os onze cards em In Progress. Se algum F-21.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n "^export function isExamLinkExpired\|^async function binaryFetch\|^const postProfessional\|export type SignatureDocumentType\|export interface HealthUnit extends\|^export async function updateUnit\|has_fhir_token: boolean\|source: \"unit\" | \"provider\" | \"fhir\" | \"consultation\""
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "patient_timeline" | "exam_tracking" | "exam_fhir_inbound" | "exam_pec_bridge" | "fake_lab";\|fake_lab: "Laboratório simulado'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '"exam-provider";\|{ label: "Comunicação", items: \[\|const isReception = \|if (item.id === "exam-provider") return false;'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '"Exames":      {\|"exam-provider": {\|IconTruckDelivery, IconUpload,'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'case "exams-admin": return\|const examPending = useExamPending\|export function shellGroups'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '{ role: "exam_regulator", label: "Regulador de exames" },\|const PRIVILEGED_ROLES = new Set'
$G -C $R/apps/dashboard show origin/main:src/lib/signature.ts | grep -n '^export function canSign\|consultation: "consulta", consultation_addendum: "adendo"'
$G -C $R/apps/dashboard show origin/main:src/lib/exams.ts | grep -n '^export const ORIGIN_LABEL\|^export const SOURCE_LABEL\|consultation: "registrado na consulta"'
$G -C $R/apps/dashboard show origin/main:src/lib/examsAdmin.ts | grep -n "^export interface ProviderDraft\|^export function providerBody"
$G -C $R/apps/dashboard show origin/main:src/modules/exams/ExamOrderDetail.tsx | grep -n '{order.result && (\|<KeyValue k="Prestador"'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitForm.tsx | grep -n "export interface UnitFormValue\|{value.screeningScope !== undefined && ("
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/Units.tsx | grep -n "await updateUnit(editing.id\|screeningScope: r.screening_scope"
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"/exams": proxy(TARGET),'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod21b -b feat/mod-21b-lab origin/main
cd $R/.worktrees/dashboard-mod21b && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 20 espera a base mais **27 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 21b** (só informativo; as Tasks 1–19 não dependem dele)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-21b-lab
docker exec -w /rails/.claude/mod21b api-dev bin/rails routes 2>/dev/null | grep -E " /lab/| /exams/orders/:id/gal" | head -80
```

Expected: a branch e as rotas `/lab/*`. Compare com o contrato e com as Divergências deste plano; diferença vira nota para a Task 20.

---

### Task 1: Tipos, cliente HTTP, interruptores, dados de teste e proxy

**Files:**
- Modify: `src/lib/api.ts` (tipo `ExamResult`, tipo `ExamOrder`, `ExamProvider`, `HealthUnit`, `updateUnit`; seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/labFixtures.ts`
- Test: `src/lib/api.lab.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `binaryFetch`, `postProfessional`, `ApiError`, `SignatureFile`, `ExamPriority`, `ExamResult`, `HealthUnit`, `UnitAddress`, `ScreeningScope`, `SessionUser` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`): os tipos `BiologicalSex`, `LabCollectionPlace`, `LabRequisitionStatus`, `LabSampleStatus`, `LabShipmentStatus`, `LabReportStatus`, `LabFlag`, `LabSignatureKind`, `LabItemStatus`, `LabValueType`, `LabReceiveDecision`, `CriticalChannel`, `LabContext`, `LabPatient`, `LabItem`, `LabSample`, `LabReportSummary`, `LabRequisition`, `LabRequisitionRow`, `LabRequisitionPage`, `LabRequisitionQuery`, `LabCandidateOrder`, `LabExamRef`, `LabPaperOrder`, `LabRequisitionInput`, `LabShipmentSample`, `LabShipment`, `LabShipmentInput`, `LabPendingSample`, `LabUnitPending`, `LabReceiveInput`, `LabReceiveResult`, `LabResultRow`, `LabReference`, `LabAnalyteEntry`, `LabEntryItem`, `LabWorklistItem`, `LabWorklistGroup`, `LabWorklistQuery`, `LabReleaseItem`, `LabReleaseRequisition`, `LabCriticalRow`, `LabCriticalInput`, `LabReleaseInput`, `LabRectifyInput`, `LabReferenceRange`, `LabAnalyteTemplate`, `LabTemplate`, `LabAnalyteVersion`, `LabExamVersion`, `LabExam`, `LabExamAdoptInput`, `LabExamSettingsInput`, `LabVersionInput`, `LabRejectionReason`, `LabRejectionReasonInput`, `LabReleaser`, `LabReleaserInput`, `LabReportQuery`, `LabProductionReport`, `LabRejectionRow`, `LabTurnaroundRow`, `LabReportKind`; a constante `LAB_PAGE_SIZE = 50`; as funções (assinaturas exatas no Step 3). `ExamResult.source` ganha `"lab"` e os campos `replaces_result_id?`/`superseded_at?`; `ExamOrder` ganha `superseded_results?` e `gal_requisition?`; `ExamProvider` ganha `uses_municipal_lab?` e `has_middleware_token?`; `ExamProviderInput` ganha `uses_municipal_lab?`; `HealthUnit` ganha `collects_samples?`; `updateUnit` ganha o 6º parâmetro `collectsSamples?: boolean`. `FeatureKey` ganha `municipal_lab`, `lab_hl7_inbound`, `fake_middleware`.
- Produces (em `src/test/labFixtures.ts`): `NOW21B`, `TODAY21B`, `LAB_PROVIDER`, `UBS21B`, `labUser(roles, over)`, `labContext(over)`, `labSample(over)`, `labItem(over)`, `labRequisition(over)`, `labResultRow(over)`, `glucoseEntry(over)`, `lipidEntry(over)`, `releaseRequisition(over)`, `labReport(over)`, `labTemplate(over)`, `labExam(over)`, `labVersion(over)`, `labShipment(over)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.lab.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  collectSampleByBarcode, countUnnotifiedCriticals, createLabRequisition, enterLabResults, fetchLabLabels, fetchLabReportCsv,
  fetchWorklistPdf, getLabEntry, listLabRequisitions, listWorklist, receiveSample, receiveShipment, releaseRequisition,
  revokeMiddlewareToken, rotateMiddlewareToken, setGalRequisition, updateUnit
} from "./api";

function ok(body: unknown): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
}

describe("cliente do laboratório (contrato §3–§6)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => {
    fetchMock = vi.fn(() => ok({ requisitions: [], next_before: null }));
    vi.stubGlobal("fetch", fetchMock);
  });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => fetchMock.mock.calls[i][1] as RequestInit;
  const headersOf = (i = 0) => initOf(i).headers as Record<string, string>;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));

  it("lista de requisições: status[] repetido, laboratório, unidade, número, etiqueta, cursor e limit", async () => {
    await listLabRequisitions({ statuses: [ "open", "typed" ], providerId: "pv9", originUnitId: "u1", number: "2026-000123",
      barcode: "LB0001234", before: "c1" });
    const url = urlOf();
    expect(url.pathname).toBe("/lab/requisitions");
    expect(url.searchParams.getAll("status[]")).toEqual([ "open", "typed" ]);
    expect(url.searchParams.get("provider_id")).toBe("pv9");
    expect(url.searchParams.get("origin_unit_id")).toBe("u1");
    expect(url.searchParams.get("number")).toBe("2026-000123");
    expect(url.searchParams.get("barcode")).toBe("LB0001234");
    expect(url.searchParams.get("before")).toBe("c1");
    expect(url.searchParams.get("limit")).toBe("50");
    expect(initOf().credentials).toBe("include");
  });

  it("requisição nova: nome da mãe e informação clínica só no corpo JSON", async () => {
    fetchMock.mockReturnValueOnce(ok({ id: "rq1" }));
    await createLabRequisition({
      patient_id: "pa1", provider_id: "pv9", origin_unit_id: "u1", collection_place: "ubs", biological_sex: "female",
      mother_name: "Maria Lima", urgent: false, clinical_info: "jejum de 12 h", exam_order_ids: [ "eo1" ],
      paper_orders: [ { sigtap_code: "0202010473", requester: { name: "Dr. Externo", cns: "702002887429583" } } ]
    });
    expect(urlOf().pathname).toBe("/lab/requisitions");
    expect(urlOf().search).toBe("");
    expect(initOf().method).toBe("POST");
    expect(headersOf()["Content-Type"]).toBe("application/json");
    expect(bodyOf().mother_name).toBe("Maria Lima");
    expect(bodyOf().paper_orders[0].requester.cns).toBe("702002887429583");
  });

  it("coleta, recebimento e conferência por leitura: o código vai no corpo, nunca na URL", async () => {
    fetchMock.mockImplementation(() => ok({}));
    await collectSampleByBarcode("LB0001234");
    await receiveSample({ barcode: "LB0001234", decision: "reject", reason_id: "rr1" });
    await receiveShipment("sh1", [ "LB0001234", "LB0001235" ]);
    expect(urlOf(0).pathname).toBe("/lab/samples/collect");
    expect(urlOf(1).pathname).toBe("/lab/samples/receive");
    expect(urlOf(2).pathname).toBe("/lab/shipments/sh1/receive");
    for (const i of [ 0, 1, 2 ]) {
      expect(urlOf(i).search).toBe("");
      expect(initOf(i).method).toBe("POST");
      expect(headersOf(i)["Content-Type"]).toBe("application/json");
    }
    expect(bodyOf(0)).toEqual({ barcode: "LB0001234" });
    expect(bodyOf(1)).toEqual({ barcode: "LB0001234", decision: "reject", reason_id: "rr1" });
    expect(bodyOf(2)).toEqual({ barcodes: [ "LB0001234", "LB0001235" ] });
  });

  it("etiquetas: sample_ids[] na URL e Accept de PDF", async () => {
    fetchMock.mockReturnValueOnce(Promise.resolve(new Response(new Blob([ "%PDF" ]), { status: 200 })));
    await fetchLabLabels("rq1", [ "sa1", "sa2" ]);
    expect(urlOf().pathname).toBe("/lab/requisitions/rq1/labels.pdf");
    expect(urlOf().searchParams.getAll("sample_ids[]")).toEqual([ "sa1", "sa2" ]);
    expect(headersOf().Accept).toBe("application/pdf");
  });

  it("mapa de trabalho: filtros na URL; o PDF usa os mesmos filtros", async () => {
    fetchMock.mockImplementation(() => ok({ groups: [] }));
    const q = { providerId: "pv9", sector: "bioquimica", bench: "B1", urgent: true, date: "2026-10-15", pending: true };
    await listWorklist(q);
    fetchMock.mockReturnValueOnce(Promise.resolve(new Response(new Blob([ "%PDF" ]), { status: 200 })));
    await fetchWorklistPdf(q);
    expect(urlOf(0).pathname).toBe("/lab/worklists");
    expect(urlOf(1).pathname).toBe("/lab/worklists.pdf");
    expect(urlOf(0).search).toBe(urlOf(1).search);
    expect(Object.fromEntries(urlOf(0).searchParams)).toEqual({
      provider_id: "pv9", sector: "bioquimica", bench: "B1", urgent: "true", date: "2026-10-15", pending: "true"
    });
  });

  it("digitação: abre por item ou por etiqueta; valores só no corpo", async () => {
    fetchMock.mockImplementation(() => ok({ items: [] }));
    await getLabEntry({ itemId: "it1" });
    await getLabEntry({ barcode: "LB0001234" });
    await enterLabResults({ item_id: "it1", values: [ { analyte_key: "glicose", value: "98" } ] });
    expect(urlOf(0).searchParams.get("item_id")).toBe("it1");
    expect(urlOf(1).searchParams.get("barcode")).toBe("LB0001234");
    expect(urlOf(2).pathname).toBe("/lab/results");
    expect(urlOf(2).search).toBe("");
    expect(bodyOf(2).values).toEqual([ { analyte_key: "glicose", value: "98" } ]);
  });

  it("liberação: TOTP só na avançada e só no corpo", async () => {
    fetchMock.mockImplementation(() => ok({ id: "lr1" }));
    await releaseRequisition("rq1", { item_ids: [ "it1" ], signature: "icp" });
    await releaseRequisition("rq1", { item_ids: [ "it1" ], signature: "advanced", totp: "123456" });
    expect(bodyOf(0)).toEqual({ item_ids: [ "it1" ], signature: "icp" });
    expect(bodyOf(1)).toEqual({ item_ids: [ "it1" ], signature: "advanced", totp: "123456" });
    expect(urlOf(1).search).toBe("");
  });

  it("contagem de críticos sem aviso: rota própria, sem lista", async () => {
    fetchMock.mockReturnValueOnce(ok({ count: 3 }));
    expect(await countUnnotifiedCriticals()).toBe(3);
    expect(urlOf().pathname).toBe("/lab/criticals/unnotified/count");
  });

  it("token do middleware: POST devolve o token; DELETE manda corpo JSON vazio", async () => {
    fetchMock.mockReturnValueOnce(ok({ token: "mw-secret" }));
    expect(await rotateMiddlewareToken("pv9")).toBe("mw-secret");
    fetchMock.mockReturnValueOnce(Promise.resolve(new Response(null, { status: 204 })));
    await revokeMiddlewareToken("pv9");
    expect(urlOf(1).pathname).toBe("/lab/admin/providers/pv9/middleware_token");
    expect(initOf(1).method).toBe("DELETE");
    expect(initOf(1).body).toBe("{}");
    expect(headersOf(1)["Content-Type"]).toBe("application/json");
  });

  it("relatórios: CSV pela mesma consulta, com Accept text/csv", async () => {
    fetchMock.mockReturnValueOnce(Promise.resolve(new Response(new Blob([ "a\n" ]), { status: 200 })));
    await fetchLabReportCsv("rejections", { providerId: "pv9", month: "2026-10" });
    expect(urlOf().pathname).toBe("/lab/reports/rejections.csv");
    expect(Object.fromEntries(urlOf().searchParams)).toEqual({ provider_id: "pv9", month: "2026-10" });
    expect(headersOf().Accept).toBe("text/csv");
  });

  it("GAL: número só no corpo", async () => {
    fetchMock.mockReturnValueOnce(ok({}));
    await setGalRequisition("eo1", "GAL-2026-77");
    expect(urlOf().pathname).toBe("/exams/orders/eo1/gal_requisition");
    expect(bodyOf()).toEqual({ gal_requisition: "GAL-2026-77" });
  });

  it("unidade: collects_samples só quando informado", async () => {
    fetchMock.mockImplementation(() => ok({ unit: { id: "u1" } }));
    const address = { address_street: null, address_number: null, address_complement: null, address_zip: null, neighborhood_id: null };
    await updateUnit("u1", "UBS Centro", "ubs", address, "walk_in", true);
    await updateUnit("u1", "UBS Centro", "ubs", address, "walk_in");
    expect(bodyOf(0).collects_samples).toBe(true);
    expect("collects_samples" in bodyOf(1)).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/api.lab.test.ts`
Expected: FAIL — `listLabRequisitions` (e as demais) não exportadas de `./api`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, ajuste os tipos do 21a e da unidade (âncoras da Task 0):

```ts
// interface ExamResult — troque a linha do `source` e acrescente os dois campos no fim da interface:
  id: string; origin: "order" | "external"; source: "unit" | "provider" | "fhir" | "consultation" | "lab";
  // Módulo 21b (contrato §9): o resultado que este substitui (retificação) e,
  // nos substituídos, quando deixaram de valer. Só na visão clínica.
  replaces_result_id?: string | null; superseded_at?: string | null;
```

```ts
// interface ExamOrder — acrescente antes do fechamento:
  // Módulo 21b (Divergência D16): laudos substituídos por retificação, do mais
  // novo para o mais antigo; só na visão clínica (ausente na administrativa).
  superseded_results?: ExamResult[];
  // F-21.24 (Divergência D17): número da requisição no GAL; null sem registro.
  gal_requisition?: string | null;
```

```ts
// interface ExamProvider — acrescente:
  // Módulo 21b: o laboratório usa o fluxo analítico do Rota Saúde (resultado
  // pela liberação, nunca pelo portal nem pelo FHIR) e se tem token de middleware.
  uses_municipal_lab?: boolean; has_middleware_token?: boolean;
```

```ts
// type ExamProviderInput — troque por:
export type ExamProviderInput = Pick<ExamProvider, "name" | "cnes" | "kind" | "active"> & { items: ExamProviderItem[]; uses_municipal_lab?: boolean };
```

```ts
// interface HealthUnit — troque por:
export interface HealthUnit extends Partial<UnitAddress> {
  id: string; name: string; kind: string; screening_scope?: ScreeningScope;
  // Módulo 21b: a unidade coleta amostras para o laboratório municipal.
  collects_samples?: boolean;
}
```

```ts
// updateUnit — troque a função inteira por:
export async function updateUnit(
  id: string, name: string, kind: string, address: UnitAddress, screeningScope?: ScreeningScope, collectsSamples?: boolean
): Promise<HealthUnitRow> {
  const body: Record<string, unknown> = { name, kind, ...address };
  if (screeningScope) body.screening_scope = screeningScope;
  if (collectsSamples !== undefined) body.collects_samples = collectsSamples;
  const payload = await jsonFetch<{ unit: HealthUnitRow }>(`${ATTENDANCE_BASE}/units/${encodeURIComponent(id)}`, {
    method: "POST", body: JSON.stringify(body)
  });
  return payload.unit;
}
```

No fim de `src/lib/api.ts`, a seção nova:

```ts
// ─── Laboratório municipal (módulo 21b; contrato 2026-10-15) ────────────────
// Fluxo analítico sobre o 21a: requisição, amostra, remessa, mapa, digitação,
// liberação e laudo. Na URL só ids, situações, setor, bancada, datas, mês,
// número da requisição, código de barras e o cursor; o resto vai no corpo.
const LAB_BASE = import.meta.env.VITE_LAB_BASE || "/lab";

export type BiologicalSex = "male" | "female";
export type LabCollectionPlace = "ubs" | "lab";
export type LabRequisitionStatus = "open" | "in_analysis" | "typed" | "released" | "delivered";
export type LabSampleStatus = "awaiting" | "in_transit" | "received" | "rejected" | "accepted_with_restriction" | "not_received";
export type LabShipmentStatus = "open" | "sent" | "received";
export type LabReportStatus = "pending_signature" | "released" | "rectified";
export type LabFlag = "normal" | "low" | "high" | "critical_low" | "critical_high";
export type LabSignatureKind = "icp" | "advanced";
export type LabItemStatus = "pending" | "typed" | "released" | "cancelled";
export type LabValueType = "numeric" | "text" | "choice";
export type LabReceiveDecision = "accept" | "reject" | "accept_with_restriction";
export type CriticalChannel = "telefone" | "presencial" | "sistema";
export const LAB_PAGE_SIZE = 50;

export interface LabProviderRef { id: string; name: string }
// Divergência D1: laboratórios em que a sessão atua e o conselho do liberador.
export interface LabContext { labs: LabProviderRef[]; releaser_of: string[]; council: string | null }

export interface LabPatient { id: string; display_name: string; birth_date: string; biological_sex: BiologicalSex | string }
export interface LabItem {
  id: string; exam_order_id: string; sigtap_code: string; label: string; version: number;
  sample_id: string | null; status: LabItemStatus | string;
}
export interface LabSample {
  id: string; barcode: string; material: string; container: string; status: LabSampleStatus | string;
  collected_at: string | null; collected_by: string | null; received_at: string | null;
  rejection_reason: string | null; restriction_note: string | null; recollection_of_id: string | null; shipment_id: string | null;
}
export interface LabReportSummary {
  id: string; version: number; partial: boolean; status: LabReportStatus | string; signature_kind: LabSignatureKind | string;
  released_at: string | null; released_by: string | null; short_code: string | null; pdf_url: string | null;
  replaces_report_id: string | null;
}
export interface LabRequisition {
  id: string; number: string; patient: LabPatient; origin_unit: { id: string; name: string };
  collection_place: LabCollectionPlace | string; urgent: boolean; status: LabRequisitionStatus | string;
  expected_delivery_on: string | null; items: LabItem[]; samples: LabSample[]; reports: LabReportSummary[];
  delivered_to: string | null; delivered_at: string | null;
}
export type LabRequisitionRow = Omit<LabRequisition, "items" | "samples">;
export interface LabRequisitionPage { requisitions: LabRequisitionRow[]; next_before: string | null }
export interface LabRequisitionQuery {
  statuses?: LabRequisitionStatus[]; providerId?: string; originUnitId?: string; number?: string; barcode?: string; before?: string | null;
}
// Divergência D2: forma do candidato (exame do 21a sem item de requisição).
export interface LabCandidateOrder {
  id: string; short_code: string; sigtap_code: string; label: string; priority: ExamPriority; status: string; requester_name: string;
}
// Divergência D3: exames ativos do laboratório, para o pedido em papel.
export interface LabExamRef { sigtap_code: string; name: string; material: string; delivery_days: number }
export interface LabPaperOrder { sigtap_code: string; requester: { user_id: string } | { name: string; cns: string } }
export interface LabRequisitionInput {
  patient_id: string; provider_id: string; origin_unit_id: string; collection_place: LabCollectionPlace;
  biological_sex: BiologicalSex; mother_name: string; urgent: boolean; clinical_info?: string;
  exam_order_ids: string[]; paper_orders?: LabPaperOrder[];
}
// Divergência D4: forma da remessa e da amostra dentro dela.
export interface LabShipmentSample { id: string; barcode: string; material: string; status: LabSampleStatus | string; requisition_number: string }
export interface LabShipment {
  id: string; origin_unit: { id: string; name: string }; provider: LabProviderRef; status: LabShipmentStatus | string;
  temperature_celsius: number | null; sent_at: string | null; sent_by: string | null; received_at: string | null;
  received_by: string | null; samples: LabShipmentSample[];
}
export interface LabShipmentInput { origin_unit_id: string; provider_id: string; barcodes: string[]; temperature_celsius: number }
// Divergência D5: recoletas e não recebidas da UBS.
export interface LabPendingSample extends LabSample { requisition: { id: string; number: string; patient_display_name: string } }
export interface LabUnitPending { recollections: LabPendingSample[]; not_received: LabPendingSample[] }
export interface LabReceiveInput { barcode: string; decision: LabReceiveDecision; reason_id?: string; restriction_note?: string }
// Divergência D6: o recebimento devolve a amostra e, na rejeição, a recoleta.
export interface LabReceiveResult { sample: LabSample & { requisition_number: string }; recollection: LabSample | null }

// `notified` só em linha crítica (Divergência D9); `value` só para quem é do laboratório.
export interface LabResultRow {
  id: string; item_id: string; analyte_key: string; name: string; value: string; unit: string | null; flag: LabFlag | string;
  source: "typed" | "hl7" | string; entered_at: string; entered_by: string | null; chosen: boolean; notified?: boolean;
}
export interface LabReference { low: number | null; high: number | null; text: string | null }
// Divergência D7: componente para digitar, com a faixa já escolhida pelo api
// (sexo e idade na data da coleta) e os limites da versão.
export interface LabAnalyteEntry {
  key: string; name: string; unit: string | null; value_type: LabValueType | string; choices: string[] | null; decimals: number;
  reference: LabReference | null; entry_min: number | null; entry_max: number | null;
  critical_low: number | null; critical_high: number | null; formula: string | null;
}
export interface LabEntryItem {
  item_id: string; requisition_id: string; requisition_number: string; barcode: string;
  patient: { display_name: string; birth_date: string; biological_sex: string };
  sigtap_code: string; label: string; version: number; status: LabItemStatus | string; sample_status: LabSampleStatus | string;
  restriction_note: string | null; analytes: LabAnalyteEntry[]; results: LabResultRow[];
}
// Divergência D8: forma do mapa de trabalho.
export interface LabWorklistItem {
  item_id: string; requisition_id: string; requisition_number: string; barcode: string; patient_display_name: string;
  sigtap_code: string; label: string; urgent: boolean; received_at: string | null; status: LabItemStatus | string;
}
export interface LabWorklistGroup { sector: string; bench: string | null; items: LabWorklistItem[] }
export interface LabWorklistQuery { providerId: string; sector?: string; bench?: string; urgent?: boolean; date?: string; pending?: boolean }
// Divergência D9: forma da fila de liberação.
export interface LabReleaseItem extends LabItem { restriction_note: string | null; results: LabResultRow[] }
export interface LabReleaseRequisition {
  id: string; number: string; patient: LabPatient; urgent: boolean; items: LabReleaseItem[]; reports: LabReportSummary[];
}
// Divergência D10: forma da lista de críticos sem aviso (sem valor).
export interface LabCriticalRow {
  result_id: string; requisition_id: string; requisition_number: string; patient_display_name: string;
  analyte_name: string; flag: LabFlag | string; released_at: string | null;
}
export interface LabCriticalInput {
  result_id: string; notified_person: string; notified_role: string; channel: CriticalChannel; notified_at: string;
}
export interface LabReleaseInput { item_ids: string[]; signature: LabSignatureKind; totp?: string }
export interface LabRectifyInput {
  item_ids: string[]; reason: string; contacts: { person: string; channel: CriticalChannel; at: string }[];
  signature: LabSignatureKind; totp?: string;
}

// Admin do laboratório (contrato §3).
export interface LabReferenceRange {
  sex: "male" | "female" | "any"; age_min_days: number | null; age_max_days: number | null;
  low: number | null; high: number | null; text: string | null;
}
export interface LabAnalyteTemplate {
  key: string; name: string; loinc: string | null; unit: string | null; value_type: LabValueType;
  choices?: string[] | null; decimals: number; reference_ranges: LabReferenceRange[];
}
// Divergência D12: forma do catálogo-base, do exame adotado e da versão.
export interface LabTemplate {
  id: string; sigtap_code: string; name: string; sector: string; material: string; container: string;
  suggested_method: string | null; preparation: string | null; analytes: LabAnalyteTemplate[];
}
export interface LabAnalyteVersion extends LabAnalyteTemplate {
  entry_min: number | null; entry_max: number | null; critical_low: number | null; critical_high: number | null; formula: string | null;
}
export interface LabExamVersion {
  id: string; version: number; method: string; material: string; container: string; instrument_test_code: string | null;
  analytes: LabAnalyteVersion[]; created_by: string | null; created_at: string;
}
export interface LabExam {
  id: string; provider_id: string; template_id: string; sigtap_code: string; name: string; sector: string; bench: string | null;
  delivery_days: number; active: boolean; current_version: LabExamVersion | null;
  versions?: { id: string; version: number; created_at: string; created_by: string | null }[];
}
export interface LabExamAdoptInput { provider_id: string; template_id: string; sector: string; bench: string | null; delivery_days: number }
export interface LabExamSettingsInput { sector: string; bench: string | null; delivery_days: number; active: boolean }
export type LabVersionInput = Omit<LabExamVersion, "id" | "version" | "created_by" | "created_at">;
export interface LabRejectionReason { id: string; provider_id: string; name: string; active: boolean }
export interface LabRejectionReasonInput { provider_id: string; name: string; active: boolean }
export interface LabReleaser {
  id: string; user_id: string; name: string; email: string; provider_id: string;
  council: string | null; council_uf: string | null; council_number: string | null; technical_lead: boolean; technical_deputy: boolean;
}
export interface LabReleaserInput { user_id: string; provider_id: string; technical_lead: boolean; technical_deputy: boolean }

// Relatórios (contrato §6; Divergência D13).
export interface LabReportQuery { providerId: string | null; month: string }
export type LabReportKind = "production" | "rejections" | "turnaround";
export interface LabProductionRow {
  origin_unit: { id: string; name: string }; sigtap_code: string; exam_name: string; sector: string;
  releaser: { id: string; name: string } | null; quantity: number; sigtap_value_cents: number;
}
export interface LabProductionReport { rows: LabProductionRow[]; total_quantity: number; total_value_cents: number }
export interface LabRejectionRow { reason: { id: string; name: string }; origin_unit: { id: string; name: string }; count: number }
export interface LabTurnaroundRow { sigtap_code: string; exam_name: string; on_time: number; late: number; pending: number }

const labPath = (...parts: string[]) => [ LAB_BASE, ...parts.map(encodeURIComponent) ].join("/");
const labQuery = (path: string, params: URLSearchParams) => (params.toString() ? `${path}?${params.toString()}` : path);
const labPost = (body: unknown): RequestInit => ({ method: "POST", body: JSON.stringify(body) });

export function getLabContext(): Promise<LabContext> {
  return jsonFetch(labPath("context"));
}

export function listLabRequisitions(q: LabRequisitionQuery): Promise<LabRequisitionPage> {
  const p = new URLSearchParams();
  for (const s of q.statuses ?? []) p.append("status[]", s);
  if (q.providerId) p.set("provider_id", q.providerId);
  if (q.originUnitId) p.set("origin_unit_id", q.originUnitId);
  if (q.number) p.set("number", q.number);
  if (q.barcode) p.set("barcode", q.barcode);
  if (q.before) p.set("before", q.before);
  p.set("limit", String(LAB_PAGE_SIZE));
  return jsonFetch(labQuery(labPath("requisitions"), p));
}

export function getLabRequisition(id: string): Promise<LabRequisition> {
  return jsonFetch(labPath("requisitions", id));
}

export async function listLabCandidates(patientId: string, providerId: string): Promise<LabCandidateOrder[]> {
  const p = new URLSearchParams({ patient_id: patientId, provider_id: providerId });
  return (await jsonFetch<{ orders: LabCandidateOrder[] }>(labQuery(labPath("candidate_orders"), p))).orders;
}

export async function listLabExams(providerId: string): Promise<LabExamRef[]> {
  const p = new URLSearchParams({ provider_id: providerId });
  return (await jsonFetch<{ exams: LabExamRef[] }>(labQuery(labPath("exams"), p))).exams;
}

export function createLabRequisition(input: LabRequisitionInput): Promise<LabRequisition> {
  return jsonFetch(labPath("requisitions"), labPost(input));
}

export function fetchLabReceipt(id: string): Promise<SignatureFile> {
  return binaryFetch(labPath("requisitions", id, "receipt.pdf"), "application/pdf");
}

export function fetchLabLabels(id: string, sampleIds: string[]): Promise<SignatureFile> {
  const p = new URLSearchParams();
  for (const s of sampleIds) p.append("sample_ids[]", s);
  return binaryFetch(labQuery(labPath("requisitions", id, "labels.pdf"), p), "application/pdf");
}

export function collectSample(id: string): Promise<LabSample> {
  return jsonFetch(labPath("samples", id, "collect"), labPost({}));
}

export function collectSampleByBarcode(barcode: string): Promise<LabSample & { requisition_number?: string }> {
  return jsonFetch(labPath("samples", "collect"), labPost({ barcode }));
}

export function createShipment(input: LabShipmentInput): Promise<LabShipment> {
  return jsonFetch(labPath("shipments"), labPost(input));
}

export function sendShipment(id: string): Promise<LabShipment> {
  return jsonFetch(labPath("shipments", id, "send"), labPost({}));
}

export function receiveShipment(id: string, barcodes: string[]): Promise<LabShipment> {
  return jsonFetch(labPath("shipments", id, "receive"), labPost({ barcodes }));
}

export async function listShipments(q: { providerId?: string; originUnitId?: string; statuses: LabShipmentStatus[] }): Promise<LabShipment[]> {
  const p = new URLSearchParams();
  if (q.providerId) p.set("provider_id", q.providerId);
  if (q.originUnitId) p.set("origin_unit_id", q.originUnitId);
  for (const s of q.statuses) p.append("status[]", s);
  return (await jsonFetch<{ shipments: LabShipment[] }>(labQuery(labPath("shipments"), p))).shipments;
}

export function receiveSample(input: LabReceiveInput): Promise<LabReceiveResult> {
  return jsonFetch(labPath("samples", "receive"), labPost(input));
}

export async function listUnitPending(originUnitId: string): Promise<LabUnitPending> {
  const p = new URLSearchParams({ origin_unit_id: originUnitId });
  const body = await jsonFetch<Partial<LabUnitPending>>(labQuery(labPath("recollections"), p));
  return { recollections: body.recollections ?? [], not_received: body.not_received ?? [] };
}

export function deliverRequisition(id: string, deliveredTo: string): Promise<LabRequisition> {
  return jsonFetch(labPath("requisitions", id, "deliver"), labPost({ delivered_to: deliveredTo }));
}

function worklistParams(q: LabWorklistQuery): URLSearchParams {
  const p = new URLSearchParams({ provider_id: q.providerId });
  if (q.sector) p.set("sector", q.sector);
  if (q.bench) p.set("bench", q.bench);
  if (q.urgent) p.set("urgent", "true");
  if (q.date) p.set("date", q.date);
  if (q.pending) p.set("pending", "true");
  return p;
}

export async function listWorklist(q: LabWorklistQuery): Promise<LabWorklistGroup[]> {
  return (await jsonFetch<{ groups: LabWorklistGroup[] }>(labQuery(labPath("worklists"), worklistParams(q)))).groups;
}

export function fetchWorklistPdf(q: LabWorklistQuery): Promise<SignatureFile> {
  return binaryFetch(labQuery(labPath("worklists.pdf"), worklistParams(q)), "application/pdf");
}

export async function getLabEntry(by: { itemId: string } | { barcode: string }): Promise<LabEntryItem[]> {
  const p = "itemId" in by ? new URLSearchParams({ item_id: by.itemId }) : new URLSearchParams({ barcode: by.barcode });
  return (await jsonFetch<{ items: LabEntryItem[] }>(labQuery(labPath("entry"), p))).items;
}

export async function enterLabResults(input: { item_id: string; values: { analyte_key: string; value: string }[] }): Promise<LabResultRow[]> {
  return (await jsonFetch<{ results: LabResultRow[] }>(labPath("results"), labPost(input))).results;
}

export function chooseLabResult(id: string): Promise<LabResultRow> {
  return jsonFetch(labPath("results", id, "choose"), labPost({}));
}

export async function listReleases(providerId: string): Promise<LabReleaseRequisition[]> {
  const p = new URLSearchParams({ provider_id: providerId });
  return (await jsonFetch<{ requisitions: LabReleaseRequisition[] }>(labQuery(labPath("releases"), p))).requisitions;
}

// Divergência D9: a mesma forma para uma requisição só, em qualquer situação
// (conferência e retificação de laudo já liberado).
export function getReleaseRequisition(id: string): Promise<LabReleaseRequisition> {
  return jsonFetch(labPath("releases", id));
}

export async function listUnnotifiedCriticals(): Promise<LabCriticalRow[]> {
  return (await jsonFetch<{ criticals: LabCriticalRow[] }>(labPath("criticals", "unnotified"))).criticals;
}

// Divergência D11: só a contagem, sem dado clínico e sem trilha.
export async function countUnnotifiedCriticals(): Promise<number> {
  const body = await jsonFetch<{ count?: unknown }>(labPath("criticals", "unnotified", "count"));
  const n = Number(body.count ?? 0);
  return Number.isFinite(n) ? n : 0;
}

export async function recordCriticalNotice(input: LabCriticalInput): Promise<void> {
  await jsonFetch<unknown>(labPath("criticals"), labPost(input));
}

export function releaseRequisition(id: string, input: LabReleaseInput): Promise<LabReportSummary> {
  return jsonFetch(labPath("requisitions", id, "release"), labPost(input));
}

export function rectifyLabReport(id: string, input: LabRectifyInput): Promise<LabReportSummary> {
  return jsonFetch(labPath("reports", id, "rectify"), labPost(input));
}

// Admin (`/lab/admin`, municipal_admin; escrita com step-up pelo SensitiveAction).
export async function listLabTemplates(): Promise<LabTemplate[]> {
  return (await jsonFetch<{ templates: LabTemplate[] }>(labPath("admin", "templates"))).templates;
}

export async function listLabAdminExams(providerId: string): Promise<LabExam[]> {
  const p = new URLSearchParams({ provider_id: providerId });
  return (await jsonFetch<{ exams: LabExam[] }>(labQuery(labPath("admin", "exams"), p))).exams;
}

export function getLabAdminExam(id: string): Promise<LabExam> {
  return jsonFetch(labPath("admin", "exams", id));
}

export function adoptLabExam(input: LabExamAdoptInput): Promise<LabExam> {
  return jsonFetch(labPath("admin", "exams"), labPost(input));
}

export function updateLabExam(id: string, input: LabExamSettingsInput): Promise<LabExam> {
  return jsonFetch(labPath("admin", "exams", id), { method: "PATCH", body: JSON.stringify(input) });
}

export function createLabExamVersion(id: string, input: LabVersionInput): Promise<LabExamVersion> {
  return jsonFetch(labPath("admin", "exams", id, "versions"), labPost(input));
}

export async function listRejectionReasons(providerId: string): Promise<LabRejectionReason[]> {
  const p = new URLSearchParams({ provider_id: providerId });
  return (await jsonFetch<{ reasons: LabRejectionReason[] }>(labQuery(labPath("admin", "rejection_reasons"), p))).reasons;
}

// Divergência D14: o recebimento lê os motivos ativos fora do admin.
export async function listActiveRejectionReasons(providerId: string): Promise<LabRejectionReason[]> {
  const p = new URLSearchParams({ provider_id: providerId });
  return (await jsonFetch<{ reasons: LabRejectionReason[] }>(labQuery(labPath("rejection_reasons"), p))).reasons;
}

export function createRejectionReason(input: LabRejectionReasonInput): Promise<LabRejectionReason> {
  return jsonFetch(labPath("admin", "rejection_reasons"), labPost(input));
}

export function updateRejectionReason(id: string, input: { name: string; active: boolean }): Promise<LabRejectionReason> {
  return jsonFetch(labPath("admin", "rejection_reasons", id), { method: "PATCH", body: JSON.stringify(input) });
}

export async function listLabReleasers(): Promise<LabReleaser[]> {
  return (await jsonFetch<{ releasers: LabReleaser[] }>(labPath("admin", "releasers"))).releasers;
}

export function addLabReleaser(input: LabReleaserInput): Promise<LabReleaser> {
  return jsonFetch(labPath("admin", "releasers"), labPost(input));
}

export async function removeLabReleaser(id: string): Promise<void> {
  await jsonFetch<unknown>(labPath("admin", "releasers", id), { method: "DELETE", body: "{}" });
}

// Mostrado uma vez: quem chama guarda só no estado do componente.
export async function rotateMiddlewareToken(providerId: string): Promise<string> {
  return (await jsonFetch<{ token: string }>(labPath("admin", "providers", providerId, "middleware_token"), labPost({}))).token;
}

export async function revokeMiddlewareToken(providerId: string): Promise<void> {
  await jsonFetch<unknown>(labPath("admin", "providers", providerId, "middleware_token"), { method: "DELETE", body: "{}" });
}

function labReportParams(q: LabReportQuery): URLSearchParams {
  const p = new URLSearchParams();
  if (q.providerId) p.set("provider_id", q.providerId);
  p.set("month", q.month);
  return p;
}

export function getLabProduction(q: LabReportQuery): Promise<LabProductionReport> {
  return jsonFetch(labQuery(labPath("reports", "production"), labReportParams(q)));
}

export async function getLabRejections(q: LabReportQuery): Promise<LabRejectionRow[]> {
  return (await jsonFetch<{ rows: LabRejectionRow[] }>(labQuery(labPath("reports", "rejections"), labReportParams(q)))).rows;
}

export async function getLabTurnaround(q: LabReportQuery): Promise<LabTurnaroundRow[]> {
  return (await jsonFetch<{ rows: LabTurnaroundRow[] }>(labQuery(labPath("reports", "turnaround"), labReportParams(q)))).rows;
}

export function fetchLabReportCsv(kind: LabReportKind, q: LabReportQuery): Promise<SignatureFile> {
  return binaryFetch(labQuery(labPath("reports", `${kind}.csv`), labReportParams(q)), "text/csv");
}

// F-21.24 (Divergência D17): número da requisição no GAL do LACEN.
export async function setGalRequisition(orderId: string, value: string): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", orderId, "gal_requisition"), postProfessional({ gal_requisition: value }));
}
```

Em `src/lib/features.ts`, acrescente as três chaves ao tipo e os rótulos:

```ts
export type FeatureKey =
  | "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature" | "clinical_documents" | "controlled_prescriptions"
  | "patient_timeline" | "exam_tracking" | "exam_fhir_inbound" | "exam_pec_bridge" | "fake_lab"
  | "municipal_lab" | "lab_hl7_inbound" | "fake_middleware";
```

```ts
// em FEATURE_LABEL, depois de fake_lab:
  municipal_lab: "Laboratório municipal",
  lab_hl7_inbound: "Entrada de resultados de aparelhos (HL7)",
  fake_middleware: "Middleware de aparelhos simulado (fora de produção)"
```

Em `src/lib/features.test.ts`, acrescente ao `describe` existente:

```ts
  it("rótulos do laboratório municipal (módulo 21b)", () => {
    expect(featureLabel("municipal_lab")).toBe("Laboratório municipal");
    expect(featureLabel("lab_hl7_inbound")).toBe("Entrada de resultados de aparelhos (HL7)");
    expect(featureLabel("fake_middleware")).toBe("Middleware de aparelhos simulado (fora de produção)");
    expect(hasFeature({ features: [ "exam_tracking", "municipal_lab" ] }, "municipal_lab")).toBe(true);
  });
```

(Confira que `featureLabel` e `hasFeature` já estão no `import` do arquivo; se não, acrescente.)

Em `vite.config.ts`, depois de `"/exams": proxy(TARGET),`:

```ts
      // Módulo 21b: laboratório municipal (requisição, amostra, remessa, mapa, digitação, liberação e admin).
      // Regex: não casa /labels nem outro caminho que só comece com "lab".
      "^/lab(/|$)": proxy(TARGET),
```

e, no comentário do topo, depois da linha de `/exams`: `//   /lab        → laboratório municipal (módulo 21b).`

Crie `src/test/labFixtures.ts`:

```ts
// src/test/labFixtures.ts
// Dados comuns aos testes do módulo 21b. Cada arquivo de teste faz o próprio
// vi.mock("…/lib/api"); aqui só tipos e a sessão.
import type {
  HealthUnit, LabContext, LabEntryItem, LabExam, LabExamVersion, LabItem, LabReleaseRequisition, LabReportSummary, LabRequisition,
  LabResultRow, LabSample, LabShipment, LabTemplate, SessionUser
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW21B = "2026-10-15T10:00:00-03:00";
export const TODAY21B = "2026-10-15";
export const LAB_PROVIDER = { id: "pv9", name: "Laboratório Municipal" };
export const UBS21B: HealthUnit = { id: "u1", name: "UBS Centro", kind: "ubs", collects_samples: true };

// Sessão com exames e laboratório ligados; `mfa_verified_at` = agora.
export function labUser(roles: string[] = [ "lab_staff" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, {
    id: "us9", email_address: "lab@curitiba.demo", features: [ "exam_tracking", "municipal_lab" ], ...over
  });
}

export function labContext(over: Partial<LabContext> = {}): LabContext {
  return { labs: [ LAB_PROVIDER ], releaser_of: [], council: null, ...over };
}

export function labSample(over: Partial<LabSample> = {}): LabSample {
  return {
    id: "sa1", barcode: "LB0001234", material: "sangue total", container: "tubo roxo (EDTA)", status: "awaiting",
    collected_at: null, collected_by: null, received_at: null, rejection_reason: null, restriction_note: null,
    recollection_of_id: null, shipment_id: null, ...over
  };
}

export function labItem(over: Partial<LabItem> = {}): LabItem {
  return { id: "it1", exam_order_id: "eo1", sigtap_code: "0202020380", label: "Hemograma completo", version: 2, sample_id: "sa1", status: "pending", ...over };
}

export function labReport(over: Partial<LabReportSummary> = {}): LabReportSummary {
  return {
    id: "lr1", version: 1, partial: false, status: "released", signature_kind: "advanced", released_at: "2026-10-15T09:00:00-03:00",
    released_by: "Bia Farmacêutica", short_code: "QX7M2K9P", pdf_url: "/lab/files/tok-lr1", replaces_report_id: null, ...over
  };
}

export function labRequisition(over: Partial<LabRequisition> = {}): LabRequisition {
  return {
    id: "rq1", number: "2026-000123",
    patient: { id: "pa1", display_name: "Joana Lima", birth_date: "1980-05-02", biological_sex: "female" },
    origin_unit: { id: "u1", name: "UBS Centro" }, collection_place: "ubs", urgent: false, status: "open",
    expected_delivery_on: "2026-10-20", items: [ labItem() ], samples: [ labSample() ], reports: [],
    delivered_to: null, delivered_at: null, ...over
  };
}

export function labResultRow(over: Partial<LabResultRow> = {}): LabResultRow {
  return {
    id: "res1", item_id: "it2", analyte_key: "glicose", name: "Glicose", value: "98", unit: "mg/dL", flag: "normal",
    source: "typed", entered_at: "2026-10-15T08:30:00-03:00", entered_by: "Téc. Carla", chosen: true, ...over
  };
}

export function glucoseEntry(over: Partial<LabEntryItem> = {}): LabEntryItem {
  return {
    item_id: "it2", requisition_id: "rq1", requisition_number: "2026-000123", barcode: "LB0001235",
    patient: { display_name: "Joana Lima", birth_date: "1980-05-02", biological_sex: "female" },
    sigtap_code: "0202010473", label: "Glicose", version: 1, status: "pending", sample_status: "received", restriction_note: null,
    analytes: [ {
      key: "glicose", name: "Glicose", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
      reference: { low: 70, high: 99, text: null }, entry_min: 10, entry_max: 1500, critical_low: 40, critical_high: 400, formula: null
    } ],
    results: [], ...over
  };
}

// Colesterol: total e HDL digitados, não-HDL calculado pelo api.
export function lipidEntry(over: Partial<LabEntryItem> = {}): LabEntryItem {
  return glucoseEntry({
    item_id: "it3", label: "Colesterol total e frações", sigtap_code: "0202010295",
    analytes: [
      { key: "ct", name: "Colesterol total", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
        reference: { low: null, high: 190, text: "desejável < 190" }, entry_min: 20, entry_max: 1000, critical_low: null, critical_high: null, formula: null },
      { key: "hdl", name: "HDL", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
        reference: { low: 40, high: null, text: "> 40" }, entry_min: 5, entry_max: 300, critical_low: null, critical_high: null, formula: null },
      { key: "nao_hdl", name: "Não-HDL (calculado)", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
        reference: { low: null, high: 160, text: null }, entry_min: null, entry_max: null, critical_low: null, critical_high: null,
        formula: "ct - hdl" }
    ], ...over
  });
}

export function releaseRequisition(over: Partial<LabReleaseRequisition> = {}): LabReleaseRequisition {
  return {
    id: "rq1", number: "2026-000123",
    patient: { id: "pa1", display_name: "Joana Lima", birth_date: "1980-05-02", biological_sex: "female" }, urgent: false,
    items: [ { ...labItem({ id: "it2", label: "Glicose", sigtap_code: "0202010473", status: "typed" }), restriction_note: null,
      results: [ labResultRow() ] } ],
    reports: [], ...over
  };
}

export function labTemplate(over: Partial<LabTemplate> = {}): LabTemplate {
  return {
    id: "tp1", sigtap_code: "0202010473", name: "Glicose", sector: "bioquimica", material: "soro", container: "tubo cinza",
    suggested_method: "enzimático (GOD-PAP)", preparation: "jejum de 8 h",
    analytes: [ { key: "glicose", name: "Glicose", loinc: "2345-7", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
      reference_ranges: [ { sex: "any", age_min_days: 6570, age_max_days: null, low: 70, high: 99, text: null } ] } ],
    ...over
  };
}

export function labVersion(over: Partial<LabExamVersion> = {}): LabExamVersion {
  return {
    id: "ev1", version: 1, method: "enzimático (GOD-PAP)", material: "soro", container: "tubo cinza", instrument_test_code: "GLU",
    analytes: [ { key: "glicose", name: "Glicose", loinc: "2345-7", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
      reference_ranges: [ { sex: "any", age_min_days: 6570, age_max_days: null, low: 70, high: 99, text: null } ],
      entry_min: 10, entry_max: 1500, critical_low: 40, critical_high: 400, formula: null } ],
    created_by: "Admin", created_at: "2026-10-14T10:00:00-03:00", ...over
  };
}

export function labExam(over: Partial<LabExam> = {}): LabExam {
  return {
    id: "le1", provider_id: "pv9", template_id: "tp1", sigtap_code: "0202010473", name: "Glicose", sector: "bioquimica", bench: "B1",
    delivery_days: 2, active: true, current_version: labVersion(), versions: [ { id: "ev1", version: 1, created_at: "2026-10-14T10:00:00-03:00", created_by: "Admin" } ],
    ...over
  };
}

export function labShipment(over: Partial<LabShipment> = {}): LabShipment {
  return {
    id: "sh1", origin_unit: { id: "u1", name: "UBS Centro" }, provider: LAB_PROVIDER, status: "sent", temperature_celsius: 4,
    sent_at: "2026-10-15T07:30:00-03:00", sent_by: "Enf. Rita", received_at: null, received_by: null,
    samples: [
      { id: "sa1", barcode: "LB0001234", material: "sangue total", status: "in_transit", requisition_number: "2026-000123" },
      { id: "sa2", barcode: "LB0001235", material: "soro", status: "in_transit", requisition_number: "2026-000123" }
    ],
    ...over
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/api.lab.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` limpo (os testes antigos de unidade e prestador seguem verdes: os campos novos são opcionais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.lab.test.ts src/lib/features.ts src/lib/features.test.ts src/test/labFixtures.ts vite.config.ts
$G commit -m "feat: add municipal lab HTTP client, feature keys and dev proxy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela do laboratório

**Files:**
- Create: `src/lib/lab.ts`
- Test: `src/lib/lab.test.ts`

**Interfaces:**
- Consumes: os tipos da Task 1; `errorCode`, `ApiError` (`api.ts`); `featureDisabledKey`, `hasFeature` (`features.ts`); `cityLocalIso` (`format.ts`); `isValidCns` (`professionals.ts`); `EXAMS_DISABLED` (`exams.ts`); `Tone` (`theme/tokens`).
- Produces (em `src/lib/lab.ts`):
  - chaves de cache `LAB_CONTEXT_KEY`, `LAB_REQUISITIONS_KEY`, `LAB_REQUISITION_KEY`, `LAB_CANDIDATES_KEY`, `LAB_EXAMS_KEY`, `LAB_SHIPMENTS_KEY`, `LAB_UNIT_PENDING_KEY`, `LAB_WORKLIST_KEY`, `LAB_ENTRY_KEY`, `LAB_RELEASES_KEY`, `LAB_CRITICALS_KEY`, `LAB_CRITICAL_COUNT_KEY`, `LAB_REASONS_KEY`, `LAB_ADMIN_EXAMS_KEY`, `LAB_TEMPLATES_KEY`, `LAB_RELEASERS_KEY`, `LAB_REPORTS_KEY`;
  - frases `LAB_DISABLED`, `LAB_NO_ROLE`, `LAB_NO_LAB`, `BARCODE_INVALID`, `UNIT_DOES_NOT_COLLECT`;
  - `interface LabRoles { staff: boolean; releaser: boolean; admin: boolean; professional: boolean }`, `labRoles(user): LabRoles`, `canUseLab(user): boolean`;
  - `requisitionStatusLabel(s: string): string`, `requisitionStatusTone(s: string): Tone | undefined`, `itemStatusLabel(s: string): string`, `sampleStatusLabel(s: { status: string; collected_at: string | null }): string`, `sampleStatusTone(status: string): Tone | undefined`, `shipmentStatusLabel(s: string): string`, `reportStatusLabel(s: string): string`, `flagLabel(f: string): string`, `flagTone(f: string): Tone | undefined`, `signatureLabel(k: string): string`, `SECTOR_LABEL`, `sectorLabel(s: string): string`, `COLLECTION_PLACE_LABEL`, `CHANNEL_OPTIONS`, `SEX_OPTIONS`;
  - `normalizeBarcode(raw: string): string | null`, `normalizeNumber(raw: string): string | null`, `entryProblem(a: LabAnalyteEntry, raw: string): string | null`, `previewFlag(a: LabAnalyteEntry, raw: string): LabFlag | null`, `isCritical(flag: string): boolean`;
  - `interface RequisitionDraft`, `EMPTY_REQUISITION`, `requisitionProblem(d): string | null`, `requisitionBody(d, ctx): LabRequisitionInput`;
  - `parseTemperature(raw: string): number | null`, `temperatureProblem(raw: string): string | null`;
  - `interface CriticalDraft`, `EMPTY_CRITICAL`, `criticalProblem(d, nowIso): string | null`, `criticalBody(resultId, d): LabCriticalInput`;
  - `interface ContactDraft`, `RECTIFY_REASON_MIN = 10`, `rectifyProblem(reason, contacts, nowIso): string | null`, `contactsBody(contacts)`;
  - `totpProblem(code: string): string | null`, `onlyIcp(council: string | null): boolean`;
  - `labError(err: unknown): string`, `refreshLab(client: QueryClient): Promise<void>`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/lab.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  BARCODE_INVALID, EMPTY_CRITICAL, EMPTY_REQUISITION, LAB_DISABLED, canUseLab, criticalBody, criticalProblem, entryProblem, flagLabel,
  flagTone, labError, labRoles, normalizeBarcode, normalizeNumber, onlyIcp, parseTemperature, previewFlag, rectifyProblem,
  requisitionBody, requisitionProblem, requisitionStatusLabel, sampleStatusLabel, temperatureProblem, totpProblem
} from "./lab";
import { glucoseEntry, labUser } from "../test/labFixtures";

const GLU = glucoseEntry().analytes[0];
const err = (status: number, body: unknown) => new ApiError(status, body, "x");

describe("rótulos", () => {
  it("situação da requisição na ordem da spec §5.11; desconhecida crua", () => {
    expect([ "open", "in_analysis", "typed", "released", "delivered" ].map(requisitionStatusLabel))
      .toEqual([ "recebida", "em análise", "digitada", "liberada", "entregue" ]);
    expect(requisitionStatusLabel("archived")).toBe("archived");
  });

  it("amostra aguardando: antes e depois da coleta", () => {
    expect(sampleStatusLabel({ status: "awaiting", collected_at: null })).toBe("aguardando coleta");
    expect(sampleStatusLabel({ status: "awaiting", collected_at: "2026-10-15T08:00:00-03:00" })).toBe("coletada");
    expect(sampleStatusLabel({ status: "not_received", collected_at: "x" })).toBe("não chegou");
  });

  it("marca: crítico em vermelho, alto/baixo em alerta, normal sem rótulo", () => {
    expect(flagLabel("critical_high")).toBe("crítico alto");
    expect(flagTone("critical_low")).toBe("down");
    expect(flagTone("high")).toBe("warn");
    expect(flagLabel("normal")).toBe("");
  });
});

describe("papéis e interruptor", () => {
  it("lab_staff, lab_releaser e admin; operador nunca", () => {
    expect(labRoles(labUser([ "lab_staff" ]))).toEqual({ staff: true, releaser: false, admin: false, professional: false });
    expect(labRoles(labUser([ "lab_releaser", "health_professional" ]))).toEqual({ staff: false, releaser: true, admin: false, professional: true });
    expect(labRoles({ operator: true, memberships: [ { role: "lab_staff" } ] }).staff).toBe(false);
  });

  it("sem municipal_lab na sessão, nada do laboratório", () => {
    expect(canUseLab(labUser())).toBe(true);
    expect(canUseLab(labUser([ "lab_staff" ], { features: [ "exam_tracking" ] }))).toBe(false);
  });
});

describe("leitura do código de barras", () => {
  it("apara espaços e o Enter; recusa vazio, espaço no meio e caractere de controle", () => {
    expect(normalizeBarcode("  LB0001234\r\n")).toBe("LB0001234");
    expect(normalizeBarcode("")).toBeNull();
    expect(normalizeBarcode("LB 0001")).toBeNull();
    expect(normalizeBarcode("LB\u0007001")).toBeNull();
    expect(normalizeBarcode("x".repeat(65))).toBeNull();
    expect(BARCODE_INVALID).toBe("código ilegível — leia a etiqueta de novo");
  });
});

describe("número com vírgula e limite de digitação", () => {
  it("vírgula vira ponto; texto não numérico é recusado", () => {
    expect(normalizeNumber("13,5")).toBe("13.5");
    expect(normalizeNumber(" 98 ")).toBe("98");
    expect(normalizeNumber("-2,0")).toBe("-2.0");
    expect(normalizeNumber("1.234,5")).toBeNull();
    expect(normalizeNumber("abc")).toBeNull();
    expect(normalizeNumber("")).toBeNull();
  });

  it("fora de entry_min/entry_max bloqueia com o limite na frase", () => {
    expect(entryProblem(GLU, "98")).toBeNull();
    expect(entryProblem(GLU, "1600")).toBe("fora do limite de digitação (10 a 1500 mg/dL) — confira o valor");
    expect(entryProblem(GLU, "5")).toBe("fora do limite de digitação (10 a 1500 mg/dL) — confira o valor");
    expect(entryProblem(GLU, "nove")).toBe("valor numérico inválido — use dígitos e vírgula");
    expect(entryProblem(GLU, "")).toBeNull();
  });

  it("escolha precisa estar entre as opções; texto livre passa", () => {
    const choice = { ...GLU, value_type: "choice", choices: [ "negativo", "positivo" ], entry_min: null, entry_max: null };
    expect(entryProblem(choice, "positivo")).toBeNull();
    expect(entryProblem(choice, "talvez")).toBe("escolha uma das opções");
    expect(entryProblem({ ...GLU, value_type: "text", entry_min: null, entry_max: null }, "amarelo citrino")).toBeNull();
  });

  it("prévia da marca: crítico antes de alto/baixo; sem faixa, nada", () => {
    expect(previewFlag(GLU, "98")).toBe("normal");
    expect(previewFlag(GLU, "120")).toBe("high");
    expect(previewFlag(GLU, "65")).toBe("low");
    expect(previewFlag(GLU, "450")).toBe("critical_high");
    expect(previewFlag(GLU, "30")).toBe("critical_low");
    expect(previewFlag({ ...GLU, reference: null, critical_low: null, critical_high: null }, "120")).toBeNull();
    expect(previewFlag(GLU, "")).toBeNull();
  });
});

describe("requisição", () => {
  const ok = { ...EMPTY_REQUISITION, patientId: "pa1", biologicalSex: "female" as const, motherName: "Maria Lima", orderIds: [ "eo1" ] };

  it("exige paciente, sexo, nome da mãe e ao menos um exame", () => {
    expect(requisitionProblem({ ...ok, patientId: null })).toBe("busque o paciente pelo CPF");
    expect(requisitionProblem({ ...ok, biologicalSex: "" })).toBe("informe o sexo biológico (a referência depende dele)");
    expect(requisitionProblem({ ...ok, motherName: "  " })).toBe("informe o nome da mãe (identificação da amostra)");
    expect(requisitionProblem({ ...ok, orderIds: [] })).toBe("escolha ao menos um exame");
    expect(requisitionProblem(ok)).toBeNull();
  });

  it("pedido em papel: exame e quem pediu (da rede ou digitado com CNS válido)", () => {
    const paper = (requester: object) => ({ ...ok, orderIds: [], paper: [ { sigtapCode: "0202010473", ...requester } ] });
    expect(requisitionProblem(paper({ mode: "typed", userId: "", name: "Dr. Externo", cns: "123" }))).toBe("pedido em papel 1: CNS de quem pediu inválido");
    expect(requisitionProblem(paper({ mode: "linked", userId: "", name: "", cns: "" }))).toBe("pedido em papel 1: escolha o profissional que pediu");
    expect(requisitionProblem({ ...ok, paper: [ { sigtapCode: "", mode: "linked", userId: "us1", name: "", cns: "" } ] })).toBe("pedido em papel 1: escolha o exame");
  });

  it("corpo: só dígitos no CNS, nome aparado, sem clinical_info vazio", () => {
    const body = requisitionBody({
      ...ok, clinicalInfo: "  ", paper: [ { sigtapCode: "0202010473", mode: "typed", userId: "", name: " Dr. Externo ", cns: "702 0028 8742 9583" } ]
    }, { providerId: "pv9", originUnitId: "u1", place: "ubs" });
    expect(body).toEqual({
      patient_id: "pa1", provider_id: "pv9", origin_unit_id: "u1", collection_place: "ubs", biological_sex: "female",
      mother_name: "Maria Lima", urgent: false, exam_order_ids: [ "eo1" ],
      paper_orders: [ { sigtap_code: "0202010473", requester: { name: "Dr. Externo", cns: "702002887429583" } } ]
    });
  });
});

describe("remessa, crítico, retificação e assinatura", () => {
  it("temperatura em °C com vírgula, de -30 a 50", () => {
    expect(parseTemperature("4,5")).toBe(4.5);
    expect(temperatureProblem("")).toBe("informe a temperatura da caixa em °C");
    expect(temperatureProblem("80")).toBe("temperatura fora do esperado (-30 a 50 °C) — confira o termômetro");
    expect(temperatureProblem("8")).toBeNull();
  });

  it("comunicação do crítico: pessoa, função, canal e hora não futura", () => {
    const now = "2026-10-15T10:00:00-03:00";
    const d = { ...EMPTY_CRITICAL, person: "Dra. Ana", role: "médica da UBS", channel: "telefone" as const, at: "2026-10-15T09:40" };
    expect(criticalProblem({ ...d, person: "" }, now)).toBe("informe quem foi avisado");
    expect(criticalProblem({ ...d, at: "2026-10-15T11:00" }, now)).toBe("a hora do aviso não pode ser no futuro");
    expect(criticalProblem(d, now)).toBeNull();
    expect(criticalBody("res1", d)).toEqual({
      result_id: "res1", notified_person: "Dra. Ana", notified_role: "médica da UBS", channel: "telefone", notified_at: "2026-10-15T09:40:00-03:00"
    });
  });

  it("retificação: motivo de 10+ caracteres e ao menos um contato completo", () => {
    const now = "2026-10-15T10:00:00-03:00";
    const c = { person: "Dra. Ana", channel: "telefone" as const, at: "2026-10-15T09:00" };
    expect(rectifyProblem("erro", [ c ], now)).toBe("escreva o motivo da retificação (ao menos 10 caracteres)");
    expect(rectifyProblem("valor digitado trocado", [], now)).toBe("registre quem foi avisado da retificação");
    expect(rectifyProblem("valor digitado trocado", [ { ...c, person: "" } ], now)).toBe("contato 1: informe quem foi avisado");
    expect(rectifyProblem("valor digitado trocado", [ c ], now)).toBeNull();
  });

  it("TOTP de 6 dígitos; CRM só ICP", () => {
    expect(totpProblem("12345")).toBe("digite os 6 dígitos do autenticador");
    expect(totpProblem("123456")).toBeNull();
    expect(onlyIcp("CRM")).toBe(true);
    expect(onlyIcp("CRF")).toBe(false);
    expect(onlyIcp(null)).toBe(false);
  });
});

describe("recusas", () => {
  it("interruptor do laboratório, do rastreamento e códigos do contrato", () => {
    expect(labError(err(403, { error: "feature_disabled", feature: "municipal_lab" }))).toBe(LAB_DISABLED);
    expect(labError(err(403, { error: "feature_disabled", feature: "exam_tracking" }))).toBe("o rastreamento de exames está desligado nesta cidade");
    expect(labError(err(422, { error: "icp_required" }))).toBe("médico libera só com ICP-Brasil — escolha ICP");
    expect(labError(err(422, { error: "out_of_entry_range", analyte_key: "glicose" }))).toBe("valor fora do limite de digitação — confira");
    expect(labError(err(404, { error: "not_found" }))).toBe("etiqueta ou requisição não encontrada — confira o código");
    expect(labError(err(401, { error: "mfa_required" }))).toBe("o código do autenticador não valeu — digite o código atual");
    expect(labError(err(500, { error: "boom" }))).toBe("não foi possível concluir (500: boom) — tente de novo");
    expect(labError(new Error("offline"))).toBe("sem conexão com o servidor — tente de novo");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/lab.test.ts`
Expected: FAIL — `Cannot find module './lab'`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/lab.ts
// Laboratório municipal no painel da cidade (módulo 21b, ADR 0037; contrato
// 2026-10-15). Regras de tela, puras: rótulos, papéis, leitura do código de
// barras, número digitado, prévia da marca, validação e as frases das recusas.
// Quem pode de verdade, a faixa aplicável, a fórmula, a marca final e a
// situação são do api.
import type { QueryClient } from "@tanstack/react-query";
import {
  ApiError, errorCode,
  type BiologicalSex, type CriticalChannel, type LabAnalyteEntry, type LabCollectionPlace, type LabCriticalInput, type LabFlag,
  type LabRequisitionInput
} from "./api";
import { EXAMS_DISABLED } from "./exams";
import { featureDisabledKey, hasFeature } from "./features";
import { cityLocalIso } from "./format";
import { isValidCns } from "./professionals";
import type { Tone } from "../theme/tokens";

export const LAB_CONTEXT_KEY = "labContext";
export const LAB_REQUISITIONS_KEY = "labRequisitions";
export const LAB_REQUISITION_KEY = "labRequisition";
export const LAB_CANDIDATES_KEY = "labCandidates";
export const LAB_EXAMS_KEY = "labExams";
export const LAB_SHIPMENTS_KEY = "labShipments";
export const LAB_UNIT_PENDING_KEY = "labUnitPending";
export const LAB_WORKLIST_KEY = "labWorklist";
export const LAB_ENTRY_KEY = "labEntry";
export const LAB_RELEASES_KEY = "labReleases";
export const LAB_CRITICALS_KEY = "labCriticals";
export const LAB_CRITICAL_COUNT_KEY = "labCriticalCount";
export const LAB_REASONS_KEY = "labReasons";
export const LAB_ADMIN_EXAMS_KEY = "labAdminExams";
export const LAB_TEMPLATES_KEY = "labTemplates";
export const LAB_RELEASERS_KEY = "labReleasers";
export const LAB_REPORTS_KEY = "labReports";

// Tudo o que muda quando uma amostra, um resultado ou um laudo anda.
const MOVING_KEYS = [
  LAB_REQUISITIONS_KEY, LAB_REQUISITION_KEY, LAB_SHIPMENTS_KEY, LAB_UNIT_PENDING_KEY, LAB_WORKLIST_KEY, LAB_ENTRY_KEY,
  LAB_RELEASES_KEY, LAB_CRITICALS_KEY, LAB_CRITICAL_COUNT_KEY
];

export const LAB_DISABLED = "o laboratório municipal está desligado nesta cidade";
export const LAB_NO_ROLE = "seu papel não permite ver esta tela";
export const LAB_NO_LAB = "nenhum laboratório municipal para você — fale com o administrador";
export const BARCODE_INVALID = "código ilegível — leia a etiqueta de novo";
export const UNIT_DOES_NOT_COLLECT = "esta unidade não coleta amostras para o laboratório — o administrador marca isso no cadastro da unidade";

export interface LabRoles { staff: boolean; releaser: boolean; admin: boolean; professional: boolean }

export function labRoles(user: { operator: boolean; memberships?: { role: string }[] } | null): LabRoles {
  const roles = user && !user.operator ? (user.memberships ?? []).map((m) => m.role) : [];
  return {
    staff: roles.includes("lab_staff"),
    releaser: roles.includes("lab_releaser"),
    admin: roles.includes("municipal_admin"),
    professional: roles.includes("health_professional")
  };
}

export function canUseLab(user: { operator: boolean; features?: unknown } | null): boolean {
  return !!user && !user.operator && hasFeature(user, "municipal_lab");
}

const REQUISITION_STATUS: Record<string, string> = {
  open: "recebida", in_analysis: "em análise", typed: "digitada", released: "liberada", delivered: "entregue"
};
const REQUISITION_TONE: Record<string, Tone> = { in_analysis: "info", typed: "warn", released: "ok", delivered: "neutral" };

export function requisitionStatusLabel(s: string): string { return REQUISITION_STATUS[s] ?? s; }
export function requisitionStatusTone(s: string): Tone | undefined { return REQUISITION_TONE[s]; }

const ITEM_STATUS: Record<string, string> = { pending: "aguardando resultado", typed: "digitado", released: "liberado", cancelled: "cancelado" };
export function itemStatusLabel(s: string): string { return ITEM_STATUS[s] ?? s; }

const SAMPLE_STATUS: Record<string, string> = {
  in_transit: "em trânsito", received: "recebida", rejected: "rejeitada", accepted_with_restriction: "aceita com restrição",
  not_received: "não chegou"
};
const SAMPLE_TONE: Record<string, Tone> = { received: "ok", rejected: "down", accepted_with_restriction: "warn", not_received: "down", in_transit: "info" };

export function sampleStatusLabel(s: { status: string; collected_at: string | null }): string {
  if (s.status === "awaiting") return s.collected_at ? "coletada" : "aguardando coleta";
  return SAMPLE_STATUS[s.status] ?? s.status;
}
export function sampleStatusTone(status: string): Tone | undefined { return SAMPLE_TONE[status]; }

const SHIPMENT_STATUS: Record<string, string> = { open: "montando", sent: "enviada", received: "conferida" };
export function shipmentStatusLabel(s: string): string { return SHIPMENT_STATUS[s] ?? s; }

const REPORT_STATUS: Record<string, string> = {
  pending_signature: "aguardando assinatura ICP", released: "liberado", rectified: "retificado (substituído)"
};
export function reportStatusLabel(s: string): string { return REPORT_STATUS[s] ?? s; }

const FLAG: Record<string, string> = { normal: "", low: "baixo", high: "alto", critical_low: "crítico baixo", critical_high: "crítico alto" };
const FLAG_TONE: Record<string, Tone> = { low: "warn", high: "warn", critical_low: "down", critical_high: "down" };
export function flagLabel(f: string): string { return f in FLAG ? FLAG[f] : f; }
export function flagTone(f: string): Tone | undefined { return FLAG_TONE[f]; }
export function isCritical(flag: string): boolean { return flag === "critical_low" || flag === "critical_high"; }

const SIGNATURE: Record<string, string> = { icp: "ICP-Brasil", advanced: "assinatura eletrônica avançada" };
export function signatureLabel(k: string): string { return SIGNATURE[k] ?? k; }

export const SECTOR_LABEL: Record<string, string> = {
  bioquimica: "Bioquímica", hematologia: "Hematologia", imunologia: "Imunologia", urinalise: "Urinálise",
  parasitologia: "Parasitologia", microbiologia: "Microbiologia"
};
export function sectorLabel(s: string): string { return SECTOR_LABEL[s] ?? s; }

export const COLLECTION_PLACE_LABEL: Record<LabCollectionPlace, string> = { ubs: "na UBS", lab: "no posto do laboratório" };
export const CHANNEL_OPTIONS: { value: CriticalChannel; label: string }[] = [
  { value: "telefone", label: "Telefone" }, { value: "presencial", label: "Presencial" }, { value: "sistema", label: "Pelo sistema" }
];
export const SEX_OPTIONS: { value: BiologicalSex; label: string }[] = [ { value: "female", label: "Feminino" }, { value: "male", label: "Masculino" } ];

// Leitor = teclado: o código chega com Enter (às vezes \r\n). Só ASCII
// imprimível, sem espaço, até 64 caracteres (Code 128 da etiqueta).
export function normalizeBarcode(raw: string): string | null {
  const code = raw.trim();
  return /^[\x21-\x7E]{1,64}$/.test(code) ? code : null;
}

// "13,5" → "13.5". Separador de milhar não é aceito (ambíguo na bancada).
export function normalizeNumber(raw: string): string | null {
  const t = raw.trim().replace(",", ".");
  return /^-?\d+(\.\d+)?$/.test(t) ? t : null;
}

const fmtLimit = (n: number) => String(n).replace(".", ",");

export function entryProblem(a: LabAnalyteEntry, raw: string): string | null {
  if (raw.trim() === "") return null;
  if (a.value_type === "choice") return a.choices && a.choices.includes(raw) ? null : "escolha uma das opções";
  if (a.value_type !== "numeric") return null;
  const n = normalizeNumber(raw);
  if (n === null) return "valor numérico inválido — use dígitos e vírgula";
  const v = Number(n);
  if ((a.entry_min !== null && v < a.entry_min) || (a.entry_max !== null && v > a.entry_max)) {
    const range = `${a.entry_min === null ? "—" : fmtLimit(a.entry_min)} a ${a.entry_max === null ? "—" : fmtLimit(a.entry_max)}`;
    return `fora do limite de digitação (${range}${a.unit ? ` ${a.unit}` : ""}) — confira o valor`;
  }
  return null;
}

// Só uma prévia para quem digita; a marca que vale é a que o api devolve.
export function previewFlag(a: LabAnalyteEntry, raw: string): LabFlag | null {
  if (a.value_type !== "numeric") return null;
  const n = normalizeNumber(raw);
  if (n === null) return null;
  const v = Number(n);
  if (a.critical_low !== null && v < a.critical_low) return "critical_low";
  if (a.critical_high !== null && v > a.critical_high) return "critical_high";
  if (!a.reference || (a.reference.low === null && a.reference.high === null)) return null;
  if (a.reference.low !== null && v < a.reference.low) return "low";
  if (a.reference.high !== null && v > a.reference.high) return "high";
  return "normal";
}

export interface PaperDraft { sigtapCode: string; mode: "linked" | "typed"; userId: string; name: string; cns: string }
export interface RequisitionDraft {
  patientId: string | null; biologicalSex: BiologicalSex | ""; motherName: string; urgent: boolean; clinicalInfo: string;
  orderIds: string[]; paper: PaperDraft[];
}
export const EMPTY_REQUISITION: RequisitionDraft = {
  patientId: null, biologicalSex: "", motherName: "", urgent: false, clinicalInfo: "", orderIds: [], paper: []
};
export const EMPTY_PAPER: PaperDraft = { sigtapCode: "", mode: "linked", userId: "", name: "", cns: "" };

export function requisitionProblem(d: RequisitionDraft): string | null {
  if (!d.patientId) return "busque o paciente pelo CPF";
  if (!d.biologicalSex) return "informe o sexo biológico (a referência depende dele)";
  if (!d.motherName.trim()) return "informe o nome da mãe (identificação da amostra)";
  if (d.orderIds.length === 0 && d.paper.length === 0) return "escolha ao menos um exame";
  for (const [ i, p ] of d.paper.entries()) {
    const n = `pedido em papel ${i + 1}`;
    if (!p.sigtapCode) return `${n}: escolha o exame`;
    if (p.mode === "linked" && !p.userId) return `${n}: escolha o profissional que pediu`;
    if (p.mode === "typed" && !p.name.trim()) return `${n}: informe o nome de quem pediu`;
    if (p.mode === "typed" && !isValidCns(p.cns)) return `${n}: CNS de quem pediu inválido`;
  }
  return null;
}

export function requisitionBody(
  d: RequisitionDraft, ctx: { providerId: string; originUnitId: string; place: LabCollectionPlace }
): LabRequisitionInput {
  const info = d.clinicalInfo.trim();
  return {
    patient_id: d.patientId as string, provider_id: ctx.providerId, origin_unit_id: ctx.originUnitId, collection_place: ctx.place,
    biological_sex: d.biologicalSex as BiologicalSex, mother_name: d.motherName.trim(), urgent: d.urgent,
    ...(info ? { clinical_info: info } : {}),
    exam_order_ids: d.orderIds,
    ...(d.paper.length > 0 ? {
      paper_orders: d.paper.map((p) => ({
        sigtap_code: p.sigtapCode,
        requester: p.mode === "linked" ? { user_id: p.userId } : { name: p.name.trim(), cns: p.cns.replace(/\D/g, "") }
      }))
    } : {})
  };
}

export function parseTemperature(raw: string): number | null {
  const n = normalizeNumber(raw);
  return n === null ? null : Number(n);
}

export function temperatureProblem(raw: string): string | null {
  if (raw.trim() === "") return "informe a temperatura da caixa em °C";
  const t = parseTemperature(raw);
  if (t === null) return "temperatura inválida — use dígitos e vírgula";
  if (t < -30 || t > 50) return "temperatura fora do esperado (-30 a 50 °C) — confira o termômetro";
  return null;
}

export interface CriticalDraft { person: string; role: string; channel: CriticalChannel | ""; at: string }
export const EMPTY_CRITICAL: CriticalDraft = { person: "", role: "", channel: "", at: "" };

export function criticalProblem(d: CriticalDraft, nowIso: string): string | null {
  if (!d.person.trim()) return "informe quem foi avisado";
  if (!d.role.trim()) return "informe a função de quem foi avisado";
  if (!d.channel) return "escolha como foi o aviso";
  const at = d.at ? cityLocalIso(d.at) : null;
  if (!at) return "informe a data e a hora do aviso";
  if (new Date(at).getTime() > new Date(nowIso).getTime()) return "a hora do aviso não pode ser no futuro";
  return null;
}

export function criticalBody(resultId: string, d: CriticalDraft): LabCriticalInput {
  return {
    result_id: resultId, notified_person: d.person.trim(), notified_role: d.role.trim(), channel: d.channel as CriticalChannel,
    notified_at: cityLocalIso(d.at) as string
  };
}

export interface ContactDraft { person: string; channel: CriticalChannel | ""; at: string }
export const RECTIFY_REASON_MIN = 10;

export function rectifyProblem(reason: string, contacts: ContactDraft[], nowIso: string): string | null {
  if (reason.trim().length < RECTIFY_REASON_MIN) return `escreva o motivo da retificação (ao menos ${RECTIFY_REASON_MIN} caracteres)`;
  if (contacts.length === 0) return "registre quem foi avisado da retificação";
  for (const [ i, c ] of contacts.entries()) {
    const n = `contato ${i + 1}`;
    if (!c.person.trim()) return `${n}: informe quem foi avisado`;
    if (!c.channel) return `${n}: escolha como foi o aviso`;
    const at = c.at ? cityLocalIso(c.at) : null;
    if (!at) return `${n}: informe a data e a hora`;
    if (new Date(at).getTime() > new Date(nowIso).getTime()) return `${n}: a hora não pode ser no futuro`;
  }
  return null;
}

export function contactsBody(contacts: ContactDraft[]): { person: string; channel: CriticalChannel; at: string }[] {
  return contacts.map((c) => ({ person: c.person.trim(), channel: c.channel as CriticalChannel, at: cityLocalIso(c.at) as string }));
}

export function totpProblem(code: string): string | null {
  return /^\d{6}$/.test(code) ? null : "digite os 6 dígitos do autenticador";
}

// CFM: médico só assina laudo com ICP. O api confere de novo (422 icp_required).
export function onlyIcp(council: string | null): boolean {
  return council === "CRM";
}

const ERRORS: Record<string, string> = {
  invalid_transition: "a situação mudou — a tela foi atualizada",
  not_found: "etiqueta ou requisição não encontrada — confira o código",
  order_not_eligible: "um dos exames não pode entrar na requisição (já está em outra ou mudou de situação)",
  exam_not_in_lab_catalog: "este exame não está no catálogo do laboratório",
  mother_name_required: "informe o nome da mãe (identificação da amostra)",
  biological_sex_required: "informe o sexo biológico (a referência depende dele)",
  out_of_entry_range: "valor fora do limite de digitação — confira",
  unknown_analyte: "componente que não existe na versão deste exame — atualize a tela",
  sample_not_received: "a amostra ainda não foi recebida no laboratório",
  icp_required: "médico libera só com ICP-Brasil — escolha ICP",
  no_active_certificate: "sem certificado ICP ativo — vincule o certificado em Assinatura digital ou use a assinatura avançada",
  mfa_required: "o código do autenticador não valeu — digite o código atual",
  invalid_analytes: "componentes inválidos — confira chave, nome, tipo e casas decimais",
  invalid_reference_ranges: "faixas de referência inválidas — confira sexo, idades e limites",
  invalid_formula: "fórmula inválida — use só as chaves dos componentes e + - * / ( )",
  duplicate_instrument_code: "este código do aparelho já está em outro exame do laboratório",
  council_missing: "o profissional não tem conselho, UF e registro no perfil — complete em Profissionais",
  lead_taken: "o laboratório já tem responsável técnico — tire a marca do atual antes",
  provider_uses_municipal_lab: "este prestador usa o laboratório municipal — o resultado sai pela liberação do laudo",
  missing_role: "seu papel não permite esta ação",
  not_unit_staff: "você não atua nesta unidade",
  unit_does_not_collect: UNIT_DOES_NOT_COLLECT,
  reason_required: "escolha o motivo da rejeição",
  restriction_note_required: "escreva a restrição que vai ao laudo",
  invalid_temperature: "temperatura inválida",
  shipment_empty: "a remessa não tem amostras",
  sample_not_collected: "a amostra ainda não foi coletada",
  sample_in_other_shipment: "a amostra já está em outra remessa",
  wrong_origin_unit: "a amostra é de outra unidade",
  patient_not_found: "nenhum paciente com este CPF na cidade",
  citizen_not_verified: "este CPF ainda não tem cadastro validado — faça a validação presencial no balcão antes",
  invalid_month: "mês inválido",
  link_expired: "o link do laudo expirou — abra de novo",
  too_many_requests: "muitas tentativas seguidas — espere um minuto e tente de novo"
};

export function labError(err: unknown): string {
  const feature = featureDisabledKey(err);
  if (feature === "exam_tracking") return EXAMS_DISABLED;
  if (feature !== null) return LAB_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  if (err instanceof ApiError) return `não foi possível concluir (${err.status}${code ? `: ${code}` : ""}) — tente de novo`;
  return "sem conexão com o servidor — tente de novo";
}

export async function refreshLab(client: QueryClient): Promise<void> {
  await Promise.all(MOVING_KEYS.map((key) => client.invalidateQueries({ queryKey: [ key ] })));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/lab.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/lab.ts src/lib/lab.test.ts
$G commit -m "feat: add municipal lab screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Laboratório" no shell, contador de críticos sem aviso e papéis no convite

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Create: `src/hooks/useLabCriticalCount.ts`, `src/modules/lab/LabCriticalNotice.tsx`
- Test: `src/shell/modules.lab.test.ts`, `src/modules/lab/LabCriticalNotice.test.tsx`

**Interfaces:**
- Consumes: `hasFeature` (`features.ts`); `countUnnotifiedCriticals` (Task 1); `LAB_CRITICAL_COUNT_KEY`, `canUseLab`, `labRoles` (Task 2).
- Produces: `ModuleId` ganha `"lab-requisitions" | "lab-collection" | "lab-shipments" | "lab-reception" | "lab-worklist" | "lab-entry" | "lab-release" | "lab-admin" | "lab-reports"`; grupo `"Laboratório"` em `NAV_GROUPS` (depois de "Exames") e em `CATALOG`; `withLabCriticalCount(groups: NavGroupDef[], count: number): NavGroupDef[]`; `shellGroups(groups, pending, examPending = 0, labCritical = 0)`; `useLabCriticalCount(enabled: boolean, userId: string | null)` (`LAB_CRITICAL_REFRESH_MS = 60_000`); `LabCriticalNotice({ count, onOpen })`; `INVITE_ROLES` e `isPrivilegedRole` com `lab_staff` ("Equipe do laboratório") e `lab_releaser` ("Liberador de laudos"). As telas entram no `renderModule` nas Tasks 6–18; até lá o `default` (Placeholder) cobre o id.

- [ ] **Step 1: Write the failing tests**

```ts
// src/shell/modules.lab.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor, withLabCriticalCount } from "./modules";

const ON = { features: [ "exam_tracking", "municipal_lab" ] };
const userWith = (roles: string[], extra: object = ON) => ({ operator: false, memberships: roles.map((role) => ({ role })), ...extra });
const LAB_IDS = [ "lab-requisitions", "lab-collection", "lab-shipments", "lab-reception", "lab-worklist", "lab-entry", "lab-release", "lab-admin", "lab-reports" ];
const labIds = (user: Parameters<typeof navGroupsFor>[0]) =>
  navGroupsFor(user).flatMap((g) => g.items.map((i) => i.id)).filter((id) => LAB_IDS.includes(id));

describe("grupo Laboratório (módulo 21b)", () => {
  it("fica logo depois de Exames, com as nove telas", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Laboratório")).toBe(labels.indexOf("Exames") + 1);
    expect(NAV_GROUPS.find((g) => g.label === "Laboratório")?.items.map((i) => i.id)).toEqual(LAB_IDS);
  });

  it("sem municipal_lab, nenhuma tela do laboratório para ninguém", () => {
    for (const role of [ "municipal_admin", "health_professional", "lab_staff", "lab_releaser" ]) {
      expect(labIds(userWith([ role ], { features: [ "exam_tracking" ] }))).toEqual([]);
    }
    expect(navGroupsFor(userWith([ "lab_staff" ], {})).some((g) => g.label === "Laboratório")).toBe(false);
  });

  it("cada papel vê o seu", () => {
    expect(labIds(userWith([ "lab_staff" ]))).toEqual([
      "lab-requisitions", "lab-collection", "lab-shipments", "lab-reception", "lab-worklist", "lab-entry"
    ]);
    expect(labIds(userWith([ "lab_releaser" ]))).toEqual([ "lab-worklist", "lab-release", "lab-reports" ]);
    expect(labIds(userWith([ "health_professional" ]))).toEqual([ "lab-requisitions", "lab-collection", "lab-shipments" ]);
    expect(labIds(userWith([ "municipal_admin" ]))).toEqual([ "lab-admin", "lab-reports" ]);
    expect(labIds(userWith([ "citizen_verifier" ]))).toEqual([]);
    expect(labIds({ operator: true, memberships: [], ...ON })).toEqual([]);
  });

  it("o prestador do 21a continua só com o portal e a Segurança", () => {
    expect(navGroupsFor(userWith([ "exam_provider" ])).flatMap((g) => g.items.map((i) => i.id))).toEqual([ "exam-provider", "security" ]);
  });

  it("contador de críticos sem aviso no rótulo da liberação", () => {
    const item = (n: number) => withLabCriticalCount(NAV_GROUPS, n).flatMap((g) => g.items).find((i) => i.id === "lab-release");
    expect(item(0)?.label).toBe("Liberação de laudos");
    expect(item(2)?.label).toBe("Liberação de laudos (2 sem aviso)");
  });
});
```

```tsx
// src/modules/lab/LabCriticalNotice.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { LabCriticalNotice } from "./LabCriticalNotice";

afterEach(cleanup);

describe("LabCriticalNotice", () => {
  it("nada com zero", () => {
    const { container } = render(<LabCriticalNotice count={0} onOpen={vi.fn()} />);
    expect(container.textContent).toBe("");
  });

  it("singular, plural e o botão que leva à liberação", () => {
    const onOpen = vi.fn();
    render(<LabCriticalNotice count={1} onOpen={onOpen} />);
    expect(screen.getByRole("status").textContent).toContain("1 valor crítico liberado sem registro de aviso");
    fireEvent.click(screen.getByRole("button", { name: "Registrar avisos" }));
    expect(onOpen).toHaveBeenCalled();
    cleanup();
    render(<LabCriticalNotice count={3} onOpen={onOpen} />);
    expect(screen.getByRole("status").textContent).toContain("3 valores críticos liberados sem registro de aviso");
  });
});
```

E, em `src/lib/team.test.ts`, troque a expectativa da lista de papéis do convite:

```ts
    expect(INVITE_ROLES.map((r) => r.role).sort()).toEqual([
      "analyst", "campaign_manager", "citizen_verifier", "exam_regulator", "health_professional", "lab_releaser", "lab_staff",
      "municipal_admin", "protocol_author", "protocol_publisher", "protocol_reviewer", "viewer"
    ]);
    expect(INVITE_ROLES.find((r) => r.role === "exam_regulator")?.label).toBe("Regulador de exames");
    expect(INVITE_ROLES.find((r) => r.role === "lab_staff")?.label).toBe("Equipe do laboratório");
    expect(INVITE_ROLES.find((r) => r.role === "lab_releaser")?.label).toBe("Liberador de laudos");
    expect(isPrivilegedRole("lab_staff")).toBe(true);
    expect(isPrivilegedRole("lab_releaser")).toBe(true);
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/shell/modules.lab.test.ts src/modules/lab/LabCriticalNotice.test.tsx src/lib/team.test.ts`
Expected: FAIL — grupo "Laboratório" ausente, `withLabCriticalCount` e `LabCriticalNotice` não existem, papéis fora do convite.

- [ ] **Step 3: Write minimal implementation**

Em `src/shell/modules.ts`:

```ts
// no tipo ModuleId, troque a última linha por:
  | "exam-acknowledgements" | "exams-unit" | "exams-overdue" | "exam-regulation" | "exam-report" | "exams-admin" | "exam-provider"
  | "lab-requisitions" | "lab-collection" | "lab-shipments" | "lab-reception" | "lab-worklist" | "lab-entry" | "lab-release"
  | "lab-admin" | "lab-reports";
```

```ts
// em NAV_GROUPS, logo depois do grupo "Exames" e antes de "Comunicação":
  { label: "Laboratório", items: [
    { id: "lab-requisitions", label: "Requisições e situação", icon: "☰" },
    { id: "lab-collection", label: "Coleta", icon: "⌁" },
    { id: "lab-shipments", label: "Remessas", icon: "⇢" },
    { id: "lab-reception", label: "Recebimento de amostras", icon: "⇲" },
    { id: "lab-worklist", label: "Mapa de trabalho", icon: "▦" },
    { id: "lab-entry", label: "Digitação de resultados", icon: "✎" },
    { id: "lab-release", label: "Liberação de laudos", icon: "✔" },
    { id: "lab-admin", label: "Exames e liberadores do laboratório", icon: "⚙" },
    { id: "lab-reports", label: "Relatórios do laboratório", icon: "▤" }
  ]},
```

```ts
// em navGroupsFor, depois de `const isReception = roles.includes("citizen_verifier");`:
  // Módulo 21b: tudo do laboratório só com `municipal_lab` ligado na sessão.
  // A UBS que coleta (health_professional) requisita, coleta e monta remessa;
  // o api confere a unidade (`unit_does_not_collect`, `not_unit_staff`).
  const labOn = !user?.operator && hasFeature(user, "municipal_lab");
  const isLabStaff = roles.includes("lab_staff");
  const isReleaser = roles.includes("lab_releaser");
```

```ts
// no filtro de itens, antes de `if (item.id === "integrations" || item.id === "cnes")`:
      if (item.id === "lab-requisitions" || item.id === "lab-collection" || item.id === "lab-shipments") return labOn && (isLabStaff || isProfessional);
      if (item.id === "lab-reception" || item.id === "lab-entry") return labOn && isLabStaff;
      if (item.id === "lab-worklist") return labOn && (isLabStaff || isReleaser);
      if (item.id === "lab-release") return labOn && isReleaser;
      if (item.id === "lab-admin") return labOn && isAdmin;
      if (item.id === "lab-reports") return labOn && (isAdmin || isReleaser);
```

```ts
// no fim do arquivo:
// Módulo 21b: críticos liberados sem registro de aviso, no item da liberação.
// O número vem da rota de contagem (sem trilha, D11).
export function withLabCriticalCount(groups: NavGroupDef[], count: number): NavGroupDef[] {
  if (count <= 0) return groups;
  return groups.map((group) => ({
    ...group,
    items: group.items.map((item) => item.id === "lab-release" ? { ...item, label: `${item.label} (${count} sem aviso)` } : item)
  }));
}
```

Em `src/shell/modules.test.ts`, o grupo novo some para o operador e sem sessão (nenhum item passa): troque `NAV_GROUPS.length - 8` por `NAV_GROUPS.length - 9` e `NAV_GROUPS.length - 7` por `NAV_GROUPS.length - 8`, e acrescente `expect(groups.some((g) => g.label === "Laboratório")).toBe(false);` nos dois testes.

Em `src/shell/catalog.ts`, confira primeiro que os ícones existem na versão instalada:

```bash
cd $W && for n in Flask Barcode Truck PackageImport ListDetails Keyboard RosetteDiscountCheck Adjustments ChartBar FileBarcode; do grep -q "Icon$n\b" node_modules/@tabler/icons-react/dist/index.d.ts && echo "ok $n" || echo "FALTA $n"; done
```

Expected: dez `ok`. Um `FALTA` → use, para aquela tela, um ícone já importado no arquivo (`IconTestPipe`, `IconClipboardList`, `IconReportAnalytics`, `IconSettings`) e anote. Depois acrescente os nomes ao `import` de `@tabler/icons-react` e:

```ts
// em groups, depois de "Exames":
    "Laboratório": { icon: IconFlask, description: "Amostras, mapa, digitação e laudos" },
```

```ts
// em screens, depois de "exam-provider":
    "lab-requisitions": { icon: IconFileBarcode, description: "Requisição, etiquetas, entrega e situação" },
    "lab-collection": { icon: IconBarcode, description: "Registrar a coleta lendo a etiqueta" },
    "lab-shipments": { icon: IconTruck, description: "Remessa da UBS e conferência no laboratório" },
    "lab-reception": { icon: IconPackageImport, description: "Aceitar, rejeitar ou aceitar com restrição" },
    "lab-worklist": { icon: IconListDetails, description: "Amostras por setor e bancada, com PDF" },
    "lab-entry": { icon: IconKeyboard, description: "Resultados pelo mapa ou pela etiqueta" },
    "lab-release": { icon: IconRosetteDiscountCheck, description: "Conferir, assinar e liberar laudos" },
    "lab-admin": { icon: IconAdjustments, description: "Exames, versões, motivos e liberadores" },
    "lab-reports": { icon: IconChartBar, description: "Produção, rejeição e prazo de entrega" },
```

Crie `src/hooks/useLabCriticalCount.ts`:

```ts
// src/hooks/useLabCriticalCount.ts
// Contagem de valores críticos liberados sem registro de aviso (módulo 21b;
// Divergência D11). Só a rota /count: a lista grava leitura clínica na
// trilha, então o contador nunca a lê. Uma requisição por minuto; nada fica
// no cache quando ninguém observa (gcTime 0).
import { useQuery } from "@tanstack/react-query";
import { countUnnotifiedCriticals } from "../lib/api";
import { LAB_CRITICAL_COUNT_KEY } from "../lib/lab";

export const LAB_CRITICAL_REFRESH_MS = 60_000;

export function useLabCriticalCount(enabled: boolean, userId: string | null) {
  return useQuery({
    queryKey: [ LAB_CRITICAL_COUNT_KEY, userId ],
    queryFn: () => countUnnotifiedCriticals(),
    enabled,
    refetchInterval: enabled ? LAB_CRITICAL_REFRESH_MS : false,
    gcTime: 0, staleTime: 0, retry: false
  });
}
```

Crie `src/modules/lab/LabCriticalNotice.tsx`:

```tsx
// src/modules/lab/LabCriticalNotice.tsx
// Aviso do Início para o liberador (spec §5.7): crítico liberado sem registro
// de quem foi avisado. Só a contagem; a lista abre na tela de liberação.
import { secondaryButtonStyle } from "../../components/formStyles";
import { box, row } from "../exams/styles";

export function LabCriticalNotice({ count, onOpen }: { count: number; onOpen(): void }) {
  if (count <= 0) return null;
  const text = count === 1
    ? "1 valor crítico liberado sem registro de aviso"
    : `${count} valores críticos liberados sem registro de aviso`;
  return (
    <div role="status" style={{ ...box, borderColor: "var(--down)" }}>
      <div style={row}>
        <strong>{text}</strong>
        <button type="button" style={secondaryButtonStyle} onClick={onOpen}>Registrar avisos</button>
      </div>
    </div>
  );
}
```

Em `src/App.tsx`:

```ts
// imports
import { NAV_GROUPS, labelFor, isExamProvider, navGroupsFor, withExamPendingCount, withLabCriticalCount, withPendingCount, type ModuleId, type NavGroupDef } from "./shell/modules";
import { useLabCriticalCount } from "./hooks/useLabCriticalCount";
import { canUseLab, labRoles } from "./lib/lab";
import { LabCriticalNotice } from "./modules/lab/LabCriticalNotice";
```

```ts
// shellGroups: troque a assinatura e a primeira linha; acrescente o ponto do item:
export function shellGroups(groups: NavGroupDef[], pending: number, examPending = 0, labCritical = 0): ShellGroup[] {
  return withLabCriticalCount(withExamPendingCount(withPendingCount(groups, pending), examPending), labCritical).map((g) => ({
    label: g.label,
    slug: slugOf(g.label),
    items: g.items.map((i) => {
      if (i.id === "signature-pending" && pending > 0) return { id: i.id, label: i.label, count: pending };
      // Módulo 21a: `examPending` acende o item "Resultados aguardando ciência".
      if (i.id === "exam-acknowledgements" && examPending > 0) return { id: i.id, label: i.label, count: examPending };
      // Módulo 21b: críticos sem aviso acendem a liberação.
      if (i.id === "lab-release" && labCritical > 0) return { id: i.id, label: i.label, count: labCritical };
      return { id: i.id, label: i.label };
    })
  }));
}
```

```ts
// em ShellInner, depois de `const deniedCount = …`:
  const labCriticalQuery = useLabCriticalCount(canUseLab(auth.user) && labRoles(auth.user).releaser, auth.user?.id ?? null);
  const labCritical = labCriticalQuery.data ?? 0;
```

```ts
// groups: acrescente labCritical
  const groups = useMemo(
    () => shellGroups(navGroupsFor(auth.user), pending.data?.length ?? 0, examCount, labCritical),
    [ auth.user, pending.data?.length, examCount, labCritical ]
  );
```

```tsx
// home: depois do AcknowledgementNotice
      <LabCriticalNotice count={labCritical} onOpen={() => navigate("lab-release")} />
```

Em `src/lib/team.ts`:

```ts
// em INVITE_ROLES, depois de exam_regulator:
  { role: "lab_staff", label: "Equipe do laboratório" },
  { role: "lab_releaser", label: "Liberador de laudos" },
```

```ts
const PRIVILEGED_ROLES = new Set([
  "municipal_admin", REVIEWER_ROLE, VERIFIER_ROLE, PROFESSIONAL_ROLE, CAMPAIGN_MANAGER_ROLE, "exam_regulator", "lab_staff", "lab_releaser"
]);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/shell src/modules/lab/LabCriticalNotice.test.tsx src/lib/team.test.ts src/App.navigation.test.tsx && npx tsc --noEmit`
Expected: PASS (inclusive `catalog.test.ts`, que cobra ícone e descrição das telas novas).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.lab.test.ts src/shell/catalog.ts src/App.tsx \
  src/hooks/useLabCriticalCount.ts src/modules/lab/LabCriticalNotice.tsx src/modules/lab/LabCriticalNotice.test.tsx src/lib/team.ts src/lib/team.test.ts
$G commit -m "feat: add laboratory menu group, critical count and lab roles to invitations

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Leitura por teclado, PDF em janela nova, contexto do laboratório e portão das telas

**Files:**
- Create: `src/modules/lab/ScanField.tsx`, `src/modules/lab/openLabPdf.ts`, `src/modules/lab/useLabContext.ts`, `src/modules/lab/LabGate.tsx`
- Test: `src/modules/lab/ScanField.test.tsx`, `src/modules/lab/openLabPdf.test.ts`

**Interfaces:**
- Consumes: `getLabContext`, `SignatureFile`, `ApiError` (Task 1); `normalizeBarcode`, `BARCODE_INVALID`, `LAB_CONTEXT_KEY`, `LAB_DISABLED`, `LAB_NO_ROLE`, `LAB_NO_LAB`, `canUseLab`, `labRoles`, `labError` (Task 2); `useAuth`; `POPUP_BLOCKED` (`modules/exams/openExamPdf.ts`).
- Produces:
  - `ScanField({ label, onScan, hint? })` com `onScan(code: string): Promise<ScanOutcome>` e `interface ScanOutcome { ok: boolean; text: string }`; constantes `SCAN_DUPLICATE_MS = 2000`, `SCAN_LOG_SIZE = 8`, `SCAN_FOCUS_HINT = "clique no campo para ler"`, `SCAN_DUPLICATE = "lida duas vezes seguidas — a segunda foi ignorada"`.
  - `openLabPdf(fetchFile: () => Promise<SignatureFile>): Promise<{ ok: true } | { ok: false; text: string }>`.
  - `useLabContext(enabled: boolean)` → `UseQueryResult<LabContext>`.
  - `LabGate({ title, sub, allow, children })` com `allow(roles: LabRoles): boolean` — mostra `LAB_DISABLED` sem o interruptor e `LAB_NO_ROLE` sem o papel; `children` é `(roles: LabRoles) => ReactNode`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/lab/ScanField.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { SCAN_DUPLICATE, SCAN_FOCUS_HINT, ScanField, type ScanOutcome } from "./ScanField";

afterEach(() => { cleanup(); vi.useRealTimers(); });

function read(input: HTMLElement, code: string) {
  fireEvent.change(input, { target: { value: code } });
  fireEvent.keyDown(input, { key: "Enter" });
}

describe("ScanField", () => {
  it("nasce com foco; Enter entrega o código aparado e limpa o campo", async () => {
    const onScan = vi.fn(async (): Promise<ScanOutcome> => ({ ok: true, text: "coleta registrada" }));
    render(<ScanField label="Etiqueta" onScan={onScan} />);
    const input = screen.getByLabelText("Etiqueta") as HTMLInputElement;
    expect(document.activeElement).toBe(input);
    read(input, "  LB0001234 ");
    expect(input.value).toBe("");
    await waitFor(() => expect(onScan).toHaveBeenCalledWith("LB0001234"));
    expect(await screen.findByText("LB0001234 · coleta registrada")).not.toBeNull();
  });

  it("três leituras seguidas: grava as três, na ordem, uma por vez", async () => {
    const resolvers: ((o: ScanOutcome) => void)[] = [];
    const onScan = vi.fn(() => new Promise<ScanOutcome>((r) => { resolvers.push(r); }));
    render(<ScanField label="Etiqueta" onScan={onScan} />);
    const input = screen.getByLabelText("Etiqueta");
    read(input, "A1"); read(input, "A2"); read(input, "A3");
    await waitFor(() => expect(onScan).toHaveBeenCalledTimes(1));
    // A1 gravando, A2 e A3 na fila: as três aparecem "aguardando…".
    expect(screen.getAllByText(/aguardando/).length).toBe(3);
    await act(async () => { resolvers[0]({ ok: true, text: "ok" }); });
    await waitFor(() => expect(onScan).toHaveBeenCalledTimes(2));
    await act(async () => { resolvers[1]({ ok: false, text: "não encontrada" }); });
    await waitFor(() => expect(onScan).toHaveBeenCalledTimes(3));
    await act(async () => { resolvers[2]({ ok: true, text: "ok" }); });
    expect(onScan.mock.calls.map((c) => c[0])).toEqual([ "A1", "A2", "A3" ]);
    expect(await screen.findByText("A2 · não encontrada")).not.toBeNull();
  });

  it("o mesmo código duas vezes em 2 s: a segunda é ignorada com aviso", async () => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-15T10:00:00-03:00"));
    const onScan = vi.fn(async (): Promise<ScanOutcome> => ({ ok: true, text: "ok" }));
    render(<ScanField label="Etiqueta" onScan={onScan} />);
    const input = screen.getByLabelText("Etiqueta");
    read(input, "LB1");
    read(input, "LB1");
    await waitFor(() => expect(onScan).toHaveBeenCalledTimes(1));
    expect(screen.getByText(`LB1 · ${SCAN_DUPLICATE}`)).not.toBeNull();
    vi.setSystemTime(new Date("2026-10-15T10:00:03-03:00"));
    read(input, "LB1");
    await waitFor(() => expect(onScan).toHaveBeenCalledTimes(2));
  });

  it("código ilegível não chama quem grava", async () => {
    const onScan = vi.fn();
    render(<ScanField label="Etiqueta" onScan={onScan} />);
    read(screen.getByLabelText("Etiqueta"), "LB 01");
    expect(await screen.findByText(/código ilegível/)).not.toBeNull();
    expect(onScan).not.toHaveBeenCalled();
  });

  it("falha inesperada vira frase, e a fila segue", async () => {
    const onScan = vi.fn()
      .mockRejectedValueOnce(new Error("rede"))
      .mockResolvedValueOnce({ ok: true, text: "ok" });
    render(<ScanField label="Etiqueta" onScan={onScan} />);
    const input = screen.getByLabelText("Etiqueta");
    read(input, "B1"); read(input, "B2");
    expect(await screen.findByText("B1 · não foi possível registrar — leia de novo")).not.toBeNull();
    expect(await screen.findByText("B2 · ok")).not.toBeNull();
  });

  it("sem foco: avisa e não rouba o foco", async () => {
    render(<><ScanField label="Etiqueta" onScan={vi.fn()} /><button type="button">Filtro</button></>);
    const other = screen.getByRole("button", { name: "Filtro" });
    other.focus();
    fireEvent.blur(screen.getByLabelText("Etiqueta"));
    expect(await screen.findByText(SCAN_FOCUS_HINT)).not.toBeNull();
    expect(document.activeElement).toBe(other);
    fireEvent.click(screen.getByRole("button", { name: "Voltar a ler" }));
    expect(document.activeElement).toBe(screen.getByLabelText("Etiqueta"));
  });
});
```

```ts
// src/modules/lab/openLabPdf.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { ApiError } from "../../lib/api";
import { openLabPdf } from "./openLabPdf";

describe("openLabPdf", () => {
  let win: { location: { href: string }; close: ReturnType<typeof vi.fn>; opener: unknown };
  beforeEach(() => {
    win = { location: { href: "" }, close: vi.fn(), opener: {} };
    vi.spyOn(window, "open").mockReturnValue(win as unknown as Window);
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:pdf-1") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  });
  afterEach(() => vi.restoreAllMocks());

  it("abre a janela no clique e põe o blob nela", async () => {
    const out = await openLabPdf(async () => ({ blob: new Blob([ "%PDF" ]), filename: null }));
    expect(out).toEqual({ ok: true });
    expect(win.location.href).toBe("blob:pdf-1");
    expect(win.opener).toBeNull();
  });

  it("recusa do api fecha a janela e devolve a frase, sem o caminho", async () => {
    const out = await openLabPdf(async () => { throw new ApiError(403, { error: "feature_disabled", feature: "municipal_lab" }, "403 on /lab/requisitions/rq1/labels.pdf"); });
    expect(win.close).toHaveBeenCalled();
    expect(out).toEqual({ ok: false, text: "o laboratório municipal está desligado nesta cidade" });
  });

  it("pop-up bloqueado: não busca o arquivo", async () => {
    vi.spyOn(window, "open").mockReturnValue(null);
    const fetchFile = vi.fn();
    const out = await openLabPdf(fetchFile);
    expect(out.ok).toBe(false);
    expect(fetchFile).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/modules/lab/ScanField.test.tsx src/modules/lab/openLabPdf.test.ts`
Expected: FAIL — módulos inexistentes.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/ScanField.tsx
// Campo de leitura do código de barras (spec §5.2–§5.4). O leitor é um
// teclado: digita o código e manda Enter. As leituras entram numa fila e são
// gravadas uma por vez, na ordem (o leitor é mais rápido que a rede); o mesmo
// código repetido em até 2 s é a mesma etiqueta lida duas vezes e é ignorado.
// O campo nasce com foco, mas nunca o rouba de volta: sem foco, avisa.
// Cada leitura leva a função da hora em que foi lida (ex.: a decisão do
// recebimento escolhida antes da leitura).
import { useEffect, useId, useRef, useState } from "react";
import { BARCODE_INVALID, normalizeBarcode } from "../../lib/lab";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { Tag } from "../../components/Tag";
import { box, fieldLabel, list, muted, row } from "../exams/styles";

export interface ScanOutcome { ok: boolean; text: string }
export const SCAN_DUPLICATE_MS = 2000;
export const SCAN_LOG_SIZE = 8;
export const SCAN_FOCUS_HINT = "clique no campo para ler";
export const SCAN_DUPLICATE = "lida duas vezes seguidas — a segunda foi ignorada";
const SCAN_FAILED = "não foi possível registrar — leia de novo";

interface Entry { id: number; code: string; state: "waiting" | "ok" | "error" | "ignored"; text: string }
interface Job { id: number; code: string; run(code: string): Promise<ScanOutcome> }

export function ScanField({ label, onScan, hint }: { label: string; onScan(code: string): Promise<ScanOutcome>; hint?: string }) {
  const inputId = useId();
  const inputRef = useRef<HTMLInputElement | null>(null);
  const [ value, setValue ] = useState("");
  const [ focused, setFocused ] = useState(true);
  const [ log, setLog ] = useState<Entry[]>([]);
  const queue = useRef<Job[]>([]);
  const running = useRef(false);
  const seq = useRef(0);
  const last = useRef<{ code: string; at: number } | null>(null);

  useEffect(() => { inputRef.current?.focus(); }, []);

  const push = (entry: Entry) => setLog((l) => [ entry, ...l ].slice(0, SCAN_LOG_SIZE));
  const settle = (id: number, state: Entry["state"], text: string) =>
    setLog((l) => l.map((e) => (e.id === id ? { ...e, state, text } : e)));

  async function drain() {
    if (running.current) return;
    running.current = true;
    try {
      while (queue.current.length > 0) {
        const job = queue.current.shift() as Job;
        try {
          const out = await job.run(job.code);
          settle(job.id, out.ok ? "ok" : "error", out.text);
        } catch {
          settle(job.id, "error", SCAN_FAILED);
        }
      }
    } finally {
      running.current = false;
    }
  }

  function submit() {
    const raw = value;
    setValue("");
    const id = ++seq.current;
    const code = normalizeBarcode(raw);
    if (code === null) { push({ id, code: raw.trim() || "—", state: "error", text: BARCODE_INVALID }); return; }
    const now = Date.now();
    if (last.current && last.current.code === code && now - last.current.at < SCAN_DUPLICATE_MS) {
      push({ id, code, state: "ignored", text: SCAN_DUPLICATE });
      return;
    }
    last.current = { code, at: now };
    push({ id, code, state: "waiting", text: "aguardando…" });
    queue.current.push({ id, code, run: onScan });
    void drain();
  }

  return (
    // Sem aria-label na caixa: o rótulo é só do campo (getByLabelText acha um elemento).
    <div style={box}>
      <label htmlFor={inputId} style={fieldLabel}>{label}</label>
      <input
        id={inputId} ref={inputRef} value={value} autoComplete="off" spellCheck={false} inputMode="text" style={inputStyle}
        onChange={(e) => setValue(e.target.value)}
        onKeyDown={(e) => { if (e.key === "Enter") { e.preventDefault(); submit(); } }}
        onFocus={() => setFocused(true)} onBlur={() => setFocused(false)}
      />
      {hint && <p style={muted}>{hint}</p>}
      {!focused && (
        <div style={row}>
          <p role="status" style={muted}>{SCAN_FOCUS_HINT}</p>
          <button type="button" style={secondaryButtonStyle} onClick={() => inputRef.current?.focus()}>Voltar a ler</button>
        </div>
      )}
      {log.length > 0 && (
        <ol aria-label="leituras" aria-live="polite" style={list}>
          {log.map((e) => (
            <li key={e.id}>
              {e.state === "error" && <Tag tone="down">erro</Tag>}
              {e.state === "ignored" && <Tag tone="neutral">repetida</Tag>}
              {e.state === "ok" && <Tag tone="ok">ok</Tag>}{" "}
              <span className="mono">{`${e.code} · ${e.text}`}</span>
            </li>
          ))}
        </ol>
      )}
    </div>
  );
}
```

> Nota para o teste de duplicata: o relógio falso só cobre `Date`, então `Date.now()` anda com `vi.setSystemTime` e as promessas seguem reais.

```ts
// src/modules/lab/openLabPdf.ts
// Comprovante, etiquetas, mapa de trabalho e laudo numa janela nova por um
// endereço `blob:` (nada do conteúdo passa pela URL da aplicação nem pelo
// cache do React Query). A janela abre no clique — depois de um await o
// navegador trataria como pop-up. A mensagem do erro (que traz o caminho)
// nunca sai daqui: só a frase.
import type { SignatureFile } from "../../lib/api";
import { labError } from "../../lib/lab";
import { POPUP_BLOCKED } from "../exams/openExamPdf";

const BLOB_TTL_MS = 60_000;

export async function openLabPdf(fetchFile: () => Promise<SignatureFile>): Promise<{ ok: true } | { ok: false; text: string }> {
  const win = window.open("", "_blank");
  if (!win) return { ok: false, text: POPUP_BLOCKED.replace("o laudo", "o PDF") };
  try { win.opener = null; } catch { /* algumas janelas não deixam */ }
  try {
    const file = await fetchFile();
    const url = URL.createObjectURL(file.blob);
    win.location.href = url;
    setTimeout(() => URL.revokeObjectURL(url), BLOB_TTL_MS);
    return { ok: true };
  } catch (err) {
    win.close();
    return { ok: false, text: labError(err) };
  }
}
```

```ts
// src/modules/lab/useLabContext.ts
// Laboratórios em que a sessão atua e o conselho de quem libera (Divergência
// D1). Sem dado clínico; lido uma vez por tela.
import { useQuery } from "@tanstack/react-query";
import { getLabContext } from "../../lib/api";
import { LAB_CONTEXT_KEY } from "../../lib/lab";

export function useLabContext(enabled: boolean) {
  return useQuery({ queryKey: [ LAB_CONTEXT_KEY ], queryFn: getLabContext, enabled, retry: false, staleTime: 60_000 });
}
```

```tsx
// src/modules/lab/LabGate.tsx
// Portão comum das telas do laboratório: interruptor e papel antes de
// qualquer leitura (o api recusaria de novo; aqui só não se oferece a porta).
import type { ReactNode } from "react";
import { useAuth } from "../../lib/auth";
import { LAB_DISABLED, LAB_NO_ROLE, canUseLab, labRoles, type LabRoles } from "../../lib/lab";
import { PageHeader } from "../../components/PageHeader";
import { EmptyState } from "../../components/EmptyState";
import { stack } from "../exams/styles";

export function LabGate({ title, sub, allow, children }: {
  title: string; sub: string; allow(roles: LabRoles): boolean; children(roles: LabRoles): ReactNode;
}) {
  const { user } = useAuth();
  if (!user) return null;
  const roles = labRoles(user);
  let body: ReactNode;
  if (!canUseLab(user)) body = <EmptyState title={LAB_DISABLED} />;
  else if (!allow(roles)) body = <EmptyState title={LAB_NO_ROLE} />;
  else body = children(roles);
  return (
    <div style={stack}>
      <PageHeader title={title} sub={sub} />
      {body}
    </div>
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/modules/lab/ScanField.test.tsx src/modules/lab/openLabPdf.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/lab/ScanField.tsx src/modules/lab/ScanField.test.tsx src/modules/lab/openLabPdf.ts src/modules/lab/openLabPdf.test.ts \
  src/modules/lab/useLabContext.ts src/modules/lab/LabGate.tsx
$G commit -m "feat: add barcode scan field, lab PDF opener and lab screen gate

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Formulário da requisição — paciente, exames candidatos e pedido em papel

**Files:**
- Create: `src/modules/lab/RequisitionForm.tsx`
- Test: `src/modules/lab/RequisitionForm.test.tsx`

**Interfaces:**
- Consumes: `lookupExamPatient` (via `PatientLookup` do 21a; Divergência D2 estende a rota a `lab_staff`), `listLabCandidates`, `listLabExams`, `listExamRequesters`, `listActiveUnits`, `createLabRequisition`, `errorCode`, `HealthUnit`, `LabRequisition`, `ExamPatient` (Task 1 e 21a); `EMPTY_REQUISITION`, `EMPTY_PAPER`, `SEX_OPTIONS`, `requisitionProblem`, `requisitionBody`, `labError`, `LAB_CANDIDATES_KEY`, `LAB_EXAMS_KEY`, `COLLECTION_PLACE_LABEL` (Task 2); `PatientLookup` (`modules/exams/PatientLookup.tsx`).
- Produces: `RequisitionForm({ place, providerId, originUnit, onDone, onCancel })` — `place: "ubs" | "lab"`; `originUnit: HealthUnit | null` (a UBS fixa; no laboratório, `null` e a tela pede a unidade de origem); `onDone(r: LabRequisition)`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/lab/RequisitionForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), lookupExamPatient: vi.fn(), listLabCandidates: vi.fn(), listLabExams: vi.fn(),
    listExamRequesters: vi.fn(), listActiveUnits: vi.fn(), createLabRequisition: vi.fn() };
});

import * as api from "../../lib/api";
import { RequisitionForm } from "./RequisitionForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { UBS21B, labRequisition, labUser } from "../../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const CANDIDATE = { id: "eo1", short_code: "K7M2Q9XW3P", sigtap_code: "0202020380", label: "Hemograma completo", priority: "routine", status: "awaiting_schedule", requester_name: "Dra. Ana" };

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.lookupExamPatient, api.listLabCandidates, api.listLabExams, api.listExamRequesters,
    api.listActiveUnits, api.createLabRequisition ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "health_professional" ]));
  mocked(api.lookupExamPatient).mockResolvedValue({ id: "pa1", display_name: "Joana Lima", cpf_masked: "***.***.***-25" });
  mocked(api.listLabCandidates).mockResolvedValue([ CANDIDATE ]);
  mocked(api.listLabExams).mockResolvedValue([ { sigtap_code: "0202010473", name: "Glicose", material: "soro", delivery_days: 2 } ]);
  mocked(api.listExamRequesters).mockResolvedValue([ { user_id: "us1", name: "Dra. Ana" } ]);
  mocked(api.createLabRequisition).mockResolvedValue(labRequisition());
});

async function findPatient() {
  fireEvent.change(await screen.findByLabelText("CPF do paciente"), { target: { value: "529.982.247-25" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar paciente" }));
  await screen.findByText(/Paciente: Joana Lima/);
}

describe("RequisitionForm", () => {
  it("UBS: candidatos do 21a, sexo, mãe e envio com a unidade e o local fixos", async () => {
    const onDone = vi.fn();
    renderWithProviders(<RequisitionForm place="ubs" providerId="pv9" originUnit={UBS21B} onDone={onDone} onCancel={vi.fn()} />);
    await findPatient();
    expect(api.listLabCandidates).toHaveBeenCalledWith("pa1", "pv9");
    fireEvent.click(await screen.findByRole("checkbox", { name: /Hemograma completo/ }));
    fireEvent.change(screen.getByLabelText("Sexo biológico"), { target: { value: "female" } });
    fireEvent.change(screen.getByLabelText("Nome da mãe"), { target: { value: "Maria Lima" } });
    fireEvent.click(screen.getByRole("button", { name: "Abrir requisição" }));
    await waitFor(() => expect(api.createLabRequisition).toHaveBeenCalledWith({
      patient_id: "pa1", provider_id: "pv9", origin_unit_id: "u1", collection_place: "ubs", biological_sex: "female",
      mother_name: "Maria Lima", urgent: false, exam_order_ids: [ "eo1" ]
    }));
    expect(onDone).toHaveBeenCalledWith(labRequisition());
  });

  it("sem nome da mãe não envia e diz por quê", async () => {
    renderWithProviders(<RequisitionForm place="ubs" providerId="pv9" originUnit={UBS21B} onDone={vi.fn()} onCancel={vi.fn()} />);
    await findPatient();
    fireEvent.click(await screen.findByRole("checkbox", { name: /Hemograma completo/ }));
    fireEvent.change(screen.getByLabelText("Sexo biológico"), { target: { value: "female" } });
    fireEvent.click(screen.getByRole("button", { name: "Abrir requisição" }));
    expect(screen.getByRole("alert").textContent).toBe("informe o nome da mãe (identificação da amostra)");
    expect(api.createLabRequisition).not.toHaveBeenCalled();
  });

  it("laboratório: pede a unidade de origem; pedido em papel com solicitante digitado", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
    mocked(api.listActiveUnits).mockResolvedValue([ UBS21B, { id: "u2", name: "UBS Norte", kind: "ubs" } ]);
    mocked(api.listLabCandidates).mockResolvedValue([]);
    renderWithProviders(<RequisitionForm place="lab" providerId="pv9" originUnit={null} onDone={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.change(await screen.findByLabelText("Unidade de origem"), { target: { value: "u2" } });
    await findPatient();
    expect(await screen.findByText("nenhum exame do paciente aguardando este laboratório — use o pedido em papel")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Incluir pedido em papel" }));
    const paper = screen.getByRole("group", { name: "Pedido em papel 1" });
    fireEvent.change(within(paper).getByLabelText("Exame"), { target: { value: "0202010473" } });
    fireEvent.click(within(paper).getByLabelText("Digitar quem pediu"));
    fireEvent.change(within(paper).getByLabelText("Nome de quem pediu"), { target: { value: "Dr. Externo" } });
    fireEvent.change(within(paper).getByLabelText("CNS de quem pediu"), { target: { value: "123" } });
    fireEvent.change(screen.getByLabelText("Sexo biológico"), { target: { value: "male" } });
    fireEvent.change(screen.getByLabelText("Nome da mãe"), { target: { value: "Maria Lima" } });
    fireEvent.click(screen.getByRole("button", { name: "Abrir requisição" }));
    expect(screen.getByRole("alert").textContent).toBe("pedido em papel 1: CNS de quem pediu inválido");
    fireEvent.change(within(paper).getByLabelText("CNS de quem pediu"), { target: { value: "702002887429583" } });
    fireEvent.click(screen.getByRole("button", { name: "Abrir requisição" }));
    await waitFor(() => expect(mocked(api.createLabRequisition).mock.calls[0][0]).toMatchObject({
      origin_unit_id: "u2", collection_place: "lab", exam_order_ids: [],
      paper_orders: [ { sigtap_code: "0202010473", requester: { name: "Dr. Externo", cns: "702002887429583" } } ]
    }));
  });

  it("422 order_not_eligible: explica e relê os candidatos", async () => {
    mocked(api.createLabRequisition).mockRejectedValueOnce(new api.ApiError(422, { error: "order_not_eligible" }, "x"));
    renderWithProviders(<RequisitionForm place="ubs" providerId="pv9" originUnit={UBS21B} onDone={vi.fn()} onCancel={vi.fn()} />);
    await findPatient();
    fireEvent.click(await screen.findByRole("checkbox", { name: /Hemograma completo/ }));
    fireEvent.change(screen.getByLabelText("Sexo biológico"), { target: { value: "female" } });
    fireEvent.change(screen.getByLabelText("Nome da mãe"), { target: { value: "Maria Lima" } });
    fireEvent.click(screen.getByRole("button", { name: "Abrir requisição" }));
    expect((await screen.findByRole("alert")).textContent).toContain("um dos exames não pode entrar na requisição");
    await waitFor(() => expect(api.listLabCandidates).toHaveBeenCalledTimes(2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/lab/RequisitionForm.test.tsx`
Expected: FAIL — `./RequisitionForm` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/RequisitionForm.tsx
// Requisição do laboratório (spec §5.1; contrato §4). Junta os exames do 21a
// do paciente para o laboratório (candidatos) e o pedido em papel (vira
// exame `lab_reception` no api). A UBS que coleta abre com a própria unidade
// e `collection_place: ubs`; a recepção do laboratório escolhe a unidade de
// origem e coleta no posto (`lab`). Nome da mãe e sexo identificam a amostra
// e escolhem a referência; vão só no corpo.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  createLabRequisition, errorCode, listActiveUnits, listExamRequesters, listLabCandidates, listLabExams,
  type ExamPatient, type HealthUnit, type LabRequisition
} from "../../lib/api";
import {
  COLLECTION_PLACE_LABEL, EMPTY_PAPER, EMPTY_REQUISITION, LAB_CANDIDATES_KEY, LAB_EXAMS_KEY, SEX_OPTIONS, labError,
  requisitionBody, requisitionProblem, type PaperDraft, type RequisitionDraft
} from "../../lib/lab";
import { PRIORITY_LABEL } from "../../lib/exams";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { PatientLookup } from "../exams/PatientLookup";
import { alert, checkLabel, fieldLabel, fieldset, form, legend, muted, row } from "../exams/styles";

export function RequisitionForm({ place, providerId, originUnit, onDone, onCancel }: {
  place: "ubs" | "lab"; providerId: string; originUnit: HealthUnit | null; onDone(r: LabRequisition): void; onCancel(): void;
}) {
  const queryClient = useQueryClient();
  const [ patient, setPatient ] = useState<ExamPatient | null>(null);
  const [ unitId, setUnitId ] = useState(originUnit?.id ?? "");
  const [ draft, setDraft ] = useState<RequisitionDraft>(EMPTY_REQUISITION);
  const [ error, setError ] = useState<string | null>(null);
  const set = (patch: Partial<RequisitionDraft>) => setDraft((d) => ({ ...d, ...patch }));

  const units = useQuery({ queryKey: [ "activeUnits" ], queryFn: listActiveUnits, enabled: place === "lab", retry: false });
  const candidates = useQuery({
    queryKey: [ LAB_CANDIDATES_KEY, patient?.id ?? null, providerId ], queryFn: () => listLabCandidates(patient?.id as string, providerId),
    enabled: !!patient, gcTime: 0, staleTime: 0, retry: false
  });
  const exams = useQuery({ queryKey: [ LAB_EXAMS_KEY, providerId ], queryFn: () => listLabExams(providerId), retry: false });
  const requesters = useQuery({
    queryKey: [ "examRequesters", unitId ], queryFn: () => listExamRequesters(unitId), enabled: !!unitId && draft.paper.length > 0, retry: false
  });

  function setPaper(i: number, patch: Partial<PaperDraft>) {
    set({ paper: draft.paper.map((p, j) => (j === i ? { ...p, ...patch } : p)) });
  }

  function toggleOrder(id: string, on: boolean) {
    set({ orderIds: on ? [ ...draft.orderIds, id ] : draft.orderIds.filter((x) => x !== id) });
  }

  async function submit() {
    if (!unitId) { setError("escolha a unidade de origem"); return; }
    const problem = requisitionProblem({ ...draft, patientId: patient?.id ?? null });
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      const created = await createLabRequisition(requisitionBody({ ...draft, patientId: patient?.id ?? null }, { providerId, originUnitId: unitId, place }));
      onDone(created);
    } catch (err) {
      setError(labError(err));
      if (errorCode(err) === "order_not_eligible") {
        set({ orderIds: [] });
        await queryClient.invalidateQueries({ queryKey: [ LAB_CANDIDATES_KEY ] });
      }
    }
  }

  return (
    <section aria-label="Nova requisição" style={form}>
      <p style={muted}>{`Coleta ${COLLECTION_PLACE_LABEL[place]}${originUnit ? ` · ${originUnit.name}` : ""}`}</p>
      {place === "lab" && (
        <label style={fieldLabel}>
          Unidade de origem
          <select value={unitId} style={inputStyle} onChange={(e) => setUnitId(e.target.value)}>
            <option value="">—</option>
            {(units.data ?? []).map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
          </select>
        </label>
      )}
      <PatientLookup patient={patient} onFound={(p) => { setPatient(p); set({ orderIds: [] }); }} />
      {patient && (
        <>
          <label style={fieldLabel}>
            Sexo biológico
            <select value={draft.biologicalSex} style={inputStyle} onChange={(e) => set({ biologicalSex: e.target.value as RequisitionDraft["biologicalSex"] })}>
              <option value="">—</option>
              {SEX_OPTIONS.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>
            Nome da mãe
            <input value={draft.motherName} autoComplete="off" style={inputStyle} onChange={(e) => set({ motherName: e.target.value })} />
          </label>
          <label style={checkLabel}>
            <input type="checkbox" checked={draft.urgent} onChange={(e) => set({ urgent: e.target.checked })} />Urgente
          </label>
          <label style={fieldLabel}>
            Informação clínica (opcional)
            <textarea value={draft.clinicalInfo} rows={2} style={inputStyle} onChange={(e) => set({ clinicalInfo: e.target.value })} />
          </label>

          <fieldset style={fieldset}>
            <legend style={legend}>Exames pedidos no Rota Saúde</legend>
            {candidates.isPending && <p className="mono" style={muted}>carregando…</p>}
            {candidates.isError && <p role="alert" style={alert}>{labError(candidates.error)}</p>}
            {candidates.isSuccess && candidates.data.length === 0 && (
              <p style={muted}>nenhum exame do paciente aguardando este laboratório — use o pedido em papel</p>
            )}
            {candidates.data?.map((c) => (
              <label key={c.id} style={checkLabel}>
                <input type="checkbox" checked={draft.orderIds.includes(c.id)} onChange={(e) => toggleOrder(c.id, e.target.checked)} />
                {`${c.label} · ${c.short_code} · ${PRIORITY_LABEL[c.priority] ?? c.priority} · ${c.requester_name}`}
              </label>
            ))}
          </fieldset>

          {draft.paper.map((p, i) => (
            <fieldset key={i} role="group" aria-label={`Pedido em papel ${i + 1}`} style={fieldset}>
              <legend style={legend}>{`Pedido em papel ${i + 1}`}</legend>
              <label style={fieldLabel}>
                Exame
                <select value={p.sigtapCode} style={inputStyle} onChange={(e) => setPaper(i, { sigtapCode: e.target.value })}>
                  <option value="">—</option>
                  {(exams.data ?? []).map((x) => <option key={x.sigtap_code} value={x.sigtap_code}>{`${x.name} (${x.material})`}</option>)}
                </select>
              </label>
              <div style={row}>
                <label style={checkLabel}>
                  <input type="radio" name={`requester-${i}`} checked={p.mode === "linked"} onChange={() => setPaper(i, { mode: "linked" })} />
                  Profissional da rede
                </label>
                <label style={checkLabel}>
                  <input type="radio" name={`requester-${i}`} checked={p.mode === "typed"} onChange={() => setPaper(i, { mode: "typed" })} />
                  Digitar quem pediu
                </label>
              </div>
              {p.mode === "linked" ? (
                <label style={fieldLabel}>
                  Quem pediu
                  <select value={p.userId} style={inputStyle} onChange={(e) => setPaper(i, { userId: e.target.value })}>
                    <option value="">—</option>
                    {(requesters.data ?? []).map((r) => <option key={r.user_id} value={r.user_id}>{r.name}</option>)}
                  </select>
                </label>
              ) : (
                <>
                  <label style={fieldLabel}>Nome de quem pediu<input value={p.name} style={inputStyle} onChange={(e) => setPaper(i, { name: e.target.value })} /></label>
                  <label style={fieldLabel}>CNS de quem pediu<input value={p.cns} inputMode="numeric" style={inputStyle} onChange={(e) => setPaper(i, { cns: e.target.value })} /></label>
                </>
              )}
              <div>
                <button type="button" style={secondaryButtonStyle} onClick={() => set({ paper: draft.paper.filter((_, j) => j !== i) })}>Tirar este pedido</button>
              </div>
            </fieldset>
          ))}
          <div>
            <button type="button" style={secondaryButtonStyle} onClick={() => set({ paper: [ ...draft.paper, EMPTY_PAPER ] })}>Incluir pedido em papel</button>
          </div>
        </>
      )}
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton onClick={submit} disabledReason={patient ? null : "busque o paciente pelo CPF"}>Abrir requisição</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}
```

> O `PatientLookup` do 21a rotula o campo "CPF do paciente", o botão "Buscar paciente" e mostra "Paciente: <nome> · CPF <mascarado>" depois de achar (conferido em `origin/main`). Ele não muda.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/lab/RequisitionForm.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/lab/RequisitionForm.tsx src/modules/lab/RequisitionForm.test.tsx
$G commit -m "feat: add lab requisition form with candidate and paper orders

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Requisições e situação — escopo (laboratório ou UBS), lista, detalhe, comprovante, etiquetas e entrega

**Files:**
- Create: `src/modules/lab/LabScope.tsx`, `src/modules/LabRequisitions.tsx`, `src/modules/lab/LabRequisitionDetail.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabRequisitions.test.tsx`, `src/modules/lab/LabRequisitionDetail.test.tsx`

**Interfaces:**
- Consumes: `listLabRequisitions`, `getLabRequisition`, `fetchLabReceipt`, `fetchLabLabels`, `deliverRequisition`, `listActiveUnits`, `errorCode`, `HealthUnit`, `LabRequisitionRow`, `LabRequisition` (Task 1); `labError`, `refreshLab`, `requisitionStatusLabel`, `requisitionStatusTone`, `sampleStatusLabel`, `sampleStatusTone`, `reportStatusLabel`, `signatureLabel`, `LAB_NO_LAB`, `UNIT_DOES_NOT_COLLECT`, `LAB_REQUISITIONS_KEY`, `LAB_REQUISITION_KEY`, `normalizeBarcode`, `type LabRoles` (Task 2); `LabGate`, `useLabContext`, `openLabPdf` (Task 4); `RequisitionForm` (Task 5); `openExamPdf` (21a); `UnitPicker`; `fmtDay` (`lib/exams`).
- Produces:
  - `type LabScopeValue = { kind: "lab"; providerId: string; providerName: string } | { kind: "ubs"; unit: HealthUnit; providerId: string; providerName: string }` e `LabScope({ roles, children })` — `children(scope: LabScopeValue): ReactNode`. Quem tem `lab_staff` ou `lab_releaser` trabalha no escopo do laboratório; senão, da UBS escolhida no `UnitPicker` (unidade sem `collects_samples === true` → `UNIT_DOES_NOT_COLLECT`).
  - `LabRequisitionDetail({ id, onClose })`.
  - `LabRequisitions()` registrado em `renderModule` como `case "lab-requisitions"`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/LabRequisitions.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listLabRequisitions: vi.fn(), getLabRequisition: vi.fn(),
    listActiveUnits: vi.fn() };
});

import * as api from "../lib/api";
import { LabRequisitions } from "./LabRequisitions";
import { renderWithProviders } from "../test/campaignFixtures";
import { LAB_PROVIDER, UBS21B, labContext, labRequisition, labUser } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const { items: _i, samples: _s, ...ROW } = labRequisition();

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  localStorage.clear();
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.listLabRequisitions, api.getLabRequisition, api.listActiveUnits ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.listLabRequisitions).mockResolvedValue({ requisitions: [ ROW ], next_before: null });
  mocked(api.getLabRequisition).mockResolvedValue(labRequisition());
  mocked(api.listActiveUnits).mockResolvedValue([ UBS21B, { id: "u2", name: "UBS Sul", kind: "ubs", collects_samples: false } ]);
});

describe("LabRequisitions", () => {
  it("laboratório: lista do próprio laboratório e situação pelo número", async () => {
    renderWithProviders(<LabRequisitions />);
    expect(await screen.findByText("2026-000123")).not.toBeNull();
    expect(api.listLabRequisitions).toHaveBeenCalledWith({ providerId: LAB_PROVIDER.id, before: null });
    fireEvent.change(screen.getByLabelText("Número da requisição ou etiqueta"), { target: { value: "2026-000123" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver situação" }));
    await waitFor(() => expect(api.listLabRequisitions).toHaveBeenLastCalledWith({ providerId: "pv9", number: "2026-000123", before: null }));
  });

  it("situação pela leitura da etiqueta: código de etiqueta vai como barcode", async () => {
    renderWithProviders(<LabRequisitions />);
    await screen.findByText("2026-000123");
    fireEvent.change(screen.getByLabelText("Número da requisição ou etiqueta"), { target: { value: "LB0001234" } });
    fireEvent.keyDown(screen.getByLabelText("Número da requisição ou etiqueta"), { key: "Enter" });
    await waitFor(() => expect(api.listLabRequisitions).toHaveBeenLastCalledWith({ providerId: "pv9", barcode: "LB0001234", before: null }));
  });

  it("UBS: escolhe a unidade; unidade que não coleta explica e não lista", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "health_professional" ]));
    renderWithProviders(<LabRequisitions />);
    fireEvent.click(await screen.findByRole("button", { name: "UBS Sul" }));
    expect(await screen.findByText(/esta unidade não coleta amostras/)).not.toBeNull();
    expect(api.listLabRequisitions).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "trocar" }));
    fireEvent.click(await screen.findByRole("button", { name: "UBS Centro" }));
    await waitFor(() => expect(api.listLabRequisitions).toHaveBeenCalledWith({ originUnitId: "u1", before: null }));
  });

  it("sem laboratório no contexto: explica", async () => {
    mocked(api.getLabContext).mockResolvedValue(labContext({ labs: [] }));
    renderWithProviders(<LabRequisitions />);
    expect(await screen.findByText("nenhum laboratório municipal para você — fale com o administrador")).not.toBeNull();
  });

  it("recepção sem papel do laboratório não entra", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "citizen_verifier" ]));
    renderWithProviders(<LabRequisitions />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.getLabContext).not.toHaveBeenCalled();
  });
});
```

```tsx
// src/modules/lab/LabRequisitionDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabRequisition: vi.fn(), fetchLabReceipt: vi.fn(), fetchLabLabels: vi.fn(),
    deliverRequisition: vi.fn() };
});

import * as api from "../../lib/api";
import { LabRequisitionDetail } from "./LabRequisitionDetail";
import { renderWithProviders } from "../../test/campaignFixtures";
import { labReport, labRequisition, labSample, labUser } from "../../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const PDF = { blob: new Blob([ "%PDF" ]), filename: null };

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getLabRequisition, api.fetchLabReceipt, api.fetchLabLabels, api.deliverRequisition ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
  mocked(api.getLabRequisition).mockResolvedValue(labRequisition({
    samples: [ labSample(), labSample({ id: "sa2", barcode: "LB0001235", material: "soro", container: "tubo amarelo", status: "rejected", rejection_reason: "hemolisada" }) ]
  }));
  mocked(api.fetchLabReceipt).mockResolvedValue(PDF);
  mocked(api.fetchLabLabels).mockResolvedValue(PDF);
  vi.spyOn(window, "open").mockReturnValue({ location: { href: "" }, close: vi.fn(), opener: null } as unknown as Window);
  Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") });
  Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
});

describe("LabRequisitionDetail", () => {
  it("mostra situação, prazo previsto, exames e amostras com o motivo da rejeição", async () => {
    renderWithProviders(<LabRequisitionDetail id="rq1" onClose={vi.fn()} />);
    const detail = await screen.findByRole("region", { name: "Requisição 2026-000123" });
    expect(within(detail).getByText("recebida")).not.toBeNull();
    expect(within(detail).getByText("20/10/2026")).not.toBeNull();
    expect(within(detail).getByText("Hemograma completo")).not.toBeNull();
    expect(within(detail).getByText("aguardando coleta")).not.toBeNull();
    expect(within(detail).getByText("rejeitada — hemolisada")).not.toBeNull();
  });

  it("comprovante e etiquetas de todas as amostras", async () => {
    renderWithProviders(<LabRequisitionDetail id="rq1" onClose={vi.fn()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Comprovante" }));
    await waitFor(() => expect(api.fetchLabReceipt).toHaveBeenCalledWith("rq1"));
    fireEvent.click(screen.getByRole("button", { name: "Imprimir etiquetas" }));
    await waitFor(() => expect(api.fetchLabLabels).toHaveBeenCalledWith("rq1", [ "sa1", "sa2" ]));
  });

  it("reimpressão de uma etiqueta pede confirmação (fica na trilha)", async () => {
    renderWithProviders(<LabRequisitionDetail id="rq1" onClose={vi.fn()} />);
    const row = (await screen.findByText("LB0001234")).closest("[role='row']") as HTMLElement;
    fireEvent.click(within(row).getByRole("button", { name: "Reimprimir" }));
    expect(api.fetchLabLabels).not.toHaveBeenCalled();
    expect(screen.getByText("A reimpressão fica registrada. Reimprimir a etiqueta?")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.fetchLabLabels).toHaveBeenCalledWith("rq1", [ "sa1" ]));
  });

  it("entrega do laudo liberado: exige quem retirou; 409 relê", async () => {
    mocked(api.getLabRequisition).mockResolvedValue(labRequisition({ status: "released", reports: [ labReport() ] }));
    mocked(api.deliverRequisition).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    renderWithProviders(<LabRequisitionDetail id="rq1" onClose={vi.fn()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Registrar entrega" }));
    expect(screen.getByRole("alert").textContent).toBe("informe quem retirou o laudo");
    fireEvent.change(screen.getByLabelText("Quem retirou"), { target: { value: "Joana Lima (a própria)" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar entrega" }));
    await waitFor(() => expect(api.deliverRequisition).toHaveBeenCalledWith("rq1", "Joana Lima (a própria)"));
    expect((await screen.findByRole("alert")).textContent).toBe("a situação mudou — a tela foi atualizada");
    await waitFor(() => expect(api.getLabRequisition).toHaveBeenCalledTimes(2));
  });

  it("entregue: mostra quem retirou e quando, sem botão", async () => {
    mocked(api.getLabRequisition).mockResolvedValue(labRequisition({ status: "delivered", delivered_to: "Joana Lima", delivered_at: "2026-10-15T09:30:00-03:00" }));
    renderWithProviders(<LabRequisitionDetail id="rq1" onClose={vi.fn()} />);
    expect(await screen.findByText(/Joana Lima · 15\/10\/2026/)).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Registrar entrega" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/modules/LabRequisitions.test.tsx src/modules/lab/LabRequisitionDetail.test.tsx`
Expected: FAIL — módulos inexistentes.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/LabScope.tsx
// Onde a pessoa trabalha: `lab_staff` e `lab_releaser` no laboratório (escolhe entre os do
// contexto, se houver mais de um); a UBS que coleta, na unidade escolhida no
// UnitPicker. Unidade sem a marca `collects_samples` não abre nada.
import { useState, type ReactNode } from "react";
import type { HealthUnit } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { LAB_NO_LAB, UNIT_DOES_NOT_COLLECT, labError, type LabRoles } from "../../lib/lab";
import { EmptyState } from "../../components/EmptyState";
import { inputStyle } from "../../components/formStyles";
import { UnitPicker } from "../attendance/UnitPicker";
import { useLabContext } from "./useLabContext";
import { alert, fieldLabel, muted, stack } from "../exams/styles";

export type LabScopeValue =
  | { kind: "lab"; providerId: string; providerName: string }
  | { kind: "ubs"; unit: HealthUnit; providerId: string; providerName: string };

export function LabScope({ roles, children }: { roles: LabRoles; children(scope: LabScopeValue): ReactNode }) {
  const { user } = useAuth();
  const context = useLabContext(true);
  const [ providerId, setProviderId ] = useState("");
  const [ unit, setUnit ] = useState<HealthUnit | null>(null);
  if (!user) return null;
  if (context.isPending) return <p className="mono" style={muted}>carregando…</p>;
  if (context.isError) return <p role="alert" style={alert}>{labError(context.error)}</p>;
  const labs = context.data.labs;
  if (labs.length === 0) return <EmptyState title={LAB_NO_LAB} />;
  const lab = labs.find((l) => l.id === providerId) ?? labs[0];
  const labPicker = labs.length > 1 && (
    <label style={fieldLabel}>
      Laboratório
      <select value={lab.id} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
        {labs.map((l) => <option key={l.id} value={l.id}>{l.name}</option>)}
      </select>
    </label>
  );
  if (roles.staff || roles.releaser) {
    return <div style={stack}>{labPicker}{children({ kind: "lab", providerId: lab.id, providerName: lab.name })}</div>;
  }
  return (
    <div style={stack}>
      <UnitPicker userId={user.id} onChange={setUnit} />
      {labPicker}
      {unit && unit.collects_samples !== true && <EmptyState title={UNIT_DOES_NOT_COLLECT} />}
      {unit && unit.collects_samples === true && children({ kind: "ubs", unit, providerId: lab.id, providerName: lab.name })}
    </div>
  );
}
```

```tsx
// src/modules/LabRequisitions.tsx
// Requisições do laboratório e situação (spec §5.1, §5.10, §5.11; contrato §4).
// O laboratório vê as suas; a UBS, as que abriu. A busca aceita o número da
// requisição ou a leitura da etiqueta (o leitor manda Enter). Lista com
// cursor; nada fica no cache (gcTime 0).
import { useState } from "react";
import { useInfiniteQuery } from "@tanstack/react-query";
import { listLabRequisitions, type LabRequisitionRow } from "../lib/api";
import { LAB_REQUISITIONS_KEY, labError, requisitionStatusLabel, requisitionStatusTone } from "../lib/lab";
import { fmtDay } from "../lib/exams";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabScope, type LabScopeValue } from "./lab/LabScope";
import { LabRequisitionDetail } from "./lab/LabRequisitionDetail";
import { RequisitionForm } from "./lab/RequisitionForm";
import { alert, fieldLabel, muted, okText, row, stack } from "./exams/styles";

export function LabRequisitions() {
  return (
    <LabGate title="Requisições e situação" sub="laboratório municipal · requisição, etiquetas, entrega e situação"
      allow={(r) => r.staff || r.professional}>
      {(roles) => <LabScope roles={roles}>{(scope) => <RequisitionsBody scope={scope} />}</LabScope>}
    </LabGate>
  );
}

// Requisição: dígitos e hífen ("2026-000123"); o resto é etiqueta.
const looksLikeNumber = (text: string) => /^\d{4}-\d{3,}$/.test(text);

function RequisitionsBody({ scope }: { scope: LabScopeValue }) {
  const [ search, setSearch ] = useState("");
  const [ filter, setFilter ] = useState<{ number?: string; barcode?: string }>({});
  const [ openId, setOpenId ] = useState<string | null>(null);
  const [ creating, setCreating ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const base = scope.kind === "lab" ? { providerId: scope.providerId } : { originUnitId: scope.unit.id };
  const query = useInfiniteQuery({
    queryKey: [ LAB_REQUISITIONS_KEY, base, filter ],
    queryFn: ({ pageParam }) => listLabRequisitions({ ...base, ...filter, before: pageParam }),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.next_before ?? undefined,
    gcTime: 0, staleTime: 0, retry: false
  });
  const rows = query.data?.pages.flatMap((p) => p.requisitions) ?? [];

  function find() {
    const text = search.trim();
    if (!text) { setFilter({}); return; }
    setFilter(looksLikeNumber(text) ? { number: text } : { barcode: text });
  }

  return (
    <div style={stack}>
      {notice && <p role="status" style={okText}>{notice}</p>}
      {creating ? (
        <RequisitionForm place={scope.kind} providerId={scope.providerId} originUnit={scope.kind === "ubs" ? scope.unit : null}
          onCancel={() => setCreating(false)}
          onDone={(r) => { setCreating(false); setNotice(`Requisição ${r.number} aberta.`); setOpenId(r.id); void query.refetch(); }} />
      ) : (
        <div><button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setCreating(true); }}>Nova requisição</button></div>
      )}
      <Panel title="Requisições" sub={scope.kind === "lab" ? scope.providerName : `${scope.unit.name} · ${scope.providerName}`}>
        <div style={stack}>
          <div style={{ ...row, alignItems: "flex-end" }}>
            <label style={fieldLabel}>
              Número da requisição ou etiqueta
              <input value={search} autoComplete="off" style={inputStyle} onChange={(e) => setSearch(e.target.value)}
                onKeyDown={(e) => { if (e.key === "Enter") { e.preventDefault(); find(); } }} />
            </label>
            <button type="button" style={secondaryButtonStyle} onClick={find}>Ver situação</button>
          </div>
          {query.isError && <p role="alert" style={alert}>{labError(query.error)}</p>}
          {query.isPending && <p className="mono" style={muted}>carregando…</p>}
          {query.isSuccess && (
            <DataTable<LabRequisitionRow>
              cols={[
                { label: "Número", w: "1fr", card: "title", render: (r) => <span className="mono">{r.number}</span> },
                { label: "Paciente", w: "1.6fr", render: (r) => r.patient.display_name },
                { label: "Origem", w: "1.2fr", render: (r) => r.origin_unit.name },
                { label: "Situação", w: "1fr", render: (r) => <Tag tone={requisitionStatusTone(r.status)}>{requisitionStatusLabel(r.status)}</Tag> },
                { label: "Previsão", w: "0.9fr", render: (r) => fmtDay(r.expected_delivery_on) },
                { label: "", w: "auto", card: "actions", render: (r) => (
                  <button type="button" style={secondaryButtonStyle} onClick={() => setOpenId(r.id)}>Abrir</button>
                ) }
              ]}
              rows={rows} rowKey={(r) => r.id} empty="nenhuma requisição"
            />
          )}
          {query.hasNextPage && (
            <div><button type="button" style={secondaryButtonStyle} onClick={() => void query.fetchNextPage()}>Carregar mais</button></div>
          )}
        </div>
      </Panel>
      {openId && <LabRequisitionDetail key={openId} id={openId} onClose={() => setOpenId(null)} />}
    </div>
  );
}
```

```tsx
// src/modules/lab/LabRequisitionDetail.tsx
// A requisição (contrato §2 `<lab_requisition>`): situação, prazo previsto,
// exames, amostras e laudos; comprovante, etiquetas (reimpressão confirmada:
// fica na trilha), laudo e entrega. 409 relê; nada no cache (gcTime 0).
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  deliverRequisition, errorCode, fetchLabLabels, fetchLabReceipt, getLabRequisition,
  type LabItem, type LabReportSummary, type LabSample
} from "../../lib/api";
import {
  LAB_REQUISITION_KEY, itemStatusLabel, labError, refreshLab, reportStatusLabel, requisitionStatusLabel, requisitionStatusTone,
  sampleStatusLabel, sampleStatusTone, signatureLabel
} from "../../lib/lab";
import { fmtDay } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { ActionButton } from "../../components/ActionButton";
import { ConfirmButton } from "../../components/ConfirmButton";
import { DataTable } from "../../components/DataTable";
import { KeyValue } from "../../components/KeyValue";
import { Tag } from "../../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { openExamPdf } from "../exams/openExamPdf";
import { openLabPdf } from "./openLabPdf";
import { alert, fieldLabel, grid, muted, panel, row, title } from "../exams/styles";

export function LabRequisitionDetail({ id, onClose }: { id: string; onClose(): void }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ LAB_REQUISITION_KEY, id ], queryFn: () => getLabRequisition(id), gcTime: 0, staleTime: 0, retry: false });
  const [ error, setError ] = useState<string | null>(null);
  const [ deliveredTo, setDeliveredTo ] = useState("");
  const r = query.data;

  async function pdf(fetcher: Parameters<typeof openLabPdf>[0]) {
    setError(null);
    const out = await openLabPdf(fetcher);
    if (!out.ok) setError(out.text);
  }

  async function report(rep: LabReportSummary) {
    if (!rep.pdf_url) return;
    setError(null);
    const out = await openExamPdf(rep.pdf_url);
    if (!out.ok) { setError(out.text); if (out.reread) void query.refetch(); }
  }

  async function deliver() {
    if (!deliveredTo.trim()) { setError("informe quem retirou o laudo"); return; }
    setError(null);
    try {
      await deliverRequisition(id, deliveredTo.trim());
      setDeliveredTo("");
      await refreshLab(queryClient);
    } catch (err) {
      setError(labError(err));
      if (errorCode(err) === "invalid_transition") void query.refetch();
    }
  }

  if (query.isPending) return <p className="mono" style={muted}>carregando a requisição…</p>;
  if (query.isError || !r) return <p role="alert" style={alert}>{labError(query.error)}</p>;

  return (
    <section aria-label={`Requisição ${r.number}`} style={panel}>
      <div style={row}>
        <h3 style={title}>{`Requisição ${r.number}`}</h3>
        <Tag tone={requisitionStatusTone(r.status)}>{requisitionStatusLabel(r.status)}</Tag>
        {r.urgent && <Tag tone="urgent">urgente</Tag>}
      </div>
      <div style={grid}>
        <KeyValue k="Paciente" v={r.patient.display_name} mono={false} />
        <KeyValue k="Nascimento" v={fmtDay(r.patient.birth_date)} />
        <KeyValue k="Origem" v={r.origin_unit.name} mono={false} />
        <KeyValue k="Previsão de entrega" v={fmtDay(r.expected_delivery_on)} />
      </div>
      <DataTable<LabItem>
        cols={[
          { label: "Exame", w: "2fr", card: "title", render: (i) => i.label },
          { label: "SIGTAP", w: "1fr", render: (i) => <span className="mono">{i.sigtap_code}</span> },
          { label: "Versão", w: "0.6fr", render: (i) => `v${i.version}` },
          { label: "Situação", w: "1fr", render: (i) => itemStatusLabel(i.status) }
        ]}
        rows={r.items} rowKey={(i) => i.id}
      />
      <DataTable<LabSample>
        cols={[
          { label: "Etiqueta", w: "1.2fr", card: "title", render: (s) => <span className="mono">{s.barcode}</span> },
          { label: "Material", w: "1.6fr", render: (s) => `${s.material} · ${s.container}` },
          { label: "Situação", w: "1.6fr", render: (s) => (
            <Tag tone={sampleStatusTone(s.status)}>
              {`${sampleStatusLabel(s)}${s.rejection_reason ? ` — ${s.rejection_reason}` : ""}${s.restriction_note ? ` — ${s.restriction_note}` : ""}`}
            </Tag>
          ) },
          { label: "", w: "auto", card: "actions", render: (s) => (
            <ConfirmButton label="Reimprimir" question="A reimpressão fica registrada. Reimprimir a etiqueta?"
              onConfirm={() => pdf(() => fetchLabLabels(r.id, [ s.id ]))} />
          ) }
        ]}
        rows={r.samples} rowKey={(s) => s.id}
      />
      {r.reports.length > 0 && (
        <ul aria-label="laudos" style={{ margin: 0, paddingLeft: 18, fontSize: 12.5 }}>
          {r.reports.map((rep) => (
            <li key={rep.id}>
              {`Laudo v${rep.version}${rep.partial ? " (parcial)" : ""} · ${reportStatusLabel(rep.status)} · ${signatureLabel(rep.signature_kind)}`}
              {rep.pdf_url && <> <button type="button" style={secondaryButtonStyle} onClick={() => void report(rep)}>Abrir laudo</button></>}
            </li>
          ))}
        </ul>
      )}
      <div style={row}>
        <ActionButton variant="secondary" onClick={() => pdf(() => fetchLabReceipt(r.id))}>Comprovante</ActionButton>
        <ActionButton variant="secondary" onClick={() => pdf(() => fetchLabLabels(r.id, r.samples.map((s) => s.id)))}>Imprimir etiquetas</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>
      </div>
      {r.status === "delivered" && r.delivered_to && (
        <p style={muted}>{`Entregue a ${r.delivered_to} · ${fmtDateTime(r.delivered_at)}`}</p>
      )}
      {r.status === "released" && (
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>
            Quem retirou
            <input value={deliveredTo} autoComplete="off" style={inputStyle} onChange={(e) => setDeliveredTo(e.target.value)} />
          </label>
          <ActionButton onClick={deliver}>Registrar entrega</ActionButton>
        </div>
      )}
      {error && <p role="alert" style={alert}>{error}</p>}
    </section>
  );
}
```

> Formato de data: `fmtDateTime` usa o `DATE_TIME` da cidade (dia/mês/ano e hora); o teste "entregue" casa por expressão regular só com o nome e a data.

Em `src/App.tsx`: `import { LabRequisitions } from "./modules/LabRequisitions";` e, em `renderModule`, depois de `case "exams-admin"`: `case "lab-requisitions": return <LabRequisitions />;`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/modules/LabRequisitions.test.tsx src/modules/lab/LabRequisitionDetail.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/lab/LabScope.tsx src/modules/LabRequisitions.tsx src/modules/LabRequisitions.test.tsx \
  src/modules/lab/LabRequisitionDetail.tsx src/modules/lab/LabRequisitionDetail.test.tsx src/App.tsx
$G commit -m "feat: add lab requisitions screen with status lookup, labels and delivery

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Coleta por leitura

**Files:**
- Create: `src/modules/LabCollection.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabCollection.test.tsx`

**Interfaces:**
- Consumes: `collectSampleByBarcode`, `errorCode` (Task 1); `labError`, `refreshLab` (Task 2); `LabGate`, `ScanField`, `ScanOutcome` (Task 4); `LabScope` (Task 6).
- Produces: `LabCollection()` em `renderModule` como `case "lab-collection"`; `collectOutcome(err: unknown): ScanOutcome` (exportada para o teste).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabCollection.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), collectSampleByBarcode: vi.fn(), listActiveUnits: vi.fn() };
});

import * as api from "../lib/api";
import { LabCollection } from "./LabCollection";
import { renderWithProviders } from "../test/campaignFixtures";
import { UBS21B, labContext, labSample, labUser } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const read = (code: string) => {
  const input = screen.getByLabelText("Etiqueta do tubo");
  fireEvent.change(input, { target: { value: code } });
  fireEvent.keyDown(input, { key: "Enter" });
};

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  localStorage.clear();
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.collectSampleByBarcode, api.listActiveUnits ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "health_professional" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.listActiveUnits).mockResolvedValue([ UBS21B ]);
  mocked(api.collectSampleByBarcode).mockResolvedValue({ ...labSample({ collected_at: "2026-10-15T09:00:00-03:00" }), requisition_number: "2026-000123" });
});

describe("LabCollection", () => {
  it("UBS: a leitura registra a coleta e diz a requisição", async () => {
    renderWithProviders(<LabCollection />);
    fireEvent.click(await screen.findByRole("button", { name: "UBS Centro" }));
    await screen.findByLabelText("Etiqueta do tubo");
    read("LB0001234");
    await waitFor(() => expect(api.collectSampleByBarcode).toHaveBeenCalledWith("LB0001234"));
    expect(await screen.findByText("LB0001234 · coleta registrada · requisição 2026-000123")).not.toBeNull();
  });

  it("etiqueta desconhecida e amostra já coletada têm frases próprias", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
    mocked(api.collectSampleByBarcode)
      .mockRejectedValueOnce(new api.ApiError(404, { error: "not_found" }, "x"))
      .mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    renderWithProviders(<LabCollection />);
    await screen.findByLabelText("Etiqueta do tubo");
    read("ZZ1"); read("LB0001234");
    expect(await screen.findByText("ZZ1 · etiqueta ou requisição não encontrada — confira o código")).not.toBeNull();
    expect(await screen.findByText("LB0001234 · esta amostra já foi coletada ou não está aguardando coleta")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabCollection.test.tsx`
Expected: FAIL — `./LabCollection` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/LabCollection.tsx
// Coleta (spec §5.2): ler a etiqueta registra quem coletou e quando; no 21a o
// exame vai a `collected`. A UBS que coleta e o posto do laboratório usam a
// mesma tela; o api confere a unidade e a situação da amostra.
import { useQueryClient } from "@tanstack/react-query";
import { collectSampleByBarcode, errorCode } from "../lib/api";
import { labError, refreshLab } from "../lib/lab";
import { Panel } from "../components/Panel";
import { LabGate } from "./lab/LabGate";
import { LabScope } from "./lab/LabScope";
import { ScanField, type ScanOutcome } from "./lab/ScanField";

export function collectOutcome(err: unknown): ScanOutcome {
  if (errorCode(err) === "invalid_transition") return { ok: false, text: "esta amostra já foi coletada ou não está aguardando coleta" };
  return { ok: false, text: labError(err) };
}

export function LabCollection() {
  const queryClient = useQueryClient();
  async function collect(code: string): Promise<ScanOutcome> {
    try {
      const sample = await collectSampleByBarcode(code);
      void refreshLab(queryClient);
      return { ok: true, text: `coleta registrada${sample.requisition_number ? ` · requisição ${sample.requisition_number}` : ""}` };
    } catch (err) {
      return collectOutcome(err);
    }
  }
  return (
    <LabGate title="Coleta" sub="laboratório municipal · registrar a coleta lendo a etiqueta" allow={(r) => r.staff || r.professional}>
      {(roles) => (
        <LabScope roles={roles}>
          {(scope) => (
            <Panel title="Ler etiquetas" sub={scope.kind === "ubs" ? scope.unit.name : scope.providerName}>
              <ScanField label="Etiqueta do tubo" onScan={collect}
                hint="Aponte o leitor para a etiqueta do tubo depois de coletar; cada leitura registra a coleta com o seu nome e a hora." />
            </Panel>
          )}
        </LabScope>
      )}
    </LabGate>
  );
}
```

Em `src/App.tsx`: `import { LabCollection } from "./modules/LabCollection";` e `case "lab-collection": return <LabCollection />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabCollection.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabCollection.tsx src/modules/LabCollection.test.tsx src/App.tsx
$G commit -m "feat: add barcode-driven sample collection screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Remessas — montar e enviar na UBS, recoletas e não recebidas; conferência no laboratório

**Files:**
- Create: `src/modules/LabShipments.tsx`, `src/modules/lab/ShipmentBuilder.tsx`, `src/modules/lab/ShipmentCheck.tsx`, `src/modules/lab/UnitPendingSamples.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabShipments.test.tsx`

**Interfaces:**
- Consumes: `createShipment`, `sendShipment`, `listShipments`, `receiveShipment`, `listUnitPending`, `fetchLabLabels`, `errorCode`, `LabShipment`, `LabPendingSample` (Task 1); `labError`, `refreshLab`, `parseTemperature`, `temperatureProblem`, `shipmentStatusLabel`, `LAB_SHIPMENTS_KEY`, `LAB_UNIT_PENDING_KEY` (Task 2); `LabGate`, `ScanField`, `openLabPdf` (Task 4); `LabScope`, `LabScopeValue` (Task 6).
- Produces: `LabShipments()` em `renderModule` como `case "lab-shipments"`; `ShipmentBuilder({ unitId, providerId, onSent })`; `ShipmentCheck({ providerId })`; `UnitPendingSamples({ unitId })`; `shipmentErrorText(err: unknown): string` (prefixa o código de barras quando o api o devolve — Divergência D4).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabShipments.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listActiveUnits: vi.fn(), createShipment: vi.fn(), sendShipment: vi.fn(),
    listShipments: vi.fn(), receiveShipment: vi.fn(), listUnitPending: vi.fn(), fetchLabLabels: vi.fn() };
});

import * as api from "../lib/api";
import { LabShipments } from "./LabShipments";
import { renderWithProviders } from "../test/campaignFixtures";
import { UBS21B, labContext, labSample, labShipment, labUser } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const read = (label: string, code: string) => {
  const input = screen.getByLabelText(label);
  fireEvent.change(input, { target: { value: code } });
  fireEvent.keyDown(input, { key: "Enter" });
};

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  localStorage.clear();
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.listActiveUnits, api.createShipment, api.sendShipment, api.listShipments,
    api.receiveShipment, api.listUnitPending, api.fetchLabLabels ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "health_professional" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.listActiveUnits).mockResolvedValue([ UBS21B ]);
  mocked(api.listShipments).mockResolvedValue([]);
  mocked(api.listUnitPending).mockResolvedValue({ recollections: [], not_received: [] });
  mocked(api.createShipment).mockResolvedValue(labShipment({ status: "open", sent_at: null }));
  mocked(api.sendShipment).mockResolvedValue(labShipment());
});

async function openUbs() {
  renderWithProviders(<LabShipments />);
  fireEvent.click(await screen.findByRole("button", { name: "UBS Centro" }));
  await screen.findByLabelText("Etiqueta do tubo para a remessa");
}

describe("LabShipments — UBS", () => {
  it("monta pela leitura (sem repetir), exige a temperatura e envia", async () => {
    await openUbs();
    read("Etiqueta do tubo para a remessa", "LB0001234");
    read("Etiqueta do tubo para a remessa", "LB0001235");
    read("Etiqueta do tubo para a remessa", "LB0001235x");
    expect(await screen.findByText("LB0001235x · incluída")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Tirar LB0001235x" }));
    fireEvent.click(screen.getByRole("button", { name: "Enviar remessa (2)" }));
    expect(screen.getByRole("alert").textContent).toBe("informe a temperatura da caixa em °C");
    expect(api.createShipment).not.toHaveBeenCalled();
    fireEvent.change(screen.getByLabelText("Temperatura da caixa (°C)"), { target: { value: "4,5" } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar remessa (2)" }));
    await waitFor(() => expect(api.createShipment).toHaveBeenCalledWith({
      origin_unit_id: "u1", provider_id: "pv9", barcodes: [ "LB0001234", "LB0001235" ], temperature_celsius: 4.5
    }));
    await waitFor(() => expect(api.sendShipment).toHaveBeenCalledWith("sh1"));
    expect(await screen.findByText("Remessa enviada com 2 amostras.")).not.toBeNull();
  });

  it("a mesma etiqueta lida de novo não entra duas vezes", async () => {
    await openUbs();
    read("Etiqueta do tubo para a remessa", "LB0001234");
    await screen.findByText("LB0001234 · incluída");
    // depois da janela de leitura dupla: o ScanField entrega, o montador recusa
    vi.spyOn(Date, "now").mockReturnValue(Date.now() + 5000);
    read("Etiqueta do tubo para a remessa", "LB0001234");
    expect(await screen.findByText("LB0001234 · já está nesta remessa")).not.toBeNull();
  });

  it("422 com o código: diz qual amostra; envio que falha deixa a remessa montada para reenviar", async () => {
    mocked(api.createShipment).mockRejectedValueOnce(new api.ApiError(422, { error: "sample_not_collected", barcode: "LB0001234" }, "x"));
    await openUbs();
    read("Etiqueta do tubo para a remessa", "LB0001234");
    fireEvent.change(await screen.findByLabelText("Temperatura da caixa (°C)"), { target: { value: "5" } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar remessa (1)" }));
    expect((await screen.findByRole("alert")).textContent).toBe("LB0001234: a amostra ainda não foi coletada");
    mocked(api.sendShipment).mockRejectedValueOnce(new Error("rede"));
    mocked(api.listShipments).mockResolvedValue([ labShipment({ status: "open", sent_at: null }) ]);
    fireEvent.click(screen.getByRole("button", { name: "Enviar remessa (1)" }));
    expect(await screen.findByText(/remessa montada, mas não enviada/)).not.toBeNull();
    const open = await screen.findByRole("region", { name: "Remessas montadas" });
    fireEvent.click(within(open).getByRole("button", { name: "Enviar" }));
    await waitFor(() => expect(api.sendShipment).toHaveBeenCalledTimes(2));
  });

  it("recoletas e amostras que não chegaram, com a etiqueta da recoleta", async () => {
    const pending = (over: object) => ({ ...labSample(over), requisition: { id: "rq1", number: "2026-000123", patient_display_name: "Joana Lima" } });
    mocked(api.listUnitPending).mockResolvedValue({
      recollections: [ pending({ id: "sa9", barcode: "LB0009999", recollection_of_id: "sa2" }) ],
      not_received: [ pending({ id: "sa3", barcode: "LB0001236", status: "not_received" }) ]
    });
    mocked(api.fetchLabLabels).mockResolvedValue({ blob: new Blob([ "%PDF" ]), filename: null });
    vi.spyOn(window, "open").mockReturnValue({ location: { href: "" }, close: vi.fn(), opener: null } as unknown as Window);
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") });
    await openUbs();
    const rec = await screen.findByRole("region", { name: "Recoletas pedidas pelo laboratório" });
    expect(within(rec).getByText("Joana Lima")).not.toBeNull();
    fireEvent.click(within(rec).getByRole("button", { name: "Etiqueta" }));
    await waitFor(() => expect(api.fetchLabLabels).toHaveBeenCalledWith("rq1", [ "sa9" ]));
    const lost = screen.getByRole("region", { name: "Amostras que não chegaram ao laboratório" });
    expect(within(lost).getByText("LB0001236")).not.toBeNull();
  });
});

describe("LabShipments — laboratório", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
    mocked(api.listShipments).mockResolvedValue([ labShipment() ]);
    mocked(api.receiveShipment).mockResolvedValue(labShipment({ status: "received" }));
  });

  it("confere pela leitura; o que não foi lido vira 'não chegou' depois da confirmação", async () => {
    renderWithProviders(<LabShipments />);
    fireEvent.click(await screen.findByRole("button", { name: "Conferir" }));
    const check = await screen.findByRole("region", { name: "Conferência da remessa de UBS Centro" });
    read("Etiqueta do tubo recebido", "LB0001234");
    read("Etiqueta do tubo recebido", "LB7777777");
    expect(await within(check).findByText("LB7777777 · não é desta remessa")).not.toBeNull();
    expect(within(check).getByText("1 de 2 lidas")).not.toBeNull();
    fireEvent.click(within(check).getByRole("button", { name: "Concluir conferência" }));
    expect(within(check).getByText("1 amostra não lida ficará como \"não chegou\" e a UBS será avisada. Concluir?")).not.toBeNull();
    fireEvent.click(within(check).getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.receiveShipment).toHaveBeenCalledWith("sh1", [ "LB0001234" ]));
    expect(await screen.findByText("Remessa conferida. Siga para o recebimento de cada amostra.")).not.toBeNull();
  });

  it("409: outra pessoa já conferiu — explica e relê a lista", async () => {
    mocked(api.receiveShipment).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    renderWithProviders(<LabShipments />);
    fireEvent.click(await screen.findByRole("button", { name: "Conferir" }));
    read("Etiqueta do tubo recebido", "LB0001234");
    read("Etiqueta do tubo recebido", "LB0001235");
    fireEvent.click(await screen.findByRole("button", { name: "Concluir conferência" }));
    await waitFor(() => expect(api.receiveShipment).toHaveBeenCalled());
    expect((await screen.findByRole("alert")).textContent).toBe("a situação mudou — a tela foi atualizada");
    await waitFor(() => expect(api.listShipments).toHaveBeenCalledTimes(2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabShipments.test.tsx`
Expected: FAIL — `./LabShipments` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/ShipmentBuilder.tsx
// Remessa da UBS (spec §5.3): a caixa é montada lendo as etiquetas (só na
// tela, sem chamar o api a cada leitura), com a temperatura, e enviada.
// Criar e enviar são duas chamadas: se o envio falhar, a remessa fica
// montada e aparece em "Remessas montadas" para reenviar.
import { useRef, useState } from "react";
import { createShipment, errorCode, sendShipment, type LabShipment } from "../../lib/api";
import { labError, parseTemperature, temperatureProblem } from "../../lib/lab";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ScanField, type ScanOutcome } from "./ScanField";
import { alert, fieldLabel, list, muted, okText, row, stack } from "../exams/styles";

export function shipmentErrorText(err: unknown): string {
  const body = (err as { body?: { barcode?: unknown } } | null)?.body;
  const code = errorCode(err) ? body?.barcode : undefined;
  return typeof code === "string" ? `${code}: ${labError(err)}` : labError(err);
}

export function ShipmentBuilder({ unitId, providerId, onSent }: { unitId: string; providerId: string; onSent(s: LabShipment | null): void }) {
  const [ barcodes, setBarcodesState ] = useState<string[]>([]);
  // Espelho síncrono da lista: o ScanField pode entregar duas leituras antes
  // de um re-render, e a decisão "incluída" × "já está" não pode ler estado velho.
  const listRef = useRef<string[]>([]);
  const setBarcodes = (next: string[]) => { listRef.current = next; setBarcodesState(next); };
  const [ temperature, setTemperature ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  async function add(code: string): Promise<ScanOutcome> {
    if (listRef.current.includes(code)) return { ok: false, text: "já está nesta remessa" };
    setBarcodes([ ...listRef.current, code ]);
    return { ok: true, text: "incluída" };
  }

  async function send() {
    const problem = barcodes.length === 0 ? "leia ao menos uma etiqueta" : temperatureProblem(temperature);
    if (problem) { setError(problem); return; }
    setError(null);
    setNotice(null);
    let created: LabShipment;
    try {
      created = await createShipment({ origin_unit_id: unitId, provider_id: providerId, barcodes, temperature_celsius: parseTemperature(temperature) as number });
    } catch (err) {
      setError(shipmentErrorText(err));
      return;
    }
    try {
      const sent = await sendShipment(created.id);
      setNotice(`Remessa enviada com ${barcodes.length} amostra${barcodes.length === 1 ? "" : "s"}.`);
      setBarcodes([]);
      setTemperature("");
      onSent(sent);
    } catch (err) {
      setError(`remessa montada, mas não enviada (${labError(err)}) — envie de novo em "Remessas montadas"`);
      setBarcodes([]);
      onSent(null);
    }
  }

  return (
    <div style={stack}>
      <ScanField label="Etiqueta do tubo para a remessa" onScan={add} hint="Leia cada tubo que vai na caixa." />
      {barcodes.length > 0 && (
        <ul aria-label="na remessa" style={list}>
          {barcodes.map((b) => (
            <li key={b} style={row}>
              <span className="mono">{b}</span>
              <button type="button" style={secondaryButtonStyle} aria-label={`Tirar ${b}`}
                onClick={() => setBarcodes(listRef.current.filter((x) => x !== b))}>Tirar</button>
            </li>
          ))}
        </ul>
      )}
      {barcodes.length === 0 && <p style={muted}>nenhum tubo lido ainda</p>}
      <label style={fieldLabel}>
        Temperatura da caixa (°C)
        <input value={temperature} inputMode="decimal" style={inputStyle} onChange={(e) => setTemperature(e.target.value)} />
      </label>
      <div><ActionButton onClick={send}>{`Enviar remessa (${barcodes.length})`}</ActionButton></div>
      {error && <p role="alert" style={alert}>{error}</p>}
      {notice && <p role="status" style={okText}>{notice}</p>}
    </div>
  );
}
```

> No teste "a mesma etiqueta…", o `vi.spyOn(Date, "now")` passa da janela de leitura dupla do `ScanField` (2 s): assim quem recusa é o montador, provado sozinho.

```tsx
// src/modules/lab/UnitPendingSamples.tsx
// O que o laboratório devolveu para a UBS (spec §5.3–§5.4): recoletas (nova
// amostra ligada à rejeitada; imprime a etiqueta nova) e amostras da remessa
// que não chegaram. Lido ao abrir; sem releitura automática.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { fetchLabLabels, listUnitPending, type LabPendingSample } from "../../lib/api";
import { LAB_UNIT_PENDING_KEY, labError } from "../../lib/lab";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { ActionButton } from "../../components/ActionButton";
import { openLabPdf } from "./openLabPdf";
import { alert, muted, stack } from "../exams/styles";

export function UnitPendingSamples({ unitId }: { unitId: string }) {
  const query = useQuery({ queryKey: [ LAB_UNIT_PENDING_KEY, unitId ], queryFn: () => listUnitPending(unitId), gcTime: 0, staleTime: 0, retry: false });
  const [ error, setError ] = useState<string | null>(null);
  if (query.isPending) return <p className="mono" style={muted}>carregando…</p>;
  if (query.isError) return <p role="alert" style={alert}>{labError(query.error)}</p>;
  const label = async (s: LabPendingSample) => {
    setError(null);
    const out = await openLabPdf(() => fetchLabLabels(s.requisition.id, [ s.id ]));
    if (!out.ok) setError(out.text);
  };
  const cols = (withLabel: boolean) => [
    { label: "Etiqueta", w: "1.1fr", card: "title" as const, render: (s: LabPendingSample) => <span className="mono">{s.barcode}</span> },
    { label: "Requisição", w: "1fr", render: (s: LabPendingSample) => <span className="mono">{s.requisition.number}</span> },
    { label: "Paciente", w: "1.6fr", render: (s: LabPendingSample) => s.requisition.patient_display_name },
    { label: "Material", w: "1.2fr", render: (s: LabPendingSample) => s.material },
    ...(withLabel
      ? [ { label: "", w: "auto", card: "actions" as const, render: (s: LabPendingSample) => <ActionButton variant="secondary" onClick={() => label(s)}>Etiqueta</ActionButton> } ]
      : [ { label: "Coletada", w: "1fr", render: (s: LabPendingSample) => fmtDateTime(s.collected_at) } ])
  ];
  return (
    <div style={stack}>
      <Panel title="Recoletas pedidas pelo laboratório" sub="amostra rejeitada · colete de novo e envie">
        <DataTable<LabPendingSample> cols={cols(true)} rows={query.data.recollections} rowKey={(s) => s.id} empty="nenhuma recoleta" />
      </Panel>
      <Panel title="Amostras que não chegaram ao laboratório" sub="estavam na remessa e não foram lidas na conferência">
        <DataTable<LabPendingSample> cols={cols(false)} rows={query.data.not_received} rowKey={(s) => s.id} empty="nenhuma" />
      </Panel>
      {error && <p role="alert" style={alert}>{error}</p>}
    </div>
  );
}
```

```tsx
// src/modules/lab/ShipmentCheck.tsx
// Conferência da remessa no laboratório (spec §5.3): lê as etiquetas da
// caixa; o que estava na lista e não foi lido vira `not_received` e avisa a
// UBS. Confirmação antes, com a contagem do que falta. Aceitar ou rejeitar
// cada amostra é do Recebimento.
import { useRef, useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, listShipments, receiveShipment, type LabShipment } from "../../lib/api";
import { LAB_SHIPMENTS_KEY, labError, refreshLab } from "../../lib/lab";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { ConfirmButton } from "../../components/ConfirmButton";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { ScanField, type ScanOutcome } from "./ScanField";
import { alert, muted, okText, panel, row, stack } from "../exams/styles";

export function ShipmentCheck({ providerId }: { providerId: string }) {
  const queryClient = useQueryClient();
  const shipments = useQuery({
    queryKey: [ LAB_SHIPMENTS_KEY, "lab", providerId ], queryFn: () => listShipments({ providerId, statuses: [ "sent" ] }),
    gcTime: 0, staleTime: 0, retry: false
  });
  const [ open, setOpen ] = useState<LabShipment | null>(null);
  const [ read, setReadState ] = useState<string[]>([]);
  const readRef = useRef<string[]>([]);
  const setRead = (next: string[]) => { readRef.current = next; setReadState(next); };
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  function start(s: LabShipment) { setOpen(s); setRead([]); setError(null); setNotice(null); }

  async function scan(code: string): Promise<ScanOutcome> {
    if (!open) return { ok: false, text: "escolha a remessa" };
    if (!open.samples.some((s) => s.barcode === code)) return { ok: false, text: "não é desta remessa" };
    if (readRef.current.includes(code)) return { ok: false, text: "já conferida" };
    setRead([ ...readRef.current, code ]);
    return { ok: true, text: "conferida" };
  }

  async function finish() {
    if (!open) return;
    setError(null);
    try {
      await receiveShipment(open.id, readRef.current);
      setOpen(null);
      setNotice("Remessa conferida. Siga para o recebimento de cada amostra.");
      await refreshLab(queryClient);
    } catch (err) {
      setError(labError(err));
      if (errorCode(err) === "invalid_transition") { setOpen(null); void shipments.refetch(); }
    }
  }

  const missing = open ? open.samples.length - read.length : 0;
  return (
    <div style={stack}>
      {notice && <p role="status" style={okText}>{notice}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}
      <Panel title="Remessas a caminho" sub="enviadas pelas UBS e ainda não conferidas">
        {shipments.isError ? <p role="alert" style={alert}>{labError(shipments.error)}</p> : (
          <DataTable<LabShipment>
            cols={[
              { label: "UBS", w: "1.6fr", card: "title", render: (s) => s.origin_unit.name },
              { label: "Enviada", w: "1.2fr", render: (s) => fmtDateTime(s.sent_at) },
              { label: "Amostras", w: "0.8fr", align: "right", render: (s) => String(s.samples.length) },
              { label: "Temperatura", w: "0.9fr", render: (s) => (s.temperature_celsius === null ? "—" : `${String(s.temperature_celsius).replace(".", ",")} °C`) },
              { label: "", w: "auto", card: "actions", render: (s) => <button type="button" style={secondaryButtonStyle} onClick={() => start(s)}>Conferir</button> }
            ]}
            rows={shipments.data ?? []} rowKey={(s) => s.id} empty={shipments.isPending ? "carregando…" : "nenhuma remessa a caminho"}
          />
        )}
      </Panel>
      {open && (
        <section aria-label={`Conferência da remessa de ${open.origin_unit.name}`} style={panel}>
          <ScanField label="Etiqueta do tubo recebido" onScan={scan} />
          <p style={muted}>{`${read.length} de ${open.samples.length} lidas`}</p>
          <ul aria-label="amostras da remessa" style={{ margin: 0, paddingLeft: 18, fontSize: 12.5 }}>
            {open.samples.map((s) => (
              <li key={s.id}>
                <span className="mono">{s.barcode}</span>{` · ${s.material} · ${s.requisition_number} `}
                {read.includes(s.barcode) ? <Tag tone="ok">lida</Tag> : <Tag tone="warn">falta</Tag>}
              </li>
            ))}
          </ul>
          <div style={row}>
            {missing > 0 ? (
              <ConfirmButton label="Concluir conferência"
                question={`${missing} amostra${missing === 1 ? "" : "s"} não lida${missing === 1 ? "" : "s"} ficará${missing === 1 ? "" : "ão"} como "não chegou" e a UBS será avisada. Concluir?`}
                onConfirm={finish} />
            ) : (
              <button type="button" style={secondaryButtonStyle} onClick={() => void finish()}>Concluir conferência</button>
            )}
            <button type="button" style={secondaryButtonStyle} onClick={() => setOpen(null)}>Fechar</button>
          </div>
        </section>
      )}
    </div>
  );
}
```

```tsx
// src/modules/LabShipments.tsx
// Remessas (spec §5.3; contrato §4): na UBS que coleta, montar e enviar,
// remessas montadas e não enviadas, recoletas e não recebidas; no
// laboratório, a conferência das remessas a caminho.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, listShipments, sendShipment, type LabShipment } from "../lib/api";
import { LAB_SHIPMENTS_KEY, labError, refreshLab, shipmentStatusLabel } from "../lib/lab";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { ActionButton } from "../components/ActionButton";
import { LabGate } from "./lab/LabGate";
import { LabScope } from "./lab/LabScope";
import { ShipmentBuilder } from "./lab/ShipmentBuilder";
import { ShipmentCheck } from "./lab/ShipmentCheck";
import { UnitPendingSamples } from "./lab/UnitPendingSamples";
import { alert, stack } from "./exams/styles";

export function LabShipments() {
  return (
    <LabGate title="Remessas" sub="laboratório municipal · da UBS ao laboratório" allow={(r) => r.staff || r.professional}>
      {(roles) => (
        <LabScope roles={roles}>
          {(scope) => scope.kind === "lab"
            ? <ShipmentCheck providerId={scope.providerId} />
            : <UbsShipments unitId={scope.unit.id} providerId={scope.providerId} unitName={scope.unit.name} />}
        </LabScope>
      )}
    </LabGate>
  );
}

function UbsShipments({ unitId, providerId, unitName }: { unitId: string; providerId: string; unitName: string }) {
  const queryClient = useQueryClient();
  const [ error, setError ] = useState<string | null>(null);
  const openShipments = useQuery({
    queryKey: [ LAB_SHIPMENTS_KEY, "ubs", unitId ], queryFn: () => listShipments({ originUnitId: unitId, statuses: [ "open" ] }),
    gcTime: 0, staleTime: 0, retry: false
  });

  async function resend(s: LabShipment) {
    setError(null);
    try {
      await sendShipment(s.id);
      await refreshLab(queryClient);
    } catch (err) {
      setError(labError(err));
      if (errorCode(err) === "invalid_transition") void openShipments.refetch();
    }
  }

  return (
    <div style={stack}>
      <Panel title="Nova remessa" sub={unitName}>
        <ShipmentBuilder unitId={unitId} providerId={providerId} onSent={() => void openShipments.refetch()} />
      </Panel>
      {(openShipments.data?.length ?? 0) > 0 && (
        <Panel title="Remessas montadas" sub="criadas e ainda não enviadas">
          <DataTable<LabShipment>
            cols={[
              { label: "Amostras", w: "1fr", card: "title", render: (s) => s.samples.map((x) => x.barcode).join(", ") },
              { label: "Situação", w: "0.8fr", render: (s) => shipmentStatusLabel(s.status) },
              { label: "", w: "auto", card: "actions", render: (s) => <ActionButton variant="secondary" onClick={() => resend(s)}>Enviar</ActionButton> }
            ]}
            rows={openShipments.data ?? []} rowKey={(s) => s.id}
          />
        </Panel>
      )}
      {error && <p role="alert" style={alert}>{error}</p>}
      <UnitPendingSamples unitId={unitId} />
    </div>
  );
}
```

> `Panel` rotula a região com o `title` (`aria-label`), por isso o teste acha "Remessas montadas", "Recoletas pedidas pelo laboratório" e "Amostras que não chegaram ao laboratório" como `region`.

Em `src/App.tsx`: `import { LabShipments } from "./modules/LabShipments";` e `case "lab-shipments": return <LabShipments />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabShipments.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabShipments.tsx src/modules/LabShipments.test.tsx src/modules/lab/ShipmentBuilder.tsx src/modules/lab/ShipmentCheck.tsx \
  src/modules/lab/UnitPendingSamples.tsx src/App.tsx
$G commit -m "feat: add UBS shipments, recollections and lab shipment check

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Recebimento — aceitar, rejeitar (com recoleta) ou aceitar com restrição

**Files:**
- Create: `src/modules/LabReception.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabReception.test.tsx`

**Interfaces:**
- Consumes: `receiveSample`, `listActiveRejectionReasons`, `errorCode`, `LabReceiveDecision` (Task 1); `labError`, `refreshLab`, `LAB_REASONS_KEY` (Task 2); `LabGate`, `ScanField`, `ScanOutcome` (Task 4); `LabScope` (Task 6).
- Produces: `LabReception()` em `renderModule` como `case "lab-reception"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabReception.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), receiveSample: vi.fn(), listActiveRejectionReasons: vi.fn() };
});

import * as api from "../lib/api";
import { LabReception } from "./LabReception";
import { renderWithProviders } from "../test/campaignFixtures";
import { labContext, labSample, labUser } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const read = (code: string) => {
  const input = screen.getByLabelText("Etiqueta do tubo");
  fireEvent.change(input, { target: { value: code } });
  fireEvent.keyDown(input, { key: "Enter" });
};

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.receiveSample, api.listActiveRejectionReasons ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.listActiveRejectionReasons).mockResolvedValue([ { id: "rr1", provider_id: "pv9", name: "hemolisada", active: true } ]);
  mocked(api.receiveSample).mockResolvedValue({ sample: { ...labSample({ status: "received" }), requisition_number: "2026-000123" }, recollection: null });
});

describe("LabReception", () => {
  it("aceitar é o padrão: a leitura aceita", async () => {
    renderWithProviders(<LabReception />);
    await screen.findByLabelText("Etiqueta do tubo");
    read("LB0001234");
    await waitFor(() => expect(api.receiveSample).toHaveBeenCalledWith({ barcode: "LB0001234", decision: "accept" }));
    expect(await screen.findByText("LB0001234 · aceita · requisição 2026-000123")).not.toBeNull();
  });

  it("rejeitar sem motivo não chama o api; com motivo, avisa a recoleta", async () => {
    mocked(api.receiveSample).mockResolvedValue({
      sample: { ...labSample({ status: "rejected" }), requisition_number: "2026-000123" }, recollection: labSample({ id: "sa9", barcode: "LB0009999" })
    });
    renderWithProviders(<LabReception />);
    fireEvent.click(await screen.findByLabelText("Rejeitar"));
    read("LB0001234");
    expect(await screen.findByText("LB0001234 · escolha o motivo da rejeição antes de ler")).not.toBeNull();
    expect(api.receiveSample).not.toHaveBeenCalled();
    fireEvent.change(screen.getByLabelText("Motivo da rejeição"), { target: { value: "rr1" } });
    vi.spyOn(Date, "now").mockReturnValue(Date.now() + 5000);
    read("LB0001234");
    await waitFor(() => expect(api.receiveSample).toHaveBeenCalledWith({ barcode: "LB0001234", decision: "reject", reason_id: "rr1" }));
    expect(await screen.findByText("LB0001234 · rejeitada · recoleta LB0009999 pedida à UBS de origem")).not.toBeNull();
  });

  it("aceitar com restrição leva a nota que vai ao laudo", async () => {
    renderWithProviders(<LabReception />);
    fireEvent.click(await screen.findByLabelText("Aceitar com restrição"));
    fireEvent.change(screen.getByLabelText("Restrição (vai ao laudo)"), { target: { value: "lipêmica" } });
    read("LB0001234");
    await waitFor(() => expect(api.receiveSample).toHaveBeenCalledWith({
      barcode: "LB0001234", decision: "accept_with_restriction", restriction_note: "lipêmica"
    }));
  });

  it("409: amostra já recebida ou fora de trânsito tem frase própria", async () => {
    mocked(api.receiveSample).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    renderWithProviders(<LabReception />);
    await screen.findByLabelText("Etiqueta do tubo");
    read("LB0001234");
    expect(await screen.findByText("LB0001234 · esta amostra já foi recebida, rejeitada ou ainda não foi coletada")).not.toBeNull();
  });

  it("a UBS não recebe", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "health_professional" ]));
    renderWithProviders(<LabReception />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabReception.test.tsx`
Expected: FAIL — `./LabReception` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/LabReception.tsx
// Recebimento (spec §5.4; contrato §4): escolha a decisão e leia a etiqueta.
// Rejeição exige o motivo do cadastro e gera a recoleta na UBS de origem; a
// restrição vai ao laudo. A decisão vale para as leituras seguintes até ser
// trocada (cada leitura leva a decisão da hora em que foi lida).
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, listActiveRejectionReasons, receiveSample, type LabReceiveDecision } from "../lib/api";
import { LAB_REASONS_KEY, labError, refreshLab } from "../lib/lab";
import { Panel } from "../components/Panel";
import { inputStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabScope } from "./lab/LabScope";
import { ScanField, type ScanOutcome } from "./lab/ScanField";
import { alert, checkLabel, fieldLabel, fieldset, legend, stack } from "./exams/styles";

const DECISIONS: { value: LabReceiveDecision; label: string }[] = [
  { value: "accept", label: "Aceitar" }, { value: "reject", label: "Rejeitar" }, { value: "accept_with_restriction", label: "Aceitar com restrição" }
];

export function LabReception() {
  return (
    <LabGate title="Recebimento de amostras" sub="laboratório municipal · aceitar, rejeitar ou aceitar com restrição" allow={(r) => r.staff}>
      {(roles) => <LabScope roles={roles}>{(scope) => <ReceptionBody providerId={scope.providerId} />}</LabScope>}
    </LabGate>
  );
}

function ReceptionBody({ providerId }: { providerId: string }) {
  const queryClient = useQueryClient();
  const reasons = useQuery({ queryKey: [ LAB_REASONS_KEY, providerId ], queryFn: () => listActiveRejectionReasons(providerId), retry: false });
  const [ decision, setDecision ] = useState<LabReceiveDecision>("accept");
  const [ reasonId, setReasonId ] = useState("");
  const [ note, setNote ] = useState("");

  const receive = (d: LabReceiveDecision, reason: string, restriction: string) => async (code: string): Promise<ScanOutcome> => {
    if (d === "reject" && !reason) return { ok: false, text: "escolha o motivo da rejeição antes de ler" };
    if (d === "accept_with_restriction" && !restriction.trim()) return { ok: false, text: "escreva a restrição antes de ler" };
    try {
      const out = await receiveSample({
        barcode: code, decision: d,
        ...(d === "reject" ? { reason_id: reason } : {}),
        ...(d === "accept_with_restriction" ? { restriction_note: restriction.trim() } : {})
      });
      void refreshLab(queryClient);
      if (d === "reject") {
        return { ok: true, text: `rejeitada${out.recollection ? ` · recoleta ${out.recollection.barcode} pedida à UBS de origem` : ""}` };
      }
      return { ok: true, text: `${d === "accept" ? "aceita" : "aceita com restrição"} · requisição ${out.sample.requisition_number}` };
    } catch (err) {
      if (errorCode(err) === "invalid_transition") return { ok: false, text: "esta amostra já foi recebida, rejeitada ou ainda não foi coletada" };
      return { ok: false, text: labError(err) };
    }
  };

  return (
    <Panel title="Receber" sub="a decisão escolhida vale para as próximas leituras">
      <div style={stack}>
        <fieldset style={fieldset}>
          <legend style={legend}>Decisão</legend>
          {DECISIONS.map((d) => (
            <label key={d.value} style={checkLabel}>
              <input type="radio" name="decision" checked={decision === d.value} onChange={() => setDecision(d.value)} />{d.label}
            </label>
          ))}
        </fieldset>
        {decision === "reject" && (
          <label style={fieldLabel}>
            Motivo da rejeição
            <select value={reasonId} style={inputStyle} onChange={(e) => setReasonId(e.target.value)}>
              <option value="">—</option>
              {(reasons.data ?? []).map((r) => <option key={r.id} value={r.id}>{r.name}</option>)}
            </select>
          </label>
        )}
        {reasons.isError && <p role="alert" style={alert}>{labError(reasons.error)}</p>}
        {decision === "accept_with_restriction" && (
          <label style={fieldLabel}>
            Restrição (vai ao laudo)
            <input value={note} style={inputStyle} onChange={(e) => setNote(e.target.value)} />
          </label>
        )}
        <ScanField label="Etiqueta do tubo" onScan={receive(decision, reasonId, note)} />
      </div>
    </Panel>
  );
}
```

Em `src/App.tsx`: `import { LabReception } from "./modules/LabReception";` e `case "lab-reception": return <LabReception />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabReception.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabReception.tsx src/modules/LabReception.test.tsx src/App.tsx
$G commit -m "feat: add lab sample reception with rejection and restriction

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Digitação — formulário de resultado e digitação pela leitura

**Files:**
- Create: `src/modules/lab/ResultEntryForm.tsx`, `src/modules/LabEntry.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/lab/ResultEntryForm.test.tsx`, `src/modules/LabEntry.test.tsx`

**Interfaces:**
- Consumes: `enterLabResults`, `getLabEntry`, `errorCode`, `LabEntryItem`, `LabAnalyteEntry`, `LabResultRow` (Task 1); `entryProblem`, `normalizeNumber`, `previewFlag`, `flagLabel`, `flagTone`, `labError`, `refreshLab`, `itemStatusLabel` (Task 2); `LabGate`, `ScanField` (Task 4); `LabScope` (Task 6); `fmtDay` (21a).
- Produces: `ResultEntryForm({ item, onSaved, onClose })` — `onSaved(): void` (quem chama relê o item); `referenceText(a: LabAnalyteEntry): string` (exportada); `LabEntry()` em `renderModule` como `case "lab-entry"`. A Task 11 usa `ResultEntryForm` no mapa.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/lab/ResultEntryForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), enterLabResults: vi.fn() };
});

import * as api from "../../lib/api";
import { ResultEntryForm, referenceText } from "./ResultEntryForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { glucoseEntry, labResultRow, labUser, lipidEntry } from "../../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(labUser([ "lab_staff" ]));
  mocked(api.enterLabResults).mockReset().mockResolvedValue([ labResultRow() ]);
});

describe("ResultEntryForm", () => {
  it("referência da faixa aplicável, prévia da marca e vírgula convertida no envio", async () => {
    const onSaved = vi.fn();
    renderWithProviders(<ResultEntryForm item={glucoseEntry()} onSaved={onSaved} onClose={vi.fn()} />);
    const field = await screen.findByLabelText("Glicose (mg/dL)");
    expect(screen.getByText("referência 70 a 99 mg/dL · limite de digitação 10 a 1500")).not.toBeNull();
    fireEvent.change(field, { target: { value: "120,5" } });
    expect(screen.getByText("alto")).not.toBeNull();
    fireEvent.change(field, { target: { value: "450" } });
    expect(screen.getByText("crítico alto")).not.toBeNull();
    fireEvent.change(field, { target: { value: "98,0" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar resultado" }));
    await waitFor(() => expect(api.enterLabResults).toHaveBeenCalledWith({ item_id: "it2", values: [ { analyte_key: "glicose", value: "98.0" } ] }));
    expect(onSaved).toHaveBeenCalled();
    expect(await screen.findByText("Resultado salvo.")).not.toBeNull();
    expect((screen.getByLabelText("Glicose (mg/dL)") as HTMLInputElement).value).toBe("");
  });

  it("bloqueia fora do limite sem chamar o api", async () => {
    renderWithProviders(<ResultEntryForm item={glucoseEntry()} onSaved={vi.fn()} onClose={vi.fn()} />);
    fireEvent.change(await screen.findByLabelText("Glicose (mg/dL)"), { target: { value: "1600" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar resultado" }));
    expect(screen.getByRole("alert").textContent).toBe("corrija os valores marcados");
    expect(screen.getByText("fora do limite de digitação (10 a 1500 mg/dL) — confira o valor")).not.toBeNull();
    expect(api.enterLabResults).not.toHaveBeenCalled();
  });

  it("422 out_of_entry_range: marca o componente e mantém os valores", async () => {
    mocked(api.enterLabResults).mockRejectedValueOnce(new api.ApiError(422, { error: "out_of_entry_range", analyte_key: "glicose" }, "x"));
    renderWithProviders(<ResultEntryForm item={glucoseEntry()} onSaved={vi.fn()} onClose={vi.fn()} />);
    fireEvent.change(await screen.findByLabelText("Glicose (mg/dL)"), { target: { value: "900" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar resultado" }));
    expect((await screen.findByRole("alert")).textContent).toBe("valor fora do limite de digitação — confira");
    expect(screen.getByText("fora do limite de digitação da versão atual — confira")).not.toBeNull();
    expect((screen.getByLabelText("Glicose (mg/dL)") as HTMLInputElement).value).toBe("900");
  });

  it("calculado não se digita; nada digitado não envia", async () => {
    renderWithProviders(<ResultEntryForm item={lipidEntry()} onSaved={vi.fn()} onClose={vi.fn()} />);
    expect(await screen.findByText("Não-HDL (calculado) · calculado ao salvar")).not.toBeNull();
    expect(screen.queryByLabelText("Não-HDL (calculado) (mg/dL)")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Salvar resultado" }));
    expect(screen.getByRole("alert").textContent).toBe("digite ao menos um valor");
  });

  it("repetições: histórico com a origem (aparelho ou digitado) e qual vale", async () => {
    const item = glucoseEntry({ results: [
      labResultRow({ id: "r1", value: "410", flag: "critical_high", source: "hl7", chosen: false, entered_by: null }),
      labResultRow({ id: "r2", value: "405", flag: "critical_high", source: "typed", chosen: true })
    ] });
    renderWithProviders(<ResultEntryForm item={item} onSaved={vi.fn()} onClose={vi.fn()} />);
    const hist = await screen.findByRole("list", { name: "valores de Glicose" });
    const lines = within(hist).getAllByRole("listitem").map((li) => li.textContent);
    expect(lines[0]).toContain("410 mg/dL");
    expect(lines[0]).toContain("aparelho");
    expect(lines[1]).toContain("vale");
  });

  it("item com laudo liberado avisa que o valor novo só vale numa retificação", async () => {
    renderWithProviders(<ResultEntryForm item={glucoseEntry({ status: "released" })} onSaved={vi.fn()} onClose={vi.fn()} />);
    expect(await screen.findByText("laudo já liberado — o valor novo só entra numa retificação")).not.toBeNull();
  });

  it("referência em texto quando a faixa não tem números", () => {
    expect(referenceText({ ...glucoseEntry().analytes[0], reference: { low: null, high: null, text: "negativo" }, entry_min: null, entry_max: null })).toBe("referência negativo");
  });
});
```

```tsx
// src/modules/LabEntry.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), getLabEntry: vi.fn(), enterLabResults: vi.fn() };
});

import * as api from "../lib/api";
import { LabEntry } from "./LabEntry";
import { renderWithProviders } from "../test/campaignFixtures";
import { glucoseEntry, labContext, labUser, lipidEntry } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.getLabEntry, api.enterLabResults ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.getLabEntry).mockResolvedValue([ glucoseEntry(), lipidEntry() ]);
});

describe("LabEntry", () => {
  it("a leitura abre os exames da amostra; escolher um abre o formulário", async () => {
    renderWithProviders(<LabEntry />);
    const input = await screen.findByLabelText("Etiqueta do tubo");
    fireEvent.change(input, { target: { value: "LB0001235" } });
    fireEvent.keyDown(input, { key: "Enter" });
    await waitFor(() => expect(api.getLabEntry).toHaveBeenCalledWith({ barcode: "LB0001235" }));
    expect(await screen.findByText("LB0001235 · 2 exames nesta amostra")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Colesterol total e frações" }));
    expect(await screen.findByLabelText("Colesterol total (mg/dL)")).not.toBeNull();
  });

  it("sem exame para digitar: diz e não abre nada", async () => {
    mocked(api.getLabEntry).mockResolvedValue([]);
    renderWithProviders(<LabEntry />);
    const input = await screen.findByLabelText("Etiqueta do tubo");
    fireEvent.change(input, { target: { value: "LB0001235" } });
    fireEvent.keyDown(input, { key: "Enter" });
    expect(await screen.findByText("LB0001235 · nenhum exame para digitar nesta amostra")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/modules/lab/ResultEntryForm.test.tsx src/modules/LabEntry.test.tsx`
Expected: FAIL — módulos inexistentes.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/ResultEntryForm.tsx
// Digitação do resultado (spec §5.6; contrato §5). A faixa de referência já
// vem escolhida pelo api (sexo e idade na data da coleta); a tela só mostra a
// prévia da marca, bloqueia fora de entry_min/entry_max e converte a vírgula.
// Calculado (formula) não se digita: o api preenche ao salvar. Toda gravação
// cria linha nova (repetição); quem escolhe a que vale é o liberador.
import { useId, useState } from "react";
import { enterLabResults, errorCode, type LabAnalyteEntry, type LabEntryItem, type LabResultRow } from "../../lib/api";
import { entryProblem, flagLabel, flagTone, labError, normalizeNumber, previewFlag } from "../../lib/lab";
import { fmtDay } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { ActionButton } from "../../components/ActionButton";
import { Tag } from "../../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, list, muted, okText, row, title } from "../exams/styles";

const n = (v: number) => String(v).replace(".", ",");

export function referenceText(a: LabAnalyteEntry): string {
  const parts: string[] = [];
  const ref = a.reference;
  if (ref && (ref.low !== null || ref.high !== null)) {
    const unit = a.unit ? ` ${a.unit}` : "";
    if (ref.low !== null && ref.high !== null) parts.push(`referência ${n(ref.low)} a ${n(ref.high)}${unit}`);
    else if (ref.low !== null) parts.push(`referência ≥ ${n(ref.low)}${unit}`);
    else parts.push(`referência ≤ ${n(ref.high as number)}${unit}`);
  } else if (ref?.text) {
    parts.push(`referência ${ref.text}`);
  }
  if (a.entry_min !== null || a.entry_max !== null) {
    parts.push(`limite de digitação ${a.entry_min === null ? "—" : n(a.entry_min)} a ${a.entry_max === null ? "—" : n(a.entry_max)}`);
  }
  return parts.join(" · ");
}

function rowText(r: LabResultRow): string {
  const who = r.source === "hl7" ? "aparelho" : `digitado${r.entered_by ? ` por ${r.entered_by}` : ""}`;
  return `${r.value}${r.unit ? ` ${r.unit}` : ""} · ${who} · ${fmtDateTime(r.entered_at)}`;
}

export function ResultEntryForm({ item, onSaved, onClose }: { item: LabEntryItem; onSaved(): void; onClose(): void }) {
  const [ values, setValues ] = useState<Record<string, string>>({});
  const [ fieldErrors, setFieldErrors ] = useState<Record<string, string>>({});
  const [ error, setError ] = useState<string | null>(null);
  const [ saved, setSaved ] = useState(false);
  const baseId = useId();
  const typed = item.analytes.filter((a) => !a.formula);

  async function save() {
    setSaved(false);
    const errors: Record<string, string> = {};
    const out: { analyte_key: string; value: string }[] = [];
    for (const a of typed) {
      const raw = values[a.key] ?? "";
      if (raw.trim() === "") continue;
      const problem = entryProblem(a, raw);
      if (problem) { errors[a.key] = problem; continue; }
      out.push({ analyte_key: a.key, value: a.value_type === "numeric" ? (normalizeNumber(raw) as string) : raw.trim() });
    }
    setFieldErrors(errors);
    if (Object.keys(errors).length > 0) { setError("corrija os valores marcados"); return; }
    if (out.length === 0) { setError("digite ao menos um valor"); return; }
    setError(null);
    try {
      await enterLabResults({ item_id: item.item_id, values: out });
      setValues({});
      setSaved(true);
      onSaved();
    } catch (err) {
      setError(labError(err));
      const body = (err as { body?: { analyte_key?: unknown } }).body;
      if (errorCode(err) === "out_of_entry_range" && typeof body?.analyte_key === "string") {
        setFieldErrors({ [body.analyte_key]: "fora do limite de digitação da versão atual — confira" });
      }
      if (errorCode(err) === "invalid_transition" || errorCode(err) === "unknown_analyte") onSaved();
    }
  }

  return (
    <section aria-label={`Digitação de ${item.label}`} style={{ ...box, maxWidth: 720 }}>
      <div style={row}>
        <h3 style={title}>{item.label}</h3>
        <Tag>{`v${item.version}`}</Tag>
        <span className="mono" style={muted}>{`${item.requisition_number} · ${item.barcode}`}</span>
      </div>
      <p style={muted}>{`${item.patient.display_name} · ${item.patient.biological_sex === "female" ? "feminino" : item.patient.biological_sex === "male" ? "masculino" : item.patient.biological_sex} · nascimento ${fmtDay(item.patient.birth_date)}`}</p>
      {item.restriction_note && <p style={alert}>{`amostra aceita com restrição: ${item.restriction_note}`}</p>}
      {item.status === "released" && <p style={alert}>laudo já liberado — o valor novo só entra numa retificação</p>}
      <div style={form}>
        {item.analytes.map((a) => {
          const id = `${baseId}-${a.key}`;
          const history = item.results.filter((r) => r.analyte_key === a.key);
          const preview = previewFlag(a, values[a.key] ?? "");
          return (
            <div key={a.key} style={{ display: "flex", flexDirection: "column", gap: 4 }}>
              {a.formula ? (
                <p style={muted}>{`${a.name} · calculado ao salvar`}</p>
              ) : (
                <label htmlFor={id} style={fieldLabel}>{`${a.name}${a.unit ? ` (${a.unit})` : ""}`}</label>
              )}
              {!a.formula && a.value_type === "choice" && (
                <select id={id} value={values[a.key] ?? ""} style={inputStyle}
                  onChange={(e) => setValues((v) => ({ ...v, [a.key]: e.target.value }))}>
                  <option value="">—</option>
                  {(a.choices ?? []).map((c) => <option key={c} value={c}>{c}</option>)}
                </select>
              )}
              {!a.formula && a.value_type !== "choice" && (
                <div style={row}>
                  <input id={id} value={values[a.key] ?? ""} autoComplete="off" inputMode={a.value_type === "numeric" ? "decimal" : "text"}
                    style={inputStyle} aria-describedby={fieldErrors[a.key] ? `${id}-err` : undefined}
                    onChange={(e) => setValues((v) => ({ ...v, [a.key]: e.target.value }))} />
                  {preview && preview !== "normal" && <Tag tone={flagTone(preview)}>{flagLabel(preview)}</Tag>}
                </div>
              )}
              {referenceText(a) && <p style={muted}>{referenceText(a)}</p>}
              {fieldErrors[a.key] && <p id={`${id}-err`} style={alert}>{fieldErrors[a.key]}</p>}
              {history.length > 0 && (
                <ul aria-label={`valores de ${a.name}`} style={list}>
                  {history.map((r) => (
                    <li key={r.id}>
                      {rowText(r)}
                      {r.flag !== "normal" && <> <Tag tone={flagTone(r.flag)}>{flagLabel(r.flag)}</Tag></>}
                      {r.chosen && <> <Tag tone="ok">vale</Tag></>}
                    </li>
                  ))}
                </ul>
              )}
            </div>
          );
        })}
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      {saved && <p role="status" style={okText}>Resultado salvo.</p>}
      <div style={row}>
        <ActionButton onClick={save}>Salvar resultado</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>
      </div>
    </section>
  );
}
```

```tsx
// src/modules/LabEntry.tsx
// Digitação pela leitura (spec §5.6): lê a etiqueta, escolhe o exame da
// amostra e digita. A leitura do item grava a trilha no api: só acontece
// quando alguém lê uma etiqueta ou salva — nunca por releitura automática.
import { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { getLabEntry, type LabEntryItem } from "../lib/api";
import { itemStatusLabel, labError, refreshLab } from "../lib/lab";
import { Panel } from "../components/Panel";
import { secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabScope } from "./lab/LabScope";
import { ResultEntryForm } from "./lab/ResultEntryForm";
import { ScanField, type ScanOutcome } from "./lab/ScanField";
import { alert, row, stack } from "./exams/styles";

export function LabEntry() {
  return (
    <LabGate title="Digitação de resultados" sub="laboratório municipal · pela etiqueta (pelo mapa: Mapa de trabalho)" allow={(r) => r.staff}>
      {(roles) => <LabScope roles={roles}>{() => <EntryBody />}</LabScope>}
    </LabGate>
  );
}

function EntryBody() {
  const queryClient = useQueryClient();
  const [ items, setItems ] = useState<LabEntryItem[]>([]);
  const [ openId, setOpenId ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);

  async function scan(code: string): Promise<ScanOutcome> {
    try {
      const found = await getLabEntry({ barcode: code });
      setItems(found);
      setOpenId(found.length === 1 ? found[0].item_id : null);
      if (found.length === 0) return { ok: false, text: "nenhum exame para digitar nesta amostra" };
      return { ok: true, text: found.length === 1 ? "1 exame nesta amostra" : `${found.length} exames nesta amostra` };
    } catch (err) {
      return { ok: false, text: labError(err) };
    }
  }

  async function reload(itemId: string) {
    try {
      const [ fresh ] = await getLabEntry({ itemId });
      setItems((list) => (fresh ? list.map((i) => (i.item_id === itemId ? fresh : i)) : list.filter((i) => i.item_id !== itemId)));
      void refreshLab(queryClient);
    } catch (err) {
      setError(labError(err));
    }
  }

  const open = items.find((i) => i.item_id === openId) ?? null;
  return (
    <div style={stack}>
      <Panel title="Ler a etiqueta" sub="cada leitura abre os exames daquela amostra">
        <ScanField label="Etiqueta do tubo" onScan={scan} />
      </Panel>
      {items.length > 1 && (
        <div style={row}>
          {items.map((i) => (
            <button key={i.item_id} type="button" style={secondaryButtonStyle} aria-pressed={i.item_id === openId}
              title={itemStatusLabel(i.status)} onClick={() => setOpenId(i.item_id)}>{i.label}</button>
          ))}
        </div>
      )}
      {error && <p role="alert" style={alert}>{error}</p>}
      {open && <ResultEntryForm key={open.item_id} item={open} onSaved={() => void reload(open.item_id)} onClose={() => setOpenId(null)} />}
    </div>
  );
}
```

Em `src/App.tsx`: `import { LabEntry } from "./modules/LabEntry";` e `case "lab-entry": return <LabEntry />;`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/modules/lab/ResultEntryForm.test.tsx src/modules/LabEntry.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/lab/ResultEntryForm.tsx src/modules/lab/ResultEntryForm.test.tsx src/modules/LabEntry.tsx src/modules/LabEntry.test.tsx src/App.tsx
$G commit -m "feat: add lab result entry with reference preview and entry limits

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Mapa de trabalho com filtros, PDF e digitação pelo mapa

**Files:**
- Create: `src/modules/LabWorklist.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabWorklist.test.tsx`

**Interfaces:**
- Consumes: `listWorklist`, `fetchWorklistPdf`, `getLabEntry`, `LabWorklistGroup`, `LabWorklistItem`, `LabWorklistQuery`, `LabEntryItem` (Task 1); `LAB_WORKLIST_KEY`, `SECTOR_LABEL`, `sectorLabel`, `itemStatusLabel`, `labError` (Task 2); `LabGate`, `openLabPdf` (Task 4); `LabScope` (Task 6); `ResultEntryForm` (Task 10); `todayInCity` (`lib/campaigns`); `fmtDateTime`.
- Produces: `LabWorklist()` em `renderModule` como `case "lab-worklist"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabWorklist.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listWorklist: vi.fn(), fetchWorklistPdf: vi.fn(), getLabEntry: vi.fn() };
});

import * as api from "../lib/api";
import { LabWorklist } from "./LabWorklist";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21B, glucoseEntry, labContext, labUser } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ITEM = { item_id: "it2", requisition_id: "rq1", requisition_number: "2026-000123", barcode: "LB0001235", patient_display_name: "Joana Lima",
  sigtap_code: "0202010473", label: "Glicose", urgent: true, received_at: "2026-10-15T08:00:00-03:00", status: "pending" };

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21B));
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.listWorklist, api.fetchWorklistPdf, api.getLabEntry ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.listWorklist).mockResolvedValue([ { sector: "bioquimica", bench: "B1", items: [ ITEM ] } ]);
  mocked(api.fetchWorklistPdf).mockResolvedValue({ blob: new Blob([ "%PDF" ]), filename: null });
  mocked(api.getLabEntry).mockResolvedValue([ glucoseEntry() ]);
  vi.spyOn(window, "open").mockReturnValue({ location: { href: "" }, close: vi.fn(), opener: null } as unknown as Window);
  Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") });
  Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
});

describe("LabWorklist", () => {
  it("abre com hoje e só pendentes; agrupa por setor e bancada", async () => {
    renderWithProviders(<LabWorklist />);
    const group = await screen.findByRole("region", { name: "Bioquímica · bancada B1" });
    expect(within(group).getByText("2026-000123")).not.toBeNull();
    expect(within(group).getByText("urgente")).not.toBeNull();
    expect(api.listWorklist).toHaveBeenCalledWith({ providerId: "pv9", date: "2026-10-15", pending: true });
  });

  it("filtros vão para a consulta e o PDF usa os mesmos", async () => {
    renderWithProviders(<LabWorklist />);
    await screen.findByRole("region", { name: "Bioquímica · bancada B1" });
    fireEvent.change(screen.getByLabelText("Setor"), { target: { value: "hematologia" } });
    fireEvent.change(screen.getByLabelText("Bancada"), { target: { value: "H2" } });
    fireEvent.click(screen.getByLabelText("Só urgentes"));
    const q = { providerId: "pv9", sector: "hematologia", bench: "H2", urgent: true, date: "2026-10-15", pending: true };
    await waitFor(() => expect(api.listWorklist).toHaveBeenLastCalledWith(q));
    fireEvent.click(screen.getByRole("button", { name: "Baixar PDF do mapa" }));
    await waitFor(() => expect(api.fetchWorklistPdf).toHaveBeenCalledWith(q));
  });

  it("digitar pelo mapa abre o formulário do item", async () => {
    renderWithProviders(<LabWorklist />);
    const group = await screen.findByRole("region", { name: "Bioquímica · bancada B1" });
    fireEvent.click(within(group).getByRole("button", { name: "Digitar" }));
    await waitFor(() => expect(api.getLabEntry).toHaveBeenCalledWith({ itemId: "it2" }));
    expect(await screen.findByLabelText("Glicose (mg/dL)")).not.toBeNull();
  });

  it("o liberador vê o mapa, sem digitar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_releaser" ]));
    renderWithProviders(<LabWorklist />);
    const group = await screen.findByRole("region", { name: "Bioquímica · bancada B1" });
    expect(within(group).queryByRole("button", { name: "Digitar" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabWorklist.test.tsx`
Expected: FAIL — `./LabWorklist` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/LabWorklist.tsx
// Mapa de trabalho (spec §5.5; contrato §5): amostras recebidas por setor e
// bancada, com filtros de urgência, data e pendentes, e o PDF com os mesmos
// filtros. A equipe digita direto do mapa; o liberador só vê.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { fetchWorklistPdf, getLabEntry, listWorklist, type LabEntryItem, type LabWorklistItem, type LabWorklistQuery } from "../lib/api";
import { LAB_WORKLIST_KEY, SECTOR_LABEL, itemStatusLabel, labError, sectorLabel, type LabRoles } from "../lib/lab";
import { todayInCity } from "../lib/campaigns";
import { fmtDateTime } from "../lib/format";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { ActionButton } from "../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabScope } from "./lab/LabScope";
import { ResultEntryForm } from "./lab/ResultEntryForm";
import { openLabPdf } from "./lab/openLabPdf";
import { alert, checkLabel, fieldLabel, muted, row, stack } from "./exams/styles";

export function LabWorklist() {
  return (
    <LabGate title="Mapa de trabalho" sub="laboratório municipal · por setor e bancada" allow={(r) => r.staff || r.releaser}>
      {(roles) => <LabScope roles={roles}>{(scope) => <WorklistBody providerId={scope.providerId} roles={roles} />}</LabScope>}
    </LabGate>
  );
}

function WorklistBody({ providerId, roles }: { providerId: string; roles: LabRoles }) {
  const [ sector, setSector ] = useState("");
  const [ bench, setBench ] = useState("");
  const [ urgent, setUrgent ] = useState(false);
  const [ date, setDate ] = useState(() => todayInCity());
  const [ pending, setPending ] = useState(true);
  const [ entry, setEntry ] = useState<LabEntryItem | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const q: LabWorklistQuery = {
    providerId, ...(sector ? { sector } : {}), ...(bench.trim() ? { bench: bench.trim() } : {}), ...(urgent ? { urgent: true } : {}),
    ...(date ? { date } : {}), ...(pending ? { pending: true } : {})
  };
  const groups = useQuery({ queryKey: [ LAB_WORKLIST_KEY, q ], queryFn: () => listWorklist(q), gcTime: 0, staleTime: 0, retry: false });

  async function openEntry(item: LabWorklistItem) {
    setError(null);
    try {
      const [ found ] = await getLabEntry({ itemId: item.item_id });
      setEntry(found ?? null);
      if (!found) setError("este exame não está mais para digitar — atualize o mapa");
    } catch (err) {
      setError(labError(err));
    }
  }

  async function reloadEntry(itemId: string) {
    try {
      const [ fresh ] = await getLabEntry({ itemId });
      setEntry(fresh ?? null);
      void groups.refetch();
    } catch (err) {
      setError(labError(err));
    }
  }

  async function pdf() {
    setError(null);
    const out = await openLabPdf(() => fetchWorklistPdf(q));
    if (!out.ok) setError(out.text);
  }

  return (
    <div style={stack}>
      <div style={{ ...row, alignItems: "flex-end" }}>
        <label style={fieldLabel}>
          Setor
          <select value={sector} style={inputStyle} onChange={(e) => setSector(e.target.value)}>
            <option value="">todos</option>
            {Object.entries(SECTOR_LABEL).map(([ k, v ]) => <option key={k} value={k}>{v}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>Bancada<input value={bench} style={inputStyle} onChange={(e) => setBench(e.target.value)} /></label>
        <label style={fieldLabel}>Data<input type="date" value={date} style={inputStyle} onChange={(e) => setDate(e.target.value)} /></label>
        <label style={checkLabel}><input type="checkbox" checked={urgent} onChange={(e) => setUrgent(e.target.checked)} />Só urgentes</label>
        <label style={checkLabel}><input type="checkbox" checked={pending} onChange={(e) => setPending(e.target.checked)} />Só pendentes</label>
        <ActionButton variant="secondary" onClick={pdf}>Baixar PDF do mapa</ActionButton>
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      {groups.isPending && <p className="mono" style={muted}>carregando…</p>}
      {groups.isError && <p role="alert" style={alert}>{labError(groups.error)}</p>}
      {groups.isSuccess && groups.data.length === 0 && <p style={muted}>nenhuma amostra no mapa com estes filtros</p>}
      {groups.data?.map((g) => (
        <Panel key={`${g.sector}:${g.bench ?? ""}`} title={`${sectorLabel(g.sector)}${g.bench ? ` · bancada ${g.bench}` : ""}`}>
          <DataTable<LabWorklistItem>
            cols={[
              { label: "Requisição", w: "1fr", card: "title", render: (i) => <span className="mono">{i.requisition_number}</span> },
              { label: "Etiqueta", w: "1fr", render: (i) => <span className="mono">{i.barcode}</span> },
              { label: "Paciente", w: "1.4fr", render: (i) => i.patient_display_name },
              { label: "Exame", w: "1.4fr", render: (i) => <>{i.label}{i.urgent && <> <Tag tone="urgent">urgente</Tag></>}</> },
              { label: "Recebida", w: "1fr", render: (i) => fmtDateTime(i.received_at) },
              { label: "Situação", w: "0.9fr", render: (i) => itemStatusLabel(i.status) },
              ...(roles.staff ? [ { label: "", w: "auto", card: "actions" as const, render: (i: LabWorklistItem) => (
                <button type="button" style={secondaryButtonStyle} onClick={() => void openEntry(i)}>Digitar</button>
              ) } ] : [])
            ]}
            rows={g.items} rowKey={(i) => i.item_id}
          />
        </Panel>
      ))}
      {entry && <ResultEntryForm key={entry.item_id} item={entry} onSaved={() => void reloadEntry(entry.item_id)} onClose={() => setEntry(null)} />}
    </div>
  );
}
```

Em `src/App.tsx`: `import { LabWorklist } from "./modules/LabWorklist";` e `case "lab-worklist": return <LabWorklist />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabWorklist.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabWorklist.tsx src/modules/LabWorklist.test.tsx src/App.tsx
$G commit -m "feat: add lab worklist with filters, PDF and entry from the map

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Liberação — fila, repetições, críticos e assinatura (ICP ou avançada com TOTP)

**Files:**
- Create: `src/modules/LabRelease.tsx`, `src/modules/lab/ReleaseReview.tsx`, `src/modules/lab/SignatureChoice.tsx`, `src/modules/lab/CriticalNoticeForm.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabRelease.test.tsx`

**Interfaces:**
- Consumes: `listReleases`, `getReleaseRequisition`, `listUnnotifiedCriticals`, `recordCriticalNotice`, `chooseLabResult`, `releaseRequisition`, `errorCode`, `LabReleaseRequisition`, `LabReleaseItem`, `LabResultRow`, `LabCriticalRow`, `LabSignatureKind`, `LabReportSummary` (Task 1); `LAB_RELEASES_KEY`, `LAB_CRITICALS_KEY`, `CHANNEL_OPTIONS`, `EMPTY_CRITICAL`, `criticalBody`, `criticalProblem`, `flagLabel`, `flagTone`, `isCritical`, `itemStatusLabel`, `labError`, `onlyIcp`, `refreshLab`, `signatureLabel`, `totpProblem` (Task 2); `LabGate`, `useLabContext` (Task 4); `LabScope` (Task 6); `fmtDay` (21a); `cityIsoDate`, `fmtHourMinute`, `fmtDateTime` (`format.ts`); `ModuleId`.
- Produces:
  - `SignatureChoice({ council, kind, onKind, totp, onTotp })` — `kind: LabSignatureKind`.
  - `cityNowLocal(): string` (valor de `datetime-local` na hora da cidade) e `CriticalNoticeForm({ resultId, label, onDone, onCancel })` em `src/modules/lab/CriticalNoticeForm.tsx`.
  - `ReleaseReview({ requisitionId, council, onReleased, onConflict, onNavigate, onClose })` — `onReleased(report: LabReportSummary)`, `onConflict(text: string)`, `onNavigate(id: ModuleId)`. A Task 13 acrescenta a lista de laudos e a retificação dentro dele.
  - `LabRelease({ onNavigate })` em `renderModule` como `case "lab-release": return <LabRelease onNavigate={setActive} />`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabRelease.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), listReleases: vi.fn(), getReleaseRequisition: vi.fn(),
    listUnnotifiedCriticals: vi.fn(), recordCriticalNotice: vi.fn(), chooseLabResult: vi.fn(), releaseRequisition: vi.fn(), listLabRequisitions: vi.fn() };
});

import * as api from "../lib/api";
import { LabRelease } from "./LabRelease";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21B, labContext, labItem, labReport, labResultRow, labUser, releaseRequisition } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getLabContext, api.listReleases, api.getReleaseRequisition, api.listUnnotifiedCriticals,
  api.recordCriticalNotice, api.chooseLabResult, api.releaseRequisition, api.listLabRequisitions ];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_releaser" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext({ releaser_of: [ "pv9" ], council: "CRF" }));
  mocked(api.listReleases).mockResolvedValue([ releaseRequisition() ]);
  mocked(api.getReleaseRequisition).mockResolvedValue(releaseRequisition());
  mocked(api.listUnnotifiedCriticals).mockResolvedValue([]);
  mocked(api.releaseRequisition).mockResolvedValue(labReport());
  mocked(api.listLabRequisitions).mockResolvedValue({ requisitions: [], next_before: null });
});

async function openReview() {
  renderWithProviders(<LabRelease onNavigate={vi.fn()} />);
  const queue = await screen.findByRole("region", { name: "Fila de liberação" });
  fireEvent.click(within(queue).getByRole("button", { name: "Conferir" }));
  return screen.findByRole("region", { name: "Conferência da requisição 2026-000123" });
}

describe("LabRelease", () => {
  it("assinatura avançada: manda o TOTP e apaga o código; laudo liberado relê a fila", async () => {
    const review = await openReview();
    fireEvent.click(within(review).getByLabelText("Assinatura eletrônica avançada (código do autenticador)"));
    fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    await waitFor(() => expect(api.releaseRequisition).toHaveBeenCalledWith("rq1", { item_ids: [ "it2" ], signature: "advanced", totp: "123456" }));
    expect(await screen.findByText("Laudo liberado com assinatura eletrônica avançada.")).not.toBeNull();
    await waitFor(() => expect(api.listReleases).toHaveBeenCalledTimes(2));
  });

  it("assinatura avançada com código recusado: apaga o código e pede de novo", async () => {
    mocked(api.releaseRequisition).mockRejectedValueOnce(new api.ApiError(401, { error: "mfa_required" }, "x"));
    const review = await openReview();
    fireEvent.click(within(review).getByLabelText("Assinatura eletrônica avançada (código do autenticador)"));
    fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "654321" } });
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    expect((await within(review).findByRole("alert")).textContent).toBe("o código do autenticador não valeu — digite o código atual");
    expect((within(review).getByLabelText("Código do autenticador") as HTMLInputElement).value).toBe("");
    expect(api.releaseRequisition).toHaveBeenCalledTimes(1);
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    expect(within(review).getByRole("alert").textContent).toBe("digite os 6 dígitos do autenticador");
    expect(api.releaseRequisition).toHaveBeenCalledTimes(1);
  });

  it("409 invalid_transition na liberação: explica, limpa o código e relê a fila", async () => {
    mocked(api.releaseRequisition).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    const review = await openReview();
    fireEvent.click(within(review).getByLabelText("Assinatura eletrônica avançada (código do autenticador)"));
    fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    expect(await screen.findByText("a situação mudou — a tela foi atualizada")).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Conferência da requisição 2026-000123" })).toBeNull();
    await waitFor(() => expect(api.listReleases).toHaveBeenCalledTimes(2));
  });

  it("médico (CRM): só ICP; o laudo vai para Pendentes de assinatura", async () => {
    mocked(api.getLabContext).mockResolvedValue(labContext({ releaser_of: [ "pv9" ], council: "CRM" }));
    mocked(api.releaseRequisition).mockResolvedValue(labReport({ status: "pending_signature", signature_kind: "icp", pdf_url: null, released_at: null }));
    const onNavigate = vi.fn();
    renderWithProviders(<LabRelease onNavigate={onNavigate} />);
    fireEvent.click(within(await screen.findByRole("region", { name: "Fila de liberação" })).getByRole("button", { name: "Conferir" }));
    const review = await screen.findByRole("region", { name: "Conferência da requisição 2026-000123" });
    expect(within(review).queryByLabelText("Assinatura eletrônica avançada (código do autenticador)")).toBeNull();
    expect(within(review).getByText("médico (CRM) libera só com ICP-Brasil")).not.toBeNull();
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    await waitFor(() => expect(api.releaseRequisition).toHaveBeenCalledWith("rq1", { item_ids: [ "it2" ], signature: "icp" }));
    fireEvent.click(await screen.findByRole("button", { name: "Ir para Pendentes de assinatura" }));
    expect(onNavigate).toHaveBeenCalledWith("signature-pending");
  });

  it("repetição: escolher o valor que vale relê a requisição", async () => {
    mocked(api.getReleaseRequisition).mockResolvedValue(releaseRequisition({ items: [ {
      ...labItem({ id: "it2", label: "Glicose", status: "typed" }), restriction_note: "lipêmica",
      results: [ labResultRow({ id: "r1", value: "410", source: "hl7", chosen: true, flag: "critical_high", notified: true }),
        labResultRow({ id: "r2", value: "98", chosen: false }) ]
    } ] }));
    mocked(api.chooseLabResult).mockResolvedValue(labResultRow({ id: "r3", value: "98", chosen: true }));
    const review = await openReview();
    expect(within(review).getByText("restrição da amostra: lipêmica")).not.toBeNull();
    const values = within(review).getByRole("list", { name: "Glicose · valores" });
    expect(within(values).getAllByRole("listitem")[0].textContent).toContain("aparelho");
    fireEvent.click(within(values).getByRole("button", { name: "Usar 98" }));
    await waitFor(() => expect(api.chooseLabResult).toHaveBeenCalledWith("r2"));
    await waitFor(() => expect(api.getReleaseRequisition).toHaveBeenCalledTimes(2));
  });

  it("crítico sem aviso: confirma antes de liberar, ou registra o aviso", async () => {
    // "Quando" não pode ser no futuro: o relógio da cidade fica em 15/10/2026 10:00.
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW21B));
    mocked(api.getReleaseRequisition).mockResolvedValue(releaseRequisition({ items: [ {
      ...labItem({ id: "it2", label: "Glicose", status: "typed" }), restriction_note: null,
      results: [ labResultRow({ id: "r1", value: "450", flag: "critical_high", chosen: true, notified: false }) ]
    } ] }));
    const review = await openReview();
    fireEvent.click(within(review).getByLabelText("Assinatura eletrônica avançada (código do autenticador)"));
    fireEvent.change(within(review).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(review).getByRole("button", { name: "Liberar laudo" }));
    expect(within(review).getByText("1 valor crítico sem registro de aviso. Liberar mesmo assim? O aviso fica pendente em Críticos sem aviso.")).not.toBeNull();
    expect(api.releaseRequisition).not.toHaveBeenCalled();
    fireEvent.click(within(review).getByRole("button", { name: "Registrar o aviso antes" }));
    const form = within(review).getByRole("region", { name: "Aviso do valor crítico — Glicose" });
    fireEvent.change(within(form).getByLabelText("Quem foi avisado"), { target: { value: "Dra. Ana" } });
    fireEvent.change(within(form).getByLabelText("Função"), { target: { value: "médica da UBS" } });
    fireEvent.change(within(form).getByLabelText("Como"), { target: { value: "telefone" } });
    fireEvent.change(within(form).getByLabelText("Quando"), { target: { value: "2026-10-15T09:40" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar aviso" }));
    await waitFor(() => expect(api.recordCriticalNotice).toHaveBeenCalledWith({
      result_id: "r1", notified_person: "Dra. Ana", notified_role: "médica da UBS", channel: "telefone", notified_at: "2026-10-15T09:40:00-03:00"
    }));
  });

  it("sem releitura automática: a fila e os críticos são lidos uma vez", async () => {
    vi.useFakeTimers({ shouldAdvanceTime: true });
    renderWithProviders(<LabRelease onNavigate={vi.fn()} />);
    await screen.findByRole("region", { name: "Fila de liberação" });
    await act(async () => { vi.advanceTimersByTime(10 * 60_000); });
    expect(api.listReleases).toHaveBeenCalledTimes(1);
    expect(api.listUnnotifiedCriticals).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabRelease.test.tsx`
Expected: FAIL — `./LabRelease` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/SignatureChoice.tsx
// ICP-Brasil (19b; o laudo espera a assinatura em Pendentes) ou assinatura
// eletrônica avançada do Rota Saúde (código do autenticador na hora). Médico
// (CRM) só ICP — o api confere de novo. O código vive só no estado de quem
// chama; aqui não se guarda nada.
import type { LabSignatureKind } from "../../lib/api";
import { onlyIcp } from "../../lib/lab";
import { inputStyle } from "../../components/formStyles";
import { checkLabel, fieldLabel, fieldset, legend, muted } from "../exams/styles";

export function SignatureChoice({ council, kind, onKind, totp, onTotp }: {
  council: string | null; kind: LabSignatureKind; onKind(k: LabSignatureKind): void; totp: string; onTotp(code: string): void;
}) {
  const icpOnly = onlyIcp(council);
  return (
    <fieldset style={fieldset}>
      <legend style={legend}>Assinatura do laudo</legend>
      <label style={checkLabel}>
        <input type="radio" name="lab-signature" checked={kind === "icp"} onChange={() => onKind("icp")} />
        ICP-Brasil (assina em Pendentes de assinatura)
      </label>
      {!icpOnly && (
        <label style={checkLabel}>
          <input type="radio" name="lab-signature" checked={kind === "advanced"} onChange={() => onKind("advanced")} />
          Assinatura eletrônica avançada (código do autenticador)
        </label>
      )}
      {icpOnly && <p style={muted}>médico (CRM) libera só com ICP-Brasil</p>}
      {kind === "advanced" && !icpOnly && (
        <label style={fieldLabel}>
          Código do autenticador
          <input value={totp} inputMode="numeric" autoComplete="one-time-code" maxLength={6} style={inputStyle}
            onChange={(e) => onTotp(e.target.value.replace(/\D/g, ""))} />
        </label>
      )}
    </fieldset>
  );
}
```

```tsx
// src/modules/lab/CriticalNoticeForm.tsx
// Registro da comunicação do valor crítico (spec §5.7; contrato §5): quem foi
// avisado, a função, o canal e a hora (padrão: agora, na hora da cidade).
import { useState } from "react";
import { recordCriticalNotice } from "../../lib/api";
import { CHANNEL_OPTIONS, EMPTY_CRITICAL, criticalBody, criticalProblem, labError, type CriticalDraft } from "../../lib/lab";
import { cityIsoDate, fmtHourMinute } from "../../lib/format";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, row } from "../exams/styles";

export function cityNowLocal(): string {
  return `${cityIsoDate()}T${fmtHourMinute(new Date().toISOString())}`;
}

export function CriticalNoticeForm({ resultId, label, onDone, onCancel }: { resultId: string; label: string; onDone(): void; onCancel(): void }) {
  const [ draft, setDraft ] = useState<CriticalDraft>(() => ({ ...EMPTY_CRITICAL, at: cityNowLocal() }));
  const [ error, setError ] = useState<string | null>(null);
  const set = (patch: Partial<CriticalDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function save() {
    const problem = criticalProblem(draft, new Date().toISOString());
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await recordCriticalNotice(criticalBody(resultId, draft));
      onDone();
    } catch (err) {
      setError(labError(err));
    }
  }

  return (
    <section aria-label={`Aviso do valor crítico — ${label}`} style={box}>
      <label style={fieldLabel}>Quem foi avisado<input value={draft.person} style={inputStyle} onChange={(e) => set({ person: e.target.value })} /></label>
      <label style={fieldLabel}>Função<input value={draft.role} style={inputStyle} onChange={(e) => set({ role: e.target.value })} /></label>
      <label style={fieldLabel}>
        Como
        <select value={draft.channel} style={inputStyle} onChange={(e) => set({ channel: e.target.value as CriticalDraft["channel"] })}>
          <option value="">—</option>
          {CHANNEL_OPTIONS.map((c) => <option key={c.value} value={c.value}>{c.label}</option>)}
        </select>
      </label>
      <label style={fieldLabel}>Quando<input type="datetime-local" value={draft.at} style={inputStyle} onChange={(e) => set({ at: e.target.value })} /></label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton onClick={save}>Registrar aviso</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}
```

```tsx
// src/modules/lab/ReleaseReview.tsx
// Conferência e liberação de uma requisição (spec §5.6–§5.7; contrato §5):
// valores com marca e origem (aparelho chega "aguardando conferência"),
// repetições (o liberador escolhe a que vale), restrição da amostra, críticos
// (aviso antes ou confirmação de liberar sem aviso), exames a liberar
// (parcial quando nem todos) e a assinatura. O código do autenticador é
// apagado antes de cada envio; 409 devolve à fila.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import {
  chooseLabResult, errorCode, getReleaseRequisition, releaseRequisition,
  type LabReleaseItem, type LabReportSummary, type LabResultRow, type LabSignatureKind
} from "../../lib/api";
import { LAB_RELEASES_KEY, flagLabel, flagTone, isCritical, itemStatusLabel, labError, onlyIcp, totpProblem } from "../../lib/lab";
import { fmtDay } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import type { ModuleId } from "../../shell/modules";
import { ActionButton } from "../../components/ActionButton";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { CriticalNoticeForm } from "./CriticalNoticeForm";
import { SignatureChoice } from "./SignatureChoice";
import { alert, box, checkLabel, list, muted, panel, row, title } from "../exams/styles";

function byAnalyte(rows: LabResultRow[]): [ string, LabResultRow[] ][] {
  const groups = new Map<string, LabResultRow[]>();
  for (const r of rows) groups.set(r.analyte_key, [ ...(groups.get(r.analyte_key) ?? []), r ]);
  return [ ...groups.entries() ];
}

export function ReleaseReview({ requisitionId, council, onReleased, onConflict, onNavigate, onClose }: {
  requisitionId: string; council: string | null; onReleased(r: LabReportSummary): void; onConflict(text: string): void;
  onNavigate(id: ModuleId): void; onClose(): void;
}) {
  const query = useQuery({
    queryKey: [ LAB_RELEASES_KEY, "one", requisitionId ], queryFn: () => getReleaseRequisition(requisitionId), gcTime: 0, staleTime: 0, retry: false
  });
  const [ selected, setSelected ] = useState<string[] | null>(null);
  const [ kind, setKind ] = useState<LabSignatureKind>("icp");
  const [ totp, setTotp ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ certificateLink, setCertificateLink ] = useState(false);
  const [ confirming, setConfirming ] = useState(false);
  const [ noticeFor, setNoticeFor ] = useState<LabResultRow | null>(null);
  const r = query.data;

  if (query.isPending) return <p className="mono" style={muted}>carregando a requisição…</p>;
  if (query.isError || !r) return <p role="alert" style={alert}>{labError(query.error)}</p>;

  const releasable = r.items.filter((i) => i.status === "typed");
  const chosenIds = selected ?? releasable.map((i) => i.id);
  const signature: LabSignatureKind = onlyIcp(council) ? "icp" : kind;
  const unnotified = r.items
    .filter((i) => chosenIds.includes(i.id))
    .flatMap((i) => i.results.filter((x) => x.chosen && isCritical(x.flag) && x.notified === false));
  const partial = chosenIds.length < r.items.filter((i) => i.status !== "cancelled" && i.status !== "released").length;

  async function choose(row: LabResultRow) {
    setError(null);
    try { await chooseLabResult(row.id); await query.refetch(); } catch (err) { setError(labError(err)); }
  }

  async function release(confirmed: boolean) {
    if (chosenIds.length === 0) { setError("escolha ao menos um exame"); return; }
    if (signature === "advanced") {
      const problem = totpProblem(totp);
      if (problem) { setError(problem); return; }
    }
    if (unnotified.length > 0 && !confirmed) { setConfirming(true); return; }
    setConfirming(false);
    setError(null);
    setCertificateLink(false);
    const code = totp;
    setTotp("");
    try {
      const report = await releaseRequisition(r!.id, signature === "advanced"
        ? { item_ids: chosenIds, signature, totp: code }
        : { item_ids: chosenIds, signature });
      onReleased(report);
    } catch (err) {
      const reason = errorCode(err);
      if (reason === "invalid_transition") { onConflict(labError(err)); return; }
      if (reason === "icp_required") setKind("icp");
      if (reason === "no_active_certificate") setCertificateLink(true);
      setError(labError(err));
    }
  }

  const itemBlock = (i: LabReleaseItem) => (
    <div key={i.id} style={box}>
      <label style={checkLabel}>
        <input type="checkbox" disabled={i.status !== "typed"} checked={chosenIds.includes(i.id)}
          onChange={(e) => setSelected(e.target.checked ? [ ...chosenIds, i.id ] : chosenIds.filter((x) => x !== i.id))} />
        <strong>{i.label}</strong> <span style={muted}>{`v${i.version} · ${itemStatusLabel(i.status)}`}</span>
      </label>
      {i.restriction_note && <p style={alert}>{`restrição da amostra: ${i.restriction_note}`}</p>}
      {byAnalyte(i.results).map(([ key, rows ]) => (
        <ul key={key} aria-label={`${rows[0].name} · valores`} style={list}>
          {rows.map((x) => (
            <li key={x.id} style={row}>
              <span>{`${x.value}${x.unit ? ` ${x.unit}` : ""} · ${x.source === "hl7" ? "aparelho — conferir" : `digitado${x.entered_by ? ` por ${x.entered_by}` : ""}`} · ${fmtDateTime(x.entered_at)}`}</span>
              {x.flag !== "normal" && <Tag tone={flagTone(x.flag)}>{flagLabel(x.flag)}</Tag>}
              {x.chosen ? <Tag tone="ok">vale</Tag> : rows.length > 1 && (
                <button type="button" style={secondaryButtonStyle} onClick={() => void choose(x)}>{`Usar ${x.value}`}</button>
              )}
              {x.chosen && isCritical(x.flag) && x.notified === false && (
                <button type="button" style={secondaryButtonStyle} onClick={() => setNoticeFor(x)}>Registrar aviso</button>
              )}
            </li>
          ))}
        </ul>
      ))}
      {noticeFor && i.results.some((x) => x.id === noticeFor.id) && (
        <CriticalNoticeForm resultId={noticeFor.id} label={noticeFor.name}
          onCancel={() => setNoticeFor(null)} onDone={() => { setNoticeFor(null); void query.refetch(); }} />
      )}
    </div>
  );

  return (
    <section aria-label={`Conferência da requisição ${r.number}`} style={panel}>
      <div style={row}>
        <h3 style={title}>{`Requisição ${r.number}`}</h3>
        {r.urgent && <Tag tone="urgent">urgente</Tag>}
      </div>
      <p style={muted}>{`${r.patient.display_name} · nascimento ${fmtDay(r.patient.birth_date)}`}</p>
      {r.items.map(itemBlock)}
      <SignatureChoice council={council} kind={signature} onKind={setKind} totp={totp} onTotp={setTotp} />
      {confirming && (
        <div role="group" aria-label="crítico sem aviso" style={box}>
          <p style={{ margin: 0 }}>{`${unnotified.length === 1 ? "1 valor crítico" : `${unnotified.length} valores críticos`} sem registro de aviso. Liberar mesmo assim? O aviso fica pendente em Críticos sem aviso.`}</p>
          <div style={row}>
            <button type="button" style={buttonStyle} onClick={() => void release(true)}>Liberar sem aviso</button>
            <button type="button" style={secondaryButtonStyle} onClick={() => { setConfirming(false); setNoticeFor(unnotified[0]); }}>Registrar o aviso antes</button>
          </div>
        </div>
      )}
      {error && <p role="alert" style={alert}>{error}</p>}
      {certificateLink && (
        <div><button type="button" style={secondaryButtonStyle} onClick={() => onNavigate("signature")}>Ir para Assinatura digital</button></div>
      )}
      <div style={row}>
        <ActionButton onClick={() => release(false)}>
          {partial ? `Liberar laudo parcial (${chosenIds.length} de ${r.items.length})` : "Liberar laudo"}
        </ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>
      </div>
    </section>
  );
}
```

```tsx
// src/modules/LabRelease.tsx
// Liberação de laudos (spec §5.7; contrato §5): críticos sem aviso, fila de
// liberação e a conferência. Leitura só ao abrir e depois de cada ação —
// nenhuma releitura automática (as rotas gravam trilha); o contador do menu
// é a rota /count (Task 3).
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listReleases, listUnnotifiedCriticals, type LabCriticalRow, type LabReleaseRequisition, type LabReportSummary } from "../lib/api";
import { LAB_CRITICALS_KEY, LAB_RELEASES_KEY, flagLabel, labError, refreshLab } from "../lib/lab";
import { fmtDateTime } from "../lib/format";
import type { ModuleId } from "../shell/modules";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { secondaryButtonStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabScope } from "./lab/LabScope";
import { CriticalNoticeForm } from "./lab/CriticalNoticeForm";
import { ReleaseReview } from "./lab/ReleaseReview";
import { useLabContext } from "./lab/useLabContext";
import { alert, muted, okText, row, stack } from "./exams/styles";

export function LabRelease({ onNavigate }: { onNavigate(id: ModuleId): void }) {
  return (
    <LabGate title="Liberação de laudos" sub="laboratório municipal · conferir, assinar e liberar" allow={(r) => r.releaser}>
      {(roles) => <LabScope roles={roles}>{(scope) => <ReleaseBody providerId={scope.providerId} onNavigate={onNavigate} />}</LabScope>}
    </LabGate>
  );
}

function ReleaseBody({ providerId, onNavigate }: { providerId: string; onNavigate(id: ModuleId): void }) {
  const queryClient = useQueryClient();
  const context = useLabContext(true);
  const council = context.data?.council ?? null;
  const queue = useQuery({ queryKey: [ LAB_RELEASES_KEY, providerId ], queryFn: () => listReleases(providerId), gcTime: 0, staleTime: 0, retry: false });
  const criticals = useQuery({ queryKey: [ LAB_CRITICALS_KEY ], queryFn: listUnnotifiedCriticals, gcTime: 0, staleTime: 0, retry: false });
  const [ openId, setOpenId ] = useState<string | null>(null);
  const [ noticeFor, setNoticeFor ] = useState<LabCriticalRow | null>(null);
  const [ message, setMessage ] = useState<{ kind: "ok" | "error"; text: string; pending?: boolean } | null>(null);

  async function after() {
    await refreshLab(queryClient);
  }

  function released(rep: LabReportSummary) {
    setOpenId(null);
    setMessage(rep.status === "pending_signature"
      ? { kind: "ok", text: "Laudo enviado para assinatura ICP — assine em Pendentes de assinatura.", pending: true }
      : { kind: "ok", text: "Laudo liberado com assinatura eletrônica avançada." });
    void after();
  }

  return (
    <div style={stack}>
      {message && (
        <div style={row}>
          <p role={message.kind === "error" ? "alert" : "status"} style={message.kind === "error" ? alert : okText}>{message.text}</p>
          {message.pending && (
            <button type="button" style={secondaryButtonStyle} onClick={() => onNavigate("signature-pending")}>Ir para Pendentes de assinatura</button>
          )}
        </div>
      )}
      <Panel title="Críticos sem aviso" sub="liberados sem o registro de quem foi avisado — registre o aviso">
        {criticals.isError ? <p role="alert" style={alert}>{labError(criticals.error)}</p> : (
          <DataTable<LabCriticalRow>
            cols={[
              { label: "Requisição", w: "1fr", card: "title", render: (c) => <span className="mono">{c.requisition_number}</span> },
              { label: "Paciente", w: "1.4fr", render: (c) => c.patient_display_name },
              { label: "Componente", w: "1.2fr", render: (c) => <>{c.analyte_name} <Tag tone="down">{flagLabel(c.flag)}</Tag></> },
              { label: "Liberado", w: "1fr", render: (c) => fmtDateTime(c.released_at) },
              { label: "", w: "auto", card: "actions", render: (c) => (
                <button type="button" style={secondaryButtonStyle} onClick={() => setNoticeFor(c)}>Registrar aviso</button>
              ) }
            ]}
            rows={criticals.data ?? []} rowKey={(c) => c.result_id} empty={criticals.isPending ? "carregando…" : "nenhum crítico sem aviso"}
          />
        )}
        {noticeFor && (
          <CriticalNoticeForm resultId={noticeFor.result_id} label={noticeFor.analyte_name} onCancel={() => setNoticeFor(null)}
            onDone={() => { setNoticeFor(null); setMessage({ kind: "ok", text: "Aviso registrado." }); void criticals.refetch(); void after(); }} />
        )}
      </Panel>
      <Panel title="Fila de liberação" sub="exames digitados ou recebidos do aparelho, esperando conferência">
        {queue.isError ? <p role="alert" style={alert}>{labError(queue.error)}</p> : (
          <DataTable<LabReleaseRequisition>
            cols={[
              { label: "Requisição", w: "1fr", card: "title", render: (q) => <span className="mono">{q.number}</span> },
              { label: "Paciente", w: "1.6fr", render: (q) => q.patient.display_name },
              { label: "Exames", w: "0.8fr", align: "right", render: (q) => String(q.items.length) },
              { label: "", w: "1fr", render: (q) => <>{q.urgent && <Tag tone="urgent">urgente</Tag>}{q.items.some((i) => i.results.some((x) => x.chosen && (x.flag === "critical_low" || x.flag === "critical_high"))) && <> <Tag tone="down">crítico</Tag></>}</> },
              { label: "", w: "auto", card: "actions", render: (q) => (
                <button type="button" style={secondaryButtonStyle} onClick={() => { setMessage(null); setOpenId(q.id); }}>Conferir</button>
              ) }
            ]}
            rows={queue.data ?? []} rowKey={(q) => q.id} empty={queue.isPending ? "carregando…" : "nada esperando liberação"}
          />
        )}
      </Panel>
      {context.isError && <p role="alert" style={alert}>{labError(context.error)}</p>}
      {openId && !context.isPending && (
        <ReleaseReview key={openId} requisitionId={openId} council={council} onNavigate={onNavigate} onClose={() => setOpenId(null)}
          onReleased={released}
          onConflict={(text) => { setOpenId(null); setMessage({ kind: "error", text }); void queue.refetch(); }} />
      )}
      {!openId && <p style={muted}>abra uma requisição da fila para conferir os valores e liberar o laudo</p>}
    </div>
  );
}
```

> `released()` chama `refreshLab`, que invalida `LAB_RELEASES_KEY` e relê a fila (o teste conta duas leituras). No 409, `queue.refetch()` relê; a conferência fecha e o código já foi apagado antes do envio.

Em `src/App.tsx`: `import { LabRelease } from "./modules/LabRelease";` e `case "lab-release": return <LabRelease onNavigate={setActive} />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabRelease.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabRelease.tsx src/modules/LabRelease.test.tsx src/modules/lab/ReleaseReview.tsx src/modules/lab/SignatureChoice.tsx \
  src/modules/lab/CriticalNoticeForm.tsx src/App.tsx
$G commit -m "feat: add lab report release with ICP or advanced signature and critical notices

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Laudos da requisição, PDF e retificação

**Files:**
- Create: `src/modules/lab/LabReportList.tsx`, `src/modules/lab/RectifyForm.tsx`
- Modify: `src/modules/lab/ReleaseReview.tsx`, `src/modules/LabRelease.tsx`
- Test: `src/modules/lab/RectifyForm.test.tsx`

**Interfaces:**
- Consumes: `rectifyLabReport`, `listLabRequisitions`, `errorCode`, `LabReportSummary`, `LabReleaseItem`, `LabSignatureKind`, `CriticalChannel` (Task 1); `CHANNEL_OPTIONS`, `contactsBody`, `rectifyProblem`, `reportStatusLabel`, `signatureLabel`, `labError`, `onlyIcp`, `totpProblem`, `type ContactDraft` (Task 2); `SignatureChoice`, `cityNowLocal`, `ReleaseReview` (Task 12); `openExamPdf` (21a).
- Produces: `LabReportList({ reports, onRectify, onExpired })`; `RectifyForm({ report, items, council, onDone, onConflict, onCancel })` — `onDone(r: LabReportSummary)`, `onConflict(text: string)`; `ReleaseReview` mostra os laudos e abre a retificação; `LabRelease` ganha o painel "Laudo já liberado" (busca pelo número).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/lab/RectifyForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), rectifyLabReport: vi.fn(), fetchExamPdf: vi.fn() };
});

import * as api from "../../lib/api";
import { RectifyForm } from "./RectifyForm";
import { LabReportList } from "./LabReportList";
import { renderWithProviders } from "../../test/campaignFixtures";
import { NOW21B, labItem, labReport, labResultRow, labUser } from "../../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ITEMS = [ { ...labItem({ id: "it2", label: "Glicose", status: "released" }), restriction_note: null, results: [ labResultRow() ] } ];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21B));
  mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(labUser([ "lab_releaser" ]));
  mocked(api.rectifyLabReport).mockReset().mockResolvedValue(labReport({ id: "lr2", version: 2, replaces_report_id: "lr1" }));
});

function fill(form: HTMLElement) {
  fireEvent.change(within(form).getByLabelText("Motivo da retificação"), { target: { value: "valor de glicose digitado trocado" } });
  fireEvent.click(within(form).getByRole("button", { name: "Incluir contato" }));
  const contact = within(form).getByRole("group", { name: "Contato 1" });
  fireEvent.change(within(contact).getByLabelText("Quem foi avisado"), { target: { value: "Dra. Ana" } });
  fireEvent.change(within(contact).getByLabelText("Como"), { target: { value: "telefone" } });
  fireEvent.change(within(contact).getByLabelText("Quando"), { target: { value: "2026-10-15T09:00" } });
}

describe("RectifyForm", () => {
  it("exige motivo e quem foi avisado antes de enviar", async () => {
    renderWithProviders(<RectifyForm report={labReport()} items={ITEMS} council="CRF" onDone={vi.fn()} onConflict={vi.fn()} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Retificar laudo v1" });
    fireEvent.click(within(form).getByRole("button", { name: "Emitir laudo retificado" }));
    expect(within(form).getByRole("alert").textContent).toBe("escreva o motivo da retificação (ao menos 10 caracteres)");
    fireEvent.change(within(form).getByLabelText("Motivo da retificação"), { target: { value: "valor de glicose digitado trocado" } });
    fireEvent.click(within(form).getByRole("button", { name: "Emitir laudo retificado" }));
    expect(within(form).getByRole("alert").textContent).toBe("registre quem foi avisado da retificação");
    expect(api.rectifyLabReport).not.toHaveBeenCalled();
  });

  it("assinatura avançada: contatos com a hora da cidade e o código só no corpo", async () => {
    const onDone = vi.fn();
    renderWithProviders(<RectifyForm report={labReport()} items={ITEMS} council="CRF" onDone={onDone} onConflict={vi.fn()} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Retificar laudo v1" });
    fill(form);
    fireEvent.click(within(form).getByLabelText("Assinatura eletrônica avançada (código do autenticador)"));
    fireEvent.change(within(form).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(form).getByRole("button", { name: "Emitir laudo retificado" }));
    await waitFor(() => expect(api.rectifyLabReport).toHaveBeenCalledWith("lr1", {
      item_ids: [ "it2" ], reason: "valor de glicose digitado trocado",
      contacts: [ { person: "Dra. Ana", channel: "telefone", at: "2026-10-15T09:00:00-03:00" } ], signature: "advanced", totp: "123456"
    }));
    expect(onDone).toHaveBeenCalledWith(labReport({ id: "lr2", version: 2, replaces_report_id: "lr1" }));
  });

  it("código recusado: apaga e pede de novo; 409 devolve a quem chamou", async () => {
    mocked(api.rectifyLabReport)
      .mockRejectedValueOnce(new api.ApiError(401, { error: "mfa_required" }, "x"))
      .mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    const onConflict = vi.fn();
    renderWithProviders(<RectifyForm report={labReport()} items={ITEMS} council="CRF" onDone={vi.fn()} onConflict={onConflict} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Retificar laudo v1" });
    fill(form);
    fireEvent.click(within(form).getByLabelText("Assinatura eletrônica avançada (código do autenticador)"));
    fireEvent.change(within(form).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(form).getByRole("button", { name: "Emitir laudo retificado" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("o código do autenticador não valeu — digite o código atual");
    expect((within(form).getByLabelText("Código do autenticador") as HTMLInputElement).value).toBe("");
    fireEvent.change(within(form).getByLabelText("Código do autenticador"), { target: { value: "222222" } });
    fireEvent.click(within(form).getByRole("button", { name: "Emitir laudo retificado" }));
    await waitFor(() => expect(onConflict).toHaveBeenCalledWith("a situação mudou — a tela foi atualizada"));
  });
});

describe("LabReportList", () => {
  it("liberado pode ser retificado; retificado e pendente não", async () => {
    const onRectify = vi.fn();
    renderWithProviders(<LabReportList onRectify={onRectify} onExpired={vi.fn()} reports={[
      labReport({ id: "lr2", version: 2, replaces_report_id: "lr1" }),
      labReport({ id: "lr1", status: "rectified" }),
      labReport({ id: "lr3", version: 3, status: "pending_signature", signature_kind: "icp", pdf_url: null })
    ]} />);
    const list = await screen.findByRole("list", { name: "laudos" });
    const items = within(list).getAllByRole("listitem");
    expect(items[0].textContent).toContain("substitui a v1");
    expect(items[1].textContent).toContain("retificado (substituído)");
    expect(items[2].textContent).toContain("aguardando assinatura ICP");
    expect(within(items[1]).queryByRole("button", { name: "Retificar" })).toBeNull();
    expect(within(items[2]).queryByRole("button", { name: "Retificar" })).toBeNull();
    fireEvent.click(within(items[0]).getByRole("button", { name: "Retificar" }));
    expect(onRectify).toHaveBeenCalledWith(expect.objectContaining({ id: "lr2" }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/lab/RectifyForm.test.tsx`
Expected: FAIL — módulos inexistentes.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/lab/LabReportList.tsx
// Laudos de uma requisição: versão, parcial, situação, assinatura e o PDF
// (URL curta assinada — vencida relê). Só o laudo `released` se retifica.
import { useState } from "react";
import type { LabReportSummary } from "../../lib/api";
import { reportStatusLabel, signatureLabel } from "../../lib/lab";
import { fmtDateTime } from "../../lib/format";
import { secondaryButtonStyle } from "../../components/formStyles";
import { openExamPdf } from "../exams/openExamPdf";
import { alert, list, row } from "../exams/styles";

export function LabReportList({ reports, onRectify, onExpired }: {
  reports: LabReportSummary[]; onRectify(r: LabReportSummary): void; onExpired(): void;
}) {
  const [ error, setError ] = useState<string | null>(null);
  const byId = new Map(reports.map((r) => [ r.id, r ]));
  async function open(r: LabReportSummary) {
    if (!r.pdf_url) return;
    setError(null);
    const out = await openExamPdf(r.pdf_url);
    if (!out.ok) { setError(out.text); if (out.reread) onExpired(); }
  }
  return (
    <>
      <ul aria-label="laudos" style={list}>
        {reports.map((r) => (
          <li key={r.id} style={row}>
            <span>
              {`Laudo v${r.version}${r.partial ? " (parcial)" : ""} · ${reportStatusLabel(r.status)} · ${signatureLabel(r.signature_kind)}`}
              {r.released_at ? ` · ${fmtDateTime(r.released_at)}${r.released_by ? ` por ${r.released_by}` : ""}` : ""}
              {r.replaces_report_id ? ` · substitui a v${byId.get(r.replaces_report_id)?.version ?? "anterior"}` : ""}
            </span>
            {r.pdf_url && <button type="button" style={secondaryButtonStyle} onClick={() => void open(r)}>Abrir laudo</button>}
            {r.status === "released" && <button type="button" style={secondaryButtonStyle} onClick={() => onRectify(r)}>Retificar</button>}
          </li>
        ))}
      </ul>
      {error && <p role="alert" style={alert}>{error}</p>}
    </>
  );
}
```

```tsx
// src/modules/lab/RectifyForm.tsx
// Retificação (spec §5.9; RDC 978 art. 170): laudo novo com o motivo e quem
// foi avisado (pessoa, canal, hora); o anterior fica `rectified` e, no 21a,
// "substituído". Mesma assinatura da liberação; o código é apagado antes do
// envio. Os valores que entram são os escolhidos agora (digite o novo valor
// e escolha-o antes).
import { useState } from "react";
import { errorCode, rectifyLabReport, type LabReleaseItem, type LabReportSummary, type LabSignatureKind } from "../../lib/api";
import { CHANNEL_OPTIONS, contactsBody, labError, onlyIcp, rectifyProblem, totpProblem, type ContactDraft } from "../../lib/lab";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { SignatureChoice } from "./SignatureChoice";
import { cityNowLocal } from "./CriticalNoticeForm";
import { alert, box, checkLabel, fieldLabel, fieldset, legend, muted, row } from "../exams/styles";

export function RectifyForm({ report, items, council, onDone, onConflict, onCancel }: {
  report: LabReportSummary; items: LabReleaseItem[]; council: string | null;
  onDone(r: LabReportSummary): void; onConflict(text: string): void; onCancel(): void;
}) {
  const released = items.filter((i) => i.status === "released");
  const [ itemIds, setItemIds ] = useState<string[]>(released.map((i) => i.id));
  const [ reason, setReason ] = useState("");
  const [ contacts, setContacts ] = useState<ContactDraft[]>([]);
  const [ kind, setKind ] = useState<LabSignatureKind>("icp");
  const [ totp, setTotp ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const signature: LabSignatureKind = onlyIcp(council) ? "icp" : kind;

  const setContact = (i: number, patch: Partial<ContactDraft>) =>
    setContacts((list) => list.map((c, j) => (j === i ? { ...c, ...patch } : c)));

  async function submit() {
    const problem = rectifyProblem(reason, contacts, new Date().toISOString())
      ?? (itemIds.length === 0 ? "escolha ao menos um exame" : null)
      ?? (signature === "advanced" ? totpProblem(totp) : null);
    if (problem) { setError(problem); return; }
    setError(null);
    const code = totp;
    setTotp("");
    try {
      const body = { item_ids: itemIds, reason: reason.trim(), contacts: contactsBody(contacts), signature };
      onDone(await rectifyLabReport(report.id, signature === "advanced" ? { ...body, totp: code } : body));
    } catch (err) {
      if (errorCode(err) === "invalid_transition") { onConflict(labError(err)); return; }
      if (errorCode(err) === "icp_required") setKind("icp");
      setError(labError(err));
    }
  }

  return (
    <section aria-label={`Retificar laudo v${report.version}`} style={box}>
      <p style={muted}>O laudo atual fica marcado como substituído; o exame volta para a ciência de quem pediu.</p>
      <fieldset style={fieldset}>
        <legend style={legend}>Exames do laudo novo</legend>
        {released.map((i) => (
          <label key={i.id} style={checkLabel}>
            <input type="checkbox" checked={itemIds.includes(i.id)}
              onChange={(e) => setItemIds((l) => (e.target.checked ? [ ...l, i.id ] : l.filter((x) => x !== i.id)))} />
            {i.label}
          </label>
        ))}
      </fieldset>
      <label style={fieldLabel}>
        Motivo da retificação
        <textarea value={reason} rows={3} style={inputStyle} onChange={(e) => setReason(e.target.value)} />
      </label>
      {contacts.map((c, i) => (
        <fieldset key={i} role="group" aria-label={`Contato ${i + 1}`} style={fieldset}>
          <legend style={legend}>{`Contato ${i + 1}`}</legend>
          <label style={fieldLabel}>Quem foi avisado<input value={c.person} style={inputStyle} onChange={(e) => setContact(i, { person: e.target.value })} /></label>
          <label style={fieldLabel}>
            Como
            <select value={c.channel} style={inputStyle} onChange={(e) => setContact(i, { channel: e.target.value as ContactDraft["channel"] })}>
              <option value="">—</option>
              {CHANNEL_OPTIONS.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>Quando<input type="datetime-local" value={c.at} style={inputStyle} onChange={(e) => setContact(i, { at: e.target.value })} /></label>
          <div><button type="button" style={secondaryButtonStyle} onClick={() => setContacts((l) => l.filter((_, j) => j !== i))}>Tirar contato</button></div>
        </fieldset>
      ))}
      <div>
        <button type="button" style={secondaryButtonStyle}
          onClick={() => setContacts((l) => [ ...l, { person: "", channel: "", at: cityNowLocal() } ])}>Incluir contato</button>
      </div>
      <SignatureChoice council={council} kind={signature} onKind={setKind} totp={totp} onTotp={setTotp} />
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton onClick={submit}>Emitir laudo retificado</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}
```

Em `src/modules/lab/ReleaseReview.tsx`, acrescente os laudos e a retificação:

```tsx
// imports
import { LabReportList } from "./LabReportList";
import { RectifyForm } from "./RectifyForm";
```

```tsx
// estado, junto dos outros useState (antes dos returns antecipados):
  const [ rectifying, setRectifying ] = useState<LabReportSummary | null>(null);
```

```tsx
// no JSX, logo depois de {r.items.map(itemBlock)}:
      {r.reports.length > 0 && (
        <LabReportList reports={r.reports} onRectify={setRectifying} onExpired={() => void query.refetch()} />
      )}
      {rectifying && (
        <RectifyForm report={rectifying} items={r.items} council={council} onCancel={() => setRectifying(null)}
          onConflict={onConflict} onDone={(rep) => { setRectifying(null); onReleased(rep); }} />
      )}
```

e esconda a assinatura e o botão de liberar quando não há item `typed` (requisição já liberada aberta para retificar):

```tsx
      {releasable.length > 0 && <SignatureChoice council={council} kind={signature} onKind={setKind} totp={totp} onTotp={setTotp} />}
```

```tsx
        {releasable.length > 0 && (
          <ActionButton onClick={() => release(false)}>
            {partial ? `Liberar laudo parcial (${chosenIds.length} de ${r.items.length})` : "Liberar laudo"}
          </ActionButton>
        )}
```

Em `src/modules/LabRelease.tsx`, o painel "Laudo já liberado" (busca pelo número e abre a mesma conferência, que mostra os laudos):

```tsx
// imports: acrescente listLabRequisitions ao import de ../lib/api, e inputStyle ao de formStyles; fieldLabel ao de ./exams/styles
```

```tsx
// em ReleaseBody, estado:
  const [ number, setNumber ] = useState("");
  const [ searchError, setSearchError ] = useState<string | null>(null);

  async function findReleased() {
    const text = number.trim();
    if (!text) { setSearchError("informe o número da requisição"); return; }
    setSearchError(null);
    try {
      const page = await listLabRequisitions({ providerId, number: text, before: null });
      const found = page.requisitions[0];
      if (!found) { setSearchError("nenhuma requisição com este número neste laboratório"); return; }
      setMessage(null);
      setOpenId(found.id);
    } catch (err) {
      setSearchError(labError(err));
    }
  }
```

```tsx
// JSX, depois do painel "Fila de liberação":
      <Panel title="Laudo já liberado" sub="abrir os laudos de uma requisição para ver ou retificar">
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>
            Número da requisição
            <input value={number} style={inputStyle} onChange={(e) => setNumber(e.target.value)}
              onKeyDown={(e) => { if (e.key === "Enter") { e.preventDefault(); void findReleased(); } }} />
          </label>
          <button type="button" style={secondaryButtonStyle} onClick={() => void findReleased()}>Abrir</button>
        </div>
        {searchError && <p role="alert" style={alert}>{searchError}</p>}
      </Panel>
```

e, em `released()`, use o texto da retificação quando o laudo substitui outro:

```tsx
  function released(rep: LabReportSummary) {
    setOpenId(null);
    const what = rep.replaces_report_id ? "Laudo retificado" : "Laudo";
    setMessage(rep.status === "pending_signature"
      ? { kind: "ok", text: `${what} enviado para assinatura ICP — assine em Pendentes de assinatura.`, pending: true }
      : { kind: "ok", text: `${what} liberado com assinatura eletrônica avançada.` });
    void after();
  }
```

> O texto "Laudo liberado com assinatura eletrônica avançada." do teste da Task 12 continua igual (laudo sem `replaces_report_id`).

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/modules/lab/RectifyForm.test.tsx src/modules/LabRelease.test.tsx && npx tsc --noEmit`
Expected: PASS (a Task 12 segue verde).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/lab/LabReportList.tsx src/modules/lab/RectifyForm.tsx src/modules/lab/RectifyForm.test.tsx src/modules/lab/ReleaseReview.tsx src/modules/LabRelease.tsx
$G commit -m "feat: add lab report list and rectification with contacts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Laudo ICP nas Pendentes de assinatura

**Files:**
- Modify: `src/lib/api.ts` (`SignatureDocumentType`, `SignatureRequest.consultation_id`), `src/lib/signature.ts`, `src/modules/SignaturePending.tsx`
- Test: `src/lib/signature.lab.test.ts`, `src/modules/SignaturePending.lab.test.tsx`

**Interfaces:**
- Consumes: `canSign`, `documentLabel` (19b); `signer`, `pendingRequest` (`src/test/signatureFixtures.ts`); `labUser` (Task 1).
- Produces: `SignatureDocumentType` ganha `"lab_report"`; `SignatureRequest.consultation_id: string | null`; `canSign(user)` vale também para `lab_releaser` (com `digital_signature`); `documentLabel("lab_report") === "laudo laboratorial"`; a linha de laudo laboratorial não oferece "Voltar ao papel" (Divergência D19).

- [ ] **Step 1: Write the failing tests**

```ts
// src/lib/signature.lab.test.ts
import { describe, expect, it } from "vitest";
import { canSign, documentLabel } from "./signature";
import { labUser } from "../test/labFixtures";

describe("assinatura ICP do laudo laboratorial (módulo 21b)", () => {
  it("o liberador assina com digital_signature ligado; sem ele, não", () => {
    expect(canSign(labUser([ "lab_releaser" ], { features: [ "exam_tracking", "municipal_lab", "digital_signature" ] }))).toBe(true);
    expect(canSign(labUser([ "lab_releaser" ]))).toBe(false);
    expect(canSign(labUser([ "lab_staff" ], { features: [ "digital_signature" ] }))).toBe(false);
  });

  it("rótulo do documento", () => {
    expect(documentLabel("lab_report")).toBe("laudo laboratorial");
  });
});
```

```tsx
// src/modules/SignaturePending.lab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listPendingSignatures: vi.fn(), startSignatureBatch: vi.fn(), returnToPaper: vi.fn() };
});

import * as api from "../lib/api";
import { SignaturePending } from "./SignaturePending";
import { renderWithProviders } from "../test/campaignFixtures";
import { pendingRequest, signer } from "../test/signatureFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(cleanup);

beforeEach(() => {
  vi.clearAllMocks();
  mocked(api.fetchCurrentSession).mockResolvedValue(signer());
  mocked(api.listPendingSignatures).mockResolvedValue([
    pendingRequest(),
    pendingRequest({ id: "sr9", document_type: "lab_report", document_id: "lr3", consultation_id: null, patient_display_name: "Joana Lima" })
  ]);
});

describe("SignaturePending — laudo laboratorial", () => {
  it("aparece como laudo laboratorial e não volta ao papel", async () => {
    renderWithProviders(<SignaturePending onNavigate={vi.fn()} redirect={vi.fn()} />);
    const lab = (await screen.findByText("laudo laboratorial")).closest("[role='row']") as HTMLElement;
    expect(within(lab).queryByRole("button", { name: "Voltar ao papel" })).toBeNull();
    expect(within(lab).getByText("assine pelo lote")).not.toBeNull();
    const consult = (screen.getAllByText("consulta")[0]).closest("[role='row']") as HTMLElement;
    expect(within(consult).getByRole("button", { name: "Voltar ao papel" })).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/lib/signature.lab.test.ts src/modules/SignaturePending.lab.test.tsx`
Expected: FAIL — `canSign` falso para o liberador, rótulo cru e botão "Voltar ao papel" no laudo.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`:

```ts
export type SignatureDocumentType = "consultation" | "consultation_addendum" | "clinical_document" | "lab_report";
```

```ts
// interface SignatureRequest — troque a linha do consultation_id:
  id: string; document_type: SignatureDocumentType; document_id: string; consultation_id: string | null;
```

(Confira com `npx tsc --noEmit` que nenhum uso de `SignatureRequest.consultation_id` exige `string`; em `origin/main` não há uso fora do tipo.)

Em `src/lib/signature.ts`:

```ts
// Módulo 19b; 21b: o liberador do laboratório também assina (laudo ICP).
export function canSign(user: SessionLike): boolean {
  const roles = rolesOf(user);
  return !!user && !user.operator && (roles.includes("health_professional") || roles.includes("lab_releaser")) && hasFeature(user, "digital_signature");
}
```

```ts
const DOCUMENT_LABEL: Record<string, string> = {
  consultation: "consulta", consultation_addendum: "adendo", clinical_document: "documento clínico", lab_report: "laudo laboratorial"
};
```

Em `src/modules/SignaturePending.tsx`, na coluna de ação:

```tsx
    {
      label: "", w: "1.2fr", render: (r) => (r.document_type === "lab_report"
        // O laudo do laboratório não volta ao papel (RDC 978: laudo assinado); se o ICP falhar,
        // o liberador assina de novo pelo lote (Divergência D19).
        ? <span style={{ fontSize: 12, color: "var(--ink3)" }}>assine pelo lote</span>
        : <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setPaperFor(r); }}>Voltar ao papel</button>)
    }
```

e troque o `sub` do `Panel` por `"consultas, adendos e laudos que esperam a sua assinatura digital"`. Se o `SignaturePending.test.tsx` antigo conferir esse texto, atualize a expectativa para o novo.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/lib/signature.lab.test.ts src/modules/SignaturePending.lab.test.tsx src/modules/SignaturePending.test.tsx src/lib/signature.test.ts src/shell && npx tsc --noEmit`
Expected: PASS (o menu do liberador com `digital_signature` passa a ter "Assinatura digital" e "Pendentes de assinatura"; nenhum teste antigo dá papel `lab_releaser`).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/signature.ts src/lib/signature.lab.test.ts src/modules/SignaturePending.tsx src/modules/SignaturePending.lab.test.tsx
$G commit -m "feat: let lab releasers sign lab reports with ICP in pending signatures

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Inclua `src/modules/SignaturePending.test.tsx` no `git add` só se precisou mudar a expectativa do subtítulo.)

---

### Task 15: Admin do laboratório — adotar exame do catálogo-base e criar versões

**Files:**
- Create: `src/lib/labAdmin.ts`, `src/modules/LabAdmin.tsx`, `src/modules/labAdmin/LabExamsPanel.tsx`, `src/modules/labAdmin/VersionForm.tsx`
- Modify: `src/App.tsx`
- Test: `src/lib/labAdmin.test.ts`, `src/modules/labAdmin/VersionForm.test.tsx`

**Interfaces:**
- Consumes: `listLabTemplates`, `listLabAdminExams`, `getLabAdminExam`, `adoptLabExam`, `updateLabExam`, `createLabExamVersion`, `listExamProviders`, `LabTemplate`, `LabExam`, `LabExamVersion`, `LabAnalyteVersion`, `LabReferenceRange`, `LabVersionInput`, `LabExamAdoptInput` (Task 1); `SECTOR_LABEL`, `sectorLabel`, `labError`, `LAB_TEMPLATES_KEY`, `LAB_ADMIN_EXAMS_KEY` (Task 2); `LabGate` (Task 4); `SensitiveAction`; `EXAM_ADMIN_PROVIDERS_KEY` (21a).
- Produces (em `src/lib/labAdmin.ts`): `interface RangeDraft`, `interface AnalyteDraft`, `interface VersionDraft`, `ageToDays(value: string, unit: "days" | "months" | "years"): number | null`, `daysToAge(days: number | null): { value: string; unit: "days" | "months" | "years" }`, `versionDraft(v: LabExamVersion | null, t?: LabTemplate): VersionDraft`, `versionProblem(d: VersionDraft): string | null`, `versionBody(d: VersionDraft): LabVersionInput`, `formulaKeys(formula: string): string[]`, `labProviders(list: ExamProvider[]): ExamProvider[]` (só `uses_municipal_lab`).
- Produces (telas): `VersionForm({ exam, onSaved, onCancel, onGoToSecurity })`; `LabExamsPanel({ providerId, onGoToSecurity })`; `LabAdmin({ onGoToSecurity })` em `renderModule` como `case "lab-admin"`. A Task 16 acrescenta os outros painéis ao `LabAdmin`.

- [ ] **Step 1: Write the failing tests**

```ts
// src/lib/labAdmin.test.ts
import { describe, expect, it } from "vitest";
import { ageToDays, daysToAge, formulaKeys, labProviders, versionBody, versionDraft, versionProblem } from "./labAdmin";
import { labVersion } from "../test/labFixtures";
import { examProvider } from "../test/examFixtures";

describe("idade das faixas", () => {
  it("anos, meses e dias viram dias; e voltam na unidade mais redonda", () => {
    expect(ageToDays("18", "years")).toBe(6570);
    expect(ageToDays("6", "months")).toBe(180);
    expect(ageToDays("", "years")).toBeNull();
    expect(ageToDays("-1", "days")).toBeNull();
    expect(daysToAge(6570)).toEqual({ value: "18", unit: "years" });
    expect(daysToAge(180)).toEqual({ value: "6", unit: "months" });
    expect(daysToAge(10)).toEqual({ value: "10", unit: "days" });
    expect(daysToAge(null)).toEqual({ value: "", unit: "years" });
  });
});

describe("versão", () => {
  it("parte da versão atual e devolve o mesmo corpo", () => {
    const v = labVersion();
    const body = versionBody(versionDraft(v));
    expect(body).toEqual({
      method: v.method, material: v.material, container: v.container, instrument_test_code: "GLU",
      analytes: [ { key: "glicose", name: "Glicose", loinc: "2345-7", unit: "mg/dL", value_type: "numeric", choices: null, decimals: 0,
        reference_ranges: [ { sex: "any", age_min_days: 6570, age_max_days: null, low: 70, high: 99, text: null } ],
        entry_min: 10, entry_max: 1500, critical_low: 40, critical_high: 400, formula: null } ]
    });
  });

  it("recusas: chave, faixa sem limite nem texto, idade invertida, crítico dentro da referência, fórmula com chave desconhecida", () => {
    const base = versionDraft(labVersion());
    const a = base.analytes[0];
    const with1 = (patch: object) => ({ ...base, analytes: [ { ...a, ...patch } ] });
    const range = a.ranges[0];
    expect(versionProblem(with1({ key: "Glicose!" }))).toBe("componente 1: a chave usa letras minúsculas, números e _");
    expect(versionProblem(with1({ ranges: [ { ...range, low: "", high: "", text: "" } ] })))
      .toBe("componente 1, faixa 1: informe mínimo, máximo ou um texto");
    expect(versionProblem(with1({ ranges: [ { ...range, ageMin: "60", ageMinUnit: "years", ageMax: "18", ageMaxUnit: "years" } ] })))
      .toBe("componente 1, faixa 1: a idade mínima passa da máxima");
    expect(versionProblem(with1({ entryMin: "200", entryMax: "100" }))).toBe("componente 1: o limite de digitação mínimo passa do máximo");
    expect(versionProblem({ ...base, analytes: [ a, { ...a, key: "calc", name: "Calculado", formula: "glicose * k" } ] }))
      .toBe("componente 2: a fórmula usa k, que não é componente deste exame");
    expect(versionProblem({ ...base, method: " " })).toBe("informe o método");
    expect(versionProblem(base)).toBeNull();
  });

  it("chaves da fórmula", () => {
    expect(formulaKeys("ct - hdl - tg / 5")).toEqual([ "ct", "hdl", "tg" ]);
    expect(formulaKeys("(a+b)*2.5")).toEqual([ "a", "b" ]);
  });

  it("só prestadores com o laboratório municipal", () => {
    expect(labProviders([ examProvider(), examProvider({ id: "pv9", uses_municipal_lab: true }) ]).map((p) => p.id)).toEqual([ "pv9" ]);
  });
});
```

```tsx
// src/modules/labAdmin/VersionForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), createLabExamVersion: vi.fn() };
});

import * as api from "../../lib/api";
import { VersionForm } from "./VersionForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { labExam, labUser, labVersion } from "../../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(labUser([ "municipal_admin" ]));
  mocked(api.createLabExamVersion).mockReset().mockResolvedValue(labVersion({ id: "ev2", version: 2 }));
});

describe("VersionForm", () => {
  it("nova versão: faixa por sexo e idade, com step-up; nunca edita a atual", async () => {
    const onSaved = vi.fn();
    renderWithProviders(<VersionForm exam={labExam()} onSaved={onSaved} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Nova versão de Glicose (atual v1)" });
    const analyte = within(form).getByRole("group", { name: "Componente 1" });
    fireEvent.click(within(analyte).getByRole("button", { name: "Incluir faixa" }));
    const range = within(analyte).getByRole("group", { name: "Faixa 2" });
    fireEvent.change(within(range).getByLabelText("Sexo"), { target: { value: "female" } });
    fireEvent.change(within(range).getByLabelText("Idade mínima"), { target: { value: "60" } });
    fireEvent.change(within(range).getByLabelText("Mínimo"), { target: { value: "70" } });
    fireEvent.change(within(range).getByLabelText("Máximo"), { target: { value: "110" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar versão" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar versão 2" }));
    await waitFor(() => expect(api.createLabExamVersion).toHaveBeenCalled());
    const [ id, body ] = mocked(api.createLabExamVersion).mock.calls[0];
    expect(id).toBe("le1");
    expect(body.analytes[0].reference_ranges[1]).toEqual({ sex: "female", age_min_days: 21900, age_max_days: null, low: 70, high: 110, text: null });
    expect(onSaved).toHaveBeenCalled();
  });

  it("problema no formulário não abre a confirmação", async () => {
    renderWithProviders(<VersionForm exam={labExam()} onSaved={vi.fn()} onCancel={vi.fn()} />);
    const form = await screen.findByRole("region", { name: "Nova versão de Glicose (atual v1)" });
    fireEvent.change(within(form).getByLabelText("Método"), { target: { value: "" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar versão" }));
    expect(within(form).getByRole("alert").textContent).toBe("informe o método");
    expect(screen.queryByRole("button", { name: "Criar versão 2" })).toBeNull();
  });

  it("422 do api vira frase (ex.: código do aparelho repetido)", async () => {
    mocked(api.createLabExamVersion).mockRejectedValueOnce(new api.ApiError(422, { error: "duplicate_instrument_code" }, "x"));
    renderWithProviders(<VersionForm exam={labExam()} onSaved={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Revisar e salvar versão" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar versão 2" }));
    expect(await screen.findByText("este código do aparelho já está em outro exame do laboratório")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/lib/labAdmin.test.ts src/modules/labAdmin/VersionForm.test.tsx`
Expected: FAIL — módulos inexistentes.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/labAdmin.ts
// Formulário da versão do exame do laboratório (contrato §3): componentes,
// faixas de referência por sexo e idade, limites de digitação, críticos e
// fórmula. Cada salvamento cria uma versão nova e imutável no api; aqui só
// se monta e confere o corpo. Idade em anos, meses ou dias na tela; em dias
// no corpo (1 mês = 30 dias, 1 ano = 365 dias, como o catálogo-base).
import type { ExamProvider, LabExamVersion, LabReferenceRange, LabTemplate, LabValueType, LabVersionInput } from "./api";

export type AgeUnit = "days" | "months" | "years";
const UNIT_DAYS: Record<AgeUnit, number> = { days: 1, months: 30, years: 365 };

export interface RangeDraft {
  sex: "male" | "female" | "any"; ageMin: string; ageMinUnit: AgeUnit; ageMax: string; ageMaxUnit: AgeUnit;
  low: string; high: string; text: string;
}
export interface AnalyteDraft {
  key: string; name: string; loinc: string; unit: string; valueType: LabValueType; choices: string; decimals: string;
  ranges: RangeDraft[]; entryMin: string; entryMax: string; criticalLow: string; criticalHigh: string; formula: string;
}
export interface VersionDraft { method: string; material: string; container: string; instrumentTestCode: string; analytes: AnalyteDraft[] }

export const EMPTY_RANGE: RangeDraft = { sex: "any", ageMin: "", ageMinUnit: "years", ageMax: "", ageMaxUnit: "years", low: "", high: "", text: "" };

export function ageToDays(value: string, unit: AgeUnit): number | null {
  const t = value.trim();
  if (!/^\d+$/.test(t)) return null;
  return Number(t) * UNIT_DAYS[unit];
}

export function daysToAge(days: number | null): { value: string; unit: AgeUnit } {
  if (days === null) return { value: "", unit: "years" };
  if (days % 365 === 0 && days > 0) return { value: String(days / 365), unit: "years" };
  if (days % 30 === 0 && days > 0) return { value: String(days / 30), unit: "months" };
  return { value: String(days), unit: "days" };
}

const num = (v: number | null | undefined) => (v === null || v === undefined ? "" : String(v));
const parse = (v: string): number | null => {
  const t = v.trim().replace(",", ".");
  return t === "" || !/^-?\d+(\.\d+)?$/.test(t) ? null : Number(t);
};

function rangeDraft(r: LabReferenceRange): RangeDraft {
  const min = daysToAge(r.age_min_days);
  const max = daysToAge(r.age_max_days);
  return { sex: r.sex, ageMin: min.value, ageMinUnit: min.unit, ageMax: max.value, ageMaxUnit: max.unit, low: num(r.low), high: num(r.high), text: r.text ?? "" };
}

export function versionDraft(v: LabExamVersion | null, t?: LabTemplate): VersionDraft {
  const analytes = v?.analytes ?? (t?.analytes ?? []).map((a) => ({ ...a, entry_min: null, entry_max: null, critical_low: null, critical_high: null, formula: null }));
  return {
    method: v?.method ?? t?.suggested_method ?? "", material: v?.material ?? t?.material ?? "", container: v?.container ?? t?.container ?? "",
    instrumentTestCode: v?.instrument_test_code ?? "",
    analytes: analytes.map((a) => ({
      key: a.key, name: a.name, loinc: a.loinc ?? "", unit: a.unit ?? "", valueType: a.value_type, choices: (a.choices ?? []).join("; "),
      decimals: String(a.decimals), ranges: a.reference_ranges.map(rangeDraft), entryMin: num(a.entry_min), entryMax: num(a.entry_max),
      criticalLow: num(a.critical_low), criticalHigh: num(a.critical_high), formula: a.formula ?? ""
    }))
  };
}

export function formulaKeys(formula: string): string[] {
  return [ ...new Set(formula.match(/[a-z][a-z0-9_]*/g) ?? []) ];
}

const badNumber = (v: string) => v.trim() !== "" && parse(v) === null;

export function versionProblem(d: VersionDraft): string | null {
  if (!d.method.trim()) return "informe o método";
  if (!d.material.trim()) return "informe o material";
  if (!d.container.trim()) return "informe o recipiente";
  if (d.analytes.length === 0) return "inclua ao menos um componente";
  const keys = new Set<string>();
  for (const [ i, a ] of d.analytes.entries()) {
    const n = `componente ${i + 1}`;
    if (!/^[a-z][a-z0-9_]{0,39}$/.test(a.key.trim())) return `${n}: a chave usa letras minúsculas, números e _`;
    if (keys.has(a.key.trim())) return `${n}: a chave ${a.key.trim()} repete`;
    keys.add(a.key.trim());
    if (!a.name.trim()) return `${n}: informe o nome`;
    if (a.loinc.trim() && !/^\d{1,7}-\d$/.test(a.loinc.trim())) return `${n}: LOINC no formato 2345-7`;
    if (!/^\d$/.test(a.decimals.trim())) return `${n}: casas decimais de 0 a 9`;
    if (a.valueType === "choice" && a.choices.split(";").map((c) => c.trim()).filter(Boolean).length < 2) return `${n}: escreva ao menos duas opções separadas por ;`;
    for (const v of [ a.entryMin, a.entryMax, a.criticalLow, a.criticalHigh ]) if (badNumber(v)) return `${n}: número inválido nos limites`;
    const [ emin, emax ] = [ parse(a.entryMin), parse(a.entryMax) ];
    if (emin !== null && emax !== null && emin > emax) return `${n}: o limite de digitação mínimo passa do máximo`;
    for (const [ j, r ] of a.ranges.entries()) {
      const rn = `${n}, faixa ${j + 1}`;
      if (badNumber(r.low) || badNumber(r.high)) return `${rn}: número inválido`;
      if (r.low.trim() === "" && r.high.trim() === "" && !r.text.trim()) return `${rn}: informe mínimo, máximo ou um texto`;
      const [ lo, hi ] = [ parse(r.low), parse(r.high) ];
      if (lo !== null && hi !== null && lo > hi) return `${rn}: o mínimo passa do máximo`;
      const [ amin, amax ] = [ ageToDays(r.ageMin, r.ageMinUnit), ageToDays(r.ageMax, r.ageMaxUnit) ];
      if ((r.ageMin.trim() && amin === null) || (r.ageMax.trim() && amax === null)) return `${rn}: idade em número inteiro`;
      if (amin !== null && amax !== null && amin > amax) return `${rn}: a idade mínima passa da máxima`;
    }
  }
  for (const [ i, a ] of d.analytes.entries()) {
    if (!a.formula.trim()) continue;
    if (!/^[a-z0-9_+\-*/().\s]+$/.test(a.formula)) return `componente ${i + 1}: a fórmula usa só chaves, números e + - * / ( )`;
    const unknown = formulaKeys(a.formula).find((k) => !keys.has(k) || k === a.key.trim());
    if (unknown) return `componente ${i + 1}: a fórmula usa ${unknown}, que não é componente deste exame`;
  }
  return null;
}

export function versionBody(d: VersionDraft): LabVersionInput {
  return {
    method: d.method.trim(), material: d.material.trim(), container: d.container.trim(),
    instrument_test_code: d.instrumentTestCode.trim() || null,
    analytes: d.analytes.map((a) => ({
      key: a.key.trim(), name: a.name.trim(), loinc: a.loinc.trim() || null, unit: a.unit.trim() || null, value_type: a.valueType,
      choices: a.valueType === "choice" ? a.choices.split(";").map((c) => c.trim()).filter(Boolean) : null,
      decimals: Number(a.decimals.trim()),
      reference_ranges: a.ranges.map((r) => ({
        sex: r.sex, age_min_days: ageToDays(r.ageMin, r.ageMinUnit), age_max_days: ageToDays(r.ageMax, r.ageMaxUnit),
        low: parse(r.low), high: parse(r.high), text: r.text.trim() || null
      })),
      entry_min: parse(a.entryMin), entry_max: parse(a.entryMax), critical_low: parse(a.criticalLow), critical_high: parse(a.criticalHigh),
      formula: a.formula.trim() || null
    }))
  };
}

export function labProviders(list: ExamProvider[]): ExamProvider[] {
  return list.filter((p) => p.uses_municipal_lab === true);
}
```

> Na fixture, a faixa de 18 anos é `age_min_days: 6570` (18 × 365) e o rascunho a mostra como "18 anos"; 60 anos viram `21900`. O teste do componente confere isso.

```tsx
// src/modules/labAdmin/VersionForm.tsx
// Nova versão do exame (spec §4; contrato §3): parte da versão atual; cada
// salvamento cria versão nova e imutável (o laudo guarda a versão). Step-up
// pelo SensitiveAction.
import { useState } from "react";
import { createLabExamVersion, type LabExam } from "../../lib/api";
import { EMPTY_RANGE, versionBody, versionDraft, versionProblem, type AgeUnit, type AnalyteDraft, type RangeDraft, type VersionDraft } from "../../lib/labAdmin";
import { labError } from "../../lib/lab";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, fieldset, form, legend, muted, row } from "../exams/styles";

const AGE_UNITS: { value: AgeUnit; label: string }[] = [ { value: "years", label: "anos" }, { value: "months", label: "meses" }, { value: "days", label: "dias" } ];
const SEX: { value: RangeDraft["sex"]; label: string }[] = [ { value: "any", label: "qualquer" }, { value: "female", label: "feminino" }, { value: "male", label: "masculino" } ];

export function VersionForm({ exam, onSaved, onCancel, onGoToSecurity }: { exam: LabExam; onSaved(): void; onCancel(): void; onGoToSecurity?(): void }) {
  const [ draft, setDraft ] = useState<VersionDraft>(() => versionDraft(exam.current_version));
  const [ error, setError ] = useState<string | null>(null);
  const [ confirming, setConfirming ] = useState(false);
  const next = (exam.current_version?.version ?? 0) + 1;
  const set = (patch: Partial<VersionDraft>) => setDraft((d) => ({ ...d, ...patch }));
  const setA = (i: number, patch: Partial<AnalyteDraft>) => set({ analytes: draft.analytes.map((a, j) => (j === i ? { ...a, ...patch } : a)) });
  const setR = (i: number, k: number, patch: Partial<RangeDraft>) =>
    setA(i, { ranges: draft.analytes[i].ranges.map((r, j) => (j === k ? { ...r, ...patch } : r)) });

  function review() {
    const problem = versionProblem(draft);
    if (problem) { setError(problem); return; }
    setError(null);
    setConfirming(true);
  }

  const text = (label: string, value: string, onChange: (v: string) => void, numeric = false) => (
    <label style={fieldLabel}>{label}<input value={value} inputMode={numeric ? "decimal" : "text"} style={inputStyle} onChange={(e) => onChange(e.target.value)} /></label>
  );

  return (
    <section aria-label={`Nova versão de ${exam.name} (atual v${exam.current_version?.version ?? 0})`} style={box}>
      <p style={muted}>A versão atual não muda: salvar cria a versão {next}, que passa a valer para as próximas requisições.</p>
      <div style={form}>
        {text("Método", draft.method, (v) => set({ method: v }))}
        {text("Material", draft.material, (v) => set({ material: v }))}
        {text("Recipiente", draft.container, (v) => set({ container: v }))}
        {text("Código do exame no aparelho (HL7)", draft.instrumentTestCode, (v) => set({ instrumentTestCode: v }))}
        {draft.analytes.map((a, i) => (
          <fieldset key={i} role="group" aria-label={`Componente ${i + 1}`} style={fieldset}>
            <legend style={legend}>{`Componente ${i + 1}${a.name ? ` — ${a.name}` : ""}`}</legend>
            <div style={row}>
              {text("Chave", a.key, (v) => setA(i, { key: v }))}
              {text("Nome", a.name, (v) => setA(i, { name: v }))}
              {text("LOINC", a.loinc, (v) => setA(i, { loinc: v }))}
              {text("Unidade", a.unit, (v) => setA(i, { unit: v }))}
              {text("Casas decimais", a.decimals, (v) => setA(i, { decimals: v }), true)}
            </div>
            <div style={row}>
              {text("Limite de digitação mínimo", a.entryMin, (v) => setA(i, { entryMin: v }), true)}
              {text("Limite de digitação máximo", a.entryMax, (v) => setA(i, { entryMax: v }), true)}
              {text("Crítico abaixo de", a.criticalLow, (v) => setA(i, { criticalLow: v }), true)}
              {text("Crítico acima de", a.criticalHigh, (v) => setA(i, { criticalHigh: v }), true)}
            </div>
            {text("Fórmula (calculado; deixe em branco se digitado)", a.formula, (v) => setA(i, { formula: v }))}
            {a.ranges.map((r, k) => (
              <fieldset key={k} role="group" aria-label={`Faixa ${k + 1}`} style={fieldset}>
                <legend style={legend}>{`Faixa ${k + 1}`}</legend>
                <div style={row}>
                  <label style={fieldLabel}>Sexo
                    <select value={r.sex} style={inputStyle} onChange={(e) => setR(i, k, { sex: e.target.value as RangeDraft["sex"] })}>
                      {SEX.map((s) => <option key={s.value} value={s.value}>{s.label}</option>)}
                    </select>
                  </label>
                  {text("Idade mínima", r.ageMin, (v) => setR(i, k, { ageMin: v }), true)}
                  <label style={fieldLabel}>Unidade da idade mínima
                    <select value={r.ageMinUnit} style={inputStyle} onChange={(e) => setR(i, k, { ageMinUnit: e.target.value as AgeUnit })}>
                      {AGE_UNITS.map((u) => <option key={u.value} value={u.value}>{u.label}</option>)}
                    </select>
                  </label>
                  {text("Idade máxima", r.ageMax, (v) => setR(i, k, { ageMax: v }), true)}
                  <label style={fieldLabel}>Unidade da idade máxima
                    <select value={r.ageMaxUnit} style={inputStyle} onChange={(e) => setR(i, k, { ageMaxUnit: e.target.value as AgeUnit })}>
                      {AGE_UNITS.map((u) => <option key={u.value} value={u.value}>{u.label}</option>)}
                    </select>
                  </label>
                </div>
                <div style={row}>
                  {text("Mínimo", r.low, (v) => setR(i, k, { low: v }), true)}
                  {text("Máximo", r.high, (v) => setR(i, k, { high: v }), true)}
                  {text("Texto da referência", r.text, (v) => setR(i, k, { text: v }))}
                  <button type="button" style={secondaryButtonStyle} onClick={() => setA(i, { ranges: a.ranges.filter((_, j) => j !== k) })}>Tirar faixa</button>
                </div>
              </fieldset>
            ))}
            <div><button type="button" style={secondaryButtonStyle} onClick={() => setA(i, { ranges: [ ...a.ranges, EMPTY_RANGE ] })}>Incluir faixa</button></div>
          </fieldset>
        ))}
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      {!confirming && (
        <div style={row}>
          <button type="button" style={buttonStyle} onClick={review}>Revisar e salvar versão</button>
          <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
        </div>
      )}
      {confirming && (
        <SensitiveAction
          title={`Criar versão ${next} de ${exam.name}`}
          description="Os laudos já emitidos continuam com a versão deles; a nova vale para as próximas requisições."
          requiresStepUp confirmLabel={`Criar versão ${next}`}
          run={async () => { await createLabExamVersion(exam.id, versionBody(draft)); }}
          onDone={() => { setConfirming(false); onSaved(); }}
          onCancel={() => setConfirming(false)} onGoToSecurity={onGoToSecurity}
          translateError={(err) => labError(err)}
        />
      )}
    </section>
  );
}
```

> O `SensitiveAction` mostra o erro traduzido dentro dele (por `translateError`), por isso o teste do 422 acha a frase na tela sem `role` fixo. A sessão de teste tem a janela de step-up aberta (`mfa_verified_at` = agora), então o botão de confirmar roda direto.

```tsx
// src/modules/labAdmin/LabExamsPanel.tsx
// Exames do laboratório (contrato §3): adotar do catálogo-base (setor,
// bancada, prazo de entrega), ajustar e criar versões. Escrita com step-up.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { adoptLabExam, getLabAdminExam, listLabAdminExams, listLabTemplates, updateLabExam, type LabExam, type LabTemplate } from "../../lib/api";
import { LAB_ADMIN_EXAMS_KEY, LAB_TEMPLATES_KEY, SECTOR_LABEL, labError, sectorLabel } from "../../lib/lab";
import { fmtDateTime } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { SensitiveAction } from "../../components/SensitiveAction";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { VersionForm } from "./VersionForm";
import { alert, fieldLabel, list, muted, row, stack } from "../exams/styles";

interface Settings { sector: string; bench: string; deliveryDays: string; active: boolean }

export function LabExamsPanel({ providerId, onGoToSecurity }: { providerId: string; onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const exams = useQuery({ queryKey: [ LAB_ADMIN_EXAMS_KEY, providerId ], queryFn: () => listLabAdminExams(providerId), retry: false });
  const templates = useQuery({ queryKey: [ LAB_TEMPLATES_KEY ], queryFn: listLabTemplates, retry: false });
  const [ adopting, setAdopting ] = useState<LabTemplate | null>(null);
  const [ editing, setEditing ] = useState<LabExam | null>(null);
  const [ versioning, setVersioning ] = useState<LabExam | null>(null);
  const [ settings, setSettings ] = useState<Settings>({ sector: "", bench: "", deliveryDays: "2", active: true });
  const [ confirm, setConfirm ] = useState<null | "adopt" | "edit">(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const adopted = new Set((exams.data ?? []).map((e) => e.template_id));
  const refresh = () => queryClient.invalidateQueries({ queryKey: [ LAB_ADMIN_EXAMS_KEY ] });

  function check(): boolean {
    if (!settings.sector) { setError("escolha o setor"); return false; }
    if (!/^\d{1,3}$/.test(settings.deliveryDays.trim()) || Number(settings.deliveryDays) < 1) { setError("prazo de entrega em dias inteiros (1 a 999)"); return false; }
    setError(null);
    return true;
  }

  async function openVersion(e: LabExam) {
    setError(null);
    try { setVersioning(await getLabAdminExam(e.id)); } catch (err) { setError(labError(err)); }
  }

  const settingsFields = (
    <div style={row}>
      <label style={fieldLabel}>Setor
        <select value={settings.sector} style={inputStyle} onChange={(e) => setSettings((s) => ({ ...s, sector: e.target.value }))}>
          <option value="">—</option>
          {Object.entries(SECTOR_LABEL).map(([ k, v ]) => <option key={k} value={k}>{v}</option>)}
        </select>
      </label>
      <label style={fieldLabel}>Bancada<input value={settings.bench} style={inputStyle} onChange={(e) => setSettings((s) => ({ ...s, bench: e.target.value }))} /></label>
      <label style={fieldLabel}>Prazo de entrega (dias)<input value={settings.deliveryDays} inputMode="numeric" style={inputStyle} onChange={(e) => setSettings((s) => ({ ...s, deliveryDays: e.target.value }))} /></label>
    </div>
  );

  return (
    <Panel title="Exames do laboratório" sub="adotados do catálogo-base; cada ajuste de componente vira uma versão nova">
      <div style={stack}>
        {notice && <p role="status" style={muted}>{notice}</p>}
        {exams.isError && <p role="alert" style={alert}>{labError(exams.error)}</p>}
        <DataTable<LabExam>
          cols={[
            { label: "Exame", w: "1.6fr", card: "title", render: (e) => e.name },
            { label: "Setor", w: "1fr", render: (e) => `${sectorLabel(e.sector)}${e.bench ? ` · ${e.bench}` : ""}` },
            { label: "Versão", w: "0.6fr", render: (e) => (e.current_version ? `v${e.current_version.version}` : "—") },
            { label: "Prazo", w: "0.6fr", render: (e) => `${e.delivery_days} d` },
            { label: "Situação", w: "0.8fr", render: (e) => <Tag tone={e.active ? "ok" : undefined}>{e.active ? "ativo" : "inativo"}</Tag> },
            { label: "", w: "auto", card: "actions", render: (e) => (
              <div style={row}>
                <button type="button" style={secondaryButtonStyle} onClick={() => { setEditing(e); setSettings({ sector: e.sector, bench: e.bench ?? "", deliveryDays: String(e.delivery_days), active: e.active }); }}>Ajustar</button>
                <button type="button" style={secondaryButtonStyle} onClick={() => void openVersion(e)}>Nova versão</button>
              </div>
            ) }
          ]}
          rows={exams.data ?? []} rowKey={(e) => e.id} empty={exams.isPending ? "carregando…" : "nenhum exame adotado"}
        />
        {versioning && (
          <>
            <ul aria-label={`versões de ${versioning.name}`} style={list}>
              {(versioning.versions ?? []).map((v) => <li key={v.id}>{`v${v.version} · ${fmtDateTime(v.created_at)}${v.created_by ? ` · ${v.created_by}` : ""}`}</li>)}
            </ul>
            <VersionForm exam={versioning} onGoToSecurity={onGoToSecurity} onCancel={() => setVersioning(null)}
              onSaved={() => { setVersioning(null); setNotice("Versão nova criada."); void refresh(); }} />
          </>
        )}
        {editing && (
          <div style={stack}>
            <strong>{`Ajustar ${editing.name}`}</strong>
            {settingsFields}
            <label style={{ ...fieldLabel, flexDirection: "row", gap: 6 }}>
              <input type="checkbox" checked={settings.active} onChange={(e) => setSettings((s) => ({ ...s, active: e.target.checked }))} />Ativo
            </label>
            {confirm !== "edit" && <div style={row}>
              <button type="button" style={secondaryButtonStyle} onClick={() => { if (check()) setConfirm("edit"); }}>Salvar ajuste</button>
              <button type="button" style={secondaryButtonStyle} onClick={() => setEditing(null)}>Cancelar</button>
            </div>}
            {confirm === "edit" && (
              <SensitiveAction title={`Ajustar ${editing.name}`} description="Setor, bancada, prazo e situação. Componentes e referências mudam só por versão nova."
                requiresStepUp confirmLabel="Salvar"
                run={async () => { await updateLabExam(editing.id, { sector: settings.sector, bench: settings.bench.trim() || null, delivery_days: Number(settings.deliveryDays), active: settings.active }); }}
                onDone={() => { setConfirm(null); setEditing(null); void refresh(); }} onCancel={() => setConfirm(null)} onGoToSecurity={onGoToSecurity}
                translateError={(err) => labError(err)} />
            )}
          </div>
        )}
        <strong>Catálogo-base da plataforma</strong>
        {templates.isError && <p role="alert" style={alert}>{labError(templates.error)}</p>}
        <DataTable<LabTemplate>
          cols={[
            { label: "Exame", w: "1.6fr", card: "title", render: (t) => t.name },
            { label: "SIGTAP", w: "1fr", render: (t) => <span className="mono">{t.sigtap_code}</span> },
            { label: "Setor", w: "1fr", render: (t) => sectorLabel(t.sector) },
            { label: "Componentes", w: "0.8fr", align: "right", render: (t) => String(t.analytes.length) },
            { label: "", w: "auto", card: "actions", render: (t) => (adopted.has(t.id) ? <Tag tone="ok">adotado</Tag> : (
              <button type="button" style={secondaryButtonStyle} onClick={() => { setAdopting(t); setSettings({ sector: t.sector, bench: "", deliveryDays: "2", active: true }); }}>Adotar</button>
            )) }
          ]}
          rows={templates.data ?? []} rowKey={(t) => t.id} empty={templates.isPending ? "carregando…" : "catálogo-base vazio"}
        />
        {adopting && (
          <div style={stack}>
            <strong>{`Adotar ${adopting.name}`}</strong>
            <p style={muted}>A referência de partida é do catálogo-base: ajuste ao método do seu laboratório criando uma versão nova.</p>
            {settingsFields}
            {confirm !== "adopt" && <div style={row}>
              <button type="button" style={secondaryButtonStyle} onClick={() => { if (check()) setConfirm("adopt"); }}>Adotar exame</button>
              <button type="button" style={secondaryButtonStyle} onClick={() => setAdopting(null)}>Cancelar</button>
            </div>}
            {confirm === "adopt" && (
              <SensitiveAction title={`Adotar ${adopting.name}`} description="O exame entra no laboratório com a versão 1 copiada do catálogo-base."
                requiresStepUp confirmLabel="Adotar"
                run={async () => { await adoptLabExam({ provider_id: providerId, template_id: adopting.id, sector: settings.sector, bench: settings.bench.trim() || null, delivery_days: Number(settings.deliveryDays) }); }}
                onDone={() => { setConfirm(null); setAdopting(null); setNotice("Exame adotado."); void refresh(); }} onCancel={() => setConfirm(null)}
                onGoToSecurity={onGoToSecurity} translateError={(err) => labError(err)} />
            )}
          </div>
        )}
        {error && <p role="alert" style={alert}>{error}</p>}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/LabAdmin.tsx
// Admin do laboratório municipal (spec §7 "Admin"; contrato §3): por
// laboratório (prestador com `uses_municipal_lab`), exames e versões;
// a Task 16 acrescenta motivos de rejeição, liberadores e o token do middleware.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { listExamProviders } from "../lib/api";
import { EXAM_ADMIN_PROVIDERS_KEY } from "../lib/exams";
import { labError } from "../lib/lab";
import { labProviders } from "../lib/labAdmin";
import { EmptyState } from "../components/EmptyState";
import { inputStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { LabExamsPanel } from "./labAdmin/LabExamsPanel";
import { alert, fieldLabel, muted, stack } from "./exams/styles";

export function LabAdmin({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  return (
    <LabGate title="Exames e liberadores do laboratório" sub="cidade · laboratório municipal" allow={(r) => r.admin}>
      {() => <AdminBody onGoToSecurity={onGoToSecurity} />}
    </LabGate>
  );
}

function AdminBody({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const providers = useQuery({ queryKey: [ EXAM_ADMIN_PROVIDERS_KEY ], queryFn: listExamProviders, retry: false });
  const [ providerId, setProviderId ] = useState("");
  if (providers.isPending) return <p className="mono" style={muted}>carregando…</p>;
  if (providers.isError) return <p role="alert" style={alert}>{labError(providers.error)}</p>;
  const labs = labProviders(providers.data);
  if (labs.length === 0) {
    return <EmptyState title="nenhum prestador usa o laboratório municipal" sub="marque o laboratório em Exames → Catálogo e prestadores" />;
  }
  const lab = labs.find((l) => l.id === providerId) ?? labs[0];
  return (
    <div style={stack}>
      {labs.length > 1 && (
        <label style={fieldLabel}>Laboratório
          <select value={lab.id} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
            {labs.map((l) => <option key={l.id} value={l.id}>{l.name}</option>)}
          </select>
        </label>
      )}
      <LabExamsPanel key={lab.id} providerId={lab.id} onGoToSecurity={onGoToSecurity} />
    </div>
  );
}
```

Em `src/App.tsx`: `import { LabAdmin } from "./modules/LabAdmin";` e `case "lab-admin": return <LabAdmin onGoToSecurity={() => setActive("security")} />;`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/lib/labAdmin.test.ts src/modules/labAdmin/VersionForm.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/labAdmin.ts src/lib/labAdmin.test.ts src/modules/LabAdmin.tsx src/modules/labAdmin/LabExamsPanel.tsx src/modules/labAdmin/VersionForm.tsx \
  src/modules/labAdmin/VersionForm.test.tsx src/App.tsx
$G commit -m "feat: add lab admin to adopt base exams and create versions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Admin do laboratório — motivos de rejeição, liberadores (RT e substituto) e token do middleware

**Files:**
- Create: `src/modules/labAdmin/RejectionReasonsPanel.tsx`, `src/modules/labAdmin/ReleasersPanel.tsx`, `src/modules/labAdmin/MiddlewareTokenPanel.tsx`
- Modify: `src/modules/LabAdmin.tsx`
- Test: `src/modules/labAdmin/LabAccessPanels.test.tsx`

**Interfaces:**
- Consumes: `listRejectionReasons`, `createRejectionReason`, `updateRejectionReason`, `listLabReleasers`, `addLabReleaser`, `removeLabReleaser`, `listMemberships`, `rotateMiddlewareToken`, `revokeMiddlewareToken`, `ExamProvider`, `LabRejectionReason`, `LabReleaser` (Task 1 e existentes); `LAB_REASONS_KEY`, `LAB_RELEASERS_KEY`, `labError` (Task 2); `hasFeature`; `SensitiveAction`; `EXAM_ADMIN_PROVIDERS_KEY`.
- Produces: `RejectionReasonsPanel({ providerId, onGoToSecurity })`, `ReleasersPanel({ providerId, onGoToSecurity })`, `MiddlewareTokenPanel({ provider, onChanged, onGoToSecurity })`; `LabAdmin` mostra os três abaixo dos exames.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/labAdmin/LabAccessPanels.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listRejectionReasons: vi.fn(), createRejectionReason: vi.fn(), updateRejectionReason: vi.fn(),
    listLabReleasers: vi.fn(), addLabReleaser: vi.fn(), removeLabReleaser: vi.fn(), listMemberships: vi.fn(),
    rotateMiddlewareToken: vi.fn(), revokeMiddlewareToken: vi.fn() };
});

import * as api from "../../lib/api";
import { RejectionReasonsPanel } from "./RejectionReasonsPanel";
import { ReleasersPanel } from "./ReleasersPanel";
import { MiddlewareTokenPanel } from "./MiddlewareTokenPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { labUser } from "../../test/labFixtures";
import { examProvider } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const LAB = examProvider({ id: "pv9", name: "Laboratório Municipal", kind: "municipal", uses_municipal_lab: true, has_middleware_token: false });

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.listRejectionReasons, api.createRejectionReason, api.updateRejectionReason, api.listLabReleasers,
    api.addLabReleaser, api.removeLabReleaser, api.listMemberships, api.rotateMiddlewareToken, api.revokeMiddlewareToken ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "municipal_admin" ], { features: [ "exam_tracking", "municipal_lab", "lab_hl7_inbound" ] }));
  mocked(api.listRejectionReasons).mockResolvedValue([ { id: "rr1", provider_id: "pv9", name: "hemolisada", active: true } ]);
  mocked(api.createRejectionReason).mockResolvedValue({ id: "rr2", provider_id: "pv9", name: "sem identificação", active: true });
  mocked(api.listLabReleasers).mockResolvedValue([]);
  mocked(api.listMemberships).mockResolvedValue([
    { id: "m1", user: { id: "us7", email_address: "bia@curitiba.demo" }, role: "lab_releaser", granted_at: "2026-10-01T10:00:00-03:00" },
    { id: "m2", user: { id: "us8", email_address: "rita@curitiba.demo" }, role: "health_professional", granted_at: "2026-10-01T10:00:00-03:00" }
  ]);
  mocked(api.rotateMiddlewareToken).mockResolvedValue("mw_secret_123");
});

describe("motivos de rejeição", () => {
  it("lista e inclui com step-up", async () => {
    renderWithProviders(<RejectionReasonsPanel providerId="pv9" />);
    const panel = await screen.findByRole("region", { name: "Motivos de rejeição" });
    expect(within(panel).getByText("hemolisada")).not.toBeNull();
    fireEvent.change(within(panel).getByLabelText("Novo motivo"), { target: { value: "sem identificação" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Incluir motivo" }));
    fireEvent.click(await screen.findByRole("button", { name: "Incluir" }));
    await waitFor(() => expect(api.createRejectionReason).toHaveBeenCalledWith({ provider_id: "pv9", name: "sem identificação", active: true }));
  });
});

describe("liberadores", () => {
  it("só quem tem o papel lab_releaser; RT marcado; conselho faltando vira frase", async () => {
    mocked(api.addLabReleaser).mockRejectedValueOnce(new api.ApiError(422, { error: "council_missing" }, "x"));
    renderWithProviders(<ReleasersPanel providerId="pv9" />);
    const panel = await screen.findByRole("region", { name: "Liberadores" });
    const select = within(panel).getByLabelText("Pessoa com o papel de liberador") as HTMLSelectElement;
    expect([ ...select.options ].map((o) => o.textContent)).toEqual([ "—", "bia@curitiba.demo" ]);
    fireEvent.change(select, { target: { value: "us7" } });
    fireEvent.click(within(panel).getByLabelText("Responsável técnico (RT)"));
    fireEvent.click(within(panel).getByRole("button", { name: "Incluir liberador" }));
    fireEvent.click(await screen.findByRole("button", { name: "Incluir" }));
    await waitFor(() => expect(api.addLabReleaser).toHaveBeenCalledWith({ user_id: "us7", provider_id: "pv9", technical_lead: true, technical_deputy: false }));
    expect(await screen.findByText("o profissional não tem conselho, UF e registro no perfil — complete em Profissionais")).not.toBeNull();
  });

  it("RT e substituto não podem ser a mesma marca da mesma pessoa", async () => {
    renderWithProviders(<ReleasersPanel providerId="pv9" />);
    const panel = await screen.findByRole("region", { name: "Liberadores" });
    fireEvent.change(within(panel).getByLabelText("Pessoa com o papel de liberador"), { target: { value: "us7" } });
    fireEvent.click(within(panel).getByLabelText("Responsável técnico (RT)"));
    fireEvent.click(within(panel).getByLabelText("Substituto do RT"));
    fireEvent.click(within(panel).getByRole("button", { name: "Incluir liberador" }));
    expect(within(panel).getByRole("alert").textContent).toBe("a mesma pessoa não é RT e substituto ao mesmo tempo");
  });
});

describe("token do middleware", () => {
  it("token do middleware: aparece uma vez, some com Já guardei e nunca vai ao cache", async () => {
    const spies = [ "log", "info", "warn", "error", "debug" ].map((m) => vi.spyOn(console, m as "log"));
    const onChanged = vi.fn();
    const { client } = renderWithProviders(<MiddlewareTokenPanel provider={LAB} onChanged={onChanged} />);
    fireEvent.click(await screen.findByRole("button", { name: "Gerar token do middleware" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Gerar token do middleware — Laboratório Municipal" })).getByRole("button", { name: "Gerar token" }));
    const shown = await screen.findByRole("region", { name: "Token do middleware novo" });
    expect(within(shown).getByText("mw_secret_123")).not.toBeNull();
    expect(onChanged).toHaveBeenCalled();
    expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state))).not.toContain("mw_secret_123");
    expect(JSON.stringify({ ...localStorage })).not.toContain("mw_secret_123");
    expect(JSON.stringify({ ...sessionStorage })).not.toContain("mw_secret_123");
    for (const spy of spies) expect(JSON.stringify(spy.mock.calls)).not.toContain("mw_secret_123");
    fireEvent.click(within(shown).getByRole("button", { name: "Já guardei" }));
    expect(document.body.innerHTML).not.toContain("mw_secret_123");
  });

  it("sem a entrada HL7 ligada, avisa que o token ainda não funciona", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "municipal_admin" ]));
    renderWithProviders(<MiddlewareTokenPanel provider={{ ...LAB, has_middleware_token: true }} onChanged={vi.fn()} />);
    expect(await screen.findByText("a entrada de resultados de aparelhos (HL7) está desligada nesta cidade — o token só funciona quando ela for ligada")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Revogar token do middleware" })).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/labAdmin/LabAccessPanels.test.tsx`
Expected: FAIL — módulos inexistentes.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/labAdmin/RejectionReasonsPanel.tsx
// Motivos de rejeição do laboratório (spec §4; contrato §3): incluir, ativar e
// desativar (nunca apagar: amostras antigas apontam para eles). Step-up.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createRejectionReason, listRejectionReasons, updateRejectionReason, type LabRejectionReason } from "../../lib/api";
import { LAB_REASONS_KEY, labError } from "../../lib/lab";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { SensitiveAction } from "../../components/SensitiveAction";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, row, stack } from "../exams/styles";

type Pending = null | { kind: "create"; name: string } | { kind: "toggle"; reason: LabRejectionReason };

export function RejectionReasonsPanel({ providerId, onGoToSecurity }: { providerId: string; onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const reasons = useQuery({ queryKey: [ LAB_REASONS_KEY, "admin", providerId ], queryFn: () => listRejectionReasons(providerId), retry: false });
  const [ name, setName ] = useState("");
  const [ pending, setPending ] = useState<Pending>(null);
  const [ error, setError ] = useState<string | null>(null);
  const refresh = () => queryClient.invalidateQueries({ queryKey: [ LAB_REASONS_KEY ] });

  return (
    <Panel title="Motivos de rejeição" sub="usados no recebimento; desative em vez de apagar">
      <div style={stack}>
        {reasons.isError && <p role="alert" style={alert}>{labError(reasons.error)}</p>}
        <DataTable<LabRejectionReason>
          cols={[
            { label: "Motivo", w: "2fr", card: "title", render: (r) => r.name },
            { label: "Situação", w: "1fr", render: (r) => <Tag tone={r.active ? "ok" : undefined}>{r.active ? "ativo" : "inativo"}</Tag> },
            { label: "", w: "auto", card: "actions", render: (r) => (
              <button type="button" style={secondaryButtonStyle} onClick={() => setPending({ kind: "toggle", reason: r })}>{r.active ? "Desativar" : "Ativar"}</button>
            ) }
          ]}
          rows={reasons.data ?? []} rowKey={(r) => r.id} empty={reasons.isPending ? "carregando…" : "nenhum motivo cadastrado"}
        />
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>Novo motivo<input value={name} style={inputStyle} onChange={(e) => setName(e.target.value)} /></label>
          <button type="button" style={secondaryButtonStyle}
            onClick={() => { if (!name.trim()) { setError("escreva o motivo"); return; } setError(null); setPending({ kind: "create", name: name.trim() }); }}>Incluir motivo</button>
        </div>
        {error && <p role="alert" style={alert}>{error}</p>}
        {pending?.kind === "create" && (
          <SensitiveAction title={`Incluir motivo — ${pending.name}`} description="O motivo passa a aparecer no recebimento." requiresStepUp confirmLabel="Incluir"
            run={async () => { await createRejectionReason({ provider_id: providerId, name: pending.name, active: true }); }}
            onDone={() => { setPending(null); setName(""); void refresh(); }} onCancel={() => setPending(null)} onGoToSecurity={onGoToSecurity}
            translateError={(err) => labError(err)} />
        )}
        {pending?.kind === "toggle" && (
          <SensitiveAction title={`${pending.reason.active ? "Desativar" : "Ativar"} motivo — ${pending.reason.name}`}
            description={pending.reason.active ? "O motivo some do recebimento; as rejeições antigas continuam com ele." : "O motivo volta ao recebimento."}
            requiresStepUp confirmLabel={pending.reason.active ? "Desativar" : "Ativar"}
            run={async () => { await updateRejectionReason(pending.reason.id, { name: pending.reason.name, active: !pending.reason.active }); }}
            onDone={() => { setPending(null); void refresh(); }} onCancel={() => setPending(null)} onGoToSecurity={onGoToSecurity}
            translateError={(err) => labError(err)} />
        )}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/labAdmin/ReleasersPanel.tsx
// Liberadores do laboratório (spec §4; contrato §3): quem tem o papel
// `lab_releaser` (dado pela Equipe) é ligado ao laboratório, com as marcas de
// responsável técnico (RT) e substituto. O conselho vem do perfil profissional
// (módulo 10); sem ele, 422 council_missing. Trocar a marca = tirar e incluir.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { addLabReleaser, listLabReleasers, listMemberships, removeLabReleaser, type LabReleaser } from "../../lib/api";
import { LAB_RELEASERS_KEY, labError } from "../../lib/lab";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { SensitiveAction } from "../../components/SensitiveAction";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, checkLabel, fieldLabel, muted, row, stack } from "../exams/styles";

type Pending = null | { kind: "add"; userId: string; email: string; lead: boolean; deputy: boolean } | { kind: "remove"; releaser: LabReleaser };

export function ReleasersPanel({ providerId, onGoToSecurity }: { providerId: string; onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const releasers = useQuery({ queryKey: [ LAB_RELEASERS_KEY ], queryFn: listLabReleasers, retry: false });
  const members = useQuery({ queryKey: [ "memberships" ], queryFn: listMemberships, retry: false });
  const [ userId, setUserId ] = useState("");
  const [ lead, setLead ] = useState(false);
  const [ deputy, setDeputy ] = useState(false);
  const [ pending, setPending ] = useState<Pending>(null);
  const [ error, setError ] = useState<string | null>(null);
  const mine = (releasers.data ?? []).filter((r) => r.provider_id === providerId);
  const candidates = (members.data ?? []).filter((m) => m.role === "lab_releaser" && !mine.some((r) => r.user_id === m.user.id));
  const refresh = () => queryClient.invalidateQueries({ queryKey: [ LAB_RELEASERS_KEY ] });

  function add() {
    if (!userId) { setError("escolha a pessoa"); return; }
    if (lead && deputy) { setError("a mesma pessoa não é RT e substituto ao mesmo tempo"); return; }
    setError(null);
    const email = candidates.find((c) => c.user.id === userId)?.user.email_address ?? "";
    setPending({ kind: "add", userId, email, lead, deputy });
  }

  return (
    <Panel title="Liberadores" sub="quem assina laudos neste laboratório · RT e substituto aparecem no laudo">
      <div style={stack}>
        {releasers.isError && <p role="alert" style={alert}>{labError(releasers.error)}</p>}
        <DataTable<LabReleaser>
          cols={[
            { label: "Pessoa", w: "1.6fr", card: "title", render: (r) => `${r.name} · ${r.email}` },
            { label: "Conselho", w: "1.2fr", render: (r) => (r.council ? `${r.council}-${r.council_uf ?? "?"} ${r.council_number ?? ""}` : "—") },
            { label: "Marca", w: "1fr", render: (r) => <>{r.technical_lead && <Tag tone="accent">RT</Tag>}{r.technical_deputy && <Tag>substituto</Tag>}</> },
            { label: "", w: "auto", card: "actions", render: (r) => (
              <button type="button" style={secondaryButtonStyle} onClick={() => setPending({ kind: "remove", releaser: r })}>Tirar</button>
            ) }
          ]}
          rows={mine} rowKey={(r) => r.id} empty={releasers.isPending ? "carregando…" : "nenhum liberador neste laboratório"}
        />
        <p style={muted}>O papel de liberador é dado na Equipe; o conselho, a UF e o registro vêm do perfil em Profissionais.</p>
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>
            Pessoa com o papel de liberador
            <select value={userId} style={inputStyle} onChange={(e) => setUserId(e.target.value)}>
              <option value="">—</option>
              {candidates.map((c) => <option key={c.user.id} value={c.user.id}>{c.user.email_address}</option>)}
            </select>
          </label>
          <label style={checkLabel}><input type="checkbox" checked={lead} onChange={(e) => setLead(e.target.checked)} />Responsável técnico (RT)</label>
          <label style={checkLabel}><input type="checkbox" checked={deputy} onChange={(e) => setDeputy(e.target.checked)} />Substituto do RT</label>
          <button type="button" style={secondaryButtonStyle} onClick={add}>Incluir liberador</button>
        </div>
        {error && <p role="alert" style={alert}>{error}</p>}
        {pending?.kind === "add" && (
          <SensitiveAction title={`Incluir liberador — ${pending.email}`}
            description={`Passa a liberar laudos deste laboratório${pending.lead ? " como responsável técnico" : pending.deputy ? " como substituto do RT" : ""}.`}
            requiresStepUp confirmLabel="Incluir"
            run={async () => { await addLabReleaser({ user_id: pending.userId, provider_id: providerId, technical_lead: pending.lead, technical_deputy: pending.deputy }); }}
            onDone={() => { setPending(null); setUserId(""); setLead(false); setDeputy(false); void refresh(); }} onCancel={() => setPending(null)}
            onGoToSecurity={onGoToSecurity} translateError={(err) => labError(err)} />
        )}
        {pending?.kind === "remove" && (
          <SensitiveAction title={`Tirar liberador — ${pending.releaser.name}`} description="Deixa de liberar laudos deste laboratório; os laudos já assinados não mudam."
            requiresStepUp confirmLabel="Tirar"
            run={async () => { await removeLabReleaser(pending.releaser.id); }}
            onDone={() => { setPending(null); void refresh(); }} onCancel={() => setPending(null)} onGoToSecurity={onGoToSecurity}
            translateError={(err) => labError(err)} />
        )}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/labAdmin/MiddlewareTokenPanel.tsx
// Token do middleware do fornecedor (spec §6; contrato §3 e §7): mostrado UMA
// vez — vive só neste estado, nunca no cache; some com "Já guardei" ou ao
// sair. Trocar invalida o anterior. Sem `lab_hl7_inbound`, avisa que o token
// só funciona quando a entrada for ligada (no maintenance).
import { useState } from "react";
import { revokeMiddlewareToken, rotateMiddlewareToken, type ExamProvider } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { hasFeature } from "../../lib/features";
import { labError } from "../../lib/lab";
import { Panel } from "../../components/Panel";
import { SensitiveAction } from "../../components/SensitiveAction";
import { dangerButtonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, muted, okText, row, stack } from "../exams/styles";

export function MiddlewareTokenPanel({ provider, onChanged, onGoToSecurity }: { provider: ExamProvider; onChanged(): void; onGoToSecurity?(): void }) {
  const { user } = useAuth();
  const [ confirming, setConfirming ] = useState<null | "rotate" | "revoke">(null);
  const [ token, setToken ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const hasToken = provider.has_middleware_token === true;
  const idle = confirming === null && token === null;

  return (
    <Panel title="Middleware dos aparelhos" sub="token do sistema do fornecedor que envia resultados por HL7">
      <div style={stack}>
        {!hasFeature(user, "lab_hl7_inbound") && (
          <p style={muted}>a entrada de resultados de aparelhos (HL7) está desligada nesta cidade — o token só funciona quando ela for ligada</p>
        )}
        {notice && <p role="status" style={okText}>{notice}</p>}
        {token && (
          <section aria-label="Token do middleware novo" style={box}>
            <p style={alert}>Copie agora: o token não aparece de novo. Ele substitui o anterior, que deixa de funcionar.</p>
            <code className="mono" style={{ overflowWrap: "anywhere", fontSize: 13, userSelect: "all" }}>{token}</code>
            <div style={row}><button type="button" style={secondaryButtonStyle} onClick={() => setToken(null)}>Já guardei</button></div>
          </section>
        )}
        {idle && (
          <div style={row}>
            <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setConfirming("rotate"); }}>
              {hasToken ? "Trocar token do middleware" : "Gerar token do middleware"}
            </button>
            {hasToken && <button type="button" style={dangerButtonStyle} onClick={() => { setNotice(null); setConfirming("revoke"); }}>Revogar token do middleware</button>}
          </div>
        )}
        {confirming === "rotate" && (
          <SensitiveAction title={`${hasToken ? "Trocar" : "Gerar"} token do middleware — ${provider.name}`}
            description={hasToken ? "O token atual deixa de funcionar na hora." : "O middleware usa o token para pedir a lista de trabalho e enviar resultados."}
            requiresStepUp confirmLabel={hasToken ? "Trocar token" : "Gerar token"}
            run={async () => { setToken(await rotateMiddlewareToken(provider.id)); }}
            onDone={() => { setConfirming(null); onChanged(); }} onCancel={() => setConfirming(null)} onGoToSecurity={onGoToSecurity}
            translateError={(err) => labError(err)} />
        )}
        {confirming === "revoke" && (
          <SensitiveAction title={`Revogar token do middleware — ${provider.name}`} description="O middleware deixa de enviar resultados até receber um token novo."
            requiresStepUp confirmLabel="Revogar token"
            run={async () => { await revokeMiddlewareToken(provider.id); }}
            onDone={() => { setConfirming(null); setNotice("Token revogado."); onChanged(); }} onCancel={() => setConfirming(null)}
            onGoToSecurity={onGoToSecurity} translateError={(err) => labError(err)} />
        )}
      </div>
    </Panel>
  );
}
```

Em `src/modules/LabAdmin.tsx`, depois do `LabExamsPanel`:

```tsx
// imports
import { useQueryClient } from "@tanstack/react-query";
import { RejectionReasonsPanel } from "./labAdmin/RejectionReasonsPanel";
import { ReleasersPanel } from "./labAdmin/ReleasersPanel";
import { MiddlewareTokenPanel } from "./labAdmin/MiddlewareTokenPanel";
```

```tsx
// em AdminBody, junto dos hooks (antes dos returns): const queryClient = useQueryClient();
// e no JSX, depois do LabExamsPanel:
      <RejectionReasonsPanel key={`r-${lab.id}`} providerId={lab.id} onGoToSecurity={onGoToSecurity} />
      <ReleasersPanel key={`l-${lab.id}`} providerId={lab.id} onGoToSecurity={onGoToSecurity} />
      <MiddlewareTokenPanel key={`m-${lab.id}`} provider={lab} onGoToSecurity={onGoToSecurity}
        onChanged={() => void queryClient.invalidateQueries({ queryKey: [ EXAM_ADMIN_PROVIDERS_KEY ] })} />
```

> O `key` por laboratório descarta o token na tela quando o admin troca de laboratório (o estado do painel some com ele).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/labAdmin && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/labAdmin/RejectionReasonsPanel.tsx src/modules/labAdmin/ReleasersPanel.tsx src/modules/labAdmin/MiddlewareTokenPanel.tsx \
  src/modules/labAdmin/LabAccessPanels.test.tsx src/modules/LabAdmin.tsx
$G commit -m "feat: add rejection reasons, releasers and middleware token to lab admin

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: Marcas `uses_municipal_lab` (prestador) e `collects_samples` (unidade)

**Files:**
- Modify: `src/lib/examsAdmin.ts`, `src/modules/examsAdmin/ProviderForm.tsx`, `src/modules/examsAdmin/ProvidersPanel.tsx`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/examsAdmin/ProviderForm.lab.test.tsx`, `src/modules/attendance/UnitForm.lab.test.tsx`

**Interfaces:**
- Consumes: `ExamProvider`, `ExamProviderInput`, `updateUnit` (6º parâmetro), `HealthUnitRow` (Task 1); `hasFeature`; `useAuth`.
- Produces: `ProviderDraft.usesMunicipalLab: boolean`; `providerDraft(p)` lê `uses_municipal_lab`; `providerBody(d, labOn = false)` manda `uses_municipal_lab` só com `labOn`; `ProviderForm` ganha a prop `labOn?: boolean` e o checkbox "Usa o laboratório municipal do Rota Saúde"; `UnitFormValue.collectsSamples?: boolean` e o checkbox "Coleta amostras para o laboratório municipal" (só na edição com `municipal_lab`); `Units({ labOn = false })` manda `collectsSamples` ao `updateUnit` e mostra a coluna "Coleta" com `labOn`; `Attendance` passa `labOn={hasFeature(user, "municipal_lab")}`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/examsAdmin/ProviderForm.lab.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { ProviderForm } from "./ProviderForm";
import { providerBody, providerDraft } from "../../lib/examsAdmin";
import { examCatalogItem, examProvider } from "../../test/examFixtures";

afterEach(cleanup);

describe("ProviderForm — laboratório municipal", () => {
  it("com municipal_lab: checkbox vai no corpo", () => {
    const onReview = vi.fn();
    render(<ProviderForm initial={examProvider({ kind: "municipal" })} catalog={[ examCatalogItem() ]} labOn onReview={onReview} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByLabelText("Usa o laboratório municipal do Rota Saúde (resultado sai pela liberação do laudo)"));
    fireEvent.click(screen.getByRole("button", { name: "Revisar e salvar" }));
    expect(onReview.mock.calls[0][0].uses_municipal_lab).toBe(true);
  });

  it("sem municipal_lab: nem checkbox nem campo no corpo", () => {
    render(<ProviderForm initial={examProvider({ uses_municipal_lab: true })} catalog={[ examCatalogItem() ]} onReview={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.queryByLabelText(/Usa o laboratório municipal/)).toBeNull();
    expect("uses_municipal_lab" in providerBody(providerDraft(examProvider({ uses_municipal_lab: true })))).toBe(false);
    expect(providerBody(providerDraft(examProvider({ uses_municipal_lab: true })), true).uses_municipal_lab).toBe(true);
  });
});
```

> O botão de revisão do `ProviderForm` em `origin/main` é "Revisar e salvar" (`type="submit"` dentro do `<form>`; o clique dispara o envio no jsdom).

```tsx
// src/modules/attendance/UnitForm.lab.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { UnitForm, type UnitFormValue } from "./UnitForm";

afterEach(cleanup);

const BASE: UnitFormValue = { name: "UBS Centro", kind: "ubs", street: "", number: "", complement: "", zip: "", neighborhoodId: "", screeningScope: "walk_in" };

describe("UnitForm — coleta de amostras (módulo 21b)", () => {
  it("na edição com o campo presente, o checkbox muda o valor salvo", () => {
    const onSave = vi.fn();
    render(<UnitForm initial={{ ...BASE, collectsSamples: false }} neighborhoods={[]} busy={false} onSave={onSave} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByLabelText("Coleta amostras para o laboratório municipal"));
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    expect(onSave.mock.calls[0][0].collectsSamples).toBe(true);
  });

  it("sem o campo (interruptor desligado ou unidade nova), nada aparece", () => {
    render(<UnitForm initial={BASE} neighborhoods={[]} busy={false} onSave={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.queryByLabelText("Coleta amostras para o laboratório municipal")).toBeNull();
  });
});
```

> Os campos de endereço de `UnitFormValue` vêm de `AddressFields` (`src/lib/unitAddress.ts`: `street`, `number`, `complement`, `zip`, `neighborhoodId`, conferidos em `origin/main`). O `UnitForm` só salva com nome preenchido — o `BASE` já tem.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/modules/examsAdmin/ProviderForm.lab.test.tsx src/modules/attendance/UnitForm.lab.test.tsx`
Expected: FAIL — checkbox inexistente, `providerBody` sem o 2º parâmetro.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/examsAdmin.ts`:

```ts
export interface ProviderDraft {
  name: string; cnes: string; kind: ExamProviderKind; active: boolean;
  // catalog_item_id → prazos próprios; presente = o prestador faz o exame.
  items: Record<string, DeadlineDraft>;
  // Módulo 21b: resultado pela liberação do laboratório municipal.
  usesMunicipalLab: boolean;
}

export function providerDraft(p?: ExamProvider | null): ProviderDraft {
  return {
    name: p?.name ?? "",
    cnes: p?.cnes ?? "",
    kind: p?.kind ?? "accredited",
    active: p?.active ?? true,
    items: Object.fromEntries((p?.items ?? []).map((i) => [ i.catalog_item_id, deadlineDraft(i.deadlines) ])),
    usesMunicipalLab: p?.uses_municipal_lab === true
  };
}

// Só os campos que o admin escreve: `simulated`, token e usuários nunca vão.
// `uses_municipal_lab` só com o interruptor `municipal_lab` (o api recusaria o campo sem ele).
export function providerBody(d: ProviderDraft, labOn = false): ExamProviderInput {
  return {
    name: d.name.trim(),
    cnes: d.cnes.trim() || null,
    kind: d.kind,
    active: d.active,
    items: Object.entries(d.items).map(([ catalog_item_id, deadlines ]) => ({ catalog_item_id, deadlines: deadlinesFrom(deadlines) })),
    ...(labOn ? { uses_municipal_lab: d.usesMunicipalLab } : {})
  };
}
```

Em `src/modules/examsAdmin/ProviderForm.tsx`: acrescente `labOn = false` às props (`{ initial, catalog, labOn = false, onReview, onCancel }` com `labOn?: boolean` no tipo), troque `onReview(providerBody(draft))` por `onReview(providerBody(draft, labOn))` e, logo depois do checkbox "Ativo":

```tsx
      {labOn && (
        <label style={checkLabel}>
          <input type="checkbox" checked={draft.usesMunicipalLab} onChange={(e) => set({ usesMunicipalLab: e.target.checked })} />
          Usa o laboratório municipal do Rota Saúde (resultado sai pela liberação do laudo)
        </label>
      )}
```

Em `ProvidersPanel.tsx` (hoje sem `useAuth`; ele só roda dentro do `ExamsAdmin`, que tem o `AuthProvider` acima, inclusive no teste com `renderWithProviders`): acrescente `import { useAuth } from "../../lib/auth";`, `import { hasFeature } from "../../lib/features";`, `const { user } = useAuth();` no começo do componente e passe `labOn={hasFeature(user, "municipal_lab")}` no `<ProviderForm …>`. O teste antigo `ExamsAdmin.providers.test.tsx` segue verde: sem o interruptor, o corpo não muda.

Em `src/modules/attendance/UnitForm.tsx`:

```ts
// `collectsSamples` (módulo 21b) só existe na edição com `municipal_lab` ligado.
export interface UnitFormValue extends AddressFields { name: string; kind: string; screeningScope?: ScreeningScope; collectsSamples?: boolean }
```

e, depois do bloco do `screeningScope`:

```tsx
      {value.collectsSamples !== undefined && (
        <label style={{ ...labelStyle, flexDirection: "row", alignItems: "center", gap: 6 }}>
          <input type="checkbox" checked={value.collectsSamples} onChange={(e) => set({ collectsSamples: e.target.checked })} />
          Coleta amostras para o laboratório municipal
        </label>
      )}
```

Em `src/modules/attendance/Units.tsx` (o `Units.test.tsx` renderiza sem `AuthProvider`, e `useAuth` lança fora dele — por isso a marca chega por prop):

```ts
// Módulo 21b: `labOn` = `municipal_lab` ligado (vem do Atendimento, que já lê a sessão).
export function Units({ labOn = false }: { labOn?: boolean } = {}) {
```

```ts
// em save():
        await updateUnit(editing.id, value.name, value.kind, address, value.screeningScope, value.collectsSamples);
```

```tsx
// no botão Editar, o initial ganha a marca só com o interruptor:
                      onClick={() => setEditing({ id: r.id, initial: {
                        name: r.name, kind: r.kind, ...addressFieldsFrom(r), screeningScope: r.screening_scope ?? "walk_in",
                        ...(labOn ? { collectsSamples: r.collects_samples === true } : {})
                      } })}>
```

```tsx
// na lista, depois da coluna "Acolhimento", só com o interruptor:
              ...(labOn ? [ { label: "Coleta", w: "0.8fr", render: (r: HealthUnitRow) => (r.collects_samples ? "coleta" : "—") } ] : []),
```

Em `src/modules/Attendance.tsx`, troque `{isAdmin && <Units />}` por `{isAdmin && <Units labOn={hasFeature(user, "municipal_lab")} />}` (acrescente `import { hasFeature } from "../lib/features";` se faltar).

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/modules/examsAdmin src/modules/attendance src/modules/Attendance.test.tsx src/modules/ExamsAdmin.providers.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/examsAdmin.ts src/modules/examsAdmin/ProviderForm.tsx src/modules/examsAdmin/ProvidersPanel.tsx src/modules/examsAdmin/ProviderForm.lab.test.tsx \
  src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/attendance/UnitForm.lab.test.tsx src/modules/Attendance.tsx
$G commit -m "feat: add municipal lab flag to providers and sample collection flag to units

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 18: Relatórios de produção, rejeição e prazo, com CSV

**Files:**
- Create: `src/modules/LabReports.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/LabReports.test.tsx`

**Interfaces:**
- Consumes: `getLabProduction`, `getLabRejections`, `getLabTurnaround`, `fetchLabReportCsv`, `LabReportKind`, `LabProductionRow`, `LabRejectionRow`, `LabTurnaroundRow`, `SignatureFile` (Task 1); `LAB_REPORTS_KEY`, `labError`, `sectorLabel` (Task 2); `LabGate`, `useLabContext` (Task 4); `todayInCity`.
- Produces: `LabReports()` em `renderModule` como `case "lab-reports"`; `fmtCents(cents: number): string` (exportada).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/LabReports.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getLabContext: vi.fn(), getLabProduction: vi.fn(), getLabRejections: vi.fn(),
    getLabTurnaround: vi.fn(), fetchLabReportCsv: vi.fn() };
});

import * as api from "../lib/api";
import { LabReports, fmtCents } from "./LabReports";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21B, labContext, labUser } from "../test/labFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let downloads: string[];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date(NOW21B));
  for (const fn of [ api.fetchCurrentSession, api.getLabContext, api.getLabProduction, api.getLabRejections, api.getLabTurnaround, api.fetchLabReportCsv ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "municipal_admin" ]));
  mocked(api.getLabContext).mockResolvedValue(labContext());
  mocked(api.getLabProduction).mockResolvedValue({
    rows: [ { origin_unit: { id: "u1", name: "UBS Centro" }, sigtap_code: "0202010473", exam_name: "Glicose", sector: "bioquimica",
      releaser: { id: "us7", name: "Bia Farmacêutica" }, quantity: 120, sigtap_value_cents: 22800 } ],
    total_quantity: 120, total_value_cents: 22800
  });
  mocked(api.getLabRejections).mockResolvedValue([ { reason: { id: "rr1", name: "hemolisada" }, origin_unit: { id: "u1", name: "UBS Centro" }, count: 4 } ]);
  mocked(api.getLabTurnaround).mockResolvedValue([ { sigtap_code: "0202010473", exam_name: "Glicose", on_time: 110, late: 6, pending: 4 } ]);
  mocked(api.fetchLabReportCsv).mockResolvedValue({ blob: new Blob([ "a\n" ], { type: "text/csv" }), filename: "x.csv" });
  downloads = [];
  Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:csv") });
  Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) { downloads.push(this.download); });
});

describe("LabReports", () => {
  it("mês da cidade e todos os laboratórios; produção com quantidade e valor SIGTAP", async () => {
    renderWithProviders(<LabReports />);
    const prod = await screen.findByRole("region", { name: "Produção" });
    expect(await within(prod).findByText("Bia Farmacêutica")).not.toBeNull();
    // A linha e o total mostram o mesmo valor.
    expect(within(prod).getAllByText("R$ 228,00", { exact: false }).length).toBe(2);
    expect(api.getLabProduction).toHaveBeenCalledWith({ providerId: null, month: "2026-10" });
    fireEvent.change(screen.getByLabelText("Laboratório"), { target: { value: "pv9" } });
    await waitFor(() => expect(api.getLabTurnaround).toHaveBeenLastCalledWith({ providerId: "pv9", month: "2026-10" }));
  });

  it("rejeição e prazo", async () => {
    renderWithProviders(<LabReports />);
    const rej = await screen.findByRole("region", { name: "Rejeições" });
    expect(await within(rej).findByText("hemolisada")).not.toBeNull();
    const tat = screen.getByRole("region", { name: "Prazo de entrega" });
    expect(await within(tat).findByText("110")).not.toBeNull();
  });

  it("CSV de cada relatório com o nome montado pela tela", async () => {
    renderWithProviders(<LabReports />);
    const rej = await screen.findByRole("region", { name: "Rejeições" });
    fireEvent.click(within(rej).getByRole("button", { name: "Baixar CSV" }));
    await waitFor(() => expect(api.fetchLabReportCsv).toHaveBeenCalledWith("rejections", { providerId: null, month: "2026-10" }));
    await waitFor(() => expect(downloads).toEqual([ "laboratorio-rejeicoes-2026-10.csv" ]));
  });

  it("a equipe do laboratório não vê os relatórios", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(labUser([ "lab_staff" ]));
    renderWithProviders(<LabReports />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.getLabProduction).not.toHaveBeenCalled();
  });

  it("valor em reais", () => {
    expect(fmtCents(22800).replace(/\s/g, " ")).toBe("R$ 228,00");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/LabReports.test.tsx`
Expected: FAIL — `./LabReports` inexistente.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/LabReports.tsx
// Relatórios do laboratório (spec §7; contrato §6; Divergência D13):
// produção por unidade de origem, exame, setor e liberador com quantidade e
// valor SIGTAP; rejeições por motivo e unidade; prazo de entrega (no prazo,
// atrasado, pendente). Tela e CSV; admin municipal e liberador. Sem dado de
// paciente.
import { useRef, useState, type ReactNode } from "react";
import { useQuery } from "@tanstack/react-query";
import {
  fetchLabReportCsv, getLabProduction, getLabRejections, getLabTurnaround,
  type LabProductionRow, type LabRejectionRow, type LabReportKind, type LabReportQuery, type LabTurnaroundRow, type SignatureFile
} from "../lib/api";
import { LAB_REPORTS_KEY, labError, sectorLabel } from "../lib/lab";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { ActionButton } from "../components/ActionButton";
import { inputStyle } from "../components/formStyles";
import { LabGate } from "./lab/LabGate";
import { useLabContext } from "./lab/useLabContext";
import { alert, fieldLabel, muted, row, stack } from "./exams/styles";

const BLOB_TTL_MS = 60_000;
const FILE_NAME: Record<LabReportKind, string> = { production: "producao", rejections: "rejeicoes", turnaround: "prazo" };

export function fmtCents(cents: number): string {
  return (cents / 100).toLocaleString("pt-BR", { style: "currency", currency: "BRL" });
}

function saveFile(file: SignatureFile, name: string) {
  const url = URL.createObjectURL(file.blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = name;
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(url), BLOB_TTL_MS);
}

export function LabReports() {
  return (
    <LabGate title="Relatórios do laboratório" sub="laboratório municipal · produção, rejeição e prazo" allow={(r) => r.admin || r.releaser}>
      {() => <ReportsBody />}
    </LabGate>
  );
}

function ReportsBody() {
  const context = useLabContext(true);
  const [ month, setMonth ] = useState(() => todayInCity().slice(0, 7));
  const [ providerId, setProviderId ] = useState("");
  const [ downloadError, setDownloadError ] = useState<string | null>(null);
  const busy = useRef(false);
  const [ downloading, setDownloading ] = useState(false);
  const valid = /^\d{4}-\d{2}$/.test(month);
  const q: LabReportQuery = { providerId: providerId || null, month };
  const production = useQuery({ queryKey: [ LAB_REPORTS_KEY, "production", q ], queryFn: () => getLabProduction(q), enabled: valid, retry: false });
  const rejections = useQuery({ queryKey: [ LAB_REPORTS_KEY, "rejections", q ], queryFn: () => getLabRejections(q), enabled: valid, retry: false });
  const turnaround = useQuery({ queryKey: [ LAB_REPORTS_KEY, "turnaround", q ], queryFn: () => getLabTurnaround(q), enabled: valid, retry: false });

  async function download(kind: LabReportKind) {
    if (busy.current) return;
    busy.current = true;
    setDownloading(true);
    setDownloadError(null);
    try {
      saveFile(await fetchLabReportCsv(kind, q), `laboratorio-${FILE_NAME[kind]}-${month}.csv`);
    } catch (err) {
      setDownloadError(labError(err));
    } finally {
      busy.current = false;
      setDownloading(false);
    }
  }

  const csv = (kind: LabReportKind) => (
    <ActionButton variant="secondary" busy={downloading} disabledReason={valid ? null : "escolha o mês"} onClick={() => download(kind)}>Baixar CSV</ActionButton>
  );
  const state = (query: { isPending: boolean; isError: boolean; error: unknown }, body: ReactNode) =>
    query.isError ? <p role="alert" style={alert}>{labError(query.error)}</p> : query.isPending && valid ? <p className="mono" style={muted}>carregando…</p> : body;

  return (
    <div style={stack}>
      <div style={{ ...row, alignItems: "flex-end" }}>
        <label style={fieldLabel}>Mês<input type="month" value={month} style={inputStyle} onChange={(e) => setMonth(e.target.value)} /></label>
        <label style={fieldLabel}>
          Laboratório
          <select value={providerId} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
            <option value="">todos</option>
            {(context.data?.labs ?? []).map((l) => <option key={l.id} value={l.id}>{l.name}</option>)}
          </select>
        </label>
      </div>
      {!valid && <p role="alert" style={alert}>escolha o mês</p>}
      {downloadError && <p role="alert" style={alert}>{downloadError}</p>}
      <Panel title="Produção" sub="por unidade de origem, exame, setor e liberador · quantidade e valor SIGTAP" right={csv("production")}>
        {state(production, production.data && (
          <>
            <DataTable<LabProductionRow>
              cols={[
                { label: "Unidade", w: "1.3fr", card: "title", render: (r) => r.origin_unit.name },
                { label: "Exame", w: "1.4fr", render: (r) => `${r.exam_name} (${r.sigtap_code})` },
                { label: "Setor", w: "1fr", render: (r) => sectorLabel(r.sector) },
                { label: "Liberador", w: "1.2fr", render: (r) => r.releaser?.name ?? "—" },
                { label: "Quantidade", w: "0.8fr", align: "right", render: (r) => String(r.quantity) },
                { label: "Valor SIGTAP", w: "1fr", align: "right", render: (r) => fmtCents(r.sigtap_value_cents) }
              ]}
              rows={production.data.rows} rowKey={(r, i) => `${r.origin_unit.id}:${r.sigtap_code}:${r.releaser?.id ?? "-"}:${i}`} empty="nenhum laudo liberado neste mês"
            />
            <p style={muted}>{`Total: ${production.data.total_quantity} exames · ${fmtCents(production.data.total_value_cents)}`}</p>
          </>
        ))}
      </Panel>
      <Panel title="Rejeições" sub="por motivo e unidade de origem" right={csv("rejections")}>
        {state(rejections, rejections.data && (
          <DataTable<LabRejectionRow>
            cols={[
              { label: "Motivo", w: "1.4fr", card: "title", render: (r) => r.reason.name },
              { label: "Unidade", w: "1.4fr", render: (r) => r.origin_unit.name },
              { label: "Amostras", w: "0.8fr", align: "right", render: (r) => String(r.count) }
            ]}
            rows={rejections.data} rowKey={(r) => `${r.reason.id}:${r.origin_unit.id}`} empty="nenhuma rejeição neste mês"
          />
        ))}
      </Panel>
      <Panel title="Prazo de entrega" sub="laudos no prazo, atrasados e pendentes, por exame" right={csv("turnaround")}>
        {state(turnaround, turnaround.data && (
          <DataTable<LabTurnaroundRow>
            cols={[
              { label: "Exame", w: "1.6fr", card: "title", render: (r) => `${r.exam_name} (${r.sigtap_code})` },
              { label: "No prazo", w: "0.8fr", align: "right", render: (r) => String(r.on_time) },
              { label: "Atrasados", w: "0.8fr", align: "right", render: (r) => String(r.late) },
              { label: "Pendentes", w: "0.8fr", align: "right", render: (r) => String(r.pending) }
            ]}
            rows={turnaround.data} rowKey={(r) => r.sigtap_code} empty="nenhum exame neste mês"
          />
        ))}
      </Panel>
    </div>
  );
}
```

> O `Panel` aceita `right` (slot do cabeçalho) — conferido em `src/components/Panel.tsx`. O CSV leva só o que a tela mostra: nenhum dado de paciente.

Em `src/App.tsx`: `import { LabReports } from "./modules/LabReports";` e `case "lab-reports": return <LabReports />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/LabReports.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/LabReports.tsx src/modules/LabReports.test.tsx src/App.tsx
$G commit -m "feat: add lab production, rejection and turnaround reports with CSV

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: No 21a — resultado do laboratório municipal, laudo substituído, origem `lab_reception` e GAL

**Files:**
- Create: `src/modules/exams/GalRequisitionForm.tsx`
- Modify: `src/lib/exams.ts`, `src/modules/exams/ExamResultBlock.tsx`, `src/modules/exams/ExamOrderDetail.tsx`
- Test: `src/modules/exams/ExamResultBlock.lab.test.tsx`, `src/modules/exams/ExamOrderDetail.lab.test.tsx`

**Interfaces:**
- Consumes: `ExamResult` (com `source: "lab"`, `replaces_result_id`, `superseded_at`), `ExamOrder` (com `superseded_results`, `gal_requisition`), `setGalRequisition` (Task 1); `examError`, `examRoles`, `refreshExams` (21a); `examOrder`, `examResult`, `examUser` (`examFixtures`).
- Produces: `ORIGIN_LABEL.lab_reception = "recepção do laboratório"`; `SOURCE_LABEL.lab = "laudo do laboratório municipal"`; `ERRORS.invalid_gal_requisition`; `galProblem(value: string): string | null` em `lib/exams.ts`; `ExamResultBlock` ganha a prop `superseded?: boolean`; `ExamOrderDetail` mostra "Laudos substituídos" e o GAL; `GalRequisitionForm({ orderId, current, onDone, onCancel })`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/exams/ExamResultBlock.lab.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, within } from "@testing-library/react";
import { ExamResultBlock } from "./ExamResultBlock";
import { examResult } from "../../test/examFixtures";

afterEach(cleanup);

describe("ExamResultBlock — laboratório municipal (21b)", () => {
  it("fonte lab e retificação vigente", () => {
    render(<ExamResultBlock result={examResult({ source: "lab", laboratory_name: "Laboratório Municipal", replaces_result_id: "er0" })} onExpired={vi.fn()} />);
    const block = screen.getByRole("region", { name: "Resultado" });
    expect(within(block).getByText("Laboratório Municipal · laudo do laboratório municipal")).not.toBeNull();
    expect(within(block).getByText("laudo retificado — substitui o anterior")).not.toBeNull();
  });

  it("substituído: marca, data e título próprio", () => {
    render(<ExamResultBlock superseded result={examResult({ source: "lab", superseded_at: "2026-10-15T11:00:00-03:00" })} onExpired={vi.fn()} />);
    const block = screen.getByRole("region", { name: "Resultado substituído" });
    expect(within(block).getByText("substituído")).not.toBeNull();
    expect(within(block).getByText(/deixou de valer em 15\/10\/2026/)).not.toBeNull();
  });
});
```

```tsx
// src/modules/exams/ExamOrderDetail.lab.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getExamOrder: vi.fn(), listEnabledProviders: vi.fn(), setGalRequisition: vi.fn() };
});

import * as api from "../../lib/api";
import { ExamOrderDetail } from "./ExamOrderDetail";
import { renderWithProviders } from "../../test/campaignFixtures";
import { examOrder, examResult, examUser } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getExamOrder, api.listEnabledProviders, api.setGalRequisition ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "health_professional" ]));
  mocked(api.listEnabledProviders).mockResolvedValue([]);
  mocked(api.setGalRequisition).mockResolvedValue(undefined);
});

describe("ExamOrderDetail — 21b", () => {
  it("origem lab_reception e laudo substituído só como histórico", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder({
      origin: "lab_reception", status: "resulted",
      result: examResult({ id: "er2", source: "lab", replaces_result_id: "er1" }),
      superseded_results: [ examResult({ id: "er1", source: "lab", superseded_at: "2026-10-15T11:00:00-03:00" }) ]
    }));
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    expect(await screen.findByText("recepção do laboratório")).not.toBeNull();
    const old = screen.getByRole("region", { name: "Laudos substituídos" });
    expect(within(old).getByText("o cidadão vê só o laudo vigente")).not.toBeNull();
    expect(within(old).getByRole("region", { name: "Resultado substituído" })).not.toBeNull();
  });

  it("GAL: mostra o número e a unidade registra", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "scheduled", gal_requisition: null }));
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Registrar requisição do GAL" }));
    fireEvent.change(screen.getByLabelText("Número da requisição no GAL"), { target: { value: "GAL 2026/77" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar número do GAL" }));
    expect(screen.getByRole("alert").textContent).toBe("número do GAL inválido — use letras, números e hífen (até 30)");
    fireEvent.change(screen.getByLabelText("Número da requisição no GAL"), { target: { value: "GAL-2026-77" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar número do GAL" }));
    await waitFor(() => expect(api.setGalRequisition).toHaveBeenCalledWith("eo1", "GAL-2026-77"));
    await waitFor(() => expect(api.getExamOrder).toHaveBeenCalledTimes(2));
  });

  it("api sem o campo GAL: nada de GAL na tela", async () => {
    const order = examOrder({ status: "scheduled" });
    delete (order as { gal_requisition?: unknown }).gal_requisition;
    mocked(api.getExamOrder).mockResolvedValue(order);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    await screen.findByText("Hemograma completo");
    expect(screen.queryByRole("button", { name: "Registrar requisição do GAL" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd $W && npx vitest run src/modules/exams/ExamResultBlock.lab.test.tsx src/modules/exams/ExamOrderDetail.lab.test.tsx`
Expected: FAIL — rótulos crus, sem a seção de substituídos e sem GAL.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/exams.ts`:

```ts
export const ORIGIN_LABEL: Record<string, string> = {
  consultation: "consulta", reception_guide: "guia da recepção", pec: "PEC", lab_reception: "recepção do laboratório"
};
export const SOURCE_LABEL: Record<string, string> = {
  unit: "registrado pela unidade", provider: "registrado pelo prestador", fhir: "recebido do laboratório (FHIR)",
  consultation: "registrado na consulta", lab: "laudo do laboratório municipal"
};
```

```ts
// em ERRORS:
  invalid_gal_requisition: "número do GAL inválido — use letras, números e hífen (até 30)",
  provider_uses_municipal_lab: "este prestador usa o laboratório municipal — o resultado sai pela liberação do laudo",
```

```ts
// F-21.24: número da requisição no GAL do LACEN (só registro).
export function galProblem(value: string): string | null {
  return /^[A-Za-z0-9-]{1,30}$/.test(value.trim()) ? null : ERRORS.invalid_gal_requisition;
}
```

(`galProblem` fica depois da constante `ERRORS` no arquivo.)

Em `src/modules/exams/ExamResultBlock.tsx`:

```tsx
export function ExamResultBlock({ result, onExpired, critical, superseded = false }: {
  result: ExamResult; onExpired(): void; critical?: boolean; superseded?: boolean;
}) {
```

```tsx
    <section aria-label={superseded ? "Resultado substituído" : "Resultado"} style={box}>
      <div style={row}>
        <strong>{`Resultado liberado em ${fmtDay(result.released_on)}`}</strong>
        {superseded && <Tag tone="neutral">substituído</Tag>}
        {isCritical && <Tag tone="down">crítico</Tag>}
        {result.origin === "external" && <Tag>externo</Tag>}
      </div>
      <p style={muted}>{`${result.laboratory_name} · ${SOURCE_LABEL[result.source] ?? result.source}`}</p>
      {superseded && result.superseded_at && <p style={muted}>{`deixou de valer em ${fmtDateTime(result.superseded_at)}`}</p>}
      {!superseded && result.replaces_result_id && <p style={muted}>laudo retificado — substitui o anterior</p>}
```

(o resto do bloco não muda).

Crie `src/modules/exams/GalRequisitionForm.tsx`:

```tsx
// src/modules/exams/GalRequisitionForm.tsx
// F-21.24: o número da requisição no GAL (LACEN), só como registro; o laudo do
// LACEN entra como resultado externo do 21a.
import { useState } from "react";
import { setGalRequisition } from "../../lib/api";
import { examError, galProblem } from "../../lib/exams";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, row } from "./styles";

export function GalRequisitionForm({ orderId, current, onDone, onCancel }: {
  orderId: string; current: string | null; onDone(): void; onCancel(): void;
}) {
  const [ value, setValue ] = useState(current ?? "");
  const [ error, setError ] = useState<string | null>(null);
  async function save() {
    const problem = galProblem(value);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await setGalRequisition(orderId, value.trim());
      onDone();
    } catch (err) {
      setError(examError(err));
    }
  }
  return (
    <div style={box}>
      <label style={fieldLabel}>Número da requisição no GAL<input value={value} autoComplete="off" style={inputStyle} onChange={(e) => setValue(e.target.value)} /></label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton onClick={save}>Salvar número do GAL</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </div>
  );
}
```

Em `src/modules/exams/ExamOrderDetail.tsx`:

```tsx
// imports
import { GalRequisitionForm } from "./GalRequisitionForm";
```

```tsx
// type Mode: acrescente "gal"
type Mode = null | "schedule" | "result" | "cancel" | "acknowledge" | "gal";
```

```tsx
// na grade de KeyValue, depois de "Prestador":
        {order.gal_requisition && <KeyValue k="Requisição GAL" v={order.gal_requisition} />}
```

```tsx
// depois do bloco {order.result && (…)}:
      {order.superseded_results && order.superseded_results.length > 0 && (
        <section aria-label="Laudos substituídos" style={{ display: "flex", flexDirection: "column", gap: 8 }}>
          <p style={muted}>o cidadão vê só o laudo vigente</p>
          {order.superseded_results.map((r) => (
            <ExamResultBlock key={r.id} result={r} superseded onExpired={() => void query.refetch()} />
          ))}
        </section>
      )}
```

```tsx
// na fileira de botões (mode === null && !rereading), depois do "Registrar resultado":
          {order.gal_requisition !== undefined && (roles.reception || roles.professional) && !CLOSED.has(order.status) && (
            <button type="button" style={secondaryButtonStyle} onClick={() => setMode("gal")}>
              {order.gal_requisition ? "Trocar requisição do GAL" : "Registrar requisição do GAL"}
            </button>
          )}
```

```tsx
// junto dos outros formulários por modo:
      {mode === "gal" && (
        <GalRequisitionForm orderId={order.id} current={order.gal_requisition ?? null} onCancel={() => setMode(null)}
          onDone={() => { setMode(null); void refreshExams(queryClient); void query.refetch(); }} />
      )}
```

com, no topo do arquivo, `const CLOSED = new Set([ "denied", "cancelled" ]);` e, no corpo, a linha `const actions = examActions(order, examRoles(user), user?.id ?? null);` (linha 110 em `origin/main`) trocada por:

```tsx
  const roles = examRoles(user);
  const actions = examActions(order, roles, user?.id ?? null);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd $W && npx vitest run src/modules/exams src/lib/exams.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos do `ExamOrderDetail` e do `ExamResultBlock` seguem verdes: sem `superseded_results` nem `gal_requisition`, a tela é a de antes).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/exams.ts src/modules/exams/ExamResultBlock.tsx src/modules/exams/ExamOrderDetail.tsx src/modules/exams/GalRequisitionForm.tsx \
  src/modules/exams/ExamResultBlock.lab.test.tsx src/modules/exams/ExamOrderDetail.lab.test.tsx
$G commit -m "feat: show municipal lab results, superseded reports and GAL number on the exam

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 20: Suíte, build, revisão e prova no navegador (api real na 3042 e `fake-middleware`)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–19 na branch `feat/mod-21b-lab`; o api do 21b na **3042** com a semente do laboratório e o serviço `fake-middleware`.
- Produces: branch pronta para merge (depois do merge do api do 21b); evidências em `evidencias/2026-10-15-modulo-21b/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **27 arquivos de teste novos** (`api.lab`, `lab`, `modules.lab`, `LabCriticalNotice`, `ScanField`, `openLabPdf`, `RequisitionForm`, `LabRequisitions`, `LabRequisitionDetail`, `LabCollection`, `LabShipments`, `LabReception`, `ResultEntryForm`, `LabEntry`, `LabWorklist`, `LabRelease`, `RectifyForm`, `signature.lab`, `SignaturePending.lab`, `labAdmin`, `VersionForm`, `LabAccessPanels`, `ProviderForm.lab`, `UnitForm.lab`, `LabReports`, `ExamResultBlock.lab`, `ExamOrderDetail.lab`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/SegmentedControl.tsx src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css
```

Expected: `status` vazio e nenhum componente comum, peça do shell, tema ou `responsive.css` mudado.

- [ ] **Step 2: Nada vaza e nada relê sozinho**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage" src/lib/lab.ts src/lib/labAdmin.ts src/modules/lab src/modules/labAdmin src/modules/Lab*.tsx src/hooks/useLabCriticalCount.ts | grep -v "\.test\."
grep -rn "refetchInterval\|refetchOnWindowFocus" src/modules/lab src/modules/labAdmin src/modules/Lab*.tsx src/hooks/useLabCriticalCount.ts | grep -v "\.test\."
sed -n '/─── Laboratório municipal (módulo 21b/,$p' src/lib/api.ts | grep -n "p.set(\|p.append(\|new URLSearchParams({"
```

Expected: o primeiro sem saída; o segundo só com `useLabCriticalCount.ts` (a rota `/count`, sem trilha); o terceiro só com `status[]`, `provider_id`, `origin_unit_id`, `number`, `barcode`, `before`, `limit`, `sample_ids[]`, `patient_id`, `sector`, `bench`, `urgent`, `date`, `pending`, `item_id`, `month` — nenhum `cpf`, `mother`, `value`, `reason`, `note`, `totp` ou `contact`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§2, §4–§9, §11), o ADR 0037, o contrato (§1–§6, §9, §11) e as Divergências deste plano. Pontos de atenção:
- sem `municipal_lab`, menu, prestador, unidade, exame e assinatura são exatamente os de antes (testes antigos verdes; só as duas contagens de `modules.test.ts` e a lista de papéis de `team.test.ts` mudaram);
- nenhuma consulta com `refetchInterval` além da contagem de críticos; nenhuma rota que grava trilha relida sozinha;
- leitura de código de barras: Enter encerra; fila sem perda; duplicata ignorada; nunca rouba o foco;
- TOTP só no estado do formulário, apagado antes de cada envio; token do middleware só no estado do painel;
- 409 `invalid_transition` sempre explica e relê; 401 `mfa_required` apaga o código; CRM sem a opção avançada;
- laudo substituído só na visão clínica, com a frase "o cidadão vê só o laudo vigente";
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 21b na **3042** (plano do api: servidor do worktree `apps/api/.claude/mod21b` no `api-dev`, semente com o "Laboratório Municipal" `uses_municipal_lab`, uma UBS `collects_samples`, exames adotados — glicose, hemograma, colesterol com não-HDL calculado —, motivos de rejeição, um liberador CRF (RT), um liberador CRM com certificado no PSC simulado e o token do middleware de dev; o serviço `fake-middleware` no compose). Ligue em Curitiba, pelo maintenance, `exam_tracking`, `municipal_lab`, `lab_hl7_inbound`, `fake_middleware`, `digital_signature` e o PSC simulado (`signature_psc_mock`). O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo: o do worktree é do macOS e não roda no Alpine):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod21b --network rota-saude_default -p 5194:5194 \
  -v $R/.worktrees/dashboard-mod21b:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3042 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5194 --host 0.0.0.0"
docker logs -f dashboard-mod21b   # espere o "ready"; Ctrl+C sai do log, o container segue
docker compose up -d fake-middleware
```

(Se o plano do api subir o servidor do 21b num container próprio, use o nome dele no lugar de `api`. O `fake-middleware` usa o `FAKE_MIDDLEWARE_TOKEN` de dev do plano do api, já gravado como hash no laboratório da semente: **não** gere token novo para o "Laboratório Municipal" na prova do admin — o novo invalida o da semente e o `fake-middleware` passa a levar `AR`. Gere o token num segundo laboratório criado na prova. Se precisar trocar o do `fake-middleware`, ponha o novo no ambiente do serviço e recrie com `docker compose up -d --force-recreate fake-middleware` — variável nova no arquivo não chega a container já rodando; confira com `docker exec <container do fake-middleware> printenv FAKE_MIDDLEWARE_TOKEN | wc -c`. As contas da semente são as que o plano do api fixar para equipe do laboratório, liberador CRF, liberador CRM, enfermagem da UBS e admin.)

Abra `http://curitiba.localhost:5194/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Para a leitura, use um leitor de código de barras USB de verdade se houver; senão, digite o código no campo e tecle Enter (é exatamente o que o leitor faz). Confira com screenshot (salve em `evidencias/2026-10-15-modulo-21b/dashboard/`):
1. **Admin**: Exames → Catálogo e prestadores — o "Laboratório Municipal" com "Usa o laboratório municipal" marcado; Atendimento → Unidades — a UBS com "Coleta amostras" marcado. Laboratório → Exames e liberadores: adotar um exame do catálogo-base (step-up); criar a versão 2 da glicose com uma faixa feminina de 60 anos ou mais; incluir um motivo de rejeição; ver os liberadores (RT marcado); num laboratório de prova, **Gerar token do middleware** → aparece uma vez → "Já guardei" → some. Equipe → convite com "Equipe do laboratório" e "Liberador de laudos".
2. **Enfermagem da UBS que coleta**: Laboratório → Requisições → **Nova requisição** — CPF da semente; os exames do 21a do paciente aparecem como candidatos; marcar dois, sexo, nome da mãe → requisição aberta; **Comprovante** (data prevista) e **Imprimir etiquetas** — PDF em tamanho de etiqueta de tubo, com Code 128, nome, nascimento, material e número; leia uma etiqueta impressa com o leitor no campo de Coleta (se houver leitor).
3. **Coleta**: ler as etiquetas → "coleta registrada · requisição …"; na tela do exame (21a) o exame vai a "coletado". Ler a mesma etiqueta duas vezes seguidas → "repetida".
4. **Remessa**: ler as etiquetas, temperatura "4,5", **Enviar remessa**.
5. **Equipe do laboratório**: Remessas → **Conferir** — ler só uma das etiquetas → **Concluir conferência** com a confirmação de "não chegou"; na UBS, "Amostras que não chegaram ao laboratório" mostra a outra.
6. **Recebimento**: aceitar uma amostra; **Rejeitar** outra com motivo → "recoleta … pedida à UBS de origem"; na UBS, a recoleta aparece em Remessas com **Etiqueta**; coletar e remeter a recoleta; aceitar com restrição ("lipêmica").
7. **Mapa de trabalho**: hoje, só pendentes; filtrar por setor e urgência; **Baixar PDF do mapa**.
8. **Digitação**: pelo mapa (**Digitar**) e pela leitura — "13,5" vira 13,5 salvo; um valor fora do limite é bloqueado com a frase; o colesterol mostra o não-HDL "calculado ao salvar" e, depois de salvar, o valor calculado.
9. **HL7**: em até um ciclo do `fake-middleware`, amostras recebidas ganham valores "aparelho — conferir", com um crítico e uma repetição.
10. **Liberador CRF**: Liberação de laudos — a repetição: **Usar …**; o crítico sem aviso pede confirmação; **Registrar o aviso antes**; liberar com **assinatura avançada** (código do autenticador digitado pelo usuário) → "Laudo liberado com assinatura eletrônica avançada."; o contador "sem aviso" do menu e o aviso do Início com um crítico liberado sem aviso numa segunda requisição; **Registrar aviso** em "Críticos sem aviso" → o contador some no minuto seguinte.
11. **Liberador CRM**: só ICP na tela; liberar → "Laudo enviado para assinatura ICP" → **Ir para Pendentes de assinatura** → "laudo laboratorial" na lista, sem "Voltar ao papel" → **Assinar todas** com o PSC simulado → o laudo vira liberado.
12. **No 21a**: na tela do exame (profissional que pediu), o resultado "laudo do laboratório municipal" com os valores e **Abrir laudo** (PDF com os 14 itens do art. 167, tipo de assinatura e QR de conferência); a ciência funciona como no 21a; no wpda (cidadão da semente), o laudo segue a política do exame.
13. **Retificação**: Liberação → **Laudo já liberado** pelo número → **Retificar** com motivo e um contato → laudo v2; no 21a, "laudo retificado — substitui o anterior" e "Laudos substituídos" com a v1 (só para o profissional), e o exame volta a "resultado registrado".
14. **Entrega e situação**: Requisições → situação pelo número e pela leitura da etiqueta; **Registrar entrega** com quem retirou.
15. **Relatórios** (admin e liberador): mês corrente; produção com quantidade e valor SIGTAP, rejeições (o motivo usado), prazo; **Baixar CSV** dos três — abra e confira que não há dado de paciente.
16. **GAL**: num exame de prestador LACEN, **Registrar requisição do GAL** → o número aparece na tela.
17. **Sem polling**: com a aba de rede do navegador aberta na Liberação por 2 minutos, só `/lab/criticals/unnotified/count` se repete; nenhuma chamada a `/lab/releases`, `/lab/entry` ou `/lab/criticals/unnotified` sem ação da pessoa.
18. **Telefone** (largura 375): Requisições, Recebimento, Mapa e Liberação viram cartões, sem rolagem lateral; o campo de leitura cabe na tela.
19. No maintenance, desligar `municipal_lab` e recarregar: sem o grupo "Laboratório", sem as marcas no prestador e na unidade; Exames continua como antes.

Depois: `docker rm -f dashboard-mod21b`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

Para cada card F-21.14..F-21.24: se a parte do api dele já está Done, mova para **Done** (o mesmo `gh project item-edit` da Task 0, com o id da opção Done `98236657`); senão deixe em In Progress e anote. **Pare.** O merge do dashboard só vem depois do merge do api do 21b, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard → maintenance (contrato §13). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21b-lab
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod21b
/opt/homebrew/bin/git branch -d feat/mod-21b-lab
```

---

## Divergências propostas ao contrato

1. **D1 — Contexto do laboratório.** O contrato não diz como a tela sabe em que laboratório a pessoa atua nem o conselho de quem libera (a liberação recusa CRM sem ICP, mas a tela precisa saber antes para não oferecer a opção). Proposta: `GET /lab/context` → `{ "labs": [ { "id", "name" } ], "releaser_of": [ <provider_id> ], "council": "CRF"|"CRBM"|"CRM"|null }` — `lab_staff` e `municipal_admin`: todos os prestadores `uses_municipal_lab` ativos; `lab_releaser`: os da ligação de liberador; UBS (`health_professional`): os laboratórios municipais ativos. Sem dado clínico, sem trilha.
2. **D2 — Candidatos e busca do paciente.** `GET /lab/candidate_orders` → `{ "orders": [ { "id", "short_code", "sigtap_code", "label", "priority", "status", "requester_name" } ] }` (visão administrativa: sem CID, valores ou motivo). A busca pelo CPF reusa `POST /exams/patient_lookup` (A19 do 21a), que passa a aceitar `lab_staff` (hoje: recepção, profissional e admin).
3. **D3 — Exames do laboratório fora do admin.** O pedido em papel precisa dos exames ativos do laboratório; `/lab/admin/exams` é só do admin. Proposta: `GET /lab/exams?provider_id=` → `{ "exams": [ { "sigtap_code", "name", "material", "delivery_days" } ] }` para `lab_staff` e UBS que coleta.
4. **D4 — Remessa.** Faltam a leitura e a forma. Proposta: `GET /lab/shipments?provider_id=|origin_unit_id=&status[]=` → `{ "shipments": [<lab_shipment>] }` com `<lab_shipment>` = `{ "id", "origin_unit": { "id", "name" }, "provider": { "id", "name" }, "status", "temperature_celsius", "sent_at", "sent_by", "received_at", "received_by", "samples": [ { "id", "barcode", "material", "status", "requisition_number" } ] }` (também na resposta de `POST /lab/shipments`, `/send` e `/receive`). Recusas 422 da criação com o código da amostra no corpo: `{ "error": "sample_not_collected"|"sample_in_other_shipment"|"wrong_origin_unit", "barcode" }`, além de `shipment_empty` e `invalid_temperature`. A conferência só marca chegada/`not_received`; aceitar ou rejeitar é do `POST /lab/samples/receive`.
5. **D5 — O que volta para a UBS.** `GET /lab/recollections?origin_unit_id=` → `{ "recollections": [<lab_pending_sample>], "not_received": [<lab_pending_sample>] }`, com `<lab_pending_sample>` = `<lab_sample>` + `"requisition": { "id", "number", "patient_display_name" }` (o "alerta a UBS" da spec §5.3).
6. **D6 — Resposta do recebimento.** `POST /lab/samples/receive` → `{ "sample": <lab_sample> + "requisition_number", "recollection": <lab_sample>|null }`; 422 `reason_required` (rejeição sem motivo) e `restriction_note_required`. A coleta por código (`POST /lab/samples/collect`) devolve `<lab_sample>` + `requisition_number`.
7. **D7 — Digitação.** Falta a leitura do item para digitar. Proposta: `GET /lab/entry?item_id=|barcode=` → `{ "items": [ { "item_id", "requisition_id", "requisition_number", "barcode", "patient": { "display_name", "birth_date", "biological_sex" }, "sigtap_code", "label", "version", "status", "sample_status", "restriction_note", "analytes": [ { "key", "name", "unit", "value_type", "choices", "decimals", "reference": { "low", "high", "text" }|null, "entry_min", "entry_max", "critical_low", "critical_high", "formula" } ], "results": [<lab_result_row>] } ] }` — `reference` já é a faixa do sexo e da idade na data da coleta (a tela não escolhe faixa). `POST /lab/results` → `{ "results": [<lab_result_row>] }` (inclui os calculados); 422 `out_of_entry_range` traz `analyte_key`. `POST /lab/results` aceita item `released` (o valor novo vale só numa retificação).
8. **D8 — Mapa.** `GET /lab/worklists` → `{ "groups": [ { "sector", "bench"|null, "items": [ { "item_id", "requisition_id", "requisition_number", "barcode", "patient_display_name", "sigtap_code", "label", "urgent", "received_at", "status" } ] } ] }`.
9. **D9 — Fila de liberação e conferência.** `GET /lab/releases?provider_id=` → `{ "requisitions": [ { "id", "number", "patient": <lab_patient>, "urgent", "items": [ <lab_item> + "restriction_note", "results": [<lab_result_row>] ], "reports": [<lab_report_summary>] } ] }`; e `GET /lab/releases/:requisition_id` com a mesma forma para uma requisição em qualquer situação (conferência e retificação de laudo já liberado). `<lab_result_row>` ganha `notified` (bool) nas linhas `critical_*`.
10. **D10 — Críticos sem aviso.** `GET /lab/criticals/unnotified` → `{ "criticals": [ { "result_id", "requisition_id", "requisition_number", "patient_display_name", "analyte_name", "flag", "released_at" } ] }` (sem valor), do laboratório do liberador.
11. **D11 — Contador sem trilha.** `GET /lab/criticals/unnotified/count` → `{ "count" }`, sem dado clínico e sem `clinical_record.viewed` (molde da A46 do 21a). É a única rota que o dashboard relê sozinho (a cada minuto, para o menu e o aviso do Início).
12. **D12 — Admin: formas.** `GET /lab/admin/templates` → `{ "templates": [ { "id", "sigtap_code", "name", "sector", "material", "container", "suggested_method", "preparation", "analytes": [ … ] } ] }`; `GET /lab/admin/exams?provider_id=` → `{ "exams": [ { "id", "provider_id", "template_id", "sigtap_code", "name", "sector", "bench", "delivery_days", "active", "current_version": <version>|null } ] }`; `GET /lab/admin/exams/:id` acrescenta `versions: [ { "id", "version", "created_at", "created_by" } ]`. Adotar cria a **versão 1 copiada do catálogo-base** (o contrato não diz). Ajustar setor, bancada, prazo e situação: `PATCH /lab/admin/exams/:id { "sector", "bench", "delivery_days", "active" }` (não existe no contrato); componentes e referências só por versão nova.
13. **D13 — Relatórios: formas e CSV.** `production` → `{ "rows": [ { "origin_unit": { "id", "name" }, "sigtap_code", "exam_name", "sector", "releaser": { "id", "name" }|null, "quantity", "sigtap_value_cents" } ], "total_quantity", "total_value_cents" }`; `rejections` → `{ "rows": [ { "reason": { "id", "name" }, "origin_unit": { "id", "name" }, "count" } ] }`; `turnaround` → `{ "rows": [ { "sigtap_code", "exam_name", "on_time", "late", "pending" } ] }`. O contrato só põe `.csv` na produção; a tarefa pede CSV nos três: `/lab/reports/rejections.csv` e `/lab/reports/turnaround.csv` com os mesmos filtros. `provider_id` opcional (todos os laboratórios que a sessão vê).
14. **D14 — Motivos de rejeição.** O corpo do admin leva `provider_id` (`{ "provider_id", "name", "active" }`); `GET /lab/admin/rejection_reasons?provider_id=` → `{ "reasons": [ { "id", "provider_id", "name", "active" } ] }`. O recebimento (`lab_staff`) lê os ativos por `GET /lab/rejection_reasons?provider_id=` (mesma forma; o admin é só do `municipal_admin`).
15. **D15 — Unidade que coleta.** `collects_samples` sai em `GET /attendance/units` e `/attendance/units/all` (o `UnitPicker` e a lista de unidades) e entra pelo `POST /attendance/units/:id` existente (o contrato fala em `PATCH`; a rota real de edição é `POST`). Requisição ou remessa de unidade sem a marca → 422 `unit_does_not_collect`.
16. **D16 — Laudo substituído no 21a.** O §9 do contrato diz que o anterior fica `superseded` e só o profissional vê, sem a forma. Proposta: `<exam_order>.superseded_results: [<exam_result> + "superseded_at"]` (do mais novo ao mais antigo) só na visão clínica; `<exam_result>.replaces_result_id` no vigente. `exam-order-v1` acompanha antes da tag `lab-v1.0.0`.
17. **D17 — GAL.** A spec (§4, §11 F-21.24) tem `exam_orders.gal_requisition` com tela, mas o contrato não tem rota. Proposta: `POST /exams/orders/:id/gal_requisition { "gal_requisition" }` (unidade do exame; 422 `invalid_gal_requisition` — letras, números e hífen, até 30) e `gal_requisition` (string | null) no `<exam_order>` das duas visões.
18. **D18 — Conselho do liberador.** O contrato §3 diz que o perfil profissional ganha `council`, `council_uf`, `council_number`; o perfil do módulo 10 **já tem** `council`, `council_state` e `registration_number` (com `CRF`, `CRBM` e `CRM` na lista). Proposta: reusar esses campos (nada novo no perfil) e o api permitir perfil profissional para quem tem `lab_releaser` sem `health_professional`. `<lab_releaser>` (`GET /lab/admin/releasers` → `{ "releasers": [ { "id", "user_id", "name", "email", "provider_id", "council", "council_uf", "council_number", "technical_lead", "technical_deputy" } ] }`) traz os valores do perfil; trocar a marca de RT/substituto = tirar e incluir.
19. **D19 — Laudo ICP nas Pendentes.** O laudo `pending_signature` precisa aparecer em `GET /signature/pending` do liberador: `document_type: "lab_report"`, `consultation_id: null`, `patient_display_name`; `canSign` vale para `lab_releaser` com `digital_signature`. "Voltar ao papel" não se aplica a laudo (o api recusa com 422 `not_applicable`); se o ICP falhar, o laudo segue pendente para o próximo lote.
20. **D20 — Prestador no admin do 21a.** `GET /exams/admin/providers` passa a trazer `uses_municipal_lab` e `has_middleware_token` (o token nunca volta); `PATCH`/`POST` aceitam `uses_municipal_lab` só com `municipal_lab` ligado.

Alinhamento com o plano do api (escrito em paralelo): as D1–D20 precisam entrar no contrato e no plano do api antes da Task 20; as que só nomeiam forma (D4–D10, D12–D14) podem ser resolvidas pelo api escolhendo outra forma, desde que o contrato a fixe e este plano ajuste `src/lib/api.ts` e as fixtures.

## Self-review

- **Cobertura (spec §5, §7 e contrato §3–§6, §9):** requisição com candidatos do 21a e pedido em papel — Task 5; comprovante, etiquetas e reimpressão com trilha — Task 6; coleta por leitura (teclado + Enter) — Tasks 4 e 7; remessa na UBS (montar, temperatura, enviar), recoletas e não recebidas — Task 8; conferência da remessa no laboratório — Task 8; recebimento (aceitar, rejeitar com recoleta, aceitar com restrição) — Task 9; mapa com filtros e PDF — Task 11; digitação por mapa e por leitura, referência, limite, calculados e repetições — Tasks 10 e 11; fila de liberação, assinatura ICP ou avançada com TOTP, médico só ICP, críticos sem aviso e registro da comunicação — Task 12; ICP nas Pendentes — Task 14; retificação com motivo e contatos — Task 13; entrega e situação — Task 6; admin (adotar, versões com componentes e faixas por sexo e idade, motivos, liberadores com RT e substituto, token do middleware uma vez) — Tasks 15 e 16; marcas `uses_municipal_lab` e `collects_samples` — Task 17; relatórios de produção, rejeição e prazo com CSV — Task 18; no 21a, `source: lab`, laudo retificado/substituído só para o profissional, origem `lab_reception` e GAL — Task 19; menu, papéis e contador — Task 3; prova no navegador com api real na 3042 e `fake-middleware` — Task 20. Interruptores e catálogo-base no maintenance, perfil HL7 e `fake-middleware` são de outros planos; o wpda não muda.
- **Placeholders:** nenhum código faltando. Pontos que dependem de nome do outro plano estão marcados para conferência: as contas da semente e o nome do container do api (Task 20), e as formas propostas nas Divergências.
- **Tipos:** `LabRequisition`, `LabSample`, `LabEntryItem`, `LabReleaseRequisition`, `LabReportSummary`, `LabExam`/`LabExamVersion` (Task 1) são os usados nas Tasks 4–18; `LabScopeValue` (Task 6) é o das Tasks 7–12; `ScanField.onScan` devolve `Promise<ScanOutcome>` em todas as telas; `ResultEntryForm({ item, onSaved, onClose })` é o mesmo nas Tasks 10 e 11; `SignatureChoice` e `cityNowLocal` (Task 12) são reusados na Task 13; `providerBody(d, labOn)` (Task 17) mantém o 21a igual sem o 2º argumento; `updateUnit(…, collectsSamples?)` (Task 1) é o da Task 17; `ExamResultBlock.superseded` (Task 19) é opcional.
- **Review Focus:** 1 — Task 4 ("três leituras seguidas…", "o mesmo código duas vezes em 2 s…", "sem foco: avisa e não rouba o foco"); 2 — Task 12 ("409 invalid_transition na liberação…"); 3 — Task 12 ("assinatura avançada com código recusado…") e Task 13 ("código recusado: apaga e pede de novo…"); 4 — Task 2 ("fora de entry_min/entry_max bloqueia…", "vírgula vira ponto…") e Task 10 ("bloqueia fora do limite sem chamar o api", "422 out_of_entry_range: marca o componente e mantém os valores"); 5 — Task 16 ("token do middleware: aparece uma vez…").
