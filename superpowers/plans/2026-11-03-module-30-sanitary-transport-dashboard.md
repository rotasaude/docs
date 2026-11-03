# Módulo 30 — Transporte sanitário e TFD (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 30** (`feat/mod-30-sanitary-transport`, worktree `apps/api/.claude/mod30`, porta de dev **3052**) com as rotas do contrato §3–§9 (e as Divergências D1–D18 deste plano que o plano do api adotar), os interruptores `sanitary_transport` e `tfd` na sessão, os papéis `transport_dispatcher`, `tfd_committee` e `driver`, o tipo de unidade `health_management` e a semente do 30. Ele é **mergeado antes** deste (contrato §13: api → dashboard → wpda → maintenance). As Tasks 0–15 não dependem dele (dublês de `src/lib/api` com fixtures nas formas do contrato §2); a prova da Task 16 depende.

**Goal:** Lado dashboard do 30: a **central de transporte** (veículos, motoristas com alerta de CNH, destinos com km, agenda de viagens por dia e veículo, montar a viagem com os atendimentos marcados na data, passageiro sem atendimento com motivo, séries recorrentes, pendências, lista de passageiros em PDF, faltas e correção de presença), a **tela do motorista** no celular (viagens do dia → passageiros por parada → embarcou/faltou → fechamento com odômetro, abastecimento e despesas), a **comissão de TFD** (fila sem dado clínico, leitura do laudo com trilha, autorizar/devolver/recusar com step-up, atendimento no destino, internação, trechos, ajuda de custo, pagamento registrado e prestação de contas com upload do comprovante), o botão **"Pedir TFD"** na consulta finalizada e os **meus pedidos de TFD** do médico, as **regras da cidade** (distância mínima, acompanhante, CNES do TFD, complemento por código) e o tipo de unidade **"Gestão"**, os papéis novos na **Equipe** e no **menu**, o prestador externo do 26a com o **município (IBGE)** e o agendamento externo com o **prestador**, a fonte **`transport`** na linha do tempo e o **painel** de transporte e TFD com "menos de 5".

**Architecture:** O cliente HTTP ganha a seção "Transporte sanitário e TFD (módulo 30)" no fim de `src/lib/api.ts` (tipos = formas do contrato §2–§9; `CareProvider`, `RegulationScheduleInput` e `TimelineKind` ganham o que é do 30). As regras de tela ficam puras em `src/lib/transport.ts` (portão, papéis, rótulos, validações de frota, viagem, passageiro, série e fechamento, dinheiro, recusas) e `src/lib/tfd.ts` (portão, papéis, laudo, decisão, trechos, pagamento, arquivo). As peças ficam em `src/modules/transport/` e `src/modules/tfd/`, e as telas nos grupos novos **"Transporte"** e **"TFD"** (`modules.ts`, `catalog.ts`, `App.tsx`), depois de "Saúde bucal". O motorista que só tem o papel `driver` vê um shell reduzido (como o prestador do 21a): "Minhas viagens" e "Segurança". O dashboard nunca decide o que o api garante: sobreposição, lotação, CNH, regra do acompanhante, distância do TFD, cálculo da ajuda de custo, BPA, avisos, trilha, imutabilidade e retenção LGPD são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md` (§1 intenção, sucesso e antiobjetivos; §2 decisões 1–23 e as do coordenador; §4 telas; §5 segurança e LGPD; §6 testes; §8 F-IDs; §9 riscos), `docs/adr/0048.md`, a pesquisa `docs/pesquisa/2026-11-03-transporte-e-tfd-30.md` e o contrato `docs/superpowers/plans/2026-11-03-module-30-sanitary-transport-contracts.md` (§1–§13), com as convenções herdadas (29 §15, 28 §16). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D18); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptores `sanitary_transport` (sem pré-requisito) e `tfd` (requer `feature:sanitary_transport feature:clinical_record`); papéis `transport_dispatcher` e `tfd_committee` (privilegiados) e `driver` (não privilegiado); veículo `van` | `minibus` | `bus` | `car` | `ambulance_a`; CNH `B` | `C` | `D` | `E`, `license_status` `valid` | `expiring` | `expired`; viagem `planned` | `confirmed` | `in_progress` | `done` | `cancelled`; origem `regulation` | `appointment` | `tfd` | `series` | `exception`; motivo `follow_up_return` | `legacy_patient` | `scheduling_pending` | `other`; presença `pending` | `boarded` | `no_show`; TFD `draft` | `submitted` | `returned` | `authorized` | `denied` | `scheduled` | `travelling` | `completed` | `cancelled`; trecho `fleet` | `bus_ticket` | `air` | `river` e `outbound` | `return`; ajuda `meal_overnight` | `meal_day` | `ground_km` | `air_miles` | `river_miles`; pagamento `cash` | `transfer` | `pix` | `check`. **Código desconhecido aparece cru, nunca some.**
- **Interruptores** (spec decisão 23): sem `sanitary_transport` na sessão, o grupo "Transporte" some; sem `tfd` (ou sem `sanitary_transport` ou sem `clinical_record`), o grupo "TFD" e o botão "Pedir TFD" somem. 403 `feature_disabled` → `TRANSPORT_DISABLED` / `TFD_DISABLED`. O dashboard nunca escreve interruptor; as duas chaves ganham só rótulo em `features.ts`. A entrada `transport` já gravada continua legível na linha do tempo (o interruptor só tira o filtro).
- **Quem faz o quê** (contrato §3–§6, §9; decisões do coordenador): `municipal_admin` — regras da cidade (`/transport/admin/settings`, step-up) e papéis pela Equipe; `transport_dispatcher` — frota, motoristas, destinos, viagens, passageiros, séries, pendências, lista impressa, faltas e correção de presença; **nunca lê o laudo** (nenhuma tela da central chama `/tfd/*`); `driver` — só as próprias viagens do dia; `tfd_committee` — fila, laudo, decisão, atendimento, internação, trechos, ajuda, pagamento e prestação de contas; médico (`health_professional` com CBO `225*`, autor da consulta) — pede o TFD e edita o próprio pedido em `draft`/`returned`; painel — `municipal_admin`, `transport_dispatcher` e `tfd_committee`.
- **Sem dado clínico na central e no motorista** (spec decisão 14, §5): a lista de passageiros, a tela do motorista, o PDF e as sugestões de atendimento nunca mostram CID, diagnóstico, procedimento, especialidade, justificativa ou o texto do pedido — a tela só tem os campos de `<passenger>` (contrato §2). A origem aparece pelo rótulo (`originLabel`), nunca pelo conteúdo da origem.
- **Leitura nominal com trilha, sem polling** (spec §5; contrato §6): o laudo (`POST /tfd/cases/:id/read`) é lido por **mutação no clique**, com `gcTime: 0`, nunca sozinho; fila, frota, viagens, séries, pendências, faltas e painel são `STATIC_QUERY` (ao abrir, ao filtrar, depois de uma ação). **Nenhum `refetchInterval` no módulo.**
- **Dado pessoal ou clínico só no corpo** (brief; contrato, convenções): CPF, CNS, nome, CNH, documento do acompanhante, destino de tratamento, CID e justificativa nunca na URL, em mensagem de erro, em `console`, em Analytics, em `localStorage`, `sessionStorage` ou IndexedDB. Na URL vão só ids, `date`, `month`, `from`, `to`, `status`, `before` e `consultation_id`. Busca de paciente por CPF por POST. O comprovante da prestação de contas vai em base64 no corpo, sem o nome do arquivo (D11).
- **Todas as recusas de uma vez** (29 §15; 28 §16): a tela confere o formulário inteiro antes de enviar e mostra cada problema; a recusa do api com `problems: [ { "path", "error" } ]` mostra todas, traduzidas (`transportRefusals`, `tfdRefusals`); sem `problems`, a do topo. `vehicle_overlap` e `driver_overlap` são 422 com frase, nunca "erro inesperado".
- **Recusar em vez de cortar** (21a §15): placa fora do padrão, RENAVAM sem 11 dígitos, km ≤ 0 em destino intermunicipal, retorno antes da saída, odômetro de chegada menor que o de saída, valor com três casas, data de pagamento futura, arquivo acima de 10 MB — recusa com a frase; nada é arredondado, cortado nem descartado em silêncio. Dinheiro aceita vírgula ou ponto e milhar com ponto (`"1.234,56"`), duas casas no máximo.
- **Celular e tablet** (brief): a tela do motorista funciona a **375 px** — cartões empilhados, sem `DataTable` nem grade horizontal, botões "Embarcou"/"Faltou" de alvo ≥ 44 px (`buttonStyle`) com `aria-pressed`; a central funciona em tablet e só com teclado — formulários nativos (`<form>` com `Enter` enviando, `<select>` e `<input>` com rótulo), ordem de Tab = ordem visual, foco vai ao título da viagem aberta e à frase de sucesso depois de salvar; situação nunca só por cor (texto no `Tag`); `outline: "none"` proibido pela guarda.
- **Sem duplo envio**: todo botão de escrita fica desabilitado enquanto a chamada corre (`busy`), e a resposta de uma ação já trocada (outra data, outra viagem, outro processo) é ignorada (`seq`), como no `PrescriptionLookup` do 27a.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo — `{}` quando não há campo).
- Textos (D15; se o api fixar outros, o api vence): motivo (cancelamento, retirada, exceção `other`, suspensão, decisão, correção) 10–500; justificativa do laudo 20–2000; nome do acompanhante 3–120; documento do acompanhante ≤ 20; rótulo de parada 3–120; posto, empresa da passagem e observação de despesa ≤ 120; número do recibo 1–40; modelo do veículo 2–80.
- "Hoje" é o dia da cidade (`todayInCity()` de `src/lib/campaigns.ts`) e "agora" é `new Date()`; telas recebem `today`/`now` por prop (padrão o real) e os testes passam `TODAY30 = "2026-11-03"` (terça) e `NOW30 = new Date("2026-11-03T10:00:00-03:00")`, sem relógio falso. Hora digitada é hora de parede da cidade (`cityLocalIso` de `src/lib/format.ts`).
- Interface (ciclo de interface, subprojetos 1–6 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `formStyles`, os estilos de `src/modules/exams/styles.ts`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-30.1..F-30.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-30.1: api, dashboard e maintenance; F-30.8: api, wpda e dashboard; F-30.2..F-30.6: api e dashboard; F-30.7 é só do api) e a **Verified** depois da prova no navegador (Task 16).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod30` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-30-sanitary-transport` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod30`, `G=/opt/homebrew/bin/git`.
- Testes: `$R/bin/test dashboard -w dashboard-mod30 <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx` (`sessionWith` abre a janela de step-up; `{ mfa_verified_at: null }` força o código); a sessão entra por `fetchCurrentSession` mockado; as fixtures do 30 vêm de `src/test/transportFixtures.ts` (Task 1).

## Review Focus

1. **Motorista toca "Embarcou" duas vezes seguidas no celular, ou a rede cai no meio do toque.** Espera-se: uma chamada só a `recordAttendance` por toque (o par de botões fica desabilitado até a resposta); na falha, a presença volta ao valor anterior e a frase aparece **na linha daquele passageiro**, sem mexer nos outros; "Fechar viagem" continua barrado enquanto houver `pending`. Teste: Task 11, "toque duplo grava uma vez; falha devolve a presença anterior na linha".
2. **Confirmar uma viagem cujo veículo e motorista já estão em outra viagem no mesmo horário.** Espera-se: as duas frases de uma vez (`vehicle_overlap` e `driver_overlap`), a viagem continua `planned`, o formulário fica como estava e a agenda do dia é relida; nada de "erro inesperado". Teste: Task 8, "sobreposição de veículo e motorista: as duas frases, nada muda".
3. **Escalar motorista com CNH vencida, que vence na semana, ou com categoria B numa van de 15 lugares.** Espera-se: o vencido aparece no seletor como "(CNH vencida)" e `confirm` é barrado na tela com a frase; o que vence em até 30 dias aparece com aviso "CNH vence em dd/mm/aaaa" e é aceito; B numa van (`seats > 8`) é barrado antes de enviar com "veículo com mais de 8 lugares exige CNH D ou E". Teste: Task 2, "driverProblems: vencida, categoria, a vencer não bloqueia", e Task 8, "confirmar com motorista B na van recusa antes de enviar".
4. **Comissão clica "Autorizar" sem atendimento marcado no destino, ou a janela do segundo fator fecha entre abrir e confirmar.** Espera-se: sem `appointment_at`, a tela recusa com "marque o atendimento no destino antes de autorizar" e põe o foco em "Marcar atendimento", sem chamar `decideTfdCase`; com a janela fechada, o `SensitiveAction` pede o código e repete a decisão uma vez só; o motivo (10–500) é obrigatório nas três decisões. Teste: Task 13, "autorizar sem atendimento não envia; step-up repete uma vez".
5. **Prestação de contas com um PDF de 12 MB, um `.docx`, ou o mesmo clique duas vezes.** Espera-se: tamanho e tipo recusados antes de ler o arquivo (nenhum `readBase64`, nenhuma chamada); o nome do arquivo não sai da tela nem vai no corpo; com PNG de 200 KB, uma chamada só, e o processo aparece `completed` com o comprovante. Teste: Task 14, "comprovante grande ou de outro tipo não sai; duplo clique envia uma vez".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.transport.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/transportFixtures.ts` | tipos e cliente HTTP do 30; interruptores; proxy; dados de teste | 1 |
| `src/lib/transport.ts`, `src/lib/transport.test.ts` | regras de tela da central e do motorista | 2 |
| `src/lib/tfd.ts`, `src/lib/tfd.test.ts` | regras de tela do TFD | 3 |
| `src/lib/team.ts`, `src/lib/team.test.ts`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.transport.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/transport/TransportGate.tsx`, `src/modules/transport/TransportGate.test.tsx`, telas-portão `src/modules/Transport{Trips,Series,Pending,Fleet,MyTrips,Panel,Settings}.tsx`, `src/modules/Tfd{Queue,MyCases}.tsx` | papéis na Equipe, grupos "Transporte" e "TFD", shell do motorista, portões | 4 |
| `src/modules/TransportSettings.tsx`, `src/modules/TransportSettings.test.tsx`, `src/lib/attendance.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx` | regras da cidade; unidade "Gestão" (F-30.1) | 5 |
| `src/modules/TransportFleet.tsx`, `src/modules/transport/VehiclesPanel.tsx`, `src/modules/transport/DriversPanel.tsx`, `src/modules/transport/DestinationsPanel.tsx`, `src/modules/TransportFleet.test.tsx` | veículos, motoristas com alerta de CNH, destinos (F-30.1) | 6 |
| `src/lib/regulation.ts`, `src/modules/regulation/ProviderForm.tsx`, `src/modules/regulation/ScheduleOutcome.tsx`, `src/modules/regulation/ScheduleOutcome.provider.test.tsx` | município do prestador e prestador no agendamento externo do 26a (F-30.1/F-30.2) | 7 |
| `src/modules/TransportTrips.tsx`, `src/modules/transport/TripCard.tsx`, `src/modules/transport/TripForm.tsx`, `src/modules/TransportTrips.test.tsx` | agenda do dia, criar, editar, confirmar e cancelar viagem (F-30.2) | 8 |
| `src/modules/transport/PassengerList.tsx`, `src/modules/transport/ScheduledCare.tsx`, `src/modules/transport/PassengerForm.tsx`, `src/modules/transport/TransportPersonPick.tsx`, `src/modules/transport/openManifest.ts`, `src/modules/transport/Passengers.test.tsx` | montar a viagem, passageiro com e sem atendimento, retirar, PDF (F-30.2) | 9 |
| `src/modules/TransportSeries.tsx`, `src/modules/transport/SeriesForm.tsx`, `src/modules/TransportPending.tsx`, `src/modules/TransportSeries.test.tsx` | séries, suspensão, pendências e faltas (F-30.3) | 10 |
| `src/modules/TransportMyTrips.tsx`, `src/modules/transport/DriverTrip.tsx`, `src/modules/transport/CloseTripForm.tsx`, `src/modules/transport/AttendanceCorrection.tsx`, `src/modules/TransportMyTrips.test.tsx` | tela do motorista, presença, fechamento; correção pela central (F-30.4) | 11 |
| `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/tfd/TfdRequestForm.tsx`, `src/modules/TfdMyCases.tsx`, `src/modules/tfd/TfdRequest.test.tsx` | "Pedir TFD" e meus pedidos (F-30.5) | 12 |
| `src/modules/TfdQueue.tsx`, `src/modules/tfd/CaseView.tsx`, `src/modules/tfd/DecisionPanel.tsx`, `src/modules/tfd/AppointmentForm.tsx`, `src/modules/tfd/InpatientForm.tsx`, `src/modules/TfdQueue.test.tsx` | fila, leitura com trilha, decisão com step-up, atendimento, internação (F-30.5) | 13 |
| `src/modules/tfd/LegsPanel.tsx`, `src/modules/tfd/AllowancesPanel.tsx`, `src/modules/tfd/PaymentForm.tsx`, `src/modules/tfd/AccountabilityForm.tsx`, `src/modules/tfd/CaseMoney.test.tsx` | trechos, ajuda de custo, pagamento, prestação de contas (F-30.6) | 14 |
| `src/lib/timeline.ts`, `src/lib/timeline.transport.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/TransportPanel.tsx`, `src/modules/TransportPanel.test.tsx` | fonte `transport` na linha do tempo; painel (F-30.8) | 15 |
| — | suíte, build, revisão, prova no navegador | 16 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, CNH, nome, documento do acompanhante, CID, justificativa, arquivo) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa —: placa antiga e Mercosul, CNH vencida × a vencer × categoria, retorno antes da saída, lotação e cadeira, motivo de exceção, dinheiro com milhar e três casas, odômetro, distância e região metropolitana do TFD, autorização sem atendimento, arquivo (Tasks 2 e 3); menu e portão por papel e interruptor, inclusive o shell do motorista (Task 4); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e fixtures do contrato §2, cobrindo o caminho feliz, o teclado, cada recusa que muda a tela (`vehicle_overlap`, `driver_overlap`, `seats_full`, `companion_not_allowed`, `attendance_pending`, `appointment_required`, `distance_below_minimum`), o duplo envio e o que **não** pode aparecer (CID, justificativa e procedimento fora da comissão; nenhuma chamada a `/tfd/*` nas telas da central) (Tasks 5–15); `tsc`, build e a suíte inteira (Task 16), e a prova no navegador com o api real na 3052, o motorista a 375 px (Task 16). O dashboard não testa o que é do api (sobreposição no banco, cálculo da ajuda, BPA, aviso, SMS, trilha).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`022ccd3` ou mais novo, com o 29 mergeado); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod30`.
- Produces: cards F-30.1..F-30.8 em In Progress; worktree `$W` na branch `feat/mod-30-sanitary-transport`; a base anotada; as rotas e as formas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-30\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api, do wpda e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-30.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '"vaccination" | "rnds_immunization" | "rnds_mock";\|  rnds_mock: "RNDS simulada'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Sala de vacina (módulo 29\|^export interface CareProvider {\|^export interface CareProviderInput\|^export interface RegulationScheduleInput\|^  | "vaccination";\|^function readBase64\|^const withQuery = \|^async function binaryFetch\|^export async function lookupVaccinationPerson'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '{ role: "stock_keeper", label: "Almoxarife" },\|^const PRIVILEGED_ROLES'
$G -C $R/apps/dashboard show origin/main:src/lib/team.test.ts | grep -n 'oferece os 18 papéis'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "dental-prosthesis" | "dental-ceo-goals" | "dental-indicators" | "dental-admin";\|  { label: "Exames", items: \[\|^export function isExamProvider\|      if (item.id === "dental-admin") return dentalOn && dental.admin;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 17\|NAV_GROUPS.length - 16'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n 'IconVaccine, IconNeedle\|    "dental-admin": {'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'const providerOnly = isExamProvider(auth.user);\|    case "dental-admin":'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"/vaccination\|\^/vaccination'
$G -C $R/apps/dashboard show origin/main:src/lib/attendance.ts | grep -n '^export function unitKindOptions'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/UnitForm.tsx | grep -n 'unitKindOptions(dentalOn'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/Units.tsx | grep -n 'const KIND_LABEL'
$G -C $R/apps/dashboard show origin/main:src/modules/regulation/ProviderForm.tsx | grep -n 'const \[ cnes, setCnes \]'
$G -C $R/apps/dashboard show origin/main:src/modules/regulation/ScheduleOutcome.tsx | grep -n 'scheduleRegulationRequest'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationView.tsx | grep -n '  regulation?: boolean;\|const showRegulation = '
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n 'const regulationOn = \|regulation={regulationOn} notSent={notSent}'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '^export function timelineKinds\|  { key: "vaccination", label: "Vacina" }'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n 'timelineKinds('
$G -C $R/apps/dashboard show origin/main:src/modules/territory/PersonStart.tsx | grep -n '^export interface PersonSource'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod30 -b feat/mod-30-sanitary-transport origin/main
cd $R/.worktrees/dashboard-mod30 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod30
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Bus|SteeringWheel|CalendarRepeat|UserX|Car|MapPins|Gavel|FilePlus|ScaleOutline|ClipboardHeart|Cash)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os ícones (falta algum → escolha outro que exista e use o mesmo na Task 4; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 16 espera a base mais **17 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 30** (informativo para as Tasks 1–15)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-30-sanitary-transport
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod30 api bin/rails routes 2>/dev/null | grep -E "transport|tfd|regulation/(providers|requests/.*/schedule)" | head -80
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-11-03-module-30-sanitary-transport-api.md 2>/dev/null | grep -n "^## \|^### Task\|Divergência\|Desvio" | head -60
```

Expected: a lista das rotas. Se o api adotou outra forma para alguma Divergência (D1–D18), **use a do api** (ele é mergeado antes) e anote no commit da task que a consome; se não adotou, a task usa a forma da Divergência e a Task 16 confere.

---

### Task 1: Tipos, cliente HTTP, interruptores, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; `CareProvider`, `CareProviderInput`, `RegulationScheduleInput`, `RegulationRequest.schedule` e `TimelineKind`)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/transportFixtures.ts`
- Test: `src/lib/api.transport.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError`, `withQuery`, `readBase64`, `PharmacyPerson`, `DuplicateQuery`, `DuplicateCandidate`, `SessionUser` (existentes em `src/lib/api.ts`); `sessionWith` de `src/test/campaignFixtures.tsx`.
- Produces (todas em `src/lib/api.ts`, exportadas):
  - Tipos: `VehicleKind`, `LicenseCategory`, `LicenseStatus`, `TripStatus`, `PassengerOrigin`, `ExceptionReason`, `Attendance`, `TfdStatus`, `TfdLegMode`, `TfdLegDirection`, `TfdAllowanceKind`, `TfdPaymentMethod`, `ExpenseKind` (`"toll" | "parking" | "meal" | "other"`, D7), `TransportCount = number | "lt5"`.
  - Formas (contrato §2 + D): `TransportVehicle`, `VehicleInput = Omit<TransportVehicle, "id">`, `TransportDriver`, `DriverInput { user_id; license_number?: string; license_category; license_expires_on }`, `DriverCandidate { user_id; name }` (D2), `TransportDestination`, `DestinationInput = Omit<TransportDestination, "id">`, `TripPatient { id; display_name: string | null; has_cpf?: boolean; cns_masked?: string | null; phone_masked: string | null }`, `TripPassenger` (com `destination.name`, D16), `TripStop`, `TripStopInput { id?: string; position: number; kind: "boarding" | "destination"; destination_id?: string | null; label: string }` (D3), `TripLog`, `Trip`, `TripInput`, `PassengerInput`, `ScheduledCareItem` (D4), `TransportSeries`, `SeriesInput` (D5), `PendingItem`, `NoShowRow` (D6), `AttendanceItem`, `FuelInput { liters: number; amount_cents: number; station: string }`, `ExpenseInput { kind: ExpenseKind; amount_cents: number; note: string }`, `TripCloseInput`, `AttendanceCorrectionInput { items: AttendanceItem[]; reason: string }` (D7), `TransportSettings` (com `allowance_codes`, D17), `TransportSettingsInput = Omit<TransportSettings, "allowance_codes">`, `TransportDashboard`, `TfdReport`, `TfdReportInput = Omit<TfdReport, "sigtap_name">`, `TfdCase` (com `trip_passengers`, D10), `TfdCaseRow`, `TfdCaseInput`, `TfdOptions` (D8), `TfdPrefill`, `TfdLeg`, `TfdLegInput` (com `companion?: boolean`, D10), `TfdAllowance`, `TfdPayment`, `TfdPaymentInput`.
  - Funções: `listVehicles(): Promise<TransportVehicle[]>`, `createVehicle(i: VehicleInput)`, `updateVehicle(id, i)`, `listDrivers(): Promise<TransportDriver[]>`, `listDriverCandidates(): Promise<DriverCandidate[]>`, `createDriver(i: DriverInput)`, `updateDriver(id, i)`, `listDestinations(): Promise<TransportDestination[]>`, `createDestination(i)`, `updateDestination(id, i)`, `listTrips(q: { date: string; vehicleId?: string | null }): Promise<Trip[]>`, `createTrip(i: TripInput): Promise<Trip>`, `updateTrip(id, i): Promise<Trip>`, `confirmTrip(id): Promise<Trip>`, `cancelTrip(id, reason): Promise<Trip>`, `findScheduledCare(q: { date: string; destinationId?: string | null }): Promise<ScheduledCareItem[]>`, `addPassenger(tripId, i: PassengerInput): Promise<TripPassenger>`, `removePassenger(tripId, passengerId, reason): Promise<void>`, `fetchTripManifest(tripId): Promise<{ blob: Blob; filename: string | null }>`, `lookupTransportPerson(cpf): Promise<PharmacyPerson>`, `findTransportDuplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>` (D1), `listSeries(): Promise<TransportSeries[]>`, `createSeries(i)`, `updateSeries(id, i)`, `suspendSeries(id, { from_on, reason }): Promise<TransportSeries>`, `listTransportPending(): Promise<PendingItem[]>`, `listNoShows(q: { from: string; to: string }): Promise<NoShowRow[]>`, `listMyTrips(date): Promise<Trip[]>`, `startMyTrip(id): Promise<Trip>`, `recordAttendance(id, items: AttendanceItem[]): Promise<Trip>`, `closeMyTrip(id, i: TripCloseInput): Promise<Trip>`, `correctAttendance(tripId, i: AttendanceCorrectionInput): Promise<Trip>`, `getTransportSettings()`, `updateTransportSettings(i)`, `getTransportDashboard(month): Promise<TransportDashboard>`, `getTfdOptions(): Promise<TfdOptions>`, `getTfdPrefill(consultationId): Promise<TfdPrefill>`, `createTfdCase(i): Promise<TfdCase>`, `updateTfdCase(id, i): Promise<TfdCase>`, `submitTfdCase(id): Promise<TfdCase>`, `listTfdCases(q: { status?: string | null; before?: string | null }): Promise<{ items: TfdCaseRow[]; next_before: string | null }>`, `listMyTfdCases(): Promise<TfdCaseRow[]>` (D9), `readTfdCase(id): Promise<TfdCase>`, `decideTfdCase(id, { decision, reason }): Promise<TfdCase>`, `setTfdAppointment(id, { appointment_at, provider_id? }): Promise<TfdCase>`, `setTfdInpatient(id, { from, to? }): Promise<TfdCase>`, `addTfdLeg(caseId, i: TfdLegInput): Promise<TfdCase>`, `markTfdLegDone(legId): Promise<TfdCase>`, `listTfdAllowances(caseId): Promise<TfdAllowance[]>`, `recordTfdPayment(caseId, i): Promise<TfdCase>`, `uploadTfdAccountability(caseId, file: File): Promise<TfdCase>` (D11). Toda escrita do TFD devolve o `<tfd-case>` inteiro (D12).
  - Regulação (D13): `CareProvider.ibge_code: string | null`, `CareProviderInput.ibge_code?: string | null`, `RegulationScheduleInput.care_provider_id?: string | null`, `RegulationRequest.schedule.care_provider?: { id: string; name: string } | null`.
  - Linha do tempo: `TimelineKind` ganha `"transport"`.
  - `src/test/transportFixtures.ts`: `TODAY30`, `NOW30`, `VAN`, `CAR`, `DRIVER_D`, `DRIVER_B`, `DRIVER_EXPIRED`, `DRIVER_EXPIRING`, `DEST_CURITIBA`, `DEST_NEAR`, `DEST_METRO`, `DEST_RURAL`, `passenger(o?)`, `trip(o?)`, `series(o?)`, `tfdCase(o?)`, `tfdRow(o?)`, `SETTINGS`, `trUser(roles, features?)`.

