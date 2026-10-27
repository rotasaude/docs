# Módulo 25 — Vigilância epidemiológica (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos e bloqueio.** (1) O **api do 25** (`feat/mod-25-disease-surveillance`, worktree `apps/api/.claude/mod25`, porta de dev **3047**) com as rotas `/surveillance/*` do contrato (§3–§8), a rota de sugestões da consulta (§4; D1), o papel `surveillance`, o interruptor `surveillance` na sessão e a semente do 25. Ele é **mergeado antes** deste (contrato §13: api → dashboard → maintenance → catálogo). As Tasks 0–7 e 9–12 não dependem dele (dublês de `src/lib/api`); a prova da Task 13 depende. (2) **A Task 8 (ficha complementar) fica BLOQUEADA até a Task 0 do api publicar o §14 do contrato** (campos da ficha de notificação individual do Sinan que não vêm do paciente). O Step 1 da Task 8 confere isso no `origin/main` do docs; sem o §14, **pule a Task 8**, siga para as Tasks 9–12 e avise o coordenador. Nenhuma chave da ficha é escrita por adivinhação: os testes da lógica da Task 8 usam uma tabela sintética, e a tabela real é transcrita do §14.

**Goal:** Lado dashboard do módulo 25: na **consulta (19a)**, o aviso "CID de notificação obrigatória: notificar suspeita de X" com a data dos primeiros sintomas e o botão **"Notificar suspeita"** sempre disponível para médico (CBO `225*`) e enfermeiro (`2235*`), **sem nunca bloquear a finalização**; o grupo **"Vigilância"** com a **fila** (imediatas e vencidas primeiro, possíveis não feitas), o **detalhe** com andamento, número oficial, descarte de possível e a **ficha complementar** (campos do §14), a **notificação manual** (busca de pessoa e de possíveis duplicados do 24a), a **notificação negativa** por unidade e semana epidemiológica, os **relatórios** (ficha em PDF, CSV por semana, encerradas sem número oficial), o **painel** bairro × semana epidemiológica com supressão e sinal, e as **doenças locais**; fora do grupo, a marca **"unidade notificadora"** na tela de unidades do admin e o papel `surveillance` na Equipe.

**Architecture:** O cliente HTTP ganha a seção "Vigilância epidemiológica (módulo 25)" no fim de `src/lib/api.ts` (tipos do contrato §2–§8 e das Divergências D1–D16; na URL só ids, códigos de doença, situação, semana e cursor; CPF, nome, mãe e ficha sempre no corpo). As regras de tela ficam puras em `src/lib/surveillance.ts` (portão e papéis, CBO que notifica, rótulos dos códigos do contrato §1, mapa de transições, **semana epidemiológica**, conferência de formato — recusar, nunca cortar —, frases das recusas, constantes de consulta) e, na Task 8, `src/lib/surveillanceForm.ts` (a tabela do §14 e a conferência da ficha). As peças ficam em `src/modules/surveillance/` e as telas em `src/modules/` (`SurveillanceQueue`, `SurveillanceNegatives`, `SurveillanceReports`, `SurveillancePanel`, `SurveillanceDiseases`), no grupo novo (`modules.ts`, `catalog.ts`, `App.tsx`). A consulta ganha uma peça (`NotificationSuggestions`) dentro do `ConsultationEditor`, ligada pelo `ConsultationWorkspace` como a prioridade de exame do 21a. A busca de pessoa reaproveita o `PersonStart` do 24a com uma **fonte** trocável. O dashboard nunca decide o que o api garante: lista vigente, duplicidade de notificação, prazos, transições, retenção LGPD, supressão e sinal são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md` (§1 intenção e antiobjetivos, §2 decisões 1–12 e as do coordenador, §3 interruptor, §5 telas, §6 segurança e LGPD, §7 testes, §9 F-IDs), `docs/adr/0043.md` e o contrato `docs/superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md` (§1–§10, §12, §13 e o §14 que a Task 0 do api publica), com as convenções herdadas (21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12, 24a §14). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim (D1–D16); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptor `surveillance`; papel `surveillance`; prazo `immediate` (24 h) | `weekly` (7 dias); origem `consultation` | `manual` | `safety_net`; situação `possible` | `notified` | `investigating` | `closed` | `entered` | `dismissed`; classificação `confirmed` | `discarded` | `inconclusive`; semana epidemiológica `YYYY-WW`. Código desconhecido aparece cru, nunca some.
- Interruptor (spec §3, decisão 11): `surveillance` **sem** exigir o modo prontuário — o grupo "Vigilância" depende só dele; a sugestão na consulta exige também `clinical_record`. Desligado, o dashboard é o de hoje: o grupo some, a consulta não mostra aviso nem botão, a tela de unidades não mostra a marca. 403 `feature_disabled` → `SURVEILLANCE_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo em `features.ts`.
- **A notificação nunca bloqueia a finalização da consulta** (ADR 0043, invariante 1; spec antiobjetivo): a peça da consulta não entra em `finalizeProblems`, não desabilita "Confirmar finalização", e uma falha dela (leitura ou escrita) só aparece nela.
- **Dado do paciente só no corpo** (ADR 0043; contrato, convenções): na URL vão só ids (`id`, `unit_id`, `consultation_id` no caminho), `status`, `disease` (código da doença), `overdue`, `week`, `from_week`, `to_week`, `before` e `limit`. CPF, CNS, nome, nascimento, mãe e ficha nunca na URL, em mensagem de erro, em `console` ou em Analytics; nada em `localStorage`, `sessionStorage` ou IndexedDB. Nome de arquivo baixado sem dado pessoal (`notificacoes-2026-43.csv`, `ficha-notificacao-<id>.pdf`).
- **Nenhum polling em rota que grava trilha** (spec §6): fila, detalhe, lista de encerradas sem número, sugestões da consulta e a busca de pessoa leem só ao abrir, filtrar, trocar de aba, clicar ou depois de uma ação — `NOMINAL_QUERY` do módulo 23. Só `GET /surveillance/notifications/count` (sem trilha) relê a cada minuto (`COUNT_QUERY`). A busca de pessoa e de duplicados é mutação no clique.
- **Aumento atípico é sinal, nunca surto** (ADR 0043, invariante 3): o painel rotula a célula como "sinal" e mostra `SIGNAL_NOTE`; a palavra "surto" não aparece em nenhuma tela deste módulo.
- Supressão (spec decisão 9; contrato §8): contagem `null` aparece como "< 5" (`SUPPRESSED_LABEL` de `src/lib/smallCount.ts`), nunca como 0 nem vazio; zero aparece 0.
- Quem pode (contrato §3–§8): fila, detalhe, manual, ficha, negativa, relatórios e doenças locais só `surveillance`; o painel `surveillance` e `analyst` (agregado sem nome); a sugestão e o "Notificar suspeita" só `health_professional` com CBO `225*` ou `2235*` (o api devolve `cbo_not_allowed` para os outros); a marca "unidade notificadora" só `municipal_admin`. A tela não oferece o que o papel não pode; o api recusa de novo.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo). Doença local passa pelo `SensitiveAction` com step-up (contrato §3); conceder o papel `surveillance` também (é privilegiado: `isPrivilegedRole`).
- Recusar em vez de cortar: nenhum `maxLength` silencioso. Motivo até 500 caracteres (no descarte, de 3 a 500); nome de doença local 3–120; código de doença local `local_[a-z0-9_]{1,40}` (D4); CID-10 `A00`/`A000`/`A00X`; CIAP-2 `A00`; janela de repetição 1–36500 dias; IBGE de residência 7 dígitos; número oficial `[0-9A-Za-z./-]{1,30}` (D6); faixa do painel até 53 semanas (D8) — os valores do plano do api ("Valores fixados" 2 e 4) — tudo conferido antes de enviar e de novo no api.
- Semana epidemiológica (contrato §1; D16): domingo a sábado; a semana 1 é a que contém 4 de janeiro (regra do Sinan); um ano pode ter 53 semanas. O §14 do api confirma; divergência se resolve a favor do §14, com o teste da Task 2 ajustado e nota na Task 13.
- "Hoje" é o dia da cidade (`todayInCity()`); as telas recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY25 = "2026-10-27"` (terça; semana `2026-43`), sem relógio falso.
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `saveFile` de `src/modules/lab/saveFile.ts`); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-25.1..F-25.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-25.1: api, dashboard e maintenance; F-25.2 e F-25.4..F-25.8: api e dashboard; F-25.3: só api) e a **Verified** depois da prova no navegador (Task 13).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod25` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-25-disease-surveillance` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod25`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test dashboard -w dashboard-mod25 <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx` (sessão com `mfa_verified_at` de agora = janela de step-up aberta).
- A prova (Task 13) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5202**, com proxy para o api do 25 na **3047**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Médico ignora o aviso, ou a leitura das sugestões falha (500, rede).** Espera-se: "Confirmar finalização" continua habilitado e finaliza; o aviso pendente diz, dentro do quadro de finalizar, que a vigilância verá o caso como possível notificação não feita; a falha aparece só na peça. Teste: Task 4, "sugestão pendente ou leitura que falha: a finalização segue igual".
2. **Fila aberta o turno todo.** Espera-se: a lista nominal (grava trilha) não relê ao voltar à aba, ao reconectar nem com o tempo; só a contagem relê a cada minuto, e o menu mostra as imediatas abertas. Testes: Task 5, "volta à aba e reconecta: a lista não relê"; Task 3, "a contagem relê a cada minuto e acende o item com as imediatas".
3. **Duas pessoas da vigilância mexem na mesma notificação.** Espera-se: 409 `stale_version` vira "outra pessoa alterou esta notificação…", o que foi digitado (classificação, número, motivo, ficha) fica na tela, nada é reenviado sozinho, "Recarregar" traz a versão nova. Testes: Task 6, "409 stale_version na transição: frase, o formulário fica, recarregar relê"; Task 8, "409 stale_version na ficha: frase e os valores ficam".
4. **CPF ou nome da notificação manual vazando.** Espera-se: a busca por CPF, a de duplicados e o `new_patient` vão só no corpo de POST; nenhuma URL chamada tem CPF, nome ou mãe; nada em `localStorage`/`sessionStorage`. Teste: Task 7, "busca e criação: CPF, nome e mãe só no corpo; nada no storage".
5. **Virada de ano e semana 53.** A vigilância abre a negativa ou o CSV na primeira semana do ano. Espera-se: a semana anterior a `2026-01` é `2025-53`; 1º de janeiro de 2022 é `2021-52`; a negativa e o CSV vão com a semana certa. Testes: Task 2, "semana epidemiológica: domingo a sábado, 4 de janeiro, semana 53"; Task 9, "na semana 2026-01, a anterior é 2025-53 e a negativa vai com ela".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, situação do §14 | 0 |
| `src/lib/api.ts`, `src/lib/api.surveillance.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/surveillanceFixtures.ts` | tipos e cliente HTTP do 25; interruptor; proxy; dados de teste | 1 |
| `src/lib/surveillance.ts`, `src/lib/surveillance.test.ts` | regras de tela: portão, CBO, rótulos, transições, semana epidemiológica, formato, frases | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.surveillance.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/hooks/useSurveillanceCount.ts`, `src/hooks/useSurveillanceCount.test.tsx`, `src/modules/surveillance/SurveillanceGate.tsx`, cinco telas-portão, `src/lib/team.ts`, `src/lib/team.test.ts` | grupo "Vigilância", contador no menu, portão, papel na Equipe | 3 |
| `src/modules/consultation/NotificationSuggestions.tsx`, `src/modules/consultation/NotificationSuggestions.test.tsx`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationEditor.test.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.surveillance.test.tsx` | aviso e "Notificar suspeita" na consulta (F-25.2) | 4 |
| `src/modules/SurveillanceQueue.tsx`, `src/modules/surveillance/NotificationTable.tsx`, `src/modules/SurveillanceQueue.test.tsx` | fila com prazo, abas e filtros (F-25.4) | 5 |
| `src/modules/surveillance/NotificationDetail.tsx`, `src/modules/surveillance/TransitionForm.tsx`, `src/modules/surveillance/NotificationDetail.test.tsx` | detalhe, andamento, número oficial, descarte, PDF (F-25.5, F-25.7) | 6 |
| `src/modules/territory/PersonStart.tsx`, `src/modules/territory/PersonStart.test.tsx`, `src/lib/territoryDictionaries.ts`, `src/modules/surveillance/ManualNotificationPanel.tsx`, `src/modules/surveillance/ManualNotificationPanel.test.tsx`, `src/modules/SurveillanceQueue.tsx` | notificação manual com a busca de pessoa e duplicados do 24a (F-25.4) | 7 |
| `src/lib/surveillanceForm.ts`, `src/lib/surveillanceForm.test.ts`, `src/modules/surveillance/NotificationFormSection.tsx`, `src/modules/surveillance/NotificationFormSection.test.tsx`, `src/modules/surveillance/NotificationDetail.tsx` | ficha complementar do §14 (**bloqueada até o §14**) | 8 |
| `src/modules/SurveillanceNegatives.tsx`, `src/modules/SurveillanceNegatives.test.tsx`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/UnitForm.surveillance.test.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx` | notificação negativa e a marca "unidade notificadora" (F-25.6) | 9 |
| `src/modules/SurveillanceReports.tsx`, `src/modules/SurveillanceReports.test.tsx` | CSV por semana e encerradas sem número (F-25.7) | 10 |
| `src/modules/SurveillancePanel.tsx`, `src/modules/SurveillancePanel.test.tsx` | painel bairro × semana, supressão e sinal (F-25.8) | 11 |
| `src/modules/SurveillanceDiseases.tsx`, `src/modules/SurveillanceDiseases.test.tsx` | lista nacional (leitura) e doenças locais com step-up (F-25.1) | 12 |
| — | suíte, build, revisão, prova no navegador | 13 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, nome, mãe, ficha) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa —, a semana epidemiológica nas viradas de ano e o mapa de transições (Task 2); menu por papel e interruptor e o contador (Task 3); cada tela e peça pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel e interruptor, cobrindo o caminho feliz, cada recusa que muda a tela (`already_notified`, `cbo_not_allowed`, `stale_version`, `invalid_transition`, `classification_required`, `official_number_required`, `invalid_form` com `path`, `unit_has_notifications`, `already_recorded`, `future_week`, `invalid_disease`, `duplicate_code`, `duplicate_suspected`, `feature_disabled`), o step-up, a ausência de releitura, a ausência de armazenamento local e a finalização que nunca trava (Tasks 4–12). A prova final é no navegador, contra o api real na 3047 (Task 13).

---

