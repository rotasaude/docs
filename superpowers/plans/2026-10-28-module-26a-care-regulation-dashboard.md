# Módulo 26a — Regulação municipal (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 26a** (`feat/mod-26a-care-regulation`, worktree `apps/api/.claude/mod26a`, porta de dev **3048**) com as rotas do contrato §3–§8 (e as Divergências D1–D18 deste plano que o plano do api adotar), o papel `care_regulator`, o interruptor `care_regulation` na sessão, o `kind: regulated` no pedido de agendamento do 17 e a semente do 26a. Ele é **mergeado antes** deste (contrato §12: api → dashboard e wpda → maintenance → catálogo). As Tasks 0–13 não dependem dele (dublês de `src/lib/api`); a prova da Task 14 depende.

**Goal:** Lado dashboard do 26a: na **consulta (19a)**, a peça **"Pedir regulação"** (especialidade CBO ou procedimento SIGTAP com busca, CID-10, justificativa, cor de risco sugerida, "regulado pelo estado"), em rascunho, que vai para a fila na finalização ou no adendo **sem nunca bloquear a finalização**, e a **guia de encaminhamento em PDF**; o grupo **"Regulação"** com a **fila regulada** do `care_regulator` (cor, prazo da cor, tempo de espera, vencidos no topo), o **detalhe** com andamento, **autorizar escolhendo oferta com saldo** (com `color_reason` e `skip_reason` quando o api exigir), **devolver**, **negar**, **cancelar**, **agendamento externo e desfecho** (realizado/falta), **número externo** e **exportação CSV**; a tela do **solicitante** para devolvidos e complemento; **prestadores, ofertas e cotas mensais** com saldo (admin, step-up; leitura do regulador); **prazos por cor** e **lista do enfermeiro** (admin, step-up); e o rótulo do `kind: regulated` na fila da recepção do 17.

**Architecture:** O cliente HTTP ganha a seção "Regulação municipal (módulo 26a)" no fim de `src/lib/api.ts` (tipos do contrato §1–§8; na URL só ids, códigos de alvo, situação, cor, mês e cursor; CID, justificativa e motivos sempre no corpo). As regras de tela ficam puras em `src/lib/regulation.ts` (portão e papéis, quem pede pelo CBO, rótulos e tons das cores e situações, ações por situação, espera e prazo, conferência de formato — recusar, nunca cortar —, frases das recusas, chaves e constantes de consulta). As peças ficam em `src/modules/regulation/` e as telas em `src/modules/` (`RegulationQueue`, `RegulationReturned`, `RegulationOffers`, `RegulationSettings`), no grupo novo (`modules.ts`, `catalog.ts`, `App.tsx`). A consulta ganha a peça `RegulationRequestsField` dentro do `ConsultationEditor` (rascunho) e do `ConsultationView` (adendo), ligada pelo `ConsultationWorkspace` como a notificação do 25. O dashboard nunca decide o que o api garante: quem pode pedir, saldo, ordem da fila, prazos, transições, retenção LGPD e consumo de cota são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-28-module-26a-care-regulation-design.md` (§1 intenção e antiobjetivos, §2 decisões 1–12 e as do coordenador, §4 telas, §5 segurança e LGPD, §6 testes, §8 F-IDs, §9 riscos), `docs/adr/0044.md` e o contrato `docs/superpowers/plans/2026-10-28-module-26a-care-regulation-contracts.md` (§1–§8, §11, §12), com as convenções herdadas (21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12, 24a §14, 25 §15). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D18); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptor `care_regulation` (requer `clinical_record`); papel `care_regulator`; tipo `specialty` | `procedure`; cor `red` | `yellow` | `green` | `blue`; situação `draft` | `pending` | `returned` | `authorized` | `scheduled` | `completed` | `missed` | `denied` | `cancelled` | `closed` | `external`; prestador `own` | `contracted` | `consortium` | `ppi`; sistema externo `sisreg` | `care` | `cross` | `other`. Código desconhecido aparece cru, nunca some.
- Interruptor (spec decisão 12): sem `care_regulation` na sessão, o dashboard é o de hoje — o grupo "Regulação" some, a consulta não mostra a peça, o desfecho "Encaminhado" não mostra a dica, a recepção segue igual. A peça da consulta exige também `clinical_record`. 403 `feature_disabled` → `REGULATION_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo ("Regulação municipal") em `features.ts`.
- **O pedido de regulação nunca bloqueia a finalização da consulta** (spec §2, "o pedido nasce no rascunho e vale na finalização"; brief): a peça não entra em `finalizeProblems`, `missing` nem `blockedFinal`; uma falha dela (leitura ou escrita) só aparece nela; um formulário aberto e não salvo não vai junto, e a caixa de finalizar diz isso.
- Quem pede (spec, decisões do coordenador; contrato §1): `health_professional` com CBO `225*` (médico: especialidade ou procedimento), `2235*` (enfermeiro: só a lista municipal) ou `2232*` (dentista: só especialidades `2232*`, sem procedimento). A tela oferece só o tipo que o CBO pode e só os alvos que ele pode pedir (`nurse_allowed` da busca para o enfermeiro, `2232*` para o dentista); o api recusa de novo (`cbo_not_allowed`).
- **Dado do paciente só no corpo** (ADR 0044, invariantes; contrato, convenções): na URL vão só ids (`id`, `unit_id`, `consultation_id` no caminho), `status`, `type`, `target_code`, `color`, `overdue`, `month`, `before`, `limit` e, só na busca de alvo, `q`. Nome, CPF, CNS, CID, justificativa e motivos nunca na URL, em mensagem de erro, em `console` ou em Analytics; nada em `localStorage`, `sessionStorage` ou IndexedDB. A busca de alvo é `GET /regulation/targets?type=&q=&limit=` (R2 do plano do api): o campo pede o nome da especialidade ou do procedimento (rótulo "Especialidade ou procedimento", exemplo "cardiologia"), nunca a queixa, e o termo vai sem cache (`gcTime: 0`). Nome de arquivo baixado sem dado pessoal (`guia-regulacao-<id>.pdf`, `regulacao-estado-AAAA-MM-DD.csv`).
- **Nenhum polling em rota que grava trilha** (spec §5): fila, detalhe, devolvidos, pedidos da consulta e ofertas leem só ao abrir, filtrar, trocar de aba, clicar ou depois de uma ação (`NOMINAL_QUERY` do 23). Só `GET /regulation/requests/count` (sem trilha) relê a cada minuto (`COUNT_QUERY`).
- **O cidadão nunca vê posição** e **ninguém fura a fila sem motivo** (ADR 0044, invariantes): a fila mostra cor, prazo e espera, nunca um número de posição; `skip_reason` e `color_reason` vêm do api (422) e a tela só os pede e manda.
- Quem pode (contrato §3–§8): fila, detalhe, decisões, agendamento externo, desfecho, número externo e CSV só `care_regulator`; devolvidos e complemento, `health_professional` da unidade solicitante; prestadores, ofertas, prazos e lista do enfermeiro, `municipal_admin` (escrita com step-up pelo `SensitiveAction`); ofertas e prestadores também leitura do `care_regulator`. Conceder `care_regulator` é privilegiado (`isPrivilegedRole`). A tela não oferece o que o papel não pode; o api recusa de novo.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo, inclusive o `DELETE` do rascunho e o do item da lista do enfermeiro, que mandam `{}`).
- Recusar em vez de cortar: nenhum `maxLength` silencioso. Valores (os do plano do api, "Valores fixados" 2 e R19; se mudarem lá, o api vence): justificativa 10–2000 caracteres; motivos (devolver, negar, cancelar, cor, fora da ordem) 10–500; local do agendamento 1–200; observação até 500; número externo `[0-9A-Za-z./-]{1,30}`; nome do prestador 3–120; CNES 7 dígitos; quantidade inteira 0–9999 (vírgula ou ponto recusados); prazos inteiros 1–365, vermelho ≤ amarelo ≤ verde ≤ azul (D10); mês `AAAA-MM`; CID-10 `A00`/`A000`/`A00X` (o ponto digitado sai: `normalizeCid10` do 19a); termo de busca 3–60 (o api recusa procedimento com menos de 3: `invalid_query`). Tudo conferido antes de enviar e de novo no api.
- "Hoje" é o dia da cidade (`todayInCity()`); telas recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY26 = "2026-10-28"` (quarta), sem relógio falso. O mês corrente da cota é `today.slice(0, 7)`.
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`: vermelho `down`, amarelo `warn`, verde `ok`, azul `info`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `saveFile` de `src/modules/lab/saveFile.ts`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-26.1..F-26.7** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-26.1: api, dashboard e maintenance; F-26.2..F-26.6: api e dashboard; F-26.7: api e wpda) e a **Verified** depois da prova no navegador (Task 14).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod26a` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-26a-care-regulation` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod26a`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test dashboard -w dashboard-mod26a <arquivos>` (o runner único; saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx` (sessão com `mfa_verified_at` de agora = janela de step-up aberta: o `SensitiveAction` não pede código).
- A prova (Task 14) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5203**, com proxy para o api do 26a na **3048**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Médico finaliza com o formulário de regulação meio preenchido, ou a leitura dos pedidos falha.** Espera-se: "Confirmar finalização" segue habilitado e finaliza; a caixa de finalizar diz quantos rascunhos vão e que o formulário não salvo não vai junto; a falha aparece só na peça. Testes: Task 5, "formulário aberto ou leitura que falha: a finalização segue igual" e "ao finalizar: a conta dos rascunhos e o aviso do não salvo, dentro da caixa".
2. **Dois reguladores autorizam a última vaga.** Espera-se: 409 `quota_exhausted` vira "a cota desta oferta acabou…", as ofertas são relidas (a esgotada aparece desabilitada), a escolha de oferta é limpa e cor e motivos digitados ficam; nada é reenviado sozinho. Teste: Task 8, "409 quota_exhausted: frase, ofertas relidas, oferta limpa, o resto fica".
3. **Autorizar fora da ordem ou mudar a cor.** Espera-se: mudar a cor sem motivo é recusado antes de enviar; 422 `skip_reason_required` mostra a explicação e o campo do motivo, com a oferta e a cor mantidas, e o reenvio leva `skip_reason`. Teste: Task 8, "mudar a cor pede motivo antes; 422 skip_reason_required pede o motivo e reenvia com ele".
4. **Fila aberta o turno todo.** Espera-se: a lista nominal não relê ao voltar à aba, ao reconectar nem com o tempo; só a contagem relê a cada minuto e o menu mostra os vencidos. Testes: Task 7, "volta à aba e reconecta: a lista não relê"; Task 3, "a contagem relê a cada minuto e acende o item com os vencidos".
5. **CID, justificativa ou motivo vazando.** Espera-se: nenhuma URL chamada leva CID, justificativa, motivo ou nome; arquivos baixados com nome sem dado pessoal; nada no storage. Testes: Task 1, "nada sensível na URL: CID, justificativa e motivos só no corpo"; Task 13, "CSV com nome sem dado pessoal; nada no storage".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.regulation.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/regulationFixtures.ts` | tipos e cliente HTTP do 26a; interruptor; proxy; dados de teste | 1 |
| `src/lib/regulation.ts`, `src/lib/regulation.test.ts` | regras de tela: portão, CBO, rótulos, ações, espera e prazo, formato, frases | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.regulation.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/hooks/useRegulationCount.ts`, `src/hooks/useRegulationCount.test.tsx`, `src/modules/regulation/RegulationGate.tsx`, quatro telas-portão, `src/lib/team.ts`, `src/lib/team.test.ts` | grupo "Regulação", contador no menu, portão, papel na Equipe | 3 |
| `src/modules/regulation/TargetSearch.tsx`, `src/modules/regulation/RequestForm.tsx`, `src/modules/regulation/RequestForm.test.tsx` | busca de alvo e formulário do pedido (reaproveitados pela consulta e pelo complemento) | 4 |
| `src/modules/consultation/RegulationRequestsField.tsx`, `src/modules/consultation/RegulationRequestsField.test.tsx`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationEditor.test.tsx`, `src/modules/attendance/OutcomeFields.tsx` | "Pedir regulação" no rascunho, sem bloquear (F-26.2) | 5 |
| `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.regulation.test.tsx`, `src/modules/regulation/openGuide.ts` | pedido por adendo, situação dos pedidos, guia em PDF, ligação no atendimento (F-26.2) | 6 |
| `src/modules/RegulationQueue.tsx`, `src/modules/regulation/RequestTable.tsx`, `src/modules/RegulationQueue.test.tsx` | fila com abas, filtros, cor, prazo, espera (F-26.3) | 7 |
| `src/modules/regulation/RequestDetail.tsx`, `src/modules/regulation/AuthorizeForm.tsx`, `src/modules/regulation/ReasonAction.tsx`, `src/modules/regulation/RequestDetail.test.tsx` | detalhe, andamento, autorizar, devolver, negar, cancelar (F-26.3) | 8 |
| `src/modules/RegulationReturned.tsx`, `src/modules/RegulationReturned.test.tsx` | devolvidos da unidade e complemento (F-26.3) | 9 |
| `src/modules/RegulationOffers.tsx`, `src/modules/regulation/ProviderForm.tsx`, `src/modules/regulation/OfferForm.tsx`, `src/modules/RegulationOffers.test.tsx` | prestadores, ofertas e cotas com saldo (F-26.4) | 10 |
| `src/modules/RegulationSettings.tsx`, `src/modules/RegulationSettings.test.tsx` | prazos por cor e lista do enfermeiro (F-26.1) | 11 |
| `src/modules/regulation/ScheduleOutcome.tsx`, `src/modules/regulation/ScheduleOutcome.test.tsx`, `src/modules/regulation/RequestDetail.tsx`, `src/lib/api.ts`, `src/lib/scheduling.ts`, `src/lib/scheduling.test.ts` | agendamento externo, desfecho e o `regulated` da recepção (F-26.5) | 12 |
| `src/modules/regulation/ExternalNumberForm.tsx`, `src/modules/regulation/ExternalNumberForm.test.tsx`, `src/modules/regulation/RequestDetail.tsx`, `src/modules/RegulationQueue.tsx`, `src/modules/RegulationQueue.test.tsx` | número externo e exportação CSV (F-26.6) | 13 |
| — | suíte, build, revisão, prova no navegador | 14 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CID, justificativa, motivos) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa —, quem pede pelo CBO, ações por situação, espera e prazo (Task 2); menu por papel e interruptor e o contador (Task 3); cada tela e peça pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api e sessão por papel e interruptor, cobrindo o caminho feliz, cada recusa que muda a tela (`cbo_not_allowed`, `invalid_target`, `invalid_cid10`, `invalid_request` com `path`, `quota_exhausted`, `skip_reason_required`, `color_reason_required`, `stale_version`, `invalid_transition`, `quantity_below_used`, `duplicate_offer`, `duplicate_item`, `appointment_scheduled`, `citizen_not_verified`, `regulation_request_required`, `feature_disabled`), o step-up, a ausência de releitura, a ausência de armazenamento local e a finalização que nunca trava (Tasks 4–13). A prova final é no navegador, contra o api real na 3048 (Task 14).

---

