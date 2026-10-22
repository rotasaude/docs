# Módulo 23 — Linhas de cuidado (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 23** (`feat/mod-23-care-lines`, worktree `apps/api/.claude/mod23`, porta de dev **3045**) com as rotas `/care_lines/*` do contrato (§3–§6) na forma do plano do api (`2026-10-22-module-23-care-lines-api.md`, Divergências L1–L14) e as que ainda faltam neste plano (fim do arquivo: D2, D6, D7, D10), o interruptor `care_lines` na sessão (`requires: ["feature:clinical_record"]`) e a semente do 23 (Curitiba em modo `record`, duas equipes com INE na UBS Vila Verde e na UBS Boqueirão, uma enfermeira e um médico com vínculo ativo na UBS Vila Verde, uma técnica de enfermagem com vínculo e CBO 322245, um diabético com consulta em dia e HbA1c vencida, uma gestante sugerida, um cidadão validado que nunca consultou e com 62 anos). Ele é **mergeado antes** deste (contrato §10: api → dashboard e wpda → maintenance → catálogo). As Tasks 0–11 não dependem dele (os testes usam dublês de `src/lib/api`); a prova da Task 12 depende.

**Goal:** Lado dashboard do módulo 23: o grupo **"Linhas de cuidado"** com três telas — **Linhas da cidade** (o `municipal_admin` adota as linhas do catálogo-base, adota a versão nova e cria linha própria, tudo com step-up), **Acompanhamento da equipe** (o profissional com vínculo na unidade da equipe vê o painel nominal por equipe e linha, com o filtro de busca ativa, a situação por boa prática, o registro de contato, o atalho para marcar horário do módulo 17, as sugestões a confirmar — só médico ou enfermeiro aceita, e a DUM é obrigatória na gestante —, a inscrição manual e o encerramento com motivo) e **Prévia dos indicadores** (C2 a C7 por equipe para `municipal_admin` e `analyst`, sempre com o aviso de estimativa). Os contadores leem só as rotas `/count`; nada que grava trilha relê sozinho.

**Architecture:** O cliente HTTP ganha a seção "Linhas de cuidado" no fim de `src/lib/api.ts` (tipos do contrato §1–§6 e das Divergências; na URL só ids, o filtro de situação, o indicador e o cursor; CPF, motivos e notas no corpo). As regras de tela ficam puras em `src/lib/careLines.ts` (rótulos dos códigos do contrato §1, frase de cada boa prática, papel e interruptor, formulários — contato, encerramento, recusa, DUM, linha própria —, opções das consultas com e sem trilha, frases das recusas). As peças ficam em `src/modules/careLines/` (`CareLinesGate`, `CustomLineForm`, `EnrollmentList`, `EnrollmentDetail`, `ContactForm`, `EndEnrollmentForm`, `CareAppointment`, `SuggestionsPanel`, `ManualEnrollPanel`, `PreviewCard`) e as três telas em `src/modules/` (`CareLinesAdmin`, `CareLinesTeam`, `CareLinesPreview`), registradas no grupo novo (`modules.ts`, `catalog.ts`, `App.tsx`). O atalho para agendar reaproveita o `BookPanel` do módulo 17 sem mudá-lo. O dashboard nunca decide o que o api garante: elegibilidade, situação de cada boa prática, pior situação, quem confirma, a pontuação da prévia e a trilha são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-22-module-23-care-lines-design.md` (§2 decisões e decisões do coordenador, §3 interruptor, §4 dados, §5 acompanhamento, §6 telas, §7 segurança, §8 testes, §10 F-IDs), `docs/adr/0040.md` e o contrato `docs/superpowers/plans/2026-10-22-module-23-care-lines-contracts.md` (§1–§6, §9, §10), com as convenções herdadas (21a §15, 21b §14, 21c §11, 22 §12–§13). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim (D1–D12, cada uma marcada como adotada pela L do plano do api ou ainda aberta); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): linhas `child_c2`, `pregnancy_c3`, `diabetes_c4`, `hypertension_c5`, `elderly_c6`, `women_c7`; fonte `consultation`, `blood_pressure`, `weight_height`, `exam:<sigtap>`, `procedure:<sigtap>`, `acs_visit`, `vaccine:<código>`; situação `ok` | `due_soon` | `overdue` | `untracked`; cronograma `proportional` | `end_of_window`; âncora `enrollment` | `birth` | `lmp` | `rolling`; origem da sugestão `problem` | `age_sex` | `declared` | `resolved_problem`; saída `discharge` | `death` | `moved` | `pregnancy_ended` | `other`; canal `phone` | `home_visit` | `wpda_message` | `in_person`; desfecho `reached` | `no_answer` | `scheduled` | `refused`; condição `diabetes` | `hypertension` | `pregnancy` | `mental_health` | `none`. Código desconhecido aparece cru, nunca some.
- Interruptor (spec §3; contrato §8): `care_lines`, que exige `clinical_record`. Desligado, o dashboard é exatamente o de hoje: o grupo "Linhas de cuidado" some do menu e das telas. 403 `feature_disabled` → `CARE_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo em `features.ts`.
- **Nenhuma inscrição sem um profissional confirmar** (ADR 0040, invariante): a tela só cria inscrição por "Aceitar" numa sugestão ou por "Inscrever" manual, os dois com clique do profissional; nada inscreve sozinho. **Só médico ou enfermeiro aceita e inscreve à mão** (CBO 225* ou 2235*, e 2231* pela L3 do plano do api): o botão "Aceitar" e o "Inscrever pessoa" só aparecem com `can_confirm` da equipe (D2); o api recusa de novo com 403 `cbo_not_allowed`. **Gestante: a DUM é obrigatória** (spec, decisões do coordenador): na linha com `requires_anchor: "lmp"` (L10) o aceite e a inscrição manual pedem a DUM, entre hoje − 336 dias e hoje (L4 `invalid_anchor`); 422 `anchor_required` abre o campo.
- **Boa prática sem fonte no sistema nunca conta como atraso** (ADR 0040): `untracked` aparece como "não acompanhado pelo sistema", em tom neutro, e nunca entra na pior situação calculada pela tela (só quando o api não manda `worst_status`).
- **Nenhum polling em rota que grava trilha** (spec §6 e §7; contrato §5): lista de inscrições, detalhe, sugestões e a busca de pessoa leem só ao abrir, ao filtrar, ao trocar de aba ou depois de uma ação — `refetchInterval: false`, `refetchOnWindowFocus: false`, `refetchOnReconnect: false`, `gcTime: 0` (`NOMINAL_QUERY`). Só os contadores (`/suggestions/count`, `/enrollments/overdue/count`, sem trilha) releem a cada minuto (`COUNT_QUERY`). A prévia é agregada (sem nome) e lê só no clique de "Calcular prévia".
- **A prévia nunca é valor oficial** (ADR 0040): o aviso "Estimativa — o valor oficial é calculado pelo SIAPS; visitas do ACS e vacinas não entram." aparece sempre que a tela da prévia aparece (inclusive vazia ou com erro); pontuação com vírgula decimal (`37,5`), nunca ponto.
- Na URL vão só ids (`team_id`, `care_line_id`, ids de inscrição e sugestão), `status` (`overdue` | `due_soon` | `all`), `indicator`, `before` e `limit` — nunca CPF, nome, nota de contato, motivo de recusa ou de encerramento. O CPF da inscrição manual vai no corpo de um POST (`/care_lines/person_lookup`, D6).
- CPF, nome, condição declarada e nota de contato nunca em `console`, `localStorage`, `sessionStorage`, Analytics ou mensagem de erro.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo). Adotar linha, adotar versão nova, ativar/desativar e salvar linha própria passam pelo `SensitiveAction` com step-up (contrato §3).
- Recusar em vez de cortar: nenhum `maxLength` silencioso; nota de contato até 500; motivo de recusa de sugestão de 1 a 500 (D8); nota do encerramento até 500 (obrigatória em "outro motivo"); nome de linha própria e rótulo de boa prática até 120, texto para o cidadão até 80 — mais longo é recusado com a frase do limite.
- Quem pode (spec, decisões do coordenador; contrato §3–§6): `municipal_admin` adota e cria linhas e vê a prévia; `analyst` vê a prévia; `health_professional` com vínculo ativo na unidade da equipe vê o painel, registra contato, inscreve, encerra e recusa sugestões; aceitar sugestão só médico ou enfermeiro. A tela não oferece o que o papel não pode; o api recusa de novo (`missing_role`, `out_of_scope`, `cbo_not_allowed`).
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem `outline: "none"`, sem `textOverflow` sem `title`, sem cor literal (guardas `src/theme/a11y-guards.test.ts`, `usability-guards.test.ts`, `literals.test.ts`, `css-guards.test.ts`). Use os componentes existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, o `BookPanel` do módulo 17); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- "Hoje" é o dia da cidade (`todayInCity()`); as telas recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY23`, sem relógio falso.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-23.1..F-23.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-23.2, F-23.3, F-23.5, F-23.6: api e dashboard; F-23.1: api e maintenance; F-23.7 e F-23.8: api e wpda; F-23.4: só api) e a **Verified** depois da prova no navegador (Task 12).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod23` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-23-care-lines` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod23`, `G=/opt/homebrew/bin/git`.
- Testes: `cd $W && npx vitest run <arquivos>`. Tipos antes de cada commit: `npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. Sessão de teste com `mfa_verified_at` de agora (`sessionWith`) = janela de step-up aberta (o `SensitiveAction` não pede código); com `mfa_verified_at: null`, pede.
- A prova (Task 12) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5198**, com proxy para o api do 23 na **3045**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Gestante aceita sem DUM, com DUM no futuro ou de mais de 336 dias.** Espera-se: o formulário de aceite da linha que pede âncora não envia sem a DUM e recusa data depois de hoje ou de mais de 336 dias (gestação e puerpério; L4 `invalid_anchor`); se o api responder 422 `anchor_required` (linha que a tela não sabia que pedia), o campo da DUM aparece e nada é inscrito. Teste: Task 9, "gestante: sem DUM não envia; DUM no futuro recusa; 422 anchor_required abre o campo".
2. **Técnica de enfermagem (ou ACS) na equipe abre as sugestões.** Espera-se: vê as sugestões e pode recusar, mas não vê "Aceitar" (`can_confirm: false`); se o api recusar um aceite com 403 `cbo_not_allowed`, a frase "só médico ou enfermeiro confirma a inscrição" e nada muda na lista. Teste: Task 9, "sem can_confirm: só Recusar; 403 cbo_not_allowed vira frase e nada muda".
3. **Painel aberto o turno todo.** Espera-se: a lista nominal (que grava `clinical_record.viewed`) não relê ao voltar à aba, ao reconectar nem com o tempo; só os contadores das abas releem. Teste: Task 6, "volta à aba e passa o tempo: a lista não relê; os contadores usam COUNT_QUERY".
4. **Boa prática que o sistema não acompanha (visita do ACS, vacina).** Espera-se: aparece como "não acompanhado pelo sistema", em tom neutro, nunca como vencida, e não piora a situação da pessoa quando a tela calcula. Testes: Task 2, "untracked nunca piora e tem frase própria"; Task 6, "visita do ACS aparece como não acompanhada, em tom neutro".
5. **Prévia lida como valor oficial.** Espera-se: o aviso de estimativa sempre visível (até com erro ou equipe vazia), pontuação com vírgula, as práticas não rastreadas marcadas e "visita do ACS" e "vacina" listadas como fora da conta. Teste: Task 11, "o aviso de estimativa aparece sempre; pontuação com vírgula; fora da conta listado".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras | 0 |
| `src/lib/api.ts`, `src/lib/api.careLines.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/careLineFixtures.ts` | tipos e cliente HTTP do 23; interruptor; proxy `/care_lines`; dados de teste | 1 |
| `src/lib/careLines.ts`, `src/lib/careLines.test.ts` | regras de tela das linhas de cuidado | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.careLines.test.ts`, `src/shell/catalog.ts`, `src/modules/careLines/CareLinesGate.tsx` | grupo "Linhas de cuidado", catálogo, portão | 3 |
| `src/modules/CareLinesAdmin.tsx`, `src/modules/CareLinesAdmin.test.tsx`, `src/App.tsx` | linhas da cidade: adotar, versão nova, ativar/desativar | 4 |
| `src/modules/careLines/CustomLineForm.tsx`, `src/modules/careLines/CustomLineForm.test.tsx`, `src/modules/CareLinesAdmin.tsx` | linha própria (criar e editar) | 5 |
| `src/modules/careLines/EnrollmentList.tsx`, `src/modules/CareLinesTeam.tsx`, `src/modules/CareLinesTeam.test.tsx`, `src/App.tsx` | painel nominal com busca ativa e contadores | 6 |
| `src/modules/careLines/EnrollmentDetail.tsx`, `src/modules/careLines/ContactForm.tsx`, `src/modules/careLines/EnrollmentDetail.test.tsx`, `src/modules/careLines/EnrollmentList.tsx`, `src/modules/CareLinesTeam.tsx` | detalhe da inscrição e registro de contato | 7 |
| `src/modules/careLines/EndEnrollmentForm.tsx`, `src/modules/careLines/CareAppointment.tsx`, `src/modules/careLines/EnrollmentDetail.tsx`, `src/modules/careLines/EnrollmentActions.test.tsx` | encerrar com motivo; atalho para agendar (módulo 17) | 8 |
| `src/modules/careLines/SuggestionsPanel.tsx`, `src/modules/careLines/SuggestionsPanel.test.tsx`, `src/modules/CareLinesTeam.tsx` | sugestões a confirmar | 9 |
| `src/modules/careLines/ManualEnrollPanel.tsx`, `src/modules/careLines/ManualEnrollPanel.test.tsx`, `src/modules/CareLinesTeam.tsx` | inscrição manual | 10 |
| `src/modules/careLines/PreviewCard.tsx`, `src/modules/CareLinesPreview.tsx`, `src/modules/CareLinesPreview.test.tsx`, `src/App.tsx` | prévia C2–C7 por equipe | 11 |
| — | suíte, build, revisão, prova no navegador | 12 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, motivo, nota) (Task 1); regras puras com tabela de casos: frase de cada boa prática (contagem proporcional, última vez, vencimento, `untracked`), pior situação sem `untracked`, formulários (o que recusa e o que **não** recusa), DUM, linha própria (códigos CIAP/CID, SIGTAP, idade em anos, forma L1/L2), frases das recusas (Task 2); menu por papel e interruptor (Task 3); cada tela e peça pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel e interruptor, cobrindo o caminho feliz, cada recusa que muda a tela (`cbo_not_allowed`, `anchor_required`, `already_enrolled`, `citizen_not_verified`, `duplicate_template`, `invalid_transition`, `feature_disabled`, `missing_role`), o step-up e a ausência de releitura (Tasks 4–11). A prova final é no navegador, contra o api real na 3045 (Task 12).

---

### Task 0: Board, worktree, base e âncoras

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`e45ca71` ou mais novo); o board #1 (`rotasaude`).
- Produces: cards F-23.1..F-23.8 em In Progress; worktree `$W` na branch `feat/mod-23-care-lines`; a base anotada.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-23\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api, do wpda e do maintenance movem os mesmos cards), com o `id` do item e o id da opção **In Progress**:

```bash
gh project item-edit --id PVTI_xxx --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-23.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "procedure_billing";\|  procedure_billing: "Faturamento de procedimentos (BPA)"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const withQuery = \|^export interface RequestRow\|^export interface HealthUnit extends\|^export function saveBillingSettings'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "billing-procedures" | "billing-periods" | "billing-settings";\|    { id: "open-drafts", label: "Rascunhos abertos", icon: "✎" }\|  const isBillingClerk = roles.includes("billing_clerk");\|      if (item.id === "integrations" || item.id === "cnes") return canIntegrations;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 10\|NAV_GROUPS.length - 9'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '  IconUser, IconUserCircle, IconUsers, IconUsersGroup, IconCalendarStats, IconFileInvoice, IconReceipt, IconReceiptTax\|    "Exames":      { icon: IconMicroscope\|    "billing-settings": { icon: IconReceiptTax'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '^import { BillingSettings }\|    case "billing-settings": return <BillingSettings'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/billing(/|\$)": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/BookPanel.tsx | grep -n '^interface Props { row: RequestRow; unit: HealthUnit; onCancel(): void; onDone(): void }\|^export function BookPanel'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod23 -b feat/mod-23-care-lines origin/main
cd $R/.worktrees/dashboard-mod23 && npm ci --silent && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(HeartHandshake|UserSearch|Gauge|Route)\.mjs$'
```

Expected: tudo verde, `tsc-ok` e os quatro ícones. **Anote** a base (arquivos e testes). A Task 12 espera a base mais **11 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 23** (só informativo; as Tasks 1–11 não dependem dele)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-23-care-lines
docker exec -w /rails/.claude/mod23 api-dev bin/rails routes 2>/dev/null | grep -E " /care_lines/" | head -40
```

Expected: a branch e as rotas. Compare com o plano do api (L1–L14) e com as Divergências abertas deste plano (D2, D6, D7, D10); diferença de forma (envelope, nome de campo, forma da `eligibility`) vira ajuste em `src/lib/api.ts` e `src/test/careLineFixtures.ts` antes da prova, e nota para a Task 12.

---

### Task 1: Tipos, cliente HTTP, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/careLineFixtures.ts`
- Test: `src/lib/api.careLines.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `HealthUnit`, `RequestRow`, `SessionUser` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`): os tipos `CareStatus`, `CareSchedule`, `CareAnchor`, `SuggestionReason`, `EndReason`, `ContactChannel`, `ContactOutcome`, `DeclaredCondition`, `CareIndicator`, `CareEligibility`, `CarePractice`, `CareLineTemplate`, `CareLine`, `CareLineInput`, `CareLinePatch`, `CareTeam`, `CareLineRef`, `CareContext`, `CarePerson`, `CarePracticeStatus`, `CareContact`, `CareEnrollment`, `CareEnrollmentPage`, `EnrollmentQuery`, `CareSuggestion`, `CareSuggestionPage`, `SuggestionQuery`, `CareConcept`, `CarePreviewPractice`, `CarePreview`, `CarePersonLookup`, `CareAppointmentTarget`; a constante `CARE_PAGE_SIZE = 50`; as funções `getCareContext(): Promise<CareContext>`, `listCareTemplates(): Promise<CareLineTemplate[]>`, `listCityCareLines(): Promise<CareLine[]>`, `adoptCareLine(templateKey, version): Promise<CareLine>`, `createCustomCareLine(input): Promise<CareLine>`, `updateCareLine(id, patch): Promise<CareLine>`, `listCareSuggestions(q): Promise<CareSuggestionPage>`, `acceptCareSuggestion(id, body): Promise<void>`, `dismissCareSuggestion(id, reason): Promise<void>`, `enrollCare(body): Promise<CareEnrollment>`, `endCareEnrollment(id, reason, note): Promise<void>`, `listCareEnrollments(q): Promise<CareEnrollmentPage>`, `getCareEnrollment(id): Promise<CareEnrollment>`, `recordCareContact(id, body): Promise<void>`, `countCareSuggestions(teamId): Promise<number>`, `countOverdueEnrollments(teamId): Promise<number>`, `getCarePreview(teamId, indicator): Promise<CarePreview>`, `lookupCarePerson(cpf): Promise<CarePersonLookup>`, `openCareAppointment(enrollmentId): Promise<CareAppointmentTarget>`. `FeatureKey` ganha `care_lines`.
- Produces (em `src/test/careLineFixtures.ts`): `TODAY23`, `TEAM_A`, `TEAM_B`, `DIABETES_REF`, `PREGNANCY_REF`, `careUser(roles, over)`, `careContext(over)`, `practice(over)`, `enrollment(over)`, `suggestion(over)`, `DIABETES_TEMPLATE`, `PREGNANCY_TEMPLATE`, `cityLine(over)`, `preview(indicator, over)`, `UBS_VILA_VERDE`, `requestRow(over)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.careLines.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  acceptCareSuggestion, adoptCareLine, countCareSuggestions, countOverdueEnrollments, createCustomCareLine, dismissCareSuggestion,
  endCareEnrollment, enrollCare, getCareContext, getCareEnrollment, getCarePreview, listCareEnrollments, listCareSuggestions,
  listCareTemplates, listCityCareLines, lookupCarePerson, openCareAppointment, recordCareContact, updateCareLine
} from "./api";

function ok(body: unknown, init: ResponseInit = {}): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" }, ...init }));
}

describe("cliente das linhas de cuidado (contrato §3–§6)", () => {
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

  it("contexto: as duas listas de apoio (L10); ausentes viram vazias", async () => {
    fetchMock.mockImplementation((url: string) => ok(String(url).endsWith("/teams")
      ? { teams: [ { id: "tm1", can_confirm: true } ] } : { lines: [ { id: "cl4", requires_anchor: null } ] }));
    expect(await getCareContext()).toEqual({ teams: [ { id: "tm1", can_confirm: true } ], lines: [ { id: "cl4", requires_anchor: null } ] });
    expect(fetchMock.mock.calls.map((c) => new URL(String(c[0]), "http://x").pathname).sort()).toEqual([ "/care_lines/lines", "/care_lines/teams" ]);
    fetchMock.mockImplementation(() => ok({}));
    expect(await getCareContext()).toEqual({ teams: [], lines: [] });
  });

  it("admin: catálogo-base e linhas da cidade em envelope; adotar e criar com JSON", async () => {
    fetchMock
      .mockImplementationOnce(() => ok({ templates: [ { key: "diabetes_c4" } ] }))
      .mockImplementationOnce(() => ok({ lines: [ { id: "cl4" } ] }))
      .mockImplementationOnce(() => ok({ line: { id: "cl3" } }))
      .mockImplementationOnce(() => ok({ line: { id: "cl9" } }))
      .mockImplementationOnce(() => ok({ line: { id: "cl4" } }));
    expect(await listCareTemplates()).toEqual([ { key: "diabetes_c4" } ]);
    expect(urlOf(0).pathname).toBe("/care_lines/admin/templates");
    expect(await listCityCareLines()).toEqual([ { id: "cl4" } ]);
    expect(urlOf(1).pathname).toBe("/care_lines/admin/lines");
    expect(await adoptCareLine("pregnancy_c3", 1)).toEqual({ id: "cl3" });
    expect(urlOf(2).pathname).toBe("/care_lines/admin/lines");
    expect(initOf(2).method).toBe("POST");
    expect(headersOf(2)["Content-Type"]).toBe("application/json");
    expect(bodyOf(2)).toEqual({ template_key: "pregnancy_c3", template_version: 1 });
    const input = {
      name: "Saúde mental", declared_condition: "mental_health" as const,
      eligibility: { problems: { ciap2: [ "P76" ], cid10: [ "F32" ] }, age_min_years: 18, age_max_years: null, sex: null,
        declared: [ "mental_health" ], requires_anchor: null },
      practices: [ { key: "consulta", label: "Consulta", source: "consultation", window_days: 90, min_count: 1,
        schedule: "end_of_window" as const, anchor: "rolling" as const, cbo_filter: [ "2231", "2251", "2252", "2253", "2235" ] } ]
    };
    expect(await createCustomCareLine(input)).toEqual({ id: "cl9" });
    expect(bodyOf(3)).toEqual(input);
    expect(await updateCareLine("cl4", { template_version: 3 })).toEqual({ id: "cl4" });
    expect(urlOf(4).pathname).toBe("/care_lines/admin/lines/cl4");
    expect(initOf(4).method).toBe("PATCH");
    expect(bodyOf(4)).toEqual({ template_version: 3 });
  });

  it("listas: só ids, situação e cursor na URL; envelope com next_before", async () => {
    fetchMock.mockImplementationOnce(() => ok({ enrollments: [ { id: "en1" } ], next_before: "c2" }));
    expect(await listCareEnrollments({ teamId: "tm1", careLineId: "cl4", status: "overdue", before: "c1" }))
      .toEqual({ enrollments: [ { id: "en1" } ], next_before: "c2" });
    expect(urlOf(0).pathname).toBe("/care_lines/enrollments");
    expect(Object.fromEntries(urlOf(0).searchParams)).toEqual({ team_id: "tm1", care_line_id: "cl4", status: "overdue", before: "c1", limit: "50" });
    expect(await listCareEnrollments({ teamId: "tm1", status: "all" })).toEqual({ enrollments: [], next_before: null });
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ team_id: "tm1", status: "all", limit: "50" });
    fetchMock.mockImplementationOnce(() => ok({ suggestions: [ { id: "sg1" } ], next_before: null }));
    expect(await listCareSuggestions({ teamId: "tm1" })).toEqual({ suggestions: [ { id: "sg1" } ], next_before: null });
    expect(urlOf(2).pathname).toBe("/care_lines/suggestions");
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ team_id: "tm1", limit: "50" });
  });

  it("aceitar, recusar, inscrever, encerrar e contato: tudo no corpo, nada na URL", async () => {
    await acceptCareSuggestion("sg1", { team_id: "tm1", anchor_date: "2026-08-01" });
    expect(urlOf(0).pathname).toBe("/care_lines/suggestions/sg1/accept");
    expect(urlOf(0).search).toBe("");
    expect(bodyOf(0)).toEqual({ team_id: "tm1", anchor_date: "2026-08-01" });
    await dismissCareSuggestion("sg1", "já acompanhada no CAPS da região");
    expect(urlOf(1).pathname).toBe("/care_lines/suggestions/sg1/dismiss");
    expect(bodyOf(1)).toEqual({ reason: "já acompanhada no CAPS da região" });
    fetchMock.mockImplementationOnce(() => ok({ enrollment: { id: "en9" } }));
    await enrollCare({ citizen_id: "ci3", care_line_id: "cl6", team_id: "tm1" });
    expect(urlOf(2).pathname).toBe("/care_lines/enrollments");
    expect(bodyOf(2)).toEqual({ citizen_id: "ci3", care_line_id: "cl6", team_id: "tm1" });
    await endCareEnrollment("en1", "moved", "");
    expect(urlOf(3).pathname).toBe("/care_lines/enrollments/en1/end");
    expect(bodyOf(3)).toEqual({ reason: "moved" });
    await endCareEnrollment("en1", "other", "acompanhamento no plano de saúde");
    expect(bodyOf(4)).toEqual({ reason: "other", note: "acompanhamento no plano de saúde" });
    await recordCareContact("en1", { channel: "phone", outcome: "no_answer", note: "" });
    expect(urlOf(5).pathname).toBe("/care_lines/enrollments/en1/contacts");
    expect(urlOf(5).search).toBe("");
    expect(bodyOf(5)).toEqual({ channel: "phone", outcome: "no_answer" });
    for (let i = 0; i <= 5; i += 1) {
      expect(initOf(i).method).toBe("POST");
      expect(headersOf(i)["Content-Type"]).toBe("application/json");
    }
  });

  it("detalhe; contadores sem trilha com { count }; ausente vira 0", async () => {
    fetchMock
      .mockImplementationOnce(() => ok({ enrollment: { id: "en1", contacts: [] } }))
      .mockImplementationOnce(() => ok({ count: 3 }))
      .mockImplementationOnce(() => ok({ count: 12 }))
      .mockImplementationOnce(() => ok({}));
    expect(await getCareEnrollment("en1")).toEqual({ id: "en1", contacts: [] });
    expect(urlOf(0).pathname).toBe("/care_lines/enrollments/en1");
    expect(await countCareSuggestions("tm1")).toBe(3);
    expect(urlOf(1).pathname).toBe("/care_lines/suggestions/count");
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ team_id: "tm1" });
    expect(await countOverdueEnrollments("tm1")).toBe(12);
    expect(urlOf(2).pathname).toBe("/care_lines/enrollments/overdue/count");
    expect(await countOverdueEnrollments("tm1")).toBe(0);
  });

  it("prévia: equipe e indicador na URL; ausentes normalizados e estimate fica true", async () => {
    fetchMock.mockImplementationOnce(() => ok({ indicator: "C4", team: { id: "tm1" }, template_version: 1, denominator: 41, score: "37.5",
      max_score: 100, concept: "sufficient" }));
    expect(await getCarePreview("tm1", "C4")).toEqual({
      indicator: "C4", team: { id: "tm1" }, template_version: 1, denominator: 41, score: "37.5", max_score: 100, concept: "sufficient",
      practices: [], excluded: [], estimate: true
    });
    expect(urlOf().pathname).toBe("/care_lines/preview");
    expect(Object.fromEntries(urlOf().searchParams)).toEqual({ team_id: "tm1", indicator: "C4" });
  });

  it("pessoa pelo CPF no corpo de um POST; atalho do agendamento devolve pedido e unidade", async () => {
    fetchMock
      .mockImplementationOnce(() => ok({ person: { patient_id: null, citizen_id: "ci3", display_name: null, cpf_masked: "***.982.247-**", verified: true } }))
      .mockImplementationOnce(() => ok({ request: { id: "rq1" }, unit: { id: "u1" } }));
    expect((await lookupCarePerson("52998224725")).citizen_id).toBe("ci3");
    expect(urlOf(0).pathname).toBe("/care_lines/person_lookup");
    expect(urlOf(0).search).toBe("");
    expect(initOf(0).method).toBe("POST");
    expect(bodyOf(0)).toEqual({ cpf: "52998224725" });
    expect(await openCareAppointment("en1")).toEqual({ request: { id: "rq1" }, unit: { id: "u1" } });
    expect(urlOf(1).pathname).toBe("/care_lines/enrollments/en1/appointment_request");
    expect(initOf(1).method).toBe("POST");
    expect(bodyOf(1)).toEqual({});
  });
});
```

No fim de `src/lib/features.test.ts`:

```ts
describe("módulo 23", () => {
  it("care_lines tem nome e vem da sessão", () => {
    expect(featureLabel("care_lines")).toBe("Linhas de cuidado");
    expect(hasFeature({ features: [ "clinical_record", "care_lines" ] }, "care_lines")).toBe(true);
    expect(hasFeature({ features: [ "clinical_record" ] }, "care_lines")).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/api.careLines.test.ts src/lib/features.test.ts`
Expected: FAIL — `getCareContext` (e as outras) não existem; `featureLabel("care_lines")` devolve a chave crua.

- [ ] **Step 3: Write minimal implementation**

No fim de `src/lib/api.ts`:

