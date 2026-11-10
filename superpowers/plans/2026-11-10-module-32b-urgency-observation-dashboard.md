# Módulo 32b — Observação, prescrição interna e checagem na UPA (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 32b** (`feat/mod-32b-urgency-observation`, worktree `apps/api/.claude/mod32b`, porta de dev **3055**) com as rotas do contrato 2026-11-10 §3–§9 (e as Divergências D1–D19 deste plano que o plano do api adotar), o interruptor novo `urgency_stock_deduction` e a semente do 32b (leitos, itens do catálogo com saldo no local `unit` da UPA). Ordem de entrega do contrato §11: api → dashboard → maintenance. As Tasks 0–10 não dependem do api (dublês de `src/lib/api` com fixtures nas formas do contrato §2–§9 e das Divergências); a prova da Task 12 depende.

**Goal:** Lado dashboard do 32b: o **cadastro de leitos de observação** da UPA (tela irmã no grupo "Urgência", com step-up); o **mapa de observação** como painel novo na mesa da UPA (`UrgencyDesk`), com um cartão por leito, horas desde a primeira entrada, alertas de 20 h e 24 h por texto, entrada, troca de leito e saída sem desfecho, polling de 30 s com `clearTimeout`; a coluna de observação em "Em atendimento" da fila; a **prescrição interna** no painel de atendimento (`UrgencyCare`), entre a consulta e o desfecho: itens só do catálogo (controlados aceitos, com a marca), dose, via, frequência, primeira dose, versão nova a cada alteração e suspensão com motivo; a **checagem** (administrado, não administrado, recusado) no painel de atendimento e no **painel de medicação** da enfermagem e do técnico, com o aprazamento lido sob demanda, lote e quantidade só com `urgency_stock_deduction` utilizável e a falta de saldo avisada sem recusar; as **faltas da UPA** numa tela nova do grupo "Farmácia", com conciliação, e a contagem nos alertas da farmácia; o bloco `observation` no Painel da urgência; `observation_hours` na linha do tempo profissional; a chave `urgency_stock_deduction` em `features.ts`.

**Architecture:** O cliente HTTP ganha a seção "Urgência — observação, prescrição e checagem (módulo 32b)" logo depois da seção do 32a em `src/lib/api.ts`, reaproveitando `urgencyPath`, `urgencyWrite`, `urgencyQuery` e `phPath`/`phWrite` (tipos = formas do contrato e das Divergências). As regras de tela ficam puras num arquivo novo, `src/lib/urgencyObservation.ts` (frequência, alertas, janela de checagem, validação e corpo da prescrição e da checagem); as frases de **todos** os códigos novos entram no `URGENCY_ERRORS` de `src/lib/urgency.ts`, que continua o único tradutor (`urgencyError`/`urgencyRefusals`), e `urgencyRoles` ganha o técnico 3222 e os papéis derivados. As peças novas ficam em `src/modules/urgency/` (mapa, prescrição, checagem, leitos), a tela de faltas em `src/modules/PharmacyUrgencyShortfalls.tsx`, sobre `PharmacyGate` + `LocationPick` do 27a. **Copiar o molde, não generalizar:** o mapa copia o `BedCard` de `src/modules/CapsBeds.tsx` e o cadastro copia `src/modules/psychosocial/BedsAdmin.tsx`; nenhum dos dois muda. O polling do mapa usa o `usePolling` do 32a; o aprazamento grava trilha, então é lido só sob demanda, com `NOMINAL_QUERY`. O dashboard nunca decide o que o api garante: leito livre, horas e alertas, versão ativa, horários do aprazamento, FEFO, saldo, falta, BPA e imutabilidade são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (decisões 2, 3, 20–23, a R7 e o §3 "Cidade (32b)"; §4 telas do 32b; §8 F-32.7 e F-32.8), `docs/adr/0050.md`, o contrato do 32b `docs/superpowers/plans/2026-11-10-module-32b-urgency-observation-contracts.md` (§1–§12, com P1 e P2) e o do 32a `docs/superpowers/plans/2026-11-05-module-32a-urgency-contracts.md` (inteiro, até Q31). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D19); nada foi resolvido em silêncio.

## Global Constraints