### Task 0: Board, worktree, base, âncoras e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`956f57f` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod26a`.
- Produces: cards F-26.1..F-26.7 em In Progress; worktree `$W` na branch `feat/mod-26a-care-regulation`; a base anotada; as rotas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-26\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api, do wpda e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os sete cards em In Progress ou além. Se algum F-26.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "procedure_billing" | "care_lines" | "territory_registry" | "surveillance";\|  surveillance: "Vigilância epidemiológica"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const consultationPath\|^async function binaryFetch\|^export interface SignatureFile\|^export interface RequestRow {\|  id: string; kind: "return" | "referral" | "triage" | "screening" | "care_line"; origin: "attendance" | "triage" | "care_line";\|  id: string; scheduled_at: string; kind: "return" | "referral" | "triage" | "screening" | "care_line"; unit_name: string;\|^export async function getMyProfessional\|^export async function listAllUnits'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "surveillance-queue" | "surveillance-negatives" | "surveillance-reports" | "surveillance-panel" | "surveillance-diseases";\|      if (item.id.startsWith("surveillance-")) return survOn && isSurveillance;\|^export function withSurveillanceCount'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 13\|NAV_GROUPS.length - 12'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '  IconVirus, IconBellRinging, IconCalendarX, IconFileSpreadsheet, IconGridDots, IconVirusSearch\|    "surveillance-diseases": { icon: IconVirusSearch'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '^export function shellGroups(\|    case "surveillance-diseases": return <SurveillanceDiseases\|  const survQuery = useSurveillanceCount('
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/surveillance(/|$)": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '  { role: "surveillance", label: "Vigilância epidemiológica" },\|^const PRIVILEGED_ROLES'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationEditor.tsx | grep -n '^  notifications?: boolean;\|          noteTarget={noteHost} />\|          {props.notifications && <div ref={setNoteHost}'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n '                notifications={canSuggestNotification(user) && cboCanNotify(consultation.cbo_code)}\|              <ConsultationView consultation={consultation} options={options.data ?? null} patientProblems={record.data.problems}'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/OutcomeFields.tsx | grep -n '^interface Props {\|      {value.outcome === "referred" && ('
$G -C $R/apps/dashboard show origin/main:src/lib/scheduling.ts | grep -n '^export function requestKindLabel\|  if (row.kind === "care_line") return "Linha de cuidado";'
$G -C $R/apps/dashboard show origin/main:src/lib/consultation.ts | grep -n '^export function normalizeCid10\|^export function cid10Problem'
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^export const NOMINAL_QUERY\|^export const COUNT_QUERY\|^export const STATIC_QUERY'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod26a -b feat/mod-26a-care-regulation origin/main
cd $R/.worktrees/dashboard-mod26a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod26a
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(ArrowsSplit2|ListNumbers|ArrowBackUp|BuildingStore|ClockHour4)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os cinco ícones (falta algum → escolha outro que exista e use o mesmo na Task 3; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 14 espera a base mais **14 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 26a** (informativo para as Tasks 1–13)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-26a-care-regulation
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod26a api bin/rails routes 2>/dev/null | grep -E " /regulation/|regulation_requests" | head -60
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-28-module-26a-care-regulation-api.md 2>/dev/null | grep -n "^## \|^### Task\|^- \*\*R[0-9]*\|Valores fixados" | head -60
```

Expected: a branch e as rotas (compare com o contrato §3–§8, com o plano do api — R1–R19 — e com as Divergências D1–D18: diferença de forma vira ajuste em `src/lib/api.ts` e `src/test/regulationFixtures.ts` antes da prova, com nota para a Task 14). Os limites de texto e de quantidade do api, se diferentes dos de "Global Constraints", **vencem**: ajuste `src/lib/regulation.ts` e o teste da Task 2. Anote como ficaram as abertas D3, D4, D8, D17 e D18.

---

### Task 1: Tipos, cliente HTTP, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/regulationFixtures.ts`
- Test: `src/lib/api.regulation.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `binaryFetch`, `consultationPath`, `SignatureFile` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`):
  - tipos `RegulationType = "specialty" | "procedure"`, `RiskColor = "red" | "yellow" | "green" | "blue"`, `RegulationStatus` (os 11 do §1), `ProviderKind`, `ExternalSystem`; `RegulationTarget { type: RegulationType; code: string; name: string; group?: string | null; nurse_allowed?: boolean }`; `RegulationEvent { status: RegulationStatus | string; at: string; by: { name: string } | null; reason: string | null }`; `RegulationPatient { id; display_name: string | null; has_cpf: boolean; cns_masked: string | null }`; `RegulationRequest` (a `<regulation_request>` do §2, com `events?`; `requested_at` e `due_on` nulos no rascunho); `RegulationRequestInput { type; target_code; cid10?: string | null; justification; suggested_color: RiskColor; state_regulated?: boolean }`; `RegulationQuery { status?: string | null; overdue?: boolean; type?: RegulationType | null; targetCode?: string | null; color?: RiskColor | null; before?: string | null }`; `RegulationPage { requests: RegulationRequest[]; next_before: string | null }`; `RegulationCounts { pending; overdue; returned; authorized; external }`; `AuthorizeInput { offer_id; color?; color_reason?; skip_reason?; version }`; `ComplementInput { justification; cid10?: string | null; suggested_color?: RiskColor; version }`; `ScheduleInput { at; place; notes?: string | null; version }`; `CareProvider { id; name; cnes; kind: ProviderKind | string; health_unit: { id; name } | null; active: boolean }`; `CareProviderInput { name; cnes?: string; kind: ProviderKind; health_unit_id?: string | null; active: boolean }`; `CareOffer` (a `<offer>` do §2); `CareOfferInput { provider_id; target_type: RegulationType; target_code; month; quantity: number }`; `RiskDeadlines { red; yellow; green; blue }` (dias); `NurseItem { target_type: RegulationType; target_code; name }`.
  - constante `REGULATION_PAGE_SIZE = 50`.
  - funções `searchRegulationTargets(q: string, type: RegulationType): Promise<RegulationTarget[]>`, `listConsultationRegulationRequests(consultationId: string): Promise<RegulationRequest[]>`, `createRegulationRequest(consultationId: string, input: RegulationRequestInput): Promise<RegulationRequest>`, `updateRegulationRequest(consultationId: string, id: string, input: RegulationRequestInput): Promise<RegulationRequest>`, `deleteRegulationRequest(consultationId: string, id: string): Promise<void>`, `fetchRegulationGuide(id: string): Promise<SignatureFile>`, `listRegulationRequests(q: RegulationQuery): Promise<RegulationPage>`, `getRegulationRequest(id: string): Promise<RegulationRequest>`, `countRegulationRequests(): Promise<RegulationCounts>`, `authorizeRegulationRequest(id: string, input: AuthorizeInput): Promise<RegulationRequest>`, `returnRegulationRequest(id: string, reason: string, version: number)`, `denyRegulationRequest(…)`, `cancelRegulationRequest(…)` (as três `Promise<RegulationRequest>`), `listReturnedRequests(unitId: string): Promise<RegulationRequest[]>`, `complementRegulationRequest(id: string, input: ComplementInput): Promise<RegulationRequest>`, `scheduleRegulationRequest(id: string, input: ScheduleInput): Promise<RegulationRequest>`, `recordRegulationOutcome(id: string, result: "completed" | "missed", version: number): Promise<RegulationRequest>`, `setRegulationExternal(id: string, system: ExternalSystem, number: string, version: number): Promise<RegulationRequest>`, `fetchStateExportCsv(status?: string | null): Promise<SignatureFile>`, `listCareProviders(): Promise<CareProvider[]>`, `createCareProvider(input: CareProviderInput): Promise<CareProvider>`, `updateCareProvider(id: string, input: CareProviderInput): Promise<CareProvider>`, `listCareOffers(q: { month?: string | null; targetCode?: string | null }): Promise<CareOffer[]>`, `createCareOffer(input: CareOfferInput): Promise<CareOffer>`, `updateCareOffer(id: string, quantity: number): Promise<CareOffer>`, `getRiskDeadlines(): Promise<RiskDeadlines>`, `updateRiskDeadlines(d: RiskDeadlines): Promise<RiskDeadlines>`, `listNurseItems(): Promise<NurseItem[]>`, `addNurseItem(input: { target_type: RegulationType; target_code: string }): Promise<NurseItem>`, `removeNurseItem(item: { target_type: RegulationType; target_code: string }): Promise<void>`.
  - `FeatureKey` ganha `care_regulation`.
- Produces (em `src/test/regulationFixtures.ts`): `TODAY26`, `UBS_CENTRO`, `AMB_ESPEC`, `CARDIO`, `ECO`, `ENDO`, `regUser(roles?, over?)`, `regRequest(over?)`, `regCounts(over?)`, `provider(over?)`, `offer(over?)`, `deadlines(over?)`, `nurseItem(over?)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.regulation.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  addNurseItem, authorizeRegulationRequest, cancelRegulationRequest, complementRegulationRequest, countRegulationRequests, createCareOffer,
  createCareProvider, createRegulationRequest, deleteRegulationRequest, denyRegulationRequest, fetchRegulationGuide, fetchStateExportCsv,
  getRegulationRequest, getRiskDeadlines, listCareOffers, listCareProviders, listConsultationRegulationRequests, listNurseItems,
  listRegulationRequests, listReturnedRequests, recordRegulationOutcome, removeNurseItem, returnRegulationRequest, scheduleRegulationRequest,
  searchRegulationTargets, setRegulationExternal, updateCareOffer, updateRegulationRequest, updateRiskDeadlines
} from "./api";

function ok(body: unknown, headers: Record<string, string> = { "Content-Type": "application/json" }): Promise<Response> {
  return Promise.resolve(new Response(typeof body === "string" ? body : JSON.stringify(body), { status: 200, headers }));
}
// CID-10, trechos da justificativa e dos motivos usados abaixo; nada disso pode aparecer em URL.
const SENSITIVE = /I20|I209|dor|tor[aá]cic|esfor|motivo|falta de exame|duplicad|cardio|justif|cid/i;

describe("cliente da regulação (contrato §3–§8)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(() => ok({})); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => (fetchMock.mock.calls[i][1] ?? {}) as RequestInit;
  const headersOf = (i = 0) => initOf(i).headers as Record<string, string>;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));
  const INPUT = { type: "specialty" as const, target_code: "225120", cid10: "I209", suggested_color: "yellow" as const,
    justification: "Dor torácica aos esforços há 3 meses, teste ergométrico alterado." };

  it("busca de alvo (R2): GET com tipo, termo e limite; envelope; ausente vira lista vazia", async () => {
    const CARDIO_T = { type: "specialty", code: "225120", name: "Cardiologia", group: null, nurse_allowed: false };
    fetchMock.mockImplementationOnce(() => ok({ targets: [ CARDIO_T ] })).mockImplementationOnce(() => ok({}));
    expect(await searchRegulationTargets("cardio", "specialty")).toEqual([ CARDIO_T ]);
    expect(urlOf(0).pathname).toBe("/regulation/targets");
    expect(Object.fromEntries(urlOf(0).searchParams)).toEqual({ type: "specialty", q: "cardio", limit: "20" });
    expect(await searchRegulationTargets("eco", "procedure")).toEqual([]);
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ type: "procedure", q: "eco", limit: "20" });
  });

  it("pedidos da consulta: listar, criar, editar e retirar o rascunho, com JSON", async () => {
    fetchMock.mockImplementationOnce(() => ok({ requests: [ { id: "rr1" } ] })).mockImplementationOnce(() => ok({ id: "rr1" }))
      .mockImplementationOnce(() => ok({ id: "rr1" })).mockImplementationOnce(() => Promise.resolve(new Response(null, { status: 204 })));
    expect(await listConsultationRegulationRequests("cs1")).toEqual([ { id: "rr1" } ]);
    expect(urlOf(0).pathname).toBe("/attendance/consultations/cs1/regulation_requests");
    await createRegulationRequest("cs1", INPUT);
    expect([ urlOf(1).pathname, initOf(1).method ]).toEqual([ "/attendance/consultations/cs1/regulation_requests", "POST" ]);
    expect(headersOf(1)["Content-Type"]).toBe("application/json");
    expect(bodyOf(1)).toEqual(INPUT);
    await updateRegulationRequest("cs1", "rr1", { ...INPUT, suggested_color: "red" });
    expect([ urlOf(2).pathname, initOf(2).method, bodyOf(2).suggested_color ]).toEqual([ "/attendance/consultations/cs1/regulation_requests/rr1", "PATCH", "red" ]);
    await deleteRegulationRequest("cs1", "rr1");
    expect([ urlOf(3).pathname, initOf(3).method ]).toEqual([ "/attendance/consultations/cs1/regulation_requests/rr1", "DELETE" ]);
    expect(headersOf(3)["Content-Type"]).toBe("application/json");
    expect(bodyOf(3)).toEqual({});
  });

  it("fila: só os filtros que vieram, com limite; página e contagem com padrões", async () => {
    fetchMock.mockImplementationOnce(() => ok({ requests: [ { id: "rr1" } ], next_before: "c1" })).mockImplementationOnce(() => ok({}))
      .mockImplementationOnce(() => ok({ pending: 4 }));
    expect(await listRegulationRequests({ status: "pending", type: "specialty", targetCode: "225120", color: "red", before: "c0" }))
      .toEqual({ requests: [ { id: "rr1" } ], next_before: "c1" });
    expect(urlOf(0).pathname).toBe("/regulation/requests");
    expect(Object.fromEntries(urlOf(0).searchParams)).toEqual({ status: "pending", type: "specialty", target_code: "225120", color: "red", before: "c0", limit: "50" });
    expect(await listRegulationRequests({ overdue: true, status: null, type: null, targetCode: null, color: null }))
      .toEqual({ requests: [], next_before: null });
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ overdue: "true", limit: "50" });
    expect(await countRegulationRequests()).toEqual({ pending: 4, overdue: 0, returned: 0, authorized: 0, external: 0 });
    expect(urlOf(2).pathname).toBe("/regulation/requests/count");
    await listRegulationRequests({ status: "pending", overdue: false });
    expect(Object.fromEntries(urlOf(3).searchParams)).toEqual({ status: "pending", overdue: "false", limit: "50" });
  });

  it("decisões: versão e motivos só no corpo", async () => {
    await getRegulationRequest("rr1");
    await authorizeRegulationRequest("rr1", { offer_id: "of1", color: "red", color_reason: "motivo da cor", skip_reason: "falta de exame", version: 3 });
    await returnRegulationRequest("rr1", "motivo de devolver", 3);
    await denyRegulationRequest("rr1", "motivo de negar", 3);
    await cancelRegulationRequest("rr1", "duplicado", 3);
    expect(urlOf(0).pathname).toBe("/regulation/requests/rr1");
    expect([ urlOf(1).pathname, bodyOf(1) ]).toEqual([ "/regulation/requests/rr1/authorize",
      { offer_id: "of1", color: "red", color_reason: "motivo da cor", skip_reason: "falta de exame", version: 3 } ]);
    expect([ urlOf(2).pathname, bodyOf(2) ]).toEqual([ "/regulation/requests/rr1/return", { reason: "motivo de devolver", version: 3 } ]);
    expect([ urlOf(3).pathname, bodyOf(3) ]).toEqual([ "/regulation/requests/rr1/deny", { reason: "motivo de negar", version: 3 } ]);
    expect([ urlOf(4).pathname, bodyOf(4) ]).toEqual([ "/regulation/requests/rr1/cancel", { reason: "duplicado", version: 3 } ]);
    for (let i = 1; i <= 4; i++) expect([ initOf(i).method, headersOf(i)["Content-Type"] ]).toEqual([ "POST", "application/json" ]);
  });

  it("solicitante, agendamento, desfecho e número externo", async () => {
    fetchMock.mockImplementationOnce(() => ok({ requests: [ { id: "rr1" } ] }));
    expect(await listReturnedRequests("u1")).toEqual([ { id: "rr1" } ]);
    expect([ urlOf(0).pathname, urlOf(0).search ]).toEqual([ "/regulation/returned", "?unit_id=u1" ]);
    await complementRegulationRequest("rr1", { justification: INPUT.justification, cid10: "I209", suggested_color: "yellow", version: 4 });
    expect([ urlOf(1).pathname, bodyOf(1) ]).toEqual([ "/regulation/requests/rr1/complement",
      { justification: INPUT.justification, cid10: "I209", suggested_color: "yellow", version: 4 } ]);
    await scheduleRegulationRequest("rr1", { at: "2026-11-10T14:30:00-03:00", place: "Clínica do Coração, Rua XV, 100", notes: null, version: 5 });
    expect([ urlOf(2).pathname, bodyOf(2) ]).toEqual([ "/regulation/requests/rr1/schedule",
      { at: "2026-11-10T14:30:00-03:00", place: "Clínica do Coração, Rua XV, 100", notes: null, version: 5 } ]);
    await recordRegulationOutcome("rr1", "missed", 6);
    expect([ urlOf(3).pathname, bodyOf(3) ]).toEqual([ "/regulation/requests/rr1/outcome", { result: "missed", version: 6 } ]);
    await setRegulationExternal("rr1", "sisreg", "2026/123456", 7);
    expect([ urlOf(4).pathname, bodyOf(4) ]).toEqual([ "/regulation/requests/rr1/external", { system: "sisreg", number: "2026/123456", version: 7 } ]);
  });

  it("guia e CSV lidos com fetch; nome do arquivo seguro ou nulo", async () => {
    fetchMock.mockImplementationOnce(() => ok("%PDF", { "Content-Type": "application/pdf", "Content-Disposition": 'attachment; filename="guia-regulacao-rr1.pdf"' }))
      .mockImplementationOnce(() => ok("a;b", { "Content-Type": "text/csv", "Content-Disposition": 'attachment; filename="Joana Lima.csv"' }));
    expect((await fetchRegulationGuide("rr1")).filename).toBe("guia-regulacao-rr1.pdf");
    expect(urlOf(0).pathname).toBe("/regulation/requests/rr1/guide.pdf");
    expect((await fetchStateExportCsv("pending")).filename).toBeNull();
    expect([ urlOf(1).pathname, urlOf(1).search ]).toEqual([ "/regulation/exports/state.csv", "?status=pending" ]);
  });

  it("administração: prestadores, ofertas, prazos e lista do enfermeiro", async () => {
    fetchMock.mockImplementationOnce(() => ok({ providers: [ { id: "pr1" } ] })).mockImplementationOnce(() => ok({ id: "pr1" }))
      .mockImplementationOnce(() => ok({ offers: [ { id: "of1" } ] })).mockImplementationOnce(() => ok({ id: "of1" }))
      .mockImplementationOnce(() => ok({ id: "of1" })).mockImplementationOnce(() => ok({ red: 7, yellow: 30, green: 90, blue: 180 }))
      .mockImplementationOnce(() => ok({ red: 5, yellow: 30, green: 90, blue: 180 })).mockImplementationOnce(() => ok({ items: [] }))
      .mockImplementationOnce(() => ok({ target_type: "procedure", target_code: "0205010032", name: "Ecocardiografia transtorácica" }))
      .mockImplementationOnce(() => Promise.resolve(new Response(null, { status: 204 })));
    expect(await listCareProviders()).toEqual([ { id: "pr1" } ]);
    await createCareProvider({ name: "UBS Especialidades", cnes: "2384310", kind: "own", health_unit_id: "u2", active: true });
    expect([ urlOf(1).pathname, initOf(1).method, bodyOf(1).kind ]).toEqual([ "/regulation/admin/providers", "POST", "own" ]);
    expect(await listCareOffers({ month: "2026-10", targetCode: "225120" })).toEqual([ { id: "of1" } ]);
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ month: "2026-10", target_code: "225120" });
    await createCareOffer({ provider_id: "pr1", target_type: "specialty", target_code: "225120", month: "2026-11", quantity: 10 });
    expect([ urlOf(3).pathname, bodyOf(3).quantity ]).toEqual([ "/regulation/admin/offers", 10 ]);
    await updateCareOffer("of1", 12);
    expect([ urlOf(4).pathname, initOf(4).method, bodyOf(4) ]).toEqual([ "/regulation/admin/offers/of1", "PATCH", { quantity: 12 } ]);
    expect(await getRiskDeadlines()).toEqual({ red: 7, yellow: 30, green: 90, blue: 180 });
    await updateRiskDeadlines({ red: 5, yellow: 30, green: 90, blue: 180 });
    expect([ urlOf(6).pathname, initOf(6).method, bodyOf(6) ]).toEqual([ "/regulation/admin/deadlines", "PATCH", { red: 5, yellow: 30, green: 90, blue: 180 } ]);
    expect(await listNurseItems()).toEqual([]);
    await addNurseItem({ target_type: "procedure", target_code: "0205010032" });
    expect([ urlOf(8).pathname, bodyOf(8) ]).toEqual([ "/regulation/admin/nurse_items", { target_type: "procedure", target_code: "0205010032" } ]);
    await removeNurseItem({ target_type: "procedure", target_code: "0205010032" });
    expect([ urlOf(9).pathname, initOf(9).method, bodyOf(9) ]).toEqual([ "/regulation/admin/nurse_items", "DELETE",
      { target_type: "procedure", target_code: "0205010032" } ]);
  });

  it("nada sensível na URL: CID, justificativa e motivos só no corpo", async () => {
    await createRegulationRequest("cs1", INPUT);
    await authorizeRegulationRequest("rr1", { offer_id: "of1", color_reason: "motivo da cor", skip_reason: "falta de exame", version: 3 });
    await returnRegulationRequest("rr1", "motivo de devolver", 3);
    await complementRegulationRequest("rr1", { justification: INPUT.justification, cid10: "I209", version: 4 });
    for (const call of fetchMock.mock.calls) expect(decodeURIComponent(String(call[0]))).not.toMatch(SENSITIVE);
  });
});
```

E no fim de `src/lib/features.test.ts`:

```ts
describe("módulo 26a", () => {
  it("care_regulation tem nome em português", () => {
    expect(featureLabel("care_regulation")).toBe("Regulação municipal");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/lib/api.regulation.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções não existem e o rótulo é a chave crua.

- [ ] **Step 3: Implemente a seção "Regulação municipal (módulo 26a)" no fim de `src/lib/api.ts`**

No molde da seção do 25: `REGULATION_BASE = import.meta.env.VITE_REGULATION_BASE || "/regulation"`, `regPath(...parts)` com `encodeURIComponent` em cada parte, `regWrite(method, body)` (`POST`/`PATCH`/`DELETE`, sempre com corpo — `{}` no `DELETE` do rascunho e `{ target_type, target_code }` no do item da lista do enfermeiro, para o `jsonFetch` pôr o `Content-Type`); a busca de alvo é `GET` por `withQuery(regPath("targets"), { type, q, limit: "20" })`; os pedidos da consulta pelo `consultationPath(id, "regulation_requests")` (com `/<rid>` no PATCH/DELETE). Envelopes: `targets`, `requests`, `providers`, `offers`, `items` (ausente → `[]`); página com `next_before ?? null`; contagem com `?? 0` em cada campo; `204` do `DELETE` vira `void` (como no 24a). `regulationParams(q)` põe, nesta ordem e só se vierem, `status`, `overdue=true`/`overdue=false` (só quando é booleano), `type`, `target_code`, `color`, `before`, e sempre `limit=50`. Guia e CSV pelo `binaryFetch` (`application/pdf`, `text/csv`). Os tipos da lista de Interfaces, com `| string` nos códigos que vêm do api (cor, situação, tipo, sistema). Em `features.ts`, `care_regulation` no `FeatureKey` e no `FEATURE_LABEL` (`// Módulo 26a (ADR 0044): exige clinical_record.` + `care_regulation: "Regulação municipal"`). Em `vite.config.ts`, depois do proxy de `/surveillance`: `"^/regulation(/|$)": proxy(TARGET)`.

Crie `src/test/regulationFixtures.ts` (hoje: quarta, 2026-10-28; dados ilustrativos — a lista real de alvos vem do api):

```ts
import type { CareOffer, CareProvider, NurseItem, RegulationCounts, RegulationRequest, RegulationTarget, RiskDeadlines, SessionUser } from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY26 = "2026-10-28";
export const UBS_CENTRO = { id: "u1", name: "UBS Centro" };
export const AMB_ESPEC = { id: "u2", name: "Ambulatório de Especialidades" };
export const CARDIO: RegulationTarget = { type: "specialty", code: "225120", name: "Cardiologia", group: null, nurse_allowed: false };
export const ECO: RegulationTarget = { type: "procedure", code: "0205010032", name: "Ecocardiografia transtorácica", group: null, nurse_allowed: true };
export const ENDO: RegulationTarget = { type: "specialty", code: "223212", name: "Endodontia", group: null, nurse_allowed: false };

export function regUser(roles: string[] = [ "care_regulator" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { features: [ "care_regulation", "clinical_record" ], ...over });
}
export function regRequest(over: Partial<RegulationRequest> = {}): RegulationRequest {
  return {
    id: "rr1", patient: { id: "pa1", display_name: "Joana Lima", has_cpf: true, cns_masked: null }, type: "specialty",
    target: { code: "225120", name: "Cardiologia" }, cid10: "I209",
    justification: "Dor torácica aos esforços há 3 meses, teste ergométrico alterado.", suggested_color: "yellow", color: "yellow",
    color_reason: null, status: "pending", requested_at: "2026-10-08T10:00:00-03:00", due_on: "2026-11-07", overdue: false,
    requester: { name: "Dra. Helena Prado", cbo: "225142", unit: UBS_CENTRO }, consultation_id: "cs1", authorization: null,
    appointment_request_id: null, schedule: null, outcome: null, misses: 0, external: null, state_regulated: false, version: 3,
    events: [ { status: "pending", at: "2026-10-08T10:00:00-03:00", by: { name: "Dra. Helena Prado" }, reason: null } ], ...over
  };
}
export function regCounts(over: Partial<RegulationCounts> = {}): RegulationCounts {
  return { pending: 12, overdue: 3, returned: 2, authorized: 5, external: 1, ...over };
}
export function provider(over: Partial<CareProvider> = {}): CareProvider {
  return { id: "pr1", name: "Ambulatório de Especialidades", cnes: "2384310", kind: "own", health_unit: AMB_ESPEC, active: true, ...over };
}
export function offer(over: Partial<CareOffer> = {}): CareOffer {
  return { id: "of1", provider: { id: "pr1", name: "Ambulatório de Especialidades", kind: "own" }, target: CARDIO, month: "2026-10",
    quantity: 10, used: 8, available: 2, ...over };
}
export function deadlines(over: Partial<RiskDeadlines> = {}): RiskDeadlines { return { red: 7, yellow: 30, green: 90, blue: 180, ...over }; }
export function nurseItem(over: Partial<NurseItem> = {}): NurseItem {
  return { target_type: "procedure", target_code: "0205010032", name: "Ecocardiografia transtorácica", ...over };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/lib/api.regulation.test.ts src/lib/features.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.regulation.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/regulationFixtures.ts
$G commit -m "feat: add the care regulation api client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/regulation.ts`)

**Files:**
- Create: `src/lib/regulation.ts`
- Modify: `src/lib/attendance.ts` (a frase de `regulation_request_required` no "Encerrar" da fila)
- Test: `src/lib/regulation.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `errorCode`, tipos da Task 1; `hasFeature`, `featureDisabledKey` (`features.ts`); `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY` (`careLines.ts`); `normalizeCid10`, `cid10Problem` (`consultation.ts`); `cityIsoDate`, `cityLocalIso` (`format.ts`).
- Produces:
  - frases `REGULATION_DISABLED = "a regulação municipal está desligada nesta cidade"`, `REGULATION_NO_ROLE = "seu papel não permite usar a regulação municipal"`, `STALE_TEXT = "outra pessoa alterou este pedido enquanto você decidia — recarregue para ver a versão nova"`, `QUOTA_EXHAUSTED_TEXT = "a cota desta oferta acabou enquanto você decidia — escolha outra oferta"`, `SKIP_REASON_TEXT = "Há pedido mais antigo, de cor igual ou mais grave, esperando nesta especialidade ou procedimento com oferta disponível. Para autorizar este antes, registre o motivo."`, `REFERRAL_HINT = "Encaminhamento a especialista ou procedimento é pelo “Pedir regulação” na consulta; este desfecho é só para outra unidade da rede."`, `UNSAVED_REQUEST_NOTE = "O pedido de regulação que você está escrevendo ainda não foi salvo e não vai junto."`, `ADDENDUM_NOTE = "Pedidos feitos aqui ficam em rascunho e vão para a regulação quando você registrar um adendo."`;
  - chaves `REG_LIST_KEY`, `REG_DETAIL_KEY`, `REG_COUNT_KEY`, `REG_CONSULTATION_KEY`, `REG_TARGETS_KEY`, `REG_OFFERS_KEY`, `REG_PROVIDERS_KEY`, `REG_DEADLINES_KEY`, `REG_NURSE_KEY`, `REG_RETURNED_KEY` (strings `"regulation…"`); re-export de `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`;
  - `canUseRegulation(user): boolean`, `regulationRoles(user): RegulationRoles { regulator; admin; professional }`, `canRequestRegulation(user): boolean`, `requesterKind(cbo): "physician" | "nurse" | "dentist" | null`, `allowedTypes(cbo): RegulationType[]`, `targetsFor(cbo: string | null | undefined, targets: RegulationTarget[]): RegulationTarget[]` (enfermeiro: só `nurse_allowed`; dentista: só especialidade `2232*`; médico ou sem CBO — regulador e admin —: todos);
  - `COLORS: { key: RiskColor; label: string; tone: string }[]`, `colorLabel(code)`, `colorTone(code)`, `statusLabel(code)`, `statusTone(code)`, `typeLabel(code)`, `providerKindLabel(code)`, `EXTERNAL_SYSTEMS: { key: ExternalSystem; label: string }[]`, `systemLabel(code)`;
  - `actionsFor(r: RegulationRequest): RegulationActions { authorize; returnBack; deny; cancel; schedule; outcome; external }`;
  - `waitDays(requestedAt: string, today: string): number`, `waitText(days: number): string`, `dueText(dueOn: string | null, overdue: boolean, today: string): string`, `currentMonth(today: string): string`, `monthLabel(month: string): string`;
  - `RequestDraft { type: RegulationType; target: RegulationTarget | null; cid10: string; justification: string; color: RiskColor | ""; stateRegulated: boolean }`, `EMPTY_DRAFT(type)`, `draftFromRequest(r)`, `requestProblem(d: RequestDraft, cid10Allowed: boolean): { field: "target" | "cid10" | "justification" | "color"; text: string } | null`, `requestBody(d: RequestDraft, cid10Allowed: boolean): RegulationRequestInput`;
  - `justificationProblem`, `reasonProblem`, `placeProblem`, `notesProblem`, `externalNumberProblem`, `cnesProblem`, `providerNameProblem`, `quantityProblem(raw: string): string | null` com `parseQuantity(raw): number`, `monthProblem`, `targetTermProblem`, `scheduleAtProblem(local: string): string | null`, `deadlinesProblem(raw: Record<RiskColor, string>): string | null`;
  - `regulationError(err: unknown): string`, `REGULATION_REQUIRED_TEXT = "encaminhamento sem unidade de destino exige um pedido de regulação: escolha a unidade da rede ou use “Pedir regulação” na consulta"`, `DRAFT_NO_LONGER_VALID_TEXT = "um pedido de regulação em rascunho não vale mais para o seu CBO ou saiu da lista — edite ou retire o pedido e finalize de novo"`, `regulationFinalizeError(err: unknown): string | null` (`regulation_request_required` → `REGULATION_REQUIRED_TEXT`; recusa com `index` numérico no corpo — a do `Promote` do api — → `DRAFT_NO_LONGER_VALID_TEXT`; outra → `null`, e o editor usa a frase de hoje), `guideFilename(id)`, `exportFilename(today)`, `draftsNote(n: number): string`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/regulation.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  actionsFor, allowedTypes, canRequestRegulation, canUseRegulation, colorLabel, colorTone, deadlinesProblem, draftsNote, dueText, exportFilename,
  externalNumberProblem, guideFilename, monthLabel, parseQuantity, quantityProblem, reasonProblem, regulationError, requestBody, requestProblem,
  regulationFinalizeError, requesterKind, scheduleAtProblem, statusLabel, targetsFor, targetTermProblem, waitDays, waitText, DRAFT_NO_LONGER_VALID_TEXT,
  QUOTA_EXHAUSTED_TEXT, REGULATION_DISABLED, REGULATION_REQUIRED_TEXT, STALE_TEXT, type RequestDraft
} from "./regulation";
import { attendanceError } from "./attendance";
import { CARDIO, ECO, ENDO, TODAY26, regRequest, regUser } from "../test/regulationFixtures";

const err = (status: number, error: string, extra: Record<string, unknown> = {}) => new ApiError(status, { error, ...extra }, "x");

describe("portão e quem pede", () => {
  it("interruptor, operador e o prontuário para pedir", () => {
    expect(canUseRegulation(regUser())).toBe(true);
    expect(canUseRegulation(regUser([ "care_regulator" ], { features: [] }))).toBe(false);
    expect(canUseRegulation(regUser([ "care_regulator" ], { operator: true }))).toBe(false);
    expect(canRequestRegulation(regUser([ "health_professional" ]))).toBe(true);
    expect(canRequestRegulation(regUser([ "health_professional" ], { features: [ "care_regulation" ] }))).toBe(false);
    expect(canRequestRegulation(regUser([ "care_regulator" ]))).toBe(false);
  });
  it.each([
    [ "225142", "physician", [ "specialty", "procedure" ] ], [ "225120", "physician", [ "specialty", "procedure" ] ],
    [ "223505", "nurse", [ "specialty", "procedure" ] ], [ "223208", "dentist", [ "specialty" ] ],
    [ "322205", null, [] ], [ "", null, [] ], [ null, null, [] ]
  ])("CBO %s → %s", (cbo, kind, types) => {
    expect(requesterKind(cbo)).toBe(kind);
    expect(allowedTypes(cbo)).toEqual(types);
  });
  it("alvos que cada CBO vê na busca (o api recusa de novo)", () => {
    const all = [ CARDIO, ECO, ENDO ];
    expect(targetsFor("225142", all)).toEqual(all);
    expect(targetsFor("223505", all)).toEqual([ ECO ]);
    expect(targetsFor("223208", all)).toEqual([ ENDO ]);
    expect(targetsFor(null, all)).toEqual(all);
  });
});

describe("rótulos: código novo aparece cru", () => {
  it("cores e situações", () => {
    expect([ colorLabel("red"), colorTone("red"), colorLabel("blue"), colorTone("blue"), colorLabel("purple"), colorTone("purple") ])
      .toEqual([ "vermelho", "down", "azul", "info", "purple", "neutral" ]);
    expect([ statusLabel("returned"), statusLabel("closed"), statusLabel("external"), statusLabel("archived") ])
      .toEqual([ "devolvido para complementar", "encerrado (segunda falta)", "regulado pelo estado", "archived" ]);
  });
});

describe("ações por situação (D13, D16)", () => {
  const own = { offer_id: "of1", provider: { id: "pr1", name: "Ambulatório", kind: "own" }, month: "2026-10", skip_reason: null,
    at: "2026-10-20T10:00:00-03:00", by: { name: "Regulador" } };
  const contracted = { ...own, provider: { id: "pr2", name: "Clínica do Coração", kind: "contracted" } };
  it.each([
    [ "pending", null, false, { authorize: true, returnBack: true, deny: true, cancel: true, schedule: false, outcome: false, external: true } ],
    [ "returned", null, false, { authorize: false, returnBack: false, deny: true, cancel: true, schedule: false, outcome: false, external: false } ],
    [ "authorized", own, false, { authorize: false, returnBack: false, deny: false, cancel: true, schedule: false, outcome: false, external: false } ],
    [ "authorized", contracted, false, { authorize: false, returnBack: false, deny: false, cancel: true, schedule: true, outcome: false, external: false } ],
    [ "scheduled", contracted, false, { authorize: false, returnBack: false, deny: false, cancel: true, schedule: true, outcome: true, external: false } ],
    [ "scheduled", own, false, { authorize: false, returnBack: false, deny: false, cancel: false, schedule: false, outcome: false, external: false } ],
    [ "external", null, true, { authorize: false, returnBack: false, deny: false, cancel: true, schedule: true, outcome: false, external: true } ],
    [ "scheduled", null, true, { authorize: false, returnBack: false, deny: false, cancel: true, schedule: true, outcome: true, external: true } ],
    [ "completed", own, false, { authorize: false, returnBack: false, deny: false, cancel: false, schedule: false, outcome: false, external: false } ],
    [ "archived", null, false, { authorize: false, returnBack: false, deny: false, cancel: false, schedule: false, outcome: false, external: false } ]
  ])("%s (%#)", (status, authorization, stateRegulated, expected) => {
    expect(actionsFor(regRequest({ status, authorization, state_regulated: stateRegulated }))).toEqual(expected);
  });
});

describe("espera e prazo no dia da cidade", () => {
  it("espera conta do dia do pedido, no fuso da cidade", () => {
    expect(waitDays("2026-10-08T10:00:00-03:00", TODAY26)).toBe(20);
    // 01h30 UTC do dia 28 ainda é dia 27 em São Paulo.
    expect(waitDays("2026-10-28T01:30:00Z", TODAY26)).toBe(1);
    expect([ waitText(0), waitText(1), waitText(20) ]).toEqual([ "hoje", "1 dia", "20 dias" ]);
  });
  it("prazo da cor", () => {
    expect(dueText("2026-11-07", false, TODAY26)).toBe("até 07/11");
    expect(dueText("2026-10-28", false, TODAY26)).toBe("vence hoje");
    expect(dueText("2026-10-25", true, TODAY26)).toBe("vencido há 3 dias");
    expect(dueText("2026-10-27", true, TODAY26)).toBe("vencido há 1 dia");
    expect(dueText("2027-02-01", false, TODAY26)).toBe("até 01/02/2027");
    expect(dueText(null, false, TODAY26)).toBe("—");
    expect(monthLabel("2026-10")).toBe("outubro de 2026");
  });
});

describe("formato: recusar, nunca cortar", () => {
  const draft = (over: Partial<RequestDraft> = {}): RequestDraft => ({ type: "specialty", target: CARDIO, cid10: "i20.9",
    justification: "Dor torácica aos esforços há 3 meses.", color: "yellow", stateRegulated: false, ...over });
  it("pedido: alvo, CID, justificativa e cor, nesta ordem", () => {
    expect(requestProblem(draft({ target: null }), true)).toEqual({ field: "target", text: "escolha a especialidade ou o procedimento" });
    expect(requestProblem(draft({ cid10: "" }), true)).toEqual({ field: "cid10", text: "informe o CID-10" });
    expect(requestProblem(draft({ cid10: "I2" }), true)?.field).toBe("cid10");
    expect(requestProblem(draft({ cid10: "" }), false)).toBeNull();
    expect(requestProblem(draft({ justification: "  curta   " }), true))
      .toEqual({ field: "justification", text: "a justificativa precisa de pelo menos 10 caracteres" });
    expect(requestProblem(draft({ justification: "x".repeat(2001) }), true))
      .toEqual({ field: "justification", text: "a justificativa passa de 2000 caracteres" });
    expect(requestProblem(draft({ justification: "x".repeat(2000) }), true)).toBeNull();
    expect(requestProblem(draft({ color: "" }), true)).toEqual({ field: "color", text: "escolha a cor sugerida" });
  });
  it("corpo: CID normalizado, sem CID para quem não usa, estado só quando marcado", () => {
    expect(requestBody(draft(), true)).toEqual({ type: "specialty", target_code: "225120", cid10: "I209",
      justification: "Dor torácica aos esforços há 3 meses.", suggested_color: "yellow" });
    expect(requestBody(draft({ stateRegulated: true }), false)).toEqual({ type: "specialty", target_code: "225120",
      justification: "Dor torácica aos esforços há 3 meses.", suggested_color: "yellow", state_regulated: true });
  });
  it("motivo, número externo, quantidade, data", () => {
    expect([ reasonProblem("curto     "), reasonProblem("motivo ok!"), reasonProblem("x".repeat(501)) ])
      .toEqual([ "o motivo precisa de 10 a 500 caracteres", null, "o motivo precisa de 10 a 500 caracteres" ]);
    expect([ targetTermProblem("ca"), targetTermProblem("car"), targetTermProblem("c".repeat(61)) ])
      .toEqual([ "o termo de busca tem de 3 a 60 caracteres", null, "o termo de busca tem de 3 a 60 caracteres" ]);
    expect([ externalNumberProblem("2026/123456"), externalNumberProblem("SR 12"), externalNumberProblem("1".repeat(31)) ])
      .toEqual([ null, "número inválido: até 30 letras, números, ponto, barra ou hífen", "número inválido: até 30 letras, números, ponto, barra ou hífen" ]);
    expect([ quantityProblem("10"), quantityProblem("0"), quantityProblem("10,5"), quantityProblem("1.0"), quantityProblem("10000"), quantityProblem("") ])
      .toEqual([ null, null, "quantidade é um número inteiro de 0 a 9999", "quantidade é um número inteiro de 0 a 9999",
        "quantidade é um número inteiro de 0 a 9999", "quantidade é um número inteiro de 0 a 9999" ]);
    expect(parseQuantity(" 12 ")).toBe(12);
    expect([ scheduleAtProblem("2026-11-10T14:30"), scheduleAtProblem("2026-13-10T14:30"), scheduleAtProblem("") ])
      .toEqual([ null, "data e hora inválidas", "data e hora inválidas" ]);
  });
  it("prazos: inteiros de 1 a 365, do mais grave ao menos grave", () => {
    expect(deadlinesProblem({ red: "7", yellow: "30", green: "90", blue: "180" })).toBeNull();
    expect(deadlinesProblem({ red: "7", yellow: "30", green: "90", blue: "366" })).toBe("azul: prazo é um número inteiro de 1 a 365 dias");
    expect(deadlinesProblem({ red: "7,5", yellow: "30", green: "90", blue: "180" })).toBe("vermelho: prazo é um número inteiro de 1 a 365 dias");
    expect(deadlinesProblem({ red: "7", yellow: "5", green: "90", blue: "180" })).toBe("o prazo do amarelo não pode ser menor que o do vermelho");
  });
});

describe("frases das recusas, sem repetir o que foi enviado", () => {
  it.each([
    [ err(403, "feature_disabled", { feature: "care_regulation" }), REGULATION_DISABLED ],
    [ err(409, "stale_version"), STALE_TEXT ],
    [ err(409, "quota_exhausted"), QUOTA_EXHAUSTED_TEXT ],
    [ err(422, "color_reason_required"), "ao mudar a cor, informe o motivo" ],
    [ err(403, "cbo_not_allowed"), "seu CBO não pode pedir este encaminhamento: médico pede qualquer especialidade ou procedimento; enfermeiro, só os da lista do município; dentista, só especialidades odontológicas" ],
    [ err(422, "invalid_request", { path: "justification" }), "confira o campo: justificativa" ],
    [ err(422, "quantity_below_used", { used: 8 }), "a quantidade não pode ficar abaixo do que já foi autorizado no mês (8)" ],
    [ err(409, "appointment_scheduled"), "a unidade já marcou o horário: o cidadão cancela pelo celular, ou a falta devolve o pedido à regulação" ],
    [ err(409, "citizen_not_verified"), "o paciente não tem cadastro validado no posto, e a rede própria precisa dele para marcar o horário — escolha um prestador externo ou peça a validação" ],
    [ err(422, "invalid_offer"), "esta oferta não serve para este pedido (outro alvo, outro mês ou prestador inativo) — escolha outra" ],
    [ err(422, "invalid_request", { path: "consultation" }), "no máximo 10 pedidos de regulação por consulta" ],
    [ err(500, "boom"), "não foi possível concluir — tente de novo" ],
    [ new TypeError("Failed to fetch"), "sem conexão com o servidor — tente de novo" ]
  ])("%#", (e, text) => expect(regulationError(e)).toBe(text));
  it("recusas da finalização por causa da regulação; e o Encerrar da fila", () => {
    expect(regulationFinalizeError(err(422, "regulation_request_required"))).toBe(REGULATION_REQUIRED_TEXT);
    expect(regulationFinalizeError(err(403, "cbo_not_allowed", { index: 0, path: "target_code" }))).toBe(DRAFT_NO_LONGER_VALID_TEXT);
    // Sem `index`, o cbo_not_allowed é o da própria consulta: fica com a frase de hoje.
    expect(regulationFinalizeError(err(403, "cbo_not_allowed"))).toBeNull();
    expect(attendanceError(err(422, "regulation_request_required")))
      .toBe("com a regulação ligada, encaminhamento sem unidade de destino é pelo “Pedir regulação” na consulta — escolha a unidade da rede");
  });
  it("arquivos e avisos", () => {
    expect(guideFilename("rr1/../x")).toBe("guia-regulacao-rr1x.pdf");
    expect(exportFilename(TODAY26)).toBe("regulacao-estado-2026-10-28.csv");
    expect([ draftsNote(1), draftsNote(2) ]).toEqual([ "1 pedido de regulação vai para a fila da regulação ao finalizar.",
      "2 pedidos de regulação vão para a fila da regulação ao finalizar." ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/lib/regulation.test.ts`
Expected: FAIL — `./regulation` não existe.

- [ ] **Step 3: Implemente `src/lib/regulation.ts`**

Tabelas de rótulo (código novo cru):
- situação: `draft` "rascunho", `pending` "aguardando regulação", `returned` "devolvido para complementar", `authorized` "autorizado", `scheduled` "agendado", `completed` "realizado", `missed` "faltou", `denied` "negado", `cancelled` "cancelado", `closed` "encerrado (segunda falta)", `external` "regulado pelo estado"; tons `pending`/`returned` `warn`, `authorized`/`scheduled` `info`, `completed` `ok`, `denied`/`closed` `down`, o resto `neutral`;
- tipo: `specialty` "consulta especializada", `procedure` "procedimento"; prestador: `own` "rede própria", `contracted` "contratado", `consortium` "consórcio", `ppi` "PPI"; sistemas: `sisreg` "SISREG", `care` "CARE", `cross` "CROSS", `other` "outro".
- `actionsFor`: `authorize`, `returnBack` e `deny` só em `pending` (`deny` também em `returned`); `cancel` em `pending`, `returned`, `authorized`, `scheduled` (não na rede própria com horário marcado: o api recusa com 409 `appointment_scheduled`, R4) e `external`; `schedule` quando não é rede própria — (`authorized` ou `scheduled`) com `authorization.provider.kind !== "own"`, ou `state_regulated` em `external`/`scheduled`; `outcome` em `scheduled` não-próprio (ou `state_regulated`); `external` em `pending` e em pedido `state_regulated` ainda não encerrado. A tabela do teste é a regra.
- `waitDays`: `cityIsoDate(new Date(requestedAt))` contra `today`, em dias inteiros (as duas datas em UTC ao meio-dia, sem deslocamento de fuso). `dueText` usa `fmtDueOn` de `scheduling.ts` para o "até …" (ano só quando difere).
- `requestProblem` na ordem do teste; `cid10` obrigatório só com `cid10Allowed` (D3); o texto **nunca** é cortado: `justificationProblem` mede `trim()`, mas `requestBody` manda o texto como foi digitado, só sem espaços nas pontas.
- `reasonProblem` mede `trim()` em 10–500; `placeProblem`: vazio ou acima de 200 → "informe o local (até 200 caracteres)"; `notesProblem`: acima de 500 → "a observação passa de 500 caracteres"; `targetTermProblem` 3–60.
- `quantityProblem`: `/^\s*\d{1,4}\s*$/`; `deadlinesProblem`: cada cor por `/^\d{1,3}$/` e 1–365, depois a ordem `red ≤ yellow ≤ green ≤ blue` (frase "o prazo do <cor> não pode ser menor que o do <cor mais grave>").
- `regulationError`: `featureDisabledKey(err) !== null` → `REGULATION_DISABLED`; `TypeError` → "sem conexão com o servidor — tente de novo"; `missing_role` → `REGULATION_NO_ROLE`; `not_found` → "pedido não encontrado"; `invalid_transition`/`not_draft` → "este pedido mudou de situação — recarregue"; `invalid_target` → "especialidade ou procedimento inválido" (com `hint: "exam_order"`: "este procedimento é exame: peça pelo pedido de exame da consulta"); `invalid_month` → "o mês da oferta não pode ser anterior ao corrente"; `cnes_taken` → "já existe prestador com este CNES"; `unit_taken` → "esta unidade já é um prestador da rede própria"; `invalid_query` → "o termo de busca tem de 3 a 60 caracteres"; `invalid_cid10` → "CID-10 inválido"; `skip_reason_required` → `SKIP_REASON_TEXT`; `duplicate_offer` → "já existe oferta deste prestador para este alvo neste mês"; `duplicate_item` → "este item já está na lista"; `invalid_deadlines` → "prazos inválidos"; `invalid_request` com `path` → "confira o campo: <rótulo do path>" (`justification` "justificativa", `cid10` "CID-10", `reason` "motivo", `place` "local", `at` "data e hora", `number` "número", `cnes` "CNES", `quantity` "quantidade"; outro path cru); `invalid_request` com `path: "consultation"` → "no máximo 10 pedidos de regulação por consulta"; os demais do teste; qualquer outro → "não foi possível concluir — tente de novo". A frase nunca repete valor enviado.
- `guideFilename`: só `[A-Za-z0-9-]` do id.
- Em `src/lib/attendance.ts`, no `MESSAGES`: `regulation_request_required: "com a regulação ligada, encaminhamento sem unidade de destino é pelo “Pedir regulação” na consulta — escolha a unidade da rede"` (comentário: Módulo 26a, Desvio 2 do plano do api).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/lib/regulation.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/regulation.ts src/lib/regulation.test.ts
$G commit -m "feat: add care regulation screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Regulação", contador, portão e papel na Equipe

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Create: `src/hooks/useRegulationCount.ts`, `src/modules/regulation/RegulationGate.tsx`, `src/modules/RegulationQueue.tsx`, `src/modules/RegulationReturned.tsx`, `src/modules/RegulationOffers.tsx`, `src/modules/RegulationSettings.tsx` (portão + `EmptyState` "em construção"; as Tasks 7, 9, 10 e 11 trocam o corpo)
- Test: `src/shell/modules.regulation.test.ts`, `src/hooks/useRegulationCount.test.tsx`

**Interfaces:**
- Consumes: `canUseRegulation`, `regulationRoles`, `REG_COUNT_KEY`, `COUNT_QUERY` (Task 2); `countRegulationRequests` (Task 1).
- Produces: `ModuleId` com `"regulation-queue" | "regulation-returned" | "regulation-offers" | "regulation-settings"`; o grupo `{ label: "Regulação" }` logo depois de "Vigilância"; `withRegulationCount(groups, overdue: number): NavGroupDef[]`; `shellGroups(…, surveillanceImmediate = 0, regulationOverdue = 0)`; `useRegulationCount(enabled: boolean, userId: string | null)` (chave `[REG_COUNT_KEY, userId]`, `REGULATION_COUNT_REFRESH_MS = 60_000`); `RegulationGate({ title, sub, allow(roles), children(roles) })`; papel `care_regulator` "Regulador de encaminhamentos", privilegiado.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.regulation.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor, withRegulationCount } from "./modules";
import { CATALOG } from "./catalog";

const user = (roles: string[], features: string[] | undefined = [ "care_regulation", "clinical_record" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });
const itemsOf = (groups: ReturnType<typeof navGroupsFor>) => groups.find((g) => g.label === "Regulação")?.items.map((i) => i.id) ?? [];

describe("grupo Regulação (módulo 26a)", () => {
  it("cada papel vê o seu", () => {
    expect(itemsOf(navGroupsFor(user([ "care_regulator" ])))).toEqual([ "regulation-queue", "regulation-offers" ]);
    expect(itemsOf(navGroupsFor(user([ "health_professional" ])))).toEqual([ "regulation-returned" ]);
    expect(itemsOf(navGroupsFor(user([ "municipal_admin" ])))).toEqual([ "regulation-offers", "regulation-settings" ]);
    expect(itemsOf(navGroupsFor(user([ "care_regulator", "municipal_admin" ])))).toEqual([
      "regulation-queue", "regulation-offers", "regulation-settings" ]);
  });
  it("some sem o interruptor, para outros papéis e para o operador", () => {
    expect(itemsOf(navGroupsFor(user([ "care_regulator", "municipal_admin", "health_professional" ], [ "clinical_record" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "citizen_verifier", "analyst", "exam_regulator" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([], undefined, true)))).toEqual([]);
  });
  it("vem depois da Vigilância, com ícone e descrição", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Regulação")).toBe(labels.indexOf("Vigilância") + 1);
    expect(CATALOG.groups["Regulação"].description).toBe("Encaminhamentos a especialista e procedimento, fila, cotas e prazos");
    expect(CATALOG.screens["regulation-queue"].description).toBe("Pedidos por cor e espera, vencidos primeiro");
    expect(CATALOG.screens["regulation-returned"].description).toBe("Pedidos da unidade devolvidos para complementar");
    expect(CATALOG.screens["regulation-offers"].description).toBe("Prestadores, ofertas e cota mensal com saldo");
    expect(CATALOG.screens["regulation-settings"].description).toBe("Prazos por cor e o que o enfermeiro pode pedir");
  });
  it("os vencidos no rótulo da fila", () => {
    const groups = navGroupsFor(user([ "care_regulator" ]));
    expect(withRegulationCount(groups, 0)).toBe(groups);
    const label = (n: number) => withRegulationCount(groups, n).find((g) => g.label === "Regulação")!.items[0].label;
    expect(label(1)).toBe("Fila regulada (1 vencido)");
    expect(label(4)).toBe("Fila regulada (4 vencidos)");
  });
});
```

```tsx
// src/hooks/useRegulationCount.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, renderHook, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), countRegulationRequests: vi.fn() }));
import * as api from "../lib/api";
import { REGULATION_COUNT_REFRESH_MS, useRegulationCount } from "./useRegulationCount";
import { regCounts } from "../test/regulationFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const wrapper = ({ children }: { children: ReactNode }) =>
  <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>{children}</QueryClientProvider>;

describe("useRegulationCount", () => {
  it("a contagem relê a cada minuto e acende o item com os vencidos", async () => {
    vi.useFakeTimers({ shouldAdvanceTime: true });
    mocked(api.countRegulationRequests).mockReset().mockResolvedValue(regCounts());
    const { result } = renderHook(() => useRegulationCount(true, "u-1"), { wrapper });
    await waitFor(() => expect(result.current.data?.overdue).toBe(3));
    expect(mocked(api.countRegulationRequests)).toHaveBeenCalledTimes(1);
    await act(async () => { await vi.advanceTimersByTimeAsync(REGULATION_COUNT_REFRESH_MS); });
    expect(mocked(api.countRegulationRequests)).toHaveBeenCalledTimes(2);
  });
  it("desligado não lê", () => {
    mocked(api.countRegulationRequests).mockReset();
    renderHook(() => useRegulationCount(false, "u-1"), { wrapper });
    expect(mocked(api.countRegulationRequests)).not.toHaveBeenCalled();
  });
});
```

Em `src/shell/modules.test.ts`, nos dois casos que conferem grupos escondidos, acrescente `expect(groups.some((g) => g.label === "Regulação")).toBe(false);` e troque `NAV_GROUPS.length - 13` por `- 14` e `NAV_GROUPS.length - 12` por `- 13`. Em `src/lib/team.test.ts`, o convite passa a ter **15** papéis (`"care_regulator"` na lista ordenada, entre `"campaign_manager"` e `"citizen_verifier"`) e:

```ts
    expect(INVITE_ROLES.find((r) => r.role === "care_regulator")?.label).toBe("Regulador de encaminhamentos");
    expect(isPrivilegedRole("care_regulator")).toBe(true);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/shell src/hooks/useRegulationCount.test.tsx src/lib/team.test.ts`
Expected: FAIL — grupo, ícones, contador e papel não existem.

- [ ] **Step 3: Implemente**

- `modules.ts`: o grupo depois de "Vigilância" com `{ id: "regulation-queue", label: "Fila regulada", icon: "⇉" }`, `{ id: "regulation-returned", label: "Encaminhamentos devolvidos", icon: "↩" }`, `{ id: "regulation-offers", label: "Prestadores e cotas", icon: "▣" }`, `{ id: "regulation-settings", label: "Prazos e lista do enfermeiro", icon: "⏲" }`; no filtro, `regOn = canUseRegulation(user)` e `isCareRegulator = roles.includes("care_regulator")`: fila só `regOn && isCareRegulator`; devolvidos `regOn && isProfessional && hasFeature(user, "clinical_record")`; ofertas `regOn && (isAdmin || isCareRegulator)`; configurações `regOn && isAdmin`. `withRegulationCount` no molde do `withSurveillanceCount` ("1 vencido"/"N vencidos" no item `regulation-queue`).
- `catalog.ts`: grupo `"Regulação": { icon: IconArrowsSplit2, description: "Encaminhamentos a especialista e procedimento, fila, cotas e prazos" }`; telas com `IconListNumbers`, `IconArrowBackUp`, `IconBuildingStore`, `IconClockHour4` e as descrições do teste.
- `useRegulationCount.ts`: cópia do `useSurveillanceCount` com `countRegulationRequests` e `REG_COUNT_KEY`.
- `App.tsx`: `useRegulationCount(canUseRegulation(auth.user) && regulationRoles(auth.user).regulator, auth.user?.id ?? null)`; `shellGroups` ganha `regulationOverdue = 0` (envolve com `withRegulationCount` e põe `count` no item da fila, como a vigilância); os quatro `case` (`regulation-settings` com `onGoToSecurity={() => setActive("security")}`, como as telas com step-up).
- `RegulationGate.tsx`: cópia do `SurveillanceGate` com as frases e o portão do 26a. As quatro telas nascem com o portão (`allow` pelo papel da Task 3) e um `EmptyState` provisório.
- `team.ts`: `{ role: "care_regulator", label: "Regulador de encaminhamentos" }` antes de `municipal_admin` e no `PRIVILEGED_ROLES`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/shell src/hooks src/lib/team.test.ts src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos de `modules.test.ts` só mudaram as contagens).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.regulation.test.ts src/shell/catalog.ts src/App.tsx src/lib/team.ts src/lib/team.test.ts \
  src/hooks/useRegulationCount.ts src/hooks/useRegulationCount.test.tsx src/modules/regulation/RegulationGate.tsx \
  src/modules/RegulationQueue.tsx src/modules/RegulationReturned.tsx src/modules/RegulationOffers.tsx src/modules/RegulationSettings.tsx
$G commit -m "feat: add the care regulation menu group and regulator role

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Busca de alvo e formulário do pedido

**Files:**
- Create: `src/modules/regulation/TargetSearch.tsx`, `src/modules/regulation/RequestForm.tsx`
- Test: `src/modules/regulation/RequestForm.test.tsx`

**Interfaces:**
- Consumes: `searchRegulationTargets` (Task 1); `RequestDraft`, `requestProblem`, `requestBody`, `allowedTypes`, `targetsFor`, `COLORS`, `targetTermProblem`, `REG_TARGETS_KEY`, `regulationError` (Task 2); `useDebouncedValue`.
- Produces:
  - `TargetSearch({ type, cbo?, label?, delayMs?, onPick(t: RegulationTarget) })` — termo de 3 a 60 (recusa fora disso, sem chamar), resultados filtrados por `targetsFor(cbo, …)`, `gcTime: 0`, lista de botões `"<código> — <nome>"`, limpa o campo ao escolher.
  - `RequestForm({ cbo, cid10Allowed, initial?: RequestDraft, submitLabel, mode: "create" | "complement", delayMs?, onSubmit(body: RegulationRequestInput): Promise<void>, onCancel(), onDirty?(dirty: boolean) })` — em `complement`, tipo e alvo ficam só leitura e o "regulado pelo estado" não aparece; recusa antes de enviar com a frase da Task 2 e o foco no campo; a recusa do api (lançada por `onSubmit`) aparece em `role="alert"` e o formulário fica; `onDirty(true)` quando há texto ou escolha não enviada.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/regulation/RequestForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  searchRegulationTargets: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { RequestForm } from "./RequestForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CARDIO, ECO, ENDO, regUser } from "../../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const JUST = "Dor torácica aos esforços há 3 meses, teste ergométrico alterado.";

describe("RequestForm", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "health_professional" ]));
    mocked(api.searchRegulationTargets).mockReset().mockImplementation(async (_q: string, type: string) => (type === "procedure" ? [ ECO ] : [ CARDIO, ENDO ]));
  });
  const mount = (over: Partial<Parameters<typeof RequestForm>[0]> = {}) => {
    const onSubmit = vi.fn(async () => undefined);
    renderWithProviders(<RequestForm cbo="225142" cid10Allowed submitLabel="Salvar pedido" mode="create" delayMs={0}
      onSubmit={onSubmit} onCancel={vi.fn()} {...over} />);
    return onSubmit;
  };
  const pick = async (term: string, name: string) => {
    text("Especialidade ou procedimento", term);
    fireEvent.click(await screen.findByRole("button", { name: new RegExp(name) }));
  };

  it("médico: alvo pela busca, CID, justificativa e cor; o corpo exato", async () => {
    const onSubmit = mount();
    await pick("cardio", "Cardiologia");
    expect(mocked(api.searchRegulationTargets)).toHaveBeenLastCalledWith("cardio", "specialty");
    text("CID-10", "i20.9"); text("Justificativa", JUST); text("Cor de risco sugerida", "yellow");
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({ type: "specialty", target_code: "225120", cid10: "I209", justification: JUST,
      suggested_color: "yellow" }));
  });

  it("procedimento e regulado pelo estado", async () => {
    const onSubmit = mount();
    fireEvent.click(screen.getByLabelText("Procedimento"));
    await pick("eco", "Ecocardiografia");
    text("CID-10", "I209"); text("Justificativa", JUST); text("Cor de risco sugerida", "green");
    fireEvent.click(screen.getByLabelText("Regulado pelo estado (SISREG, CARE, CROSS)"));
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    await waitFor(() => expect(onSubmit.mock.calls[0][0]).toMatchObject({ type: "procedure", target_code: "0205010032", state_regulated: true }));
  });

  it("recusa antes de enviar e o texto fica, sem cortar", async () => {
    const onSubmit = mount();
    text("Justificativa", "x".repeat(2001));
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    expect((await screen.findByRole("alert")).textContent).toBe("escolha a especialidade ou o procedimento");
    await pick("cardio", "Cardiologia");
    text("CID-10", "I2"); text("Cor de risco sugerida", "red");
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    expect((await screen.findByRole("alert")).textContent).toBe("use um código CID-10, ex.: E11 ou E119");
    text("CID-10", "I209");
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    expect((await screen.findByRole("alert")).textContent).toBe("a justificativa passa de 2000 caracteres");
    expect((screen.getByLabelText("Justificativa") as HTMLTextAreaElement).value).toHaveLength(2001);
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("dentista: só especialidades odontológicas; enfermeiro só a lista municipal e sem CID-10", async () => {
    mocked(api.searchRegulationTargets).mockResolvedValue([ CARDIO, ENDO ]);
    mount({ cbo: "223208" });
    expect(screen.queryByLabelText("Procedimento")).toBeNull();
    text("Especialidade ou procedimento", "endo");
    expect(await screen.findByRole("button", { name: /Endodontia/ })).not.toBeNull();
    expect(screen.queryByRole("button", { name: /Cardiologia/ })).toBeNull();
    cleanup();
    mocked(api.searchRegulationTargets).mockResolvedValue([ ECO, { ...ECO, code: "0205010040", name: "Ecografia de abdômen total", nurse_allowed: false } ]);
    const onSubmit = mount({ cbo: "223505", cid10Allowed: false });
    expect(screen.queryByLabelText("CID-10")).toBeNull();
    fireEvent.click(screen.getByLabelText("Procedimento"));
    text("Especialidade ou procedimento", "eco");
    expect(await screen.findByRole("button", { name: /Ecocardiografia/ })).not.toBeNull();
    expect(screen.queryByRole("button", { name: /abdômen/ })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: /Ecocardiografia/ }));
    text("Justificativa", JUST); text("Cor de risco sugerida", "blue");
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    await waitFor(() => expect(onSubmit.mock.calls[0][0]).not.toHaveProperty("cid10"));
  });

  it("recusa do api (403 cbo_not_allowed) fica no formulário, com o que foi digitado", async () => {
    const onSubmit = mount();
    onSubmit.mockRejectedValueOnce(new ApiError(403, { error: "cbo_not_allowed" }, "x"));
    await pick("cardio", "Cardiologia");
    text("CID-10", "I209"); text("Justificativa", JUST); text("Cor de risco sugerida", "yellow");
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    expect((await screen.findByRole("alert")).textContent).toMatch(/^seu CBO não pode pedir este encaminhamento/);
    expect((screen.getByLabelText("Justificativa") as HTMLTextAreaElement).value).toBe(JUST);
  });

  it("termo curto ou longo não chama o api", async () => {
    mount();
    text("Especialidade ou procedimento", "ca");
    expect(await screen.findByText("o termo de busca tem de 3 a 60 caracteres")).not.toBeNull();
    text("Especialidade ou procedimento", "c".repeat(61));
    expect(await screen.findByText("o termo de busca tem de 3 a 60 caracteres")).not.toBeNull();
    expect(mocked(api.searchRegulationTargets)).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation/RequestForm.test.tsx`
Expected: FAIL — os componentes não existem.

- [ ] **Step 3: Implemente `TargetSearch` e `RequestForm`**

`TargetSearch` segue o `SigtapSearch` do 22 (termo debounced, recusa de formato sem chamada, `gcTime: 0`, `retry: false`, sem refetch em foco/reconexão), com `searchRegulationTargets(term, type)` e o resultado passado por `targetsFor(cbo, …)` (o enfermeiro vê só a lista municipal; o dentista, só especialidades odontológicas). Busca de especialidade e de procedimento com o mesmo mínimo de 3 caracteres. `RequestForm`: tipo por rádio só se `allowedTypes(cbo).length > 1` (rótulos "Especialidade" e "Procedimento"; trocar de tipo limpa o alvo); alvo escolhido mostrado como texto com "Trocar"; `CID-10` (input de texto, só com `cid10Allowed`); `Justificativa` (textarea, com o contador "N de 2000", sem `maxLength`); `Cor de risco sugerida` (select com as quatro cores de `COLORS` e "escolha…"); "Regulado pelo estado (SISREG, CARE, CROSS)" (checkbox, só em `create`). Os estilos de `src/modules/exams/styles.ts`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation/RequestForm.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/regulation/TargetSearch.tsx src/modules/regulation/RequestForm.tsx src/modules/regulation/RequestForm.test.tsx
$G commit -m "feat: add the regulation request form with target search

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: "Pedir regulação" no rascunho da consulta, sem bloquear (F-26.2)

**Files:**
- Create: `src/modules/consultation/RegulationRequestsField.tsx`
- Modify: `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/attendance/OutcomeFields.tsx`
- Test: `src/modules/consultation/RegulationRequestsField.test.tsx`, `src/modules/consultation/ConsultationEditor.test.tsx` (describe novo)

**Interfaces:**
- Consumes: `RequestForm` (Task 4); `listConsultationRegulationRequests`, `createRegulationRequest`, `updateRegulationRequest`, `deleteRegulationRequest` (Task 1); `REG_CONSULTATION_KEY`, `NOMINAL_QUERY`, `draftsNote`, `UNSAVED_REQUEST_NOTE`, `ADDENDUM_NOTE`, `REFERRAL_HINT`, `colorLabel`, `colorTone`, `statusLabel`, `draftFromRequest`, `regulationError` (Task 2).
- Produces:
  - `RegulationRequestsField({ consultationId, cbo, cid10Allowed, mode: "draft" | "addendum", finalizing?: boolean, noteTarget?: HTMLElement | null, searchDelayMs?, onOpenGuide?(id: string): void })` — `region` "Pedidos de regulação"; lista os pedidos da consulta (`draft` com "Editar" e "Retirar"; os já enviados com a situação e, se `onOpenGuide`, "Guia em PDF"); botão "Pedir regulação" abre o `RequestForm`; com `finalizing`, põe no destino o `draftsNote(n)` (se há rascunho) e o `UNSAVED_REQUEST_NOTE` (se o formulário está sujo); em `addendum`, mostra `ADDENDUM_NOTE`.
  - `RegulationRequestsField` aceita também `refreshKey?: number` (mudou → relê a lista).
  - `ConsultationEditorProps.regulation?: boolean` (a peça vem depois da notificação do 25, antes de "Finalizar"); no `confirm`, `regulationFinalizeError(err) ?? consultationError(err)` e, quando a recusa é da regulação, `refreshKey + 1` para a peça reler; o destino dos avisos passa a existir quando `notifications || regulation`.
  - `OutcomeFields` ganha `regulationHint?: boolean` (mostra `REFERRAL_HINT` sob "Encaminhado").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/RegulationRequestsField.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  searchRegulationTargets: vi.fn(), listConsultationRegulationRequests: vi.fn(), createRegulationRequest: vi.fn(), updateRegulationRequest: vi.fn(),
  deleteRegulationRequest: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { RegulationRequestsField } from "./RegulationRequestsField";
import { ADDENDUM_NOTE, UNSAVED_REQUEST_NOTE } from "../../lib/regulation";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CARDIO, regRequest, regUser } from "../../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const JUST = "Dor torácica aos esforços há 3 meses, teste ergométrico alterado.";
const DRAFT = regRequest({ status: "draft", requested_at: null, due_on: null, version: 1, events: [] });

describe("RegulationRequestsField", () => {
  beforeEach(() => {
    for (const fn of [ api.listConsultationRegulationRequests, api.createRegulationRequest, api.updateRegulationRequest, api.deleteRegulationRequest ])
      mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "health_professional" ]));
    mocked(api.searchRegulationTargets).mockResolvedValue([ CARDIO ]);
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([]);
    mocked(api.createRegulationRequest).mockResolvedValue(DRAFT);
  });
  const mount = (over: Partial<Parameters<typeof RegulationRequestsField>[0]> = {}) => renderWithProviders(
    <RegulationRequestsField consultationId="cs1" cbo="225142" cid10Allowed mode="draft" searchDelayMs={0} {...over} />);
  const fill = async () => {
    fireEvent.click(await screen.findByRole("button", { name: "Pedir regulação" }));
    text("Especialidade ou procedimento", "cardio");
    fireEvent.click(await screen.findByRole("button", { name: /Cardiologia/ }));
    text("CID-10", "I209"); text("Justificativa", JUST); text("Cor de risco sugerida", "yellow");
  };

  it("pedir: cria o rascunho e ele aparece na lista com a cor e a situação", async () => {
    mount();
    await fill();
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([ DRAFT ]);
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    await waitFor(() => expect(mocked(api.createRegulationRequest)).toHaveBeenCalledWith("cs1",
      { type: "specialty", target_code: "225120", cid10: "I209", justification: JUST, suggested_color: "yellow" }));
    const item = await screen.findByRole("listitem", { name: "Cardiologia" });
    expect(within(item).getByText("amarelo")).not.toBeNull();
    expect(within(item).getByText("rascunho")).not.toBeNull();
    expect(screen.queryByLabelText("Justificativa")).toBeNull();
  });

  it("editar manda o corpo inteiro; retirar pede confirmação", async () => {
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([ DRAFT ]);
    mocked(api.updateRegulationRequest).mockResolvedValue(DRAFT);
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Editar o pedido de Cardiologia" }));
    text("Cor de risco sugerida", "red");
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    await waitFor(() => expect(mocked(api.updateRegulationRequest)).toHaveBeenCalledWith("cs1", "rr1",
      { type: "specialty", target_code: "225120", cid10: "I209", justification: DRAFT.justification, suggested_color: "red" }));
    fireEvent.click(screen.getByRole("button", { name: "Retirar o pedido de Cardiologia" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.deleteRegulationRequest)).toHaveBeenCalledWith("cs1", "rr1"));
  });

  it("recusa do api e falha da leitura ficam na peça", async () => {
    mocked(api.createRegulationRequest).mockRejectedValue(new ApiError(422, { error: "invalid_target" }, "x"));
    mount();
    await fill();
    fireEvent.click(screen.getByRole("button", { name: "Salvar pedido" }));
    expect((await screen.findByRole("alert")).textContent).toBe("especialidade ou procedimento inválido");
    cleanup();
    mocked(api.listConsultationRegulationRequests).mockRejectedValue(new ApiError(500, {}, "x"));
    mount();
    expect((await screen.findByRole("alert")).textContent).toBe("não foi possível concluir — tente de novo");
  });

  it("ao finalizar: a conta dos rascunhos e o aviso do não salvo", async () => {
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([ DRAFT, regRequest({ id: "rr2", status: "draft", target: { code: "0205010032", name: "Ecocardiografia" } }) ]);
    mount({ finalizing: true });
    expect(await screen.findByText("2 pedidos de regulação vão para a fila da regulação ao finalizar.")).not.toBeNull();
    expect(screen.queryByText(UNSAVED_REQUEST_NOTE)).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Pedir regulação" }));
    text("Justificativa", "algo digitado e não salvo");
    expect(await screen.findByText(UNSAVED_REQUEST_NOTE)).not.toBeNull();
  });

  it("no adendo: o aviso de que vai com o próximo adendo e a situação dos já enviados", async () => {
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([ regRequest({ status: "authorized" }) ]);
    const onOpenGuide = vi.fn();
    mount({ mode: "addendum", onOpenGuide });
    expect(await screen.findByText(ADDENDUM_NOTE)).not.toBeNull();
    const item = screen.getByRole("listitem", { name: "Cardiologia" });
    expect(within(item).getByText("autorizado")).not.toBeNull();
    expect(within(item).queryByRole("button", { name: /Editar/ })).toBeNull();
    fireEvent.click(within(item).getByRole("button", { name: "Guia em PDF" }));
    expect(onOpenGuide).toHaveBeenCalledWith("rr1");
  });
});
```

No fim de `src/modules/consultation/ConsultationEditor.test.tsx` (acrescente `listConsultationRegulationRequests: vi.fn(), createRegulationRequest: vi.fn(), searchRegulationTargets: vi.fn()` ao `vi.mock` do topo e importe `UNSAVED_REQUEST_NOTE` de `../../lib/regulation`):

```tsx
describe("ConsultationEditor — pedido de regulação (módulo 26a)", () => {
  // O pedido nunca bloqueia a finalização: a peça fica fora de `missing`, `finalizeProblems` e `blockedFinal`.
  beforeEach(() => {
    for (const fn of [ api.saveConsultationDraft, api.finalizeConsultation, api.listConsultationRegulationRequests ]) mocked(fn).mockReset();
    mocked(api.saveConsultationDraft).mockImplementation(async () => consultation());
    mocked(api.finalizeConsultation).mockResolvedValue(finalized());
    mocked(api.searchRegulationTargets).mockResolvedValue([]);
  });
  const mount = (regulation = true) => render(
    <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
      <ConsultationEditor consultation={consultation()} record={record()} options={options()} unit={unit} units={units}
        regulation={regulation} autosaveDelayMs={0} searchDelayMs={0} onFinalized={vi.fn()} onLocked={vi.fn()} />
    </QueryClientProvider>);

  it.each([
    [ "formulário aberto e não salvo", () => { mocked(api.listConsultationRegulationRequests).mockResolvedValue([]); }, true ],
    [ "leitura que falha", () => { mocked(api.listConsultationRegulationRequests).mockRejectedValue(new ApiError(500, {}, "x")); }, false ],
    [ "leitura sem rede", () => { mocked(api.listConsultationRegulationRequests).mockRejectedValue(new TypeError("Failed to fetch")); }, false ]
  ])("formulário aberto ou leitura que falha (%s): a finalização segue igual", async (_, arrange, typing) => {
    arrange();
    mount();
    fillMinimum();
    const piece = await screen.findByRole("region", { name: "Pedidos de regulação" });
    if (typing) {
      fireEvent.click(within(piece).getByRole("button", { name: "Pedir regulação" }));
      fireEvent.change(within(piece).getByLabelText("Justificativa"), { target: { value: "começou e não salvou" } });
    } else {
      await within(piece).findByRole("alert");
    }
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    const box = screen.getByRole("region", { name: "Finalizar consulta" });
    if (typing) expect(await within(box).findByText(UNSAVED_REQUEST_NOTE)).not.toBeNull();
    expect(within(box).queryByRole("alert")).toBeNull();
    expect(within(box).queryByRole("list", { name: "o que falta para finalizar" })).toBeNull();
    const confirm = within(box).getByRole("button", { name: "Confirmar finalização" }) as HTMLButtonElement;
    expect(confirm.disabled).toBe(false);
    fireEvent.click(confirm);
    await waitFor(() => expect(mocked(api.finalizeConsultation)).toHaveBeenCalledTimes(1));
  });

  it.each([
    [ new ApiError(422, { error: "regulation_request_required" }, "x"), REGULATION_REQUIRED_TEXT ],
    [ new ApiError(403, { error: "cbo_not_allowed", index: 0, path: "target_code" }, "x"), DRAFT_NO_LONGER_VALID_TEXT ]
  ])("finalização recusada pela regra da regulação: a frase diz o que fazer, nada se perde, a peça relê (%#)", async (refusal, phrase) => {
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([]);
    mocked(api.finalizeConsultation).mockRejectedValueOnce(refusal);
    mount();
    fillMinimum();
    await screen.findByRole("region", { name: "Pedidos de regulação" });
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    const box = screen.getByRole("region", { name: "Finalizar consulta" });
    fireEvent.click(within(box).getByRole("button", { name: "Confirmar finalização" }));
    expect((await within(box).findByRole("alert")).textContent).toBe(phrase);
    expect((screen.getByLabelText("Plano (P)") as HTMLTextAreaElement).value).toBe("Ajuste de dose; retorno em 30 dias.");
    await waitFor(() => expect(mocked(api.listConsultationRegulationRequests).mock.calls.length).toBeGreaterThan(1));
  });

  it("ao finalizar: a conta dos rascunhos e o aviso do não salvo, dentro da caixa", async () => {
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([ { ...regRequest({ status: "draft" }) } ]);
    mount();
    await screen.findByRole("listitem", { name: "Cardiologia" });
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    const box = screen.getByRole("region", { name: "Finalizar consulta" });
    expect(await within(box).findByText("1 pedido de regulação vai para a fila da regulação ao finalizar.")).not.toBeNull();
  });

  it("desfecho Encaminhado: a dica de que especialista é pelo pedido; sem a prop, nem peça nem dica", async () => {
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([]);
    mount();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
    expect(screen.getByText(/só para outra unidade da rede/)).not.toBeNull();
    cleanup();
    mount(false);
    expect(screen.queryByRole("region", { name: "Pedidos de regulação" })).toBeNull();
    expect(mocked(api.listConsultationRegulationRequests)).toHaveBeenCalledTimes(1);
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred" } });
    expect(screen.queryByText(/só para outra unidade da rede/)).toBeNull();
  });
});
```

(Importe `regRequest` de `../../test/regulationFixtures` e `REGULATION_REQUIRED_TEXT`, `DRAFT_NO_LONGER_VALID_TEXT` de `../../lib/regulation`.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/consultation/RegulationRequestsField.test.tsx src/modules/consultation/ConsultationEditor.test.tsx`
Expected: FAIL — a peça e a prop não existem.

