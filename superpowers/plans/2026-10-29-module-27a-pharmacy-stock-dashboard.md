# Módulo 27a — Estoque e dispensação (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 27a** (`feat/mod-27a-pharmacy-stock`, worktree `apps/api/.claude/mod27a`, porta de dev **3049**) com as rotas do contrato §3–§9 (e as Divergências D1–D25 deste plano que o plano do api adotar), os papéis `pharmacist`, `pharmacy_assistant` e `stock_keeper`, o interruptor `pharmacy` na sessão, o item `dispensation` na linha do tempo, `last_dispensed_on` na lista de medicamentos e a semente do 27a. Ele é **mergeado antes** deste (contrato §12: api → dashboard → maintenance → catálogo). As Tasks 0–14 não dependem dele (dublês de `src/lib/api`); a prova da Task 15 depende.

**Goal:** Lado dashboard do 27a: o grupo **"Farmácia"** com a **administração** (locais de estoque central e de unidade, equipe por local, configurações da cidade); o **almoxarifado** (entrada por nota ou doação com lote, validade e conversão de embalagem; inventário de abertura; transferência ao local da unidade; posição por local e lote com extrato e estoque mínimo; ajustes, perdas e vencimentos com step-up e aprovação pendente); a **farmácia da unidade** (recebimento conferido lote a lote, com diferença e motivo; **dispensação** pela receita digital — busca pelo código ou pelo paciente —, item a item, com quantidade dispensada, lote sugerido pelo vencimento mais próximo, "dispensado por completo", próxima retirada do uso contínuo, item em texto livre mapeado pelo farmacêutico, controlado só pelo farmacêutico com retenção da receita; **receita externa**; histórico de retiradas do paciente); os **alertas**; e, no prontuário, a "última retirada" na lista de medicamentos e o item **Dispensação** na linha do tempo.

**Architecture:** O cliente HTTP ganha a seção "Farmácia e almoxarifado (módulo 27a)" no fim de `src/lib/api.ts` (tipos do contrato §2–§9; na URL só ids, situação, filtros booleanos, nome do medicamento na busca da posição, cursor e limite; código da receita, CPF, nome, receita, motivo e número de notificação sempre no corpo). As regras de tela ficam puras em `src/lib/pharmacy.ts` (portão e papéis por local, rótulos, quantidade decimal com vírgula e até 2 casas — recusar, nunca cortar —, código curto da receita, lote sugerido, o que impede dispensar, conferências dos formulários com **todas** as recusas de uma vez, frases das recusas do api, chaves de consulta). As peças ficam em `src/modules/pharmacy/` e as telas em `src/modules/Pharmacy*.tsx`, no grupo novo (`modules.ts`, `catalog.ts`, `App.tsx`). O prontuário só ganha leitura (`PatientTimeline`, `MedicationsBlock`). O dashboard nunca decide o que o api garante: saldo nunca negativo, travas (lote → local → receita), validade da receita, intervalo do uso contínuo, quem dispensa controlado, aprovação acima do limite, retenção LGPD e imutabilidade são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md` (§1 intenção, sucesso e antiobjetivos, §2 decisões 3–12 e as do coordenador, §4 telas, §5 segurança e LGPD, §6 testes, §8 F-IDs, §9 riscos), `docs/adr/0045.md` e o contrato `docs/superpowers/plans/2026-10-29-module-27a-pharmacy-stock-contracts.md` (§1–§9, §11, §12), com as convenções herdadas (21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12, 24a §14, 25 §15, 26a §13). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D25); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptor `pharmacy` (sem `requires`); papéis `pharmacist` (CBO `2234*`), `pharmacy_assistant`, `stock_keeper`, vinculados a locais; local `central` | `unit`; movimento `opening` | `receipt` | `donation` | `transfer_out` | `transfer_in` | `dispensing` | `loss` | `expiry` | `adjustment` | `return`; transferência `sent` | `received` | `received_with_difference`; ajuste `applied` | `pending` | `approved` | `rejected`; receita `digital` | `external`. Código desconhecido aparece cru, nunca some.
- Interruptor (spec decisão 12): sem `pharmacy` na sessão, o dashboard é o de hoje — o grupo "Farmácia" some e o filtro da linha do tempo não oferece o tipo "Dispensação"; o que já foi dispensado continua aparecendo na linha do tempo e na "última retirada" (registro clínico retido; P10 e Desvio 14 do plano do api). 403 `feature_disabled` → `PHARMACY_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo ("Farmácia e almoxarifado") em `features.ts`.
- **Quem atua onde** (spec, decisões do coordenador; contrato §3–§7): tudo de estoque e dispensação é **por local** — a tela lê os locais da pessoa (`GET /pharmacy/locations`, D1) e oferece só o que o papel **naquele local** permite: `stock_keeper` (entradas, abertura, transferência — envio só do central —, mínimos, ajustes); `pharmacist` (dispensa tudo, mapeia texto livre, registra retenção, faz a abertura, ajusta e aprova); `pharmacy_assistant` (dispensa o que não é controlado nem texto livre, recebe transferência); `stock_keeper` do central lê a posição e os alertas de todos os locais; `municipal_admin` administra locais, equipe e configurações (escrita com step-up pelo `SensitiveAction`). O api recusa de novo.
- **Dado do paciente só no corpo** (spec §5; contrato, convenções): na URL vão só `location_id`, `patient_id`, `catalog_item_id`, `status`, `expiring`, `below_minimum`, `q` (nome do **medicamento**, só na posição), `before` e `limit`. Código da receita, CPF, CNS, nome, conteúdo da receita, profissional, motivo e número de notificação nunca na URL, em mensagem de erro, em `console` ou em Analytics; nada em `localStorage`, `sessionStorage` ou IndexedDB. A busca da receita (pelo código ou pelo paciente) é `POST` no clique (grava trilha) — nunca reação ao que se digita.
- **Nenhum polling em rota que grava trilha** (spec §5): busca da receita, receita externa e histórico do paciente leem só no clique ou depois de uma ação (`NOMINAL_QUERY`). Posição, extrato, transferências, ajustes e mínimos não têm dado pessoal: leem ao abrir, filtrar ou depois de uma ação (`STATIC_QUERY`). Só os alertas (sem trilha) releem a cada minuto (`PHARMACY_ALERTS_REFRESH_MS = 60_000`).
- **Quantidade** (spec decisão 5; contrato, convenções): na tela com vírgula, até 2 casas, maior que zero (o mínimo aceita zero), no máximo `99999,99`; ponto é recusado com `DOT_IN_NUMBER` (o mesmo do laboratório); no api, string com ponto (`"12.5"`). Nada é arredondado nem cortado: recusa com a frase. O sistema **não converte** a receita em quantidade: o profissional digita, vendo o texto da receita (decisão do coordenador). A embalagem da entrada é a única conta da tela (`unidades por embalagem × embalagens`, inteiros), e a conta aparece antes de enviar.
- **Antiobjetivos da spec, na tela:** nunca oferecer "Dispensar" em receita cancelada ou vencida; nunca oferecer quantidade em item controlado a quem não é farmacêutico; nunca enviar dispensação sem lote; nunca enviar saída maior que o saldo lido; nenhum botão de editar ou apagar movimento (correção é ajuste novo).
- **Todas as recusas de uma vez** (lição do 21b §14 B26): os formulários com várias linhas (entrada, transferência, recebimento, dispensação, receita externa) conferem todas as linhas e mostram cada problema com o item; a recusa do api com `problems` (D20) mostra todas; com só `path`, marca a linha.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo, inclusive o `DELETE` da equipe e a aprovação, que mandam `{ user_id, role }` e `{}`).
- Textos (valores do contrato e D13–D16, D24; se o api fixar outros, o api vence): motivo 10–500; número do lote 1–40; número do documento 1–60; fornecedor ou doador 1–120; fabricante até 120; número de notificação `[0-9A-Za-z./-]{1,30}`; nome do profissional 3–120; conselho `CRM` | `CRO` | `COREN`; UF do conselho 2 letras; número no conselho 1–10 dígitos; posologia 3–500; unidade de medida 1–30; unidades por embalagem 1–10000; embalagens 1–100000; configurações: validade comum e contínua 1–365, intervalo do contínuo 1–90, alerta de validade 1–365, limite do ajuste 0–100 (inteiros).
- "Hoje" é o dia da cidade (`todayInCity()`); telas recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY27 = "2026-10-29"` (quinta), sem relógio falso.
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `SensitiveAction`, `ActionButton`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `PersonStart` do 24a com fonte trocável); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-27.1..F-27.7** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-27.1: api, dashboard e maintenance; F-27.2..F-27.7: api e dashboard) e a **Verified** depois da prova no navegador (Task 15).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod27a` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-27a-pharmacy-stock` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod27a`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test dashboard -w dashboard-mod27a <arquivos>` (o runner único; saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx` (sessão `u-1` com `mfa_verified_at` de agora = janela de step-up aberta: o `SensitiveAction` não pede código).
- A prova (Task 15) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5205**, com proxy para o api do 27a na **3049**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Duas farmácias dispensam o último comprimido do mesmo lote ao mesmo tempo.** Espera-se: a segunda recebe 422 `insufficient_balance` com `path`; a tela diz que o saldo mudou, relê os lotes do item, mantém quantidades, lotes e retenção digitados e não reenvia sozinha. Teste: Task 10, "insufficient_balance: relê os lotes e mantém o que foi digitado; too_early com a data".
2. **Receita cancelada pelo médico enquanto o farmacêutico preenche.** Espera-se: 422 `prescription_cancelled` (ou `prescription_expired`) vira a frase, o formulário avisa quem chamou e a tela pede uma nova busca; nada é reenviado. Teste: Task 10, "receita cancelada enquanto dispensava: frase e nova busca".
3. **Quantidade digitada com ponto ou com três casas** ("1.000", "2,555"). Espera-se: recusa com a frase, sem arredondar nem cortar, e nada enviado. Teste: Task 2, "quantidade: vírgula, até 2 casas, maior que zero; recusa em vez de cortar".
4. **Recebimento com uma caixa a menos.** Espera-se: a diferença por lote pede motivo (10–500) antes de enviar; recebido maior que o enviado é recusado; a frase final diz que a diferença entrou como perda. Teste: Task 7, "unidade recebe: conferência lote a lote; diferença pede motivo; frase da diferença".
5. **Farmacêutico tenta aprovar o próprio ajuste**, ou a aprovação chega depois de outra pessoa decidir. Espera-se: o próprio ajuste não oferece aprovar nem rejeitar; 403 `same_user` e 409 `not_pending` viram frase e a lista é relida. Teste: Task 8, "pendentes: aprovar e rejeitar com motivo; o próprio não oferece; 409 relê".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.pharmacy.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/pharmacyFixtures.ts` | tipos e cliente HTTP do 27a; interruptor; proxy; dados de teste | 1 |
| `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts` | regras de tela: portão, papéis por local, rótulos, quantidade, código, lote sugerido, bloqueios, conferências, frases | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.pharmacy.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`, `src/modules/pharmacy/PharmacyGate.tsx`, `src/modules/pharmacy/LocationPick.tsx`, oito telas-portão `src/modules/Pharmacy*.tsx` | grupo "Farmácia", portão, escolha do local, papéis na Equipe | 3 |
| `src/modules/PharmacyAdmin.tsx`, `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/pharmacy/StaffPanel.tsx`, `src/modules/pharmacy/SettingsPanel.tsx`, `src/modules/PharmacyAdmin.test.tsx` | locais, equipe por local, configurações (F-27.1) | 4 |
| `src/modules/pharmacy/ItemSearch.tsx`, `src/modules/pharmacy/ReceiptForm.tsx`, `src/modules/PharmacyEntries.tsx`, `src/modules/PharmacyEntries.test.tsx` | busca no catálogo, entrada com lote e embalagem, abertura (F-27.2) | 5 |
| `src/modules/PharmacyStock.tsx`, `src/modules/pharmacy/LotStatement.tsx`, `src/modules/pharmacy/MinimumsPanel.tsx`, `src/modules/PharmacyStock.test.tsx` | posição por local e lote, extrato, mínimos (F-27.7) | 6 |
| `src/modules/PharmacyTransfers.tsx`, `src/modules/pharmacy/TransferForm.tsx`, `src/modules/pharmacy/ReceiveForm.tsx`, `src/modules/PharmacyTransfers.test.tsx` | envio do central, recebimento com diferença (F-27.3) | 7 |
| `src/modules/PharmacyAdjustments.tsx`, `src/modules/pharmacy/AdjustmentForm.tsx`, `src/modules/pharmacy/PendingAdjustments.tsx`, `src/modules/PharmacyAdjustments.test.tsx` | ajuste, perda, vencimento, aprovação (F-27.4) | 8 |
| `src/modules/PharmacyDispensing.tsx`, `src/modules/pharmacy/PrescriptionLookup.tsx`, `src/modules/pharmacy/PrescriptionCard.tsx`, `src/modules/PharmacyDispensing.test.tsx` | busca da receita por código ou paciente (F-27.5) | 9 |
| `src/modules/pharmacy/DispenseForm.tsx`, `src/modules/pharmacy/DispenseForm.test.tsx` | itens, lote sugerido, quantidade, completo, contínuo, texto livre, retenção (F-27.5, F-27.6) | 10 |
| `src/modules/PharmacyExternal.tsx`, `src/modules/pharmacy/ExternalPrescriptionForm.tsx`, `src/modules/PharmacyExternal.test.tsx` | receita externa e controlado de fora (F-27.6) | 11 |
| `src/modules/pharmacy/PatientHistory.tsx`, `src/modules/pharmacy/PatientHistory.test.tsx`, `src/modules/PharmacyDispensing.tsx` | histórico de retiradas do paciente (F-27.5) | 12 |
| `src/modules/PharmacyAlerts.tsx`, `src/modules/PharmacyAlerts.test.tsx` | vencendo, vencidos, abaixo do mínimo, em trânsito, ajustes pendentes (F-27.7) | 13 |
| `src/lib/timeline.ts`, `src/lib/timeline.test.ts`, `src/lib/timeline.exam.test.ts`, `src/lib/timeline.procedure.test.ts`, `src/lib/timeline.pharmacy.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/consultation/MedicationsBlock.tsx`, `src/modules/consultation/MedicationsBlock.test.tsx` | dispensação na linha do tempo e "última retirada" (spec decisão 11) | 14 |
| — | suíte, build, revisão, prova no navegador | 15 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (código da receita, CPF, nome, motivo, número de notificação, posologia) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa —, quantidade, código curto, lote sugerido, bloqueios por item e as recusas todas de uma vez (Task 2); menu por papel e interruptor e os papéis na Equipe (Task 3); cada tela e peça pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e local, cobrindo o caminho feliz, cada recusa que muda a tela (`insufficient_balance`, `too_early`, `prescription_cancelled`, `prescription_expired`, `controlled_requires_pharmacist`, `free_text_requires_pharmacist`, `retention_required`, `lot_expired`, `expired_lot`, `invalid_item`, `invalid_quantity` com `path`, `opening_exists`, `already_received`, `same_user`, `not_pending`, `invalid_cbo`, `duplicate_suspected`, `feature_disabled`), o step-up, a ausência de releitura e de armazenamento local (Tasks 4–14). A prova final é no navegador, contra o api real na 3049 (Task 15).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`fb18217` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod27a`.
- Produces: cards F-27.1..F-27.7 em In Progress; worktree `$W` na branch `feat/mod-27a-pharmacy-stock`; a base anotada; as rotas e os limites do api anotados.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-27\\.[1-7] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os sete cards em In Progress ou além. Se algum F-27.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "procedure_billing" | "care_lines" | "territory_registry" | "surveillance" | "care_regulation";\|  care_regulation: "Regulação municipal"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const withQuery = \|^export interface PatientMedication {\|^  | "clinical_document" | "controlled_notification" | "campaign" | "exam" | "procedure";\|^export interface DuplicateCandidate\|^export interface TerritoryPerson\|^export interface NewPatient\|^export async function listMemberships\|^export async function removeNurseItem'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "regulation-queue" | "regulation-returned" | "regulation-offers" | "regulation-settings";\|      if (item.id === "regulation-settings") return regOn && isAdmin;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 14\|NAV_GROUPS.length - 13'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '  IconArrowsSplit2, IconListNumbers, IconArrowBackUp, IconBuildingStore, IconClockHour4\|    "regulation-settings": { icon: IconClockHour4'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "regulation-settings": return <RegulationSettings'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/regulation(/|$)": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '  { role: "care_regulator", label: "Regulador de encaminhamentos" },\|^const PRIVILEGED_ROLES'
$G -C $R/apps/dashboard show origin/main:src/lib/team.test.ts | grep -n 'oferece os 15 papéis'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '^export function timelineKinds\|  { key: "procedure", label: "Procedimento" }\|  recorded: "registrado"'
$G -C $R/apps/dashboard show origin/main:src/modules/timeline/PatientTimeline.tsx | grep -n 'const kinds = timelineKinds(hasFeature(user, "exam_tracking"), hasFeature(user, "procedure_billing"));'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/MedicationsBlock.tsx | grep -n '{m.continuous && <Tag mono={false}>uso contínuo</Tag>}'
$G -C $R/apps/dashboard show origin/main:src/modules/territory/PersonStart.tsx | grep -n '^export interface PersonSource\|^export function PersonStart'
$G -C $R/apps/dashboard show origin/main:src/lib/lab.ts | grep -n '^export const DOT_IN_NUMBER'
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^export const NOMINAL_QUERY\|^export const STATIC_QUERY'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod27a -b feat/mod-27a-pharmacy-stock origin/main
cd $R/.worktrees/dashboard-mod27a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod27a
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(BuildingWarehouse|Prescription|FilePlus|Transfer|TruckLoading|ListSearch|Scale|AlertTriangle|BuildingCog)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os nove ícones (falta algum → escolha outro que exista e use o mesmo na Task 3; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 15 espera a base mais **14 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 27a** (informativo para as Tasks 1–14)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-27a-pharmacy-stock
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod27a api bin/rails routes 2>/dev/null | grep -E " /pharmacy/" | head -60
grep -n 'PRIVILEGED_ROLES' -A3 $R/apps/api/.claude/mod27a/app/models/membership.rb
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-29-module-27a-pharmacy-stock-api.md 2>/dev/null | grep -n "^## \|^### Task\|Valores fixados\|Divergência" | head -60
```

Expected: a branch e as rotas (compare com o contrato §3–§9, com o plano do api e com as Divergências D1–D25: diferença de forma vira ajuste em `src/lib/api.ts` e `src/test/pharmacyFixtures.ts` antes da prova, com nota para a Task 15). Os limites de texto e de quantidade do api, se diferentes dos de "Global Constraints", **vencem**: ajuste `src/lib/pharmacy.ts` e o teste da Task 2. Se os três papéis não estiverem todos em `PRIVILEGED_ROLES` do api (D19), a Task 3 espelha o que o api fizer. Anote como ficaram as abertas.

---

### Task 1: Tipos, cliente HTTP, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo; `PatientMedication` e `TimelineKind` ganham um campo/valor), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/pharmacyFixtures.ts`
- Test: `src/lib/api.pharmacy.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `DuplicateCandidate`, `DuplicateQuery`, `TerritoryPerson`, `NewPatient` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`):
  - tipos `PharmacyRole = "pharmacist" | "pharmacy_assistant" | "stock_keeper"`, `StockLocationKind = "central" | "unit"`, `StockMovementKind` (os 10 do §1), `TransferStatus`, `AdjustmentStatus`, `AdjustmentKind = "adjustment" | "loss" | "expiry"`, `PrescriptionKind = "digital" | "external"`;
  - `StockItem { catalog_item_id: string; catmat_code: string; name: string; unit: string; controlled: boolean; controlled_list: string | null }`; `StockLot { id; item: StockItem; lot_number; expires_on; manufacturer: string | null }`; `LocationRef { id; name; kind: StockLocationKind | string }`;
  - `PharmacyLocation extends LocationRef { health_unit: { id; name } | null; active: boolean; roles: string[] }` (D1); `LocationStaff { user_id; name: string | null; role: PharmacyRole | string }`; `AdminLocation extends LocationRef { health_unit; active; staff: LocationStaff[] }`; `AdminLocationInput { kind: StockLocationKind; health_unit_id?: string | null; name: string; active: boolean }`; `PharmacySettings { common_validity_days; continuous_validity_days; continuous_interval_days; expiry_alert_days; adjustment_approval_percent }` (números); `StockMinimum { catalog_item_id; minimum: string; item?: StockItem }`;
  - `LotBalance { lot: StockLot; balance: string; in_transit: string }`; `StockPosition` (a `<position>` do §2); `PositionQuery { locationId?: string | null; q?: string | null; catalogItemId?: string | null; expiring?: boolean; belowMinimum?: boolean; before?: string | null }`; `PositionPage { positions; next_before }`; `StockMovement { id; kind: StockMovementKind | string; quantity: string; document: { type: string; id: string | null; number: string | null } | null; reason?: string | null; by: { name: string } | null; at: string }` (a forma do `Pharmacy::Statement` do plano do api); `MovementPage { movements; next_before }`;
  - `PackageInput { units_per_package: number; packages: number }`; `ReceiptItemInput { catalog_item_id; lot_number; expires_on; manufacturer?: string; quantity: string; package?: PackageInput }`; `ReceiptInput { location_id; document: { kind: "invoice" | "donation"; number: string; supplier: string }; items: ReceiptItemInput[] }`; `OpeningInput { location_id; items: ReceiptItemInput[] }` (D8); `StockReceipt { id; location: LocationRef; kind: string; items: { lot: StockLot; quantity: string }[]; at: string }`;
  - `TransferItem { lot: StockLot; quantity_sent: string; quantity_received: string | null; reason: string | null }`; `StockTransfer { id; from: LocationRef; to: LocationRef; status: TransferStatus | string; sent_at; sent_by: { name } | null; received_at: string | null; received_by: { name } | null; items: TransferItem[] }`; `TransferInput { to_location_id; items: { lot_id; quantity: string }[] }`; `ReceiveInput { items: { lot_id; quantity_received: string; reason?: string }[] }`; `TransferQuery { locationId; status?: string | null; before?: string | null }`; `TransferPage { transfers; next_before }`;
  - `StockAdjustment { id; location: LocationRef; lot: StockLot; kind: AdjustmentKind | string; status: AdjustmentStatus | string; balance_before: string; counted_quantity: string | null; quantity: string; reason: string; by: { user_id: string; name: string } | null; decided_by: { name: string } | null; decision_reason: string | null; at: string }`; `AdjustmentInput { location_id; lot_id; kind: AdjustmentKind; counted_quantity?: string; quantity?: string; reason: string }`; `AdjustmentPage { adjustments; next_before }`;
  - `PharmacyPatient { id; display_name: string | null; has_cpf: boolean; cns_masked: string | null }`; `PrescriptionItemView` (a do §2 mais `valid_until?: string` e `expired?: boolean` por item — P4 do plano do api) e `PrescriptionView` (a do §2; quantidades string); `PrescriptionLookupInput = { location_id: string; short_code: string } | { location_id: string; patient_id: string }`; `PrescriberInput { name; council; council_state; council_number }`; `ExternalItemInput { catalog_item_id; quantity: string; quantity_unit; dosage_instructions; continuous: boolean }`; `ExternalPrescriptionInput = ({ patient_id: string } | { new_patient: NewPatient; duplicates_reviewed?: boolean }) & { prescriber: PrescriberInput; issued_on: string; notification_number?: string; items: ExternalItemInput[] }` (D4);
  - `DispensationItemInput { prescription_item_id: string; catalog_item_id?: string; lot_id: string; quantity: string; complete?: boolean }` (D11); `DispensationInput { location_id; prescription: { kind: PrescriptionKind; id: string }; items: DispensationItemInput[]; retention?: { notification_number?: string } }` (a tela manda `{}`: o número retido é o do documento — D14); `DispensedItem { prescription_item_id: string | null; catalog_item_id; item?: StockItem; lot_id; lot?: { id; lot_number; expires_on }; quantity: string; mapped_from_free_text: boolean }` (D12); `Dispensation` (a `<dispensation>` do §2 com `DispensedItem`); `DispensationPage { dispensations; next_before }`;
  - `PharmacyAlerts { expiring: number; expired: number; below_minimum: number; in_transit: number; pending_adjustments: number }` (contagens do local; D15); `PharmacyPerson extends TerritoryPerson { has_cpf: boolean; cns_masked: string | null }`;
  - constante `PHARMACY_PAGE_SIZE = 50`;
  - funções: `listMyPharmacyLocations(): Promise<PharmacyLocation[]>`, `listAdminLocations(): Promise<AdminLocation[]>`, `createAdminLocation(input: AdminLocationInput): Promise<AdminLocation>`, `updateAdminLocation(id: string, input: AdminLocationInput): Promise<AdminLocation>`, `addLocationStaff(locationId: string, userId: string, role: PharmacyRole): Promise<AdminLocation>`, `removeLocationStaff(locationId: string, userId: string, role: string): Promise<void>`, `getPharmacySettings(): Promise<PharmacySettings>`, `updatePharmacySettings(s: PharmacySettings): Promise<PharmacySettings>`, `getMinimums(locationId: string): Promise<StockMinimum[]>`, `putMinimums(locationId: string, rows: { catalog_item_id: string; minimum: string }[]): Promise<StockMinimum[]>`, `searchPharmacyItems(q: string): Promise<StockItem[]>`, `recordReceipt(input: ReceiptInput): Promise<StockReceipt>`, `recordOpening(input: OpeningInput): Promise<StockReceipt>`, `listPositions(q: PositionQuery): Promise<PositionPage>`, `listLotMovements(lotId: string, locationId: string, before: string | null): Promise<MovementPage>`, `sendTransfer(input: TransferInput): Promise<StockTransfer>`, `receiveTransfer(id: string, input: ReceiveInput): Promise<StockTransfer>`, `listTransfers(q: TransferQuery): Promise<TransferPage>`, `recordAdjustment(input: AdjustmentInput): Promise<StockAdjustment>`, `approveAdjustment(id: string): Promise<StockAdjustment>`, `rejectAdjustment(id: string, reason: string): Promise<StockAdjustment>`, `listAdjustments(q: { locationId: string; status?: string | null; before?: string | null }): Promise<AdjustmentPage>`, `lookupPrescriptions(input: PrescriptionLookupInput): Promise<PrescriptionView[]>`, `createExternalPrescription(input: ExternalPrescriptionInput): Promise<PrescriptionView>`, `recordDispensation(input: DispensationInput): Promise<Dispensation>`, `listDispensations(patientId: string, before: string | null): Promise<DispensationPage>`, `getPharmacyAlerts(locationId: string): Promise<PharmacyAlerts>`, `lookupPharmacyPerson(cpf: string): Promise<PharmacyPerson>`, `findPharmacyDuplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>`;
  - `PatientMedication` ganha `last_dispensed_on?: string | null` (D23); `TimelineKind` ganha `"dispensation"` (D22);
  - `FeatureKey` ganha `pharmacy`.