### Task 0: Board, worktree, base, âncoras e situação do §14

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`8d388b8` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o `origin/main` do docs.
- Produces: cards F-25.1..F-25.8 em In Progress; worktree `$W` na branch `feat/mod-25-disease-surveillance`; a base anotada; a situação do §14 anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-25\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-25.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "procedure_billing" | "care_lines" | "territory_registry";\|  territory_registry: "Cadastro individual e domiciliar"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export interface HealthUnit extends\|^  collects_samples?: boolean;\|^export async function updateUnit(\|^export interface NewPatient {\|^export interface DuplicateCandidate \|^export interface TerritoryPerson \|^export interface SignatureFile \|^async function binaryFetch'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "territory-team" | "registry-status" | "territory-micro-areas";\|      if (item.id === "territory-micro-areas") return territoryOn && isAdmin;\|^export function withQcBlockCount'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 12\|NAV_GROUPS.length - 11'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '  IconHomeHeart, IconMap2, IconCalendarDue, IconHexagons\|    "Cadastro territorial": { icon: IconHomeHeart'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "territory-micro-areas": return <TerritoryMicroAreas\|^export function shellGroups(\|  const qcBlocks = qcBlockQuery.data ?? 0;'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/care_lines(/|$)": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^export const NOMINAL_QUERY\|^export const COUNT_QUERY\|^export const STATIC_QUERY\|^export const COUNT_REFRESH_MS'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '  { role: "billing_clerk", label: "Faturista" },\|^const PRIVILEGED_ROLES'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationEditor.tsx | grep -n '^  examPriority?: boolean;\|      <ExamRequestsField value={draft.exams}'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n 'referenceUnitIds={props.referenceUnitIds} unit={props.unit} units={props.units} examPriority={examsOn}'
$G -C $R/apps/dashboard show origin/main:src/modules/territory/PersonStart.tsx | grep -n '^export function PersonStart(\|    mutationFn: (digits: string) => lookupTerritoryPerson(digits),\|    mutationFn: (q: DuplicateQuery) => findDuplicates(q),'
$G -C $R/apps/dashboard show origin/main:src/lib/territoryDictionaries.ts | grep -n '^function newPatientProblem(raw: NewPatient, today: string): Problem | null {'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitForm.tsx | grep -n 'collectsSamples?: boolean }'
$G -C $R/apps/dashboard show origin/main:src/modules/Attendance.tsx | grep -n '{isAdmin && <Units labOn={hasFeature(user, "municipal_lab")} />}'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod25 -b feat/mod-25-disease-surveillance origin/main
cd $R/.worktrees/dashboard-mod25 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod25
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Virus|BellRinging|CalendarX|FileSpreadsheet|GridDots|VirusSearch)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os seis ícones (falta algum → escolha outro que exista e use o mesmo na Task 3; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 13 espera a base mais **16 arquivos de teste novos** (listados lá; 14 se a Task 8 ficar bloqueada).

- [ ] **Step 4: Confira o api do 25 e o §14** (informativo para as Tasks 1–7 e 9–12; **decisivo** para a Task 8)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-25-disease-surveillance
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod25 api bin/rails routes 2>/dev/null | grep -E " /surveillance/|notification_suggestions|/attendance/units" | head -60
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md | grep -n '^## '
```

Expected: a branch e as rotas (compare com o contrato §3–§8 e com as Divergências D1–D16: diferença de forma vira ajuste em `src/lib/api.ts` e `src/test/surveillanceFixtures.ts` antes da prova, com nota para a Task 13). No contrato, **anote** se já existe o `## 14.` (ficha). Sem ele, a Task 8 espera.

---

### Task 1: Tipos, cliente HTTP, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo; `HealthUnit` e `updateUnit`), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/surveillanceFixtures.ts`
- Test: `src/lib/api.surveillance.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `binaryFetch`, `NewPatient`, `DuplicateQuery`, `DuplicateCandidate`, `TerritoryPerson`, `SignatureFile`, `SessionUser` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`): os tipos `DiseaseDeadline`, `NotificationOrigin`, `NotificationStatus`, `NotificationClassification`, `NotifiableDisease`, `LocalDisease`, `LocalDiseaseInput`, `NotificationSuggestion`, `NotificationForm`, `NotificationEvent`, `NotificationPatient`, `SurveillanceNotification`, `NotificationQuery`, `NotificationPage`, `NotificationCounts`, `ConsultationNotificationInput`, `ManualNotificationInput`, `TransitionInput`, `WeeklyNegativeStatus`, `WeeklyNegatives`, `SurveillanceUnit`, `PanelQuery`, `SurveillancePanel`; a constante `SURVEILLANCE_PAGE_SIZE = 50`; as funções `listSurveillanceDiseases(): Promise<NotifiableDisease[]>`, `getNotificationSuggestions(consultationId: string): Promise<NotificationSuggestion[]>`, `createConsultationNotification(input: ConsultationNotificationInput): Promise<SurveillanceNotification>`, `createManualNotification(input: ManualNotificationInput): Promise<SurveillanceNotification>`, `listNotifications(q: NotificationQuery): Promise<NotificationPage>`, `getNotification(id: string): Promise<SurveillanceNotification>`, `countNotifications(): Promise<NotificationCounts>`, `updateNotificationForm(id: string, form: NotificationForm, version: number): Promise<SurveillanceNotification>`, `transitionNotification(id: string, input: TransitionInput): Promise<SurveillanceNotification>`, `dismissNotification(id: string, reason: string, version: number): Promise<SurveillanceNotification>`, `listSurveillanceUnits(): Promise<SurveillanceUnit[]>`, `lookupSurveillancePerson(cpf: string): Promise<TerritoryPerson>`, `findSurveillanceDuplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>`, `listWeeklyNegatives(week: string): Promise<WeeklyNegatives>`, `recordWeeklyNegative(unitId: string, week: string): Promise<void>`, `fetchNotificationPdf(id: string): Promise<SignatureFile>`, `fetchWeekCsv(week: string): Promise<SignatureFile>`, `getSurveillancePanel(q: PanelQuery): Promise<SurveillancePanel>`, `listLocalDiseases(): Promise<LocalDisease[]>`, `createLocalDisease(input: LocalDiseaseInput): Promise<LocalDisease>`, `updateLocalDisease(id: string, input: LocalDiseaseInput): Promise<LocalDisease>`. `HealthUnit` ganha `notifying?: boolean`; `updateUnit` ganha o 7º parâmetro `notifying?: boolean`. `FeatureKey` ganha `surveillance`.
- Produces (em `src/test/surveillanceFixtures.ts`): `TODAY25`, `UBS_CENTRO`, `UBS_VILA`, `DENGUE`, `MEASLES`, `survUser(roles?, over?)`, `disease(over?)`, `notification(over?)`, `counts(over?)`, `localDisease(over?)`, `negatives(over?)`, `panel(over?)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.surveillance.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  countNotifications, createConsultationNotification, createLocalDisease, createManualNotification, dismissNotification, fetchNotificationPdf,
  fetchWeekCsv, findSurveillanceDuplicates, getNotification, getNotificationSuggestions, getSurveillancePanel, listLocalDiseases, listNotifications,
  listSurveillanceDiseases, listSurveillanceUnits, listWeeklyNegatives, lookupSurveillancePerson, recordWeeklyNegative, transitionNotification,
  updateLocalDisease, updateNotificationForm, updateUnit
} from "./api";

function ok(body: unknown, headers: Record<string, string> = { "Content-Type": "application/json" }): Promise<Response> {
  return Promise.resolve(new Response(typeof body === "string" ? body : JSON.stringify(body), { status: 200, headers }));
}
const SENSITIVE = /cpf|cns|name|nome|mother|mae|birth|nasc|form|ficha|\d{11}/i;

describe("cliente da vigilância (contrato §3–§8)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(() => ok({})); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => (fetchMock.mock.calls[i][1] ?? {}) as RequestInit;
  const headersOf = (i = 0) => initOf(i).headers as Record<string, string>;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));

  it("doenças da cidade e sugestões da consulta em envelope; ausente vira lista vazia", async () => {
    fetchMock.mockImplementationOnce(() => ok({ diseases: [ { code: "dengue" } ] })).mockImplementationOnce(() => ok({}));
    expect(await listSurveillanceDiseases()).toEqual([ { code: "dengue" } ]);
    expect(urlOf(0).pathname).toBe("/surveillance/diseases");
    expect(await getNotificationSuggestions("cs1")).toEqual([]);
    expect(urlOf(1).pathname).toBe("/attendance/consultations/cs1/notification_suggestions");
  });

  it("notificar na consulta: só ids, código e data no corpo, com JSON", async () => {
    fetchMock.mockImplementationOnce(() => ok({ id: "nt1" }));
    await createConsultationNotification({ consultation_id: "cs1", disease_code: "dengue", symptoms_on: "2026-10-25" });
    expect(urlOf().pathname).toBe("/surveillance/notifications");
    expect(initOf().method).toBe("POST");
    expect(headersOf()["Content-Type"]).toBe("application/json");
    expect(bodyOf()).toEqual({ consultation_id: "cs1", disease_code: "dengue", symptoms_on: "2026-10-25" });
  });

  it("manual: a pessoa e a residência no corpo, nunca na URL", async () => {
    await createManualNotification({ new_patient: { full_name: "José Souza", birth_date: "2026-09-30", sex: "male", no_cpf_reason: "3" },
      duplicates_reviewed: true, disease_code: "sarampo", suspected_on: "2026-10-27", symptoms_on: null, unit_id: "u1",
      resident_elsewhere: true, residence_ibge: "4106902" });
    expect(urlOf().pathname + urlOf().search).toBe("/surveillance/notifications");
    expect(bodyOf().new_patient.full_name).toBe("José Souza");
    expect(bodyOf().residence_ibge).toBe("4106902");
  });

  it("fila: filtros e cursor na URL; contagem sem envelope", async () => {
    fetchMock.mockImplementationOnce(() => ok({ notifications: [], next_before: "c1" }))
      .mockImplementationOnce(() => ok({ possible: 2, notified: 5, investigating: 1, overdue: 3, immediate_open: 1, without_official_number: 4 }));
    expect(await listNotifications({ status: "notified", disease: "dengue", overdue: true, unitId: "u1", before: "c0" }))
      .toEqual({ notifications: [], next_before: "c1" });
    expect(urlOf(0).pathname + urlOf(0).search).toBe("/surveillance/notifications?status=notified&disease=dengue&overdue=true&unit_id=u1&before=c0&limit=50");
    expect((await countNotifications()).immediate_open).toBe(1);
    expect(urlOf(1).pathname).toBe("/surveillance/notifications/count");
  });

  it("detalhe, ficha, transição e descarte: versão no corpo", async () => {
    fetchMock.mockImplementation(() => ok({ id: "nt1" }));
    await getNotification("nt1");
    expect(urlOf(0).pathname).toBe("/surveillance/notifications/nt1");
    await updateNotificationForm("nt1", { campo: "1" }, 3);
    expect(initOf(1).method).toBe("PATCH");
    expect(bodyOf(1)).toEqual({ form: { campo: "1" }, version: 3 });
    await transitionNotification("nt1", { to: "closed", classification: "confirmed", version: 4 });
    expect(urlOf(2).pathname).toBe("/surveillance/notifications/nt1/transition");
    expect(bodyOf(2)).toEqual({ to: "closed", classification: "confirmed", version: 4 });
    await dismissNotification("nt1", "não era caso", 5);
    expect(urlOf(3).pathname).toBe("/surveillance/notifications/nt1/dismiss");
    expect(bodyOf(3)).toEqual({ reason: "não era caso", version: 5 });
    for (let i = 0; i < 4; i++) expect(urlOf(i).search).toBe("");
  });

  it("unidades e busca de pessoa da vigilância (D2, D3): CPF e nome só no corpo", async () => {
    fetchMock.mockImplementationOnce(() => ok({ units: [ { id: "u1" } ] }))
      .mockImplementationOnce(() => ok({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", has_cpf: true, cns_masked: null }))
      .mockImplementationOnce(() => ok({ candidates: [ { patient_id: "pa7" } ] }));
    expect(await listSurveillanceUnits()).toEqual([ { id: "u1" } ]);
    expect(urlOf(0).pathname).toBe("/surveillance/units");
    expect(await lookupSurveillancePerson("52998224725")).toEqual({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", has_cpf: true, cns_masked: null, registration_id: null });
    expect(urlOf(1).pathname + urlOf(1).search).toBe("/surveillance/person_lookup");
    expect(bodyOf(1)).toEqual({ cpf: "52998224725" });
    expect(await findSurveillanceDuplicates({ full_name: "Maria da Silva", birth_date: "1960-02-01", mother_name: "Ana" }))
      .toEqual([ { patient_id: "pa7", registration_id: null } ]);
    expect(urlOf(2).pathname + urlOf(2).search).toBe("/surveillance/patients/duplicates");
    for (let i = 0; i < 3; i++) expect(urlOf(i).search).not.toMatch(SENSITIVE);
  });

  it("negativa, saídas e painel: só semana e doença na URL", async () => {
    fetchMock.mockImplementationOnce(() => ok({ week: "2026-42", units: [] })).mockImplementationOnce(() => ok({}))
      .mockImplementationOnce(() => ok("%PDF", { "Content-Type": "application/pdf", "Content-Disposition": 'attachment; filename="ficha-notificacao-nt1.pdf"' }))
      .mockImplementationOnce(() => ok("a;b", { "Content-Type": "text/csv" }))
      .mockImplementationOnce(() => ok({ weeks: [ "2026-42" ], rows: [], suppressed: true }));
    expect(await listWeeklyNegatives("2026-42")).toEqual({ week: "2026-42", units: [] });
    expect(urlOf(0).pathname + urlOf(0).search).toBe("/surveillance/weekly_negatives?week=2026-42");
    await recordWeeklyNegative("u1", "2026-42");
    expect(bodyOf(1)).toEqual({ unit_id: "u1", week: "2026-42" });
    expect((await fetchNotificationPdf("nt1")).filename).toBe("ficha-notificacao-nt1.pdf");
    expect(urlOf(2).pathname).toBe("/surveillance/notifications/nt1/form.pdf");
    expect((await fetchWeekCsv("2026-42")).filename).toBeNull();
    expect(urlOf(3).pathname + urlOf(3).search).toBe("/surveillance/exports/week.csv?week=2026-42");
    await getSurveillancePanel({ fromWeek: "2026-35", toWeek: "2026-42", disease: "dengue" });
    expect(urlOf(4).pathname + urlOf(4).search).toBe("/surveillance/panel?from_week=2026-35&to_week=2026-42&disease=dengue");
  });

  it("doenças locais: lista, criação e alteração com JSON (D4)", async () => {
    fetchMock.mockImplementationOnce(() => ok({ diseases: [ { id: "ld1" } ] })).mockImplementation(() => ok({ id: "ld1" }));
    expect(await listLocalDiseases()).toEqual([ { id: "ld1" } ]);
    expect(urlOf(0).pathname).toBe("/surveillance/admin/local_diseases");
    const input = { code: "local_febre_oropouche", name: "Febre do Oropouche", deadline: "immediate" as const, cid10: [ "A938" ], ciap2: [], repeat_after_days: 30 };
    await createLocalDisease(input);
    expect(initOf(1).method).toBe("POST");
    expect(bodyOf(1)).toEqual(input);
    await updateLocalDisease("ld1", { ...input, active: false });
    expect(urlOf(2).pathname).toBe("/surveillance/admin/local_diseases/ld1");
    expect(initOf(2).method).toBe("PATCH");
  });

  it("unidade notificadora vai no corpo da edição da unidade (D15)", async () => {
    fetchMock.mockImplementationOnce(() => ok({ unit: { id: "u1" } }));
    await updateUnit("u1", "UBS Centro", "ubs", { address_street: null, address_number: null, address_complement: null, address_zip: null, neighborhood_id: null },
      undefined, undefined, false);
    expect(bodyOf().notifying).toBe(false);
  });
});
```

E, em `src/lib/features.test.ts`, no fim:

```ts
describe("módulo 25", () => {
  it("surveillance tem rótulo em português", () => {
    expect(featureLabel("surveillance")).toBe("Vigilância epidemiológica");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/lib/api.surveillance.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções não existem; `featureLabel("surveillance")` devolve a chave.

- [ ] **Step 3: Implemente a seção "Vigilância epidemiológica (módulo 25)" no fim de `src/lib/api.ts`**

Base `const SURVEILLANCE_BASE = import.meta.env.VITE_SURVEILLANCE_BASE || "/surveillance"`, `survPath(...parts)` com `encodeURIComponent` em cada parte, `survWrite(method, body)` como o `territoryWrite`. Listas desembrulham o envelope com `?? []`; escritas e detalhe devolvem o objeto puro. A sugestão usa `consultationPath(id, "notification_suggestions")` (D1). `lookupSurveillancePerson` lê a pessoa sem envelope (forma do 24a, mais `has_cpf` e `cns_masked`; plano do api, Task 10) e completa `registration_id: null`; `findSurveillanceDuplicates` completa `registration_id: null` em cada candidato (a vigilância não abre cadastro territorial: o `PersonStart` então nunca oferece "Já tem cadastro — abrir"). PDF e CSV por `binaryFetch` (`application/pdf`, `text/csv`). A query da fila só põe o que veio, na ordem `status`, `disease`, `overdue` (`"true"` só quando verdadeiro), `unit_id`, `before`, e sempre `limit=50`. Tipos (os campos opcionais vêm das Divergências):

```ts
export type DiseaseDeadline = "immediate" | "weekly";
export type NotificationOrigin = "consultation" | "manual" | "safety_net";
export type NotificationStatus = "possible" | "notified" | "investigating" | "closed" | "entered" | "dismissed";
export type NotificationClassification = "confirmed" | "discarded" | "inconclusive";
export interface NotifiableDisease {
  code: string; name: string; deadline: DiseaseDeadline | string; cid10: string[]; ciap2: string[]; repeat_after_days: number;
  sinan_form: string | null; local: boolean; list_version: number | null;
}
// D4: a doença local tem id e pode ser desativada (some das sugestões; o que foi notificado fica).
export interface LocalDisease extends NotifiableDisease { id: string; active: boolean }
export interface LocalDiseaseInput {
  code?: string; name: string; deadline: DiseaseDeadline; cid10: string[]; ciap2: string[]; repeat_after_days: number; active?: boolean;
}
export interface NotificationSuggestion { disease: NotifiableDisease; matched_code: string }
// Chaves do §14 (Task 8); valores escalares.
export type NotificationForm = Record<string, string | number | boolean | null>;
export interface NotificationEvent { status: NotificationStatus | string; at: string; by: { name: string } | null; reason: string | null }
export interface NotificationPatient { id: string; display_name: string | null; has_cpf: boolean; cns_masked: string | null }
export interface SurveillanceNotification {
  id: string; patient: NotificationPatient; disease: { code: string; name: string; deadline: DiseaseDeadline | string };
  origin: NotificationOrigin | string; status: NotificationStatus | string; classification: NotificationClassification | string | null;
  suspected_on: string; symptoms_on: string | null; unit: { id: string; cnes: string | null; name: string }; notifier: { name: string } | null;
  due_at: string; overdue: boolean; official_number: string | null; resident_elsewhere: boolean; residence_ibge: string | null;
  neighborhood: { id: string | null; name: string } | null; version: number;
  // Só no detalhe (a lista não traz).
  form?: NotificationForm; events?: NotificationEvent[];
}
export interface NotificationQuery { status?: NotificationStatus | null; disease?: string | null; overdue?: boolean; unitId?: string | null; before?: string | null }
export interface NotificationPage { notifications: SurveillanceNotification[]; next_before: string | null }
export interface NotificationCounts {
  possible: number; notified: number; investigating: number; overdue: number; immediate_open: number; without_official_number: number;
}
export interface ConsultationNotificationInput { consultation_id: string; disease_code: string; symptoms_on?: string | null }
export type ManualNotificationInput = (
  { patient_id: string } | { citizen_id: string } | { new_patient: NewPatient; duplicates_reviewed?: boolean }
) & {
  disease_code: string; suspected_on: string; symptoms_on?: string | null; unit_id: string;
  resident_elsewhere?: boolean; residence_ibge?: string | null;
};
export interface TransitionInput {
  to: NotificationStatus; classification?: NotificationClassification; official_number?: string; reason?: string; version: number;
}
export type WeeklyNegativeStatus = "with_cases" | "negative" | "pending";
export interface WeeklyNegatives { week: string; units: { unit: { id: string; cnes: string | null; name: string }; status: WeeklyNegativeStatus | string }[] }
// D2: unidades ativas para a vigilância (a rota do atendimento não é do papel surveillance).
export interface SurveillanceUnit { id: string; cnes: string | null; name: string; kind: string; notifying: boolean }
export interface PanelQuery { fromWeek: string; toWeek: string; disease?: string | null }
export interface SurveillancePanel {
  weeks: string[]; rows: { neighborhood: { id: string | null; name: string }; counts: (number | null)[]; signal: boolean[] }[]; suppressed: boolean;
}
```

Em `HealthUnit`, depois de `collects_samples?: boolean;`, `// Módulo 25 (D15): unidade notificadora (notificação negativa semanal).` e `notifying?: boolean;`. Em `updateUnit`, o 7º parâmetro `notifying?: boolean` e `if (notifying !== undefined) body.notifying = notifying;`.

Em `src/lib/features.ts`: `FeatureKey` ganha `| "surveillance"` e `FEATURE_LABEL` ganha, depois de `territory_registry`, `// Módulo 25 (ADR 0043): sem pré-requisito de modo de prontuário; a sugestão na consulta exige clinical_record.` e `surveillance: "Vigilância epidemiológica"`.

Em `vite.config.ts`, depois de `"^/care_lines(/|$)": proxy(TARGET)`, a vírgula e `// Módulo 25: vigilância epidemiológica. Regex como os vizinhos.` e `"^/surveillance(/|$)": proxy(TARGET)`.

`src/test/surveillanceFixtures.ts` (doenças ilustrativas: a lista real vem da Task 0 do api):

```ts
// Dados comuns aos testes do módulo 25. Hoje: terça, 2026-10-27 (semana 2026-43).
import type { LocalDisease, NotifiableDisease, NotificationCounts, SessionUser, SurveillanceNotification, SurveillancePanel, WeeklyNegatives } from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY25 = "2026-10-27";
export const UBS_CENTRO = { id: "u1", cnes: "2384299", name: "UBS Centro" };
export const UBS_VILA = { id: "u2", cnes: "2384302", name: "UBS Vila Verde" };
export const DENGUE: NotifiableDisease = { code: "dengue", name: "Dengue", deadline: "weekly", cid10: [ "A90" ], ciap2: [],
  repeat_after_days: 30, sinan_form: "dengue", local: false, list_version: 1 };
export const MEASLES: NotifiableDisease = { code: "sarampo", name: "Sarampo", deadline: "immediate", cid10: [ "B059" ], ciap2: [ "A71" ],
  repeat_after_days: 30, sinan_form: null, local: false, list_version: 1 };

export function survUser(roles: string[] = [ "surveillance" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { features: [ "surveillance" ], ...over });
}
export function disease(over: Partial<NotifiableDisease> = {}): NotifiableDisease { return { ...DENGUE, ...over }; }
export function notification(over: Partial<SurveillanceNotification> = {}): SurveillanceNotification {
  return {
    id: "nt1", patient: { id: "pa1", display_name: "Joana Lima", has_cpf: true, cns_masked: null },
    disease: { code: "dengue", name: "Dengue", deadline: "weekly" }, origin: "consultation", status: "notified", classification: null,
    suspected_on: "2026-10-26", symptoms_on: "2026-10-24", unit: UBS_CENTRO, notifier: { name: "Dra. Helena Prado" },
    due_at: "2026-11-02T10:00:00-03:00", overdue: false, official_number: null, resident_elsewhere: false, residence_ibge: null,
    neighborhood: { id: "nb1", name: "Centro" }, version: 1, form: {},
    events: [ { status: "notified", at: "2026-10-26T10:00:00-03:00", by: { name: "Dra. Helena Prado" }, reason: null } ], ...over
  };
}
export function counts(over: Partial<NotificationCounts> = {}): NotificationCounts {
  return { possible: 2, notified: 5, investigating: 1, overdue: 3, immediate_open: 1, without_official_number: 4, ...over };
}
export function localDisease(over: Partial<LocalDisease> = {}): LocalDisease {
  return { id: "ld1", code: "local_febre_oropouche", name: "Febre do Oropouche", deadline: "immediate", cid10: [ "A938" ], ciap2: [],
    repeat_after_days: 30, sinan_form: null, local: true, list_version: null, active: true, ...over };
}
export function negatives(over: Partial<WeeklyNegatives> = {}): WeeklyNegatives {
  return { week: "2026-42", units: [ { unit: UBS_CENTRO, status: "with_cases" }, { unit: UBS_VILA, status: "pending" } ], ...over };
}
export function panel(over: Partial<SurveillancePanel> = {}): SurveillancePanel {
  return { weeks: [ "2026-41", "2026-42" ], suppressed: true, rows: [
    { neighborhood: { id: "nb1", name: "Centro" }, counts: [ 6, 14 ], signal: [ false, true ] },
    { neighborhood: { id: null, name: "não informado" }, counts: [ null, 0 ], signal: [ false, false ] } ], ...over };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/lib && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.surveillance.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/surveillanceFixtures.ts
$G commit -m "feat: add the disease surveillance api client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/surveillance.ts`)

**Files:**
- Create: `src/lib/surveillance.ts`
- Test: `src/lib/surveillance.test.ts`

**Interfaces:**
- Consumes: `hasFeature`, `featureDisabledKey` (`features.ts`); `errorCode`, `ApiError` (`api.ts`); `describeActionError` (`actionErrors.ts`); `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `fmtDay` (`careLines.ts`); `DUPLICATE_SUSPECTED_TEXT` (`territoryRegistry.ts`).
- Produces: `SURVEILLANCE_DISABLED`, `SURVEILLANCE_NO_ROLE`, `STALE_TEXT`, `SIGNAL_NOTE`, `NO_NAME`, `NO_NEIGHBORHOOD`; as chaves `DISEASES_KEY`, `SUGGESTIONS_KEY`, `NOTIFICATIONS_KEY`, `NOTIFICATION_KEY`, `SURVEILLANCE_COUNT_KEY`, `UNITS_KEY`, `NEGATIVES_KEY`, `PANEL_KEY`, `LOCAL_DISEASES_KEY`; reexporta `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `fmtDay`; `canUseSurveillance(user): boolean`, `surveillanceRoles(user): SurveillanceRoles` (`{ surveillance, analyst, professional, admin }`), `canSuggestNotification(user): boolean`, `cboCanNotify(cbo: string | null | undefined): boolean`; rótulos `statusLabel`, `statusTone`, `originLabel`, `classificationLabel`, `deadlineLabel`, `negativeStatusLabel`, `CLASSIFICATIONS`; `NEXT: Record<string, NotificationStatus[]>`, `transitionLabel(to)`; `epiWeekOf(isoDate): string`, `epiWeekStart(week): string`, `shiftEpiWeek(week, delta): string`, `previousEpiWeek(today): string`, `weekLabel(week): string`; `weekProblem(week, today): string | null`, `closedWeekProblem(week, today): string | null`, `weekRangeProblem(from, to, today): string | null`, `pastDateProblem(value, label, today, required): string | null`, `symptomsProblem(symptoms, suspected, today): string | null`, `officialNumberProblem(text): string | null`, `residenceProblem(elsewhere, ibge): string | null`, `reasonProblem(text, required): string | null`, `codesOf(text): string[]`, `localDiseaseProblem(draft, creating): string | null`; `surveillanceError(err): string`, `isStale(err): boolean`, `existingNotificationId(err): string | null`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/surveillance.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  NEXT, SURVEILLANCE_DISABLED, canSuggestNotification, canUseSurveillance, cboCanNotify, closedWeekProblem, codesOf, epiWeekOf, epiWeekStart,
  existingNotificationId, localDiseaseProblem, officialNumberProblem, previousEpiWeek, reasonProblem, residenceProblem, shiftEpiWeek,
  statusLabel, surveillanceError, surveillanceRoles, symptomsProblem, weekLabel, weekProblem, weekRangeProblem
} from "./surveillance";