- [ ] **Step 3: Implemente**

- `RegulationRequestsField`: `useQuery({ queryKey: [REG_CONSULTATION_KEY, consultationId], queryFn, ...NOMINAL_QUERY })`; depois de criar, editar ou retirar, `refetch()` (nada de polling). Cada item é `<li aria-label={target.name}>` com o tipo, a cor (`Tag` com `colorTone`), a situação (`statusLabel`) e, no `draft`, "Editar o pedido de X" (abre o `RequestForm` com `draftFromRequest`) e "Retirar o pedido de X" (`ConfirmButton`). O aviso de finalização vai por `createPortal` para `noteTarget`, como o `NotificationSuggestions` (`undefined` = aqui mesmo; `null` = destino ainda não existe). Recusas e falhas em `role="alert"` **dentro** da peça.
- `ConsultationEditor`: prop `regulation`; a peça depois do `NotificationSuggestions` com `cid10Allowed={options.cid10_allowed_for_cbo}`, `cbo={consultation.cbo_code}`, `finalizing`, `noteTarget={noteHost}`; o `<div ref={setNoteHost}>` passa a existir quando `props.notifications || props.regulation`; passa `regulationHint={props.regulation}` ao `OutcomeFields`. **Nada** em `missing`, `finalizeProblems` ou `blockedFinal`.
- `OutcomeFields`: `regulationHint` mostra `<small>` com `REFERRAL_HINT` logo abaixo do select "Unidade de destino" (só em `referred`). O `ClosePanel` da fila não passa a prop: fica igual.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/consultation src/modules/attendance src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos do editor e da fila seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/consultation/RegulationRequestsField.tsx src/modules/consultation/RegulationRequestsField.test.tsx \
  src/modules/consultation/ConsultationEditor.tsx src/modules/consultation/ConsultationEditor.test.tsx src/modules/attendance/OutcomeFields.tsx