- Produces (em `src/test/pharmacyFixtures.ts`): `TODAY27`, `CENTRAL`, `UBS_FARM`, `CENTRAL_REF`, `UBS_REF`, `AMOXI`, `CLONA`, `phUser(roles?, over?)`, `stockLot(over?)`, `lotBalance(over?)`, `position(over?)`, `rxItem(over?)`, `prescription(over?)`, `dispensation(over?)`, `transfer(over?)`, `adjustment(over?)`, `phSettings(over?)`, `adminLocation(over?)`, `phAlerts(over?)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.pharmacy.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  addLocationStaff, approveAdjustment, createAdminLocation, createExternalPrescription, findPharmacyDuplicates, getMinimums, getPharmacyAlerts,
  getPharmacySettings, listAdjustments, listAdminLocations, listDispensations, listLotMovements, listMyPharmacyLocations, listPositions,
  listTransfers, lookupPharmacyPerson, lookupPrescriptions, putMinimums, receiveTransfer, recordAdjustment, recordDispensation, recordOpening,
  recordReceipt, rejectAdjustment, removeLocationStaff, searchPharmacyItems, sendTransfer, updateAdminLocation, updatePharmacySettings
} from "./api";

function ok(body: unknown): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
}
// Código da receita, CPF, nome, profissional, posologia, motivo e número de notificação: nunca em URL.
const SENSITIVE = /ABCDE|23456|12345678909|Joana|Helena|cápsula de 8|quebra|contagem|B1-0042|retid/i;
const ITEM = { catalog_item_id: "ci1", lot_number: "AMX2611", expires_on: "2026-11-20", quantity: "300", package: { units_per_package: 30, packages: 10 } };

describe("cliente da farmácia (contrato §3–§9)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(() => ok({})); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => (fetchMock.mock.calls[i][1] ?? {}) as RequestInit;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));
  const writeOf = (i = 0) => [ urlOf(i).pathname, initOf(i).method, (initOf(i).headers as Record<string, string>)["Content-Type"] ];

  it("locais, equipe, configurações e mínimos: envelopes e JSON em toda escrita", async () => {
    fetchMock.mockImplementationOnce(() => ok({ locations: [ { id: "l2" } ] })).mockImplementationOnce(() => ok({}));
    expect(await listMyPharmacyLocations()).toEqual([ { id: "l2" } ]);
    expect(urlOf(0).pathname).toBe("/pharmacy/locations");
    expect(await listAdminLocations()).toEqual([]);
    expect(urlOf(1).pathname).toBe("/pharmacy/admin/locations");
    await createAdminLocation({ kind: "unit", health_unit_id: "u1", name: "Farmácia UBS Centro", active: true });
    expect(writeOf(2)).toEqual([ "/pharmacy/admin/locations", "POST", "application/json" ]);
    expect(bodyOf(2)).toEqual({ kind: "unit", health_unit_id: "u1", name: "Farmácia UBS Centro", active: true });
    await updateAdminLocation("l2", { kind: "unit", health_unit_id: "u1", name: "Farmácia UBS Centro", active: false });
    expect(writeOf(3)).toEqual([ "/pharmacy/admin/locations/l2", "PATCH", "application/json" ]);
    await addLocationStaff("l2", "u-9", "pharmacist");
    expect([ ...writeOf(4), bodyOf(4) ]).toEqual([ "/pharmacy/admin/locations/l2/staff", "POST", "application/json", { user_id: "u-9", role: "pharmacist" } ]);
    fetchMock.mockImplementationOnce(() => Promise.resolve(new Response(null, { status: 204 })));
    await removeLocationStaff("l2", "u-9", "pharmacist");
    expect([ ...writeOf(5), bodyOf(5) ]).toEqual([ "/pharmacy/admin/locations/l2/staff", "DELETE", "application/json", { user_id: "u-9", role: "pharmacist" } ]);
    await getPharmacySettings();
    expect(urlOf(6).pathname).toBe("/pharmacy/admin/settings");
    const S = { common_validity_days: 30, continuous_validity_days: 180, continuous_interval_days: 25, expiry_alert_days: 90, adjustment_approval_percent: 10 };
    await updatePharmacySettings(S);
    expect([ ...writeOf(7), bodyOf(7) ]).toEqual([ "/pharmacy/admin/settings", "PATCH", "application/json", S ]);
    fetchMock.mockImplementationOnce(() => ok({ minimums: [ { catalog_item_id: "ci1", minimum: "50" } ] }));
    expect(await getMinimums("l2")).toEqual([ { catalog_item_id: "ci1", minimum: "50" } ]);
    expect(urlOf(8).pathname).toBe("/pharmacy/locations/l2/minimums");
    await putMinimums("l2", [ { catalog_item_id: "ci1", minimum: "60" } ]);
    expect([ ...writeOf(9), bodyOf(9) ]).toEqual([ "/pharmacy/locations/l2/minimums", "PUT", "application/json", { minimums: [ { catalog_item_id: "ci1", minimum: "60" } ] } ]);
  });

  it("busca no catálogo no corpo; entrada e abertura com a forma do §4", async () => {
    fetchMock.mockImplementationOnce(() => ok({ items: [ { catalog_item_id: "ci1" } ] }));
    expect(await searchPharmacyItems("amox")).toEqual([ { catalog_item_id: "ci1" } ]);
    expect([ ...writeOf(0), urlOf(0).search, bodyOf(0) ]).toEqual([ "/pharmacy/items/search", "POST", "application/json", "", { q: "amox" } ]);
    const RECEIPT = { location_id: "l1", document: { kind: "invoice" as const, number: "NF 1234", supplier: "Distribuidora Sul" }, items: [ ITEM ] };
    await recordReceipt(RECEIPT);
    expect([ ...writeOf(1), bodyOf(1) ]).toEqual([ "/pharmacy/receipts", "POST", "application/json", RECEIPT ]);
    await recordOpening({ location_id: "l2", items: [ ITEM ] });
    expect([ ...writeOf(2), bodyOf(2) ]).toEqual([ "/pharmacy/openings", "POST", "application/json", { location_id: "l2", items: [ ITEM ] } ]);
  });

  it("posição: só os filtros que vieram, com limite; extrato do lote pelo local", async () => {
    fetchMock.mockImplementationOnce(() => ok({ positions: [ { balance: "30" } ], next_before: "c1" }));
    expect(await listPositions({ locationId: "l2", q: "amox", expiring: true, belowMinimum: false, catalogItemId: null, before: "c0" }))
      .toEqual({ positions: [ { balance: "30" } ], next_before: "c1" });
    expect(urlOf(0).pathname).toBe("/pharmacy/positions");
    expect(Object.fromEntries(urlOf(0).searchParams)).toEqual({ location_id: "l2", q: "amox", expiring: "true", below_minimum: "false", before: "c0", limit: "50" });
    expect(await listPositions({ catalogItemId: "ci1" })).toEqual({ positions: [], next_before: null });
    expect(Object.fromEntries(urlOf(1).searchParams)).toEqual({ catalog_item_id: "ci1", limit: "50" });
    expect(await listLotMovements("lot1", "l2", null)).toEqual({ movements: [], next_before: null });
    expect([ urlOf(2).pathname, Object.fromEntries(urlOf(2).searchParams) ]).toEqual([ "/pharmacy/lots/lot1/movements", { location_id: "l2", limit: "50" } ]);
  });

  it("transferências e ajustes", async () => {
    await sendTransfer({ to_location_id: "l2", items: [ { lot_id: "lot1", quantity: "100" } ] });
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/pharmacy/transfers", "POST", "application/json", { to_location_id: "l2", items: [ { lot_id: "lot1", quantity: "100" } ] } ]);
    await receiveTransfer("t1", { items: [ { lot_id: "lot1", quantity_received: "90", reason: "uma caixa chegou quebrada" } ] });
    expect([ ...writeOf(1), bodyOf(1).items[0].quantity_received ]).toEqual([ "/pharmacy/transfers/t1/receive", "POST", "application/json", "90" ]);
    expect(await listTransfers({ locationId: "l2", status: "sent" })).toEqual({ transfers: [], next_before: null });
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ location_id: "l2", status: "sent", limit: "50" });
    await recordAdjustment({ location_id: "l2", lot_id: "lot1", kind: "adjustment", counted_quantity: "20", reason: "contagem do fim do mês" });
    expect([ ...writeOf(3), bodyOf(3).counted_quantity ]).toEqual([ "/pharmacy/adjustments", "POST", "application/json", "20" ]);
    await approveAdjustment("a1");
    expect([ ...writeOf(4), bodyOf(4) ]).toEqual([ "/pharmacy/adjustments/a1/approve", "POST", "application/json", {} ]);
    await rejectAdjustment("a1", "contagem refeita, saldo confere");
    expect([ ...writeOf(5), bodyOf(5) ]).toEqual([ "/pharmacy/adjustments/a1/reject", "POST", "application/json", { reason: "contagem refeita, saldo confere" } ]);
    expect(await listAdjustments({ locationId: "l2", status: "pending" })).toEqual({ adjustments: [], next_before: null });
    expect([ urlOf(6).pathname, Object.fromEntries(urlOf(6).searchParams) ]).toEqual([ "/pharmacy/adjustments", { location_id: "l2", status: "pending", limit: "50" } ]);
  });

  it("dispensação: receita pelo código ou paciente, externa, dispensar e histórico", async () => {
    fetchMock.mockImplementationOnce(() => ok({ prescriptions: [ { id: "doc1" } ] }));
    expect(await lookupPrescriptions({ location_id: "l2", short_code: "ABCDE23456" })).toEqual([ { id: "doc1" } ]);
    expect([ ...writeOf(0), urlOf(0).search, bodyOf(0) ]).toEqual([ "/pharmacy/prescriptions/lookup", "POST", "application/json", "", { location_id: "l2", short_code: "ABCDE23456" } ]);
    expect(await lookupPrescriptions({ location_id: "l2", patient_id: "pa1" })).toEqual([]);
    const EXT = { patient_id: "pa1", prescriber: { name: "Dra. Helena Prado", council: "CRM", council_state: "PR", council_number: "12345" }, issued_on: "2026-10-28",
      items: [ { catalog_item_id: "ci1", quantity: "21", quantity_unit: "cápsula", dosage_instructions: "1 cápsula de 8 em 8 horas", continuous: false } ] };
    await createExternalPrescription(EXT);
    expect([ ...writeOf(2), bodyOf(2) ]).toEqual([ "/pharmacy/external_prescriptions", "POST", "application/json", EXT ]);
    const DISP = { location_id: "l2", prescription: { kind: "digital" as const, id: "doc1" }, items: [ { prescription_item_id: "pi1", lot_id: "lot1", quantity: "21", complete: true } ],
      retention: {} };
    await recordDispensation(DISP);
    expect([ ...writeOf(3), bodyOf(3) ]).toEqual([ "/pharmacy/dispensations", "POST", "application/json", DISP ]);
    expect(await listDispensations("pa1", "c0")).toEqual({ dispensations: [], next_before: null });
    expect([ urlOf(4).pathname, Object.fromEntries(urlOf(4).searchParams) ]).toEqual([ "/pharmacy/dispensations", { patient_id: "pa1", before: "c0", limit: "50" } ]);
  });

  it("pessoa e alertas: busca no corpo; listas ausentes viram vazias", async () => {
    fetchMock.mockImplementationOnce(() => ok({ patient_id: "pa1", display_name: "Joana Lima", has_cpf: true, cns_masked: null }))
      .mockImplementationOnce(() => ok({ candidates: [ { patient_id: "pa1", display_name: "Joana Lima", birth_date: "1980-02-03", has_cpf: false } ] }))
      .mockImplementationOnce(() => ok({ expiring: 2 }));
    const person = await lookupPharmacyPerson("12345678909");
    expect([ person.patient_id, person.citizen_id, person.registration_id ]).toEqual([ "pa1", null, null ]);
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/pharmacy/person_lookup", "POST", "application/json", { cpf: "12345678909" } ]);
    expect((await findPharmacyDuplicates({ full_name: "Joana Lima", birth_date: "1980-02-03" }))[0].registration_id).toBeNull();
    expect(urlOf(1).pathname).toBe("/pharmacy/patients/duplicates");
    expect(await getPharmacyAlerts("l2")).toEqual({ expiring: 2, expired: 0, below_minimum: 0, in_transit: 0, pending_adjustments: 0 });
    expect([ urlOf(2).pathname, urlOf(2).search ]).toEqual([ "/pharmacy/alerts", "?location_id=l2" ]);
  });

  it("nada sensível na URL: código, CPF, nome, receita, motivo e notificação só no corpo", async () => {
    await lookupPrescriptions({ location_id: "l2", short_code: "ABCDE23456" });
    await lookupPharmacyPerson("12345678909");
    await createExternalPrescription({ patient_id: "pa1", prescriber: { name: "Dra. Helena Prado", council: "CRM", council_state: "PR", council_number: "12345" },
      issued_on: "2026-10-28", notification_number: "B1-0042", items: [ { catalog_item_id: "ci2", quantity: "30", quantity_unit: "comprimido",
        dosage_instructions: "1 cápsula de 8 em 8 horas", continuous: true } ] });
    await recordDispensation({ location_id: "l2", prescription: { kind: "external", id: "ex1" }, items: [ { prescription_item_id: "pi1", lot_id: "lot1", quantity: "30" } ],
      retention: {} });
    await receiveTransfer("t1", { items: [ { lot_id: "lot1", quantity_received: "90", reason: "uma caixa chegou quebrada" } ] });
    await recordAdjustment({ location_id: "l2", lot_id: "lot1", kind: "loss", quantity: "2", reason: "contagem: frasco retido quebrado" });
    for (const call of fetchMock.mock.calls) expect(decodeURIComponent(String(call[0]))).not.toMatch(SENSITIVE);
  });
});
```

E no fim de `src/lib/features.test.ts`:

```ts
describe("módulo 27a", () => {
  it("pharmacy tem nome em português", () => {
    expect(featureLabel("pharmacy")).toBe("Farmácia e almoxarifado");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/lib/api.pharmacy.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções não existem e o rótulo é a chave crua.

- [ ] **Step 3: Implemente a seção "Farmácia e almoxarifado (módulo 27a)" no fim de `src/lib/api.ts`**

No molde da seção do 26a: `PHARMACY_BASE = import.meta.env.VITE_PHARMACY_BASE || "/pharmacy"`, `phPath(...parts)` com `encodeURIComponent` em cada parte, `phWrite(method: "POST" | "PATCH" | "PUT" | "DELETE", body)` sempre com corpo (`{}` na aprovação; `{ user_id, role }` no `DELETE` da equipe; `{ minimums }` no `PUT` dos mínimos). Busca no catálogo (`POST /pharmacy/items/search`), busca da receita e da pessoa e duplicados são `POST` com o termo no corpo. Envelopes: `locations`, `minimums`, `items`, `positions`, `movements`, `transfers`, `adjustments`, `prescriptions`, `dispensations`, `candidates` (ausente → `[]`; os candidatos ganham `registration_id: null`, como no 25); `lookupPharmacyPerson` completa `citizen_id: null` e `registration_id: null` (o api cria o paciente do par validado — P3 do plano do api); páginas com `next_before ?? null`; alertas com `?? 0` em cada contagem; `204` do `DELETE` vira `void`. `positionParams(q)` põe, nesta ordem e só se vierem, `location_id`, `q`, `catalog_item_id`, `expiring`/`below_minimum` (só quando booleano, `"true"`/`"false"`), `before`, e sempre `limit=50`; `listTransfers`, `listAdjustments`, `listLotMovements` e `listDispensations` seguem a mesma regra com os seus filtros. Na seção do 19c, `PatientMedication` ganha `last_dispensed_on?: string | null` (comentário: "Módulo 27a (D23): última retirada na farmácia, só leitura"); na do 20, `TimelineKind` ganha `| "dispensation"`. Os tipos da lista de Interfaces, com `| string` nos códigos que vêm do api. Em `features.ts`, `pharmacy` no `FeatureKey` e no `FEATURE_LABEL` (`// Módulo 27a (ADR 0045): sem pré-requisito.` + `pharmacy: "Farmácia e almoxarifado"`). Em `vite.config.ts`, depois do proxy de `/regulation`: `"^/pharmacy(/|$)": proxy(TARGET)` e, no comentário do topo, `//   /pharmacy   → farmácia e almoxarifado (módulo 27a: locais, estoque, transferências, ajustes e dispensação).`.

Crie `src/test/pharmacyFixtures.ts` (hoje: quinta, 2026-10-29; dados ilustrativos):

```ts
import type {
  AdminLocation, Dispensation, LocationRef, LotBalance, PharmacyAlerts, PharmacyLocation, PharmacySettings, PrescriptionItemView, PrescriptionView,
  SessionUser, StockAdjustment, StockItem, StockLot, StockPosition, StockTransfer
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY27 = "2026-10-29";
export const CENTRAL_REF: LocationRef = { id: "l1", name: "Almoxarifado Central", kind: "central" };
export const UBS_REF: LocationRef = { id: "l2", name: "Farmácia UBS Centro", kind: "unit" };
export const CENTRAL: PharmacyLocation = { ...CENTRAL_REF, health_unit: null, active: true, roles: [ "stock_keeper" ] };
export const UBS_FARM: PharmacyLocation = { ...UBS_REF, health_unit: { id: "u1", name: "UBS Centro" }, active: true, roles: [ "pharmacist" ] };
export const AMOXI: StockItem = { catalog_item_id: "ci1", catmat_code: "BR0267645", name: "Amoxicilina 500 mg cápsula", unit: "cápsula", controlled: false, controlled_list: null };
export const CLONA: StockItem = { catalog_item_id: "ci2", catmat_code: "BR0268370", name: "Clonazepam 2 mg comprimido", unit: "comprimido", controlled: true, controlled_list: "B1" };

export function phUser(roles: string[] = [ "pharmacist" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { features: [ "pharmacy" ], ...over });
}
export function stockLot(over: Partial<StockLot> = {}): StockLot {
  return { id: "lot1", item: AMOXI, lot_number: "AMX2611", expires_on: "2026-11-20", manufacturer: "EMS", ...over };
}
export function lotBalance(over: Partial<LotBalance> = {}): LotBalance { return { lot: stockLot(), balance: "30", in_transit: "0", ...over }; }
export function position(over: Partial<StockPosition> = {}): StockPosition {
  return { location: UBS_REF, item: AMOXI, lots: [ lotBalance() ], balance: "30", minimum: "50", below_minimum: true, expiring_soon: true, ...over };
}
export function rxItem(over: Partial<PrescriptionItemView> = {}): PrescriptionItemView {
  return {
    id: "pi1", catalog_item_id: "ci1", free_text: null, printed_description: "Amoxicilina 500 mg cápsula", quantity: "21", quantity_unit: "cápsula",
    dosage_instructions: "1 cápsula de 8 em 8 horas por 7 dias", continuous: false, controlled: false, dispensed_total: "0",
    last_dispensed_on: null, next_allowed_on: null, completed: false, ...over
  };
}
export function prescription(over: Partial<PrescriptionView> = {}): PrescriptionView {
  return {
    kind: "digital", id: "doc1", short_code: "ABCDE23456", issued_on: "2026-10-27", valid_until: "2026-11-26", expired: false, cancelled: false,
    prescriber: { name: "Dra. Helena Prado", council: "CRM-PR 12345" }, patient: { id: "pa1", display_name: "Joana Lima", has_cpf: true, cns_masked: null },
    items: [ rxItem() ], ...over
  };
}
export function dispensation(over: Partial<Dispensation> = {}): Dispensation {
  return {
    id: "dp1", location: UBS_REF, prescription: { kind: "digital", id: "doc1" }, patient: { id: "pa1", display_name: "Joana Lima", has_cpf: true, cns_masked: null },
    by: { name: "Marta Souza", role: "pharmacist" }, at: "2026-10-29T10:15:00-03:00",
    items: [ { prescription_item_id: "pi1", catalog_item_id: "ci1", item: AMOXI, lot_id: "lot1", lot: { id: "lot1", lot_number: "AMX2611", expires_on: "2026-11-20" },
      quantity: "21", mapped_from_free_text: false } ],
    retention: null, ...over
  };
}
export function transfer(over: Partial<StockTransfer> = {}): StockTransfer {
  return {
    id: "t1", from: CENTRAL_REF, to: UBS_REF, status: "sent", sent_at: "2026-10-28T09:00:00-03:00", sent_by: { name: "Paulo Almeida" },
    received_at: null, received_by: null, items: [ { lot: stockLot(), quantity_sent: "100", quantity_received: null, reason: null } ], ...over
  };
}
export function adjustment(over: Partial<StockAdjustment> = {}): StockAdjustment {
  return {
    id: "a1", location: UBS_REF, lot: stockLot(), kind: "adjustment", status: "pending", balance_before: "30", counted_quantity: "20", quantity: "-10",
    reason: "Contagem do fim do mês", by: { user_id: "u-9", name: "Paulo Almeida" }, decided_by: null, decision_reason: null,
    at: "2026-10-29T08:00:00-03:00", ...over
  };
}
export function phSettings(over: Partial<PharmacySettings> = {}): PharmacySettings {
  return { common_validity_days: 30, continuous_validity_days: 180, continuous_interval_days: 25, expiry_alert_days: 90, adjustment_approval_percent: 10, ...over };
}
export function adminLocation(over: Partial<AdminLocation> = {}): AdminLocation {
  return { ...UBS_REF, health_unit: { id: "u1", name: "UBS Centro" }, active: true,
    staff: [ { user_id: "u-9", name: "Marta Souza", role: "pharmacist" } ], ...over };
}
export function phAlerts(over: Partial<PharmacyAlerts> = {}): PharmacyAlerts {
  return { expiring: 2, expired: 1, below_minimum: 1, in_transit: 1, pending_adjustments: 1, ...over };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/lib/api.pharmacy.test.ts src/lib/features.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.pharmacy.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/pharmacyFixtures.ts
$G commit -m "feat: add the pharmacy stock api client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/pharmacy.ts`)