const user = (roles: string[], features: string[] = [ "surveillance" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });
const refused = (status: number, body: unknown) => new ApiError(status, body, "x");

describe("portão e papéis", () => {
  it("só com o interruptor e nunca para o operador; a sugestão exige também o prontuário", () => {
    expect(canUseSurveillance(user([ "surveillance" ]))).toBe(true);
    expect(canUseSurveillance(user([ "surveillance" ], []))).toBe(false);
    expect(canUseSurveillance(user([], [ "surveillance" ], true))).toBe(false);
    expect(canSuggestNotification(user([ "health_professional" ], [ "surveillance" ]))).toBe(false);
    expect(canSuggestNotification(user([ "health_professional" ], [ "surveillance", "clinical_record" ]))).toBe(true);
    expect(canSuggestNotification(user([ "citizen_verifier" ], [ "surveillance", "clinical_record" ]))).toBe(false);
    expect(surveillanceRoles(user([ "analyst" ]))).toEqual({ surveillance: false, analyst: true, professional: false, admin: false });
  });
  it("CBO que notifica: médico 225*, enfermeiro 2235*; técnico e dentista não", () => {
    expect(cboCanNotify("225142")).toBe(true);
    expect(cboCanNotify("223505")).toBe(true);
    expect(cboCanNotify("322205")).toBe(false);
    expect(cboCanNotify("223208")).toBe(false);
    expect(cboCanNotify(null)).toBe(false);
  });
});

describe("rótulos e transições (D5)", () => {
  it("código desconhecido aparece cru", () => {
    expect(statusLabel("possible")).toBe("possível notificação não feita");
    expect(statusLabel("entered")).toBe("digitada no sistema oficial");
    expect(statusLabel("reopened")).toBe("reopened");
  });
  it("próximos passos por situação", () => {
    expect(NEXT.possible).toEqual([ "notified" ]);
    expect(NEXT.notified).toEqual([ "investigating" ]);
    expect(NEXT.investigating).toEqual([ "closed" ]);
    expect(NEXT.closed).toEqual([ "entered" ]);
    expect(NEXT.entered).toEqual([]);
    expect(NEXT.dismissed).toEqual([]);
  });
});

describe("semana epidemiológica: domingo a sábado, 4 de janeiro, semana 53 (D16)", () => {
  it("datas conhecidas", () => {
    expect(epiWeekOf("2026-10-27")).toBe("2026-43");
    expect(epiWeekOf("2026-10-25")).toBe("2026-43");
    expect(epiWeekOf("2026-10-24")).toBe("2026-42");
    expect(epiWeekOf("2026-01-04")).toBe("2026-01");
    expect(epiWeekOf("2026-01-03")).toBe("2025-53");
    expect(epiWeekOf("2024-12-29")).toBe("2025-01");
    expect(epiWeekOf("2024-12-28")).toBe("2024-52");
    expect(epiWeekOf("2022-01-01")).toBe("2021-52");
    expect(epiWeekOf("2021-01-01")).toBe("2020-53");
  });
  it("início, deslocamento, anterior e rótulo", () => {
    expect(epiWeekStart("2026-43")).toBe("2026-10-25");
    expect(epiWeekStart("2025-53")).toBe("2025-12-28");
    expect(shiftEpiWeek("2026-01", -1)).toBe("2025-53");
    expect(shiftEpiWeek("2025-53", 1)).toBe("2026-01");
    expect(shiftEpiWeek("2026-43", -7)).toBe("2026-36");
    expect(previousEpiWeek("2026-01-05")).toBe("2025-53");
    expect(weekLabel("2026-43")).toBe("semana 43/2026 (25/10 a 31/10)");
  });
  it("formato, semana encerrada e faixa do painel (D7, D8)", () => {
    expect(weekProblem("2026-43", "2026-10-27")).toBeNull();
    expect(weekProblem("2026-54", "2026-10-27")).toBe("semana inválida: use AAAA-SS (ex.: 2026-43)");
    expect(weekProblem("2026-44", "2026-10-27")).toBe("semana no futuro");
    expect(weekProblem("2026-4", "2026-10-27")).toBe("semana inválida: use AAAA-SS (ex.: 2026-43)");
    expect(closedWeekProblem("2026-42", "2026-10-27")).toBeNull();
    expect(closedWeekProblem("2026-43", "2026-10-27")).toBe("a semana ainda não terminou");
    expect(weekRangeProblem("2026-36", "2026-43", "2026-10-27")).toBeNull();
    expect(weekRangeProblem("2026-43", "2026-36", "2026-10-27")).toBe("a semana inicial vem depois da final");
    expect(weekRangeProblem("2025-42", "2026-43", "2026-10-27")).toBe("no máximo 53 semanas por consulta");
    expect(weekRangeProblem("2025-44", "2026-43", "2026-10-27")).toBeNull();
  });
});

describe("formato: recusar, nunca cortar", () => {
  it("datas", () => {
    expect(symptomsProblem("", "2026-10-27", "2026-10-27")).toBeNull();
    expect(symptomsProblem("2026-10-28", "2026-10-27", "2026-10-27")).toBe("data dos primeiros sintomas depois de hoje");
    expect(symptomsProblem("2026-10-26", "2026-10-25", "2026-10-27")).toBe("primeiros sintomas depois da data da suspeita");
  });
  it("número oficial, residência, motivo", () => {
    expect(officialNumberProblem("1234567")).toBeNull();
    expect(officialNumberProblem("")).toBe("informe o número da notificação no sistema oficial");
    expect(officialNumberProblem("2026/0001234-5")).toBeNull();
    expect(officialNumberProblem("12 45")).toBe("número oficial de 1 a 30 caracteres: letras, números, ponto, barra ou hífen");
    expect(officialNumberProblem("1".repeat(31))).toBe("número oficial de 1 a 30 caracteres: letras, números, ponto, barra ou hífen");
    expect(residenceProblem(false, "")).toBeNull();
    expect(residenceProblem(true, "4106902")).toBeNull();
    expect(residenceProblem(true, "410690")).toBe("código IBGE do município de residência com 7 dígitos");
    expect(reasonProblem("", true)).toBe("informe o motivo");
    expect(reasonProblem("ok", true)).toBe("motivo de 3 a 500 caracteres");
    expect(reasonProblem("", false)).toBeNull();
    expect(reasonProblem("x".repeat(501), false)).toBe("motivo com mais de 500 caracteres");
    expect(reasonProblem("ç".repeat(500), true)).toBeNull();
  });
  it("doença local (D4)", () => {
    const ok = { code: "local_febre_x", name: "Febre X (vigilância estadual)", deadline: "immediate", cid10: "A93.8", ciap2: "", repeat_after_days: "30" };
    expect(codesOf(" a93.8, b05 ;A90 a90")).toEqual([ "A938", "B05", "A90" ]);
    expect(localDiseaseProblem(ok, true)).toBeNull();
    expect(localDiseaseProblem({ ...ok, code: "febre_x" }, true)).toBe("código começa com local_ e segue com letras minúsculas, números ou _ (até 46)");
    expect(localDiseaseProblem({ ...ok, code: "local_Febre" }, true)).toBe("código começa com local_ e segue com letras minúsculas, números ou _ (até 46)");
    expect(localDiseaseProblem({ ...ok, code: "" }, false)).toBeNull();
    expect(localDiseaseProblem({ ...ok, name: "Fe" }, true)).toBe("nome de 3 a 120 caracteres");
    expect(localDiseaseProblem({ ...ok, cid10: "A9" }, true)).toBe("CID-10 inválido: A9");
    expect(localDiseaseProblem({ ...ok, ciap2: "A7" }, true)).toBe("CIAP-2 inválido: A7");
    expect(localDiseaseProblem({ ...ok, repeat_after_days: "0" }, true)).toBe("janela de repetição de 1 a 36500 dias");
    expect(localDiseaseProblem({ ...ok, repeat_after_days: "30,5" }, true)).toBe("janela de repetição de 1 a 36500 dias");
  });
});

