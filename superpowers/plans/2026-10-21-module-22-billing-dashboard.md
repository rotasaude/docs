# Módulo 22 — Procedimentos e faturamento BPA (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 22** (`feat/mod-22-billing`, worktree `apps/api/.claude/mod22`, porta de dev **3044**) com as rotas `/billing/*` do contrato e as das **Divergências** deste plano (fim do arquivo), o interruptor `procedure_billing` na sessão, o papel `billing_clerk` em `Membership::ROLES`/`PRIVILEGED_ROLES`, o `kind` `procedure` na linha do tempo e a semente do 22 (UPA com CNES, profissional com CNS e vínculo na UPA, profissional sem CNS, faturista, laudo liberado do laboratório municipal). Ele é **mergeado antes** deste (contrato §11: contracts → api → dashboard → maintenance → catálogo). As Tasks 0–16 não dependem dele (os testes usam dublês de `src/lib/api`); a prova da Task 17 depende.

**Goal:** Lado dashboard do módulo 22: registrar o procedimento realizado no atendimento e na consulta (busca SIGTAP, quantidade, CID quando a tabela exige, caráter) e avulso pela unidade, mostrando a crítica como "não faturável — motivo" sem nunca bloquear; o grupo **"Faturamento"** com a lista de procedimentos filtrável e o lançamento manual do `billing_clerk`, as competências por estabelecimento (resumo, pendências agrupadas por correção, fechar e reabrir com step-up, versões do arquivo e download com trilha); a configuração do faturamento do `municipal_admin` com step-up; o papel `billing_clerk` no convite; o tipo `procedure` na linha do tempo do profissional; e os dados do paciente para o BPA-I (raça/cor, etnia, CEP e endereço — contrato §12, D-U1) na recepção (check-in e validação do cadastro) e na pendência do faturista.

**Architecture:** O cliente HTTP ganha a seção "Faturamento de procedimentos" no fim de `src/lib/api.ts` (tipos do contrato §1–§6 e as funções; na URL só ids, códigos, competência e o termo da busca SIGTAP; CPF e textos no corpo). As regras de tela ficam puras em `src/lib/billing.ts` (rótulos dos códigos, veredito da crítica, formulário do procedimento, filtros, pendências agrupadas, nome do arquivo, configuração, frases das recusas). As peças ficam em `src/modules/billing/` (`SigtapSearch`, `ProcedureForm`, `ProcedureVerdict`, `ProcedureTable`, `ProceduresPanel`, `UnitProcedures`, `BillingPatientLookup`, `BillingGate`, `ManualEntryPanel`, `PendingPanel`, `ExportsPanel`) e as três telas em `src/modules/` (`BillingProcedures`, `BillingPeriods`, `BillingSettings`), registradas no grupo novo "Faturamento" (`modules.ts`, `catalog.ts`, `App.tsx`). Mudam o mínimo: `UnitQueue` (botão Procedimentos no modo integrado), `ConsultationWorkspace` (painel na consulta), `Attendance` (avulso), `team.ts` (papel), `timeline.ts`/`PatientTimeline` (tipo novo), `features.ts`, `vite.config.ts` (proxy `/billing`). O dashboard nunca decide o que o api garante: a crítica SIGTAP, o instrumento, o estabelecimento faturante, a competência de apresentação, as pendências e o arquivo são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-21-module-22-procedures-billing-design.md` (§2 decisões, §3 interruptor e papel, §4 dados, §5 fluxo, §6 arquivo, §7 linha do tempo, §8 segurança, §9 testes, §11 F-IDs), `docs/adr/0039.md` e o contrato `docs/superpowers/plans/2026-10-21-module-22-billing-contracts.md` (§1–§6, §9, §10), com as convenções herdadas (21a §15, 21b §14, 21c §11). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim (B1–B13); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): papel `billing_clerk`; origem `attendance` | `consultation` | `lab_report` | `manual`; instrumento `bpa_i` | `bpa_c` | `not_bpa`; caráter `elective` | `urgent`; procedimento `recorded` | `cancelled`; competência `open` | `closed`; estabelecimento `{ type: "health_unit" | "lab_provider", id, cnes, name }`; os 13 motivos de crítica do contrato §1. Código desconhecido aparece cru, nunca some.
- Interruptor (contrato §1; spec §3): `procedure_billing`, sem pré-requisito de modo. Desligado, o dashboard é exatamente o de hoje: sem o grupo Faturamento, sem Procedimentos no atendimento e na consulta, sem avulso, sem o filtro Procedimento na linha do tempo. 403 `feature_disabled` → `BILLING_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo em `features.ts`.
- **A crítica marca e não bloqueia** (ADR 0039, invariante): o formulário só recusa o que é formato (procedimento não escolhido, quantidade fora de 1..999, CID mal formado, data no futuro); CID ausente num procedimento que exige **não** impede o envio — o 201 volta "não faturável — o procedimento exige CID-10".
- **UBS nunca fatura BPA** (ADR 0039, invariante; spec §2 decisão 2): em unidade `kind: "ubs"` o dashboard não oferece Procedimentos no atendimento, na consulta nem o avulso (`unitBills`). O api, se receber, grava com `establishment_is_aps`.
- **Nenhum polling no faturamento.** Lista de procedimentos, competências, pendências (trilha: contêm paciente) e versões só leem ao abrir a tela/painel, ao filtrar ou depois de uma ação; nenhum `refetchInterval` nem `refetchOnWindowFocus` nelas; pendências só no clique de "Pendências" ou "Conferir de novo".
- Na URL vão só ids, `establishment_type`, códigos de motivo, `competence` (AAAAMM), `billable`, o termo da busca SIGTAP e o cursor — nunca CPF, CNS, nome, motivo de cancelamento, motivo de reabertura ou CNPJ. CPF da busca de paciente vai no corpo de um POST.
- CPF, CNS, nome de paciente e o conteúdo do arquivo nunca em `console`, `localStorage`, `sessionStorage`, Analytics ou mensagem de erro. O arquivo baixado vai direto ao `saveFile` (link temporário revogado em 60 s) com nome sem dado pessoal (`bpa-<cnes>-<competência>-v<versão>.txt` ou o seguro do api).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; `POST …/close` manda `{}`). Fechar e reabrir competência, salvar a configuração e convidar faturista com step-up pelo `SensitiveAction`.
- Recusar em vez de cortar: nenhum `maxLength` silencioso; motivo de cancelamento e de reabertura de 10 a 500 caracteres; nome do órgão de origem até 30, sigla até 6, órgão de destino até 40 (larguras `cbc-rsp`, `cbc-sgl`, `cbc-dst` do layout do DATASUS de 09/07/2026) — mais longo é recusado com a frase do limite.
- Quem pode (spec §3; contrato §3–§6): profissional (`health_professional` com vínculo na unidade) registra e cancela os **próprios** procedimentos no atendimento, na consulta e avulso; `billing_clerk` lê tudo, lança manualmente, cancela qualquer um, fecha, reabre e baixa; `municipal_admin` lê procedimentos, competências, pendências e versões (sem baixar) e edita a configuração. A tela não oferece o que o papel não pode; o api recusa de novo (`missing_role`).
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem `outline: "none"`, sem `textOverflow` sem `title`, sem cor literal (guardas `src/theme/a11y-guards.test.ts`, `usability-guards.test.ts`, `literals.test.ts`, `css-guards.test.ts`). Use os componentes existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `ActionButton`, `SensitiveAction`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `saveFile` de `src/modules/lab/saveFile.ts`); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- **Dados do paciente para o BPA-I** (contrato §12, D-U1): raça/cor da lista do layout **sem** "sem informação" (01–05), etnia (4 dígitos) só e obrigatória para indígena, CEP, logradouro, número, complemento e bairro, com as larguras do layout 05.00 (30/5/10/30) recusadas, nunca cortadas. Na recepção são **opcionais e incentivados** (nota "ajuda o faturamento"), nunca bloqueiam o check-in nem a validação; só o preenchido vai no corpo. O faturista completa pela pendência (`PATCH /billing/patients/:id/demographics`) e a lista de pendências é relida.
- "Hoje" é o dia da cidade (`todayInCity()`); as telas e painéis recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY22`, sem relógio falso.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-22.1..F-22.7** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-22.1: api, maintenance, dashboard; F-22.2, F-22.4, F-22.5: api, dashboard; F-22.6: api, contracts, dashboard; F-22.3 e F-22.7 são só do api) e a **Verified** depois da prova no navegador (Task 17).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod22` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-22-billing` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod22`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos antes de cada commit: `npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. O jsdom não tem `URL.createObjectURL`: defina-o no teste que baixa arquivo. Sessão de teste com `mfa_verified_at` de agora (`sessionWith`) = janela de step-up aberta (o `SensitiveAction` não pede código); com `mfa_verified_at: null`, pede.
- A prova (Task 17) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5197**, com proxy para o api do 22 na **3044**.

## Review Focus

1. **Procedimento que a SIGTAP só fatura com CID, registrado sem CID (ou que falha em qualquer crítica).** Espera-se: o formulário avisa e **envia** (nada bloqueia o cuidado); o 201 volta "não faturável" e a tela diz o motivo em português ("não faturável — o procedimento exige CID-10"), mantém o item na lista e diz que o fechamento confere de novo. Testes: Task 5, "sem CID num procedimento que exige: avisa, mas envia — quem decide é o api"; Task 6, "lista do atendimento e registro com o id da consulta; 201 não faturável diz o motivo e relê".
2. **Profissional na UBS com o interruptor ligado.** Espera-se: nada de Procedimentos no atendimento, na consulta ou avulso — a produção da APS segue pelo e-SUS, sem duplicar (antiobjetivo da spec §1); na UPA, aparece. Testes: Task 2, "unitBills e showUnitProcedures: UBS nunca, UPA sim"; Task 7, "UBS: sem o botão (a produção da APS vai pelo e-SUS)" e "UBS ou sem o interruptor: sem o painel".
3. **Fechar com a janela de step-up vencida e a cidade sem configuração.** Espera-se: o código do autenticador é pedido; 422 `settings_missing` diz o que falta e onde a administração preenche; nada muda na lista (só relê depois de sucesso). Teste: Task 12, "janela fechada pede o código; 422 settings_missing aponta a configuração e nada muda".
4. **Download do arquivo.** Espera-se: só o `billing_clerk` vê "Baixar"; o admin vê as versões sem o botão; o nome salvo é o seguro do api ou `bpa-<cnes>-<competência>-v<versão>.txt` (sem nome de paciente); a tela diz que o download fica na trilha. Teste: Task 12, "versões: o faturista baixa com o nome do api ou o de reserva; o admin vê sem baixar".
5. **Pendências.** Espera-se: lidas só no clique (a rota grava trilha), agrupadas por correção na ordem profissional → paciente → estabelecimento → procedimento, motivo ou correção que o api inventar depois aparece cru, e "Conferir de novo" relê (corrigido o cadastro, a pendência sai sem relançar). Teste: Task 11, "pendências: só no clique, agrupadas por correção, motivo novo cru; Conferir de novo relê".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras | 0 |
| `src/lib/api.ts`, `src/lib/api.billing.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/billingFixtures.ts` | tipos e cliente HTTP do 22; interruptor; proxy `/billing`; dados de teste | 1 |
| `src/lib/billing.ts`, `src/lib/billing.test.ts` | regras de tela do faturamento | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.billing.test.ts`, `src/shell/catalog.ts`, `src/lib/team.ts`, `src/lib/team.test.ts` | grupo Faturamento, catálogo, papel `billing_clerk` no convite | 3 |
| `src/lib/api.ts` (`TimelineKind`), `src/lib/timeline.ts`, `src/lib/timeline.test.ts`, `src/lib/timeline.exam.test.ts`, `src/lib/timeline.procedure.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/timeline/PatientTimeline.procedure.test.tsx` | tipo `procedure` na linha do tempo | 4 |
| `src/modules/billing/SigtapSearch.tsx`, `src/modules/billing/ProcedureVerdict.tsx`, `src/modules/billing/ProcedureForm.tsx`, `src/modules/billing/ProcedureForm.test.tsx` | busca SIGTAP, formulário, veredito da crítica | 5 |
| `src/modules/billing/ProcedureTable.tsx`, `src/modules/billing/ProceduresPanel.tsx`, `src/modules/billing/ProceduresPanel.test.tsx` | lista com cancelamento; painel do atendimento | 6 |
| `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/UnitQueue.procedures.test.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.procedures.test.tsx` | painel no atendimento (integrado) e na consulta | 7 |
| `src/modules/billing/BillingPatientLookup.tsx`, `src/modules/billing/UnitProcedures.tsx`, `src/modules/billing/UnitProcedures.test.tsx`, `src/modules/Attendance.tsx` | procedimento avulso pela unidade | 8 |
| `src/modules/billing/BillingGate.tsx`, `src/modules/BillingProcedures.tsx`, `src/modules/BillingProcedures.test.tsx`, `src/App.tsx` | lista de procedimentos com filtros | 9 |
| `src/modules/billing/ManualEntryPanel.tsx`, `src/modules/billing/ManualEntryPanel.test.tsx`, `src/modules/BillingProcedures.tsx` | lançamento manual | 10 |
| `src/modules/BillingPeriods.tsx`, `src/modules/billing/PendingPanel.tsx`, `src/modules/BillingPeriods.test.tsx`, `src/App.tsx` | competências, resumo e pendências | 11 |
| `src/modules/billing/ExportsPanel.tsx`, `src/modules/BillingPeriods.tsx`, `src/modules/BillingPeriods.close.test.tsx` | fechar e reabrir com step-up; versões e download | 12 |
| `src/modules/BillingSettings.tsx`, `src/modules/BillingSettings.test.tsx`, `src/App.tsx` | configuração do faturamento | 13 |
| `src/lib/demographics.ts`, `src/lib/demographics.test.ts`, `src/lib/api.ts`, `src/lib/api.billing.test.ts`, `src/modules/attendance/DemographicsForm.tsx`, `src/modules/attendance/DemographicsForm.test.tsx` | dados do paciente para o BPA-I: regras, cliente e formulário comum | 14 |
| `src/modules/attendance/CheckIn.tsx`, `src/modules/attendance/CheckIn.demographics.test.tsx`, `src/modules/Attendance.tsx`, `src/modules/Attendance.demographics.test.tsx` | recepção: check-in e validação do cadastro | 15 |
| `src/modules/billing/PendingPanel.tsx`, `src/modules/BillingPeriods.tsx`, `src/lib/billing.ts`, `src/lib/billing.test.ts`, `src/modules/BillingPeriods.demographics.test.tsx` | faturista completa os dados pela pendência | 16 |
| — | suíte, build, revisão, prova no navegador | 17 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, motivos, CNPJ) (Task 1); regras puras com tabela de casos: veredito, formulário (o que recusa e o que **não** recusa), filtros, pendências agrupadas, configuração com as larguras do layout, frases das recusas (Task 2); menu por papel e interruptor e o papel no convite (Task 3); a linha do tempo com e sem o interruptor (Task 4); cada peça e tela pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel e interruptor, cobrindo o caminho feliz, cada recusa que muda a tela (422 de formato, 409 `already_exported`, 422 `settings_missing`/`nothing_to_export`, `feature_disabled`, `missing_role`), UBS e o papel sem acesso (Tasks 5–13). A prova final é no navegador, contra o api real na 3044 (Task 17).

---

### Task 0: Board, worktree, base e âncoras

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`6b9d0c3` ou mais novo); o board #1 (`rotasaude`).
- Produces: cards F-22.1..F-22.7 em In Progress; worktree `$W` na branch `feat/mod-22-billing`; a base anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 400 --format json --jq '.items[] | select(.title | test("^F-22\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (o plano do api move os mesmos cards), com o `id` do item e o id da opção **In Progress**:

```bash
gh project item-edit --id PVTI_xxx --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os sete cards em In Progress ou além. Se algum F-22.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "municipal_lab" | "lab_hl7_inbound" | "fake_middleware" | "lab_quality_control" | "lab_inventory";\|  lab_inventory: "Estoque do laboratório"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const withQuery = \|^async function binaryFetch\|^export interface SignatureFile\|  | "clinical_document" | "controlled_notification" | "campaign" | "exam";\|^export function fetchConsumptionCsv'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "lab-admin" | "lab-reports";\|    { id: "lab-reports", label: "Relatórios do laboratório", icon: "▤" }\|  const invOn = labOn && hasFeature(user, "lab_inventory");\|      if (item.id === "integrations" || item.id === "cnes") return canIntegrations;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 9\|NAV_GROUPS.length - 8'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '  IconUser, IconUserCircle, IconUsers, IconUsersGroup\|    "Laboratório": { icon: IconFlask\|    "lab-reports": { icon: IconChartBar'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '  { role: "lab_releaser", label: "Liberador de laudos" },\|"exam_regulator", "lab_staff", "lab_releaser" \]);'
$G -C $R/apps/dashboard show origin/main:src/lib/team.test.ts | grep -n 'oferece os 12 papéis\|"analyst", "campaign_manager", "citizen_verifier", "exam_regulator", "health_professional", "lab_releaser", "lab_staff",'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '  { key: "exam", label: "Exame" }\|^export function timelineKinds\|  external: "resultado externo"'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n 'const kinds = timelineKinds(hasFeature(user, "exam_tracking"));'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitQueue.tsx | grep -n '  | { kind: "timeline"; attendanceId: string };\|  const timelineButton = canCare && patientTimeline && !clinicalRecord;\|  patientTimeline = false\|        {timelineButton && panel?.kind === "timeline" && ('
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n '  const examsOn = hasFeature(user, "exam_tracking");\|                onClose={props.onClose} />'
$G -C $R/apps/dashboard show origin/main:src/modules/Attendance.tsx | grep -n '          patientTimeline={hasFeature(user, "patient_timeline")}\|      {canVerify && unit && <div id="sec-pedidos">'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'case "lab-instruments": return <LabInstruments\|^import { LabQcBlocks }'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/v(/|\$)": proxy(TARGET)'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod22 -b feat/mod-22-billing origin/main
cd $R/.worktrees/dashboard-mod22 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 17 espera a base mais **20 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 22** (só informativo; as Tasks 1–16 não dependem dele)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-22-billing
docker exec -w /rails/.claude/mod22 api-dev bin/rails routes 2>/dev/null | grep -E " /billing/" | head -40
```

Expected: a branch e as rotas. Compare com o contrato e com as Divergências deste plano (B1–B13); diferença vira nota para a Task 17 — e, se for de forma (envelope, nome de campo), ajuste `src/lib/api.ts` e `src/test/billingFixtures.ts` antes da prova.

---

### Task 1: Tipos, cliente HTTP, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/billingFixtures.ts`
- Test: `src/lib/api.billing.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `binaryFetch`, `withQuery`, `SignatureFile`, `HealthUnit`, `SessionUser` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`): os tipos `ProcedureSource`, `BillingInstrument`, `ProcedureCharacter`, `ProcedureStatus`, `BillingPeriodStatus`, `EstablishmentType`, `PendingFix`, `BillingEstablishment`, `PerformedProcedure`, `ProcedurePage`, `ProcedureInput`, `ProcedureQuery`, `SigtapProcedure`, `BillingPatient`, `BillingProfessional`, `BillingSummary`, `BillingPeriod`, `PendingItem`, `BillingExport`, `BillingSettings`, `BillingSettingsInput`; a constante `BILLING_PAGE_SIZE = 50`; as funções `searchBillingSigtap(q, competence): Promise<SigtapProcedure[]>`, `recordProcedure(input): Promise<PerformedProcedure>`, `listProcedures(q): Promise<ProcedurePage>`, `cancelProcedure(id, reason): Promise<PerformedProcedure>`, `lookupBillingPatient(cpf): Promise<BillingPatient>`, `listBillingEstablishments(): Promise<BillingEstablishment[]>`, `listBillingProfessionals(est): Promise<BillingProfessional[]>`, `listBillingPeriods(competence): Promise<BillingPeriod[]>`, `getPeriodPending(id): Promise<PendingItem[]>`, `closeBillingPeriod(id): Promise<void>`, `reopenBillingPeriod(id, reason): Promise<void>`, `listPeriodExports(id): Promise<BillingExport[]>`, `fetchBillingExportFile(id): Promise<SignatureFile>`, `getBillingSettings(): Promise<BillingSettings>`, `saveBillingSettings(input): Promise<BillingSettings>`. `FeatureKey` ganha `procedure_billing`.
- Produces (em `src/test/billingFixtures.ts`): `NOW22`, `TODAY22`, `UPA`, `UBS`, `UPA_EST`, `LAB_EST`, `billingUser(roles, over)`, `URGENCY`, `SUTURE`, `GLUCOSE`, `procedure(over)`, `period(over)`, `pending(over)`, `billingExport(over)`, `PROFESSIONALS`, `PATIENT`, `SETTINGS`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.billing.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  cancelProcedure, closeBillingPeriod, fetchBillingExportFile, getBillingSettings, getPeriodPending, listBillingEstablishments,
  listBillingPeriods, listBillingProfessionals, listPeriodExports, listProcedures, lookupBillingPatient, recordProcedure,
  reopenBillingPeriod, saveBillingSettings, searchBillingSigtap
} from "./api";

function ok(body: unknown, init: ResponseInit = {}): Promise<Response> {
  return Promise.resolve(new Response(typeof body === "string" ? body : JSON.stringify(body), {
    status: 200, headers: { "Content-Type": "application/json" }, ...init
  }));
}

describe("cliente do faturamento (contrato §3–§6)", () => {
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

  it("busca SIGTAP: termo e competência na URL; lista em procedures", async () => {
    fetchMock.mockImplementationOnce(() => ok({ procedures: [ { code: "0301060061" } ] }));
    expect(await searchBillingSigtap("urgencia", "202610")).toEqual([ { code: "0301060061" } ]);
    expect(urlOf().pathname).toBe("/billing/sigtap/search");
    expect(Object.fromEntries(urlOf().searchParams)).toEqual({ q: "urgencia", competence: "202610" });
  });

  it("registrar: tudo no corpo, com JSON; nada na URL", async () => {
    const input = {
      patient_id: "pa1", establishment: { type: "health_unit" as const, id: "u7" }, performed_on: "2026-10-21",
      sigtap_code: "0401010066", quantity: 2, cid10: "S610", character: "urgent" as const, attendance_id: "a1"
    };
    await recordProcedure(input);
    expect(urlOf().pathname).toBe("/billing/procedures");
    expect(urlOf().search).toBe("");
    expect(initOf().method).toBe("POST");
    expect(headersOf()["Content-Type"]).toBe("application/json");
    expect(bodyOf()).toEqual(input);
  });

  it("lista: só ids, códigos e competência na URL; envelope com next_before", async () => {
    fetchMock.mockImplementationOnce(() => ok({ procedures: [], next_before: "c2" }));
    expect(await listProcedures({
      establishmentType: "lab_provider", establishmentId: "pv9", competence: "202609", billable: false,
      reason: "cbo_not_allowed", patientId: "pa1", before: "c1"
    })).toEqual({ procedures: [], next_before: "c2" });
    expect(urlOf().pathname).toBe("/billing/procedures");
    expect(Object.fromEntries(urlOf().searchParams)).toEqual({
      establishment_type: "lab_provider", establishment_id: "pv9", competence: "202609", billable: "false",
      reason: "cbo_not_allowed", patient_id: "pa1", before: "c1", limit: "50"
    });
    fetchMock.mockImplementationOnce(() => ok({}));
    expect(await listProcedures({ attendanceId: "a1" })).toEqual({ procedures: [], next_before: null });
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ attendance_id: "a1", limit: "50" });
  });

  it("cancelar e reabrir: motivo no corpo; fechar manda {} com JSON", async () => {
    await cancelProcedure("pp1", "lançado no paciente errado");
    expect(urlOf().pathname).toBe("/billing/procedures/pp1/cancel");
    expect(urlOf().search).toBe("");
    expect(bodyOf()).toEqual({ reason: "lançado no paciente errado" });
    await closeBillingPeriod("bp1");
    expect(urlOf(1).pathname).toBe("/billing/periods/bp1/close");
    expect(initOf(1).method).toBe("POST");
    expect(bodyOf(1)).toEqual({});
    expect(headersOf(1)["Content-Type"]).toBe("application/json");
    await reopenBillingPeriod("bp1", "CNS do profissional corrigido");
    expect(urlOf(2).pathname).toBe("/billing/periods/bp1/reopen");
    expect(urlOf(2).search).toBe("");
    expect(bodyOf(2)).toEqual({ reason: "CNS do profissional corrigido" });
  });

  it("paciente pelo CPF no corpo de um POST, nunca na URL", async () => {
    fetchMock.mockImplementationOnce(() => ok({ patient: { id: "pa1", display_name: "Joana Lima", cpf_masked: "***.982.247-**" } }));
    expect((await lookupBillingPatient("52998224725")).id).toBe("pa1");
    expect(urlOf().pathname).toBe("/billing/patient_lookup");
    expect(urlOf().search).toBe("");
    expect(initOf().method).toBe("POST");
    expect(bodyOf()).toEqual({ cpf: "52998224725" });
  });

  it("estabelecimentos, profissionais, competências, pendências e versões: envelopes", async () => {
    fetchMock
      .mockImplementationOnce(() => ok({ establishments: [ { id: "u7" } ] }))
      .mockImplementationOnce(() => ok({ professionals: [ { user_id: "us1" } ] }))
      .mockImplementationOnce(() => ok({ periods: [ { id: "bp1" } ] }))
      .mockImplementationOnce(() => ok({ items: [ { procedure_id: "pp1" } ] }))
      .mockImplementationOnce(() => ok({ exports: [ { id: "ex1" } ] }));
    expect(await listBillingEstablishments()).toEqual([ { id: "u7" } ]);
    expect(urlOf(0).pathname).toBe("/billing/establishments");
    expect(await listBillingProfessionals({ type: "health_unit", id: "u7" })).toEqual([ { user_id: "us1" } ]);
    expect(urlOf(1).pathname).toBe("/billing/professionals");
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ establishment_type: "health_unit", establishment_id: "u7" });
    expect(await listBillingPeriods("202609")).toEqual([ { id: "bp1" } ]);
    expect(urlOf(2).pathname).toBe("/billing/periods");
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ competence: "202609" });
    expect(await getPeriodPending("bp1")).toEqual([ { procedure_id: "pp1" } ]);
    expect(urlOf(3).pathname).toBe("/billing/periods/bp1/pending");
    expect(await listPeriodExports("bp1")).toEqual([ { id: "ex1" } ]);
    expect(urlOf(4).pathname).toBe("/billing/periods/bp1/exports");
  });

  it("envelope ausente vira lista vazia", async () => {
    expect(await listBillingEstablishments()).toEqual([]);
    expect(await listBillingPeriods("202609")).toEqual([]);
    expect(await getPeriodPending("bp1")).toEqual([]);
    expect(await listPeriodExports("bp1")).toEqual([]);
  });

  it("arquivo: text/plain lido com fetch, com o nome seguro do api", async () => {
    fetchMock.mockImplementationOnce(() => Promise.resolve(new Response("01#BPA#202609\r\n", {
      status: 200, headers: { "Content-Type": "text/plain", "Content-Disposition": "attachment; filename=\"bpa-2384299-202609-v1.txt\"" }
    })));
    const file = await fetchBillingExportFile("ex1");
    expect(urlOf().pathname).toBe("/billing/exports/ex1/file");
    expect((initOf().headers as Record<string, string>).Accept).toBe("text/plain");
    expect(file.filename).toBe("bpa-2384299-202609-v1.txt");
    expect(file.blob.size).toBe(15);
  });

  it("configuração: GET normaliza ausentes e gestão estranha para null; PUT com JSON", async () => {
    fetchMock.mockImplementationOnce(() => ok({ management: "X" }));
    expect(await getBillingSettings()).toEqual({
      origin_name: null, origin_acronym: null, origin_cnpj: null, destination_body: null, management: null
    });
    const input = {
      origin_name: "SMS CURITIBA", origin_acronym: "SMS", origin_cnpj: "11222333000181",
      destination_body: "SECRETARIA MUNICIPAL DE SAUDE", management: "M" as const
    };
    await saveBillingSettings(input);
    expect(urlOf(1).pathname).toBe("/billing/settings");
    expect(initOf(1).method).toBe("PUT");
    expect(headersOf(1)["Content-Type"]).toBe("application/json");
    expect(bodyOf(1)).toEqual(input);
  });
});
```

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 22", () => {
  it("procedure_billing tem nome e vem da sessão", () => {
    expect(featureLabel("procedure_billing")).toBe("Faturamento de procedimentos (BPA)");
    expect(hasFeature({ features: [ "procedure_billing" ] }, "procedure_billing")).toBe(true);
    expect(hasFeature({ features: [ "municipal_lab" ] }, "procedure_billing")).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/api.billing.test.ts src/lib/features.test.ts`
Expected: FAIL — `searchBillingSigtap` (e as outras) não existem; `featureLabel("procedure_billing")` devolve a chave crua.

- [ ] **Step 3: Write minimal implementation**

No fim de `src/lib/api.ts`:

```ts
// ─── Faturamento de procedimentos (módulo 22, ADR 0039; contrato 2026-10-21) ───
// Na URL só ids, tipo do estabelecimento, códigos (SIGTAP, motivo), competência
// AAAAMM, o termo da busca SIGTAP e o cursor — nunca CPF, CNS, nome, motivo de
// cancelamento ou de reabertura, nem CNPJ. CPF da busca e textos vão no corpo.
// A crítica, o instrumento, as pendências e o arquivo são do api.

const BILLING_BASE = import.meta.env.VITE_BILLING_BASE || "/billing";
const billingPath = (...parts: string[]) => [ BILLING_BASE, ...parts.map(encodeURIComponent) ].join("/");
const billingPost = (body: unknown): RequestInit => ({ method: "POST", body: JSON.stringify(body) });

export type ProcedureSource = "attendance" | "consultation" | "lab_report" | "manual";
export type BillingInstrument = "bpa_i" | "bpa_c" | "not_bpa";
export type ProcedureCharacter = "elective" | "urgent";
export type ProcedureStatus = "recorded" | "cancelled";
export type BillingPeriodStatus = "open" | "closed";
export type EstablishmentType = "health_unit" | "lab_provider";
export type PendingFix = "professional" | "patient" | "establishment" | "procedure";

export interface BillingEstablishment { type: EstablishmentType; id: string; cnes: string | null; name: string }

export interface PerformedProcedure {
  id: string;
  patient: { id: string; display_name: string };
  establishment: BillingEstablishment;
  professional: { user_id: string; name: string; cbo_code: string | null };
  performed_on: string; sigtap_code: string; label: string; sigtap_competence: string;
  quantity: number; cid10: string | null; character: ProcedureCharacter | string;
  source: ProcedureSource | string; source_ref: string | null;
  instrument: BillingInstrument | string; billable: boolean; blocking_reasons: string[];
  status: ProcedureStatus | string; cancel_reason: string | null;
  billing_competence: string | null; sigtap_fallback: boolean;
  // Divergência B7: apresentação em atraso (entrou na competência aberta seguinte). Ausente = false.
  late?: boolean;
}
export interface ProcedurePage { procedures: PerformedProcedure[]; next_before: string | null }

// Divergências B1 (patient_id opcional com atendimento ou consulta) e B5
// (professional_user_id e cbo_code só no lançamento manual do faturista).
export interface ProcedureInput {
  patient_id?: string; establishment: { type: EstablishmentType; id: string };
  performed_on: string; sigtap_code: string; quantity: number; cid10?: string; character?: ProcedureCharacter;
  attendance_id?: string; consultation_id?: string; professional_user_id?: string; cbo_code?: string;
}
// Divergência B2: `attendanceId` lista os procedimentos do atendimento.
export interface ProcedureQuery {
  establishmentType?: EstablishmentType; establishmentId?: string; competence?: string; billable?: boolean;
  reason?: string; patientId?: string; attendanceId?: string; before?: string | null;
}
export interface SigtapProcedure {
  code: string; name: string; instruments: string[]; requires_cid: boolean;
  sex: string | null; age_min_months: number | null; age_max_months: number | null;
}
// Divergência B3.
export interface BillingPatient { id: string; display_name: string; cpf_masked: string }
// Divergência B5.
export interface BillingProfessional { user_id: string; name: string; cbo_codes: string[] }
export interface BillingSummary { bpa_i_lines: number; bpa_c_lines: number; not_billable: Record<string, number>; pending: number }
// `version` = versão do último arquivo gerado; 0 antes do primeiro fechamento (Divergência B11).
export interface BillingPeriod {
  id: string; establishment: BillingEstablishment; competence: string; status: BillingPeriodStatus | string; version: number;
  closed_at: string | null; closed_by: string | null; reopened_reason: string | null; summary: BillingSummary;
}
// Divergência B6: o que a tela mostra de cada pendência (opcional em api sem a B6).
export interface PendingItem {
  procedure_id: string; reasons: string[]; fix: PendingFix | string;
  performed_on?: string; sigtap_code?: string; label?: string;
  patient?: { id: string; display_name: string }; professional?: { user_id: string; name: string };
}
export interface BillingExport {
  id: string; version: number; sha256: string; lines_bpa_i: number; lines_bpa_c: number; generated_at: string; generated_by: string;
}
// Divergência B10: `origin_acronym` (sigla do órgão de origem, cbc-sgl do layout).
export interface BillingSettings {
  origin_name: string | null; origin_acronym: string | null; origin_cnpj: string | null;
  destination_body: string | null; management: "M" | "E" | null;
}
export interface BillingSettingsInput {
  origin_name: string; origin_acronym: string; origin_cnpj: string; destination_body: string; management: "M" | "E";
}

export const BILLING_PAGE_SIZE = 50;

function procedureParams(q: ProcedureQuery): URLSearchParams {
  const p = new URLSearchParams();
  if (q.establishmentType && q.establishmentId) {
    p.set("establishment_type", q.establishmentType);
    p.set("establishment_id", q.establishmentId);
  }
  if (q.competence) p.set("competence", q.competence);
  if (q.billable !== undefined) p.set("billable", String(q.billable));
  if (q.reason) p.set("reason", q.reason);
  if (q.patientId) p.set("patient_id", q.patientId);
  if (q.attendanceId) p.set("attendance_id", q.attendanceId);
  if (q.before) p.set("before", q.before);
  p.set("limit", String(BILLING_PAGE_SIZE));
  return p;
}

export async function searchBillingSigtap(q: string, competence: string): Promise<SigtapProcedure[]> {
  const body = await jsonFetch<{ procedures?: SigtapProcedure[] }>(withQuery(billingPath("sigtap", "search"), new URLSearchParams({ q, competence })));
  return body.procedures ?? [];
}

// Sempre grava (201); a crítica vem nos campos do procedimento.
export function recordProcedure(input: ProcedureInput): Promise<PerformedProcedure> {
  return jsonFetch(billingPath("procedures"), billingPost(input));
}

// A lista traz paciente: o api grava trilha; nunca é relida sozinha.
export async function listProcedures(q: ProcedureQuery): Promise<ProcedurePage> {
  const body = await jsonFetch<Partial<ProcedurePage>>(withQuery(billingPath("procedures"), procedureParams(q)));
  return { procedures: body.procedures ?? [], next_before: body.next_before ?? null };
}

export function cancelProcedure(id: string, reason: string): Promise<PerformedProcedure> {
  return jsonFetch(billingPath("procedures", id, "cancel"), billingPost({ reason }));
}

export async function lookupBillingPatient(cpf: string): Promise<BillingPatient> {
  return (await jsonFetch<{ patient: BillingPatient }>(billingPath("patient_lookup"), billingPost({ cpf }))).patient;
}

export async function listBillingEstablishments(): Promise<BillingEstablishment[]> {
  return (await jsonFetch<{ establishments?: BillingEstablishment[] }>(billingPath("establishments"))).establishments ?? [];
}

export async function listBillingProfessionals(est: { type: EstablishmentType; id: string }): Promise<BillingProfessional[]> {
  const p = new URLSearchParams({ establishment_type: est.type, establishment_id: est.id });
  return (await jsonFetch<{ professionals?: BillingProfessional[] }>(withQuery(billingPath("professionals"), p))).professionals ?? [];
}

export async function listBillingPeriods(competence: string): Promise<BillingPeriod[]> {
  return (await jsonFetch<{ periods?: BillingPeriod[] }>(withQuery(billingPath("periods"), new URLSearchParams({ competence })))).periods ?? [];
}

// Grava trilha (contém paciente): só no clique.
export async function getPeriodPending(id: string): Promise<PendingItem[]> {
  return (await jsonFetch<{ items?: PendingItem[] }>(billingPath("periods", id, "pending"))).items ?? [];
}

// Step-up: quem trata 401 mfa_required é o SensitiveAction.
export async function closeBillingPeriod(id: string): Promise<void> {
  await jsonFetch<unknown>(billingPath("periods", id, "close"), billingPost({}));
}

export async function reopenBillingPeriod(id: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(billingPath("periods", id, "reopen"), billingPost({ reason }));
}

export async function listPeriodExports(id: string): Promise<BillingExport[]> {
  return (await jsonFetch<{ exports?: BillingExport[] }>(billingPath("periods", id, "exports"))).exports ?? [];
}

// Só billing_clerk; cada download vira `billing_export.downloaded` na trilha.
export function fetchBillingExportFile(id: string): Promise<SignatureFile> {
  return binaryFetch(billingPath("exports", id, "file"), "text/plain");
}

export async function getBillingSettings(): Promise<BillingSettings> {
  const b = await jsonFetch<Partial<BillingSettings>>(billingPath("settings"));
  return {
    origin_name: b.origin_name ?? null, origin_acronym: b.origin_acronym ?? null, origin_cnpj: b.origin_cnpj ?? null,
    destination_body: b.destination_body ?? null, management: b.management === "M" || b.management === "E" ? b.management : null
  };
}

// Step-up (contrato §6): quem trata 401 mfa_required é o SensitiveAction.
export function saveBillingSettings(input: BillingSettingsInput): Promise<BillingSettings> {
  return jsonFetch(billingPath("settings"), { method: "PUT", body: JSON.stringify(input) });
}
```

Em `src/lib/features.ts`, troque

```ts
  | "municipal_lab" | "lab_hl7_inbound" | "fake_middleware" | "lab_quality_control" | "lab_inventory";
```

por

```ts
  | "municipal_lab" | "lab_hl7_inbound" | "fake_middleware" | "lab_quality_control" | "lab_inventory"
  | "procedure_billing";
```

e, em `FEATURE_LABEL`, troque

```ts
  lab_inventory: "Estoque do laboratório"
};
```

por

```ts
  lab_inventory: "Estoque do laboratório",
  // Módulo 22 (ADR 0039): sem pré-requisito de modo de prontuário.
  procedure_billing: "Faturamento de procedimentos (BPA)"
};
```

Em `vite.config.ts`, no comentário do topo, depois da linha de `/lab`, acrescente

```ts
//   /billing    → faturamento de procedimentos (módulo 22: registro, críticas SIGTAP, competências e arquivo BPA).
```

e, em `proxy`, troque