**Files:**
- Create: `src/lib/pharmacy.ts`
- Test: `src/lib/pharmacy.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `errorCode`, tipos da Task 1; `hasFeature`, `featureDisabledKey` (`features.ts`); `NOMINAL_QUERY`, `STATIC_QUERY` (`careLines.ts`); `DOT_IN_NUMBER` (`lab.ts`); `fmtDay` (`exams.ts`).
- Produces:
  - frases `PHARMACY_DISABLED = "a farmácia e o almoxarifado estão desligados nesta cidade"`, `PHARMACY_NO_ROLE = "seu papel não permite usar a farmácia"`, `NO_LOCATION = "você não está vinculado a nenhum local de estoque — peça ao administrador da cidade"`, `CONTROLLED_PHARMACIST_ONLY = "medicamento controlado: só o farmacêutico dispensa"`, `FREE_TEXT_PHARMACIST_ONLY = "item em texto livre: só o farmacêutico dispensa, escolhendo o item do catálogo"`, `ITEM_COMPLETED = "item já dispensado por completo"`, `RETENTION_REQUIRED = "medicamento controlado: registre a retenção da receita"`, `NOTHING_TO_DISPENSE = "informe a quantidade de ao menos um item"`, `NO_PATIENT_RECORD = "esta pessoa ainda não tem receita no Rota Saúde — se trouxe receita de fora da rede, use Receita externa"`, `BALANCE_CHANGED = "o saldo do lote mudou — os lotes foram relidos; confira a quantidade"`, `PRESCRIPTION_CHANGED = "a receita mudou enquanto você dispensava — busque a receita de novo"`;
  - `PHARMACY_ALERTS_REFRESH_MS = 60_000`; chaves `PH_LOCATIONS_KEY`, `PH_ADMIN_KEY`, `PH_SETTINGS_KEY`, `PH_MINIMUMS_KEY`, `PH_POSITIONS_KEY`, `PH_MOVEMENTS_KEY`, `PH_TRANSFERS_KEY`, `PH_ADJUSTMENTS_KEY`, `PH_DISPENSATIONS_KEY`, `PH_ALERTS_KEY`, `PH_ITEMS_KEY` (strings `"pharmacy…"`); re-export de `NOMINAL_QUERY`, `STATIC_QUERY`;
  - `canUsePharmacy(user): boolean` (interruptor e nunca o operador); `pharmacyRoles(user): { pharmacist; assistant; keeper; admin }`; `locationRoles(loc: PharmacyLocation): { pharmacist; assistant; keeper }` (pelo `roles`);
  - rótulos `locationKindLabel`, `movementLabel`, `transferStatusLabel` (+ `transferStatusTone`), `adjustmentStatusLabel` (+ tone), `adjustmentKindLabel`, `pharmacyRoleLabel`, `COUNCILS = [ "CRM", "CRO", "COREN" ]`, `UFS` (as 27 siglas);
  - quantidade: `quantityProblem(raw: string, label?: string, opts?: { allowZero?: boolean }): string | null`, `toApiQuantity(raw: string): string`, `fmtQuantity(api: string | null | undefined): string`, `fmtSigned(api: string): string`, `cents(api: string): number`;
  - `normalizeShortCode(raw: string): string | null`;
  - lotes: `availableLots(p: StockPosition | undefined, today: string): LotBalance[]` (saldo > 0, não vencido, validade crescente, depois número do lote), `suggestedLotId(p: StockPosition | undefined, today: string): string` (o id do primeiro de `availableLots`, ou `""`), `lotsWithBalance(p): LotBalance[]` (saldo > 0, inclusive vencido — para perda e vencimento), `lotOptionLabel(b: LotBalance, suggested: boolean): string`;
  - receita: `prescriptionBlock(p: PrescriptionView): string | null`, `itemBlock(i: PrescriptionItemView, ctx: { pharmacist: boolean; today: string }): string | null`;
  - dispensação: `DispenseLine { catalogItemId: string | null; lotId: string; quantity: string; complete: boolean }`, `DispenseDraft { lines: Record<string, DispenseLine>; retained: boolean }`, `EMPTY_DISPENSE: DispenseDraft`, `filledLines(d)`, `needsRetention(d: DispenseDraft, p: PrescriptionView): boolean`, `dispenseProblems(d, p, ctx: { pharmacist: boolean; today: string; balances: Record<string, string> }): { path: string; text: string }[]`, `dispenseBody(d, p, locationId): DispensationInput`;
  - entrada: `ReceiptLine { key: string; item: StockItem | null; lotNumber; expiresOn; manufacturer; byPackage: boolean; quantity; unitsPerPackage; packages }`, `newReceiptLine(key: string): ReceiptLine`, `lineQuantity(l: ReceiptLine): string | null` (o total em formato do api, ou null se não dá para calcular), `receiptProblems(lines, today, doc: { kind: "invoice" | "donation" | "opening"; number: string; supplier: string }): { path: string; text: string }[]`, `receiptItems(lines): ReceiptItemInput[]`;
  - transferência e recebimento: `TransferLine { key; position: StockPosition | null; lotId: string; quantity: string }`, `transferProblems(lines, today)`, `ReceiveLine { lotId: string; received: string; reason: string }`, `receiveProblems(lines: ReceiveLine[], t: StockTransfer)`, `receiveBody(lines): ReceiveInput`, `hasDifference(lines, t): boolean`;
  - ajuste: `AdjustmentDraft { lotId: string; kind: AdjustmentKind | ""; amount: string; reason: string }`, `adjustmentProblem(d, balance: string): string | null`, `adjustmentDelta(d, balance): string` (formato do api, com sinal), `adjustmentBody(d, locationId): AdjustmentInput`;
  - textos: `reasonProblem(text: string): string | null` (10–500), `notificationNumberProblem(text: string): string | null` (vazio aceito; receita externa), `prescriberProblem(p: PrescriberInput): string | null`, `issuedOnProblem(date: string, today: string): string | null`, `settingsProblem(raw: Record<keyof PharmacySettings, string>): string | null`;
  - recusas: `pharmacyError(err: unknown): string`, `pharmacyProblems(err: unknown): { path: string; text: string }[]` (lê `problems` do corpo, D20; sem ele, `[]`), `errorPath(err): string | null`; `NOTIFICATION_LISTS = [ "A1", "A2", "A3", "B1", "B2" ]` (lista que exige o número da notificação na receita externa).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/pharmacy.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  CONTROLLED_PHARMACIST_ONLY, EMPTY_DISPENSE, FREE_TEXT_PHARMACIST_ONLY, ITEM_COMPLETED, NOTHING_TO_DISPENSE, PHARMACY_DISABLED, RETENTION_REQUIRED,
  adjustmentDelta, adjustmentProblem, availableLots, canUsePharmacy, cents, dispenseBody, dispenseProblems, fmtQuantity, fmtSigned, issuedOnProblem,
  itemBlock, lineQuantity, locationRoles, lotsWithBalance, newReceiptLine, normalizeShortCode, notificationNumberProblem, pharmacyError,
  pharmacyProblems, prescriberProblem, prescriptionBlock, quantityProblem, reasonProblem, receiveProblems, receiptProblems, settingsProblem,
  suggestedLotId, toApiQuantity
} from "./pharmacy";
import { AMOXI, CENTRAL, TODAY27, UBS_FARM, lotBalance, position, prescription, rxItem, stockLot, transfer } from "../test/pharmacyFixtures";

const user = (roles: string[], features: string[] = [ "pharmacy" ], operator = false) => ({ operator, memberships: roles.map((role) => ({ role })), features });

describe("portão e papéis", () => {
  it("interruptor e nunca o operador; papéis pelo local", () => {
    expect(canUsePharmacy(user([ "pharmacist" ]))).toBe(true);
    expect(canUsePharmacy(user([ "pharmacist" ], []))).toBe(false);
    expect(canUsePharmacy(user([], [ "pharmacy" ], true))).toBe(false);
    expect(locationRoles(UBS_FARM)).toEqual({ pharmacist: true, assistant: false, keeper: false });
    expect(locationRoles(CENTRAL)).toEqual({ pharmacist: false, assistant: false, keeper: true });
  });
});

describe("quantidade: vírgula, até 2 casas, maior que zero; recusa em vez de cortar", () => {
  it("o que recusa e o que não recusa", () => {
    expect(quantityProblem("")).toBe("informe a quantidade");
    expect(quantityProblem("1.000")).toBe("use vírgula para decimais e não use ponto de milhar (ex.: 150000 ou 4,5)");
    expect(quantityProblem("2,555")).toBe("a quantidade tem no máximo 2 casas depois da vírgula");
    expect(quantityProblem("0")).toBe("a quantidade precisa ser maior que zero");
    expect(quantityProblem("0", "o mínimo", { allowZero: true })).toBeNull();
    expect(quantityProblem("100000")).toBe("a quantidade pode ser no máximo 99999,99");
    expect(quantityProblem("-1")).toBe("a quantidade: use só dígitos e vírgula");
    expect(quantityProblem("12,5")).toBeNull();
    expect(quantityProblem(" 30 ")).toBeNull();
  });
  it("conversões", () => {
    expect(toApiQuantity("12,5")).toBe("12.5");
    expect(toApiQuantity(" 30 ")).toBe("30");
    expect(fmtQuantity("12.50")).toBe("12,5");
    expect(fmtQuantity("30.00")).toBe("30");
    expect(fmtQuantity("0.25")).toBe("0,25");
    expect(fmtQuantity(null)).toBe("—");
    expect(fmtSigned("-3.5")).toBe("-3,5");
    expect(fmtSigned("30")).toBe("+30");
    expect(cents("12.5")).toBe(1250);
    expect(cents("0.07")).toBe(7);
  });
});

describe("código da receita", () => {
  it("minúsculas, espaço e hífen; 10 do alfabeto sem ambíguos", () => {
    expect(normalizeShortCode("abcde-23456")).toBe("ABCDE23456");
    expect(normalizeShortCode(" ABCDE 23456 ")).toBe("ABCDE23456");
    expect(normalizeShortCode("ABCDE2345")).toBeNull();
    expect(normalizeShortCode("ABCDE2345O")).toBeNull();
    expect(normalizeShortCode("ABCDE23451")).toBeNull();
  });
});

describe("lote sugerido: o que vence primeiro, com saldo e não vencido", () => {
  const P = position({ lots: [
    lotBalance({ lot: stockLot({ id: "B", lot_number: "B", expires_on: "2026-12-01" }), balance: "10" }),
    lotBalance({ lot: stockLot({ id: "A", lot_number: "A", expires_on: "2026-11-15" }), balance: "0" }),
    lotBalance({ lot: stockLot({ id: "C", lot_number: "C", expires_on: "2026-10-28" }), balance: "5" }),
    lotBalance({ lot: stockLot({ id: "D", lot_number: "D", expires_on: "2026-11-20" }), balance: "3" })
  ] });
  it("ordem, sugestão e lotes para perda", () => {
    expect(availableLots(P, TODAY27).map((b) => b.lot.id)).toEqual([ "D", "B" ]);
    expect(suggestedLotId(P, TODAY27)).toBe("D");
    expect(suggestedLotId(undefined, TODAY27)).toBe("");
    expect(lotsWithBalance(P).map((b) => b.lot.id).sort()).toEqual([ "B", "C", "D" ]);
  });
});

describe("o que impede dispensar, na ordem", () => {
  it("receita", () => {
    expect(prescriptionBlock(prescription())).toBeNull();
    expect(prescriptionBlock(prescription({ cancelled: true }))).toBe("receita cancelada — não pode ser dispensada");
    expect(prescriptionBlock(prescription({ expired: true, valid_until: "2026-10-20" }))).toBe("receita vencida em 20/10/2026 — não pode ser dispensada");
  });
  it("item", () => {
    const ph = { pharmacist: true, today: TODAY27 }; const aux = { pharmacist: false, today: TODAY27 };
    expect(itemBlock(rxItem({ completed: true }), ph)).toBe(ITEM_COMPLETED);
    expect(itemBlock(rxItem({ expired: true, valid_until: "2026-10-20" }), ph)).toBe("item vencido em 20/10/2026");
    expect(itemBlock(rxItem({ continuous: true, next_allowed_on: "2026-11-03" }), ph)).toBe("uso contínuo: próxima retirada a partir de 03/11/2026");
    expect(itemBlock(rxItem({ continuous: true, next_allowed_on: TODAY27 }), ph)).toBeNull();
    expect(itemBlock(rxItem({ controlled: true }), aux)).toBe(CONTROLLED_PHARMACIST_ONLY);
    expect(itemBlock(rxItem({ catalog_item_id: null, free_text: "xarope caseiro" }), aux)).toBe(FREE_TEXT_PHARMACIST_ONLY);
    expect(itemBlock(rxItem({ controlled: true }), ph)).toBeNull();
  });
});

describe("dispensação: todas as recusas de uma vez; linha vazia não vai; retenção só com controlado", () => {
  const P = prescription({ items: [ rxItem(), rxItem({ id: "pi2", catalog_item_id: "ci2", printed_description: "Clonazepam 2 mg comprimido", controlled: true }),
    rxItem({ id: "pi3", catalog_item_id: null, free_text: "xarope de guaco", printed_description: "xarope de guaco" }) ] });
  const ctx = { pharmacist: true, today: TODAY27, balances: { lot1: "30", lot2: "60" } };
  it("problemas", () => {
    expect(dispenseProblems(EMPTY_DISPENSE, P, ctx)).toEqual([ { path: "items", text: NOTHING_TO_DISPENSE } ]);
    const d = { ...EMPTY_DISPENSE, lines: {
      pi1: { catalogItemId: "ci1", lotId: "lot1", quantity: "40", complete: false },
      pi2: { catalogItemId: "ci2", lotId: "lot2", quantity: "2,555", complete: false },
      pi3: { catalogItemId: null, lotId: "", quantity: "1", complete: false } } };
    expect(dispenseProblems(d, P, ctx)).toEqual([
      { path: "pi1", text: "Amoxicilina 500 mg cápsula: saída maior que o saldo do lote (30)" },
      { path: "pi2", text: "Clonazepam 2 mg comprimido: a quantidade tem no máximo 2 casas depois da vírgula" },
      { path: "pi3", text: "xarope de guaco: escolha o item do catálogo" },
      { path: "retention", text: RETENTION_REQUIRED }
    ]);
  });
  it("corpo", () => {
    const d = { lines: {
      pi1: { catalogItemId: "ci1", lotId: "lot1", quantity: "21", complete: true },
      pi2: { catalogItemId: "ci2", lotId: "lot2", quantity: "", complete: false },
      pi3: { catalogItemId: "ci9", lotId: "lot9", quantity: "1,5", complete: false } }, retained: false };
    expect(dispenseBody(d, P, "l2")).toEqual({ location_id: "l2", prescription: { kind: "digital", id: "doc1" }, items: [
      { prescription_item_id: "pi1", lot_id: "lot1", quantity: "21", complete: true },
      { prescription_item_id: "pi3", catalog_item_id: "ci9", lot_id: "lot9", quantity: "1.5" } ] });
    const withControlled = { ...d, lines: { ...d.lines, pi2: { catalogItemId: "ci2", lotId: "lot2", quantity: "30", complete: false } }, retained: true };
    expect(dispenseBody(withControlled, P, "l2").retention).toEqual({});
  });
});

describe("entrada, recebimento e ajuste", () => {
  it("entrada: lote vencido não entra; por embalagem multiplica; abertura sem documento", () => {
    const l = { ...newReceiptLine("k1"), item: AMOXI, lotNumber: "AMX2611", expiresOn: "2026-11-20", byPackage: true, unitsPerPackage: "30", packages: "10" };
    expect(lineQuantity(l)).toBe("300");
    expect(lineQuantity({ ...l, unitsPerPackage: "0" })).toBeNull();
    expect(receiptProblems([ l ], TODAY27, { kind: "invoice", number: "NF 1234", supplier: "Distribuidora Sul" })).toEqual([]);
    expect(receiptProblems([ { ...l, expiresOn: "2026-10-28" } ], TODAY27, { kind: "invoice", number: "NF 1234", supplier: "Distribuidora Sul" }))
      .toEqual([ { path: "k1", text: "Amoxicilina 500 mg cápsula: lote vencido — não entra no estoque" } ]);
    expect(receiptProblems([ l ], TODAY27, { kind: "invoice", number: "", supplier: "" }).map((p) => p.path)).toEqual([ "document.number", "document.supplier" ]);
    expect(receiptProblems([ l ], TODAY27, { kind: "opening", number: "", supplier: "" })).toEqual([]);
    expect(receiptProblems([], TODAY27, { kind: "opening", number: "", supplier: "" })).toEqual([ { path: "items", text: "acrescente ao menos um item" } ]);
  });
  it("recebimento: diferença pede motivo de 10 a 500; maior que o enviado recusa", () => {
    const t = transfer();
    expect(receiveProblems([ { lotId: "lot1", received: "100", reason: "" } ], t)).toEqual([]);
    expect(receiveProblems([ { lotId: "lot1", received: "90", reason: "" } ], t)).toEqual([ { path: "lot1", text: "AMX2611: recebido diferente do enviado — escreva o motivo (10 a 500 caracteres)" } ]);
    expect(receiveProblems([ { lotId: "lot1", received: "110", reason: "" } ], t)).toEqual([ { path: "lot1", text: "AMX2611: recebido maior que o enviado (100)" } ]);
    expect(receiveProblems([ { lotId: "lot1", received: "", reason: "" } ], t)).toEqual([ { path: "lot1", text: "AMX2611: informe a quantidade recebida (0 se nada chegou)" } ]);
  });
  it("ajuste: contagem igual ao saldo não é ajuste; perda maior que o saldo recusada; motivo", () => {
    const base = { lotId: "lot1", kind: "adjustment" as const, amount: "28,5", reason: "contagem do fim do mês" };
    expect(adjustmentProblem(base, "30")).toBeNull();
    expect(adjustmentDelta(base, "30")).toBe("-1.5");
    expect(adjustmentDelta({ ...base, amount: "31" }, "30")).toBe("+1");
    expect(adjustmentProblem({ ...base, amount: "30" }, "30")).toBe("a contagem é igual ao saldo — não há o que ajustar");
    expect(adjustmentProblem({ ...base, kind: "loss", amount: "31" }, "30")).toBe("saída maior que o saldo do lote (30)");
    expect(adjustmentProblem({ ...base, reason: "curto" }, "30")).toBe("o motivo tem de 10 a 500 caracteres");
  });
});

describe("textos da receita externa e configurações", () => {
  it("profissional, data, notificação, motivo e faixas", () => {
    const p = { name: "Dra. Helena Prado", council: "CRM", council_state: "PR", council_number: "12345" };
    expect(prescriberProblem(p)).toBeNull();
    expect(prescriberProblem({ ...p, council: "CRF" })).toBe("escolha o conselho (CRM, CRO ou COREN)");
    expect(prescriberProblem({ ...p, council_state: "XX" })).toBe("escolha a UF do conselho");
    expect(prescriberProblem({ ...p, council_number: "12a" })).toBe("o número no conselho tem só dígitos (até 10)");
    expect(prescriberProblem({ ...p, name: "Dr" })).toBe("o nome do profissional tem de 3 a 120 caracteres");
    expect(issuedOnProblem("2026-10-30", TODAY27)).toBe("a data da receita não pode ser depois de hoje");
    expect(issuedOnProblem("", TODAY27)).toBe("informe a data da receita");
    expect(notificationNumberProblem("")).toBeNull();
    expect(notificationNumberProblem("B1 0042")).toBe("número da notificação: até 30 letras, números, ponto, barra ou hífen");
    expect(reasonProblem("a".repeat(501))).toBe("o motivo tem de 10 a 500 caracteres");
    const s = { common_validity_days: "30", continuous_validity_days: "180", continuous_interval_days: "25", expiry_alert_days: "90", adjustment_approval_percent: "10" };
    expect(settingsProblem(s)).toBeNull();
    expect(settingsProblem({ ...s, continuous_interval_days: "91" })).toBe("intervalo do uso contínuo de 1 a 90 dias");
    expect(settingsProblem({ ...s, adjustment_approval_percent: "10,5" })).toBe("limite do ajuste: número inteiro de 0 a 100");
  });
});

describe("recusas do api", () => {
  const e = (status: number, body: unknown) => new ApiError(status, body, String(status));
  it("frases; código novo cru; problems de uma vez", () => {
    expect(pharmacyError(e(403, { error: "feature_disabled", feature: "pharmacy" }))).toBe(PHARMACY_DISABLED);
    expect(pharmacyError(e(422, { error: "too_early", next_allowed_on: "2026-11-03", path: "items[0]" }))).toBe("uso contínuo: próxima retirada deste item a partir de 03/11/2026");
    expect(pharmacyError(e(422, { error: "insufficient_balance", path: "items[0].quantity" }))).toBe("o saldo do lote mudou — os lotes foram relidos; confira a quantidade");
    expect(pharmacyError(e(403, { error: "controlled_requires_pharmacist" }))).toBe(CONTROLLED_PHARMACIST_ONLY);
    expect(pharmacyError(e(422, { error: "xyz_new" }))).toBe("não foi possível concluir (xyz_new) — tente de novo");
    expect(pharmacyError(new Error("offline"))).toBe("sem conexão com o servidor — tente de novo");
    expect(pharmacyProblems(e(422, { error: "insufficient_balance", problems: [ { path: "items[0].quantity", error: "insufficient_balance" },
      { path: "items[1]", error: "lot_expired" } ] }))).toEqual([
      { path: "items[0].quantity", text: "o saldo do lote mudou — os lotes foram relidos; confira a quantidade" },
      { path: "items[1]", text: "lote vencido — escolha outro lote" } ]);
    expect(pharmacyProblems(e(422, { error: "lot_expired" }))).toEqual([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/lib/pharmacy.test.ts`