describe("frases das recusas", () => {
  it("códigos do contrato viram frase; nada do enviado volta", () => {
    expect(surveillanceError(refused(403, { error: "feature_disabled", feature: "surveillance" }))).toBe(SURVEILLANCE_DISABLED);
    expect(surveillanceError(refused(409, { error: "already_notified", notification_id: "nt9" })))
      .toBe("já existe notificação desta doença em aberto (ou recente) para o paciente");
    expect(existingNotificationId(refused(409, { error: "already_notified", notification_id: "nt9" }))).toBe("nt9");
    expect(existingNotificationId(refused(409, { error: "already_notified" }))).toBeNull();
    expect(surveillanceError(refused(403, { error: "cbo_not_allowed" }))).toBe("só médico ou enfermeiro notifica pela consulta");
    expect(surveillanceError(refused(422, { error: "classification_required" }))).toBe("escolha a classificação final para encerrar");
    expect(surveillanceError(refused(422, { error: "official_number_required" }))).toBe("informe o número da notificação no sistema oficial");
    expect(surveillanceError(refused(409, { error: "unit_has_notifications" }))).toBe("a unidade tem notificação nesta semana; a negativa não cabe");
    expect(surveillanceError(refused(422, { error: "future_week" }))).toBe("a semana ainda não terminou");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/lib/surveillance.test.ts`
Expected: FAIL — o módulo não existe.

- [ ] **Step 3: Implemente `src/lib/surveillance.ts`**

Cabeçalho: "Regras de tela da vigilância epidemiológica (módulo 25; ADR 0043; contrato 2026-10-27 §1–§10). Prazos, transições válidas, supressão e sinal são do api; aqui só se traduzem os códigos e se recusa o que o api recusaria por formato. Nada aqui guarda dado nem escreve em log; a frase de erro nunca repete o enviado."

Valores fixados:
- `SURVEILLANCE_DISABLED = "a vigilância epidemiológica está desligada nesta cidade"`, `SURVEILLANCE_NO_ROLE = "seu papel não permite usar a vigilância epidemiológica"`, `STALE_TEXT = "outra pessoa alterou esta notificação enquanto você editava — recarregue para ver a versão nova"`, `NO_NAME = "pessoa sem nome no cadastro"`, `NO_NEIGHBORHOOD = "não informado"`, `SIGNAL_NOTE = "Sinal para avaliar: a semana passou do dobro da média das 4 semanas anteriores, com pelo menos 5 casos. É um aviso para a vigilância olhar, não uma conclusão."`.
- Rótulos (mapa + fallback cru): situação `possible` "possível notificação não feita", `notified` "notificada", `investigating` "em investigação", `closed` "encerrada, sem número oficial", `entered` "digitada no sistema oficial", `dismissed` "descartada (não era caso)"; tom `possible` warn, `notified` warn, `investigating` warn, `closed` neutral, `entered` ok, `dismissed` neutral. Origem `consultation` "na consulta", `manual` "manual", `safety_net` "rede de segurança (consulta finalizada)". Classificação `confirmed` "confirmado", `discarded` "descartado", `inconclusive` "inconclusivo" (`CLASSIFICATIONS` nessa ordem). Prazo `immediate` "imediata (24 h)", `weekly` "semanal (7 dias)". Negativa `with_cases` "com casos", `negative` "negativa registrada", `pending` "pendente".
- `NEXT` como no teste; `transitionLabel`: `notified` "Confirmar notificação", `investigating` "Iniciar investigação", `closed` "Encerrar", `entered` "Registrar número oficial".
- `canSuggestNotification` = `canUseSurveillance(user) && hasFeature(user, "clinical_record") && roles.professional`. `cboCanNotify` = `/^225/` ou `/^2235/`.
- Semana: tudo em UTC sobre datas `AAAA-MM-DD` (sem fuso: a data já é a da cidade). Algoritmo:

```ts
// Semana 1 = a de domingo a sábado que contém 4 de janeiro (regra do Sinan; D16).
function week1Start(year: number): number {           // ms UTC do domingo
  const jan4 = Date.UTC(year, 0, 4);
  return jan4 - new Date(jan4).getUTCDay() * DAY;
}
export function epiWeekOf(iso: string): string {
  const t = Date.parse(`${iso}T00:00:00Z`);
  let year = Number(iso.slice(0, 4)) + 1;
  while (t < week1Start(year)) year -= 1;              // no máximo dois passos
  const n = Math.floor((t - week1Start(year)) / (7 * DAY)) + 1;
  return `${year}-${String(n).padStart(2, "0")}`;
}
```

`epiWeekStart(week)` = `week1Start(ano) + (n − 1) × 7 dias`; `shiftEpiWeek` = `epiWeekOf(início + delta × 7 dias)`; `previousEpiWeek(today)` = `shiftEpiWeek(epiWeekOf(today), -1)`; `weekLabel` com `dd/MM` do domingo e do sábado. `weekProblem`: formato `^\d{4}-\d{2}$`, número de 1 até o total do ano (`epiWeekOf(epiWeekStart(semana))` precisa voltar igual), não depois de `epiWeekOf(today)`. `closedWeekProblem`: `weekProblem` e, além, a semana precisa ser anterior à de hoje. `weekRangeProblem`: os dois válidos, `from ≤ to` (comparação de texto serve: `AAAA-SS`) e no máximo 53 semanas contando as duas pontas (`shiftEpiWeek(from, 52) ≥ to`; frase "no máximo 53 semanas por consulta").
- Formato: contagem de caracteres por `[...texto].length`; `codesOf` = maiúsculas, sem ponto, separado por espaço, vírgula ou ponto e vírgula, sem repetidos, na ordem; `localDiseaseProblem(draft: { code, name, deadline, cid10: string, ciap2: string, repeat_after_days: string }, creating)` confere, nessa ordem, código (só quando `creating`: `^local_[a-z0-9_]{1,40}$`), nome (3–120 depois de `trim`), prazo (`immediate`/`weekly`, senão "escolha o prazo"), cada CID-10 (`^[A-Z]\d{2}[0-9X]?$`), cada CIAP-2 (`^[A-Z]\d{2}$`), janela (inteiro de 1 a 36500, só dígitos). `officialNumberProblem`: vazio → "informe o número da notificação no sistema oficial"; fora de `^[0-9A-Za-z./-]{1,30}$` → a frase do teste. `reasonProblem(text, required)`: obrigatório vazio → "informe o motivo"; obrigatório com menos de 3 → "motivo de 3 a 500 caracteres"; mais de 500 → "motivo com mais de 500 caracteres".
- `surveillanceError`: `featureDisabledKey(err) !== null` → `SURVEILLANCE_DISABLED`; por `errorCode`: as frases do teste e mais `stale_version` → `STALE_TEXT`, `invalid_transition` → "essa mudança não vale para a situação atual da notificação — recarregue", `invalid_form` → "confira o campo indicado na ficha", `already_recorded` → "a negativa desta unidade e semana já foi registrada", `invalid_disease` → "doença inválida — confira os campos", `duplicate_code` → "já existe doença com este código", `unknown_disease` → "doença fora da lista vigente — recarregue a lista", `invalid_range` → "faixa de semanas inválida", `invalid_week` → "semana inválida", `invalid_residence` → "município de residência inválido (não pode ser o desta cidade)", `invalid_symptoms_on` → "data dos primeiros sintomas inválida", `invalid_unit` → "unidade inválida, inativa ou não notificadora", `invalid_reason` → "motivo de 3 a 500 caracteres", `invalid_patient` → "confira os dados da pessoa", `duplicate_suspected` → `DUPLICATE_SUSPECTED_TEXT`, `person_not_found` → "nenhuma pessoa com este CPF; cadastre a pessoa", `citizen_not_verified` → "cidadão sem identidade validada no posto"; o resto, `describeActionError(err)`. `existingNotificationId` lê `notification_id` do corpo de `already_notified`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/lib/surveillance.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/surveillance.ts src/lib/surveillance.test.ts
$G commit -m "feat: add disease surveillance screen rules and epidemiological weeks

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Vigilância", contador, portão e papel na Equipe

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Create: `src/modules/surveillance/SurveillanceGate.tsx`, `src/hooks/useSurveillanceCount.ts`, `src/modules/SurveillanceQueue.tsx`, `src/modules/SurveillanceNegatives.tsx`, `src/modules/SurveillanceReports.tsx`, `src/modules/SurveillancePanel.tsx`, `src/modules/SurveillanceDiseases.tsx` (as cinco telas nascem como portão com `EmptyState title="em construção"`; as Tasks 5, 9, 10, 11 e 12 trocam o corpo)
- Test: `src/shell/modules.surveillance.test.ts`, `src/hooks/useSurveillanceCount.test.tsx`

**Interfaces:**
- Consumes: `canUseSurveillance`, `surveillanceRoles`, `SURVEILLANCE_DISABLED`, `SURVEILLANCE_NO_ROLE`, `SURVEILLANCE_COUNT_KEY` (Task 2); `countNotifications` (Task 1).
- Produces: `ModuleId` ganha `"surveillance-queue" | "surveillance-negatives" | "surveillance-reports" | "surveillance-panel" | "surveillance-diseases"`; o grupo `{ label: "Vigilância", items: [ surveillance-queue "Fila da vigilância" ⚠, surveillance-negatives "Notificação negativa" ∅, surveillance-reports "Relatórios da vigilância" ▤, surveillance-panel "Painel por bairro e semana" ▦, surveillance-diseases "Doenças notificáveis" ☰ ] }` depois de "Cadastro territorial"; `withSurveillanceCount(groups, immediate: number): NavGroupDef[]`; `shellGroups(..., qcBlocks = 0, surveillanceImmediate = 0)`; `SURVEILLANCE_COUNT_REFRESH_MS = 60_000` e `useSurveillanceCount(enabled: boolean, userId: string | null)` (TanStack `useQuery` de `NotificationCounts`); `SurveillanceGate({ title, sub, allow(roles), children(roles) })`; `SurveillanceQueue({ today? })`, `SurveillanceNegatives({ today? })`, `SurveillanceReports({ today? })`, `SurveillancePanel({ today? })`, `SurveillanceDiseases({ onGoToSecurity? })`; o papel `surveillance` em `INVITE_ROLES` ("Vigilância epidemiológica") e em `PRIVILEGED_ROLES`.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.surveillance.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor, withSurveillanceCount } from "./modules";
import { CATALOG } from "./catalog";

const user = (roles: string[], features = [ "surveillance" ], operator = false) => ({ operator, memberships: roles.map((role) => ({ role })), features });
const itemsOf = (groups: ReturnType<typeof navGroupsFor>) => groups.find((g) => g.label === "Vigilância")?.items.map((i) => i.id) ?? [];

describe("grupo Vigilância (módulo 25)", () => {
  it("a vigilância vê tudo; o analyst só o painel; sem exigir o prontuário", () => {
    expect(itemsOf(navGroupsFor(user([ "surveillance" ])))).toEqual([
      "surveillance-queue", "surveillance-negatives", "surveillance-reports", "surveillance-panel", "surveillance-diseases" ]);
    expect(itemsOf(navGroupsFor(user([ "analyst" ])))).toEqual([ "surveillance-panel" ]);
  });
  it("some sem o interruptor, para o profissional, o admin e o operador", () => {
    expect(itemsOf(navGroupsFor(user([ "surveillance" ], [])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "health_professional", "municipal_admin" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([], undefined, true)))).toEqual([]);
  });
  it("vem depois do Cadastro territorial, com ícone e descrição", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Vigilância")).toBe(labels.indexOf("Cadastro territorial") + 1);
    expect(CATALOG.groups["Vigilância"].description).toBe("Notificação compulsória, prazos, negativa semanal e painel");
    expect(CATALOG.screens["surveillance-queue"].description).toBe("Notificações com prazo, imediatas e vencidas primeiro");
  });
  it("as imediatas abertas no rótulo da fila", () => {
    const groups = navGroupsFor(user([ "surveillance" ]));
    expect(withSurveillanceCount(groups, 0)).toBe(groups);
    const label = (n: number) => withSurveillanceCount(groups, n).find((g) => g.label === "Vigilância")!.items[0].label;
    expect(label(1)).toBe("Fila da vigilância (1 imediata)");
    expect(label(3)).toBe("Fila da vigilância (3 imediatas)");
  });
});
```

```tsx
// src/hooks/useSurveillanceCount.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, renderHook, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), countNotifications: vi.fn() }));
import * as api from "../lib/api";
import { SURVEILLANCE_COUNT_REFRESH_MS, useSurveillanceCount } from "./useSurveillanceCount";
import { counts } from "../test/surveillanceFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const wrapper = ({ children }: { children: ReactNode }) => <QueryClientProvider client={new QueryClient()}>{children}</QueryClientProvider>;

describe("useSurveillanceCount", () => {
  it("a contagem relê a cada minuto e acende o item com as imediatas", async () => {
    vi.useFakeTimers({ shouldAdvanceTime: true });
    const fn = api.countNotifications as unknown as ReturnType<typeof vi.fn>;
    fn.mockResolvedValue(counts({ immediate_open: 2 }));
    const { result } = renderHook(() => useSurveillanceCount(true, "u-1"), { wrapper });
    await waitFor(() => expect(result.current.data?.immediate_open).toBe(2));
    expect(SURVEILLANCE_COUNT_REFRESH_MS).toBe(60_000);
    await act(async () => { vi.advanceTimersByTime(60_000); });
    await waitFor(() => expect(fn).toHaveBeenCalledTimes(2));
  });
  it("desligado não lê", () => {
    const fn = api.countNotifications as unknown as ReturnType<typeof vi.fn>;
    fn.mockReset();
    renderHook(() => useSurveillanceCount(false, "u-1"), { wrapper });
    expect(fn).not.toHaveBeenCalled();
  });
});
```

Em `src/shell/modules.test.ts`, nos dois testes de "sem papel" e "sem sessão", acrescente `expect(groups.some((g) => g.label === "Vigilância")).toBe(false);` e troque `NAV_GROUPS.length - 12` por `NAV_GROUPS.length - 13` e `NAV_GROUPS.length - 11` por `NAV_GROUPS.length - 12`.

Em `src/lib/team.test.ts`, no teste dos papéis convidáveis, acrescente `"surveillance"` à lista ordenada esperada e:

```ts
    expect(INVITE_ROLES.find((r) => r.role === "surveillance")?.label).toBe("Vigilância epidemiológica");
    expect(isPrivilegedRole("surveillance")).toBe(true);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/shell src/hooks/useSurveillanceCount.test.tsx src/lib/team.test.ts`
Expected: FAIL — o grupo, o hook e o papel não existem; as contagens antigas mudam.

- [ ] **Step 3: Implemente**

- `modules.ts`: os cinco ids no `ModuleId`; o grupo no `NAV_GROUPS` depois de "Cadastro territorial"; em `navGroupsFor`, `const survOn = canUseSurveillance(user); const isSurveillance = roles.includes("surveillance");` (comentário: "Módulo 25 (ADR 0043): `surveillance`, sem exigir o prontuário; tudo do papel surveillance, o painel agregado também do analyst") e os filtros `surveillance-panel` → `survOn && (isSurveillance || isAnalyst)`, os outros quatro → `survOn && isSurveillance`. `withSurveillanceCount` no molde do `withQcBlockCount` (rótulo `(1 imediata)` / `(N imediatas)` no item `surveillance-queue`; `count <= 0` devolve o mesmo array).
- `useSurveillanceCount.ts` no molde do `useLabCriticalCount`: chave `[SURVEILLANCE_COUNT_KEY, userId]`, `queryFn: countNotifications`, `refetchInterval: enabled ? SURVEILLANCE_COUNT_REFRESH_MS : false`, `gcTime: 0`, `staleTime: 0`, `retry: false`, `refetchOnWindowFocus: false`. A fila (Task 5) usa o mesmo hook (a mesma chave: uma leitura por minuto, não duas).
- `App.tsx`: `const survQuery = useSurveillanceCount(canUseSurveillance(auth.user) && surveillanceRoles(auth.user).surveillance, auth.user?.id ?? null); const survImmediate = survQuery.data?.immediate_open ?? 0;` passado como 6º argumento de `shellGroups` (que aplica `withSurveillanceCount` por fora dos outros e acende `count` no item `surveillance-queue` quando > 0) e nas dependências do `useMemo`; os cinco `case` depois de `territory-micro-areas` (`<SurveillanceDiseases onGoToSecurity={() => setActive("security")} />`).
- `catalog.ts`: importe `IconVirus, IconBellRinging, IconCalendarX, IconFileSpreadsheet, IconGridDots, IconVirusSearch`; grupo `"Vigilância": { icon: IconVirus, description: "Notificação compulsória, prazos, negativa semanal e painel" }`; telas `"surveillance-queue": { icon: IconBellRinging, description: "Notificações com prazo, imediatas e vencidas primeiro" }`, `"surveillance-negatives": { icon: IconCalendarX, description: "Unidades sem casos na semana epidemiológica" }`, `"surveillance-reports": { icon: IconFileSpreadsheet, description: "CSV por semana e encerradas sem número oficial" }`, `"surveillance-panel": { icon: IconGridDots, description: "Casos por bairro e semana epidemiológica" }`, `"surveillance-diseases": { icon: IconVirusSearch, description: "Lista nacional e doenças da cidade" }`.
- `SurveillanceGate.tsx`: cópia do molde do `CareLinesGate` com `canUseSurveillance`/`surveillanceRoles` e as frases da Task 2.
- As cinco telas: `SurveillanceGate` com o título ("Fila da vigilância", "Notificação negativa", "Relatórios da vigilância", "Painel por bairro e semana epidemiológica", "Doenças notificáveis"), `allow` (`r.surveillance`; o painel `r.surveillance || r.analyst`) e, por enquanto, `EmptyState title="em construção"`.
- `team.ts`: depois de `billing_clerk`, `// Módulo 25 (ADR 0043): fila nominal da vigilância; privilegiado (step-up para conceder).` e `{ role: "surveillance", label: "Vigilância epidemiológica" }`; `"surveillance"` em `PRIVILEGED_ROLES`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/shell src/hooks src/lib/team.test.ts src/App.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.surveillance.test.ts src/shell/catalog.ts src/App.tsx src/lib/team.ts src/lib/team.test.ts src/hooks/useSurveillanceCount.ts src/hooks/useSurveillanceCount.test.tsx src/modules/surveillance/SurveillanceGate.tsx src/modules/SurveillanceQueue.tsx src/modules/SurveillanceNegatives.tsx src/modules/SurveillanceReports.tsx src/modules/SurveillancePanel.tsx src/modules/SurveillanceDiseases.tsx
$G commit -m "feat: add the disease surveillance menu group and role

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Aviso e "Notificar suspeita" na consulta (F-25.2)

**Files:**
- Create: `src/modules/consultation/NotificationSuggestions.tsx`
- Modify: `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationEditor.test.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/consultation/NotificationSuggestions.test.tsx`, `src/modules/consultation/ConsultationWorkspace.surveillance.test.tsx`

**Interfaces:**
- Consumes: `getNotificationSuggestions`, `createConsultationNotification`, `listSurveillanceDiseases` (Task 1); `canSuggestNotification`, `cboCanNotify`, `deadlineLabel`, `symptomsProblem`, `surveillanceError`, `SUGGESTIONS_KEY`, `DISEASES_KEY`, `NOMINAL_QUERY`, `STATIC_QUERY` (Task 2); `consultation`, `options`, `record` (`consultationFixtures`); `DENGUE`, `MEASLES` (Task 1).
- Produces: `NotificationSuggestions({ consultationId, problemsKey, today, finalizing })`; `ConsultationEditor` ganha a prop `notifications?: boolean`; `SUGGESTION_PENDING_NOTE` (exportada da peça).

Comportamento: a peça só existe com `notifications` verdadeiro (o `ConsultationWorkspace` passa `canSuggestNotification(user) && cboCanNotify(consultation.cbo_code)`). Lê as sugestões com `NOMINAL_QUERY` pela chave `[SUGGESTIONS_KEY, consultationId, problemsKey]`: o api lê o **rascunho salvo**, então o editor só troca `problemsKey` quando o autosave assentou — `addedKey` = os `terminology:code` dos problemas com `action: "add"`, ordenados e unidos por vírgula; `const [ problemsKey, setProblemsKey ] = useState(addedKey)` e `useEffect(() => { if (!autosave.pending && autosave.status.kind !== "saving") setProblemsKey(addedKey); }, [ addedKey, autosave.pending, autosave.status.kind ])`. Cada sugestão: "CID de notificação obrigatória (A90): notificar suspeita de Dengue", a `Tag` do prazo quando imediata, "Primeiros sintomas (opcional)" (`type="date"`) e o botão "Notificar suspeita de Dengue". "Notificar outra suspeita" abre a escolha da doença (a lista da cidade, `STATIC_QUERY`) com a mesma data — o caminho do enfermeiro com CIAP-2 sem correspondência e de quem quer notificar sem CID. O corpo leva `consultation_id`, `disease_code` e `symptoms_on` só quando preenchida (conferida por `symptomsProblem` com a suspeita = hoje). Sucesso: `role="status"` "Suspeita de Dengue notificada à vigilância." e relê as sugestões. Toda recusa fica na peça (`role="alert"`); 409 `already_notified` também tira a sugestão da tela. Com `finalizing` e sugestão pendente, a peça mostra `SUGGESTION_PENDING_NOTE` = "Há CID de notificação obrigatória sem notificação. Você pode notificar agora ou finalizar assim mesmo: a vigilância receberá o caso como possível notificação não feita." No editor, a peça fica **depois** de `ExamRequestsField` e **fora** de `finalizeProblems`/`missing`/`blockedFinal`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/NotificationSuggestions.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, getNotificationSuggestions: vi.fn(), createConsultationNotification: vi.fn(), listSurveillanceDiseases: vi.fn() };
});
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { NotificationSuggestions, SUGGESTION_PENDING_NOTE } from "./NotificationSuggestions";
import { DENGUE, MEASLES, TODAY25, notification } from "../../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const mount = (finalizing = false, key = "cid10:A90") => render(
  <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
    <NotificationSuggestions consultationId="cs1" problemsKey={key} today={TODAY25} finalizing={finalizing} />
  </QueryClientProvider>);

describe("NotificationSuggestions", () => {
  beforeEach(() => {
    for (const fn of [ api.getNotificationSuggestions, api.createConsultationNotification, api.listSurveillanceDiseases ]) mocked(fn).mockReset();
    mocked(api.getNotificationSuggestions).mockResolvedValue([ { disease: DENGUE, matched_code: "A90" } ]);
    mocked(api.listSurveillanceDiseases).mockResolvedValue([ DENGUE, MEASLES ]);
    mocked(api.createConsultationNotification).mockResolvedValue(notification());
  });

  it("sugestão do CID: notifica com a data dos sintomas; o corpo leva só ids, código e data", async () => {
    mount();
    expect(await screen.findByText("CID de notificação obrigatória (A90): notificar suspeita de Dengue")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Primeiros sintomas (opcional)"), { target: { value: "2026-10-24" } });
    fireEvent.click(screen.getByRole("button", { name: "Notificar suspeita de Dengue" }));
    expect((await screen.findByRole("status")).textContent).toBe("Suspeita de Dengue notificada à vigilância.");
    expect(mocked(api.createConsultationNotification).mock.calls[0][0]).toEqual({ consultation_id: "cs1", disease_code: "dengue", symptoms_on: "2026-10-24" });
    await waitFor(() => expect(mocked(api.getNotificationSuggestions)).toHaveBeenCalledTimes(2));
  });

  it("data dos sintomas no futuro é recusada antes de enviar", async () => {
    mount();
    fireEvent.change(await screen.findByLabelText("Primeiros sintomas (opcional)"), { target: { value: "2026-10-28" } });
    fireEvent.click(screen.getByRole("button", { name: "Notificar suspeita de Dengue" }));
    expect((await screen.findByRole("alert")).textContent).toBe("data dos primeiros sintomas depois de hoje");
    expect(mocked(api.createConsultationNotification)).not.toHaveBeenCalled();
  });

  it("409 already_notified: frase e a sugestão some", async () => {
    mocked(api.createConsultationNotification).mockRejectedValue(new ApiError(409, { error: "already_notified" }, "x"));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Notificar suspeita de Dengue" }));
    expect((await screen.findByRole("alert")).textContent).toBe("já existe notificação desta doença em aberto (ou recente) para o paciente");
    expect(screen.queryByRole("button", { name: "Notificar suspeita de Dengue" })).toBeNull();
  });

  it("Notificar outra suspeita: escolhe a doença da lista da cidade", async () => {
    mocked(api.getNotificationSuggestions).mockResolvedValue([]);
    mount(false, "");
    fireEvent.click(await screen.findByRole("button", { name: "Notificar outra suspeita" }));
    await screen.findByRole("option", { name: "Sarampo" });
    fireEvent.change(screen.getByLabelText("Doença ou agravo"), { target: { value: "sarampo" } });
    fireEvent.click(screen.getByRole("button", { name: "Notificar suspeita" }));
    await waitFor(() => expect(mocked(api.createConsultationNotification).mock.calls[0][0]).toEqual({ consultation_id: "cs1", disease_code: "sarampo" }));
  });

  it("finalizando com sugestão pendente: o aviso diz que a vigilância verá como possível", async () => {
    mount(true);
    expect(await screen.findByText(SUGGESTION_PENDING_NOTE)).not.toBeNull();
  });
});
```

Em `src/modules/consultation/ConsultationEditor.test.tsx`, acrescente `getNotificationSuggestions: vi.fn(), createConsultationNotification: vi.fn(), listSurveillanceDiseases: vi.fn()` ao `vi.mock`, e no fim:

```tsx
describe("ConsultationEditor — notificação (módulo 25)", () => {
  beforeEach(() => {
    for (const fn of [ api.saveConsultationDraft, api.finalizeConsultation, api.getNotificationSuggestions ]) mocked(fn).mockReset();
    mocked(api.saveConsultationDraft).mockImplementation(async () => consultation());
    mocked(api.finalizeConsultation).mockResolvedValue(finalized());
    mocked(api.listSurveillanceDiseases).mockResolvedValue([]);
  });
  const mount = () => render(<QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
    <ConsultationEditor consultation={consultation()} record={record()} options={options()} unit={unit} units={units} notifications
      autosaveDelayMs={0} searchDelayMs={0} onFinalized={vi.fn()} onLocked={vi.fn()} /></QueryClientProvider>);

  it.each([
    [ "pendente", () => mocked(api.getNotificationSuggestions).mockResolvedValue([ { disease: { code: "dengue", name: "Dengue", deadline: "weekly", cid10: [ "A90" ],
      ciap2: [], repeat_after_days: 30, sinan_form: null, local: false, list_version: 1 }, matched_code: "A90" } ]) ],
    [ "leitura que falha", () => mocked(api.getNotificationSuggestions).mockRejectedValue(new ApiError(500, {}, "x")) ]
  ])("sugestão %s: a finalização segue igual", async (_, arrange) => {
    arrange();
    mount();
    fillMinimum();
    fireEvent.click(screen.getByRole("button", { name: "Finalizar consulta" }));
    const confirm = screen.getByRole("button", { name: "Confirmar finalização" }) as HTMLButtonElement;
    expect(confirm.disabled).toBe(false);
    fireEvent.click(confirm);
    await waitFor(() => expect(mocked(api.finalizeConsultation)).toHaveBeenCalledTimes(1));
  });
});
```

(`fillMinimum` e o caminho até "Confirmar finalização" são os dos testes de finalização que já existem no arquivo.)

```tsx
// src/modules/consultation/ConsultationWorkspace.surveillance.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn(), startConsultation: vi.fn(),
    getConsultation: vi.fn(), getConsultationProblems: vi.fn() };
});
vi.mock("./ConsultationEditor", () => ({ ConsultationEditor: (p: { notifications?: boolean }) => <span>{`editor ${p.notifications ? "com" : "sem"} notificação`}</span> }));

import * as api from "../../lib/api";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";
import { consultation, options, problem, record } from "../../test/consultationFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const unit = { id: "u1", name: "UBS Centro", kind: "ubs" };