- [ ] **Step 1: Escreva as fixtures** `src/test/transportFixtures.ts` (o único arquivo com corpo completo nesta task; as formas são o contrato §2 com as Divergências)

```ts
// Dados dos testes do módulo 30 (transporte e TFD). Hoje: terça, 2026-11-03, 10h. Formas do contrato §2 e das Divergências do plano.
import type {
  SessionUser, TfdCase, TfdCaseRow, TransportDestination, TransportDriver, TransportSeries, TransportSettings, TransportVehicle, Trip, TripPassenger
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY30 = "2026-11-03";
export const NOW30 = new Date("2026-11-03T10:00:00-03:00");
export const VAN: TransportVehicle = { id: "v1", plate: "ABC1D23", renavam: "12345678901", model: "Sprinter 415", kind: "van",
  renem_code: null, seats: 15, wheelchair_spots: 1, cnes: "2384299", active: true };
export const CAR: TransportVehicle = { ...VAN, id: "v2", plate: "XYZ9876", model: "Spin", kind: "car", seats: 4, wheelchair_spots: 0 };
export const DRIVER_D: TransportDriver = { id: "d1", user: { id: "u-d1", name: "João Motorista" }, license_category: "D",
  license_expires_on: "2028-05-10", license_status: "valid", license_number_last4: "4321" };
export const DRIVER_B: TransportDriver = { ...DRIVER_D, id: "d2", user: { id: "u-d2", name: "Ana Condutora" }, license_category: "B" };
export const DRIVER_EXPIRED: TransportDriver = { ...DRIVER_D, id: "d3", user: { id: "u-d3", name: "Pedro Vencido" },
  license_expires_on: "2026-10-30", license_status: "expired" };
export const DRIVER_EXPIRING: TransportDriver = { ...DRIVER_D, id: "d4", user: { id: "u-d4", name: "Rita Avencer" },
  license_expires_on: "2026-11-20", license_status: "expiring" };
export const DEST_CURITIBA: TransportDestination = { id: "t1", ibge_code: "4106902", name: "Curitiba", distance_km: 110, metropolitan: false, active: true };
export const DEST_NEAR: TransportDestination = { id: "t2", ibge_code: "4119905", name: "Ponta Grossa", distance_km: 30, metropolitan: false, active: true };
export const DEST_METRO: TransportDestination = { id: "t3", ibge_code: "4104204", name: "Campo Largo", distance_km: 60, metropolitan: true, active: true };
export const DEST_RURAL: TransportDestination = { id: "t4", ibge_code: "4100000", name: "Distrito de Santa Cruz", distance_km: 0, metropolitan: false, active: true };

export function passenger(o: Partial<TripPassenger> = {}): TripPassenger {
  return { id: "p1", patient: { id: "pt1", display_name: "Maria da Silva", has_cpf: true, cns_masked: "*** **** **** 1234", phone_masked: "(41) ****-1234" },
    stop_id: "s1", destination: { provider_name: "Hospital Regional", address: "Rua XV, 100", destination_id: "t1", name: "Curitiba" },
    appointment_at: "2026-11-03T09:00:00-03:00", takes_seat: true, wheelchair: false, companion: null, origin: "regulation",
    exception_reason: null, attendance: "pending", companion_attendance: null, ...o };
}

export function trip(o: Partial<Trip> = {}): Trip {
  return { id: "tr1", date: TODAY30, departure_at: "2026-11-03T05:30:00-03:00", return_at: "2026-11-03T18:00:00-03:00",
    vehicle: { id: VAN.id, plate: VAN.plate, model: VAN.model, kind: VAN.kind, seats: VAN.seats, wheelchair_spots: VAN.wheelchair_spots },
    driver: { id: DRIVER_D.id, name: DRIVER_D.user.name }, status: "confirmed",
    stops: [ { id: "s1", position: 1, kind: "boarding", destination_id: null, label: "Praça da Matriz" },
      { id: "s2", position: 2, kind: "destination", destination_id: "t1", label: "Curitiba" } ],
    passengers: [ passenger() ], occupancy: { seats_used: 1, seats: 15, wheelchair_used: 0, wheelchair_spots: 1 }, series_id: null, log: null, ...o };
}

export function series(o: Partial<TransportSeries> = {}): TransportSeries {
  return { id: "se1", patient: { id: "pt2", display_name: "José Hemodiálise" }, destination: { id: "t1", name: "Curitiba" },
    provider_name: "Clínica do Rim", weekdays: [ 1, 3, 5 ], appointment_time: "07:00", departure_time: "05:30", starts_on: "2026-11-02",
    ends_on: null, wheelchair: false, companion: null, status: "active", suspended_from: null, ...o };
}

export function tfdCase(o: Partial<TfdCase> = {}): TfdCase {
  return { id: "c1", patient: { id: "pt1", display_name: "Maria da Silva", has_cpf: true, cns_masked: "*** **** **** 1234" }, status: "submitted",
    requested_by: { name: "Dra. Clara", cbo: "225125" }, consultation_id: "co1", regulation_request_id: null,
    report: { sigtap_code: "0405050372", sigtap_name: "FACOEMULSIFICACAO", cid10: "H25.1", justification: "Sem oftalmologia cirúrgica no município.",
      companion_indicated: false, companion_reason: null, transport_mode: "fleet" },
    destination: DEST_CURITIBA, provider: { id: "cp1", name: "Hospital de Olhos" }, appointment_at: null, inpatient: [], legs: [], allowances: [],
    payments: [], accountability: null, trip_passengers: [], events: [ { kind: "submitted", at: "2026-11-02T14:00:00-03:00", by_name: "Dra. Clara", reason: null } ], ...o };
}

export function tfdRow(o: Partial<TfdCaseRow> = {}): TfdCaseRow {
  return { id: "c1", patient: { display_name: "Maria da Silva" }, destination: { name: "Curitiba" }, status: "submitted", submitted_at: "2026-11-02T14:00:00-03:00", ...o };
}

export const SETTINGS: TransportSettings = { tfd_min_distance_km: 50, companion_rule: { max_age_child: 15, min_age_elderly: 60 },
  tfd_billing_health_unit_id: null, allowance_complements: {},
  allowance_codes: [ { code: "0803010125", name: "AJUDA DE CUSTO PARA ALIMENTAÇÃO/PERNOITE DE PACIENTE", kind: "meal_overnight" } ] };

export function trUser(roles: string[] = [ "transport_dispatcher" ], features: string[] = [ "sanitary_transport", "tfd", "clinical_record" ]): SessionUser {
  return sessionWith(roles, { features });
}
```

- [ ] **Step 2: Escreva os testes do cliente** `src/lib/api.transport.test.ts`

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

