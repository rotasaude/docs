# Módulo 31 — Atenção psicossocial (CAPS) (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 31** (`feat/mod-31-psychosocial-care`, worktree `apps/api/.claude/mod31`, porta de dev **3053**) com as rotas do contrato §3–§11 (e as Divergências D1–D20 deste plano que o plano do api adotar), o interruptor `psychosocial_care` na sessão, os papéis `psychosocial_staff` e `psychosocial_lead`, o tipo de unidade `caps` e a semente do 31. Ele é **mergeado antes** deste (contrato §14: api → dashboard → maintenance → wpda). As Tasks 0–14 não dependem dele (dublês de `src/lib/api` com fixtures nas formas do contrato §2 e das Divergências); a prova da Task 15 depende.

**Goal:** Lado dashboard do 31: o **cadastro do CAPS** (tipo de unidade "CAPS" com modalidade e classificação 115, leitos, limites do acolhimento noturno e prazo do PTS), o grupo **"CAPS"** no shell com **Acolhimento** (fila de encaminhamentos e demanda espontânea, acolhimento com ou sem matrícula), **Usuários do CAPS** (matrícula, PTS com versões, ações com a crítica do SIGTAP, receita pelo fluxo do 19/19d com `psychosocial_action_id`, encerramento com destino, linha do tempo só do CAPS), **Hoje no CAPS**, **Grupos e oficinas** (grade semanal e presença usável a 375 px), **Leitos** (mapa, entrada, saída, limite e reavaliação), **Matriciamento**, **RAAS do mês** (pendências, situação de rua por registro, fechamento e reabertura com step-up, remessa com `raas_layout_unavailable`, fichas espelho em PDF) e o **Painel**; na tela do paciente da UBS, o **cartão do vínculo com o CAPS** com os casos compartilhados e o botão **"Encaminhar ao CAPS"**; o marcador do CAPS na linha do tempo geral.