```ts
// ─── Linhas de cuidado (módulo 23, ADR 0040; contrato 2026-10-22) ───
// Na URL só ids (equipe, linha, inscrição, sugestão), o filtro de situação,
// o indicador e o cursor — nunca CPF, nome, nota de contato ou motivo. CPF da
// busca e textos vão no corpo. Elegibilidade, situação de cada boa prática,
// pior situação, quem confirma e a prévia são do api.

const CARE_BASE = import.meta.env.VITE_CARE_LINES_BASE || "/care_lines";
const carePath = (...parts: string[]) => [ CARE_BASE, ...parts.map(encodeURIComponent) ].join("/");
const carePost = (body: unknown): RequestInit => ({ method: "POST", body: JSON.stringify(body) });

export type CareStatus = "ok" | "due_soon" | "overdue" | "untracked";
export type CareSchedule = "proportional" | "end_of_window";
export type CareAnchor = "enrollment" | "birth" | "lmp" | "rolling";
export type SuggestionReason = "problem" | "age_sex" | "declared" | "resolved_problem";
export type EndReason = "discharge" | "death" | "moved" | "pregnancy_ended" | "other";
export type ContactChannel = "phone" | "home_visit" | "wpda_message" | "in_person";
export type ContactOutcome = "reached" | "no_answer" | "scheduled" | "refused";
export type DeclaredCondition = "diabetes" | "hypertension" | "pregnancy" | "mental_health" | "none";
export type CareIndicator = "C2" | "C3" | "C4" | "C5" | "C6" | "C7";

// Elegibilidade (L2 do plano do api): códigos casados por prefixo, CID sem
// ponto; idade em dias ou anos; tudo opcional menos `problems`.
export interface CareEligibility {
  problems: { ciap2: string[]; cid10: string[] };
  exclude_problems?: { ciap2: string[]; cid10: string[] };
  age_min_days?: number | null; age_max_days?: number | null; age_min_years?: number | null; age_max_years?: number | null;
  sex?: "female" | "male" | null; gender_identities_included?: string[]; gender_identities_excluded?: string[];
  declared?: string[]; requires_anchor?: "lmp" | null;
}
// Boa prática (contrato §2 com L1): `points` no catálogo-base; os campos
// avançados (all_of, code_filter, applies_to) só vêm das linhas do catálogo.
export interface CarePractice {
  key: string; label: string; citizen_label?: string | null; source: string; window_start_days?: number | null; window_days: number;
  min_count: number; schedule: CareSchedule | string; anchor: CareAnchor | string; cbo_filter: string[] | null; points?: number | null;
  code_filter?: { ciap2?: string[]; cid10?: string[] } | null; applies_to?: { age_min_years?: number; age_max_years?: number; sex?: string } | null;
  all_of?: { source: string; window_days?: number }[][] | null;
}
export interface CareLineTemplate {
  key: string; name: string; indicator: CareIndicator | string; version: number; eligibility: CareEligibility; practices: CarePractice[];
}
// Divergência D10: `latest_template_version` para oferecer "Adotar versão N".
export interface CareLine {
  id: string; name: string; source: "template" | "custom" | string; template_key: string | null; template_version: number | null;
  indicator: string | null; eligibility: CareEligibility; practices: CarePractice[];
  declared_condition: DeclaredCondition | string | null; active: boolean; latest_template_version?: number | null;
}
export interface CareLineInput {
  name: string; eligibility: CareEligibility; practices: CarePractice[]; declared_condition?: DeclaredCondition;
}
export type CareLinePatch = Partial<CareLineInput> & { active?: boolean; template_version?: number; declared_condition?: DeclaredCondition | null };
// `GET /care_lines/teams` (L10). Divergência D2: `can_confirm` por equipe
// (médico ou enfermeiro com vínculo na unidade dela); ausente = a tela oferece
// e o api recusa com cbo_not_allowed.
export interface CareTeam { id: string; ine: string; name: string; unit: { id: string; name: string }; can_confirm?: boolean }
// `GET /care_lines/lines` (L10): `requires_anchor` "lmp" = a linha pede a DUM.
export interface CareLineRef { id: string; name: string; indicator?: string | null; requires_anchor?: "lmp" | string | null }
// O contexto do painel: as duas listas de apoio, lidas juntas.
export interface CareContext { teams: CareTeam[]; lines: CareLineRef[] }
export interface CarePerson { patient_id: string | null; citizen_id: string | null; display_name: string | null; age: number | null }
export interface CarePracticeStatus {
  key: string; label: string; status: CareStatus | string; done_at: string | null; due_on: string | null;
  count: number | null; expected: number | null;
}
// L12: o histórico de contatos só no detalhe; `last_contact` nunca traz a nota.
export interface CareContact {
  id?: string; channel: ContactChannel | string; outcome: ContactOutcome | string; note: string | null;
  recorded_by: { id: string; name: string } | null; recorded_at: string;
}
export interface CareEnrollment {
  id: string; person: CarePerson; care_line: CareLineRef; team: { id: string; ine: string; name: string };
  anchor_date: string | null; status: "active" | "ended" | string; enrolled_at: string; worst_status: CareStatus | string | null;
  ended_reason?: EndReason | string | null; ended_at?: string | null;
  practices: CarePracticeStatus[]; last_contact: { channel: string; outcome: string; at: string } | null; contacts?: CareContact[];
}
export interface CareEnrollmentPage { enrollments: CareEnrollment[]; next_before: string | null }
export interface EnrollmentQuery { teamId: string; careLineId?: string; status: "overdue" | "due_soon" | "all"; before?: string | null }
// L5: a sugestão nasce numa equipe; `enrollment_id` só na de encerramento.
export interface CareSuggestion {
  id: string; person: CarePerson; care_line: CareLineRef; reason: SuggestionReason | string; created_at: string;
  team_id?: string | null; enrollment_id?: string | null;
}
export interface CareSuggestionPage { suggestions: CareSuggestion[]; next_before: string | null }
export interface SuggestionQuery { teamId: string; careLineId?: string; before?: string | null }
// L8: `score` e `weight` em texto com uma casa e ponto; `score`/`concept` nulos sem população.
export type CareConcept = "optimal" | "good" | "sufficient" | "regular";
export interface CarePreviewPractice {
  key: string; label?: string; tracked: boolean; points?: number; weight: string | number; achieved: number; denominator?: number;
}
export interface CarePreview {
  indicator: string; team: { id: string; ine?: string; name?: string }; template_version: number | null; denominator: number;
  score: string | number | null; max_score: number; concept: CareConcept | string | null;
  practices: CarePreviewPractice[]; excluded: string[]; estimate: true;
}
// Divergência D6.
export interface CarePersonLookup {
  patient_id: string | null; citizen_id: string | null; display_name: string | null; cpf_masked: string; verified: boolean;
}
// Divergência D7: o pedido de agendamento (módulo 17) que o atalho abre.
export interface CareAppointmentTarget { request: RequestRow; unit: HealthUnit }

export const CARE_PAGE_SIZE = 50;

// As duas listas de apoio (L10), sem trilha: equipes que a pessoa vê e linhas ativas.
export async function getCareContext(): Promise<CareContext> {
  const [ teams, lines ] = await Promise.all([
    jsonFetch<{ teams?: CareTeam[] }>(carePath("teams")), jsonFetch<{ lines?: CareLineRef[] }>(carePath("lines"))
  ]);
  return { teams: teams.teams ?? [], lines: lines.lines ?? [] };
}

export async function listCareTemplates(): Promise<CareLineTemplate[]> {
  return (await jsonFetch<{ templates?: CareLineTemplate[] }>(carePath("admin", "templates"))).templates ?? [];
}

export async function listCityCareLines(): Promise<CareLine[]> {
  return (await jsonFetch<{ lines?: CareLine[] }>(carePath("admin", "lines"))).lines ?? [];
}

// Step-up (contrato §3): quem trata 401 mfa_required é o SensitiveAction.
export async function adoptCareLine(templateKey: string, version: number): Promise<CareLine> {
  return (await jsonFetch<{ line: CareLine }>(carePath("admin", "lines"), carePost({ template_key: templateKey, template_version: version }))).line;
}

export async function createCustomCareLine(input: CareLineInput): Promise<CareLine> {
  return (await jsonFetch<{ line: CareLine }>(carePath("admin", "lines"), carePost(input))).line;
}

export async function updateCareLine(id: string, patch: CareLinePatch): Promise<CareLine> {
  return (await jsonFetch<{ line: CareLine }>(carePath("admin", "lines", id), { method: "PATCH", body: JSON.stringify(patch) })).line;
}

function teamParams(teamId: string, careLineId?: string): URLSearchParams {
  const p = new URLSearchParams({ team_id: teamId });
  if (careLineId) p.set("care_line_id", careLineId);
  return p;
}

// Nominal: o api grava trilha a cada leitura; nunca é relida sozinha.
export async function listCareSuggestions(q: SuggestionQuery): Promise<CareSuggestionPage> {
  const p = teamParams(q.teamId, q.careLineId);
  if (q.before) p.set("before", q.before);
  p.set("limit", String(CARE_PAGE_SIZE));
  const b = await jsonFetch<Partial<CareSuggestionPage>>(withQuery(carePath("suggestions"), p));
  return { suggestions: b.suggestions ?? [], next_before: b.next_before ?? null };
}

// Sugestão de encerramento (L5): `{}` encerra com discharge.
export async function acceptCareSuggestion(id: string, body: { team_id?: string; anchor_date?: string }): Promise<void> {
  await jsonFetch<unknown>(carePath("suggestions", id, "accept"), carePost(body));
}

export async function dismissCareSuggestion(id: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(carePath("suggestions", id, "dismiss"), carePost({ reason }));
}

export async function enrollCare(body: {
  patient_id?: string; citizen_id?: string; care_line_id: string; team_id: string; anchor_date?: string;
}): Promise<CareEnrollment> {
  return (await jsonFetch<{ enrollment: CareEnrollment }>(carePath("enrollments"), carePost(body))).enrollment;
}

// Nota vazia não vai.
export async function endCareEnrollment(id: string, reason: EndReason, note: string): Promise<void> {
  const text = note.trim();
  await jsonFetch<unknown>(carePath("enrollments", id, "end"), carePost(text ? { reason, note: text } : { reason }));
}

export async function listCareEnrollments(q: EnrollmentQuery): Promise<CareEnrollmentPage> {
  const p = teamParams(q.teamId, q.careLineId);
  p.set("status", q.status);
  if (q.before) p.set("before", q.before);
  p.set("limit", String(CARE_PAGE_SIZE));
  const b = await jsonFetch<Partial<CareEnrollmentPage>>(withQuery(carePath("enrollments"), p));
  return { enrollments: b.enrollments ?? [], next_before: b.next_before ?? null };
}

export async function getCareEnrollment(id: string): Promise<CareEnrollment> {
  return (await jsonFetch<{ enrollment: CareEnrollment }>(carePath("enrollments", id))).enrollment;
}

export async function recordCareContact(id: string, body: { channel: ContactChannel; outcome: ContactOutcome; note: string }): Promise<void> {
  const note = body.note.trim();
  await jsonFetch<unknown>(carePath("enrollments", id, "contacts"),
    carePost(note ? { channel: body.channel, outcome: body.outcome, note } : { channel: body.channel, outcome: body.outcome }));
}

// Contadores (contrato §5): sem trilha; podem reler a cada minuto.
export async function countCareSuggestions(teamId: string): Promise<number> {
  return (await jsonFetch<{ count?: number }>(withQuery(carePath("suggestions", "count"), teamParams(teamId)))).count ?? 0;
}

export async function countOverdueEnrollments(teamId: string): Promise<number> {
  return (await jsonFetch<{ count?: number }>(withQuery(carePath("enrollments", "overdue", "count"), teamParams(teamId)))).count ?? 0;
}

// Agregado, sem nome (contrato §6). `estimate` é sempre true para a tela.
export async function getCarePreview(teamId: string, indicator: CareIndicator): Promise<CarePreview> {
  const b = await jsonFetch<Partial<CarePreview>>(withQuery(carePath("preview"), new URLSearchParams({ team_id: teamId, indicator })));
  return {
    indicator: b.indicator ?? indicator, team: b.team ?? { id: teamId }, template_version: b.template_version ?? null,
    denominator: b.denominator ?? 0, score: b.score ?? null, max_score: b.max_score ?? 100, concept: b.concept ?? null,
    practices: b.practices ?? [], excluded: b.excluded ?? [], estimate: true
  };
}

export async function lookupCarePerson(cpf: string): Promise<CarePersonLookup> {
  return (await jsonFetch<{ person: CarePersonLookup }>(carePath("person_lookup"), carePost({ cpf }))).person;
}

export function openCareAppointment(enrollmentId: string): Promise<CareAppointmentTarget> {
  return jsonFetch(carePath("enrollments", enrollmentId, "appointment_request"), carePost({}));
}
```

Em `src/lib/features.ts`, troque

```ts
  | "procedure_billing";
```

por

```ts
  | "procedure_billing" | "care_lines";
```

e, em `FEATURE_LABEL`, troque

```ts
  procedure_billing: "Faturamento de procedimentos (BPA)"
};
```

por

```ts
  procedure_billing: "Faturamento de procedimentos (BPA)",
  // Módulo 23 (ADR 0040): exige clinical_record (modo record).
  care_lines: "Linhas de cuidado"
};
```

Em `vite.config.ts`, no comentário do topo, depois da linha do `/clinical_record`, acrescente

```ts
//   /care_lines → linhas de cuidado (módulo 23: linhas da cidade, painel da equipe, sugestões e prévia).
```

e, em `proxy`, troque

```ts
      "^/billing(/|$)": proxy(TARGET)
```

por

```ts
      "^/billing(/|$)": proxy(TARGET),
      // Módulo 23: linhas de cuidado. Regex como os vizinhos.
      "^/care_lines(/|$)": proxy(TARGET)
```

Crie `src/test/careLineFixtures.ts`:

```ts
// src/test/careLineFixtures.ts
// Dados comuns aos testes do módulo 23. Cada arquivo de teste faz o próprio
// vi.mock("…/lib/api"); aqui só tipos e a sessão. Códigos com cara de real
// (INE, SIGTAP, CIAP/CID — semente realista), nomes fictícios.
import type {
  CareContext, CareEnrollment, CareLine, CareLineTemplate, CarePracticeStatus, CarePreview, CareSuggestion, HealthUnit, RequestRow,
  SessionUser
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY23 = "2026-10-22";
export const TEAM_A = { id: "tm1", ine: "0001234567", name: "ESF Vila Verde", unit: { id: "u1", name: "UBS Vila Verde" }, can_confirm: true };
export const TEAM_B = { id: "tm2", ine: "0001234575", name: "ESF Boqueirão", unit: { id: "u2", name: "UBS Boqueirão" }, can_confirm: true };
export const DIABETES_REF = { id: "cl4", name: "Pessoa com diabetes", indicator: "C4", requires_anchor: null };
export const PREGNANCY_REF = { id: "cl3", name: "Gestante e puérpera", indicator: "C3", requires_anchor: "lmp" };
export const UBS_VILA_VERDE: HealthUnit = { id: "u1", name: "UBS Vila Verde", kind: "ubs" };

// Sessão com as linhas ligadas; `mfa_verified_at` = agora (janela de step-up aberta).
export function careUser(roles: string[] = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { id: "us23", email_address: "enfermeira@curitiba.demo", features: [ "clinical_record", "care_lines" ], ...over });
}

// `canConfirm: false` = a técnica de enfermagem (D2: can_confirm por equipe).
export function careContext(over: Partial<CareContext> & { canConfirm?: boolean } = {}): CareContext {
  const { canConfirm = true, ...rest } = over;
  return { teams: [ TEAM_A, TEAM_B ].map((t) => ({ ...t, can_confirm: canConfirm })), lines: [ DIABETES_REF, PREGNANCY_REF ], ...rest };
}

export function practice(over: Partial<CarePracticeStatus> = {}): CarePracticeStatus {
  return { key: "consultation", label: "Consulta com médico ou enfermeiro", status: "ok", done_at: "2026-07-10", due_on: "2027-01-06",
    count: null, expected: null, ...over };
}

export function enrollment(over: Partial<CareEnrollment> = {}): CareEnrollment {
  return {
    id: "en1", person: { patient_id: "pa1", citizen_id: "ci1", display_name: "Maria das Dores", age: 58 }, care_line: DIABETES_REF,
    team: { id: TEAM_A.id, ine: TEAM_A.ine, name: TEAM_A.name }, anchor_date: null, status: "active",
    enrolled_at: "2026-03-02T10:00:00-03:00", worst_status: "overdue",
    practices: [
      practice(),
      practice({ key: "blood_pressure", label: "Aferição da pressão arterial", done_at: "2026-07-10", due_on: "2027-01-06" }),
      practice({ key: "weight_height", label: "Peso e altura", status: "due_soon", done_at: "2025-11-12", due_on: "2026-11-12" }),
      practice({ key: "acs_visit", label: "Visita do ACS", status: "untracked", done_at: null, due_on: null }),
      practice({ key: "hba1c", label: "Hemoglobina glicada", status: "overdue", done_at: "2025-08-20", due_on: "2026-08-20" })
    ],
    last_contact: null, ...over
  };
}

export function suggestion(over: Partial<CareSuggestion> = {}): CareSuggestion {
  return {
    id: "sg1", person: { patient_id: "pa2", citizen_id: "ci2", display_name: "João Batista", age: 63 }, care_line: DIABETES_REF,
    reason: "problem", created_at: "2026-10-21T06:00:00-03:00", enrollment_id: null, ...over
  };
}

const MD_NURSE = [ "2231", "2251", "2252", "2253", "2235" ];
export const DIABETES_TEMPLATE: CareLineTemplate = {
  key: "diabetes_c4", name: "Pessoa com diabetes", indicator: "C4", version: 2,
  eligibility: { problems: { ciap2: [ "T89", "T90" ], cid10: [ "E10", "E11", "E14" ] }, declared: [ "diabetes" ], requires_anchor: null },
  practices: [
    { key: "c4_a", label: "Consulta com médico ou enfermeiro", citizen_label: "consulta com a equipe", source: "consultation",
      window_days: 183, min_count: 1, schedule: "end_of_window", anchor: "rolling", cbo_filter: MD_NURSE, points: 20 },
    { key: "c4_e", label: "Hemoglobina glicada", citizen_label: "exame de hemoglobina glicada", source: "exam:0202010503",
      window_days: 365, min_count: 1, schedule: "end_of_window", anchor: "rolling", cbo_filter: null, points: 15 }
  ]
};
export const PREGNANCY_TEMPLATE: CareLineTemplate = {
  key: "pregnancy_c3", name: "Gestante e puérpera", indicator: "C3", version: 1,
  eligibility: { problems: { ciap2: [ "W78" ], cid10: [ "Z34", "Z35" ] }, declared: [ "pregnancy" ], requires_anchor: "lmp" },
  practices: [
    { key: "c3_b", label: "Consultas de pré-natal", citizen_label: "consulta de pré-natal", source: "consultation", window_start_days: 0,
      window_days: 294, min_count: 7, schedule: "proportional", anchor: "lmp", cbo_filter: MD_NURSE, points: 9 }
  ]
};

export function cityLine(over: Partial<CareLine> = {}): CareLine {
  return {
    id: "cl4", name: "Pessoa com diabetes", source: "template", template_key: "diabetes_c4", template_version: 1, indicator: "C4",
    eligibility: DIABETES_TEMPLATE.eligibility, practices: DIABETES_TEMPLATE.practices, declared_condition: "diabetes", active: true,
    latest_template_version: 2, ...over
  };
}

export function preview(indicator: string, over: Partial<CarePreview> = {}): CarePreview {
  return {
    indicator, team: { id: TEAM_A.id, ine: TEAM_A.ine, name: TEAM_A.name }, template_version: 1, denominator: 41, score: "37.5",
    max_score: 100, concept: "sufficient",
    practices: [
      { key: "c4_a", label: "Consulta com médico ou enfermeiro", tracked: true, points: 20, weight: "25.0", achieved: 30, denominator: 41 },
      { key: "c4_d", label: "Visitas do ACS", tracked: false, points: 20, weight: "0.0", achieved: 0, denominator: 41 }
    ],
    excluded: [ "acs_visit", "vaccine", "oral_health" ], estimate: true, ...over
  };
}

export function requestRow(over: Partial<RequestRow> = {}): RequestRow {
  return {
    id: "rq1", kind: "return", origin: "attendance", origin_unit_name: "UBS Vila Verde", created_at: "2026-10-22T09:00:00-03:00",
    cpf_masked: "***.982.247-**", note: null, reopened_reason: null, appointment_type_key: "consulta_enfermagem",
    appointment_type_name: "Consulta de enfermagem", priority: "routine", due_on: "2026-11-21", triage_priority: null, overdue: false,
    reschedule_requested: false, reschedule_reason_code: null, preferred_period: null, reschedule_count: 0, needs_reschedule: false,
    target_unit_id: "u1", appointment: null, ...over
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/api.careLines.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS. (Se o `tsc` acusar campo de `RequestRow` que mudou no main, ajuste só o `requestRow` dos dados de teste ao tipo real.)

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.careLines.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/careLineFixtures.ts
$G commit -m "feat: add the care lines API client and switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 2: Regras de tela das linhas de cuidado (`src/lib/careLines.ts`)

**Files:**
- Create: `src/lib/careLines.ts`
- Test: `src/lib/careLines.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `ApiError`, `errorCode` (`api.ts`); `featureDisabledKey`, `hasFeature` (`features.ts`).
- Produces (todas exportadas de `src/lib/careLines.ts`):
  - constantes: `CARE_DISABLED`, `CARE_NO_ROLE`, `ESTIMATE_NOTICE`, `UNTRACKED_TEXT`, `NO_NAME`, `REASON_MAX = 500`, `NOTE_MAX = 500`, `NAME_MAX = 120`, `LABEL_MAX = 120`, `CITIZEN_LABEL_MAX = 80`, `PRACTICES_MAX = 20`, `MAX_WINDOW_DAYS = 7300`, `MAX_MIN_COUNT = 50`, `MAX_AGE_YEARS = 130`, `LMP_MAX_DAYS = 336`, `DOCTOR_NURSE_CBO`, `COUNT_REFRESH_MS = 60_000`; chaves `CONTEXT_KEY`, `ENROLLMENTS_KEY`, `ENROLLMENT_KEY`, `SUGGESTIONS_KEY`, `SUGGESTION_COUNT_KEY`, `OVERDUE_COUNT_KEY`, `TEMPLATES_KEY`, `LINES_KEY`, `PREVIEW_KEY`; opções de consulta `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`;
  - `canUseCareLines(user): boolean`, `careLineRoles(user): CareRoles` (`{ professional, admin, analyst }`), `personName(p): string`;
  - rótulos: `statusLabel(code)`, `statusTone(code)`, `reasonLabel(code)`, `endReasonLabel(code)`, `channelLabel(code)`, `outcomeLabel(code)`, `conditionLabel(code)`, `indicatorLabel(code)`, `sourceLabel(source)`, `anchorLabel(code)`, `ruleText(p: CarePractice)`, `originText(line)`, `excludedLabel(code)`, `conceptLabel(code)`; listas `END_REASONS`, `CHANNELS`, `OUTCOMES`, `CONDITIONS`, `INDICATORS`, `SOURCE_KINDS`, `CUSTOM_ANCHORS`, `ENROLLMENT_FILTERS`;
  - `fmtDay(iso)`, `decimalBr(n)`, `practiceText(p)`, `worstStatus(practices)`, `personStatus(e)`, `newerVersion(line, templates)`;
  - formulários: `reasonProblem(text)`, `noteProblem(text)`, `ContactDraft`, `EMPTY_CONTACT`, `contactProblem(d)`, `EndDraft`, `EMPTY_END`, `endProblem(d)`, `daysBetween(from, to)`, `anchorProblem(lmp, today)`, `PracticeDraft`, `CustomLineDraft`, `emptyPractice()`, `emptyCustomLine()`, `draftOfLine(line)`, `codeList(text)`, `slugKey(label, taken)`, `customLineProblem(d)`, `customLineBody(d): CareLineInput`;
  - `careLineError(err): string`, `isStaleCare(err): boolean`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/careLines.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  CARE_DISABLED, CARE_NO_ROLE, COUNT_QUERY, NOMINAL_QUERY, UNTRACKED_TEXT, anchorProblem, canUseCareLines, careLineError, careLineRoles,
  codeList, contactProblem, customLineBody, customLineProblem, decimalBr, draftOfLine, emptyCustomLine, emptyPractice, endProblem, fmtDay,
  isStaleCare, newerVersion, originText, personName, personStatus, practiceText, reasonProblem, ruleText, slugKey, sourceLabel, statusLabel,
  statusTone, worstStatus, type CustomLineDraft
} from "./careLines";
import { DIABETES_TEMPLATE, TODAY23, cityLine, practice } from "../test/careLineFixtures";

const user = (roles: string[], features = [ "clinical_record", "care_lines" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });

describe("papel e interruptor", () => {
  it("só com care_lines na sessão e fora do operador", () => {
    expect(canUseCareLines(user([ "health_professional" ]))).toBe(true);
    expect(canUseCareLines(user([ "health_professional" ], [ "clinical_record" ]))).toBe(false);
    expect(canUseCareLines(user([ "municipal_admin" ], [ "care_lines" ], true))).toBe(false);
    expect(canUseCareLines(null)).toBe(false);
    expect(careLineRoles(user([ "health_professional", "analyst" ]))).toEqual({ professional: true, admin: false, analyst: true });
    expect(careLineRoles(user([ "municipal_admin" ]))).toEqual({ professional: false, admin: true, analyst: false });
  });

  it("nome nulo tem frase própria", () => {
    expect(personName({ display_name: null })).toBe("cidadão sem nome no cadastro");
    expect(personName({ display_name: "Maria" })).toBe("Maria");
  });
});

describe("situação e boa prática", () => {
  it("rótulos e tons; código novo aparece cru", () => {
    expect(statusLabel("ok")).toBe("em dia");
    expect(statusLabel("due_soon")).toBe("vence em até 30 dias");
    expect(statusLabel("overdue")).toBe("vencida");
    expect(statusLabel("untracked")).toBe(UNTRACKED_TEXT);
    expect(statusLabel("paused")).toBe("paused");
    expect([ "ok", "due_soon", "overdue", "untracked", "paused" ].map(statusTone)).toEqual([ "ok", "warn", "down", "neutral", "neutral" ]);
  });

  it("frase da boa prática: última vez, vencimento e contagem proporcional", () => {
    expect(practiceText(practice({ status: "ok", done_at: "2026-07-10", due_on: "2027-01-06" })))
      .toBe("em dia · última em 10/07/2026 · vence em 06/01/2027");
    expect(practiceText(practice({ status: "overdue", done_at: "2025-08-20", due_on: "2026-08-20" })))
      .toBe("vencida · última em 20/08/2025 · venceu em 20/08/2026");
    expect(practiceText(practice({ status: "overdue", done_at: null, due_on: "2026-09-01", count: 2, expected: 4 })))
      .toBe("vencida · 2 de 4 esperadas até agora · nunca registrada · venceu em 01/09/2026");
  });

  it("untracked nunca piora e tem frase própria", () => {
    expect(practiceText(practice({ status: "untracked", done_at: null, due_on: "2026-01-01" }))).toBe(UNTRACKED_TEXT);
    expect(worstStatus([ practice({ status: "ok" }), practice({ status: "untracked" }) ])).toBe("ok");
    expect(worstStatus([ practice({ status: "untracked" }) ])).toBeNull();
    expect(worstStatus([ practice({ status: "due_soon" }), practice({ status: "overdue" }), practice({ status: "untracked" }) ])).toBe("overdue");
    expect(personStatus({ worst_status: null, practices: [ practice({ status: "due_soon" }), practice({ status: "untracked" }) ] })).toBe("due_soon");
    expect(personStatus({ worst_status: "ok", practices: [ practice({ status: "overdue" }) ] })).toBe("ok");
  });

  it("datas sem fuso, vírgula decimal e fontes", () => {
    expect(fmtDay("2026-10-22")).toBe("22/10/2026");
    expect(fmtDay("2026-10-22T08:00:00-03:00")).toBe("22/10/2026");
    expect(fmtDay(null)).toBe("—");
    expect(decimalBr(37.5)).toBe("37,5");
    expect(decimalBr(40)).toBe("40");
    expect(decimalBr(12.345)).toBe("12,3");
    expect(decimalBr(null)).toBe("—");
    expect(decimalBr("37.5")).toBe("37,5");
    expect(decimalBr("")).toBe("—");
    expect(sourceLabel("exam:0202010503")).toBe("exame SIGTAP 0202010503 (pedido ou resultado)");
    expect(sourceLabel("procedure:0301100039")).toBe("procedimento SIGTAP 0301100039");
    expect(sourceLabel("vaccine:9")).toBe(`vacina 9 — ${UNTRACKED_TEXT}`);
    expect(sourceLabel("acs_visit")).toBe(`visita do ACS — ${UNTRACKED_TEXT}`);
    expect(sourceLabel("teleconsulta")).toBe("teleconsulta");
    expect(ruleText(DIABETES_TEMPLATE.practices[0])).toBe("1× em 183 dias · janela móvel · só médico ou enfermeiro · 20 pontos");
    expect(ruleText({ ...DIABETES_TEMPLATE.practices[0], min_count: 7, window_days: 294, schedule: "proportional", anchor: "lmp", cbo_filter: null,
      points: null })).toBe("7× em 294 dias, proporcional ao tempo · desde a DUM");
    expect(ruleText({ ...DIABETES_TEMPLATE.practices[0], window_start_days: 196, window_days: 294, anchor: "lmp", cbo_filter: null, points: 9 }))
      .toBe("1× do dia 196 ao 294 · desde a DUM · 9 pontos");
  });

  it("origem e versão nova", () => {
    expect(originText(cityLine())).toBe("catálogo C4 · versão 1");
    expect(originText(cityLine({ source: "custom", template_key: null, template_version: null, indicator: null }))).toBe("própria da cidade");
    expect(newerVersion(cityLine(), [ DIABETES_TEMPLATE ])).toBe(2);
    expect(newerVersion(cityLine({ template_version: 2 }), [ DIABETES_TEMPLATE ])).toBeNull();
    expect(newerVersion(cityLine({ latest_template_version: undefined }), [ DIABETES_TEMPLATE ])).toBe(2);
    expect(newerVersion(cityLine({ source: "custom", template_key: null }), [ DIABETES_TEMPLATE ])).toBeNull();
  });
});

describe("formulários", () => {
  it("motivo de recusa: 1 a 500, recusado e nunca cortado", () => {
    expect(reasonProblem("   ")).toBe("escreva o motivo");
    expect(reasonProblem("convênio")).toBeNull();
    expect(reasonProblem("  já acompanhada no CAPS  ")).toBeNull();
    expect(reasonProblem("x".repeat(501))).toBe("motivo com até 500 caracteres — encurte");
  });

  it("contato: canal, resultado e nota até 500", () => {
    expect(contactProblem({ channel: "", outcome: "", note: "" })).toBe("escolha como foi o contato");
    expect(contactProblem({ channel: "phone", outcome: "", note: "" })).toBe("escolha o resultado do contato");
    expect(contactProblem({ channel: "phone", outcome: "no_answer", note: "" })).toBeNull();
    expect(contactProblem({ channel: "phone", outcome: "reached", note: "x".repeat(501) })).toBe("nota com até 500 caracteres — encurte");
  });

  it("encerramento: motivo obrigatório; outro motivo pede a nota", () => {
    expect(endProblem({ reason: "", note: "" })).toBe("escolha o motivo do encerramento");
    expect(endProblem({ reason: "moved", note: "" })).toBeNull();
    expect(endProblem({ reason: "other", note: "  " })).toBe("descreva o motivo em “outro motivo”");
    expect(endProblem({ reason: "other", note: "x".repeat(501) })).toBe("nota com até 500 caracteres — encurte");
  });

  it("DUM: obrigatória, válida, não futura e até 336 dias (gestação e puerpério)", () => {
    expect(anchorProblem("", TODAY23)).toBe("informe a DUM (data da última menstruação)");
    expect(anchorProblem("2026-02-30", TODAY23)).toBe("DUM inválida — confira a data");
    expect(anchorProblem("2026-10-23", TODAY23)).toBe("a DUM não pode ser depois de hoje");
    expect(anchorProblem("2025-11-19", TODAY23)).toBe("DUM de mais de 48 semanas (gestação e puerpério) — confira a data");
    expect(anchorProblem("2025-11-20", TODAY23)).toBeNull();
    expect(anchorProblem(TODAY23, TODAY23)).toBeNull();
  });
});