describe("cliente do transporte e TFD (módulo 30)", () => {
  it("viagens do dia: só data e veículo na URL", async () => {
    stub({ items: [] });
    await api.listTrips({ date: "2026-11-03", vehicleId: "v1" });
    expect(last().url).toBe("/transport/trips?date=2026-11-03&vehicle_id=v1");
    await api.listTrips({ date: "2026-11-03", vehicleId: null });
    expect(last().url).toBe("/transport/trips?date=2026-11-03");
  });
  it("motorista: a CNH vai no corpo, nunca na URL", async () => {
    stub({ id: "d1" }, 201);
    await api.createDriver({ user_id: "u-d1", license_number: "01234567890", license_category: "D", license_expires_on: "2028-05-10" });
    expect(last().url).toBe("/transport/drivers");
    expect(last().url).not.toContain("01234567890");
    expect(bodyOf().license_number).toBe("01234567890");
    expect((last().init.headers as Record<string, string>)["Content-Type"]).toBe("application/json");
  });
  it("busca de pessoa pela central é POST com o CPF no corpo", async () => {
    stub({ patient_id: "pt1", display_name: "Maria", has_cpf: true, cns_masked: null });
    await api.lookupTransportPerson("12345678909");
    expect(last().url).toBe("/transport/person_lookup");
    expect(last().init.method).toBe("POST");
    expect(bodyOf()).toEqual({ cpf: "12345678909" });
  });
  it("passageiro: acompanhante e documento só no corpo", async () => {
    stub({ id: "p9" }, 201);
    await api.addPassenger("tr1", { patient_id: "pt1", origin: "exception", exception_reason: "follow_up_return", stop_id: "s1",
      destination_id: "t1", takes_seat: true, wheelchair: false, companion: { name: "Joana", document: "MG1234567", reason: "idosa" } });
    expect(last().url).toBe("/transport/trips/tr1/passengers");
    expect(last().url).not.toMatch(/Joana|MG1234567/);
    expect(bodyOf().companion.document).toBe("MG1234567");
  });
  it("retirar passageiro: DELETE com motivo no corpo", async () => {
    stub("", 204, {});
    await api.removePassenger("tr1", "p1", "paciente desistiu da viagem");
    expect(last().init.method).toBe("DELETE");
    expect(bodyOf()).toEqual({ reason: "paciente desistiu da viagem" });
  });
  it("lista de passageiros em PDF por fetch, com o nome do arquivo do cabeçalho", async () => {
    stub("%PDF", 200, { "Content-Type": "application/pdf", "Content-Disposition": "attachment; filename=\"lista-2026-11-03.pdf\"" });
    const file = await api.fetchTripManifest("tr1");
    expect(last().url).toBe("/transport/trips/tr1/manifest.pdf");
    expect(file.filename).toBe("lista-2026-11-03.pdf");
  });
  it("presença em lote e fechamento", async () => {
    stub({ id: "tr1" });
    await api.recordAttendance("tr1", [ { passenger_id: "p1", attendance: "boarded" } ]);
    expect(last().url).toBe("/transport/my_trips/tr1/attendance");
    expect(bodyOf()).toEqual({ items: [ { passenger_id: "p1", attendance: "boarded" } ] });
    await api.startMyTrip("tr1");
    expect(bodyOf()).toEqual({});
  });
  it("TFD: prefill só com o id da consulta; laudo e decisão no corpo", async () => {
    stub({ patient_id: "pt1", cid10: null, sigtap_code: null });
    await api.getTfdPrefill("co1");
    expect(last().url).toBe("/tfd/prefill?consultation_id=co1");
    stub({ id: "c1" });
    await api.decideTfdCase("c1", { decision: "deny", reason: "destino a menos de 50 km" });
    expect(last().url).toBe("/tfd/cases/c1/decide");
    expect(bodyOf()).toEqual({ decision: "deny", reason: "destino a menos de 50 km" });
    await api.readTfdCase("c1");
    expect(last().url).toBe("/tfd/cases/c1/read");
    expect(last().init.method).toBe("POST");
  });
  it("fila da comissão: situação e cursor na URL", async () => {
    stub({ items: [], next_before: null });
    await api.listTfdCases({ status: "submitted", before: "abc" });
    expect(last().url).toBe("/tfd/cases?status=submitted&before=abc");
  });
  it("prestação de contas em base64, sem o nome do arquivo (D11)", async () => {
    stub({ id: "c1", status: "completed" });
    const file = new File([ new Uint8Array([ 137, 80, 78, 71 ]) ], "comprovante-maria.png", { type: "image/png" });
    await api.uploadTfdAccountability("c1", file);
    expect(last().url).toBe("/tfd/cases/c1/accountability");
    const sent = bodyOf();
    expect(sent.content_type).toBe("image/png");
    expect(typeof sent.data).toBe("string");
    expect(JSON.stringify(sent)).not.toContain("comprovante-maria");
  });
  it("painel: mês na URL; lt5 chega como string", async () => {
    stub({ trips: "lt5", occupancy_pct: 80, no_shows: 7, km: 1200, fuel_cents: 50000, tfd_by_status: { submitted: "lt5" }, tfd_paid_cents: 0, exceptions: 0 });
    const d = await api.getTransportDashboard("2026-11");
    expect(last().url).toBe("/transport/dashboard?month=2026-11");
    expect(d.trips).toBe("lt5");
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/api.transport.test.ts`
Expected: FAIL — `listTrips`, `createDriver` etc. não existem.

- [ ] **Step 4: Implemente a seção "Transporte sanitário e TFD (módulo 30; ADR 0048; contrato 2026-11-03 §2–§9)" no fim de `src/lib/api.ts`**

`TRANSPORT_BASE = import.meta.env.VITE_TRANSPORT_BASE || "/transport"` e `TFD_BASE = import.meta.env.VITE_TFD_BASE || "/tfd"`; `trPath`/`tfdPath` com `encodeURIComponent` por parte (molde de `vacPath`); `trWrite(method, body = {})`. Listas em envelope `{ items }` (ausente = vazio). `fetchTripManifest` usa o mesmo corpo de `binaryFetch` do 19b (`Accept: application/pdf`; recusa vira `ApiError` com o corpo) — mova `binaryFetch` para ser reusável sem mudar o comportamento dele. `uploadTfdAccountability` lê com `readBase64` e envia `{ content_type: file.type, data }`. Parâmetros vazios ou nulos nunca vão na URL (`withQuery`). Comentário de cabeçalho da seção: "Na URL só ids, data, mês, período, situação e cursor; CPF, CNH, nome, documento do acompanhante, CID, justificativa e arquivo vão no corpo." Acrescente os campos de D13 aos tipos da regulação e `"transport"` ao `TimelineKind` com o comentário "Módulo 30 (D12): viagem e TFD, detail_ref nulo".

- [ ] **Step 5: Interruptores e proxy**

`src/lib/features.ts`: `FeatureKey` ganha `"sanitary_transport" | "tfd"`; `FEATURE_LABEL` ganha, com o comentário "Módulo 30 (ADR 0048): tfd exige sanitary_transport e clinical_record", `sanitary_transport: "Transporte sanitário"` e `tfd: "Tratamento Fora de Domicílio (TFD)"`. Em `src/lib/features.test.ts`, no teste "rótulo conhecido em português", acrescente:

```ts
expect(featureLabel("sanitary_transport")).toBe("Transporte sanitário");
expect(featureLabel("tfd")).toBe("Tratamento Fora de Domicílio (TFD)");
```

`vite.config.ts`: no comentário, `/transport → transporte sanitário (módulo 30: frota, viagens, séries, motorista, painel)` e `/tfd → TFD (módulo 30: laudo, comissão, trechos, ajuda de custo, pagamento)`; no proxy, `"^/transport(/|$)": proxy(TARGET)` e `"^/tfd(/|$)": proxy(TARGET)` (regex: não casam outro caminho que só comece com as letras).

- [ ] **Step 6: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/api.transport.test.ts src/lib/features.test.ts src/lib/api.signature.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS (o teste do 19b prova que `binaryFetch` não mudou); `tsc` sem erro.

- [ ] **Step 7: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.transport.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/transportFixtures.ts
$G commit -m "feat: add the sanitary transport and TFD client" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela da central e do motorista (`src/lib/transport.ts`)

**Files:**
- Create: `src/lib/transport.ts`
- Test: `src/lib/transport.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `hasFeature`, `featureDisabledKey`, `featureLabel` (`features.ts`); `pharmacyRefusals`, `Problem`, `Refusal` (`pharmacy.ts`); `NOMINAL_QUERY`, `STATIC_QUERY` (`careLines.ts`); `cityLocalIso`, `cityDateFormat` (`format.ts`); `cnesProblem` (`regulation.ts`).
- Produces (exportadas):
  - Frases: `TRANSPORT_DISABLED = "o transporte sanitário está desligado nesta cidade"`, `TRANSPORT_NO_ROLE = "seu papel não permite usar o transporte sanitário"`, `SMALL_COUNT = "menos de 5"`, `NO_CLINICAL_NOTE = "A lista não leva diagnóstico, procedimento nem especialidade."`.
  - Chaves de query: `TR_VEHICLES_KEY`, `TR_DRIVERS_KEY`, `TR_DESTINATIONS_KEY`, `TR_TRIPS_KEY`, `TR_SERIES_KEY`, `TR_PENDING_KEY`, `TR_NO_SHOWS_KEY`, `TR_MY_TRIPS_KEY`, `TR_SETTINGS_KEY`, `TR_DASHBOARD_KEY`; reexporta `STATIC_QUERY`, `NOMINAL_QUERY`.
  - Portão: `canUseTransport(user): boolean` (`!operator && sanitary_transport`); `transportRoles(user): { dispatcher; driver; admin; committee; professional }`; `isDriverOnly(user): boolean` (não operador, tem `driver` e nenhum outro papel).
  - Rótulos (`pick`, código novo cru): `tripStatusLabel`, `tripStatusTone`, `vehicleKindLabel`, `originLabel`, `exceptionReasonLabel`, `attendanceLabel`, `expenseKindLabel`, `weekdaysLabel(days: number[]): string`, `licenseText(d: TransportDriver): string`, `licenseTone(d): string`.
  - Dinheiro e números: `parseReais(raw: string): number | null` (centavos), `fmtCents(cents: number): string`, `parseLiters(raw: string): number | null`, `fmtCount(v: TransportCount): string`.
  - Frota: `normalizePlate(raw): string`, `vehicleProblems(d: VehicleDraft): Problem[]`, `vehicleBody(d): VehicleInput`, `driverProblems(d: DriverDraft, creating: boolean): Problem[]`, `destinationProblems(d: DestinationDraft): Problem[]`; tipos `VehicleDraft`, `DriverDraft`, `DestinationDraft` (campos como string de formulário).
  - Viagem: `TripDraft { date; departureTime; returnDate; returnTime; vehicleId; driverId; boarding: string[]; destinationIds: string[] }`, `newTripDraft(date: string): TripDraft`, `tripDraftFrom(t: Trip): TripDraft`, `driverFitProblems(driver: TransportDriver | undefined, vehicle: Pick<TransportVehicle, "seats"> | undefined, date: string): Problem[]`, `tripProblems(d: TripDraft, ctx: { vehicles: TransportVehicle[]; drivers: TransportDriver[]; confirm: boolean }): Problem[]`, `tripBody(d: TripDraft, stops?: TripStop[]): TripInput`, `occupancyText(t: Trip): string`.
  - Passageiro: `PassengerDraft`, `passengerDraftFrom(item: ScheduledCareItem, boardingStopId: string): PassengerDraft`, `passengerProblems(d: PassengerDraft, t: Trip): Problem[]`, `passengerBody(d): PassengerInput`.
  - Série: `SeriesDraft`, `seriesProblems(d, today): Problem[]`, `seriesBody(d): SeriesInput`.
  - Fechamento: `CloseDraft { odometerStart; odometerEnd; fuel: { liters; amount; station }[]; expenses: { kind; amount; note }[] }`, `pendingAttendance(t: Trip): number`, `closeProblems(d: CloseDraft, t: Trip): Problem[]`, `closeBody(d): TripCloseInput`.
  - Recusas: `transportRefusals(err): Refusal[]`, `transportError(err): string`.

- [ ] **Step 1: Escreva os testes** `src/lib/transport.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  canUseTransport, closeProblems, driverFitProblems, fmtCents, fmtCount, isDriverOnly, licenseText, newTripDraft, normalizePlate, occupancyText,
  parseReais, passengerProblems, tripBody, tripProblems, transportRefusals, vehicleProblems, weekdaysLabel, seriesProblems, originLabel
} from "./transport";
import { CAR, DRIVER_B, DRIVER_D, DRIVER_EXPIRED, DRIVER_EXPIRING, TODAY30, VAN, passenger, trip, trUser } from "../test/transportFixtures";

const codes = (list: { path: string }[]) => list.map((p) => p.path);

describe("portão e papéis", () => {
  it("interruptor e operador", () => {
    expect(canUseTransport(trUser())).toBe(true);
    expect(canUseTransport(trUser([ "transport_dispatcher" ], []))).toBe(false);
    expect(canUseTransport({ ...trUser(), operator: true })).toBe(false);
  });
  it("isDriverOnly: só com driver e nada mais", () => {
    expect(isDriverOnly(trUser([ "driver" ]))).toBe(true);
    expect(isDriverOnly(trUser([ "driver", "viewer" ]))).toBe(false);
    expect(isDriverOnly(trUser([]))).toBe(false);
    expect(isDriverOnly(null)).toBe(false);
  });
});

describe("frota", () => {
  it("placa antiga e Mercosul; hífen e minúscula normalizam; outra coisa recusa", () => {
    expect(normalizePlate("abc-1234")).toBe("ABC1234");
    const base = { plate: "ABC1D23", renavam: "12345678901", model: "Sprinter", kind: "van", renem_code: "", seats: "15", wheelchair_spots: "1", cnes: "", active: true };
    expect(vehicleProblems(base)).toEqual([]);
    expect(vehicleProblems({ ...base, plate: "abc-1234" })).toEqual([]);
    expect(codes(vehicleProblems({ ...base, plate: "AB12345", renavam: "123", seats: "0", cnes: "12" }))).toEqual([ "plate", "renavam", "seats", "cnes" ]);
  });
});

describe("CNH e categoria (Review Focus 3)", () => {
  it("driverProblems: vencida, categoria, a vencer não bloqueia", () => {
    expect(driverFitProblems(DRIVER_EXPIRED, VAN, TODAY30).map((p) => p.text)).toEqual([ "a CNH deste motorista está vencida na data da viagem" ]);
    expect(driverFitProblems(DRIVER_B, VAN, TODAY30).map((p) => p.text)).toEqual([ "veículo com mais de 8 lugares exige CNH D ou E" ]);
    expect(driverFitProblems(DRIVER_B, CAR, TODAY30)).toEqual([]);
    expect(driverFitProblems(DRIVER_EXPIRING, VAN, TODAY30)).toEqual([]);
    // Vence antes da viagem marcada para depois: conta a data da viagem, não hoje.
    expect(driverFitProblems(DRIVER_EXPIRING, VAN, "2026-11-25")).toHaveLength(1);
  });
  it("texto da CNH", () => {
    expect(licenseText(DRIVER_EXPIRING)).toBe("CNH vence em 20/11/2026");
    expect(licenseText(DRIVER_EXPIRED)).toBe("CNH vencida em 30/10/2026");
    expect(licenseText(DRIVER_D)).toBe("CNH D válida até 10/05/2028");
  });
});

describe("viagem", () => {
  const ctx = { vehicles: [ VAN, CAR ], drivers: [ DRIVER_D, DRIVER_B, DRIVER_EXPIRED ], confirm: false };
  it("retorno antes da saída recusa; planejada sem escala é aceita; confirmar exige os dois", () => {
    const d = { ...newTripDraft(TODAY30), departureTime: "05:30", returnTime: "05:00", boarding: [ "Praça da Matriz" ], destinationIds: [ "t1" ] };
    expect(codes(tripProblems(d, ctx))).toEqual([ "return_at" ]);
    const ok = { ...d, returnTime: "18:00" };
    expect(tripProblems(ok, ctx)).toEqual([]);
    expect(codes(tripProblems(ok, { ...ctx, confirm: true }))).toEqual([ "vehicle_id", "driver_id" ]);
    expect(codes(tripProblems({ ...ok, vehicleId: "v1", driverId: "d2" }, { ...ctx, confirm: true }))).toEqual([ "driver_id" ]);
  });
  it("corpo: hora de parede da cidade, retorno no dia seguinte, paradas em ordem", () => {
    const d = { ...newTripDraft(TODAY30), departureTime: "22:00", returnDate: "2026-11-04", returnTime: "06:00", boarding: [ "Praça", "Distrito" ], destinationIds: [ "t1" ] };
    const body = tripBody(d);
    expect(body.departure_at).toBe("2026-11-03T22:00:00-03:00");
    expect(body.return_at).toBe("2026-11-04T06:00:00-03:00");
    expect(body.stops.map((s) => [ s.position, s.kind, s.label ])).toEqual([ [ 1, "boarding", "Praça" ], [ 2, "boarding", "Distrito" ], [ 3, "destination", "" ] ]);
    expect(body.stops[2].destination_id).toBe("t1");
    expect("vehicle_id" in body).toBe(false);
  });
  it("ocupação", () => {
    expect(occupancyText(trip())).toBe("1 de 15 lugares · 0 de 1 cadeira");
  });
});

describe("passageiro", () => {
  const full = trip({ occupancy: { seats_used: 15, seats: 15, wheelchair_used: 1, wheelchair_spots: 1 } });
  const base = { patientId: "pt9", origin: "exception", originId: "", exceptionReason: "", exceptionNote: "", stopId: "s1", destinationId: "t1",
    providerName: "", address: "", appointmentAt: "", takesSeat: true, wheelchair: false, companion: null } as const;
  it("sem origem nem motivo; outro sem texto; lotado; cadeira", () => {
    expect(codes(passengerProblems({ ...base }, trip()))).toEqual([ "exception_reason" ]);
    expect(codes(passengerProblems({ ...base, exceptionReason: "other", exceptionNote: "curto" }, trip()))).toEqual([ "exception_note" ]);
    expect(codes(passengerProblems({ ...base, exceptionReason: "follow_up_return", wheelchair: true }, full))).toEqual([ "takes_seat", "wheelchair" ]);
    expect(passengerProblems({ ...base, exceptionReason: "follow_up_return", takesSeat: false }, full)).toEqual([]);
  });
  it("acompanhante sem nome recusa", () => {
    expect(codes(passengerProblems({ ...base, exceptionReason: "legacy_patient", companion: { name: "", document: "", reason: "" } }, trip()))).toEqual([ "companion.name" ]);
  });
});

describe("dinheiro e contagens", () => {
  it("parseReais aceita vírgula, ponto e milhar; recusa três casas e negativo", () => {
    expect(parseReais("12,34")).toBe(1234);
    expect(parseReais("12.34")).toBe(1234);
    expect(parseReais("1.234,56")).toBe(123456);
    expect(parseReais("1234")).toBe(123400);
    expect(parseReais("12,345")).toBeNull();
    expect(parseReais("-5")).toBeNull();
    expect(parseReais("")).toBeNull();
  });
  it("fmtCents e lt5", () => {
    expect(fmtCents(123456).replace(/\s/g, " ")).toBe("R$ 1.234,56");
    expect(fmtCount("lt5")).toBe("menos de 5");
    expect(fmtCount(12)).toBe("12");
  });
});

describe("fechamento", () => {
  it("odômetro e presença pendente", () => {
    const t = trip({ status: "in_progress" });
    const d = { odometerStart: "1000", odometerEnd: "900", fuel: [], expenses: [] };
    expect(codes(closeProblems(d, t))).toEqual([ "odometer_end", "attendance" ]);
    const ok = trip({ status: "in_progress", passengers: [ passenger({ attendance: "boarded" }) ] });
    expect(closeProblems({ ...d, odometerEnd: "1220", fuel: [ { liters: "40,5", amount: "250,00", station: "Posto Central" } ] }, ok)).toEqual([]);
    expect(codes(closeProblems({ ...d, odometerEnd: "1220", fuel: [ { liters: "0", amount: "x", station: "" } ] }, ok))).toEqual([ "fuel[0].liters", "fuel[0].amount_cents", "fuel[0].station" ]);
  });
  it("acompanhante também precisa de presença", () => {
    const t = trip({ passengers: [ passenger({ attendance: "boarded", companion: { name: "Joana", document_last4: null }, companion_attendance: "pending" }) ] });
    expect(codes(closeProblems({ odometerStart: "1", odometerEnd: "2", fuel: [], expenses: [] }, t))).toEqual([ "attendance" ]);
  });
});

describe("série e rótulos", () => {
  it("série: dias, horários, fim antes do início", () => {
    const d = { patientId: "pt2", destinationId: "t1", providerName: "", weekdays: [], appointmentTime: "07:00", departureTime: "08:00",
      startsOn: TODAY30, endsOn: "2026-11-01", wheelchair: false, companion: null };
    expect(codes(seriesProblems(d, TODAY30))).toEqual([ "weekdays", "departure_time", "ends_on" ]);
    expect(weekdaysLabel([ 1, 3, 5 ])).toBe("seg, qua, sex");
  });
  it("código desconhecido aparece cru", () => {
    expect(originLabel("regulation")).toBe("regulação");
    expect(originLabel("novo_codigo")).toBe("novo_codigo");
  });
});

describe("recusas do api", () => {
  it("todas as recusas, traduzidas", () => {
    const err = new ApiError(422, { error: "invalid", problems: [ { path: "vehicle_id", error: "vehicle_overlap" }, { path: "driver_id", error: "driver_overlap" } ] }, "422");
    expect(transportRefusals(err).map((r) => r.text)).toEqual([
      "o veículo já está em outra viagem nesse horário", "o motorista já está em outra viagem nesse horário"
    ]);
  });
  it("feature_disabled e código desconhecido", () => {
    expect(transportRefusals(new ApiError(403, { error: "feature_disabled", feature: "sanitary_transport" }, "403"))[0].text).toBe("o transporte sanitário está desligado nesta cidade");
    expect(transportRefusals(new ApiError(422, { error: "novo_erro" }, "422"))[0].text).toBe("recusado pelo servidor (novo_erro)");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/transport.test.ts`
Expected: FAIL — módulo `./transport` não existe.

- [ ] **Step 3: Implemente `src/lib/transport.ts`**

Cabeçalho: "Regras de tela do módulo 30 (transporte sanitário; contrato 2026-11-03 §1–§5, §9) sem React. Sobreposição, lotação, CNH, regra do acompanhante, avisos e imutabilidade são do api; a tela só não envia o que já sabe que será recusado." Valores que a spec e o contrato fixam:
- Placa: `/^[A-Z]{3}[0-9][A-Z0-9][0-9]{2}$/` depois de `normalizePlate` (maiúscula, sem hífen e espaço). RENAVAM 11 dígitos. CNES 7 dígitos (opcional; reusa `cnesProblem`). `seats` inteiro 1–60; `wheelchair_spots` inteiro 0–10. Modelo 2–80. CNH: `license_number` 11 dígitos, obrigatório só ao criar (`creating`). IBGE 7 dígitos; `distance_km` inteiro ≥ 0, e > 0 quando o nome não é da própria cidade não é verificável — só inteiro ≥ 0.
- `driverFitProblems`: `license_expires_on < date` → path `driver_id`, "a CNH deste motorista está vencida na data da viagem"; `seats > 8` e categoria fora de `D`/`E` → path `driver_id`, "veículo com mais de 8 lugares exige CNH D ou E" (CTB art. 143, decisão 6). `tripProblems` devolve um problema por campo (o primeiro de cada `path`), na ordem `return_at`, `boarding`, `destinations`, `vehicle_id`, `driver_id`; com `confirm`, veículo e motorista são obrigatórios ("escolha o veículo" / "escolha o motorista"); veículo inativo → "este veículo está inativo".
- `tripBody`: `cityLocalIso(\`${date}T${time}\`)`; paradas = embarques em ordem (posição 1..n, `kind: "boarding"`, `label`), depois um destino por `destinationIds` (`kind: "destination"`, `destination_id`, `label: ""` — o api põe o nome); com `stops` da viagem já existente, mantém o `id` da parada de mesmo `kind` e `label`/`destination_id` (D3). `vehicle_id`/`driver_id` só quando escolhidos.
- `passengerProblems`: `origin === "exception"` sem `exceptionReason` → `exception_reason` ("informe por que o passageiro vai sem atendimento marcado"); `other` com nota de menos de 10 caracteres → `exception_note`; `takesSeat` com `seats_used >= seats` → `takes_seat` ("a viagem está lotada"); `wheelchair` com `wheelchair_used >= wheelchair_spots` → `wheelchair` ("não há vaga de cadeirante nesta viagem"); acompanhante com nome fora de 3–120 → `companion.name`; documento > 20 → `companion.document`. **Não** confere a regra de idade do acompanhante (o api tem a idade; recusa `companion_not_allowed`, Task 9).
- `parseReais`: aceita `^\d{1,3}(\.\d{3})*(,\d{1,2})?$`, `^\d+(,\d{1,2})?$` e `^\d+(\.\d{1,2})?$`; vazio, negativo, três casas → `null`. `fmtCents`: `Intl.NumberFormat("pt-BR", { style: "currency", currency: "BRL" })`. `parseLiters`: até duas casas, > 0.
- `closeProblems`: inteiros ≥ 0; `odometerEnd < odometerStart` → `odometer_end` ("o odômetro de chegada é menor que o de saída"); `pendingAttendance(t) > 0` → path `attendance`, "marque embarcou ou faltou para todos, inclusive acompanhantes"; abastecimento: `fuel[i].liters`, `fuel[i].amount_cents`, `fuel[i].station` (1–120); despesa: `expenses[i].kind`, `expenses[i].amount_cents`, `expenses[i].note` (≤ 120; obrigatório em `other`).
- `seriesProblems`: `weekdays` vazio; `departure_time >= appointment_time` → `departure_time` ("a saída precisa ser antes do horário do atendimento"); `ends_on < starts_on` → `ends_on`; `starts_on` obrigatório. Semana ISO (1 = segunda … 7 = domingo, D5); `weekdaysLabel` em `seg ter qua qui sex sáb dom`.
- Rótulos: viagem `planned` "planejada" (info), `confirmed` "confirmada" (ok), `in_progress` "em andamento" (info), `done` "concluída" (neutral), `cancelled` "cancelada" (down); veículo `van` "Van", `minibus` "Micro-ônibus", `bus` "Ônibus", `car` "Carro", `ambulance_a` "Ambulância tipo A"; origem `regulation` "regulação", `appointment` "agenda da cidade", `tfd` "TFD", `series` "série recorrente", `exception` "sem atendimento marcado"; motivo `follow_up_return` "retorno", `legacy_patient` "paciente antigo sem processo", `scheduling_pending` "agendamento em andamento", `other` "outro motivo"; presença `pending` "a marcar", `boarded` "embarcou", `no_show` "faltou"; despesa `toll` "pedágio", `parking` "estacionamento", `meal` "alimentação", `other` "outra". `licenseText`: `valid` → "CNH <cat> válida até dd/mm/aaaa"; `expiring` → "CNH vence em dd/mm/aaaa"; `expired` → "CNH vencida em dd/mm/aaaa"; `licenseTone`: ok / warn / down.
- `transportRefusals`: molde de `vaccinationRefusals` (feature_disabled primeiro; depois `pharmacyRefusals` com o vocabulário do 30; código desconhecido → "recusado pelo servidor (<código>)"). Frases: `vehicle_overlap` "o veículo já está em outra viagem nesse horário", `driver_overlap` "o motorista já está em outra viagem nesse horário", `driver_license_expired` "a CNH deste motorista está vencida na data da viagem", `driver_license_category` "veículo com mais de 8 lugares exige CNH D ou E", `vehicle_inactive` "este veículo está inativo", `return_before_departure` "o retorno precisa ser depois da saída", `seats_full` "a viagem está lotada", `wheelchair_full` "não há vaga de cadeirante nesta viagem", `origin_required` "informe o atendimento marcado ou o motivo da viagem sem atendimento", `companion_not_allowed` "acompanhante fora da regra da cidade — informe o motivo", `patient_already_on_trip` "este paciente já está nesta viagem", `origin_mismatch` "o atendimento escolhido não é deste paciente", `odometer_invalid` "confira o odômetro de saída e de chegada", `attendance_pending` "marque embarcou ou faltou para todos, inclusive acompanhantes", `stop_in_use` "esta parada tem passageiro — mude o passageiro de parada antes", `user_not_driver` "esta pessoa não tem o papel de motorista na Equipe", `trip_not_today` "a viagem só começa no dia dela", `invalid_transition` "a viagem mudou de situação — recarregue", `missing_role` "seu papel não permite esta ação".

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/transport.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/transport.ts src/lib/transport.test.ts
$G commit -m "feat: add sanitary transport screen rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Regras de tela do TFD (`src/lib/tfd.ts`)

**Files:**
- Create: `src/lib/tfd.ts`
- Test: `src/lib/tfd.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `canUseTransport`, `parseReais`, `transportRefusals` (Task 2); `hasFeature`; `parseCityLocal`, `cityLocalIso`.
- Produces (exportadas):
  - Frases: `TFD_DISABLED = "o TFD está desligado nesta cidade"`, `TFD_NO_ROLE = "seu papel não permite usar o TFD"`, `APPOINTMENT_REQUIRED = "marque o atendimento no destino antes de autorizar"`, `FILE_TOO_BIG = "o comprovante passa de 10 MB"`, `FILE_TYPE = "o comprovante precisa ser PDF, JPG ou PNG"`, `READ_TRAIL_NOTE = "Abrir o laudo fica registrado na trilha de acesso."`.
  - Chaves: `TFD_CASES_KEY`, `TFD_MY_CASES_KEY`, `TFD_OPTIONS_KEY`.
  - Portão: `canUseTfd(user)` (`canUseTransport && tfd && clinical_record`); `tfdRoles(user): { committee; professional; admin }`; `canRequestTfd(user, cbo: string | null | undefined): boolean` (`canUseTfd`, `health_professional` e CBO começando por `225`).
  - Rótulos: `tfdStatusLabel`, `tfdStatusTone`, `legModeLabel`, `directionLabel`, `allowanceKindLabel`, `paymentMethodLabel`, `transportModeLabel`, `decisionLabel`.
  - Laudo: `ReportDraft { sigtapCode; cid10; justification; companionIndicated; companionReason; transportMode; destinationId; providerId }`, `reportDraftFrom(prefill: TfdPrefill | null, c?: TfdCase): ReportDraft`, `reportProblems(d: ReportDraft, o: TfdOptions): Problem[]`, `caseBody(d, ctx: { patientId; consultationId?; regulationRequestId? }): TfdCaseInput`, `canEditCase(c: TfdCaseRow | TfdCase): boolean`.
  - Comissão: `decisionProblems(decision: "authorize" | "return" | "deny", reason: string, c: TfdCase): Problem[]`, `appointmentProblems(value: string, now: Date): Problem[]`, `inpatientProblems(from: string, to: string): Problem[]`.
  - Dinheiro: `LegDraft { direction; mode; date; tripPassengerId; amount; carrier; companion }`, `legProblems(d: LegDraft, c: TfdCase): Problem[]`, `legBody(d): TfdLegInput`, `PaymentDraft { paidOn; amount; method; receiptNumber }`, `paymentProblems(d, today): Problem[]`, `paymentBody(d): TfdPaymentInput`, `allowanceTotal(list: TfdAllowance[]): number`, `paidTotal(list: TfdPayment[]): number`, `accountabilityFileProblem(file: File): string | null`.
  - Recusas: `tfdRefusals(err): Refusal[]`, `tfdError(err): string`.

- [ ] **Step 1: Escreva os testes** `src/lib/tfd.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  accountabilityFileProblem, allowanceTotal, appointmentProblems, canRequestTfd, canUseTfd, decisionProblems, legProblems, paymentProblems,
  reportProblems, tfdRefusals, tfdStatusLabel
} from "./tfd";
import { DEST_CURITIBA, DEST_METRO, DEST_NEAR, NOW30, TODAY30, tfdCase, trUser } from "../test/transportFixtures";

const codes = (list: { path: string }[]) => list.map((p) => p.path);
const OPTIONS = { destinations: [ DEST_CURITIBA, DEST_NEAR, DEST_METRO ], providers: [], min_distance_km: 50 };
const report = { sigtapCode: "0405050372", cid10: "H25.1", justification: "Sem oftalmologia cirúrgica no município.", companionIndicated: false,
  companionReason: "", transportMode: "fleet", destinationId: "t1", providerId: "" };

describe("portão do TFD", () => {
  it("exige tfd, sanitary_transport e clinical_record", () => {
    expect(canUseTfd(trUser([ "tfd_committee" ]))).toBe(true);
    expect(canUseTfd(trUser([ "tfd_committee" ], [ "tfd", "clinical_record" ]))).toBe(false);
    expect(canUseTfd(trUser([ "tfd_committee" ], [ "tfd", "sanitary_transport" ]))).toBe(false);
  });
  it("pedir TFD: médico (CBO 225*) com health_professional", () => {
    expect(canRequestTfd(trUser([ "health_professional" ]), "225125")).toBe(true);
    expect(canRequestTfd(trUser([ "health_professional" ]), "223505")).toBe(false);
    expect(canRequestTfd(trUser([ "tfd_committee" ]), "225125")).toBe(false);
    expect(canRequestTfd(trUser([ "health_professional" ]), null)).toBe(false);
  });
});

describe("laudo", () => {
  it("completo passa", () => { expect(reportProblems(report, OPTIONS)).toEqual([]); });
  it("menos de 50 km e região metropolitana recusam antes de enviar", () => {
    expect(reportProblems({ ...report, destinationId: "t2" }, OPTIONS).map((p) => p.text)).toEqual([ "destino a menos de 50 km da sede — não cabe TFD" ]);
    expect(reportProblems({ ...report, destinationId: "t3" }, OPTIONS).map((p) => p.text)).toEqual([ "destino na mesma região metropolitana — não cabe TFD" ]);
    expect(reportProblems({ ...report, destinationId: "t2" }, { ...OPTIONS, min_distance_km: 25 })).toEqual([]);
  });
  it("CID, justificativa, procedimento e motivo do acompanhante", () => {
    expect(codes(reportProblems({ ...report, cid10: "", justification: "curta", sigtapCode: "", companionIndicated: true }, OPTIONS)))
      .toEqual([ "report.sigtap_code", "report.cid10", "report.justification", "report.companion_reason" ]);
  });
});

describe("decisão (Review Focus 4)", () => {
  it("autorizar sem atendimento marcado recusa; devolver e recusar não exigem", () => {
    expect(decisionProblems("authorize", "laudo completo e destino confirmado", tfdCase()).map((p) => p.text)).toEqual([ "marque o atendimento no destino antes de autorizar" ]);
    expect(decisionProblems("authorize", "laudo completo e destino confirmado", tfdCase({ appointment_at: "2026-11-20T08:00:00-03:00" }))).toEqual([]);
    expect(decisionProblems("return", "falta o exame de vista", tfdCase())).toEqual([]);
    expect(codes(decisionProblems("deny", "curto", tfdCase()))).toEqual([ "reason" ]);
  });
  it("atendimento no passado recusa", () => {
    expect(codes(appointmentProblems("2026-11-02T08:00", NOW30))).toEqual([ "appointment_at" ]);
    expect(appointmentProblems("2026-11-20T08:00", NOW30)).toEqual([]);
  });
});

describe("trechos, pagamento e arquivo", () => {
  it("frota exige o passageiro da viagem; passagem exige valor e empresa", () => {
    const c = tfdCase({ status: "authorized", appointment_at: "2026-11-20T08:00:00-03:00" });
    expect(codes(legProblems({ direction: "outbound", mode: "fleet", date: "2026-11-20", tripPassengerId: "", amount: "", carrier: "", companion: false }, c))).toEqual([ "trip_passenger_id" ]);
    expect(codes(legProblems({ direction: "outbound", mode: "bus_ticket", date: "2026-11-20", tripPassengerId: "", amount: "", carrier: "", companion: false }, c))).toEqual([ "ticket.amount_cents", "ticket.carrier" ]);
  });
  it("pagamento: data futura, valor e recibo", () => {
    expect(codes(paymentProblems({ paidOn: "2026-11-04", amount: "0", method: "pix", receiptNumber: "" }, TODAY30))).toEqual([ "paid_on", "amount_cents", "receipt_number" ]);
    expect(paymentProblems({ paidOn: TODAY30, amount: "1.234,56", method: "pix", receiptNumber: "R-77" }, TODAY30)).toEqual([]);
  });
  it("arquivo: 10 MB e tipo", () => {
    const big = new File([ new Uint8Array(10 * 1024 * 1024 + 1) ], "a.pdf", { type: "application/pdf" });
    const doc = new File([ "x" ], "a.docx", { type: "application/vnd.openxmlformats-officedocument.wordprocessingml.document" });
    const png = new File([ "x" ], "a.png", { type: "image/png" });
    expect(accountabilityFileProblem(big)).toBe("o comprovante passa de 10 MB");
    expect(accountabilityFileProblem(doc)).toBe("o comprovante precisa ser PDF, JPG ou PNG");
    expect(accountabilityFileProblem(png)).toBeNull();
  });
  it("totais", () => {
    expect(allowanceTotal([ { id: "a", person: "patient", kind: "meal_day", sigtap_code: "x", quantity: 1, table_cents: 840, complement_cents: 1000, total_cents: 1840 },
      { id: "b", person: "companion", kind: "meal_day", sigtap_code: "x", quantity: 1, table_cents: 840, complement_cents: 0, total_cents: 840 } ])).toBe(2680);
  });
});

describe("rótulos e recusas", () => {
  it("situação e código cru", () => {
    expect(tfdStatusLabel("returned")).toBe("devolvido para complementar");
    expect(tfdStatusLabel("x_novo")).toBe("x_novo");
  });
  it("recusas do envio do laudo, todas", () => {
    const err = new ApiError(422, { error: "invalid", problems: [ { path: "destination_id", error: "distance_below_minimum" }, { path: "report.sigtap_code", error: "primary_care_procedure" } ] }, "422");
    expect(tfdRefusals(err).map((r) => r.text)).toEqual([ "destino a menos da distância mínima da cidade — não cabe TFD", "procedimento da atenção básica — não cabe TFD" ]);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/tfd.test.ts`
Expected: FAIL — módulo `./tfd` não existe.

- [ ] **Step 3: Implemente `src/lib/tfd.ts`**

Cabeçalho: "Regras de tela do TFD (módulo 30; Portaria SAS/MS 55/1999; contrato §6). Distância, região metropolitana, procedimento da atenção básica, cálculo da ajuda de custo, BPA e trilha são do api; a tela espelha as recusas que já conhece." Valores:
- `reportProblems`, na ordem: `report.sigtap_code` (10 dígitos) "escolha o procedimento (SIGTAP)"; `report.cid10` "informe o CID-10"; `report.justification` 20–2000 "explique por que o tratamento não pode ser feito na cidade (20 a 2000 caracteres)"; `report.companion_reason` 10–500 quando `companionIndicated`; `destination_id` obrigatório; destino `distance_km < min_distance_km` → "destino a menos de <min> km da sede — não cabe TFD"; `metropolitan` → "destino na mesma região metropolitana — não cabe TFD" (decisão 16; o limite vem de `TfdOptions.min_distance_km`, D8). A lista de grupos da atenção básica é só do api (`primary_care_procedure`).
- `canEditCase`: `status` em `draft` ou `returned`.
- `decisionProblems`: motivo 10–500 em todas (path `reason`); `authorize` sem `appointment_at` → path `appointment_at`, `APPOINTMENT_REQUIRED`.
- `appointmentProblems`: `parseCityLocal(value)` nulo → "informe data e hora"; antes de `now` → "o atendimento não pode estar no passado". `inpatientProblems`: `from` obrigatório; `to` (opcional) ≥ `from`.
- `legProblems`: `date` obrigatória; volta antes da última ida do processo → `date` "a volta precisa ser no dia da ida ou depois"; `fleet` sem `tripPassengerId` → `trip_passenger_id` "escolha a viagem da frota em que o paciente está"; `bus_ticket`/`air`/`river` sem valor válido (`parseReais`) → `ticket.amount_cents`, sem empresa (1–120) → `ticket.carrier`.
- `paymentProblems`: `paid_on` > hoje → "a data do pagamento não pode ser futura"; valor ≤ 0 ou inválido → `amount_cents`; recibo 1–40 → `receipt_number`. `method` só dos quatro.
- `accountabilityFileProblem`: tipos `application/pdf`, `image/jpeg`, `image/png`; tamanho > 10 MB (10 × 1024 × 1024 bytes) → `FILE_TOO_BIG`; o tamanho é conferido antes do tipo.
- Rótulos: `draft` "rascunho", `submitted` "aguardando a comissão" (info), `returned` "devolvido para complementar" (warn), `authorized` "autorizado" (ok), `denied` "recusado" (down), `scheduled` "viagem marcada" (ok), `travelling` "em viagem" (info), `completed` "concluído" (neutral), `cancelled` "cancelado" (down); modal `fleet` "frota da cidade", `bus_ticket` "passagem de ônibus", `air` "passagem aérea", `river` "passagem fluvial"; `outbound` "ida", `return` "volta"; ajuda `meal_overnight` "alimentação com pernoite", `meal_day` "alimentação sem pernoite", `ground_km` "deslocamento terrestre (50 km)", `air_miles` "deslocamento aéreo (200 milhas)", `river_miles` "deslocamento fluvial (27 milhas)"; pagamento `cash` "dinheiro", `transfer` "transferência", `pix` "Pix", `check` "cheque"; decisão `authorize` "Autorizar", `return` "Devolver para complementar", `deny` "Recusar".
- `tfdRefusals`: molde de `transportRefusals` (feature_disabled de `tfd` → `TFD_DISABLED`; das outras chaves → "<rótulo> está desligado nesta cidade"), com as frases: `distance_below_minimum` "destino a menos da distância mínima da cidade — não cabe TFD", `metropolitan_destination` "destino na mesma região metropolitana — não cabe TFD", `primary_care_procedure` "procedimento da atenção básica — não cabe TFD", `cid_required` "informe o CID-10", `justification_required` "explique por que o tratamento não pode ser feito na cidade", `appointment_required` `APPOINTMENT_REQUIRED`, `not_author` "só o médico que pediu edita este laudo", `invalid_transition` "o processo mudou de situação — recarregue", `trip_passenger_mismatch` "esta viagem não é deste paciente", `file_too_large` `FILE_TOO_BIG`, `invalid_file_type` `FILE_TYPE`, `cbo_not_allowed` "só médico (CBO 225) pede TFD"; o resto cai em `transportRefusals`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/tfd.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/tfd.ts src/lib/tfd.test.ts
$G commit -m "feat: add TFD screen rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Papéis na Equipe, grupos "Transporte" e "TFD", shell do motorista e portões

**Files:**
- Modify: `src/lib/team.ts`, `src/lib/team.test.ts`, `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`
- Create: `src/modules/transport/TransportGate.tsx`; telas-portão `src/modules/TransportTrips.tsx`, `TransportSeries.tsx`, `TransportPending.tsx`, `TransportFleet.tsx`, `TransportMyTrips.tsx`, `TransportPanel.tsx`, `TransportSettings.tsx`, `TfdQueue.tsx`, `TfdMyCases.tsx`
- Test: `src/shell/modules.transport.test.ts`, `src/modules/transport/TransportGate.test.tsx`

**Interfaces:**
- Consumes: `canUseTransport`, `transportRoles`, `isDriverOnly`, `TRANSPORT_DISABLED`, `TRANSPORT_NO_ROLE` (Task 2); `canUseTfd`, `tfdRoles`, `TFD_DISABLED`, `TFD_NO_ROLE` (Task 3).
- Produces:
  - `ModuleId` ganha `"transport-trips" | "transport-series" | "transport-pending" | "transport-fleet" | "transport-my-trips" | "transport-panel" | "transport-settings" | "tfd-queue" | "tfd-my-cases"`.
  - `TransportGate({ title, sub, area: "transport" | "tfd", allow(roles: TransportRoles): boolean, children(roles: TransportRoles): ReactNode })` em `src/modules/transport/TransportGate.tsx`, com `export type TransportRoles = ReturnType<typeof transportRoles>`.
  - Telas exportadas com props: `TransportTrips({ today?, onGoToSecurity? })`, `TransportSeries({ today?, onGoToSecurity? })`, `TransportPending({ today? })`, `TransportFleet({ today?, onGoToSecurity? })`, `TransportMyTrips({ today?, now? })`, `TransportPanel({ today? })`, `TransportSettings({ onGoToSecurity? })`, `TfdQueue({ today?, now?, onGoToSecurity? })`, `TfdMyCases({ today? })`. Nesta task cada uma é só o portão com o corpo vazio (`() => null`); as Tasks 5–15 trocam o corpo.
  - `INVITE_ROLES` ganha `transport_dispatcher` ("Central de transporte"), `driver` ("Motorista"), `tfd_committee` ("Comissão de TFD"); `PRIVILEGED_ROLES` ganha `transport_dispatcher` e `tfd_committee`.

- [ ] **Step 1: Escreva os testes**

`src/shell/modules.transport.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { navGroupsFor } from "./modules";
import { trUser } from "../test/transportFixtures";

const ids = (u: Parameters<typeof navGroupsFor>[0], group: string) => navGroupsFor(u).find((g) => g.label === group)?.items.map((i) => i.id) ?? [];

describe("menu do transporte e do TFD (módulo 30)", () => {
  it("central vê viagens, séries, pendências, frota e painel; não vê o motorista nem o TFD", () => {
    expect(ids(trUser([ "transport_dispatcher" ]), "Transporte")).toEqual([ "transport-trips", "transport-series", "transport-pending", "transport-fleet", "transport-panel" ]);
    expect(ids(trUser([ "transport_dispatcher" ]), "TFD")).toEqual([]);
  });
  it("comissão vê a fila e o painel; médico vê os próprios pedidos; admin vê regras e painel", () => {
    expect(ids(trUser([ "tfd_committee" ]), "TFD")).toEqual([ "tfd-queue" ]);
    expect(ids(trUser([ "tfd_committee" ]), "Transporte")).toEqual([ "transport-panel" ]);
    expect(ids(trUser([ "health_professional" ]), "TFD")).toEqual([ "tfd-my-cases" ]);
    expect(ids(trUser([ "municipal_admin" ]), "Transporte")).toEqual([ "transport-panel", "transport-settings" ]);
  });
  it("sem tfd: grupo TFD some; sem sanitary_transport: os dois somem", () => {
    expect(ids(trUser([ "tfd_committee" ], [ "sanitary_transport", "clinical_record" ]), "TFD")).toEqual([]);
    const off = trUser([ "transport_dispatcher", "tfd_committee" ], [ "tfd", "clinical_record" ]);
    expect(navGroupsFor(off).some((g) => g.label === "Transporte" || g.label === "TFD")).toBe(false);
  });
  it("motorista só com o papel driver: Minhas viagens e Segurança, mais nada", () => {
    const groups = navGroupsFor(trUser([ "driver" ]));
    expect(groups.flatMap((g) => g.items.map((i) => i.id))).toEqual([ "transport-my-trips", "security" ]);
    expect(navGroupsFor(trUser([ "driver" ], [])).flatMap((g) => g.items.map((i) => i.id))).toEqual([ "security" ]);
  });
  it("motorista com outro papel vê o menu normal mais Minhas viagens", () => {
    const groups = navGroupsFor(trUser([ "driver", "viewer" ]));
    expect(groups.some((g) => g.label === "Visão geral")).toBe(true);
    expect(ids(trUser([ "driver", "viewer" ]), "Transporte")).toEqual([ "transport-my-trips" ]);
  });
});
```

`src/modules/transport/TransportGate.test.tsx`:

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { trUser } from "../../test/transportFixtures";
import * as api from "../../lib/api";
import { TransportGate } from "./TransportGate";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount(user = trUser(), area: "transport" | "tfd" = "transport") {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(user);
  return renderWithProviders(
    <TransportGate title="Viagens do dia" sub="agenda" area={area} allow={(r) => (area === "tfd" ? r.committee : r.dispatcher)}>
      {() => <p>dentro</p>}
    </TransportGate>);
}

describe("portão do transporte e do TFD", () => {
  it("interruptor desligado: frase", async () => {
    mount(trUser([ "transport_dispatcher" ], []));
    expect(await screen.findByText("o transporte sanitário está desligado nesta cidade")).not.toBeNull();
  });
  it("papel errado: frase", async () => {
    mount(trUser([ "driver" ]));
    expect(await screen.findByText("seu papel não permite usar o transporte sanitário")).not.toBeNull();
  });
  it("TFD desligado e TFD com papel", async () => {
    mount(trUser([ "tfd_committee" ], [ "sanitary_transport", "clinical_record" ]), "tfd");
    expect(await screen.findByText("o TFD está desligado nesta cidade")).not.toBeNull();
    cleanup();
    mount(trUser([ "tfd_committee" ]), "tfd");
    expect(await screen.findByText("dentro")).not.toBeNull();
  });
});
```

Em `src/lib/team.test.ts`, troque o teste "oferece os 18 papéis" por "oferece os 21 papéis" com a lista ordenada acrescida de `"driver"`, `"tfd_committee"` e `"transport_dispatcher"`, e acrescente:

```ts
// Módulo 30 (ADR 0048): central e comissão leem dado nominal — privilegiadas; o motorista não.
expect(INVITE_ROLES.find((r) => r.role === "transport_dispatcher")?.label).toBe("Central de transporte");
expect(INVITE_ROLES.find((r) => r.role === "tfd_committee")?.label).toBe("Comissão de TFD");
expect(INVITE_ROLES.find((r) => r.role === "driver")?.label).toBe("Motorista");
expect(isPrivilegedRole("transport_dispatcher")).toBe(true);
expect(isPrivilegedRole("tfd_committee")).toBe(true);
expect(isPrivilegedRole("driver")).toBe(false);
```

Em `src/shell/modules.test.ts`, os dois testes que contam grupos escondidos ganham `expect(groups.some((g) => g.label === "Transporte")).toBe(false);` e `… "TFD" …` e passam de `NAV_GROUPS.length - 17` para `- 19` e de `- 16` para `- 18`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/shell/modules.transport.test.ts src/modules/transport/TransportGate.test.tsx src/lib/team.test.ts src/shell/modules.test.ts`
Expected: FAIL (grupos, gate e papéis não existem).

- [ ] **Step 3: Implemente**

- `team.ts`: depois de `stock_keeper`, com o comentário "Módulo 30 (ADR 0048): central e comissão leem dado nominal — privilegiadas (step-up para conceder); o motorista só vê as próprias viagens", os três papéis na ordem `transport_dispatcher`, `driver`, `tfd_committee`; `PRIVILEGED_ROLES` ganha os dois privilegiados.
- `modules.ts`: grupos novos **depois de "Saúde bucal"**: `{ label: "Transporte", items: [ transport-trips "Viagens do dia" "⇶", transport-series "Séries recorrentes" "↻", transport-pending "Pendências e faltas" "⚠", transport-fleet "Frota, motoristas e destinos" "▣", transport-my-trips "Minhas viagens" "☰", transport-panel "Painel de transporte e TFD" "▦", transport-settings "Regras do TFD e do acompanhante" "⚙" ] }` e `{ label: "TFD", items: [ tfd-queue "Comissão de TFD" "⚖", tfd-my-cases "Meus pedidos de TFD" "✎" ] }`. Em `navGroupsFor`, logo depois do ramo do prestador: `if (isDriverOnly(user))` devolve só `transport-my-trips` (com `canUseTransport`) e `security`, mantendo os grupos de origem ("Transporte" e "Conta"). Nos filtros por item (comentário "Módulo 30 (ADR 0048): `sanitary_transport` para Transporte; `tfd` (com sanitary_transport e clinical_record) para TFD. A central nunca vê o TFD; o painel é do admin, da central e da comissão"): `transport-trips`/`series`/`pending`/`fleet` → `trOn && tr.dispatcher`; `transport-my-trips` → `trOn && tr.driver`; `transport-panel` → `trOn && (isAdmin || tr.dispatcher || tr.committee)`; `transport-settings` → `trOn && isAdmin`; `tfd-queue` → `tfdOn && tr.committee`; `tfd-my-cases` → `tfdOn && isProfessional`.
- `catalog.ts`: grupos `"Transporte": { icon: IconBus, description: "Frota, viagens, séries, motorista e lista de passageiros" }` e `"TFD": { icon: IconGavel, description: "Laudo, comissão, ajuda de custo e prestação de contas" }`; itens: `transport-trips` IconRoute "Agenda por dia e veículo, passageiros e lista impressa"; `transport-series` IconCalendarRepeat "Hemodiálise, radioterapia e outros tratamentos contínuos"; `transport-pending` IconUserX "Séries sem vaga e faltas por paciente"; `transport-fleet` IconCar "Veículos, motoristas com CNH e destinos com km"; `transport-my-trips` IconSteeringWheel "Viagens do dia, embarque e fechamento"; `transport-panel` IconChartBar "Viagens, ocupação, faltas, km e TFD do mês"; `transport-settings` IconSettings "Distância mínima, acompanhante, CNES e complemento"; `tfd-queue` IconScale (ou o que existir, Task 0) "Laudos aguardando, decisão, trechos e pagamento"; `tfd-my-cases` IconFilePlus "Pedidos de TFD que você fez e os devolvidos".
- `App.tsx`: `const driverOnly = isDriverOnly(auth.user);` e `useAlerts(!providerOnly && !driverOnly)`, `notifications={providerOnly || driverOnly ? null : …}`; os nove `case` em `renderModule`, as telas que pedem step-up com `onGoToSecurity={() => setActive("security")}`.
- `TransportGate`: molde do `VaccinationGate`; `area: "tfd"` usa `canUseTfd`/`TFD_DISABLED`/`TFD_NO_ROLE`, `"transport"` usa os do transporte; `roles` = `transportRoles(user)` (que já traz `committee` e `professional`).
- As nove telas: `export function X(props) { return <TransportGate title="<rótulo do menu>" sub="<descrição do catálogo>" area=… allow={…}>{() => null}</TransportGate>; }` com o mesmo `allow` do menu.

- [ ] **Step 4: Rode e veja passar; tipos e guardas**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/shell src/lib/team.test.ts src/modules/transport/TransportGate.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (inclusive `catalog.test.ts`, `router.test.ts` e `Shell.test.tsx`).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/team.ts src/lib/team.test.ts src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.transport.test.ts src/shell/catalog.ts src/App.tsx \
  src/modules/transport/TransportGate.tsx src/modules/transport/TransportGate.test.tsx src/modules/TransportTrips.tsx src/modules/TransportSeries.tsx \
  src/modules/TransportPending.tsx src/modules/TransportFleet.tsx src/modules/TransportMyTrips.tsx src/modules/TransportPanel.tsx src/modules/TransportSettings.tsx \
  src/modules/TfdQueue.tsx src/modules/TfdMyCases.tsx
$G commit -m "feat: add transport and TFD menu groups, roles and driver shell" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Regras da cidade e unidade "Gestão" (F-30.1)

**Files:**
- Modify: `src/modules/TransportSettings.tsx`, `src/lib/attendance.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`
- Test: `src/modules/TransportSettings.test.tsx` (e um caso novo em `src/lib/attendance.test.ts`)

**Interfaces:**
- Consumes: `getTransportSettings`, `updateTransportSettings`, `listAllUnits` (Task 1/existente); `parseReais`, `fmtCents`, `transportError`, `TR_SETTINGS_KEY`, `STATIC_QUERY` (Task 2); `canUseTfd` (Task 3); `SensitiveAction`.
- Produces: `HEALTH_MANAGEMENT_UNIT_KIND = { value: "health_management", label: "Gestão (secretaria ou central de regulação)" }` e `unitKindOptions(dentalOn: boolean, tfdOn = false)` em `src/lib/attendance.ts`; `UnitForm` com prop `tfdOn?: boolean`; `settingsProblems(d: SettingsDraft): Problem[]` exportada de `TransportSettings.tsx`.

- [ ] **Step 1: Escreva os testes**

`src/modules/TransportSettings.test.tsx`:

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { renderWithProviders } from "../test/campaignFixtures";
import { SETTINGS, trUser } from "../test/transportFixtures";
import * as api from "../lib/api";
import { TransportSettings } from "./TransportSettings";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getTransportSettings: vi.fn(),
  updateTransportSettings: vi.fn(), listAllUnits: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

const UNITS = [ { id: "u1", name: "UBS Centro", kind: "ubs", active: true }, { id: "u9", name: "Secretaria de Saúde", kind: "health_management", active: true } ];
function mount() {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "municipal_admin" ]));
  vi.mocked(api.getTransportSettings).mockResolvedValue(SETTINGS);
  vi.mocked(api.listAllUnits).mockResolvedValue(UNITS as never);
  return renderWithProviders(<TransportSettings />);
}

describe("regras do TFD e do acompanhante", () => {
  it("lê, mostra só unidades de gestão para o CNES do TFD e grava com step-up", async () => {
    vi.mocked(api.updateTransportSettings).mockResolvedValue({ ...SETTINGS, tfd_billing_health_unit_id: "u9" });
    mount();
    const min = (await screen.findByLabelText("Distância mínima do TFD (km)")) as HTMLInputElement;
    expect(min.value).toBe("50");
    const unit = screen.getByLabelText("Unidade do BPA do TFD") as HTMLSelectElement;
    expect([ ...unit.options ].map((o) => o.textContent)).toEqual([ "Escolha a unidade de gestão", "Secretaria de Saúde" ]);
    fireEvent.change(unit, { target: { value: "u9" } });
    fireEvent.change(screen.getByLabelText(/AJUDA DE CUSTO PARA ALIMENTAÇÃO\/PERNOITE/), { target: { value: "10,00" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar regras" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.updateTransportSettings).toHaveBeenCalledWith({
      tfd_min_distance_km: 50, companion_rule: { max_age_child: 15, min_age_elderly: 60 }, tfd_billing_health_unit_id: "u9",
      allowance_complements: { "0803010125": 1000 }
    }));
  });
  it("todas as recusas de uma vez, antes de enviar", async () => {
    mount();
    fireEvent.change(await screen.findByLabelText("Distância mínima do TFD (km)"), { target: { value: "0" } });
    fireEvent.change(screen.getByLabelText("Criança até (anos)"), { target: { value: "20" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar regras" }));
    expect(screen.getByText("a distância mínima precisa ser de 1 a 500 km")).not.toBeNull();
    expect(screen.getByText("a idade da criança precisa ser de 0 a 17 anos")).not.toBeNull();
    expect(api.updateTransportSettings).not.toHaveBeenCalled();
  });
});
```

Em `src/lib/attendance.test.ts`:

```ts
it("unidade de gestão só com o TFD ligado (módulo 30)", () => {
  expect(unitKindOptions(false, true).map((k) => k.value)).toContain("health_management");
  expect(unitKindOptions(false).map((k) => k.value)).not.toContain("health_management");
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportSettings.test.tsx src/lib/attendance.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `TransportSettings` (corpo do portão, `allow: r.admin`): `useQuery([TR_SETTINGS_KEY], getTransportSettings, STATIC_QUERY)` e `listAllUnits`; formulário nativo com "Distância mínima do TFD (km)" (inteiro 1–500), "Criança até (anos)" (0–17), "Idoso a partir de (anos)" (50–80), "Unidade do BPA do TFD" (só `kind === "health_management"`, primeira opção "Escolha a unidade de gestão" com valor vazio → `null`), e uma linha por `allowance_codes` com o rótulo `<code> — <name>` e o complemento em R$ (vazio = sem complemento, não vai no mapa). "Salvar regras" confere tudo (`settingsProblems`, um problema por campo, frases do teste) e abre `SensitiveAction` com `requiresStepUp`, `description` "As regras valem para os próximos laudos e viagens; o que já foi calculado não muda." e `run` → `updateTransportSettings(body)`; recusa do api por `transportError` (`billing_unit_invalid` → "a unidade do BPA precisa ser de gestão e ter CNES"). Texto fixo abaixo do campo do acompanhante: "Padrão da CIT 13/2017: paciente até 15 anos ou a partir de 60 anos tem acompanhante; fora disso, a central informa o motivo."
- `attendance.ts`: `HEALTH_MANAGEMENT_UNIT_KIND` e `unitKindOptions(dentalOn, tfdOn = false)` acrescenta o tipo novo depois do CEO. `UnitForm`: prop `tfdOn` e `unitKindOptions(dentalOn || value.kind === "ceo", tfdOn || value.kind === "health_management")`. `Units.tsx`: `KIND_LABEL` com o tipo novo e `tfdOn={canUseTfd(user)}` passado ao `UnitForm`.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportSettings.test.tsx src/lib/attendance.test.ts src/modules/attendance && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TransportSettings.tsx src/modules/TransportSettings.test.tsx src/lib/attendance.ts src/lib/attendance.test.ts src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx
$G commit -m "feat: add TFD and companion settings and the health management unit kind" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Veículos, motoristas com alerta de CNH e destinos (F-30.1)

**Files:**
- Modify: `src/modules/TransportFleet.tsx`
- Create: `src/modules/transport/VehiclesPanel.tsx`, `src/modules/transport/DriversPanel.tsx`, `src/modules/transport/DestinationsPanel.tsx`
- Test: `src/modules/TransportFleet.test.tsx`

**Interfaces:**
- Consumes: `listVehicles`, `createVehicle`, `updateVehicle`, `listDrivers`, `listDriverCandidates`, `createDriver`, `updateDriver`, `listDestinations`, `createDestination`, `updateDestination` (Task 1); `vehicleProblems`, `vehicleBody`, `driverProblems`, `destinationProblems`, `licenseText`, `licenseTone`, `vehicleKindLabel`, `transportRefusals`, chaves `TR_*` (Task 2).
- Produces: `VehiclesPanel()`, `DriversPanel({ today })`, `DestinationsPanel()`; `cnhAlertText(drivers: TransportDriver[]): string | null` exportada de `DriversPanel.tsx` ("1 motorista com CNH vencida · 2 vencem em até 30 dias").

- [ ] **Step 1: Escreva os testes** `src/modules/TransportFleet.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { ApiError } from "../lib/api";
import * as api from "../lib/api";
import { renderWithProviders } from "../test/campaignFixtures";
import { DEST_CURITIBA, DRIVER_D, DRIVER_EXPIRED, DRIVER_EXPIRING, TODAY30, VAN, trUser } from "../test/transportFixtures";
import { TransportFleet } from "./TransportFleet";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listVehicles: vi.fn(), createVehicle: vi.fn(),
  updateVehicle: vi.fn(), listDrivers: vi.fn(), listDriverCandidates: vi.fn(), createDriver: vi.fn(), updateDriver: vi.fn(), listDestinations: vi.fn(),
  createDestination: vi.fn(), updateDestination: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount() {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser());
  vi.mocked(api.listVehicles).mockResolvedValue([ VAN ]);
  vi.mocked(api.listDrivers).mockResolvedValue([ DRIVER_D, DRIVER_EXPIRED, DRIVER_EXPIRING ]);
  vi.mocked(api.listDriverCandidates).mockResolvedValue([ { user_id: "u-n", name: "Novo Motorista" } ]);
  vi.mocked(api.listDestinations).mockResolvedValue([ DEST_CURITIBA ]);
  return renderWithProviders(<TransportFleet today={TODAY30} />);
}

describe("frota, motoristas e destinos", () => {
  it("alerta de CNH no topo e na linha; texto, não só cor", async () => {
    mount();
    expect(await screen.findByText("1 motorista com CNH vencida · 1 vence em até 30 dias")).not.toBeNull();
    expect(screen.getByText("CNH vencida em 30/10/2026")).not.toBeNull();
    expect(screen.getByText("CNH vence em 20/11/2026")).not.toBeNull();
    // Só os 4 últimos dígitos da CNH aparecem (contrato §2).
    expect(screen.getAllByText("final 4321").length).toBe(3);
  });
  it("novo motorista: escolhe entre quem tem o papel; CNH só no corpo", async () => {
    vi.mocked(api.createDriver).mockResolvedValue({ ...DRIVER_D, id: "d9", user: { id: "u-n", name: "Novo Motorista" } });
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Novo motorista" }));
    const form = screen.getByRole("form", { name: "Motorista" });
    fireEvent.change(within(form).getByLabelText("Pessoa"), { target: { value: "u-n" } });
    fireEvent.change(within(form).getByLabelText("Número da CNH"), { target: { value: "01234567890" } });
    fireEvent.change(within(form).getByLabelText("Categoria"), { target: { value: "D" } });
    fireEvent.change(within(form).getByLabelText("Validade da CNH"), { target: { value: "2029-01-31" } });
    fireEvent.submit(form);
    await waitFor(() => expect(api.createDriver).toHaveBeenCalledWith({ user_id: "u-n", license_number: "01234567890", license_category: "D", license_expires_on: "2029-01-31" }));
    expect(await screen.findByText("Motorista cadastrado.")).not.toBeNull();
  });
  it("veículo: placa inválida e RENAVAM curto juntos; recusa do api traduzida", async () => {
    vi.mocked(api.createVehicle).mockRejectedValue(new ApiError(422, { error: "invalid", problems: [ { path: "plate", error: "taken" } ] }, "422"));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Novo veículo" }));
    const form = screen.getByRole("form", { name: "Veículo" });
    fireEvent.change(within(form).getByLabelText("Placa"), { target: { value: "AB-12" } });
    fireEvent.change(within(form).getByLabelText("RENAVAM"), { target: { value: "123" } });
    fireEvent.submit(form);
    expect(screen.getByText("placa fora do padrão (ABC1234 ou ABC1D23)")).not.toBeNull();
    expect(screen.getByText("o RENAVAM tem 11 dígitos")).not.toBeNull();
    expect(api.createVehicle).not.toHaveBeenCalled();
    fireEvent.change(within(form).getByLabelText("Placa"), { target: { value: "abc-1d23" } });
    fireEvent.change(within(form).getByLabelText("RENAVAM"), { target: { value: "12345678901" } });
    fireEvent.change(within(form).getByLabelText("Modelo"), { target: { value: "Sprinter" } });
    fireEvent.change(within(form).getByLabelText("Lugares para passageiros"), { target: { value: "15" } });
    fireEvent.submit(form);
    expect(await screen.findByText("já existe um veículo com esta placa")).not.toBeNull();
  });
});
```

(Que a central nunca chama `/tfd/*` é conferido pela varredura da Task 16 Step 2, sobre o código de todas as telas da central.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportFleet.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

`TransportFleet` (portão `r.dispatcher`) empilha três `Panel`: "Veículos", "Motoristas", "Destinos". Cada painel: `DataTable` (`card: "title"` na primeira coluna, `card: "actions"` no Editar), botão "Novo …" que abre o formulário **acima da tabela** (`<form aria-label="Veículo">`, `"Motorista"`, `"Destino"`), `Enter` envia, foco vai ao primeiro campo ao abrir e à frase de sucesso ("Veículo salvo.", "Motorista cadastrado." / "Motorista atualizado.", "Destino salvo.") depois; recusas por campo com `aria-invalid` + `aria-describedby`; `taken` em `plate` → "já existe um veículo com esta placa". Veículo: placa, RENAVAM, modelo, tipo (select com os cinco rótulos), código RENEM (opcional), lugares para passageiros, vagas para cadeirante, CNES (opcional), ativo. Motorista: "Pessoa" (select de `listDriverCandidates`, só ao criar — ao editar, o nome fixo), "Número da CNH" (`type="password"`-like? não: `inputMode="numeric"`, `autoComplete="off"`; ao editar vazio = não muda, e o campo diz "deixe vazio para manter"), "Categoria", "Validade da CNH"; a coluna CNH mostra `Tag` com `licenseText` e `licenseTone` e "final <last4>". `cnhAlertText` no topo do painel (com `role="status"`), nulo quando todos `valid`. Destino: código IBGE, nome, distância em km a partir da sede (0 = dentro do município), "Região metropolitana", ativo; texto fixo "A distância é a rodoviária, cadastrada pela cidade; ela decide o TFD e o deslocamento no BPA." Leituras `STATIC_QUERY`; depois de salvar, `invalidateQueries` da chave do painel.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportFleet.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TransportFleet.tsx src/modules/transport/VehiclesPanel.tsx src/modules/transport/DriversPanel.tsx src/modules/transport/DestinationsPanel.tsx src/modules/TransportFleet.test.tsx
$G commit -m "feat: add vehicles, drivers with license alert and destinations" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Município do prestador e prestador no agendamento externo do 26a

**Files:**
- Modify: `src/lib/regulation.ts`, `src/modules/regulation/ProviderForm.tsx`, `src/modules/regulation/ScheduleOutcome.tsx`
- Test: `src/modules/regulation/ScheduleOutcome.provider.test.tsx`

**Interfaces:**
- Consumes: `CareProvider.ibge_code`, `RegulationScheduleInput.care_provider_id`, `listCareProviders`, `scheduleRegulationRequest`, `updateCareProvider` (Task 1/existentes); `regulationFixtures` existentes.
- Produces: `ibgeProblem(raw: string): string | null` em `src/lib/regulation.ts` (vazio = ok; senão 7 dígitos, "o código IBGE do município tem 7 dígitos"); `ProviderForm` com o campo "Município (código IBGE)" para prestador não próprio; `ScheduleOutcome` com o select opcional "Prestador cadastrado".

- [ ] **Step 1: Escreva os testes** `src/modules/regulation/ScheduleOutcome.provider.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import * as api from "../../lib/api";
import { renderWithProviders } from "../../test/campaignFixtures";
import { ScheduleOutcome } from "./ScheduleOutcome";
import { ProviderForm } from "./ProviderForm";
import { regRequest, regUser } from "../../test/regulationFixtures";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listCareProviders: vi.fn(),
  scheduleRegulationRequest: vi.fn(), createCareProvider: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

const PROVIDERS = [ { id: "cp1", name: "Hospital de Olhos", cnes: "1234567", kind: "contracted", health_unit: null, active: true, ibge_code: "4106902" },
  { id: "cp0", name: "UBS Centro", cnes: null, kind: "own", health_unit: { id: "u1", name: "UBS Centro" }, active: true, ibge_code: null } ];

describe("agendamento externo com prestador (módulo 30, D13)", () => {
  it("o prestador cadastrado vai no corpo; o local livre continua", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
    vi.mocked(api.listCareProviders).mockResolvedValue(PROVIDERS as never);
    const req = regRequest({ status: "authorized", authorization: { offer_id: "o1", provider: { id: "cp1", name: "Hospital de Olhos", kind: "contracted" },
      month: "2026-11", skip_reason: null, at: "2026-11-01T10:00:00-03:00", by: { name: "Reg" } } });
    vi.mocked(api.scheduleRegulationRequest).mockResolvedValue({ ...req, status: "scheduled" });
    renderWithProviders(<ScheduleOutcome request={req} onChanged={() => {}} />);
    fireEvent.click(await screen.findByRole("button", { name: "Registrar agendamento" }));
    const select = (await screen.findByLabelText("Prestador cadastrado")) as HTMLSelectElement;
    expect([ ...select.options ].map((o) => o.textContent)).toEqual([ "Nenhum (só o local)", "Hospital de Olhos" ]);
    fireEvent.change(select, { target: { value: "cp1" } });
    fireEvent.change(screen.getByLabelText("Data e hora"), { target: { value: "2026-11-20T08:00" } });
    fireEvent.change(screen.getByLabelText("Local"), { target: { value: "Rua XV, 100" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar agendamento" }));
    await waitFor(() => expect(vi.mocked(api.scheduleRegulationRequest).mock.calls[0][1]).toMatchObject({ care_provider_id: "cp1", place: "Rua XV, 100" }));
  });
  it("prestador externo: IBGE com 7 dígitos ou vazio", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(regUser());
    renderWithProviders(<ProviderForm units={[]} onSaved={() => {}} onCancel={() => {}} />);
    fireEvent.change(await screen.findByLabelText("Município (código IBGE)"), { target: { value: "41069" } });
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    expect(screen.getByText("o código IBGE do município tem 7 dígitos")).not.toBeNull();
    expect(api.createCareProvider).not.toHaveBeenCalled();
  });
});
```

(Os rótulos "Registrar agendamento", "Data e hora", "Local", "Salvar agendamento" e o botão "Revisar" do `ProviderForm`, e as fábricas `regRequest`/`regUser` de `src/test/regulationFixtures.ts`, são os do `origin/main` `022ccd3`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/regulation/ScheduleOutcome.provider.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

`ProviderForm`: campo opcional "Município (código IBGE)" só para `kind !== "own"`, `inputMode="numeric"`, texto de ajuda "Usado pelo transporte para saber a cidade do atendimento"; entra no corpo como `ibge_code` (vazio → `null`); `ibgeProblem` no `check`. `ScheduleOutcome`: no formulário do agendamento, select "Prestador cadastrado" com os prestadores **ativos e não próprios** de `listCareProviders` (`STATIC_QUERY`, só lidos quando o formulário abre), primeira opção "Nenhum (só o local)"; ao escolher um, o campo "Local" é preenchido com o nome se estiver vazio (continua editável); corpo com `care_provider_id` só quando escolhido. Nada mais muda no 26a.

- [ ] **Step 4: Rode e veja passar** (inclui os testes antigos da regulação)

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/regulation src/modules/RegulationOffers.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/regulation.ts src/modules/regulation/ProviderForm.tsx src/modules/regulation/ScheduleOutcome.tsx src/modules/regulation/ScheduleOutcome.provider.test.tsx
$G commit -m "feat: link external regulation schedules to a registered provider and city" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Agenda do dia — criar, editar, confirmar e cancelar viagem (F-30.2)

**Files:**
- Modify: `src/modules/TransportTrips.tsx`
- Create: `src/modules/transport/TripCard.tsx`, `src/modules/transport/TripForm.tsx`
- Test: `src/modules/TransportTrips.test.tsx`

**Interfaces:**
- Consumes: `listTrips`, `createTrip`, `updateTrip`, `confirmTrip`, `cancelTrip`, `listVehicles`, `listDrivers`, `listDestinations` (Task 1); `newTripDraft`, `tripDraftFrom`, `tripProblems`, `tripBody`, `driverFitProblems`, `licenseText`, `occupancyText`, `tripStatusLabel`, `tripStatusTone`, `transportRefusals`, `TR_TRIPS_KEY` (Task 2).
- Produces: `TripForm({ trip?: Trip; date: string; vehicles; drivers; destinations; onSaved(t: Trip): void; onCancel(): void })`; `TripCard({ trip: Trip; today: string; children?: ReactNode; onChanged(t: Trip | null, text: string): void; onGoToSecurity?(): void })` — o slot `children` recebe a lista de passageiros (Task 9) e a correção (Task 11).

- [ ] **Step 1: Escreva os testes** `src/modules/TransportTrips.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { renderWithProviders } from "../test/campaignFixtures";
import { CAR, DEST_CURITIBA, DRIVER_B, DRIVER_D, DRIVER_EXPIRED, DRIVER_EXPIRING, TODAY30, VAN, trip, trUser } from "../test/transportFixtures";
import { TransportTrips } from "./TransportTrips";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listTrips: vi.fn(), createTrip: vi.fn(),
  updateTrip: vi.fn(), confirmTrip: vi.fn(), cancelTrip: vi.fn(), listVehicles: vi.fn(), listDrivers: vi.fn(), listDestinations: vi.fn(),
  findScheduledCare: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount(trips = [ trip({ status: "planned", vehicle: null, driver: null }) ]) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser());
  vi.mocked(api.listTrips).mockResolvedValue(trips);
  vi.mocked(api.listVehicles).mockResolvedValue([ VAN, CAR ]);
  vi.mocked(api.listDrivers).mockResolvedValue([ DRIVER_D, DRIVER_B, DRIVER_EXPIRED, DRIVER_EXPIRING ]);
  vi.mocked(api.listDestinations).mockResolvedValue([ DEST_CURITIBA ]);
  return renderWithProviders(<TransportTrips today={TODAY30} />);
}

describe("viagens do dia", () => {
  it("lê o dia de hoje; trocar a data relê", async () => {
    mount();
    expect(await screen.findByText("planejada")).not.toBeNull();
    expect(api.listTrips).toHaveBeenCalledWith({ date: TODAY30, vehicleId: null });
    fireEvent.change(screen.getByLabelText("Dia"), { target: { value: "2026-11-04" } });
    fireEvent.submit(screen.getByRole("form", { name: "Escolher o dia" }));
    await waitFor(() => expect(api.listTrips).toHaveBeenLastCalledWith({ date: "2026-11-04", vehicleId: null }));
  });
  it("nova viagem pelo teclado: Enter envia; motorista vencido aparece marcado", async () => {
    vi.mocked(api.createTrip).mockResolvedValue(trip({ id: "tr2", status: "planned" }));
    mount([]);
    fireEvent.click(await screen.findByRole("button", { name: "Nova viagem" }));
    const form = screen.getByRole("form", { name: "Viagem" });
    const driver = within(form).getByLabelText("Motorista") as HTMLSelectElement;
    expect([ ...driver.options ].map((o) => o.textContent)).toContain("Pedro Vencido (CNH vencida)");
    expect([ ...driver.options ].map((o) => o.textContent)).toContain("Rita Avencer (CNH vence em 20/11/2026)");
    fireEvent.change(within(form).getByLabelText("Saída"), { target: { value: "05:30" } });
    fireEvent.change(within(form).getByLabelText("Retorno"), { target: { value: "18:00" } });
    fireEvent.change(within(form).getByLabelText("Embarque 1"), { target: { value: "Praça da Matriz" } });
    fireEvent.change(within(form).getByLabelText("Destino 1"), { target: { value: "t1" } });
    fireEvent.submit(form);
    await waitFor(() => expect(api.createTrip).toHaveBeenCalledTimes(1));
    expect(vi.mocked(api.createTrip).mock.calls[0][0].departure_at).toBe("2026-11-03T05:30:00-03:00");
    expect(await screen.findByText("Viagem criada.")).not.toBeNull();
  });
  it("confirmar com motorista B na van recusa antes de enviar", async () => {
    mount([ trip({ status: "planned", driver: { id: "d2", name: "Ana Condutora" } }) ]);
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar viagem" }));
    expect(await screen.findByText("veículo com mais de 8 lugares exige CNH D ou E")).not.toBeNull();
    expect(api.confirmTrip).not.toHaveBeenCalled();
  });
  it("sobreposição de veículo e motorista: as duas frases, nada muda (Review Focus 2)", async () => {
    vi.mocked(api.confirmTrip).mockRejectedValue(new ApiError(422, { error: "invalid", problems: [
      { path: "vehicle_id", error: "vehicle_overlap" }, { path: "driver_id", error: "driver_overlap" } ] }, "422"));
    mount([ trip({ status: "planned" }) ]);
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar viagem" }));
    expect(await screen.findByText("o veículo já está em outra viagem nesse horário")).not.toBeNull();
    expect(screen.getByText("o motorista já está em outra viagem nesse horário")).not.toBeNull();
    expect(screen.getByText("planejada")).not.toBeNull();
    expect(api.listTrips).toHaveBeenCalledTimes(2);
  });
  it("cancelar pede motivo e avisa que os passageiros serão avisados", async () => {
    vi.mocked(api.cancelTrip).mockResolvedValue(trip({ status: "cancelled" }));
    mount([ trip() ]);
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar viagem" }));
    expect(screen.getByText(/os passageiros recebem o aviso do cancelamento/)).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "veículo quebrado na garagem" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.cancelTrip).toHaveBeenCalledWith("tr1", "veículo quebrado na garagem"));
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportTrips.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `TransportTrips` (portão `r.dispatcher`): `<form aria-label="Escolher o dia">` com "Dia" (date, padrão `today`) e "Veículo" (todos ou um), "Ver viagens" envia; `useQuery([TR_TRIPS_KEY, date, vehicleId], …, STATIC_QUERY)`. Lista de `TripCard` em ordem de `departure_at`; "Nova viagem" abre `TripForm` acima da lista; depois de criar, a lista é relida e o foco vai à frase "Viagem criada.". Texto fixo `NO_CLINICAL_NOTE` no rodapé.
- `TripForm` (`<form aria-label="Viagem">`): "Saída" (time), "Data do retorno" (date, padrão o dia), "Retorno" (time), "Veículo" (ativos; opção "Sem veículo por enquanto"), "Motorista" (todos; vencido com o sufixo " (CNH vencida)" e `disabled`; a vencer com " (CNH vence em dd/mm/aaaa)"; opção "Sem motorista por enquanto"), embarques ("Embarque 1"…, botão "Adicionar embarque" e "Remover" por linha, mínimo um), destinos ("Destino 1"…, select dos destinos ativos, mínimo um). Envio confere `tripProblems(…, confirm: false)` + `driverFitProblems` quando há os dois; recusas do api por `transportRefusals`, mostradas no campo do `path` ou no topo; `busy` desabilita.
- `TripCard` (`<article aria-labelledby>` com título "05:30 → 18:00 · Curitiba" e foco programático quando aberto): `Tag` da situação, veículo (placa e modelo) ou "sem veículo", motorista ou "sem motorista", `occupancyText`, paradas em ordem. Ações por situação: `planned` → Editar, "Confirmar viagem", "Cancelar viagem"; `confirmed` → Editar, "Cancelar viagem"; `in_progress`/`done`/`cancelled` → nenhuma (a correção da Task 11 entra pelo `children`). "Confirmar viagem" confere `tripProblems(draft, confirm: true)` com `tripDraftFrom(trip)` antes de chamar `confirmTrip`; recusa → frases e `refetch` da agenda (Review Focus 2). "Cancelar viagem" usa `SensitiveAction` sem step-up, campo "Motivo" (10–500, multiline), descrição "A viagem sai da agenda e os passageiros recebem o aviso do cancelamento."; `seq` ignora resposta de outra data.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportTrips.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TransportTrips.tsx src/modules/transport/TripCard.tsx src/modules/transport/TripForm.tsx src/modules/TransportTrips.test.tsx
$G commit -m "feat: add the daily trip agenda with confirmation and cancellation" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Montar a viagem — passageiros, sem atendimento com motivo, retirar e PDF (F-30.2)

**Files:**
- Modify: `src/modules/TransportTrips.tsx` (passa `PassengerList` ao `TripCard`)
- Create: `src/modules/transport/PassengerList.tsx`, `src/modules/transport/ScheduledCare.tsx`, `src/modules/transport/PassengerForm.tsx`, `src/modules/transport/TransportPersonPick.tsx`, `src/modules/transport/openManifest.ts`
- Test: `src/modules/transport/Passengers.test.tsx`

**Interfaces:**
- Consumes: `findScheduledCare`, `addPassenger`, `removePassenger`, `fetchTripManifest`, `lookupTransportPerson`, `findTransportDuplicates` (Task 1); `passengerDraftFrom`, `passengerProblems`, `passengerBody`, `originLabel`, `exceptionReasonLabel`, `attendanceLabel`, `transportRefusals` (Task 2); `PersonStart`, `PersonSource` (24a); `TripCard` (Task 8).
- Produces: `PassengerList({ trip: Trip; onChanged(text: string): void })`; `ScheduledCare({ trip: Trip; onPick(item: ScheduledCareItem): void })`; `PassengerForm({ trip: Trip; initial: PassengerDraft; onSaved(p: TripPassenger): void; onCancel(): void })`; `TransportPersonPick({ onPick(patientId: string, name: string | null): void; onCancel?(): void })`; `openManifest(tripId: string): Promise<{ ok: true } | { ok: false; text: string }>`; `transportPersonSource: PersonSource`.

- [ ] **Step 1: Escreva os testes** `src/modules/transport/Passengers.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { renderWithProviders } from "../../test/campaignFixtures";
import { passenger, trip, trUser } from "../../test/transportFixtures";
import { PassengerList } from "./PassengerList";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), findScheduledCare: vi.fn(),
  addPassenger: vi.fn(), removePassenger: vi.fn(), fetchTripManifest: vi.fn(), lookupTransportPerson: vi.fn(), findTransportDuplicates: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

const SUGGESTION = { patient: { id: "pt7", display_name: "Carlos Souza", has_cpf: true, cns_masked: null, phone_masked: null }, origin: "regulation",
  origin_id: "rr7", appointment_at: "2026-11-03T10:00:00-03:00", destination: { destination_id: "t1", provider_name: "Hospital Regional", address: "Rua XV, 100" } };

function mount(t = trip()) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser());
  return renderWithProviders(<PassengerList trip={t} onChanged={() => {}} />);
}

describe("passageiros da viagem", () => {
  it("a lista não mostra dado clínico; mostra origem pelo rótulo", async () => {
    mount(trip({ passengers: [ passenger({ origin: "tfd" }) ] }));
    expect(await screen.findByText("Maria da Silva")).not.toBeNull();
    expect(screen.getByText("TFD")).not.toBeNull();
    expect(screen.queryByText(/H25|CID|FACOEMULSIFICACAO|oftalmo/i)).toBeNull();
  });
  it("atendimentos marcados na data viram passageiro com a origem; CPF nunca na URL", async () => {
    vi.mocked(api.findScheduledCare).mockResolvedValue([ SUGGESTION ] as never);
    vi.mocked(api.addPassenger).mockResolvedValue(passenger({ id: "p7", patient: { ...SUGGESTION.patient } }));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Buscar atendimentos marcados na data" }));
    expect(api.findScheduledCare).toHaveBeenCalledWith({ date: "2026-11-03", destinationId: null });
    fireEvent.click(await screen.findByRole("button", { name: "Pôr Carlos Souza na viagem" }));
    fireEvent.submit(screen.getByRole("form", { name: "Passageiro" }));
    await waitFor(() => expect(vi.mocked(api.addPassenger).mock.calls[0][1]).toMatchObject({ patient_id: "pt7", origin: "regulation", origin_id: "rr7", destination_id: "t1", stop_id: "s1" }));
  });
  it("sem atendimento: busca por CPF e exige o motivo", async () => {
    vi.mocked(api.lookupTransportPerson).mockResolvedValue({ patient_id: "pt9", citizen_id: null, display_name: "Lúcia Lima", registration_id: null, has_cpf: true, cns_masked: null });
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Passageiro sem atendimento marcado" }));
    fireEvent.change(screen.getByLabelText("CPF"), { target: { value: "123.456.789-09" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: /Lúcia Lima/ }));
    fireEvent.submit(screen.getByRole("form", { name: "Passageiro" }));
    expect(screen.getByText("informe por que o passageiro vai sem atendimento marcado")).not.toBeNull();
    expect(api.addPassenger).not.toHaveBeenCalled();
  });
  it("acompanhante fora da regra: a frase pede o motivo e o campo ganha foco", async () => {
    vi.mocked(api.findScheduledCare).mockResolvedValue([ SUGGESTION ] as never);
    vi.mocked(api.addPassenger).mockRejectedValue(new ApiError(422, { error: "invalid", problems: [ { path: "companion.reason", error: "companion_not_allowed" } ] }, "422"));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Buscar atendimentos marcados na data" }));
    fireEvent.click(await screen.findByRole("button", { name: "Pôr Carlos Souza na viagem" }));
    const form = screen.getByRole("form", { name: "Passageiro" });
    fireEvent.click(within(form).getByLabelText("Com acompanhante"));
    fireEvent.change(within(form).getByLabelText("Nome do acompanhante"), { target: { value: "Joana Souza" } });
    fireEvent.submit(form);
    expect(await screen.findByText("acompanhante fora da regra da cidade — informe o motivo")).not.toBeNull();
    expect(document.activeElement).toBe(within(form).getByLabelText("Motivo do acompanhante"));
  });
  it("lotada: a tela avisa antes de enviar", async () => {
    vi.mocked(api.findScheduledCare).mockResolvedValue([ SUGGESTION ] as never);
    mount(trip({ occupancy: { seats_used: 15, seats: 15, wheelchair_used: 0, wheelchair_spots: 1 } }));
    fireEvent.click(await screen.findByRole("button", { name: "Buscar atendimentos marcados na data" }));
    fireEvent.click(await screen.findByRole("button", { name: "Pôr Carlos Souza na viagem" }));
    fireEvent.submit(screen.getByRole("form", { name: "Passageiro" }));
    expect(screen.getByText("a viagem está lotada")).not.toBeNull();
  });
  it("retirar pede motivo; PDF abre em janela nova sem dado na URL", async () => {
    vi.mocked(api.removePassenger).mockResolvedValue(undefined);
    vi.mocked(api.fetchTripManifest).mockResolvedValue({ blob: new Blob([ "%PDF" ], { type: "application/pdf" }), filename: null });
    const win = { location: { href: "" }, opener: {}, close: vi.fn() };
    vi.stubGlobal("open", vi.fn(() => win));
    vi.stubGlobal("URL", { ...URL, createObjectURL: vi.fn(() => "blob:x"), revokeObjectURL: vi.fn() });
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Lista de passageiros (PDF)" }));
    await waitFor(() => expect(win.location.href).toBe("blob:x"));
    expect(window.open).toHaveBeenCalledWith("", "_blank");
    fireEvent.click(screen.getByRole("button", { name: "Retirar Maria da Silva" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "consulta remarcada pelo hospital" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.removePassenger).toHaveBeenCalledWith("tr1", "p1", "consulta remarcada pelo hospital"));
    vi.unstubAllGlobals();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/transport/Passengers.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `PassengerList`: `DataTable` com colunas Passageiro (`card: "title"`; nome, `cns_masked` quando houver), Embarque (rótulo da parada), Destino (`destination.name` · `provider_name`), Hora do atendimento (`fmtHourMinute`), Vaga/Cadeira, Acompanhante (nome e "doc. final 1234"), Origem (`originLabel`, e o motivo quando `exception`), Presença (`attendanceLabel`), Ações (`card: "actions"`; "Retirar <nome>" com `SensitiveAction` sem step-up e campo "Motivo" 10–500, descrição "O passageiro sai da viagem e recebe o aviso."). Botões do topo: "Buscar atendimentos marcados na data", "Passageiro sem atendimento marcado", "Lista de passageiros (PDF)". Só para `planned`/`confirmed`; nas outras situações, só a lista e o PDF.
- `ScheduledCare`: mutação `findScheduledCare({ date: trip.date, destinationId: null })` no clique (POST nominal, sem cache), lista "Pôr <nome> na viagem" com origem, hora e destino; itens já na viagem (mesmo `patient.id`) aparecem "já nesta viagem" sem botão; chave da linha `${origin}:${origin_id}` (D4).
- `PassengerForm` (`<form aria-label="Passageiro">`): paciente fixo (nome), "Parada de embarque" (embarques da viagem; padrão o primeiro), "Destino" (destinos da viagem; padrão o da sugestão), "Prestador" e "Endereço" (texto, da sugestão), "Hora do atendimento" (datetime-local, opcional), "Ocupa lugar" (padrão marcado), "Cadeira de rodas", "Com acompanhante" → "Nome do acompanhante", "Documento do acompanhante" (opcional), "Motivo do acompanhante" (opcional; obrigatório quando o api recusa `companion_not_allowed`); sem origem (pessoa avulsa) → "Motivo da viagem sem atendimento" (os quatro rótulos) e "Descreva o motivo" quando `other`. `passengerProblems` antes de enviar; recusa → frase no campo do `path` e foco nele (`companion.reason`, `exception_reason`, `takes_seat`, `wheelchair`) ou no topo (`patient_already_on_trip`, `origin_mismatch`); sucesso → "Passageiro incluído." e a agenda relê.
- `TransportPersonPick`: `PersonStart` do 24a com `transportPersonSource = { lookup: lookupTransportPerson, duplicates: findTransportDuplicates, describe: transportError, pickLabel: (n) => n }`; escolha `new` (sem cadastro) → frase "esta pessoa não tem cadastro — cadastre no balcão de atendimento antes de pôr na viagem" (a central não cria paciente, D1).
- `openManifest`: molde de `openGuide` do 26a (janela aberta no clique, `blob:`, revogado em 60 s, 404 → "lista não encontrada", outras → `transportError`).
- `TransportTrips`: passa `<PassengerList trip={t} onChanged={…} />` como `children` do `TripCard`.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/transport src/modules/TransportTrips.test.tsx src/modules/territory && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TransportTrips.tsx src/modules/transport/PassengerList.tsx src/modules/transport/ScheduledCare.tsx src/modules/transport/PassengerForm.tsx \
  src/modules/transport/TransportPersonPick.tsx src/modules/transport/openManifest.ts src/modules/transport/Passengers.test.tsx
$G commit -m "feat: assemble trips from scheduled care with exceptions and a printable passenger list" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Séries recorrentes, pendências e faltas (F-30.3)

**Files:**
- Modify: `src/modules/TransportSeries.tsx`, `src/modules/TransportPending.tsx`
- Create: `src/modules/transport/SeriesForm.tsx`
- Test: `src/modules/TransportSeries.test.tsx`

**Interfaces:**
- Consumes: `listSeries`, `createSeries`, `updateSeries`, `suspendSeries`, `listTransportPending`, `listNoShows`, `listDestinations` (Task 1); `seriesProblems`, `seriesBody`, `weekdaysLabel`, `transportRefusals`, `TR_SERIES_KEY`, `TR_PENDING_KEY`, `TR_NO_SHOWS_KEY` (Task 2); `TransportPersonPick` (Task 9).
- Produces: `SeriesForm({ series?: TransportSeries; destinations; onSaved(s: TransportSeries): void; onCancel(): void })`; `pendingReasonLabel(code: string): string` em `TransportPending.tsx` (`no_seat` "sem lugar em viagem confirmada", `no_wheelchair_spot` "sem vaga de cadeirante em viagem confirmada").

- [ ] **Step 1: Escreva os testes** `src/modules/TransportSeries.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import * as api from "../lib/api";
import { renderWithProviders } from "../test/campaignFixtures";
import { DEST_CURITIBA, TODAY30, series, trUser } from "../test/transportFixtures";
import { TransportSeries } from "./TransportSeries";
import { TransportPending } from "./TransportPending";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listSeries: vi.fn(), createSeries: vi.fn(),
  updateSeries: vi.fn(), suspendSeries: vi.fn(), listTransportPending: vi.fn(), listNoShows: vi.fn(), listDestinations: vi.fn(),
  lookupTransportPerson: vi.fn(), findTransportDuplicates: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("séries recorrentes", () => {
  it("lista com dias e horários; suspender pede data e motivo e diz o que acontece", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser());
    vi.mocked(api.listSeries).mockResolvedValue([ series() ]);
    vi.mocked(api.listDestinations).mockResolvedValue([ DEST_CURITIBA ]);
    vi.mocked(api.suspendSeries).mockResolvedValue(series({ status: "suspended", suspended_from: "2026-11-05" }));
    renderWithProviders(<TransportSeries today={TODAY30} />);
    expect(await screen.findByText("seg, qua, sex · atendimento 07:00 · saída 05:30")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Suspender a série de José Hemodiálise" }));
    expect(screen.getByText(/o paciente sai das viagens a partir desta data/)).not.toBeNull();
    fireEvent.change(screen.getByLabelText("A partir de"), { target: { value: "2026-11-05" } });
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "paciente transferido para outra clínica" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.suspendSeries).toHaveBeenCalledWith("se1", { from_on: "2026-11-05", reason: "paciente transferido para outra clínica" }));
  });
  it("nova série: dias e horários conferidos antes de enviar", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser());
    vi.mocked(api.listSeries).mockResolvedValue([]);
    vi.mocked(api.listDestinations).mockResolvedValue([ DEST_CURITIBA ]);
    vi.mocked(api.lookupTransportPerson).mockResolvedValue({ patient_id: "pt2", citizen_id: null, display_name: "José", registration_id: null, has_cpf: true, cns_masked: null });
    renderWithProviders(<TransportSeries today={TODAY30} />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova série" }));
    fireEvent.change(screen.getByLabelText("CPF"), { target: { value: "12345678909" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: /José/ }));
    const form = screen.getByRole("form", { name: "Série recorrente" });
    fireEvent.change(within(form).getByLabelText("Horário do atendimento"), { target: { value: "07:00" } });
    fireEvent.change(within(form).getByLabelText("Horário de saída"), { target: { value: "08:00" } });
    fireEvent.submit(form);
    expect(screen.getByText("escolha ao menos um dia da semana")).not.toBeNull();
    expect(screen.getByText("a saída precisa ser antes do horário do atendimento")).not.toBeNull();
    expect(api.createSeries).not.toHaveBeenCalled();
  });
});

describe("pendências e faltas", () => {
  it("pendência mostra paciente pela série; faltas por período", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser());
    vi.mocked(api.listSeries).mockResolvedValue([ series() ]);
    vi.mocked(api.listTransportPending).mockResolvedValue([ { series_id: "se1", date: "2026-11-05", reason: "no_seat" } ]);
    vi.mocked(api.listNoShows).mockResolvedValue([ { patient: { id: "pt1", display_name: "Maria da Silva" }, count: 2, dates: [ "2026-10-20", "2026-10-27" ] } ]);
    renderWithProviders(<TransportPending today={TODAY30} />);
    expect(await screen.findByText("José Hemodiálise")).not.toBeNull();
    expect(screen.getByText("sem lugar em viagem confirmada")).not.toBeNull();
    expect(screen.getByText(/resolva em "Viagens do dia"/)).not.toBeNull();
    expect(api.listNoShows).toHaveBeenCalledWith({ from: "2026-10-04", to: TODAY30 });
    expect(await screen.findByText("20/10/2026, 27/10/2026")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportSeries.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `TransportSeries` (portão `r.dispatcher`): `DataTable` de séries — Paciente (`card: "title"`), Destino e prestador, Quando (`"<dias> · atendimento HH:MM · saída HH:MM"`), Período (`desde dd/mm/aaaa` e `até …`), Situação (`active` "ativa" ok, `suspended` "suspensa a partir de dd/mm/aaaa" warn, `ended` "encerrada"), Ações ("Editar", "Suspender a série de <nome>" — `SensitiveAction` sem step-up com campos "A partir de" (date ≥ hoje) e "Motivo" (10–500), descrição "O paciente sai das viagens a partir desta data; as viagens já feitas ficam como estão."). Texto fixo: "As viagens das próximas três semanas são criadas todo dia de manhã; sem vaga em viagem confirmada, a série aparece em Pendências."
- `SeriesForm` (`<form aria-label="Série recorrente">`): paciente por `TransportPersonPick` (só ao criar), "Destino", "Prestador" (texto), sete checkboxes "seg"…"dom" (valores ISO 1–7, D5) agrupados em `<fieldset>` "Dias da semana", "Horário do atendimento", "Horário de saída", "Início", "Fim" (opcional), "Cadeira de rodas", acompanhante (nome, documento, motivo); `seriesProblems` antes; sucesso "Série salva.".
- `TransportPending` (portão `r.dispatcher`): painel "Pendências das séries" (`listTransportPending` + `listSeries` para o nome e o destino, pelo `series_id`; série que não está na lista → "série <id>"), cada linha com data, paciente, motivo e o texto fixo 'Ponha o paciente numa viagem do dia em "Viagens do dia" ou crie uma viagem nova; resolva em "Viagens do dia".'; painel "Faltas" com `<form>` "De"/"Até" (padrão hoje − 30 dias até hoje) e `DataTable` paciente, quantidade e datas (`dd/mm/aaaa` separadas por vírgula). Nenhum destino de tratamento nas faltas.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportSeries.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TransportSeries.tsx src/modules/TransportPending.tsx src/modules/transport/SeriesForm.tsx src/modules/TransportSeries.test.tsx
$G commit -m "feat: add recurring series, pending items and no-shows" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Tela do motorista — presença e fechamento; correção pela central (F-30.4)

**Files:**
- Modify: `src/modules/TransportMyTrips.tsx`, `src/modules/TransportTrips.tsx` (passa a correção ao `TripCard`)
- Create: `src/modules/transport/DriverTrip.tsx`, `src/modules/transport/CloseTripForm.tsx`, `src/modules/transport/AttendanceCorrection.tsx`
- Test: `src/modules/TransportMyTrips.test.tsx`

**Interfaces:**
- Consumes: `listMyTrips`, `startMyTrip`, `recordAttendance`, `closeMyTrip`, `correctAttendance` (Task 1); `pendingAttendance`, `closeProblems`, `closeBody`, `attendanceLabel`, `expenseKindLabel`, `transportRefusals`, `TR_MY_TRIPS_KEY` (Task 2).
- Produces: `DriverTrip({ trip: Trip; today: string; onChanged(t: Trip): void })`; `CloseTripForm({ trip: Trip; onClosed(t: Trip): void })`; `AttendanceCorrection({ trip: Trip; onCorrected(t: Trip): void; onGoToSecurity?(): void })`.

- [ ] **Step 1: Escreva os testes** `src/modules/TransportMyTrips.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY30, passenger, trip, trUser } from "../test/transportFixtures";
import { TransportMyTrips } from "./TransportMyTrips";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyTrips: vi.fn(), startMyTrip: vi.fn(),
  recordAttendance: vi.fn(), closeMyTrip: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

const twoPassengers = trip({ status: "in_progress", passengers: [
  passenger({ patient: { id: "pt1", display_name: "Maria da Silva", phone_masked: "(41) ****-1234" } }),
  passenger({ id: "p2", stop_id: "s1", patient: { id: "pt2", display_name: "Carlos Souza", phone_masked: null }, companion: { name: "Joana", document_last4: null }, companion_attendance: "pending" })
] });

function mount(t = twoPassengers) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "driver" ]));
  vi.mocked(api.listMyTrips).mockResolvedValue([ t ]);
  return renderWithProviders(<TransportMyTrips today={TODAY30} />);
}

describe("tela do motorista (celular)", () => {
  it("cartões por parada, sem tabela; sem CNS nem dado clínico", async () => {
    mount();
    expect(await screen.findByRole("heading", { name: /Praça da Matriz/ })).not.toBeNull();
    expect(screen.queryByRole("table")).toBeNull();
    expect(screen.queryByText(/\*\*\* \*\*\*\*/)).toBeNull();
    expect(screen.getAllByRole("button", { name: /^Embarcou/ }).length).toBe(3);
  });
  it("iniciar só na viagem confirmada do dia", async () => {
    vi.mocked(api.startMyTrip).mockResolvedValue(trip({ status: "in_progress" }));
    mount(trip({ status: "confirmed" }));
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar viagem" }));
    await waitFor(() => expect(api.startMyTrip).toHaveBeenCalledWith("tr1"));
  });
  it("toque duplo grava uma vez; falha devolve a presença anterior na linha (Review Focus 1)", async () => {
    let fail!: (e: unknown) => void;
    vi.mocked(api.recordAttendance).mockImplementationOnce(() => new Promise((_, rej) => { fail = rej; }));
    mount();
    const card = (await screen.findByText("Maria da Silva")).closest("section") as HTMLElement;
    const boarded = within(card).getByRole("button", { name: "Embarcou — Maria da Silva" });
    fireEvent.click(boarded);
    fireEvent.click(boarded);
    expect(api.recordAttendance).toHaveBeenCalledTimes(1);
    expect(vi.mocked(api.recordAttendance).mock.calls[0][1]).toEqual([ { passenger_id: "p1", attendance: "boarded" } ]);
    fail(new TypeError("network"));
    expect(await within(card).findByRole("alert")).not.toBeNull();
    expect(boarded.getAttribute("aria-pressed")).toBe("false");
    const other = (screen.getByText("Carlos Souza").closest("section")) as HTMLElement;
    expect(within(other).queryByRole("alert")).toBeNull();
  });
  it("fechar barrado com presença pendente; com tudo marcado, envia odômetro e abastecimento", async () => {
    vi.mocked(api.closeMyTrip).mockResolvedValue(trip({ status: "done" }));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Fechar viagem" }));
    const form = screen.getByRole("form", { name: "Fechamento da viagem" });
    fireEvent.change(within(form).getByLabelText("Odômetro na saída"), { target: { value: "1000" } });
    fireEvent.change(within(form).getByLabelText("Odômetro na chegada"), { target: { value: "1220" } });
    fireEvent.submit(form);
    expect(screen.getByText("marque embarcou ou faltou para todos, inclusive acompanhantes")).not.toBeNull();
    expect(api.closeMyTrip).not.toHaveBeenCalled();
    cleanup();
    mount(trip({ status: "in_progress", passengers: [ passenger({ attendance: "no_show" }) ] }));
    fireEvent.click(await screen.findByRole("button", { name: "Fechar viagem" }));
    const f2 = screen.getByRole("form", { name: "Fechamento da viagem" });
    fireEvent.change(within(f2).getByLabelText("Odômetro na saída"), { target: { value: "1000" } });
    fireEvent.change(within(f2).getByLabelText("Odômetro na chegada"), { target: { value: "1220" } });
    fireEvent.click(within(f2).getByRole("button", { name: "Adicionar abastecimento" }));
    fireEvent.change(within(f2).getByLabelText("Litros 1"), { target: { value: "40,5" } });
    fireEvent.change(within(f2).getByLabelText("Valor 1"), { target: { value: "250,00" } });
    fireEvent.change(within(f2).getByLabelText("Posto 1"), { target: { value: "Posto Central" } });
    fireEvent.submit(f2);
    await waitFor(() => expect(api.closeMyTrip).toHaveBeenCalledWith("tr1", { odometer_start: 1000, odometer_end: 1220,
      fuel: [ { liters: 40.5, amount_cents: 25000, station: "Posto Central" } ], expenses: [] }));
  });
  it("recusa attendance_pending do api também aparece", async () => {
    vi.mocked(api.closeMyTrip).mockRejectedValue(new ApiError(422, { error: "attendance_pending" }, "422"));
    mount(trip({ status: "in_progress", passengers: [ passenger({ attendance: "boarded" }) ] }));
    fireEvent.click(await screen.findByRole("button", { name: "Fechar viagem" }));
    const form = screen.getByRole("form", { name: "Fechamento da viagem" });
    fireEvent.change(within(form).getByLabelText("Odômetro na saída"), { target: { value: "1" } });
    fireEvent.change(within(form).getByLabelText("Odômetro na chegada"), { target: { value: "2" } });
    fireEvent.submit(form);
    expect(await screen.findByText("marque embarcou ou faltou para todos, inclusive acompanhantes")).not.toBeNull();
  });
});
```

Acrescente `correctAttendance: vi.fn()` à fábrica do `vi.mock` de `src/modules/TransportTrips.test.tsx`, `passenger` ao import das fixtures, e o caso:

```tsx
it("viagem concluída: correção de presença com motivo e step-up", async () => {
  vi.mocked(api.correctAttendance).mockResolvedValue(trip({ status: "done", passengers: [ passenger({ attendance: "boarded" }) ] }));
  mount([ trip({ status: "done", passengers: [ passenger({ attendance: "no_show" }) ] }) ]);
  fireEvent.click(await screen.findByRole("button", { name: "Corrigir presença" }));
  fireEvent.change(screen.getByLabelText("Presença de Maria da Silva"), { target: { value: "boarded" } });
  fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "motorista marcou errado no celular" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.correctAttendance).toHaveBeenCalledWith("tr1", { items: [ { passenger_id: "p1", attendance: "boarded" } ], reason: "motorista marcou errado no celular" }));
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportMyTrips.test.tsx src/modules/TransportTrips.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `TransportMyTrips` (portão `r.driver`): "Dia" fixo em hoje (o motorista não navega datas; contrato §5 `date=`), `useQuery([TR_MY_TRIPS_KEY, today], () => listMyTrips(today), STATIC_QUERY)`, botão "Atualizar" (sem polling). Lista de `DriverTrip`. Sem viagem: `EmptyState` "nenhuma viagem sua hoje".
- `DriverTrip`: cabeçalho com hora de saída e retorno, veículo (placa), situação; `confirmed` e `trip.date === today` → "Iniciar viagem"; passageiros **agrupados por parada de embarque, na ordem das paradas** (`<h3>` com o rótulo da parada), cada passageiro em `<section aria-label=nome>`: nome, destino (`destination.name` · prestador), hora do atendimento, telefone mascarado (quando houver), "cadeira de rodas", e dois botões grandes `Embarcou — <nome>` / `Faltou — <nome>` com `aria-pressed`; acompanhante com o próprio par (`Embarcou — acompanhante de <nome>`). Cada toque envia **um** item (`recordAttendance(trip.id, [item])`) e trava o par daquela linha até a resposta (estado `busyRow`); a tela mostra o valor otimista, e na falha volta ao anterior com `role="alert"` na linha (`transportError`). Presença só em `in_progress`. Estilo: coluna única, `maxWidth: "100%"`, sem `DataTable`, sem `gridTemplateColumns`; alvos com `buttonStyle`/`secondaryButtonStyle` (≥ 44 px). "Fechar viagem" abre `CloseTripForm`.
- `CloseTripForm` (`<form aria-label="Fechamento da viagem">`): "Odômetro na saída", "Odômetro na chegada" (`inputMode="numeric"`), abastecimentos (Litros n, Valor n, Posto n; "Adicionar abastecimento"/"Remover"), despesas (Tipo n, Valor n, Observação n; "Adicionar despesa"); `closeProblems` antes, `closeBody` no envio, `busy`; sucesso → "Viagem fechada." e a lista relê; depois de `done`, o cartão mostra o resumo do `log` e nenhum botão.
- `AttendanceCorrection` (central, `trip.status === "done"`): botão "Corrigir presença" abre `SensitiveAction` **com step-up** (contrato §5), um select por passageiro e acompanhante ("Presença de <nome>", "Presença do acompanhante de <nome>") e o campo "Motivo" (10–500); envia só os itens alterados; descrição "A presença fechada só muda por aqui, com motivo; a mudança fica no histórico."; nada alterado → frase "mude ao menos uma presença". `TransportTrips` passa `<AttendanceCorrection …/>` ao `TripCard` junto com a lista.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TransportMyTrips.test.tsx src/modules/TransportTrips.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TransportMyTrips.tsx src/modules/TransportTrips.tsx src/modules/transport/DriverTrip.tsx src/modules/transport/CloseTripForm.tsx \
  src/modules/transport/AttendanceCorrection.tsx src/modules/TransportMyTrips.test.tsx src/modules/TransportTrips.test.tsx
