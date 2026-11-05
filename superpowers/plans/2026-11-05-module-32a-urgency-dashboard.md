# Módulo 32a — Urgência (UPA) (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 32a** (`feat/mod-32a-urgency`, worktree `apps/api/.claude/mod32a`, porta de dev **3054**) com as rotas do contrato §3–§11 (e as Divergências D1–D22 deste plano que o plano do api adotar), os interruptores `urgency_care` e `waiting_room_display` na sessão e a semente do 32a; o **contracts** com a tag `protocols-v1.7.0`. Ordem de entrega do contrato §13: contracts → api → dashboard → wpda → maintenance. As Tasks 0–13 não dependem do api (dublês de `src/lib/api` com fixtures nas formas do contrato §2 e das Divergências); a prova da Task 14 depende.

**Goal:** Lado dashboard do 32a: o protocolo `urgency_classification` (5 cores e tempos máximos) no editor de protocolos assinados do 18; numa unidade `upa` com `urgency_care`, o Atendimento troca a recepção, a escuta e a fila do 13 pela **mesa da UPA** — recepção (busca por POST, paciente novo, não identificado com descrição, senha exibida para entregar), **fila por cor** com tempo e estouro e a faixa "aguardando classificação" (polling com `clearTimeout`), **classificação** (sinais vitais do 18, cor sugerida com o critério, justificativa, reclassificação), **chamada** para classificação e para atendimento com local de chamada (lembrado na sessão) e "chamar de novo", **consulta do 19a `kind: "urgency"`** aberta da fila, **desfecho** com os 6 tipos e contrarreferência, e **identificar paciente** com confirmação forte da fusão; na UBS, a senha no check-in, o local na chamada do 13 e a lista de retornos pedidos pela urgência; o grupo **"Urgência"** com **Senhas, locais e TVs** (código de pareamento, revogar) e o **Painel da urgência** com `lt5`; a **TV** em `/#/tv`, tela cheia fora do shell (pareamento, polling de 5 s, legível a distância, sem dado pessoal); o kind `urgency` na linha do tempo.

**Architecture:** O cliente HTTP ganha a seção "Urgência (módulo 32a)" no fim de `src/lib/api.ts` (tipos = formas do contrato §2–§11 e das Divergências); `callAttendance`/`callNext` ganham o local, `startConsultation` aceita `kind: "urgency"` e `finalizeConsultation` aceita desfecho nulo. As regras de tela ficam puras em `src/lib/urgency.ts` (portões, papéis pelo CBO do vínculo, cores, validações, frases de **todos** os códigos de erro do contrato) e o polling num hook só, `src/lib/usePolling.ts` (cadeia de `setTimeout`, `clearTimeout` ao desmontar, resposta velha descartada). As peças ficam em `src/modules/urgency/`; a mesa entra pelo `Attendance.tsx` (mesma escolha de unidade), a TV entra pelo `main.tsx` **antes** do `AuthProvider` (nenhuma chamada de sessão, nenhum cookie). O dashboard nunca decide o que o api garante: ordem da fila, estouro, cor sugerida, CBO, senha única, local obrigatório, fusão, BPA, trilha e imutabilidade são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (só o 32a: §1 intenção e antiobjetivos; §2 decisões 1, 4–19, 23 e as do coordenador R1–R6; §4 telas; §5 segurança e LGPD; §6 testes; §8 F-32.1..F-32.6), `docs/adr/0050.md`, a pesquisa `docs/pesquisa/2026-11-05-urgencia-upa-32.md` e o contrato `docs/superpowers/plans/2026-11-05-module-32a-urgency-contracts.md` (§1–§13). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D22); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptores `urgency_care` (requer `clinical_record`) e `waiting_room_display` (independente; UPA e UBS); unidade `health_units.kind = "upa"`, `ticket_prefix` (1 a 3 letras maiúsculas, padrão `"U"` na UPA e `"A"` nas demais), `show_wait_times`; estado do encontro `waiting_classification` | `classified` | `in_care` | `closed`; cores `red`, `orange`, `yellow`, `green`, `blue` **nessa ordem**, tempos padrão 0, 10, 60, 120 e 240 min; desfecho `discharged` | `discharged_to_primary_care` | `transferred` | `left_without_care` | `left_against_advice` | `death`; chamada `classification` | `care`; consulta `kind: "urgency"`. **Código desconhecido aparece cru, nunca some.**
- **Interruptores:** sem `urgency_care` **ou** sem `clinical_record` na sessão, a mesa da UPA, o modelo de protocolo de urgência, o Painel da urgência, a lista de retornos e o kind na linha do tempo somem, e a UPA segue o 13 de hoje; sem `waiting_room_display`, senha, local de chamada, "Chamar de novo", "Senhas, locais e TVs" e o prefixo no cadastro de unidade somem (R6). 403 `feature_disabled` → `URGENCY_DISABLED` / `WAITING_ROOM_DISABLED`. O dashboard nunca escreve interruptor; as chaves ganham só rótulo em `features.ts`.
- **Quem faz o quê** (contrato §1): recepção = `citizen_verifier`; classificação = `health_professional` com vínculo ativo na UPA e CBO `2235*` ou `3222*`; atendimento médico = vínculo com CBO `225*`; configuração (protocolo pelo editor do 18, locais, TVs, prefixo) = `municipal_admin` com step-up; painel = `municipal_admin` e `analyst`. O papel na UPA sai de `getMyProfessional().links` (vínculo sem `ended_at` naquela unidade); o api confere de novo.
- **Nada clínico em lista** (contrato §6): fila, faixa "aguardando classificação", "em atendimento" e retornos mostram só `<patient-ref>` (nome, CNS mascarado, "não identificado"), senha, horários, minutos, cor e estouro — **nunca** queixa, sinais, justificativa, CID, descrição do não identificado, destino ou orientação, mesmo que o api os mande por engano (a lista lê campo a campo). O detalhe do encontro é lido **só no clique**, com `NOMINAL_QUERY` (`gcTime: 0`), e sai do cache ao fechar ou trocar de unidade.
- **Dado pessoal só no corpo:** CPF, CNS, nome, queixa, CID e a descrição do não identificado nunca na URL, em mensagem de erro, em `console`, em Analytics ou em storage. Na URL vão só ids, `health_unit_id` e `month`. Busca de pessoa por `POST /urgency/person_lookup`.
- **Storage permitido, e só este:** `sessionStorage` guarda o **id** do local de chamada por usuário e unidade (`callLocation.<userId>.<unitId>`), e o `localStorage` **do aparelho da TV** guarda o `display_token` (`waitingRoom.displayToken`) — sem ele a TV pediria o código a cada religar (D17). Todo acesso em `try/catch`. Nada mais.
- **A TV nunca mostra** nome, CPF, CNS, cor da pessoa, queixa ou id interno: só senha, local, tipo da chamada, hora e o tempo de espera por cor; lê campo a campo (D18). A TV não manda cookie (`credentials: "omit"`), só `X-Display-Token`.
- **Polling:** só três telas fazem polling e todas pelo `usePolling` — fila da UPA (`URGENCY_QUEUE_POLL_MS = 10_000`), TV (`WAITING_ROOM_POLL_MS = 5_000`) e nada mais; próximo pedido só depois da resposta do anterior; `clearTimeout` e descarte da resposta ao desmontar ou trocar de unidade. Nenhum `refetchInterval` novo.
- **Todas as recusas de uma vez:** a tela confere o formulário inteiro antes de enviar e mostra cada problema; a recusa do api com `problems[]` mostra todas, traduzidas por `urgencyRefusals`; sem `problems`, a do topo. Todo código do contrato §1–§8 tem frase em `URGENCY_ERRORS` (o teste da Task 2 cobra a lista inteira); código novo do api aparece como "recusado pelo servidor (<código>)".
- **Recusar em vez de cortar** (D16): queixa 3–500; justificativa 10–500; descrição do não identificado 10–500; destino 3–200; pedido de vaga até 2000; motivo do retorno 10–500; prazo do retorno 1–90 dias inteiros; nome do local e da TV 2–40; prefixo `^[A-Z]{1,3}$`; código de pareamento 6 dígitos; dor 0–10 inteiro.
- **Sem duplo envio:** botão de escrita desabilitado enquanto a chamada corre (`busy` + `useRef`), e a resposta de uma ação já trocada (outra unidade, outro encontro) é ignorada (`seq`).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo — `{}` quando não há campo).
- "Agora" é `new Date()`; telas recebem `now` por prop e os testes passam `NOW32 = new Date("2026-11-05T10:00:00-03:00")`; relógio falso (`vi.useFakeTimers`) **só** nos testes de polling (Tasks 2, 6 e 12), com `vi.useRealTimers()` no `afterEach`.
- Interface (ciclo de interface): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--accent`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). A cor de risco **sempre com texto** ("laranja"), nunca só tom (o tema não tem papel laranja; laranja usa `tone: "urgent"`, ver D22). A mesa da UPA funciona em desktop e tablet (1024×768) e só com teclado; a TV em 1920×1080 e 1280×720. Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `ActionMessage`, `formStyles`, `VitalSignsFields`, `ConditionBuilder`, `ConsultationWorkspace`, `PersonStart`, `newPatientProblem`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nada na mesa da UPA nem na chamada aceita CAPS: a unidade da mesa é sempre `kind === "upa"`; `caps_unit_not_allowed` tem frase (P32 do 31).
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-32.1..F-32.6** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-32.1: api, dashboard, maintenance e contracts; F-32.6: api, dashboard e wpda; F-32.2..F-32.5: api e dashboard) e a **Verified** depois da prova no navegador (Task 14).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod32a` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-32a-urgency` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod32a`, `G=/opt/homebrew/bin/git`.
- Testes: `$R/bin/test dashboard -w dashboard-mod32a <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem jest-dom (`not.toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx`; as fixtures do 32a vêm de `src/test/urgencyFixtures.ts` (Task 1).

## Review Focus

1. **A TV recebe por engano um campo pessoal, perde a rede ou tem o token revogado no meio da noite.** Espera-se: a TV mostra só senha, local, tipo e hora (um `name`, `cpf`, `color` ou `complaint` dentro de `calls[]` não aparece no DOM); sem rede, fica a última tela com "sem conexão — tentando de novo" e o polling continua a cada 5 s; 401 `display_revoked` apaga o token do aparelho e volta ao pareamento, sem novo pedido ao board. Teste: Task 12, "board lê campo a campo; sem rede mantém a tela; revogada volta ao pareamento".
2. **Trocar de UPA (ou sair da tela) com um pedido da fila no ar.** Espera-se: nenhum `setTimeout` pendente depois de desmontar; a resposta atrasada da unidade antiga não aparece na nova; nenhum aviso de `setState` em componente desmontado; o próximo pedido só sai depois da resposta do anterior. Teste: Task 2, "usePolling: um pedido por vez, clearTimeout ao desmontar, resposta velha descartada", e Task 6, "trocar de unidade descarta a fila antiga".
3. **Duplo clique em "Registrar chegada" e o mesmo paciente chegando duas vezes.** Espera-se: **uma** chamada a `createUrgencyArrival`; 409 `encounter_open` mostra a frase e não limpa o formulário; o não identificado sem descrição (ou com 9 caracteres) não sai; as três formas nunca vão juntas no corpo. Teste: Task 5, "duplo clique envia uma vez; encounter_open; uma forma só no corpo".
4. **Identificar o não identificado com um paciente que já existe, por engano, ou depois de outra pessoa já ter identificado.** Espera-se: a fusão pede a caixa "Confirmo que é a mesma pessoa…" **e** o segundo passo do `ConfirmButton`, com os dois lados lado a lado; 409 `already_identified` relê o encontro e mostra o paciente real, sem nova tentativa; o paciente novo não pede fusão. Teste: Task 10, "fusão exige caixa e confirmação; already_identified relê".
5. **Classificar sem protocolo ativo, mudar a cor sugerida, ou o api pedir justificativa que a tela não pediu.** Espera-se: sem sugestão, a justificativa é obrigatória (10–500); cor diferente da sugerida também; 422 `justification_required` mostra o campo com a frase e mantém tudo digitado; a reclassificação mostra a cor vigente e o histórico sem apagar nada; o duplo clique em "Classificar" envia uma vez. Teste: Task 7, "sem protocolo pede justificativa; cor trocada; justification_required do api; reclassificação".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.urgency.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/urgencyFixtures.ts` | tipos e cliente HTTP do 32a; local na chamada do 13; `kind: "urgency"`; prefixo na unidade; interruptores; proxy; dados de teste | 1 |
| `src/lib/urgency.ts`, `src/lib/urgency.test.ts`, `src/lib/usePolling.ts`, `src/lib/usePolling.test.tsx` | portões, papéis pelo CBO, cores, validações, frases de todos os códigos; o hook de polling | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.urgency.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/main.tsx`, `src/lib/tvRoute.ts`, `src/lib/tvRoute.test.ts`, `src/modules/UrgencyPanel.tsx`, `src/modules/WaitingRoomSettings.tsx`, `src/modules/WaitingRoomTv.tsx` | grupo "Urgência", telas-portão, rota da TV fora do shell | 3 |
| `src/lib/urgencyRules.ts`, `src/lib/urgencyRules.test.ts`, `src/lib/condition.ts`, `src/modules/protocolEditor/UrgencyLevelsPanel.tsx`, `src/modules/protocolEditor/UrgencyRulesPanel.tsx`, `src/modules/ProtocolEditor.tsx`, `src/modules/protocolEditor/UrgencyPanels.test.tsx` | protocolo `urgency_classification` no editor (F-32.1) | 4 |
| `src/modules/Attendance.tsx`, `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/ArrivalForm.tsx`, `src/modules/urgency/TicketCard.tsx`, `src/modules/urgency/UrgencyReception.test.tsx`, `src/modules/Attendance.urgency.test.tsx` | mesa da UPA no Atendimento; recepção e senha (F-32.2) | 5 |
| `src/modules/urgency/UrgencyQueue.tsx`, `src/modules/urgency/UrgencyQueue.test.tsx` | fila por cor, estouro, "aguardando classificação", "em atendimento" (F-32.3) | 6 |
| `src/modules/urgency/ClassificationForm.tsx`, `src/modules/urgency/EncounterSummary.tsx`, `src/modules/urgency/ClassificationForm.test.tsx` | classificação, cor sugerida, justificativa, reclassificação (F-32.3) | 7 |
| `src/modules/attendance/CallLocationPicker.tsx`, `src/modules/attendance/CallLocationPicker.test.tsx`, `src/modules/urgency/UrgencyQueue.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/CheckIn.tsx`, `src/modules/attendance/UnitQueue.waitingRoom.test.tsx` | local de chamada lembrado, chamada para classificação e atendimento, "Chamar de novo", senha na UBS (F-32.5) | 8 |
| `src/modules/urgency/UrgencyCare.tsx`, `src/modules/urgency/OutcomeForm.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/urgency/UrgencyCare.test.tsx` | consulta `urgency` aberta da fila, desfecho com os 6 tipos e contrarreferência, fechamento do 13 numa UPA (F-32.4) | 9 |
| `src/modules/urgency/IdentifyPatient.tsx`, `src/modules/urgency/IdentifyPatient.test.tsx` | identificar o não identificado, com confirmação forte da fusão (F-32.2) | 10 |
| `src/modules/WaitingRoomSettings.tsx`, `src/modules/waitingRoom/CallLocationsPanel.tsx`, `src/modules/waitingRoom/DisplaysPanel.tsx`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/WaitingRoomSettings.test.tsx`, `src/modules/attendance/UnitForm.ticket.test.tsx` | locais de chamada, TVs (código, revogar), prefixo da senha (F-32.5, F-32.1) | 11 |
| `src/modules/WaitingRoomTv.tsx`, `src/modules/waitingRoom/TvPairing.tsx`, `src/modules/waitingRoom/TvBoard.tsx`, `src/modules/waitingRoom/tv.css`, `src/modules/WaitingRoomTv.test.tsx` | a TV: pareamento, board, polling de 5 s, tela cheia (F-32.5) | 12 |
| `src/modules/UrgencyPanel.tsx`, `src/modules/UrgencyPanel.test.tsx`, `src/lib/timeline.ts`, `src/lib/timeline.urgency.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/urgency/CounterReferrals.tsx`, `src/modules/urgency/CounterReferrals.test.tsx`, `src/modules/Attendance.tsx` | painel do gestor, kind `urgency` na linha do tempo, retornos pedidos pela urgência na UBS (F-32.6, F-32.4) | 13 |
| — | suíte, build, varredura, revisão, prova no navegador | 14 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho, `credentials` e corpo — sobretudo o que **não** pode ir na URL (CPF, CNS, nome, queixa, descrição) e que a TV não manda cookie (Task 1); regras puras com tabela de casos, inclusive a cobertura de **todos** os códigos de erro do contrato e o polling com relógio falso (Task 2); menu e portão por papel e interruptor, e a rota da TV fora do shell (Task 3); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e fixtures do contrato §2, cobrindo o caminho feliz, o teclado, cada recusa que muda a tela, o duplo envio e o que **não** pode aparecer (texto clínico em lista; dado pessoal na TV; mesa da UPA sem o interruptor) (Tasks 4–13); `tsc`, build, a suíte inteira e as varreduras, e a prova no navegador com o api real na 3054, em tablet e na TV em 1920×1080 e 1280×720 (Task 14). O dashboard não testa o que é do api (ordem da fila, estouro, cor sugerida, senha única, fusão, BPA, LEDI, wpda).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`3671781` ou mais novo, com o 31 mergeado); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod32a`.
- Produces: cards F-32.1..F-32.6 em In Progress; worktree `$W` na branch `feat/mod-32a-urgency`; a base anotada; as rotas e as formas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-32\\.[1-6] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os seis cards em In Progress ou além. Se algum F-32.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Atenção psicossocial (módulo 31\|^export async function callAttendance\|^export async function callNext\|^export function startConsultation\|^export function finalizeConsultation\|^export type ConsultationKind\|^export interface UnitCapsInput\|^export interface Attendance {\|^export interface QueueRow {\|^export interface NewPatient {\|^export interface VitalSigns {'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  psychosocial_care: "Atenção psicossocial (CAPS)"\|  | "sanitary_transport" | "tfd" | "psychosocial_care";'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "caps-settings";\|    { id: "caps-settings", label: "CAPS: leitos e regras"\|  const psy = psychosocialRoles(user);'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - '
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '    "caps-settings": {\|    "CAPS": {'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "caps-settings":'
$G -C $R/apps/dashboard show origin/main:src/main.tsx | grep -n '^createRoot('
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"/psychosocial"\|"\^/psychosocial'
$G -C $R/apps/dashboard show origin/main:src/modules/Attendance.tsx | grep -n '{canVerify && unit && <div id="sec-checkin">\|{canCare && unit && <ScreeningQueue\|{isAdmin && <Units '
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitQueue.tsx | grep -n 'const result = await callNext(unit.id);\|const result = await callAttendance(row.id, unit.id);\|^function ClosePanel'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/CheckIn.tsx | grep -n '>Atendimento iniciado<'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n '  onGoToSecurity?(): void;\|const proceduresOn = '
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationEditor.tsx | grep -n 'finalizeConsultation(consultation.id, outcomeBody(outcome, referralUnitId))\|<OutcomeFields idPrefix="consultation-"'
$G -C $R/apps/dashboard show origin/main:src/modules/ProtocolEditor.tsx | grep -n '"__new_screening__"\|const screeningKind = '
$G -C $R/apps/dashboard show origin/main:src/lib/condition.ts | grep -n '^export type ConditionContext\|  if (context === "screening")'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitForm.tsx | grep -n '^export interface UnitFormValue\|  capsOn?: boolean;'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '  { key: "psychosocial_link", label: "CAPS" }\|^export function timelineKinds'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n 'timelineKinds('
$G -C $R/apps/dashboard show origin/main:src/lib/territoryDictionaries.ts | grep -n '^export function newPatientProblem'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod32a -b feat/mod-32a-urgency origin/main
cd $R/.worktrees/dashboard-mod32a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod32a
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Ambulance|DeviceTv|ChartBar)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os três ícones (falta algum → escolha outro que exista e use o mesmo na Task 3). **Anote** a base (arquivos e testes). A Task 14 espera a base mais **21 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira as rotas e formas do api do 32a**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api log --oneline -5 feat/mod-32a-urgency 2>/dev/null || echo "api do 32a ainda sem branch"
$G -C $R/apps/api show feat/mod-32a-urgency:config/routes.rb 2>/dev/null | grep -nE 'urgency|waiting_room|call_locations|recall'
$G -C $R/apps/contracts tag -l 'protocols-v1.7*'
```