describe("linha própria", () => {
  const filled = (over: Partial<CustomLineDraft> = {}): CustomLineDraft => ({
    ...emptyCustomLine(), name: "Saúde mental", declaredCondition: "mental_health", ciap: "p76, P74", cid10: "f32.1 F41", ageMinYears: "18",
    practices: [ { ...emptyPractice(), label: "Consulta de acompanhamento", citizenLabel: "consulta com a equipe", windowDays: "90" },
      { ...emptyPractice(), label: "Consulta de acompanhamento", sourceKind: "procedure", sigtap: "0301080232", doctorOrNurse: false,
        anchor: "enrollment" } ],
    ...over
  });

  it("códigos: maiúsculas, sem ponto, sem repetir", () => {
    expect(codeList(" t90; t90 , e11.9 ")).toEqual([ "T90", "E119" ]);
    expect(codeList("")).toEqual([]);
  });

  it("chave da boa prática: sem acento, única e começando com letra", () => {
    const taken = new Set([ "consulta" ]);
    expect(slugKey("Consulta", taken)).toBe("consulta_2");
    expect(slugKey("Pressão arterial", new Set())).toBe("pressao_arterial");
    expect(slugKey("3 consultas", new Set())).toBe("p_3_consultas");
    expect(slugKey("!!!", new Set())).toBe("pratica");
  });

  it("recusa o que o api recusaria por formato", () => {
    expect(customLineProblem(filled({ name: "" }))).toBe("dê um nome à linha");
    expect(customLineProblem(filled({ name: "x".repeat(121) }))).toBe("nome com até 120 caracteres — encurte");
    expect(customLineProblem(filled({ ciap: "P76, PX" }))).toBe("código CIAP-2 inválido: PX");
    expect(customLineProblem(filled({ cid10: "F3" }))).toBe("código CID-10 inválido: F3");
    expect(customLineProblem(filled({ ageMinYears: "1,5" }))).toBe("idade em anos inteiros, de 0 a 130");
    expect(customLineProblem(filled({ ageMinYears: "60", ageMaxYears: "18" }))).toBe("a idade mínima passa da máxima");
    expect(customLineProblem(filled({ practices: [] }))).toBe("inclua pelo menos uma boa prática");
    expect(customLineProblem(filled({ practices: [ { ...emptyPractice(), label: "" } ] }))).toBe("boa prática 1: escreva o que deve ser feito");
    expect(customLineProblem(filled({ practices: [ { ...emptyPractice(), label: "Exame", sourceKind: "exam", sigtap: "02020105" } ] })))
      .toBe("boa prática 1: código SIGTAP com 10 dígitos");
    expect(customLineProblem(filled({ practices: [ { ...emptyPractice(), label: "Consulta", citizenLabel: "x".repeat(81) } ] })))
      .toBe("boa prática 1: texto para o cidadão até 80 caracteres — encurte");
    expect(customLineProblem(filled({ practices: [ { ...emptyPractice(), label: "Consulta", windowDays: "0" } ] })))
      .toBe("boa prática 1: janela de 1 a 7300 dias");
    expect(customLineProblem(filled({ practices: [ { ...emptyPractice(), label: "Consulta", minCount: "51" } ] })))
      .toBe("boa prática 1: quantidade de 1 a 50");
    expect(customLineProblem(filled())).toBeNull();
  });

  it("corpo (L1/L2): problemas, idade em anos, condição declarada, fonte com SIGTAP, CBO só na consulta, chaves únicas", () => {
    expect(customLineBody(filled())).toEqual({
      name: "Saúde mental", declared_condition: "mental_health",
      eligibility: { problems: { ciap2: [ "P76", "P74" ], cid10: [ "F321", "F41" ] }, age_min_years: 18, age_max_years: null, sex: null,
        declared: [ "mental_health" ], requires_anchor: null },
      practices: [
        { key: "consulta_de_acompanhamento", label: "Consulta de acompanhamento", citizen_label: "consulta com a equipe", source: "consultation",
          window_days: 90, min_count: 1, schedule: "end_of_window", anchor: "rolling", cbo_filter: [ "2231", "2251", "2252", "2253", "2235" ] },
        { key: "consulta_de_acompanhamento_2", label: "Consulta de acompanhamento", source: "procedure:0301080232", window_days: 180,
          min_count: 1, schedule: "end_of_window", anchor: "enrollment", cbo_filter: null }
      ]
    });
    expect(customLineBody(filled({ declaredCondition: "", ageMaxYears: "64" })).eligibility).toMatchObject({ age_max_years: 64, declared: [] });
    expect("declared_condition" in customLineBody(filled({ declaredCondition: "" }))).toBe(false);
  });

  it("editar: o rascunho da linha guarda as chaves e volta ao mesmo corpo", () => {
    const line = cityLine({ source: "custom", template_key: null, template_version: null, indicator: null, ...customLineBody(filled()) });
    const draft = draftOfLine(line);
    expect(draft.ageMinYears).toBe("18");
    expect(draft.practices.map((p) => p.key)).toEqual([ "consulta_de_acompanhamento", "consulta_de_acompanhamento_2" ]);
    expect(customLineBody(draft)).toEqual(customLineBody(filled()));
  });
});

describe("consultas e recusas", () => {
  it("nominal nunca relê sozinha; contador relê por minuto", () => {
    expect(NOMINAL_QUERY).toMatchObject({ refetchInterval: false, refetchOnWindowFocus: false, refetchOnReconnect: false, gcTime: 0 });
    expect(COUNT_QUERY.refetchInterval).toBe(60_000);
  });

  it("frases das recusas; código novo cru; stale relê", () => {
    const refusal = (status: number, error: string, extra: object = {}) => new ApiError(status, { error, ...extra }, "x");
    expect(careLineError(refusal(403, "feature_disabled", { feature: "care_lines" }))).toBe(CARE_DISABLED);
    expect(careLineError(refusal(403, "missing_role"))).toBe(CARE_NO_ROLE);
    expect(careLineError(refusal(403, "cbo_not_allowed"))).toBe("só médico ou enfermeiro confirma a inscrição");
    expect(careLineError(refusal(422, "anchor_required"))).toBe("informe a DUM para inscrever na linha da gestante");
    expect(careLineError(refusal(409, "already_enrolled"))).toBe("a pessoa já está inscrita nesta linha");
    expect(careLineError(refusal(409, "citizen_not_verified"))).toBe("o cadastro ainda não foi validado no posto — valide antes de inscrever");
    expect(careLineError(refusal(422, "duplicate_template"))).toBe("esta linha do catálogo já foi adotada pela cidade");
    expect(careLineError(refusal(422, "brand_new_code"))).toBe("brand_new_code");
    expect(careLineError(new ApiError(404, "", "x"))).toBe("registro não encontrado — atualize a tela");
    expect(careLineError(new TypeError("fetch"))).toBe("sem conexão com o servidor — tente de novo");
    expect(careLineError(refusal(403, "out_of_scope"))).toBe("você não tem vínculo ativo na unidade desta equipe");
    expect(careLineError(refusal(422, "invalid_anchor"))).toBe("DUM fora do prazo (até 48 semanas atrás) ou em linha que não pede DUM");
    expect(isStaleCare(refusal(409, "invalid_transition"))).toBe(true);
    expect(isStaleCare(refusal(404, "not_found"))).toBe(true);
    expect(isStaleCare(refusal(409, "already_enrolled"))).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/lib/careLines.test.ts`
Expected: FAIL — `./careLines` não existe.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/careLines.ts
// Regras de tela das linhas de cuidado (módulo 23; ADR 0040; contrato
// 2026-10-22). Elegibilidade, situação de cada boa prática, pior situação,
// quem confirma e a prévia são do api: aqui só se traduzem os códigos, se
// monta o corpo e se recusa o que o api recusaria por formato. Nada inscreve
// sem o clique de um profissional (invariante do ADR 0040).
import {
  ApiError, errorCode, type CareEnrollment, type CareIndicator, type CareLine, type CareLineInput, type CareLineTemplate, type CarePractice,
  type CarePracticeStatus, type CareSchedule, type CareStatus, type ContactChannel, type ContactOutcome, type DeclaredCondition,
  type EndReason
} from "./api";
import { featureDisabledKey, hasFeature } from "./features";

export const CARE_DISABLED = "as linhas de cuidado estão desligadas nesta cidade";
export const CARE_NO_ROLE = "seu papel não permite usar as linhas de cuidado";
export const ESTIMATE_NOTICE = "Estimativa — o valor oficial é calculado pelo SIAPS; visitas do ACS e vacinas não entram.";
export const UNTRACKED_TEXT = "não acompanhado pelo sistema";
export const NO_NAME = "cidadão sem nome no cadastro";
export const personName = (p: { display_name: string | null } | null | undefined): string => p?.display_name || NO_NAME;

export const CONTEXT_KEY = "careContext";
export const ENROLLMENTS_KEY = "careEnrollments";
export const ENROLLMENT_KEY = "careEnrollment";
export const SUGGESTIONS_KEY = "careSuggestions";
export const SUGGESTION_COUNT_KEY = "careSuggestionCount";
export const OVERDUE_COUNT_KEY = "careOverdueCount";
export const TEMPLATES_KEY = "careTemplates";
export const LINES_KEY = "careCityLines";
export const PREVIEW_KEY = "carePreview";

export const COUNT_REFRESH_MS = 60_000;
// Leitura nominal grava trilha (clinical_record.viewed, access: care_line): nunca relê sozinha.
export const NOMINAL_QUERY = {
  retry: false, staleTime: 0, gcTime: 0, refetchInterval: false, refetchOnWindowFocus: false, refetchOnReconnect: false
} as const;
// Contadores (/count) não gravam trilha: releem a cada minuto.
export const COUNT_QUERY = {
  retry: false, staleTime: 0, gcTime: 0, refetchInterval: COUNT_REFRESH_MS, refetchOnWindowFocus: false, refetchOnReconnect: false
} as const;
// Sem pessoa (catálogo, linhas, contexto): lê ao abrir e depois de uma ação.
export const STATIC_QUERY = { retry: false, refetchOnWindowFocus: false, refetchOnReconnect: false } as const;

type SessionLike = { operator: boolean; memberships?: { role: string }[]; features?: unknown } | null | undefined;

export function canUseCareLines(user: SessionLike): boolean {
  return !!user && !user.operator && hasFeature(user, "care_lines");
}

export interface CareRoles { professional: boolean; admin: boolean; analyst: boolean }

export function careLineRoles(user: SessionLike): CareRoles {
  const roles = user?.memberships?.map((m) => m.role) ?? [];
  return { professional: roles.includes("health_professional"), admin: roles.includes("municipal_admin"), analyst: roles.includes("analyst") };
}

// ─── Rótulos (contrato §1). Código novo do api aparece cru. ────────────────
const STATUS_LABEL: Record<string, string> = { ok: "em dia", due_soon: "vence em até 30 dias", overdue: "vencida", untracked: UNTRACKED_TEXT };
export function statusLabel(code: string | null | undefined): string {
  return code ? STATUS_LABEL[code] ?? code : "sem boa prática acompanhada";
}
const STATUS_TONE: Record<string, string> = { ok: "ok", due_soon: "warn", overdue: "down", untracked: "neutral" };
export function statusTone(code: string | null | undefined): string {
  return (code && STATUS_TONE[code]) || "neutral";
}

const REASON_LABEL: Record<string, string> = {
  problem: "problema ativo na lista de problemas", age_sex: "idade e sexo", declared: "condição declarada pelo cidadão",
  resolved_problem: "problema resolvido — sugestão de encerrar"
};
export function reasonLabel(code: string): string { return REASON_LABEL[code] ?? code; }

export const END_REASONS: { key: EndReason; label: string }[] = [
  { key: "discharge", label: "alta da linha" }, { key: "death", label: "óbito" }, { key: "moved", label: "mudou de território" },
  { key: "pregnancy_ended", label: "fim da gestação" }, { key: "other", label: "outro motivo" }
];
export function endReasonLabel(code: string): string { return END_REASONS.find((r) => r.key === code)?.label ?? code; }

export const CHANNELS: { key: ContactChannel; label: string }[] = [
  { key: "phone", label: "telefone" }, { key: "home_visit", label: "visita domiciliar" },
  { key: "wpda_message", label: "mensagem pelo canal do cidadão" }, { key: "in_person", label: "presencial na unidade" }
];
export function channelLabel(code: string): string { return CHANNELS.find((c) => c.key === code)?.label ?? code; }

export const OUTCOMES: { key: ContactOutcome; label: string }[] = [
  { key: "reached", label: "conversou" }, { key: "no_answer", label: "não atendeu" }, { key: "scheduled", label: "agendou" },
  { key: "refused", label: "recusou" }
];
export function outcomeLabel(code: string): string { return OUTCOMES.find((o) => o.key === code)?.label ?? code; }

export const CONDITIONS: { key: Exclude<DeclaredCondition, "none">; label: string }[] = [
  { key: "diabetes", label: "diabetes" }, { key: "hypertension", label: "hipertensão" }, { key: "pregnancy", label: "gestação" },
  { key: "mental_health", label: "saúde mental" }
];
export function conditionLabel(code: string): string { return CONDITIONS.find((c) => c.key === code)?.label ?? code; }

export const INDICATORS: { key: CareIndicator; label: string }[] = [
  { key: "C2", label: "Cuidado da criança até 2 anos" }, { key: "C3", label: "Gestante e puérpera" },
  { key: "C4", label: "Pessoa com diabetes" }, { key: "C5", label: "Pessoa com hipertensão" },
  { key: "C6", label: "Pessoa idosa" }, { key: "C7", label: "Prevenção do câncer na mulher" }
];
export function indicatorLabel(code: string): string { return INDICATORS.find((i) => i.key === code)?.label ?? code; }

// L8: famílias de prática que a prévia deixa fora (o sistema não registra).
const EXCLUDED_LABEL: Record<string, string> = { acs_visit: "visita do ACS", vaccine: "vacina", oral_health: "saúde bucal" };
export function excludedLabel(code: string): string { return EXCLUDED_LABEL[code] ?? code; }

const CONCEPT_LABEL: Record<string, string> = { optimal: "ótimo", good: "bom", sufficient: "suficiente", regular: "regular" };
export function conceptLabel(code: string | null | undefined): string { return code ? CONCEPT_LABEL[code] ?? code : "—"; }

const SOURCE_LABEL: Record<string, string> = {
  consultation: "consulta finalizada", blood_pressure: "pressão arterial (escuta ou consulta)",
  weight_height: "peso e altura (escuta ou consulta)", acs_visit: `visita do ACS — ${UNTRACKED_TEXT}`,
  oral_health: `saúde bucal — ${UNTRACKED_TEXT}`
};
export function sourceLabel(source: string): string {
  if (SOURCE_LABEL[source]) return SOURCE_LABEL[source];
  const [ kind, code ] = source.split(":");
  if (kind === "exam" && code) return `exame SIGTAP ${code} (pedido ou resultado)`;
  if (kind === "procedure" && code) return `procedimento SIGTAP ${code}`;
  if (kind === "vaccine" && code) return `vacina ${code} — ${UNTRACKED_TEXT}`;
  return source;
}

const ANCHOR_LABEL: Record<string, string> = {
  enrollment: "desde a inscrição", birth: "desde o nascimento", lmp: "desde a DUM", rolling: "janela móvel"
};
export function anchorLabel(code: string): string { return ANCHOR_LABEL[code] ?? code; }

export function ruleText(p: CarePractice): string {
  const span = p.window_start_days ? `do dia ${p.window_start_days} ao ${p.window_days}` : `em ${p.window_days} dias`;
  const count = `${p.min_count}× ${span}${p.schedule === "proportional" ? ", proporcional ao tempo" : ""}`;
  const parts = [ count, anchorLabel(p.anchor) ];
  if (p.cbo_filter && p.cbo_filter.length > 0) parts.push("só médico ou enfermeiro");
  if (typeof p.points === "number") parts.push(`${p.points} pontos`);
  return parts.join(" · ");
}

export function originText(line: Pick<CareLine, "source" | "indicator" | "template_version">): string {
  return line.source === "custom" ? "própria da cidade" : `catálogo ${line.indicator ?? "—"} · versão ${line.template_version ?? "—"}`;
}

// Versão do catálogo mais nova que a adotada (D10); null se não há.
export function newerVersion(line: CareLine, templates: CareLineTemplate[]): number | null {
  if (line.source !== "template" || !line.template_key || line.template_version === null) return null;
  const latest = line.latest_template_version ?? templates.find((t) => t.key === line.template_key)?.version ?? null;
  return latest !== null && latest > line.template_version ? latest : null;
}

// ─── Datas, números e situação ────────────────────────────────────────────
// Dia de calendário sem new Date (nenhum fuso desloca a data).
export function fmtDay(iso: string | null | undefined): string {
  if (!iso) return "—";
  const m = /^(\d{4})-(\d{2})-(\d{2})/.exec(iso);
  return m ? `${m[3]}/${m[2]}/${m[1]}` : iso;
}

// Vírgula decimal, uma casa (lição do contrato do 21a §15). A prévia manda
// texto com ponto (L8: "37.5"); a tela nunca mostra o ponto.
export function decimalBr(n: number | string | null | undefined): string {
  const v = typeof n === "string" && n.trim() !== "" ? Number(n) : n;
  if (typeof v !== "number" || !Number.isFinite(v)) return "—";
  return String(Math.round(v * 10) / 10).replace(".", ",");
}

export function practiceText(p: CarePracticeStatus): string {
  if (p.status === "untracked") return UNTRACKED_TEXT;
  const parts = [ statusLabel(p.status) ];
  if (p.count !== null && p.expected !== null) parts.push(`${p.count} de ${p.expected} esperadas até agora`);
  parts.push(p.done_at ? `última em ${fmtDay(p.done_at)}` : "nunca registrada");
  if (p.due_on) parts.push(`${p.status === "overdue" ? "venceu em" : "vence em"} ${fmtDay(p.due_on)}`);
  return parts.join(" · ");
}

// untracked (e código desconhecido) não piora a pessoa (ADR 0040).
const SEVERITY: Record<string, number> = { ok: 0, due_soon: 1, overdue: 2 };
export function worstStatus(practices: Pick<CarePracticeStatus, "status">[]): CareStatus | null {
  let worst: string | null = null;
  for (const p of practices) {
    if (!(p.status in SEVERITY)) continue;
    if (worst === null || SEVERITY[p.status] > SEVERITY[worst]) worst = p.status;
  }
  return worst as CareStatus | null;
}
// O api manda `worst_status`; a tela só calcula quando ele não vem.
export function personStatus(e: Pick<CareEnrollment, "worst_status" | "practices">): string | null {
  return e.worst_status ?? worstStatus(e.practices);
}

export const ENROLLMENT_FILTERS: { key: "overdue" | "due_soon" | "all"; label: string }[] = [
  { key: "overdue", label: "Busca ativa" }, { key: "due_soon", label: "Vencendo" }, { key: "all", label: "Todos" }
];

// ─── Formulários ─────────────────────────────────────────────────────────
// Limites do plano do api ("Valores fixados" 4).
export const REASON_MAX = 500;
export const NOTE_MAX = 500;
export const NAME_MAX = 120;
export const LABEL_MAX = 120;
export const CITIZEN_LABEL_MAX = 80;
export const PRACTICES_MAX = 20;
export const MAX_WINDOW_DAYS = 7300;
export const MAX_MIN_COUNT = 50;
export const MAX_AGE_YEARS = 130;
export const LMP_MAX_DAYS = 336; // gestação (294) + puerpério (42)
// Consulta que conta (notas C2–C7, item 24): médico 2231/2251/2252/2253, enfermeiro 2235.
export const DOCTOR_NURSE_CBO = [ "2231", "2251", "2252", "2253", "2235" ];

export function reasonProblem(text: string): string | null {
  const t = text.trim();
  if (t.length === 0) return "escreva o motivo";
  if (t.length > REASON_MAX) return `motivo com até ${REASON_MAX} caracteres — encurte`;
  return null;
}

export function noteProblem(text: string): string | null {
  return text.trim().length > NOTE_MAX ? `nota com até ${NOTE_MAX} caracteres — encurte` : null;
}

export interface ContactDraft { channel: ContactChannel | ""; outcome: ContactOutcome | ""; note: string }
export const EMPTY_CONTACT: ContactDraft = { channel: "", outcome: "", note: "" };
export function contactProblem(d: ContactDraft): string | null {
  if (!d.channel) return "escolha como foi o contato";
  if (!d.outcome) return "escolha o resultado do contato";
  return noteProblem(d.note);
}

export interface EndDraft { reason: EndReason | ""; note: string }
export const EMPTY_END: EndDraft = { reason: "", note: "" };
export function endProblem(d: EndDraft): string | null {
  if (!d.reason) return "escolha o motivo do encerramento";
  if (d.reason === "other" && d.note.trim() === "") return "descreva o motivo em “outro motivo”";
  return noteProblem(d.note);
}

const ISO_DAY = /^\d{4}-\d{2}-\d{2}$/;
function validDay(s: string): boolean {
  if (!ISO_DAY.test(s)) return false;
  const t = Date.parse(`${s}T12:00:00Z`);
  return !Number.isNaN(t) && new Date(t).toISOString().slice(0, 10) === s;
}
export function daysBetween(from: string, to: string): number {
  return Math.round((Date.parse(`${to}T12:00:00Z`) - Date.parse(`${from}T12:00:00Z`)) / 86_400_000);
}
// DUM da linha da gestante (decisões do coordenador; L4 invalid_anchor): obrigatória, entre hoje − 336 e hoje.
export function anchorProblem(lmp: string, today: string): string | null {
  if (!lmp) return "informe a DUM (data da última menstruação)";
  if (!validDay(lmp)) return "DUM inválida — confira a data";
  if (lmp > today) return "a DUM não pode ser depois de hoje";
  if (daysBetween(lmp, today) > LMP_MAX_DAYS) return "DUM de mais de 48 semanas (gestação e puerpério) — confira a data";
  return null;
}

// ─── Linha própria (mesma forma das do catálogo; D1) ──────────────────────
export type PracticeSourceKind = "consultation" | "blood_pressure" | "weight_height" | "exam" | "procedure";
export const SOURCE_KINDS: { key: PracticeSourceKind; label: string }[] = [
  { key: "consultation", label: "consulta" }, { key: "blood_pressure", label: "pressão arterial" },
  { key: "weight_height", label: "peso e altura" }, { key: "exam", label: "exame (código SIGTAP)" },
  { key: "procedure", label: "procedimento (código SIGTAP)" }
];
// Nascimento e DUM são das linhas federais; a própria conta da inscrição ou em janela móvel.
export const CUSTOM_ANCHORS: { key: "enrollment" | "rolling"; label: string }[] = [
  { key: "rolling", label: "janela móvel (últimos N dias)" }, { key: "enrollment", label: "desde a inscrição" }
];

export interface PracticeDraft {
  key: string; label: string; citizenLabel: string; sourceKind: PracticeSourceKind | string; sigtap: string; windowDays: string;
  minCount: string; schedule: CareSchedule | string; anchor: string; doctorOrNurse: boolean;
}
export interface CustomLineDraft {
  name: string; declaredCondition: "" | DeclaredCondition; ciap: string; cid10: string; ageMinYears: string; ageMaxYears: string;
  sex: "" | "female" | "male"; practices: PracticeDraft[];
}

export function emptyPractice(): PracticeDraft {
  return { key: "", label: "", citizenLabel: "", sourceKind: "consultation", sigtap: "", windowDays: "180", minCount: "1",
    schedule: "end_of_window", anchor: "rolling", doctorOrNurse: true };
}
export function emptyCustomLine(): CustomLineDraft {
  return { name: "", declaredCondition: "", ciap: "", cid10: "", ageMinYears: "", ageMaxYears: "", sex: "", practices: [ emptyPractice() ] };
}

function practiceDraftOf(p: CarePractice): PracticeDraft {
  const [ kind, code ] = p.source.split(":");
  return {
    key: p.key, label: p.label, citizenLabel: p.citizen_label ?? "", sourceKind: kind, sigtap: code ?? "", windowDays: String(p.window_days),
    minCount: String(p.min_count), schedule: p.schedule, anchor: p.anchor, doctorOrNurse: !!p.cbo_filter && p.cbo_filter.length > 0
  };
}

export function draftOfLine(line: CareLine): CustomLineDraft {
  const e = line.eligibility;
  return {
    name: line.name, declaredCondition: (line.declared_condition ?? "") as CustomLineDraft["declaredCondition"],
    ciap: e.problems.ciap2.join(", "), cid10: e.problems.cid10.join(", "),
    ageMinYears: e.age_min_years == null ? "" : String(e.age_min_years),
    ageMaxYears: e.age_max_years == null ? "" : String(e.age_max_years),
    sex: e.sex ?? "", practices: line.practices.map(practiceDraftOf)
  };
}

export function codeList(text: string): string[] {
  return [ ...new Set(text.toUpperCase().replace(/\./g, "").split(/[\s,;]+/).filter(Boolean)) ];
}

export function slugKey(label: string, taken: Set<string>): string {
  const raw = label.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().replace(/[^a-z0-9]+/g, "_").replace(/^_+|_+$/g, "").slice(0, 40);
  const base = raw === "" ? "pratica" : /^[a-z]/.test(raw) ? raw : `p_${raw}`;
  let key = base;
  for (let n = 2; taken.has(key); n += 1) key = `${base}_${n}`;
  return key;
}

const CIAP_RE = /^[A-Z]\d{2}$/;
const CID_RE = /^[A-Z]\d{2}[0-9X]?$/;
const SIGTAP_RE = /^\d{10}$/;
const isSigtapKind = (kind: string) => kind === "exam" || kind === "procedure";

function intOf(raw: string): number | null {
  const t = raw.trim();
  return /^\d+$/.test(t) ? Number(t) : null;
}
// "" = sem limite (null); inválido = undefined.
function yearsOf(raw: string): number | null | undefined {
  if (raw.trim() === "") return null;
  const n = intOf(raw);
  return n === null || n > MAX_AGE_YEARS ? undefined : n;
}

export function customLineProblem(d: CustomLineDraft): string | null {
  const name = d.name.trim();
  if (!name) return "dê um nome à linha";
  if (name.length > NAME_MAX) return `nome com até ${NAME_MAX} caracteres — encurte`;
  const badCiap = codeList(d.ciap).find((c) => !CIAP_RE.test(c));
  if (badCiap) return `código CIAP-2 inválido: ${badCiap}`;
  const badCid = codeList(d.cid10).find((c) => !CID_RE.test(c));
  if (badCid) return `código CID-10 inválido: ${badCid}`;
  const min = yearsOf(d.ageMinYears);
  const max = yearsOf(d.ageMaxYears);
  if (min === undefined || max === undefined) return `idade em anos inteiros, de 0 a ${MAX_AGE_YEARS}`;
  if (min !== null && max !== null && min > max) return "a idade mínima passa da máxima";
  if (d.practices.length === 0) return "inclua pelo menos uma boa prática";
  if (d.practices.length > PRACTICES_MAX) return `no máximo ${PRACTICES_MAX} boas práticas`;
  for (const [ i, p ] of d.practices.entries()) {
    const n = `boa prática ${i + 1}`;
    const label = p.label.trim();
    if (!label) return `${n}: escreva o que deve ser feito`;
    if (label.length > LABEL_MAX) return `${n}: até ${LABEL_MAX} caracteres — encurte`;
    if (p.citizenLabel.trim().length > CITIZEN_LABEL_MAX) return `${n}: texto para o cidadão até ${CITIZEN_LABEL_MAX} caracteres — encurte`;
    if (isSigtapKind(p.sourceKind) && !SIGTAP_RE.test(p.sigtap.trim())) return `${n}: código SIGTAP com 10 dígitos`;
    const w = intOf(p.windowDays);
    if (w === null || w < 1 || w > MAX_WINDOW_DAYS) return `${n}: janela de 1 a ${MAX_WINDOW_DAYS} dias`;
    const c = intOf(p.minCount);
    if (c === null || c < 1 || c > MAX_MIN_COUNT) return `${n}: quantidade de 1 a ${MAX_MIN_COUNT}`;
  }
  return null;
}

// Só depois de customLineProblem(d) === null. Forma L1/L2 do plano do api;
// a linha própria não usa os campos avançados (all_of, code_filter, applies_to).
export function customLineBody(d: CustomLineDraft): CareLineInput {
  const taken = new Set(d.practices.map((p) => p.key).filter(Boolean));
  const practices: CarePractice[] = d.practices.map((p) => {
    const key = p.key || slugKey(p.label, taken);
    taken.add(key);
    const citizen = p.citizenLabel.trim();
    return {
      key, label: p.label.trim(), ...(citizen ? { citizen_label: citizen } : {}),
      source: isSigtapKind(p.sourceKind) ? `${p.sourceKind}:${p.sigtap.trim()}` : p.sourceKind,
      window_days: Number(p.windowDays.trim()), min_count: Number(p.minCount.trim()), schedule: p.schedule, anchor: p.anchor,
      cbo_filter: p.sourceKind === "consultation" && p.doctorOrNurse ? [ ...DOCTOR_NURSE_CBO ] : null
    };
  });
  return {
    name: d.name.trim(),
    eligibility: {
      problems: { ciap2: codeList(d.ciap), cid10: codeList(d.cid10) },
      age_min_years: yearsOf(d.ageMinYears) ?? null, age_max_years: yearsOf(d.ageMaxYears) ?? null, sex: d.sex || null,
      declared: d.declaredCondition ? [ d.declaredCondition ] : [], requires_anchor: null
    },
    practices,
    ...(d.declaredCondition ? { declared_condition: d.declaredCondition } : {})
  };
}

// ─── Recusas do api ──────────────────────────────────────────────────────
// Códigos do contrato e do plano do api (L4).
const ERRORS: Record<string, string> = {
  missing_role: CARE_NO_ROLE,
  out_of_scope: "você não tem vínculo ativo na unidade desta equipe",
  cbo_not_allowed: "só médico ou enfermeiro confirma a inscrição",
  anchor_required: "informe a DUM para inscrever na linha da gestante",
  invalid_anchor: "DUM fora do prazo (até 48 semanas atrás) ou em linha que não pede DUM",
  invalid_transition: "isso já foi decidido ou encerrado por outra pessoa — a lista foi relida",
  immutable_rules: "as regras de uma linha do catálogo não mudam aqui — adote a versão nova",
  duplicate_declared_condition: "já existe uma linha da cidade para esta condição declarada",
  template_not_found: "versão do catálogo não encontrada — atualize a tela",
  invalid_name: "nome inválido — de 1 a 120 caracteres",
  invalid_declared_condition: "condição declarada desconhecida — atualize a tela",
  invalid_person: "pessoa inválida — busque de novo pelo CPF",
  invalid_care_line: "linha inválida — atualize a tela",
  invalid_filter: "filtro inválido — atualize a tela",
  invalid_indicator: "indicador inválido — use C2 a C7",
  already_enrolled: "a pessoa já está inscrita nesta linha",
  citizen_not_verified: "o cadastro ainda não foi validado no posto — valide antes de inscrever",
  invalid_practices: "boas práticas inválidas — confira os campos",
  invalid_eligibility: "regras de elegibilidade inválidas — confira os códigos e as idades",
  duplicate_template: "esta linha do catálogo já foi adotada pela cidade",
  reason_required: "escreva o motivo",
  too_long: "texto maior que o permitido — encurte",
  invalid_reason: "motivo desconhecido — atualize a tela",
  invalid_channel: "canal de contato desconhecido — atualize a tela",
  invalid_outcome: "resultado de contato desconhecido — atualize a tela",
  line_inactive: "esta linha foi desativada pela administração",
  invalid_cpf: "CPF inválido",
  person_not_found: "nenhum cidadão com este CPF na cidade",
  no_open_slot_unit: "a equipe não tem unidade com agenda para marcar — fale com a recepção",
  invalid_cursor: "a lista mudou enquanto você rolava — recomeçando do início",
  not_found: "registro não encontrado — atualize a tela"
};

export function careLineError(err: unknown): string {
  if (featureDisabledKey(err) !== null) return CARE_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  if (code) return code;
  if (err instanceof ApiError && err.status === 404) return ERRORS.not_found;
  if (err instanceof ApiError) return `não foi possível concluir (${err.status}) — tente de novo`;
  return "sem conexão com o servidor — tente de novo";
}

// A lista mudou por outra pessoa: quem chama relê.
export function isStaleCare(err: unknown): boolean {
  const code = errorCode(err);
  return code === "not_found" || code === "invalid_transition" || code === "invalid_cursor";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/lib/careLines.test.ts && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/careLines.ts src/lib/careLines.test.ts
$G commit -m "feat: add care lines screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Linhas de cuidado", catálogo do shell e portão

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`
- Create: `src/modules/careLines/CareLinesGate.tsx`
- Test: `src/shell/modules.careLines.test.ts`

**Interfaces:**
- Consumes: `hasFeature` (`features.ts`); `canUseCareLines`, `careLineRoles`, `CARE_DISABLED`, `CARE_NO_ROLE`, `CareRoles` (Task 2).
- Produces: `ModuleId` com `"care-lines-team" | "care-lines-preview" | "care-lines-admin"`; o grupo "Linhas de cuidado" logo depois do "Atendimento"; `CareLinesGate({ title, sub, allow(roles), children(roles) })`.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.careLines.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor } from "./modules";
import { CATALOG } from "./catalog";

const CARE_IDS = [ "care-lines-team", "care-lines-preview", "care-lines-admin" ];
const ON = { features: [ "clinical_record", "care_lines" ] };
const userWith = (roles: string[], extra: object = ON) => ({ operator: false, memberships: roles.map((role) => ({ role })), ...extra });
const careIds = (user: Parameters<typeof navGroupsFor>[0]) =>
  navGroupsFor(user).flatMap((g) => g.items.map((i) => i.id)).filter((id) => CARE_IDS.includes(id));

describe("grupo Linhas de cuidado (módulo 23)", () => {
  it("fica logo depois do Atendimento, com as três telas, ícone e descrição", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Linhas de cuidado")).toBe(labels.indexOf("Atendimento") + 1);
    expect(NAV_GROUPS.find((g) => g.label === "Linhas de cuidado")?.items.map((i) => i.id)).toEqual(CARE_IDS);
    for (const id of CARE_IDS) expect(CATALOG.screens[id]?.description).toBeTruthy();
  });

  it("sem care_lines, ninguém vê o grupo (nem com clinical_record)", () => {
    for (const role of [ "health_professional", "municipal_admin", "analyst" ]) {
      expect(careIds(userWith([ role ], { features: [ "clinical_record" ] }))).toEqual([]);
    }
    expect(navGroupsFor(userWith([ "health_professional" ], {})).some((g) => g.label === "Linhas de cuidado")).toBe(false);
  });

  it("profissional vê o painel; admin a prévia e as linhas; analyst só a prévia", () => {
    expect(careIds(userWith([ "health_professional" ]))).toEqual([ "care-lines-team" ]);
    expect(careIds(userWith([ "municipal_admin" ]))).toEqual([ "care-lines-preview", "care-lines-admin" ]);
    expect(careIds(userWith([ "analyst" ]))).toEqual([ "care-lines-preview" ]);
    expect(careIds(userWith([ "health_professional", "municipal_admin" ]))).toEqual(CARE_IDS);
    for (const role of [ "citizen_verifier", "billing_clerk", "lab_staff" ]) expect(careIds(userWith([ role ]))).toEqual([]);
  });

  it("operador e sessão vazia nunca", () => {
    expect(careIds({ operator: true, memberships: [], ...ON })).toEqual([]);
    expect(careIds(null)).toEqual([]);
  });
});
```

Em `src/shell/modules.test.ts`, troque

```ts
      expect(groups.length).toBe(NAV_GROUPS.length - 10);
```

por

```ts
      expect(groups.some((g) => g.label === "Linhas de cuidado")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 11);
```

e

```ts
      expect(groups.length).toBe(NAV_GROUPS.length - 9);
```

por

```ts
      expect(groups.some((g) => g.label === "Linhas de cuidado")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 10);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/shell/modules.careLines.test.ts src/shell/modules.test.ts src/shell/catalog.test.ts`
Expected: FAIL — não há o grupo; as contagens antigas.

- [ ] **Step 3: Write minimal implementation**

Em `src/shell/modules.ts`, troque

```ts
  | "billing-procedures" | "billing-periods" | "billing-settings";
```

por

```ts
  | "billing-procedures" | "billing-periods" | "billing-settings"
  | "care-lines-team" | "care-lines-preview" | "care-lines-admin";
```

Depois do item `{ id: "open-drafts", label: "Rascunhos abertos", icon: "✎" }` e do `]},` que fecha o grupo "Atendimento", acrescente o grupo:

```ts
  { label: "Linhas de cuidado", items: [
    { id: "care-lines-team", label: "Acompanhamento da equipe", icon: "♡" },
    { id: "care-lines-preview", label: "Prévia dos indicadores", icon: "◑" },
    { id: "care-lines-admin", label: "Linhas da cidade", icon: "⚙" }
  ]},