```ts
      "^/v(/|$)": proxy(TARGET)
```

por

```ts
      "^/v(/|$)": proxy(TARGET),
      // Módulo 22: faturamento de procedimentos (BPA).
      "/billing": proxy(TARGET)
```

Crie `src/test/billingFixtures.ts`:

```ts
// src/test/billingFixtures.ts
// Dados comuns aos testes do módulo 22. Cada arquivo de teste faz o próprio
// vi.mock("…/lib/api"); aqui só tipos e a sessão. Códigos SIGTAP, CBO e CNES
// com cara de real (semente realista), nomes fictícios.
import type {
  BillingEstablishment, BillingExport, BillingPatient, BillingPeriod, BillingProfessional, BillingSettings, HealthUnit, PendingItem,
  PerformedProcedure, SessionUser, SigtapProcedure
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW22 = "2026-10-21T10:00:00-03:00";
export const TODAY22 = "2026-10-21";
export const UPA: HealthUnit = { id: "u7", name: "UPA Boa Vista", kind: "upa" };
export const UBS: HealthUnit = { id: "u1", name: "UBS Centro", kind: "ubs" };
export const UPA_EST: BillingEstablishment = { type: "health_unit", id: "u7", cnes: "2384299", name: "UPA Boa Vista" };
export const LAB_EST: BillingEstablishment = { type: "lab_provider", id: "pv9", cnes: "7654321", name: "Laboratório Municipal" };
export const PATIENT: BillingPatient = { id: "pa1", display_name: "Joana Lima", cpf_masked: "***.982.247-**" };

// Sessão com o faturamento ligado; `mfa_verified_at` = agora (janela de step-up aberta).
export function billingUser(roles: string[] = [ "billing_clerk" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { id: "us22", email_address: "faturamento@curitiba.demo", features: [ "procedure_billing" ], ...over });
}

const sigtap = (code: string, name: string, instruments: string[], requiresCid = false): SigtapProcedure => ({
  code, name, instruments, requires_cid: requiresCid, sex: null, age_min_months: null, age_max_months: null
});
export const URGENCY = sigtap("0301060061", "ATENDIMENTO DE URGENCIA EM ATENCAO ESPECIALIZADA", [ "bpa_i" ]);
export const SUTURE = sigtap("0401010066", "EXCISAO E/OU SUTURA SIMPLES DE PEQUENAS LESOES/FERIMENTOS DE PELE/ANEXOS E MUCOSA", [ "bpa_i" ], true);
export const GLUCOSE = sigtap("0202010473", "DOSAGEM DE GLICOSE", [ "bpa_c" ]);

export function procedure(over: Partial<PerformedProcedure> = {}): PerformedProcedure {
  return {
    id: "pp1", patient: { id: "pa1", display_name: "Joana Lima" }, establishment: UPA_EST,
    professional: { user_id: "us1", name: "Dra. Helena Prado", cbo_code: "225125" }, performed_on: TODAY22,
    sigtap_code: URGENCY.code, label: URGENCY.name, sigtap_competence: "202610", quantity: 1, cid10: null, character: "urgent",
    source: "attendance", source_ref: "a1", instrument: "bpa_i", billable: true, blocking_reasons: [], status: "recorded",
    cancel_reason: null, billing_competence: null, sigtap_fallback: false, ...over
  };
}

export function period(over: Partial<BillingPeriod> = {}): BillingPeriod {
  return {
    id: "bp1", establishment: UPA_EST, competence: "202609", status: "open", version: 0, closed_at: null, closed_by: null,
    reopened_reason: null, summary: { bpa_i_lines: 12, bpa_c_lines: 3, not_billable: { cbo_not_allowed: 2, missing_professional_cns: 1 }, pending: 1 },
    ...over
  };
}

export function pending(over: Partial<PendingItem> = {}): PendingItem {
  return {
    procedure_id: "pp2", reasons: [ "missing_professional_cns" ], fix: "professional", performed_on: "2026-09-14",
    sigtap_code: URGENCY.code, label: URGENCY.name, patient: { id: "pa1", display_name: "Joana Lima" },
    professional: { user_id: "us3", name: "Enf. Lúcia Prado" }, ...over
  };
}

export function billingExport(over: Partial<BillingExport> = {}): BillingExport {
  return {
    id: "ex1", version: 1, sha256: "9f2c4e7a1b3d5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8",
    lines_bpa_i: 12, lines_bpa_c: 3, generated_at: "2026-10-05T09:00:00-03:00", generated_by: "Rita Faturista", ...over
  };
}

export const PROFESSIONALS: BillingProfessional[] = [
  { user_id: "us1", name: "Dra. Helena Prado", cbo_codes: [ "225125" ] },
  { user_id: "us3", name: "Enf. Lúcia Prado", cbo_codes: [ "223505", "223565" ] },
  { user_id: "us4", name: "Téc. Caio Lima", cbo_codes: [] }
];

export const SETTINGS: BillingSettings = {
  origin_name: "SMS CURITIBA", origin_acronym: "SMS", origin_cnpj: "11222333000181",
  destination_body: "SECRETARIA MUNICIPAL DE SAUDE", management: "M"
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/api.billing.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.billing.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/billingFixtures.ts
$G commit -m "feat: add the billing API client and the procedure billing switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela do faturamento (`src/lib/billing.ts`)

**Files:**
- Create: `src/lib/billing.ts`
- Test: `src/lib/billing.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `ApiError`, `errorCode` (`api.ts`); `featureDisabledKey`, `hasFeature` (`features.ts`); `cnpjDigits`, `isValidCnpj` (`clinicalDocuments.ts`); `competenceLabel`, `competenceOf`, `recentCompetences` (`competence.ts`).
- Produces (todas exportadas de `src/lib/billing.ts`):
  - constantes: `BILLING_CLERK_ROLE`, `BILLING_DISABLED`, `BILLING_NO_ROLE`, `LATE_TEXT`, `QUANTITY_MAX = 999`, `REASON_MIN = 10`, `REASON_MAX = 500`, `ORIGIN_NAME_MAX = 30`, `ORIGIN_ACRONYM_MAX = 6`, `DESTINATION_MAX = 40`, `REASON_CODES`, `CHARACTER_OPTIONS`, `FIX_ORDER`; chaves de consulta `PROCEDURES_KEY`, `CARE_PROCEDURES_KEY`, `PERIODS_KEY`, `PENDING_KEY`, `EXPORTS_KEY`, `SETTINGS_KEY`, `ESTABLISHMENTS_KEY`, `PROFESSIONALS_KEY`, `SIGTAP_KEY`;
  - `canUseBilling(user): boolean`, `billingRoles(user): BillingRoles` (`{ clerk, admin, professional }`), `unitBills(unit): boolean`, `showUnitProcedures({ user, canCare, unit }): boolean`;
  - rótulos: `reasonLabel(code)`, `sourceLabel(code)`, `instrumentLabel(code)`, `characterLabel(code)`, `fixInfo(fix): { title; hint }`;
  - `verdict(p): { tone: "ok" | "warn" | "neutral"; text }`, `fallbackNotice(p): string | null`, `savedText(p): string`;
  - formulário: `ProcedureDraft`, `emptyProcedure(today)`, `normalizeCid(text)`, `cidHint(d)`, `procedureProblem(d, today)`, `ProcedureContext`, `procedureBody(d, ctx): ProcedureInput`, `sigtapCompetence(d)`;
  - `reasonProblem(text): string | null`;
  - manual: `ManualChoice`, `manualProblem(m): string | null`;
  - estabelecimento e filtros: `establishmentLabel(e)`, `establishmentKey(e)`, `parseEstablishmentKey(key)`, `ProcedureFilters`, `EMPTY_FILTERS(today)`, `procedureQuery(f): ProcedureQuery`;
  - competência: `closingCompetence(today)`, `periodStatusText(p)`, `notBillableText(summary)`, `closeDescription(p)`, `groupPending(items)`, `exportFileName(period, exp)`;
  - configuração: `SettingsDraft`, `settingsDraft(s)`, `settingsProblem(d)`, `settingsBody(d)`, `settingsComplete(s)`;
  - erros: `billingError(err): string`, `isStale(err): boolean`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/billing.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  BILLING_DISABLED, EMPTY_FILTERS, billingError, billingRoles, canUseBilling, cidHint, closeDescription, closingCompetence, emptyProcedure,
  establishmentKey, establishmentLabel, exportFileName, fallbackNotice, fixInfo, groupPending, isStale, manualProblem, normalizeCid,
  notBillableText, parseEstablishmentKey, periodStatusText, procedureBody, procedureProblem, procedureQuery, reasonLabel, reasonProblem,
  savedText, settingsBody, settingsComplete, settingsDraft, settingsProblem, showUnitProcedures, sigtapCompetence, sourceLabel, unitBills,
  verdict
} from "./billing";
import { LAB_EST, SETTINGS, SUTURE, TODAY22, UPA, UBS, UPA_EST, URGENCY, billingExport, billingUser, pending, period, procedure } from "../test/billingFixtures";

const refusal = (status: number, error: string) => new ApiError(status, { error }, String(status));

describe("quem usa", () => {
  it("interruptor na sessão; operador nunca", () => {
    expect(canUseBilling(billingUser())).toBe(true);
    expect(canUseBilling(billingUser([ "billing_clerk" ], { features: [] }))).toBe(false);
    expect(canUseBilling({ ...billingUser(), operator: true })).toBe(false);
    expect(canUseBilling(null)).toBe(false);
  });

  it("papéis", () => {
    expect(billingRoles(billingUser([ "billing_clerk", "municipal_admin" ]))).toEqual({ clerk: true, admin: true, professional: false });
    expect(billingRoles(billingUser([ "health_professional" ]))).toEqual({ clerk: false, admin: false, professional: true });
  });

  it("unitBills e showUnitProcedures: UBS nunca, UPA sim", () => {
    expect(unitBills(UBS)).toBe(false);
    expect(unitBills(UPA)).toBe(true);
    expect(unitBills({ kind: "hospital" })).toBe(true);
    expect(unitBills(null)).toBe(false);
    const pro = billingUser([ "health_professional" ]);
    expect(showUnitProcedures({ user: pro, canCare: true, unit: UPA })).toBe(true);
    expect(showUnitProcedures({ user: pro, canCare: true, unit: UBS })).toBe(false);
    expect(showUnitProcedures({ user: pro, canCare: false, unit: UPA })).toBe(false);
    expect(showUnitProcedures({ user: billingUser([ "health_professional" ], { features: [] }), canCare: true, unit: UPA })).toBe(false);
  });
});

describe("veredito da crítica", () => {
  it("faturável com o instrumento; não faturável com os motivos em português; cancelado", () => {
    expect(verdict(procedure())).toEqual({ tone: "ok", text: "faturável · BPA-I" });
    expect(verdict(procedure({ billable: false, instrument: "bpa_i", blocking_reasons: [ "cid_required", "missing_professional_cns" ] })))
      .toEqual({ tone: "warn", text: "não faturável — o procedimento exige CID-10; profissional sem CNS no cadastro" });
    expect(verdict(procedure({ billable: false, blocking_reasons: [] }))).toEqual({ tone: "warn", text: "não faturável" });
    expect(verdict(procedure({ status: "cancelled" }))).toEqual({ tone: "neutral", text: "cancelado" });
  });

  it("os 13 motivos do contrato têm frase; motivo novo sai cru", () => {
    for (const code of [ "not_in_sigtap", "cbo_not_allowed", "age_out_of_range", "sex_not_allowed", "cid_required", "cid_not_allowed",
      "not_bpa_instrument", "establishment_is_aps", "missing_professional_cns", "missing_professional_cbo", "missing_patient_id",
      "missing_required_patient_field", "missing_establishment_cnes" ]) {
      expect(reasonLabel(code)).not.toBe(code);
    }
    expect(reasonLabel("establishment_is_aps")).toBe("unidade de atenção primária — a produção vai pelo e-SUS APS");
    expect(reasonLabel("habilitation_missing")).toBe("habilitation_missing");
    expect(sourceLabel("lab_report")).toBe("laboratório municipal");
    expect(sourceLabel("rnds")).toBe("rnds");
  });

  it("SIGTAP de outra competência e a frase depois de registrar", () => {
    expect(fallbackNotice(procedure())).toBeNull();
    expect(fallbackNotice(procedure({ sigtap_fallback: true, sigtap_competence: "202609" })))
      .toBe("conferido com a SIGTAP de 09/2026: a tabela da competência ainda não foi importada — o fechamento confere de novo");
    expect(savedText(procedure())).toBe("Procedimento registrado — faturável.");
    expect(savedText(procedure({ billable: false, blocking_reasons: [ "cid_required" ] })))
      .toBe("Procedimento registrado — não faturável — o procedimento exige CID-10. O registro vale; o faturamento confere de novo no fechamento.");
  });
});

describe("formulário do procedimento", () => {
  const ready = { ...emptyProcedure(TODAY22), sigtap: URGENCY };

  it("começa com quantidade 1, eletivo e a data de hoje", () => {
    expect(emptyProcedure(TODAY22)).toEqual({ sigtap: null, quantity: "1", cid10: "", character: "elective", performedOn: TODAY22 });
    expect(sigtapCompetence(ready)).toBe("202610");
  });

  it("recusa só formato", () => {
    expect(procedureProblem(emptyProcedure(TODAY22), TODAY22)).toBe("escolha o procedimento pela busca");
    for (const q of [ "0", "1000", "1,5", "", "-1" ]) expect(procedureProblem({ ...ready, quantity: q }, TODAY22)).toBe("quantidade de 1 a 999, só números");
    expect(procedureProblem({ ...ready, quantity: "999" }, TODAY22)).toBeNull();
    expect(procedureProblem({ ...ready, cid10: "61S" }, TODAY22)).toBe("CID-10 inválido — ex.: S61 ou S610");
    expect(procedureProblem({ ...ready, cid10: "s61.0" }, TODAY22)).toBeNull();
    expect(procedureProblem({ ...ready, performedOn: "2026-10-22" }, TODAY22)).toBe("a data do procedimento não pode ser no futuro");
    expect(procedureProblem({ ...ready, performedOn: "" }, TODAY22)).toBe("informe a data do procedimento");
  });

  it("CID ausente num procedimento que exige: só aviso, nunca bloqueio", () => {
    const suture = { ...emptyProcedure(TODAY22), sigtap: SUTURE };
    expect(procedureProblem(suture, TODAY22)).toBeNull();
    expect(cidHint(suture)).toBe("a SIGTAP exige CID-10 para faturar este procedimento — sem ele, o registro vale, mas fica não faturável");
    expect(cidHint({ ...suture, cid10: "S61" })).toBeNull();
    expect(cidHint(ready)).toBeNull();
  });

  it("corpo: CID normalizado, opcionais só quando há valor", () => {
    expect(normalizeCid(" s61.0 ")).toBe("S610");
    expect(procedureBody({ ...ready, quantity: "2", cid10: "s61.0", character: "urgent" }, {
      establishment: { type: "health_unit", id: "u7" }, patientId: "pa1", attendanceId: "a1", consultationId: null
    })).toEqual({
      establishment: { type: "health_unit", id: "u7" }, performed_on: TODAY22, sigtap_code: "0301060061", quantity: 2,
      character: "urgent", cid10: "S610", patient_id: "pa1", attendance_id: "a1"
    });
    expect(procedureBody(ready, {
      establishment: { type: "lab_provider", id: "pv9" }, patientId: "pa1", professionalUserId: "us5", cboCode: "221205"
    })).toEqual({
      establishment: { type: "lab_provider", id: "pv9" }, performed_on: TODAY22, sigtap_code: "0301060061", quantity: 1,
      character: "elective", patient_id: "pa1", professional_user_id: "us5", cbo_code: "221205"
    });
  });

  it("motivo de cancelamento e de reabertura: 10 a 500, recusado e não cortado", () => {
    expect(reasonProblem("curto")).toBe("motivo com pelo menos 10 caracteres");
    expect(reasonProblem("          x")).toBe("motivo com pelo menos 10 caracteres");
    expect(reasonProblem("x".repeat(501))).toBe("motivo com até 500 caracteres — encurte o texto");
    expect(reasonProblem("lançado no paciente errado")).toBeNull();
  });

  it("lançamento manual: estabelecimento, paciente, profissional e o CBO quando há escolha", () => {
    const base = { establishment: "health_unit:u7", patientId: "pa1", professionalId: "us3", cboOptions: [ "223505", "223565" ], cbo: "" };
    expect(manualProblem({ ...base, establishment: "" })).toBe("escolha o estabelecimento");
    expect(manualProblem({ ...base, patientId: null })).toBe("busque o paciente pelo CPF");
    expect(manualProblem({ ...base, professionalId: "" })).toBe("escolha o profissional");
    expect(manualProblem(base)).toBe("escolha o CBO do profissional");
    expect(manualProblem({ ...base, cbo: "223505" })).toBeNull();
    // Sem CBO no vínculo: segue — a crítica marca missing_professional_cbo.
    expect(manualProblem({ ...base, cboOptions: [], cbo: "" })).toBeNull();
  });
});

describe("estabelecimentos e filtros", () => {
  it("rótulo com CNES ou sem; chave ida e volta", () => {
    expect(establishmentLabel(UPA_EST)).toBe("UPA Boa Vista · CNES 2384299");
    expect(establishmentLabel({ ...LAB_EST, cnes: null })).toBe("Laboratório Municipal · sem CNES");
    expect(establishmentKey(LAB_EST)).toBe("lab_provider:pv9");
    expect(parseEstablishmentKey("lab_provider:pv9")).toEqual({ type: "lab_provider", id: "pv9" });
    expect(parseEstablishmentKey("")).toBeNull();
    expect(parseEstablishmentKey("clinic:x")).toBeNull();
  });

  it("filtros viram só códigos e ids; motivo só com não faturáveis", () => {
    expect(EMPTY_FILTERS(TODAY22)).toEqual({ establishment: "", competence: "202610", billable: "", reason: "", patientId: null });
    expect(procedureQuery(EMPTY_FILTERS(TODAY22))).toEqual({ competence: "202610" });
    expect(procedureQuery({ establishment: "lab_provider:pv9", competence: "202609", billable: "false", reason: "cbo_not_allowed", patientId: "pa1" }))
      .toEqual({ establishmentType: "lab_provider", establishmentId: "pv9", competence: "202609", billable: false, reason: "cbo_not_allowed", patientId: "pa1" });
    expect(procedureQuery({ establishment: "", competence: "202609", billable: "true", reason: "cbo_not_allowed", patientId: null }))
      .toEqual({ competence: "202609", billable: true });
  });
});

describe("competência", () => {
  it("abre na anterior; situação e não faturáveis em frase", () => {
    expect(closingCompetence(TODAY22)).toBe("202609");
    expect(closingCompetence("2026-01-05")).toBe("202512");
    expect(periodStatusText(period())).toBe("aberta");
    expect(periodStatusText(period({ status: "open", version: 2 }))).toBe("aberta · reaberta depois da versão 2");
    expect(periodStatusText(period({ status: "closed", version: 1 }))).toBe("fechada · versão 1");
    expect(periodStatusText(period({ status: "archived" }))).toBe("archived");
    expect(notBillableText(period().summary)).toBe("3 — CBO do profissional não permitido para o procedimento: 2; profissional sem CNS no cadastro: 1");
    expect(notBillableText({ ...period().summary, not_billable: {} })).toBe("nenhum");
  });

  it("descrição do fechamento diz o que entra, o que fica fora e que o arquivo não muda", () => {
    expect(closeDescription(period())).toBe(
      "As críticas são conferidas de novo com o cadastro de agora. Pela última conferência, entram 12 linhas de BPA-I e 3 de BPA-C; " +
      "1 pendência fica fora do arquivo até ser corrigida. Depois de gerado, o arquivo não muda: reabrir gera uma nova versão."
    );
  });

  it("pendências agrupadas por correção, na ordem profissional → paciente → estabelecimento → procedimento; correção nova no fim", () => {
    const groups = groupPending([
      pending({ procedure_id: "a", fix: "procedure" }), pending({ procedure_id: "b", fix: "rnds_link" }),
      pending({ procedure_id: "c", fix: "patient" }), pending({ procedure_id: "d", fix: "professional" }), pending({ procedure_id: "e", fix: "patient" })
    ]);
    expect(groups.map((g) => [ g.fix, g.items.map((i) => i.procedure_id) ])).toEqual([
      [ "professional", [ "d" ] ], [ "patient", [ "c", "e" ] ], [ "procedure", [ "a" ] ], [ "rnds_link", [ "b" ] ]
    ]);
    expect(fixInfo("professional").title).toBe("Cadastro do profissional");
    expect(fixInfo("rnds_link")).toEqual({ title: "rnds_link", hint: "" });
  });

  it("nome do arquivo sem dado pessoal", () => {
    expect(exportFileName(period(), billingExport({ version: 2 }))).toBe("bpa-2384299-202609-v2.txt");
    expect(exportFileName(period({ establishment: { ...UPA_EST, cnes: null } }), billingExport())).toBe("bpa-sem-cnes-202609-v1.txt");
  });
});

describe("configuração", () => {
  const draft = settingsDraft(SETTINGS);

  it("rascunho a partir do api; completo só com tudo", () => {
    expect(draft).toEqual({ originName: "SMS CURITIBA", originAcronym: "SMS", originCnpj: "11222333000181", destinationBody: "SECRETARIA MUNICIPAL DE SAUDE", management: "M" });
    expect(settingsDraft(null).management).toBe("");
    expect(settingsComplete(SETTINGS)).toBe(true);
    expect(settingsComplete({ ...SETTINGS, origin_acronym: null })).toBe(false);
    expect(settingsComplete(null)).toBe(false);
  });

  it("larguras do layout recusadas, nunca cortadas; CNPJ e gestão conferidos", () => {
    expect(settingsProblem(draft)).toBeNull();
    expect(settingsProblem({ ...draft, originName: "x".repeat(31) })).toBe("nome do órgão de origem com até 30 caracteres (largura do arquivo BPA) — abrevie");
    expect(settingsProblem({ ...draft, originAcronym: "SMSCWB1" })).toBe("sigla do órgão de origem com até 6 caracteres (largura do arquivo BPA)");
    expect(settingsProblem({ ...draft, destinationBody: "x".repeat(41) })).toBe("órgão de destino com até 40 caracteres (largura do arquivo BPA) — abrevie");
    expect(settingsProblem({ ...draft, originCnpj: "11222333000180" })).toBe("CNPJ de origem inválido");
    expect(settingsProblem({ ...draft, management: "" })).toBe("escolha a gestão: municipal (M) ou estadual (E)");
    expect(settingsProblem({ ...draft, originName: "  " })).toBe("informe o nome do órgão de origem");
    expect(settingsBody({ ...draft, originCnpj: "11.222.333/0001-81", originName: " SMS CURITIBA " })).toEqual({
      origin_name: "SMS CURITIBA", origin_acronym: "SMS", origin_cnpj: "11222333000181", destination_body: "SECRETARIA MUNICIPAL DE SAUDE", management: "M"
    });
  });
});

describe("recusas", () => {
  it("frases do contrato e das divergências; desconhecido com o status", () => {
    expect(billingError(new ApiError(403, { error: "feature_disabled", feature: "procedure_billing" }, "403"))).toBe(BILLING_DISABLED);
    expect(billingError(refusal(422, "settings_missing"))).toBe(
      "falta a configuração do faturamento (nome, sigla e CNPJ de origem, órgão de destino e gestão) — a administração municipal preenche em Faturamento → Configuração do faturamento"
    );
    expect(billingError(refusal(422, "nothing_to_export"))).toBe("nada a exportar: nenhum procedimento faturável nesta competência e estabelecimento");
    expect(billingError(refusal(409, "already_exported"))).toBe("este procedimento já está no arquivo de uma competência fechada — para cancelar, o faturista reabre a competência");
    expect(billingError(refusal(422, "invalid_date"))).toBe("data inválida: fora da competência atual e das 3 anteriores (BPA Magnético 05.00)");
    expect(billingError(refusal(422, "reason_required"))).toBe("informe o motivo, de 10 a 500 caracteres");
    expect(billingError(refusal(404, "not_found"))).toBe("registro não encontrado — a tela foi atualizada");
    expect(billingError(refusal(422, "citizen_not_verified"))).toBe("o cadastro desta pessoa ainda não foi validado na unidade — valide antes de registrar");
    expect(billingError(new ApiError(500, "", "500"))).toBe("não foi possível concluir (500) — tente de novo");
    expect(billingError(new Error("rede"))).toBe("sem conexão com o servidor — tente de novo");
  });

  it("recusas que pedem reler a tela", () => {
    expect(isStale(refusal(404, "not_found"))).toBe(true);
    expect(isStale(refusal(409, "invalid_transition"))).toBe(true);
    expect(isStale(refusal(409, "already_exported"))).toBe(true);
    expect(isStale(refusal(422, "invalid_date"))).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/billing.test.ts`
Expected: FAIL — o módulo `./billing` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/billing.ts
// Regras de tela do faturamento de procedimentos (módulo 22; ADR 0039;
// contrato 2026-10-21). A crítica SIGTAP, o instrumento, o estabelecimento
// faturante, as pendências e o arquivo são do api: aqui só se traduzem os
// códigos, se monta o corpo e se recusa o que o api recusaria por formato.
// A crítica nunca impede o registro (invariante do ADR 0039).
import {
  ApiError, errorCode, type BillingEstablishment, type BillingExport, type BillingPeriod, type BillingSettings, type BillingSettingsInput,
  type EstablishmentType, type PendingItem, type PerformedProcedure, type ProcedureCharacter, type ProcedureInput, type ProcedureQuery,
  type SigtapProcedure
} from "./api";
import { featureDisabledKey, hasFeature } from "./features";
import { cnpjDigits, isValidCnpj } from "./clinicalDocuments";
import { competenceLabel, competenceOf, recentCompetences } from "./competence";

export const BILLING_CLERK_ROLE = "billing_clerk";
export const BILLING_DISABLED = "o faturamento de procedimentos está desligado nesta cidade";
export const BILLING_NO_ROLE = "seu papel não permite usar o faturamento";
export const LATE_TEXT = "apresentação em atraso";

export const PROCEDURES_KEY = "billingProcedures";
export const CARE_PROCEDURES_KEY = "careProcedures";
export const PERIODS_KEY = "billingPeriods";
export const PENDING_KEY = "billingPending";
export const EXPORTS_KEY = "billingExports";
export const SETTINGS_KEY = "billingSettings";
export const ESTABLISHMENTS_KEY = "billingEstablishments";
export const PROFESSIONALS_KEY = "billingProfessionals";
export const SIGTAP_KEY = "billingSigtap";

type SessionLike = { operator: boolean; memberships?: { role: string }[]; features?: unknown } | null | undefined;

export function canUseBilling(user: SessionLike): boolean {
  return !!user && !user.operator && hasFeature(user, "procedure_billing");
}

export interface BillingRoles { clerk: boolean; admin: boolean; professional: boolean }

export function billingRoles(user: SessionLike): BillingRoles {
  const roles = user?.memberships?.map((m) => m.role) ?? [];
  return {
    clerk: roles.includes(BILLING_CLERK_ROLE), admin: roles.includes("municipal_admin"), professional: roles.includes("health_professional")
  };
}

// ADR 0039: unidade de atenção primária nunca gera BPA (a produção vai pelo
// e-SUS APS, sem duplicar). O dashboard não oferece o registro na UBS; o api,
// se receber, grava com establishment_is_aps.
export function unitBills(unit: { kind: string } | null | undefined): boolean {
  return !!unit && unit.kind !== "ubs";
}

export function showUnitProcedures(a: { user: SessionLike; canCare: boolean; unit: { kind: string } | null | undefined }): boolean {
  return canUseBilling(a.user) && a.canCare && unitBills(a.unit);
}

// Motivos de crítica (contrato §1). Código novo do api aparece cru.
const REASON_LABEL: Record<string, string> = {
  not_in_sigtap: "procedimento fora da SIGTAP da competência",
  cbo_not_allowed: "CBO do profissional não permitido para o procedimento",
  age_out_of_range: "idade do paciente fora da faixa do procedimento",
  sex_not_allowed: "sexo do paciente não permitido para o procedimento",
  cid_required: "o procedimento exige CID-10",
  cid_not_allowed: "CID-10 não compatível com o procedimento",
  not_bpa_instrument: "procedimento não é registrado por BPA",
  establishment_is_aps: "unidade de atenção primária — a produção vai pelo e-SUS APS",
  missing_professional_cns: "profissional sem CNS no cadastro",
  missing_professional_cbo: "profissional sem CBO no vínculo",
  missing_patient_id: "paciente sem CPF nem CNS",
  missing_required_patient_field: "falta dado obrigatório do paciente no cadastro",
  missing_establishment_cnes: "estabelecimento sem CNES"
};
export const REASON_CODES = Object.keys(REASON_LABEL);

export function reasonLabel(code: string): string {
  return REASON_LABEL[code] ?? code;
}

const SOURCE_LABEL: Record<string, string> = {
  attendance: "atendimento", consultation: "consulta", lab_report: "laboratório municipal", manual: "lançamento manual"
};
export function sourceLabel(code: string): string {
  return SOURCE_LABEL[code] ?? code;
}

const INSTRUMENT_LABEL: Record<string, string> = { bpa_i: "BPA-I", bpa_c: "BPA-C", not_bpa: "fora do BPA" };
export function instrumentLabel(code: string): string {
  return INSTRUMENT_LABEL[code] ?? code;
}

export const CHARACTER_OPTIONS: { key: ProcedureCharacter; label: string }[] = [
  { key: "elective", label: "eletivo" }, { key: "urgent", label: "urgência" }
];
export function characterLabel(code: string): string {
  return CHARACTER_OPTIONS.find((o) => o.key === code)?.label ?? code;
}

export type VerdictTone = "ok" | "warn" | "neutral";

export function verdict(p: Pick<PerformedProcedure, "status" | "billable" | "blocking_reasons" | "instrument">): { tone: VerdictTone; text: string } {
  if (p.status === "cancelled") return { tone: "neutral", text: "cancelado" };
  if (p.billable) return { tone: "ok", text: `faturável · ${instrumentLabel(p.instrument)}` };
  const reasons = p.blocking_reasons.map(reasonLabel);
  return { tone: "warn", text: reasons.length > 0 ? `não faturável — ${reasons.join("; ")}` : "não faturável" };
}

export function fallbackNotice(p: Pick<PerformedProcedure, "sigtap_fallback" | "sigtap_competence">): string | null {
  return p.sigtap_fallback
    ? `conferido com a SIGTAP de ${competenceLabel(p.sigtap_competence)}: a tabela da competência ainda não foi importada — o fechamento confere de novo`
    : null;
}

export function savedText(p: PerformedProcedure): string {
  if (p.billable) return "Procedimento registrado — faturável.";
  return `Procedimento registrado — ${verdict(p).text}. O registro vale; o faturamento confere de novo no fechamento.`;
}

// ─── Formulário do procedimento ────────────────────────────────────────────
export const QUANTITY_MAX = 999;

export interface ProcedureDraft {
  sigtap: SigtapProcedure | null; quantity: string; cid10: string; character: ProcedureCharacter; performedOn: string;
}

export function emptyProcedure(today: string): ProcedureDraft {
  return { sigtap: null, quantity: "1", cid10: "", character: "elective", performedOn: today };
}

export function normalizeCid(text: string): string {
  return text.toUpperCase().replace(/[\s.]/g, "");
}

const CID_RE = /^[A-Z]\d{2}[0-9X]?$/;

// Aviso, nunca bloqueio: sem CID, o api grava e marca cid_required.
export function cidHint(d: ProcedureDraft): string | null {
  return d.sigtap?.requires_cid && !normalizeCid(d.cid10)
    ? "a SIGTAP exige CID-10 para faturar este procedimento — sem ele, o registro vale, mas fica não faturável"
    : null;
}

export function procedureProblem(d: ProcedureDraft, today: string): string | null {
  if (!d.sigtap) return "escolha o procedimento pela busca";
  const q = d.quantity.trim();
  if (!/^\d+$/.test(q) || Number(q) < 1 || Number(q) > QUANTITY_MAX) return `quantidade de 1 a ${QUANTITY_MAX}, só números`;
  const cid = normalizeCid(d.cid10);
  if (cid && !CID_RE.test(cid)) return "CID-10 inválido — ex.: S61 ou S610";
  if (!/^\d{4}-\d{2}-\d{2}$/.test(d.performedOn)) return "informe a data do procedimento";
  if (d.performedOn > today) return "a data do procedimento não pode ser no futuro";
  return null;
}

export interface ProcedureContext {
  establishment: { type: EstablishmentType; id: string };
  patientId?: string | null; attendanceId?: string | null; consultationId?: string | null;
  professionalUserId?: string | null; cboCode?: string | null;
}

export function procedureBody(d: ProcedureDraft, ctx: ProcedureContext): ProcedureInput {
  if (!d.sigtap) throw new Error("procedureBody sem procedimento");
  const body: ProcedureInput = {
    establishment: ctx.establishment, performed_on: d.performedOn, sigtap_code: d.sigtap.code,
    quantity: Number(d.quantity.trim()), character: d.character
  };
  const cid = normalizeCid(d.cid10);
  if (cid) body.cid10 = cid;
  if (ctx.patientId) body.patient_id = ctx.patientId;
  if (ctx.attendanceId) body.attendance_id = ctx.attendanceId;
  if (ctx.consultationId) body.consultation_id = ctx.consultationId;
  if (ctx.professionalUserId) body.professional_user_id = ctx.professionalUserId;
  if (ctx.cboCode) body.cbo_code = ctx.cboCode;
  return body;
}

// A SIGTAP da busca é a da competência da data do procedimento.
export function sigtapCompetence(d: ProcedureDraft): string {
  return competenceOf(d.performedOn);
}

// ─── Motivos (cancelamento e reabertura; Divergência B8) ───────────────────
export const REASON_MIN = 10;
export const REASON_MAX = 500;

export function reasonProblem(text: string): string | null {
  const t = text.trim();
  if (t.length < REASON_MIN) return `motivo com pelo menos ${REASON_MIN} caracteres`;
  if (t.length > REASON_MAX) return `motivo com até ${REASON_MAX} caracteres — encurte o texto`;
  return null;
}

// ─── Lançamento manual (Divergência B5) ────────────────────────────────────
export interface ManualChoice { establishment: string; patientId: string | null; professionalId: string; cboOptions: string[]; cbo: string }

// Profissional sem CBO no vínculo segue: a crítica marca missing_professional_cbo.
export function manualProblem(m: ManualChoice): string | null {
  if (!parseEstablishmentKey(m.establishment)) return "escolha o estabelecimento";
  if (!m.patientId) return "busque o paciente pelo CPF";
  if (!m.professionalId) return "escolha o profissional";
  if (m.cboOptions.length > 1 && !m.cboOptions.includes(m.cbo)) return "escolha o CBO do profissional";
  return null;
}

// ─── Estabelecimentos e filtros ─────────────────────────────────────────────
export function establishmentLabel(e: BillingEstablishment): string {
  return e.cnes ? `${e.name} · CNES ${e.cnes}` : `${e.name} · sem CNES`;
}

export function establishmentKey(e: { type: string; id: string }): string {
  return `${e.type}:${e.id}`;
}

export function parseEstablishmentKey(key: string): { type: EstablishmentType; id: string } | null {
  const m = /^(health_unit|lab_provider):(.+)$/.exec(key);
  return m ? { type: m[1] as EstablishmentType, id: m[2] } : null;
}

export interface ProcedureFilters {
  establishment: string; competence: string; billable: "" | "true" | "false"; reason: string; patientId: string | null;
}

export function EMPTY_FILTERS(today: string): ProcedureFilters {
  return { establishment: "", competence: competenceOf(today), billable: "", reason: "", patientId: null };
}

export function procedureQuery(f: ProcedureFilters): ProcedureQuery {
  const q: ProcedureQuery = {};
  const est = parseEstablishmentKey(f.establishment);
  if (est) { q.establishmentType = est.type; q.establishmentId = est.id; }
  if (f.competence) q.competence = f.competence;
  if (f.billable) q.billable = f.billable === "true";
  if (f.billable === "false" && f.reason) q.reason = f.reason;
  if (f.patientId) q.patientId = f.patientId;
  return q;
}

// ─── Competência ────────────────────────────────────────────────────────────
// A tela abre na competência anterior: é a que o faturista fecha no começo do mês.
export function closingCompetence(today: string): string {
  return recentCompetences(today, 2)[1];
}

export function periodStatusText(p: Pick<BillingPeriod, "status" | "version">): string {
  if (p.status === "open") return p.version > 0 ? `aberta · reaberta depois da versão ${p.version}` : "aberta";
  if (p.status === "closed") return `fechada · versão ${p.version}`;
  return p.status;
}

export function notBillableText(summary: BillingPeriod["summary"]): string {
  const entries = Object.entries(summary.not_billable ?? {}).filter(([ , n ]) => n > 0);
  if (entries.length === 0) return "nenhum";
  const total = entries.reduce((sum, [ , n ]) => sum + n, 0);
  return `${total} — ${entries.map(([ code, n ]) => `${reasonLabel(code)}: ${n}`).join("; ")}`;
}

const plural = (n: number, one: string, many: string) => `${n} ${n === 1 ? one : many}`;

export function closeDescription(p: BillingPeriod): string {
  const s = p.summary;
  const pendingText = s.pending === 0
    ? "nenhuma pendência fica fora"
    : `${plural(s.pending, "pendência fica", "pendências ficam")} fora do arquivo até ${s.pending === 1 ? "ser corrigida" : "serem corrigidas"}`;
  return "As críticas são conferidas de novo com o cadastro de agora. " +
    `Pela última conferência, entram ${plural(s.bpa_i_lines, "linha", "linhas")} de BPA-I e ${s.bpa_c_lines} de BPA-C; ${pendingText}. ` +
    "Depois de gerado, o arquivo não muda: reabrir gera uma nova versão.";
}

// ─── Pendências ─────────────────────────────────────────────────────────────
const FIX_INFO: Record<string, { title: string; hint: string }> = {
  professional: { title: "Cadastro do profissional", hint: "CNS ou CBO do profissional — a administração corrige em Equipe → Profissionais" },
  patient: { title: "Cadastro do paciente", hint: "CPF, CNS ou dado obrigatório do paciente — corrigido na validação presencial do Atendimento" },
  establishment: { title: "Cadastro do estabelecimento", hint: "CNES da unidade ou do laboratório — a administração corrige no cadastro da unidade ou do prestador" },
  procedure: { title: "Regra da SIGTAP", hint: "o procedimento não passa na SIGTAP — se foi registrado errado, cancele e registre de novo" }
};
export const FIX_ORDER = [ "professional", "patient", "establishment", "procedure" ];

export function fixInfo(fix: string): { title: string; hint: string } {
  return FIX_INFO[fix] ?? { title: fix, hint: "" };
}

export function groupPending(items: PendingItem[]): { fix: string; items: PendingItem[] }[] {
  const groups = new Map<string, PendingItem[]>();
  for (const item of items) groups.set(item.fix, [ ...(groups.get(item.fix) ?? []), item ]);
  const rank = (fix: string) => { const i = FIX_ORDER.indexOf(fix); return i === -1 ? FIX_ORDER.length : i; };
  return [ ...groups.entries() ]
    .sort(([ a ], [ b ]) => rank(a) - rank(b) || a.localeCompare(b))
    .map(([ fix, list ]) => ({ fix, items: list }));
}

// ─── Arquivo ────────────────────────────────────────────────────────────────
// Nome de reserva (o do api vem pelo Content-Disposition seguro): sem dado pessoal.
export function exportFileName(period: Pick<BillingPeriod, "competence" | "establishment">, exp: Pick<BillingExport, "version">): string {
  return `bpa-${period.establishment.cnes ?? "sem-cnes"}-${period.competence}-v${exp.version}.txt`;
}

// ─── Configuração (contrato §6; Divergência B10) ───────────────────────────
// Larguras do cabeçalho do layout do DATASUS de 09/07/2026: cbc-rsp (30),
// cbc-sgl (6), cbc-dst (40). Mais longo é recusado, nunca cortado.
export const ORIGIN_NAME_MAX = 30;
export const ORIGIN_ACRONYM_MAX = 6;
export const DESTINATION_MAX = 40;

export interface SettingsDraft {
  originName: string; originAcronym: string; originCnpj: string; destinationBody: string; management: "" | "M" | "E";
}

export function settingsDraft(s: BillingSettings | null): SettingsDraft {
  return {
    originName: s?.origin_name ?? "", originAcronym: s?.origin_acronym ?? "", originCnpj: s?.origin_cnpj ?? "",
    destinationBody: s?.destination_body ?? "", management: s?.management ?? ""
  };
}

export function settingsComplete(s: BillingSettings | null): boolean {
  return !!s && !!s.origin_name && !!s.origin_acronym && !!s.origin_cnpj && !!s.destination_body && !!s.management;
}

export function settingsProblem(d: SettingsDraft): string | null {
  const name = d.originName.trim();
  if (!name) return "informe o nome do órgão de origem";
  if (name.length > ORIGIN_NAME_MAX) return `nome do órgão de origem com até ${ORIGIN_NAME_MAX} caracteres (largura do arquivo BPA) — abrevie`;
  const acronym = d.originAcronym.trim();
  if (!acronym) return "informe a sigla do órgão de origem";
  if (acronym.length > ORIGIN_ACRONYM_MAX) return `sigla do órgão de origem com até ${ORIGIN_ACRONYM_MAX} caracteres (largura do arquivo BPA)`;
  if (!isValidCnpj(d.originCnpj)) return "CNPJ de origem inválido";
  const destination = d.destinationBody.trim();
  if (!destination) return "informe o órgão de destino";
  if (destination.length > DESTINATION_MAX) return `órgão de destino com até ${DESTINATION_MAX} caracteres (largura do arquivo BPA) — abrevie`;
  if (d.management !== "M" && d.management !== "E") return "escolha a gestão: municipal (M) ou estadual (E)";
  return null;
}

export function settingsBody(d: SettingsDraft): BillingSettingsInput {
  return {
    origin_name: d.originName.trim(), origin_acronym: d.originAcronym.trim(), origin_cnpj: cnpjDigits(d.originCnpj),
    destination_body: d.destinationBody.trim(), management: d.management === "E" ? "E" : "M"
  };
}

// ─── Recusas ────────────────────────────────────────────────────────────────
const ERRORS: Record<string, string> = {
  missing_role: "seu papel não permite esta ação no faturamento",
  invalid_sigtap_code: "procedimento não encontrado na SIGTAP — escolha pela busca",
  invalid_quantity: "quantidade de 1 a 999, só números",
  invalid_date: "data inválida: fora da competência atual e das 3 anteriores (BPA Magnético 05.00)",
  invalid_cid10: "CID-10 inválido — ex.: S61 ou S610",
  invalid_establishment: "este estabelecimento não fatura BPA — escolha outro",
  invalid_professional: "este profissional não tem vínculo com o estabelecimento — escolha outro",
  // Contrato §12 (B1, B3, B10, D-L2).
  invalid_cpf: "CPF inválido",
  citizen_not_verified: "o cadastro desta pessoa ainda não foi validado na unidade — valide antes de registrar",
  field_required: "preencha todos os campos da configuração do faturamento",
  too_many_sheets: "a competência passou de 999 folhas no arquivo — fale com o suporte",
  patient_not_found: "paciente não encontrado — confira o CPF",
  nothing_to_export: "nada a exportar: nenhum procedimento faturável nesta competência e estabelecimento",
  settings_missing: "falta a configuração do faturamento (nome, sigla e CNPJ de origem, órgão de destino e gestão) — a administração municipal preenche em Faturamento → Configuração do faturamento",
  invalid_cnpj: "CNPJ de origem inválido",
  invalid_management: "escolha a gestão: municipal (M) ou estadual (E)",
  too_long: "texto maior que a largura do arquivo BPA — abrevie",
  reason_required: `informe o motivo, de ${REASON_MIN} a ${REASON_MAX} caracteres`,
  already_exported: "este procedimento já está no arquivo de uma competência fechada — para cancelar, o faturista reabre a competência",
  invalid_transition: "a competência mudou de situação — a tela foi atualizada",
  not_found: "registro não encontrado — a tela foi atualizada"
};

export function billingError(err: unknown): string {
  if (featureDisabledKey(err) !== null) return BILLING_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  if (err instanceof ApiError && err.status === 404) return ERRORS.not_found;
  if (err instanceof ApiError) return `não foi possível concluir (${err.status}) — tente de novo`;
  return "sem conexão com o servidor — tente de novo";
}

// A lista mudou por outra pessoa (sumiu, mudou de situação, entrou num arquivo): quem chama relê.
export function isStale(err: unknown): boolean {
  const code = errorCode(err);
  return code === "not_found" || code === "invalid_transition" || code === "already_exported";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/billing.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/billing.ts src/lib/billing.test.ts
$G commit -m "feat: add the billing screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Faturamento", catálogo do shell e o papel `billing_clerk` no convite

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts` (as duas contagens de grupos), `src/shell/catalog.ts`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Test: `src/shell/modules.billing.test.ts`

**Interfaces:**
- Consumes: `hasFeature` (`features.ts`).
- Produces: `ModuleId` ganha `"billing-procedures" | "billing-periods" | "billing-settings"`; o grupo `"Faturamento"` logo depois de `"Laboratório"`; `INVITE_ROLES` com `{ role: "billing_clerk", label: "Faturista" }` e `isPrivilegedRole("billing_clerk") === true`. As três telas caem no `Placeholder` de `renderModule` até as Tasks 9, 11 e 13.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.billing.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor } from "./modules";

const BILLING_IDS = [ "billing-procedures", "billing-periods", "billing-settings" ];
const ON = { features: [ "procedure_billing" ] };
const userWith = (roles: string[], extra: object = ON) => ({ operator: false, memberships: roles.map((role) => ({ role })), ...extra });
const billingIds = (user: Parameters<typeof navGroupsFor>[0]) =>
  navGroupsFor(user).flatMap((g) => g.items.map((i) => i.id)).filter((id) => BILLING_IDS.includes(id));

describe("grupo Faturamento (módulo 22)", () => {
  it("fica logo depois do Laboratório, com as três telas", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Faturamento")).toBe(labels.indexOf("Laboratório") + 1);
    expect(NAV_GROUPS.find((g) => g.label === "Faturamento")?.items.map((i) => i.id)).toEqual(BILLING_IDS);
  });

  it("sem procedure_billing, ninguém vê o grupo", () => {
    for (const role of [ "billing_clerk", "municipal_admin" ]) {
      expect(billingIds(userWith([ role ], { features: [ "municipal_lab" ] }))).toEqual([]);
    }
    expect(navGroupsFor(userWith([ "billing_clerk" ], {})).some((g) => g.label === "Faturamento")).toBe(false);
  });

  it("faturista: procedimentos e competências; admin: as três; profissional, laboratório e operador: nada", () => {
    expect(billingIds(userWith([ "billing_clerk" ]))).toEqual([ "billing-procedures", "billing-periods" ]);
    expect(billingIds(userWith([ "municipal_admin" ]))).toEqual(BILLING_IDS);
    expect(billingIds(userWith([ "billing_clerk", "municipal_admin" ]))).toEqual(BILLING_IDS);
    for (const role of [ "health_professional", "lab_releaser", "citizen_verifier", "analyst" ]) expect(billingIds(userWith([ role ]))).toEqual([]);
    expect(billingIds({ operator: true, memberships: [], ...ON })).toEqual([]);
    expect(billingIds(null)).toEqual([]);
  });

  it("o faturista sem outro papel não ganha Atendimento nem Laboratório", () => {
    const labels = navGroupsFor(userWith([ "billing_clerk" ], { features: [ "procedure_billing", "municipal_lab", "exam_tracking" ] })).map((g) => g.label);
    expect(labels).toContain("Faturamento");
    expect(labels).not.toContain("Atendimento");
    expect(labels).not.toContain("Laboratório");
  });
});
```

Em `src/lib/team.test.ts`, no `describe("convite")`, troque o primeiro teste por:

```ts
  it("oferece os 13 papéis da cidade que se convidam pela Equipe", () => {
    expect(INVITE_ROLES.map((r) => r.role).sort()).toEqual([
      "analyst", "billing_clerk", "campaign_manager", "citizen_verifier", "exam_regulator", "health_professional", "lab_releaser", "lab_staff",
      "municipal_admin", "protocol_author", "protocol_publisher", "protocol_reviewer", "viewer"
    ]);
    expect(INVITE_ROLES.find((r) => r.role === "exam_regulator")?.label).toBe("Regulador de exames");
    expect(isPrivilegedRole("exam_regulator")).toBe(true);
    expect(INVITE_ROLES.find((r) => r.role === "lab_staff")?.label).toBe("Equipe do laboratório");
    expect(INVITE_ROLES.find((r) => r.role === "lab_releaser")?.label).toBe("Liberador de laudos");
    expect(isPrivilegedRole("lab_staff")).toBe(true);
    expect(isPrivilegedRole("lab_releaser")).toBe(true);
    // Módulo 22: o faturista baixa arquivo com dado de paciente — privilegiado (step-up no convite).
    expect(INVITE_ROLES.find((r) => r.role === "billing_clerk")?.label).toBe("Faturista");
    expect(isPrivilegedRole("billing_clerk")).toBe(true);
    // O exam_provider só entra pelo cadastro do prestador.
    expect(INVITE_ROLES.some((r) => r.role === "exam_provider")).toBe(false);
    expect(INVITE_ROLES.find((r) => r.role === "campaign_manager")?.label).toBe("Gestor de campanhas");
    expect(INVITE_ROLES.find((r) => r.role === "analyst")?.label).toBe("Análise");
  });
```

Em `src/shell/modules.test.ts`, o grupo novo some para o operador e para quem ainda não tem sessão (não há item visível). Troque

```ts
      expect(groups.some((g) => g.label === "Laboratório")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 9);
```

por

```ts
      expect(groups.some((g) => g.label === "Laboratório")).toBe(false);
      expect(groups.some((g) => g.label === "Faturamento")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 10);
```

e

```ts
      expect(groups.some((g) => g.label === "Laboratório")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 8);
```

por

```ts
      expect(groups.some((g) => g.label === "Laboratório")).toBe(false);
      expect(groups.some((g) => g.label === "Faturamento")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 9);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/shell/modules.billing.test.ts src/shell/modules.test.ts src/lib/team.test.ts`
Expected: FAIL — o grupo "Faturamento" não existe, as contagens não batem e `billing_clerk` não está no convite.

- [ ] **Step 3: Write minimal implementation**

`src/shell/modules.ts` — o tipo:

```ts
  | "lab-admin" | "lab-reports"
  | "billing-procedures" | "billing-periods" | "billing-settings";
```

(no lugar de `  | "lab-admin" | "lab-reports";`). Em `NAV_GROUPS`, logo depois do fechamento do grupo "Laboratório" (depois de `{ id: "lab-reports", label: "Relatórios do laboratório", icon: "▤" }` e do `]},` que o fecha):

```ts
  { label: "Faturamento", items: [
    { id: "billing-procedures", label: "Procedimentos realizados", icon: "✚" },
    { id: "billing-periods", label: "Competências e arquivo BPA", icon: "▥" },
    { id: "billing-settings", label: "Configuração do faturamento", icon: "⚙" }
  ]},
