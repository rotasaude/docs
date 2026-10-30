# Módulo 28 — Saúde bucal (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 28** (`feat/mod-28-oral-health`, worktree `apps/api/.claude/mod28`, porta de dev **3050**) com as rotas do contrato §3–§10 (e as Divergências D1–D24 deste plano que o plano do api adotar), o interruptor `dental_care` na sessão, a unidade `ceo`, os CBOs de saúde bucal em `consultation_options` e a semente do 28. Ele é **mergeado antes** deste (contrato §15: api → dashboard → maintenance → catálogo). As Tasks 0–12 não dependem dele (dublês de `src/lib/api` com fixtures nas formas do contrato §2); a prova da Task 13 depende.

**Goal:** Lado dashboard do 28: a **consulta odontológica** dentro da tela da consulta do 19a (escolha do tipo; **registro inicial**; campos da FAO com as regras condicionais e as recusas `problems[]` todas de uma vez), o **odontograma** (dentição permanente, decídua e mista; faces; local por dente, face, sextante, arcada ou boca; achados × procedimentos; teclado e tablet), o **plano de tratamento** ao lado com a sugestão de "tratamento concluído", o **histórico do dente**, o **registro periodontal simplificado**, o **periograma**, a **endodontia**, a **prótese** (pedido, etapas e instalação na consulta), os **documentos e a regulação** do dentista (o auxiliar sem eles), o **adendo** odontológico, a leitura da consulta odontológica finalizada, o **cadastro do CEO** e da **lista do TSB/ASB**, os painéis de **metas do CEO** e **B1/B2**, e o interruptor `dental_care` escondendo tudo.

**Architecture:** O cliente HTTP ganha a seção "Saúde bucal (módulo 28)" no fim de `src/lib/api.ts` (tipos = formas do contrato §2–§10; `Consultation`, `ConsultationOptions`, `ConsultationDraftInput`, `AddendumChanges` e `startConsultation` ganham o que é odontológico). As regras de tela ficam puras em `src/lib/dental.ts` (dentes FDI, linhas do odontograma, faces por quadrante, locais, rótulos dos códigos do LEDI, espelho das recusas da finalização, limites de quantidade, sugestão de tratamento concluído, frases das recusas). As peças ficam em `src/modules/dental/`; a consulta odontológica **não é tela nova**: o `ConsultationWorkspace` e o `ConsultationEditor` do 19a ganham o ramo `kind: "dental"` e montam o `DentalEditor`. As telas próprias (próteses, metas do CEO, B1/B2, administração) ficam no grupo novo "Saúde bucal" (`modules.ts`, `catalog.ts`, `App.tsx`). O dashboard nunca decide o que o api garante: imutabilidade dos eventos, cálculo do odontograma, agregação da FAO, regras de B1/B2 e das metas, trilha e retenção LGPD são do api; a tela só não envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-30-module-28-oral-health-design.md` (§1 intenção, sucesso e antiobjetivos, §2 decisões 3–17 e as do coordenador, §4 telas, §5 segurança e LGPD, §6 testes, §8 F-IDs, §9 riscos), `docs/adr/0046.md`, a pesquisa `docs/pesquisa/2026-10-30-odontologia-28.md` (§1 campos da FAO, §2 CBOs, §4 metas, §5 B1/B2) e o contrato `docs/superpowers/plans/2026-10-30-module-28-oral-health-contracts.md` (§1–§11, §14, §15), com as convenções herdadas (21a §15, 21b §14, 27a §13). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D24); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptor `dental_care` (requer `clinical_record`); `Consultation.kind` `general` | `dental`; `baseline` (bool, só `dental`); papel na consulta `dentist` (CBO `2232*`) | `auxiliary` (TSB `322405`, `322425`; ASB `322415`, `322430`); unidade `ceo`; `ceo_type` `I` | `II` | `III`; especialidades `endodontics`, `periodontics`, `oral_surgery`, `stomatology`, `special_needs`, `prosthesis`; dentes FDI permanentes `11–18, 21–28, 31–38, 41–48` e decíduos `51–55, 61–65, 71–75, 81–85`; faces `M` | `D` | `O` | `V` | `L`; local `tooth` | `face` | `sextant` (`S1`–`S6`) | `arch` (`upper`/`lower`) | `mouth`; plano `open` | `completed` | `cancelled`, item `planned` | `done` | `cancelled`; prótese `impression` | `sent` | `received` | `installed` | `cancelled`; campos da FAO com os códigos do LEDI 8.7.0 sem tradução (`care_type` 2/4/5/6, `dental_consultation_type` 1/2/4, `oral_health_surveillance` 1–6 e 99, `supplies` 1–3, `dental_outcomes` 1–12 e 14–18). Código desconhecido aparece cru, nunca some.
- **Interruptor** (spec decisão 17): sem `dental_care` (ou sem `clinical_record`) na sessão, o dashboard é o de hoje — o grupo "Saúde bucal" some, "Iniciar consulta" não oferece o tipo odontológico, a unidade `ceo` não aparece no cadastro de unidades. A consulta odontológica **já finalizada** continua legível na consulta e na linha do tempo (registro clínico retido, ADR 0026). 403 `feature_disabled` → `DENTAL_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo ("Saúde bucal") em `features.ts`.
- **Quem faz o quê** (spec decisão 8 e decisões do coordenador; contrato §1, §4, §8): o **dentista** registra tudo; o **auxiliar** (TSB/ASB) registra a consulta odontológica só com procedimentos da lista da cidade (`auxiliary_allowed`), marca achados e pode dar baixa num item do plano com SIGTAP da lista — **não** cria nem cancela item, **não** encaminha (desfechos 1–11), **não** emite documento, **não** pede regulação, **não** faz registro inicial, **não** pede prótese nem preenche especialidades. A tela não oferece; o api recusa de novo. O papel vem do CBO da consulta (`dentalRole(cbo_code)`) e, antes de iniciar, de `consultation_options.dental_role` (D3).
- **Leitura nominal com trilha, sem polling** (spec §5; contrato §5–§7): odontograma, eventos do dente, planos, endodontia e próteses leem com `NOMINAL_QUERY` (de `src/lib/careLines.ts`): ao abrir, ao filtrar, no clique ou depois de uma ação — nunca sozinhos. Painéis de metas e B1/B2 são contagens sem trilha: `STATIC_QUERY` (ao abrir e ao filtrar). Nenhum `refetchInterval` no módulo.
- **Dado clínico só no corpo** (spec §5; brief): dente, face, situação, procedimento do paciente, periograma, motivo do cancelamento e justificativa nunca na URL, em mensagem de erro, em `console` ou em Analytics; nada em `localStorage`, `sessionStorage` ou IndexedDB. Na URL vão só ids (`patient_id`, `health_unit_id`, `team_id`), `status`, `stage`, `month`, `period`, `before`, `limit` e — exceção do contrato §5, D9 — `tooth` no filtro do histórico de eventos, que a Divergência D9 propõe levar ao corpo.
- **Todas as recusas de uma vez** (21b §14 B26; contrato, convenções): a tela confere a finalização inteira antes de enviar (`dentalFinalizeProblems`) e mostra cada problema numa lista; a recusa do api com `problems: [ { "path", "error" } ]` (27a §13 H8) mostra todas, traduzidas; sem `problems`, a do topo.
- **Recusar em vez de cortar** (21a §15): quantidade acima do limite do local (dente/face ≤ 32, sextante ≤ 6, arcada ≤ 2, boca ≤ 1 por SIGTAP na consulta), periograma fora da faixa, motivo curto — recusa com a frase; nada é arredondado, cortado nem descartado em silêncio.
- **Odontograma acessível e de tablet** (brief): cada dente é um `button` com rótulo completo ("Dente 16: cariado; face oclusal: cariada"); uma parada de Tab para o odontograma inteiro (tabindex móvel), setas movem entre dentes na ordem visual, `Home`/`End` vão ao começo e fim da linha, `Enter`/`Espaço` selecionam, `Escape` limpa a seleção; a face se escolhe no painel do dente (grupo de botões com rótulo) — o toque direto na face do desenho é atalho de ponteiro; alvo de toque ≥ 44 px; a situação nunca é só cor (sigla visível). Sem foco escondido (`outline: "none"` proibido pela guarda).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo).
- Textos (D18, D19; se o api fixar outros, o api vence): motivo de cancelamento de item do plano e de pedido de prótese 10–500; nome do laboratório terceiro 3–120; profundidade de sondagem 0–20 mm e recessão −10 a 20 mm (inteiros); mobilidade e furca 0–3; canais 1–5; quantidade inteira ≥ 1.
- "Hoje" é o dia da cidade (`todayInCity()`); telas recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY28 = "2026-10-30"` (sexta), sem relógio falso.
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--info`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `SensitiveAction`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts`); **nenhum componente de `src/components` muda**. Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-28.1..F-28.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-28.1: api, dashboard e maintenance; F-28.2..F-28.8: api e dashboard) e a **Verified** depois da prova no navegador (Task 13).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod28` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-28-oral-health` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod28`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test dashboard -w dashboard-mod28 <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx`; a sessão entra por `fetchCurrentSession` mockado (como nos testes do `ConsultationWorkspace`); `consultation`, `finalized`, `options`, `record`, `problem` vêm de `src/test/consultationFixtures.ts`.
- A prova (Task 13) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5206**, com proxy para o api do 28 na **3050**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Auxiliar (TSB/ASB) abre a consulta odontológica.** Espera-se: nada de "Registro inicial", aba Documentos, "Pedir regulação", desfechos 1–11, criar/cancelar item do plano, especialidades nem prótese; a busca de procedimento só lista `auxiliary_allowed`; se o api ainda recusar (`auxiliary_cannot_refer`, `auxiliary_cannot_plan`, `auxiliary_procedure_not_allowed`, `auxiliary_cannot_issue`), a frase aparece e nada some do rascunho. Teste: Task 7, "auxiliar: sem documentos, regulação, registro inicial, encaminhamento nem plano".
2. **Mesmo SIGTAP em 7 sextantes, 3 arcadas ou 2 vezes na boca toda.** Espera-se: recusa antes de enviar com a frase do local e o limite ("raspagem supragengival: no máximo 6 por sextante nesta consulta"), sem cortar nem juntar linhas; dente e face somam no mesmo limite de 32. Teste: Task 2, "limites por local: recusa sem cortar; dente e face somam juntos".
3. **Item do plano feito em outra consulta enquanto este rascunho o baixava.** Espera-se: a finalização volta 422 com `plan_item_not_open` (path do procedimento); a tela relê o plano, desfaz só o vínculo daquele procedimento (o procedimento fica), mostra a frase junto das demais recusas e não reenvia sozinha. Teste: Task 7, "plan_item_not_open: relê o plano, desfaz o vínculo e mantém o procedimento".
4. **Paciente do sexo masculino com inscrição antiga de pré-natal, ou paciente feminina com `pregnant` vazio.** Espera-se: para masculino o campo "Gestante" não aparece e vai `null` (mesmo que o rascunho tenha vindo `true`: a tela avisa e manda `null`); para feminino, "Gestante" é sim/não obrigatório e o `true` pré-preenchido vem marcado com "pré-preenchido pela linha de pré-natal — confirme". Teste: Task 2, "gestante: obrigatória no feminino, nula no masculino" e Task 6, "gestante pré-preenchida pede confirmação; masculino sem o campo".
5. **Dentista que só usa teclado (ou tablet) marca a face oclusal do 36 e registra uma restauração.** Espera-se: uma parada de Tab entra no odontograma, setas chegam ao 36 atravessando quadrantes na ordem visual, `Enter` seleciona, o painel do dente oferece "Face oclusal" como botão com rótulo, o procedimento entra com o local "dente 36, face oclusal"; nenhum alvo menor que 44 px. Teste: Task 4, "teclado: tabindex móvel, setas atravessam quadrantes, Enter seleciona, Escape limpa".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.dental.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/dentalFixtures.ts` | tipos e cliente HTTP do 28; interruptor; proxy; dados de teste | 1 |
| `src/lib/dental.ts`, `src/lib/dental.test.ts` | regras de tela: portão, papel, dentes, locais, rótulos, limites, espelho das recusas, sugestão, frases | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.dental.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/attendance.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/attendance/UnitForm.dental.test.tsx`, `src/modules/dental/DentalGate.tsx`, telas-portão `src/modules/Dental{Prosthesis,CeoGoals,Indicators,Admin}.tsx` | grupo "Saúde bucal", portão, unidade CEO no cadastro | 3 |
| `src/modules/dental/Odontogram.tsx`, `src/modules/dental/Odontogram.test.tsx` | odontograma (dentições, faces, rótulos, teclado, toque) | 4 |
| `src/modules/dental/TreatmentPlanPanel.tsx`, `src/modules/dental/TreatmentPlanPanel.test.tsx` | plano da unidade, mudanças do rascunho, sugestão de tratamento concluído | 5 |
| `src/modules/dental/ToothActions.tsx`, `src/modules/dental/ProcedureSearch.tsx`, `src/modules/dental/FaoFields.tsx`, `src/modules/dental/DentalEditor.tsx`, `src/modules/dental/DentalEditor.test.tsx` | achados, procedimentos, local, campos da FAO, montagem do bloco | 6 |
| `src/lib/consultation.ts`, `src/lib/consultation.test.ts`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/consultation/ConsultationWorkspace.dental.test.tsx` | escolha do tipo, registro inicial, autosave e finalização com `problems[]`, documentos e regulação do dentista | 7 |
| `src/modules/dental/DentalSummary.tsx`, `src/modules/dental/DentalAddendumFields.tsx`, `src/modules/dental/ToothHistory.tsx`, `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/AddendumForm.tsx`, `src/modules/dental/DentalReading.test.tsx` | consulta finalizada, adendo odontológico, histórico do dente | 8 |
| `src/modules/dental/PeriodontalScreeningFields.tsx`, `src/modules/dental/PeriodontalChartFields.tsx`, `src/modules/dental/EndodonticsFields.tsx`, `src/modules/dental/Specialties.test.tsx`, `src/modules/dental/DentalEditor.tsx` | registro periodontal simplificado, periograma, endodontia | 9 |
| `src/modules/dental/ProsthesisPanel.tsx`, `src/modules/DentalProsthesis.tsx`, `src/modules/dental/Prosthesis.test.tsx`, `src/modules/dental/DentalEditor.tsx` | pedido, etapas, instalação na consulta | 10 |
| `src/modules/DentalAdmin.tsx`, `src/modules/DentalAdmin.test.tsx` | perfil do CEO e lista do TSB/ASB | 11 |
| `src/modules/DentalCeoGoals.tsx`, `src/modules/DentalIndicators.tsx`, `src/modules/DentalPanels.test.tsx` | metas do CEO e B1/B2 | 12 |
| — | suíte, build, revisão, prova no navegador | 13 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (dente, face, situação, motivo, procedimento) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa —: as 17 recusas da finalização, limites por local, gestante, sugestão de tratamento concluído, dentes e faces por quadrante (Task 2); menu e portão por papel e interruptor (Task 3); cada peça pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api, sessão por papel e fixtures do contrato §2, cobrindo o caminho feliz, o teclado, cada recusa que muda a tela (`cbo_not_allowed`, `baseline_requires_dentist`, `feature_disabled`, `plan_item_not_open`, `auxiliary_*`, `quantity_exceeds_location_limit`, `baseline_only_findings`, `regulation_request_required`), o step-up, a ausência de releitura e de armazenamento local (Tasks 4–12). A prova final é no navegador, contra o api real na 3050 (Task 13).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`fadfa87` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod28`.
- Produces: cards F-28.1..F-28.8 em In Progress; worktree `$W` na branch `feat/mod-28-oral-health`; a base anotada; as rotas e as formas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-28\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-28.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "pharmacy";\|  pharmacy: "Farmácia e almoxarifado"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export interface Consultation {\|^export interface ConsultationOptions \|^export interface ConsultationDraftInput {\|^export interface AddendumChanges \|^export function startConsultation\|^const ATTENDANCE_BASE\|^const withQuery = \|^const postProfessional'
$G -C $R/apps/dashboard show origin/main:src/lib/consultation.ts | grep -n '^export interface ConsultationDraft {\|^export function draftFrom\|^export function checkDraft\|^export function finalizeProblems'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationEditor.tsx | grep -n 'Tipo de atendimento\|<ConductsField\|props.onFinalized(await finalizeConsultation'
$G -C $R/apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n 'setConsultation(await startConsultation(props.attendanceId))\|const proceduresOn = \|const regulationOn = \|const showTabs = '
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "pharmacy-alerts" | "pharmacy-admin";\|      if (item.id === "pharmacy-admin") return phOn && isAdmin;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 15\|NAV_GROUPS.length - 14'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n 'IconAlertTriangle, IconBuildingCog\|    "pharmacy-admin": { icon: IconBuildingCog'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "pharmacy-admin": return <PharmacyAdmin'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"^/pharmacy(/|$)": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/attendance.ts | grep -n '^export const UNIT_KINDS'
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^export const NOMINAL_QUERY\|^export const STATIC_QUERY'
$G -C $R/apps/dashboard show origin/main:src/lib/clinicalDocuments.ts | grep -n 'if (code.startsWith("2232")) return "dentist";'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod28 -b feat/mod-28-oral-health origin/main
cd $R/.worktrees/dashboard-mod28 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod28
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Dental|DentalBroken|TargetArrow|ChartBar|Settings)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os cinco ícones (falta algum → escolha outro que exista e use o mesmo na Task 3; o teste cobra só a descrição). **Anote** a base (arquivos e testes). A Task 13 espera a base mais **13 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 28** (informativo para as Tasks 1–12)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-28-oral-health
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod28 api bin/rails routes 2>/dev/null | grep -E "dental|consultation_options|attendances/:id/consultation|endodontic|prosthesis" | head -60
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-30-module-28-oral-health-api.md 2>/dev/null | grep -n "^## \|^### Task\|Divergência\|Desvio" | head -60
```

Expected: anote os prefixos reais (este plano supõe `/attendance/…` para consulta, paciente e busca de procedimento e `/dental/…` para o resto — D1), as formas de `problems`, e quais Divergências o plano do api adotou. O que vier diferente vira ajuste em `src/lib/api.ts`, nas fixtures e nas frases (o api vence); se o api ainda não existir, siga com os dublês.

---