Expected: as rotas do contrato §3–§8. Para cada Divergência (D1–D22) anote se o api adotou a forma proposta ou outra; **onde o api diz outra coisa, vale o api** — a task dona ajusta tipo, fixture e teste antes de seguir (registre na descrição do commit). Sem branch do api ainda: siga pelas formas deste plano e repita este passo antes da Task 14.

---

### Task 1: Tipos, cliente HTTP, local na chamada do 13, consulta de urgência, prefixo, interruptores, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; `Attendance`, `QueueRow`, `HealthUnit`, `UnitCapsInput`, `callAttendance`, `callNext`, `ConsultationKind`, `finalizeConsultation`)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/urgencyFixtures.ts`
- Test: `src/lib/api.urgency.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError`, `withQuery`, `ATTENDANCE_BASE`, `VitalSigns`, `NewPatient`, `SessionUser`, `MyProfessional`, `ProfessionalLink`, `TimelineEntry` (existentes em `src/lib/api.ts`); `sessionWith` de `src/test/campaignFixtures.tsx`.
- Produces (todas em `src/lib/api.ts`, exportadas):
  - Tipos: `UrgencyColor = "red" | "orange" | "yellow" | "green" | "blue"`, `UrgencyState`, `UrgencyOutcomeKind`, `CallKind = "classification" | "care"`, `UrgencyCount = number | "lt5"`.
  - Formas (contrato §2 + D): `UrgencyPatientRef { patient_id: string; name: string; cns_masked: string | null; unidentified: boolean; description?: string | null }`; `UrgencyLookup { patient: UrgencyPatientRef | null; candidates: UrgencyPatientRef[] }`; `UrgencyArrivalInput = { health_unit_id: string } & ({ patient_id: string } | { new_patient: NewPatient } | { unidentified: { description: string } })` (D1); `UrgencyArrival { attendance_id; encounter_id; ticket: string | null; patient: UrgencyPatientRef }`; `UrgencyLastCall { kind: CallKind; location: string | null; called_at: string }`; `UrgencyWaitingRow { encounter_id; attendance_id; ticket: string | null; patient: UrgencyPatientRef; arrived_at; waited_minutes: number; last_call: UrgencyLastCall | null }` (D2); `UrgencyColorRow extends UrgencyWaitingRow { color: UrgencyColor; overdue: boolean; max_wait_minutes: number }`; `UrgencyInCareRow { encounter_id; attendance_id; ticket; patient; color: UrgencyColor | null; consultation_id: string | null; consultation_status: "draft" | "finalized" | null; called_by_name: string | null; called_at: string }` (D2); `UrgencyQueue { awaiting_classification: UrgencyWaitingRow[]; by_color: UrgencyColorRow[]; in_care: UrgencyInCareRow[] }`; `UrgencyLevel { color: UrgencyColor; max_wait_minutes: number }`; `UrgencySuggestInput { encounter_id: string; complaint: string; vital_signs: VitalSigns; pain_scale: number | null }`; `UrgencySuggestion { suggested: { color: UrgencyColor; criterion: string; rule_id: string } | null; levels: UrgencyLevel[] }` (D3); `UrgencyClassificationInput { complaint; vital_signs; pain_scale; color: UrgencyColor; justification?: string }`; `UrgencyClassificationView { id; color; suggested: { color; criterion } | null; complaint; vital_signs; pain_scale: number | null; justification: string | null; classified_at; by: { name: string } }`; `UrgencyOutcomeInput { kind: UrgencyOutcomeKind; destination?: string; vacancy_request?: string; return_guidance?: { days: number; reason: string } }`; `UrgencyOutcomeView extends UrgencyOutcomeInput { recorded_at; by: { name } }`; `UrgencyEncounter { id; attendance_id; health_unit_id; state: UrgencyState; ticket; arrived_at; patient: UrgencyPatientRef; classifications: UrgencyClassificationView[]; consultation_id: string | null; consultation_status: "draft" | "finalized" | null; outcome: UrgencyOutcomeView | null }` (D4); `IdentifyInput = { patient_id: string } | { new_patient: NewPatient }`; `CallLocation { id; health_unit_id; name; active: boolean }`; `CallLocationList { items: CallLocation[]; location_required: boolean }` (D6); `WaitingRoomDisplay { id; health_unit_id; name; active; show_wait_times: boolean; paired: boolean; last_seen_at: string | null }` (D7); `PairingCode { code: string; expires_at: string }`; `BoardCall { ticket: string; location: string; kind: CallKind; called_at: string }`; `BoardWaitTime { color: UrgencyColor; median_minutes: number | "lt5_patients" }` (D18); `WaitingRoomBoard { unit: { name: string }; calls: BoardCall[]; wait_times: BoardWaitTime[] | null; updated_at: string }`; `UrgencyTimes { median: number; p90: number } | "lt5"`; `UrgencyDashboard` (D9); `CounterReferral { encounter_id; patient: UrgencyPatientRef; upa: { name: string }; closed_at; return_days: number; due_on: string }` (D10).
  - Funções: `lookupUrgencyPerson(q: { cpf: string } | { cns: string }): Promise<UrgencyLookup>`, `createUrgencyArrival(i: UrgencyArrivalInput): Promise<UrgencyArrival>`, `getUrgencyQueue(unitId): Promise<UrgencyQueue>`, `getUrgencyEncounter(id): Promise<UrgencyEncounter>`, `suggestUrgencyColor(i: UrgencySuggestInput): Promise<UrgencySuggestion>`, `classifyUrgency(encounterId, i: UrgencyClassificationInput): Promise<{ encounter: UrgencyEncounter; suggested: UrgencySuggestion["suggested"] }>`, `callForClassification(encounterId, callLocationId: string | null): Promise<UrgencyEncounter>`, `recallAttendance(attendanceId, callLocationId: string | null): Promise<{ call_id: string }>`, `recordUrgencyOutcome(encounterId, i: UrgencyOutcomeInput): Promise<UrgencyEncounter>`, `identifyUrgencyPatient(encounterId, i: IdentifyInput): Promise<UrgencyEncounter>`, `listCallLocations(unitId): Promise<CallLocationList>`, `createCallLocation(i: { health_unit_id; name; active })`, `updateCallLocation(id, i: { name; active })`, `listWaitingRoomDisplays(unitId): Promise<WaitingRoomDisplay[]>`, `createWaitingRoomDisplay(i: { health_unit_id; name; active; show_wait_times })`, `updateWaitingRoomDisplay(id, i: { name; active; show_wait_times })`, `createPairingCode(displayId): Promise<PairingCode>`, `revokeWaitingRoomDisplay(displayId): Promise<WaitingRoomDisplay>`, `pairWaitingRoomDisplay(code): Promise<{ display_token: string }>`, `fetchWaitingRoomBoard(token): Promise<WaitingRoomBoard>`, `getUrgencyDashboard(q: { month: string; unitId?: string | null }): Promise<UrgencyDashboard>`, `listCounterReferrals(unitId): Promise<CounterReferral[]>`.
  - Mudanças no 13/19a (contrato §1, §6, §7): `Attendance` e `QueueRow` ganham `ticket?: string | null`; `callAttendance(id, healthUnitId, callLocationId?: string | null)` e `callNext(unitId, callLocationId?: string | null)` mandam `call_location_id` só quando recebido; `ConsultationKind = "general" | "dental" | "urgency"`; `finalizeConsultation(id, outcome: OutcomeBody | null)` manda `{}` com `null` (D5); `HealthUnit` ganha `ticket_prefix?: string | null` e `show_wait_times?: boolean`; `UnitCapsInput` ganha `ticket_prefix?: string | null` (só vai no corpo quando recebido, como os campos do 31).
  - `src/test/urgencyFixtures.ts`: `NOW32`, `MONTH32 = "2026-11"`, `UPA`, `UBS`, `NURSE_LINK`, `DOCTOR_LINK`, `myProfessional(links)`, `patientRef(o?)`, `waitingRow(o?)`, `colorRow(o?)`, `inCareRow(o?)`, `queue(o?)`, `encounter(o?)`, `classification(o?)`, `suggestion(o?)`, `board(o?)`, `urgencyUser(roles, features?)`.

- [ ] **Step 1: Escreva as fixtures** `src/test/urgencyFixtures.ts` (o único arquivo com corpo completo nesta task)

```ts
// Dados dos testes do módulo 32a (urgência). Agora: quinta, 2026-11-05, 10h. Formas do contrato §2 e das Divergências do plano.
import type {
  HealthUnit, MyProfessional, ProfessionalLink, SessionUser, UrgencyClassificationView, UrgencyColorRow, UrgencyEncounter,
  UrgencyInCareRow, UrgencyPatientRef, UrgencyQueue, UrgencySuggestion, UrgencyWaitingRow, WaitingRoomBoard
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW32 = new Date("2026-11-05T10:00:00-03:00");
export const MONTH32 = "2026-11";
export const UPA: HealthUnit = { id: "upa1", name: "UPA Boqueirão", kind: "upa", ticket_prefix: "U", show_wait_times: true };
export const UBS: HealthUnit = { id: "ubs1", name: "UBS Centro", kind: "ubs", ticket_prefix: "A", show_wait_times: false };

const link = (o: Partial<ProfessionalLink>): ProfessionalLink => ({
  id: "l1", health_unit_id: "upa1", unit_name: "UPA Boqueirão", cbo_code: "223505", cbo_title: "Enfermeiro",
  started_at: "2026-01-01", started_by: "admin", ended_at: null, ...o
} as ProfessionalLink);
export const NURSE_LINK = link({});
export const DOCTOR_LINK = link({ id: "l2", cbo_code: "225125", cbo_title: "Médico clínico" });
export const myProfessional = (links: ProfessionalLink[]): MyProfessional =>
  ({ professional: { id: "p1", user_id: "u-1", professional_name: "Ana Souza" }, links, shifts: [] } as unknown as MyProfessional);

export const patientRef = (o: Partial<UrgencyPatientRef> = {}): UrgencyPatientRef =>
  ({ patient_id: "pa1", name: "Maria Lima", cns_masked: "*** **** **** 1234", unidentified: false, ...o });
export const waitingRow = (o: Partial<UrgencyWaitingRow> = {}): UrgencyWaitingRow => ({
  encounter_id: "en1", attendance_id: "at1", ticket: "U-042", patient: patientRef(), arrived_at: "2026-11-05T09:48:00-03:00",
  waited_minutes: 12, last_call: null, ...o
});
export const colorRow = (o: Partial<UrgencyColorRow> = {}): UrgencyColorRow =>
  ({ ...waitingRow({ encounter_id: "en2", attendance_id: "at2", ticket: "U-040" }), color: "yellow", overdue: false, max_wait_minutes: 60, ...o });
export const inCareRow = (o: Partial<UrgencyInCareRow> = {}): UrgencyInCareRow => ({
  encounter_id: "en3", attendance_id: "at3", ticket: "U-038", patient: patientRef({ patient_id: "pa3", name: "João Reis" }), color: "orange",
  consultation_id: null, consultation_status: null, called_by_name: "Dr. Caio", called_at: "2026-11-05T09:55:00-03:00", ...o
});
export const queue = (o: Partial<UrgencyQueue> = {}): UrgencyQueue =>
  ({ awaiting_classification: [ waitingRow() ], by_color: [ colorRow() ], in_care: [ inCareRow() ], ...o });
export const classification = (o: Partial<UrgencyClassificationView> = {}): UrgencyClassificationView => ({
  id: "c1", color: "yellow", suggested: { color: "yellow", criterion: "febre acima de 39 °C" }, complaint: "febre e dor de cabeça",
  vital_signs: { temperature_c: 39.4 }, pain_scale: 5, justification: null, classified_at: "2026-11-05T09:52:00-03:00", by: { name: "Ana Souza" }, ...o
});
export const encounter = (o: Partial<UrgencyEncounter> = {}): UrgencyEncounter => ({
  id: "en1", attendance_id: "at1", health_unit_id: "upa1", state: "waiting_classification", ticket: "U-042",
  arrived_at: "2026-11-05T09:48:00-03:00", patient: patientRef(), classifications: [], consultation_id: null, consultation_status: null,
  outcome: null, ...o
});
export const LEVELS = [
  { color: "red", max_wait_minutes: 0 }, { color: "orange", max_wait_minutes: 10 }, { color: "yellow", max_wait_minutes: 60 },
  { color: "green", max_wait_minutes: 120 }, { color: "blue", max_wait_minutes: 240 }
] as UrgencySuggestion["levels"];
export const suggestion = (o: Partial<UrgencySuggestion> = {}): UrgencySuggestion =>
  ({ suggested: { color: "yellow", criterion: "febre acima de 39 °C", rule_id: "r3" }, levels: LEVELS, ...o });
export const board = (o: Partial<WaitingRoomBoard> = {}): WaitingRoomBoard => ({
  unit: { name: "UPA Boqueirão" },
  calls: [ { ticket: "U-042", location: "Consultório 2", kind: "care", called_at: "2026-11-05T09:59:00-03:00" },
    { ticket: "U-041", location: "Classificação", kind: "classification", called_at: "2026-11-05T09:57:00-03:00" } ],
  wait_times: [ { color: "yellow", median_minutes: 34 }, { color: "green", median_minutes: "lt5_patients" } ],
  updated_at: "2026-11-05T10:00:00-03:00", ...o
});
export const urgencyUser = (roles: string[], features = [ "urgency_care", "clinical_record", "waiting_room_display" ]): SessionUser =>
  sessionWith(roles, { features } as Partial<SessionUser>);
```

(Os campos de `Professional` e `ProfessionalLink` que faltarem são conferidos na Task 0 contra o `src/lib/api.ts` real; o `as` fica só nas fixtures.)

- [ ] **Step 2: Escreva os testes do cliente** `src/lib/api.urgency.test.ts` (mesmo `fetch` falso de `src/lib/api.psychosocial.test.ts`: `vi.stubGlobal("fetch", …)` devolvendo `{ ok: true, status: 200, json }`)

```ts
it("busca de pessoa e chegada: tudo no corpo, nada na URL", async () => {
  await lookupUrgencyPerson({ cpf: "52998224725" });
  expect(lastCall()).toEqual({ url: "/urgency/person_lookup", method: "POST", body: { cpf: "52998224725" } });
  await createUrgencyArrival({ health_unit_id: "upa1", unidentified: { description: "homem, cerca de 40 anos, achado na rodoviária" } });
  expect(lastCall().url).toBe("/urgency/arrivals");
  expect(lastCall().body).toEqual({ health_unit_id: "upa1", unidentified: { description: "homem, cerca de 40 anos, achado na rodoviária" } });
  for (const c of calls()) expect(c.url).not.toMatch(/52998224725|rodovi/);
});
it("fila e painel: só ids e mês na URL", async () => {
  await getUrgencyQueue("upa1");
  expect(lastCall()).toEqual({ url: "/urgency/queue?health_unit_id=upa1", method: "GET", body: undefined });
  await getUrgencyDashboard({ month: "2026-11", unitId: null });
  expect(lastCall().url).toBe("/urgency/dashboard?month=2026-11");
  await getUrgencyDashboard({ month: "2026-11", unitId: "upa1" });
  expect(lastCall().url).toBe("/urgency/dashboard?month=2026-11&health_unit_id=upa1");
});
it("chamada do 13 leva o local só quando recebido; chamar de novo; classificação", async () => {
  await callNext("ubs1");
  expect(lastCall().body).toEqual({});
  await callNext("ubs1", "loc1");
  expect(lastCall().body).toEqual({ call_location_id: "loc1" });
  await callAttendance("at1", "ubs1", "loc1");
  expect(lastCall()).toEqual({ url: "/attendance/attendances/at1/call", method: "POST", body: { health_unit_id: "ubs1", call_location_id: "loc1" } });
  await recallAttendance("at1", "loc1");
  expect(lastCall()).toEqual({ url: "/attendance/attendances/at1/recall", method: "POST", body: { call_location_id: "loc1" } });
  await callForClassification("en1", null);
  expect(lastCall()).toEqual({ url: "/urgency/encounters/en1/call_for_classification", method: "POST", body: {} });
});
it("consulta de urgência e finalização sem desfecho do 13", async () => {
  await startConsultation("at1", { kind: "urgency" });
  expect(lastCall().body).toEqual({ kind: "urgency" });
  await finalizeConsultation("co1", null);
  expect(lastCall().body).toEqual({});
});
it("TV: sem cookie, com o token no cabeçalho; 401 vira ApiError display_revoked", async () => {
  await fetchWaitingRoomBoard("tok-abc");
  const init = lastInit();
  expect(init.credentials).toBe("omit");
  expect((init.headers as Record<string, string>)["X-Display-Token"]).toBe("tok-abc");
  expect(lastCall().url).toBe("/waiting_room/board");
  failNext(401, { error: "display_revoked" });
  await expect(fetchWaitingRoomBoard("tok-abc")).rejects.toMatchObject({ status: 401 });
  await pairWaitingRoomDisplay("123456");
  expect(lastInit().credentials).toBe("omit");
  expect(lastCall().body).toEqual({ code: "123456" });
});
it("admin de locais e TVs pelos caminhos do contrato §8", async () => {
  await listCallLocations("upa1");
  expect(lastCall().url).toBe("/attendance/call_locations?health_unit_id=upa1");
  await updateCallLocation("loc1", { name: "Consultório 1", active: false });
  expect(lastCall()).toEqual({ url: "/attendance/call_locations/loc1", method: "PATCH", body: { name: "Consultório 1", active: false } });
  await createPairingCode("d1");
  expect(lastCall()).toEqual({ url: "/waiting_room/displays/d1/pairing_code", method: "POST", body: {} });
  await revokeWaitingRoomDisplay("d1");
  expect(lastCall()).toEqual({ url: "/waiting_room/displays/d1/revoke", method: "POST", body: {} });
});
it("prefixo da senha só vai quando recebido", async () => {
  await updateUnit("upa1", "UPA", "upa", ADDRESS, undefined, undefined, undefined, undefined, { ticket_prefix: "UB" });
  expect(lastCall().body).toMatchObject({ ticket_prefix: "UB" });
  await updateUnit("upa1", "UPA", "upa", ADDRESS);
  expect("ticket_prefix" in (lastCall().body as object)).toBe(false);
});
```

E em `src/lib/features.test.ts`: `featureLabel("urgency_care")` é `"Urgência (UPA)"` e `featureLabel("waiting_room_display")` é `"Painel de chamada na sala de espera (TV)"`.

- [ ] **Step 3: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/lib/api.urgency.test.ts src/lib/features.test.ts`
Expected: FAIL (funções e rótulos inexistentes).

- [ ] **Step 4: Implemente**

- `api.ts`, seção `// ─── Urgência (módulo 32a; ADR 0050; contrato 2026-11-05 §2–§11) ───` no fim: bases `URGENCY_BASE = import.meta.env.VITE_URGENCY_BASE || "/urgency"` e `WAITING_ROOM_BASE = … || "/waiting_room"`; escritas com `{ method, body: JSON.stringify(corpo ?? {}) }`; listas desembrulham `items` (`{ items }` → array, D6/D7/D10). `fetchWaitingRoomBoard` e `pairWaitingRoomDisplay` **não** usam `jsonFetch` (ele manda cookie): `fetch(url, { credentials: "omit", headers: { Accept: "application/json", "X-Display-Token": token } })` e, fora de 2xx, `throw new ApiError(res.status, body, …)` como o `jsonFetchWithStatus`.
- `callAttendance`/`callNext`: o terceiro/segundo parâmetro opcional entra como `call_location_id` só quando não é `undefined` nem `null`.
- `finalizeConsultation(id, outcome: OutcomeBody | null)`: `body: JSON.stringify(outcome ?? {})`.
- `features.ts`: as duas chaves em `FeatureKey` e `FEATURE_LABEL` (rótulos acima, com o comentário "Módulo 32a (ADR 0050): urgency_care exige clinical_record; waiting_room_display é independente").
- `vite.config.ts`: `"/urgency": proxy(TARGET)` e `"/waiting_room": proxy(TARGET)`, e as duas linhas no comentário do topo.