Expected: FAIL — o módulo não existe.

- [ ] **Step 3: Implemente `src/lib/pharmacy.ts`**

Funções puras com os nomes e frases da lista de Interfaces e do teste. Escolhas que o teste não fixa sozinho:
- quantidade: `^\d{1,5}(,\d{1,2})?$` depois de `trim` (o `QUANTITY` do api; ponto → `DOT_IN_NUMBER` antes de tudo; sinal ou letra → "<rótulo>: use só dígitos e vírgula"; mais de 2 casas e mais de 5 dígitos inteiros com as frases do teste, nessa ordem); `cents` por `Math.round(Number(api) * 100)`; `fmtQuantity` tira zeros à direita e troca o ponto pela vírgula.
- `normalizeShortCode`: maiúsculas, sem espaço e hífen, exatamente 10 de `23456789ABCDEFGHJKMNPQRSTUVWXYZ` (o `ClinicalDocuments::Codes` do api).
- `itemBlock` na ordem do teste: completo → item vencido (`expired` do item, P4) → próxima retirada futura (só quando `continuous` e `next_allowed_on > today`) → controlado sem farmacêutico → texto livre sem farmacêutico.
- `dispenseProblems`: uma entrada por linha preenchida, na ordem dos itens da receita; para cada uma, a primeira recusa entre `itemBlock`, texto livre sem `catalogItemId` ("escolha o item do catálogo"), lote vazio ("escolha o lote"), `quantityProblem` e saldo (`cents(toApiQuantity(q)) > cents(balances[lotId])` → "saída maior que o saldo do lote (<fmtQuantity>)"), prefixada por `printed_description`; depois `retention` quando `needsRetention` e não `retained`; nenhuma linha preenchida → só `NOTHING_TO_DISPENSE`. `needsRetention` = alguma linha preenchida de item `controlled`.
- `dispenseBody`: só linhas com quantidade; `catalog_item_id` só para item sem `catalog_item_id` na receita; `complete: true` só quando marcado; `retention: {}` só quando `needsRetention` (o número retido é o do documento — `sncr.number` da receita digital ou a notificação da externa —, nunca digitado aqui; D14).
- `receiptProblems`: documento só fora da abertura (`document.number`: vazio → "informe o número do documento", acima de 60 → "o número do documento tem até 60 caracteres"; `document.supplier`: vazio → "informe o fornecedor ou doador", acima de 120 → "o fornecedor ou doador tem até 120 caracteres"); por linha, a **primeira** recusa da linha, prefixada pelo nome do item: lote (1–40), validade (vazia → "informe a validade"; vencida → "lote vencido — não entra no estoque"), fabricante até 120, quantidade (`quantityProblem`) ou, por embalagem, unidades por embalagem 1–10000 e embalagens 1–100000, inteiros; nenhuma linha → "acrescente ao menos um item". As conferências com várias linhas aparecem numa lista `<ul role="alert">`, um `<li>` por problema, em todas as telas do módulo.
- `pharmacyError`: `featureDisabledKey === "pharmacy"` → `PHARMACY_DISABLED`; tabela de códigos com, além dos do teste: `prescription_cancelled`/`prescription_expired` → `PRESCRIPTION_CHANGED`; `free_text_requires_pharmacist` → `FREE_TEXT_PHARMACIST_ONLY`; `retention_required` → `RETENTION_REQUIRED`; `lot_expired`/`expired_lot` → "lote vencido — escolha outro lote"; `invalid_item` → "item do catálogo inválido ou inativo"; `invalid_quantity` → "quantidade inválida — use até 2 casas depois da vírgula"; `opening_exists` → "este item já teve inventário de abertura neste local — corrija por ajuste"; `already_received` → "esta transferência já foi recebida — a lista foi relida"; `same_user` → "outra pessoa do local precisa aprovar este ajuste"; `not_pending` → "este ajuste já foi decidido — a lista foi relida"; `cbo_not_allowed` → "farmacêutico precisa de vínculo profissional com CBO 2234"; `role_not_granted` → "a pessoa precisa ter este papel na Equipe antes"; `unit_taken` → "esta unidade já tem um local de estoque"; `central_exists` → "a cidade já tem um almoxarifado central"; `location_has_stock` → "o local ainda tem saldo — transfira ou ajuste antes de desativar"; `already_linked` → "esta pessoa já está no local com este papel"; `location_inactive` → "este local está inativo"; `no_difference` → "a contagem é igual ao saldo — não há o que ajustar"; `lot_not_expired` → "vencimento só de lote já vencido — use perda"; `lot_mismatch` → "este lote já existe com outra validade ou fabricante — confira"; `item_completed` → `ITEM_COMPLETED`; `notification_required` → "receita de lista A ou B exige o número da notificação"; `prescription_not_found` → "receita não encontrada nesta cidade — confira o código"; `person_not_found` → "ninguém com este CPF no Rota Saúde"; `reason_required`/`invalid_reason` → "o motivo tem de 10 a 500 caracteres"; `invalid_items` → "informe todos os lotes da transferência"; `too_many_requests` → "muitas buscas em pouco tempo — espere um minuto"; `catalog_unavailable` → "o catálogo de medicamentos está indisponível — tente de novo em instantes"; `not_found` (404) → "não encontrado — a tela foi atualizada"; `duplicate_suspected` → a frase do 24a (`DUPLICATE_SUSPECTED_TEXT`); `mfa_required` → "o código do autenticador não valeu — digite o código atual"; `missing_role` → `PHARMACY_NO_ROLE`. Corpo com `next_allowed_on` completa a frase do `too_early`. Outro `ApiError` → "não foi possível concluir (<código ou status>) — tente de novo"; outro erro → "sem conexão com o servidor — tente de novo". Nenhuma frase repete valor enviado.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/lib/pharmacy.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/pharmacy.ts src/lib/pharmacy.test.ts
$G commit -m "feat: add pharmacy screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Farmácia", portão, escolha do local e papéis na Equipe

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Create: `src/modules/pharmacy/PharmacyGate.tsx`, `src/modules/pharmacy/LocationPick.tsx`, `src/modules/PharmacyDispensing.tsx`, `src/modules/PharmacyExternal.tsx`, `src/modules/PharmacyTransfers.tsx`, `src/modules/PharmacyEntries.tsx`, `src/modules/PharmacyStock.tsx`, `src/modules/PharmacyAdjustments.tsx`, `src/modules/PharmacyAlerts.tsx`, `src/modules/PharmacyAdmin.tsx` (portão + `EmptyState` provisório; as Tasks 4–13 trocam o corpo)
- Test: `src/shell/modules.pharmacy.test.ts`

**Interfaces:**
- Consumes: `canUsePharmacy`, `pharmacyRoles`, `locationRoles`, `PH_LOCATIONS_KEY`, `STATIC_QUERY`, `NO_LOCATION`, `pharmacyError` (Task 2); `listMyPharmacyLocations` (Task 1).
- Produces: `ModuleId` com `"pharmacy-dispensing" | "pharmacy-external" | "pharmacy-transfers" | "pharmacy-entries" | "pharmacy-stock" | "pharmacy-adjustments" | "pharmacy-alerts" | "pharmacy-admin"`; o grupo `{ label: "Farmácia" }` logo depois de "Regulação"; `PharmacyGate({ title, sub, allow(roles), children(roles) })`; `LocationPick({ allow(r: { pharmacist; assistant; keeper }): boolean; children(loc: PharmacyLocation): ReactNode })` (lê os locais com `STATIC_QUERY`, mostra só os ativos que o `allow` aceita, seletor "Local" quando há mais de um, `NO_LOCATION` quando nenhum, remonta o conteúdo ao trocar); papéis na Equipe `pharmacist` "Farmacêutico", `pharmacy_assistant` "Auxiliar de farmácia", `stock_keeper` "Almoxarife"; privilegiados só os dois primeiros (D19, P17 do plano do api).

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.pharmacy.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor } from "./modules";
import { CATALOG } from "./catalog";

const user = (roles: string[], features: string[] | undefined = [ "pharmacy" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });
const itemsOf = (groups: ReturnType<typeof navGroupsFor>) => groups.find((g) => g.label === "Farmácia")?.items.map((i) => i.id) ?? [];

describe("grupo Farmácia (módulo 27a)", () => {
  it("cada papel vê o seu", () => {
    expect(itemsOf(navGroupsFor(user([ "pharmacist" ])))).toEqual([
      "pharmacy-dispensing", "pharmacy-external", "pharmacy-transfers", "pharmacy-entries", "pharmacy-stock", "pharmacy-adjustments", "pharmacy-alerts" ]);
    expect(itemsOf(navGroupsFor(user([ "pharmacy_assistant" ])))).toEqual([
      "pharmacy-dispensing", "pharmacy-external", "pharmacy-transfers", "pharmacy-stock", "pharmacy-alerts" ]);
    expect(itemsOf(navGroupsFor(user([ "stock_keeper" ])))).toEqual([
      "pharmacy-transfers", "pharmacy-entries", "pharmacy-stock", "pharmacy-adjustments", "pharmacy-alerts" ]);
    expect(itemsOf(navGroupsFor(user([ "municipal_admin" ])))).toEqual([ "pharmacy-admin" ]);
  });
  it("some sem o interruptor, para outros papéis e para o operador", () => {
    expect(itemsOf(navGroupsFor(user([ "pharmacist", "stock_keeper", "municipal_admin" ], [ "clinical_record" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "health_professional", "citizen_verifier", "care_regulator" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([], undefined, true)))).toEqual([]);
  });
  it("vem depois da Regulação, com ícone e descrição", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Farmácia")).toBe(labels.indexOf("Regulação") + 1);
    expect(CATALOG.groups["Farmácia"].description).toBe("Almoxarifado, farmácia da unidade, dispensação e alertas");
    expect(CATALOG.screens["pharmacy-dispensing"].description).toBe("Receita pelo código ou pelo paciente, lote e retirada");
    expect(CATALOG.screens["pharmacy-external"].description).toBe("Receita de fora da rede, com profissional e conselho");
    expect(CATALOG.screens["pharmacy-transfers"].description).toBe("Envio do almoxarifado e recebimento conferido na unidade");
    expect(CATALOG.screens["pharmacy-entries"].description).toBe("Nota fiscal, doação e inventário de abertura, com lote");
    expect(CATALOG.screens["pharmacy-stock"].description).toBe("Saldo por local e lote, extrato e estoque mínimo");
    expect(CATALOG.screens["pharmacy-adjustments"].description).toBe("Ajustes, perdas e vencimentos, com aprovação");
    expect(CATALOG.screens["pharmacy-alerts"].description).toBe("Vencendo, abaixo do mínimo, em trânsito e pendências");
    expect(CATALOG.screens["pharmacy-admin"].description).toBe("Locais de estoque, equipe por local e prazos");
  });
});
```

Em `src/shell/modules.test.ts`, nos dois casos que conferem grupos escondidos, acrescente `expect(groups.some((g) => g.label === "Farmácia")).toBe(false);` e troque `NAV_GROUPS.length - 14` por `- 15` e `NAV_GROUPS.length - 13` por `- 14`. Em `src/lib/team.test.ts`, o convite passa a ter **18** papéis (renomeie o caso para "oferece os 18 papéis…"; lista ordenada com `"pharmacist", "pharmacy_assistant"` entre `"municipal_admin"` e `"protocol_author"` e `"stock_keeper"` entre `"protocol_reviewer"` e `"surveillance"`) e:

```ts
    expect(INVITE_ROLES.find((r) => r.role === "pharmacist")?.label).toBe("Farmacêutico");
    expect(INVITE_ROLES.find((r) => r.role === "pharmacy_assistant")?.label).toBe("Auxiliar de farmácia");
    expect(INVITE_ROLES.find((r) => r.role === "stock_keeper")?.label).toBe("Almoxarife");
    expect(isPrivilegedRole("pharmacist")).toBe(true);
    expect(isPrivilegedRole("pharmacy_assistant")).toBe(true);
    expect(isPrivilegedRole("stock_keeper")).toBe(false);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/shell src/lib/team.test.ts`
Expected: FAIL — grupo, ícones e papéis não existem.

- [ ] **Step 3: Implemente**

- `modules.ts`: o grupo depois de "Regulação" com `{ id: "pharmacy-dispensing", label: "Dispensação", icon: "℞" }`, `{ id: "pharmacy-external", label: "Receita externa", icon: "✎" }`, `{ id: "pharmacy-transfers", label: "Transferências", icon: "⇄" }`, `{ id: "pharmacy-entries", label: "Entradas e abertura", icon: "⇲" }`, `{ id: "pharmacy-stock", label: "Posição e extrato", icon: "▦" }`, `{ id: "pharmacy-adjustments", label: "Inventário e ajustes", icon: "±" }`, `{ id: "pharmacy-alerts", label: "Alertas da farmácia", icon: "⚠" }`, `{ id: "pharmacy-admin", label: "Locais e configurações", icon: "⚙" }`; no filtro, com o comentário do módulo, `phOn = canUsePharmacy(user)` e `isPharmacist`, `isPharmacyAssistant`, `isStockKeeper`: dispensação e receita externa `phOn && (isPharmacist || isPharmacyAssistant)`; transferências, posição e alertas `phOn && (isPharmacist || isPharmacyAssistant || isStockKeeper)`; entradas `phOn && (isStockKeeper || isPharmacist)` (o farmacêutico faz a abertura — P6 do plano do api); ajustes `phOn && (isPharmacist || isStockKeeper)`; administração `phOn && isAdmin`.
- `catalog.ts`: grupo `"Farmácia": { icon: IconBuildingWarehouse, description: … }`; telas com `IconPrescription`, `IconFilePlus`, `IconTransfer`, `IconTruckLoading`, `IconListSearch`, `IconScale`, `IconAlertTriangle`, `IconBuildingCog` e as descrições do teste.
- `App.tsx`: os oito `case` (`pharmacy-admin` e `pharmacy-adjustments` com `onGoToSecurity={() => setActive("security")}`, como as telas com step-up).
- `PharmacyGate.tsx`: cópia do `RegulationGate` com `PHARMACY_DISABLED`, `PHARMACY_NO_ROLE`, `canUsePharmacy` e `pharmacyRoles`. As oito telas nascem com o portão (`allow` pelo papel do filtro de `modules.ts`) e um `EmptyState` provisório.
- `LocationPick.tsx`: no molde do `LabPick`; nada em storage.
- `team.ts`: os três papéis antes de `municipal_admin` (comentário: "Módulo 27a (ADR 0045): vinculados a locais de estoque; farmacêutico e auxiliar leem e gravam dado nominal — privilegiados (step-up para conceder)") e `pharmacist`, `pharmacy_assistant` no `PRIVILEGED_ROLES`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/shell src/lib/team.test.ts src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos de `modules.test.ts` só mudaram as contagens).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.pharmacy.test.ts src/shell/catalog.ts src/App.tsx src/lib/team.ts src/lib/team.test.ts \
  src/modules/pharmacy/PharmacyGate.tsx src/modules/pharmacy/LocationPick.tsx src/modules/PharmacyDispensing.tsx src/modules/PharmacyExternal.tsx \
  src/modules/PharmacyTransfers.tsx src/modules/PharmacyEntries.tsx src/modules/PharmacyStock.tsx src/modules/PharmacyAdjustments.tsx \
  src/modules/PharmacyAlerts.tsx src/modules/PharmacyAdmin.tsx
$G commit -m "feat: add the pharmacy menu group and roles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Locais, equipe por local e configurações (F-27.1)

**Files:**
- Create: `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/pharmacy/StaffPanel.tsx`, `src/modules/pharmacy/SettingsPanel.tsx`
- Modify: `src/modules/PharmacyAdmin.tsx`
- Test: `src/modules/PharmacyAdmin.test.tsx`

**Interfaces:**
- Consumes: `listAdminLocations`, `createAdminLocation`, `updateAdminLocation`, `addLocationStaff`, `removeLocationStaff`, `getPharmacySettings`, `updatePharmacySettings`, `listAllUnits`, `listMemberships` (Task 1 e existentes); `locationKindLabel`, `pharmacyRoleLabel`, `settingsProblem`, `pharmacyError`, `PH_ADMIN_KEY`, `PH_SETTINGS_KEY`, `STATIC_QUERY` (Task 2); `PharmacyGate` (Task 3); `SensitiveAction`.
- Produces: `PharmacyAdmin({ onGoToSecurity? })` com três painéis: **Locais de estoque** (tabela nome, tipo, unidade, situação, equipe; "Novo local" e "Editar" → formulário → "Revisar" → `SensitiveAction` com step-up; tipo "Almoxarifado central" sem unidade, "Farmácia da unidade" com a unidade — `listAllUnits` ativas —; o tipo não muda depois de criado); **Equipe do local** (o local escolhido na tabela; cada vínculo com o nome — ou o e-mail de `listMemberships` quando sem nome — e o papel; papel → pessoa: só quem tem esse papel em `listMemberships`; "Vincular" e "Desvincular" com step-up); **Configurações** (os cinco números; "Revisar" → step-up; o corpo vai com números).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyAdmin.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listAdminLocations: vi.fn(), createAdminLocation: vi.fn(), updateAdminLocation: vi.fn(), addLocationStaff: vi.fn(), removeLocationStaff: vi.fn(),
  getPharmacySettings: vi.fn(), updatePharmacySettings: vi.fn(), listAllUnits: vi.fn(), listMemberships: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { PharmacyAdmin } from "./PharmacyAdmin";
import { renderWithProviders } from "../test/campaignFixtures";
import { adminLocation, phSettings, phUser } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("PharmacyAdmin", () => {
  beforeEach(() => {
    for (const fn of [ api.createAdminLocation, api.updateAdminLocation, api.addLocationStaff, api.removeLocationStaff, api.updatePharmacySettings ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "municipal_admin" ]));
    mocked(api.listAdminLocations).mockResolvedValue([ adminLocation(), adminLocation({ id: "l1", name: "Almoxarifado Central", kind: "central", health_unit: null, staff: [] }) ]);
    mocked(api.listAllUnits).mockResolvedValue([ { id: "u1", name: "UBS Centro", kind: "ubs", active: true }, { id: "u3", name: "UBS Norte", kind: "ubs", active: true } ]);
    mocked(api.listMemberships).mockResolvedValue([
      { id: "m1", user: { id: "u-9", email_address: "marta@curitiba.demo" }, role: "pharmacist", granted_at: "2026-10-01T00:00:00Z" },
      { id: "m2", user: { id: "u-7", email_address: "paulo@curitiba.demo" }, role: "stock_keeper", granted_at: "2026-10-01T00:00:00Z" } ]);
    mocked(api.getPharmacySettings).mockResolvedValue(phSettings());
    mocked(api.createAdminLocation).mockResolvedValue(adminLocation({ id: "l3", name: "Farmácia UBS Norte", health_unit: { id: "u3", name: "UBS Norte" }, staff: [] }));
    mocked(api.updatePharmacySettings).mockResolvedValue(phSettings({ continuous_interval_days: 28 }));
  });

  it("locais: tipo, unidade, situação e equipe", async () => {
    renderWithProviders(<PharmacyAdmin />);
    const row = await screen.findByRole("row", { name: /Farmácia UBS Centro/ });
    expect(row.textContent).toContain("Farmácia da unidade");
    expect(row.textContent).toContain("UBS Centro");
    expect(row.textContent).toContain("1 pessoa");
    expect(screen.getByRole("row", { name: /Almoxarifado Central/ }).textContent).toContain("Almoxarifado central");
  });

  it("novo local: unidade exigida na farmácia da unidade; step-up e o corpo", async () => {
    renderWithProviders(<PharmacyAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Novo local" }));
    text("Tipo", "unit"); text("Nome", "Farmácia UBS Norte");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    expect((await screen.findByRole("alert")).textContent).toBe("escolha a unidade de saúde da farmácia");
    text("Unidade de saúde", "u3");
    fireEvent.click(screen.getByRole("button", { name: "Revisar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar local" }));
    await waitFor(() => expect(mocked(api.createAdminLocation)).toHaveBeenCalledWith({ kind: "unit", health_unit_id: "u3", name: "Farmácia UBS Norte", active: true }));
  });

  it("equipe: só quem tem o papel; farmacêutico sem CBO 2234 recusado com a frase", async () => {
    mocked(api.addLocationStaff).mockRejectedValue(new ApiError(422, { error: "cbo_not_allowed" }, "422"));
    renderWithProviders(<PharmacyAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Equipe de Farmácia UBS Centro" }));
    const panel = within(await screen.findByRole("region", { name: "Equipe de Farmácia UBS Centro" }));
    expect(panel.getByText(/Marta Souza/)).not.toBeNull();
    text("Papel no local", "stock_keeper");
    expect(screen.getAllByRole("option").map((o) => o.textContent)).toContain("paulo@curitiba.demo");
    expect(screen.getAllByRole("option").map((o) => o.textContent)).not.toContain("marta@curitiba.demo");
    text("Papel no local", "pharmacist"); text("Pessoa", "u-9");
    fireEvent.click(panel.getByRole("button", { name: "Vincular" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar vínculo" }));
    expect((await screen.findByText("farmacêutico precisa de vínculo profissional com CBO 2234"))).not.toBeNull();
    expect(mocked(api.addLocationStaff)).toHaveBeenCalledWith("l2", "u-9", "pharmacist");
  });

  it("configurações: faixas conferidas antes; corpo com números depois do step-up", async () => {
    renderWithProviders(<PharmacyAdmin />);
    await screen.findByDisplayValue("25");
    text("Intervalo do uso contínuo (dias)", "91");
    fireEvent.click(screen.getByRole("button", { name: "Revisar configurações" }));
    expect((await screen.findByRole("alert")).textContent).toBe("intervalo do uso contínuo de 1 a 90 dias");
    text("Intervalo do uso contínuo (dias)", "28");
    fireEvent.click(screen.getByRole("button", { name: "Revisar configurações" }));
    fireEvent.click(await screen.findByRole("button", { name: "Salvar configurações" }));
    await waitFor(() => expect(mocked(api.updatePharmacySettings)).toHaveBeenCalledWith(phSettings({ continuous_interval_days: 28 })));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyAdmin.test.tsx`
Expected: FAIL — o corpo da tela ainda é o provisório.

- [ ] **Step 3: Implemente** no molde de `RegulationOffers` (conferência antes, escrita no `SensitiveAction` com `requiresStepUp`, recusa traduzida por `translateError={pharmacyError}`, releitura de `PH_ADMIN_KEY` depois de cada escrita). Rótulos dos campos: "Tipo" (`central` → "Almoxarifado central", `unit` → "Farmácia da unidade"; desabilitado na edição), "Nome" (3–120; "o nome do local tem de 3 a 120 caracteres"), "Unidade de saúde" (só no `unit`; vazia → "escolha a unidade de saúde da farmácia"), "Ativo"; botão do `SensitiveAction`: "Criar local" / "Salvar local"; a coluna da equipe diz "1 pessoa" / "N pessoas" / "ninguém". Na equipe: "Papel no local", "Pessoa" (só e-mails com o papel escolhido em `listMemberships`, sem quem já está no local com ele), "Vincular" → `SensitiveAction` "Confirmar vínculo"; "Desvincular <e-mail>" → "Confirmar desvínculo". Configurações: "Revisar configurações" → "Salvar configurações". O painel da equipe é uma `region` com `aria-label` "Equipe de <nome>", aberta pelo botão de mesmo nome na linha. Configurações: "Validade da receita comum (dias)", "Validade da receita de uso contínuo (dias)", "Intervalo do uso contínuo (dias)", "Alerta de validade (dias)", "Limite do ajuste sem aprovação (%)"; o texto do `SensitiveAction` diz que receitas e ajustes já registrados não mudam.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyAdmin.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyAdmin.tsx src/modules/PharmacyAdmin.test.tsx src/modules/pharmacy/LocationsPanel.tsx src/modules/pharmacy/StaffPanel.tsx src/modules/pharmacy/SettingsPanel.tsx
$G commit -m "feat: manage pharmacy locations, staff and settings

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Busca no catálogo, entradas com lote e embalagem, inventário de abertura (F-27.2)

**Files:**
- Create: `src/modules/pharmacy/ItemSearch.tsx`, `src/modules/pharmacy/ReceiptForm.tsx`
- Modify: `src/modules/PharmacyEntries.tsx`
- Test: `src/modules/PharmacyEntries.test.tsx`

**Interfaces:**
- Consumes: `searchPharmacyItems`, `recordReceipt`, `recordOpening`, `listMyPharmacyLocations` (Task 1); `newReceiptLine`, `lineQuantity`, `receiptProblems`, `receiptItems`, `pharmacyError`, `pharmacyProblems`, `errorPath`, `fmtQuantity`, `PH_ITEMS_KEY`, `PH_POSITIONS_KEY` (Task 2); `PharmacyGate`, `LocationPick` (Task 3); `useDebouncedValue`.
- Produces:
  - `ItemSearch({ label, onPick(item: StockItem), delayMs?, disabled? })` — termo de 3 a 60 no corpo, `gcTime: 0`, lista de botões `"<nome> · <unidade>"` com `Tag` "controlado · lista <X>" quando `controlled`; limpa ao escolher;
  - `ReceiptForm({ location: PharmacyLocation, mode: "receipt" | "opening", today, delayMs?, onDone(r: StockReceipt) })` — em `receipt`: "Tipo de entrada" (`invoice` "Nota fiscal" | `donation` "Doação"), "Número do documento", "Fornecedor ou doador"; linhas por item com "Lote", "Validade", "Fabricante (opcional)", "Por embalagem" (marca: "Unidades por embalagem" e "Embalagens", e a conta "= 300 cápsula"), ou "Quantidade (<unidade>)"; "Remover"; envio único; todas as recusas de uma vez, cada uma com o item;
  - `PharmacyEntries({ today? })` — `LocationPick` com `allow: keeper || pharmacist`; a escolha "Entrada" | "Inventário de abertura" só para o `stock_keeper` do local; o farmacêutico sem o papel de almoxarife vê só a abertura (P6 do plano do api).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyEntries.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), searchPharmacyItems: vi.fn(), recordReceipt: vi.fn(), recordOpening: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { PharmacyEntries } from "./PharmacyEntries";