```

Em `navGroupsFor`, depois de `const invOn = labOn && hasFeature(user, "lab_inventory");`:

```ts
  // Módulo 22 (ADR 0039): faturamento com `procedure_billing`. O faturista
  // opera; o municipal_admin lê e configura. O profissional registra no
  // atendimento, não por este grupo.
  const billingOn = !user?.operator && hasFeature(user, "procedure_billing");
  const isBillingClerk = roles.includes("billing_clerk");
```

E no filtro de itens, antes de `if (item.id === "integrations" || item.id === "cnes") return canIntegrations;`:

```ts
      if (item.id === "billing-procedures" || item.id === "billing-periods") return billingOn && (isBillingClerk || isAdmin);
      if (item.id === "billing-settings") return billingOn && isAdmin;
```

`src/shell/catalog.ts` — no import, troque `  IconUser, IconUserCircle, IconUsers, IconUsersGroup` por `  IconUser, IconUserCircle, IconUsers, IconUsersGroup, IconCalendarStats, IconFileInvoice, IconReceipt, IconReceiptTax`; em `groups`, depois de `"Laboratório"`:

```ts
    "Faturamento": { icon: IconReceipt, description: "Procedimentos, competências e arquivo BPA" },
```

e em `screens`, depois de `"lab-reports"`:

```ts
    "billing-procedures": { icon: IconFileInvoice, description: "Procedimentos com a crítica SIGTAP e lançamento manual" },
    "billing-periods": { icon: IconCalendarStats, description: "Fechamento por estabelecimento, pendências e arquivo" },
    "billing-settings": { icon: IconReceiptTax, description: "Órgão de origem, destino e gestão do arquivo BPA" },
```

`src/lib/team.ts` — em `INVITE_ROLES`, depois de `{ role: "lab_releaser", label: "Liberador de laudos" },`:

```ts
  // Módulo 22 (ADR 0039): fecha competência e baixa o arquivo BPA (dado de paciente).
  { role: "billing_clerk", label: "Faturista" },
```

e troque

```ts
const PRIVILEGED_ROLES = new Set([ "municipal_admin", REVIEWER_ROLE, VERIFIER_ROLE, PROFESSIONAL_ROLE, CAMPAIGN_MANAGER_ROLE, "exam_regulator", "lab_staff", "lab_releaser" ]);
```

por

```ts
const PRIVILEGED_ROLES = new Set([ "municipal_admin", REVIEWER_ROLE, VERIFIER_ROLE, PROFESSIONAL_ROLE, CAMPAIGN_MANAGER_ROLE, "exam_regulator", "lab_staff", "lab_releaser", "billing_clerk" ]);
```

A tela Equipe não muda: o seletor lê `INVITE_ROLES` e o `SensitiveAction` pede step-up por `isPrivilegedRole`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/shell src/lib/team.test.ts src/modules/Team.test.tsx src/App.navigation.test.tsx && npx tsc --noEmit`
Expected: PASS (inclusive `catalog.test.ts`, que cobra ícone e descrição do grupo e das três telas).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.billing.test.ts src/shell/catalog.ts src/lib/team.ts src/lib/team.test.ts
$G commit -m "feat: add the billing group to the menu and the billing clerk invitation role

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Procedimentos na linha do tempo do profissional

**Files:**
- Modify: `src/lib/api.ts` (`TimelineKind`), `src/lib/timeline.ts`, `src/lib/timeline.test.ts`, `src/lib/timeline.exam.test.ts`, `src/modules/timeline/PatientTimeline.tsx`
- Test: `src/lib/timeline.procedure.test.ts`, `src/modules/timeline/PatientTimeline.procedure.test.tsx`

**Interfaces:**
- Consumes: `entry`, `page` (`src/test/timelineFixtures.ts`); `hasFeature`.
- Produces: `TimelineKind` ganha `"procedure"`; `TIMELINE_KINDS` termina em `{ key: "procedure", label: "Procedimento" }`; `timelineKinds(examTracking: boolean, procedureBilling = false)`; `statusLabel("recorded") === "registrado"`. Contrato §9: a entrada `procedure` mostra nome, data, unidade e profissional — o api manda `title` (nome do procedimento), `unit`, `professional` e `status`; `detail_ref` nulo (não abre detalhe).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/timeline.procedure.test.ts
import { describe, expect, it } from "vitest";
import { EMPTY_DRAFT, TIMELINE_KINDS, detailTarget, filtersFrom, statusLabel, timelineKinds, timelineKindLabel, toggleKind } from "./timeline";
import { entry } from "../test/timelineFixtures";

describe("linha do tempo — procedimentos (módulo 22; contrato §9)", () => {
  const proc = entry({
    id: "procedure:pp1", kind: "procedure", title: "Atendimento de urgência em atenção especializada", summary: null,
    status: "recorded", detail_ref: null
  });

  it("tipo e situação em português; não abre detalhe", () => {
    expect(timelineKindLabel("procedure")).toBe("Procedimento");
    expect(TIMELINE_KINDS[TIMELINE_KINDS.length - 1]).toEqual({ key: "procedure", label: "Procedimento" });
    expect(statusLabel("recorded")).toBe("registrado");
    expect(statusLabel("cancelled")).toBe("cancelado");
    expect(detailTarget(proc, true)).toBeNull();
  });

  it("sem procedure_billing, o tipo não existe nos filtros nem vai no pedido", () => {
    const kinds = timelineKinds(true, false);
    expect(kinds.some((k) => k.key === "procedure")).toBe(false);
    expect(kinds.some((k) => k.key === "exam")).toBe(true);
    expect(timelineKinds(false, true).map((k) => k.key)).toContain("procedure");
    expect(timelineKinds(true, true)).toEqual(TIMELINE_KINDS);
    expect(filtersFrom(toggleKind(EMPTY_DRAFT, "campaign"), kinds).kinds).not.toContain("procedure");
  });

  it("com ele, só procedimento vai como kinds[]=procedure", () => {
    expect(filtersFrom({ kinds: [ "procedure" ], from: "", to: "" }).kinds).toEqual([ "procedure" ]);
  });
});
```

```tsx
// src/modules/timeline/PatientTimeline.procedure.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getTimeline: vi.fn() };
});

import * as api from "../../lib/api";
import { PatientTimeline } from "./PatientTimeline";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { entry, page } from "../../test/timelineFixtures";

const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