- **Nomes no código** (contrato §1): interruptor novo `urgency_stock_deduction` (requer `urgency_care` e `pharmacy`); leitos `urgency_beds` (`label` 1–20); estadia `end_kind` = `bed_change` | `observation_ended` | `outcome`; alerta `null` | `"near_limit"` (≥ 20 h) | `"over_limit"` (≥ 24 h); prescrição `status` = `active` | `superseded` | `suspended` | `ended`; frequência `once` | `interval` (1–24 h) | `if_needed` (com `max_per_day`); via = `PrescriptionItem::ROUTES` (no dashboard, `ROUTES`/`ROUTE_LABEL` de `src/lib/clinicalDocuments.ts`); checagem `administered` | `not_administered` | `refused`; falta `cause` = `no_location` | `insufficient_balance` | `lot_unavailable`. **Código desconhecido aparece cru, nunca some.**
- **Interruptores:** observação, prescrição e checagem vivem sob `urgency_care` (`canUseUrgency`, que exige também `clinical_record`); sem ele, nada do 32b aparece na mesa. A sessão traz a chave **ligada**, não utilizável (`src/lib/features.ts:1-4`): **lote e quantidade** só aparecem quando o aprazamento diz `stock_deduction: true` (D5), nunca pela sessão. Com `stock_deduction: false` o corpo nunca leva `stock_lot_id` nem `stock_quantity` (assim o 422 `stock_deduction_disabled` não acontece pela tela). A tela de faltas exige só `pharmacy` (contrato §7). O dashboard nunca escreve interruptor; a chave nova ganha só rótulo.
- **Quem faz o quê** (contrato §1): cadastro de leitos = `municipal_admin` com step-up; mapa (leitura) = recepção (`citizen_verifier`), enfermeiro 2235\*, técnico 3222\* e médico 225\* vinculados à UPA; entrada, troca e saída = médico ou enfermeiro; prescrição, alteração e suspensão = médico; checagem = enfermeiro, técnico ou médico; faltas = farmacêutico e almoxarife **do local `unit` da UPA** (`locationRoles` do `LocationPick`). O papel na UPA sai de `getMyProfessional().links` (vínculo ativo na unidade, como `urgencyRoles`); o api confere de novo. O técnico continua fora da classificação (Q2).
- **Nada clínico em lista** (contrato §4): o mapa, a coluna de observação da fila e a lista de faltas mostram só `<patient-ref>` (nome, CNS mascarado, "sem identificação"), senha, leito, horários, horas e o alerta — **nunca** medicamento, dose, motivo, CID ou queixa; as faltas **nem o nome do paciente** (contrato §7). Linhas montadas campo a campo (`safe*`), como na fila do 32a: o que o api mandar a mais não passa.
- **Trilha:** o mapa e as faltas não gravam trilha e podem fazer polling/releitura. O detalhe do encontro (com `observation`, `prescription` e `administrations`) e o aprazamento (`medication_schedule`) gravam: lidos **só no clique** ou depois de uma escrita, com `NOMINAL_QUERY` (`gcTime: 0`, sem `refetchInterval`, sem releitura por foco), e saem do cache ao fechar o painel.
- **Dado sensível só no corpo:** nome, CPF, CNS, CID, queixa, **medicamento** e **motivo** nunca na URL, em mensagem de erro, em `console`, em Analytics ou em storage (contrato §11). Na URL vão só ids, `health_unit_id`, `stock_location_id` e `state`. A busca de medicamento continua por `POST` com o termo no corpo (`searchMedications`). Nenhum storage novo neste módulo.
- **Polling:** só o mapa de observação faz polling novo, pelo `usePolling` (`URGENCY_OBSERVATION_POLL_MS = 30_000`): um pedido por vez, `clearTimeout` ao desmontar ou trocar de unidade, resposta velha descartada, parada nas recusas 403/404/422 com "Tentar de novo" (o mesmo critério do `UrgencyQueue`). Nenhum `refetchInterval` novo. O aprazamento **não** faz polling (grava trilha): mostra "lido às hh:mm" e um botão "Atualizar".
- **Imutável e sem correção no 32b** (contrato §6): a checagem passa por `ConfirmButton` ("A checagem não pode ser alterada. Confirmar?"); a tela só oferece "Checar" para horários até **60 min à frente** (D8); o duplo clique envia uma vez (`busy` + `useRef`).
- **Todas as recusas de uma vez:** a tela confere o formulário inteiro antes de enviar (`Problem[]` com o `path` do api, `items.<n>.<campo>`); a recusa do api com `problems[]` mostra todas, por `urgencyRefusals`; sem `problems`, a do topo. Todo código novo do contrato §3–§7 tem frase em `URGENCY_ERRORS` (o teste da Task 2 cobra a lista). Código novo do api aparece como "recusado pelo servidor (<código>)".
- **Recusar em vez de cortar** (D7): rótulo do leito 1–20; 1 a 20 itens; dose decimal > 0 (vírgula ou ponto); unidade da dose 1–20; intervalo 1–24 h inteiro; máximo por dia 1–24 inteiro; orientação até 500; quantidade de estoque por dose decimal > 0; motivo da suspensão, da não administração e da recusa 10–500; nota da conciliação 10–500.
- **Sem duplo envio:** botão de escrita desabilitado enquanto a chamada corre, e a resposta de uma ação já trocada (outro leito, outro encontro, outra unidade) é ignorada (`seq`), como no 32a.
- Toda escrita com corpo JSON (`urgencyWrite`/`phWrite`; `{}` quando não há campo).
- "Agora" é `new Date()`; telas recebem `now` por prop e os testes passam `NOW32B = new Date("2026-11-10T14:00:00-03:00")`; relógio falso (`vi.useFakeTimers`) **só** no teste de polling do mapa (Task 5), com `vi.useRealTimers()` no `afterEach`.
- Interface (ciclo de interface): toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição); `DataTable` com `card: "title"`/`"actions"` para o telefone; cartões do mapa em `flex-wrap` com `flex: "1 1 240px"` e `minWidth: 0` (como `CapsBeds`); cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`); sem cor literal, sem `outline: "none"`, sem `textOverflow`/ellipsis (o api corta nada: texto longo quebra linha), alvos de toque dos botões pelos estilos de `formStyles` (guardas `src/theme/*.test.ts`: `css-guards`, `a11y-guards`, `usability-guards`, `layout-classes`, `literals`). Alerta **sempre com texto** ("mais de 24 h em observação"), nunca só tom. A mesa funciona em 1280 px e em 375 px (telefone) e só com teclado. Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `formStyles`, `MedicationSearch`, `ConsultationWorkspace`, `PharmacyGate`, `LocationPick`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa (vai ao usuário; ciclo de interface).
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-32.7** e **F-32.8** do board #1 para **In Progress** (Task 0); F-32.7 (api, dashboard) vai a **Done** quando o dashboard termina; F-32.8 (api, dashboard, maintenance) espera o maintenance; **Verified** só depois da prova no navegador (Task 12).
- **Relato por task:** ao encerrar cada task, uma linha à sessão "Próximos módulos" (o coordenador), com o que diz o item "Relato ao coordenador" da task.

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod32b` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-32b-urgency-observation` a partir de `origin/main` (`979e881` ou mais novo), com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod32b`, `G=/opt/homebrew/bin/git`.
- Testes: `$R/bin/test dashboard -w dashboard-mod32b <arquivos>` (o runner único; uma execução por vez na máquina; a saída é o resumo, o log fica em `.test-runs/`). Nunca `vitest` direto. Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem jest-dom (`not.toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx`; as fixtures do 32a vêm de `src/test/urgencyFixtures.ts`; as do 32b, de `src/test/urgencyObservationFixtures.ts` (Task 1).

## Review Focus

1. **Duas pessoas tentam o mesmo leito, ou o leito é desativado enquanto o mapa está aberto.** Espera-se: 409 `bed_taken` e 422 `bed_unavailable` mostram a frase, relêem o mapa na hora (sem esperar os 30 s) e não deixam o formulário de entrada aberto sobre um leito que já mudou; a troca de leito oferece só leitos livres e ativos da mesma UPA; nenhum pedido sai depois de desmontar. Teste: Task 5, "bed_taken relê o mapa; troca só para leito livre; nada depois de desmontar".
2. **A enfermeira checa a dose errada, ou a mesma dose duas vezes.** Espera-se: "Checar" só nos horários até 60 min à frente; o `ConfirmButton` pede o segundo passo com o nome do item e o horário; o duplo clique envia **uma** vez; 409 `slot_taken` relê o aprazamento e mostra quem checou; `not_administered`/`refused` sem motivo (ou com 9 caracteres) não sai. Teste: Task 7, "checagem: janela de 60 min, confirmação, uma chamada, slot_taken relê, motivo".
3. **Estoque ligado sem saldo, desligado, ou o api muda de ideia no meio.** Espera-se: com `stock_deduction: true`, lote (FEFO pré-escolhido) e quantidade aparecem e só vão no corpo quando mudados; a resposta com `shortfall` mostra "registrado — o estoque ficou com pendência para a farmácia (sem saldo suficiente)" e **não** é erro; com `stock_deduction: false`, nenhum campo de estoque e nenhum `stock_*` no corpo; 422 `lot_invalid` relê o aprazamento e mantém o resto do formulário. Teste: Task 7, "estoque ligado, desligado, falta e lot_invalid".
4. **Alterar a prescrição sem perder o que já foi checado.** Espera-se: "Alterar prescrição" abre os itens da versão ativa, mantém o `first_dose_at` original dos itens que ficaram (D18), cria a versão nova com uma chamada só, e a tela diz "versão N ativa; a versão N−1 ficou substituída"; controlado entra com a marca "controlado · lista X" (P1), sem o bloqueio do 19d; suspender pede motivo de 10 caracteres e o `ConfirmButton`. Teste: Task 6, "alterar mantém a primeira dose; controlado aceito; suspender".
5. **Lista que não pode vazar.** Espera-se: o mapa e a coluna "Observação" da fila não mostram medicamento, dose nem motivo mesmo que o api os mande; a lista de faltas não mostra nome do paciente (só senha e hora); a falta conciliada some da aba "Abertas" e `already_reconciled` relê. Teste: Tasks 5 e 9, "mapa e faltas campo a campo".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.urgencyObservation.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/urgencyObservationFixtures.ts` | tipos e cliente HTTP do 32b; interruptor; dados de teste | 1 |
| `src/lib/urgencyObservation.ts`, `src/lib/urgencyObservation.test.ts`, `src/lib/urgency.ts`, `src/lib/urgency.test.ts`, `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts` | regras de tela, papéis com o técnico, frases dos códigos novos, rótulo do movimento `administration` | 2 |
| `src/shell/modules.ts`, `src/shell/modules.urgency.test.ts`, `src/shell/modules.pharmacy.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/UrgencyBedsSettings.tsx`, `src/modules/PharmacyUrgencyShortfalls.tsx` | telas novas no menu e no catálogo (portões) | 3 |
| `src/modules/UrgencyBedsSettings.tsx`, `src/modules/urgency/BedsAdmin.tsx`, `src/modules/UrgencyBedsSettings.test.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/attendance/Units.caps.test.tsx` | cadastro de leitos com step-up; `upa_has_beds` no cadastro de unidade (F-32.7) | 4 |
| `src/modules/urgency/ObservationMap.tsx`, `src/modules/urgency/ObservationMap.test.tsx`, `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/UrgencyQueue.tsx`, `src/modules/urgency/UrgencyQueue.test.tsx` e os testes da mesa | mapa na mesa, entrada, troca e saída, técnico na mesa, coluna "Observação" na fila (F-32.7) | 5 |
| `src/modules/urgency/PrescriptionPanel.tsx`, `src/modules/urgency/PrescriptionItemFields.tsx`, `src/modules/urgency/PrescriptionPanel.test.tsx`, `src/modules/urgency/UrgencyCare.tsx` | prescrição interna: emitir, alterar (versão), suspender (F-32.8) | 6 |
| `src/modules/urgency/MedicationPanel.tsx`, `src/modules/urgency/AdministrationForm.tsx`, `src/modules/urgency/MedicationPanel.test.tsx`, `src/modules/urgency/UrgencyCare.tsx`, `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/ObservationMap.tsx`, `src/modules/urgency/UrgencyQueue.tsx` | aprazamento e checagem, com estoque e falta; painel de medicação da enfermagem (F-32.8) | 7 |
| `src/modules/urgency/ObservationSummary.tsx`, `src/modules/urgency/UrgencyCare.tsx`, `src/modules/urgency/OutcomeForm.tsx`, `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/UrgencyCare.observation.test.tsx` | observação no atendimento, evolução pelo adendo, desfecho que encerra a estadia (F-32.7) | 8 |
| `src/modules/PharmacyUrgencyShortfalls.tsx`, `src/modules/PharmacyUrgencyShortfalls.test.tsx`, `src/modules/PharmacyAlerts.tsx`, `src/modules/PharmacyAlerts.test.tsx` | faltas da UPA e conciliação; contagem nos alertas (F-32.8) | 9 |
| `src/modules/UrgencyPanel.tsx`, `src/modules/UrgencyPanel.test.tsx`, `src/modules/timeline/PatientTimeline.tsx`, `src/lib/timeline.urgency.test.ts` | bloco `observation` no painel do gestor; `observation_hours` na linha do tempo (F-32.7) | 10 |
| — | suíte, build, varredura, revisão | 11 |
| — | roteiro e prova no navegador (api 3055, Vite 5215; 1280 px e 375 px) | 12 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método e corpo — sobretudo o que **não** pode ir na URL (medicamento, motivo, nome) e que os campos de estoque só vão quando recebidos (Task 1); regras puras com tabela de casos: frequência, alertas, janela de 60 min, validação da prescrição e da checagem com os `path` do api, papéis com o técnico e a cobertura de **todos** os códigos novos (Task 2); menu e portão por papel e interruptor (Task 3); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e as fixtures do contrato, cobrindo o caminho feliz, o teclado, cada recusa que muda a tela, o duplo envio e o que **não** pode aparecer (texto clínico no mapa e na fila; paciente nas faltas; estoque com o interruptor desligado) (Tasks 4–10); o mapa com relógio falso para o polling de 30 s e o `clearTimeout` (Task 5); `tsc`, build, a suíte inteira, as guardas do tema e as varreduras (Task 11); a prova no navegador com o api real na 3055, em 1280 px e 375 px (Task 12). O dashboard não testa o que é do api (exclusion, FEFO, razão, BPA, trilha, imutabilidade, eventos).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`979e881` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod32b`.
- Produces: cards F-32.7 e F-32.8 em In Progress; worktree `$W` na branch `feat/mod-32b-urgency-observation`; a base anotada (arquivos e testes); as rotas e formas do api do 32b anotadas, por Divergência.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-32\\.[78] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os dois cards em In Progress ou além. Se algum não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Urgência (módulo 32a\|^const urgencyWrite\|^const urgencyQuery\|^export interface UrgencyInCareRow\|^export interface UrgencyEncounter {\|^export interface UrgencyDashboard {\|^export interface PharmacyAlerts\|^export interface TimelineEntry {\|^export async function searchMedications\|^export interface MedicationSearchItem\|^const phPath\|^const phWrite\|^export function fetchWaitingRoomBoard'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "urgency_care" | "waiting_room_display";\|  waiting_room_display: "Painel de chamada'
$G -C $R/apps/dashboard show origin/main:src/lib/urgency.ts | grep -n '^export type UrgencyRoles\|^export function urgencyRoles\|^const NURSE_CBO\|^export const URGENCY_ERRORS\|^const FIELD_TEXT\|^export const URGENCY_QUEUE_POLL_MS'
$G -C $R/apps/dashboard show origin/main:src/lib/pharmacy.ts | grep -n '^const MOVEMENT_LABEL\|^const DOCUMENT_LABEL\|^const ERRORS: Record\|^const PATH_RULES'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "urgency-panel" | "waiting-room-settings";\|{ id: "pharmacy-alerts"\|{ id: "waiting-room-settings"\|"waiting-room-settings") return\|item.id === "pharmacy-alerts") {'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '"waiting-room-settings": {\|"pharmacy-alerts": {\|IconBed\|IconPill'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'case "waiting-room-settings":\|case "pharmacy-alerts":'
$G -C $R/apps/dashboard show origin/main:src/modules/urgency/UrgencyDesk.tsx | grep -n 'if (!roles.reception && !roles.nurse && !roles.doctor)\|<UrgencyQueue ref={queueRef}\|const closeCare = '
$G -C $R/apps/dashboard show origin/main:src/modules/urgency/UrgencyQueue.tsx | grep -n 'const safeInCare = \|const hasRole = \|const inCareCols'
$G -C $R/apps/dashboard show origin/main:src/modules/urgency/UrgencyCare.tsx | grep -n '<ConsultationWorkspace attendanceId\|<OutcomeForm encounter={target}'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/Units.tsx | grep -n '^const CAPS_UNIT_ERRORS\|^const unitError'
$G -C $R/apps/dashboard show origin/main:src/modules/PharmacyAlerts.tsx | grep -n 'Ajustes aguardando aprovação'
$G -C $R/apps/dashboard show origin/main:src/modules/UrgencyPanel.tsx | grep -n 'label="Permanência · p90"'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n '{e.summary && <span style={muted}>{e.summary}</span>}'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationView.tsx | grep -n 'const isAuthor = '
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod32b -b feat/mod-32b-urgency-observation origin/main
cd $R/.worktrees/dashboard-mod32b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod32b
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(BedFlat|PillOff|Bed)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os ícones (`IconBedFlat` para os leitos de observação, `IconPillOff` para as faltas; falta algum → escolha outro que exista e use o mesmo na Task 3). **Anote** a base (arquivos e testes). A Task 11 espera a base mais **8 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira as rotas e formas do api do 32b**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api log --oneline -5 feat/mod-32b-urgency-observation 2>/dev/null || echo "api do 32b ainda sem branch"
$G -C $R/apps/api show feat/mod-32b-urgency-observation:config/routes.rb 2>/dev/null | grep -nE 'beds|observation|prescriptions|medication_schedule|administrations|urgency_shortfalls'
$G -C $R/apps/api show feat/mod-32b-urgency-observation:app/services/urgency/access.rb 2>/dev/null | grep -n '3222\|technician\|medication'
```

Expected: as rotas do contrato §3–§7. Para cada Divergência (D1–D19), anote se o api adotou a forma proposta ou outra; **onde o api diz outra coisa, vale o api** — a task dona ajusta tipo, fixture e teste antes de seguir (registre na descrição do commit). Sem branch do api ainda: siga pelas formas deste plano e repita este passo antes da Task 12.

**Relato ao coordenador:** "dashboard 32b: worktree dashboard-mod32b em <base>, F-32.7/F-32.8 em In Progress; api do 32b <com/sem> branch; Divergências adotadas pelo api: <lista>".

---

### Task 1: Tipos, cliente HTTP, interruptor e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova logo depois da do 32a; `UrgencyInCareRow`, `UrgencyEncounter`, `UrgencyDashboard`, `PharmacyAlerts`, `TimelineEntry`)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/test/urgencyObservationFixtures.ts`
- Test: `src/lib/api.urgencyObservation.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `urgencyPath`, `urgencyWrite`, `urgencyQuery`, `phPath`, `phWrite`, `MedicationRoute`, `UrgencyPatientRef`, `UrgencyCount`, `UrgencyTime` (existentes em `src/lib/api.ts`); as fixtures de `src/test/urgencyFixtures.ts`.
- Produces (todas em `src/lib/api.ts`, exportadas):
  - Tipos: `ObservationAlert = "near_limit" | "over_limit" | null`; `BedStayEndKind = "bed_change" | "observation_ended" | "outcome"`; `FrequencyKind = "once" | "interval" | "if_needed"`; `UrgencyPrescriptionStatus = "active" | "superseded" | "suspended" | "ended"`; `AdministrationStatus = "administered" | "not_administered" | "refused"`; `ShortfallCause = "no_location" | "insufficient_balance" | "lot_unavailable"`; `ShortfallState = "open" | "reconciled"`.
  - Leitos (§3): `UrgencyBed { id; health_unit_id; label; active: boolean }`; `UrgencyBedInput { health_unit_id; label; active }`; `UrgencyBedPatch { label?: string; active?: boolean }`.
  - Mapa (§4 + D3): `ObservationStayRef { stay_id; encounter_id; ticket: string | null; patient: UrgencyPatientRef; started_at; hours: number; alert: ObservationAlert }`; `ObservationBed { bed_id; label; active: boolean; stay: ObservationStayRef | null }`; `ObservationWaiting { encounter_id; attendance_id; ticket: string | null; patient: UrgencyPatientRef; called_at: string | null }`; `UrgencyObservationMap { beds: ObservationBed[]; waiting_bed: ObservationWaiting[] }`; `UrgencyInCareObservation { bed_label: string; hours: number; alert: ObservationAlert }` e `UrgencyInCareRow` ganha `observation?: UrgencyInCareObservation | null`.
  - Detalhe (§2 + D4): `UrgencyStay { stay_id; bed: { id; label }; started_at; ended_at: string | null; end_kind: BedStayEndKind | null; started_by: { name }; ended_by: { name } | null }`; `UrgencyObservationBlock { open: UrgencyStay | null; stays: UrgencyStay[]; hours: number | null; alert: ObservationAlert }`; `UrgencyFrequency { kind: FrequencyKind; interval_hours?: number; max_per_day?: number }`; `UrgencyPrescriptionItem { id; position: number; catalog_item_id; label; catmat_code: number | null; controlled_list: string | null; dose: string; dose_unit; route: MedicationRoute; frequency: UrgencyFrequency; first_dose_at; instructions: string | null; stock_quantity_per_dose: string; dispensing_unit: string | null }`; `UrgencyPrescription { id; encounter_id; version: number; status: UrgencyPrescriptionStatus; supersedes_id: string | null; prescribed_at; prescribed_by: { name }; suspension_reason?: string | null; status_changed_at: string | null; items: UrgencyPrescriptionItem[] }`; `UrgencyAdministration { id; prescription_item_id; scheduled_at: string | null; status: AdministrationStatus; reason?: string | null; administered_at: string | null; recorded_at; recorded_by: { name }; stock: { lot_number; expires_on; quantity: string; dispensing_unit: string } | null; shortfall: { cause: ShortfallCause } | null }`; `UrgencyEncounter` ganha `observation?: UrgencyObservationBlock | null`, `prescription?: UrgencyPrescription | null`, `administrations?: UrgencyAdministration[]` (opcionais: sem acesso clínico, Q19, não vêm).
  - Prescrição (§5): `UrgencyPrescriptionItemInput { catalog_item_id; dose: number; dose_unit; route: MedicationRoute; frequency: UrgencyFrequency; first_dose_at?: string; instructions?: string; stock_quantity_per_dose?: number }`.
  - Aprazamento (§5 + D5, D18): `ScheduleSlot { scheduled_at; administration: UrgencyAdministration | null }`; `ScheduleLot { stock_lot_id; lot_number; expires_on; balance: string }`; `ScheduleItem { item: UrgencyPrescriptionItem; slots: ScheduleSlot[]; if_needed: { last_24h: UrgencyAdministration[]; remaining: number } | null; last_administration: UrgencyAdministration | null; lots?: ScheduleLot[] }`; `UrgencyMedicationSchedule { encounter_id; prescription: { id; version: number; status: UrgencyPrescriptionStatus } | null; stock_deduction: boolean; items: ScheduleItem[] }`.
  - Checagem (§6 + D9): `UrgencyAdministrationInput { prescription_item_id; scheduled_at?: string; status: AdministrationStatus; reason?: string; administered_at?: string; stock_lot_id?: string; stock_quantity?: number }`; `UrgencyAdministrationResult { administration: UrgencyAdministration; shortfall: { cause: ShortfallCause } | null }`.
  - Faltas (§7 + D16): `UrgencyShortfall { shortfall_id; created_at; catalog_item: { id; label; dispensing_unit: string | null }; quantity: string; cause: ShortfallCause; administration: { administered_at: string | null; ticket: string | null }; reconciled_at?: string | null; reconciled_by?: { name } | null }`; `PharmacyAlerts` ganha `urgency_shortfalls?: number`.
  - Painel (§9 + D12): `UrgencyDashboard` ganha `observation?: { stays: UrgencyCount; over_24h: UrgencyCount; median_hours: UrgencyTime; p90_hours: UrgencyTime; bed_hours: number }`. Linha do tempo (§9 + D11): `TimelineEntry` ganha `observation_hours?: number | null`.
  - Funções: `listUrgencyBeds(unitId): Promise<UrgencyBed[]>`, `createUrgencyBed(i: UrgencyBedInput): Promise<UrgencyBed>`, `updateUrgencyBed(id, p: UrgencyBedPatch): Promise<UrgencyBed>`, `getUrgencyObservation(unitId): Promise<UrgencyObservationMap>`, `startObservation(encounterId, bedId): Promise<UrgencyStay>`, `endObservation(encounterId): Promise<UrgencyStay>`, `issueUrgencyPrescription(encounterId, items: UrgencyPrescriptionItemInput[]): Promise<UrgencyPrescription>`, `suspendUrgencyPrescription(prescriptionId, reason): Promise<UrgencyPrescription>`, `getMedicationSchedule(encounterId): Promise<UrgencyMedicationSchedule>`, `recordAdministration(i: UrgencyAdministrationInput): Promise<UrgencyAdministrationResult>`, `listUrgencyShortfalls(locationId, state: ShortfallState): Promise<UrgencyShortfall[]>`, `reconcileUrgencyShortfall(id, note): Promise<UrgencyShortfall>`.
  - `src/test/urgencyObservationFixtures.ts`: `NOW32B`, `TECH_LINK`, `bed(o?)`, `stayRef(o?)`, `observationBed(o?)`, `observationMap(o?)`, `stay(o?)`, `observationBlock(o?)`, `prescriptionItem(o?)`, `prescription(o?)`, `administration(o?)`, `scheduleItem(o?)`, `schedule(o?)`, `shortfall(o?)`, `observedEncounter(o?)`.

- [ ] **Step 1: Escreva as fixtures** `src/test/urgencyObservationFixtures.ts` (o único arquivo com corpo completo nesta task)

```ts
// Dados dos testes do módulo 32b (observação, prescrição interna, checagem). Agora: segunda, 2026-11-10, 14h.
// Formas do contrato 2026-11-10 §2–§9 e das Divergências do plano do dashboard.
import type {
  ObservationBed, ObservationStayRef, ProfessionalLink, ScheduleItem, UrgencyAdministration, UrgencyBed, UrgencyEncounter,
  UrgencyMedicationSchedule, UrgencyObservationBlock, UrgencyObservationMap, UrgencyPrescription, UrgencyPrescriptionItem,
  UrgencyShortfall, UrgencyStay
} from "../lib/api";
import { DOCTOR_LINK, encounter, patientRef } from "./urgencyFixtures";

export const NOW32B = new Date("2026-11-10T14:00:00-03:00");
export const TECH_LINK = { ...DOCTOR_LINK, id: "l3", cbo_code: "322205", cbo_title: "Técnico de enfermagem" } as ProfessionalLink;

export const bed = (o: Partial<UrgencyBed> = {}): UrgencyBed => ({ id: "b1", health_unit_id: "upa1", label: "Obs 1", active: true, ...o });
export const stayRef = (o: Partial<ObservationStayRef> = {}): ObservationStayRef => ({
  stay_id: "st1", encounter_id: "en3", ticket: "U-038", patient: patientRef({ patient_id: "pa3", name: "João Reis" }),
  started_at: "2026-11-10T08:00:00-03:00", hours: 6, alert: null, ...o
});
export const observationBed = (o: Partial<ObservationBed> = {}): ObservationBed =>
  ({ bed_id: "b1", label: "Obs 1", active: true, stay: stayRef(), ...o });
export const observationMap = (o: Partial<UrgencyObservationMap> = {}): UrgencyObservationMap => ({
  beds: [ observationBed(), observationBed({ bed_id: "b2", label: "Obs 2", stay: null }) ],
  waiting_bed: [ { encounter_id: "en4", attendance_id: "at4", ticket: "U-041", patient: patientRef({ patient_id: "pa4", name: "Rita Melo" }),
    called_at: "2026-11-10T13:30:00-03:00" } ],
  ...o
});
export const stay = (o: Partial<UrgencyStay> = {}): UrgencyStay => ({
  stay_id: "st1", bed: { id: "b1", label: "Obs 1" }, started_at: "2026-11-10T08:00:00-03:00", ended_at: null, end_kind: null,
  started_by: { name: "Dr. Caio" }, ended_by: null, ...o
});
export const observationBlock = (o: Partial<UrgencyObservationBlock> = {}): UrgencyObservationBlock =>
  ({ open: stay(), stays: [ stay() ], hours: 6, alert: null, ...o });
export const prescriptionItem = (o: Partial<UrgencyPrescriptionItem> = {}): UrgencyPrescriptionItem => ({
  id: "pi1", position: 1, catalog_item_id: "cat-dipirona", label: "Dipirona 500 mg/mL solução injetável", catmat_code: 267_501,
  controlled_list: null, dose: "1", dose_unit: "g", route: "intravenous", frequency: { kind: "interval", interval_hours: 6 },
  first_dose_at: "2026-11-10T08:00:00-03:00", instructions: null, stock_quantity_per_dose: "1", dispensing_unit: "ampola", ...o
});
export const prescription = (o: Partial<UrgencyPrescription> = {}): UrgencyPrescription => ({
  id: "pr1", encounter_id: "en3", version: 1, status: "active", supersedes_id: null, prescribed_at: "2026-11-10T08:00:00-03:00",
  prescribed_by: { name: "Dr. Caio" }, suspension_reason: null, status_changed_at: null, items: [ prescriptionItem() ], ...o
});
export const administration = (o: Partial<UrgencyAdministration> = {}): UrgencyAdministration => ({
  id: "ad1", prescription_item_id: "pi1", scheduled_at: "2026-11-10T08:00:00-03:00", status: "administered", reason: null,
  administered_at: "2026-11-10T08:05:00-03:00", recorded_at: "2026-11-10T08:06:00-03:00", recorded_by: { name: "Ana Souza" },
  stock: null, shortfall: null, ...o
});
// Item de 6 em 6 h desde as 8h: 8h (checado), 14h (agora), 20h, 2h.
export const scheduleItem = (o: Partial<ScheduleItem> = {}): ScheduleItem => ({
  item: prescriptionItem(),
  slots: [
    { scheduled_at: "2026-11-10T08:00:00-03:00", administration: administration() },
    { scheduled_at: "2026-11-10T14:00:00-03:00", administration: null },
    { scheduled_at: "2026-11-10T20:00:00-03:00", administration: null },
    { scheduled_at: "2026-11-11T02:00:00-03:00", administration: null }
  ],
  if_needed: null, last_administration: administration(), ...o
});
export const schedule = (o: Partial<UrgencyMedicationSchedule> = {}): UrgencyMedicationSchedule =>
  ({ encounter_id: "en3", prescription: { id: "pr1", version: 1, status: "active" }, stock_deduction: false, items: [ scheduleItem() ], ...o });
export const shortfall = (o: Partial<UrgencyShortfall> = {}): UrgencyShortfall => ({
  shortfall_id: "sf1", created_at: "2026-11-10T08:06:00-03:00",
  catalog_item: { id: "cat-dipirona", label: "Dipirona 500 mg/mL solução injetável", dispensing_unit: "ampola" },
  quantity: "1", cause: "insufficient_balance", administration: { administered_at: "2026-11-10T08:05:00-03:00", ticket: "U-038" },
  reconciled_at: null, reconciled_by: null, ...o
});
// O detalhe que o médico lê, com os três blocos do 32b.
export const observedEncounter = (o: Partial<UrgencyEncounter> = {}): UrgencyEncounter => encounter({
  id: "en3", attendance_id: "at3", state: "in_care", ticket: "U-038", color: "orange", consultation_id: "co1",
  consultation_status: "finalized", observation: observationBlock(), prescription: prescription(), administrations: [ administration() ], ...o
});
```

(Os campos de `UrgencyEncounter` e `ProfessionalLink` que faltarem são conferidos na Task 0 contra o `src/lib/api.ts` real; o `as` fica só nas fixtures.)

- [ ] **Step 2: Escreva os testes do cliente** `src/lib/api.urgencyObservation.test.ts` (mesmo `fetch` falso de `src/lib/api.urgency.test.ts`)

```ts
it("leitos e mapa: só ids na URL; PATCH parcial", async () => {
  await listUrgencyBeds("upa1");
  expect(lastCall()).toEqual({ url: "/urgency/beds?health_unit_id=upa1", method: "GET", body: undefined });
  await createUrgencyBed({ health_unit_id: "upa1", label: "Obs 3", active: true });
  expect(lastCall()).toEqual({ url: "/urgency/beds", method: "POST", body: { health_unit_id: "upa1", label: "Obs 3", active: true } });
  await updateUrgencyBed("b1", { active: false });
  expect(lastCall()).toEqual({ url: "/urgency/beds/b1", method: "PATCH", body: { active: false } });
  await getUrgencyObservation("upa1");
  expect(lastCall().url).toBe("/urgency/observation?health_unit_id=upa1");
});
it("entrada, troca e saída da observação", async () => {
  await startObservation("en3", "b2");
  expect(lastCall()).toEqual({ url: "/urgency/encounters/en3/observation", method: "POST", body: { bed_id: "b2" } });
  await endObservation("en3");
  expect(lastCall()).toEqual({ url: "/urgency/encounters/en3/observation/end", method: "POST", body: {} });
});
it("prescrição e suspensão: medicamento e motivo só no corpo", async () => {
  await issueUrgencyPrescription("en3", [ { catalog_item_id: "cat-dipirona", dose: 1, dose_unit: "g", route: "intravenous",
    frequency: { kind: "interval", interval_hours: 6 } } ]);
  expect(lastCall().url).toBe("/urgency/encounters/en3/prescriptions");
  expect(lastCall().body).toEqual({ items: [ { catalog_item_id: "cat-dipirona", dose: 1, dose_unit: "g", route: "intravenous",
    frequency: { kind: "interval", interval_hours: 6 } } ] });
  await suspendUrgencyPrescription("pr1", "reação alérgica após a dose");
  expect(lastCall()).toEqual({ url: "/urgency/prescriptions/pr1/suspend", method: "POST", body: { reason: "reação alérgica após a dose" } });
  for (const c of calls()) expect(c.url).not.toMatch(/dipirona|alérg|al%C3%A9rg/i);
});
it("aprazamento e checagem: os campos de estoque só quando recebidos", async () => {
  await getMedicationSchedule("en3");
  expect(lastCall()).toEqual({ url: "/urgency/encounters/en3/medication_schedule", method: "GET", body: undefined });
  await recordAdministration({ prescription_item_id: "pi1", scheduled_at: "2026-11-10T14:00:00-03:00", status: "administered" });
  expect(lastCall()).toEqual({ url: "/urgency/administrations", method: "POST",
    body: { prescription_item_id: "pi1", scheduled_at: "2026-11-10T14:00:00-03:00", status: "administered" } });
  await recordAdministration({ prescription_item_id: "pi1", status: "refused", reason: "paciente recusou a medicação" });
  expect("stock_lot_id" in (lastCall().body as object)).toBe(false);
});
it("faltas da UPA: local e estado na URL; nota no corpo", async () => {
  await listUrgencyShortfalls("loc-upa", "open");
  expect(lastCall().url).toBe("/pharmacy/urgency_shortfalls?stock_location_id=loc-upa&state=open");
  await reconcileUrgencyShortfall("sf1", "ajuste de contagem feito no estoque");
  expect(lastCall()).toEqual({ url: "/pharmacy/urgency_shortfalls/sf1/reconcile", method: "POST", body: { note: "ajuste de contagem feito no estoque" } });
});
```

E em `src/lib/features.test.ts`: `featureLabel("urgency_stock_deduction")` é `"Baixa de estoque na medicação da UPA"` (o mesmo rótulo do maintenance, contrato §11) e `hasFeature({ features: [ "urgency_stock_deduction" ] }, "urgency_stock_deduction")` é `true`.

- [ ] **Step 3: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/lib/api.urgencyObservation.test.ts src/lib/features.test.ts`
Expected: FAIL (funções e rótulo inexistentes).

- [ ] **Step 4: Implemente**

- `api.ts`, seção `// ─── Urgência — observação, prescrição e checagem (módulo 32b; ADR 0050; contrato 2026-11-10 §2–§9) ───` logo depois do fim da seção do 32a (depois de `fetchWaitingRoomBoard`): reusa `urgencyPath`, `urgencyWrite`, `urgencyQuery`; listas desembrulham `{ items }` (`items ?? []`); o mapa e o aprazamento chegam sem embrulho (contrato §4, D5); `getMedicationSchedule` com o comentário "Grava trilha no api". Os campos opcionais do corpo só entram quando não são `undefined` (o api recusa chave desconhecida, Q23; e recusa `stock_*` com o interruptor desligado).
- Faltas no fim da seção da farmácia (27a), com `phPath("urgency_shortfalls", …)` e `phWrite("POST", { note })`.
- Os campos novos de `UrgencyInCareRow`, `UrgencyEncounter`, `UrgencyDashboard`, `PharmacyAlerts` e `TimelineEntry` são **opcionais**: as fixtures e os testes do 32a continuam compilando sem mudar.
- `features.ts`: `| "urgency_stock_deduction"` no fim da união e, em `FEATURE_LABEL`, com o comentário "Módulo 32b (ADR 0050): exige urgency_care e pharmacy", `urgency_stock_deduction: "Baixa de estoque na medicação da UPA"`.
- `vite.config.ts` não muda: `/urgency` e `/pharmacy` já estão no proxy (linhas 96 e 107).

- [ ] **Step 5: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/lib/api.urgencyObservation.test.ts src/lib/api.urgency.test.ts src/lib/api.pharmacy.test.ts src/lib/features.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.urgencyObservation.test.ts src/lib/features.ts src/lib/features.test.ts src/test/urgencyObservationFixtures.ts
$G commit -m "feat: add urgency observation, internal prescription and medication check API client" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 1: cliente HTTP do 32b e `urgency_stock_deduction` em features.ts; formas assumidas = D3–D5, D9, D11, D12, D16 (ou as do api, se a Task 0 achou outras)".

---

### Task 2: Regras de tela, papéis com o técnico e frases dos códigos novos

**Files:**
- Create: `src/lib/urgencyObservation.ts`
- Modify: `src/lib/urgency.ts` (`UrgencyRoles`, `urgencyRoles`, `URGENCY_ERRORS`, `FIELD_TEXT`), `src/lib/pharmacy.ts` (`MOVEMENT_LABEL`, `DOCUMENT_LABEL`, `ERRORS`, `PATH_RULES`)
- Test: `src/lib/urgencyObservation.test.ts`, `src/lib/urgency.test.ts`, `src/lib/pharmacy.test.ts`

**Interfaces:**
- Consumes: os tipos da Task 1; `ROUTES` (`src/lib/clinicalDocuments.ts:82`); `Problem` (`src/lib/pharmacy.ts:234`); `chars` (`src/lib/urgency.ts`).
- Produces:
  - `urgency.ts`: `UrgencyRoles` ganha `technician: boolean` (vínculo ativo 3222\*), `observation: boolean` (= `nurse || doctor`: entrada, troca e saída), `medication: boolean` (= `nurse || technician || doctor`: checagem), `mapReader: boolean` (= `reception || nurse || technician || doctor`); `prescriber` não ganha campo (é `doctor`). `URGENCY_ERRORS` ganha `bed_unavailable`, `bed_taken`, `not_in_observation`, `consultation_required`, `catalog_item_invalid`, `prescription_not_active`, `slot_invalid`, `slot_taken`, `slot_not_due` (D8), `max_per_day_reached`, `stock_deduction_disabled`, `lot_invalid`, `label_taken`, `bed_occupied`, `upa_has_beds`; `FIELD_TEXT` ganha `bed_id`, `label`, `items`, `items.<n>.<campo>`, `scheduled_at`, `administered_at`, `stock_lot_id`, `stock_quantity`.
  - `urgencyObservation.ts`: `URGENCY_OBSERVATION_POLL_MS = 30_000`; `URGENCY_BEDS_KEY = "urgencyBeds"`; `MEDICATION_SCHEDULE_KEY = "urgencyMedicationSchedule"`; `URGENCY_SHORTFALLS_KEY = "urgencyShortfalls"`; `CHECK_AHEAD_MINUTES = 60`; `MAX_ITEMS = 20`; frases fixas (`BED_FREE`, `NO_UPA_BEDS`, `OBSERVATION_STARTED`, `BED_CHANGED`, `OBSERVATION_ENDED`, `NO_CLINICAL_IN_MAP`, `CHECK_IMMUTABLE`, `STOCK_PENDING`, `SHORTFALL_NO_PATIENT`); `alertText(a): string | null`, `alertTone(a): Tone`; `hoursText(h): string`; `FREQUENCY_KINDS`, `frequencyText(f): string`; `ADMIN_STATUSES`, `adminStatusLabel(code)`; `shortfallCauseLabel(code)`; `endKindLabel(code)`; `slotState(slot, now): "done" | "late" | "due" | "future"`; `canCheck(slot, now): boolean`; `ItemDraft`, `emptyItemDraft()`, `itemDraftFrom(item)`, `prescriptionProblems(drafts): Problem[]`, `prescriptionBody(drafts): UrgencyPrescriptionItemInput[]`; `AdministrationDraft`, `administrationProblems(d, ctx): Problem[]`, `administrationBody(d, ctx): UrgencyAdministrationInput`; `reasonProblem(text): string | null` (10–500); `bedLabelProblem(label): string | null` (1–20); `parseDecimal(raw): number | null`.
  - `pharmacy.ts`: `movementLabel("administration")` = "administração na UPA"; `movementDocument({ type: "urgency_administration", number: null })` = "administração na UPA"; `pharmacyError` traduz `already_reconciled` ("esta falta já foi conciliada por outra pessoa — a lista foi atualizada") e o `path` `note` ("a nota tem de 10 a 500 caracteres").

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/urgencyObservation.test.ts
it("alertas e horas sempre com texto", () => {
  expect(alertText(null)).toBeNull();
  expect(alertText("near_limit")).toBe("perto de 24 h em observação");
  expect(alertText("over_limit")).toBe("mais de 24 h em observação");
  expect(alertTone("over_limit")).toBe("down");
  expect(alertText("weird" as ObservationAlert)).toBe("weird");
  expect(hoursText(1)).toBe("1 h em observação");
  expect(hoursText(21)).toBe("21 h em observação");
});
it("frequência por extenso", () => {
  expect(frequencyText({ kind: "once" })).toBe("dose única");
  expect(frequencyText({ kind: "interval", interval_hours: 6 })).toBe("de 6 em 6 h");
  expect(frequencyText({ kind: "if_needed", max_per_day: 4 })).toBe("se necessário, até 4 por dia");
});
it("janela da checagem: até 60 min à frente; checado é done; passado sem checagem é late", () => {
  const at = (iso: string, done = false) => ({ scheduled_at: iso, administration: done ? administration() : null });
  expect(slotState(at("2026-11-10T08:00:00-03:00", true), NOW32B)).toBe("done");
  expect(slotState(at("2026-11-10T12:00:00-03:00"), NOW32B)).toBe("late");
  expect(slotState(at("2026-11-10T14:50:00-03:00"), NOW32B)).toBe("due");
  expect(slotState(at("2026-11-10T15:01:00-03:00"), NOW32B)).toBe("future");
  expect(canCheck(at("2026-11-10T15:01:00-03:00"), NOW32B)).toBe(false);
});
it("prescrição: todos os problemas de uma vez, com o path do api", () => {
  const d = { ...emptyItemDraft(), item: null, dose: "0", doseUnit: "", kind: "interval" as const, intervalHours: "25" };
  expect(prescriptionProblems([ d ]).map((p) => p.path)).toEqual([
    "items.0.catalog_item_id", "items.0.dose", "items.0.dose_unit", "items.0.frequency.interval_hours" ]);
  expect(prescriptionProblems([]).map((p) => p.path)).toEqual([ "items" ]);
  expect(prescriptionProblems(Array.from({ length: 21 }, () => okDraft())).map((p) => p.path)).toEqual([ "items" ]);
});
it("prescrição: vírgula decimal, campos só do tipo, primeira dose original mantida", () => {
  const body = prescriptionBody([ { ...okDraft(), dose: "0,5", kind: "if_needed", maxPerDay: "4", intervalHours: "6", firstDoseAt: "" } ]);
  expect(body[0]).toEqual({ catalog_item_id: "cat-dipirona", dose: 0.5, dose_unit: "g", route: "intravenous", frequency: { kind: "if_needed", max_per_day: 4 } });
  expect(prescriptionBody([ itemDraftFrom(prescriptionItem()) ])[0].first_dose_at).toBe("2026-11-10T08:00:00-03:00");
});
it("checagem: motivo fora de administrado; estoque só com o interruptor e só o que mudou", () => {
  const ctx = { stockDeduction: true, defaultLotId: "lot-a", defaultQuantity: "1", now: NOW32B };
  expect(administrationProblems({ status: "refused", reason: "não quis", lotId: "lot-a", quantity: "1", administeredAt: "" }, ctx)
    .map((p) => p.path)).toEqual([ "reason" ]);
  expect(administrationBody({ status: "administered", reason: "", lotId: "lot-a", quantity: "1", administeredAt: "" }, ctx, { itemId: "pi1", slot: "2026-11-10T14:00:00-03:00" }))
    .toEqual({ prescription_item_id: "pi1", scheduled_at: "2026-11-10T14:00:00-03:00", status: "administered" });
  expect(administrationBody({ status: "administered", reason: "", lotId: "lot-b", quantity: "2", administeredAt: "" }, ctx, { itemId: "pi1", slot: null }))
    .toEqual({ prescription_item_id: "pi1", status: "administered", stock_lot_id: "lot-b", stock_quantity: 2 });
  expect(administrationBody({ status: "administered", reason: "", lotId: "lot-b", quantity: "2", administeredAt: "" },
    { ...ctx, stockDeduction: false }, { itemId: "pi1", slot: null })).toEqual({ prescription_item_id: "pi1", status: "administered" });
});
```

Em `src/lib/urgency.test.ts`: `urgencyRoles` com `TECH_LINK` dá `{ technician: true, nurse: false, medication: true, observation: false, mapReader: true }`; o vínculo 3222 encerrado (`ended_at`) não conta; e o caso "todo código do contrato tem frase" ganha a lista do 32b (os 16 códigos acima), conferindo que `urgencyError(new ApiError(422, { error: code }, "x"))` não começa com "recusado pelo servidor". Mais: `urgencyError(new ApiError(422, { error: "invalid_request", path: "items.2.dose" }, "x"))` é "item 3: confira a dose".

Em `src/lib/pharmacy.test.ts`: `movementLabel("administration")`, `movementDocument({ type: "urgency_administration", number: null })` e `pharmacyError` com `already_reconciled`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/lib/urgencyObservation.test.ts src/lib/urgency.test.ts src/lib/pharmacy.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `urgency.ts`: `const TECH_CBO = /^3222/;` ao lado de `NURSE_CBO`/`DOCTOR_CBO`, com o comentário "32b (contrato §1): o técnico checa a medicação e lê o mapa; continua fora da classificação (Q2)". Os quatro campos novos em `UrgencyRoles` e em `urgencyRoles`. Frases (sem interpolar nada do paciente nem do medicamento):
  - `bed_unavailable`: "este leito está inativo ou é de outra unidade — o mapa foi atualizado";
  - `bed_taken`: "outra pessoa ocupou este leito agora — o mapa foi atualizado";
  - `not_in_observation`: "este paciente já não está em observação";
  - `consultation_required`: "inicie a consulta de urgência antes de prescrever";
  - `catalog_item_invalid`: "este medicamento saiu do catálogo — escolha outro";
  - `prescription_not_active`: "esta prescrição não é mais a ativa — a tela foi atualizada";
  - `slot_invalid`: "este horário não está no aprazamento — a tela foi atualizada";
  - `slot_taken`: "este horário já foi checado — a tela foi atualizada";
  - `slot_not_due`: "este horário ainda não chegou — checagem só até 60 minutos antes";
  - `max_per_day_reached`: "já foram dadas todas as doses permitidas nas últimas 24 h";
  - `stock_deduction_disabled`: "a baixa de estoque está desligada — registre sem lote";
  - `lot_invalid`: "o lote escolhido está vencido, sem saldo ou é de outro item — escolha outro";
  - `label_taken`: "já existe um leito com este rótulo nesta UPA";
  - `bed_occupied`: "este leito está ocupado — encerre a observação ou troque o paciente de leito antes";
  - `upa_has_beds`: "esta UPA tem leitos de observação ativos — desative os leitos antes";
  - em `FIELD_TEXT`: `^bed_id$` → "escolha o leito"; `^label$` → "o rótulo tem de 1 a 20 caracteres"; `^items$` → "a prescrição tem de 1 a 20 itens"; `^items[.\[](\d+)\]?\.(.+)$` → "item <n+1>: confira <campo>" (tabela fechada de campos: `catalog_item_id` "o medicamento", `dose` "a dose", `dose_unit` "a unidade da dose", `route` "a via", `frequency…` "a frequência", `first_dose_at` "a primeira dose", `instructions` "a orientação", `stock_quantity_per_dose` "a quantidade por dose"; campo fora da tabela → "item <n+1>: confira o item"); `^scheduled_at$` → "escolha o horário"; `^administered_at$` → "confira o horário da administração"; `^stock_lot_id$` → "escolha o lote"; `^stock_quantity$` → "confira a quantidade".
- `urgencyObservation.ts`: só regras puras; `prescriptionProblems` na ordem do formulário (item a item: medicamento, dose, unidade, via, frequência, primeira dose, orientação, quantidade por dose); `prescriptionBody` manda `interval_hours` só em `interval`, `max_per_day` só em `if_needed`, `instructions` só não vazia, `first_dose_at` só preenchida, `stock_quantity_per_dose` só diferente de 1; `itemDraftFrom` carrega o `first_dose_at` do item (D18). `administrationBody` manda `scheduled_at` só com horário (o `if_needed` não aceita), `reason` só fora de `administered`, `administered_at` só se mudado, e `stock_lot_id`/`stock_quantity` só com `ctx.stockDeduction`, `status === "administered"` e valor diferente do padrão.
- `pharmacy.ts`: `administration: "administração na UPA"` em `MOVEMENT_LABEL` e `urgency_administration: "administração na UPA"` em `DOCUMENT_LABEL` (D15); `already_reconciled` em `ERRORS`; `[ /^note$/, "a nota tem de 10 a 500 caracteres" ]` em `PATH_RULES`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/lib && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/urgencyObservation.ts src/lib/urgencyObservation.test.ts src/lib/urgency.ts src/lib/urgency.test.ts src/lib/pharmacy.ts src/lib/pharmacy.test.ts
$G commit -m "feat: add observation, prescription and medication check screen rules with nursing technician role" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 2: regras e frases dos 16 códigos novos (inclui `slot_not_due`, proposto em D8); técnico 3222 como papel de checagem na tela".

---

### Task 3: Telas novas no menu e no catálogo

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/catalog.ts`, `src/App.tsx`
- Create: `src/modules/UrgencyBedsSettings.tsx`, `src/modules/PharmacyUrgencyShortfalls.tsx` (portão com o `PageHeader` e um `EmptyState`; o corpo vem nas Tasks 4 e 9)
- Test: `src/shell/modules.urgency.test.ts`, `src/shell/modules.pharmacy.test.ts`

**Interfaces:**
- Consumes: `canUseUrgency`, `canUsePharmacy`.
- Produces: `ModuleId` ganha `"urgency-beds"` e `"pharmacy-urgency-shortfalls"`; `NAV_GROUPS` "Urgência" ganha `{ id: "urgency-beds", label: "Leitos de observação", icon: "▭" }` depois de "Senhas, locais e TVs"; "Farmácia" ganha `{ id: "pharmacy-urgency-shortfalls", label: "Faltas da UPA", icon: "⚠" }` depois de "Alertas da farmácia"; `CATALOG.screens` com `IconBedFlat` ("Leitos da observação da UPA, com step-up") e `IconPillOff` ("Medicação dada na UPA sem saldo no estoque"); `App.tsx` com os dois `case`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/shell/modules.urgency.test.ts (casos novos)
it("leitos de observação só do admin, com urgency_care e clinical_record (sem depender do waiting_room_display)", () => {
  expect(ids(urgencyUser([ "municipal_admin" ], [ "urgency_care", "clinical_record" ]))).toContain("urgency-beds");
  expect(ids(urgencyUser([ "municipal_admin" ], [ "waiting_room_display" ]))).not.toContain("urgency-beds");
  expect(ids(urgencyUser([ "analyst" ]))).not.toContain("urgency-beds");
  expect(ids(urgencyUser([ "health_professional" ]))).not.toContain("urgency-beds");
  expect(CATALOG.screens["urgency-beds"].description).toBe("Leitos da observação da UPA, com step-up");
});
```

Em `src/shell/modules.pharmacy.test.ts`, as listas exatas de itens ganham `"pharmacy-urgency-shortfalls"` logo depois de `"pharmacy-alerts"` para o farmacêutico e o almoxarife (o auxiliar e o admin não: o contrato §1 dá a conciliação ao farmacêutico e ao almoxarife); sem `pharmacy`, nada; e o item aparece **sem** `urgency_stock_deduction` (contrato §7). `src/shell/modules.test.ts` não muda (nenhum grupo novo: `NAV_GROUPS.length - 21` e `- 20` seguem).

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/shell`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `modules.ts`: as duas ids na união e nos grupos; no filtro, ao lado das linhas 431–436 e 460–461: `if (item.id === "pharmacy-urgency-shortfalls") return phOn && (isPharmacist || isStockKeeper);` e `if (item.id === "urgency-beds") return urgOn && isAdmin;`, com o comentário "Módulo 32b: leitos da observação são do admin com urgency_care (D1); faltas da UPA, do farmacêutico e do almoxarife com pharmacy, mesmo com a baixa desligada (contrato §7)".
- `catalog.ts`: os dois ícones importados e as duas descrições.
- `App.tsx`: `case "urgency-beds": return <UrgencyBedsSettings onGoToSecurity={() => setActive("security")} />;` e `case "pharmacy-urgency-shortfalls": return <PharmacyUrgencyShortfalls />;`.
- As duas telas, por ora: `PageHeader` ("Leitos de observação" / "Faltas da UPA") e o portão (sem interruptor → `URGENCY_DISABLED` / `PHARMACY_DISABLED`; sem papel → "você não tem acesso aos leitos de observação" / `PHARMACY_NO_ROLE`).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/shell src/App.navigation.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/catalog.ts src/App.tsx src/modules/UrgencyBedsSettings.tsx src/modules/PharmacyUrgencyShortfalls.tsx \
  src/shell/modules.urgency.test.ts src/shell/modules.pharmacy.test.ts
$G commit -m "feat: add observation beds and urgency shortfalls screens to the menu" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 3: 'Leitos de observação' (grupo Urgência) e 'Faltas da UPA' (grupo Farmácia) no menu — decisão do cadastro de leitos registrada como D1".

---

### Task 4: Cadastro de leitos de observação, com step-up (F-32.7)

**Files:**
- Modify: `src/modules/UrgencyBedsSettings.tsx`, `src/modules/attendance/Units.tsx`
- Create: `src/modules/urgency/BedsAdmin.tsx` (cópia de `src/modules/psychosocial/BedsAdmin.tsx`, que não muda)
- Test: `src/modules/UrgencyBedsSettings.test.tsx`, `src/modules/attendance/Units.caps.test.tsx`

**Interfaces:**
- Consumes: `listUrgencyBeds`, `createUrgencyBed`, `updateUrgencyBed` (Task 1); `listActiveUnits`, `isUpa`, `canUseUrgency`, `urgencyError`, `bedLabelProblem`, `URGENCY_BEDS_KEY` (Task 2); `SensitiveAction`, `DataTable`, `Tag`.
- Produces: `UrgencyBedsSettings({ onGoToSecurity })` (escolha da UPA — só `kind === "upa"`, com a mesma chave `[ "activeUnits" ]` do Atendimento — e o `BedsAdmin` da unidade, com `key={unit.id}`); `BedsAdmin({ unit, onGoToSecurity })`; `Units` traduz `upa_has_beds` por `urgencyError`.

- [ ] **Step 1: Escreva os testes**

```tsx
it("só UPAs na escolha; adicionar passa pelo step-up e relê", async () => {
  vi.mocked(api.listActiveUnits).mockResolvedValue([ UPA, UBS ]);
  vi.mocked(api.listUrgencyBeds).mockResolvedValue([ bed() ]);
  vi.mocked(api.createUrgencyBed).mockResolvedValue(bed({ id: "b3", label: "Obs 3" }));
  mount(adminUser());
  const select = await screen.findByLabelText("UPA");
  expect(within(select).queryByRole("option", { name: "UBS Centro" })).toBeNull();
  fireEvent.change(select, { target: { value: "upa1" } });
  fireEvent.change(await screen.findByLabelText("Rótulo do leito"), { target: { value: "Obs 3" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar leito" }));
  await confirmStepUp();
  await waitFor(() => expect(vi.mocked(api.createUrgencyBed).mock.calls[0]).toEqual([ { health_unit_id: "upa1", label: "Obs 3", active: true } ]));
  expect(await screen.findByText("Leito adicionado ao mapa da observação.")).not.toBeNull();
  expect(vi.mocked(api.listUrgencyBeds).mock.calls.length).toBe(2);
});
it("rótulo de 21 caracteres não sai; label_taken e bed_occupied mostram a frase", async () => {
  // …"o rótulo tem de 1 a 20 caracteres"; createUrgencyBed rejeitado 422 label_taken → "já existe um leito com este rótulo nesta UPA";
  // updateUrgencyBed rejeitado 409 bed_occupied → "este leito está ocupado — encerre a observação ou troque o paciente de leito antes".
});
it("sem urgency_care ou sem admin, o portão", async () => {
  // features sem urgency_care → URGENCY_DISABLED; health_professional → "você não tem acesso aos leitos de observação".
});
```

Em `src/modules/attendance/Units.caps.test.tsx`, um caso a mais: desativar uma UPA rejeitado com 422 `upa_has_beds` mostra "esta UPA tem leitos de observação ativos — desative os leitos antes".

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/UrgencyBedsSettings.test.tsx src/modules/attendance/Units.caps.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `urgency/BedsAdmin.tsx`: o molde do 31 com `listUrgencyBeds`/`createUrgencyBed({ health_unit_id, label, active: true })`/`updateUrgencyBed(id, { active })`, `bedLabelProblem`, `urgencyError` como `translateError`, `URGENCY_BEDS_KEY` com `"admin"` e o id da UPA na chave, `STATIC_QUERY`. Textos: título "Leitos de observação · <UPA>", sub "um paciente por leito; o mapa da observação fica na mesa da UPA"; avisos "Leito adicionado ao mapa da observação.", "Leito desativado.", "Leito reativado."; descrição do step-up "O leito “<rótulo>” sai do mapa; quem está nele precisa ter saído antes." (o rótulo do leito não é dado de paciente).
- `UrgencyBedsSettings.tsx`: o portão da Task 3 mais a escolha da UPA (`listActiveUnits` → `filter(isUpa)`), "nenhuma UPA ativa" sem nenhuma, e o `BedsAdmin`.
- `Units.tsx`: `const URGENCY_UNIT_ERRORS = [ "upa_has_beds" ];` e `unitError` tenta `urgencyError` para eles antes do `psyError`/`attendanceError`.

- [ ] **Step 4: Rode e veja passar; guardas; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/UrgencyBedsSettings.test.tsx src/modules/attendance src/modules/psychosocial src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (o `BedsAdmin` do 31 e o `CapsSettings` seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/UrgencyBedsSettings.tsx src/modules/urgency/BedsAdmin.tsx src/modules/UrgencyBedsSettings.test.tsx \
  src/modules/attendance/Units.tsx src/modules/attendance/Units.caps.test.tsx
$G commit -m "feat: manage urgency observation beds with step-up" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 4: cadastro de leitos (GET/POST/PATCH /urgency/beds, step-up) e `upa_has_beds` no cadastro de unidade".

---

### Task 5: Mapa de observação na mesa da UPA, técnico na mesa e coluna na fila (F-32.7)

**Files:**
- Create: `src/modules/urgency/ObservationMap.tsx`
- Modify: `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/UrgencyQueue.tsx`
- Test: `src/modules/urgency/ObservationMap.test.tsx`, `src/modules/urgency/UrgencyQueue.test.tsx`; e os testes que montam a mesa ganham o dublê `getUrgencyObservation` (`mockResolvedValue({ beds: [], waiting_bed: [] })`): `UrgencyReception.test.tsx`, `UrgencyCare.test.tsx`, `ClassificationForm.test.tsx`, `UrgencyQueue.test.tsx`, `IdentifyPatient.test.tsx`, `src/modules/Attendance.urgency.test.tsx`

**Interfaces:**
- Consumes: `getUrgencyObservation`, `startObservation`, `endObservation` (Task 1); `usePolling` (32a); `URGENCY_OBSERVATION_POLL_MS`, `alertText`, `alertTone`, `hoursText`, frases (Task 2); `urgencyError`; `ConfirmButton`, `Panel`, `Tag`, `EmptyState`.
- Produces: `ObservationMap` (`forwardRef<ObservationMapHandle, { unit; roles: UrgencyRoles; onMedication?(target: MedicationTarget): void; now?: Date }>`, com `refresh()`); `MedicationTarget = { encounterId: string; ticket: string | null; patientName: string }` (exportado, usado na Task 7); `UrgencyDesk` monta o mapa entre a fila e a lista de provisórios, aceita o técnico e relê o mapa depois do desfecho; `UrgencyQueue` mostra a coluna "Observação" em "Em atendimento" e aceita o técnico como leitor.

- [ ] **Step 1: Escreva os testes**

```tsx
it("um cartão por leito: livre ou ocupado com senha, nome, horas e alerta por texto; nada clínico", async () => {
  vi.mocked(api.getUrgencyObservation).mockResolvedValue(observationMap({ beds: [
    observationBed({ stay: { ...stayRef({ hours: 25, alert: "over_limit" }), complaint: "dor torácica", medication: "morfina" } as never }),
    observationBed({ bed_id: "b2", label: "Obs 2", stay: null })
  ] }));
  mountMap(roles({ nurse: true }));
  const occupied = await screen.findByRole("region", { name: "Obs 1" });
  expect(within(occupied).getByText("U-038 · João Reis")).not.toBeNull();
  expect(within(occupied).getByText("25 h em observação")).not.toBeNull();
  expect(within(occupied).getByText("mais de 24 h em observação")).not.toBeNull();
  expect(screen.queryByText(/dor torácica|morfina/)).toBeNull();
  expect(within(screen.getByRole("region", { name: "Obs 2" })).getByText("livre")).not.toBeNull();
});
it("entrada: escolhe quem está em atendimento sem leito; uma chamada; relê", async () => {
  vi.mocked(api.startObservation).mockResolvedValue(stay({ stay_id: "st2", bed: { id: "b2", label: "Obs 2" } }));
  mountMap(roles({ doctor: true }));
  fireEvent.click(await screen.findByRole("button", { name: "Colocar em observação no Obs 2" }));
  fireEvent.change(screen.getByLabelText("Paciente em atendimento"), { target: { value: "en4" } });
  const send = screen.getByRole("button", { name: "Confirmar entrada" });
  fireEvent.click(send); fireEvent.click(send);
  await waitFor(() => expect(vi.mocked(api.startObservation).mock.calls).toEqual([ [ "en4", "b2" ] ]));
  expect(await screen.findByText("Paciente em observação.")).not.toBeNull();
});
it("bed_taken relê o mapa na hora e fecha o formulário; troca só para leito livre e ativo", async () => {
  vi.mocked(api.startObservation).mockRejectedValue(new ApiError(409, { error: "bed_taken" }, "x"));
  // … "outra pessoa ocupou este leito agora — o mapa foi atualizado"; getUrgencyObservation chamado de novo sem esperar 30 s.
  // "Trocar de leito" no Obs 1 oferece só "Obs 2" (o inativo e o ocupado ficam fora); confirma → startObservation("en3", "b2").
});
it("encerrar sem desfecho pede confirmação; not_in_observation relê", async () => {
  // "Encerrar observação de U-038" → ConfirmButton "O paciente volta ao atendimento, sem leito. Confirmar?" → endObservation("en3").
});
it("papéis: recepção e técnico só leem; o técnico vê 'Medicação'; sem nenhum, não pede", async () => {
  // reception → nenhum botão de escrita; technician → só "Medicação de U-038"; doctor → entrada, troca, encerrar e "Medicação".
});
it("polling de 30 s, um pedido por vez, nada depois de desmontar", async () => {
  vi.useFakeTimers();
  vi.mocked(api.getUrgencyObservation).mockResolvedValue(observationMap());
  const { unmount } = mountMap(roles({ nurse: true }));
  await vi.advanceTimersByTimeAsync(0);
  expect(vi.mocked(api.getUrgencyObservation).mock.calls.length).toBe(1);
  await vi.advanceTimersByTimeAsync(30_000);
  expect(vi.mocked(api.getUrgencyObservation).mock.calls.length).toBe(2);
  unmount();
  await vi.advanceTimersByTimeAsync(120_000);
  expect(vi.mocked(api.getUrgencyObservation).mock.calls.length).toBe(2);
  expect(vi.getTimerCount()).toBe(0);
});
```

Em `UrgencyQueue.test.tsx`: a linha de "Em atendimento" com `observation: { bed_label: "Obs 1", hours: 21, alert: "near_limit" }` mostra "Obs 1 · 21 h em observação · perto de 24 h em observação" na coluna "Observação"; sem `observation`, "—"; um campo extra dentro de `observation` (`medication`) não aparece. E na mesa (`UrgencyCare.test.tsx` ou `Attendance.urgency.test.tsx`): um técnico (`TECH_LINK`) vê o mapa e não vê "seu papel não atua nesta UPA"; a fila, se o api responder 403 `cbo_not_allowed` (D2), mostra a frase e o mapa continua.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ObservationMap.tsx`: `Panel` "Observação" (sub "um paciente por leito · horas desde a 1ª entrada · atualiza a cada 30 s"); `usePolling(roles.mapReader ? () => getUrgencyObservation(unit.id) : null, URGENCY_OBSERVATION_POLL_MS, unit.id)`, com a mesma parada nas recusas 403/404/422 e o rodapé "atualiza a cada 30 s · hh:mm" / "parado às hh:mm" + "Tentar de novo" do `UrgencyQueue`; linhas campo a campo (`safeBed`, `safeStay`, `safeWaiting`); cartões (`BedCard` copiado de `CapsBeds.tsx`, `section aria-label={label}`, `flex: "1 1 240px"`, `minWidth: 0`): livre → `Tag tone="ok"` "livre" e, para `roles.observation`, "Colocar em observação" (`aria-label="Colocar em observação no <leito>"`); ocupado → "<senha> · <nome>" (e "sem identificação" no provisório), "Entrada hh:mm", `hoursText(hours)` e o alerta em `Tag` com o texto, e os botões: "Trocar de leito", "Encerrar observação" (`ConfirmButton`), "Medicação" (para `roles.medication`, chama `onMedication`). Leitos inativos com estadia aberta aparecem no grupo "Leitos inativos" (o api só desativa leito livre, `bed_occupied`; defensivo). A entrada abre um formulário inline abaixo do cartão (`select` "Paciente em atendimento" com os `waiting_bed[]` — senha e nome; vazio → "ninguém em atendimento sem leito"); a troca, um `select` "Novo leito" só com leitos ativos e livres. Envio único (`busy` + `useRef`), `seq` por leito; depois de qualquer escrita (ou das recusas `bed_taken`, `bed_unavailable`, `not_in_care`, `not_in_observation`, `encounter_closed`), `polling.refresh()` e o formulário fecha; o aviso em `role="status"`. Nenhum texto do paciente na frase de aviso.
- `UrgencyDesk.tsx`: o portão de papel passa a `!roles.mapReader` (o técnico entra); `mapRef` e `<ObservationMap ref={mapRef} unit={unit} roles={roles} onMedication={…} />` entre a fila e `ProvisionalEncounters`; `closeCare`/`closeLeaving` com desfecho também chamam `mapRef.current?.refresh()` (o desfecho fecha a estadia, contrato §4). A fila só monta para `reception || nurse || doctor || technician`; o técnico não ganha ação nenhuma da fila além da "Medicação" (Task 7).
- `UrgencyQueue.tsx`: `QueueRoles` ganha `technician?: boolean` e `hasRole` o considera (D2); `safeInCare` copia `observation` campo a campo (`bed_label`, `hours`, `alert`); a coluna "Observação" (`w: "1.2fr"`) com o texto acima.

- [ ] **Step 4: Rode e veja passar; guardas; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency src/modules/Attendance.urgency.test.tsx src/theme src/lib/usePolling.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/ObservationMap.tsx src/modules/urgency/ObservationMap.test.tsx src/modules/urgency/UrgencyDesk.tsx \
  src/modules/urgency/UrgencyQueue.tsx src/modules/urgency/UrgencyQueue.test.tsx src/modules/urgency/UrgencyReception.test.tsx \
  src/modules/urgency/UrgencyCare.test.tsx src/modules/urgency/ClassificationForm.test.tsx src/modules/urgency/IdentifyPatient.test.tsx \
  src/modules/Attendance.urgency.test.tsx
$G commit -m "feat: add the observation bed map to the urgency desk" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 5: mapa de observação na mesa (polling 30 s, entrada/troca/saída), coluna na fila; o técnico depende de D2 (leitura da fila) e D3 (`waiting_bed`)".

---

### Task 6: Prescrição interna — emitir, alterar em versão nova e suspender (F-32.8)

**Files:**
- Create: `src/modules/urgency/PrescriptionPanel.tsx`, `src/modules/urgency/PrescriptionItemFields.tsx`
- Modify: `src/modules/urgency/UrgencyCare.tsx`
- Test: `src/modules/urgency/PrescriptionPanel.test.tsx`

**Interfaces:**
- Consumes: `issueUrgencyPrescription`, `suspendUrgencyPrescription`, `searchMedications` (Task 1 / 19c); `ItemDraft`, `emptyItemDraft`, `itemDraftFrom`, `prescriptionProblems`, `prescriptionBody`, `frequencyText`, `reasonProblem`, `MAX_ITEMS` (Task 2); `ROUTES`, `ROUTE_LABEL`; `MedicationSearch` com `blockControlled={false}` e sem `blockReason` (P1); `urgencyRefusals`; `ConfirmButton`.
- Produces: `PrescriptionPanel({ encounter: UrgencyEncounter; unit: HealthUnit; consultationStarted: boolean; onChanged(): void })`: lê `encounter.prescription` (do detalhe já lido pelo `UrgencyCare`); `PrescriptionItemFields({ index, draft, onChange, onRemove, problems, unitId })`.

- [ ] **Step 1: Escreva os testes**

```tsx
it("sem consulta iniciada, pede a consulta antes; sem prescrição, 'Prescrever'", async () => {
  mountPanel(observedEncounter({ consultation_id: null, consultation_status: null, prescription: null }), { consultationStarted: false });
  expect(screen.getByText("inicie a consulta de urgência antes de prescrever")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Prescrever" })).toBeNull();
});
it("emitir: item do catálogo, dose com vírgula, intervalo; todas as recusas de uma vez; uma chamada", async () => {
  vi.mocked(api.searchMedications).mockResolvedValue([ medication({ id: "cat-dipirona", label: "Dipirona 500 mg/mL" }) ]);
  vi.mocked(api.issueUrgencyPrescription).mockResolvedValue(prescription());
  mountPanel(observedEncounter({ prescription: null }));
  fireEvent.click(screen.getByRole("button", { name: "Prescrever" }));
  fireEvent.click(screen.getByRole("button", { name: "Emitir prescrição" }));
  expect(screen.getByText("item 1: escolha o medicamento do catálogo")).not.toBeNull();
  expect(screen.getByText("item 1: informe a dose")).not.toBeNull();
  await pickMedication("Medicamento do item 1", "dipi", "Dipirona 500 mg/mL");
  fireEvent.change(screen.getByLabelText("Dose do item 1"), { target: { value: "0,5" } });
  fireEvent.change(screen.getByLabelText("Unidade da dose do item 1"), { target: { value: "g" } });
  fireEvent.change(screen.getByLabelText("Via do item 1"), { target: { value: "intravenous" } });
  fireEvent.click(screen.getByRole("radio", { name: "De tantas em tantas horas (item 1)" }));
  fireEvent.change(screen.getByLabelText("Intervalo em horas do item 1"), { target: { value: "6" } });
  const send = screen.getByRole("button", { name: "Emitir prescrição" });
  fireEvent.click(send); fireEvent.click(send);
  await waitFor(() => expect(vi.mocked(api.issueUrgencyPrescription).mock.calls).toEqual([ [ "en3", [ { catalog_item_id: "cat-dipirona",
    dose: 0.5, dose_unit: "g", route: "intravenous", frequency: { kind: "interval", interval_hours: 6 } } ] ] ]));
  expect(await screen.findByText("Prescrição emitida (versão 1).")).not.toBeNull();
});
it("controlado aceito, com a marca, sem o bloqueio do 19d (P1)", async () => {
  vi.mocked(api.searchMedications).mockResolvedValue([ medication({ id: "cat-morfina", label: "Morfina 10 mg/mL", controlled: true, controlled_list: "A1" }) ]);
  // o resultado não está desabilitado; escolhido, o item mostra "controlado · lista A1"; o corpo leva catalog_item_id "cat-morfina".
});
it("alterar abre os itens da ativa, mantém a primeira dose e cria a versão nova", async () => {
  vi.mocked(api.issueUrgencyPrescription).mockResolvedValue(prescription({ id: "pr2", version: 2, supersedes_id: "pr1" }));
  mountPanel(observedEncounter());
  fireEvent.click(screen.getByRole("button", { name: "Alterar prescrição" }));
  fireEvent.change(screen.getByLabelText("Dose do item 1"), { target: { value: "2" } });
  fireEvent.click(screen.getByRole("button", { name: "Emitir versão 2" }));
  await waitFor(() => expect(vi.mocked(api.issueUrgencyPrescription).mock.calls[0][1][0]).toMatchObject({
    dose: 2, first_dose_at: "2026-11-10T08:00:00-03:00" }));
  expect(await screen.findByText("Versão 2 ativa; a versão 1 ficou substituída.")).not.toBeNull();
});
it("suspender: motivo de 10 caracteres e confirmação; prescription_not_active relê", async () => {
  // "Suspender prescrição" → "Motivo da suspensão" com 9 caracteres → REASON_TEXT; com 10 → ConfirmButton
  // "A prescrição suspensa não volta; para retomar, prescreva de novo. Confirmar?" → suspendUrgencyPrescription("pr1", motivo) → onChanged.
  // 422 prescription_not_active → a frase e onChanged (o UrgencyCare relê o detalhe).
});
it("recusa do api com problems[] mostra cada item; catalog_item_invalid no item certo", async () => {
  // 422 { error: "invalid_request", problems: [ { error: "catalog_item_invalid", path: "items.0.catalog_item_id" },
  //   { error: "invalid_request", path: "items.0.frequency.interval_hours" } ] } → as duas frases, nada apagado.
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency/PrescriptionPanel.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `PrescriptionPanel.tsx`: `<section aria-label="Prescrição da urgência">`. Leitura: a versão ativa (versão, quem e quando, e cada item numa linha: rótulo do catálogo, "controlado · lista X" em `Tag tone="warn"` quando `controlled_list`, dose e unidade, `ROUTE_LABEL[route] ?? route`, `frequencyText`, "1ª dose hh:mm", orientação quando houver); sem ativa, "Sem prescrição ativa" e, se a última foi suspensa, "Suspensa em <data> — o motivo fica no prontuário" (o motivo **não** aparece fora do detalhe). Botões do médico (`roles.doctor`, recebido do `UrgencyCare`): "Prescrever" (sem ativa), "Alterar prescrição" e "Suspender prescrição" (com ativa). O formulário: 1 a 20 `PrescriptionItemFields` ("Adicionar item" some no 20º), "Remover item" por item, "Emitir prescrição" / "Emitir versão N+1", nota fixa "Alterar sempre cria uma versão nova; a anterior fica registrada como está."; problemas por item com `role="alert"` junto do campo; `urgencyRefusals` do api distribuídos pelo `path`; envio único.
- `PrescriptionItemFields.tsx`: `MedicationSearch` ("Medicamento do item N", `unitId={unit.id}`, `blockControlled={false}`) — o escolhido vira um cartão com "Trocar"; "Dose do item N" (`inputMode="decimal"`), "Unidade da dose do item N", "Via do item N" (`select` com `ROUTES`), frequência em `fieldset` com três rádios ("Dose única", "De tantas em tantas horas", "Se necessário"), "Intervalo em horas do item N" só em `interval`, "Máximo por dia do item N" só em `if_needed`, "Primeira dose (opcional) do item N" (`datetime-local`, vazio = agora), "Orientação (opcional) do item N" (`textarea`, até 500), "Quantidade do estoque por dose do item N" (padrão 1, com a `dispensing_unit` do item quando conhecida, D6, e a nota "usada só quando a baixa de estoque estiver ligada").
- `UrgencyCare.tsx`: o `PrescriptionPanel` entre o `ConsultationWorkspace` e o `OutcomeForm`, com `consultationStarted = encounter.consultation_id !== null && encounter.consultation_status !== "discarded"`; `onChanged` = `detail.refetch()`.

- [ ] **Step 4: Rode e veja passar; regressão dos documentos (a `MedicationSearch` do 19c não muda); tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency src/modules/documents src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/PrescriptionPanel.tsx src/modules/urgency/PrescriptionItemFields.tsx src/modules/urgency/PrescriptionPanel.test.tsx \
  src/modules/urgency/UrgencyCare.tsx
$G commit -m "feat: issue, revise and suspend the urgency internal prescription" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 6: prescrição interna com versões e suspensão, controlados aceitos (P1); a busca usa /attendance/medications/search, que exige clinical_documents (D6)".

---

### Task 7: Aprazamento e checagem, com estoque e falta; painel de medicação da enfermagem (F-32.8)

**Files:**
- Create: `src/modules/urgency/MedicationPanel.tsx`, `src/modules/urgency/AdministrationForm.tsx`
- Modify: `src/modules/urgency/UrgencyCare.tsx`, `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/UrgencyQueue.tsx`
- Test: `src/modules/urgency/MedicationPanel.test.tsx`

**Interfaces:**
- Consumes: `getMedicationSchedule`, `recordAdministration` (Task 1); `slotState`, `canCheck`, `administrationProblems`, `administrationBody`, `adminStatusLabel`, `shortfallCauseLabel`, `frequencyText`, `CHECK_IMMUTABLE`, `STOCK_PENDING`, `MEDICATION_SCHEDULE_KEY` (Task 2); `NOMINAL_QUERY`; `urgencyRefusals`; `ConfirmButton`; `MedicationTarget` (Task 5).
- Produces: `MedicationPanel({ target: MedicationTarget; now?: Date; onClose(): void; onEncounterClosed(): void })`; `AdministrationForm({ item: ScheduleItem; slot: string | null; stockDeduction: boolean; now: Date; onRecorded(r: UrgencyAdministrationResult): void; onStale(): void; onCancel(): void })`; `QueueActions` ganha `onMedication?(row: UrgencyInCareRow): void` (botão "Medicação" em "Em atendimento" para `roles.medication`); `UrgencyDesk` abre o `MedicationPanel` pelo mapa e pela fila (um por vez, `key={encounterId}`, foco no título por `useReveal`, Escape fecha); `UrgencyCare` mostra o mesmo `MedicationPanel` abaixo da prescrição quando há versão ativa.

- [ ] **Step 1: Escreva os testes**

```tsx
it("aprazamento lido sob demanda, sem polling; horários com estado por texto", async () => {
  vi.mocked(api.getMedicationSchedule).mockResolvedValue(schedule());
  mountPanel();
  const item = await screen.findByRole("region", { name: "Dipirona 500 mg/mL solução injetável" });
  expect(within(item).getByText("08:00 · administrado por Ana Souza às 08:05")).not.toBeNull();
  expect(within(item).getByRole("button", { name: "Checar 14:00 — Dipirona 500 mg/mL solução injetável" })).not.toBeNull();
  expect(within(item).queryByRole("button", { name: /Checar 20:00/ })).toBeNull();
  expect(within(item).getByText("20:00 · a seguir")).not.toBeNull();
  expect(screen.getByText(/lido às \d{2}:\d{2}/)).not.toBeNull();
  expect(vi.mocked(api.getMedicationSchedule).mock.calls.length).toBe(1);
});
it("checar: confirmação forte, uma chamada só, relê depois", async () => {
  vi.mocked(api.getMedicationSchedule).mockResolvedValue(schedule());
  vi.mocked(api.recordAdministration).mockResolvedValue({ administration: administration({ id: "ad2", scheduled_at: "2026-11-10T14:00:00-03:00" }), shortfall: null });
  mountPanel();
  fireEvent.click(await screen.findByRole("button", { name: "Checar 14:00 — Dipirona 500 mg/mL solução injetável" }));
  fireEvent.click(screen.getByRole("radio", { name: "Administrado" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar checagem" }));
  expect(screen.getByText("A checagem não pode ser alterada. Confirmar?")).not.toBeNull();
  const yes = screen.getByRole("button", { name: "Confirmar" });
  fireEvent.click(yes); fireEvent.click(yes);
  await waitFor(() => expect(vi.mocked(api.recordAdministration).mock.calls).toEqual([ [ {
    prescription_item_id: "pi1", scheduled_at: "2026-11-10T14:00:00-03:00", status: "administered" } ] ]));
  await waitFor(() => expect(vi.mocked(api.getMedicationSchedule).mock.calls.length).toBe(2));
});
it("não administrado e recusado exigem motivo de 10 caracteres", async () => {
  // "Recusado" + "não quis" → "o motivo tem de 10 a 500 caracteres"; nada enviado.
});
it("slot_taken relê e mostra quem checou; encounter_closed fecha o painel", async () => {
  // 409 slot_taken → "este horário já foi checado — a tela foi atualizada" + getMedicationSchedule de novo;
  // 409 encounter_closed → onEncounterClosed().
});
it("se necessário: sem horário, quantas ainda cabem; max_per_day_reached", async () => {
  vi.mocked(api.getMedicationSchedule).mockResolvedValue(schedule({ items: [ scheduleItem({
    item: prescriptionItem({ id: "pi2", label: "Ondansetrona 2 mg/mL", frequency: { kind: "if_needed", max_per_day: 3 } }), slots: [],
    if_needed: { last_24h: [ administration({ prescription_item_id: "pi2", scheduled_at: null }) ], remaining: 2 } }) ] }));
  // "Se necessário, até 3 por dia · ainda cabem 2 nas últimas 24 h"; "Registrar dose — Ondansetrona 2 mg/mL" → corpo sem scheduled_at;
  // remaining 0 → o botão some e aparece "já foram dadas todas as doses permitidas nas últimas 24 h".
});
it("estoque ligado: lote FEFO e quantidade; só o que mudou vai no corpo; falta é aviso, não erro", async () => {
  vi.mocked(api.getMedicationSchedule).mockResolvedValue(schedule({ stock_deduction: true, items: [ scheduleItem({ lots: [
    { stock_lot_id: "lot-a", lot_number: "A123", expires_on: "2026-12-01", balance: "3" },
    { stock_lot_id: "lot-b", lot_number: "B456", expires_on: "2027-03-01", balance: "10" } ] }) ] }));
  vi.mocked(api.recordAdministration).mockResolvedValue({ administration: administration(), shortfall: { cause: "insufficient_balance" } });
  // o select "Lote" vem em "A123 · vence 01/12/2026 (o que vence primeiro)"; "Quantidade (ampola)" = 1; sem mexer → corpo sem stock_*;
  // escolhendo B456 → stock_lot_id "lot-b"; resposta com shortfall → "Checagem registrada. O estoque ficou com pendência para a farmácia
  // (sem saldo suficiente)." em role="status".
});
it("estoque desligado: nenhum campo de lote e nada de stock_* no corpo; lot_invalid relê mantendo o resto", async () => {
  // stock_deduction false → queryByLabelText("Lote") null; 422 lot_invalid (com true) → frase, releitura, status e horário mantidos.
});
it("o técnico abre pela fila e pelo mapa; o painel não lê o detalhe do encontro", async () => {
  // UrgencyDesk com TECH_LINK: "Medicação de U-038" no mapa abre o painel; getUrgencyEncounter não é chamado.
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency/MedicationPanel.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `MedicationPanel.tsx`: `Panel` "Medicação — <senha>" (sub "aprazamento das próximas 24 h · a checagem não pode ser alterada"); `useQuery({ queryKey: [ MEDICATION_SCHEDULE_KEY, encounterId ], queryFn: () => getMedicationSchedule(encounterId), ...NOMINAL_QUERY })`; sai do cache ao fechar (`removeQueries`); rodapé "lido às hh:mm" + "Atualizar" (`refetch`), sem timer. Sem prescrição ativa: "Sem prescrição ativa — a prescrição é do médico". Cada item é uma `section aria-label={label}` com o rótulo, a marca de controlado (P1), dose, via e `frequencyText`, a orientação, e "última dose: hh:mm (versão anterior)" quando `last_administration` vier de outra versão (D18). Horários em lista: `done` → "hh:mm · <status> por <nome> às hh:mm" (e "· lote <número>" com estoque, "· pendência para a farmácia" com falta); `late` → `Tag tone="warn"` "pendente desde hh:mm" + "Checar"; `due` → "hh:mm · agora" + "Checar"; `future` → "hh:mm · a seguir" (sem botão, D8). `if_needed`: "ainda cabem N nas últimas 24 h" + "Registrar dose", e a lista das últimas 24 h. Uma `AdministrationForm` aberta por vez, logo abaixo do horário.
- `AdministrationForm.tsx`: `fieldset` "Situação" (três rádios); "Motivo" (`textarea`) fora de "Administrado"; "Horário da administração" (`datetime-local`, padrão agora; não aceita futuro); com `stockDeduction` e "Administrado": "Lote" (`select` com os `lots` do item na ordem do api, o 1º marcado "(o que vence primeiro)"; sem `lots`, a nota "o lote sai pelo que vence primeiro, no estoque da UPA") e "Quantidade (<dispensing_unit>)" com o `stock_quantity_per_dose`; "Registrar checagem" → `ConfirmButton` ("A checagem não pode ser alterada. Confirmar?"); recusas com `urgencyRefusals`; `slot_taken`, `slot_invalid`, `prescription_not_active`, `lot_invalid` e `max_per_day_reached` chamam `onStale` (relê) e mantêm o que foi digitado; `encounter_closed` sobe.
- `UrgencyDesk.tsx`: estado `medication: MedicationTarget | null`; abre pelo `ObservationMap.onMedication` e pela fila (`onMedication` para `roles.medication`); fecha com Escape fora dos campos; `onEncounterClosed` fecha, avisa `URGENCY_ERRORS.encounter_closed` e relê fila e mapa.
- `UrgencyQueue.tsx`: botão "Medicação" em "Em atendimento" quando `onMedication && (roles.nurse || roles.technician || roles.doctor)`.
- `UrgencyCare.tsx`: com `encounter.prescription?.status === "active"`, o `MedicationPanel` (o médico também checa) abaixo da prescrição.

- [ ] **Step 4: Rode e veja passar; guardas; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/MedicationPanel.tsx src/modules/urgency/AdministrationForm.tsx src/modules/urgency/MedicationPanel.test.tsx \
  src/modules/urgency/UrgencyCare.tsx src/modules/urgency/UrgencyDesk.tsx src/modules/urgency/UrgencyQueue.tsx
$G commit -m "feat: check scheduled medication doses with optional stock deduction" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 7: aprazamento e checagem (enfermagem, técnico, médico), lote e quantidade só com `stock_deduction: true`; depende de D5 (forma do aprazamento e `lots`), D8 (janela de 60 min) e D9 (resposta com `shortfall`)".

---

### Task 8: Observação no atendimento, evolução pelo adendo e desfecho que encerra a estadia (F-32.7)

**Files:**
- Create: `src/modules/urgency/ObservationSummary.tsx`
- Modify: `src/modules/urgency/UrgencyCare.tsx`, `src/modules/urgency/OutcomeForm.tsx`
- Test: `src/modules/urgency/UrgencyCare.observation.test.tsx`

**Interfaces:**
- Consumes: `encounter.observation`, `encounter.administrations` (Task 1); `alertText`, `hoursText`, `endKindLabel`, `adminStatusLabel` (Task 2); `ConsultationWorkspace` (o adendo do 19a já existe na consulta finalizada, `src/modules/consultation/ConsultationView.tsx:65,199`).
- Produces: `ObservationSummary({ block: UrgencyObservationBlock; administrations: UrgencyAdministration[] })`; `OutcomeForm` ganha `inObservation?: { bedLabel: string } | null` e `prescriptionActive?: boolean`.

- [ ] **Step 1: Escreva os testes**

```tsx
it("no atendimento: leito, horas, alerta, estadias anteriores e checagens das últimas 48 h", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(observedEncounter({ observation: observationBlock({ hours: 21, alert: "near_limit",
    stays: [ stay({ stay_id: "st0", bed: { id: "b2", label: "Obs 2" }, ended_at: "2026-11-10T10:00:00-03:00", end_kind: "bed_change" }), stay() ] }) }));
  mountCare();
  const block = await screen.findByRole("region", { name: "Observação" });
  expect(within(block).getByText("Obs 1 · 21 h em observação")).not.toBeNull();
  expect(within(block).getByText("perto de 24 h em observação")).not.toBeNull();
  expect(within(block).getByText("Obs 2 · 08:00–10:00 · troca de leito")).not.toBeNull();
  expect(within(block).getByText(/administrado por Ana Souza/)).not.toBeNull();
});
it("evolução: o atalho leva ao adendo da consulta finalizada", async () => {
  // consultation_status "finalized" → botão "Registrar evolução (adendo)" foca o formulário de adendo do ConsultationView
  // (dublê com `section aria-label="Adendo"`); sem consulta finalizada, a frase "a evolução é um adendo da consulta: finalize a consulta antes".
});
it("o desfecho avisa que encerra a estadia e a prescrição", async () => {
  // OutcomeForm com inObservation { bedLabel: "Obs 1" } e prescriptionActive → "O desfecho encerra a observação no Obs 1 e a prescrição ativa."
});
it("sem acesso clínico os blocos não vêm e nada quebra", async () => {
  // receptionEncounter() sem observation/prescription/administrations → sem a região "Observação".
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency/UrgencyCare.observation.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ObservationSummary.tsx`: `<section aria-label="Observação">`; estadia aberta "<leito> · <horas>" e o alerta por texto; sem estadia aberta e com estadias anteriores, "Saiu da observação às hh:mm"; lista curta das estadias (leito, início–fim, `endKindLabel`); "Checagens das últimas 48 h" (horário, situação, quem; o motivo só aqui, no detalhe clínico, com `whiteSpace: "pre-wrap"`). Campo a campo.
- `UrgencyCare.tsx`: `ObservationSummary` entre o `EncounterSummary` e o `ConsultationWorkspace` quando `encounter.observation` vem; o atalho "Registrar evolução (adendo)" com a consulta finalizada (rola e foca o `AddendumForm` pelo `aria-label`; não cria formulário novo, contrato §4); `OutcomeForm` recebe `inObservation` e `prescriptionActive`.
- `OutcomeForm.tsx`: a frase acima do botão quando um dos dois vem; nada muda no corpo do desfecho (quem fecha estadia e prescrição é o api, contrato §4 e §5).

- [ ] **Step 4: Rode e veja passar; regressão do 32a; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/urgency src/modules/consultation && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/ObservationSummary.tsx src/modules/urgency/UrgencyCare.tsx src/modules/urgency/OutcomeForm.tsx \
  src/modules/urgency/UrgencyCare.observation.test.tsx
$G commit -m "feat: show the observation stay in urgency care and warn that the outcome closes it" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 8: observação no atendimento e atalho ao adendo; a evolução por outro médico de plantão depende de D10 (adendo só da autora)".

---

### Task 9: Faltas da UPA na farmácia e contagem nos alertas (F-32.8)

**Files:**
- Modify: `src/modules/PharmacyUrgencyShortfalls.tsx`, `src/modules/PharmacyAlerts.tsx`
- Test: `src/modules/PharmacyUrgencyShortfalls.test.tsx`, `src/modules/PharmacyAlerts.test.tsx`

**Interfaces:**
- Consumes: `listUrgencyShortfalls`, `reconcileUrgencyShortfall` (Task 1); `shortfallCauseLabel`, `reasonProblem`, `URGENCY_SHORTFALLS_KEY`, `SHORTFALL_NO_PATIENT` (Task 2); `PharmacyGate`, `LocationPick`, `locationRoles`, `pharmacyError`, `fmtQuantity`, `STATIC_QUERY`; `SegmentedControl` (`src/shell/SegmentedControl.tsx`).
- Produces: `PharmacyUrgencyShortfalls()` — `PharmacyGate` ("Faltas da UPA", `allow={(r) => r.pharmacist || r.keeper}`) → `LocationPick allow={(r) => r.pharmacist || r.keeper}` (o local escolhido tem de ser `kind === "unit"`; outro tipo → "as faltas da UPA ficam no local de farmácia da unidade") → abas "Abertas" / "Conciliadas" → `DataTable`; `PharmacyAlerts` mostra "Faltas da UPA: N" com o atalho quando `urgency_shortfalls` vem.

- [ ] **Step 1: Escreva os testes**

```tsx
it("lista campo a campo: item, quantidade, causa, senha e hora; nunca o paciente", async () => {
  vi.mocked(api.listUrgencyShortfalls).mockResolvedValue([ { ...shortfall(), patient: { name: "João Reis" } } as never ]);
  mountShortfalls(pharmacistAt(unitLocation()));
  const rows = await screen.findAllByRole("row");
  expect(screen.getByText("Dipirona 500 mg/mL solução injetável")).not.toBeNull();
  expect(screen.getByText("1 ampola")).not.toBeNull();
  expect(screen.getByText("sem saldo suficiente")).not.toBeNull();
  expect(screen.getByText("U-038 · 10/11/2026 08:05")).not.toBeNull();
  expect(screen.queryByText(/João Reis/)).toBeNull();
  expect(vi.mocked(api.listUrgencyShortfalls).mock.calls[0]).toEqual([ "loc-upa", "open" ]);
  expect(rows.length).toBeGreaterThan(1);
});
it("conciliar: nota de 10 caracteres, uma chamada, some das abertas; already_reconciled relê", async () => {
  vi.mocked(api.listUrgencyShortfalls).mockResolvedValueOnce([ shortfall() ]).mockResolvedValueOnce([]);
  vi.mocked(api.reconcileUrgencyShortfall).mockResolvedValue(shortfall({ reconciled_at: "2026-11-10T15:00:00-03:00" }));
  // "Conciliar falta de Dipirona…" → "Nota da conciliação" curta → "a nota tem de 10 a 500 caracteres";
  // "ajuste feito no estoque da UPA" → duplo clique → uma chamada ("sf1", nota) → "Falta conciliada." e a lista relida vazia.
  // 409 already_reconciled → "esta falta já foi conciliada por outra pessoa — a lista foi atualizada" e releitura.
});
it("conciliadas: data e quem conciliou; sem botão", async () => {});
it("auxiliar não concilia; local que não é de unidade explica", async () => {});
```

Em `PharmacyAlerts.test.tsx`: com `urgency_shortfalls: 2`, a linha "Faltas da UPA: 2" e o link "Faltas da UPA" (para o farmacêutico ou almoxarife do local); sem o campo (api do 27a), a linha não aparece.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/PharmacyUrgencyShortfalls.test.tsx src/modules/PharmacyAlerts.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `PharmacyUrgencyShortfalls.tsx`: `useQuery({ queryKey: [ URGENCY_SHORTFALLS_KEY, loc.id, state ], queryFn: …, ...STATIC_QUERY })` (sem trilha, sem dado pessoal: relê depois de cada conciliação e ao trocar de aba); colunas "Item" (`card: "title"`), "Quantidade" (`fmtQuantity` + `dispensing_unit`), "Causa", "Administração" (senha · data e hora), e, em "Conciliadas", "Conciliada" (data · nome); ação "Conciliar" (`card: "actions"`, `aria-label="Conciliar falta de <item>"`) abre a nota inline (`textarea`, 10–500) e "Confirmar conciliação"; nota fixa "Resolva no estoque antes (ajuste ou transferência); conciliar não gera movimento." e `SHORTFALL_NO_PATIENT` ("A lista não mostra o paciente nem dado clínico."). Erros por `pharmacyError`.
- `PharmacyAlerts.tsx`: depois da linha "Ajustes aguardando aprovação", `data.urgency_shortfalls !== undefined && line(\`Faltas da UPA: ${data.urgency_shortfalls}\`, canShortfalls && <a href={screenLink("pharmacy-urgency-shortfalls")}>Faltas da UPA</a>)`, com `canShortfalls = roles.pharmacist || roles.keeper`.

- [ ] **Step 4: Rode e veja passar; regressão da farmácia; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/Pharmacy* src/modules/pharmacy src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/PharmacyUrgencyShortfalls.tsx src/modules/PharmacyUrgencyShortfalls.test.tsx src/modules/PharmacyAlerts.tsx src/modules/PharmacyAlerts.test.tsx
$G commit -m "feat: list and reconcile urgency stock shortfalls in the pharmacy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 9: 'Faltas da UPA' com conciliação e a contagem nos alertas; forma da lista = D16".

---

### Task 10: Observação no Painel da urgência e na linha do tempo (F-32.7)

**Files:**
- Modify: `src/modules/UrgencyPanel.tsx`, `src/modules/timeline/PatientTimeline.tsx`
- Test: `src/modules/UrgencyPanel.test.tsx`, `src/lib/timeline.urgency.test.ts` (ou `src/modules/timeline/PatientTimeline.test.tsx`, onde o render da linha é testado)

**Interfaces:**
- Consumes: `UrgencyDashboard.observation`, `TimelineEntry.observation_hours` (Task 1); `countLabel`, `SMALL_COUNT` (32a).
- Produces: bloco "Observação" no painel; "Observação: N h" na entrada `urgency` da linha do tempo profissional.

- [ ] **Step 1: Escreva os testes**

```tsx
it("bloco de observação com 'menos de 5'", async () => {
  vi.mocked(api.getUrgencyDashboard).mockResolvedValue(dashboard({ observation: { stays: 12, over_24h: "lt5", median_hours: 9, p90_hours: 22, bed_hours: 140 } }));
  mountPanel();
  const block = await screen.findByRole("region", { name: "Observação" });
  expect(within(block).getByText("12")).not.toBeNull();
  expect(within(block).getByText("menos de 5")).not.toBeNull();
  expect(within(block).getByText("9 h")).not.toBeNull();
  expect(within(block).getByText("140 h")).not.toBeNull();
});
it("sem o bloco (api do 32a), o painel segue igual", async () => {});
```

Na linha do tempo: a entrada `urgency` com `observation_hours: 18` mostra "Observação: 18 h" abaixo do resumo; com `null` ou sem o campo, nada; outro `kind` com o campo, nada.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/UrgencyPanel.test.tsx src/modules/timeline src/lib/timeline.urgency.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `UrgencyPanel.tsx`: `<section aria-label="Observação">` depois dos `StatTile` de permanência: "Encontros com observação" (`countLabel(stays)`), "Mais de 24 h" (`countLabel(over_24h)`), "Permanência na observação · mediana" e "· p90" (`hoursText` curto: número → "N h", `"lt5"` → `SMALL_COUNT`, `null` → "—"), "Leito-hora no mês" (`bed_hours` + " h").
- `PatientTimeline.tsx`: na coluna "Registro", `e.kind === "urgency" && typeof e.observation_hours === "number" && <span style={muted}>{\`Observação: ${e.observation_hours} h\`}</span>`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32b src/modules/UrgencyPanel.test.tsx src/modules/timeline src/lib/timeline.urgency.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/UrgencyPanel.tsx src/modules/UrgencyPanel.test.tsx src/modules/timeline/PatientTimeline.tsx src/lib/timeline.urgency.test.ts
$G commit -m "feat: show observation figures in the urgency panel and observation hours in the timeline" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "dashboard 32b Task 10: bloco `observation` no painel (D12) e `observation_hours` na linha do tempo (D11)".

---

### Task 11: Suíte, build, varredura e revisão

**Files:** nenhum novo (correções entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod32b
$R/bin/test dashboard -w dashboard-mod32b
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **8 arquivos de teste novos**: `src/lib/api.urgencyObservation.test.ts`, `src/lib/urgencyObservation.test.ts`, `src/modules/UrgencyBedsSettings.test.tsx`, `src/modules/urgency/ObservationMap.test.tsx`, `src/modules/urgency/PrescriptionPanel.test.tsx`, `src/modules/urgency/MedicationPanel.test.tsx`, `src/modules/urgency/UrgencyCare.observation.test.tsx`, `src/modules/PharmacyUrgencyShortfalls.test.tsx`. Contagem que dobra = artefato de build descoberto pelo vitest: confira antes de seguir.

- [ ] **Step 2: Varredura de dado sensível, polling e storage**

```bash
cd $W
grep -rnE "refetchInterval" src/modules/urgency src/modules/PharmacyUrgencyShortfalls.tsx src/modules/UrgencyBedsSettings.tsx && echo "ERRO: polling fora do usePolling" || echo ok-polling
grep -rnE "setInterval|setTimeout" src/modules/urgency/ObservationMap.tsx src/modules/urgency/MedicationPanel.tsx src/modules/urgency/AdministrationForm.tsx src/modules/urgency/PrescriptionPanel.tsx && echo "revisar: timer fora do hook" || echo ok-timers
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" src/modules/urgency/ObservationMap.tsx src/modules/urgency/Prescription*.tsx src/modules/urgency/MedicationPanel.tsx src/modules/urgency/AdministrationForm.tsx src/modules/urgency/ObservationSummary.tsx src/modules/urgency/BedsAdmin.tsx src/modules/PharmacyUrgencyShortfalls.tsx src/lib/urgencyObservation.ts && echo "ERRO: storage/log" || echo ok-storage
sed -n '/Urgência — observação, prescrição e checagem/,/^\/\/ ─── [^U]/p' src/lib/api.ts | grep -nE "urgencyQuery\(" | grep -vE "health_unit_id" && echo "revisar: URL" || echo ok-url
grep -rnE "\.\.\.(bed|stay|item|row|r|s|w)\b" src/modules/urgency/ObservationMap.tsx src/modules/PharmacyUrgencyShortfalls.tsx && echo "revisar: resposta espalhada" || echo ok-spread
grep -rnE "textOverflow|ellipsis" src/modules/urgency src/modules/PharmacyUrgencyShortfalls.tsx src/modules/UrgencyBedsSettings.tsx && echo "ERRO: ellipsis" || echo ok-ellipsis
```

Expected: `ok-polling`, `ok-timers`, `ok-storage`, `ok-url`, `ok-spread`, `ok-ellipsis`.

- [ ] **Step 3: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, no mapa (polling, campo a campo, recusas que relêem), na checagem (janela, confirmação, estoque só com `stock_deduction`), na prescrição (P1, versões, primeira dose mantida), nas faltas (sem paciente) e na regressão do 32a (`UrgencyDesk`, `UrgencyQueue`, `UrgencyCare`, `OutcomeForm` iguais sem observação e sem prescrição) e do 31 (`CapsBeds` e `BedsAdmin` intocados).

- [ ] **Step 4: Board**

F-32.7 → **Done** se o api já terminou o dele (o dashboard é o último app desse card); F-32.8 espera o maintenance. **Verified** só depois da prova da Task 12, com o usuário.

**Relato ao coordenador:** "dashboard 32b Task 11: suíte <N> verde (base + 8), build ok, varreduras ok; revisão: <achados>; F-32.7 <Done/aguardando api>".

---

### Task 12: Roteiro e prova no navegador (api 3055, Vite 5215; 1280 px e 375 px)

**Files:** nenhum novo (achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai ao usuário e não vira tarefa).

**Pré-condições:** o api do 32b no worktree `apps/api/.claude/mod32b`, servindo na **3055** dentro da rede do compose (`http://api:3055`), com as migrações da cidade a partir de `20261110900001` aplicadas e a semente do 32b — **migrar e semear o banco de dev só com autorização do usuário**; o serviço do maintenance do compose parado só se a prova do maintenance correr junto (contrato, valores fixos). Interruptores na cidade de prova (Curitiba): `urgency_care`, `clinical_record`, `waiting_room_display`, `pharmacy` e, a partir do passo 9, `urgency_stock_deduction` (ligado pelo maintenance, não pelo dashboard).

- [ ] **Step 1: Suba o Vite do worktree na rede do compose**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod32b --network rota-saude_default -p 5215:5215 \
  -v $R/.worktrees/dashboard-mod32b:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3055 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5215 --host 0.0.0.0"
docker logs -f dashboard-mod32b   # espere o "ready"; Ctrl+C sai do log, o container segue
```

Abra `http://curitiba.localhost:5215/dashboard/` no painel do navegador (`preview_start` com a URL). Contas da semente de dev (admin, recepção, enfermeira 2235, técnico 3222, médico 225, farmacêutico do local `unit` da UPA): as do `db:seed` do api; não repita senha no relato.

- [ ] **Step 2: Percorra, primeiro em 1280 px** (`resize_window` 1280×800)

1. **Admin — Leitos de observação** (Urgência → Leitos de observação, step-up): na UPA, "Obs 1", "Obs 2", "Obs 3"; repetir "Obs 1" (`label_taken`); desativar "Obs 3". Cadastro de unidade: desativar a UPA com leito ativo → `upa_has_beds`.
2. **Recepção**: chegada de dois pacientes; a enfermeira classifica; o médico chama o primeiro e inicia a consulta de urgência.
3. **Mapa — entrada**: o médico, no mapa da mesa, "Colocar em observação no Obs 1" com o paciente em atendimento; o cartão mostra senha, nome, "0 h em observação"; a fila mostra "Obs 1 · 0 h em observação" em "Em atendimento". Em outra aba (enfermeira), tentar o mesmo leito para o segundo paciente → `bed_taken` e o mapa relido.
4. **Troca de leito**: "Trocar de leito" do Obs 1 para o Obs 2; o Obs 1 fica livre; as horas não zeram (contrato §4).
5. **Alertas**: um encontro da semente com entrada antiga (≥ 20 h e ≥ 24 h) mostra "perto de 24 h em observação" e "mais de 24 h em observação" no cartão e na fila, sempre com texto; nada é bloqueado.
6. **Prescrição** (médico, no atendimento): um item `interval` de 6 h (dipirona IV), um `if_needed` com máximo 3/dia e um **controlado** (lista A ou B) → aceito com a marca "controlado · lista X" (P1). Alterar a dose de um item → "Versão 2 ativa; a versão 1 ficou substituída", a primeira dose mantida.
7. **Checagem — enfermeira** (pela fila, "Medicação"): o horário de agora → "Administrado" com confirmação; o horário seguinte "a seguir" sem botão (D8); um "Recusado" com motivo; o duplo clique em "Confirmar" grava uma vez; em outra aba, checar o mesmo horário → `slot_taken` e releitura.
8. **Checagem — técnico 3222** (pelo mapa, "Medicação"): o aprazamento abre sem o detalhe do encontro; checa o `if_needed`; o contador "ainda cabem N" desce. Se o api manteve a fila fechada ao técnico (D2), anote o que a mesa mostra.
9. **Estoque ligado** (`urgency_stock_deduction` ligado pelo maintenance; recarregue a mesa): o lote vem pré-escolhido pelo que vence primeiro e a quantidade em "ampola"; checar → em Farmácia → Posição e extrato, o movimento "administração na UPA" no local `unit` da UPA. Zerar o saldo (ajuste na farmácia) e checar de novo → "Checagem registrada. O estoque ficou com pendência para a farmácia (sem saldo suficiente)."; o extrato não fica negativo.
10. **Faltas da UPA** (farmacêutico do local `unit`): a falta aparece em "Abertas" sem nome de paciente (senha e hora); conciliar com nota → some e aparece em "Conciliadas"; Alertas da farmácia mostram "Faltas da UPA: N". Desligar `urgency_stock_deduction`: a checagem não mostra lote nem quantidade; as faltas antigas continuam visíveis.
11. **Suspensão e saída**: suspender a prescrição com motivo; "Encerrar observação" de um paciente (volta ao atendimento, sem leito); para o outro, finalizar a consulta, "Registrar evolução (adendo)" e o desfecho "Alta" com o aviso "O desfecho encerra a observação no Obs 2 e a prescrição ativa." → o leito fica livre no mapa em até 30 s (ou na hora, pela releitura da mesa).
12. **Painel da urgência** (admin): bloco "Observação" com "menos de 5" onde couber. **Linha do tempo** do paciente: a entrada "Urgência" com "Observação: N h".
13. **Teclado**: passos 3, 6 e 7 só com Tab/Enter/Espaço/Escape (foco visível, Escape fecha o painel de medicação fora dos campos, o foco volta ao botão que abriu).

- [ ] **Step 3: Repita em 375 px** (`resize_window` preset `mobile`, 375×812, e recarregue)

Passos 3, 4, 6, 7, 9 e 10: o mapa vira uma coluna de cartões sem rolagem horizontal da página; as linhas da fila e das faltas viram cartões (`card: "title"`/`"actions"`); o formulário de prescrição empilha os campos de cada item; os botões têm alvo de toque confortável e nenhum texto some cortado (quebra de linha, nunca reticências). Ao terminar, `resize_window` preset `desktop`.

- [ ] **Step 4: Registre e encerre**

Anote cada passo com o que viu (print das telas-chave em 1280 e 375). Achado que muda comportamento → teste na task dona e correção; achado só visual → lista para o usuário. Pare o container (`docker stop dashboard-mod32b`). Cards: F-32.7 → **Verified** com o usuário; F-32.8 → Verified depois do maintenance. Merge, push e remoção do worktree (`$G -C $R/apps/dashboard worktree remove $W`) só com autorização.

**Relato ao coordenador:** "dashboard 32b Task 12: prova em 1280/375 com api 3055 e Vite 5215 — passos 1–13 <ok/achados>; F-32.7 <Verified/pendente>".

---

## Divergências propostas ao contrato

Cada uma diz o problema, a evidência no código real e a proposta. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D1 — Onde fica o cadastro de leitos (decisão do plano, contrato §11).** "Senhas, locais e TVs" é portada **só** por `waiting_room_display` (`src/shell/modules.ts:461`, `src/modules/WaitingRoomSettings.tsx:24`, e o teste `src/shell/modules.urgency.test.ts:15-18` fixa que ela aparece sem `urgency_care`). Leitos de observação dependem de `urgency_care`, não da TV: uma cidade com urgência e sem TV ficaria sem cadastro de leitos, e renomear a tela para "Configuração da UPA" a faria sumir com a TV desligada. **Escolha:** tela irmã **"Leitos de observação"** (`urgency-beds`) no grupo "Urgência", portada por `urgency_care` (com `clinical_record`) e `municipal_admin`; "Senhas, locais e TVs" fica como está. Registrar na §12.
- **D2 — Técnico 3222 sem leitura.** O contrato §1 põe o técnico no mapa e na checagem, mas `Urgency::Access.clinical_link` só aceita 2235\* e 225\* (`ro-api/app/services/urgency/access.rb:33-36`, `catalog.rb:33-34`), e a fila recusa o técnico com `cbo_not_allowed` (`ro-api/app/controllers/urgency/queue_controller.rb:15-18`). Sem a fila, o técnico só alcança quem está num leito; o paciente medicado fora da observação (cadeira de medicação) fica sem checagem dele. **Proposta:** o api aceita 3222\* em `GET /urgency/observation`, `GET …/medication_schedule`, `POST /urgency/administrations` **e** `GET /urgency/queue` (só leitura, sem texto clínico); o detalhe do encontro continua fechado a ele (Q19). Sem a fila, a mesa do técnico mostra a recusa no lugar da fila e o mapa segue (Task 5).
- **D3 — `waiting_bed: []` sem forma.** O §4 escreve `waiting_bed: []` sem dizer o que é. A entrada pelo mapa precisa saber **quem** pode entrar num leito livre. **Proposta:** `waiting_bed[]` = encontros da UPA em `in_care` sem estadia aberta, `{encounter_id, attendance_id, ticket, patient: <patient-ref>, called_at}`, na ordem da chamada; sem texto clínico, sem trilha.
- **D4 — Formas dos três blocos do detalhe e das respostas de observação.** O §2 nomeia `observation`, `prescription` e `administrations` sem chaves. **Proposta:** `observation: {open: Stay | null, stays: Stay[], hours, alert}` com `Stay = {stay_id, bed: {id, label}, started_at, ended_at, end_kind, started_by: {name}, ended_by: {name} | null}`; `prescription` = a versão ativa (ou a última, se suspensa/encerrada) `{id, encounter_id, version, status, supersedes_id, prescribed_at, prescribed_by: {name}, suspension_reason, status_changed_at, items[]}`, com o item `{id, position, catalog_item_id, label, catmat_code, controlled_list, dose, dose_unit, route, frequency: {kind, interval_hours?, max_per_day?}, first_dose_at, instructions, stock_quantity_per_dose, dispensing_unit}`; `administrations[]` como em D9. `POST …/observation` e `…/observation/end` devolvem o `Stay`. Decimais (`dose`, `stock_quantity_per_dose`, `quantity`, `balance`) como **texto**, como o 27a (`fmtQuantity`).
- **D5 — Forma do aprazamento, interruptor utilizável e lotes.** O §5 diz só `slots` e, no `if_needed`, as administrações de 24 h e "quantas cabem". A sessão traz a chave ligada, não utilizável (`src/lib/features.ts:1-4`), e a enfermagem não lê lotes da farmácia (rotas `/pharmacy/*` exigem papel de farmácia por local, `src/modules/pharmacy/LocationPick.tsx:17-27`). **Proposta:** `GET …/medication_schedule` → `{encounter_id, prescription: {id, version, status} | null, stock_deduction: bool (utilizável agora), items: [{item, slots: [{scheduled_at, administration | null}], if_needed: {last_24h[], remaining} | null, last_administration | null, lots?: [{stock_lot_id, lot_number, expires_on, balance}]}]}`, com `lots` só quando `stock_deduction` (na ordem FEFO do §6, só lotes válidos e com saldo do local `unit` da UPA). Sem `lots`, a tela não oferece escolha (o api escolhe por FEFO).
- **D6 — Busca de medicamento e unidade de dispensação.** A busca do catálogo para o profissional é `POST /attendance/medications/search`, que exige `clinical_documents` para quem não é admin (`ro-api/app/controllers/medication_searches_controller.rb:14-16,28-32`); o 32b vive só sob `urgency_care` (contrato §1). Numa cidade sem `clinical_documents`, o médico da UPA não consegue prescrever. E o resultado (`MedicationSearchItem`, `src/lib/api.ts:2150` + `CatalogItemRef` 2065) não traz `dispensing_unit`, que o §1 usa para `stock_quantity_per_dose`. **Proposta:** a busca aceita também o médico 225\* vinculado a uma UPA com `urgency_care` (ou rota irmã `POST /urgency/medications/search`, mesmo corpo e forma), e o item ganha `dispensing_unit`. Escondido/inativo continua fora (o `catalog_item_invalid` do §5 cobre a corrida).
- **D7 — Limites de texto e número.** O contrato não fixa: `dose_unit` (proposta: texto 1–20, sem lista fechada no 32b), `instructions` (≤ 500), `max_per_day` (1–24 inteiro), `first_dose_at` (proposta: de 24 h atrás até 24 h à frente, `invalid_request` em `items.<n>.first_dose_at`), `reason` da checagem e `suspension_reason` (10–500), `reconciliation_note` (10–500), `stock_quantity` (> 0, até 3 casas). O dashboard usa esses valores e recusa em vez de cortar.
- **D8 — Checar horário futuro.** A checagem é imutável e sem correção no 32b (§6, §11), mas o §6 aceita qualquer `scheduled_at` do aprazamento (até 24 h à frente): um clique na linha errada registra a dose da madrugada como dada. **Proposta:** o api recusa horário mais de 60 min à frente com 422 `slot_not_due`; o dashboard já só oferece "Checar" até 60 min à frente (`CHECK_AHEAD_MINUTES`).
- **D9 — Resposta da checagem.** O §6 não diz o que volta. A enfermagem precisa saber, na hora, se houve baixa ou falta (a falta não é erro). **Proposta:** `201 {administration: {id, prescription_item_id, scheduled_at, status, reason, administered_at, recorded_at, recorded_by: {name}, stock: {lot_number, expires_on, quantity, dispensing_unit} | null, shortfall: {cause} | null}, shortfall: {cause} | null}`; os mesmos campos nos `administrations[]` do detalhe e nos `slots` do aprazamento.
- **D10 — Evolução por outro médico de plantão.** O §4 faz da evolução um adendo da consulta `urgency`, mas o adendo é só da autora (`ro-api/app/controllers/consultations_controller.rb:115-118`; no dashboard, `isAuthor` em `src/modules/consultation/ConsultationView.tsx:65`), e o prontuário do atendimento só abre para quem chamou (`not_caller`, `src/lib/urgency.ts:175`). Uma observação de até 24 h atravessa plantões: o médico seguinte não evolui, não prescreve sobre o prontuário aberto e não registra o desfecho pela consulta dele. **Proposta:** decisão do coordenador entre (a) o api aceitar adendo e leitura do prontuário de consulta `urgency` por qualquer 225\* vinculado à UPA enquanto o encontro estiver aberto, com trilha; ou (b) aceitar a limitação no 32b e abrir pendência de ciclo. O plano implementa o atalho só sobre o adendo que o 19a já mostra (a quem ele já mostra).
- **D11 — `observation_hours` na linha do tempo.** `TimelineEntry` não tem campo por tipo (`src/lib/api.ts:2397-2408`); a tela mostra `title` e `summary` prontos. **Proposta:** campo opcional `observation_hours` na entrada `urgency` (o dashboard mostra "Observação: N h"), e o api **não** repete isso no `summary`. Se o api preferir o `summary`, a Task 10 não muda a linha do tempo.
- **D12 — Forma do bloco `observation` do painel.** O §9 nomeia os campos sem tipo. **Proposta:** `{stays: number | "lt5", over_24h: number | "lt5", median_hours: number | "lt5" | null, p90_hours: number | "lt5" | null, bed_hours: number}`, horas inteiras; `bed_hours` é soma, não contagem de pessoas, e não sofre a supressão "lt5" (dizer se o coordenador entender diferente). O bloco é opcional na leitura (api do 32a sem ele).
- **D13 — Códigos sem nome no contrato.** Suspender versão que não está `active` (proposta: 422 `prescription_not_active`, o mesmo da checagem); mais de 20 itens ou nenhum (`invalid_request`, `path: "items"`); `interval_hours`/`max_per_day` fora do campo do tipo (`invalid_request`, `path: "items.<n>.frequency.<campo>"`); leito de outra UPA no PATCH (`not_found`); `problems[]` com `path` no formato `items.<n>.<campo>` (o dashboard aceita também `items[<n>].<campo>`).
- **D14 — Técnico e encontro fechado na checagem.** O §6 dá 409 `encounter_closed` com o encontro fechado; com a estadia encerrada sem desfecho (`observation_ended`), a prescrição continua ativa e a checagem segue. O plano assume isso (o painel de medicação abre pela fila também). Se o api quiser encerrar a prescrição ao sair da observação, dizer.
- **D15 — Rótulo do movimento e do documento no extrato.** O extrato do 27a traduz `kind` e `document.type` por tabela fechada (`src/lib/pharmacy.ts:77-91`). **Proposta:** o movimento sai com `kind: "administration"` e `document: {type: "urgency_administration", number: null}`; a tela mostra "administração na UPA" nos dois, nunca o paciente. (Os `kind` do 29 — `vial_opening`, `immunization`, `immunization_reversal`, `ro-api/app/services/pharmacy/catalog.rb:11-14` — também não têm rótulo na tabela do dashboard; fica para o card do 29, fora deste plano.)
- **D16 — Forma da lista de faltas.** O §7 dá os campos de topo. **Proposta:** `GET /pharmacy/urgency_shortfalls?stock_location_id&state=open|reconciled` → `{items: [{shortfall_id, created_at, catalog_item: {id, label, dispensing_unit}, quantity, cause, administration: {administered_at, ticket}, reconciled_at, reconciled_by: {name} | null}]}`, as abertas da mais antiga para a mais nova, as conciliadas da mais nova para a mais antiga, até 200 por página (sem paginação no 32b); `POST …/reconcile` devolve o item. Com `no_location`, `stock_location_id` é nulo e a falta não aparece em local nenhum: **proposta** — o api a lista para o filtro do local `unit` da UPA quando ele passar a existir, ou o coordenador decide outra vitrine (hoje ela ficaria invisível).
- **D17 — Faltas no menu sem urgência.** O §7 exige só `pharmacy`; com isso, "Faltas da UPA" aparece também em cidade sem UPA ou com `urgency_care` desligado (a lista fica vazia). O plano segue o contrato (as faltas antigas precisam continuar visíveis depois de desligar a urgência); registrar como aceito.
- **D18 — Alterar a prescrição sem dar a mesma dose duas vezes.** O §5 cria versão nova a cada alteração e o índice único de checagem é (`urgency_prescription_item_id`, `scheduled_at`) (§1): o horário das 14 h checado na versão 1 aparece livre na versão 2, e a enfermagem pode dar a dose de novo. **Proposta:** (a) o dashboard reenvia o `first_dose_at` original dos itens mantidos (a grade de horários não muda); (b) o api trata como `slot_taken` o mesmo `catalog_item_id` no mesmo `scheduled_at` dentro do encontro, em qualquer versão, e o aprazamento mostra nesses horários a checagem da versão anterior; (c) cada item do aprazamento traz `last_administration` do mesmo `catalog_item_id` no encontro.
- **D19 — Mapa e fila com recusa definitiva.** O §4 não diz o que o mapa faz com 403/404/422. O plano repete o critério da fila do 32a (para nas recusas definitivas e mostra "Tentar de novo"; segue tentando em 401, 408, 425, 429, 5xx e sem rede). Registrar para que o api devolva 403 `missing_link`/`cbo_not_allowed` (e não 404) a quem perde o vínculo.

## Self-review

1. **Cobertura (32b):** decisão 2 e 3 (entrega 32b sobre o 32a) → todas as tasks; decisão 20 (leitos, exclusion, alertas, reavaliação no encontro, desfecho ao sair) → Tasks 4, 5 e 8; decisão 21 (prescrição do catálogo, versões, checagem imutável) → Tasks 6 e 7; decisão 22 e R7 (estoque, FEFO, falta, nunca negativo) → Tasks 7 e 9 (a regra é do api; a tela mostra lote, quantidade e a pendência); decisão 23 (interruptor) → Tasks 1 e 7; §9 painel e linha do tempo → Task 10; §11 telas do dashboard → Tasks 3–9; P1 → Task 6; P2 (sem BPA da administração) → nada na tela; F-32.7 → Tasks 4, 5, 8, 10; F-32.8 → Tasks 6, 7, 9.
2. **Passos:** cada passo de código diz arquivo, assinatura e os valores do contrato e das Divergências; corpos completos só nas fixtures, nos testes e nas frases fixas; os rótulos de componentes existentes que os testes usam (`MedicationSearch`, `ConfirmButton`, `SensitiveAction`, `LocationPick`, `ConsultationView`) são conferidos na Task 0 contra o `979e881`.
3. **Tipos:** `UrgencyObservationMap`, `ObservationBed`, `UrgencyStay`, `UrgencyPrescription`, `UrgencyPrescriptionItem`, `UrgencyMedicationSchedule`, `ScheduleItem`, `UrgencyAdministrationResult`, `UrgencyShortfall` e as funções da Task 1 são os mesmos nas Tasks 2–10; `MedicationTarget` (Task 5) é usado pela mesa, pelo mapa e pela fila (Task 7); `UrgencyRoles.technician/observation/medication/mapReader` (Task 2) servem às Tasks 5 e 7; `usePolling` só no mapa (Task 5).
4. **Review Focus:** as cinco linhas têm teste na task dona (5; 7; 7; 6; 5 e 9).
5. **Proporção e simplicidade:** nenhum componente genérico novo; o mapa e o cadastro copiam os moldes do 31; um tradutor de erro só (`URGENCY_ERRORS`); nenhum storage novo; um polling novo (o mapa), pelo hook do 32a.