```

Em `navGroupsFor`, depois de `  const isBillingClerk = roles.includes("billing_clerk");`, acrescente

```ts
  // Módulo 23 (ADR 0040): linhas de cuidado com `care_lines` (que exige
  // clinical_record). O painel nominal é do profissional (o api confere o
  // vínculo na unidade da equipe); a prévia agregada, do admin e do analyst;
  // adotar e criar linhas, só do admin.
  const careOn = !user?.operator && hasFeature(user, "care_lines");
  const isAnalyst = roles.includes("analyst");
```

e, antes de `      if (item.id === "integrations" || item.id === "cnes") return canIntegrations;`, acrescente

```ts
      if (item.id === "care-lines-team") return careOn && isProfessional;
      if (item.id === "care-lines-preview") return careOn && (isAdmin || isAnalyst);
      if (item.id === "care-lines-admin") return careOn && isAdmin;
```

Em `src/shell/catalog.ts`, troque

```ts
  IconUser, IconUserCircle, IconUsers, IconUsersGroup, IconCalendarStats, IconFileInvoice, IconReceipt, IconReceiptTax
```

por

```ts
  IconUser, IconUserCircle, IconUsers, IconUsersGroup, IconCalendarStats, IconFileInvoice, IconReceipt, IconReceiptTax,
  IconHeartHandshake, IconUserSearch, IconGauge, IconRoute
```

em `groups`, depois da linha `"Atendimento": …`, acrescente

```ts
    "Linhas de cuidado": { icon: IconHeartHandshake, description: "Busca ativa, sugestões e prévia dos indicadores" },
```

e, em `screens`, depois de `"billing-settings": …`, acrescente

```ts
    "care-lines-team": { icon: IconUserSearch, description: "Quem procurar, sugestões e inscrições da equipe" },
    "care-lines-preview": { icon: IconGauge, description: "Estimativa de C2 a C7 por equipe" },
    "care-lines-admin": { icon: IconRoute, description: "Adotar linhas do catálogo e criar linhas próprias" },
```

Crie `src/modules/careLines/CareLinesGate.tsx`:

```tsx
// src/modules/careLines/CareLinesGate.tsx
// Portão das telas das linhas de cuidado: interruptor e papel antes de
// qualquer leitura (o api recusaria de novo; aqui só não se oferece a porta).
import type { ReactNode } from "react";
import { useAuth } from "../../lib/auth";
import { CARE_DISABLED, CARE_NO_ROLE, canUseCareLines, careLineRoles, type CareRoles } from "../../lib/careLines";
import { PageHeader } from "../../components/PageHeader";
import { EmptyState } from "../../components/EmptyState";
import { stack } from "../exams/styles";

export function CareLinesGate({ title, sub, allow, children }: {
  title: string; sub: string; allow(roles: CareRoles): boolean; children(roles: CareRoles): ReactNode;
}) {
  const { user } = useAuth();
  if (!user) return null;
  const roles = careLineRoles(user);
  let body: ReactNode;
  if (!canUseCareLines(user)) body = <EmptyState title={CARE_DISABLED} />;
  else if (!allow(roles)) body = <EmptyState title={CARE_NO_ROLE} />;
  else body = children(roles);
  return (
    <div style={stack}>
      <PageHeader title={title} sub={sub} />
      {body}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/shell && npx tsc --noEmit`
Expected: PASS (o `catalog.test.ts` cobra ícone e descrição das três telas e do grupo).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.careLines.test.ts src/shell/catalog.ts src/modules/careLines/CareLinesGate.tsx
$G commit -m "feat: add the care lines menu group and gate

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 4: Linhas da cidade — adotar, adotar versão nova, ativar e desativar

**Files:**
- Create: `src/modules/CareLinesAdmin.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/CareLinesAdmin.test.tsx`

**Interfaces:**
- Consumes: `listCareTemplates`, `listCityCareLines`, `adoptCareLine`, `updateCareLine`, `errorCode`, `CareLine`, `CareLineTemplate` (Task 1); `TEMPLATES_KEY`, `LINES_KEY`, `STATIC_QUERY`, `careLineError`, `conditionLabel`, `newerVersion`, `originText`, `ruleText`, `sourceLabel` (Task 2); `CareLinesGate` (Task 3); `SensitiveAction`, `Panel`, `DataTable`, `Tag`, `EmptyState`.
- Produces: `CareLinesAdmin({ onGoToSecurity? })`; dentro dele, `AdminBody` com o estado `editing` que a Task 5 usa (ver Task 5).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/CareLinesAdmin.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), listCareTemplates: vi.fn(), listCityCareLines: vi.fn(),
    adoptCareLine: vi.fn(), updateCareLine: vi.fn(), createCustomCareLine: vi.fn()
  };
});

import * as api from "../lib/api";
import { CareLinesAdmin } from "./CareLinesAdmin";
import { CARE_DISABLED, CARE_NO_ROLE } from "../lib/careLines";
import { renderWithProviders } from "../test/campaignFixtures";
import { DIABETES_TEMPLATE, PREGNANCY_TEMPLATE, careUser, cityLine } from "../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.listCareTemplates, api.listCityCareLines, api.adoptCareLine,
    api.updateCareLine, api.createCustomCareLine ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "municipal_admin" ]));
  mocked(api.listCareTemplates).mockResolvedValue([ PREGNANCY_TEMPLATE, DIABETES_TEMPLATE ]);
  mocked(api.listCityCareLines).mockResolvedValue([ cityLine() ]);
  mocked(api.adoptCareLine).mockResolvedValue(cityLine({ id: "cl3" }));
  mocked(api.updateCareLine).mockResolvedValue(cityLine());
  mocked(api.stepUpMfa).mockResolvedValue(undefined);
});

const templatesTable = () => screen.getByRole("region", { name: "Catálogo-base" });
const linesTable = () => screen.getByRole("region", { name: "Linhas da cidade" });

describe("CareLinesAdmin", () => {
  it("lista o catálogo e as linhas da cidade; adotada mostra a versão e a origem", async () => {
    renderWithProviders(<CareLinesAdmin />);
    expect(await screen.findByText("Gestante e puérpera")).not.toBeNull();
    expect(await within(linesTable()).findByText("catálogo C4 · versão 1")).not.toBeNull();
    expect(within(templatesTable()).getByText("não adotada")).not.toBeNull();
    expect(within(templatesTable()).getByText("adotada · v1")).not.toBeNull();
    expect(within(linesTable()).getByText("diabetes")).not.toBeNull();
    expect(within(linesTable()).getByText("ativa")).not.toBeNull();
  });

  it("adota com step-up: manda a chave e a versão do catálogo e relê as linhas", async () => {
    renderWithProviders(<CareLinesAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Adotar" }));
    const confirm = screen.getByRole("region", { name: "Adotar a linha Gestante e puérpera" });
    expect(confirm.textContent).toMatch(/Nenhuma pessoa é inscrita sem um profissional confirmar/);
    fireEvent.click(within(confirm).getByRole("button", { name: "Adotar linha" }));
    await waitFor(() => expect(api.adoptCareLine).toHaveBeenCalledWith("pregnancy_c3", 1));
    expect((await screen.findByRole("status")).textContent).toBe("Linha Gestante e puérpera adotada.");
    await waitFor(() => expect(api.listCityCareLines).toHaveBeenCalledTimes(2));
  });

  it("versão nova: Adotar versão 2; com a janela fechada pede o código antes", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "municipal_admin" ], { mfa_verified_at: null }));
    renderWithProviders(<CareLinesAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Adotar versão 2" }));
    const confirm = screen.getByRole("region", { name: "Adotar a versão 2 de Pessoa com diabetes" });
    expect(confirm.textContent).toMatch(/passam a ser avaliadas pelas boas práticas da versão nova/);
    fireEvent.change(within(confirm).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(confirm).getByRole("button", { name: "Adotar versão" }));
    await waitFor(() => expect(api.stepUpMfa).toHaveBeenCalledWith("123456"));
    await waitFor(() => expect(api.updateCareLine).toHaveBeenCalledWith("cl4", { template_version: 2 }));
  });

  it("desativar diz o que acontece com as inscrições; 422 vira frase e nada muda", async () => {
    mocked(api.updateCareLine).mockRejectedValueOnce(new api.ApiError(422, { error: "line_inactive" }, "x"));
    renderWithProviders(<CareLinesAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Desativar" }));
    const confirm = screen.getByRole("region", { name: "Desativar Pessoa com diabetes" });
    expect(confirm.textContent).toMatch(/as inscrições ficam guardadas/);
    fireEvent.click(within(confirm).getByRole("button", { name: "Desativar linha" }));
    expect((await within(confirm).findByRole("alert")).textContent).toBe("esta linha foi desativada pela administração");
    expect(api.updateCareLine).toHaveBeenCalledWith("cl4", { active: false });
    expect(api.listCityCareLines).toHaveBeenCalledTimes(1);
  });

  it("duplicate_template: a frase da recusa", async () => {
    mocked(api.listCityCareLines).mockResolvedValue([]);
    mocked(api.adoptCareLine).mockRejectedValueOnce(new api.ApiError(422, { error: "duplicate_template" }, "x"));
    renderWithProviders(<CareLinesAdmin />);
    expect(await screen.findByText("nenhuma linha adotada ainda")).not.toBeNull();
    fireEvent.click((await screen.findAllByRole("button", { name: "Adotar" }))[1]);
    fireEvent.click(screen.getByRole("button", { name: "Adotar linha" }));
    expect((await screen.findByRole("alert")).textContent).toBe("esta linha do catálogo já foi adotada pela cidade");
  });

  it("profissional não vê a tela; sem o interruptor, nada é lido", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "health_professional" ]));
    renderWithProviders(<CareLinesAdmin />);
    expect(await screen.findByText(CARE_NO_ROLE)).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "municipal_admin" ], { features: [ "clinical_record" ] }));
    renderWithProviders(<CareLinesAdmin />);
    expect(await screen.findByText(CARE_DISABLED)).not.toBeNull();
    expect(api.listCareTemplates).not.toHaveBeenCalled();
    expect(api.listCityCareLines).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/CareLinesAdmin.test.tsx`
Expected: FAIL — `./CareLinesAdmin` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/CareLinesAdmin.tsx
// Linhas da cidade (contrato §3): o municipal_admin adota as linhas do
// catálogo-base (C2–C7), adota a versão nova de uma linha já adotada,
// ativa/desativa e cria ou edita linha própria (Task 5). Toda escrita com
// step-up. Versão nova do catálogo não muda as inscrições até a cidade adotá-la
// (spec §4): adotar a versão é o que muda a avaliação. Nada aqui inscreve ninguém.
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { adoptCareLine, errorCode, listCareTemplates, listCityCareLines, updateCareLine, type CareLine, type CareLineTemplate } from "../lib/api";
import { LINES_KEY, STATIC_QUERY, TEMPLATES_KEY, careLineError, conditionLabel, newerVersion, originText } from "../lib/careLines";
import { Panel } from "../components/Panel";
import { DataTable } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { EmptyState } from "../components/EmptyState";
import { SensitiveAction } from "../components/SensitiveAction";
import { buttonStyle, secondaryButtonStyle } from "../components/formStyles";
import { alert, muted, row, stack } from "./exams/styles";
import { CareLinesGate } from "./careLines/CareLinesGate";

type Pending =
  | { kind: "adopt"; template: CareLineTemplate }
  | { kind: "upgrade"; line: CareLine; version: number }
  | { kind: "toggle"; line: CareLine }
  | null;

export function CareLinesAdmin({ onGoToSecurity }: { onGoToSecurity?(): void } = {}) {
  return (
    <CareLinesGate title="Linhas da cidade" sub="catálogo-base (C2–C7) e linhas próprias · administração municipal" allow={(r) => r.admin}>
      {() => <AdminBody onGoToSecurity={onGoToSecurity} />}
    </CareLinesGate>
  );
}

// O que cada confirmação diz e faz (spec 6a §3.5: dizer antes do Confirmar).
function confirmation(p: Exclude<Pending, null>): { title: string; description: string; confirmLabel: string; run(): Promise<unknown>; done: string } {
  if (p.kind === "adopt") return {
    title: `Adotar a linha ${p.template.name}`,
    description: "A linha entra no painel das equipes e nas sugestões a partir de agora. Nenhuma pessoa é inscrita sem um profissional confirmar.",
    confirmLabel: "Adotar linha", run: () => adoptCareLine(p.template.key, p.template.version), done: `Linha ${p.template.name} adotada.`
  };
  if (p.kind === "upgrade") return {
    title: `Adotar a versão ${p.version} de ${p.line.name}`,
    description: "As inscrições desta linha passam a ser avaliadas pelas boas práticas da versão nova. Ninguém é inscrito nem encerrado por isso.",
    confirmLabel: "Adotar versão", run: () => updateCareLine(p.line.id, { template_version: p.version }),
    done: `${p.line.name} agora usa a versão ${p.version}.`
  };
  return p.line.active ? {
    title: `Desativar ${p.line.name}`,
    description: "A linha sai do painel e das sugestões novas; as inscrições ficam guardadas e voltam se a linha for reativada.",
    confirmLabel: "Desativar linha", run: () => updateCareLine(p.line.id, { active: false }), done: `${p.line.name} desativada.`
  } : {
    title: `Ativar ${p.line.name}`, description: "A linha volta ao painel das equipes e às sugestões.",
    confirmLabel: "Ativar linha", run: () => updateCareLine(p.line.id, { active: true }), done: `${p.line.name} ativada.`
  };
}

function AdminBody({ onGoToSecurity }: { onGoToSecurity?(): void }) {
  const queryClient = useQueryClient();
  const templates = useQuery({ queryKey: [ TEMPLATES_KEY ], queryFn: listCareTemplates, ...STATIC_QUERY });
  const lines = useQuery({ queryKey: [ LINES_KEY ], queryFn: listCityCareLines, ...STATIC_QUERY });
  const [ pending, setPending ] = useState<Pending>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const cityLines = lines.data ?? [];
  const adopted = new Map(cityLines.filter((l) => l.template_key).map((l) => [ l.template_key as string, l ]));
  const start = (next: Pending) => { setNotice(null); setPending(next); };
  const confirm = pending ? confirmation(pending) : null;

  return (
    <div style={stack}>
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      <Panel title="Catálogo-base" sub="as seis linhas das notas metodológicas do componente de qualidade da APS">
        <div style={{ ...stack, padding: 16 }}>
          {templates.isPending && <p className="mono" style={muted}>carregando…</p>}
          {templates.isError && <p role="alert" style={alert}>{careLineError(templates.error)}</p>}
          {templates.data && (
            <DataTable<CareLineTemplate>
              cols={[
                { label: "Indicador", w: "90px", render: (t) => t.indicator },
                { label: "Linha", w: "2fr", card: "title", render: (t) => t.name },
                { label: "Versão", w: "80px", render: (t) => `v${t.version}` },
                { label: "Na cidade", w: "1.2fr", render: (t) => { const l = adopted.get(t.key); return l ? `adotada · v${l.template_version ?? "—"}` : "não adotada"; } },
                {
                  label: "Ação", w: "1.2fr", card: "actions", render: (t) => {
                    const l = adopted.get(t.key);
                    if (!l) return <button type="button" style={buttonStyle} onClick={() => start({ kind: "adopt", template: t })}>Adotar</button>;
                    const newer = newerVersion(l, templates.data ?? []);
                    return newer === null ? null
                      : <button type="button" style={secondaryButtonStyle} onClick={() => start({ kind: "upgrade", line: l, version: newer })}>Adotar versão {newer}</button>;
                  }
                }
              ]}
              rows={templates.data} rowKey={(t) => t.key} empty="o catálogo-base está vazio"
            />
          )}
        </div>
      </Panel>
      <Panel title="Linhas da cidade" sub="as que o painel das equipes e as sugestões usam">
        <div style={{ ...stack, padding: 16 }}>
          {lines.isPending && <p className="mono" style={muted}>carregando…</p>}
          {lines.isError && <p role="alert" style={alert}>{careLineError(lines.error)}</p>}
          {lines.data && (cityLines.length === 0 ? <EmptyState title="nenhuma linha adotada ainda" /> : (
            <DataTable<CareLine>
              cols={[
                { label: "Linha", w: "2fr", card: "title", render: (l) => l.name },
                { label: "Origem", w: "1.4fr", render: (l) => originText(l) },
                { label: "Condição declarada", w: "1.1fr", render: (l) => (l.declared_condition ? conditionLabel(l.declared_condition) : "—") },
                { label: "Situação", w: "100px", render: (l) => <Tag tone={l.active ? "ok" : "neutral"}>{l.active ? "ativa" : "inativa"}</Tag> },
                {
                  label: "Ação", w: "1.4fr", card: "actions", render: (l) => (
                    <div style={row}>
                      <button type="button" style={secondaryButtonStyle} onClick={() => start({ kind: "toggle", line: l })}>
                        {l.active ? "Desativar" : "Ativar"}
                      </button>
                    </div>
                  )
                }
              ]}
              rows={cityLines} rowKey={(l) => l.id}
            />
          ))}
        </div>
      </Panel>
      {pending && confirm && (
        <SensitiveAction
          key={confirm.title}
          title={confirm.title}
          description={confirm.description}
          requiresStepUp
          confirmLabel={confirm.confirmLabel}
          run={async () => { await confirm.run(); }}
          onDone={() => {
            setPending(null); setNotice(confirm.done);
            void queryClient.invalidateQueries({ queryKey: [ LINES_KEY ] });
          }}
          onCancel={() => setPending(null)}
          onGoToSecurity={onGoToSecurity}
          translateError={(err) => (errorCode(err) ? careLineError(err) : null)}
        />
      )}
    </div>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

Em `src/App.tsx`, depois de `import { BillingSettings } from "./modules/BillingSettings";`, acrescente

```ts
import { CareLinesAdmin } from "./modules/CareLinesAdmin";
```

e, depois de `    case "billing-settings": return <BillingSettings onGoToSecurity={() => setActive("security")} />;`, acrescente

```tsx
    case "care-lines-admin": return <CareLinesAdmin onGoToSecurity={() => setActive("security")} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/CareLinesAdmin.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/CareLinesAdmin.tsx src/modules/CareLinesAdmin.test.tsx src/App.tsx
$G commit -m "feat: let the city admin adopt care lines and their new versions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Linha própria — criar e editar

**Files:**
- Create: `src/modules/careLines/CustomLineForm.tsx`
- Modify: `src/modules/CareLinesAdmin.tsx`
- Test: `src/modules/careLines/CustomLineForm.test.tsx`

**Interfaces:**
- Consumes: `createCustomCareLine`, `updateCareLine`, `errorCode`, `CareLine` (Task 1); `CONDITIONS`, `CUSTOM_ANCHORS`, `SOURCE_KINDS`, `CustomLineDraft`, `PracticeDraft`, `careLineError`, `customLineBody`, `customLineProblem`, `draftOfLine`, `emptyCustomLine`, `emptyPractice` (Task 2); `SensitiveAction`.
- Produces: `CustomLineForm({ line, onSaved(message), onCancel, onGoToSecurity? })` — `line: CareLine | null` (null = nova).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/careLines/CustomLineForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), createCustomCareLine: vi.fn(), updateCareLine: vi.fn(),
    listCareTemplates: vi.fn(), listCityCareLines: vi.fn(), adoptCareLine: vi.fn() };
});

import * as api from "../../lib/api";
import { CustomLineForm } from "./CustomLineForm";
import { CareLinesAdmin } from "../CareLinesAdmin";
import { customLineBody, draftOfLine, emptyCustomLine } from "../../lib/careLines";
import { renderWithProviders } from "../../test/campaignFixtures";
import { careUser, cityLine } from "../../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.stepUpMfa, api.createCustomCareLine, api.updateCareLine, api.listCareTemplates,
    api.listCityCareLines, api.adoptCareLine ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "municipal_admin" ]));
  mocked(api.createCustomCareLine).mockResolvedValue(cityLine({ id: "cl9" }));
  mocked(api.updateCareLine).mockResolvedValue(cityLine({ id: "cl9" }));
  mocked(api.listCareTemplates).mockResolvedValue([]);
  mocked(api.listCityCareLines).mockResolvedValue([]);
});

const form = () => screen.getByRole("region", { name: "Linha própria" });
const set = (label: string, value: string, scope: HTMLElement = form()) =>
  fireEvent.change(within(scope).getByLabelText(label), { target: { value } });

describe("CustomLineForm", () => {
  it("cria com step-up: idade em meses, SIGTAP na fonte e CBO só na consulta", async () => {
    const onSaved = vi.fn();
    renderWithProviders(<CustomLineForm line={null} onSaved={onSaved} onCancel={vi.fn()} />);
    set("Nome da linha", "Saúde mental");
    set("Condição declarada que sugere esta linha", "mental_health");
    set("Códigos CIAP-2", "P76, P74");
    set("Códigos CID-10", "F32");
    set("Idade mínima (anos)", "18");
    const first = within(form()).getByRole("group", { name: "Boa prática 1" });
    set("O que deve ser feito", "Consulta de acompanhamento", first);
    set("Janela (dias)", "90", first);
    fireEvent.click(within(form()).getByRole("button", { name: "Incluir boa prática" }));
    const second = within(form()).getByRole("group", { name: "Boa prática 2" });
    set("O que deve ser feito", "Matriciamento", second);
    set("Fonte", "procedure", second);
    set("Código SIGTAP", "0301080232", second);
    fireEvent.click(within(form()).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar linha" }));
    await waitFor(() => expect(api.createCustomCareLine).toHaveBeenCalledWith({
      name: "Saúde mental", declared_condition: "mental_health",
      eligibility: { problems: { ciap2: [ "P76", "P74" ], cid10: [ "F32" ] }, age_min_years: 18, age_max_years: null, sex: null,
        declared: [ "mental_health" ], requires_anchor: null },
      practices: [
        { key: "consulta_de_acompanhamento", label: "Consulta de acompanhamento", source: "consultation", window_days: 90, min_count: 1,
          schedule: "end_of_window", anchor: "rolling", cbo_filter: [ "2231", "2251", "2252", "2253", "2235" ] },
        { key: "matriciamento", label: "Matriciamento", source: "procedure:0301080232", window_days: 180, min_count: 1,
          schedule: "end_of_window", anchor: "rolling", cbo_filter: null }
      ]
    }));
    expect(onSaved).toHaveBeenCalledWith("Linha Saúde mental criada.");
  });

  it("recusa antes de enviar e mantém o digitado: código inválido e SIGTAP curto", () => {
    renderWithProviders(<CustomLineForm line={null} onSaved={vi.fn()} onCancel={vi.fn()} />);
    set("Nome da linha", "Saúde mental");
    set("Códigos CID-10", "F3");
    fireEvent.click(within(form()).getByRole("button", { name: "Revisar e salvar" }));
    expect(within(form()).getByRole("alert").textContent).toBe("código CID-10 inválido: F3");
    expect((within(form()).getByLabelText("Códigos CID-10") as HTMLInputElement).value).toBe("F3");
    set("Códigos CID-10", "F32");
    const first = within(form()).getByRole("group", { name: "Boa prática 1" });
    set("O que deve ser feito", "Exame", first);
    set("Fonte", "exam", first);
    set("Código SIGTAP", "02020105", first);
    fireEvent.click(within(form()).getByRole("button", { name: "Revisar e salvar" }));
    expect(within(form()).getByRole("alert").textContent).toBe("boa prática 1: código SIGTAP com 10 dígitos");
    expect(screen.queryByRole("button", { name: "Salvar linha" })).toBeNull();
    expect(api.createCustomCareLine).not.toHaveBeenCalled();
  });

  it("consulta mostra a caixa de médico ou enfermeiro; outra fonte não", () => {
    renderWithProviders(<CustomLineForm line={null} onSaved={vi.fn()} onCancel={vi.fn()} />);
    const first = within(form()).getByRole("group", { name: "Boa prática 1" });
    expect(within(first).getByLabelText("Só consulta de médico ou enfermeiro conta")).not.toBeNull();
    set("Fonte", "blood_pressure", first);
    expect(within(first).queryByLabelText("Só consulta de médico ou enfermeiro conta")).toBeNull();
    expect(within(first).queryByLabelText("Código SIGTAP")).toBeNull();
  });

  it("editar: preenche, guarda as chaves e manda declared_condition null ao tirar a condição", async () => {
    const line = cityLine({ id: "cl9", source: "custom", template_key: null, template_version: null, indicator: null,
      ...customLineBody({ ...draftOfLine(cityLine()), name: "Saúde mental" }), declared_condition: "mental_health" });
    renderWithProviders(<CustomLineForm line={line} onSaved={vi.fn()} onCancel={vi.fn()} />);
    expect((within(form()).getByLabelText("Nome da linha") as HTMLInputElement).value).toBe("Saúde mental");
    set("Condição declarada que sugere esta linha", "");
    fireEvent.click(within(form()).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar linha" }));
    await waitFor(() => expect(api.updateCareLine).toHaveBeenCalled());
    const [ id, patch ] = mocked(api.updateCareLine).mock.calls[0];
    expect(id).toBe("cl9");
    expect(patch.declared_condition).toBeNull();
    expect(patch.practices.map((p: { key: string }) => p.key)).toEqual([ "c4_a", "c4_e" ]);
    expect(patch.eligibility.declared).toEqual([]);
  });

  it("422 invalid_practices do api vira frase", async () => {
    mocked(api.createCustomCareLine).mockRejectedValueOnce(new api.ApiError(422, { error: "invalid_practices" }, "x"));
    renderWithProviders(<CustomLineForm line={null} onSaved={vi.fn()} onCancel={vi.fn()} />);
    set("Nome da linha", "Saúde mental");
    set("O que deve ser feito", "Consulta", within(form()).getByRole("group", { name: "Boa prática 1" }));
    fireEvent.click(within(form()).getByRole("button", { name: "Revisar e salvar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar linha" }));
    expect((await screen.findByRole("alert")).textContent).toBe("boas práticas inválidas — confira os campos");
  });

  it("na tela do admin: Nova linha própria abre o formulário; Editar só nas próprias", async () => {
    mocked(api.listCityCareLines).mockResolvedValue([ cityLine(), cityLine({ id: "cl9", name: "Saúde mental", source: "custom",
      template_key: null, template_version: null, indicator: null }) ]);
    renderWithProviders(<CareLinesAdmin />);
    const lines = await screen.findByRole("region", { name: "Linhas da cidade" });
    expect(await within(lines).findByText("Saúde mental")).not.toBeNull();
    expect(within(lines).getAllByRole("button", { name: "Editar" })).toHaveLength(1);
    fireEvent.click(within(lines).getByRole("button", { name: "Nova linha própria" }));
    expect((within(form()).getByLabelText("Nome da linha") as HTMLInputElement).value).toBe("");
    expect(emptyCustomLine().practices).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/careLines/CustomLineForm.test.tsx`
Expected: FAIL — `./CustomLineForm` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/CustomLineForm.tsx
// Linha própria da cidade (contrato §3; spec, decisões do coordenador): a
// mesma forma das linhas do catálogo — elegibilidade por CIAP-2/CID-10 ativos,
// idade e sexo; boas práticas por fonte, janela, quantidade, cronograma e
// âncora — e a condição declarada que a sugere (ex.: saúde mental). Salvar
// passa pelo step-up. Formato é conferido aqui (recusar, nunca cortar) e de
// novo no api (invalid_practices, invalid_eligibility).
import { useState, type CSSProperties } from "react";
import { createCustomCareLine, errorCode, updateCareLine, type CareLine } from "../../lib/api";
import {
  CONDITIONS, CUSTOM_ANCHORS, SOURCE_KINDS, careLineError, customLineBody, customLineProblem, draftOfLine, emptyCustomLine, emptyPractice,
  type CustomLineDraft, type PracticeDraft
} from "../../lib/careLines";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, checkLabel, fieldLabel, fieldset, form, legend, row } from "../exams/styles";

export function CustomLineForm({ line, onSaved, onCancel, onGoToSecurity }: {
  line: CareLine | null; onSaved(message: string): void; onCancel(): void; onGoToSecurity?(): void;
}) {
  const [ draft, setDraft ] = useState<CustomLineDraft>(() => (line ? draftOfLine(line) : emptyCustomLine()));
  const [ error, setError ] = useState<string | null>(null);
  const [ confirming, setConfirming ] = useState(false);
  const set = (patch: Partial<CustomLineDraft>) => { setError(null); setDraft((d) => ({ ...d, ...patch })); };
  const setPractice = (i: number, patch: Partial<PracticeDraft>) =>
    set({ practices: draft.practices.map((p, j) => (j === i ? { ...p, ...patch } : p)) });

  function review() {
    const problem = customLineProblem(draft);
    if (problem) { setError(problem); return; }
    setConfirming(true);
  }

  async function save() {
    const body = customLineBody(draft);
    if (line) await updateCareLine(line.id, { ...body, declared_condition: draft.declaredCondition || null });
    else await createCustomCareLine(body);
  }

  const text = (label: string, value: string, onChange: (v: string) => void, hint?: string) => (
    <div style={{ display: "flex", flexDirection: "column", gap: 2 }}>
      <label style={fieldLabel}>
        {label}
        <input value={value} style={inputStyle} onChange={(e) => onChange(e.target.value)} />
      </label>
      {hint && <span style={hintStyle}>{hint}</span>}
    </div>
  );

  if (confirming) {
    return (
      <SensitiveAction
        title={line ? `Salvar ${draft.name.trim()}` : `Criar a linha ${draft.name.trim()}`}
        description={`${draft.practices.length} boa(s) prática(s). ${line ? "As inscrições desta linha passam a ser avaliadas pela forma nova." : "A linha entra no painel das equipes; ninguém é inscrito sem um profissional confirmar."}`}
        requiresStepUp
        confirmLabel="Salvar linha"
        run={save}
        onDone={() => onSaved(line ? `Linha ${draft.name.trim()} salva.` : `Linha ${draft.name.trim()} criada.`)}
        onCancel={() => setConfirming(false)}
        onGoToSecurity={onGoToSecurity}
        translateError={(err) => (errorCode(err) ? careLineError(err) : null)}
      />
    );
  }

  return (
    <section aria-label="Linha própria" style={box}>
      <div style={form}>
        {text("Nome da linha", draft.name, (v) => set({ name: v }), "até 120 caracteres")}
        <label style={fieldLabel}>
          Condição declarada que sugere esta linha
          <select value={draft.declaredCondition} style={inputStyle}
            onChange={(e) => set({ declaredCondition: e.target.value as CustomLineDraft["declaredCondition"] })}>
            <option value="">nenhuma</option>
            {CONDITIONS.map((c) => <option key={c.key} value={c.key}>{c.label}</option>)}
          </select>
        </label>
        <fieldset style={fieldset}>
          <legend style={legend}>Quem é sugerido (problema ativo, idade e sexo)</legend>
          {text("Códigos CIAP-2", draft.ciap, (v) => set({ ciap: v }), "separados por vírgula, ex.: P76, P74")}
          {text("Códigos CID-10", draft.cid10, (v) => set({ cid10: v }), "separados por vírgula, ex.: F32, F41")}
          <div style={row}>
            {text("Idade mínima (anos)", draft.ageMinYears, (v) => set({ ageMinYears: v }), "vazio = sem limite")}
            {text("Idade máxima (anos)", draft.ageMaxYears, (v) => set({ ageMaxYears: v }), "vazio = sem limite")}
            <label style={fieldLabel}>
              Sexo
              <select value={draft.sex} style={inputStyle} onChange={(e) => set({ sex: e.target.value as CustomLineDraft["sex"] })}>
                <option value="">qualquer</option>
                <option value="female">feminino</option>
                <option value="male">masculino</option>
              </select>
            </label>
          </div>
        </fieldset>
        {draft.practices.map((p, i) => (
          <fieldset key={i} role="group" aria-label={`Boa prática ${i + 1}`} style={fieldset}>
            <legend style={legend}>Boa prática {i + 1}</legend>
            {text("O que deve ser feito", p.label, (v) => setPractice(i, { label: v }), "até 120 caracteres, ex.: consulta de acompanhamento")}
            {text("Como o cidadão lê (opcional)", p.citizenLabel, (v) => setPractice(i, { citizenLabel: v }),
              "até 80 caracteres, em linguagem simples, ex.: consulta com a equipe — aparece em Meu acompanhamento")}
            <div style={row}>
              <label style={fieldLabel}>
                Fonte
                <select value={p.sourceKind} style={inputStyle} onChange={(e) => setPractice(i, { sourceKind: e.target.value })}>
                  {SOURCE_KINDS.map((s) => <option key={s.key} value={s.key}>{s.label}</option>)}
                </select>
              </label>
              {(p.sourceKind === "exam" || p.sourceKind === "procedure") &&
                text("Código SIGTAP", p.sigtap, (v) => setPractice(i, { sigtap: v }), "10 dígitos")}
              {text("Janela (dias)", p.windowDays, (v) => setPractice(i, { windowDays: v }))}
              {text("Quantidade na janela", p.minCount, (v) => setPractice(i, { minCount: v }))}
            </div>
            <div style={row}>
              <label style={fieldLabel}>
                Contar a partir de
                <select value={p.anchor} style={inputStyle} onChange={(e) => setPractice(i, { anchor: e.target.value })}>
                  {CUSTOM_ANCHORS.map((a) => <option key={a.key} value={a.key}>{a.label}</option>)}
                </select>
              </label>
              <label style={fieldLabel}>
                Cronograma
                <select value={p.schedule} style={inputStyle} onChange={(e) => setPractice(i, { schedule: e.target.value })}>
                  <option value="end_of_window">até o fim da janela</option>
                  <option value="proportional">proporcional ao tempo</option>
                </select>
              </label>
            </div>
            {p.sourceKind === "consultation" && (
              <label style={checkLabel}>
                <input type="checkbox" checked={p.doctorOrNurse} onChange={(e) => setPractice(i, { doctorOrNurse: e.target.checked })} />
                Só consulta de médico ou enfermeiro conta
              </label>
            )}
            {draft.practices.length > 1 && (
              <div>
                <button type="button" style={secondaryButtonStyle}
                  onClick={() => set({ practices: draft.practices.filter((_, j) => j !== i) })}>Tirar boa prática {i + 1}</button>
              </div>
            )}
          </fieldset>
        ))}
        <div>
          <button type="button" style={secondaryButtonStyle} onClick={() => set({ practices: [ ...draft.practices, emptyPractice() ] })}>
            Incluir boa prática
          </button>
        </div>
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <button type="button" style={buttonStyle} onClick={review}>Revisar e salvar</button>
          <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}

const hintStyle: CSSProperties = { fontSize: 11.5, color: "var(--ink3)" };
```

Em `src/modules/CareLinesAdmin.tsx`:

1. Troque o import

```ts
import { CareLinesGate } from "./careLines/CareLinesGate";
```

por

```ts
import { CareLinesGate } from "./careLines/CareLinesGate";
import { CustomLineForm } from "./careLines/CustomLineForm";
```

2. Em `AdminBody`, depois de `  const [ notice, setNotice ] = useState<string | null>(null);`, acrescente

```ts
  // Linha própria em edição: "new" para criar, a linha para editar.
  const [ editing, setEditing ] = useState<CareLine | "new" | null>(null);
```

3. Troque

```tsx
      <Panel title="Linhas da cidade" sub="as que o painel das equipes e as sugestões usam">
```

por

```tsx
      <Panel title="Linhas da cidade" sub="as que o painel das equipes e as sugestões usam"
        right={<button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setPending(null); setEditing("new"); }}>Nova linha própria</button>}>
```

4. Na coluna "Ação" das linhas da cidade, troque

```tsx
                      <button type="button" style={secondaryButtonStyle} onClick={() => start({ kind: "toggle", line: l })}>
                        {l.active ? "Desativar" : "Ativar"}
                      </button>
```

por

```tsx
                      <button type="button" style={secondaryButtonStyle} onClick={() => start({ kind: "toggle", line: l })}>
                        {l.active ? "Desativar" : "Ativar"}
                      </button>
                      {l.source === "custom" && (
                        <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setPending(null); setEditing(l); }}>
                          Editar
                        </button>
                      )}