- [ ] **Step 5: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/lib/api.urgency.test.ts src/lib/features.test.ts src/lib/api.test.ts src/lib/api.consultation.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.urgency.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/urgencyFixtures.ts
$G commit -m "feat: add urgency and waiting room API client, call location on attendance calls and urgency consultations" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela da urgência e o hook de polling

**Files:**
- Create: `src/lib/urgency.ts`, `src/lib/usePolling.ts`
- Test: `src/lib/urgency.test.ts`, `src/lib/usePolling.test.tsx`

**Interfaces:**
- Consumes: tipos da Task 1; `hasFeature`, `featureDisabledKey` (`src/lib/features.ts`); `NOMINAL_QUERY` (`src/lib/careLines.ts`); `Problem`, `Refusal` (`src/lib/pharmacy.ts`); `newPatientProblem` (`src/lib/territoryDictionaries.ts`); `attendanceError` (`src/lib/attendance.ts`); `screeningError` (`src/lib/screening.ts`).
- Produces (em `src/lib/urgency.ts`):
  - Frases: `URGENCY_DISABLED = "a urgência está desligada nesta cidade"`, `WAITING_ROOM_DISABLED = "o painel da sala de espera está desligado nesta cidade"`, `NO_CLINICAL_IN_LIST = "A fila não mostra queixa, sinais, CID nem a descrição de quem não foi identificado."`, `PRIORITY_PHRASE = "Quem corre mais risco é atendido primeiro, mesmo que tenha chegado depois."`, `SMALL_COUNT = "menos de 5"`.
  - Chaves: `URGENCY_QUEUE_KEY = "urgencyQueue"`, `URGENCY_DETAIL_KEY = "urgencyDetail"`, `CALL_LOCATIONS_KEY = "callLocations"`, `DISPLAYS_KEY = "waitingRoomDisplays"`, `URGENCY_DASHBOARD_KEY = "urgencyDashboard"`, `COUNTER_REFERRALS_KEY = "counterReferrals"`; `URGENCY_QUEUE_POLL_MS = 10_000`, `WAITING_ROOM_POLL_MS = 5_000`.
  - Portões: `canUseUrgency(user): boolean` (não operador, `urgency_care` **e** `clinical_record`); `canUseWaitingRoom(user): boolean` (não operador, `waiting_room_display`); `isUpa(unit: { kind: string }): boolean`; `urgencyRoles(user, links: ProfessionalLink[], unitId): { reception: boolean; nurse: boolean; doctor: boolean }` — `reception` = `citizen_verifier`; `nurse` = `health_professional` com vínculo sem `ended_at` na unidade e CBO `/^(2235|3222)/`; `doctor` = idem com `/^225/`.
  - Cores: `URGENCY_COLORS: UrgencyColor[]` (ordem do contrato), `URGENCY_COLOR_LABEL = { red: "vermelho", orange: "laranja", yellow: "amarelo", green: "verde", blue: "azul" }`, `URGENCY_COLOR_TONE = { red: "down", orange: "urgent", yellow: "warn", green: "ok", blue: "info" }`, `DEFAULT_LEVELS: UrgencyLevel[]` (0, 10, 60, 120, 240), `colorLabel(code)` (cru quando desconhecido), `maxWaitLabel(min: number): string` ("imediato" para 0, senão "até N min").
  - Desfecho: `OUTCOME_KINDS` (ordem do contrato §7), `OUTCOME_KIND_LABEL = { discharged: "Alta", discharged_to_primary_care: "Alta com retorno na UBS", transferred: "Transferência", left_without_care: "Saiu sem atendimento", left_against_advice: "Saiu contra orientação médica", death: "Óbito" }`, `outcomeLabel(code)`.
  - Validações (cada uma devolve `Problem[]`, todos de uma vez): `arrivalProblems(d: ArrivalDraft, today: string)`, `classificationProblems(d: ClassificationDraft, suggested: UrgencyColor | null)`, `outcomeProblems(d: OutcomeDraft, ctx: { consultationFinalized: boolean; doctor: boolean })`, `locationNameProblem(name)`, `ticketPrefixProblem(p)`, `pairingCodeProblem(code)`; `justificationNeeded(suggested: UrgencyColor | null, final: UrgencyColor | null): boolean` (`true` sem sugestão ou com cor diferente).
  - Rascunhos: `ArrivalDraft = { mode: "existing"; patient: UrgencyPatientRef | null } | { mode: "new"; patient: NewPatient } | { mode: "unidentified"; description: string }`; `arrivalBody(unitId, d): UrgencyArrivalInput`; `ClassificationDraft = { complaint: string; vitals: VitalsForm; color: UrgencyColor | null; justification: string }`; `classificationBody(d, parsedVitals: VitalSigns): UrgencyClassificationInput` (tira `pain_score` de `vital_signs` e manda como `pain_scale`, D11); `OutcomeDraft = { kind: UrgencyOutcomeKind | null; destination: string; vacancyRequest: string; returnDays: string; returnReason: string }`; `outcomeBody(d): UrgencyOutcomeInput` (só os campos do tipo escolhido).
  - Erros: `URGENCY_ERRORS: Record<string, string>`, `urgencyError(err): string`, `urgencyRefusals(err): Refusal[]`.
  - Local lembrado: `callLocationKey(userId, unitId) = "callLocation.<userId>.<unitId>"`, `readCallLocation(userId, unitId): string | null`, `writeCallLocation(userId, unitId, id: string | null): void` (`sessionStorage`, `try/catch`).
  - Contagens: `countLabel(c: UrgencyCount): string`, `timesLabel(t: UrgencyTimes): string` ("mediana 12 min · p90 40 min" ou `SMALL_COUNT`).
- Produces (em `src/lib/usePolling.ts`): `usePolling<T>(fetcher: (() => Promise<T>) | null, intervalMs: number, key: string): { data: T | undefined; error: unknown; updatedAt: number | null; refresh(): void }` — `fetcher` nulo para; `key` troca zera os dados e descarta a resposta em voo.

- [ ] **Step 1: Escreva os testes**

`src/lib/urgency.test.ts`:

```ts
const EVERY_CONTRACT_CODE = [
  "feature_disabled", "screening_not_in_upa", "not_an_upa", "caps_unit_not_allowed", "encounter_open", "already_identified",
  "justification_required", "call_location_required", "consultation_not_finalized", "urgency_outcome_required", "display_revoked",
  // Divergência D12: códigos que o contrato implica sem nome.
  "not_classified", "invalid_pairing_code", "destination_required", "return_guidance_required", "encounter_closed", "not_unidentified",
  "cbo_not_allowed", "missing_link", "already_called", "queue_empty", "patient_not_found", "invalid_ticket_prefix"
];
it("todo código do contrato tem frase própria", () => {
  for (const code of EVERY_CONTRACT_CODE) {
    expect(URGENCY_ERRORS[code], code).toBeTruthy();
    expect(urgencyError(new ApiError(422, { error: code }, "x"))).toBe(URGENCY_ERRORS[code]);
  }
  expect(urgencyError(new ApiError(422, { error: "brand_new" }, "x"))).toBe("recusado pelo servidor (brand_new)");
});
it("frases fixas dos códigos que mudam a tela", () => {
  expect(URGENCY_ERRORS.encounter_open).toBe("esta pessoa já tem um atendimento aberto nesta UPA");
  expect(URGENCY_ERRORS.call_location_required).toBe("escolha o local da chamada — a unidade tem TV na sala de espera");
  expect(URGENCY_ERRORS.urgency_outcome_required).toBe("numa UPA, o atendimento termina pelo desfecho da urgência");
  expect(URGENCY_ERRORS.already_identified).toBe("este paciente já foi identificado por outra pessoa");
  expect(URGENCY_ERRORS.justification_required).toBe("explique a cor escolhida (pelo menos 10 caracteres)");
});
it("papéis pelo CBO do vínculo ativo na unidade", () => {
  const u = urgencyUser([ "health_professional" ]);
  expect(urgencyRoles(u, [ NURSE_LINK ], "upa1")).toEqual({ reception: false, nurse: true, doctor: false });
  expect(urgencyRoles(u, [ { ...NURSE_LINK, cbo_code: "322205" } ], "upa1").nurse).toBe(true);
  expect(urgencyRoles(u, [ DOCTOR_LINK ], "upa1")).toEqual({ reception: false, nurse: false, doctor: true });
  expect(urgencyRoles(u, [ { ...DOCTOR_LINK, ended_at: "2026-10-01" } ], "upa1").doctor).toBe(false);
  expect(urgencyRoles(u, [ DOCTOR_LINK ], "ubs1").doctor).toBe(false);
  expect(urgencyRoles(urgencyUser([ "citizen_verifier" ]), [], "upa1").reception).toBe(true);
});
it("portões", () => {
  expect(canUseUrgency(urgencyUser([ "citizen_verifier" ]))).toBe(true);
  expect(canUseUrgency(urgencyUser([ "citizen_verifier" ], [ "urgency_care" ]))).toBe(false);
  expect(canUseWaitingRoom(urgencyUser([ "municipal_admin" ], [ "waiting_room_display" ]))).toBe(true);
  expect(canUseUrgency({ ...urgencyUser([ "municipal_admin" ]), operator: true })).toBe(false);
});
it("justificativa: sem sugestão sempre; com sugestão só se a cor mudou", () => {
  expect(justificationNeeded(null, "green")).toBe(true);
  expect(justificationNeeded("yellow", "yellow")).toBe(false);
  expect(justificationNeeded("yellow", "orange")).toBe(true);
  const d = { complaint: "dor no peito", vitals: EMPTY_VITALS_FORM, color: "orange" as const, justification: "curta" };
  expect(classificationProblems(d, "yellow").map((p) => p.path)).toEqual([ "justification" ]);
  expect(classificationProblems({ ...d, complaint: "", color: null }, null).map((p) => p.path)).toEqual([ "complaint", "color", "justification" ]);
});
it("chegada: uma forma só; descrição de 10 a 500", () => {
  expect(arrivalProblems({ mode: "unidentified", description: "homem 40a" }, "2026-11-05").map((p) => p.text))
    .toEqual([ "descreva a pessoa (de 10 a 500 caracteres): sexo e idade aparentes, sinais, onde foi encontrada" ]);
  expect(arrivalProblems({ mode: "existing", patient: null }, "2026-11-05").map((p) => p.path)).toEqual([ "patient" ]);
  expect(arrivalBody("upa1", { mode: "existing", patient: patientRef() })).toEqual({ health_unit_id: "upa1", patient_id: "pa1" });
});
it("desfecho: campos por tipo; consulta finalizada; só a evasão para quem não é médico", () => {
  const base = { kind: null, destination: "", vacancyRequest: "", returnDays: "", returnReason: "" } as OutcomeDraft;
  expect(outcomeProblems({ ...base, kind: "transferred" }, { consultationFinalized: true, doctor: true }).map((p) => p.path)).toEqual([ "destination" ]);
  expect(outcomeProblems({ ...base, kind: "discharged_to_primary_care", returnDays: "0" }, { consultationFinalized: true, doctor: true })
    .map((p) => p.path)).toEqual([ "return_guidance.days", "return_guidance.reason" ]);
  expect(outcomeProblems({ ...base, kind: "discharged" }, { consultationFinalized: false, doctor: true })[0].text)
    .toBe(URGENCY_ERRORS.consultation_not_finalized);
  expect(outcomeProblems({ ...base, kind: "left_without_care" }, { consultationFinalized: false, doctor: false })).toEqual([]);
  expect(outcomeProblems({ ...base, kind: "death" }, { consultationFinalized: true, doctor: false })[0].path).toBe("kind");
  expect(outcomeBody({ ...base, kind: "discharged_to_primary_care", returnDays: "7", returnReason: "reavaliar a pressão", destination: "x" }))
    .toEqual({ kind: "discharged_to_primary_care", return_guidance: { days: 7, reason: "reavaliar a pressão" } });
});
it("local lembrado na sessão, por usuário e unidade, e sobrevive a storage bloqueado", () => {
  writeCallLocation("u-1", "upa1", "loc2");
  expect(readCallLocation("u-1", "upa1")).toBe("loc2");
  expect(readCallLocation("u-1", "ubs1")).toBeNull();
  vi.spyOn(Storage.prototype, "getItem").mockImplementation(() => { throw new Error("blocked"); });
  expect(readCallLocation("u-1", "upa1")).toBeNull();
});
it("prefixo, código e nomes", () => {
  expect(ticketPrefixProblem("UB")).toBeNull();
  expect(ticketPrefixProblem("u")).toBe("o prefixo tem de 1 a 3 letras maiúsculas, sem acento");
  expect(pairingCodeProblem("12345")).toBe("o código tem 6 dígitos");
  expect(locationNameProblem("C")).toBe("o nome tem de 2 a 40 caracteres");
  expect(maxWaitLabel(0)).toBe("imediato");
  expect(colorLabel("purple")).toBe("purple");
});
```

`src/lib/usePolling.test.tsx` (relógio falso só aqui):

```tsx
beforeEach(() => vi.useFakeTimers());
afterEach(() => { cleanup(); vi.useRealTimers(); });
it("um pedido por vez: o próximo só depois da resposta e do intervalo", async () => {
  let resolve!: (v: number) => void;
  const fetcher = vi.fn(() => new Promise<number>((r) => { resolve = r; }));
  renderHook(() => usePolling(fetcher, 5000, "a"));
  expect(fetcher).toHaveBeenCalledTimes(1);
  await vi.advanceTimersByTimeAsync(20_000);
  expect(fetcher).toHaveBeenCalledTimes(1);
  await act(async () => resolve(1));
  await vi.advanceTimersByTimeAsync(5000);
  expect(fetcher).toHaveBeenCalledTimes(2);
});
it("desmontar limpa o timer e descarta a resposta em voo", async () => {
  let resolve!: (v: number) => void;
  const fetcher = vi.fn(() => new Promise<number>((r) => { resolve = r; }));
  const { unmount } = renderHook(() => usePolling(fetcher, 5000, "a"));
  unmount();
  await act(async () => resolve(1));
  expect(vi.getTimerCount()).toBe(0);
  await vi.advanceTimersByTimeAsync(60_000);
  expect(fetcher).toHaveBeenCalledTimes(1);
});
it("trocar a chave zera os dados e ignora a resposta da chave antiga", async () => {
  const pending: ((v: string) => void)[] = [];
  const fetcher = vi.fn(() => new Promise<string>((r) => pending.push(r)));
  const { result, rerender } = renderHook(({ k }) => usePolling(fetcher, 5000, k), { initialProps: { k: "upa1" } });
  rerender({ k: "upa2" });
  await act(async () => pending[0]("fila da upa1"));
  expect(result.current.data).toBeUndefined();
  await act(async () => pending[1]("fila da upa2"));
  expect(result.current.data).toBe("fila da upa2");
});
it("erro não para o polling; refresh pede na hora sem duplicar o timer", async () => {
  const fetcher = vi.fn().mockRejectedValueOnce(new Error("rede")).mockResolvedValue(2);
  const { result } = renderHook(() => usePolling(fetcher, 5000, "a"));
  await vi.advanceTimersByTimeAsync(0);
  expect(result.current.error).not.toBeNull();
  await vi.advanceTimersByTimeAsync(5000);
  expect(result.current.data).toBe(2);
  act(() => result.current.refresh());
  await vi.advanceTimersByTimeAsync(0);
  expect(vi.getTimerCount()).toBe(1);
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/lib/urgency.test.ts src/lib/usePolling.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `urgency.ts`: as frases de `URGENCY_ERRORS` para os códigos acima (as cinco do teste exatas; as demais em minúscula, sem nome, CPF, queixa ou descrição: `not_an_upa` "esta unidade não é uma UPA"; `caps_unit_not_allowed` "o CAPS não recebe atendimento de urgência"; `screening_not_in_upa` "na UPA a classificação de risco substitui a escuta"; `consultation_not_finalized` "finalize a consulta antes deste desfecho"; `display_revoked` "esta TV foi desligada pela administração — peça um código novo"; `not_classified` "classifique a pessoa antes de chamar para o atendimento"; `invalid_pairing_code` "código inválido ou vencido — peça um código novo à administração"; `destination_required` "informe o destino da transferência"; `return_guidance_required` "informe o prazo e o motivo do retorno na UBS"; `encounter_closed` "este atendimento já foi encerrado"; `not_unidentified` "só o paciente não identificado pode ser identificado"; `cbo_not_allowed` "seu CBO não permite esta etapa na UPA"; `missing_link` "você não tem vínculo com esta unidade"; `patient_not_found` "nenhum paciente com este documento"; `invalid_ticket_prefix` o texto do `ticketPrefixProblem`; `already_called` "outra pessoa chamou primeiro"; `queue_empty` "ninguém aguardando"; `feature_disabled` → `URGENCY_DISABLED`). `urgencyError`: `feature_disabled` com `featureDisabledKey(err) === "waiting_room_display"` → `WAITING_ROOM_DISABLED`; `implausible_vital` → `screeningError(err)`; 401 → `attendanceError(err)`; código conhecido → a frase; outro código → `recusado pelo servidor (<código>)`. `urgencyRefusals`: `problems[]` aceita `error` ou `code` (D12), cada um com `path`. `arrivalProblems` do modo `new` reusa `newPatientProblem(patient, today)`.
- `usePolling.ts`: `useEffect` por `[ key, intervalMs, fetcher === null ]` com `let live = true; let timer: ReturnType<typeof setTimeout> | undefined; const seq = ++seqRef.current`; `tick()` chama o `fetcherRef.current()`, e no `finally`, se `live && seq === seqRef.current`, agenda `timer = setTimeout(tick, intervalMs)`; o retorno faz `live = false; clearTimeout(timer)`. Ao trocar `key`, `setData(undefined)`. `refresh()` faz `clearTimeout(timer)` e `tick()` (guardados em ref). O `fetcher` vai num `useRef` atualizado a cada render, para não reiniciar o ciclo.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/lib/urgency.test.ts src/lib/usePolling.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/urgency.ts src/lib/urgency.test.ts src/lib/usePolling.ts src/lib/usePolling.test.tsx
$G commit -m "feat: add urgency screen rules, error texts and a leak-free polling hook" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Urgência", telas-portão e a rota da TV fora do shell

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/main.tsx`
- Create: `src/lib/tvRoute.ts`, `src/modules/UrgencyPanel.tsx`, `src/modules/WaitingRoomSettings.tsx`, `src/modules/WaitingRoomTv.tsx` (casca; o conteúdo vem nas Tasks 11–13)
- Test: `src/shell/modules.urgency.test.ts`, `src/lib/tvRoute.test.ts`

**Interfaces:**
- Consumes: `canUseUrgency`, `canUseWaitingRoom` (Task 2).
- Produces: `ModuleId` ganha `"urgency-panel" | "waiting-room-settings"`; grupo `{ label: "Urgência", items: [ { id: "urgency-panel", label: "Painel da urgência", icon: "▦" }, { id: "waiting-room-settings", label: "Senhas, locais e TVs", icon: "▭" } ] }` depois de "CAPS"; `isTvHash(hash: string): boolean` (`"#/tv"` e `"#/tv/"`); `WaitingRoomTv()` (sem props); `UrgencyPanel()`, `WaitingRoomSettings({ onGoToSecurity })`.

- [ ] **Step 1: Escreva os testes**

`src/shell/modules.urgency.test.ts`:

```ts
const ids = (u: SessionUser) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));
it("painel para admin e analista, com urgency_care e clinical_record", () => {
  expect(ids(urgencyUser([ "municipal_admin" ]))).toContain("urgency-panel");
  expect(ids(urgencyUser([ "analyst" ]))).toContain("urgency-panel");
  expect(ids(urgencyUser([ "health_professional" ]))).not.toContain("urgency-panel");
  expect(ids(urgencyUser([ "municipal_admin" ], [ "urgency_care", "waiting_room_display" ]))).not.toContain("urgency-panel");
});
it("senhas, locais e TVs só do admin e só com waiting_room_display (vale também sem urgency_care)", () => {
  expect(ids(urgencyUser([ "municipal_admin" ], [ "waiting_room_display" ]))).toContain("waiting-room-settings");
  expect(ids(urgencyUser([ "municipal_admin" ], [ "urgency_care", "clinical_record" ]))).not.toContain("waiting-room-settings");
  expect(ids(urgencyUser([ "analyst" ]))).not.toContain("waiting-room-settings");
});
it("o grupo some inteiro sem nada visível e tem ícone e descrição no catálogo", () => {
  expect(navGroupsFor(urgencyUser([ "citizen_verifier" ])).some((g) => g.label === "Urgência")).toBe(false);
  expect(CATALOG.groups["Urgência"].description).toBe("UPA: tempos por cor, desfechos e o painel de chamada da sala de espera");
  expect(CATALOG.items["urgency-panel"].description).toBe("Tempos por cor, fora do tempo, desfechos e evasão");
  expect(CATALOG.items["waiting-room-settings"].description).toBe("Locais de chamada e TVs da sala de espera");
});
```

