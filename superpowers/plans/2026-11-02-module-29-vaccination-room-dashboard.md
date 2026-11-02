# Módulo 29 — Sala de vacina (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 29** (`feat/mod-29-vaccination-room`, worktree `apps/api/.claude/mod29`, porta de dev **3051**) com as rotas do contrato §3–§11 (e as Divergências D1–D19 deste plano que o plano do api adotar), os interruptores `vaccination`, `rnds_immunization` e `rnds_mock` na sessão, a credencial `rnds`, os locais `vaccine_room`/`cold_chain_central` e a semente do 29. Ele é **mergeado antes** deste (contrato §14: api → dashboard → wpda → maintenance). As Tasks 0–15 não dependem dele (dublês de `src/lib/api` com fixtures nas formas do contrato §2); a prova da Task 16 depende.

**Goal:** Lado dashboard do 29: **administração** (salas de vacina e equipe, rede de frio municipal e a equipe dela, data do go-live da cidade, credencial **RNDS** — certificado A1 — na tela de Integrações), a **sala de vacina** (busca do cidadão → **caderneta** conferida ou "não conferida" → **registro da dose** com lote do estoque da sala e frasco aberto, **avisos com motivo** — intervalo e idade —, **transcrição** de caderneta, **cancelamento** com motivo e substituição, situação do envio à RNDS por dose e **pendências do envio** da sala), **imunobiológicos e frascos** (entrada e inventário inicial na rede de frio, transferência às salas, recebimento, posição com lotes bloqueados, abertura e descarte de frasco, ajustes — reaproveitando as peças do 27a), **resumo mensal** no formato do SIES/SIPNI, **cadeia de frio** (equipamentos, leituras, excursão e conduta), **painel de vacinação**, critérios **"vacina em atraso"** e **"caderneta não conferida"** e a contagem de **convocados vacinados** nas campanhas, a fonte **`vaccination`** na linha do tempo, **"não conferido"** nas linhas de cuidado e o botão **"Notificar ESAVI"** na dose.