```

5. Antes de `      {pending && confirm && (`, acrescente

```tsx
      {editing && (
        <CustomLineForm
          key={editing === "new" ? "new" : editing.id}
          line={editing === "new" ? null : editing}
          onSaved={(message) => {
            setEditing(null); setNotice(message);
            void queryClient.invalidateQueries({ queryKey: [ LINES_KEY ] });
          }}
          onCancel={() => setEditing(null)}
          onGoToSecurity={onGoToSecurity}
        />
      )}
```

O `Panel` já aceita `right` (slot do cabeçalho).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/careLines/CustomLineForm.test.tsx src/modules/CareLinesAdmin.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/CustomLineForm.tsx src/modules/careLines/CustomLineForm.test.tsx src/modules/CareLinesAdmin.tsx
$G commit -m "feat: let the city admin create and edit its own care lines

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 6: Painel nominal da equipe — busca ativa, filtros e contadores

**Files:**
- Create: `src/modules/careLines/EnrollmentList.tsx`, `src/modules/CareLinesTeam.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/CareLinesTeam.test.tsx`

**Interfaces:**
- Consumes: `getCareContext`, `listCareEnrollments`, `countOverdueEnrollments`, `CareEnrollment`, `CarePracticeStatus` (Task 1); `CONTEXT_KEY`, `ENROLLMENTS_KEY`, `OVERDUE_COUNT_KEY`, `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `ENROLLMENT_FILTERS`, `careLineError`, `isStaleCare`, `channelLabel`, `outcomeLabel`, `fmtDay`, `personName`, `personStatus`, `practiceText`, `statusLabel`, `statusTone` (Task 2); `CareLinesGate` (Task 3); `todayInCity` (`src/lib/campaigns.ts`).
- Produces:
  - `EnrollmentList({ rows, onOpen? })` e `PracticeList({ practices })` em `src/modules/careLines/EnrollmentList.tsx`;
  - `CareLinesTeam({ today? })` em `src/modules/CareLinesTeam.tsx`, com `TeamBody` e o tipo `TeamTab = "overdue" | "due_soon" | "all"` que as Tasks 7, 9 e 10 estendem (ver cada uma).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/CareLinesTeam.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { focusManager, onlineManager } from "@tanstack/react-query";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), getCareContext: vi.fn(), listCareEnrollments: vi.fn(), countOverdueEnrollments: vi.fn(),
    countCareSuggestions: vi.fn(), listCareSuggestions: vi.fn(), getCareEnrollment: vi.fn()
  };
});

import * as api from "../lib/api";
import { CareLinesTeam } from "./CareLinesTeam";
import { CARE_NO_ROLE, UNTRACKED_TEXT } from "../lib/careLines";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY23, careContext, careUser, enrollment } from "../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.useRealTimers(); });
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getCareContext, api.listCareEnrollments, api.countOverdueEnrollments,
    api.countCareSuggestions, api.listCareSuggestions, api.getCareEnrollment ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser());
  mocked(api.getCareContext).mockResolvedValue(careContext());
  mocked(api.listCareEnrollments).mockResolvedValue({ enrollments: [ enrollment() ], next_before: null });
  mocked(api.countOverdueEnrollments).mockResolvedValue(12);
  mocked(api.countCareSuggestions).mockResolvedValue(3);
  mocked(api.listCareSuggestions).mockResolvedValue({ suggestions: [], next_before: null });
});

const NAME = "Maria das Dores · 58 anos";
const itemOf = (label: string) => screen.getByText(label, { selector: "strong" }).closest("li") as HTMLElement;

describe("CareLinesTeam — painel nominal", () => {
  it("abre na busca ativa da primeira equipe; situação da pessoa e de cada boa prática", async () => {
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    expect(await screen.findByText(NAME)).not.toBeNull();
    expect(api.listCareEnrollments).toHaveBeenCalledTimes(1);
    expect(api.listCareEnrollments).toHaveBeenCalledWith({ teamId: "tm1", careLineId: undefined, status: "overdue", before: null });
    expect(itemOf("Hemoglobina glicada").textContent).toBe("Hemoglobina glicada: vencida · última em 20/08/2025 · venceu em 20/08/2026");
    expect(itemOf("Peso e altura").textContent).toBe("Peso e altura: vence em até 30 dias · última em 12/11/2025 · vence em 12/11/2026");
    expect(screen.getByText("vencida")).not.toBeNull();
    expect(screen.getByText("nenhum")).not.toBeNull();
  });

  it("visita do ACS aparece como não acompanhada, em tom neutro", async () => {
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    await screen.findByText(NAME);
    const acs = itemOf("Visita do ACS");
    expect(acs.textContent).toBe(`Visita do ACS: ${UNTRACKED_TEXT}`);
    expect((acs.querySelector("span") as HTMLElement).style.color).toBe("var(--ink3)");
    expect(acs.textContent).not.toMatch(/vencida/);
  });

  it("troca de equipe, linha e aba relê com os filtros; contador na aba", async () => {
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    await screen.findByText(NAME);
    expect(await screen.findByRole("button", { name: "Busca ativa (12)" })).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Equipe"), { target: { value: "tm2" } });
    await waitFor(() => expect(api.listCareEnrollments).toHaveBeenLastCalledWith({ teamId: "tm2", careLineId: undefined, status: "overdue", before: null }));
    await waitFor(() => expect(api.countOverdueEnrollments).toHaveBeenCalledWith("tm2"));
    fireEvent.change(screen.getByLabelText("Linha"), { target: { value: "cl3" } });
    await waitFor(() => expect(api.listCareEnrollments).toHaveBeenLastCalledWith({ teamId: "tm2", careLineId: "cl3", status: "overdue", before: null }));
    fireEvent.click(screen.getByRole("button", { name: "Todos" }));
    await waitFor(() => expect(api.listCareEnrollments).toHaveBeenLastCalledWith({ teamId: "tm2", careLineId: "cl3", status: "all", before: null }));
    expect(screen.getByRole("button", { name: "Todos" }).getAttribute("aria-pressed")).toBe("true");
  });

  it("volta à aba e passa o tempo: a lista não relê; os contadores usam COUNT_QUERY", async () => {
    vi.useFakeTimers({ shouldAdvanceTime: true });
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    await screen.findByText(NAME);
    await act(async () => {
      focusManager.setFocused(false); focusManager.setFocused(true);
      onlineManager.setOnline(false); onlineManager.setOnline(true);
    });
    await act(async () => { await vi.advanceTimersByTimeAsync(125_000); });
    expect(api.listCareEnrollments).toHaveBeenCalledTimes(1);
    expect(mocked(api.countOverdueEnrollments).mock.calls.length).toBeGreaterThanOrEqual(3);
  });

  it("Carregar mais pede a próxima página pelo cursor e não relê as anteriores", async () => {
    mocked(api.listCareEnrollments)
      .mockResolvedValueOnce({ enrollments: [ enrollment() ], next_before: "c2" })
      .mockResolvedValueOnce({ enrollments: [ enrollment({ id: "en2", person: { patient_id: "pa9", citizen_id: "ci9", display_name: "Rosa Lima", age: 71 } }) ], next_before: null });
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    await screen.findByText(NAME);
    fireEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
    expect(await screen.findByText("Rosa Lima · 71 anos")).not.toBeNull();
    expect(api.listCareEnrollments).toHaveBeenLastCalledWith({ teamId: "tm1", careLineId: undefined, status: "overdue", before: "c2" });
    expect(api.listCareEnrollments).toHaveBeenCalledTimes(2);
    expect(screen.queryByRole("button", { name: "Carregar mais" })).toBeNull();
  });

  it("página curta com next_before (L6): diz que há mais e só carrega no clique, sem laço", async () => {
    mocked(api.listCareEnrollments)
      .mockResolvedValueOnce({ enrollments: [], next_before: "c5" })
      .mockResolvedValueOnce({ enrollments: [ enrollment() ], next_before: null });
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    expect(await screen.findByText("Nada nesta parte da lista; carregue mais para continuar a busca.")).not.toBeNull();
    expect(screen.queryByText("ninguém nesta lista")).toBeNull();
    expect(api.listCareEnrollments).toHaveBeenCalledTimes(1);
    fireEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
    expect(await screen.findByText(NAME)).not.toBeNull();
    expect(api.listCareEnrollments).toHaveBeenLastCalledWith({ teamId: "tm1", careLineId: undefined, status: "overdue", before: "c5" });
  });

  it("sem equipe com vínculo: diz isso e não lê lista nem contador", async () => {
    mocked(api.getCareContext).mockResolvedValue(careContext({ teams: [] }));
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    expect(await screen.findByText("você não está em nenhuma equipe com vínculo ativo na unidade")).not.toBeNull();
    expect(api.listCareEnrollments).not.toHaveBeenCalled();
    expect(api.countOverdueEnrollments).not.toHaveBeenCalled();
  });

  it("403 out_of_scope vira frase; admin sem papel de profissional não vê o painel", async () => {
    mocked(api.listCareEnrollments).mockRejectedValue(new api.ApiError(403, { error: "out_of_scope" }, "x"));
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    expect((await screen.findByRole("alert")).textContent).toBe("você não tem vínculo ativo na unidade desta equipe");
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "municipal_admin" ]));
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    expect(await screen.findByText(CARE_NO_ROLE)).not.toBeNull();
  });

  it("lista vazia na busca ativa: ninguém para procurar", async () => {
    mocked(api.listCareEnrollments).mockResolvedValue({ enrollments: [], next_before: null });
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    const list = await screen.findByRole("region", { name: "Pessoas da equipe" });
    expect(await within(list).findByText("ninguém nesta lista")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/CareLinesTeam.test.tsx`
Expected: FAIL — `./CareLinesTeam` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/EnrollmentList.tsx
// Lista nominal das inscrições (contrato §5): pessoa, linha, a pior situação
// (do api; a tela só calcula sem `worst_status`, e `untracked` nunca piora) e
// cada boa prática em frase. No telefone vira cartão (pessoa no título).
import type { CareEnrollment, CarePracticeStatus } from "../../lib/api";
import { channelLabel, fmtDay, outcomeLabel, personName, personStatus, practiceText, statusLabel, statusTone } from "../../lib/careLines";
import { DataTable, type Column } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { list } from "../exams/styles";

const PRACTICE_COLOR: Record<string, string> = { overdue: "var(--down)", untracked: "var(--ink3)" };

export function PracticeList({ practices }: { practices: CarePracticeStatus[] }) {
  return (
    <ul style={list}>
      {practices.map((p) => (
        <li key={p.key}>
          <strong>{p.label}</strong>: <span style={{ color: PRACTICE_COLOR[p.status] ?? "var(--ink)" }}>{practiceText(p)}</span>
        </li>
      ))}
    </ul>
  );
}

export const personLine = (e: Pick<CareEnrollment, "person">) =>
  `${personName(e.person)}${e.person.age !== null ? ` · ${e.person.age} anos` : ""}`;

export function EnrollmentList({ rows, onOpen }: { rows: CareEnrollment[]; onOpen?(e: CareEnrollment): void }) {
  const cols: Column<CareEnrollment>[] = [
    { label: "Pessoa", w: "1.5fr", card: "title", render: personLine },
    { label: "Linha", w: "1.1fr", render: (e) => e.care_line.name },
    { label: "Situação", w: "130px", render: (e) => { const s = personStatus(e); return <Tag tone={statusTone(s)}>{statusLabel(s)}</Tag>; } },
    { label: "Boas práticas", w: "3fr", render: (e) => <PracticeList practices={e.practices} /> },
    {
      label: "Último contato", w: "1.3fr",
      render: (e) => (e.last_contact ? `${channelLabel(e.last_contact.channel)} · ${outcomeLabel(e.last_contact.outcome)} · ${fmtDay(e.last_contact.at)}` : "nenhum")
    }
  ];
  if (onOpen) cols.push({
    label: "Ação", w: "100px", card: "actions",
    render: (e) => <button type="button" style={secondaryButtonStyle} aria-label={`Abrir ${personName(e.person)}`} onClick={() => onOpen(e)}>Abrir</button>
  });
  return <DataTable<CareEnrollment> cols={cols} rows={rows} rowKey={(e) => e.id} empty="ninguém nesta lista" />;
}
```

```tsx
// src/modules/CareLinesTeam.tsx
// Acompanhamento da equipe (contrato §4–§5; spec §6 "Equipe"): o profissional
// com vínculo ativo na unidade da equipe vê, pessoa a pessoa, a situação de
// cada boa prática, filtra a busca ativa (alguma vencida), registra contato,
// agenda, inscreve e encerra; as sugestões ficam na aba própria (Task 9).
// A lista é NOMINAL e GRAVA trilha (clinical_record.viewed, access: care_line):
// lida só ao abrir, ao filtrar, ao trocar de aba ou depois de uma ação —
// nunca sozinha. Só os contadores (/count, sem trilha) releem por minuto.
import { useState, type CSSProperties } from "react";
import { useInfiniteQuery, useQuery, useQueryClient } from "@tanstack/react-query";
import { countOverdueEnrollments, getCareContext, listCareEnrollments } from "../lib/api";
import {
  CONTEXT_KEY, COUNT_QUERY, ENROLLMENTS_KEY, ENROLLMENT_FILTERS, NOMINAL_QUERY, OVERDUE_COUNT_KEY, STATIC_QUERY, careLineError, isStaleCare
} from "../lib/careLines";
import { todayInCity } from "../lib/campaigns";
import { Panel } from "../components/Panel";
import { EmptyState } from "../components/EmptyState";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";
import { alert, fieldLabel, muted, row, stack } from "./exams/styles";
import { CareLinesGate } from "./careLines/CareLinesGate";
import { EnrollmentList } from "./careLines/EnrollmentList";

export function CareLinesTeam({ today = todayInCity() }: { today?: string } = {}) {
  return (
    <CareLinesGate title="Acompanhamento da equipe" sub="linhas de cuidado · busca ativa, sugestões e inscrições" allow={(r) => r.professional}>
      {() => <TeamBody today={today} />}
    </CareLinesGate>
  );
}

export type TeamTab = "overdue" | "due_soon" | "all";

function TeamBody({ today }: { today: string }) {
  const queryClient = useQueryClient();
  const context = useQuery({ queryKey: [ CONTEXT_KEY ], queryFn: getCareContext, ...STATIC_QUERY });
  const [ teamChoice, setTeamChoice ] = useState("");
  const [ lineId, setLineId ] = useState("");
  const [ tab, setTab ] = useState<TeamTab>("overdue");
  const [ notice, setNotice ] = useState<string | null>(null);
  const teams = context.data?.teams ?? [];
  const teamId = teamChoice || teams[0]?.id || "";
  const overdueCount = useQuery({
    queryKey: [ OVERDUE_COUNT_KEY, teamId ], queryFn: () => countOverdueEnrollments(teamId), enabled: !!teamId, ...COUNT_QUERY
  });
  const query = useInfiniteQuery({
    queryKey: [ ENROLLMENTS_KEY, teamId, lineId, tab ],
    queryFn: ({ pageParam }) => listCareEnrollments({ teamId, careLineId: lineId || undefined, status: tab, before: pageParam }),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.next_before ?? undefined,
    enabled: !!teamId,
    ...NOMINAL_QUERY
  });
  const rows = (query.data?.pages ?? []).flatMap((p) => p.enrollments);
  // Depois de uma ação: volta à 1ª página (reset) e relê os contadores.
  const reload = () => {
    void queryClient.resetQueries({ queryKey: [ ENROLLMENTS_KEY ] });
    void queryClient.invalidateQueries({ queryKey: [ OVERDUE_COUNT_KEY ] });
  };
  const changed = (message: string) => { setNotice(message); reload(); };

  async function loadMore() {
    const result = await query.fetchNextPage();
    if (result.isError && isStaleCare(result.error)) { setNotice(careLineError(result.error)); reload(); }
  }
  void today; void changed; // usados nas Tasks 7–10

  const tabLabel = (key: TeamTab, label: string) =>
    key === "overdue" && typeof overdueCount.data === "number" ? `${label} (${overdueCount.data})` : label;

  if (context.isPending) return <p className="mono" style={muted}>carregando…</p>;
  if (context.isError) return <p role="alert" style={alert}>{careLineError(context.error)}</p>;
  if (teams.length === 0) return <EmptyState title="você não está em nenhuma equipe com vínculo ativo na unidade" />;

  return (
    <div style={stack}>
      <Panel title="Equipe e linha">
        <div style={{ ...row, alignItems: "flex-end", padding: 16 }}>
          <label style={fieldLabel}>
            Equipe
            <select value={teamId} style={inputStyle} onChange={(e) => { setNotice(null); setTeamChoice(e.target.value); }}>
              {teams.map((t) => <option key={t.id} value={t.id}>{`${t.name} · INE ${t.ine} · ${t.unit.name}`}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>
            Linha
            <select value={lineId} style={inputStyle} onChange={(e) => { setNotice(null); setLineId(e.target.value); }}>
              <option value="">todas as linhas</option>
              {(context.data?.lines ?? []).map((l) => <option key={l.id} value={l.id}>{l.name}</option>)}
            </select>
          </label>
        </div>
      </Panel>
      <div role="group" aria-label="Lista" style={row}>
        {ENROLLMENT_FILTERS.map((f) => (
          <button key={f.key} type="button" aria-pressed={tab === f.key} style={tab === f.key ? buttonStyle : secondaryButtonStyle}
            onClick={() => { setNotice(null); setTab(f.key); }}>
            {tabLabel(f.key, f.label)}
          </button>
        ))}
      </div>
      {notice && <p role="status" style={statusStyle}>{notice}</p>}
      <Panel title="Pessoas da equipe" sub={tab === "overdue" ? "busca ativa: alguma boa prática vencida" : tab === "due_soon" ? "alguma boa prática vence em até 30 dias" : "todas as inscrições ativas"}>
        <div style={{ ...stack, padding: 16 }}>
          {query.isPending && <p className="mono" style={muted}>carregando…</p>}
          {query.isError && <p role="alert" style={alert}>{careLineError(query.error)}</p>}
          {/* L6: a busca ativa pode devolver página curta (ou vazia) com next_before — carrega sob demanda, sem laço. */}
          {query.data && (rows.length === 0 && query.hasNextPage
            ? <p style={muted}>Nada nesta parte da lista; carregue mais para continuar a busca.</p>
            : <EnrollmentList rows={rows} />)}
          {query.hasNextPage && (
            <div>
              <button type="button" style={secondaryButtonStyle} disabled={query.isFetchingNextPage} onClick={() => void loadMore()}>
                Carregar mais
              </button>
            </div>
          )}
        </div>
      </Panel>
    </div>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

`today` e `changed` ficam sem uso nesta task (as Tasks 7–10 os usam); a linha `void today; void changed;` acima mantém o `noUnusedLocals` verde e sai nas Tasks 7 (`changed`) e 9 (`today`).

Em `src/App.tsx`, depois de `import { CareLinesAdmin } from "./modules/CareLinesAdmin";`, acrescente

```ts
import { CareLinesTeam } from "./modules/CareLinesTeam";
```

e, antes de `    case "care-lines-admin": …`, acrescente

```tsx
    case "care-lines-team": return <CareLinesTeam />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/CareLinesTeam.test.tsx && npx tsc --noEmit`
Expected: PASS. Se o teste de tempo oscilar por causa do relógio falso com a consulta inicial, mantenha `shouldAdvanceTime: true` (os `findBy` precisam do tempo real andando) e não troque por `waitFor` com relógio parado.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/EnrollmentList.tsx src/modules/CareLinesTeam.tsx src/modules/CareLinesTeam.test.tsx src/App.tsx
$G commit -m "feat: show the team's care line panel with the active search filter

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 7: Detalhe da inscrição e registro de contato

**Files:**
- Create: `src/modules/careLines/EnrollmentDetail.tsx`, `src/modules/careLines/ContactForm.tsx`
- Modify: `src/modules/CareLinesTeam.tsx`
- Test: `src/modules/careLines/EnrollmentDetail.test.tsx`

**Interfaces:**
- Consumes: `getCareEnrollment`, `recordCareContact`, `ContactChannel`, `ContactOutcome`, `CareContact`, `CarePracticeStatus` (Task 1); `ENROLLMENT_KEY`, `NOMINAL_QUERY`, `CHANNELS`, `OUTCOMES`, `EMPTY_CONTACT`, `NOTE_MAX`, `ContactDraft`, `contactProblem`, `careLineError`, `isStaleCare`, `channelLabel`, `outcomeLabel`, `fmtDay`, `personName`, `personStatus`, `statusLabel`, `statusTone` (Task 2); `EnrollmentList` com `onOpen` (Task 6).
- Produces: `EnrollmentDetail({ enrollmentId, onChanged(message, close?), onClose })`, com o estado `action` e a função `done(message, close?)` que a Task 8 usa; `ContactForm({ enrollmentId, initialOutcome?, onSaved, onStale(message), onCancel })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/careLines/EnrollmentDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), getCareContext: vi.fn(), listCareEnrollments: vi.fn(), countOverdueEnrollments: vi.fn(),
    countCareSuggestions: vi.fn(), getCareEnrollment: vi.fn(), recordCareContact: vi.fn()
  };
});