$G commit -m "feat: request regulation from the consultation draft without blocking finalization

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Pedido por adendo, guia em PDF e a ligação no atendimento (F-26.2)

**Files:**
- Create: `src/modules/regulation/openGuide.ts`
- Modify: `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/consultation/ConsultationWorkspace.regulation.test.tsx`

**Interfaces:**
- Consumes: `RegulationRequestsField` (Task 5); `fetchRegulationGuide` (Task 1); `canRequestRegulation`, `allowedTypes`, `regulationError` (Task 2).
- Produces: `openGuide(id: string): Promise<{ ok: true } | { ok: false; text: string }>` (janela aberta no clique, conteúdo por `blob:`, revogado em 60 s, `POPUP_BLOCKED = "o navegador bloqueou a janela nova — permita janelas deste site para ver a guia"`); `ConsultationViewProps.regulation?: boolean` (a autora, na consulta finalizada, vê a peça em modo `addendum` com "Guia em PDF"); o `ConsultationWorkspace` passa `regulation` ao editor e à leitura quando `canRequestRegulation(user) && allowedTypes(consultation.cbo_code).length > 0`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/ConsultationWorkspace.regulation.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn(), startConsultation: vi.fn(),
    getConsultation: vi.fn(), getConsultationProblems: vi.fn(), listConsultationRegulationRequests: vi.fn(), fetchRegulationGuide: vi.fn() };
});
vi.mock("./ConsultationEditor", () => ({ ConsultationEditor: (p: { regulation?: boolean }) => <span>{`editor ${p.regulation ? "com" : "sem"} regulação`}</span> }));