**Architecture:** O cliente HTTP ganha a seção "Atenção psicossocial (módulo 31)" no fim de `src/lib/api.ts` (tipos = formas do contrato §2–§11 e das Divergências); `createUnit`/`updateUnit` ganham os campos do CAPS. As regras de tela ficam puras em `src/lib/psychosocial.ts` (portão, papéis, rótulos, validações, texto de **todos** os códigos de erro do contrato). As peças ficam em `src/modules/psychosocial/` e as telas no grupo novo **"CAPS"** (`modules.ts`, `catalog.ts`, `App.tsx`), depois de "TFD". O `PsychosocialGate` confere interruptor e papel e escolhe o CAPS de lotação (`listPsyUnits`); toda tela da equipe recebe o CAPS escolhido e remonta ao trocá-lo. **Lista nunca leva dado clínico** (CID, substância, evolução, PTS, motivo, texto compartilhado); o detalhe é lido só no clique, com `NOMINAL_QUERY` (sem cache), e o api grava a trilha. A receita reusa o `PrescriptionForm` do 19c/19d com outro `onSubmit`. O dashboard nunca decide o que o api garante: lotação, crítica do SIGTAP, CBO, mês fechado, leito sem sobreposição, limite de noites, continuidade, BPA-C, trilha, imutabilidade e retenção LGPD são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md` (§1 intenção e antiobjetivos; §2 decisões 1–24 e as do coordenador; §4 telas; §5 segurança e LGPD; §6 testes; §8 F-IDs; §9 riscos), `docs/adr/0049.md`, a pesquisa `docs/pesquisa/2026-11-04-atencao-psicossocial-31.md` e o contrato `docs/superpowers/plans/2026-11-04-module-31-psychosocial-care-contracts.md` (§1–§14). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D20); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptor `psychosocial_care` (requer `clinical_record`); papéis `psychosocial_staff` e `psychosocial_lead` (os dois privilegiados; `psychosocial_lead` implica `psychosocial_staff`); `health_units.kind = "caps"`, `caps_modality` `caps_i` | `caps_ii` | `caps_iii` | `caps_i_child` | `caps_ad` | `caps_ad_iii` | `caps_ad_iv`, `service_classification` de 3 dígitos; `NIGHT_MODALITIES = caps_iii, caps_ad_iii, caps_ad_iv` (só esses têm leito); instrumento `raas_psi` | `bpa_c` | `bpa_i`; intensidade `intensive` | `semi_intensive` | `non_intensive`; desfecho do acolhimento `enrolled` | `referred_elsewhere` | `no_follow_up`; prioridade `routine` | `priority`; grupo `group` | `workshop` | `assembly`; local `caps` | `territory`; RAAS `open` | `closed` | `exported`; destino `"00"` = permanência. **Código desconhecido aparece cru, nunca some.**
- **Interruptor** (decisão 24): sem `psychosocial_care` **ou** sem `clinical_record` na sessão, o grupo "CAPS", o cartão do vínculo, o botão "Encaminhar ao CAPS", o tipo "CAPS" no cadastro de unidade e o marcador na linha do tempo somem; 403 `feature_disabled` → `PSY_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo em `features.ts`.
- **Quem faz o quê** (contrato §3–§11): `municipal_admin` — tipo CAPS no cadastro de unidade, leitos, limites e prazo do PTS (step-up), painel; `psychosocial_staff` — acolhimento, matrícula, PTS, ações, grupos, hoje, leitos (entrada/saída), matriciamento, leitura da RAAS e situação de rua, **só nos CAPS em que está lotado** (`not_on_caps_team`); `psychosocial_lead` — tudo da equipe mais cancelar ação de mês fechado, passar do limite de noites com reavaliação, fechar e reabrir o mês (step-up), gerar remessa, baixar remessa e fichas, painel; `billing_clerk` — só baixar remessa e fichas; `analyst` — só o painel; `health_professional` da UBS — cartão do vínculo e "Encaminhar ao CAPS".
- **Nada clínico em lista** (decisão 3; contrato §2 "Texto clínico cifrado", §14): fila de encaminhamentos, lista de usuários, Hoje no CAPS, mapa de leitos, lista de ações, pendências da RAAS e matriciamento mostram só `<patient-ref>`, datas, contagens e códigos de procedimento — **nunca** CID, substância, evolução (`notes`), PTS, `reason` do encaminhamento ou `shared_text`. O detalhe (matrícula, ação, encaminhamento, PTS, caso) é lido **só no clique**, com `NOMINAL_QUERY` (`gcTime: 0`, sem refetch), e sai do cache ao fechar ou trocar de CAPS. A trilha `clinical_record.viewed` é do api. **Nenhum `refetchInterval` no módulo.**
- **Dado pessoal ou clínico só no corpo**: CPF, CNS, nome, CID, substância e texto clínico nunca na URL, em mensagem de erro, em `console`, em Analytics, em `localStorage`, `sessionStorage` ou IndexedDB. Na URL vão só ids, `health_unit_id`, `status`, `date`, `month`, `before` e `enrollment_id`. Busca de paciente por CPF por POST (D3); busca por nome na lista de usuários é **local** (o dashboard nunca manda `q=`, D4).
- **Nada do CAPS para o cidadão** (decisão 4): nenhuma tela nova pede aviso, SMS ou agendamento do 17; a receita do CAPS não aparece em "Documentos" do atendimento da UBS.
- **Todas as recusas de uma vez** (29 §15): a tela confere o formulário inteiro antes de enviar e mostra cada problema; a recusa do api com `problems[]` mostra todas, traduzidas por `psyRefusals`, e o problema com `participant_index` vai **para a linha daquele participante**; sem `problems`, a do topo. Todo código do contrato §1–§7 tem frase em `PSY_ERRORS` (o teste da Task 2 cobra a lista inteira); código novo do api aparece como "recusado pelo servidor (<código>)".
- **Recusar em vez de cortar**: classificação 115 sem 3 dígitos, CNES da USF sem 7 dígitos, quantidade não inteira ou acima do máximo, data futura, motivo curto, texto acima do limite — recusa com a frase; nada é cortado em silêncio.
- **Celular, tablet e teclado**: as telas do CAPS funcionam em tablet (1024×768) e desktop, só com teclado (`<form>` com `Enter`, rótulo em todo campo, foco no título do detalhe aberto e na frase de sucesso); a **lista de presença** do grupo funciona a **375 px** — cartões empilhados, sem `DataTable`, linha de presença com alvo ≥ 44 px e `aria-pressed`, contador "N presentes" e o botão "Registrar presença" no fim; situação nunca só por cor (texto no `Tag`).
- **Sem duplo envio**: botão de escrita desabilitado enquanto a chamada corre (`busy` + `useRef`), e a resposta de uma ação já trocada (outro CAPS, outro usuário, outro mês) é ignorada (`seq`).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo — `{}` quando não há campo).
- Textos (D15; se o api fixar outros, o api vence): motivo (cancelar ação, reabrir mês, reavaliação) 10–500; motivo do encaminhamento 10–2000; evolução (`notes`) e acolhimento até 4000; problemas e necessidades do PTS 10–4000; objetivo 3–500; tema do matriciamento 3–200; `shared_text` 10–2000; nome do grupo 3–80; rótulo do leito 1–20; nome do familiar 3–120; número de autorização só dígitos, 1–20.
- "Hoje" é o dia da cidade (`todayInCity()` de `src/lib/campaigns.ts`) e "agora" é `new Date()`; telas recebem `today`/`now` por prop (padrão o real) e os testes passam `TODAY31 = "2026-11-04"` (quarta) e `NOW31 = new Date("2026-11-04T10:00:00-03:00")`, sem relógio falso. Hora digitada é hora de parede da cidade (`cityLocalIso` de `src/lib/format.ts`).
- Interface (ciclo de interface): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `SegmentedControl`, `CodeSearch`, `PersonStart`, `PrescriptionForm`, `CancelDocument`, `printDocument`, `saveFile`, `openBlobWindow`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-31.1..F-31.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-31.1: api, dashboard e maintenance; F-31.8: api, dashboard e wpda; F-31.2..F-31.7: api e dashboard) e a **Verified** depois da prova no navegador (Task 15).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod31` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-31-psychosocial-care` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod31`, `G=/opt/homebrew/bin/git`.
- Testes: `$R/bin/test dashboard -w dashboard-mod31 <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx` (`sessionWith` abre a janela de step-up e usa `id: "u-1"`; `{ mfa_verified_at: null }` força o código); a sessão entra por `fetchCurrentSession` mockado; as fixtures do 31 vêm de `src/test/psychosocialFixtures.ts` (Task 1).

## Review Focus

1. **Grupo de 25 pessoas registrado num celular de 375 px, com toque duplo numa linha e duplo clique em "Registrar presença".** Espera-se: o toque duplo marca e desmarca (o contador acompanha); o duplo clique faz **uma** chamada a `createPsyAction` com `group_id` e um participante por matrícula marcada; a recusa com `participant_index` aparece na linha daquela pessoa e as marcações ficam como estavam. Teste: Task 9, "presença: toque duplo desmarca; duplo clique envia uma vez; recusa na linha do participante".
2. **Ação com três recusas de uma vez (`cbo_incompatible`, `quantity_exceeded`, `family_only_for_family_care` no 2º participante), ou `month_closed` para quem não é coordenação.** Espera-se: as três frases juntas, a do familiar na linha dele, o formulário intacto; com `month_closed`, a frase diz que só a coordenação corrige; o cancelamento de ação de mês fechado pela equipe recebe `lead_required` com frase, e o da coordenação passa. Teste: Task 7, "três recusas de uma vez; mês fechado e lead_required".
3. **Lista com dado clínico ou detalhe lido sem clique.** Espera-se: nenhuma lista (usuários, fila, ações, Hoje, leitos, RAAS, matriciamento) contém CID, substância, `notes`, `reason` ou `shared_text` mesmo que o api os mande por engano; `getPsyEnrollment`, `getPsyAction` e `getPsyReferral` só são chamados depois do clique; trocar de CAPS desmonta o detalhe e apaga o cache nominal. Teste: Task 6, "lista ignora campo clínico; detalhe só no clique; trocar de CAPS limpa", e Task 5, "fila sem motivo; motivo só ao abrir".
4. **Fechar o mês com a janela do segundo fator fechada e com uma pendência nova que apareceu depois da leitura; gerar a remessa sem layout homologado.** Espera-se: o `SensitiveAction` pede o código e repete o fechamento uma vez só; `month_has_problems` relê o mês e mostra a lista nova; `raas_layout_unavailable` mostra a frase e as fichas espelho continuam baixáveis; reabrir depois da exportação dá `already_exported` com frase. Teste: Task 11, "fechar com step-up e pendência nova; remessa sem layout; reabrir exportado".
5. **Duas pessoas põem usuários no mesmo leito ao mesmo tempo; a equipe tenta passar do limite de noites.** Espera-se: `bed_taken` mostra a frase e relê o mapa; `night_limit_reached` mostra "N de M noites" sem o botão de reavaliação para a equipe; para a coordenação, aparece o motivo (10–500) e a segunda chamada leva `reassessment: { reason }` uma vez só. Teste: Task 10, "leito tomado relê o mapa; limite: equipe não passa, coordenação passa com motivo".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.psychosocial.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/psychosocialFixtures.ts` | tipos e cliente HTTP do 31; campos do CAPS na unidade; interruptor; proxy; dados de teste | 1 |
| `src/lib/psychosocial.ts`, `src/lib/psychosocial.test.ts` | portão, papéis, rótulos, validações, frases de todos os códigos | 2 |
| `src/lib/team.ts`, `src/lib/team.test.ts`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.psychosocial.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/psychosocial/PsychosocialGate.tsx`, `src/modules/psychosocial/PsychosocialGate.test.tsx`, telas-portão `src/modules/Caps{Reception,Today,Enrollments,Groups,Beds,Matrix,Raas,Panel,Settings}.tsx` | papéis na Equipe, grupo "CAPS", portão com escolha do CAPS | 3 |
| `src/lib/attendance.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitForm.caps.test.tsx`, `src/modules/CapsSettings.tsx`, `src/modules/psychosocial/BedsAdmin.tsx`, `src/modules/CapsSettings.test.tsx` | tipo CAPS, modalidade e classificação; leitos; limites e prazo do PTS (F-31.1) | 4 |
| `src/modules/CapsReception.tsx`, `src/modules/psychosocial/PsyPersonPick.tsx`, `src/modules/psychosocial/ReceptionForm.tsx`, `src/modules/psychosocial/EnrollmentFields.tsx`, `src/modules/CapsReception.test.tsx` | fila de encaminhamentos, demanda espontânea, acolhimento com ou sem matrícula (F-31.2) | 5 |
| `src/modules/CapsEnrollments.tsx`, `src/modules/psychosocial/EnrollmentView.tsx`, `src/modules/psychosocial/EnrollmentEdit.tsx`, `src/modules/psychosocial/CloseEnrollment.tsx`, `src/modules/psychosocial/CarePlans.tsx`, `src/modules/CapsEnrollments.test.tsx` | lista sem dado clínico, detalhe com trilha, edição, encerramento, PTS com versões (F-31.2, F-31.3) | 6 |
| `src/modules/psychosocial/ActionsPanel.tsx`, `src/modules/psychosocial/ActionForm.tsx`, `src/modules/psychosocial/ActionDetail.tsx`, `src/modules/psychosocial/Actions.test.tsx` | ações individuais, familiares e no território; crítica na hora; cancelamento (F-31.4) | 7 |
| `src/modules/psychosocial/ActionDocuments.tsx`, `src/modules/psychosocial/ActionDocuments.test.tsx` | receita pelo fluxo do 19/19d a partir da ação (F-31.4, decisão 19) | 8 |
| `src/modules/CapsGroups.tsx`, `src/modules/psychosocial/GroupForm.tsx`, `src/modules/psychosocial/PresenceList.tsx`, `src/modules/CapsToday.tsx`, `src/modules/CapsGroups.test.tsx`, `src/modules/CapsToday.test.tsx` | grade semanal, presença a 375 px, Hoje no CAPS (F-31.4) | 9 |
| `src/modules/CapsBeds.tsx`, `src/modules/psychosocial/NightStayForm.tsx`, `src/modules/psychosocial/CloseNightStay.tsx`, `src/modules/CapsBeds.test.tsx` | mapa dos leitos, entrada, saída, limite e reavaliação (F-31.5) | 10 |
| `src/modules/CapsRaas.tsx`, `src/modules/psychosocial/MonthRecords.tsx`, `src/modules/psychosocial/RaasExports.tsx`, `src/modules/psychosocial/openSheets.ts`, `src/modules/CapsRaas.test.tsx` | pendências, situação de rua, fechar/reabrir, remessa, fichas (F-31.6) | 11 |
| `src/modules/CapsMatrix.tsx`, `src/modules/psychosocial/MatrixForm.tsx`, `src/modules/psychosocial/MatrixCaseForm.tsx`, `src/modules/CapsMatrix.test.tsx` | matriciamento e casos compartilhados (F-31.7) | 12 |
| `src/modules/psychosocial/PsychosocialLinkCard.tsx`, `src/modules/psychosocial/ReferralForm.tsx`, `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/clinicalRecord/JustifiedRecord.tsx`, `src/modules/psychosocial/PsychosocialLinkCard.test.tsx` | cartão do vínculo na UBS e "Encaminhar ao CAPS" (F-31.7, F-31.2) | 13 |
| `src/modules/psychosocial/CapsTimeline.tsx`, `src/modules/psychosocial/CapsTimeline.test.tsx`, `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/CapsPanel.tsx`, `src/modules/CapsPanel.test.tsx` | linha do tempo do CAPS; marcador na geral; painel (F-31.8) | 14 |
| — | suíte, build, varredura, revisão, prova no navegador | 15 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, nome, CID, substância, motivo, evolução, texto compartilhado) (Task 1); regras puras com tabela de casos — inclusive a cobertura de **todos** os códigos de erro do contrato e das pendências da RAAS, a crítica local da ação e a matrícula (Task 2); menu e portão por papel e interruptor (Task 3); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e fixtures do contrato §2, cobrindo o caminho feliz, o teclado, cada recusa que muda a tela, o duplo envio e o que **não** pode aparecer (dado clínico em lista; detalhe sem clique; nada do CAPS para quem não é da equipe) (Tasks 4–14); `tsc`, build, a suíte inteira e as varreduras (Task 15), e a prova no navegador com o api real na 3053, a presença a 375 px (Task 15). O dashboard não testa o que é do api (crítica do SIGTAP, sobreposição no banco, continuidade, BPA-C, trilha, LEDI, wpda).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`f6fb6e4` ou mais novo, com o 30 mergeado); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod31`.
- Produces: cards F-31.1..F-31.8 em In Progress; worktree `$W` na branch `feat/mod-31-psychosocial-care`; a base anotada; as rotas e as formas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-31\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api, do wpda e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-31.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  tfd: "Tratamento Fora de Domicílio (TFD)"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Transporte sanitário e TFD (módulo 30\|^export async function createUnit\|^export async function updateUnit\|^export interface HealthUnit extends\|^  | "transport";\|^async function binaryFetch\|^const withQuery = \|^export interface ClinicalDocument {'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '{ role: "tfd_committee", label: "Comissão de TFD" },\|^const PRIVILEGED_ROLES'
$G -C $R/apps/dashboard show origin/main:src/lib/team.test.ts | grep -n 'oferece os 21 papéis'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "transport-settings" | "tfd-queue" | "tfd-my-cases";\|    { id: "tfd-my-cases", label: "Meus pedidos de TFD"\|  const tfdOn = canUseTfd(user);'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 19\|NAV_GROUPS.length - 18'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n 'IconBus, IconCalendarRepeat\|    "tfd-my-cases": {'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "tfd-my-cases":'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"\^/tfd(/|\$)"'
$G -C $R/apps/dashboard show origin/main:src/lib/attendance.ts | grep -n '^export function unitKindOptions\|HEALTH_MANAGEMENT_UNIT_KIND ='
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitForm.tsx | grep -n 'unitKindOptions(dentalOn'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/Units.tsx | grep -n 'const KIND_LABEL\|await createUnit(value.name'
$G -C $R/apps/dashboard show origin/main:src/modules/Attendance.tsx | grep -n 'tfdOn={canUseTransport(user)}'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/PatientPanel.tsx | grep -n '^export function PatientPanel\|  exams?: PatientExamsProps;'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n '<PatientPanel record={record.data}'
$G -C $R/apps/dashboard show origin/main:src/modules/clinicalRecord/JustifiedRecord.tsx | grep -n '<PatientPanel'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '^export function timelineKinds\|  { key: "transport", label: "Transporte" }'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n 'timelineKinds('
$G -C $R/apps/dashboard show origin/main:src/modules/documents/PrescriptionForm.tsx | grep -n '^export interface ControlledContext\|  onSubmit(content: PrescriptionInput): Promise<void>;'
$G -C $R/apps/dashboard show origin/main:src/modules/territory/PersonStart.tsx | grep -n '^export interface PersonSource'
$G -C $R/apps/dashboard show origin/main:src/modules/lab/saveFile.ts | grep -n '^export function saveFile'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod31 -b feat/mod-31-psychosocial-care origin/main
cd $R/.worktrees/dashboard-mod31 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod31
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Brain|DoorEnter|CalendarEvent|UsersGroup|Bed|Share|FileReport|Puzzle|UserHeart)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os ícones (falta algum → escolha outro que exista e use o mesmo na Task 3; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 15 espera a base mais **18 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira as rotas e formas do api do 31**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api log --oneline -5 feat/mod-31-psychosocial-care 2>/dev/null || echo "api do 31 ainda sem branch"
$G -C $R/apps/api show feat/mod-31-psychosocial-care:config/routes.rb 2>/dev/null | sed -n '/scope "\/psychosocial"/,/^  end/p'
$G -C $R/apps/api show feat/mod-31-psychosocial-care:config/psychosocial/raas.yml 2>/dev/null | head -40
```

Expected: o escopo `/psychosocial` com as rotas do contrato §3–§11. Para cada Divergência (D1–D20) anote se o api adotou a forma proposta ou outra; **onde o api diz outra coisa, vale o api** — a task dona ajusta tipo, fixture e teste antes de seguir (registre na descrição do commit). Sem branch do api ainda: siga pelas formas deste plano e repita este passo antes da Task 15.

---

### Task 1: Tipos, cliente HTTP, campos do CAPS na unidade, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; `HealthUnit`, `createUnit`, `updateUnit`)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/psychosocialFixtures.ts`
- Test: `src/lib/api.psychosocial.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError`, `withQuery`, `binaryFetch`, `SignatureFile`, `PharmacyPerson`, `DuplicateQuery`, `DuplicateCandidate`, `ClinicalDocument`, `PrescriptionInput`, `TimelineEntry`, `SessionUser` (existentes em `src/lib/api.ts`); `sessionWith` de `src/test/campaignFixtures.tsx`.
- Produces (todas em `src/lib/api.ts`, exportadas):
  - Tipos: `CapsModality`, `PsyInstrument`, `CareIntensity`, `ReceptionOutcome`, `ReferralPriority`, `PsyGroupKind`, `PsyPlace`, `MonthStatus`, `PsyCount = number | "lt5"`.
  - Formas (contrato §2 + D): `PsyPatientRef { patient_id: string; name: string; cns_masked?: string | null }`, `PsyPro { user_id: string; name: string; cbo_code: string | null }`, `PsySigtap { code: string; name: string; instrument: PsyInstrument | string }`, `PsyUnit { id; name; caps_modality: CapsModality | string; has_beds: boolean }` (D1), `PsyCode { code; label }`, `PsyProcedure { code; name; instrument; max_quantity: number | null; requires_authorization: boolean; family_care: boolean; collective: boolean }`, `PsyHealthTeam { id; name; ine: string | null; unit: { id; name } }`, `PsyOptions { origins: PsyCode[]; destinations: PsyCode[]; substances: PsyCode[]; procedures: PsyProcedure[]; staff: PsyPro[]; health_teams: PsyHealthTeam[]; me: PsyPro | null; review: boolean }` (D2), `PsyReferralRow`, `PsyReferral`, `PsyReceptionInput`, `PsyEnrollmentInput`, `PsyEnrollmentRow`, `PsyEnrollment`, `PsyEnrollmentPatch`, `PsyPrefill { esf_cnes: string | null; esf_team_name: string | null }` (D5), `PsyCarePlan`, `PsyCarePlanInput`, `PsyGroup`, `PsyGroupInput`, `PsyToday`, `PsyParticipantInput`, `PsyActionInput`, `PsyActionRow`, `PsyAction`, `PsyBed`, `PsyNightStay`, `PsyBedBoard`, `PsyNightStayInput`, `PsyMonthRecord`, `PsyMonth`, `PsyRaasExport`, `PsyMatrixSupportRow`, `PsyMatrixSupport`, `PsyMatrixSupportInput`, `PsyLink`, `PsySettings`, `PsyDashboard`, `PsyTimelinePage { entries: TimelineEntry[]; next_before: string | null }`.
  - Funções: `listPsyUnits(): Promise<PsyUnit[]>`, `getPsyOptions(unitId): Promise<PsyOptions>`, `lookupPsyPerson(cpf): Promise<PharmacyPerson>`, `findPsyDuplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>` (D3), `listPsyReferrals(unitId): Promise<PsyReferralRow[]>`, `getPsyReferral(id): Promise<PsyReferral>`, `createPsyReferral(i: { patient_id; health_unit_id; reason; priority }): Promise<PsyReferralRow>`, `recordPsyReception(i: PsyReceptionInput): Promise<{ reception_id: string; enrollment_id: string | null }>`, `getPsyPrefill(unitId, patientId): Promise<PsyPrefill>`, `createPsyEnrollment(i): Promise<PsyEnrollment>`, `listPsyEnrollments(q: { unitId; status: "open" | "closed" }): Promise<PsyEnrollmentRow[]>`, `getPsyEnrollment(id): Promise<PsyEnrollment>`, `updatePsyEnrollment(id, p: PsyEnrollmentPatch): Promise<PsyEnrollment>`, `closePsyEnrollment(id, { destination, closed_on }): Promise<PsyEnrollment>`, `listPsyCarePlans(enrollmentId): Promise<PsyCarePlan[]>`, `createPsyCarePlan(enrollmentId, i): Promise<PsyCarePlan>`, `listPsyGroups(unitId): Promise<PsyGroup[]>`, `createPsyGroup(i)`, `updatePsyGroup(id, i)`, `getPsyToday(unitId, date): Promise<PsyToday>`, `createPsyAction(i: PsyActionInput): Promise<PsyAction>`, `listPsyActions(enrollmentId): Promise<PsyActionRow[]>`, `getPsyAction(id): Promise<PsyAction>`, `cancelPsyAction(id, reason): Promise<PsyAction>`, `listPsyActionDocuments(actionId): Promise<ClinicalDocument[]>` (D8), `issuePsyPrescription(actionId, content: PrescriptionInput): Promise<ClinicalDocument>`, `listPsyBeds(unitId): Promise<PsyBed[]>`, `createPsyBed(i: { health_unit_id; label; active })`, `updatePsyBed(id, i: { label; active })`, `getPsyBedBoard(unitId, date): Promise<PsyBedBoard>`, `startPsyNightStay(i: PsyNightStayInput): Promise<PsyNightStay>`, `closePsyNightStay(id, { ended_at, reason }): Promise<PsyNightStay>`, `getPsySettings(): Promise<PsySettings>` (D10), `updatePsySettings(s: PsySettings): Promise<PsySettings>`, `getPsyMonth(month, unitId): Promise<PsyMonth>`, `setPsyHomeless(recordId, homeless: boolean): Promise<PsyMonthRecord>`, `closePsyMonth(month, unitId): Promise<PsyMonth>`, `reopenPsyMonth(month, unitId, reason): Promise<PsyMonth>`, `createPsyRaasExport(unitId, month): Promise<PsyRaasExport>`, `listPsyRaasExports(unitId, month): Promise<PsyRaasExport[]>` (D11), `fetchPsyRaasFile(id): Promise<SignatureFile>`, `fetchPsySheets(month, q: { unitId: string; enrollmentId?: string | null }): Promise<SignatureFile>`, `listPsyMatrixSupports(unitId, month): Promise<PsyMatrixSupportRow[]>` (D12), `getPsyMatrixSupport(id): Promise<PsyMatrixSupport>`, `createPsyMatrixSupport(i): Promise<PsyMatrixSupport>`, `addPsyMatrixCase(supportId, { patient_id, shared_text }): Promise<PsyMatrixSupport>`, `getPsyLink(patientId): Promise<PsyLink>` (D13), `getPsyEnrollmentTimeline(enrollmentId, before): Promise<PsyTimelinePage>`, `getPsyDashboard(q: { month; unitId?: string | null }): Promise<PsyDashboard>` (D14).
  - Unidade (contrato §3): `HealthUnit` ganha `caps_modality?: CapsModality | string | null` e `service_classification?: string | null`; `createUnit(name, kind, address, cnes?, caps?)` e `updateUnit(…, cnes?, caps?)` ganham o último parâmetro `caps?: { caps_modality: string | null; service_classification: string | null }` (só vai no corpo quando recebido).
  - `src/test/psychosocialFixtures.ts`: `TODAY31`, `NOW31`, `MONTH31`, `CAPS_II`, `CAPS_AD_III`, `OPTIONS`, `ME_DOCTOR`, `ME_PSY`, `referralRow(o?)`, `enrollmentRow(o?)`, `enrollment(o?)`, `carePlan(o?)`, `group(o?)`, `actionRow(o?)`, `action(o?)`, `bedBoard(o?)`, `month(o?)`, `monthRecord(o?)`, `psyUser(roles, features?)`.

- [ ] **Step 1: Escreva as fixtures** `src/test/psychosocialFixtures.ts` (o único arquivo com corpo completo nesta task; as formas são o contrato §2 com as Divergências)

```ts
// Dados dos testes do módulo 31 (CAPS). Hoje: quarta, 2026-11-04, 10h. Formas do contrato §2 e das Divergências do plano.
import type {
  PsyAction, PsyActionRow, PsyBedBoard, PsyCarePlan, PsyEnrollment, PsyEnrollmentRow, PsyGroup, PsyMonth, PsyMonthRecord, PsyOptions,
  PsyPro, PsyReferralRow, PsyUnit, SessionUser
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY31 = "2026-11-04";
export const NOW31 = new Date("2026-11-04T10:00:00-03:00");
export const MONTH31 = "2026-11";
export const CAPS_II: PsyUnit = { id: "caps2", name: "CAPS II Centro", caps_modality: "caps_ii", has_beds: false };
export const CAPS_AD_III: PsyUnit = { id: "capsad3", name: "CAPS AD III Boqueirão", caps_modality: "caps_ad_iii", has_beds: true };
export const ME_PSY: PsyPro = { user_id: "u-1", name: "Lia Psicóloga", cbo_code: "251510" };
export const ME_DOCTOR: PsyPro = { user_id: "u-1", name: "Dr. Caio Psiquiatra", cbo_code: "225133" };

export const OPTIONS: PsyOptions = {
  origins: [ { code: "01", label: "demanda espontânea" }, { code: "04", label: "atenção básica" } ],
  destinations: [ { code: "00", label: "permanência" }, { code: "01", label: "alta" }, { code: "02", label: "transferência" } ],
  substances: [ { code: "alcohol", label: "álcool" }, { code: "crack", label: "crack" }, { code: "none", label: "nenhuma" } ],
  procedures: [
    { code: "0301080232", name: "ACOLHIMENTO INICIAL POR CAPS", instrument: "raas_psi", max_quantity: 1, requires_authorization: false, family_care: false, collective: false },
    { code: "0301080208", name: "ATENDIMENTO INDIVIDUAL DE PACIENTE EM CAPS", instrument: "raas_psi", max_quantity: 31, requires_authorization: false, family_care: false, collective: false },
    { code: "0301080216", name: "ATENDIMENTO EM GRUPO DE PACIENTE EM CAPS", instrument: "raas_psi", max_quantity: 31, requires_authorization: false, family_care: false, collective: true },
    { code: "0301080224", name: "ATENDIMENTO FAMILIAR EM CAPS", instrument: "raas_psi", max_quantity: 4, requires_authorization: false, family_care: true, collective: false },
    { code: "0301080313", name: "MATRICIAMENTO DE EQUIPES DA ATENÇÃO BÁSICA", instrument: "bpa_c", max_quantity: null, requires_authorization: false, family_care: false, collective: false },
    { code: "0301089999", name: "PROCEDIMENTO COM AUTORIZAÇÃO", instrument: "raas_psi", max_quantity: 1, requires_authorization: true, family_care: false, collective: false }
  ],
  staff: [ ME_PSY, { user_id: "u-2", name: "Rui Enfermeiro", cbo_code: "223505" } ],
  health_teams: [ { id: "ht1", name: "ESF Vila Verde", ine: "0001234567", unit: { id: "ubs1", name: "UBS Vila Verde" } } ],
  me: ME_PSY, review: true
};

export function referralRow(o: Partial<PsyReferralRow> = {}): PsyReferralRow {
  return { id: "rf1", patient: { patient_id: "pt1", name: "Maria da Silva" }, from_unit: { id: "ubs1", name: "UBS Vila Verde" },
    priority: "priority", status: "pending", created_at: "2026-11-03T14:00:00-03:00", ...o };
}

export function enrollmentRow(o: Partial<PsyEnrollmentRow> = {}): PsyEnrollmentRow {
  return { id: "en1", health_unit_id: CAPS_II.id, patient: { patient_id: "pt1", name: "Maria da Silva", cns_masked: "*** **** **** 1234" },
    started_on: "2026-09-10", status: "open", reference: { user_id: "u-1", name: ME_PSY.name }, next_visit_at: "2026-11-04T14:00:00-03:00",
    care_plan_review_on: "2026-12-01", care_plan_overdue: false, ...o };
}

export function enrollment(o: Partial<PsyEnrollment> = {}): PsyEnrollment {
  return { ...enrollmentRow(), origin: "04", cid_main: { code: "F32.2", label: "Episódio depressivo grave" }, cid_secondary: [],
    substances: [ "none" ], esf_covered: true, esf_cnes: "2384299", destination: null, closed_on: null, raas_exported: false,
    active_night_stay: null, ...o };
}

export function carePlan(o: Partial<PsyCarePlan> = {}): PsyCarePlan {
  return { id: "cp1", version: 1, needs: "Isolamento social e insônia.", goals: [ { text: "Voltar ao grupo de convivência", responsible: { user_id: "u-1", name: ME_PSY.name }, due_on: null } ],
    intensity: "semi_intensive", review_on: "2026-12-01", created_at: "2026-09-10T11:00:00-03:00", author: { name: ME_PSY.name }, ...o };
}

export function group(o: Partial<PsyGroup> = {}): PsyGroup {
  return { id: "g1", health_unit_id: CAPS_II.id, name: "Oficina de artes", kind: "workshop", weekday: 3, starts_at: "14:00", ends_at: "16:00",
    responsible: { user_id: "u-1", name: ME_PSY.name }, capacity: 25, sigtap_code: "0301080216", active: true, ...o };
}

export function actionRow(o: Partial<PsyActionRow> = {}): PsyActionRow {
  return { id: "ac1", sigtap: { code: "0301080208", name: "ATENDIMENTO INDIVIDUAL DE PACIENTE EM CAPS", instrument: "raas_psi" },
    performed_at: "2026-11-04T09:00:00-03:00", place: "caps", quantity: 1, professional: ME_PSY, participants_count: 1, group: null,
    status: "recorded", month_closed: false, ...o };
}

export function action(o: Partial<PsyAction> = {}): PsyAction {
  return { ...actionRow(), notes: "Relata melhora do sono.", authorization_number: null,
    participants: [ { kind: "enrollment", enrollment_id: "en1", name: "Maria da Silva" } ], cancel_reason: null, ...o };
}

export function bedBoard(o: Partial<PsyBedBoard> = {}): PsyBedBoard {
  return { date: TODAY31, window_days: 30, beds: [
    { bed: { id: "b1", health_unit_id: CAPS_AD_III.id, label: "Leito 1", active: true }, stay: null },
    { bed: { id: "b2", health_unit_id: CAPS_AD_III.id, label: "Leito 2", active: true }, stay: { id: "ns1", bed_id: "b2", enrollment_id: "en2",
      patient: { patient_id: "pt2", name: "José Pereira", cns_masked: null }, started_at: "2026-11-02T19:00:00-03:00",
      expected_end_at: "2026-11-06T08:00:00-03:00", ended_at: null, nights_used: 9, limit: 14 } }
  ], ...o };
}

export function monthRecord(o: Partial<PsyMonthRecord> = {}): PsyMonthRecord {
  return { id: "mr1", enrollment_id: "en1", patient: { patient_id: "pt1", name: "Maria da Silva", cns_masked: "*** **** **** 1234" },
    homeless: null, actions_count: 3, status: "open", problems: [ { code: "homeless_missing" } ], ...o };
}

export function month(o: Partial<PsyMonth> = {}): PsyMonth {
  return { month: MONTH31, health_unit_id: CAPS_II.id, status: "open", closure: null,
    counts: { records: 1, with_problems: 1, actions: 3 }, records: [ monthRecord() ], ...o };
}

export function psyUser(roles: string[] = [ "psychosocial_staff" ], features: string[] = [ "psychosocial_care", "clinical_record" ]): SessionUser {
  return sessionWith(roles, { features });
}
```

- [ ] **Step 2: Escreva os testes do cliente** `src/lib/api.psychosocial.test.ts`

```ts
import { afterEach, describe, expect, it, vi } from "vitest";
import * as api from "./api";

const calls: { url: string; init: RequestInit }[] = [];
function stub(body: unknown, status = 200, headers: Record<string, string> = { "Content-Type": "application/json" }) {
  vi.stubGlobal("fetch", vi.fn(async (url: string, init: RequestInit = {}) => {
    calls.push({ url, init });
    return new Response(typeof body === "string" ? body : JSON.stringify(body), { status, headers });
  }));
}
afterEach(() => { calls.length = 0; vi.unstubAllGlobals(); });
const last = () => calls[calls.length - 1];
const bodyOf = () => JSON.parse(String(last().init.body));

describe("cliente do CAPS (módulo 31)", () => {
  it("lista de usuários: só unidade e situação na URL, nunca q", async () => {
    stub({ items: [] });
    await api.listPsyEnrollments({ unitId: "caps2", status: "open" });
    expect(last().url).toBe("/psychosocial/enrollments?health_unit_id=caps2&status=open");
  });
  it("encaminhamento da UBS: motivo só no corpo, com JSON", async () => {
    stub({ id: "rf1" }, 201);
    await api.createPsyReferral({ patient_id: "pt1", health_unit_id: "caps2", reason: "Ideação suicida relatada na consulta", priority: "priority" });
    expect(last().url).toBe("/psychosocial/referrals");
    expect(last().url).not.toMatch(/Idea/);
    expect(bodyOf().reason).toBe("Ideação suicida relatada na consulta");
    expect((last().init.headers as Record<string, string>)["Content-Type"]).toBe("application/json");
  });
  it("fila pendente e detalhe do encaminhamento", async () => {
    stub({ items: [] });
    await api.listPsyReferrals("caps2");
    expect(last().url).toBe("/psychosocial/referrals?health_unit_id=caps2&status=pending");
    stub({ id: "rf1", reason: "x" });
    await api.getPsyReferral("rf1");
    expect(last().url).toBe("/psychosocial/referrals/rf1");
  });
  it("busca de pessoa pelo CAPS é POST com o CPF no corpo (D3)", async () => {
    stub({ patient_id: "pt1", display_name: "Maria", has_cpf: true, cns_masked: null });
    await api.lookupPsyPerson("12345678909");
    expect(last().url).toBe("/psychosocial/person_lookup");
    expect(last().init.method).toBe("POST");
    expect(bodyOf()).toEqual({ cpf: "12345678909" });
  });
  it("acolhimento com matrícula na mesma chamada; CID e substâncias no corpo", async () => {
    stub({ reception_id: "re1", enrollment_id: "en9" }, 201);
    await api.recordPsyReception({ health_unit_id: "caps2", patient_id: "pt1", at: "2026-11-04T09:00:00-03:00", sigtap_code: "0301080232",
      notes: "", outcome: "enrolled", enrollment: { started_on: "2026-11-04", origin: "01", cid_main: "F32.2", cid_secondary: [], substances: [ "alcohol" ],
        esf_covered: false, esf_cnes: null, reference_user_id: "u-1" } });
    expect(last().url).toBe("/psychosocial/receptions");
    expect(last().url).not.toMatch(/F32|alcohol/);
    expect(bodyOf().enrollment.cid_main).toBe("F32.2");
  });
  it("ação coletiva: participantes no corpo", async () => {
    stub({ id: "ac9" }, 201);
    await api.createPsyAction({ health_unit_id: "caps2", sigtap_code: "0301080216", performed_at: "2026-11-04T14:00:00-03:00", place: "caps",
      quantity: 1, group_id: "g1", participants: [ { enrollment_id: "en1" }, { enrollment_id: "en2" } ] });
    expect(last().url).toBe("/psychosocial/actions");
    expect(bodyOf().participants).toHaveLength(2);
  });
  it("cancelar ação e receita da ação", async () => {
    stub({ id: "ac1", status: "cancelled" });
    await api.cancelPsyAction("ac1", "registrada no usuário errado");
    expect(last().url).toBe("/psychosocial/actions/ac1/cancel");
    expect(bodyOf()).toEqual({ reason: "registrada no usuário errado" });
    stub({ id: "d1" }, 201);
    await api.issuePsyPrescription("ac1", { items: [] });
    expect(last().url).toBe("/psychosocial/actions/ac1/documents");
    expect(bodyOf()).toEqual({ kind: "prescription", content: { items: [] } });
  });
  it("leito: entrada com reavaliação só quando passada", async () => {
    stub({ id: "ns1" }, 201);
    await api.startPsyNightStay({ bed_id: "b1", enrollment_id: "en1", started_at: "2026-11-04T19:00:00-03:00", expected_end_at: "2026-11-05T08:00:00-03:00" });
    expect(bodyOf().reassessment).toBeUndefined();
    await api.startPsyNightStay({ bed_id: "b1", enrollment_id: "en1", started_at: "2026-11-04T19:00:00-03:00", expected_end_at: "2026-11-05T08:00:00-03:00",
      reassessment: { reason: "risco de recaída na semana" } });
    expect(bodyOf().reassessment).toEqual({ reason: "risco de recaída na semana" });
    stub({ beds: [] });
    await api.getPsyBedBoard("capsad3", "2026-11-04");
    expect(last().url).toBe("/psychosocial/beds/board?health_unit_id=capsad3&date=2026-11-04");
  });
  it("mês: leitura, fechar, reabrir e situação de rua", async () => {
    stub({ month: "2026-11" });
    await api.getPsyMonth("2026-11", "caps2");
    expect(last().url).toBe("/psychosocial/months/2026-11?health_unit_id=caps2");
    await api.closePsyMonth("2026-11", "caps2");
    expect(last().url).toBe("/psychosocial/months/2026-11/close");
    expect(bodyOf()).toEqual({ health_unit_id: "caps2" });
    await api.reopenPsyMonth("2026-11", "caps2", "ação lançada no mês errado");
    expect(bodyOf()).toEqual({ health_unit_id: "caps2", reason: "ação lançada no mês errado" });
    await api.setPsyHomeless("mr1", false);
    expect(last().url).toBe("/psychosocial/monthly_records/mr1");
    expect(last().init.method).toBe("PATCH");
    expect(bodyOf()).toEqual({ homeless: false });
  });
  it("fichas espelho e remessa por fetch binário", async () => {
    stub("%PDF", 200, { "Content-Type": "application/pdf", "Content-Disposition": "attachment; filename=\"fichas-2026-11.pdf\"" });
    const f = await api.fetchPsySheets("2026-11", { unitId: "caps2", enrollmentId: "en1" });
    expect(last().url).toBe("/psychosocial/months/2026-11/sheets.pdf?health_unit_id=caps2&enrollment_id=en1");
    expect(f.filename).toBe("fichas-2026-11.pdf");
    stub("RAAS", 200, { "Content-Type": "text/plain", "Content-Disposition": "attachment; filename=\"raas.txt\"" });
    await api.fetchPsyRaasFile("rx1");
    expect(last().url).toBe("/psychosocial/raas_exports/rx1/file");
  });
  it("vínculo na UBS (D13) e painel com lt5", async () => {
    stub({ enrolled: false });
    await api.getPsyLink("pt1");
    expect(last().url).toBe("/psychosocial/patients/pt1/link");
    stub({ month: "2026-11", new_in_month: "lt5" });
    const d = await api.getPsyDashboard({ month: "2026-11", unitId: null });
    expect(last().url).toBe("/psychosocial/dashboard?month=2026-11");
    expect(d.new_in_month).toBe("lt5");
  });
  it("unidade CAPS: modalidade e classificação no corpo da unidade", async () => {
    stub({ unit: { id: "caps2" } }, 201);
    await api.createUnit("CAPS II Centro", "caps", { zip: "", street: "", number: "", complement: "", neighborhood_id: null } as never, "2384299",
      { caps_modality: "caps_ii", service_classification: "002" });
    expect(bodyOf()).toMatchObject({ kind: "caps", caps_modality: "caps_ii", service_classification: "002", cnes: "2384299" });
    await api.createUnit("UBS Nova", "ubs", { zip: "", street: "", number: "", complement: "", neighborhood_id: null } as never);
    expect("caps_modality" in bodyOf()).toBe(false);
  });
});
```

(Na Task 0 confira o tipo real de `UnitAddress` e troque o `as never` pelo objeto vazio correto.)

- [ ] **Step 3: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/lib/api.psychosocial.test.ts`
Expected: FAIL — `listPsyEnrollments`, `createPsyReferral` etc. não existem.

- [ ] **Step 4: Implemente a seção "Atenção psicossocial (módulo 31; ADR 0049; contrato 2026-11-04 §2–§11)" no fim de `src/lib/api.ts`**

`PSY_BASE = import.meta.env.VITE_PSYCHOSOCIAL_BASE || "/psychosocial"`; `psyPath(...parts)` com `encodeURIComponent` por parte (molde de `trPath`); escrita com `{ method, body: JSON.stringify(body ?? {}) }`. Listas em envelope `{ items }` (ausente = vazio); `listPsyUnits` lê `{ units }`. Parâmetros vazios ou nulos nunca vão na URL (`withQuery`). `fetchPsyRaasFile` e `fetchPsySheets` usam `binaryFetch` (aceite `text/plain` e `application/pdf`). `issuePsyPrescription` manda `{ kind: "prescription", content }`. Comentário de cabeçalho da seção: "Na URL só ids, unidade, situação, data, mês e cursor; CPF, nome, CID, substância, motivo, evolução, PTS e texto compartilhado vão no corpo. Lista nunca traz texto clínico; o detalhe grava a trilha no api." `HealthUnit`, `createUnit` e `updateUnit` ganham os campos do CAPS (o parâmetro `caps` só entra no corpo quando recebido, com as duas chaves).

- [ ] **Step 5: Interruptor e proxy**

`src/lib/features.ts`: `FeatureKey` ganha `"psychosocial_care"`; `FEATURE_LABEL` ganha, com o comentário "Módulo 31 (ADR 0049): exige clinical_record", `psychosocial_care: "Atenção psicossocial (CAPS)"`. Em `src/lib/features.test.ts`, no teste "rótulo conhecido em português", acrescente `expect(featureLabel("psychosocial_care")).toBe("Atenção psicossocial (CAPS)");`.

`vite.config.ts`: no comentário, `/psychosocial → CAPS (módulo 31: acolhimento, matrícula, PTS, ações, leitos, RAAS, matriciamento, painel)`; no proxy, `"^/psychosocial(/|$)": proxy(TARGET)`.

- [ ] **Step 6: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/lib/api.psychosocial.test.ts src/lib/features.test.ts src/lib/api.signature.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS; `tsc` sem erro.

- [ ] **Step 7: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.psychosocial.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/psychosocialFixtures.ts
$G commit -m "feat: add psychosocial care API client and fixtures" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela do CAPS (`src/lib/psychosocial.ts`)

**Files:**
- Create: `src/lib/psychosocial.ts`
- Test: `src/lib/psychosocial.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `ApiError`, `errorCode`; `featureDisabledKey`, `featureLabel`, `hasFeature`; `NOMINAL_QUERY`, `STATIC_QUERY` de `src/lib/careLines.ts`; `cityLocalIso`, `parseCityLocal` de `src/lib/format.ts`; `Problem`, `Refusal` de `src/lib/pharmacy.ts`.
- Produces (exportados):
  - Frases: `PSY_DISABLED = "a atenção psicossocial está desligada nesta cidade"`, `PSY_NO_ROLE = "seu papel não permite usar o CAPS"`, `NO_CAPS_UNIT = "você não está lotado em nenhum CAPS — a lotação vem do vínculo CNES ativo na unidade"`, `SMALL_COUNT = "menos de 5"`, `TRAIL_NOTE = "Abrir registra a leitura no prontuário do CAPS."`, `NO_CLINICAL_IN_LIST = "A lista não mostra CID, substâncias, evolução nem PTS."`, `REASON_TEXT = "o motivo tem de 10 a 500 caracteres"`.
  - Chaves: `PSY_UNITS_KEY`, `PSY_OPTIONS_KEY`, `PSY_REFERRALS_KEY`, `PSY_ENROLLMENTS_KEY`, `PSY_DETAIL_KEY` (raiz de todo detalhe nominal), `PSY_ACTIONS_KEY`, `PSY_GROUPS_KEY`, `PSY_TODAY_KEY`, `PSY_BEDS_KEY`, `PSY_MONTH_KEY`, `PSY_EXPORTS_KEY`, `PSY_MATRIX_KEY`, `PSY_SETTINGS_KEY`, `PSY_DASHBOARD_KEY`, e o reexporte de `NOMINAL_QUERY`, `STATIC_QUERY`.
  - Portão e papéis: `canUsePsychosocial(user): boolean` (não operador, `psychosocial_care` **e** `clinical_record`); `psychosocialRoles(user): PsyRoles` com `type PsyRoles = { staff: boolean; lead: boolean; admin: boolean; analyst: boolean; billing: boolean; professional: boolean }` (`staff` é verdadeiro também para `psychosocial_lead`); `isPrescriber(cbo: string | null): boolean` (`225*`).
  - Rótulos (código novo aparece cru): `modalityLabel`, `instrumentLabel`, `intensityLabel`, `outcomeLabel`, `priorityLabel`, `groupKindLabel`, `placeLabel`, `monthStatusLabel`, `monthProblemLabel`, `weekdayLabel(n: number)`, `countLabel(c: PsyCount)` (`"lt5"` → `SMALL_COUNT`), `codeLabel(list: PsyCode[], code)`; `hasBeds(modality)` = `NIGHT_MODALITIES.includes(modality)` com `NIGHT_MODALITIES = [ "caps_iii", "caps_ad_iii", "caps_ad_iv" ]`; `CAPS_MODALITIES` (ordem do contrato).
  - Validações (cada uma devolve `Problem[]` com `path` igual ao do api): `unitCapsProblems({ kind, capsModality, serviceClassification })`, `enrollmentProblems(d: EnrollmentDraft, today)`, `enrollmentInput(d, unitId, patientId): PsyEnrollmentInput`, `actionProblems(d: ActionDraft, proc: PsyProcedure | undefined, now)`, `actionInput(d, unitId): PsyActionInput`, `carePlanProblems(d: CarePlanDraft, today)`, `reasonProblem(reason, min = 10, max = 500): string | null`, `textProblem(value, min, max, label): string | null`, `settingsProblems(s: PsySettings)`.
  - Recusas: `PSY_ERRORS: Record<string, string>`, `MONTH_PROBLEM_TEXT: Record<string, string>`, `nightLimitText(used: number, limit: number): string`, `psyRefusals(err): (Refusal & { participantIndex: number | null })[]`, `psyError(err): string`, `isStale(err): boolean` (`not_found`, `referral_not_pending`).
  - Rascunhos: `type EnrollmentDraft = { startedOn; origin; cidMain: { code; label } | null; cidSecondary: { code; label }[]; substances: string[]; esfCovered: boolean; esfCnes: string; referenceUserId }`, `type ParticipantDraft = { kind: "enrollment"; enrollmentId; name } | { kind: "patient"; patientId; name } | { kind: "family"; name }`, `type ActionDraft = { sigtapCode; performedAt: string /* "YYYY-MM-DDTHH:MM" */; place: PsyPlace; quantity: string; notes; authorizationNumber; groupId: string | null; participants: ParticipantDraft[] }`, `type CarePlanDraft = { needs; goals: { text; responsibleUserId; dueOn }[]; intensity: CareIntensity | ""; reviewOn }`, `emptyEnrollmentDraft(today, meId)`, `emptyActionDraft(now)`.

- [ ] **Step 1: Escreva os testes** `src/lib/psychosocial.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  MONTH_PROBLEM_TEXT, PSY_DISABLED, PSY_ERRORS, actionProblems, canUsePsychosocial, countLabel, emptyActionDraft, emptyEnrollmentDraft,
  enrollmentProblems, hasBeds, isPrescriber, modalityLabel, nightLimitText, psyError, psyRefusals, psychosocialRoles, unitCapsProblems
} from "./psychosocial";
import { NOW31, OPTIONS, TODAY31, psyUser } from "../test/psychosocialFixtures";

const proc = (code: string) => OPTIONS.procedures.find((p) => p.code === code);
const refusal = (status: number, body: unknown) => new ApiError(status, body);

describe("regras do CAPS (módulo 31)", () => {
  it("portão exige psychosocial_care e clinical_record; lead implica equipe", () => {
    expect(canUsePsychosocial(psyUser())).toBe(true);
    expect(canUsePsychosocial(psyUser([ "psychosocial_staff" ], [ "psychosocial_care" ]))).toBe(false);
    expect(canUsePsychosocial({ ...psyUser(), operator: true })).toBe(false);
    expect(psychosocialRoles(psyUser([ "psychosocial_lead" ]))).toMatchObject({ staff: true, lead: true });
    expect(psychosocialRoles(psyUser([ "billing_clerk" ]))).toMatchObject({ staff: false, billing: true });
  });

  it("todo código do contrato tem frase própria, nunca o código cru", () => {
    const contract = [ "feature_disabled", "not_on_caps_team", "caps_modality_mismatch", "invalid_service_classification", "caps_without_beds",
      "enrollment_open", "instrument_mismatch", "cbo_incompatible", "quantity_exceeded", "performed_in_future", "month_closed", "enrollment_required",
      "authorization_required", "family_only_for_family_care", "lead_required", "prescriber_required", "destination_is_permanence", "bed_taken",
      "already_staying", "month_has_problems", "already_exported", "raas_layout_unavailable",
      "esf_cnes_required", "closed_on_in_future", "origin_locked", "referral_not_pending" ];
    for (const code of contract) {
      const text = code === "feature_disabled" ? PSY_DISABLED : PSY_ERRORS[code];
      expect(text, code).toBeTruthy();
      expect(psyError(refusal(422, { error: code })), code).not.toContain(code);
    }
    for (const code of [ "patient_cns_missing", "phone_missing", "homeless_missing", "professional_cns_missing", "professional_not_in_cnes",
      "cid_missing", "no_actions", "action_out_of_rule" ]) expect(MONTH_PROBLEM_TEXT[code], code).toBeTruthy();
  });

  it("limite de noites com os números; código desconhecido aparece cru", () => {
    expect(nightLimitText(14, 14)).toBe("limite de noites atingido: 14 de 14 na janela de 30 dias — só a coordenação passa, com reavaliação");
    expect(psyError(refusal(422, { error: "night_limit_reached", used: 14, limit: 14 }))).toBe(nightLimitText(14, 14));
    expect(psyError(refusal(422, { error: "something_new" }))).toBe("recusado pelo servidor (something_new)");
    expect(psyError(refusal(403, { error: "feature_disabled", feature: "psychosocial_care" }))).toBe(PSY_DISABLED);
  });

  it("todas as recusas de uma vez, com o índice do participante (code ou error)", () => {
    const r = psyRefusals(refusal(422, { error: "invalid_action", problems: [
      { code: "cbo_incompatible" }, { error: "quantity_exceeded", path: "quantity" }, { code: "family_only_for_family_care", participant_index: 1 } ] }));
    expect(r.map((x) => x.code)).toEqual([ "cbo_incompatible", "quantity_exceeded", "family_only_for_family_care" ]);
    expect(r[2].participantIndex).toBe(1);
    expect(r[0].participantIndex).toBeNull();
  });

  it("unidade CAPS: modalidade obrigatória, classificação de 3 dígitos; fora do CAPS, nada", () => {
    expect(unitCapsProblems({ kind: "caps", capsModality: "", serviceClassification: "02" }).map((p) => p.path))
      .toEqual([ "caps_modality", "service_classification" ]);
    expect(unitCapsProblems({ kind: "caps", capsModality: "caps_ii", serviceClassification: "002" })).toEqual([]);
    expect(unitCapsProblems({ kind: "ubs", capsModality: "", serviceClassification: "" })).toEqual([]);
    expect(hasBeds("caps_ad_iii")).toBe(true);
    expect(hasBeds("caps_ii")).toBe(false);
    expect(modalityLabel("caps_i_child")).toBe("CAPS i (infantojuvenil)");
    expect(modalityLabel("caps_x")).toBe("caps_x");
  });

  it("matrícula: CID, origem, técnico, CNES da USF quando coberto, nenhuma exclusiva, início não futuro", () => {
    const d = { ...emptyEnrollmentDraft(TODAY31, "u-1"), origin: "01", cidMain: { code: "F32.2", label: "x" }, substances: [ "none" ] };
    expect(enrollmentProblems(d, TODAY31)).toEqual([]);
    const paths = (o: object) => enrollmentProblems({ ...d, ...o }, TODAY31).map((p) => p.path);
    expect(paths({ cidMain: null })).toEqual([ "cid_main" ]);
    expect(paths({ esfCovered: true, esfCnes: "123" })).toEqual([ "esf_cnes" ]);
    expect(paths({ substances: [ "none", "alcohol" ] })).toEqual([ "substances" ]);
    expect(paths({ substances: [] })).toEqual([ "substances" ]);
    expect(paths({ startedOn: "2026-11-05" })).toEqual([ "started_on" ]);
    expect(paths({ referenceUserId: "" })).toEqual([ "reference_user_id" ]);
  });

  it("ação: crítica local (futuro, máximo, autorização, familiar, matrícula na RAAS); o resto é do api", () => {
    const base = { ...emptyActionDraft(NOW31), sigtapCode: "0301080208", performedAt: "2026-11-04T09:00", quantity: "1",
      participants: [ { kind: "enrollment" as const, enrollmentId: "en1", name: "Maria" } ] };
    expect(actionProblems(base, proc("0301080208"), NOW31)).toEqual([]);
    const codes = (o: object, code = "0301080208") => actionProblems({ ...base, ...o }, proc(code), NOW31).map((p) => p.path);
    expect(codes({ performedAt: "2026-11-04T11:00" })).toEqual([ "performed_at" ]);
    expect(codes({ quantity: "32" })).toEqual([ "quantity" ]);
    expect(codes({ quantity: "1.5" })).toEqual([ "quantity" ]);
    expect(codes({ quantity: "1" }, "0301089999")).toEqual([ "authorization_number" ]);
    expect(codes({ participants: [ ...base.participants, { kind: "family", name: "Joana Silva" } ] })).toEqual([ "participants.1" ]);
    expect(codes({ participants: [ ...base.participants, { kind: "family", name: "Joana Silva" } ] }, "0301080224")).toEqual([]);
    expect(codes({ participants: [] })).toEqual([ "participants" ]);
    expect(codes({ participants: [] }, "0301080313")).toEqual([]);
    expect(codes({ sigtapCode: "" }, "")).toEqual([ "sigtap_code" ]);
  });

  it("prescritor é CBO 225; contagem pequena", () => {
    expect(isPrescriber("225133")).toBe(true);
    expect(isPrescriber("251510")).toBe(false);
    expect(isPrescriber(null)).toBe(false);
    expect(countLabel("lt5")).toBe("menos de 5");
    expect(countLabel(12)).toBe("12");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/lib/psychosocial.test.ts`
Expected: FAIL — módulo não existe.

- [ ] **Step 3: Implemente `src/lib/psychosocial.ts`**

Cabeçalho: "Regras de tela do módulo 31 (CAPS; contrato 2026-11-04 §1–§11) sem React. Lotação, crítica do SIGTAP, CBO, mês fechado, leito, limite de noites, BPA-C, trilha e imutabilidade são do api; a tela só não envia o que já sabe que será recusado." Rótulos com o `pick` do 30 (código novo cru): modalidade `caps_i` "CAPS I", `caps_ii` "CAPS II", `caps_iii` "CAPS III", `caps_i_child` "CAPS i (infantojuvenil)", `caps_ad` "CAPS AD", `caps_ad_iii` "CAPS AD III", `caps_ad_iv` "CAPS AD IV"; instrumento `raas_psi` "RAAS-PSI", `bpa_c` "BPA-C", `bpa_i` "BPA-I"; intensidade "intensivo" / "semi-intensivo" / "não intensivo"; desfecho `enrolled` "matricular no CAPS", `referred_elsewhere` "encaminhar a outro serviço", `no_follow_up` "sem seguimento"; prioridade "rotina" / "prioritário"; grupo "grupo" / "oficina" / "assembleia"; local "no CAPS" / "no território"; mês `open` "aberto", `closed` "fechado", `exported` "remessa gerada"; dia ISO 1 = "segunda" … 7 = "domingo".

Validações (regras exatas):
- `unitCapsProblems`: só com `kind === "caps"`; `caps_modality` vazio → "escolha a modalidade do CAPS"; `service_classification` fora de `/^\d{3}$/` → `PSY_ERRORS.invalid_service_classification`.
- `enrollmentProblems`: `started_on` vazio ou depois de `today` → "o início não pode ser futuro"; `origin` vazio → "escolha a origem do usuário"; `cid_main` nulo → "informe o CID principal"; `cid_secondary` com mais de 4 → "no máximo 4 CIDs secundários"; `substances` vazio → "marque as substâncias ou nenhuma"; `none` junto com outra → "nenhuma não combina com outra substância"; `esf_covered` com `esf_cnes` fora de `/^\d{7}$/` → `PSY_ERRORS.esf_cnes_required`; `reference_user_id` vazio → "escolha o técnico de referência".
- `actionProblems`: sem código → "escolha o procedimento"; `performed_at` inválido ou depois de `now` → `PSY_ERRORS.performed_in_future`; `quantity` que não é inteiro ≥ 1 → "a quantidade é um número inteiro a partir de 1", e acima de `max_quantity` (quando não nulo) → `PSY_ERRORS.quantity_exceeded`; `requires_authorization` sem número em `/^\d{1,20}$/` → `PSY_ERRORS.authorization_required`; participante `family` com `family_care` falso → `PSY_ERRORS.family_only_for_family_care` no `path` `participants.<i>`; nome do familiar fora de 3–120 → "nome do familiar: de 3 a 120 caracteres" no mesmo `path`; instrumento `raas_psi` sem participante `enrollment` → `PSY_ERRORS.enrollment_required` em `participants`; `bpa_c` aceita zero; `notes` acima de 4000 → "a evolução tem até 4000 caracteres".
- `carePlanProblems`: `needs` 10–4000; ao menos um objetivo, cada `text` 3–500 com responsável; `intensity` escolhida; `review_on` depois de `today`.
- `settingsProblems`: todo número inteiro ≥ 1; `window_days` ≤ 90; `care_plan_review_days` de 7 a 365.
- `actionInput`: `performed_at` com `cityLocalIso`; `quantity` número; participantes `{ enrollment_id }` / `{ patient_id }` / `{ family_name }`; `notes`, `authorization_number` e `group_id` omitidos quando vazios.

Recusas — `PSY_ERRORS` (cópia fixa):

```ts
export const PSY_ERRORS: Record<string, string> = {
  not_on_caps_team: "você não está lotado neste CAPS — o vínculo CNES ativo na unidade é que dá acesso",
  caps_modality_mismatch: "a modalidade só vale para unidade do tipo CAPS, e todo CAPS precisa dela",
  invalid_service_classification: "a classificação do serviço 115 tem 3 dígitos",
  caps_without_beds: "só CAPS III, CAPS AD III e CAPS AD IV têm leitos",
  enrollment_open: "este usuário já tem matrícula aberta neste CAPS",
  instrument_mismatch: "este procedimento não é de RAAS-PSI, BPA-C ou BPA-I no SIGTAP",
  cbo_incompatible: "o seu CBO neste CAPS não é compatível com este procedimento no SIGTAP",
  quantity_exceeded: "a quantidade passa do máximo do SIGTAP para este procedimento",
  performed_in_future: "a data da ação não pode ser futura",
  month_closed: "o mês desta ação está fechado — só a coordenação do CAPS corrige",
  enrollment_required: "ação de RAAS precisa de ao menos um participante com matrícula aberta",
  authorization_required: "este procedimento exige o número de autorização",
  family_only_for_family_care: "familiar sem cadastro só entra no atendimento familiar",
  lead_required: "só a coordenação do CAPS faz isso com o mês fechado",
  prescriber_required: "a receita sai só de atendimento individual, com um usuário matriculado, por médico (CBO 225)",
  destination_is_permanence: "escolha o destino do usuário — permanência não encerra a matrícula",
  bed_taken: "este leito já está ocupado nesse período",
  already_staying: "este usuário já está em outra permanência aberta",
  month_has_problems: "o mês tem pendências — resolva a lista antes de fechar",
  already_exported: "a remessa deste mês já foi gerada — não dá mais para reabrir",
  raas_layout_unavailable: "o layout oficial da remessa RAAS ainda não foi homologado — use as fichas espelho para digitar no aplicativo RAAS",
  esf_cnes_required: "informe o CNES da USF que cobre o usuário (7 dígitos)",
  closed_on_in_future: "a data de encerramento não pode ser futura",
  origin_locked: "origem e início só mudam antes da primeira RAAS exportada",
  referral_not_pending: "este encaminhamento já foi acolhido — a fila foi atualizada",
  person_not_found: "não encontramos ninguém com este CPF",
  invalid_cpf: "CPF inválido — confira os 11 dígitos",
  citizen_not_verified: "o CPF ainda não foi conferido no cadastro nacional — tente de novo mais tarde",
  too_many_requests: "muitas buscas em pouco tempo — espere um minuto",
  not_found: "não encontrado — a tela foi atualizada",
  unauthenticated: "sua sessão terminou — entre de novo",
  mfa_required: "confirme sua identidade com o código do autenticador e tente de novo",
  missing_role: "seu papel não permite esta ação"
};
export const MONTH_PROBLEM_TEXT: Record<string, string> = {
  patient_cns_missing: "CNS do usuário ausente", phone_missing: "telefone do usuário ausente",
  homeless_missing: "situação de rua do mês não informada", professional_cns_missing: "CNS do profissional ausente",
  professional_not_in_cnes: "profissional fora do CNES local", cid_missing: "CID ausente", no_actions: "nenhuma ação no mês",
  action_out_of_rule: "ação fora da regra do SIGTAP"
};
```

`nightLimitText(used, limit)` = `` `limite de noites atingido: ${used} de ${limit} na janela de 30 dias — só a coordenação passa, com reavaliação` `` (a janela vem de `window_days` no corpo quando houver; sem ela, 30). `psyRefusals`: 403 `feature_disabled` → `PSY_DISABLED` (outra chave → "<rótulo> está desligado nesta cidade"); `problems[]` com `code` **ou** `error`, `path` e `participant_index` (D16); `night_limit_reached` com `used`/`limit` do corpo; `invalid_request` → frase por `path` (`reason` → `REASON_TEXT`; outro → "pedido inválido — recarregue a tela e tente de novo"); código conhecido → frase; código desconhecido do servidor → "recusado pelo servidor (<código>)"; erro de rede → "não foi possível falar com o servidor — tente de novo". Nenhuma frase interpola nome, CPF, CID ou texto do corpo.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/lib/psychosocial.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/psychosocial.ts src/lib/psychosocial.test.ts
$G commit -m "feat: add psychosocial screen rules and refusal texts" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Papéis na Equipe, grupo "CAPS" e portão com escolha do CAPS

**Files:**
- Modify: `src/lib/team.ts`, `src/lib/team.test.ts`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`
- Create: `src/modules/psychosocial/PsychosocialGate.tsx`; telas-portão `src/modules/CapsReception.tsx`, `CapsToday.tsx`, `CapsEnrollments.tsx`, `CapsGroups.tsx`, `CapsBeds.tsx`, `CapsMatrix.tsx`, `CapsRaas.tsx`, `CapsPanel.tsx`, `CapsSettings.tsx`
- Test: `src/shell/modules.psychosocial.test.ts`, `src/modules/psychosocial/PsychosocialGate.test.tsx`

**Interfaces:**
- Consumes: `canUsePsychosocial`, `psychosocialRoles`, `PsyRoles`, `PSY_DISABLED`, `PSY_NO_ROLE`, `NO_CAPS_UNIT`, `PSY_UNITS_KEY`, `STATIC_QUERY`, `modalityLabel`, `psyError` (Task 2); `listPsyUnits` (Task 1).
- Produces:
  - `ModuleId` ganha `"caps-reception" | "caps-today" | "caps-enrollments" | "caps-groups" | "caps-beds" | "caps-matrix" | "caps-raas" | "caps-panel" | "caps-settings"`.
  - `PsychosocialGate({ title, sub, allow(r: PsyRoles): boolean, unit?: "team" | "any" | "beds" | "none", children(ctx: { roles: PsyRoles; unit: PsyUnit | null }): ReactNode })` em `src/modules/psychosocial/PsychosocialGate.tsx`. `unit` (padrão `"team"`): lê `listPsyUnits()`; com um CAPS, escolhe sozinho; com vários, mostra o seletor "CAPS" (`<select>` com `nome · modalidade`); `"beds"` lista só `has_beds`; `"none"` não lê. Os filhos montam com `key={unit.id}` e, ao trocar de CAPS, o portão chama `queryClient.removeQueries({ queryKey: [ PSY_DETAIL_KEY ] })`. Lista vazia → `NO_CAPS_UNIT` (com `"beds"`: "nenhum CAPS seu tem leitos — só CAPS III, AD III e AD IV").
  - Telas exportadas com props: `CapsReception({ today?, now?, onGoToSecurity? })`, `CapsToday({ today?, now? })`, `CapsEnrollments({ today?, now?, onGoToSecurity? })`, `CapsGroups({ today?, now? })`, `CapsBeds({ today?, now? })`, `CapsMatrix({ today? })`, `CapsRaas({ today?, onGoToSecurity? })`, `CapsPanel({ today? })`, `CapsSettings({ onGoToSecurity? })`. Nesta task cada uma é só o portão com o corpo vazio (`() => null`); as Tasks 4–14 trocam o corpo.
  - `INVITE_ROLES` ganha `psychosocial_staff` ("Equipe do CAPS") e `psychosocial_lead` ("Coordenação do CAPS"); `PRIVILEGED_ROLES` ganha os dois.

- [ ] **Step 1: Escreva os testes**

`src/shell/modules.psychosocial.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { navGroupsFor } from "./modules";
import { psyUser } from "../test/psychosocialFixtures";

const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).find((g) => g.label === "CAPS")?.items.map((i) => i.id) ?? [];
const TEAM = [ "caps-reception", "caps-today", "caps-enrollments", "caps-groups", "caps-beds", "caps-matrix", "caps-raas" ];

describe("menu do CAPS (módulo 31)", () => {
  it("equipe vê o trabalho do CAPS; coordenação vê também o painel", () => {
    expect(ids(psyUser([ "psychosocial_staff" ]))).toEqual(TEAM);
    expect(ids(psyUser([ "psychosocial_lead" ]))).toEqual([ ...TEAM, "caps-panel" ]);
  });
  it("faturista só a RAAS; analista só o painel; admin painel e cadastro; profissional da UBS nada", () => {
    expect(ids(psyUser([ "billing_clerk" ]))).toEqual([ "caps-raas" ]);
    expect(ids(psyUser([ "analyst" ]))).toEqual([ "caps-panel" ]);
    expect(ids(psyUser([ "municipal_admin" ]))).toEqual([ "caps-panel", "caps-settings" ]);
    expect(ids(psyUser([ "health_professional" ]))).toEqual([]);
  });
  it("sem psychosocial_care ou sem clinical_record, o grupo some", () => {
    expect(ids(psyUser([ "psychosocial_lead" ], [ "clinical_record" ]))).toEqual([]);
    expect(ids(psyUser([ "psychosocial_lead" ], [ "psychosocial_care" ]))).toEqual([]);
  });
});
```

`src/modules/psychosocial/PsychosocialGate.test.tsx`:

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CAPS_AD_III, CAPS_II, psyUser } from "../../test/psychosocialFixtures";
import * as api from "../../lib/api";
import { PsychosocialGate } from "./PsychosocialGate";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listPsyUnits: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount(user = psyUser(), unit: "team" | "beds" = "team") {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(user);
  return renderWithProviders(
    <PsychosocialGate title="Usuários do CAPS" sub="matrícula" unit={unit} allow={(r) => r.staff}>
      {({ unit: u }) => <p>{`dentro de ${u?.name}`}</p>}
    </PsychosocialGate>);
}

describe("portão do CAPS", () => {
  it("interruptor desligado e papel errado: frases", async () => {
    mount(psyUser([ "psychosocial_staff" ], [ "clinical_record" ]));
    expect(await screen.findByText("a atenção psicossocial está desligada nesta cidade")).not.toBeNull();
    cleanup();
    mount(psyUser([ "health_professional" ]));
    expect(await screen.findByText("seu papel não permite usar o CAPS")).not.toBeNull();
  });
  it("sem lotação: frase; um CAPS: entra direto; dois: seletor e troca", async () => {
    vi.mocked(api.listPsyUnits).mockResolvedValue([]);
    mount();
    expect(await screen.findByText(/você não está lotado em nenhum CAPS/)).not.toBeNull();
    cleanup();
    vi.mocked(api.listPsyUnits).mockResolvedValue([ CAPS_II ]);
    mount();
    expect(await screen.findByText("dentro de CAPS II Centro")).not.toBeNull();
    cleanup();
    vi.mocked(api.listPsyUnits).mockResolvedValue([ CAPS_II, CAPS_AD_III ]);
    mount();
    fireEvent.change(await screen.findByLabelText("CAPS"), { target: { value: CAPS_AD_III.id } });
    expect(await screen.findByText("dentro de CAPS AD III Boqueirão")).not.toBeNull();
  });
  it("leitos: só CAPS com leito", async () => {
    vi.mocked(api.listPsyUnits).mockResolvedValue([ CAPS_II ]);
    mount(psyUser(), "beds");
    expect(await screen.findByText(/nenhum CAPS seu tem leitos/)).not.toBeNull();
  });
});
```

Em `src/lib/team.test.ts`, troque "oferece os 21 papéis" por "oferece os 23 papéis" com a lista ordenada acrescida de `"psychosocial_lead"` e `"psychosocial_staff"`, e acrescente:

```ts
// Módulo 31 (ADR 0049): equipe e coordenação do CAPS leem prontuário fechado — privilegiadas.
expect(INVITE_ROLES.find((r) => r.role === "psychosocial_staff")?.label).toBe("Equipe do CAPS");
expect(INVITE_ROLES.find((r) => r.role === "psychosocial_lead")?.label).toBe("Coordenação do CAPS");
expect(isPrivilegedRole("psychosocial_staff")).toBe(true);
expect(isPrivilegedRole("psychosocial_lead")).toBe(true);
```

Em `src/shell/modules.test.ts`, os dois testes que contam grupos escondidos ganham `expect(groups.some((g) => g.label === "CAPS")).toBe(false);` e passam de `NAV_GROUPS.length - 19` para `- 20` e de `- 18` para `- 19`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/shell/modules.psychosocial.test.ts src/modules/psychosocial/PsychosocialGate.test.tsx src/lib/team.test.ts src/shell/modules.test.ts`
Expected: FAIL (grupo, portão e papéis não existem).

- [ ] **Step 3: Implemente**

- `team.ts`: depois de `tfd_committee`, com o comentário "Módulo 31 (ADR 0049): o prontuário do CAPS é fechado à equipe lotada — privilegiadas (step-up para conceder)", os papéis `psychosocial_staff` e `psychosocial_lead`; `PRIVILEGED_ROLES` ganha os dois.
- `modules.ts`: grupo novo **depois de "TFD"**: `{ label: "CAPS", items: [ caps-reception "Acolhimento" "⇲", caps-today "Hoje no CAPS" "◷", caps-enrollments "Usuários do CAPS" "☰", caps-groups "Grupos e oficinas" "◎", caps-beds "Leitos" "▭", caps-matrix "Matriciamento" "⇄", caps-raas "RAAS do mês" "▤", caps-panel "Painel do CAPS" "▦", caps-settings "CAPS: leitos e regras" "⚙" ] }`. Em `navGroupsFor`, com o comentário "Módulo 31 (ADR 0049): `psychosocial_care` E `clinical_record` (canUsePsychosocial); a equipe trabalha, a coordenação também fecha e vê o painel; faturista só baixa a RAAS", `const psyOn = canUsePsychosocial(user); const psy = psychosocialRoles(user);` e os filtros: os sete itens da equipe → `psyOn && psy.staff`, salvo `caps-raas` → `psyOn && (psy.staff || psy.billing)`; `caps-panel` → `psyOn && (psy.lead || psy.admin || psy.analyst)`; `caps-settings` → `psyOn && psy.admin`.
- `catalog.ts`: grupo `"CAPS": { icon: IconBrain, description: "Atenção psicossocial: acolhimento, PTS, grupos, leitos e RAAS" }`; itens: `caps-reception` IconDoorEnter "Encaminhamentos da UBS e demanda espontânea"; `caps-today` IconCalendarEvent "Grupos e atendimentos do dia"; `caps-enrollments` IconUserHeart "Matrícula, PTS, ações, receitas e encerramento"; `caps-groups` IconUsersGroup "Grade semanal e lista de presença"; `caps-beds` IconBed "Acolhimento noturno: leitos, entrada e saída"; `caps-matrix` IconShare "Apoio às equipes da UBS e casos compartilhados"; `caps-raas` IconFileReport "Pendências, fechamento, remessa e fichas espelho"; `caps-panel` IconChartBar "Matrículas, PTS, ações, leitos e RAAS do mês"; `caps-settings` IconSettings "Leitos, limite de noites e prazo do PTS" (ícone que faltar na Task 0 → outro que exista).
- `App.tsx`: os nove `case` em `renderModule`, as telas que pedem step-up com `onGoToSecurity={() => setActive("security")}`.
- `PsychosocialGate`: molde do `TransportGate`, com `useQuery({ queryKey: [ PSY_UNITS_KEY ], queryFn: listPsyUnits, ...STATIC_QUERY, enabled: unit !== "none" && canUsePsychosocial(user) && allow(roles) })`; erro → `psyError`.
- As nove telas: `export function X(props) { return <PsychosocialGate title="<rótulo do menu>" sub="<descrição do catálogo>" unit=… allow={…}>{() => null}</PsychosocialGate>; }` com o mesmo `allow` do menu; `unit`: `"beds"` em `CapsBeds`, `"any"` em `CapsRaas` e `CapsPanel` (o faturista, o admin e o analista veem todos os CAPS; D1), `"none"` em `CapsSettings`, `"team"` nas demais.

- [ ] **Step 4: Rode e veja passar; tipos e guardas**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/shell src/lib/team.test.ts src/modules/psychosocial/PsychosocialGate.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (inclusive `catalog.test.ts`, `router.test.ts` e `Shell.test.tsx`).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/team.ts src/lib/team.test.ts src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.psychosocial.test.ts src/shell/catalog.ts src/App.tsx \
  src/modules/psychosocial/PsychosocialGate.tsx src/modules/psychosocial/PsychosocialGate.test.tsx src/modules/CapsReception.tsx src/modules/CapsToday.tsx \
  src/modules/CapsEnrollments.tsx src/modules/CapsGroups.tsx src/modules/CapsBeds.tsx src/modules/CapsMatrix.tsx src/modules/CapsRaas.tsx src/modules/CapsPanel.tsx src/modules/CapsSettings.tsx
$G commit -m "feat: add CAPS menu group, roles and unit-scoped gate" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Tipo CAPS no cadastro de unidade, leitos e regras da cidade (F-31.1)

**Files:**
- Modify: `src/lib/attendance.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`, `src/modules/CapsSettings.tsx`
- Create: `src/modules/psychosocial/BedsAdmin.tsx`
- Test: `src/modules/attendance/UnitForm.caps.test.tsx`, `src/modules/CapsSettings.test.tsx`

**Interfaces:**
- Consumes: `createUnit`/`updateUnit` com `caps` (Task 1), `listAllUnits`, `listPsyBeds`, `createPsyBed`, `updatePsyBed`, `getPsySettings`, `updatePsySettings`; `unitCapsProblems`, `CAPS_MODALITIES`, `modalityLabel`, `hasBeds`, `settingsProblems`, `psyRefusals` (Task 2); `SensitiveAction`.
- Produces: `CAPS_UNIT_KIND = { value: "caps", label: "CAPS" }` e `unitKindOptions(dentalOn, tfdOn = false, capsOn = false)` em `src/lib/attendance.ts`; `UnitFormValue` ganha `capsModality?: string; serviceClassification?: string`; `UnitForm` e `Units` ganham a prop `capsOn?: boolean`; `BedsAdmin({ unit: HealthUnitRow, onGoToSecurity? })`.

- [ ] **Step 1: Escreva os testes**

`src/modules/attendance/UnitForm.caps.test.tsx` (monta `UnitForm` com `initial` vazio, `neighborhoods: []`, `onSave` espião):

```tsx
it("CAPS só com o interruptor; pede modalidade e classificação; recusa classificação de 2 dígitos", async () => {
  const onSave = vi.fn();
  const { rerender } = render(<UnitForm initial={EMPTY} neighborhoods={[]} busy={false} onSave={onSave} onCancel={() => {}} />);
  expect(screen.queryByRole("option", { name: "CAPS" })).toBeNull();
  rerender(<UnitForm initial={EMPTY} neighborhoods={[]} busy={false} capsOn onSave={onSave} onCancel={() => {}} />);
  fireEvent.change(screen.getByLabelText("Tipo"), { target: { value: "caps" } });
  fireEvent.change(screen.getByLabelText("Modalidade do CAPS"), { target: { value: "caps_ad_iii" } });
  fireEvent.change(screen.getByLabelText("Classificação do serviço 115"), { target: { value: "02" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
  expect(screen.getByText("a classificação do serviço 115 tem 3 dígitos")).not.toBeNull();
  expect(onSave).not.toHaveBeenCalled();
  fireEvent.change(screen.getByLabelText("Classificação do serviço 115"), { target: { value: "002" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
  expect(onSave.mock.calls[0][0]).toMatchObject({ kind: "caps", capsModality: "caps_ad_iii", serviceClassification: "002" });
});
it("unidade CAPS já gravada aparece com o tipo mesmo sem o interruptor", () => {
  render(<UnitForm initial={{ ...EMPTY, kind: "caps", capsModality: "caps_ii", serviceClassification: "002" }} neighborhoods={[]} busy={false} onSave={vi.fn()} onCancel={() => {}} />);
  expect((screen.getByLabelText("Modalidade do CAPS") as HTMLSelectElement).value).toBe("caps_ii");
});
```

(Com os nomes reais dos rótulos "Tipo" e "Salvar" do `UnitForm` conferidos na Task 0; `EMPTY` é o `UnitFormValue` vazio do teste existente do formulário.)

`src/modules/CapsSettings.test.tsx` (sessão `municipal_admin`; `listAllUnits` com um CAPS II e um CAPS AD III; `getPsySettings` com os padrões do contrato §3):

```tsx
it("limites e prazo: mostra os padrões, recusa 0 e grava com step-up", async () => {
  mount(psyUser([ "municipal_admin" ], [ "psychosocial_care", "clinical_record" ]));
  expect((await screen.findByLabelText("CAPS AD III — noites em 30 dias") as HTMLInputElement).value).toBe("14");
  expect((screen.getByLabelText("CAPS III — noites corridas") as HTMLInputElement).value).toBe("7");
  expect((screen.getByLabelText("CAPS III — noites intercaladas") as HTMLInputElement).value).toBe("10");
  expect((screen.getByLabelText("Prazo de revisão do PTS (dias)") as HTMLInputElement).value).toBe("90");
  fireEvent.change(screen.getByLabelText("Prazo de revisão do PTS (dias)"), { target: { value: "0" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar regras" }));
  expect(screen.getByText("o prazo de revisão do PTS vai de 7 a 365 dias")).not.toBeNull();
  expect(api.updatePsySettings).not.toHaveBeenCalled();
  fireEvent.change(screen.getByLabelText("Prazo de revisão do PTS (dias)"), { target: { value: "60" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar regras" }));
  fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(vi.mocked(api.updatePsySettings).mock.calls[0][0].care_plan_review_days).toBe(60));
});
it("leitos só para CAPS com leito; caps_without_beds vira frase", async () => {
  mount(psyUser([ "municipal_admin" ]));
  expect(await screen.findByRole("heading", { name: /Leitos · CAPS AD III Boqueirão/ })).not.toBeNull();
  expect(screen.queryByRole("heading", { name: /Leitos · CAPS II Centro/ })).toBeNull();
  vi.mocked(api.createPsyBed).mockRejectedValueOnce(new ApiError(422, { error: "caps_without_beds" }));
  fireEvent.change(screen.getByLabelText("Rótulo do leito"), { target: { value: "Leito 3" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar leito" }));
  expect(await screen.findByText("só CAPS III, CAPS AD III e CAPS AD IV têm leitos")).not.toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/attendance/UnitForm.caps.test.tsx src/modules/CapsSettings.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `attendance.ts`: `CAPS_UNIT_KIND` e o terceiro parâmetro de `unitKindOptions`.
- `UnitForm.tsx`: com `kind === "caps"`, os campos "Modalidade do CAPS" (`<select>` de `CAPS_MODALITIES` com `modalityLabel`) e "Classificação do serviço 115" (`inputMode="numeric"`, `maxLength={3}`, dica "3 dígitos, como no CNES"); ao salvar, `unitCapsProblems` antes do `onSave`. O tipo CAPS aparece com `capsOn || value.kind === "caps"`.
- `Units.tsx`: `KIND_LABEL` ganha `CAPS_UNIT_KIND`; a coluna "Tipo" mostra `CAPS · <modalidade>` quando houver; salvar repassa `caps: { caps_modality, service_classification }` quando o tipo é CAPS, e `{ caps_modality: null, service_classification: null }` quando a unidade era CAPS e deixou de ser; recusas `caps_modality_mismatch` e `invalid_service_classification` por `psyError`.
- `Attendance.tsx`: onde passa `tfdOn={canUseTransport(user)}`, passa também `capsOn={canUsePsychosocial(user)}`.
- `CapsSettings.tsx` (portão `unit="none"`, `allow={(r) => r.admin}`): painel "Limite do acolhimento noturno" com CAPS III (noites corridas, intercaladas, janela), CAPS AD III e CAPS AD IV (noites, janela) e "Prazo de revisão do PTS (dias)", `settingsProblems` antes de enviar (frase do prazo: "o prazo de revisão do PTS vai de 7 a 365 dias"), gravação em `SensitiveAction` com `requiresStepUp` e `confirmLabel="Confirmar"`; aviso fixo "Os limites do CAPS III estão a confirmar na norma (gate do go-live)." Abaixo, um `BedsAdmin` por unidade de `listAllUnits()` com `kind === "caps"` e `hasBeds(caps_modality)`.
- `BedsAdmin.tsx`: título `Leitos · <nome>`, `DataTable` (rótulo, situação com `Tag`, ação "Desativar"/"Reativar"), formulário "Rótulo do leito" (1–20) + "Adicionar leito"; escrita sem step-up no dashboard (o `SensitiveAction` trata `mfa_required` se o api pedir).

- [ ] **Step 4: Rode e veja passar; tipos e regressões do cadastro**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/attendance src/modules/CapsSettings.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS (os testes existentes de `Units` e `UnitForm` continuam verdes).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/attendance.ts src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/Attendance.tsx \
  src/modules/attendance/UnitForm.caps.test.tsx src/modules/CapsSettings.tsx src/modules/psychosocial/BedsAdmin.tsx src/modules/CapsSettings.test.tsx
$G commit -m "feat: register CAPS units, beds and night-stay limits" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Acolhimento — fila de encaminhamentos, demanda espontânea e acolhimento com ou sem matrícula (F-31.2)

**Files:**
- Modify: `src/modules/CapsReception.tsx`, `src/App.tsx` (`onOpenEnrollment` → `caps-enrollments` com `initialEnrollmentId`)
- Create: `src/modules/psychosocial/PsyPersonPick.tsx`, `src/modules/psychosocial/ReceptionForm.tsx`, `src/modules/psychosocial/EnrollmentFields.tsx`
- Test: `src/modules/CapsReception.test.tsx`

**Interfaces:**
- Consumes: `listPsyReferrals`, `getPsyReferral`, `recordPsyReception`, `getPsyOptions`, `getPsyPrefill`, `lookupPsyPerson`, `findPsyDuplicates`, `searchTerminology` (CID-10, existente); `emptyEnrollmentDraft`, `enrollmentProblems`, `enrollmentInput`, `outcomeLabel`, `priorityLabel`, `psyRefusals`, `isStale`, `NOMINAL_QUERY`, `STATIC_QUERY`, `PSY_DETAIL_KEY`, `TRAIL_NOTE` (Task 2); `PersonStart`, `PersonSource` (24a); `CodeSearch`.
- Produces:
  - `PsyPersonPick({ onPick(patientId: string, name: string): void; onCancel?(): void })` — o `PersonStart` com `psyPersonSource` (sem cadastro: "esta pessoa não tem cadastro — cadastre no balcão de atendimento antes do acolhimento").
  - `EnrollmentFields({ unitId, patientId, options: PsyOptions, value: EnrollmentDraft, onChange(d), problems: Problem[], today })` — origem, CID principal (`CodeSearch` com `searchTerminology(q, "cid10")`) e até 4 secundários, substâncias (caixas; "nenhuma" desmarca as outras), "Coberto por ESF" com o CNES da USF pré-preenchido por `getPsyPrefill` (com o nome da equipe como dica; editável), técnico de referência (`options.staff`). Usado aqui e na Task 6.
  - `ReceptionForm({ unit, patient: { patientId, name }, referralId: string | null, options, today, now, onDone(r: { enrollment_id: string | null }): void, onCancel })`.

- [ ] **Step 1: Escreva o teste** `src/modules/CapsReception.test.tsx` (sessão `psychosocial_staff`; `listPsyUnits` → `[CAPS_II]`; `getPsyOptions` → `OPTIONS`; `listPsyReferrals` → `[referralRow({ reason: "NÃO DEVE APARECER" } as never)]`)

```tsx
it("fila sem motivo; motivo só ao abrir, e sai do cache ao fechar", async () => {
  mount();
  expect(await screen.findByText("Maria da Silva")).not.toBeNull();
  expect(screen.getByText("prioritário")).not.toBeNull();
  expect(screen.queryByText("NÃO DEVE APARECER")).toBeNull();
  expect(api.getPsyReferral).not.toHaveBeenCalled();
  vi.mocked(api.getPsyReferral).mockResolvedValue({ ...referralRow(), reason: "Ideação suicida relatada", requested_by: { name: "Dra. Ana" } });
  fireEvent.click(screen.getByRole("button", { name: "Abrir encaminhamento de Maria da Silva" }));
  expect(await screen.findByText("Ideação suicida relatada")).not.toBeNull();
  expect(screen.getByText("Abrir registra a leitura no prontuário do CAPS.")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Fechar" }));
  expect(screen.queryByText("Ideação suicida relatada")).toBeNull();
});
it("acolher com matrícula: valida o bloco inteiro antes de enviar e manda numa chamada só", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Acolher Maria da Silva" }));
  fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "enrolled" } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar acolhimento" }));
  expect(screen.getByText("informe o CID principal")).not.toBeNull();
  expect(screen.getByText("escolha a origem do usuário")).not.toBeNull();
  expect(api.recordPsyReception).not.toHaveBeenCalled();
  // preenche origem, CID (pela busca mockada), substância "nenhuma" e técnico; envia duas vezes
  fillEnrollment();
  fireEvent.click(screen.getByRole("button", { name: "Registrar acolhimento" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar acolhimento" }));
  await waitFor(() => expect(api.recordPsyReception).toHaveBeenCalledTimes(1));
  const sent = vi.mocked(api.recordPsyReception).mock.calls[0][0];
  expect(sent).toMatchObject({ referral_id: "rf1", outcome: "enrolled", sigtap_code: "0301080232" });
  expect(sent.enrollment?.cid_main).toBe("F32.2");
});
it("sem matrícula não manda o bloco; enrollment_open e referral_not_pending viram frase e relêem a fila", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Acolher Maria da Silva" }));
  fireEvent.change(screen.getByLabelText("Desfecho"), { target: { value: "referred_elsewhere" } });
  vi.mocked(api.recordPsyReception).mockRejectedValueOnce(new ApiError(409, { error: "referral_not_pending" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar acolhimento" }));
  expect(await screen.findByText("este encaminhamento já foi acolhido — a fila foi atualizada")).not.toBeNull();
  expect(vi.mocked(api.recordPsyReception).mock.calls[0][0].enrollment).toBeUndefined();
  expect(api.listPsyReferrals).toHaveBeenCalledTimes(2);
});
it("demanda espontânea: busca a pessoa pelo CPF por POST", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Acolher demanda espontânea" }));
  fireEvent.change(screen.getByLabelText("CPF"), { target: { value: "123.456.789-09" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
  await waitFor(() => expect(api.lookupPsyPerson).toHaveBeenCalledWith("12345678909"));
});
```

(`fillEnrollment` é um ajudante do teste: `searchTerminology` mockado devolve `[{ code: "F32.2", label: "Episódio depressivo grave" }]`; os rótulos reais da busca do `PersonStart` conferidos na Task 0.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsReception.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsReception.tsx` (portão `"team"`, `allow={(r) => r.staff}`): painel "Fila de encaminhamentos" (`DataTable`: paciente (`card: "title"`), de qual UBS, prioridade com `Tag` `warn` para prioritário, quando, ações "Abrir encaminhamento de <nome>" e "Acolher <nome>" (`card: "actions"`)); `NO_CLINICAL_IN_LIST` como nota; botão "Acolher demanda espontânea" abre `PsyPersonPick`. O detalhe do encaminhamento usa `useQuery({ queryKey: [ PSY_DETAIL_KEY, "referral", id ], queryFn, ...NOMINAL_QUERY })` montado só depois do clique; "Fechar" desmonta e remove a chave.
- `ReceptionForm.tsx`: data e hora (padrão `now`), código do acolhimento (`<select>` com `options.procedures` cujo nome contém "ACOLHIMENTO" e instrumento `raas_psi` ou `bpa_i` — D2 traz `procedures` já filtrados pelo CBO; o primeiro é o padrão), "Anotações do acolhimento" (até 4000), "Desfecho" (`enrolled`/`referred_elsewhere`/`no_follow_up`); com `enrolled`, `EnrollmentFields`. Problemas locais todos de uma vez; recusa do api por `psyRefusals` (todas); `isStale` relê a fila e fecha o formulário. Sucesso: "Acolhimento registrado" e, com matrícula, "Matrícula aberta" com o botão "Abrir matrícula" que leva à Task 6 pela prop `onOpenEnrollment?(id)` do `CapsReception` (no `App`, o `CapsReception` recebe `onOpenEnrollment` que troca para `caps-enrollments`; sem rota, o id fica num estado do `App` passado como `initialEnrollmentId` ao `CapsEnrollments`).
- O texto das anotações e o motivo nunca vão para mensagem de erro, `console` ou URL.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsReception.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CapsReception.tsx src/modules/psychosocial/PsyPersonPick.tsx src/modules/psychosocial/ReceptionForm.tsx \
  src/modules/psychosocial/EnrollmentFields.tsx src/modules/CapsReception.test.tsx src/App.tsx
$G commit -m "feat: add CAPS reception queue and enrollment on reception" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Usuários do CAPS — lista, matrícula com trilha, edição, encerramento e PTS com versões (F-31.2, F-31.3)

**Files:**
- Modify: `src/modules/CapsEnrollments.tsx`
- Create: `src/modules/psychosocial/EnrollmentView.tsx`, `src/modules/psychosocial/EnrollmentEdit.tsx`, `src/modules/psychosocial/CloseEnrollment.tsx`, `src/modules/psychosocial/CarePlans.tsx`
- Test: `src/modules/CapsEnrollments.test.tsx`

**Interfaces:**
- Consumes: `listPsyEnrollments`, `getPsyEnrollment`, `updatePsyEnrollment`, `closePsyEnrollment`, `listPsyCarePlans`, `createPsyCarePlan`, `getPsyOptions`; `EnrollmentFields` (Task 5); `carePlanProblems`, `intensityLabel`, `codeLabel`, `psyRefusals`, `NOMINAL_QUERY`, `PSY_DETAIL_KEY`, `TRAIL_NOTE`, `NO_CLINICAL_IN_LIST` (Task 2); `SegmentedControl`.
- Produces: `EnrollmentView({ enrollmentId, unit: PsyUnit, options: PsyOptions, today, now, onClose(), onGoToSecurity? })` com as abas "Matrícula", "PTS", "Ações" (Task 7 encaixa `ActionsPanel`), "Linha do tempo" (Task 14 encaixa `CapsTimeline`) — as abas ainda vazias renderizam `null` nesta task; `CapsEnrollments({ today?, now?, initialEnrollmentId?, onGoToSecurity? })`.

- [ ] **Step 1: Escreva o teste** `src/modules/CapsEnrollments.test.tsx` (sessão `psychosocial_staff`; `listPsyUnits` → `[CAPS_II, CAPS_AD_III]`; `listPsyEnrollments` → `[{ ...enrollmentRow(), cid_main: { code: "F32.2", label: "Episódio depressivo grave" }, notes: "segredo" } as never]`)

```tsx
it("lista ignora campo clínico; detalhe só no clique; trocar de CAPS limpa", async () => {
  const { queryClient } = mount();
  expect(await screen.findByText("Maria da Silva")).not.toBeNull();
  expect(screen.queryByText(/F32/)).toBeNull();
  expect(screen.queryByText("segredo")).toBeNull();
  expect(api.getPsyEnrollment).not.toHaveBeenCalled();
  vi.mocked(api.getPsyEnrollment).mockResolvedValue(enrollment());
  fireEvent.click(screen.getByRole("button", { name: "Abrir Maria da Silva" }));
  expect(await screen.findByText(/F32\.2/)).not.toBeNull();
  expect(screen.getByText("Abrir registra a leitura no prontuário do CAPS.")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("CAPS"), { target: { value: CAPS_AD_III.id } });
  await waitFor(() => expect(screen.queryByText(/F32\.2/)).toBeNull());
  expect(queryClient.getQueriesData({ queryKey: [ PSY_DETAIL_KEY ] })).toHaveLength(0);
});
it("busca por nome é local: nenhuma chamada nova, nada na URL", async () => {
  mount();
  await screen.findByText("Maria da Silva");
  fireEvent.change(screen.getByLabelText("Filtrar por nome"), { target: { value: "jos" } });
  expect(screen.queryByText("Maria da Silva")).toBeNull();
  expect(api.listPsyEnrollments).toHaveBeenCalledTimes(1);
});
it("encerrar: destino permanência é recusado antes; data futura também; envia uma vez", async () => {
  await openDetail();
  fireEvent.click(screen.getByRole("button", { name: "Encerrar matrícula" }));
  fireEvent.change(screen.getByLabelText("Destino do usuário"), { target: { value: "00" } });
  fireEvent.change(screen.getByLabelText("Data de conclusão"), { target: { value: "2026-11-05" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar encerramento" }));
  expect(screen.getByText("escolha o destino do usuário — permanência não encerra a matrícula")).not.toBeNull();
  expect(screen.getByText("a data de encerramento não pode ser futura")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Destino do usuário"), { target: { value: "01" } });
  fireEvent.change(screen.getByLabelText("Data de conclusão"), { target: { value: TODAY31 } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar encerramento" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar encerramento" }));
  await waitFor(() => expect(api.closePsyEnrollment).toHaveBeenCalledTimes(1));
  expect(vi.mocked(api.closePsyEnrollment).mock.calls[0]).toEqual([ "en1", { destination: "01", closed_on: TODAY31 } ]);
});
it("origem e início travados depois da RAAS exportada; origin_locked do api vira frase", async () => {
  vi.mocked(api.getPsyEnrollment).mockResolvedValue(enrollment({ raas_exported: true }));
  await openDetail();
  fireEvent.click(screen.getByRole("button", { name: "Editar matrícula" }));
  expect((screen.getByLabelText("Origem do usuário") as HTMLSelectElement).disabled).toBe(true);
  expect((screen.getByLabelText("Início") as HTMLInputElement).disabled).toBe(true);
});
it("PTS: versões em ordem, nova versão valida tudo e cria a versão seguinte; vencido aparece com aviso", async () => {
  vi.mocked(api.listPsyCarePlans).mockResolvedValue([ carePlan({ version: 2, review_on: "2026-11-01" }), carePlan() ]);
  await openDetail();
  fireEvent.click(screen.getByRole("radio", { name: "PTS" }));
  expect(await screen.findByText("Versão 2 · revisão vencida em 01/11/2026")).not.toBeNull();
  expect(screen.getByText("Versão 1")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Revisar PTS (nova versão)" }));
  fireEvent.change(screen.getByLabelText("Problemas e necessidades"), { target: { value: "curto" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar nova versão" }));
  expect(screen.getByText("problemas e necessidades: de 10 a 4000 caracteres")).not.toBeNull();
  expect(api.createPsyCarePlan).not.toHaveBeenCalled();
});
```

(`openDetail` = `mount()` + clique em "Abrir Maria da Silva"; o rótulo das abas é o que o `SegmentedControl` expõe — conferir na Task 0 se é `radio` ou `button` e usar o mesmo.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsEnrollments.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsEnrollments.tsx`: `SegmentedControl` "Em acompanhamento"/"Encerradas" (`status` open/closed), "Filtrar por nome" local (sem acento, sem caixa), `DataTable` só com **campos de `PsyEnrollmentRow`** lidos campo a campo (nome, CNS mascarado, início, técnico, próximo atendimento, `Tag` "PTS vencido" `warn` quando `care_plan_overdue`) — a linha nunca espalha o objeto da resposta; ação "Abrir <nome>". `initialEnrollmentId` abre o detalhe direto (vindo da Task 5).
- `EnrollmentView.tsx`: `useQuery({ queryKey: [ PSY_DETAIL_KEY, "enrollment", id ], queryFn: () => getPsyEnrollment(id), ...NOMINAL_QUERY })`; título com o nome e foco nele ao abrir; aba "Matrícula" com `KeyValue` (início, origem por `codeLabel`, CID principal e secundários, substâncias, ESF e CNES, técnico, próximo atendimento, permanência noturna ativa com o leito, destino e conclusão quando encerrada) e os botões "Editar matrícula", "Encerrar matrícula" (só `open`) e "Fechar"; ao desmontar, `removeQueries({ queryKey: [ PSY_DETAIL_KEY ] })`.
- `EnrollmentEdit.tsx`: `EnrollmentFields` com `origin` e `started_on` desabilitados quando `raas_exported` e o campo "Próximo atendimento" (data e hora → `next_visit_at`); `PATCH` só com os campos que mudaram.
- `CloseEnrollment.tsx`: "Destino do usuário" (`options.destinations`), "Data de conclusão" (padrão `today`); "00" → `PSY_ERRORS.destination_is_permanence`, depois de `today` → `PSY_ERRORS.closed_on_in_future`, as duas de uma vez; aviso fixo "Encerrar fecha a RAAS do mês com o destino."
- `CarePlans.tsx`: versões em ordem decrescente, cada uma em `Panel` "Versão N" (com " · revisão vencida em dd/mm/aaaa" e `Tag` `warn` quando `review_on < today`), necessidades, objetivos com responsável e prazo, intensidade e revisão; "Revisar PTS (nova versão)" abre o formulário **pré-preenchido com a última versão** (`needs`, `goals`, `intensity`; `review_on` = hoje + `care_plan_review_days`, lido de `getPsySettings` quando disponível, senão 90 dias); `carePlanProblems` antes de enviar ("problemas e necessidades: de 10 a 4000 caracteres", "objetivo: de 3 a 500 caracteres, com responsável", "escolha a frequência proposta", "a revisão precisa ser depois de hoje"); a versão nova aparece no topo depois de gravar. As versões não têm botão de editar (imutáveis).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsEnrollments.test.tsx src/modules/CapsReception.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CapsEnrollments.tsx src/modules/psychosocial/EnrollmentView.tsx src/modules/psychosocial/EnrollmentEdit.tsx \
  src/modules/psychosocial/CloseEnrollment.tsx src/modules/psychosocial/CarePlans.tsx src/modules/CapsEnrollments.test.tsx
$G commit -m "feat: add CAPS enrollment record, closure and versioned care plans" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Ações individuais, familiares e no território — crítica na hora e cancelamento (F-31.4)

**Files:**
- Create: `src/modules/psychosocial/ActionsPanel.tsx`, `src/modules/psychosocial/ActionForm.tsx`, `src/modules/psychosocial/ActionDetail.tsx`
- Modify: `src/modules/psychosocial/EnrollmentView.tsx` (aba "Ações")
- Test: `src/modules/psychosocial/Actions.test.tsx`

**Interfaces:**
- Consumes: `listPsyActions`, `getPsyAction`, `createPsyAction`, `cancelPsyAction`; `emptyActionDraft`, `actionProblems`, `actionInput`, `psyRefusals`, `reasonProblem`, `instrumentLabel`, `placeLabel`, `NOMINAL_QUERY`, `PSY_DETAIL_KEY` (Task 2).
- Produces: `ActionForm({ unit, options, enrollment: { id; name } | null, group?: PsyGroup, initialParticipants?: ParticipantDraft[], now, onSaved(a: PsyAction), onCancel })` (só da aba "Ações"; a presença da Task 9 tem formulário próprio); `ActionDetail({ actionId, roles: PsyRoles, unit, onChanged(), children?(a: PsyAction): ReactNode })` — `children` recebe a ação lida (a Task 8 põe a receita ali); `ActionsPanel({ enrollment, unit, options, roles, now })`.

- [ ] **Step 1: Escreva o teste** `src/modules/psychosocial/Actions.test.tsx` (monta `ActionsPanel` com `enrollment()`, `CAPS_II`, `OPTIONS`, `NOW31`; `listPsyActions` → `[{ ...actionRow(), notes: "NÃO NA LISTA" } as never]`)

```tsx
it("lista sem evolução; detalhe só no clique", async () => {
  mount();
  expect(await screen.findByText("ATENDIMENTO INDIVIDUAL DE PACIENTE EM CAPS")).not.toBeNull();
  expect(screen.queryByText("NÃO NA LISTA")).toBeNull();
  expect(api.getPsyAction).not.toHaveBeenCalled();
  fireEvent.click(screen.getByRole("button", { name: "Abrir ação de 04/11/2026 09:00" }));
  expect(await screen.findByText("Relata melhora do sono.")).not.toBeNull();
});
it("três recusas de uma vez; mês fechado e lead_required", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Registrar ação" }));
  fireEvent.change(screen.getByLabelText("Procedimento"), { target: { value: "0301080208" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar familiar sem cadastro" }));
  fireEvent.change(screen.getByLabelText("Nome do familiar 2"), { target: { value: "Joana Silva" } });
  // local: familiar fora do atendimento familiar
  fireEvent.click(screen.getByRole("button", { name: "Salvar ação" }));
  expect(within(screen.getByTestId("participant-1")).getByText("familiar sem cadastro só entra no atendimento familiar")).not.toBeNull();
  expect(api.createPsyAction).not.toHaveBeenCalled();
  // com o código familiar, o api recusa três coisas
  fireEvent.change(screen.getByLabelText("Procedimento"), { target: { value: "0301080224" } });
  vi.mocked(api.createPsyAction).mockRejectedValueOnce(new ApiError(422, { error: "invalid_action", problems: [
    { code: "cbo_incompatible" }, { code: "quantity_exceeded", path: "quantity" }, { code: "family_only_for_family_care", participant_index: 1 } ] }));
  fireEvent.click(screen.getByRole("button", { name: "Salvar ação" }));
  expect(await screen.findByText("o seu CBO neste CAPS não é compatível com este procedimento no SIGTAP")).not.toBeNull();
  expect(screen.getByText("a quantidade passa do máximo do SIGTAP para este procedimento")).not.toBeNull();
  expect(within(screen.getByTestId("participant-1")).getByText("familiar sem cadastro só entra no atendimento familiar")).not.toBeNull();
  expect((screen.getByLabelText("Nome do familiar 2") as HTMLInputElement).value).toBe("Joana Silva");
  vi.mocked(api.createPsyAction).mockRejectedValueOnce(new ApiError(422, { error: "month_closed" }));
  fireEvent.click(screen.getByRole("button", { name: "Salvar ação" }));
  expect(await screen.findByText("o mês desta ação está fechado — só a coordenação do CAPS corrige")).not.toBeNull();
});
it("cancelar: motivo de 10 a 500; equipe com mês fechado recebe lead_required; a lista é relida", async () => {
  vi.mocked(api.getPsyAction).mockResolvedValue(action({ month_closed: true }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir ação de 04/11/2026 09:00" }));
  fireEvent.click(await screen.findByRole("button", { name: "Cancelar ação" }));
  fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "errado" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
  expect(screen.getByText("o motivo tem de 10 a 500 caracteres")).not.toBeNull();
  vi.mocked(api.cancelPsyAction).mockRejectedValueOnce(new ApiError(403, { error: "lead_required" }));
  fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "registrada no usuário errado" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
  expect(await screen.findByText("só a coordenação do CAPS faz isso com o mês fechado")).not.toBeNull();
});
it("ação cancelada aparece riscada com o motivo só no detalhe, sem botão de cancelar", async () => {
  vi.mocked(api.listPsyActions).mockResolvedValue([ actionRow({ status: "cancelled" }) ]);
  vi.mocked(api.getPsyAction).mockResolvedValue(action({ status: "cancelled", cancel_reason: "registrada no usuário errado" }));
  mount();
  expect(await screen.findByText("cancelada")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Abrir ação de 04/11/2026 09:00" }));
  expect(await screen.findByText("registrada no usuário errado")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Cancelar ação" })).toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial/Actions.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ActionsPanel.tsx`: `DataTable` com data e hora, procedimento (nome; código no `title`), instrumento (`instrumentLabel`), local, quantidade, profissional, participantes (contagem), situação (`Tag`: "registrada" `ok`, "cancelada" `down`, "mês fechado" `neutral`) e "Abrir ação de <dd/mm/aaaa hh:mm>"; nenhuma coluna de evolução. "Registrar ação" abre o `ActionForm` com o usuário da matrícula como primeiro participante.
- `ActionForm.tsx`: "Procedimento" (`<select>` com `options.procedures`, agrupados por instrumento com `<optgroup>`, mostrando "código · nome"), "Data e hora" (`datetime-local`, padrão `now`), "Local" (no CAPS / no território), "Quantidade" (padrão 1; dica "máximo do SIGTAP: N" quando houver), "Número de autorização" (só quando `requires_authorization`), "Evolução" (até 4000), participantes: o matriculado fixo, "Adicionar usuário matriculado" (lista local das matrículas abertas da unidade, lida com `listPsyEnrollments`), "Adicionar familiar sem cadastro" (nome); cada participante numa linha `data-testid="participant-<i>"` com o botão "Retirar" e a frase do problema daquele índice. `actionProblems` antes de enviar; recusas por `psyRefusals` — `participantIndex` na linha, o resto no topo, todas de uma vez; `busy` + `useRef` contra duplo envio; sucesso chama `onSaved` e a lista relê.
- `ActionDetail.tsx`: `useQuery({ queryKey: [ PSY_DETAIL_KEY, "action", id ], ...NOMINAL_QUERY })`; mostra evolução, participantes (familiar como "familiar: <nome>"), autorização e, se cancelada, o motivo; "Cancelar ação" (só `recorded`; para a equipe com `month_closed`, o botão aparece com a dica "mês fechado: só a coordenação" — o api decide) com "Motivo do cancelamento" (10–500); `children?.(a)` abaixo.
- `EnrollmentView.tsx`: a aba "Ações" monta `ActionsPanel`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial/Actions.test.tsx src/modules/CapsEnrollments.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/psychosocial/ActionsPanel.tsx src/modules/psychosocial/ActionForm.tsx src/modules/psychosocial/ActionDetail.tsx \
  src/modules/psychosocial/EnrollmentView.tsx src/modules/psychosocial/Actions.test.tsx
$G commit -m "feat: record and cancel CAPS actions with immediate SIGTAP critique" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Receita a partir da ação, pelo fluxo do 19/19d (F-31.4, decisão 19)

**Files:**
- Create: `src/modules/psychosocial/ActionDocuments.tsx`
- Modify: `src/modules/psychosocial/ActionsPanel.tsx` (passa `ActionDocuments` como `children` do `ActionDetail`)
- Test: `src/modules/psychosocial/ActionDocuments.test.tsx`

**Interfaces:**
- Consumes: `listPsyActionDocuments`, `issuePsyPrescription` (Task 1); `PrescriptionForm`, `ControlledContext` (19c/19d), `CancelDocument`, `printDocument`, `documentError`, `documentSituation`, `issueModeLabel`, `controlledOn`, `usesSimulatedSncr`, `controlledIssuedNotice`, `SNCR_STOCK_KEY` (existentes); `isPrescriber` (Task 2); `hasFeature(user, "digital_signature")`.
- Produces: `ActionDocuments({ action: PsyAction, unit: PsyUnit, today, onGoToSecurity? })`.

- [ ] **Step 1: Escreva o teste** `src/modules/psychosocial/ActionDocuments.test.tsx`

```tsx
it("só para o próprio atendimento individual de prescritor com um matriculado", async () => {
  mount({ action: action({ professional: ME_PSY }) });
  expect(screen.queryByRole("button", { name: "Emitir receita" })).toBeNull();
  cleanup();
  mount({ action: action({ professional: ME_DOCTOR, participants_count: 2 }) });
  expect(screen.queryByRole("button", { name: "Emitir receita" })).toBeNull();
  cleanup();
  mount({ action: action({ professional: { ...ME_DOCTOR, user_id: "u-9" } }) });
  expect(screen.queryByRole("button", { name: "Emitir receita" })).toBeNull();
  cleanup();
  mount({ action: action({ professional: ME_DOCTOR }) });
  expect(await screen.findByRole("button", { name: "Emitir receita" })).not.toBeNull();
});
it("emite pela rota da ação com o mesmo formulário do 19c; prescriber_required vira frase", async () => {
  mount({ action: action({ professional: ME_DOCTOR }) });
  fireEvent.click(await screen.findByRole("button", { name: "Emitir receita" }));
  vi.mocked(api.issuePsyPrescription).mockRejectedValueOnce(new ApiError(422, { error: "prescriber_required" }));
  fillPrescriptionItem(); // texto livre "Sertralina 50 mg", 30 comprimidos, oral, "1 comprimido pela manhã"
  fireEvent.click(screen.getByRole("button", { name: "Emitir" }));
  expect(await screen.findByText("a receita sai só de atendimento individual, com um usuário matriculado, por médico (CBO 225)")).not.toBeNull();
  expect(vi.mocked(api.issuePsyPrescription).mock.calls[0][0]).toBe("ac1");
  expect(api.issueDocument).not.toHaveBeenCalled();
});
it("lista os documentos da ação com situação; imprimir e cancelar usam a unidade do CAPS", async () => {
  vi.mocked(api.listPsyActionDocuments).mockResolvedValue([ prescriptionDoc({ id: "d1" }) ]);
  mount({ action: action({ professional: ME_DOCTOR }) });
  expect(await screen.findByText("Receita")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Imprimir" }));
  await waitFor(() => expect(printDocumentMock).toHaveBeenCalledWith("d1", expect.any(Function), CAPS_II.id));
});
```

(`prescriptionDoc` vem de `src/test/documentFixtures.ts`; `fillPrescriptionItem` usa os rótulos reais do `PrescriptionForm`, conferidos na Task 0; `printDocument` mockado por `vi.mock("../documents/printDocument")`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial/ActionDocuments.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente `ActionDocuments.tsx`**

Mostra-se quando `action.status === "recorded"`, `action.professional.user_id === user.id`, `isPrescriber(action.professional.cbo_code)` e há um participante só, matriculado (a regra final é do api, `prescriber_required`). Lista: `useQuery({ queryKey: [ PSY_DETAIL_KEY, "action-docs", action.id ], queryFn: () => listPsyActionDocuments(action.id), ...NOMINAL_QUERY })` com tipo (`documentTitle`), modo, situação, código curto, "Imprimir" (`printDocument(id, documentError, unit.id)`) e "Cancelar" (`CancelDocument` com `healthUnitId={unit.id}`). "Emitir receita" abre o `PrescriptionForm` com `cboCode = action.professional.cbo_code`, `unitId = unit.id`, `today`, `initial = { items: [], protocolVersionId: "" }`, `controlled` montado como no `ConsultationDocuments` (com `controlledOn(user)`; **sem** `consultationId`, então "usar a última identificação" não aparece) e `onSubmit = (content) => issuePsyPrescription(action.id, content)`; depois de emitir, a frase do 19d (`controlledIssuedNotice`) ou "Receita emitida", relê a lista e o estoque do SNCR. Comentário no topo: "Receita do CAPS (decisão 19): o mesmo formulário do 19c/19d com outro destino; o documento fica fora dos documentos do wpda, da linha do tempo geral e do LEDI (api)." Só `prescription` é oferecido (D8). Sem polling de assinatura (a lista relê ao emitir e ao cancelar; "Atualizar" manual).

- [ ] **Step 4: Rode e veja passar; regressão do 19c/19d; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial src/modules/documents && cd $W && npx tsc --noEmit`
Expected: PASS (os testes de `src/modules/documents` continuam verdes: nenhum componente deles mudou).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/psychosocial/ActionDocuments.tsx src/modules/psychosocial/ActionsPanel.tsx src/modules/psychosocial/ActionDocuments.test.tsx
$G commit -m "feat: issue prescriptions from CAPS actions through the clinical documents flow" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Grupos e oficinas — grade semanal, presença a 375 px — e Hoje no CAPS (F-31.4)

**Files:**
- Modify: `src/modules/CapsGroups.tsx`, `src/modules/CapsToday.tsx`
- Create: `src/modules/psychosocial/GroupForm.tsx`, `src/modules/psychosocial/PresenceList.tsx`
- Test: `src/modules/CapsGroups.test.tsx`, `src/modules/CapsToday.test.tsx`

**Interfaces:**
- Consumes: `listPsyGroups`, `createPsyGroup`, `updatePsyGroup`, `listPsyEnrollments`, `createPsyAction`, `getPsyToday`; `groupKindLabel`, `weekdayLabel`, `psyRefusals`, `textProblem`, `STATIC_QUERY` (Task 2); `EnrollmentView` (Task 6).
- Produces: `PresenceList({ unit, group: PsyGroup, options, date: string, now, onSaved(a: PsyAction), onCancel })`; `GroupForm({ unit, options, initial?: PsyGroup, onSaved, onCancel })`.

- [ ] **Step 1: Escreva os testes**

`src/modules/CapsGroups.test.tsx` (sessão `psychosocial_staff`; `listPsyGroups` → `[group()]`; `listPsyEnrollments` → 25 `enrollmentRow` com ids `en1`..`en25` e nomes `Pessoa 01`..`Pessoa 25`):

```tsx
it("grade por dia da semana, sem dado clínico", async () => {
  mount();
  expect(await screen.findByText("Oficina de artes")).not.toBeNull();
  expect(screen.getByText("quarta · 14:00–16:00")).not.toBeNull();
  expect(screen.getByText("oficina")).not.toBeNull();
});
it("presença: toque duplo desmarca; duplo clique envia uma vez; recusa na linha do participante", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Registrar presença de Oficina de artes" }));
  const p3 = await screen.findByRole("button", { name: "Pessoa 03" });
  fireEvent.click(p3); fireEvent.click(p3);
  expect(p3.getAttribute("aria-pressed")).toBe("false");
  fireEvent.click(screen.getByRole("button", { name: "Pessoa 01" }));
  fireEvent.click(screen.getByRole("button", { name: "Pessoa 02" }));
  expect(screen.getByText("2 presentes")).not.toBeNull();
  vi.mocked(api.createPsyAction).mockRejectedValueOnce(new ApiError(422, { error: "invalid_action",
    problems: [ { code: "enrollment_required", participant_index: 1 } ] }));
  const send = screen.getByRole("button", { name: "Registrar presença" });
  fireEvent.click(send); fireEvent.click(send);
  await waitFor(() => expect(api.createPsyAction).toHaveBeenCalledTimes(1));
  const sent = vi.mocked(api.createPsyAction).mock.calls[0][0];
  expect(sent).toMatchObject({ group_id: "g1", sigtap_code: "0301080216", quantity: 1 });
  expect(sent.participants).toEqual([ { enrollment_id: "en1" }, { enrollment_id: "en2" } ]);
  expect(await within(screen.getByTestId("presence-en2")).findByText("ação de RAAS precisa de ao menos um participante com matrícula aberta")).not.toBeNull();
  expect(screen.getByRole("button", { name: "Pessoa 01" }).getAttribute("aria-pressed")).toBe("true");
});
it("presença em telefone: cartões empilhados, sem tabela, alvo de 44 px", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Registrar presença de Oficina de artes" }));
  const row = await screen.findByRole("button", { name: "Pessoa 01" });
  expect(screen.queryByRole("table")).toBeNull();
  expect(row.style.minHeight).toBe("44px");
});
it("presença acima das vagas avisa mas não bloqueia; nenhuma presença não envia", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Registrar presença de Oficina de artes" }));
  fireEvent.click(await screen.findByRole("button", { name: "Registrar presença" }));
  expect(screen.getByText("marque ao menos uma pessoa presente")).not.toBeNull();
  expect(api.createPsyAction).not.toHaveBeenCalled();
});
it("nova oficina: nome, dia, horário, responsável, vagas e código coletivo", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Novo grupo ou oficina" }));
  fireEvent.change(screen.getByLabelText("Nome"), { target: { value: "Assembleia" } });
  fireEvent.change(screen.getByLabelText("Termina"), { target: { value: "13:00" } });
  fireEvent.change(screen.getByLabelText("Começa"), { target: { value: "14:00" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar grupo" }));
  expect(screen.getByText("o fim precisa ser depois do começo")).not.toBeNull();
  expect(api.createPsyGroup).not.toHaveBeenCalled();
});
```

`src/modules/CapsToday.test.tsx` (`getPsyToday` → `{ date: TODAY31, groups: [{ ...group(), action_id: null }], visits: [{ enrollment_id: "en1", patient: enrollmentRow().patient, next_visit_at: "2026-11-04T14:00:00-03:00", reference: { name: ME_PSY.name } }] }`):

```tsx
it("junta grupos e atendimentos do dia; nada clínico; abre o prontuário pelo clique", async () => {
  mount();
  expect(await screen.findByText("Oficina de artes")).not.toBeNull();
  expect(screen.getByText("14:00 · Maria da Silva · Lia Psicóloga")).not.toBeNull();
  expect(vi.mocked(api.getPsyToday).mock.calls[0]).toEqual([ CAPS_II.id, TODAY31 ]);
  expect(api.getPsyEnrollment).not.toHaveBeenCalled();
  fireEvent.click(screen.getByRole("button", { name: "Abrir Maria da Silva" }));
  await waitFor(() => expect(api.getPsyEnrollment).toHaveBeenCalledWith("en1"));
});
it("grupo com presença já registrada mostra 'presença registrada' e não oferece outra", async () => {
  vi.mocked(api.getPsyToday).mockResolvedValue({ date: TODAY31, groups: [ { ...group(), action_id: "ac7" } ], visits: [] });
  mount();
  expect(await screen.findByText("presença registrada")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Registrar presença de Oficina de artes" })).toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsGroups.test.tsx src/modules/CapsToday.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsGroups.tsx`: grade em `Panel` por dia da semana (segunda a domingo, só os dias com grupo), cada grupo com nome, "<dia> · HH:MM–HH:MM", tipo (`Tag`), responsável, vagas, situação ativa/inativa e os botões "Registrar presença de <nome>" e "Editar"; "Novo grupo ou oficina" abre `GroupForm`.
- `GroupForm.tsx`: "Nome" (3–80), "Tipo" (grupo/oficina/assembleia), "Dia da semana", "Começa", "Termina" (fim depois do começo: "o fim precisa ser depois do começo"), "Responsável" (`options.staff`), "Vagas" (1–200), "Procedimento" (só `options.procedures` com `collective`), "Ativo".
- `PresenceList.tsx`: data (padrão hoje, nunca futura) e a lista das matrículas abertas da unidade em **cartões empilhados** (`display: flex; flex-direction: column`), cada pessoa um `<button type="button" aria-pressed>` com `minHeight: 44`, `width: "100%"`, nome e marca "presente" em texto (não só cor), dentro de `<div data-testid="presence-<enrollment_id>">` que mostra a frase do problema daquele participante; "Filtrar por nome" local no topo; contador "N presentes" (com " · acima das N vagas" quando passar de `capacity`, só aviso); "Evolução do grupo" opcional (até 4000); "Registrar presença" no fim, com `busy` + `useRef`. Envia `createPsyAction` com `group_id`, `sigtap_code` do grupo, `performed_at` = data + `starts_at`, `place: "caps"`, `quantity: 1` e um `{ enrollment_id }` por marcado, na ordem da lista; `participant_index` da recusa é mapeado de volta ao `enrollment_id` pelo índice enviado. Nenhum CID nem dado clínico na lista.
- `CapsToday.tsx`: `getPsyToday(unit.id, today)`; painel "Grupos de hoje" (com "Registrar presença de <nome>" abrindo `PresenceList` ali mesmo, ou "presença registrada" quando `action_id`) e painel "Atendimentos de hoje" com uma linha por visita "HH:MM · <nome> · <técnico>" e "Abrir <nome>" que monta `EnrollmentView` abaixo. Aviso fixo: "Nada daqui gera aviso ou SMS ao usuário."

- [ ] **Step 4: Rode e veja passar; guardas do tema; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsGroups.test.tsx src/modules/CapsToday.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CapsGroups.tsx src/modules/CapsToday.tsx src/modules/psychosocial/GroupForm.tsx src/modules/psychosocial/PresenceList.tsx \
  src/modules/CapsGroups.test.tsx src/modules/CapsToday.test.tsx
$G commit -m "feat: add CAPS weekly groups, phone-ready attendance list and today view" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Leitos — mapa, entrada, saída, limite e reavaliação (F-31.5)

**Files:**
- Modify: `src/modules/CapsBeds.tsx`
- Create: `src/modules/psychosocial/NightStayForm.tsx`, `src/modules/psychosocial/CloseNightStay.tsx`
- Test: `src/modules/CapsBeds.test.tsx`

**Interfaces:**
- Consumes: `getPsyBedBoard`, `startPsyNightStay`, `closePsyNightStay`, `listPsyEnrollments`; `psyRefusals`, `nightLimitText`, `reasonProblem`, `PSY_BEDS_KEY`, `STATIC_QUERY` (Task 2).
- Produces: `NightStayForm({ unit, bed: PsyBed, roles: PsyRoles, now, onSaved, onCancel })`; `CloseNightStay({ stay: PsyNightStay, now, onSaved, onCancel })`.

- [ ] **Step 1: Escreva o teste** `src/modules/CapsBeds.test.tsx` (`listPsyUnits` → `[CAPS_AD_III]`; `getPsyBedBoard` → `bedBoard()`; `listPsyEnrollments` → `[enrollmentRow()]`)

```tsx
it("mapa: leito livre e ocupado com noites usadas, sem dado clínico", async () => {
  mount();
  expect(await screen.findByText("Leito 1")).not.toBeNull();
  expect(screen.getByText("livre")).not.toBeNull();
  expect(screen.getByText("José Pereira · 9 de 14 noites em 30 dias")).not.toBeNull();
});
it("leito tomado relê o mapa; limite: equipe não passa, coordenação passa com motivo", async () => {
  mount();
  await enterStay("Leito 1", "Maria da Silva");
  vi.mocked(api.startPsyNightStay).mockRejectedValueOnce(new ApiError(409, { error: "bed_taken" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar entrada" }));
  expect(await screen.findByText("este leito já está ocupado nesse período")).not.toBeNull();
  expect(api.getPsyBedBoard).toHaveBeenCalledTimes(2);
  vi.mocked(api.startPsyNightStay).mockRejectedValueOnce(new ApiError(422, { error: "night_limit_reached", used: 14, limit: 14 }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar entrada" }));
  expect(await screen.findByText("limite de noites atingido: 14 de 14 na janela de 30 dias — só a coordenação passa, com reavaliação")).not.toBeNull();
  expect(screen.queryByLabelText("Motivo da reavaliação")).toBeNull();
});
it("coordenação: reavaliação com motivo de 10 a 500, enviada uma vez", async () => {
  mount(psyUser([ "psychosocial_lead" ]));
  await enterStay("Leito 1", "Maria da Silva");
  vi.mocked(api.startPsyNightStay).mockRejectedValueOnce(new ApiError(422, { error: "night_limit_reached", used: 14, limit: 14 }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar entrada" }));
  fireEvent.change(await screen.findByLabelText("Motivo da reavaliação"), { target: { value: "curto" } });
  fireEvent.click(screen.getByRole("button", { name: "Passar do limite com reavaliação" }));
  expect(screen.getByText("o motivo tem de 10 a 500 caracteres")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Motivo da reavaliação"), { target: { value: "risco de recaída na semana de festas" } });
  const pass = screen.getByRole("button", { name: "Passar do limite com reavaliação" });
  fireEvent.click(pass); fireEvent.click(pass);
  await waitFor(() => expect(api.startPsyNightStay).toHaveBeenCalledTimes(2));
  expect(vi.mocked(api.startPsyNightStay).mock.calls[1][0].reassessment).toEqual({ reason: "risco de recaída na semana de festas" });
});
it("saída: data e motivo; saída antes da entrada é recusada antes", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Registrar saída de José Pereira" }));
  fireEvent.change(screen.getByLabelText("Saída"), { target: { value: "2026-11-01T08:00" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar saída" }));
  expect(screen.getByText("a saída precisa ser depois da entrada")).not.toBeNull();
  expect(api.closePsyNightStay).not.toHaveBeenCalled();
});
```

(`enterStay(label, name)`: clica "Registrar entrada no <label>", escolhe o usuário pelo `<select>` "Usuário" e preenche "Entrada" e "Saída prevista".)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsBeds.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsBeds.tsx` (portão `"beds"`): data (padrão hoje); grade de cartões, um por leito ativo: rótulo, "livre" (`Tag` `ok`) ou "<nome> · N de M noites em <janela> dias" (`Tag` `warn` quando `nights_used >= limit - 1`), entrada e saída prevista; botões "Registrar entrada no <rótulo>" (livre) e "Registrar saída de <nome>" (ocupado). Leito inativo aparece em "Leitos inativos" sem botões. `STATIC_QUERY`; relê depois de cada escrita e em `bed_taken`/`already_staying`.
- `NightStayForm.tsx`: "Usuário" (matrículas abertas, lista local), "Entrada" (padrão agora), "Saída prevista" (depois da entrada); `night_limit_reached` → frase com `used`/`limit`; com `roles.lead`, aparece "Motivo da reavaliação" (10–500) e "Passar do limite com reavaliação", que repete a mesma chamada com `reassessment: { reason }`; para a equipe, só a frase. Aviso fixo: "Cada noite fechada vira ação de acolhimento noturno na RAAS (api)."
- `CloseNightStay.tsx`: "Saída" (padrão agora; depois de `started_at`, senão "a saída precisa ser depois da entrada"; não futura) e "Motivo da saída" (10–500).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsBeds.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CapsBeds.tsx src/modules/psychosocial/NightStayForm.tsx src/modules/psychosocial/CloseNightStay.tsx src/modules/CapsBeds.test.tsx
$G commit -m "feat: add CAPS bed board with night limit and reassessment" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: RAAS do mês — pendências, situação de rua, fechar e reabrir, remessa e fichas (F-31.6)

**Files:**
- Modify: `src/modules/CapsRaas.tsx`
- Create: `src/modules/psychosocial/MonthRecords.tsx`, `src/modules/psychosocial/RaasExports.tsx`, `src/modules/psychosocial/openSheets.ts`
- Test: `src/modules/CapsRaas.test.tsx`

**Interfaces:**
- Consumes: `getPsyMonth`, `setPsyHomeless`, `closePsyMonth`, `reopenPsyMonth`, `createPsyRaasExport`, `listPsyRaasExports`, `fetchPsyRaasFile`, `fetchPsySheets`; `monthStatusLabel`, `MONTH_PROBLEM_TEXT`, `monthProblemLabel`, `psyRefusals`, `reasonProblem`, `PSY_MONTH_KEY`, `PSY_EXPORTS_KEY`, `STATIC_QUERY` (Task 2); `SensitiveAction`, `saveFile`, `openBlobWindow`, `competenceLabel`.
- Produces: `openSheets(month, q: { unitId; enrollmentId?: string | null }): Promise<{ ok: true } | { ok: false; text: string }>` (molde do `openManifest`; bloqueio de janela: "o navegador bloqueou a janela nova — permita janelas deste site para ver as fichas").

- [ ] **Step 1: Escreva o teste** `src/modules/CapsRaas.test.tsx` (sessão `psychosocial_lead` salvo dito; `listPsyUnits` → `[CAPS_II]`; `getPsyMonth` → `month()`; `listPsyRaasExports` → `[]`)

```tsx
it("pendências por registro, sem dado clínico; situação de rua por registro", async () => {
  mount();
  expect(await screen.findByText("Maria da Silva")).not.toBeNull();
  expect(screen.getByText("situação de rua do mês não informada")).not.toBeNull();
  expect(screen.getByText("1 registro · 1 com pendência · 3 ações")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Situação de rua em novembro de 2026 — Maria da Silva"), { target: { value: "no" } });
  await waitFor(() => expect(api.setPsyHomeless).toHaveBeenCalledWith("mr1", false));
  expect(api.getPsyMonth).toHaveBeenCalledTimes(2);
});
it("fechar com step-up e pendência nova; remessa sem layout; reabrir exportado", async () => {
  mount(psyUser([ "psychosocial_lead" ]), { mfa_verified_at: null });
  vi.mocked(api.getPsyMonth).mockResolvedValue(month({ counts: { records: 1, with_problems: 0, actions: 3 }, records: [ monthRecord({ problems: [], homeless: false }) ] }));
  fireEvent.click(await screen.findByRole("button", { name: "Fechar o mês" }));
  vi.mocked(api.closePsyMonth).mockRejectedValueOnce(new ApiError(422, { error: "month_has_problems" }));
  fillTotpAndConfirm("123456");
  expect(await screen.findByText("o mês tem pendências — resolva a lista antes de fechar")).not.toBeNull();
  expect(api.closePsyMonth).toHaveBeenCalledTimes(1);
  expect(api.getPsyMonth).toHaveBeenCalledTimes(2);
  vi.mocked(api.getPsyMonth).mockResolvedValue(month({ status: "closed", counts: { records: 1, with_problems: 0, actions: 3 }, records: [ monthRecord({ problems: [], status: "closed" }) ] }));
  vi.mocked(api.createPsyRaasExport).mockRejectedValueOnce(new ApiError(422, { error: "raas_layout_unavailable" }));
  rerenderMonth();
  fireEvent.click(await screen.findByRole("button", { name: "Gerar remessa" }));
  expect(await screen.findByText("o layout oficial da remessa RAAS ainda não foi homologado — use as fichas espelho para digitar no aplicativo RAAS")).not.toBeNull();
  expect(screen.getByRole("button", { name: "Fichas espelho do mês (PDF)" })).not.toBeNull();
  vi.mocked(api.reopenPsyMonth).mockRejectedValueOnce(new ApiError(409, { error: "already_exported" }));
  fireEvent.click(screen.getByRole("button", { name: "Reabrir o mês" }));
  fireEvent.change(screen.getByLabelText("Motivo da reabertura"), { target: { value: "ação lançada no mês errado" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByText("a remessa deste mês já foi gerada — não dá mais para reabrir")).not.toBeNull();
});
it("mês aberto: fichas com PRÉVIA; equipe só lê (sem fechar, gerar ou baixar)", async () => {
  mount(psyUser([ "psychosocial_staff" ]));
  await screen.findByText("Maria da Silva");
  expect(screen.queryByRole("button", { name: "Fechar o mês" })).toBeNull();
  expect(screen.queryByRole("button", { name: "Gerar remessa" })).toBeNull();
  expect(screen.queryByRole("button", { name: "Fichas espelho do mês (PDF)" })).toBeNull();
});
it("faturista: só remessas e fichas do CAPS e do mês, sem a lista nominal", async () => {
  vi.mocked(api.listPsyRaasExports).mockResolvedValue([ { id: "rx1", month: MONTH31, competence: "2026-11", created_at: "2026-12-02T10:00:00-03:00", layout_version: "2026.1" } ]);
  mount(psyUser([ "billing_clerk" ]));
  expect(await screen.findByRole("button", { name: "Baixar remessa de novembro de 2026" })).not.toBeNull();
  expect(api.getPsyMonth).not.toHaveBeenCalled();
  expect(screen.queryByText("Maria da Silva")).toBeNull();
});
```

(`fillTotpAndConfirm` usa os rótulos reais do `SensitiveAction` com `stepUpMfa` mockado, como nos testes do 30; `rerenderMonth` invalida a consulta do mês pelo `queryClient` do `renderWithProviders`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsRaas.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsRaas.tsx` (portão `"any"`, `allow={(r) => r.staff || r.billing}`): "Mês de atendimento" (`<select>` do mês corrente e dos 12 anteriores, rótulo "novembro de 2026"); para `staff`, `MonthRecords`; para `lead` e `billing`, `RaasExports`.
- `MonthRecords.tsx`: cabeçalho com a situação (`monthStatusLabel`, `Tag`), "N registro(s) · N com pendência · N ações" e, fechado, "fechado em dd/mm/aaaa por <nome>"; `DataTable` (nome, CNS mascarado, "Situação de rua em <mês> — <nome>" como `<select>` "não informada"/"sim"/"não" — editável só com o mês aberto —, ações do mês, pendências como `Tag`s com `MONTH_PROBLEM_TEXT`); filtro "Só com pendência". Para `lead`: "Fechar o mês" (desabilitado com pendência na tela, com a dica "resolva as N pendências"), em `SensitiveAction` com `requiresStepUp` e `translateError` (`month_has_problems` relê o mês); "Reabrir o mês" (só `closed`) em `SensitiveAction` com step-up e o campo "Motivo da reabertura" (10–500).
- `RaasExports.tsx`: com o mês `closed` e `lead`, "Gerar remessa" (competência de apresentação = a aberta do 22, mostrada pelo `competence` da resposta); `raas_layout_unavailable` mostra a frase em `role="status"` (não é erro: o caminho é a ficha); lista de remessas (`listPsyRaasExports`: competência, quando, versão do layout, "Baixar remessa de <mês>" → `fetchPsyRaasFile` + `saveFile(file, file.filename ?? "raas-<mês>.txt")`); "Fichas espelho do mês (PDF)" e, na linha de cada registro (só `lead`), "Ficha de <nome>" → `openSheets` (o PDF traz "PRÉVIA" com o mês aberto, api). Aviso fixo: "Baixar a remessa ou as fichas fica registrado."

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsRaas.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CapsRaas.tsx src/modules/psychosocial/MonthRecords.tsx src/modules/psychosocial/RaasExports.tsx src/modules/psychosocial/openSheets.ts src/modules/CapsRaas.test.tsx
$G commit -m "feat: add monthly RAAS review, closing, export and mirror sheets" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Matriciamento e casos compartilhados (F-31.7)

**Files:**
- Modify: `src/modules/CapsMatrix.tsx`
- Create: `src/modules/psychosocial/MatrixForm.tsx`, `src/modules/psychosocial/MatrixCaseForm.tsx`
- Test: `src/modules/CapsMatrix.test.tsx`

**Interfaces:**
- Consumes: `listPsyMatrixSupports`, `getPsyMatrixSupport`, `createPsyMatrixSupport`, `addPsyMatrixCase`; `PsyPersonPick` (Task 5); `textProblem`, `psyRefusals`, `PSY_MATRIX_KEY`, `NOMINAL_QUERY`, `PSY_DETAIL_KEY` (Task 2).
- Produces: `MatrixForm({ unit, options, today, onSaved(s: PsyMatrixSupport), onCancel })`; `MatrixCaseForm({ support: PsyMatrixSupport, onSaved(s), onCancel })`.

- [ ] **Step 1: Escreva o teste** `src/modules/CapsMatrix.test.tsx`

```tsx
it("lista do mês sem texto compartilhado; detalhe no clique", async () => {
  vi.mocked(api.listPsyMatrixSupports).mockResolvedValue([ { id: "ms1", health_team: { id: "ht1", name: "ESF Vila Verde", unit_name: "UBS Vila Verde" },
    held_on: "2026-11-03", topic: "Casos de uso de álcool", cases_count: 1 } ]);
  vi.mocked(api.getPsyMatrixSupport).mockResolvedValue({ id: "ms1", health_team: { id: "ht1", name: "ESF Vila Verde", unit_name: "UBS Vila Verde" },
    held_on: "2026-11-03", topic: "Casos de uso de álcool", participants: [ { user_id: "u-1", name: ME_PSY.name } ],
    cases: [ { id: "mc1", patient: { patient_id: "pt1", name: "Maria da Silva" }, shared_text: "Combinar visita domiciliar com a ACS." } ] });
  mount();
  expect(await screen.findByText("ESF Vila Verde · UBS Vila Verde")).not.toBeNull();
  expect(screen.queryByText("Combinar visita domiciliar com a ACS.")).toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Abrir matriciamento de 03/11/2026" }));
  expect(await screen.findByText("Combinar visita domiciliar com a ACS.")).not.toBeNull();
  expect(screen.getByText("A equipe matriciada lê este texto; o resto do prontuário do CAPS continua fechado.")).not.toBeNull();
});
it("registrar: equipe, data, tema, participantes e código BPA-C; tudo de uma vez", async () => {
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Registrar matriciamento" }));
  fireEvent.click(screen.getByRole("button", { name: "Salvar matriciamento" }));
  expect(screen.getByText("escolha a equipe da UBS")).not.toBeNull();
  expect(screen.getByText("tema: de 3 a 200 caracteres")).not.toBeNull();
  expect(screen.getByText("marque ao menos um profissional do CAPS")).not.toBeNull();
  expect(api.createPsyMatrixSupport).not.toHaveBeenCalled();
  fireEvent.change(screen.getByLabelText("Equipe da UBS"), { target: { value: "ht1" } });
  fireEvent.change(screen.getByLabelText("Tema"), { target: { value: "Casos de uso de álcool" } });
  fireEvent.click(screen.getByRole("checkbox", { name: ME_PSY.name }));
  fireEvent.click(screen.getByRole("button", { name: "Salvar matriciamento" }));
  await waitFor(() => expect(vi.mocked(api.createPsyMatrixSupport).mock.calls[0][0]).toMatchObject({
    health_unit_id: CAPS_II.id, health_team_id: "ht1", sigtap_code: "0301080313", participants_user_ids: [ "u-1" ] }));
});
it("caso: paciente pelo CPF e texto compartilhado de 10 a 2000", async () => {
  await openSupport();
  fireEvent.click(screen.getByRole("button", { name: "Adicionar caso" }));
  pickPerson("pt1", "Maria da Silva");
  fireEvent.change(screen.getByLabelText("Texto compartilhado com a equipe"), { target: { value: "curto" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar caso" }));
  expect(screen.getByText("texto compartilhado: de 10 a 2000 caracteres")).not.toBeNull();
  expect(api.addPsyMatrixCase).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsMatrix.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsMatrix.tsx` (portão `"team"`): mês (padrão o corrente); `DataTable` de `listPsyMatrixSupports` (data, "equipe · unidade", tema, casos (contagem), "Abrir matriciamento de <data>"); detalhe com `NOMINAL_QUERY` (participantes, casos com nome e texto compartilhado, a nota fixa da leitura pela equipe matriciada e "Adicionar caso").
- `MatrixForm.tsx`: "Equipe da UBS" (`options.health_teams`, "nome · unidade"), "Data" (padrão hoje, não futura), "Tema" (3–200), "Profissionais do CAPS" (caixas de `options.staff`), "Procedimento" (`options.procedures` com `bpa_c`; o primeiro de nome "MATRICIAMENTO" é o padrão); aviso fixo "Vai ao BPA-C como ação da equipe, sem paciente." Frases: "escolha a equipe da UBS", "tema: de 3 a 200 caracteres", "marque ao menos um profissional do CAPS".
- `MatrixCaseForm.tsx`: `PsyPersonPick` e "Texto compartilhado com a equipe" (10–2000, contador); aviso fixo "Não escreva CID, substância nem evolução aqui: a UBS lê este texto." (orientação; o api não filtra).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/CapsMatrix.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CapsMatrix.tsx src/modules/psychosocial/MatrixForm.tsx src/modules/psychosocial/MatrixCaseForm.tsx src/modules/CapsMatrix.test.tsx
$G commit -m "feat: record CAPS matrix support with shared cases" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Cartão do vínculo na tela do paciente da UBS e "Encaminhar ao CAPS" (F-31.7, F-31.2)

**Files:**
- Create: `src/modules/psychosocial/PsychosocialLinkCard.tsx`, `src/modules/psychosocial/ReferralForm.tsx`
- Modify: `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/clinicalRecord/JustifiedRecord.tsx`, `src/lib/psychosocial.ts`
- Test: `src/modules/psychosocial/PsychosocialLinkCard.test.tsx`

**Interfaces:**
- Consumes: `getPsyLink`, `createPsyReferral`, `listActiveUnits` (existente; CAPS = `kind === "caps"`); `canUsePsychosocial`, `priorityLabel`, `textProblem`, `psyRefusals`, `NOMINAL_QUERY`, `PSY_DETAIL_KEY` (Task 2).
- Produces: `PsychosocialLinkCard({ patientId: string; canRefer: boolean })`; `PatientPanel` ganha a prop `psychosocial?: { patientId: string; canRefer: boolean }`, renderizada no Resumo, logo depois dos problemas; `canSeePsychosocialLink(user): boolean` em `src/lib/psychosocial.ts` (= `canUsePsychosocial(user)` e papel `health_professional`).

- [ ] **Step 1: Escreva o teste** `src/modules/psychosocial/PsychosocialLinkCard.test.tsx`

```tsx
it("sem matrícula: só o botão de encaminhar", async () => {
  vi.mocked(api.getPsyLink).mockResolvedValue({ enrolled: false });
  mount({ patientId: "pt1", canRefer: true });
  expect(await screen.findByRole("button", { name: "Encaminhar ao CAPS" })).not.toBeNull();
  expect(screen.queryByText(/em acompanhamento/)).toBeNull();
});
it("com matrícula: vínculo, técnico, contato e casos da equipe; nunca CID nem PTS mesmo que venham", async () => {
  vi.mocked(api.getPsyLink).mockResolvedValue({ enrolled: true, caps: { id: "caps2", name: "CAPS II Centro", phone: "(41) 3333-4444" },
    since: "2026-09-10", reference: { name: "Lia Psicóloga" },
    shared_cases: [ { held_on: "2026-11-03", caps_name: "CAPS II Centro", shared_text: "Combinar visita domiciliar com a ACS." } ],
    cid_main: "F32.2", care_plan: "segredo" } as never);
  mount({ patientId: "pt1", canRefer: true });
  expect(await screen.findByText("Em acompanhamento no CAPS II Centro desde 10/09/2026")).not.toBeNull();
  expect(screen.getByText("Técnico de referência: Lia Psicóloga · (41) 3333-4444")).not.toBeNull();
  expect(screen.getByText("Combinar visita domiciliar com a ACS.")).not.toBeNull();
  expect(screen.queryByText(/F32/)).toBeNull();
  expect(screen.queryByText("segredo")).toBeNull();
  expect(screen.queryByRole("button", { name: "Encaminhar ao CAPS" })).toBeNull();
});
it("encaminhar: CAPS, motivo de 10 a 2000 e prioridade; uma chamada; motivo nunca na tela depois", async () => {
  vi.mocked(api.getPsyLink).mockResolvedValue({ enrolled: false });
  vi.mocked(api.listActiveUnits).mockResolvedValue([ { id: "ubs1", name: "UBS Vila Verde", kind: "ubs" }, { id: "caps2", name: "CAPS II Centro", kind: "caps", caps_modality: "caps_ii" } ]);
  mount({ patientId: "pt1", canRefer: true });
  fireEvent.click(await screen.findByRole("button", { name: "Encaminhar ao CAPS" }));
  expect(screen.queryByRole("option", { name: /UBS Vila Verde/ })).toBeNull();
  fireEvent.change(screen.getByLabelText("Motivo do encaminhamento"), { target: { value: "curto" } });
  fireEvent.click(screen.getByRole("button", { name: "Enviar encaminhamento" }));
  expect(screen.getByText("motivo do encaminhamento: de 10 a 2000 caracteres")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Motivo do encaminhamento"), { target: { value: "Ideação suicida relatada na consulta" } });
  fireEvent.change(screen.getByLabelText("Prioridade"), { target: { value: "priority" } });
  const send = screen.getByRole("button", { name: "Enviar encaminhamento" });
  fireEvent.click(send); fireEvent.click(send);
  await waitFor(() => expect(api.createPsyReferral).toHaveBeenCalledTimes(1));
  expect(vi.mocked(api.createPsyReferral).mock.calls[0][0]).toEqual({ patient_id: "pt1", health_unit_id: "caps2", reason: "Ideação suicida relatada na consulta", priority: "priority" });
  expect(await screen.findByText("Encaminhamento enviado ao CAPS II Centro.")).not.toBeNull();
  expect(screen.queryByText("Ideação suicida relatada na consulta")).toBeNull();
});
it("PatientPanel sem a prop não lê o vínculo", () => {
  render(<PatientPanel record={recordFixture()} onOpenConsultation={() => {}} />);
  expect(api.getPsyLink).not.toHaveBeenCalled();
});
```

(`recordFixture` vem de `src/test/consultationFixtures.ts`; conferir o nome real na Task 0.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial/PsychosocialLinkCard.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `PsychosocialLinkCard.tsx`: `useQuery({ queryKey: [ PSY_DETAIL_KEY, "link", patientId ], queryFn: () => getPsyLink(patientId), ...NOMINAL_QUERY })` — lê só os campos do contrato §7 (`enrolled`, `caps`, `since`, `reference`, `shared_cases`), nunca espalha a resposta; título "Atenção psicossocial"; com `enrolled`, "Em acompanhamento no CAPS <nome> desde <dd/mm/aaaa>", "Técnico de referência: <nome> · <telefone>" e, havendo casos, "Matriciamento com a sua equipe" com data, CAPS e texto; sem `enrolled` e com `canRefer`, "Encaminhar ao CAPS" abre `ReferralForm`. 403 `feature_disabled` some o cartão (não mostra erro); outra recusa vira `psyError` discreto.
- `ReferralForm.tsx`: "CAPS" (`listActiveUnits()` filtrado por `kind === "caps"`, "nome · modalidade"), "Motivo do encaminhamento" (10–2000, contador), "Prioridade" (rotina/prioritário); `busy` + `useRef`; sucesso fecha o formulário, mostra "Encaminhamento enviado ao <CAPS>." e **descarta o motivo** do estado; aviso fixo "O CAPS recebe na fila de acolhimento. O usuário não recebe aviso."
- `PatientPanel.tsx`: prop `psychosocial?`; no Resumo, depois de `besideProblems`, `{psychosocial && <PsychosocialLinkCard patientId={psychosocial.patientId} canRefer={psychosocial.canRefer} />}`.
- `ConsultationWorkspace.tsx` e `JustifiedRecord.tsx`: passam `psychosocial={canSeePsychosocialLink(user) ? { patientId: record.data.patient.id, canRefer: true } : undefined}` (com o nome real da variável do prontuário em cada um).

- [ ] **Step 4: Rode e veja passar; regressão da consulta e do prontuário; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial/PsychosocialLinkCard.test.tsx src/modules/consultation src/modules/clinicalRecord && cd $W && npx tsc --noEmit`
Expected: PASS (os testes existentes não ligam `psychosocial_care`, então o cartão não aparece neles).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/psychosocial/PsychosocialLinkCard.tsx src/modules/psychosocial/ReferralForm.tsx src/modules/consultation/PatientPanel.tsx \
  src/modules/consultation/ConsultationWorkspace.tsx src/modules/clinicalRecord/JustifiedRecord.tsx src/lib/psychosocial.ts src/modules/psychosocial/PsychosocialLinkCard.test.tsx
$G commit -m "feat: show the CAPS link on the primary care patient and refer to CAPS" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Linha do tempo do CAPS, marcador na linha do tempo geral e painel (F-31.8)

**Files:**
- Create: `src/modules/psychosocial/CapsTimeline.tsx`, `src/modules/psychosocial/CapsTimeline.test.tsx`
- Modify: `src/modules/psychosocial/EnrollmentView.tsx` (aba "Linha do tempo"), `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/CapsPanel.tsx`
- Test: `src/modules/CapsPanel.test.tsx`

**Interfaces:**
- Consumes: `getPsyEnrollmentTimeline`, `getPsyDashboard`; `entryWhen`, `entryWhere`, `mergePages`, `timelineKindLabel` (`src/lib/timeline.ts`); `countLabel`, `modalityLabel`, `instrumentLabel`, `codeLabel`, `monthStatusLabel`, `PSY_DASHBOARD_KEY`, `STATIC_QUERY`, `NOMINAL_QUERY`, `PSY_DETAIL_KEY` (Task 2); `StatTile`.
- Produces: `TimelineKind` ganha `"psychosocial_link"` (D18); `TIMELINE_KINDS` ganha `{ key: "psychosocial_link", label: "CAPS" }`; `timelineKinds(examTracking, procedureBilling = false, pharmacy = false, vaccination = false, transport = false, psychosocial = false)`; `CapsTimeline({ enrollmentId })`.

- [ ] **Step 1: Escreva os testes**

`src/modules/psychosocial/CapsTimeline.test.tsx`:

```tsx
it("linha do tempo do CAPS: entradas da fonte psychosocial em páginas, sem abrir detalhe", async () => {
  vi.mocked(api.getPsyEnrollmentTimeline).mockResolvedValueOnce({ entries: [ entry({ id: "e1", kind: "psychosocial_action", title: "Atendimento individual" }) ], next_before: "c1" })
    .mockResolvedValueOnce({ entries: [ entry({ id: "e2", kind: "psychosocial_care_plan", title: "PTS versão 1" }) ], next_before: null });
  mount({ enrollmentId: "en1" });
  expect(await screen.findByText("Atendimento individual")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
  expect(await screen.findByText("PTS versão 1")).not.toBeNull();
  expect(vi.mocked(api.getPsyEnrollmentTimeline).mock.calls[1]).toEqual([ "en1", "c1" ]);
  expect(screen.queryByRole("button", { name: "Carregar mais" })).toBeNull();
});
it("marcador do CAPS na geral: rótulo CAPS, filtro só com o interruptor, sem detalhe", () => {
  expect(timelineKindLabel("psychosocial_link")).toBe("CAPS");
  expect(timelineKinds(false).some((k) => k.key === "psychosocial_link")).toBe(false);
  expect(timelineKinds(false, false, false, false, false, true).some((k) => k.key === "psychosocial_link")).toBe(true);
  expect(detailTarget(entry({ kind: "psychosocial_link", detail_ref: null }), true)).toBeNull();
});
```

(`entry(o)` vem de `src/test/timelineFixtures.ts`; conferir o nome real na Task 0.)

`src/modules/CapsPanel.test.tsx` (`getPsyDashboard` → `{ month: MONTH31, active_by_unit: [ { unit: { id: "caps2", name: "CAPS II Centro", caps_modality: "caps_ii" }, count: 37 }, { unit: { id: "capsad3", name: "CAPS AD III Boqueirão", caps_modality: "caps_ad_iii" }, count: "lt5" } ], new_in_month: "lt5", closed_by_destination: [ { destination: "01", count: 6 } ], care_plans_overdue: 8, actions_by_instrument: { raas_psi: 410, bpa_c: 12, bpa_i: "lt5" }, bed_occupancy_pct: 72, raas: { open: 30, closed: 7, exported: 0 } }`):

```tsx
it("só contagens, com menos de 5; nenhum nome", async () => {
  mount(psyUser([ "analyst" ]));
  expect(await screen.findByText("CAPS II Centro · CAPS II")).not.toBeNull();
  expect(screen.getByText("37")).not.toBeNull();
  expect(screen.getAllByText("menos de 5").length).toBe(3);
  expect(screen.getByText("alta")).not.toBeNull();
  expect(screen.getByText("72%")).not.toBeNull();
  expect(vi.mocked(api.getPsyDashboard).mock.calls[0][0]).toEqual({ month: MONTH31, unitId: null });
});
it("filtro por CAPS e por mês vão na consulta", async () => {
  mount(psyUser([ "psychosocial_lead" ]));
  fireEvent.change(await screen.findByLabelText("CAPS do painel"), { target: { value: "caps2" } });
  await waitFor(() => expect(vi.mocked(api.getPsyDashboard).mock.calls.at(-1)?.[0]).toEqual({ month: MONTH31, unitId: "caps2" }));
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial/CapsTimeline.test.tsx src/modules/CapsPanel.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CapsTimeline.tsx`: páginas com `useInfiniteQuery({ queryKey: [ PSY_DETAIL_KEY, "timeline", enrollmentId ], ...NOMINAL_QUERY })`, `mergePages`, cada entrada com quando (`entryWhen`), onde/quem (`entryWhere`), tipo (`timelineKindLabel`, cru quando desconhecido) e o título do api; "Carregar mais" enquanto houver `next_before`; nota fixa "Só a equipe do CAPS vê esta linha do tempo." `EnrollmentView` monta na aba "Linha do tempo".
- `timeline.ts`: o kind e o rótulo, `timelineKinds` com o sexto parâmetro e o comentário "Módulo 31 (D18): marcador sem texto 'Acompanhamento no CAPS <nome>', detail_ref nulo; o resto do CAPS nunca entra na geral". `detailTarget` já devolve `null` com `detail_ref` nulo (o teste prova).
- `PatientTimeline.tsx`: passa `canUsePsychosocial(user)` como sexto argumento de `timelineKinds`.
- `CapsPanel.tsx` (portão `"any"`, `allow={(r) => r.lead || r.admin || r.analyst}`): "Mês" e "CAPS do painel" (todos ou um); `StatTile`s "Matrículas novas no mês", "PTS vencidos", "Ocupação dos leitos" (`bed_occupancy_pct` com "%"; nulo → "sem leitos"); `DataTable`s "Matrículas ativas por CAPS" ("nome · modalidade", contagem), "Encerradas por destino" (`codeLabel` de `getPsyOptions` não está disponível para analista → usar um mapa local dos destinos do contrato com o código cru como reserva; D19), "Ações por instrumento", "RAAS do mês" (aberto/fechado/remessa gerada); todos com `countLabel`; nota "Contagens de 1 a 4 aparecem como “menos de 5”."

- [ ] **Step 4: Rode e veja passar; regressão da linha do tempo; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod31 src/modules/psychosocial src/modules/CapsPanel.test.tsx src/lib/timeline* src/modules/timeline && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/psychosocial/CapsTimeline.tsx src/modules/psychosocial/CapsTimeline.test.tsx src/modules/psychosocial/EnrollmentView.tsx src/lib/timeline.ts \
  src/modules/timeline/PatientTimeline.tsx src/modules/CapsPanel.tsx src/modules/CapsPanel.test.tsx
$G commit -m "feat: add CAPS-only timeline, general timeline marker and CAPS panel" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Suíte, build, varredura, revisão e prova no navegador (api real na 3053)

**Files:** nenhum novo (correções que a prova pedir entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod31
$R/bin/test dashboard -w dashboard-mod31
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **18 arquivos de teste novos**: `src/lib/api.psychosocial.test.ts`, `src/lib/psychosocial.test.ts`, `src/shell/modules.psychosocial.test.ts`, `src/modules/psychosocial/PsychosocialGate.test.tsx`, `src/modules/attendance/UnitForm.caps.test.tsx`, `src/modules/CapsSettings.test.tsx`, `src/modules/CapsReception.test.tsx`, `src/modules/CapsEnrollments.test.tsx`, `src/modules/psychosocial/Actions.test.tsx`, `src/modules/psychosocial/ActionDocuments.test.tsx`, `src/modules/CapsGroups.test.tsx`, `src/modules/CapsToday.test.tsx`, `src/modules/CapsBeds.test.tsx`, `src/modules/CapsRaas.test.tsx`, `src/modules/CapsMatrix.test.tsx`, `src/modules/psychosocial/PsychosocialLinkCard.test.tsx`, `src/modules/psychosocial/CapsTimeline.test.tsx`, `src/modules/CapsPanel.test.tsx`. Contagem que dobra = artefato de build descoberto pelo vitest: confira antes de seguir.

- [ ] **Step 2: Varredura de dado sensível e de releitura**

```bash
cd $W
grep -rnE "refetchInterval" src/modules/psychosocial src/modules/Caps*.tsx && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" src/modules/psychosocial src/modules/Caps*.tsx src/lib/psychosocial.ts && echo "ERRO: storage/log" || echo ok-storage
sed -n '/Atenção psicossocial (módulo 31/,$p' src/lib/api.ts | grep -nE "params\.(set|append)\(|URLSearchParams\(\{" | grep -vE '"?(health_unit_id|status|date|month|before|enrollment_id)"?' && echo "revisar" || echo ok-url
grep -rnE "\.\.\.(row|r|item|e|data|link)\b" src/modules/psychosocial/*List*.tsx src/modules/psychosocial/PsychosocialLinkCard.tsx src/modules/CapsEnrollments.tsx 2>/dev/null && echo "revisar: resposta espalhada" || echo ok-spread
grep -rnE "psychosocial|Psy" src/modules/documents src/modules/timeline/TimelineDetail.tsx && echo "revisar: CAPS em documentos gerais" || echo ok-docs
```

Expected: `ok-polling`, `ok-storage`, `ok-url`, `ok-spread`, `ok-docs`.

- [ ] **Step 3: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, em "nada clínico em lista" (toda lista lê campo a campo), no `PrescriptionForm` reusado (nada do 19c/19d mudou), no `PatientPanel` (o cartão só com o interruptor) e no cadastro de unidade (o CEO, a Gestão e as unidades comuns continuam iguais).

- [ ] **Step 4: Prova no navegador** (só com o api do 31 mergeado ou no worktree dele, na 3053, e o banco de dev migrado e semeado **com autorização do usuário**)

Suba o Vite do worktree na rede do compose, porta **5211**, com proxy para `http://api:3053` (mesmo comando da prova do 30, trocando porta e worktree), abra `curitiba.localhost:5211/dashboard/` e percorra, com `psychosocial_care` ligado:
1. Equipe: convidar `psychosocial_staff` e `psychosocial_lead` (step-up). Unidades: criar "CAPS II Centro" (CAPS II, 115 "002", CNES) e "CAPS AD III" com leitos; o vínculo CNES da equipe nos CAPS (Profissionais). "CAPS: leitos e regras": limites e prazo do PTS (step-up), dois leitos no AD III.
2. UBS: profissional abre um paciente na consulta → cartão "Atenção psicossocial" sem vínculo → "Encaminhar ao CAPS" (prioritário).
3. Acolhimento: o encaminhamento na fila sem motivo; abrir (trilha) → acolher com matrícula (CID, substância, ESF pré-preenchida, técnico); uma demanda espontânea pelo CPF com "encaminhar a outro serviço"; a segunda matrícula do mesmo paciente (`enrollment_open`).
4. Usuários do CAPS: lista sem CID; abrir; PTS v1 e revisão v2; ação individual; ação com três recusas; atendimento familiar com familiar sem cadastro; cancelar ação; receita de controlado por médico (19d, SNCR simulado) e a recusa `prescriber_required` para a psicóloga; encerrar com destino.
5. Grupos e oficinas: oficina na quarta; **presença em 375×812** (`resize_window` mobile, recarregar) com 10+ pessoas, toque duplo, envio; Hoje no CAPS.
6. Leitos: entrada, `bed_taken` em duas abas, `night_limit_reached` (semente com noites usadas) pela equipe e passagem pela coordenação; saída.
7. RAAS do mês: pendências, situação de rua, fechar com step-up, gerar remessa (`raas_layout_unavailable` enquanto `layout_version: null`), fichas espelho (PRÉVIA com o mês aberto), reabrir; faturista baixando as fichas.
8. Matriciamento com a ESF do paciente e um caso; de volta à UBS, o cartão mostra o vínculo e o caso; um profissional de outra unidade não vê o caso.
9. Linha do tempo geral do paciente com o marcador "CAPS" sem detalhe; linha do tempo do CAPS na matrícula; painel com "menos de 5".
10. Tablet (1024×768) e só teclado nos fluxos 3, 4 e 7; telefone (375×812) nas listas (cartões). Ao terminar, `resize_window` preset desktop.

Anote cada passo com o que viu. Achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai para o usuário (não vira tarefa).

- [ ] **Step 5: Board e encerramento**

Cards do dashboard prontos: F-31.2..F-31.7 → **Done** se o api já terminou o deles; F-31.1 espera o maintenance e F-31.8 o wpda (o último app move). **Verified** só depois da prova do Step 4, com o usuário. Merge, push e tag só com autorização. Worktree: `$G -C $R/apps/dashboard worktree remove $W` depois do merge.

---

## Divergências propostas ao contrato

Cada uma diz o que a tela precisa, o que o plano assume e quem decide. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D1 — CAPS de lotação.** Toda rota da equipe recebe `health_unit_id`, mas a tela não sabe em quais CAPS a pessoa está lotada (o `/session` não traz `professional_links`). Proposta: `GET /psychosocial/units` → `{ units: [ { id, name, caps_modality, has_beds } ] }`: para `psychosocial_staff`/`psychosocial_lead`, os CAPS com vínculo ativo; para `billing_clerk`, `municipal_admin` e `analyst`, todos os CAPS ativos.
- **D2 — Opções da tela.** As listas de `config/psychosocial/raas.yml`, os procedimentos permitidos ("vêm do SIGTAP pelo instrumento, nunca de lista fixa") e a equipe para técnico de referência e responsável não têm rota. Proposta: `GET /psychosocial/options?health_unit_id` → `{ origins, destinations, substances: [ { code, label } ], procedures: [ { code, name, instrument, max_quantity, requires_authorization, family_care, collective } ], staff: [ <pro> ], health_teams: [ { id, name, ine, unit: { id, name } } ], me: <pro>|null, review }`, com `procedures` já filtrados pelo CBO de quem pede naquele CAPS; `family_care`/`collective` marcados pelo api a partir do código (a tela não decide por nome).
- **D3 — Busca de paciente pelo CAPS.** Acolhimento por demanda espontânea e caso de matriciamento pedem `patient_id`. Proposta: `POST /psychosocial/person_lookup { cpf }` e `POST /psychosocial/patients/duplicates { full_name, birth_date, mother_name? }` (forma e serviço do 24a/30 D1), portão `psychosocial_care` + `psychosocial_staff`, com trilha; o CAPS não cria paciente.
- **D4 — Sem `q=` na lista de usuários.** `GET /psychosocial/enrollments?…&q=` poria nome na URL (Global Constraints). O dashboard não manda `q` e filtra a lista localmente; proposta: tirar `q` do contrato (ou trocar por POST se a lista ficar grande).
- **D5 — Pré-preenchimento da ESF.** "`esf_cnes` é pré-preenchido pelo território do 11" sem rota. Proposta: `GET /psychosocial/prefill?health_unit_id&patient_id` → `{ esf_cnes, esf_team_name }` (nulos sem território).
- **D6 — Formas de leitura.** O contrato não fixa as linhas. Propostas (campos clínicos **só** nos detalhes): `<referral-row>` `{ id, patient: {patient_id, name}, from_unit: {id, name}, priority, status: "pending"|"received", created_at }` e o detalhe + `reason`, `requested_by: {name}`; `<enrollment-row>` `{ id, health_unit_id, patient: <patient-ref>, started_on, status, reference: {user_id, name}, next_visit_at, care_plan_review_on, care_plan_overdue }` e o detalhe + `origin, cid_main: {code,label}, cid_secondary[], substances[], esf_covered, esf_cnes, destination, closed_on, raas_exported, active_night_stay: {id, bed_label}|null`; `<care-plan>` `{ id, version, needs, goals: [ {text, responsible: {user_id,name}, due_on} ], intensity, review_on, created_at, author: {name} }`; `<action-row>` `{ id, sigtap: <sigtap>, performed_at, place, quantity, professional: <pro>, participants_count, group: {id,name}|null, status: "recorded"|"cancelled", month_closed }` e o detalhe + `notes, authorization_number, participants: [ {kind: "enrollment"|"patient"|"family", enrollment_id?, name} ], cancel_reason`; `<group>` com `weekday` ISO (1 = segunda … 7 = domingo) e horas `"HH:MM"`; `GET /psychosocial/today` → `{ date, groups: [ <group> + action_id|null ], visits: [ {enrollment_id, patient: <patient-ref>, next_visit_at, reference: {name}} ] }`.
- **D7 — Acolhimento: resposta e bloco da matrícula.** Proposta: `POST /psychosocial/receptions` → `{ reception_id, enrollment_id|null }`; o bloco `enrollment` leva os campos da matrícula **sem** `health_unit_id` e `patient_id` (vêm do acolhimento); encaminhamento já acolhido → 409 `referral_not_pending`.
- **D8 — Documentos da ação.** A tela precisa listar e reimprimir as receitas da ação. Proposta: `GET /psychosocial/actions/:id/documents` → `{ items: [<clinical-document>] }` (com trilha); `POST /psychosocial/actions/:id/documents { kind: "prescription", content }` → `<clinical-document>`; **só `prescription`** no CAPS (atestado, declaração e requisição dependem de atendimento ou consulta do 19); imprimir e cancelar pelas rotas existentes `/attendance/documents/:id/print` e `/cancel` com `health_unit_id` do CAPS, aceitas para o autor lotado.
- **D9 — Unidade CAPS pela rota real.** O contrato §3 diz `POST/PATCH /health_units`; a rota real é `POST /attendance/units` e `POST /attendance/units/:id` (sem step-up hoje). Proposta: os campos do CAPS entram nessas rotas, sem step-up novo; step-up só em leitos (se o api quiser) e em `/psychosocial/settings`.
- **D10 — Leitura das regras.** Só há `PATCH /psychosocial/settings`. Proposta: `GET /psychosocial/settings` → a mesma forma, com os padrões quando a cidade não configurou.
- **D11 — Remessas para o faturista.** Para baixar sem ler a RAAS nominal, `GET /psychosocial/raas_exports?health_unit_id&month` → `{ items: [ {id, month, competence, created_at, layout_version} ] }` para `psychosocial_lead` e `billing_clerk`; `POST /psychosocial/raas_exports` devolve o mesmo item.
- **D12 — Matriciamento: leitura.** Proposta: `GET /psychosocial/matrix_supports?health_unit_id&month` → `{ items: [ {id, health_team: {id, name, unit_name}, held_on, topic, cases_count} ] }` (sem texto) e `GET /psychosocial/matrix_supports/:id` → `+ participants: [ {user_id, name} ], cases: [ {id, patient: {patient_id, name}, shared_text} ]`; `POST …/cases` devolve o matriciamento inteiro.
- **D13 — Caminho do vínculo.** `GET /patients/:id/psychosocial_link` fica fora de todo escopo do api e do proxy. Proposta: `GET /psychosocial/patients/:id/link`, mesma forma e regras do §7.
- **D14 — Painel por mês e forma.** O §11 tem só `health_unit_id`. Proposta: `GET /psychosocial/dashboard?month&health_unit_id?` → `{ month, active_by_unit: [ {unit: {id, name, caps_modality}, count} ], new_in_month, closed_by_destination: [ {destination, count} ], care_plans_overdue, actions_by_instrument: {raas_psi, bpa_c, bpa_i}, bed_occupancy_pct: number|null, raas: {open, closed, exported} }`, com `"lt5"` para 1–4 (como o 30).
- **D15 — Limites de texto.** Motivos 10–500; motivo do encaminhamento 10–2000; `notes` ≤ 4000; `needs` 10–4000; objetivo 3–500; tema 3–200; `shared_text` 10–2000; nome do grupo 3–80; rótulo do leito 1–20; familiar 3–120; autorização 1–20 dígitos; até 4 CIDs secundários.
- **D16 — Forma de `problems[]`.** O §2 diz `{error, path}` e o §4 diz que cada problema traz `code` e `participant_index`. Proposta: `{ error, path, participant_index? }` (a convenção dos módulos anteriores); o dashboard aceita `code` ou `error`.
- **D17 — Códigos sem nome no contrato.** `esf_cnes` obrigatório (422 `esf_cnes_required`), `closed_on` futuro (422 `closed_on_in_future`), `origin`/`started_on` depois da exportação (422 `origin_locked`), `night_limit_reached` com `{ used, limit, window_days }`.
- **D18 — Marcador na linha do tempo geral.** O §10 não dá o `kind`. Proposta: `{ kind: "psychosocial_link", occurred_at: <início>, unit: {id, name} (do CAPS), professional: null, title: "Acompanhamento no CAPS <nome>", summary: null, status: "open"|"closed", detail_ref: null }`; os kinds de `CAPS_ONLY_KINDS` (servidos só em `/psychosocial/enrollments/:id/timeline` → `{ entries, next_before }`) com o prefixo `psychosocial_` e `detail_ref: null`.
- **D19 — Rótulos do destino para o analista.** O analista não lê `/psychosocial/options` (é da equipe). Proposta: `closed_by_destination[].label` no painel; sem ele, a tela usa o código cru.
- **D20 — Mês: forma e situação de rua.** Proposta: `GET /psychosocial/months/:month` → `{ month, health_unit_id, status, closure: {closed_at, closed_by: {name}}|null, counts: {records, with_problems, actions}, records: [ {id, enrollment_id, patient: <patient-ref>, homeless: bool|null, actions_count, status, problems: [ {code, action_id?} ]} ] }`; `PATCH /psychosocial/monthly_records/:id { homeless }` por `psychosocial_staff` só com o mês aberto (`month_closed` depois).

## Self-review

1. **Cobertura da spec:** decisão 1 → Tasks 6, 7, 10; 2 → 11 (remessa e ficha); 3 → 6, 12, 13 (nada clínico fora do CAPS); 4 → Global Constraints, 9 (sem aviso), 8 (receita fora dos documentos gerais, api); 5 → 4; 6 → 3 (papéis, portão por lotação), 7 e 11 (coordenação); 7 → 5 e 6 (matrícula única, ESF, destino); 8 → 5 (desfecho sem matrícula); 9 → 6 (versões imutáveis, revisão vencida); 10 → 7 (familiar, território, códigos das opções); 11 → 2 e 7 (crítica local e do api, cancelamento); 12 → 11 (situação de rua por mês); 13 → 11 (fechar, reabrir, competência do 22 mostrada, quem baixa); 14 → 11; 15 → 12 (BPA-C é do api; a tela avisa); 16 → 12 e 13; 17 → 4 e 10; 18 → 9; 19 → 8; 20 → 13 e 5; 21 → 14; 22 → api (nenhuma tela manda ao LEDI); 23 → 14; 24 → 1, 2 e 3. §4 telas → Tasks 4–14; §5 → Global Constraints e Task 15 Step 2.
2. **Passos:** cada passo de código diz arquivo, assinatura e os valores da spec/contrato; corpos só para as fixtures, os testes e as frases fixas (`PSY_ERRORS`, `MONTH_PROBLEM_TEXT`). Os rótulos de componentes existentes que os testes usam (`UnitForm`, `PersonStart`, `SensitiveAction`, `PrescriptionForm`, `SegmentedControl`) são conferidos na Task 0 contra o `f6fb6e4`.
3. **Tipos:** `PsyUnit`, `PsyOptions`, `PsyEnrollment`, `PsyAction`, `ParticipantDraft`, `ActionDraft`, `PsyRoles` e as funções da Task 1 são os mesmos nas Tasks 2–14; `EnrollmentFields` nasce na Task 5 e serve à 6; `ActionForm` (Task 7) não é reusado pela presença (a `PresenceList` chama `createPsyAction` direto); `ActionDetail.children` (Task 7) recebe a `ActionDocuments` (Task 8); `EnrollmentView` (Task 6) recebe as abas das Tasks 7 e 14 e é montado pela 9.
4. **Review Focus:** as cinco linhas têm teste na task dona (9; 7; 6 e 5; 11; 10).
5. **Proporção:** o código do plano é teste, fixture e frase fixa; implementação por assinatura e valores.