import * as api from "../../lib/api";
import { CareLinesTeam } from "../CareLinesTeam";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY23, careContext, careUser, enrollment } from "../../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const WITH_CONTACTS = enrollment({
  last_contact: { channel: "home_visit", outcome: "reached", at: "2026-10-20T15:00:00-03:00" },
  contacts: [
    { id: "ct1", channel: "phone", outcome: "no_answer", recorded_at: "2026-10-10T09:00:00-03:00", recorded_by: { id: "us3", name: "Enf. Lúcia Prado" }, note: null },
    { id: "ct2", channel: "home_visit", outcome: "reached", recorded_at: "2026-10-20T15:00:00-03:00", recorded_by: { id: "us7", name: "ACS Pedro Alves" },
      note: "vai à unidade na sexta" }
  ]
});
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getCareContext, api.listCareEnrollments, api.countOverdueEnrollments,
    api.countCareSuggestions, api.getCareEnrollment, api.recordCareContact ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser());
  mocked(api.getCareContext).mockResolvedValue(careContext());
  mocked(api.listCareEnrollments).mockResolvedValue({ enrollments: [ enrollment() ], next_before: null });
  mocked(api.countOverdueEnrollments).mockResolvedValue(12);
  mocked(api.countCareSuggestions).mockResolvedValue(0);
  mocked(api.getCareEnrollment).mockResolvedValue(WITH_CONTACTS);
  mocked(api.recordCareContact).mockResolvedValue(undefined);
});

async function openDetail() {
  renderWithProviders(<CareLinesTeam today={TODAY23} />);
  fireEvent.click(await screen.findByRole("button", { name: "Abrir Maria das Dores" }));
  return screen.findByRole("region", { name: "Inscrição de Maria das Dores" });
}