import { renderWithProviders } from "../test/campaignFixtures";
import { AMOXI, CENTRAL, TODAY27, UBS_FARM, phUser, stockLot } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

async function addAmoxi() {
  text("Buscar medicamento", "amox");
  fireEvent.click(await screen.findByRole("button", { name: "Amoxicilina 500 mg cápsula · cápsula" }));
}

describe("PharmacyEntries", () => {
  beforeEach(() => {
    for (const fn of [ api.recordReceipt, api.recordOpening, api.searchPharmacyItems ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "stock_keeper" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ CENTRAL ]);
    mocked(api.searchPharmacyItems).mockResolvedValue([ AMOXI ]);
    mocked(api.recordReceipt).mockResolvedValue({ id: "r1", location: CENTRAL, kind: "receipt", items: [ { lot: stockLot(), quantity: "300" } ], at: "2026-10-29T09:00:00-03:00" });
    mocked(api.recordOpening).mockResolvedValue({ id: "r2", location: CENTRAL, kind: "opening", items: [ { lot: stockLot(), quantity: "40" } ], at: "2026-10-29T09:00:00-03:00" });
  });

  it("nota fiscal: item, lote, validade e embalagem; a conta antes; corpo com quantidade e embalagem", async () => {
    renderWithProviders(<PharmacyEntries today={TODAY27} />);
    await screen.findByText("Almoxarifado Central");
    text("Tipo de entrada", "invoice"); text("Número do documento", "NF 1234"); text("Fornecedor ou doador", "Distribuidora Sul");
    await addAmoxi();
    text("Lote", "AMX2611"); text("Validade", "2026-11-20");
    fireEvent.click(screen.getByLabelText("Por embalagem"));
    text("Unidades por embalagem", "30"); text("Embalagens", "10");
    expect(screen.getByText("= 300 cápsula")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Registrar entrada" }));
    await waitFor(() => expect(mocked(api.recordReceipt)).toHaveBeenCalledWith({
      location_id: "l1", document: { kind: "invoice", number: "NF 1234", supplier: "Distribuidora Sul" },
      items: [ { catalog_item_id: "ci1", lot_number: "AMX2611", expires_on: "2026-11-20", quantity: "300", package: { units_per_package: 30, packages: 10 } } ] }));
    expect((await screen.findByRole("status")).textContent).toBe("Entrada registrada: 1 lote.");
  });

  it("todas as recusas de uma vez; a recusa do api com path marca o item e o formulário fica", async () => {
    mocked(api.recordReceipt).mockRejectedValue(new ApiError(422, { error: "invalid_item", path: "items[0].catalog_item_id" }, "422"));
    renderWithProviders(<PharmacyEntries today={TODAY27} />);
    await screen.findByText("Almoxarifado Central");
    await addAmoxi();
    text("Lote", "AMX2611"); text("Validade", "2026-10-28"); text("Quantidade (cápsula)", "1.000");
    fireEvent.click(screen.getByRole("button", { name: "Registrar entrada" }));
    const alerts = (await screen.findAllByRole("listitem")).map((li) => li.textContent);
    expect(alerts).toEqual(expect.arrayContaining([ "informe o número do documento", "informe o fornecedor ou doador",
      "Amoxicilina 500 mg cápsula: lote vencido — não entra no estoque" ]));
    expect(mocked(api.recordReceipt)).not.toHaveBeenCalled();
    text("Número do documento", "NF 1234"); text("Fornecedor ou doador", "Distribuidora Sul"); text("Validade", "2026-11-20"); text("Quantidade (cápsula)", "300");
    fireEvent.click(screen.getByRole("button", { name: "Registrar entrada" }));
    expect(await screen.findByText("Amoxicilina 500 mg cápsula: item do catálogo inválido ou inativo")).not.toBeNull();
    expect((screen.getByLabelText("Lote") as HTMLInputElement).value).toBe("AMX2611");
  });

  it("farmacêutico: só a abertura, sem documento; opening_exists com a frase", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ UBS_FARM ]);
    mocked(api.recordOpening).mockRejectedValueOnce(new ApiError(409, { error: "opening_exists", path: "items[0].catalog_item_id" }, "409"));
    renderWithProviders(<PharmacyEntries today={TODAY27} />);
    await screen.findByText("Farmácia UBS Centro");
    expect(screen.queryByLabelText("Entrada")).toBeNull();
    expect(screen.queryByLabelText("Número do documento")).toBeNull();
    await addAmoxi();
    text("Lote", "AMX2611"); text("Validade", "2026-11-20"); text("Quantidade (cápsula)", "40");
    fireEvent.click(screen.getByRole("button", { name: "Registrar abertura" }));
    expect(await screen.findByText("Amoxicilina 500 mg cápsula: este item já teve inventário de abertura neste local — corrija por ajuste")).not.toBeNull();
    expect(mocked(api.recordOpening)).toHaveBeenCalledWith({ location_id: "l2",
      items: [ { catalog_item_id: "ci1", lot_number: "AMX2611", expires_on: "2026-11-20", quantity: "40" } ] });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyEntries.test.tsx`
Expected: FAIL — `ItemSearch` e `ReceiptForm` não existem.

- [ ] **Step 3: Implemente** `ItemSearch` no molde do `MedicationSearch` do 19c (busca pelo api da farmácia, D3) e `ReceiptForm` com `receiptProblems`/`receiptItems` (a recusa do api com `path` `items[i]…` vira a frase prefixada pelo nome do item i; com `problems`, todas). Depois do sucesso: limpa as linhas, mantém o tipo de entrada, invalida `PH_POSITIONS_KEY` e diz "Entrada registrada: N lote(s)." ("Abertura registrada: N lote(s)." no modo abertura). Os rótulos de campo das linhas levam o item no `aria-label` só quando há mais de uma linha (`Lote — <nome>`), para os testes de uma linha usarem o rótulo curto.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyEntries.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyEntries.tsx src/modules/PharmacyEntries.test.tsx src/modules/pharmacy/ItemSearch.tsx src/modules/pharmacy/ReceiptForm.tsx
$G commit -m "feat: record pharmacy receipts and opening inventory

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Posição por local e lote, extrato e estoque mínimo (F-27.7)

**Files:**
- Create: `src/modules/pharmacy/LotStatement.tsx`, `src/modules/pharmacy/MinimumsPanel.tsx`
- Modify: `src/modules/PharmacyStock.tsx`
- Test: `src/modules/PharmacyStock.test.tsx`

**Interfaces:**
- Consumes: `listPositions`, `listLotMovements`, `getMinimums`, `putMinimums` (Task 1); `fmtQuantity`, `fmtSigned`, `movementLabel`, `quantityProblem`, `toApiQuantity`, `pharmacyError`, `PH_POSITIONS_KEY`, `PH_MOVEMENTS_KEY`, `PH_MINIMUMS_KEY`, `STATIC_QUERY` (Task 2); `LocationPick`, `ItemSearch` (Tasks 3 e 5).
- Produces: `PharmacyStock({ today? })` — filtros "Medicamento" (nome, vai como `q`), "Vencendo" e "Abaixo do mínimo" (marcas), e, para o `stock_keeper` do central, "Todos os locais" (sem `location_id`); tabela por posição (local, medicamento, saldo com a unidade, mínimo, `Tag` "abaixo do mínimo"/"vencendo"); "Lotes" abre os lotes do item (lote, validade, saldo, em trânsito, "Extrato"); "Carregar mais" pelo cursor; `LotStatement({ lotId, locationId, lotNumber })` com movimentos (quando, movimento, quantidade com sinal, documento, motivo quando houver, por) e "Carregar mais"; `MinimumsPanel({ location })` só para o `stock_keeper` do local: a lista inteira com "Mínimo (<unidade>)" por item, acrescentar pelo `ItemSearch`, "Salvar mínimos" (`PUT` da lista inteira; zero aceito).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyStock.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), listPositions: vi.fn(), listLotMovements: vi.fn(), getMinimums: vi.fn(), putMinimums: vi.fn(), searchPharmacyItems: vi.fn() }));
import * as api from "../lib/api";
import { PharmacyStock } from "./PharmacyStock";
import { renderWithProviders } from "../test/campaignFixtures";
import { AMOXI, CENTRAL, TODAY27, UBS_FARM, UBS_REF, phUser, position } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("PharmacyStock", () => {
  beforeEach(() => {
    for (const fn of [ api.listPositions, api.listLotMovements, api.putMinimums ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist", "stock_keeper" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ { ...UBS_FARM, roles: [ "pharmacist", "stock_keeper" ] } ]);
    mocked(api.listPositions).mockResolvedValue({ positions: [ position() ], next_before: null });
    mocked(api.listLotMovements).mockResolvedValue({ movements: [
      { id: "m2", kind: "dispensing", quantity: "-21.00", document: { type: "dispensation", id: "dp1", number: null },
        by: { name: "Marta Souza" }, at: "2026-10-29T10:15:00-03:00" },
      { id: "m1", kind: "transfer_in", quantity: "51.00", document: { type: "transfer", id: "t1", number: null },
        by: { name: "Marta Souza" }, at: "2026-10-28T15:00:00-03:00" } ], next_before: null });
    mocked(api.getMinimums).mockResolvedValue([ { catalog_item_id: "ci1", minimum: "50", item: AMOXI } ]);
    mocked(api.putMinimums).mockResolvedValue([ { catalog_item_id: "ci1", minimum: "60.5", item: AMOXI } ]);
  });

  it("posição do local: saldo, mínimo e marcas; filtros na URL sem dado pessoal", async () => {
    renderWithProviders(<PharmacyStock today={TODAY27} />);
    const row = await screen.findByRole("row", { name: /Amoxicilina 500 mg cápsula/ });
    expect(row.textContent).toContain("30 cápsula");
    expect(within(row).getByText("abaixo do mínimo")).not.toBeNull();
    expect(within(row).getByText("vencendo")).not.toBeNull();
    expect(mocked(api.listPositions)).toHaveBeenLastCalledWith({ locationId: "l2", q: null, expiring: undefined, belowMinimum: undefined, before: null });
    fireEvent.change(screen.getByLabelText("Medicamento"), { target: { value: "amox" } });
    fireEvent.click(screen.getByLabelText("Abaixo do mínimo"));
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    await waitFor(() => expect(mocked(api.listPositions)).toHaveBeenLastCalledWith({ locationId: "l2", q: "amox", expiring: undefined, belowMinimum: true, before: null }));
  });

  it("central: todos os locais sem location_id", async () => {
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ CENTRAL ]);
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "stock_keeper" ]));
    renderWithProviders(<PharmacyStock today={TODAY27} />);
    fireEvent.click(await screen.findByLabelText("Todos os locais"));
    fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
    await waitFor(() => expect(mocked(api.listPositions)).toHaveBeenLastCalledWith({ locationId: null, q: null, expiring: undefined, belowMinimum: undefined, before: null }));
  });

  it("extrato do lote: movimentos com sinal e documento; dispensação sem nome de paciente", async () => {
    renderWithProviders(<PharmacyStock today={TODAY27} />);
    fireEvent.click(await screen.findByRole("button", { name: "Lotes de Amoxicilina 500 mg cápsula" }));
    fireEvent.click(await screen.findByRole("button", { name: "Extrato do lote AMX2611" }));
    const table = within(await screen.findByRole("region", { name: "Extrato do lote AMX2611" }));
    expect((await table.findByRole("row", { name: /dispensação/ })).textContent).toContain("-21");
    expect(table.getByRole("row", { name: /entrada por transferência/ }).textContent).toContain("+51");
    expect(mocked(api.listLotMovements)).toHaveBeenCalledWith("lot1", "l2", null);
    expect(document.body.textContent).not.toMatch(/Joana/);
  });

  it("mínimos: a lista inteira, vírgula e zero aceito", async () => {
    renderWithProviders(<PharmacyStock today={TODAY27} />);
    const input = await screen.findByLabelText("Mínimo de Amoxicilina 500 mg cápsula (cápsula)");
    fireEvent.change(input, { target: { value: "60.5" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar mínimos" }));
    expect((await screen.findByRole("alert")).textContent).toContain("use vírgula");
    fireEvent.change(input, { target: { value: "60,5" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar mínimos" }));
    await waitFor(() => expect(mocked(api.putMinimums)).toHaveBeenCalledWith("l2", [ { catalog_item_id: "ci1", minimum: "60.5" } ]));
    expect((await screen.findByRole("status")).textContent).toBe("Mínimos salvos.");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyStock.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** com `useInfiniteQuery` (`STATIC_QUERY`, `getNextPageParam: (p) => p.next_before ?? undefined`) para posição e extrato; os filtros só valem no "Filtrar" (o campo digitado não lê sozinho; "Medicamento" com 1 letra → "digite ao menos 2 letras do medicamento", sem ler); `movementLabel`: `opening` "abertura", `receipt` "entrada por nota", `donation` "doação", `transfer_out` "saída por transferência", `transfer_in` "entrada por transferência", `dispensing` "dispensação", `loss` "perda", `expiry` "vencimento", `adjustment` "ajuste", `return` "devolução"; a coluna do documento mostra o número quando houver ("NF 1234") e, para dispensação e transferência, só o tipo. O painel de mínimos aparece só quando `locationRoles(loc).keeper`; zero vale; linha com problema não deixa salvar e a frase diz o item.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyStock.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyStock.tsx src/modules/PharmacyStock.test.tsx src/modules/pharmacy/LotStatement.tsx src/modules/pharmacy/MinimumsPanel.tsx
$G commit -m "feat: show pharmacy stock position, lot statement and minimums

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Transferência do almoxarifado e recebimento na unidade (F-27.3)

**Files:**
- Create: `src/modules/pharmacy/TransferForm.tsx`, `src/modules/pharmacy/ReceiveForm.tsx`
- Modify: `src/modules/PharmacyTransfers.tsx`
- Test: `src/modules/PharmacyTransfers.test.tsx`

**Interfaces:**
- Consumes: `listMyPharmacyLocations`, `listPositions`, `sendTransfer`, `receiveTransfer`, `listTransfers` (Task 1); `availableLots`, `suggestedLotId`, `lotOptionLabel`, `transferProblems`, `receiveProblems`, `receiveBody`, `hasDifference`, `transferStatusLabel`, `fmtQuantity`, `pharmacyError`, `PH_TRANSFERS_KEY`, `PH_POSITIONS_KEY`, `STATIC_QUERY` (Task 2); `LocationPick` (Task 3).
- Produces: `PharmacyTransfers({ today? })` — no central, para o `stock_keeper`: `TransferForm` ("Destino": locais `unit` ativos de `listMyPharmacyLocations` — D1 —; linhas: medicamento entre as posições do central, "Lote" com o que vence primeiro pré-escolhido e marcado "(sugerido)", "Quantidade" até o saldo; "Enviar transferência"); em qualquer local, a lista de transferências do local (origem, destino, enviada em, situação; `Tag` por situação); no destino, "Receber" numa `sent` abre `ReceiveForm` (por lote: enviado, "Recebido" em branco — conferência, nunca pré-preenchida —, e "Motivo da diferença" quando difere; "Confirmar recebimento").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyTransfers.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), listPositions: vi.fn(), sendTransfer: vi.fn(), receiveTransfer: vi.fn(), listTransfers: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { PharmacyTransfers } from "./PharmacyTransfers";
import { renderWithProviders } from "../test/campaignFixtures";
import { CENTRAL, CENTRAL_REF, TODAY27, UBS_FARM, lotBalance, phUser, position, stockLot, transfer } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("PharmacyTransfers", () => {
  beforeEach(() => {
    for (const fn of [ api.sendTransfer, api.receiveTransfer, api.listTransfers ]) mocked(fn).mockReset();
    mocked(api.listTransfers).mockResolvedValue({ transfers: [ transfer() ], next_before: null });
  });

  it("central envia: destino, lote que vence primeiro sugerido, quantidade até o saldo; corpo", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "stock_keeper" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ CENTRAL, { ...UBS_FARM, roles: [] } ]);
    mocked(api.listPositions).mockResolvedValue({ positions: [ position({ location: CENTRAL_REF, balance: "600", lots: [
      lotBalance({ lot: stockLot({ id: "lotB", lot_number: "AMX2702", expires_on: "2027-02-01" }), balance: "500" }),
      lotBalance({ balance: "100" }) ] }) ], next_before: null });
    mocked(api.sendTransfer).mockResolvedValue(transfer());
    renderWithProviders(<PharmacyTransfers today={TODAY27} />);
    await screen.findByLabelText("Destino"); text("Destino", "l2");
    fireEvent.click(await screen.findByRole("button", { name: "Acrescentar Amoxicilina 500 mg cápsula" }));
    expect((screen.getByLabelText("Lote") as HTMLSelectElement).value).toBe("lot1");
    expect(screen.getByRole("option", { name: "AMX2611 · vence 20/11/2026 · saldo 100 (sugerido)" })).not.toBeNull();
    text("Quantidade", "150");
    fireEvent.click(screen.getByRole("button", { name: "Enviar transferência" }));
    expect((await screen.findByRole("alert")).textContent).toBe("Amoxicilina 500 mg cápsula: saída maior que o saldo do lote (100)");
    text("Quantidade", "100");
    fireEvent.click(screen.getByRole("button", { name: "Enviar transferência" }));
    await waitFor(() => expect(mocked(api.sendTransfer)).toHaveBeenCalledWith({ to_location_id: "l2", items: [ { lot_id: "lot1", quantity: "100" } ] }));
    expect((await screen.findByRole("status")).textContent).toBe("Transferência enviada: fica em trânsito até a unidade receber.");
  });

  it("unidade recebe: conferência lote a lote; diferença pede motivo; frase da diferença", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacy_assistant" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ { ...UBS_FARM, roles: [ "pharmacy_assistant" ] } ]);
    mocked(api.receiveTransfer).mockResolvedValue(transfer({ status: "received_with_difference" }));
    renderWithProviders(<PharmacyTransfers today={TODAY27} />);
    expect(screen.queryByRole("button", { name: "Enviar transferência" })).toBeNull();
    fireEvent.click(await screen.findByRole("button", { name: "Receber transferência de Almoxarifado Central" }));
    expect((screen.getByLabelText("Recebido do lote AMX2611 (enviado 100)") as HTMLInputElement).value).toBe("");
    text("Recebido do lote AMX2611 (enviado 100)", "90");
    fireEvent.click(screen.getByRole("button", { name: "Confirmar recebimento" }));
    expect((await screen.findByRole("alert")).textContent).toBe("AMX2611: recebido diferente do enviado — escreva o motivo (10 a 500 caracteres)");
    text("Motivo da diferença do lote AMX2611", "uma caixa de 10 chegou molhada");
    fireEvent.click(screen.getByRole("button", { name: "Confirmar recebimento" }));
    await waitFor(() => expect(mocked(api.receiveTransfer)).toHaveBeenCalledWith("t1",
      { items: [ { lot_id: "lot1", quantity_received: "90", reason: "uma caixa de 10 chegou molhada" } ] }));
    expect((await screen.findByRole("status")).textContent).toBe("Recebida com diferença: a diferença entrou como perda, com o motivo.");
  });

  it("já recebida por outra pessoa: frase e a lista relida", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ UBS_FARM ]);
    mocked(api.receiveTransfer).mockRejectedValue(new ApiError(409, { error: "already_received" }, "409"));
    renderWithProviders(<PharmacyTransfers today={TODAY27} />);
    fireEvent.click(await screen.findByRole("button", { name: "Receber transferência de Almoxarifado Central" }));
    text("Recebido do lote AMX2611 (enviado 100)", "100");
    fireEvent.click(screen.getByRole("button", { name: "Confirmar recebimento" }));
    expect(await screen.findByText("esta transferência já foi recebida — a lista foi relida")).not.toBeNull();
    await waitFor(() => expect(mocked(api.listTransfers).mock.calls.length).toBeGreaterThanOrEqual(2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyTransfers.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente.** O envio só aparece quando o local é `central` e `locationRoles(loc).keeper`; as posições do central vêm de `listPositions({ locationId: central.id })` (`STATIC_QUERY`; botão "Acrescentar <nome>" por posição com lote disponível); o seletor de lote usa `availableLots` e `lotOptionLabel` (`"<lote> · vence <dd/mm/aaaa> · saldo <q>"` + `" (sugerido)"` no primeiro) e pré-escolhe `suggestedLotId`; o recebimento só aparece no destino (`to.id === loc.id`) e em `sent`. Frases: `received` → "Recebida."; `received_with_difference` → a do teste. Depois de cada escrita, `PH_TRANSFERS_KEY` e `PH_POSITIONS_KEY` são relidas; `already_received`/`not_found` também releem.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyTransfers.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyTransfers.tsx src/modules/PharmacyTransfers.test.tsx src/modules/pharmacy/TransferForm.tsx src/modules/pharmacy/ReceiveForm.tsx
$G commit -m "feat: send stock transfers and receive them lot by lot

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Inventário, ajustes, perdas e aprovação (F-27.4)

**Files:**
- Create: `src/modules/pharmacy/AdjustmentForm.tsx`, `src/modules/pharmacy/PendingAdjustments.tsx`
- Modify: `src/modules/PharmacyAdjustments.tsx`
- Test: `src/modules/PharmacyAdjustments.test.tsx`

**Interfaces:**
- Consumes: `listPositions`, `recordAdjustment`, `approveAdjustment`, `rejectAdjustment`, `listAdjustments` (Task 1); `lotsWithBalance`, `adjustmentProblem`, `adjustmentDelta`, `adjustmentBody`, `adjustmentKindLabel`, `fmtQuantity`, `fmtSigned`, `reasonProblem`, `pharmacyError`, `PH_ADJUSTMENTS_KEY`, `PH_POSITIONS_KEY`, `STATIC_QUERY` (Task 2); `LocationPick`, `SensitiveAction`; `useAuth` (o id da sessão).
- Produces: `PharmacyAdjustments({ today?, onGoToSecurity? })` — `LocationPick` com `allow: pharmacist || keeper`; `AdjustmentForm` (medicamento entre as posições do local, "Lote" entre `lotsWithBalance` — em "Vencimento", só os lotes já vencidos (`lot_not_expired`, P6 do plano do api) —, "Tipo": "Contagem de inventário" (`adjustment`, campo "Quantidade contada") | "Perda" | "Vencimento" (campo "Quantidade"), "Motivo"; "Revisar" mostra "Saldo 30 → 20 (−10)" e abre o `SensitiveAction` com step-up); `PendingAdjustments` (ajustes `pending` do local: lote, tipo, saldo antes, contado, diferença, motivo, por; "Aprovar" e "Rejeitar" — este com "Motivo da rejeição" 10–500 —, ambos com step-up; o ajuste da própria pessoa não oferece os dois e diz "aguardando outra pessoa do local").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyAdjustments.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), listPositions: vi.fn(), recordAdjustment: vi.fn(), approveAdjustment: vi.fn(), rejectAdjustment: vi.fn(), listAdjustments: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { PharmacyAdjustments } from "./PharmacyAdjustments";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY27, UBS_FARM, adjustment, phUser, position } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("PharmacyAdjustments", () => {
  beforeEach(() => {
    for (const fn of [ api.recordAdjustment, api.approveAdjustment, api.rejectAdjustment, api.listAdjustments ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ UBS_FARM ]);
    mocked(api.listPositions).mockResolvedValue({ positions: [ position() ], next_before: null });
    mocked(api.listAdjustments).mockResolvedValue({ adjustments: [ adjustment(), adjustment({ id: "a2", by: { user_id: "u-1", name: "Eu Mesma" } }) ], next_before: null });
  });

  it("contagem: a diferença antes; step-up; ajuste acima do limite fica pendente", async () => {
    mocked(api.recordAdjustment).mockResolvedValue(adjustment({ id: "a3", status: "pending" }));
    renderWithProviders(<PharmacyAdjustments today={TODAY27} />);
    await screen.findByLabelText("Medicamento"); text("Medicamento", "ci1");
    text("Lote", "lot1"); text("Tipo", "adjustment"); text("Quantidade contada", "20"); text("Motivo", "contagem do fim do mês");
    fireEvent.click(screen.getByRole("button", { name: "Revisar ajuste" }));
    expect(await screen.findByText("Saldo 30 → 20 (-10)")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Registrar ajuste" }));
    await waitFor(() => expect(mocked(api.recordAdjustment)).toHaveBeenCalledWith({ location_id: "l2", lot_id: "lot1", kind: "adjustment",
      counted_quantity: "20", reason: "contagem do fim do mês" }));
    expect((await screen.findByRole("status")).textContent).toBe("Ajuste acima do limite da cidade: aguardando a aprovação de outra pessoa do local.");
  });

  it("perda maior que o saldo recusada antes; perda aplicada", async () => {
    mocked(api.recordAdjustment).mockResolvedValue(adjustment({ kind: "loss", status: "applied", quantity: "-2", counted_quantity: null }));
    renderWithProviders(<PharmacyAdjustments today={TODAY27} />);
    await screen.findByLabelText("Medicamento"); text("Medicamento", "ci1");
    text("Lote", "lot1"); text("Tipo", "loss"); text("Quantidade", "31"); text("Motivo", "frasco caiu e quebrou no balcão");
    fireEvent.click(screen.getByRole("button", { name: "Revisar ajuste" }));
    expect((await screen.findByRole("alert")).textContent).toBe("saída maior que o saldo do lote (30)");
    text("Quantidade", "2");
    fireEvent.click(screen.getByRole("button", { name: "Revisar ajuste" }));
    fireEvent.click(await screen.findByRole("button", { name: "Registrar ajuste" }));
    expect((await screen.findByRole("status")).textContent).toBe("Perda registrada.");
  });

  it("pendentes: aprovar e rejeitar com motivo; o próprio não oferece; 409 relê", async () => {
    mocked(api.approveAdjustment).mockRejectedValueOnce(new ApiError(409, { error: "not_pending" }, "409"));
    mocked(api.rejectAdjustment).mockResolvedValue(adjustment({ status: "rejected" }));
    renderWithProviders(<PharmacyAdjustments today={TODAY27} />);
    const panel = within(await screen.findByRole("region", { name: "Ajustes aguardando aprovação" }));
    const mine = await panel.findByRole("row", { name: /Eu Mesma/ });
    expect(within(mine).queryByRole("button", { name: /Aprovar|Rejeitar/ })).toBeNull();
    expect(mine.textContent).toContain("aguardando outra pessoa do local");
    const other = panel.getByRole("row", { name: /Paulo Almeida/ });
    fireEvent.click(within(other).getByRole("button", { name: "Aprovar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Aprovar ajuste" }));
    expect(await screen.findByText("este ajuste já foi decidido — a lista foi relida")).not.toBeNull();
    await waitFor(() => expect(mocked(api.listAdjustments).mock.calls.length).toBeGreaterThanOrEqual(2));
    fireEvent.click(within(panel.getByRole("row", { name: /Paulo Almeida/ })).getByRole("button", { name: "Rejeitar" }));
    fireEvent.change(await screen.findByLabelText("Motivo da rejeição"), { target: { value: "contagem refeita, saldo confere" } });
    fireEvent.click(screen.getByRole("button", { name: "Rejeitar ajuste" }));
    await waitFor(() => expect(mocked(api.rejectAdjustment)).toHaveBeenCalledWith("a1", "contagem refeita, saldo confere"));
    expect(mocked(api.listAdjustments)).toHaveBeenCalledWith({ locationId: "l2", status: "pending", before: null });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyAdjustments.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente.** Perda e vencimento nunca vão a aprovação (P6 do plano do api); só a contagem pode ficar pendente. A resposta decide a frase: `applied` → "Ajuste registrado." / "Perda registrada." / "Vencimento registrado."; `pending` → a do teste (o limite é do api; a tela não o calcula). "Rejeitar" usa o `SensitiveAction` com o campo `reason` (`fields: [ { name: "reason", label: "Motivo da rejeição", required: true, multiline: true } ]`) e confere `reasonProblem` antes de chamar; `same_user`, `not_pending` e `not_found` releem `PH_ADJUSTMENTS_KEY`. O id da sessão (`useAuth().user.id`) decide "o próprio" (`adjustment.by?.user_id`, D10). Nenhum botão de editar ou apagar movimento.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyAdjustments.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyAdjustments.tsx src/modules/PharmacyAdjustments.test.tsx src/modules/pharmacy/AdjustmentForm.tsx src/modules/pharmacy/PendingAdjustments.tsx
$G commit -m "feat: record stock adjustments with step-up and second approval

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Busca da receita pelo código ou pelo paciente (F-27.5)

**Files:**
- Create: `src/modules/pharmacy/PrescriptionLookup.tsx`, `src/modules/pharmacy/PrescriptionCard.tsx`
- Modify: `src/modules/PharmacyDispensing.tsx`
- Test: `src/modules/PharmacyDispensing.test.tsx`

**Interfaces:**
- Consumes: `lookupPrescriptions`, `lookupPharmacyPerson`, `findPharmacyDuplicates` (Task 1); `normalizeShortCode`, `prescriptionBlock`, `NO_PATIENT_RECORD`, `pharmacyError`, `locationRoles` (Task 2); `LocationPick` (Task 3); `PersonStart`, `PersonSource` (24a); `fmtDay`.
- Produces:
  - `PHARMACY_PERSON_SOURCE: PersonSource` (em `PrescriptionLookup.tsx`): `lookup` → `lookupPharmacyPerson`, `duplicates` → `findPharmacyDuplicates`, `describe` → `pharmacyError`, `pickLabel` → `` `Usar ${name}` ``;
  - `PrescriptionLookup({ location, onFound(list: PrescriptionView[], patientId: string | null) })` — "Pelo código" (campo "Código da receita", "Buscar receita"; código inválido → "o código da receita tem 10 letras e números (ex.: ABCDE-23456)", sem chamar) e "Pelo paciente" (`PersonStart` com a fonte da farmácia; `patient` → busca pelo `patient_id`; `new` — ninguém com o CPF — ou `citizen` → `NO_PATIENT_RECORD`); cada busca é uma `useMutation` no clique;
  - `PrescriptionCard({ p, onDispense?() })` — "Receita digital"/"Receita externa", paciente (nome, "sem CPF" quando `has_cpf` é falso, `cns_masked`), "emitida em dd/mm/aaaa · vale até dd/mm/aaaa", profissional, itens com a quantidade e a posologia, e o bloqueio da receita (`Tag` `down`) **no lugar** do botão "Dispensar";
  - `PharmacyDispensing({ today? })` — `LocationPick` com `allow: pharmacist || assistant`; a busca, os cartões e, ao "Dispensar", o `DispenseForm` da Task 10 (esta task deixa o slot com `EmptyState` provisório); depois de uma dispensação, a mesma busca é refeita **uma vez** e aparece "Dispensação registrada.".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyDispensing.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), lookupPrescriptions: vi.fn(), lookupPharmacyPerson: vi.fn(), findPharmacyDuplicates: vi.fn(),
  listPositions: vi.fn(), recordDispensation: vi.fn(), listDispensations: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { PharmacyDispensing } from "./PharmacyDispensing";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY27, UBS_FARM, phUser, prescription } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("PharmacyDispensing — busca", () => {
  beforeEach(() => {
    for (const fn of [ api.lookupPrescriptions, api.lookupPharmacyPerson ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ UBS_FARM ]);
    mocked(api.lookupPrescriptions).mockResolvedValue([ prescription() ]);
  });

  it("pelo código: normaliza e busca com o local; inválido não chama", async () => {
    renderWithProviders(<PharmacyDispensing today={TODAY27} />);
    await screen.findByLabelText("Código da receita"); text("Código da receita", "ABC");
    fireEvent.click(screen.getByRole("button", { name: "Buscar receita" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o código da receita tem 10 letras e números (ex.: ABCDE-23456)");
    expect(mocked(api.lookupPrescriptions)).not.toHaveBeenCalled();
    text("Código da receita", "abcde-23456");
    fireEvent.click(screen.getByRole("button", { name: "Buscar receita" }));
    await waitFor(() => expect(mocked(api.lookupPrescriptions)).toHaveBeenCalledWith({ location_id: "l2", short_code: "ABCDE23456" }));
    const card = within(await screen.findByRole("article", { name: /Receita digital/ }));
    expect(card.getByText("Joana Lima")).not.toBeNull();
    expect(card.getByText("emitida em 27/10/2026 · vale até 26/11/2026")).not.toBeNull();
    expect(card.getByText(/1 cápsula de 8 em 8 horas por 7 dias/)).not.toBeNull();
    expect(card.getByRole("button", { name: "Dispensar" })).not.toBeNull();
  });

  it("receita vencida ou cancelada: o motivo no lugar de Dispensar", async () => {
    mocked(api.lookupPrescriptions).mockResolvedValue([ prescription({ expired: true, valid_until: "2026-10-20" }), prescription({ id: "doc2", cancelled: true }) ]);
    renderWithProviders(<PharmacyDispensing today={TODAY27} />);
    await screen.findByLabelText("Código da receita"); text("Código da receita", "ABCDE23456");
    fireEvent.click(screen.getByRole("button", { name: "Buscar receita" }));
    expect(await screen.findByText("receita vencida em 20/10/2026 — não pode ser dispensada")).not.toBeNull();
    expect(screen.getByText("receita cancelada — não pode ser dispensada")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Dispensar" })).toBeNull();
  });

  it("pelo paciente: CPF pela rota da farmácia; sem paciente, aponta a receita externa", async () => {
    mocked(api.lookupPharmacyPerson).mockResolvedValueOnce({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null, has_cpf: true, cns_masked: null })
      .mockRejectedValueOnce(new ApiError(404, { error: "person_not_found" }, "404"));
    renderWithProviders(<PharmacyDispensing today={TODAY27} />);
    fireEvent.click(await screen.findByLabelText("Pelo paciente"));
    text("CPF", "123.456.789-09");
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Usar Joana Lima" }));
    await waitFor(() => expect(mocked(api.lookupPrescriptions)).toHaveBeenCalledWith({ location_id: "l2", patient_id: "pa1" }));
    expect(mocked(api.lookupPharmacyPerson)).toHaveBeenCalledWith("12345678909");
    fireEvent.click(screen.getByRole("button", { name: "Trocar paciente" }));
    text("CPF", "987.654.321-00");
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar pessoa nova com este CPF" }));
    expect(await screen.findByText("esta pessoa ainda não tem receita no Rota Saúde — se trouxe receita de fora da rede, use Receita externa")).not.toBeNull();
    expect(mocked(api.lookupPrescriptions)).toHaveBeenCalledTimes(1);
  });

  it("sem local da pessoa: a frase; nada guardado no navegador", async () => {
    mocked(api.listMyPharmacyLocations).mockResolvedValue([]);
    renderWithProviders(<PharmacyDispensing today={TODAY27} />);
    expect(await screen.findByText("você não está vinculado a nenhum local de estoque — peça ao administrador da cidade")).not.toBeNull();
    expect(window.localStorage.length + window.sessionStorage.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyDispensing.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente.** "Pelo código" e "Pelo paciente" são dois `radio` (o primeiro marcado); o cartão é um `<article aria-label="Receita digital de <dd/mm/aaaa>">` (ou "Receita externa …"); depois de escolher a pessoa, "Trocar paciente" volta ao `PersonStart`. A lista encontrada vive só no estado da tela (sai ao trocar de busca); nada de `useQuery` com a receita no cache. Nenhum log do código digitado.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyDispensing.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyDispensing.tsx src/modules/PharmacyDispensing.test.tsx src/modules/pharmacy/PrescriptionLookup.tsx src/modules/pharmacy/PrescriptionCard.tsx
$G commit -m "feat: look up prescriptions by code or patient at the pharmacy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Dispensação item a item — lote sugerido, contínuo, texto livre e controlado (F-27.5, F-27.6)

**Files:**
- Create: `src/modules/pharmacy/DispenseForm.tsx`
- Modify: `src/modules/PharmacyDispensing.tsx` (o slot da Task 9 passa a abrir o `DispenseForm`)
- Test: `src/modules/pharmacy/DispenseForm.test.tsx`

**Interfaces:**
- Consumes: `listPositions`, `recordDispensation`, `searchPharmacyItems` (Task 1); `itemBlock`, `prescriptionBlock`, `availableLots`, `suggestedLotId`, `lotOptionLabel`, `dispenseProblems`, `dispenseBody`, `needsRetention`, `EMPTY_DISPENSE`, `fmtQuantity`, `pharmacyError`, `pharmacyProblems`, `BALANCE_CHANGED`, `PRESCRIPTION_CHANGED`, `PH_POSITIONS_KEY`, `STATIC_QUERY` (Task 2); `ItemSearch` (Task 5).
- Produces: `DispenseForm({ location: PharmacyLocation, prescription: PrescriptionView, today?, delayMs?, onDone(d: Dispensation): void, onStale(): void, onCancel(): void })` — por item da receita: o texto (descrição, "Prescrito: 21 cápsula", posologia, `Tag` "uso contínuo" e "controlado · lista B1", "Já dispensado: 21 · última retirada dd/mm/aaaa"); o bloqueio do item (`itemBlock`) **no lugar** dos campos; senão: no texto livre do farmacêutico, `ItemSearch` "Item do catálogo para <texto>"; "Lote — <descrição>" (lotes do item no local, `availableLots`, sugerido pré-escolhido); "Quantidade — <descrição> (<unidade>)"; "Dispensado por completo — <descrição>"; a **retenção** quando há controlado preenchido (a marca "Receita retida na farmácia"; o número retido é o do documento — `sncr.number` ou a notificação da receita externa —, nunca digitado aqui: D14); "Registrar dispensação" (sem step-up); todas as recusas de uma vez numa lista `role="alert"`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/pharmacy/DispenseForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  listPositions: vi.fn(), recordDispensation: vi.fn(), searchPharmacyItems: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { DispenseForm } from "./DispenseForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { AMOXI, CLONA, TODAY27, UBS_FARM, dispensation, lotBalance, phUser, position, prescription, rxItem, stockLot } from "../../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
const AMOX = "Amoxicilina 500 mg cápsula"; const CLON = "Clonazepam 2 mg comprimido";
const CLONA_ITEM = rxItem({ id: "pi2", catalog_item_id: "ci2", printed_description: CLON, quantity: "30", quantity_unit: "comprimido",
  dosage_instructions: "1 comprimido à noite", controlled: true });
const CLONA_POS = position({ item: CLONA, lots: [ lotBalance({ lot: stockLot({ id: "lot2", item: CLONA, lot_number: "CLZ2701", expires_on: "2027-01-10" }), balance: "60" }) ] });

function renderForm(p = prescription(), pharmacist = true, handlers = { onDone: vi.fn(), onStale: vi.fn(), onCancel: vi.fn() }) {
  const loc = { ...UBS_FARM, roles: [ pharmacist ? "pharmacist" : "pharmacy_assistant" ] };
  renderWithProviders(<DispenseForm location={loc} prescription={p} today={TODAY27} delayMs={0} {...handlers} />);
  return handlers;
}

describe("DispenseForm", () => {
  beforeEach(() => {
    for (const fn of [ api.listPositions, api.recordDispensation, api.searchPharmacyItems ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listPositions).mockImplementation(({ catalogItemId }: { catalogItemId: string }) => Promise.resolve({ next_before: null,
      positions: catalogItemId === "ci2" ? [ CLONA_POS ] : [ position({ lots: [
        lotBalance({ lot: stockLot({ id: "lotB", lot_number: "AMX2702", expires_on: "2027-02-01" }), balance: "500" }), lotBalance() ] }) ] }));
    mocked(api.recordDispensation).mockResolvedValue(dispensation());
  });

  it("lote que vence primeiro sugerido; o texto da receita à vista; corpo com completo", async () => {
    const h = renderForm();
    expect(await screen.findByText("Prescrito: 21 cápsula")).not.toBeNull();
    expect(screen.getByText("1 cápsula de 8 em 8 horas por 7 dias")).not.toBeNull();
    await waitFor(() => expect((screen.getByLabelText(`Lote — ${AMOX}`) as HTMLSelectElement).value).toBe("lot1"));
    expect(screen.getByRole("option", { name: "AMX2611 · vence 20/11/2026 · saldo 30 (sugerido)" })).not.toBeNull();
    expect(mocked(api.listPositions)).toHaveBeenCalledWith({ locationId: "l2", catalogItemId: "ci1" });
    text(`Quantidade — ${AMOX} (cápsula)`, "21");
    fireEvent.click(screen.getByLabelText(`Dispensado por completo — ${AMOX}`));
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    await waitFor(() => expect(mocked(api.recordDispensation)).toHaveBeenCalledWith({ location_id: "l2", prescription: { kind: "digital", id: "doc1" },
      items: [ { prescription_item_id: "pi1", lot_id: "lot1", quantity: "21", complete: true } ] }));
    expect(h.onDone).toHaveBeenCalledWith(dispensation());
    expect(screen.queryByText(/retenção|retida/i)).toBeNull();
  });

  it("todas as recusas de uma vez, antes de enviar", async () => {
    renderForm(prescription({ items: [ rxItem(), CLONA_ITEM ] }));
    await waitFor(() => expect((screen.getByLabelText(`Lote — ${CLON}`) as HTMLSelectElement).value).toBe("lot2"));
    text(`Quantidade — ${AMOX} (cápsula)`, "40"); text(`Quantidade — ${CLON} (comprimido)`, "2,555");
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    const items = (await screen.findAllByRole("listitem")).map((li) => li.textContent);
    expect(items).toEqual(expect.arrayContaining([ `${AMOX}: saída maior que o saldo do lote (30)`,
      `${CLON}: a quantidade tem no máximo 2 casas depois da vírgula`, "medicamento controlado: registre a retenção da receita" ]));
    expect(mocked(api.recordDispensation)).not.toHaveBeenCalled();
  });

  it("auxiliar: controlado e texto livre bloqueados com o motivo; contínuo antes do dia", async () => {
    renderForm(prescription({ items: [ CLONA_ITEM, rxItem({ id: "pi3", catalog_item_id: null, free_text: "xarope de guaco", printed_description: "xarope de guaco" }),
      rxItem({ id: "pi4", printed_description: "Losartana 50 mg comprimido", continuous: true, next_allowed_on: "2026-11-03", last_dispensed_on: "2026-10-09", dispensed_total: "30" }) ] }), false);
    expect(await screen.findByText("medicamento controlado: só o farmacêutico dispensa")).not.toBeNull();
    expect(screen.getByText("item em texto livre: só o farmacêutico dispensa, escolhendo o item do catálogo")).not.toBeNull();
    expect(screen.getByText("uso contínuo: próxima retirada a partir de 03/11/2026")).not.toBeNull();
    expect(screen.getByText("Já dispensado: 30 · última retirada 09/10/2026")).not.toBeNull();
    expect(screen.queryByLabelText(`Quantidade — ${CLON} (comprimido)`)).toBeNull();
    expect(screen.queryByLabelText("Quantidade — Losartana 50 mg comprimido (cápsula)")).toBeNull();
  });

  it("texto livre: o farmacêutico escolhe o item do catálogo; a escolha vai no corpo", async () => {
    mocked(api.searchPharmacyItems).mockResolvedValue([ AMOXI ]);
    renderForm(prescription({ items: [ rxItem({ catalog_item_id: null, free_text: "amoxil 500", printed_description: "amoxil 500" }) ] }));
    await screen.findByLabelText("Item do catálogo para amoxil 500"); text("Item do catálogo para amoxil 500", "amox");
    fireEvent.click(await screen.findByRole("button", { name: `${AMOX} · cápsula` }));
    await waitFor(() => expect((screen.getByLabelText("Lote — amoxil 500") as HTMLSelectElement).value).toBe("lot1"));
    text("Quantidade — amoxil 500 (cápsula)", "21");
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    await waitFor(() => expect(mocked(api.recordDispensation).mock.calls[0][0].items).toEqual([
      { prescription_item_id: "pi1", catalog_item_id: "ci1", lot_id: "lot1", quantity: "21" } ]));
  });

  it("controlado: retenção obrigatória; vai a marca, sem número digitado", async () => {
    renderForm(prescription({ items: [ CLONA_ITEM ] }));
    await waitFor(() => expect((screen.getByLabelText(`Lote — ${CLON}`) as HTMLSelectElement).value).toBe("lot2"));
    text(`Quantidade — ${CLON} (comprimido)`, "30");
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    expect((await screen.findAllByRole("listitem")).map((li) => li.textContent)).toContain("medicamento controlado: registre a retenção da receita");
    expect(screen.queryByLabelText(/Número da notificação/)).toBeNull();
    fireEvent.click(screen.getByLabelText("Receita retida na farmácia"));
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    await waitFor(() => expect(mocked(api.recordDispensation).mock.calls[0][0].retention).toEqual({}));
  });

  it("insufficient_balance: relê os lotes e mantém o que foi digitado; too_early com a data", async () => {
    mocked(api.recordDispensation).mockRejectedValueOnce(new ApiError(422, { error: "insufficient_balance", path: "items[0].quantity" }, "422"))
      .mockRejectedValueOnce(new ApiError(422, { error: "too_early", next_allowed_on: "2026-11-03", path: "items[0]" }, "422"));
    renderForm();
    await waitFor(() => expect((screen.getByLabelText(`Lote — ${AMOX}`) as HTMLSelectElement).value).toBe("lot1"));
    const reads = mocked(api.listPositions).mock.calls.length;
    text(`Quantidade — ${AMOX} (cápsula)`, "21");
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    expect(await screen.findByText("o saldo do lote mudou — os lotes foram relidos; confira a quantidade")).not.toBeNull();
    await waitFor(() => expect(mocked(api.listPositions).mock.calls.length).toBeGreaterThan(reads));
    expect((screen.getByLabelText(`Quantidade — ${AMOX} (cápsula)`) as HTMLInputElement).value).toBe("21");
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    expect(await screen.findByText("uso contínuo: próxima retirada deste item a partir de 03/11/2026")).not.toBeNull();
    expect(mocked(api.recordDispensation)).toHaveBeenCalledTimes(2);
  });

  it("receita cancelada enquanto dispensava: frase e nova busca", async () => {
    mocked(api.recordDispensation).mockRejectedValue(new ApiError(422, { error: "prescription_cancelled" }, "422"));
    const h = renderForm();
    await waitFor(() => expect((screen.getByLabelText(`Lote — ${AMOX}`) as HTMLSelectElement).value).toBe("lot1"));
    text(`Quantidade — ${AMOX} (cápsula)`, "21");
    fireEvent.click(screen.getByRole("button", { name: "Registrar dispensação" }));
    expect(await screen.findByText("a receita mudou enquanto você dispensava — busque a receita de novo")).not.toBeNull();
    expect(h.onStale).toHaveBeenCalledTimes(1);
    expect(mocked(api.recordDispensation)).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/pharmacy/DispenseForm.test.tsx`
Expected: FAIL — o componente não existe.

- [ ] **Step 3: Implemente `DispenseForm`.** Uma `useQuery` por item com catálogo (chave `[PH_POSITIONS_KEY, location.id, catalogItemId]`, `STATIC_QUERY`, `listPositions({ locationId, catalogItemId })`); o lote sugerido entra no rascunho quando os lotes chegam e o campo ainda está vazio (nunca troca a escolha da pessoa). `balances` para `dispenseProblems` vêm dos lotes lidos. O envio passa por `useMutation`, sem novo envio sozinho. Recusas: `insufficient_balance` e `lot_expired` → invalida `[PH_POSITIONS_KEY, location.id]` e mantém tudo; `prescription_cancelled`/`prescription_expired`/`not_found` → frase e `onStale()`; `too_early`, `controlled_requires_pharmacist`, `free_text_requires_pharmacist`, `retention_required` → frase; com `problems` (D20), uma linha por problema; com só `path` `items[i]…`, a frase prefixada pela descrição do item i. O bloqueio da receita inteira (`prescriptionBlock`) não deixa o formulário abrir (o cartão da Task 9 já não oferece). Em `PharmacyDispensing.tsx`, "Dispensar" abre o formulário no lugar dos cartões; `onDone` fecha, mostra "Dispensação registrada." e refaz a última busca uma vez; `onStale` fecha e deixa a frase.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/pharmacy/DispenseForm.test.tsx src/modules/PharmacyDispensing.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/pharmacy/DispenseForm.tsx src/modules/pharmacy/DispenseForm.test.tsx src/modules/PharmacyDispensing.tsx
$G commit -m "feat: dispense prescription items with suggested lot and retention

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Receita externa e controlado de fora da rede (F-27.6)

**Files:**
- Create: `src/modules/pharmacy/ExternalPrescriptionForm.tsx`
- Modify: `src/modules/PharmacyExternal.tsx`
- Test: `src/modules/PharmacyExternal.test.tsx`

**Interfaces:**
- Consumes: `createExternalPrescription`, `lookupPharmacyPerson`, `findPharmacyDuplicates` (Task 1); `prescriberProblem`, `issuedOnProblem`, `notificationNumberProblem`, `quantityProblem`, `toApiQuantity`, `COUNCILS`, `UFS`, `pharmacyError`, `locationRoles` (Task 2); `LocationPick` (Task 3); `ItemSearch` (Task 5); `PHARMACY_PERSON_SOURCE` (Task 9); `DispenseForm` (Task 10); `PersonStart`, `newPatientProblem`, `SEX_OPTIONS` e o fieldset "Pessoa nova" no molde do `ManualNotificationPanel` do 25.
- Produces: `PharmacyExternal({ today?, delayMs? })` — `LocationPick` com `allow: pharmacist || assistant`; `PersonStart` → `ExternalPrescriptionForm({ choice, location, today, delayMs?, onCreated(p: PrescriptionView), onRestart(r: PersonRestart), onChangePerson() })` com "Pessoa nova" (como no 25), "Nome do profissional", "Conselho", "UF do conselho", "Número no conselho", "Data da receita", "Número da notificação" (só para o farmacêutico; obrigatório com item de lista A ou B — `NOTIFICATION_LISTS`), itens (`ItemSearch` "Acrescentar medicamento", por item "Quantidade prescrita", "Unidade" — padrão a do catálogo —, "Posologia", "Uso contínuo", "Remover"); "Registrar receita externa"; ao criar, o `DispenseForm` da receita criada abre em seguida.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyExternal.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), lookupPharmacyPerson: vi.fn(), findPharmacyDuplicates: vi.fn(), searchPharmacyItems: vi.fn(),
  createExternalPrescription: vi.fn(), listPositions: vi.fn(), recordDispensation: vi.fn() }));