describe("ConsultationWorkspace — notificação (módulo 25)", () => {
  beforeEach(() => {
    mocked(api.getAttendanceRecord).mockResolvedValue(record());
    mocked(api.getConsultationOptions).mockResolvedValue(options());
    mocked(api.getConsultationProblems).mockResolvedValue([ problem() ]);
  });
  const open = async (features: string[], cbo = "225142") => {
    mocked(api.startConsultation).mockResolvedValue(consultation({ cbo_code: cbo }));
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "health_professional" ], { id: "us1", features }));
    renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} onClose={() => undefined} onFinalized={() => undefined} />);
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
  };

  it("médico com surveillance e prontuário: o editor recebe a notificação", async () => {
    await open([ "clinical_record", "surveillance" ]);
    expect(await screen.findByText("editor com notificação")).not.toBeNull();
  });
  it.each([
    [ "sem surveillance", [ "clinical_record" ], "225142" ],
    [ "técnico de enfermagem", [ "clinical_record", "surveillance" ], "322205" ]
  ])("%s: sem notificação", async (_, features, cbo) => {
    await open(features, cbo);
    expect(await screen.findByText("editor sem notificação")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/consultation/NotificationSuggestions.test.tsx src/modules/consultation/ConsultationEditor.test.tsx src/modules/consultation/ConsultationWorkspace.surveillance.test.tsx`
Expected: FAIL — a peça e a prop não existem.

- [ ] **Step 3: Implemente** `NotificationSuggestions` como descrito, a prop `notifications` no `ConsultationEditor` (comentário: "Módulo 25 (ADR 0043): a notificação nunca bloqueia a finalização — fora de `missing`") e, no `ConsultationWorkspace`, `notifications={canSuggestNotification(user) && cboCanNotify(consultation.cbo_code)}` ao lado de `examPriority`. Estilos de `src/modules/exams/styles.ts` (`box`, `row`, `fieldLabel`, `muted`, `alert`, `okText`); botões de `formStyles`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/consultation && cd $W && npx tsc --noEmit`
Expected: PASS (todos os testes antigos da consulta seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/consultation/NotificationSuggestions.tsx src/modules/consultation/NotificationSuggestions.test.tsx src/modules/consultation/ConsultationEditor.tsx src/modules/consultation/ConsultationEditor.test.tsx src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationWorkspace.surveillance.test.tsx
$G commit -m "feat: suggest notifiable disease suspicions in the consultation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Fila da vigilância — abas, prazo, filtros (F-25.4)

**Files:**
- Modify: `src/modules/SurveillanceQueue.tsx`
- Create: `src/modules/surveillance/NotificationTable.tsx`
- Test: `src/modules/SurveillanceQueue.test.tsx`

**Interfaces:**
- Consumes: `listNotifications`, `listSurveillanceDiseases`, `listSurveillanceUnits` (Task 1); `useSurveillanceCount`, `SurveillanceGate` (Task 3); `NOMINAL_QUERY`, `STATIC_QUERY`, `NOTIFICATIONS_KEY`, `statusLabel`, `statusTone`, `deadlineLabel`, `NO_NAME`, `NO_NEIGHBORHOOD`, `surveillanceError` (Task 2); `documentText` (`patientDocument.ts`); `fmtDateTime` (`format.ts`).
- Produces: `NotificationTable({ rows, onOpen(id) })` (colunas: Prazo — `Tag` "vencida" `down` e "imediata" `warn` antes de `fmtDateTime(due_at)` —, Pessoa `card: "title"` com o documento, Doença, Unidade, Bairro, Situação, ação "Abrir" `card: "actions"`); `QUEUE_TABS` (`possible` "Possíveis não feitas", `notified` "Notificadas", `investigating` "Em investigação", `overdue` "Vencidas", `closed` "Sem número oficial", `entered` "Digitadas", `dismissed` "Descartadas"); `SurveillanceQueue({ today? })` com o detalhe e a notificação manual entrando nas Tasks 6 e 7 por estado (`openId`, `manual`).

Comportamento: banner `role="alert"` quando `immediate_open > 0` ou `overdue > 0` ("1 notificação imediata aberta · 3 vencidas"); abas `role="tab"` com o número da contagem nas cinco primeiras (`without_official_number` em "Sem número oficial"); aba inicial "Notificadas"; a aba "Vencidas" chama com `overdue: true` e sem `status`; filtros de doença (lista da cidade) e unidade (D2) em `<select>`; a lista em `useQuery` com `NOMINAL_QUERY` e chave `[NOTIFICATIONS_KEY, aba, doença, unidade]`; "Carregar mais" com `next_before` (clique; as páginas extras ficam num estado que zera quando a chave muda). A ordem é a do api (imediatas e vencidas primeiro): a tela não reordena.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SurveillanceQueue.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { focusManager, onlineManager } from "@tanstack/react-query";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listNotifications: vi.fn(), countNotifications: vi.fn(), listSurveillanceDiseases: vi.fn(),
    listSurveillanceUnits: vi.fn(), getNotification: vi.fn() };
});
import * as api from "../lib/api";
import type { NotificationQuery } from "../lib/api";
import { SurveillanceQueue } from "./SurveillanceQueue";
import { SURVEILLANCE_DISABLED, SURVEILLANCE_NO_ROLE } from "../lib/surveillance";
import { renderWithProviders } from "../test/campaignFixtures";
import { DENGUE, MEASLES, TODAY25, UBS_CENTRO, counts, notification, survUser } from "../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const queries = () => mocked(api.listNotifications).mock.calls.map((c) => c[0] as NotificationQuery);

describe("SurveillanceQueue", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchCurrentSession, api.listNotifications, api.countNotifications, api.getNotification ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser());
    mocked(api.countNotifications).mockResolvedValue(counts());
    mocked(api.listSurveillanceDiseases).mockResolvedValue([ DENGUE, MEASLES ]);
    mocked(api.listSurveillanceUnits).mockResolvedValue([ { ...UBS_CENTRO, kind: "ubs", notifying: true } ]);
    mocked(api.listNotifications).mockResolvedValue({ notifications: [
      notification({ id: "nt2", disease: { code: "sarampo", name: "Sarampo", deadline: "immediate" }, overdue: true }), notification() ], next_before: "c1" });
  });

  it("abas com as contagens, alerta das imediatas e vencidas, a ordem do api", async () => {
    renderWithProviders(<SurveillanceQueue today={TODAY25} />);
    expect((await screen.findByRole("alert")).textContent).toBe("1 notificação imediata aberta · 3 vencidas");
    expect(screen.getByRole("tab", { name: "Possíveis não feitas (2)" })).not.toBeNull();
    expect(screen.getByRole("tab", { name: "Sem número oficial (4)" })).not.toBeNull();
    const rows = await screen.findAllByRole("row", { name: /Joana Lima/ });
    expect(within(rows[0]).getByText("vencida")).not.toBeNull();
    expect(within(rows[0]).getByText("imediata")).not.toBeNull();
    expect(within(rows[0]).getByText("Sarampo")).not.toBeNull();
    expect(queries()[0]).toEqual({ status: "notified", disease: null, unitId: null });
  });

  it("aba Vencidas pede overdue sem situação; filtros por doença e unidade; carregar mais", async () => {
    renderWithProviders(<SurveillanceQueue today={TODAY25} />);
    fireEvent.click(await screen.findByRole("tab", { name: "Vencidas (3)" }));
    await waitFor(() => expect(queries().at(-1)).toEqual({ overdue: true, disease: null, unitId: null }));
    fireEvent.change(await screen.findByLabelText("Doença"), { target: { value: "dengue" } });
    fireEvent.change(screen.getByLabelText("Unidade"), { target: { value: "u1" } });
    await waitFor(() => expect(queries().at(-1)).toEqual({ overdue: true, disease: "dengue", unitId: "u1" }));
    fireEvent.click(await screen.findByRole("button", { name: "Carregar mais" }));
    await waitFor(() => expect(queries().at(-1)).toEqual({ overdue: true, disease: "dengue", unitId: "u1", before: "c1" }));
  });

  it("volta à aba e reconecta: a lista não relê", async () => {
    renderWithProviders(<SurveillanceQueue today={TODAY25} />);
    await screen.findAllByRole("row", { name: /Joana Lima/ });
    act(() => { focusManager.setFocused(false); focusManager.setFocused(true); onlineManager.setOnline(false); onlineManager.setOnline(true); });
    await new Promise((r) => setTimeout(r, 20));
    expect(mocked(api.listNotifications)).toHaveBeenCalledTimes(1);
  });

  it("sem o interruptor ou sem o papel: só a frase, nada é lido", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser([ "surveillance" ], { features: [] }));
    renderWithProviders(<SurveillanceQueue today={TODAY25} />);
    expect(await screen.findByText(SURVEILLANCE_DISABLED)).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser([ "analyst" ]));
    renderWithProviders(<SurveillanceQueue today={TODAY25} />);
    expect(await screen.findByText(SURVEILLANCE_NO_ROLE)).not.toBeNull();
    expect(mocked(api.listNotifications)).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceQueue.test.tsx`
Expected: FAIL — a tela é o portão "em construção".

- [ ] **Step 3: Implemente** `NotificationTable` e `SurveillanceQueue` como descritos (a lista nunca relê sozinha; só a contagem, pelo hook da Task 3). "Abrir" põe o id em `openId` (a Task 6 desenha o detalhe; até lá, nada).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceQueue.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/SurveillanceQueue.tsx src/modules/surveillance/NotificationTable.tsx src/modules/SurveillanceQueue.test.tsx
$G commit -m "feat: add the surveillance queue with deadlines and filters

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Detalhe — andamento, número oficial, descarte de possível e ficha em PDF (F-25.5, F-25.7)

**Files:**
- Create: `src/modules/surveillance/NotificationDetail.tsx`, `src/modules/surveillance/TransitionForm.tsx`
- Modify: `src/modules/SurveillanceQueue.tsx`
- Test: `src/modules/surveillance/NotificationDetail.test.tsx`

**Interfaces:**
- Consumes: `getNotification`, `transitionNotification`, `dismissNotification`, `fetchNotificationPdf` (Task 1); `NEXT`, `transitionLabel`, `CLASSIFICATIONS`, `classificationLabel`, `statusLabel`, `statusTone`, `originLabel`, `deadlineLabel`, `officialNumberProblem`, `reasonProblem`, `surveillanceError`, `isStale`, `STALE_TEXT`, `NOMINAL_QUERY`, `NOTIFICATION_KEY`, `NOTIFICATIONS_KEY`, `SURVEILLANCE_COUNT_KEY`, `NO_NEIGHBORHOOD` (Task 2); `saveFile` (`src/modules/lab/saveFile.ts`); `KeyValue`, `Tag`, `patientDocument`.
- Produces: `NotificationDetail({ id, onClose, onChanged? , formSlot? })` — `formSlot?: (n: SurveillanceNotification, onSaved: (n: SurveillanceNotification) => void) => ReactNode` (a Task 8 o preenche); `TransitionForm({ notification, onDone(n) })`.

Comportamento: lê com `NOMINAL_QUERY` (`[NOTIFICATION_KEY, id]`). Cabeçalho: doença, `Tag` do prazo, `Tag` da situação, "vencida" quando `overdue`. `KeyValue`: Pessoa, Documento (`patientDocument`), Bairro (`neighborhood?.name ?? NO_NEIGHBORHOOD`), Residência ("residente em outro município (IBGE 4106902)" ou "no município"), Unidade ("UBS Centro · CNES 2384299"), Notificador ("—" sem), Origem, Suspeita em, Primeiros sintomas, Prazo (`fmtDateTime(due_at)`), Classificação, Número oficial. Andamento: lista "notificada · 26/10/2026 10:00 · Dra. Helena Prado" (+ " · <motivo>"; sem autor, "sistema"). `TransitionForm`: um botão por próximo passo (`NEXT[status]`); escolhido, mostra só o que ele pede — "encerrar" pede a classificação (`<select>` obrigatório), "registrar número oficial" pede o número —, o motivo opcional (até 500) e "Confirmar"; o corpo leva só `to`, o que foi pedido, `reason` quando preenchido e `version`. Em `possible`, além de "Confirmar notificação", "Descartar possível" pede o motivo (obrigatório) e chama `dismissNotification`. Sucesso: `setQueryData` com a resposta, invalida `NOTIFICATIONS_KEY` e `SURVEILLANCE_COUNT_KEY`, `role="status"` "Andamento registrado." 409 `stale_version`: `STALE_TEXT`, o formulário fica preenchido e aparece "Recarregar" (relê o detalhe). "Ficha em PDF": `fetchNotificationPdf(id)` → `saveFile(file, file.filename ?? \`ficha-notificacao-${id}.pdf\`)`; erro na frase. Na fila, `openId` mostra o detalhe acima da tabela com "Fechar".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/surveillance/NotificationDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getNotification: vi.fn(), transitionNotification: vi.fn(), dismissNotification: vi.fn(),
    fetchNotificationPdf: vi.fn() };
});
vi.mock("../lab/saveFile", () => ({ saveFile: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { saveFile } from "../lab/saveFile";
import { NotificationDetail } from "./NotificationDetail";
import { STALE_TEXT } from "../../lib/surveillance";
import { renderWithProviders } from "../../test/campaignFixtures";
import { notification, survUser } from "../../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const open = () => renderWithProviders(<NotificationDetail id="nt1" onClose={vi.fn()} />);

describe("NotificationDetail", () => {
  beforeEach(() => {
    for (const fn of [ api.getNotification, api.transitionNotification, api.dismissNotification, api.fetchNotificationPdf ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser());
    mocked(api.getNotification).mockResolvedValue(notification({ status: "investigating", version: 4 }));
  });

  it("mostra a pessoa, o prazo, o bairro e o andamento", async () => {
    mocked(api.getNotification).mockResolvedValue(notification({ resident_elsewhere: true, residence_ibge: "4106902", neighborhood: null }));
    open();
    expect(await screen.findByText("Dengue")).not.toBeNull();
    expect(screen.getByText("residente em outro município (IBGE 4106902)")).not.toBeNull();
    expect(screen.getByText("não informado")).not.toBeNull();
    expect(screen.getByText("UBS Centro · CNES 2384299")).not.toBeNull();
    expect(screen.getByText("notificada · 26/10/2026 10:00 · Dra. Helena Prado")).not.toBeNull();
  });

  it("encerrar pede a classificação; o corpo leva só to, classificação e versão", async () => {
    mocked(api.transitionNotification).mockResolvedValue(notification({ status: "closed", classification: "confirmed", version: 5 }));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("escolha a classificação final para encerrar");
    fireEvent.change(screen.getByLabelText("Classificação final"), { target: { value: "confirmed" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.transitionNotification).mock.calls[0]).toEqual([ "nt1", { to: "closed", classification: "confirmed", version: 4 } ]));
    expect((await screen.findByRole("status")).textContent).toBe("Andamento registrado.");
  });

  it("número oficial: formato antes; vai com a transição para entered", async () => {
    mocked(api.getNotification).mockResolvedValue(notification({ status: "closed", classification: "confirmed", version: 6 }));
    mocked(api.transitionNotification).mockResolvedValue(notification({ status: "entered", official_number: "1234567", version: 7 }));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Registrar número oficial" }));
    fireEvent.change(screen.getByLabelText("Número da notificação no sistema oficial"), { target: { value: "12 34" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("número oficial de 1 a 30 caracteres: letras, números, ponto, barra ou hífen");
    fireEvent.change(screen.getByLabelText("Número da notificação no sistema oficial"), { target: { value: "1234567" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.transitionNotification).mock.calls[0][1]).toEqual({ to: "entered", official_number: "1234567", version: 6 }));
  });

  it("409 stale_version na transição: frase, o formulário fica, recarregar relê", async () => {
    mocked(api.transitionNotification).mockRejectedValue(new ApiError(409, { error: "stale_version" }, "x"));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Encerrar" }));
    fireEvent.change(screen.getByLabelText("Classificação final"), { target: { value: "discarded" } });
    fireEvent.change(screen.getByLabelText("Motivo (opcional)"), { target: { value: "exame negativo" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    expect((await screen.findByRole("alert")).textContent).toBe(STALE_TEXT);
    expect((screen.getByLabelText("Motivo (opcional)") as HTMLTextAreaElement).value).toBe("exame negativo");
    expect(mocked(api.transitionNotification)).toHaveBeenCalledTimes(1);
    fireEvent.click(screen.getByRole("button", { name: "Recarregar" }));
    await waitFor(() => expect(mocked(api.getNotification)).toHaveBeenCalledTimes(2));
  });

  it("possível: descartar exige motivo; confirmar vira notificada", async () => {
    mocked(api.getNotification).mockResolvedValue(notification({ status: "possible", origin: "safety_net", version: 1 }));
    mocked(api.dismissNotification).mockResolvedValue(notification({ status: "dismissed", version: 2 }));
    open();
    expect(await screen.findByRole("button", { name: "Confirmar notificação" })).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Descartar possível" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar descarte" }));
    expect((await screen.findByRole("alert")).textContent).toBe("informe o motivo");
    fireEvent.change(screen.getByLabelText("Motivo do descarte"), { target: { value: "CID registrado por engano" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar descarte" }));
    await waitFor(() => expect(mocked(api.dismissNotification).mock.calls[0]).toEqual([ "nt1", "CID registrado por engano", 1 ]));
  });

  it("ficha em PDF baixa com nome sem dado pessoal", async () => {
    mocked(api.fetchNotificationPdf).mockResolvedValue({ blob: new Blob([ "%PDF" ]), filename: null });
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Ficha em PDF" }));
    await waitFor(() => expect(mocked(saveFile).mock.calls[0][1]).toBe("ficha-notificacao-nt1.pdf"));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/surveillance/NotificationDetail.test.tsx`
Expected: FAIL — o detalhe não existe.

- [ ] **Step 3: Implemente** `NotificationDetail` e `TransitionForm` como descritos e ligue o detalhe na fila (`openId`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/surveillance src/modules/SurveillanceQueue.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/surveillance/NotificationDetail.tsx src/modules/surveillance/TransitionForm.tsx src/modules/surveillance/NotificationDetail.test.tsx src/modules/SurveillanceQueue.tsx
$G commit -m "feat: add notification detail with progress, official number and dismissal

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Notificação manual — pessoa, duplicados, doença, unidade e residência (F-25.4)

**Files:**
- Modify: `src/modules/territory/PersonStart.tsx`, `src/modules/territory/PersonStart.test.tsx`, `src/lib/territoryDictionaries.ts`, `src/modules/SurveillanceQueue.tsx`
- Create: `src/modules/surveillance/ManualNotificationPanel.tsx`
- Test: `src/modules/surveillance/ManualNotificationPanel.test.tsx`

**Interfaces:**
- Consumes: `PersonStart`, `PersonChoice`, `PersonRestart` (24a); `newPatientProblem`, `DOMAINS` (`territoryDictionaries.ts`); `lookupSurveillancePerson`, `findSurveillanceDuplicates`, `createManualNotification`, `listSurveillanceDiseases`, `listSurveillanceUnits` (Task 1); `pastDateProblem`, `symptomsProblem`, `residenceProblem`, `surveillanceError`, `existingNotificationId` (Task 2).
- Produces: em `PersonStart.tsx`, `export interface PersonSource { lookup(cpf: string): Promise<TerritoryPerson>; duplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>; describe(err: unknown): string; pickLabel(name: string): string }`, `export const TERRITORY_PERSON_SOURCE: PersonSource` (as funções e o rótulo de hoje, `Cadastrar ${name}`) e a prop `source?: PersonSource` (padrão `TERRITORY_PERSON_SOURCE`) em `PersonStart`, `ByCpf` e `WithoutCpf`; `newPatientProblem` passa a ser exportada; `SURVEILLANCE_PERSON_SOURCE: PersonSource` e `ManualNotificationPanel({ today?, onCreated(n), onOpenExisting(id), onCancel })`.

Comportamento: passo 1, `PersonStart` com `SURVEILLANCE_PERSON_SOURCE` (`describe` = `surveillanceError`; `pickLabel` = `Notificar ${name}`). Passo 2, só para `kind: "new"`: sexo ("Feminino"/"Masculino"), sem CPF o motivo (`DOMAINS.justificativa_nao_possui_cpf`) e o CNS (opcional), conferidos por `newPatientProblem` com o `path` do api (`new_patient.*`). Passo 3: doença (obrigatória, lista da cidade), data da suspeita (obrigatória, padrão `today`, não depois de hoje), primeiros sintomas (opcional, `symptomsProblem`), unidade notificadora (obrigatória, D2; opções "UBS Centro · CNES 2384299"), "Mora em outro município" e, marcado, o IBGE (`residenceProblem`). O corpo: `patient_id` | `citizen_id` | `new_patient` (+ `duplicates_reviewed` quando a escolha trouxe), `disease_code`, `suspected_on`, `symptoms_on` só quando preenchida, `unit_id`, e `resident_elsewhere`/`residence_ibge` só quando marcado. 409 `already_notified` com `notification_id` → frase e "Abrir a notificação existente" (`onOpenExisting`); 409 `duplicate_suspected` → volta ao passo 1 com `PersonRestart { query, cpf?, search: true }` (o molde do 24a). Na fila, o botão "Notificação manual" abre o painel; criado, abre o detalhe da nova.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/surveillance/ManualNotificationPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), lookupSurveillancePerson: vi.fn(), findSurveillanceDuplicates: vi.fn(),
    createManualNotification: vi.fn(), listSurveillanceDiseases: vi.fn(), listSurveillanceUnits: vi.fn(), lookupTerritoryPerson: vi.fn(), findDuplicates: vi.fn() };
});
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ManualNotificationPanel } from "./ManualNotificationPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { DENGUE, MEASLES, TODAY25, UBS_CENTRO, notification, survUser } from "../../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const panel = (over: Partial<Parameters<typeof ManualNotificationPanel>[0]> = {}) =>
  renderWithProviders(<ManualNotificationPanel today={TODAY25} onCreated={vi.fn()} onOpenExisting={vi.fn()} onCancel={vi.fn()} {...over} />);
async function fillCase() {
  await screen.findByRole("option", { name: "Sarampo" });
  await screen.findByRole("option", { name: "UBS Centro · CNES 2384299" });
  text("Doença ou agravo", "sarampo");
  text("Unidade notificadora", "u1");
}

describe("ManualNotificationPanel", () => {
  beforeEach(() => {
    for (const fn of [ api.lookupSurveillancePerson, api.findSurveillanceDuplicates, api.createManualNotification, api.lookupTerritoryPerson, api.findDuplicates ])
      mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser());
    mocked(api.listSurveillanceDiseases).mockResolvedValue([ DENGUE, MEASLES ]);
    mocked(api.listSurveillanceUnits).mockResolvedValue([ { ...UBS_CENTRO, kind: "ubs", notifying: true } ]);
    mocked(api.createManualNotification).mockResolvedValue(notification({ id: "nt9", origin: "manual" }));
  });

  it("paciente pelo CPF: manda patient_id, doença, datas e unidade; usa a busca da vigilância", async () => {
    mocked(api.lookupSurveillancePerson).mockResolvedValue({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null });
    const onCreated = vi.fn();
    panel({ onCreated });
    text("CPF", "529.982.247-25");
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Notificar Joana Lima" }));
    await fillCase();
    text("Primeiros sintomas (opcional)", "2026-10-25");
    fireEvent.click(screen.getByRole("button", { name: "Registrar notificação" }));
    await waitFor(() => expect(mocked(api.createManualNotification).mock.calls[0][0]).toEqual({
      patient_id: "pa1", disease_code: "sarampo", suspected_on: "2026-10-27", symptoms_on: "2026-10-25", unit_id: "u1" }));
    expect(mocked(api.lookupTerritoryPerson)).not.toHaveBeenCalled();
    expect(onCreated.mock.calls[0][0].id).toBe("nt9");
  });

  it("sem CPF: só cria depois da busca de duplicados; new_patient com sexo, motivo e duplicates_reviewed", async () => {
    mocked(api.findSurveillanceDuplicates).mockResolvedValue([]);
    panel();
    fireEvent.click(screen.getByLabelText("Não tem CPF"));
    text("Nome completo", "José Souza"); text("Data de nascimento", "2026-09-30"); text("Nome da mãe", "Maria Souza");
    expect(screen.queryByRole("button", { name: "Criar pessoa sem CPF" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Buscar possíveis duplicados" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar pessoa sem CPF" }));
    text("Sexo", "male"); text("Motivo de não ter CPF", "3");
    await fillCase();
    fireEvent.click(screen.getByRole("button", { name: "Registrar notificação" }));
    await waitFor(() => expect(mocked(api.createManualNotification)).toHaveBeenCalled());
    const body = mocked(api.createManualNotification).mock.calls[0][0];
    expect(body.new_patient).toEqual({ full_name: "José Souza", birth_date: "2026-09-30", mother_name: "Maria Souza", sex: "male", no_cpf_reason: "3" });
    expect(body.duplicates_reviewed).toBe(true);
  });

  it("residente em outro município exige IBGE de 7 dígitos", async () => {
    mocked(api.lookupSurveillancePerson).mockResolvedValue({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null });
    panel();
    text("CPF", "52998224725");
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Notificar Joana Lima" }));
    await fillCase();
    fireEvent.click(screen.getByLabelText("Mora em outro município"));
    text("Código IBGE do município de residência", "410690");
    fireEvent.click(screen.getByRole("button", { name: "Registrar notificação" }));
    expect((await screen.findByRole("alert")).textContent).toBe("código IBGE do município de residência com 7 dígitos");
    text("Código IBGE do município de residência", "4106902");
    fireEvent.click(screen.getByRole("button", { name: "Registrar notificação" }));
    await waitFor(() => expect(mocked(api.createManualNotification).mock.calls[0][0]).toMatchObject({ resident_elsewhere: true, residence_ibge: "4106902" }));
  });

  it("409 already_notified com notification_id: abrir a existente", async () => {
    mocked(api.lookupSurveillancePerson).mockResolvedValue({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null });
    mocked(api.createManualNotification).mockRejectedValue(new ApiError(409, { error: "already_notified", notification_id: "nt3" }, "x"));
    const onOpenExisting = vi.fn();
    panel({ onOpenExisting });
    text("CPF", "52998224725");
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Notificar Joana Lima" }));
    await fillCase();
    fireEvent.click(screen.getByRole("button", { name: "Registrar notificação" }));
    fireEvent.click(await screen.findByRole("button", { name: "Abrir a notificação existente" }));
    expect(onOpenExisting).toHaveBeenCalledWith("nt3");
  });

  it("busca e criação: CPF, nome e mãe só no corpo; nada no storage", async () => {
    mocked(api.findSurveillanceDuplicates).mockResolvedValue([]);
    panel();
    fireEvent.click(screen.getByLabelText("Não tem CPF"));
    text("Nome completo", "José Souza"); text("Data de nascimento", "2026-09-30"); text("Nome da mãe", "Maria Souza");
    fireEvent.click(screen.getByRole("button", { name: "Buscar possíveis duplicados" }));
    await screen.findByRole("button", { name: "Criar pessoa sem CPF" });
    expect(mocked(api.findSurveillanceDuplicates).mock.calls[0][0]).toEqual({ full_name: "José Souza", birth_date: "2026-09-30", mother_name: "Maria Souza" });
    expect(window.localStorage.length).toBe(0);
    expect(window.sessionStorage.length).toBe(0);
  });
});
```

(`fillCase` seleciona por rótulo; as opções de unidade são "nome · CNES". Que a URL não leva nada disso já está provado pelo cliente, Task 1.)

Em `src/modules/territory/PersonStart.test.tsx`, no fim:

```tsx
describe("PersonStart — outra fonte (módulo 25)", () => {
  it("com source, busca pelas funções dela e não pelas do território", async () => {
    const source = { lookup: vi.fn().mockResolvedValue({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null }),
      duplicates: vi.fn().mockResolvedValue([]), describe: () => "frase da fonte", pickLabel: (name: string) => `Notificar ${name}` };
    renderWithProviders(<PersonStart onChoose={vi.fn()} onCancel={vi.fn()} source={source} />);
    fireEvent.change(screen.getByLabelText("CPF"), { target: { value: "52998224725" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    await waitFor(() => expect(source.lookup).toHaveBeenCalledWith("52998224725"));
    expect(await screen.findByRole("button", { name: "Notificar Joana Lima" })).not.toBeNull();
    expect(mocked(api.lookupTerritoryPerson)).not.toHaveBeenCalled();
  });
});
```

(Use os nomes de importação e de `mocked`/`renderWithProviders` que o arquivo já tem; se o `vi.mock` dele não dubla `lookupTerritoryPerson`, acrescente.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/surveillance/ManualNotificationPanel.test.tsx src/modules/territory/PersonStart.test.tsx`
Expected: FAIL — o painel e a prop `source` não existem.

- [ ] **Step 3: Implemente** a fonte trocável no `PersonStart` (as duas `useMutation` chamam `source.lookup`/`source.duplicates`; as frases, `source.describe`; o padrão mantém o 24a idêntico), exporte `newPatientProblem`, e o `ManualNotificationPanel` como descrito; ligue o botão "Notificação manual" na fila.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/surveillance src/modules/territory src/modules/SurveillanceQueue.test.tsx src/lib/territoryDictionaries.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS (os testes do 24a seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/territory/PersonStart.tsx src/modules/territory/PersonStart.test.tsx src/lib/territoryDictionaries.ts src/modules/surveillance/ManualNotificationPanel.tsx src/modules/surveillance/ManualNotificationPanel.test.tsx src/modules/SurveillanceQueue.tsx
$G commit -m "feat: add manual notifications with person and duplicate search

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Ficha complementar do §14 — BLOQUEADA até o §14

**Files:**
- Create: `src/lib/surveillanceForm.ts`, `src/modules/surveillance/NotificationFormSection.tsx`
- Modify: `src/modules/surveillance/NotificationDetail.tsx` (passa o `formSlot`)
- Test: `src/lib/surveillanceForm.test.ts`, `src/modules/surveillance/NotificationFormSection.test.tsx`

**Interfaces:**
- Consumes: `updateNotificationForm` (Task 1); `isStale`, `STALE_TEXT`, `surveillanceError` (Task 2); `NotificationDetail.formSlot` (Task 6); `errorPath` (`territoryRegistry.ts`, lê `path` do corpo da recusa).
- Produces: `FormFieldKind = "text" | "date" | "code" | "boolean" | "integer"`; `FormFieldSpec { key; label; kind; required?: boolean; max?: number; options?: { code: string; label: string }[]; when?: { key: string; equals: string | boolean } }`; `SINAN_FORM_FIELDS: FormFieldSpec[]` (transcrição do §14); `formValues(form, fields): Record<string, string | boolean>`; `formBody(values, fields): NotificationForm`; `formProblem(values, fields, today): { path: string; message: string } | null`; `FORM_EDITABLE: ReadonlySet<string>` (`possible`, `notified`, `investigating`, `closed`; D9); `NotificationFormSection({ notification, today?, fields?, onSaved })`.

- [ ] **Step 1: Confira o §14 no docs** (sem ele, **pare esta task**)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md | sed -n '/^## 14\./,/^## 15\./p'
```

Expected: a tabela do §14 com, por campo, a chave, o rótulo, o tipo, o domínio (códigos do Sinan), o limite, a obrigatoriedade e as condições. Sem ela: anote "Task 8 bloqueada", siga para a Task 9 e avise o coordenador. Com ela: **transcreva** para `SINAN_FORM_FIELDS`, na ordem da tabela, cada linha com a chave, o rótulo e o domínio exatos (código → rótulo); o tipo do §14 mapeia para `FormFieldKind` (texto, data, código de domínio, sim/não, inteiro). O que o §14 disser sobre o formato do `path` de `invalid_form` vence a D9.

- [ ] **Step 2: Write the failing test**

```ts
// src/lib/surveillanceForm.test.ts
import { describe, expect, it } from "vitest";
import { SINAN_FORM_FIELDS, formBody, formProblem, formValues, type FormFieldSpec } from "./surveillanceForm";

// Tabela sintética: prova a lógica sem depender das chaves do §14.
const FIELDS: FormFieldSpec[] = [
  { key: "campo_texto", label: "Texto", kind: "text", max: 10 },
  { key: "campo_codigo", label: "Código", kind: "code", required: true, options: [ { code: "1", label: "Sim" }, { code: "2", label: "Não" }, { code: "9", label: "Ignorado" } ] },
  { key: "campo_condicional", label: "Condicional", kind: "date", required: true, when: { key: "campo_codigo", equals: "1" } },
  { key: "campo_bool", label: "Marca", kind: "boolean" },
  { key: "campo_inteiro", label: "Inteiro", kind: "integer", max: 99 }
];

describe("ficha complementar (lógica)", () => {
  it("valores da notificação: código e inteiro em texto; ausente vazio", () => {
    expect(formValues({ campo_codigo: "2", campo_inteiro: 3, campo_bool: true }, FIELDS))
      .toEqual({ campo_texto: "", campo_codigo: "2", campo_condicional: "", campo_bool: true, campo_inteiro: "3" });
  });
  it("corpo: só os campos visíveis; vazio vira null; inteiro vira número; texto aparado", () => {
    expect(formBody({ campo_texto: " abc ", campo_codigo: "2", campo_condicional: "2026-10-01", campo_bool: false, campo_inteiro: "12" }, FIELDS))
      .toEqual({ campo_texto: "abc", campo_codigo: "2", campo_bool: false, campo_inteiro: 12 });
    expect(formBody({ campo_texto: "", campo_codigo: "1", campo_condicional: "2026-10-01", campo_bool: false, campo_inteiro: "" }, FIELDS))
      .toEqual({ campo_texto: null, campo_codigo: "1", campo_condicional: "2026-10-01", campo_bool: false, campo_inteiro: null });
  });
  it("conferência: obrigatório, condição, domínio, limite e data, com o path do api", () => {
    const v = { campo_texto: "", campo_codigo: "1", campo_condicional: "", campo_bool: false, campo_inteiro: "" };
    expect(formProblem({ ...v, campo_codigo: "" }, FIELDS, "2026-10-27")).toEqual({ path: "form.campo_codigo", message: "Código: obrigatório" });
    expect(formProblem(v, FIELDS, "2026-10-27")).toEqual({ path: "form.campo_condicional", message: "Condicional: obrigatório" });
    expect(formProblem({ ...v, campo_codigo: "7" }, FIELDS, "2026-10-27")).toEqual({ path: "form.campo_codigo", message: "Código: escolha um valor da lista" });
    expect(formProblem({ ...v, campo_condicional: "2026-10-28" }, FIELDS, "2026-10-27")).toEqual({ path: "form.campo_condicional", message: "Condicional: data depois de hoje" });
    expect(formProblem({ ...v, campo_condicional: "2026-10-01", campo_texto: "x".repeat(11) }, FIELDS, "2026-10-27"))
      .toEqual({ path: "form.campo_texto", message: "Texto: no máximo 10 caracteres" });
    expect(formProblem({ ...v, campo_condicional: "2026-10-01", campo_inteiro: "3,5" }, FIELDS, "2026-10-27"))
      .toEqual({ path: "form.campo_inteiro", message: "Inteiro: número inteiro de 0 a 99" });
    expect(formProblem({ ...v, campo_codigo: "2" }, FIELDS, "2026-10-27")).toBeNull();
  });
});

describe("tabela do §14", () => {
  it("transcrita inteira, na ordem, com chaves únicas e domínio em todo código", () => {
    const keys = SINAN_FORM_FIELDS.map((f) => f.key);
    expect(keys).toEqual([ /* as chaves do §14, na ordem da tabela, copiadas no Step 1 */ ]);
    expect(new Set(keys).size).toBe(keys.length);
    for (const f of SINAN_FORM_FIELDS) if (f.kind === "code") expect((f.options ?? []).length).toBeGreaterThan(0);
    for (const f of SINAN_FORM_FIELDS) if (f.when) expect(keys).toContain(f.when.key);
  });
});
```

(O array do primeiro `expect` da tabela do §14 é a lista de chaves copiada do §14 no Step 1 — o único valor lido na execução, por bloqueio explícito.)

```tsx
// src/modules/surveillance/NotificationFormSection.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), updateNotificationForm: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { NotificationFormSection } from "./NotificationFormSection";
import type { FormFieldSpec } from "../../lib/surveillanceForm";
import { STALE_TEXT } from "../../lib/surveillance";
import { TODAY25, notification } from "../../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const FIELDS: FormFieldSpec[] = [
  { key: "campo_codigo", label: "Código", kind: "code", required: true, options: [ { code: "1", label: "Sim" }, { code: "2", label: "Não" } ] },
  { key: "campo_texto", label: "Texto", kind: "text", max: 10 }
];
const mount = (status = "investigating", onSaved = vi.fn()) => render(<QueryClientProvider client={new QueryClient()}>
  <NotificationFormSection notification={notification({ status, version: 4, form: { campo_codigo: "2" } })} today={TODAY25} fields={FIELDS} onSaved={onSaved} />
</QueryClientProvider>);

describe("NotificationFormSection", () => {
  beforeEach(() => mocked(api.updateNotificationForm).mockReset());

  it("salva o corpo da ficha com a versão", async () => {
    mocked(api.updateNotificationForm).mockResolvedValue(notification({ version: 5 }));
    const onSaved = vi.fn();
    mount("investigating", onSaved);
    fireEvent.change(screen.getByLabelText("Texto"), { target: { value: "abc" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar ficha" }));
    await waitFor(() => expect(mocked(api.updateNotificationForm).mock.calls[0]).toEqual([ "nt1", { campo_codigo: "2", campo_texto: "abc" }, 4 ]));
    expect(onSaved.mock.calls[0][0].version).toBe(5);
  });

  it("422 invalid_form com path: a frase e o foco no campo", async () => {
    mocked(api.updateNotificationForm).mockRejectedValue(new ApiError(422, { error: "invalid_form", path: "form.campo_texto" }, "x"));
    mount();
    fireEvent.click(screen.getByRole("button", { name: "Salvar ficha" }));
    expect((await screen.findByRole("alert")).textContent).toBe("confira o campo indicado na ficha");
    expect(document.activeElement).toBe(screen.getByLabelText("Texto"));
  });

  it("409 stale_version na ficha: frase e os valores ficam", async () => {
    mocked(api.updateNotificationForm).mockRejectedValue(new ApiError(409, { error: "stale_version" }, "x"));
    mount();
    fireEvent.change(screen.getByLabelText("Texto"), { target: { value: "abc" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar ficha" }));
    expect((await screen.findByRole("alert")).textContent).toBe(STALE_TEXT);
    expect((screen.getByLabelText("Texto") as HTMLInputElement).value).toBe("abc");
  });

  it("digitada ou descartada: só leitura", () => {
    mount("entered");
    expect(screen.queryByRole("button", { name: "Salvar ficha" })).toBeNull();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/lib/surveillanceForm.test.ts src/modules/surveillance/NotificationFormSection.test.tsx`
Expected: FAIL — os módulos não existem.

- [ ] **Step 4: Implemente** `surveillanceForm.ts` (a tabela do §14 e as três funções; mensagens como no teste: "<rótulo>: obrigatório", "<rótulo>: escolha um valor da lista", "<rótulo>: data depois de hoje", "<rótulo>: no máximo N caracteres", "<rótulo>: número inteiro de 0 a N"; campo com `when` não satisfeito some da tela e do corpo) e `NotificationFormSection` (um campo por linha da tabela: texto `input`, data `type="date"`, código `<select>` com "— escolha —", sim/não `checkbox`, inteiro `inputMode="numeric"`; `ref` por chave para o foco do `path`; "Salvar ficha" → `formProblem` antes, `updateNotificationForm(id, formBody(...), version)` depois; fora de `FORM_EDITABLE`, os valores em `KeyValue`, sem botão). No `NotificationDetail`, o `formSlot` padrão passa a ser `NotificationFormSection` (com `onSaved` atualizando o detalhe).

- [ ] **Step 5: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/lib/surveillanceForm.test.ts src/modules/surveillance && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd $W
$G add src/lib/surveillanceForm.ts src/lib/surveillanceForm.test.ts src/modules/surveillance/NotificationFormSection.tsx src/modules/surveillance/NotificationFormSection.test.tsx src/modules/surveillance/NotificationDetail.tsx
$G commit -m "feat: add the complementary notification form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Notificação negativa e a marca "unidade notificadora" (F-25.6)

**Files:**
- Modify: `src/modules/SurveillanceNegatives.tsx`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/SurveillanceNegatives.test.tsx`, `src/modules/attendance/UnitForm.surveillance.test.tsx`

**Interfaces:**
- Consumes: `listWeeklyNegatives`, `recordWeeklyNegative`, `updateUnit` (Task 1); `previousEpiWeek`, `shiftEpiWeek`, `closedWeekProblem`, `weekLabel`, `negativeStatusLabel`, `surveillanceError`, `NEGATIVES_KEY`, `NOMINAL_QUERY` (Task 2); `SurveillanceGate` (Task 3).
- Produces: `SurveillanceNegatives({ today? })`; `UnitFormValue.notifying?: boolean`; `Units({ labOn?, surveillanceOn? })`.

Comportamento: semana inicial `previousEpiWeek(today)`; "Semana anterior"/"Semana seguinte" (a seguinte desabilitada na semana anterior à de hoje) e o campo "Semana (AAAA-SS)" conferido por `closedWeekProblem` antes de ler; o rótulo `weekLabel`. Uma linha por unidade notificadora: unidade · CNES, `Tag` da situação (`with_cases` ok, `negative` ok, `pending` warn) e, em `pending`, "Registrar negativa" → `recordWeeklyNegative(unit.id, week)` e relê; recusas na frase (`unit_has_notifications`, `already_recorded` também relê). Lista por `NOMINAL_QUERY` (não grava trilha nominal, mas não há o que reler sozinho). Na unidade: `UnitForm` mostra "Unidade notificadora (notificação negativa semanal)" quando `value.notifying !== undefined`; `Units` com `surveillanceOn` passa `notifying: r.notifying !== false` ao editar, salva pelo 7º argumento de `updateUnit` e mostra a coluna "Notificadora" ("notificadora"/"—"); `Attendance` passa `surveillanceOn={hasFeature(user, "surveillance")}`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SurveillanceNegatives.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listWeeklyNegatives: vi.fn(), recordWeeklyNegative: vi.fn() };
});
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SurveillanceNegatives } from "./SurveillanceNegatives";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY25, negatives, survUser } from "../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("SurveillanceNegatives", () => {
  beforeEach(() => {
    for (const fn of [ api.listWeeklyNegatives, api.recordWeeklyNegative ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser());
    mocked(api.listWeeklyNegatives).mockImplementation(async (week: string) => negatives({ week }));
    mocked(api.recordWeeklyNegative).mockResolvedValue(undefined);
  });

  it("abre na semana anterior; registra a negativa da unidade pendente e relê", async () => {
    renderWithProviders(<SurveillanceNegatives today={TODAY25} />);
    expect(await screen.findByText("semana 42/2026 (18/10 a 24/10)")).not.toBeNull();
    expect(mocked(api.listWeeklyNegatives).mock.calls[0][0]).toBe("2026-42");
    const vila = screen.getByRole("row", { name: /UBS Vila Verde/ });
    expect(within(vila).getByText("pendente")).not.toBeNull();
    expect(within(screen.getByRole("row", { name: /UBS Centro/ })).queryByRole("button", { name: "Registrar negativa" })).toBeNull();
    fireEvent.click(within(vila).getByRole("button", { name: "Registrar negativa" }));
    await waitFor(() => expect(mocked(api.recordWeeklyNegative).mock.calls[0]).toEqual([ "u2", "2026-42" ]));
    await waitFor(() => expect(mocked(api.listWeeklyNegatives)).toHaveBeenCalledTimes(2));
    expect((screen.getByRole("button", { name: "Semana seguinte" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("na semana 2026-01, a anterior é 2025-53 e a negativa vai com ela", async () => {
    renderWithProviders(<SurveillanceNegatives today="2026-01-06" />);
    await screen.findByText(/semana 53\/2025/);
    fireEvent.click(within(screen.getByRole("row", { name: /UBS Vila Verde/ })).getByRole("button", { name: "Registrar negativa" }));
    await waitFor(() => expect(mocked(api.recordWeeklyNegative).mock.calls[0]).toEqual([ "u2", "2025-53" ]));
  });

  it("semana em curso digitada é recusada antes de ler; 409 da unidade com casos vira frase", async () => {
    mocked(api.recordWeeklyNegative).mockRejectedValue(new ApiError(409, { error: "unit_has_notifications" }, "x"));
    renderWithProviders(<SurveillanceNegatives today={TODAY25} />);
    await screen.findByText(/semana 42\/2026/);
    fireEvent.change(screen.getByLabelText("Semana (AAAA-SS)"), { target: { value: "2026-43" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver semana" }));
    expect((await screen.findByRole("alert")).textContent).toBe("a semana ainda não terminou");
    expect(mocked(api.listWeeklyNegatives)).toHaveBeenCalledTimes(1);
    fireEvent.change(screen.getByLabelText("Semana (AAAA-SS)"), { target: { value: "2026-42" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver semana" }));
    fireEvent.click(within(await screen.findByRole("row", { name: /UBS Vila Verde/ })).getByRole("button", { name: "Registrar negativa" }));
    expect((await screen.findByRole("alert")).textContent).toBe("a unidade tem notificação nesta semana; a negativa não cabe");
  });
});
```

```tsx
// src/modules/attendance/UnitForm.surveillance.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { UnitForm, type UnitFormValue } from "./UnitForm";

afterEach(cleanup);
const BASE: UnitFormValue = { name: "UBS Centro", kind: "ubs", street: "", number: "", complement: "", zip: "", neighborhoodId: "", screeningScope: "walk_in" };
const LABEL = "Unidade notificadora (notificação negativa semanal)";

describe("UnitForm — unidade notificadora (módulo 25)", () => {
  it("na edição com o campo presente, o checkbox muda o valor salvo", () => {
    const onSave = vi.fn();
    render(<UnitForm initial={{ ...BASE, notifying: true }} neighborhoods={[]} busy={false} onSave={onSave} onCancel={vi.fn()} />);
    expect((screen.getByLabelText(LABEL) as HTMLInputElement).checked).toBe(true);
    fireEvent.click(screen.getByLabelText(LABEL));
    fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
    expect(onSave.mock.calls[0][0].notifying).toBe(false);
  });
  it("sem o campo (interruptor desligado ou unidade nova), nada aparece", () => {
    render(<UnitForm initial={BASE} neighborhoods={[]} busy={false} onSave={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.queryByLabelText(LABEL)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceNegatives.test.tsx src/modules/attendance/UnitForm.surveillance.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** como descrito (o checkbox no molde do de coleta do 21b).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceNegatives.test.tsx src/modules/attendance src/modules/Attendance.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos de unidades seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/SurveillanceNegatives.tsx src/modules/SurveillanceNegatives.test.tsx src/modules/attendance/UnitForm.tsx src/modules/attendance/UnitForm.surveillance.test.tsx src/modules/attendance/Units.tsx src/modules/Attendance.tsx
$G commit -m "feat: add weekly negative reports and the notifying unit flag

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Relatórios — CSV por semana e encerradas sem número oficial (F-25.7)

**Files:**
- Modify: `src/modules/SurveillanceReports.tsx`
- Test: `src/modules/SurveillanceReports.test.tsx`

**Interfaces:**
- Consumes: `fetchWeekCsv`, `listNotifications` (Task 1); `previousEpiWeek`, `weekProblem`, `weekLabel`, `surveillanceError`, `NOTIFICATIONS_KEY`, `NOMINAL_QUERY` (Task 2); `NotificationTable` (Task 5); `NotificationDetail` (Task 6); `saveFile`.
- Produces: `SurveillanceReports({ today? })`.

Comportamento: bloco "CSV por semana epidemiológica": "Semana (AAAA-SS)" com `previousEpiWeek(today)` (a semana em curso é aceita: `weekProblem`), "Baixar CSV" → `fetchWeekCsv(week)` → `saveFile(file, file.filename ?? \`notificacoes-${week}.csv\`)`; a nota "Sem CPF; CNS mascarado." Bloco "Encerradas sem número oficial": lista `status: "closed"` em `NotificationTable` só depois do clique "Mostrar a lista" (leitura nominal com trilha), com "Abrir" levando ao `NotificationDetail` (onde estão o PDF e o "Registrar número oficial").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SurveillanceReports.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), fetchWeekCsv: vi.fn(), listNotifications: vi.fn(), getNotification: vi.fn() };
});
vi.mock("./lab/saveFile", () => ({ saveFile: vi.fn() }));
import * as api from "../lib/api";
import { saveFile } from "./lab/saveFile";
import { SurveillanceReports } from "./SurveillanceReports";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY25, notification, survUser } from "../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("SurveillanceReports", () => {
  beforeEach(() => {
    for (const fn of [ api.fetchWeekCsv, api.listNotifications, saveFile ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser());
    mocked(api.fetchWeekCsv).mockResolvedValue({ blob: new Blob([ "a;b" ]), filename: null });
    mocked(api.listNotifications).mockResolvedValue({ notifications: [ notification({ status: "closed", classification: "confirmed" }) ], next_before: null });
  });

  it("CSV da semana anterior, com nome sem dado pessoal", async () => {
    renderWithProviders(<SurveillanceReports today={TODAY25} />);
    expect(((await screen.findByLabelText("Semana (AAAA-SS)")) as HTMLInputElement).value).toBe("2026-42");
    fireEvent.click(screen.getByRole("button", { name: "Baixar CSV" }));
    await waitFor(() => expect(mocked(saveFile).mock.calls[0][1]).toBe("notificacoes-2026-42.csv"));
    expect(mocked(api.fetchWeekCsv).mock.calls[0][0]).toBe("2026-42");
  });

  it("semana inválida não baixa", async () => {
    renderWithProviders(<SurveillanceReports today={TODAY25} />);
    fireEvent.change(await screen.findByLabelText("Semana (AAAA-SS)"), { target: { value: "2026-44" } });
    fireEvent.click(screen.getByRole("button", { name: "Baixar CSV" }));
    expect((await screen.findByRole("alert")).textContent).toBe("semana no futuro");
    expect(mocked(api.fetchWeekCsv)).not.toHaveBeenCalled();
  });

  it("encerradas sem número: só lê no clique, com status closed", async () => {
    renderWithProviders(<SurveillanceReports today={TODAY25} />);
    await screen.findByRole("button", { name: "Mostrar a lista" });
    expect(mocked(api.listNotifications)).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Mostrar a lista" }));
    expect(await screen.findByRole("row", { name: /Joana Lima/ })).not.toBeNull();
    expect(mocked(api.listNotifications).mock.calls[0][0]).toEqual({ status: "closed" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceReports.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** como descrito.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceReports.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/SurveillanceReports.tsx src/modules/SurveillanceReports.test.tsx
$G commit -m "feat: add surveillance reports with weekly csv and closed cases without number

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Painel bairro × semana epidemiológica, com supressão e sinal (F-25.8)

**Files:**
- Modify: `src/modules/SurveillancePanel.tsx`
- Test: `src/modules/SurveillancePanel.test.tsx`

**Interfaces:**
- Consumes: `getSurveillancePanel`, `listSurveillanceDiseases` (Task 1); `previousEpiWeek`, `shiftEpiWeek`, `weekRangeProblem`, `SIGNAL_NOTE`, `PANEL_KEY`, `STATIC_QUERY`, `surveillanceError` (Task 2); `SUPPRESSED_LABEL`, `SUPPRESSED_HINT` não (a frase do painel é própria); `fmtNumber`.
- Produces: `SurveillancePanel({ today? })`.

Comportamento: filtros "De (AAAA-SS)" (`shiftEpiWeek(previousEpiWeek(today), -11)`: as 12 semanas do padrão do api), "Até (AAAA-SS)" (`previousEpiWeek(today)`) e "Doença" (opcional, "todas"); "Atualizar" aplica (a consulta usa os filtros aplicados; `weekRangeProblem` antes). Uma `DataTable`: coluna "Bairro" `card: "title"`, uma coluna por semana com rótulo `SS/AAAA`; célula `null` → "< 5" (`SUPPRESSED_LABEL`), número → `fmtNumber`, e `Tag tone="warn"` "sinal" ao lado quando `signal[i]`. Notas: "Contagens de 1 a 4 aparecem como “< 5”, para não identificar ninguém." e `SIGNAL_NOTE`. Sem nome de pessoa (o api não manda). Leitura agregada (sem trilha) com `STATIC_QUERY` e chave dos filtros aplicados.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SurveillancePanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getSurveillancePanel: vi.fn(), listSurveillanceDiseases: vi.fn() };
});
import * as api from "../lib/api";
import { SurveillancePanel } from "./SurveillancePanel";
import { SIGNAL_NOTE } from "../lib/surveillance";
import { renderWithProviders } from "../test/campaignFixtures";
import { DENGUE, TODAY25, panel, survUser } from "../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("SurveillancePanel", () => {
  beforeEach(() => {
    mocked(api.getSurveillancePanel).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser([ "analyst" ]));
    mocked(api.getSurveillancePanel).mockResolvedValue(panel());
    mocked(api.listSurveillanceDiseases).mockResolvedValue([ DENGUE ]);
  });

  it("analyst vê o painel das 12 semanas anteriores; contagem nula vira < 5 e zero continua 0", async () => {
    renderWithProviders(<SurveillancePanel today={TODAY25} />);
    const unknown = await screen.findByRole("row", { name: /não informado/ });
    expect(within(unknown).getByText("< 5")).not.toBeNull();
    expect(within(unknown).getByText("0")).not.toBeNull();
    expect(mocked(api.getSurveillancePanel).mock.calls[0][0]).toEqual({ fromWeek: "2026-31", toWeek: "2026-42", disease: null });
  });

  it("sinal rotulado para avaliar; a palavra surto não aparece", async () => {
    renderWithProviders(<SurveillancePanel today={TODAY25} />);
    const centro = await screen.findByRole("row", { name: /Centro/ });
    expect(within(centro).getByText("14")).not.toBeNull();
    expect(within(centro).getAllByText("sinal")).toHaveLength(1);
    expect(screen.getByText(SIGNAL_NOTE)).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/surto/i);
  });

  it("faixa de mais de 53 semanas é recusada antes de pedir; doença e semanas aplicadas", async () => {
    renderWithProviders(<SurveillancePanel today={TODAY25} />);
    await screen.findByRole("row", { name: /Centro/ });
    fireEvent.change(screen.getByLabelText("De (AAAA-SS)"), { target: { value: "2025-40" } });
    fireEvent.click(screen.getByRole("button", { name: "Atualizar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("no máximo 53 semanas por consulta");
    expect(mocked(api.getSurveillancePanel)).toHaveBeenCalledTimes(1);
    fireEvent.change(screen.getByLabelText("De (AAAA-SS)"), { target: { value: "2026-30" } });
    fireEvent.change(screen.getByLabelText("Doença"), { target: { value: "dengue" } });
    fireEvent.click(screen.getByRole("button", { name: "Atualizar" }));
    await waitFor(() => expect(mocked(api.getSurveillancePanel).mock.calls.at(-1)![0]).toEqual({ fromWeek: "2026-30", toWeek: "2026-42", disease: "dengue" }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillancePanel.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** como descrito.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillancePanel.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/SurveillancePanel.tsx src/modules/SurveillancePanel.test.tsx
$G commit -m "feat: add the neighborhood by epidemiological week panel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Doenças notificáveis — lista nacional e doenças locais com step-up (F-25.1)

**Files:**
- Modify: `src/modules/SurveillanceDiseases.tsx`
- Test: `src/modules/SurveillanceDiseases.test.tsx`

**Interfaces:**
- Consumes: `listSurveillanceDiseases`, `listLocalDiseases`, `createLocalDisease`, `updateLocalDisease` (Task 1); `codesOf`, `localDiseaseProblem`, `deadlineLabel`, `surveillanceError`, `DISEASES_KEY`, `LOCAL_DISEASES_KEY`, `STATIC_QUERY` (Task 2); `SensitiveAction`.
- Produces: `SurveillanceDiseases({ onGoToSecurity? })`.

Comportamento: "Lista nacional (versão N)" — N é o `list_version` das doenças nacionais; as doenças com `local: false`, só leitura (nome, prazo, CID-10, CIAP-2, janela de repetição), com a frase "A lista nacional é mantida pela plataforma, a partir da portaria vigente." Doenças locais: tabela (nome, código, prazo, códigos, situação "ativa"/"inativa", "Editar", "Desativar"/"Reativar") e "Nova doença local" (código, nome, prazo, CID-10 e CIAP-2 em texto separados por vírgula, janela em dias com padrão 30). "Revisar" confere `localDiseaseProblem` e abre o `SensitiveAction` (`requiresStepUp`) com a frase do efeito ("as consultas passam a sugerir esta doença quando um dos códigos for registrado"); desativar diz "some das sugestões; as notificações já feitas ficam". Corpo: `code` só na criação; `cid10`/`ciap2` por `codesOf`; `repeat_after_days` número. Depois de salvar, relê as duas listas. 409 `duplicate_code` e 422 `invalid_disease` ficam no formulário.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SurveillanceDiseases.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listSurveillanceDiseases: vi.fn(), listLocalDiseases: vi.fn(), createLocalDisease: vi.fn(),
    updateLocalDisease: vi.fn() };
});
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SurveillanceDiseases } from "./SurveillanceDiseases";
import { renderWithProviders } from "../test/campaignFixtures";
import { DENGUE, MEASLES, localDisease, survUser } from "../test/surveillanceFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("SurveillanceDiseases", () => {
  beforeEach(() => {
    for (const fn of [ api.createLocalDisease, api.updateLocalDisease ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(survUser());
    mocked(api.listSurveillanceDiseases).mockResolvedValue([ DENGUE, MEASLES, { ...localDisease(), local: true } ]);
    mocked(api.listLocalDiseases).mockResolvedValue([ localDisease() ]);
    mocked(api.createLocalDisease).mockResolvedValue(localDisease({ id: "ld2" }));
    mocked(api.updateLocalDisease).mockResolvedValue(localDisease({ active: false }));
  });

  it("lista nacional só leitura; doenças locais com situação", async () => {
    renderWithProviders(<SurveillanceDiseases />);
    expect(await screen.findByText("Lista nacional (versão 1)")).not.toBeNull();
    const sarampo = screen.getByRole("row", { name: /Sarampo/ });
    expect(within(sarampo).getByText("imediata (24 h)")).not.toBeNull();
    expect(within(sarampo).queryByRole("button")).toBeNull();
    expect(within(screen.getByRole("row", { name: /Febre do Oropouche/ })).getByText("ativa")).not.toBeNull();
  });

  it("nova doença local: formato antes, step-up e o corpo com códigos normalizados", async () => {
    renderWithProviders(<SurveillanceDiseases />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova doença local" }));
    text("Código", "Febre"); text("Nome", "Febre amarela silvestre"); text("Prazo", "immediate"); text("CID-10", "a95.0, A95.9");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("código começa com local_ e segue com letras minúsculas, números ou _ (até 46)");
    text("Código", "local_febre_amarela_silvestre");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.createLocalDisease).mock.calls[0][0]).toEqual({
      code: "local_febre_amarela_silvestre", name: "Febre amarela silvestre", deadline: "immediate", cid10: [ "A950", "A959" ], ciap2: [], repeat_after_days: 30 }));
  });

  it("desativar diz que as notificações ficam e manda active false sem código", async () => {
    renderWithProviders(<SurveillanceDiseases />);
    fireEvent.click(within(await screen.findByRole("row", { name: /Febre do Oropouche/ })).getByRole("button", { name: "Desativar" }));
    expect(screen.getByText(/as notificações já feitas ficam/)).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.updateLocalDisease).mock.calls[0][0]).toBe("ld1"));
    expect(mocked(api.updateLocalDisease).mock.calls[0][1]).toEqual({ name: "Febre do Oropouche", deadline: "immediate", cid10: [ "A938" ], ciap2: [],
      repeat_after_days: 30, active: false });
  });

  it("409 duplicate_code fica no formulário", async () => {
    mocked(api.createLocalDisease).mockRejectedValue(new ApiError(409, { error: "duplicate_code" }, "x"));
    renderWithProviders(<SurveillanceDiseases />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova doença local" }));
    text("Código", "local_febre_oropouche"); text("Nome", "Febre do Oropouche"); text("Prazo", "weekly");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    expect(await screen.findByText("já existe doença com este código")).not.toBeNull();
    expect((screen.getByLabelText("Nome") as HTMLInputElement).value).toBe("Febre do Oropouche");
  });
});
```

(O rótulo do botão de confirmar é o `confirmLabel` passado ao `SensitiveAction`; com a sessão de `mfa_verified_at` de agora, ele não pede código. Se o padrão do componente for outro rótulo, use o mesmo nos três testes.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceDiseases.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** como descrito (o molde das microáreas do 24a: conferência antes, escrita no `SensitiveAction`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod25 src/modules/SurveillanceDiseases.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/SurveillanceDiseases.tsx src/modules/SurveillanceDiseases.test.tsx
$G commit -m "feat: add the notifiable disease list and local diseases

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Suíte, build, revisão e prova no navegador (api real na 3047)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–12 na branch `feat/mod-25-disease-surveillance`; o api do 25 na **3047** com a semente do 25.
- Produces: branch pronta para merge (depois do merge do api do 25); evidências em `evidencias/2026-10-27-modulo-25/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test dashboard -w dashboard-mod25 && cd $W && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base da Task 0 mais **16 arquivos de teste novos** (`api.surveillance`, `surveillance`, `modules.surveillance`, `useSurveillanceCount`, `NotificationSuggestions`, `ConsultationWorkspace.surveillance`, `SurveillanceQueue`, `NotificationDetail`, `ManualNotificationPanel`, `surveillanceForm`, `NotificationFormSection`, `SurveillanceNegatives`, `UnitForm.surveillance`, `SurveillanceReports`, `SurveillancePanel`, `SurveillanceDiseases`; **14** sem a Task 8) e os existentes com casos novos (`features.test.ts`, `modules.test.ts`, `team.test.ts`, `ConsultationEditor.test.tsx`, `PersonStart.test.tsx`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css package.json package-lock.json
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css` ou dependência mudados (o `vite.config.ts` muda só pelo proxy da Task 1).

- [ ] **Step 2: Nada vaza, nada fica no navegador, nada que grava trilha relê sozinho, nada bloqueia a consulta**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage\|indexedDB" src/lib/surveillance.ts src/lib/surveillanceForm.ts src/modules/surveillance src/modules/Surveillance*.tsx src/modules/consultation/NotificationSuggestions.tsx | grep -v "\.test\."
grep -rn "refetchInterval\|refetchOnWindowFocus: true" src/modules/surveillance src/modules/Surveillance*.tsx src/modules/consultation/NotificationSuggestions.tsx | grep -v "\.test\."
sed -n '/─── Vigilância epidemiológica (módulo 25/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [A-Za-z_, :]* })'
grep -rn "maxLength" src/modules/surveillance src/modules/Surveillance*.tsx src/modules/consultation/NotificationSuggestions.tsx
grep -rni "surto" src/modules/surveillance src/modules/Surveillance*.tsx src/lib/surveillance.ts | grep -v "\.test\."
grep -n "notification\|Notification" src/modules/consultation/ConsultationEditor.tsx
```

Expected: o primeiro, o segundo, o quarto e o quinto sem saída; o terceiro só com `status`, `disease`, `overdue`, `unit_id`, `before`, `limit`, `week`, `from_week`, `to_week`; o último só a prop `notifications` e o `<NotificationSuggestions …/>` — nada em `missing`, `finalizeProblems` ou `blockedFinal`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§1–§7), o ADR 0043, o contrato (§1–§10, §12) com o §14 e as Divergências deste plano. Pontos de atenção:
- sem `surveillance`, o menu, a consulta e a tela de unidades são os de hoje (testes antigos verdes; só as contagens de `modules.test.ts` mudaram);
- a notificação nunca trava a finalização; o aviso pendente aparece no quadro de finalizar;
- só médico `225*` e enfermeiro `2235*` veem a peça; o api recusa de novo;
- fila, detalhe, encerradas sem número, sugestões e busca de pessoa com `NOMINAL_QUERY`; só a contagem relê;
- `stale_version` mantém o que foi digitado; nenhuma reescrita automática;
- notificação manual: "Criar pessoa sem CPF" só depois da busca de duplicados; CPF, nome e mãe só no corpo; o `PersonStart` do 24a idêntico com a fonte padrão;
- painel: `null` como "< 5", sinal como "sinal" com `SIGNAL_NOTE`, nada de "surto";
- semana epidemiológica igual à do §14 (D16) em painel, CSV e negativa;
- ficha (Task 8) espelhando o §14 campo a campo, com o mesmo `path` do api;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 25 na **3047** (plano do api: servidor do worktree `apps/api/.claude/mod25`, migrações de cidade e plataforma a partir de `20261027900001` aplicadas no banco de dev **com autorização do usuário**, e a semente do 25: Curitiba com `clinical_record` e a lista nacional carregada, uma usuária com o papel `surveillance`, uma médica e uma enfermeira em UBS, duas UBS notificadoras, um paciente com domicílio do 24a num bairro, um paciente sem CPF e algumas notificações de semanas anteriores para o painel). Ligue em Curitiba, pelo maintenance, `surveillance`. O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod25 --network rota-saude_default -p 5202:5202 \
  -v $R/.worktrees/dashboard-mod25:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3047 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5202 --host 0.0.0.0"
docker logs -f dashboard-mod25   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 25 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5202/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Roteiro, com screenshot de cada passo em `evidencias/2026-10-27-modulo-25/dashboard/`:
1. **Admin** — Equipe: conceder o papel "Vigilância epidemiológica" pede step-up. Atendimento → Unidades: editar uma UBS mostra "Unidade notificadora…" marcada; desmarcar e salvar; a coluna "Notificadora" muda; remarcar.
2. **Médica** — atendimento chamado → Iniciar consulta → incluir o problema CID-10 A90: depois do "salvo às…", o aviso "CID de notificação obrigatória (A90): notificar suspeita de Dengue"; informar os primeiros sintomas e "Notificar suspeita de Dengue" → "Suspeita de Dengue notificada à vigilância."; a sugestão some.
3. **Médica** — em outro atendimento, incluir um CID de sarampo e **não** notificar: "Finalizar consulta" mostra o aviso pendente e "Confirmar finalização" finaliza normalmente.
4. **Enfermeira** — consulta com CIAP-2 A71: aviso de sarampo; com A77 (ambíguo, fora da lista — plano do api, Leitura 3) nenhum aviso, e "Notificar outra suspeita" lista as doenças da cidade para notificar dengue à mão.
5. **Vigilância** — o menu mostra "Fila da vigilância (1 imediata)" quando houver imediata; a fila mostra o alerta, as abas com números, a notificação do passo 2 em "Notificadas" e a do passo 3 em "Possíveis não feitas" (rede de segurança).
6. **Detalhe** — abrir a do passo 2: pessoa, bairro do domicílio (24a), prazo, andamento; "Iniciar investigação" → "Encerrar" (classificação) → "Registrar número oficial" (o formato é conferido antes); o andamento cresce a cada passo. "Ficha em PDF" baixa `ficha-notificacao-<id>.pdf` no formato do Sinan.
7. **Concorrência** — o mesmo detalhe em duas abas; encerrar na primeira e tentar na segunda → "outra pessoa alterou…", o motivo digitado continua; "Recarregar" traz a versão nova.
8. **Possível** — abrir a do passo 3: "Descartar possível" exige motivo; descartar; ela vai para "Descartadas".
9. **Manual** — "Notificação manual" → "Não tem CPF" → nome, nascimento e mãe → "Buscar possíveis duplicados" → criar pessoa → sexo e motivo → doença, unidade, "Mora em outro município" com IBGE → registrar; abre o detalhe da nova, marcada como residente em outro município. Repetir para a mesma pessoa e doença → "já existe notificação…" com "Abrir a notificação existente".
10. **Ficha** (se a Task 8 entrou) — preencher a ficha complementar; um campo fora do domínio é recusado com o campo em foco; salvar.
11. **Negativa** — abre na semana anterior; a UBS sem casos aparece "pendente" → "Registrar negativa"; a UBS com caso não oferece o botão; digitar a semana em curso é recusado.
12. **Relatórios** — "Baixar CSV" da semana anterior (`notificacoes-AAAA-SS.csv`, sem CPF); "Mostrar a lista" das encerradas sem número.
13. **Painel** — como **analyst**: só o painel no grupo; bairros × 12 semanas, "< 5" nas células pequenas, "sinal" onde a semente provocar, a nota do sinal; nenhum nome.
14. **Doenças** — lista nacional só leitura; criar uma doença local com CID (step-up); numa consulta nova, o CID dela gera a sugestão; desativar a doença local e a sugestão some.
15. **Sem polling e sem vazamento** — com a aba de rede aberta na fila por 3 minutos: nenhuma chamada a `/surveillance/notifications?` sem ação; só `/surveillance/notifications/count` a cada minuto. Nenhuma URL com CPF ou nome; `localStorage` e `sessionStorage` sem nada da vigilância.
16. **Telefone** (largura 375): fila, negativa, painel e doenças viram cartões, sem rolagem lateral.
17. No maintenance, desligar `surveillance` e recarregar: o grupo some, a consulta não mostra aviso e a unidade não mostra a marca; religar devolve, com tudo o que foi gravado.

Depois: `docker rm -f dashboard-mod25`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

F-25.2 e F-25.4..F-25.8 vão a **Done** se a parte do api já está Done (o dashboard é o último app deles; F-25.4 e F-25.5 só se a Task 8 entrou ou se o coordenador aceitar a ficha depois); F-25.1 espera o maintenance; F-25.3 é só do api. Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified**. Use o mesmo `gh project item-edit` da Task 0 com o id da opção Done (`98236657`) e o de Verified (da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 25, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-25-disease-surveillance
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod25
/opt/homebrew/bin/git branch -d feat/mod-25-disease-surveillance
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (D1–D16). **Abertas** = o contrato não tem e a tela precisa; quem decide é o plano do api (ou o coordenador), e a forma usada aqui é a proposta.

1. **D1 — Prefixo da sugestão.** Alinhado com o plano do api (V5). O §4 escreve `GET /consultations/:id/notification_suggestions`, mas as rotas da consulta vivem em `/attendance/consultations/:id/*`. Proposta: `GET /attendance/consultations/:id/notification_suggestions` (mesmo controle de acesso da consulta: autora, em contexto).
2. **D2 — Unidades para a vigilância. Aberta (o plano do api não tem).** A notificação manual exige `unit_id`, e o `GET /attendance/units` não aceita o papel `surveillance` (`require_unit_reader`); sem uma rota, a tela não tem de onde tirar a unidade. Proposta: `GET /surveillance/units` → `{ "units": [ { "id", "cnes", "name", "kind", "notifying" } ] }` (ativas; `surveillance`; sem trilha).
3. **D3 — Busca de pessoa da vigilância.** "Reaproveitando a busca de CPF ou de duplicados do 24a", mas as rotas `/territory/*` exigem `territory_registry` (que exige `clinical_record`) e o papel do território; o `surveillance` liga sem o prontuário (decisão 11). Alinhado com o plano do api (V4): `POST /surveillance/person_lookup { "cpf" }` → `{ "patient_id"|null, "citizen_id"|null, "display_name"|null, "has_cpf", "cns_masked" }` (sem envelope, como o 24a; 404 `person_not_found`, 409 `citizen_not_verified`) e `POST /surveillance/patients/duplicates { "full_name", "birth_date", "mother_name"? }` → `{ "candidates": [ { "patient_id", "display_name", "birth_date", "has_cpf" } ] }`, ambas `surveillance`, com trilha e o mesmo limite do 24a (30 por minuto); e o 409 `duplicate_suspected` no `POST /surveillance/notifications` com `new_patient` sem `duplicates_reviewed`.
4. **D4 — Doença local.** O §3 tem `PATCH /surveillance/admin/local_diseases/:id`, mas `<disease>` não tem `id`. Proposta: `<local_disease>` = `<disease>` + `{ "id", "active" }`; `GET` do admin lista ativas e inativas; `PATCH` muda tudo menos `code`; desativar = `active: false` (some das sugestões e de `/surveillance/diseases`; o que foi notificado fica). Código local `local_[a-z0-9_]{1,40}` e colisão de código, CID ou CIAP com a lista vigente → 409 `duplicate_code` (alinhado com o plano do api, V2); falta só o `id`/`active` na forma.
5. **D5 — Mapa de transições.** O §5 não diz quais transições valem. Alinhado com o plano do api (V9): lineares, `possible → notified` (ou `dismiss`), `notified → investigating`, `investigating → closed`, `closed → entered`; `entered` e `dismissed` finais; `reason` opcional ≤ 500 nas transições, obrigatório 3..500 no descarte (`invalid_reason`). Proposta: o contrato §5 registra o mapa.
6. **D6 — Formato do número oficial.** O §5 não fixa. Alinhado com o plano do api (`OFFICIAL_NUMBER = [0-9A-Za-z./-]{1,30}`, V9). Proposta: o contrato registra o formato e a recusa de formato inválido (`official_number_required` ou código próprio — a tela traduz os dois).
7. **D7 — Semana da negativa.** O §6 tem 422 `future_week`, sem dizer se a semana em curso conta. Alinhado com o plano do api (V10: a semana cujo sábado é depois de hoje, a corrente inclusive). Proposta: o contrato §6 registra; a tela oferece até a semana anterior.
8. **D8 — Faixa do painel.** O §8 não limita. Alinhado com o plano do api (V14): `from_week ≤ to_week`, no máximo 53 semanas, padrão 12 até a anterior; 422 `invalid_range`/`invalid_week`. Proposta: o contrato §8 registra.
9. **D9 — Ficha: `path` e quando edita.** Alinhado com o plano do api (V16): `PATCH` com a ficha inteira, `invalid_form` com `path` = `form.<chave>`; editável em `possible`, `notified`, `investigating` e `closed`; em `entered` e `dismissed`, 422 `invalid_transition` (a tela nem oferece). Proposta: o contrato §5 registra.
10. **D10 — Contagem das imediatas.** O §5 tem `immediate_open` sem definição. Alinhado com o plano do api (Task 11): imediatas em `possible`, `notified` ou `investigating`. Proposta: o contrato §5 registra.
11. **D11 — Resposta da negativa.** O §6 não diz. Alinhado com o plano do api (Task 14): 201 `{ week, unit, status: "negative" }`; a tela relê a lista de qualquer forma.
12. **D12 — Nomes de arquivo.** Proposta: `Content-Disposition` sem dado pessoal — `ficha-notificacao-<id>.pdf` e `notificacoes-<AAAA-SS>.csv`; a tela usa esses mesmos nomes quando o cabeçalho falta.
13. **D13 — Sugestão depois de finalizar.** O §4 não diz o que acontece com `POST /surveillance/notifications { consultation_id }` de consulta finalizada que já gerou `possible`. Proposta: a tela só oferece a notificação no rascunho; depois de finalizada, o caso é da vigilância (confirmar a `possible`). Se o api quiser aceitar o POST depois, que transforme a `possible` em `notified` em vez de 409.
14. **D14 — Unidade nos dados de outra rota.** Proposta: o api trata `notifying` como as outras chaves da unidade (`POST /attendance/units/:id` aceita, `GET /attendance/units/all` devolve); unidade nova nasce com o padrão do §6 (APS ativa → verdadeiro).
15. **D15 — `notifying` no corpo da unidade.** Mesma proposta de D14 do lado da tela: o 7º argumento de `updateUnit` só vai quando o interruptor está ligado (`surveillanceOn`); sem ele, o corpo é o de hoje.
16. **D16 — Regra da semana epidemiológica.** Alinhado com o plano do api (V1). O §1 diz "semana 1 = a que contém o primeiro sábado do ano", o que discorda da regra do Sinan quando 1º de janeiro é sábado ou sexta (2022: pelo §1, a semana 1 seria 26/12/2021–01/01/2022; pelo Sinan, 02–08/01/2022). Proposta: a regra do Sinan — a semana 1 termina no primeiro sábado de janeiro com pelo menos quatro dias no ano (= contém 4 de janeiro) —, confirmada pela Task 0 do api no §14; a Task 2 segue o §14.

## Self-review

- **Cobertura (spec §2, §3, §5–§7; contrato §1–§10):** interruptor `surveillance` sem o prontuário — Tasks 1 (rótulo) e 3 (menu e portão); papel `surveillance` privilegiado — Task 3; sugestão por CID-10 e CIAP-2 e "Notificar suspeita" sempre à mão, só médico e enfermeiro, sem bloquear — Task 4 (F-25.2; Review Focus 1); rede de segurança (`possible`) visível e descartável — Tasks 5 e 6 (F-25.3 é do api); fila com prazo, imediatas e vencidas primeiro, alerta no painel — Tasks 3 e 5 (F-25.4, F-25.6); detalhe, andamento, classificação, número oficial — Task 6 (F-25.5); notificação manual com a busca e duplicados do 24a, residência em outro município — Task 7 (F-25.4); ficha complementar do §14, dependência explícita e bloqueante — Tasks 0 (Step 4) e 8; notificação negativa por unidade e semana e a marca da unidade — Task 9 (F-25.6); ficha em PDF, CSV por semana e encerradas sem número — Tasks 6 e 10 (F-25.7); painel bairro × semana com supressão e sinal, `surveillance` e `analyst` — Task 11 (F-25.8); doenças locais com step-up e a lista nacional à vista — Task 12 (F-25.1); bairro "não informado" — Tasks 6 e 11; sem dado pessoal na URL e no navegador — Tasks 1, 7 e 13; sem polling em rota com trilha — Tasks 3, 5 e 13; prova no navegador com roteiro — Task 13. E-mails e jobs (§9) são do api.
- **Placeholders:** os valores lidos na execução são as chaves do §14 (Task 8, com a regra escrita: o §14 vence), os ids dos cards (Task 0) e os ícones, se algum faltar (Task 0, com a regra). Nenhum passo diz "trate os casos".
- **Tipos e nomes:** `SurveillanceNotification`, `NotificationQuery`, `NotificationCounts`, `TransitionInput`, `ManualNotificationInput`, `WeeklyNegatives`, `SurveillancePanel`, `LocalDisease`/`LocalDiseaseInput` (Task 1) são os usados nas Tasks 4–12; `listNotifications(q)` recebe `{ status?, disease?, overdue?, unitId?, before? }` em todas; `transitionNotification(id, input)` com a versão dentro do `input`, e `dismissNotification(id, reason, version)` / `updateNotificationForm(id, form, version)` com a versão como último argumento, iguais nas Tasks 1, 6 e 8; `epiWeekOf`, `previousEpiWeek`, `shiftEpiWeek`, `weekLabel`, `closedWeekProblem`, `weekProblem`, `weekRangeProblem` (Task 2) batem com as Tasks 9–11; `useSurveillanceCount` e `SURVEILLANCE_COUNT_KEY` iguais nas Tasks 3, 5 e 6; `NotificationTable` (Task 5) é o das Tasks 10; `NotificationDetail.formSlot` (Task 6) é o que a Task 8 preenche; `PersonSource`/`TERRITORY_PERSON_SOURCE` (Task 7) mantêm o 24a; `ConsultationEditor.notifications` (Task 4) é o que o Workspace passa.
- **Review Focus:** 1 — Task 4 ("sugestão pendente ou leitura que falha: a finalização segue igual"); 2 — Task 5 ("volta à aba e reconecta: a lista não relê") e Task 3 ("a contagem relê a cada minuto e acende o item com as imediatas"); 3 — Task 6 ("409 stale_version na transição…") e Task 8 ("409 stale_version na ficha…"); 4 — Task 7 ("busca e criação: CPF, nome e mãe só no corpo; nada no storage"); 5 — Task 2 ("semana epidemiológica: domingo a sábado, 4 de janeiro, semana 53") e Task 9 ("na semana 2026-01, a anterior é 2025-53 e a negativa vai com ela").
- **Proporção:** os blocos de código são testes (os nomes, rótulos e valores que o implementador não decide sozinho), os tipos do contrato e o algoritmo da semana epidemiológica; os corpos das telas ficam com a assinatura e o comportamento.