describe("EnrollmentDetail", () => {
  it("Abrir mostra o detalhe com trilha uma vez; contatos do mais novo ao mais antigo", async () => {
    const detail = await openDetail();
    expect(api.getCareEnrollment).toHaveBeenCalledTimes(1);
    expect(api.getCareEnrollment).toHaveBeenCalledWith("en1");
    expect(await within(detail).findByText("Hemoglobina glicada")).not.toBeNull();
    const contacts = within(within(detail).getByRole("region", { name: "Contatos" })).getAllByRole("listitem");
    expect(contacts.map((li) => li.textContent)).toEqual([
      "20/10/2026 · visita domiciliar · conversou · ACS Pedro Alves vai à unidade na sexta",
      "10/10/2026 · telefone · não atendeu · Enf. Lúcia Prado"
    ]);
  });

  it("registra contato: nota vazia não vai; relê o detalhe e a lista", async () => {
    const detail = await openDetail();
    fireEvent.click(await within(detail).findByRole("button", { name: "Registrar contato" }));
    const contactForm = within(detail).getByRole("region", { name: "Registrar contato" });
    fireEvent.change(within(contactForm).getByLabelText("Como foi o contato"), { target: { value: "phone" } });
    fireEvent.change(within(contactForm).getByLabelText("Resultado"), { target: { value: "no_answer" } });
    fireEvent.click(within(contactForm).getByRole("button", { name: "Salvar contato" }));
    await waitFor(() => expect(api.recordCareContact).toHaveBeenCalledWith("en1", { channel: "phone", outcome: "no_answer", note: "" }));
    expect((await screen.findAllByRole("status"))[0].textContent).toBe("Contato registrado.");
    await waitFor(() => expect(api.getCareEnrollment).toHaveBeenCalledTimes(2));
    await waitFor(() => expect(api.listCareEnrollments).toHaveBeenCalledTimes(2));
    expect(within(detail).queryByRole("region", { name: "Registrar contato" })).toBeNull();
  });

  it("sem canal não envia; nota de 501 é recusada e fica no campo", async () => {
    const detail = await openDetail();
    fireEvent.click(await within(detail).findByRole("button", { name: "Registrar contato" }));
    const contactForm = within(detail).getByRole("region", { name: "Registrar contato" });
    fireEvent.click(within(contactForm).getByRole("button", { name: "Salvar contato" }));
    expect(within(contactForm).getByRole("alert").textContent).toBe("escolha como foi o contato");
    fireEvent.change(within(contactForm).getByLabelText("Como foi o contato"), { target: { value: "phone" } });
    fireEvent.change(within(contactForm).getByLabelText("Resultado"), { target: { value: "reached" } });
    fireEvent.change(within(contactForm).getByLabelText("Nota (opcional)"), { target: { value: "x".repeat(501) } });
    fireEvent.click(within(contactForm).getByRole("button", { name: "Salvar contato" }));
    expect(within(contactForm).getByRole("alert").textContent).toBe("nota com até 500 caracteres — encurte");
    expect((within(contactForm).getByLabelText("Nota (opcional)") as HTMLTextAreaElement).value).toHaveLength(501);
    expect(api.recordCareContact).not.toHaveBeenCalled();
  });

  it("inscrição encerrada por outra pessoa (409 invalid_transition): frase, fecha e relê a lista", async () => {
    mocked(api.recordCareContact).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    const detail = await openDetail();
    fireEvent.click(await within(detail).findByRole("button", { name: "Registrar contato" }));
    const contactForm = within(detail).getByRole("region", { name: "Registrar contato" });
    fireEvent.change(within(contactForm).getByLabelText("Como foi o contato"), { target: { value: "in_person" } });
    fireEvent.change(within(contactForm).getByLabelText("Resultado"), { target: { value: "reached" } });
    fireEvent.click(within(contactForm).getByRole("button", { name: "Salvar contato" }));
    expect((await screen.findByRole("status")).textContent).toBe("isso já foi decidido ou encerrado por outra pessoa — a lista foi relida");
    expect(screen.queryByRole("region", { name: "Inscrição de Maria das Dores" })).toBeNull();
    await waitFor(() => expect(api.listCareEnrollments).toHaveBeenCalledTimes(2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/careLines/EnrollmentDetail.test.tsx`
Expected: FAIL — não há botão "Abrir Maria das Dores" (a lista ainda não recebe `onOpen`).

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/ContactForm.tsx
// Tentativa de contato da busca ativa (contrato §5): canal, resultado e nota
// opcional (até 500; guardada cifrada pelo api). Só acréscimo. A nota vai só
// no corpo; vazia não vai.
import { useRef, useState } from "react";
import { recordCareContact, type ContactChannel, type ContactOutcome } from "../../lib/api";
import { CHANNELS, EMPTY_CONTACT, NOTE_MAX, OUTCOMES, careLineError, contactProblem, isStaleCare, type ContactDraft } from "../../lib/careLines";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, muted, row } from "../exams/styles";

export function ContactForm({ enrollmentId, initialOutcome, onSaved, onStale, onCancel }: {
  enrollmentId: string; initialOutcome?: ContactOutcome; onSaved(): void; onStale(message: string): void; onCancel(): void;
}) {
  const [ d, setD ] = useState<ContactDraft>({ ...EMPTY_CONTACT, outcome: initialOutcome ?? "" });
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);
  const set = (patch: Partial<ContactDraft>) => { setError(null); setD((x) => ({ ...x, ...patch })); };

  async function submit() {
    const problem = contactProblem(d);
    if (problem) { setError(problem); return; }
    if (running.current) return;
    running.current = true;
    setBusy(true);
    try {
      await recordCareContact(enrollmentId, { channel: d.channel as ContactChannel, outcome: d.outcome as ContactOutcome, note: d.note });
      onSaved();
    } catch (err) {
      if (isStaleCare(err)) onStale(careLineError(err));
      else setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  return (
    <section aria-label="Registrar contato" style={box}>
      <div style={form}>
        <div style={row}>
          <label style={fieldLabel}>
            Como foi o contato
            <select value={d.channel} style={inputStyle} onChange={(e) => set({ channel: e.target.value as ContactDraft["channel"] })}>
              <option value="">escolha</option>
              {CHANNELS.map((c) => <option key={c.key} value={c.key}>{c.label}</option>)}
            </select>
          </label>
          <label style={fieldLabel}>
            Resultado
            <select value={d.outcome} style={inputStyle} onChange={(e) => set({ outcome: e.target.value as ContactDraft["outcome"] })}>
              <option value="">escolha</option>
              {OUTCOMES.map((o) => <option key={o.key} value={o.key}>{o.label}</option>)}
            </select>
          </label>
        </div>
        <label style={fieldLabel}>
          Nota (opcional)
          <textarea rows={3} value={d.note} style={inputStyle} onChange={(e) => set({ note: e.target.value })} />
        </label>
        <p style={muted}>até {NOTE_MAX} caracteres; a nota fica guardada cifrada e só a equipe a vê</p>
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <button type="button" style={buttonStyle} disabled={busy} onClick={() => void submit()}>Salvar contato</button>
          <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}
```

```tsx
// src/modules/careLines/EnrollmentDetail.tsx
// Detalhe de uma inscrição (contrato §5; Divergência D5): cada boa prática
// com a última vez, o vencimento e a contagem; o histórico de contatos;
// registrar contato. Leitura NOMINAL com trilha: ao abrir e depois de uma
// ação, nunca sozinha. Inscrição encerrada fica só leitura.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { getCareEnrollment, type CareContact, type CarePracticeStatus } from "../../lib/api";
import {
  ENROLLMENT_KEY, NOMINAL_QUERY, careLineError, channelLabel, endReasonLabel, fmtDay, outcomeLabel, personName, personStatus, statusLabel,
  statusTone
} from "../../lib/careLines";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { KeyValue } from "../../components/KeyValue";
import { secondaryButtonStyle } from "../../components/formStyles";
import { alert, grid, list, muted, row, stack } from "../exams/styles";
import { ContactForm } from "./ContactForm";

export type DetailAction = "contact" | null;

const contactLine = (c: CareContact) =>
  `${fmtDay(c.recorded_at)} · ${channelLabel(c.channel)} · ${outcomeLabel(c.outcome)}${c.recorded_by ? ` · ${c.recorded_by.name}` : ""}`;

export function EnrollmentDetail({ enrollmentId, onChanged, onClose }: {
  enrollmentId: string; onChanged(message: string, close?: boolean): void; onClose(): void;
}) {
  const query = useQuery({ queryKey: [ ENROLLMENT_KEY, enrollmentId ], queryFn: () => getCareEnrollment(enrollmentId), ...NOMINAL_QUERY });
  const [ action, setAction ] = useState<DetailAction>(null);
  const [ notice, setNotice ] = useState<string | null>(null);
  const e = query.data;
  const active = e?.status === "active";
  const contacts = [ ...(e?.contacts ?? []) ].sort((a, b) => b.recorded_at.localeCompare(a.recorded_at));

  // Depois de uma ação: relê o detalhe (ou fecha) e avisa a tela, que relê a lista.
  function done(message: string, close = false) {
    setAction(null);
    setNotice(message);
    if (!close) void query.refetch();
    onChanged(message, close);
  }

  return (
    <Panel title={e ? `Inscrição de ${personName(e.person)}` : "Inscrição"} sub={e ? `${e.care_line.name} · ${e.team.name}` : undefined}
      right={<button type="button" style={secondaryButtonStyle} onClick={onClose}>Fechar</button>}>
      <div style={{ ...stack, padding: 16 }}>
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{careLineError(query.error)}</p>}
        {notice && <p role="status" style={statusStyle}>{notice}</p>}
        {e && (
          <>
            <div style={grid}>
              <KeyValue k="Situação" v={statusLabel(personStatus(e))} mono={false} />
              <KeyValue k="Inscrita em" v={fmtDay(e.enrolled_at)} />
              {e.anchor_date && <KeyValue k={e.care_line.requires_anchor === "lmp" ? "DUM" : "Data de referência"} v={fmtDay(e.anchor_date)} />}
              {!active && <KeyValue k="Inscrição" v={e.ended_reason ? `encerrada — ${endReasonLabel(e.ended_reason)}` : "encerrada"} mono={false} />}
            </div>
            <DataTable<CarePracticeStatus>
              cols={[
                { label: "Boa prática", w: "2fr", card: "title", render: (p) => p.label },
                { label: "Situação", w: "150px", render: (p) => <Tag tone={statusTone(p.status)}>{statusLabel(p.status)}</Tag> },
                { label: "Última vez", w: "110px", render: (p) => (p.status === "untracked" ? "—" : p.done_at ? fmtDay(p.done_at) : "nunca") },
                { label: "Vence", w: "110px", render: (p) => (p.status === "untracked" ? "—" : fmtDay(p.due_on)) },
                { label: "Contagem", w: "110px", render: (p) => (p.count !== null && p.expected !== null ? `${p.count} de ${p.expected}` : "—") }
              ]}
              rows={e.practices} rowKey={(p) => p.key}
            />
            <section aria-label="Contatos" style={stack}>
              <strong>Contatos</strong>
              {contacts.length === 0 ? <p style={muted}>nenhum contato registrado</p> : (
                <ul style={list}>
                  {contacts.map((c, i) => (
                    <li key={c.id ?? `${c.recorded_at}-${i}`}>
                      {contactLine(c)}
                      {c.note && <span style={noteStyle}> {c.note}</span>}
                    </li>
                  ))}
                </ul>
              )}
            </section>
            {active && action === null && (
              <div style={row}>
                <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setAction("contact"); }}>Registrar contato</button>
              </div>
            )}
            {active && action === "contact" && (
              <ContactForm enrollmentId={e.id} onSaved={() => done("Contato registrado.")} onStale={(m) => done(m, true)}
                onCancel={() => setAction(null)} />
            )}
          </>
        )}
      </div>
    </Panel>
  );
}

const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
const noteStyle: CSSProperties = { color: "var(--ink2)" };
```

Em `src/modules/CareLinesTeam.tsx`:

1. Troque `import { EnrollmentList } from "./careLines/EnrollmentList";` por

```ts
import { EnrollmentList } from "./careLines/EnrollmentList";
import { EnrollmentDetail } from "./careLines/EnrollmentDetail";
```

2. Depois de `  const [ notice, setNotice ] = useState<string | null>(null);`, acrescente

```ts
  const [ openId, setOpenId ] = useState<string | null>(null);
```

3. Troque `  void today; void changed; // usados nas Tasks 7–10` por

```ts
  void today; // usado nas Tasks 9 e 10
```

4. Troque `            : <EnrollmentList rows={rows} />)}` por

```tsx
            : <EnrollmentList rows={rows} onOpen={(e) => { setNotice(null); setOpenId(e.id); }} />)}
```

5. Antes de `      <Panel title="Pessoas da equipe"`, acrescente

```tsx
      {openId && (
        <EnrollmentDetail key={openId} enrollmentId={openId} onClose={() => setOpenId(null)}
          onChanged={(message, close) => { if (close) setOpenId(null); changed(message); }} />
      )}
```

Assim, o aviso da ação aparece no detalhe e na tela (`role="status"` nos dois; o detalhe some quando fecha).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/careLines/EnrollmentDetail.test.tsx src/modules/CareLinesTeam.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/EnrollmentDetail.tsx src/modules/careLines/ContactForm.tsx src/modules/careLines/EnrollmentDetail.test.tsx src/modules/CareLinesTeam.tsx
$G commit -m "feat: open a care line enrollment and record outreach contacts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 8: Encerrar com motivo e atalho para agendar (módulo 17)

**Files:**
- Create: `src/modules/careLines/EndEnrollmentForm.tsx`, `src/modules/careLines/CareAppointment.tsx`
- Modify: `src/modules/careLines/EnrollmentDetail.tsx`
- Test: `src/modules/careLines/EnrollmentActions.test.tsx`

**Interfaces:**
- Consumes: `endCareEnrollment`, `openCareAppointment`, `CareAppointmentTarget`, `ContactOutcome`, `EndReason` (Task 1); `END_REASONS`, `EMPTY_END`, `EndDraft`, `NOTE_MAX`, `endProblem`, `careLineError`, `isStaleCare`, `personName` (Task 2); `EnrollmentDetail`, `ContactForm` (Task 7); `BookPanel({ row, unit, onCancel, onDone })` (módulo 17, sem mudança).
- Produces: `EndEnrollmentForm({ enrollmentId, person, initialReason?, onEnded(message), onStale(message), onCancel })`; `CareAppointment({ enrollmentId, onBooked })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/careLines/EnrollmentActions.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getCareEnrollment: vi.fn(), endCareEnrollment: vi.fn(), openCareAppointment: vi.fn(),
    recordCareContact: vi.fn() };
});
// O BookPanel do módulo 17 tem os próprios testes; aqui só o que o atalho entrega a ele.
vi.mock("../attendance/BookPanel", () => ({
  BookPanel: (p: { row: { id: string }; unit: { name: string }; onDone(): void; onCancel(): void }) => (
    <div role="region" aria-label="BookPanel do módulo 17">
      <span>{`pedido ${p.row.id} · ${p.unit.name}`}</span>
      <button type="button" onClick={p.onDone}>marcar (dublê)</button>
    </div>
  )
}));

import * as api from "../../lib/api";
import { EnrollmentDetail } from "./EnrollmentDetail";
import { renderWithProviders } from "../../test/campaignFixtures";
import { UBS_VILA_VERDE, careUser, enrollment, requestRow } from "../../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getCareEnrollment, api.endCareEnrollment, api.openCareAppointment, api.recordCareContact ]) {
    mocked(fn).mockReset();
  }
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser());
  mocked(api.getCareEnrollment).mockResolvedValue(enrollment({ contacts: [] }));
  mocked(api.endCareEnrollment).mockResolvedValue(undefined);
  mocked(api.openCareAppointment).mockResolvedValue({ request: requestRow(), unit: UBS_VILA_VERDE });
});

function renderDetail(onChanged = vi.fn()) {
  renderWithProviders(<EnrollmentDetail enrollmentId="en1" onChanged={onChanged} onClose={vi.fn()} />);
  return onChanged;
}

describe("ações da inscrição", () => {
  it("Agendar abre o pedido do módulo 17 da pessoa; marcado, o contato vem com agendou", async () => {
    renderDetail();
    fireEvent.click(await screen.findByRole("button", { name: "Agendar" }));
    const book = await screen.findByRole("region", { name: "BookPanel do módulo 17" });
    expect(api.openCareAppointment).toHaveBeenCalledWith("en1");
    expect(within(book).getByText("pedido rq1 · UBS Vila Verde")).not.toBeNull();
    fireEvent.click(within(book).getByRole("button", { name: "marcar (dublê)" }));
    expect((await screen.findByRole("status")).textContent).toBe("Horário marcado. Registre o contato como “agendou”.");
    const contactForm = screen.getByRole("region", { name: "Registrar contato" });
    expect((within(contactForm).getByLabelText("Resultado") as HTMLSelectElement).value).toBe("scheduled");
    expect(screen.queryByRole("region", { name: "BookPanel do módulo 17" })).toBeNull();
  });

  it("pedido que o api não consegue abrir: frase e nada de BookPanel", async () => {
    mocked(api.openCareAppointment).mockRejectedValueOnce(new api.ApiError(422, { error: "no_open_slot_unit" }, "x"));
    renderDetail();
    fireEvent.click(await screen.findByRole("button", { name: "Agendar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("a equipe não tem unidade com agenda para marcar — fale com a recepção");
    expect(screen.queryByRole("region", { name: "BookPanel do módulo 17" })).toBeNull();
  });

  it("encerrar: motivo obrigatório; outro motivo pede nota; manda motivo e nota e fecha", async () => {
    const onChanged = renderDetail();
    fireEvent.click(await screen.findByRole("button", { name: "Encerrar inscrição" }));
    const endForm = screen.getByRole("region", { name: "Encerrar a inscrição de Maria das Dores" });
    fireEvent.click(within(endForm).getByRole("button", { name: "Encerrar inscrição" }));
    expect(within(endForm).getByRole("alert").textContent).toBe("escolha o motivo do encerramento");
    fireEvent.change(within(endForm).getByLabelText("Motivo do encerramento"), { target: { value: "other" } });
    fireEvent.click(within(endForm).getByRole("button", { name: "Encerrar inscrição" }));
    expect(within(endForm).getByRole("alert").textContent).toBe("descreva o motivo em “outro motivo”");
    fireEvent.change(within(endForm).getByLabelText("Nota"), { target: { value: "acompanhamento no plano de saúde" } });
    fireEvent.click(within(endForm).getByRole("button", { name: "Encerrar inscrição" }));
    await waitFor(() => expect(api.endCareEnrollment).toHaveBeenCalledWith("en1", "other", "acompanhamento no plano de saúde"));
    await waitFor(() => expect(onChanged).toHaveBeenCalledWith("Inscrição de Maria das Dores encerrada.", true));
  });

  it("inscrição encerrada: só leitura — sem Agendar, Encerrar nem Registrar contato", async () => {
    mocked(api.getCareEnrollment).mockResolvedValue(enrollment({ status: "ended", ended_reason: "moved", ended_at: "2026-10-21T10:00:00-03:00", contacts: [] }));
    renderDetail();
    expect(await screen.findByText("encerrada — mudou de território")).not.toBeNull();
    for (const name of [ "Agendar", "Encerrar inscrição", "Registrar contato" ]) expect(screen.queryByRole("button", { name })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/careLines/EnrollmentActions.test.tsx`
Expected: FAIL — não há "Agendar" nem "Encerrar inscrição".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/EndEnrollmentForm.tsx
// Encerrar a inscrição com motivo (contrato §1 e §4): alta, óbito, mudança,
// fim da gestação ou outro (com nota). O encerramento não apaga nada: a
// inscrição e os contatos ficam (documento clínico, api#84). Nota até 500,
// só no corpo; vazia não vai.
import { useRef, useState } from "react";
import { endCareEnrollment, type EndReason } from "../../lib/api";
import { EMPTY_END, END_REASONS, NOTE_MAX, careLineError, endProblem, isStaleCare, type EndDraft } from "../../lib/careLines";
import { dangerButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, muted, row } from "../exams/styles";

export function EndEnrollmentForm({ enrollmentId, person, initialReason, onEnded, onStale, onCancel }: {
  enrollmentId: string; person: string; initialReason?: EndReason; onEnded(message: string): void; onStale(message: string): void; onCancel(): void;
}) {
  const [ d, setD ] = useState<EndDraft>({ ...EMPTY_END, reason: initialReason ?? "" });
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);
  const set = (patch: Partial<EndDraft>) => { setError(null); setD((x) => ({ ...x, ...patch })); };

  async function submit() {
    const problem = endProblem(d);
    if (problem) { setError(problem); return; }
    if (running.current) return;
    running.current = true;
    setBusy(true);
    try {
      await endCareEnrollment(enrollmentId, d.reason as EndReason, d.note);
      onEnded(`Inscrição de ${person} encerrada.`);
    } catch (err) {
      if (isStaleCare(err)) onStale(careLineError(err));
      else setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  return (
    <section aria-label={`Encerrar a inscrição de ${person}`} style={box}>
      <div style={form}>
        <label style={fieldLabel}>
          Motivo do encerramento
          <select value={d.reason} style={inputStyle} onChange={(e) => set({ reason: e.target.value as EndDraft["reason"] })}>
            <option value="">escolha</option>
            {END_REASONS.map((r) => <option key={r.key} value={r.key}>{r.label}</option>)}
          </select>
        </label>
        <label style={fieldLabel}>
          Nota
          <textarea rows={2} value={d.note} style={inputStyle} onChange={(e) => set({ note: e.target.value })} />
        </label>
        <p style={muted}>obrigatória em “outro motivo”; até {NOTE_MAX} caracteres. O histórico da pessoa não é apagado.</p>
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <button type="button" style={dangerButtonStyle} disabled={busy} onClick={() => void submit()}>Encerrar inscrição</button>
          <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}
```

```tsx
// src/modules/careLines/CareAppointment.tsx
// Atalho para agendar (contrato §5; Divergência D7): pede ao api o pedido de
// agendamento do módulo 17 da pessoa na unidade da equipe (o aberto, ou um
// novo) e abre o BookPanel do módulo 17 com ele — a marcação é a mesma da
// recepção (vagas, encaixe, prazos). Só no clique (é uma escrita).
import { useRef, useState } from "react";
import { openCareAppointment, type CareAppointmentTarget } from "../../lib/api";
import { careLineError } from "../../lib/careLines";
import { secondaryButtonStyle } from "../../components/formStyles";
import { alert, stack } from "../exams/styles";
import { BookPanel } from "../attendance/BookPanel";

export function CareAppointment({ enrollmentId, onBooked }: { enrollmentId: string; onBooked(): void }) {
  const [ target, setTarget ] = useState<CareAppointmentTarget | null>(null);
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);

  async function open() {
    if (running.current) return;
    running.current = true;
    setBusy(true);
    setError(null);
    try {
      setTarget(await openCareAppointment(enrollmentId));
    } catch (err) {
      setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  if (target) {
    return (
      <section aria-label="Marcar horário" style={stack}>
        <BookPanel row={target.request} unit={target.unit} onCancel={() => setTarget(null)} onDone={() => { setTarget(null); onBooked(); }} />
      </section>
    );
  }
  return (
    <div style={stack}>
      <div><button type="button" style={secondaryButtonStyle} disabled={busy} onClick={() => void open()}>Agendar</button></div>
      {error && <p role="alert" style={alert}>{error}</p>}
    </div>
  );
}
```

Em `src/modules/careLines/EnrollmentDetail.tsx`:

1. Troque `import { getCareEnrollment, type CareContact, type CarePracticeStatus } from "../../lib/api";` por

```ts
import { getCareEnrollment, type CareContact, type CarePracticeStatus, type ContactOutcome } from "../../lib/api";
```

e `import { ContactForm } from "./ContactForm";` por

```ts
import { ContactForm } from "./ContactForm";
import { EndEnrollmentForm } from "./EndEnrollmentForm";
import { CareAppointment } from "./CareAppointment";
```

2. Troque `export type DetailAction = "contact" | null;` por

```ts
export type DetailAction = "contact" | "end" | null;
```

3. Depois de `  const [ notice, setNotice ] = useState<string | null>(null);`, acrescente

```ts
  // Depois de marcar horário, o contato já vem com "agendou".
  const [ contactOutcome, setContactOutcome ] = useState<ContactOutcome | undefined>(undefined);
```

4. Troque o bloco

```tsx
            {active && action === null && (
              <div style={row}>
                <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setAction("contact"); }}>Registrar contato</button>
              </div>
            )}
            {active && action === "contact" && (
              <ContactForm enrollmentId={e.id} onSaved={() => done("Contato registrado.")} onStale={(m) => done(m, true)}
                onCancel={() => setAction(null)} />
            )}
```

por

```tsx
            {active && action === null && (
              <div style={row}>
                <button type="button" style={secondaryButtonStyle}
                  onClick={() => { setNotice(null); setContactOutcome(undefined); setAction("contact"); }}>Registrar contato</button>
                <CareAppointment enrollmentId={e.id} onBooked={() => {
                  setNotice("Horário marcado. Registre o contato como “agendou”.");
                  setContactOutcome("scheduled");
                  setAction("contact");
                }} />
                <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setAction("end"); }}>Encerrar inscrição</button>
              </div>
            )}
            {active && action === "contact" && (
              <ContactForm key={contactOutcome ?? "blank"} enrollmentId={e.id} initialOutcome={contactOutcome}
                onSaved={() => done("Contato registrado.")} onStale={(m) => done(m, true)} onCancel={() => setAction(null)} />
            )}
            {active && action === "end" && (
              <EndEnrollmentForm enrollmentId={e.id} person={personName(e.person)} onEnded={(m) => done(m, true)}
                onStale={(m) => done(m, true)} onCancel={() => setAction(null)} />
            )}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/careLines && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/EndEnrollmentForm.tsx src/modules/careLines/CareAppointment.tsx src/modules/careLines/EnrollmentDetail.tsx src/modules/careLines/EnrollmentActions.test.tsx
$G commit -m "feat: end care line enrollments and book from the outreach panel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 9: Sugestões a confirmar

**Files:**
- Create: `src/modules/careLines/SuggestionsPanel.tsx`
- Modify: `src/modules/CareLinesTeam.tsx`
- Test: `src/modules/careLines/SuggestionsPanel.test.tsx`

**Interfaces:**
- Consumes: `listCareSuggestions`, `acceptCareSuggestion`, `dismissCareSuggestion`, `countCareSuggestions`, `errorCode`, `CareSuggestion`, `CareTeam` (Task 1); `NOMINAL_QUERY`, `COUNT_QUERY`, `SUGGESTIONS_KEY`, `SUGGESTION_COUNT_KEY`, `anchorProblem`, `careLineError`, `fmtDay`, `isStaleCare`, `personName`, `reasonLabel`, `reasonProblem` (Task 2); `TeamBody` (Tasks 6–7).
- Produces: `SuggestionsPanel({ teamId, careLineId, teams, canConfirm, today, onChanged(message) })`; `TeamTab` passa a incluir `"suggestions"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/careLines/SuggestionsPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), getCareContext: vi.fn(), listCareEnrollments: vi.fn(), countOverdueEnrollments: vi.fn(),
    countCareSuggestions: vi.fn(), listCareSuggestions: vi.fn(), acceptCareSuggestion: vi.fn(), dismissCareSuggestion: vi.fn(),
    endCareEnrollment: vi.fn()
  };
});

import * as api from "../../lib/api";
import { CareLinesTeam } from "../CareLinesTeam";
import { renderWithProviders } from "../../test/campaignFixtures";
import { PREGNANCY_REF, TODAY23, careContext, careUser, enrollment, suggestion } from "../../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const ANA = suggestion({ id: "sg2", person: { patient_id: null, citizen_id: "ci7", display_name: "Ana Souza", age: 27 }, care_line: PREGNANCY_REF,
  reason: "declared" });
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getCareContext, api.listCareEnrollments, api.countOverdueEnrollments,
    api.countCareSuggestions, api.listCareSuggestions, api.acceptCareSuggestion, api.dismissCareSuggestion, api.endCareEnrollment ]) {
    mocked(fn).mockReset();
  }
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser());
  mocked(api.getCareContext).mockResolvedValue(careContext());
  mocked(api.listCareEnrollments).mockResolvedValue({ enrollments: [ enrollment() ], next_before: null });
  mocked(api.countOverdueEnrollments).mockResolvedValue(12);
  mocked(api.countCareSuggestions).mockResolvedValue(3);
  mocked(api.listCareSuggestions).mockResolvedValue({ suggestions: [ suggestion() ], next_before: null });
  mocked(api.acceptCareSuggestion).mockResolvedValue(undefined);
  mocked(api.dismissCareSuggestion).mockResolvedValue(undefined);
  mocked(api.endCareEnrollment).mockResolvedValue(undefined);
});

async function openSuggestions() {
  renderWithProviders(<CareLinesTeam today={TODAY23} />);
  fireEvent.click(await screen.findByRole("button", { name: "Sugestões (3)" }));
  return screen.findByRole("region", { name: "Sugestões a confirmar" });
}

describe("SuggestionsPanel", () => {
  it("a lista só é lida no clique da aba; o motivo sem texto clínico", async () => {
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    await screen.findByRole("button", { name: "Sugestões (3)" });
    expect(api.listCareSuggestions).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Sugestões (3)" }));
    const panel = await screen.findByRole("region", { name: "Sugestões a confirmar" });
    expect(await within(panel).findByText("problema ativo na lista de problemas")).not.toBeNull();
    expect(api.listCareSuggestions).toHaveBeenCalledWith({ teamId: "tm1", careLineId: undefined, before: null });
    expect(api.listCareSuggestions).toHaveBeenCalledTimes(1);
  });

  it("aceita escolhendo a equipe de referência; relê a lista e os contadores", async () => {
    const panel = await openSuggestions();
    fireEvent.click(await within(panel).findByRole("button", { name: "Aceitar sugestão de João Batista" }));
    const accept = within(panel).getByRole("region", { name: "Aceitar João Batista em Pessoa com diabetes" });
    expect(within(accept).queryByLabelText("DUM (data da última menstruação)")).toBeNull();
    fireEvent.change(within(accept).getByLabelText("Equipe de referência"), { target: { value: "tm2" } });
    fireEvent.click(within(accept).getByRole("button", { name: "Confirmar inscrição" }));
    await waitFor(() => expect(api.acceptCareSuggestion).toHaveBeenCalledWith("sg1", { team_id: "tm2" }));
    expect((await screen.findByRole("status")).textContent).toBe("Inscrição de João Batista em Pessoa com diabetes confirmada.");
    await waitFor(() => expect(api.listCareSuggestions).toHaveBeenCalledTimes(2));
    await waitFor(() => expect(mocked(api.countCareSuggestions).mock.calls.length).toBeGreaterThanOrEqual(2));
  });

  it("gestante: sem DUM não envia; DUM no futuro recusa; 422 anchor_required abre o campo", async () => {
    mocked(api.listCareSuggestions).mockResolvedValue({ suggestions: [ ANA, suggestion() ], next_before: null });
    const panel = await openSuggestions();
    fireEvent.click(await within(panel).findByRole("button", { name: "Aceitar sugestão de Ana Souza" }));
    const ana = within(panel).getByRole("region", { name: "Aceitar Ana Souza em Gestante e puérpera" });
    fireEvent.click(within(ana).getByRole("button", { name: "Confirmar inscrição" }));
    expect(within(ana).getByRole("alert").textContent).toBe("informe a DUM (data da última menstruação)");
    fireEvent.change(within(ana).getByLabelText("DUM (data da última menstruação)"), { target: { value: "2026-10-23" } });
    fireEvent.click(within(ana).getByRole("button", { name: "Confirmar inscrição" }));
    expect(within(ana).getByRole("alert").textContent).toBe("a DUM não pode ser depois de hoje");
    expect(api.acceptCareSuggestion).not.toHaveBeenCalled();
    fireEvent.change(within(ana).getByLabelText("DUM (data da última menstruação)"), { target: { value: "2026-08-01" } });
    fireEvent.click(within(ana).getByRole("button", { name: "Confirmar inscrição" }));
    await waitFor(() => expect(api.acceptCareSuggestion).toHaveBeenCalledWith("sg2", { team_id: "tm1", anchor_date: "2026-08-01" }));

    // Linha que a tela não sabia que pedia âncora: o api recusa e o campo aparece.
    mocked(api.acceptCareSuggestion).mockRejectedValueOnce(new api.ApiError(422, { error: "anchor_required" }, "x"));
    fireEvent.click(await within(panel).findByRole("button", { name: "Aceitar sugestão de João Batista" }));
    const joao = within(panel).getByRole("region", { name: "Aceitar João Batista em Pessoa com diabetes" });
    fireEvent.click(within(joao).getByRole("button", { name: "Confirmar inscrição" }));
    expect((await within(joao).findByRole("alert")).textContent).toBe("informe a DUM para inscrever na linha da gestante");
    expect(within(joao).getByLabelText("DUM (data da última menstruação)")).not.toBeNull();
    expect(api.acceptCareSuggestion).toHaveBeenCalledTimes(2);
  });

  it("sem can_confirm: só Recusar; 403 cbo_not_allowed vira frase e nada muda", async () => {
    mocked(api.getCareContext).mockResolvedValue(careContext({ canConfirm: false }));
    let panel = await openSuggestions();
    expect(await within(panel).findByRole("button", { name: "Recusar sugestão de João Batista" })).not.toBeNull();
    expect(within(panel).queryByRole("button", { name: "Aceitar sugestão de João Batista" })).toBeNull();
    cleanup();
    mocked(api.getCareContext).mockResolvedValue(careContext());
    mocked(api.listCareSuggestions).mockClear();
    mocked(api.acceptCareSuggestion).mockRejectedValueOnce(new api.ApiError(403, { error: "cbo_not_allowed" }, "x"));
    panel = await openSuggestions();
    fireEvent.click(await within(panel).findByRole("button", { name: "Aceitar sugestão de João Batista" }));
    fireEvent.click(within(panel).getByRole("button", { name: "Confirmar inscrição" }));
    expect((await within(panel).findByRole("alert")).textContent).toBe("só médico ou enfermeiro confirma a inscrição");
    expect(api.listCareSuggestions).toHaveBeenCalledTimes(1);
    expect(screen.queryByRole("status")).toBeNull();
  });

  it("recusar exige motivo (1 a 500) e manda o motivo no corpo", async () => {
    const panel = await openSuggestions();
    fireEvent.click(await within(panel).findByRole("button", { name: "Recusar sugestão de João Batista" }));
    const dismiss = within(panel).getByRole("region", { name: "Recusar a sugestão de João Batista" });
    fireEvent.click(within(dismiss).getByRole("button", { name: "Recusar sugestão" }));
    expect(within(dismiss).getByRole("alert").textContent).toBe("escreva o motivo");
    fireEvent.change(within(dismiss).getByLabelText("Motivo da recusa"), { target: { value: "  já acompanhado no ambulatório de endocrinologia " } });
    fireEvent.click(within(dismiss).getByRole("button", { name: "Recusar sugestão" }));
    await waitFor(() => expect(api.dismissCareSuggestion).toHaveBeenCalledWith("sg1", "já acompanhado no ambulatório de endocrinologia"));
    expect((await screen.findByRole("status")).textContent).toBe("Sugestão de João Batista recusada.");
  });

  it("decidida por outra pessoa (409 invalid_transition): frase e relê", async () => {
    mocked(api.acceptCareSuggestion).mockRejectedValueOnce(new api.ApiError(409, { error: "invalid_transition" }, "x"));
    const panel = await openSuggestions();
    fireEvent.click(await within(panel).findByRole("button", { name: "Aceitar sugestão de João Batista" }));
    fireEvent.click(within(panel).getByRole("button", { name: "Confirmar inscrição" }));
    expect((await screen.findByRole("status")).textContent).toBe("isso já foi decidido ou encerrado por outra pessoa — a lista foi relida");
    await waitFor(() => expect(api.listCareSuggestions).toHaveBeenCalledTimes(2));
  });

  it("problema resolvido (L5): Encerrar aceita a sugestão sem equipe e encerra com alta; Manter pede motivo", async () => {
    mocked(api.listCareSuggestions).mockResolvedValue({ suggestions: [ suggestion({ reason: "resolved_problem", enrollment_id: "en1" }) ], next_before: null });
    const panel = await openSuggestions();
    expect(await within(panel).findByText("problema resolvido — sugestão de encerrar")).not.toBeNull();
    expect(within(panel).queryByRole("button", { name: "Aceitar sugestão de João Batista" })).toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Manter inscrição de João Batista" }));
    expect(within(panel).getByLabelText("Por que manter a inscrição")).not.toBeNull();
    fireEvent.click(within(panel).getByRole("button", { name: "Encerrar inscrição de João Batista" }));
    const endForm = within(panel).getByRole("region", { name: "Encerrar a inscrição de João Batista" });
    fireEvent.click(within(endForm).getByRole("button", { name: "Encerrar com alta" }));
    await waitFor(() => expect(api.acceptCareSuggestion).toHaveBeenCalledWith("sg1", {}));
    expect(api.endCareEnrollment).not.toHaveBeenCalled();
    expect((await screen.findByRole("status")).textContent).toBe("Inscrição de João Batista encerrada com alta.");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/careLines/SuggestionsPanel.test.tsx`
Expected: FAIL — não há a aba "Sugestões (3)".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/SuggestionsPanel.tsx
// Sugestões a confirmar (contrato §4; spec §2 decisões 2 e 8; ADR 0040): o
// sistema sugere (problema ativo, idade e sexo, condição declarada) e um
// profissional SEMPRE decide. Aceitar é só de médico ou enfermeiro
// (`can_confirm`, D2; o api recusa com cbo_not_allowed), escolhendo a equipe
// de referência; a linha da gestante pede a DUM (D3; 422 anchor_required abre
// o campo). Recusar pede motivo (1 a 500, D8). Problema resolvido sugere
// ENCERRAR (D4) e nunca encerra sozinho. A lista é NOMINAL com trilha: lida ao
// abrir a aba e depois de uma ação, nunca sozinha.
import { useRef, useState } from "react";
import { useInfiniteQuery } from "@tanstack/react-query";
import { acceptCareSuggestion, dismissCareSuggestion, errorCode, listCareSuggestions, type CareSuggestion, type CareTeam } from "../../lib/api";
import {
  NOMINAL_QUERY, SUGGESTIONS_KEY, anchorProblem, careLineError, fmtDay, isStaleCare, personName, reasonLabel, reasonProblem
} from "../../lib/careLines";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, muted, row, stack } from "../exams/styles";
type Acting = { kind: "accept" | "dismiss" | "end"; suggestion: CareSuggestion } | null;
type Done = (message: string) => void;

export function SuggestionsPanel({ teamId, careLineId, teams, canConfirm, today, onChanged }: {
  teamId: string; careLineId: string; teams: CareTeam[]; canConfirm: boolean; today: string; onChanged: Done;
}) {
  const query = useInfiniteQuery({
    queryKey: [ SUGGESTIONS_KEY, teamId, careLineId ],
    queryFn: ({ pageParam }) => listCareSuggestions({ teamId, careLineId: careLineId || undefined, before: pageParam }),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.next_before ?? undefined,
    ...NOMINAL_QUERY
  });
  const rows = (query.data?.pages ?? []).flatMap((p) => p.suggestions);
  const [ acting, setActing ] = useState<Acting>(null);
  const finish: Done = (message) => { setActing(null); onChanged(message); };
  const act = (kind: "accept" | "dismiss" | "end", s: CareSuggestion) => setActing({ kind, suggestion: s });

  return (
    <Panel title="Sugestões a confirmar" sub="nada é inscrito sem um profissional confirmar">
      <div style={{ ...stack, padding: 16 }}>
        {query.isPending && <p className="mono" style={muted}>carregando…</p>}
        {query.isError && <p role="alert" style={alert}>{careLineError(query.error)}</p>}
        {query.data && (
          <DataTable<CareSuggestion>
            cols={[
              { label: "Pessoa", w: "1.5fr", card: "title", render: (s) => `${personName(s.person)}${s.person.age !== null ? ` · ${s.person.age} anos` : ""}` },
              { label: "Linha", w: "1.2fr", render: (s) => s.care_line.name },
              { label: "Por quê", w: "1.6fr", render: (s) => reasonLabel(s.reason) },
              { label: "Desde", w: "100px", render: (s) => fmtDay(s.created_at) },
              {
                label: "Ação", w: "1.6fr", card: "actions", render: (s) => {
                  const name = personName(s.person);
                  if (s.reason === "resolved_problem") return (
                    <div style={row}>
                      {canConfirm && <button type="button" style={buttonStyle} aria-label={`Encerrar inscrição de ${name}`} onClick={() => act("end", s)}>Encerrar inscrição</button>}
                      <button type="button" style={secondaryButtonStyle} aria-label={`Manter inscrição de ${name}`} onClick={() => act("dismiss", s)}>Manter</button>
                    </div>
                  );
                  return (
                    <div style={row}>
                      {canConfirm && <button type="button" style={buttonStyle} aria-label={`Aceitar sugestão de ${name}`} onClick={() => act("accept", s)}>Aceitar</button>}
                      <button type="button" style={secondaryButtonStyle} aria-label={`Recusar sugestão de ${name}`} onClick={() => act("dismiss", s)}>Recusar</button>
                    </div>
                  );
                }
              }
            ]}
            rows={rows} rowKey={(s) => s.id} empty="nenhuma sugestão a confirmar"
          />
        )}
        {query.hasNextPage && (
          <div>
            <button type="button" style={secondaryButtonStyle} disabled={query.isFetchingNextPage} onClick={() => void query.fetchNextPage()}>Carregar mais</button>
          </div>
        )}
        {!canConfirm && <p style={muted}>Só médico ou enfermeiro confirma a inscrição; você pode recusar sugestões.</p>}
        {acting?.kind === "accept" && (
          <AcceptForm key={acting.suggestion.id} suggestion={acting.suggestion} teams={teams} teamId={teamId} today={today}
            onDone={finish} onCancel={() => setActing(null)} />
        )}
        {acting?.kind === "dismiss" && (
          <DismissForm key={acting.suggestion.id} suggestion={acting.suggestion} onDone={finish} onCancel={() => setActing(null)} />
        )}
        {acting?.kind === "end" && (
          <CloseForm key={acting.suggestion.id} suggestion={acting.suggestion} onDone={finish} onCancel={() => setActing(null)} />
        )}
      </div>
    </Panel>
  );
}

// Sugestão de encerramento (L5): aceitar encerra a inscrição com "alta da linha" (discharge).
function CloseForm({ suggestion, onDone, onCancel }: { suggestion: CareSuggestion; onDone: Done; onCancel(): void }) {
  const name = personName(suggestion.person);
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);

  async function submit() {
    if (running.current) return;
    running.current = true;
    setBusy(true);
    setError(null);
    try {
      await acceptCareSuggestion(suggestion.id, {});
      onDone(`Inscrição de ${name} encerrada com alta.`);
    } catch (err) {
      if (isStaleCare(err)) onDone(careLineError(err));
      else setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  return (
    <section aria-label={`Encerrar a inscrição de ${name}`} style={box}>
      <div style={form}>
        <p style={muted}>
          O problema que trouxe a pessoa para {suggestion.care_line.name} foi resolvido. Encerrar dá alta da linha; o histórico fica.
        </p>
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <button type="button" style={buttonStyle} disabled={busy} onClick={() => void submit()}>Encerrar com alta</button>
          <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}

function AcceptForm({ suggestion, teams, teamId, today, onDone, onCancel }: {
  suggestion: CareSuggestion; teams: CareTeam[]; teamId: string; today: string; onDone: Done; onCancel(): void;
}) {
  const name = personName(suggestion.person);
  // L5: a sugestão já nasce numa equipe; a tela começa por ela.
  const [ team, setTeam ] = useState(suggestion.team_id ?? teamId);
  const [ lmp, setLmp ] = useState("");
  const [ needAnchor, setNeedAnchor ] = useState(suggestion.care_line.requires_anchor === "lmp");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);

  async function submit() {
    if (needAnchor) {
      const problem = anchorProblem(lmp, today);
      if (problem) { setError(problem); return; }
    }
    if (running.current) return;
    running.current = true;
    setBusy(true);
    setError(null);
    try {
      await acceptCareSuggestion(suggestion.id, needAnchor ? { team_id: team, anchor_date: lmp } : { team_id: team });
      onDone(`Inscrição de ${name} em ${suggestion.care_line.name} confirmada.`);
    } catch (err) {
      if (errorCode(err) === "anchor_required") { setNeedAnchor(true); setError(careLineError(err)); }
      else if (isStaleCare(err)) onDone(careLineError(err));
      else setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  return (
    <section aria-label={`Aceitar ${name} em ${suggestion.care_line.name}`} style={box}>
      <div style={form}>
        <p style={muted}>A inscrição é um ato clínico: confirme depois de conferir o motivo da sugestão.</p>
        <label style={fieldLabel}>
          Equipe de referência
          <select value={team} style={inputStyle} onChange={(e) => { setError(null); setTeam(e.target.value); }}>
            {teams.map((t) => <option key={t.id} value={t.id}>{`${t.name} · INE ${t.ine}`}</option>)}
          </select>
        </label>
        {needAnchor && (
          <label style={fieldLabel}>
            DUM (data da última menstruação)
            <input type="date" value={lmp} style={inputStyle} onChange={(e) => { setError(null); setLmp(e.target.value); }} />
          </label>
        )}
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <button type="button" style={buttonStyle} disabled={busy} onClick={() => void submit()}>Confirmar inscrição</button>
          <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}

function DismissForm({ suggestion, onDone, onCancel }: { suggestion: CareSuggestion; onDone: Done; onCancel(): void }) {
  const name = personName(suggestion.person);
  const keep = suggestion.reason === "resolved_problem";
  const [ reason, setReason ] = useState("");
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);

  async function submit() {
    const problem = reasonProblem(reason);
    if (problem) { setError(problem); return; }
    if (running.current) return;
    running.current = true;
    setBusy(true);
    try {
      await dismissCareSuggestion(suggestion.id, reason.trim());
      onDone(keep ? `Inscrição de ${name} mantida.` : `Sugestão de ${name} recusada.`);
    } catch (err) {
      if (isStaleCare(err)) onDone(careLineError(err));
      else setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  return (
    <section aria-label={keep ? `Manter a inscrição de ${name}` : `Recusar a sugestão de ${name}`} style={box}>
      <div style={form}>
        <label style={fieldLabel}>
          {keep ? "Por que manter a inscrição" : "Motivo da recusa"}
          <textarea rows={2} value={reason} style={inputStyle} onChange={(e) => { setError(null); setReason(e.target.value); }} />
        </label>
        <p style={muted}>até 500 caracteres; fica no registro da decisão, o cidadão não vê</p>
        {error && <p role="alert" style={alert}>{error}</p>}
        <div style={row}>
          <button type="button" style={buttonStyle} disabled={busy} onClick={() => void submit()}>{keep ? "Manter inscrição" : "Recusar sugestão"}</button>
          <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={onCancel}>Cancelar</button>
        </div>
      </div>
    </section>
  );
}
```

Em `src/modules/CareLinesTeam.tsx`:

1. Troque `import { countOverdueEnrollments, getCareContext, listCareEnrollments } from "../lib/api";` por

```ts
import { countCareSuggestions, countOverdueEnrollments, getCareContext, listCareEnrollments } from "../lib/api";
```

troque

```ts
  CONTEXT_KEY, COUNT_QUERY, ENROLLMENTS_KEY, ENROLLMENT_FILTERS, NOMINAL_QUERY, OVERDUE_COUNT_KEY, STATIC_QUERY, careLineError, isStaleCare
```

por

```ts
  CONTEXT_KEY, COUNT_QUERY, ENROLLMENTS_KEY, ENROLLMENT_FILTERS, NOMINAL_QUERY, OVERDUE_COUNT_KEY, STATIC_QUERY, SUGGESTIONS_KEY,
  SUGGESTION_COUNT_KEY, careLineError, isStaleCare
```

e acrescente, depois de `import { EnrollmentDetail } from "./careLines/EnrollmentDetail";`,

```ts
import { SuggestionsPanel } from "./careLines/SuggestionsPanel";
```

2. Troque `export type TeamTab = "overdue" | "due_soon" | "all";` por

```ts
export type TeamTab = "overdue" | "due_soon" | "all" | "suggestions";
```

   E, logo depois de `  const teamId = teamChoice || teams[0]?.id || "";`, acrescente

```ts
  // D2: médico ou enfermeiro com vínculo na unidade desta equipe. Ausente
  // (api sem o campo) = a tela oferece e o api recusa com cbo_not_allowed.
  const canConfirm = teams.find((t) => t.id === teamId)?.can_confirm !== false;
```

3. Depois do bloco de `overdueCount`, acrescente

```ts
  const suggestionCount = useQuery({
    queryKey: [ SUGGESTION_COUNT_KEY, teamId ], queryFn: () => countCareSuggestions(teamId), enabled: !!teamId, ...COUNT_QUERY
  });
```

4. Na `useInfiniteQuery` das inscrições, troque

```ts
    queryFn: ({ pageParam }) => listCareEnrollments({ teamId, careLineId: lineId || undefined, status: tab, before: pageParam }),
```

por

```ts
    queryFn: ({ pageParam }) => listCareEnrollments({
      teamId, careLineId: lineId || undefined, status: tab === "suggestions" ? "all" : tab, before: pageParam
    }),
```

e `    enabled: !!teamId,` (o da lista, logo depois de `getNextPageParam`) por

```ts
    enabled: !!teamId && tab !== "suggestions",
```

5. Troque o corpo de `reload` por

```ts
  const reload = () => {
    void queryClient.resetQueries({ queryKey: [ ENROLLMENTS_KEY ] });
    void queryClient.resetQueries({ queryKey: [ SUGGESTIONS_KEY ] });
    void queryClient.invalidateQueries({ queryKey: [ OVERDUE_COUNT_KEY ] });
    void queryClient.invalidateQueries({ queryKey: [ SUGGESTION_COUNT_KEY ] });
  };
```

6. Troque `  void today; // usado nas Tasks 9 e 10` por nada (apague a linha).

7. Depois do `.map` de `ENROLLMENT_FILTERS` (antes do `</div>` do grupo "Lista"), acrescente

```tsx
        <button type="button" aria-pressed={tab === "suggestions"} style={tab === "suggestions" ? buttonStyle : secondaryButtonStyle}
          onClick={() => { setNotice(null); setTab("suggestions"); }}>
          {typeof suggestionCount.data === "number" ? `Sugestões (${suggestionCount.data})` : "Sugestões"}
        </button>
```

8. Troque `      <Panel title="Pessoas da equipe"` por

```tsx
      {tab === "suggestions" ? (
        <SuggestionsPanel teamId={teamId} careLineId={lineId} teams={teams} canConfirm={canConfirm} today={today}
          onChanged={changed} />
      ) : (
      <Panel title="Pessoas da equipe"
```

e o fim do componente

```tsx
      </Panel>
    </div>
  );
}
```

por

```tsx
      </Panel>
      )}
    </div>
  );
}
```

(é o único `</Panel>` seguido de `</div>` e `);` no arquivo).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/careLines src/modules/CareLinesTeam.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/SuggestionsPanel.tsx src/modules/careLines/SuggestionsPanel.test.tsx src/modules/CareLinesTeam.tsx
$G commit -m "feat: confirm or dismiss care line suggestions in the team panel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 10: Inscrição manual

**Files:**
- Create: `src/modules/careLines/ManualEnrollPanel.tsx`
- Modify: `src/modules/CareLinesTeam.tsx`
- Test: `src/modules/careLines/ManualEnrollPanel.test.tsx`

**Interfaces:**
- Consumes: `lookupCarePerson`, `enrollCare`, `errorCode`, `CarePersonLookup`, `CareLineRef`, `CareTeam` (Task 1); `anchorProblem`, `careLineError`, `personName` (Task 2); `isValidCpf`, `onlyDigits` (`src/lib/attendance.ts`); `TeamBody` (Tasks 6–9).
- Produces: `ManualEnrollPanel({ teamId, teams, lines, today, onEnrolled(message), onCancel })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/careLines/ManualEnrollPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), getCareContext: vi.fn(), listCareEnrollments: vi.fn(), countOverdueEnrollments: vi.fn(),
    countCareSuggestions: vi.fn(), lookupCarePerson: vi.fn(), enrollCare: vi.fn()
  };
});

import * as api from "../../lib/api";
import { CareLinesTeam } from "../CareLinesTeam";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY23, careContext, careUser, enrollment } from "../../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const BENEDITA = { patient_id: null, citizen_id: "ci3", display_name: "Benedita Ramos", cpf_masked: "***.982.247-**", verified: true };
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getCareContext, api.listCareEnrollments, api.countOverdueEnrollments,
    api.countCareSuggestions, api.lookupCarePerson, api.enrollCare ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser());
  mocked(api.getCareContext).mockResolvedValue(careContext());
  mocked(api.listCareEnrollments).mockResolvedValue({ enrollments: [ enrollment() ], next_before: null });
  mocked(api.countOverdueEnrollments).mockResolvedValue(12);
  mocked(api.countCareSuggestions).mockResolvedValue(0);
  mocked(api.lookupCarePerson).mockResolvedValue(BENEDITA);
  mocked(api.enrollCare).mockResolvedValue(enrollment({ id: "en9" }));
});

async function openPanel() {
  renderWithProviders(<CareLinesTeam today={TODAY23} />);
  fireEvent.click(await screen.findByRole("button", { name: "Inscrever pessoa" }));
  return screen.getByRole("region", { name: "Inscrever pessoa" });
}
async function find(panel: HTMLElement, cpf = "529.982.247-25") {
  fireEvent.change(within(panel).getByLabelText("CPF da pessoa"), { target: { value: cpf } });
  fireEvent.click(within(panel).getByRole("button", { name: "Buscar" }));
}

describe("ManualEnrollPanel", () => {
  it("inscreve cidadão validado que nunca consultou: CPF no corpo, citizen_id, linha e equipe", async () => {
    const panel = await openPanel();
    await find(panel);
    expect(await within(panel).findByText("Benedita Ramos · CPF ***.982.247-**")).not.toBeNull();
    expect(api.lookupCarePerson).toHaveBeenCalledWith("52998224725");
    expect((within(panel).getByLabelText("CPF da pessoa") as HTMLInputElement).value).toBe("");
    fireEvent.change(within(panel).getByLabelText("Linha"), { target: { value: "cl4" } });
    fireEvent.change(within(panel).getByLabelText("Equipe de referência"), { target: { value: "tm2" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Inscrever" }));
    await waitFor(() => expect(api.enrollCare).toHaveBeenCalledWith({ citizen_id: "ci3", care_line_id: "cl4", team_id: "tm2" }));
    expect((await screen.findByRole("status")).textContent).toBe("Inscrição de Benedita Ramos em Pessoa com diabetes confirmada.");
    await waitFor(() => expect(api.listCareEnrollments).toHaveBeenCalledTimes(2));
    expect(screen.queryByRole("region", { name: "Inscrever pessoa" })).toBeNull();
  });

  it("paciente vai por patient_id; gestante pede a DUM antes de enviar", async () => {
    mocked(api.lookupCarePerson).mockResolvedValue({ ...BENEDITA, patient_id: "pa5", display_name: "Ana Souza" });
    const panel = await openPanel();
    await find(panel);
    await within(panel).findByText("Ana Souza · CPF ***.982.247-**");
    fireEvent.change(within(panel).getByLabelText("Linha"), { target: { value: "cl3" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Inscrever" }));
    expect(within(panel).getByRole("alert").textContent).toBe("informe a DUM (data da última menstruação)");
    fireEvent.change(within(panel).getByLabelText("DUM (data da última menstruação)"), { target: { value: "2026-07-15" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Inscrever" }));
    await waitFor(() => expect(api.enrollCare).toHaveBeenCalledWith({ patient_id: "pa5", care_line_id: "cl3", team_id: "tm1", anchor_date: "2026-07-15" }));
  });

  it("cadastro não validado: avisa e não oferece Inscrever; 409 already_enrolled vira frase", async () => {
    mocked(api.lookupCarePerson).mockResolvedValueOnce({ ...BENEDITA, verified: false });
    const panel = await openPanel();
    await find(panel);
    expect(await within(panel).findByText("cadastro ainda não validado no posto — valide antes de inscrever")).not.toBeNull();
    expect(within(panel).queryByRole("button", { name: "Inscrever" })).toBeNull();
    mocked(api.enrollCare).mockRejectedValueOnce(new api.ApiError(409, { error: "already_enrolled" }, "x"));
    await find(panel);
    await within(panel).findByRole("button", { name: "Inscrever" });
    fireEvent.change(within(panel).getByLabelText("Linha"), { target: { value: "cl4" } });
    fireEvent.click(within(panel).getByRole("button", { name: "Inscrever" }));
    expect((await within(panel).findByRole("alert")).textContent).toBe("a pessoa já está inscrita nesta linha");
  });

  it("CPF inválido não busca; sem can_confirm não há Inscrever pessoa", async () => {
    const panel = await openPanel();
    await find(panel, "529.982.247-24");
    expect(within(panel).getByRole("alert").textContent).toBe("CPF inválido");
    expect(api.lookupCarePerson).not.toHaveBeenCalled();
    cleanup();
    mocked(api.getCareContext).mockResolvedValue(careContext({ canConfirm: false }));
    renderWithProviders(<CareLinesTeam today={TODAY23} />);
    await screen.findByText("Maria das Dores · 58 anos");
    expect(screen.queryByRole("button", { name: "Inscrever pessoa" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/careLines/ManualEnrollPanel.test.tsx`
Expected: FAIL — não há o botão "Inscrever pessoa".

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/ManualEnrollPanel.tsx
// Inscrição manual (contrato §4; Divergências D6 e D9): o médico ou o
// enfermeiro acha a pessoa pelo CPF (no corpo de um POST, nunca na URL),
// escolhe a linha e a equipe de referência e confirma. Só cadastro validado
// (409 citizen_not_verified); quem nunca consultou vira paciente no api
// (Patients::Resolve). A linha da gestante pede a DUM. A tela guarda só ids,
// o nome de exibição e o CPF mascarado.
import { useRef, useState } from "react";
import { enrollCare, errorCode, lookupCarePerson, type CareLineRef, type CarePersonLookup, type CareTeam } from "../../lib/api";
import { isValidCpf, onlyDigits } from "../../lib/attendance";
import { anchorProblem, careLineError, personName } from "../../lib/careLines";
import { Tag } from "../../components/Tag";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alert, box, fieldLabel, form, muted, row } from "../exams/styles";

export function ManualEnrollPanel({ teamId, teams, lines, today, onEnrolled, onCancel }: {
  teamId: string; teams: CareTeam[]; lines: CareLineRef[]; today: string; onEnrolled(message: string): void; onCancel(): void;
}) {
  const [ cpf, setCpf ] = useState("");
  const [ person, setPerson ] = useState<CarePersonLookup | null>(null);
  const [ lineId, setLineId ] = useState("");
  const [ team, setTeam ] = useState(teamId);
  const [ lmp, setLmp ] = useState("");
  const [ forcedAnchor, setForcedAnchor ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ busy, setBusy ] = useState(false);
  const running = useRef(false);
  const line = lines.find((l) => l.id === lineId) ?? null;
  const needAnchor = forcedAnchor || line?.requires_anchor === "lmp";

  async function run(action: () => Promise<void>) {
    if (running.current) return;
    running.current = true;
    setBusy(true);
    setError(null);
    try {
      await action();
    } catch (err) {
      if (errorCode(err) === "anchor_required") setForcedAnchor(true);
      setError(careLineError(err));
    } finally {
      running.current = false;
      setBusy(false);
    }
  }

  function find() {
    if (!isValidCpf(cpf)) { setError("CPF inválido"); return; }
    void run(async () => {
      const found = await lookupCarePerson(onlyDigits(cpf));
      setCpf("");
      setPerson(found);
    });
  }

  function enroll() {
    if (!person) return;
    if (!line) { setError("escolha a linha"); return; }
    if (needAnchor) {
      const problem = anchorProblem(lmp, today);
      if (problem) { setError(problem); return; }
    }
    const who = person.patient_id ? { patient_id: person.patient_id } : { citizen_id: person.citizen_id ?? "" };
    void run(async () => {
      await enrollCare({ ...who, care_line_id: line.id, team_id: team, ...(needAnchor ? { anchor_date: lmp } : {}) });
      onEnrolled(`Inscrição de ${personName(person)} em ${line.name} confirmada.`);
    });
  }

  return (
    <section aria-label="Inscrever pessoa" style={box}>
      <div style={form}>
        <div style={{ ...row, alignItems: "flex-end" }}>
          <label style={fieldLabel}>
            CPF da pessoa
            <input value={cpf} inputMode="numeric" autoComplete="off" style={inputStyle} onChange={(e) => { setError(null); setCpf(e.target.value); }} />
          </label>
          <button type="button" style={secondaryButtonStyle} disabled={busy} onClick={find}>Buscar</button>
        </div>
        {person && (
          <>
            <div style={row}>
              <span>{`${personName(person)} · CPF ${person.cpf_masked}`}</span>
              <Tag tone={person.verified ? "ok" : "warn"}>{person.verified ? "cadastro validado" : "não validado"}</Tag>
            </div>
            {!person.verified && <p style={muted}>cadastro ainda não validado no posto — valide antes de inscrever</p>}
            {person.verified && (
              <>
                <div style={row}>
                  <label style={fieldLabel}>
                    Linha
                    <select value={lineId} style={inputStyle} onChange={(e) => { setError(null); setForcedAnchor(false); setLineId(e.target.value); }}>
                      <option value="">escolha</option>
                      {lines.map((l) => <option key={l.id} value={l.id}>{l.name}</option>)}
                    </select>
                  </label>
                  <label style={fieldLabel}>
                    Equipe de referência
                    <select value={team} style={inputStyle} onChange={(e) => { setError(null); setTeam(e.target.value); }}>
                      {teams.map((t) => <option key={t.id} value={t.id}>{`${t.name} · INE ${t.ine}`}</option>)}
                    </select>
                  </label>
                  {needAnchor && (
                    <label style={fieldLabel}>
                      DUM (data da última menstruação)
                      <input type="date" value={lmp} style={inputStyle} onChange={(e) => { setError(null); setLmp(e.target.value); }} />
                    </label>
                  )}
                </div>
                <div><button type="button" style={buttonStyle} disabled={busy} onClick={enroll}>Inscrever</button></div>
              </>
            )}
          </>
        )}
        {error && <p role="alert" style={alert}>{error}</p>}
        <div><button type="button" style={secondaryButtonStyle} onClick={onCancel}>Fechar</button></div>
      </div>
    </section>
  );
}
```

Em `src/modules/CareLinesTeam.tsx`:

1. Depois de `import { SuggestionsPanel } from "./careLines/SuggestionsPanel";`, acrescente

```ts
import { ManualEnrollPanel } from "./careLines/ManualEnrollPanel";
```

2. Depois de `  const [ openId, setOpenId ] = useState<string | null>(null);`, acrescente

```ts
  const [ enrolling, setEnrolling ] = useState(false);
```

3. Depois do botão "Sugestões" (fim do grupo "Lista"), acrescente

```tsx
        {canConfirm && (
          <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setEnrolling(true); }}>Inscrever pessoa</button>
        )}
```

4. Depois de `      {notice && <p role="status" style={statusStyle}>{notice}</p>}`, acrescente

```tsx
      {enrolling && (
        <ManualEnrollPanel teamId={teamId} teams={teams} lines={context.data?.lines ?? []} today={today}
          onEnrolled={(message) => { setEnrolling(false); changed(message); }} onCancel={() => setEnrolling(false)} />
      )}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/careLines src/modules/CareLinesTeam.test.tsx && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/ManualEnrollPanel.tsx src/modules/careLines/ManualEnrollPanel.test.tsx src/modules/CareLinesTeam.tsx
$G commit -m "feat: enroll a person in a care line by hand

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 11: Prévia dos indicadores C2–C7 por equipe

**Files:**
- Create: `src/modules/careLines/PreviewCard.tsx`, `src/modules/CareLinesPreview.tsx`
- Modify: `src/App.tsx`
- Test: `src/modules/CareLinesPreview.test.tsx`

**Interfaces:**
- Consumes: `getCareContext`, `getCarePreview`, `CarePreview`, `CarePreviewPractice` (Task 1); `CONTEXT_KEY`, `PREVIEW_KEY`, `STATIC_QUERY`, `ESTIMATE_NOTICE`, `INDICATORS`, `careLineError`, `conceptLabel`, `decimalBr`, `excludedLabel`, `indicatorLabel` (Task 2); `CareLinesGate` (Task 3).
- Produces: `CareLinesPreview()`; `PreviewCard({ indicator, data, error, pending })`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/CareLinesPreview.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getCareContext: vi.fn(), getCarePreview: vi.fn() };
});

import * as api from "../lib/api";
import { CareLinesPreview } from "./CareLinesPreview";
import { CARE_NO_ROLE, ESTIMATE_NOTICE } from "../lib/careLines";
import { renderWithProviders } from "../test/campaignFixtures";
import { careContext, careUser, preview } from "../test/careLineFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getCareContext, api.getCarePreview ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "analyst" ]));
  mocked(api.getCareContext).mockResolvedValue(careContext({ canConfirm: false }));
  mocked(api.getCarePreview).mockImplementation((_team: string, indicator: string) => Promise.resolve(preview(indicator)));
});

describe("CareLinesPreview", () => {
  it("o aviso de estimativa aparece sempre; pontuação com vírgula; fora da conta listado", async () => {
    mocked(api.getCarePreview).mockImplementation((_team: string, indicator: string) =>
      indicator === "C5" ? Promise.reject(new api.ApiError(500, "", "x")) : Promise.resolve(preview(indicator)));
    renderWithProviders(<CareLinesPreview />);
    expect((await screen.findByRole("note", { name: "aviso de estimativa" })).textContent).toBe(ESTIMATE_NOTICE);
    expect(api.getCarePreview).not.toHaveBeenCalled();
    fireEvent.click(await screen.findByRole("button", { name: "Calcular prévia" }));
    await waitFor(() => expect(api.getCarePreview).toHaveBeenCalledTimes(6));
    expect(mocked(api.getCarePreview).mock.calls.map((c) => c[1])).toEqual([ "C2", "C3", "C4", "C5", "C6", "C7" ]);
    expect(mocked(api.getCarePreview).mock.calls.every((c) => c[0] === "tm1")).toBe(true);
    const c4 = await screen.findByRole("region", { name: "C4 — Pessoa com diabetes" });
    expect(await within(c4).findByText("37,5 de 100")).not.toBeNull();
    expect(within(c4).getByText("suficiente")).not.toBeNull();
    expect(within(c4).getByText("41")).not.toBeNull();
    expect(within(c4).getByText("30 de 41")).not.toBeNull();
    expect(within(c4).getByText("25")).not.toBeNull();
    expect(within(c4).getByText("fora da conta — peso redistribuído")).not.toBeNull();
    expect(within(c4).getByText("Fora da conta: visita do ACS, vacina, saúde bucal")).not.toBeNull();
    const c5 = screen.getByRole("region", { name: "C5 — Pessoa com hipertensão" });
    expect((await within(c5).findByRole("alert")).textContent).toBe("não foi possível concluir (500) — tente de novo");
    expect(screen.getByRole("note", { name: "aviso de estimativa" })).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/37\.5|25\.0/);
  });

  it("troca de equipe e calcula de novo; nada é lido sem o clique", async () => {
    renderWithProviders(<CareLinesPreview />);
    fireEvent.change(await screen.findByLabelText("Equipe"), { target: { value: "tm2" } });
    expect(api.getCarePreview).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Calcular prévia" }));
    await waitFor(() => expect(api.getCarePreview).toHaveBeenCalledTimes(6));
    expect(mocked(api.getCarePreview).mock.calls.every((c) => c[0] === "tm2")).toBe(true);
  });

  it("equipe sem elegíveis: diz que não há população elegível", async () => {
    mocked(api.getCarePreview).mockImplementation((_team: string, indicator: string) =>
      Promise.resolve(preview(indicator, { denominator: 0, score: null, concept: null, practices: [] })));
    renderWithProviders(<CareLinesPreview />);
    fireEvent.click(await screen.findByRole("button", { name: "Calcular prévia" }));
    const c2 = await screen.findByRole("region", { name: "C2 — Cuidado da criança até 2 anos" });
    expect(await within(c2).findByText("nenhuma pessoa elegível nesta equipe")).not.toBeNull();
  });

  it("profissional não vê a prévia; admin vê", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "health_professional" ]));
    renderWithProviders(<CareLinesPreview />);
    expect(await screen.findByText(CARE_NO_ROLE)).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(careUser([ "municipal_admin" ]));
    renderWithProviders(<CareLinesPreview />);
    expect(await screen.findByRole("button", { name: "Calcular prévia" })).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd $W && npx vitest run src/modules/CareLinesPreview.test.tsx`
Expected: FAIL — `./CareLinesPreview` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/careLines/PreviewCard.tsx
// Um indicador da prévia (contrato §6): pontuação estimada sobre a população
// elegível da equipe, as boas práticas com peso e quantos atingiram, as não
// rastreadas marcadas (peso redistribuído) e o que fica fora da conta. Nunca
// "valor oficial": o rótulo de estimativa é da tela (CareLinesPreview).
import type { CarePreview, CarePreviewPractice } from "../../lib/api";
import { careLineError, conceptLabel, decimalBr, excludedLabel, indicatorLabel } from "../../lib/careLines";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { KeyValue } from "../../components/KeyValue";
import { EmptyState } from "../../components/EmptyState";
import { alert, grid, muted, stack } from "../exams/styles";

export function PreviewCard({ indicator, data, error, pending }: {
  indicator: string; data: CarePreview | undefined; error: unknown; pending: boolean;
}) {
  return (
    <Panel title={`${indicator} — ${indicatorLabel(indicator)}`} sub="estimativa">
      <div style={{ ...stack, padding: 16 }}>
        {pending && !error && <p className="mono" style={muted}>calculando…</p>}
        {error ? <p role="alert" style={alert}>{careLineError(error)}</p> : null}
        {data && (data.denominator === 0 ? <EmptyState title="nenhuma pessoa elegível nesta equipe" /> : (
          <>
            <div style={grid}>
              <KeyValue k="Pontuação estimada" v={`${decimalBr(data.score)} de ${decimalBr(data.max_score)}`} />
              <KeyValue k="Conceito estimado" v={conceptLabel(data.concept)} mono={false} />
              <KeyValue k="População elegível" v={String(data.denominator)} />
              {data.template_version !== null && <KeyValue k="Versão do catálogo" v={`v${data.template_version}`} />}
            </div>
            <DataTable<CarePreviewPractice>
              cols={[
                { label: "Boa prática", w: "2fr", card: "title", render: (p) => p.label ?? p.key },
                { label: "Pontos", w: "80px", align: "right", render: (p) => (typeof p.points === "number" ? String(p.points) : "—") },
                { label: "Peso na prévia", w: "110px", align: "right", render: (p) => decimalBr(p.weight) },
                { label: "Atingiram", w: "110px", align: "right",
                  render: (p) => (typeof p.denominator === "number" ? `${p.achieved} de ${p.denominator}` : String(p.achieved)) },
                { label: "Na conta", w: "1.4fr", render: (p) => (p.tracked ? "sim" : "fora da conta — peso redistribuído") }
              ]}
              rows={data.practices} rowKey={(p) => p.key}
            />
            {data.excluded.length > 0 && <p style={muted}>{`Fora da conta: ${data.excluded.map(excludedLabel).join(", ")}`}</p>}
          </>
        ))}
      </div>
    </Panel>
  );
}
```