import * as api from "../../lib/api";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { consultation, finalized, options, problem, record } from "../../test/consultationFixtures";
import { regRequest } from "../../test/regulationFixtures";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

describe("ConsultationWorkspace — regulação (módulo 26a)", () => {
  beforeEach(() => {
    mocked(api.getAttendanceRecord).mockResolvedValue(record());
    mocked(api.getConsultationOptions).mockResolvedValue(options());
    mocked(api.getConsultationProblems).mockResolvedValue([ problem() ]);
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([ regRequest({ status: "pending" }) ]);
  });
  const open = async (features: string[], cbo = "225142", started = consultation({ cbo_code: cbo })) => {
    mocked(api.startConsultation).mockResolvedValue(started);
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features }));
    renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={() => undefined} onFinalized={() => undefined} />);
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
  };

  it.each([
    [ [ "clinical_record", "care_regulation" ], "225142", "com" ], [ [ "clinical_record", "care_regulation" ], "223505", "com" ],
    [ [ "clinical_record", "care_regulation" ], "223208", "com" ], [ [ "clinical_record", "care_regulation" ], "322205", "sem" ],
    [ [ "clinical_record" ], "225142", "sem" ]
  ])("features %j, CBO %s: editor %s regulação", async (features, cbo, expected) => {
    await open(features, cbo);
    expect(await screen.findByText(`editor ${expected} regulação`)).not.toBeNull();
  });

  it("consulta finalizada da autora: pedidos com situação, aviso do adendo e a guia por blob", async () => {
    const win = { location: { href: "" }, close: vi.fn(), opener: {} };
    vi.stubGlobal("open", vi.fn(() => win));
    class FakeURL extends URL {}
    Object.assign(FakeURL, { createObjectURL: vi.fn(() => "blob:http://curitiba.localhost/g1"), revokeObjectURL: vi.fn() });
    vi.stubGlobal("URL", FakeURL);
    mocked(api.fetchRegulationGuide).mockResolvedValue({ blob: new Blob([ "%PDF" ]), filename: "guia-regulacao-rr1.pdf" });
    await open([ "clinical_record", "care_regulation" ], "225142", finalized({ author: { id: "us1", name: "Dra. Helena Prado" } }));
    expect(await screen.findByText(/vão para a regulação quando você registrar um adendo/)).not.toBeNull();
    fireEvent.click(await screen.findByRole("button", { name: "Guia em PDF" }));
    await waitFor(() => expect(win.location.href).toBe("blob:http://curitiba.localhost/g1"));
    expect(mocked(api.fetchRegulationGuide)).toHaveBeenCalledWith("rr1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/consultation/ConsultationWorkspace.regulation.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

`openGuide` no molde do `openExamPdf` (sem link vencido: 404 → "guia não encontrada"; outras falhas → `regulationError`). No `ConsultationView`, com `regulation && isAuthor && c.status === "finalized"`, a peça em `addendum` (cbo da consulta, `cid10Allowed` de `options?.cid10_allowed_for_cbo ?? false`) com `onOpenGuide` que chama `openGuide` e mostra a falha na própria peça. No `Workspace`, `const regulationOn = canRequestRegulation(user) && allowedTypes(consultation?.cbo_code).length > 0` passado ao editor e ao `ConsultationView` (o `ConsultationLoader` das consultas antigas não recebe: lá é só leitura).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/consultation && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/regulation/openGuide.ts src/modules/consultation/ConsultationView.tsx src/modules/consultation/ConsultationWorkspace.tsx \
  src/modules/consultation/ConsultationWorkspace.regulation.test.tsx
$G commit -m "feat: add regulation requests by addendum and the referral guide

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Fila regulada — abas, filtros, cor, prazo e espera (F-26.3)

**Files:**
- Create: `src/modules/regulation/RequestTable.tsx`
- Modify: `src/modules/RegulationQueue.tsx` (o corpo)
- Test: `src/modules/RegulationQueue.test.tsx`

**Interfaces:**
- Consumes: `listRegulationRequests`, `RegulationQuery` (Task 1); `useRegulationCount` (Task 3); `TargetSearch` (Task 4); `REG_LIST_KEY`, `NOMINAL_QUERY`, `colorLabel`, `colorTone`, `statusLabel`, `waitDays`, `waitText`, `dueText`, `COLORS`, `regulationError` (Task 2); `RequestDetail` (Task 8 — até lá, o clique em "Abrir" guarda o id e mostra só "carregando…").
- Produces: `RegulationQueue({ today?: string })`; `QUEUE_TABS` (`pending` "Aguardando" — com dois blocos, "Vencidos" no topo (`overdue: true`) e "No prazo" (`overdue: false`) —, `returned` "Devolvidos", `authorized` "Autorizados", `scheduled` "Agendados", `external` "Pelo estado", `finished` "Encerrados" com o select "Situação" — `completed`, `denied`, `cancelled`, `closed`); `RequestTable({ rows, today, onOpen(r) })` (colunas Cor, Encaminhamento [`card: "title"`], Paciente, Solicitante, Pedido em, Espera, Prazo, ação [`card: "actions"`]); nenhuma coluna de posição.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/RegulationQueue.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { focusManager, onlineManager } from "@tanstack/react-query";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listRegulationRequests: vi.fn(), countRegulationRequests: vi.fn(), getRegulationRequest: vi.fn(), searchRegulationTargets: vi.fn(),
  listCareOffers: vi.fn(), fetchStateExportCsv: vi.fn() }));
import * as api from "../lib/api";
import type { RegulationQuery } from "../lib/api";
import { RegulationQueue } from "./RegulationQueue";
import { REGULATION_DISABLED, REGULATION_NO_ROLE } from "../lib/regulation";
import { renderWithProviders } from "../test/campaignFixtures";
import { CARDIO, TODAY26, regCounts, regRequest, regUser } from "../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const queries = () => mocked(api.listRegulationRequests).mock.calls.map((c) => c[0] as RegulationQuery);

describe("RegulationQueue", () => {
  const OVERDUE = regRequest({ id: "rr2", color: "red", suggested_color: "yellow", due_on: "2026-10-25", overdue: true,
    requested_at: "2026-10-18T09:00:00-03:00", patient: { id: "pa2", display_name: "Marcos Souza", has_cpf: false, cns_masked: "*** **** **** 4321" } });
  beforeEach(() => {
    for (const fn of [ api.listRegulationRequests, api.countRegulationRequests, api.searchRegulationTargets ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
    mocked(api.countRegulationRequests).mockResolvedValue(regCounts());
    mocked(api.searchRegulationTargets).mockResolvedValue([ CARDIO ]);
    // A ordem do api é cor e data (R8): os vencidos vêm de uma leitura própria e ficam no topo.
    mocked(api.listRegulationRequests).mockImplementation(async (q: RegulationQuery) =>
      (q.overdue === true ? { requests: [ OVERDUE ], next_before: null } : { requests: [ regRequest() ], next_before: "c1" }));
  });

  it("abas com contagens, aviso e bloco dos vencidos no topo; cor, espera e prazo; sem posição", async () => {
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    expect((await screen.findByRole("status")).textContent).toBe("3 pedidos vencidos");
    expect(screen.getByRole("tab", { name: "Aguardando (12)" }).getAttribute("aria-selected")).toBe("true");
    for (const name of [ "Devolvidos (2)", "Autorizados (5)", "Agendados", "Pelo estado (1)", "Encerrados" ])
      expect(screen.getByRole("tab", { name })).not.toBeNull();
    const late = await screen.findByRole("region", { name: "Vencidos" });
    const onTime = screen.getByRole("region", { name: "No prazo" });
    expect(late.compareDocumentPosition(onTime) & Node.DOCUMENT_POSITION_FOLLOWING).toBeTruthy();
    const lateRow = await within(late).findByRole("row", { name: /Marcos Souza/ });
    expect(within(lateRow).getByText("vermelho")).not.toBeNull();
    expect(within(lateRow).getByText("vencido há 3 dias")).not.toBeNull();
    expect(within(lateRow).getByText("10 dias")).not.toBeNull();
    const row = await within(onTime).findByRole("row", { name: /Joana Lima/ });
    expect(within(row).getByText("amarelo")).not.toBeNull();
    expect(within(row).getByText("até 07/11")).not.toBeNull();
    expect(within(row).getByText("20 dias")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/posição|position|#\d/i);
    expect(queries()).toContainEqual({ status: "pending", overdue: true, type: null, targetCode: null, color: null });
    expect(queries()).toContainEqual({ status: "pending", overdue: false, type: null, targetCode: null, color: null });
  });

  it("filtros e abas: o que vai ao api", async () => {
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    await screen.findByRole("region", { name: "Vencidos" });
    fireEvent.change(screen.getByLabelText("Cor"), { target: { value: "red" } });
    fireEvent.change(screen.getByLabelText("Especialidade ou procedimento"), { target: { value: "cardio" } });
    fireEvent.click(await screen.findByRole("button", { name: /225120 — Cardiologia/ }));
    await waitFor(() => expect(queries()).toContainEqual({ status: "pending", overdue: true, type: "specialty", targetCode: "225120", color: "red" }));
    expect(queries()).toContainEqual({ status: "pending", overdue: false, type: "specialty", targetCode: "225120", color: "red" });
    fireEvent.click(screen.getByRole("tab", { name: "Devolvidos (2)" }));
    await waitFor(() => expect(queries().at(-1)).toEqual({ status: "returned", type: "specialty", targetCode: "225120", color: "red" }));
    expect(screen.queryByRole("region", { name: "Vencidos" })).toBeNull();
    fireEvent.click(screen.getByRole("tab", { name: "Encerrados" }));
    fireEvent.change(screen.getByLabelText("Situação"), { target: { value: "denied" } });
    await waitFor(() => expect(queries().at(-1)).toMatchObject({ status: "denied" }));
  });

  it("Carregar mais usa o cursor da lista no prazo", async () => {
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    fireEvent.click(await within(await screen.findByRole("region", { name: "No prazo" })).findByRole("button", { name: "Carregar mais" }));
    await waitFor(() => expect(queries().at(-1)).toMatchObject({ status: "pending", overdue: false, before: "c1" }));
  });

  it("volta à aba e reconecta: a lista não relê", async () => {
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    await screen.findAllByRole("row", { name: /Cardiologia/ });
    const before = mocked(api.listRegulationRequests).mock.calls.length;
    act(() => { focusManager.setFocused(false); focusManager.setFocused(true); onlineManager.setOnline(false); onlineManager.setOnline(true); });
    await new Promise((r) => setTimeout(r, 20));
    expect(mocked(api.listRegulationRequests).mock.calls.length).toBe(before);
  });

  it("portão: interruptor e papel", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "care_regulator" ], { features: [] }));
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    expect(await screen.findByText(REGULATION_DISABLED)).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "health_professional" ]));
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    expect(await screen.findByText(REGULATION_NO_ROLE)).not.toBeNull();
    expect(mocked(api.listRegulationRequests)).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationQueue.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