### Task 1: Tipos, cliente HTTP, interruptor, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; `Consultation`, `ConsultationOptions`, `ConsultationDraftInput`, `AddendumChanges` e `startConsultation` ganham campos), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/dentalFixtures.ts`
- Test: `src/lib/api.dental.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `postProfessional`, `ATTENDANCE_BASE` (existentes em `api.ts`); `consultation`, `options` (`src/test/consultationFixtures.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`):
  - tipos `ConsultationKind = "general" | "dental"`, `DentalRole = "dentist" | "auxiliary"`, `ToothFace = "M" | "D" | "O" | "V" | "L"`, `DentalLocationKind = "tooth" | "face" | "sextant" | "arch" | "mouth"`, `Sextant = "S1" | "S2" | "S3" | "S4" | "S5" | "S6"`, `Arch = "upper" | "lower"`, `Dentition = "permanent" | "deciduous" | "mixed"`, `DentalPlanStatus`, `DentalPlanItemStatus`, `ProsthesisStage`, `CeoType = "I" | "II" | "III"`, `CeoSpecialty` (as 6 do §1), `PerioSite = "MV" | "V" | "DV" | "ML" | "L" | "DL"`;
  - `DentalLocation { kind: DentalLocationKind; tooth: string | null; face: ToothFace | null; sextant: Sextant | null; arch: Arch | null }`; `DentalUnitRef { id: string; name: string; kind: string }`;
  - `OdontogramTooth { tooth: string; situation: string | null; faces: Partial<Record<ToothFace, string | null>>; last_event_at: string | null; notes_count: number }`; `Odontogram { patient: { id: string; display_name: string | null; has_cpf: boolean; cns_masked: string | null }; dentition: Dentition; teeth: OdontogramTooth[]; open_plans: { id: string; health_unit: DentalUnitRef }[] }`;
  - `ChartEvent { id; tooth; face: ToothFace | null; situation: string | null; procedure: { sigtap: string; name: string } | null; consultation_id: string; addendum_id: string | null; by: { name: string; cbo: string }; at: string }`; `ChartEventPage { events: ChartEvent[]; next_before: string | null }`;
  - `DentalPlanItem { id; sigtap; name; location: DentalLocation; status: DentalPlanItemStatus | string; done_in_consultation_id: string | null; cancel_reason: string | null }`; `DentalPlan { id; health_unit: DentalUnitRef; status: DentalPlanStatus | string; opened_in_consultation_id: string; closed_in_consultation_id: string | null; items: DentalPlanItem[] }`;
  - `DentalProcedureLine { id?: string; sigtap: string; name?: string; location: DentalLocation; quantity: number; plan_item_id: string | null; prosthesis_order_id?: string | null }` (`name` só na leitura e no estado da tela; `prosthesis_order_id` — D6); `DentalFinding { tooth: string; face: ToothFace | null; situation: string }`; `PlanAdd { sigtap: string; name?: string; location: DentalLocation }`; `PlanCancel { item_id: string; reason: string }`; `DentalPlanChanges { add: PlanAdd[]; cancel: PlanCancel[] }`;
  - `PeriodontalScreening { sextants: { sextant: Sextant; code: number | null; star: boolean }[] }` (`code: null` = sextante excluído "X", D17); `PeriodontalSiteValue { site: PerioSite; probing_depth_mm: number | null; recession_mm: number | null; bleeding: boolean }`; `PeriodontalTooth { tooth: string; mobility: number | null; furcation: number | null; sites: PeriodontalSiteValue[] }`; `PeriodontalChart { teeth: PeriodontalTooth[] }`; `EndodonticEntry { tooth: string; canals: number; status: "in_progress" | "completed" }`;
  - `DentalFields { care_type: string | null; dental_consultation_type: string | null; oral_health_surveillance: string[]; supplies: string[]; dental_outcomes: string[]; pregnant: boolean | null; special_needs: boolean; procedures: DentalProcedureLine[]; findings: DentalFinding[]; plan_changes: DentalPlanChanges; periodontal_screening: PeriodontalScreening | null; periodontal_chart: PeriodontalChart | null; endodontics: EndodonticEntry[] }` (= `<dental_fields>`); `DentalBlock extends DentalFields { suggest_completed?: boolean }`;
  - `Consultation` ganha `kind?: ConsultationKind` (ausente = `general`), `baseline?: boolean`, `dental?: DentalBlock | null`; `ConsultationOptions` ganha `kinds?: ConsultationKind[]` (ausente = `["general"]`), `dental_role?: DentalRole | null` (D3), `dental_situations?: DentalSituation[]` (D4); `ConsultationDraftInput` ganha `dental?: DentalFields`; `AddendumChanges` ganha `dental?: { findings: DentalFinding[]; plan_changes: { cancel: PlanCancel[] } }` (D13);
  - `DentalSituation { code: string; label: string; applies_to: "tooth" | "face" | "both" }`; `DentalProcedureRef { sigtap: string; name: string; locations: DentalLocationKind[]; max_quantity: number; auxiliary_allowed: boolean; prosthesis: boolean }` (D5);
  - `EndodonticTreatment { tooth: string; canals: number; status: string; sessions: { consultation_id: string; at: string; by: { name: string } | null }[] }` (D16);
  - `ProsthesisOrder { id; patient: { id: string; display_name: string | null }; health_unit: { id: string; name: string }; sigtap: string; name: string; laboratory: { care_provider_id: string | null; name: string }; stage: ProsthesisStage | string; stages: { stage: string; at: string; by: { name: string } | null; reason: string | null }[]; installed_in_consultation_id: string | null; created_at: string }` (D14); `ProsthesisOrderInput { patient_id: string; consultation_id: string; sigtap: string; laboratory: { care_provider_id: string } | { name: string } }`; `ProsthesisQuery { healthUnitId?: string | null; patientId?: string | null; stage?: string | null; before?: string | null }` (D15); `ProsthesisPage { orders: ProsthesisOrder[]; next_before: string | null }`; `DentalLaboratory { id: string; name: string }` (D14);
  - `CeoProfile { health_unit: { id: string; name: string }; ceo_type: CeoType | null; specialties: CeoSpecialty[] }` (lista, D20); `AuxiliaryProcedure { sigtap: string; name: string }` (D21); `CeoGoalRow { key: string; goal: number; done: number; composition: { rule: string; ok: boolean; value: number } | null; met: boolean }`; `CeoGoals { ceo_type: CeoType | string; specialties: CeoGoalRow[]; misses_last_12: Record<string, { consecutive: number; total: number }>; suspension_risk: boolean; history?: { month: string; specialties: { key: string; done: number; met: boolean }[] }[] }` (`history` — D22); `DentalTeam { id: string; name: string; ine: string | null; health_unit: { id: string; name: string } }` (D23); `DentalIndicators { b1: { numerator: number; denominator: number }; b2: { numerator: number; denominator: number } }`;
  - constante `DENTAL_PAGE_SIZE = 50`;
  - funções: `startConsultation(attendanceId: string, input?: { kind: ConsultationKind; baseline?: boolean }): Promise<Consultation>` (corpo `input ?? {}` — chamada sem argumento continua mandando `{}`), `getDentalChart(patientId: string): Promise<Odontogram>`, `listChartEvents(patientId: string, q: { tooth?: string | null; before?: string | null }): Promise<ChartEventPage>`, `listDentalPlans(patientId: string, q: { healthUnitId?: string | null; status?: string | null }): Promise<DentalPlan[]>`, `searchDentalProcedures(q: string): Promise<DentalProcedureRef[]>`, `listEndodonticTreatments(patientId: string): Promise<EndodonticTreatment[]>`, `createProsthesisOrder(input: ProsthesisOrderInput): Promise<ProsthesisOrder>`, `changeProsthesisStage(id: string, input: { stage: "sent" | "received" | "cancelled"; reason?: string }): Promise<ProsthesisOrder>`, `listProsthesisOrders(q: ProsthesisQuery): Promise<ProsthesisPage>`, `listDentalLaboratories(): Promise<DentalLaboratory[]>`, `listCeoProfiles(): Promise<CeoProfile[]>`, `putCeoProfile(unitId: string, input: { ceo_type: CeoType; specialties: CeoSpecialty[] }): Promise<CeoProfile>`, `getAuxiliaryProcedures(): Promise<AuxiliaryProcedure[]>`, `putAuxiliaryProcedures(sigtaps: string[]): Promise<AuxiliaryProcedure[]>`, `getCeoGoals(unitId: string, month: string): Promise<CeoGoals>`, `listDentalTeams(): Promise<DentalTeam[]>`, `getDentalIndicators(teamId: string, period: string): Promise<DentalIndicators>`;
  - `FeatureKey` ganha `dental_care`.
- Produces (em `src/test/dentalFixtures.ts`): `TODAY28`, `UBS`, `CEO_UNIT`, `SITUATIONS`, `RESIN`, `SCALING`, `FLUOR`, `DENTURE`, `dentalUser(roles?, over?)`, `dentalOptions(over?)`, `auxOptions(over?)`, `fields(over?)`, `dentalDraft(over?)`, `dentalFinalized(over?)`, `toothLoc(tooth, face?)`, `odontogram(over?)`, `chartEvent(over?)`, `planItem(over?)`, `plan(over?)`, `prosthesisOrder(over?)`, `ceoProfile(over?)`, `ceoGoals(over?)`, `dentalTeam(over?)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.dental.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  changeProsthesisStage, createProsthesisOrder, getAuxiliaryProcedures, getCeoGoals, getDentalChart, getDentalIndicators, listCeoProfiles,
  listChartEvents, listDentalLaboratories, listDentalPlans, listDentalTeams, listEndodonticTreatments, listProsthesisOrders, putAuxiliaryProcedures,
  putCeoProfile, saveConsultationDraft, searchDentalProcedures, startConsultation
} from "./api";
import { fields, toothLoc } from "../test/dentalFixtures";

function ok(body: unknown): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
}
// Dente, face, situação, procedimento e motivo nunca na URL (exceto o filtro `tooth` do histórico — D9).
const SENSITIVE = /caries|restored|RESTAURA|quebrou|desistiu|oclusal|0307010031/i;