(`CATALOG.groups`/`CATALOG.items`: confira os nomes reais das chaves em `src/shell/types.ts` na Task 0; em `modules.test.ts`, o deslocamento `NAV_GROUPS.length - N` sobe 1.)

`src/lib/tvRoute.test.ts`: `isTvHash("#/tv")`, `isTvHash("#/tv/")` são `true`; `isTvHash("#/")`, `isTvHash("#/tv-x")`, `isTvHash("#/grupo/tv")` são `false`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/shell/modules.urgency.test.ts src/lib/tvRoute.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `modules.ts`: ids, grupo e, em `navGroupsFor`, `const urgOn = canUseUrgency(user); const wrOn = canUseWaitingRoom(user);` com `if (item.id === "urgency-panel") return urgOn && (isAdmin || isAnalyst);` e `if (item.id === "waiting-room-settings") return wrOn && isAdmin;`.
- `catalog.ts`: grupo com `IconAmbulance`, `urgency-panel` com `IconChartBar`, `waiting-room-settings` com `IconDeviceTv`, e as descrições do teste.
- `App.tsx`: `case "urgency-panel": return <UrgencyPanel />;` e `case "waiting-room-settings": return <WaitingRoomSettings onGoToSecurity={() => setActive("security")} />;`.
- `tvRoute.ts`: `isTvHash`.
- `main.tsx`: antes do `createRoot(...).render(<StrictMode><AuthProvider>…`, `if (isTvHash(window.location.hash)) { createRoot(root).render(<StrictMode><WaitingRoomTv /></StrictMode>); } else { …o render de hoje… }` — a TV nunca monta `AuthProvider` (nenhum `/session`), nunca entra no `useHashRoute` do shell e importa `./theme/global.css` como hoje.
- Cascas: `UrgencyPanel` e `WaitingRoomSettings` com `PageHeader` e `EmptyState`; `WaitingRoomTv` com `<main aria-label="Painel de chamada">`.

- [ ] **Step 4: Rode e veja passar; regressão do shell; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/shell src/lib/tvRoute.test.ts src/App.navigation.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.urgency.test.ts src/shell/catalog.ts src/App.tsx src/main.tsx \
  src/lib/tvRoute.ts src/lib/tvRoute.test.ts src/modules/UrgencyPanel.tsx src/modules/WaitingRoomSettings.tsx src/modules/WaitingRoomTv.tsx
$G commit -m "feat: add the urgency menu group and a TV route outside the shell" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Protocolo `urgency_classification` no editor de protocolos (F-32.1)

**Files:**
- Create: `src/lib/urgencyRules.ts`, `src/modules/protocolEditor/UrgencyLevelsPanel.tsx`, `src/modules/protocolEditor/UrgencyRulesPanel.tsx`
- Modify: `src/lib/condition.ts`, `src/modules/ProtocolEditor.tsx`
- Test: `src/lib/urgencyRules.test.ts`, `src/modules/protocolEditor/UrgencyPanels.test.tsx`

**Interfaces:**
- Consumes: `ConditionBuilder`, `fieldsFor`, `ConditionTree`; `URGENCY_COLORS`, `URGENCY_COLOR_LABEL`, `DEFAULT_LEVELS` (Task 2); a gravação, o gate e o ciclo de vida do 18 como estão (`gateProtocol`, `saveProtocolDraft`, Protocols).
- Produces: `ConditionContext` ganha `"urgency"` (`fieldsFor("urgency")` = os campos de sinais vitais do 18 **sem** `vitals.pain_score`, mais `{ id: "pain_scale", label: "dor (0 a 10)", kind: "number", group: "Dor" }`, idade e sexo; D13); em `urgencyRules.ts`: `URGENCY_PROTOCOL_NAME = "classificacao_de_risco"`, `URGENCY_RULES_MAX = 100`, `isUrgencyDefinition(def): boolean`, `readLevels(def): { ok: true; levels: UrgencyLevel[] } | { ok: false; reason: string }`, `writeLevels(def, levels): unknown`, `levelProblem(l: UrgencyLevel): string | null`, `UrgencyRuleDraft { id: string; when: ConditionTree | null; color: UrgencyColor; criterion: string }`, `readUrgencyRules(def)`, `writeUrgencyRules(def, rules)`, `urgencyRuleProblems(r): string[]`, `nextRuleId(rules): string`, `URGENCY_TEMPLATE: string`; `UrgencyLevelsPanel({ definition, onChange })`, `UrgencyRulesPanel({ definition, onChange })`.

- [ ] **Step 1: Escreva os testes**

`src/lib/urgencyRules.test.ts`:

```ts
it("modelo: kind, as 5 cores na ordem com os tempos padrão, regras com id, condição, cor e critério", () => {
  const def = JSON.parse(URGENCY_TEMPLATE);
  expect(def.kind).toBe("urgency_classification");
  expect(def.name).toBe("classificacao_de_risco");
  expect(def.levels).toEqual([ { color: "red", max_wait_minutes: 0 }, { color: "orange", max_wait_minutes: 10 },
    { color: "yellow", max_wait_minutes: 60 }, { color: "green", max_wait_minutes: 120 }, { color: "blue", max_wait_minutes: 240 } ]);
  expect(def.rules[0]).toEqual({ id: "r1", when: { lte: [ "vitals.spo2", 89 ] }, color: "red", criterion: "saturação de oxigênio até 89%" });
  expect(isUrgencyDefinition(def)).toBe(true);
  expect(isUrgencyDefinition({ kind: "screening" })).toBe(false);
});
it("níveis: lê e escreve sem tocar no resto; tempo inteiro de 0 a 1440 e não decrescente", () => {
  const def = { kind: "urgency_classification", name: "x", levels: DEFAULT_LEVELS, rules: [] };
  const next = writeLevels(def, DEFAULT_LEVELS.map((l) => (l.color === "yellow" ? { ...l, max_wait_minutes: 45 } : l))) as typeof def;
  expect(next.name).toBe("x");
  expect(readLevels(next)).toEqual({ ok: true, levels: expect.arrayContaining([ { color: "yellow", max_wait_minutes: 45 } ]) });
  expect(levelProblem({ color: "green", max_wait_minutes: 1.5 })).toBe("o tempo é um número inteiro de minutos, de 0 a 1440");
  expect(readLevels({ kind: "urgency_classification", levels: [ { color: "red", max_wait_minutes: 0 } ] }))
    .toEqual({ ok: false, reason: "“levels” precisa das 5 cores, na ordem vermelho, laranja, amarelo, verde, azul: corrija no JSON" });
});
it("regra: critério de 3 a 120 caracteres e condição definida; ids únicos", () => {
  expect(urgencyRuleProblems({ id: "r1", when: null, color: "red", criterion: "x" }))
    .toEqual([ "defina quando sugerir esta cor", "o critério tem de 3 a 120 caracteres" ]);
  expect(nextRuleId([ { id: "r1" }, { id: "r7" } ] as UrgencyRuleDraft[])).toBe("r8");
});
```

`src/modules/protocolEditor/UrgencyPanels.test.tsx` (o `ProtocolEditor` montado com `gateProtocol`, `listAuthorProtocols`, `listAppointmentTypes` mockados):