`RegulationQueue` no molde do `SurveillanceQueue` (portão, `role="tablist"`, aviso `role="status"` "N pedido(s) vencido(s)" a partir de `useRegulationCount` — o mesmo hook do menu, mesma chave, uma leitura por minuto). A lista é `useInfiniteQuery` com `queryKey [REG_LIST_KEY, query]`, `...NOMINAL_QUERY` e `getNextPageParam: (p) => p.next_before ?? undefined`; a consulta é `{ ...aba, type, targetCode, color }` com nulos; na aba "Aguardando" são duas listas, cada uma com o seu cursor e o seu "Carregar mais": `{ status: "pending", overdue: true, … }` no `<section aria-label="Vencidos">` (só aparece se houver) e `{ status: "pending", overdue: false, … }` no `<section aria-label="No prazo">` — é assim que os vencidos ficam no topo sem o api mudar a ordem (D6). O filtro de alvo usa o `TargetSearch` sem `consultationId` (tipo pelo select "Tipo", padrão "specialty"; "Limpar" volta a nulo). Dentro de cada lista, a ordem é a do api (cor, depois data — R8): a tela não reordena. Paciente sem nome → "pessoa sem nome no cadastro". Clicar "Abrir" mostra o detalhe (Task 8) no lugar da lista, com "Voltar à fila" que relê a página atual.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationQueue.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/RegulationQueue.tsx src/modules/regulation/RequestTable.tsx src/modules/RegulationQueue.test.tsx
$G commit -m "feat: add the regulated queue by colour and waiting time

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Detalhe e decisões — autorizar, devolver, negar, cancelar (F-26.3)

**Files:**
- Create: `src/modules/regulation/RequestDetail.tsx`, `src/modules/regulation/AuthorizeForm.tsx`, `src/modules/regulation/ReasonAction.tsx`
- Modify: `src/modules/RegulationQueue.tsx` (abre o detalhe)
- Test: `src/modules/regulation/RequestDetail.test.tsx`

**Interfaces:**
- Consumes: `getRegulationRequest`, `listCareOffers`, `authorizeRegulationRequest`, `returnRegulationRequest`, `denyRegulationRequest`, `cancelRegulationRequest` (Task 1); `actionsFor`, `currentMonth`, `COLORS`, `reasonProblem`, `regulationError`, `STALE_TEXT`, `QUOTA_EXHAUSTED_TEXT`, `SKIP_REASON_TEXT`, `REG_DETAIL_KEY`, `REG_OFFERS_KEY`, `NOMINAL_QUERY` (Task 2); `openGuide` (Task 6); `patientDocument` (`src/lib/patientDocument.ts`).
- Produces: `RequestDetail({ id, today, onBack(), onChanged(text: string) })` — `region` com o nome do alvo; `KeyValue` de paciente (documento por `patientDocument`), tipo, alvo, CID-10, justificativa, cor sugerida e confirmada (com `color_reason`), situação, solicitante e unidade, pedido em, espera, prazo, faltas, número externo, autorização (oferta, prestador, mês, `skip_reason`), agendamento; o **andamento** (`events`); "Guia em PDF"; as ações de `actionsFor`; um slot `extra?: ReactNode` que as Tasks 12 e 13 preenchem. `AuthorizeForm({ request, today, onDone(text), onStale(text), onCancel })`. `ReasonAction({ label, submitLabel, onSubmit(reason): Promise<void>, onCancel })` (10–500, recusa sem cortar; erro pelo `regulationError`).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/regulation/RequestDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  getRegulationRequest: vi.fn(), listCareOffers: vi.fn(), authorizeRegulationRequest: vi.fn(), returnRegulationRequest: vi.fn(),
  denyRegulationRequest: vi.fn(), cancelRegulationRequest: vi.fn(), fetchRegulationGuide: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { RequestDetail } from "./RequestDetail";
import { QUOTA_EXHAUSTED_TEXT, SKIP_REASON_TEXT, STALE_TEXT } from "../../lib/regulation";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY26, offer, regRequest, regUser } from "../../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const OWN = offer();
const FULL = offer({ id: "of2", provider: { id: "pr2", name: "Clínica do Coração", kind: "contracted" }, used: 6, quantity: 6, available: 0 });
const CONTRACTED = offer({ id: "of3", provider: { id: "pr3", name: "Consórcio Leste", kind: "consortium" }, available: 4 });

describe("RequestDetail", () => {
  let onChanged: ReturnType<typeof vi.fn>;
  beforeEach(() => {
    for (const fn of [ api.getRegulationRequest, api.listCareOffers, api.authorizeRegulationRequest, api.returnRegulationRequest,
      api.denyRegulationRequest, api.cancelRegulationRequest ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
    mocked(api.getRegulationRequest).mockResolvedValue(regRequest());
    mocked(api.listCareOffers).mockResolvedValue([ OWN, FULL, CONTRACTED ]);
    mocked(api.authorizeRegulationRequest).mockResolvedValue(regRequest({ status: "authorized" }));
    onChanged = vi.fn();
  });
  const mount = () => renderWithProviders(<RequestDetail id="rr1" today={TODAY26} onBack={vi.fn()} onChanged={onChanged} />);
  const openAuthorize = async () => {
    fireEvent.click(await screen.findByRole("button", { name: "Autorizar" }));
    return screen.findByRole("form", { name: "Autorizar o pedido" });
  };

  it("mostra o pedido com andamento; as ofertas do mês e do alvo, esgotada desabilitada", async () => {
    mount();
    const detail = await screen.findByRole("region", { name: "Cardiologia" });
    for (const t of [ "Joana Lima", "I209", "Dor torácica aos esforços há 3 meses, teste ergométrico alterado.", "Dra. Helena Prado", "UBS Centro", "20 dias", "até 07/11" ])
      expect(within(detail).getAllByText(t).length).toBeGreaterThan(0);
    expect(within(screen.getByRole("list", { name: "Andamento" })).getByText(/aguardando regulação/)).not.toBeNull();
    const form = await openAuthorize();
    expect(mocked(api.listCareOffers)).toHaveBeenCalledWith({ month: "2026-10", targetCode: "225120" });
    expect((within(form).getByLabelText(/Clínica do Coração/) as HTMLInputElement).disabled).toBe(true);
    expect(within(form).getByText("esgotada")).not.toBeNull();
    expect(within(form).getByLabelText(/Ambulatório de Especialidades · rede própria · saldo 2 de 10/)).not.toBeNull();
  });

  it("autorizar na rede própria: corpo com a versão e a frase do pedido de agendamento", async () => {
    mount();
    const form = await openAuthorize();
    fireEvent.click(within(form).getByLabelText(/Ambulatório de Especialidades/));
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar autorização" }));
    await waitFor(() => expect(mocked(api.authorizeRegulationRequest)).toHaveBeenCalledWith("rr1", { offer_id: "of1", version: 3 }));
    expect(onChanged).toHaveBeenCalledWith("Autorizado em Ambulatório de Especialidades. A unidade vai marcar o horário pelo pedido de agendamento.");
  });

  it("mudar a cor pede motivo antes; 422 skip_reason_required pede o motivo e reenvia com ele", async () => {
    mount();
    const form = await openAuthorize();
    fireEvent.click(within(form).getByLabelText(/Consórcio Leste/));
    text("Cor confirmada", "red");
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar autorização" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("ao mudar a cor, informe o motivo");
    expect(mocked(api.authorizeRegulationRequest)).not.toHaveBeenCalled();
    text("Motivo da mudança de cor", "Piora da dor em repouso relatada hoje");
    mocked(api.authorizeRegulationRequest).mockRejectedValueOnce(new ApiError(422, { error: "skip_reason_required" }, "x"));
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar autorização" }));
    expect(await within(form).findByText(SKIP_REASON_TEXT)).not.toBeNull();
    expect((within(form).getByLabelText(/Consórcio Leste/) as HTMLInputElement).checked).toBe(true);
    text("Motivo para autorizar fora da ordem", "Risco maior que o dos pedidos mais antigos");
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar autorização" }));
    await waitFor(() => expect(mocked(api.authorizeRegulationRequest)).toHaveBeenLastCalledWith("rr1", { offer_id: "of3", color: "red",
      color_reason: "Piora da dor em repouso relatada hoje", skip_reason: "Risco maior que o dos pedidos mais antigos", version: 3 }));
    expect(onChanged.mock.calls[0][0]).toBe("Autorizado em Consórcio Leste. Registre a data e o local quando o prestador marcar.");
  });

  it("409 quota_exhausted: frase, ofertas relidas, oferta limpa, o resto fica", async () => {
    mount();
    const form = await openAuthorize();
    fireEvent.click(within(form).getByLabelText(/Consórcio Leste/));
    text("Cor confirmada", "red"); text("Motivo da mudança de cor", "Piora da dor em repouso relatada hoje");
    mocked(api.authorizeRegulationRequest).mockRejectedValueOnce(new ApiError(409, { error: "quota_exhausted" }, "x"));
    mocked(api.listCareOffers).mockResolvedValue([ OWN, FULL, { ...CONTRACTED, used: 10, available: 0 } ]);
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar autorização" }));
    expect((await within(form).findByRole("alert")).textContent).toBe(QUOTA_EXHAUSTED_TEXT);
    await waitFor(() => expect(mocked(api.listCareOffers)).toHaveBeenCalledTimes(2));
    await waitFor(() => expect((within(form).getByLabelText(/Consórcio Leste/) as HTMLInputElement).disabled).toBe(true));
    expect((within(form).getByLabelText(/Consórcio Leste/) as HTMLInputElement).checked).toBe(false);
    expect((screen.getByLabelText("Motivo da mudança de cor") as HTMLTextAreaElement).value).toBe("Piora da dor em repouso relatada hoje");
    expect(mocked(api.authorizeRegulationRequest)).toHaveBeenCalledTimes(1);
  });

  it("409 stale_version: frase, nada reenviado, Recarregar relê", async () => {
    mount();
    const form = await openAuthorize();
    fireEvent.click(within(form).getByLabelText(/Ambulatório de Especialidades/));
    mocked(api.authorizeRegulationRequest).mockRejectedValueOnce(new ApiError(409, { error: "stale_version" }, "x"));
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar autorização" }));
    expect((await screen.findByRole("alert")).textContent).toBe(STALE_TEXT);
    fireEvent.click(screen.getByRole("button", { name: "Recarregar" }));
    await waitFor(() => expect(mocked(api.getRegulationRequest)).toHaveBeenCalledTimes(2));
    expect(mocked(api.authorizeRegulationRequest)).toHaveBeenCalledTimes(1);
  });

  it("devolver, negar e cancelar: motivo de 10 a 500, só no corpo", async () => {
    mocked(api.returnRegulationRequest).mockResolvedValue(regRequest({ status: "returned" }));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Devolver para complementar" }));
    text("Motivo da devolução", "falta");
    fireEvent.click(screen.getByRole("button", { name: "Devolver" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o motivo precisa de 10 a 500 caracteres");
    text("Motivo da devolução", "Falta o resultado do teste ergométrico");
    fireEvent.click(screen.getByRole("button", { name: "Devolver" }));
    await waitFor(() => expect(mocked(api.returnRegulationRequest)).toHaveBeenCalledWith("rr1", "Falta o resultado do teste ergométrico", 3));
    expect(onChanged).toHaveBeenCalledWith("Pedido devolvido à unidade solicitante para complementar.");
  });

  it("ações por situação: autorizado não nega nem autoriza", async () => {
    mocked(api.getRegulationRequest).mockResolvedValue(regRequest({ status: "authorized", authorization: { offer_id: "of1",
      provider: { id: "pr1", name: "Ambulatório de Especialidades", kind: "own" }, month: "2026-10", skip_reason: null,
      at: "2026-10-20T10:00:00-03:00", by: { name: "Regulador" } } }));
    mount();
    await screen.findByRole("button", { name: "Cancelar o pedido" });
    expect(screen.queryByRole("button", { name: "Autorizar" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Negar" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation/RequestDetail.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `RequestDetail`: `useQuery([REG_DETAIL_KEY, id], getRegulationRequest, NOMINAL_QUERY)`; `refetch` no "Recarregar" e depois de cada ação; botões pelo `actionsFor`: "Autorizar", "Devolver para complementar", "Negar", "Cancelar o pedido" (rótulos dos `ReasonAction`: "Motivo da devolução"/"Devolver", "Motivo da negativa"/"Negar pedido", "Motivo do cancelamento"/"Cancelar pedido"; frases de sucesso: "Pedido devolvido à unidade solicitante para complementar.", "Pedido negado.", "Pedido cancelado." e, se havia autorização sem desfecho, "Pedido cancelado; a vaga volta para a cota do mês."). `stale_version` e `invalid_transition` em qualquer ação: `STALE_TEXT`/frase, o formulário aberto fica, "Recarregar". O detalhe nunca mostra posição.
- `AuthorizeForm` (`<form aria-label="Autorizar o pedido">`): ofertas `listCareOffers({ month: currentMonth(today), targetCode: request.target.code })` com `NOMINAL_QUERY`, como rádios "<prestador> · <tipo do prestador> · saldo <available> de <quantity>" (`available <= 0` → desabilitada, com `Tag` "esgotada"); nenhuma oferta → "nenhuma oferta com saldo para este alvo neste mês — cadastre em Prestadores e cotas" e o botão desabilitado; "Cor confirmada" (padrão `request.color`); "Motivo da mudança de cor" aparece quando a cor muda (10–500); "Motivo para autorizar fora da ordem" aparece só depois do 422 `skip_reason_required` (com `SKIP_REASON_TEXT` acima). Corpo: `offer_id`, `version` e só o que existe (`color` e `color_reason` quando mudou; `skip_reason` quando pedido). 409 `quota_exhausted` → `QUOTA_EXHAUSTED_TEXT`, `refetch` das ofertas, oferta limpa. Frases de sucesso: `own` → "Autorizado em <prestador>. A unidade vai marcar o horário pelo pedido de agendamento."; outro → "Autorizado em <prestador>. Registre a data e o local quando o prestador marcar.".
- Na fila, "Abrir" mostra o `RequestDetail`; `onChanged(text)` volta à fila com a frase (`role="status"`) e relê a página e a contagem.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation src/modules/RegulationQueue.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/regulation/RequestDetail.tsx src/modules/regulation/AuthorizeForm.tsx src/modules/regulation/ReasonAction.tsx \
  src/modules/regulation/RequestDetail.test.tsx src/modules/RegulationQueue.tsx
$G commit -m "feat: add regulator decisions with quota-backed authorization

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Devolvidos da unidade e complemento (F-26.3)

**Files:**
- Modify: `src/modules/RegulationReturned.tsx` (o corpo)
- Test: `src/modules/RegulationReturned.test.tsx`

**Interfaces:**
- Consumes: `getMyProfessional` (existente), `listReturnedRequests`, `complementRegulationRequest` (Task 1); `RequestForm` em `complement` (Task 4); `REG_RETURNED_KEY`, `NOMINAL_QUERY`, `STALE_TEXT`, `regulationError`, `draftFromRequest`, `requestProblem` (Task 2).
- Produces: `RegulationReturned()` — "Unidade" (os vínculos ativos de `getMyProfessional`; um só → sem select); a lista dos devolvidos (paciente, alvo, cor, pedido em, **motivo da devolução** — o `reason` do último evento `returned`, D8); "Complementar" abre o `RequestForm` (`mode="complement"`, justificativa atual preenchida, `cbo` do próprio vínculo); envio `complementRegulationRequest(id, { justification, cid10?, suggested_color, version })`; sucesso "Pedido complementado: voltou para a fila da regulação, com a data do pedido original." e relê.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/RegulationReturned.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  getMyProfessional: vi.fn(), listReturnedRequests: vi.fn(), complementRegulationRequest: vi.fn(), searchRegulationTargets: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { RegulationReturned } from "./RegulationReturned";
import { STALE_TEXT } from "../lib/regulation";
import { renderWithProviders } from "../test/campaignFixtures";
import { regRequest, regUser } from "../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const link = (id: string, unit: string, name: string, ended: string | null = null) => ({ id, health_unit_id: unit, unit_name: name, cbo_code: "225142",
  cbo_title: "Médico clínico", started_at: "2026-01-01", started_by: "adm", ended_at: ended, ended_by: null });
const RETURNED = regRequest({ status: "returned", version: 4, events: [
  { status: "pending", at: "2026-10-08T10:00:00-03:00", by: { name: "Dra. Helena Prado" }, reason: null },
  { status: "returned", at: "2026-10-20T15:00:00-03:00", by: { name: "Regulador" }, reason: "Falta o resultado do teste ergométrico" } ] });

describe("RegulationReturned", () => {
  beforeEach(() => {
    for (const fn of [ api.listReturnedRequests, api.complementRegulationRequest ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "health_professional" ]));
    mocked(api.getMyProfessional).mockResolvedValue({ professional: { id: "p1" }, shifts: [],
      links: [ link("l1", "u1", "UBS Centro"), link("l2", "u3", "UBS Vila"), link("l3", "u9", "UBS Antiga", "2026-05-01") ] });
    mocked(api.listReturnedRequests).mockResolvedValue([ RETURNED ]);
    mocked(api.complementRegulationRequest).mockResolvedValue(regRequest({ status: "pending", version: 5 }));
  });

  it("escolhe a unidade entre os vínculos ativos e mostra o motivo da devolução", async () => {
    renderWithProviders(<RegulationReturned />);
    const select = await screen.findByLabelText("Unidade");
    expect(Array.from((select as HTMLSelectElement).options).map((o) => o.textContent)).toEqual([ "UBS Centro", "UBS Vila" ]);
    await waitFor(() => expect(mocked(api.listReturnedRequests)).toHaveBeenCalledWith("u1"));
    const row = await screen.findByRole("listitem", { name: "Cardiologia de Joana Lima" });
    expect(within(row).getByText("Falta o resultado do teste ergométrico")).not.toBeNull();
    fireEvent.change(select, { target: { value: "u3" } });
    await waitFor(() => expect(mocked(api.listReturnedRequests)).toHaveBeenLastCalledWith("u3"));
  });

  it("complementar: justificativa nova, cor e versão; volta à fila com a data original", async () => {
    renderWithProviders(<RegulationReturned />);
    fireEvent.click(await screen.findByRole("button", { name: "Complementar" }));
    const field = screen.getByLabelText("Justificativa") as HTMLTextAreaElement;
    expect(field.value).toBe(RETURNED.justification);
    const next = `${RETURNED.justification} Teste ergométrico de 15/10: infradesnivelamento de ST.`;
    fireEvent.change(field, { target: { value: next } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar complemento" }));
    await waitFor(() => expect(mocked(api.complementRegulationRequest)).toHaveBeenCalledWith("rr1",
      { justification: next, cid10: "I209", suggested_color: "yellow", version: 4 }));
    expect(await screen.findByText("Pedido complementado: voltou para a fila da regulação, com a data do pedido original.")).not.toBeNull();
    expect(mocked(api.listReturnedRequests)).toHaveBeenCalledTimes(2);
  });

  it("409 stale_version: frase e o texto fica", async () => {
    mocked(api.complementRegulationRequest).mockRejectedValue(new ApiError(409, { error: "stale_version" }, "x"));
    renderWithProviders(<RegulationReturned />);
    fireEvent.click(await screen.findByRole("button", { name: "Complementar" }));
    fireEvent.change(screen.getByLabelText("Justificativa"), { target: { value: "Texto novo do complemento, com o exame." } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar complemento" }));
    expect((await screen.findByRole("alert")).textContent).toBe(STALE_TEXT);
    expect((screen.getByLabelText("Justificativa") as HTMLTextAreaElement).value).toBe("Texto novo do complemento, com o exame.");
  });

  it("sem devolvidos: estado vazio", async () => {
    mocked(api.listReturnedRequests).mockResolvedValue([]);
    renderWithProviders(<RegulationReturned />);
    expect(await screen.findByText("nenhum pedido devolvido para esta unidade")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationReturned.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** a tela com o `RegulationGate` (`allow: professional`), a leitura dos vínculos (`STATIC_QUERY`), a lista com `NOMINAL_QUERY` (chave `[REG_RETURNED_KEY, unitId]`) e o `RequestForm` em `complement` com `submitLabel="Enviar complemento"`; `cid10` só vai se o pedido tinha CID ou se foi digitado.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationReturned.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/RegulationReturned.tsx src/modules/RegulationReturned.test.tsx
$G commit -m "feat: let the requesting unit complement returned requests

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Prestadores, ofertas e cotas mensais com saldo (F-26.4)

**Files:**
- Create: `src/modules/regulation/ProviderForm.tsx`, `src/modules/regulation/OfferForm.tsx`
- Modify: `src/modules/RegulationOffers.tsx` (o corpo)
- Test: `src/modules/RegulationOffers.test.tsx`

**Interfaces:**
- Consumes: `listCareProviders`, `createCareProvider`, `updateCareProvider`, `listCareOffers`, `createCareOffer`, `updateCareOffer`, `listAllUnits` (Task 1 e existente); `TargetSearch` (Task 4); `providerKindLabel`, `cnesProblem`, `providerNameProblem`, `quantityProblem`, `parseQuantity`, `monthProblem`, `currentMonth`, `monthLabel`, `regulationError`, `REG_PROVIDERS_KEY`, `REG_OFFERS_KEY` (Task 2); `SensitiveAction`.
- Produces: `RegulationOffers({ today?: string; onGoToSecurity?(): void })` — painel "Prestadores" (nome, CNES, tipo, unidade da rede, ativo) e painel "Ofertas e cotas de <mês>" (seletor de mês, padrão o corrente; por oferta: prestador, tipo, alvo, quantidade, usadas, saldo; `Tag` "esgotada" com saldo 0 e o aviso `role="status"` "N oferta(s) esgotada(s) neste mês"); `municipal_admin` cria e edita (step-up); `care_regulator` só lê.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/RegulationOffers.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listCareProviders: vi.fn(), createCareProvider: vi.fn(), updateCareProvider: vi.fn(), listCareOffers: vi.fn(), createCareOffer: vi.fn(),
  updateCareOffer: vi.fn(), listAllUnits: vi.fn(), searchRegulationTargets: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { RegulationOffers } from "./RegulationOffers";
import { renderWithProviders } from "../test/campaignFixtures";
import { AMB_ESPEC, CARDIO, TODAY26, offer, provider, regUser } from "../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("RegulationOffers", () => {
  beforeEach(() => {
    for (const fn of [ api.createCareProvider, api.createCareOffer, api.updateCareOffer, api.listCareOffers ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "municipal_admin" ]));
    mocked(api.listCareProviders).mockResolvedValue([ provider(), provider({ id: "pr2", name: "Clínica do Coração", cnes: "7654321", kind: "contracted", health_unit: null }) ]);
    mocked(api.listAllUnits).mockResolvedValue([ { ...AMB_ESPEC, kind: "other", active: true } ]);
    mocked(api.listCareOffers).mockResolvedValue([ offer(), offer({ id: "of2", provider: { id: "pr2", name: "Clínica do Coração", kind: "contracted" },
      quantity: 6, used: 6, available: 0 }) ]);
    mocked(api.searchRegulationTargets).mockResolvedValue([ CARDIO ]);
    mocked(api.createCareProvider).mockResolvedValue(provider());
    mocked(api.createCareOffer).mockResolvedValue(offer());
  });

  it("ofertas do mês corrente com usadas, saldo e esgotada", async () => {
    renderWithProviders(<RegulationOffers today={TODAY26} />);
    expect(await screen.findByText("Ofertas e cotas de outubro de 2026")).not.toBeNull();
    expect(mocked(api.listCareOffers)).toHaveBeenCalledWith({ month: "2026-10" });
    const full = screen.getByRole("row", { name: /Clínica do Coração/ });
    expect(within(full).getByText("esgotada")).not.toBeNull();
    expect(within(screen.getByRole("row", { name: /Ambulatório de Especialidades.*Cardiologia/ })).getByText("2")).not.toBeNull();
    expect(screen.getByRole("status").textContent).toBe("1 oferta esgotada neste mês");
  });

  it("novo prestador: CNES conferido antes; na rede própria, a unidade no lugar do CNES (R17); step-up e o corpo", async () => {
    renderWithProviders(<RegulationOffers today={TODAY26} />);
    fireEvent.click(await screen.findByRole("button", { name: "Novo prestador" }));
    text("Nome", "Clínica do Coração"); text("Tipo", "contracted"); text("CNES", "765432");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("CNES tem 7 dígitos");
    text("CNES", "7654321");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.createCareProvider)).toHaveBeenCalledWith({ name: "Clínica do Coração", cnes: "7654321",
      kind: "contracted", active: true }));
    fireEvent.click(await screen.findByRole("button", { name: "Novo prestador" }));
    text("Nome", "Ambulatório de Especialidades"); text("Tipo", "own");
    expect(screen.queryByLabelText("CNES")).toBeNull();
    text("Unidade da rede", "u2");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.createCareProvider)).toHaveBeenLastCalledWith({ name: "Ambulatório de Especialidades", kind: "own",
      health_unit_id: "u2", active: true }));
  });

  it("nova oferta: alvo pela busca, mês e quantidade inteira; 10,5 recusado", async () => {
    renderWithProviders(<RegulationOffers today={TODAY26} />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova oferta" }));
    text("Prestador", "pr2"); text("Mês", "2026-11"); text("Quantidade no mês", "10,5");
    text("Especialidade ou procedimento", "cardio");
    fireEvent.click(await screen.findByRole("button", { name: /225120 — Cardiologia/ }));
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("quantidade é um número inteiro de 0 a 9999");
    text("Quantidade no mês", "10");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.createCareOffer)).toHaveBeenCalledWith({ provider_id: "pr2", target_type: "specialty",
      target_code: "225120", month: "2026-11", quantity: 10 }));
  });

  it("mudar a quantidade abaixo do usado: 422 quantity_below_used fica no formulário", async () => {
    mocked(api.updateCareOffer).mockRejectedValue(new ApiError(422, { error: "quantity_below_used", used: 8 }, "x"));
    renderWithProviders(<RegulationOffers today={TODAY26} />);
    fireEvent.click(within(await screen.findByRole("row", { name: /Ambulatório de Especialidades.*Cardiologia/ })).getByRole("button", { name: "Mudar quantidade" }));
    text("Quantidade no mês", "5");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    expect(await screen.findByText("a quantidade não pode ficar abaixo do que já foi autorizado no mês (8)")).not.toBeNull();
    expect((screen.getByLabelText("Quantidade no mês") as HTMLInputElement).value).toBe("5");
  });

  it("regulador lê sem botões", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "care_regulator" ]));
    renderWithProviders(<RegulationOffers today={TODAY26} />);
    await screen.findByRole("row", { name: /Clínica do Coração/ });
    expect(screen.queryByRole("button", { name: "Nova oferta" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Novo prestador" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Mudar quantidade" })).toBeNull();
  });
});
```

(O rótulo do botão de confirmar é o `confirmLabel` passado ao `SensitiveAction`; com a sessão de `mfa_verified_at` de agora, ele não pede código. Se o padrão do componente for outro rótulo, use o mesmo nos testes.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationOffers.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** no molde das microáreas do 24a e das doenças locais do 25 (conferência antes, escrita no `SensitiveAction` com `requiresStepUp`): `ProviderForm` (nome 3–120; tipo, que não muda depois de criado (R17); fora de `own`, CNES de 7 dígitos; em `own`, "Unidade da rede" — `listAllUnits` ativas — no lugar do CNES, que o api herda da unidade; "Ativo"); `OfferForm` (prestador ativo, tipo + `TargetSearch` sem consulta, mês `AAAA-MM` com `<input type="month">`, quantidade; em edição só a quantidade). O mês da oferta não pode ser anterior ao corrente ("o mês da oferta não pode ser anterior ao corrente", conferido antes; o api recusa com `invalid_month`). Recusas do api (`duplicate_offer`, `cnes_taken`, `unit_taken`, `quantity_below_used`, `invalid_month`, `invalid_request` com `path`) ficam no formulário com os valores.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationOffers.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/RegulationOffers.tsx src/modules/regulation/ProviderForm.tsx src/modules/regulation/OfferForm.tsx src/modules/RegulationOffers.test.tsx
$G commit -m "feat: add care providers and monthly quotas with balance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Prazos por cor e lista do enfermeiro (F-26.1)

**Files:**
- Modify: `src/modules/RegulationSettings.tsx` (o corpo)
- Test: `src/modules/RegulationSettings.test.tsx`

**Interfaces:**
- Consumes: `getRiskDeadlines`, `updateRiskDeadlines`, `listNurseItems`, `addNurseItem`, `removeNurseItem` (Task 1); `TargetSearch` (Task 4); `COLORS`, `deadlinesProblem`, `typeLabel`, `regulationError`, `REG_DEADLINES_KEY`, `REG_NURSE_KEY` (Task 2); `SensitiveAction`.
- Produces: `RegulationSettings({ onGoToSecurity?(): void })` — painel "Prazos por cor" (quatro campos "Vermelho (dias)"… com os valores do api; a nota "o prazo conta da data do pedido; mudar vale para os pedidos novos" (D10)); painel "O que o enfermeiro pode pedir" (lista com tipo, código, nome e "Retirar"; "Acrescentar" com tipo + `TargetSearch`), tudo com step-up.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/RegulationSettings.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  getRiskDeadlines: vi.fn(), updateRiskDeadlines: vi.fn(), listNurseItems: vi.fn(), addNurseItem: vi.fn(), removeNurseItem: vi.fn(),
  searchRegulationTargets: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { RegulationSettings } from "./RegulationSettings";
import { renderWithProviders } from "../test/campaignFixtures";
import { ECO, deadlines, nurseItem, regUser } from "../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("RegulationSettings", () => {
  beforeEach(() => {
    for (const fn of [ api.updateRiskDeadlines, api.addNurseItem, api.removeNurseItem ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser([ "municipal_admin" ]));
    mocked(api.getRiskDeadlines).mockResolvedValue(deadlines());
    mocked(api.listNurseItems).mockResolvedValue([ nurseItem({ target_type: "specialty", target_code: "225275", name: "Ginecologia" }) ]);
    mocked(api.searchRegulationTargets).mockResolvedValue([ ECO ]);
    mocked(api.updateRiskDeadlines).mockImplementation(async (d) => d);
    mocked(api.addNurseItem).mockResolvedValue(nurseItem());
  });

  it("prazos: ordem conferida antes; o corpo com inteiros", async () => {
    renderWithProviders(<RegulationSettings />);
    expect((await screen.findByLabelText("Vermelho (dias)") as HTMLInputElement).value).toBe("7");
    text("Amarelo (dias)", "5");
    fireEvent.click(screen.getByRole("button", { name: "Salvar prazos" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o prazo do amarelo não pode ser menor que o do vermelho");
    text("Amarelo (dias)", "20");
    fireEvent.click(screen.getByRole("button", { name: "Salvar prazos" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.updateRiskDeadlines)).toHaveBeenCalledWith({ red: 7, yellow: 20, green: 90, blue: 180 }));
  });

  it("lista do enfermeiro: acrescentar pela busca, retirar pelo tipo e código; duplicado fica na tela", async () => {
    renderWithProviders(<RegulationSettings />);
    expect(await screen.findByRole("row", { name: /Ginecologia/ })).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar item" }));
    fireEvent.click(screen.getByLabelText("Procedimento"));
    text("Especialidade ou procedimento", "eco");
    fireEvent.click(await screen.findByRole("button", { name: /0205010032 — Ecocardiografia/ }));
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.addNurseItem)).toHaveBeenCalledWith({ target_type: "procedure", target_code: "0205010032" }));
    fireEvent.click(within(screen.getByRole("row", { name: /Ginecologia/ })).getByRole("button", { name: "Retirar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.removeNurseItem)).toHaveBeenCalledWith({ target_type: "specialty", target_code: "225275" }));
    mocked(api.addNurseItem).mockRejectedValue(new ApiError(409, { error: "duplicate_item" }, "x"));
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar item" }));
    text("Especialidade ou procedimento", "eco");
    fireEvent.click(await screen.findByRole("button", { name: /0205010032 — Ecocardiografia/ }));
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    expect(await screen.findByText("este item já está na lista")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationSettings.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** com o `RegulationGate` (`allow: admin`), `STATIC_QUERY` nas leituras (não são nominais) e as escritas no `SensitiveAction`; depois de cada escrita, relê o painel.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/RegulationSettings.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/RegulationSettings.tsx src/modules/RegulationSettings.test.tsx
$G commit -m "feat: add colour deadlines and the nurse requestable list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Agendamento externo, desfecho e o `regulated` da recepção (F-26.5)

**Files:**
- Create: `src/modules/regulation/ScheduleOutcome.tsx`
- Modify: `src/modules/regulation/RequestDetail.tsx` (slot `extra`), `src/lib/api.ts` (`RequestRow` e `CheckInAppointment`), `src/lib/scheduling.ts`, `src/lib/scheduling.test.ts`
- Test: `src/modules/regulation/ScheduleOutcome.test.tsx`

**Interfaces:**
- Consumes: `scheduleRegulationRequest`, `recordRegulationOutcome` (Task 1); `actionsFor`, `scheduleAtProblem`, `placeProblem`, `notesProblem`, `regulationError`, `STALE_TEXT` (Task 2); `cityLocalIso`, `fmtDateTime` (`format.ts`).
- Produces: `ScheduleOutcome({ request, onChanged(text) })` — "Registrar agendamento" (ou "Remarcar", se já há agendamento): "Data e hora" (`datetime-local`), "Local", "Observação (opcional)"; "Realizado" e "Faltou" (`ConfirmButton`). Na rede própria, sem formulário: "O agendamento é feito pela unidade de destino (pedido de agendamento do módulo 17)." e o id do pedido de agendamento. `RequestRow.kind` e `CheckInAppointment.kind` ganham `"regulated"`; `RequestRow.origin` ganha `"regulation"` (D15); `requestKindLabel` → `"Regulação (pedido de <unidade>)"` ou `"Regulação"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/regulation/ScheduleOutcome.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  scheduleRegulationRequest: vi.fn(), recordRegulationOutcome: vi.fn() }));
import * as api from "../../lib/api";
import { ScheduleOutcome } from "./ScheduleOutcome";
import { renderWithProviders } from "../../test/campaignFixtures";
import { regRequest, regUser } from "../../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const auth = (kind: string) => ({ offer_id: "of3", provider: { id: "pr3", name: "Consórcio Leste", kind }, month: "2026-10", skip_reason: null,
  at: "2026-10-20T10:00:00-03:00", by: { name: "Regulador" } });