describe("PatientTimeline — procedimentos (módulo 22)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getTimeline ]) m(fn).mockReset();
    m(api.getTimeline).mockResolvedValue(page({ entries: [ entry({
      id: "procedure:pp1", kind: "procedure", occurred_at: "2026-10-21T09:10:00-03:00", unit: { id: "u7", name: "UPA Boa Vista" },
      professional: { name: "Dra. Helena Prado", cbo_code: "225125" }, title: "Atendimento de urgência em atenção especializada",
      summary: null, status: "recorded", detail_ref: null
    }) ] }));
  });

  it("sem o interruptor não há filtro Procedimento; com ele, o tipo aparece, a entrada mostra unidade e profissional e não abre", async () => {
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline" ] }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    await screen.findByText("Linha do tempo de Joana Lima");
    expect(screen.queryByLabelText("Procedimento")).toBeNull();
    cleanup();
    m(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "patient_timeline", "procedure_billing" ] }));
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    await screen.findByLabelText("Procedimento");
    expect(screen.getByText("Atendimento de urgência em atenção especializada")).not.toBeNull();
    expect(screen.getByText("UPA Boa Vista")).not.toBeNull();
    expect(screen.getByText("Dra. Helena Prado")).not.toBeNull();
    expect(screen.getByText("registrado")).not.toBeNull();
    expect(screen.queryByRole("button", { name: /^Abrir procedimento/ })).toBeNull();
    fireEvent.click(screen.getByLabelText("Campanha"));
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    await waitFor(() => expect(api.getTimeline).toHaveBeenLastCalledWith({ attendanceId: "a1" }, {
      kinds: [ "triage", "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification", "procedure" ],
      from: "", to: ""
    }, null));
  });
});
```

Ajustes nos testes existentes (o tipo novo entra no fim da lista do contrato):

Em `src/lib/timeline.test.ts`, troque

```ts
  it("nove tipos do contrato, em português; desconhecido sai cru", () => {
    expect(TIMELINE_KINDS.map((k) => k.key)).toEqual([
      "triage", "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification", "campaign", "exam"
    ]);
```

por

```ts
  it("dez tipos do contrato, em português; desconhecido sai cru", () => {
    expect(TIMELINE_KINDS.map((k) => k.key)).toEqual([
      "triage", "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification", "campaign", "exam",
      "procedure"
    ]);
```

e

```ts
      kinds: [ "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification", "exam" ], // + exame (módulo 21a)
```

por

```ts
      kinds: [ "attendance", "screening", "appointment", "consultation", "clinical_document", "controlled_notification", "exam", "procedure" ], // + exame (21a) e procedimento (22)
```

Em `src/lib/timeline.exam.test.ts`, troque

```ts
    expect(TIMELINE_KINDS[TIMELINE_KINDS.length - 1]).toEqual({ key: "exam", label: "Exame" });
```

por

```ts
    expect(TIMELINE_KINDS.find((k) => k.key === "exam")).toEqual({ key: "exam", label: "Exame" });
```

e

```ts
    expect(timelineKinds(true)).toEqual(TIMELINE_KINDS);
```

por

```ts
    expect(timelineKinds(true, true)).toEqual(TIMELINE_KINDS);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/timeline.procedure.test.ts src/modules/timeline/PatientTimeline.procedure.test.tsx src/lib/timeline.test.ts src/lib/timeline.exam.test.ts`
Expected: FAIL — `procedure` não está em `TIMELINE_KINDS`, `statusLabel("recorded")` sai cru e não há filtro Procedimento.

- [ ] **Step 3: Write minimal implementation**

`src/lib/api.ts` — troque

```ts
  | "clinical_document" | "controlled_notification" | "campaign" | "exam";
```

por

```ts
  | "clinical_document" | "controlled_notification" | "campaign" | "exam" | "procedure";
```

`src/lib/timeline.ts` — em `TIMELINE_KINDS`, troque `  { key: "exam", label: "Exame" }` por

```ts
  { key: "exam", label: "Exame" },
  { key: "procedure", label: "Procedimento" }
```

troque a função `timelineKinds` por

```ts
// O tipo Exame só existe com o rastreamento de exames ligado (módulo 21a); o
// Procedimento, com o faturamento de procedimentos (módulo 22).
export function timelineKinds(examTracking: boolean, procedureBilling = false): typeof TIMELINE_KINDS {
  return TIMELINE_KINDS.filter((k) => (k.key !== "exam" || examTracking) && (k.key !== "procedure" || procedureBilling));
}
```

e, em `STATUS_LABEL`, troque `  external: "resultado externo"` por

```ts
  external: "resultado externo",
  // Módulo 22 (contrato §1): procedimento realizado (cancelled já está acima).
  recorded: "registrado"
```

`src/modules/timeline/PatientTimeline.tsx` — troque

```ts
  const kinds = timelineKinds(hasFeature(user, "exam_tracking"));
```

por

```ts
  const kinds = timelineKinds(hasFeature(user, "exam_tracking"), hasFeature(user, "procedure_billing"));
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/timeline src/modules/timeline src/modules/attendance/UnitQueue.timeline.test.tsx src/modules/consultation/PatientPanel.timeline.test.tsx src/modules/ClinicalRecord.timeline.test.tsx && npx tsc --noEmit`
Expected: PASS (as chamadas com "todos marcados" continuam sem `kinds[]`).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/timeline.ts src/lib/timeline.test.ts src/lib/timeline.exam.test.ts src/lib/timeline.procedure.test.ts src/modules/timeline/PatientTimeline.tsx src/modules/timeline/PatientTimeline.procedure.test.tsx
$G commit -m "feat: show performed procedures in the professional timeline

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Busca SIGTAP, formulário do procedimento e veredito da crítica

**Files:**
- Create: `src/modules/billing/SigtapSearch.tsx`, `src/modules/billing/ProcedureVerdict.tsx`, `src/modules/billing/ProcedureForm.tsx`
- Test: `src/modules/billing/ProcedureForm.test.tsx`

**Interfaces:**
- Consumes: `searchBillingSigtap`, `PerformedProcedure`, `ProcedureCharacter`, `SigtapProcedure` (Task 1); `CHARACTER_OPTIONS`, `LATE_TEXT`, `SIGTAP_KEY`, `billingError`, `cidHint`, `emptyProcedure`, `fallbackNotice`, `instrumentLabel`, `procedureProblem`, `sigtapCompetence`, `verdict`, `ProcedureDraft` (Task 2); `useDebouncedValue`; `competenceLabel`; `fmtDay` (`src/lib/exams.ts`).
- Produces:
  - `SigtapSearch({ competence, onPick, delayMs? })` — campo rotulado **"Procedimento (SIGTAP)"**, resultados como botões `"<código> — <nome> · <instrumentos>"`;
  - `ProcedureVerdict({ p })` — `Tag` com o veredito, marca "apresentação em atraso" e o aviso da SIGTAP de outra competência;
  - `ProcedureForm({ today, dateEditable, before?, beforeProblem?, searchDelayMs?, submit(draft): Promise<PerformedProcedure>, onSaved(p), onCancel() })` — região "Registrar procedimento", campos "Data do procedimento" (só com `dateEditable`), "Quantidade", "CID-10", "Caráter", botões "Registrar procedimento", "Trocar procedimento" e "Cancelar".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/billing/ProcedureForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), searchBillingSigtap: vi.fn() };
});

import * as api from "../../lib/api";
import { ProcedureForm } from "./ProcedureForm";
import { ProcedureVerdict } from "./ProcedureVerdict";
import type { ProcedureDraft } from "../../lib/billing";
import { renderWithProviders } from "../../test/campaignFixtures";
import { SUTURE, TODAY22, URGENCY, billingUser, procedure } from "../../test/billingFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(billingUser([ "health_professional" ]));
  mocked(api.searchBillingSigtap).mockReset().mockResolvedValue([ URGENCY, SUTURE ]);
});

function setup(dateEditable = false) {
  const submit = vi.fn<(d: ProcedureDraft) => Promise<api.PerformedProcedure>>().mockResolvedValue(procedure());
  const onSaved = vi.fn();
  renderWithProviders(
    <ProcedureForm today={TODAY22} dateEditable={dateEditable} searchDelayMs={0} submit={submit} onSaved={onSaved} onCancel={() => undefined} />
  );
  return { submit, onSaved, form: screen.getByRole("region", { name: "Registrar procedimento" }) };
}

async function pick(form: HTMLElement, term: string, code: string) {
  fireEvent.change(within(form).getByLabelText("Procedimento (SIGTAP)"), { target: { value: term } });
  fireEvent.click(await within(form).findByRole("button", { name: new RegExp(`^${code} — `) }));
}

const send = (form: HTMLElement) => fireEvent.click(within(form).getByRole("button", { name: "Registrar procedimento" }));

describe("ProcedureForm", () => {
  it("busca na SIGTAP da competência da data e registra com quantidade e caráter", async () => {
    const { form, submit, onSaved } = setup();
    expect(within(form).getByText("Data: 21/10/2026")).not.toBeNull();
    expect(within(form).queryByLabelText("Data do procedimento")).toBeNull();
    await pick(form, "urgencia", "0301060061");
    expect(api.searchBillingSigtap).toHaveBeenCalledWith("urgencia", "202610");
    expect(within(form).getByText("0301060061 — ATENDIMENTO DE URGENCIA EM ATENCAO ESPECIALIZADA")).not.toBeNull();
    fireEvent.change(within(form).getByLabelText("Caráter"), { target: { value: "urgent" } });
    send(form);
    await waitFor(() => expect(submit).toHaveBeenCalledWith({ sigtap: URGENCY, quantity: "1", cid10: "", character: "urgent", performedOn: TODAY22 }));
    expect(onSaved).toHaveBeenCalledWith(procedure());
  });

  it("sem CID num procedimento que exige: avisa, mas envia — quem decide é o api", async () => {
    const { form, submit } = setup();
    await pick(form, "sutura", "0401010066");
    expect(within(form).getByRole("note").textContent).toBe(
      "a SIGTAP exige CID-10 para faturar este procedimento — sem ele, o registro vale, mas fica não faturável"
    );
    send(form);
    await waitFor(() => expect(submit).toHaveBeenCalledTimes(1));
    expect(within(form).queryByRole("alert")).toBeNull();
  });

  it("quantidade fora de 1..999, CID mal formado e data no futuro não enviam", async () => {
    const { form, submit } = setup(true);
    await pick(form, "urgencia", "0301060061");
    fireEvent.change(within(form).getByLabelText("Quantidade"), { target: { value: "1000" } });
    send(form);
    expect(within(form).getByRole("alert").textContent).toBe("quantidade de 1 a 999, só números");
    fireEvent.change(within(form).getByLabelText("Quantidade"), { target: { value: "2" } });
    fireEvent.change(within(form).getByLabelText("CID-10"), { target: { value: "61S" } });
    send(form);
    expect(within(form).getByRole("alert").textContent).toBe("CID-10 inválido — ex.: S61 ou S610");
    fireEvent.change(within(form).getByLabelText("CID-10"), { target: { value: "s61.0" } });
    fireEvent.change(within(form).getByLabelText("Data do procedimento"), { target: { value: "2026-10-22" } });
    send(form);
    expect(within(form).getByRole("alert").textContent).toBe("a data do procedimento não pode ser no futuro");
    expect(submit).not.toHaveBeenCalled();
  });

  it("data editável troca a competência da busca", async () => {
    const { form } = setup(true);
    fireEvent.change(within(form).getByLabelText("Data do procedimento"), { target: { value: "2026-09-30" } });
    expect(within(form).getByText("tabela SIGTAP da competência 09/2026")).not.toBeNull();
    await pick(form, "urgencia", "0301060061");
    expect(api.searchBillingSigtap).toHaveBeenLastCalledWith("urgencia", "202609");
  });

  it("sem procedimento escolhido não envia; 422 do api vira frase e o formulário fica", async () => {
    const { form, submit } = setup();
    send(form);
    expect(within(form).getByRole("alert").textContent).toBe("escolha o procedimento pela busca");
    submit.mockRejectedValueOnce(new api.ApiError(422, { error: "invalid_date" }, "x"));
    await pick(form, "urgencia", "0301060061");
    send(form);
    expect((await within(form).findByRole("alert")).textContent).toBe(
      "data inválida: fora da competência atual e das 3 anteriores (BPA Magnético 05.00)"
    );
    expect(within(form).getByText("0301060061 — ATENDIMENTO DE URGENCIA EM ATENCAO ESPECIALIZADA")).not.toBeNull();
  });

  it("busca sem resultado e recusa da busca", async () => {
    const { form } = setup();
    mocked(api.searchBillingSigtap).mockResolvedValueOnce([]);
    fireEvent.change(within(form).getByLabelText("Procedimento (SIGTAP)"), { target: { value: "zz" } });
    expect(await within(form).findByText("nenhum procedimento encontrado")).not.toBeNull();
    mocked(api.searchBillingSigtap).mockRejectedValueOnce(new api.ApiError(403, { error: "feature_disabled", feature: "procedure_billing" }, "x"));
    fireEvent.change(within(form).getByLabelText("Procedimento (SIGTAP)"), { target: { value: "zzz" } });
    expect((await within(form).findByRole("alert")).textContent).toBe("o faturamento de procedimentos está desligado nesta cidade");
  });
});

describe("ProcedureVerdict", () => {
  it("faturável, não faturável com motivo, atraso e SIGTAP de outra competência", () => {
    renderWithProviders(<ProcedureVerdict p={procedure({ billable: false, blocking_reasons: [ "cbo_not_allowed" ], late: true, sigtap_fallback: true, sigtap_competence: "202609" })} />);
    expect(screen.getByText("não faturável — CBO do profissional não permitido para o procedimento")).not.toBeNull();
    expect(screen.getByText("apresentação em atraso")).not.toBeNull();
    expect(screen.getByText(/conferido com a SIGTAP de 09\/2026/)).not.toBeNull();
    cleanup();
    renderWithProviders(<ProcedureVerdict p={procedure()} />);
    expect(screen.getByText("faturável · BPA-I")).not.toBeNull();
    expect(screen.queryByText("apresentação em atraso")).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/billing/ProcedureForm.test.tsx`
Expected: FAIL — `./ProcedureForm` e `./ProcedureVerdict` não existem.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/SigtapSearch.tsx
// Busca na SIGTAP da competência (contrato §3; reaproveita a tabela da
// plataforma): pelo nome ou pelo código, a partir de 2 caracteres. O cache
// sai da memória quando a tela fecha (gcTime 0).
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { searchBillingSigtap, type SigtapProcedure } from "../../lib/api";
import { SIGTAP_KEY, billingError, instrumentLabel } from "../../lib/billing";
import { useDebouncedValue } from "../../lib/useDebouncedValue";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, muted } from "../exams/styles";

export const SIGTAP_MIN_CHARS = 2;

export function SigtapSearch({ competence, onPick, delayMs = 300 }: { competence: string; onPick(p: SigtapProcedure): void; delayMs?: number }) {
  const [ text, setText ] = useState("");
  const typed = text.trim();
  const term = useDebouncedValue(typed, delayMs);
  const enabled = term.length >= SIGTAP_MIN_CHARS;
  const query = useQuery({
    queryKey: [ SIGTAP_KEY, competence, term ], queryFn: () => searchBillingSigtap(term, competence), enabled, gcTime: 0, retry: false
  });
  // Depois de escolher, o campo limpa na hora; o termo "atrasado" não mantém a lista velha.
  const show = enabled && typed.length >= SIGTAP_MIN_CHARS;
  const optionText = (p: SigtapProcedure) =>
    `${p.code} — ${p.name}${p.instruments.length > 0 ? ` · ${p.instruments.map(instrumentLabel).join(", ")}` : ""}`;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
      <label style={fieldLabel}>
        Procedimento (SIGTAP)
        <input value={text} placeholder="nome ou código" style={inputStyle} onChange={(e) => setText(e.target.value)} />
      </label>
      {typed.length > 0 && typed.length < SIGTAP_MIN_CHARS && <p style={muted}>digite pelo menos {SIGTAP_MIN_CHARS} caracteres</p>}
      {show && query.isPending && <p className="mono" style={muted}>buscando…</p>}
      {show && query.isError && <p role="alert" style={alert}>{billingError(query.error)}</p>}
      {show && query.isSuccess && query.data.length === 0 && <p style={muted}>nenhum procedimento encontrado</p>}
      {show && query.isSuccess && query.data.length > 0 && (
        <ul aria-label="resultados da SIGTAP" style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 4 }}>
          {query.data.map((p) => (
            <li key={p.code}>
              <button type="button" style={{ ...secondaryButtonStyle, width: "100%", textAlign: "left" }} onClick={() => { onPick(p); setText(""); }}>
                {optionText(p)}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

```tsx
// src/modules/billing/ProcedureVerdict.tsx
// A crítica do procedimento como o api devolveu (contrato §2): "faturável ·
// BPA-I" ou "não faturável — motivo"; a apresentação em atraso (B7) e a
// SIGTAP de outra competência quando a da data não foi importada.
import type { PerformedProcedure } from "../../lib/api";
import { LATE_TEXT, fallbackNotice, verdict } from "../../lib/billing";
import { Tag } from "../../components/Tag";

export function ProcedureVerdict({ p }: { p: PerformedProcedure }) {
  const v = verdict(p);
  const fallback = fallbackNotice(p);
  return (
    <span style={{ display: "flex", flexDirection: "column", gap: 4, alignItems: "flex-start" }}>
      <Tag tone={v.tone} mono={false}>{v.text}</Tag>
      {p.late && <Tag tone="info" mono={false}>{LATE_TEXT}</Tag>}
      {fallback && <span style={{ fontSize: 11.5, color: "var(--ink2)" }}>{fallback}</span>}
    </span>
  );
}
```

```tsx
// src/modules/billing/ProcedureForm.tsx
// Registro de um procedimento realizado (contrato §3; spec §5.1): busca na
// SIGTAP da competência da data, quantidade, CID-10 e caráter. A crítica é do
// api e nunca impede o registro (ADR 0039): CID ausente num procedimento que
// exige só avisa — o 201 volta "não faturável". `before` são os campos de
// quem chama (lançamento manual) e `beforeProblem`, o que falta neles.
import { useState, type ReactNode } from "react";
import type { PerformedProcedure, ProcedureCharacter } from "../../lib/api";
import {
  CHARACTER_OPTIONS, billingError, cidHint, emptyProcedure, procedureProblem, sigtapCompetence, type ProcedureDraft
} from "../../lib/billing";
import { competenceLabel } from "../../lib/competence";
import { fmtDay } from "../../lib/exams";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, muted, row } from "../exams/styles";
import { SigtapSearch } from "./SigtapSearch";

interface Props {
  today: string;
  // No atendimento e na consulta a data é a de hoje; no avulso e no manual, editável.
  dateEditable: boolean;
  before?: ReactNode;
  beforeProblem?: string | null;
  searchDelayMs?: number;
  submit(draft: ProcedureDraft): Promise<PerformedProcedure>;
  onSaved(p: PerformedProcedure): void;
  onCancel(): void;
}

export function ProcedureForm({ today, dateEditable, before, beforeProblem = null, searchDelayMs, submit, onSaved, onCancel }: Props) {
  const [ draft, setDraft ] = useState<ProcedureDraft>(() => emptyProcedure(today));
  const [ error, setError ] = useState<string | null>(null);
  const set = (patch: Partial<ProcedureDraft>) => { setError(null); setDraft((d) => ({ ...d, ...patch })); };
  const hint = cidHint(draft);
  const competence = sigtapCompetence(draft);

  async function save() {
    const problem = beforeProblem ?? procedureProblem(draft, today);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      onSaved(await submit(draft));
    } catch (err) {
      setError(billingError(err));
    }
  }

  return (
    <section aria-label="Registrar procedimento" style={box}>
      <div style={form}>
        {before}
        {dateEditable ? (
          <label style={fieldLabel}>
            Data do procedimento
            <input type="date" value={draft.performedOn} max={today} style={inputStyle} onChange={(e) => set({ performedOn: e.target.value })} />
          </label>
        ) : <p style={muted}>{`Data: ${fmtDay(draft.performedOn)}`}</p>}
        {draft.sigtap ? (
          <div style={row}>
            <span>{`${draft.sigtap.code} — ${draft.sigtap.name}`}</span>
            <button type="button" style={secondaryButtonStyle} onClick={() => set({ sigtap: null })}>Trocar procedimento</button>
          </div>
        ) : (
          <SigtapSearch competence={competence} delayMs={searchDelayMs} onPick={(p) => set({ sigtap: p })} />
        )}
        <p style={muted}>{`tabela SIGTAP da competência ${competenceLabel(competence)}`}</p>
        <div style={row}>
          <label style={fieldLabel}>
            Quantidade
            <input value={draft.quantity} inputMode="numeric" style={inputStyle} onChange={(e) => set({ quantity: e.target.value })} />
          </label>
          <label style={fieldLabel}>
            CID-10
            <input value={draft.cid10} autoComplete="off" style={inputStyle} onChange={(e) => set({ cid10: e.target.value })} />
          </label>
          <label style={fieldLabel}>
            Caráter
            <select value={draft.character} style={inputStyle} onChange={(e) => set({ character: e.target.value as ProcedureCharacter })}>
              {CHARACTER_OPTIONS.map((o) => <option key={o.key} value={o.key}>{o.label}</option>)}
            </select>
          </label>
        </div>
        {hint && <p role="note" style={muted}>{hint}</p>}
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <ActionButton onClick={save}>Registrar procedimento</ActionButton>
          <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/billing/ProcedureForm.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS (as guardas de tema também: sem cor literal, sem `outline: "none"`).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/billing/SigtapSearch.tsx src/modules/billing/ProcedureVerdict.tsx src/modules/billing/ProcedureForm.tsx src/modules/billing/ProcedureForm.test.tsx
$G commit -m "feat: add the procedure form with the SIGTAP search and the billing verdict

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Lista com cancelamento e o painel de procedimentos do atendimento

**Files:**
- Create: `src/modules/billing/ProcedureTable.tsx`, `src/modules/billing/ProceduresPanel.tsx`
- Test: `src/modules/billing/ProceduresPanel.test.tsx`

**Interfaces:**
- Consumes: `cancelProcedure`, `listProcedures`, `recordProcedure`, `HealthUnit`, `PerformedProcedure` (Task 1); `CARE_PROCEDURES_KEY`, `billingError`, `characterLabel`, `establishmentLabel`, `isStale`, `procedureBody`, `reasonProblem`, `savedText`, `sourceLabel` (Task 2); `ProcedureForm`, `ProcedureVerdict` (Task 5); `todayInCity` (`src/lib/campaigns.ts`); `useAuth`.
- Produces:
  - `ProcedureTable({ rows, billing?, canCancel(p), onChanged(message), onStale? })` — colunas Data, Procedimento, (com `billing`: Paciente, Estabelecimento, Profissional, Origem), Situação; botão "Cancelar procedimento" por linha que pode; o formulário "Cancelar <código>" com "Motivo do cancelamento", "Confirmar cancelamento" e "Voltar";
  - `ProceduresPanel({ unit, attendanceId, consultationId?, patientId?, today?, searchDelayMs? })` — região "Procedimentos realizados"; lê `listProcedures({ attendanceId })` ao abrir e depois de cada ação; botão "Registrar procedimento"; cancela só os próprios.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/billing/ProceduresPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listProcedures: vi.fn(), recordProcedure: vi.fn(), cancelProcedure: vi.fn(), searchBillingSigtap: vi.fn() };
});

import * as api from "../../lib/api";
import { ProceduresPanel } from "./ProceduresPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { SUTURE, TODAY22, UPA, billingUser, procedure } from "../../test/billingFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listProcedures, api.recordProcedure, api.cancelProcedure, api.searchBillingSigtap ];
afterEach(cleanup);
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "health_professional" ], { id: "us1" }));
  mocked(api.listProcedures).mockResolvedValue({ procedures: [ procedure() ], next_before: null });
  mocked(api.searchBillingSigtap).mockResolvedValue([ SUTURE ]);
  mocked(api.recordProcedure).mockResolvedValue(procedure({
    id: "pp2", sigtap_code: SUTURE.code, label: SUTURE.name, billable: false, blocking_reasons: [ "cid_required" ]
  }));
});

async function register(panel: HTMLElement) {
  fireEvent.click(within(panel).getByRole("button", { name: "Registrar procedimento" }));
  const form = within(panel).getByRole("region", { name: "Registrar procedimento" });
  fireEvent.change(within(form).getByLabelText("Procedimento (SIGTAP)"), { target: { value: "sutura" } });
  fireEvent.click(await within(form).findByRole("button", { name: /^0401010066 — / }));
  fireEvent.click(within(form).getByRole("button", { name: "Registrar procedimento" }));
}

describe("ProceduresPanel", () => {
  it("lista do atendimento e registro com o id da consulta; 201 não faturável diz o motivo e relê", async () => {
    renderWithProviders(<ProceduresPanel unit={UPA} attendanceId="a1" consultationId="cs1" patientId="pa1" today={TODAY22} searchDelayMs={0} />);
    const panel = screen.getByRole("region", { name: "Procedimentos realizados" });
    expect(await within(panel).findByText("faturável · BPA-I")).not.toBeNull();
    expect(api.listProcedures).toHaveBeenCalledWith({ attendanceId: "a1" });
    await register(panel);
    await waitFor(() => expect(api.recordProcedure).toHaveBeenCalledWith({
      establishment: { type: "health_unit", id: "u7" }, performed_on: TODAY22, sigtap_code: "0401010066", quantity: 1,
      character: "elective", patient_id: "pa1", attendance_id: "a1", consultation_id: "cs1"
    }));
    expect((await within(panel).findByRole("status")).textContent).toBe(
      "Procedimento registrado — não faturável — o procedimento exige CID-10. O registro vale; o faturamento confere de novo no fechamento."
    );
    await waitFor(() => expect(api.listProcedures).toHaveBeenCalledTimes(2));
    expect(within(panel).queryByRole("region", { name: "Registrar procedimento" })).toBeNull();
  });

  it("modo integrado (sem consulta nem paciente): vai só o atendimento", async () => {
    renderWithProviders(<ProceduresPanel unit={UPA} attendanceId="a3" today={TODAY22} searchDelayMs={0} />);
    const panel = screen.getByRole("region", { name: "Procedimentos realizados" });
    await within(panel).findByText("faturável · BPA-I");
    await register(panel);
    await waitFor(() => expect(api.recordProcedure).toHaveBeenCalledWith({
      establishment: { type: "health_unit", id: "u7" }, performed_on: TODAY22, sigtap_code: "0401010066", quantity: 1,
      character: "elective", attendance_id: "a3"
    }));
  });

  it("cancelar: só o próprio, com motivo; 409 already_exported relê e diz por quê", async () => {
    mocked(api.listProcedures).mockResolvedValue({ procedures: [
      procedure({ id: "pp1" }),
      procedure({ id: "pp3", professional: { user_id: "us9", name: "Dr. Caio Mendes", cbo_code: "225125" } })
    ], next_before: null });
    renderWithProviders(<ProceduresPanel unit={UPA} attendanceId="a1" today={TODAY22} />);
    const panel = screen.getByRole("region", { name: "Procedimentos realizados" });
    await waitFor(() => expect(within(panel).getAllByRole("button", { name: "Cancelar procedimento" })).toHaveLength(1));
    fireEvent.click(within(panel).getByRole("button", { name: "Cancelar procedimento" }));
    const form = within(panel).getByRole("region", { name: "Cancelar 0301060061" });
    fireEvent.change(within(form).getByLabelText("Motivo do cancelamento"), { target: { value: "errado" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar cancelamento" }));
    expect(within(form).getByRole("alert").textContent).toBe("motivo com pelo menos 10 caracteres");
    expect(api.cancelProcedure).not.toHaveBeenCalled();
    mocked(api.cancelProcedure).mockRejectedValueOnce(new api.ApiError(409, { error: "already_exported" }, "x"));
    fireEvent.change(within(form).getByLabelText("Motivo do cancelamento"), { target: { value: "lançado no paciente errado" } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar cancelamento" }));
    expect((await within(form).findByRole("alert")).textContent).toBe(
      "este procedimento já está no arquivo de uma competência fechada — para cancelar, o faturista reabre a competência"
    );
    expect(api.cancelProcedure).toHaveBeenCalledWith("pp1", "lançado no paciente errado");
    await waitFor(() => expect(api.listProcedures).toHaveBeenCalledTimes(2));
  });

  it("cancelado com sucesso: frase e lista relida; cancelado não oferece cancelar de novo", async () => {
    mocked(api.cancelProcedure).mockResolvedValue(procedure({ status: "cancelled" }));
    renderWithProviders(<ProceduresPanel unit={UPA} attendanceId="a1" today={TODAY22} />);
    const panel = screen.getByRole("region", { name: "Procedimentos realizados" });
    fireEvent.click(await within(panel).findByRole("button", { name: "Cancelar procedimento" }));
    fireEvent.change(within(panel).getByLabelText("Motivo do cancelamento"), { target: { value: "lançado no paciente errado" } });
    mocked(api.listProcedures).mockResolvedValue({ procedures: [ procedure({ status: "cancelled", cancel_reason: "lançado no paciente errado" }) ], next_before: null });
    fireEvent.click(within(panel).getByRole("button", { name: "Confirmar cancelamento" }));
    expect((await within(panel).findByRole("status")).textContent).toBe("Procedimento 0301060061 cancelado.");
    expect(await within(panel).findByText("cancelado")).not.toBeNull();
    expect(within(panel).queryByRole("button", { name: "Cancelar procedimento" })).toBeNull();
  });

  it("interruptor desligado no meio: a frase do faturamento desligado", async () => {
    mocked(api.listProcedures).mockRejectedValue(new api.ApiError(403, { error: "feature_disabled", feature: "procedure_billing" }, "x"));
    renderWithProviders(<ProceduresPanel unit={UPA} attendanceId="a1" today={TODAY22} />);
    expect((await screen.findByRole("alert")).textContent).toBe("o faturamento de procedimentos está desligado nesta cidade");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/billing/ProceduresPanel.test.tsx`
Expected: FAIL — `./ProceduresPanel` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/ProcedureTable.tsx
// Procedimentos com a crítica (contrato §2): "faturável · BPA-I" ou "não
// faturável — motivo"; o cancelado fica na lista com a situação. Cancelar pede
// motivo (correção = cancelar e registrar de novo; spec §4). `billing` mostra
// o que o faturista precisa: paciente, estabelecimento, profissional e origem.
import { useState } from "react";
import { cancelProcedure, type PerformedProcedure } from "../../lib/api";
import { billingError, characterLabel, establishmentLabel, isStale, reasonProblem, sourceLabel } from "../../lib/billing";
import { fmtDay } from "../../lib/exams";
import { DataTable, type Column } from "../../components/DataTable";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, row } from "../exams/styles";
import { ProcedureVerdict } from "./ProcedureVerdict";

interface Props {
  rows: PerformedProcedure[];
  billing?: boolean;
  canCancel(p: PerformedProcedure): boolean;
  onChanged(message: string): void;
  // 404/409 ao cancelar: a lista mudou por outra pessoa; quem chama relê.
  onStale?(): void;
}

export function ProcedureTable({ rows, billing = false, canCancel, onChanged, onStale }: Props) {
  const [ cancelling, setCancelling ] = useState<PerformedProcedure | null>(null);
  const billingCols: Column<PerformedProcedure>[] = billing ? [
    { label: "Paciente", w: "1.2fr", render: (p) => p.patient.display_name },
    { label: "Estabelecimento", w: "1.3fr", render: (p) => establishmentLabel(p.establishment) },
    { label: "Profissional", w: "1.3fr", render: (p) => `${p.professional.name}${p.professional.cbo_code ? ` · CBO ${p.professional.cbo_code}` : ""}` },
    { label: "Origem", w: "0.9fr", render: (p) => sourceLabel(p.source) }
  ] : [];
  const cols: Column<PerformedProcedure>[] = [
    { label: "Data", w: "0.8fr", render: (p) => fmtDay(p.performed_on) },
    { label: "Procedimento", w: "2fr", card: "title", render: (p) => (
      <span style={{ display: "flex", flexDirection: "column" }}>
        <span>{`${p.sigtap_code} — ${p.label}`}</span>
        <span className="mono" style={{ fontSize: 11, color: "var(--ink2)" }}>
          {`qtd ${p.quantity}${p.cid10 ? ` · CID ${p.cid10}` : ""} · ${characterLabel(p.character)}`}
        </span>
      </span>
    ) },
    ...billingCols,
    { label: "Situação", w: "1.8fr", render: (p) => <ProcedureVerdict p={p} /> },
    { label: "", w: "auto", align: "right", card: "actions", render: (p) => (canCancel(p) && p.status === "recorded" ? (
      <button type="button" style={secondaryButtonStyle} onClick={() => setCancelling(p)}>Cancelar procedimento</button>
    ) : null) }
  ];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
      <DataTable<PerformedProcedure> cols={cols} rows={rows} rowKey={(p) => p.id} empty="nenhum procedimento registrado" />
      {cancelling && (
        <CancelForm key={cancelling.id} p={cancelling} onClose={() => setCancelling(null)} onStale={onStale}
          onDone={(message) => { setCancelling(null); onChanged(message); }} />
      )}
    </div>
  );
}

function CancelForm({ p, onClose, onDone, onStale }: { p: PerformedProcedure; onClose(): void; onDone(message: string): void; onStale?(): void }) {
  const [ reason, setReason ] = useState("");
  const [ error, setError ] = useState<string | null>(null);

  async function confirm() {
    const problem = reasonProblem(reason);
    if (problem) { setError(problem); return; }
    setError(null);
    try {
      await cancelProcedure(p.id, reason.trim());
      onDone(`Procedimento ${p.sigtap_code} cancelado.`);
    } catch (err) {
      setError(billingError(err));
      if (isStale(err)) onStale?.();
    }
  }

  return (
    <section aria-label={`Cancelar ${p.sigtap_code}`} style={box}>
      <label style={fieldLabel}>
        Motivo do cancelamento
        <textarea value={reason} rows={2} style={inputStyle} onChange={(e) => { setError(null); setReason(e.target.value); }} />
      </label>
      {error && <p role="alert" style={alert}>{error}</p>}
      <div style={row}>
        <ActionButton variant="danger" onClick={confirm}>Confirmar cancelamento</ActionButton>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Voltar</button>
      </div>
    </section>
  );
}
```

```tsx
// src/modules/billing/ProceduresPanel.tsx
// Procedimentos realizados no atendimento chamado (contrato §3; spec §5.1):
// a lista do atendimento (Divergência B2) e o registro, com a data de hoje.
// Na consulta vão também o id dela e o paciente; no modo integrado, só o
// atendimento (Divergência B1). Lido ao abrir e depois de cada ação; nada
// relê sozinho. O profissional cancela só os próprios (o api confere).
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listProcedures, recordProcedure, type HealthUnit } from "../../lib/api";
import { CARE_PROCEDURES_KEY, billingError, procedureBody, savedText } from "../../lib/billing";
import { todayInCity } from "../../lib/campaigns";
import { useAuth } from "../../lib/auth";
import { buttonStyle } from "../../components/formStyles";
import { alert, muted, panel, row } from "../exams/styles";
import { ProcedureForm } from "./ProcedureForm";
import { ProcedureTable } from "./ProcedureTable";

interface Props {
  unit: HealthUnit;
  attendanceId: string;
  consultationId?: string | null;
  patientId?: string | null;
  today?: string;
  searchDelayMs?: number;
}

export function ProceduresPanel({ unit, attendanceId, consultationId = null, patientId = null, today = todayInCity(), searchDelayMs }: Props) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const queryKey = [ CARE_PROCEDURES_KEY, "attendance", attendanceId ];
  const list = useQuery({ queryKey, queryFn: () => listProcedures({ attendanceId }), gcTime: 0, staleTime: 0, retry: false });
  const [ adding, setAdding ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const reload = () => void queryClient.invalidateQueries({ queryKey });

  return (
    <section aria-label="Procedimentos realizados" style={panel}>
      <div style={row}>
        <strong style={{ flex: 1 }}>Procedimentos realizados</strong>
        {!adding && (
          <button type="button" style={buttonStyle} onClick={() => { setNotice(null); setAdding(true); }}>Registrar procedimento</button>
        )}
      </div>
      <p style={muted}>{`${unit.name} · a conferência com a SIGTAP não impede o registro`}</p>
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      {list.isPending && <p className="mono" style={muted}>carregando…</p>}
      {list.isError && <p role="alert" style={alert}>{billingError(list.error)}</p>}
      {list.data && (
        <ProcedureTable rows={list.data.procedures} canCancel={(p) => !!user && p.professional.user_id === user.id}
          onChanged={(message) => { setNotice(message); reload(); }} onStale={reload} />
      )}
      {adding && (
        <ProcedureForm today={today} dateEditable={false} searchDelayMs={searchDelayMs}
          submit={(draft) => recordProcedure(procedureBody(draft, {
            establishment: { type: "health_unit", id: unit.id }, patientId, attendanceId, consultationId
          }))}
          onSaved={(p) => { setAdding(false); setNotice(savedText(p)); reload(); }}
          onCancel={() => setAdding(false)} />
      )}
    </section>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/billing && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/billing/ProcedureTable.tsx src/modules/billing/ProceduresPanel.tsx src/modules/billing/ProceduresPanel.test.tsx
$G commit -m "feat: add the performed procedures panel with cancellation by reason

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Procedimentos no atendimento chamado (modo integrado) e na consulta

**Files:**
- Modify: `src/modules/attendance/UnitQueue.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/attendance/UnitQueue.procedures.test.tsx`, `src/modules/consultation/ConsultationWorkspace.procedures.test.tsx`

**Interfaces:**
- Consumes: `ProceduresPanel` (Task 6); `canUseBilling`, `unitBills` (Task 2).
- Produces: `UnitQueue` ganha a prop `procedureBilling?: boolean` (padrão `false`) e, na linha de "Em atendimento", o botão "Procedimentos" quando `canCare && procedureBilling && !clinicalRecord && unitBills(unit)`, que abre o painel com "Fechar procedimentos"; `ConsultationWorkspace` mostra o `ProceduresPanel` (com `patientId` do prontuário e `consultationId` da consulta aberta, se houver) na aba Consulta quando `canUseBilling(user) && unitBills(unit)`. Com o prontuário ligado o painel vive só na consulta (como a linha do tempo do 20).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/UnitQueue.procedures.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listUnitQueue: vi.fn(), listProcedures: vi.fn() };
});

import * as api from "../../lib/api";
import { AuthProvider } from "../../lib/auth";
import { UnitQueue } from "./UnitQueue";
import { UBS, UPA, billingUser, procedure } from "../../test/billingFixtures";

afterEach(cleanup);
const m = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const inCare = [ {
  id: "a3", cpf_masked: "***.333.444-**", checked_in_at: "2026-10-21T08:00:00-03:00", protocol_name: "triage-dor", priority: 3,
  source: "triage" as const, appointment_time: null, called_at: "2026-10-21T09:00:00-03:00", called_by_name: "Dra. Helena Prado"
} ];

function renderQueue(unit: api.HealthUnit, props: { canCare: boolean; clinicalRecord?: boolean; procedureBilling?: boolean }) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<UnitQueue unit={unit} units={[ unit ]} {...props} />, { wrapper });
}

describe("UnitQueue — procedimentos (módulo 22)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listUnitQueue, api.listProcedures ]) m(fn).mockReset();
    m(api.fetchCurrentSession).mockResolvedValue(billingUser([ "health_professional" ], { id: "us1" }));
    m(api.listUnitQueue).mockResolvedValue({ waiting: [], in_care: inCare });
    m(api.listProcedures).mockResolvedValue({ procedures: [ procedure() ], next_before: null });
  });

  it("UPA, modo integrado: Procedimentos no atendimento chamado abre o painel do atendimento", async () => {
    renderQueue(UPA, { canCare: true, procedureBilling: true });
    fireEvent.click(await screen.findByRole("button", { name: "Procedimentos" }));
    expect(screen.getByRole("region", { name: "Procedimentos realizados" })).not.toBeNull();
    expect(await screen.findByText("faturável · BPA-I")).not.toBeNull();
    expect(api.listProcedures).toHaveBeenCalledWith({ attendanceId: "a3" });
    fireEvent.click(screen.getByRole("button", { name: "Fechar procedimentos" }));
    expect(screen.queryByRole("region", { name: "Procedimentos realizados" })).toBeNull();
  });

  it("UBS: sem o botão (a produção da APS vai pelo e-SUS)", async () => {
    renderQueue(UBS, { canCare: true, procedureBilling: true });
    await screen.findByText("***.333.444-**");
    expect(screen.queryByRole("button", { name: "Procedimentos" })).toBeNull();
  });

  it("sem o interruptor, sem papel clínico ou com o prontuário (o painel fica na consulta): sem o botão", async () => {
    renderQueue(UPA, { canCare: true });
    await screen.findByText("***.333.444-**");
    expect(screen.queryByRole("button", { name: "Procedimentos" })).toBeNull();
    cleanup();
    renderQueue(UPA, { canCare: false, procedureBilling: true });
    await screen.findByText("***.333.444-**");
    expect(screen.queryByRole("button", { name: "Procedimentos" })).toBeNull();
    cleanup();
    renderQueue(UPA, { canCare: true, procedureBilling: true, clinicalRecord: true });
    expect(await screen.findByRole("button", { name: "Consulta" })).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Procedimentos" })).toBeNull();
    expect(api.listProcedures).not.toHaveBeenCalled();
  });
});
```

```tsx
// src/modules/consultation/ConsultationWorkspace.procedures.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn(),
    startConsultation: vi.fn(), getConsultation: vi.fn(), getConsultationProblems: vi.fn() };
});
vi.mock("./ConsultationEditor", () => ({ ConsultationEditor: () => <span>editor (dublê)</span> }));
vi.mock("../billing/ProceduresPanel", () => ({
  ProceduresPanel: (p: { attendanceId: string; consultationId?: string | null; patientId?: string | null; unit: { id: string } }) =>
    <span>{`procedimentos ${p.unit.id} ${p.attendanceId} ${p.consultationId ?? "sem consulta"} ${p.patientId ?? "sem paciente"}`}</span>
}));

import * as api from "../../lib/api";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { consultation, options, problem, record } from "../../test/consultationFixtures";
import { UBS, UPA } from "../../test/billingFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("ConsultationWorkspace — procedimentos (módulo 22)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.getAttendanceRecord, api.getConsultationOptions, api.startConsultation, api.getConsultationProblems ]) mocked(fn).mockReset();
    mocked(api.getAttendanceRecord).mockResolvedValue(record());
    mocked(api.getConsultationOptions).mockResolvedValue(options());
    mocked(api.startConsultation).mockResolvedValue(consultation());
    mocked(api.getConsultationProblems).mockResolvedValue([ problem() ]);
  });

  const open = (unit: api.HealthUnit) => renderWithProviders(
    <ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={() => undefined} onFinalized={() => undefined} />
  );

  it("UPA com procedure_billing: o painel aparece antes de iniciar e, depois, leva a consulta", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record", "procedure_billing" ] }));
    open(UPA);
    expect(await screen.findByText("procedimentos u7 a1 sem consulta pa1")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Iniciar consulta" }));
    expect(await screen.findByText("procedimentos u7 a1 cs1 pa1")).not.toBeNull();
  });

  it("UBS ou sem o interruptor: sem o painel", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record", "procedure_billing" ] }));
    open(UBS);
    await screen.findByRole("button", { name: "Iniciar consulta" });
    expect(screen.queryByText(/^procedimentos /)).toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features: [ "clinical_record" ] }));
    open(UPA);
    await screen.findByRole("button", { name: "Iniciar consulta" });
    expect(screen.queryByText(/^procedimentos /)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/attendance/UnitQueue.procedures.test.tsx src/modules/consultation/ConsultationWorkspace.procedures.test.tsx`
Expected: FAIL — não há botão "Procedimentos" nem o painel na consulta.

- [ ] **Step 3: Write minimal implementation**

`src/modules/attendance/UnitQueue.tsx`:

1. Imports — depois de `import { PatientTimeline } from "../timeline/PatientTimeline";`:

```ts
import { ProceduresPanel } from "../billing/ProceduresPanel";
import { unitBills } from "../../lib/billing";
```

2. Em `Props`, depois do bloco de `patientTimeline?: boolean;`:

```ts
  // Módulo 22: `procedure_billing` ligado na sessão. Sem o prontuário (modo
  // integrated), os procedimentos abrem pelo atendimento chamado; nunca na UBS.
  procedureBilling?: boolean;
```

3. No tipo `ScreeningPanel`, troque

```ts
  | { kind: "timeline"; attendanceId: string };
```

por

```ts
  | { kind: "timeline"; attendanceId: string }
  // Módulo 22: os procedimentos do atendimento chamado, quando não há prontuário.
  | { kind: "procedures"; attendanceId: string };
```

4. Na assinatura, troque `  clinicalDocuments = false, professional = false, patientTimeline = false` por

```ts
  clinicalDocuments = false, professional = false, patientTimeline = false, procedureBilling = false
```

e, depois de `const timelineButton = canCare && patientTimeline && !clinicalRecord;`:

```ts
  const proceduresButton = canCare && procedureBilling && !clinicalRecord && unitBills(unit);
```

5. Nas ações de "Em atendimento", logo depois do bloco `{timelineButton && ( … Linha do tempo … )}`:

```tsx
                          {proceduresButton && (
                            <button type="button" style={secondaryButtonStyle}
                              onClick={() => setPanel({ kind: "procedures", attendanceId: r.id })}>
                              Procedimentos
                            </button>
                          )}
```

6. Logo depois do bloco `{timelineButton && panel?.kind === "timeline" && ( … )}`:

```tsx
        {proceduresButton && panel?.kind === "procedures" && (
          <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
            <ProceduresPanel key={panel.attendanceId} unit={unit} attendanceId={panel.attendanceId} />
            <div><button type="button" style={secondaryButtonStyle} onClick={() => setPanel(null)}>Fechar procedimentos</button></div>
          </div>
        )}
```

`src/modules/consultation/ConsultationWorkspace.tsx`:

1. Imports — depois de `import { ConsultationLoader, ConsultationView } from "./ConsultationView";`:

```ts
import { ProceduresPanel } from "../billing/ProceduresPanel";
import { canUseBilling, unitBills } from "../../lib/billing";
```

2. Depois de `const examsOn = hasFeature(user, "exam_tracking");`:

```ts
  // Módulo 22: procedimentos realizados no atendimento, nunca na UBS (ADR 0039).
  const proceduresOn = canUseBilling(user) && unitBills(props.unit);
```

3. Dentro do `<div hidden={onDocuments} …>`, logo depois do bloco `{consultation?.status === "finalized" && ( <ConsultationView … onClose={props.onClose} /> )}`:

```tsx
            {proceduresOn && (
              <ProceduresPanel unit={props.unit} attendanceId={props.attendanceId} consultationId={consultation?.id ?? null}
                patientId={record.data.patient.id} searchDelayMs={props.searchDelayMs} />
            )}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/attendance src/modules/consultation && npx tsc --noEmit`
Expected: PASS (os testes antigos da fila e da consulta seguem iguais: sem `procedure_billing`, nada muda).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/attendance/UnitQueue.tsx src/modules/attendance/UnitQueue.procedures.test.tsx src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationWorkspace.procedures.test.tsx
$G commit -m "feat: record performed procedures in the called attendance and in the consultation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Procedimento avulso pela unidade

**Files:**
- Create: `src/modules/billing/BillingPatientLookup.tsx`, `src/modules/billing/UnitProcedures.tsx`
- Modify: `src/modules/Attendance.tsx`
- Test: `src/modules/billing/UnitProcedures.test.tsx`

**Interfaces:**
- Consumes: `lookupBillingPatient`, `listProcedures`, `recordProcedure`, `BillingPatient`, `HealthUnit` (Task 1); `CARE_PROCEDURES_KEY`, `billingError`, `procedureBody`, `savedText`, `showUnitProcedures` (Task 2); `ProcedureForm` (Task 5); `ProcedureTable` (Task 6); `isValidCpf`, `maskCpf`, `onlyDigits` (`src/lib/attendance.ts`); `Panel`.
- Produces:
  - `BillingPatientLookup({ patient, onFound })` — campo "CPF do paciente", botão "Buscar paciente", e "Paciente: <nome> · CPF <mascarado>" com "Trocar paciente" (CPF no corpo de um POST, Divergência B3);
  - `UnitProcedures({ unit, today?, searchDelayMs? })` — painel "Procedimento avulso": paciente pelo CPF, a lista dos **próprios** procedimentos para ele (`listProcedures({ patientId })`, contrato §3) e o registro com data editável, sem atendimento;
  - `UnitQueue` recebe `procedureBilling` de `Attendance`; `Attendance` mostra `UnitProcedures` quando `showUnitProcedures({ user, canCare, unit })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/billing/UnitProcedures.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), lookupBillingPatient: vi.fn(), listProcedures: vi.fn(), recordProcedure: vi.fn(), searchBillingSigtap: vi.fn() };
});

import * as api from "../../lib/api";
import { UnitProcedures } from "./UnitProcedures";
import { renderWithProviders } from "../../test/campaignFixtures";
import { PATIENT, TODAY22, UPA, URGENCY, billingUser, procedure } from "../../test/billingFixtures";
import { VALID_CPF } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.lookupBillingPatient, api.listProcedures, api.recordProcedure, api.searchBillingSigtap ];
afterEach(cleanup);
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "health_professional" ], { id: "us1" }));
  mocked(api.lookupBillingPatient).mockResolvedValue(PATIENT);
  mocked(api.listProcedures).mockResolvedValue({ procedures: [ procedure({ source: "manual", source_ref: null }) ], next_before: null });
  mocked(api.searchBillingSigtap).mockResolvedValue([ URGENCY ]);
  mocked(api.recordProcedure).mockResolvedValue(procedure({ id: "pp9", performed_on: "2026-10-20" }));
});

async function findPatient() {
  const panel = await screen.findByRole("region", { name: "Procedimento avulso" });
  fireEvent.change(within(panel).getByLabelText("CPF do paciente"), { target: { value: VALID_CPF } });
  fireEvent.click(within(panel).getByRole("button", { name: "Buscar paciente" }));
  await within(panel).findByText("Paciente: Joana Lima · CPF ***.982.247-**");
  return panel;
}