describe("cliente da saúde bucal (contrato §3–§10)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(() => ok({})); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => (fetchMock.mock.calls[i][1] ?? {}) as RequestInit;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));
  const writeOf = (i = 0) => [ urlOf(i).pathname, initOf(i).method, (initOf(i).headers as Record<string, string>)["Content-Type"] ];

  it("iniciar: sem argumento manda {}; odontológica e registro inicial no corpo", async () => {
    await startConsultation("a1");
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/attendance/attendances/a1/consultation", "POST", "application/json", {} ]);
    await startConsultation("a1", { kind: "dental" });
    expect(bodyOf(1)).toEqual({ kind: "dental" });
    await startConsultation("a1", { kind: "dental", baseline: true });
    expect(bodyOf(2)).toEqual({ kind: "dental", baseline: true });
  });

  it("rascunho leva o bloco dental no corpo do PATCH", async () => {
    const dental = fields({ procedures: [ { sigtap: "0307010031", location: toothLoc("16", "O"), quantity: 1, plan_item_id: "pi1" } ] });
    await saveConsultationDraft("cs1", { subjective: "", objective: "", assessment: "", plan: "", vitals: {}, care_type: null,
      evaluated_problems: [], conducts: [], exam_requests: [], dental });
    expect([ ...writeOf(0), bodyOf(0).dental ]).toEqual([ "/attendance/consultations/cs1", "PATCH", "application/json", dental ]);
  });

  it("odontograma, eventos, planos e endodontia: só ids e filtros na URL", async () => {
    await getDentalChart("pa1");
    expect(urlOf(0).pathname).toBe("/attendance/patients/pa1/dental_chart");
    fetchMock.mockImplementationOnce(() => ok({ events: [ { id: "ev1" } ], next_before: "c1" }));
    expect(await listChartEvents("pa1", { tooth: "16", before: "c0" })).toEqual({ events: [ { id: "ev1" } ], next_before: "c1" });
    expect([ urlOf(1).pathname, Object.fromEntries(urlOf(1).searchParams) ])
      .toEqual([ "/attendance/patients/pa1/dental_chart/events", { tooth: "16", before: "c0", limit: "50" } ]);
    expect(await listChartEvents("pa1", {})).toEqual({ events: [], next_before: null });
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ limit: "50" });
    fetchMock.mockImplementationOnce(() => ok({ plans: [ { id: "dp1" } ] }));
    expect(await listDentalPlans("pa1", { healthUnitId: "u1", status: "open" })).toEqual([ { id: "dp1" } ]);
    expect([ urlOf(3).pathname, Object.fromEntries(urlOf(3).searchParams) ])
      .toEqual([ "/attendance/patients/pa1/dental_plans", { health_unit_id: "u1", status: "open" } ]);
    expect(await listEndodonticTreatments("pa1")).toEqual([]);
    expect(urlOf(4).pathname).toBe("/attendance/patients/pa1/endodontic_treatments");
  });

  it("busca de procedimento no corpo (D5)", async () => {
    fetchMock.mockImplementationOnce(() => ok({ items: [ { sigtap: "0307010031" } ] }));
    expect(await searchDentalProcedures("resina")).toEqual([ { sigtap: "0307010031" } ]);
    expect([ ...writeOf(0), urlOf(0).search, bodyOf(0) ]).toEqual([ "/dental/procedures/search", "POST", "application/json", "", { q: "resina" } ]);
  });

  it("prótese: pedido, etapa com motivo no corpo, lista e laboratórios", async () => {
    const input = { patient_id: "pa1", consultation_id: "cs1", sigtap: "0701070137", laboratory: { name: "Lab Sorriso" } };
    await createProsthesisOrder(input);
    expect([ ...writeOf(0), bodyOf(0) ]).toEqual([ "/dental/prosthesis_orders", "POST", "application/json", input ]);
    await changeProsthesisStage("po1", { stage: "cancelled", reason: "paciente desistiu do tratamento" });
    expect([ ...writeOf(1), bodyOf(1) ]).toEqual([ "/dental/prosthesis_orders/po1/stages", "POST", "application/json",
      { stage: "cancelled", reason: "paciente desistiu do tratamento" } ]);
    expect(await listProsthesisOrders({ healthUnitId: "u1", stage: "sent", before: null })).toEqual({ orders: [], next_before: null });
    expect(Object.fromEntries(urlOf(2).searchParams)).toEqual({ health_unit_id: "u1", stage: "sent", limit: "50" });
    await listProsthesisOrders({ patientId: "pa1" });
    expect(Object.fromEntries(urlOf(3).searchParams)).toEqual({ patient_id: "pa1", limit: "50" });
    expect(await listDentalLaboratories()).toEqual([]);
    expect(urlOf(4).pathname).toBe("/dental/laboratories");
  });

  it("administração: perfis do CEO e lista do auxiliar com envelope", async () => {
    fetchMock.mockImplementationOnce(() => ok({ profiles: [ { ceo_type: "II" } ] }));
    expect(await listCeoProfiles()).toEqual([ { ceo_type: "II" } ]);
    expect(urlOf(0).pathname).toBe("/dental/admin/ceo_profiles");
    await putCeoProfile("u9", { ceo_type: "II", specialties: [ "endodontics", "periodontics" ] });
    expect([ ...writeOf(1), bodyOf(1) ]).toEqual([ "/dental/admin/ceo_profiles/u9", "PUT", "application/json",
      { ceo_type: "II", specialties: [ "endodontics", "periodontics" ] } ]);
    fetchMock.mockImplementationOnce(() => ok({ procedures: [ { sigtap: "0101020074", name: "FLÚOR" } ] }));
    expect(await getAuxiliaryProcedures()).toEqual([ { sigtap: "0101020074", name: "FLÚOR" } ]);
    await putAuxiliaryProcedures([ "0101020074" ]);
    expect([ ...writeOf(3), bodyOf(3) ]).toEqual([ "/dental/admin/auxiliary_procedures", "PUT", "application/json",
      { procedures: [ { sigtap: "0101020074" } ] } ]);
  });

  it("painéis: só id, mês e período na URL", async () => {
    await getCeoGoals("u9", "2026-10");
    expect([ urlOf(0).pathname, Object.fromEntries(urlOf(0).searchParams) ]).toEqual([ "/dental/ceo_goals", { health_unit_id: "u9", month: "2026-10" } ]);
    expect(await listDentalTeams()).toEqual([]);
    expect(urlOf(1).pathname).toBe("/dental/teams");
    await getDentalIndicators("t1", "2026-10");
    expect([ urlOf(2).pathname, Object.fromEntries(urlOf(2).searchParams) ]).toEqual([ "/dental/indicators", { team_id: "t1", period: "2026-10" } ]);
  });

  it("nada clínico na URL", async () => {
    await saveConsultationDraft("cs1", { subjective: "", objective: "", assessment: "", plan: "", vitals: {}, care_type: null, evaluated_problems: [],
      conducts: [], exam_requests: [], dental: fields({ findings: [ { tooth: "16", face: "O", situation: "caries" } ],
        plan_changes: { add: [], cancel: [ { item_id: "pi1", reason: "dente quebrou e vai para extração" } ] } }) });
    await changeProsthesisStage("po1", { stage: "cancelled", reason: "paciente desistiu do tratamento" });
    await searchDentalProcedures("RESTAURAÇÃO 0307010031");
    for (const call of fetchMock.mock.calls) expect(decodeURIComponent(String(call[0]))).not.toMatch(SENSITIVE);
  });
});
```

E no fim de `src/lib/features.test.ts`:

```ts
describe("módulo 28", () => {
  it("dental_care tem nome em português", () => {
    expect(featureLabel("dental_care")).toBe("Saúde bucal");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/lib/api.dental.test.ts src/lib/features.test.ts`
Expected: FAIL — `startConsultation` ignora o segundo argumento, as funções novas e `src/test/dentalFixtures.ts` não existem, `featureLabel("dental_care")` devolve a chave.

- [ ] **Step 3: Implemente os tipos e o cliente**

Em `src/lib/api.ts`: os campos novos de `Consultation`, `ConsultationOptions`, `ConsultationDraftInput` e `AddendumChanges` onde esses tipos estão (comentário `// Módulo 28 (contrato §2/§4)`); `startConsultation` com o segundo argumento opcional; e a seção `// ─── Saúde bucal (módulo 28; ADR 0046; contrato §3–§10) ───` no fim, com `const DENTAL_BASE = import.meta.env.VITE_DENTAL_BASE || "/dental"` e `const dentalPath = (...parts: string[]) => [ DENTAL_BASE, ...parts.map(encodeURIComponent) ].join("/")`. As leituras do paciente usam `${ATTENDANCE_BASE}/patients/<id>/…` (D1). Envelopes: `events`/`next_before`, `plans`, `items`, `treatments`, `orders`/`next_before`, `laboratories`, `profiles`, `procedures`, `teams` — ausente vira `[]` (e `next_before` ausente vira `null`). `limit=50` (`DENTAL_PAGE_SIZE`) só nas listas com cursor (eventos e próteses). Parâmetro nulo ou vazio não entra na URL. Escrita com `postProfessional` ou `{ method: "PUT", body }`.

Em `src/lib/features.ts`: `| "dental_care"` em `FeatureKey` e `// Módulo 28 (ADR 0046): exige clinical_record. dental_care: "Saúde bucal"` em `FEATURE_LABEL`. Em `vite.config.ts`, depois da farmácia: `// Módulo 28: saúde bucal. Regex como os vizinhos.` e `"^/dental(/|$)": proxy(TARGET)`.

Em `src/test/dentalFixtures.ts` (códigos SIGTAP ilustrativos, como os do `consultationFixtures`; os reais vêm do api):

```ts
// Dados dos testes do módulo 28 (saúde bucal). Hoje: sexta, 2026-10-30. Formas do contrato §2.
import type { CeoGoals, CeoProfile, ChartEvent, Consultation, ConsultationOptions, DentalFields, DentalLocation, DentalPlan, DentalPlanItem,
  DentalProcedureRef, DentalSituation, DentalTeam, Odontogram, ProsthesisOrder, SessionUser, ToothFace } from "../lib/api";
import { consultation, finalized, options } from "./consultationFixtures";
import { sessionWith } from "./campaignFixtures";

export const TODAY28 = "2026-10-30";
export const UBS = { id: "u1", name: "UBS Centro", kind: "ubs" };
export const CEO_UNIT = { id: "u9", name: "CEO Boqueirão", kind: "ceo" };
export const SITUATIONS: DentalSituation[] = [
  { code: "sound", label: "Hígido", applies_to: "both" }, { code: "caries", label: "Cariado", applies_to: "both" },
  { code: "restored", label: "Restaurado", applies_to: "both" }, { code: "missing", label: "Ausente", applies_to: "tooth" },
  { code: "extraction_indicated", label: "Extração indicada", applies_to: "tooth" }, { code: "prosthesis", label: "Prótese", applies_to: "tooth" }
];
export const RESIN: DentalProcedureRef = { sigtap: "0307010031", name: "RESTAURAÇÃO DE DENTE PERMANENTE POSTERIOR COM RESINA COMPOSTA",
  locations: [ "face", "tooth" ], max_quantity: 32, auxiliary_allowed: false, prosthesis: false };
export const SCALING: DentalProcedureRef = { sigtap: "0307030024", name: "RASPAGEM SUPRAGENGIVAL (POR SEXTANTE)",
  locations: [ "sextant" ], max_quantity: 6, auxiliary_allowed: true, prosthesis: false };
export const FLUOR: DentalProcedureRef = { sigtap: "0101020074", name: "APLICAÇÃO TÓPICA DE FLÚOR (INDIVIDUAL POR SESSÃO)",
  locations: [ "mouth" ], max_quantity: 1, auxiliary_allowed: true, prosthesis: false };
export const DENTURE: DentalProcedureRef = { sigtap: "0701070137", name: "PRÓTESE TOTAL MAXILAR",
  locations: [ "arch" ], max_quantity: 2, auxiliary_allowed: false, prosthesis: true };

export const dentalUser = (roles = [ "health_professional" ], over: Partial<SessionUser> = {}) =>
  sessionWith(roles, { id: "us1", features: [ "clinical_record", "clinical_documents", "dental_care" ], ...over });
export const dentalOptions = (over: Partial<ConsultationOptions> = {}) =>
  options({ kinds: [ "dental" ], dental_role: "dentist", dental_situations: SITUATIONS, ...over });
export const auxOptions = (over: Partial<ConsultationOptions> = {}) => dentalOptions({ dental_role: "auxiliary", ...over });
export function toothLoc(tooth: string, face: ToothFace | null = null): DentalLocation {
  return { kind: face ? "face" : "tooth", tooth, face, sextant: null, arch: null };
}
export function fields(over: Partial<DentalFields> = {}): DentalFields {
  return { care_type: "2", dental_consultation_type: "1", oral_health_surveillance: [ "99" ], supplies: [], dental_outcomes: [ "16" ],
    pregnant: false, special_needs: false, procedures: [], findings: [], plan_changes: { add: [], cancel: [] },
    periodontal_screening: null, periodontal_chart: null, endodontics: [], ...over };
}
export const dentalDraft = (over: Partial<Consultation> = {}) =>
  consultation({ kind: "dental", baseline: false, cbo_code: "223208", care_type: null, author: { id: "us1", name: "Dr. Caio Nunes" }, dental: fields(), ...over });
export const dentalFinalized = (over: Partial<Consultation> = {}) =>
  finalized({ kind: "dental", baseline: false, cbo_code: "223208", care_type: null, conducts: [], author: { id: "us1", name: "Dr. Caio Nunes" },
    dental: fields({ procedures: [ { id: "dpr1", sigtap: RESIN.sigtap, name: RESIN.name, location: toothLoc("16", "O"), quantity: 1, plan_item_id: "pi1" } ] }), ...over });
export function odontogram(over: Partial<Odontogram> = {}): Odontogram {
  const perm = [ 18, 17, 16, 15, 14, 13, 12, 11, 21, 22, 23, 24, 25, 26, 27, 28, 48, 47, 46, 45, 44, 43, 42, 41, 31, 32, 33, 34, 35, 36, 37, 38 ];
  return { patient: { id: "pa1", display_name: "Joana Lima", has_cpf: true, cns_masked: null }, dentition: "permanent",
    teeth: perm.map(String).map((tooth) => ({ tooth, situation: tooth === "36" ? "missing" : "sound",
      faces: tooth === "16" ? { O: "caries" } : tooth === "26" ? { M: "restored", O: "restored" } : {},
      last_event_at: "2026-09-10T14:30:00-03:00", notes_count: 0 })),
    open_plans: [ { id: "dp1", health_unit: UBS } ], ...over };
}
export function chartEvent(over: Partial<ChartEvent> = {}): ChartEvent {
  return { id: "ev1", tooth: "16", face: "O", situation: "caries", procedure: null, consultation_id: "cs0", addendum_id: null,
    by: { name: "Dr. Caio Nunes", cbo: "223208" }, at: "2026-09-10T14:30:00-03:00", ...over };
}
export function planItem(over: Partial<DentalPlanItem> = {}): DentalPlanItem {
  return { id: "pi1", sigtap: RESIN.sigtap, name: RESIN.name, location: toothLoc("16", "O"), status: "planned",
    done_in_consultation_id: null, cancel_reason: null, ...over };
}
export function plan(over: Partial<DentalPlan> = {}): DentalPlan {
  return { id: "dp1", health_unit: UBS, status: "open", opened_in_consultation_id: "cs0", closed_in_consultation_id: null, items: [ planItem() ], ...over };
}
export function prosthesisOrder(over: Partial<ProsthesisOrder> = {}): ProsthesisOrder {
  return { id: "po1", patient: { id: "pa1", display_name: "Joana Lima" }, health_unit: UBS, sigtap: DENTURE.sigtap, name: DENTURE.name,
    laboratory: { care_provider_id: null, name: "Lab Sorriso" }, stage: "impression",
    stages: [ { stage: "impression", at: "2026-10-20T10:00:00-03:00", by: { name: "Dr. Caio Nunes" }, reason: null } ],
    installed_in_consultation_id: null, created_at: "2026-10-20T10:00:00-03:00", ...over };
}
export const ceoProfile = (over: Partial<CeoProfile> = {}): CeoProfile =>
  ({ health_unit: { id: CEO_UNIT.id, name: CEO_UNIT.name }, ceo_type: "II", specialties: [ "endodontics", "periodontics", "oral_surgery", "special_needs" ], ...over });
export function ceoGoals(over: Partial<CeoGoals> = {}): CeoGoals {
  return { ceo_type: "II", specialties: [
    { key: "basic", goal: 110, done: 120, composition: { rule: "restorations_min_50", ok: true, value: 0.62 }, met: true },
    { key: "periodontics", goal: 90, done: 40, composition: null, met: false },
    { key: "endodontics", goal: 60, done: 70, composition: { rule: "multirooted_min_20", ok: false, value: 0.1 }, met: false },
    { key: "oral_surgery", goal: 90, done: 95, composition: null, met: true } ],
    misses_last_12: { periodontics: { consecutive: 1, total: 2 }, endodontics: { consecutive: 0, total: 1 } }, suspension_risk: true, ...over };
}
export const dentalTeam = (over: Partial<DentalTeam> = {}): DentalTeam =>
  ({ id: "t1", name: "eSB Centro 1", ine: "0001234567", health_unit: { id: UBS.id, name: UBS.name }, ...over });
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/lib/api.dental.test.ts src/lib/features.test.ts src/lib/api.consultation.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS (o `api.consultation.test.ts` existente confere que `startConsultation(id)` ainda manda `{}`).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.dental.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/dentalFixtures.ts
$G commit -m "feat: add the oral health api client, feature key and fixtures

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela (`src/lib/dental.ts`)

**Files:**
- Create: `src/lib/dental.ts`
- Test: `src/lib/dental.test.ts`

**Interfaces:**
- Consumes: os tipos da Task 1; `ApiError`, `errorCode` (`api.ts`); `hasFeature`, `featureDisabledKey` (`features.ts`); `NOMINAL_QUERY`, `STATIC_QUERY` (`careLines.ts`, reexportados).
- Produces (em `src/lib/dental.ts`):
  - frases e chaves: `DENTAL_DISABLED = "a saúde bucal está desligada nesta cidade"`, `DENTAL_NO_ROLE = "seu papel não permite usar a saúde bucal"`, `DENTAL_CHART_KEY = "dentalChart"`, `CHART_EVENTS_KEY`, `DENTAL_PLANS_KEY`, `ENDO_KEY`, `PROSTHESIS_KEY`, `LABS_KEY`, `CEO_PROFILES_KEY`, `AUX_PROCEDURES_KEY`, `CEO_GOALS_KEY`, `DENTAL_TEAMS_KEY`, `DENTAL_INDICATORS_KEY`; reexporta `NOMINAL_QUERY` e `STATIC_QUERY`;
  - portão: `canUseDental(user): boolean` (`!operator` e `dental_care` **e** `clinical_record` na sessão); `dentalRoles(user): { professional: boolean; admin: boolean; analyst: boolean }`; `dentalRole(cbo: string | null | undefined): DentalRole | null` (`2232*` → `dentist`; `322405`, `322425`, `322415`, `322430` → `auxiliary`; resto `null`); `offeredKinds(options: ConsultationOptions | undefined, dentalOn: boolean): ConsultationKind[]`;
  - dentes: `chartRows(dentition: Dentition): string[][]` (linhas de cima para baixo; permanente `[18..11 21..28]`, `[48..41 31..38]`; decídua `[55..51 61..65]`, `[85..81 71..75]`; mista = permanente superior, decídua superior, decídua inferior, permanente inferior); `isDeciduous(tooth)`; `toothSextant(tooth): Sextant`; `faceLayout(tooth): { top: ToothFace; bottom: ToothFace; left: ToothFace; right: ToothFace }` (superior: V em cima, L embaixo; inferior: L em cima, V embaixo; quadrantes 1, 4, 5, 8 — à esquerda da tela — com D à esquerda e M à direita; 2, 3, 6, 7 ao contrário);
  - rótulos: `FACE_LABEL: Record<ToothFace, string>` (`M` "mesial", `D` "distal", `O` "oclusal/incisal", `V` "vestibular", `L` "lingual/palatina"); `locationLabel(loc: DentalLocation): string` ("dente 16", "dente 16, face oclusal/incisal", "sextante S1", "arcada superior", "boca toda"); `locationKey(loc): string`; `sextantLoc(s)`, `archLoc(a)`, `MOUTH: DentalLocation`; `situationLabel(code, list?: DentalSituation[]): string` (cru se desconhecido); `situationAbbr(code, list?)` (3 letras do rótulo, maiúsculas; cru curto se desconhecido); `situationTone(code): string` (`caries` → `down`, `extraction_indicated` → `warn`, `restored`/`prosthesis` → `info`, `sound` → `ok`, resto `neutral`); `situationsFor(level: "tooth" | "face", list?)`; `toothAriaLabel(t: OdontogramTooth | undefined, tooth: string, list?, pendingCount?: number): string` — `"Dente <n>: <situação em minúsculas, código cru ou 'sem registro'>"`, depois `"; face <FACE_LABEL>: <situação em minúsculas>"` por face com valor (na ordem M, D, O, V, L) e `"; N marcação|marcações nesta consulta"` quando `pendingCount > 0`;
  - FAO (rótulos da pesquisa §1): `CARE_TYPES` (2 "Consulta agendada", 4 "Escuta inicial / orientação", 5 "Consulta no dia", 6 "Atendimento de urgência"), `CONSULTATION_TYPES` (1 "Primeira consulta odontológica programática", 2 "Consulta de retorno", 4 "Consulta de manutenção"), `SURVEILLANCE` (1 "Abscesso dentoalveolar", 2 "Alteração em tecidos moles", 3 "Dor de dente", 4 "Fendas ou fissuras labiopalatais", 5 "Fluorose dentária moderada ou severa", 6 "Traumatismo dentoalveolar", 99 "Não identificado"), `SUPPLIES` (1 "Escova dental", 2 "Creme dental", 3 "Fio dental"), `OUTCOMES` (1–11 "Encaminhamento: pacientes com necessidades especiais", "… cirurgia bucomaxilofacial", "… endodontia", "… estomatologia", "… implantodontia", "… odontopediatria", "… ortodontia/ortopedia", "… periodontia", "… prótese dentária", "… radiologia", "… outros"; 12 "Agendamento para outros profissionais da AB"; 14 "Agendamento para grupos"; 15 "Tratamento concluído"; 16 "Retorno para consulta agendada"; 17 "Alta do episódio"; 18 "Agendamento para eMulti"), `REFERRAL_OUTCOMES = ["1".."11"]`, `toggleSurveillance(list: string[], code: string): string[]` ("99" exclui os demais e vice-versa);
  - mudanças do rascunho: `withFinding(f: DentalFields, x: DentalFinding): DentalFields` (substitui o achado do mesmo dente e face), `withoutFinding(f, i)`, `withProcedure(f, line: DentalProcedureLine)`, `withoutProcedure(f, i)`, `linkPlanItem(f, i, itemId: string | null)`, `withPlanAdd(f, add: PlanAdd)`, `withPlanCancel(f, c: PlanCancel)`, `withoutPlanChange(f, kind: "add" | "cancel", i)`;
  - limites: `LOCATION_LIMIT = { tooth: 32, face: 32, sextant: 6, arch: 2, mouth: 1 }`; `quantityProblems(procs: DentalProcedureLine[]): string[]` (por SIGTAP, somando dente+face, sextante, arcada e boca à parte; frase `"<nome ou sigtap>: no máximo N por <local> nesta consulta"`); quantidade não inteira ou < 1 → `"<nome>: a quantidade é um número inteiro a partir de 1"`;
  - plano: `planItemsAfter(plan: DentalPlan | null, f: DentalFields): { item: DentalPlanItem; state: "planned" | "done_here" | "cancelled_here" | "done" | "cancelled" }[]`; `suggestCompleted(plan, f): boolean` (plano aberto com ≥ 1 item, nenhum `planned` depois do rascunho e nenhuma inclusão nova); `canAddPlanItems(unit: { kind: string }, plan: DentalPlan | null, f: DentalFields, role: DentalRole): boolean` (dentista; no CEO sempre; na UBS com plano aberto, ou sem plano só com `dental_consultation_type === "1"` — D11);
  - finalização: `DentalCheckCtx { role: DentalRole; baseline: boolean; sex: "female" | "male"; problemsCount: number }`; `dentalFinalizeProblems(f: DentalFields, ctx: DentalCheckCtx): { code: string; text: string }[]` (todas de uma vez, na ordem da tabela do teste); `pregnantFor(f, sex): boolean | null` (masculino → `null`);
  - recusas: `dentalProblemText(code: string): string` (código desconhecido cru); `dentalError(err: unknown): string`; `dentalApiProblems(err: unknown): { code: string; path: string | null; text: string }[]` (lê `problems` do corpo; sem ele, o do topo; outro erro → `[]`); `planItemPathIndex(path: string | null): number | null` (`"dental.procedures[2].plan_item_id"` → 2);
  - corpo: `dentalFieldsFrom(c: Consultation): DentalFields` (sem `suggest_completed`); `dentalPayload(f: DentalFields): DentalFields` (sem `name` nas linhas e nas inclusões);
  - especialidades: `PERIO_SITES: PerioSite[]` (`MV, V, DV, ML, L, DL`); `perioProblems(chart: PeriodontalChart): string[]`; `screeningProblems(s: PeriodontalScreening): string[]`; `endoProblems(list: EndodonticEntry[]): string[]`;
  - prótese, CEO e painéis: `prosthesisStageLabel(code)`, `prosthesisStageTone(code)`, `nextStages(stage): ("sent" | "received" | "cancelled")[]` (`impression` → sent, cancelled; `sent` → received, cancelled; `received` → cancelled; resto → []); `reasonProblem(text): string | null` (10–500); `labNameProblem(text)` (3–120); `CEO_TYPES`, `CEO_SPECIALTIES` (com rótulos), `goalKeyLabel(key)` (`basic` "Procedimentos básicos", `periodontics` "Periodontia", `endodontics` "Endodontia", `oral_surgery` "Cirurgia oral"); `compositionLabel(rule)`; `ratioLabel(n: number, d: number): string` ("42,5%"; `d = 0` → "—"); `monthOf(today: string): string` ("2026-10").

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/dental.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  canAddPlanItems, canUseDental, chartRows, dentalApiProblems, dentalError, dentalFinalizeProblems, dentalPayload, dentalRole, faceLayout,
  locationLabel, nextStages, offeredKinds, perioProblems, planItemPathIndex, pregnantFor, quantityProblems, ratioLabel, sextantLoc, archLoc, MOUTH,
  suggestCompleted, toggleSurveillance, toothAriaLabel, toothSextant, withFinding
} from "./dental";
import { CEO_UNIT, DENTURE, FLUOR, RESIN, SCALING, SITUATIONS, UBS, fields, plan, planItem, toothLoc } from "../test/dentalFixtures";

const ctx = { role: "dentist" as const, baseline: false, sex: "female" as const, problemsCount: 1 };
const codes = (f = fields(), c = ctx) => dentalFinalizeProblems(f, c).map((p) => p.code);

describe("portão e papel", () => {
  it("dental_care exige clinical_record; operador nunca", () => {
    expect(canUseDental({ operator: false, features: [ "dental_care", "clinical_record" ] })).toBe(true);
    expect(canUseDental({ operator: false, features: [ "dental_care" ] })).toBe(false);
    expect(canUseDental({ operator: true, features: [ "dental_care", "clinical_record" ] })).toBe(false);
  });
  it("papel pelo CBO; tipos oferecidos pelo options e pelo interruptor", () => {
    expect([ "223208", "223293", "322405", "322425", "322415", "322430", "322410", "225142", null ].map(dentalRole))
      .toEqual([ "dentist", "dentist", "auxiliary", "auxiliary", "auxiliary", "auxiliary", null, null, null ]);
    expect(offeredKinds(undefined, true)).toEqual([ "general" ]);
    expect(offeredKinds({ care_types: [], conducts: [], cid10_allowed_for_cbo: true, kinds: [ "general", "dental" ] }, false)).toEqual([ "general" ]);
    expect(offeredKinds({ care_types: [], conducts: [], cid10_allowed_for_cbo: true, kinds: [ "dental" ] }, true)).toEqual([ "dental" ]);
  });
});

describe("dentes, faces e locais", () => {
  it("linhas por dentição, na ordem da tela", () => {
    expect(chartRows("permanent").map((r) => [ r[0], r[7], r[8], r[15] ])).toEqual([ [ "18", "11", "21", "28" ], [ "48", "41", "31", "38" ] ]);
    expect(chartRows("deciduous")).toEqual([ [ "55", "54", "53", "52", "51", "61", "62", "63", "64", "65" ], [ "85", "84", "83", "82", "81", "71", "72", "73", "74", "75" ] ]);
    expect(chartRows("mixed").map((r) => r.length)).toEqual([ 16, 10, 10, 16 ]);
  });
  it("faces por quadrante: mesial sempre para o meio", () => {
    expect(faceLayout("16")).toEqual({ top: "V", bottom: "L", left: "D", right: "M" });
    expect(faceLayout("26")).toEqual({ top: "V", bottom: "L", left: "M", right: "D" });
    expect(faceLayout("36")).toEqual({ top: "L", bottom: "V", left: "M", right: "D" });
    expect(faceLayout("46")).toEqual({ top: "L", bottom: "V", left: "D", right: "M" });
    expect(faceLayout("84")).toEqual({ top: "L", bottom: "V", left: "D", right: "M" });
  });
  it("sextantes e rótulos de local", () => {
    expect([ "17", "13", "23", "24", "37", "33", "43", "44", "55", "63", "75", "84" ].map(toothSextant))
      .toEqual([ "S1", "S2", "S2", "S3", "S4", "S5", "S5", "S6", "S1", "S2", "S4", "S6" ]);
    expect([ toothLoc("16"), toothLoc("16", "O"), sextantLoc("S1"), archLoc("upper"), MOUTH ].map(locationLabel))
      .toEqual([ "dente 16", "dente 16, face oclusal/incisal", "sextante S1", "arcada superior", "boca toda" ]);
  });
  it("rótulo do dente diz situação, faces e o que está marcado hoje", () => {
    expect(toothAriaLabel({ tooth: "16", situation: "sound", faces: { O: "caries", M: null }, last_event_at: null, notes_count: 0 }, "16", SITUATIONS, 1))
      .toBe("Dente 16: hígido; face oclusal/incisal: cariado; 1 marcação nesta consulta");
    expect(toothAriaLabel({ tooth: "36", situation: "implant_x", faces: {}, last_event_at: null, notes_count: 0 }, "36", SITUATIONS)).toBe("Dente 36: implant_x");
    expect(toothAriaLabel(undefined, "55", SITUATIONS)).toBe("Dente 55: sem registro");
  });
  it("achado do mesmo dente e face substitui o anterior", () => {
    const f = withFinding(withFinding(fields(), { tooth: "16", face: "O", situation: "caries" }), { tooth: "16", face: "O", situation: "restored" });
    expect(f.findings).toEqual([ { tooth: "16", face: "O", situation: "restored" } ]);
  });
});

describe("limites por local", () => {
  const line = (ref: typeof RESIN, location = toothLoc("16"), quantity = 1) => ({ sigtap: ref.sigtap, name: ref.name, location, quantity, plan_item_id: null });
  it("limites por local: recusa sem cortar; dente e face somam juntos", () => {
    const seven = [ "S1", "S2", "S3", "S4", "S5", "S6", "S1" ].map((s) => line(SCALING, sextantLoc(s as "S1")));
    expect(quantityProblems(seven)).toEqual([ "RASPAGEM SUPRAGENGIVAL (POR SEXTANTE): no máximo 6 por sextante nesta consulta" ]);
    expect(quantityProblems([ line(DENTURE, archLoc("upper")), line(DENTURE, archLoc("lower")), line(DENTURE, archLoc("upper")) ]))
      .toEqual([ "PRÓTESE TOTAL MAXILAR: no máximo 2 por arcada nesta consulta" ]);
    expect(quantityProblems([ line(FLUOR, MOUTH), line(FLUOR, MOUTH) ])).toEqual([ "APLICAÇÃO TÓPICA DE FLÚOR (INDIVIDUAL POR SESSÃO): no máximo 1 por boca toda nesta consulta" ]);
    expect(quantityProblems([ line(RESIN, toothLoc("16"), 30), line(RESIN, toothLoc("26", "O"), 3) ]))
      .toEqual([ `${RESIN.name}: no máximo 32 por dente nesta consulta` ]);
    expect(quantityProblems([ line(RESIN, toothLoc("16"), 0), line(RESIN, toothLoc("17"), 1.5) ]))
      .toEqual([ `${RESIN.name}: a quantidade é um número inteiro a partir de 1` ]);
    expect(quantityProblems([ ...[ "S1", "S2", "S3", "S4", "S5", "S6" ].map((s) => line(SCALING, sextantLoc(s as "S1"))), line(RESIN, toothLoc("16", "O")) ])).toEqual([]);
  });
});

describe("finalização: espelho das recusas do api (contrato §4)", () => {
  it.each([
    [ "vigilância vazia", fields({ oral_health_surveillance: [] }), ctx, [ "oral_health_surveillance_required" ] ],
    [ "agendada sem tipo", fields({ care_type: "2", dental_consultation_type: null }), ctx, [ "dental_consultation_type_required" ] ],
    [ "escuta com tipo", fields({ care_type: "4", dental_consultation_type: "4" }), ctx, [ "dental_consultation_type_forbidden" ] ],
    [ "no dia sem tipo", fields({ care_type: "5", dental_consultation_type: null }), ctx, [] ],
    [ "urgência e retorno", fields({ care_type: "6", dental_consultation_type: "2" }), ctx, [ "return_forbidden_on_urgency" ] ],
    [ "concluído na manutenção", fields({ dental_consultation_type: "4", dental_outcomes: [ "15" ] }), ctx, [ "completed_requires_first_or_return" ] ],
    [ "concluído no retorno", fields({ dental_consultation_type: "2", dental_outcomes: [ "15" ] }), ctx, [] ],
    [ "alta na primeira", fields({ dental_outcomes: [ "17" ] }), ctx, [ "discharge_forbidden_with_first_or_return" ] ],
    [ "sem desfecho", fields({ dental_outcomes: [] }), ctx, [ "dental_outcome_required" ] ],
    [ "sem tipo de atendimento", fields({ care_type: null, dental_consultation_type: null }), ctx, [ "care_type_required" ] ],
    [ "gestante vazia", fields({ pregnant: null }), ctx, [ "pregnant_required" ] ],
    [ "gestante no masculino", fields({ pregnant: true }), { ...ctx, sex: "male" as const }, [ "pregnant_forbidden_male" ] ],
    [ "sem problema", fields(), { ...ctx, problemsCount: 0 }, [ "problem_required" ] ],
    [ "auxiliar encaminha", fields({ dental_outcomes: [ "3" ] }), { ...ctx, role: "auxiliary" as const }, [ "auxiliary_cannot_refer" ] ],
    [ "auxiliar planeja", fields({ plan_changes: { add: [ { sigtap: RESIN.sigtap, location: toothLoc("16") } ], cancel: [] } }), { ...ctx, role: "auxiliary" as const }, [ "auxiliary_cannot_plan" ] ],
    [ "registro inicial com procedimento", fields({ care_type: null, dental_consultation_type: null, dental_outcomes: [], oral_health_surveillance: [], pregnant: null,
      findings: [ { tooth: "16", face: null, situation: "caries" } ], procedures: [ { sigtap: FLUOR.sigtap, location: MOUTH, quantity: 1, plan_item_id: null } ] }),
      { ...ctx, baseline: true, problemsCount: 0 }, [ "baseline_only_findings" ] ],
    [ "registro inicial vazio", fields({ care_type: null, dental_consultation_type: null, dental_outcomes: [], oral_health_surveillance: [], pregnant: null }),
      { ...ctx, baseline: true, problemsCount: 0 }, [ "baseline_requires_findings" ] ]
  ])("%s", (_name, f, c, expected) => {
    expect(codes(f, c)).toEqual(expected);
  });
  it("todas de uma vez, com frase", () => {
    const out = dentalFinalizeProblems(fields({ care_type: "4", dental_consultation_type: "1", oral_health_surveillance: [], dental_outcomes: [ "15", "3" ], pregnant: null }),
      { ...ctx, role: "auxiliary", problemsCount: 0 });
    expect(out.map((p) => p.code)).toEqual([ "oral_health_surveillance_required", "dental_consultation_type_forbidden", "pregnant_required",
      "problem_required", "auxiliary_cannot_refer" ]);
    expect(out[0].text).toBe("marque a vigilância em saúde bucal (ou \"não identificado\")");
  });
  it("gestante: obrigatória no feminino, nula no masculino", () => {
    expect(pregnantFor(fields({ pregnant: true }), "male")).toBeNull();
    expect(pregnantFor(fields({ pregnant: true }), "female")).toBe(true);
    expect(codes(fields({ pregnant: null }), { ...ctx, sex: "male" })).toEqual([]);
  });
  it("vigilância: 'não identificado' exclui os demais", () => {
    expect(toggleSurveillance([ "3" ], "99")).toEqual([ "99" ]);
    expect(toggleSurveillance([ "99" ], "3")).toEqual([ "3" ]);
    expect(toggleSurveillance([ "3", "6" ], "3")).toEqual([ "6" ]);
  });
});

describe("plano", () => {
  it("sugere concluído só sem item planejado depois do rascunho", () => {
    const p = plan({ items: [ planItem(), planItem({ id: "pi2", status: "done" }), planItem({ id: "pi3", status: "planned" }) ] });
    const linked = fields({ procedures: [ { sigtap: RESIN.sigtap, location: toothLoc("16", "O"), quantity: 1, plan_item_id: "pi1" } ] });
    expect(suggestCompleted(p, linked)).toBe(false);
    expect(suggestCompleted(p, { ...linked, plan_changes: { add: [], cancel: [ { item_id: "pi3", reason: "paciente recusou o tratamento" } ] } })).toBe(true);
    expect(suggestCompleted(p, { ...linked, plan_changes: { add: [ { sigtap: FLUOR.sigtap, location: MOUTH } ], cancel: [ { item_id: "pi3", reason: "paciente recusou o tratamento" } ] } })).toBe(false);
    expect(suggestCompleted(null, linked)).toBe(false);
    expect(suggestCompleted(plan({ items: [] }), fields())).toBe(false);
  });
  it("quem acrescenta item: dentista; na UBS sem plano só na primeira consulta; no CEO sempre", () => {
    expect(canAddPlanItems(UBS, plan(), fields({ dental_consultation_type: "2" }), "dentist")).toBe(true);
    expect(canAddPlanItems(UBS, null, fields({ dental_consultation_type: "2" }), "dentist")).toBe(false);
    expect(canAddPlanItems(UBS, null, fields({ dental_consultation_type: "1" }), "dentist")).toBe(true);
    expect(canAddPlanItems(CEO_UNIT, null, fields({ dental_consultation_type: null, care_type: "5" }), "dentist")).toBe(true);
    expect(canAddPlanItems(CEO_UNIT, plan(), fields(), "auxiliary")).toBe(false);
  });
});

describe("recusas do api, corpo e especialidades", () => {
  const e = (status: number, body: unknown) => new ApiError(status, body, "x");
  it("problems todos de uma vez; código novo cru; caminho do item do plano", () => {
    const err = e(422, { error: "oral_health_surveillance_required", problems: [ { path: "dental.oral_health_surveillance", error: "oral_health_surveillance_required" },
      { path: "dental.procedures[1].plan_item_id", error: "plan_item_not_open" }, { path: null, error: "brand_new_rule" } ] });
    expect(dentalApiProblems(err).map((p) => [ p.code, p.path ])).toEqual([ [ "oral_health_surveillance_required", "dental.oral_health_surveillance" ],
      [ "plan_item_not_open", "dental.procedures[1].plan_item_id" ], [ "brand_new_rule", null ] ]);
    expect(dentalApiProblems(err)[2].text).toBe("brand_new_rule");
    expect(dentalApiProblems(e(422, { error: "pregnant_required" })).map((p) => p.code)).toEqual([ "pregnant_required" ]);
    expect(planItemPathIndex("dental.procedures[1].plan_item_id")).toBe(1);
    expect(planItemPathIndex("dental.findings[0]")).toBeNull();
  });
  it("frases das recusas de início e de documento", () => {
    expect(dentalError(e(403, { error: "feature_disabled", feature: "dental_care" }))).toBe("a saúde bucal está desligada nesta cidade");
    expect(dentalError(e(403, { error: "cbo_not_allowed" }))).toBe("seu CBO não registra consulta odontológica");
    expect(dentalError(e(422, { error: "baseline_requires_dentist" }))).toBe("só o dentista faz o registro inicial do odontograma");
    expect(dentalError(e(403, { error: "auxiliary_cannot_issue" }))).toBe("TSB e ASB não emitem documento");
  });
  it("corpo sem nome e sem sugestão", () => {
    const f = fields({ procedures: [ { sigtap: RESIN.sigtap, name: RESIN.name, location: toothLoc("16"), quantity: 1, plan_item_id: null } ],
      plan_changes: { add: [ { sigtap: FLUOR.sigtap, name: FLUOR.name, location: MOUTH } ], cancel: [] } });
    expect(dentalPayload(f).procedures[0]).toEqual({ sigtap: RESIN.sigtap, location: toothLoc("16"), quantity: 1, plan_item_id: null });
    expect(dentalPayload(f).plan_changes.add[0]).toEqual({ sigtap: FLUOR.sigtap, location: MOUTH });
  });
  it("periograma: faixa e inteiro, recusa sem cortar", () => {
    const site = (over = {}) => ({ site: "MV" as const, probing_depth_mm: 3, recession_mm: 0, bleeding: false, ...over });
    expect(perioProblems({ teeth: [ { tooth: "16", mobility: 1, furcation: null, sites: [ site(), site({ site: "V", probing_depth_mm: 21 }),
      site({ site: "DV", recession_mm: -11 }), site({ site: "ML", probing_depth_mm: 2.5 }) ] }, { tooth: "26", mobility: 4, furcation: 0, sites: [] } ] }))
      .toEqual([ "dente 16, sítio V: profundidade de sondagem de 0 a 20 mm", "dente 16, sítio DV: recessão de -10 a 20 mm",
        "dente 16, sítio ML: profundidade de sondagem em milímetros inteiros", "dente 26: mobilidade de 0 a 3" ]);
  });
  it("prótese e painéis", () => {
    expect([ "impression", "sent", "received", "installed", "cancelled" ].map(nextStages))
      .toEqual([ [ "sent", "cancelled" ], [ "received", "cancelled" ], [ "cancelled" ], [], [] ]);
    expect([ ratioLabel(17, 40), ratioLabel(0, 0), ratioLabel(1, 3) ]).toEqual([ "42,5%", "—", "33,3%" ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/lib/dental.test.ts`
Expected: FAIL — `./dental` não existe.

- [ ] **Step 3: Implemente `src/lib/dental.ts`**

As assinaturas do Interfaces. Pontos que os testes não decidem sozinhos:
- `dentalFinalizeProblems` confere, nesta ordem (a do teste "todas de uma vez"): registro inicial (`baseline_only_findings` se houver procedimento, desfecho, `dental_consultation_type`, `care_type` ou mudança de plano; `baseline_requires_findings` se não houver achado — e nada mais é conferido); `care_type_required`; `oral_health_surveillance_required`; as três regras do tipo de consulta (`_required` com 2, `_forbidden` com 4, `return_forbidden_on_urgency` com 6 + "2"); `dental_outcome_required`; `completed_requires_first_or_return` (15 sem "1"/"2"); `discharge_forbidden_with_first_or_return` (17 com "1"/"2"); `pregnant_required` (feminino com `null`); `pregnant_forbidden_male` (masculino com `true`); `problem_required`; `auxiliary_cannot_refer` (auxiliar com algum de 1–11); `auxiliary_cannot_plan` (auxiliar com `add` ou `cancel`); `quantity_exceeds_location_limit` (com o texto de `quantityProblems`, um por linha). `auxiliary_procedure_not_allowed` e `plan_item_not_open` só o api apura (a busca já filtra; o plano pode mudar depois da leitura) — só frase.
- Frases (`dentalProblemText`): `care_type_required` "escolha o tipo de atendimento"; `oral_health_surveillance_required` "marque a vigilância em saúde bucal (ou \"não identificado\")"; `dental_consultation_type_required` "consulta agendada exige o tipo de consulta odontológica"; `dental_consultation_type_forbidden` "escuta inicial não leva tipo de consulta odontológica"; `return_forbidden_on_urgency` "atendimento de urgência não pode ser consulta de retorno"; `dental_outcome_required` "marque ao menos um desfecho"; `completed_requires_first_or_return` "\"tratamento concluído\" só vale com primeira consulta programática ou retorno"; `discharge_forbidden_with_first_or_return` "\"alta do episódio\" não vale com primeira consulta programática ou retorno"; `pregnant_required` "informe se a paciente está gestante"; `pregnant_forbidden_male` "gestante não se aplica a paciente do sexo masculino"; `problem_required` "inclua ao menos um problema (CID-10 ou CIAP-2)"; `auxiliary_cannot_refer` "TSB e ASB não registram encaminhamento"; `auxiliary_procedure_not_allowed` "procedimento fora da lista do TSB/ASB da cidade"; `auxiliary_cannot_plan` "TSB e ASB não criam nem cancelam item do plano"; `quantity_exceeds_location_limit` "quantidade acima do limite do local"; `baseline_only_findings` "o registro inicial só aceita a situação dos dentes"; `baseline_requires_findings` "marque ao menos um dente no registro inicial"; `plan_item_not_open` "um item do plano já foi feito ou cancelado em outra consulta — o plano foi relido; confira o procedimento"; `plan_requires_first_consultation` "na UBS, o plano começa na primeira consulta programática"; `cbo_not_allowed` "seu CBO não registra consulta odontológica"; `baseline_requires_dentist` "só o dentista faz o registro inicial do odontograma"; `auxiliary_cannot_issue` "TSB e ASB não emitem documento"; `feature_disabled` → `DENTAL_DISABLED`; `invalid_stage` "o pedido de prótese mudou de etapa — a lista foi relida"; desconhecido → o código. `dentalError` cai no `consultationError` (de `src/lib/consultation.ts`) para o que não for daqui.
- `quantityProblems` agrupa por `sigtap` e por grupo de limite (`tooth`+`face` → "dente"; `sextant` → "sextante"; `arch` → "arcada"; `mouth` → "boca toda"), soma `quantity` e compara com `LOCATION_LIMIT`; uma frase por (SIGTAP, grupo) estourado, na ordem da primeira linha; a linha com quantidade inválida gera a frase de inteiro uma vez por SIGTAP e não entra na soma.
- `ratioLabel` usa `Intl.NumberFormat("pt-BR", { minimumFractionDigits: 1, maximumFractionDigits: 1 })` sobre `n / d * 100`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/lib/dental.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/dental.ts src/lib/dental.test.ts
$G commit -m "feat: add the oral health screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Saúde bucal", portão e unidade CEO no cadastro (F-28.1)

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts` (contagens de grupos), `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/attendance.ts`, `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`
- Create: `src/modules/dental/DentalGate.tsx`, `src/modules/DentalProsthesis.tsx`, `src/modules/DentalCeoGoals.tsx`, `src/modules/DentalIndicators.tsx`, `src/modules/DentalAdmin.tsx` (telas-portão; o corpo vem nas Tasks 10–12)
- Test: `src/shell/modules.dental.test.ts`, `src/modules/attendance/UnitForm.dental.test.tsx`

**Interfaces:**
- Consumes: `canUseDental`, `dentalRoles`, `DENTAL_DISABLED`, `DENTAL_NO_ROLE` (Task 2).
- Produces: `ModuleId` ganha `"dental-prosthesis" | "dental-ceo-goals" | "dental-indicators" | "dental-admin"`; grupo `{ label: "Saúde bucal", items: [ dental-prosthesis "Próteses" "⌓", dental-ceo-goals "Metas do CEO" "◎", dental-indicators "Indicadores B1 e B2" "◑", dental-admin "CEO e lista do TSB/ASB" "⚙" ] }` depois de "Farmácia"; regras: próteses → `canUseDental && isProfessional`; metas e indicadores → `canUseDental && (isAdmin || isAnalyst)`; administração → `canUseDental && isAdmin`. `DentalGate({ title, sub, allow(roles): boolean, children(roles): ReactNode })` (molde do `PharmacyGate`). `CEO_UNIT_KIND = { value: "ceo", label: "CEO (Centro de Especialidades Odontológicas)" }` e `unitKindOptions(dentalOn: boolean)` em `attendance.ts`; o rótulo `ceo` entra sempre no `KIND_LABEL` de `Units.tsx` (unidade já criada continua legível com o interruptor desligado). Catálogo: grupo "Saúde bucal" `IconDental` "Prontuário odontológico, CEO e prótese"; `dental-prosthesis` `IconDentalBroken` "Pedidos ao laboratório, etapas e instalação"; `dental-ceo-goals` `IconTargetArrow` "Meta mensal por especialidade e risco de suspensão"; `dental-indicators` `IconChartBar` "Primeira consulta e tratamento concluído por equipe"; `dental-admin` `IconSettings` "Tipo e especialidades do CEO; o que TSB e ASB registram".

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.dental.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor } from "./modules";
import { CATALOG } from "./catalog";

const user = (roles: string[], features = [ "clinical_record", "dental_care" ]) =>
  ({ operator: false, memberships: roles.map((role) => ({ role })), features });
const items = (u: ReturnType<typeof user>) => navGroupsFor(u).find((g) => g.label === "Saúde bucal")?.items.map((i) => i.id) ?? [];

describe("grupo Saúde bucal (módulo 28)", () => {
  it("cada papel vê o seu", () => {
    expect(items(user([ "health_professional" ]))).toEqual([ "dental-prosthesis" ]);
    expect(items(user([ "analyst" ]))).toEqual([ "dental-ceo-goals", "dental-indicators" ]);
    expect(items(user([ "municipal_admin" ]))).toEqual([ "dental-ceo-goals", "dental-indicators", "dental-admin" ]);
    expect(items(user([ "citizen_verifier" ]))).toEqual([]);
  });
  it("sem dental_care, ou sem clinical_record, o grupo some", () => {
    expect(items(user([ "municipal_admin", "health_professional" ], [ "clinical_record" ]))).toEqual([]);
    expect(items(user([ "municipal_admin", "health_professional" ], [ "dental_care" ]))).toEqual([]);
  });
  it("grupo e itens no catálogo, com ícone e descrição", () => {
    const group = NAV_GROUPS.find((g) => g.label === "Saúde bucal")!;
    expect(CATALOG.groups["Saúde bucal"]?.description).toBe("Prontuário odontológico, CEO e prótese");
    for (const item of group.items) expect(CATALOG.items[item.id]?.description).toBeTruthy();
  });
});
```

```tsx
// src/modules/attendance/UnitForm.dental.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen } from "@testing-library/react";
vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn() }));
import * as api from "../../lib/api";
import { UnitForm } from "./UnitForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { dentalUser } from "../../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("UnitForm — CEO (módulo 28)", () => {
  it.each([ [ [ "clinical_record", "dental_care" ], true ], [ [ "clinical_record" ], false ] ])("features %j: CEO oferecido = %s", async (features, offered) => {
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser([ "municipal_admin" ], { features }));
    renderWithProviders(<UnitForm neighborhoods={[]} busy={false} onSave={() => undefined} onCancel={() => undefined} />);
    await screen.findByRole("option", { name: "UBS" });
    expect(screen.queryByRole("option", { name: "CEO (Centro de Especialidades Odontológicas)" }) !== null).toBe(offered);
  });
});
```

(Confira as props obrigatórias reais do `UnitForm` no Step 2 da Task 0 e ajuste a chamada; o teste cobra só a opção.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/shell/modules.dental.test.ts src/modules/attendance/UnitForm.dental.test.tsx`
Expected: FAIL — o grupo e a opção não existem.

- [ ] **Step 3: Implemente**

`modules.ts`: os quatro ids, o grupo e as regras (comentário `// Módulo 28 (ADR 0046): dental_care E clinical_record (canUseDental). Próteses do profissional (o api confere o CBO 2232); metas e B1/B2 do admin e do analyst; cadastro do CEO e lista do TSB/ASB do admin.`). `modules.test.ts`: os dois testes que contam grupos escondidos passam de `- 15` para `- 16` e de `- 14` para `- 15`, com `expect(groups.some((g) => g.label === "Saúde bucal")).toBe(false)`. `catalog.ts` e `App.tsx` (`case "dental-…": return <Dental… />`, com `onGoToSecurity={() => setActive("security")}` no `DentalAdmin`). As quatro telas-portão: `DentalGate` com o título do item e `allow` igual à regra do menu; o corpo, por ora, `<EmptyState title="Em construção nesta branch." />` — substituído nas Tasks 10–12 (nenhum commit da branch vai para a main antes disso). `UnitForm` lê `useAuth()` e usa `unitKindOptions(canUseDental(user))`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/shell src/modules/attendance && cd $W && npx tsc --noEmit`
Expected: PASS (inclusive `catalog.test.ts` e os testes antigos de unidades).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.dental.test.ts src/shell/catalog.ts src/App.tsx src/lib/attendance.ts \
  src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/attendance/UnitForm.dental.test.tsx \
  src/modules/dental/DentalGate.tsx src/modules/DentalProsthesis.tsx src/modules/DentalCeoGoals.tsx src/modules/DentalIndicators.tsx src/modules/DentalAdmin.tsx
$G commit -m "feat: add the oral health menu group, gate and the CEO unit kind

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Odontograma (F-28.3)

**Files:**
- Create: `src/modules/dental/Odontogram.tsx`
- Test: `src/modules/dental/Odontogram.test.tsx`

**Interfaces:**
- Consumes: `chartRows`, `faceLayout`, `situationAbbr`, `situationTone`, `situationLabel`, `toothAriaLabel`, `FACE_LABEL`, `isDeciduous` (Task 2); `SegmentedControl`.
- Produces: `Odontogram({ teeth: OdontogramTooth[]; dentition: Dentition; situations?: DentalSituation[]; pending?: Record<string, number>; selected: { tooth: string; face: ToothFace | null } | null; onSelect(sel: { tooth: string; face: ToothFace | null } | null): void; onDentitionChange?(d: Dentition): void })` — a `SegmentedControl` "Permanente | Decídua | Mista" (só com `onDentitionChange`); `role="group"` com `aria-label="Odontograma"`; uma linha por `chartRows(dentition)`, dividida ao meio por uma régua vertical (linha média), e as superiores separadas das inferiores por uma régua horizontal; cada dente é um `button` (`aria-label` = `toothAriaLabel`, `aria-pressed` quando selecionado) com o número, a sigla da situação e o desenho das cinco faces (`aria-hidden`, cada face um `span` com a cor do `situationTone` da face, por papel do tema, e `onPointerUp` que seleciona `{ tooth, face }` sem tirar o foco do botão); dente sem registro no `teeth` aparece com situação vazia (`—`); marcação pendente desta consulta (`pending[tooth] > 0`) vira borda tracejada e "•" no número. Tamanho: cada botão `minWidth: 44, minHeight: 56`; a grade é `gridTemplateColumns: repeat(N, minmax(44px, 1fr))` dentro de um contêiner `overflowX: "auto"` com `maxWidth: "100%"` (no telefone a linha rola dentro do odontograma, nunca a página).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/dental/Odontogram.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { Odontogram } from "./Odontogram";
import { SITUATIONS, odontogram } from "../../test/dentalFixtures";

afterEach(cleanup);
const teeth = odontogram().teeth;
const buttons = () => screen.getAllByRole("button").filter((b) => /^Dente \d\d/.test(b.getAttribute("aria-label") ?? ""));

describe("Odontogram", () => {
  it("dentições: 32, 20 e 52 dentes, cada um com rótulo completo", () => {
    const { rerender } = render(<Odontogram teeth={teeth} dentition="permanent" situations={SITUATIONS} selected={null} onSelect={vi.fn()} />);
    expect(buttons()).toHaveLength(32);
    expect(screen.getByRole("button", { name: "Dente 16: hígido; face oclusal/incisal: cariado" })).not.toBeNull();
    expect(screen.getByRole("button", { name: "Dente 36: ausente" })).not.toBeNull();
    rerender(<Odontogram teeth={teeth} dentition="deciduous" situations={SITUATIONS} selected={null} onSelect={vi.fn()} />);
    expect(buttons()).toHaveLength(20);
    expect(screen.getByRole("button", { name: "Dente 55: sem registro" })).not.toBeNull();
    rerender(<Odontogram teeth={teeth} dentition="mixed" situations={SITUATIONS} selected={null} onSelect={vi.fn()} />);
    expect(buttons()).toHaveLength(52);
  });

  it("teclado: tabindex móvel, setas atravessam quadrantes, Enter seleciona, Escape limpa", () => {
    const onSelect = vi.fn();
    render(<Odontogram teeth={teeth} dentition="permanent" situations={SITUATIONS} selected={null} onSelect={onSelect} />);
    const tabbable = buttons().filter((b) => b.tabIndex === 0);
    expect(tabbable.map((b) => b.textContent?.slice(0, 2))).toEqual([ "18" ]);
    const b11 = screen.getByRole("button", { name: /^Dente 11:/ });
    b11.focus();
    fireEvent.keyDown(b11, { key: "ArrowRight" });
    expect(document.activeElement?.getAttribute("aria-label")).toMatch(/^Dente 21:/);
    fireEvent.keyDown(document.activeElement!, { key: "ArrowDown" });
    expect(document.activeElement?.getAttribute("aria-label")).toMatch(/^Dente 31:/);
    fireEvent.keyDown(document.activeElement!, { key: "End" });
    expect(document.activeElement?.getAttribute("aria-label")).toMatch(/^Dente 38:/);
    fireEvent.keyDown(document.activeElement!, { key: "ArrowLeft" });
    fireEvent.keyDown(document.activeElement!, { key: "ArrowLeft" });
    fireEvent.keyDown(document.activeElement!, { key: "Enter" });
    expect(onSelect).toHaveBeenLastCalledWith({ tooth: "36", face: null });
    expect(buttons().filter((b) => b.tabIndex === 0).map((b) => b.getAttribute("aria-label")?.slice(0, 8))).toEqual([ "Dente 36" ]);
    fireEvent.keyDown(document.activeElement!, { key: "Escape" });
    expect(onSelect).toHaveBeenLastCalledWith(null);
  });

  it("toque na face seleciona dente e face; selecionado fica pressionado; pendência marcada", () => {
    const onSelect = vi.fn();
    render(<Odontogram teeth={teeth} dentition="permanent" situations={SITUATIONS} pending={{ "26": 2 }} selected={{ tooth: "16", face: "O" }} onSelect={onSelect} />);
    expect(screen.getByRole("button", { name: /^Dente 16:/ }).getAttribute("aria-pressed")).toBe("true");
    expect(screen.getByRole("button", { name: /^Dente 26:.*2 marcações nesta consulta$/ })).not.toBeNull();
    const face = screen.getByRole("button", { name: /^Dente 46:/ }).querySelector('[data-face="V"]')!;
    fireEvent.pointerUp(face);
    expect(onSelect).toHaveBeenLastCalledWith({ tooth: "46", face: "V" });
  });

  it("alvo de toque de tablet e situação nunca só por cor", () => {
    render(<Odontogram teeth={teeth} dentition="permanent" situations={SITUATIONS} selected={null} onSelect={vi.fn()} />);
    for (const b of buttons()) {
      expect(parseInt(b.style.minWidth, 10)).toBeGreaterThanOrEqual(44);
      expect(parseInt(b.style.minHeight, 10)).toBeGreaterThanOrEqual(44);
    }
    expect(screen.getByRole("button", { name: /^Dente 36:/ }).textContent).toContain("AUS");
  });

  it("troca de dentição pelo controle", () => {
    const onDentitionChange = vi.fn();
    render(<Odontogram teeth={teeth} dentition="permanent" selected={null} onSelect={vi.fn()} onDentitionChange={onDentitionChange} />);
    fireEvent.click(screen.getByRole("button", { name: "Mista" }));
    expect(onDentitionChange).toHaveBeenCalledWith("mixed");
  });
});
```

(Se o `SegmentedControl` expõe as opções com outro papel — `radio`, `tab` —, ajuste o `getByRole` do último teste ao papel real; o comportamento é o mesmo.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/Odontogram.test.tsx`
Expected: FAIL — `./Odontogram` não existe.

- [ ] **Step 3: Implemente `Odontogram`**

Navegação: a matriz é `chartRows(dentition)`; `ArrowLeft`/`ArrowRight` andam na linha (sem dar a volta), `ArrowUp`/`ArrowDown` vão à linha de cima/baixo na mesma **posição relativa à linha média** (na mista, do 21 para o 61 e do 11 para o 51; dente que não existe na outra linha → o mais próximo da linha média do mesmo lado), `Home`/`End` ao começo/fim da linha. O dente com `tabIndex=0` é o selecionado, ou o último focado, ou o primeiro da primeira linha. `Enter`/`Espaço` chamam `onSelect({ tooth, face: null })`; `Escape` chama `onSelect(null)`. Cores do desenho só por `var(--…)` (via `toneColor` do tema, como o `Tag`), e o foco visível herdado (sem `outline: "none"`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/Odontogram.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (as guardas do tema também).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/dental/Odontogram.tsx src/modules/dental/Odontogram.test.tsx
$G commit -m "feat: add the accessible odontogram component

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Plano de tratamento ao lado, com a sugestão de "tratamento concluído" (F-28.4)

**Files:**
- Create: `src/modules/dental/TreatmentPlanPanel.tsx`
- Test: `src/modules/dental/TreatmentPlanPanel.test.tsx`

**Interfaces:**
- Consumes: `planItemsAfter`, `suggestCompleted`, `canAddPlanItems`, `withPlanCancel`, `withoutPlanChange`, `locationLabel`, `reasonProblem` (Task 2); `Tag`.
- Produces: `TreatmentPlanPanel({ plan: DentalPlan | null; value: DentalFields; unit: { kind: string }; role: DentalRole; readOnly?: boolean; onChange(f: DentalFields): void; onMarkCompleted(): void })` — `aside` com `aria-label="Plano de tratamento"`; sem plano: "Sem plano aberto nesta unidade." e, se `canAddPlanItems` for falso para o dentista na UBS, "O plano começa na primeira consulta programática."; com plano: cada item com SIGTAP, nome, `locationLabel` e um `Tag` ("planejado" `info`, "feito nesta consulta" `ok`, "cancelado nesta consulta" `warn`, "feito" `ok`, "cancelado" `neutral`); inclusões do rascunho ("novo — <nome> · <local>", com "Desfazer"); para o dentista, em item `planned` não ligado: "Cancelar item" abre o motivo (10–500, recusa antes) e "Confirmar cancelamento"; o auxiliar não vê "Cancelar item" nem inclusão; com `suggestCompleted` verdadeiro e sem "15" nos desfechos: `role="status"` "Todos os itens do plano estão feitos ou cancelados. Marque o desfecho \"Tratamento concluído\"." e o botão "Marcar tratamento concluído" (`onMarkCompleted`); com "15" já marcado: "Tratamento concluído marcado — o plano fecha ao finalizar.".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/dental/TreatmentPlanPanel.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { TreatmentPlanPanel } from "./TreatmentPlanPanel";
import { CEO_UNIT, FLUOR, RESIN, UBS, fields, plan, planItem, toothLoc } from "../../test/dentalFixtures";
import { MOUTH } from "../../lib/dental";

afterEach(cleanup);
const linked = fields({ procedures: [ { sigtap: RESIN.sigtap, location: toothLoc("16", "O"), quantity: 1, plan_item_id: "pi1" } ] });

describe("TreatmentPlanPanel", () => {
  it("itens com situação depois do rascunho e a sugestão de concluído", () => {
    const onMarkCompleted = vi.fn();
    render(<TreatmentPlanPanel plan={plan({ items: [ planItem(), planItem({ id: "pi2", status: "done", location: toothLoc("26") }) ] })} value={linked}
      unit={UBS} role="dentist" onChange={vi.fn()} onMarkCompleted={onMarkCompleted} />);
    expect(screen.getByText("dente 16, face oclusal/incisal")).not.toBeNull();
    expect(screen.getByText("feito nesta consulta")).not.toBeNull();
    expect(screen.getByText("feito")).not.toBeNull();
    expect(screen.getByRole("status").textContent).toBe("Todos os itens do plano estão feitos ou cancelados. Marque o desfecho \"Tratamento concluído\".");
    fireEvent.click(screen.getByRole("button", { name: "Marcar tratamento concluído" }));
    expect(onMarkCompleted).toHaveBeenCalled();
  });

  it("com 15 marcado, só o aviso; com item planejado, nenhuma sugestão", () => {
    const { rerender } = render(<TreatmentPlanPanel plan={plan()} value={{ ...linked, dental_outcomes: [ "15" ] }} unit={UBS} role="dentist"
      onChange={vi.fn()} onMarkCompleted={vi.fn()} />);
    expect(screen.getByText("Tratamento concluído marcado — o plano fecha ao finalizar.")).not.toBeNull();
    rerender(<TreatmentPlanPanel plan={plan()} value={fields()} unit={UBS} role="dentist" onChange={vi.fn()} onMarkCompleted={vi.fn()} />);
    expect(screen.queryByRole("button", { name: "Marcar tratamento concluído" })).toBeNull();
  });

  it("cancelar item pede motivo de 10 a 500 e entra no rascunho", () => {
    const onChange = vi.fn();
    render(<TreatmentPlanPanel plan={plan()} value={fields()} unit={UBS} role="dentist" onChange={onChange} onMarkCompleted={vi.fn()} />);
    fireEvent.click(screen.getByRole("button", { name: `Cancelar item — ${RESIN.name}` }));
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "curto" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
    expect(screen.getByText("o motivo tem de 10 a 500 caracteres")).not.toBeNull();
    expect(onChange).not.toHaveBeenCalled();
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "dente indicado para extração" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
    expect(onChange).toHaveBeenCalledWith(fields({ plan_changes: { add: [], cancel: [ { item_id: "pi1", reason: "dente indicado para extração" } ] } }));
  });

  it("auxiliar não cancela nem inclui; inclusão do rascunho desfaz", () => {
    const onChange = vi.fn();
    const withAdd = fields({ plan_changes: { add: [ { sigtap: FLUOR.sigtap, name: FLUOR.name, location: MOUTH } ], cancel: [] } });
    const { rerender } = render(<TreatmentPlanPanel plan={plan()} value={fields()} unit={UBS} role="auxiliary" onChange={onChange} onMarkCompleted={vi.fn()} />);
    expect(screen.queryByRole("button", { name: /^Cancelar item/ })).toBeNull();
    rerender(<TreatmentPlanPanel plan={plan()} value={withAdd} unit={UBS} role="dentist" onChange={onChange} onMarkCompleted={vi.fn()} />);
    expect(screen.getByText(`novo — ${FLUOR.name} · boca toda`)).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: `Desfazer — ${FLUOR.name}` }));
    expect(onChange).toHaveBeenCalledWith(fields());
  });

  it("sem plano: na UBS explica onde o plano começa; no CEO, não", () => {
    const { rerender } = render(<TreatmentPlanPanel plan={null} value={fields({ dental_consultation_type: "2" })} unit={UBS} role="dentist" onChange={vi.fn()} onMarkCompleted={vi.fn()} />);
    expect(screen.getByText("Sem plano aberto nesta unidade.")).not.toBeNull();
    expect(screen.getByText("O plano começa na primeira consulta programática.")).not.toBeNull();
    rerender(<TreatmentPlanPanel plan={null} value={fields()} unit={CEO_UNIT} role="dentist" onChange={vi.fn()} onMarkCompleted={vi.fn()} />);
    expect(screen.queryByText("O plano começa na primeira consulta programática.")).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/TreatmentPlanPanel.test.tsx`
Expected: FAIL — o componente não existe.

- [ ] **Step 3: Implemente `TreatmentPlanPanel`** (a inclusão de item nasce no `ToothActions`, Task 6; aqui só aparece e se desfaz)

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/TreatmentPlanPanel.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/dental/TreatmentPlanPanel.tsx src/modules/dental/TreatmentPlanPanel.test.tsx
$G commit -m "feat: add the treatment plan panel with the completed treatment suggestion

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Bloco odontológico — achados, procedimentos, local e campos da FAO (F-28.2, F-28.3)

**Files:**
- Create: `src/modules/dental/ProcedureSearch.tsx`, `src/modules/dental/ToothActions.tsx`, `src/modules/dental/FaoFields.tsx`, `src/modules/dental/DentalEditor.tsx`
- Test: `src/modules/dental/DentalEditor.test.tsx`

**Interfaces:**
- Consumes: `getDentalChart`, `listDentalPlans`, `searchDentalProcedures` (Task 1); `withFinding`, `withoutFinding`, `withProcedure`, `withoutProcedure`, `linkPlanItem`, `withPlanAdd`, `canAddPlanItems`, `situationsFor`, `locationLabel`, `sextantLoc`, `archLoc`, `MOUTH`, `quantityProblems`, `CARE_TYPES`, `CONSULTATION_TYPES`, `SURVEILLANCE`, `SUPPLIES`, `OUTCOMES`, `REFERRAL_OUTCOMES`, `toggleSurveillance`, `pregnantFor`, `DENTAL_CHART_KEY`, `DENTAL_PLANS_KEY`, `NOMINAL_QUERY`, `dentalError` (Task 2); `Odontogram` (Task 4); `TreatmentPlanPanel` (Task 5).
- Produces:
  - `ProcedureSearch({ role: DentalRole; only?: "prosthesis"; delayMs?: number; onPick(ref: DentalProcedureRef): void })` — campo "Procedimento (SIGTAP)" que busca a partir de 3 letras, depois de `delayMs` (padrão 300), pelo `searchDentalProcedures`; o auxiliar só vê `auxiliary_allowed`; `only: "prosthesis"` só `prosthesis`; resultado é lista de botões "<sigtap> — <nome>".
  - `ToothActions({ selection: { tooth: string; face: ToothFace | null } | null; value: DentalFields; plan: DentalPlan | null; unit; role; baseline: boolean; situations?: DentalSituation[]; searchDelayMs?; onSelectFace(face: ToothFace | null): void; onChange(f: DentalFields): void })` — `section` "Dente selecionado"; faces como grupo de 5 botões "Face mesial" … com `aria-pressed` e "Dente inteiro"; "Achado" (`select` com `situationsFor` do nível — dente ou face — e "Marcar achado"); fora do registro inicial: os botões de local "Sextante S1".."S6" (com o sextante do dente pré-escolhido), "Arcada superior", "Arcada inferior", "Boca toda"; `ProcedureSearch`; escolhido o procedimento, o local tem de estar em `ref.locations` (senão a frase "<nome>: escolha um local do tipo <tipos>" e nada entra); "Quantidade" (inteiro, padrão 1); "Item do plano" (`select` com os itens `planned` de mesmo SIGTAP; o auxiliar só pode ligar item cujo SIGTAP a busca devolveu como `auxiliary_allowed`); "Registrar procedimento" e, para o dentista com `canAddPlanItems`, "Incluir no plano".
  - `FaoFields({ value: DentalFields; sex: Sex; role: DentalRole; prefilledPregnant: boolean; onChange(f: DentalFields): void })` — `fieldset` "Ficha odontológica (e-SUS)": "Tipo de atendimento" (`select`); "Tipo de consulta odontológica" (`select` com "—"; escolher o tipo de atendimento "4" limpa o valor e desabilita o campo); "Vigilância em saúde bucal" (caixas, `toggleSurveillance`); "Fornecimento" (caixas); "Desfecho" (caixas; para o auxiliar, 1–11 não aparecem); "Gestante" (rádios Sim/Não, **só** no feminino; com `prefilledPregnant` e valor `true`: "pré-preenchido pela linha de pré-natal — confirme"); "Paciente com necessidades especiais" (caixa).
  - `DentalEditor({ consultation: Consultation; record: ClinicalRecord; unit: HealthUnit; options: ConsultationOptions; value: DentalFields; role: DentalRole; baseline: boolean; planRefreshKey?: number; searchDelayMs?: number; onChange(f: DentalFields): void; onPlanRead?(plan: DentalPlan | null): void })` — lê o odontograma (`[DENTAL_CHART_KEY, patientId]`) e o plano aberto da unidade (`[DENTAL_PLANS_KEY, patientId, unit.id, planRefreshKey]`) com `NOMINAL_QUERY`; grade `repeat(auto-fit, minmax(320px, 1fr))`: à esquerda o `Odontogram` (dentição inicial do api; `pending` = achados e procedimentos do rascunho por dente) com o `ToothActions` embaixo; à direita o `TreatmentPlanPanel` (fora do registro inicial); abaixo, "Procedimentos desta consulta" (lista com local, quantidade, item do plano e "Remover") e "Achados desta consulta" (com "Remover"); as recusas de `quantityProblems` em `role="alert"`; fora do registro inicial, o `FaoFields`. O registro inicial mostra só odontograma, achados e a frase "Registro inicial: marque a situação de cada dente. Não gera ficha do e-SUS.".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/dental/DentalEditor.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { useState } from "react";
vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  getDentalChart: vi.fn(), listDentalPlans: vi.fn(), searchDentalProcedures: vi.fn() }));
import * as api from "../../lib/api";
import type { DentalFields, DentalRole, Sex } from "../../lib/api";
import { DentalEditor } from "./DentalEditor";
import { renderWithProviders } from "../../test/campaignFixtures";
import { record } from "../../test/consultationFixtures";
import { FLUOR, RESIN, SCALING, UBS, dentalDraft, dentalOptions, dentalUser, fields, odontogram, plan } from "../../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let last: DentalFields | null = null;

function Harness({ role = "dentist", baseline = false, initial = fields(), sex = "female" }:
  { role?: DentalRole; baseline?: boolean; initial?: DentalFields; sex?: Sex }) {
  const [ value, setValue ] = useState(initial);
  return <DentalEditor consultation={dentalDraft()} record={record({ patient: { ...record().patient, sex } })} unit={UBS} options={dentalOptions()}
    value={value} role={role} baseline={baseline} searchDelayMs={0} onChange={(f) => { last = f; setValue(f); }} />;
}

describe("DentalEditor", () => {
  beforeEach(() => {
    last = null;
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser());
    mocked(api.getDentalChart).mockResolvedValue(odontogram());
    mocked(api.listDentalPlans).mockResolvedValue([ plan() ]);
    mocked(api.searchDentalProcedures).mockResolvedValue([ RESIN, SCALING, FLUOR ]);
  });

  it("lê odontograma e plano uma vez, pelo paciente e pela unidade", async () => {
    renderWithProviders(<Harness />);
    expect(await screen.findByRole("button", { name: /^Dente 16:/ })).not.toBeNull();
    expect(mocked(api.listDentalPlans)).toHaveBeenCalledWith("pa1", { healthUnitId: "u1", status: "open" });
    expect(mocked(api.getDentalChart)).toHaveBeenCalledTimes(1);
  });

  it("achado na face e procedimento ligado ao plano entram no rascunho", async () => {
    renderWithProviders(<Harness />);
    fireEvent.click(await screen.findByRole("button", { name: /^Dente 16:/ }));
    fireEvent.click(screen.getByRole("button", { name: "Face oclusal/incisal" }));
    fireEvent.change(screen.getByLabelText("Achado"), { target: { value: "caries" } });
    fireEvent.click(screen.getByRole("button", { name: "Marcar achado" }));
    expect(last!.findings).toEqual([ { tooth: "16", face: "O", situation: "caries" } ]);
    fireEvent.change(screen.getByLabelText("Procedimento (SIGTAP)"), { target: { value: "resina" } });
    fireEvent.click(await screen.findByRole("button", { name: `${RESIN.sigtap} — ${RESIN.name}` }));
    fireEvent.change(screen.getByLabelText("Item do plano"), { target: { value: "pi1" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar procedimento" }));
    expect(last!.procedures).toEqual([ { sigtap: RESIN.sigtap, name: RESIN.name, location: { kind: "face", tooth: "16", face: "O", sextant: null, arch: null },
      quantity: 1, plan_item_id: "pi1" } ]);
    expect(screen.getByRole("button", { name: /^Dente 16:.*2 marcações nesta consulta$/ })).not.toBeNull();
  });

  it("local fora do tipo do procedimento é recusado; sextante e boca toda", async () => {
    renderWithProviders(<Harness />);
    fireEvent.click(await screen.findByRole("button", { name: /^Dente 16:/ }));
    fireEvent.change(screen.getByLabelText("Procedimento (SIGTAP)"), { target: { value: "raspagem" } });
    fireEvent.click(await screen.findByRole("button", { name: `${SCALING.sigtap} — ${SCALING.name}` }));
    fireEvent.click(screen.getByRole("button", { name: "Registrar procedimento" }));
    expect(screen.getByText(`${SCALING.name}: escolha um local do tipo sextante`)).not.toBeNull();
    expect(last).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Sextante S1" }));
    fireEvent.click(screen.getByRole("button", { name: "Registrar procedimento" }));
    expect(last!.procedures[0].location).toEqual({ kind: "sextant", tooth: null, face: null, sextant: "S1", arch: null });
  });

  it("FAO: tipo de consulta some na escuta; 99 exclui os demais", async () => {
    renderWithProviders(<Harness />);
    await screen.findByRole("button", { name: /^Dente 16:/ });
    fireEvent.change(screen.getByLabelText("Tipo de atendimento"), { target: { value: "4" } });
    expect((screen.getByLabelText("Tipo de consulta odontológica") as HTMLSelectElement).disabled).toBe(true);
    expect(last!.dental_consultation_type).toBeNull();
    fireEvent.click(screen.getByLabelText("Dor de dente"));
    expect(last!.oral_health_surveillance).toEqual([ "3" ]);
  });

  it("gestante pré-preenchida pede confirmação; masculino sem o campo", async () => {
    const { unmount } = renderWithProviders(<Harness initial={fields({ pregnant: true })} />);
    expect(await screen.findByText("pré-preenchido pela linha de pré-natal — confirme")).not.toBeNull();
    unmount();
    renderWithProviders(<Harness initial={fields({ pregnant: true })} sex="male" />);
    await screen.findByRole("button", { name: /^Dente 16:/ });
    expect(screen.queryByRole("group", { name: "Gestante" })).toBeNull();
    expect(screen.getByText("Gestante não se aplica: vai em branco na ficha.")).not.toBeNull();
  });

  it("auxiliar: sem desfechos de encaminhamento, sem incluir no plano, busca só da lista", async () => {
    mocked(api.searchDentalProcedures).mockResolvedValue([ RESIN, FLUOR ]);
    renderWithProviders(<Harness role="auxiliary" />);
    fireEvent.click(await screen.findByRole("button", { name: /^Dente 16:/ }));
    expect(screen.queryByLabelText("Encaminhamento: endodontia")).toBeNull();
    fireEvent.change(screen.getByLabelText("Procedimento (SIGTAP)"), { target: { value: "flu" } });
    expect(await screen.findByRole("button", { name: `${FLUOR.sigtap} — ${FLUOR.name}` })).not.toBeNull();
    expect(screen.queryByRole("button", { name: `${RESIN.sigtap} — ${RESIN.name}` })).toBeNull();
    expect(screen.queryByRole("button", { name: "Incluir no plano" })).toBeNull();
  });

  it("registro inicial: só achados; sem FAO, plano nem procedimento", async () => {
    renderWithProviders(<Harness baseline initial={fields({ care_type: null, dental_consultation_type: null, oral_health_surveillance: [], dental_outcomes: [], pregnant: null })} />);
    fireEvent.click(await screen.findByRole("button", { name: /^Dente 36:/ }));
    expect(screen.getByText("Registro inicial: marque a situação de cada dente. Não gera ficha do e-SUS.")).not.toBeNull();
    expect(screen.queryByLabelText("Procedimento (SIGTAP)")).toBeNull();
    expect(screen.queryByRole("complementary", { name: "Plano de tratamento" })).toBeNull();
    expect(screen.queryByLabelText("Tipo de atendimento")).toBeNull();
    await waitFor(() => expect(mocked(api.listDentalPlans)).not.toHaveBeenCalled());
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/DentalEditor.test.tsx`
Expected: FAIL — os componentes não existem.

- [ ] **Step 3: Implemente as quatro peças**

`DentalEditor` não lê o plano no registro inicial (`enabled: !baseline`) e chama `onPlanRead` quando o plano chega (a Task 7 usa para o `suggestCompleted` e o `plan_item_not_open`). Erro da leitura do odontograma ou do plano → `dentalError` em `role="alert"`, sem esconder o resto (o rascunho continua editável). O `FaoFields` com `sex === "male"` mostra "Gestante não se aplica: vai em branco na ficha." e chama `onChange` com `pregnant: null` uma vez, se veio outro valor. Rótulos dos desfechos de 1 a 11: "Encaminhamento: <especialidade em minúsculas>" (ex.: "Encaminhamento: endodontia").

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/dental/ProcedureSearch.tsx src/modules/dental/ToothActions.tsx src/modules/dental/FaoFields.tsx src/modules/dental/DentalEditor.tsx src/modules/dental/DentalEditor.test.tsx
$G commit -m "feat: add the dental consultation block with findings, procedures and the FAO fields

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Consulta odontológica na tela da consulta do 19a (F-28.2, F-28.4, F-28.7)

**Files:**
- Modify: `src/lib/consultation.ts`, `src/lib/consultation.test.ts`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/consultation/ConsultationWorkspace.dental.test.tsx`

**Interfaces:**
- Consumes: `startConsultation(id, input?)`, `saveConsultationDraft`, `finalizeConsultation` (Task 1); `offeredKinds`, `dentalRole`, `dentalFieldsFrom`, `dentalPayload`, `dentalFinalizeProblems`, `dentalApiProblems`, `planItemPathIndex`, `linkPlanItem`, `suggestCompleted`, `dentalError`, `canUseDental` (Task 2); `DentalEditor` (Task 6).
- Produces:
  - em `src/lib/consultation.ts`: `ConsultationDraft` ganha `dental: DentalFields | null`; `draftFrom(c)` preenche com `c.kind === "dental" ? dentalFieldsFrom(c) : null`; `checkDraft(d)` põe `dental: dentalPayload(d.dental)` no `input` só quando há bloco (chave ausente na consulta comum — o corpo de hoje não muda); `finalizeProblems(check, mode: "general" | "dental" | "baseline" = "general")` — `dental` não exige conduta; `baseline` não exige problema, conduta nem A/P (D12);
  - `ConsultationEditor` ganha a prop `dental?: { role: DentalRole; baseline: boolean; sex: Sex }`: com ela, some o "Tipo de atendimento" do 19a e o `ConductsField`; o `DentalEditor` entra depois dos problemas (no registro inicial, também somem problemas e exames); "o que falta para finalizar" junta `finalizeProblems(check, mode)` e `dentalFinalizeProblems(...)` (textos); a recusa da finalização com `problems` vira a lista `aria-label="recusas do api"` (todas, traduzidas); `plan_item_not_open` com caminho de procedimento relê o plano (`planRefreshKey + 1`) e chama `linkPlanItem(f, i, null)` no rascunho; nada é reenviado sozinho;
  - `ConsultationWorkspace`: com `offeredKinds(options.data, canUseDental(user))` contendo `dental`, os botões "Iniciar consulta odontológica" e — com `options.dental_role === "dentist"` — "Registro inicial do odontograma"; "Iniciar consulta" (comum) aparece enquanto o `options` carrega ou se `general` for oferecido; a recusa do início vira `dentalError`; para a consulta `dental`: aba Documentos só para o dentista e fora do registro inicial; `regulationOn` falso no registro inicial (o auxiliar já não tem `allowedTypes`); o `ProceduresPanel` do 22 nunca aparece (o 28 não gera BPA — spec §9).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/consultation/ConsultationWorkspace.dental.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getAttendanceRecord: vi.fn(), getConsultationOptions: vi.fn(), startConsultation: vi.fn(),
    getConsultation: vi.fn(), saveConsultationDraft: vi.fn(), finalizeConsultation: vi.fn(), getConsultationProblems: vi.fn(),
    getDentalChart: vi.fn(), listDentalPlans: vi.fn(), searchDentalProcedures: vi.fn(), listConsultationRegulationRequests: vi.fn() };
});
// O painel do 22 aparece em unidade que fatura (o CEO fatura; a UBS não): aqui ele só se anuncia.
vi.mock("../billing/ProceduresPanel", () => ({ ProceduresPanel: () => <span>painel do faturamento</span> }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ConsultationWorkspace } from "./ConsultationWorkspace";
import { renderWithProviders } from "../../test/campaignFixtures";
import { options, problem, record } from "../../test/consultationFixtures";
import { CEO_UNIT, RESIN, UBS, auxOptions, dentalDraft, dentalFinalized, dentalOptions, dentalUser, fields, odontogram, plan, toothLoc } from "../../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const withProblem = { evaluated_problems: [ { problem_id: "pp1", terminology: "ciap2" as const, code: "D82", label: "Doença dos dentes/gengivas", action: "evaluate" as const } ] };

function open(features = [ "clinical_record", "clinical_documents", "dental_care", "care_regulation" ], opts = dentalOptions(),
  unit: { id: string; name: string; kind: string } = UBS) {
  mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser([ "health_professional" ], { features }));
  mocked(api.getConsultationOptions).mockResolvedValue(opts);
  renderWithProviders(<ConsultationWorkspace attendanceId="a1" unit={unit} units={[ unit ]} autosaveDelayMs={0} searchDelayMs={0}
    onClose={() => undefined} onFinalized={() => undefined} />);
}

describe("ConsultationWorkspace — consulta odontológica (módulo 28)", () => {
  beforeEach(() => {
    for (const fn of [ api.startConsultation, api.finalizeConsultation, api.saveConsultationDraft ]) mocked(fn).mockReset();
    mocked(api.getAttendanceRecord).mockResolvedValue(record({ problems: [ problem() ] }));
    mocked(api.getConsultationProblems).mockResolvedValue([ problem() ]);
    mocked(api.getDentalChart).mockResolvedValue(odontogram());
    mocked(api.listDentalPlans).mockResolvedValue([ plan() ]);
    mocked(api.searchDentalProcedures).mockResolvedValue([ RESIN ]);
    mocked(api.listConsultationRegulationRequests).mockResolvedValue([]);
    mocked(api.saveConsultationDraft).mockImplementation((_id: string, input: unknown) => Promise.resolve({ ...dentalDraft(), ...(input as object) }));
  });

  it("dentista: inicia odontológica e registro inicial; o tipo vai no corpo", async () => {
    mocked(api.startConsultation).mockResolvedValue(dentalDraft());
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta odontológica" }));
    await waitFor(() => expect(mocked(api.startConsultation)).toHaveBeenCalledWith("a1", { kind: "dental" }));
    expect(await screen.findByRole("group", { name: "Odontograma" })).not.toBeNull();
    // O "Tipo de atendimento" do 19a some; fica só o da ficha odontológica.
    expect(screen.getAllByLabelText("Tipo de atendimento")).toHaveLength(1);
    cleanup();
    mocked(api.startConsultation).mockResolvedValue(dentalDraft({ baseline: true, dental: fields({ care_type: null, dental_consultation_type: null,
      oral_health_surveillance: [], dental_outcomes: [], pregnant: null }) }));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Registro inicial do odontograma" }));
    await waitFor(() => expect(mocked(api.startConsultation)).toHaveBeenLastCalledWith("a1", { kind: "dental", baseline: true }));
  });

  it("médico: só 'Iniciar consulta', sem argumento; sem dental_care o dentista não vê o tipo", async () => {
    mocked(api.startConsultation).mockResolvedValue(dentalDraft({ kind: "general", cbo_code: "225142", dental: null }));
    open(undefined, options());
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta" }));
    await waitFor(() => expect(mocked(api.startConsultation)).toHaveBeenCalledWith("a1"));
    cleanup();
    open([ "clinical_record" ], dentalOptions());
    await screen.findByText("Se você já começou esta consulta, o seu rascunho abre.");
    expect(screen.queryByRole("button", { name: "Iniciar consulta odontológica" })).toBeNull();
  });

  it("recusa do início vira frase", async () => {
    mocked(api.startConsultation).mockRejectedValue(new ApiError(403, { error: "cbo_not_allowed" }, "x"));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta odontológica" }));
    expect(await screen.findByText("seu CBO não registra consulta odontológica")).not.toBeNull();
  });

  it("auxiliar: sem documentos, regulação, registro inicial, encaminhamento nem plano", async () => {
    mocked(api.startConsultation).mockResolvedValue(dentalDraft({ cbo_code: "322405" }));
    open(undefined, auxOptions());
    await screen.findByRole("button", { name: "Iniciar consulta odontológica" });
    expect(screen.queryByRole("button", { name: "Registro inicial do odontograma" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Iniciar consulta odontológica" }));
    await screen.findByRole("group", { name: "Odontograma" });
    expect(screen.queryByRole("button", { name: "Documentos" })).toBeNull();
    expect(screen.queryByRole("region", { name: "Pedir regulação" })).toBeNull();
    expect(screen.queryByLabelText("Encaminhamento: endodontia")).toBeNull();
    expect(screen.queryByRole("button", { name: /^Cancelar item/ })).toBeNull();
  });

  it("finalizar: o que falta antes; recusas do api todas de uma vez", async () => {
    mocked(api.startConsultation).mockResolvedValue(dentalDraft({ dental: fields({ oral_health_surveillance: [], pregnant: null }) }));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta odontológica" }));
    fireEvent.click(await screen.findByRole("button", { name: "Finalizar consulta" }));
    const missing = screen.getByRole("list", { name: "o que falta para finalizar" });
    expect(missing.textContent).toContain("marque a vigilância em saúde bucal");
    expect(missing.textContent).toContain("informe se a paciente está gestante");
    expect(missing.textContent).toContain("avalie, inclua ou resolva ao menos um problema");
    expect(missing.textContent).not.toContain("marque ao menos uma conduta");
    expect(mocked(api.finalizeConsultation)).not.toHaveBeenCalled();
  });

  it("plan_item_not_open: relê o plano, desfaz o vínculo e mantém o procedimento", async () => {
    const draft = dentalDraft({ ...withProblem, assessment: "Cárie oclusal no 16.",
      dental: fields({ procedures: [ { id: "dpr1", sigtap: RESIN.sigtap, name: RESIN.name, location: toothLoc("16", "O"), quantity: 1, plan_item_id: "pi1" } ] }) });
    mocked(api.startConsultation).mockResolvedValue(draft);
    mocked(api.finalizeConsultation).mockRejectedValue(new ApiError(422, { error: "plan_item_not_open",
      problems: [ { path: "dental.procedures[0].plan_item_id", error: "plan_item_not_open" }, { path: "dental.pregnant", error: "pregnant_required" } ] }, "x"));
    open();
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta odontológica" }));
    fireEvent.click(await screen.findByRole("button", { name: "Finalizar consulta" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar finalização" }));
    const refusals = await screen.findByRole("list", { name: "recusas do api" });
    expect(refusals.textContent).toContain("um item do plano já foi feito ou cancelado em outra consulta");
    expect(refusals.textContent).toContain("informe se a paciente está gestante");
    await waitFor(() => expect(mocked(api.listDentalPlans)).toHaveBeenCalledTimes(2));
    await waitFor(() => expect(mocked(api.saveConsultationDraft)).toHaveBeenLastCalledWith("cs1", expect.objectContaining({
      dental: expect.objectContaining({ procedures: [ { id: "dpr1", sigtap: RESIN.sigtap, location: toothLoc("16", "O"), quantity: 1, plan_item_id: null } ] }) })));
    expect(mocked(api.finalizeConsultation)).toHaveBeenCalledTimes(1);
  });

  it("consulta odontológica finalizada do dentista no CEO: aba Documentos; nunca o painel do faturamento", async () => {
    mocked(api.startConsultation).mockResolvedValue(dentalFinalized());
    open([ "clinical_record", "clinical_documents", "dental_care", "procedure_billing" ], dentalOptions(), CEO_UNIT);
    fireEvent.click(await screen.findByRole("button", { name: "Iniciar consulta odontológica" }));
    expect(await screen.findByRole("button", { name: "Documentos" })).not.toBeNull();
    expect(screen.queryByText("painel do faturamento")).toBeNull();
  });
});
```

E em `src/lib/consultation.test.ts`:

```ts
describe("módulo 28 — rascunho odontológico", () => {
  it("consulta comum: corpo igual ao de antes; odontológica leva dental sem nome", () => {
    expect("dental" in checkDraft(draftFrom(consultation())).input).toBe(false);
    const d = draftFrom(dentalDraft({ dental: { ...fields(), suggest_completed: true,
      procedures: [ { sigtap: RESIN.sigtap, name: RESIN.name, location: toothLoc("16"), quantity: 1, plan_item_id: null } ] } }));
    expect(checkDraft(d).input.dental).toEqual(fields({ procedures: [ { sigtap: RESIN.sigtap, location: toothLoc("16"), quantity: 1, plan_item_id: null } ] }));
  });
  it("modos da finalização: odontológica sem conduta; registro inicial sem problema, conduta nem A/P", () => {
    const check = checkDraft(draftFrom(consultation()));
    expect(finalizeProblems(check, "dental")).toEqual([ "avalie, inclua ou resolva ao menos um problema", "escreva a avaliação (A) ou o plano (P)" ]);
    expect(finalizeProblems(check, "baseline")).toEqual([]);
  });
});
```

(com `dentalDraft`, `fields`, `RESIN`, `toothLoc` importados de `../test/dentalFixtures`.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/consultation/ConsultationWorkspace.dental.test.tsx src/lib/consultation.test.ts`
Expected: FAIL — o tipo não é escolhido, `finalizeProblems` não tem modo, o editor não monta o bloco.

- [ ] **Step 3: Implemente**

`ConsultationWorkspace`: `open(input?)` — `startConsultation(props.attendanceId)` sem argumento na consulta comum (os testes antigos conferem a chamada com um argumento só) e com `{ kind: "dental", baseline? }` nos botões novos; a frase da recusa vem de `dentalError` quando o início é odontológico. O `ConsultationEditor` recebe `dental={ consultation.kind === "dental" ? { role: dentalRole(consultation.cbo_code) ?? "auxiliary", baseline: !!consultation.baseline, sex: record.data.patient.sex } : undefined }`. `ConsultationEditor`: o estado `planRefreshKey` e o `plan` lido (via `onPlanRead`) ficam no editor; `onMarkCompleted` acrescenta "15" a `dental_outcomes`; o `catch` da finalização, para consulta `dental`, guarda `dentalApiProblems(err)` (lista) em vez da frase única, e para cada `plan_item_not_open` com `planItemPathIndex` aplica `linkPlanItem(…, null)` e incrementa `planRefreshKey`; `regulation_request_required` segue o caminho do 26a (`regulationFinalizeError`, releitura da peça). `proceduresOn` passa a `canUseBilling(user) && unitBills(props.unit) && consultation?.kind !== "dental"` (confira em `unitBills` que a unidade `ceo` fatura; se não faturar, o teste do CEO continua valendo). Nada muda na consulta comum (os testes do 19a continuam verdes).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/consultation src/lib/consultation.test.ts src/lib/consultation.priority.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS — inclusive todos os `ConsultationWorkspace.*.test.tsx`, `ConsultationEditor.test.tsx` e `StructuredFields.test.tsx` existentes.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/consultation.ts src/lib/consultation.test.ts src/modules/consultation/ConsultationEditor.tsx src/modules/consultation/ConsultationWorkspace.tsx \
  src/modules/consultation/ConsultationWorkspace.dental.test.tsx
$G commit -m "feat: add the dental kind to the consultation screen with the finalize refusals

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Consulta odontológica finalizada, adendo e histórico do dente (F-28.2, F-28.3)

**Files:**
- Create: `src/modules/dental/DentalSummary.tsx`, `src/modules/dental/DentalAddendumFields.tsx`, `src/modules/dental/ToothHistory.tsx`
- Modify: `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/AddendumForm.tsx`, `src/modules/dental/DentalEditor.tsx` (o `ToothHistory` no painel do dente)
- Test: `src/modules/dental/DentalReading.test.tsx`

**Interfaces:**
- Consumes: `listChartEvents`, `listDentalPlans`, `addAddendum` (Task 1 e existente); `locationLabel`, `situationLabel`, `CARE_TYPES`, `CONSULTATION_TYPES`, `SURVEILLANCE`, `SUPPLIES`, `OUTCOMES`, `withFinding`, `reasonProblem`, `CHART_EVENTS_KEY`, `NOMINAL_QUERY`, `dentalError` (Task 2).
- Produces:
  - `DentalSummary({ consultation: Consultation; situations?: DentalSituation[] })` — só leitura, a partir de `consultation.dental` (nunca relê o odontograma): "Registro inicial (sem ficha do e-SUS)" quando `baseline`; os campos da FAO com rótulo (código desconhecido cru); "Procedimentos" (`<nome> · <local> · <quantidade>`), "Achados" (`<local>: <situação>`), mudanças do plano, periograma, registro periodontal e endodontia, quando houver. Aparece no `ConsultationView` de toda consulta `kind: "dental"` — também com `dental_care` desligado (registro retido).
  - `DentalAddendumFields({ consultation: Consultation; value: { findings: DentalFinding[]; cancel: PlanCancel[] }; onChange(v): void })` — no adendo da consulta odontológica: "Acrescentar achado" (dente, face opcional, situação) e, só para o dentista, "Cancelar item do plano" (itens `planned` do plano da unidade, lido no clique "Ver plano", com motivo 10–500). Nada de procedimento novo (contrato §4). O `AddendumForm` manda `changes.dental = { findings, plan_changes: { cancel } }` (D13) quando há algo.
  - `ToothHistory({ patientId: string; tooth: string })` — botão "Histórico do dente <n>"; no clique lê `listChartEvents(patientId, { tooth })` (`NOMINAL_QUERY`, nunca sozinho), lista "<data> · <situação ou procedimento> · <face> · <quem> (<CBO>) · consulta|adendo" e "Ver mais" pelo `next_before`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/dental/DentalReading.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  listChartEvents: vi.fn(), listDentalPlans: vi.fn(), addAddendum: vi.fn(), getConsultationProblems: vi.fn() }));
import * as api from "../../lib/api";
import { DentalSummary } from "./DentalSummary";
import { ToothHistory } from "./ToothHistory";
import { AddendumForm } from "../consultation/AddendumForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { RESIN, SITUATIONS, chartEvent, dentalFinalized, dentalOptions, dentalUser, fields, plan } from "../../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("leitura odontológica", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser());
    mocked(api.listChartEvents).mockReset();
    mocked(api.addAddendum).mockResolvedValue({ id: "ad1", author_name: "Dr. Caio Nunes", created_at: "2026-10-30T11:00:00-03:00", reason: "", text: "", changes: null });
  });

  it("resumo da finalizada com rótulos; registro inicial sem ficha", () => {
    const { rerender } = renderWithProviders(<DentalSummary consultation={dentalFinalized()} situations={SITUATIONS} />);
    expect(screen.getByText("Consulta agendada")).not.toBeNull();
    expect(screen.getByText("Primeira consulta odontológica programática")).not.toBeNull();
    expect(screen.getByText("Não identificado")).not.toBeNull();
    expect(screen.getByText(`${RESIN.name} · dente 16, face oclusal/incisal · 1`)).not.toBeNull();
    rerender(<DentalSummary consultation={dentalFinalized({ baseline: true, dental: fields({ care_type: null, dental_consultation_type: null,
      dental_outcomes: [], oral_health_surveillance: [], findings: [ { tooth: "36", face: null, situation: "missing" } ] }) })} situations={SITUATIONS} />);
    expect(screen.getByText("Registro inicial (sem ficha do e-SUS)")).not.toBeNull();
    expect(screen.getByText("dente 36: Ausente")).not.toBeNull();
  });

  it("histórico do dente só no clique, com paginação", async () => {
    mocked(api.listChartEvents).mockResolvedValueOnce({ events: [ chartEvent() ], next_before: "c1" })
      .mockResolvedValueOnce({ events: [ chartEvent({ id: "ev0", situation: null, procedure: { sigtap: RESIN.sigtap, name: RESIN.name }, addendum_id: "ad9" }) ], next_before: null });
    renderWithProviders(<ToothHistory patientId="pa1" tooth="16" />);
    expect(mocked(api.listChartEvents)).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Histórico do dente 16" }));
    expect(await screen.findByText(/caries|Cariado/)).not.toBeNull();
    expect(mocked(api.listChartEvents)).toHaveBeenCalledWith("pa1", { tooth: "16", before: null });
    fireEvent.click(screen.getByRole("button", { name: "Ver mais" }));
    expect(await screen.findByText(new RegExp(`${RESIN.name}.*adendo`))).not.toBeNull();
    expect(mocked(api.listChartEvents)).toHaveBeenLastCalledWith("pa1", { tooth: "16", before: "c1" });
  });

  it("adendo odontológico: achado e cancelamento de item, sem procedimento", async () => {
    mocked(api.listDentalPlans).mockResolvedValue([ plan() ]);
    renderWithProviders(<AddendumForm consultation={dentalFinalized()} options={dentalOptions()} patientProblems={[]} onDone={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.change(screen.getByLabelText("Motivo do adendo"), { target: { value: "achado depois da radiografia" } });
    fireEvent.change(screen.getByLabelText("Texto do adendo"), { target: { value: "Cárie distal no 26 vista na radiografia." } });
    fireEvent.click(screen.getByLabelText("Mudar o odontograma ou o plano"));
    expect(screen.queryByLabelText("Procedimento (SIGTAP)")).toBeNull();
    fireEvent.change(screen.getByLabelText("Dente"), { target: { value: "26" } });
    fireEvent.change(screen.getByLabelText("Face"), { target: { value: "D" } });
    fireEvent.change(screen.getByLabelText("Situação"), { target: { value: "caries" } });
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar achado" }));
    fireEvent.click(screen.getByRole("button", { name: "Ver plano" }));
    fireEvent.click(await screen.findByRole("button", { name: `Cancelar item — ${RESIN.name}` }));
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "dente indicado para extração" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
    fireEvent.click(screen.getByRole("button", { name: "Registrar adendo" }));
    await waitFor(() => expect(mocked(api.addAddendum)).toHaveBeenCalledWith("cs1", expect.objectContaining({ changes: expect.objectContaining({
      dental: { findings: [ { tooth: "26", face: "D", situation: "caries" } ], plan_changes: { cancel: [ { item_id: "pi1", reason: "dente indicado para extração" } ] } } }) })));
  });
});
```

(Confira no `AddendumForm` real o rótulo da caixa "com mudanças" e do botão de envio; o rótulo novo "Mudar o odontograma ou o plano" é deste plano.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/DentalReading.test.tsx`
Expected: FAIL — as peças não existem.

- [ ] **Step 3: Implemente as três peças e a ligação no `ConsultationView` (o `DentalSummary` depois do bloco SOAP quando `kind === "dental"`), no `AddendumForm` (o `DentalAddendumFields` em vez de condutas na consulta odontológica; o cancelamento de item só com `dentalRole(cbo_code) === "dentist"`) e no `ToothActions` (o `ToothHistory` do dente selecionado)**

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental src/modules/consultation && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos de `ConsultationView` e `AddendumForm` continuam verdes).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/dental/DentalSummary.tsx src/modules/dental/DentalAddendumFields.tsx src/modules/dental/ToothHistory.tsx src/modules/dental/DentalReading.test.tsx \
  src/modules/dental/ToothActions.tsx src/modules/consultation/ConsultationView.tsx src/modules/consultation/AddendumForm.tsx
$G commit -m "feat: show the finalized dental consultation, the dental addendum and the tooth history

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Registro periodontal simplificado, periograma e endodontia (F-28.5)

**Files:**
- Create: `src/modules/dental/PeriodontalScreeningFields.tsx`, `src/modules/dental/PeriodontalChartFields.tsx`, `src/modules/dental/EndodonticsFields.tsx`
- Modify: `src/modules/dental/DentalEditor.tsx` (seção "Especialidades", só do dentista e fora do registro inicial)
- Test: `src/modules/dental/Specialties.test.tsx`

**Interfaces:**
- Consumes: `listEndodonticTreatments` (Task 1); `PERIO_SITES`, `perioProblems`, `screeningProblems`, `endoProblems`, `ENDO_KEY`, `NOMINAL_QUERY` (Task 2).
- Produces:
  - `PeriodontalScreeningFields({ value: PeriodontalScreening | null; onChange(v: PeriodontalScreening | null): void })` — `fieldset` "Registro periodontal simplificado"; botão "Preencher o registro periodontal" cria os seis sextantes vazios; por sextante, `select` "Sextante S1" com "0".."4" e "X — excluído" (`code: null`, D17) e a caixa "S1 com asterisco"; "Remover o registro" volta a `null`.
  - `PeriodontalChartFields({ value: PeriodontalChart | null; teeth: string[]; onChange(v): void })` — `fieldset` "Periograma"; "Dente" (`select` com os dentes do odontograma que não estão ausentes) e "Acrescentar dente"; por dente, um cartão "Dente 16" com as duas linhas de sítios (vestibular MV, V, DV; lingual ML, L, DL), cada sítio com "PS (mm)", "Recessão (mm)" e "Sangramento", rotulados "Dente 16, sítio MV, profundidade de sondagem (mm)" etc.; "Mobilidade" e "Furca" (0–3 ou "—"); "Remover dente"; as frases de `perioProblems` em `role="alert"` (o editor as soma ao "o que falta para finalizar"); campo vazio vai `null`, nunca 0.
  - `EndodonticsFields({ value: EndodonticEntry[]; teeth: string[]; patientId: string; onChange(v): void })` — `fieldset` "Endodontia"; linhas com "Dente", "Canais" (1–5) e "Situação" ("em andamento" | "concluída"); "Acrescentar dente"; "Ver histórico de endodontia" lê `listEndodonticTreatments` no clique (`NOMINAL_QUERY`) e lista "dente 46 · 3 canais · em andamento · 2 sessões".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/dental/Specialties.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import { useState } from "react";
vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(), listEndodonticTreatments: vi.fn() }));
import * as api from "../../lib/api";
import type { EndodonticEntry, PeriodontalChart, PeriodontalScreening } from "../../lib/api";
import { PeriodontalScreeningFields } from "./PeriodontalScreeningFields";
import { PeriodontalChartFields } from "./PeriodontalChartFields";
import { EndodonticsFields } from "./EndodonticsFields";
import { renderWithProviders } from "../../test/campaignFixtures";
import { dentalUser } from "../../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let out: unknown = null;
function Wrap<T>({ init, render: r }: { init: T; render(v: T, set: (v: T) => void): JSX.Element }) {
  const [ v, setV ] = useState(init);
  return r(v, (x) => { out = x; setV(x); });
}

describe("especialidades", () => {
  it("registro periodontal: seis sextantes, X e asterisco", () => {
    render(<Wrap<PeriodontalScreening | null> init={null} render={(v, set) => <PeriodontalScreeningFields value={v} onChange={set} />} />);
    fireEvent.click(screen.getByRole("button", { name: "Preencher o registro periodontal" }));
    fireEvent.change(screen.getByLabelText("Sextante S1"), { target: { value: "3" } });
    fireEvent.click(screen.getByLabelText("S1 com asterisco"));
    fireEvent.change(screen.getByLabelText("Sextante S4"), { target: { value: "X" } });
    expect((out as PeriodontalScreening).sextants.slice(0, 4)).toEqual([ { sextant: "S1", code: 3, star: true }, { sextant: "S2", code: 0, star: false },
      { sextant: "S3", code: 0, star: false }, { sextant: "S4", code: null, star: false } ]);
  });

  it("periograma: dente a dente, seis sítios rotulados; vazio é nulo; fora da faixa recusa", () => {
    render(<Wrap<PeriodontalChart | null> init={null} render={(v, set) => <PeriodontalChartFields value={v} teeth={[ "16", "26", "36" ]} onChange={set} />} />);
    expect(screen.queryByRole("option", { name: "36" })).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Dente"), { target: { value: "16" } });
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar dente" }));
    fireEvent.change(screen.getByLabelText("Dente 16, sítio MV, profundidade de sondagem (mm)"), { target: { value: "5" } });
    fireEvent.click(screen.getByLabelText("Dente 16, sítio MV, sangramento"));
    fireEvent.change(screen.getByLabelText("Dente 16, sítio DL, recessão (mm)"), { target: { value: "-2" } });
    const tooth = (out as PeriodontalChart).teeth[0];
    expect(tooth.sites.find((s) => s.site === "MV")).toEqual({ site: "MV", probing_depth_mm: 5, recession_mm: null, bleeding: true });
    expect(tooth.sites.find((s) => s.site === "DL")?.recession_mm).toBe(-2);
    expect(tooth.sites).toHaveLength(6);
    fireEvent.change(screen.getByLabelText("Dente 16, sítio V, profundidade de sondagem (mm)"), { target: { value: "25" } });
    expect(screen.getByRole("alert").textContent).toContain("dente 16, sítio V: profundidade de sondagem de 0 a 20 mm");
  });

  it("endodontia: canais e situação; histórico só no clique", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser());
    mocked(api.listEndodonticTreatments).mockResolvedValue([ { tooth: "46", canals: 3, status: "in_progress",
      sessions: [ { consultation_id: "cs0", at: "2026-10-01T09:00:00-03:00", by: { name: "Dra. Lia Moura" } }, { consultation_id: "cs1", at: "2026-10-15T09:00:00-03:00", by: null } ] } ]);
    renderWithProviders(<Wrap<EndodonticEntry[]> init={[]} render={(v, set) => <EndodonticsFields value={v} teeth={[ "46" ]} patientId="pa1" onChange={set} />} />);
    fireEvent.change(screen.getByLabelText("Dente"), { target: { value: "46" } });
    fireEvent.click(screen.getByRole("button", { name: "Acrescentar dente" }));
    fireEvent.change(screen.getByLabelText("Dente 46, canais"), { target: { value: "3" } });
    expect(out).toEqual([ { tooth: "46", canals: 3, status: "in_progress" } ]);
    expect(mocked(api.listEndodonticTreatments)).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Ver histórico de endodontia" }));
    expect(await screen.findByText("dente 46 · 3 canais · em andamento · 2 sessões")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/Specialties.test.tsx`
Expected: FAIL — as peças não existem.

- [ ] **Step 3: Implemente as três peças e a seção "Especialidades" no `DentalEditor`** (três seções recolhíveis com `<details>`, abertas quando já têm valor; os dentes oferecidos vêm do odontograma, sem os `missing`; as frases de `perioProblems`, `screeningProblems` e `endoProblems` entram no "o que falta para finalizar" do editor da Task 7)

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/dental/PeriodontalScreeningFields.tsx src/modules/dental/PeriodontalChartFields.tsx src/modules/dental/EndodonticsFields.tsx \
  src/modules/dental/Specialties.test.tsx src/modules/dental/DentalEditor.tsx
$G commit -m "feat: add periodontal screening, periodontal chart and endodontics fields

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Prótese — pedido, etapas e instalação na consulta (F-28.6)

**Files:**
- Create: `src/modules/dental/ProsthesisPanel.tsx`
- Modify: `src/modules/DentalProsthesis.tsx` (corpo da tela-portão), `src/modules/dental/DentalEditor.tsx` (o painel, só do dentista e fora do registro inicial)
- Test: `src/modules/dental/Prosthesis.test.tsx`

**Interfaces:**
- Consumes: `createProsthesisOrder`, `changeProsthesisStage`, `listProsthesisOrders`, `listDentalLaboratories`, `listActiveUnits` (Task 1 e existente); `nextStages`, `prosthesisStageLabel`, `prosthesisStageTone`, `reasonProblem`, `labNameProblem`, `withProcedure`, `PROSTHESIS_KEY`, `LABS_KEY`, `NOMINAL_QUERY`, `STATIC_QUERY`, `dentalError` (Task 2); `ProcedureSearch` (Task 6); `DentalGate` (Task 3); `DataTable`.
- Produces:
  - `ProsthesisPanel({ consultation: Consultation; patientId: string; value: DentalFields; onChange(f: DentalFields): void; searchDelayMs? })` — `section` "Prótese"; "Ver pedidos de prótese do paciente" lê `listProsthesisOrders({ patientId })` no clique; cada pedido com nome, laboratório, etapa (`Tag`) e o histórico de etapas; pedido `received` oferece "Registrar instalação nesta consulta" → escolhe o local entre os `locations` do SIGTAP (arcada superior/inferior ou dente) e acrescenta o procedimento com `prosthesis_order_id` ao rascunho (a instalação vale ao finalizar — contrato §7); "Novo pedido": `ProcedureSearch only="prosthesis"`, "Laboratório" (`select` de `listDentalLaboratories` + "Outro (terceiro)" com "Nome do laboratório" 3–120), "Registrar pedido" → `createProsthesisOrder({ patient_id, consultation_id, sigtap, laboratory })` e relê a lista.
  - `DentalProsthesis` (tela): `DentalGate` "Próteses"; "Unidade" (`listActiveUnits`, `STATIC_QUERY`) e "Etapa" (todas, moldagem, enviada, recebida); `DataTable` (`card: "title"` no paciente, `card: "actions"` nas ações) lido com `NOMINAL_QUERY` ao abrir e ao filtrar; ações por `nextStages`: "Enviada ao laboratório", "Recebida", "Cancelar pedido" (motivo 10–500 antes); 409 `invalid_stage` → frase e releitura; "Ver mais" por `next_before`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/dental/Prosthesis.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
vi.mock("../../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  listProsthesisOrders: vi.fn(), createProsthesisOrder: vi.fn(), changeProsthesisStage: vi.fn(), listDentalLaboratories: vi.fn(),
  listActiveUnits: vi.fn(), searchDentalProcedures: vi.fn() }));
import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ProsthesisPanel } from "./ProsthesisPanel";
import { DentalProsthesis } from "../DentalProsthesis";
import { renderWithProviders } from "../../test/campaignFixtures";
import { DENTURE, UBS, dentalDraft, dentalUser, fields, prosthesisOrder } from "../../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("prótese", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser());
    mocked(api.listDentalLaboratories).mockResolvedValue([ { id: "cp1", name: "LRPD Municipal" } ]);
    mocked(api.listActiveUnits).mockResolvedValue([ UBS ]);
    mocked(api.searchDentalProcedures).mockResolvedValue([ DENTURE ]);
    mocked(api.createProsthesisOrder).mockResolvedValue(prosthesisOrder());
    mocked(api.listProsthesisOrders).mockResolvedValue({ orders: [ prosthesisOrder({ stage: "received" }) ], next_before: null });
  });

  it("na consulta: novo pedido com laboratório terceiro e instalação do recebido", async () => {
    const onChange = vi.fn();
    renderWithProviders(<ProsthesisPanel consultation={dentalDraft()} patientId="pa1" value={fields()} onChange={onChange} searchDelayMs={0} />);
    expect(mocked(api.listProsthesisOrders)).not.toHaveBeenCalled();
    fireEvent.click(screen.getByRole("button", { name: "Novo pedido" }));
    fireEvent.change(screen.getByLabelText("Procedimento (SIGTAP)"), { target: { value: "total" } });
    fireEvent.click(await screen.findByRole("button", { name: `${DENTURE.sigtap} — ${DENTURE.name}` }));
    fireEvent.change(await screen.findByLabelText("Laboratório"), { target: { value: "other" } });
    fireEvent.change(screen.getByLabelText("Nome do laboratório"), { target: { value: "Lab Sorriso" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar pedido" }));
    await waitFor(() => expect(mocked(api.createProsthesisOrder)).toHaveBeenCalledWith({ patient_id: "pa1", consultation_id: "cs1", sigtap: DENTURE.sigtap,
      laboratory: { name: "Lab Sorriso" } }));
    fireEvent.click(screen.getByRole("button", { name: "Ver pedidos de prótese do paciente" }));
    fireEvent.click(await screen.findByRole("button", { name: `Registrar instalação nesta consulta — ${DENTURE.name}` }));
    fireEvent.click(screen.getByRole("button", { name: "Arcada superior" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar instalação" }));
    expect(onChange).toHaveBeenCalledWith(fields({ procedures: [ { sigtap: DENTURE.sigtap, name: DENTURE.name,
      location: { kind: "arch", tooth: null, face: null, sextant: null, arch: "upper" }, quantity: 1, plan_item_id: null, prosthesis_order_id: "po1" } ] }));
  });

  it("tela: etapas permitidas, cancelamento com motivo e etapa mudada por outro", async () => {
    mocked(api.listProsthesisOrders).mockResolvedValue({ orders: [ prosthesisOrder() ], next_before: null });
    mocked(api.changeProsthesisStage).mockResolvedValueOnce(prosthesisOrder({ stage: "sent" }))
      .mockRejectedValueOnce(new ApiError(409, { error: "invalid_stage" }, "x"));
    renderWithProviders(<DentalProsthesis />);
    fireEvent.change(await screen.findByLabelText("Unidade"), { target: { value: "u1" } });
    expect(await screen.findByRole("button", { name: "Enviada ao laboratório — Joana Lima" })).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Recebida — Joana Lima" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Enviada ao laboratório — Joana Lima" }));
    await waitFor(() => expect(mocked(api.changeProsthesisStage)).toHaveBeenCalledWith("po1", { stage: "sent" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar pedido — Joana Lima" }));
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "curto" } });
    expect((screen.getByRole("button", { name: "Confirmar cancelamento" }) as HTMLButtonElement).disabled).toBe(true);
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "paciente desistiu do tratamento" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
    expect(await screen.findByText("o pedido de prótese mudou de etapa — a lista foi relida")).not.toBeNull();
    expect(mocked(api.listProsthesisOrders).mock.calls.length).toBeGreaterThanOrEqual(3);
    expect(mocked(api.listProsthesisOrders)).toHaveBeenCalledWith({ healthUnitId: "u1", stage: null, before: null });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental/Prosthesis.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente o `ProsthesisPanel` e o corpo do `DentalProsthesis`** (a tela começa sem unidade escolhida e não lê nada até a escolha; nome do paciente só no corpo da resposta, na tela; rótulos das ações levam o nome do paciente para os leitores de tela)

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/dental src/modules/DentalProsthesis.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/dental/ProsthesisPanel.tsx src/modules/dental/Prosthesis.test.tsx src/modules/DentalProsthesis.tsx src/modules/dental/DentalEditor.tsx
$G commit -m "feat: add prosthesis orders, stages and installation in the consultation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Cadastro do CEO e lista do TSB/ASB (F-28.1)

**Files:**
- Modify: `src/modules/DentalAdmin.tsx` (corpo da tela-portão)
- Test: `src/modules/DentalAdmin.test.tsx`

**Interfaces:**
- Consumes: `listCeoProfiles`, `putCeoProfile`, `getAuxiliaryProcedures`, `putAuxiliaryProcedures` (Task 1); `CEO_TYPES`, `CEO_SPECIALTIES`, `CEO_PROFILES_KEY`, `AUX_PROCEDURES_KEY`, `STATIC_QUERY`, `dentalError` (Task 2); `ProcedureSearch` (Task 6); `SensitiveAction` (escrita com step-up); `DentalGate`.
- Produces: `DentalAdmin({ onGoToSecurity?(): void })` — dois `Panel`: **"Unidades CEO"** (uma linha por perfil — unidade `kind: ceo` —; "Tipo do CEO" I/II/III, as seis especialidades em caixas; "Salvar perfil — <unidade>" pelo `SensitiveAction`; sem unidade CEO: "Nenhuma unidade CEO. Cadastre a unidade com o tipo CEO em Atendimento → Unidades."; tipo vazio recusa antes "escolha o tipo do CEO") e **"Procedimentos do TSB e do ASB"** (lista com "<sigtap> — <nome>" e "Remover"; `ProcedureSearch role="dentist"` para acrescentar — repetido não duplica; "Salvar lista" pelo `SensitiveAction` com `putAuxiliaryProcedures(sigtaps)`; a frase "TSB e ASB registram só estes procedimentos e só dão baixa no plano com eles.").

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/DentalAdmin.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listCeoProfiles: vi.fn(), putCeoProfile: vi.fn(), getAuxiliaryProcedures: vi.fn(), putAuxiliaryProcedures: vi.fn(), searchDentalProcedures: vi.fn() }));
import * as api from "../lib/api";
import { DentalAdmin } from "./DentalAdmin";
import { renderWithProviders } from "../test/campaignFixtures";
import { FLUOR, SCALING, ceoProfile, dentalUser } from "../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("DentalAdmin", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser([ "municipal_admin" ]));
    mocked(api.listCeoProfiles).mockResolvedValue([ ceoProfile({ ceo_type: null, specialties: [] }) ]);
    mocked(api.putCeoProfile).mockResolvedValue(ceoProfile());
    mocked(api.getAuxiliaryProcedures).mockResolvedValue([ { sigtap: FLUOR.sigtap, name: FLUOR.name } ]);
    mocked(api.putAuxiliaryProcedures).mockResolvedValue([]);
    mocked(api.searchDentalProcedures).mockResolvedValue([ SCALING, FLUOR ]);
  });

  it("perfil do CEO: tipo obrigatório antes; especialidades no corpo, com step-up", async () => {
    renderWithProviders(<DentalAdmin />);
    fireEvent.click(await screen.findByRole("button", { name: "Salvar perfil — CEO Boqueirão" }));
    expect(screen.getByText("escolha o tipo do CEO")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Tipo do CEO — CEO Boqueirão"), { target: { value: "II" } });
    fireEvent.click(screen.getByLabelText("Endodontia — CEO Boqueirão"));
    fireEvent.click(screen.getByLabelText("Prótese — CEO Boqueirão"));
    fireEvent.click(screen.getByRole("button", { name: "Salvar perfil — CEO Boqueirão" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.putCeoProfile)).toHaveBeenCalledWith("u9", { ceo_type: "II", specialties: [ "endodontics", "prosthesis" ] }));
  });

  it("lista do TSB/ASB: acrescenta sem repetir, remove e salva", async () => {
    renderWithProviders(<DentalAdmin />);
    expect(await screen.findByText(`${FLUOR.sigtap} — ${FLUOR.name}`)).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Procedimento (SIGTAP)"), { target: { value: "rasp" } });
    fireEvent.click(await screen.findByRole("button", { name: `${SCALING.sigtap} — ${SCALING.name}` }));
    fireEvent.change(screen.getByLabelText("Procedimento (SIGTAP)"), { target: { value: "flu" } });
    fireEvent.click(await screen.findByRole("button", { name: `${FLUOR.sigtap} — ${FLUOR.name}` }));
    expect(screen.getAllByText(`${FLUOR.sigtap} — ${FLUOR.name}`).filter((el) => el.tagName !== "BUTTON")).toHaveLength(1);
    fireEvent.click(screen.getByRole("button", { name: "Salvar lista" }));
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
    await waitFor(() => expect(mocked(api.putAuxiliaryProcedures)).toHaveBeenCalledWith([ FLUOR.sigtap, SCALING.sigtap ]));
  });

  it("sem unidade CEO, o caminho para cadastrar", async () => {
    mocked(api.listCeoProfiles).mockResolvedValue([]);
    renderWithProviders(<DentalAdmin />);
    expect(await screen.findByText("Nenhuma unidade CEO. Cadastre a unidade com o tipo CEO em Atendimento → Unidades.")).not.toBeNull();
  });
});
```

(Confira o rótulo real do botão de confirmação do `SensitiveAction` — `confirmLabel` padrão — e use o mesmo.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/DentalAdmin.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente o corpo de `DentalAdmin`** (depois de salvar, invalida `CEO_PROFILES_KEY`/`AUX_PROCEDURES_KEY` e diz "Perfil salvo." / "Lista salva."; 401 `mfa_required` é do `SensitiveAction`; o resto, `dentalError`)

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/DentalAdmin.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/DentalAdmin.tsx src/modules/DentalAdmin.test.tsx
$G commit -m "feat: add the CEO profile and the auxiliary procedure list administration

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Painéis — metas do CEO e B1/B2 (F-28.8)

**Files:**
- Modify: `src/modules/DentalCeoGoals.tsx`, `src/modules/DentalIndicators.tsx` (corpos das telas-portão)
- Test: `src/modules/DentalPanels.test.tsx`

**Interfaces:**
- Consumes: `listCeoProfiles`, `getCeoGoals`, `listDentalTeams`, `getDentalIndicators` (Task 1); `goalKeyLabel`, `compositionLabel`, `ratioLabel`, `monthOf`, `CEO_GOALS_KEY`, `DENTAL_TEAMS_KEY`, `DENTAL_INDICATORS_KEY`, `CEO_PROFILES_KEY`, `STATIC_QUERY`, `dentalError` (Task 2); `DentalGate`, `DataTable`, `StatTile`, `Tag`.
- Produces:
  - `DentalCeoGoals({ today?: string })` — "Unidade CEO" (perfis; a primeira pré-escolhida) e "Mês" (`<input type="month">`, padrão `monthOf(today)`); `DataTable` por especialidade: "Especialidade", "Meta", "Feito", "Composição" ("restaurações ≥ 50%: 62,0% · ok" / "dentes com 3+ raízes ≥ 20%: 10,0% · abaixo"), "Situação" (`Tag` "atingida" `ok` / "não atingida" `down`), "Faltas em 12 meses" ("1 seguida · 2 no total"); com `suspension_risk`: `role="alert"` "Risco de suspensão do incentivo: meta não atingida 2 meses seguidos ou 3 em 12 meses em alguma especialidade."; com `history` (D22), a tabela "Últimos 12 meses" (mês × especialidade, "feito · ✓/✗"); perfil sem tipo: "Defina o tipo do CEO em CEO e lista do TSB/ASB." e nada é lido.
  - `DentalIndicators({ today?: string })` — "Equipe de saúde bucal" (`listDentalTeams`, "<nome> · INE <ine>") e "Período" (`<input type="month">`, padrão `monthOf(today)`, D23); dois `StatTile`: "B1 — primeira consulta programática" com `ratioLabel` e "17 de 40 pessoas", e "B2 — tratamento concluído" com "9 de 17 primeiras consultas"; a nota "Estimativa — o valor oficial é calculado pelo SIAPS." (o `ESTIMATE_NOTICE` do 23 não serve: fala de ACS e vacinas); sem equipe: "Nenhuma equipe de saúde bucal importada do CNES.".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/DentalPanels.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
vi.mock("../lib/api", async (importOriginal) => ({ ...(await importOriginal<typeof import("../lib/api")>()), fetchCurrentSession: vi.fn(),
  listCeoProfiles: vi.fn(), getCeoGoals: vi.fn(), listDentalTeams: vi.fn(), getDentalIndicators: vi.fn() }));
import * as api from "../lib/api";
import { DentalCeoGoals } from "./DentalCeoGoals";
import { DentalIndicators } from "./DentalIndicators";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY28, ceoGoals, ceoProfile, dentalTeam, dentalUser } from "../test/dentalFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

describe("painéis da saúde bucal", () => {
  beforeEach(() => {
    mocked(api.fetchCurrentSession).mockResolvedValue(dentalUser([ "analyst" ]));
    mocked(api.listCeoProfiles).mockResolvedValue([ ceoProfile() ]);
    mocked(api.getCeoGoals).mockResolvedValue(ceoGoals());
    mocked(api.listDentalTeams).mockResolvedValue([ dentalTeam() ]);
    mocked(api.getDentalIndicators).mockResolvedValue({ b1: { numerator: 17, denominator: 40 }, b2: { numerator: 9, denominator: 17 } });
  });

  it("metas: mês corrente, composição, situação, faltas e risco de suspensão", async () => {
    renderWithProviders(<DentalCeoGoals today={TODAY28} />);
    await waitFor(() => expect(mocked(api.getCeoGoals)).toHaveBeenCalledWith("u9", "2026-10"));
    expect(await screen.findByText("Periodontia")).not.toBeNull();
    expect(screen.getByText("restaurações ≥ 50%: 62,0% · ok")).not.toBeNull();
    expect(screen.getByText("dentes com 3+ raízes ≥ 20%: 10,0% · abaixo")).not.toBeNull();
    expect(screen.getByText("1 seguida · 2 no total")).not.toBeNull();
    expect(screen.getByRole("alert").textContent).toContain("Risco de suspensão do incentivo");
    fireEvent.change(screen.getByLabelText("Mês"), { target: { value: "2026-09" } });
    await waitFor(() => expect(mocked(api.getCeoGoals)).toHaveBeenLastCalledWith("u9", "2026-09"));
  });

  it("perfil sem tipo não lê metas", async () => {
    mocked(api.listCeoProfiles).mockResolvedValue([ ceoProfile({ ceo_type: null }) ]);
    renderWithProviders(<DentalCeoGoals today={TODAY28} />);
    expect(await screen.findByText("Defina o tipo do CEO em CEO e lista do TSB/ASB.")).not.toBeNull();
    expect(mocked(api.getCeoGoals)).not.toHaveBeenCalled();
  });

  it("B1 e B2 por equipe, com vírgula decimal e a nota de estimativa", async () => {
    renderWithProviders(<DentalIndicators today={TODAY28} />);
    await waitFor(() => expect(mocked(api.getDentalIndicators)).toHaveBeenCalledWith("t1", "2026-10"));
    expect(await screen.findByText("42,5%")).not.toBeNull();
    expect(screen.getByText("17 de 40 pessoas")).not.toBeNull();
    expect(screen.getByText("52,9%")).not.toBeNull();
    expect(screen.getByText("Estimativa — o valor oficial é calculado pelo SIAPS.")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/DentalPanels.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente os dois corpos** (`STATIC_QUERY`, chaves com unidade/equipe e mês; nenhum `refetchInterval`; nenhuma pessoa na tela)

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod28 src/modules/DentalPanels.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/DentalCeoGoals.tsx src/modules/DentalIndicators.tsx src/modules/DentalPanels.test.tsx
$G commit -m "feat: add the CEO goals and B1/B2 oral health panels

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Suíte, build, revisão e prova no navegador (api real na 3050)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–12 na branch `feat/mod-28-oral-health`; o api do 28 na **3050** com a semente do 28.
- Produces: branch pronta para merge (depois do merge do api do 28); evidências em `evidencias/2026-10-30-modulo-28/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test dashboard -w dashboard-mod28 && cd $W && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base da Task 0 mais **13 arquivos de teste novos** (`api.dental`, `dental`, `modules.dental`, `UnitForm.dental`, `Odontogram`, `TreatmentPlanPanel`, `DentalEditor`, `ConsultationWorkspace.dental`, `DentalReading`, `Specialties`, `Prosthesis`, `DentalAdmin`, `DentalPanels`) e os existentes com casos novos (`features.test.ts`, `modules.test.ts`, `consultation.test.ts`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css package.json package-lock.json src/lib/timeline.ts src/modules/billing
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css`, dependência, linha do tempo ou faturamento mudados (o `vite.config.ts` muda só pelo proxy da Task 1).

- [ ] **Step 2: Nada vaza, nada fica no navegador, nada que grava trilha relê sozinho**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage\|indexedDB" src/lib/dental.ts src/modules/dental src/modules/Dental*.tsx | grep -v "\.test\."
grep -rn "refetchInterval" src/modules/dental src/modules/Dental*.tsx | grep -v "\.test\."
sed -n '/─── Saúde bucal (módulo 28/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [A-Za-z_, :]* })'
grep -rn 'outline: "none"' src/modules/dental src/modules/Dental*.tsx
```

Expected: o primeiro, o segundo e o quarto sem saída; o terceiro só com `tooth` (D9), `before`, `limit`, `health_unit_id`, `status`, `patient_id`, `stage`, `month`, `team_id` e `period`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§1–§6, §8), o ADR 0046, o contrato (§1–§11) e as Divergências deste plano. Pontos de atenção:
- sem `dental_care` o dashboard é o de hoje (testes antigos verdes; só contagens de grupos mudaram); consulta odontológica finalizada continua legível;
- a consulta comum não mudou de corpo nem de regra (`startConsultation(id)` manda `{}`; `checkDraft` sem `dental`);
- o auxiliar não tem documento, regulação, registro inicial, encaminhamento, plano, especialidade nem prótese; o api recusa de novo;
- toda recusa da finalização aparece de uma vez; `plan_item_not_open` relê o plano e não reenvia;
- odontograma operável só por teclado e por toque, com rótulos completos e alvo ≥ 44 px;
- leitura nominal só ao abrir, no clique ou depois de uma ação; painéis só contagens;
- dente, face, situação, procedimento e motivo só no corpo (exceto o `tooth` do D9);
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova e o grupo no `CATALOG`; o `ProceduresPanel` do 22 nunca na consulta odontológica.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 28 na **3050** (plano do api: servidor do worktree `apps/api/.claude/mod28`, migrações de cidade e plataforma a partir de `20261030900001` aplicadas no banco de dev **com autorização do usuário**, e a semente do 28: Curitiba com `clinical_record`, `clinical_documents` e `dental_care`; a UBS Centro e um CEO tipo II; um dentista (223208) na UBS, um TSB (322405) na UBS e um endodontista no CEO; uma paciente gestante inscrita na linha de pré-natal e um paciente masculino; uma eSB importada; um laboratório cadastrado). Ligue em Curitiba, pelo maintenance, `dental_care`. O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod28 --network rota-saude_default -p 5206:5206 \
  -v $R/.worktrees/dashboard-mod28:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3050 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5206 --host 0.0.0.0"
docker logs -f dashboard-mod28   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 28 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5206/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Roteiro, com screenshot de cada passo em `evidencias/2026-10-30-modulo-28/dashboard/`:
1. **Admin** — Atendimento → Unidades: o tipo "CEO" aparece; Saúde bucal → **CEO e lista do TSB/ASB**: tipo II e especialidades (step-up); acrescentar raspagem à lista do TSB/ASB (step-up).
2. **Dentista, registro inicial** — chamar a paciente; "Registro inicial do odontograma"; marcar ausente, cariado e restaurado em dentes e faces **só pelo teclado** (Tab, setas atravessando quadrantes, Enter, a face pelo painel do dente); finalizar; nenhuma ficha FAO nova na Produção e-SUS.
3. **Dentista, primeira consulta** — nova consulta odontológica: o odontograma mostra o registro inicial; tipo de atendimento "consulta agendada" sem tipo de consulta → "o que falta" aponta; "primeira consulta"; vigilância "dor de dente"; gestante vem "sim — pré-preenchido, confirme"; incluir no plano uma restauração (face) e uma raspagem (sextante S1); registrar a restauração ligada ao item; desfecho "retorno"; finalizar; o plano aparece com um item feito.
4. **Limites e regras** — tentar 7 raspagens por sextante e 2 flúor na boca toda → recusas antes; marcar "tratamento concluído" com "manutenção" → recusa antes; "alta do episódio" com "primeira consulta" → recusa antes.
5. **Concorrência** — abrir a mesma paciente em duas consultas (dois dentistas); ligar o mesmo item do plano nas duas; finalizar a primeira e depois a segunda → a segunda mostra `plan_item_not_open` com as outras recusas, o plano relido e o vínculo desfeito, o procedimento mantido.
6. **Tratamento concluído** — consulta de retorno: registrar a raspagem ligada ao último item → aparece "Marcar tratamento concluído"; marcar e finalizar → plano concluído.
7. **TSB** — consulta odontológica: sem "Registro inicial", aba Documentos, "Pedir regulação", encaminhamentos e "Cancelar item"; a busca só mostra a lista; baixa num item com SIGTAP da lista.
8. **Paciente masculino** — o campo "Gestante" não aparece.
9. **Documentos e regulação** — dentista: aba Documentos emite atestado e receita (com a política de assinatura de hoje); desfecho "encaminhamento: endodontia" com `care_regulation` ligado → sem pedido, a recusa do 26a; com o pedido de especialidade odontológica, finaliza.
10. **CEO** — endodontista no CEO: plano próprio do CEO (o da UBS não aparece como aberto aqui); periograma de dois dentes só pelo teclado, registro periodontal com um X, endodontia do 46 (3 canais, em andamento) e o histórico de endodontia; o histórico do dente 16 mostra os eventos com consulta e adendo.
11. **Adendo** — na consulta finalizada, um achado novo e o cancelamento de um item com motivo; o odontograma da próxima consulta mostra o achado.
12. **Prótese** — novo pedido de prótese total maxilar ao laboratório terceiro; na tela **Próteses**, "Enviada" e "Recebida"; na consulta seguinte, "Registrar instalação nesta consulta" na arcada superior, finalizar → o pedido fica "instalada".
13. **Painéis** — admin ou analista: **Metas do CEO** do mês com a composição e o risco; **Indicadores B1 e B2** da eSB.
14. **Linha do tempo** — a consulta odontológica aparece como consulta (título do api) e abre com o resumo odontológico.
15. **Sem polling e sem vazamento** — aba de rede aberta por 3 minutos com a consulta odontológica e a tela de próteses abertas: nenhuma chamada a `dental_chart`, `dental_plans`, `events`, `endodontic_treatments` ou `prosthesis_orders` sem ação; nenhuma URL com situação, procedimento, face ou motivo; `localStorage` e `sessionStorage` sem nada da saúde bucal.
16. **Tablet e telefone** (larguras 1024, 768 e 375): o odontograma cabe ou rola dentro de si, sem rolagem lateral da página; alvos de toque confortáveis; as listas viram cartões no telefone.
17. No maintenance, desligar `dental_care` e recarregar: o grupo some, "Iniciar consulta odontológica" some, a unidade CEO não é oferecida no cadastro; a consulta odontológica finalizada continua legível na linha do tempo; religar devolve tudo.

Depois: `docker rm -f dashboard-mod28`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

F-28.2..F-28.8 vão a **Done** se a parte do api já está Done (o dashboard é o último app deles); F-28.1 espera o maintenance. Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified**. Use o mesmo `gh project item-edit` da Task 0 com o id da opção Done (`98236657`) e o de Verified (da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 28, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-28-oral-health
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod28
/opt/homebrew/bin/git branch -d feat/mod-28-oral-health
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (D1–D24). O plano do api ainda não existe na escrita deste; quem decide é o coordenador. A Task 0 (Step 4) confere o que o api implementou; o que vier diferente vira ajuste em `src/lib/api.ts`, nas fixtures e nas frases, antes da prova.

1. **D1 — Prefixos das rotas.** O contrato escreve as rotas sem prefixo. Proposta, seguindo o código: consulta, paciente e opções sob `/attendance` (`/attendance/attendances/:id/consultation`, `/attendance/consultations/:id`, `/attendance/consultation_options`, `/attendance/patients/:id/dental_chart`, `…/dental_chart/events`, `…/dental_plans`, `…/endodontic_treatments`); o resto sob `/dental` (proxy próprio no Vite).
2. **D2 — Erros da finalização.** O contrato diz `problems[]` sem a forma. Proposta: a do 27a §13 H8 — `error` e `path` do primeiro no topo e `problems: [ { "path", "error" } ]`, com `path` no formato `dental.procedures[i].plan_item_id`, `dental.oral_health_surveillance`, `dental.pregnant`.
3. **D3 — Papel antes de iniciar.** O `GET /consultation_options` ganha `dental_role: "dentist" | "auxiliary" | null` (a tela precisa saber se oferece "Registro inicial" antes de ter a consulta).
4. **D4 — Vocabulário das situações.** O contrato deixa os códigos para o YAML do api. Proposta: `consultation_options.dental_situations: [ { "code", "label", "applies_to": "tooth" | "face" | "both" } ]`, para a tela rotular e filtrar por nível; sem ele, o código aparece cru.
5. **D5 — Busca de procedimento odontológico.** Falta no contrato. Proposta: `POST /dental/procedures/search { "q" }` → `{ "items": [ { "sigtap", "name", "locations": [<kind>], "max_quantity", "auxiliary_allowed", "prosthesis" } ] }`, com `auxiliary_allowed` calculado pela lista da cidade.
6. **D6 — Instalação da prótese.** O §7 fala em `prosthesis_order_id` no procedimento, mas o `<dental_fields>.procedures[]` do §2 não o tem. Proposta: `procedures[]` ganha `prosthesis_order_id | null`.
7. **D7 — Nome na leitura.** `procedures[]` e `plan_changes.add[]` da leitura trazem `name` (a tela não relê o SIGTAP para mostrar o rascunho); a escrita ignora `name` (a tela não o manda).
8. **D8 — Recusas que faltam na finalização.** Propostas: `care_type_required` (tipo de atendimento é obrigatório na FAO), `dental_outcome_required` (desfecho é obrigatório, 1–17) e `baseline_requires_findings` (registro inicial sem achado).
9. **D9 — Filtro por dente na URL.** `GET …/dental_chart/events?tooth=` põe o dente na URL (dado clínico, pela regra do brief). Proposta: aceitar como está (número do dente sem situação nem paciente identificável — o `patient_id` já é id) **ou** trocar por `POST …/dental_chart/events/search { "tooth", "before" }`. O plano usa a forma do contrato e a Task 13 lista a exceção.
10. **D10 — Paginação dos eventos e próteses.** `limit` (padrão 50) nas duas listas com `next_before`.
11. **D11 — Plano sem plano aberto.** Proposta: na UBS, incluir item sem plano aberto fora da primeira consulta programática → 422 `plan_requires_first_consultation`; no CEO, a primeira consulta do paciente abre o plano.
12. **D12 — Regras do 19a na consulta odontológica.** O contrato não diz se as recusas do 19a valem no `dental`. Proposta: na odontológica não se exige conduta (o desfecho da FAO a substitui) nem o `care_type` do 19a; vale "A ou P"; no registro inicial não se exige problema, conduta nem A/P. O desfecho do atendimento (`outcome` do `finalize`) continua exigido nos dois.
13. **D13 — Forma do adendo.** Proposta: `changes.dental = { "findings": [<finding>], "plan_changes": { "cancel": [ { "item_id", "reason" } ] } }`; só o dentista cancela item.
14. **D14 — Pedido de prótese na leitura e laboratórios.** Proposta: `<prosthesis_order>` como o tipo `ProsthesisOrder` da Task 1 (com `stages[]` e `installed_in_consultation_id`) e `GET /dental/laboratories` → `{ "laboratories": [ { "id", "name" } ] }` (prestadores do 26a que fazem prótese), legível pelo dentista (a lista de prestadores do 26a é do admin).
15. **D15 — Próteses do paciente.** A lista do §7 filtra por unidade e etapa; a instalação na consulta precisa dos pedidos **do paciente**. Proposta: `patient_id=` no mesmo `GET`.
16. **D16 — Histórico de endodontia.** Proposta: `{ "treatments": [ { "tooth", "canals", "status", "sessions": [ { "consultation_id", "at", "by": { "name" } } ] } ] }`.
17. **D17 — Sextante excluído.** O registro periodontal simplificado usa "X" para sextante sem dentes funcionais. Proposta: `code: null` = X.
18. **D18 — Faixas do periograma.** Proposta: profundidade de sondagem 0–20 mm e recessão −10 a 20 mm, inteiros; mobilidade e furca 0–3; `null` = não medido. Recusa `invalid_request` com `path`.
19. **D19 — Motivos e nomes.** Motivo de cancelamento de item e de prótese 10–500; nome do laboratório terceiro 3–120.
20. **D20 — Lista dos perfis do CEO.** O §3 tem só `GET/PUT …/ceo_profiles/:health_unit_id`. Proposta: `GET /dental/admin/ceo_profiles` → `{ "profiles": [ <perfil com health_unit> ] }` (todas as unidades `ceo`, com `ceo_type: null` antes de configurar), legível também pelo analista (o painel de metas escolhe o CEO por ela).
21. **D21 — Lista do TSB/ASB.** Proposta: `GET` com `name` em cada linha e envelope `{ "procedures": [...] }` nas duas direções.
22. **D22 — Histórico das metas.** A spec pede "mês corrente e últimos 12 meses"; o §10 traz só as contagens de faltas. Proposta: `history: [ { "month", "specialties": [ { "key", "done", "met" } ] } ]` (12 meses até o pedido); sem ele, a tela mostra só as faltas. Chaves dos grupos: `basic`, `periodontics`, `endodontics`, `oral_surgery`; regras de composição: `restorations_min_50`, `multirooted_min_20`.
23. **D23 — Equipes e período do B1/B2.** Proposta: `GET /dental/teams` → `{ "teams": [ { "id", "name", "ine", "health_unit" } ] }` (eSB do escopo de quem lê) e `period=YYYY-MM` (mês de referência; o api aplica a janela da nota metodológica).
24. **D24 — Linha do tempo.** O §8 fala em "item `consultation` com `kind: "dental"`", mas a entrada da linha do tempo não tem `kind` de consulta. Proposta: o api manda o título pronto ("Consulta odontológica") e os procedimentos no `summary`; o dashboard não muda a linha do tempo (o `detail_ref` já abre a consulta, e a Task 8 mostra o resumo odontológico).

## Self-review

- **Cobertura (spec §2–§6, §8; contrato §1–§11, §14):** interruptor, unidade CEO, perfil do CEO e lista do TSB/ASB — Tasks 1, 3 e 11 (F-28.1; D20, D21); consulta odontológica, escolha do tipo, registro inicial, campos da FAO e regras condicionais, recusas `problems[]` de uma vez — Tasks 2, 6 e 7 (F-28.2; Review Focus 1 e 4; D2, D3, D8, D12); odontograma (dentições, faces, locais, achados × procedimentos, teclado, tablet), histórico do dente e adendo — Tasks 4, 6 e 8 (F-28.3; Review Focus 5; D4, D5, D9, D13); plano e sugestão de tratamento concluído — Tasks 2, 5 e 7 (F-28.4; Review Focus 3; D11); especialidades — Task 9 (F-28.5; D16–D18); prótese — Task 10 (F-28.6; D6, D14, D15); documentos e regulação do dentista, auxiliar sem eles — Task 7 (F-28.7; Review Focus 1); metas do CEO e B1/B2 — Task 12 (F-28.8; D22, D23); marco `c3_k` e CNES — só api (contrato §11–§12), a tela não muda; linha do tempo — D24; limites de quantidade — Task 2 (Review Focus 2). Imutabilidade, cálculo do odontograma, agregação da FAO, trilha e retenção LGPD são do api.
- **Placeholders:** valores lidos na execução são as rotas, formas e frases do api (Task 0, com a regra: o api vence), os ids dos cards e os ícones (Task 0, com a regra), as props reais do `UnitForm`, do `SegmentedControl`, do `AddendumForm` e do `SensitiveAction` (anotadas nos testes onde o rótulo vem de lá). As telas-portão da Task 3 têm corpo provisório preenchido pelas Tasks 10–12 dentro da mesma branch.
- **Tipos e nomes:** `DentalFields`, `DentalLocation`, `DentalPlan`, `Odontogram`, `DentalProcedureRef`, `ProsthesisOrder` (Task 1) são os das Tasks 2–12; `toothLoc` das fixtures e `sextantLoc`/`archLoc`/`MOUTH` (Task 2) montam o mesmo `DentalLocation`; `dentalFinalizeProblems`, `dentalApiProblems`, `planItemPathIndex`, `linkPlanItem` (Task 2) são os da Task 7; `ProcedureSearch` (Task 6) é o das Tasks 10 e 11; `DentalGate` (Task 3) em todas as telas; `finalizeProblems(check, mode)` (Task 7) com padrão `"general"` mantém as chamadas antigas.
- **Review Focus:** 1 — Task 7 ("auxiliar: sem documentos, regulação, registro inicial, encaminhamento nem plano"); 2 — Task 2 ("limites por local: recusa sem cortar; dente e face somam juntos"); 3 — Task 7 ("plan_item_not_open: relê o plano, desfaz o vínculo e mantém o procedimento"); 4 — Task 2 ("gestante: obrigatória no feminino, nula no masculino") e Task 6 ("gestante pré-preenchida pede confirmação; masculino sem o campo"); 5 — Task 4 ("teclado: tabindex móvel, setas atravessam quadrantes, Enter seleciona, Escape limpa").
- **Proporção:** os blocos de código são testes (rótulos, frases e valores que o implementador não decide sozinho), as fixtures e a lista de tipos do contrato; os corpos dos componentes ficam com a assinatura, o comportamento e as frases.