import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { PharmacyExternal } from "./PharmacyExternal";
import { renderWithProviders } from "../test/campaignFixtures";
import { AMOXI, CLONA, TODAY27, UBS_FARM, phUser, position, prescription } from "../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const text = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

async function choosePatient() {
  await screen.findByLabelText("CPF"); text("CPF", "123.456.789-09");
  fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
  fireEvent.click(await screen.findByRole("button", { name: "Usar Joana Lima" }));
}
function prescriber(date = "2026-10-28") {
  text("Nome do profissional", "Dr. Ivo Teles"); text("Conselho", "CRM"); text("UF do conselho", "SC"); text("Número no conselho", "45678"); text("Data da receita", date);
}
async function addItem(name: string, unit: string) {
  text("Acrescentar medicamento", name.slice(0, 5).toLowerCase());
  fireEvent.click(await screen.findByRole("button", { name: `${name} · ${unit}` }));
}

describe("PharmacyExternal", () => {
  beforeEach(() => {
    for (const fn of [ api.createExternalPrescription, api.searchPharmacyItems ]) mocked(fn).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ UBS_FARM ]);
    mocked(api.lookupPharmacyPerson).mockResolvedValue({ patient_id: "pa1", citizen_id: null, display_name: "Joana Lima", registration_id: null, has_cpf: true, cns_masked: null });
    mocked(api.searchPharmacyItems).mockImplementation((q: string) => Promise.resolve(q.startsWith("clona") ? [ CLONA ] : [ AMOXI ]));
    mocked(api.listPositions).mockResolvedValue({ positions: [ position() ], next_before: null });
    mocked(api.createExternalPrescription).mockResolvedValue(prescription({ kind: "external", id: "ex1", short_code: null }));
  });

  it("paciente, profissional, item; corpo; a dispensação abre em seguida", async () => {
    renderWithProviders(<PharmacyExternal today={TODAY27} delayMs={0} />);
    await choosePatient(); prescriber();
    await addItem("Amoxicilina 500 mg cápsula", "cápsula");
    expect((screen.getByLabelText("Unidade") as HTMLInputElement).value).toBe("cápsula");
    text("Quantidade prescrita", "21"); text("Posologia", "1 cápsula de 8 em 8 horas por 7 dias");
    expect(screen.queryByLabelText("Número da notificação")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Registrar receita externa" }));
    await waitFor(() => expect(mocked(api.createExternalPrescription)).toHaveBeenCalledWith({ patient_id: "pa1",
      prescriber: { name: "Dr. Ivo Teles", council: "CRM", council_state: "SC", council_number: "45678" }, issued_on: "2026-10-28",
      items: [ { catalog_item_id: "ci1", quantity: "21", quantity_unit: "cápsula", dosage_instructions: "1 cápsula de 8 em 8 horas por 7 dias", continuous: false } ] }));
    expect(await screen.findByRole("button", { name: "Registrar dispensação" })).not.toBeNull();
  });

  it("auxiliar: controlado recusado antes, sem campo de notificação", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacy_assistant" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ { ...UBS_FARM, roles: [ "pharmacy_assistant" ] } ]);
    renderWithProviders(<PharmacyExternal today={TODAY27} delayMs={0} />);
    await choosePatient(); prescriber();
    expect(screen.queryByLabelText("Número da notificação")).toBeNull();
    await addItem("Clonazepam 2 mg comprimido", "comprimido");
    text("Quantidade prescrita", "30"); text("Posologia", "1 comprimido à noite");
    fireEvent.click(screen.getByRole("button", { name: "Registrar receita externa" }));
    expect((await screen.findAllByRole("listitem")).map((li) => li.textContent))
      .toContain("Clonazepam 2 mg comprimido: receita externa de controlado só o farmacêutico registra");
    expect(mocked(api.createExternalPrescription)).not.toHaveBeenCalled();
  });

  it("data no futuro, conselho incompleto e sem item: todas de uma vez; a recusa do api fica no formulário", async () => {
    mocked(api.createExternalPrescription).mockRejectedValueOnce(new ApiError(422, { error: "invalid_prescriber", path: "prescriber.council_number" }, "422"));
    renderWithProviders(<PharmacyExternal today={TODAY27} delayMs={0} />);
    await choosePatient(); prescriber("2026-10-30"); text("Número no conselho", "");
    fireEvent.click(screen.getByRole("button", { name: "Registrar receita externa" }));
    expect((await screen.findAllByRole("listitem")).map((li) => li.textContent)).toEqual(expect.arrayContaining([
      "a data da receita não pode ser depois de hoje", "o número no conselho tem só dígitos (até 10)", "acrescente ao menos um medicamento" ]));
    prescriber();
    await addItem("Amoxicilina 500 mg cápsula", "cápsula");
    text("Quantidade prescrita", "21"); text("Posologia", "1 cápsula de 8 em 8 horas");
    fireEvent.click(screen.getByRole("button", { name: "Registrar receita externa" }));
    expect(await screen.findByText("não foi possível concluir (invalid_prescriber) — tente de novo")).not.toBeNull();
    expect((screen.getByLabelText("Nome do profissional") as HTMLInputElement).value).toBe("Dr. Ivo Teles");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyExternal.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** no molde do `CaseForm` do `ManualNotificationPanel` (25): a pessoa no corpo é uma das duas (`patient_id` | `new_patient` + `duplicates_reviewed`; o par validado já vira paciente no `person_lookup` do api), `newPatientProblem` antes; 409 `duplicate_suspected` volta ao `PersonStart` com a busca refeita (`onRestart`). Conferências todas de uma vez (lista `role="alert"` com `<li>`): profissional, data, notificação (`notificationNumberProblem`; só o farmacêutico vê o campo; item de lista A ou B sem número → "receita de lista A ou B exige o número da notificação"), cada item (quantidade, unidade 1–30, posologia 3–500, e "receita externa de controlado só o farmacêutico registra" para controlado de quem não é farmacêutico), "acrescente ao menos um medicamento". Recusa do api: frase do `pharmacyError`; tudo digitado fica. Criada a receita, a tela troca para o `DispenseForm` com ela; `onDone` → "Dispensação registrada." e volta ao começo; `onCancel` → "Receita externa registrada; dispense depois pela busca do paciente.".

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyExternal.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyExternal.tsx src/modules/PharmacyExternal.test.tsx src/modules/pharmacy/ExternalPrescriptionForm.tsx
$G commit -m "feat: register external prescriptions at the pharmacy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Histórico de retiradas do paciente (F-27.5)

**Files:**
- Create: `src/modules/pharmacy/PatientHistory.tsx`
- Modify: `src/modules/PharmacyDispensing.tsx` (botão "Histórico de retiradas" quando há paciente: o da busca por paciente ou o do cartão)
- Test: `src/modules/pharmacy/PatientHistory.test.tsx`

**Interfaces:**
- Consumes: `listDispensations` (Task 1); `fmtQuantity`, `pharmacyRoleLabel`, `pharmacyError`, `PH_DISPENSATIONS_KEY`, `NOMINAL_QUERY` (Task 2); `fmtDateTime`.
- Produces: `PatientHistory({ patientId })` — `useInfiniteQuery` com `NOMINAL_QUERY` (lê ao abrir; "Carregar mais"; sem releitura); tabela: quando, local, itens ("Amoxicilina 500 mg cápsula — 21 (lote AMX2611)", "item da receita em texto livre" quando `mapped_from_free_text`), por (nome e papel), `Tag` "receita retida" quando `retention`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/pharmacy/PatientHistory.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(), listDispensations: vi.fn() }));
import * as api from "../../lib/api";
import { PatientHistory } from "./PatientHistory";
import { renderWithProviders } from "../../test/campaignFixtures";
import { dispensation, phUser } from "../../test/pharmacyFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("PatientHistory", () => {
  beforeEach(() => {
    mocked(api.listDispensations).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listDispensations).mockResolvedValueOnce({ dispensations: [ dispensation({ retention: { notification_number: null } }) ], next_before: "c1" })
      .mockResolvedValueOnce({ dispensations: [ dispensation({ id: "dp0", at: "2026-09-29T09:00:00-03:00",
        items: [ { ...dispensation().items[0], mapped_from_free_text: true } ] }) ], next_before: null });
  });

  it("lê ao abrir, carrega mais pelo cursor e não relê sozinho", async () => {
    renderWithProviders(<PatientHistory patientId="pa1" />);
    const row = await screen.findByRole("row", { name: /29\/10\/2026/ });
    expect(row.textContent).toContain("Amoxicilina 500 mg cápsula — 21 (lote AMX2611)");
    expect(row.textContent).toContain("Marta Souza · Farmacêutico");
    expect(row.textContent).toContain("receita retida");
    expect(mocked(api.listDispensations)).toHaveBeenCalledWith("pa1", null);
    fireEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
    expect((await screen.findByRole("row", { name: /29\/09\/2026/ })).textContent).toContain("item da receita em texto livre");
    expect(mocked(api.listDispensations)).toHaveBeenLastCalledWith("pa1", "c1");
    await act(async () => { window.dispatchEvent(new Event("focus")); window.dispatchEvent(new Event("online")); });
    await waitFor(() => expect(mocked(api.listDispensations)).toHaveBeenCalledTimes(2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/pharmacy/PatientHistory.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** no molde da lista paginada do `PatientTimeline` (cursor, `gcTime: 0`). Em `PharmacyDispensing.tsx`, o botão "Histórico de retiradas" abre o painel só no clique (cada abertura é uma leitura nominal com trilha); o paciente vem da escolha no `PersonStart` ou de `prescription.patient.id` do cartão.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/pharmacy/PatientHistory.test.tsx src/modules/PharmacyDispensing.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/pharmacy/PatientHistory.tsx src/modules/pharmacy/PatientHistory.test.tsx src/modules/PharmacyDispensing.tsx
$G commit -m "feat: show the patient dispensing history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Alertas da farmácia (F-27.7)

**Files:**
- Modify: `src/modules/PharmacyAlerts.tsx`
- Test: `src/modules/PharmacyAlerts.test.tsx`

**Interfaces:**
- Consumes: `getPharmacyAlerts`, `listPositions` (Task 1); `PHARMACY_ALERTS_REFRESH_MS`, `PH_ALERTS_KEY`, `PH_POSITIONS_KEY`, `STATIC_QUERY`, `fmtQuantity`, `pharmacyError`, `locationRoles` (Task 2); `LocationPick` (Task 3).
- Produces: `PharmacyAlerts({ today? })` — `LocationPick` com todos os papéis; as **cinco contagens** do local (`getPharmacyAlerts(loc.id)`, rota sem trilha, `refetchInterval: PHARMACY_ALERTS_REFRESH_MS`): "Vencendo: N", "Vencidos: N", "Abaixo do mínimo: N", "Em trânsito: N", "Ajustes aguardando aprovação: N" (as duas últimas apontam para Transferências e Inventário e ajustes); e duas listas lidas ao abrir (`STATIC_QUERY`, sem releitura): "Vencendo e vencidos" (`listPositions({ locationId, expiring: true })`: lote · medicamento, local, validade, saldo, `Tag` `down` "vencido" quando a validade passou) e "Abaixo do mínimo" (`listPositions({ locationId, belowMinimum: true })`: local, medicamento, "30 cápsula (mínimo 50)"); para o `stock_keeper` do central, a marca "Todos os locais" troca as **listas** para `locationId: null` (as contagens são do local — D15).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/PharmacyAlerts.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listMyPharmacyLocations: vi.fn(), getPharmacyAlerts: vi.fn(), listPositions: vi.fn() }));
import * as api from "../lib/api";
import { PharmacyAlerts } from "./PharmacyAlerts";
import { PHARMACY_ALERTS_REFRESH_MS } from "../lib/pharmacy";
import { renderWithProviders } from "../test/campaignFixtures";
import { CENTRAL, TODAY27, UBS_FARM, lotBalance, phAlerts, phUser, position, stockLot } from "../test/pharmacyFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const EXPIRING = position({ lots: [ lotBalance(), lotBalance({ lot: stockLot({ id: "lot0", lot_number: "AMX2609", expires_on: "2026-09-30" }), balance: "4" }) ] });

describe("PharmacyAlerts", () => {
  beforeEach(() => {
    mocked(api.getPharmacyAlerts).mockReset().mockResolvedValue(phAlerts());
    mocked(api.listPositions).mockReset().mockImplementation((q: { expiring?: boolean }) =>
      Promise.resolve({ positions: [ q.expiring ? EXPIRING : position() ], next_before: null }));
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "pharmacist" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ UBS_FARM ]);
  });

  it("as cinco contagens relidas a cada minuto (sem trilha); as listas leem ao abrir e não releem", async () => {
    vi.useFakeTimers({ shouldAdvanceTime: true });
    renderWithProviders(<PharmacyAlerts today={TODAY27} />);
    for (const t of [ "Vencendo: 2", "Vencidos: 1", "Abaixo do mínimo: 1", "Em trânsito: 1", "Ajustes aguardando aprovação: 1" ]) {
      expect(await screen.findByText(t)).not.toBeNull();
    }
    const expiring = within(await screen.findByRole("region", { name: "Vencendo e vencidos" }));
    expect((await expiring.findByRole("row", { name: /AMX2611 · Amoxicilina 500 mg cápsula/ })).textContent).toContain("20/11/2026");
    expect(within(expiring.getByRole("row", { name: /AMX2609/ })).getByText("vencido")).not.toBeNull();
    expect(within(screen.getByRole("region", { name: "Abaixo do mínimo" })).getByText("30 cápsula (mínimo 50)")).not.toBeNull();
    expect(mocked(api.getPharmacyAlerts)).toHaveBeenCalledWith("l2");
    expect(mocked(api.listPositions)).toHaveBeenCalledWith({ locationId: "l2", expiring: true });
    expect(mocked(api.listPositions)).toHaveBeenCalledWith({ locationId: "l2", belowMinimum: true });
    const reads = mocked(api.listPositions).mock.calls.length;
    await act(async () => { await vi.advanceTimersByTimeAsync(PHARMACY_ALERTS_REFRESH_MS); });
    await waitFor(() => expect(mocked(api.getPharmacyAlerts)).toHaveBeenCalledTimes(2));
    expect(mocked(api.listPositions).mock.calls.length).toBe(reads);
  });

  it("central: as listas de todos os locais; as contagens, do central", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(phUser([ "stock_keeper" ]));
    mocked(api.listMyPharmacyLocations).mockResolvedValue([ CENTRAL ]);
    renderWithProviders(<PharmacyAlerts today={TODAY27} />);
    fireEvent.click(await screen.findByLabelText("Todos os locais"));
    await waitFor(() => expect(mocked(api.listPositions)).toHaveBeenLastCalledWith({ locationId: null, belowMinimum: true }));
    expect(mocked(api.listPositions)).toHaveBeenCalledWith({ locationId: null, expiring: true });
    expect(mocked(api.getPharmacyAlerts)).toHaveBeenCalledWith("l1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyAlerts.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** no molde do `AlertsPanel` do 21c (o mesmo comentário: a rota de contagem não grava trilha; nada de temporizador próprio). Cada lista é um `Panel` com o título como `aria-label`; lista vazia → "nada aqui". As listas usam só a primeira página (50 posições) com "Ver na posição" apontando para **Posição e extrato** quando houver `next_before`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/modules/PharmacyAlerts.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/PharmacyAlerts.tsx src/modules/PharmacyAlerts.test.tsx
$G commit -m "feat: show pharmacy expiry, minimum and pending alerts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Dispensação na linha do tempo e "última retirada" na lista de medicamentos (spec decisão 11)

**Files:**
- Modify: `src/lib/timeline.ts`, `src/lib/timeline.test.ts`, `src/lib/timeline.exam.test.ts`, `src/lib/timeline.procedure.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/consultation/MedicationsBlock.tsx`, `src/modules/consultation/MedicationsBlock.test.tsx`
- Test: `src/lib/timeline.pharmacy.test.ts`

**Interfaces:**
- Consumes: `TimelineKind` com `"dispensation"` e `PatientMedication.last_dispensed_on` (Task 1); `hasFeature`; `fmtDay`.
- Produces: `TIMELINE_KINDS` com `{ key: "dispensation", label: "Dispensação" }` no fim; `timelineKinds(examTracking: boolean, procedureBilling = false, pharmacy = false)`; `statusLabel("dispensed")` → "dispensado"; `PatientTimeline` passa `hasFeature(user, "pharmacy")`; `MedicationsBlock` mostra `Tag` "última retirada dd/mm/aaaa" quando o item tem `last_dispensed_on` (só leitura; nada muda em `patient_medications`).

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/timeline.pharmacy.test.ts
import { describe, expect, it } from "vitest";
import { TIMELINE_KINDS, statusLabel, timelineKindLabel, timelineKinds } from "./timeline";

describe("dispensação na linha do tempo (módulo 27a)", () => {
  it("só com a farmácia ligada; rótulo e situação em português", () => {
    expect(timelineKinds(false, false, true).map((k) => k.key)).toContain("dispensation");
    expect(timelineKinds(true, true).map((k) => k.key)).not.toContain("dispensation");
    expect(timelineKinds(true, true, true)).toEqual(TIMELINE_KINDS);
    expect(timelineKindLabel("dispensation")).toBe("Dispensação");
    expect(statusLabel("dispensed")).toBe("dispensado");
  });
});
```

Em `src/lib/timeline.test.ts`, o caso "dez tipos…" vira "onze tipos do contrato, em português; desconhecido sai cru" com `"dispensation"` depois de `"procedure"`. Em `timeline.exam.test.ts` e `timeline.procedure.test.ts`, `timelineKinds(true, true)).toEqual(TIMELINE_KINDS)` vira `timelineKinds(true, true, true)).toEqual(TIMELINE_KINDS)`. No fim de `src/modules/consultation/MedicationsBlock.test.tsx`, dentro do `describe("MedicationsBlock")`:

```tsx
  it("última retirada na farmácia, só leitura; sem retirada, nada", async () => {
    mocked(api.listPatientMedications).mockResolvedValue([ medication({ last_dispensed_on: "2026-10-20" }), medication({ id: "pm2", label: "Sinvastatina 20 mg" }) ]);
    renderWithProviders(<MedicationsBlock patientId="pa1" draftConsultationId={null} />);
    const list = within(await screen.findByRole("list", { name: "medicamentos em uso" }));
    expect(within(list.getByText("Metformina 850 mg").closest("li")!).getByText("última retirada 20/10/2026")).not.toBeNull();
    expect(within(list.getByText("Sinvastatina 20 mg").closest("li")!).queryByText(/última retirada/)).toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod27a src/lib/timeline.pharmacy.test.ts src/lib/timeline.test.ts src/modules/consultation/MedicationsBlock.test.tsx`
Expected: FAIL — o tipo, o terceiro parâmetro e a marca não existem.

- [ ] **Step 3: Implemente.** Em `timeline.ts`: o tipo no fim de `TIMELINE_KINDS`, o comentário ("a Dispensação, com a farmácia — módulo 27a") e o filtro `(k.key !== "dispensation" || pharmacy)`; `dispensed: "dispensado"` no `STATUS_LABEL` (comentário "Módulo 27a"). A entrada de dispensação não abre detalhe (`detail_ref` nulo, D22): `detailTarget` não muda. Em `PatientTimeline.tsx`, o terceiro argumento `hasFeature(user, "pharmacy")`. Em `MedicationsBlock.tsx`, depois do `Tag` de uso contínuo: `{m.last_dispensed_on && <Tag mono={false}>{`última retirada ${fmtDay(m.last_dispensed_on)}`}</Tag>}`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod27a src/lib/timeline.pharmacy.test.ts src/lib/timeline.test.ts src/lib/timeline.exam.test.ts src/lib/timeline.procedure.test.ts src/modules/timeline src/modules/consultation/MedicationsBlock.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS (os testes da linha do tempo existentes seguem iguais, salvo as três trocas acima).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/timeline.ts src/lib/timeline.test.ts src/lib/timeline.exam.test.ts src/lib/timeline.procedure.test.ts src/lib/timeline.pharmacy.test.ts \
  src/modules/timeline/PatientTimeline.tsx src/modules/consultation/MedicationsBlock.tsx src/modules/consultation/MedicationsBlock.test.tsx
$G commit -m "feat: show dispensing in the patient timeline and medication list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 15: Suíte, build, revisão e prova no navegador (api real na 3049)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–14 na branch `feat/mod-27a-pharmacy-stock`; o api do 27a na **3049** com a semente do 27a.
- Produces: branch pronta para merge (depois do merge do api do 27a); evidências em `evidencias/2026-10-29-modulo-27a/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test dashboard -w dashboard-mod27a && cd $W && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base da Task 0 mais **14 arquivos de teste novos** (`api.pharmacy`, `pharmacy`, `modules.pharmacy`, `PharmacyAdmin`, `PharmacyEntries`, `PharmacyStock`, `PharmacyTransfers`, `PharmacyAdjustments`, `PharmacyDispensing`, `DispenseForm`, `PharmacyExternal`, `PatientHistory`, `PharmacyAlerts`, `timeline.pharmacy`) e os existentes com casos novos (`features.test.ts`, `modules.test.ts`, `team.test.ts`, `timeline.test.ts`, `timeline.exam.test.ts`, `timeline.procedure.test.ts`, `MedicationsBlock.test.tsx`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css package.json package-lock.json src/modules/territory
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css`, dependência ou o `PersonStart` do 24a mudados (o `vite.config.ts` muda só pelo proxy da Task 1).

- [ ] **Step 2: Nada vaza, nada fica no navegador, nada que grava trilha relê sozinho, nada se apaga**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage\|indexedDB" src/lib/pharmacy.ts src/modules/pharmacy src/modules/Pharmacy*.tsx | grep -v "\.test\."
grep -rn "refetchInterval" src/modules/pharmacy src/modules/Pharmacy*.tsx | grep -v "\.test\."
sed -n '/─── Farmácia e almoxarifado (módulo 27a/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [A-Za-z_, :]* })'
grep -rn "maxLength" src/modules/pharmacy src/modules/Pharmacy*.tsx
grep -rni "excluir movimento\|apagar movimento\|editar movimento\|deleteMovement\|updateMovement" src/modules/pharmacy src/modules/Pharmacy*.tsx src/lib/api.ts
```

Expected: o primeiro, o quarto e o quinto sem saída; o segundo só em `PharmacyAlerts.tsx`; o terceiro só com `location_id`, `q`, `catalog_item_id`, `expiring`, `below_minimum`, `status`, `patient_id`, `before` e `limit`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§1–§6, §8), o ADR 0045, o contrato (§1–§9) e as Divergências deste plano. Pontos de atenção:
- sem `pharmacy`, menu, linha do tempo e lista de medicamentos são os de hoje (testes antigos verdes; só contagens e o terceiro argumento mudaram);
- cada tela oferece só o que o papel **no local** permite; o api recusa de novo;
- nunca "Dispensar" em receita cancelada ou vencida; controlado e texto livre só pelo farmacêutico; retenção exigida; lote sempre escolhido; quantidade nunca arredondada;
- `insufficient_balance`, `too_early`, `prescription_cancelled`, `already_received`, `not_pending` e `same_user` mantêm o que foi digitado e releem o que mudou; nada reenviado sozinho;
- busca da receita e do paciente só no clique; histórico com `NOMINAL_QUERY`; só os alertas releem;
- código da receita, CPF, nome, profissional, posologia, motivo e número de notificação só no corpo;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 27a na **3049** (plano do api: servidor do worktree `apps/api/.claude/mod27a`, migrações de cidade e plataforma a partir de `20261029900001` aplicadas no banco de dev **com autorização do usuário**, e a semente do 27a: Curitiba com `pharmacy`; o almoxarifado central e a farmácia da UBS Centro; um almoxarife no central e outro na UBS; uma farmacêutica (CBO 2234) e um auxiliar na UBS; amoxicilina e clonazepam do catálogo; receitas digitais — comum válida, cancelada, vencida, de uso contínuo retirada há 20 dias, de controle especial e com item em texto livre). Ligue em Curitiba, pelo maintenance, `pharmacy`. O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod27a --network rota-saude_default -p 5205:5205 \
  -v $R/.worktrees/dashboard-mod27a:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3049 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5205 --host 0.0.0.0"
docker logs -f dashboard-mod27a   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 27a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5205/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Roteiro, com screenshot de cada passo em `evidencias/2026-10-29-modulo-27a/dashboard/`:
1. **Admin** — Equipe: conceder "Farmacêutico", "Auxiliar de farmácia" e "Almoxarife" pede step-up. Farmácia → **Locais e configurações**: criar a farmácia de outra unidade (step-up); vincular a farmacêutica e o almoxarife à UBS Centro; vincular como farmacêutico alguém sem CBO 2234 → recusa com a frase; configurações 30/180/25/90/10, tentar intervalo 91 → recusa antes.
2. **Almoxarife do central** — **Entradas e abertura**: inventário de abertura do central (dois itens, um lote cada); entrada por nota fiscal "por embalagem" (30 × 10 = 300) com lote e validade; lote vencido recusado antes; repetir a abertura do mesmo item → frase do `opening_exists`.
3. **Transferências** — enviar 100 de amoxicilina e 60 de clonazepam à UBS Centro (o lote que vence primeiro sugerido; quantidade acima do saldo recusada antes); a transferência aparece "enviada"; **Posição** do central mostra o saldo menor e a da UBS, "em trânsito".
4. **Auxiliar da UBS** — receber: digitar 90 de amoxicilina → pede o motivo; com o motivo, "Recebida com diferença…"; o extrato do lote na UBS mostra a entrada por transferência e a perda de 10.
5. **Farmacêutica** — **Dispensação**: buscar pelo código impresso da receita comum (com hífen e minúsculas); o lote sugerido é o que vence primeiro; dispensar parcial (sem "completo") → "Dispensação registrada."; a receita relida mostra "Já dispensado"; buscar a cancelada e a vencida → sem "Dispensar", com o motivo.
6. **Contínuo** — a receita de uso contínuo retirada há 20 dias: o item mostra "próxima retirada a partir de …" (dia 25) e não oferece quantidade.
7. **Texto livre e controlado** — como **auxiliar**, a receita com texto livre e a de controle especial mostram os bloqueios; como **farmacêutica**, mapear o texto livre para o item do catálogo e dispensar; dispensar o clonazepam exige "Receita retida na farmácia"; o histórico mostra o número SNCR do documento como retido.
8. **Concorrência** — duas abas (farmacêutica e auxiliar) com o mesmo lote de saldo baixo; dispensar o saldo inteiro na primeira e na segunda → "o saldo do lote mudou…", os lotes relidos, a quantidade fica.
9. **Receita externa** — paciente pelo CPF; profissional, CRM, UF, número e data; data de amanhã recusada antes; registrar e dispensar em seguida; como **auxiliar**, um controlado é recusado antes; como **farmacêutica**, um diazepam (B1) sem número da notificação é recusado antes e, com o número, registrado; pessoa sem CPF criada como no 24a.
10. **Histórico** — "Histórico de retiradas" da paciente mostra as dispensações com lote, quem e "receita retida".
11. **Ajustes** — almoxarife da UBS: contagem com diferença pequena → "Ajuste registrado." (step-up); contagem com diferença acima de 10% → pendente; a farmacêutica aprova (step-up); o almoxarife não vê "Aprovar" no próprio; uma perda maior que o saldo recusada antes.
12. **Alertas** — a UBS vê as cinco contagens (vencendo, vencidos, abaixo do mínimo — depois de salvar um mínimo em **Posição e extrato** —, em trânsito e pendentes) e as listas; o almoxarife do central, as listas de "Todos os locais".
13. **Prontuário** — como profissional, a lista de medicamentos da paciente mostra "última retirada dd/mm/aaaa"; a linha do tempo mostra "Dispensação" com os medicamentos (sem lote) e o filtro por tipo.
14. **Sem polling e sem vazamento** — com a aba de rede aberta na dispensação e no histórico por 3 minutos: nenhuma chamada a `/pharmacy/prescriptions/lookup` ou `/pharmacy/dispensations?` sem clique; só `/pharmacy/alerts` a cada minuto quando a tela de alertas está aberta. Nenhuma URL com código da receita, CPF, nome, motivo ou número de notificação; `localStorage` e `sessionStorage` sem nada da farmácia.
15. **Telefone** (largura 375): posição, transferências, ajustes, dispensação e alertas viram cartões, sem rolagem lateral.
16. No maintenance, desligar `pharmacy` e recarregar: o grupo some e o filtro da linha do tempo não oferece "Dispensação"; as dispensações já feitas continuam na linha do tempo e na "última retirada" (registro retido); religar devolve o grupo, com tudo o que foi gravado.

Depois: `docker rm -f dashboard-mod27a`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

F-27.2..F-27.7 vão a **Done** se a parte do api já está Done (o dashboard é o último app deles); F-27.1 espera o maintenance. Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified**. Use o mesmo `gh project item-edit` da Task 0 com o id da opção Done (`98236657`) e o de Verified (da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 27a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-27a-pharmacy-stock
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod27a
/opt/homebrew/bin/git branch -d feat/mod-27a-pharmacy-stock
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (D1–D25), conferida com o plano do api (`2026-10-29-module-27a-pharmacy-stock-api.md`, Desvios 1–14 e P1–P18). **Adotada** = a forma do plano do api, já usada aqui; **aberta** = falta ou discordo; quem decide é o coordenador. A Task 0 (Step 4) confere o que o api implementou; o que vier diferente vira ajuste em `src/lib/api.ts`, nas fixtures e nas frases, antes da prova.

1. **D1 — Locais da pessoa. Adotada (P3).** `GET /pharmacy/locations` → `{ "locations": [ <location> + "roles" ] }`. Acréscimo: para o `stock_keeper` do central, os demais locais ativos vêm com `roles: []` (a tela precisa deles como destino da transferência).
2. **D2 — Administração dos locais. Adotada (P18, Task 4 do api).** `staff: [ { "user_id", "name", "role" } ]` (sem e-mail: a tela casa o e-mail pelo `listMemberships`); `DELETE …/staff` com `{ "user_id", "role" }`; recusas `role_not_granted`, `cbo_not_allowed`, `already_linked`, `unit_taken`, `central_exists`, `location_has_stock`.
3. **D3 — Busca no catálogo. Adotada (P3).** `POST /pharmacy/items/search { "q" }` → `{ "items": [<item>] }`; 503 `catalog_unavailable`.
4. **D4 — Pessoa. Adotada em parte (P3); aberta nos duplicados.** `POST /pharmacy/person_lookup { "cpf" }` cria o paciente do par validado (a receita externa leva só `patient_id` ou `new_patient`). Falta `POST /pharmacy/patients/duplicates`: o `PersonStart` do 24a, sem CPF, busca possíveis duplicados **antes** de criar; sem a rota, a receita externa de pessoa sem CPF não tem essa conferência. Proposta: a rota com a forma do 25.
5. **D5 — Mínimos. Aberta (forma do `GET`).** `PUT` com `{ "minimums": [...] }` substitui a lista (adotado, Task 4 do api); o `GET` precisa trazer `item` (nome e unidade) em cada linha.
6. **D6 — Posição por item. Aberta.** O lote sugerido precisa dos lotes de um item num local: `catalog_item_id=` na `GET /pharmacy/positions` (o api resolve `q` em ids; o filtro por id é o mesmo caminho).
7. **D7 — Formas das escritas e das listas. Adotada em parte.** Extrato como o `Pharmacy::Statement` (`document.type`); envelopes `positions`, `movements`, `transfers`, `adjustments`, `dispensations` com `next_before`. Pedido: `<transfer>` com `items[].quantity_sent`/`quantity_received`/`reason` e `<adjustment>` com `balance_before`, `counted_quantity`, `quantity` (delta com sinal), `by`, `decided_by`, `decision_reason`.
8. **D8 — Inventário de abertura. Adotada (Desvio 5).** Corpo `{ "location_id", "items" }`; 409 `opening_exists` com `path`; também pelo `pharmacist` do local.
9. **D9 — Embalagem. Adotada (Desvio 4).** `units_per_package` e `packages` inteiros 1–10000; `quantity` igual ao produto.
10. **D10 — Ajustes pendentes. Adotada (P3, P6, Desvio 7); um acréscimo aberto.** `GET /pharmacy/adjustments`; perda e vencimento nunca pendentes; `no_difference`, `lot_not_expired`, `not_pending`, `same_user`. Acréscimo: `<adjustment>.by` com `user_id`, para a tela não oferecer aprovar o próprio.
11. **D11 — "Dispensado por completo". Adotada (Desvio 9).** `complete: true` opcional no item; depois, `item_completed`.
12. **D12 — Itens da dispensação com nome. Aberta.** `<dispensation>.items[]` com `item` (`<item>`) e `lot` (`{ id, lot_number, expires_on }`), para o histórico não precisar de outra leitura.
13. **D13 — Quantidade. Adotada (P13).** Até 5 dígitos inteiros e 2 casas, string com ponto; saída com duas casas (`"30.00"`); a tela mostra sem zeros à direita.
14. **D14 — Retenção. Adotada (P8).** A tela manda `retention: {}`; o número retido é o `sncr.number` da receita digital ou o `notification_number` da externa. Fica com o coordenador a decisão pedida no P8 (antimicrobiano).
15. **D15 — Alertas. Adotada (Task 8 do api).** `GET /pharmacy/alerts?location_id=` (obrigatório) → cinco **contagens**; as listas da tela vêm da posição (`expiring=true`, `below_minimum=true`), que aceita "todos os locais" para o almoxarife do central.
16. **D16 — Profissional da receita externa. Adotada (P9, Task 10 do api).** `council` em `CRM` | `CRO` | `COREN`; UF; 1–10 dígitos; `issued_on` não futura; posologia 3–500; `notification_required` com item de lista A ou B.
17. **D17 — Quem ajusta e aprova. Adotada (Desvio 2).** `stock_keeper` ou `pharmacist` do local criam e aprovam (outra pessoa).
18. **D18 — Recebimento. Adotada (Desvio 6).** Todos os lotes, uma vez cada (`invalid_items`); de 0 ao enviado; `reason` só quando difere (`reason_required`); 409 `already_received`.
19. **D19 — Papéis privilegiados. Adotada (P17).** `pharmacist` e `pharmacy_assistant`; `stock_keeper` não.
20. **D20 — Todas as recusas da dispensação de uma vez. Aberta.** O plano do api devolve a **primeira** recusa com `path`. Lição do 21b (B26): quando mais de um item falha, o 422 traz o primeiro código em `error` e todos em `problems: [ { "path", "error", "next_allowed_on"? } ]`. A tela já confere tudo antes de enviar; sem `problems`, mostra a primeira.
21. **D21 — Receita não encontrada. Adotada (P12).** 404 `prescription_not_found`; código fora do formato → `invalid_request` com `path: "short_code"`; 429 `too_many_requests`.
22. **D22 — Linha do tempo. Adotada (P10).** `kind: "dispensation"`, `summary` com os nomes dos medicamentos (sem lote nem quantidade), `detail_ref: null`, visível com o interruptor desligado. Pedido: `status: "dispensed"` (a tela traduz) e `kinds[]=dispensation` aceito no filtro.
23. **D23 — Última retirada. Adotada (Task 13 do api).** `last_dispensed_on` por item da lista de medicamentos, só leitura, também com o interruptor desligado.
24. **D24 — Configurações. Adotada (Valores fixados 2).** As faixas de `SETTING_RANGES`; 422 `invalid_request` com `path` da chave.
25. **D25 — Devolução sem tela. Aberta.** O plano do api cria `POST /pharmacy/returns` (P3), mas `<dispensation>.items[]` não traz o id do item dispensado que a rota pede. Proposta: o id em cada item (`id`) e a tela "Registrar devolução" no histórico de retiradas numa entrega seguinte (a brief do 27a não a pede); até lá a devolução fica sem tela.

## Self-review

- **Cobertura (spec §1–§9; contrato §1–§9, §11, §12):** interruptor, papéis e locais — Tasks 1, 3 e 4 (F-27.1; D1, D2, D19); entradas, lotes, validade, embalagem e abertura — Task 5 (F-27.2; risco "unidade de compra"; D3, D8, D9); transferência em dois passos com diferença — Task 7 (F-27.3; Review Focus 4; D18); inventário, ajustes, perdas, vencimento, step-up e segundo responsável — Task 8 (F-27.4; Review Focus 5; D10, D17); dispensação pela receita (código ou paciente), parcial, completo, lote sugerido, contínuo (25º dia), receita cancelada e vencida — Tasks 9 e 10 (F-27.5; Review Focus 1 e 2; D6, D11, D20, D21); receita externa, controlado só pelo farmacêutico, retenção, texto livre mapeado — Tasks 10 e 11 (F-27.6; D4, D14, D16); alertas, posição, extrato e mínimos — Tasks 6 e 13 (F-27.7; D5, D15); linha do tempo e lista de medicamentos — Task 14 (decisão 11; D22, D23); quantidade decimal sem cortar — Task 2 (Review Focus 3; D13); nada sensível na URL e no navegador, nada que grava trilha relê — Tasks 1, 9, 12 e 15; prova no navegador com roteiro — Task 15. Devolução (movimento `return` da decisão 6): sem tela nesta entrega (D25). Saldo nunca negativo sob concorrência, imutabilidade, travas, retenção LGPD e trilha são do api; a tela mostra o que ele devolve e reage às recusas.
- **Placeholders:** valores lidos na execução são as rotas, formas e limites do api (Task 0, com a regra: o api vence), os ids dos cards (Task 0) e os ícones, se algum faltar (Task 0, com a regra). Nenhum passo diz "trate os casos".
- **Tipos e nomes:** `StockItem`, `StockLot`, `LotBalance`, `StockPosition`, `PrescriptionView`, `PrescriptionItemView`, `Dispensation`, `StockTransfer`, `StockAdjustment`, `PharmacyLocation` (Task 1) são os das Tasks 2–14; `listPositions(q)` recebe `{ locationId?, q?, catalogItemId?, expiring?, belowMinimum?, before? }` em todas; `availableLots`, `suggestedLotId`, `lotOptionLabel` (Task 2) são os das Tasks 7 e 10; `dispenseProblems`/`dispenseBody` (Task 2) são os do `DispenseForm` (Task 10), usado pelas Tasks 9 e 11; `ItemSearch` (Task 5) é o das Tasks 6, 10 e 11; `PHARMACY_PERSON_SOURCE` (Task 9) é o da Task 11; `LocationPick` e `PharmacyGate` (Task 3) em todas as telas; `timelineKinds(exam, procedure, pharmacy)` (Task 14) igual no `PatientTimeline`.
- **Review Focus:** 1 — Task 10 ("insufficient_balance: relê os lotes e mantém o que foi digitado; too_early com a data"); 2 — Task 10 ("receita cancelada enquanto dispensava: frase e nova busca"); 3 — Task 2 ("quantidade: vírgula, até 2 casas, maior que zero; recusa em vez de cortar"); 4 — Task 7 ("unidade recebe: conferência lote a lote; diferença pede motivo; frase da diferença"); 5 — Task 8 ("pendentes: aprovar e rejeitar com motivo; o próprio não oferece; 409 relê").
- **Proporção:** os blocos de código são testes (nomes, rótulos, frases e valores que o implementador não decide sozinho), as fixtures e a lista de tipos do contrato; os corpos das telas ficam com a assinatura, o comportamento e as frases.