$G commit -m "feat: add the driver phone screen with attendance and trip closing" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: "Pedir TFD" na consulta e os meus pedidos (F-30.5)

**Files:**
- Modify: `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/TfdMyCases.tsx`
- Create: `src/modules/tfd/TfdRequestForm.tsx`
- Test: `src/modules/tfd/TfdRequest.test.tsx`

**Interfaces:**
- Consumes: `getTfdOptions`, `getTfdPrefill`, `createTfdCase`, `updateTfdCase`, `submitTfdCase`, `listMyTfdCases`, `readTfdCase`, `searchSigtap`, `searchTerminology` (Task 1/existentes); `canRequestTfd`, `reportDraftFrom`, `reportProblems`, `caseBody`, `canEditCase`, `tfdStatusLabel`, `tfdRefusals`, `TFD_OPTIONS_KEY`, `TFD_MY_CASES_KEY` (Task 3); `CodeSearch` (19a).
- Produces: `ConsultationView` com a prop `tfd?: boolean`; `TfdRequestForm({ source: { consultationId: string; patientId?: string } | { tfdCase: TfdCase }; searchDelayMs?: number; onDone(c: TfdCase, text: string): void; onCancel(): void })`.

- [ ] **Step 1: Escreva os testes** `src/modules/tfd/TfdRequest.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { renderWithProviders } from "../../test/campaignFixtures";
import { DEST_CURITIBA, DEST_NEAR, tfdCase, tfdRow, trUser } from "../../test/transportFixtures";
import { TfdRequestForm } from "./TfdRequestForm";
import { TfdMyCases } from "../TfdMyCases";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getTfdOptions: vi.fn(), getTfdPrefill: vi.fn(),
  createTfdCase: vi.fn(), updateTfdCase: vi.fn(), submitTfdCase: vi.fn(), listMyTfdCases: vi.fn(), readTfdCase: vi.fn(), searchSigtap: vi.fn(), searchTerminology: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

const OPTIONS = { destinations: [ DEST_CURITIBA, DEST_NEAR ], providers: [ { id: "cp1", name: "Hospital de Olhos", ibge_code: "4106902" } ], min_distance_km: 50 };

function mountForm() {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "health_professional" ]));
  vi.mocked(api.getTfdOptions).mockResolvedValue(OPTIONS);
  vi.mocked(api.getTfdPrefill).mockResolvedValue({ patient_id: "pt1", cid10: "H25.1", sigtap_code: "0405050372" });
  return renderWithProviders(<TfdRequestForm source={{ consultationId: "co1" }} searchDelayMs={0} onDone={() => {}} onCancel={() => {}} />);
}

describe("pedir TFD a partir da consulta", () => {
  it("abre pré-preenchido; destino perto recusa antes de enviar", async () => {
    mountForm();
    const form = await screen.findByRole("form", { name: "Laudo de TFD" });
    expect(api.getTfdPrefill).toHaveBeenCalledWith("co1");
    expect(within(form).getByText("H25.1")).not.toBeNull();
    fireEvent.change(within(form).getByLabelText("Cidade de destino"), { target: { value: "t2" } });
    fireEvent.change(within(form).getByLabelText("Justificativa"), { target: { value: "Sem oftalmologia cirúrgica no município." } });
    fireEvent.click(within(form).getByRole("button", { name: "Enviar à comissão" }));
    expect(screen.getByText("destino a menos de 50 km da sede — não cabe TFD")).not.toBeNull();
    expect(api.createTfdCase).not.toHaveBeenCalled();
  });
  it("salvar e enviar: cria o rascunho e envia; CID e justificativa só no corpo", async () => {
    vi.mocked(api.createTfdCase).mockResolvedValue(tfdCase({ status: "draft" }));
    vi.mocked(api.submitTfdCase).mockResolvedValue(tfdCase({ status: "submitted" }));
    mountForm();
    const form = await screen.findByRole("form", { name: "Laudo de TFD" });
    fireEvent.change(within(form).getByLabelText("Cidade de destino"), { target: { value: "t1" } });
    fireEvent.change(within(form).getByLabelText("Justificativa"), { target: { value: "Sem oftalmologia cirúrgica no município." } });
    fireEvent.click(within(form).getByRole("button", { name: "Enviar à comissão" }));
    await waitFor(() => expect(api.submitTfdCase).toHaveBeenCalledWith("c1"));
    expect(vi.mocked(api.createTfdCase).mock.calls[0][0]).toMatchObject({ patient_id: "pt1", consultation_id: "co1", destination_id: "t1",
      report: { cid10: "H25.1", sigtap_code: "0405050372", transport_mode: "fleet", companion_indicated: false } });
  });
  it("recusa do envio mostra todas as frases", async () => {
    vi.mocked(api.createTfdCase).mockResolvedValue(tfdCase({ status: "draft" }));
    vi.mocked(api.submitTfdCase).mockRejectedValue(new ApiError(422, { error: "invalid", problems: [ { path: "report.sigtap_code", error: "primary_care_procedure" } ] }, "422"));
    mountForm();
    const form = await screen.findByRole("form", { name: "Laudo de TFD" });
    fireEvent.change(within(form).getByLabelText("Cidade de destino"), { target: { value: "t1" } });
    fireEvent.change(within(form).getByLabelText("Justificativa"), { target: { value: "Sem oftalmologia cirúrgica no município." } });
    fireEvent.click(within(form).getByRole("button", { name: "Enviar à comissão" }));
    expect(await screen.findByText("procedimento da atenção básica — não cabe TFD")).not.toBeNull();
    expect(screen.getByText("O rascunho ficou salvo em Meus pedidos de TFD.")).not.toBeNull();
  });
});

describe("meus pedidos de TFD", () => {
  it("lista sem dado clínico; devolvido abre pela leitura e permite editar", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "health_professional" ]));
    vi.mocked(api.listMyTfdCases).mockResolvedValue([ tfdRow({ status: "returned" }) ]);
    vi.mocked(api.readTfdCase).mockResolvedValue(tfdCase({ status: "returned", events: [ { kind: "returned", at: "2026-11-03T09:00:00-03:00", by_name: "Comissão", reason: "falta o exame de vista" } ] }));
    vi.mocked(api.getTfdOptions).mockResolvedValue(OPTIONS);
    renderWithProviders(<TfdMyCases />);
    expect(await screen.findByText("devolvido para complementar")).not.toBeNull();
    expect(screen.queryByText("H25.1")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Abrir o pedido de Maria da Silva" }));
    expect(await screen.findByText("falta o exame de vista")).not.toBeNull();
    expect(api.readTfdCase).toHaveBeenCalledTimes(1);
    expect(screen.getByRole("form", { name: "Laudo de TFD" })).not.toBeNull();
  });
});
```