describe("UnitProcedures", () => {
  it("acha o paciente pelo CPF (no corpo), lista os próprios e registra com data editável e sem atendimento", async () => {
    renderWithProviders(<UnitProcedures unit={UPA} today={TODAY22} searchDelayMs={0} />);
    expect(api.listProcedures).not.toHaveBeenCalled();
    const panel = await findPatient();
    expect(api.lookupBillingPatient).toHaveBeenCalledWith(VALID_CPF);
    await waitFor(() => expect(api.listProcedures).toHaveBeenCalledWith({ patientId: "pa1" }));
    expect(await within(panel).findByText("faturável · BPA-I")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Registrar procedimento" }));
    const form = within(panel).getByRole("region", { name: "Registrar procedimento" });
    fireEvent.change(within(form).getByLabelText("Data do procedimento"), { target: { value: "2026-10-20" } });
    fireEvent.change(within(form).getByLabelText("Procedimento (SIGTAP)"), { target: { value: "urgencia" } });
    fireEvent.click(await within(form).findByRole("button", { name: /^0301060061 — / }));
    fireEvent.click(within(form).getByRole("button", { name: "Registrar procedimento" }));
    await waitFor(() => expect(api.recordProcedure).toHaveBeenCalledWith({
      establishment: { type: "health_unit", id: "u7" }, performed_on: "2026-10-20", sigtap_code: "0301060061", quantity: 1,
      character: "elective", patient_id: "pa1"
    }));
    expect((await within(panel).findByRole("status")).textContent).toBe("Procedimento registrado — faturável.");
    await waitFor(() => expect(api.listProcedures).toHaveBeenCalledTimes(2));
  });

  it("CPF inválido não busca; 404 patient_not_found vira frase; trocar paciente limpa a lista", async () => {
    renderWithProviders(<UnitProcedures unit={UPA} today={TODAY22} searchDelayMs={0} />);
    const panel = await screen.findByRole("region", { name: "Procedimento avulso" });
    fireEvent.change(within(panel).getByLabelText("CPF do paciente"), { target: { value: "111.111.111-11" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Buscar paciente" }));
    expect(within(panel).getByRole("alert").textContent).toBe("CPF inválido");
    expect(api.lookupBillingPatient).not.toHaveBeenCalled();
    mocked(api.lookupBillingPatient).mockRejectedValueOnce(new api.ApiError(404, { error: "patient_not_found" }, "x"));
    fireEvent.change(within(panel).getByLabelText("CPF do paciente"), { target: { value: VALID_CPF } });
    fireEvent.click(within(panel).getByRole("button", { name: "Buscar paciente" }));
    expect((await within(panel).findByRole("alert")).textContent).toBe("paciente não encontrado — confira o CPF");
    fireEvent.click(within(panel).getByRole("button", { name: "Buscar paciente" }));
    await within(panel).findByText("Paciente: Joana Lima · CPF ***.982.247-**");
    fireEvent.click(within(panel).getByRole("button", { name: "Trocar paciente" }));
    expect(within(panel).queryByText("faturável · BPA-I")).toBeNull();
    expect(within(panel).queryByRole("button", { name: "Registrar procedimento" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/billing/UnitProcedures.test.tsx`
Expected: FAIL — `./UnitProcedures` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/BillingPatientLookup.tsx
// O paciente pelo CPF para o faturamento (Divergência B3): CPF no corpo de um
// POST, nunca na URL; a tela guarda só o id, o nome de exibição e o CPF
// mascarado. Uma busca por vez.
import { useRef, useState } from "react";
import { lookupBillingPatient, type BillingPatient } from "../../lib/api";
import { isValidCpf, maskCpf, onlyDigits } from "../../lib/attendance";
import { billingError } from "../../lib/billing";
import { ActionButton } from "../../components/ActionButton";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, fieldLabel, row } from "../exams/styles";

export function BillingPatientLookup({ patient, onFound }: { patient: BillingPatient | null; onFound(p: BillingPatient | null): void }) {
  const [ cpf, setCpf ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);

  async function find() {
    if (running.current) return;
    if (!isValidCpf(cpf)) { setError("CPF inválido"); return; }
    running.current = true;
    setBusy(true);
    setError(null);
    try {
      const found = await lookupBillingPatient(onlyDigits(cpf));
      setCpf("");
      onFound(found);
    } catch (err) {
      setError(billingError(err));
    } finally {
      running.current = false;
      setBusy(false);
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
          <input value={cpf} inputMode="numeric" autoComplete="off" style={inputStyle} onChange={(e) => { setError(null); setCpf(maskCpf(e.target.value)); }}
            onKeyDown={(e) => { if (e.key === "Enter") { e.preventDefault(); void find(); } }} />
        </label>
        <ActionButton variant="secondary" busy={busy} onClick={find}>Buscar paciente</ActionButton>
      </div>
      {error && <p role="alert" style={alert}>{error}</p>}
    </div>
  );
}
```

```tsx
// src/modules/billing/UnitProcedures.tsx
// Procedimento avulso pela unidade (spec §5.1): fora de atendimento, para um
// paciente achado pelo CPF. Só em unidade que fatura BPA (quem chama confere
// `showUnitProcedures`). A lista é a dos procedimentos do próprio profissional
// para o paciente (contrato §3: profissional só os próprios por patient_id).
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { listProcedures, recordProcedure, type BillingPatient, type HealthUnit } from "../../lib/api";
import { CARE_PROCEDURES_KEY, billingError, procedureBody, savedText } from "../../lib/billing";
import { todayInCity } from "../../lib/campaigns";
import { useAuth } from "../../lib/auth";
import { Panel } from "../../components/Panel";
import { buttonStyle } from "../../components/formStyles";
import { alert, muted, stack } from "../exams/styles";
import { BillingPatientLookup } from "./BillingPatientLookup";
import { ProcedureForm } from "./ProcedureForm";
import { ProcedureTable } from "./ProcedureTable";

export function UnitProcedures({ unit, today = todayInCity(), searchDelayMs }: { unit: HealthUnit; today?: string; searchDelayMs?: number }) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const [ patient, setPatient ] = useState<BillingPatient | null>(null);
  const [ adding, setAdding ] = useState(false);
  const [ notice, setNotice ] = useState<string | null>(null);
  const queryKey = [ CARE_PROCEDURES_KEY, "patient", patient?.id ?? null ];
  const list = useQuery({
    queryKey, queryFn: () => listProcedures({ patientId: patient!.id }), enabled: !!patient, gcTime: 0, staleTime: 0, retry: false
  });
  const reload = () => void queryClient.invalidateQueries({ queryKey });

  // O Panel é a região "Procedimento avulso" (aria-label = título).
  return (
    <Panel title="Procedimento avulso" sub={`fora do atendimento · ${unit.name}`}>
      <div style={{ ...stack, padding: 16 }}>
        <BillingPatientLookup patient={patient} onFound={(p) => { setPatient(p); setAdding(false); setNotice(null); }} />
        {notice && <p role="status" style={statusStyle}>{notice}</p>}
        {patient && list.isPending && <p className="mono" style={muted}>carregando…</p>}
        {patient && list.isError && <p role="alert" style={alert}>{billingError(list.error)}</p>}
        {patient && list.data && (
          <ProcedureTable rows={list.data.procedures} canCancel={(p) => !!user && p.professional.user_id === user.id}
            onChanged={(message) => { setNotice(message); reload(); }} onStale={reload} />
        )}
        {patient && !adding && (
          <div><button type="button" style={buttonStyle} onClick={() => { setNotice(null); setAdding(true); }}>Registrar procedimento</button></div>
        )}
        {patient && adding && (
          <ProcedureForm today={today} dateEditable searchDelayMs={searchDelayMs}
            submit={(draft) => recordProcedure(procedureBody(draft, { establishment: { type: "health_unit", id: unit.id }, patientId: patient.id }))}
            onSaved={(p) => { setAdding(false); setNotice(savedText(p)); reload(); }}
            onCancel={() => setAdding(false)} />
        )}
      </div>
    </Panel>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

`src/modules/Attendance.tsx`:

1. Imports — depois de `import { hasFeature } from "../lib/features";`:

```ts
import { showUnitProcedures } from "../lib/billing";
import { UnitProcedures } from "./billing/UnitProcedures";
```

2. Na `<UnitQueue …>`, depois de `patientTimeline={hasFeature(user, "patient_timeline")}`:

```tsx
          procedureBilling={hasFeature(user, "procedure_billing")}
```

3. Logo antes de `{canVerify && unit && <div id="sec-pedidos"><Requests unit={unit} /></div>}`:

```tsx
      {/* Módulo 22: procedimento avulso, só do profissional com vínculo e nunca na UBS. */}
      {unit && showUnitProcedures({ user, canCare, unit }) && <UnitProcedures key={unit.id} unit={unit} />}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/billing src/modules/Attendance*.test.tsx src/modules/attendance && npx tsc --noEmit`
Expected: PASS (os testes antigos do Atendimento seguem iguais: a sessão deles não tem `procedure_billing`).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/billing/BillingPatientLookup.tsx src/modules/billing/UnitProcedures.tsx src/modules/billing/UnitProcedures.test.tsx src/modules/Attendance.tsx
$G commit -m "feat: record walk-in procedures by unit outside the attendance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Portão do faturamento e a lista de procedimentos com filtros

**Files:**
- Create: `src/modules/billing/BillingGate.tsx`, `src/modules/BillingProcedures.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/BillingProcedures.test.tsx`

**Interfaces:**
- Consumes: `listBillingEstablishments`, `listProcedures`, `BillingEstablishment`, `BillingPatient` (Task 1); `BILLING_DISABLED`, `BILLING_NO_ROLE`, `ESTABLISHMENTS_KEY`, `PROCEDURES_KEY`, `REASON_CODES`, `EMPTY_FILTERS`, `billingError`, `billingRoles`, `canUseBilling`, `establishmentKey`, `establishmentLabel`, `procedureQuery`, `reasonLabel`, `BillingRoles`, `ProcedureFilters` (Task 2); `ProcedureTable` (Task 6); `BillingPatientLookup` (Task 8); `competenceOptions`, `competenceLabel` (`competence.ts`); `todayInCity`.
- Produces:
  - `BillingGate({ title, sub, allow(roles), children(roles) })` — `PageHeader` + estado vazio `BILLING_DISABLED` (interruptor) ou `BILLING_NO_ROLE` (papel);
  - `BillingProcedures({ today? })` — tela "Procedimentos realizados": filtros "Estabelecimento", "Competência", "Crítica", "Motivo" (só com "não faturáveis") e o paciente pelo CPF; botões "Filtrar", "Limpar filtros" e "Carregar mais"; o faturista cancela qualquer procedimento; o admin só lê. Dentro, `ProceduresBody({ clerk, today })` com o ponto de extensão `headerActions` usado pela Task 10;
  - `App.tsx`: `case "billing-procedures": return <BillingProcedures />;`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/BillingProcedures.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listBillingEstablishments: vi.fn(), listProcedures: vi.fn(), cancelProcedure: vi.fn(),
    lookupBillingPatient: vi.fn() };
});

import * as api from "../lib/api";
import { BillingProcedures } from "./BillingProcedures";
import { BILLING_DISABLED, BILLING_NO_ROLE } from "../lib/billing";
import { renderWithProviders } from "../test/campaignFixtures";
import { GLUCOSE, LAB_EST, PATIENT, TODAY22, UPA_EST, billingUser, procedure } from "../test/billingFixtures";
import { VALID_CPF } from "../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listBillingEstablishments, api.listProcedures, api.cancelProcedure, api.lookupBillingPatient ];
const LAB_ROW = procedure({
  id: "pp4", source: "lab_report", source_ref: "it7", establishment: LAB_EST, billable: false, blocking_reasons: [ "missing_professional_cns" ],
  professional: { user_id: "us5", name: "Bia Farmacêutica", cbo_code: "221205" }, sigtap_code: GLUCOSE.code, label: GLUCOSE.name, instrument: "bpa_c"
});
afterEach(cleanup);
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ]));
  mocked(api.listBillingEstablishments).mockResolvedValue([ UPA_EST, LAB_EST ]);
  mocked(api.listProcedures).mockResolvedValue({ procedures: [ procedure(), LAB_ROW ], next_before: "c1" });
  mocked(api.lookupBillingPatient).mockResolvedValue(PATIENT);
});

describe("BillingProcedures", () => {
  it("abre na competência corrente; filtra por estabelecimento, crítica e motivo; só códigos na consulta", async () => {
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    expect(await screen.findByText("0202010473 — DOSAGEM DE GLICOSE")).not.toBeNull();
    expect(api.listProcedures).toHaveBeenLastCalledWith({ competence: "202610", before: null });
    expect(screen.getByText("laboratório municipal")).not.toBeNull();
    // Na opção do filtro e na linha do procedimento.
    await waitFor(() => expect(screen.getAllByText("Laboratório Municipal · CNES 7654321")).toHaveLength(2));
    expect(screen.getByText("Bia Farmacêutica · CBO 221205")).not.toBeNull();
    expect(screen.getByText("não faturável — profissional sem CNS no cadastro")).not.toBeNull();
    expect(screen.queryByLabelText("Motivo")).toBeNull();
    fireEvent.change(await screen.findByLabelText("Estabelecimento"), { target: { value: "lab_provider:pv9" } });
    fireEvent.change(screen.getByLabelText("Competência"), { target: { value: "202609" } });
    fireEvent.change(screen.getByLabelText("Crítica"), { target: { value: "false" } });
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "missing_professional_cns" } });
    expect(api.listProcedures).toHaveBeenCalledTimes(1);
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    await waitFor(() => expect(api.listProcedures).toHaveBeenLastCalledWith({
      establishmentType: "lab_provider", establishmentId: "pv9", competence: "202609", billable: false, reason: "missing_professional_cns", before: null
    }));
    fireEvent.click(screen.getByRole("button", { name: "Limpar filtros" }));
    await waitFor(() => expect(api.listProcedures).toHaveBeenLastCalledWith({ competence: "202610", before: null }));
  });

  it("Carregar mais usa o cursor e junta as páginas", async () => {
    mocked(api.listProcedures)
      .mockResolvedValueOnce({ procedures: [ procedure() ], next_before: "c1" })
      .mockResolvedValueOnce({ procedures: [ LAB_ROW ], next_before: null });
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    fireEvent.click(await screen.findByRole("button", { name: "Carregar mais" }));
    expect(await screen.findByText("0202010473 — DOSAGEM DE GLICOSE")).not.toBeNull();
    expect(api.listProcedures).toHaveBeenLastCalledWith({ competence: "202610", before: "c1" });
    expect(screen.getByText("0301060061 — ATENDIMENTO DE URGENCIA EM ATENCAO ESPECIALIZADA")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Carregar mais" })).toBeNull();
  });

  it("filtro por paciente: CPF no corpo da busca; na lista, só o id", async () => {
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    await screen.findByText("0202010473 — DOSAGEM DE GLICOSE");
    const filters = screen.getByRole("group", { name: "filtros dos procedimentos" });
    fireEvent.change(within(filters).getByLabelText("CPF do paciente"), { target: { value: VALID_CPF } });
    fireEvent.click(within(filters).getByRole("button", { name: "Buscar paciente" }));
    await within(filters).findByText("Paciente: Joana Lima · CPF ***.982.247-**");
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    await waitFor(() => expect(api.listProcedures).toHaveBeenLastCalledWith({ competence: "202610", patientId: "pa1", before: null }));
    expect(api.lookupBillingPatient).toHaveBeenCalledWith(VALID_CPF);
  });

  it("faturista cancela qualquer procedimento; o admin só lê", async () => {
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    await screen.findByText("0202010473 — DOSAGEM DE GLICOSE");
    expect(screen.getAllByRole("button", { name: "Cancelar procedimento" })).toHaveLength(2);
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "municipal_admin" ]));
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    await screen.findByText("0202010473 — DOSAGEM DE GLICOSE");
    expect(screen.queryByRole("button", { name: "Cancelar procedimento" })).toBeNull();
  });

  it("sem o interruptor ou sem papel: estado vazio e nenhuma leitura", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ], { features: [] }));
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    expect(await screen.findByText(BILLING_DISABLED)).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "health_professional" ]));
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    expect(await screen.findByText(BILLING_NO_ROLE)).not.toBeNull();
    expect(api.listProcedures).not.toHaveBeenCalled();
    expect(api.listBillingEstablishments).not.toHaveBeenCalled();
  });

  it("lista vazia e recusa da lista", async () => {
    mocked(api.listProcedures).mockResolvedValueOnce({ procedures: [], next_before: null });
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    expect(await screen.findByText("nenhum procedimento registrado")).not.toBeNull();
    cleanup();
    mocked(api.listProcedures).mockRejectedValueOnce(new api.ApiError(403, { error: "missing_role" }, "x"));
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    expect((await screen.findByRole("alert")).textContent).toBe("seu papel não permite esta ação no faturamento");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/BillingProcedures.test.tsx`
Expected: FAIL — `./BillingProcedures` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/BillingGate.tsx
// Portão das telas do faturamento: interruptor e papel antes de qualquer
// leitura (o api recusaria de novo; aqui só não se oferece a porta).
import type { ReactNode } from "react";
import { useAuth } from "../../lib/auth";
import { BILLING_DISABLED, BILLING_NO_ROLE, billingRoles, canUseBilling, type BillingRoles } from "../../lib/billing";
import { PageHeader } from "../../components/PageHeader";
import { EmptyState } from "../../components/EmptyState";
import { stack } from "../exams/styles";

export function BillingGate({ title, sub, allow, children }: {
  title: string; sub: string; allow(roles: BillingRoles): boolean; children(roles: BillingRoles): ReactNode;
}) {
  const { user } = useAuth();
  if (!user) return null;
  const roles = billingRoles(user);
  let body: ReactNode;
  if (!canUseBilling(user)) body = <EmptyState title={BILLING_DISABLED} />;
  else if (!allow(roles)) body = <EmptyState title={BILLING_NO_ROLE} />;
  else body = children(roles);
  return (
    <div style={stack}>
      <PageHeader title={title} sub={sub} />
      {body}
    </div>
  );
}
```

```tsx
// src/modules/BillingProcedures.tsx
// Procedimentos realizados da cidade (contrato §3): o faturista e o
// municipal_admin filtram por estabelecimento, competência, crítica, motivo e
// paciente; o faturista cancela qualquer procedimento (Divergência B12). A
// lista traz paciente (trilha): lida ao abrir, ao filtrar e depois de uma
// ação — nunca sozinha. Na URL só ids, códigos e a competência.
import { useState, type CSSProperties, type ReactNode } from "react";
import { useInfiniteQuery, useQuery, useQueryClient } from "@tanstack/react-query";
import { listBillingEstablishments, listProcedures, type BillingEstablishment, type BillingPatient } from "../lib/api";
import {
  EMPTY_FILTERS, ESTABLISHMENTS_KEY, PROCEDURES_KEY, REASON_CODES, billingError, establishmentKey, establishmentLabel, procedureQuery,
  reasonLabel, type ProcedureFilters
} from "../lib/billing";
import { competenceLabel, competenceOptions } from "../lib/competence";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { alert, fieldLabel, muted, row, stack } from "./exams/styles";
import { BillingGate } from "./billing/BillingGate";
import { BillingPatientLookup } from "./billing/BillingPatientLookup";
import { ProcedureTable } from "./billing/ProcedureTable";

export function BillingProcedures({ today = todayInCity() }: { today?: string } = {}) {
  return (
    <BillingGate title="Procedimentos realizados" sub="faturamento · crítica SIGTAP, cancelamento e lançamento manual"
      allow={(r) => r.clerk || r.admin}>
      {(roles) => <ProceduresBody clerk={roles.clerk} today={today} />}
    </BillingGate>
  );
}

// `headerActions` recebe os estabelecimentos e o aviso da tela (Task 10: lançamento manual).
export function ProceduresBody({ clerk, today, headerActions }: {
  clerk: boolean; today: string;
  headerActions?(a: { establishments: BillingEstablishment[]; onSaved(message: string): void }): ReactNode;
}) {
  const queryClient = useQueryClient();
  const establishments = useQuery({ queryKey: [ ESTABLISHMENTS_KEY ], queryFn: listBillingEstablishments, retry: false });
  const [ draft, setDraft ] = useState<ProcedureFilters>(() => EMPTY_FILTERS(today));
  const [ patient, setPatient ] = useState<BillingPatient | null>(null);
  const [ applied, setApplied ] = useState<ProcedureFilters>(() => EMPTY_FILTERS(today));
  const [ notice, setNotice ] = useState<string | null>(null);
  const set = (patch: Partial<ProcedureFilters>) => setDraft((d) => ({ ...d, ...patch }));
  const query = useInfiniteQuery({
    queryKey: [ PROCEDURES_KEY, applied ],
    queryFn: ({ pageParam }) => listProcedures({ ...procedureQuery(applied), before: pageParam }),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.next_before ?? undefined,
    gcTime: 0, staleTime: 0, retry: false
  });
  const rows = (query.data?.pages ?? []).flatMap((p) => p.procedures);
  const reload = () => void queryClient.invalidateQueries({ queryKey: [ PROCEDURES_KEY ] });

  function apply() {
    setNotice(null);
    setApplied({ ...draft, patientId: patient?.id ?? null, reason: draft.billable === "false" ? draft.reason : "" });
  }

  function clear() {
    setNotice(null);
    setPatient(null);
    setDraft(EMPTY_FILTERS(today));
    setApplied(EMPTY_FILTERS(today));
  }

  return (
    <div style={stack}>
      {headerActions?.({ establishments: establishments.data ?? [], onSaved: (message) => { setNotice(message); reload(); } })}
      <Panel title="Filtros">
        <fieldset aria-label="filtros dos procedimentos" style={fieldsetStyle}>
          <div style={{ ...row, alignItems: "flex-end" }}>
            <label style={fieldLabel}>
              Estabelecimento
              <select value={draft.establishment} style={inputStyle} onChange={(e) => set({ establishment: e.target.value })}>
                <option value="">todos</option>
                {(establishments.data ?? []).map((e) => <option key={establishmentKey(e)} value={establishmentKey(e)}>{establishmentLabel(e)}</option>)}
              </select>
            </label>
            <label style={fieldLabel}>
              Competência
              <select value={draft.competence} style={inputStyle} onChange={(e) => set({ competence: e.target.value })}>
                {competenceOptions(today, draft.competence).map((c) => <option key={c} value={c}>{competenceLabel(c)}</option>)}
              </select>
            </label>
            <label style={fieldLabel}>
              Crítica
              <select value={draft.billable} style={inputStyle} onChange={(e) => set({ billable: e.target.value as ProcedureFilters["billable"] })}>
                <option value="">todos</option>
                <option value="true">faturáveis</option>
                <option value="false">não faturáveis</option>
              </select>
            </label>
            {draft.billable === "false" && (
              <label style={fieldLabel}>
                Motivo
                <select value={draft.reason} style={inputStyle} onChange={(e) => set({ reason: e.target.value })}>
                  <option value="">todos os motivos</option>
                  {REASON_CODES.map((code) => <option key={code} value={code}>{reasonLabel(code)}</option>)}
                </select>
              </label>
            )}
          </div>
          <BillingPatientLookup patient={patient} onFound={setPatient} />
          <div style={row}>
            <button type="button" style={buttonStyle} onClick={apply}>Filtrar</button>
            <button type="button" style={secondaryButtonStyle} onClick={clear}>Limpar filtros</button>
          </div>
          {establishments.isError && <p role="alert" style={alert}>{billingError(establishments.error)}</p>}
        </fieldset>
      </Panel>
      <Panel title="Procedimentos" sub={`competência ${competenceLabel(applied.competence)}`}>
        <div style={{ ...stack, padding: 16 }}>
          {notice && <p role="status" style={statusStyle}>{notice}</p>}
          {query.isPending && <p className="mono" style={muted}>carregando…</p>}
          {query.isError && <p role="alert" style={alert}>{billingError(query.error)}</p>}
          {query.data && (
            <ProcedureTable rows={rows} billing canCancel={() => clerk}
              onChanged={(message) => { setNotice(message); reload(); }} onStale={reload} />
          )}
          {query.hasNextPage && (
            <div>
              <button type="button" style={secondaryButtonStyle} disabled={query.isFetchingNextPage} onClick={() => void query.fetchNextPage()}>
                Carregar mais
              </button>
            </div>
          )}
        </div>
      </Panel>
    </div>
  );
}

const fieldsetStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, border: 0, margin: 0, padding: 16 };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

`src/App.tsx` — import, depois de `import { LabQcBlocks } from "./modules/LabQcBlocks";`:

```ts
import { BillingProcedures } from "./modules/BillingProcedures";
```

e em `renderModule`, depois de `case "lab-instruments": …`:

```tsx
    case "billing-procedures": return <BillingProcedures />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/BillingProcedures.test.tsx src/App.navigation.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/billing/BillingGate.tsx src/modules/BillingProcedures.tsx src/modules/BillingProcedures.test.tsx src/App.tsx
$G commit -m "feat: add the performed procedures list with billing filters

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Lançamento manual pelo faturista

**Files:**
- Create: `src/modules/billing/ManualEntryPanel.tsx`
- Modify: `src/modules/BillingProcedures.tsx`, `src/modules/BillingProcedures.test.tsx`
- Test: `src/modules/billing/ManualEntryPanel.test.tsx`

**Interfaces:**
- Consumes: `listBillingProfessionals`, `recordProcedure`, `BillingEstablishment`, `BillingPatient`, `PerformedProcedure` (Task 1); `PROFESSIONALS_KEY`, `billingError`, `establishmentKey`, `establishmentLabel`, `manualProblem`, `parseEstablishmentKey`, `procedureBody`, `savedText` (Task 2); `ProcedureForm` (Task 5); `BillingPatientLookup` (Task 8); `ProceduresBody` e `headerActions` (Task 9).
- Produces:
  - `ManualEntryPanel({ establishments, today, searchDelayMs?, onSaved(p), onClose() })` — região "Lançamento manual" com "Estabelecimento", o paciente pelo CPF, "Profissional" (profissionais com vínculo no estabelecimento, Divergência B5), "CBO" quando há mais de um, e o `ProcedureForm` com data editável; manda `professional_user_id` e `cbo_code` (sem CBO no vínculo, segue sem `cbo_code`: a crítica marca);
  - `ManualEntrySection({ establishments, today, searchDelayMs?, onSaved(message) })` — botão "Lançamento manual" que abre e fecha o painel;
  - `BillingProcedures` passa `headerActions` só para o `billing_clerk`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/billing/ManualEntryPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listBillingProfessionals: vi.fn(), lookupBillingPatient: vi.fn(), searchBillingSigtap: vi.fn(),
    recordProcedure: vi.fn() };
});

import * as api from "../../lib/api";
import { ManualEntryPanel, ManualEntrySection } from "./ManualEntryPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { LAB_EST, PATIENT, PROFESSIONALS, TODAY22, UPA_EST, URGENCY, billingUser, procedure } from "../../test/billingFixtures";
import { VALID_CPF } from "../../test/examFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.listBillingProfessionals, api.lookupBillingPatient, api.searchBillingSigtap, api.recordProcedure ];
afterEach(cleanup);
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ]));
  mocked(api.listBillingProfessionals).mockResolvedValue(PROFESSIONALS);
  mocked(api.lookupBillingPatient).mockResolvedValue(PATIENT);
  mocked(api.searchBillingSigtap).mockResolvedValue([ URGENCY ]);
  mocked(api.recordProcedure).mockResolvedValue(procedure({ id: "pp7", source: "manual", source_ref: null }));
});

function open(onSaved = vi.fn()) {
  renderWithProviders(<ManualEntryPanel establishments={[ UPA_EST, LAB_EST ]} today={TODAY22} searchDelayMs={0} onSaved={onSaved} onClose={() => undefined} />);
  return { panel: screen.getByRole("region", { name: "Lançamento manual" }), onSaved };
}

async function choose(panel: HTMLElement, professional: string) {
  fireEvent.change(within(panel).getByLabelText("Estabelecimento"), { target: { value: "health_unit:u7" } });
  await within(panel).findByRole("option", { name: "Dra. Helena Prado" });
  fireEvent.change(within(panel).getByLabelText("Profissional"), { target: { value: professional } });
}

async function patientAndProcedure(panel: HTMLElement) {
  fireEvent.change(within(panel).getByLabelText("CPF do paciente"), { target: { value: VALID_CPF } });
  fireEvent.click(within(panel).getByRole("button", { name: "Buscar paciente" }));
  await within(panel).findByText("Paciente: Joana Lima · CPF ***.982.247-**");
  fireEvent.change(within(panel).getByLabelText("Procedimento (SIGTAP)"), { target: { value: "urgencia" } });
  fireEvent.click(await within(panel).findByRole("button", { name: /^0301060061 — / }));
}

const send = (panel: HTMLElement) => fireEvent.click(within(panel).getByRole("button", { name: "Registrar procedimento" }));

describe("ManualEntryPanel", () => {
  it("estabelecimento → profissionais dele; CBO único vem escolhido; registra com profissional e CBO, sem atendimento", async () => {
    const { panel, onSaved } = open();
    await choose(panel, "us1");
    expect(api.listBillingProfessionals).toHaveBeenCalledWith({ type: "health_unit", id: "u7" });
    expect(within(panel).getByText("CBO 225125")).not.toBeNull();
    expect(within(panel).queryByLabelText("CBO")).toBeNull();
    await patientAndProcedure(panel);
    fireEvent.change(within(panel).getByLabelText("Data do procedimento"), { target: { value: "2026-10-02" } });
    send(panel);
    await waitFor(() => expect(api.recordProcedure).toHaveBeenCalledWith({
      establishment: { type: "health_unit", id: "u7" }, performed_on: "2026-10-02", sigtap_code: "0301060061", quantity: 1,
      character: "elective", patient_id: "pa1", professional_user_id: "us1", cbo_code: "225125"
    }));
    expect(onSaved).toHaveBeenCalledWith(procedure({ id: "pp7", source: "manual", source_ref: null }));
  });

  it("CBO com mais de uma opção exige escolha; sem CBO no vínculo segue sem cbo_code", async () => {
    const { panel } = open();
    await choose(panel, "us3");
    await patientAndProcedure(panel);
    send(panel);
    expect(within(panel).getByRole("alert").textContent).toBe("escolha o CBO do profissional");
    fireEvent.change(within(panel).getByLabelText("CBO"), { target: { value: "223565" } });
    send(panel);
    await waitFor(() => expect(api.recordProcedure).toHaveBeenLastCalledWith(expect.objectContaining({ professional_user_id: "us3", cbo_code: "223565" })));
    fireEvent.change(within(panel).getByLabelText("Profissional"), { target: { value: "us4" } });
    expect(within(panel).getByText(/profissional sem CBO no vínculo/)).not.toBeNull();
    send(panel);
    await waitFor(() => expect(api.recordProcedure).toHaveBeenCalledTimes(2));
    expect(mocked(api.recordProcedure).mock.calls[1][0]).not.toHaveProperty("cbo_code");
    expect(mocked(api.recordProcedure).mock.calls[1][0].professional_user_id).toBe("us4");
  });

  it("sem estabelecimento ou paciente não envia; o que falta vem primeiro", async () => {
    const { panel } = open();
    fireEvent.change(within(panel).getByLabelText("Procedimento (SIGTAP)"), { target: { value: "urgencia" } });
    fireEvent.click(await within(panel).findByRole("button", { name: /^0301060061 — / }));
    send(panel);
    expect(within(panel).getByRole("alert").textContent).toBe("escolha o estabelecimento");
    await choose(panel, "us1");
    send(panel);
    expect(within(panel).getByRole("alert").textContent).toBe("busque o paciente pelo CPF");
    expect(api.recordProcedure).not.toHaveBeenCalled();
  });

  it("422 invalid_professional (vínculo encerrado no meio) vira frase", async () => {
    mocked(api.recordProcedure).mockRejectedValueOnce(new api.ApiError(422, { error: "invalid_professional" }, "x"));
    const { panel } = open();
    await choose(panel, "us1");
    await patientAndProcedure(panel);
    send(panel);
    expect((await within(panel).findByRole("alert")).textContent).toBe("este profissional não tem vínculo com o estabelecimento — escolha outro");
  });
});

describe("ManualEntrySection", () => {
  it("abre pelo botão, fecha ao salvar e devolve a frase da crítica", async () => {
    const onSaved = vi.fn();
    mocked(api.recordProcedure).mockResolvedValueOnce(procedure({ billable: false, blocking_reasons: [ "missing_professional_cns" ] }));
    renderWithProviders(<ManualEntrySection establishments={[ UPA_EST ]} today={TODAY22} searchDelayMs={0} onSaved={onSaved} />);
    fireEvent.click(screen.getByRole("button", { name: "Lançamento manual" }));
    const panel = screen.getByRole("region", { name: "Lançamento manual" });
    await choose(panel, "us1");
    await patientAndProcedure(panel);
    send(panel);
    await waitFor(() => expect(onSaved).toHaveBeenCalledWith(
      "Procedimento registrado — não faturável — profissional sem CNS no cadastro. O registro vale; o faturamento confere de novo no fechamento."
    ));
    expect(screen.queryByRole("region", { name: "Lançamento manual" })).toBeNull();
  });
});
```

No fim de `src/modules/BillingProcedures.test.tsx`, dentro do `describe("BillingProcedures")`:

```tsx
  it("só o faturista vê o lançamento manual", async () => {
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    expect(await screen.findByRole("button", { name: "Lançamento manual" })).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "municipal_admin" ]));
    renderWithProviders(<BillingProcedures today={TODAY22} />);
    await screen.findByText("0202010473 — DOSAGEM DE GLICOSE");
    expect(screen.queryByRole("button", { name: "Lançamento manual" })).toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/billing/ManualEntryPanel.test.tsx src/modules/BillingProcedures.test.tsx`
Expected: FAIL — `./ManualEntryPanel` não existe e não há botão "Lançamento manual".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/ManualEntryPanel.tsx
// Lançamento manual pelo faturista (spec §5.3; Divergência B5): produção feita
// fora do sistema, com o estabelecimento, o paciente (CPF no corpo), o
// profissional com vínculo e o CBO escolhidos; mesma crítica SIGTAP.
// Profissional sem CBO no vínculo segue: a crítica marca missing_professional_cbo.
import { useState } from "react";
import { useQuery } from "@tanstack/react-query";
import { listBillingProfessionals, recordProcedure, type BillingEstablishment, type BillingPatient, type PerformedProcedure } from "../../lib/api";
import {
  PROFESSIONALS_KEY, billingError, establishmentKey, establishmentLabel, manualProblem, parseEstablishmentKey, procedureBody, savedText
} from "../../lib/billing";
import { buttonStyle, inputStyle } from "../../components/formStyles";
import { alert, fieldLabel, muted, panel } from "../exams/styles";
import { BillingPatientLookup } from "./BillingPatientLookup";
import { ProcedureForm } from "./ProcedureForm";

interface PanelProps {
  establishments: BillingEstablishment[];
  today: string;
  searchDelayMs?: number;
  onSaved(p: PerformedProcedure): void;
  onClose(): void;
}

export function ManualEntryPanel({ establishments, today, searchDelayMs, onSaved, onClose }: PanelProps) {
  const [ estKey, setEstKey ] = useState("");
  const [ patient, setPatient ] = useState<BillingPatient | null>(null);
  const [ professionalId, setProfessionalId ] = useState("");
  const [ cbo, setCbo ] = useState("");
  const est = parseEstablishmentKey(estKey);
  const professionals = useQuery({
    queryKey: [ PROFESSIONALS_KEY, estKey ], queryFn: () => listBillingProfessionals(est!), enabled: !!est, retry: false
  });
  const professional = professionals.data?.find((p) => p.user_id === professionalId) ?? null;
  const cboOptions = professional?.cbo_codes ?? [];
  const chosenCbo = cboOptions.length === 1 ? cboOptions[0] : cbo;
  const problem = manualProblem({ establishment: estKey, patientId: patient?.id ?? null, professionalId, cboOptions, cbo: chosenCbo });

  const before = (
    <>
      <label style={fieldLabel}>
        Estabelecimento
        <select value={estKey} style={inputStyle} onChange={(e) => { setEstKey(e.target.value); setProfessionalId(""); setCbo(""); }}>
          <option value="">escolha o estabelecimento</option>
          {establishments.map((e) => <option key={establishmentKey(e)} value={establishmentKey(e)}>{establishmentLabel(e)}</option>)}
        </select>
      </label>
      <BillingPatientLookup patient={patient} onFound={setPatient} />
      {est && (
        <label style={fieldLabel}>
          Profissional
          <select value={professionalId} style={inputStyle} onChange={(e) => { setProfessionalId(e.target.value); setCbo(""); }}>
            <option value="">escolha o profissional</option>
            {(professionals.data ?? []).map((p) => <option key={p.user_id} value={p.user_id}>{p.name}</option>)}
          </select>
        </label>
      )}
      {professionals.isError && <p role="alert" style={alert}>{billingError(professionals.error)}</p>}
      {professional && cboOptions.length > 1 && (
        <label style={fieldLabel}>
          CBO
          <select value={cbo} style={inputStyle} onChange={(e) => setCbo(e.target.value)}>
            <option value="">escolha o CBO</option>
            {cboOptions.map((c) => <option key={c} value={c}>{c}</option>)}
          </select>
        </label>
      )}
      {professional && cboOptions.length === 1 && <p style={muted}>{`CBO ${cboOptions[0]}`}</p>}
      {professional && cboOptions.length === 0 && (
        <p style={muted}>profissional sem CBO no vínculo — o procedimento fica não faturável até a administração corrigir em Equipe → Profissionais</p>
      )}
    </>
  );

  return (
    <section aria-label="Lançamento manual" style={panel}>
      <strong>Lançamento manual</strong>
      <p style={muted}>Produção feita fora do sistema. Passa pela mesma conferência com a SIGTAP e não impede o registro.</p>
      <ProcedureForm today={today} dateEditable searchDelayMs={searchDelayMs} before={before} beforeProblem={problem}
        submit={(draft) => recordProcedure(procedureBody(draft, {
          establishment: est!, patientId: patient!.id, professionalUserId: professionalId, cboCode: chosenCbo || null
        }))}
        onSaved={onSaved} onCancel={onClose} />
    </section>
  );
}

export function ManualEntrySection({ establishments, today, searchDelayMs, onSaved }: {
  establishments: BillingEstablishment[]; today: string; searchDelayMs?: number; onSaved(message: string): void;
}) {
  const [ open, setOpen ] = useState(false);
  if (!open) {
    return <div><button type="button" style={buttonStyle} onClick={() => setOpen(true)}>Lançamento manual</button></div>;
  }
  return (
    <ManualEntryPanel establishments={establishments} today={today} searchDelayMs={searchDelayMs}
      onSaved={(p) => { setOpen(false); onSaved(savedText(p)); }} onClose={() => setOpen(false)} />
  );
}
```

`src/modules/BillingProcedures.tsx` — import, depois de `import { ProcedureTable } from "./billing/ProcedureTable";`:

```ts
import { ManualEntrySection } from "./billing/ManualEntryPanel";
```

e troque

```tsx
      {(roles) => <ProceduresBody clerk={roles.clerk} today={today} />}
```

por

```tsx
      {(roles) => (
        <ProceduresBody clerk={roles.clerk} today={today} headerActions={roles.clerk ? ({ establishments, onSaved }) => (
          <ManualEntrySection establishments={establishments} today={today} onSaved={onSaved} />
        ) : undefined} />
      )}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/billing src/modules/BillingProcedures.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/billing/ManualEntryPanel.tsx src/modules/billing/ManualEntryPanel.test.tsx src/modules/BillingProcedures.tsx src/modules/BillingProcedures.test.tsx
$G commit -m "feat: let the billing clerk record procedures done outside the system

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Competências por estabelecimento, resumo e pendências agrupadas

**Files:**
- Create: `src/modules/BillingPeriods.tsx`, `src/modules/billing/PendingPanel.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/BillingPeriods.test.tsx`

**Interfaces:**
- Consumes: `getPeriodPending`, `listBillingPeriods`, `BillingPeriod`, `PendingItem` (Task 1); `PENDING_KEY`, `PERIODS_KEY`, `billingError`, `closingCompetence`, `establishmentLabel`, `fixInfo`, `groupPending`, `notBillableText`, `periodStatusText`, `reasonLabel` (Task 2); `BillingGate` (Task 9); `competenceLabel`, `competenceOptions`; `fmtDay`.
- Produces:
  - `BillingPeriods({ today? })` — tela "Competências e arquivo BPA": seletor "Competência" (abre na anterior), tabela por estabelecimento (Estabelecimento, Situação, BPA-I, BPA-C, Não faturáveis, Pendências) e o botão "Pendências" por linha. Dentro, `PeriodsBody({ today })` (a Task 12 troca por uma versão com fechamento, reabertura e versões);
  - `PendingPanel({ period, onClose })` — região "Pendências de <MM/AAAA> · <estabelecimento>", grupos (regiões com o título da correção) em `FIX_ORDER`, "Conferir de novo" e "Fechar pendências";
  - `App.tsx`: `case "billing-periods": return <BillingPeriods />;`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/BillingPeriods.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listBillingPeriods: vi.fn(), getPeriodPending: vi.fn() };
});

import * as api from "../lib/api";
import { BillingPeriods } from "./BillingPeriods";
import { renderWithProviders } from "../test/campaignFixtures";
import { LAB_EST, TODAY22, billingUser, pending, period } from "../test/billingFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const CLOSED_LAB = period({
  id: "bp2", establishment: LAB_EST, status: "closed", version: 1, closed_at: "2026-10-05T09:00:00-03:00", closed_by: "Rita Faturista",
  summary: { bpa_i_lines: 0, bpa_c_lines: 40, not_billable: {}, pending: 0 }
});
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.listBillingPeriods, api.getPeriodPending ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ]));
  mocked(api.listBillingPeriods).mockResolvedValue([ period(), CLOSED_LAB ]);
  mocked(api.getPeriodPending).mockResolvedValue([
    pending(),
    pending({ procedure_id: "pp5", fix: "patient", reasons: [ "missing_patient_id" ], patient: { id: "pa2", display_name: "Caio Souza" } }),
    pending({ procedure_id: "pp6", fix: "procedure", reasons: [ "habilitation_missing" ] })
  ]);
});