**Architecture:** O cliente HTTP ganha a seção "Sala de vacina (módulo 29)" no fim de `src/lib/api.ts` (tipos = formas do contrato §2–§11; `CredentialKind`, `StockItem`, `LotBalance`, `TimelineKind`, `Criterion`, `CampaignOptions` e `Campaign` ganham o que é da vacina). As regras de tela ficam puras em `src/lib/vaccination.ts` (portão, papéis por local, rótulos, situação efetiva da caderneta, espelho das recusas da dose e da transcrição, avisos com motivo, lotes e frascos oferecidos, temperatura, frases). As peças ficam em `src/modules/vaccination/` e as telas no grupo novo **"Vacinação"** (`modules.ts`, `catalog.ts`, `App.tsx`), depois de "Farmácia". O estoque **não é tela nova do zero**: `TransferForm`, `ReceiveForm`, `AdjustmentForm`, `PendingAdjustments`, `LotStatement` e `StaffPanel` do 27a são reusados com mudanças pequenas (identidade do item por `itemKey`, destinos e papéis por prop), e só a entrada de imunobiológico (fabricante por código, doses por frasco) e a posição com frascos são peças novas. O dashboard nunca decide o que o api garante: calendário, aprazamento, atraso, imutabilidade, saldo, validade do frasco, envio à RNDS, trilha e retenção LGPD são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-02-module-29-vaccination-room-design.md` (§1 intenção, sucesso e antiobjetivos; §2 decisões 3a, 3–18 e as do coordenador; §4 telas; §5 segurança e LGPD; §6 testes; §8 F-IDs; §9 riscos), `docs/adr/0047.md`, a pesquisa `docs/pesquisa/2026-11-02-sala-de-vacina-29.md` e o contrato `docs/superpowers/plans/2026-11-02-module-29-vaccination-room-contracts.md` (§1–§11, §13, §14), com as convenções herdadas (27a §13, 28 §16). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D19); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptores `vaccination` (requer `record_mode:record`), `rnds_immunization` (requer `feature:vaccination credential:rnds`), `rnds_mock` (só dev/test); credencial `rnds` com segredo `{ pfx_base64, pfx_password, requester_cnes }` (nunca devolvido); locais `vaccine_room` (um por unidade) e `cold_chain_central` (um por cidade); equipe da sala `vaccinator` | `room_lead` (`room_lead` exige CBO `2235*`; vacinador CBO `3222*`, `2235*`, `2251*`, `2252*`, `2253*`); rede de frio `stock_keeper`; `rnds_status` `not_sent` | `pending` | `sending` | `accepted` | `rejected` | `failed` | `blocked`; motivo de bloqueio `professional_cns_not_in_cnes` | `patient_identity_unverified` | `patient_without_cns_or_cpf`; conferência `transcribed` | `confirmed_complete` | `born_after_go_live`; situação por vacina `due` | `overdue` | `upcoming` | `complete` | `not_applicable` | `unchecked`; frasco `open` | `expired` | `closed_out`; equipamento `refrigerator` | `chamber` | `freezer`; terminologias da RNDS (`immunobiological`, `dose`, `strategy`, `route`, `site`, `manufacturer`, `care_group`, `maternal_condition`) **sem tradução** — o nome vem do catálogo do api; código desconhecido aparece cru, nunca some.
- **Interruptor** (spec decisão 18): sem `vaccination` na sessão, o dashboard é o de hoje — o grupo "Vacinação" some, a campanha não oferece os critérios de vacina (o api só manda `immunobiologicals` nas opções com o interruptor ligado, D12), a linha do tempo não oferece o filtro "Vacina". A dose **já registrada** continua legível na linha do tempo (registro oficial retido, ADR 0026). 403 `feature_disabled` → `VACCINATION_DISABLED`. O dashboard nunca escreve interruptor; as três chaves ganham só rótulo em `features.ts`. `vaccination` **não** exige `pharmacy` (decisão do coordenador): nada das telas de vacina depende do grupo "Farmácia".
- **Quem faz o quê** (contrato §3, §5, §7, §8): `municipal_admin` administra salas, equipe, rede de frio, go-live e a credencial; na sala, **qualquer** `vaccinator` (e o `room_lead`) lê a caderneta, aplica, transcreve, abre frasco, registra temperatura e recebe transferência; **só** o `room_lead` descarta frasco, faz ajuste/perda/inventário e decide a conduta da excursão; na rede de frio, o `stock_keeper` faz entrada, transferência, ajuste e conduta; cancelar dose é do autor ou do `room_lead` (`can_cancel`, D18); o painel é do `municipal_admin` e do `analyst`. O papel **por local** vem de `GET /vaccination/my_locations` (D2); a tela não oferece o que o papel não permite, e o api recusa de novo.
- **Caderneta não conferida** (spec decisão 3a; ADR 0047, invariante): sem `checked`, a tela **nunca** mostra "em atraso" — um item `overdue` vindo com `checked: null` é mostrado como `unchecked` ("não conferida") por `effectiveStatus`; o aviso `UNCHECKED_NOTICE` aparece com as duas saídas ("Transcrever caderneta" e "Confirmar caderneta completa").
- **Leitura nominal com trilha, sem polling** (spec §5; contrato §5): caderneta (`POST …/card`), detalhe da dose e o retorno da transcrição são lidos por **mutação no clique** ou depois de uma ação (`gcTime: 0`), nunca sozinhos; pendências, frascos, posição, cadeia de frio, resumo mensal e painel são sem pessoa: `STATIC_QUERY` (ao abrir, ao filtrar, depois de uma ação). Nenhum `refetchInterval` no módulo.
- **Dado pessoal ou clínico só no corpo** (spec §5; brief): CPF, CNS, nome, vacina, dose, lote do paciente, motivos e justificativas nunca na URL, em mensagem de erro, em `console`, em Analytics, em `localStorage`, `sessionStorage` ou IndexedDB. Na URL vão só ids (`room_id`, `location_id`, `health_unit_id`), `status`, `month`, `before` e `limit`. Certificado e senha da RNDS só no corpo do `PUT` e nunca voltam; o nome do arquivo `.pfx` não sai da tela.
- **Todas as recusas de uma vez** (27a §13 H8; 28 §16): a tela confere o registro da dose, a transcrição, a leitura de temperatura e a conduta inteiros antes de enviar e mostra cada problema; a recusa do api com `problems: [ { "path", "error" } ]` mostra todas, traduzidas; sem `problems`, a do topo. **Avisos com motivo** (contrato §5): `interval_too_short` e `age_out_of_range` são avisos — a tela mostra a frase e o campo "Motivo para aplicar mesmo assim" (10–500); com o motivo, reenvia com `override_reason`; se houver recusa dura junto (ex.: `lot_expired`), o motivo não basta e a frase dela continua.
- **Recusar em vez de cortar** (21a §15): data futura, lote vencido ou bloqueado, frasco vencido ou vazio, temperatura fora de −50…50 °C, mínima acima da atual, motivo curto — recusa com a frase; nada é arredondado, cortado nem descartado em silêncio. Temperatura aceita vírgula ou ponto, uma casa decimal.
- **Tablet e teclado** (brief): toda a sala funciona sem mouse — formulários nativos (`<form>` com `Enter` enviando, `<select>` e `<input>` com rótulo), ordem de Tab = ordem visual, foco vai ao título da caderneta quando ela abre e à frase de sucesso depois de registrar; alvo de toque ≥ 44 px (`buttonStyle` e `inputStyle` existentes); situação nunca só por cor (texto no `Tag`). Sem foco escondido (`outline: "none"` proibido pela guarda).
- **Sem duplo envio**: todo botão de escrita fica desabilitado enquanto a chamada corre (`busy`), e uma resposta de uma ação já trocada (outro paciente, outro local) é ignorada (`seq`), como no `PrescriptionLookup` do 27a.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo — `{}` quando não há campo).
- Textos (D14; se o api fixar outros, o api vence): `override_reason`, motivo de cancelamento, de descarte de frasco e de cada conduta 10–500; nome da sala e do equipamento 3–120; lote da transcrição ≤ 40; fabricante e local da transcrição ≤ 120; CNES solicitante 7 dígitos; certificado `.pfx`/`.p12` até 100 KB.
- "Hoje" é o dia da cidade (`todayInCity()` de `src/lib/campaigns.ts`) e "agora" é `new Date()`; telas recebem `today`/`now` por prop (padrão o real) e os testes passam `TODAY29 = "2026-11-02"` (segunda) e `NOW29 = new Date("2026-11-02T10:00:00-03:00")`, sem relógio falso. `datetime-local` é lido como hora de parede da cidade (`parseCityLocal` de `src/lib/format.ts`).
- Interface (ciclo de interface, subprojetos 1–6 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `formStyles`, os estilos de `src/modules/exams/styles.ts`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-29.1..F-29.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-29.1: api, dashboard e maintenance; F-29.7: api, wpda e dashboard; F-29.2..F-29.6 e F-29.8: api e dashboard) e a **Verified** depois da prova no navegador (Task 16).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod29` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-29-vaccination-room` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod29`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test dashboard -w dashboard-mod29 <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx`; a sessão entra por `fetchCurrentSession` mockado (como nos testes da farmácia); as fixtures do 27a (`position`, `lotBalance`, `stockLot`) vêm de `src/test/pharmacyFixtures.ts`.
- Step-up nos testes: `sessionWith` grava `mfa_verified_at` agora, então a janela de 5 min está aberta e o `SensitiveAction` confirma sem pedir código (o fluxo do código já é coberto por `SensitiveAction.test.tsx`); os testes clicam direto no botão de confirmação.
- A prova (Task 16) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5207**, com proxy para o api do 29 na **3051**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Caderneta não conferida de criança de 3 anos, cujo api devolve um item `overdue` (ou a Penta com `due_on` no passado).** Espera-se: nenhuma palavra "atraso" na tela; o item aparece "não conferida" com o aviso e as duas saídas; depois de "Confirmar caderneta completa" a caderneta volta conferida e só então mostra "em atraso". Teste: Task 2, "effectiveStatus: overdue sem conferência vira unchecked", e Task 6, "caderneta não conferida nunca mostra atraso; confirmar completa relê".
2. **O lote escolhido é bloqueado por uma excursão (ou vence à meia-noite) entre abrir o formulário e clicar em Registrar.** Espera-se: a recusa `lot_blocked`/`lot_expired` aparece na linha do lote, a posição da sala é relida, a escolha do lote é limpa e o resto do formulário (vacina, dose, estratégia, via, local, hora) fica como estava; nada é reenviado sozinho. Teste: Task 7, "lot_blocked: relê a posição, limpa o lote e mantém o resto".
3. **Duplo clique (ou `Enter` duas vezes no tablet) em "Registrar dose".** Espera-se: uma chamada só a `recordImmunization`; o botão fica desabilitado até a resposta; a frase de sucesso aparece uma vez. Teste: Task 7, "duplo envio registra uma dose só".
4. **Aviso de intervalo curto junto com uma recusa dura (frasco vencido).** Espera-se: as duas frases de uma vez; o campo "Motivo para aplicar mesmo assim" aparece, mas reenviar só com o motivo continua barrado na tela até trocar o frasco; motivo com menos de 10 caracteres é recusado antes de enviar; com frasco válido e motivo, o corpo leva `override_reason`. Teste: Task 7, "aviso e recusa dura juntos: o motivo não basta".
5. **Temperatura digitada com vírgula ("8,5"), negativa ("-1") ou com a mínima acima da atual.** Espera-se: "8,5" vira `8.5` no corpo; "-1" é aceito como número e, fora de +2 a +8 °C, a tela avisa antes de enviar que os lotes do local ficarão bloqueados e, depois do 201 com `excursion_id`, abre a conduta; mínima > atual ou atual > máxima é recusa com frase; "8,55" (duas casas) é recusa. Teste: Task 2, "parseCelsius e readingProblems", e Task 12, "leitura fora da faixa abre a conduta".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.vaccination.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/vaccinationFixtures.ts` | tipos e cliente HTTP do 29; interruptores; proxy; dados de teste | 1 |
| `src/lib/vaccination.ts`, `src/lib/vaccination.test.ts`, `src/lib/pharmacy.ts` (`itemKey`, rótulos dos locais novos) | regras de tela: portão, papéis por local, rótulos, situação efetiva, recusas, avisos, lotes, frascos, temperatura, frases | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.vaccination.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/vaccination/VaccinationGate.tsx`, `src/modules/vaccination/VaccineLocationPick.tsx`, `src/modules/vaccination/VaccinationGate.test.tsx`, telas-portão `src/modules/Vaccination{Room,Pending,Stock,ColdChain,Summary,Panel,Admin}.tsx` | grupo "Vacinação", portão, escolha do local | 3 |
| `src/modules/VaccinationAdmin.tsx`, `src/modules/vaccination/RoomsPanel.tsx`, `src/modules/vaccination/RoomStaffPanel.tsx`, `src/modules/vaccination/GoLivePanel.tsx`, `src/modules/pharmacy/StaffPanel.tsx`, `src/modules/VaccinationAdmin.test.tsx` | salas, equipe, rede de frio e equipe dela, go-live (F-29.1) | 4 |
| `src/lib/integrations.ts`, `src/lib/integrations.test.ts`, `src/modules/Integrations.tsx`, `src/modules/integrations/RndsCredentialForm.tsx`, `src/modules/Integrations.rnds.test.tsx` | credencial RNDS (certificado A1) e pendências novas dos interruptores (F-29.1) | 5 |
| `src/modules/vaccination/PatientPick.tsx`, `src/modules/vaccination/CardView.tsx`, `src/modules/vaccination/ImmunizationRow.tsx`, `src/modules/VaccinationRoom.tsx`, `src/modules/vaccination/CardView.test.tsx` | busca do cidadão, caderneta, conferência (F-29.2) | 6 |
| `src/modules/vaccination/DoseForm.tsx`, `src/modules/vaccination/DoseForm.test.tsx` | registro da dose com lote, frasco, avisos com motivo e aprazamento (F-29.3) | 7 |
| `src/modules/vaccination/TranscriptionForm.tsx`, `src/modules/vaccination/CancelDose.tsx`, `src/modules/vaccination/Transcription.test.tsx` | transcrição, cancelamento e substituição (F-29.3) | 8 |
| `src/modules/vaccination/RndsStatusTag.tsx`, `src/modules/VaccinationPending.tsx`, `src/modules/VaccinationPending.test.tsx` | situação do envio por dose e pendências da sala (F-29.4) | 9 |
| `src/modules/pharmacy/TransferForm.tsx`, `src/modules/pharmacy/AdjustmentForm.tsx`, `src/modules/PharmacyTransfers.tsx`, `src/modules/vaccination/VaccineReceiptForm.tsx`, `src/modules/vaccination/VaccinePositions.tsx`, `src/modules/VaccinationStock.tsx`, `src/modules/VaccinationStock.test.tsx` | estoque de imunobiológicos com as peças do 27a (F-29.5) | 10 |
| `src/modules/vaccination/VialsPanel.tsx`, `src/modules/VaccinationSummary.tsx`, `src/modules/vaccination/Vials.test.tsx` | frascos abertos e descarte; resumo mensal (F-29.5) | 11 |
| `src/modules/VaccinationColdChain.tsx`, `src/modules/vaccination/EquipmentForm.tsx`, `src/modules/vaccination/ReadingForm.tsx`, `src/modules/vaccination/ExcursionDecision.tsx`, `src/modules/VaccinationColdChain.test.tsx` | cadeia de frio (F-29.6) | 12 |
| `src/lib/timeline.ts`, `src/lib/timeline.vaccination.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/timeline/TimelineDetail.tsx`, `src/modules/vaccination/ImmunizationDetail.tsx`, `src/modules/timeline/PatientTimeline.vaccination.test.tsx`, `src/lib/careLines.ts`, `src/lib/careLines.test.ts` | fonte `vaccination` na linha do tempo; "não conferido" nas linhas de cuidado (F-29.7, F-29.8) | 13 |
| `src/lib/campaigns.ts`, `src/lib/campaigns.vaccination.test.ts`, `src/modules/campaigns/AudienceBuilder.tsx`, `src/modules/campaigns/CriterionCard.tsx`, `src/modules/campaigns/CampaignPanel.tsx`, `src/modules/campaigns/Vaccination.test.tsx` | critérios de vacina e convocados vacinados (F-29.8) | 14 |
| `src/modules/vaccination/EsaviButton.tsx`, `src/modules/vaccination/ImmunizationRow.tsx`, `src/modules/VaccinationPanel.tsx`, `src/modules/vaccination/EsaviPanel.test.tsx` | botão ESAVI; painel de vacinação (F-29.8) | 15 |
| — | suíte, build, revisão, prova no navegador | 16 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, nome, vacina, dose, lote, motivo, senha, certificado) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa —: situação efetiva sem conferência, recusas e avisos da dose, transcrição, lotes oferecidos (bloqueado, vencido, de outra vacina), frascos (vencido, vazio, de outro lote), temperatura com vírgula e sinal, motivo curto (Task 2); menu e portão por papel e interruptor (Task 3); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e fixtures do contrato §2, cobrindo o caminho feliz, o teclado, cada recusa que muda a tela (`lot_blocked`, `vial_expired`, `vial_required`, `interval_too_short`, `cbo_not_allowed`, `room_lead_requires_nurse`, `already_exists`, `invalid_certificate`, `feature_disabled`), o step-up, a ausência de releitura sozinha e de armazenamento local (Tasks 4–15); as peças do 27a tocadas continuam com os testes delas verdes (Task 10). A prova final é no navegador, contra o api real na 3051 (Task 16).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`fadfa87` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod29`.
- Produces: cards F-29.1..F-29.8 em In Progress; worktree `$W` na branch `feat/mod-29-vaccination-room`; a base anotada; as rotas e as formas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-29\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api, do wpda e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-29.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "pharmacy";\|  pharmacy: "Farmácia e almoxarifado"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export type CredentialKind\|^export interface StockItem {\|^export interface LotBalance\|^function readBase64\|^const withQuery = \|^  | "dispensation";\|^export type Criterion =\|^export interface CampaignOptions {\|^export interface Campaign extends\|^export interface ReceiptItemInput'
$G -C $R/apps/dashboard show origin/main:src/lib/integrations.ts | grep -n 'const CREDENTIAL_LABEL\|case "credential_unauthorized":'
$G -C $R/apps/dashboard show origin/main:src/modules/Integrations.tsx | grep -n '{editing && (\|setIntegrationCredential(editing.kind'
$G -C $R/apps/dashboard show origin/main:src/lib/pharmacy.ts | grep -n 'const KIND_LABEL\|^export function pharmacyRefusals'
$G -C $R/apps/dashboard show origin/main:src/modules/pharmacy/TransferForm.tsx | grep -n 'l.kind === "unit"\|p.item.catalog_item_id === l.snapshot.item.catalog_item_id\|key={p.item.catalog_item_id}'
$G -C $R/apps/dashboard show origin/main:src/modules/pharmacy/AdjustmentForm.tsx | grep -n 'p.item.catalog_item_id === itemId\|<option key={p.item.catalog_item_id}'
$G -C $R/apps/dashboard show origin/main:src/modules/pharmacy/StaffPanel.tsx | grep -n '^const ROLES'
$G -C $R/apps/dashboard show origin/main:src/modules/PharmacyTransfers.tsx | grep -n '^function Body\|const canSend = '
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "pharmacy-alerts" | "pharmacy-admin";\|      if (item.id === "pharmacy-admin") return phOn && isAdmin;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 15\|NAV_GROUPS.length - 14'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n 'IconAlertTriangle, IconBuildingCog\|    "pharmacy-admin": { icon: IconBuildingCog'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "pharmacy-admin": return <PharmacyAdmin'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/pharmacy(/|$)": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '^export function timelineKinds\|^export type DetailType\|  dispensed: "dispensado"'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n 'timelineKinds('
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^const STATUS_LABEL\|^const STATUS_TONE\|if (p.status === "untracked")'
$G -C $R/apps/dashboard show origin/main:src/lib/campaigns.ts | grep -n '^export const CRITERION_KINDS\|^export function newCriterion\|^function cleanCriterion'
$G -C $R/apps/dashboard show origin/main:src/modules/campaigns/AudienceBuilder.tsx | grep -n '{CRITERION_KINDS.map'
$G -C $R/apps/dashboard show origin/main:src/modules/campaigns/CampaignPanel.tsx | grep -n '<Stat label="Lidos"'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod29 -b feat/mod-29-vaccination-room origin/main
cd $R/.worktrees/dashboard-mod29 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod29
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Vaccine|Needle|CloudUpload|VaccineBottle|TemperatureSnow|CalendarMonth|ReportMedical|Fridge)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os oito ícones (falta algum → escolha outro que exista e use o mesmo na Task 3; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 16 espera a base mais **18 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 29** (informativo para as Tasks 1–15)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-29-vaccination-room
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod29 api bin/rails routes 2>/dev/null | grep -E "vaccination|integrations|citizen/vaccinations|pharmacy/(receipts|openings|positions|transfers|adjustments)" | head -80
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-11-02-module-29-vaccination-room-api.md 2>/dev/null | grep -n "^## \|^### Task\|Divergência\|Desvio" | head -60
```

Expected: a lista das rotas. Se o api adotou outra forma para alguma Divergência (D1–D19), **use a do api** (ele é mergeado antes) e anote no commit da task que a consome; se não adotou, a task usa a forma da Divergência e a Task 16 confere.

---

### Task 1: Tipos, cliente HTTP, interruptores, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; `CredentialKind`, `StockItem`, `LotBalance`, `StockLot`, `TimelineKind`, `Criterion`, `CampaignOptions`, `Campaign` ganham campos), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/vaccinationFixtures.ts`
- Test: `src/lib/api.vaccination.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `readBase64`, `INTEGRATIONS_BASE`, `PHARMACY_BASE`/`phPath`/`phWrite` (existentes em `api.ts`); `checkIntegrationCredential` (existente); `sessionWith` (`src/test/campaignFixtures.tsx`); `position`, `lotBalance`, `stockLot` (`src/test/pharmacyFixtures.ts`).
- Produces (em `src/lib/api.ts`):
  - `CredentialKind = "ledi" | "cadsus" | "rnds"`;
  - `StockItem` passa a `catalog_item_id: string | null; catmat_code: number | null` e ganha `immunobiological_code?: string | null; item_kind?: "medication" | "immunobiological"` (D5); `StockLot` ganha `manufacturer_code?: string | null` (D5); `LotBalance` ganha `blocked?: boolean` (D5); `TimelineKind` ganha `"vaccination"`; `Criterion` ganha `{ kind: "vaccine_overdue"; immunobiological_codes: string[]; age_min_months?: number; age_max_months?: number }` e `{ kind: "vaccination_record_unchecked"; age_min_months?: number; age_max_months?: number }`; `CampaignOptions` ganha `immunobiologicals?: CodeName[]` (D12); `Campaign` ganha `vaccinated_count?: number | null` (D12);
  - tipos `VaccineLocationKind = "vaccine_room" | "cold_chain_central"`, `VaccineStaffRole = "vaccinator" | "room_lead"`, `RndsStatus`, `RndsBlockReason`, `RecordCheckMode`, `CardItemStatus`, `VialStatus`, `ColdChainEquipmentKind` (valores do §1 / Global Constraints);
  - `CodeName { code: string; name: string }`; `ImmunobiologicalManufacturer { code: string; name: string; validity_after_opening_hours: number | null }`; `Immunobiological { code; name; multidose: boolean; doses_per_vial: number; default_route: string | null; default_site: string | null; manufacturers: ImmunobiologicalManufacturer[] }`; `VaccinationTerminologies { dose: CodeName[]; strategy: CodeName[]; route: CodeName[]; site: CodeName[]; care_group: CodeName[]; maternal_condition: CodeName[] }` (D15); `VaccinationCatalog { calendar_version: string; immunobiologicals: Immunobiological[]; terminologies: VaccinationTerminologies }`;
  - `VaccineLocation extends PharmacyLocation { room_id: string | null }` (D2: `kind` `vaccine_room`/`cold_chain_central`; `roles` ⊂ `vaccinator`, `room_lead`, `stock_keeper`);
  - `ImmunizationRnds { status: RndsStatus | string; sent_at: string | null; accepted_at: string | null; block_reason: RndsBlockReason | string | null; error: string | null }`; `Immunization` (= `<immunization>` do §2, mais `can_cancel?: boolean` — D18);
  - `CardPatient { id; display_name: string | null; birth_date: string; sex: string; has_cpf: boolean; cns_masked: string | null }`; `CardCheck { mode: RecordCheckMode | string; at: string; by_name: string | null }`; `CardItem { immunobiological: CodeName; dose: CodeName; status: CardItemStatus | string; due_on: string | null; applied: { id: string; applied_at: string; transcription: boolean } | null }` (D15); `VaccinationCard { patient: CardPatient; checked: CardCheck | null; calendar_version: string; items: CardItem[]; immunizations: Immunization[] }`;
  - `CardRequest = { patient_id: string } | { citizen_id: string } | { new_patient: NewPatient; duplicates_reviewed?: boolean }` (D3; as três formas da notificação manual do 25); `ImmunizationInput { patient_id; immunobiological_code; dose_code; strategy; care_group; route; site; maternal_condition?: string; applied_at: string; lot_id: string; vial_id?: string; campaign_id?: string; override_reason?: string }`; `TranscriptionItemInput { immunobiological_code; dose_code; applied_on; lot_number?; manufacturer_name?; place? }`; `TranscriptionInput { patient_id: string; items: TranscriptionItemInput[]; close_check: boolean }`; `CancelInput { reason: string; replacement?: ImmunizationInput }`;
  - `RndsPendingItem { immunization_id: string; patient_id: string; immunobiological: CodeName; applied_at: string; rnds: ImmunizationRnds }`; `RndsPendingPage { items: RndsPendingItem[]; next_before: string | null }` (D10);
  - `Vial` (= `<vial>`); `MonthlySummaryRow { immunobiological: CodeName; lot: { id: string; number: string }; opening_balance: number; received: number; transferred_in: number; transferred_out: number; vials_opened: number; doses_applied: number; technical_loss: number; physical_loss: Record<string, number>; closing_balance: number }`; `MonthlySummary { month: string; location: LocationRef; rows: MonthlySummaryRow[] }` (D8);
  - `ColdChainEquipment { id; location_id; name; kind: ColdChainEquipmentKind | string; min_c: number; max_c: number; active: boolean }`; `EquipmentInput = Omit<ColdChainEquipment, "id">`; `ReadingInput { read_at: string; current_c: number; min_c: number; max_c: number }`; `ColdChainReading extends ReadingInput { id: string; out_of_range: boolean; by: { name: string | null } | null }`; `ReadingResult { reading: ColdChainReading; excursion_id: string | null; blocked_lots: number }`; `ExcursionLot { lot_id: string; number: string; immunobiological: CodeName; balance: string }`; `ColdChainExcursion { id: string; equipment_id: string; reading: ColdChainReading; lots: ExcursionLot[]; opened_at: string }`; `ColdChainStatus { equipments: (ColdChainEquipment & { last_reading: ColdChainReading | null; readings_today: number })[]; open_excursions: ColdChainExcursion[] }`; `ExcursionDecisionLine { lot_id: string; decision: "use" | "discard"; reason: string }`; `ExcursionDecisionResult { excursion_id: string; adjustments: StockAdjustment[] }` (D9);
  - `VaccineRoomStaff { user_id: string; name: string | null; role: VaccineStaffRole | string; cbo: string | null }`; `VaccineRoom { id; name; active: boolean; health_unit: { id: string; name: string }; stock_location_id: string; staff: VaccineRoomStaff[] }`; `VaccineRoomInput { health_unit_id: string; name: string; active: boolean }`; `VaccineRoomsAdmin { rooms: VaccineRoom[]; cold_chain_central: AdminLocation | null }` (D4); `VaccinationSettings { go_live_on: string | null }`;
  - `VaccineReceiptItemInput { immunobiological_code: string; manufacturer_code: string; lot_number: string; expires_on: string; quantity: string; package?: PackageInput }`; `VaccineReceiptInput { location_id: string; document: { kind: "invoice" | "donation"; number: string; supplier: string }; items: VaccineReceiptItemInput[] }`; `VaccineOpeningInput { location_id: string; items: VaccineReceiptItemInput[] }` (D6);
  - `VaccinationDashboard { doses_by_immunobiological: { immunobiological: CodeName; room: { id: string; name: string }; count: number }[]; c2_coverage_by_team: { team: { id: string; name: string }; eligible: number; complete: number }[]; rnds: { pending: number; blocked: number; rejected: number; late: number }; losses: { technical: number; physical: number }; unchecked_cards: number }` (D13);
  - constante `VACCINATION_PAGE_SIZE = 50`;
  - funções: `listMyVaccineLocations(): Promise<VaccineLocation[]>`, `getVaccinationCatalog(): Promise<VaccinationCatalog>`, `lookupVaccinationPerson(cpf: string): Promise<PharmacyPerson>`, `findVaccinationDuplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>`, `getVaccinationCard(roomId: string, req: CardRequest): Promise<VaccinationCard>`, `confirmCardComplete(roomId: string, patientId: string): Promise<VaccinationCard>`, `recordImmunization(roomId: string, input: ImmunizationInput): Promise<Immunization>`, `recordTranscriptions(roomId: string, input: TranscriptionInput): Promise<{ immunizations: Immunization[]; card: VaccinationCard }>`, `cancelImmunization(id: string, input: CancelInput): Promise<{ cancelled: Immunization; replacement: Immunization | null }>`, `getImmunization(id: string): Promise<Immunization>` (D11), `listRndsPending(roomId: string, before: string | null): Promise<RndsPendingPage>`, `openVial(roomId: string, lotId: string): Promise<Vial>`, `listVials(roomId: string, status: VialStatus | null): Promise<Vial[]>`, `closeOutVial(id: string, reason: string): Promise<Vial>`, `getMonthlySummary(scope: { roomId: string } | { central: true }, month: string): Promise<MonthlySummary>`, `listEquipments(locationId: string): Promise<ColdChainEquipment[]>`, `createEquipment(input: EquipmentInput): Promise<ColdChainEquipment>`, `updateEquipment(id: string, input: EquipmentInput): Promise<ColdChainEquipment>`, `recordReading(equipmentId: string, input: ReadingInput): Promise<ReadingResult>`, `decideExcursion(id: string, lots: ExcursionDecisionLine[]): Promise<ExcursionDecisionResult>`, `getColdChainStatus(locationId: string): Promise<ColdChainStatus>`, `notifyEsavi(immunizationId: string): Promise<{ notification_id: string }>`, `getVaccinationDashboard(q: { month: string; healthUnitId?: string | null }): Promise<VaccinationDashboard>`, `listVaccineRooms(): Promise<VaccineRoomsAdmin>`, `createVaccineRoom(input: VaccineRoomInput): Promise<VaccineRoom>`, `updateVaccineRoom(id: string, input: VaccineRoomInput): Promise<VaccineRoom>`, `addRoomStaff(roomId: string, userId: string, role: VaccineStaffRole): Promise<VaccineRoom>`, `removeRoomStaff(roomId: string, userId: string, role: string): Promise<void>`, `createColdChainCentral(): Promise<AdminLocation>`, `getVaccinationSettings(): Promise<VaccinationSettings>`, `updateVaccinationSettings(s: VaccinationSettings): Promise<VaccinationSettings>`, `setRndsCredential(file: Blob, password: string, requesterCnes: string): Promise<IntegrationCredential>` (D1), `recordVaccineReceipt(input: VaccineReceiptInput): Promise<StockReceipt>`, `recordVaccineOpening(input: VaccineOpeningInput): Promise<StockReceipt>`;
  - `FeatureKey` ganha `vaccination`, `rnds_immunization`, `rnds_mock`.
- Produces (em `src/test/vaccinationFixtures.ts`): `TODAY29`, `NOW29`, `ROOM`, `LEAD_ROOM`, `COLD_CENTRAL`, `PENTA`, `BCG`, `TERMS`, `CATALOG29`, `PENTA_ITEM`, `BCG_ITEM`, `vacUser(roles?, over?)`, `vacPosition(over?)`, `vacLotBalance(over?)`, `cardPatient(over?)`, `cardItem(over?)`, `immunization(over?)`, `card(over?)`, `vial(over?)`, `pendingItem(over?)`, `vaccineRoom(over?)`, `equipment(over?)`, `coldStatus(over?)`, `monthly(over?)`, `vacDashboard(over?)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.vaccination.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  addRoomStaff, cancelImmunization, closeOutVial, confirmCardComplete, createColdChainCentral, createEquipment, createVaccineRoom,
  decideExcursion, findVaccinationDuplicates, getColdChainStatus, getImmunization, getMonthlySummary, getVaccinationCard,
  getVaccinationCatalog, getVaccinationDashboard, getVaccinationSettings, listEquipments, listMyVaccineLocations, listRndsPending,
  listVaccineRooms, listVials, lookupVaccinationPerson, notifyEsavi, openVial, recordImmunization, recordReading, recordTranscriptions,
  recordVaccineOpening, recordVaccineReceipt, removeRoomStaff, setRndsCredential, updateVaccinationSettings, updateVaccineRoom
} from "./api";

function ok(body: unknown, status = 200): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
}
// CPF, nome, vacina, dose, lote, motivo, senha e certificado nunca na URL.
const SENSITIVE = /12345678909|Joana|Penta|BCG|D1|PNT2611|febre alta|desistiu|s3nh4|TUlJ|2001234/i;
const IMM = { patient_id: "pa1", immunobiological_code: "42", dose_code: "D1", strategy: "1", care_group: "1", route: "IM", site: "VLD",
  applied_at: "2026-11-02T10:00:00-03:00", lot_id: "lot9" };

describe("cliente da sala de vacina (contrato §3–§11)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(() => ok({})); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => (fetchMock.mock.calls[i][1] ?? {}) as RequestInit;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));
  const writeOf = (i = 0) => [ urlOf(i).pathname, initOf(i).method, (initOf(i).headers as Record<string, string>)["Content-Type"] ];

  it("locais, catálogo e pessoa: envelopes e corpo", async () => {
    fetchMock.mockImplementationOnce(() => ok({ locations: [ { id: "l5" } ] }));
    expect(await listMyVaccineLocations()).toEqual([ { id: "l5" } ]);
    expect(urlOf(0).pathname).toBe("/vaccination/my_locations");
    await getVaccinationCatalog();
    expect(urlOf(1).pathname).toBe("/vaccination/catalog");
    await lookupVaccinationPerson("12345678909");
    expect([ ...writeOf(2), bodyOf(2) ]).toEqual([ "/vaccination/person_lookup", "POST", "application/json", { cpf: "12345678909" } ]);
    expect(await findVaccinationDuplicates({ full_name: "Joana Lima", birth_date: "2023-05-01" })).toEqual([]);
    expect(writeOf(3)).toEqual([ "/vaccination/patients/duplicates", "POST", "application/json" ]);
  });

  it("caderneta, conferência, dose, transcrição e cancelamento", async () => {
    await getVaccinationCard("r1", { patient_id: "pa1" });
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/vaccination/rooms/r1/card", "POST", "application/json", { patient_id: "pa1" } ]);
    await confirmCardComplete("r1", "pa1");
    expect([ ...writeOf(1), bodyOf(1) ]).toEqual([ "/vaccination/rooms/r1/checks", "POST", "application/json",
      { patient_id: "pa1", mode: "confirmed_complete" } ]);
    await recordImmunization("r1", { ...IMM, vial_id: "v1", override_reason: "atraso por internação longa" });
    expect([ ...writeOf(2), bodyOf(2) ]).toEqual([ "/vaccination/rooms/r1/immunizations", "POST", "application/json",
      { ...IMM, vial_id: "v1", override_reason: "atraso por internação longa" } ]);
    fetchMock.mockImplementationOnce(() => ok({ immunizations: [ { id: "im1" } ], card: { items: [] } }, 201));
    const t = { patient_id: "pa1", items: [ { immunobiological_code: "15", dose_code: "D1", applied_on: "2023-05-02" } ], close_check: true };
    expect(await recordTranscriptions("r1", t)).toEqual({ immunizations: [ { id: "im1" } ], card: { items: [] } });
    expect([ ...writeOf(3), bodyOf(3) ]).toEqual([ "/vaccination/rooms/r1/transcriptions", "POST", "application/json", t ]);
    await cancelImmunization("im1", { reason: "lote errado digitado na sala", replacement: IMM });
    expect([ ...writeOf(4), bodyOf(4) ]).toEqual([ "/vaccination/immunizations/im1/cancel", "POST", "application/json",
      { reason: "lote errado digitado na sala", replacement: IMM } ]);
    await getImmunization("im1");
    expect(urlOf(5).pathname).toBe("/vaccination/immunizations/im1");
  });

  it("pendências e frascos: só ids, status e cursor na URL", async () => {
    fetchMock.mockImplementationOnce(() => ok({ items: [ { immunization_id: "im1" } ], next_before: "c1" }));
    expect(await listRndsPending("r1", "c0")).toEqual({ items: [ { immunization_id: "im1" } ], next_before: "c1" });
    expect([ urlOf(0).pathname, Object.fromEntries(urlOf(0).searchParams) ]).toEqual([ "/vaccination/rooms/r1/pending", { before: "c0", limit: "50" } ]);
    expect(await listRndsPending("r1", null)).toEqual({ items: [], next_before: null });
    await openVial("r1", "lot9");
    expect([ ...writeOf(2), bodyOf(2) ]).toEqual([ "/vaccination/rooms/r1/vials", "POST", "application/json", { lot_id: "lot9" } ]);
    fetchMock.mockImplementationOnce(() => ok({ vials: [ { id: "v1" } ] }));
    expect(await listVials("r1", "open")).toEqual([ { id: "v1" } ]);
    expect([ urlOf(3).pathname, Object.fromEntries(urlOf(3).searchParams) ]).toEqual([ "/vaccination/rooms/r1/vials", { status: "open" } ]);
    await closeOutVial("v1", "frasco caiu e trincou");
    expect([ ...writeOf(4), bodyOf(4) ]).toEqual([ "/vaccination/vials/v1/close_out", "POST", "application/json", { reason: "frasco caiu e trincou" } ]);
  });

  it("resumo mensal da sala e da rede de frio", async () => {
    await getMonthlySummary({ roomId: "r1" }, "2026-10");
    expect([ urlOf(0).pathname, Object.fromEntries(urlOf(0).searchParams) ]).toEqual([ "/vaccination/rooms/r1/monthly_summary", { month: "2026-10" } ]);
    await getMonthlySummary({ central: true }, "2026-10");
    expect(urlOf(1).pathname).toBe("/vaccination/cold_chain_central/monthly_summary");
  });

  it("cadeia de frio: equipamentos, leitura, conduta e situação", async () => {
    fetchMock.mockImplementationOnce(() => ok({ equipments: [ { id: "eq1" } ] }));
    expect(await listEquipments("l5")).toEqual([ { id: "eq1" } ]);
    expect(Object.fromEntries(urlOf(0).searchParams)).toEqual({ location_id: "l5" });
    const eq = { location_id: "l5", name: "Geladeira 1", kind: "refrigerator", min_c: 2, max_c: 8, active: true };
    await createEquipment(eq);
    expect([ ...writeOf(1), bodyOf(1) ]).toEqual([ "/vaccination/cold_chain/equipments", "POST", "application/json", eq ]);
    const r = { read_at: "2026-11-02T08:00:00-03:00", current_c: 8.5, min_c: 3, max_c: 9 };
    await recordReading("eq1", r);
    expect([ ...writeOf(2), bodyOf(2) ]).toEqual([ "/vaccination/cold_chain/equipments/eq1/readings", "POST", "application/json", r ]);
    const lots = [ { lot_id: "lot9", decision: "discard" as const, reason: "febre alta do equipamento por 6 horas" } ];
    await decideExcursion("ex1", lots);
    expect([ ...writeOf(3), bodyOf(3) ]).toEqual([ "/vaccination/cold_chain/excursions/ex1/decide", "POST", "application/json", { lots } ]);
    await getColdChainStatus("l5");
    expect([ urlOf(4).pathname, Object.fromEntries(urlOf(4).searchParams) ]).toEqual([ "/vaccination/cold_chain/status", { location_id: "l5" } ]);
  });

  it("ESAVI e painel", async () => {
    await notifyEsavi("im1");
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/vaccination/immunizations/im1/esavi", "POST", "application/json", {} ]);
    await getVaccinationDashboard({ month: "2026-10", healthUnitId: "u1" });
    expect([ urlOf(1).pathname, Object.fromEntries(urlOf(1).searchParams) ]).toEqual([ "/vaccination/dashboard", { month: "2026-10", health_unit_id: "u1" } ]);
    await getVaccinationDashboard({ month: "2026-10", healthUnitId: null });
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ month: "2026-10" });
  });

  it("administração: salas, equipe, rede de frio e go-live", async () => {
    fetchMock.mockImplementationOnce(() => ok({ rooms: [ { id: "r1" } ] }));
    expect(await listVaccineRooms()).toEqual({ rooms: [ { id: "r1" } ], cold_chain_central: null });
    await createVaccineRoom({ health_unit_id: "u1", name: "Sala UBS Centro", active: true });
    expect([ ...writeOf(1), bodyOf(1) ]).toEqual([ "/vaccination/admin/rooms", "POST", "application/json", { health_unit_id: "u1", name: "Sala UBS Centro", active: true } ]);
    await updateVaccineRoom("r1", { health_unit_id: "u1", name: "Sala UBS Centro", active: false });
    expect(writeOf(2)).toEqual([ "/vaccination/admin/rooms/r1", "PATCH", "application/json" ]);
    await addRoomStaff("r1", "us7", "room_lead");
    expect([ ...writeOf(3), bodyOf(3) ]).toEqual([ "/vaccination/admin/rooms/r1/staff", "POST", "application/json", { user_id: "us7", role: "room_lead" } ]);
    fetchMock.mockImplementationOnce(() => Promise.resolve(new Response(null, { status: 204 })));
    await removeRoomStaff("r1", "us7", "room_lead");
    expect([ ...writeOf(4), bodyOf(4) ]).toEqual([ "/vaccination/admin/rooms/r1/staff", "DELETE", "application/json", { user_id: "us7", role: "room_lead" } ]);
    await createColdChainCentral();
    expect([ ...writeOf(5), bodyOf(5) ]).toEqual([ "/vaccination/admin/cold_chain_central", "POST", "application/json", {} ]);
    await getVaccinationSettings();
    await updateVaccinationSettings({ go_live_on: "2026-12-01" });
    expect([ ...writeOf(7), bodyOf(7) ]).toEqual([ "/vaccination/admin/settings", "PATCH", "application/json", { go_live_on: "2026-12-01" } ]);
  });

  it("credencial RNDS: arquivo em base64, senha e CNES só no corpo do PUT (D1)", async () => {
    await setRndsCredential(new Blob([ "MII" ], { type: "application/x-pkcs12" }), "s3nh4", "2001234");
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/integrations/credentials/rnds", "PUT", "application/json",
      { pfx_base64: "TUlJ", pfx_password: "s3nh4", requester_cnes: "2001234" } ]);
  });

  it("entrada e inventário inicial de imunobiológico pelas rotas do 27a (D6)", async () => {
    const item = { immunobiological_code: "42", manufacturer_code: "SER", lot_number: "PNT2611", expires_on: "2027-03-31", quantity: "100" };
    await recordVaccineReceipt({ location_id: "l9", document: { kind: "invoice", number: "NF 88", supplier: "SESA-PR" }, items: [ item ] });
    expect([ ...writeOf(0), bodyOf(0).items ]).toEqual([ "/pharmacy/receipts", "POST", "application/json", [ item ] ]);
    await recordVaccineOpening({ location_id: "l9", items: [ item ] });
    expect(writeOf(1)).toEqual([ "/pharmacy/openings", "POST", "application/json" ]);
  });

  it("nada pessoal ou clínico na URL", async () => {
    await lookupVaccinationPerson("12345678909");
    await getVaccinationCard("r1", { new_patient: { full_name: "Joana Lima", birth_date: "2023-05-01", sex: "female" } });
    await recordImmunization("r1", { ...IMM, override_reason: "febre alta na última dose" });
    await cancelImmunization("im1", { reason: "paciente desistiu, dose registrada por engano" });
    await setRndsCredential(new Blob([ "MII" ]), "s3nh4", "2001234");
    for (const call of fetchMock.mock.calls) expect(decodeURIComponent(String(call[0]))).not.toMatch(SENSITIVE);
  });
});
```

E no fim de `src/lib/features.test.ts`:

```ts
describe("módulo 29", () => {
  it("as três chaves têm nome em português", () => {
    expect(featureLabel("vaccination")).toBe("Sala de vacina");
    expect(featureLabel("rnds_immunization")).toBe("Envio das vacinas à RNDS");
    expect(featureLabel("rnds_mock")).toBe("RNDS simulada (fora de produção)");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/api.vaccination.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções novas e `src/test/vaccinationFixtures.ts` não existem; `featureLabel("vaccination")` devolve a chave.

- [ ] **Step 3: Implemente os tipos e o cliente**

Em `src/lib/api.ts`: os campos novos de `StockItem`, `StockLot`, `LotBalance`, `TimelineKind`, `Criterion`, `CampaignOptions`, `Campaign` e `CredentialKind` onde esses tipos estão (comentário `// Módulo 29 (contrato §2; Divergência Dn)`); e a seção `// ─── Sala de vacina (módulo 29; ADR 0047; contrato 2026-11-02 §2–§11) ───` no fim, com `const VACCINATION_BASE = import.meta.env.VITE_VACCINATION_BASE || "/vaccination"`, `const vacPath = (...parts: string[]) => [ VACCINATION_BASE, ...parts.map(encodeURIComponent) ].join("/")` e `const vacWrite = (method: "POST" | "PATCH" | "PUT" | "DELETE", body: unknown = {}): RequestInit => ({ method, body: JSON.stringify(body) })` (sempre com corpo). Envelopes: `locations`, `equipments`, `vials`, `items`/`next_before`, `rooms`/`cold_chain_central` — ausente vira `[]`/`null`. `limit=50` (`VACCINATION_PAGE_SIZE`) só em `listRndsPending`. Parâmetro nulo ou vazio não entra na URL. `setRndsCredential` usa o `readBase64` existente e manda `PUT ${INTEGRATIONS_BASE}/credentials/rnds` (D1); `recordVaccineReceipt`/`recordVaccineOpening` usam `phPath("receipts")`/`phPath("openings")` com `phWrite("POST", input)`. A mudança de `StockItem.catalog_item_id`/`catmat_code` para anuláveis quebra o `tsc` nos lugares que supõem medicamento: nos fluxos só de medicamento (dispensação, mínimos, receita externa, `receiptItems`), use `?? ""` com o comentário `// só medicamento: local de vacina nunca chega aqui`; a identidade de item nas peças reusadas vem da Task 2 (`itemKey`).

Em `src/lib/features.ts`: `| "vaccination" | "rnds_immunization" | "rnds_mock"` em `FeatureKey` e, em `FEATURE_LABEL`, `// Módulo 29 (ADR 0047): vaccination exige record_mode:record; o envio exige a credencial rnds.` com `vaccination: "Sala de vacina"`, `rnds_immunization: "Envio das vacinas à RNDS"`, `rnds_mock: "RNDS simulada (fora de produção)"`. Em `vite.config.ts`, depois da farmácia: `// Módulo 29: sala de vacina. Regex como os vizinhos.` e `"^/vaccination(/|$)": proxy(TARGET)`.

Em `src/test/vaccinationFixtures.ts` (códigos ilustrativos; os reais vêm do catálogo do api):

```ts
// Dados dos testes do módulo 29 (sala de vacina). Hoje: segunda, 2026-11-02, 10h. Formas do contrato §2 e das Divergências.
import type {
  CardItem, CardPatient, CodeName, ColdChainEquipment, ColdChainStatus, Immunization, Immunobiological, LotBalance, MonthlySummary,
  RndsPendingItem, SessionUser, StockItem, StockPosition, VaccinationCard, VaccinationCatalog, VaccinationDashboard, VaccineLocation,
  VaccineRoom, Vial
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";
import { lotBalance, position, stockLot } from "./pharmacyFixtures";

export const TODAY29 = "2026-11-02";
export const NOW29 = new Date("2026-11-02T10:00:00-03:00");
export const ROOM: VaccineLocation = { id: "l5", name: "Sala de vacina UBS Centro", kind: "vaccine_room",
  health_unit: { id: "u1", name: "UBS Centro" }, active: true, roles: [ "vaccinator" ], room_id: "r1" };
export const LEAD_ROOM: VaccineLocation = { ...ROOM, roles: [ "room_lead" ] };
export const COLD_CENTRAL: VaccineLocation = { id: "l9", name: "Rede de frio municipal", kind: "cold_chain_central", health_unit: null,
  active: true, roles: [ "stock_keeper" ], room_id: null };
export const PENTA: Immunobiological = { code: "42", name: "Penta (DTP/HepB/Hib)", multidose: false, doses_per_vial: 1, default_route: "IM",
  default_site: "VLD", manufacturers: [ { code: "SER", name: "Serum Institute", validity_after_opening_hours: null } ] };
export const BCG: Immunobiological = { code: "15", name: "BCG", multidose: true, doses_per_vial: 10, default_route: "ID", default_site: "DD",
  manufacturers: [ { code: "FAP", name: "Fundação Ataulpho de Paiva", validity_after_opening_hours: 6 } ] };
const t = (pairs: [ string, string ][]): CodeName[] => pairs.map(([ code, name ]) => ({ code, name }));
export const TERMS = {
  dose: t([ [ "D1", "1ª dose" ], [ "D2", "2ª dose" ], [ "D3", "3ª dose" ], [ "REF", "Reforço" ] ]),
  strategy: t([ [ "1", "Rotina" ], [ "2", "Campanha" ] ]),
  route: t([ [ "IM", "Intramuscular" ], [ "ID", "Intradérmica" ], [ "VO", "Oral" ] ]),
  site: t([ [ "VLD", "Vasto lateral da coxa direita" ], [ "DD", "Deltoide direito" ] ]),
  care_group: t([ [ "1", "Criança" ], [ "2", "Gestante" ] ]),
  maternal_condition: t([ [ "G", "Gestante" ], [ "P", "Puérpera" ] ])
};
export const CATALOG29: VaccinationCatalog = { calendar_version: "2026.1", immunobiologicals: [ PENTA, BCG ], terminologies: TERMS };
const vacItem = (code: string, name: string): StockItem => ({ catalog_item_id: null, catmat_code: null, immunobiological_code: code,
  item_kind: "immunobiological", name, unit: "dose", dispensing_unit: "dose", controlled: false, controlled_list: null });
export const PENTA_ITEM = vacItem("42", "Penta (DTP/HepB/Hib)");
export const BCG_ITEM = vacItem("15", "BCG");

export const vacUser = (roles = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser =>
  sessionWith(roles, { id: "us1", features: [ "clinical_record", "vaccination" ], ...over });
export const vacLotBalance = (over: Partial<LotBalance> = {}): LotBalance => lotBalance({
  lot: stockLot({ id: "lot9", item: PENTA_ITEM, lot_number: "PNT2611", expires_on: "2027-03-31", manufacturer: "Serum Institute", manufacturer_code: "SER" }),
  balance: "20", in_transit: "0", expiring_soon: false, expired: false, blocked: false, ...over });
export const vacPosition = (over: Partial<StockPosition> = {}): StockPosition =>
  position({ location: { id: ROOM.id, name: ROOM.name, kind: ROOM.kind }, item: PENTA_ITEM, lots: [ vacLotBalance() ], balance: "20",
    minimum: null, below_minimum: false, expiring_soon: false, ...over });
export const cardPatient = (over: Partial<CardPatient> = {}): CardPatient =>
  ({ id: "pa1", display_name: "Joana Lima", birth_date: "2026-07-01", sex: "female", has_cpf: false, cns_masked: "*** **** **** 1234", ...over });
export const cardItem = (over: Partial<CardItem> = {}): CardItem =>
  ({ immunobiological: { code: "42", name: PENTA.name }, dose: { code: "D2", name: "2ª dose" }, status: "due", due_on: TODAY29, applied: null, ...over });
export function immunization(over: Partial<Immunization> = {}): Immunization {
  return { id: "im1", immunobiological: { code: "42", name: PENTA.name }, dose: { code: "D1", name: "1ª dose" }, strategy: "1", care_group: "1",
    route: "IM", site: "VLD", maternal_condition: null, applied_at: "2026-09-01T09:30:00-03:00",
    lot: { id: "lot9", number: "PNT2611", manufacturer: { code: "SER", name: "Serum Institute" }, expires_on: "2027-03-31" }, transcription: false,
    room: { id: "r1", name: ROOM.name, health_unit: { id: "u1", name: "UBS Centro" } }, by: { name: "Téc. Ana Souza", cbo: "322205" },
    campaign_id: null, status: "active", cancel_reason: null, replaces_id: null, replaced_by_id: null,
    rnds: { status: "accepted", sent_at: "2026-09-01T09:31:00-03:00", accepted_at: "2026-09-01T09:31:05-03:00", block_reason: null, error: null },
    can_cancel: true, ...over };
}
export function card(over: Partial<VaccinationCard> = {}): VaccinationCard {
  return { patient: cardPatient(), checked: { mode: "born_after_go_live", at: "2026-07-01T00:00:00-03:00", by_name: null }, calendar_version: "2026.1",
    items: [ cardItem({ dose: { code: "D1", name: "1ª dose" }, status: "complete", due_on: "2026-09-01", applied: { id: "im1", applied_at: "2026-09-01T09:30:00-03:00", transcription: false } }),
      cardItem(), cardItem({ dose: { code: "D3", name: "3ª dose" }, status: "upcoming", due_on: "2027-01-01" }) ],
    immunizations: [ immunization() ], ...over };
}
export const vial = (over: Partial<Vial> = {}): Vial => ({ id: "v1", lot: { id: "lot15", number: "BCG2610", immunobiological_code: "15" },
  opened_at: "2026-11-02T08:00:00-03:00", expires_at: "2026-11-02T14:00:00-03:00", doses_per_vial: 10, doses_used: 3, doses_left: 7, status: "open", ...over });
export const pendingItem = (over: Partial<RndsPendingItem> = {}): RndsPendingItem => ({ immunization_id: "im2", patient_id: "pa1",
  immunobiological: { code: "42", name: PENTA.name }, applied_at: "2026-11-01T09:00:00-03:00",
  rnds: { status: "blocked", sent_at: null, accepted_at: null, block_reason: "professional_cns_not_in_cnes", error: null }, ...over });
export const vaccineRoom = (over: Partial<VaccineRoom> = {}): VaccineRoom => ({ id: "r1", name: ROOM.name, active: true,
  health_unit: { id: "u1", name: "UBS Centro" }, stock_location_id: "l5",
  staff: [ { user_id: "us1", name: "Téc. Ana Souza", role: "vaccinator", cbo: "322205" } ], ...over });
export const equipment = (over: Partial<ColdChainEquipment> = {}): ColdChainEquipment =>
  ({ id: "eq1", location_id: "l5", name: "Geladeira 1", kind: "refrigerator", min_c: 2, max_c: 8, active: true, ...over });
export const coldStatus = (over: Partial<ColdChainStatus> = {}): ColdChainStatus => ({ equipments: [ { ...equipment(), last_reading: null, readings_today: 0 } ],
  open_excursions: [], ...over });
export const monthly = (over: Partial<MonthlySummary> = {}): MonthlySummary => ({ month: "2026-10", location: { id: ROOM.id, name: ROOM.name, kind: ROOM.kind },
  rows: [ { immunobiological: { code: "15", name: "BCG" }, lot: { id: "lot15", number: "BCG2610" }, opening_balance: 40, received: 20, transferred_in: 0,
    transferred_out: 0, vials_opened: 3, doses_applied: 24, technical_loss: 6, physical_loss: { broken: 10 }, closing_balance: 20 } ], ...over });
export const vacDashboard = (over: Partial<VaccinationDashboard> = {}): VaccinationDashboard => ({
  doses_by_immunobiological: [ { immunobiological: { code: "42", name: PENTA.name }, room: { id: "r1", name: ROOM.name }, count: 31 } ],
  c2_coverage_by_team: [ { team: { id: "t1", name: "eSF Centro 1" }, eligible: 40, complete: 30 } ],
  rnds: { pending: 2, blocked: 1, rejected: 0, late: 1 }, losses: { technical: 6, physical: 10 }, unchecked_cards: 12, ...over });
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/api.vaccination.test.ts src/lib/features.test.ts src/lib/api.pharmacy.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS (o `api.pharmacy.test.ts` existente confere que o cliente da farmácia não mudou).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.vaccination.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/vaccinationFixtures.ts
$G add <arquivos do 27a que o tsc pediu para ajustar, um por um>
$G commit -m "feat: add the vaccination room api client, feature keys and fixtures

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/vaccination.ts`)

**Files:**
- Create: `src/lib/vaccination.ts`
- Modify: `src/lib/pharmacy.ts` (`itemKey`; rótulos de `vaccine_room`/`cold_chain_central` em `KIND_LABEL`)
- Test: `src/lib/vaccination.test.ts`

**Interfaces:**
- Consumes: os tipos da Task 1; `ApiError`, `errorCode` (`api.ts`); `hasFeature`, `featureDisabledKey` (`features.ts`); `NOMINAL_QUERY`, `STATIC_QUERY` (`careLines.ts`, reexportados); `parseCityLocal`, `cityIsoDate`, `fmtDateTime` (`format.ts`); `fmtDay` (`exams.ts`); `Problem`, `Refusal`, `pharmacyError` (`pharmacy.ts`).
- Produces (em `src/lib/pharmacy.ts`): `itemKey(item: StockItem): string` (= `catalog_item_id` ou `imm:<immunobiological_code>`); `locationKindLabel("vaccine_room") === "Sala de vacina"`, `locationKindLabel("cold_chain_central") === "Rede de frio municipal"`.
- Produces (em `src/lib/vaccination.ts`):
  - frases: `VACCINATION_DISABLED = "a sala de vacina está desligada nesta cidade"`, `VACCINATION_NO_ROLE = "seu papel não permite usar a sala de vacina"`, `NO_VACCINE_LOCATION = "você não está na equipe de nenhuma sala de vacina nem da rede de frio — peça ao administrador da cidade"`, `UNCHECKED_NOTICE = "Caderneta não conferida: transcreva as doses de fora ou confirme que está completa para ver as vacinas pendentes."` (sem a palavra "atraso": a tela inteira não a usa enquanto não há conferência), `DOSE_SAVED = "Dose registrada."`, `LOT_REREAD = "o lote mudou — a lista de lotes foi relida; escolha de novo"`, `VIAL_REREAD = "o frasco mudou — a lista de frascos foi relida; abra outro frasco ou escolha outro"`;
  - chaves: `VAC_LOCATIONS_KEY`, `VAC_CATALOG_KEY`, `VAC_PENDING_KEY`, `VAC_VIALS_KEY`, `VAC_SUMMARY_KEY`, `VAC_COLD_KEY`, `VAC_EQUIPMENTS_KEY`, `VAC_DASHBOARD_KEY`, `VAC_ADMIN_KEY`, `VAC_SETTINGS_KEY`; reexporta `NOMINAL_QUERY`, `STATIC_QUERY`;
  - portão e papéis: `canUseVaccination(user): boolean` (interruptor e nunca o operador); `vaccinationRoles(user): { professional: boolean; keeper: boolean; admin: boolean; analyst: boolean }`; `locationAbilities(loc: VaccineLocation): { vaccinate: boolean; lead: boolean; keeper: boolean }` (`vaccinate` = `vaccinator` ou `room_lead`);
  - rótulos (código novo cru): `cardStatusLabel`, `cardStatusTone`, `rndsStatusLabel`, `rndsStatusTone`, `blockReasonLabel`, `checkModeLabel`, `vialStatusLabel`, `equipmentKindLabel`, `termName(list: CodeName[], code: string | null): string`;
  - `effectiveStatus(item: CardItem, checked: CardCheck | null): string`; `cardGroups(card: VaccinationCard): { today: CardItem[]; overdue: CardItem[]; upcoming: CardItem[]; done: CardItem[]; unchecked: CardItem[] }` (`today` = `due`; `done` = `complete`; `not_applicable` some);
  - dose: `DoseDraft { immunobiologicalCode; doseCode; strategy; careGroup; route; site; maternalCondition; appliedAt; lotId; vialId; campaignId; overrideReason }` (strings); `newDoseDraft(imm: Immunobiological | null, doseCode: string, now: Date): DoseDraft` (via e local padrão do catálogo; `appliedAt` = agora em `datetime-local`); `lotsForDose(positions: StockPosition[], code: string, today: string): LotBalance[]` (saldo > 0, não vencido, não bloqueado, do imunobiológico; ordem por validade); `vialsForDose(vials: Vial[], lotIds: string[], now: Date): Vial[]` (abertos, não vencidos, com sobra); `doseProblems(d: DoseDraft, ctx: { catalog: VaccinationCatalog; lots: LotBalance[]; vials: Vial[]; now: Date }): Problem[]`; `doseBody(d: DoseDraft, patientId: string): ImmunizationInput`;
  - avisos: `OVERRIDABLE = [ "interval_too_short", "age_out_of_range" ]`; `splitRefusals(list: Refusal[]): { warnings: Refusal[]; blocking: Refusal[] }`; `reasonProblem(text: string, label: string): string | null` (10–500);
  - transcrição: `TranscriptionLine { key; immunobiologicalCode; doseCode; appliedOn; lotNumber; manufacturerName; place }`; `newTranscriptionLine(key: string): TranscriptionLine`; `transcriptionProblems(lines: TranscriptionLine[], ctx: { today: string; birthDate: string; closeCheck: boolean }): Problem[]`; `transcriptionBody(lines, patientId, closeCheck): TranscriptionInput`;
  - temperatura: `parseCelsius(raw: string): number | null`; `ReadingDraft { readAt: string; current: string; min: string; max: string }`; `readingProblems(d: ReadingDraft, now: Date): Problem[]`; `readingBody(d: ReadingDraft): ReadingInput`; `outOfRange(r: ReadingInput, eq: Pick<ColdChainEquipment, "min_c" | "max_c">): boolean`; `fmtCelsius(n: number): string` ("8,5 °C");
  - frasco e mês: `vialRemaining(v: Vial, now: Date): string`; `monthProblem(m: string, today: string): string | null`; `currentMonth(today: string): string`;
  - recusas: `vaccinationError(err: unknown): string`; `vaccinationRefusals(err: unknown): Refusal[]` (todas as de `problems[]`, traduzidas; sem `problems`, a do topo) e o mapa de frases do §5–§8 (abaixo).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/vaccination.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import { itemKey, locationKindLabel } from "./pharmacy";
import {
  OVERRIDABLE, canUseVaccination, cardGroups, doseBody, doseProblems, effectiveStatus, locationAbilities, lotsForDose, monthProblem, newDoseDraft,
  newTranscriptionLine, outOfRange, parseCelsius, readingBody, readingProblems, reasonProblem, rndsStatusLabel, blockReasonLabel, splitRefusals,
  termName, transcriptionProblems, vaccinationRefusals, vialRemaining, vialsForDose
} from "./vaccination";
import {
  BCG, CATALOG29, COLD_CENTRAL, LEAD_ROOM, NOW29, PENTA, PENTA_ITEM, ROOM, TERMS, TODAY29, card, cardItem, vacLotBalance, vacPosition, vacUser, vial
} from "../test/vaccinationFixtures";
import { AMOXI } from "../test/pharmacyFixtures";

const refusal = (problems: { path: string; error: string }[]) => new ApiError(422, { error: problems[0].error, problems }, "422");

describe("portão, papéis e rótulos", () => {
  it("interruptor e operador", () => {
    expect(canUseVaccination(vacUser())).toBe(true);
    expect(canUseVaccination(vacUser([ "health_professional" ], { features: [ "clinical_record" ] }))).toBe(false);
    expect(canUseVaccination(vacUser([], { operator: true }))).toBe(false);
  });
  it("papel por local", () => {
    expect(locationAbilities(ROOM)).toEqual({ vaccinate: true, lead: false, keeper: false });
    expect(locationAbilities(LEAD_ROOM)).toEqual({ vaccinate: true, lead: true, keeper: false });
    expect(locationAbilities(COLD_CENTRAL)).toEqual({ vaccinate: false, lead: false, keeper: true });
  });
  it("rótulos e código cru", () => {
    expect(locationKindLabel("vaccine_room")).toBe("Sala de vacina");
    expect(locationKindLabel("cold_chain_central")).toBe("Rede de frio municipal");
    expect(rndsStatusLabel("accepted")).toBe("enviada ao Ministério");
    expect(rndsStatusLabel("not_sent")).toBe("envio desligado — na fila");
    expect(rndsStatusLabel("nova_situacao")).toBe("nova_situacao");
    expect(blockReasonLabel("professional_cns_not_in_cnes")).toBe("o CNS de quem vacinou não está no CNES importado da cidade");
    expect(termName(TERMS.route, "IM")).toBe("Intramuscular");
    expect(termName(TERMS.route, "XX")).toBe("XX");
    expect(itemKey(PENTA_ITEM)).toBe("imm:42");
    expect(itemKey(AMOXI)).toBe("ci1");
  });
});

describe("caderneta", () => {
  it("effectiveStatus: overdue sem conferência vira unchecked", () => {
    expect(effectiveStatus(cardItem({ status: "overdue" }), null)).toBe("unchecked");
    expect(effectiveStatus(cardItem({ status: "overdue" }), { mode: "confirmed_complete", at: "2026-10-01T10:00:00-03:00", by_name: "Ana" })).toBe("overdue");
    expect(effectiveStatus(cardItem({ status: "due" }), null)).toBe("due");
  });
  it("grupos: hoje, atraso, próximas, feitas; não se aplica some", () => {
    const g = cardGroups(card({ items: [ cardItem(), cardItem({ status: "overdue" }), cardItem({ status: "upcoming" }), cardItem({ status: "complete" }),
      cardItem({ status: "not_applicable" }) ] }));
    expect([ g.today.length, g.overdue.length, g.upcoming.length, g.done.length, g.unchecked.length ]).toEqual([ 1, 1, 1, 1, 0 ]);
    expect(cardGroups(card({ checked: null, items: [ cardItem({ status: "overdue" }) ] })).overdue).toEqual([]);
  });
});

describe("dose", () => {
  const lots = [ vacLotBalance() ];
  const ctx = { catalog: CATALOG29, lots, vials: [], now: NOW29 };
  const ok = { ...newDoseDraft(PENTA, "D2", NOW29), strategy: "1", careGroup: "1", lotId: "lot9" };

  it("rascunho com via e local padrão e hora de agora", () => {
    expect(newDoseDraft(PENTA, "D2", NOW29)).toMatchObject({ immunobiologicalCode: "42", doseCode: "D2", route: "IM", site: "VLD", appliedAt: "2026-11-02T10:00" });
  });
  it("lotes oferecidos: só da vacina, com saldo, válido e não bloqueado", () => {
    const pos = [ vacPosition({ lots: [ vacLotBalance(), vacLotBalance({ lot: { ...vacLotBalance().lot, id: "b" }, blocked: true }),
      vacLotBalance({ lot: { ...vacLotBalance().lot, id: "e", expires_on: "2026-11-01" }, expired: true }), vacLotBalance({ lot: { ...vacLotBalance().lot, id: "z" }, balance: "0" }) ] }) ];
    expect(lotsForDose(pos, "42", TODAY29).map((b) => b.lot.id)).toEqual([ "lot9" ]);
    expect(lotsForDose(pos, "15", TODAY29)).toEqual([]);
  });
  it("frascos oferecidos: abertos, do lote, não vencidos e com sobra", () => {
    const list = [ vial(), vial({ id: "v2", expires_at: "2026-11-02T09:59:00-03:00" }), vial({ id: "v3", doses_left: 0 }), vial({ id: "v4", lot: { ...vial().lot, id: "x" } }) ];
    expect(vialsForDose(list, [ "lot15" ], NOW29).map((v) => v.id)).toEqual([ "v1" ]);
  });
  it("recusas de uma vez; o que não recusa", () => {
    expect(doseProblems(ok, ctx)).toEqual([]);
    const bad = doseProblems({ ...ok, strategy: "", lotId: "", appliedAt: "2026-11-02T10:01" }, ctx).map((p) => p.path);
    expect(bad).toEqual([ "strategy", "applied_at", "lot_id" ]);
    const bcg = { ...newDoseDraft(BCG, "D1", NOW29), strategy: "1", careGroup: "1", lotId: "lot15" };
    const bcgLots = [ vacLotBalance({ lot: { ...vacLotBalance().lot, id: "lot15" } }) ];
    expect(doseProblems(bcg, { ...ctx, lots: bcgLots }).map((p) => p.text)).toContain("BCG é multidose: escolha o frasco aberto ou abra um frasco");
    expect(doseProblems({ ...bcg, vialId: "v2" }, { ...ctx, lots: bcgLots, vials: [ vial({ id: "v2", expires_at: "2026-11-02T09:00:00-03:00" }) ] })
      .map((p) => p.path)).toEqual([ "vial_id" ]);
  });
  it("corpo: sem opcionais vazios; motivo só quando há", () => {
    expect(doseBody(ok, "pa1")).toEqual({ patient_id: "pa1", immunobiological_code: "42", dose_code: "D2", strategy: "1", care_group: "1", route: "IM",
      site: "VLD", applied_at: "2026-11-02T13:00:00.000Z", lot_id: "lot9" });
    expect(doseBody({ ...ok, overrideReason: "  intervalo menor por viagem  ", maternalCondition: "G" }, "pa1"))
      .toMatchObject({ override_reason: "intervalo menor por viagem", maternal_condition: "G" });
  });
  it("avisos e recusas duras separados", () => {
    const list = vaccinationRefusals(refusal([ { path: "applied_at", error: "interval_too_short" }, { path: "vial_id", error: "vial_expired" } ]));
    expect(OVERRIDABLE).toEqual([ "interval_too_short", "age_out_of_range" ]);
    const { warnings, blocking } = splitRefusals(list);
    expect(warnings.map((r) => r.code)).toEqual([ "interval_too_short" ]);
    expect(blocking.map((r) => r.text)).toEqual([ "o frasco venceu — descarte e abra outro" ]);
    expect(reasonProblem("curto", "o motivo")).toBe("o motivo precisa ter de 10 a 500 caracteres");
    expect(reasonProblem("aplicado antes por viagem", "o motivo")).toBeNull();
  });
});

describe("transcrição", () => {
  const line = { ...newTranscriptionLine("k1"), immunobiologicalCode: "15", doseCode: "D1", appliedOn: "2026-07-02" };
  it("data entre o nascimento e hoje; tamanhos; ao menos uma linha", () => {
    const ctx = { today: TODAY29, birthDate: "2026-07-01", closeCheck: true };
    expect(transcriptionProblems([ line ], ctx)).toEqual([]);
    expect(transcriptionProblems([ { ...line, appliedOn: "2026-11-03" } ], ctx).map((p) => p.text)).toEqual([ "BCG: a data não pode ser futura" ]);
    expect(transcriptionProblems([ { ...line, appliedOn: "2026-06-30" } ], ctx).map((p) => p.text)).toEqual([ "BCG: a data é antes do nascimento" ]);
    expect(transcriptionProblems([ { ...line, lotNumber: "x".repeat(41) } ], ctx).map((p) => p.text)).toEqual([ "BCG: o lote tem até 40 caracteres" ]);
    expect(transcriptionProblems([], ctx).map((p) => p.text)).toEqual([ "acrescente ao menos uma dose ou use “Confirmar caderneta completa”" ]);
  });
});

describe("temperatura", () => {
  it("parseCelsius e readingProblems", () => {
    expect([ parseCelsius("8,5"), parseCelsius("-1"), parseCelsius(" 4 "), parseCelsius("8,55"), parseCelsius("abc"), parseCelsius("51") ])
      .toEqual([ 8.5, -1, 4, null, null, null ]);
    const d = { readAt: "2026-11-02T08:00", current: "8,5", min: "3", max: "9" };
    expect(readingProblems(d, NOW29)).toEqual([]);
    expect(readingBody(d)).toEqual({ read_at: "2026-11-02T11:00:00.000Z", current_c: 8.5, min_c: 3, max_c: 9 });
    expect(readingProblems({ ...d, min: "9", max: "10" }, NOW29).map((p) => p.text)).toEqual([ "a mínima não pode ser maior que a atual" ]);
    expect(readingProblems({ ...d, readAt: "2026-11-02T10:30" }, NOW29).map((p) => p.text)).toEqual([ "a hora da leitura não pode ser futura" ]);
    expect(outOfRange(readingBody(d), { min_c: 2, max_c: 8 })).toBe(true);
    expect(outOfRange({ ...readingBody(d), current_c: 5, max_c: 8 }, { min_c: 2, max_c: 8 })).toBe(false);
  });
  it("frasco e mês", () => {
    expect(vialRemaining(vial(), NOW29)).toBe("vence em 4 h 0 min");
    expect(vialRemaining(vial({ expires_at: "2026-11-02T09:00:00-03:00" }), NOW29)).toBe("vencido");
    expect([ monthProblem("2026-11", TODAY29), monthProblem("2026-12", TODAY29), monthProblem("2026-13", TODAY29) ])
      .toEqual([ null, "o mês não pode ser futuro", "informe o mês (AAAA-MM)" ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/vaccination.test.ts`
Expected: FAIL — `src/lib/vaccination.ts` não existe; `itemKey` não existe.

- [ ] **Step 3: Implemente `src/lib/vaccination.ts` e `itemKey`**

Cabeçalho: `// Regras de tela do módulo 29 (sala de vacina; contrato 2026-11-02 §1–§11) sem React. Calendário, atraso, aprazamento, saldo, validade do frasco, envio à RNDS e trilha são do api; a tela só não envia o que já sabe que será recusado.` As funções seguem as assinaturas do Interfaces; os valores fixos:

- rótulos: `rnds_status` — `not_sent: "envio desligado — na fila"`, `pending: "na fila de envio"`, `sending: "enviando"`, `accepted: "enviada ao Ministério"`, `rejected: "recusada pela RNDS"`, `failed: "falha no envio"`, `blocked: "parada: cadastro a corrigir"` (tons: `accepted` ok; `pending`/`sending`/`not_sent` info; `rejected`/`failed`/`blocked` down); motivo de bloqueio — `professional_cns_not_in_cnes: "o CNS de quem vacinou não está no CNES importado da cidade"`, `patient_identity_unverified: "o CPF/CNS do cidadão não foi conferido no CADSUS"`, `patient_without_cns_or_cpf: "o cidadão não tem CPF nem CNS no cadastro"`; situação — `due: "devida hoje"`, `overdue: "em atraso"`, `upcoming: "próxima"`, `complete: "aplicada"`, `not_applicable: "não se aplica"`, `unchecked: "não conferida"` (tons: `due` info, `overdue` down, `upcoming` neutral, `complete` ok, `unchecked` warn); conferência — `transcribed: "conferida por transcrição"`, `confirmed_complete: "conferida: caderneta completa"`, `born_after_go_live: "nasceu depois do início — conferida desde o nascimento"`; frasco — `open: "aberto"`, `expired: "vencido"`, `closed_out: "descartado"`; equipamento — `refrigerator: "geladeira"`, `chamber: "câmara"`, `freezer: "freezer"`.
- `doseProblems` confere, nesta ordem e todos de uma vez (path = campo do corpo): vacina do catálogo (`immunobiological_code`: "escolha a vacina"), dose (`dose_code`: "escolha a dose"), estratégia (`strategy`: "informe a estratégia"), grupo (`care_group`: "informe o grupo de atendimento"), via (`route`: "informe a via"), local (`site`: "informe o local de aplicação"), hora (`applied_at`: vazia → "informe data e hora da aplicação"; depois de `now` → "a hora da aplicação não pode ser futura"), lote (`lot_id`: vazio → "escolha o lote do estoque da sala"; fora de `ctx.lots` → `LOT_REREAD`), frasco (`vial_id`: multidose sem frasco → "<nome> é multidose: escolha o frasco aberto ou abra um frasco"; frasco fora de `vialsForDose(ctx.vials, [lotId], now)` → `VIAL_REREAD`; frasco de outro lote → "o frasco é de outro lote"), motivo (`override_reason`: preenchido e fora de 10–500).
- `doseBody`: `applied_at` = `parseCityLocal(appliedAt).toISOString()`; textos com `trim()`; opcionais vazios (`maternal_condition`, `vial_id`, `campaign_id`, `override_reason`) saem do objeto.
- `parseCelsius`: `/^-?\d{1,2}([.,]\d)?$/` depois de `trim()`, faixa −50…50; fora disso `null`. `readingProblems`: cada campo vazio ou inválido → "informe a temperatura <atual|mínima|máxima> (ex.: 4,5)"; hora vazia/futura; mínima > atual → "a mínima não pode ser maior que a atual"; atual > máxima → "a máxima não pode ser menor que a atual". `outOfRange` = qualquer de atual/mínima/máxima fora de `[min_c, max_c]`.
- `vialRemaining`: `expires_at <= now` → "vencido"; senão "vence em H h M min".
- Frases das recusas (contrato §5–§8; com `vaccinationRefusals` igual ao `pharmacyRefusals` do 27a — `problems[]` primeiro, senão o topo; código novo vira "recusado pelo servidor (<código>)"): `lot_not_in_room: "o lote não está no estoque desta sala"`, `lot_expired: "o lote venceu — escolha outro"`, `lot_blocked: "o lote está bloqueado por temperatura fora da faixa até a conduta"`, `vial_expired: "o frasco venceu — descarte e abra outro"`, `vial_empty: "o frasco não tem mais doses — abra outro"`, `vial_lot_mismatch: "o frasco é de outro lote"`, `vial_required: "vacina multidose: escolha o frasco aberto ou abra um frasco"`, `applied_at_in_future: "a hora da aplicação não pode ser futura"`, `dose_not_in_calendar: "esta dose não existe no calendário para esta vacina"`, `interval_too_short: "intervalo menor que o mínimo desde a dose anterior"`, `age_out_of_range: "idade fora da faixa do calendário para esta dose"`, `sex_not_allowed: "esta vacina não é indicada para o sexo do cadastro"`, `maternal_condition_required: "informe a condição maternal (gestante ou puérpera)"`, `cancel_not_allowed: "só quem registrou ou o responsável pela sala cancela"`, `already_cancelled: "esta dose já foi cancelada — abra a caderneta de novo"`, `not_room_staff: "você não está na equipe desta sala"`, `cbo_not_allowed: "o CBO desta pessoa não permite vacinar"`, `room_lead_requires_nurse: "o responsável pela sala precisa ser enfermeiro (CBO 2235)"`, `already_exists: "a rede de frio da cidade já existe"`, `already_notified: "esta dose já tem notificação de ESAVI aberta"`, `excursion_closed: "a conduta desta excursão já foi decidida — recarregue"`, `room_lead_required: "só o responsável pela sala faz isso"`, `feature_disabled` → `VACCINATION_DISABLED`.

Em `src/lib/pharmacy.ts`: `export const itemKey = (item: StockItem): string => item.catalog_item_id ?? \`imm:${item.immunobiological_code ?? ""}\`;` e as duas entradas novas em `KIND_LABEL`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/vaccination.test.ts src/lib/pharmacy.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/vaccination.ts src/lib/vaccination.test.ts src/lib/pharmacy.ts
$G commit -m "feat: add the vaccination room screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Vacinação", portão e escolha do local

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts` (as duas contagens `NAV_GROUPS.length - 15` → `- 16` e `- 14` → `- 15`, mais a linha `expect(groups.some((g) => g.label === "Vacinação")).toBe(false);` nos dois blocos), `src/shell/catalog.ts`, `src/App.tsx`
- Create: `src/modules/vaccination/VaccinationGate.tsx`, `src/modules/vaccination/VaccineLocationPick.tsx`, as telas-portão `src/modules/VaccinationRoom.tsx`, `src/modules/VaccinationPending.tsx`, `src/modules/VaccinationStock.tsx`, `src/modules/VaccinationColdChain.tsx`, `src/modules/VaccinationSummary.tsx`, `src/modules/VaccinationPanel.tsx`, `src/modules/VaccinationAdmin.tsx` (cada uma só o portão e um `EmptyState` "em construção" que as Tasks 4–15 substituem)
- Test: `src/shell/modules.vaccination.test.ts`, `src/modules/vaccination/VaccinationGate.test.tsx`

**Interfaces:**
- Consumes: `canUseVaccination`, `vaccinationRoles`, `locationAbilities`, `VACCINATION_DISABLED`, `VACCINATION_NO_ROLE`, `NO_VACCINE_LOCATION`, `VAC_LOCATIONS_KEY`, `STATIC_QUERY`, `vaccinationError` (Task 2); `listMyVaccineLocations`, `VaccineLocation` (Task 1); `locationKindLabel` (`pharmacy.ts`).
- Produces:
  - `ModuleId` ganha `"vaccination-room" | "vaccination-pending" | "vaccination-stock" | "vaccination-cold-chain" | "vaccination-summary" | "vaccination-panel" | "vaccination-admin"`; grupo `{ label: "Vacinação", items: [ sala "Sala de vacina" ✚, pendências "Pendências do envio" ⇪, estoque "Imunobiológicos e frascos" ▣, frio "Cadeia de frio" ❄, resumo "Resumo mensal" ▤, painel "Painel de vacinação" ▦, admin "Salas e rede de frio" ⚙ ] }` logo depois de "Farmácia";
  - visibilidade (`vacOn = canUseVaccination(user)`): sala e pendências — `vacOn && isProfessional`; estoque, frio e resumo — `vacOn && (isProfessional || isStockKeeper)`; painel — `vacOn && (isAdmin || isAnalyst)`; admin — `vacOn && isAdmin`;
  - `VaccinationGate({ title, sub, allow, children }: { title: string; sub: string; allow(r: ReturnType<typeof vaccinationRoles>): boolean; children(r): ReactNode })` (molde do `PharmacyGate`);
  - `VaccineLocationPick({ allow, children }: { allow(a: ReturnType<typeof locationAbilities>, loc: VaccineLocation): boolean; children(loc: VaccineLocation): ReactNode })` (molde do `LocationPick`: lê `listMyVaccineLocations` com `STATIC_QUERY`, mostra só os ativos que o `allow` aceita, seletor "Local" quando há mais de um, `NO_VACCINE_LOCATION` quando nenhum; trocar de local remonta o conteúdo; nada em storage).

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.vaccination.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor } from "./modules";
import { CATALOG } from "./catalog";

const user = (roles: string[], features: string[] | undefined = [ "clinical_record", "vaccination" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });
const itemsOf = (groups: ReturnType<typeof navGroupsFor>) => groups.find((g) => g.label === "Vacinação")?.items.map((i) => i.id) ?? [];

describe("grupo Vacinação (módulo 29)", () => {
  it("cada papel vê o seu", () => {
    expect(itemsOf(navGroupsFor(user([ "health_professional" ])))).toEqual([
      "vaccination-room", "vaccination-pending", "vaccination-stock", "vaccination-cold-chain", "vaccination-summary" ]);
    expect(itemsOf(navGroupsFor(user([ "stock_keeper" ])))).toEqual([ "vaccination-stock", "vaccination-cold-chain", "vaccination-summary" ]);
    expect(itemsOf(navGroupsFor(user([ "municipal_admin" ])))).toEqual([ "vaccination-panel", "vaccination-admin" ]);
    expect(itemsOf(navGroupsFor(user([ "analyst" ])))).toEqual([ "vaccination-panel" ]);
  });
  it("some sem o interruptor, para outros papéis e para o operador; não depende da farmácia", () => {
    expect(itemsOf(navGroupsFor(user([ "health_professional", "municipal_admin" ], [ "clinical_record", "pharmacy" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "citizen_verifier", "campaign_manager", "pharmacist" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([], undefined, true)))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "stock_keeper" ], [ "vaccination" ])))).toEqual([ "vaccination-stock", "vaccination-cold-chain", "vaccination-summary" ]);
  });
  it("vem depois da Farmácia, com ícone e descrição", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Vacinação")).toBe(labels.indexOf("Farmácia") + 1);
    expect(CATALOG.groups["Vacinação"].description).toBe("Sala de vacina, caderneta, envio à RNDS, estoque e cadeia de frio");
    expect(CATALOG.screens["vaccination-room"].description).toBe("Caderneta do cidadão, dose com lote e frasco, transcrição");
    expect(CATALOG.screens["vaccination-pending"].description).toBe("Doses paradas, recusadas ou perto do prazo de 24 h");
    expect(CATALOG.screens["vaccination-stock"].description).toBe("Entrada, transferência, frascos abertos, posição e perdas");
    expect(CATALOG.screens["vaccination-cold-chain"].description).toBe("Equipamentos, temperatura duas vezes ao dia e conduta");
    expect(CATALOG.screens["vaccination-summary"].description).toBe("Movimentação do mês para lançar no SIES/SIPNI");
    expect(CATALOG.screens["vaccination-panel"].description).toBe("Doses, cobertura do C2, envio à RNDS e perdas");
    expect(CATALOG.screens["vaccination-admin"].description).toBe("Salas, vacinadores, rede de frio e início na cidade");
  });
});
```

```tsx
// src/modules/vaccination/VaccinationGate.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { COLD_CENTRAL, LEAD_ROOM, ROOM, vacUser } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { VaccinationGate } from "./VaccinationGate";
import { VaccineLocationPick } from "./VaccineLocationPick";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyVaccineLocations: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount(user = vacUser(), allow = () => true) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(user);
  return renderWithProviders(
    <VaccinationGate title="Sala de vacina" sub="caderneta e dose" allow={(r) => r.professional}>
      {() => <VaccineLocationPick allow={allow}>{(loc) => <p>{`dentro de ${loc.name}`}</p>}</VaccineLocationPick>}
    </VaccinationGate>);
}

describe("portão e local da vacinação", () => {
  it("interruptor desligado: frase, sem ler locais", async () => {
    mount(vacUser([ "health_professional" ], { features: [ "clinical_record" ] }));
    expect(await screen.findByText("a sala de vacina está desligada nesta cidade")).not.toBeNull();
    expect(api.listMyVaccineLocations).not.toHaveBeenCalled();
  });
  it("papel errado: frase", async () => {
    mount(vacUser([ "citizen_verifier" ]));
    expect(await screen.findByText("seu papel não permite usar a sala de vacina")).not.toBeNull();
  });
  it("sem local: frase; com dois, seletor e troca remonta", async () => {
    vi.mocked(api.listMyVaccineLocations).mockResolvedValueOnce([]);
    mount();
    expect(await screen.findByText(/não está na equipe de nenhuma sala/)).not.toBeNull();
    cleanup();
    vi.mocked(api.listMyVaccineLocations).mockResolvedValueOnce([ ROOM, { ...LEAD_ROOM, id: "l6", name: "Sala UBS Norte", room_id: "r2" }, COLD_CENTRAL ]);
    mount(vacUser(), (a: { vaccinate: boolean }) => a.vaccinate);
    expect(await screen.findByText("dentro de Sala de vacina UBS Centro")).not.toBeNull();
    const select = screen.getByLabelText("Local") as HTMLSelectElement;
    expect([ ...select.options ].map((o) => o.textContent)).toEqual([ "Sala de vacina UBS Centro — Sala de vacina", "Sala UBS Norte — Sala de vacina" ]);
    fireEvent.change(select, { target: { value: "l6" } });
    await waitFor(() => expect(screen.getByText("dentro de Sala UBS Norte")).not.toBeNull());
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/shell/modules.vaccination.test.ts src/shell/modules.test.ts src/modules/vaccination/VaccinationGate.test.tsx src/shell/catalog.test.ts`
Expected: FAIL — o grupo, as entradas do catálogo e as peças não existem.

- [ ] **Step 3: Implemente o grupo, o catálogo, o portão e as telas-portão**

`modules.ts`: o grupo e as regras de visibilidade do Interfaces, com o comentário `// Módulo 29 (ADR 0047): vaccination, sem exigir pharmacy. Sala e pendências: profissional (o papel na sala vem de /vaccination/my_locations); estoque, frio e resumo: profissional ou almoxarife; painel: admin e analyst; administração: admin.` `catalog.ts`: `IconVaccine` no grupo e `IconNeedle`, `IconCloudUpload`, `IconVaccineBottle`, `IconTemperatureSnow`, `IconCalendarMonth`, `IconReportMedical`, `IconFridge` nas telas, com as descrições do teste. `App.tsx`: um `case` por tela (`onGoToSecurity={() => setActive("security")}` em `VaccinationRoom`, `VaccinationStock`, `VaccinationColdChain` e `VaccinationAdmin`). Cada tela-portão: `VaccinationGate` com título e subtítulo do catálogo e o `allow` da visibilidade (painel: `r.admin || r.analyst`; admin: `r.admin`; o resto: `r.professional || r.keeper`, sala e pendências só `r.professional`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/shell src/modules/vaccination/VaccinationGate.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (as guardas do tema aceitam as telas-portão).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.vaccination.test.ts src/shell/catalog.ts src/App.tsx \
  src/modules/vaccination/VaccinationGate.tsx src/modules/vaccination/VaccineLocationPick.tsx src/modules/vaccination/VaccinationGate.test.tsx \
  src/modules/VaccinationRoom.tsx src/modules/VaccinationPending.tsx src/modules/VaccinationStock.tsx src/modules/VaccinationColdChain.tsx \
  src/modules/VaccinationSummary.tsx src/modules/VaccinationPanel.tsx src/modules/VaccinationAdmin.tsx
$G commit -m "feat: add the vaccination menu group, gate and location picker

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Salas, vacinadores, rede de frio e data do go-live (F-29.1)

**Files:**
- Modify: `src/modules/VaccinationAdmin.tsx`, `src/modules/pharmacy/StaffPanel.tsx` (prop `roles?: PharmacyRole[]`, padrão os três de hoje)
- Create: `src/modules/vaccination/RoomsPanel.tsx`, `src/modules/vaccination/RoomStaffPanel.tsx`, `src/modules/vaccination/GoLivePanel.tsx`
- Test: `src/modules/VaccinationAdmin.test.tsx`

**Interfaces:**
- Consumes: `listVaccineRooms`, `createVaccineRoom`, `updateVaccineRoom`, `addRoomStaff`, `removeRoomStaff`, `createColdChainCentral`, `getVaccinationSettings`, `updateVaccinationSettings`, `listAllUnits`, `listMemberships` (Task 1 e existentes); `VAC_ADMIN_KEY`, `VAC_SETTINGS_KEY`, `STATIC_QUERY`, `vaccinationError`, `reasonProblem` (Task 2); `SensitiveAction`; `StaffPanel` (27a).
- Produces: `VaccinationAdmin({ onGoToSecurity }: { onGoToSecurity?(): void })`; `RoomsPanel({ rooms, units, selectedId, onSelectStaff, onSaved, onGoToSecurity })` (tabela de salas: unidade, nome, ativa, nº de vacinadores; "Nova sala" e "Editar" num formulário — unidade só das unidades ativas sem sala, nome 3–120, ativa —, "Revisar" abre o `SensitiveAction` com step-up); `RoomStaffPanel({ room, memberships, onChanged, onGoToSecurity })` (pessoas de `listMemberships` com papel `health_professional`, uma por pessoa; papel `vaccinator` | `room_lead`; vincular/desvincular com step-up; recusas `cbo_not_allowed`/`room_lead_requires_nurse` pela frase da Task 2); bloco "Rede de frio municipal" (sem central: botão "Criar a rede de frio" com step-up; com central: `StaffPanel` do 27a com `roles={[ "stock_keeper" ]}`); `GoLivePanel` (data `go_live_on`, "Salvar" com step-up; texto fixo `GO_LIVE_HINT = "Quem nasceu a partir desta data já tem a caderneta conferida desde o nascimento."`).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/VaccinationAdmin.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../test/campaignFixtures";
import { vacUser, vaccineRoom } from "../test/vaccinationFixtures";
import * as api from "../lib/api";
import { VaccinationAdmin } from "./VaccinationAdmin";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listVaccineRooms: vi.fn(),
  createVaccineRoom: vi.fn(), addRoomStaff: vi.fn(), removeRoomStaff: vi.fn(), createColdChainCentral: vi.fn(), getVaccinationSettings: vi.fn(),
  updateVaccinationSettings: vi.fn(), listAllUnits: vi.fn(), listMemberships: vi.fn(), addLocationStaff: vi.fn() }));
const refuse = (error: string) => new api.ApiError(422, { error }, "422");
const CENTRAL_ADMIN = { id: "l9", name: "Rede de frio municipal", kind: "cold_chain_central", health_unit: null, active: true, staff: [] };

beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser([ "municipal_admin" ]));
  vi.mocked(api.listVaccineRooms).mockResolvedValue({ rooms: [ vaccineRoom() ], cold_chain_central: null });
  vi.mocked(api.listAllUnits).mockResolvedValue([ { id: "u1", name: "UBS Centro", active: true }, { id: "u2", name: "UBS Norte", active: true } ] as never);
  vi.mocked(api.listMemberships).mockResolvedValue([
    { id: "m1", user: { id: "us7", email_address: "enf.bia@curitiba.demo" }, role: "health_professional" },
    { id: "m2", user: { id: "us8", email_address: "farm@curitiba.demo" }, role: "pharmacist" } ] as never);
  vi.mocked(api.getVaccinationSettings).mockResolvedValue({ go_live_on: null });
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("administração da vacinação (F-29.1)", () => {
  it("nova sala: só unidade sem sala; step-up; relê", async () => {
    vi.mocked(api.createVaccineRoom).mockResolvedValue(vaccineRoom({ id: "r2", name: "Sala UBS Norte", health_unit: { id: "u2", name: "UBS Norte" } }));
    renderWithProviders(<VaccinationAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova sala" }));
    const unit = screen.getByLabelText("Unidade") as HTMLSelectElement;
    expect([ ...unit.options ].map((o) => o.value)).toEqual([ "", "u2" ]);
    fireEvent.change(unit, { target: { value: "u2" } });
    fireEvent.change(screen.getByLabelText("Nome da sala"), { target: { value: "Sala UBS Norte" } });
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar sala" }));
    await waitFor(() => expect(api.createVaccineRoom).toHaveBeenCalledWith({ health_unit_id: "u2", name: "Sala UBS Norte", active: true }));
    await waitFor(() => expect(api.listVaccineRooms).toHaveBeenCalledTimes(2));
  });

  it("equipe: só profissional de saúde; recusa do CBO pela frase", async () => {
    vi.mocked(api.addRoomStaff).mockRejectedValue(refuse("room_lead_requires_nurse"));
    renderWithProviders(<VaccinationAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Equipe — Sala de vacina UBS Centro" }));
    const staff = screen.getByRole("region", { name: "Equipe da sala" });
    fireEvent.change(within(staff).getByLabelText("Papel na sala"), { target: { value: "room_lead" } });
    const person = within(staff).getByLabelText("Pessoa") as HTMLSelectElement;
    expect([ ...person.options ].map((o) => o.textContent)).toEqual([ "escolha…", "enf.bia@curitiba.demo" ]);
    fireEvent.change(person, { target: { value: "us7" } });
    fireEvent.click(within(staff).getByRole("button", { name: "Vincular" }));
    fireEvent.click(await screen.findByRole("button", { name: "Vincular à sala" }));
    expect(await screen.findByText("o responsável pela sala precisa ser enfermeiro (CBO 2235)")).not.toBeNull();
  });

  it("rede de frio: cria uma vez; depois mostra a equipe só com almoxarife", async () => {
    vi.mocked(api.createColdChainCentral).mockRejectedValueOnce(new api.ApiError(409, { error: "already_exists" }, "409"));
    renderWithProviders(<VaccinationAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Criar a rede de frio" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar" }));
    expect(await screen.findByText("a rede de frio da cidade já existe")).not.toBeNull();
    cleanup();
    vi.mocked(api.listVaccineRooms).mockResolvedValue({ rooms: [], cold_chain_central: CENTRAL_ADMIN as never });
    renderWithProviders(<VaccinationAdmin />);
    const role = await screen.findByLabelText("Papel no local") as HTMLSelectElement;
    expect([ ...role.options ].map((o) => o.value)).toEqual([ "", "stock_keeper" ]);
  });

  it("go-live: data com step-up e a explicação", async () => {
    vi.mocked(api.updateVaccinationSettings).mockResolvedValue({ go_live_on: "2026-12-01" });
    renderWithProviders(<VaccinationAdmin />);
    expect(await screen.findByText("Quem nasceu a partir desta data já tem a caderneta conferida desde o nascimento.")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Início da sala de vacina na cidade"), { target: { value: "2026-12-01" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar data" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar" }));
    await waitFor(() => expect(api.updateVaccinationSettings).toHaveBeenCalledWith({ go_live_on: "2026-12-01" }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationAdmin.test.tsx`
Expected: FAIL — a tela é só o portão.

- [ ] **Step 3: Implemente** `VaccinationAdmin` (molde do `PharmacyAdmin`: lê salas, unidades, vínculos e configurações com `STATIC_QUERY`; depois de cada escrita relê `VAC_ADMIN_KEY`/`VAC_SETTINGS_KEY`), `RoomsPanel`, `RoomStaffPanel` (`<section aria-label="Equipe da sala">`), `GoLivePanel` e a prop `roles` do `StaffPanel` (`const ROLES = props.roles ?? [ "pharmacist", "pharmacy_assistant", "stock_keeper" ]`; o teste do 27a continua sem passar a prop). Botões de rótulo exato: "Nova sala", "Editar — <nome>", "Equipe — <nome>", "Revisar", "Criar sala"/"Salvar sala" (confirmação), "Vincular"/"Vincular à sala", "Desvincular — <pessoa>", "Criar a rede de frio"/"Criar", "Salvar data"/"Salvar". As recusas passam por `translateError: (err) => vaccinationError(err)` só para os códigos do mapa da Task 2.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationAdmin.test.tsx src/modules/PharmacyAdmin.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/VaccinationAdmin.tsx src/modules/VaccinationAdmin.test.tsx src/modules/vaccination/RoomsPanel.tsx \
  src/modules/vaccination/RoomStaffPanel.tsx src/modules/vaccination/GoLivePanel.tsx src/modules/pharmacy/StaffPanel.tsx
$G commit -m "feat: add vaccine rooms, staff, cold chain and go-live administration

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Credencial RNDS na tela de Integrações (F-29.1)

**Files:**
- Modify: `src/lib/integrations.ts`, `src/lib/integrations.test.ts`, `src/modules/Integrations.tsx`
- Create: `src/modules/integrations/RndsCredentialForm.tsx`
- Test: `src/modules/Integrations.rnds.test.tsx`

**Interfaces:**
- Consumes: `setRndsCredential`, `checkIntegrationCredential`, `getIntegrations` (Task 1 e existentes); `SensitiveAction`; `featureLabel`.
- Produces:
  - em `integrations.ts`: `credentialLabel("rnds") === "RNDS — vacinação (certificado A1)"`; `missingPhrase` ganha `record_mode_not_record` → "o modo de prontuário da cidade não é o prontuário do Rota Saúde (quem define é o operador da plataforma)" e `<chave>_disabled` → `ligue antes: ${featureLabel(chave)}` (o que hoje cai em "pendência não reconhecida"); `MESSAGES` ganha `invalid_certificate: "o certificado não abriu com essa senha — confira o arquivo .pfx e a senha"`, `certificate_expired: "o certificado está vencido — use um certificado A1 válido"`, `invalid_requester_cnes: "o CNES solicitante tem 7 dígitos"` (D1); `pfxProblem(file: File | null): string | null` (sem arquivo → "escolha o arquivo do certificado (.pfx ou .p12)"; extensão fora de `.pfx`/`.p12` → "o certificado A1 é um arquivo .pfx ou .p12"; acima de 100 KB → "o arquivo passa de 100 KB — confira se é o certificado A1"); `cnesProblem(raw: string): string | null` (7 dígitos);
  - `RndsCredentialForm({ credential, onDone, onCancel, onGoToSecurity }: { credential: IntegrationCredential; onDone(message: string): void; onCancel(): void; onGoToSecurity?(): void })`: passo 1 — arquivo (`<input type="file" accept=".pfx,.p12,application/x-pkcs12">`) e "CNES solicitante", "Continuar" confere os dois (`pfxProblem`, `cnesProblem`); passo 2 — `SensitiveAction` com step-up e o campo "Senha do certificado" (`type: "password"`), `run` chama `setRndsCredential(file, senha, cnes)`; o nome do arquivo só aparece na tela ("escolhido: <nome>"); fechar descarta o arquivo da memória;
  - em `Integrations.tsx`: `editing.kind === "rnds"` monta o `RndsCredentialForm` em vez do usuário/senha.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/Integrations.rnds.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { renderWithProviders, sessionWith } from "../test/campaignFixtures";
import * as api from "../lib/api";
import { missingPhrase } from "../lib/integrations";
import { Integrations } from "./Integrations";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getIntegrations: vi.fn(),
  setRndsCredential: vi.fn(), checkIntegrationCredential: vi.fn() }));
const RNDS = { kind: "rnds" as const, set: false, set_at: null, set_by: null, last_check_at: null, last_check_status: null, last_check_message: null };
const pfx = (name = "secretaria.pfx", size = 3000) => new File([ new Uint8Array(size) ], name, { type: "application/x-pkcs12" });

beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ], { features: [ "vaccination" ] }));
  vi.mocked(api.getIntegrations).mockResolvedValue({ record_mode: "record", pec_url_set: true, ibge_code_set: true, credentials: [ RNDS ],
    features: [ { key: "rnds_immunization", enabled: true, usable: false, missing: [ "credential_missing:rnds", "vaccination_disabled" ] } ] });
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("credencial RNDS (F-29.1)", () => {
  it("pendências novas em português", () => {
    expect(missingPhrase("record_mode_not_record")).toMatch(/não é o prontuário do Rota Saúde/);
    expect(missingPhrase("vaccination_disabled")).toBe("ligue antes: Sala de vacina");
  });

  it("arquivo e CNES conferidos antes; senha no step-up; nada do segredo volta", async () => {
    vi.mocked(api.setRndsCredential).mockResolvedValue({ ...RNDS, set: true, set_at: "2026-11-02T10:00:00-03:00", set_by: "admin" });
    renderWithProviders(<Integrations />);
    expect(await screen.findByText("ligue antes: Sala de vacina")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Cadastrar — RNDS — vacinação (certificado A1)" }));
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(screen.getByText("escolha o arquivo do certificado (.pfx ou .p12)")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Certificado A1"), { target: { files: [ pfx("certificado.pdf") ] } });
    fireEvent.change(screen.getByLabelText("CNES solicitante"), { target: { value: "20012" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(screen.getByText("o certificado A1 é um arquivo .pfx ou .p12")).not.toBeNull();
    expect(screen.getByText("o CNES solicitante tem 7 dígitos")).not.toBeNull();
    const file = pfx();
    fireEvent.change(screen.getByLabelText("Certificado A1"), { target: { files: [ file ] } });
    fireEvent.change(screen.getByLabelText("CNES solicitante"), { target: { value: "2001234" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.change(await screen.findByLabelText("Senha do certificado"), { target: { value: "s3nh4" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar certificado" }));
    await waitFor(() => expect(api.setRndsCredential).toHaveBeenCalledWith(file, "s3nh4", "2001234"));
    expect(await screen.findByText(/Certificado da RNDS salvo/)).not.toBeNull();
    expect(screen.queryByText(/s3nh4/)).toBeNull();
  });

  it("certificado que não abre: frase da tela", async () => {
    vi.mocked(api.setRndsCredential).mockRejectedValue(new api.ApiError(422, { error: "invalid_certificate" }, "422"));
    renderWithProviders(<Integrations />);
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar — RNDS — vacinação (certificado A1)" }));
    fireEvent.change(screen.getByLabelText("Certificado A1"), { target: { files: [ pfx() ] } });
    fireEvent.change(screen.getByLabelText("CNES solicitante"), { target: { value: "2001234" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.change(await screen.findByLabelText("Senha do certificado"), { target: { value: "errada" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar certificado" }));
    expect(await screen.findByText("o certificado não abriu com essa senha — confira o arquivo .pfx e a senha")).not.toBeNull();
  });
});
```

E em `src/lib/integrations.test.ts`: `expect(credentialLabel("rnds")).toBe("RNDS — vacinação (certificado A1)")`, `expect(pfxProblem(new File([ new Uint8Array(102_401) ], "a.pfx"))).toBe("o arquivo passa de 100 KB — confira se é o certificado A1")`, `expect(pfxProblem(new File([ "x" ], "A.P12"))).toBeNull()` e `expect(cnesProblem(" 2001234 ")).toBeNull()`.

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/Integrations.rnds.test.tsx src/lib/integrations.test.ts`
Expected: FAIL — rótulo cru, `pfxProblem`/`cnesProblem` não existem, o formulário de usuário/senha aparece para a RNDS.

- [ ] **Step 3: Implemente** o que o Interfaces descreve. O `SensitiveAction` não muda (passo 1 fora dele, passo 2 com o único campo de senha). Frase de sucesso: `Certificado da RNDS salvo. Teste a conexão para conferir a troca de token.` O "Testar conexão" segue o fluxo existente (`checkIntegrationCredential("rnds")`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/Integrations.rnds.test.tsx src/modules/Integrations.test.tsx src/lib/integrations.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS (o teste existente da tela segue verde para LEDI e CADSUS).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/integrations.ts src/lib/integrations.test.ts src/modules/Integrations.tsx src/modules/integrations/RndsCredentialForm.tsx src/modules/Integrations.rnds.test.tsx
$G commit -m "feat: add the RNDS A1 certificate to the integrations screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Busca do cidadão, caderneta e conferência (F-29.2)

**Files:**
- Modify: `src/modules/VaccinationRoom.tsx`
- Create: `src/modules/vaccination/PatientPick.tsx` (com `NewPersonForm`), `src/modules/vaccination/CardView.tsx`, `src/modules/vaccination/ImmunizationRow.tsx`
- Test: `src/modules/vaccination/CardView.test.tsx`

**Interfaces:**
- Consumes: `getVaccinationCard`, `confirmCardComplete`, `lookupVaccinationPerson`, `findVaccinationDuplicates`, `getVaccinationCatalog` (Task 1); `cardGroups`, `effectiveStatus`, `cardStatusLabel`, `cardStatusTone`, `checkModeLabel`, `UNCHECKED_NOTICE`, `VAC_CATALOG_KEY`, `STATIC_QUERY`, `vaccinationError` (Task 2); `VaccinationGate`, `VaccineLocationPick` (Task 3); `PersonStart`, `PersonChoice`, `PersonRestart`, `PersonSource` (`src/modules/territory/PersonStart.tsx`); `newPatientProblem` (`src/lib/territoryDictionaries.ts`).
- Produces:
  - `VACCINATION_PERSON_SOURCE: PersonSource` (`lookupVaccinationPerson`, `findVaccinationDuplicates`, `vaccinationError`, `pickLabel: (name) => \`Abrir a caderneta de ${name}\``);
  - `PatientPick({ today, onPick, restart }: { today: string; onPick(req: CardRequest): void; restart?: PersonRestart })` — o `PersonStart` com a fonte da vacinação; `patient` → `{ patient_id }`; `citizen` → `{ citizen_id }`; `new` → o formulário `NewPersonForm` (nome completo, nascimento, sexo, mãe; CPF da busca, se havia; conferido por `newPatientProblem` de `src/lib/territoryDictionaries.ts`) → `{ new_patient, duplicates_reviewed? }` (D3; recém-nascido sem cadastro). 409 `duplicate_suspected` na leitura da caderneta volta ao `PersonStart` com `restart` (molde do `ManualNotificationPanel` do 25);
  - `CardView({ room, card, catalog, now, onCard, onRecord, onTranscribe, onGoToSecurity }: { room: VaccineLocation; card: VaccinationCard; catalog: VaccinationCatalog; now: Date; onCard(c: VaccinationCard): void; onRecord(item: CardItem | null): void; onTranscribe(): void; onGoToSecurity?(): void })` — cabeçalho (`<h2 tabIndex={-1}>` com o nome, que recebe o foco ao abrir; nascimento; "com CPF"/"CNS <mascarado>"; `Tag` da conferência ou "não conferida"); com `checked: null`, `UNCHECKED_NOTICE` e os botões "Transcrever caderneta" e "Confirmar caderneta completa" (dois passos: o segundo clique é "Sim, a caderneta está completa"); quatro blocos — "Devidas hoje", "Em atraso", "Próximas", "Aplicadas" — e, sem conferência, "Não conferidas" no lugar de "Em atraso"; cada devida/atrasada tem "Registrar <vacina> — <dose>" (`onRecord(item)`); "Outra vacina" (`onRecord(null)`); a lista "Doses registradas" com `ImmunizationRow`;
  - `ImmunizationRow({ imm, onChanged, onGoToSecurity, actions? }: { imm: Immunization; onChanged(): void; onGoToSecurity?(): void; actions?: ReactNode })` — vacina, dose, data, lote ("transcrição" quando for), sala, quem aplicou, situação ativa/cancelada com motivo, e o espaço de ações que as Tasks 8, 9 e 15 preenchem;
  - `VaccinationRoom({ now?, onGoToSecurity? })`: portão → local onde `vaccinate` → busca → `getVaccinationCard` por **mutação** (`gcTime: 0`, `seq` contra resposta atrasada) → `CardView`; "Outro cidadão" volta à busca e limpa o estado.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/vaccination/CardView.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CATALOG29, NOW29, ROOM, card, cardItem, immunization, vacUser } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { VaccinationRoom } from "../VaccinationRoom";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyVaccineLocations: vi.fn(),
  getVaccinationCatalog: vi.fn(), lookupVaccinationPerson: vi.fn(), findVaccinationDuplicates: vi.fn(), getVaccinationCard: vi.fn(),
  confirmCardComplete: vi.fn() }));
beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
  vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ ROOM ]);
  vi.mocked(api.getVaccinationCatalog).mockResolvedValue(CATALOG29);
  vi.mocked(api.lookupVaccinationPerson).mockResolvedValue({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null,
    has_cpf: true, cns_masked: null });
});
afterEach(() => { cleanup(); vi.clearAllMocks(); localStorage.clear(); sessionStorage.clear(); });

async function openCard() {
  renderWithProviders(<VaccinationRoom now={NOW29} />);
  fireEvent.change(await screen.findByLabelText("CPF"), { target: { value: "123.456.789-09" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
  fireEvent.click(await screen.findByRole("button", { name: "Abrir a caderneta de Joana Lima" }));
}

describe("caderneta na sala (F-29.2)", () => {
  it("abre pela busca, uma leitura só, foco no nome; blocos e ação de registrar", async () => {
    vi.mocked(api.getVaccinationCard).mockResolvedValue(card());
    await openCard();
    const title = await screen.findByRole("heading", { name: "Joana Lima" });
    await waitFor(() => expect(document.activeElement).toBe(title));
    expect(api.getVaccinationCard).toHaveBeenCalledTimes(1);
    expect(api.getVaccinationCard).toHaveBeenCalledWith("r1", { patient_id: "pa1" });
    expect(screen.getByText("nasceu depois do início — conferida desde o nascimento")).not.toBeNull();
    const today = screen.getByRole("region", { name: "Devidas hoje" });
    expect(within(today).getByRole("button", { name: "Registrar Penta (DTP/HepB/Hib) — 2ª dose" })).not.toBeNull();
    expect(within(screen.getByRole("region", { name: "Próximas" })).getByText("3ª dose")).not.toBeNull();
    expect(within(screen.getByRole("region", { name: "Doses registradas" })).getByText("PNT2611")).not.toBeNull();
    expect(JSON.stringify({ ...localStorage, ...sessionStorage })).toBe("{}");
  });

  it("caderneta não conferida nunca mostra atraso; confirmar completa relê", async () => {
    vi.mocked(api.getVaccinationCard).mockResolvedValue(card({ checked: null, items: [ cardItem({ status: "overdue", due_on: "2026-09-01" }) ] }));
    vi.mocked(api.confirmCardComplete).mockResolvedValue(card({ checked: { mode: "confirmed_complete", at: "2026-11-02T10:00:00-03:00", by_name: "Ana" },
      items: [ cardItem({ status: "overdue", due_on: "2026-09-01" }) ] }));
    await openCard();
    expect(await screen.findByText(/Caderneta não conferida/)).not.toBeNull();
    expect(screen.queryByText(/atraso/i)).toBeNull();
    expect(screen.getByRole("region", { name: "Não conferidas" })).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Confirmar caderneta completa" }));
    expect(api.confirmCardComplete).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Sim, a caderneta está completa" }));
    await waitFor(() => expect(api.confirmCardComplete).toHaveBeenCalledWith("r1", "pa1"));
    expect(await screen.findByRole("region", { name: "Em atraso" })).not.toBeNull();
    expect(screen.getByText("conferida: caderneta completa")).not.toBeNull();
  });

  it("dose cancelada aparece com o motivo; código de vacina novo aparece cru", async () => {
    vi.mocked(api.getVaccinationCard).mockResolvedValue(card({ immunizations: [ immunization({ status: "cancelled", cancel_reason: "lote errado digitado na sala" }),
      immunization({ id: "im3", immunobiological: { code: "999", name: "999" } }) ] }));
    await openCard();
    const list = await screen.findByRole("region", { name: "Doses registradas" });
    expect(within(list).getByText("cancelada: lote errado digitado na sala")).not.toBeNull();
    expect(within(list).getByText("999")).not.toBeNull();
  });

  it("recém-nascido sem cadastro: pessoa nova vai no corpo da caderneta", async () => {
    vi.mocked(api.lookupVaccinationPerson).mockRejectedValue(new api.ApiError(404, { error: "person_not_found" }, "404"));
    vi.mocked(api.getVaccinationCard).mockResolvedValue(card({ checked: { mode: "born_after_go_live", at: "2026-11-01T00:00:00-03:00", by_name: null } }));
    renderWithProviders(<VaccinationRoom now={NOW29} />);
    fireEvent.change(await screen.findByLabelText("CPF"), { target: { value: "123.456.789-09" } });
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar pessoa nova com este CPF" }));
    fireEvent.change(screen.getByLabelText("Nome completo"), { target: { value: "Davi Lima" } });
    fireEvent.change(screen.getByLabelText("Data de nascimento"), { target: { value: "2026-11-01" } });
    fireEvent.change(screen.getByLabelText("Sexo"), { target: { value: "male" } });
    fireEvent.click(screen.getByRole("button", { name: "Abrir a caderneta" }));
    await waitFor(() => expect(api.getVaccinationCard).toHaveBeenCalledWith("r1",
      { new_patient: { full_name: "Davi Lima", birth_date: "2026-11-01", sex: "male", cpf: "12345678909" } }));
  });

  it("fora da equipe da sala: frase, nada de caderneta", async () => {
    vi.mocked(api.getVaccinationCard).mockRejectedValue(new api.ApiError(403, { error: "not_room_staff" }, "403"));
    await openCard();
    expect(await screen.findByText("você não está na equipe desta sala")).not.toBeNull();
    expect(screen.queryByRole("heading", { name: "Joana Lima" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/CardView.test.tsx`
Expected: FAIL — a tela é só o portão.

- [ ] **Step 3: Implemente** `PatientPick`, `CardView`, `ImmunizationRow` e o corpo de `VaccinationRoom` como no Interfaces. Cada bloco é `<section aria-label="<título>">` com `DataTable` (`card: "title"` na vacina, `card: "actions"` no botão). Datas: `fmtDay` para `due_on`, `fmtDateTime` para `applied_at`. O catálogo vem de `getVaccinationCatalog` com `STATIC_QUERY` (sem pessoa). O texto "cancelada: <motivo>" só aparece na caderneta aberta (leitura com trilha), nunca em lista.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/CardView.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/VaccinationRoom.tsx src/modules/vaccination/PatientPick.tsx src/modules/vaccination/CardView.tsx \
  src/modules/vaccination/ImmunizationRow.tsx src/modules/vaccination/CardView.test.tsx
$G commit -m "feat: add the vaccination card with unchecked record handling

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Registro da dose — lote, frasco, avisos com motivo e aprazamento (F-29.3)

**Files:**
- Create: `src/modules/vaccination/DoseForm.tsx`
- Modify: `src/modules/VaccinationRoom.tsx` (abre o `DoseForm` no `onRecord` e volta à caderneta relida)
- Test: `src/modules/vaccination/DoseForm.test.tsx`

**Interfaces:**
- Consumes: `recordImmunization`, `listPositions`, `listVials`, `openVial`, `getVaccinationCard` (Task 1 e 27a); `newDoseDraft`, `lotsForDose`, `vialsForDose`, `doseProblems`, `doseBody`, `splitRefusals`, `vaccinationRefusals`, `reasonProblem`, `termName`, `vialRemaining`, `DOSE_SAVED`, `LOT_REREAD`, `VIAL_REREAD`, `VAC_VIALS_KEY`, `STATIC_QUERY` (Task 2); `PH_POSITIONS_KEY`, `lotOptionLabel` (`pharmacy.ts`).
- Produces: `DoseForm({ room, patient, catalog, item, now, onSaved, onCancel }: { room: VaccineLocation; patient: CardPatient; catalog: VaccinationCatalog; item: CardItem | null; now: Date; onSaved(imm: Immunization): void; onCancel(): void })`:
  - campos com rótulo, na ordem do Tab: "Vacina" (fixa quando veio do item; `select` do catálogo em "Outra vacina"), "Dose", "Estratégia", "Grupo de atendimento", "Via", "Local de aplicação", "Condição maternal" (só sexo feminino; opcional), "Data e hora da aplicação" (`datetime-local`, `max` = agora), "Lote" (`select` de `lotsForDose`, FEFO, com validade e saldo), "Frasco aberto" (só multidose: `select` de `vialsForDose` com "restam N doses · vence em…", sugerido o que vence primeiro; botão "Abrir frasco deste lote" chama `openVial` e escolhe o frasco novo);
  - "Registrar dose" (`type="submit"`; `busy` desabilita; `Enter` no formulário envia) confere `doseProblems` e mostra todos; recusa do api → `vaccinationRefusals` dividido por `splitRefusals`: as duras numa lista, os avisos com a frase e o campo "Motivo para aplicar mesmo assim" (10–500); com aviso e sem recusa dura, reenviar leva `override_reason`; `lot_blocked`/`lot_expired`/`lot_not_in_room` → relê `PH_POSITIONS_KEY` do local e limpa `lotId` (e `vialId`); `vial_expired`/`vial_empty`/`vial_lot_mismatch` → relê `VAC_VIALS_KEY` e limpa `vialId`; o resto do formulário fica;
  - sucesso: `onSaved(imm)`; quem chama relê a caderneta (mutação) e mostra `DOSE_SAVED` com o aprazamento — "Próxima dose: <vacina> <dose> em <data>" para o primeiro item `upcoming`/`due` do mesmo imunobiológico na caderneta relida (nenhum → "Esquema desta vacina completo no calendário.") — em `<p role="status" tabIndex={-1}>`, que recebe o foco.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/vaccination/DoseForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { BCG, BCG_ITEM, CATALOG29, NOW29, ROOM, cardItem, cardPatient, immunization, vacLotBalance, vacPosition, vacUser, vial } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { DoseForm } from "./DoseForm";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listPositions: vi.fn(), listVials: vi.fn(),
  openVial: vi.fn(), recordImmunization: vi.fn() }));
const refuse = (problems: { path: string; error: string }[]) => new api.ApiError(422, { error: problems[0].error, problems }, "422");
const onSaved = vi.fn();

beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
  vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition() ], next_before: null });
  vi.mocked(api.listVials).mockResolvedValue([]);
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

function mount(item = cardItem()) {
  return renderWithProviders(<DoseForm room={ROOM} patient={cardPatient()} catalog={CATALOG29} item={item} now={NOW29} onSaved={onSaved} onCancel={vi.fn()} />);
}
async function fill() {
  mount();
  fireEvent.change(await screen.findByLabelText("Estratégia"), { target: { value: "1" } });
  fireEvent.change(screen.getByLabelText("Grupo de atendimento"), { target: { value: "1" } });
  await waitFor(() => expect((screen.getByLabelText("Lote") as HTMLSelectElement).options.length).toBe(2));
  fireEvent.change(screen.getByLabelText("Lote"), { target: { value: "lot9" } });
}

describe("registro da dose (F-29.3)", () => {
  it("caminho feliz pelo teclado: Enter envia o corpo do contrato", async () => {
    vi.mocked(api.recordImmunization).mockResolvedValue(immunization({ id: "im9", dose: { code: "D2", name: "2ª dose" } }));
    await fill();
    expect((screen.getByLabelText("Via") as HTMLSelectElement).value).toBe("IM");
    fireEvent.submit(screen.getByRole("form", { name: "Registrar dose" }));
    await waitFor(() => expect(api.recordImmunization).toHaveBeenCalledWith("r1", { patient_id: "pa1", immunobiological_code: "42", dose_code: "D2",
      strategy: "1", care_group: "1", route: "IM", site: "VLD", applied_at: "2026-11-02T13:00:00.000Z", lot_id: "lot9" }));
    await waitFor(() => expect(onSaved).toHaveBeenCalledWith(expect.objectContaining({ id: "im9" })));
  });

  it("duplo envio registra uma dose só", async () => {
    let resolve!: (v: api.Immunization) => void;
    vi.mocked(api.recordImmunization).mockReturnValue(new Promise((r) => { resolve = r; }));
    await fill();
    const button = screen.getByRole("button", { name: "Registrar dose" });
    fireEvent.click(button); fireEvent.click(button);
    fireEvent.submit(screen.getByRole("form", { name: "Registrar dose" }));
    expect(api.recordImmunization).toHaveBeenCalledTimes(1);
    expect((button as HTMLButtonElement).disabled).toBe(true);
    resolve(immunization());
    await waitFor(() => expect(onSaved).toHaveBeenCalledTimes(1));
  });

  it("recusas locais de uma vez: hora futura e lote", async () => {
    mount();
    fireEvent.change(await screen.findByLabelText("Data e hora da aplicação"), { target: { value: "2026-11-02T10:05" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar dose" }));
    expect(screen.getByText("a hora da aplicação não pode ser futura")).not.toBeNull();
    expect(screen.getByText("escolha o lote do estoque da sala")).not.toBeNull();
    expect(screen.getByText("informe a estratégia")).not.toBeNull();
    expect(api.recordImmunization).not.toHaveBeenCalled();
  });

  it("lot_blocked: relê a posição, limpa o lote e mantém o resto", async () => {
    vi.mocked(api.recordImmunization).mockRejectedValue(refuse([ { path: "lot_id", error: "lot_blocked" } ]));
    await fill();
    vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition({ lots: [ vacLotBalance({ blocked: true }) ] }) ], next_before: null });
    fireEvent.click(screen.getByRole("button", { name: "Registrar dose" }));
    expect(await screen.findByText("o lote está bloqueado por temperatura fora da faixa até a conduta")).not.toBeNull();
    await waitFor(() => expect(api.listPositions).toHaveBeenCalledTimes(2));
    expect((screen.getByLabelText("Lote") as HTMLSelectElement).value).toBe("");
    expect((screen.getByLabelText("Estratégia") as HTMLSelectElement).value).toBe("1");
    expect(api.recordImmunization).toHaveBeenCalledTimes(1);
  });

  it("aviso e recusa dura juntos: o motivo não basta", async () => {
    const bcgLot = vacLotBalance({ lot: { ...vacLotBalance().lot, id: "lot15", item: BCG_ITEM, lot_number: "BCG2610" } });
    vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition({ item: BCG_ITEM, lots: [ bcgLot ] }) ], next_before: null });
    vi.mocked(api.listVials).mockResolvedValue([ vial(), vial({ id: "v2", expires_at: "2026-11-02T18:00:00-03:00" }) ]);
    vi.mocked(api.recordImmunization)
      .mockRejectedValueOnce(refuse([ { path: "applied_at", error: "interval_too_short" }, { path: "vial_id", error: "vial_expired" } ]))
      .mockResolvedValueOnce(immunization());
    mount(cardItem({ immunobiological: { code: BCG.code, name: BCG.name }, dose: { code: "D1", name: "1ª dose" } }));
    fireEvent.change(await screen.findByLabelText("Estratégia"), { target: { value: "1" } });
    fireEvent.change(screen.getByLabelText("Grupo de atendimento"), { target: { value: "1" } });
    await waitFor(() => expect((screen.getByLabelText("Lote") as HTMLSelectElement).options.length).toBe(2));
    fireEvent.change(screen.getByLabelText("Lote"), { target: { value: "lot15" } });
    fireEvent.change(await screen.findByLabelText("Frasco aberto"), { target: { value: "v1" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar dose" }));
    expect(await screen.findByText("intervalo menor que o mínimo desde a dose anterior")).not.toBeNull();
    expect(screen.getByText("o frasco venceu — descarte e abra outro")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Motivo para aplicar mesmo assim"), { target: { value: "curto" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar dose" }));
    expect(screen.getByText("o motivo precisa ter de 10 a 500 caracteres")).not.toBeNull();
    expect(screen.getByText("BCG é multidose: escolha o frasco aberto ou abra um frasco")).not.toBeNull();
    expect(api.recordImmunization).toHaveBeenCalledTimes(1);
    fireEvent.change(screen.getByLabelText("Motivo para aplicar mesmo assim"), { target: { value: "mãe adiantou a dose por viagem" } });
    fireEvent.change(screen.getByLabelText("Frasco aberto"), { target: { value: "v2" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar dose" }));
    await waitFor(() => expect(api.recordImmunization).toHaveBeenLastCalledWith("r1",
      expect.objectContaining({ vial_id: "v2", override_reason: "mãe adiantou a dose por viagem" })));
  });

  it("multidose sem frasco aberto: abre o frasco do lote e o escolhe", async () => {
    const bcgLot = vacLotBalance({ lot: { ...vacLotBalance().lot, id: "lot15", item: BCG_ITEM, lot_number: "BCG2610" } });
    vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition({ item: BCG_ITEM, lots: [ bcgLot ] }) ], next_before: null });
    vi.mocked(api.openVial).mockResolvedValue(vial({ id: "v7", doses_used: 0, doses_left: 10 }));
    mount(cardItem({ immunobiological: { code: BCG.code, name: BCG.name }, dose: { code: "D1", name: "1ª dose" } }));
    await waitFor(() => expect((screen.getByLabelText("Lote") as HTMLSelectElement).options.length).toBe(2));
    fireEvent.change(screen.getByLabelText("Lote"), { target: { value: "lot15" } });
    fireEvent.click(await screen.findByRole("button", { name: "Abrir frasco deste lote" }));
    await waitFor(() => expect(api.openVial).toHaveBeenCalledWith("r1", "lot15"));
    await waitFor(() => expect((screen.getByLabelText("Frasco aberto") as HTMLSelectElement).value).toBe("v7"));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/DoseForm.test.tsx`
Expected: FAIL — `DoseForm` não existe.

- [ ] **Step 3: Implemente `DoseForm`** como no Interfaces: `<form aria-label="Registrar dose" onSubmit>`; posição do local por `useInfiniteQuery([ PH_POSITIONS_KEY, room.id, "dose" ], listPositions({ locationId: room.id }))` com `STATIC_QUERY`; frascos por `useQuery([ VAC_VIALS_KEY, room.room_id, "open" ], listVials(room.room_id, "open"))` só quando a vacina é multidose; `busy` por `useRef` (o segundo `submit` antes do `setState` também é barrado). A frase `"<vacina> é multidose: escolha o frasco aberto ou abra um frasco"` (frasco limpo depois de `vial_expired`) é a de `doseProblems` para `vial_id` vazio em multidose. Nada do formulário vai a storage, URL ou `console`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/DoseForm.test.tsx src/modules/vaccination/CardView.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/vaccination/DoseForm.tsx src/modules/vaccination/DoseForm.test.tsx src/modules/VaccinationRoom.tsx
$G commit -m "feat: record the administered dose with lot, vial and justified warnings

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Transcrição de caderneta, cancelamento e substituição (F-29.3)

**Files:**
- Create: `src/modules/vaccination/TranscriptionForm.tsx`, `src/modules/vaccination/CancelDose.tsx`
- Modify: `src/modules/VaccinationRoom.tsx` (abre a transcrição), `src/modules/vaccination/ImmunizationRow.tsx` (botão "Cancelar dose")
- Test: `src/modules/vaccination/Transcription.test.tsx`

**Interfaces:**
- Consumes: `recordTranscriptions`, `cancelImmunization` (Task 1); `newTranscriptionLine`, `transcriptionProblems`, `transcriptionBody`, `reasonProblem`, `vaccinationRefusals`, `doseProblems`, `doseBody`, `newDoseDraft` (Task 2); `DoseForm` (Task 7, os campos reusados no modo substituição); `SensitiveAction`.
- Produces:
  - `TranscriptionForm({ room, card, catalog, today, onDone, onCancel }: { room: VaccineLocation; card: VaccinationCard; catalog: VaccinationCatalog; today: string; onDone(c: VaccinationCard): void; onCancel(): void })` — uma linha por dose (vacina, dose, "Data da aplicação" `date` com `max=today` e `min=birth_date`, lote, fabricante, local — opcionais), "Acrescentar dose", "Remover"; caixa "Esta caderneta está conferida depois desta transcrição" (marcada por padrão quando a caderneta não está conferida; `close_check`); envio único com `transcriptionProblems` e as recusas do api na linha pelo índice `items[i]`; sucesso devolve o `card` da resposta (sem nova leitura);
  - `CancelDose({ imm, room, patient, catalog, now, onDone, onCancel, onGoToSecurity })` — só quando `imm.can_cancel` e `status === "active"`; "Motivo do cancelamento" (10–500), caixa "Registrar a dose correta agora" que mostra os campos do `DoseForm` em modo `replacement` (sem enviar sozinho); "Revisar" confere tudo e abre o `SensitiveAction` (step-up) que chama `cancelImmunization(imm.id, { reason, replacement? })`; recusa com `problems` de path `replacement.*` aparece nos campos da substituição e nada é cancelado (D17); sucesso mostra "Dose cancelada." ou "Dose cancelada e substituída." e relê a caderneta.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/vaccination/Transcription.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { CATALOG29, NOW29, ROOM, TODAY29, card, cardPatient, immunization, vacPosition, vacUser } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { TranscriptionForm } from "./TranscriptionForm";
import { CancelDose } from "./CancelDose";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), recordTranscriptions: vi.fn(),
  cancelImmunization: vi.fn(), listPositions: vi.fn(), listVials: vi.fn() }));
beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
  vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition() ], next_before: null });
  vi.mocked(api.listVials).mockResolvedValue([]);
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("transcrição (F-29.3)", () => {
  it("duas doses, fecha a conferência, devolve a caderneta da resposta", async () => {
    const after = card({ checked: { mode: "transcribed", at: "2026-11-02T10:00:00-03:00", by_name: "Ana" } });
    vi.mocked(api.recordTranscriptions).mockResolvedValue({ immunizations: [], card: after });
    const onDone = vi.fn();
    renderWithProviders(<TranscriptionForm room={ROOM} card={card({ checked: null })} catalog={CATALOG29} today={TODAY29} onDone={onDone} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar dose" }));
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar dose" }));
    const [ l1, l2 ] = screen.getAllByRole("group", { name: /^Dose transcrita/ });
    fireEvent.change(within(l1).getByLabelText("Vacina"), { target: { value: "15" } });
    fireEvent.change(within(l1).getByLabelText("Dose"), { target: { value: "D1" } });
    fireEvent.change(within(l1).getByLabelText("Data da aplicação"), { target: { value: "2026-07-02" } });
    fireEvent.change(within(l2).getByLabelText("Vacina"), { target: { value: "42" } });
    fireEvent.change(within(l2).getByLabelText("Dose"), { target: { value: "D1" } });
    fireEvent.change(within(l2).getByLabelText("Data da aplicação"), { target: { value: "2026-09-01" } });
    fireEvent.change(within(l2).getByLabelText("Lote (opcional)"), { target: { value: "PNT2608" } });
    expect((screen.getByLabelText("Esta caderneta está conferida depois desta transcrição") as HTMLInputElement).checked).toBe(true);
    fireEvent.click(screen.getByRole("button", { name: "Registrar transcrição" }));
    await waitFor(() => expect(api.recordTranscriptions).toHaveBeenCalledWith("r1", { patient_id: "pa1", close_check: true, items: [
      { immunobiological_code: "15", dose_code: "D1", applied_on: "2026-07-02" },
      { immunobiological_code: "42", dose_code: "D1", applied_on: "2026-09-01", lot_number: "PNT2608" } ] }));
    await waitFor(() => expect(onDone).toHaveBeenCalledWith(after));
  });

  it("data antes do nascimento: recusa na linha, sem enviar", async () => {
    renderWithProviders(<TranscriptionForm room={ROOM} card={card({ checked: null, patient: cardPatient({ birth_date: "2026-07-01" }) })}
      catalog={CATALOG29} today={TODAY29} onDone={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar dose" }));
    const line = screen.getByRole("group", { name: /^Dose transcrita/ });
    fireEvent.change(within(line).getByLabelText("Vacina"), { target: { value: "15" } });
    fireEvent.change(within(line).getByLabelText("Dose"), { target: { value: "D1" } });
    fireEvent.change(within(line).getByLabelText("Data da aplicação"), { target: { value: "2026-06-01" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar transcrição" }));
    expect(within(line).getByText("BCG: a data é antes do nascimento")).not.toBeNull();
    expect(api.recordTranscriptions).not.toHaveBeenCalled();
  });
});

describe("cancelamento (F-29.3)", () => {
  const mount = (imm = immunization()) => renderWithProviders(<CancelDose imm={imm} room={ROOM} patient={cardPatient()} catalog={CATALOG29} now={NOW29}
    onDone={vi.fn()} onCancel={vi.fn()} />);

  it("sem permissão de cancelar, nada é oferecido", () => {
    mount(immunization({ can_cancel: false }));
    expect(screen.queryByLabelText("Motivo do cancelamento")).toBeNull();
    expect(screen.getByText("só quem registrou ou o responsável pela sala cancela")).not.toBeNull();
  });

  it("motivo curto recusado; com substituição, uma chamada com o par; step-up", async () => {
    vi.mocked(api.cancelImmunization).mockResolvedValue({ cancelled: immunization({ status: "cancelled" }), replacement: immunization({ id: "im8" }) });
    mount();
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "errado" } });
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    expect(screen.getByText("o motivo precisa ter de 10 a 500 caracteres")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "lote errado digitado na sala" } });
    fireEvent.click(screen.getByLabelText("Registrar a dose correta agora"));
    fireEvent.change(await screen.findByLabelText("Estratégia"), { target: { value: "1" } });
    fireEvent.change(screen.getByLabelText("Grupo de atendimento"), { target: { value: "1" } });
    await waitFor(() => expect((screen.getByLabelText("Lote") as HTMLSelectElement).options.length).toBe(2));
    fireEvent.change(screen.getByLabelText("Lote"), { target: { value: "lot9" } });
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar e substituir" }));
    await waitFor(() => expect(api.cancelImmunization).toHaveBeenCalledWith("im1", { reason: "lote errado digitado na sala",
      replacement: expect.objectContaining({ patient_id: "pa1", immunobiological_code: "42", dose_code: "D1", lot_id: "lot9" }) }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/Transcription.test.tsx`
Expected: FAIL — as peças não existem.

- [ ] **Step 3: Implemente** `TranscriptionForm` (cada linha um `<fieldset>` com `<legend>Dose transcrita N</legend>`; datas `date`; recusas do api por `problemItemIndex`) e `CancelDose`. Para a substituição, extraia do `DoseForm` a parte de campos como `DoseFields({ room, patient, catalog, draft, onChange, problems, now })` exportado do mesmo arquivo — o `DoseForm` passa a montá-la (os testes da Task 7 continuam verdes). O botão de confirmação do `SensitiveAction` é "Cancelar dose" sem substituição e "Cancelar e substituir" com ela. No `ImmunizationRow`, o botão "Cancelar dose — <vacina> <dose>" aparece só com `can_cancel` e dose ativa.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/vaccination/TranscriptionForm.tsx src/modules/vaccination/CancelDose.tsx src/modules/vaccination/Transcription.test.tsx \
  src/modules/vaccination/DoseForm.tsx src/modules/vaccination/ImmunizationRow.tsx src/modules/VaccinationRoom.tsx
$G commit -m "feat: add card transcription and dose cancellation with replacement

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Situação do envio à RNDS por dose e pendências da sala (F-29.4)

**Files:**
- Create: `src/modules/vaccination/RndsStatusTag.tsx`
- Modify: `src/modules/vaccination/ImmunizationRow.tsx`, `src/modules/VaccinationPending.tsx`
- Test: `src/modules/VaccinationPending.test.tsx`

**Interfaces:**
- Consumes: `listRndsPending`, `getVaccinationCard` (Task 1); `rndsStatusLabel`, `rndsStatusTone`, `blockReasonLabel`, `VAC_PENDING_KEY`, `STATIC_QUERY`, `vaccinationError` (Task 2); `VaccineLocationPick` (Task 3); `CardView` (Task 6).
- Produces:
  - `RndsStatusTag({ rnds }: { rnds: ImmunizationRnds })` — `Tag` com `rndsStatusLabel`; abaixo, em texto pequeno, o motivo (`blockReasonLabel`) ou o `error` da RNDS ("RNDS: <texto>", como veio — já limpo pelo api) e, aceita, "aceita em <data>"; transcrição e cancelamento usam o mesmo componente;
  - `VaccinationPending({ now?, onGoToSecurity? })`: local onde `vaccinate` → `useInfiniteQuery([ VAC_PENDING_KEY, room_id ], listRndsPending)` com `STATIC_QUERY` → `DataTable` (vacina, aplicada em, situação, motivo, "Abrir caderneta") sem nome; "Atualizar" relê; "Carregar mais" segue `next_before`; "Abrir caderneta" chama `getVaccinationCard(room_id, { patient_id })` por mutação e mostra o `CardView` abaixo da lista, com "Fechar caderneta" que a descarta; o texto fixo `PENDING_HINT = "Corrija o cadastro (CNS no CNES, CPF ou CNS do cidadão) e a dose volta para a fila sozinha. Doses só ficam aqui enquanto não forem aceitas."`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/VaccinationPending.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../test/campaignFixtures";
import { CATALOG29, NOW29, ROOM, card, immunization, pendingItem, vacUser } from "../test/vaccinationFixtures";
import * as api from "../lib/api";
import { VaccinationPending } from "./VaccinationPending";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyVaccineLocations: vi.fn(),
  listRndsPending: vi.fn(), getVaccinationCard: vi.fn(), getVaccinationCatalog: vi.fn() }));
beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
  vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ ROOM ]);
  vi.mocked(api.getVaccinationCatalog).mockResolvedValue(CATALOG29);
  vi.mocked(api.listRndsPending).mockResolvedValueOnce({ items: [ pendingItem(), pendingItem({ immunization_id: "im3",
    rnds: { status: "rejected", sent_at: "2026-11-01T09:01:00-03:00", accepted_at: null, block_reason: null, error: "lote não reconhecido" } }) ], next_before: "c1" })
    .mockResolvedValueOnce({ items: [ pendingItem({ immunization_id: "im4", rnds: { status: "pending", sent_at: null, accepted_at: null, block_reason: null, error: null } }) ], next_before: null });
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("pendências do envio (F-29.4)", () => {
  it("lista sem nome, motivo em português, paginação e sem releitura sozinha", async () => {
    renderWithProviders(<VaccinationPending now={NOW29} />);
    expect(await screen.findByText("o CNS de quem vacinou não está no CNES importado da cidade")).not.toBeNull();
    expect(screen.getByText("RNDS: lote não reconhecido")).not.toBeNull();
    expect(screen.queryByText(/Joana/)).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
    await waitFor(() => expect(api.listRndsPending).toHaveBeenLastCalledWith("r1", "c1"));
    expect(await screen.findByText("na fila de envio")).not.toBeNull();
    expect(api.listRndsPending).toHaveBeenCalledTimes(2);
  });

  it("abrir caderneta lê no clique e mostra a situação de cada dose", async () => {
    vi.mocked(api.getVaccinationCard).mockResolvedValue(card({ immunizations: [ immunization({
      rnds: { status: "blocked", sent_at: null, accepted_at: null, block_reason: "patient_without_cns_or_cpf", error: null } }) ] }));
    renderWithProviders(<VaccinationPending now={NOW29} />);
    const rows = await screen.findAllByRole("button", { name: /^Abrir caderneta/ });
    expect(api.getVaccinationCard).not.toHaveBeenCalled();
    fireEvent.click(rows[0]);
    await waitFor(() => expect(api.getVaccinationCard).toHaveBeenCalledWith("r1", { patient_id: "pa1" }));
    const list = await screen.findByRole("region", { name: "Doses registradas" });
    expect(within(list).getByText("parada: cadastro a corrigir")).not.toBeNull();
    expect(within(list).getByText("o cidadão não tem CPF nem CNS no cadastro")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationPending.test.tsx`
Expected: FAIL — a tela é só o portão.

- [ ] **Step 3: Implemente** `RndsStatusTag`, coloque-o no `ImmunizationRow` (coluna "Envio à RNDS") e o corpo de `VaccinationPending`. O botão de cada linha tem o rótulo `Abrir caderneta — <vacina> de <data>` (sem nome). Nenhum `refetchInterval`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationPending.test.tsx src/modules/vaccination && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/vaccination/RndsStatusTag.tsx src/modules/vaccination/ImmunizationRow.tsx src/modules/VaccinationPending.tsx src/modules/VaccinationPending.test.tsx
$G commit -m "feat: show the RNDS delivery status per dose and the room pending list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Estoque de imunobiológicos com as peças do 27a (F-29.5)

**Files:**
- Modify: `src/modules/pharmacy/TransferForm.tsx` (prop `destinations?: PharmacyLocation[]`; identidade por `itemKey`), `src/modules/pharmacy/AdjustmentForm.tsx` (identidade por `itemKey`), `src/modules/PharmacyTransfers.tsx` (o `Body` vira `export function TransfersBody` com `canSend?` e `destinations?`), `src/modules/VaccinationStock.tsx`, `src/lib/vaccination.ts` (regras da entrada)
- Create: `src/modules/vaccination/VaccineReceiptForm.tsx`, `src/modules/vaccination/VaccinePositions.tsx`
- Test: `src/modules/VaccinationStock.test.tsx`

**Interfaces:**
- Consumes: `recordVaccineReceipt`, `recordVaccineOpening`, `listPositions`, `listTransfers`, `sendTransfer`, `receiveTransfer`, `recordAdjustment`, `listMyVaccineLocations`, `getVaccinationCatalog` (Tasks 1 e 27a); `itemKey`, `fmtQuantity`, `lotOptionLabel`, `PH_POSITIONS_KEY` (`pharmacy.ts`); `TransferForm`, `ReceiveForm`, `AdjustmentForm`, `PendingAdjustments`, `LotStatement` (27a); `VaccineLocationPick`, `locationAbilities` (Tasks 2–3).
- Produces:
  - em `src/lib/vaccination.ts`: `VaccineReceiptLine { key; immunobiologicalCode; manufacturerCode; lotNumber; expiresOn; vials }`; `newVaccineReceiptLine(key: string): VaccineReceiptLine`; `vaccineReceiptProblems(lines, ctx: { catalog: VaccinationCatalog; today: string; doc: { kind: "invoice" | "donation" | "opening"; number: string; supplier: string } }): Problem[]` (documento como no 27a; por linha: vacina, fabricante do catálogo daquela vacina, lote 1–40, validade ≥ hoje — "lote vencido — não entra no estoque", frascos inteiro 1–10000); `vaccineReceiptItems(lines, catalog): VaccineReceiptItemInput[]` (`quantity` = frascos × `doses_per_vial`, em texto; `package: { units_per_package: doses_per_vial, packages: frascos }`);
  - `VaccineReceiptForm({ location, catalog, mode, today, onDone }: { location: VaccineLocation; catalog: VaccinationCatalog; mode: "receipt" | "opening"; today: string; onDone(r: StockReceipt): void })` — uma linha por lote: "Vacina", "Fabricante" (só os do catálogo daquela vacina), "Lote", "Validade", "Frascos" com a conta "= N doses" ao lado; envio único; recusas do api na linha por `items[i]`;
  - `VaccinePositions({ location, today, onStatement? })` — posição do local em doses, um lote por linha: vacina, lote, fabricante, validade, saldo e etiquetas `bloqueado — aguarda a conduta` (down), `vencido` (down), `vence em breve` (warn); "Extrato — <lote>" abre o `LotStatement` do 27a;
  - `TransfersBody({ location, today, canSend, destinations }: { location: PharmacyLocation; today: string; canSend?: boolean; destinations?: PharmacyLocation[] })` (padrão de `canSend` = a regra de hoje `kind === "central" && keeper`);
  - `VaccinationStock({ today?, now?, onGoToSecurity? })`: local por `VaccineLocationPick` (onde a pessoa tem papel: `vaccinate` ou `keeper`; as salas com `roles: []` que o almoxarife recebe não entram) → na **rede de frio** (`keeper`): "Entrada e inventário inicial" (alternância "Entrada por nota ou doação" / "Inventário inicial"), "Transferências" (`TransfersBody` com `canSend` e `destinations` = salas ativas da lista de locais), "Posição", "Inventário e ajustes" (`AdjustmentForm` + `PendingAdjustments`); na **sala**: "Frascos abertos" (Task 11), "Posição", "Transferências" (só receber), e "Inventário e ajustes" só para o `room_lead`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/VaccinationStock.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../test/campaignFixtures";
import { BCG_ITEM, CATALOG29, COLD_CENTRAL, LEAD_ROOM, NOW29, PENTA_ITEM, ROOM, TODAY29, vacLotBalance, vacPosition, vacUser } from "../test/vaccinationFixtures";
import * as api from "../lib/api";
import { VaccinationStock } from "./VaccinationStock";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyVaccineLocations: vi.fn(),
  getVaccinationCatalog: vi.fn(), listPositions: vi.fn(), listTransfers: vi.fn(), sendTransfer: vi.fn(), recordVaccineReceipt: vi.fn(),
  listAdjustments: vi.fn(), listVials: vi.fn() }));
const lot = (id: string, item = PENTA_ITEM, over = {}) => vacLotBalance({ lot: { ...vacLotBalance().lot, id, item, lot_number: `L-${id}` }, ...over });

beforeEach(() => {
  vi.mocked(api.getVaccinationCatalog).mockResolvedValue(CATALOG29);
  vi.mocked(api.listTransfers).mockResolvedValue({ transfers: [], next_before: null });
  vi.mocked(api.listAdjustments).mockResolvedValue({ adjustments: [], next_before: null });
  vi.mocked(api.listVials).mockResolvedValue([]);
  vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition({ lots: [ lot("a"), lot("b", PENTA_ITEM, { blocked: true }) ] }),
    vacPosition({ item: BCG_ITEM, lots: [ lot("c", BCG_ITEM) ] }) ], next_before: null });
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

// O almoxarife da rede de frio recebe também as salas, com `roles: []` (D2): são os destinos da transferência.
function mount(loc = COLD_CENTRAL, roles = [ "stock_keeper" ]) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser(roles));
  vi.mocked(api.listMyVaccineLocations).mockResolvedValue(loc.kind === "cold_chain_central" ? [ loc, { ...ROOM, roles: [] } ] : [ loc ]);
  return renderWithProviders(<VaccinationStock today={TODAY29} now={NOW29} />);
}

describe("estoque de imunobiológicos (F-29.5)", () => {
  it("entrada na rede de frio: frascos × doses, fabricante por código, validade conferida", async () => {
    vi.mocked(api.recordVaccineReceipt).mockResolvedValue({ id: "rc1", kind: "invoice", location: COLD_CENTRAL, items: [], at: "2026-11-02T10:00:00-03:00" });
    mount();
    const entry = await screen.findByRole("region", { name: "Entrada e inventário inicial" });
    fireEvent.change(within(entry).getByLabelText("Número do documento"), { target: { value: "NF 88" } });
    fireEvent.change(within(entry).getByLabelText("Fornecedor ou doador"), { target: { value: "SESA-PR" } });
    fireEvent.click(within(entry).getByRole("button", { name: "Acrescentar lote" }));
    fireEvent.change(within(entry).getByLabelText("Vacina"), { target: { value: "15" } });
    expect([ ...(within(entry).getByLabelText("Fabricante") as HTMLSelectElement).options ].map((o) => o.value)).toEqual([ "", "FAP" ]);
    fireEvent.change(within(entry).getByLabelText("Fabricante"), { target: { value: "FAP" } });
    fireEvent.change(within(entry).getByLabelText("Lote"), { target: { value: "BCG2611" } });
    fireEvent.change(within(entry).getByLabelText("Validade"), { target: { value: "2026-11-01" } });
    fireEvent.change(within(entry).getByLabelText("Frascos"), { target: { value: "5" } });
    expect(within(entry).getByText("= 50 doses")).not.toBeNull();
    fireEvent.click(within(entry).getByRole("button", { name: "Registrar entrada" }));
    expect(within(entry).getByText("BCG: lote vencido — não entra no estoque")).not.toBeNull();
    fireEvent.change(within(entry).getByLabelText("Validade"), { target: { value: "2027-05-31" } });
    fireEvent.click(within(entry).getByRole("button", { name: "Registrar entrada" }));
    await waitFor(() => expect(api.recordVaccineReceipt).toHaveBeenCalledWith({ location_id: "l9",
      document: { kind: "invoice", number: "NF 88", supplier: "SESA-PR" },
      items: [ { immunobiological_code: "15", manufacturer_code: "FAP", lot_number: "BCG2611", expires_on: "2027-05-31", quantity: "50",
        package: { units_per_package: 10, packages: 5 } } ] }));
  });

  it("transferência: destino são as salas; vacinas diferentes não se confundem (itemKey)", async () => {
    mount();
    const transfers = await screen.findByRole("region", { name: "Transferências" });
    const dest = within(transfers).getByLabelText(/Destino/) as HTMLSelectElement;
    expect([ ...dest.options ].map((o) => o.value)).toEqual([ "", "l5" ]);
    fireEvent.click(await within(transfers).findByRole("button", { name: "Acrescentar BCG" }));
    expect(within(transfers).getByText(/L-c/)).not.toBeNull();
    expect(within(transfers).queryByText(/L-a/)).toBeNull();
  });

  it("sala: lote bloqueado na posição; vacinador sem ajustes, responsável com", async () => {
    mount(ROOM, [ "health_professional" ]);
    const pos = await screen.findByRole("region", { name: "Posição" });
    expect(within(pos).getByText("bloqueado — aguarda a conduta")).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Inventário e ajustes" })).toBeNull();
    expect(within(screen.getByRole("region", { name: "Transferências" })).queryByLabelText(/Destino/)).toBeNull();
    cleanup();
    mount(LEAD_ROOM, [ "health_professional" ]);
    expect(await screen.findByRole("region", { name: "Inventário e ajustes" })).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationStock.test.tsx`
Expected: FAIL — a tela é só o portão; `TransferForm` não aceita destinos.

- [ ] **Step 3: Implemente** as regras da entrada em `vaccination.ts`, `VaccineReceiptForm`, `VaccinePositions`, a troca de `catalog_item_id` por `itemKey(...)` nos quatro pontos do `TransferForm` e do `AdjustmentForm` anotados na Task 0, a prop `destinations` (quando vem, a consulta a `listMyPharmacyLocations` não roda), o `TransfersBody` exportado (o `PharmacyTransfers` passa a montá-lo sem props novas) e o corpo de `VaccinationStock` com cada bloco em `<section aria-label="<título>">`. Os rótulos de quantidade das peças do 27a usam `dispensing_unit`, que no imunobiológico é "dose" — nada a mudar.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationStock.test.tsx src/modules/PharmacyTransfers.test.tsx src/modules/PharmacyAdjustments.test.tsx src/modules/PharmacyStock.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS (as telas da farmácia continuam iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/pharmacy/TransferForm.tsx src/modules/pharmacy/AdjustmentForm.tsx src/modules/PharmacyTransfers.tsx src/modules/VaccinationStock.tsx \
  src/modules/VaccinationStock.test.tsx src/modules/vaccination/VaccineReceiptForm.tsx src/modules/vaccination/VaccinePositions.tsx src/lib/vaccination.ts
$G commit -m "feat: manage immunobiological stock with the pharmacy ledger screens

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Frascos abertos, descarte e resumo mensal (F-29.5)

**Files:**
- Create: `src/modules/vaccination/VialsPanel.tsx`
- Modify: `src/modules/VaccinationStock.tsx` (monta o `VialsPanel` na sala), `src/modules/VaccinationSummary.tsx`
- Test: `src/modules/vaccination/Vials.test.tsx`

**Interfaces:**
- Consumes: `listVials`, `openVial`, `closeOutVial`, `listPositions`, `getMonthlySummary` (Task 1); `vialRemaining`, `vialStatusLabel`, `reasonProblem`, `monthProblem`, `currentMonth`, `VAC_VIALS_KEY`, `VAC_SUMMARY_KEY`, `STATIC_QUERY`, `vaccinationError` (Task 2); `VaccineLocationPick` (Task 3).
- Produces:
  - `VialsPanel({ room, now, lead }: { room: VaccineLocation; now: Date; lead: boolean })` — `<section aria-label="Frascos abertos">`: tabela (vacina, lote, aberto em, "restam N de M doses", tempo restante por `vialRemaining`, `Tag` "vencido" quando for); "Abrir frasco" com o lote (só lotes multidose disponíveis da posição); só para o `room_lead`, "Descartar — <lote>" pede "Motivo do descarte" (10–500) e confirma em dois passos; depois de cada escrita relê `VAC_VIALS_KEY` e `PH_POSITIONS_KEY`; frase fixa `VIAL_LOSS_NOTE = "As doses que sobram no frasco vencido ou descartado entram como perda técnica no resumo do mês."`;
  - `VaccinationSummary({ today? })`: local (sala ou rede de frio onde a pessoa tem papel), "Mês" (`<input type="month">`, padrão `currentMonth(today)`), "Ver resumo" confere `monthProblem` e lê `getMonthlySummary` (`{ roomId }` na sala, `{ central: true }` na rede de frio) com `STATIC_QUERY`; tabela com as colunas do SIES/SIPNI: vacina, lote, saldo inicial, entradas, transferências recebidas, enviadas, frascos abertos, doses aplicadas, perda técnica, perda física (soma, com o detalhe por motivo em texto), saldo final; linha "Total"; números com separador de milhar.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/vaccination/Vials.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { BCG_ITEM, COLD_CENTRAL, LEAD_ROOM, NOW29, ROOM, TODAY29, monthly, vacLotBalance, vacPosition, vacUser, vial } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { VialsPanel } from "./VialsPanel";
import { VaccinationSummary } from "../VaccinationSummary";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyVaccineLocations: vi.fn(),
  listVials: vi.fn(), openVial: vi.fn(), closeOutVial: vi.fn(), listPositions: vi.fn(), getMonthlySummary: vi.fn() }));
beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
  vi.mocked(api.listVials).mockResolvedValue([ vial(), vial({ id: "v2", expires_at: "2026-11-02T09:00:00-03:00", status: "expired" }) ]);
  vi.mocked(api.listPositions).mockResolvedValue({ positions: [ vacPosition({ item: BCG_ITEM, lots: [ vacLotBalance({ lot: { ...vacLotBalance().lot, id: "lot15",
    item: BCG_ITEM, lot_number: "BCG2610" } }) ] }) ], next_before: null });
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("frascos (F-29.5)", () => {
  it("tempo restante e doses; vacinador não descarta", async () => {
    renderWithProviders(<VialsPanel room={ROOM} now={NOW29} lead={false} />);
    expect(await screen.findByText("vence em 4 h 0 min")).not.toBeNull();
    expect(screen.getByText("restam 7 de 10 doses")).not.toBeNull();
    expect(screen.getAllByText("vencido").length).toBeGreaterThan(0);
    expect(screen.queryByRole("button", { name: /^Descartar/ })).toBeNull();
  });

  it("responsável descarta com motivo e confirmação; relê", async () => {
    vi.mocked(api.closeOutVial).mockResolvedValue(vial({ status: "closed_out" }));
    renderWithProviders(<VialsPanel room={LEAD_ROOM} now={NOW29} lead />);
    fireEvent.click((await screen.findAllByRole("button", { name: "Descartar — BCG2610" }))[0]);
    fireEvent.change(screen.getByLabelText("Motivo do descarte"), { target: { value: "caiu" } });
    fireEvent.click(screen.getByRole("button", { name: "Descartar frasco" }));
    expect(screen.getByText("o motivo precisa ter de 10 a 500 caracteres")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Motivo do descarte"), { target: { value: "frasco caiu e trincou" } });
    fireEvent.click(screen.getByRole("button", { name: "Descartar frasco" }));
    fireEvent.click(screen.getByRole("button", { name: "Sim, descartar" }));
    await waitFor(() => expect(api.closeOutVial).toHaveBeenCalledWith("v1", "frasco caiu e trincou"));
    await waitFor(() => expect(api.listVials).toHaveBeenCalledTimes(2));
  });
});

describe("resumo mensal (F-29.5)", () => {
  it("sala e rede de frio; mês futuro recusado; colunas do SIES", async () => {
    vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ ROOM, { ...COLD_CENTRAL, roles: [ "stock_keeper" ] } ]);
    vi.mocked(api.getMonthlySummary).mockResolvedValue(monthly());
    renderWithProviders(<VaccinationSummary today={TODAY29} />);
    fireEvent.change(await screen.findByLabelText("Mês"), { target: { value: "2026-12" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver resumo" }));
    expect(screen.getByText("o mês não pode ser futuro")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Mês"), { target: { value: "2026-10" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver resumo" }));
    await waitFor(() => expect(api.getMonthlySummary).toHaveBeenCalledWith({ roomId: "r1" }, "2026-10"));
    const table = await screen.findByRole("region", { name: "Movimentação do mês" });
    expect(within(table).getByText("BCG2610")).not.toBeNull();
    expect(within(table).getByText("broken: 10")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Local"), { target: { value: "l9" } });
    fireEvent.change(await screen.findByLabelText("Mês"), { target: { value: "2026-10" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver resumo" }));
    await waitFor(() => expect(api.getMonthlySummary).toHaveBeenLastCalledWith({ central: true }, "2026-10"));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/Vials.test.tsx`
Expected: FAIL — as peças não existem.

- [ ] **Step 3: Implemente** `VialsPanel` e `VaccinationSummary` como no Interfaces. Motivo de perda física aparece como `<motivo do api>: N` (o código cru: o contrato não fixa a lista — D8). O resumo só tem contagens: nenhuma coluna de pessoa.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/Vials.test.tsx src/modules/VaccinationStock.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/vaccination/VialsPanel.tsx src/modules/vaccination/Vials.test.tsx src/modules/VaccinationStock.tsx src/modules/VaccinationSummary.tsx
$G commit -m "feat: add open vials, vial close-out and the monthly stock summary

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Cadeia de frio — equipamentos, leituras, excursão e conduta (F-29.6)

**Files:**
- Modify: `src/modules/VaccinationColdChain.tsx`
- Create: `src/modules/vaccination/EquipmentForm.tsx`, `src/modules/vaccination/ReadingForm.tsx`, `src/modules/vaccination/ExcursionDecision.tsx`
- Test: `src/modules/VaccinationColdChain.test.tsx`

**Interfaces:**
- Consumes: `getColdChainStatus`, `listEquipments`, `createEquipment`, `updateEquipment`, `recordReading`, `decideExcursion` (Task 1); `parseCelsius`, `readingProblems`, `readingBody`, `outOfRange`, `fmtCelsius`, `equipmentKindLabel`, `reasonProblem`, `VAC_COLD_KEY`, `VAC_EQUIPMENTS_KEY`, `STATIC_QUERY`, `vaccinationRefusals` (Task 2); `adjustmentSavedText` (27a); `SensitiveAction`; `VaccineLocationPick`, `locationAbilities` (Tasks 2–3).
- Produces:
  - `VaccinationColdChain({ now?, onGoToSecurity? })`: local (onde a pessoa tem papel: `vaccinate` ou `keeper`; as salas com `roles: []` que o almoxarife recebe não entram) → situação por `getColdChainStatus` (`STATIC_QUERY`; "Atualizar" relê): um cartão por equipamento com tipo, faixa ("+2 a +8 °C"), última leitura e "leituras de hoje: N de 2" (`Tag` warn com menos de 2 depois do meio-dia; o texto diz o número, nunca só a cor); botão "Registrar temperatura — <equipamento>" (qualquer da equipe); "Excursões abertas" com a conduta (`lead` na sala, `keeper` na central); "Equipamentos" com "Novo equipamento"/"Editar" (`lead`/`keeper`);
  - `EquipmentForm({ locationId, initial, onSaved, onCancel })` — nome 3–120, tipo, mínima e máxima (padrão 2 e 8; mínima < máxima), ativo;
  - `ReadingForm({ equipment, now, onSaved, onCancel })` — "Hora da leitura" (`datetime-local`, padrão agora), "Temperatura atual", "Mínima", "Máxima" (vírgula ou ponto); confere `readingProblems`; fora da faixa do equipamento, antes de enviar mostra `OUT_OF_RANGE_WARNING = "Fora de +2 a +8 °C: ao salvar, os lotes deste local ficam bloqueados até a conduta."` (com a faixa real do equipamento) e o botão vira "Salvar e bloquear os lotes"; resposta com `excursion_id` → "Excursão aberta: N lotes bloqueados." e a conduta abre;
  - `ExcursionDecision({ excursion, onDone, onGoToSecurity })` — uma linha por lote (vacina, lote, saldo): "Usar" ou "Dar perda" e "Motivo" (10–500); "Revisar" confere todas as linhas (cada lote com decisão e motivo) e abre o `SensitiveAction` (step-up) "Registrar conduta"; o retorno com `adjustments` pendentes mostra `adjustmentSavedText("loss", status)` por lote; `excursion_closed` → frase e "Atualizar".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/VaccinationColdChain.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../test/campaignFixtures";
import { LEAD_ROOM, NOW29, ROOM, coldStatus, equipment, vacUser } from "../test/vaccinationFixtures";
import * as api from "../lib/api";
import { VaccinationColdChain } from "./VaccinationColdChain";

vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listMyVaccineLocations: vi.fn(),
  getColdChainStatus: vi.fn(), listEquipments: vi.fn(), recordReading: vi.fn(), decideExcursion: vi.fn() }));
const reading = { id: "rd1", read_at: "2026-11-02T08:00:00-03:00", current_c: 8.5, min_c: 3, max_c: 9, out_of_range: true, by: { name: "Ana" } };
const excursion = { id: "ex1", equipment_id: "eq1", reading, opened_at: "2026-11-02T08:00:00-03:00",
  lots: [ { lot_id: "lot9", number: "PNT2611", immunobiological: { code: "42", name: "Penta (DTP/HepB/Hib)" }, balance: "20" },
    { lot_id: "lot15", number: "BCG2610", immunobiological: { code: "15", name: "BCG" }, balance: "40" } ] };

beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
  vi.mocked(api.listEquipments).mockResolvedValue([ equipment() ]);
  vi.mocked(api.getColdChainStatus).mockResolvedValue(coldStatus());
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("cadeia de frio (F-29.6)", () => {
  it("leitura fora da faixa abre a conduta", async () => {
    vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ LEAD_ROOM ]);
    vi.mocked(api.recordReading).mockResolvedValue({ reading, excursion_id: "ex1", blocked_lots: 2 });
    renderWithProviders(<VaccinationColdChain now={NOW29} />);
    expect(await screen.findByText("leituras de hoje: 0 de 2")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Registrar temperatura — Geladeira 1" }));
    fireEvent.change(screen.getByLabelText("Hora da leitura"), { target: { value: "2026-11-02T08:00" } });
    fireEvent.change(screen.getByLabelText("Temperatura atual"), { target: { value: "8,5" } });
    fireEvent.change(screen.getByLabelText("Mínima"), { target: { value: "3" } });
    fireEvent.change(screen.getByLabelText("Máxima"), { target: { value: "9" } });
    expect(screen.getByText("Fora de +2 a +8 °C: ao salvar, os lotes deste local ficam bloqueados até a conduta.")).not.toBeNull();
    vi.mocked(api.getColdChainStatus).mockResolvedValue(coldStatus({ open_excursions: [ excursion ] }));
    fireEvent.click(screen.getByRole("button", { name: "Salvar e bloquear os lotes" }));
    await waitFor(() => expect(api.recordReading).toHaveBeenCalledWith("eq1", { read_at: "2026-11-02T11:00:00.000Z", current_c: 8.5, min_c: 3, max_c: 9 }));
    expect(await screen.findByText("Excursão aberta: 2 lotes bloqueados.")).not.toBeNull();
    expect(await screen.findByRole("region", { name: "Conduta — Geladeira 1" })).not.toBeNull();
  });

  it("mínima acima da atual e duas casas: recusa sem enviar", async () => {
    vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ ROOM ]);
    renderWithProviders(<VaccinationColdChain now={NOW29} />);
    fireEvent.click(await screen.findByRole("button", { name: "Registrar temperatura — Geladeira 1" }));
    fireEvent.change(screen.getByLabelText("Temperatura atual"), { target: { value: "4,55" } });
    fireEvent.change(screen.getByLabelText("Mínima"), { target: { value: "5" } });
    fireEvent.change(screen.getByLabelText("Máxima"), { target: { value: "6" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar leitura" }));
    expect(screen.getByText("informe a temperatura atual (ex.: 4,5)")).not.toBeNull();
    expect(api.recordReading).not.toHaveBeenCalled();
  });

  it("conduta: todas as linhas conferidas; vacinador não decide", async () => {
    vi.mocked(api.getColdChainStatus).mockResolvedValue(coldStatus({ open_excursions: [ excursion ] }));
    vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ ROOM ]);
    renderWithProviders(<VaccinationColdChain now={NOW29} />);
    expect(await screen.findByText(/aguarda a conduta do responsável/)).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Revisar" })).toBeNull();
    cleanup();
    vi.mocked(api.listMyVaccineLocations).mockResolvedValue([ LEAD_ROOM ]);
    vi.mocked(api.decideExcursion).mockResolvedValue({ excursion_id: "ex1", adjustments: [] });
    renderWithProviders(<VaccinationColdChain now={NOW29} />);
    const box = await screen.findByRole("region", { name: "Conduta — Geladeira 1" });
    const [ penta, bcg ] = within(box).getAllByRole("group");
    fireEvent.click(within(penta).getByLabelText("Usar"));
    fireEvent.change(within(penta).getByLabelText("Motivo"), { target: { value: "laudo do fabricante permite uso" } });
    fireEvent.click(within(box).getByRole("button", { name: "Revisar" }));
    expect(within(bcg).getByText("BCG2610: escolha usar ou dar perda")).not.toBeNull();
    fireEvent.click(within(bcg).getByLabelText("Dar perda"));
    fireEvent.change(within(bcg).getByLabelText("Motivo"), { target: { value: "exposição acima do tolerado" } });
    fireEvent.click(within(box).getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Registrar conduta" }));
    await waitFor(() => expect(api.decideExcursion).toHaveBeenCalledWith("ex1", [
      { lot_id: "lot9", decision: "use", reason: "laudo do fabricante permite uso" },
      { lot_id: "lot15", decision: "discard", reason: "exposição acima do tolerado" } ]));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationColdChain.test.tsx`
Expected: FAIL — a tela é só o portão.

- [ ] **Step 3: Implemente** as peças como no Interfaces. A excursão sem papel de conduta mostra `"Lotes bloqueados — aguarda a conduta do responsável pela sala."` (na central: "…do almoxarife da rede de frio."). Cada lote da conduta é um `<fieldset>` (role `group`) com dois rádios e o motivo. Depois do registro de leitura e da conduta, relê `VAC_COLD_KEY` e `PH_POSITIONS_KEY` (o bloqueio muda a posição e o registro da dose).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/VaccinationColdChain.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/VaccinationColdChain.tsx src/modules/VaccinationColdChain.test.tsx src/modules/vaccination/EquipmentForm.tsx \
  src/modules/vaccination/ReadingForm.tsx src/modules/vaccination/ExcursionDecision.tsx
$G commit -m "feat: add cold chain equipment, temperature readings and excursion decisions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Fonte `vaccination` na linha do tempo e "não conferido" nas linhas de cuidado (F-29.7, F-29.8)

**Files:**
- Modify: `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/timeline/TimelineDetail.tsx`, `src/lib/careLines.ts`, `src/lib/careLines.test.ts`
- Create: `src/modules/vaccination/ImmunizationDetail.tsx`
- Test: `src/lib/timeline.vaccination.test.ts`, `src/modules/timeline/PatientTimeline.vaccination.test.tsx`

**Interfaces:**
- Consumes: `getImmunization` (Task 1, D11); `ImmunizationRow`, `RndsStatusTag` (Tasks 6 e 9); `vaccinationError` (Task 2); `getTimeline` (existente).
- Produces:
  - em `timeline.ts`: `TIMELINE_KINDS` ganha `{ key: "vaccination", label: "Vacina" }`; `timelineKinds(examTracking, procedureBilling = false, pharmacy = false, vaccination = false)` (o filtro "Vacina" só com `vaccination`); `DetailType` e `OPENABLE` ganham `"immunization"` (abre só em contexto, como documento e escuta); `STATUS_LABEL` ganha `active: "aplicada"` (`cancelled` já é "cancelado");
  - `PatientTimeline` passa `hasFeature(user, "vaccination")` como quarto argumento;
  - `ImmunizationDetail({ id, onClose }: { id: string; onClose(): void })` — lê `getImmunization` por `useQuery` com `NOMINAL_QUERY` (lê ao abrir, trilha no api, `gcTime: 0`) e mostra o `ImmunizationRow` sem ações; `TimelineDetail` ganha `case "immunization"`;
  - em `careLines.ts`: `STATUS_LABEL.unchecked = "não conferido"`, `STATUS_TONE.unchecked = "neutral"`, e `practiceText` de `unchecked` = `"não conferido — a caderneta de vacina ainda não foi conferida na sala de vacina"`; `worstStatus` continua ignorando (`unchecked` não piora a pessoa nem entra em "busca ativa").

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/timeline.vaccination.test.ts
import { describe, expect, it } from "vitest";
import { TIMELINE_KINDS, detailTarget, statusLabel, timelineKindLabel, timelineKinds } from "./timeline";
import type { TimelineEntry } from "./api";

const entry = (over: Partial<TimelineEntry> = {}): TimelineEntry => ({ id: "e1", kind: "vaccination", occurred_at: "2026-09-01T09:30:00-03:00",
  unit: { id: "u1", name: "UBS Centro" }, professional: { name: "Téc. Ana Souza", cbo_code: "322205" }, title: "Vacina Penta — 1ª dose",
  summary: null, status: "active", detail_ref: { type: "immunization", id: "im1" }, ...over });

describe("vacina na linha do tempo (módulo 29; fecha api#71)", () => {
  it("filtro só com o interruptor; rótulos", () => {
    expect(timelineKinds(false, false, false, true).map((k) => k.key)).toContain("vaccination");
    expect(timelineKinds(true, true, true).map((k) => k.key)).not.toContain("vaccination");
    expect(timelineKinds(true, true, true, true)).toEqual(TIMELINE_KINDS);
    expect(timelineKindLabel("vaccination")).toBe("Vacina");
    expect([ statusLabel("active"), statusLabel("cancelled") ]).toEqual([ "aplicada", "cancelado" ]);
  });
  it("abre o detalhe da dose só em contexto", () => {
    expect(detailTarget(entry(), true)).toEqual({ type: "immunization", id: "im1" });
    expect(detailTarget(entry(), false)).toBeNull();
    expect(detailTarget(entry({ detail_ref: null }), true)).toBeNull();
  });
});
```

```tsx
// src/modules/timeline/PatientTimeline.vaccination.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { immunization, vacUser } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { PatientTimeline } from "./PatientTimeline";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getTimeline: vi.fn(), getImmunization: vi.fn() }));
beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser([ "health_professional" ], { features: [ "clinical_record", "patient_timeline", "vaccination" ] }));
  vi.mocked(api.getTimeline).mockResolvedValue({ person: { display_name: "Joana Lima", partial: false, patient_id: "pa1" }, next_before: null,
    entries: [ { id: "e1", kind: "vaccination", occurred_at: "2026-09-01T09:30:00-03:00", unit: { id: "u1", name: "UBS Centro" },
      professional: { name: "Téc. Ana Souza", cbo_code: "322205" }, title: "Vacina Penta (DTP/HepB/Hib) — 1ª dose", summary: null, status: "active",
      detail_ref: { type: "immunization", id: "im1" } } ] });
  vi.mocked(api.getImmunization).mockResolvedValue(immunization());
});
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("linha do tempo com vacina", () => {
  it("filtro Vacina, entrada e detalhe lido no clique", async () => {
    renderWithProviders(<PatientTimeline subject={{ attendanceId: "a1" }} />);
    expect(await screen.findByText("Vacina Penta (DTP/HepB/Hib) — 1ª dose")).not.toBeNull();
    expect(screen.getByLabelText("Vacina")).not.toBeNull();
    expect(api.getImmunization).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: /^Abrir vacina de/ }));
    await waitFor(() => expect(api.getImmunization).toHaveBeenCalledWith("im1"));
    expect(await screen.findByText("enviada ao Ministério")).not.toBeNull();
  });
});
```

E em `src/lib/careLines.test.ts`:

```ts
describe("vacina não conferida (módulo 29)", () => {
  it("rótulo, tom e texto; não piora a pessoa", () => {
    expect([ statusLabel("unchecked"), statusTone("unchecked") ]).toEqual([ "não conferido", "neutral" ]);
    expect(practiceText({ key: "vaccine:42", label: "Penta", status: "unchecked", done_at: null, due_on: null, count: null, expected: null }))
      .toBe("não conferido — a caderneta de vacina ainda não foi conferida na sala de vacina");
    expect(worstStatus([ { status: "unchecked" }, { status: "ok" } ])).toBe("ok");
    expect(worstStatus([ { status: "unchecked" } ])).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/timeline.vaccination.test.ts src/modules/timeline/PatientTimeline.vaccination.test.tsx src/lib/careLines.test.ts`
Expected: FAIL — o tipo, o rótulo, o detalhe e o texto não existem.

- [ ] **Step 3: Implemente** o que o Interfaces descreve. O filtro do `PatientTimeline` é um checkbox por tipo com o rótulo do `TIMELINE_KINDS` ("Vacina") e o botão de cada entrada tem `aria-label={openLabel(e)}` ("Abrir vacina de <quando>"). A entrada da vacina segue a forma real da linha do tempo (`occurred_at`, `detail_ref.type`) — D11.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/timeline.vaccination.test.ts src/lib/timeline.test.ts src/lib/timeline.pharmacy.test.ts src/modules/timeline src/lib/careLines.test.ts src/modules/careLines && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/timeline.ts src/lib/timeline.vaccination.test.ts src/modules/timeline/PatientTimeline.tsx src/modules/timeline/TimelineDetail.tsx \
  src/modules/timeline/PatientTimeline.vaccination.test.tsx src/modules/vaccination/ImmunizationDetail.tsx src/lib/careLines.ts src/lib/careLines.test.ts
$G commit -m "feat: show vaccinations in the timeline and unchecked vaccine milestones

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Critérios de vacina e convocados vacinados nas campanhas (F-29.8)

**Files:**
- Modify: `src/lib/campaigns.ts`, `src/modules/campaigns/AudienceBuilder.tsx`, `src/modules/campaigns/CriterionCard.tsx`, `src/modules/campaigns/CampaignPanel.tsx`
- Test: `src/lib/campaigns.vaccination.test.ts`, `src/modules/campaigns/Vaccination.test.tsx`

**Interfaces:**
- Consumes: `Criterion`, `CampaignOptions.immunobiologicals`, `Campaign.vaccinated_count` (Task 1).
- Produces (em `campaigns.ts`):
  - `CRITERION_LABEL` ganha `vaccine_overdue: "Vacina em atraso"`, `vaccination_record_unchecked: "Caderneta de vacina não conferida"`;
  - `criterionKindsFor(options: CampaignOptions): CriterionKind[]` — os dois só quando `options.immunobiologicals` vier (o api só manda com `vaccination` ligado, D12); o `AudienceBuilder` troca `CRITERION_KINDS` por ela;
  - `newCriterion("vaccine_overdue", today)` = `{ kind, immunobiological_codes: [] }`; `newCriterion("vaccination_record_unchecked", today)` = `{ kind }`;
  - `validateCriterion`: `vaccine_overdue` sem vacina → "marque ao menos uma vacina"; idade (meses) inteira de 0 a 1200 → senão "idade em meses: número inteiro de 0 a 1200"; mínima > máxima → "a idade mínima é maior que a máxima" (nos dois critérios);
  - `cleanCriterion`: idade vazia sai do objeto; vacina repetida vira uma;
  - texto fixo `VACCINE_OVERDUE_NOTE = "Só entram cadernetas conferidas na sala de vacina. Para chamar quem ainda não trouxe a caderneta, use “Caderneta de vacina não conferida”."`.
- Produces (telas): `CriterionCard` com os dois tipos (caixas por vacina em ordem de nome; "Idade mínima (meses)" e "Idade máxima (meses)"; a nota acima); `CampaignPanel` mostra `<Stat label="Vacinados depois do envio" value=…>` quando `vaccinated_count` é número.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/campaigns.vaccination.test.ts
import { describe, expect, it } from "vitest";
import { buildAudience, criterionKindsFor, newCriterion, validateCriterion } from "./campaigns";
import type { CampaignOptions } from "./api";

const OPTIONS: CampaignOptions = { protocols: [], tiers: [], outcomes: [], neighborhoods: [], units: [] };
const VAC = { ...OPTIONS, immunobiologicals: [ { code: "42", name: "Penta" }, { code: "15", name: "BCG" } ] };

describe("critérios de vacina (módulo 29)", () => {
  it("só com o catálogo de vacinas nas opções", () => {
    expect(criterionKindsFor(OPTIONS)).not.toContain("vaccine_overdue");
    expect(criterionKindsFor(VAC).slice(-2)).toEqual([ "vaccine_overdue", "vaccination_record_unchecked" ]);
  });
  it("validação e JSON limpo", () => {
    const c = newCriterion("vaccine_overdue", "2026-11-02");
    expect(validateCriterion(c, "2026-11-02")).toBe("marque ao menos uma vacina");
    expect(validateCriterion({ kind: "vaccine_overdue", immunobiological_codes: [ "42" ], age_min_months: 24, age_max_months: 12 }, "2026-11-02"))
      .toBe("a idade mínima é maior que a máxima");
    expect(validateCriterion({ kind: "vaccination_record_unchecked", age_max_months: 1.5 }, "2026-11-02")).toBe("idade em meses: número inteiro de 0 a 1200");
    expect(validateCriterion({ kind: "vaccination_record_unchecked" }, "2026-11-02")).toBeNull();
    const audience = buildAudience({ geo: { scope: "city" }, criteria: [
      { key: "k1", criterion: { kind: "vaccine_overdue", immunobiological_codes: [ "42", "42", "15" ], age_max_months: 23 } } ] });
    expect(audience.clinical.all).toEqual([ { kind: "vaccine_overdue", immunobiological_codes: [ "42", "15" ], age_max_months: 23 } ]);
  });
});
```

```tsx
// src/modules/campaigns/Vaccination.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { AudienceBuilder } from "./AudienceBuilder";
import { emptyAudienceDraft } from "../../lib/campaigns";

afterEach(cleanup);
const OPTIONS = { protocols: [], tiers: [], outcomes: [], neighborhoods: [], units: [], immunobiologicals: [ { code: "42", name: "Penta" }, { code: "15", name: "BCG" } ] };

describe("público com vacina em atraso", () => {
  it("oferece o critério, as vacinas em ordem e a nota", () => {
    const onChange = vi.fn();
    render(<AudienceBuilder draft={emptyAudienceDraft()} options={OPTIONS} today="2026-11-02" onChange={onChange} />);
    fireEvent.click(screen.getByRole("button", { name: "Adicionar critério clínico" }));
    fireEvent.click(screen.getByRole("button", { name: "Vacina em atraso" }));
    const next = onChange.mock.calls[0][0];
    cleanup();
    render(<AudienceBuilder draft={next} options={OPTIONS} today="2026-11-02" onChange={vi.fn()} />);
    expect(screen.getAllByRole("checkbox").map((c) => (c.parentElement?.textContent ?? "").trim())).toEqual([ "BCG", "Penta" ]);
    expect(screen.getByText(/Só entram cadernetas conferidas/)).not.toBeNull();
    expect(screen.getByText("marque ao menos uma vacina")).not.toBeNull();
  });
  it("sem o catálogo nas opções, não oferece", () => {
    render(<AudienceBuilder draft={emptyAudienceDraft()} options={{ ...OPTIONS, immunobiologicals: undefined }} today="2026-11-02" onChange={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: "Adicionar critério clínico" }));
    expect(screen.queryByRole("button", { name: "Vacina em atraso" })).toBeNull();
  });
});
```

E no teste existente de `CampaignPanel` (`src/modules/campaigns/CampaignPanel.test.tsx`), um caso: campanha `sent` com `vaccinated_count: 7` mostra "Vacinados depois do envio" e "7"; com `vaccinated_count: null`, o rótulo não aparece.

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/campaigns.vaccination.test.ts src/modules/campaigns`
Expected: FAIL — os tipos novos não têm rótulo, regra nem cartão.

- [ ] **Step 3: Implemente** o que o Interfaces descreve. No `AudienceBuilder`, o grupo "Tipos de critério" (aberto por "Adicionar critério clínico") passa a listar `criterionKindsFor(options)`. O `Record<CriterionKind, …>` de `CRITERION_LABEL` obriga as duas entradas; nenhum outro critério muda.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/lib/campaigns.vaccination.test.ts src/lib/campaigns.test.ts src/modules/campaigns src/modules/Campaigns.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/campaigns.ts src/lib/campaigns.vaccination.test.ts src/modules/campaigns/AudienceBuilder.tsx src/modules/campaigns/CriterionCard.tsx \
  src/modules/campaigns/CampaignPanel.tsx src/modules/campaigns/CampaignPanel.test.tsx src/modules/campaigns/Vaccination.test.tsx
$G commit -m "feat: add overdue vaccine and unchecked card campaign criteria

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Botão "Notificar ESAVI" e painel de vacinação (F-29.8)

**Files:**
- Create: `src/modules/vaccination/EsaviButton.tsx`
- Modify: `src/modules/vaccination/ImmunizationRow.tsx`, `src/modules/VaccinationPanel.tsx`
- Test: `src/modules/vaccination/EsaviPanel.test.tsx`

**Interfaces:**
- Consumes: `notifyEsavi`, `getVaccinationDashboard`, `listAllUnits` (Task 1 e existente); `canUseSurveillance`, `surveillanceRoles` (`surveillance.ts`); `NotificationDetail` (25); `VAC_DASHBOARD_KEY`, `STATIC_QUERY`, `monthProblem`, `currentMonth`, `vaccinationError` (Task 2); `StatTile`, `DataTable`.
- Produces:
  - `EsaviButton({ imm }: { imm: Immunization })` — só com `surveillance` na sessão e dose ativa; dois passos ("Notificar ESAVI — <vacina> <dose>" → "Sim, abrir a notificação"); sucesso (ou 409 `already_notified` com `notification_id`, D16): `ESAVI_OPENED = "Notificação de ESAVI aberta na vigilância, presa a esta dose."` (ou a frase de `already_notified`) e, para quem tem o papel `surveillance`, o `NotificationDetail` da notificação logo abaixo (com "Fechar"); para os outros, `ESAVI_HINT = "A equipe da vigilância completa a ficha e envia ao e-SUS Notifica."`; botão desabilitado enquanto corre;
  - `VaccinationPanel({ today? })`: "Mês" (padrão o atual) e "Unidade" (todas ou uma, de `listAllUnits`), "Ver painel" lê `getVaccinationDashboard` com `STATIC_QUERY`; `StatTile`s: "Doses na fila da RNDS" (`pending`), "Paradas por cadastro" (`blocked`), "Recusadas pela RNDS" (`rejected`), "Fora do prazo de 24 h" (`late`), "Perda técnica (doses)", "Perda física (doses)", "Cadernetas não conferidas"; tabela "Doses aplicadas por vacina e sala"; tabela "Cobertura das vacinas do C2 por equipe" com "completas / elegíveis" e a porcentagem com vírgula ("75,0%"; elegíveis 0 → "—"); só contagens, nenhum nome.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/vaccination/EsaviPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY29, immunization, vacDashboard, vacUser } from "../../test/vaccinationFixtures";
import * as api from "../../lib/api";
import { EsaviButton } from "./EsaviButton";
import { VaccinationPanel } from "../VaccinationPanel";

vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), notifyEsavi: vi.fn(),
  getNotification: vi.fn(), getVaccinationDashboard: vi.fn(), listAllUnits: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });

describe("ESAVI (F-29.8)", () => {
  beforeEach(() => vi.mocked(api.notifyEsavi).mockResolvedValue({ notification_id: "n9" }));

  it("sem a vigilância ligada, nada é oferecido", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser());
    renderWithProviders(<EsaviButton imm={immunization()} />);
    await waitFor(() => expect(api.fetchCurrentSession).toHaveBeenCalled());
    expect(screen.queryByRole("button", { name: /Notificar ESAVI/ })).toBeNull();
  });

  it("vacinador: dois passos, uma chamada, frase e dica", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser([ "health_professional" ], { features: [ "clinical_record", "vaccination", "surveillance" ] }));
    renderWithProviders(<EsaviButton imm={immunization()} />);
    fireEvent.click(await screen.findByRole("button", { name: "Notificar ESAVI — Penta (DTP/HepB/Hib) 1ª dose" }));
    expect(api.notifyEsavi).not.toHaveBeenCalled();
    const yes = screen.getByRole("button", { name: "Sim, abrir a notificação" });
    fireEvent.click(yes); fireEvent.click(yes);
    await waitFor(() => expect(api.notifyEsavi).toHaveBeenCalledTimes(1));
    expect(await screen.findByText("Notificação de ESAVI aberta na vigilância, presa a esta dose.")).not.toBeNull();
    expect(screen.getByText("A equipe da vigilância completa a ficha e envia ao e-SUS Notifica.")).not.toBeNull();
    expect(api.getNotification).not.toHaveBeenCalled();
  });

  it("papel surveillance: abre a notificação; já notificada abre a existente", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser([ "health_professional", "surveillance" ], { features: [ "clinical_record", "vaccination", "surveillance" ] }));
    vi.mocked(api.notifyEsavi).mockRejectedValue(new api.ApiError(409, { error: "already_notified", notification_id: "n3" }, "409"));
    renderWithProviders(<EsaviButton imm={immunization()} />);
    fireEvent.click(await screen.findByRole("button", { name: /^Notificar ESAVI/ }));
    fireEvent.click(screen.getByRole("button", { name: "Sim, abrir a notificação" }));
    expect(await screen.findByText("esta dose já tem notificação de ESAVI aberta")).not.toBeNull();
    await waitFor(() => expect(api.getNotification).toHaveBeenCalledWith("n3"));
  });
});

describe("painel de vacinação (F-29.8)", () => {
  it("só contagens; mês e unidade na URL; cobertura com vírgula", async () => {
    vi.mocked(api.fetchCurrentSession).mockResolvedValue(vacUser([ "analyst" ]));
    vi.mocked(api.listAllUnits).mockResolvedValue([ { id: "u1", name: "UBS Centro", active: true } ] as never);
    vi.mocked(api.getVaccinationDashboard).mockResolvedValue(vacDashboard({ c2_coverage_by_team: [
      { team: { id: "t1", name: "eSF Centro 1" }, eligible: 40, complete: 30 }, { team: { id: "t2", name: "eSF Norte" }, eligible: 0, complete: 0 } ] }));
    renderWithProviders(<VaccinationPanel today={TODAY29} />);
    fireEvent.change(await screen.findByLabelText("Unidade"), { target: { value: "u1" } });
    fireEvent.click(screen.getByRole("button", { name: "Ver painel" }));
    await waitFor(() => expect(api.getVaccinationDashboard).toHaveBeenCalledWith({ month: "2026-11", healthUnitId: "u1" }));
    expect(await screen.findByText("Fora do prazo de 24 h")).not.toBeNull();
    const c2 = screen.getByRole("region", { name: "Cobertura das vacinas do C2 por equipe" });
    expect(within(c2).getByText("75,0%")).not.toBeNull();
    expect(within(c2).getByText("—")).not.toBeNull();
    expect(api.getVaccinationDashboard).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination/EsaviPanel.test.tsx`
Expected: FAIL — as peças não existem.

- [ ] **Step 3: Implemente** `EsaviButton` (coloque-o no espaço de ações do `ImmunizationRow`, ao lado de "Cancelar dose"), e o corpo de `VaccinationPanel`. O `NotificationDetail` do 25 lê por `getNotification(id)` (`NOMINAL_QUERY`). A 403 `feature_disabled` com `feature: "surveillance"` mostra `"a vigilância está desligada nesta cidade — notifique o ESAVI pelo e-SUS Notifica"`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod29 src/modules/vaccination src/modules/VaccinationPending.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/vaccination/EsaviButton.tsx src/modules/vaccination/ImmunizationRow.tsx src/modules/VaccinationPanel.tsx src/modules/vaccination/EsaviPanel.test.tsx
$G commit -m "feat: add the ESAVI notification button and the vaccination panel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Suíte, build, revisão e prova no navegador (api real na 3051)

**Files:** nenhum novo (correções que a prova pedir entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod29
$R/bin/test dashboard -w dashboard-mod29
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **18 arquivos de teste novos**: `src/lib/api.vaccination.test.ts`, `src/lib/vaccination.test.ts`, `src/shell/modules.vaccination.test.ts`, `src/modules/vaccination/VaccinationGate.test.tsx`, `src/modules/VaccinationAdmin.test.tsx`, `src/modules/Integrations.rnds.test.tsx`, `src/modules/vaccination/CardView.test.tsx`, `src/modules/vaccination/DoseForm.test.tsx`, `src/modules/vaccination/Transcription.test.tsx`, `src/modules/VaccinationPending.test.tsx`, `src/modules/VaccinationStock.test.tsx`, `src/modules/vaccination/Vials.test.tsx`, `src/modules/VaccinationColdChain.test.tsx`, `src/lib/timeline.vaccination.test.ts`, `src/modules/timeline/PatientTimeline.vaccination.test.tsx`, `src/lib/campaigns.vaccination.test.ts`, `src/modules/campaigns/Vaccination.test.tsx`, `src/modules/vaccination/EsaviPanel.test.tsx`. Contagem dobrada = artefato de build descoberto pelo vitest (lição do admin): apague o artefato, não o teste.

- [ ] **Step 2: Varredura de dado sensível e de releitura**

```bash
cd $W
grep -rnE "refetchInterval" src/modules/vaccination src/modules/Vaccination*.tsx && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" src/modules/vaccination src/modules/Vaccination*.tsx src/modules/integrations src/lib/vaccination.ts && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "searchParams|URLSearchParams" src/lib/api.ts | sed -n '/Sala de vacina/,$p' | grep -vE "location_id|month|health_unit_id|status|before|limit" && echo "revisar" || echo ok-url
```

Expected: `ok-polling`, `ok-storage`, `ok-url`.

- [ ] **Step 3: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus e no que o 27a reusado pode ter quebrado (dispensação, transferência e ajustes de medicamento).

- [ ] **Step 4: Prova no navegador** (só com o api do 29 mergeado ou no worktree dele, na 3051, e o banco de dev migrado e semeado **com autorização do usuário**)

Suba o Vite do worktree na rede do compose, porta **5207**, com proxy para `http://api:3051` (mesmo comando da prova do 28, trocando porta e worktree), abra `curitiba.localhost:5207/dashboard/` e percorra, com `rnds_mock` ligado:
1. Integrações: cadastrar o certificado de teste do `fake-rnds`, "Testar conexão" ok; o interruptor `rnds_immunization` sai de "parado".
2. Salas e rede de frio: criar sala, vincular vacinador e responsável (e ver a recusa de CBO), criar a rede de frio e o almoxarife, gravar o go-live.
3. Imunobiológicos: entrada na rede de frio (BCG multidose e Penta unidose), transferir à sala, receber na sala.
4. Sala: cidadão com caderneta não conferida (sem "atraso"), transcrever e fechar a conferência, registrar Penta (unidose) e BCG (abrir frasco), provocar o aviso de intervalo e aplicar com motivo; cancelar com substituição; no `fake-rnds`, um aceite, uma recusa por marcador e um bloqueio por CNS fora do CNES — conferir as situações na caderneta e em "Pendências do envio".
5. Cadeia de frio: leitura fora da faixa → lotes bloqueados (o lote some do registro da dose) → conduta "usar" num e "dar perda" no outro.
6. Resumo mensal, painel, linha do tempo (entrada "Vacina" e detalhe), linhas de cuidado ("não conferido"), campanha com "Vacina em atraso" e o ESAVI aberto na vigilância.
7. Tablet (1024×768) e só teclado no fluxo 4; telefone (375×812) nas listas (cartões).

Anote cada passo com o que viu. Achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai para o usuário (não vira tarefa).

- [ ] **Step 5: Board e encerramento**

Cards do dashboard prontos: F-29.2..F-29.6 e F-29.8 → **Done** se o api já terminou o deles; F-29.1 espera o maintenance e F-29.7 o wpda (o último app move). **Verified** só depois da prova do Step 4, com o usuário. Merge, push e tag só com autorização. Worktree: `$G -C $R/apps/dashboard worktree remove $W` depois do merge.

---

## Divergências propostas ao contrato

Cada uma diz o que a tela precisa, o que o plano assume e quem decide. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D1 — Rota e corpo da credencial RNDS.** O contrato §3 diz `PUT /integrations/rnds`; a rota real do 16 é `PUT /integrations/credentials/:kind` e `POST /integrations/credentials/:kind/check`. Proposta: `PUT /integrations/credentials/rnds` com `{ pfx_base64, pfx_password, requester_cnes }` (a resposta é a `IntegrationCredential` de sempre, sem segredo); recusas `invalid_certificate` (o `.pfx` não abre com a senha), `certificate_expired`, `invalid_requester_cnes` (422). `IntegrationCredential::KINDS` ganha `rnds` e a validação do segredo por tipo (hoje só aceita `username`/`password`).
- **D2 — Os locais da pessoa.** O contrato não tem leitura dos locais de vacina da pessoa; `GET /pharmacy/locations` exige `pharmacy`. Proposta: `GET /vaccination/my_locations` → `{ locations: [ { id (stock_location_id), name, kind: "vaccine_room"|"cold_chain_central", health_unit|null, active, roles: [...], room_id|null } ] }` com `roles` ⊂ `vaccinator`, `room_lead`, `stock_keeper`; o almoxarife da rede de frio recebe também as salas ativas com `roles: []` (destinos da transferência, como o D1 do 27a).
- **D3 — Busca do cidadão na sala.** O contrato só tem `POST …/card { patient_id }`. A sala precisa achar a pessoa e atender quem não tem cadastro (recém-nascido). Proposta: `POST /vaccination/person_lookup { cpf }` e `POST /vaccination/patients/duplicates { full_name, birth_date, mother_name? }` (formas do 27a/25) e `POST …/card` aceitando `{ patient_id }` | `{ citizen_id }` | `{ new_patient, duplicates_reviewed? }` (o api cria o paciente; 409 `duplicate_suspected` como no 24a).
- **D4 — Administração.** Formas não fixadas. Proposta: `GET /vaccination/admin/rooms` → `{ rooms: [ { id, name, active, health_unit: { id, name }, stock_location_id, staff: [ { user_id, name|null, role, cbo|null } ] } ], cold_chain_central: <AdminLocation do 27a>|null }`; escritas devolvem a sala inteira; `DELETE …/staff` → 204; `POST …/cold_chain_central` → a `AdminLocation`; `GET/PATCH …/settings` → `{ go_live_on: "YYYY-MM-DD"|null }`; todas as escritas com step-up.
- **D5 — Imunobiológico nas respostas do 27a.** Proposta: `StockItem` de vacina com `catalog_item_id: null`, `catmat_code: null`, `immunobiological_code`, `item_kind: "immunobiological"`, `name` do catálogo, `unit`/`dispensing_unit` = `"dose"`; `StockLot` com `manufacturer_code` (código RNDS) além do `manufacturer` (nome); `LotBalance.blocked: boolean` (excursão aberta) na posição.
- **D6 — Entrada de imunobiológico.** Proposta: item de `POST /pharmacy/receipts` e `/openings` com `{ immunobiological_code, manufacturer_code, lot_number, expires_on, quantity (doses, texto), package?: { units_per_package: doses_por_frasco, packages: frascos } }`; recusas por item em `problems[]` (`items[i].manufacturer_code` → `manufacturer_not_in_catalog`).
- **D7 — Frascos.** `GET …/vials` em envelope `{ vials: [...] }`; `close_out` com motivo de 10–500 e recusa `room_lead_required`.
- **D8 — Resumo mensal.** Proposta: `GET /vaccination/rooms/:room_id/monthly_summary?month=` e `GET /vaccination/cold_chain_central/monthly_summary?month=` → `{ month, location: LocationRef, rows: [ { immunobiological: { code, name }, lot: { id, number }, opening_balance, received, transferred_in, transferred_out, vials_opened, doses_applied, technical_loss, physical_loss: { <motivo>: n }, closing_balance } ] }` (inteiros, em doses). A lista de motivos da perda física fica para o api fixar (a tela mostra o código cru até lá).
- **D9 — Cadeia de frio.** Proposta: `GET …/equipments?location_id=` → `{ equipments: [...] }`; leitura → 201 `{ reading: { id, read_at, current_c, min_c, max_c, out_of_range, by }, excursion_id|null, blocked_lots }`; `GET …/status?location_id=` → `{ equipments: [ { ...equipamento, last_reading|null, readings_today } ], open_excursions: [ { id, equipment_id, reading, lots: [ { lot_id, number, immunobiological, balance } ], opened_at } ] }`; `decide` → `{ excursion_id, adjustments: [<StockAdjustment do 27a>] }` (o "dar perda" acima do limite fica `pending`); `excursion_closed` (409) quando já decidida. Temperaturas como número com uma casa.
- **D10 — Pendências.** `GET …/pending` → `{ items: [ { immunization_id, patient_id, immunobiological: { code, name }, applied_at, rnds: <rnds> } ], next_before }`.
- **D11 — Linha do tempo.** O §10 usa `at` e `detail_ref.kind`; a forma real é `occurred_at` e `detail_ref.type`. Proposta: `{ kind: "vaccination", occurred_at, unit, professional, title: "Vacina <nome> — <dose>", summary: null, status: "active"|"cancelled", detail_ref: { type: "immunization", id } }` e a leitura nominal `GET /vaccination/immunizations/:id` → `<immunization>` (trilha, mesma regra de acesso da linha do tempo). Sem a rota, a entrada não abre detalhe (`detail_ref: null`).
- **D12 — Campanhas.** O campaign_manager não tem acesso ao catálogo da sala. Proposta: `GET /campaigns/options` ganha `immunobiologicals: [ { code, name } ]` só com `vaccination` ligado; `Campaign.vaccinated_count: number|null` (número só com campanha enviada e critério `vaccine_overdue`).
- **D13 — Painel.** Formas de `doses_by_immunobiological` (`{ immunobiological, room, count }`) e `c2_coverage_by_team` (`{ team: { id, name }, eligible, complete }`).
- **D14 — Limites de texto.** `override_reason`, motivo do cancelamento, do descarte e de cada conduta: 10–500; nome de sala e de equipamento: 3–120; lote da transcrição ≤ 40; fabricante e local da transcrição ≤ 120; certificado ≤ 100 KB.
- **D15 — Formas pequenas.** `terminologies.*` como `[ { code, name } ]`; `<card>.items[].applied` (a "immunization-resumo") como `{ id, applied_at, transcription }`.
- **D16 — ESAVI repetido.** `POST …/esavi` numa dose já notificada → 409 `already_notified` com `notification_id` (a tela abre a existente). Com `surveillance` desligado → 403 `feature_disabled` com `feature: "surveillance"`.
- **D17 — Cancelar e substituir é atômico.** Recusa da substituição não cancela nada; os problemas dela vêm em `problems[]` com path `replacement.<campo>`.
- **D18 — Quem cancela.** `<immunization>` não diz quem registrou (só nome e CBO). Proposta: `can_cancel: boolean` calculado pelo api para quem lê (autor ou `room_lead` da sala, dose ativa).
- **D19 — Avisos.** A tela trata `interval_too_short` e `age_out_of_range` como avisos que aceitam `override_reason`. Se o api criar outro aviso, proposta: marcar no problema `"overridable": true` (a tela passa a ler a marca em vez da lista fixa).

## Self-review

1. **Cobertura da spec:** decisão 3a → Tasks 2 e 6 (situação efetiva, conferência, aviso) e 14 (`vaccination_record_unchecked`); 3 → 14; 4 → 7; 5 → 8; 6 → 4 (CBO e papel na sala); 7 → 5 e 9; 8 → 9 (situação e pendências; o alerta de 24 h é do api — no painel como `late`, Task 15); 9 → 9 (motivos de bloqueio); 10 → 6 e 7 (catálogo e aprazamento lidos do api); 11 → 10; 12 → 7 e 11 (frasco, perda técnica no resumo); 13 → 11; 14 → 12; 15 → 6 (dashboard; o wpda tem plano próprio); 16 → 13, 14, 15; 17 → 15; 18 → 1, 3 e 5 (interruptores e pendências novas). Decisões do coordenador: local próprio e rede de frio → 4 e 10; sem dependência da farmácia → 3 (teste) e 10; papéis na sala e na rede de frio → 3, 10, 11, 12; credencial por cidade → 5. §4 telas → todas; §5 → Global Constraints e os testes de URL/storage (Tasks 1, 6, 16). Nenhuma lacuna.
2. **Passos:** cada passo de código diz arquivo, assinatura e os valores da spec/contrato; corpos só para a fixture e os testes. Os pontos que dependem do texto real (botão "Adicionar critério clínico", `openLabel`, `getNotification` do 25, "Buscar pelo CPF" do `PersonStart`) foram conferidos no `origin/main` do dashboard (`fadfa87`).
3. **Tipos:** `VaccineLocation`, `CardRequest`, `ImmunizationInput`, `Vial`, `ReadingInput`, `VaccineReceiptItemInput` e as funções da Task 1 são os mesmos nas Tasks 2–15; `DoseFields` nasce na Task 8 de dentro do `DoseForm` da Task 7; `TransfersBody` e `itemKey` nascem nas Tasks 10 e 2.
4. **Review Focus:** as cinco linhas têm teste na task dona (2, 6, 7, 7, 7, 2 e 12).
5. **Proporção:** o código é teste e fixture; implementação por assinatura e valores.