Em `src/modules/consultation/ConsultationView.test.tsx` (existente), acrescente: com `tfd` e autor da consulta finalizada, o botão "Pedir TFD" aparece e abre o `TfdRequestForm`; sem `tfd`, ou com outra pessoa lendo, ou em rascunho, não aparece.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/tfd/TfdRequest.test.tsx src/modules/consultation/ConsultationView.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ConsultationWorkspace`: `const tfdOn = canRequestTfd(user, consultation?.cbo_code) && !baseline;` e `tfd={tfdOn}` no `ConsultationView`. `ConsultationView`: `showTfd = !!props.tfd && isAuthor && c.status === "finalized"`; botão secundário "Pedir TFD" na área das ações da consulta; aberto, mostra `TfdRequestForm` com `source={{ consultationId: c.id }}`; ao terminar, a frase de `onDone` fica na consulta ("Pedido de TFD enviado à comissão." ou "Rascunho de TFD salvo.").
- `TfdRequestForm` (`<form aria-label="Laudo de TFD">`): lê `getTfdOptions` (`STATIC_QUERY`) e, com `consultationId`, `getTfdPrefill` (`gcTime: 0`); com `tfdCase`, `reportDraftFrom(null, tfdCase)`. Campos: "Procedimento (SIGTAP)" (`CodeSearch` com `searchSigtap`; mostra o escolhido "<código> — <nome>"), "CID-10" (`CodeSearch` com `searchTerminology(q, "cid10")`; o pré-preenchido aparece como escolhido), "Justificativa" (textarea, contador 20–2000), "Indicação de acompanhante" + "Motivo da indicação" (Portaria 55, art. 7º), "Tipo de transporte" (`transportModeLabel` dos quatro), "Cidade de destino" (destinos com o km: "Curitiba — 110 km"), "Prestador" (opcional; os de `providers` cujo `ibge_code` é o do destino primeiro). Botões "Salvar rascunho" e "Enviar à comissão": `reportProblems` antes (os dois); salvar = `createTfdCase` (ou `updateTfdCase` quando já existe); enviar = salvar e `submitTfdCase`. Recusa do envio depois do rascunho criado → frases + "O rascunho ficou salvo em Meus pedidos de TFD." Nada do laudo em storage nem em log.
- `TfdMyCases` (portão `area: "tfd"`, `r.professional`): `listMyTfdCases` (`STATIC_QUERY`), `DataTable` com Paciente (`card: "title"`), Destino, Situação, Enviado em, e "Abrir o pedido de <nome>" → mutação `readTfdCase` (trilha; `READ_TRAIL_NOTE` ao lado do botão); aberto: eventos (data, quem, motivo) e, se `canEditCase`, o `TfdRequestForm` com `source={{ tfdCase }}`; senão, o laudo só leitura.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/tfd src/modules/consultation && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/consultation/ConsultationView.tsx src/modules/consultation/ConsultationView.test.tsx src/modules/consultation/ConsultationWorkspace.tsx \
  src/modules/tfd/TfdRequestForm.tsx src/modules/TfdMyCases.tsx src/modules/tfd/TfdRequest.test.tsx
$G commit -m "feat: request TFD from a finalized consultation and list own requests" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Comissão de TFD — fila, leitura com trilha, decisão com step-up, atendimento e internação (F-30.5)

**Files:**
- Modify: `src/modules/TfdQueue.tsx`
- Create: `src/modules/tfd/CaseView.tsx`, `src/modules/tfd/DecisionPanel.tsx`, `src/modules/tfd/AppointmentForm.tsx`, `src/modules/tfd/InpatientForm.tsx`
- Test: `src/modules/TfdQueue.test.tsx`

**Interfaces:**
- Consumes: `listTfdCases`, `readTfdCase`, `decideTfdCase`, `setTfdAppointment`, `setTfdInpatient`, `getTfdOptions` (Task 1); `decisionProblems`, `appointmentProblems`, `inpatientProblems`, `tfdStatusLabel`, `tfdStatusTone`, `decisionLabel`, `tfdRefusals`, `APPOINTMENT_REQUIRED`, `READ_TRAIL_NOTE`, `TFD_CASES_KEY` (Task 3); `SensitiveAction`.
- Produces: `CaseView({ tfdCase: TfdCase; now: Date; today: string; onChanged(c: TfdCase, text: string): void; onGoToSecurity?(): void; children?: ReactNode })` — o `children` recebe os painéis de dinheiro da Task 14; `DecisionPanel({ tfdCase; onDecided(c: TfdCase): void; onNeedAppointment(): void; onGoToSecurity? })`; `AppointmentForm({ tfdCase; providers; now; onSaved(c: TfdCase): void })`; `InpatientForm({ tfdCase; onSaved(c: TfdCase): void })`.

- [ ] **Step 1: Escreva os testes** `src/modules/TfdQueue.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { renderWithProviders } from "../test/campaignFixtures";
import { DEST_CURITIBA, NOW30, TODAY30, tfdCase, tfdRow, trUser } from "../test/transportFixtures";
import { TfdQueue } from "./TfdQueue";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listTfdCases: vi.fn(), readTfdCase: vi.fn(),
  decideTfdCase: vi.fn(), setTfdAppointment: vi.fn(), setTfdInpatient: vi.fn(), getTfdOptions: vi.fn(), stepUpMfa: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount(user = trUser([ "tfd_committee" ])) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(user);
  vi.mocked(api.listTfdCases).mockResolvedValue({ items: [ tfdRow() ], next_before: null });
  vi.mocked(api.getTfdOptions).mockResolvedValue({ destinations: [ DEST_CURITIBA ], providers: [ { id: "cp1", name: "Hospital de Olhos", ibge_code: "4106902" } ], min_distance_km: 50 });
  return renderWithProviders(<TfdQueue today={TODAY30} now={NOW30} />);
}