describe("BillingPeriods", () => {
  it("abre na competência anterior, com o resumo por estabelecimento", async () => {
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    expect(await screen.findByText("UPA Boa Vista · CNES 2384299")).not.toBeNull();
    expect(api.listBillingPeriods).toHaveBeenCalledWith("202609");
    expect(screen.getByText("Laboratório Municipal · CNES 7654321")).not.toBeNull();
    expect(screen.getByText("aberta")).not.toBeNull();
    expect(screen.getByText("fechada · versão 1")).not.toBeNull();
    expect(screen.getByText("3 — CBO do profissional não permitido para o procedimento: 2; profissional sem CNS no cadastro: 1")).not.toBeNull();
    expect(screen.getByText("nenhum")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Competência"), { target: { value: "202608" } });
    await waitFor(() => expect(api.listBillingPeriods).toHaveBeenLastCalledWith("202608"));
  });

  it("pendências: só no clique, agrupadas por correção, motivo novo cru; Conferir de novo relê", async () => {
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    await screen.findByText("UPA Boa Vista · CNES 2384299");
    expect(api.getPeriodPending).not.toHaveBeenCalled();
    fireEvent.click(screen.getAllByRole("button", { name: "Pendências" })[0]);
    const box = await screen.findByRole("region", { name: "Pendências de 09/2026 · UPA Boa Vista" });
    await within(box).findByText("Cadastro do profissional · 1");
    expect(api.getPeriodPending).toHaveBeenCalledWith("bp1");
    expect(within(box).getAllByText(/^(Cadastro|Regra)[^·]+ · \d+$/).map((e) => e.textContent)).toEqual([
      "Cadastro do profissional · 1", "Cadastro do paciente · 1", "Regra da SIGTAP · 1"
    ]);
    const professional = within(box).getByRole("region", { name: "Cadastro do profissional" });
    expect(within(professional).getByText("CNS ou CBO do profissional — a administração corrige em Equipe → Profissionais")).not.toBeNull();
    expect(within(professional).getByText("profissional sem CNS no cadastro")).not.toBeNull();
    expect(within(professional).getByText("Enf. Lúcia Prado")).not.toBeNull();
    expect(within(box).getByText("paciente sem CPF nem CNS")).not.toBeNull();
    expect(within(box).getByText("habilitation_missing")).not.toBeNull();
    fireEvent.click(within(box).getByRole("button", { name: "Conferir de novo" }));
    await waitFor(() => expect(api.getPeriodPending).toHaveBeenCalledTimes(2));
    fireEvent.click(within(box).getByRole("button", { name: "Fechar pendências" }));
    expect(screen.queryByRole("region", { name: "Pendências de 09/2026 · UPA Boa Vista" })).toBeNull();
    expect(api.listBillingPeriods).toHaveBeenCalledTimes(1);
  });

  it("sem pendência: estado vazio; o municipal_admin também lê", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "municipal_admin" ]));
    mocked(api.getPeriodPending).mockResolvedValue([]);
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click((await screen.findAllByRole("button", { name: "Pendências" }))[1]);
    expect(await screen.findByText("nenhuma pendência nesta competência")).not.toBeNull();
    expect(api.getPeriodPending).toHaveBeenCalledWith("bp2");
  });

  it("competência sem produção: estado vazio; recusa da lista vira frase", async () => {
    mocked(api.listBillingPeriods).mockResolvedValueOnce([]);
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    expect(await screen.findByText("nenhum estabelecimento com produção nesta competência")).not.toBeNull();
    cleanup();
    mocked(api.listBillingPeriods).mockRejectedValueOnce(new api.ApiError(403, { error: "feature_disabled", feature: "procedure_billing" }, "x"));
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    expect((await screen.findByRole("alert")).textContent).toBe("o faturamento de procedimentos está desligado nesta cidade");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/BillingPeriods.test.tsx`
Expected: FAIL — `./BillingPeriods` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/PendingPanel.tsx
// Pendências da competência (contrato §5; Divergência B6): críticas
// recalculadas com o cadastro de agora, agrupadas pelo que corrigir. A rota
// grava trilha (contém paciente): lê ao abrir e no "Conferir de novo", nunca
// sozinha. Corrigido o cadastro, a pendência sai sem relançar (spec §2).
import { useQuery } from "@tanstack/react-query";
import { getPeriodPending, type BillingPeriod, type PendingItem } from "../../lib/api";
import { PENDING_KEY, billingError, establishmentLabel, fixInfo, groupPending, reasonLabel } from "../../lib/billing";
import { competenceLabel } from "../../lib/competence";
import { fmtDay } from "../../lib/exams";
import { DataTable, type Column } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, muted, panel, row } from "../exams/styles";

const COLS: Column<PendingItem>[] = [
  { label: "Data", w: "0.8fr", render: (i) => fmtDay(i.performed_on ?? null) },
  { label: "Procedimento", w: "2fr", card: "title", render: (i) => (i.sigtap_code ? `${i.sigtap_code} — ${i.label ?? ""}` : i.procedure_id) },
  { label: "Paciente", w: "1.2fr", render: (i) => i.patient?.display_name ?? "—" },
  { label: "Profissional", w: "1.2fr", render: (i) => i.professional?.name ?? "—" },
  { label: "Motivo", w: "2fr", render: (i) => i.reasons.map(reasonLabel).join("; ") }
];

export function PendingPanel({ period, onClose }: { period: BillingPeriod; onClose(): void }) {
  const query = useQuery({
    queryKey: [ PENDING_KEY, period.id ], queryFn: () => getPeriodPending(period.id), gcTime: 0, staleTime: 0, retry: false
  });
  const groups = groupPending(query.data ?? []);
  return (
    <section aria-label={`Pendências de ${competenceLabel(period.competence)} · ${period.establishment.name}`} style={panel}>
      <div style={row}>
        <strong style={{ flex: 1 }}>{`Pendências de ${competenceLabel(period.competence)} · ${establishmentLabel(period.establishment)}`}</strong>
        <button type="button" style={secondaryButtonStyle} disabled={query.isFetching} onClick={() => void query.refetch()}>Conferir de novo</button>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar pendências</button>
      </div>
      <p style={muted}>
        As críticas são recalculadas com o cadastro de agora. Corrigido o cadastro, a pendência sai daqui — nada precisa ser relançado.
        Pendência não entra no arquivo.
      </p>
      {query.isPending && <p className="mono" style={muted}>conferindo…</p>}
      {query.isError && <p role="alert" style={alert}>{billingError(query.error)}</p>}
      {query.isSuccess && groups.length === 0 && <EmptyState title="nenhuma pendência nesta competência" />}
      {groups.map((g) => {
        const info = fixInfo(g.fix);
        return (
          <section key={g.fix} aria-label={info.title} style={box}>
            <strong>{`${info.title} · ${g.items.length}`}</strong>
            {info.hint && <p style={muted}>{info.hint}</p>}
            <DataTable<PendingItem> cols={COLS} rows={g.items} rowKey={(i) => i.procedure_id} />
          </section>
        );
      })}
    </section>
  );
}
```

```tsx
// src/modules/BillingPeriods.tsx
// Competências por estabelecimento faturante (contrato §5; spec §5.4–§5.6):
// unidades não-APS e laboratório municipal com produção na competência, com o
// resumo da última conferência (BPA-I, BPA-C, não faturáveis por motivo,
// pendências). As pendências, recalculadas na hora, só no clique. Abre na
// competência anterior (a que se fecha no começo do mês). Nada relê sozinho.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { listBillingPeriods, type BillingPeriod } from "../lib/api";
import { PERIODS_KEY, billingError, closingCompetence, establishmentLabel, notBillableText, periodStatusText } from "../lib/billing";
import { competenceLabel, competenceOptions } from "../lib/competence";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { DataTable, type Column } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { Tag } from "../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { alert, muted, row, stack } from "./exams/styles";
import { BillingGate } from "./billing/BillingGate";
import { PendingPanel } from "./billing/PendingPanel";

export function BillingPeriods({ today = todayInCity() }: { today?: string } = {}) {
  return (
    <BillingGate title="Competências e arquivo BPA" sub="faturamento · fechamento por estabelecimento (CNES), pendências e arquivo do BPA Magnético"
      allow={(r) => r.clerk || r.admin}>
      {() => <PeriodsBody today={today} />}
    </BillingGate>
  );
}

type Opened = { kind: "pending"; period: BillingPeriod };

function PeriodsBody({ today }: { today: string }) {
  const [ competence, setCompetence ] = useState(() => closingCompetence(today));
  const [ opened, setOpened ] = useState<Opened | null>(null);
  const list = useQuery({ queryKey: [ PERIODS_KEY, competence ], queryFn: () => listBillingPeriods(competence), gcTime: 0, retry: false });

  const cols: Column<BillingPeriod>[] = [
    { label: "Estabelecimento", w: "1.8fr", card: "title", render: (p) => establishmentLabel(p.establishment) },
    { label: "Situação", w: "1.3fr", render: (p) => <Tag tone={p.status === "closed" ? "ok" : "neutral"} mono={false}>{periodStatusText(p)}</Tag> },
    { label: "BPA-I", w: "0.6fr", align: "right", render: (p) => String(p.summary.bpa_i_lines) },
    { label: "BPA-C", w: "0.6fr", align: "right", render: (p) => String(p.summary.bpa_c_lines) },
    { label: "Não faturáveis", w: "2fr", render: (p) => notBillableText(p.summary) },
    { label: "Pendências", w: "0.8fr", align: "right", render: (p) => String(p.summary.pending) },
    { label: "", w: "auto", align: "right", card: "actions", render: (p) => (
      <div data-row-actions style={row}>
        <button type="button" style={secondaryButtonStyle} onClick={() => setOpened({ kind: "pending", period: p })}>Pendências</button>
      </div>
    ) }
  ];

  return (
    <div style={stack}>
      <Panel title="Estabelecimentos" sub="não-APS e laboratório municipal com produção na competência" right={
        <label style={selectLabel}>
          Competência
          <select value={competence} style={inputStyle} onChange={(e) => { setOpened(null); setCompetence(e.target.value); }}>
            {competenceOptions(today, competence).map((c) => <option key={c} value={c}>{competenceLabel(c)}</option>)}
          </select>
        </label>
      }>
        <div style={{ ...stack, padding: 16 }}>
          {list.isPending && <p className="mono" style={muted}>carregando…</p>}
          {list.isError && <p role="alert" style={alert}>{billingError(list.error)}</p>}
          {list.isSuccess && list.data.length === 0 && <EmptyState title="nenhum estabelecimento com produção nesta competência" />}
          {list.isSuccess && list.data.length > 0 && <DataTable<BillingPeriod> cols={cols} rows={list.data} rowKey={(p) => p.id} />}
        </div>
      </Panel>
      {opened?.kind === "pending" && <PendingPanel key={opened.period.id} period={opened.period} onClose={() => setOpened(null)} />}
    </div>
  );
}

const selectLabel: CSSProperties = { display: "flex", alignItems: "center", gap: 8, fontSize: 12, color: "var(--ink2)" };
```

`src/App.tsx` — import, depois de `import { BillingProcedures } from "./modules/BillingProcedures";`:

```ts
import { BillingPeriods } from "./modules/BillingPeriods";
```

e em `renderModule`, depois de `case "billing-procedures": …`:

```tsx
    case "billing-periods": return <BillingPeriods />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/BillingPeriods.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/BillingPeriods.tsx src/modules/billing/PendingPanel.tsx src/modules/BillingPeriods.test.tsx src/App.tsx
$G commit -m "feat: add billing periods by establishment with pending fixes grouped by correction

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Fechar e reabrir com step-up; versões do arquivo e download

**Files:**
- Create: `src/modules/billing/ExportsPanel.tsx`
- Modify: `src/modules/BillingPeriods.tsx` (reescrito por inteiro abaixo), `src/App.tsx`
- Test: `src/modules/BillingPeriods.close.test.tsx`

**Interfaces:**
- Consumes: `closeBillingPeriod`, `reopenBillingPeriod`, `listPeriodExports`, `fetchBillingExportFile`, `ApiError`, `errorCode`, `BillingExport`, `BillingPeriod` (Task 1); `EXPORTS_KEY`, `billingError`, `closeDescription`, `exportFileName`, `isStale`, `reasonProblem` (Task 2); `PendingPanel` (Task 11); `SensitiveAction`; `saveFile` (`src/modules/lab/saveFile.ts`); `fmtDateTime`.
- Produces:
  - `BillingPeriods({ onGoToSecurity?, today? })` — por linha: "Pendências", "Versões"; só o faturista: "Fechar e gerar arquivo" (aberta) ou "Reabrir" (fechada), cada um num `SensitiveAction` com step-up (regiões "Fechar <MM/AAAA> · <estabelecimento>" e "Reabrir …", botões "Confirmar fechamento" e "Confirmar reabertura", campo "Motivo da reabertura");
  - `ExportsPanel({ period, canDownload, onClose })` — região "Versões de <MM/AAAA> · <estabelecimento>": versão, gerado em, por, linhas, SHA-256 (16 primeiros, inteiro no `title`), "Baixar versão N" só com `canDownload`;
  - `App.tsx`: `case "billing-periods": return <BillingPeriods onGoToSecurity={() => setActive("security")} />;`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/BillingPeriods.close.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), listBillingPeriods: vi.fn(), getPeriodPending: vi.fn(),
    closeBillingPeriod: vi.fn(), reopenBillingPeriod: vi.fn(), listPeriodExports: vi.fn(), fetchBillingExportFile: vi.fn() };
});

import * as api from "../lib/api";
import { BillingPeriods } from "./BillingPeriods";
import { renderWithProviders } from "../test/campaignFixtures";
import { LAB_EST, TODAY22, billingExport, billingUser, period } from "../test/billingFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ALL = [ api.fetchCurrentSession, api.stepUpMfa, api.listBillingPeriods, api.getPeriodPending, api.closeBillingPeriod, api.reopenBillingPeriod,
  api.listPeriodExports, api.fetchBillingExportFile ];
const CLOSED_LAB = period({
  id: "bp2", establishment: LAB_EST, status: "closed", version: 1, closed_at: "2026-10-05T09:00:00-03:00", closed_by: "Rita Faturista",
  summary: { bpa_i_lines: 0, bpa_c_lines: 40, not_billable: {}, pending: 0 }
});
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of ALL) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ]));
  mocked(api.stepUpMfa).mockResolvedValue(undefined);
  mocked(api.listBillingPeriods).mockResolvedValue([ period(), CLOSED_LAB ]);
  mocked(api.closeBillingPeriod).mockResolvedValue(undefined);
  mocked(api.reopenBillingPeriod).mockResolvedValue(undefined);
  mocked(api.listPeriodExports).mockResolvedValue([
    billingExport({ id: "ex2", version: 2, generated_at: "2026-10-06T10:00:00-03:00" }), billingExport()
  ]);
  mocked(api.fetchBillingExportFile).mockResolvedValue({ blob: new Blob([ "01#BPA#202609" ]), filename: null });
});

describe("BillingPeriods — fechamento, reabertura e arquivo", () => {
  it("fechar com step-up: descreve, fecha, relê a lista e as versões", async () => {
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click(await screen.findByRole("button", { name: "Fechar e gerar arquivo" }));
    const dialog = screen.getByRole("region", { name: "Fechar 09/2026 · UPA Boa Vista" });
    expect(within(dialog).getByText(/entram 12 linhas de BPA-I e 3 de BPA-C; 1 pendência fica fora do arquivo até ser corrigida/)).not.toBeNull();
    expect(within(dialog).getByText(/reabrir gera uma nova versão/)).not.toBeNull();
    expect(within(dialog).queryByLabelText("Código do autenticador")).toBeNull();
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar fechamento" }));
    await waitFor(() => expect(api.closeBillingPeriod).toHaveBeenCalledWith("bp1"));
    expect((await screen.findByRole("status")).textContent).toBe("Competência 09/2026 fechada para UPA Boa Vista; o arquivo está em Versões.");
    await waitFor(() => expect(api.listBillingPeriods).toHaveBeenCalledTimes(2));
    expect(screen.queryByRole("region", { name: "Fechar 09/2026 · UPA Boa Vista" })).toBeNull();
  });

  it("janela fechada pede o código; 422 settings_missing aponta a configuração e nada muda", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ], { mfa_verified_at: null }));
    mocked(api.closeBillingPeriod).mockRejectedValueOnce(new api.ApiError(422, { error: "settings_missing" }, "x"));
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click(await screen.findByRole("button", { name: "Fechar e gerar arquivo" }));
    const dialog = screen.getByRole("region", { name: "Fechar 09/2026 · UPA Boa Vista" });
    fireEvent.change(within(dialog).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar fechamento" }));
    expect((await within(dialog).findByRole("alert")).textContent).toBe(
      "falta a configuração do faturamento (nome, sigla e CNPJ de origem, órgão de destino e gestão) — a administração municipal preenche em Faturamento → Configuração do faturamento"
    );
    expect(api.stepUpMfa).toHaveBeenCalledWith("123456");
    expect(api.listBillingPeriods).toHaveBeenCalledTimes(1);
    expect(screen.queryByRole("status")).toBeNull();
  });

  it("409 invalid_transition (outra pessoa fechou antes): frase e a lista relê", async () => {
    mocked(api.closeBillingPeriod).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click(await screen.findByRole("button", { name: "Fechar e gerar arquivo" }));
    const dialog = screen.getByRole("region", { name: "Fechar 09/2026 · UPA Boa Vista" });
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar fechamento" }));
    expect((await within(dialog).findByRole("alert")).textContent).toBe("a competência mudou de situação — a tela foi atualizada");
    await waitFor(() => expect(api.listBillingPeriods).toHaveBeenCalledTimes(2));
  });

  it("reabrir exige motivo de 10 a 500 caracteres; com motivo, reabre e diz a próxima versão", async () => {
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click(await screen.findByRole("button", { name: "Reabrir" }));
    const dialog = screen.getByRole("region", { name: "Reabrir 09/2026 · Laboratório Municipal" });
    expect(within(dialog).getByText(/A versão 1 do arquivo fica no histórico; o próximo fechamento gera a versão 2/)).not.toBeNull();
    fireEvent.change(within(dialog).getByLabelText("Motivo da reabertura"), { target: { value: "curto" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar reabertura" }));
    expect((await within(dialog).findByRole("alert")).textContent).toBe("informe o motivo, de 10 a 500 caracteres");
    expect(api.reopenBillingPeriod).not.toHaveBeenCalled();
    fireEvent.change(within(dialog).getByLabelText("Motivo da reabertura"), { target: { value: "CNS do profissional corrigido no cadastro" } });
    fireEvent.click(within(dialog).getByRole("button", { name: "Confirmar reabertura" }));
    await waitFor(() => expect(api.reopenBillingPeriod).toHaveBeenCalledWith("bp2", "CNS do profissional corrigido no cadastro"));
    expect((await screen.findByRole("status")).textContent).toBe("Competência reaberta. O próximo fechamento gera a versão 2.");
  });

  it("versões: o faturista baixa com o nome do api ou o de reserva; o admin vê sem baixar", async () => {
    const names: string[] = [];
    vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) { names.push(this.download); });
    URL.createObjectURL = vi.fn(() => "blob:bpa");
    URL.revokeObjectURL = vi.fn();
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click((await screen.findAllByRole("button", { name: "Versões" }))[1]);
    const box = await screen.findByRole("region", { name: "Versões de 09/2026 · Laboratório Municipal" });
    expect(api.listPeriodExports).toHaveBeenCalledWith("bp2");
    fireEvent.click(await within(box).findByRole("button", { name: "Baixar versão 2" }));
    await waitFor(() => expect(api.fetchBillingExportFile).toHaveBeenCalledWith("ex2"));
    expect((await within(box).findByRole("status")).textContent).toBe("Versão 2 baixada — o download fica registrado na trilha.");
    expect(names).toEqual([ "bpa-7654321-202609-v2.txt" ]);
    mocked(api.fetchBillingExportFile).mockResolvedValueOnce({ blob: new Blob([ "01#BPA#202609" ]), filename: "bpa-7654321-202609-v1.txt" });
    fireEvent.click(within(box).getByRole("button", { name: "Baixar versão 1" }));
    await waitFor(() => expect(names).toEqual([ "bpa-7654321-202609-v2.txt", "bpa-7654321-202609-v1.txt" ]));
    expect(within(box).getByTitle(billingExport().sha256)).not.toBeNull();

    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "municipal_admin" ]));
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click((await screen.findAllByRole("button", { name: "Versões" }))[1]);
    const adminBox = await screen.findByRole("region", { name: "Versões de 09/2026 · Laboratório Municipal" });
    expect(await within(adminBox).findByText("versão 2")).not.toBeNull();
    expect(within(adminBox).queryByRole("button", { name: /^Baixar/ })).toBeNull();
    expect(screen.queryByRole("button", { name: "Fechar e gerar arquivo" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Reabrir" })).toBeNull();
  });

  it("403 missing_role ao baixar vira frase; nada é salvo", async () => {
    const names: string[] = [];
    vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) { names.push(this.download); });
    mocked(api.fetchBillingExportFile).mockRejectedValueOnce(new api.ApiError(403, { error: "missing_role" }, "x"));
    renderWithProviders(<BillingPeriods today={TODAY22} />);
    fireEvent.click((await screen.findAllByRole("button", { name: "Versões" }))[1]);
    const box = await screen.findByRole("region", { name: "Versões de 09/2026 · Laboratório Municipal" });
    fireEvent.click(await within(box).findByRole("button", { name: "Baixar versão 2" }));
    expect((await within(box).findByRole("alert")).textContent).toBe("seu papel não permite esta ação no faturamento");
    expect(names).toEqual([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/BillingPeriods.close.test.tsx`
Expected: FAIL — não há "Fechar e gerar arquivo", "Reabrir" nem "Versões".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/billing/ExportsPanel.tsx
// Versões do arquivo da competência (contrato §5; Divergência B9). O arquivo
// gerado nunca muda; reabrir gera nova versão e a anterior fica aqui. Baixar é
// só do faturista (o admin vê a lista) e cada download fica na trilha
// (billing_export.downloaded). O conteúdo vai direto ao saveFile: nada em
// console, storage ou mensagem; o nome não leva dado pessoal.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { fetchBillingExportFile, listPeriodExports, type BillingExport, type BillingPeriod } from "../../lib/api";
import { EXPORTS_KEY, billingError, exportFileName } from "../../lib/billing";
import { competenceLabel } from "../../lib/competence";
import { fmtDateTime } from "../../lib/format";
import { DataTable, type Column } from "../../components/DataTable";
import { ActionButton } from "../../components/ActionButton";
import { secondaryButtonStyle } from "../../components/formStyles";
import { saveFile } from "../lab/saveFile";
import { alert, muted, panel, row } from "../exams/styles";

export function ExportsPanel({ period, canDownload, onClose }: { period: BillingPeriod; canDownload: boolean; onClose(): void }) {
  const list = useQuery({ queryKey: [ EXPORTS_KEY, period.id ], queryFn: () => listPeriodExports(period.id), gcTime: 0, retry: false });
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const name = `${competenceLabel(period.competence)} · ${period.establishment.name}`;

  async function download(exp: BillingExport) {
    setNotice(null); setError(null);
    try {
      const file = await fetchBillingExportFile(exp.id);
      saveFile(file, file.filename ?? exportFileName(period, exp));
      setNotice(`Versão ${exp.version} baixada — o download fica registrado na trilha.`);
    } catch (err) {
      setError(billingError(err));
    }
  }

  const downloadCol: Column<BillingExport>[] = canDownload ? [ {
    label: "", w: "auto", align: "right", card: "actions",
    render: (e) => <ActionButton variant="secondary" onClick={() => download(e)}>{`Baixar versão ${e.version}`}</ActionButton>
  } ] : [];
  const cols: Column<BillingExport>[] = [
    { label: "Versão", w: "0.6fr", card: "title", render: (e) => `versão ${e.version}` },
    { label: "Gerado em", w: "1.1fr", render: (e) => fmtDateTime(e.generated_at) },
    { label: "Por", w: "1.1fr", render: (e) => e.generated_by },
    { label: "Linhas", w: "1fr", render: (e) => `BPA-I ${e.lines_bpa_i} · BPA-C ${e.lines_bpa_c}` },
    { label: "Conferência (SHA-256)", w: "1.3fr", render: (e) => <span className="mono" title={e.sha256}>{e.sha256.slice(0, 16)}</span> },
    ...downloadCol
  ];

  return (
    <section aria-label={`Versões de ${name}`} style={panel}>
      <div style={row}>
        <strong style={{ flex: 1 }}>{`Versões do arquivo · ${name}`}</strong>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar versões</button>
      </div>
      {!canDownload && <p style={muted}>Baixar o arquivo é do faturista; aqui a administração vê as versões geradas.</p>}
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      {error && <p role="alert" style={alert}>{error}</p>}
      {list.isPending && <p className="mono" style={muted}>carregando…</p>}
      {list.isError && <p role="alert" style={alert}>{billingError(list.error)}</p>}
      {list.data && (
        <DataTable<BillingExport> cols={cols} rows={list.data} rowKey={(e) => e.id} empty="nenhum arquivo gerado nesta competência" />
      )}
    </section>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

Reescreva `src/modules/BillingPeriods.tsx` por inteiro:

```tsx
// src/modules/BillingPeriods.tsx
// Competências por estabelecimento faturante (contrato §5; spec §5.4–§5.6):
// unidades não-APS e laboratório municipal com produção na competência, com o
// resumo da última conferência (BPA-I, BPA-C, não faturáveis por motivo,
// pendências). As pendências, recalculadas na hora, só no clique. O faturista
// fecha (gera o arquivo, nova versão a cada fechamento) e reabre com motivo,
// os dois com step-up; o municipal_admin lê. Abre na competência anterior.
// Nada relê sozinho: só ao abrir, ao trocar a competência e depois de uma ação.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { ApiError, closeBillingPeriod, errorCode, listBillingPeriods, reopenBillingPeriod, type BillingPeriod } from "../lib/api";
import {
  EXPORTS_KEY, PERIODS_KEY, billingError, closeDescription, closingCompetence, establishmentLabel, isStale, notBillableText, periodStatusText,
  reasonProblem
} from "../lib/billing";
import { competenceLabel, competenceOptions } from "../lib/competence";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { DataTable, type Column } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { SensitiveAction } from "../components/SensitiveAction";
import { Tag } from "../components/Tag";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { alert, muted, row, stack } from "./exams/styles";
import { BillingGate } from "./billing/BillingGate";
import { ExportsPanel } from "./billing/ExportsPanel";
import { PendingPanel } from "./billing/PendingPanel";

export function BillingPeriods({ onGoToSecurity, today = todayInCity() }: { onGoToSecurity?(): void; today?: string } = {}) {
  return (
    <BillingGate title="Competências e arquivo BPA" sub="faturamento · fechamento por estabelecimento (CNES), pendências e arquivo do BPA Magnético"
      allow={(r) => r.clerk || r.admin}>
      {(roles) => <PeriodsBody clerk={roles.clerk} today={today} onGoToSecurity={onGoToSecurity} />}
    </BillingGate>
  );
}

type Opened = { kind: "pending" | "exports" | "close" | "reopen"; period: BillingPeriod };

function PeriodsBody({ clerk, today, onGoToSecurity }: { clerk: boolean; today: string; onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const [ competence, setCompetence ] = useState(() => closingCompetence(today));
  const [ opened, setOpened ] = useState<Opened | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const list = useQuery({ queryKey: [ PERIODS_KEY, competence ], queryFn: () => listBillingPeriods(competence), gcTime: 0, retry: false });
  const reload = () => void queryClient.invalidateQueries({ queryKey: [ PERIODS_KEY ] });
  const open = (kind: Opened["kind"], period: BillingPeriod) => { setNotice(null); setOpened({ kind, period }); };
  const nameOf = (p: BillingPeriod) => `${competenceLabel(p.competence)} · ${p.establishment.name}`;
  // Recusa que diz que a competência mudou por outra pessoa: relê a lista.
  const translate = (err: unknown) => {
    if (isStale(err)) reload();
    return errorCode(err) ? billingError(err) : null;
  };

  const cols: Column<BillingPeriod>[] = [
    { label: "Estabelecimento", w: "1.8fr", card: "title", render: (p) => establishmentLabel(p.establishment) },
    { label: "Situação", w: "1.3fr", render: (p) => <Tag tone={p.status === "closed" ? "ok" : "neutral"} mono={false}>{periodStatusText(p)}</Tag> },
    { label: "BPA-I", w: "0.6fr", align: "right", render: (p) => String(p.summary.bpa_i_lines) },
    { label: "BPA-C", w: "0.6fr", align: "right", render: (p) => String(p.summary.bpa_c_lines) },
    { label: "Não faturáveis", w: "2fr", render: (p) => notBillableText(p.summary) },
    { label: "Pendências", w: "0.8fr", align: "right", render: (p) => String(p.summary.pending) },
    { label: "", w: "auto", align: "right", card: "actions", render: (p) => (
      <div data-row-actions style={row}>
        <button type="button" style={secondaryButtonStyle} onClick={() => open("pending", p)}>Pendências</button>
        <button type="button" style={secondaryButtonStyle} onClick={() => open("exports", p)}>Versões</button>
        {clerk && p.status === "open" && <button type="button" style={buttonStyle} onClick={() => open("close", p)}>Fechar e gerar arquivo</button>}
        {clerk && p.status === "closed" && <button type="button" style={secondaryButtonStyle} onClick={() => open("reopen", p)}>Reabrir</button>}
      </div>
    ) }
  ];

  return (
    <div style={stack}>
      <Panel title="Estabelecimentos" sub="não-APS e laboratório municipal com produção na competência" right={
        <label style={selectLabel}>
          Competência
          <select value={competence} style={inputStyle} onChange={(e) => { setOpened(null); setNotice(null); setCompetence(e.target.value); }}>
            {competenceOptions(today, competence).map((c) => <option key={c} value={c}>{competenceLabel(c)}</option>)}
          </select>
        </label>
      }>
        <div style={{ ...stack, padding: 16 }}>
          {notice && <p role="status" style={statusStyle}>{notice}</p>}
          {list.isPending && <p className="mono" style={muted}>carregando…</p>}
          {list.isError && <p role="alert" style={alert}>{billingError(list.error)}</p>}
          {list.isSuccess && list.data.length === 0 && <EmptyState title="nenhum estabelecimento com produção nesta competência" />}
          {list.isSuccess && list.data.length > 0 && <DataTable<BillingPeriod> cols={cols} rows={list.data} rowKey={(p) => p.id} />}
        </div>
      </Panel>

      {opened?.kind === "pending" && <PendingPanel key={opened.period.id} period={opened.period} onClose={() => setOpened(null)} />}
      {opened?.kind === "exports" && (
        <ExportsPanel key={opened.period.id} period={opened.period} canDownload={clerk} onClose={() => setOpened(null)} />
      )}
      {clerk && opened?.kind === "close" && (
        <SensitiveAction
          key={`close:${opened.period.id}`}
          title={`Fechar ${nameOf(opened.period)}`}
          description={closeDescription(opened.period)}
          requiresStepUp
          confirmLabel="Confirmar fechamento"
          run={async () => { await closeBillingPeriod(opened.period.id); }}
          onDone={() => {
            setNotice(`Competência ${competenceLabel(opened.period.competence)} fechada para ${opened.period.establishment.name}; o arquivo está em Versões.`);
            setOpened(null);
            reload();
            void queryClient.invalidateQueries({ queryKey: [ EXPORTS_KEY ] });
          }}
          onCancel={() => setOpened(null)}
          onGoToSecurity={onGoToSecurity}
          translateError={translate}
        />
      )}
      {clerk && opened?.kind === "reopen" && (
        <SensitiveAction
          key={`reopen:${opened.period.id}`}
          title={`Reabrir ${nameOf(opened.period)}`}
          description={`A versão ${opened.period.version} do arquivo fica no histórico; o próximo fechamento gera a versão ${opened.period.version + 1}. O motivo fica na trilha.`}
          requiresStepUp
          confirmLabel="Confirmar reabertura"
          fields={[ { name: "reason", label: "Motivo da reabertura", required: true, multiline: true } ]}
          run={async (values) => {
            const reason = values.reason ?? "";
            // Recusar em vez de cortar: fora de 10..500, o api nem é chamado.
            if (reasonProblem(reason)) throw new ApiError(422, { error: "reason_required" }, "local");
            await reopenBillingPeriod(opened.period.id, reason.trim());
          }}
          onDone={() => {
            setNotice(`Competência reaberta. O próximo fechamento gera a versão ${opened.period.version + 1}.`);
            setOpened(null);
            reload();
          }}
          onCancel={() => setOpened(null)}
          onGoToSecurity={onGoToSecurity}
          translateError={translate}
        />
      )}
    </div>
  );
}

const selectLabel: CSSProperties = { display: "flex", alignItems: "center", gap: 8, fontSize: 12, color: "var(--ink2)" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

`src/App.tsx` — troque `    case "billing-periods": return <BillingPeriods />;` por

```tsx
    case "billing-periods": return <BillingPeriods onGoToSecurity={() => setActive("security")} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/BillingPeriods.test.tsx src/modules/BillingPeriods.close.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS (os testes da Task 11 seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/billing/ExportsPanel.tsx src/modules/BillingPeriods.tsx src/modules/BillingPeriods.close.test.tsx src/App.tsx
$G commit -m "feat: close and reopen billing periods with step-up and download the BPA file versions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Configuração do faturamento (municipal_admin, com step-up)

**Files:**
- Create: `src/modules/BillingSettings.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/BillingSettings.test.tsx`

**Interfaces:**
- Consumes: `getBillingSettings`, `saveBillingSettings`, `BillingSettings` (Task 1); `SETTINGS_KEY`, `ORIGIN_NAME_MAX`, `ORIGIN_ACRONYM_MAX`, `DESTINATION_MAX`, `billingError`, `settingsBody`, `settingsComplete`, `settingsDraft`, `settingsProblem`, `SettingsDraft` (Task 2); `BillingGate` (Task 9); `formatCnpj` (`clinicalDocuments.ts`); `SensitiveAction`, `KeyValue`, `Panel`.
- Produces: `BillingSettings({ onGoToSecurity? })` — tela "Configuração do faturamento" (só `municipal_admin`): os cinco campos do cabeçalho do arquivo (contrato §6 + Divergência B10), "Configurar faturamento"/"Editar configuração" → formulário ("Nome do órgão de origem", "Sigla do órgão de origem", "CNPJ de origem", "Órgão de destino", "Gestão") → "Revisar e salvar" → `SensitiveAction` com step-up e "Salvar configuração". `App.tsx`: `case "billing-settings": return <BillingSettings onGoToSecurity={() => setActive("security")} />;`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/BillingSettings.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getBillingSettings: vi.fn(), saveBillingSettings: vi.fn() };
});

import * as api from "../lib/api";
import { BillingSettings } from "./BillingSettings";
import { BILLING_NO_ROLE } from "../lib/billing";
import { renderWithProviders } from "../test/campaignFixtures";
import { SETTINGS, billingUser } from "../test/billingFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const EMPTY = { origin_name: null, origin_acronym: null, origin_cnpj: null, destination_body: null, management: null };
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.getBillingSettings, api.saveBillingSettings ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "municipal_admin" ]));
  mocked(api.getBillingSettings).mockResolvedValue(EMPTY);
  mocked(api.saveBillingSettings).mockResolvedValue(SETTINGS);
});

function fill(form: HTMLElement, over: Partial<Record<string, string>> = {}) {
  const values: Record<string, string> = {
    "Nome do órgão de origem": "SMS CURITIBA", "Sigla do órgão de origem": "SMS", "CNPJ de origem": "11.222.333/0001-81",
    "Órgão de destino": "SECRETARIA MUNICIPAL DE SAUDE", "Gestão": "M", ...over
  };
  for (const [ label, value ] of Object.entries(values)) fireEvent.change(within(form).getByLabelText(label), { target: { value } });
}