```tsx
it("'Nova classificação de risco (modelo)' carrega o modelo e troca a coluna da direita", async () => {
  render(<ProtocolEditor />);
  fireEvent.change(screen.getByLabelText("Carregar versão"), { target: { value: "__new_urgency__" } });
  expect(await screen.findByRole("heading", { name: "Cores e tempos máximos" })).not.toBeNull();
  expect(screen.getByRole("heading", { name: "Regras de cor (classificação de risco)" })).not.toBeNull();
  expect(screen.queryByRole("heading", { name: "Oferta e sugestões" })).toBeNull();
});
it("editar o tempo do amarelo e o critério reescreve o JSON", async () => {
  render(<ProtocolEditor />);
  fireEvent.change(screen.getByLabelText("Carregar versão"), { target: { value: "__new_urgency__" } });
  fireEvent.change(await screen.findByLabelText("Tempo máximo do amarelo (min)"), { target: { value: "45" } });
  fireEvent.change(screen.getAllByLabelText("Critério (aparece para quem classifica)")[0], { target: { value: "SpO2 baixa" } });
  const json = JSON.parse((screen.getByLabelText("Definição JSON") as HTMLTextAreaElement).value);
  expect(json.levels[2]).toEqual({ color: "yellow", max_wait_minutes: 45 });
  expect(json.rules[0].criterion).toBe("SpO2 baixa");
});
it("aviso de revisão clínica e de conteúdo licenciado", async () => {
  render(<ProtocolEditor />);
  fireEvent.change(screen.getByLabelText("Carregar versão"), { target: { value: "__new_urgency__" } });
  expect(await screen.findByText("O modelo é só exemplo: as regras precisam de revisão clínica e das assinaturas antes de valer. Fluxogramas do Manchester dependem de licença da cidade.")).not.toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/lib/urgencyRules.test.ts src/modules/protocolEditor/UrgencyPanels.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `urgencyRules.ts` no padrão de `riskRules.ts` (a definição JSON do editor é a fonte; as funções devolvem uma nova sem tocar no resto). O modelo tem as 5 cores padrão e 5 regras de exemplo: `r1` SpO2 ≤ 89 → red "saturação de oxigênio até 89%"; `r2` sistólica ≥ 180 → orange "pressão sistólica a partir de 180 mmHg"; `r3` temperatura ≥ 39 → yellow "febre a partir de 39 °C"; `r4` `pain_scale` ≥ 8 → orange "dor intensa (8 a 10)"; `r5` `pain_scale` ≥ 4 → yellow "dor moderada (4 a 7)". Forma do JSON: D13.
- `condition.ts`: o contexto `"urgency"` em `fieldsFor`.
- `UrgencyLevelsPanel`: título "Cores e tempos máximos", uma linha por cor com "Tempo máximo do <cor> (min)" (`inputMode="numeric"`), o problema sob o campo e a frase "Vale para o estouro na fila e para o painel do gestor." Cores fixas, sem adicionar nem remover.
- `UrgencyRulesPanel`: como o `RiskRulesPanel` (cor sugerida nas 5 cores, `ConditionBuilder` com `fieldsFor("urgency")`, remover, "+ regra de cor" até `URGENCY_RULES_MAX`), mais "Critério (aparece para quem classifica)" e a frase "Vale a cor mais grave entre as regras que casarem. É só sugestão: a cor final é de quem classifica, com justificativa quando muda."
- `ProtocolEditor.tsx`: opção `<option value="__new_urgency__">Nova classificação de risco (modelo)</option>` só com `canUseUrgency(user)` (lido de `useAuth()`); `onPick` carrega `URGENCY_TEMPLATE`; `const urgencyKind = current.ok && isUrgencyDefinition(current.value)` troca a coluna da direita pelos dois painéis e o aviso do teste. Sem simulador (D3). O ciclo de vida (enviar, assinar, publicar, ativar) segue no "Protocolos" do 18 sem mudança.

- [ ] **Step 4: Rode e veja passar; regressão do editor; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/lib/urgencyRules.test.ts src/modules/protocolEditor src/modules/ProtocolEditor.test.tsx src/lib/condition.test.ts src/lib/riskRules.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/urgencyRules.ts src/lib/urgencyRules.test.ts src/lib/condition.ts src/modules/protocolEditor/UrgencyLevelsPanel.tsx \
  src/modules/protocolEditor/UrgencyRulesPanel.tsx src/modules/ProtocolEditor.tsx src/modules/protocolEditor/UrgencyPanels.test.tsx
$G commit -m "feat: edit the urgency classification protocol with five colors and maximum waits" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Mesa da UPA no Atendimento — recepção, paciente novo, não identificado e senha (F-32.2)

**Files:**
- Create: `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/urgency/ArrivalForm.tsx`, `src/modules/urgency/TicketCard.tsx`
- Modify: `src/modules/Attendance.tsx`
- Test: `src/modules/urgency/UrgencyReception.test.tsx`, `src/modules/Attendance.urgency.test.tsx`

**Interfaces:**
- Consumes: `lookupUrgencyPerson`, `createUrgencyArrival`, `getMyProfessional` (Task 1); `canUseUrgency`, `isUpa`, `urgencyRoles`, `arrivalProblems`, `arrivalBody`, `urgencyError`, `urgencyRefusals` (Task 2); `newPatientProblem`, `maskCpf`, `isValidCpf`, `onlyDigits`.
- Produces: `UrgencyDesk({ unit, user, links, now?, onGoToSecurity? })` — monta, por papel, `ArrivalForm` (recepção), `UrgencyQueue` (Task 6) e os painéis das Tasks 7–10; `ArrivalForm({ unit, onArrived(a: UrgencyArrival): void })`; `TicketCard({ ticket, unitName, onDone })`.

- [ ] **Step 1: Escreva os testes**

`src/modules/urgency/UrgencyReception.test.tsx` (mesa montada com `urgencyUser(["citizen_verifier"])`, `UPA`, `links=[]`):

```tsx
it("CPF achado: um clique registra a chegada e mostra a senha grande para entregar", async () => {
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: patientRef(), candidates: [] });
  vi.mocked(api.createUrgencyArrival).mockResolvedValue({ attendance_id: "at9", encounter_id: "en9", ticket: "U-043", patient: patientRef() });
  mount();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "529.982.247-25" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  fireEvent.click(await screen.findByRole("button", { name: "Registrar chegada de Maria Lima" }));
  expect(await screen.findByRole("heading", { name: "Senha U-043" })).not.toBeNull();
  expect(screen.getByText("Entregue a senha à pessoa: ela será chamada por ela na TV.")).not.toBeNull();
  expect(vi.mocked(api.lookupUrgencyPerson).mock.calls[0][0]).toEqual({ cpf: "52998224725" });
  expect(vi.mocked(api.createUrgencyArrival).mock.calls[0][0]).toEqual({ health_unit_id: "upa1", patient_id: "pa1" });
});
it("CNS de 15 dígitos vai como cns; sem senha (cidade sem TV) mostra 'Chegada registrada' sem número", async () => {
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: patientRef(), candidates: [] });
  vi.mocked(api.createUrgencyArrival).mockResolvedValue({ attendance_id: "at9", encounter_id: "en9", ticket: null, patient: patientRef() });
  mount();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "700000000000005" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  fireEvent.click(await screen.findByRole("button", { name: "Registrar chegada de Maria Lima" }));
  expect(await screen.findByText("Chegada registrada. A pessoa aguarda a classificação de risco.")).not.toBeNull();
  expect(screen.queryByRole("heading", { name: /^Senha/ })).toBeNull();
  expect(vi.mocked(api.lookupUrgencyPerson).mock.calls[0][0]).toEqual({ cns: "700000000000005" });
});
it("não identificado: descrição obrigatória de 10 a 500; o corpo leva só unidentified", async () => {
  vi.mocked(api.createUrgencyArrival).mockResolvedValue({ attendance_id: "at9", encounter_id: "en9", ticket: "U-044",
    patient: patientRef({ name: "Não identificado", unidentified: true }) });
  mount();
  fireEvent.click(screen.getByRole("button", { name: "Pessoa sem identificação" }));
  fireEvent.change(screen.getByLabelText("Descrição"), { target: { value: "homem 40a" } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar chegada" }));
  expect(screen.getByText("descreva a pessoa (de 10 a 500 caracteres): sexo e idade aparentes, sinais, onde foi encontrada")).not.toBeNull();
  expect(api.createUrgencyArrival).not.toHaveBeenCalled();
  fireEvent.change(screen.getByLabelText("Descrição"), { target: { value: "homem, cerca de 40 anos, achado na rodoviária" } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar chegada" }));
  expect(await screen.findByRole("heading", { name: "Senha U-044" })).not.toBeNull();
  expect(vi.mocked(api.createUrgencyArrival).mock.calls[0][0]).toEqual({ health_unit_id: "upa1",
    unidentified: { description: "homem, cerca de 40 anos, achado na rodoviária" } });
});
it("paciente novo: todas as recusas de uma vez e o corpo só com new_patient", async () => {
  mount();
  fireEvent.click(screen.getByRole("button", { name: "Paciente novo" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar chegada" }));
  expect(screen.getAllByRole("alert").length).toBeGreaterThan(0);
  expect(api.createUrgencyArrival).not.toHaveBeenCalled();
});
it("duplo clique envia uma vez; encounter_open mostra a frase e mantém o formulário", async () => {
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: patientRef(), candidates: [] });
  vi.mocked(api.createUrgencyArrival).mockRejectedValue(new ApiError(409, { error: "encounter_open" }, "x"));
  mount();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "52998224725" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  const go = await screen.findByRole("button", { name: "Registrar chegada de Maria Lima" });
  fireEvent.click(go); fireEvent.click(go);
  expect(await screen.findByText("esta pessoa já tem um atendimento aberto nesta UPA")).not.toBeNull();
  expect(api.createUrgencyArrival).toHaveBeenCalledTimes(1);
  expect(screen.getByRole("button", { name: "Registrar chegada de Maria Lima" })).not.toBeNull();
});
it("sem ninguém pelo documento: oferece paciente novo e não identificado, e o CPF não aparece em lugar nenhum além do campo", async () => {
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: null, candidates: [] });
  mount();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "52998224725" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  expect(await screen.findByText("Ninguém com este documento. Cadastre como paciente novo ou registre sem identificação.")).not.toBeNull();
  expect(document.body.textContent).not.toContain("52998224725");
});
```

`src/modules/Attendance.urgency.test.tsx`:

```tsx
it("UPA com urgency_care: a mesa da UPA no lugar do check-in, da escuta e da fila do 13", async () => {
  pickUnit(UPA); mountAttendance(urgencyUser([ "citizen_verifier" ]));
  expect(await screen.findByRole("heading", { name: "Recepção da UPA" })).not.toBeNull();
  expect(screen.queryByRole("heading", { name: "Check-in" })).toBeNull();
  expect(screen.queryByRole("heading", { name: "Fila" })).toBeNull();
});
it("UPA sem urgency_care e UBS: o Atendimento de hoje", async () => {
  pickUnit(UPA); mountAttendance(urgencyUser([ "citizen_verifier" ], [ "clinical_record" ]));
  expect(await screen.findByRole("heading", { name: "Check-in" })).not.toBeNull();
  cleanup(); pickUnit(UBS); mountAttendance(urgencyUser([ "citizen_verifier" ]));
  expect(await screen.findByRole("heading", { name: "Check-in" })).not.toBeNull();
});
```

(`pickUnit` grava `currentUnitKey("u-1")` e mocka `listActiveUnits`; os títulos reais do 13 — "Check-in", "Fila" — conferidos na Task 0 em `Attendance.test.tsx`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency/UrgencyReception.test.tsx src/modules/Attendance.urgency.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `Attendance.tsx`: `const upaDesk = !!unit && isUpa(unit) && canUseUrgency(user);` — com `upaDesk`, no lugar de `CheckIn`, `ScreeningQueue`, `UnitQueue`, `UnitProcedures`, `Requests` e `Agenda` (e do `PageIndex`), `<UrgencyDesk unit={unit} user={user} links={myProfessional.data?.links ?? []} onGoToSecurity={goToSecurity} />`. A validação presencial (`Counter`), os pedidos sem unidade, a exclusão, o histórico e o cadastro de unidades ficam como estão.
- `UrgencyDesk.tsx`: `urgencyRoles(user, links, unit.id)`; sem papel nenhum na UPA, `EmptyState` "seu papel não atua nesta UPA"; com `reception`, o `Panel` "Recepção da UPA" com `ArrivalForm` e, depois da chegada, `TicketCard`; a fila (Task 6) para os três papéis. Monta com `key={unit.id}` (trocar de UPA remonta tudo).
- `ArrivalForm.tsx`: um `<form>` "CPF ou CNS" (11 dígitos válidos → `{ cpf }`, 15 → `{ cns }`, senão a frase "digite o CPF (11 dígitos) ou o CNS (15 dígitos)") + "Buscar" (mutação no clique, nunca ao digitar); resultado com o paciente ("Registrar chegada de <nome>") e os `candidates` como botões iguais; e dois botões fixos, "Paciente novo" (nome completo, nascimento, sexo, CPF ou motivo sem CPF da lista do 24a, CNS; `newPatientProblem`) e "Pessoa sem identificação" ("Descrição" com contador "N/500" e a dica "sexo e idade aparentes, sinais, roupas, onde foi encontrada"). `busy` + `useRef`; recusa por `urgencyRefusals` no campo do `path`. Sucesso limpa o CPF digitado e os campos.
- `TicketCard.tsx`: `<h2>` "Senha U-043" em fonte grande (`fontSize: 44`, `fontFamily: var(--font-mono)`), a unidade, a frase do teste, e "Próxima chegada" (foco volta ao "CPF ou CNS"). Sem senha: só a frase "Chegada registrada…". Foco no título ao aparecer.

- [ ] **Step 4: Rode e veja passar; regressão do Atendimento; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency src/modules/Attendance*.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/UrgencyDesk.tsx src/modules/urgency/ArrivalForm.tsx src/modules/urgency/TicketCard.tsx src/modules/Attendance.tsx \
  src/modules/urgency/UrgencyReception.test.tsx src/modules/Attendance.urgency.test.tsx
$G commit -m "feat: add the UPA reception with unidentified patients and the ticket to hand over" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Fila da UPA por cor, estouro, "aguardando classificação" e "em atendimento" (F-32.3)

**Files:**
- Create: `src/modules/urgency/UrgencyQueue.tsx`
- Modify: `src/modules/urgency/UrgencyDesk.tsx`
- Test: `src/modules/urgency/UrgencyQueue.test.tsx`

**Interfaces:**
- Consumes: `getUrgencyQueue` (Task 1); `usePolling`, `URGENCY_QUEUE_POLL_MS`, `URGENCY_COLOR_LABEL`, `URGENCY_COLOR_TONE`, `NO_CLINICAL_IN_LIST`, `urgencyError` (Task 2); `waitLabel` (`src/lib/screening.ts`); `DataTable`, `Tag`.
- Produces: `UrgencyQueue({ unit, roles, now?, actions: QueueActions })` com `QueueActions { onClassify?(row: UrgencyWaitingRow | UrgencyColorRow): void; onCallClassification?(row): void; onCallCare?(row: UrgencyColorRow): void; onRecall?(row: { attendance_id: string }): void; onOpenCare?(row: UrgencyInCareRow): void; onIdentify?(row): void; onLeft?(row): void }` — as Tasks 7–10 preenchem; `refresh` exposto por `ref` (`useImperativeHandle`, `UrgencyQueueHandle { refresh(): void }`).

- [ ] **Step 1: Escreva os testes**

```tsx
it("três faixas: aguardando classificação, por cor (na ordem do api) e em atendimento", async () => {
  vi.mocked(api.getUrgencyQueue).mockResolvedValue(queue({
    by_color: [ colorRow({ encounter_id: "r", color: "red", ticket: "U-039", waited_minutes: 3, overdue: true, max_wait_minutes: 0 }),
      colorRow({ encounter_id: "y", color: "yellow", ticket: "U-040", waited_minutes: 75, overdue: true }),
      colorRow({ encounter_id: "g", color: "green", ticket: "U-041", waited_minutes: 20 }) ]
  }));
  mount({ roles: { reception: false, nurse: true, doctor: true } });
  const awaiting = await screen.findByRole("region", { name: "Aguardando classificação" });
  expect(within(awaiting).getByText("U-042")).not.toBeNull();
  expect(within(awaiting).getByText("12 min desde a chegada")).not.toBeNull();
  const byColor = screen.getByRole("region", { name: "Classificados, por cor" });
  expect(within(byColor).getAllByRole("row").slice(1).map((r) => within(r).getByTestId("ticket").textContent)).toEqual([ "U-039", "U-040", "U-041" ]);
  expect(within(byColor).getByText("vermelho")).not.toBeNull();
  expect(within(byColor).getAllByText(/passou do tempo/).length).toBe(2);
  expect(within(byColor).getByText("75 min · passou do tempo (até 60 min)")).not.toBeNull();
  expect(screen.getByRole("region", { name: "Em atendimento" })).not.toBeNull();
});
it("lista não mostra texto clínico nem a descrição, mesmo que o api mande", async () => {
  const leaky = { ...waitingRow({ patient: patientRef({ name: "Não identificado", unidentified: true, description: "achado na rodoviária" }) }),
    complaint: "dor no peito", cid: "I21" } as never;
  vi.mocked(api.getUrgencyQueue).mockResolvedValue(queue({ awaiting_classification: [ leaky ] }));
  mount({ roles: { reception: true, nurse: false, doctor: false } });
  expect(await screen.findByText("Não identificado")).not.toBeNull();
  expect(document.body.textContent).not.toMatch(/rodoviária|dor no peito|I21/);
  expect(screen.getByText(NO_CLINICAL_IN_LIST)).not.toBeNull();
});
it("estouro também por texto, nunca só por cor", async () => {
  vi.mocked(api.getUrgencyQueue).mockResolvedValue(queue({ by_color: [ colorRow({ overdue: true, waited_minutes: 61 }) ] }));
  mount({ roles: { reception: false, nurse: true, doctor: false } });
  expect(await screen.findByText("61 min · passou do tempo (até 60 min)")).not.toBeNull();
});
describe("polling", () => {
  beforeEach(() => vi.useFakeTimers()); afterEach(() => vi.useRealTimers());
  it("relê a cada 10 s; trocar de unidade descarta a fila antiga e para o timer dela", async () => {
    vi.mocked(api.getUrgencyQueue).mockResolvedValue(queue());
    const { rerender, unmount } = mount({ roles: { reception: true, nurse: false, doctor: false } });
    await vi.advanceTimersByTimeAsync(10_000);
    expect(api.getUrgencyQueue).toHaveBeenCalledTimes(2);
    rerender({ unit: { ...UPA, id: "upa2" } });
    expect(vi.mocked(api.getUrgencyQueue).mock.calls.at(-1)?.[0]).toBe("upa2");
    unmount();
    expect(vi.getTimerCount()).toBe(0);
  });
});
```

(`mount` aceita `rerender` com novas props; o `UrgencyDesk` já remonta por `key`, e o teste prova também o `usePolling` sem a remontagem.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency/UrgencyQueue.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `UrgencyQueue.tsx`: `usePolling(() => getUrgencyQueue(unit.id), URGENCY_QUEUE_POLL_MS, unit.id)`; três `<section aria-label>` com `DataTable` (`card: "title"` na senha e no nome, `card: "actions"` nos botões). Linhas montadas **campo a campo** a partir de `encounter_id`, `attendance_id`, `ticket`, `patient.name`, `patient.cns_masked`, `patient.unidentified`, `arrived_at`, `waited_minutes`, `color`, `overdue`, `max_wait_minutes`, `last_call` — nunca `...row`. Colunas: Senha (`data-testid="ticket"`, mono, "—" sem senha), Pessoa (nome; "Não identificado" com `Tag` "sem identificação"), Cor (`Tag` com o texto da cor), Espera ("N min" e, com `overdue`, "· passou do tempo (<maxWaitLabel>)" em `var(--down)`), Última chamada ("classificação · Consultório 1 · 09:58"), Ações (os botões que as `actions` e os `roles` permitirem). Erro de leitura: `urgencyError` em `role="alert"`, mantendo a última fila. Rodapé `NO_CLINICAL_IN_LIST` e "atualiza a cada 10 s · <hora da última leitura>".

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/UrgencyQueue.tsx src/modules/urgency/UrgencyDesk.tsx src/modules/urgency/UrgencyQueue.test.tsx
$G commit -m "feat: show the UPA queue by risk color with overdue waits and the awaiting classification band" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Classificação de risco — sinais vitais, cor sugerida com critério, justificativa e reclassificação (F-32.3)

**Files:**
- Create: `src/modules/urgency/ClassificationForm.tsx`, `src/modules/urgency/EncounterSummary.tsx`
- Modify: `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/attendance/VitalSignsFields.tsx` (prop `omit?: VitalKey[]`)
- Test: `src/modules/urgency/ClassificationForm.test.tsx`

**Interfaces:**
- Consumes: `getUrgencyEncounter`, `suggestUrgencyColor`, `classifyUrgency` (Task 1); `classificationProblems`, `classificationBody`, `justificationNeeded`, `DEFAULT_LEVELS`, `URGENCY_COLORS`, `maxWaitLabel`, `urgencyRefusals`, `URGENCY_DETAIL_KEY`, `NOMINAL_QUERY` (Task 2); `VitalSignsFields`, `parseVitals`, `EMPTY_VITALS_FORM` (18); `useDebouncedValue`.
- Produces: `ClassificationForm({ encounterId, suggestDelayMs?, onDone(e: UrgencyEncounter): void, onCancel(): void })`; `EncounterSummary({ encounter })` (cor vigente, histórico de classificações em ordem, a descrição do não identificado — o único lugar além do detalhe de identificação onde ela aparece); `VitalSignsFields` ganha `omit?: VitalKey[]` (padrão `[]`, o 18 não muda).

- [ ] **Step 1: Escreva os testes**

```tsx
it("sugestão com critério; cor igual não pede justificativa; envia queixa, sinais sem a dor e a dor à parte", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(encounter());
  vi.mocked(api.suggestUrgencyColor).mockResolvedValue(suggestion());
  vi.mocked(api.classifyUrgency).mockResolvedValue({ encounter: encounter({ state: "classified", classifications: [ classification() ] }),
    suggested: suggestion().suggested });
  mount({ suggestDelayMs: 0 });
  fireEvent.change(await screen.findByLabelText("Queixa principal"), { target: { value: "febre e dor de cabeça" } });
  fireEvent.change(screen.getByLabelText("Temperatura (°C)"), { target: { value: "39,4" } });
  fireEvent.change(screen.getByLabelText("Dor (0 a 10)"), { target: { value: "5" } });
  expect(await screen.findByText("Sugerida: amarelo — febre acima de 39 °C")).not.toBeNull();
  fireEvent.click(screen.getByRole("radio", { name: "amarelo · até 60 min" }));
  expect(screen.queryByLabelText("Justificativa")).toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Classificar" }));
  await waitFor(() => expect(api.classifyUrgency).toHaveBeenCalledTimes(1));
  const [ id, body ] = vi.mocked(api.classifyUrgency).mock.calls[0];
  expect(id).toBe("en1");
  expect(body).toEqual({ complaint: "febre e dor de cabeça", vital_signs: { temperature_c: 39.4 }, pain_scale: 5, color: "yellow" });
});
it("sem protocolo ativo: aviso e justificativa obrigatória", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(encounter());
  vi.mocked(api.suggestUrgencyColor).mockResolvedValue(suggestion({ suggested: null }));
  mount({ suggestDelayMs: 0 });
  fireEvent.change(await screen.findByLabelText("Queixa principal"), { target: { value: "corte no braço" } });
  expect(await screen.findByText("Sem cor sugerida: a cidade não tem protocolo ativo ou nenhuma regra casou. Explique a cor escolhida.")).not.toBeNull();
  fireEvent.click(screen.getByRole("radio", { name: "verde · até 120 min" }));
  fireEvent.click(screen.getByRole("button", { name: "Classificar" }));
  expect(screen.getByText("explique a cor escolhida (pelo menos 10 caracteres)")).not.toBeNull();
  expect(api.classifyUrgency).not.toHaveBeenCalled();
});
it("cor trocada pede justificativa; justification_required do api mostra o campo e mantém tudo; duplo clique envia uma vez", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(encounter());
  vi.mocked(api.suggestUrgencyColor).mockResolvedValue(suggestion());
  vi.mocked(api.classifyUrgency).mockRejectedValue(new ApiError(422, { error: "justification_required" }, "x"));
  mount({ suggestDelayMs: 0 });
  fireEvent.change(await screen.findByLabelText("Queixa principal"), { target: { value: "febre" } });
  await screen.findByText("Sugerida: amarelo — febre acima de 39 °C");
  fireEvent.click(screen.getByRole("radio", { name: "laranja · até 10 min" }));
  fireEvent.change(screen.getByLabelText("Justificativa"), { target: { value: "criança de 2 meses, febre alta" } });
  const go = screen.getByRole("button", { name: "Classificar" });
  fireEvent.click(go); fireEvent.click(go);
  expect(await screen.findByText("explique a cor escolhida (pelo menos 10 caracteres)")).not.toBeNull();
  expect(api.classifyUrgency).toHaveBeenCalledTimes(1);
  expect((screen.getByLabelText("Queixa principal") as HTMLInputElement).value).toBe("febre");
  expect((screen.getByLabelText("Justificativa") as HTMLTextAreaElement).value).toBe("criança de 2 meses, febre alta");
});
it("reclassificação: mostra a cor vigente e o histórico; o botão diz 'Reclassificar'", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(encounter({ state: "classified", classifications: [
    classification({ id: "c1", color: "green", classified_at: "2026-11-05T09:20:00-03:00" }),
    classification({ id: "c2", color: "yellow", classified_at: "2026-11-05T09:50:00-03:00", justification: "piora da dor" }) ] }));
  vi.mocked(api.suggestUrgencyColor).mockResolvedValue(suggestion());
  mount({ suggestDelayMs: 0 });
  expect(await screen.findByText("Cor vigente: amarelo (09:50, Ana Souza)")).not.toBeNull();
  expect(screen.getAllByRole("listitem").map((li) => li.textContent)).toEqual([
    expect.stringContaining("09:20 · verde"), expect.stringContaining("09:50 · amarelo") ]);
  expect(screen.getByRole("button", { name: "Reclassificar" })).not.toBeNull();
});
it("sinal fora do plausível: a frase do 18 no campo, sem enviar", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(encounter());
  mount({ suggestDelayMs: 0 });
  fireEvent.change(await screen.findByLabelText("Saturação (SpO2) (%)"), { target: { value: "140" } });
  fireEvent.click(screen.getByRole("button", { name: "Classificar" }));
  expect(screen.getAllByRole("alert").length).toBeGreaterThan(0);
  expect(api.classifyUrgency).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency/ClassificationForm.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `VitalSignsFields.tsx`: `omit` filtra `VITALS` antes do `map` (o 18 continua igual: o teste dele roda no Step 4).
- `ClassificationForm.tsx`: lê o encontro por `useQuery({ queryKey: [ URGENCY_DETAIL_KEY, encounterId ], queryFn: …, ...NOMINAL_QUERY })`; `EncounterSummary` no topo; "Queixa principal" (3–500, contador), `VitalSignsFields` com `omit={[ "pain_score" ]}`, "Dor (0 a 10)" à parte (inteiro); a sugestão por `useDebouncedValue` da chave (queixa + sinais + dor) e `suggestUrgencyColor({ encounter_id, complaint, vital_signs, pain_scale })` com `seq` (resposta velha ignorada); o `radiogroup` "Cor final" com as 5 cores na ordem, cada uma "<cor> · <maxWaitLabel(levels)>" (níveis da sugestão; sem eles, `DEFAULT_LEVELS`); a frase "Sugerida: <cor> — <critério>"; "Justificativa" (10–500) quando `justificationNeeded` ou depois de `justification_required`. Envio por `classificationBody`; recusas por `urgencyRefusals` no campo; sucesso chama `onDone(encounter)` e a mesa relê a fila (`queueRef.current.refresh()`).
- `UrgencyDesk.tsx`: `onClassify` (enfermagem) abre o formulário num `Panel` "Classificação de risco — <senha>"; foco no título; Escape/"Cancelar" fecha e limpa o detalhe do cache (`queryClient.removeQueries({ queryKey: [ URGENCY_DETAIL_KEY ] })`). Linha classificada ganha "Reclassificar".

- [ ] **Step 4: Rode e veja passar; regressão da escuta do 18; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency src/modules/attendance/VitalSignsFields.test.tsx src/modules/attendance/ScreeningForm.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/ClassificationForm.tsx src/modules/urgency/EncounterSummary.tsx src/modules/urgency/UrgencyDesk.tsx \
  src/modules/attendance/VitalSignsFields.tsx src/modules/urgency/ClassificationForm.test.tsx
$G commit -m "feat: classify urgency risk with the suggested color, criterion, justification and reclassification" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Local de chamada lembrado, chamada para classificação e atendimento, "Chamar de novo" e senha na UBS (F-32.5)

**Files:**
- Create: `src/modules/attendance/CallLocationPicker.tsx`
- Modify: `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/attendance/UnitQueue.tsx`, `src/modules/attendance/CheckIn.tsx`
- Test: `src/modules/attendance/CallLocationPicker.test.tsx`, `src/modules/attendance/UnitQueue.waitingRoom.test.tsx`

**Interfaces:**
- Consumes: `listCallLocations`, `callForClassification`, `callAttendance`, `callNext`, `recallAttendance` (Task 1); `canUseWaitingRoom`, `readCallLocation`, `writeCallLocation`, `CALL_LOCATIONS_KEY`, `urgencyError` (Task 2).
- Produces: `useCallLocation(userId, unitId, enabled): { locations: CallLocation[]; required: boolean; selected: string | null; select(id: string | null): void; problem: string | null; setProblem(p: string | null): void }` e `CallLocationPicker({ state, label? })` em `CallLocationPicker.tsx` — `select` grava em `sessionStorage`; id lembrado que saiu da lista (local desativado) vira `null`; `required` vem de `location_required` (D6).

- [ ] **Step 1: Escreva os testes**

`src/modules/attendance/CallLocationPicker.test.tsx`:

```tsx
it("lembra o local na sessão por unidade; local desativado some da escolha", async () => {
  vi.mocked(api.listCallLocations).mockResolvedValue({ items: [ { id: "loc1", health_unit_id: "upa1", name: "Consultório 1", active: true },
    { id: "loc2", health_unit_id: "upa1", name: "Sala de medicação", active: false } ], location_required: true });
  writeCallLocation("u-1", "upa1", "loc2");
  mountPicker();
  const select = await screen.findByLabelText("Local da chamada") as HTMLSelectElement;
  expect(select.value).toBe("");
  expect([ ...select.options ].map((o) => o.text)).toEqual([ "— escolha —", "Consultório 1" ]);
  fireEvent.change(select, { target: { value: "loc1" } });
  expect(readCallLocation("u-1", "upa1")).toBe("loc1");
});
it("sem waiting_room_display não busca nem aparece", () => {
  mountPicker({ enabled: false });
  expect(screen.queryByLabelText("Local da chamada")).toBeNull();
  expect(api.listCallLocations).not.toHaveBeenCalled();
});
```

`src/modules/attendance/UnitQueue.waitingRoom.test.tsx` (UBS com `waiting_room_display`, profissional com vínculo):

```tsx
it("UBS com TV: senha na fila, 'Chamar próximo' leva o local, 'Chamar de novo' em quem foi chamado", async () => {
  vi.mocked(api.listUnitQueue).mockResolvedValue({ waiting: [ queueRow({ id: "a1", ticket: "A-007" }) ],
    in_care: [ queueRow({ id: "a2", ticket: "A-006", called_at: "2026-11-05T09:58:00-03:00" }) ] });
  vi.mocked(api.listCallLocations).mockResolvedValue({ items: [ { id: "loc1", health_unit_id: "ubs1", name: "Consultório 1", active: true } ], location_required: true });
  vi.mocked(api.callNext).mockResolvedValue({ attendance: attendanceFixture({ id: "a1" }) });
  vi.mocked(api.recallAttendance).mockResolvedValue({ call_id: "c9" });
  mountUnitQueue(UBS);
  expect(await screen.findByText("A-007")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Local da chamada"), { target: { value: "loc1" } });
  fireEvent.click(screen.getByRole("button", { name: "Chamar próximo" }));
  await waitFor(() => expect(vi.mocked(api.callNext).mock.calls[0]).toEqual([ "ubs1", "loc1" ]));
  fireEvent.click(screen.getByRole("button", { name: "Chamar de novo A-006" }));
  await waitFor(() => expect(vi.mocked(api.recallAttendance).mock.calls[0]).toEqual([ "a2", "loc1" ]));
  expect(await screen.findByText("A-006 chamada de novo em Consultório 1")).not.toBeNull();
});
it("local obrigatório: sem escolha não chama; call_location_required do api põe a frase e o foco no seletor", async () => {
  vi.mocked(api.listUnitQueue).mockResolvedValue({ waiting: [ queueRow({ id: "a1", ticket: "A-007" }) ], in_care: [] });
  vi.mocked(api.listCallLocations).mockResolvedValue({ items: [ { id: "loc1", health_unit_id: "ubs1", name: "Consultório 1", active: true } ], location_required: true });
  mountUnitQueue(UBS);
  fireEvent.click(await screen.findByRole("button", { name: "Chamar próximo" }));
  expect(screen.getByText("escolha o local da chamada — a unidade tem TV na sala de espera")).not.toBeNull();
  expect(api.callNext).not.toHaveBeenCalled();
  expect(document.activeElement).toBe(screen.getByLabelText("Local da chamada"));
});
it("sem waiting_room_display: a fila e a chamada do 13 iguais a hoje", async () => {
  vi.mocked(api.listUnitQueue).mockResolvedValue({ waiting: [ queueRow({ id: "a1" }) ], in_care: [] });
  vi.mocked(api.callNext).mockResolvedValue({ attendance: attendanceFixture({ id: "a1" }) });
  mountUnitQueue(UBS, [ "clinical_record" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Chamar próximo" }));
  await waitFor(() => expect(vi.mocked(api.callNext).mock.calls[0]).toEqual([ "ubs1" ]));
  expect(screen.queryByLabelText("Local da chamada")).toBeNull();
});
it("check-in da UBS com senha mostra o número para entregar", async () => {
  vi.mocked(api.checkIn).mockResolvedValue({ attendance: attendanceFixture({ ticket: "A-008" }), verified: false });
  doCheckIn(UBS);
  expect(await screen.findByText("Senha A-008 — entregue à pessoa")).not.toBeNull();
});
```

E no `UrgencyQueue` (no mesmo arquivo de teste da Task 6, novo `describe("chamada")`): a enfermagem tem "Chamar para classificação U-042" em "aguardando classificação" → `callForClassification("en1", "loc1")`; o médico tem "Chamar U-040" em "por cor" e "Chamar próximo" no topo → `callNext("upa1", "loc1")`; o médico **não** tem botão de chamada em "aguardando classificação"; 422 `not_classified` mostra a frase; "Chamar de novo U-038" em "em atendimento" → `recallAttendance("at3", "loc1")`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/attendance/CallLocationPicker.test.tsx src/modules/attendance/UnitQueue.waitingRoom.test.tsx src/modules/urgency/UrgencyQueue.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `CallLocationPicker.tsx`: `useQuery({ queryKey: [ CALL_LOCATIONS_KEY, unitId ], queryFn: () => listCallLocations(unitId), enabled })`; só os `active`; `<select aria-label="Local da chamada">` com "— escolha —"; o `problem` sob o seletor (`role="alert"`) e `ref` para o foco. Sem locais e `required`: a frase "cadastre os locais de chamada em Urgência → Senhas, locais e TVs".
- `UnitQueue.tsx`: com `canUseWaitingRoom(user)`, o seletor ao lado de "Chamar próximo"; coluna "Senha" quando alguma linha tem `ticket`; `onCallNext`/`onCall` conferem `required && !selected` antes de chamar (frase + foco) e passam `selected ?? undefined`; 422 `call_location_required` idem; "Chamar de novo <senha>" nas linhas de "Em atendimento" (`recallAttendance(row.id, selected)`), com a frase de sucesso "<senha> chamada de novo em <local>". Sem o interruptor, nada muda (o teste prova a chamada sem segundo argumento).
- `CheckIn.tsx`: com `attendance.ticket`, a linha "Senha <ticket> — entregue à pessoa" abaixo de "Atendimento iniciado" (no check-in normal e no por exceção).
- `UrgencyDesk.tsx`/`UrgencyQueue`: o mesmo seletor no topo da fila da UPA; as `actions` de chamada conforme os papéis (enfermagem: classificação; médico: atendimento e "Chamar próximo", que no api segue a ordem por cor); `already_called` e `queue_empty` só releem.

- [ ] **Step 4: Rode e veja passar; regressão do 13; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/attendance src/modules/urgency src/modules/Attendance*.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/attendance/CallLocationPicker.tsx src/modules/attendance/CallLocationPicker.test.tsx src/modules/attendance/UnitQueue.tsx \
  src/modules/attendance/CheckIn.tsx src/modules/attendance/UnitQueue.waitingRoom.test.tsx src/modules/urgency/UrgencyDesk.tsx \
  src/modules/urgency/UrgencyQueue.tsx src/modules/urgency/UrgencyQueue.test.tsx
$G commit -m "feat: call by ticket with a remembered call location and recall, at the UPA and the UBS" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Atendimento médico pela consulta `urgency`, desfecho com os 6 tipos e contrarreferência (F-32.4)

**Files:**
- Create: `src/modules/urgency/UrgencyCare.tsx`, `src/modules/urgency/OutcomeForm.tsx`
- Modify: `src/modules/urgency/UrgencyDesk.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/attendance/UnitQueue.tsx`
- Test: `src/modules/urgency/UrgencyCare.test.tsx`

**Interfaces:**
- Consumes: `startConsultation`, `finalizeConsultation`, `recordUrgencyOutcome`, `getUrgencyEncounter` (Task 1); `OUTCOME_KINDS`, `OUTCOME_KIND_LABEL`, `outcomeProblems`, `outcomeBody`, `urgencyRefusals`, `URGENCY_ERRORS` (Task 2); `ConsultationWorkspace` (19a).
- Produces: `ConsultationWorkspace` ganha `urgency?: boolean` (só "Iniciar atendimento de urgência" → `open({ kind: "urgency" })`; sem o painel de procedimentos do 22); `ConsultationEditor` ganha `urgency?: boolean` (sem `OutcomeFields`; finaliza com `finalizeConsultation(id, null)`; a frase "O desfecho da urgência vem depois de finalizar."); `UrgencyCare({ row: UrgencyInCareRow, unit, units, onClosed(): void, onGoToSecurity? })`; `OutcomeForm({ encounter, doctor: boolean, onRecorded(e: UrgencyEncounter): void })`.

- [ ] **Step 1: Escreva os testes**

```tsx
it("da fila: 'Atender' abre a consulta de urgência, sem desfecho do 13 e sem procedimentos do 22", async () => {
  vi.mocked(api.getAttendanceRecord).mockResolvedValue(recordFixture());
  vi.mocked(api.getConsultationOptions).mockResolvedValue({ ...optionsFixture(), kinds: [ "urgency" ] });
  vi.mocked(api.startConsultation).mockResolvedValue(consultationFixture({ id: "co1", kind: "urgency", status: "draft" }));
  mountCare(inCareRow({ consultation_id: null }));
  fireEvent.click(await screen.findByRole("button", { name: "Iniciar atendimento de urgência" }));
  await waitFor(() => expect(vi.mocked(api.startConsultation).mock.calls[0]).toEqual([ "at3", { kind: "urgency" } ]));
  expect(screen.queryByRole("button", { name: "Iniciar consulta" })).toBeNull();
  expect(screen.queryByRole("group", { name: /Desfecho do atendimento/ })).toBeNull();
  expect(screen.getByText("O desfecho da urgência vem depois de finalizar.")).not.toBeNull();
  expect(screen.queryByRole("region", { name: "Procedimentos realizados" })).toBeNull();
});
it("finalizar manda {} e abre o desfecho; alta com retorno exige prazo e motivo", async () => {
  vi.mocked(api.finalizeConsultation).mockResolvedValue(consultationFixture({ id: "co1", kind: "urgency", status: "finalized" }));
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(encounter({ state: "in_care", consultation_id: "co1", consultation_status: "finalized" }));
  vi.mocked(api.recordUrgencyOutcome).mockResolvedValue(encounter({ state: "closed" }));
  await finalizeDraft();
  expect(vi.mocked(api.finalizeConsultation).mock.calls[0]).toEqual([ "co1", null ]);
  const form = await screen.findByRole("group", { name: "Desfecho da urgência" });
  fireEvent.click(within(form).getByRole("radio", { name: "Alta com retorno na UBS" }));
  fireEvent.click(within(form).getByRole("button", { name: "Registrar desfecho" }));
  expect(within(form).getByText("o prazo do retorno é de 1 a 90 dias")).not.toBeNull();
  fireEvent.change(within(form).getByLabelText("Procurar a UBS em até (dias)"), { target: { value: "7" } });
  fireEvent.change(within(form).getByLabelText("Motivo do retorno"), { target: { value: "reavaliar a pressão arterial" } });
  fireEvent.click(within(form).getByRole("button", { name: "Registrar desfecho" }));
  await waitFor(() => expect(vi.mocked(api.recordUrgencyOutcome).mock.calls[0]).toEqual([ "en3",
    { kind: "discharged_to_primary_care", return_guidance: { days: 7, reason: "reavaliar a pressão arterial" } } ]));
  expect(await screen.findByText("Atendimento encerrado: alta com retorno na UBS. A UBS de referência recebe o aviso.")).not.toBeNull();
});
it("os 6 tipos na ordem; transferência pede destino; pedido de vaga é opcional", async () => {
  mountOutcome(encounter({ state: "in_care", consultation_id: "co1", consultation_status: "finalized" }), { doctor: true });
  expect(screen.getAllByRole("radio").map((r) => (r as HTMLInputElement).labels?.[0]?.textContent)).toEqual([
    "Alta", "Alta com retorno na UBS", "Transferência", "Saiu sem atendimento", "Saiu contra orientação médica", "Óbito" ]);
  fireEvent.click(screen.getByRole("radio", { name: "Transferência" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar desfecho" }));
  expect(screen.getByText("informe o destino da transferência")).not.toBeNull();
  expect(screen.getByLabelText("Pedido de vaga (opcional)")).not.toBeNull();
});
it("antes da consulta finalizada só 'Saiu sem atendimento'; recepção e enfermagem só veem esse", async () => {
  mountOutcome(encounter({ state: "classified" }), { doctor: false });
  expect(screen.getAllByRole("radio").length).toBe(1);
  expect(screen.getByRole("radio", { name: "Saiu sem atendimento" })).not.toBeNull();
});
it("consultation_not_finalized do api mostra a frase; o 13 numa UPA recusa encerrar sem desfecho", async () => {
  vi.mocked(api.recordUrgencyOutcome).mockRejectedValue(new ApiError(422, { error: "consultation_not_finalized" }, "x"));
  mountOutcome(encounter({ state: "in_care", consultation_id: "co1", consultation_status: "finalized" }), { doctor: true });
  fireEvent.click(screen.getByRole("radio", { name: "Alta" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar desfecho" }));
  expect(await screen.findByText("finalize a consulta antes deste desfecho")).not.toBeNull();
});
```

E em `UnitQueue.waitingRoom.test.tsx`, um caso a mais: `closeAttendance` rejeitado com 422 `urgency_outcome_required` mostra "numa UPA, o atendimento termina pelo desfecho da urgência" (o `ClosePanel` passa a traduzir o código por `urgencyError` antes do `attendanceError`).

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency/UrgencyCare.test.tsx src/modules/attendance/UnitQueue.waitingRoom.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ConsultationWorkspace.tsx`: prop `urgency`; com ela, um botão só, "Iniciar atendimento de urgência" (os de consulta comum e odontológica somem, e `noKindReason` vira "seu CBO não atende urgência nesta unidade" quando `options.kinds` não traz `urgency`); `proceduresOn && !props.urgency`; passa `urgency` ao `ConsultationEditor`. Documentos do 19, exames do 21, sugestão de notificação do 25, adendos e assinatura seguem iguais.
- `ConsultationEditor.tsx`: com `urgency`, sem `OutcomeFields` nem `outcomeIssue`, a frase fixa, e `finalizeConsultation(consultation.id, null)`.
- `OutcomeForm.tsx`: `<fieldset aria-label="Desfecho da urgência">`; os tipos permitidos = `doctor && consultation_status === "finalized"` → os 6; senão só `left_without_care` (D14); campos por tipo: transferência → "Destino" (3–200) e "Pedido de vaga (opcional)" (até 2000); alta com retorno → "Procurar a UBS em até (dias)" (1–90) e "Motivo do retorno" (10–500), com a nota "O motivo vai para a equipe da UBS do paciente e para o histórico do cidadão."; óbito e "contra orientação" → só a confirmação por `ConfirmButton` ("Registrar desfecho" → "O desfecho não pode ser alterado. Confirmar?"). Sucesso: frase "Atendimento encerrado: <rótulo em minúscula>." (mais "A UBS de referência recebe o aviso." no retorno) e `onRecorded`.
- `UrgencyCare.tsx`: `Panel` "Atendimento — <senha>" com `EncounterSummary` (cor vigente e o critério), `ConsultationWorkspace` com `urgency` e, com a consulta finalizada (ou já finalizada no `in_care` da fila), o `OutcomeForm`. Fecha o painel e relê a fila depois do desfecho.
- `UrgencyDesk.tsx`: médico chamado (`onCallCare`/`callNext`) abre `UrgencyCare` para a linha chamada; "Atender" em "em atendimento" reabre; "Saiu sem atendimento" (todos os papéis) abre o `OutcomeForm` com `doctor: false`.
- `UnitQueue.tsx`: `ClosePanel` traduz `urgency_outcome_required` (`urgencyError`) antes do `attendanceError`.

- [ ] **Step 4: Rode e veja passar; regressão da consulta; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency src/modules/consultation src/modules/attendance && cd $W && npx tsc --noEmit`
Expected: PASS (nenhum teste existente liga `urgency`).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/UrgencyCare.tsx src/modules/urgency/OutcomeForm.tsx src/modules/urgency/UrgencyDesk.tsx \
  src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationEditor.tsx src/modules/attendance/UnitQueue.tsx \
  src/modules/urgency/UrgencyCare.test.tsx src/modules/attendance/UnitQueue.waitingRoom.test.tsx
$G commit -m "feat: open the urgency consultation from the queue and close it with one of six outcomes" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Identificar o paciente não identificado, com confirmação forte da fusão (F-32.2)

**Files:**
- Create: `src/modules/urgency/IdentifyPatient.tsx`
- Modify: `src/modules/urgency/UrgencyDesk.tsx`
- Test: `src/modules/urgency/IdentifyPatient.test.tsx`

**Interfaces:**
- Consumes: `lookupUrgencyPerson`, `identifyUrgencyPatient`, `getUrgencyEncounter` (Task 1); `newPatientProblem`, `urgencyError`, `urgencyRefusals`, `URGENCY_DETAIL_KEY` (Task 2); `ConfirmButton`.
- Produces: `IdentifyPatient({ encounterId, onIdentified(e: UrgencyEncounter): void, onCancel(): void })`. Só aparece em linha com `patient.unidentified`, para recepção, enfermagem e médico da UPA (D15).

- [ ] **Step 1: Escreva os testes**

```tsx
const provisional = encounter({ patient: patientRef({ patient_id: "prov1", name: "Não identificado", unidentified: true,
  description: "homem, cerca de 40 anos, achado na rodoviária" }) });
it("paciente existente: lado a lado, caixa obrigatória e confirmação em dois passos", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(provisional);
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: patientRef({ patient_id: "pa7", name: "Carlos Dias" }), candidates: [] });
  vi.mocked(api.identifyUrgencyPatient).mockResolvedValue(encounter({ patient: patientRef({ patient_id: "pa7", name: "Carlos Dias" }) }));
  mount();
  expect(await screen.findByText("homem, cerca de 40 anos, achado na rodoviária")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "52998224725" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  fireEvent.click(await screen.findByRole("button", { name: "É Carlos Dias" }));
  const compare = screen.getByRole("group", { name: "Confira antes de ligar" });
  expect(within(compare).getByText("Carlos Dias")).not.toBeNull();
  expect(within(compare).getByText("homem, cerca de 40 anos, achado na rodoviária")).not.toBeNull();
  const confirm = screen.getByRole("button", { name: "Ligar ao paciente existente" });
  expect((confirm as HTMLButtonElement).disabled).toBe(true);
  fireEvent.click(screen.getByLabelText("Confirmo que é a mesma pessoa. O atendimento, a consulta, os documentos e os exames passam para Carlos Dias, e isso não pode ser desfeito."));
  fireEvent.click(screen.getByRole("button", { name: "Ligar ao paciente existente" }));
  expect(api.identifyUrgencyPatient).not.toHaveBeenCalled();
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(vi.mocked(api.identifyUrgencyPatient).mock.calls[0]).toEqual([ "en1", { patient_id: "pa7" } ]));
  expect(await screen.findByText("Identificado: Carlos Dias.")).not.toBeNull();
});
it("paciente novo: o provisório recebe os dados, sem fusão nem caixa", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValue(provisional);
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: null, candidates: [] });
  vi.mocked(api.identifyUrgencyPatient).mockResolvedValue(encounter());
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Cadastrar como paciente novo" }));
  fillNewPatient({ full_name: "Carlos Dias", birth_date: "1986-03-02", sex: "male", cpf: "52998224725" });
  fireEvent.click(screen.getByRole("button", { name: "Identificar" }));
  await waitFor(() => expect(vi.mocked(api.identifyUrgencyPatient).mock.calls[0][1]).toMatchObject({ new_patient: { full_name: "Carlos Dias" } }));
  expect(screen.queryByText(/não pode ser desfeito/)).toBeNull();
});
it("already_identified: a frase, relê o encontro e mostra quem é; sem segunda tentativa", async () => {
  vi.mocked(api.getUrgencyEncounter).mockResolvedValueOnce(provisional)
    .mockResolvedValueOnce(encounter({ patient: patientRef({ patient_id: "pa8", name: "Pedro Alves" }) }));
  vi.mocked(api.lookupUrgencyPerson).mockResolvedValue({ patient: patientRef({ patient_id: "pa7", name: "Carlos Dias" }), candidates: [] });
  vi.mocked(api.identifyUrgencyPatient).mockRejectedValue(new ApiError(409, { error: "already_identified" }, "x"));
  mount();
  await pickAndConfirm("Carlos Dias");
  expect(await screen.findByText("este paciente já foi identificado por outra pessoa")).not.toBeNull();
  expect(await screen.findByText("Agora é: Pedro Alves.")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Ligar ao paciente existente" })).toBeNull();
  expect(api.identifyUrgencyPatient).toHaveBeenCalledTimes(1);
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency/IdentifyPatient.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `IdentifyPatient.tsx`: lê o encontro (`NOMINAL_QUERY`); mostra a descrição e a chegada; a busca por CPF ou CNS como na recepção (Task 5, `POST`); achado → "É <nome>" abre o `<fieldset aria-label="Confira antes de ligar">` com as duas colunas ("Quem está na UPA": descrição, senha, chegada; "Paciente escolhido": nome, CNS mascarado) + a caixa (texto exato do teste, com o nome) + `ConfirmButton label="Ligar ao paciente existente" question="Ligar agora? Não dá para desfazer."` desabilitado sem a caixa (`disabledReason="marque a confirmação"`); ninguém → "Cadastrar como paciente novo" com os campos do paciente novo (`newPatientProblem`) e "Identificar" sem caixa. `busy` + `useRef`; 409 `already_identified` → frase, `refetch` do encontro, "Agora é: <nome>." e o formulário some; outras recusas por `urgencyRefusals`. Sucesso: "Identificado: <nome>.", `onIdentified`, e a mesa relê a fila.
- `UrgencyDesk.tsx`: "Identificar" nas linhas com `patient.unidentified` (as três faixas) → `Panel` "Identificar paciente — <senha>".

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/urgency && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/urgency/IdentifyPatient.tsx src/modules/urgency/UrgencyDesk.tsx src/modules/urgency/IdentifyPatient.test.tsx
$G commit -m "feat: identify an unidentified urgency patient once, with a strong merge confirmation" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Senhas, locais e TVs — locais de chamada, TVs com código de pareamento e revogação, prefixo da senha (F-32.5, F-32.1)

**Files:**
- Create: `src/modules/waitingRoom/CallLocationsPanel.tsx`, `src/modules/waitingRoom/DisplaysPanel.tsx`
- Modify: `src/modules/WaitingRoomSettings.tsx`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/WaitingRoomSettings.test.tsx`, `src/modules/attendance/UnitForm.ticket.test.tsx`

**Interfaces:**
- Consumes: `listActiveUnits`, `listCallLocations`, `createCallLocation`, `updateCallLocation`, `listWaitingRoomDisplays`, `createWaitingRoomDisplay`, `updateWaitingRoomDisplay`, `createPairingCode`, `revokeWaitingRoomDisplay`, `updateUnit`/`createUnit` com `ticket_prefix` (Task 1); `locationNameProblem`, `ticketPrefixProblem`, `urgencyError`, `CALL_LOCATIONS_KEY`, `DISPLAYS_KEY` (Task 2); `SensitiveAction` (step-up, como em `CapsSettings`); `withoutCaps`.
- Produces: `WaitingRoomSettings({ onGoToSecurity })` com o seletor "Unidade" (`withoutCaps(listActiveUnits())`), `CallLocationsPanel({ unit })` e `DisplaysPanel({ unit, onGoToSecurity })`; `UnitFormValue` ganha `ticketPrefix?: string`; `UnitForm` e `Units` ganham `waitingRoomOn?: boolean`.

- [ ] **Step 1: Escreva os testes**

`src/modules/WaitingRoomSettings.test.tsx` (sessão `urgencyUser(["municipal_admin"], ["waiting_room_display"])`, step-up aberto):

```tsx
it("locais: criar, renomear e desativar, com step-up", async () => {
  vi.mocked(api.listCallLocations).mockResolvedValue({ items: [], location_required: false });
  vi.mocked(api.createCallLocation).mockResolvedValue({ id: "loc1", health_unit_id: "upa1", name: "Consultório 1", active: true });
  mount();
  fireEvent.change(await screen.findByLabelText("Unidade"), { target: { value: "upa1" } });
  fireEvent.change(await screen.findByLabelText("Nome do local"), { target: { value: "Consultório 1" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar local" }));
  await waitFor(() => expect(vi.mocked(api.createCallLocation).mock.calls[0][0]).toEqual({ health_unit_id: "upa1", name: "Consultório 1", active: true }));
});
it("TV: cadastrar, gerar o código de 6 dígitos com validade e as instruções; revogar com confirmação", async () => {
  vi.mocked(api.listWaitingRoomDisplays).mockResolvedValue([ { id: "d1", health_unit_id: "upa1", name: "TV da recepção", active: true,
    show_wait_times: true, paired: true, last_seen_at: "2026-11-05T09:59:00-03:00" } ]);
  vi.mocked(api.createPairingCode).mockResolvedValue({ code: "482913", expires_at: "2026-11-05T10:10:00-03:00" });
  vi.mocked(api.revokeWaitingRoomDisplay).mockResolvedValue({ id: "d1", health_unit_id: "upa1", name: "TV da recepção", active: false,
    show_wait_times: true, paired: false, last_seen_at: null });
  mount();
  fireEvent.change(await screen.findByLabelText("Unidade"), { target: { value: "upa1" } });
  expect(await screen.findByText("pareada · vista às 09:59")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Gerar código para TV da recepção" }));
  expect(await screen.findByText("482913")).not.toBeNull();
  expect(screen.getByText("Vale até 10:10. Na TV, abra <endereço da cidade>/dashboard/#/tv e digite o código.")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Revogar TV da recepção" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.revokeWaitingRoomDisplay).toHaveBeenCalledWith("d1"));
  expect(await screen.findByText("TV da recepção revogada: a tela volta a pedir código.")).not.toBeNull();
});
it("'Mostrar o tempo de espera por cor' é por TV", async () => {
  vi.mocked(api.listWaitingRoomDisplays).mockResolvedValue([ { id: "d1", health_unit_id: "upa1", name: "TV", active: true, show_wait_times: true,
    paired: false, last_seen_at: null } ]);
  vi.mocked(api.updateWaitingRoomDisplay).mockResolvedValue({ id: "d1", health_unit_id: "upa1", name: "TV", active: true, show_wait_times: false,
    paired: false, last_seen_at: null });
  mount();
  fireEvent.change(await screen.findByLabelText("Unidade"), { target: { value: "upa1" } });
  fireEvent.click(await screen.findByLabelText("Mostrar o tempo de espera por cor em TV"));
  await waitFor(() => expect(vi.mocked(api.updateWaitingRoomDisplay).mock.calls[0]).toEqual([ "d1", { name: "TV", active: true, show_wait_times: false } ]));
});
it("CAPS não aparece entre as unidades; o código nunca vai para storage", async () => {
  vi.mocked(api.listActiveUnits).mockResolvedValue([ UPA, UBS, { id: "caps1", name: "CAPS II", kind: "caps" } ]);
  mount();
  const select = await screen.findByLabelText("Unidade") as HTMLSelectElement;
  expect([ ...select.options ].map((o) => o.text)).not.toContain("CAPS II");
});
```

`src/modules/attendance/UnitForm.ticket.test.tsx`: com `waitingRoomOn`, o campo "Prefixo da senha" aparece (placeholder `U` para `upa`, `A` para os demais), `"u1"` dá "o prefixo tem de 1 a 3 letras maiúsculas, sem acento" sem salvar, `"UB"` vai em `onSave` como `ticketPrefix: "UB"`; sem `waitingRoomOn`, o campo não aparece e `ticketPrefix` não vai.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/WaitingRoomSettings.test.tsx src/modules/attendance/UnitForm.ticket.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `WaitingRoomSettings.tsx`: portão (`canUseWaitingRoom` e `municipal_admin`; senão `EmptyState` com `WAITING_ROOM_DISABLED` ou "seu papel não permite configurar a sala de espera"); "Unidade" (UPA e UBS, sem CAPS); os dois painéis com `key={unit.id}`.
- `CallLocationsPanel.tsx`: `DataTable` (Nome, Situação, Ações "Renomear"/"Desativar"/"Ativar"); formulário "Nome do local" + "Adicionar local"; toda escrita dentro do `SensitiveAction` (o step-up repete a ação uma vez só).
- `DisplaysPanel.tsx`: `DataTable` (Nome, Situação "pareada · vista às HH:MM" | "aguardando pareamento" | "revogada", "Mostrar o tempo de espera por cor em <nome>" (checkbox), Ações "Gerar código para <nome>", "Revogar <nome>" (`ConfirmButton`, `question="Revogar a TV? Ela volta a pedir código."`)); "Nome da TV" + "Cadastrar TV"; o código aparece grande (mono, `fontSize: 32`) só na memória do componente, com a validade e a instrução; some ao gerar outro ou sair da tela.
- `UnitForm.tsx`/`Units.tsx`/`Attendance.tsx`: `waitingRoomOn={canUseWaitingRoom(user)}`; campo "Prefixo da senha" (`ticketPrefixProblem`), que vai em `updateUnit`/`createUnit` pelo objeto do último parâmetro (`{ ticket_prefix }`) só com o interruptor.

- [ ] **Step 4: Rode e veja passar; regressão do cadastro de unidade; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/WaitingRoomSettings.test.tsx src/modules/attendance/UnitForm*.test.tsx src/modules/attendance/Units*.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/WaitingRoomSettings.tsx src/modules/waitingRoom/CallLocationsPanel.tsx src/modules/waitingRoom/DisplaysPanel.tsx \
  src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/Attendance.tsx src/modules/WaitingRoomSettings.test.tsx \
  src/modules/attendance/UnitForm.ticket.test.tsx
$G commit -m "feat: manage call locations, waiting room TVs with pairing codes and the ticket prefix" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: A TV da sala de espera — pareamento, chamadas, tempos por cor, polling de 5 s, tela cheia (F-32.5)

**Files:**
- Create: `src/modules/waitingRoom/TvPairing.tsx`, `src/modules/waitingRoom/TvBoard.tsx`, `src/modules/waitingRoom/tv.css`
- Modify: `src/modules/WaitingRoomTv.tsx`
- Test: `src/modules/WaitingRoomTv.test.tsx`

**Interfaces:**
- Consumes: `pairWaitingRoomDisplay`, `fetchWaitingRoomBoard` (Task 1); `usePolling`, `WAITING_ROOM_POLL_MS`, `PRIORITY_PHRASE`, `URGENCY_COLOR_LABEL`, `URGENCY_COLOR_TONE`, `pairingCodeProblem`, `urgencyError` (Task 2).
- Produces: `WaitingRoomTv({ now?, storage? = window.localStorage })` — sem token: `TvPairing`; com token: `TvBoard`; `DISPLAY_TOKEN_KEY = "waitingRoom.displayToken"`; `TvBoard({ token, onRevoked(): void, now? })`; `TvPairing({ onPaired(token: string): void })`.

- [ ] **Step 1: Escreva os testes**

```tsx
afterEach(() => { cleanup(); vi.useRealTimers(); localStorage.clear(); });
it("primeiro acesso pede o código; 6 dígitos; pareia e guarda só o token", async () => {
  vi.mocked(api.pairWaitingRoomDisplay).mockResolvedValue({ display_token: "tok-abc" });
  vi.mocked(api.fetchWaitingRoomBoard).mockResolvedValue(board());
  render(<WaitingRoomTv now={() => NOW32} />);
  fireEvent.change(screen.getByLabelText("Código de pareamento"), { target: { value: "48291" } });
  fireEvent.click(screen.getByRole("button", { name: "Parear esta TV" }));
  expect(screen.getByText("o código tem 6 dígitos")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Código de pareamento"), { target: { value: "482913" } });
  fireEvent.click(screen.getByRole("button", { name: "Parear esta TV" }));
  expect(await screen.findByText("U-042")).not.toBeNull();
  expect(localStorage.getItem("waitingRoom.displayToken")).toBe("tok-abc");
  expect(Object.keys(localStorage)).toEqual([ "waitingRoom.displayToken" ]);
});
it("código inválido ou vencido: a frase, sem guardar nada", async () => {
  vi.mocked(api.pairWaitingRoomDisplay).mockRejectedValue(new ApiError(422, { error: "invalid_pairing_code" }, "x"));
  render(<WaitingRoomTv />);
  fireEvent.change(screen.getByLabelText("Código de pareamento"), { target: { value: "000000" } });
  fireEvent.click(screen.getByRole("button", { name: "Parear esta TV" }));
  expect(await screen.findByText("código inválido ou vencido — peça um código novo à administração")).not.toBeNull();
  expect(localStorage.getItem("waitingRoom.displayToken")).toBeNull();
});
it("board lê campo a campo: chamada atual em destaque, as anteriores, tempos por cor e a frase da prioridade", async () => {
  localStorage.setItem("waitingRoom.displayToken", "tok-abc");
  const leaky = board({ calls: [ { ...board().calls[0], name: "Maria Lima", cpf: "52998224725", color: "red", complaint: "dor no peito" } as never,
    board().calls[1] ] });
  vi.mocked(api.fetchWaitingRoomBoard).mockResolvedValue(leaky);
  render(<WaitingRoomTv now={() => NOW32} />);
  const current = await screen.findByRole("region", { name: "Chamada agora" });
  expect(within(current).getByText("U-042")).not.toBeNull();
  expect(within(current).getByText("Consultório 2")).not.toBeNull();
  expect(within(screen.getByRole("region", { name: "Chamadas anteriores" })).getByText("U-041")).not.toBeNull();
  expect(screen.getByText("Classificação de risco")).not.toBeNull();
  const waits = screen.getByRole("region", { name: "Tempo de espera agora" });
  expect(within(waits).getByText("amarelo")).not.toBeNull();
  expect(within(waits).getByText("cerca de 34 min")).not.toBeNull();
  expect(within(waits).getByText("poucas pessoas aguardando")).not.toBeNull();
  expect(screen.getByText("Quem corre mais risco é atendido primeiro, mesmo que tenha chegado depois.")).not.toBeNull();
  expect(document.body.textContent).not.toMatch(/Maria|52998224725|dor no peito/);
});
it("sem tempos (unidade não mostra): a seção some; sem chamadas: 'Aguarde ser chamado pela sua senha.'", async () => {
  localStorage.setItem("waitingRoom.displayToken", "tok-abc");
  vi.mocked(api.fetchWaitingRoomBoard).mockResolvedValue(board({ calls: [], wait_times: null }));
  render(<WaitingRoomTv />);
  expect(await screen.findByText("Aguarde ser chamado pela sua senha.")).not.toBeNull();
  expect(screen.queryByRole("region", { name: "Tempo de espera agora" })).toBeNull();
});
describe("polling e falhas", () => {
  beforeEach(() => vi.useFakeTimers());
  it("relê a cada 5 s; sem rede mantém a última tela com o aviso; revogada apaga o token e volta ao pareamento", async () => {
    localStorage.setItem("waitingRoom.displayToken", "tok-abc");
    vi.mocked(api.fetchWaitingRoomBoard).mockResolvedValueOnce(board()).mockRejectedValueOnce(new TypeError("Failed to fetch"))
      .mockRejectedValueOnce(new ApiError(401, { error: "display_revoked" }, "x"));
    const { unmount } = render(<WaitingRoomTv />);
    await vi.advanceTimersByTimeAsync(0);
    expect(screen.getByText("U-042")).not.toBeNull();
    await vi.advanceTimersByTimeAsync(5000);
    expect(screen.getByText("U-042")).not.toBeNull();
    expect(screen.getByText("sem conexão — tentando de novo")).not.toBeNull();
    await vi.advanceTimersByTimeAsync(5000);
    expect(screen.getByLabelText("Código de pareamento")).not.toBeNull();
    expect(localStorage.getItem("waitingRoom.displayToken")).toBeNull();
    await vi.advanceTimersByTimeAsync(30_000);
    expect(api.fetchWaitingRoomBoard).toHaveBeenCalledTimes(3);
    unmount();
    expect(vi.getTimerCount()).toBe(0);
  });
});
it("nenhuma chamada de sessão nem de outra rota do dashboard", async () => {
  localStorage.setItem("waitingRoom.displayToken", "tok-abc");
  vi.mocked(api.fetchWaitingRoomBoard).mockResolvedValue(board());
  render(<WaitingRoomTv />);
  await screen.findByText("U-042");
  expect(api.fetchCurrentSession).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/WaitingRoomTv.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `WaitingRoomTv.tsx`: lê o token de `storage` (`try/catch`); `TvPairing` ou `TvBoard`; `onRevoked` remove a chave e volta ao pareamento.
- `TvPairing.tsx`: `<form>` com "Código de pareamento" (`inputMode="numeric"`, `autoComplete="one-time-code"`), "Parear esta TV", a frase "Peça o código à administração da unidade (Urgência → Senhas, locais e TVs)."; `busy` + `useRef`.
- `TvBoard.tsx`: `usePolling(() => fetchWaitingRoomBoard(token), WAITING_ROOM_POLL_MS, token)`; 401 `display_revoked` → `onRevoked()` (o polling para porque o componente desmonta); outra falha → mantém o último board e mostra "sem conexão — tentando de novo". Monta **campo a campo**: `unit.name`, para cada chamada só `ticket`, `location`, `kind` ("Classificação de risco" | "Atendimento") e `called_at` (HH:MM); tempos só `color` (texto da cor + `Tag`) e `median_minutes` ("cerca de N min"; `"lt5_patients"` → "poucas pessoas aguardando"). A primeira chamada em `<section aria-label="Chamada agora">`, as demais (até 7) em "Chamadas anteriores"; a frase da prioridade sempre que houver tempos; rodapé "atualizado às HH:MM". Botão "Tela cheia" (`document.documentElement.requestFullscreen?.()`, some depois de entrar; tecla `F` também). `aria-live="polite"` na chamada atual.
- `tv.css`: tela inteira em `var(--bg)`, tipografia em `clamp()` pela largura (senha atual `clamp(72px, 12vw, 220px)`, local `clamp(40px, 6vw, 110px)`, anteriores `clamp(28px, 3.2vw, 60px)`), grade de duas colunas a partir de 1280 px e uma abaixo; só variáveis do tema; sem `outline: none`. Legível a 5 m em 1280×720 e 1920×1080 (conferido na Task 14).

- [ ] **Step 4: Rode e veja passar; guardas do tema; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/WaitingRoomTv.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/WaitingRoomTv.tsx src/modules/waitingRoom/TvPairing.tsx src/modules/waitingRoom/TvBoard.tsx src/modules/waitingRoom/tv.css \
  src/modules/WaitingRoomTv.test.tsx
$G commit -m "feat: add the full-screen waiting room TV with pairing, calls and wait times by color" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Painel da urgência, kind `urgency` na linha do tempo e retornos pedidos pela urgência na UBS (F-32.6, F-32.4)

**Files:**
- Create: `src/modules/urgency/CounterReferrals.tsx`
- Modify: `src/modules/UrgencyPanel.tsx`, `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/Attendance.tsx`
- Test: `src/modules/UrgencyPanel.test.tsx`, `src/lib/timeline.urgency.test.ts`, `src/modules/urgency/CounterReferrals.test.tsx`

**Interfaces:**
- Consumes: `getUrgencyDashboard`, `listCounterReferrals`, `listActiveUnits` (Task 1); `countLabel`, `timesLabel`, `colorLabel`, `outcomeLabel`, `URGENCY_DASHBOARD_KEY`, `COUNTER_REFERRALS_KEY`, `canUseUrgency`, `isUpa` (Task 2); `StatTile`, `DataTable`; `timelineKinds`, `timelineKindLabel`, `detailTarget`.
- Produces: `TimelineKind` ganha `"urgency"`; `TIMELINE_KINDS` ganha `{ key: "urgency", label: "Urgência" }`; `timelineKinds(…, psychosocial = false, urgency = false)` (sétimo parâmetro); `CounterReferrals({ unit })`.

- [ ] **Step 1: Escreva os testes**

`src/modules/UrgencyPanel.test.tsx` (`getUrgencyDashboard` → a forma da D9: `{ month: MONTH32, unit: null, by_color: [ { color: "red", encounters: "lt5", door_to_classification: "lt5", classification_to_doctor: "lt5", overdue_pct: "lt5" }, { color: "yellow", encounters: 212, door_to_classification: { median: 9, p90: 25 }, classification_to_doctor: { median: 41, p90: 95 }, overdue_pct: 18 } ], by_outcome: [ { kind: "discharged", count: 180 }, { kind: "death", count: "lt5" } ], left_without_care: { before_classification: 12, after_classification: "lt5" }, length_of_stay: { median: 150, p90: 320 }, total: 431 }`):

```tsx
it("só contagens por cor, com menos de 5, e nenhum nome", async () => {
  mount(urgencyUser([ "analyst" ]));
  const table = await screen.findByRole("table", { name: "Tempos por cor" });
  const yellow = within(table).getByRole("row", { name: /amarelo/ });
  expect(within(yellow).getByText("212")).not.toBeNull();
  expect(within(yellow).getByText("mediana 9 min · p90 25 min")).not.toBeNull();
  expect(within(yellow).getByText("18%")).not.toBeNull();
  expect(within(within(table).getByRole("row", { name: /vermelho/ })).getAllByText("menos de 5").length).toBe(4);
  expect(screen.getByText("Óbito")).not.toBeNull();
  expect(screen.getByText("Contagens de 1 a 4 aparecem como “menos de 5”.")).not.toBeNull();
  expect(vi.mocked(api.getUrgencyDashboard).mock.calls[0][0]).toEqual({ month: MONTH32, unitId: null });
});
it("filtro por UPA e por mês vão na consulta; só UPAs no seletor", async () => {
  vi.mocked(api.listActiveUnits).mockResolvedValue([ UPA, UBS ]);
  mount(urgencyUser([ "municipal_admin" ]));
  const select = await screen.findByLabelText("UPA do painel") as HTMLSelectElement;
  expect([ ...select.options ].map((o) => o.text)).toEqual([ "Todas as UPAs", "UPA Boqueirão" ]);
  fireEvent.change(select, { target: { value: "upa1" } });
  await waitFor(() => expect(vi.mocked(api.getUrgencyDashboard).mock.calls.at(-1)?.[0]).toEqual({ month: MONTH32, unitId: "upa1" }));
});
```

`src/lib/timeline.urgency.test.ts`: `timelineKindLabel("urgency")` é `"Urgência"`; `timelineKinds(false).some((k) => k.key === "urgency")` é `false`; `timelineKinds(false, false, false, false, false, false, true)` contém `urgency`; `detailTarget(entry({ kind: "urgency", detail_ref: { type: "consultation", id: "co1" } }), true)` é `{ type: "consultation", id: "co1" }` (a entrada abre a consulta do 19a pelo caminho que já existe).

`src/modules/urgency/CounterReferrals.test.tsx`:

```tsx
it("UBS: retornos pedidos pela urgência, sem o motivo na lista", async () => {
  vi.mocked(api.listCounterReferrals).mockResolvedValue([ { encounter_id: "en5", patient: patientRef(), upa: { name: "UPA Boqueirão" },
    closed_at: "2026-11-04T22:10:00-03:00", return_days: 7, due_on: "2026-11-11", reason: "reavaliar a pressão" } as never ]);
  mount(UBS);
  expect(await screen.findByText("Maria Lima")).not.toBeNull();
  expect(screen.getByText("procurar a UBS até 11/11/2026")).not.toBeNull();
  expect(document.body.textContent).not.toContain("reavaliar a pressão");
  expect(screen.getByText("O motivo está na linha do tempo do paciente.")).not.toBeNull();
});
it("só com urgency_care, e nunca na própria UPA", () => {
  mount(UPA);
  expect(screen.queryByRole("region", { name: "Retornos pedidos pela urgência" })).toBeNull();
  expect(api.listCounterReferrals).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/UrgencyPanel.test.tsx src/lib/timeline.urgency.test.ts src/modules/urgency/CounterReferrals.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `UrgencyPanel.tsx` (portão `canUseUrgency` e admin ou analista): "Mês" (`<input type="month">`, padrão o mês da cidade) e "UPA do painel" (todas ou uma, só `isUpa`); `StatTile`s "Atendimentos", "Saíram sem atendimento" (antes e depois da classificação), "Permanência" (`timesLabel`); `DataTable` "Tempos por cor" (Cor, Atendimentos, Porta → classificação, Classificação → médico, Fora do tempo máximo) na ordem das cores, e "Por desfecho" (`outcomeLabel`); tudo por `countLabel`/`timesLabel`; a nota do "menos de 5". `useQuery` com chave `[ URGENCY_DASHBOARD_KEY, month, unitId ]`, sem polling.
- `timeline.ts`: kind, rótulo, sétimo parâmetro e o comentário "Módulo 32a: fonte `urgency`; o cidadão vê o desfecho e a orientação, o profissional também a cor e o CID (vêm no título e no resumo do api)". `PatientTimeline.tsx`: passa `canUseUrgency(user)`.
- `CounterReferrals.tsx`: `useQuery([ COUNTER_REFERRALS_KEY, unit.id ])` (sem polling); `<section aria-label="Retornos pedidos pela urgência">` com `DataTable` (Paciente, UPA, Alta em, "procurar a UBS até dd/mm/aaaa"), lida campo a campo (nunca `reason`), e a nota. `Attendance.tsx`: monta para unidade não UPA com `canUseUrgency(user)` e papel `citizen_verifier` ou `health_professional`.

- [ ] **Step 4: Rode e veja passar; regressão da linha do tempo; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod32a src/modules/UrgencyPanel.test.tsx src/lib/timeline* src/modules/timeline src/modules/urgency src/modules/Attendance*.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/UrgencyPanel.tsx src/modules/UrgencyPanel.test.tsx src/lib/timeline.ts src/lib/timeline.urgency.test.ts \
  src/modules/timeline/PatientTimeline.tsx src/modules/urgency/CounterReferrals.tsx src/modules/urgency/CounterReferrals.test.tsx src/modules/Attendance.tsx
$G commit -m "feat: add the urgency manager panel, the urgency timeline kind and counter-referrals at the UBS" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Suíte, build, varredura, revisão e prova no navegador (api real na 3054)

**Files:** nenhum novo (correções que a prova pedir entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod32a
$R/bin/test dashboard -w dashboard-mod32a
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **21 arquivos de teste novos**: `src/lib/api.urgency.test.ts`, `src/lib/urgency.test.ts`, `src/lib/usePolling.test.tsx`, `src/shell/modules.urgency.test.ts`, `src/lib/tvRoute.test.ts`, `src/lib/urgencyRules.test.ts`, `src/modules/protocolEditor/UrgencyPanels.test.tsx`, `src/modules/urgency/UrgencyReception.test.tsx`, `src/modules/Attendance.urgency.test.tsx`, `src/modules/urgency/UrgencyQueue.test.tsx`, `src/modules/urgency/ClassificationForm.test.tsx`, `src/modules/attendance/CallLocationPicker.test.tsx`, `src/modules/attendance/UnitQueue.waitingRoom.test.tsx`, `src/modules/urgency/UrgencyCare.test.tsx`, `src/modules/urgency/IdentifyPatient.test.tsx`, `src/modules/WaitingRoomSettings.test.tsx`, `src/modules/attendance/UnitForm.ticket.test.tsx`, `src/modules/WaitingRoomTv.test.tsx`, `src/modules/UrgencyPanel.test.tsx`, `src/lib/timeline.urgency.test.ts`, `src/modules/urgency/CounterReferrals.test.tsx`. Contagem que dobra = artefato de build descoberto pelo vitest: confira antes de seguir.

- [ ] **Step 2: Varredura de dado sensível, polling e storage**

```bash
cd $W
grep -rnE "refetchInterval" src/modules/urgency src/modules/waitingRoom src/modules/Urgency*.tsx src/modules/WaitingRoom*.tsx && echo "ERRO: polling fora do usePolling" || echo ok-polling
grep -rnE "setInterval|setTimeout" src/modules/urgency src/modules/waitingRoom src/modules/WaitingRoomTv.tsx && echo "revisar: timer fora do hook" || echo ok-timers
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" src/modules/urgency src/modules/waitingRoom src/modules/Urgency*.tsx src/modules/WaitingRoom*.tsx src/lib/urgency.ts \
  | grep -vE "readCallLocation|writeCallLocation|DISPLAY_TOKEN_KEY|storage\." && echo "ERRO: storage/log" || echo ok-storage
sed -n '/Urgência (módulo 32a/,$p' src/lib/api.ts | grep -nE "withQuery\(" | grep -vE "health_unit_id|month" && echo "revisar: URL" || echo ok-url
grep -rnE "\.\.\.(row|r|item|call|c|entry)\b" src/modules/urgency/UrgencyQueue.tsx src/modules/waitingRoom/TvBoard.tsx src/modules/urgency/CounterReferrals.tsx && echo "revisar: resposta espalhada" || echo ok-spread
```

Expected: `ok-polling`, `ok-timers`, `ok-storage`, `ok-url`, `ok-spread`.

- [ ] **Step 3: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, na TV (fora do `AuthProvider`, sem cookie, campo a campo), no `usePolling` (um pedido por vez, `clearTimeout`), em "nada clínico em lista", no `ConsultationWorkspace`/`ConsultationEditor` (a consulta comum e a odontológica continuam iguais) e no `UnitQueue`/`CheckIn` do 13 (iguais sem `waiting_room_display`).

- [ ] **Step 4: Prova no navegador** (só com o api do 32a mergeado ou no worktree dele, na 3054, e o banco de dev migrado e semeado **com autorização do usuário**)

Suba o Vite do worktree na rede do compose, porta **5213**, com proxy para `http://api:3054` (mesmo comando da prova do 31, trocando porta e worktree), abra `curitiba.localhost:5213/dashboard/` e percorra, com `urgency_care` e `waiting_room_display` ligados:
1. Editor de protocolo: "Nova classificação de risco (modelo)", mudar um tempo e um critério, salvar; em Protocolos, enviar, assinar (dois revisores), publicar e ativar.
2. Senhas, locais e TVs (step-up): na UPA, "Consultório 1", "Consultório 2", "Classificação"; uma TV; gerar o código. Cadastro da UPA com prefixo "U".
3. Numa segunda aba anônima, `curitiba.localhost:5213/dashboard/#/tv`: parear com o código; tela cheia; conferir em **1920×1080** e **1280×720** (`resize_window`) a 5 m de distância simulada (zoom do print a 25%): senha e local legíveis.
4. Recepção (`citizen_verifier`): chegada por CPF, por CNS, paciente novo sem CPF, não identificado com descrição; a senha grande; a mesma pessoa de novo (`encounter_open`).
5. Fila: a faixa "aguardando classificação" com o tempo; a enfermeira escolhe "Classificação", chama (a TV mostra "U-0xx → Classificação" em até 5 s), classifica com a cor sugerida, outra com a cor trocada e justificativa, e reclassifica; a fila ordena por cor e mostra o estouro (semente com chegada antiga).
6. Médico: escolhe "Consultório 2", "Chamar próximo" (segue a cor), a TV chama; consulta de urgência com CID, exame do 21 e receita do 19; finalizar; desfecho "Alta com retorno na UBS" (7 dias); "Chamar de novo" num outro.
7. Não identificado: atendido, depois identificado com paciente existente (caixa + confirmação) e, num segundo, como paciente novo; a segunda tentativa noutra aba dá `already_identified`.
8. Desfechos restantes: transferência com destino, saiu sem atendimento antes da classificação, óbito.
9. Revogar a TV: a aba da TV volta ao pareamento em até 5 s. Derrubar a rede da aba da TV (offline no DevTools): "sem conexão", e volta sozinha.
10. UBS: check-in com senha "A-0xx", chamada do 13 com local e "Chamar de novo"; "Retornos pedidos pela urgência" com o paciente do passo 6; linha do tempo do paciente com o kind "Urgência".
11. Painel da urgência com "menos de 5"; tablet (1024×768) e só teclado nos passos 4–6; ao terminar, `resize_window` preset desktop.

Anote cada passo com o que viu. Achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai para o usuário (não vira tarefa).

- [ ] **Step 5: Board e encerramento**

Cards do dashboard prontos: F-32.2..F-32.5 → **Done** se o api já terminou o deles; F-32.1 espera o maintenance e o contracts, F-32.6 o wpda (o último app move). **Verified** só depois da prova do Step 4, com o usuário. Merge, push e tag só com autorização. Worktree: `$G -C $R/apps/dashboard worktree remove $W` depois do merge.

---

## Divergências propostas ao contrato

Cada uma diz o que a tela precisa, o que o plano assume e quem decide. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D1 — Paciente novo na chegada e na identificação.** O §4 diz `new_patient: {name, birth_date, sex, cpf?, cns?, no_cpf_reason?}`; o api já tem `NewPatient` do 24a/25 (`Patients::CreateTyped`) com `full_name`, `mother_name?` e a validação de `no_cpf_reason` pelo dicionário. Proposta: reusar `NewPatient` (`full_name`) nas duas rotas; o dashboard reusa `newPatientProblem`.
- **D2 — Linhas da fila.** O §6 não traz `attendance_id` (a chamada, o "chamar de novo" e a consulta são pelo atendimento), nem a última chamada (a enfermagem precisa ver quem já chamou), nem quem está em atendimento (o médico reabre a consulta). Proposta: `awaiting_classification[]` e `by_color[]` com `attendance_id` e `last_call: {kind, location, called_at} | null`; `by_color[]` com `max_wait_minutes`; e um terceiro grupo `in_care[]` `{encounter_id, attendance_id, ticket, patient, color, consultation_id, consultation_status, called_by_name, called_at}`.
- **D3 — Sugestão e tempos.** O formulário mostra o tempo máximo de cada cor e não tem outra fonte dos níveis do protocolo ativo. Proposta: `POST /urgency/suggest {encounter_id, complaint, vital_signs, pain_scale}` → `{suggested: {color, criterion, rule_id} | null, levels: [{color, max_wait_minutes}]}` (padrão 0/10/60/120/240 sem protocolo). Sem simulador no editor: a prévia de um rascunho não ativo fica para depois.
- **D4 — Detalhe do encontro.** O contrato não tem leitura do encontro. Proposta: `GET /urgency/encounters/:id` (com trilha) → `{id, attendance_id, health_unit_id, state, ticket, arrived_at, patient: <patient-ref com description>, classifications: [{id, color, suggested, complaint, vital_signs, pain_scale, justification, classified_at, by: {name}}], consultation_id, consultation_status, outcome}`; a classificação, a identificação e o desfecho devolvem a mesma forma (a classificação, em `{encounter, suggested}`).
- **D5 — Finalizar a consulta `urgency`.** A finalização do 19a fecha o atendimento com o `outcome` do 13; na UPA quem fecha é o desfecho (§7). Proposta: `POST /consultations/:id/finalize {}` numa consulta `urgency` não exige nem aceita `outcome` e não fecha o atendimento; `GET /consultation_options` traz `kinds: ["urgency"]` para o CBO 225 vinculado à UPA.
- **D6 — Locais de chamada: quando o local é obrigatório.** A tela não sabe se a unidade tem TV ativa. Proposta: `GET /attendance/call_locations?health_unit_id` → `{items: [{id, health_unit_id, name, active}], location_required: bool}`; `PATCH /attendance/call_locations/:id {name, active}`.
- **D7 — TVs: forma.** Proposta: `GET /waiting_room/displays?health_unit_id` → `{items: [{id, health_unit_id, name, active, show_wait_times, paired, last_seen_at}]}` (sem token); `POST /waiting_room/displays/:id/pairing_code {}` → `{code, expires_at}`; `POST …/revoke {}` → o item. O `show_wait_times` vale **por TV**; o da unidade (§1) seria só o padrão de TV nova — proposta: ficar só o da TV e tirar a coluna da unidade, ou dizer qual vence.
- **D8 — Caminho do "chamar de novo".** O §6 diz `POST /attendances/:id/recall`; as rotas reais do 13 são `/attendance/attendances/:id/call`. Proposta: `POST /attendance/attendances/:id/recall {call_location_id}` → `{call_id}`.
- **D9 — Painel do gestor: forma.** Proposta: `GET /urgency/dashboard?month&health_unit_id?` → `{month, unit: {id, name} | null, total, by_color: [{color, encounters, door_to_classification: {median, p90} | "lt5", classification_to_doctor: {median, p90} | "lt5", overdue_pct: number | "lt5"}], by_outcome: [{kind, count}], left_without_care: {before_classification, after_classification}, length_of_stay: {median, p90} | "lt5"}`, minutos inteiros e `"lt5"` de 1 a 4.
- **D10 — Contrarreferência visível na UBS.** O §7 fala de "aviso interno" sem rota; sem ela, a UBS não vê. Proposta: `GET /urgency/counter_referrals?health_unit_id` → `{items: [{encounter_id, patient: <patient-ref>, upa: {name}, closed_at, return_days, due_on}]}` (sem o motivo; o motivo fica na linha do tempo profissional), para `citizen_verifier` e `health_professional` vinculado àquela UBS.
- **D11 — Dor.** O `VitalSigns` do 18 já tem `pain_score`; o §5 põe `pain_scale` fora dos sinais. O dashboard manda `pain_scale` e **tira** `pain_score` de `vital_signs`; proposta: o api recusar os dois juntos ou dizer qual vale.
- **D12 — Códigos sem nome no contrato** e forma de `problems[]`: `not_classified` (médico chama quem não foi classificado), `invalid_pairing_code` (inválido ou vencido), `destination_required`, `return_guidance_required`, `encounter_closed` (desfecho repetido), `not_unidentified` (identificar quem não é provisório), `cbo_not_allowed` (CBO fora da etapa), `patient_not_found`, `invalid_ticket_prefix`; `problems[]` como `{error, path}` (o dashboard aceita também `code`).
- **D13 — Forma do protocolo `urgency_classification`.** O §3 não fixa as chaves. Proposta para o schema do contracts: `{name, version, kind: "urgency_classification", levels: [{color, max_wait_minutes}] (exatamente as 5, na ordem), rules: [{id, when, color, criterion}] (1 a 100; `criterion` 3–120)}`; vocabulário da condição = `vitals.*` do 18 sem `vitals.pain_score`, mais `pain_scale`, `citizen.age` e `citizen.sex`; a queixa é texto livre e não entra em condição.
- **D14 — Quem registra o desfecho.** O §7 não diz. Proposta: o médico (CBO 225 vinculado) registra qualquer desfecho; recepção e enfermagem só `left_without_care`.
- **D15 — Quem identifica.** O §4 não diz. Proposta: `citizen_verifier` e `health_professional` vinculado à UPA.
- **D16 — Limites de texto.** Queixa 3–500; justificativa 10–500; descrição 10–500; destino 3–200; pedido de vaga até 2000; motivo do retorno 10–500; prazo 1–90 dias; nome de local e de TV 2–40.
- **D17 — Token da TV no aparelho.** A TV precisa sobreviver a religar sem pedir código; o dashboard guarda o `display_token` no `localStorage` do aparelho (única exceção de storage do módulo). Proposta: o token não expira sozinho, só pela revogação (ou dizer a validade, e a TV pede código ao vencer com `display_revoked`).
- **D18 — `wait_times`.** O §8 escreve `{color, median_minutes | "lt5_patients"}`. O plano lê `median_minutes: number | "lt5_patients"`; e a TV ignora qualquer campo fora de `ticket`, `location`, `kind` e `called_at` em `calls[]`.
- **D19 — Senha na resposta do 13.** O §1 dá `ticket` ao `attendance`; proposta: `ticket` em `POST /attendance/check_ins`, `/check_ins/exception` e nas linhas de `GET /attendance/units/:id/queue` (nulo sem `waiting_room_display`).
- **D20 — Pareamento sem cookie.** `POST /waiting_room/pair` e `GET /waiting_room/board` aceitam requisição sem sessão nem CSRF (a TV não tem cookie); o resto de `/waiting_room/*` exige `municipal_admin` com step-up.
- **D21 — CBO da enfermagem.** O §1 diz 2235\* ou 3222\*; o dashboard mostra "Chamar para classificação" e "Classificar" só com esse CBO no vínculo ativo da UPA, e o api confere (`cbo_not_allowed`).
- **D22 — Laranja no tema.** O tema não tem papel laranja; o dashboard usa `tone: "urgent"` (mesmo tom do vermelho) com o texto "laranja" sempre junto. Um papel de cor novo é decisão do ciclo de interface (`contracts/design-tokens`), fora deste módulo.

## Self-review

1. **Cobertura da spec (32a):** decisão 1 → Task 4; 4 (TV e histórico) → Tasks 12 e 13 (o wpda é outro plano); 6 e R1 → Task 5 (mesa no Atendimento, `walk_in` é do api); 7 e R4 → Tasks 5 e 10; 8 → Task 7; 9 e R5 → Tasks 6 e 8; 10 e R3 → Task 9; 11 → Task 9; 12 → Tasks 9 e 13; 13 e R6 → Tasks 8 e 11; 14 → Tasks 11 e 12; 15 → api (nada na tela); 16 → Task 9 (a sugestão do 25 vem da consulta sem mudança); 17 → Task 13; 18 → Task 13; 19 → Tasks 2, 5–10; 23 → Tasks 1–3. §4 telas → Tasks 4–13; §5 → Global Constraints, Tasks 6, 12 e 14 Step 2; §6 dashboard → todas as tasks com teste de componente e a TV.
2. **Passos:** cada passo de código diz arquivo, assinatura e os valores da spec/contrato; corpos só nas fixtures, nos testes e nas frases fixas (`URGENCY_ERRORS`, `PRIORITY_PHRASE`); os rótulos de componentes existentes que os testes usam (`VitalSignsFields`, `ConfirmButton`, `SensitiveAction`, `ConsultationWorkspace`, títulos do Atendimento) são conferidos na Task 0 contra o `3671781`.
3. **Tipos:** `UrgencyQueue`, `UrgencyWaitingRow`, `UrgencyColorRow`, `UrgencyInCareRow`, `UrgencyEncounter`, `UrgencySuggestion`, `CallLocationList`, `WaitingRoomBoard` e as funções da Task 1 são os mesmos nas Tasks 2–13; `QueueActions` (Task 6) recebe as ações das Tasks 7–10; `useCallLocation` (Task 8) serve à UPA e à UBS; `ConsultationWorkspace.urgency` (Task 9) só é usado por `UrgencyCare`; `usePolling` (Task 2) só por `UrgencyQueue` e `TvBoard`.
4. **Review Focus:** as cinco linhas têm teste na task dona (12; 2 e 6; 5; 10; 7).
5. **Proporção:** o código do plano é teste, fixture e frase fixa; implementação por assinatura e valores.