describe("comissão de TFD", () => {
  it("a fila não traz dado clínico; o laudo só abre no clique, uma vez", async () => {
    vi.mocked(api.readTfdCase).mockResolvedValue(tfdCase());
    mount();
    expect(await screen.findByText("Maria da Silva")).not.toBeNull();
    expect(api.listTfdCases).toHaveBeenCalledWith({ status: "submitted", before: null });
    expect(screen.queryByText("H25.1")).toBeNull();
    expect(api.readTfdCase).not.toHaveBeenCalled();
    expect(screen.getByText("Abrir o laudo fica registrado na trilha de acesso.")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Ler o laudo de Maria da Silva" }));
    expect(await screen.findByText("H25.1")).not.toBeNull();
    expect(screen.getByText("Sem oftalmologia cirúrgica no município.")).not.toBeNull();
    expect(api.readTfdCase).toHaveBeenCalledTimes(1);
  });
  it("autorizar sem atendimento não envia; step-up repete uma vez (Review Focus 4)", async () => {
    vi.mocked(api.readTfdCase).mockResolvedValue(tfdCase());
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Ler o laudo de Maria da Silva" }));
    fireEvent.click(await screen.findByRole("button", { name: "Autorizar" }));
    expect(screen.getByText("marque o atendimento no destino antes de autorizar")).not.toBeNull();
    expect(api.decideTfdCase).not.toHaveBeenCalled();
    expect(document.activeElement).toBe(screen.getByRole("button", { name: "Marcar atendimento" }));
  });
  it("com atendimento: autoriza com motivo; mfa_required pede o código e repete uma vez", async () => {
    vi.mocked(api.readTfdCase).mockResolvedValue(tfdCase({ appointment_at: "2026-11-20T08:00:00-03:00" }));
    vi.mocked(api.decideTfdCase)
      .mockRejectedValueOnce(new ApiError(401, { error: "mfa_required" }, "401"))
      .mockResolvedValueOnce(tfdCase({ status: "authorized", appointment_at: "2026-11-20T08:00:00-03:00" }));
    vi.mocked(api.stepUpMfa).mockResolvedValue(undefined as never);
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Ler o laudo de Maria da Silva" }));
    fireEvent.click(await screen.findByRole("button", { name: "Autorizar" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "laudo completo e destino confirmado" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    fireEvent.change(await screen.findByLabelText(/código/i), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(api.decideTfdCase).toHaveBeenCalledTimes(2));
    expect(vi.mocked(api.decideTfdCase).mock.calls[1]).toEqual([ "c1", { decision: "authorize", reason: "laudo completo e destino confirmado" } ]);
    expect(await screen.findByText("autorizado")).not.toBeNull();
  });
  it("marcar atendimento: no passado recusa; no futuro envia hora da cidade e prestador", async () => {
    vi.mocked(api.readTfdCase).mockResolvedValue(tfdCase());
    vi.mocked(api.setTfdAppointment).mockResolvedValue(tfdCase({ appointment_at: "2026-11-20T08:00:00-03:00" }));
    mount();
    fireEvent.click(await screen.findByRole("button", { name: "Ler o laudo de Maria da Silva" }));
    fireEvent.click(await screen.findByRole("button", { name: "Marcar atendimento" }));
    fireEvent.change(screen.getByLabelText("Data e hora do atendimento"), { target: { value: "2026-11-02T08:00" } });
    fireEvent.submit(screen.getByRole("form", { name: "Atendimento no destino" }));
    expect(screen.getByText("o atendimento não pode estar no passado")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Data e hora do atendimento"), { target: { value: "2026-11-20T08:00" } });
    fireEvent.change(screen.getByLabelText("Prestador"), { target: { value: "cp1" } });
    fireEvent.submit(screen.getByRole("form", { name: "Atendimento no destino" }));
    await waitFor(() => expect(api.setTfdAppointment).toHaveBeenCalledWith("c1", { appointment_at: "2026-11-20T08:00:00-03:00", provider_id: "cp1" }));
  });
  it("sem o papel: portão; filtro por situação vai na URL", async () => {
    mount(trUser([ "transport_dispatcher" ]));
    expect(await screen.findByText("seu papel não permite usar o TFD")).not.toBeNull();
    cleanup();
    mount();
    fireEvent.change(await screen.findByLabelText("Situação"), { target: { value: "returned" } });
    await waitFor(() => expect(api.listTfdCases).toHaveBeenLastCalledWith({ status: "returned", before: null }));
  });
});
```

(`stepUpMfa` é o que o `useStepUp` chama no `origin/main`; o molde do código e da repetição é `src/components/SensitiveAction.test.tsx` — confira ali o rótulo do campo do código.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TfdQueue.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `TfdQueue` (portão `area: "tfd"`, `r.committee`): select "Situação" (padrão `submitted`; opções das nove situações e "Todas"), `useQuery([TFD_CASES_KEY, status, before], …, STATIC_QUERY)` com "Mais antigos" por `next_before`; `DataTable` Paciente (`card: "title"`), Destino, Situação (`Tag`), Enviado em, Ações ("Ler o laudo de <nome>", `card: "actions"`); `READ_TRAIL_NOTE` acima da tabela. O clique roda a mutação `readTfdCase` (`gcTime: 0`), guarda o `<tfd-case>` em estado local (nunca em cache de query), e abre `CaseView` acima da lista com foco no título; "Fechar laudo" limpa o estado (o laudo some da memória da tela). Resposta de um processo já trocado é ignorada (`seq`).
- `CaseView`: cabeçalho (paciente, `Tag` da situação, médico e CBO), laudo (procedimento código e nome, CID-10, justificativa, acompanhante indicado e motivo, tipo de transporte), destino (nome, km, metropolitano), prestador, atendimento no destino (ou "não marcado"), internações, histórico de eventos (data, rótulo, quem, motivo). Ações por situação: `submitted` → `DecisionPanel` e "Marcar atendimento"; `authorized`/`scheduled`/`travelling` → "Marcar atendimento" (remarcar) e "Registrar internação"; `children` (Task 14) para `authorized` em diante. Toda escrita devolve o `<tfd-case>` (D12), que substitui o estado e dispara a releitura da fila.
- `DecisionPanel`: três botões "Autorizar", "Devolver para complementar", "Recusar"; o clique roda `decisionProblems` — com `appointment_required`, mostra `APPOINTMENT_REQUIRED` e chama `onNeedAppointment()` (que põe o foco no botão "Marcar atendimento"), sem abrir o step-up; senão abre `SensitiveAction` com `requiresStepUp`, campo "Motivo" (multiline, 10–500), descrição por decisão ("O processo fica autorizado e a central pode pôr o paciente numa viagem." / "O médico recebe o pedido de volta para complementar." / "O processo é encerrado como recusado; o paciente vê a situação no celular."), `translateError` por `tfdRefusals`.
- `AppointmentForm` (`<form aria-label="Atendimento no destino">`): "Data e hora do atendimento" (`datetime-local`), "Prestador" (opcional; providers de `getTfdOptions`); `appointmentProblems(value, now)`; corpo com `cityLocalIso`. `InpatientForm` (`<form aria-label="Internação no destino">`): "Internado desde", "Alta em" (opcional); `inpatientProblems`; texto fixo "Sem diária enquanto o paciente está internado no destino."

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/TfdQueue.test.tsx src/components/SensitiveAction.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TfdQueue.tsx src/modules/tfd/CaseView.tsx src/modules/tfd/DecisionPanel.tsx src/modules/tfd/AppointmentForm.tsx src/modules/tfd/InpatientForm.tsx src/modules/TfdQueue.test.tsx
$G commit -m "feat: add the TFD committee queue with audited reading and step-up decisions" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Trechos, ajuda de custo, pagamento e prestação de contas (F-30.6)

**Files:**
- Modify: `src/modules/TfdQueue.tsx` (passa os painéis ao `CaseView`)
- Create: `src/modules/tfd/LegsPanel.tsx`, `src/modules/tfd/AllowancesPanel.tsx`, `src/modules/tfd/PaymentForm.tsx`, `src/modules/tfd/AccountabilityForm.tsx`
- Test: `src/modules/tfd/CaseMoney.test.tsx`

**Interfaces:**
- Consumes: `addTfdLeg`, `markTfdLegDone`, `listTfdAllowances`, `recordTfdPayment`, `uploadTfdAccountability` (Task 1); `legProblems`, `legBody`, `paymentProblems`, `paymentBody`, `allowanceTotal`, `paidTotal`, `accountabilityFileProblem`, `legModeLabel`, `directionLabel`, `allowanceKindLabel`, `paymentMethodLabel`, `tfdRefusals` (Task 3); `fmtCents`, `parseReais` (Task 2); `CaseView` (Task 13).
- Produces: `LegsPanel({ tfdCase; today; onChanged(c: TfdCase, text: string): void })`, `AllowancesPanel({ tfdCase })`, `PaymentForm({ tfdCase; today; onSaved(c: TfdCase): void })`, `AccountabilityForm({ tfdCase; onDone(c: TfdCase): void })`.

- [ ] **Step 1: Escreva os testes** `src/modules/tfd/CaseMoney.test.tsx`

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import * as api from "../../lib/api";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY30, tfdCase, trUser } from "../../test/transportFixtures";
import { LegsPanel } from "./LegsPanel";
import { AllowancesPanel } from "./AllowancesPanel";
import { PaymentForm } from "./PaymentForm";
import { AccountabilityForm } from "./AccountabilityForm";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), addTfdLeg: vi.fn(), markTfdLegDone: vi.fn(),
  recordTfdPayment: vi.fn(), uploadTfdAccountability: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });
const authorized = tfdCase({ status: "authorized", appointment_at: "2026-11-20T08:00:00-03:00",
  trip_passengers: [ { id: "p5", date: "2026-11-20", departure_at: "2026-11-20T04:00:00-03:00", attendance: "pending", companion_attendance: null } ] });
const as = () => vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "tfd_committee" ]));

describe("trechos", () => {
  it("frota: escolhe a viagem do paciente; passagem: valor e empresa; km vem do destino", async () => {
    as();
    vi.mocked(api.addTfdLeg).mockResolvedValue(authorized);
    renderWithProviders(<LegsPanel tfdCase={authorized} today={TODAY30} onChanged={() => {}} />);
    fireEvent.click(await screen.findByRole("button", { name: "Novo trecho" }));
    const form = screen.getByRole("form", { name: "Trecho" });
    expect(within(form).getByText("110 km (do cadastro do destino)")).not.toBeNull();
    fireEvent.change(within(form).getByLabelText("Data"), { target: { value: "2026-11-20" } });
    fireEvent.change(within(form).getByLabelText("Viagem da frota"), { target: { value: "p5" } });
    fireEvent.submit(form);
    await waitFor(() => expect(api.addTfdLeg).toHaveBeenCalledWith("c1", { direction: "outbound", mode: "fleet", date: "2026-11-20", trip_passenger_id: "p5" }));
  });
  it("trecho de passagem realizado: botão próprio; o de frota não tem", async () => {
    as();
    const c = tfdCase({ status: "scheduled", legs: [
      { id: "l1", direction: "outbound", mode: "fleet", date: "2026-11-20", distance_km: 110, trip_passenger_id: "p5", ticket: null, done: false },
      { id: "l2", direction: "return", mode: "bus_ticket", date: "2026-11-21", distance_km: 110, trip_passenger_id: null, ticket: { amount_cents: 8500, carrier: "Viação Sul" }, done: false } ] });
    vi.mocked(api.markTfdLegDone).mockResolvedValue({ ...c, legs: [ c.legs[0], { ...c.legs[1], done: true } ] });
    renderWithProviders(<LegsPanel tfdCase={c} today={TODAY30} onChanged={() => {}} />);
    expect(await screen.findByText("o trecho da frota se cumpre com o embarque marcado pelo motorista")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Marcar volta de 21/11/2026 como realizada" }));
    await waitFor(() => expect(api.markTfdLegDone).toHaveBeenCalledWith("l2"));
  });
});

describe("ajuda de custo e pagamento", () => {
  it("ajuda calculada pelo api: tabela, complemento e total; pago de devido", async () => {
    as();
    const c = tfdCase({ status: "travelling",
      allowances: [ { id: "a1", person: "patient", kind: "meal_overnight", sigtap_code: "0803010125", quantity: 1, table_cents: 2475, complement_cents: 1000, total_cents: 3475 },
        { id: "a2", person: "companion", kind: "meal_overnight", sigtap_code: "0803010133", quantity: 1, table_cents: 2475, complement_cents: 0, total_cents: 2475 } ],
      payments: [ { id: "py1", paid_on: TODAY30, amount_cents: 3000, method: "pix", receipt_number: "R-1", by_name: "Ana" } ] });
    renderWithProviders(<AllowancesPanel tfdCase={c} />);
    expect((await screen.findByText(/Total devido/)).textContent?.replace(/\s/g, " ")).toBe("Total devido: R$ 59,50 · pago R$ 30,00");
    expect(screen.getByText("acompanhante")).not.toBeNull();
    expect(screen.getByText("O valor é calculado pela regra (tabela SIGTAP da competência + complemento da cidade); a tela não recalcula.")).not.toBeNull();
  });
  it("pagamento: data futura recusa; valor com milhar vai em centavos", async () => {
    as();
    vi.mocked(api.recordTfdPayment).mockResolvedValue(authorized);
    renderWithProviders(<PaymentForm tfdCase={authorized} today={TODAY30} onSaved={() => {}} />);
    const form = await screen.findByRole("form", { name: "Pagamento registrado" });
    fireEvent.change(within(form).getByLabelText("Pago em"), { target: { value: "2026-11-04" } });
    fireEvent.submit(form);
    expect(screen.getByText("a data do pagamento não pode ser futura")).not.toBeNull();
    fireEvent.change(within(form).getByLabelText("Pago em"), { target: { value: TODAY30 } });
    fireEvent.change(within(form).getByLabelText("Valor"), { target: { value: "1.234,56" } });
    fireEvent.change(within(form).getByLabelText("Forma"), { target: { value: "transfer" } });
    fireEvent.change(within(form).getByLabelText("Número do recibo"), { target: { value: "R-77" } });
    fireEvent.submit(form);
    await waitFor(() => expect(api.recordTfdPayment).toHaveBeenCalledWith("c1", { paid_on: TODAY30, amount_cents: 123456, method: "transfer", receipt_number: "R-77" }));
    expect(screen.getByText("O pagamento registrado não se apaga nem se edita.")).not.toBeNull();
  });
});

describe("prestação de contas (Review Focus 5)", () => {
  it("comprovante grande ou de outro tipo não sai; duplo clique envia uma vez", async () => {
    as();
    let finish!: (c: unknown) => void;
    vi.mocked(api.uploadTfdAccountability).mockImplementation(() => new Promise((res) => { finish = res; }) as never);
    renderWithProviders(<AccountabilityForm tfdCase={tfdCase({ status: "travelling" })} onDone={() => {}} />);
    const input = (await screen.findByLabelText("Comprovante de comparecimento")) as HTMLInputElement;
    const big = new File([ new Uint8Array(12 * 1024 * 1024) ], "grande.pdf", { type: "application/pdf" });
    fireEvent.change(input, { target: { files: [ big ] } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar e concluir o processo" }));
    expect(screen.getByText("o comprovante passa de 10 MB")).not.toBeNull();
    const doc = new File([ "x" ], "a.docx", { type: "application/msword" });
    fireEvent.change(input, { target: { files: [ doc ] } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar e concluir o processo" }));
    expect(screen.getByText("o comprovante precisa ser PDF, JPG ou PNG")).not.toBeNull();
    expect(api.uploadTfdAccountability).not.toHaveBeenCalled();
    const png = new File([ new Uint8Array(200 * 1024) ], "comprovante-maria.png", { type: "image/png" });
    fireEvent.change(input, { target: { files: [ png ] } });
    const send = screen.getByRole("button", { name: "Enviar e concluir o processo" });
    fireEvent.click(send);
    fireEvent.click(send);
    expect(api.uploadTfdAccountability).toHaveBeenCalledTimes(1);
    expect(screen.queryByText("comprovante-maria.png")).toBeNull();
    finish(tfdCase({ status: "completed", accountability: { closed_at: "2026-11-03T10:00:00-03:00", document: { id: "f1", filename: "comprovante.png" } } }));
    expect(await screen.findByText("Processo concluído com a prestação de contas.")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/tfd/CaseMoney.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `LegsPanel`: tabela dos trechos (Sentido, Data, Modal, Distância, Viagem da frota ou passagem "R$ 85,00 · Viação Sul", Situação "realizado"/"a realizar"); trecho de frota não realizado mostra a frase "o trecho da frota se cumpre com o embarque marcado pelo motorista"; trecho de passagem não realizado → botão "Marcar <ida|volta> de dd/mm/aaaa como realizada" (`ConfirmButton`, uma chamada, `busy`). "Novo trecho" (`<form aria-label="Trecho">`) só em `authorized`/`scheduled`/`travelling`: "Sentido" (padrão: ida se não há ida; senão volta), "Modal" (padrão o `transport_mode` do laudo), "Data", e, por modal: `fleet` → "Viagem da frota" (select de `tfd_case.trip_passengers` "dd/mm/aaaa saída HH:MM", D10; vazio → texto "o paciente ainda não está numa viagem da frota — peça à central"); passagem → "Valor da passagem", "Empresa", "Acompanhante viaja junto" (D10). A distância é só texto ("<km> km (do cadastro do destino)"). `legProblems` antes; `legBody` não manda campo vazio.
- `AllowancesPanel`: tabela das ajudas (Pessoa "paciente"/"acompanhante", Tipo, Código SIGTAP, Quantidade, Tabela, Complemento, Total) com `fmtCents`; linha "Total devido: R$ … · pago R$ …" (`allowanceTotal`, `paidTotal`); texto fixo "O valor é calculado pela regra (tabela SIGTAP da competência + complemento da cidade); a tela não recalcula."; sem ajudas → "a ajuda de custo é calculada quando a volta é marcada como realizada".
- `PaymentForm` (`<form aria-label="Pagamento registrado">`): "Pago em" (padrão hoje), "Valor", "Forma" (quatro), "Número do recibo"; lista dos pagamentos (data, valor, forma, recibo, quem registrou) e o texto fixo "O pagamento registrado não se apaga nem se edita." `paymentProblems` antes; `busy`. Disponível a partir de `authorized`.
- `AccountabilityForm`: `<input type="file" accept="application/pdf,image/jpeg,image/png">` rotulado "Comprovante de comparecimento"; "Enviar e concluir o processo" confere `accountabilityFileProblem` **antes** de chamar o cliente (nada de `readBase64` antes da conferência), trava no envio, nunca mostra nem guarda o nome do arquivo (o rótulo diz só "arquivo escolhido: PDF/PNG/JPG, <n> KB"); sucesso → "Processo concluído com a prestação de contas." e o comprovante aparece como "comprovante enviado em dd/mm/aaaa". Disponível em `travelling` (e em `scheduled` com a volta realizada).
- `TfdQueue`: `CaseView` recebe `<LegsPanel/><AllowancesPanel/><PaymentForm/><AccountabilityForm/>` como `children`, cada um em `Panel` próprio, conforme a situação.

- [ ] **Step 4: Rode e veja passar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/modules/tfd src/modules/TfdQueue.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/TfdQueue.tsx src/modules/tfd/LegsPanel.tsx src/modules/tfd/AllowancesPanel.tsx src/modules/tfd/PaymentForm.tsx src/modules/tfd/AccountabilityForm.tsx src/modules/tfd/CaseMoney.test.tsx
$G commit -m "feat: add TFD legs, allowances, payments and accountability upload" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Fonte `transport` na linha do tempo e painel de transporte e TFD (F-30.8)

**Files:**
- Modify: `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/TransportPanel.tsx`
- Test: `src/lib/timeline.transport.test.ts`, `src/modules/TransportPanel.test.tsx`

**Interfaces:**
- Consumes: `TimelineKind` com `"transport"` (Task 1); `getTransportDashboard` (Task 1); `fmtCount`, `fmtCents`, `SMALL_COUNT`, `TR_DASHBOARD_KEY`, `STATIC_QUERY` (Task 2); `canUseTfd`, `tfdStatusLabel` (Task 3); `monthProblem`, `currentMonth` (`src/lib/vaccination.ts`, reusados).
- Produces: `TIMELINE_KINDS` com `{ key: "transport", label: "Transporte" }`; `timelineKinds(examTracking, procedureBilling = false, pharmacy = false, vaccination = false, transport = false)`; `TransportPanel` com `occupancyText30(v: TransportCount): string` exportada ("80%" ou "oculto" quando `"lt5"`).

- [ ] **Step 1: Escreva os testes**

`src/lib/timeline.transport.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { detailTarget, timelineKindLabel, timelineKinds } from "./timeline";
import type { TimelineEntry } from "./api";

describe("linha do tempo: transporte (módulo 30)", () => {
  it("filtro só com sanitary_transport; rótulo", () => {
    expect(timelineKinds(false, false, false, false, true).map((k) => k.key)).toContain("transport");
    expect(timelineKinds(false).map((k) => k.key)).not.toContain("transport");
    expect(timelineKindLabel("transport")).toBe("Transporte");
  });
  it("entrada de viagem não abre detalhe (D12)", () => {
    const entry: TimelineEntry = { id: "e1", kind: "transport", occurred_at: "2026-11-03T05:30:00-03:00", unit: null, professional: null,
      title: "Viagem para Curitiba", summary: null, status: null, detail_ref: null };
    expect(detailTarget(entry, true)).toBeNull();
  });
});
```

`src/modules/TransportPanel.test.tsx`:

```tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import * as api from "../lib/api";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY30, trUser } from "../test/transportFixtures";
import { TransportPanel } from "./TransportPanel";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getTransportDashboard: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

const DASH = { trips: 42, occupancy_pct: 78, no_shows: "lt5", km: 3150, fuel_cents: 412300, exceptions: "lt5",
  tfd_by_status: { submitted: 6, authorized: "lt5", completed: 12 }, tfd_paid_cents: 98000 };

describe("painel de transporte e TFD", () => {
  it("contagens pequenas como 'menos de 5', nunca somadas; valores em R$", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "municipal_admin" ]));
    vi.mocked(api.getTransportDashboard).mockResolvedValue(DASH as never);
    renderWithProviders(<TransportPanel today={TODAY30} />);
    fireEvent.click(await screen.findByRole("button", { name: "Ver painel" }));
    await waitFor(() => expect(api.getTransportDashboard).toHaveBeenCalledWith("2026-11"));
    expect(await screen.findByText("42")).not.toBeNull();
    expect(screen.getAllByText("menos de 5").length).toBe(3);
    expect(screen.getByText("aguardando a comissão")).not.toBeNull();
    expect(screen.queryByText(/Total de processos/)).toBeNull();
    expect(screen.getByText(/R\$\s4\.123,00/)).not.toBeNull();
  });
  it("sem TFD ligado, o bloco do TFD some", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(trUser([ "transport_dispatcher" ], [ "sanitary_transport" ]));
    vi.mocked(api.getTransportDashboard).mockResolvedValue(DASH as never);
    renderWithProviders(<TransportPanel today={TODAY30} />);
    fireEvent.click(await screen.findByRole("button", { name: "Ver painel" }));
    expect(await screen.findByText("42")).not.toBeNull();
    expect(screen.queryByText("TFD por situação")).toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/timeline.transport.test.ts src/modules/TransportPanel.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `timeline.ts`: `TIMELINE_KINDS` ganha `{ key: "transport", label: "Transporte" }` depois de "Vacina"; `timelineKinds` ganha o quinto parâmetro `transport = false` e o filtro `(k.key !== "transport" || transport)`; comentário: "o Transporte, com o transporte sanitário — módulo 30 (a entrada de TFD também é `transport`, com título 'TFD para <cidade>'; nenhuma abre detalhe, D12)". `PatientTimeline.tsx`: passa `hasFeature(user, "sanitary_transport")`.
- `TransportPanel` (portão `r.admin || r.dispatcher || r.committee`): molde do `VaccinationPanel` (mês, "Ver painel", `monthProblem`), `useQuery([TR_DASHBOARD_KEY, month], …, STATIC_QUERY)`; `StatTile` para Viagens, Ocupação (`occupancyText30`), Faltas, Km rodados, Combustível (R$), Viagens sem atendimento marcado (`exceptions`); com `canUseTfd(user)`, `Panel` "TFD por situação" (`DataTable` situação → quantidade, `fmtCount`) e `StatTile` "Pago no TFD (R$)". Texto fixo: "Contagens de 1 a 4 aparecem como 'menos de 5' para não identificar ninguém; elas nunca entram em soma." Nenhum total calculado na tela.

- [ ] **Step 4: Rode e veja passar** (inclui os testes antigos da linha do tempo)

Run: `$R/bin/test dashboard -w dashboard-mod30 src/lib/timeline src/modules/timeline src/modules/TransportPanel.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/timeline.ts src/lib/timeline.transport.test.ts src/modules/timeline/PatientTimeline.tsx src/modules/TransportPanel.tsx src/modules/TransportPanel.test.tsx
$G commit -m "feat: add the transport timeline source and the transport and TFD panel" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Suíte, build, revisão e prova no navegador (api real na 3052)

**Files:** nenhum novo (correções que a prova pedir entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod30
$R/bin/test dashboard -w dashboard-mod30
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **17 arquivos de teste novos**: `src/lib/api.transport.test.ts`, `src/lib/transport.test.ts`, `src/lib/tfd.test.ts`, `src/shell/modules.transport.test.ts`, `src/modules/transport/TransportGate.test.tsx`, `src/modules/TransportSettings.test.tsx`, `src/modules/TransportFleet.test.tsx`, `src/modules/regulation/ScheduleOutcome.provider.test.tsx`, `src/modules/TransportTrips.test.tsx`, `src/modules/transport/Passengers.test.tsx`, `src/modules/TransportSeries.test.tsx`, `src/modules/TransportMyTrips.test.tsx`, `src/modules/tfd/TfdRequest.test.tsx`, `src/modules/TfdQueue.test.tsx`, `src/modules/tfd/CaseMoney.test.tsx`, `src/lib/timeline.transport.test.ts`, `src/modules/TransportPanel.test.tsx`. Contagem que dobra = artefato de build descoberto pelo vitest (lição do admin): confira antes de seguir.

- [ ] **Step 2: Varredura de dado sensível, de releitura e da separação central × laudo**

```bash
cd $W
grep -rnE "refetchInterval" src/modules/transport src/modules/tfd src/modules/Transport*.tsx src/modules/Tfd*.tsx && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" src/modules/transport src/modules/tfd src/modules/Transport*.tsx src/modules/Tfd*.tsx src/lib/transport.ts src/lib/tfd.ts && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "Tfd|/tfd" src/modules/transport src/modules/TransportTrips.tsx src/modules/TransportSeries.tsx src/modules/TransportPending.tsx src/modules/TransportFleet.tsx src/modules/TransportMyTrips.tsx && echo "ERRO: central lê TFD" || echo ok-separacao
grep -rnE "cid10|justification|sigtap" src/modules/transport src/modules/TransportMyTrips.tsx && echo "ERRO: dado clínico na central" || echo ok-clinico
sed -n '/Transporte sanitário e TFD (módulo 30/,$p' src/lib/api.ts | grep -nE "params\.(set|append)\(" | grep -vE '"(date|vehicle_id|destination_id|from|to|month|status|before|consultation_id)"' && echo "revisar" || echo ok-url
```

Expected: `ok-polling`, `ok-storage`, `ok-separacao`, `ok-clinico`, `ok-url`.

- [ ] **Step 3: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, na separação central × laudo, no que o 26a reusado pode ter quebrado (agendamento externo, prestadores) e no shell do motorista (nenhum alerta de infra para quem só tem `driver`).

- [ ] **Step 4: Prova no navegador** (só com o api do 30 mergeado ou no worktree dele, na 3052, e o banco de dev migrado e semeado **com autorização do usuário**)

Suba o Vite do worktree na rede do compose, porta **5209**, com proxy para `http://api:3052` (mesmo comando da prova do 29, trocando porta e worktree), abra `curitiba.localhost:5209/dashboard/` e percorra, com `sanitary_transport` e `tfd` ligados:
1. Equipe: convidar `transport_dispatcher`, `tfd_committee` (step-up) e `driver` (sem step-up). Unidades: criar a "Secretaria de Saúde" com o tipo "Gestão" e CNES. Regras: distância mínima, acompanhante e a unidade do BPA do TFD.
2. Frota: veículo van (15 lugares, 1 cadeira) e carro; motorista D válido, motorista B, motorista com CNH a vencer (alerta); destinos Curitiba (110 km), Ponta Grossa (30 km) e um metropolitano.
3. Regulação: prestador externo com IBGE; agendamento externo com o prestador.
4. Viagens do dia: viagem planejada sem escala; montar com os atendimentos marcados; passageiro sem atendimento (motivo); acompanhante fora da regra (recusa e motivo); lotação; confirmar com B na van (recusa), com D (ok); segunda viagem no mesmo horário com o mesmo veículo (as duas frases de sobreposição); PDF sem dado clínico.
5. Séries: hemodiálise seg/qua/sex; ver as viagens criadas pelo job; suspender; pendência sem vaga.
6. Motorista em **375×812** (`resize_window` mobile, recarregar): iniciar, marcar embarcou/faltou (inclusive acompanhante), tentar fechar com pendente, fechar com odômetro e abastecimento; na central, corrigir presença com step-up; ver a falta em "Pendências e faltas".
7. TFD: médico finaliza consulta → "Pedir TFD" (Ponta Grossa recusa; Curitiba ok) → comissão lê (trilha) → autorizar sem atendimento (barrado) → marcar atendimento → autorizar com step-up → trecho de frota ligado ao passageiro (embarque marcado pelo motorista) → volta por passagem marcada realizada → ajuda de custo calculada → pagamento → prestação de contas com PNG → `completed`; devolver outro pedido e o médico complementar em "Meus pedidos".
8. Linha do tempo ("Viagem para Curitiba", "TFD para Curitiba", sem detalhe), painel com "menos de 5".
9. Tablet (1024×768) e só teclado nos fluxos 4 e 7; telefone (375×812) nas listas (cartões). Ao terminar, `resize_window` preset desktop.

Anote cada passo com o que viu. Achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai para o usuário (não vira tarefa).

- [ ] **Step 5: Board e encerramento**

Cards do dashboard prontos: F-30.2..F-30.6 → **Done** se o api já terminou o deles; F-30.1 espera o maintenance e F-30.8 o wpda (o último app move); F-30.7 é do api. **Verified** só depois da prova do Step 4, com o usuário. Merge, push e tag só com autorização. Worktree: `$G -C $R/apps/dashboard worktree remove $W` depois do merge.

---

## Divergências propostas ao contrato

Cada uma diz o que a tela precisa, o que o plano assume e quem decide. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D1 — Busca de paciente pela central.** O contrato pede `patient_id` em passageiro e série, mas a central não tem como achar a pessoa (sem atendimento marcado ou série nova). Proposta: `POST /transport/person_lookup { cpf }` → `{ patient_id, display_name, has_cpf, cns_masked }` e `POST /transport/patients/duplicates { full_name, birth_date, mother_name? }` (forma e serviço do 24a/V21), portão `sanitary_transport` + `transport_dispatcher`, com trilha como no 27a. A central **não cria paciente**: sem cadastro, a tela manda ao balcão.
- **D2 — Quem pode virar motorista.** `POST /transport/drivers` exige `user_id` com papel `driver`, mas a central não lê `/team`. Proposta: `GET /transport/driver_candidates` → `{ items: [ { user_id, name } ] }` (usuários com membership `driver` ativa e sem cadastro de motorista); recusa `user_not_driver` (422) quando o `user_id` não tem o papel.
- **D3 — Paradas na escrita da viagem.** O contrato diz `stops: [..]` sem forma. Proposta: `{ id?, position, kind: "boarding"|"destination", destination_id?: string|null, label }`; `PATCH` substitui a lista inteira, mantendo as paradas que vêm com `id`; parada com passageiro não sai (`stop_in_use`, 422); `label` do destino vazio vira o nome do destino.
- **D4 — Item de `scheduled_care`.** `destination: {…}` sem forma. Proposta: `destination: { destination_id: string|null, provider_name: string|null, address: string|null }` (`destination_id` nulo quando o agendamento não tem prestador estruturado — a central escolhe); a chave estável é `origin` + `origin_id`; itens de série entram com `origin: "series"`.
- **D5 — Série.** A leitura não tem forma. Proposta: `<series>`: `{ id, patient: { id, display_name }, destination: { id, name }, provider_name|null, weekdays: [int], appointment_time, departure_time, starts_on, ends_on|null, wheelchair, companion: { name, document_last4|null }|null, status: "active"|"suspended"|"ended", suspended_from|null }`; `weekdays` em ISO 8601 (1 = segunda … 7 = domingo).
- **D6 — Faltas.** `GET /transport/no_shows` sem forma. Proposta: `{ items: [ { patient: { id, display_name }, count, dates: [ "YYYY-MM-DD" ] } ] }`, ordenado por `count` decrescente; sem destino nem origem.
- **D7 — Despesas e correção.** `expenses[].kind` sem valores e a correção sem corpo. Proposta: `kind`: `toll` | `parking` | `meal` | `other` (`note` obrigatória em `other`); `POST /transport/trips/:id/attendance_correction { items: [ { passenger_id, attendance, companion_attendance? } ], reason }` (motivo 10–500, step-up) → `<trip>`.
- **D8 — Opções do laudo.** O médico e a comissão precisam dos destinos e prestadores, mas `GET /transport/destinations` é da central. Proposta: `GET /tfd/options` → `{ destinations: [<destination> ativos], providers: [ { id, name, ibge_code } ], min_distance_km }`, para `health_professional` com `tfd` e para `tfd_committee`.
- **D9 — Pedidos do médico.** O autor edita em `draft`/`returned`, mas só a comissão lista. Proposta: `GET /tfd/my_cases` → `{ items: [ { id, patient: { display_name }, destination: { name }, status, submitted_at|null } ] }` (só os do autor, sem dado clínico); o autor abre pelo `POST /tfd/cases/:id/read` (com trilha).
- **D10 — Trecho de frota e acompanhante na passagem.** A comissão precisa escolher o `trip_passenger_id` e dizer se o acompanhante viajou na passagem. Proposta: `<tfd-case>.trip_passengers: [ { id, date, departure_at, attendance, companion_attendance } ]` (passageiros com `origin: "tfd"` deste processo); `TfdLegInput.companion?: boolean` só para passagem, e `<tfd-leg>.companion: boolean`.
- **D11 — Upload da prestação de contas.** O §6 diz multipart; a convenção do módulo é "JSON em toda escrita por cookie" e o dashboard sobe arquivo em base64 (`uploadExamFile` do 21a, D1 dele). Proposta: `POST /tfd/cases/:id/accountability { content_type, data }` (PDF/JPG/PNG, 10 MB **decodificados**, `file_too_large`/`invalid_file_type`), **sem o nome do arquivo**; `document.filename` é gerado pelo api (ex.: `comprovante.pdf`).
- **D12 — Respostas do TFD e linha do tempo.** (a) As escritas do TFD (`decide`, `appointment`, `inpatient`, `legs`, `legs/:id/done`, `payments`, `accountability`) não têm resposta fixada; proposta: todas devolvem o `<tfd-case>` inteiro, sem nova trilha além da leitura. (b) O §9 usa `at` e `detail_ref.kind`; a forma real da linha do tempo é `occurred_at` e `detail_ref.type`. Proposta: `{ kind: "transport", occurred_at, unit: null, professional: null, title, summary: null, status, detail_ref: null }` — não há tela de detalhe de viagem, e abrir o TFD seria dado clínico fora da comissão.
- **D13 — Prestador no 26a.** O §1 cita `care_providers.ibge_code` e a spec o `care_provider_id` no agendamento externo, mas o contrato não dá as formas. Proposta: `CareProvider.ibge_code: string|null` em `GET/POST/PATCH /regulation/providers`; `POST /regulation/requests/:id/schedule` aceita `care_provider_id?` (prestador ativo, não próprio); `schedule.care_provider: { id, name }|null` na leitura.
- **D14 — Lista da comissão com cursor.** `GET /tfd/cases?status=&before=` → `{ items, next_before }` (a convenção de listas), ordenada por `submitted_at` crescente (mais antigo primeiro).
- **D15 — Limites de texto.** Motivos 10–500; justificativa 20–2000; nome do acompanhante 3–120; documento ≤ 20; rótulo de parada 3–120; posto, empresa e observação ≤ 120; recibo 1–40; modelo 2–80.
- **D16 — Destino do passageiro com nome.** O motorista não lê `/transport/destinations`. Proposta: `<passenger>.destination` ganha `name` (município do destino).
- **D17 — Códigos da ajuda na configuração.** Para o admin preencher o complemento por código, `GET /transport/admin/settings` ganha `allowance_codes: [ { code, name, kind } ]` (só leitura, de `config/tfd/sigtap_codes.yml`); recusa da unidade do BPA: `billing_unit_invalid` (não é `health_management` ou não tem CNES).
- **D18 — Viagem só no dia.** `POST /transport/my_trips/:id/start` fora do dia ou de viagem não `confirmed`: códigos `trip_not_today` e `invalid_transition` (422).

## Self-review

1. **Cobertura da spec:** decisão 1 → Tasks 9 e 14 (TFD entra na mesma lista pela origem `tfd`); 2 → 14 (registra, não paga); 3 → 11; 4 → 2 e 9 (motivo de lista fechada e texto); 5 → 2 e 6; 6 → 2, 6 e 8 (categoria D, CNH vencida bloqueia, aviso de 30 dias); 7 → 2, 8 e 9 (sobreposição em 422, lotação e cadeira); 8 → 6 e 7; 9 → 8 e 9; 10 → 9 (sugestões por origem; a regra intramunicipal da agenda é do api); 11 → 5 e 9; 12 → 10; 13 → 11; 14 → 9 e 11 (lista, PDF e motorista sem dado clínico; guarda na Task 16); 15 → 12; 16 → 3 e 12; 17 → 13; 18 → 14 (calculada pelo api, mostrada); 19 → 14; 20 → só api (Task 5 configura o CNES); 21 → wpda (plano próprio); 22 → 15; 23 → 1, 3 e 4. Decisões do coordenador: destino estruturado → 7 e 9; papéis → 4; trechos e internação → 13 e 14; viagem sem escala → 8 e 10; LGPD e avisos → api (a tela avisa no cancelamento e na retirada). §5 → Global Constraints e Task 16 Step 2.
2. **Passos:** cada passo de código diz arquivo, assinatura e os valores da spec/contrato; corpos só para a fixture e os testes. Os textos do `origin/main` que os testes usam (rótulos do `ScheduleOutcome` e do `ProviderForm`, "Buscar pelo CPF" do `PersonStart`, `stepUpMfa`, `regRequest`/`regUser`) foram conferidos no `022ccd3`.
3. **Tipos:** `Trip`, `TripPassenger`, `TfdCase`, `TfdLegInput`, `PassengerDraft`, `TripDraft` e as funções da Task 1 são os mesmos nas Tasks 2–15; `TransportRoles` nasce na Task 4 e serve a todos os portões; `CaseView.children` (Task 13) recebe os painéis da Task 14; `TripCard.children` (Task 8) recebe a lista (Task 9) e a correção (Task 11).
4. **Review Focus:** as cinco linhas têm teste na task dona (11; 8; 2 e 8; 13; 14).
5. **Proporção:** o código é teste e fixture; implementação por assinatura e valores.