describe("ScheduleOutcome", () => {
  let onChanged: ReturnType<typeof vi.fn>;
  beforeEach(() => {
    for (const fn of [ api.scheduleRegulationRequest, api.recordRegulationOutcome ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
    onChanged = vi.fn();
  });

  it("prestador externo: data no fuso da cidade, local e versão; datas e local conferidos antes", async () => {
    mocked(api.scheduleRegulationRequest).mockResolvedValue(regRequest({ status: "scheduled" }));
    renderWithProviders(<ScheduleOutcome request={regRequest({ status: "authorized", authorization: auth("consortium"), version: 5 })} onChanged={onChanged} />);
    fireEvent.click(await screen.findByRole("button", { name: "Registrar agendamento" }));
    text("Data e hora", "2026-11-10T14:30");
    fireEvent.click(screen.getByRole("button", { name: "Salvar agendamento" }));
    expect((await screen.findByRole("alert")).textContent).toBe("informe o local (até 200 caracteres)");
    text("Local", "Consórcio Leste — Av. Brasil, 2000");
    fireEvent.click(screen.getByRole("button", { name: "Salvar agendamento" }));
    await waitFor(() => expect(mocked(api.scheduleRegulationRequest)).toHaveBeenCalledWith("rr1",
      { at: "2026-11-10T14:30:00-03:00", place: "Consórcio Leste — Av. Brasil, 2000", notes: null, version: 5 }));
    expect(onChanged).toHaveBeenCalledWith("Agendamento registrado: o cidadão vê a data e o local no celular.");
  });

  it.each([
    [ "completed", "Realizado", regRequest({ status: "completed" }), "Atendimento realizado registrado." ],
    [ "missed", "Faltou", regRequest({ status: "pending", misses: 1 }), "Falta registrada: o pedido voltou para a fila (primeira falta; a vaga não volta para a cota)." ],
    [ "missed", "Faltou", regRequest({ status: "closed", misses: 2 }), "Segunda falta: o pedido foi encerrado." ]
  ])("desfecho %s pela resposta do api (%#)", async (result, button, after, phrase) => {
    mocked(api.recordRegulationOutcome).mockResolvedValue(after);
    renderWithProviders(<ScheduleOutcome request={regRequest({ status: "scheduled", authorization: auth("contracted"), version: 6,
      schedule: { at: "2026-11-10T14:30:00-03:00", place: "Clínica", notes: null } })} onChanged={onChanged} />);
    fireEvent.click(await screen.findByRole("button", { name: button }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.recordRegulationOutcome)).toHaveBeenCalledWith("rr1", result, 6));
    expect(onChanged).toHaveBeenCalledWith(phrase);
  });

  it("rede própria: segue o 17, sem formulário", async () => {
    renderWithProviders(<ScheduleOutcome request={regRequest({ status: "authorized", authorization: auth("own"), appointment_request_id: "ar9" })} onChanged={onChanged} />);
    expect(await screen.findByText("O agendamento é feito pela unidade de destino (pedido de agendamento do módulo 17).")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Registrar agendamento" })).toBeNull();
    expect(screen.queryByRole("button", { name: "Faltou" })).toBeNull();
  });
});
```

Em `src/lib/scheduling.test.ts`, no describe de `requestKindLabel`:

```ts
    expect(requestKindLabel({ kind: "regulated", origin_unit_name: "UBS Centro" })).toBe("Regulação (pedido de UBS Centro)");
    expect(requestKindLabel({ kind: "regulated", origin_unit_name: null })).toBe("Regulação");
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation/ScheduleOutcome.test.tsx src/lib/scheduling.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente** — `ScheduleOutcome` pelo `actionsFor` (o `RequestDetail` o põe no slot `extra` quando `schedule || outcome || authorization?.provider.kind === "own"`); `at` por `cityLocalIso` (a hora digitada é a da cidade); `notes` vazio vai `null`; frases de desfecho pela resposta (`completed` → realizado; `pending` com `misses: 1` → primeira falta; `closed` → segunda falta; outra → "Desfecho registrado."). `stale_version` → `STALE_TEXT`, formulário fica. Em `api.ts`, as uniões dos `kind`/`origin`; em `scheduling.ts`, o ramo `regulated` antes do `return` final.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation src/lib/scheduling.test.ts src/modules/attendance && cd $W && npx tsc --noEmit`
Expected: PASS (a fila da recepção e o check-in seguem iguais para os outros `kind`).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/regulation/ScheduleOutcome.tsx src/modules/regulation/ScheduleOutcome.test.tsx src/modules/regulation/RequestDetail.tsx \
  src/lib/api.ts src/lib/scheduling.ts src/lib/scheduling.test.ts
$G commit -m "feat: record external schedules and outcomes and label regulated requests

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Número externo e exportação CSV (F-26.6)

**Files:**
- Create: `src/modules/regulation/ExternalNumberForm.tsx`
- Modify: `src/modules/regulation/RequestDetail.tsx` (slot `extra`), `src/modules/RegulationQueue.tsx`, `src/modules/RegulationQueue.test.tsx`
- Test: `src/modules/regulation/ExternalNumberForm.test.tsx`

**Interfaces:**
- Consumes: `setRegulationExternal`, `fetchStateExportCsv` (Task 1); `EXTERNAL_SYSTEMS`, `externalNumberProblem`, `exportFilename`, `regulationError`, `STALE_TEXT` (Task 2); `saveFile` (`src/modules/lab/saveFile.ts`).
- Produces: `ExternalNumberForm({ request, onChanged(text) })` — "Sistema" (SISREG, CARE, CROSS, outro) e "Número no sistema do estado"; sucesso "Número externo registrado."; no detalhe, quando `actionsFor(r).external`. Na fila, o botão "Exportar CSV para o sistema do estado" (o `status` da aba, se houver; nome `regulacao-estado-<hoje>.csv` quando o api não manda um nome seguro) e, na aba "Pelo estado", a nota "Regulados pelo estado não entram na fila municipal nem consomem cota."

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/regulation/ExternalNumberForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  setRegulationExternal: vi.fn() }));
import * as api from "../../lib/api";
import { ExternalNumberForm } from "./ExternalNumberForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { regRequest, regUser } from "../../test/regulationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("ExternalNumberForm", () => {
  beforeEach(() => {
    mocked(api.setRegulationExternal).mockReset().mockResolvedValue(regRequest({ external: { system: "sisreg", number: "2026/123456" } }));
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
  });
  it("formato conferido antes; sistema, número e versão no corpo", async () => {
    const onChanged = vi.fn();
    renderWithProviders(<ExternalNumberForm request={regRequest({ status: "external", state_regulated: true, version: 2 })} onChanged={onChanged} />);
    text("Sistema", "sisreg"); text("Número no sistema do estado", "SR 12");
    fireEvent.click(screen.getByRole("button", { name: "Registrar número" }));
    expect((await screen.findByRole("alert")).textContent).toBe("número inválido: até 30 letras, números, ponto, barra ou hífen");
    text("Número no sistema do estado", "2026/123456");
    fireEvent.click(screen.getByRole("button", { name: "Registrar número" }));
    await waitFor(() => expect(mocked(api.setRegulationExternal)).toHaveBeenCalledWith("rr1", "sisreg", "2026/123456", 2));
    expect(onChanged).toHaveBeenCalledWith("Número externo registrado.");
  });
});
```

No fim de `src/modules/RegulationQueue.test.tsx`:

```tsx
describe("RegulationQueue — exportação para o sistema do estado", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
    mocked(api.countRegulationRequests).mockResolvedValue(regCounts());
    mocked(api.listRegulationRequests).mockResolvedValue({ requests: [ regRequest({ status: "external", state_regulated: true }) ], next_before: null });
    mocked(api.fetchStateExportCsv).mockReset().mockResolvedValue({ blob: new Blob([ "a;b" ]), filename: null });
  });
  afterEach(() => vi.unstubAllGlobals());

  it("CSV com nome sem dado pessoal; nada no storage", async () => {
    const setItem = vi.spyOn(Storage.prototype, "setItem");
    const anchors: HTMLAnchorElement[] = [];
    const realCreate = document.createElement.bind(document);
    vi.spyOn(document, "createElement").mockImplementation((tag: string) => {
      const el = realCreate(tag);
      if (tag === "a") { anchors.push(el as HTMLAnchorElement); (el as HTMLAnchorElement).click = vi.fn(); }
      return el;
    });
    class FakeURL extends URL {}
    Object.assign(FakeURL, { createObjectURL: vi.fn(() => "blob:x"), revokeObjectURL: vi.fn() });
    vi.stubGlobal("URL", FakeURL);
    renderWithProviders(<RegulationQueue today={TODAY26} />);
    fireEvent.click(await screen.findByRole("tab", { name: "Pelo estado (1)" }));
    expect(await screen.findByText("Regulados pelo estado não entram na fila municipal nem consomem cota.")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Exportar CSV para o sistema do estado" }));
    await waitFor(() => expect(mocked(api.fetchStateExportCsv)).toHaveBeenCalledWith("external"));
    await waitFor(() => expect(anchors.at(-1)?.download).toBe("regulacao-estado-2026-10-28.csv"));
    expect(setItem).not.toHaveBeenCalled();
    vi.restoreAllMocks();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation/ExternalNumberForm.test.tsx src/modules/RegulationQueue.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** o formulário (recusa antes; `stale_version` → `STALE_TEXT`, o número fica) e, na fila, o botão (a aba "Aguardando" exporta `pending`; o padrão do api sem `status` é `external`, R18); `saveFile(file, file.filename ?? exportFilename(today))`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod26a src/modules/regulation src/modules/RegulationQueue.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/regulation/ExternalNumberForm.tsx src/modules/regulation/ExternalNumberForm.test.tsx src/modules/regulation/RequestDetail.tsx \
  src/modules/RegulationQueue.tsx src/modules/RegulationQueue.test.tsx
$G commit -m "feat: add the state system number and the export for manual entry

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Suíte, build, revisão e prova no navegador (api real na 3048)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–13 na branch `feat/mod-26a-care-regulation`; o api do 26a na **3048** com a semente do 26a.
- Produces: branch pronta para merge (depois do merge do api do 26a); evidências em `evidencias/2026-10-28-modulo-26a/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test dashboard -w dashboard-mod26a && cd $W && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base da Task 0 mais **14 arquivos de teste novos** (`api.regulation`, `regulation`, `modules.regulation`, `useRegulationCount`, `RequestForm`, `RegulationRequestsField`, `ConsultationWorkspace.regulation`, `RegulationQueue`, `RequestDetail`, `RegulationReturned`, `RegulationOffers`, `RegulationSettings`, `ScheduleOutcome`, `ExternalNumberForm`) e os existentes com casos novos (`features.test.ts`, `modules.test.ts`, `team.test.ts`, `ConsultationEditor.test.tsx`, `scheduling.test.ts`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css package.json package-lock.json
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css` ou dependência mudados (o `vite.config.ts` muda só pelo proxy da Task 1).

- [ ] **Step 2: Nada vaza, nada fica no navegador, nada que grava trilha relê sozinho, nada bloqueia a consulta**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage\|indexedDB" src/lib/regulation.ts src/modules/regulation src/modules/Regulation*.tsx src/modules/consultation/RegulationRequestsField.tsx | grep -v "\.test\."
grep -rn "refetchInterval\|refetchOnWindowFocus: true" src/modules/regulation src/modules/Regulation*.tsx src/modules/consultation/RegulationRequestsField.tsx | grep -v "\.test\."
sed -n '/─── Regulação municipal (módulo 26a/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [A-Za-z_, :]* })'
grep -rn "maxLength" src/modules/regulation src/modules/Regulation*.tsx src/modules/consultation/RegulationRequestsField.tsx
grep -rni "posição\|position" src/modules/regulation src/modules/Regulation*.tsx | grep -v "\.test\."
grep -n "regulation\|Regulation" src/modules/consultation/ConsultationEditor.tsx
```

Expected: o primeiro, o segundo, o quarto e o quinto sem saída; o terceiro só com `status`, `overdue`, `type`, `target_code`, `color`, `before`, `limit`, `unit_id`, `month` e `q` (só na busca de alvo); o último só a prop `regulation`, o `<RegulationRequestsField …/>`, o destino dos avisos e o `regulationHint` — nada em `missing`, `finalizeProblems` ou `blockedFinal`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§1–§6, §8), o ADR 0044, o contrato (§1–§8) e as Divergências deste plano. Pontos de atenção:
- sem `care_regulation`, menu, consulta, desfecho e recepção são os de hoje (testes antigos verdes; só as contagens de `modules.test.ts` mudaram);
- o pedido nunca trava a finalização; os avisos aparecem na caixa de finalizar; o não salvo é avisado;
- só médico `225*`, enfermeiro `2235*` e dentista `2232*` veem a peça; dentista sem procedimento; o api recusa de novo;
- fila, detalhe, devolvidos e pedidos da consulta com `NOMINAL_QUERY`; só a contagem relê;
- `quota_exhausted`, `skip_reason_required`, `color_reason_required` e `stale_version` mantêm o que foi digitado; nada reenviado sozinho;
- nenhuma posição na fila; a ordem é a do api;
- CID, justificativa, motivos e termo só no corpo; nomes de arquivo sem dado pessoal;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 26a na **3048** (plano do api: servidor do worktree `apps/api/.claude/mod26a`, migrações de cidade e plataforma a partir de `20261028900001` aplicadas no banco de dev **com autorização do usuário**, e a semente do 26a: Curitiba com `clinical_record`; uma usuária `care_regulator`; uma médica, uma enfermeira e uma dentista em UBS; a UBS Centro e o Ambulatório de Especialidades (rede própria); um prestador contratado; ofertas de outubro com saldo — uma com uma vaga só —; a lista do enfermeiro com um item; pedidos de várias cores e idades, um vencido). Ligue em Curitiba, pelo maintenance, `care_regulation`. O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod26a --network rota-saude_default -p 5203:5203 \
  -v $R/.worktrees/dashboard-mod26a:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3048 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5203 --host 0.0.0.0"
docker logs -f dashboard-mod26a   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 26a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5203/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Roteiro, com screenshot de cada passo em `evidencias/2026-10-28-modulo-26a/dashboard/`:
1. **Admin** — Equipe: conceder "Regulador de encaminhamentos" pede step-up. Regulação → **Prazos e lista do enfermeiro**: prazos 7/30/90/180; tentar amarelo 5 é recusado; acrescentar "Ecocardiografia transtorácica" à lista (step-up). **Prestadores e cotas**: criar um prestador contratado (CNES de 7 dígitos), uma oferta de novembro com 10; mudar a quantidade de uma oferta abaixo do usado → recusa com o número.
2. **Médica** — atendimento chamado → Iniciar consulta → **Pedir regulação**: buscar "cardio", escolher Cardiologia, CID I20.9, justificativa, cor amarela → "Salvar pedido"; o rascunho aparece. Abrir outro pedido, digitar e **não** salvar; "Finalizar consulta": a caixa diz "1 pedido de regulação vai…" e "O pedido … não foi salvo e não vai junto"; o desfecho "Encaminhado" mostra a dica; "Confirmar finalização" finaliza. Na consulta finalizada, "Guia em PDF" abre a guia; um pedido novo + adendo leva o segundo pedido à fila.
3. **Enfermeira** — "Pedir regulação": a busca só traz itens da lista (Ecocardiografia); sem CID-10 (se o CBO não usa); salvar e finalizar.
4. **Dentista** — só "Especialidade", só especialidades odontológicas.
5. **Regulador** — o menu mostra "Fila regulada (N vencidos)"; a fila abre em "Aguardando" com cores, espera e prazo, o vencido no topo; nenhuma posição; filtrar por Cardiologia e por cor.
6. **Autorizar** — abrir o pedido da médica: ofertas do mês com saldo; a esgotada desabilitada; autorizar na rede própria → frase do pedido de agendamento. Em outro pedido mais novo com um mais antigo da mesma cor esperando, autorizar → o api pede o motivo de fora da ordem; registrar e autorizar. Mudar a cor exige motivo.
7. **Concorrência** — a oferta de uma vaga aberta em duas abas em dois pedidos; autorizar na primeira e na segunda → "a cota desta oferta acabou…", a oferta aparece esgotada, a cor e os motivos ficam.
8. **Devolver** — devolver um pedido com motivo; como **médica**, Regulação → **Encaminhamentos devolvidos**: o pedido com o motivo; complementar; ele volta à fila do regulador com a data original.
9. **Negar e cancelar** — negar um pendente; cancelar um autorizado (a cota volta: conferir o saldo em Prestadores e cotas).
10. **Externo** — autorizar num contratado; "Registrar agendamento" com data, hora e local; "Faltou" → volta à fila (primeira falta); autorizar de novo, agendar, "Faltou" → "Segunda falta: o pedido foi encerrado."; noutro, "Realizado".
11. **Recepção** (rede própria) — como atendente na unidade de destino, a fila de pedidos de agendamento mostra "Regulação (pedido de UBS Centro)"; marcar o horário; no detalhe do regulador, o pedido aparece agendado.
12. **Estado** — um pedido "regulado pelo estado" aparece em "Pelo estado", fora da fila; registrar o número SISREG; "Exportar CSV para o sistema do estado" baixa `regulacao-estado-AAAA-MM-DD.csv` sem CPF (CNS só com os 4 últimos dígitos).
13. **Sem polling e sem vazamento** — com a aba de rede aberta na fila por 3 minutos: nenhuma chamada a `/regulation/requests?` sem ação; só `/regulation/requests/count` a cada minuto. Nenhuma URL com CID, justificativa, motivo ou nome; `localStorage` e `sessionStorage` sem nada da regulação.
14. **Telefone** (largura 375): fila, devolvidos, ofertas e configurações viram cartões, sem rolagem lateral.
15. No maintenance, desligar `care_regulation` e recarregar: o grupo some, a consulta não mostra a peça e o desfecho não mostra a dica; religar devolve, com tudo o que foi gravado.

Depois: `docker rm -f dashboard-mod26a`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

F-26.2..F-26.6 vão a **Done** se a parte do api já está Done (o dashboard é o último app deles); F-26.1 espera o maintenance; F-26.7 é do wpda. Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified**. Use o mesmo `gh project item-edit` da Task 0 com o id da opção Done (`98236657`) e o de Verified (da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 26a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-26a-care-regulation
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod26a
/opt/homebrew/bin/git branch -d feat/mod-26a-care-regulation
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (D1–D18), conferida com o plano do api (`2026-10-28-module-26a-care-regulation-api.md`, R1–R19). **Adotada** = a forma do plano do api, já usada aqui; **aberta** = falta ou discordo; quem decide é o coordenador. A Task 0 (Step 4) confere o que o api implementou.

1. **D1 — Busca de alvos. Adotada (R2).** `GET /regulation/targets?type=&q=&limit=` → `{ "targets": [ { "type", "code", "name", "group", "nurse_allowed" } ] }`; procedimento com `q` ≥ 3 (`invalid_query`). A tela filtra por `nurse_allowed` (enfermeiro) e por `2232*` (dentista) e usa 3–60 também na especialidade. Ressalva: o termo vai na URL; o campo pede o nome da especialidade ou do procedimento (nunca a queixa) e a busca não guarda cache.
2. **D2 — Pedidos da consulta. Adotada (R14).** `GET` → `{ "requests": [<regulation_request>] }` (rascunhos e promovidos, sem trilha); `POST`/`PATCH` devolvem `<regulation_request>`; `DELETE` → 204; no máximo 10 por consulta (`invalid_request` com `path: "consultation"`).
3. **D3 — CID-10 de quem não usa CID. Aberta.** O contrato e o `SaveDraft` do api pedem `cid10`, mas o enfermeiro em várias cidades não tem CID-10 liberado (`cid10_allowed_for_cbo: false` no 19a). Proposta: `cid10` obrigatório só quando o CBO da consulta pode usar CID-10; senão, nulo aceito (o regulador vê "—"). Sem isso, a tela do enfermeiro sem CID-10 recebe `invalid_cid10` e a frase manda falar com o médico.
4. **D4 — A finalização e os rascunhos. Aberta (discordo do Desvio 1 do api).** O api revalida alvo e CBO na finalização e, se um rascunho deixou de valer, **recusa a finalização** (rollback). A spec e esta tarefa pedem "sem bloquear a finalização". Proposta: o rascunho que deixou de valer fica `draft` (com o motivo) e a consulta finaliza; o profissional resolve depois pelo adendo. Enquanto o api recusar, a tela traduz a recusa (corpo com `index` → `DRAFT_NO_LONGER_VALID_TEXT`), mantém tudo o que foi digitado e relê a peça (Task 5). Pedido: que essa recusa traga `index` sempre (é como a tela a distingue do `cbo_not_allowed` da própria consulta).
5. **D5 — Pedido por adendo. Adotada (R14).** Rascunho em consulta `finalized` (só a autora) e promoção no próximo adendo.
6. **D6 — Vencidos no topo. Resolvida na tela (R8).** O api ordena por cor e data (contrato §4); a spec pede "vencidos no topo". A aba "Aguardando" faz duas leituras (`overdue=true` e `overdue=false`) e mostra o bloco "Vencidos" acima do "No prazo" — duas trilhas por abertura da aba (uma por leitura nominal). Pedido ao api: aceitar `overdue=false` (o filtro já é `"true"`/`"false"` no plano dele).
7. **D7 — `requested_at` e `due_on` no rascunho. Adotada (Desvio 5 e o CHECK do api).** Nulos em `draft`; `due_on` calculado da data do pedido e recalculado só quando a cor efetiva muda.
8. **D8 — Devolvidos do solicitante. Adotada (R15), com um acréscimo.** `GET /regulation/returned?unit_id=` exige vínculo ativo na unidade e CBO `225*`/`2235*`/`2232*`. Acréscimo: cada pedido vem com `events` (o motivo da devolução está no evento `returned`); sem eles a tela não tem o motivo.
9. **D9 — Ofertas e prestadores. Adotada (R1, R10, R17).** `<provider>` com `health_unit`; `own` herda o CNES da unidade e não muda de tipo; `PATCH` da oferta só com `quantity`; mês passado recusado (`invalid_month`); `duplicate_offer`, `cnes_taken`, `unit_taken`.
10. **D10 — Prazos por cor. Adotada (R19), com um acréscimo.** 1..365, mudar não mexe em pedido feito. Acréscimo: a tela recusa antes um prazo de cor menos grave menor que o da mais grave (vermelho ≤ amarelo ≤ verde ≤ azul); proposta de o api recusar igual (`invalid_request` com `path` da cor).
11. **D11 — Lista do enfermeiro. Adotada (R2).** `{ "items": [ { "target_type", "target_code", "name" } ] }`; `DELETE /regulation/admin/nurse_items` com `{ "target_type", "target_code" }` no corpo; `duplicate_item`.
12. **D12 — Limites. Adotada (Valores fixados 2, R19).** Justificativa 10–2000; motivos 10–500; `place` 1–200; `notes` até 500; número externo `[0-9A-Za-z./-]{1,30}`; nome 3–120 (o api fixa só o máximo de 120: proposta de mínimo 3); quantidade 0–9999.
13. **D13 — Agendamento externo e desfecho. Adotada (R11, R7).** `schedule` em `external` e em `authorized`/`scheduled` não própria; `at` de 30 dias atrás a 365 à frente; `outcome` em `scheduled` não própria; `missed` nunca é situação atual.
14. **D14 — Contagem. Adotada (Task 7 do api).** `overdue` sobre `pending`, `returned`, `authorized` e `external`. O menu usa `overdue`.
15. **D15 — O pedido de agendamento do 17. Adotada (R13).** `kind: "regulated"`, `origin: "regulation"`, `origin_unit` = unidade solicitante; o rótulo na recepção é desta tela (Task 12).
16. **D16 — Rede própria × externa na autorização. Adotada (Desvio 6).** `authorization.provider.kind` decide o que a tela oferece; na rede própria, cancelar com horário marcado é 409 `appointment_scheduled` (R4) e a tela não oferece; sem par validado, 409 `citizen_not_verified` (R3).
17. **D17 — Nomes de arquivo. Aberta.** `Content-Disposition` sem dado pessoal — `guia-regulacao-<id>.pdf` e `regulacao-estado-<AAAA-MM-DD>.csv`; a tela usa esses nomes quando o cabeçalho falta ou não é seguro.
18. **D18 — `skip_reason_required` sem dado de outro paciente. Aberta.** O 422 traz só `{ "error": "skip_reason_required" }` — nem id, nem nome, nem quantidade de pedidos mais antigos (seria a posição por outro caminho).

## Self-review

- **Cobertura (spec §2 decisões 2–8, 10–12 e as do coordenador, §4 telas, §5, §6, §8; contrato §1–§8):** pedido na consulta com especialidade/procedimento, CID, justificativa, cor e guia — Tasks 4, 5 e 6 (F-26.2; Review Focus 1); quem pede pelo CBO — Tasks 2, 4 e 6; pedido por adendo — Task 6 (D5); encaminhamento simples continua no desfecho, com a dica — Task 5; fila por cor e espera, vencidos no topo, sem posição — Task 7 (F-26.3; Review Focus 4; D6); autorizar com oferta com saldo, cor com motivo, fora da ordem com motivo, devolver, negar, cancelar — Task 8 (Review Focus 2 e 3); complemento pelo solicitante, mantendo a data — Task 9; prestadores, ofertas e cotas com saldo e alerta de esgotada — Task 10 (F-26.4; risco "cota mal cadastrada"); prazos por cor e lista do enfermeiro — Task 11 (F-26.1); papel e menu — Task 3; agendamento externo, falta e reentrada, segunda falta, rede própria pelo 17 — Task 12 (F-26.5; D13, D15); número externo, "regulado pelo estado" fora da fila e CSV — Tasks 4, 7 e 13 (F-26.6); sem dado sensível na URL e no navegador — Tasks 1, 13 e 14 (Review Focus 5); prova no navegador com roteiro — Task 14. Retenção LGPD, concorrência real da cota e fuso dos prazos são do api; a tela mostra o que ele devolve.
- **Placeholders:** os valores lidos na execução são as rotas e limites do api (Task 0, com a regra: o api vence), os ids dos cards (Task 0) e os ícones, se algum faltar (Task 0, com a regra). Nenhum passo diz "trate os casos".
- **Tipos e nomes:** `RegulationRequest`, `RegulationQuery`, `RegulationCounts`, `AuthorizeInput`, `ComplementInput`, `ScheduleInput`, `CareOffer`, `CareProvider`, `RiskDeadlines`, `NurseItem` (Task 1) são os usados nas Tasks 4–13; `listRegulationRequests(q)` recebe `{ status?, overdue?, type?, targetCode?, color?, before? }` em todas; as ações com a versão como último argumento (`returnRegulationRequest(id, reason, version)`, `recordRegulationOutcome(id, result, version)`, `setRegulationExternal(id, system, number, version)`) iguais nas Tasks 1, 8, 12 e 13; `RequestDraft`, `requestProblem`, `requestBody`, `draftFromRequest`, `actionsFor`, `waitDays`, `dueText` (Task 2) batem com as Tasks 4–13; `RequestForm` (Task 4) é o das Tasks 5 e 9; `RequestDetail.extra` (Task 8) é o que as Tasks 12 e 13 preenchem; `useRegulationCount` e `REG_COUNT_KEY` iguais nas Tasks 3 e 7; `ConsultationEditor.regulation` e `ConsultationView.regulation` (Tasks 5 e 6) são o que o Workspace passa.
- **Review Focus:** 1 — Task 5 ("formulário aberto ou leitura que falha…" e "ao finalizar: a conta dos rascunhos e o aviso do não salvo, dentro da caixa"); 2 — Task 8 ("409 quota_exhausted…"); 3 — Task 8 ("mudar a cor pede motivo antes; 422 skip_reason_required…"); 4 — Task 7 ("volta à aba e reconecta: a lista não relê") e Task 3 ("a contagem relê a cada minuto…"); 5 — Task 1 ("nada sensível na URL…") e Task 13 ("CSV com nome sem dado pessoal; nada no storage").
- **Proporção:** os blocos de código são testes (os nomes, rótulos e valores que o implementador não decide sozinho), os tipos do contrato e as fixtures; os corpos das telas ficam com a assinatura, o comportamento e as frases.