describe("BillingSettings", () => {
  it("sem configuração: diz que o fechamento não sai até preencher", async () => {
    renderWithProviders(<BillingSettings />);
    expect(await screen.findByText("Sem a configuração completa, o faturista não consegue fechar competência.")).not.toBeNull();
    expect(screen.getAllByText("não preenchido").length).toBe(5);
    expect(screen.getByRole("button", { name: "Configurar faturamento" })).not.toBeNull();
  });

  it("edita, revisa e salva com step-up; CNPJ vai só com dígitos e a tela relê", async () => {
    renderWithProviders(<BillingSettings />);
    fireEvent.click(await screen.findByRole("button", { name: "Configurar faturamento" }));
    const form = screen.getByRole("region", { name: "Editar configuração do faturamento" });
    fill(form);
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar configuração" }));
    await waitFor(() => expect(api.saveBillingSettings).toHaveBeenCalledWith({
      origin_name: "SMS CURITIBA", origin_acronym: "SMS", origin_cnpj: "11222333000181",
      destination_body: "SECRETARIA MUNICIPAL DE SAUDE", management: "M"
    }));
    expect((await screen.findByRole("status")).textContent).toBe("Configuração do faturamento salva.");
    await waitFor(() => expect(api.getBillingSettings).toHaveBeenCalledTimes(2));
  });

  it("nome maior que a largura do arquivo é recusado, nunca cortado; CNPJ inválido também", async () => {
    renderWithProviders(<BillingSettings />);
    fireEvent.click(await screen.findByRole("button", { name: "Configurar faturamento" }));
    const form = screen.getByRole("region", { name: "Editar configuração do faturamento" });
    fill(form, { "Nome do órgão de origem": "SECRETARIA MUNICIPAL DE SAUDE CWB" });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    expect(within(form).getByRole("alert").textContent).toBe("nome do órgão de origem com até 30 caracteres (largura do arquivo BPA) — abrevie");
    expect((within(form).getByLabelText("Nome do órgão de origem") as HTMLInputElement).value).toBe("SECRETARIA MUNICIPAL DE SAUDE CWB");
    fill(form, { "CNPJ de origem": "11.222.333/0001-80" });
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    expect(within(form).getByRole("alert").textContent).toBe("CNPJ de origem inválido");
    expect(screen.queryByRole("button", { name: "Salvar configuração" })).toBeNull();
  });

  it("configurada: mostra os valores; 422 invalid_cnpj do api vira frase", async () => {
    mocked(api.getBillingSettings).mockResolvedValue(SETTINGS);
    mocked(api.saveBillingSettings).mockRejectedValueOnce(new api.ApiError(422, { error: "invalid_cnpj" }, "x"));
    renderWithProviders(<BillingSettings />);
    expect(await screen.findByText("SMS CURITIBA")).not.toBeNull();
    expect(screen.getByText("11.222.333/0001-81")).not.toBeNull();
    expect(screen.getByText("municipal (M)")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Editar configuração" }));
    const form = screen.getByRole("region", { name: "Editar configuração do faturamento" });
    expect((within(form).getByLabelText("Sigla do órgão de origem") as HTMLInputElement).value).toBe("SMS");
    fireEvent.click(within(form).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar configuração" }));
    expect((await screen.findByRole("alert")).textContent).toBe("CNPJ de origem inválido");
  });

  it("faturista não vê a tela", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ]));
    renderWithProviders(<BillingSettings />);
    expect(await screen.findByText(BILLING_NO_ROLE)).not.toBeNull();
    expect(api.getBillingSettings).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/BillingSettings.test.tsx`
Expected: FAIL — `./BillingSettings` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/BillingSettings.tsx
// Configuração do faturamento da cidade (contrato §6; Divergência B10): o que
// vai no cabeçalho do arquivo do BPA Magnético — nome e sigla do órgão de
// origem, CNPJ de origem, órgão de destino e gestão (M/E). Só o
// municipal_admin; salvar passa pelo step-up. As larguras do layout são
// conferidas aqui (recusar, nunca cortar) e de novo no api.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, getBillingSettings, saveBillingSettings, type BillingSettings as Settings } from "../lib/api";
import {
  DESTINATION_MAX, ORIGIN_ACRONYM_MAX, ORIGIN_NAME_MAX, SETTINGS_KEY, billingError, settingsBody, settingsComplete, settingsDraft, settingsProblem,
  type SettingsDraft
} from "../lib/billing";
import { formatCnpj } from "../lib/clinicalDocuments";
import { Panel } from "../components/Panel";
import { KeyValue } from "../components/KeyValue";
import { SensitiveAction } from "../components/SensitiveAction";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { alert, box, fieldLabel, form, grid, muted, row, stack } from "./exams/styles";
import { BillingGate } from "./billing/BillingGate";

const MANAGEMENT_LABEL: Record<string, string> = { M: "municipal (M)", E: "estadual (E)" };
const NOT_SET = "não preenchido";

export function BillingSettings({ onGoToSecurity }: { onGoToSecurity?(): void } = {}) {
  return (
    <BillingGate title="Configuração do faturamento" sub="cabeçalho do arquivo do BPA Magnético · administração municipal" allow={(r) => r.admin}>
      {() => <SettingsBody onGoToSecurity={onGoToSecurity} />}
    </BillingGate>
  );
}

function SettingsBody({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const query = useQuery({ queryKey: [ SETTINGS_KEY ], queryFn: getBillingSettings, retry: false });
  const [ draft, setDraft ] = useState<SettingsDraft | null>(null);
  const [ confirming, setConfirming ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const settings: Settings | null = query.data ?? null;
  const set = (patch: Partial<SettingsDraft>) => { setError(null); setDraft((d) => (d ? { ...d, ...patch } : d)); };

  function review() {
    if (!draft) return;
    const problem = settingsProblem(draft);
    if (problem) { setError(problem); return; }
    setError(null);
    setConfirming(true);
  }

  // A dica fica fora do <label>: o nome acessível do campo é só o rótulo.
  const text = (label: string, key: keyof SettingsDraft, hint: string) => (
    <div style={{ display: "flex", flexDirection: "column", gap: 2 }}>
      <label style={fieldLabel}>
        {label}
        <input value={draft?.[key] ?? ""} style={inputStyle} onChange={(e) => set({ [key]: e.target.value } as Partial<SettingsDraft>)} />
      </label>
      <span style={hintStyle}>{hint}</span>
    </div>
  );

  return (
    <Panel title="Cabeçalho do arquivo BPA" sub="vale para todas as competências fechadas daqui em diante">
      <div style={{ ...stack, padding: 16 }}>
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{billingError(query.error)}</p>}
        {notice && <p role="status" style={statusStyle}>{notice}</p>}
        {query.isSuccess && (
          <div style={grid}>
            <KeyValue k="Órgão de origem" v={settings?.origin_name ?? NOT_SET} mono={false} />
            <KeyValue k="Sigla" v={settings?.origin_acronym ?? NOT_SET} />
            <KeyValue k="CNPJ de origem" v={settings?.origin_cnpj ? formatCnpj(settings.origin_cnpj) : NOT_SET} />
            <KeyValue k="Órgão de destino" v={settings?.destination_body ?? NOT_SET} mono={false} />
            <KeyValue k="Gestão" v={settings?.management ? MANAGEMENT_LABEL[settings.management] : NOT_SET} mono={false} />
          </div>
        )}
        {query.isSuccess && !settingsComplete(settings) && (
          <p style={muted}>Sem a configuração completa, o faturista não consegue fechar competência.</p>
        )}
        {query.isSuccess && !draft && (
          <div>
            <button type="button" style={buttonStyle} onClick={() => { setNotice(null); setConfirming(false); setDraft(settingsDraft(settings)); }}>
              {settingsComplete(settings) ? "Editar configuração" : "Configurar faturamento"}
            </button>
          </div>
        )}
        {draft && !confirming && (
          <section aria-label="Editar configuração do faturamento" style={box}>
            <div style={form}>
              {text("Nome do órgão de origem", "originName", `até ${ORIGIN_NAME_MAX} caracteres (largura do arquivo)`)}
              {text("Sigla do órgão de origem", "originAcronym", `até ${ORIGIN_ACRONYM_MAX} caracteres`)}
              {text("CNPJ de origem", "originCnpj", "secretaria ou fundo municipal de saúde")}
              {text("Órgão de destino", "destinationBody", `até ${DESTINATION_MAX} caracteres`)}
              <label style={fieldLabel}>
                Gestão
                <select value={draft.management} style={inputStyle} onChange={(e) => set({ management: e.target.value as SettingsDraft["management"] })}>
                  <option value="">escolha a gestão</option>
                  <option value="M">municipal (M)</option>
                  <option value="E">estadual (E)</option>
                </select>
              </label>
              {error && <p role="alert" style={alert}>{error}</p>}
              <div style={row}>
                <button type="button" style={buttonStyle} onClick={review}>Revisar e salvar</button>
                <button type="button" style={secondaryButtonStyle} onClick={() => { setDraft(null); setError(null); }}>Cancelar</button>
              </div>
            </div>
          </section>
        )}
        {draft && confirming && (
          <SensitiveAction
            title="Salvar a configuração do faturamento"
            description={`Origem: ${draft.originName.trim()} (${draft.originAcronym.trim()}), CNPJ ${formatCnpj(settingsBody(draft).origin_cnpj)}; destino: ${draft.destinationBody.trim()}; gestão ${MANAGEMENT_LABEL[draft.management] ?? draft.management}. Vale para os próximos arquivos; os já gerados não mudam.`}
            requiresStepUp
            confirmLabel="Salvar configuração"
            run={async () => { await saveBillingSettings(settingsBody(draft)); }}
            onDone={() => {
              setDraft(null); setConfirming(false); setNotice("Configuração do faturamento salva.");
              void queryClient.invalidateQueries({ queryKey: [ SETTINGS_KEY ] });
            }}
            onCancel={() => setConfirming(false)}
            onGoToSecurity={onGoToSecurity}
            translateError={(err) => (errorCode(err) ? billingError(err) : null)}
          />
        )}
      </div>
    </Panel>
  );
}

const hintStyle: CSSProperties = { fontSize: 11.5, color: "var(--ink3)" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

`src/App.tsx` — import, depois de `import { BillingPeriods } from "./modules/BillingPeriods";`:

```ts
import { BillingSettings } from "./modules/BillingSettings";
```

e em `renderModule`, depois de `case "billing-periods": …`:

```tsx
    case "billing-settings": return <BillingSettings onGoToSecurity={() => setActive("security")} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/BillingSettings.test.tsx src/theme src/shell && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/BillingSettings.tsx src/modules/BillingSettings.test.tsx src/App.tsx
$G commit -m "feat: let the municipal admin configure the BPA file header with step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Dados do paciente para o BPA-I — regras, cliente e formulário comum (D-U1)

Contrato §12, D-U1: `patients` ganha `race_color` (código do layout, **sem** `99`), `ethnicity` (obrigatória quando indígena), `address_zip`, `address_street`, `address_number`, `address_complement`, `address_district`; o município de residência continua o da cidade. Captados na recepção (Task 15: `POST /attendance/verifications` e `POST /attendance/verifications/:id/names`, rotas fixadas no plano do api) e pelo faturista na pendência (Task 16: `PATCH /billing/patients/:id/demographics`), sempre os mesmos 7 campos opcionais. Esta task traz o que as duas usam.

**Files:**
- Create: `src/lib/demographics.ts`, `src/modules/attendance/DemographicsForm.tsx`
- Modify: `src/lib/api.ts` (fim da seção do faturamento; `VerifyExtra` na seção do atendimento), `src/lib/api.billing.test.ts`
- Test: `src/lib/demographics.test.ts`, `src/modules/attendance/DemographicsForm.test.tsx`

**Interfaces:**
- Consumes: `jsonFetch`, `ATTENDANCE_BASE`, `billingPath` (Task 1); `ApiError`, `errorCode`; `ActionButton`, `Tag`, `formStyles`, `src/modules/exams/styles.ts`.
- Produces:
  - em `src/lib/api.ts`: `PatientDemographicsInput` (`{ race_color?, ethnicity?, address_zip?, address_street?, address_number?, address_complement?, address_district? }`; só o enviado muda); `DemographicsSaved = { race_color_set: boolean; address_set: boolean }` (o api nunca devolve o endereço); `updatePatientDemographics(patientId, input): Promise<void>` (`PATCH /billing/patients/:id/demographics`); `saveVerificationDemographics(verificationId, input): Promise<DemographicsSaved>` (`POST /attendance/verifications/:id/names` só com os campos do paciente, sem nomes); `VerifyExtra` passa a estender `PatientDemographicsInput` (os 7 campos planos no corpo de `POST /attendance/verifications`);
  - em `src/lib/demographics.ts`: `RACE_COLORS` (01 Branca, 02 Preta, 03 Parda, 04 Amarela, 05 Indígena — sem "sem informação"), `INDIGENOUS = "05"`, `STREET_MAX = 30`, `NUMBER_MAX = 5`, `COMPLEMENT_MAX = 10`, `DISTRICT_MAX = 30` (larguras `prd_end_pcnte`, `prd_num_pcnte`, `prd_compl_pcnte`, `prd_bairro_pcnte` do layout 05.00), `DEMOGRAPHICS_INCENTIVE`, `DemographicsDraft`, `EMPTY_DEMOGRAPHICS`, `demographicsFilled(d)`, `demographicsProblem(d)`, `demographicsBody(d)` (só o preenchido; etnia só com `05`; número vazio = `SN`; complemento vazio não vai), `missingText(codes)`, `demographicsError(err)` (inclusive 422 `too_long` com `field`);
  - em `src/modules/attendance/DemographicsForm.tsx`: `DemographicsFields({ value, onChange, problem? })` (campos "Raça/cor", "Etnia (código)" só para indígena, "CEP", "Logradouro", "Número", "Complemento", "Bairro", com a nota de incentivo) e `DemographicsForm({ title, missing?, save(body), onSaved?(), onCancel? })` (região com o título, botão "Salvar dados do paciente", frase "Dados do paciente salvos.").

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/demographics.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  EMPTY_DEMOGRAPHICS, RACE_COLORS, demographicsBody, demographicsError, demographicsFilled, demographicsProblem, missingText
} from "./demographics";

const ADDRESS = { ...EMPTY_DEMOGRAPHICS, zip: "80010-000", street: "RUA XV DE NOVEMBRO", number: "100", district: "CENTRO" };

describe("dados do paciente para o BPA-I (contrato §12, D-U1)", () => {
  it("lista de raça/cor do layout, sem 'sem informação' (99)", () => {
    expect(RACE_COLORS.map((r) => r.code)).toEqual([ "01", "02", "03", "04", "05" ]);
    expect(RACE_COLORS.some((r) => r.code === "99")).toBe(false);
  });

  it("tudo opcional: vazio não é problema, mas também não é 'preenchido'", () => {
    expect(demographicsFilled(EMPTY_DEMOGRAPHICS)).toBe(false);
    expect(demographicsProblem(EMPTY_DEMOGRAPHICS)).toBeNull();
    expect(demographicsFilled({ ...EMPTY_DEMOGRAPHICS, raceColor: "03" })).toBe(true);
  });

  it("indígena exige a etnia (4 dígitos); os outros não mandam etnia", () => {
    expect(demographicsProblem({ ...EMPTY_DEMOGRAPHICS, raceColor: "05" })).toBe("para indígena, informe a etnia: código de 4 dígitos da tabela do DATASUS");
    expect(demographicsProblem({ ...EMPTY_DEMOGRAPHICS, raceColor: "05", ethnicity: "12" })).toBe("para indígena, informe a etnia: código de 4 dígitos da tabela do DATASUS");
    expect(demographicsBody({ ...EMPTY_DEMOGRAPHICS, raceColor: "05", ethnicity: " 0001 " })).toEqual({ race_color: "05", ethnicity: "0001" });
    // A etnia é recusada pelo api fora de 05: nem vai no corpo.
    expect(demographicsBody({ ...EMPTY_DEMOGRAPHICS, raceColor: "03", ethnicity: "0001" })).toEqual({ race_color: "03" });
  });

  it("endereço: começou, precisa de CEP com 8 dígitos, logradouro e bairro; larguras recusadas, nunca cortadas", () => {
    expect(demographicsProblem({ ...EMPTY_DEMOGRAPHICS, street: "RUA XV" })).toBe("CEP com 8 dígitos");
    expect(demographicsProblem({ ...ADDRESS, street: "" })).toBe("informe o logradouro");
    expect(demographicsProblem({ ...ADDRESS, district: "" })).toBe("informe o bairro");
    expect(demographicsProblem({ ...ADDRESS, street: "x".repeat(31) })).toBe("logradouro com até 30 caracteres (largura do arquivo BPA) — abrevie");
    expect(demographicsProblem({ ...ADDRESS, number: "123456" })).toBe("número com até 5 caracteres");
    expect(demographicsProblem({ ...ADDRESS, complement: "x".repeat(11) })).toBe("complemento com até 10 caracteres — abrevie");
    expect(demographicsProblem({ ...ADDRESS, district: "x".repeat(31) })).toBe("bairro com até 30 caracteres (largura do arquivo BPA) — abrevie");
    expect(demographicsProblem(ADDRESS)).toBeNull();
    expect(demographicsBody(ADDRESS)).toEqual({
      address_zip: "80010000", address_street: "RUA XV DE NOVEMBRO", address_number: "100", address_district: "CENTRO"
    });
    // Sem número, "SN" (padrão do layout); complemento só se houver.
    expect(demographicsBody({ ...ADDRESS, number: " ", complement: "APTO 12" })).toEqual({
      address_zip: "80010000", address_street: "RUA XV DE NOVEMBRO", address_number: "SN", address_complement: "APTO 12", address_district: "CENTRO"
    });
  });

  it("o que falta em frase; recusas do api", () => {
    expect(missingText([ "race_color", "address" ])).toBe("raça/cor; CEP e endereço");
    expect(missingText([ "phone" ])).toBe("phone");
    expect(demographicsError(new ApiError(422, { error: "invalid_zip" }, "x"))).toBe("CEP inválido — confira os 8 dígitos");
    expect(demographicsError(new ApiError(422, { error: "ethnicity_required" }, "x"))).toBe("para indígena, informe a etnia: código de 4 dígitos da tabela do DATASUS");
    expect(demographicsError(new ApiError(403, { error: "missing_role" }, "x"))).toBe("seu papel não permite alterar os dados do paciente");
    expect(demographicsError(new ApiError(422, { error: "too_long", field: "address_street" }, "x"))).toBe("logradouro maior que a largura do arquivo BPA — abrevie");
    expect(demographicsError(new ApiError(422, { error: "too_long", field: "outro" }, "x"))).toBe("texto maior que a largura do arquivo BPA — abrevie");
    expect(demographicsError(new ApiError(500, "", "x"))).toBe("não foi possível salvar (500) — tente de novo");
  });
});
```

```tsx
// src/modules/attendance/DemographicsForm.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { ApiError } from "../../lib/api";
import { DEMOGRAPHICS_INCENTIVE } from "../../lib/demographics";
import { DemographicsForm } from "./DemographicsForm";

afterEach(cleanup);

function open(save = vi.fn().mockResolvedValue(undefined), onSaved = vi.fn()) {
  render(<DemographicsForm title="Completar dados do paciente" missing={[ "race_color", "address" ]} save={save} onSaved={onSaved} />);
  return { form: screen.getByRole("region", { name: "Completar dados do paciente" }), save, onSaved };
}

describe("DemographicsForm", () => {
  it("incentiva com o que falta; etnia só para indígena", () => {
    const { form } = open();
    expect(within(form).getByRole("note").textContent).toBe(`Falta: raça/cor; CEP e endereço. ${DEMOGRAPHICS_INCENTIVE}`);
    expect(within(form).queryByLabelText("Etnia (código)")).toBeNull();
    fireEvent.change(within(form).getByLabelText("Raça/cor"), { target: { value: "05" } });
    expect(within(form).getByLabelText("Etnia (código)")).not.toBeNull();
    expect(within(form).queryByRole("option", { name: /sem informação/i })).toBeNull();
  });

  it("vazio não salva; salva só o preenchido e avisa", async () => {
    const { form, save, onSaved } = open();
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    expect(within(form).getByRole("alert").textContent).toBe("preencha a raça/cor ou o endereço");
    fireEvent.change(within(form).getByLabelText("Raça/cor"), { target: { value: "03" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    await waitFor(() => expect(save).toHaveBeenCalledWith({ race_color: "03" }));
    expect((await within(form).findByRole("status")).textContent).toBe("Dados do paciente salvos.");
    expect(onSaved).toHaveBeenCalled();
  });

  it("logradouro longo é recusado antes do envio; 422 do api vira frase e o que foi digitado fica", async () => {
    const save = vi.fn().mockRejectedValueOnce(new ApiError(422, { error: "invalid_zip" }, "x"));
    const { form } = open(save);
    fireEvent.change(within(form).getByLabelText("CEP"), { target: { value: "80010-000" } });
    fireEvent.change(within(form).getByLabelText("Logradouro"), { target: { value: "x".repeat(31) } });
    fireEvent.change(within(form).getByLabelText("Bairro"), { target: { value: "CENTRO" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    expect(within(form).getByRole("alert").textContent).toBe("logradouro com até 30 caracteres (largura do arquivo BPA) — abrevie");
    expect(save).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("Logradouro"), { target: { value: "RUA XV DE NOVEMBRO" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("CEP inválido — confira os 8 dígitos");
    expect((within(form).getByLabelText("Logradouro") as HTMLInputElement).value).toBe("RUA XV DE NOVEMBRO");
  });
});
```

No fim do `describe` de `src/lib/api.billing.test.ts` (acrescente `updatePatientDemographics` e `saveVerificationDemographics` ao import de `./api`):

```ts
  it("dados do paciente: PATCH com JSON, tudo no corpo, só o id na URL", async () => {
    const input = { race_color: "05", ethnicity: "0001", address_zip: "80010000", address_street: "RUA XV DE NOVEMBRO",
      address_number: "100", address_district: "CENTRO" };
    await updatePatientDemographics("pa3", input);
    expect(urlOf().pathname).toBe("/billing/patients/pa3/demographics");
    expect(urlOf().search).toBe("");
    expect(initOf().method).toBe("PATCH");
    expect(headersOf()["Content-Type"]).toBe("application/json");
    expect(bodyOf()).toEqual(input);
    fetchMock.mockImplementationOnce(() => ok({ race_color_set: true, address_set: false }));
    expect(await saveVerificationDemographics("v9", { race_color: "03" })).toEqual({ race_color_set: true, address_set: false });
    expect(urlOf(1).pathname).toBe("/attendance/verifications/v9/names");
    expect(initOf(1).method).toBe("POST");
    expect(bodyOf(1)).toEqual({ race_color: "03" });
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/demographics.test.ts src/modules/attendance/DemographicsForm.test.tsx src/lib/api.billing.test.ts`
Expected: FAIL — `./demographics`, `./DemographicsForm`, `updatePatientDemographics` e `saveVerificationDemographics` não existem.

- [ ] **Step 3: Write minimal implementation**

`src/lib/api.ts` — na seção do atendimento, troque

```ts
export interface VerifyExtra { cadsus_confirmed?: boolean }
```

por

```ts
// Módulo 22 (contrato §12, D-U1): os 7 campos do paciente, opcionais e planos no corpo da validação.
export interface VerifyExtra extends PatientDemographicsInput { cadsus_confirmed?: boolean }
```

No fim de `src/lib/api.ts` (depois de `saveBillingSettings`):

```ts
// ─── Dados do paciente para o BPA-I (módulo 22; contrato §12, D-U1) ─────────
// Raça/cor (01–05, sem 99), etnia (4 dígitos, só com 05) e endereço; só o
// enviado muda. Tudo no corpo; na URL só o id. A recepção grava pela validação
// do par; o faturista, pela pendência. O api nunca devolve o endereço.
export interface PatientDemographicsInput {
  race_color?: string; ethnicity?: string;
  address_zip?: string; address_street?: string; address_number?: string; address_complement?: string; address_district?: string;
}
export interface DemographicsSaved { race_color_set: boolean; address_set: boolean }

export async function updatePatientDemographics(patientId: string, input: PatientDemographicsInput): Promise<void> {
  await jsonFetch<unknown>(billingPath("patients", patientId, "demographics"), { method: "PATCH", body: JSON.stringify(input) });
}

// No check-in: a rota de nomes do 19 aceita só os campos do paciente (sem nomes).
export async function saveVerificationDemographics(verificationId: string, input: PatientDemographicsInput): Promise<DemographicsSaved> {
  const body = await jsonFetch<Partial<DemographicsSaved>>(`${ATTENDANCE_BASE}/verifications/${encodeURIComponent(verificationId)}/names`, {
    method: "POST", body: JSON.stringify(input)
  });
  return { race_color_set: body.race_color_set === true, address_set: body.address_set === true };
}
```

```ts
// src/lib/demographics.ts
// Dados do paciente para o BPA-I (módulo 22; contrato §12, D-U1; layout do
// BPA Magnético 05.00): raça/cor sem "sem informação", etnia só para indígena
// e endereço. Tudo opcional e incentivado: a recepção pergunta, a pessoa pode
// não querer informar; o que faltar vira pendência do faturista no fechamento.
// Larguras do layout recusadas, nunca cortadas.
import { ApiError, errorCode, type PatientDemographicsInput } from "./api";

export const RACE_COLORS = [
  { code: "01", label: "Branca" }, { code: "02", label: "Preta" }, { code: "03", label: "Parda" },
  { code: "04", label: "Amarela" }, { code: "05", label: "Indígena" }
] as const;
export const INDIGENOUS = "05";
export const STREET_MAX = 30;
export const NUMBER_MAX = 5;
export const COMPLEMENT_MAX = 10;
export const DISTRICT_MAX = 30;
export const DEMOGRAPHICS_INCENTIVE =
  "Raça/cor e endereço completam o cadastro e evitam pendência no faturamento. Não é obrigatório: pergunte à pessoa.";

export interface DemographicsDraft {
  raceColor: string; ethnicity: string; zip: string; street: string; number: string; complement: string; district: string;
}
export const EMPTY_DEMOGRAPHICS: DemographicsDraft = { raceColor: "", ethnicity: "", zip: "", street: "", number: "", complement: "", district: "" };

const digits = (s: string) => s.replace(/\D/g, "");
const ETHNICITY_TEXT = "para indígena, informe a etnia: código de 4 dígitos da tabela do DATASUS";

function addressStarted(d: DemographicsDraft): boolean {
  return [ d.zip, d.street, d.number, d.complement, d.district ].some((v) => v.trim() !== "");
}

export function demographicsFilled(d: DemographicsDraft): boolean {
  return d.raceColor !== "" || addressStarted(d);
}

export function demographicsProblem(d: DemographicsDraft): string | null {
  if (d.raceColor && !RACE_COLORS.some((r) => r.code === d.raceColor)) return "raça/cor inválida";
  if (d.raceColor === INDIGENOUS && !/^\d{4}$/.test(d.ethnicity.trim())) return ETHNICITY_TEXT;
  if (addressStarted(d)) {
    if (digits(d.zip).length !== 8) return "CEP com 8 dígitos";
    if (!d.street.trim()) return "informe o logradouro";
    if (!d.district.trim()) return "informe o bairro";
  }
  if (d.street.trim().length > STREET_MAX) return `logradouro com até ${STREET_MAX} caracteres (largura do arquivo BPA) — abrevie`;
  if (d.number.trim().length > NUMBER_MAX) return `número com até ${NUMBER_MAX} caracteres`;
  if (d.complement.trim().length > COMPLEMENT_MAX) return `complemento com até ${COMPLEMENT_MAX} caracteres — abrevie`;
  if (d.district.trim().length > DISTRICT_MAX) return `bairro com até ${DISTRICT_MAX} caracteres (largura do arquivo BPA) — abrevie`;
  return null;
}

// Só o preenchido vai no corpo; o que não foi perguntado não apaga o que o cadastro já tem.
export function demographicsBody(d: DemographicsDraft): PatientDemographicsInput {
  const body: PatientDemographicsInput = {};
  if (d.raceColor) {
    body.race_color = d.raceColor;
    // O api recusa etnia fora de 05.
    if (d.raceColor === INDIGENOUS) body.ethnicity = d.ethnicity.trim();
  }
  if (addressStarted(d)) {
    body.address_zip = digits(d.zip);
    body.address_street = d.street.trim();
    // Sem número, "SN" (padrão do campo prd_num_pcnte do layout).
    body.address_number = d.number.trim() || "SN";
    if (d.complement.trim()) body.address_complement = d.complement.trim();
    body.address_district = d.district.trim();
  }
  return body;
}

const MISSING_TEXT: Record<string, string> = { race_color: "raça/cor", address: "CEP e endereço" };
export function missingText(codes: string[]): string {
  return codes.map((c) => MISSING_TEXT[c] ?? c).join("; ");
}

const ERRORS: Record<string, string> = {
  invalid_race_color: "raça/cor inválida",
  ethnicity_required: ETHNICITY_TEXT,
  invalid_ethnicity: ETHNICITY_TEXT,
  invalid_zip: "CEP inválido — confira os 8 dígitos",
  address_incomplete: "endereço incompleto: CEP, logradouro e bairro",
  too_long: "texto maior que a largura do arquivo BPA — abrevie",
  missing_role: "seu papel não permite alterar os dados do paciente",
  feature_disabled: "o faturamento de procedimentos está desligado nesta cidade",
  not_found: "paciente ou atendimento não encontrado — atualize a tela"
};

const FIELD_TEXT: Record<string, string> = {
  address_street: "logradouro", address_number: "número", address_complement: "complemento", address_district: "bairro"
};

export function demographicsError(err: unknown): string {
  const code = errorCode(err);
  if (code === "too_long" && err instanceof ApiError) {
    const field = (err.body as { field?: unknown } | null)?.field;
    if (typeof field === "string" && FIELD_TEXT[field]) return `${FIELD_TEXT[field]} maior que a largura do arquivo BPA — abrevie`;
  }
  if (code && ERRORS[code]) return ERRORS[code];
  if (err instanceof ApiError) return `não foi possível salvar (${err.status}) — tente de novo`;
  return "sem conexão com o servidor — tente de novo";
}
```

```tsx
// src/modules/attendance/DemographicsForm.tsx
// Raça/cor, etnia e endereço do paciente para o BPA-I (módulo 22; contrato
// §12, D-U1). `DemographicsFields` é só os campos (a validação presencial os
// manda junto); `DemographicsForm` salva sozinho (check-in e pendência do
// faturista). Opcionais, com incentivo visual; nada de "sem informação".
import { useState, type CSSProperties } from "react";
import type { PatientDemographicsInput } from "../../lib/api";
import {
  DEMOGRAPHICS_INCENTIVE, EMPTY_DEMOGRAPHICS, INDIGENOUS, RACE_COLORS, demographicsBody, demographicsError, demographicsFilled,
  demographicsProblem, missingText, type DemographicsDraft
} from "../../lib/demographics";
import { ActionButton } from "../../components/ActionButton";
import { Tag } from "../../components/Tag";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, muted, row } from "../exams/styles";

export function DemographicsFields({ value, onChange, problem = null, missing }: {
  value: DemographicsDraft; onChange(d: DemographicsDraft): void; problem?: string | null; missing?: string[];
}) {
  const set = (patch: Partial<DemographicsDraft>) => onChange({ ...value, ...patch });
  const text = (label: string, key: keyof DemographicsDraft, extra: { inputMode?: "numeric" } = {}) => (
    <label style={fieldLabel}>
      {label}
      <input value={value[key]} style={inputStyle} autoComplete="off" inputMode={extra.inputMode}
        onChange={(e) => set({ [key]: e.target.value } as Partial<DemographicsDraft>)} />
    </label>
  );
  return (
    <fieldset aria-label="dados do paciente para o faturamento" style={fieldset}>
      <div style={row}>
        <Tag tone="info" mono={false}>ajuda o faturamento</Tag>
        <p role="note" style={muted}>
          {missing && missing.length > 0 ? `Falta: ${missingText(missing)}. ${DEMOGRAPHICS_INCENTIVE}` : DEMOGRAPHICS_INCENTIVE}
        </p>
      </div>
      <div style={row}>
        <label style={fieldLabel}>
          Raça/cor
          <select value={value.raceColor} style={inputStyle} onChange={(e) => set({ raceColor: e.target.value })}>
            <option value="">não informada</option>
            {RACE_COLORS.map((r) => <option key={r.code} value={r.code}>{r.label}</option>)}
          </select>
        </label>
        {value.raceColor === INDIGENOUS && text("Etnia (código)", "ethnicity", { inputMode: "numeric" })}
      </div>
      <div style={row}>
        {text("CEP", "zip", { inputMode: "numeric" })}
        {text("Logradouro", "street")}
        {text("Número", "number")}
        {text("Complemento", "complement")}
        {text("Bairro", "district")}
      </div>
      {problem && <p role="alert" style={alert}>{problem}</p>}
    </fieldset>
  );
}

export function DemographicsForm({ title, missing, save, onSaved, onCancel }: {
  title: string; missing?: string[]; save(body: PatientDemographicsInput): Promise<void>; onSaved?(): void; onCancel?(): void;
}) {
  const [ draft, setDraft ] = useState<DemographicsDraft>(EMPTY_DEMOGRAPHICS);
  const [ error, setError ] = useState<string | null>(null);
  const [ saved, setSaved ] = useState(false);

  async function submit() {
    setError(null);
    if (!demographicsFilled(draft)) { setError("preencha a raça/cor ou o endereço"); return; }
    const problem = demographicsProblem(draft);
    if (problem) { setError(problem); return; }
    try {
      await save(demographicsBody(draft));
      setSaved(true);
      onSaved?.();
    } catch (err) {
      setError(demographicsError(err));
    }
  }

  return (
    <section aria-label={title} style={box}>
      <strong>{title}</strong>
      {saved ? <p role="status" style={statusStyle}>Dados do paciente salvos.</p> : (
        <>
          <DemographicsFields value={draft} onChange={(d) => { setError(null); setDraft(d); }} missing={missing} />
          {error && <p role="alert" style={alert}>{error}</p>}
          <div style={row}>
            <ActionButton onClick={submit}>Salvar dados do paciente</ActionButton>
            {onCancel && <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Agora não</button>}
          </div>
        </>
      )}
    </section>
  );
}

const fieldset: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, border: 0, margin: 0, padding: 0 };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

(`DemographicsForm` mostra o erro do envio uma vez só: o `problem` do `DemographicsFields` fica para a validação presencial, Task 15.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/demographics.test.ts src/modules/attendance/DemographicsForm.test.tsx src/lib/api.billing.test.ts src/modules/attendance/CheckIn.test.tsx src/theme && npx tsc --noEmit`
Expected: PASS (os testes do check-in seguem iguais: os tipos só ganharam campos opcionais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.billing.test.ts src/lib/demographics.ts src/lib/demographics.test.ts src/modules/attendance/DemographicsForm.tsx src/modules/attendance/DemographicsForm.test.tsx
$G commit -m "feat: add the patient demographics form for the BPA-I

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Recepção — raça/cor, etnia e endereço no check-in e na validação do cadastro

As telas reais onde a recepção cadastra e completa a pessoa são a **validação presencial** ("Validar cadastro com documento", o `Counter` de `src/modules/Attendance.tsx`, `POST /attendance/verifications`) e o **check-in** (`src/modules/attendance/CheckIn.tsx`), onde o 19 já completa os nomes do par validado por `POST /attendance/verifications/:id/names`. O api (contrato §12, D-U1, rotas fixadas no plano do api) aceita os 7 campos opcionais nas duas rotas, e no `/names` aceita também só eles, sem nomes; a resposta traz só `race_color_set` e `address_set`, nunca o endereço — a tela não reexibe o que foi salvo. O dashboard não tem outra tela de cadastro ou edição do paciente; nenhuma é criada.

**Files:**
- Modify: `src/modules/attendance/CheckIn.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/attendance/CheckIn.demographics.test.tsx`, `src/modules/Attendance.demographics.test.tsx`

**Interfaces:**
- Consumes: `saveVerificationDemographics`, `VerifyExtra` (com os 7 campos) (Task 14); `DemographicsForm`, `DemographicsFields`, `EMPTY_DEMOGRAPHICS`, `demographicsBody`, `demographicsFilled`, `demographicsProblem` (Task 14); `hasFeature`.
- Produces:
  - `CheckIn` ganha a prop `demographicsOn?: boolean` (padrão `false`; o `Attendance` passa `hasFeature(user, "procedure_billing")`). No fluxo por código, depois de "Atendimento iniciado", com `demographicsOn` e uma validação conhecida (a do par encontrado ou a criada agora pelo check-in), aparece a região "Completar dados do paciente" (opcional, "Agora não" fecha), que grava por `saveVerificationDemographics(verificationId, …)`. No fluxo por exceção a tela não conhece a validação do par: sem formulário (Divergência B14);
  - validação presencial: com `procedure_billing` ligado na sessão, os campos de `DemographicsFields` entram depois dos nomes; preenchidos, vão **no mesmo corpo** de `POST /attendance/verifications` (os 7 campos, planos, por `verifyCitizen(…, extra)`); começados e inválidos bloqueiam "Validar cadastro"; vazios, a chamada é a de hoje.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/CheckIn.demographics.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, lookupCheckIn: vi.fn(), checkIn: vi.fn(), searchCheckIn: vi.fn(), checkInByException: vi.fn(),
    completeCitizenNames: vi.fn(), saveVerificationDemographics: vi.fn() };
});

import * as api from "../../lib/api";
import { CheckIn } from "./CheckIn";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u7", name: "UPA Boa Vista", kind: "upa" };
const found = {
  citizen: { id: "c2", cpf_masked: "***.982.247-**", phone_masked: "(**) *****-5432", verification_level: "verified" as const, verification_id: "v9" },
  triage: { id: "t2", date: "2026-10-21T10:00:00Z", protocol_name: "triage-respiratoria", priority: 1 }, appointment: null
};

function renderCheckIn(demographicsOn = true) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  render(<CheckIn unit={unit} onUnitInvalid={vi.fn()} demographicsOn={demographicsOn} />, { wrapper });
}

async function checkInByCode() {
  fireEvent.change(screen.getByLabelText("CPF do cidadão (check-in)"), { target: { value: "52998224725" } });
  fireEvent.change(screen.getByLabelText("Código de check-in"), { target: { value: "123456" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar check-in" }));
  fireEvent.click(await screen.findByRole("button", { name: "Iniciar atendimento" }));
  await screen.findByText("Atendimento iniciado");
}

describe("CheckIn — dados do paciente para o BPA-I (módulo 22)", () => {
  beforeEach(() => {
    for (const fn of [ api.lookupCheckIn, api.checkIn, api.searchCheckIn, api.checkInByException, api.saveVerificationDemographics ]) mocked(fn).mockReset();
    mocked(api.lookupCheckIn).mockResolvedValue(found);
    mocked(api.checkIn).mockResolvedValue({ attendance: { id: "at1" }, verified: false });
    mocked(api.saveVerificationDemographics).mockResolvedValue({ race_color_set: true, address_set: false });
  });

  it("par validado: oferece completar e grava pela validação, só com os campos do paciente", async () => {
    renderCheckIn();
    await checkInByCode();
    const form = screen.getByRole("region", { name: "Completar dados do paciente" });
    expect(within(form).getByRole("note").textContent).toMatch(/ajuda|faturamento/);
    fireEvent.change(within(form).getByLabelText("Raça/cor"), { target: { value: "03" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    await waitFor(() => expect(api.saveVerificationDemographics).toHaveBeenCalledWith("v9", { race_color: "03" }));
    expect(await within(form).findByText("Dados do paciente salvos.")).not.toBeNull();
    expect(within(form).queryByLabelText("Raça/cor")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Próximo atendimento" }));
    expect(screen.queryByRole("region", { name: "Completar dados do paciente" })).toBeNull();
  });

  it("check-in que acabou de validar usa a validação nova; 'Agora não' fecha sem gravar", async () => {
    mocked(api.lookupCheckIn).mockResolvedValue({ ...found, citizen: { ...found.citizen, verification_level: "declared" as const, verification_id: null } });
    mocked(api.checkIn).mockResolvedValue({ attendance: { id: "at2" }, verified: true, verification_id: "v10" });
    renderCheckIn();
    await checkInByCode();
    const form = screen.getByRole("region", { name: "Completar dados do paciente" });
    fireEvent.change(within(form).getByLabelText("Raça/cor"), { target: { value: "01" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    await waitFor(() => expect(api.saveVerificationDemographics).toHaveBeenCalledWith("v10", { race_color: "01" }));
    cleanup();
    renderCheckIn();
    await checkInByCode();
    fireEvent.click(screen.getByRole("button", { name: "Agora não" }));
    expect(screen.queryByRole("region", { name: "Completar dados do paciente" })).toBeNull();
    expect(api.saveVerificationDemographics).toHaveBeenCalledTimes(1);
  });

  it("sem o interruptor, sem validação conhecida ou pela exceção: nenhum formulário", async () => {
    renderCheckIn(false);
    await checkInByCode();
    expect(screen.queryByRole("region", { name: "Completar dados do paciente" })).toBeNull();
    cleanup();
    mocked(api.lookupCheckIn).mockResolvedValue({ ...found, citizen: { ...found.citizen, verification_level: "declared" as const, verification_id: null } });
    renderCheckIn();
    await checkInByCode();
    expect(screen.queryByRole("region", { name: "Completar dados do paciente" })).toBeNull();
    cleanup();
    mocked(api.searchCheckIn).mockResolvedValue({
      triages: [ { id: "t1", date: "2026-10-21T10:00:00Z", protocol_name: "triage-respiratoria", priority: 2 } ], appointments: []
    });
    mocked(api.checkInByException).mockResolvedValue({ attendance: { id: "at3" } });
    renderCheckIn();
    fireEvent.click(screen.getByRole("button", { name: "Cidadão sem o código" }));
    fireEvent.change(screen.getByLabelText("CPF do cidadão (exceção)"), { target: { value: "52998224725" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar triagens" }));
    fireEvent.click(await screen.findByText("triage-respiratoria"));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "documento perdido" } });
    fireEvent.click(screen.getByRole("button", { name: "Iniciar atendimento" }));
    await screen.findByText("Atendimento iniciado");
    expect(screen.queryByRole("region", { name: "Completar dados do paciente" })).toBeNull();
  });
});
```

```tsx
// src/modules/Attendance.demographics.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), lookupCitizen: vi.fn(), verifyCitizen: vi.fn(), listActiveUnits: vi.fn(),
    listUnassignedRequests: vi.fn(), getMyProfessional: vi.fn(), listPendingErasures: vi.fn()
  };
});

import * as api from "../lib/api";
import { AuthProvider } from "../lib/auth";
import { Attendance } from "./Attendance";

afterEach(() => { cleanup(); localStorage.clear(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const found = {
  citizen: { id: "c1", cpf_masked: "***.982.247-**", phone_masked: "(**) *****-5432", created_at: "2026-09-20T10:00:00Z", verification_level: "declared" as const },
  triages: []
};
const PROFILE = { birth_date: "1990-05-10", sex: "female", gender_identity: null, full_name: "Rosa Lima" };

function session(features: string[]): api.SessionUser {
  return { id: "u1", email_address: "a@cidade.gov.br", operator: false, mfa_enrolled: true, mfa_verified_at: null, features,
    memberships: [ { city_slug: "m1", city_name: "Curitiba", city_uf: "PR", role: "citizen_verifier" } ] };
}

function renderAttendance() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  render(<Attendance onNavigate={vi.fn()} />, { wrapper });
}

async function fillValidation() {
  fireEvent.change(await screen.findByLabelText("CPF do cidadão (validação)"), { target: { value: "52998224725" } });
  fireEvent.change(screen.getByLabelText("Código de validação"), { target: { value: "123456" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar validação" }));
  await screen.findByText("(**) *****-5432");
  fireEvent.click(screen.getByLabelText("Conferi o documento com foto e o CPF confere"));
  fireEvent.change(screen.getByLabelText("Data de nascimento (documento)"), { target: { value: "1990-05-10" } });
  fireEvent.change(screen.getByLabelText("Sexo (documento)"), { target: { value: "female" } });
  fireEvent.change(screen.getByLabelText("Nome completo (documento)"), { target: { value: "Rosa Lima" } });
}

describe("Attendance — dados do paciente na validação (módulo 22)", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.lookupCitizen, api.verifyCitizen, api.listActiveUnits, api.listUnassignedRequests,
      api.getMyProfessional, api.listPendingErasures ]) mocked(fn).mockReset();
    mocked(api.lookupCitizen).mockResolvedValue(found);
    mocked(api.verifyCitizen).mockResolvedValue(undefined);
    mocked(api.listActiveUnits).mockResolvedValue([]);
    mocked(api.listUnassignedRequests).mockResolvedValue([]);
    mocked(api.getMyProfessional).mockResolvedValue(null);
    mocked(api.listPendingErasures).mockResolvedValue([]);
  });

  it("com procedure_billing: campos opcionais com incentivo; preenchidos vão planos no corpo da validação", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session([ "procedure_billing" ]));
    renderAttendance();
    await fillValidation();
    expect(screen.getByRole("group", { name: "dados do paciente para o faturamento" })).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Raça/cor"), { target: { value: "05" } });
    const validate = screen.getByRole("button", { name: "Validar cadastro" }) as HTMLButtonElement;
    expect(validate.disabled).toBe(true);
    expect(screen.getByText("para indígena, informe a etnia: código de 4 dígitos da tabela do DATASUS")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Etnia (código)"), { target: { value: "0001" } });
    fireEvent.change(screen.getByLabelText("CEP"), { target: { value: "80010-000" } });
    fireEvent.change(screen.getByLabelText("Logradouro"), { target: { value: "RUA XV DE NOVEMBRO" } });
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "CENTRO" } });
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456", PROFILE, {
      race_color: "05", ethnicity: "0001", address_zip: "80010000", address_street: "RUA XV DE NOVEMBRO", address_number: "SN",
      address_district: "CENTRO"
    }));
  });

  it("vazios não mudam a chamada de hoje; sem o interruptor, nem aparecem", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(session([ "procedure_billing" ]));
    renderAttendance();
    await fillValidation();
    fireEvent.click(screen.getByRole("button", { name: "Validar cadastro" }));
    await waitFor(() => expect(api.verifyCitizen).toHaveBeenCalledWith("529.982.247-25", "123456", PROFILE));
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(session([]));
    renderAttendance();
    await fillValidation();
    expect(screen.queryByLabelText("Raça/cor")).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/attendance/CheckIn.demographics.test.tsx src/modules/Attendance.demographics.test.tsx`
Expected: FAIL — não há `demographicsOn` no check-in nem os campos na validação.

- [ ] **Step 3: Write minimal implementation**

`src/modules/attendance/CheckIn.tsx`:

1. Imports — depois de `import { CompleteNames } from "./CompleteNames";`:

```ts
import { DemographicsForm } from "./DemographicsForm";
```

e acrescente `saveVerificationDemographics` ao import existente de `../../lib/api` (o que já traz `checkIn`, `checkInByException`…).

2. Em `Props`, depois de `onUnitInvalid(): void;`:

```ts
  // Módulo 22 (contrato §12, D-U1): `procedure_billing` ligado na sessão.
  demographicsOn?: boolean;
```

3. Troque `export function CheckIn({ unit, onUnitInvalid }: Props) {` por `export function CheckIn({ unit, onUnitInvalid, demographicsOn = false }: Props) {` e

```tsx
        ? <CodeFlow unit={unit} onUnitInvalid={onUnitInvalid} cpf={cpf} onCpf={setCpf} />
```

por

```tsx
        ? <CodeFlow unit={unit} onUnitInvalid={onUnitInvalid} cpf={cpf} onCpf={setCpf} demographicsOn={demographicsOn} />
```

(o `FlowProps extends Props` já tem o campo; o `ExceptionFlow` não o recebe: a tela não conhece a validação do par nesse fluxo — Divergência B14).

4. No `CodeFlow`, troque a assinatura `function CodeFlow({ unit, onUnitInvalid, cpf, onCpf: setCpf }: FlowProps) {` por

```tsx
function CodeFlow({ unit, onUnitInvalid, cpf, onCpf: setCpf, demographicsOn = false }: FlowProps) {
```

depois de `const [ newVerificationId, setNewVerificationId ] = useState<string | null>(null);`:

```ts
  // Módulo 22: raça/cor e endereço pela validação do par (POST /attendance/verifications/:id/names, só os campos do paciente).
  const [ demographicsClosed, setDemographicsClosed ] = useState(false);
```

no `reset()`, troque `    setNewVerificationId(null);` por

```ts
    setNewVerificationId(null);
    setDemographicsClosed(false);
```

e, no estado `done`, troque `          {newVerificationId && <CompleteNames verificationId={newVerificationId} />}` por

```tsx
          {newVerificationId && <CompleteNames verificationId={newVerificationId} />}
          {(() => {
            const verificationId = newVerificationId ?? found?.citizen.verification_id ?? null;
            if (!demographicsOn || !verificationId || demographicsClosed) return null;
            return (
              <DemographicsForm key={verificationId} title="Completar dados do paciente"
                save={async (body) => { await saveVerificationDemographics(verificationId, body); }}
                onCancel={() => setDemographicsClosed(true)} />
            );
          })()}
```

`src/modules/Attendance.tsx`:

1. Imports — depois de `import { UnitProcedures } from "./billing/UnitProcedures";` (Task 8):

```ts
import { DemographicsFields } from "./attendance/DemographicsForm";
import { EMPTY_DEMOGRAPHICS, demographicsBody, demographicsFilled, demographicsProblem, type DemographicsDraft } from "../lib/demographics";
import type { VerifyExtra } from "../lib/api";
```

2. Troque `      {canVerify && unit && <div id="sec-checkin"><CheckIn unit={unit} onUnitInvalid={onUnitInvalid} /></div>}` por

```tsx
      {canVerify && unit && <div id="sec-checkin"><CheckIn unit={unit} onUnitInvalid={onUnitInvalid} demographicsOn={hasFeature(user, "procedure_billing")} /></div>}
```

e `      {canVerify && <div id="sec-balcao"><Counter cadsusOn={hasFeature(user, "cadsus_lookup")} /></div>}` por

```tsx
      {canVerify && <div id="sec-balcao"><Counter cadsusOn={hasFeature(user, "cadsus_lookup")} demographicsOn={hasFeature(user, "procedure_billing")} /></div>}
```

3. Troque `function Counter({ cadsusOn }: { cadsusOn: boolean }) {` por

```tsx
// `demographicsOn` (módulo 22): raça/cor, etnia e endereço opcionais para o BPA-I, no mesmo POST da validação.
function Counter({ cadsusOn, demographicsOn = false }: { cadsusOn: boolean; demographicsOn?: boolean }) {
  const [ demographics, setDemographics ] = useState<DemographicsDraft>(EMPTY_DEMOGRAPHICS);
```

4. No `reset()` do `Counter`, troque `    setCadsusConfirmed(false);` por

```ts
    setCadsusConfirmed(false);
    setDemographics(EMPTY_DEMOGRAPHICS);
```

5. Troque `  const blocked = !checked || busy || profileProblem !== null || namesIssue !== null;` por

```ts
  // Opcionais: vazios não bloqueiam; começados, precisam estar certos.
  const demographicsIssue = demographicsOn && demographicsFilled(demographics) ? demographicsProblem(demographics) : null;
  const blocked = !checked || busy || profileProblem !== null || namesIssue !== null || demographicsIssue !== null;
```

6. Troque

```ts
      if (cadsusOn) await verifyCitizen(cpf, code, profileBody, { cadsus_confirmed: cadsusConfirmed });
      else await verifyCitizen(cpf, code, profileBody);
```

por

```ts
      const extra: VerifyExtra = demographicsOn && demographicsFilled(demographics) ? demographicsBody(demographics) : {};
      if (cadsusOn) extra.cadsus_confirmed = cadsusConfirmed;
      if (Object.keys(extra).length > 0) await verifyCitizen(cpf, code, profileBody, extra);
      else await verifyCitizen(cpf, code, profileBody);
```

7. Troque `            <NamesFields value={names} onChange={setNames} problem={namesIssue} />` por

```tsx
            <NamesFields value={names} onChange={setNames} problem={namesIssue} />
            {demographicsOn && <DemographicsFields value={demographics} onChange={setDemographics} problem={demographicsIssue} />}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/attendance src/modules/Attendance.test.tsx src/modules/Attendance.demographics.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos do check-in e da validação seguem iguais: sem o interruptor nada muda).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/attendance/CheckIn.tsx src/modules/attendance/CheckIn.demographics.test.tsx src/modules/Attendance.tsx src/modules/Attendance.demographics.test.tsx
$G commit -m "feat: capture race, ethnicity and address at check-in and citizen validation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Pendência do faturista — completar os dados do paciente e reler

**Files:**
- Modify: `src/modules/billing/PendingPanel.tsx`, `src/modules/BillingPeriods.tsx` (uma linha), `src/lib/billing.ts` (motivos e dica da correção do paciente), `src/lib/billing.test.ts`
- Test: `src/modules/BillingPeriods.demographics.test.tsx`

**Interfaces:**
- Consumes: `updatePatientDemographics` (Task 14); `DemographicsForm` (Task 14); `PendingPanel` (Task 11); `BillingPeriods` (Task 12).
- Produces: `PendingPanel({ period, onClose, canFix? })`; com `canFix` (só o `billing_clerk`), item com `patient` e motivo `missing_race_color` ou `missing_address` ganha "Completar dados" → região "Completar dados de <nome>" → `PATCH /billing/patients/:id/demographics` → a lista de pendências é relida e a tela diz "Dados de <nome> salvos; as pendências foram conferidas de novo."; `reasonLabel("missing_race_color")` = "paciente sem raça/cor"; `reasonLabel("missing_address")` = "paciente sem CEP ou endereço"; `demographicsMissingOf(reasons): string[]`.

- [ ] **Step 1: Write the failing test**

No `describe("veredito da crítica")` de `src/lib/billing.test.ts` (acrescente `demographicsMissingOf` ao import de `./billing`):

```ts
  it("motivos dos dados do paciente (contrato §12, D-U1)", () => {
    expect(reasonLabel("missing_race_color")).toBe("paciente sem raça/cor");
    expect(reasonLabel("missing_address")).toBe("paciente sem CEP ou endereço");
    expect(demographicsMissingOf([ "missing_address", "missing_patient_id", "missing_race_color" ])).toEqual([ "race_color", "address" ]);
    expect(demographicsMissingOf([ "missing_patient_id" ])).toEqual([]);
  });
```

```tsx
// src/modules/BillingPeriods.demographics.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listBillingPeriods: vi.fn(), getPeriodPending: vi.fn(), updatePatientDemographics: vi.fn() };
});

import * as api from "../lib/api";
import { BillingPeriods } from "./BillingPeriods";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY22, billingUser, pending, period } from "../test/billingFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ROSA = pending({
  procedure_id: "pp8", fix: "patient", reasons: [ "missing_race_color", "missing_address" ], patient: { id: "pa3", display_name: "Rosa Lima" }
});
const NO_ID = pending({ procedure_id: "pp9", fix: "patient", reasons: [ "missing_patient_id" ], patient: { id: "pa4", display_name: "Caio Souza" } });
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.listBillingPeriods, api.getPeriodPending, api.updatePatientDemographics ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "billing_clerk" ]));
  mocked(api.listBillingPeriods).mockResolvedValue([ period() ]);
  mocked(api.getPeriodPending).mockResolvedValue([ ROSA, NO_ID ]);
  mocked(api.updatePatientDemographics).mockResolvedValue(undefined);
});