```tsx
// src/modules/CareLinesPreview.tsx
// Prévia de C2–C7 por equipe (contrato §6; spec §6 "Gestão"; ADR 0040): para
// municipal_admin e analyst, agregada, sem nome. Denominador = população
// elegível da equipe (pacientes e cidadãos validados, inscritos ou não);
// pontuação pelas boas práticas rastreadas, com visita do ACS e vacina fora e
// os pesos redistribuídos. SEMPRE com o aviso de estimativa — nunca valor
// oficial. Lê só no clique de "Calcular prévia" (nada de polling).
import { useState, type CSSProperties } from "react";
import { useQueries, useQuery } from "@tanstack/react-query";
import { getCareContext, getCarePreview } from "../lib/api";
import { CONTEXT_KEY, ESTIMATE_NOTICE, INDICATORS, PREVIEW_KEY, STATIC_QUERY, careLineError } from "../lib/careLines";
import { Panel } from "../components/Panel";
import { EmptyState } from "../components/EmptyState";
import { buttonStyle, inputStyle } from "../components/formStyles";
import { alert, fieldLabel, row, stack } from "./exams/styles";
import { CareLinesGate } from "./careLines/CareLinesGate";
import { PreviewCard } from "./careLines/PreviewCard";

export function CareLinesPreview() {
  return (
    <CareLinesGate title="Prévia dos indicadores" sub="C2 a C7 por equipe · estimativa, não o valor oficial" allow={(r) => r.admin || r.analyst}>
      {() => <PreviewBody />}
    </CareLinesGate>
  );
}

function PreviewBody() {
  const context = useQuery({ queryKey: [ CONTEXT_KEY ], queryFn: getCareContext, ...STATIC_QUERY });
  const [ teamChoice, setTeamChoice ] = useState("");
  const [ asked, setAsked ] = useState<string | null>(null);
  const teams = context.data?.teams ?? [];
  const teamId = teamChoice || teams[0]?.id || "";
  const results = useQueries({
    queries: INDICATORS.map((ind) => ({
      queryKey: [ PREVIEW_KEY, asked, ind.key ], queryFn: () => getCarePreview(asked ?? "", ind.key), enabled: asked !== null, ...STATIC_QUERY
    }))
  });

  return (
    <div style={stack}>
      <p role="note" aria-label="aviso de estimativa" style={noteStyle}>{ESTIMATE_NOTICE}</p>
      {context.isError && <p role="alert" style={alert}>{careLineError(context.error)}</p>}
      {context.data && teams.length === 0 && <EmptyState title="nenhuma equipe cadastrada na cidade" />}
      {teams.length > 0 && (
        <Panel title="Equipe">
          <div style={{ ...row, alignItems: "flex-end", padding: 16 }}>
            <label style={fieldLabel}>
              Equipe
              <select value={teamId} style={inputStyle} onChange={(e) => setTeamChoice(e.target.value)}>
                {teams.map((t) => <option key={t.id} value={t.id}>{`${t.name} · INE ${t.ine} · ${t.unit.name}`}</option>)}
              </select>
            </label>
            <button type="button" style={buttonStyle} onClick={() => setAsked(teamId)}>Calcular prévia</button>
          </div>
        </Panel>
      )}
      {asked && (
        <div style={cards}>
          {INDICATORS.map((ind, i) => (
            <PreviewCard key={ind.key} indicator={ind.key} data={results[i].data} error={results[i].error} pending={results[i].isPending} />
          ))}
        </div>
      )}
    </div>
  );
}

const noteStyle: CSSProperties = {
  margin: 0, padding: 12, borderRadius: 8, border: "1px solid var(--warn)", background: "var(--sunken)", fontSize: 12.5, fontWeight: 600
};
const cards: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(min(100%, 420px), 1fr))", gap: 16 };
```

Em `src/App.tsx`, depois de `import { CareLinesTeam } from "./modules/CareLinesTeam";`, acrescente

```ts
import { CareLinesPreview } from "./modules/CareLinesPreview";
```

e, antes de `    case "care-lines-admin": …`, acrescente

```tsx
    case "care-lines-preview": return <CareLinesPreview />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd $W && npx vitest run src/modules/CareLinesPreview.test.tsx && npx tsc --noEmit`
Expected: PASS. (Se o `literals.test.ts` ou o `css-guards.test.ts` reclamar do `minmax(min(100%, 420px), 1fr)`, troque pelo mesmo padrão de grade usado em `src/modules/analytics` — é forma, não cor.)

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/careLines/PreviewCard.tsx src/modules/CareLinesPreview.tsx src/modules/CareLinesPreview.test.tsx src/App.tsx
$G commit -m "feat: show the C2 to C7 indicator preview per team as an estimate

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Suíte, build, revisão e prova no navegador (api real na 3045)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–11 na branch `feat/mod-23-care-lines`; o api do 23 na **3045** com a semente do 23.
- Produces: branch pronta para merge (depois do merge do api do 23); evidências em `evidencias/2026-10-22-modulo-23/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd $W && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **11 arquivos de teste novos** (`api.careLines`, `careLines`, `modules.careLines`, `CareLinesAdmin`, `CustomLineForm`, `CareLinesTeam`, `EnrollmentDetail`, `EnrollmentActions`, `SuggestionsPanel`, `ManualEnrollPanel`, `CareLinesPreview`) e dois arquivos de teste existentes com casos novos ou ajustados (`features.test.ts`, `modules.test.ts`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/SegmentedControl.tsx src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css src/modules/attendance package.json package-lock.json
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css`, tela do atendimento (o `BookPanel` é usado sem mudar) ou dependência mudados.

- [ ] **Step 2: Nada vaza e nada que grava trilha relê sozinho**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage" src/lib/careLines.ts src/modules/careLines src/modules/CareLines*.tsx | grep -v "\.test\."
grep -rn "refetchInterval\|refetchOnWindowFocus: true" src/modules/careLines src/modules/CareLines*.tsx | grep -v "\.test\."
grep -rn "COUNT_QUERY" src/modules/careLines src/modules/CareLines*.tsx | grep -v "\.test\."
sed -n '/─── Linhas de cuidado (módulo 23/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [A-Za-z_, :]* })'
grep -rn "maxLength" src/modules/careLines src/modules/CareLines*.tsx
```

Expected: o primeiro, o segundo e o último sem saída; o terceiro só nas duas consultas de contagem de `CareLinesTeam.tsx`; o quarto só com `care_line_id`, `status`, `before`, `limit` e os `URLSearchParams` de `{ team_id: teamId }` e `{ team_id: teamId, indicator }` — nenhum `cpf`, `name`, `note` ou `reason`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§2–§8), o ADR 0040, o contrato (§1–§6, §9, §10) e as Divergências deste plano. Pontos de atenção:
- sem `care_lines`, o menu, as telas e o atendimento são exatamente os de hoje (testes antigos verdes; só as contagens de `modules.test.ts` mudaram);
- nada inscreve sem clique de profissional; "Aceitar" e "Inscrever pessoa" só com `can_confirm`; a DUM obrigatória na linha que pede âncora; 422 `anchor_required` abre o campo;
- `untracked` em tom neutro, nunca vencida, e fora da pior situação calculada;
- lista, detalhe, sugestões e busca de pessoa com `NOMINAL_QUERY`; só os dois contadores com `COUNT_QUERY`; a prévia só no clique;
- a prévia sempre com o aviso de estimativa; vírgula decimal;
- adotar, versão nova, ativar/desativar e linha própria pelo `SensitiveAction` com step-up; textos recusados, nunca cortados;
- CPF só no corpo; nada em console/storage; nota e motivo só no corpo;
- o `BookPanel` do módulo 17 usado sem mudança; o atalho só chama o api no clique;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 23 na **3045** (plano do api: servidor do worktree `apps/api/.claude/mod23` no `api-dev`, migrações de cidade e plataforma a partir de `20261022900001` aplicadas no banco de dev **com autorização do usuário**, a semente do 23 e o job diário de sugestões rodado uma vez; `care_lines` em `Platform::Features::CATALOG`). Ligue em Curitiba, pelo maintenance, `clinical_record` e `care_lines` (e `patient_timeline`, se a prova do api pedir). O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo: o do worktree é do macOS e não roda no Alpine):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod23 --network rota-saude_default -p 5198:5198 \
  -v $R/.worktrees/dashboard-mod23:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3045 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5198 --host 0.0.0.0"
docker logs -f dashboard-mod23   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 23 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5198/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot (salve em `evidencias/2026-10-22-modulo-23/dashboard/`):
1. **Admin** — Linhas de cuidado → **Linhas da cidade**: as seis do catálogo; adotar "Pessoa com diabetes" e "Gestante e puérpera" (step-up); criar a linha própria "Saúde mental" com a condição declarada `saúde mental` e uma consulta a cada 90 dias (step-up); um CID mal formado é recusado antes de enviar.
2. **Enfermeira da UBS Vila Verde** — **Acompanhamento da equipe**: a aba **Sugestões (N)** mostra o paciente com problema ativo de diabetes (motivo "problema ativo na lista de problemas", sem texto clínico); **Aceitar** escolhendo a equipe → "Inscrição de … confirmada." e o contador cai.
3. **Diabético da semente** — na **Busca ativa**: consulta "em dia" e hemoglobina glicada "vencida"; a visita do ACS "não acompanhado pelo sistema", em cinza; a pessoa aparece na busca ativa por causa da HbA1c.
4. **Contato e agendamento** — **Abrir** → **Registrar contato** (telefone, não atendeu) → aparece no histórico; **Agendar** → o painel de marcação do módulo 17 com o pedido da pessoa na UBS → marcar uma vaga → o contato abre com "agendou"; salvar.
5. **Gestante** — aceitar a sugestão da gestante sem DUM é recusado; DUM no futuro é recusada; com a DUM de verdade, a inscrição mostra "DUM" e as consultas de pré-natal com a contagem "x de y esperadas até agora".
6. **Técnica de enfermagem** (CBO 322245) — vê as sugestões e **não** vê "Aceitar" nem "Inscrever pessoa"; recusar exige motivo.
7. **Inscrição manual** — pelo CPF do cidadão validado de 62 anos que nunca consultou: **Inscrever pessoa** → linha "Pessoa idosa" → confirmada (o paciente nasce no api); um CPF de par só declarado mostra "cadastro ainda não validado no posto" sem o botão.
8. **Encerrar** — numa inscrição, **Encerrar inscrição** com "mudou de território" → some da lista; "outro motivo" sem nota é recusado.
9. **Prévia** — como admin (e como analyst): **Prévia dos indicadores** → equipe → **Calcular prévia**: C2 a C7 com o aviso de estimativa no topo, pontuação com vírgula, população elegível e "Fora da conta: visita do ACS, vacina".
10. **Sem polling** — com a aba de rede aberta no Acompanhamento por 3 minutos, nenhuma chamada a `/care_lines/enrollments` nem `/care_lines/suggestions` sem ação; só `/count` a cada minuto.
11. **Telefone** (largura 375): a busca ativa, as sugestões e a prévia viram cartões, sem rolagem lateral.
12. No maintenance, desligar `care_lines` e recarregar: o grupo "Linhas de cuidado" some; religar devolve.

Depois: `docker rm -f dashboard-mod23`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

Para cada card: F-23.2, F-23.3, F-23.5 e F-23.6 vão a **Done** se a parte do api já está Done (o dashboard é o último app deles); F-23.1 é de api e maintenance (não muda aqui); F-23.4 é só do api; F-23.7 e F-23.8 são de api e wpda. Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified**. Use o mesmo `gh project item-edit` da Task 0, com o id da opção Done `98236657` (a de Verified, da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 23, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: api → dashboard e wpda → maintenance → catálogo (contrato §10). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-23-care-lines
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod23
/opt/homebrew/bin/git branch -d feat/mod-23-care-lines
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (D1–D12), alinhada ao plano do api (`2026-10-22-module-23-care-lines-api.md`, L1–L14). **Adotadas** = o plano do api já fixou e este plano usa a forma dele; **abertas** = o plano do api não tem e a tela precisa.

1. **D1 — Forma da `eligibility` e da boa prática. Adotada (L1, L2).** A tela usa `{ problems: { ciap2, cid10 }, exclude_problems?, age_min_years?, age_max_years?, age_min_days?, age_max_days?, sex?, gender_identities_*?, declared?, requires_anchor? }` e as práticas com `citizen_label`, `window_start_days`, `points`, `code_filter`, `applies_to`, `all_of`. A linha própria manda só `problems`, `age_min_years`, `age_max_years`, `sex`, `declared` (a condição dela) e `requires_anchor: null`, e práticas sem os campos avançados.
2. **D2 — Quem confirma. Aberta.** O plano do api trouxe `GET /care_lines/teams` e `GET /care_lines/lines` (L10), que a tela lê juntos, mas não diz à tela se a pessoa pode confirmar (a sessão não traz CBO). Proposta: cada equipe de `GET /care_lines/teams` ganha `can_confirm` (vínculo com CBO 225*, 2235* — e 2231*, L3 — na unidade dela; o mesmo `Access.confirmer?` do api). Sem o campo, a tela oferece "Aceitar" e "Inscrever pessoa" e trata o 403 `cbo_not_allowed` — mas a técnica de enfermagem veria um botão que sempre recusa (Review Focus 2).
3. **D3 — Âncora na linha. Adotada (L10):** `requires_anchor: "lmp"` em `GET /care_lines/lines` e no `care_line` da sugestão e da inscrição; a janela da DUM é a do api (hoje − 336 a hoje, `invalid_anchor`, L4).
4. **D4 — Sugestão de encerramento. Adotada (L5):** `enrollment_id` e `team_id` na sugestão; "Encerrar" aceita a sugestão com `{}` e o api encerra com `discharge`.
5. **D5 — Contatos no detalhe. Adotada (L12):** `contacts: [ { id, channel, outcome, note, recorded_by: { id, name }, recorded_at } ]` só no detalhe; `last_contact` sem nota; `ended_reason`/`ended_at` na inscrição encerrada.
6. **D6 — Pessoa pelo CPF. Aberta.** A inscrição manual (§4) recebe `patient_id | citizen_id`, mas nem o contrato nem o plano do api têm como achá-los. Proposta: `POST /care_lines/person_lookup { "cpf" }` → `{ "person": { "patient_id"|null, "citizen_id"|null, "display_name"|null, "cpf_masked", "verified" } }` (profissional com vínculo; trilha; 404 `person_not_found`; 422 `invalid_cpf`); CPF só no corpo. Sem ela, a Task 10 fica sem rota e a inscrição manual não sai na prova.
7. **D7 — Atalho para agendar. Aberta.** O §5 diz "o dashboard abre o pedido de agendamento do módulo 17 com o paciente preenchido (rota existente)", mas no módulo 17 o pedido só nasce do desfecho do atendimento ou da triagem (`appointment_requests.root_triage` é obrigatório) e a marcação (`BookPanel`) parte de um pedido; o plano do api não tem rota para isso. Proposta: `POST /care_lines/enrollments/:id/appointment_request` (corpo `{}`) → `{ "request": <RequestRow do módulo 17>, "unit": <HealthUnit> }`: devolve o pedido aberto da pessoa na unidade da equipe ou cria um (`kind: "return"` ou um `kind` novo), sem trilha extra; 422 `no_open_slot_unit` sem agenda na unidade. Sem ela, a Task 8 mostra a frase da recusa e a prova 4 fica sem o agendamento.
8. **D8 — Motivo da recusa. Adotada (limites do plano do api):** 1 a 500 caracteres, `reason_required`/`too_long` com `field`; recusado, nunca cortado.
9. **D9 — Quem inscreve à mão. Adotada** (plano do api, "Valores fixados" 3: inscrever à mão exige o mesmo CBO de confirmar).
10. **D10 — Versão nova do catálogo. Aberta em parte.** O `PATCH { template_version }` está no plano do api (Desvio 12). Falta à tela saber que há versão nova sem cruzar com `/admin/templates`: proposta de `latest_template_version` em `<care_line>`; sem o campo, a tela cruza com o catálogo (já implementado em `newerVersion`).
11. **D11 — Prévia legível. Adotada (L8):** `label`, `points`, `denominator` por prática; `score`/`weight` em texto com ponto (a tela mostra com vírgula); `concept` e `template_version`; `excluded` com as famílias (`acs_visit`, `vaccine`, `oral_health`).
12. **D12 — Recusas com código. Adotada (L4):** `out_of_scope`, `invalid_transition`, `invalid_anchor`, `line_inactive`, `immutable_rules`, `duplicate_declared_condition` etc. A tela relê a lista em `not_found`, `invalid_transition` e `invalid_cursor`.

Também usadas sem mudança de tela: **L6** (página curta com `next_before` — Task 6 carrega sob demanda), **L7** (trilha por pessoa mostrada — o dashboard não muda), **L13** (eventos — só api/maintenance).

## Self-review

- **Cobertura (spec §2, §3, §5–§8; contrato §1–§6, §9):** interruptor `care_lines` — Tasks 1 (rótulo) e 3 (menu e portão); adoção do catálogo-base, versão nova, ativar/desativar — Task 4; linha própria com a mesma forma e a condição declarada que a sugere — Task 5; painel nominal por equipe e linha, situação por boa prática, busca ativa, `untracked` neutro — Task 6; detalhe e registro de contato — Task 7; atalho para o agendamento do módulo 17 e encerramento com motivo — Task 8; sugestões a confirmar (aceitar com equipe, só médico ou enfermeiro, DUM obrigatória na gestante, recusar com motivo, sugestão de encerramento) — Task 9; inscrição manual (cidadão validado que nunca consultou incluído) — Task 10; prévia C2–C7 com o aviso de estimativa — Task 11; contadores só pelas rotas `/count` — Tasks 6 e 9; nenhum polling em rota com trilha — Tasks 2, 6 e o Step 2 da Task 12; prova no navegador — Task 12. F-23.4 (avaliação das boas práticas) é do api; aqui só aparece. F-23.7 e F-23.8 são do wpda.
- **Placeholders:** nenhum código faltando. Dependências do outro plano estão marcadas: as contas e dados da semente, o nome do container do api (Task 12) e as formas propostas nas Divergências.
- **Tipos e nomes:** `CareEnrollment`, `CareSuggestion`, `CareContext`, `CareLine`, `CareLineTemplate`, `CarePreview`, `CarePersonLookup`, `CareAppointmentTarget` (Task 1) são os das peças; `NOMINAL_QUERY`/`COUNT_QUERY`/`STATIC_QUERY`, `practiceText`, `personStatus`, `anchorProblem(lmp, today)`, `reasonProblem`, `contactProblem`, `endProblem`, `customLineProblem`/`customLineBody`/`draftOfLine`, `careLineError`, `isStaleCare` (Task 2) batem com os usos; `TeamTab` cresce de três para quatro valores na Task 9, com a troca escrita; `EnrollmentDetail.onChanged(message, close?)` (Task 7) é o que `TeamBody` passa; `DetailAction` cresce na Task 8; a sugestão de encerramento (L5) usa `acceptCareSuggestion(id, {})` no `CloseForm` da Task 9; `canConfirm` (D2) é derivado da equipe na Task 9 e usado nas Tasks 9 e 10; o `void today; void changed;` da Task 6 sai nas Tasks 7 e 9.
- **Review Focus:** 1 — Task 9 ("gestante: sem DUM não envia; DUM no futuro recusa; 422 anchor_required abre o campo"); 2 — Task 9 ("sem can_confirm: só Recusar; 403 cbo_not_allowed vira frase e nada muda"); 3 — Task 6 ("volta à aba e passa o tempo: a lista não relê; os contadores usam COUNT_QUERY"); 4 — Task 2 ("untracked nunca piora e tem frase própria") e Task 6 ("visita do ACS aparece como não acompanhada, em tom neutro"); 5 — Task 11 ("o aviso de estimativa aparece sempre; pontuação com vírgula; fora da conta listado").
