# Módulo 21a — Rastreamento do exame (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 21a** (`feat/mod-21a-exams`, porta de dev **3041**) com as rotas `/exams/*` do contrato e as rotas das Divergências D1–D9 e D14–D15 (fim deste plano), o interruptor `exam_tracking` na sessão, a semente com o prestador "Laboratório Simulado" e o serviço `fake-lab` no compose. Ele é **mergeado antes** deste (contrato §14: contracts → api → dashboard e wpda → maintenance). As Tasks 0–16 não dependem dele (os testes usam dublês de `src/lib/api`); a prova da Task 17 depende.

**Goal:** Lado dashboard do módulo 21a: status dos exames na consulta e na aba "Exames" do paciente; aviso e contador de "resultados aguardando ciência" no Início; tela de ciência com a fila da equipe; resultado externo; guia da recepção com anexo; agendar, registrar coleta e resultado pela unidade; painel "Exames atrasados" (etapa, prestador, exame); fila de regulação (autorizar escolhendo o prestador, negar com motivo); admin — catálogo de exames, prestadores (itens e prazos, token FHIR mostrado uma vez, convite do usuário) e relatório de conferência com CSV; portal do prestador (papel `exam_provider`, sem nada de prontuário); entrada `exam` na linha do tempo do profissional.

**Architecture:** O cliente HTTP ganha a seção "Exames" no fim de `src/lib/api.ts` (tipos do contrato §1–§7 e §9–§10 e as funções; na URL só ids, códigos, situações, etapas, mês e o cursor; arquivo sobe em JSON com base64 e sem o nome). As regras de tela ficam puras em `src/lib/exams.ts` (rótulos, quais botões cada papel vê, validação, frases das recusas, prazos herdados) e `src/lib/examsAdmin.ts` (formulários do admin). Os componentes ficam em `src/modules/exams/` (peças reusadas: `ExamOrderDetail`, `ResultForm`, `ScheduleForm`, `ReasonForm`, `UploadField`, `PatientLookup`, `ExamResultBlock`, `OverduePanel`…) e as telas novas em `src/modules/` (`ExamAcknowledgements`, `ExamsUnit`, `ExamsOverdue`, `ExamRegulation`, `ExamsAdmin`, `ExamReport`, `ProviderPortal`), registradas no grupo novo **"Exames"** do shell (`modules.ts`, `catalog.ts`, `App.tsx`). O prestador (`exam_provider`) tem o menu travado no portal e na Segurança. As telas existentes ganham o mínimo: `PatientPanel` (aba "Exames"), `ConsultationView` (situação de cada exame pedido), `ExamRequestsField`/`ConsultationEditor`/`ConsultationWorkspace` (prioridade), `lib/timeline.ts`/`TimelineDetail` (tipo `exam`). O dashboard nunca decide o que o api garante: quem lê, a situação, os prazos, o isolamento entre prestadores e a política de liberação são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5 (`useQuery`, `useInfiniteQuery`), Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`, proxy de dev), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-13-module-21a-exam-tracking-design.md` (§2 decisões, §3, §5, §6, §7, §9 telas, §11 F-IDs), `docs/adr/0036.md` e o contrato `docs/superpowers/plans/2026-10-13-module-21a-exams-contracts.md` (§1–§6, §9 por contraste, §10, §12). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim; nada foi resolvido em silêncio.

## Global Constraints

- Duas visões do exame (plano do api, Desvio 10): quem pediu, a leitura do 19a e a equipe recebem o resultado completo; unidade e regulador recebem o resultado sem valores, sem "crítico" e sem laudo. A tela mostra o que vier e nunca supõe campo ausente.
- Nomes no código (contrato §1): papéis `exam_regulator` e `exam_provider`; `priority` `routine` | `urgent`; `origin` `consultation` | `reception_guide` | `pec`; `status` `awaiting_authorization` | `denied` | `awaiting_schedule` | `scheduled` | `collected` | `resulted` | `acknowledged` | `cancelled`; etapas `authorize` | `schedule` | `collect` | `result` | `acknowledge`; `release_policy` `immediate` | `after_acknowledgement` | `never_citizen`; `kind` do prestador `municipal` | `accredited` | `consortium` | `lacen` | `imaging`; motivo do escalonamento `overdue` | `critical` | `unlinked_requester`. Código desconhecido aparece cru, nunca some.
- Prazos (spec §5; contrato §1): ordem prestador → catálogo → plataforma (autorizar 15, agendar 15, coletar 30, resultado 30, ciência 15 dias); `urgent` = `ceil(prazo / 2)`. O dashboard só **mostra** o prazo herdado (placeholder do admin); quem calcula o atraso é o api (`overdue`).
- Interruptor: `exam_tracking` na sessão liga tudo deste plano. Desligado, o dashboard é exatamente o de hoje (menu, consulta, painel do paciente, linha do tempo). 403 `feature_disabled` → `EXAMS_DISABLED` ("o rastreamento de exames está desligado nesta cidade"). O dashboard nunca escreve interruptor; `exam_fhir_inbound`, `exam_pec_bridge` e `fake_lab` ganham só rótulo em `features.ts`.
- F-21.11 (botão no PEC) não tem tela no dashboard: o formulário é HTML servido pelo api dentro da página do PEC (plano do api, Task 20), condicional à prova técnica da Task 0a do api (Divergência D12, corrigida pelo coordenador).
- Na URL vão só ids, códigos SIGTAP, situações, etapas, `YYYY-MM` e o cursor — nunca CPF, CNS, nome, motivo ou valor. CPF da busca do paciente vai no corpo de um POST (D2). Arquivo (guia, laudo) sobe **sem o nome do arquivo** (D1).
- Valor de resultado, CPF, CNS, PDF e token FHIR nunca em `console`, `localStorage`, `sessionStorage`, Analytics, mensagem de erro ou cache do React Query. Leituras com dado de saúde com `gcTime: 0` e `staleTime: 0`. O token FHIR vive só no estado do componente que o pediu e some com "Já guardei" ou ao sair (Review Focus 3).
- Toda escrita por cookie com `Content-Type: application/json`, inclusive o `DELETE` do token (`body: "{}"`) e o upload (JSON com base64, D1). Escrita do admin (`/exams/admin/*`) com step-up pelo `SensitiveAction`.
- O prestador (`exam_provider`) vê no menu só "Portal do prestador" e "Segurança"; hash para outra tela cai no Início (o roteador já faz isso). O portal não tem link, botão ou texto que leve a prontuário, linha do tempo ou consulta; mostra só nome, nascimento e os 3 últimos dígitos do CPF (contrato §2 `<provider_order>`).
- O cidadão nunca é atendido por este app: motivo de negação e a marca "crítico" aparecem para profissional, unidade e regulador (spec §6 só proíbe no wpda).
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova registrada em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--ok`, `--rule*`, `--sunken`, `--panel`, `--accent`; `Tag` com `tone`), sem `outline: "none"`, sem `textOverflow` sem `title`, sem `color: "#fff"`. Use os componentes existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `ActionButton`, `ConfirmButton`, `ActionMessage`, `SensitiveAction`, `SegmentedControl`, `UnitPicker`, `formStyles`); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date(NOW21))` e `vi.useRealTimers()` no `afterEach`; a sessão de teste é criada **depois** do `setSystemTime`.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards F-21.x deste app para **In Progress** (Task 0); cada card vai a **Done** quando a parte do dashboard dele termina **e** a do api já está Done (os cards são compartilhados).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod21` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-21a-exams` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod21`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos antes de cada commit: `npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. O jsdom não tem `URL.createObjectURL`: defina com `Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:x") })`.
- A prova (Task 17) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5192**, com proxy para `http://api:3041`.

## Review Focus

1. **Laudo aberto depois que o link assinado venceu (5 min).** A pessoa deixa o exame aberto, volta depois e clica "Abrir laudo": o api recusa o `pdf_url` velho. Espera-se a frase "o link do laudo expirou — abra de novo" e a releitura do exame (novo `pdf_url`), nunca uma janela em branco nem o PDF de outro exame. Teste: Task 5, "laudo com link vencido: avisa e relê o exame".
2. **Duas pessoas agem sobre o mesmo exame.** O prestador registra a coleta enquanto a unidade agenda; dois reguladores decidem a mesma guia. O segundo recebe 409 `invalid_transition`: a tela explica, fecha o formulário e relê — sem botão velho na tela nem segunda tentativa. Testes: Task 5, "409 invalid_transition: explica, fecha o formulário e relê"; Task 11, "outro regulador decidiu antes: fecha a decisão, avisa e relê a fila".
3. **Token FHIR mostrado uma vez.** Depois de "Já guardei" (ou ao sair da tela) o token não está na tela nem no cache do React Query; "Trocar" gera outro e o anterior deixa de valer (o api garante; a tela diz). Teste: Task 13, "token FHIR: aparece uma vez, some com Já guardei e nunca vai ao cache".
4. **O usuário do prestador tenta ir além do portal.** Menu com só o portal e a Segurança, mesmo com o rastreamento desligado (aí só a Segurança); a lista do portal não tem nome de tela clínica nem CPF inteiro. Testes: Task 3, "o prestador vê só o portal e a Segurança" e "prestador com o rastreamento desligado vê só a Segurança"; Task 15, "lista as guias só com nome, nascimento e o final do CPF".
5. **Arquivo errado ou envio que falha no meio.** Word, imagem de 12 MB, arquivo vazio ou 422 `invalid_file` do api: explica, não guarda `upload_id` e o formulário não envia; o nome do arquivo (que pode ter o nome do paciente) nunca vai ao api. Testes: Task 1, "upload: JSON com o tipo e o base64, sem o nome do arquivo"; Task 4, "recusa tipo, tamanho e vazio sem chamar o api" e "422 invalid_file: explica e não guarda o anexo"; Task 9, "guia sem anexo não é enviada".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.exams.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/examFixtures.ts`, `vite.config.ts` | tipos e cliente HTTP dos exames; interruptores; dados de teste; proxy `/exams` | 1 |
| `src/lib/exams.ts`, `src/lib/exams.test.ts`, `src/hooks/useExamOrderPages.ts` | regras de tela; páginas com cursor | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.exams.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/App.navigation.test.tsx`, `src/hooks/useExamPending.ts`, `src/modules/exams/AcknowledgementNotice.tsx` (+ teste), `src/modules/exams/styles.ts` | grupo "Exames", trava do prestador, contador e aviso no Início | 3 |
| `src/modules/exams/UploadField.tsx`, `ResultForm.tsx`, `ScheduleForm.tsx`, `ReasonForm.tsx` (+ testes) | anexo, resultado, agendamento, motivo | 4 |
| `src/modules/exams/ExamOrderDetail.tsx`, `ExamResultBlock.tsx`, `openExamPdf.ts` (+ teste) | o exame: andamento, resultado, laudo e ações | 5 |
| `src/lib/consultation.ts`, `src/lib/consultation.priority.test.ts`, `src/modules/consultation/ExamRequestsField.tsx`, `ConsultationEditor.tsx`, `ConsultationWorkspace.tsx`, `ConsultationView.tsx`, `src/modules/exams/ConsultationExamStatus.tsx` (+ testes) | prioridade no pedido; situação na consulta | 6 |
| `src/modules/consultation/PatientPanel.tsx`, `src/modules/exams/PatientExams.tsx`, `ExternalResultForm.tsx` (+ testes) | aba "Exames"; resultado externo | 7 |
| `src/modules/ExamAcknowledgements.tsx` (+ teste), `src/App.tsx` | ciência, negados e fila da equipe | 8 |
| `src/modules/ExamsUnit.tsx`, `src/modules/exams/PatientLookup.tsx`, `ReceptionGuideForm.tsx`, `UnitExamList.tsx` (+ teste), `src/App.tsx` | guia da recepção, lista da unidade, resultado externo da unidade | 9 |
| `src/modules/exams/OverduePanel.tsx`, `src/modules/ExamsOverdue.tsx` (+ teste), `src/modules/ExamsUnit.tsx`, `src/App.tsx` | painel de atrasos (unidade e coordenação) | 10 |
| `src/modules/ExamRegulation.tsx` (+ teste), `src/App.tsx` | fila de regulação | 11 |
| `src/lib/examsAdmin.ts` (+ teste), `src/modules/ExamsAdmin.tsx`, `src/modules/examsAdmin/CatalogPanel.tsx`, `CatalogItemForm.tsx` (+ teste), `src/App.tsx` | catálogo de exames | 12 |
| `src/lib/examsAdmin.ts`, `src/modules/examsAdmin/ProvidersPanel.tsx`, `ProviderForm.tsx`, `ProviderAccess.tsx` (+ teste), `src/modules/ExamsAdmin.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts` | prestadores, token FHIR, convite; papel `exam_regulator` no convite | 13 |
| `src/modules/ExamReport.tsx` (+ teste), `src/App.tsx` | relatório de conferência e CSV | 14 |
| `src/modules/ProviderPortal.tsx`, `src/modules/exams/ProviderOrderPanel.tsx` (+ teste), `src/App.tsx` | portal do prestador | 15 |
| `src/lib/api.ts`, `src/lib/timeline.ts`, `src/lib/timeline.exam.test.ts`, `src/modules/timeline/TimelineDetail.tsx`, `src/modules/timeline/TimelineDetail.exam.test.tsx` | entrada `exam` na linha do tempo | 16 |
| — | suíte, build, revisão, prova no navegador | 17 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir (CPF na URL, nome do arquivo no upload) (Task 1); regras puras com tabela de casos (Tasks 2 e 12); peças de formulário com eventos reais (`fireEvent.change` com `File`) e o api dublado (Task 4); cada tela pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel, cobrindo o caminho feliz, cada recusa que muda a tela (409 `invalid_transition`, 422 de validação, 404, `feature_disabled`, link vencido) e o papel sem acesso (Tasks 5–16); o menu por papel e por interruptor como função pura (Task 3). A prova final é no navegador, contra o api real na 3041 com o `fake-lab` ligado (Task 17).

---

### Task 0: Board, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard; o board #1 (`rotasaude`).
- Produces: cards F-21.2, .3, .4, .5, .6, .8, .9, .12 em In Progress; worktree `$W` na branch `feat/mod-21a-exams`; a base anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json
gh project field-list 1 --owner rotasaude --format json
```

Na saída, ache os itens de título `F-21.2`, `F-21.3`, `F-21.4`, `F-21.5`, `F-21.6`, `F-21.8`, `F-21.9` e `F-21.12` (o `id` `PVTI_…`) e o id da opção **In Progress** do campo Status (`PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`). Para cada card que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id PVTI_xxx --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress. Se algum F-21.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário). F-21.13 (fornecedor da linha do tempo) é do api; a Task 16 só mostra o tipo novo.

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n "export interface ExamRequest {\|^export type TimelineKind =\|^async function binaryFetch\|^const postProfessional\|export function reassignDraft"
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "patient_timeline";\|patient_timeline: "Linha do tempo do paciente"'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '"sncr" | "sncr-overview";\|{ label: "Comunicação", items: \[\|const canClinicalRecord =\|if (item.id === "production") return canProduction;\|export function withPendingCount'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n "export function shellGroups\|const pending = usePendingSignatures\|if (route.kind === \"home\") content\|case \"clinical-documents-admin\""
$G -C $R/apps/dashboard show origin/main:src/lib/consultation.ts | grep -n "^function examPayload" -A3
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/PatientPanel.tsx | grep -n 'useState<"summary" | "timeline">\|const onSummary =\|{timeline && (\|{timeline && timelineOpened'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationView.tsx | grep -n '{c.exam_requests.length > 0 && ('
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n "const timelineOn =\|<ConsultationEditor key=\|timeline={timelineOn"
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '{ key: "campaign", label: "Campanha" }\|unread: "aviso não lido"\|export type DetailType\|const OPENABLE'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '{ role: "municipal_admin", label: "Administrador municipal" }'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod21 -b feat/mod-21a-exams origin/main
cd $R/.worktrees/dashboard-mod21 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 17 espera a base mais **23 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 21a** (só informativo; as Tasks 1–16 não dependem dele)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-21a-exams
docker exec -w /rails/.claude/mod21 api-dev bin/rails routes 2>/dev/null | grep -E "exams/" | head -40
```

Expected: a branch e as rotas `/exams/*`. Compare com o contrato e com as Divergências deste plano; diferença vira nota para a Task 17.

---

### Task 1: Tipos, cliente HTTP, interruptores, dados de teste e proxy

**Files:**
- Modify: `src/lib/api.ts` (interface `ExamRequest`; seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/examFixtures.ts`
- Test: `src/lib/api.exams.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `binaryFetch`, `postProfessional`, `ApiError`, `SignatureFile`, `Invitation`, `SessionUser` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`): os tipos `ExamPriority`, `ExamOrigin`, `ExamStatus`, `ExamStage`, `ExamEventKind`, `ExamReleasePolicy`, `ExamProviderKind`, `ExamDeadlines`, `ExamResultField`, `ExamResultValue`, `ExamResult`, `ExternalExamResult`, `ExamEvent`, `ExamOverdue`, `ExamOrder`, `ExamOrderPage`, `ExamOrderQuery`, `ExamPatient`, `ExamCatalogEntry`, `ExamProviderRef`, `ExamRequester`, `ReceptionOrderInput`, `ScheduleInput`, `ScheduleWhen`, `ResultInput`, `ExternalResultInput`, `PendingAcknowledgements`, `ExamCatalogItem`, `ExamCatalogItemInput`, `ExamProviderItem`, `ExamProvider`, `ExamProviderInput`, `ExamReportRow`, `ProviderOrder`, `ProviderOrderPage`, `ProviderOrderQuery`; as constantes `EXAM_PAGE_SIZE = 50`, `EXAM_UPLOAD_MAX_BYTES`, `EXAM_UPLOAD_TYPES`; e as funções (assinaturas exatas no Step 3). `ExamRequest` ganha `priority?: ExamPriority`. `FeatureKey` ganha `exam_tracking`, `exam_fhir_inbound`, `exam_pec_bridge`, `fake_lab`.
- Produces (em `src/test/examFixtures.ts`): `NOW21`, `TODAY21`, `VALID_CPF`, `VALID_CNS`, `UNIT21`, `HEMOGRAM_FIELDS`, `HEMOGRAM_ENTRY`, `HBA1C_ENTRY`, `examUser(roles, over)`, `examResult(over)`, `examOrder(over)`, `examCatalogItem(over)`, `examProvider(over)`, `providerOrder(over)`, `pdfFile(name)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.exams.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  acknowledgeExam, fetchExamReportCsv, listExamOrders, listPendingAcknowledgements, listProviderOrders, lookupExamPatient,
  providerSchedule, revokeFhirToken, uploadExamFile
} from "./api";

function ok(body: unknown, headers: Record<string, string> = {}): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json", ...headers } }));
}

describe("cliente de exames (contrato §3–§6)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => {
    fetchMock = vi.fn(() => ok({ orders: [], next_before: null }));
    vi.stubGlobal("fetch", fetchMock);
  });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => fetchMock.mock.calls[i][1] as RequestInit;
  const headersOf = (i = 0) => initOf(i).headers as Record<string, string>;

  it("lista com status[] repetido, atraso, etapa, prestador, exame, cursor e limit", async () => {
    await listExamOrders({ unitId: "u1", statuses: [ "scheduled", "collected" ], overdue: true, stage: "collect",
      providerId: "pv1", sigtapCode: "0202020380", before: "c1" });
    const url = urlOf();
    expect(url.pathname).toBe("/exams/orders");
    expect(url.searchParams.get("unit_id")).toBe("u1");
    expect(url.searchParams.getAll("status[]")).toEqual([ "scheduled", "collected" ]);
    expect(url.searchParams.get("overdue")).toBe("true");
    expect(url.searchParams.get("stage")).toBe("collect");
    expect(url.searchParams.get("provider_id")).toBe("pv1");
    expect(url.searchParams.get("sigtap_code")).toBe("0202020380");
    expect(url.searchParams.get("before")).toBe("c1");
    expect(url.searchParams.get("limit")).toBe("50");
    expect(initOf().method).toBeUndefined();
    expect(initOf().credentials).toBe("include");
  });

  it("coordenação sem unidade: nenhum unit_id nem patient_id (Divergência D15)", async () => {
    await listExamOrders({ overdue: true, before: null });
    expect([ ...urlOf().searchParams.keys() ].sort()).toEqual([ "limit", "overdue" ]);
  });

  it("o CPF da busca do paciente vai no corpo, nunca na URL (D2)", async () => {
    fetchMock.mockImplementation(() => ok({ patient: { id: "pa1", display_name: "Joana Lima", cpf_masked: "***.982.247-**" } }));
    expect((await lookupExamPatient("52998224725")).id).toBe("pa1");
    expect(String(fetchMock.mock.calls[0][0])).toBe("/exams/patient_lookup");
    expect(initOf().method).toBe("POST");
    expect(JSON.parse(String(initOf().body))).toEqual({ cpf: "52998224725" });
    expect(headersOf()["Content-Type"]).toBe("application/json");
  });

  it("upload: JSON com o tipo e o base64, sem o nome do arquivo (D1)", async () => {
    fetchMock.mockImplementation(() => ok({ upload_id: "up1" }));
    const file = new File([ "%PDF-1.4 exemplo" ], "laudo Joana Lima.pdf", { type: "application/pdf" });
    expect(await uploadExamFile(file)).toBe("up1");
    expect(String(fetchMock.mock.calls[0][0])).toBe("/exams/uploads");
    expect(headersOf()["Content-Type"]).toBe("application/json");
    expect(JSON.parse(String(initOf().body))).toEqual({ content_type: "application/pdf", data: btoa("%PDF-1.4 exemplo") });
    expect(String(initOf().body)).not.toContain("Joana");
  });

  it("ciência sem motivo manda {}; com motivo manda o motivo", async () => {
    fetchMock.mockImplementation(() => ok({}));
    await acknowledgeExam("eo1");
    await acknowledgeExam("eo1", "Paciente avisado por telefone");
    expect(String(fetchMock.mock.calls[0][0])).toBe("/exams/orders/eo1/acknowledge");
    expect(JSON.parse(String(initOf(0).body))).toEqual({});
    expect(JSON.parse(String(initOf(1).body))).toEqual({ reason: "Paciente avisado por telefone" });
  });

  it("pendentes: resposta sem `denied` vira lista vazia (D7)", async () => {
    fetchMock.mockImplementation(() => ok({ orders: [] }));
    expect(await listPendingAcknowledgements()).toEqual({ orders: [], denied: [] });
    expect(urlOf().pathname).toBe("/exams/acknowledgements/pending");
  });

  it("revogar o token: DELETE com corpo JSON", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(new Response(null, { status: 204 })));
    await revokeFhirToken("pv1");
    expect(String(fetchMock.mock.calls[0][0])).toBe("/exams/admin/providers/pv1/fhir_token");
    expect(initOf().method).toBe("DELETE");
    expect(initOf().body).toBe("{}");
    expect(headersOf()["Content-Type"]).toBe("application/json");
  });

  it("portal do prestador: lê X-Overdue-Count; cabeçalho ausente ou torto vira 0", async () => {
    fetchMock.mockImplementationOnce(() => ok({ orders: [], next_before: null }, { "X-Overdue-Count": "3" }));
    expect((await listProviderOrders({ statuses: [ "scheduled" ], before: null })).overdue_count).toBe(3);
    expect(urlOf().pathname).toBe("/exams/provider/orders");
    expect(urlOf().searchParams.getAll("status[]")).toEqual([ "scheduled" ]);
    fetchMock.mockImplementationOnce(() => ok({ orders: [], next_before: null }, { "X-Overdue-Count": "muitos" }));
    expect((await listProviderOrders({ overdue: true, before: null })).overdue_count).toBe(0);
    expect(urlOf(1).searchParams.get("overdue")).toBe("true");
  });

  it("agendar pelo portal não manda provider_id (D9)", async () => {
    fetchMock.mockImplementation(() => ok({}));
    await providerSchedule("eo1", { scheduled_at: "2026-10-20T08:30:00-03:00", scheduled_place: "Posto de coleta" });
    expect(String(fetchMock.mock.calls[0][0])).toBe("/exams/provider/orders/eo1/schedule");
    expect(JSON.parse(String(initOf().body))).toEqual({ scheduled_at: "2026-10-20T08:30:00-03:00", scheduled_place: "Posto de coleta" });
  });

  it("CSV detalhado do relatório: mês, prestador e detail na URL; Accept text/csv", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(new Response("short_code,sigtap_code\n", {
      status: 200, headers: { "Content-Type": "text/csv", "Content-Disposition": 'attachment; filename="conferencia-2026-10.csv"' }
    })));
    const file = await fetchExamReportCsv("2026-10", "pv1", true);
    const url = urlOf();
    expect(url.pathname).toBe("/exams/admin/report.csv");
    expect(url.searchParams.get("month")).toBe("2026-10");
    expect(url.searchParams.get("provider_id")).toBe("pv1");
    expect(url.searchParams.get("detail")).toBe("true");
    expect(headersOf().Accept).toBe("text/csv");
    expect(file.filename).toBe("conferencia-2026-10.csv");
  });

  it("recusa vira ApiError com o código do corpo", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(new Response(JSON.stringify({ error: "invalid_transition", status: "scheduled" }), { status: 409 })));
    await expect(listExamOrders({ patientId: "pa1" })).rejects.toMatchObject({ status: 409, body: { error: "invalid_transition" } });
  });
});
```

Acrescente ao `describe` de `src/lib/features.test.ts`:

```ts
  it("módulo 21a: os quatro interruptores de exames têm rótulo", () => {
    expect(featureLabel("exam_tracking")).toBe("Rastreamento de exames");
    expect(featureLabel("exam_fhir_inbound")).toBe("Entrada de resultados por FHIR");
    expect(featureLabel("exam_pec_bridge")).toBe("Solicitar exames pelo PEC");
    expect(featureLabel("fake_lab")).toBe("Laboratório simulado (fora de produção)");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/api.exams.test.ts src/lib/features.test.ts`
Expected: FAIL — `listExamOrders` (e as demais) não são exportadas; os rótulos saem como a chave.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, troque a interface `ExamRequest` (seção do 19a):

```ts
// Módulo 21a (Divergência D10): prioridade do pedido; ausente = rotina.
export interface ExamRequest { sigtap_code: string; label: string; cid10_justification?: string | null; priority?: ExamPriority }
```

No **fim** de `src/lib/api.ts` (depois de `reassignDraft`), acrescente:

```ts
// ─── Exames (módulo 21a, ADR 0036; contrato 2026-10-13 §1–§10) ───────────────
// Na URL só ids, códigos SIGTAP, situações, etapas, mês e o cursor — CPF,
// nome, motivo e valores de resultado vão no corpo. Arquivo (guia, laudo)
// sobe em JSON com base64 e SEM o nome do arquivo (Divergência D1). Ler um
// exame grava a trilha no api; quem lê o quê é do api (regras do 19a e o
// isolamento do prestador).

const EXAMS_BASE = import.meta.env.VITE_EXAMS_BASE || "/exams";

export type ExamPriority = "routine" | "urgent";
export type ExamOrigin = "consultation" | "reception_guide" | "pec";
export type ExamStatus =
  | "awaiting_authorization" | "denied" | "awaiting_schedule" | "scheduled" | "collected" | "resulted" | "acknowledged" | "cancelled";
export type ExamStage = "authorize" | "schedule" | "collect" | "result" | "acknowledge";
export type ExamEventKind =
  | "requested" | "authorized" | "denied" | "scheduled" | "collected" | "resulted" | "acknowledged" | "escalated" | "cancelled";
export type ExamReleasePolicy = "immediate" | "after_acknowledgement" | "never_citizen";
export type ExamProviderKind = "municipal" | "accredited" | "consortium" | "lacen" | "imaging";
export type ExamDeadlines = Record<ExamStage, number | null>;

export interface ExamResultField { key: string; label: string; unit: string | null; loinc: string | null }
export interface ExamResultValue { key: string | null; label: string; value: string; unit: string | null; loinc: string | null }
// Visão clínica (quem pediu, leitura do 19a, equipe) traz tudo; a visão
// administrativa (unidade, regulador — plano do api, Desvio 10) só traz id,
// origem, fonte, data e laboratório: valores, crítico e laudo ausentes.
export interface ExamResult {
  id: string; origin: "order" | "external"; source: "unit" | "provider" | "fhir" | "consultation";
  released_on: string; laboratory_name: string; critical?: boolean; values?: ExamResultValue[];
  // Curta e assinada (5 min); o download grava trilha.
  pdf_url?: string; acknowledged_at?: string | null;
}
// Divergência D8: o resultado externo listado traz o exame.
export interface ExternalExamResult extends ExamResult { sigtap_code: string; label: string }
export interface ExamEvent {
  kind: ExamEventKind | string; occurred_at: string; actor: { type: "user" | "provider"; name: string }; reason: string | null;
}
export interface ExamOverdue { stage: ExamStage; due_on: string; days_late: number }
export interface ExamOrder {
  id: string; short_code: string; patient: { id: string; display_name: string };
  sigtap_code: string; sigtap_competence: string; label: string; priority: ExamPriority;
  cid10_justification: string | null; origin: ExamOrigin | string;
  requester: { user_id: string | null; name: string; cns: string | null };
  unit: { id: string; name: string }; provider: { id: string; name: string } | null;
  scheduled_at: string | null; scheduled_place: string | null; external_reference: string | null;
  status: ExamStatus | string; status_changed_at: string; overdue: ExamOverdue | null; escalated: boolean;
  result: ExamResult | null;
  // Só no GET /exams/orders/:id.
  events?: ExamEvent[];
  // Divergência D6: a consulta de origem e os campos de resultado do catálogo.
  consultation_id?: string | null; result_fields?: ExamResultField[];
}
export interface ExamOrderPage { orders: ExamOrder[]; next_before: string | null }
export interface ExamOrderQuery {
  patientId?: string; unitId?: string; consultationId?: string; statuses?: ExamStatus[];
  overdue?: boolean; stage?: ExamStage | ""; providerId?: string; sigtapCode?: string; before?: string | null;
}
export const EXAM_PAGE_SIZE = 50;

export interface ExamPatient { id: string; display_name: string; cpf_masked: string }
export interface ExamCatalogEntry { sigtap_code: string; label: string; result_fields: ExamResultField[] }
export interface ExamProviderRef { id: string; name: string }
export interface ExamRequester { user_id: string; name: string }

// Divergência D16: a guia é da unidade escolhida (`unit_id`).
export interface ReceptionOrderInput {
  patient_id: string; unit_id: string; sigtap_code: string; priority: ExamPriority; cid10_justification?: string;
  requester: { user_id: string } | { name: string; cns: string }; external_reference?: string; guide_upload_id: string;
}
export interface ScheduleWhen { scheduled_at: string; scheduled_place: string }
export interface ScheduleInput extends ScheduleWhen { provider_id: string }
export interface ResultInput {
  upload_id: string; released_on: string; laboratory_name: string; critical: boolean; values: { key: string; value: string }[];
}
export interface ExternalResultInput {
  patient_id: string; sigtap_code: string; released_on: string; laboratory_name: string; upload_id: string;
  values: { key: string; value: string }[]; consultation_id?: string;
}
export interface PendingAcknowledgements { orders: ExamOrder[]; denied: ExamOrder[] }

export interface ExamCatalogItem {
  id: string; sigtap_code: string; label: string; requires_authorization: boolean; release_policy: ExamReleasePolicy;
  deadlines: ExamDeadlines; result_fields: ExamResultField[]; active: boolean;
}
export type ExamCatalogItemInput = Omit<ExamCatalogItem, "id" | "label">;
export interface ExamProviderItem { catalog_item_id: string; deadlines: ExamDeadlines }
export interface ExamProvider {
  id: string; name: string; cnes: string | null; kind: ExamProviderKind; active: boolean; items: ExamProviderItem[];
  // Divergência D9: só se existe token ativo (o token nunca volta).
  has_fhir_token?: boolean;
}
export type ExamProviderInput = Omit<ExamProvider, "id" | "has_fhir_token">;
export interface ExamReportRow {
  provider: ExamProviderRef; authorized: number; collected: number; resulted_on_time: number; resulted_late: number; pending: number;
}
export interface ProviderOrder {
  id: string; short_code: string; patient: { name: string; birth_date: string; cpf_last3: string };
  sigtap_code: string; label: string; priority: ExamPriority; status: ExamStatus | string;
  scheduled_at: string | null; scheduled_place: string | null; overdue: ExamOverdue | null;
  // Divergência D6.
  result_fields?: ExamResultField[];
}
export interface ProviderOrderPage { orders: ProviderOrder[]; next_before: string | null; overdue_count: number }
export interface ProviderOrderQuery { statuses?: ExamStatus[]; overdue?: boolean; before?: string | null }

export const EXAM_UPLOAD_MAX_BYTES = 10 * 1024 * 1024;
export const EXAM_UPLOAD_TYPES = [ "application/pdf", "image/jpeg", "image/png" ] as const;

const examPath = (...parts: string[]) => [ EXAMS_BASE, ...parts.map(encodeURIComponent) ].join("/");
const withQuery = (path: string, params: URLSearchParams) => (params.toString() ? `${path}?${params.toString()}` : path);

function orderParams(q: ExamOrderQuery): URLSearchParams {
  const p = new URLSearchParams();
  if (q.patientId) p.set("patient_id", q.patientId);
  if (q.unitId) p.set("unit_id", q.unitId);
  if (q.consultationId) p.set("consultation_id", q.consultationId);
  for (const s of q.statuses ?? []) p.append("status[]", s);
  if (q.overdue) p.set("overdue", "true");
  if (q.stage) p.set("stage", q.stage);
  if (q.providerId) p.set("provider_id", q.providerId);
  if (q.sigtapCode) p.set("sigtap_code", q.sigtapCode);
  if (q.before) p.set("before", q.before);
  p.set("limit", String(EXAM_PAGE_SIZE));
  return p;
}

export function listExamOrders(q: ExamOrderQuery): Promise<ExamOrderPage> {
  return jsonFetch(withQuery(examPath("orders"), orderParams(q)));
}

export function getExamOrder(id: string): Promise<ExamOrder> {
  return jsonFetch(examPath("orders", id));
}

export async function lookupExamPatient(cpf: string): Promise<ExamPatient> {
  return (await jsonFetch<{ patient: ExamPatient }>(examPath("patient_lookup"), postProfessional({ cpf }))).patient;
}

export async function listExamCatalog(): Promise<ExamCatalogEntry[]> {
  return (await jsonFetch<{ items: ExamCatalogEntry[] }>(examPath("catalog"))).items;
}

export async function listEnabledProviders(sigtapCode?: string): Promise<ExamProviderRef[]> {
  const p = new URLSearchParams();
  if (sigtapCode) p.set("sigtap_code", sigtapCode);
  return (await jsonFetch<{ providers: ExamProviderRef[] }>(withQuery(examPath("providers"), p))).providers;
}

export async function listExamRequesters(unitId: string): Promise<ExamRequester[]> {
  const p = new URLSearchParams({ unit_id: unitId });
  return (await jsonFetch<{ requesters: ExamRequester[] }>(withQuery(examPath("requesters"), p))).requesters;
}

export function createReceptionOrder(input: ReceptionOrderInput): Promise<ExamOrder> {
  return jsonFetch(examPath("orders"), postProfessional(input));
}

function readBase64(file: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const text = String(reader.result);
      resolve(text.slice(text.indexOf(",") + 1));
    };
    reader.onerror = () => reject(reader.error ?? new Error("read_failed"));
    reader.readAsDataURL(file);
  });
}

// Só o tipo e o conteúdo: o nome do arquivo pode trazer o nome do paciente.
export async function uploadExamFile(file: File): Promise<string> {
  const data = await readBase64(file);
  return (await jsonFetch<{ upload_id: string }>(examPath("uploads"), postProfessional({ content_type: file.type, data }))).upload_id;
}

export async function scheduleExam(id: string, input: ScheduleInput): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "schedule"), postProfessional(input));
}

export async function collectExam(id: string): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "collect"), postProfessional({}));
}

export async function cancelExam(id: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "cancel"), postProfessional({ reason }));
}

export async function registerExamResult(id: string, input: ResultInput): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "result"), postProfessional(input));
}

export async function acknowledgeExam(id: string, reason?: string): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "acknowledge"), postProfessional(reason ? { reason } : {}));
}

export async function listPendingAcknowledgements(): Promise<PendingAcknowledgements> {
  const body = await jsonFetch<{ orders: ExamOrder[]; denied?: ExamOrder[] }>(examPath("acknowledgements", "pending"));
  return { orders: body.orders, denied: body.denied ?? [] };
}

export async function listTeamAcknowledgements(): Promise<ExamOrder[]> {
  return (await jsonFetch<{ orders: ExamOrder[] }>(examPath("acknowledgements", "team"))).orders;
}

export function recordExternalResult(input: ExternalResultInput): Promise<ExamResult> {
  return jsonFetch(examPath("external_results"), postProfessional(input));
}

export async function listExternalResults(patientId: string): Promise<ExternalExamResult[]> {
  const p = new URLSearchParams({ patient_id: patientId });
  return (await jsonFetch<{ results: ExternalExamResult[] }>(withQuery(examPath("external_results"), p))).results;
}

export function listRegulationQueue(before: string | null): Promise<ExamOrderPage> {
  const p = new URLSearchParams();
  if (before) p.set("before", before);
  p.set("limit", String(EXAM_PAGE_SIZE));
  return jsonFetch(withQuery(examPath("regulation", "queue"), p));
}

export async function authorizeExam(id: string, providerId: string): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "authorize"), postProfessional({ provider_id: providerId }));
}

export async function denyExam(id: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(examPath("orders", id, "deny"), postProfessional({ reason }));
}

// Admin municipal; toda escrita com step-up (quem trata 401 mfa_required é o SensitiveAction).
export async function listCatalogItems(): Promise<ExamCatalogItem[]> {
  return (await jsonFetch<{ items: ExamCatalogItem[] }>(examPath("admin", "catalog_items"))).items;
}

export function createCatalogItem(input: ExamCatalogItemInput): Promise<ExamCatalogItem> {
  return jsonFetch(examPath("admin", "catalog_items"), postProfessional(input));
}

export function updateCatalogItem(id: string, input: ExamCatalogItemInput): Promise<ExamCatalogItem> {
  return jsonFetch(examPath("admin", "catalog_items", id), { method: "PATCH", body: JSON.stringify(input) });
}

export async function listExamProviders(): Promise<ExamProvider[]> {
  return (await jsonFetch<{ providers: ExamProvider[] }>(examPath("admin", "providers"))).providers;
}

export function createExamProvider(input: ExamProviderInput): Promise<ExamProvider> {
  return jsonFetch(examPath("admin", "providers"), postProfessional(input));
}

export function updateExamProvider(id: string, input: ExamProviderInput): Promise<ExamProvider> {
  return jsonFetch(examPath("admin", "providers", id), { method: "PATCH", body: JSON.stringify(input) });
}

// Mostrado uma vez: quem chama guarda só no estado do componente.
export async function rotateFhirToken(id: string): Promise<string> {
  return (await jsonFetch<{ token: string }>(examPath("admin", "providers", id, "fhir_token"), postProfessional({}))).token;
}

export async function revokeFhirToken(id: string): Promise<void> {
  await jsonFetch<unknown>(examPath("admin", "providers", id, "fhir_token"), { method: "DELETE", body: "{}" });
}

export function inviteProviderUser(id: string, email: string): Promise<Invitation> {
  return jsonFetch(examPath("admin", "providers", id, "users"), postProfessional({ email }));
}

function reportParams(month: string, providerId: string | null): URLSearchParams {
  const p = new URLSearchParams({ month });
  if (providerId) p.set("provider_id", providerId);
  return p;
}

export async function getExamReport(month: string, providerId: string | null): Promise<ExamReportRow[]> {
  return (await jsonFetch<{ rows: ExamReportRow[] }>(withQuery(examPath("admin", "report"), reportParams(month, providerId)))).rows;
}

export function fetchExamReportCsv(month: string, providerId: string | null, detail: boolean): Promise<SignatureFile> {
  const p = reportParams(month, providerId);
  if (detail) p.set("detail", "true");
  return binaryFetch(withQuery(examPath("admin", "report.csv"), p), "text/csv");
}

async function jsonFetchWithHeaders<T>(url: string): Promise<{ body: T; headers: Headers }> {
  const res = await fetch(url, { credentials: "include", headers: { Accept: "application/json" } });
  if (!res.ok) {
    const text = await res.text().catch(() => "");
    let body: unknown = text;
    if (text) { try { body = JSON.parse(text); } catch { /* deixa string */ } }
    throw new ApiError(res.status, body, `${res.status} on ${url}`);
  }
  return { body: (await res.json()) as T, headers: res.headers };
}

// Portal do prestador (contrato §6): guia de outro prestador = 404 not_found.
export async function listProviderOrders(q: ProviderOrderQuery): Promise<ProviderOrderPage> {
  const p = new URLSearchParams();
  for (const s of q.statuses ?? []) p.append("status[]", s);
  if (q.overdue) p.set("overdue", "true");
  if (q.before) p.set("before", q.before);
  p.set("limit", String(EXAM_PAGE_SIZE));
  const { body, headers } = await jsonFetchWithHeaders<{ orders: ProviderOrder[]; next_before: string | null }>(
    withQuery(examPath("provider", "orders"), p));
  const count = Number(headers.get("X-Overdue-Count") ?? "0");
  return { orders: body.orders, next_before: body.next_before, overdue_count: Number.isFinite(count) ? count : 0 };
}

// Divergência D9: o prestador é o do usuário; o corpo não leva provider_id.
export async function providerSchedule(id: string, input: ScheduleWhen): Promise<void> {
  await jsonFetch<unknown>(examPath("provider", "orders", id, "schedule"), postProfessional(input));
}

export async function providerCollect(id: string): Promise<void> {
  await jsonFetch<unknown>(examPath("provider", "orders", id, "collect"), postProfessional({}));
}

export async function providerResult(id: string, input: ResultInput): Promise<void> {
  await jsonFetch<unknown>(examPath("provider", "orders", id, "result"), postProfessional(input));
}

// O `pdf_url` vem pronto do api (curto e assinado); lido com fetch para a
// tela mostrar a recusa (link vencido) em vez de uma janela em branco.
export function fetchExamPdf(pdfUrl: string): Promise<SignatureFile> {
  return binaryFetch(pdfUrl, "application/pdf");
}
```

Em `src/lib/features.ts`, troque o tipo e o mapa:

```ts
export type FeatureKey =
  | "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature" | "clinical_documents" | "controlled_prescriptions"
  | "patient_timeline" | "exam_tracking" | "exam_fhir_inbound" | "exam_pec_bridge" | "fake_lab";
```

```ts
  patient_timeline: "Linha do tempo do paciente",
  exam_tracking: "Rastreamento de exames",
  exam_fhir_inbound: "Entrada de resultados por FHIR",
  exam_pec_bridge: "Solicitar exames pelo PEC",
  fake_lab: "Laboratório simulado (fora de produção)"
```

Em `vite.config.ts`, acrescente ao comentário a linha `//   /exams      → exames (módulo 21a: pedido, guia, resultado, ciência, regulação, admin e portal do prestador).` e ao `proxy` a entrada `"/exams": proxy(TARGET),` logo depois de `"/sncr": proxy(TARGET),`.

Crie `src/test/examFixtures.ts`:

```ts
// src/test/examFixtures.ts
// Dados comuns aos testes do módulo 21a. Cada arquivo de teste faz o próprio
// vi.mock("…/lib/api"); aqui só tipos e a sessão.
import type {
  ExamCatalogEntry, ExamCatalogItem, ExamOrder, ExamProvider, ExamResult, ExamResultField, HealthUnit, ProviderOrder, SessionUser
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW21 = "2026-10-13T10:00:00-03:00";
export const TODAY21 = "2026-10-13";
export const VALID_CPF = "52998224725";
export const VALID_CNS = "702002887429583";
export const UNIT21: HealthUnit = { id: "u1", name: "UBS Centro", kind: "ubs" };

export const HEMOGRAM_FIELDS: ExamResultField[] = [
  { key: "hemoglobina", label: "Hemoglobina", unit: "g/dL", loinc: "718-7" },
  { key: "leucocitos", label: "Leucócitos", unit: "/mm³", loinc: "6690-2" }
];
export const HEMOGRAM_ENTRY: ExamCatalogEntry = { sigtap_code: "0202020380", label: "Hemograma completo", result_fields: HEMOGRAM_FIELDS };
export const HBA1C_ENTRY: ExamCatalogEntry = {
  sigtap_code: "0202010503", label: "Dosagem de hemoglobina glicosilada",
  result_fields: [ { key: "hba1c", label: "HbA1c", unit: "%", loinc: "4548-4" } ]
};

// Sessão com o rastreamento ligado; `mfa_verified_at` = agora (janela de step-up aberta).
export function examUser(roles: string[] = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { id: "us1", email_address: "medica@curitiba.demo", features: [ "clinical_record", "exam_tracking" ], ...over });
}

export function examResult(over: Partial<ExamResult> = {}): ExamResult {
  return {
    id: "er1", origin: "order", source: "fhir", released_on: "2026-10-12", laboratory_name: "Laboratório Simulado", critical: false,
    values: [ { key: "hemoglobina", label: "Hemoglobina", value: "13.5", unit: "g/dL", loinc: "718-7" } ],
    pdf_url: "/exams/results/er1/pdf?sig=abc", acknowledged_at: null, ...over
  };
}

export function examOrder(over: Partial<ExamOrder> = {}): ExamOrder {
  return {
    id: "eo1", short_code: "K7M2Q9XW3P", patient: { id: "pa1", display_name: "Joana Lima" },
    sigtap_code: "0202020380", sigtap_competence: "202609", label: "Hemograma completo", priority: "routine",
    cid10_justification: null, origin: "consultation", requester: { user_id: "us1", name: "Dra. Ana Souza", cns: null },
    unit: { id: "u1", name: "UBS Centro" }, provider: { id: "pv1", name: "Laboratório Simulado" },
    scheduled_at: null, scheduled_place: null, external_reference: null, status: "awaiting_schedule",
    status_changed_at: "2026-10-10T09:00:00-03:00", overdue: null, escalated: false, result: null,
    events: [ { kind: "requested", occurred_at: "2026-10-10T09:00:00-03:00", actor: { type: "user", name: "Dra. Ana Souza" }, reason: null } ],
    consultation_id: "c1", result_fields: HEMOGRAM_FIELDS, ...over
  };
}

export function examCatalogItem(over: Partial<ExamCatalogItem> = {}): ExamCatalogItem {
  return {
    id: "ci1", sigtap_code: "0202020380", label: "Hemograma completo", requires_authorization: false,
    release_policy: "after_acknowledgement", deadlines: { authorize: null, schedule: null, collect: null, result: 10, acknowledge: null },
    result_fields: HEMOGRAM_FIELDS, active: true, ...over
  };
}

export function examProvider(over: Partial<ExamProvider> = {}): ExamProvider {
  return {
    id: "pv1", name: "Laboratório Simulado", cnes: "1234567", kind: "accredited", active: true,
    items: [ { catalog_item_id: "ci1", deadlines: { authorize: null, schedule: null, collect: null, result: 5, acknowledge: null } } ],
    has_fhir_token: false, ...over
  };
}

export function providerOrder(over: Partial<ProviderOrder> = {}): ProviderOrder {
  return {
    id: "eo1", short_code: "K7M2Q9XW3P", patient: { name: "Joana Lima", birth_date: "1980-05-02", cpf_last3: "725" },
    sigtap_code: "0202020380", label: "Hemograma completo", priority: "routine", status: "awaiting_schedule",
    scheduled_at: null, scheduled_place: null, overdue: null, result_fields: HEMOGRAM_FIELDS, ...over
  };
}

export function pdfFile(name = "laudo.pdf"): File {
  return new File([ "%PDF-1.4" ], name, { type: "application/pdf" });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/api.exams.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.exams.test.ts src/lib/features.ts src/lib/features.test.ts src/test/examFixtures.ts vite.config.ts
$G commit -m "feat: add the exam tracking client, switches and dev proxy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 2: Regras de tela dos exames e páginas com cursor

**Files:**
- Create: `src/lib/exams.ts`, `src/hooks/useExamOrderPages.ts`
- Test: `src/lib/exams.test.ts`

**Interfaces:**
- Consumes: tipos e constantes da Task 1; `OPENING_ENDED` (`lib/clinicalRecord`), `cid10Problem` (`lib/consultation`), `featureDisabledKey`/`hasFeature` (`lib/features`), `cityLocalIso` (`lib/format`), `isValidCns` (`lib/professionals`), `Tone` (`theme/tokens`).
- Produces (em `src/lib/exams.ts`):
  ```ts
  export const EXAM_ORDERS_KEY, EXAM_ORDER_KEY, EXAM_PENDING_KEY, EXAM_TEAM_KEY, EXAM_REGULATION_KEY, EXAM_CATALOG_KEY,
    EXAM_ADMIN_CATALOG_KEY, EXAM_PROVIDER_REFS_KEY, EXAM_ADMIN_PROVIDERS_KEY, EXAM_REPORT_KEY, PROVIDER_ORDERS_KEY,
    EXTERNAL_RESULTS_KEY, EXAM_REQUESTERS_KEY: string;
  export const EXAMS_DISABLED, EXAMS_NO_ROLE, PDF_LINK_EXPIRED: string;
  export function examStatusLabel(status: string): string;
  export function examStatusTone(status: string): Tone | undefined;
  export const EXAM_STAGES: ExamStage[]; export const STAGE_LABEL: Record<ExamStage, string>;
  export const DEFAULT_DEADLINES: Record<ExamStage, number>;
  export const PRIORITY_LABEL: Record<ExamPriority, string>; export const ORIGIN_LABEL, SOURCE_LABEL: Record<string, string>;
  export function eventLabel(kind: string): string;
  export const RELEASE_POLICIES: { value: ExamReleasePolicy; label: string; hint: string }[]; export function releasePolicyLabel(p: string): string;
  export const PROVIDER_KINDS: { value: ExamProviderKind; label: string }[]; export function providerKindLabel(k: string): string;
  export type EscalationReason = "critical" | "unlinked_requester" | "overdue";
  export const ESCALATION_LABEL: Record<EscalationReason, string>; export function escalationReason(o: ExamOrder): EscalationReason;
  export interface ExamRoles { professional: boolean; reception: boolean; regulator: boolean; admin: boolean; provider: boolean }
  export function examRoles(user: { operator: boolean; memberships: { role: string }[] } | null): ExamRoles;
  export function canUseExams(user: { operator: boolean; features?: unknown } | null): boolean;
  export interface ExamActions { schedule: boolean; collect: boolean; result: boolean; cancel: boolean; acknowledge: boolean; acknowledgeNeedsReason: boolean }
  export function examActions(order: ExamOrder, roles: ExamRoles, userId: string | null): ExamActions;
  export function overdueLabel(o: ExamOverdue | null): string; export function daysLateLabel(days: number): string;
  export function fmtDay(iso: string | null | undefined): string; export function cpfTail(last3: string): string;
  export function uploadProblem(file: { size: number; type: string } | null): string | null;
  export const REASON_MIN = 5; export function reasonProblem(text: string): string | null;
  export type DeadlineDraft = Record<ExamStage, string>;
  export function deadlineDraft(d: ExamDeadlines | null | undefined): DeadlineDraft;
  export function deadlinesProblem(d: DeadlineDraft): string | null; export function deadlinesFrom(d: DeadlineDraft): ExamDeadlines;
  export function effectiveDeadline(stage: ExamStage, provider: ExamDeadlines | null | undefined, catalog: ExamDeadlines | null | undefined, priority?: ExamPriority): number;
  export function deadlinesSummary(d: ExamDeadlines): string;
  export function resultFieldsProblem(fields: ExamResultField[]): string | null;
  export function sigtapProblem(code: string): string | null; export function cnesProblem(value: string): string | null;
  export interface ResultDraft { uploadId: string | null; releasedOn: string; laboratoryName: string; critical: boolean; values: Record<string, string> }
  export function emptyResultDraft(today: string): ResultDraft; export function resultProblem(d: ResultDraft, today: string): string | null;
  export function resultBody(d: ResultDraft): ResultInput;
  export interface ScheduleDraft { providerId: string; at: string; place: string }
  export function scheduleProblem(d: ScheduleDraft, needsProvider: boolean): string | null;
  export function scheduleWhen(d: ScheduleDraft): ScheduleWhen; export function scheduleBody(d: ScheduleDraft, providerId?: string): ScheduleInput;
  export interface GuideDraft { sigtapCode: string; priority: ExamPriority; cid10: string; requesterMode: "linked" | "typed";
    requesterId: string; requesterName: string; requesterCns: string; externalReference: string; guideUploadId: string | null }
  export const EMPTY_GUIDE: GuideDraft; export function guideProblem(d: GuideDraft): string | null;
  export function guideBody(d: GuideDraft, patientId: string, unitId: string): ReceptionOrderInput;
  export function mergeById<T extends { id: string }>(pages: { orders: T[] }[]): T[];
  export function refreshExams(client: QueryClient): Promise<void>;
  export function pendingNoticeText(pending: number, denied: number): string;
  export function pdfLinkExpired(err: unknown): boolean;
  export function examError(err: unknown): string;
  ```
- Produces (em `src/hooks/useExamOrderPages.ts`): `useExamOrderPages<P extends { orders: { id: string }[]; next_before: string | null }>(queryKey: readonly unknown[], fetchPage: (before: string | null) => Promise<P>, enabled?: boolean): { query /* useInfiniteQuery, pages: P[] */; orders: P["orders"][number][] }` — `orders` sem repetição (`mergeById`), `gcTime: 0`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/exams.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  EMPTY_GUIDE, EXAMS_DISABLED, PDF_LINK_EXPIRED, cnesProblem, cpfTail, deadlineDraft, deadlinesFrom, deadlinesProblem, deadlinesSummary,
  effectiveDeadline, emptyResultDraft, escalationReason, eventLabel, examActions, examError, examRoles, examStatusLabel, fmtDay,
  guideBody, guideProblem, mergeById, overdueLabel, pdfLinkExpired, pendingNoticeText, reasonProblem, resultBody, resultFieldsProblem,
  resultProblem, scheduleBody, scheduleProblem, scheduleWhen, sigtapProblem, uploadProblem, canUseExams
} from "./exams";
import { VALID_CNS, examOrder, examResult } from "../test/examFixtures";

const NONE = { professional: false, reception: false, regulator: false, admin: false, provider: false };

describe("rótulos", () => {
  it("situação conhecida em português; desconhecida crua", () => {
    expect(examStatusLabel("awaiting_schedule")).toBe("a agendar");
    expect(examStatusLabel("resulted")).toBe("resultado registrado");
    expect(examStatusLabel("archived")).toBe("archived");
    expect(eventLabel("escalated")).toBe("enviado à equipe");
    expect(eventLabel("rerouted")).toBe("rerouted");
  });

  it("atraso, dia e o final do CPF", () => {
    expect(overdueLabel(null)).toBe("no prazo");
    expect(overdueLabel({ stage: "collect", due_on: "2026-10-10", days_late: 1 })).toBe("coleta · 1 dia de atraso");
    expect(overdueLabel({ stage: "acknowledge", due_on: "2026-10-01", days_late: 12 })).toBe("ciência · 12 dias de atraso");
    expect(fmtDay("2026-10-13")).toBe("13/10/2026");
    expect(fmtDay(null)).toBe("—");
    expect(cpfTail("725")).toBe("CPF final 725");
  });

  it("aviso do Início no singular e no plural", () => {
    expect(pendingNoticeText(0, 0)).toBe("");
    expect(pendingNoticeText(1, 0)).toBe("1 resultado aguardando sua ciência");
    expect(pendingNoticeText(3, 2)).toBe("3 resultados aguardando sua ciência · 2 exames negados");
    expect(pendingNoticeText(0, 1)).toBe("1 exame negado");
  });
});

describe("papéis e botões", () => {
  it("papéis da sessão; operador não tem nenhum", () => {
    const user = { operator: false, memberships: [ { role: "citizen_verifier" }, { role: "exam_regulator" } ] };
    expect(examRoles(user)).toEqual({ ...NONE, reception: true, regulator: true });
    expect(examRoles({ operator: true, memberships: [ { role: "municipal_admin" } ] })).toEqual(NONE);
    expect(examRoles(null)).toEqual(NONE);
    expect(canUseExams({ operator: false, features: [ "exam_tracking" ] })).toBe(true);
    expect(canUseExams({ operator: false, features: [] })).toBe(false);
    expect(canUseExams({ operator: true, features: [ "exam_tracking" ] })).toBe(false);
  });

  it("unidade agenda, coleta e registra resultado conforme a situação", () => {
    const reception = { ...NONE, reception: true };
    expect(examActions(examOrder({ status: "awaiting_schedule" }), reception, "us9")).toMatchObject({ schedule: true, collect: false, result: false, cancel: true });
    expect(examActions(examOrder({ status: "scheduled" }), reception, "us9")).toMatchObject({ schedule: false, collect: true, result: true });
    expect(examActions(examOrder({ status: "collected" }), reception, "us9")).toMatchObject({ collect: false, result: true, cancel: true });
    expect(examActions(examOrder({ status: "resulted", result: examResult() }), reception, "us9")).toMatchObject({ result: false, cancel: false, acknowledge: false });
  });

  it("ciência: quem pediu sem motivo; a equipe só no escalado e com motivo; ninguém mais", () => {
    const pro = { ...NONE, professional: true };
    const resulted = examOrder({ status: "resulted", result: examResult() });
    expect(examActions(resulted, pro, "us1")).toMatchObject({ acknowledge: true, acknowledgeNeedsReason: false });
    expect(examActions(resulted, pro, "us2")).toMatchObject({ acknowledge: false });
    expect(examActions({ ...resulted, escalated: true }, pro, "us2")).toMatchObject({ acknowledge: true, acknowledgeNeedsReason: true });
    expect(examActions({ ...resulted, escalated: true }, { ...NONE, reception: true }, "us2")).toMatchObject({ acknowledge: false });
  });

  it("cancelar: quem pediu ou a recepção/admin, só antes do resultado", () => {
    const pro = { ...NONE, professional: true };
    expect(examActions(examOrder({ status: "awaiting_authorization" }), pro, "us1").cancel).toBe(true);
    expect(examActions(examOrder({ status: "awaiting_authorization" }), pro, "us2").cancel).toBe(false);
    expect(examActions(examOrder({ status: "denied" }), { ...NONE, admin: true }, "us2").cancel).toBe(false);
  });

  it("motivo do escalonamento derivado do exame", () => {
    expect(escalationReason(examOrder({ result: examResult({ critical: true }) }))).toBe("critical");
    expect(escalationReason(examOrder({ requester: { user_id: null, name: "Dr. Externo", cns: VALID_CNS } }))).toBe("unlinked_requester");
    expect(escalationReason(examOrder({ result: examResult() }))).toBe("overdue");
  });
});

describe("validação", () => {
  it("arquivo: tipo, tamanho e vazio", () => {
    expect(uploadProblem(null)).toBe("escolha um arquivo");
    expect(uploadProblem({ size: 10, type: "application/msword" })).toBe("envie PDF, JPEG ou PNG");
    expect(uploadProblem({ size: 12 * 1024 * 1024, type: "image/jpeg" })).toBe("o arquivo passa de 10 MB");
    expect(uploadProblem({ size: 0, type: "application/pdf" })).toBe("o arquivo está vazio");
    expect(uploadProblem({ size: 2048, type: "image/png" })).toBeNull();
  });

  it("motivo com ao menos 5 caracteres", () => {
    expect(reasonProblem("  ok ")).toBe("escreva o motivo (ao menos 5 caracteres)");
    expect(reasonProblem("Paciente avisado")).toBeNull();
  });

  it("prazos: branco = herda; inteiro de 1 a 365", () => {
    const d = deadlineDraft({ authorize: null, schedule: 7, collect: null, result: 10, acknowledge: null });
    expect(d).toEqual({ authorize: "", schedule: "7", collect: "", result: "10", acknowledge: "" });
    expect(deadlinesProblem(d)).toBeNull();
    expect(deadlinesProblem({ ...d, collect: "0" })).toBe("prazo de coleta: use dias inteiros de 1 a 365, ou deixe em branco");
    expect(deadlinesProblem({ ...d, result: "2.5" })).toBe("prazo de resultado: use dias inteiros de 1 a 365, ou deixe em branco");
    expect(deadlinesFrom({ ...d, collect: " 20 " })).toEqual({ authorize: null, schedule: 7, collect: 20, result: 10, acknowledge: null });
  });

  it("prazo herdado: prestador, catálogo, plataforma; urgente usa a metade para cima", () => {
    const catalog = { authorize: null, schedule: null, collect: null, result: 10, acknowledge: null };
    const provider = { authorize: null, schedule: null, collect: null, result: 5, acknowledge: null };
    expect(effectiveDeadline("result", provider, catalog)).toBe(5);
    expect(effectiveDeadline("result", null, catalog)).toBe(10);
    expect(effectiveDeadline("authorize", null, catalog)).toBe(15);
    expect(effectiveDeadline("authorize", null, catalog, "urgent")).toBe(8);
    expect(deadlinesSummary(catalog)).toBe("15 · 15 · 30 · 10 · 15");
  });

  it("campos de resultado: chave, repetição, nome e LOINC", () => {
    const ok = [ { key: "hemoglobina", label: "Hemoglobina", unit: "g/dL", loinc: "718-7" } ];
    expect(resultFieldsProblem(ok)).toBeNull();
    expect(resultFieldsProblem([ { ...ok[0], key: "Hemoglobina" } ])).toBe("campo 1: a chave usa letras minúsculas, números e _ (ex.: hemoglobina)");
    expect(resultFieldsProblem([ ...ok, { ...ok[0], label: "Hb" } ])).toBe("campo 2: a chave hemoglobina repete");
    expect(resultFieldsProblem([ { ...ok[0], label: " " } ])).toBe("campo 1: informe o nome");
    expect(resultFieldsProblem([ { ...ok[0], loinc: "7187" } ])).toBe("campo 1: LOINC no formato 718-7");
  });

  it("SIGTAP e CNES", () => {
    expect(sigtapProblem("0202020380")).toBeNull();
    expect(sigtapProblem("02.02.02.038-0")).toBe("o código SIGTAP tem 10 dígitos");
    expect(cnesProblem("")).toBeNull();
    expect(cnesProblem("1234567")).toBeNull();
    expect(cnesProblem("123")).toBe("o CNES tem 7 dígitos");
  });

  it("resultado: anexo, data não futura, laboratório; valores vazios ficam fora", () => {
    const d = { ...emptyResultDraft("2026-10-13"), laboratoryName: "Lab" };
    expect(d.releasedOn).toBe("2026-10-13");
    expect(resultProblem(d, "2026-10-13")).toBe("anexe o laudo");
    const full = { ...d, uploadId: "up1", values: { hemoglobina: " 13.5 ", leucocitos: "" } };
    expect(resultProblem({ ...full, releasedOn: "2026-10-14" }, "2026-10-13")).toBe("a data de liberação não pode ser no futuro");
    expect(resultProblem({ ...full, laboratoryName: " " }, "2026-10-13")).toBe("informe o laboratório");
    expect(resultProblem(full, "2026-10-13")).toBeNull();
    expect(resultBody({ ...full, critical: true })).toEqual({
      upload_id: "up1", released_on: "2026-10-13", laboratory_name: "Lab", critical: true, values: [ { key: "hemoglobina", value: "13.5" } ]
    });
  });

  it("agendamento: prestador quando falta, data e hora da cidade e local", () => {
    const d = { providerId: "", at: "2026-10-20T08:30", place: " Posto de coleta " };
    expect(scheduleProblem(d, true)).toBe("escolha o prestador");
    expect(scheduleProblem(d, false)).toBeNull();
    expect(scheduleProblem({ ...d, at: "" }, false)).toBe("informe data e hora");
    expect(scheduleProblem({ ...d, place: "" }, false)).toBe("informe o local");
    expect(scheduleWhen(d)).toEqual({ scheduled_at: "2026-10-20T08:30:00-03:00", scheduled_place: "Posto de coleta" });
    expect(scheduleBody({ ...d, providerId: "pv2" })).toMatchObject({ provider_id: "pv2" });
    expect(scheduleBody(d, "pv1")).toMatchObject({ provider_id: "pv1" });
  });

  it("guia: exame, CID-10, solicitante e anexo; corpo com solicitante digitado ou da rede", () => {
    expect(guideProblem(EMPTY_GUIDE)).toBe("escolha o exame");
    const base = { ...EMPTY_GUIDE, sigtapCode: "0202020380", guideUploadId: "up1" };
    expect(guideProblem({ ...base, cid10: "xx" })).toBe("use um código CID-10, ex.: E11 ou E119");
    expect(guideProblem(base)).toBe("escolha o profissional que pediu");
    expect(guideProblem({ ...base, requesterMode: "typed", requesterName: "Dr. Externo", requesterCns: "123" })).toBe("CNS de quem pediu inválido");
    expect(guideProblem({ ...base, requesterId: "us1", guideUploadId: null })).toBe("anexe a foto ou o PDF da guia");
    expect(guideBody({ ...base, requesterId: "us1", cid10: "e11.9", priority: "urgent" }, "pa1", "u1")).toEqual({
      patient_id: "pa1", unit_id: "u1", sigtap_code: "0202020380", priority: "urgent", cid10_justification: "E119",
      requester: { user_id: "us1" }, guide_upload_id: "up1"
    });
    expect(guideBody({ ...base, requesterMode: "typed", requesterName: " Dr. Externo ", requesterCns: VALID_CNS, externalReference: " G-77 " }, "pa1", "u1"))
      .toEqual({ patient_id: "pa1", unit_id: "u1", sigtap_code: "0202020380", priority: "routine", requester: { name: "Dr. Externo", cns: VALID_CNS },
        external_reference: "G-77", guide_upload_id: "up1" });
  });
});

describe("páginas e recusas", () => {
  it("mergeById: junta sem repetir, vale a primeira", () => {
    const a = examOrder({ id: "a" });
    const b = examOrder({ id: "b" });
    expect(mergeById([ { orders: [ a, b ] }, { orders: [ { ...b, status: "scheduled" } ] } ]).map((o) => [ o.id, o.status ]))
      .toEqual([ [ "a", "awaiting_schedule" ], [ "b", "awaiting_schedule" ] ]);
  });

  it("frases das recusas", () => {
    expect(examError(new ApiError(403, { error: "feature_disabled", feature: "exam_tracking" }, "x"))).toBe(EXAMS_DISABLED);
    expect(examError(new ApiError(409, { error: "invalid_transition", status: "scheduled" }, "x"))).toBe("a situação do exame mudou — atualize para ver a atual");
    expect(examError(new ApiError(422, { error: "provider_not_enabled_for_exam" }, "x"))).toBe("este prestador não faz este exame — escolha outro ou habilite-o no cadastro");
    expect(examError(new ApiError(422, { error: "invalid_file" }, "x"))).toBe("arquivo inválido — envie PDF, JPEG ou PNG de até 10 MB");
    expect(examError(new ApiError(404, { error: "patient_not_found" }, "x"))).toBe("nenhum paciente com este CPF na cidade");
    expect(examError(new ApiError(500, "", "x"))).toBe("não foi possível concluir (500) — tente de novo");
    expect(examError(new TypeError("fetch failed"))).toBe("sem conexão com o servidor — tente de novo");
  });

  it("link do laudo vencido: 403, 410 ou link_expired", () => {
    expect(pdfLinkExpired(new ApiError(410, { error: "link_expired" }, "x"))).toBe(true);
    expect(pdfLinkExpired(new ApiError(403, "", "x"))).toBe(true);
    expect(pdfLinkExpired(new ApiError(500, "", "x"))).toBe(false);
    expect(examError(new ApiError(410, { error: "link_expired" }, "x"))).toBe(PDF_LINK_EXPIRED);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/exams.test.ts`
Expected: FAIL — `./exams` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/exams.ts
// Rastreamento de exames no painel da cidade (módulo 21a, ADR 0036; contrato
// 2026-10-13). Regras de tela, puras: rótulos, quem vê qual botão, validação
// dos formulários e as frases das recusas. Quem pode o quê de verdade, a
// situação, os prazos vencidos e a política de liberação são do api.
import type { QueryClient } from "@tanstack/react-query";
import {
  ApiError, EXAM_UPLOAD_MAX_BYTES, EXAM_UPLOAD_TYPES, errorCode,
  type ExamDeadlines, type ExamOrder, type ExamOverdue, type ExamPriority, type ExamProviderKind, type ExamReleasePolicy,
  type ExamResultField, type ExamStage, type ReceptionOrderInput, type ResultInput, type ScheduleInput, type ScheduleWhen
} from "./api";
import { OPENING_ENDED } from "./clinicalRecord";
import { cid10Problem } from "./consultation";
import { featureDisabledKey, hasFeature } from "./features";
import { cityLocalIso } from "./format";
import { isValidCns } from "./professionals";
import type { Tone } from "../theme/tokens";

export const EXAM_ORDERS_KEY = "examOrders";
export const EXAM_ORDER_KEY = "examOrder";
export const EXAM_PENDING_KEY = "examPending";
export const EXAM_TEAM_KEY = "examTeam";
export const EXAM_REGULATION_KEY = "examRegulation";
export const EXAM_CATALOG_KEY = "examCatalog";
export const EXAM_ADMIN_CATALOG_KEY = "examAdminCatalog";
export const EXAM_PROVIDER_REFS_KEY = "examProviderRefs";
export const EXAM_ADMIN_PROVIDERS_KEY = "examAdminProviders";
export const EXAM_REPORT_KEY = "examReport";
export const PROVIDER_ORDERS_KEY = "providerOrders";
export const EXTERNAL_RESULTS_KEY = "examExternalResults";
export const EXAM_REQUESTERS_KEY = "examRequesters";

// Tudo o que muda quando um exame anda (as listas e o detalhe).
const MOVING_KEYS = [ EXAM_ORDERS_KEY, EXAM_ORDER_KEY, EXAM_PENDING_KEY, EXAM_TEAM_KEY, EXAM_REGULATION_KEY, PROVIDER_ORDERS_KEY, EXTERNAL_RESULTS_KEY ];

export const EXAMS_DISABLED = "o rastreamento de exames está desligado nesta cidade";
export const EXAMS_NO_ROLE = "seu papel não permite ver esta tela";
export const PDF_LINK_EXPIRED = "o link do laudo expirou — abra de novo";

const STATUS_LABEL: Record<string, string> = {
  awaiting_authorization: "aguardando autorização",
  denied: "negado",
  awaiting_schedule: "a agendar",
  scheduled: "agendado",
  collected: "coletado",
  resulted: "resultado registrado",
  acknowledged: "ciência dada",
  cancelled: "cancelado"
};
const STATUS_TONE: Record<string, Tone> = {
  awaiting_authorization: "info", denied: "neutral", resulted: "warn", acknowledged: "ok", cancelled: "neutral"
};

export function examStatusLabel(status: string): string {
  return STATUS_LABEL[status] ?? status;
}

export function examStatusTone(status: string): Tone | undefined {
  return STATUS_TONE[status];
}

export const EXAM_STAGES: ExamStage[] = [ "authorize", "schedule", "collect", "result", "acknowledge" ];
export const STAGE_LABEL: Record<ExamStage, string> = {
  authorize: "autorização", schedule: "agendamento", collect: "coleta", result: "resultado", acknowledge: "ciência"
};
// Padrão da plataforma (spec §5; contrato §1).
export const DEFAULT_DEADLINES: Record<ExamStage, number> = { authorize: 15, schedule: 15, collect: 30, result: 30, acknowledge: 15 };

export const PRIORITY_LABEL: Record<ExamPriority, string> = { routine: "rotina", urgent: "urgente" };
export const ORIGIN_LABEL: Record<string, string> = { consultation: "consulta", reception_guide: "guia da recepção", pec: "PEC" };
export const SOURCE_LABEL: Record<string, string> = {
  unit: "registrado pela unidade", provider: "registrado pelo prestador", fhir: "recebido do laboratório (FHIR)",
  consultation: "registrado na consulta"
};

const EVENT_LABEL: Record<string, string> = {
  requested: "solicitado", authorized: "autorizado", denied: "negado", scheduled: "agendado", collected: "coletado",
  resulted: "resultado registrado", acknowledged: "ciência", escalated: "enviado à equipe", cancelled: "cancelado"
};

export function eventLabel(kind: string): string {
  return EVENT_LABEL[kind] ?? kind;
}

export const RELEASE_POLICIES: { value: ExamReleasePolicy; label: string; hint: string }[] = [
  { value: "after_acknowledgement", label: "Depois da ciência", hint: "o cidadão vê o laudo depois que quem pediu dá ciência (padrão)" },
  { value: "immediate", label: "Imediata", hint: "o cidadão vê o laudo assim que o resultado é registrado" },
  { value: "never_citizen", label: "Nunca no wpda", hint: "o cidadão vê \"retire o resultado na sua unidade\"" }
];

export function releasePolicyLabel(policy: string): string {
  return RELEASE_POLICIES.find((p) => p.value === policy)?.label ?? policy;
}

export const PROVIDER_KINDS: { value: ExamProviderKind; label: string }[] = [
  { value: "municipal", label: "Laboratório municipal" },
  { value: "accredited", label: "Credenciado" },
  { value: "consortium", label: "Consórcio" },
  { value: "lacen", label: "LACEN" },
  { value: "imaging", label: "Imagem" }
];

export function providerKindLabel(kind: string): string {
  return PROVIDER_KINDS.find((k) => k.value === kind)?.label ?? kind;
}

export type EscalationReason = "critical" | "unlinked_requester" | "overdue";
export const ESCALATION_LABEL: Record<EscalationReason, string> = {
  critical: "resultado crítico", unlinked_requester: "solicitante sem vínculo", overdue: "prazo de ciência vencido"
};

// O motivo sai do próprio exame (contrato §2 não traz o motivo na lista):
// crítico vence; solicitante digitado é escalado já no resultado; o resto é prazo.
export function escalationReason(o: ExamOrder): EscalationReason {
  if (o.result?.critical) return "critical";
  if (o.requester.user_id === null) return "unlinked_requester";
  return "overdue";
}

export interface ExamRoles { professional: boolean; reception: boolean; regulator: boolean; admin: boolean; provider: boolean }

export function examRoles(user: { operator: boolean; memberships: { role: string }[] } | null): ExamRoles {
  const roles = user && !user.operator ? user.memberships.map((m) => m.role) : [];
  return {
    professional: roles.includes("health_professional"),
    reception: roles.includes("citizen_verifier"),
    regulator: roles.includes("exam_regulator"),
    admin: roles.includes("municipal_admin"),
    provider: roles.includes("exam_provider")
  };
}

export function canUseExams(user: { operator: boolean; features?: unknown } | null): boolean {
  return !!user && !user.operator && hasFeature(user, "exam_tracking");
}

export interface ExamActions {
  schedule: boolean; collect: boolean; result: boolean; cancel: boolean; acknowledge: boolean; acknowledgeNeedsReason: boolean;
}

const BEFORE_RESULT = new Set([ "awaiting_authorization", "awaiting_schedule", "scheduled", "collected" ]);

// O que a tela oferece; o api confere de novo (papel, vínculo, equipe, situação).
export function examActions(order: ExamOrder, roles: ExamRoles, userId: string | null): ExamActions {
  const unit = roles.reception || roles.professional || roles.admin;
  const isRequester = !!userId && order.requester.user_id === userId;
  const byTeam = order.status === "resulted" && order.escalated && roles.professional && !isRequester;
  return {
    schedule: unit && order.status === "awaiting_schedule",
    collect: unit && order.status === "scheduled",
    result: unit && (order.status === "scheduled" || order.status === "collected"),
    cancel: (isRequester || roles.reception || roles.admin) && BEFORE_RESULT.has(order.status),
    acknowledge: order.status === "resulted" && (isRequester || byTeam),
    acknowledgeNeedsReason: byTeam
  };
}

export function daysLateLabel(days: number): string {
  return days === 1 ? "1 dia de atraso" : `${days} dias de atraso`;
}

export function overdueLabel(o: ExamOverdue | null): string {
  if (!o) return "no prazo";
  return `${STAGE_LABEL[o.stage] ?? o.stage} · ${daysLateLabel(o.days_late)}`;
}

export function fmtDay(iso: string | null | undefined): string {
  const m = iso ? /^(\d{4})-(\d{2})-(\d{2})/.exec(iso) : null;
  return m ? `${m[3]}/${m[2]}/${m[1]}` : "—";
}

export function cpfTail(last3: string): string {
  return `CPF final ${last3}`;
}

export function uploadProblem(file: { size: number; type: string } | null): string | null {
  if (!file) return "escolha um arquivo";
  if (!(EXAM_UPLOAD_TYPES as readonly string[]).includes(file.type)) return "envie PDF, JPEG ou PNG";
  if (file.size > EXAM_UPLOAD_MAX_BYTES) return "o arquivo passa de 10 MB";
  if (file.size === 0) return "o arquivo está vazio";
  return null;
}

export const REASON_MIN = 5;

export function reasonProblem(text: string): string | null {
  return text.trim().length < REASON_MIN ? `escreva o motivo (ao menos ${REASON_MIN} caracteres)` : null;
}

export type DeadlineDraft = Record<ExamStage, string>;

export function deadlineDraft(d: ExamDeadlines | null | undefined): DeadlineDraft {
  const out = {} as DeadlineDraft;
  for (const s of EXAM_STAGES) out[s] = d?.[s] == null ? "" : String(d[s]);
  return out;
}

export function deadlinesProblem(d: DeadlineDraft): string | null {
  for (const s of EXAM_STAGES) {
    const v = d[s].trim();
    if (v === "") continue;
    if (!/^\d+$/.test(v) || Number(v) < 1 || Number(v) > 365) {
      return `prazo de ${STAGE_LABEL[s]}: use dias inteiros de 1 a 365, ou deixe em branco`;
    }
  }
  return null;
}

export function deadlinesFrom(d: DeadlineDraft): ExamDeadlines {
  const out = {} as ExamDeadlines;
  for (const s of EXAM_STAGES) out[s] = d[s].trim() === "" ? null : Number(d[s].trim());
  return out;
}

export function effectiveDeadline(
  stage: ExamStage, provider: ExamDeadlines | null | undefined, catalog: ExamDeadlines | null | undefined, priority: ExamPriority = "routine"
): number {
  const base = provider?.[stage] ?? catalog?.[stage] ?? DEFAULT_DEADLINES[stage];
  return priority === "urgent" ? Math.ceil(base / 2) : base;
}

// "autorização · agendamento · coleta · resultado · ciência", em dias.
export function deadlinesSummary(d: ExamDeadlines): string {
  return EXAM_STAGES.map((s) => effectiveDeadline(s, null, d)).join(" · ");
}

export function resultFieldsProblem(fields: ExamResultField[]): string | null {
  const keys = new Set<string>();
  for (const [ i, f ] of fields.entries()) {
    const n = i + 1;
    const key = f.key.trim();
    if (!/^[a-z][a-z0-9_]{0,39}$/.test(key)) return `campo ${n}: a chave usa letras minúsculas, números e _ (ex.: hemoglobina)`;
    if (keys.has(key)) return `campo ${n}: a chave ${key} repete`;
    keys.add(key);
    if (!f.label.trim()) return `campo ${n}: informe o nome`;
    if (f.loinc && f.loinc.trim() && !/^\d{1,7}-\d$/.test(f.loinc.trim())) return `campo ${n}: LOINC no formato 718-7`;
  }
  return null;
}

export function sigtapProblem(code: string): string | null {
  return /^\d{10}$/.test(code.trim()) ? null : "o código SIGTAP tem 10 dígitos";
}

export function cnesProblem(value: string): string | null {
  const v = value.trim();
  return v === "" || /^\d{7}$/.test(v) ? null : "o CNES tem 7 dígitos";
}

export interface ResultDraft { uploadId: string | null; releasedOn: string; laboratoryName: string; critical: boolean; values: Record<string, string> }

export function emptyResultDraft(today: string): ResultDraft {
  return { uploadId: null, releasedOn: today, laboratoryName: "", critical: false, values: {} };
}

export function resultProblem(d: ResultDraft, today: string): string | null {
  if (!d.uploadId) return "anexe o laudo";
  if (!d.releasedOn) return "informe a data de liberação do laudo";
  if (d.releasedOn > today) return "a data de liberação não pode ser no futuro";
  if (!d.laboratoryName.trim()) return "informe o laboratório";
  return null;
}

export function resultBody(d: ResultDraft): ResultInput {
  return {
    upload_id: d.uploadId as string, released_on: d.releasedOn, laboratory_name: d.laboratoryName.trim(), critical: d.critical,
    values: Object.entries(d.values).map(([ key, value ]) => ({ key, value: value.trim() })).filter((v) => v.value !== "")
  };
}

export interface ScheduleDraft { providerId: string; at: string; place: string }

export function scheduleProblem(d: ScheduleDraft, needsProvider: boolean): string | null {
  if (needsProvider && !d.providerId) return "escolha o prestador";
  if (!d.at || !cityLocalIso(d.at)) return "informe data e hora";
  if (!d.place.trim()) return "informe o local";
  return null;
}

export function scheduleWhen(d: ScheduleDraft): ScheduleWhen {
  return { scheduled_at: cityLocalIso(d.at) as string, scheduled_place: d.place.trim() };
}

// `providerId`: o prestador já definido (autorização ou guia); senão o escolhido.
export function scheduleBody(d: ScheduleDraft, providerId?: string): ScheduleInput {
  return { provider_id: providerId ?? d.providerId, ...scheduleWhen(d) };
}

export interface GuideDraft {
  sigtapCode: string; priority: ExamPriority; cid10: string; requesterMode: "linked" | "typed";
  requesterId: string; requesterName: string; requesterCns: string; externalReference: string; guideUploadId: string | null;
}

export const EMPTY_GUIDE: GuideDraft = {
  sigtapCode: "", priority: "routine", cid10: "", requesterMode: "linked", requesterId: "", requesterName: "", requesterCns: "",
  externalReference: "", guideUploadId: null
};

const normalizeCid = (text: string) => text.trim().toUpperCase().replace(/\./g, "");

export function guideProblem(d: GuideDraft): string | null {
  if (!d.sigtapCode) return "escolha o exame";
  const cid = cid10Problem(d.cid10);
  if (cid) return cid;
  if (d.requesterMode === "linked" && !d.requesterId) return "escolha o profissional que pediu";
  if (d.requesterMode === "typed") {
    if (!d.requesterName.trim()) return "informe o nome de quem pediu";
    if (!isValidCns(d.requesterCns)) return "CNS de quem pediu inválido";
  }
  if (!d.guideUploadId) return "anexe a foto ou o PDF da guia";
  return null;
}

export function guideBody(d: GuideDraft, patientId: string, unitId: string): ReceptionOrderInput {
  const cid = normalizeCid(d.cid10);
  const ref = d.externalReference.trim();
  return {
    patient_id: patientId, unit_id: unitId, sigtap_code: d.sigtapCode, priority: d.priority,
    ...(cid ? { cid10_justification: cid } : {}),
    requester: d.requesterMode === "linked"
      ? { user_id: d.requesterId }
      : { name: d.requesterName.trim(), cns: d.requesterCns.replace(/\D/g, "") },
    ...(ref ? { external_reference: ref } : {}),
    guide_upload_id: d.guideUploadId as string
  };
}

// Um exame que muda entre duas páginas não aparece duas vezes: vale a primeira.
export function mergeById<T extends { id: string }>(pages: { orders: T[] }[]): T[] {
  const seen = new Set<string>();
  const out: T[] = [];
  for (const page of pages) {
    for (const item of page.orders) {
      if (seen.has(item.id)) continue;
      seen.add(item.id);
      out.push(item);
    }
  }
  return out;
}

export async function refreshExams(client: QueryClient): Promise<void> {
  await Promise.all(MOVING_KEYS.map((key) => client.invalidateQueries({ queryKey: [ key ] })));
}

export function pendingNoticeText(pending: number, denied: number): string {
  const parts: string[] = [];
  if (pending > 0) parts.push(pending === 1 ? "1 resultado aguardando sua ciência" : `${pending} resultados aguardando sua ciência`);
  if (denied > 0) parts.push(denied === 1 ? "1 exame negado" : `${denied} exames negados`);
  return parts.join(" · ");
}

// Divergência D11: o api recusa o `pdf_url` vencido com 403 ou 410.
export function pdfLinkExpired(err: unknown): boolean {
  return err instanceof ApiError && (err.status === 410 || err.status === 403 || errorCode(err) === "link_expired");
}

const ERRORS: Record<string, string> = {
  invalid_transition: "a situação do exame mudou — atualize para ver a atual",
  reason_required: "informe o motivo",
  provider_not_enabled_for_exam: "este prestador não faz este exame — escolha outro ou habilite-o no cadastro",
  result_already_registered: "o resultado deste exame já foi registrado",
  missing_role: "seu papel não permite esta ação",
  not_requester: "só quem pediu o exame pode fazer isto",
  not_in_team: "este exame é da fila de outra equipe",
  out_of_context: "esta pessoa não está em atendimento com você",
  opening_required: OPENING_ENDED,
  sigtap_not_in_catalog: "este exame não está no catálogo da cidade",
  invalid_cns: "CNS de quem pediu inválido",
  guide_missing: "anexe a foto ou o PDF da guia",
  invalid_file: "arquivo inválido — envie PDF, JPEG ou PNG de até 10 MB",
  sigtap_unknown: "código SIGTAP não encontrado na competência vigente",
  duplicate_sigtap: "este exame já está no catálogo",
  invalid_deadline: "prazo inválido — use dias inteiros de 1 a 365",
  patient_not_found: "nenhum paciente com este CPF na cidade",
  citizen_not_verified: "este CPF ainda não tem cadastro validado — faça a validação presencial no balcão antes",
  not_unit_staff: "você não atua nesta unidade",
  not_found: "exame não encontrado — atualize a lista",
  link_expired: PDF_LINK_EXPIRED
};

export function examError(err: unknown): string {
  if (featureDisabledKey(err) !== null) return EXAMS_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  if (err instanceof ApiError) return `não foi possível concluir (${err.status}) — tente de novo`;
  return "sem conexão com o servidor — tente de novo";
}
```

```ts
// src/hooks/useExamOrderPages.ts
// Lista de exames com "Carregar mais" pelo cursor do api (contrato §3, §4, §6).
// Dado de saúde: nada fica no cache depois que a tela sai (gcTime: 0).
import { useInfiniteQuery } from "@tanstack/react-query";
import { mergeById } from "../lib/exams";

// `P` é a página do api (ExamOrderPage, ProviderOrderPage…): quem chama lê
// o resto da página (ex.: overdue_count) em `query.data.pages`.
export function useExamOrderPages<P extends { orders: { id: string }[]; next_before: string | null }>(
  queryKey: readonly unknown[],
  fetchPage: (before: string | null) => Promise<P>,
  enabled = true
) {
  const query = useInfiniteQuery({
    queryKey,
    queryFn: ({ pageParam }) => fetchPage(pageParam),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.next_before ?? undefined,
    enabled, gcTime: 0, staleTime: 0, retry: false
  });
  const orders: P["orders"][number][] = mergeById<P["orders"][number]>(query.data?.pages ?? []);
  return { query, orders };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/exams.test.ts && npx tsc --noEmit`
Expected: PASS. (`cityLocalIso("2026-10-20T08:30")` dá `-03:00`: o Brasil não tem horário de verão desde 2019.)

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/exams.ts src/lib/exams.test.ts src/hooks/useExamOrderPages.ts
$G commit -m "feat: add exam tracking screen rules and paged order lists

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Exames" no shell, trava do prestador, contador e aviso no Início

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts` (duas contagens), `src/shell/catalog.ts`, `src/App.tsx`, `src/App.navigation.test.tsx`
- Create: `src/hooks/useExamPending.ts`, `src/modules/exams/AcknowledgementNotice.tsx`, `src/modules/exams/styles.ts`
- Test: `src/shell/modules.exams.test.ts`, `src/modules/exams/AcknowledgementNotice.test.tsx`

**Interfaces:**
- Consumes: `hasFeature` (`lib/features`), `listPendingAcknowledgements` (Task 1), `EXAM_PENDING_KEY`, `pendingNoticeText`, `canUseExams`, `examRoles` (Task 2).
- Produces: `ModuleId` com `"exam-acknowledgements" | "exams-unit" | "exams-overdue" | "exam-regulation" | "exam-report" | "exams-admin" | "exam-provider"`; grupo `"Exames"` em `NAV_GROUPS` (logo depois de "Atendimento"); `withExamPendingCount(groups, count)`; `shellGroups(groups, pending, examPending = 0)`; `useExamPending(enabled: boolean, userId: string | null)` (chave `[EXAM_PENDING_KEY, userId]`, a cada 60 s); `<AcknowledgementNotice pending denied onOpen />`; estilos compartilhados em `src/modules/exams/styles.ts` (`panel`, `box`, `form`, `row`, `grid`, `stack`, `fieldLabel`, `checkLabel`, `fieldset`, `legend`, `muted`, `alert`, `okText`, `list`, `title`). Cada tela nova entra no `renderModule` na task dela; até lá o `default` mostra o `Placeholder`.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.exams.test.ts
import { describe, expect, it } from "vitest";
import { navGroupsFor, withExamPendingCount, NAV_GROUPS } from "./modules";

const ON = { features: [ "exam_tracking" ] };
const userWith = (roles: string[], extra: object = ON) => ({ operator: false, memberships: roles.map((role) => ({ role })), ...extra });
const ids = (user: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(user).flatMap((g) => g.items.map((i) => i.id));
const EXAM_IDS = [ "exam-acknowledgements", "exams-unit", "exams-overdue", "exam-regulation", "exam-report", "exams-admin", "exam-provider" ];
const examIds = (user: Parameters<typeof navGroupsFor>[0]) => ids(user).filter((id) => EXAM_IDS.includes(id));

describe("grupo Exames (módulo 21a)", () => {
  it("sem exam_tracking, nenhum item de Exames para ninguém", () => {
    for (const role of [ "municipal_admin", "health_professional", "citizen_verifier", "exam_regulator" ]) {
      expect(examIds(userWith([ role ], {}))).toEqual([]);
    }
    expect(navGroupsFor(userWith([ "municipal_admin" ], {})).some((g) => g.label === "Exames")).toBe(false);
  });

  it("cada papel vê o seu", () => {
    expect(examIds(userWith([ "health_professional" ]))).toEqual([ "exam-acknowledgements", "exams-unit" ]);
    expect(examIds(userWith([ "citizen_verifier" ]))).toEqual([ "exams-unit" ]);
    expect(examIds(userWith([ "exam_regulator" ]))).toEqual([ "exams-overdue", "exam-regulation", "exam-report" ]);
    expect(examIds(userWith([ "municipal_admin" ]))).toEqual([ "exams-unit", "exams-overdue", "exam-regulation", "exam-report", "exams-admin" ]);
    expect(examIds({ operator: true, memberships: [], ...ON })).toEqual([]);
  });

  it("o prestador vê só o portal e a Segurança", () => {
    const provider = userWith([ "exam_provider" ]);
    expect(navGroupsFor(provider).map((g) => g.label)).toEqual([ "Exames", "Conta" ]);
    expect(ids(provider)).toEqual([ "exam-provider", "security" ]);
  });

  it("prestador com o rastreamento desligado vê só a Segurança", () => {
    expect(ids(userWith([ "exam_provider" ], {}))).toEqual([ "security" ]);
  });

  it("contador de ciência no rótulo; 200 ou mais vira 200+", () => {
    const item = (n: number) => withExamPendingCount(NAV_GROUPS, n).flatMap((g) => g.items).find((i) => i.id === "exam-acknowledgements");
    expect(item(0)?.label).toBe("Resultados aguardando ciência");
    expect(item(4)?.label).toBe("Resultados aguardando ciência (4)");
    expect(item(250)?.label).toBe("Resultados aguardando ciência (200+)");
  });
});
```

```tsx
// src/modules/exams/AcknowledgementNotice.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { AcknowledgementNotice } from "./AcknowledgementNotice";

afterEach(cleanup);

describe("AcknowledgementNotice", () => {
  it("diz quantos resultados e negados e abre a tela", () => {
    const onOpen = vi.fn();
    render(<AcknowledgementNotice pending={2} denied={1} onOpen={onOpen} />);
    expect(screen.getByRole("status").textContent).toBe("2 resultados aguardando sua ciência · 1 exame negado");
    fireEvent.click(screen.getByRole("button", { name: "Ver resultados" }));
    expect(onOpen).toHaveBeenCalledTimes(1);
  });

  it("nada pendente: não aparece", () => {
    render(<AcknowledgementNotice pending={0} denied={0} onOpen={vi.fn()} />);
    expect(screen.queryByRole("region", { name: "Seus exames" })).toBeNull();
  });
});
```

Acrescente ao `describe("grupos do shell", …)` de `src/App.navigation.test.tsx`:

```ts
  it("Resultados aguardando ciência leva a contagem no rótulo e no ponto do trilho", () => {
    const item = shellGroups(NAV_GROUPS, 0, 4).flatMap((g) => g.items).find((i) => i.id === "exam-acknowledgements");
    expect(item).toEqual({ id: "exam-acknowledgements", label: "Resultados aguardando ciência (4)", count: 4 });
  });
```

E, em `src/shell/modules.test.ts`, as duas contagens que mudam com o grupo novo (que some para o operador e para quem ainda não tem sessão):

```ts
      expect(groups.length).toBe(NAV_GROUPS.length - 8);   // era - 7 (operador)
```
```ts
      expect(groups.length).toBe(NAV_GROUPS.length - 7);   // era - 6 (sem sessão)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/shell/modules.exams.test.ts src/modules/exams/AcknowledgementNotice.test.tsx src/App.navigation.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `withExamPendingCount` não existe, o grupo não existe, o componente não existe e as contagens novas não batem.

- [ ] **Step 3: Write minimal implementation**

Em `src/shell/modules.ts`:

```ts
  | "signature" | "signature-pending" | "signature-overview" | "clinical-documents-admin" | "sncr" | "sncr-overview"
  | "exam-acknowledgements" | "exams-unit" | "exams-overdue" | "exam-regulation" | "exam-report" | "exams-admin" | "exam-provider";
```

Em `NAV_GROUPS`, logo depois do grupo "Atendimento" (antes de "Comunicação"):

```ts
  { label: "Exames", items: [
    { id: "exam-acknowledgements", label: "Resultados aguardando ciência", icon: "⚑" },
    { id: "exams-unit", label: "Exames da unidade", icon: "⚗" },
    { id: "exams-overdue", label: "Exames atrasados", icon: "⏱" },
    { id: "exam-regulation", label: "Fila de regulação", icon: "⚖" },
    { id: "exam-report", label: "Conferência de exames", icon: "▤" },
    { id: "exams-admin", label: "Catálogo e prestadores", icon: "⚙" },
    { id: "exam-provider", label: "Portal do prestador", icon: "⇆" }
  ]},
```

Em `navGroupsFor`, logo depois de `const roles = …`:

```ts
  // Módulo 21a (spec §6): o usuário do prestador vê só o portal e a
  // Segurança — nada de painel, atendimento ou prontuário. O api isola de
  // novo (403/404); aqui só não se oferece a porta.
  if (!user?.operator && roles.includes("exam_provider")) {
    return NAV_GROUPS.flatMap((group) => {
      const items = group.items.filter((item) =>
        item.id === "security" || (item.id === "exam-provider" && hasFeature(user, "exam_tracking")));
      return items.length > 0 ? [ { ...group, items } ] : [];
    });
  }
```

Logo depois de `const canClinicalRecord = …;`:

```ts
  // Módulo 21a: tudo de exames só com `exam_tracking` ligado na sessão.
  const examsOn = !user?.operator && hasFeature(user, "exam_tracking");
  const isRegulator = roles.includes("exam_regulator");
  const isReception = roles.includes("citizen_verifier");
```

No filtro de itens, antes de `if (item.id === "integrations" || item.id === "cnes") return canIntegrations;`:

```ts
      if (item.id === "exam-acknowledgements") return examsOn && isProfessional;
      if (item.id === "exams-unit") return examsOn && (isProfessional || isReception || isAdmin);
      if (item.id === "exams-overdue" || item.id === "exam-regulation" || item.id === "exam-report") return examsOn && (isAdmin || isRegulator);
      if (item.id === "exams-admin") return examsOn && isAdmin;
      // O portal só aparece pelo ramo do prestador, acima.
      if (item.id === "exam-provider") return false;
```

No fim do arquivo:

```ts
// Módulo 21a: contador de resultados aguardando ciência no item do menu.
export function withExamPendingCount(groups: NavGroupDef[], count: number): NavGroupDef[] {
  if (count <= 0) return groups;
  const shown = count >= 200 ? "200+" : String(count);
  return groups.map((group) => ({
    ...group,
    items: group.items.map((item) => item.id === "exam-acknowledgements" ? { ...item, label: `${item.label} (${shown})` } : item)
  }));
}
```

Em `src/shell/catalog.ts`, acrescente ao `import` `IconAlarm, IconChecklist, IconGavel, IconMicroscope, IconReportAnalytics, IconSettings, IconTestPipe, IconTruckDelivery` (em ordem alfabética na lista existente) e:

```ts
    "Exames":      { icon: IconMicroscope, description: "Pedidos, resultados e prestadores" },
```
(em `groups`, depois de "Atendimento") e, em `screens`:

```ts
    "exam-acknowledgements": { icon: IconChecklist, description: "Resultados dos seus pedidos e da equipe" },
    "exams-unit": { icon: IconTestPipe, description: "Guias, agendamento, coleta e resultado" },
    "exams-overdue": { icon: IconAlarm, description: "Exames fora do prazo, por etapa" },
    "exam-regulation": { icon: IconGavel, description: "Autorizar ou negar exames" },
    "exam-report": { icon: IconReportAnalytics, description: "Guias por prestador e mês" },
    "exams-admin": { icon: IconSettings, description: "Exames da cidade, prazos e prestadores" },
    "exam-provider": { icon: IconTruckDelivery, description: "Guias do seu laboratório" },
```

```ts
// src/hooks/useExamPending.ts
// Resultados aguardando a ciência de quem pediu e os negados (módulo 21a;
// contrato §3, Divergência D7). O aviso do Início, o contador do menu e a
// tela leem a mesma chave: uma requisição por minuto.
import { useQuery } from "@tanstack/react-query";
import { listPendingAcknowledgements } from "../lib/api";
import { EXAM_PENDING_KEY } from "../lib/exams";

export const EXAM_PENDING_REFRESH_MS = 60_000;

export function useExamPending(enabled: boolean, userId: string | null) {
  return useQuery({
    queryKey: [ EXAM_PENDING_KEY, userId ],
    queryFn: listPendingAcknowledgements,
    enabled,
    refetchInterval: enabled ? EXAM_PENDING_REFRESH_MS : false,
    gcTime: 0, retry: false
  });
}
```

```ts
// src/modules/exams/styles.ts
// Estilos comuns das telas de exames (só papéis do tema).
import type { CSSProperties } from "react";

export const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 12, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
export const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 12, border: "1px solid var(--rule)", borderRadius: 8, background: "var(--sunken)" };
export const form: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, maxWidth: 640 };
export const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
export const grid: CSSProperties = { display: "flex", gap: 24, flexWrap: "wrap" };
export const stack: CSSProperties = { display: "flex", flexDirection: "column", gap: 16 };
export const fieldLabel: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
export const checkLabel: CSSProperties = { display: "flex", gap: 6, alignItems: "center", fontSize: 12.5, color: "var(--ink)" };
export const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, border: "1px solid var(--rule)", borderRadius: 8, padding: 12 };
export const legend: CSSProperties = { fontSize: 12.5, fontWeight: 600 };
export const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
export const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
export const okText: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--ok)" };
export const list: CSSProperties = { margin: 0, paddingLeft: 18, fontSize: 12.5, display: "flex", flexDirection: "column", gap: 4 };
export const title: CSSProperties = { margin: 0, fontSize: 14 };
```

```tsx
// src/modules/exams/AcknowledgementNotice.tsx
// Aviso no Início para quem pede exames (spec §5 "Alertas"): resultados
// aguardando a ciência dele e exames negados pela regulação.
import { pendingNoticeText } from "../../lib/exams";
import { secondaryButtonStyle } from "../../components/formStyles";
import { row } from "./styles";

export function AcknowledgementNotice({ pending, denied, onOpen }: { pending: number; denied: number; onOpen(): void }) {
  const text = pendingNoticeText(pending, denied);
  if (!text) return null;
  return (
    <section aria-label="Seus exames"
      style={{ ...row, gap: 12, padding: 12, marginBottom: 16, border: "1px solid var(--rule)", borderRadius: 8, background: "var(--sunken)" }}>
      <p role="status" style={{ margin: 0, fontSize: 13, color: "var(--ink)" }}>{text}</p>
      <button type="button" style={secondaryButtonStyle} onClick={onOpen}>Ver resultados</button>
    </section>
  );
}
```

Em `src/App.tsx`:

```tsx
import { NAV_GROUPS, labelFor, navGroupsFor, withExamPendingCount, withPendingCount, type ModuleId, type NavGroupDef } from "./shell/modules";
import { useExamPending } from "./hooks/useExamPending";
import { canUseExams, examRoles } from "./lib/exams";
import { AcknowledgementNotice } from "./modules/exams/AcknowledgementNotice";
```

```tsx
// Módulo 21a: `examPending` acende o item "Resultados aguardando ciência".
export function shellGroups(groups: NavGroupDef[], pending: number, examPending = 0): ShellGroup[] {
  return withExamPendingCount(withPendingCount(groups, pending), examPending).map((g) => ({
    label: g.label,
    slug: slugOf(g.label),
    items: g.items.map((i) => {
      if (i.id === "signature-pending" && pending > 0) return { id: i.id, label: i.label, count: pending };
      if (i.id === "exam-acknowledgements" && examPending > 0) return { id: i.id, label: i.label, count: examPending };
      return { id: i.id, label: i.label };
    })
  }));
}
```

Em `ShellInner`, logo depois de `const pending = usePendingSignatures(canSign(auth.user));`:

```tsx
  const examPending = useExamPending(canUseExams(auth.user) && examRoles(auth.user).professional, auth.user?.id ?? null);
  const examCount = examPending.data?.orders.length ?? 0;
  const deniedCount = examPending.data?.denied.length ?? 0;
```

e troque o `useMemo` dos grupos e a linha do Início:

```tsx
  const groups = useMemo(
    () => shellGroups(navGroupsFor(auth.user), pending.data?.length ?? 0, examCount),
    [ auth.user, pending.data?.length, examCount ]
  );
```

```tsx
  if (route.kind === "home") content = (
    <>
      <AcknowledgementNotice pending={examCount} denied={deniedCount} onOpen={() => navigate("exam-acknowledgements")} />
      <HomeCards groups={groups} catalog={CATALOG} go={goShell} />
    </>
  );
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/shell src/App.navigation.test.tsx src/modules/exams/AcknowledgementNotice.test.tsx && npx tsc --noEmit`
Expected: PASS — inclusive `catalog.test.ts` (ícone e descrição das 7 telas novas e do grupo) e os testes antigos de `modules.test.ts` com as duas contagens novas.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.exams.test.ts src/shell/catalog.ts src/App.tsx src/App.navigation.test.tsx \
  src/hooks/useExamPending.ts src/modules/exams/AcknowledgementNotice.tsx src/modules/exams/AcknowledgementNotice.test.tsx src/modules/exams/styles.ts
$G commit -m "feat: add the exams menu group, provider lock and home acknowledgement notice

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 4: Anexo, resultado, agendamento e motivo

**Files:**
- Create: `src/modules/exams/UploadField.tsx`, `src/modules/exams/ResultForm.tsx`, `src/modules/exams/ScheduleForm.tsx`, `src/modules/exams/ReasonForm.tsx`
- Test: `src/modules/exams/UploadField.test.tsx`, `src/modules/exams/forms.test.tsx`

**Interfaces:**
- Consumes: `uploadExamFile`, `EXAM_UPLOAD_TYPES`, `ExamResultField`, `ExamProviderRef`, `ResultInput` (Task 1); `uploadProblem`, `examError`, `emptyResultDraft`, `resultProblem`, `resultBody`, `ResultDraft`, `scheduleProblem`, `ScheduleDraft`, `reasonProblem` (Task 2); estilos de `./styles` (Task 3).
- Produces:
  ```ts
  UploadField(props: { label: string; uploadId: string | null; onUploaded(uploadId: string | null): void })
  ResultForm(props: { fields: ExamResultField[]; today: string; submitLabel: string; allowCritical?: boolean;
    onSubmit(input: ResultInput): Promise<void>; onCancel(): void })        // <form aria-label={submitLabel}>
  ScheduleForm(props: { providers: ExamProviderRef[] | null; onSubmit(d: ScheduleDraft): Promise<void>; onCancel(): void })
                                                                            // <form aria-label="Agendar exame">; providers null = sem escolha
  ReasonForm(props: { label: string; submitLabel: string; onSubmit(reason: string): Promise<void>; onCancel(): void })
                                                                            // <form aria-label={label}>
  ```
  Os formulários mostram a recusa do `onSubmit` (`examError`) e não fecham sozinhos: quem chama fecha no sucesso. "Voltar" é sempre o botão de desistir (nunca "Cancelar", que nomeia a ação "Cancelar exame").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/exams/UploadField.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { useState } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, uploadExamFile: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { UploadField } from "./UploadField";
import { pdfFile } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let last: string | null = "inicial";

function Harness() {
  const [ id, setId ] = useState<string | null>(null);
  return <UploadField label="Laudo" uploadId={id} onUploaded={(next) => { last = next; setId(next); }} />;
}

function pick(file: File) {
  fireEvent.change(screen.getByLabelText("Laudo"), { target: { files: [ file ] } });
}

afterEach(cleanup);

describe("UploadField", () => {
  beforeEach(() => { mocked(api.uploadExamFile).mockReset(); last = "inicial"; });

  it("anexa: mostra o nome só na tela e devolve o upload_id", async () => {
    mocked(api.uploadExamFile).mockResolvedValue("up1");
    render(<Harness />);
    pick(pdfFile("laudo Joana.pdf"));
    expect((await screen.findByText("anexado: laudo Joana.pdf")).getAttribute("role")).toBe("status");
    expect(last).toBe("up1");
    expect(mocked(api.uploadExamFile).mock.calls[0][0].name).toBe("laudo Joana.pdf");
  });

  it("recusa tipo, tamanho e vazio sem chamar o api", async () => {
    render(<Harness />);
    pick(new File([ "x" ], "guia.docx", { type: "application/vnd.openxmlformats-officedocument.wordprocessingml.document" }));
    expect((await screen.findByRole("alert")).textContent).toBe("envie PDF, JPEG ou PNG");
    const big = new File([ "x" ], "foto.jpg", { type: "image/jpeg" });
    Object.defineProperty(big, "size", { value: 12 * 1024 * 1024 });
    pick(big);
    expect((await screen.findByRole("alert")).textContent).toBe("o arquivo passa de 10 MB");
    pick(new File([], "vazio.pdf", { type: "application/pdf" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o arquivo está vazio");
    expect(api.uploadExamFile).not.toHaveBeenCalled();
    expect(last).toBeNull();
  });

  it("422 invalid_file: explica e não guarda o anexo", async () => {
    mocked(api.uploadExamFile).mockRejectedValue(new ApiError(422, { error: "invalid_file" }, "x"));
    render(<Harness />);
    pick(pdfFile());
    expect((await screen.findByRole("alert")).textContent).toBe("arquivo inválido — envie PDF, JPEG ou PNG de até 10 MB");
    expect(last).toBeNull();
  });

  it("trocar o arquivo apaga o anexo anterior antes do novo envio", async () => {
    mocked(api.uploadExamFile).mockResolvedValueOnce("up1").mockReturnValueOnce(new Promise(() => {}));
    render(<Harness />);
    pick(pdfFile("a.pdf"));
    await screen.findByText("anexado: a.pdf");
    pick(pdfFile("b.pdf"));
    await waitFor(() => expect(last).toBeNull());
    expect(screen.queryByText("anexado: a.pdf")).toBeNull();
    expect(screen.getByText("enviando o arquivo…")).not.toBeNull();
  });
});
```

```tsx
// src/modules/exams/forms.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, uploadExamFile: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ResultForm } from "./ResultForm";
import { ScheduleForm } from "./ScheduleForm";
import { ReasonForm } from "./ReasonForm";
import { HEMOGRAM_FIELDS, TODAY21, pdfFile } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("ResultForm", () => {
  beforeEach(() => { mocked(api.uploadExamFile).mockReset(); mocked(api.uploadExamFile).mockResolvedValue("up1"); });

  it("exige o laudo; manda data, laboratório, valores preenchidos e crítico", async () => {
    const onSubmit = vi.fn().mockResolvedValue(undefined);
    render(<ResultForm fields={HEMOGRAM_FIELDS} today={TODAY21} submitLabel="Registrar resultado" onSubmit={onSubmit} onCancel={vi.fn()} />);
    const form = screen.getByRole("form", { name: "Registrar resultado" });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("anexe o laudo");

    fireEvent.change(within(form).getByLabelText("Laudo (PDF, JPEG ou PNG, até 10 MB)"), { target: { files: [ pdfFile() ] } });
    await within(form).findByText("anexado: laudo.pdf");
    fireEvent.change(within(form).getByLabelText("Laboratório"), { target: { value: "Laboratório Simulado" } });
    fireEvent.change(within(form).getByLabelText("Hemoglobina (g/dL)"), { target: { value: "6.9" } });
    fireEvent.click(within(form).getByLabelText("Resultado crítico (vai direto para a fila da equipe)"));
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
      upload_id: "up1", released_on: TODAY21, laboratory_name: "Laboratório Simulado", critical: true,
      values: [ { key: "hemoglobina", value: "6.9" } ]
    }));
  });

  it("data de liberação no futuro não envia", async () => {
    const onSubmit = vi.fn();
    render(<ResultForm fields={[]} today={TODAY21} submitLabel="Registrar resultado" onSubmit={onSubmit} onCancel={vi.fn()} />);
    const form = screen.getByRole("form", { name: "Registrar resultado" });
    fireEvent.change(within(form).getByLabelText("Laudo (PDF, JPEG ou PNG, até 10 MB)"), { target: { files: [ pdfFile() ] } });
    await within(form).findByText("anexado: laudo.pdf");
    fireEvent.change(within(form).getByLabelText("Data de liberação"), { target: { value: "2026-10-14" } });
    fireEvent.change(within(form).getByLabelText("Laboratório"), { target: { value: "Lab" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("a data de liberação não pode ser no futuro");
    expect(onSubmit).not.toHaveBeenCalled();
    expect(within(form).getByText("este exame não tem campos no catálogo — o laudo basta")).not.toBeNull();
  });

  it("resultado externo: sem a marca de crítico; a recusa do api aparece no formulário", async () => {
    const onSubmit = vi.fn().mockRejectedValue(new ApiError(409, { error: "result_already_registered" }, "x"));
    render(<ResultForm fields={[]} today={TODAY21} submitLabel="Registrar resultado externo" allowCritical={false} onSubmit={onSubmit} onCancel={vi.fn()} />);
    const form = screen.getByRole("form", { name: "Registrar resultado externo" });
    expect(within(form).queryByLabelText("Resultado crítico (vai direto para a fila da equipe)")).toBeNull();
    fireEvent.change(within(form).getByLabelText("Laudo (PDF, JPEG ou PNG, até 10 MB)"), { target: { files: [ pdfFile() ] } });
    await within(form).findByText("anexado: laudo.pdf");
    fireEvent.change(within(form).getByLabelText("Laboratório"), { target: { value: "Lab Externo" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado externo" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("o resultado deste exame já foi registrado");
  });
});

describe("ScheduleForm", () => {
  it("com lista: exige o prestador; manda o rascunho", async () => {
    const onSubmit = vi.fn().mockResolvedValue(undefined);
    render(<ScheduleForm providers={[ { id: "pv1", name: "Laboratório Simulado" } ]} onSubmit={onSubmit} onCancel={vi.fn()} />);
    const form = screen.getByRole("form", { name: "Agendar exame" });
    fireEvent.change(within(form).getByLabelText("Data e hora"), { target: { value: "2026-10-20T08:30" } });
    fireEvent.change(within(form).getByLabelText("Local"), { target: { value: "Rua XV, 100" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar agendamento" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("escolha o prestador");
    fireEvent.change(within(form).getByLabelText("Prestador"), { target: { value: "pv1" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar agendamento" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({ providerId: "pv1", at: "2026-10-20T08:30", place: "Rua XV, 100" }));
  });

  it("prestador já definido: sem seletor; lista vazia explica o que fazer", () => {
    render(<ScheduleForm providers={null} onSubmit={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.queryByLabelText("Prestador")).toBeNull();
    cleanup();
    render(<ScheduleForm providers={[]} onSubmit={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.getByRole("alert").textContent).toBe("nenhum prestador ativo faz este exame — cadastre em Catálogo e prestadores");
  });
});

describe("ReasonForm", () => {
  it("motivo curto não envia; o motivo vai sem espaços nas pontas", async () => {
    const onSubmit = vi.fn().mockResolvedValue(undefined);
    render(<ReasonForm label="Motivo do cancelamento" submitLabel="Cancelar exame" onSubmit={onSubmit} onCancel={vi.fn()} />);
    const form = screen.getByRole("form", { name: "Motivo do cancelamento" });
    fireEvent.click(within(form).getByRole("button", { name: "Cancelar exame" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("escreva o motivo (ao menos 5 caracteres)");
    fireEvent.change(within(form).getByLabelText("Motivo do cancelamento"), { target: { value: "  pedido em duplicidade " } });
    fireEvent.click(within(form).getByRole("button", { name: "Cancelar exame" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith("pedido em duplicidade"));
    expect(within(form).getByRole("button", { name: "Voltar" })).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/exams/UploadField.test.tsx src/modules/exams/forms.test.tsx`
Expected: FAIL — os componentes não existem.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/exams/UploadField.tsx
// Anexo de guia ou laudo (contrato §3; Divergência D1). Tipo e tamanho são
// conferidos aqui e de novo no api. O arquivo sobe sem o nome, que pode
// trazer o nome do paciente: o nome fica só na tela de quem anexou. Trocar o
// arquivo apaga o anexo anterior; uma resposta atrasada de um arquivo
// trocado é ignorada.
import { useRef, useState } from "react";
import { EXAM_UPLOAD_TYPES, uploadExamFile } from "../../lib/api";
import { examError, uploadProblem } from "../../lib/exams";
import { inputStyle } from "../../components/formStyles";
import { alert, fieldLabel, muted, okText } from "./styles";

type State = { kind: "idle" } | { kind: "sending" } | { kind: "error"; text: string };

export function UploadField({ label, uploadId, onUploaded }: { label: string; uploadId: string | null; onUploaded(uploadId: string | null): void }) {
  const [ state, setState ] = useState<State>({ kind: "idle" });
  const [ name, setName ] = useState<string | null>(null);
  const seq = useRef(0);

  async function pick(file: File | null) {
    const mine = ++seq.current;
    onUploaded(null);
    setName(null);
    const problem = uploadProblem(file);
    if (problem || !file) { setState({ kind: "error", text: problem ?? "escolha um arquivo" }); return; }
    setState({ kind: "sending" });
    try {
      const id = await uploadExamFile(file);
      if (mine !== seq.current) return;
      setName(file.name);
      setState({ kind: "idle" });
      onUploaded(id);
    } catch (err) {
      if (mine !== seq.current) return;
      setState({ kind: "error", text: examError(err) });
    }
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 4 }}>
      <label style={fieldLabel}>
        {label}
        <input type="file" accept={EXAM_UPLOAD_TYPES.join(",")} style={inputStyle}
          onChange={(e) => void pick(e.target.files?.[0] ?? null)} />
      </label>
      {state.kind === "sending" && <p role="status" style={muted}>enviando o arquivo…</p>}
      {state.kind === "error" && <p role="alert" style={alert}>{state.text}</p>}
      {uploadId && name && <p role="status" style={okText}>{`anexado: ${name}`}</p>}
    </div>
  );
}
```

```tsx
// src/modules/exams/ResultForm.tsx
// Resultado do exame (contrato §3 "result" e §6; resultado externo §3):
// laudo, data de liberação, laboratório, os campos do catálogo (todos
// opcionais) e a marca "crítico" — que manda o exame direto para a fila da
// equipe (spec §5). O resultado externo não tem a marca.
import { useState, type FormEvent } from "react";
import type { ExamResultField, ResultInput } from "../../lib/api";
import { emptyResultDraft, examError, resultBody, resultProblem, type ResultDraft } from "../../lib/exams";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { UploadField } from "./UploadField";
import { alert, checkLabel, fieldLabel, fieldset, form, legend, muted, row } from "./styles";

interface Props {
  fields: ExamResultField[]; today: string; submitLabel: string; allowCritical?: boolean;
  onSubmit(input: ResultInput): Promise<void>; onCancel(): void;
}

export function ResultForm({ fields, today, submitLabel, allowCritical = true, onSubmit, onCancel }: Props) {
  const [ draft, setDraft ] = useState<ResultDraft>(() => emptyResultDraft(today));
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const set = (patch: Partial<ResultDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const problem = resultProblem(draft, today);
    if (problem) { setError(problem); return; }
    setBusy(true);
    setError(null);
    try { await onSubmit(resultBody(draft)); } catch (err) { setError(examError(err)); } finally { setBusy(false); }
  }

  return (
    <form aria-label={submitLabel} onSubmit={submit} style={form}>
      <UploadField label="Laudo (PDF, JPEG ou PNG, até 10 MB)" uploadId={draft.uploadId} onUploaded={(uploadId) => set({ uploadId })} />
      <label style={fieldLabel}>
        Data de liberação
        <input type="date" max={today} value={draft.releasedOn} style={inputStyle} onChange={(e) => set({ releasedOn: e.target.value })} />
      </label>
      <label style={fieldLabel}>
        Laboratório
        <input value={draft.laboratoryName} style={inputStyle} onChange={(e) => set({ laboratoryName: e.target.value })} />
      </label>
      {fields.length > 0 ? (
        <fieldset style={fieldset}>
          <legend style={legend}>Valores (opcionais)</legend>
          {fields.map((f) => (
            <label key={f.key} style={fieldLabel}>
              {f.unit ? `${f.label} (${f.unit})` : f.label}
              <input value={draft.values[f.key] ?? ""} style={inputStyle}
                onChange={(e) => { const value = e.target.value; setDraft((d) => ({ ...d, values: { ...d.values, [f.key]: value } })); }} />
            </label>
          ))}
        </fieldset>
      ) : <p style={muted}>este exame não tem campos no catálogo — o laudo basta</p>}
      {allowCritical && (
        <label style={checkLabel}>
          <input type="checkbox" checked={draft.critical} onChange={(e) => set({ critical: e.target.checked })} />
          Resultado crítico (vai direto para a fila da equipe)
        </label>
      )}
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <button type="submit" aria-disabled={busy || undefined} style={busy ? disabledButtonStyle : buttonStyle}>{submitLabel}</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/exams/ScheduleForm.tsx
// Agendamento (contrato §3 "schedule" e §6): data e hora na hora da cidade
// e o local. `providers` null = o prestador já está definido (autorizado pela
// regulação, ou é o próprio prestador no portal); lista = a unidade escolhe.
import { useState, type FormEvent } from "react";
import type { ExamProviderRef } from "../../lib/api";
import { examError, scheduleProblem, type ScheduleDraft } from "../../lib/exams";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, form, row } from "./styles";

export function ScheduleForm({ providers, onSubmit, onCancel }: {
  providers: ExamProviderRef[] | null; onSubmit(d: ScheduleDraft): Promise<void>; onCancel(): void;
}) {
  const [ draft, setDraft ] = useState<ScheduleDraft>({ providerId: "", at: "", place: "" });
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const set = (patch: Partial<ScheduleDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const problem = scheduleProblem(draft, providers !== null);
    if (problem) { setError(problem); return; }
    setBusy(true);
    setError(null);
    try { await onSubmit(draft); } catch (err) { setError(examError(err)); } finally { setBusy(false); }
  }

  return (
    <form aria-label="Agendar exame" onSubmit={submit} style={form}>
      {providers !== null && providers.length === 0 && (
        <p role="alert" style={alert}>nenhum prestador ativo faz este exame — cadastre em Catálogo e prestadores</p>
      )}
      {providers !== null && providers.length > 0 && (
        <label style={fieldLabel}>
          Prestador
          <select value={draft.providerId} style={inputStyle} onChange={(e) => set({ providerId: e.target.value })}>
            <option value="">escolha…</option>
            {providers.map((p) => <option key={p.id} value={p.id}>{p.name}</option>)}
          </select>
        </label>
      )}
      <label style={fieldLabel}>
        Data e hora
        <input type="datetime-local" value={draft.at} style={inputStyle} onChange={(e) => set({ at: e.target.value })} />
      </label>
      <label style={fieldLabel}>
        Local
        <input value={draft.place} placeholder="endereço ou nome do posto de coleta" style={inputStyle} onChange={(e) => set({ place: e.target.value })} />
      </label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <button type="submit" aria-disabled={busy || undefined} style={busy ? disabledButtonStyle : buttonStyle}>Confirmar agendamento</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/exams/ReasonForm.tsx
// Motivo obrigatório (contrato §3–§4): cancelar, negar e a ciência pela equipe.
import { useState, type FormEvent } from "react";
import { examError, reasonProblem } from "../../lib/exams";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, form, row } from "./styles";

export function ReasonForm({ label, submitLabel, onSubmit, onCancel }: {
  label: string; submitLabel: string; onSubmit(reason: string): Promise<void>; onCancel(): void;
}) {
  const [ text, setText ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const problem = reasonProblem(text);
    if (problem) { setError(problem); return; }
    setBusy(true);
    setError(null);
    try { await onSubmit(text.trim()); } catch (err) { setError(examError(err)); } finally { setBusy(false); }
  }

  return (
    <form aria-label={label} onSubmit={submit} style={form}>
      <label style={fieldLabel}>
        {label}
        <textarea rows={3} value={text} style={inputStyle} onChange={(e) => setText(e.target.value)} />
      </label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <button type="submit" aria-disabled={busy || undefined} style={busy ? disabledButtonStyle : buttonStyle}>{submitLabel}</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
      </div>
    </form>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/exams/UploadField.test.tsx src/modules/exams/forms.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/exams/UploadField.tsx src/modules/exams/ResultForm.tsx src/modules/exams/ScheduleForm.tsx src/modules/exams/ReasonForm.tsx \
  src/modules/exams/UploadField.test.tsx src/modules/exams/forms.test.tsx
$G commit -m "feat: add exam upload, result, schedule and reason forms

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: O exame — andamento, resultado, laudo e ações

**Files:**
- Create: `src/modules/exams/ExamOrderDetail.tsx`, `src/modules/exams/ExamResultBlock.tsx`, `src/modules/exams/openExamPdf.ts`
- Test: `src/modules/exams/ExamOrderDetail.test.tsx`

**Interfaces:**
- Consumes: `getExamOrder`, `listEnabledProviders`, `scheduleExam`, `collectExam`, `registerExamResult`, `cancelExam`, `acknowledgeExam`, `fetchExamPdf`, `errorCode` (Task 1); de `lib/exams`: `EXAM_ORDER_KEY`, `EXAM_PROVIDER_REFS_KEY`, `ORIGIN_LABEL`, `SOURCE_LABEL`, `eventLabel`, `examActions`, `examError`, `examRoles`, `examStatusLabel`, `examStatusTone`, `fmtDay`, `overdueLabel`, `refreshExams`, `scheduleBody`, `PDF_LINK_EXPIRED`, `pdfLinkExpired` (Task 2); `ResultForm`, `ScheduleForm`, `ReasonForm` (Task 4).
- Produces:
  ```ts
  ExamOrderDetail(props: { id: string; onClose(): void; onChanged?(): void })   // <section aria-label={`Exame ${label}`}>
  ExamResultBlock(props: { result: ExamResult; onExpired(): void })             // <section aria-label="Resultado">
  openExamPdf(pdfUrl: string): Promise<{ ok: true } | { ok: false; expired: boolean; text: string }>
  ```
  Toda escrita termina com `refreshExams(queryClient)` (listas, pendentes, fila e o próprio detalhe). 409 `invalid_transition` fecha o formulário, mostra a frase e relê o exame.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/exams/ExamOrderDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getExamOrder: vi.fn(), listEnabledProviders: vi.fn(), scheduleExam: vi.fn(),
    collectExam: vi.fn(), registerExamResult: vi.fn(), cancelExam: vi.fn(), acknowledgeExam: vi.fn(), fetchExamPdf: vi.fn(),
    uploadExamFile: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ExamOrderDetail } from "./ExamOrderDetail";
import { renderWithProviders } from "../../test/campaignFixtures";
import { NOW21, examOrder, examResult, examUser } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.getExamOrder, api.listEnabledProviders, api.scheduleExam, api.collectExam,
  api.registerExamResult, api.cancelExam, api.acknowledgeExam, api.fetchExamPdf, api.uploadExamFile ];
let win: { location: { href: string }; close: ReturnType<typeof vi.fn> };

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

function sessionAs(roles: string[], id = "us1") { mocked(api.fetchCurrentSession).mockResolvedValue(examUser(roles, { id })); }
async function detail() { return screen.findByRole("region", { name: "Exame Hemograma completo" }); }

describe("ExamOrderDetail", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW21));
    for (const fn of ALL) mocked(fn).mockReset();
    sessionAs([ "health_professional" ]);
    win = { location: { href: "" }, close: vi.fn() };
    vi.spyOn(window, "open").mockImplementation(() => win as unknown as Window);
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:laudo-1") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
  });

  it("mostra situação, andamento com motivo e o resultado com os valores", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder({
      status: "resulted", result: examResult({ critical: true }),
      events: [
        { kind: "requested", occurred_at: "2026-10-10T09:00:00-03:00", actor: { type: "user", name: "Dra. Ana Souza" }, reason: null },
        { kind: "escalated", occurred_at: "2026-10-12T08:00:00-03:00", actor: { type: "user", name: "Sistema" }, reason: "critical" }
      ]
    }));
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    const region = await detail();
    expect(within(region).getByText("resultado registrado")).not.toBeNull();
    const steps = within(within(region).getByRole("list", { name: "andamento" })).getAllByRole("listitem").map((li) => li.textContent);
    expect(steps).toEqual([ "10/10/2026, 09:00 · solicitado · Dra. Ana Souza", "12/10/2026, 08:00 · enviado à equipe · Sistema — critical" ]);
    const result = within(region).getByRole("region", { name: "Resultado" });
    expect(within(result).getByText("Resultado liberado em 12/10/2026")).not.toBeNull();
    expect(within(result).getByText("crítico")).not.toBeNull();
    expect(within(result).getByText("13.5 g/dL")).not.toBeNull();
  });

  it("recepção agenda um exame sem prestador: escolhe da lista e manda a hora da cidade", async () => {
    sessionAs([ "citizen_verifier" ], "us9");
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ provider: null }));
    mocked(api.listEnabledProviders).mockResolvedValue([ { id: "pv1", name: "Laboratório Simulado" } ]);
    mocked(api.scheduleExam).mockResolvedValue(undefined);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Agendar" }));
    const form = await screen.findByRole("form", { name: "Agendar exame" });
    await within(form).findByRole("option", { name: "Laboratório Simulado" });
    expect(api.listEnabledProviders).toHaveBeenCalledWith("0202020380");
    fireEvent.change(within(form).getByLabelText("Prestador"), { target: { value: "pv1" } });
    fireEvent.change(within(form).getByLabelText("Data e hora"), { target: { value: "2026-10-20T08:30" } });
    fireEvent.change(within(form).getByLabelText("Local"), { target: { value: "Laboratório Simulado — Rua XV, 100" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar agendamento" }));
    await waitFor(() => expect(api.scheduleExam).toHaveBeenCalledWith("eo1", {
      provider_id: "pv1", scheduled_at: "2026-10-20T08:30:00-03:00", scheduled_place: "Laboratório Simulado — Rua XV, 100"
    }));
    expect((await screen.findByText("Exame agendado.")).getAttribute("role")).toBe("status");
    expect(screen.queryByRole("form", { name: "Agendar exame" })).toBeNull();
  });

  it("prestador já autorizado: agenda sem seletor, com o prestador do exame", async () => {
    sessionAs([ "citizen_verifier" ], "us9");
    mocked(api.getExamOrder).mockResolvedValue(examOrder());
    mocked(api.scheduleExam).mockResolvedValue(undefined);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Agendar" }));
    const form = await screen.findByRole("form", { name: "Agendar exame" });
    expect(within(form).queryByLabelText("Prestador")).toBeNull();
    fireEvent.change(within(form).getByLabelText("Data e hora"), { target: { value: "2026-10-20T08:30" } });
    fireEvent.change(within(form).getByLabelText("Local"), { target: { value: "Posto" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar agendamento" }));
    await waitFor(() => expect(mocked(api.scheduleExam).mock.calls[0][1]).toMatchObject({ provider_id: "pv1" }));
    expect(api.listEnabledProviders).not.toHaveBeenCalled();
  });

  it("409 invalid_transition: explica, fecha o formulário e relê", async () => {
    sessionAs([ "citizen_verifier" ], "us9");
    mocked(api.getExamOrder).mockResolvedValueOnce(examOrder({ status: "scheduled", scheduled_at: "2026-10-20T08:30:00-03:00", scheduled_place: "Posto" }))
      .mockResolvedValue(examOrder({ status: "collected" }));
    mocked(api.registerExamResult).mockRejectedValue(new ApiError(409, { error: "invalid_transition", status: "collected" }, "x"));
    mocked(api.uploadExamFile).mockResolvedValue("up1");
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Registrar resultado" }));
    const form = await screen.findByRole("form", { name: "Registrar resultado" });
    fireEvent.change(within(form).getByLabelText("Laudo (PDF, JPEG ou PNG, até 10 MB)"),
      { target: { files: [ new File([ "%PDF" ], "l.pdf", { type: "application/pdf" }) ] } });
    await within(form).findByText("anexado: l.pdf");
    fireEvent.change(within(form).getByLabelText("Laboratório"), { target: { value: "Lab" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado" }));
    expect((await screen.findByText("a situação do exame mudou — atualize para ver a atual")).getAttribute("role")).toBe("alert");
    expect(screen.queryByRole("form", { name: "Registrar resultado" })).toBeNull();
    await waitFor(() => expect(api.getExamOrder).toHaveBeenCalledTimes(2));
    expect(await screen.findByText("coletado")).not.toBeNull();
  });

  it("registrar coleta pede confirmação", async () => {
    sessionAs([ "citizen_verifier" ], "us9");
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "scheduled" }));
    mocked(api.collectExam).mockResolvedValue(undefined);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Registrar coleta" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.collectExam).toHaveBeenCalledWith("eo1"));
    expect(await screen.findByText("Coleta registrada.")).not.toBeNull();
  });

  it("quem pediu dá ciência sem motivo", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "resulted", result: examResult() }));
    mocked(api.acknowledgeExam).mockResolvedValue(undefined);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Dar ciência" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.acknowledgeExam).toHaveBeenCalledWith("eo1"));
    expect(await screen.findByText("Ciência registrada.")).not.toBeNull();
  });

  it("a equipe dá ciência do exame escalado só com motivo; exame alheio não escalado não tem botão", async () => {
    sessionAs([ "health_professional" ], "us2");
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "resulted", result: examResult(), escalated: true }));
    mocked(api.acknowledgeExam).mockResolvedValue(undefined);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Dar ciência pela equipe" }));
    const form = screen.getByRole("form", { name: "Motivo da ciência pela equipe" });
    fireEvent.change(within(form).getByLabelText("Motivo da ciência pela equipe"), { target: { value: "Paciente avisado por telefone" } });
    fireEvent.click(within(form).getByRole("button", { name: "Dar ciência" }));
    await waitFor(() => expect(api.acknowledgeExam).toHaveBeenCalledWith("eo1", "Paciente avisado por telefone"));

    cleanup();
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "resulted", result: examResult(), escalated: false }));
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    const region = await detail();
    expect(within(region).queryByRole("button", { name: /Dar ciência/ })).toBeNull();
  });

  it("cancelar pede o motivo", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder());
    mocked(api.cancelExam).mockResolvedValue(undefined);
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Cancelar exame" }));
    const form = screen.getByRole("form", { name: "Motivo do cancelamento" });
    fireEvent.change(within(form).getByLabelText("Motivo do cancelamento"), { target: { value: "pedido em duplicidade" } });
    fireEvent.click(within(form).getByRole("button", { name: "Cancelar exame" }));
    await waitFor(() => expect(api.cancelExam).toHaveBeenCalledWith("eo1", "pedido em duplicidade"));
  });

  it("abre o laudo numa janela nova por blob:", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "resulted", result: examResult() }));
    mocked(api.fetchExamPdf).mockResolvedValue({ blob: new Blob([ "%PDF" ], { type: "application/pdf" }), filename: null });
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Abrir laudo" }));
    await waitFor(() => expect(win.location.href).toBe("blob:laudo-1"));
    expect(api.fetchExamPdf).toHaveBeenCalledWith("/exams/results/er1/pdf?sig=abc");
  });

  it("laudo com link vencido: avisa e relê o exame", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "resulted", result: examResult() }));
    mocked(api.fetchExamPdf).mockRejectedValue(new ApiError(410, { error: "link_expired" }, "x"));
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    fireEvent.click(within(await detail()).getByRole("button", { name: "Abrir laudo" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o link do laudo expirou — abra de novo");
    expect(win.close).toHaveBeenCalled();
    await waitFor(() => expect(api.getExamOrder).toHaveBeenCalledTimes(2));
  });

  it("visão administrativa (unidade): resultado sem valores, sem crítico e sem laudo", async () => {
    sessionAs([ "citizen_verifier" ], "us9");
    mocked(api.getExamOrder).mockResolvedValue(examOrder({
      status: "resulted", result: { id: "er1", origin: "order", source: "fhir", released_on: "2026-10-12", laboratory_name: "Laboratório Simulado" }
    }));
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={vi.fn()} />);
    const result = within(await detail()).getByRole("region", { name: "Resultado" });
    expect(within(result).getByText("valores e laudo ficam com quem pediu e com a equipe")).not.toBeNull();
    expect(within(result).queryByRole("button", { name: "Abrir laudo" })).toBeNull();
    expect(within(result).queryByText("crítico")).toBeNull();
  });

  it("exame que o api não deixa ler: a frase da recusa e Fechar", async () => {
    mocked(api.getExamOrder).mockRejectedValue(new ApiError(403, { error: "out_of_context" }, "x"));
    const onClose = vi.fn();
    renderWithProviders(<ExamOrderDetail id="eo1" onClose={onClose} />);
    expect((await screen.findByRole("alert")).textContent).toBe("esta pessoa não está em atendimento com você");
    fireEvent.click(screen.getByRole("button", { name: "Fechar" }));
    expect(onClose).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/exams/ExamOrderDetail.test.tsx`
Expected: FAIL — `./ExamOrderDetail` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/modules/exams/openExamPdf.ts
// Laudo numa janela nova por um endereço `blob:` (nada do conteúdo passa pela
// URL da aplicação). A janela abre no clique: depois de um await o navegador
// trataria como pop-up. Link vencido (D11) devolve `expired` para quem chamou reler.
import { fetchExamPdf } from "../../lib/api";
import { PDF_LINK_EXPIRED, examError, pdfLinkExpired } from "../../lib/exams";

export const POPUP_BLOCKED = "o navegador bloqueou a janela nova — permita janelas deste site para ver o laudo";
const BLOB_TTL_MS = 60_000;

export type PdfOutcome = { ok: true } | { ok: false; expired: boolean; text: string };

export async function openExamPdf(pdfUrl: string): Promise<PdfOutcome> {
  const win = window.open("", "_blank");
  try {
    const file = await fetchExamPdf(pdfUrl);
    if (!win) return { ok: false, expired: false, text: POPUP_BLOCKED };
    const url = URL.createObjectURL(file.blob);
    win.location.href = url;
    setTimeout(() => URL.revokeObjectURL(url), BLOB_TTL_MS);
    return { ok: true };
  } catch (err) {
    win?.close();
    const expired = pdfLinkExpired(err);
    return { ok: false, expired, text: expired ? PDF_LINK_EXPIRED : examError(err) };
  }
}
```

```tsx
// src/modules/exams/ExamResultBlock.tsx
// O resultado (contrato §2 `<exam_result>`): data, laboratório, origem,
// "crítico", os valores e o laudo. Para profissional e unidade a marca
// "crítico" aparece (só o wpda nunca a mostra).
import { useState } from "react";
import type { ExamResult, ExamResultValue } from "../../lib/api";
import { SOURCE_LABEL, fmtDay } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { ActionButton } from "../../components/ActionButton";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { openExamPdf } from "./openExamPdf";
import { alert, box, muted, row } from "./styles";

export function ExamResultBlock({ result, onExpired }: { result: ExamResult; onExpired(): void }) {
  const [ error, setError ] = useState<string | null>(null);
  // Visão administrativa (unidade, regulador): sem valores, sem crítico, sem laudo.
  const values = result.values ?? [];
  const pdfUrl = result.pdf_url ?? null;

  async function open() {
    if (!pdfUrl) return;
    setError(null);
    const outcome = await openExamPdf(pdfUrl);
    if (outcome.ok) return;
    setError(outcome.text);
    if (outcome.expired) onExpired();
  }

  return (
    <section aria-label="Resultado" style={box}>
      <div style={row}>
        <strong>{`Resultado liberado em ${fmtDay(result.released_on)}`}</strong>
        {result.critical === true && <Tag tone="down">crítico</Tag>}
        {result.origin === "external" && <Tag>externo</Tag>}
      </div>
      <p style={muted}>{`${result.laboratory_name} · ${SOURCE_LABEL[result.source] ?? result.source}`}</p>
      {result.values === undefined && <p style={muted}>valores e laudo ficam com quem pediu e com a equipe</p>}
      {values.length > 0 && (
        <DataTable<ExamResultValue>
          cols={[
            { label: "Campo", w: "1.4fr", card: "title", render: (v) => v.label },
            { label: "Valor", w: "1fr", render: (v) => `${v.value}${v.unit ? ` ${v.unit}` : ""}` },
            { label: "LOINC", w: "0.8fr", render: (v) => <span className="mono">{v.loinc ?? "—"}</span> }
          ]}
          rows={values}
          rowKey={(v, i) => `${v.key ?? v.loinc ?? "valor"}:${i}`}
        />
      )}
      {result.values !== undefined && values.length === 0 && <p style={muted}>sem valores digitados — veja o laudo</p>}
      {result.acknowledged_at && <p style={muted}>{`ciência em ${fmtDateTime(result.acknowledged_at)}`}</p>}
      {pdfUrl && <div><ActionButton variant="secondary" onClick={open}>Abrir laudo</ActionButton></div>}
      {error && <p role="alert" style={alert}>{error}</p>}
    </section>
  );
}
```

```tsx
// src/modules/exams/ExamOrderDetail.tsx
// O exame (contrato §2 `<exam_order>` e §3): situação, andamento, resultado e
// as ações que o papel da sessão tem na situação de agora. Ler grava a trilha
// no api (regras do 19a); nada fica no cache (gcTime: 0). Toda ação relê as
// listas; 409 invalid_transition (alguém agiu antes) fecha o formulário,
// explica e relê o exame.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  acknowledgeExam, cancelExam, collectExam, errorCode, getExamOrder, listEnabledProviders, registerExamResult, scheduleExam
} from "../../lib/api";
import { useAuth } from "../../lib/auth";
import {
  EXAM_ORDER_KEY, EXAM_PROVIDER_REFS_KEY, ORIGIN_LABEL, eventLabel, examActions, examError, examRoles, examStatusLabel,
  examStatusTone, overdueLabel, refreshExams, scheduleBody
} from "../../lib/exams";
import { cityIsoDate, fmtDateTime } from "../../lib/format";
import { useActionMessage } from "../../lib/useActionMessage";
import { ActionMessage } from "../../components/ActionMessage";
import { ConfirmButton } from "../../components/ConfirmButton";
import { KeyValue } from "../../components/KeyValue";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { ExamResultBlock } from "./ExamResultBlock";
import { ReasonForm } from "./ReasonForm";
import { ResultForm } from "./ResultForm";
import { ScheduleForm } from "./ScheduleForm";
import { alert, grid, list, muted, panel, row, title } from "./styles";

type Mode = null | "schedule" | "result" | "cancel" | "acknowledge";

export function ExamOrderDetail({ id, onClose, onChanged }: { id: string; onClose(): void; onChanged?(): void }) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ EXAM_ORDER_KEY, id ], queryFn: () => getExamOrder(id), gcTime: 0, staleTime: 0, retry: false });
  const [ mode, setMode ] = useState<Mode>(null);
  const message = useActionMessage();
  const order = query.data;
  const needsProvider = mode === "schedule" && !!order && !order.provider;
  const providers = useQuery({
    queryKey: [ EXAM_PROVIDER_REFS_KEY, order?.sigtap_code ?? "" ],
    queryFn: () => listEnabledProviders(order?.sigtap_code),
    enabled: needsProvider, retry: false
  });

  // Formulários: a recusa comum volta para o formulário (ele mostra); a de
  // situação velha fecha o formulário e relê.
  async function act(run: () => Promise<void>, okText: string): Promise<void> {
    message.start();
    try {
      await run();
    } catch (err) {
      if (errorCode(err) !== "invalid_transition") throw err;
      setMode(null);
      message.fail(examError(err));
      await refreshExams(queryClient);
      return;
    }
    setMode(null);
    message.ok(okText);
    await refreshExams(queryClient);
    onChanged?.();
  }
  // Confirmação de um clique: a recusa aparece na mensagem do exame.
  const confirm = (run: () => Promise<void>, okText: string) => () =>
    act(run, okText).catch((err) => message.fail(examError(err)));

  if (query.isPending) {
    return <section aria-label="Exame" style={panel}><p className="mono" style={muted}>carregando o exame…</p></section>;
  }
  if (query.isError || !order) {
    return (
      <section aria-label="Exame" style={panel}>
        <p role="alert" style={alert}>{examError(query.error)}</p>
        <div><button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button></div>
      </section>
    );
  }

  const actions = examActions(order, examRoles(user), user?.id ?? null);
  const requester = order.requester.user_id ? order.requester.name : `${order.requester.name} (digitado)`;
  const scheduled = order.scheduled_at ? `${fmtDateTime(order.scheduled_at)} · ${order.scheduled_place ?? ""}` : "—";

  return (
    <section aria-label={`Exame ${order.label}`} style={panel}>
      <div style={row}>
        <h3 style={title}>{order.label}</h3>
        <Tag tone={examStatusTone(order.status)}>{examStatusLabel(order.status)}</Tag>
        {order.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}
        {order.escalated && <Tag tone="warn">na fila da equipe</Tag>}
      </div>
      <div style={grid}>
        <KeyValue k="Código" v={order.short_code} />
        <KeyValue k="Paciente" v={order.patient.display_name} mono={false} />
        <KeyValue k="SIGTAP" v={order.sigtap_code} />
        <KeyValue k="Origem" v={ORIGIN_LABEL[order.origin] ?? order.origin} mono={false} />
        <KeyValue k="Solicitante" v={requester} mono={false} />
        <KeyValue k="Unidade" v={order.unit.name} mono={false} />
        <KeyValue k="Prestador" v={order.provider?.name ?? "—"} mono={false} />
        <KeyValue k="Agendado" v={scheduled} mono={false} />
        <KeyValue k="Prazo" v={overdueLabel(order.overdue)} mono={false} />
      </div>
      {order.events && order.events.length > 0 && (
        <ol aria-label="andamento" style={list}>
          {order.events.map((e, i) => (
            <li key={`${e.kind}:${e.occurred_at}:${i}`}>
              {`${fmtDateTime(e.occurred_at)} · ${eventLabel(e.kind)} · ${e.actor.name}${e.reason ? ` — ${e.reason}` : ""}`}
            </li>
          ))}
        </ol>
      )}
      {order.result && <ExamResultBlock result={order.result} onExpired={() => void query.refetch()} />}
      <ActionMessage message={message.message} />

      {mode === null && (
        <div style={row}>
          {actions.schedule && <button type="button" style={secondaryButtonStyle} onClick={() => setMode("schedule")}>Agendar</button>}
          {actions.collect && (
            <ConfirmButton label="Registrar coleta" question="Confirmar que o material foi coletado?"
              onConfirm={confirm(() => collectExam(order.id), "Coleta registrada.")} />
          )}
          {actions.result && <button type="button" style={secondaryButtonStyle} onClick={() => setMode("result")}>Registrar resultado</button>}
          {actions.acknowledge && !actions.acknowledgeNeedsReason && (
            <ConfirmButton label="Dar ciência" variant="primary" question="Confirmar que você viu este resultado?"
              onConfirm={confirm(() => acknowledgeExam(order.id), "Ciência registrada.")} />
          )}
          {actions.acknowledgeNeedsReason && (
            <button type="button" style={secondaryButtonStyle} onClick={() => setMode("acknowledge")}>Dar ciência pela equipe</button>
          )}
          {actions.cancel && <button type="button" style={secondaryButtonStyle} onClick={() => setMode("cancel")}>Cancelar exame</button>}
          <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>
        </div>
      )}

      {mode === "schedule" && needsProvider && providers.isPending && <p className="mono" style={muted}>carregando prestadores…</p>}
      {mode === "schedule" && needsProvider && providers.isError && <p role="alert" style={alert}>{examError(providers.error)}</p>}
      {mode === "schedule" && (!needsProvider || providers.isSuccess) && (
        <ScheduleForm providers={needsProvider ? providers.data ?? [] : null} onCancel={() => setMode(null)}
          onSubmit={(d) => act(() => scheduleExam(order.id, scheduleBody(d, order.provider?.id)), "Exame agendado.")} />
      )}
      {mode === "result" && (
        <ResultForm fields={order.result_fields ?? []} today={cityIsoDate()} submitLabel="Registrar resultado" onCancel={() => setMode(null)}
          onSubmit={(input) => act(() => registerExamResult(order.id, input), "Resultado registrado.")} />
      )}
      {mode === "cancel" && (
        <ReasonForm label="Motivo do cancelamento" submitLabel="Cancelar exame" onCancel={() => setMode(null)}
          onSubmit={(reason) => act(() => cancelExam(order.id, reason), "Exame cancelado.")} />
      )}
      {mode === "acknowledge" && (
        <ReasonForm label="Motivo da ciência pela equipe" submitLabel="Dar ciência" onCancel={() => setMode(null)}
          onSubmit={(reason) => act(() => acknowledgeExam(order.id, reason), "Ciência registrada.")} />
      )}
    </section>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/exams && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/exams/ExamOrderDetail.tsx src/modules/exams/ExamResultBlock.tsx src/modules/exams/openExamPdf.ts src/modules/exams/ExamOrderDetail.test.tsx
$G commit -m "feat: show an exam order with its trail, result, report and actions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 6: Prioridade no pedido e situação dos exames na consulta

**Files:**
- Modify: `src/lib/consultation.ts` (`examPayload`, `setPriority` novo), `src/modules/consultation/ExamRequestsField.tsx`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationView.tsx`
- Create: `src/modules/exams/ConsultationExamStatus.tsx`
- Test: `src/lib/consultation.priority.test.ts`, `src/modules/consultation/ExamRequestsField.priority.test.tsx`, `src/modules/exams/ConsultationExamStatus.test.tsx`

**Interfaces:**
- Consumes: `ExamRequest.priority`, `ExamPriority`, `listExamOrders` (Task 1); `EXAM_ORDERS_KEY`, `examError`, `examStatusLabel`, `examStatusTone`, `hasFeature` (Tasks 1–2); `ExamOrderDetail` (Task 5).
- Produces: `setPriority(exams: ExamRequest[], code: string, priority: ExamPriority): ExamRequest[]` em `lib/consultation.ts`; `checkDraft` passa a manter `priority` quando presente (Divergência D10); `ExamRequestsField` aceita `priorityOn?: boolean`; `ConsultationEditorProps` ganha `examPriority?: boolean`; `ConsultationExamStatus(props: { consultationId: string; requests: ExamRequest[] })` (`<ul aria-label="exames solicitados">`).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/consultation.priority.test.ts
import { describe, expect, it } from "vitest";
import { checkDraft, draftFrom, setPriority } from "./consultation";
import { consultation } from "../test/consultationFixtures";

describe("prioridade do exame no rascunho (módulo 21a, D10)", () => {
  const exam = { sigtap_code: "0202020380", label: "Hemograma completo" };

  it("setPriority troca só o exame pedido", () => {
    const next = setPriority([ exam, { sigtap_code: "0202010503", label: "HbA1c" } ], "0202020380", "urgent");
    expect(next.map((e) => e.priority)).toEqual([ "urgent", undefined ]);
  });

  it("o corpo do autosave leva a prioridade quando há, e nada quando não há", () => {
    const d = draftFrom(consultation({ exam_requests: [ { ...exam, priority: "urgent" }, { sigtap_code: "0202010503", label: "HbA1c" } ] }));
    expect(checkDraft(d).input.exam_requests).toEqual([
      { sigtap_code: "0202020380", label: "Hemograma completo", priority: "urgent" },
      { sigtap_code: "0202010503", label: "HbA1c" }
    ]);
  });
});
```

```tsx
// src/modules/consultation/ExamRequestsField.priority.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactElement } from "react";
import { ExamRequestsField } from "./ExamRequestsField";

afterEach(cleanup);
// O campo não lê a sessão: basta o QueryClient da busca de SIGTAP.
const renderField = (ui: ReactElement) => render(<QueryClientProvider client={new QueryClient()}>{ui}</QueryClientProvider>);
const VALUE = [ { sigtap_code: "0202020380", label: "Hemograma completo" } ];

describe("ExamRequestsField — prioridade (módulo 21a)", () => {
  it("com o rastreamento: escolhe urgente por exame; padrão rotina", () => {
    const onChange = vi.fn();
    renderField(<ExamRequestsField value={VALUE} onChange={onChange} cid10Allowed={false} priorityOn searchDelayMs={0} />);
    const select = screen.getByLabelText("Prioridade (0202020380)") as HTMLSelectElement;
    expect(select.value).toBe("routine");
    fireEvent.change(select, { target: { value: "urgent" } });
    expect(onChange).toHaveBeenCalledWith([ { sigtap_code: "0202020380", label: "Hemograma completo", priority: "urgent" } ]);
  });

  it("sem o rastreamento: nada muda", () => {
    renderField(<ExamRequestsField value={VALUE} onChange={vi.fn()} cid10Allowed={false} searchDelayMs={0} />);
    expect(screen.queryByLabelText("Prioridade (0202020380)")).toBeNull();
  });
});
```

```tsx
// src/modules/exams/ConsultationExamStatus.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listExamOrders: vi.fn(), getExamOrder: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ConsultationExamStatus } from "./ConsultationExamStatus";
import { ConsultationView } from "../consultation/ConsultationView";
import { renderWithProviders } from "../../test/campaignFixtures";
import { finalized, options } from "../../test/consultationFixtures";
import { examOrder, examUser } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const REQUESTS = [
  { sigtap_code: "0202020380", label: "Hemograma completo", priority: "urgent" as const },
  { sigtap_code: "0202010503", label: "Dosagem de hemoglobina glicosilada" }
];

afterEach(cleanup);

describe("ConsultationExamStatus", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listExamOrders, api.getExamOrder ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser());
    mocked(api.listExamOrders).mockResolvedValue({ orders: [ examOrder({ status: "scheduled" }) ], next_before: null });
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "scheduled" }));
  });

  it("cada exame pedido com a situação; sem exame acompanhado diz isso; Acompanhar abre o exame", async () => {
    renderWithProviders(<ConsultationExamStatus consultationId="c1" requests={REQUESTS} />);
    const items = (await screen.findAllByRole("listitem"));
    await within(items[0]).findByText("agendado");
    expect(within(items[0]).getByText("urgente")).not.toBeNull();
    expect(within(items[1]).getByText("sem acompanhamento")).not.toBeNull();
    expect(api.listExamOrders).toHaveBeenCalledWith({ consultationId: "c1" });
    fireEvent.click(within(items[0]).getByRole("button", { name: "Acompanhar Hemograma completo" }));
    expect(await screen.findByRole("region", { name: "Exame Hemograma completo" })).not.toBeNull();
  });

  it("a leitura falha: a lista continua e a frase aparece", async () => {
    mocked(api.listExamOrders).mockRejectedValue(new ApiError(403, { error: "feature_disabled", feature: "exam_tracking" }, "x"));
    renderWithProviders(<ConsultationExamStatus consultationId="c1" requests={REQUESTS} />);
    expect((await screen.findByRole("alert")).textContent).toBe("o rastreamento de exames está desligado nesta cidade");
    expect(screen.getAllByRole("listitem")).toHaveLength(2);
  });

  it("ConsultationView usa o acompanhamento só com o rastreamento ligado e fora da leitura administrativa", async () => {
    const c = finalized({ exam_requests: REQUESTS });
    renderWithProviders(<ConsultationView consultation={c} options={options()} patientProblems={[]} onAddendumAdded={vi.fn()} onClose={vi.fn()} />);
    await screen.findByText("agendado");
    cleanup();
    mocked(api.listExamOrders).mockClear();
    renderWithProviders(<ConsultationView consultation={c} options={options()} patientProblems={[]} readOnly onAddendumAdded={vi.fn()} onClose={vi.fn()} />);
    await screen.findByRole("list", { name: "exames solicitados" });
    expect(api.listExamOrders).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/consultation.priority.test.ts src/modules/consultation/ExamRequestsField.priority.test.tsx src/modules/exams/ConsultationExamStatus.test.tsx`
Expected: FAIL — `setPriority` não existe, `priorityOn` não muda nada, `checkDraft` descarta a prioridade e `ConsultationExamStatus` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/consultation.ts`, troque `examPayload` e acrescente `setPriority` logo depois de `setJustification` (importe `type ExamPriority` de `./api` junto dos outros tipos):

```ts
function examPayload(e: ExamRequest): ExamRequest {
  const j = normalizeCid10(e.cid10_justification ?? "");
  const out: ExamRequest = { sigtap_code: e.sigtap_code, label: e.label };
  if (j !== "") out.cid10_justification = j;
  // Módulo 21a (D10): a prioridade só vai quando foi escolhida (ausente = rotina no api).
  if (e.priority) out.priority = e.priority;
  return out;
}
```

```ts
export function setPriority(exams: ExamRequest[], code: string, priority: ExamPriority): ExamRequest[] {
  return exams.map((e) => (e.sigtap_code === code ? { ...e, priority } : e));
}
```

Em `src/modules/consultation/ExamRequestsField.tsx`: importe `type ExamPriority` de `../../lib/api` e `setPriority` de `../../lib/consultation`; a interface vira

```ts
interface Props { value: ExamRequest[]; onChange(next: ExamRequest[]): void; cid10Allowed: boolean; searchDelayMs?: number; priorityOn?: boolean }
```

a assinatura `export function ExamRequestsField({ value, onChange, cid10Allowed, searchDelayMs, priorityOn = false }: Props)`, e dentro do `<li>`, logo antes do botão "Remover":

```tsx
                {priorityOn && (
                  <label style={labelStyle}>
                    {`Prioridade (${e.sigtap_code})`}
                    <select value={e.priority ?? "routine"} style={inputStyle}
                      onChange={(ev) => onChange(setPriority(value, e.sigtap_code, ev.target.value as ExamPriority))}>
                      <option value="routine">Rotina</option>
                      <option value="urgent">Urgente</option>
                    </select>
                  </label>
                )}
```

Em `src/modules/consultation/ConsultationEditor.tsx`: em `ConsultationEditorProps` acrescente `// Módulo 21a: prioridade de cada exame (com exam_tracking ligado).\n  examPriority?: boolean;` e no `<ExamRequestsField …>` acrescente `priorityOn={props.examPriority}`.

Em `src/modules/consultation/ConsultationWorkspace.tsx`: logo depois de `const timelineOn = …;` acrescente `const examsOn = hasFeature(user, "exam_tracking");` e no `<ConsultationEditor key={consultation.id} …>` acrescente `examPriority={examsOn}`.

Em `src/modules/consultation/ConsultationView.tsx`: importe `hasFeature` de `../../lib/features` e `ConsultationExamStatus` de `../exams/ConsultationExamStatus`; depois de `const isAuthor = …;` acrescente

```ts
  // Módulo 21a: na consulta finalizada, cada exame com a situação de agora
  // (não na leitura administrativa, que é só do registro).
  const examTracking = hasFeature(user, "exam_tracking") && !props.readOnly && c.status === "finalized";
```

e troque o bloco `{c.exam_requests.length > 0 && ( <ul aria-label="exames solicitados" …> … </ul> )}` por

```tsx
      {c.exam_requests.length > 0 && (examTracking
        ? <ConsultationExamStatus consultationId={c.id} requests={c.exam_requests} />
        : (
          <ul aria-label="exames solicitados" style={list}>
            {c.exam_requests.map((e) => (
              <li key={e.sigtap_code}>
                {`${e.sigtap_code} — ${e.label}${e.cid10_justification ? ` · CID-10 ${e.cid10_justification}` : ""}`}
              </li>
            ))}
          </ul>
        ))}
```

```tsx
// src/modules/exams/ConsultationExamStatus.tsx
// Os exames pedidos na consulta com a situação de cada um (módulo 21a; spec
// §9 "status dos exames na consulta"; Divergência D6: filtro consultation_id).
// O pedido sem exame acompanhado (rastreamento ligado depois, ou exame
// cancelado e trocado) aparece "sem acompanhamento".
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { listExamOrders, type ExamRequest } from "../../lib/api";
import { EXAM_ORDERS_KEY, examError, examStatusLabel, examStatusTone } from "../../lib/exams";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { ExamOrderDetail } from "./ExamOrderDetail";
import { alert, list, row } from "./styles";

export function ConsultationExamStatus({ consultationId, requests }: { consultationId: string; requests: ExamRequest[] }) {
  const query = useQuery({
    queryKey: [ EXAM_ORDERS_KEY, "consultation", consultationId ],
    queryFn: () => listExamOrders({ consultationId }),
    gcTime: 0, staleTime: 0, retry: false
  });
  const [ open, setOpen ] = useState<string | null>(null);
  const byCode = new Map((query.data?.orders ?? []).map((o) => [ o.sigtap_code, o ]));

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
      <ul aria-label="exames solicitados" style={list}>
        {requests.map((e) => {
          const order = byCode.get(e.sigtap_code);
          return (
            <li key={e.sigtap_code} style={row}>
              <span>{`${e.sigtap_code} — ${e.label}${e.cid10_justification ? ` · CID-10 ${e.cid10_justification}` : ""}`}</span>
              {e.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}
              {order && <Tag tone={examStatusTone(order.status)}>{examStatusLabel(order.status)}</Tag>}
              {!order && query.isSuccess && <Tag>sem acompanhamento</Tag>}
              {order && (
                <button type="button" aria-label={`Acompanhar ${e.label}`} style={secondaryButtonStyle} onClick={() => setOpen(order.id)}>
                  Acompanhar
                </button>
              )}
            </li>
          );
        })}
      </ul>
      {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
      {open && <ExamOrderDetail key={open} id={open} onClose={() => setOpen(null)} />}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/consultation src/modules/consultation src/modules/exams && npx tsc --noEmit`
Expected: PASS — inclusive os testes antigos do editor e da consulta (sem `exam_tracking` na sessão deles, nada muda; o corpo do autosave sem prioridade é o mesmo de antes).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/consultation.ts src/lib/consultation.priority.test.ts src/modules/consultation/ExamRequestsField.tsx \
  src/modules/consultation/ExamRequestsField.priority.test.tsx src/modules/consultation/ConsultationEditor.tsx \
  src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationView.tsx \
  src/modules/exams/ConsultationExamStatus.tsx src/modules/exams/ConsultationExamStatus.test.tsx
$G commit -m "feat: set exam priority in the consultation and show each exam status

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Aba "Exames" do paciente e resultado externo

**Files:**
- Modify: `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Create: `src/modules/exams/PatientExams.tsx`, `src/modules/exams/ExternalResultForm.tsx`
- Test: `src/modules/exams/PatientExams.test.tsx`, `src/modules/consultation/PatientPanel.exams.test.tsx`

**Interfaces:**
- Consumes: `listExamOrders`, `listExternalResults`, `listExamCatalog`, `recordExternalResult`, `ExamOrder`, `ExternalExamResult` (Task 1); `EXAM_ORDERS_KEY`, `EXTERNAL_RESULTS_KEY`, `EXAM_CATALOG_KEY`, `examError`, `examStatusLabel`, `examStatusTone`, `fmtDay`, `overdueLabel`, `refreshExams` (Task 2); `useExamOrderPages` (Task 2); `ExamOrderDetail`, `openExamPdf` (Task 5); `ResultForm` (Task 4).
- Produces: `PatientExams(props: PatientExamsProps)` com `export interface PatientExamsProps { patientId: string; consultationId?: string | null }` (`<section aria-label="Exames do paciente">`); `ExternalResultForm(props: { patientId: string; consultationId?: string | null; onDone(): void; onCancel(): void })` (`<section aria-label="Resultado externo">`); `PatientPanel` aceita `exams?: PatientExamsProps` (aba "Exames").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/exams/PatientExams.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listExamOrders: vi.fn(), listExternalResults: vi.fn(), getExamOrder: vi.fn(),
    listExamCatalog: vi.fn(), recordExternalResult: vi.fn(), uploadExamFile: vi.fn(), fetchExamPdf: vi.fn() };
});

import * as api from "../../lib/api";
import { PatientExams } from "./PatientExams";
import { renderWithProviders } from "../../test/campaignFixtures";
import { HBA1C_ENTRY, HEMOGRAM_ENTRY, NOW21, TODAY21, examOrder, examResult, examUser, pdfFile } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listExamOrders, api.listExternalResults, api.getExamOrder, api.listExamCatalog,
  api.recordExternalResult, api.uploadExamFile, api.fetchExamPdf ];

afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("PatientExams", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW21));
    for (const fn of ALL) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser());
    mocked(api.listExamOrders).mockResolvedValue({
      orders: [ examOrder({ status: "collected", priority: "urgent", overdue: { stage: "result", due_on: "2026-10-11", days_late: 2 } }) ],
      next_before: "cur1"
    });
    mocked(api.listExternalResults).mockResolvedValue([ { ...examResult({ id: "ex1", origin: "external", laboratory_name: "Lab Particular" }),
      sigtap_code: "0202010503", label: "Dosagem de hemoglobina glicosilada" } ]);
    mocked(api.listExamCatalog).mockResolvedValue([ HEMOGRAM_ENTRY, HBA1C_ENTRY ]);
    mocked(api.uploadExamFile).mockResolvedValue("up9");
    mocked(api.recordExternalResult).mockResolvedValue(examResult({ id: "ex2", origin: "external" }));
  });

  it("lista os exames do paciente com situação, prioridade e atraso; Carregar mais usa o cursor", async () => {
    renderWithProviders(<PatientExams patientId="pa1" />);
    const section = await screen.findByRole("region", { name: "Exames do paciente" });
    await within(section).findByText("coletado");
    expect(within(section).getByText("urgente")).not.toBeNull();
    expect(within(section).getByText("resultado · 2 dias de atraso")).not.toBeNull();
    expect(api.listExamOrders).toHaveBeenCalledWith({ patientId: "pa1", before: null });
    mocked(api.listExamOrders).mockResolvedValueOnce({ orders: [ examOrder({ id: "eo2", label: "Urina tipo I" }) ], next_before: null });
    fireEvent.click(within(section).getByRole("button", { name: "Carregar mais" }));
    await within(section).findByText("Urina tipo I");
    expect(api.listExamOrders).toHaveBeenLastCalledWith({ patientId: "pa1", before: "cur1" });
    expect(within(section).queryByRole("button", { name: "Carregar mais" })).toBeNull();
  });

  it("resultados externos aparecem com laboratório e data", async () => {
    renderWithProviders(<PatientExams patientId="pa1" />);
    const ext = await screen.findByRole("region", { name: "Resultados externos" });
    await within(ext).findByText("Lab Particular");
    expect(within(ext).getByText("12/10/2026")).not.toBeNull();
  });

  it("registra resultado externo com a consulta em curso", async () => {
    renderWithProviders(<PatientExams patientId="pa1" consultationId="c9" />);
    fireEvent.click(await screen.findByRole("button", { name: "Registrar resultado externo" }));
    const box = screen.getByRole("region", { name: "Resultado externo" });
    fireEvent.change(await within(box).findByLabelText("Exame"), { target: { value: "0202010503" } });
    const form = within(box).getByRole("form", { name: "Registrar resultado externo" });
    fireEvent.change(within(form).getByLabelText("Laudo (PDF, JPEG ou PNG, até 10 MB)"), { target: { files: [ pdfFile() ] } });
    await within(form).findByText("anexado: laudo.pdf");
    fireEvent.change(within(form).getByLabelText("Laboratório"), { target: { value: "Lab Particular" } });
    fireEvent.change(within(form).getByLabelText("HbA1c (%)"), { target: { value: "7,8" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado externo" }));
    await waitFor(() => expect(api.recordExternalResult).toHaveBeenCalledWith({
      patient_id: "pa1", sigtap_code: "0202010503", released_on: TODAY21, laboratory_name: "Lab Particular", upload_id: "up9",
      values: [ { key: "hba1c", value: "7,8" } ], consultation_id: "c9"
    }));
    expect((await screen.findByText("Resultado externo registrado.")).getAttribute("role")).toBe("status");
    expect(screen.queryByRole("region", { name: "Resultado externo" })).toBeNull();
  });

  it("Abrir mostra o exame", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder());
    renderWithProviders(<PatientExams patientId="pa1" />);
    fireEvent.click(await screen.findByRole("button", { name: "Abrir Hemograma completo" }));
    expect(await screen.findByRole("region", { name: "Exame Hemograma completo" })).not.toBeNull();
  });
});
```

```tsx
// src/modules/consultation/PatientPanel.exams.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listExamOrders: vi.fn(), listExternalResults: vi.fn() };
});

import * as api from "../../lib/api";
import { PatientPanel } from "./PatientPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { record } from "../../test/consultationFixtures";
import { examUser } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("PatientPanel — aba Exames (módulo 21a)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listExamOrders, api.listExternalResults ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser());
    mocked(api.listExamOrders).mockResolvedValue({ orders: [], next_before: null });
    mocked(api.listExternalResults).mockResolvedValue([]);
  });

  it("com exams: abas Resumo e Exames; a lista só lê ao abrir a aba", async () => {
    renderWithProviders(<PatientPanel record={record()} onOpenConsultation={vi.fn()} exams={{ patientId: "pa1" }} />);
    expect(screen.getAllByRole("tab").map((t) => t.textContent)).toEqual([ "Resumo", "Exames" ]);
    expect(api.listExamOrders).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("tab", { name: "Exames" }));
    expect(await screen.findByText("nenhum exame pedido")).not.toBeNull();
    expect(api.listExamOrders).toHaveBeenCalledWith({ patientId: "pa1", before: null });
    fireEvent.click(screen.getByRole("tab", { name: "Resumo" }));
    expect(screen.getByText("Problemas ativos")).not.toBeNull();
  });

  it("sem exams nem timeline: sem abas (como antes)", () => {
    renderWithProviders(<PatientPanel record={record()} onOpenConsultation={vi.fn()} />);
    expect(screen.queryAllByRole("tab")).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/exams/PatientExams.test.tsx src/modules/consultation/PatientPanel.exams.test.tsx`
Expected: FAIL — `PatientExams` não existe; `PatientPanel` ignora `exams`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/exams/ExternalResultForm.tsx
// Resultado externo (spec §6; contrato §3): exame feito fora de um pedido do
// Rota Saúde. O exame vem do catálogo da cidade (D3), os campos são os dele,
// e a ciência fica em nome de quem registra (o api grava). Sem "crítico".
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listExamCatalog, recordExternalResult } from "../../lib/api";
import { EXAM_CATALOG_KEY, examError, refreshExams } from "../../lib/exams";
import { cityIsoDate } from "../../lib/format";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ResultForm } from "./ResultForm";
import { alert, box, fieldLabel, muted } from "./styles";

export function ExternalResultForm({ patientId, consultationId = null, onDone, onCancel }: {
  patientId: string; consultationId?: string | null; onDone(): void; onCancel(): void;
}) {
  const queryClient = useQueryClient();
  const catalog = useQuery({ queryKey: [ EXAM_CATALOG_KEY ], queryFn: listExamCatalog, retry: false });
  const [ code, setCode ] = useState("");
  const entry = catalog.data?.find((e) => e.sigtap_code === code) ?? null;

  return (
    <section aria-label="Resultado externo" style={box}>
      <p style={muted}>Exame feito fora de um pedido do Rota Saúde. A ciência fica registrada em seu nome.</p>
      {catalog.isPending && <p className="mono" style={muted}>carregando o catálogo…</p>}
      {catalog.isError && <p role="alert" style={alert}>{examError(catalog.error)}</p>}
      {catalog.isSuccess && (
        <label style={fieldLabel}>
          Exame
          <select value={code} style={inputStyle} onChange={(e) => setCode(e.target.value)}>
            <option value="">escolha…</option>
            {catalog.data.map((e) => <option key={e.sigtap_code} value={e.sigtap_code}>{`${e.label} (${e.sigtap_code})`}</option>)}
          </select>
        </label>
      )}
      {entry ? (
        <ResultForm key={entry.sigtap_code} fields={entry.result_fields} today={cityIsoDate()} allowCritical={false}
          submitLabel="Registrar resultado externo" onCancel={onCancel}
          onSubmit={async (input) => {
            await recordExternalResult({
              patient_id: patientId, sigtap_code: entry.sigtap_code, released_on: input.released_on,
              laboratory_name: input.laboratory_name, upload_id: input.upload_id, values: input.values,
              ...(consultationId ? { consultation_id: consultationId } : {})
            });
            await refreshExams(queryClient);
            onDone();
          }} />
      ) : (
        <div><button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button></div>
      )}
    </section>
  );
}
```

```tsx
// src/modules/exams/PatientExams.tsx
// Aba "Exames" do paciente (spec §9): os exames dele com a situação, os
// resultados externos e o registro de um novo resultado externo. Quem lê e
// a trilha são do api (regras do 19a); nada fica no cache.
import { useState, type ReactNode } from "react";
import { useQuery } from "@tanstack/react-query";
import { listExamOrders, listExternalResults, type ExamOrder, type ExternalExamResult } from "../../lib/api";
import {
  EXAM_ORDERS_KEY, EXTERNAL_RESULTS_KEY, examError, examStatusLabel, examStatusTone, fmtDay, overdueLabel
} from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { useExamOrderPages } from "../../hooks/useExamOrderPages";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ExamOrderDetail } from "./ExamOrderDetail";
import { ExternalResultForm } from "./ExternalResultForm";
import { openExamPdf } from "./openExamPdf";
import { alert, muted, okText, row } from "./styles";

export interface PatientExamsProps { patientId: string; consultationId?: string | null }

export function PatientExams({ patientId, consultationId = null }: PatientExamsProps) {
  const { query, orders } = useExamOrderPages([ EXAM_ORDERS_KEY, "patient", patientId ],
    (before) => listExamOrders({ patientId, before }));
  const external = useQuery({
    queryKey: [ EXTERNAL_RESULTS_KEY, patientId ], queryFn: () => listExternalResults(patientId), gcTime: 0, staleTime: 0, retry: false
  });
  const [ open, setOpen ] = useState<string | null>(null);
  const [ adding, setAdding ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ pdfError, setPdfError ] = useState<string | null>(null);

  async function openPdf(r: ExternalExamResult) {
    if (!r.pdf_url) return;
    setPdfError(null);
    const outcome = await openExamPdf(r.pdf_url);
    if (outcome.ok) return;
    setPdfError(outcome.text);
    if (outcome.expired) void external.refetch();
  }

  return (
    <section aria-label="Exames do paciente" style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      {notice && <p role="status" style={okText}>{notice}</p>}
      {query.isPending && <p className="mono" style={muted}>carregando os exames…</p>}
      {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
      {query.isSuccess && (
        <DataTable<ExamOrder>
          cols={[
            { label: "Exame", w: "1.8fr", card: "title", render: (o) => <Labelled o={o} /> },
            { label: "Situação", w: "1fr", render: (o) => <Tag tone={examStatusTone(o.status)}>{examStatusLabel(o.status)}</Tag> },
            { label: "Desde", w: "1fr", render: (o) => fmtDateTime(o.status_changed_at) },
            { label: "Prazo", w: "1.2fr", render: (o) => overdueLabel(o.overdue) },
            { label: "", w: "auto", align: "right", card: "actions", render: (o) => (
              <button type="button" aria-label={`Abrir ${o.label}`} style={secondaryButtonStyle} onClick={() => setOpen(o.id)}>Abrir</button>
            ) }
          ]}
          rows={orders} rowKey={(o) => o.id} empty="nenhum exame pedido"
        />
      )}
      {query.hasNextPage && (
        <div><button type="button" style={secondaryButtonStyle} onClick={() => void query.fetchNextPage()}>Carregar mais</button></div>
      )}
      {open && <ExamOrderDetail key={open} id={open} onClose={() => setOpen(null)} />}

      <section aria-label="Resultados externos" style={{ display: "flex", flexDirection: "column", gap: 8 }}>
        <strong style={{ fontSize: 12.5 }}>Resultados externos</strong>
        {external.isError && <p role="alert" style={alert}>{examError(external.error)}</p>}
        {external.isSuccess && (
          <DataTable<ExternalExamResult>
            cols={[
              { label: "Exame", w: "1.8fr", card: "title", render: (r) => r.label },
              { label: "Laboratório", w: "1.2fr", render: (r) => r.laboratory_name },
              { label: "Liberado em", w: "1fr", render: (r) => fmtDay(r.released_on) },
              { label: "", w: "auto", align: "right", card: "actions", render: (r) => (r.pdf_url ? (
                <button type="button" aria-label={`Abrir laudo de ${r.label}`} style={secondaryButtonStyle} onClick={() => void openPdf(r)}>Abrir laudo</button>
              ) : null) }
            ]}
            rows={external.data} rowKey={(r) => r.id} empty="nenhum resultado externo"
          />
        )}
        {pdfError && <p role="alert" style={alert}>{pdfError}</p>}
      </section>

      {adding ? (
        <ExternalResultForm patientId={patientId} consultationId={consultationId} onCancel={() => setAdding(false)}
          onDone={() => { setAdding(false); setNotice("Resultado externo registrado."); }} />
      ) : (
        <div style={row}>
          <button type="button" style={buttonStyle} onClick={() => { setNotice(null); setAdding(true); }}>Registrar resultado externo</button>
        </div>
      )}
    </section>
  );
}

function Labelled({ o }: { o: ExamOrder }): ReactNode {
  return (
    <span style={row}>
      <span>{o.label}</span>
      {o.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}
    </span>
  );
}
```

Em `src/modules/consultation/PatientPanel.tsx`:

- importe `import { PatientExams, type PatientExamsProps } from "../exams/PatientExams";`;
- a assinatura passa a `export function PatientPanel({ record, onOpenConsultation, besideProblems, timeline, exams }: { record: ClinicalRecord; onOpenConsultation(id: string): void; besideProblems?: ReactNode; timeline?: PatientTimelineProps; exams?: PatientExamsProps; })`;
- troque o estado das abas e o `onSummary`:

```tsx
  // Módulo 20/21a: cada aba só lê na primeira abertura e depois fica montada
  // (escondida), para a volta ao Resumo não perder filtros nem páginas.
  type PanelTab = "summary" | "timeline" | "exams";
  const [ tab, setTab ] = useState<PanelTab>("summary");
  const [ timelineOpened, setTimelineOpened ] = useState(false);
  const [ examsOpened, setExamsOpened ] = useState(false);
  const onTimeline = !!timeline && tab === "timeline";
  const onExams = !!exams && tab === "exams";
  const onSummary = !onTimeline && !onExams;
  const tabs: { key: PanelTab; label: string }[] = [
    { key: "summary", label: "Resumo" },
    ...(timeline ? [ { key: "timeline" as const, label: "Linha do tempo" } ] : []),
    ...(exams ? [ { key: "exams" as const, label: "Exames" } ] : [])
  ];
```

- troque o bloco `{timeline && ( <SegmentedControl … /> )}` por

```tsx
      {tabs.length > 1 && (
        <SegmentedControl<PanelTab> options={tabs} value={tab} onChange={(key) => {
          setTab(key);
          if (key === "timeline") setTimelineOpened(true);
          if (key === "exams") setExamsOpened(true);
        }} />
      )}
```

- troque o bloco final da linha do tempo e acrescente o dos exames:

```tsx
      {timeline && timelineOpened && (
        <div hidden={!onTimeline} style={onTimeline ? { display: "contents" } : undefined}>
          <PatientTimeline {...timeline} patientProblems={problems} />
        </div>
      )}
      {exams && examsOpened && (
        <div hidden={!onExams} style={onExams ? { display: "contents" } : undefined}>
          <PatientExams {...exams} />
        </div>
      )}
```

Em `src/modules/consultation/ConsultationWorkspace.tsx`, no `<PatientPanel record={record.data} …>` acrescente (usa o `examsOn` da Task 6):

```tsx
            exams={examsOn ? { patientId: record.data.patient.id, consultationId: consultation?.status === "draft" ? consultation.id : null } : undefined}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/exams src/modules/consultation && npx tsc --noEmit`
Expected: PASS — inclusive `PatientPanel.timeline.test.tsx` e `PatientPanel.test.tsx` antigos.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/exams/PatientExams.tsx src/modules/exams/ExternalResultForm.tsx src/modules/exams/PatientExams.test.tsx \
  src/modules/consultation/PatientPanel.tsx src/modules/consultation/PatientPanel.exams.test.tsx src/modules/consultation/ConsultationWorkspace.tsx
$G commit -m "feat: add the patient exams tab and external results

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Resultados aguardando ciência, negados e fila da equipe

**Files:**
- Create: `src/modules/ExamAcknowledgements.tsx`
- Modify: `src/App.tsx` (`renderModule`)
- Test: `src/modules/ExamAcknowledgements.test.tsx`

**Interfaces:**
- Consumes: `listTeamAcknowledgements`, `ExamOrder` (Task 1); `useExamPending` (Task 3); de `lib/exams`: `EXAMS_DISABLED`, `EXAMS_NO_ROLE`, `EXAM_TEAM_KEY`, `ESCALATION_LABEL`, `canUseExams`, `escalationReason`, `examError`, `examRoles`, `fmtDay`, `overdueLabel` (Task 2); `ExamOrderDetail` (Task 5).
- Produces: `ExamAcknowledgements()` (tela `exam-acknowledgements`): painéis "Aguardando sua ciência", "Exames negados" (só com negados) e "Fila da equipe"; "Abrir" mostra o `ExamOrderDetail` (que tem "Dar ciência" para quem pediu e "Dar ciência pela equipe" no escalado).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ExamAcknowledgements.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listPendingAcknowledgements: vi.fn(), listTeamAcknowledgements: vi.fn(), getExamOrder: vi.fn() };
});

import * as api from "../lib/api";
import { ExamAcknowledgements } from "./ExamAcknowledgements";
import { renderWithProviders } from "../test/campaignFixtures";
import { VALID_CNS, examOrder, examResult, examUser } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("ExamAcknowledgements", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listPendingAcknowledgements, api.listTeamAcknowledgements, api.getExamOrder ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser());
    mocked(api.listPendingAcknowledgements).mockResolvedValue({
      orders: [ examOrder({ status: "resulted", result: examResult({ critical: true }), overdue: { stage: "acknowledge", due_on: "2026-10-12", days_late: 1 } }) ],
      denied: [ examOrder({ id: "eo3", status: "denied", label: "Ressonância magnética de crânio" }) ]
    });
    mocked(api.listTeamAcknowledgements).mockResolvedValue([
      examOrder({ id: "eo4", status: "resulted", escalated: true, result: examResult(), requester: { user_id: null, name: "Dr. Externo", cns: VALID_CNS },
        patient: { id: "pa2", display_name: "Carlos Dias" } })
    ]);
    mocked(api.getExamOrder).mockResolvedValue(examOrder({ status: "resulted", result: examResult() }));
  });

  it("aguardando ciência com crítico e atraso; negados; fila da equipe com o motivo", async () => {
    renderWithProviders(<ExamAcknowledgements />);
    const mine = await screen.findByRole("region", { name: "Aguardando sua ciência" });
    await within(mine).findByText("Joana Lima");
    expect(within(mine).getByText("crítico")).not.toBeNull();
    expect(within(mine).getByText("ciência · 1 dia de atraso")).not.toBeNull();
    const denied = screen.getByRole("region", { name: "Exames negados" });
    expect(within(denied).getByText("Ressonância magnética de crânio")).not.toBeNull();
    const team = screen.getByRole("region", { name: "Fila da equipe" });
    await within(team).findByText("Carlos Dias");
    expect(within(team).getByText("solicitante sem vínculo")).not.toBeNull();
    expect(within(team).getByText("Dr. Externo")).not.toBeNull();
  });

  it("Abrir mostra o exame para dar ciência", async () => {
    renderWithProviders(<ExamAcknowledgements />);
    const mine = await screen.findByRole("region", { name: "Aguardando sua ciência" });
    fireEvent.click(await within(mine).findByRole("button", { name: "Abrir Hemograma completo de Joana Lima" }));
    expect(await screen.findByRole("region", { name: "Exame Hemograma completo" })).not.toBeNull();
  });

  it("sem negados, o painel não aparece; listas vazias dizem isso", async () => {
    mocked(api.listPendingAcknowledgements).mockResolvedValue({ orders: [], denied: [] });
    mocked(api.listTeamAcknowledgements).mockResolvedValue([]);
    renderWithProviders(<ExamAcknowledgements />);
    expect(await screen.findByText("nenhum resultado aguardando sua ciência")).not.toBeNull();
    expect(await screen.findByText("a fila da equipe está vazia")).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Exames negados" })).toBeNull();
  });

  it("rastreamento desligado ou papel sem acesso", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "health_professional" ], { features: [] }));
    renderWithProviders(<ExamAcknowledgements />);
    expect(await screen.findByText("o rastreamento de exames está desligado nesta cidade")).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "citizen_verifier" ]));
    renderWithProviders(<ExamAcknowledgements />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.listPendingAcknowledgements).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/ExamAcknowledgements.test.tsx`
Expected: FAIL — `./ExamAcknowledgements` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/ExamAcknowledgements.tsx
// Resultados aguardando ciência (módulo 21a; spec §5 etapa 6 e "Alertas";
// contrato §3, D7): os do próprio solicitante, os negados pela regulação e a
// fila da equipe de referência (crítico, prazo de ciência vencido ou
// solicitante sem vínculo). Quem está na equipe é do api.
import { useState, type ReactNode } from "react";
import { useQuery } from "@tanstack/react-query";
import { listTeamAcknowledgements, type ExamOrder } from "../lib/api";
import { useAuth } from "../lib/auth";
import {
  ESCALATION_LABEL, EXAMS_DISABLED, EXAMS_NO_ROLE, EXAM_TEAM_KEY, canUseExams, escalationReason, examError, examRoles, fmtDay, overdueLabel
} from "../lib/exams";
import { fmtDateTime } from "../lib/format";
import { useExamPending } from "../hooks/useExamPending";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable, type Column } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { Tag } from "../components/Tag";
import { secondaryButtonStyle } from "../components/formStyles";
import { ExamOrderDetail } from "./exams/ExamOrderDetail";
import { alert, muted, row, stack } from "./exams/styles";

export function ExamAcknowledgements() {
  const { user } = useAuth();
  const allowed = canUseExams(user) && examRoles(user).professional;
  const pending = useExamPending(allowed, user?.id ?? null);
  const team = useQuery({
    queryKey: [ EXAM_TEAM_KEY, user?.id ?? null ], queryFn: listTeamAcknowledgements, enabled: allowed, gcTime: 0, staleTime: 0, retry: false
  });
  const [ open, setOpen ] = useState<string | null>(null);

  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  if (!allowed) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;

  const exam: Column<ExamOrder>[] = [
    { label: "Paciente", w: "1.3fr", card: "title", render: (o) => o.patient.display_name },
    { label: "Exame", w: "1.7fr", render: (o) => (
      <span style={row}><span>{o.label}</span>{o.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}</span>
    ) }
  ];
  const openCol: Column<ExamOrder> = { label: "", w: "auto", align: "right", card: "actions", render: (o) => (
    <button type="button" aria-label={`Abrir ${o.label} de ${o.patient.display_name}`} style={secondaryButtonStyle} onClick={() => setOpen(o.id)}>Abrir</button>
  ) };
  const resultCol: Column<ExamOrder> = { label: "Resultado", w: "1fr", render: (o) => (
    <span style={row}><span>{fmtDay(o.result?.released_on)}</span>{o.result?.critical && <Tag tone="down">crítico</Tag>}</span>
  ) };

  return (
    <Frame>
      {open && <ExamOrderDetail key={open} id={open} onClose={() => setOpen(null)} />}
      <Panel title="Aguardando sua ciência" sub="resultados dos exames que você pediu">
        {pending.isPending && <p className="mono" style={muted}>carregando…</p>}
        {pending.isError && <p role="alert" style={alert}>{examError(pending.error)}</p>}
        {pending.isSuccess && (
          <DataTable cols={[ ...exam, resultCol, { label: "Prazo", w: "1.2fr", render: (o) => overdueLabel(o.overdue) }, openCol ]}
            rows={pending.data.orders} rowKey={(o) => o.id} empty="nenhum resultado aguardando sua ciência" />
        )}
      </Panel>
      {pending.isSuccess && pending.data.denied.length > 0 && (
        <Panel title="Exames negados" sub="decisão da regulação · o motivo está no andamento do exame">
          <DataTable cols={[ ...exam, { label: "Desde", w: "1fr", render: (o) => fmtDateTime(o.status_changed_at) }, openCol ]}
            rows={pending.data.denied} rowKey={(o) => o.id} />
        </Panel>
      )}
      <Panel title="Fila da equipe" sub="resultados críticos, com ciência vencida ou de solicitante sem vínculo">
        {team.isPending && <p className="mono" style={muted}>carregando…</p>}
        {team.isError && <p role="alert" style={alert}>{examError(team.error)}</p>}
        {team.isSuccess && (
          <DataTable cols={[ ...exam,
            { label: "Motivo", w: "1.2fr", render: (o) => ESCALATION_LABEL[escalationReason(o)] },
            { label: "Solicitante", w: "1.2fr", render: (o) => o.requester.name },
            resultCol, openCol ]}
            rows={team.data} rowKey={(o) => o.id} empty="a fila da equipe está vazia" />
        )}
      </Panel>
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Resultados aguardando ciência" sub="exames · ciência e fila da equipe" />
      {children}
    </div>
  );
}
```

Em `src/App.tsx`: `import { ExamAcknowledgements } from "./modules/ExamAcknowledgements";` e, no `switch` de `renderModule`, antes do `default`:

```tsx
    case "exam-acknowledgements": return <ExamAcknowledgements />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/ExamAcknowledgements.test.tsx src/App.navigation.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/ExamAcknowledgements.tsx src/modules/ExamAcknowledgements.test.tsx src/App.tsx
$G commit -m "feat: add the exam acknowledgement screen with denied exams and the team queue

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 9: Exames da unidade — guia da recepção, lista e resultado externo

**Files:**
- Create: `src/modules/ExamsUnit.tsx`, `src/modules/exams/PatientLookup.tsx`, `src/modules/exams/ReceptionGuideForm.tsx`, `src/modules/exams/UnitExamList.tsx`
- Modify: `src/App.tsx` (`renderModule`)
- Test: `src/modules/ExamsUnit.test.tsx`

**Interfaces:**
- Consumes: `lookupExamPatient`, `listExamCatalog`, `listExamRequesters`, `createReceptionOrder`, `listExamOrders`, `ExamPatient`, `ExamOrder`, `ExamStatus`, `HealthUnit` (Task 1); de `lib/exams`: `EMPTY_GUIDE`, `GuideDraft`, `guideProblem`, `guideBody`, `EXAM_CATALOG_KEY`, `EXAM_REQUESTERS_KEY`, `EXAM_ORDERS_KEY`, `EXAMS_DISABLED`, `EXAMS_NO_ROLE`, `canUseExams`, `examRoles`, `examError`, `overdueLabel`, `refreshExams` (Task 2); `isValidCpf`, `maskCpf`, `onlyDigits` (`lib/attendance`); `UnitPicker` (`modules/attendance/UnitPicker`); `useExamOrderPages`; `UploadField` (Task 4); `ExamOrderDetail` (Task 5); `ExternalResultForm` (Task 7).
- Produces: `ExamsUnit()` (tela `exams-unit`); `PatientLookup(props: { patient: ExamPatient | null; onFound(p: ExamPatient | null): void })`; `ReceptionGuideForm(props: { unit: HealthUnit; onDone(order: ExamOrder): void; onCancel(): void })` (`<form aria-label="Nova guia">`); `UnitExamList(props: { unit: HealthUnit })` (`Panel` "Exames da unidade" com abas por situação e "Carregar mais").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ExamsUnit.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listActiveUnits: vi.fn(), listExamCatalog: vi.fn(), listExamRequesters: vi.fn(),
    lookupExamPatient: vi.fn(), uploadExamFile: vi.fn(), createReceptionOrder: vi.fn(), listExamOrders: vi.fn(), getExamOrder: vi.fn(),
    listEnabledProviders: vi.fn(), recordExternalResult: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { ExamsUnit } from "./ExamsUnit";
import { renderWithProviders } from "../test/campaignFixtures";
import { HBA1C_ENTRY, HEMOGRAM_ENTRY, NOW21, UNIT21, VALID_CNS, VALID_CPF, examOrder, examUser, pdfFile } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listActiveUnits, api.listExamCatalog, api.listExamRequesters, api.lookupExamPatient,
  api.uploadExamFile, api.createReceptionOrder, api.listExamOrders, api.getExamOrder, api.listEnabledProviders, api.recordExternalResult ];

afterEach(() => { cleanup(); vi.useRealTimers(); });

async function openUnit() {
  fireEvent.click(await screen.findByRole("button", { name: "UBS Centro" }));
}

async function startGuide() {
  await openUnit();
  fireEvent.click(await screen.findByRole("button", { name: "Nova guia" }));
  const form = screen.getByRole("form", { name: "Nova guia" });
  fireEvent.change(within(form).getByLabelText("CPF do paciente"), { target: { value: VALID_CPF } });
  fireEvent.click(within(form).getByRole("button", { name: "Buscar paciente" }));
  await within(form).findByText("Paciente: Joana Lima · CPF ***.982.247-**");
  fireEvent.change(await within(form).findByLabelText("Exame"), { target: { value: "0202020380" } });
  return form;
}

async function attach(form: HTMLElement) {
  fireEvent.change(within(form).getByLabelText("Foto ou PDF da guia (até 10 MB)"), { target: { files: [ pdfFile("guia.pdf") ] } });
  await within(form).findByText("anexado: guia.pdf");
}

describe("ExamsUnit", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW21));
    localStorage.clear();
    for (const fn of ALL) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "citizen_verifier" ], { id: "us9" }));
    mocked(api.listActiveUnits).mockResolvedValue([ UNIT21 ]);
    mocked(api.listExamCatalog).mockResolvedValue([ HEMOGRAM_ENTRY, HBA1C_ENTRY ]);
    mocked(api.listExamRequesters).mockResolvedValue([ { user_id: "us1", name: "Dra. Ana Souza" } ]);
    mocked(api.lookupExamPatient).mockResolvedValue({ id: "pa1", display_name: "Joana Lima", cpf_masked: "***.982.247-**" });
    mocked(api.uploadExamFile).mockResolvedValue("up1");
    mocked(api.createReceptionOrder).mockResolvedValue(examOrder({ origin: "reception_guide" }));
    mocked(api.listExamOrders).mockResolvedValue({ orders: [ examOrder() ], next_before: null });
    mocked(api.listEnabledProviders).mockResolvedValue([]);
  });

  it("guia com profissional da rede: busca o paciente pelo CPF no corpo e registra", async () => {
    renderWithProviders(<ExamsUnit />);
    const form = await startGuide();
    expect(api.lookupExamPatient).toHaveBeenCalledWith(VALID_CPF);
    expect(api.listExamRequesters).toHaveBeenCalledWith("u1");
    fireEvent.click(within(form).getByLabelText("Urgente"));
    fireEvent.change(await within(form).findByLabelText("Profissional que pediu"), { target: { value: "us1" } });
    fireEvent.change(within(form).getByLabelText("CID-10 de justificativa (opcional)"), { target: { value: "e11" } });
    await attach(form);
    fireEvent.click(within(form).getByRole("button", { name: "Registrar guia" }));
    await waitFor(() => expect(api.createReceptionOrder).toHaveBeenCalledWith({
      patient_id: "pa1", unit_id: "u1", sigtap_code: "0202020380", priority: "urgent", cid10_justification: "E11",
      requester: { user_id: "us1" }, guide_upload_id: "up1"
    }));
    expect((await screen.findByText("Guia registrada · código K7M2Q9XW3P")).getAttribute("role")).toBe("status");
    expect(screen.queryByRole("form", { name: "Nova guia" })).toBeNull();
  });

  it("solicitante digitado: CNS inválido não envia; válido manda nome e CNS", async () => {
    renderWithProviders(<ExamsUnit />);
    const form = await startGuide();
    fireEvent.click(within(form).getByLabelText("Outro (nome e CNS)"));
    fireEvent.change(within(form).getByLabelText("Nome de quem pediu"), { target: { value: "Dr. Externo" } });
    fireEvent.change(within(form).getByLabelText("CNS de quem pediu"), { target: { value: "123" } });
    await attach(form);
    fireEvent.click(within(form).getByRole("button", { name: "Registrar guia" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("CNS de quem pediu inválido");
    expect(api.createReceptionOrder).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("CNS de quem pediu"), { target: { value: VALID_CNS } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar guia" }));
    await waitFor(() => expect(mocked(api.createReceptionOrder).mock.calls[0][0].requester).toEqual({ name: "Dr. Externo", cns: VALID_CNS }));
  });

  it("guia sem anexo não é enviada", async () => {
    renderWithProviders(<ExamsUnit />);
    const form = await startGuide();
    fireEvent.change(await within(form).findByLabelText("Profissional que pediu"), { target: { value: "us1" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar guia" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("anexe a foto ou o PDF da guia");
    expect(api.createReceptionOrder).not.toHaveBeenCalled();
  });

  it("CPF sem paciente na cidade e CPF inválido explicam", async () => {
    mocked(api.lookupExamPatient).mockRejectedValue(new ApiError(404, { error: "patient_not_found" }, "x"));
    renderWithProviders(<ExamsUnit />);
    await openUnit();
    fireEvent.click(await screen.findByRole("button", { name: "Nova guia" }));
    const form = screen.getByRole("form", { name: "Nova guia" });
    fireEvent.change(within(form).getByLabelText("CPF do paciente"), { target: { value: "11111111111" } });
    fireEvent.click(within(form).getByRole("button", { name: "Buscar paciente" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("CPF inválido");
    expect(api.lookupExamPatient).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("CPF do paciente"), { target: { value: VALID_CPF } });
    fireEvent.click(within(form).getByRole("button", { name: "Buscar paciente" }));
    await waitFor(() => expect(within(form).getByRole("alert").textContent).toBe("nenhum paciente com este CPF na cidade"));
  });

  it("422 sigtap_not_in_catalog do api aparece no formulário", async () => {
    mocked(api.createReceptionOrder).mockRejectedValue(new ApiError(422, { error: "sigtap_not_in_catalog" }, "x"));
    renderWithProviders(<ExamsUnit />);
    const form = await startGuide();
    fireEvent.change(await within(form).findByLabelText("Profissional que pediu"), { target: { value: "us1" } });
    await attach(form);
    fireEvent.click(within(form).getByRole("button", { name: "Registrar guia" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("este exame não está no catálogo da cidade");
  });

  it("lista da unidade por situação; Abrir mostra o exame", async () => {
    mocked(api.getExamOrder).mockResolvedValue(examOrder());
    renderWithProviders(<ExamsUnit />);
    await openUnit();
    const list = await screen.findByRole("region", { name: "Exames da unidade" });
    await waitFor(() => expect(api.listExamOrders).toHaveBeenCalledWith({ unitId: "u1", statuses: [ "awaiting_schedule" ], before: null }));
    fireEvent.click(within(list).getByRole("tab", { name: "Agendados" }));
    await waitFor(() => expect(api.listExamOrders).toHaveBeenCalledWith({ unitId: "u1", statuses: [ "scheduled" ], before: null }));
    fireEvent.click(await within(list).findByRole("button", { name: "Abrir Hemograma completo de Joana Lima" }));
    expect(await screen.findByRole("region", { name: "Exame Hemograma completo" })).not.toBeNull();
  });

  it("papel sem acesso e rastreamento desligado", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "viewer" ]));
    renderWithProviders(<ExamsUnit />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "citizen_verifier" ], { features: [] }));
    renderWithProviders(<ExamsUnit />);
    expect(await screen.findByText("o rastreamento de exames está desligado nesta cidade")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/ExamsUnit.test.tsx`
Expected: FAIL — `./ExamsUnit` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/exams/PatientLookup.tsx
// O paciente pelo CPF (Divergência D2): CPF no corpo de um POST, nunca na
// URL; a tela guarda só o id, o nome de exibição e o CPF mascarado.
import { useState } from "react";
import { lookupExamPatient, type ExamPatient } from "../../lib/api";
import { isValidCpf, maskCpf, onlyDigits } from "../../lib/attendance";
import { examError } from "../../lib/exams";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, row } from "./styles";

export function PatientLookup({ patient, onFound }: { patient: ExamPatient | null; onFound(p: ExamPatient | null): void }) {
  const [ cpf, setCpf ] = useState("");
  const [ error, setError ] = useState<string | null>(null);

  async function find() {
    if (!isValidCpf(cpf)) { setError("CPF inválido"); return; }
    setError(null);
    try {
      onFound(await lookupExamPatient(onlyDigits(cpf)));
      setCpf("");
    } catch (err) {
      setError(examError(err));
    }
  }

  if (patient) {
    return (
      <div style={row}>
        <span>{`Paciente: ${patient.display_name} · CPF ${patient.cpf_masked}`}</span>
        <button type="button" style={secondaryButtonStyle} onClick={() => onFound(null)}>Trocar paciente</button>
      </div>
    );
  }
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <div style={{ ...row, alignItems: "flex-end" }}>
        <label style={fieldLabel}>
          CPF do paciente
          <input value={cpf} inputMode="numeric" autoComplete="off" style={inputStyle} onChange={(e) => setCpf(maskCpf(e.target.value))} />
        </label>
        <ActionButton variant="secondary" onClick={find}>Buscar paciente</ActionButton>
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
    </div>
  );
}
```

```tsx
// src/modules/exams/ReceptionGuideForm.tsx
// Guia da recepção (spec §6; contrato §3 POST /exams/orders): pedido feito
// fora do Rota Saúde, cadastrado pela guia de papel. Paciente pelo CPF,
// exame do catálogo da cidade (D3), prioridade, CID-10 opcional, quem pediu
// (profissional com vínculo na unidade — D5 — ou nome e CNS digitados) e a
// foto ou o PDF da guia (obrigatório). O CPF sai da tela no sucesso.
import { useState, type FormEvent } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  createReceptionOrder, listExamCatalog, listExamRequesters, type ExamOrder, type ExamPatient, type HealthUnit
} from "../../lib/api";
import {
  EMPTY_GUIDE, EXAM_CATALOG_KEY, EXAM_REQUESTERS_KEY, examError, guideBody, guideProblem, refreshExams, type GuideDraft
} from "../../lib/exams";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { PatientLookup } from "./PatientLookup";
import { UploadField } from "./UploadField";
import { alert, checkLabel, fieldLabel, fieldset, form, legend, row } from "./styles";

export function ReceptionGuideForm({ unit, onDone, onCancel }: { unit: HealthUnit; onDone(order: ExamOrder): void; onCancel(): void }) {
  const queryClient = useQueryClient();
  const catalog = useQuery({ queryKey: [ EXAM_CATALOG_KEY ], queryFn: listExamCatalog, retry: false });
  const requesters = useQuery({ queryKey: [ EXAM_REQUESTERS_KEY, unit.id ], queryFn: () => listExamRequesters(unit.id), retry: false });
  const [ patient, setPatient ] = useState<ExamPatient | null>(null);
  const [ draft, setDraft ] = useState<GuideDraft>(EMPTY_GUIDE);
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const set = (patch: Partial<GuideDraft>) => setDraft((d) => ({ ...d, ...patch }));

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    if (!patient) { setError("busque o paciente pelo CPF"); return; }
    const problem = guideProblem(draft);
    if (problem) { setError(problem); return; }
    setBusy(true);
    setError(null);
    try {
      const order = await createReceptionOrder(guideBody(draft, patient.id, unit.id));
      await refreshExams(queryClient);
      setPatient(null);
      setDraft(EMPTY_GUIDE);
      onDone(order);
    } catch (err) {
      setError(examError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <form aria-label="Nova guia" onSubmit={submit} style={form}>
      <PatientLookup patient={patient} onFound={setPatient} />
      {catalog.isError && <p role="alert" style={alert}>{examError(catalog.error)}</p>}
      {catalog.isSuccess && (
        <label style={fieldLabel}>
          Exame
          <select value={draft.sigtapCode} style={inputStyle} onChange={(e) => set({ sigtapCode: e.target.value })}>
            <option value="">escolha…</option>
            {catalog.data.map((c) => <option key={c.sigtap_code} value={c.sigtap_code}>{`${c.label} (${c.sigtap_code})`}</option>)}
          </select>
        </label>
      )}
      <fieldset style={fieldset}>
        <legend style={legend}>Prioridade</legend>
        <label style={checkLabel}>
          <input type="radio" name="guide-priority" checked={draft.priority === "routine"} onChange={() => set({ priority: "routine" })} />Rotina
        </label>
        <label style={checkLabel}>
          <input type="radio" name="guide-priority" checked={draft.priority === "urgent"} onChange={() => set({ priority: "urgent" })} />Urgente
        </label>
      </fieldset>
      <label style={fieldLabel}>
        CID-10 de justificativa (opcional)
        <input value={draft.cid10} placeholder="ex.: E11" style={inputStyle} onChange={(e) => set({ cid10: e.target.value })} />
      </label>
      <fieldset style={fieldset}>
        <legend style={legend}>Quem pediu</legend>
        <label style={checkLabel}>
          <input type="radio" name="guide-requester" checked={draft.requesterMode === "linked"} onChange={() => set({ requesterMode: "linked" })} />
          Profissional da rede
        </label>
        <label style={checkLabel}>
          <input type="radio" name="guide-requester" checked={draft.requesterMode === "typed"} onChange={() => set({ requesterMode: "typed" })} />
          Outro (nome e CNS)
        </label>
        {draft.requesterMode === "linked" && requesters.isError && <p role="alert" style={alert}>{examError(requesters.error)}</p>}
        {draft.requesterMode === "linked" && requesters.isSuccess && (
          <label style={fieldLabel}>
            Profissional que pediu
            <select value={draft.requesterId} style={inputStyle} onChange={(e) => set({ requesterId: e.target.value })}>
              <option value="">escolha…</option>
              {requesters.data.map((r) => <option key={r.user_id} value={r.user_id}>{r.name}</option>)}
            </select>
          </label>
        )}
        {draft.requesterMode === "typed" && (
          <>
            <label style={fieldLabel}>
              Nome de quem pediu
              <input value={draft.requesterName} style={inputStyle} onChange={(e) => set({ requesterName: e.target.value })} />
            </label>
            <label style={fieldLabel}>
              CNS de quem pediu
              <input value={draft.requesterCns} inputMode="numeric" style={inputStyle} onChange={(e) => set({ requesterCns: e.target.value })} />
            </label>
          </>
        )}
      </fieldset>
      <label style={fieldLabel}>
        Número da guia no prestador (opcional)
        <input value={draft.externalReference} style={inputStyle} onChange={(e) => set({ externalReference: e.target.value })} />
      </label>
      <UploadField label="Foto ou PDF da guia (até 10 MB)" uploadId={draft.guideUploadId} onUploaded={(guideUploadId) => set({ guideUploadId })} />
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <button type="submit" aria-disabled={busy || undefined} style={busy ? disabledButtonStyle : buttonStyle}>Registrar guia</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/exams/UnitExamList.tsx
// Os exames da unidade por situação (spec §9 "Recepção/unidade"): a unidade
// abre cada um para agendar, registrar coleta e resultado (ExamOrderDetail).
import { useState } from "react";
import { listExamOrders, type ExamOrder, type ExamStatus, type HealthUnit } from "../../lib/api";
import { EXAM_ORDERS_KEY, examError, overdueLabel } from "../../lib/exams";
import { fmtDateTime } from "../../lib/format";
import { useExamOrderPages } from "../../hooks/useExamOrderPages";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { SegmentedControl } from "../../shell/SegmentedControl";
import { ExamOrderDetail } from "./ExamOrderDetail";
import { alert, muted, row } from "./styles";

const TABS: { key: ExamStatus; label: string }[] = [
  { key: "awaiting_authorization", label: "Aguardando autorização" },
  { key: "awaiting_schedule", label: "A agendar" },
  { key: "scheduled", label: "Agendados" },
  { key: "collected", label: "Coletados" },
  { key: "resulted", label: "Com resultado" }
];

export function UnitExamList({ unit }: { unit: HealthUnit }) {
  const [ tab, setTab ] = useState<ExamStatus>("awaiting_schedule");
  const [ open, setOpen ] = useState<string | null>(null);
  const { query, orders } = useExamOrderPages([ EXAM_ORDERS_KEY, "unit", unit.id, tab ],
    (before) => listExamOrders({ unitId: unit.id, statuses: [ tab ], before }));

  return (
    <Panel title="Exames da unidade" sub={unit.name}>
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        <SegmentedControl<ExamStatus> options={TABS} value={tab} onChange={(t) => { setTab(t); setOpen(null); }} />
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
        {query.isSuccess && (
          <DataTable<ExamOrder>
            cols={[
              { label: "Código", w: "1fr", render: (o) => <span className="mono">{o.short_code}</span> },
              { label: "Paciente", w: "1.3fr", card: "title", render: (o) => o.patient.display_name },
              { label: "Exame", w: "1.6fr", render: (o) => (
                <span style={row}><span>{o.label}</span>{o.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}</span>
              ) },
              { label: "Prestador", w: "1.2fr", render: (o) => o.provider?.name ?? "—" },
              { label: "Agendado", w: "1fr", render: (o) => (o.scheduled_at ? fmtDateTime(o.scheduled_at) : "—") },
              { label: "Prazo", w: "1.2fr", render: (o) => overdueLabel(o.overdue) },
              { label: "", w: "auto", align: "right", card: "actions", render: (o) => (
                <button type="button" aria-label={`Abrir ${o.label} de ${o.patient.display_name}`} style={secondaryButtonStyle}
                  onClick={() => setOpen(o.id)}>Abrir</button>
              ) }
            ]}
            rows={orders} rowKey={(o) => o.id} empty="nenhum exame nesta situação"
          />
        )}
        {query.hasNextPage && (
          <div><button type="button" style={secondaryButtonStyle} onClick={() => void query.fetchNextPage()}>Carregar mais</button></div>
        )}
        {open && <ExamOrderDetail key={open} id={open} onClose={() => setOpen(null)} />}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/ExamsUnit.tsx
// Exames da unidade (módulo 21a; spec §9 "Recepção/unidade"): a unidade
// escolhida (a mesma do Atendimento), a guia da recepção, o resultado
// externo e a lista para agendar, registrar coleta e resultado. A guia é de
// quem o api aceita (recepção e profissional, contrato §3); o admin vê e age.
import { useState, type ReactNode } from "react";
import type { ExamPatient, HealthUnit } from "../lib/api";
import { useAuth } from "../lib/auth";
import { EXAMS_DISABLED, EXAMS_NO_ROLE, canUseExams, examRoles } from "../lib/exams";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { EmptyState } from "../components/EmptyState";
import { buttonStyle, secondaryButtonStyle } from "../components/formStyles";
import { UnitPicker } from "./attendance/UnitPicker";
import { ReceptionGuideForm } from "./exams/ReceptionGuideForm";
import { PatientLookup } from "./exams/PatientLookup";
import { ExternalResultForm } from "./exams/ExternalResultForm";
import { UnitExamList } from "./exams/UnitExamList";
import { okText, row, stack } from "./exams/styles";

type Mode = null | "guide" | "external";

export function ExamsUnit() {
  const { user } = useAuth();
  const [ unit, setUnit ] = useState<HealthUnit | null>(null);
  const [ mode, setMode ] = useState<Mode>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ externalPatient, setExternalPatient ] = useState<ExamPatient | null>(null);

  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  const roles = examRoles(user);
  if (!(roles.reception || roles.professional || roles.admin)) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;
  const canGuide = roles.reception || roles.professional;
  const close = (text: string | null) => { setExternalPatient(null); setMode(null); setNotice(text); };

  return (
    <Frame>
      <UnitPicker userId={user.id} onChange={(u) => { setUnit(u); close(null); }} />
      {unit && (
        <>
          {notice && <p role="status" style={okText}>{notice}</p>}
          {mode === null && (
            <div style={row}>
              {canGuide && <button type="button" style={buttonStyle} onClick={() => { setNotice(null); setMode("guide"); }}>Nova guia</button>}
              <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setMode("external"); }}>Resultado externo</button>
            </div>
          )}
          {mode === "guide" && (
            <Panel title="Nova guia" sub="pedido feito fora do Rota Saúde, cadastrado pela guia de papel">
              <ReceptionGuideForm unit={unit} onCancel={() => close(null)} onDone={(o) => close(`Guia registrada · código ${o.short_code}`)} />
            </Panel>
          )}
          {mode === "external" && (
            <Panel title="Registrar resultado externo" sub="exame feito fora de um pedido do Rota Saúde">
              <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
                <PatientLookup patient={externalPatient} onFound={setExternalPatient} />
                {externalPatient ? (
                  <ExternalResultForm patientId={externalPatient.id} onCancel={() => close(null)}
                    onDone={() => close("Resultado externo registrado.")} />
                ) : (
                  <div><button type="button" style={secondaryButtonStyle} onClick={() => close(null)}>Voltar</button></div>
                )}
              </div>
            </Panel>
          )}
          <UnitExamList unit={unit} />
        </>
      )}
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Exames da unidade" sub="guias, agendamento, coleta e resultado" />
      {children}
    </div>
  );
}
```

Em `src/App.tsx`: `import { ExamsUnit } from "./modules/ExamsUnit";` e `case "exams-unit": return <ExamsUnit />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/ExamsUnit.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/ExamsUnit.tsx src/modules/ExamsUnit.test.tsx src/modules/exams/PatientLookup.tsx src/modules/exams/ReceptionGuideForm.tsx \
  src/modules/exams/UnitExamList.tsx src/App.tsx
$G commit -m "feat: add the unit exams screen with the reception guide and external results

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Painel "Exames atrasados" — da unidade e da coordenação

**Files:**
- Create: `src/modules/exams/OverduePanel.tsx`, `src/modules/ExamsOverdue.tsx`
- Modify: `src/modules/ExamsUnit.tsx`, `src/App.tsx` (`renderModule`)
- Test: `src/modules/ExamsOverdue.test.tsx`

**Interfaces:**
- Consumes: `listExamOrders`, `listEnabledProviders`, `listExamCatalog`, `ExamOrderQuery`, `ExamStage` (Task 1); `EXAM_ORDERS_KEY`, `EXAM_PROVIDER_REFS_KEY`, `EXAM_CATALOG_KEY`, `EXAM_STAGES`, `STAGE_LABEL`, `daysLateLabel`, `examError`, `canUseExams`, `examRoles`, `EXAMS_DISABLED`, `EXAMS_NO_ROLE` (Task 2); `useExamOrderPages`; `ExamOrderDetail` (Task 5).
- Produces: `OverduePanel(props: { unitId?: string; canOpen: boolean })` (`Panel` "Exames atrasados"; filtros "Etapa", "Prestador", "Exame"); `ExamsOverdue()` (tela `exams-overdue`, coordenação: sem unidade, sem "Abrir" — Divergência D15). `ExamsUnit` passa a mostrar `<OverduePanel unitId={unit.id} canOpen />` depois da lista.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ExamsOverdue.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listExamOrders: vi.fn(), listEnabledProviders: vi.fn(), listExamCatalog: vi.fn(), getExamOrder: vi.fn() };
});

import * as api from "../lib/api";
import { ExamsOverdue } from "./ExamsOverdue";
import { OverduePanel } from "./exams/OverduePanel";
import { renderWithProviders } from "../test/campaignFixtures";
import { HBA1C_ENTRY, HEMOGRAM_ENTRY, examOrder, examUser } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("Exames atrasados", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listExamOrders, api.listEnabledProviders, api.listExamCatalog, api.getExamOrder ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "citizen_verifier" ]));
    mocked(api.listExamOrders).mockResolvedValue({
      orders: [ examOrder({ status: "scheduled", overdue: { stage: "collect", due_on: "2026-10-10", days_late: 3 } }) ], next_before: null
    });
    mocked(api.listEnabledProviders).mockResolvedValue([ { id: "pv1", name: "Laboratório Simulado" } ]);
    mocked(api.listExamCatalog).mockResolvedValue([ HEMOGRAM_ENTRY, HBA1C_ENTRY ]);
    mocked(api.getExamOrder).mockResolvedValue(examOrder());
  });

  it("unidade: etapa, prestador e exame vão para a consulta; linha com etapa e dias", async () => {
    renderWithProviders(<OverduePanel unitId="u1" canOpen />);
    const panel = await screen.findByRole("region", { name: "Exames atrasados" });
    await waitFor(() => expect(api.listExamOrders).toHaveBeenCalledWith({ overdue: true, before: null, unitId: "u1" }));
    expect(await within(panel).findByText("3 dias de atraso")).not.toBeNull();
    // "coleta" também é opção do filtro de etapa: confira na linha.
    expect(within(within(panel).getAllByRole("row")[1]).getByText("coleta")).not.toBeNull();
    fireEvent.change(within(panel).getByLabelText("Etapa"), { target: { value: "collect" } });
    fireEvent.change(await within(panel).findByLabelText("Prestador"), { target: { value: "pv1" } });
    fireEvent.change(await within(panel).findByLabelText("Exame"), { target: { value: "0202020380" } });
    await waitFor(() => expect(api.listExamOrders).toHaveBeenLastCalledWith({
      overdue: true, before: null, unitId: "u1", stage: "collect", providerId: "pv1", sigtapCode: "0202020380"
    }));
    expect(within(panel).getByRole("status").textContent).toBe("1 exame atrasado");
    fireEvent.click(within(panel).getByRole("button", { name: "Abrir Hemograma completo de Joana Lima" }));
    expect(await screen.findByRole("region", { name: "Exame Hemograma completo" })).not.toBeNull();
  });

  it("coordenação: a cidade inteira, com a unidade e sem abrir o exame", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "exam_regulator" ]));
    renderWithProviders(<ExamsOverdue />);
    const panel = await screen.findByRole("region", { name: "Exames atrasados" });
    await waitFor(() => expect(api.listExamOrders).toHaveBeenCalledWith({ overdue: true, before: null }));
    expect(await within(panel).findByText("UBS Centro")).not.toBeNull();
    expect(within(panel).queryByRole("button", { name: /^Abrir/ })).toBeNull();
  });

  it("coordenação é do admin e do regulador", async () => {
    renderWithProviders(<ExamsOverdue />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.listExamOrders).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/ExamsOverdue.test.tsx`
Expected: FAIL — os componentes não existem.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/exams/OverduePanel.tsx
// Painel "Exames atrasados" (spec §5 "Alertas"): o atraso por etapa é do api
// (job diário); aqui só os filtros por etapa, prestador e exame. Na unidade,
// cada exame abre; na coordenação (cidade inteira, D15) a lista é só de
// acompanhamento — a leitura do exame segue as regras do 19a na unidade.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { listEnabledProviders, listExamCatalog, listExamOrders, type ExamOrder, type ExamOrderQuery, type ExamStage } from "../../lib/api";
import {
  EXAM_CATALOG_KEY, EXAM_ORDERS_KEY, EXAM_PROVIDER_REFS_KEY, EXAM_STAGES, STAGE_LABEL, daysLateLabel, examError
} from "../../lib/exams";
import { useExamOrderPages } from "../../hooks/useExamOrderPages";
import { Panel } from "../../components/Panel";
import { DataTable, type Column } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ExamOrderDetail } from "./ExamOrderDetail";
import { alert, fieldLabel, muted, row } from "./styles";

export function OverduePanel({ unitId, canOpen }: { unitId?: string; canOpen: boolean }) {
  const [ stage, setStage ] = useState<ExamStage | "">("");
  const [ providerId, setProviderId ] = useState("");
  const [ sigtapCode, setSigtapCode ] = useState("");
  const [ open, setOpen ] = useState<string | null>(null);
  const providers = useQuery({ queryKey: [ EXAM_PROVIDER_REFS_KEY, "" ], queryFn: () => listEnabledProviders(), retry: false });
  const catalog = useQuery({ queryKey: [ EXAM_CATALOG_KEY ], queryFn: listExamCatalog, retry: false });
  const { query, orders } = useExamOrderPages(
    [ EXAM_ORDERS_KEY, "overdue", unitId ?? "city", stage, providerId, sigtapCode ],
    (before) => {
      const q: ExamOrderQuery = { overdue: true, before };
      if (unitId) q.unitId = unitId;
      if (stage) q.stage = stage;
      if (providerId) q.providerId = providerId;
      if (sigtapCode) q.sigtapCode = sigtapCode;
      return listExamOrders(q);
    });

  const cols: Column<ExamOrder>[] = [
    { label: "Código", w: "1fr", render: (o) => <span className="mono">{o.short_code}</span> },
    { label: "Paciente", w: "1.3fr", card: "title", render: (o) => o.patient.display_name },
    { label: "Exame", w: "1.6fr", render: (o) => (
      <span style={row}><span>{o.label}</span>{o.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}</span>
    ) },
    { label: "Etapa", w: "0.9fr", render: (o) => (o.overdue ? STAGE_LABEL[o.overdue.stage] ?? o.overdue.stage : "—") },
    { label: "Atraso", w: "1fr", render: (o) => (o.overdue ? daysLateLabel(o.overdue.days_late) : "—") },
    { label: "Prestador", w: "1.2fr", render: (o) => o.provider?.name ?? "—" },
    ...(unitId ? [] : [ { label: "Unidade", w: "1.2fr", render: (o: ExamOrder) => o.unit.name } ]),
    ...(canOpen ? [ { label: "", w: "auto", align: "right" as const, card: "actions" as const, render: (o: ExamOrder) => (
      <button type="button" aria-label={`Abrir ${o.label} de ${o.patient.display_name}`} style={secondaryButtonStyle}
        onClick={() => setOpen(o.id)}>Abrir</button>
    ) } ] : [])
  ];
  const count = `${orders.length}${query.hasNextPage ? "+" : ""}`;

  return (
    <Panel title="Exames atrasados" sub={unitId ? "desta unidade · por etapa, prestador e exame" : "da cidade · por etapa, prestador e exame"}>
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>
            Etapa
            <select value={stage} style={inputStyle} onChange={(e) => setStage(e.target.value as ExamStage | "")}>
              <option value="">todas</option>
              {EXAM_STAGES.map((s) => <option key={s} value={s}>{STAGE_LABEL[s]}</option>)}
            </select>
          </label>
          {providers.isSuccess && (
            <label style={fieldLabel}>
              Prestador
              <select value={providerId} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
                <option value="">todos</option>
                {providers.data.map((p) => <option key={p.id} value={p.id}>{p.name}</option>)}
              </select>
            </label>
          )}
          {catalog.isSuccess && (
            <label style={fieldLabel}>
              Exame
              <select value={sigtapCode} style={inputStyle} onChange={(e) => setSigtapCode(e.target.value)}>
                <option value="">todos</option>
                {catalog.data.map((c) => <option key={c.sigtap_code} value={c.sigtap_code}>{c.label}</option>)}
              </select>
            </label>
          )}
        </div>
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
        {query.isSuccess && (
          <>
            <p role="status" style={muted}>{orders.length === 1 && !query.hasNextPage ? "1 exame atrasado" : `${count} exames atrasados`}</p>
            <DataTable cols={cols} rows={orders} rowKey={(o) => o.id} empty="nenhum exame atrasado com estes filtros" />
          </>
        )}
        {query.hasNextPage && (
          <div><button type="button" style={secondaryButtonStyle} onClick={() => void query.fetchNextPage()}>Carregar mais</button></div>
        )}
        {open && <ExamOrderDetail key={open} id={open} onClose={() => setOpen(null)} />}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/ExamsOverdue.tsx
// Exames atrasados da cidade (coordenação: municipal_admin e exam_regulator).
import type { ReactNode } from "react";
import { useAuth } from "../lib/auth";
import { EXAMS_DISABLED, EXAMS_NO_ROLE, canUseExams, examRoles } from "../lib/exams";
import { PageHeader } from "../components/PageHeader";
import { EmptyState } from "../components/EmptyState";
import { OverduePanel } from "./exams/OverduePanel";
import { stack } from "./exams/styles";

export function ExamsOverdue() {
  const { user } = useAuth();
  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  const roles = examRoles(user);
  if (!(roles.admin || roles.regulator)) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;
  return <Frame><OverduePanel canOpen={false} /></Frame>;
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Exames atrasados" sub="coordenação · cidade inteira" />
      {children}
    </div>
  );
}
```

Em `src/modules/ExamsUnit.tsx`, importe `OverduePanel` de `./exams/OverduePanel` e, logo depois de `<UnitExamList unit={unit} />`, acrescente `<OverduePanel unitId={unit.id} canOpen />`.

Em `src/App.tsx`: `import { ExamsOverdue } from "./modules/ExamsOverdue";` e `case "exams-overdue": return <ExamsOverdue />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/ExamsOverdue.test.tsx src/modules/ExamsUnit.test.tsx && npx tsc --noEmit`
Expected: PASS (o `ExamsUnit.test.tsx` já tem `listEnabledProviders` dublado).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/exams/OverduePanel.tsx src/modules/ExamsOverdue.tsx src/modules/ExamsOverdue.test.tsx src/modules/ExamsUnit.tsx src/App.tsx
$G commit -m "feat: add the overdue exams panel for units and coordination

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Fila de regulação

**Files:**
- Create: `src/modules/ExamRegulation.tsx`
- Modify: `src/App.tsx` (`renderModule`)
- Test: `src/modules/ExamRegulation.test.tsx`

**Interfaces:**
- Consumes: `listRegulationQueue`, `listEnabledProviders`, `authorizeExam`, `denyExam`, `errorCode`, `ExamOrder` (Task 1); `EXAM_REGULATION_KEY`, `EXAM_PROVIDER_REFS_KEY`, `EXAMS_DISABLED`, `EXAMS_NO_ROLE`, `PRIORITY_LABEL`, `canUseExams`, `examRoles`, `examError`, `overdueLabel`, `refreshExams` (Task 2); `useExamOrderPages`; `ReasonForm` (Task 4).
- Produces: `ExamRegulation()` (tela `exam-regulation`): a fila na ordem do api (urgente primeiro, depois o mais antigo — contrato §4), "Decidir" abre `<section aria-label={`Decidir: ${label} de ${paciente}`}>` com "Autorizar" (prestador obrigatório) e "Negar" (motivo obrigatório).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ExamRegulation.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listRegulationQueue: vi.fn(), listEnabledProviders: vi.fn(), authorizeExam: vi.fn(), denyExam: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { ExamRegulation } from "./ExamRegulation";
import { renderWithProviders } from "../test/campaignFixtures";
import { examOrder, examUser } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const WAITING = examOrder({ status: "awaiting_authorization", priority: "urgent", provider: null, cid10_justification: "E11" });
afterEach(cleanup);

async function decide() {
  fireEvent.click(await screen.findByRole("button", { name: "Decidir Hemograma completo de Joana Lima" }));
  return screen.getByRole("region", { name: "Decidir: Hemograma completo de Joana Lima" });
}

describe("ExamRegulation", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listRegulationQueue, api.listEnabledProviders, api.authorizeExam, api.denyExam ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "exam_regulator" ], { id: "rg1" }));
    mocked(api.listRegulationQueue).mockResolvedValue({ orders: [ WAITING ], next_before: null });
    mocked(api.listEnabledProviders).mockResolvedValue([ { id: "pv1", name: "Laboratório Simulado" } ]);
    mocked(api.authorizeExam).mockResolvedValue(undefined);
    mocked(api.denyExam).mockResolvedValue(undefined);
  });

  it("fila com prioridade e justificativa; autorizar exige e manda o prestador", async () => {
    renderWithProviders(<ExamRegulation />);
    const queue = await screen.findByRole("region", { name: "Fila de regulação" });
    expect(await within(queue).findByText("urgente")).not.toBeNull();
    const panel = await decide();
    expect(within(panel).getByText("E11")).not.toBeNull();
    expect(api.listEnabledProviders).toHaveBeenCalledWith("0202020380");
    fireEvent.click(within(panel).getByRole("button", { name: "Autorizar" }));
    expect((await within(panel).findByRole("alert")).textContent).toBe("escolha o prestador");
    fireEvent.change(await within(panel).findByLabelText("Prestador"), { target: { value: "pv1" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Autorizar" }));
    await waitFor(() => expect(api.authorizeExam).toHaveBeenCalledWith("eo1", "pv1"));
    expect((await screen.findByText("Exame autorizado para Laboratório Simulado.")).getAttribute("role")).toBe("status");
    await waitFor(() => expect(api.listRegulationQueue).toHaveBeenCalledTimes(2));
  });

  it("negar exige o motivo", async () => {
    renderWithProviders(<ExamRegulation />);
    const panel = await decide();
    fireEvent.click(within(panel).getByRole("button", { name: "Negar" }));
    const form = within(panel).getByRole("form", { name: "Motivo da negativa" });
    fireEvent.click(within(form).getByRole("button", { name: "Negar exame" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("escreva o motivo (ao menos 5 caracteres)");
    fireEvent.change(within(form).getByLabelText("Motivo da negativa"), { target: { value: "Fora do protocolo municipal" } });
    fireEvent.click(within(form).getByRole("button", { name: "Negar exame" }));
    await waitFor(() => expect(api.denyExam).toHaveBeenCalledWith("eo1", "Fora do protocolo municipal"));
    expect(await screen.findByText("Exame negado. Quem pediu vê o aviso no início do painel.")).not.toBeNull();
  });

  it("outro regulador decidiu antes: fecha a decisão, avisa e relê a fila", async () => {
    mocked(api.authorizeExam).mockRejectedValue(new ApiError(409, { error: "invalid_transition", status: "awaiting_schedule" }, "x"));
    renderWithProviders(<ExamRegulation />);
    const panel = await decide();
    fireEvent.change(await within(panel).findByLabelText("Prestador"), { target: { value: "pv1" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Autorizar" }));
    expect((await screen.findByText("a situação do exame mudou — atualize para ver a atual")).getAttribute("role")).toBe("alert");
    expect(screen.queryByRole("region", { name: "Decidir: Hemograma completo de Joana Lima" })).toBeNull();
    await waitFor(() => expect(api.listRegulationQueue).toHaveBeenCalledTimes(2));
  });

  it("nenhum prestador faz o exame: diz onde cadastrar", async () => {
    mocked(api.listEnabledProviders).mockResolvedValue([]);
    renderWithProviders(<ExamRegulation />);
    const panel = await decide();
    expect((await within(panel).findByRole("alert")).textContent).toBe("nenhum prestador ativo faz este exame — cadastre em Catálogo e prestadores");
  });

  it("profissional não regula", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "health_professional" ]));
    renderWithProviders(<ExamRegulation />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.listRegulationQueue).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/ExamRegulation.test.tsx`
Expected: FAIL — `./ExamRegulation` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/ExamRegulation.tsx
// Fila de regulação (módulo 21a; spec §5 etapa 2; contrato §4): exames que
// exigem autorização, na ordem do api (urgente primeiro, depois o mais
// antigo). Autorizar exige o prestador; negar exige o motivo e fecha o exame
// (quem pediu vê no Início). Se outro regulador decidiu antes, 409
// invalid_transition fecha a decisão e relê a fila.
import { useState, type ReactNode } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { authorizeExam, denyExam, errorCode, listEnabledProviders, listRegulationQueue, type ExamOrder } from "../lib/api";
import { useAuth } from "../lib/auth";
import {
  EXAMS_DISABLED, EXAMS_NO_ROLE, EXAM_PROVIDER_REFS_KEY, EXAM_REGULATION_KEY, PRIORITY_LABEL, canUseExams, examError, examRoles,
  overdueLabel, refreshExams
} from "../lib/exams";
import { fmtDateTime } from "../lib/format";
import { useExamOrderPages } from "../hooks/useExamOrderPages";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { KeyValue } from "../components/KeyValue";
import { ActionButton } from "../components/ActionButton";
import { Tag } from "../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { ReasonForm } from "./exams/ReasonForm";
import { alert, box, fieldLabel, grid, muted, okText, row, stack } from "./exams/styles";

export function ExamRegulation() {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const roles = examRoles(user);
  const allowed = canUseExams(user) && (roles.regulator || roles.admin);
  const { query, orders } = useExamOrderPages([ EXAM_REGULATION_KEY ], (before) => listRegulationQueue(before), allowed);
  const [ deciding, setDeciding ] = useState<ExamOrder | null>(null);
  const [ notice, setNotice ] = useState<{ kind: "ok" | "error"; text: string } | null>(null);

  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  if (!allowed) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;

  function finish(next: { kind: "ok" | "error"; text: string }) {
    setDeciding(null);
    setNotice(next);
    void refreshExams(queryClient);
  }

  return (
    <Frame>
      {notice && <p role={notice.kind === "ok" ? "status" : "alert"} style={notice.kind === "ok" ? okText : alert}>{notice.text}</p>}
      {deciding && (
        <Decision key={deciding.id} order={deciding} onCancel={() => setDeciding(null)}
          onDone={(text) => finish({ kind: "ok", text })} onStale={(text) => finish({ kind: "error", text })} />
      )}
      <Panel title="Fila de regulação" sub="urgentes primeiro, depois os mais antigos">
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          {query.isPending && <p className="mono" style={muted}>carregando…</p>}
          {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
          {query.isSuccess && (
            <DataTable<ExamOrder>
              cols={[
                { label: "Prioridade", w: "0.8fr", render: (o) => <Tag tone={o.priority === "urgent" ? "urgent" : undefined}>{PRIORITY_LABEL[o.priority] ?? o.priority}</Tag> },
                { label: "Exame", w: "1.6fr", card: "title", render: (o) => o.label },
                { label: "Paciente", w: "1.3fr", render: (o) => o.patient.display_name },
                { label: "Solicitante", w: "1.2fr", render: (o) => o.requester.name },
                { label: "Unidade", w: "1.1fr", render: (o) => o.unit.name },
                { label: "Desde", w: "1fr", render: (o) => fmtDateTime(o.status_changed_at) },
                { label: "Prazo", w: "1.1fr", render: (o) => overdueLabel(o.overdue) },
                { label: "", w: "auto", align: "right", card: "actions", render: (o) => (
                  <button type="button" aria-label={`Decidir ${o.label} de ${o.patient.display_name}`} style={secondaryButtonStyle}
                    onClick={() => { setNotice(null); setDeciding(o); }}>Decidir</button>
                ) }
              ]}
              rows={orders} rowKey={(o) => o.id} empty="nenhum exame aguardando autorização"
            />
          )}
          {query.hasNextPage && (
            <div><button type="button" style={secondaryButtonStyle} onClick={() => void query.fetchNextPage()}>Carregar mais</button></div>
          )}
        </div>
      </Panel>
    </Frame>
  );
}

function Decision({ order, onDone, onStale, onCancel }: {
  order: ExamOrder; onDone(text: string): void; onStale(text: string): void; onCancel(): void;
}) {
  const providers = useQuery({
    queryKey: [ EXAM_PROVIDER_REFS_KEY, order.sigtap_code ], queryFn: () => listEnabledProviders(order.sigtap_code), retry: false
  });
  const [ providerId, setProviderId ] = useState("");
  const [ denying, setDenying ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);

  async function authorize() {
    if (!providerId) { setError("escolha o prestador"); return; }
    setError(null);
    try {
      await authorizeExam(order.id, providerId);
    } catch (err) {
      if (errorCode(err) === "invalid_transition") { onStale(examError(err)); return; }
      setError(examError(err));
      return;
    }
    const name = providers.data?.find((p) => p.id === providerId)?.name ?? "o prestador";
    onDone(`Exame autorizado para ${name}.`);
  }

  async function deny(reason: string) {
    try {
      await denyExam(order.id, reason);
    } catch (err) {
      if (errorCode(err) === "invalid_transition") { onStale(examError(err)); return; }
      throw err;
    }
    onDone("Exame negado. Quem pediu vê o aviso no início do painel.");
  }

  return (
    <section aria-label={`Decidir: ${order.label} de ${order.patient.display_name}`} style={box}>
      <strong>{`${order.label} · ${order.sigtap_code}`}</strong>
      <div style={grid}>
        <KeyValue k="Paciente" v={order.patient.display_name} mono={false} />
        <KeyValue k="Prioridade" v={PRIORITY_LABEL[order.priority] ?? order.priority} mono={false} />
        <KeyValue k="CID-10" v={order.cid10_justification ?? "—"} />
        <KeyValue k="Solicitante" v={order.requester.name} mono={false} />
        <KeyValue k="Unidade" v={order.unit.name} mono={false} />
      </div>
      {!denying && (
        <>
          {providers.isPending && <p className="mono" style={muted}>carregando prestadores…</p>}
          {providers.isError && <p role="alert" style={alert}>{examError(providers.error)}</p>}
          {providers.isSuccess && providers.data.length === 0 && (
            <p role="alert" style={alert}>nenhum prestador ativo faz este exame — cadastre em Catálogo e prestadores</p>
          )}
          {providers.isSuccess && providers.data.length > 0 && (
            <label style={fieldLabel}>
              Prestador
              <select value={providerId} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
                <option value="">escolha…</option>
                {providers.data.map((p) => <option key={p.id} value={p.id}>{p.name}</option>)}
              </select>
            </label>
          )}
          {error && <p role="alert" style={alert}>{error}</p>}
          <div style={row}>
            <ActionButton onClick={authorize}>Autorizar</ActionButton>
            <ActionButton variant="danger" onClick={() => setDenying(true)}>Negar</ActionButton>
            <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
          </div>
        </>
      )}
      {denying && <ReasonForm label="Motivo da negativa" submitLabel="Negar exame" onCancel={() => setDenying(false)} onSubmit={deny} />}
    </section>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Fila de regulação" sub="exames que exigem autorização" />
      {children}
    </div>
  );
}
```

Em `src/App.tsx`: `import { ExamRegulation } from "./modules/ExamRegulation";` e `case "exam-regulation": return <ExamRegulation />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/ExamRegulation.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/ExamRegulation.tsx src/modules/ExamRegulation.test.tsx src/App.tsx
$G commit -m "feat: add the exam regulation queue with authorize and deny

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 12: Admin — catálogo de exames

**Files:**
- Create: `src/lib/examsAdmin.ts`, `src/modules/ExamsAdmin.tsx`, `src/modules/examsAdmin/CatalogPanel.tsx`, `src/modules/examsAdmin/CatalogItemForm.tsx`
- Modify: `src/App.tsx` (`renderModule`), `src/App.navigation.test.tsx` (lista do `onGoToSecurity`)
- Test: `src/lib/examsAdmin.test.ts`, `src/modules/ExamsAdmin.catalog.test.tsx`

**Interfaces:**
- Consumes: `listCatalogItems`, `createCatalogItem`, `updateCatalogItem`, `errorCode`, `ExamCatalogItem`, `ExamCatalogItemInput`, `ExamReleasePolicy`, `ExamResultField` (Task 1); de `lib/exams`: `DeadlineDraft`, `deadlineDraft`, `deadlinesFrom`, `deadlinesProblem`, `deadlinesSummary`, `resultFieldsProblem`, `sigtapProblem`, `DEFAULT_DEADLINES`, `EXAM_STAGES`, `STAGE_LABEL`, `RELEASE_POLICIES`, `releasePolicyLabel`, `EXAM_ADMIN_CATALOG_KEY`, `EXAM_CATALOG_KEY`, `EXAMS_DISABLED`, `EXAMS_NO_ROLE`, `canUseExams`, `examRoles`, `examError` (Task 2); `SensitiveAction`.
- Produces (em `src/lib/examsAdmin.ts`): `interface CatalogDraft { sigtapCode: string; requiresAuthorization: boolean; releasePolicy: ExamReleasePolicy; deadlines: DeadlineDraft; fields: ExamResultField[]; active: boolean }`, `catalogDraft(item?: ExamCatalogItem | null): CatalogDraft`, `catalogProblem(d: CatalogDraft): string | null`, `catalogBody(d: CatalogDraft): ExamCatalogItemInput`, `EMPTY_FIELD: ExamResultField`. Componentes: `ExamsAdmin(props: { onGoToSecurity?(): void })` (tela `exams-admin`), `CatalogPanel(props: { onGoToSecurity?(): void })` (`Panel` "Catálogo de exames"), `CatalogItemForm(props: { initial?: ExamCatalogItem; onReview(input: ExamCatalogItemInput): void; onCancel(): void })` (`<form aria-label="Novo exame" | "Editar <label>">`).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/examsAdmin.test.ts
import { describe, expect, it } from "vitest";
import { catalogBody, catalogDraft, catalogProblem } from "./examsAdmin";
import { examCatalogItem } from "../test/examFixtures";

describe("catálogo (contrato §5)", () => {
  it("rascunho vazio: sem autorização, depois da ciência, prazos herdados, ativo", () => {
    expect(catalogDraft()).toEqual({
      sigtapCode: "", requiresAuthorization: false, releasePolicy: "after_acknowledgement",
      deadlines: { authorize: "", schedule: "", collect: "", result: "", acknowledge: "" }, fields: [], active: true
    });
  });

  it("valida SIGTAP, prazos e campos, nessa ordem", () => {
    const d = catalogDraft(examCatalogItem());
    expect(catalogProblem(d)).toBeNull();
    expect(catalogProblem({ ...d, sigtapCode: "123" })).toBe("o código SIGTAP tem 10 dígitos");
    expect(catalogProblem({ ...d, deadlines: { ...d.deadlines, schedule: "400" } }))
      .toBe("prazo de agendamento: use dias inteiros de 1 a 365, ou deixe em branco");
    expect(catalogProblem({ ...d, fields: [ { key: "", label: "x", unit: null, loinc: null } ] }))
      .toBe("campo 1: a chave usa letras minúsculas, números e _ (ex.: hemoglobina)");
  });

  it("corpo: prazos em número ou null; unidade e LOINC vazios viram null", () => {
    const d = { ...catalogDraft(examCatalogItem()), fields: [ { key: " hba1c ", label: " HbA1c ", unit: "", loinc: " " } ] };
    expect(catalogBody(d)).toEqual({
      sigtap_code: "0202020380", requires_authorization: false, release_policy: "after_acknowledgement",
      deadlines: { authorize: null, schedule: null, collect: null, result: 10, acknowledge: null },
      result_fields: [ { key: "hba1c", label: "HbA1c", unit: null, loinc: null } ], active: true
    });
  });
});
```

```tsx
// src/modules/ExamsAdmin.catalog.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listCatalogItems: vi.fn(), createCatalogItem: vi.fn(), updateCatalogItem: vi.fn(),
    listExamProviders: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { ExamsAdmin } from "./ExamsAdmin";
import { renderWithProviders } from "../test/campaignFixtures";
import { HEMOGRAM_FIELDS, examCatalogItem, examUser } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

async function catalogPanel() { return screen.findByRole("region", { name: "Catálogo de exames" }); }

describe("ExamsAdmin — catálogo", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listCatalogItems, api.createCatalogItem, api.updateCatalogItem, api.listExamProviders ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "municipal_admin" ], { id: "ad1" }));
    mocked(api.listCatalogItems).mockResolvedValue([ examCatalogItem() ]);
    mocked(api.createCatalogItem).mockResolvedValue(examCatalogItem({ id: "ci2" }));
    mocked(api.updateCatalogItem).mockResolvedValue(examCatalogItem());
    mocked(api.listExamProviders).mockResolvedValue([]);
  });

  it("lista com política, prazos efetivos, campos e situação", async () => {
    renderWithProviders(<ExamsAdmin />);
    const panel = await catalogPanel();
    const row = (await within(panel).findAllByRole("row"))[1];
    expect(within(row).getByText("0202020380")).not.toBeNull();
    expect(within(row).getByText("Depois da ciência")).not.toBeNull();
    expect(within(row).getByText("15 · 15 · 30 · 10 · 15")).not.toBeNull();
    expect(within(row).getByText("2")).not.toBeNull();
    expect(within(row).getByText("ativo")).not.toBeNull();
  });

  it("inclui um exame: valida, pede step-up e manda o corpo do contrato", async () => {
    renderWithProviders(<ExamsAdmin />);
    const panel = await catalogPanel();
    fireEvent.click(await within(panel).findByRole("button", { name: "Novo exame" }));
    const form = within(panel).getByRole("form", { name: "Novo exame" });
    fireEvent.change(within(form).getByLabelText("Código SIGTAP"), { target: { value: "02020" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("o código SIGTAP tem 10 dígitos");
    fireEvent.change(within(form).getByLabelText("Código SIGTAP"), { target: { value: "0202010503" } });
    fireEvent.click(within(form).getByLabelText("Exige autorização (entra na fila de regulação)"));
    fireEvent.click(within(form).getByLabelText(/^Imediata/));
    fireEvent.change(within(form).getByLabelText("Prazo de resultado"), { target: { value: "7" } });
    fireEvent.click(within(form).getByRole("button", { name: "Adicionar campo" }));
    const field = within(form).getByRole("group", { name: "Campo 1" });
    fireEvent.change(within(field).getByLabelText("Chave"), { target: { value: "hba1c" } });
    fireEvent.change(within(field).getByLabelText("Nome"), { target: { value: "HbA1c" } });
    fireEvent.change(within(field).getByLabelText("Unidade"), { target: { value: "%" } });
    fireEvent.change(within(field).getByLabelText("LOINC"), { target: { value: "4548-4" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    const confirm = screen.getByRole("region", { name: "Incluir o exame 0202010503" });
    fireEvent.click(within(confirm).getByRole("button", { name: "Salvar exame" }));
    await waitFor(() => expect(api.createCatalogItem).toHaveBeenCalledWith({
      sigtap_code: "0202010503", requires_authorization: true, release_policy: "immediate",
      deadlines: { authorize: null, schedule: null, collect: null, result: 7, acknowledge: null },
      result_fields: [ { key: "hba1c", label: "HbA1c", unit: "%", loinc: "4548-4" } ], active: true
    }));
    expect(await within(panel).findByText("Catálogo atualizado.")).not.toBeNull();
    await waitFor(() => expect(api.listCatalogItems).toHaveBeenCalledTimes(2));
  });

  it("SIGTAP fora da competência: a frase na confirmação e o rascunho volta intacto", async () => {
    mocked(api.createCatalogItem).mockRejectedValue(new ApiError(422, { error: "sigtap_unknown" }, "x"));
    renderWithProviders(<ExamsAdmin />);
    const panel = await catalogPanel();
    fireEvent.click(await within(panel).findByRole("button", { name: "Novo exame" }));
    const form = within(panel).getByRole("form", { name: "Novo exame" });
    fireEvent.change(within(form).getByLabelText("Código SIGTAP"), { target: { value: "0999999999" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    const confirm = screen.getByRole("region", { name: "Incluir o exame 0999999999" });
    fireEvent.click(within(confirm).getByRole("button", { name: "Salvar exame" }));
    expect((await within(confirm).findByRole("alert")).textContent).toBe("código SIGTAP não encontrado na competência vigente");
    fireEvent.click(within(confirm).getByRole("button", { name: "Cancelar" }));
    expect((within(panel).getByLabelText("Código SIGTAP") as HTMLInputElement).value).toBe("0999999999");
  });

  it("edita: SIGTAP travado; PATCH com o prazo novo e os campos de antes", async () => {
    renderWithProviders(<ExamsAdmin />);
    const panel = await catalogPanel();
    fireEvent.click(await within(panel).findByRole("button", { name: "Editar Hemograma completo" }));
    const form = within(panel).getByRole("form", { name: "Editar Hemograma completo" });
    expect((within(form).getByLabelText("Código SIGTAP") as HTMLInputElement).disabled).toBe(true);
    fireEvent.change(within(form).getByLabelText("Prazo de coleta"), { target: { value: "20" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Salvar Hemograma completo" })).getByRole("button", { name: "Salvar exame" }));
    await waitFor(() => expect(api.updateCatalogItem).toHaveBeenCalledWith("ci1", {
      sigtap_code: "0202020380", requires_authorization: false, release_policy: "after_acknowledgement",
      deadlines: { authorize: null, schedule: null, collect: 20, result: 10, acknowledge: null },
      result_fields: HEMOGRAM_FIELDS, active: true
    }));
  });

  it("só o admin municipal", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "exam_regulator" ]));
    renderWithProviders(<ExamsAdmin />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.listCatalogItems).not.toHaveBeenCalled();
  });
});
```

Em `src/App.navigation.test.tsx`, acrescente `"exams-admin"` à lista do `it.each([ "integrations", … ])("%s.onGoToSecurity abre Segurança", …)`.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/examsAdmin.test.ts src/modules/ExamsAdmin.catalog.test.tsx src/App.navigation.test.tsx`
Expected: FAIL — os módulos não existem e `renderModule("exams-admin")` cai no `Placeholder`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/examsAdmin.ts
// Formulários do admin municipal de exames (módulo 21a; contrato §5): o
// catálogo da cidade e (Task 13) os prestadores. Prazo em branco = herda.
import type { ExamCatalogItem, ExamCatalogItemInput, ExamReleasePolicy, ExamResultField } from "./api";
import { deadlineDraft, deadlinesFrom, deadlinesProblem, resultFieldsProblem, sigtapProblem, type DeadlineDraft } from "./exams";

export interface CatalogDraft {
  sigtapCode: string; requiresAuthorization: boolean; releasePolicy: ExamReleasePolicy;
  deadlines: DeadlineDraft; fields: ExamResultField[]; active: boolean;
}

export const EMPTY_FIELD: ExamResultField = { key: "", label: "", unit: null, loinc: null };

export function catalogDraft(item?: ExamCatalogItem | null): CatalogDraft {
  return {
    sigtapCode: item?.sigtap_code ?? "",
    requiresAuthorization: item?.requires_authorization ?? false,
    releasePolicy: item?.release_policy ?? "after_acknowledgement",
    deadlines: deadlineDraft(item?.deadlines),
    fields: (item?.result_fields ?? []).map((f) => ({ ...f })),
    active: item?.active ?? true
  };
}

export function catalogProblem(d: CatalogDraft): string | null {
  return sigtapProblem(d.sigtapCode) ?? deadlinesProblem(d.deadlines) ?? resultFieldsProblem(d.fields);
}

export function catalogBody(d: CatalogDraft): ExamCatalogItemInput {
  return {
    sigtap_code: d.sigtapCode.trim(),
    requires_authorization: d.requiresAuthorization,
    release_policy: d.releasePolicy,
    deadlines: deadlinesFrom(d.deadlines),
    result_fields: d.fields.map((f) => ({
      key: f.key.trim(), label: f.label.trim(), unit: f.unit?.trim() || null, loinc: f.loinc?.trim() || null
    })),
    active: d.active
  };
}
```

```tsx
// src/modules/examsAdmin/CatalogItemForm.tsx
// Um exame do catálogo da cidade (contrato §5): SIGTAP (travado na edição),
// autorização, política de liberação ao cidadão, prazos por etapa (em branco
// = padrão da plataforma, mostrado no placeholder) e os campos de resultado.
import { useState, type FormEvent } from "react";
import type { ExamCatalogItem, ExamCatalogItemInput, ExamResultField } from "../../lib/api";
import { DEFAULT_DEADLINES, EXAM_STAGES, RELEASE_POLICIES, STAGE_LABEL } from "../../lib/exams";
import { EMPTY_FIELD, catalogBody, catalogDraft, catalogProblem, type CatalogDraft } from "../../lib/examsAdmin";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, checkLabel, fieldLabel, fieldset, form, legend, muted, row } from "../exams/styles";

export function CatalogItemForm({ initial, onReview, onCancel }: {
  initial?: ExamCatalogItem; onReview(input: ExamCatalogItemInput): void; onCancel(): void;
}) {
  const [ draft, setDraft ] = useState<CatalogDraft>(() => catalogDraft(initial));
  const [ error, setError ] = useState<string | null>(null);
  const set = (patch: Partial<CatalogDraft>) => setDraft((d) => ({ ...d, ...patch }));
  const setField = (i: number, patch: Partial<ExamResultField>) =>
    setDraft((d) => ({ ...d, fields: d.fields.map((f, j) => (j === i ? { ...f, ...patch } : f)) }));

  function submit(event: FormEvent) {
    event.preventDefault();
    const problem = catalogProblem(draft);
    if (problem) { setError(problem); return; }
    setError(null);
    onReview(catalogBody(draft));
  }

  return (
    <form aria-label={initial ? `Editar ${initial.label}` : "Novo exame"} onSubmit={submit} style={form}>
      <label style={fieldLabel}>
        Código SIGTAP
        <input value={draft.sigtapCode} inputMode="numeric" disabled={!!initial} style={inputStyle}
          onChange={(e) => set({ sigtapCode: e.target.value })} />
      </label>
      <label style={checkLabel}>
        <input type="checkbox" checked={draft.requiresAuthorization} onChange={(e) => set({ requiresAuthorization: e.target.checked })} />
        Exige autorização (entra na fila de regulação)
      </label>
      <fieldset style={fieldset}>
        <legend style={legend}>Liberação do laudo ao cidadão</legend>
        {RELEASE_POLICIES.map((p) => (
          <label key={p.value} style={checkLabel}>
            <input type="radio" name="release-policy" checked={draft.releasePolicy === p.value} onChange={() => set({ releasePolicy: p.value })} />
            {`${p.label} — ${p.hint}`}
          </label>
        ))}
      </fieldset>
      <fieldset style={fieldset}>
        <legend style={legend}>Prazos em dias (em branco = padrão da plataforma; urgente usa a metade)</legend>
        <div style={row}>
          {EXAM_STAGES.map((s) => (
            <label key={s} style={fieldLabel}>
              {`Prazo de ${STAGE_LABEL[s]}`}
              <input value={draft.deadlines[s]} inputMode="numeric" placeholder={`padrão ${DEFAULT_DEADLINES[s]}`} style={{ ...inputStyle, width: 120 }}
                onChange={(e) => { const value = e.target.value; setDraft((d) => ({ ...d, deadlines: { ...d.deadlines, [s]: value } })); }} />
            </label>
          ))}
        </div>
      </fieldset>
      <fieldset style={fieldset}>
        <legend style={legend}>Campos de resultado</legend>
        {draft.fields.length === 0 && <p style={muted}>sem campos — o resultado será só o laudo</p>}
        {draft.fields.map((f, i) => (
          <div key={i} role="group" aria-label={`Campo ${i + 1}`} style={{ ...row, alignItems: "flex-end" }}>
            <label style={fieldLabel}>Chave<input value={f.key} style={inputStyle} onChange={(e) => setField(i, { key: e.target.value })} /></label>
            <label style={fieldLabel}>Nome<input value={f.label} style={inputStyle} onChange={(e) => setField(i, { label: e.target.value })} /></label>
            <label style={fieldLabel}>Unidade<input value={f.unit ?? ""} style={inputStyle} onChange={(e) => setField(i, { unit: e.target.value })} /></label>
            <label style={fieldLabel}>LOINC<input value={f.loinc ?? ""} style={inputStyle} onChange={(e) => setField(i, { loinc: e.target.value })} /></label>
            <button type="button" aria-label={`Remover campo ${i + 1}`} style={secondaryButtonStyle}
              onClick={() => setDraft((d) => ({ ...d, fields: d.fields.filter((_, j) => j !== i) }))}>Remover</button>
          </div>
        ))}
        <div>
          <button type="button" style={secondaryButtonStyle} onClick={() => setDraft((d) => ({ ...d, fields: [ ...d.fields, { ...EMPTY_FIELD } ] }))}>
            Adicionar campo
          </button>
        </div>
      </fieldset>
      <label style={checkLabel}>
        <input type="checkbox" checked={draft.active} onChange={(e) => set({ active: e.target.checked })} />
        Ativo
      </label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <button type="submit" style={buttonStyle}>Revisar e salvar</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/examsAdmin/CatalogPanel.tsx
// Catálogo de exames da cidade (spec §4; contrato §5). Toda escrita com
// step-up. O formulário fica montado (escondido) durante a confirmação: uma
// recusa do api (sigtap_unknown, duplicate_sigtap) volta ao rascunho intacto.
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { createCatalogItem, errorCode, listCatalogItems, updateCatalogItem, type ExamCatalogItem, type ExamCatalogItemInput } from "../../lib/api";
import {
  EXAM_ADMIN_CATALOG_KEY, EXAM_CATALOG_KEY, deadlinesSummary, examError, releasePolicyLabel
} from "../../lib/exams";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { SensitiveAction } from "../../components/SensitiveAction";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { CatalogItemForm } from "./CatalogItemForm";
import { alert, muted, okText } from "../exams/styles";

type Editing = { item: ExamCatalogItem | null };
type Confirming = { item: ExamCatalogItem | null; input: ExamCatalogItemInput };

export function CatalogPanel({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ EXAM_ADMIN_CATALOG_KEY ], queryFn: listCatalogItems, retry: false });
  const [ editing, setEditing ] = useState<Editing | null>(null);
  const [ confirming, setConfirming ] = useState<Confirming | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  function review(input: ExamCatalogItemInput) {
    setConfirming({ item: editing?.item ?? null, input });
  }

  function saved() {
    setNotice("Catálogo atualizado.");
    setConfirming(null);
    setEditing(null);
    void queryClient.invalidateQueries({ queryKey: [ EXAM_ADMIN_CATALOG_KEY ] });
    void queryClient.invalidateQueries({ queryKey: [ EXAM_CATALOG_KEY ] });
  }

  return (
    <Panel title="Catálogo de exames" sub="o que a cidade acompanha · prazos em dias: autorização · agendamento · coleta · resultado · ciência">
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
        {query.isSuccess && (
          <DataTable<ExamCatalogItem>
            cols={[
              { label: "SIGTAP", w: "1fr", render: (i) => <span className="mono">{i.sigtap_code}</span> },
              { label: "Exame", w: "1.8fr", card: "title", render: (i) => i.label },
              { label: "Autorização", w: "0.9fr", render: (i) => (i.requires_authorization ? "exige" : "não") },
              { label: "Liberação", w: "1.1fr", render: (i) => releasePolicyLabel(i.release_policy) },
              { label: "Prazos", w: "1.3fr", render: (i) => <span className="mono">{deadlinesSummary(i.deadlines)}</span> },
              { label: "Campos", w: "0.6fr", align: "right", render: (i) => String(i.result_fields.length) },
              { label: "Situação", w: "0.8fr", render: (i) => <Tag tone={i.active ? "ok" : "neutral"}>{i.active ? "ativo" : "inativo"}</Tag> },
              { label: "", w: "auto", align: "right", card: "actions", render: (i) => (
                <button type="button" aria-label={`Editar ${i.label}`} style={secondaryButtonStyle} disabled={editing !== null}
                  onClick={() => { setNotice(null); setEditing({ item: i }); }}>Editar</button>
              ) }
            ]}
            rows={query.data} rowKey={(i) => i.id} empty="nenhum exame no catálogo"
          />
        )}
        {query.isSuccess && editing === null && (
          <div><button type="button" style={buttonStyle} onClick={() => { setNotice(null); setEditing({ item: null }); }}>Novo exame</button></div>
        )}
        {editing && (
          <div hidden={confirming !== null}>
            <CatalogItemForm key={editing.item?.id ?? "new"} initial={editing.item ?? undefined} onReview={review} onCancel={() => setEditing(null)} />
          </div>
        )}
        {confirming && (
          <SensitiveAction
            title={confirming.item ? `Salvar ${confirming.item.label}` : `Incluir o exame ${confirming.input.sigtap_code}`}
            description={`${confirming.input.requires_authorization ? "Exige autorização" : "Sem autorização"} · liberação: ${releasePolicyLabel(confirming.input.release_policy)} · ${confirming.input.result_fields.length} campos de resultado.`}
            requiresStepUp
            confirmLabel="Salvar exame"
            run={async () => {
              if (confirming.item) await updateCatalogItem(confirming.item.id, confirming.input);
              else await createCatalogItem(confirming.input);
            }}
            onDone={saved}
            onCancel={() => setConfirming(null)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? examError(err) : null)}
          />
        )}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/ExamsAdmin.tsx
// Catálogo de exames e prestadores da cidade (módulo 21a; spec §9 "Admin
// municipal"; contrato §5). Só municipal_admin, com exam_tracking.
import type { ReactNode } from "react";
import { useAuth } from "../lib/auth";
import { EXAMS_DISABLED, EXAMS_NO_ROLE, canUseExams, examRoles } from "../lib/exams";
import { PageHeader } from "../components/PageHeader";
import { EmptyState } from "../components/EmptyState";
import { CatalogPanel } from "./examsAdmin/CatalogPanel";
import { stack } from "./exams/styles";

export function ExamsAdmin({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const { user } = useAuth();
  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  if (!examRoles(user).admin) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;
  return (
    <Frame>
      <CatalogPanel onGoToSecurity={onGoToSecurity} />
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Catálogo e prestadores" sub="cidade · exames, prazos, prestadores e acessos" />
      {children}
    </div>
  );
}
```

Em `src/App.tsx`: `import { ExamsAdmin } from "./modules/ExamsAdmin";` e `case "exams-admin": return <ExamsAdmin onGoToSecurity={() => setActive("security")} />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/examsAdmin.test.ts src/modules/ExamsAdmin.catalog.test.tsx src/App.navigation.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/examsAdmin.ts src/lib/examsAdmin.test.ts src/modules/ExamsAdmin.tsx src/modules/examsAdmin/CatalogPanel.tsx \
  src/modules/examsAdmin/CatalogItemForm.tsx src/modules/ExamsAdmin.catalog.test.tsx src/App.tsx src/App.navigation.test.tsx
$G commit -m "feat: add the city exam catalog admin

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Admin — prestadores, token FHIR e convite; papel de regulador no convite

**Files:**
- Modify: `src/lib/examsAdmin.ts`, `src/lib/examsAdmin.test.ts`, `src/modules/ExamsAdmin.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Create: `src/modules/examsAdmin/ProvidersPanel.tsx`, `src/modules/examsAdmin/ProviderForm.tsx`, `src/modules/examsAdmin/ProviderAccess.tsx`
- Test: `src/modules/ExamsAdmin.providers.test.tsx`

**Interfaces:**
- Consumes: `listExamProviders`, `createExamProvider`, `updateExamProvider`, `rotateFhirToken`, `revokeFhirToken`, `inviteProviderUser`, `listCatalogItems`, `errorCode`, `ExamProvider`, `ExamProviderInput`, `ExamProviderKind`, `ExamCatalogItem` (Task 1); de `lib/exams`: `DeadlineDraft`, `deadlineDraft`, `deadlinesFrom`, `deadlinesProblem`, `cnesProblem`, `effectiveDeadline`, `EXAM_STAGES`, `STAGE_LABEL`, `PROVIDER_KINDS`, `providerKindLabel`, `EXAM_ADMIN_PROVIDERS_KEY`, `EXAM_ADMIN_CATALOG_KEY`, `EXAM_PROVIDER_REFS_KEY`, `examError` (Task 2); `isValidEmail`, `inviteErrorMessage` (`lib/team`); `SensitiveAction`.
- Produces (em `src/lib/examsAdmin.ts`): `interface ProviderDraft { name: string; cnes: string; kind: ExamProviderKind; active: boolean; items: Record<string, DeadlineDraft> }` (chave = `catalog_item_id`; presente = habilitado), `providerDraft(p?: ExamProvider | null): ProviderDraft`, `providerProblem(d: ProviderDraft, labelOf: (catalogItemId: string) => string): string | null`, `providerBody(d: ProviderDraft): ExamProviderInput`. Componentes: `ProvidersPanel(props: { onGoToSecurity?(): void })` (`Panel` "Prestadores"), `ProviderForm(props: { initial?: ExamProvider; catalog: ExamCatalogItem[]; onReview(input: ExamProviderInput): void; onCancel(): void })`, `ProviderAccess(props: { provider: ExamProvider; onGoToSecurity?(): void; onChanged(): void; onClose(): void })` (`<section aria-label={`Acesso de ${name}`}>`). `INVITE_ROLES` ganha `exam_regulator` ("Regulador de exames"); o `exam_provider` só entra pelo convite do prestador.

- [ ] **Step 1: Write the failing test**

Acrescente a `src/lib/examsAdmin.test.ts`:

```ts
import { providerBody, providerDraft, providerProblem } from "./examsAdmin";
import { examProvider } from "../test/examFixtures";

describe("prestador (contrato §5)", () => {
  const labelOf = (id: string) => (id === "ci1" ? "Hemograma completo" : id);

  it("rascunho do prestador existente e corpo de volta", () => {
    const d = providerDraft(examProvider());
    expect(d.items).toEqual({ ci1: { authorize: "", schedule: "", collect: "", result: "5", acknowledge: "" } });
    expect(providerBody(d)).toEqual({
      name: "Laboratório Simulado", cnes: "1234567", kind: "accredited", active: true,
      items: [ { catalog_item_id: "ci1", deadlines: { authorize: null, schedule: null, collect: null, result: 5, acknowledge: null } } ]
    });
  });

  it("nome, CNES e prazos por exame", () => {
    const d = providerDraft();
    expect(providerProblem(d, labelOf)).toBe("informe o nome do prestador");
    expect(providerProblem({ ...d, name: "Lab", cnes: "12" }, labelOf)).toBe("o CNES tem 7 dígitos");
    const bad = { ...d, name: "Lab", items: { ci1: { authorize: "", schedule: "", collect: "x", result: "", acknowledge: "" } } };
    expect(providerProblem(bad, labelOf)).toBe("Hemograma completo: prazo de coleta: use dias inteiros de 1 a 365, ou deixe em branco");
    expect(providerBody({ ...d, name: " Lab ", cnes: " " }).cnes).toBeNull();
  });
});
```

Em `src/lib/team.test.ts`, a lista ordenada esperada de `INVITE_ROLES` passa a ser:

```ts
    expect(INVITE_ROLES.map((r) => r.role).sort()).toEqual([
      "analyst", "campaign_manager", "citizen_verifier", "exam_regulator", "health_professional", "municipal_admin",
      "protocol_author", "protocol_publisher", "protocol_reviewer", "viewer"
    ]);
    expect(INVITE_ROLES.find((r) => r.role === "exam_regulator")?.label).toBe("Regulador de exames");
    expect(isPrivilegedRole("exam_regulator")).toBe(true);
```

```tsx
// src/modules/ExamsAdmin.providers.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listCatalogItems: vi.fn(), listExamProviders: vi.fn(), createExamProvider: vi.fn(),
    updateExamProvider: vi.fn(), rotateFhirToken: vi.fn(), revokeFhirToken: vi.fn(), inviteProviderUser: vi.fn() };
});

import * as api from "../lib/api";
import { ExamsAdmin } from "./ExamsAdmin";
import { renderWithProviders } from "../test/campaignFixtures";
import { examCatalogItem, examProvider, examUser } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listCatalogItems, api.listExamProviders, api.createExamProvider, api.updateExamProvider,
  api.rotateFhirToken, api.revokeFhirToken, api.inviteProviderUser ];
afterEach(cleanup);

async function providersPanel() { return screen.findByRole("region", { name: "Prestadores" }); }
async function access(panel: HTMLElement) {
  fireEvent.click(await within(panel).findByRole("button", { name: "Acesso de Laboratório Simulado" }));
  return within(panel).getByRole("region", { name: "Acesso de Laboratório Simulado" });
}

describe("ExamsAdmin — prestadores", () => {
  beforeEach(() => {
    for (const fn of ALL) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "municipal_admin" ], { id: "ad1" }));
    mocked(api.listCatalogItems).mockResolvedValue([ examCatalogItem() ]);
    mocked(api.listExamProviders).mockResolvedValue([ examProvider() ]);
    mocked(api.createExamProvider).mockResolvedValue(examProvider({ id: "pv2" }));
    mocked(api.rotateFhirToken).mockResolvedValue("tok_abc123");
    mocked(api.revokeFhirToken).mockResolvedValue(undefined);
    mocked(api.inviteProviderUser).mockResolvedValue({ id: "iv1", email: "lab@exemplo.com", role: "exam_provider", expires_at: "2026-10-20T10:00:00-03:00" });
  });

  it("lista tipo, CNES, número de exames e o estado do token", async () => {
    renderWithProviders(<ExamsAdmin />);
    const row = (await within(await providersPanel()).findAllByRole("row"))[1];
    expect(within(row).getByText("Credenciado")).not.toBeNull();
    expect(within(row).getByText("1234567")).not.toBeNull();
    expect(within(row).getByText("1")).not.toBeNull();
    expect(within(row).getByText("sem token")).not.toBeNull();
  });

  it("inclui um prestador com o exame e um prazo próprio", async () => {
    renderWithProviders(<ExamsAdmin />);
    const panel = await providersPanel();
    fireEvent.click(await within(panel).findByRole("button", { name: "Novo prestador" }));
    const form = within(panel).getByRole("form", { name: "Novo prestador" });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("informe o nome do prestador");
    fireEvent.change(within(form).getByLabelText("Nome"), { target: { value: "Lab Bairro" } });
    fireEvent.change(within(form).getByLabelText("CNES (opcional)"), { target: { value: "123" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("o CNES tem 7 dígitos");
    fireEvent.change(within(form).getByLabelText("CNES (opcional)"), { target: { value: "" } });
    fireEvent.change(within(form).getByLabelText("Tipo"), { target: { value: "municipal" } });
    fireEvent.click(within(form).getByLabelText("Faz Hemograma completo"));
    const prazos = within(form).getByRole("group", { name: "Prazos — Hemograma completo" });
    expect((within(prazos).getByLabelText("Prazo de resultado") as HTMLInputElement).placeholder).toBe("herda 10");
    fireEvent.change(within(prazos).getByLabelText("Prazo de resultado"), { target: { value: "5" } });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Incluir o prestador Lab Bairro" })).getByRole("button", { name: "Salvar prestador" }));
    await waitFor(() => expect(api.createExamProvider).toHaveBeenCalledWith({
      name: "Lab Bairro", cnes: null, kind: "municipal", active: true,
      items: [ { catalog_item_id: "ci1", deadlines: { authorize: null, schedule: null, collect: null, result: 5, acknowledge: null } } ]
    }));
    expect(await within(panel).findByText("Prestador salvo.")).not.toBeNull();
  });

  it("token FHIR: aparece uma vez, some com Já guardei e nunca vai ao cache", async () => {
    const { client } = renderWithProviders(<ExamsAdmin />);
    const box = await access(await providersPanel());
    fireEvent.click(within(box).getByRole("button", { name: "Gerar token FHIR" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Gerar token FHIR — Laboratório Simulado" })).getByRole("button", { name: "Gerar token" }));
    const shown = await screen.findByRole("region", { name: "Token FHIR novo" });
    expect(within(shown).getByText("tok_abc123")).not.toBeNull();
    expect(within(shown).getByText("Copie agora: o token não aparece de novo. Ele substitui o anterior, que deixa de funcionar.")).not.toBeNull();
    expect(api.rotateFhirToken).toHaveBeenCalledWith("pv1");
    expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state.data))).not.toContain("tok_abc123");
    fireEvent.click(within(shown).getByRole("button", { name: "Já guardei" }));
    expect(screen.queryByText("tok_abc123")).toBeNull();
  });

  it("revoga o token que existe", async () => {
    mocked(api.listExamProviders).mockResolvedValue([ examProvider({ has_fhir_token: true }) ]);
    renderWithProviders(<ExamsAdmin />);
    const box = await access(await providersPanel());
    expect(within(box).getByRole("button", { name: "Trocar token FHIR" })).not.toBeNull();
    fireEvent.click(within(box).getByRole("button", { name: "Revogar token FHIR" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Revogar token FHIR — Laboratório Simulado" })).getByRole("button", { name: "Revogar token" }));
    await waitFor(() => expect(api.revokeFhirToken).toHaveBeenCalledWith("pv1"));
    expect(await within(box).findByText("Token revogado.")).not.toBeNull();
  });

  it("convida o usuário do prestador; e-mail inválido não abre a confirmação", async () => {
    renderWithProviders(<ExamsAdmin />);
    const box = await access(await providersPanel());
    fireEvent.change(within(box).getByLabelText("E-mail do usuário do prestador"), { target: { value: "lab@" } });
    fireEvent.click(within(box).getByRole("button", { name: "Convidar" }));
    expect((await within(box).findByRole("alert")).textContent).toBe("e-mail inválido");
    fireEvent.change(within(box).getByLabelText("E-mail do usuário do prestador"), { target: { value: "lab@exemplo.com" } });
    fireEvent.click(within(box).getByRole("button", { name: "Convidar" }));
    fireEvent.click(within(screen.getByRole("region", { name: "Convidar usuário do prestador — Laboratório Simulado" }))
      .getByRole("button", { name: "Enviar convite" }));
    await waitFor(() => expect(api.inviteProviderUser).toHaveBeenCalledWith("pv1", "lab@exemplo.com"));
    expect(await within(box).findByText("Convite enviado para lab@exemplo.com.")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/examsAdmin.test.ts src/lib/team.test.ts src/modules/ExamsAdmin.providers.test.tsx`
Expected: FAIL — funções e painel inexistentes; `exam_regulator` fora do convite.

- [ ] **Step 3: Write minimal implementation**

Acrescente a `src/lib/examsAdmin.ts` (e amplie os imports: `ExamProvider`, `ExamProviderInput`, `ExamProviderKind` de `./api`; `cnesProblem` de `./exams`):

```ts
export interface ProviderDraft {
  name: string; cnes: string; kind: ExamProviderKind; active: boolean;
  // catalog_item_id → prazos próprios; presente = o prestador faz o exame.
  items: Record<string, DeadlineDraft>;
}

export function providerDraft(p?: ExamProvider | null): ProviderDraft {
  return {
    name: p?.name ?? "",
    cnes: p?.cnes ?? "",
    kind: p?.kind ?? "accredited",
    active: p?.active ?? true,
    items: Object.fromEntries((p?.items ?? []).map((i) => [ i.catalog_item_id, deadlineDraft(i.deadlines) ]))
  };
}

export function providerProblem(d: ProviderDraft, labelOf: (catalogItemId: string) => string): string | null {
  if (!d.name.trim()) return "informe o nome do prestador";
  const cnes = cnesProblem(d.cnes);
  if (cnes) return cnes;
  for (const [ id, deadlines ] of Object.entries(d.items)) {
    const problem = deadlinesProblem(deadlines);
    if (problem) return `${labelOf(id)}: ${problem}`;
  }
  return null;
}

export function providerBody(d: ProviderDraft): ExamProviderInput {
  return {
    name: d.name.trim(),
    cnes: d.cnes.trim() || null,
    kind: d.kind,
    active: d.active,
    items: Object.entries(d.items).map(([ catalog_item_id, deadlines ]) => ({ catalog_item_id, deadlines: deadlinesFrom(deadlines) }))
  };
}
```

```tsx
// src/modules/examsAdmin/ProviderForm.tsx
// Um prestador (contrato §5): nome, CNES opcional, tipo, ativo e os exames
// que ele faz, cada um com prazos próprios opcionais (o placeholder mostra o
// que ele herda do catálogo ou da plataforma).
import { useState, type FormEvent } from "react";
import type { ExamCatalogItem, ExamProvider, ExamProviderInput, ExamProviderKind } from "../../lib/api";
import { EXAM_STAGES, PROVIDER_KINDS, STAGE_LABEL, deadlineDraft, effectiveDeadline } from "../../lib/exams";
import { providerBody, providerDraft, providerProblem, type ProviderDraft } from "../../lib/examsAdmin";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, checkLabel, fieldLabel, fieldset, form, legend, muted, row } from "../exams/styles";

export function ProviderForm({ initial, catalog, onReview, onCancel }: {
  initial?: ExamProvider; catalog: ExamCatalogItem[]; onReview(input: ExamProviderInput): void; onCancel(): void;
}) {
  const [ draft, setDraft ] = useState<ProviderDraft>(() => providerDraft(initial));
  const [ error, setError ] = useState<string | null>(null);
  const set = (patch: Partial<ProviderDraft>) => setDraft((d) => ({ ...d, ...patch }));
  // Exame inativo só aparece se o prestador já o tinha (para poder tirá-lo).
  const shown = catalog.filter((c) => c.active || draft.items[c.id]);
  const labelOf = (id: string) => catalog.find((c) => c.id === id)?.label ?? id;

  function toggle(id: string, on: boolean) {
    setDraft((d) => {
      const items = { ...d.items };
      if (on) items[id] = deadlineDraft(null); else delete items[id];
      return { ...d, items };
    });
  }

  function submit(event: FormEvent) {
    event.preventDefault();
    const problem = providerProblem(draft, labelOf);
    if (problem) { setError(problem); return; }
    setError(null);
    onReview(providerBody(draft));
  }

  return (
    <form aria-label={initial ? `Editar ${initial.name}` : "Novo prestador"} onSubmit={submit} style={form}>
      <label style={fieldLabel}>Nome<input value={draft.name} style={inputStyle} onChange={(e) => set({ name: e.target.value })} /></label>
      <label style={fieldLabel}>
        CNES (opcional)
        <input value={draft.cnes} inputMode="numeric" style={inputStyle} onChange={(e) => set({ cnes: e.target.value })} />
      </label>
      <label style={fieldLabel}>
        Tipo
        <select value={draft.kind} style={inputStyle} onChange={(e) => set({ kind: e.target.value as ExamProviderKind })}>
          {PROVIDER_KINDS.map((k) => <option key={k.value} value={k.value}>{k.label}</option>)}
        </select>
      </label>
      <label style={checkLabel}>
        <input type="checkbox" checked={draft.active} onChange={(e) => set({ active: e.target.checked })} />Ativo
      </label>
      <fieldset style={fieldset}>
        <legend style={legend}>Exames que este prestador faz</legend>
        {shown.length === 0 && <p style={muted}>o catálogo da cidade está vazio — inclua os exames primeiro</p>}
        {shown.map((c) => (
          <div key={c.id} style={{ display: "flex", flexDirection: "column", gap: 6 }}>
            <label style={checkLabel}>
              <input type="checkbox" checked={!!draft.items[c.id]} onChange={(e) => toggle(c.id, e.target.checked)} />
              {`Faz ${c.label}`}
            </label>
            {draft.items[c.id] && (
              <div role="group" aria-label={`Prazos — ${c.label}`} style={{ ...row, paddingLeft: 24 }}>
                {EXAM_STAGES.map((s) => (
                  <label key={s} style={fieldLabel}>
                    {`Prazo de ${STAGE_LABEL[s]}`}
                    <input value={draft.items[c.id][s]} inputMode="numeric" placeholder={`herda ${effectiveDeadline(s, null, c.deadlines)}`}
                      style={{ ...inputStyle, width: 110 }}
                      onChange={(e) => {
                        const value = e.target.value;
                        setDraft((d) => ({ ...d, items: { ...d.items, [c.id]: { ...d.items[c.id], [s]: value } } }));
                      }} />
                  </label>
                ))}
              </div>
            )}
          </div>
        ))}
      </fieldset>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <button type="submit" style={buttonStyle}>Revisar e salvar</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Voltar</button>
      </div>
    </form>
  );
}
```

```tsx
// src/modules/examsAdmin/ProviderAccess.tsx
// Acessos do prestador (contrato §5): o token FHIR do laboratório (mostrado
// UMA vez — vive só neste estado, nunca no cache; some com "Já guardei" ou
// ao fechar) e o convite do usuário `exam_provider` (o mecanismo de convite
// existente; o usuário entra só no portal).
import { useState } from "react";
import { errorCode, inviteProviderUser, revokeFhirToken, rotateFhirToken, type ExamProvider } from "../../lib/api";
import { examError } from "../../lib/exams";
import { inviteErrorMessage, isValidEmail } from "../../lib/team";
import { SensitiveAction } from "../../components/SensitiveAction";
import { dangerButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, muted, okText, row } from "../exams/styles";

type Confirming = null | "rotate" | "revoke" | { invite: string };
const translate = (err: unknown) => (errorCode(err) ? examError(err) : null);

export function ProviderAccess({ provider, onGoToSecurity, onChanged, onClose }: {
  provider: ExamProvider; onGoToSecurity?(): void; onChanged(): void; onClose(): void;
}) {
  const [ confirming, setConfirming ] = useState<Confirming>(null);
  const [ token, setToken ] = useState<string | null>(null);
  const [ copyNote, setCopyNote ] = useState<string | null>(null);
  const [ email, setEmail ] = useState("");
  const [ emailError, setEmailError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const hasToken = provider.has_fhir_token === true;
  const idle = confirming === null && token === null;

  async function copy() {
    try {
      await navigator.clipboard.writeText(token ?? "");
      setCopyNote("token copiado");
    } catch {
      setCopyNote("não foi possível copiar — selecione o token e copie");
    }
  }

  function invite() {
    if (!isValidEmail(email)) { setEmailError("e-mail inválido"); return; }
    setEmailError(null);
    setNotice(null);
    setConfirming({ invite: email.trim() });
  }

  return (
    <section aria-label={`Acesso de ${provider.name}`} style={box}>
      <strong>{`Acesso de ${provider.name}`}</strong>
      <p style={muted}>O token FHIR é do sistema do laboratório (resultados automáticos). O usuário do prestador entra só no portal, sem prontuário.</p>
      {notice && <p role="status" style={okText}>{notice}</p>}

      {token && (
        <section aria-label="Token FHIR novo" style={box}>
          <p style={alert}>Copie agora: o token não aparece de novo. Ele substitui o anterior, que deixa de funcionar.</p>
          <code className="mono" style={{ overflowWrap: "anywhere", fontSize: 13 }}>{token}</code>
          <div style={row}>
            <button type="button" style={secondaryButtonStyle} onClick={() => void copy()}>Copiar</button>
            <button type="button" style={secondaryButtonStyle} onClick={() => { setToken(null); setCopyNote(null); }}>Já guardei</button>
          </div>
          {copyNote && <p role="status" style={muted}>{copyNote}</p>}
        </section>
      )}

      {idle && (
        <div style={row}>
          <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setConfirming("rotate"); }}>
            {hasToken ? "Trocar token FHIR" : "Gerar token FHIR"}
          </button>
          {hasToken && (
            <button type="button" style={dangerButtonStyle} onClick={() => { setNotice(null); setConfirming("revoke"); }}>Revogar token FHIR</button>
          )}
        </div>
      )}
      {confirming === "rotate" && (
        <SensitiveAction
          title={`${hasToken ? "Trocar" : "Gerar"} token FHIR — ${provider.name}`}
          description={hasToken ? "O token atual deixa de funcionar na hora." : "O laboratório usa o token para enviar resultados e ler as guias agendadas."}
          requiresStepUp confirmLabel={hasToken ? "Trocar token" : "Gerar token"}
          run={async () => { setToken(await rotateFhirToken(provider.id)); }}
          onDone={() => { setConfirming(null); onChanged(); }}
          onCancel={() => setConfirming(null)} onGoToSecurity={onGoToSecurity} translateError={translate}
        />
      )}
      {confirming === "revoke" && (
        <SensitiveAction
          title={`Revogar token FHIR — ${provider.name}`}
          description="O laboratório deixa de enviar resultados até receber um token novo."
          requiresStepUp confirmLabel="Revogar token"
          run={async () => { await revokeFhirToken(provider.id); }}
          onDone={() => { setConfirming(null); setNotice("Token revogado."); onChanged(); }}
          onCancel={() => setConfirming(null)} onGoToSecurity={onGoToSecurity} translateError={translate}
        />
      )}

      {idle && (
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>
            E-mail do usuário do prestador
            <input type="email" value={email} autoComplete="off" style={inputStyle} onChange={(e) => setEmail(e.target.value)} />
          </label>
          <button type="button" style={secondaryButtonStyle} onClick={invite}>Convidar</button>
        </div>
      )}
      {emailError && <p role="alert" style={alert}>{emailError}</p>}
      {confirming !== null && typeof confirming === "object" && (
        <SensitiveAction
          title={`Convidar usuário do prestador — ${provider.name}`}
          description={`Convite para ${confirming.invite}. A pessoa cria a senha e o autenticador ao aceitar.`}
          requiresStepUp confirmLabel="Enviar convite"
          run={async () => { await inviteProviderUser(provider.id, confirming.invite); }}
          onDone={() => { setNotice(`Convite enviado para ${confirming.invite}.`); setConfirming(null); setEmail(""); }}
          onCancel={() => setConfirming(null)} onGoToSecurity={onGoToSecurity}
          translateError={(err) => inviteErrorMessage(err) ?? translate(err)}
        />
      )}
      <div><button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button></div>
    </section>
  );
}
```

```tsx
// src/modules/examsAdmin/ProvidersPanel.tsx
// Prestadores da cidade (spec §4 e §6; contrato §5): cadastro com os exames e
// prazos próprios, e os acessos (token FHIR e usuário do portal).
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  createExamProvider, errorCode, listCatalogItems, listExamProviders, updateExamProvider, type ExamProvider, type ExamProviderInput
} from "../../lib/api";
import {
  EXAM_ADMIN_CATALOG_KEY, EXAM_ADMIN_PROVIDERS_KEY, EXAM_PROVIDER_REFS_KEY, examError, providerKindLabel
} from "../../lib/exams";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { SensitiveAction } from "../../components/SensitiveAction";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { ProviderAccess } from "./ProviderAccess";
import { ProviderForm } from "./ProviderForm";
import { alert, muted, okText, row } from "../exams/styles";

type Editing = { provider: ExamProvider | null };
type Confirming = { provider: ExamProvider | null; input: ExamProviderInput };

export function ProvidersPanel({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const providers = useQuery({ queryKey: [ EXAM_ADMIN_PROVIDERS_KEY ], queryFn: listExamProviders, retry: false });
  const catalog = useQuery({ queryKey: [ EXAM_ADMIN_CATALOG_KEY ], queryFn: listCatalogItems, retry: false });
  const [ editing, setEditing ] = useState<Editing | null>(null);
  const [ confirming, setConfirming ] = useState<Confirming | null>(null);
  const [ access, setAccess ] = useState<ExamProvider | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const busy = editing !== null || access !== null;

  function reload() {
    void queryClient.invalidateQueries({ queryKey: [ EXAM_ADMIN_PROVIDERS_KEY ] });
    void queryClient.invalidateQueries({ queryKey: [ EXAM_PROVIDER_REFS_KEY ] });
  }

  return (
    <Panel title="Prestadores" sub="laboratórios e serviços que recebem as guias · prazos próprios por exame">
      <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
        {notice && <p role="status" style={okText}>{notice}</p>}
        {providers.isPending && <p className="mono" style={muted}>carregando…</p>}
        {providers.isError && <p role="alert" style={alert}>{examError(providers.error)}</p>}
        {providers.isSuccess && (
          <DataTable<ExamProvider>
            cols={[
              { label: "Nome", w: "1.6fr", card: "title", render: (p) => p.name },
              { label: "Tipo", w: "1fr", render: (p) => providerKindLabel(p.kind) },
              { label: "CNES", w: "0.9fr", render: (p) => <span className="mono">{p.cnes ?? "—"}</span> },
              { label: "Exames", w: "0.6fr", align: "right", render: (p) => String(p.items.length) },
              { label: "Token FHIR", w: "0.9fr", render: (p) => (p.has_fhir_token ? "ativo" : "sem token") },
              { label: "Situação", w: "0.8fr", render: (p) => <Tag tone={p.active ? "ok" : "neutral"}>{p.active ? "ativo" : "inativo"}</Tag> },
              { label: "", w: "auto", align: "right", card: "actions", render: (p) => (
                <span style={row}>
                  <button type="button" aria-label={`Editar ${p.name}`} style={secondaryButtonStyle} disabled={busy}
                    onClick={() => { setNotice(null); setEditing({ provider: p }); }}>Editar</button>
                  <button type="button" aria-label={`Acesso de ${p.name}`} style={secondaryButtonStyle} disabled={busy}
                    onClick={() => { setNotice(null); setAccess(p); }}>Acesso</button>
                </span>
              ) }
            ]}
            rows={providers.data} rowKey={(p) => p.id} empty="nenhum prestador cadastrado"
          />
        )}
        {providers.isSuccess && !busy && (
          <div><button type="button" style={buttonStyle} onClick={() => { setNotice(null); setEditing({ provider: null }); }}>Novo prestador</button></div>
        )}
        {editing && catalog.isError && <p role="alert" style={alert}>{examError(catalog.error)}</p>}
        {editing && catalog.isSuccess && (
          <div hidden={confirming !== null}>
            <ProviderForm key={editing.provider?.id ?? "new"} initial={editing.provider ?? undefined} catalog={catalog.data}
              onReview={(input) => setConfirming({ provider: editing.provider, input })} onCancel={() => setEditing(null)} />
          </div>
        )}
        {confirming && (
          <SensitiveAction
            title={confirming.provider ? `Salvar ${confirming.provider.name}` : `Incluir o prestador ${confirming.input.name}`}
            description={`${providerKindLabel(confirming.input.kind)} · ${confirming.input.items.length} exames.`}
            requiresStepUp confirmLabel="Salvar prestador"
            run={async () => {
              if (confirming.provider) await updateExamProvider(confirming.provider.id, confirming.input);
              else await createExamProvider(confirming.input);
            }}
            onDone={() => { setConfirming(null); setEditing(null); setNotice("Prestador salvo."); reload(); }}
            onCancel={() => setConfirming(null)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? examError(err) : null)}
          />
        )}
        {access && (
          <ProviderAccess key={access.id} provider={access} onGoToSecurity={onGoToSecurity} onChanged={reload} onClose={() => setAccess(null)} />
        )}
      </div>
    </Panel>
  );
}
```

> `ProviderAccess` lê `has_fhir_token` do prestador que recebeu ao abrir; depois de gerar ou revogar, feche e abra de novo para ver o estado relido (a lista já foi relida por `reload`).

Em `src/modules/ExamsAdmin.tsx`, importe `ProvidersPanel` de `./examsAdmin/ProvidersPanel` e acrescente `<ProvidersPanel onGoToSecurity={onGoToSecurity} />` depois do `<CatalogPanel …/>`.

Em `src/lib/team.ts`, acrescente `"exam_regulator"` ao `PRIVILEGED_ROLES` (o plano do api põe `exam_regulator` e `exam_provider` em `Membership::PRIVILEGED_ROLES`: convidar pede step-up já na primeira tela), troque o comentário `// Os 9 papéis da cidade (Membership::ROLES na API), na ordem do seletor do\n// convite.` por `// Papéis da cidade que se convidam pela Equipe (Membership::ROLES na API), na\n// ordem do seletor. O exam_provider entra só pelo cadastro do prestador (módulo 21a).` e acrescente antes de `{ role: "municipal_admin", … }`:

```ts
  { role: "exam_regulator", label: "Regulador de exames" },
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/examsAdmin.test.ts src/lib/team.test.ts src/modules/ExamsAdmin.providers.test.tsx src/modules/ExamsAdmin.catalog.test.tsx src/modules/Team.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/examsAdmin.ts src/lib/examsAdmin.test.ts src/lib/team.ts src/lib/team.test.ts src/modules/ExamsAdmin.tsx \
  src/modules/examsAdmin/ProvidersPanel.tsx src/modules/examsAdmin/ProviderForm.tsx src/modules/examsAdmin/ProviderAccess.tsx \
  src/modules/ExamsAdmin.providers.test.tsx
$G commit -m "feat: manage exam providers, their FHIR token and portal invitation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Relatório de conferência e CSV

**Files:**
- Create: `src/modules/ExamReport.tsx`
- Modify: `src/App.tsx` (`renderModule`)
- Test: `src/modules/ExamReport.test.tsx`

**Interfaces:**
- Consumes: `getExamReport`, `fetchExamReportCsv`, `listEnabledProviders`, `ExamReportRow`, `SignatureFile` (Task 1); `EXAM_REPORT_KEY`, `EXAM_PROVIDER_REFS_KEY`, `EXAMS_DISABLED`, `EXAMS_NO_ROLE`, `canUseExams`, `examRoles`, `examError` (Task 2); `todayInCity` (`lib/campaigns`).
- Produces: `ExamReport()` (tela `exam-report`; `municipal_admin` e `exam_regulator` — Divergência D14): mês (`<input type="month">`, padrão o mês da cidade), prestador (todos ou um), tabela por prestador e os dois CSV (resumo e detalhado).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ExamReport.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getExamReport: vi.fn(), fetchExamReportCsv: vi.fn(), listEnabledProviders: vi.fn() };
});

import * as api from "../lib/api";
import { ExamReport } from "./ExamReport";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21, examUser } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ROW = { provider: { id: "pv1", name: "Laboratório Simulado" }, authorized: 12, collected: 10, resulted_on_time: 8, resulted_late: 1, pending: 3 };
let downloads: string[];

afterEach(() => { cleanup(); vi.useRealTimers(); vi.restoreAllMocks(); });

describe("ExamReport", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW21));
    for (const fn of [ api.fetchCurrentSession, api.getExamReport, api.fetchExamReportCsv, api.listEnabledProviders ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "municipal_admin" ]));
    mocked(api.getExamReport).mockResolvedValue([ ROW ]);
    mocked(api.listEnabledProviders).mockResolvedValue([ { id: "pv1", name: "Laboratório Simulado" } ]);
    mocked(api.fetchExamReportCsv).mockResolvedValue({ blob: new Blob([ "short_code\n" ], { type: "text/csv" }), filename: null });
    downloads = [];
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:csv-1") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
    vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) { downloads.push(this.download); });
  });

  it("mês da cidade e todos os prestadores por padrão; filtros vão para a consulta", async () => {
    renderWithProviders(<ExamReport />);
    const panel = await screen.findByRole("region", { name: "Conferência por prestador" });
    const row = (await within(panel).findAllByRole("row"))[1];
    expect([ "12", "10", "8", "1", "3" ].every((n) => within(row).queryByText(n) !== null)).toBe(true);
    expect(api.getExamReport).toHaveBeenCalledWith("2026-10", null);
    fireEvent.change(await within(panel).findByLabelText("Prestador"), { target: { value: "pv1" } });
    await waitFor(() => expect(api.getExamReport).toHaveBeenLastCalledWith("2026-10", "pv1"));
    fireEvent.change(within(panel).getByLabelText("Mês"), { target: { value: "2026-09" } });
    await waitFor(() => expect(api.getExamReport).toHaveBeenLastCalledWith("2026-09", "pv1"));
  });

  it("baixa o CSV detalhado com nome pelo mês", async () => {
    renderWithProviders(<ExamReport />);
    const panel = await screen.findByRole("region", { name: "Conferência por prestador" });
    fireEvent.click(await within(panel).findByRole("button", { name: "Baixar CSV detalhado" }));
    await waitFor(() => expect(api.fetchExamReportCsv).toHaveBeenCalledWith("2026-10", null, true));
    await waitFor(() => expect(downloads).toEqual([ "conferencia-exames-2026-10-detalhado.csv" ]));
    expect(within(panel).getByText("O CSV detalhado traz só o código do exame, o SIGTAP e as datas — nenhum dado do paciente.")).not.toBeNull();
  });

  it("regulador vê; recepção não", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "exam_regulator" ]));
    renderWithProviders(<ExamReport />);
    expect(await screen.findByRole("region", { name: "Conferência por prestador" })).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "citizen_verifier" ]));
    mocked(api.getExamReport).mockClear();
    renderWithProviders(<ExamReport />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    expect(api.getExamReport).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/ExamReport.test.tsx`
Expected: FAIL — `./ExamReport` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/ExamReport.tsx
// Relatório de conferência (módulo 21a; spec §6 e decisão 8; contrato §5):
// por prestador e mês — autorizadas, coletadas, resultado no prazo e em
// atraso, pendentes. Tela e CSV; o detalhado leva só short_code, SIGTAP e
// datas. Admin municipal e regulador (D14).
import { useState, type ReactNode } from "react";
import { useQuery } from "@tanstack/react-query";
import { fetchExamReportCsv, getExamReport, listEnabledProviders, type ExamReportRow, type SignatureFile } from "../lib/api";
import { useAuth } from "../lib/auth";
import { todayInCity } from "../lib/campaigns";
import { EXAMS_DISABLED, EXAMS_NO_ROLE, EXAM_PROVIDER_REFS_KEY, EXAM_REPORT_KEY, canUseExams, examError, examRoles } from "../lib/exams";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ActionButton } from "../components/ActionButton";
import { inputStyle } from "../components/formStyles";
import { alert, fieldLabel, muted, row, stack } from "./exams/styles";

const BLOB_TTL_MS = 60_000;

function saveFile(file: SignatureFile, fallback: string) {
  const url = URL.createObjectURL(file.blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = file.filename ?? fallback;
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(url), BLOB_TTL_MS);
}

export function ExamReport() {
  const { user } = useAuth();
  const roles = examRoles(user);
  const allowed = canUseExams(user) && (roles.admin || roles.regulator);
  const [ month, setMonth ] = useState(() => todayInCity().slice(0, 7));
  const [ providerId, setProviderId ] = useState("");
  const [ downloadError, setDownloadError ] = useState<string | null>(null);
  const validMonth = /^\d{4}-\d{2}$/.test(month);
  const providers = useQuery({ queryKey: [ EXAM_PROVIDER_REFS_KEY, "" ], queryFn: () => listEnabledProviders(), enabled: allowed, retry: false });
  const report = useQuery({
    queryKey: [ EXAM_REPORT_KEY, month, providerId ], queryFn: () => getExamReport(month, providerId || null),
    enabled: allowed && validMonth, retry: false
  });

  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  if (!allowed) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;

  async function download(detail: boolean) {
    setDownloadError(null);
    try {
      const file = await fetchExamReportCsv(month, providerId || null, detail);
      saveFile(file, `conferencia-exames-${month}${detail ? "-detalhado" : ""}.csv`);
    } catch (err) {
      setDownloadError(examError(err));
    }
  }

  const n = (v: number) => String(v);
  return (
    <Frame>
      <Panel title="Conferência por prestador" sub="guias autorizadas, coletadas e resultados no prazo e em atraso">
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          <div style={{ ...row, alignItems: "flex-end" }}>
            <label style={fieldLabel}>
              Mês
              <input type="month" value={month} style={inputStyle} onChange={(e) => setMonth(e.target.value)} />
            </label>
            {providers.isSuccess && (
              <label style={fieldLabel}>
                Prestador
                <select value={providerId} style={inputStyle} onChange={(e) => setProviderId(e.target.value)}>
                  <option value="">todos</option>
                  {providers.data.map((p) => <option key={p.id} value={p.id}>{p.name}</option>)}
                </select>
              </label>
            )}
          </div>
          {!validMonth && <p role="alert" style={alert}>escolha o mês</p>}
          {report.isPending && validMonth && <p className="mono" style={muted}>carregando…</p>}
          {report.isError && <p role="alert" style={alert}>{examError(report.error)}</p>}
          {report.isSuccess && (
            <DataTable<ExamReportRow>
              cols={[
                { label: "Prestador", w: "1.8fr", card: "title", render: (r) => r.provider.name },
                { label: "Autorizadas", w: "0.9fr", align: "right", render: (r) => n(r.authorized) },
                { label: "Coletadas", w: "0.9fr", align: "right", render: (r) => n(r.collected) },
                { label: "Resultado no prazo", w: "1fr", align: "right", render: (r) => n(r.resulted_on_time) },
                { label: "Resultado em atraso", w: "1fr", align: "right", render: (r) => n(r.resulted_late) },
                { label: "Pendentes", w: "0.9fr", align: "right", render: (r) => n(r.pending) }
              ]}
              rows={report.data} rowKey={(r) => r.provider.id} empty="nenhuma guia neste mês"
            />
          )}
          <div style={row}>
            <ActionButton variant="secondary" disabledReason={validMonth ? null : "escolha o mês"} onClick={() => download(false)}>Baixar CSV</ActionButton>
            <ActionButton variant="secondary" disabledReason={validMonth ? null : "escolha o mês"} onClick={() => download(true)}>Baixar CSV detalhado</ActionButton>
          </div>
          <p style={muted}>O CSV detalhado traz só o código do exame, o SIGTAP e as datas — nenhum dado do paciente.</p>
          {downloadError && <p role="alert" style={alert}>{downloadError}</p>}
        </div>
      </Panel>
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Conferência de exames" sub="por prestador e mês · sem BPA-I" />
      {children}
    </div>
  );
}
```

Em `src/App.tsx`: `import { ExamReport } from "./modules/ExamReport";` e `case "exam-report": return <ExamReport />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/ExamReport.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/ExamReport.tsx src/modules/ExamReport.test.tsx src/App.tsx
$G commit -m "feat: add the monthly exam reconciliation report with CSV download

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Portal do prestador

**Files:**
- Create: `src/modules/ProviderPortal.tsx`, `src/modules/exams/ProviderOrderPanel.tsx`
- Modify: `src/App.tsx` (`renderModule`)
- Test: `src/modules/ProviderPortal.test.tsx`

**Interfaces:**
- Consumes: `listProviderOrders`, `providerSchedule`, `providerCollect`, `providerResult`, `errorCode`, `ProviderOrder`, `ProviderOrderPage` (Task 1); de `lib/exams`: `PROVIDER_ORDERS_KEY`, `EXAMS_DISABLED`, `EXAMS_NO_ROLE`, `canUseExams`, `examRoles`, `examError`, `examStatusLabel`, `examStatusTone`, `fmtDay`, `cpfTail`, `overdueLabel`, `refreshExams`, `scheduleWhen` (Task 2); `useExamOrderPages`; `ScheduleForm`, `ResultForm` (Task 4).
- Produces: `ProviderPortal()` (tela `exam-provider`): abas "A agendar", "Agendadas", "Coletadas", "Em atraso (N)" (N do `X-Overdue-Count`), a lista com nome, nascimento e o final do CPF, e `ProviderOrderPanel(props: { order: ProviderOrder; onDone(text: string): void; onClose(): void })` (`<section aria-label={`Guia ${short_code}`}>`) com agendar, registrar coleta e resultado (PDF, campos, crítico). Nada de prontuário, linha do tempo ou consulta.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/ProviderPortal.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listProviderOrders: vi.fn(), providerSchedule: vi.fn(), providerCollect: vi.fn(),
    providerResult: vi.fn(), uploadExamFile: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { ProviderPortal } from "./ProviderPortal";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW21, TODAY21, examUser, pdfFile, providerOrder } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listProviderOrders, api.providerSchedule, api.providerCollect, api.providerResult, api.uploadExamFile ];
const provider = () => examUser([ "exam_provider" ], { id: "lab1", email_address: "lab@curitiba.demo" });

afterEach(() => { cleanup(); vi.useRealTimers(); });

async function openGuide() {
  fireEvent.click(await screen.findByRole("button", { name: "Abrir guia K7M2Q9XW3P" }));
  return screen.getByRole("region", { name: "Guia K7M2Q9XW3P" });
}

describe("ProviderPortal", () => {
  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW21));
    for (const fn of ALL) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(provider());
    mocked(api.listProviderOrders).mockResolvedValue({ orders: [ providerOrder() ], next_before: null, overdue_count: 2 });
    mocked(api.providerSchedule).mockResolvedValue(undefined);
    mocked(api.providerCollect).mockResolvedValue(undefined);
    mocked(api.providerResult).mockResolvedValue(undefined);
    mocked(api.uploadExamFile).mockResolvedValue("up1");
  });

  it("lista as guias só com nome, nascimento e o final do CPF", async () => {
    renderWithProviders(<ProviderPortal />);
    const list = await screen.findByRole("region", { name: "Guias do prestador" });
    await within(list).findByText("Joana Lima");
    expect(within(list).getByText("02/05/1980")).not.toBeNull();
    expect(within(list).getByText("CPF final 725")).not.toBeNull();
    expect(screen.getByRole("status").textContent).toBe("2 guias em atraso");
    expect(screen.getByRole("tab", { name: "Em atraso (2)" })).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/prontu[aá]rio|linha do tempo|consulta/i);
    expect(api.listProviderOrders).toHaveBeenCalledWith({ statuses: [ "awaiting_schedule" ], before: null });
  });

  it("abas: situação ou atraso", async () => {
    renderWithProviders(<ProviderPortal />);
    fireEvent.click(await screen.findByRole("tab", { name: "Agendadas" }));
    await waitFor(() => expect(api.listProviderOrders).toHaveBeenLastCalledWith({ statuses: [ "scheduled" ], before: null }));
    fireEvent.click(screen.getByRole("tab", { name: "Em atraso (2)" }));
    await waitFor(() => expect(api.listProviderOrders).toHaveBeenLastCalledWith({ overdue: true, before: null }));
  });

  it("agenda sem escolher prestador", async () => {
    renderWithProviders(<ProviderPortal />);
    const guide = await openGuide();
    fireEvent.click(within(guide).getByRole("button", { name: "Agendar" }));
    const form = within(guide).getByRole("form", { name: "Agendar exame" });
    expect(within(form).queryByLabelText("Prestador")).toBeNull();
    fireEvent.change(within(form).getByLabelText("Data e hora"), { target: { value: "2026-10-20T08:30" } });
    fireEvent.change(within(form).getByLabelText("Local"), { target: { value: "Posto Centro" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar agendamento" }));
    await waitFor(() => expect(api.providerSchedule).toHaveBeenCalledWith("eo1", { scheduled_at: "2026-10-20T08:30:00-03:00", scheduled_place: "Posto Centro" }));
    expect(await screen.findByText("Guia agendada.")).not.toBeNull();
  });

  it("registra o resultado com campos e a marca de crítico", async () => {
    mocked(api.listProviderOrders).mockResolvedValue({ orders: [ providerOrder({ status: "collected" }) ], next_before: null, overdue_count: 0 });
    renderWithProviders(<ProviderPortal />);
    const guide = await openGuide();
    fireEvent.click(within(guide).getByRole("button", { name: "Registrar resultado" }));
    const form = within(guide).getByRole("form", { name: "Registrar resultado" });
    fireEvent.change(within(form).getByLabelText("Laudo (PDF, JPEG ou PNG, até 10 MB)"), { target: { files: [ pdfFile() ] } });
    await within(form).findByText("anexado: laudo.pdf");
    fireEvent.change(within(form).getByLabelText("Laboratório"), { target: { value: "Laboratório Simulado" } });
    fireEvent.change(within(form).getByLabelText("Hemoglobina (g/dL)"), { target: { value: "6.9" } });
    fireEvent.click(within(form).getByLabelText("Resultado crítico (vai direto para a fila da equipe)"));
    fireEvent.click(within(form).getByRole("button", { name: "Registrar resultado" }));
    await waitFor(() => expect(api.providerResult).toHaveBeenCalledWith("eo1", {
      upload_id: "up1", released_on: TODAY21, laboratory_name: "Laboratório Simulado", critical: true, values: [ { key: "hemoglobina", value: "6.9" } ]
    }));
  });

  it("guia que não é mais deste prestador: 404 explica e relê", async () => {
    mocked(api.listProviderOrders).mockResolvedValue({ orders: [ providerOrder({ status: "scheduled" }) ], next_before: null, overdue_count: 0 });
    mocked(api.providerCollect).mockRejectedValue(new ApiError(404, { error: "not_found" }, "x"));
    renderWithProviders(<ProviderPortal />);
    const guide = await openGuide();
    fireEvent.click(within(guide).getByRole("button", { name: "Registrar coleta" }));
    fireEvent.click(within(guide).getByRole("button", { name: "Confirmar" }));
    expect((await within(guide).findByRole("alert")).textContent).toBe("exame não encontrado — atualize a lista");
    await waitFor(() => expect(api.listProviderOrders).toHaveBeenCalledTimes(2));
  });

  it("só o papel do prestador, com o rastreamento ligado", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "health_professional" ]));
    renderWithProviders(<ProviderPortal />);
    expect(await screen.findByText("seu papel não permite ver esta tela")).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser([ "exam_provider" ], { features: [] }));
    renderWithProviders(<ProviderPortal />);
    expect(await screen.findByText("o rastreamento de exames está desligado nesta cidade")).not.toBeNull();
    expect(api.listProviderOrders).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/ProviderPortal.test.tsx`
Expected: FAIL — `./ProviderPortal` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/exams/ProviderOrderPanel.tsx
// Uma guia no portal do prestador (spec §6; contrato §2 `<provider_order>` e
// §6): só nome, nascimento e o final do CPF; agendar (o prestador é o do
// usuário — D9), registrar coleta e resultado com a marca "crítico". Guia
// que mudou ou não é mais dele (409/404) explica e relê a lista.
import { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { errorCode, providerCollect, providerResult, providerSchedule, type ProviderOrder } from "../../lib/api";
import { cpfTail, examError, examStatusLabel, examStatusTone, fmtDay, refreshExams, scheduleWhen } from "../../lib/exams";
import { cityIsoDate, fmtDateTime } from "../../lib/format";
import { ConfirmButton } from "../../components/ConfirmButton";
import { KeyValue } from "../../components/KeyValue";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { ResultForm } from "./ResultForm";
import { ScheduleForm } from "./ScheduleForm";
import { alert, grid, panel, row, title } from "./styles";

const STALE = new Set([ "invalid_transition", "not_found" ]);

export function ProviderOrderPanel({ order, onDone, onClose }: { order: ProviderOrder; onDone(text: string): void; onClose(): void }) {
  const queryClient = useQueryClient();
  const [ mode, setMode ] = useState<null | "schedule" | "result">(null);
  const [ error, setError ] = useState<string | null>(null);

  async function act(run: () => Promise<void>, okText: string): Promise<void> {
    setError(null);
    try {
      await run();
    } catch (err) {
      if (!STALE.has(errorCode(err) ?? "")) throw err;
      setMode(null);
      setError(examError(err));
      await refreshExams(queryClient);
      return;
    }
    await refreshExams(queryClient);
    onDone(okText);
  }

  return (
    <section aria-label={`Guia ${order.short_code}`} style={panel}>
      <div style={row}>
        <h3 style={title}>{order.label}</h3>
        <Tag tone={examStatusTone(order.status)}>{examStatusLabel(order.status)}</Tag>
        {order.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}
      </div>
      <div style={grid}>
        <KeyValue k="Paciente" v={order.patient.name} mono={false} />
        <KeyValue k="Nascimento" v={fmtDay(order.patient.birth_date)} />
        <KeyValue k="CPF" v={cpfTail(order.patient.cpf_last3)} mono={false} />
        <KeyValue k="SIGTAP" v={order.sigtap_code} />
        <KeyValue k="Agendado" v={order.scheduled_at ? `${fmtDateTime(order.scheduled_at)} · ${order.scheduled_place ?? ""}` : "—"} mono={false} />
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
      {mode === null && (
        <div style={row}>
          {order.status === "awaiting_schedule" && (
            <button type="button" style={secondaryButtonStyle} onClick={() => setMode("schedule")}>Agendar</button>
          )}
          {order.status === "scheduled" && (
            <ConfirmButton label="Registrar coleta" question="Confirmar que o material foi coletado?"
              onConfirm={() => act(() => providerCollect(order.id), "Coleta registrada.").catch((err) => setError(examError(err)))} />
          )}
          {(order.status === "scheduled" || order.status === "collected") && (
            <button type="button" style={secondaryButtonStyle} onClick={() => setMode("result")}>Registrar resultado</button>
          )}
          <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>
        </div>
      )}
      {mode === "schedule" && (
        <ScheduleForm providers={null} onCancel={() => setMode(null)}
          onSubmit={(d) => act(() => providerSchedule(order.id, scheduleWhen(d)), "Guia agendada.")} />
      )}
      {mode === "result" && (
        <ResultForm fields={order.result_fields ?? []} today={cityIsoDate()} submitLabel="Registrar resultado" onCancel={() => setMode(null)}
          onSubmit={(input) => act(() => providerResult(order.id, input), "Resultado registrado.")} />
      )}
    </section>
  );
}
```

```tsx
// src/modules/ProviderPortal.tsx
// Portal do prestador (módulo 21a; spec §6 e decisão 9; contrato §6): as
// guias do prestador do usuário `exam_provider`, por situação e em atraso.
// O isolamento é do api (guia de outro prestador = 404); a tela não tem
// caminho para prontuário, linha do tempo ou consulta.
import { useState, type ReactNode } from "react";
import { listProviderOrders, type ProviderOrder } from "../lib/api";
import { useAuth } from "../lib/auth";
import {
  EXAMS_DISABLED, EXAMS_NO_ROLE, PROVIDER_ORDERS_KEY, canUseExams, cpfTail, examError, examRoles, fmtDay, overdueLabel
} from "../lib/exams";
import { fmtDateTime } from "../lib/format";
import { useExamOrderPages } from "../hooks/useExamOrderPages";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { Tag } from "../components/Tag";
import { secondaryButtonStyle } from "../components/formStyles";
import { SegmentedControl } from "../shell/SegmentedControl";
import { ProviderOrderPanel } from "./exams/ProviderOrderPanel";
import { alert, muted, okText, row, stack } from "./exams/styles";

type PortalTab = "awaiting_schedule" | "scheduled" | "collected" | "overdue";

export function ProviderPortal() {
  const { user } = useAuth();
  const allowed = canUseExams(user) && examRoles(user).provider;
  const [ tab, setTab ] = useState<PortalTab>("awaiting_schedule");
  const [ open, setOpen ] = useState<ProviderOrder | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const { query, orders } = useExamOrderPages([ PROVIDER_ORDERS_KEY, tab ],
    (before) => listProviderOrders(tab === "overdue" ? { overdue: true, before } : { statuses: [ tab ], before }), allowed);

  if (!user) return null;
  if (!canUseExams(user)) return <Frame><EmptyState title={EXAMS_DISABLED} /></Frame>;
  if (!allowed) return <Frame><EmptyState title={EXAMS_NO_ROLE} /></Frame>;

  const overdueCount = query.data?.pages[0]?.overdue_count ?? 0;
  const tabs: { key: PortalTab; label: string }[] = [
    { key: "awaiting_schedule", label: "A agendar" },
    { key: "scheduled", label: "Agendadas" },
    { key: "collected", label: "Coletadas" },
    { key: "overdue", label: overdueCount > 0 ? `Em atraso (${overdueCount})` : "Em atraso" }
  ];

  return (
    <Frame>
      {overdueCount > 0 && (
        <p role="status" style={{ ...alert, fontWeight: 600 }}>{overdueCount === 1 ? "1 guia em atraso" : `${overdueCount} guias em atraso`}</p>
      )}
      {notice && <p style={okText}>{notice}</p>}
      {open && (
        <ProviderOrderPanel key={open.id} order={open} onClose={() => setOpen(null)}
          onDone={(text) => { setOpen(null); setNotice(text); }} />
      )}
      <Panel title="Guias do prestador" sub="coletas e resultados · prazos da cidade">
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          <SegmentedControl<PortalTab> options={tabs} value={tab} onChange={(t) => { setTab(t); setOpen(null); }} />
          {query.isPending && <p className="mono" style={muted}>carregando…</p>}
          {query.isError && <p role="alert" style={alert}>{examError(query.error)}</p>}
          {query.isSuccess && (
            <DataTable<ProviderOrder>
              cols={[
                { label: "Código", w: "1fr", render: (o) => <span className="mono">{o.short_code}</span> },
                { label: "Paciente", w: "1.4fr", card: "title", render: (o) => o.patient.name },
                { label: "Nascimento", w: "0.9fr", render: (o) => fmtDay(o.patient.birth_date) },
                { label: "CPF", w: "0.9fr", render: (o) => cpfTail(o.patient.cpf_last3) },
                { label: "Exame", w: "1.6fr", render: (o) => (
                  <span style={row}><span>{o.label}</span>{o.priority === "urgent" && <Tag tone="urgent">urgente</Tag>}</span>
                ) },
                { label: "Agendado", w: "1fr", render: (o) => (o.scheduled_at ? fmtDateTime(o.scheduled_at) : "—") },
                { label: "Prazo", w: "1.1fr", render: (o) => overdueLabel(o.overdue) },
                { label: "", w: "auto", align: "right", card: "actions", render: (o) => (
                  <button type="button" aria-label={`Abrir guia ${o.short_code}`} style={secondaryButtonStyle}
                    onClick={() => { setNotice(null); setOpen(o); }}>Abrir</button>
                ) }
              ]}
              rows={orders} rowKey={(o) => o.id} empty="nenhuma guia nesta situação"
            />
          )}
          {query.hasNextPage && (
            <div><button type="button" style={secondaryButtonStyle} onClick={() => void query.fetchNextPage()}>Carregar mais</button></div>
          )}
        </div>
      </Panel>
    </Frame>
  );
}

function Frame({ children }: { children: ReactNode }) {
  return (
    <div style={stack}>
      <PageHeader title="Portal do prestador" sub="guias do seu laboratório" />
      {children}
    </div>
  );
}
```

> O aviso de sucesso do portal não usa `role="status"` (o `status` é do aviso de atraso, que o teste lê como único): é o mesmo texto visível, sem disputar o anúncio.

Em `src/App.tsx`: `import { ProviderPortal } from "./modules/ProviderPortal";` e `case "exam-provider": return <ProviderPortal />;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/ProviderPortal.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/ProviderPortal.tsx src/modules/exams/ProviderOrderPanel.tsx src/modules/ProviderPortal.test.tsx src/App.tsx
$G commit -m "feat: add the exam provider portal

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Entrada `exam` na linha do tempo do profissional

**Files:**
- Modify: `src/lib/api.ts` (`TimelineKind`), `src/lib/timeline.ts`, `src/modules/timeline/TimelineDetail.tsx`
- Test: `src/lib/timeline.exam.test.ts`, `src/modules/timeline/TimelineDetail.exam.test.tsx`

**Interfaces:**
- Consumes: `TimelineEntry`, `detailTarget`, `statusLabel`, `timelineKindLabel`, `TIMELINE_KINDS` (módulo 20); `ExamOrderDetail` (Task 5).
- Produces: `TimelineKind` com `"exam"`; `TIMELINE_KINDS` termina com `{ key: "exam", label: "Exame" }`; `statusLabel` traduz `awaiting_authorization`, `denied`, `awaiting_schedule`, `collected`, `resulted`, `acknowledged` (`scheduled` e `cancelled` já existem); `DetailType` com `"exam_order"` (contrato §10: `detail_ref: { type: "exam_order", id }`), aberto só em contexto (como documento, escuta e pedido — Divergência D13); `TimelineDetail` mostra `ExamOrderDetail`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/timeline.exam.test.ts
import { describe, expect, it } from "vitest";
import { TIMELINE_KINDS, detailTarget, filtersFrom, statusLabel, timelineKindLabel } from "./timeline";
import { entry } from "../test/timelineFixtures";

describe("linha do tempo — exames (módulo 21a; contrato §10)", () => {
  const exam = entry({ id: "exam:eo1", kind: "exam", title: "Hemograma completo", status: "awaiting_schedule", detail_ref: { type: "exam_order", id: "eo1" } });

  it("tipo e situações em português", () => {
    expect(timelineKindLabel("exam")).toBe("Exame");
    expect(TIMELINE_KINDS[TIMELINE_KINDS.length - 1]).toEqual({ key: "exam", label: "Exame" });
    expect(statusLabel("awaiting_schedule")).toBe("a agendar");
    expect(statusLabel("resulted")).toBe("resultado registrado");
    expect(statusLabel("acknowledged")).toBe("ciência dada");
    expect(statusLabel("denied")).toBe("negado");
  });

  it("abre o exame em contexto; pela abertura justificada, não", () => {
    expect(detailTarget(exam, true)).toEqual({ type: "exam_order", id: "eo1" });
    expect(detailTarget(exam, false)).toBeNull();
  });

  it("todos os tipos marcados continuam sem kinds[]; só exame vai como kinds[]=exam", () => {
    const all = TIMELINE_KINDS.map((k) => k.key);
    expect(filtersFrom({ kinds: all, from: "", to: "" }).kinds).toEqual([]);
    expect(filtersFrom({ kinds: [ "exam" ], from: "", to: "" }).kinds).toEqual([ "exam" ]);
  });
});
```

```tsx
// src/modules/timeline/TimelineDetail.exam.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getExamOrder: vi.fn() };
});

import * as api from "../../lib/api";
import { TimelineDetail } from "./TimelineDetail";
import { renderWithProviders } from "../../test/campaignFixtures";
import { examOrder, examUser } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("TimelineDetail — exam_order", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(examUser());
    mocked(api.getExamOrder).mockResolvedValue(examOrder());
  });

  it("mostra o exame pela tela de exames e fecha", async () => {
    const onClose = vi.fn();
    renderWithProviders(<TimelineDetail target={{ type: "exam_order", id: "eo1" }} onClose={onClose} />);
    expect(await screen.findByRole("region", { name: "Exame Hemograma completo" })).not.toBeNull();
    expect(api.getExamOrder).toHaveBeenCalledWith("eo1");
    fireEvent.click(screen.getByRole("button", { name: "Fechar" }));
    expect(onClose).toHaveBeenCalled();
  });
});
```

> `entry(over)` é o construtor de `src/test/timelineFixtures.ts` (módulo 20). Confira o nome com `grep -n "export function" src/test/timelineFixtures.ts`; se for outro, use o real.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/timeline.exam.test.ts src/modules/timeline/TimelineDetail.exam.test.tsx`
Expected: FAIL — `"exam"` não é tipo conhecido e `exam_order` não abre.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`:

```ts
export type TimelineKind =
  | "triage" | "attendance" | "screening" | "appointment" | "consultation"
  | "clinical_document" | "controlled_notification" | "campaign" | "exam";
```

Em `src/lib/timeline.ts`:
- em `TIMELINE_KINDS`, depois de `{ key: "campaign", label: "Campanha" }`, acrescente `{ key: "exam", label: "Exame" }` (no fim: a ordem dos tipos de antes não muda);
- em `STATUS_LABEL`, depois de `unread: "aviso não lido"`, acrescente:

```ts
  // Módulo 21a (contrato §1): situação do exame (scheduled e cancelled já estão acima).
  awaiting_authorization: "aguardando autorização",
  denied: "negado",
  awaiting_schedule: "a agendar",
  collected: "coletado",
  resulted: "resultado registrado",
  acknowledged: "ciência dada"
```
(ponha vírgula depois de `unread: "aviso não lido"`);
- troque `DetailType` e `OPENABLE`:

```ts
export type DetailType = "consultation" | "clinical_document" | "screening" | "appointment_request" | "exam_order";
export interface DetailTarget { type: DetailType; id: string }
const OPENABLE: ReadonlySet<string> = new Set<DetailType>([ "consultation", "clinical_document", "screening", "appointment_request", "exam_order" ]);
```
(a regra `if (!inContext && ref.type !== "consultation") return null;` já deixa o exame só em contexto).

Em `src/modules/timeline/TimelineDetail.tsx`: `import { ExamOrderDetail } from "../exams/ExamOrderDetail";` e, no `switch`, depois do `case "appointment_request"`:

```tsx
    case "exam_order":
      return <ExamOrderDetail id={target.id} onClose={onClose} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/timeline src/modules/timeline src/modules/clinicalRecord src/modules/consultation && npx tsc --noEmit`
Expected: PASS. Se um teste antigo do módulo 20 contar os tipos (8 caixas de filtro), ele passa a contar 9: atualize **só** esse número, com o comentário `// + exame (módulo 21a)`.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/timeline.ts src/lib/timeline.exam.test.ts src/modules/timeline/TimelineDetail.tsx src/modules/timeline/TimelineDetail.exam.test.tsx
$G commit -m "feat: show exam entries in the professional timeline

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se a Step 4 mudou uma contagem antiga, acrescente o arquivo dela ao `git add`.)

---

### Task 17: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–16 na branch `feat/mod-21a-exams`; o api do 21a na **3041** com a semente do 21a e o `fake-lab`.
- Produces: branch pronta para merge (depois do merge do api do 21a); evidências em `evidencias/2026-10-13-modulo-21a/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **23 arquivos de teste novos** (`api.exams`, `exams`, `modules.exams`, `AcknowledgementNotice`, `UploadField`, `forms`, `ExamOrderDetail`, `consultation.priority`, `ExamRequestsField.priority`, `ConsultationExamStatus`, `PatientExams`, `PatientPanel.exams`, `ExamAcknowledgements`, `ExamsUnit`, `ExamsOverdue`, `ExamRegulation`, `examsAdmin`, `ExamsAdmin.catalog`, `ExamsAdmin.providers`, `ExamReport`, `ProviderPortal`, `timeline.exam`, `TimelineDetail.exam`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/SegmentedControl.tsx src/shell/Shell.tsx src/shell/Cards.tsx src/theme
```

Expected: `status` vazio e nenhum componente comum, peça do shell ou tema mudado.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage" src/lib/exams.ts src/lib/examsAdmin.ts src/modules/exams src/modules/examsAdmin \
  src/modules/Exam*.tsx src/modules/ExamsUnit.tsx src/modules/ExamsOverdue.tsx src/modules/ProviderPortal.tsx src/hooks/useExam*.ts | grep -v "\.test\."
sed -n '/─── Exames (módulo 21a/,$p' src/lib/api.ts | grep -n "p.set(\|p.append(" 
```

Expected: o primeiro sem saída; o segundo só com `patient_id`, `unit_id`, `consultation_id`, `status[]`, `overdue`, `stage`, `provider_id`, `sigtap_code`, `before`, `limit`, `month`, `detail` — nenhum `cpf`, `cns`, `name`, `reason` ou `value`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§2–§9, §11), o ADR 0036, o contrato (§1–§6, §10, §12) e as Divergências deste plano. Pontos de atenção:
- sem `exam_tracking`, menu, consulta, painel do paciente e linha do tempo são exatamente os de antes (testes antigos verdes, só as duas contagens de `modules.test.ts` e a lista de papéis de `team.test.ts` mudaram);
- o prestador não tem caminho para prontuário, linha do tempo ou consulta; o portal mostra só nome, nascimento e o final do CPF;
- na URL só ids, códigos, situações, etapas, mês e cursor; CPF da busca no corpo; upload sem o nome do arquivo;
- token FHIR só no estado do `ProviderAccess`, nunca no cache; some com "Já guardei";
- 409 `invalid_transition` sempre fecha o formulário, explica e relê; link de laudo vencido relê o exame;
- ciência da equipe exige motivo; cancelar e negar exigem motivo;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 21a na **3041** (plano do api: servidor do worktree `apps/api/.claude/mod21` no `api-dev`, semente com o catálogo de exames, o prestador "Laboratório Simulado" e os papéis `exam_regulator` e `exam_provider`, o serviço `fake-lab` no compose). Ligue em Curitiba, pelo maintenance, `exam_tracking`, `exam_fhir_inbound` e `fake_lab` (e `clinical_record`, se estiver desligado). O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo: o do worktree é do macOS e não roda no Alpine):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod21 --network rota-saude_default -p 5192:5192 \
  -v $R/.worktrees/dashboard-mod21:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3041 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5192 --host 0.0.0.0"
docker logs -f dashboard-mod21   # espere o "ready"; Ctrl+C sai do log, o container segue
docker compose up -d fake-lab
```

(Se o plano do api subir o servidor do 21a num container próprio, use o nome dele no lugar de `api`. O `fake-lab` usa o `FAKE_LAB_TOKEN` padrão de dev do plano do api (`dev-fake-lab-token-…`, já gravado como hash no prestador simulado da semente). **Não** gere token novo para o "Laboratório Simulado" na prova do admin (o novo invalida o da semente e o `fake-lab` passa a levar 401): gere o token num prestador criado na prova. Se precisar trocar o do `fake-lab`, ponha o novo no ambiente do serviço e recrie com `docker compose up -d --force-recreate fake-lab` — variável nova no arquivo não chega a container já rodando; confira com `docker exec <container do fake-lab> printenv FAKE_LAB_TOKEN | wc -c`. Contas da semente do api: `laboratorio@curitiba.demo` (`exam_provider`) e `regulacao@curitiba.demo` (`exam_regulator`).)

Abra `http://curitiba.localhost:5192/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot (salve em `evidencias/2026-10-13-modulo-21a/dashboard/`):
1. **Admin** (`admin@curitiba.demo`): Exames → Catálogo e prestadores — o catálogo da semente com prazos efetivos; incluir um exame com um campo de resultado (step-up); incluir um prestador de prova com esse exame; nele, **Gerar token** → aparece uma vez → "Já guardei" → some; convidar um usuário desse prestador; Equipe → convite com "Regulador de exames" na lista.
2. **Médica**: Atendimento → chamar a pessoa da semente → consulta → pedir hemograma (**Urgente**) e um exame que exige autorização → finalizar → a consulta mostra cada exame com a situação ("a agendar", "aguardando autorização"); aba **Exames** do paciente com os dois.
3. **Regulador** (`regulacao@curitiba.demo`): Fila de regulação → urgente primeiro → autorizar escolhendo "Laboratório Simulado"; negar outro com motivo → a médica vê no Início "1 exame negado" e o motivo no andamento.
4. **Recepção**: Exames da unidade → **Nova guia** (CPF da semente, profissional da rede, foto da guia) → código curto; abrir o exame → **Agendar** (prestador) → "agendado".
5. **fake-lab**: em até 30 s o exame agendado do prestador simulado vira "resultado registrado", origem "recebido do laboratório (FHIR)", com os valores (hemograma ou HbA1c); **Abrir laudo** abre o PDF de exemplo.
6. **Médica**: Início com "1 resultado aguardando sua ciência" e o contador no menu → **Dar ciência**; o contador some no minuto seguinte.
7. **Prestador** (`laboratorio@curitiba.demo`): o menu tem só "Portal do prestador" e "Segurança"; `#/attendance` na barra de endereço volta ao Início; no portal, as guias com nome, nascimento e "CPF final 123"; agendar uma guia, registrar coleta e um resultado com **crítico** → o exame entra na **Fila da equipe** de um enfermeiro da equipe de referência, que dá ciência com motivo.
8. **Atraso**: com o job diário do plano do api (o comando que ele fixar) sobre um exame de prazo vencido da semente, "Exames atrasados" da unidade e da coordenação (como regulador) mostram etapa e dias; os três filtros mudam a lista.
9. **Resultado externo**: na consulta (aba Exames → Registrar resultado externo) e pela unidade (Resultado externo → CPF); aparecem em "Resultados externos".
10. **Relatório** (admin e regulador): mês corrente e um prestador; **Baixar CSV** e **Baixar CSV detalhado** — abra o detalhado e confira que só há `short_code`, SIGTAP e datas.
11. **Linha do tempo**: na aba Linha do tempo do paciente, entradas "Exame" com a situação; **Abrir** mostra o exame.
12. **Telefone** (largura 375): portal do prestador, exames da unidade e fila de regulação viram cartões, sem rolagem lateral.
13. No maintenance, desligar `exam_tracking` e recarregar: sem o grupo "Exames", sem prioridade na consulta, sem a aba Exames; o prestador vê só "Segurança".

Anote 403 de painéis no console da sessão do prestador (achado do shell, não do 21a). Depois: `docker rm -f dashboard-mod21`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

Para cada card F-21.2, .3, .4, .5, .6, .8, .9 e .12: se a parte do api dele já está Done, mova para **Done** (o mesmo `gh project item-edit` da Task 0, com o id da opção Done `98236657`); senão deixe em In Progress e anote. **Pare.** O merge do dashboard só vem depois do merge do api do 21a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard e wpda → maintenance (contrato §14). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-21a-exams
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod21
/opt/homebrew/bin/git branch -d feat/mod-21a-exams
```

---

## Divergências propostas ao contrato

1. **D1 — Upload em JSON, não multipart.** O contrato §3 diz `POST /exams/uploads` multipart. O api recusa com 415 `json_required` toda escrita por cookie que não seja `application/json` (`concerns/authentication.rb`): é a defesa contra CSRF do projeto, porque multipart é um dos três tipos que um formulário de outro site consegue enviar sem preflight. Proposta: `POST /exams/uploads` com `{ "content_type": "application/pdf"|"image/jpeg"|"image/png", "data": "<base64>" }` (≤ 10 MB decodificado), sem nome de arquivo (o nome pode trazer o nome do paciente). Mesma resposta `{ upload_id }` e 422 `invalid_file`. Se o contrato mantiver multipart, o api precisa de outra defesa (cabeçalho exigido) e só `uploadExamFile` muda aqui.
2. **D2 — Busca do paciente pela recepção.** `POST /exams/orders` e `POST /exams/external_results` pedem `patient_id`, mas a recepção não tem como achá-lo (o plano do api aceita também `citizen_id` de um par validado, mas a tela da recepção também não o tem fora do check-in). Proposta: `POST /exams/patient_lookup` `{ "cpf" }` (CPF no corpo) → `{ "patient": { "id", "display_name", "cpf_masked" } }`, resolvendo o par validado do CPF com o mesmo `Patients::Resolve` do plano do api (Desvio 3: acha ou cria o paciente); 404 `patient_not_found`; 409 `citizen_not_verified` (par só declarado); papéis `citizen_verifier`, `health_professional`, `municipal_admin`; com trilha.
3. **D3 — Catálogo legível pela unidade e o nome do exame no admin.** A guia e o resultado externo precisam dos exames ativos com os campos de resultado; o contrato só tem `/exams/admin/catalog_items` (admin). Proposta: `GET /exams/catalog` → `{ "items": [ { "sigtap_code", "label", "result_fields" } ] }` para os papéis de exames. E `GET /exams/admin/catalog_items` responde `{ "items": [ … ] }` com `id` e `label` (nome SIGTAP, só leitura).
4. **D4 — Prestadores habilitados para um exame.** Agendar sem prestador, autorizar e os filtros de atraso e do relatório precisam escolher prestador; `/exams/admin/providers` é só do admin. Proposta: `GET /exams/providers?sigtap_code=` → `{ "providers": [ { "id", "name" } ] }` (ativos que fazem o exame; sem `sigtap_code`, todos os ativos), para unidade, regulador e admin.
5. **D5 — Quem pode ser "solicitante da rede" na guia.** Proposta: `GET /exams/requesters?unit_id=` → `{ "requesters": [ { "user_id", "name" } ] }` (profissionais com vínculo ativo na unidade).
6. **D6 — Exame ligado à consulta e campos de resultado na guia.** Para mostrar a situação na consulta, `<exam_order>` ganha `consultation_id` (nulo fora de `consultation`) e `GET /exams/orders` aceita `consultation_id=`. Para registrar resultado, `<exam_order>` (no `GET /exams/orders/:id`) e `<provider_order>` ganham `result_fields` (os do catálogo da cidade).
7. **D7 — Forma das listas de ciência.** O contrato não fixa. Proposta: `GET /exams/acknowledgements/pending` → `{ "orders": [<exam_order sem events>], "denied": [<exam_order sem events>] }` (`denied`: exames do solicitante negados nos últimos 30 dias — é o "avisa quem pediu" da spec §5); `GET /exams/acknowledgements/team` → `{ "orders": [ … ] }`.
8. **D8 — Resultados externos do paciente.** O resultado externo não é `exam_order` e não aparece em `/exams/orders`. Proposta: `GET /exams/external_results?patient_id=` → `{ "results": [ <exam_result> + "sigtap_code", "label" ] }`, pelas regras de leitura do 19a.
9. **D9 — Prestador: agendamento, token e convite.** No portal, `POST /exams/provider/orders/:id/schedule` não leva `provider_id` (é o do usuário; "mesmos corpos do §3" o exigiria). `<provider>` do admin ganha `has_fhir_token: bool` (o token nunca volta) e a lista responde `{ "providers": [ … ] }`. `POST /exams/admin/providers/:id/users` recebe `{ "email" }` e responde o `<invitation>` do convite existente (`{ id, email, role, expires_at }`); recusas iguais às do convite (`already_member`, `already_invited`, `invalid`).
10. **D10 — Prioridade no corpo da consulta.** A spec §4 diz que `consultation_exam_requests` ganha `priority`; o contrato não diz o campo. Proposta: `exam_requests[].priority` (`routine` | `urgent`) no PATCH do rascunho e na leitura da consulta; ausente = `routine`.
11. **D11 — `pdf_url` vencido.** Proposta: 410 `link_expired` (o dashboard também trata 403 como vencido e relê o exame para ganhar um `pdf_url` novo).
12. **D12 — Ponte do PEC (§8).** ~~A prova técnica falhou~~ — corrigido pelo coordenador (2026-10-14): a prova ainda não foi feita (Task 0a do api). O §8 continua condicional; nenhuma tela do dashboard para ela, porque o formulário é HTML do api.
13. **D13 — Exame pela abertura justificada.** A spec §7 cita a abertura entre as regras de leitura, mas o contrato não diz se `GET /exams/orders/:id` aceita `opening_id`. Este plano abre o exame da linha do tempo só em contexto (como documento, escuta e pedido no módulo 20). Se o api aceitar a abertura, é só tirar `exam_order` da regra de `detailTarget`.
14. **D14 — Relatório para o regulador.** A spec §6 diz "admin municipal e regulador", mas o contrato §5 põe `/exams/admin/report` só para `municipal_admin`. Proposta: o relatório (tela e CSV) aceita também `exam_regulator`.
15. **D15 — Atrasos da coordenação.** A spec §5 fala em painel "da unidade e da coordenação"; o contrato §3 exige `patient_id` ou `unit_id`. Proposta: `GET /exams/orders?overdue=true` sem unidade nem paciente para `municipal_admin` e `exam_regulator` (a cidade inteira), com `unit` em cada exame. A coordenação só lista; abrir o exame continua na unidade.

16. **D16 — A guia leva a unidade.** O contrato §3 não tem `unit_id` no corpo de `POST /exams/orders`; o plano do api o exige (a guia é da unidade e o api confere quem atua nela, `not_unit_staff`). Proposta: `unit_id` obrigatório no corpo da guia — o dashboard manda a unidade escolhida no `UnitPicker`.

Alinhamento com o plano do api (escrito em paralelo): a D1 deste plano é a D3 dele (mesmo corpo `{ content_type, data }`); a visão administrativa do resultado (Desvio 10 dele: unidade e regulador sem valores, crítico e laudo) já é tratada aqui (Task 5, `ExamResultBlock`); `exam_regulator` entra em `PRIVILEGED_ROLES` dos dois lados (Task 13); as contas `laboratorio@<slug>.demo` e `regulacao@<slug>.demo` e o `FAKE_LAB_TOKEN` de dev dele são os da prova (Task 17). As rotas das D2–D9, D14 e D15 não aparecem no plano do api: precisam entrar no contrato e nele antes da Task 17.

## Self-review

- **Cobertura (spec §9 e contrato §3–§6, §10):** status na consulta — Task 6; aba do paciente — Task 7; contador e aviso "resultados aguardando ciência" no Início — Task 3; fila da equipe e negados — Task 8; resultado externo — Tasks 7 e 9; guia da recepção com upload — Tasks 4 e 9; agendar, coleta e resultado pela unidade — Tasks 4, 5 e 9; painel "Exames atrasados" com etapa, prestador e exame (unidade e coordenação) — Task 10; fila de regulação (autorizar com prestador, negar com motivo) — Task 11; catálogo (prazos, política, campos) — Task 12; prestadores (itens e prazos, token mostrado uma vez, convite) — Task 13; relatório com CSV — Task 14; portal do prestador (lista, agendar, coleta, resultado com PDF e campos, crítico, sem prontuário) — Task 15; entrada `exam` na linha do tempo — Task 16; prova no navegador com API real na 3041 e `fake-lab` — Task 17. Botão no PEC: sem tela no dashboard (HTML do api, condicional à prova; D12). Prioridade no pedido (spec §4) — Task 6. Papel `exam_regulator` no convite — Task 13. Cidadão (wpda) e interruptores (maintenance) são de outros planos.
- **Placeholders:** nenhum código faltando. Dois pontos dependem de nome real a conferir, com instrução explícita: o construtor `entry` de `timelineFixtures` (Task 16) e o comando do job diário e o container do api (Task 17, do plano do api).
- **Tipos:** `ExamOrder`, `ExamOrderPage`, `ProviderOrder`, `ProviderOrderPage`, `ResultInput`, `ScheduleInput`/`ScheduleWhen`, `ExamCatalogItemInput`, `ExamProviderInput` (Task 1) são os usados nas Tasks 2–16; `useExamOrderPages<P>` (Task 2) devolve `P["orders"][number][]` e as telas leem `query.data.pages[0].overdue_count` no portal; `scheduleBody(d, providerId?)` e `scheduleWhen(d)` (Task 2) são os de `ExamOrderDetail` e `ProviderOrderPanel`; `ResultForm`/`ScheduleForm`/`ReasonForm` (Task 4) têm as mesmas props em todas as telas; `refreshExams(queryClient)` cobre as chaves de todas as listas que andam; `DetailType` ganha `exam_order` e `TimelineDetail` o trata.
- **Review Focus:** 1 — Task 5 ("laudo com link vencido…"); 2 — Task 5 ("409 invalid_transition…") e Task 11 ("outro regulador decidiu antes…"); 3 — Task 13 ("token FHIR: aparece uma vez…"); 4 — Task 3 ("o prestador vê só o portal…", "prestador com o rastreamento desligado…") e Task 15 ("lista as guias só com nome…"); 5 — Task 1 ("upload: JSON… sem o nome do arquivo"), Task 4 ("recusa tipo, tamanho e vazio…", "422 invalid_file…") e Task 9 ("guia sem anexo não é enviada").