async function openPending() {
  renderWithProviders(<BillingPeriods today={TODAY22} />);
  fireEvent.click(await screen.findByRole("button", { name: "Pendências" }));
  const box = await screen.findByRole("region", { name: "Pendências de 09/2026 · UPA Boa Vista" });
  await within(box).findByText("Cadastro do paciente · 2");
  return box;
}

describe("Pendência do faturista — dados do paciente (módulo 22, D-U1)", () => {
  it("raça/cor e endereço: completa pelo faturamento e a lista de pendências é relida", async () => {
    const box = await openPending();
    expect(within(box).getAllByRole("button", { name: "Completar dados" })).toHaveLength(1);
    fireEvent.click(within(box).getByRole("button", { name: "Completar dados" }));
    const form = within(box).getByRole("region", { name: "Completar dados de Rosa Lima" });
    expect(within(form).getByRole("note").textContent).toMatch(/^Falta: raça\/cor; CEP e endereço\./);
    fireEvent.change(within(form).getByLabelText("Raça/cor"), { target: { value: "05" } });
    fireEvent.change(within(form).getByLabelText("CEP"), { target: { value: "80010-000" } });
    fireEvent.change(within(form).getByLabelText("Logradouro"), { target: { value: "RUA XV DE NOVEMBRO" } });
    fireEvent.change(within(form).getByLabelText("Número"), { target: { value: "100" } });
    fireEvent.change(within(form).getByLabelText("Bairro"), { target: { value: "CENTRO" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    expect(within(form).getByRole("alert").textContent).toBe("para indígena, informe a etnia: código de 4 dígitos da tabela do DATASUS");
    fireEvent.change(within(form).getByLabelText("Etnia (código)"), { target: { value: "0001" } });
    mocked(api.getPeriodPending).mockResolvedValue([ NO_ID ]);
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    await waitFor(() => expect(api.updatePatientDemographics).toHaveBeenCalledWith("pa3", {
      race_color: "05", ethnicity: "0001", address_zip: "80010000", address_street: "RUA XV DE NOVEMBRO", address_number: "100",
      address_district: "CENTRO"
    }));
    expect((await within(box).findByRole("status")).textContent).toBe("Dados de Rosa Lima salvos; as pendências foram conferidas de novo.");
    await waitFor(() => expect(api.getPeriodPending).toHaveBeenCalledTimes(2));
    expect(await within(box).findByText("Cadastro do paciente · 1")).not.toBeNull();
    expect(within(box).queryByText("Rosa Lima")).toBeNull();
  });

  it("422 invalid_zip vira frase e nada é relido; o admin não corrige", async () => {
    mocked(api.updatePatientDemographics).mockRejectedValueOnce(new api.ApiError(422, { error: "invalid_zip" }, "x"));
    const box = await openPending();
    fireEvent.click(within(box).getByRole("button", { name: "Completar dados" }));
    const form = within(box).getByRole("region", { name: "Completar dados de Rosa Lima" });
    fireEvent.change(within(form).getByLabelText("Raça/cor"), { target: { value: "03" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar dados do paciente" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("CEP inválido — confira os 8 dígitos");
    expect(api.getPeriodPending).toHaveBeenCalledTimes(1);
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(billingUser([ "municipal_admin" ]));
    const adminBox = await openPending();
    expect(within(adminBox).queryByRole("button", { name: "Completar dados" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/BillingPeriods.demographics.test.tsx src/lib/billing.test.ts`
Expected: FAIL — não há "Completar dados", `demographicsMissingOf` não existe e os motivos novos saem crus.

- [ ] **Step 3: Write minimal implementation**

`src/lib/billing.ts` — em `REASON_LABEL`, troque `  missing_establishment_cnes: "estabelecimento sem CNES"` por

```ts
  missing_establishment_cnes: "estabelecimento sem CNES",
  // Contrato §12, D-U1: dados do paciente que o BPA-I exige.
  missing_race_color: "paciente sem raça/cor",
  missing_address: "paciente sem CEP ou endereço"
```

em `FIX_INFO`, troque a linha de `patient:` por

```ts
  patient: { title: "Cadastro do paciente", hint: "CPF ou CNS na validação presencial do Atendimento; raça/cor e endereço o faturista completa aqui" },
```

e, depois de `export function fixInfo(…) { … }`:

```ts
// Motivos que o faturista corrige pela pendência (PATCH /billing/patients/:id/demographics).
export function demographicsMissingOf(reasons: string[]): string[] {
  const out: string[] = [];
  if (reasons.includes("missing_race_color")) out.push("race_color");
  if (reasons.includes("missing_address")) out.push("address");
  return out;
}
```

`src/modules/billing/PendingPanel.tsx` — reescreva por inteiro:

```tsx
// src/modules/billing/PendingPanel.tsx
// Pendências da competência (contrato §5; Divergência B6): críticas
// recalculadas com o cadastro de agora, agrupadas pelo que corrigir. A rota
// grava trilha (contém paciente): lê ao abrir, no "Conferir de novo" e depois
// de o faturista completar raça/cor ou endereço (contrato §12, D-U1) — nunca
// sozinha. Corrigido o cadastro, a pendência sai sem relançar (spec §2).
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { getPeriodPending, updatePatientDemographics, type BillingPeriod, type PendingItem } from "../../lib/api";
import { PENDING_KEY, billingError, demographicsMissingOf, establishmentLabel, fixInfo, groupPending, reasonLabel } from "../../lib/billing";
import { competenceLabel } from "../../lib/competence";
import { fmtDay } from "../../lib/exams";
import { DataTable, type Column } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { secondaryButtonStyle } from "../../components/formStyles";
import { DemographicsForm } from "../attendance/DemographicsForm";
import { alert, box, muted, panel, row } from "../exams/styles";

export function PendingPanel({ period, onClose, canFix = false }: { period: BillingPeriod; onClose(): void; canFix?: boolean }) {
  const query = useQuery({
    queryKey: [ PENDING_KEY, period.id ], queryFn: () => getPeriodPending(period.id), gcTime: 0, staleTime: 0, retry: false
  });
  const [ fixing, setFixing ] = useState<PendingItem | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const groups = groupPending(query.data ?? []);
  const fixable = (i: PendingItem) => canFix && !!i.patient && demographicsMissingOf(i.reasons).length > 0;

  const cols: Column<PendingItem>[] = [
    { label: "Data", w: "0.8fr", render: (i) => fmtDay(i.performed_on ?? null) },
    { label: "Procedimento", w: "2fr", card: "title", render: (i) => (i.sigtap_code ? `${i.sigtap_code} — ${i.label ?? ""}` : i.procedure_id) },
    { label: "Paciente", w: "1.2fr", render: (i) => i.patient?.display_name ?? "—" },
    { label: "Profissional", w: "1.2fr", render: (i) => i.professional?.name ?? "—" },
    { label: "Motivo", w: "2fr", render: (i) => i.reasons.map(reasonLabel).join("; ") },
    ...(canFix ? [ { label: "", w: "auto", align: "right" as const, card: "actions" as const, render: (i: PendingItem) => (fixable(i) ? (
      <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setFixing(i); }}>Completar dados</button>
    ) : null) } ] : [])
  ];

  return (
    <section aria-label={`Pendências de ${competenceLabel(period.competence)} · ${period.establishment.name}`} style={panel}>
      <div style={row}>
        <strong style={{ flex: 1 }}>{`Pendências de ${competenceLabel(period.competence)} · ${establishmentLabel(period.establishment)}`}</strong>
        <button type="button" style={secondaryButtonStyle} disabled={query.isFetching} onClick={() => void query.refetch()}>Conferir de novo</button>
        <button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar pendências</button>
      </div>
      <p style={muted}>
        As críticas são recalculadas com o cadastro de agora. Corrigido o cadastro, a pendência sai daqui — nada precisa ser relançado.
        Pendência não entra no arquivo.
      </p>
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      {query.isPending && <p className="mono" style={muted}>conferindo…</p>}
      {query.isError && <p role="alert" style={alert}>{billingError(query.error)}</p>}
      {query.isSuccess && groups.length === 0 && <EmptyState title="nenhuma pendência nesta competência" />}
      {groups.map((g) => {
        const info = fixInfo(g.fix);
        return (
          <section key={g.fix} aria-label={info.title} style={box}>
            <strong>{`${info.title} · ${g.items.length}`}</strong>
            {info.hint && <p style={muted}>{info.hint}</p>}
            <DataTable<PendingItem> cols={cols} rows={g.items} rowKey={(i) => i.procedure_id} />
          </section>
        );
      })}
      {fixing?.patient && (
        <DemographicsForm key={fixing.procedure_id} title={`Completar dados de ${fixing.patient.display_name}`}
          missing={demographicsMissingOf(fixing.reasons)}
          save={(body) => updatePatientDemographics(fixing.patient!.id, body)}
          onSaved={() => {
            setNotice(`Dados de ${fixing.patient!.display_name} salvos; as pendências foram conferidas de novo.`);
            setFixing(null);
            void query.refetch();
          }}
          onCancel={() => setFixing(null)} />
      )}
    </section>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

`src/modules/BillingPeriods.tsx` — troque

```tsx
      {opened?.kind === "pending" && <PendingPanel key={opened.period.id} period={opened.period} onClose={() => setOpened(null)} />}
```

por

```tsx
      {opened?.kind === "pending" && (
        <PendingPanel key={opened.period.id} period={opened.period} canFix={clerk} onClose={() => setOpened(null)} />
      )}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/BillingPeriods*.test.tsx src/lib/billing.test.ts src/theme && npx tsc --noEmit`
Expected: PASS (os testes das Tasks 11 e 12 seguem iguais: a coluna nova só existe para o faturista e o item deles não tem os motivos novos).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/billing.ts src/lib/billing.test.ts src/modules/billing/PendingPanel.tsx src/modules/BillingPeriods.tsx src/modules/BillingPeriods.demographics.test.tsx
$G commit -m "feat: let the billing clerk complete patient demographics from the pending list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: Suíte, build, revisão e prova no navegador (api real na 3044)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–16 na branch `feat/mod-22-billing`; o api do 22 na **3044** com a semente do 22 (inclusive um paciente sem raça/cor nem endereço, com procedimento faturável na UPA).
- Produces: branch pronta para merge (depois do merge do api do 22); evidências em `evidencias/2026-10-21-modulo-22/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **20 arquivos de teste novos** (`api.billing`, `billing`, `modules.billing`, `timeline.procedure`, `PatientTimeline.procedure`, `ProcedureForm`, `ProceduresPanel`, `UnitQueue.procedures`, `ConsultationWorkspace.procedures`, `UnitProcedures`, `BillingProcedures`, `ManualEntryPanel`, `BillingPeriods`, `BillingPeriods.close`, `BillingSettings`, `demographics`, `DemographicsForm`, `CheckIn.demographics`, `Attendance.demographics`, `BillingPeriods.demographics`) e cinco arquivos de teste existentes com casos novos ou ajustados (`features.test.ts`, `modules.test.ts`, `team.test.ts`, `timeline.test.ts`, `timeline.exam.test.ts`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/SegmentedControl.tsx src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css package.json package-lock.json
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css` ou dependência mudados.

- [ ] **Step 2: Nada vaza e nada relê sozinho**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage" src/lib/billing.ts src/lib/demographics.ts src/modules/billing src/modules/Billing*.tsx src/modules/attendance/DemographicsForm.tsx | grep -v "\.test\."
grep -rn "refetchInterval\|refetchOnWindowFocus" src/lib/billing.ts src/modules/billing src/modules/Billing*.tsx | grep -v "\.test\."
sed -n '/─── Faturamento de procedimentos (módulo 22/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [a-z_, ]* })'
grep -rn "maxLength" src/modules/billing src/modules/Billing*.tsx
```

Expected: o primeiro, o segundo e o último sem saída; o terceiro só com `establishment_type`, `establishment_id`, `competence`, `billable`, `reason`, `patient_id`, `attendance_id`, `before`, `limit` e os `URLSearchParams` de `{ q, competence }`, `{ establishment_type, establishment_id }` e `{ competence }` — nenhum `cpf`, `cns`, `name`, `cnpj` ou motivo de texto.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§2–§9), o ADR 0039, o contrato (§1–§6, §9, §10) e as Divergências deste plano. Pontos de atenção:
- sem `procedure_billing`, atendimento, consulta, linha do tempo, Equipe e menu são exatamente os de hoje (testes antigos verdes; só as contagens de `modules.test.ts`, a lista de `team.test.ts` e as de `timeline*.test.ts` mudaram);
- o formulário nunca bloqueia por crítica SIGTAP (CID ausente só avisa); a frase "não faturável — motivo" vem dos códigos do api;
- UBS sem Procedimentos em lugar nenhum; o profissional cancela só os próprios; o faturista cancela qualquer um;
- nenhuma consulta do faturamento com `refetchInterval`; pendências só no clique;
- fechar, reabrir e salvar a configuração passam pelo `SensitiveAction` com step-up; reabrir exige motivo de 10 a 500; larguras do cabeçalho recusadas, nunca cortadas;
- download só do faturista, pelo `saveFile`, com nome sem dado pessoal; nada em console/storage;
- dados do paciente (D-U1): raça/cor sem "sem informação", etnia só para indígena, larguras recusadas; na recepção opcionais, nunca bloqueiam check-in nem validação vazia; o faturista completa pela pendência e a lista é relida; endereço e raça/cor nunca na URL;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 22 na **3044** (plano do api: servidor do worktree `apps/api/.claude/mod22` no `api-dev`, semente do 22 com a "UPA Boa Vista" com CNES, uma profissional médica com CNS e vínculo na UPA, uma enfermeira **sem** CNS com vínculo na UPA, a conta do faturista, o laboratório municipal com CNES e um laudo liberado; `procedure_billing` em `Platform::Features::CATALOG`). Ligue em Curitiba, pelo maintenance, `procedure_billing` (e, para a consulta, `clinical_record`; para o laboratório, `exam_tracking` e `municipal_lab`). O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo: o do worktree é do macOS e não roda no Alpine):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod22 --network rota-saude_default -p 5197:5197 \
  -v $R/.worktrees/dashboard-mod22:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3044 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5197 --host 0.0.0.0"
docker logs -f dashboard-mod22   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 22 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5197/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot (salve em `evidencias/2026-10-21-modulo-22/dashboard/`):
1. **Admin** — Equipe → convidar com o papel **Faturista** (step-up; o usuário confirma o envio). Faturamento → **Configuração do faturamento**: um nome de origem com 31 caracteres é recusado com a frase da largura; com os valores certos, salva (step-up) e os cinco campos aparecem.
2. **Profissional na UPA** — Atendimento → UPA Boa Vista → chamar → (modo record) **Consulta** mostra "Procedimentos realizados"; (modo integrado) o botão **Procedimentos** na linha. Registrar um atendimento de urgência (busca SIGTAP, quantidade 1) → "faturável · BPA-I"; registrar uma sutura **sem CID** → aviso no formulário, grava, e a lista mostra "não faturável — o procedimento exige CID-10". Cancelar a sutura com motivo → "cancelado".
3. **Profissional na UBS** — nada de Procedimentos no atendimento, na consulta nem o avulso.
4. **Avulso** — na UPA, "Procedimento avulso": paciente pelo CPF, data de ontem, registrar; a lista mostra o próprio procedimento.
5. **Laboratório** — com um laudo liberado no 21b (ou o da semente), **Faturamento → Procedimentos realizados** com "Estabelecimento: Laboratório Municipal" mostra os itens com origem "laboratório municipal" (e retificar o laudo não cria procedimento novo).
6. **Faturista** — Procedimentos realizados: filtrar por não faturáveis e motivo; **Lançamento manual** (estabelecimento, paciente, enfermeira com dois CBOs → escolher o CBO) → "não faturável — profissional sem CNS no cadastro".
7. **Competências** — 10/2026 (a corrente, para ver a produção da prova): resumo por estabelecimento; **Pendências** → grupo "Cadastro do profissional" com a enfermeira sem CNS. Como admin, cadastrar o CNS dela em Equipe → Profissionais; voltar e **Conferir de novo** → a pendência sai sem relançar.
7a. **Paciente sem os dados (D-U1)** — antes, na recepção: no check-in por código de uma pessoa da semente com par validado e sem raça/cor nem endereço, aparece "Completar dados do paciente" com a nota "ajuda o faturamento"; clique **Agora não** (o check-in segue normal) e registre para ela um procedimento faturável na UPA. Em outro check-in, preencha só a raça/cor e salve: "Dados do paciente salvos." e nada do que foi salvo é reexibido. Na validação presencial de outra pessoa, os campos aparecem opcionais; raça/cor "Indígena" sem etnia bloqueia só até a etnia ser informada. Em **Pendências**, o grupo "Cadastro do paciente" mostra a primeira pessoa com "paciente sem raça/cor; paciente sem CEP ou endereço"; como faturista, **Completar dados** → raça/cor, CEP, logradouro, número e bairro → **Salvar dados do paciente** → "Dados de … salvos; as pendências foram conferidas de novo." e ela sai da lista; no fechamento (passo 8), o procedimento dela entra no BPA-I. Como admin, a pendência aparece sem o botão.
8. **Fechar e gerar arquivo** (step-up) → "fechada · versão 1"; **Versões** → **Baixar versão 1** → o arquivo cai em Downloads com o nome sem dado pessoal; confira no terminal o início e o fim de linha (`head -c 140 ~/Downloads/bpa-*.txt | od -c | head -12` → começa com `01#BPA#`, competência, e as linhas terminam em `\r \n`). Não abra o arquivo em ferramenta online.
9. **Reabrir** com motivo curto → recusado; com motivo de verdade → "aberta · reaberta depois da versão 1"; fechar de novo → versão 2; as duas em Versões. Como admin, Versões aparece sem "Baixar".
10. **Atraso** — com a competência fechada, registrar como profissional um procedimento com data dela (avulso) → a lista mostra "apresentação em atraso".
11. **Linha do tempo** — no atendimento (ou na abertura justificada), o filtro **Procedimento** existe e a entrada mostra nome, data, unidade e profissional, sem abrir detalhe.
12. **Sem polling** — com a aba de rede aberta em Competências e em Procedimentos por 2 minutos, nenhuma chamada a `/billing/*` sem ação da pessoa.
13. **Telefone** (largura 375): Procedimentos, Competências, Pendências e Versões viram cartões, sem rolagem lateral.
14. No maintenance, desligar `procedure_billing` e recarregar: o grupo Faturamento some, o botão/painel de Procedimentos some do atendimento e da consulta, o filtro Procedimento some da linha do tempo; religar devolve tudo.

Depois: `docker rm -f dashboard-mod22`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

Para cada card: F-22.2, F-22.4 e F-22.5 vão a **Done** se a parte do api já está Done; F-22.1 só quando api **e** maintenance também estiverem; F-22.6 quando api **e** contracts estiverem (e segue **sem Verified** até a importação real no BPA Magnético — gate de go-live). Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified** (F-22.3 e F-22.7 são do api; mova se a prova 5 e 11 passaram e o plano do api já os deixou em Done). Use o mesmo `gh project item-edit` da Task 0, com o id da opção Done `98236657` (a de Verified, da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 22, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard → maintenance (contrato §11). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-22-billing
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod22
/opt/homebrew/bin/git branch -d feat/mod-22-billing
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (B1–B13). As que só nomeiam forma (B4, B6, B9, B11) podem ser resolvidas pelo api escolhendo outra forma, desde que o contrato a fixe e este plano ajuste `src/lib/api.ts` e `src/test/billingFixtures.ts`.

1. **B1 — Paciente e estabelecimento a partir do atendimento.** O §3 exige `patient_id` no `POST /billing/procedures`, mas a fila do atendimento (`QueueRow`) não traz o paciente e, no modo integrado (sem prontuário), não há consulta. Proposta: com `attendance_id` (ou `consultation_id`), `patient_id` é opcional e o api deriva o paciente do atendimento; se vier e não bater, 422 `invalid_patient`. O estabelecimento continua no corpo (a unidade do atendimento); se não bater com a do atendimento, 422 `invalid_establishment`.
2. **B2 — Lista por atendimento.** O §3 só filtra o profissional por `patient_id`. Proposta: `GET /billing/procedures?attendance_id=` para o profissional com vínculo na unidade do atendimento (mesma trilha da lista). O dashboard não usa o `procedures` embutido no JSON de `attendance`/`consultation` (§3, último item): pode ficar ou sair, sem efeito na tela.
3. **B3 — Paciente pelo CPF no faturamento.** O avulso e o lançamento manual precisam achar o paciente; a busca do 21a (`/exams/patient_lookup`) exige `exam_tracking`. Proposta: `POST /billing/patient_lookup { "cpf" }` → `{ "patient": { "id", "display_name", "cpf_masked" } }` (profissional com vínculo em unidade não-APS e `billing_clerk`; trilha; 404 `patient_not_found`); CPF só no corpo.
4. **B4 — Estabelecimentos faturantes.** Filtros e lançamento manual precisam da lista. Proposta: `GET /billing/establishments` → `{ "establishments": [<establishment>] }` (unidades ativas não-APS e laboratórios com `uses_municipal_lab`; `cnes` pode ser `null`), para `billing_clerk` e `municipal_admin`.
5. **B5 — Profissional no lançamento manual.** O corpo do §3 não tem profissional, mas o manual é produção de outra pessoa. Proposta: `professional_user_id` e `cbo_code` no corpo, aceitos **só** do `billing_clerk` (a presença de `professional_user_id` marca `source: manual`; sem ele, o registro é do próprio profissional da sessão); 422 `invalid_professional` sem vínculo com o estabelecimento; `cbo_code` opcional (ausente = o do vínculo; profissional sem CBO grava com `missing_professional_cbo`). E `GET /billing/professionals?establishment_type=&establishment_id=` → `{ "professionals": [ { "user_id", "name", "cbo_codes": [string] } ] }` (vínculos ativos na unidade; liberadores no laboratório), só `billing_clerk`.
6. **B6 — Pendência com o que a tela mostra.** O §5 devolve só `procedure_id`, `reasons` e `fix`; o faturista precisa saber de quem e do quê. Proposta: cada item ganha `performed_on`, `sigtap_code`, `label`, `patient: { id, display_name }` e `professional: { user_id, name }` (a rota já grava trilha por conter paciente). Correção nova (`fix` fora dos quatro) aparece crua no fim.
7. **B7 — `late` no procedimento.** O §5 diz "`late: true` no item", mas `<performed_procedure>` (§2) não tem o campo. Proposta: `"late": bool` em `<performed_procedure>` (ausente = `false`).
8. **B8 — Recusas e limites de texto.** Proposta: cancelar procedimento já incluído em arquivo de competência fechada → 409 `already_exported`; motivo de cancelamento e de reabertura de 10 a 500 caracteres → 422 `reason_required` (recusado, nunca cortado); fechar/reabrir na situação errada → 409 `invalid_transition`.
9. **B9 — Envelope das versões e nome do arquivo.** Proposta: `GET /billing/periods/:id/exports` → `{ "exports": [...] }`, da mais nova à mais antiga, `generated_by` = nome; `GET /billing/exports/:id/file` com `Content-Disposition` `bpa-<cnes>-<AAAAMM>-v<versão>.txt` (sem dado pessoal); a lista é do `billing_clerk` e do `municipal_admin`, o arquivo só do `billing_clerk`.
10. **B10 — Configuração.** O cabeçalho do layout do DATASUS de 09/07/2026 tem a **sigla** do órgão de origem (`cbc-sgl`, 6) além do nome (`cbc-rsp`, 30), CNPJ (`cbc-cgccpf`, 14), destino (`cbc-dst`, 40) e indicador M/E. Proposta: `origin_acronym` em `GET/PUT /billing/settings`; larguras recusadas com 422 `too_long` (e `field`), nunca cortadas; `GET` responde 200 com `null` nos campos não preenchidos; step-up só no `PUT` (o `GET` não traz dado sensível); evento `billing_settings.changed` só com ids.
11. **B11 — Identidade e versão da competência.** Proposta: toda linha de `GET /billing/periods` tem `id` (a competência nasce com o primeiro procedimento do estabelecimento nela) e `version` = versão do último arquivo gerado (0 antes do primeiro fechamento); `GET /billing/periods` também para o `municipal_admin`, e as pendências (`/pending`) também para ele (leitura com trilha).
12. **B12 — Quem cancela.** O §3 não diz. Proposta: o profissional cancela os **próprios** procedimentos (`attendance`, `consultation` e avulso); o `billing_clerk` cancela qualquer um (inclusive `lab_report` e `manual`); 403 `missing_role` para os demais.
13. **B13 — UBS sem registro na tela.** O ADR 0039 diz que a UBS nunca gera BPA. Proposta: o contrato registra que o dashboard **não oferece** o registro em unidade `ubs` (no atendimento, na consulta e no avulso) para não incentivar a produção da APS em dobro; o api segue aceitando e marcando `establishment_is_aps` (defesa em profundidade, e o laboratório/integrações futuras).

14. **B14 — Recepção pelas rotas do api (adotado).** O plano do api fixou `POST /attendance/verifications` e `POST /attendance/verifications/:id/names` com os 7 campos opcionais (o `/names` aceita só eles, sem nomes), resposta só com `race_color_set` e `address_set`. O dashboard usa a validação presencial e, no check-in por código, a validação do par (a encontrada ou a criada pelo check-in). Proposta que sobra: no **check-in por exceção** a tela não conhece a validação do par (o `POST /attendance/check_ins/search` não a devolve), então ali não há formulário; se o contrato quiser a captura também nesse fluxo, a resposta da exceção passa a trazer `verification_id` (nulo sem par validado) e a Task 15 repete o formulário do fluxo por código.
15. **B15 — O check-in não sabe o que falta.** As respostas do api não dizem se raça/cor ou endereço já existem, e o endereço nunca volta. Por isso o formulário do check-in aparece sempre que há validação conhecida (opcional, com "Agora não") e a tela não reexibe nada salvo. Proposta, se o contrato quiser oferecer só quando falta: o `lookup` do check-in devolve `race_color_set` e `address_set` (booleanos, sem dado).
16. **B16 — Forma e recusas (adotado, com leitura do dashboard).** Os 7 campos nas três rotas; só o enviado muda; `race_color` 01–05; `ethnicity` 4 dígitos, obrigatória com 05 e recusada nos demais (o dashboard nem a envia fora de 05); CEP 8 dígitos; larguras 30/5/10/30 com 422 `too_long` e `field` (o dashboard recusa antes e traduz o `field`); número vazio vai `SN`; complemento vazio não vai. A etnia é digitada como código (sem lista no dashboard); uma rota de busca das etnias do DATASUS fica como melhoria futura.
17. **D-L2 no dashboard.** O texto de `invalid_date` passa a "fora da competência atual e das 3 anteriores" (Tasks 2 e 5) e entram as frases de `invalid_cpf`, `citizen_not_verified`, `field_required` e `too_many_sheets` (Task 2). Os motivos novos `missing_race_color` e `missing_address` têm frase (Task 16).

Alinhamento com o plano do api: o contrato §12 adotou B1–B13 e M1–M3; B14–B16 registram como o dashboard usa as rotas da D-U1 que o plano do api fixou (e as duas propostas que sobram: `verification_id` na exceção e `race_color_set`/`address_set` no lookup); a Task 0 Step 4 compara as rotas reais com elas.

## Self-review

- **Cobertura (spec §2–§9; contrato §1–§6, §9, §10):** interruptor e papel — Tasks 1 (rótulo), 3 (menu e convite com step-up), 9 (portão); registro com crítica no atendimento e na consulta, busca SIGTAP da competência, quantidade, CID quando exigido, "não faturável — motivo" sem bloquear — Tasks 5, 6 e 7; avulso pela unidade — Task 8; produção do laboratório municipal visível com origem "laboratório municipal" (a criação é do api) — Task 9 e prova 5; lançamento manual — Task 10; competências por estabelecimento, resumo, pendências agrupadas por correção e recalculadas — Task 11; fechar e reabrir com step-up, versões e download com trilha — Task 12; configuração do admin com step-up — Task 13; tipo `procedure` na linha do tempo do profissional — Task 4; dados do paciente para o BPA-I (D-U1): regras, cliente e formulário — Task 14, recepção no check-in e na validação — Task 15, correção pelo faturista na pendência com releitura — Task 16, e o paciente sem os dados na prova 7a; apresentação em atraso — Tasks 2, 5 e prova 10; SIGTAP de outra competência — Tasks 2 e 5; prova no navegador com o api na 3044 — Task 17. A F-22.3 (laboratório) e a F-22.7 (fornecedor da linha do tempo) são do api; aqui só aparecem. O cidadão vê o procedimento no wpda pelo api (fora deste plano).
- **Placeholders:** nenhum código faltando. Dependências do outro plano estão marcadas: as contas e dados da semente, o nome do container do api (Task 17) e as formas propostas nas Divergências.
- **Tipos e nomes:** `PerformedProcedure`, `ProcedureInput`, `ProcedureQuery`, `BillingPeriod`, `PendingItem`, `BillingExport`, `BillingSettings`, `BillingPatient`, `BillingProfessional` (Task 1) são os das peças e telas; `ProcedureDraft`, `procedureBody(d, ctx)`, `procedureProblem(d, today)`, `manualProblem({ establishment, patientId, professionalId, cboOptions, cbo })`, `procedureQuery(filters)`, `EMPTY_FILTERS(today)`, `groupPending`, `exportFileName(period, exp)`, `settingsDraft/settingsProblem/settingsBody`, `reasonProblem`, `isStale`, `showUnitProcedures({ user, canCare, unit })` (Task 2) batem com os usos das Tasks 5–13; `ProcedureForm` (Task 5) recebe `before`/`beforeProblem` usados só na Task 10; `ProceduresBody.headerActions` (Task 9) é o ponto da Task 10; `BillingPeriods` da Task 11 é reescrito na Task 12 com a mesma assinatura mais `onGoToSecurity`; `timelineKinds(exam, procedure = false)` (Task 4) mantém as chamadas antigas; `DemographicsDraft`, `demographicsBody`, `demographicsProblem`, `demographicsFilled` (Task 14) são os das Tasks 15 e 16; `PatientDemographicsInput`, `DemographicsSaved`, `updatePatientDemographics`, `saveVerificationDemographics`, `VerifyExtra extends PatientDemographicsInput` (Task 14) batem com os usos; `PendingPanel` da Task 11 é reescrito na Task 16 com `canFix` opcional (a chamada da Task 12 segue válida até a Task 16 passar `canFix={clerk}`); `demographicsMissingOf` (Task 16) devolve os códigos que o `DemographicsForm` traduz.
- **Review Focus:** 1 — Task 5 ("sem CID num procedimento que exige: avisa, mas envia — quem decide é o api") e Task 6 ("lista do atendimento e registro com o id da consulta; 201 não faturável diz o motivo e relê"); 2 — Task 2 ("unitBills e showUnitProcedures: UBS nunca, UPA sim") e Task 7 ("UBS: sem o botão (a produção da APS vai pelo e-SUS)" e "UBS ou sem o interruptor: sem o painel"); 3 — Task 12 ("janela fechada pede o código; 422 settings_missing aponta a configuração e nada muda"); 4 — Task 12 ("versões: o faturista baixa com o nome do api ou o de reserva; o admin vê sem baixar"); 5 — Task 11 ("pendências: só no clique, agrupadas por correção, motivo novo cru; Conferir de novo relê").
