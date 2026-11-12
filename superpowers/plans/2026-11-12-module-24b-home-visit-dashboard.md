# Módulo 24b — Visita domiciliar e atividade coletiva (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 24b** (`feat/mod-24b-home-visits`, worktree `apps/api/.claude/mod24b`, porta de dev **3057**) com as rotas do contrato §5–§9.1, os interruptores `home_visits` e `collective_activities` na sessão e a **task 1 do api (prova técnica) publicada** no contrato §10–§11 e em `field-sync/domains-v1.json` (códigos dos domínios, regras cruzadas do §4, "Outros" do público-alvo, `tipo_glicemia`). Ordem de entrega do contrato §15: contracts → api → **dashboard** → wpda → maintenance → catálogo. As Tasks 0–14 não dependem do api (dublês de `src/lib/api` com fixtures nas formas do contrato e das Divergências); a prova da Task 15 depende.

**Goal:** Lado dashboard do 24b: o grupo **"Visita e atividade coletiva"** no menu (depois de "Cadastro territorial"), cada item com o seu interruptor e o papel; a **Folha do dia** no celular primeiro (visitas de hoje, nova visita pela microárea — domicílio → família ou morador — ou por imóvel sem cadastro, desfecho, motivos por grupo com as regras do §3, medidas, observação, registro rápido com a guarda de duplicado, retirar com motivo, "Fechar o dia", os `blockers` em linguagem simples); **Visitas da equipe** (folhas com a situação da ficha, visitas, contagens, detalhe com observação); **Atribuição de microáreas** do enfermeiro com step-up; **Quem visitar** por microárea com o atalho "Registrar visita"; **Atividade coletiva** (nova, com as regras do §4 vindas de `options.domains`; lista; detalhe; "Registrada por engano"); **Aparelhos de campo** (cadastrar com step-up e QR uma vez, listar, revogar) e a lista do administrador; os kinds `home_visit` e `collective_activity` na **linha do tempo**; o `min_gap_days` preservado nas **linhas de cuidado**; as fontes novas na **Produção**.

**Architecture:** O cliente HTTP ganha a seção "Visita domiciliar e atividade coletiva (módulo 24b)" no fim de `src/lib/api.ts` (tipos = formas do contrato §5–§9.1 e das Divergências). As regras de tela ficam puras em `src/lib/homeVisits.ts` (portões, frases, rótulos, situação da ficha, `blockers`, regras da visita do §3 lidas de `context.domains`, frases de **todos** os códigos de erro) e `src/lib/collectiveActivities.ts` (as regras cruzadas do §4 **lidas de `options.domains.rules`**, nunca copiadas à mão). As telas ficam em `src/modules/homeVisits/` e `src/modules/collectiveActivities/`, com as telas-raiz `src/modules/{HomeVisitDay,HomeVisitsTeam,MicroAreaAssignments,WhoToVisit,CollectiveActivities,FieldDevices,FieldDevicesAdmin}.tsx`. O dashboard nunca decide o que o api garante: escopo da microárea, folha escolhida, folha complementar, turno, `has_reason`, geração da ficha, trilha, imutabilidade e revogação do aparelho são do api; a tela só não oferece nem envia o que já sabe que será recusado, e sempre manda o `expected_state` que leu.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3, `qrcode` 1.5 (já usado em `src/modules/Security.tsx:2`).

**Spec:** `docs/superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md` (decisões 1A–7A e 1–23; §4 telas; §5 segurança e LGPD; §6 testes do dashboard; §8 F-24.9, F-24.10, F-24.12, F-24.14–F-24.17), `docs/adr/0052.md`, a pesquisa `docs/pesquisa/2026-11-12-visita-e-coletiva-24b.md` (rótulos dos domínios) e o contrato `docs/superpowers/plans/2026-11-12-module-24b-home-visit-contracts.md` (§1–§9.1, §10–§12, §13, §14). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D-1 a D-20); nada foi resolvido em silêncio.

**Reconciliação (§16 do contrato, Q1–Q30, 2026-11-12):** D-1, D-2 → Q13; D-3 → **Q11, rejeitada a rota nova**: domicílios, famílias e moradores vêm das rotas de leitura do 24a (`listHouseholds`, `getHousehold`, `listRegistrations({ householdId })`), que passam a aceitar o CBO de campo nas microáreas atribuídas; `listVisitHouseholds`/`getVisitHousehold` ficam como adaptadores sobre elas. D-4, D-5, D-20 → Q14; D-6, D-7 → Q8 (busca no padrão do 27a: ninguém = `404 person_not_found`, lido como `null`; `sex` no lookup e nos candidatos, `has_cns` nos candidatos); D-8, D-9 → Q15 (corpo plano, como o plano já faz); D-10 → Q4 (a ficha 8 usa `citizen_without_birth_date`/`citizen_without_sex`, que já têm rótulo; a ficha 6 ganha `participant_without_sex`); D-11, D-17, D-18 → Q16; D-12 → Q17; D-13 → Q18; D-14 → Q19; D-15 → Q20; D-16 e W-11 do api → Q12 (sem "Gerar de novo" para `siaps_deadline_passed`); D-19 → Q21. Aparelhos ganham `status` (Q7).

## Global Constraints

- **Nomes no código** (contrato §1–§2): interruptores `home_visits` e `collective_activities`; desfecho `1` Realizada, `2` Recusada, `3` Ausente; turno `1` Manhã, `2` Tarde, `3` Noite; `subject` `resident` | `family_responsible` | `none`; `origin` `dashboard` | `field_app`; folha `open` | `closed`, `closed_by` `user` | `job`; visita `recorded` | `withdrawn`; atividade `recorded` | `mistaken`; situação da ficha `open` | `empty` | `not_generated` | `pending` | `sending` | `accepted` | `rejected` | `failed` | `correction_pending`; escopo da microárea `responsible` | `assigned` | `team`; motivo de revogação `user` | `admin` | `membership_ended` | `user_disabled`. **Código desconhecido aparece cru, nunca some.**
- **Interruptores:** `canUseHomeVisits(user)` = `canUseTerritory(user)` (`territory_registry` **e** `clinical_record`, `src/lib/territoryRegistry.ts`) **e** `home_visits`; `canUseCollectiveActivities(user)` = não operador, `collective_activities` **e** `clinical_record`. Sem o interruptor, o item some do menu e a tela mostra `HOME_VISITS_DISABLED` / `COLLECTIVE_DISABLED` sem ler nada; o 403 `feature_disabled` do api vira a mesma frase. O dashboard nunca escreve interruptor; as duas chaves ganham só rótulo em `features.ts`.
- **Quem faz o quê:** a sessão não traz o CBO nem o profissional (`SessionUser`, `src/lib/api.ts:43`). O menu mostra as telas de equipe a todo `health_professional` com o interruptor; **o que cada um pode fazer vem do `GET /home_visits/context` e do `GET /collective_activities/options`** (`can_register`, `reads_whole_team`, `out_of_area_allowed`, e os da D-4: `can_assign`, `can_register_device`). Enfermeiro e médico leem e não registram visita; só o enfermeiro atribui; aparelho só para a Tabela 7; o `municipal_admin` só vê e revoga aparelhos. O api confere de novo: `403 cbo_not_allowed` e `403 out_of_scope` têm frase própria.
- **Dado pessoal só no corpo:** nome, CPF, CNS, nascimento, mãe, motivo, medida, observação, tema, público, prática, procedimento, motivo da retirada e do engano, e o **token** do aparelho nunca na URL, em mensagem de erro, em `console`, em Analytics ou em storage. Na URL vão só ids (`team_id`, `micro_area_id`, `professional_id`, `household_id`, `id`), datas (`from`, `to`), os filtros codificados (`state[]`, `outcome[]`, `activity_type[]`, `reason[]`), o cursor `before` e o `limit`. Busca de pessoa por `POST …/people/lookup` e `…/people/duplicates`. **Nenhum storage** no módulo.
- **Leitura nominal com trilha** (folha com visitas, visita, visitas da equipe, atividade com participantes, quem visitar): `NOMINAL_QUERY` de `src/lib/careLines.ts` (nunca relida sozinha, sem `refetchInterval`); contagens (`/count`) com `COUNT_QUERY` (sem trilha, o padrão do 23/24a); contexto, opções e listas sem pessoa com `STATIC_QUERY`. Nenhum polling novo em leitura nominal.
- **Concorrência:** "Fechar o dia" manda `{ expected_state: "open" }`; "Retirar" manda `{ expected_state: "recorded", reason }`; "Registrada por engano" manda `{ expected_state: "recorded", reason }`. 409 `day_changed`, `day_closed`, `visit_changed`, `activity_changed` ou `already_assigned` → a frase do código, **uma** releitura do que estava na tela, nenhum reenvio automático; o motivo digitado fica no formulário se a ação ainda couber, senão o formulário fecha.
- **Sem duplo envio:** botão de escrita desabilitado enquanto a chamada corre (`useFormProblems` de `src/modules/psychosocial/useFormProblems.tsx`, com `refusals`/`isStale` do 24b). **Idempotência do registro:** cada formulário de visita e de atividade nasce com `client_uuid = crypto.randomUUID()`; o mesmo uuid vai em toda tentativa daquele formulário; ele é trocado só depois de gravado (201 ou 200) ou de uma recusa do api (4xx: nada foi gravado). Falha de rede ou 5xx guarda o uuid e trava os campos em "Tentar de novo" / "Descartar"; 200 (o api devolveu a gravada) mostra "Esta visita já estava gravada" (`REPLAYED_VISIT`).
- **Recusar em vez de cortar** (contrato §3–§5, §9.1): observação até 500 (code points depois do `strip`); motivo de retirada e de engano 10–500; nome do aparelho 1–60; `place_text` até 250; `participants_count` 1–999; `altered_count` 0–999 e ≤ participantes; `procedure_code` 10 dígitos; medidas nas faixas de `domains.measurement_ranges` (D-1) com vírgula decimal aceita e o número de casas do contrato.
- **Texto congelado** (api#32): observação da visita, motivo da retirada e motivo do engano têm o `FrozenTextNotice` antes do envio (helper `expectFrozenNotice` de `src/test/frozenNotice.ts`).
- **Token do aparelho:** só no estado local do diálogo de cadastro; vira QR (`toDataURL(qr_payload)`) e some ao fechar; nunca em `queryClient`, `useState` fora do diálogo, storage, `console` ou na URL; o texto do token não é mostrado (D-15).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo — `{}` quando não há campo).
- **Lição da CI (commit `443faca`):** todo teste que faz `vi.mock("…/lib/api", async (orig) => ({ ...(await orig()), … }))` inclui `fetchCurrentSession: vi.fn()` com `mockResolvedValue(<usuário>)`. A Task 15 varre isso.
- "Agora" é `new Date()`; telas recebem `now` por prop quando o dia importa, e os testes passam `NOW24B = new Date("2026-11-12T10:00:00-03:00")`. Datas e horas da cidade por `cityIsoDate`/`cityLocalIso` de `src/lib/format.ts` (o `vitest.config.ts` fixa `TZ=America/Sao_Paulo`). Sem relógio falso.
- **Interface (ciclo de interface):** shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG.screens` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo); **375 px**: a Folha do dia e o formulário de visita são listas de cartões e campos empilhados (nenhuma `<table>`), as listas da equipe usam `DataTable` com `card: "title"`/`"actions"`; `responsive.css` intocado; cores só por papéis do tema (`Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `ActionMessage`, `FrozenTextNotice`, `formStyles`, os estilos de `src/modules/exams/styles.ts`). Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- **Board:** a primeira ação da execução é mover os cards **F-24.9, F-24.10, F-24.12, F-24.14, F-24.15, F-24.16 e F-24.17** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-24.9: api, dashboard e maintenance; F-24.14 e F-24.15: api, dashboard e wpda; os demais: api e dashboard) e a **Verified** depois da prova no navegador (Task 15).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (via `SendMessage`), no formato `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task N · <o que ficou pronto> · <divergência nova, se houver>`.

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod24b` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-24b-home-visits` a partir de `origin/main` (`4524082`), com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod24b`, `G=/opt/homebrew/bin/git`.
- Testes: `$R/bin/test dashboard -w dashboard-mod24b <arquivos>` (o runner único; uma execução por vez; a saída é o resumo, o log fica em `.test-runs/`). **Nunca `vitest` direto.** Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- Ambiente de teste: sem jest-dom (`not.toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx`; as fixtures do 24b vêm de `src/test/homeVisitFixtures.ts` (Task 1).

## Review Focus

1. **A Folha do dia no celular perde, repete ou duplica visita.** Espera-se: a folha é uma lista de cartões (nenhuma `<table>`); o duplo clique em "Registrar visita" envia uma vez; a falha de rede guarda o `client_uuid` e o "Tentar de novo" manda o mesmo; o 200 mostra "já estava gravada"; o 201 de outro dia (folha complementar ou dia anterior) diz em que folha entrou; "Fechar o dia" manda `expected_state` e o 409 `day_changed` relê **uma** vez. Teste: Task 6, "fechar: expected_state, 409 relê uma vez" e Task 7, "client_uuid: mesmo na falha de rede, novo depois de gravar".
2. **Dado sensível vaza para fora do corpo.** Espera-se: nenhuma URL do módulo com CPF, CNS, nome, motivo, medida ou observação; a busca de pessoa só por POST; nada em storage ou `console`; o token do aparelho só no diálogo, como QR, some ao fechar e não está em nenhuma chave do `queryClient`. Teste: Task 1, "nada sensível na URL"; Task 13, "token só no diálogo; fechar apaga; nada no cache".
3. **A visita sai com o que a ficha proíbe.** Espera-se: imóvel não residencial esconde pessoa e medidas e só oferece os motivos do domínio; recusada e ausente não oferecem motivo nem medida; motivos 5 e 6 indisponíveis para `male`; medida fora da faixa e pressão sem o par não saem; todas as recusas de uma vez; o `path` do api (`reasons[2]`, `measurements.systolic`, `new_patient.cpf`) cai no campo certo. Teste: Task 2, tabela de casos, e Task 7, "não residencial: sem pessoa, sem medida, motivos do domínio".
4. **A atividade coletiva copia a ficha à mão.** Espera-se: o que é obrigatório, proibido ou opcional em cada tipo vem de `options.domains.rules` (o teste troca a regra na fixture e a tela segue); participantes obrigatórios nos tipos 5 e 6; SIGTAP só com a prática 30; tabagismo só nas práticas da regra; tema 18 nunca oferecido; o 409 `duplicate_suspected` de um participante reabre a busca **daquele** participante. Teste: Task 3, "regra vem do dado", e Task 11, "tipo 5 exige participante; prática 30 pede SIGTAP; duplicado por participante".
5. **Retirar, engano e atribuição mudam o que não devem.** Espera-se: "Retirar" só em visita `recorded` de folha `open` com `can_withdraw`; "Registrada por engano" só com `can_mark_mistaken`; motivo 10–500 com o aviso de texto congelado; atribuir e encerrar pedem step-up (`SensitiveAction`); o 409 `already_assigned` relê a lista. Teste: Task 6, "retirar"; Task 9, "atribuir com step-up; already_assigned relê"; Task 12, "engano".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.homeVisits.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/homeVisitFixtures.ts` | tipos e cliente HTTP do 24b; rótulos dos interruptores; proxy; dados de teste | 1 |
| `src/lib/homeVisits.ts`, `src/lib/homeVisits.test.ts` | portões, frases, rótulos, situação da ficha, `blockers`, regras da visita do §3, todos os códigos | 2 |
| `src/lib/collectiveActivities.ts`, `src/lib/collectiveActivities.test.ts` | regras do §4 lidas de `options.domains.rules`, validação, frases | 3 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.homeVisits.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/homeVisits/HomeVisitGate.tsx`, `src/modules/homeVisits/HomeVisitGate.test.tsx`, `src/modules/homeVisits/useVisitHandoff.ts`, cascas das sete telas-raiz | grupo novo no menu, portões, atalho em memória (F-24.9) | 4 |
| `src/modules/homeVisits/PersonPick.tsx`, `src/modules/homeVisits/QuickPatientForm.tsx`, `src/modules/homeVisits/PersonPick.test.tsx` | busca por CPF/CNS, duplicados e registro rápido (7A) | 5 |
| `src/modules/HomeVisitDay.tsx`, `src/modules/homeVisits/DaySheet.tsx`, `src/modules/homeVisits/VisitCard.tsx`, `src/modules/homeVisits/WithdrawForm.tsx`, `src/modules/homeVisits/DaySheet.test.tsx` | Folha do dia: visitas, `blockers`, fechar, retirar (F-24.10) | 6 |
| `src/modules/homeVisits/VisitForm.tsx`, `src/modules/homeVisits/PlacePick.tsx`, `src/modules/homeVisits/ReasonsField.tsx`, `src/modules/homeVisits/MeasurementsField.tsx`, `src/modules/homeVisits/useClientUuid.ts`, `src/modules/homeVisits/VisitForm.test.tsx` | nova visita (F-24.10) | 7 |
| `src/modules/HomeVisitsTeam.tsx`, `src/modules/homeVisits/DayList.tsx`, `src/modules/homeVisits/VisitList.tsx`, `src/modules/homeVisits/VisitDetail.tsx`, `src/modules/HomeVisitsTeam.test.tsx` | Visitas da equipe (F-24.10) | 8 |
| `src/modules/MicroAreaAssignments.tsx`, `src/modules/MicroAreaAssignments.test.tsx` | atribuição de microáreas (F-24.17, 6A) | 9 |
| `src/modules/WhoToVisit.tsx`, `src/modules/WhoToVisit.test.tsx` | Quem visitar (F-24.16) | 10 |
| `src/modules/collectiveActivities/ActivityForm.tsx`, `src/modules/collectiveActivities/ParticipantsField.tsx`, `src/modules/collectiveActivities/CodeChecklist.tsx`, `src/modules/collectiveActivities/ActivityForm.test.tsx` | nova atividade coletiva (F-24.12) | 11 |
| `src/modules/CollectiveActivities.tsx`, `src/modules/collectiveActivities/ActivityList.tsx`, `src/modules/collectiveActivities/ActivityDetail.tsx`, `src/modules/CollectiveActivities.test.tsx` | lista, detalhe, "Registrada por engano" (F-24.12) | 12 |
| `src/modules/FieldDevices.tsx`, `src/modules/FieldDevicesAdmin.tsx`, `src/modules/homeVisits/DeviceQrDialog.tsx`, `src/modules/FieldDevices.test.tsx`, `src/modules/FieldDevicesAdmin.test.tsx` | aparelhos de campo (F-24.17) | 13 |
| `src/lib/timeline.ts`, `src/lib/timeline.homeVisit.test.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/lib/careLines.ts`, `src/lib/careLines.test.ts`, `src/lib/api.ts` (`CarePractice`), `src/lib/production.ts`, `src/lib/production.test.ts` | linha do tempo, `min_gap_days`, fontes e motivos da Produção (F-24.14, F-24.15) | 14 |
| — | suíte, build, varredura, revisão, prova no navegador | 15 |

**Estratégia de teste:** cliente HTTP com `fetch` falso (o molde de `src/lib/api.homeVisits.test.ts` é o `src/lib/api.ombudsman.test.ts`: `stub`, `last`, `bodyOf`) conferindo URL, método e corpo — sobretudo o que **não** pode ir na URL e o `expected_state` em toda ação (Task 1); regras puras com tabela de casos, inclusive o §3 inteiro e o §4 **lido do dado** (Tasks 2–3); menu e portão por papel e interruptor (Task 4); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api **com `fetchCurrentSession`**, sessão por papel, contexto por CBO e fixtures do contrato, cobrindo o caminho feliz, cada recusa que muda a tela, o duplo envio, o `client_uuid`, o 409 e o que **não** pode aparecer (pessoa em não residencial, token fora do diálogo, observação na lista) (Tasks 5–14); `tsc`, build, a suíte inteira, as varreduras e a prova no navegador com o api real na 3057 e o Vite na 5218 (Task 15). O dashboard não testa o que é do api (escopo, folha complementar, turno, ficha, trilha, imutabilidade, revogação por fim de vínculo, sincronização `/field/v1`, wpda).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`4524082` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod24b`; o contrato §10–§11 depois da task 1 do api.
- Produces: cards em In Progress; worktree `$W` na branch `feat/mod-24b-home-visits`; a base anotada; as rotas, as formas e os domínios do api anotados.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.(9|1[0-7]) ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada um de F-24.9, F-24.10, F-24.12, F-24.14, F-24.15, F-24.16 e F-24.17 que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os sete cards em In Progress ou além. Se algum F-24.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Ouvidoria (módulo 33\|^async function jsonFetchWithStatus\|^const withQuery = \|^export interface SessionUser\|^export interface CarePractice {\|^export interface NewPatient {\|^export type PatientSex\|^export type TimelineKind ='
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "urgency_stock_deduction" | "ombudsman" | "ombudsman_anonymous_web";\|  ombudsman_anonymous_web: "Denúncia anônima sem login (canal do cidadão)"'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "territory-team" | "registry-status" | "territory-micro-areas"\|    { id: "territory-micro-areas", label: "Microáreas", icon: "⬡" }\|  const territoryOn = canUseTerritory(user);\|      if (item.id === "territory-micro-areas") return territoryOn && isAdmin;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - '
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '    "Cadastro territorial": {\|    "territory-micro-areas": {\|  IconHomeHeart, IconMap2, IconCalendarDue, IconHexagons,'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "territory-micro-areas":\|^export function useOmbudsmanHandoff\|^export function renderModule'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"\^/departments(/|\$)": proxy(TARGET)\|/departments → setores da secretaria'
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^export type PracticeExtra = \|    extra: present(p, \[ "window_start_days", "code_filter", "applies_to", "all_of" \])'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '  { key: "urgency", label: "Urgência" }\|^export function timelineKinds(\|  classified: "classificado"'
$G -C $R/apps/dashboard show origin/main:src/lib/production.ts | grep -n '  quantity_exceeds_location_limit: \|^export const SOURCE_LABEL'
$G -C $R/apps/dashboard show origin/main:src/modules/psychosocial/useFormProblems.tsx | grep -n '^export function useFormProblems(opts: FormProblemsOptions = {})'
$G -C $R/apps/dashboard show origin/main:src/lib/territoryDictionaries.ts | grep -n '^export function newPatientProblem\|^const NO_CPF_REASONS'
$G -C $R/apps/dashboard show origin/main:src/lib/format.ts | grep -n '^export function cityLocalIso\|^export function cityIsoDate'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod24b -b feat/mod-24b-home-visits origin/main
cd $R/.worktrees/dashboard-mod24b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod24b
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Walk|ClipboardList|ListDetails|MapPin|UserSearch|UsersGroup|DeviceMobile|DeviceMobileCog)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os oito ícones (falta algum → escolha outro que exista e use o mesmo na Task 4). **Anote** a base (arquivos e testes). A Task 15 espera a base mais **16 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira as rotas, as formas e os domínios do api do 24b**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api log --oneline -5 feat/mod-24b-home-visits 2>/dev/null || echo "api do 24b ainda sem branch"
$G -C $R/apps/api show feat/mod-24b-home-visits:config/routes.rb 2>/dev/null | grep -nE 'home_visits|collective_activities|field_devices'
$G -C $R/apps/api show feat/mod-24b-home-visits:config/field_sync/domains-v1.json 2>/dev/null | head -40
$G -C $R/docs log --oneline -3 -- superpowers/plans/2026-11-12-module-24b-home-visit-contracts.md
```

Expected: as rotas do contrato §5–§9.1 e o `domains-v1.json` com os domínios do §10–§11. Para cada Divergência (D-1 a D-20) anote se o api adotou a forma proposta ou outra; **onde o api diz outra coisa, vale o api** — a task dona ajusta tipo, fixture e teste antes de seguir (registre na descrição do commit). Sem branch do api ainda: siga pelas formas deste plano e repita este passo antes da Task 15. **Se a task 1 do api ainda não publicou o §10–§11, as Tasks 2, 3, 7 e 11 seguem pelas fixtures** (os códigos vêm do dado do api, não do código do dashboard), e a Task 15 confere de novo.

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 0 · worktree e base (<n> arquivos, <m> testes) · <divergências que o api já resolveu ou contrariou>`.

---

### Task 1: Tipos, cliente HTTP, interruptores, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/homeVisitFixtures.ts`
- Test: `src/lib/api.homeVisits.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `jsonFetchWithStatus`, `ApiError`, `withQuery`, `SessionUser`, `PatientSex` (existentes em `src/lib/api.ts`); `sessionWith` de `src/test/campaignFixtures.tsx`.
- Produces (todas em `src/lib/api.ts`, exportadas):
  - Tipos de código: `VisitOutcome = 1 | 2 | 3`, `VisitShift = 1 | 2 | 3`, `VisitSubject = "resident" | "family_responsible" | "none"`, `VisitOrigin = "dashboard" | "field_app"`, `DayState = "open" | "closed"`, `FichaSituation` (as nove do contrato §6 `| string`), `MicroAreaScope = "responsible" | "assigned" | "team"`, `MeasurementKey = "weight_kg" | "height_cm" | "systolic" | "diastolic" | "temperature_c" | "glucose_mg_dl"`, `CollectiveField = "meeting_topics" | "target_audience" | "health_topics" | "health_practices" | "participants"`, `FieldRule = "required" | "forbidden" | "optional"`.
  - Domínios (D-1, D-2): `CodeLabel { code: number; label: string }`; `HomeVisitDomains { motivo_visita: (CodeLabel & { group: string; group_label: string })[]; desfecho: CodeLabel[]; turno: CodeLabel[]; tipo_glicemia: CodeLabel[]; tipo_de_imovel: CodeLabel[]; non_residential_property_types: number[]; non_residential_reasons: number[]; male_forbidden_reasons: number[]; measurement_ranges: Record<MeasurementKey, { min: number; max: number; decimals: number }> }`; `CollectiveDomains { activity_types: CodeLabel[]; meeting_topics: CodeLabel[]; target_audience: CodeLabel[]; health_topics: CodeLabel[]; health_practices: CodeLabel[]; rules: { by_type: Record<string, Record<CollectiveField, FieldRule>>; health_topics_with_meeting_topic: number[]; practices_only_in_type: { type: number; practices: number[] }[]; procedure_practices: number[]; smoking_practices: number[]; participant_measurements: Record<"weight_kg" | "height_cm", { min: number; max: number; decimals: number }> } }`.
  - Contexto (§6 + D-4, D-5): `VisitMicroArea { id; code; name; scope: MicroAreaScope }`; `HomeVisitTeam { id; ine; name: string | null; cbo: string; can_register: boolean; reads_whole_team: boolean; out_of_area_allowed: boolean; can_assign: boolean; can_register_device: boolean; micro_areas: VisitMicroArea[]; professionals: { id: string; name: string; cbo: string }[] }`; `HomeVisitContext { today: string; min_date: string; teams: HomeVisitTeam[]; domains: HomeVisitDomains }`.
  - Visita (§6 + D-4, D-12): `VisitMeasurements { weight_kg?: number; height_cm?: number; systolic?: number; diastolic?: number; temperature_c?: number; glucose_mg_dl?: number; glucose_type?: number }`; `VisitPatientRef { id; display_name: string | null; has_cpf: boolean; cns_masked: string | null }`; `HomeVisit { id; client_uuid; day_id; occurred_at; shift: VisitShift; origin: VisitOrigin | string; property_type: string; micro_area: { id; code } | null; out_of_area: boolean; household: { id; address_short } | null; family_id: string | null; subject: VisitSubject | string; patient: VisitPatientRef | null; outcome: VisitOutcome; reasons: number[]; shared: boolean; measurements: VisitMeasurements | null; state: "recorded" | "withdrawn" | string; withdrawn: { at: string; reason: string | null; by: { name: string } } | null; recorded_by: { name: string }; note?: string | null; can_withdraw: boolean }`; `HomeVisitDay { id; professional: { id; name }; team: { id; ine; name: string | null }; day; sequence: number; state: DayState | string; closed_at: string | null; closed_by: "user" | "job" | null; visits_count: number; withdrawn_count: number; ficha: { status: FichaSituation; reasons: string[] }; blockers: { code: string; visit_id: string | null }[]; own: boolean; visits?: HomeVisit[] }`.
  - Pessoa (§5 + D-6, D-7): `QuickPatient { full_name: string; birth_date: string; sex: PatientSex; mother_name?: string | null; cpf?: string | null; no_cpf_reason?: string | null; cns?: string | null }`; `VisitPerson { patient_id: string; display_name: string | null; birth_date: string | null; has_cpf: boolean; cns_masked: string | null; sex: PatientSex | null }`; `VisitCandidate { patient_id; display_name; birth_date; has_cpf: boolean; has_cns: boolean; sex: PatientSex | null }`; `PersonScope = "home_visits" | "collective_activities"`; `PersonDoc = { cpf: string } | { cns: string }`.
  - Domicílio para a visita (D-3 → Q11: forma de tela, montada por adaptador a partir de `Household` e `Registration` do 24a; nenhuma rota nova): `VisitHousehold { id; address_short: string; property_type: string; micro_area: { id; code }; families: { id; responsible: { patient_id: string; display_name: string | null }; moved_out: boolean; responsible_exited: boolean }[] }`; `VisitHouseholdDetail extends VisitHousehold { residents: { patient_id: string; display_name: string | null; sex: PatientSex | null; family_id: string | null }[] }`; `VisitHouseholdPage { items: VisitHousehold[]; next_before: string | null }`.
  - Entrada: `HomeVisitInput { client_uuid; team_id; occurred_at; household_id?; micro_area_id?; out_of_area?; property_type?; subject: VisitSubject; patient_id?; new_patient?: QuickPatient; family_id?; duplicates_reviewed?: boolean; outcome: VisitOutcome; reasons: number[]; shared: boolean; measurements?: VisitMeasurements; note?: string }`.
  - Listas e contagens: `HomeVisitDayQuery { team_id; professional_id?; from?; to?; state?: string[]; before?: string | null }`, `HomeVisitDayPage { items: HomeVisitDay[]; next_before }`; `HomeVisitQuery { team_id; micro_area_id?; professional_id?; from?; to?; outcome?: number[]; before?: string | null }`, `HomeVisitPage { items: HomeVisit[]; next_before }`; `HomeVisitCounts { done; refused; absent; withdrawn }`.
  - Atribuição (§6): `FieldAssignment { id; micro_area: { id; code; name }; professional: { id; name; cbo }; assigned_at; assigned_by: { name } }`; `FieldAssignmentList { items: FieldAssignment[]; candidates: { professional_id; name; cbo }[] }`.
  - Quem visitar (§8): `WhoToVisitReason = { code: "care_line"; care_line: string; practice: string; status: "overdue" | "due_soon"; due_on: string } | { code: "registration_due"; status: "due_soon" | "expired"; due_on: string } | { code: "never_visited" } | { code: string }`; `WhoToVisitItem { patient_id; display_name: string | null; age: number | null; household: { id; address_short } | null; family_id: string | null; micro_area: { id; code }; last_visit_on: string | null; reasons: WhoToVisitReason[] }`; `WhoToVisitQuery { micro_area_id; reason?: string[]; before?: string | null; limit?: number }`; `WhoToVisitPage { items; next_before }`; `WhoToVisitCounts { care_line; registration_due; never_visited; people }`.
  - Atividade (§7 + D-4, D-8, D-9): `CollectiveOptions { today; min_date; teams: { id; ine; name: string | null; cbo; can_register: boolean }[]; units: { id; name; has_cnes: boolean }[]; professionals: { id; name; cbo; has_cns: boolean; team_ids: string[] }[]; domains: CollectiveDomains }`; `CollectiveParticipant { patient: VisitPatientRef; altered_evaluation: boolean; measurements: { weight_kg?: number; height_cm?: number } | null; smoking: { quit: boolean; dropped_out: boolean } | null }`; `CollectiveActivity { id; client_uuid; team: { id; ine; name: string | null }; unit: { id; name }; held_on; shift: VisitShift; activity_type: number; place: { health_unit: { id; name } } | { text: string }; participants_count; altered_count; meeting_topics?: number[]; target_audience?: number[]; health_topics?: number[]; health_practices?: number[]; procedure_code?: string | null; professionals: { id; name; cbo }[]; participants?: CollectiveParticipant[]; recorded_by: { name }; state: "recorded" | "mistaken" | string; mistaken: { at; reason: string | null; by: { name } } | null; ficha: { status: FichaSituation; reasons: string[] }; can_mark_mistaken: boolean }`; `CollectiveParticipantInput = ({ patient_id: string } | { new_patient: QuickPatient; duplicates_reviewed?: boolean }) & { altered_evaluation?: boolean; measurements?: { weight_kg?: number; height_cm?: number }; smoking?: { quit: boolean; dropped_out: boolean } }`; `CollectiveActivityInput { client_uuid; team_id; held_on; shift: VisitShift; activity_type: number; place_health_unit_id?: string; place_text?: string; participants_count: number; altered_count: number; meeting_topics: number[]; target_audience: number[]; health_topics: number[]; health_practices: number[]; procedure_code?: string; professionals: { professional_id: string }[]; participants: CollectiveParticipantInput[] }`; `CollectiveQuery { team_id; from?; to?; activity_type?: number[]; state?: string[]; before?: string | null }`; `CollectivePage { items: CollectiveActivity[]; next_before }`.
  - Aparelhos (§9.1): `FieldDevice { id; name; created_at; last_used_at: string | null; status: "active" | "inactive" | "revoked" | string (Q7); revoked_at: string | null; revoked_reason: "user" | "admin" | "membership_ended" | "user_disabled" | string | null }`; `FieldDeviceCreated { device: FieldDevice; token: string; qr_payload: string }`; `AdminFieldDevice extends FieldDevice { user: { id: string; name: string } }`.
  - Funções: `getHomeVisitContext()`, `listHomeVisitDays(q)`, `getHomeVisitDay(id)`, `closeHomeVisitDay(id)` (corpo `{ expected_state: "open" }`), `recordHomeVisit(i): Promise<{ visit: HomeVisit; day: HomeVisitDay; replayed: boolean }>` (`replayed` = status 200, por `jsonFetchWithStatus`), `getHomeVisit(id)`, `withdrawHomeVisit(id, reason)` (corpo `{ expected_state: "recorded", reason }`), `listHomeVisits(q)`, `countHomeVisits(q: { team_id; micro_area_id?; from?; to? })`, `listVisitHouseholds(microAreaId, before?)` (Q11: adapta `listHouseholds({ microAreaId, before })`), `getVisitHousehold(id)` (Q11: `getHousehold(id)` + `listRegistrations({ householdId: id })`, moradores = cadastros ativos, com `sex` do `patient`), `lookupVisitPerson(scope, doc): Promise<VisitPerson | null>`, `findVisitDuplicates(scope, q: { full_name; birth_date; mother_name? }): Promise<VisitCandidate[]>`, `listFieldAssignments(teamId)`, `createFieldAssignment(i: { micro_area_id; professional_id })`, `endFieldAssignment(id)`, `listWhoToVisit(q)`, `countWhoToVisit(microAreaId)`, `getCollectiveOptions()`, `recordCollectiveActivity(i): Promise<{ activity: CollectiveActivity; replayed: boolean }>`, `listCollectiveActivities(q)`, `getCollectiveActivity(id)`, `markCollectiveMistaken(id, reason)`, `listFieldDevices(): Promise<FieldDevice[]>`, `registerFieldDevice(name): Promise<FieldDeviceCreated>`, `revokeFieldDevice(id)`, `listAdminFieldDevices(userId?): Promise<AdminFieldDevice[]>`, `adminRevokeFieldDevice(id)`.
  - `src/test/homeVisitFixtures.ts`: `NOW24B`, `TODAY24B`, `hvUser(roles, features?)`, `VISIT_DOMAINS`, `COLLECTIVE_DOMAINS`, `acsTeam(o?)`, `nurseTeam(o?)`, `tacsTeam(o?)`, `context(o?)`, `visit(o?)`, `day(o?)`, `household(o?)`, `householdDetail(o?)`, `person(o?)`, `candidate(o?)`, `whoItem(o?)`, `options(o?)`, `activity(o?)`, `device(o?)`.

- [ ] **Step 1: Escreva as fixtures** `src/test/homeVisitFixtures.ts` (o único arquivo com corpo completo nesta task)

```ts
// Dados dos testes do módulo 24b (visita domiciliar e atividade coletiva). Agora: quinta, 2026-11-12, 10h.
// Formas do contrato §5–§9.1 e das Divergências D-1..D-20. Rótulos: pesquisa 2026-11-12 §2–§3.
import type {
  CollectiveActivity, CollectiveDomains, CollectiveOptions, FieldDevice, HomeVisit, HomeVisitContext, HomeVisitDay, HomeVisitDomains,
  HomeVisitTeam, SessionUser, VisitCandidate, VisitHousehold, VisitHouseholdDetail, VisitPerson, WhoToVisitItem
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW24B = new Date("2026-11-12T10:00:00-03:00");
export const TODAY24B = "2026-11-12";
const ALL = [ "clinical_record", "territory_registry", "home_visits", "collective_activities" ];
export const hvUser = (roles: string[] = [ "health_professional" ], features: string[] = ALL): SessionUser => sessionWith(roles, { features });

export const VISIT_DOMAINS: HomeVisitDomains = {
  motivo_visita: [
    { code: 1, label: "Cadastramento/Atualização", group: "TIPO_VISITA", group_label: "Tipo de visita" },
    { code: 29, label: "Visita periódica", group: "TIPO_VISITA", group_label: "Tipo de visita" },
    { code: 2, label: "Consulta", group: "BUSCA_ATIVA", group_label: "Busca ativa" },
    { code: 5, label: "Gestante", group: "ACOMPANHAMENTO", group_label: "Acompanhamento" },
    { code: 6, label: "Puérpera", group: "ACOMPANHAMENTO", group_label: "Acompanhamento" },
    { code: 11, label: "Hipertensão", group: "ACOMPANHAMENTO", group_label: "Acompanhamento" },
    { code: 34, label: "Ação educativa", group: "CONTROLE_AMBIENTAL_E_VETORIAL", group_label: "Controle ambiental e vetorial" },
    { code: 35, label: "Imóvel com foco", group: "CONTROLE_AMBIENTAL_E_VETORIAL", group_label: "Controle ambiental e vetorial" },
    { code: 28, label: "Outros", group: "OUTROS", group_label: "Outros" }
  ],
  desfecho: [ { code: 1, label: "Realizada" }, { code: 2, label: "Recusada" }, { code: 3, label: "Ausente" } ],
  turno: [ { code: 1, label: "Manhã" }, { code: 2, label: "Tarde" }, { code: 3, label: "Noite" } ],
  tipo_glicemia: [ { code: 0, label: "Não especificado" }, { code: 1, label: "Jejum" }, { code: 2, label: "Pós-prandial" }, { code: 3, label: "Pré-prandial" } ],
  tipo_de_imovel: [ { code: 1, label: "Domicílio" }, { code: 2, label: "Comércio" }, { code: 3, label: "Terreno baldio" }, { code: 4, label: "Ponto estratégico" },
    { code: 7, label: "Abrigo" }, { code: 99, label: "Outros" } ],
  non_residential_property_types: [ 2, 3, 4, 5, 6, 12, 13 ],
  non_residential_reasons: [ 1, 27, 28, 31, 34, 35, 36, 37 ],
  male_forbidden_reasons: [ 5, 6 ],
  measurement_ranges: {
    weight_kg: { min: 0.5, max: 500, decimals: 3 }, height_cm: { min: 20, max: 250, decimals: 1 }, systolic: { min: 0, max: 999, decimals: 0 },
    diastolic: { min: 0, max: 999, decimals: 0 }, temperature_c: { min: 20, max: 45, decimals: 1 }, glucose_mg_dl: { min: 0, max: 800, decimals: 0 }
  }
};

const R = (meeting: string, audience: string, topics: string, practices: string, participants: string) =>
  ({ meeting_topics: meeting, target_audience: audience, health_topics: topics, health_practices: practices, participants }) as never;
export const COLLECTIVE_DOMAINS: CollectiveDomains = {
  activity_types: [ { code: 1, label: "Reunião de equipe" }, { code: 4, label: "Educação em saúde" }, { code: 5, label: "Atendimento em grupo" },
    { code: 6, label: "Avaliação/procedimento coletivo" }, { code: 7, label: "Mobilização social" } ],
  meeting_topics: [ { code: 1, label: "Questões administrativas" }, { code: 6, label: "Educação permanente" } ],
  target_audience: [ { code: 1, label: "Comunidade" }, { code: 10, label: "Pessoa idosa" } ],
  health_topics: [ { code: 1, label: "Alimentação saudável" }, { code: 4, label: "Autocuidado em doenças crônicas" } ],
  health_practices: [ { code: 2, label: "Flúor tópico" }, { code: 20, label: "Antropometria" }, { code: 25, label: "PNCT sessão 1" },
    { code: 30, label: "Outro procedimento coletivo" } ],
  rules: {
    by_type: {
      "1": R("required", "forbidden", "forbidden", "forbidden", "optional"),
      "4": R("forbidden", "required", "required", "forbidden", "optional"),
      "5": R("forbidden", "required", "required", "optional", "required"),
      "6": R("forbidden", "required", "optional", "required", "required"),
      "7": R("forbidden", "required", "required", "forbidden", "optional")
    },
    health_topics_with_meeting_topic: [ 6 ], practices_only_in_type: [ { type: 6, practices: [ 2, 9, 33 ] } ],
    procedure_practices: [ 30 ], smoking_practices: [ 25, 26, 27, 28 ],
    participant_measurements: { weight_kg: { min: 0.5, max: 500, decimals: 3 }, height_cm: { min: 20, max: 250, decimals: 1 } }
  }
};

const MA01 = { id: "ma1", code: "01", name: "Vila Verde" };
export const acsTeam = (o: Partial<HomeVisitTeam> = {}): HomeVisitTeam => ({
  id: "t1", ine: "0001234567", name: "ESF Vila Verde", cbo: "515105", can_register: true, reads_whole_team: false, out_of_area_allowed: true,
  can_assign: false, can_register_device: true, micro_areas: [ { ...MA01, scope: "responsible" } ], professionals: [], ...o
});
export const tacsTeam = (o: Partial<HomeVisitTeam> = {}) =>
  acsTeam({ cbo: "322255", out_of_area_allowed: false, micro_areas: [ { id: "ma2", code: "02", name: "Jardim Sul", scope: "assigned" } ], ...o });
export const nurseTeam = (o: Partial<HomeVisitTeam> = {}) =>
  acsTeam({ cbo: "223505", can_register: false, reads_whole_team: true, out_of_area_allowed: false, can_assign: true, can_register_device: false,
    micro_areas: [ { ...MA01, scope: "team" }, { id: "ma2", code: "02", name: "Jardim Sul", scope: "team" } ],
    professionals: [ { id: "p-acs", name: "Joana ACS", cbo: "515105" }, { id: "p-tacs", name: "Rui TACS", cbo: "322255" } ], ...o });
export const context = (o: Partial<HomeVisitContext> = {}): HomeVisitContext =>
  ({ today: TODAY24B, min_date: "2026-10-01", teams: [ acsTeam() ], domains: VISIT_DOMAINS, ...o });

export const visit = (o: Partial<HomeVisit> = {}): HomeVisit => ({
  id: "v1", client_uuid: "00000000-0000-4000-8000-000000000001", day_id: "d1", occurred_at: "2026-11-12T09:15:00-03:00", shift: 1, origin: "dashboard",
  property_type: "1", micro_area: { id: "ma1", code: "01" }, out_of_area: false, household: { id: "h1", address_short: "Rua das Flores, 120" },
  family_id: "f1", subject: "resident", patient: { id: "pa1", display_name: "Maria Lima", has_cpf: true, cns_masked: null },
  outcome: 1, reasons: [ 29, 11 ], shared: false, measurements: { systolic: 130, diastolic: 85 }, state: "recorded", withdrawn: null,
  recorded_by: { name: "Joana ACS" }, can_withdraw: true, ...o
});
export const day = (o: Partial<HomeVisitDay> = {}): HomeVisitDay => ({
  id: "d1", professional: { id: "p-acs", name: "Joana ACS" }, team: { id: "t1", ine: "0001234567", name: "ESF Vila Verde" }, day: TODAY24B,
  sequence: 1, state: "open", closed_at: null, closed_by: null, visits_count: 1, withdrawn_count: 0, ficha: { status: "open", reasons: [] },
  blockers: [], own: true, visits: [ visit() ], ...o
});

export const household = (o: Partial<VisitHousehold> = {}): VisitHousehold => ({
  id: "h1", address_short: "Rua das Flores, 120", property_type: "1", micro_area: { id: "ma1", code: "01" },
  families: [ { id: "f1", responsible: { patient_id: "pa2", display_name: "José Lima" }, moved_out: false, responsible_exited: false } ], ...o
});
export const householdDetail = (o: Partial<VisitHouseholdDetail> = {}): VisitHouseholdDetail => ({
  ...household(), residents: [ { patient_id: "pa1", display_name: "Maria Lima", sex: "female", family_id: "f1" },
    { patient_id: "pa2", display_name: "José Lima", sex: "male", family_id: "f1" } ], ...o
});
export const person = (o: Partial<VisitPerson> = {}): VisitPerson =>
  ({ patient_id: "pa9", display_name: "Ana Prado", birth_date: "1990-04-02", has_cpf: true, cns_masked: null, sex: "female", ...o });
export const candidate = (o: Partial<VisitCandidate> = {}): VisitCandidate =>
  ({ patient_id: "pa8", display_name: "Ana Prado", birth_date: "1990-04-02", has_cpf: false, has_cns: true, sex: "female", ...o });

export const whoItem = (o: Partial<WhoToVisitItem> = {}): WhoToVisitItem => ({
  patient_id: "pa1", display_name: "Maria Lima", age: 67, household: { id: "h1", address_short: "Rua das Flores, 120" }, family_id: "f1",
  micro_area: { id: "ma1", code: "01" }, last_visit_on: "2026-09-30",
  reasons: [ { code: "care_line", care_line: "Pessoa com hipertensão", practice: "Visitas do agente de saúde", status: "overdue", due_on: "2026-11-01" },
    { code: "registration_due", status: "due_soon", due_on: "2026-12-20" } ], ...o
});

export const options = (o: Partial<CollectiveOptions> = {}): CollectiveOptions => ({
  today: TODAY24B, min_date: "2026-10-01",
  teams: [ { id: "t1", ine: "0001234567", name: "ESF Vila Verde", cbo: "223505", can_register: true } ],
  units: [ { id: "u1", name: "UBS Vila Verde", has_cnes: true }, { id: "u2", name: "Posto Novo", has_cnes: false } ],
  professionals: [ { id: "p-acs", name: "Joana ACS", cbo: "515105", has_cns: true, team_ids: [ "t1" ] },
    { id: "p-sem", name: "Caio Sem CNS", cbo: "515105", has_cns: false, team_ids: [ "t1" ] } ],
  domains: COLLECTIVE_DOMAINS, ...o
});
export const activity = (o: Partial<CollectiveActivity> = {}): CollectiveActivity => ({
  id: "ca1", client_uuid: "00000000-0000-4000-8000-0000000000c1", team: { id: "t1", ine: "0001234567", name: "ESF Vila Verde" },
  unit: { id: "u1", name: "UBS Vila Verde" }, held_on: TODAY24B, shift: 1, activity_type: 5, place: { health_unit: { id: "u1", name: "UBS Vila Verde" } },
  participants_count: 12, altered_count: 1, meeting_topics: [], target_audience: [ 10 ], health_topics: [ 4 ], health_practices: [ 20 ], procedure_code: null,
  professionals: [ { id: "p-acs", name: "Joana ACS", cbo: "515105" } ],
  participants: [ { patient: { id: "pa1", display_name: "Maria Lima", has_cpf: true, cns_masked: null }, altered_evaluation: true,
    measurements: { weight_kg: 72.5, height_cm: 160 }, smoking: null } ],
  recorded_by: { name: "Rita Enfermeira" }, state: "recorded", mistaken: null, ficha: { status: "pending", reasons: [] }, can_mark_mistaken: true, ...o
});
export const device = (o: Partial<FieldDevice> = {}): FieldDevice =>
  ({ id: "fd1", name: "Celular da Joana", created_at: "2026-11-10T08:00:00-03:00", last_used_at: null, status: "active", revoked_at: null, revoked_reason: null, ...o });
```

- [ ] **Step 2: Escreva os testes do cliente** `src/lib/api.homeVisits.test.ts` (mesmo `fetch` falso de `src/lib/api.ombudsman.test.ts`: `stub`, `last`, `bodyOf`, `calls`)

```ts
// O que nunca pode ir na URL: nome, CPF, CNS, nascimento, motivo, medida, observação, motivo da retirada e o token.
const SECRET_IN_URL = /52998224725|529\.982|898000|Maria|Ana|1990-04|pressão|losartana|rsf_|retirad|engano/i;

it("folha e visitas: só ids, datas, filtros com [] e cursor na URL", async () => {
  stub({ items: [ day() ], next_before: "c2" });
  const p = await listHomeVisitDays({ team_id: "t1", from: "2026-11-01", to: TODAY24B, state: [ "open", "closed" ], before: "c1" });
  expect(last().url).toBe("/home_visits/days?team_id=t1&from=2026-11-01&to=2026-11-12&state%5B%5D=open&state%5B%5D=closed&before=c1");
  expect(p.next_before).toBe("c2");
  stub({ items: [], next_before: null });
  await listHomeVisits({ team_id: "t1", micro_area_id: "ma1", outcome: [ 2, 3 ] });
  expect(last().url).toBe("/home_visits?team_id=t1&micro_area_id=ma1&outcome%5B%5D=2&outcome%5B%5D=3");
  stub({ done: 3, refused: 1, absent: 0, withdrawn: 1 });
  await countHomeVisits({ team_id: "t1", micro_area_id: "ma1" });
  expect(last().url).toBe("/home_visits/count?team_id=t1&micro_area_id=ma1");
});
it("registrar: corpo inteiro no POST; 201 → replayed false; 200 → replayed true", async () => {
  stub({ visit: visit(), day: day() }, 201);
  const input = { client_uuid: "u-1", team_id: "t1", occurred_at: "2026-11-12T09:15:00-03:00", household_id: "h1", subject: "resident" as const,
    patient_id: "pa1", outcome: 1 as const, reasons: [ 29 ], shared: false, measurements: { systolic: 130, diastolic: 85 }, note: "Pediu retorno." };
  expect((await recordHomeVisit(input)).replayed).toBe(false);
  expect(last()).toMatchObject({ url: "/home_visits", init: { method: "POST" } });
  expect(bodyOf()).toEqual(input);
  stub({ visit: visit(), day: day() }, 200);
  expect((await recordHomeVisit(input)).replayed).toBe(true);
});
it("fechar, retirar e engano levam expected_state no corpo", async () => {
  stub(day({ state: "closed" }));
  await closeHomeVisitDay("d1");
  expect(last().url).toBe("/home_visits/days/d1/close");
  expect(bodyOf()).toEqual({ expected_state: "open" });
  stub(visit({ state: "withdrawn" }));
  await withdrawHomeVisit("v1", "Registrei na casa errada.");
  expect(last().url).toBe("/home_visits/v1/withdraw");
  expect(bodyOf()).toEqual({ expected_state: "recorded", reason: "Registrei na casa errada." });
  stub(activity({ state: "mistaken" }));
  await markCollectiveMistaken("ca1", "Atividade lançada em duplicidade.");
  expect(last().url).toBe("/collective_activities/ca1/mark_mistaken");
  expect(bodyOf()).toEqual({ expected_state: "recorded", reason: "Atividade lançada em duplicidade." });
  for (const c of calls) expect(c.url).not.toMatch(SECRET_IN_URL);
});
it("pessoa: busca e duplicados por POST, no escopo de quem chama; null = ninguém", async () => {
  stub(null);
  expect(await lookupVisitPerson("home_visits", { cpf: "52998224725" })).toBeNull();
  expect(last()).toMatchObject({ url: "/home_visits/people/lookup", init: { method: "POST" } });
  expect(bodyOf()).toEqual({ cpf: "52998224725" });
  stub({ candidates: [ candidate() ] });
  await findVisitDuplicates("collective_activities", { full_name: "Ana Prado", birth_date: "1990-04-02" });
  expect(last().url).toBe("/collective_activities/people/duplicates");
  for (const c of calls) expect(c.url).not.toMatch(SECRET_IN_URL);
});
it("quem visitar: microárea, motivos com [] e limite; contagem sem pessoa", async () => {
  stub({ items: [ whoItem() ], next_before: null });
  await listWhoToVisit({ micro_area_id: "ma1", reason: [ "care_line" ], limit: 50 });
  expect(last().url).toBe("/home_visits/who_to_visit?micro_area_id=ma1&reason%5B%5D=care_line&limit=50");
  stub({ care_line: 1, registration_due: 2, never_visited: 3, people: 5 });
  await countWhoToVisit("ma1");
  expect(last().url).toBe("/home_visits/who_to_visit/count?micro_area_id=ma1");
});
it("atribuição e aparelhos pelos caminhos do §6 e §9.1; o token volta só na resposta", async () => {
  stub({ id: "as1" }, 201);
  await createFieldAssignment({ micro_area_id: "ma2", professional_id: "p-tacs" });
  expect(last()).toMatchObject({ url: "/home_visits/assignments", init: { method: "POST" } });
  stub({ id: "as1" });
  await endFieldAssignment("as1");
  expect(last()).toMatchObject({ url: "/home_visits/assignments/as1/end", init: { method: "POST", body: "{}" } });
  stub({ device: device(), token: "rsf_abc", qr_payload: "rotasaude-field:v1:eyJ9" }, 201);
  const created = await registerFieldDevice("Celular da Joana");
  expect(created.token).toBe("rsf_abc");
  expect(bodyOf()).toEqual({ name: "Celular da Joana" });
  stub({ id: "fd1" });
  await revokeFieldDevice("fd1");
  expect(last()).toMatchObject({ url: "/field_devices/fd1/revoke", init: { method: "POST", body: "{}" } });
  stub({ items: [] });
  await listAdminFieldDevices("u-7");
  expect(last().url).toBe("/field_devices/admin?user_id=u-7");
  stub({ id: "fd1" });
  await adminRevokeFieldDevice("fd1");
  expect(last().url).toBe("/field_devices/admin/fd1/revoke");
  for (const c of calls) expect(c.url).not.toMatch(SECRET_IN_URL);
});
it("domicílios da visita pelas rotas de leitura do 24a (Q11)", async () => {
  stub({ households: [], next_before: null });
  await listVisitHouseholds("ma2", "c1");
  expect(last().url).toMatch(/^\/territory\/households\?micro_area_id=ma2&before=c1/);
  stub({ registrations: [] });
  await getVisitHousehold("h1");
  const urls = calls.map((c) => c.url);
  expect(urls).toContain("/territory/households/h1");
  expect(urls).toContain("/territory/registrations?household_id=h1");
  expect(urls.some((u) => u.startsWith("/home_visits/households"))).toBe(false);
});
it("atividade: opções, lista com filtros e gravação idempotente", async () => {
  stub(options());
  await getCollectiveOptions();
  expect(last().url).toBe("/collective_activities/options");
  stub({ items: [], next_before: null });
  await listCollectiveActivities({ team_id: "t1", activity_type: [ 5 ], state: [ "mistaken" ] });
  expect(last().url).toBe("/collective_activities?team_id=t1&activity_type%5B%5D=5&state%5B%5D=mistaken");
  stub(activity(), 200);
  expect((await recordCollectiveActivity({ client_uuid: "u-2", team_id: "t1" } as never)).replayed).toBe(true);
});
```

E em `src/lib/features.test.ts`: `featureLabel("home_visits")` é `"Visita domiciliar e territorial"` e `featureLabel("collective_activities")` é `"Atividade coletiva"` (contrato §1, rótulos curtos).

- [ ] **Step 3: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/api.homeVisits.test.ts src/lib/features.test.ts`
Expected: FAIL (funções e rótulos inexistentes).

- [ ] **Step 4: Implemente**

- `api.ts`, seção `// ─── Visita domiciliar e atividade coletiva (módulo 24b; ADR 0052; contrato 2026-11-12 §5–§9.1) ───` no fim, com o comentário de topo no molde do 33 (na URL só ids, datas, filtros codificados, cursor e limite; pessoa, motivos, medidas, observação, temas, motivos de retirada e engano e o token só no corpo; listas em `{ items, next_before }`; toda escrita com corpo). Bases `HOME_VISITS_BASE = import.meta.env.VITE_HOME_VISITS_BASE || "/home_visits"`, `COLLECTIVE_BASE = … || "/collective_activities"`, `FIELD_DEVICES_BASE = … || "/field_devices"`; `hvPath(...parts)` com `encodeURIComponent` e `hvWrite(method, body = {})` no molde de `territoryPath`/`territoryWrite`. A query usa `URLSearchParams`: os escalares preenchidos na ordem do contrato e `append("state[]", v)`/`append("outcome[]", String(v))`/`append("activity_type[]", …)`/`append("reason[]", v)` para as listas.
- `recordHomeVisit` e `recordCollectiveActivity` usam `jsonFetchWithStatus` e devolvem `replayed: status === 200`. O corpo vai como veio (a tela já tirou o opcional vazio).
- `lookupVisitPerson(scope, doc)`: `POST /<scope>/people/lookup` com `doc`; `404 person_not_found` → `null`; `409 citizen_not_verified` sobe como erro (frase) (Q8).
- Listas desembrulham `{ items }` (ausente = vazio, `next_before ?? null`); `listFieldAssignments` devolve `{ items, candidates }` com os dois vazios por padrão.
- `features.ts`: as duas chaves em `FeatureKey` e `FEATURE_LABEL`, com o comentário "Módulo 24b (ADR 0052): home_visits exige territory_registry utilizável; collective_activities exige clinical_record".
- `vite.config.ts`: `"^/home_visits(/|$)": proxy(TARGET)`, `"^/collective_activities(/|$)": proxy(TARGET)` e `"^/field_devices(/|$)": proxy(TARGET)`, e as três linhas no comentário do topo (`/home_visits → folha do dia, visitas, atribuição e quem visitar (módulo 24b)`, `/collective_activities → atividade coletiva (módulo 24b)`, `/field_devices → aparelhos de campo (módulo 24b)`). `/field/v1` **não** entra (é do app de campo).

- [ ] **Step 5: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/api.homeVisits.test.ts src/lib/features.test.ts src/lib/api.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.homeVisits.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/homeVisitFixtures.ts
$G commit -m "feat: add the home visit and collective activity API client, feature labels and dev proxy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 1 · cliente HTTP do 24b e fixtures · formas que o api contrariou (D-1..D-9, D-12)`.

---

### Task 2: Regras de tela da visita

**Files:**
- Create: `src/lib/homeVisits.ts`
- Test: `src/lib/homeVisits.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `hasFeature`, `featureDisabledKey` (`src/lib/features.ts`); `canUseTerritory` (`src/lib/territoryRegistry.ts`); `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `fmtDay` (`src/lib/careLines.ts`); `Problem`, `Refusal` (`src/lib/pharmacy.ts`); `newPatientProblem`, `DOMAINS` (`src/lib/territoryDictionaries.ts`); `ApiError`, `errorCode` (`src/lib/api.ts`); `cityLocalIso` (`src/lib/format.ts`).
- Produces (em `src/lib/homeVisits.ts`):
  - Frases: `HOME_VISITS_DISABLED = "a visita domiciliar está desligada nesta cidade"`, `COLLECTIVE_DISABLED = "a atividade coletiva está desligada nesta cidade"`, `HV_NO_ROLE = "seu papel não permite usar a visita domiciliar"`, `HV_CANNOT_REGISTER = "seu CBO não registra visita — enfermeiro e médico leem as visitas da equipe"`, `DAY_CHANGED = "a folha mudou enquanto a tela estava aberta — os dados foram relidos; confira antes de repetir"`, `VISIT_CHANGED = "a visita mudou enquanto a tela estava aberta — os dados foram relidos"`, `REPLAYED_VISIT = "Esta visita já estava gravada."`, `OTHER_DAY_TEXT = (d) => "Visita registrada na folha de " + fmtDay(d.day) + (d.sequence > 1 ? " (complementar " + d.sequence + ")" : "")`, `CLOSE_EFFECT = "Depois de fechar, as visitas deste dia não mudam e a ficha de visita vai ao e-SUS."`, `NETWORK_RETRY = "não foi possível confirmar a gravação — tente de novo: a mesma visita não é gravada duas vezes"`.
  - Chaves: `HV_CONTEXT_KEY = "hvContext"`, `HV_TODAY_KEY = "hvToday"`, `HV_DAYS_KEY`, `HV_DAY_KEY`, `HV_VISITS_KEY`, `HV_VISIT_KEY`, `HV_COUNT_KEY`, `HV_HOUSEHOLDS_KEY`, `HV_HOUSEHOLD_KEY`, `HV_ASSIGNMENTS_KEY`, `WHO_KEY`, `WHO_COUNT_KEY`, `CA_OPTIONS_KEY`, `CA_LIST_KEY`, `CA_DETAIL_KEY`, `FD_KEY`, `FD_ADMIN_KEY`; reexporta `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `fmtDay`.
  - Portões: `canUseHomeVisits(user)`, `canUseCollectiveActivities(user)`, `hvRoles(user): { professional: boolean; admin: boolean }`.
  - Rótulos (cru quando desconhecido): `outcomeLabel`, `shiftLabel`, `subjectLabel` (morador, família — documento do responsável, imóvel sem pessoa), `originLabel` (dashboard, app de campo), `dayStateLabel`, `closedByLabel` (por você/pelo profissional, automático de madrugada), `fichaLabel(f)` → `{ text, tone }` (`open` "folha aberta", `empty` "sem visita válida — sem ficha", `not_generated` "ficha não gerada: <motivos>", e os do `FICHA_STATUS` de `src/lib/production.ts`), `reasonLabel(domains, code)`, `propertyLabel(domains, code)`, `glucoseTypeLabel`, `revokedReasonLabel` (por você, pelo administrador, fim do vínculo na equipe, usuário desativado), `whoReasonText(r)` ("Pessoa com hipertensão · Visitas do agente de saúde · vencida em 01/11/2026"; "cadastro vence em 20/12/2026"; "nunca visitada").
  - `BLOCKER_TEXT` e `blockerText(code)`: `professional_without_cns` "Seu CNS não está no cadastro de profissionais", `unit_without_cnes` "A unidade da equipe não tem CNES", `cbo_not_in_ledi_table` "Seu CBO não está na tabela de quem registra visita no e-SUS", `city_profile_incomplete` "O perfil da cidade está incompleto (UF ou código IBGE)", `siaps_deadline_passed` "O prazo do SIAPS para esta competência já passou", `patient_without_identifier` "Pessoa visitada sem CPF, sem CNS e sem justificativa", `citizen_without_birth_date` "Pessoa visitada sem data de nascimento", `citizen_without_sex` "Pessoa visitada sem sexo no cadastro" (Q4).
  - Regras da visita (§3, lidas de `domains`): `isNonResidential(domains, propertyType)`, `reasonsFor(domains, { propertyType, outcome, sex }): { code; label; group; group_label; disabled: string | null }[]` (agrupados na ordem do domínio; recusada/ausente → lista vazia; não residencial → só `non_residential_reasons`; `male` → 5 e 6 com `disabled` "não se aplica a pessoa do sexo masculino"), `measurementsAllowed(domains, { propertyType, outcome })`, `parseMeasure(raw): number | null | undefined` (vírgula ou ponto; `""` = null; inválido = undefined), `measurementsProblems(domains, draft): Problem[]` (faixa, casas, par pressão, par glicemia), `occurredAt(day, time): string | null` (por `cityLocalIso`), `shiftPreview(time)`.
  - `VisitDraft` (o que o formulário guarda) e `visitProblems(draft, ctx: { domains; today; minDate; team }): Problem[]` (**todas** de uma vez, com o `path` do api: `occurred_at`, `property_type`, `micro_area_id`, `subject`, `patient_id`, `new_patient.<campo>`, `family_id`, `outcome`, `reasons[i]`, `measurements.<chave>`, `note`); `visitBody(draft): Omit<HomeVisitInput, "client_uuid" | "team_id">` (tira opcional vazio; nunca manda `property_type` e `micro_area_id` com `household_id`; `out_of_area` só `true`).
  - `quickPatientProblems(p, today): Problem[]` (envolve `newPatientProblem`, convertendo `message` → `text`), `textProblem(text, { min, max, path })` (code points depois do `trim`).
  - Erros: `HOME_VISIT_ERRORS: Record<string, string>`, `homeVisitError(err): string`, `homeVisitRefusals(err): Refusal[]` (uma recusa com o `path` do corpo do api; sem `path`, `""`), `isDayStale(err)` (`day_changed`, `day_closed`, `visit_changed`, `not_found`), `isNetworkOrServer(err)` (não `ApiError`, ou `status >= 500`).

- [ ] **Step 1: Escreva os testes** `src/lib/homeVisits.test.ts`

```ts
const EVERY_CODE = [
  // herdados (contrato, convenções)
  "feature_disabled", "mfa_required", "not_found", "invalid_request", "invalid_cursor", "invalid_limit", "too_many_requests", "payload_too_large",
  // §13
  "cbo_not_allowed", "out_of_scope", "invalid_home_visit", "visit_date_out_of_range", "non_residential_with_person", "reasons_forbidden",
  "reason_not_allowed", "measurements_forbidden", "patient_in_other_household", "family_without_responsible", "day_changed", "day_closed",
  "visit_changed", "invalid_collective_activity", "activity_date_out_of_range", "duplicate_participant", "professional_without_cns",
  "activity_changed", "duplicate_suspected", "cpf_taken", "cns_taken", "already_assigned", "device_limit",
  // Q21
  "household_inactive", "family_moved_out", "micro_area_inactive", "assignment_already_ended", "device_already_revoked",
  // do 24a, que a busca e o registro rápido devolvem
  "invalid_cpf", "invalid_cns", "identity_locked", "person_not_found", "citizen_not_verified"
];
it("todo código tem frase própria; código novo aparece cru; 403 feature_disabled vira a frase do interruptor", () => {
  for (const code of EVERY_CODE) {
    expect(HOME_VISIT_ERRORS[code], code).toBeTruthy();
    expect(homeVisitError(new ApiError(422, { error: code }, "x"))).toBe(HOME_VISIT_ERRORS[code]);
  }
  expect(homeVisitError(new ApiError(422, { error: "brand_new" }, "x"))).toBe("recusado pelo servidor (brand_new)");
  expect(homeVisitError(new ApiError(403, { error: "feature_disabled", feature: "home_visits" }, "x"))).toBe(HOME_VISITS_DISABLED);
  expect(homeVisitError(new ApiError(403, { error: "feature_disabled", feature: "collective_activities" }, "x"))).toBe(COLLECTIVE_DISABLED);
});
it("frases fixas dos códigos que mudam a tela", () => {
  expect(HOME_VISIT_ERRORS.day_changed).toBe(DAY_CHANGED);
  expect(HOME_VISIT_ERRORS.day_closed).toBe("esta folha já foi fechada — a visita não pode mais ser retirada");
  expect(HOME_VISIT_ERRORS.out_of_scope).toBe("esta microárea não está no seu escopo");
  expect(HOME_VISIT_ERRORS.cbo_not_allowed).toBe("seu CBO não permite esta ação");
  expect(HOME_VISIT_ERRORS.device_limit).toBe("você já tem 3 aparelhos ativos — revogue um antes de cadastrar outro");
  expect(blockerText("professional_without_cns")).toBe("Seu CNS não está no cadastro de profissionais");
  expect(blockerText("novo_codigo")).toBe("novo_codigo");
});
it("portões: home_visits exige o cadastro territorial e o prontuário; atividade, só o prontuário", () => {
  expect(canUseHomeVisits(hvUser())).toBe(true);
  expect(canUseHomeVisits(hvUser(undefined, [ "clinical_record", "home_visits" ]))).toBe(false);
  expect(canUseHomeVisits({ ...hvUser(), operator: true })).toBe(false);
  expect(canUseCollectiveActivities(hvUser(undefined, [ "clinical_record", "collective_activities" ]))).toBe(true);
  expect(canUseCollectiveActivities(hvUser(undefined, [ "collective_activities" ]))).toBe(false);
});
it("motivos: recusada e ausente sem motivo; não residencial só os do domínio; 5 e 6 indisponíveis para male", () => {
  expect(reasonsFor(VISIT_DOMAINS, { propertyType: "1", outcome: 2, sex: null })).toEqual([]);
  expect(reasonsFor(VISIT_DOMAINS, { propertyType: "2", outcome: 1, sex: null }).map((r) => r.code)).toEqual([ 1, 34, 35, 28 ]);
  const male = reasonsFor(VISIT_DOMAINS, { propertyType: "1", outcome: 1, sex: "male" });
  expect(male.find((r) => r.code === 5)!.disabled).toBe("não se aplica a pessoa do sexo masculino");
  expect(male.find((r) => r.code === 11)!.disabled).toBeNull();
  expect(male.map((r) => r.group_label)).toContain("Controle ambiental e vetorial");
});
it("medidas: só em realizada residencial; faixa, casas e os pares", () => {
  expect(measurementsAllowed(VISIT_DOMAINS, { propertyType: "1", outcome: 1 })).toBe(true);
  expect(measurementsAllowed(VISIT_DOMAINS, { propertyType: "4", outcome: 1 })).toBe(false);
  expect(measurementsAllowed(VISIT_DOMAINS, { propertyType: "1", outcome: 3 })).toBe(false);
  expect(parseMeasure("72,5")).toBe(72.5);
  expect(parseMeasure("")).toBeNull();
  expect(parseMeasure("7a")).toBeUndefined();
  expect(measurementsProblems(VISIT_DOMAINS, { weight_kg: "0,4", height_cm: "160,25", systolic: "130", diastolic: "", temperature_c: "", glucose_mg_dl: "110", glucose_type: "" })
    .map((p) => p.path).sort()).toEqual([ "measurements.diastolic", "measurements.glucose_type", "measurements.height_cm", "measurements.weight_kg" ]);
});
it("visitProblems: todas as recusas de uma vez, com o path do api", () => {
  const ctx = { domains: VISIT_DOMAINS, today: TODAY24B, minDate: "2026-10-01", team: acsTeam() };
  const problems = visitProblems({ ...emptyVisitDraft(), day: "2026-09-30", time: "09:00", place: { kind: "no_household", propertyType: "", microAreaId: null, outOfArea: false },
    outcome: null, note: "x".repeat(501) }, ctx);
  expect(problems.map((p) => p.path).sort()).toEqual([ "micro_area_id", "note", "occurred_at", "outcome", "property_type" ]);
  const nonRes = visitProblems({ ...emptyVisitDraft(), place: { kind: "no_household", propertyType: "2", microAreaId: "ma1", outOfArea: false },
    subject: "resident", person: { kind: "patient", patientId: "pa1", sex: null }, outcome: 1, reasons: [ 5 ] }, ctx);
  expect(nonRes.map((p) => p.path)).toEqual(expect.arrayContaining([ "subject", "reasons[0]" ]));
  const tacs = visitProblems({ ...emptyVisitDraft(), place: { kind: "no_household", propertyType: "1", microAreaId: null, outOfArea: true }, outcome: 3 },
    { ...ctx, team: tacsTeam() });
  expect(tacs.find((p) => p.path === "out_of_area")!.text).toBe("fora de área é só para o ACS");
});
it("visitBody: com domicílio não manda imóvel nem microárea; opcional vazio não vai", () => {
  const body = visitBody({ ...emptyVisitDraft(), day: TODAY24B, time: "09:15", place: { kind: "household", householdId: "h1", propertyType: "1", microAreaId: "ma1" },
    subject: "family_responsible", familyId: "f1", outcome: 1, reasons: [ 29 ], shared: true, measurements: { ...EMPTY_MEASURES, systolic: "130", diastolic: "85" }, note: "  " });
  expect(body).toEqual({ occurred_at: "2026-11-12T09:15:00-03:00", household_id: "h1", subject: "family_responsible", family_id: "f1",
    outcome: 1, reasons: [ 29 ], shared: true, measurements: { systolic: 130, diastolic: 85 } });
});
it("recusa do api: path do corpo vai ao campo; sem path, à frase geral; rede e 5xx guardam o uuid", () => {
  expect(homeVisitRefusals(new ApiError(422, { error: "reason_not_allowed", path: "reasons[1]" }, "x"))).toEqual(
    [ { code: "reason_not_allowed", path: "reasons[1]", text: HOME_VISIT_ERRORS.reason_not_allowed } ]);
  expect(isNetworkOrServer(new TypeError("Failed to fetch"))).toBe(true);
  expect(isNetworkOrServer(new ApiError(503, {}, "x"))).toBe(true);
  expect(isNetworkOrServer(new ApiError(422, { error: "invalid_home_visit" }, "x"))).toBe(false);
});
```

(`emptyVisitDraft` e `EMPTY_MEASURES` são exportados de `src/lib/homeVisits.ts`; os rascunhos dos testes só trocam o que importa.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/homeVisits.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `VisitDraft = { day: string; time: string; place: { kind: "household"; householdId; propertyType; microAreaId } | { kind: "no_household"; propertyType: string; microAreaId: string | null; outOfArea: boolean } | null; subject: VisitSubject | null; person: { kind: "patient"; patientId: string; sex: PatientSex | null } | { kind: "new"; patient: QuickPatient; duplicatesReviewed: boolean } | null; familyId: string | null; outcome: VisitOutcome | null; reasons: number[]; shared: boolean; measurements: Record<MeasurementKey | "glucose_type", string>; note: string }`.
- `visitProblems` confere, nesta ordem e sem parar no primeiro: data entre `minDate` e `today` (e hora não futura no dia de hoje → `occurred_at`, "a data da visita vai de <min> até agora"); lugar escolhido; sem domicílio, tipo de imóvel e microárea ou fora de área; fora de área sem `team.out_of_area_allowed` → `out_of_area`; não residencial com pessoa → `subject` ("imóvel não residencial não leva dados de pessoa"); residencial com `resident` sem pessoa → `patient_id`; `family_responsible` sem família → `family_id`; desfecho; motivo que `reasonsFor` não oferece (ou `disabled`) → `reasons[i]`; motivo em recusada/ausente → `reasons`; medida onde `measurementsAllowed` é falso → `measurements`; `measurementsProblems`; `quickPatientProblems` do registro rápido; observação até 500 code points.
- `homeVisitRefusals` lê `{ error, path }` (o 24b não manda `problems[]`; se mandar, lê todos — mesmo molde do `ombudsmanRefusals`).
- O turno nunca vai no corpo (§3); `shiftPreview` só mostra "Manhã/Tarde/Noite" ao lado da hora.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/homeVisits.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/homeVisits.ts src/lib/homeVisits.test.ts
$G commit -m "feat: add home visit screen rules mirrored from the visit domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 2 · regras da visita e frases de todos os códigos · domínios da visita no contexto (D-1)`.

---

### Task 3: Regras de tela da atividade coletiva (lidas do dado)

**Files:**
- Create: `src/lib/collectiveActivities.ts`
- Test: `src/lib/collectiveActivities.test.ts`

**Interfaces:**
- Consumes: tipos da Task 1; `HOME_VISIT_ERRORS`, `homeVisitRefusals`, `parseMeasure`, `quickPatientProblems`, `textProblem` (Task 2); `Problem` (`src/lib/pharmacy.ts`).
- Produces (em `src/lib/collectiveActivities.ts`):
  - `fieldRule(domains, type, field): FieldRule` (tipo sem regra → `"optional"`, e a tela mostra o tipo cru).
  - `offeredHealthTopics(domains, draft)` (nunca o tema 18 — o api não o põe no domínio, D-2; se vier, a tela o filtra); `healthTopicsAllowed(domains, draft)` (regra do tipo, ou `forbidden` com exceção quando um tema de reunião está em `health_topics_with_meeting_topic`); `offeredPractices(domains, type)` (tira `practices_only_in_type` de outro tipo); `needsProcedure(domains, practices)`; `smokingAllowed(domains, practices)`.
  - `ActivityDraft` e `activityProblems(draft, ctx: { options; today }): Problem[]` (todas de uma vez, `path` do api: `held_on`, `shift`, `activity_type`, `place_health_unit_id`, `place_text`, `participants_count`, `altered_count`, `meeting_topics`, `target_audience`, `health_topics`, `health_topics[i]`, `health_practices`, `health_practices[i]`, `procedure_code`, `professionals[i]`, `participants`, `participants[i].new_patient.<campo>`, `participants[i].measurements.<chave>`, `participants[i].smoking`); `activityBody(draft): Omit<CollectiveActivityInput, "client_uuid">` (só o que a regra do tipo permite; campo `forbidden` vai como `[]`; `altered_count` = participantes com `altered_evaluation` quando há lista).
  - `participantWithoutId(p)` e `IDENTIFIER_WARNING = "participantes sem CPF e sem CNS ficam fora da ficha e seguem contados no número de participantes"` (C9; nos tipos com `participants: "required"`, a mesma falta vira problema: "nos tipos 5 e 6 todo participante precisa de CPF ou CNS").
  - `COLLECTIVE_NOT_GENERATED: Record<string, string>` (os nove do §11 e o `participant_without_sex` da D-10).

- [ ] **Step 1: Escreva os testes** `src/lib/collectiveActivities.test.ts`

```ts
it("a regra vem do dado: trocar o by_type muda o que a tela pede", () => {
  const d = COLLECTIVE_DOMAINS;
  expect(fieldRule(d, 1, "meeting_topics")).toBe("required");
  expect(fieldRule(d, 5, "participants")).toBe("required");
  const changed = { ...d, rules: { ...d.rules, by_type: { ...d.rules.by_type, "4": { ...d.rules.by_type["4"], participants: "required" as const } } } };
  expect(fieldRule(changed, 4, "participants")).toBe("required");
  expect(fieldRule(d, 9, "participants")).toBe("optional");
});
it("temas para saúde nas reuniões só com o tema de reunião da exceção; tema 18 nunca", () => {
  const d = COLLECTIVE_DOMAINS;
  expect(healthTopicsAllowed(d, { ...emptyActivityDraft(), activityType: 1, meetingTopics: [ 1 ] })).toBe("forbidden");
  expect(healthTopicsAllowed(d, { ...emptyActivityDraft(), activityType: 1, meetingTopics: [ 6 ] })).toBe("optional");
  const with18 = { ...d, health_topics: [ ...d.health_topics, { code: 18, label: "Semana saúde na escola" } ] };
  expect(offeredHealthTopics(with18, { ...emptyActivityDraft(), activityType: 4 }).map((t) => t.code)).not.toContain(18);
});
it("práticas: só do tipo 6 ficam fora dos outros; 30 pede SIGTAP; tabagismo só nas práticas da regra", () => {
  const d = COLLECTIVE_DOMAINS;
  expect(offeredPractices(d, 5).map((p) => p.code)).not.toContain(2);
  expect(offeredPractices(d, 6).map((p) => p.code)).toContain(2);
  expect(needsProcedure(d, [ 20, 30 ])).toBe(true);
  expect(smokingAllowed(d, [ 20 ])).toBe(false);
  expect(smokingAllowed(d, [ 25 ])).toBe(true);
});
it("activityProblems: tipo 5 sem participante, sem público, SIGTAP sem a 30, contagens e local", () => {
  const p = activityProblems({ ...emptyActivityDraft(), activityType: 5, place: { kind: "text", text: "" }, participantsCount: "0", alteredCount: "3",
    healthTopics: [ 4 ], procedureCode: "0101010010", participants: [] }, { options: options(), today: TODAY24B });
  expect(p.map((x) => x.path).sort()).toEqual([ "altered_count", "participants", "participants_count", "place_text", "procedure_code", "target_audience" ]);
});
it("activityProblems: participante sem CPF e sem CNS — problema no tipo 5, aviso no tipo 4", () => {
  const anon = { kind: "patient" as const, patientId: "pa8", displayName: "Ana Prado", hasCpf: false, hasCns: false, altered: false, weight: "", height: "", smoking: null };
  const base = { ...emptyActivityDraft(), place: { kind: "unit" as const, unitId: "u1" }, participantsCount: "1", targetAudience: [ 10 ], healthTopics: [ 4 ], participants: [ anon ] };
  expect(activityProblems({ ...base, activityType: 5 }, { options: options(), today: TODAY24B }).map((x) => x.path)).toContain("participants[0]");
  expect(activityProblems({ ...base, activityType: 4 }, { options: options(), today: TODAY24B })).toEqual([]);
  expect(participantWithoutId(anon)).toBe(true);
});
it("activityBody: forbidden vai como lista vazia; altered_count conta a lista; local é um só", () => {
  const body = activityBody({ ...emptyActivityDraft(), teamId: "t1", heldOn: TODAY24B, shift: 1, activityType: 5, place: { kind: "unit", unitId: "u1" },
    participantsCount: "12", alteredCount: "", targetAudience: [ 10 ], healthTopics: [ 4 ], meetingTopics: [ 6 ], professionals: [ "p-acs" ],
    participants: [ { kind: "patient", patientId: "pa1", displayName: "Maria Lima", hasCpf: true, hasCns: false, altered: true, weight: "72,5", height: "160", smoking: null } ] });
  expect(body).toEqual({ team_id: "t1", held_on: TODAY24B, shift: 1, activity_type: 5, place_health_unit_id: "u1", participants_count: 12, altered_count: 1,
    meeting_topics: [], target_audience: [ 10 ], health_topics: [ 4 ], health_practices: [], professionals: [ { professional_id: "p-acs" } ],
    participants: [ { patient_id: "pa1", altered_evaluation: true, measurements: { weight_kg: 72.5, height_cm: 160 } } ] });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/collectiveActivities.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ActivityDraft = { teamId: string | null; heldOn: string; shift: VisitShift | null; activityType: number | null; place: { kind: "unit"; unitId: string | null } | { kind: "text"; text: string }; participantsCount: string; alteredCount: string; meetingTopics: number[]; targetAudience: number[]; healthTopics: number[]; healthPractices: number[]; procedureCode: string; professionals: string[]; participants: ParticipantDraft[] }`, com `ParticipantDraft = ({ kind: "patient"; patientId; displayName; hasCpf; hasCns } | { kind: "new"; patient: QuickPatient; duplicatesReviewed: boolean }) & { altered: boolean; weight: string; height: string; smoking: { quit: boolean; dropped_out: boolean } | null }`.
- Unidade de local sem CNES → `place_health_unit_id` "a unidade escolhida não tem CNES — escolha outra ou escreva o local"; profissional sem CNS → `professionals[i]`; data fora de `[options.min_date, today]` → `held_on`; participante repetido → `participants[i]` ("participante repetido").
- Nada de regra numérica escrita à mão além do que o §4 diz como forma (`participants_count` 1–999, `altered_count` ≤ participantes, SIGTAP 10 dígitos, local até 250): o resto sai de `options.domains`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/collectiveActivities.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/collectiveActivities.ts src/lib/collectiveActivities.test.ts
$G commit -m "feat: add collective activity rules read from the activity domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 3 · regras da atividade coletiva lidas do dado · forma de options.domains (D-2), participant_without_sex (D-10)`.

---

### Task 4: Grupo "Visita e atividade coletiva", portões e atalho em memória (F-24.9)

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`
- Create: `src/modules/homeVisits/HomeVisitGate.tsx`, `src/modules/homeVisits/useVisitHandoff.ts`, cascas `src/modules/{HomeVisitDay,HomeVisitsTeam,MicroAreaAssignments,WhoToVisit,CollectiveActivities,FieldDevices,FieldDevicesAdmin}.tsx`
- Test: `src/shell/modules.homeVisits.test.ts`, `src/modules/homeVisits/HomeVisitGate.test.tsx`

**Interfaces:**
- Consumes: `canUseHomeVisits`, `canUseCollectiveActivities`, `hvRoles`, frases (Task 2).
- Produces:
  - `ModuleId` ganha `"home-visit-day" | "home-visits-team" | "micro-area-assignments" | "who-to-visit" | "collective-activities" | "field-devices" | "field-devices-admin"`.
  - `NAV_GROUPS`: grupo `{ label: "Visita e atividade coletiva", items: [ home-visit-day "Folha do dia" "✎", who-to-visit "Quem visitar" "⚑", home-visits-team "Visitas da equipe" "☷", micro-area-assignments "Atribuição de microáreas" "⬡", collective-activities "Atividade coletiva" "☰", field-devices "Aparelhos de campo" "▭", field-devices-admin "Aparelhos de campo da cidade" "▭" ] }` logo depois de "Cadastro territorial".
  - Visibilidade: `hvOn = canUseHomeVisits(user)`, `caOn = canUseCollectiveActivities(user)`; `home-visit-day`, `who-to-visit`, `home-visits-team`, `micro-area-assignments`, `field-devices` → `hvOn && isProfessional`; `collective-activities` → `caOn && isProfessional`; `field-devices-admin` → `hvOn && isAdmin`. O que o CBO permite é a tela que diz (D-4).
  - `CATALOG`: grupo `"Visita e atividade coletiva": { icon: IconWalk, description: "Folha do dia, quem visitar, visitas da equipe, atividade coletiva e aparelhos de campo" }`; telas `home-visit-day` (`IconClipboardList`, "Visitas de hoje, nova visita e fechar o dia"), `who-to-visit` (`IconUserSearch`, "Quem precisa de visita na microárea e por quê"), `home-visits-team` (`IconListDetails`, "Folhas, visitas e situação da ficha por microárea e profissional"), `micro-area-assignments` (`IconMapPin`, "Microáreas atribuídas aos agentes de campo"), `collective-activities` (`IconUsersGroup`, "Registrar e consultar atividades coletivas"), `field-devices` (`IconDeviceMobile`, "Cadastrar e revogar o aparelho do app de campo"), `field-devices-admin` (`IconDeviceMobileCog`, "Aparelhos de campo da cidade e revogação").
  - `HomeVisitGate({ title, sub, feature: "home_visits" | "collective_activities", allow(roles), children(roles) })` no molde de `TerritoryGate`.
  - `useVisitHandoff(active)` em `App.tsx` (molde de `useOmbudsmanHandoff`): `VisitHandoff = { start: VisitStart | null; startVisit(s: VisitStart): void }`, `VisitStart = { teamId; microAreaId; householdId: string | null; familyId: string | null; patientId: string | null; displayName: string | null }` — só em memória, apagado quando a tela ativa deixa de ser `home-visit-day`; `renderModule` ganha o parâmetro `visit: VisitHandoff = NO_VISIT_HANDOFF`.
  - `renderModule`: os sete `case` (as cascas mostram o `HomeVisitGate` com `EmptyState` até as Tasks 6–13).

- [ ] **Step 1: Escreva os testes**

`src/shell/modules.homeVisits.test.ts`:

```ts
const ids = (u: SessionUser) => visibleGroups(u).flatMap((g) => g.items.map((i) => i.id));
it("o grupo vem depois de Cadastro territorial", () => {
  const labels = NAV_GROUPS.map((g) => g.label);
  expect(labels.indexOf("Visita e atividade coletiva")).toBe(labels.indexOf("Cadastro territorial") + 1);
});
it("profissional com os dois interruptores vê tudo, menos a lista da cidade", () => {
  expect(ids(hvUser())).toEqual(expect.arrayContaining([ "home-visit-day", "who-to-visit", "home-visits-team", "micro-area-assignments", "collective-activities", "field-devices" ]));
  expect(ids(hvUser())).not.toContain("field-devices-admin");
});
it("sem home_visits só a atividade coletiva; sem collective_activities, só a visita", () => {
  const onlyCa = ids(hvUser(undefined, [ "clinical_record", "collective_activities" ]));
  expect(onlyCa).toContain("collective-activities");
  expect(onlyCa).not.toContain("home-visit-day");
  const onlyHv = ids(hvUser(undefined, [ "clinical_record", "territory_registry", "home_visits" ]));
  expect(onlyHv).toContain("home-visit-day");
  expect(onlyHv).not.toContain("collective-activities");
});
it("home_visits sem territory_registry não abre a visita", () => {
  expect(ids(hvUser(undefined, [ "clinical_record", "home_visits" ]))).not.toContain("home-visit-day");
});
it("admin vê só a lista da cidade; operador nada", () => {
  expect(ids(hvUser([ "municipal_admin" ]))).toContain("field-devices-admin");
  expect(ids(hvUser([ "municipal_admin" ]))).not.toContain("home-visit-day");
  expect(ids({ ...hvUser(), operator: true })).not.toContain("home-visit-day");
});
it("toda tela nova tem ícone e descrição no catálogo", () => {
  for (const id of [ "home-visit-day", "who-to-visit", "home-visits-team", "micro-area-assignments", "collective-activities", "field-devices", "field-devices-admin" ]) {
    expect(CATALOG.screens[id as ModuleId], id).toBeTruthy();
  }
  expect(CATALOG.groups["Visita e atividade coletiva"]).toBeTruthy();
});
```

`src/modules/homeVisits/HomeVisitGate.test.tsx`:

```tsx
it("interruptor desligado: frase e nenhuma leitura", async () => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(hvUser(undefined, [ "clinical_record", "collective_activities" ]));
  renderWithProviders(<HomeVisitDay />);
  expect(await screen.findByText(HOME_VISITS_DISABLED)).not.toBeNull();
  expect(api.getHomeVisitContext).not.toHaveBeenCalled();
});
it("papel errado: frase do papel", async () => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(hvUser([ "municipal_admin" ]));
  renderWithProviders(<HomeVisitDay />);
  expect(await screen.findByText(HV_NO_ROLE)).not.toBeNull();
});
```

Em `src/shell/modules.test.ts`: os `NAV_GROUPS.length - 22` e `- 21` das linhas 39 e 64 passam a `- 23` e `- 22` (um grupo novo que some para aqueles usuários).

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/shell/modules.homeVisits.test.ts src/modules/homeVisits/HomeVisitGate.test.tsx src/shell/modules.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces. O comentário do bloco em `visibleGroups`: "Módulo 24b (ADR 0052): `home_visits` exige o cadastro territorial utilizável (canUseHomeVisits); `collective_activities` só o prontuário. A sessão não traz o CBO: o contexto de cada tela diz quem registra, lê, atribui e cadastra aparelho (D-4)."

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/shell src/modules/homeVisits/HomeVisitGate.test.tsx src/App.navigation.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.homeVisits.test.ts src/shell/catalog.ts src/App.tsx \
  src/modules/homeVisits/HomeVisitGate.tsx src/modules/homeVisits/HomeVisitGate.test.tsx src/modules/homeVisits/useVisitHandoff.ts \
  src/modules/HomeVisitDay.tsx src/modules/HomeVisitsTeam.tsx src/modules/MicroAreaAssignments.tsx src/modules/WhoToVisit.tsx \
  src/modules/CollectiveActivities.tsx src/modules/FieldDevices.tsx src/modules/FieldDevicesAdmin.tsx
$G commit -m "feat: add the home visit and collective activity menu group with feature gates" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 4 · grupo no menu e portões · capacidades por CBO no contexto (D-4)`.

---

### Task 5: Pessoa da visita e do participante — busca, duplicados e registro rápido (7A)

**Files:**
- Create: `src/modules/homeVisits/PersonPick.tsx`, `src/modules/homeVisits/QuickPatientForm.tsx`
- Test: `src/modules/homeVisits/PersonPick.test.tsx`

**Interfaces:**
- Consumes: `lookupVisitPerson`, `findVisitDuplicates` (Task 1); `quickPatientProblems`, `homeVisitError` (Task 2); `isValidCpf`, `maskCpf`, `onlyDigits` (`src/lib/attendance.ts`); `DOMAINS.justificativa_nao_possui_cpf`, `DOMAINS.sexo` (`src/lib/territoryDictionaries.generated.ts`); `candidateLine` (forma de `src/modules/territory/PersonStart.tsx`, sem o `registration_id`).
- Produces:
  - `PersonPick({ scope: PersonScope; today: string; onChoose(c: PickedPerson): void; onCancel(): void; restart?: { query: { full_name; birth_date; mother_name? }; candidates: VisitCandidate[] } })`; `PickedPerson = { kind: "patient"; patientId: string; displayName: string | null; hasCpf: boolean; hasCns: boolean; sex: PatientSex | null } | { kind: "new"; patient: QuickPatient; duplicatesReviewed: boolean }`.
  - Fluxo: "CPF ou CNS" (um campo; 11 dígitos → `{ cpf }`, 15 → `{ cns }`); achou → "Escolher <nome>"; `null` → o registro rápido com o documento já preenchido; sem documento → "Buscar por nome e nascimento" (`duplicates`), que lista os candidatos ("Escolher") e, só depois de uma busca que voltou para exatamente o que está digitado, "É outra pessoa — registrar" (`duplicatesReviewed: true`). As duas buscas são mutações no clique, nunca reações ao que se digita.
  - `QuickPatientForm({ initial, today, onDone(p: QuickPatient), onCancel })`: nome, nascimento, sexo, mãe (opcional), CPF **ou** justificativa de não ter CPF (domínio `justificativa_nao_possui_cpf`), CNS opcional; `quickPatientProblems` antes de devolver.
  - `restart` (o 409 `duplicate_suspected` da gravação): abre com a frase `DUPLICATE_SUSPECTED_TEXT` (`src/lib/territoryRegistry.ts`) e os `candidates` do 409, sem nova busca.

- [ ] **Step 1: Escreva os testes** `src/modules/homeVisits/PersonPick.test.tsx`

```tsx
it("CPF achado: escolher devolve o paciente com sexo; nada na URL", async () => {
  vi.mocked(api.lookupVisitPerson).mockResolvedValue(person({ sex: "male" }));
  const onChoose = vi.fn();
  mount({ onChoose });
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "529.982.247-25" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  fireEvent.click(await screen.findByRole("button", { name: "Escolher Ana Prado" }));
  expect(api.lookupVisitPerson).toHaveBeenCalledWith("home_visits", { cpf: "52998224725" });
  expect(onChoose).toHaveBeenCalledWith({ kind: "patient", patientId: "pa9", displayName: "Ana Prado", hasCpf: true, hasCns: false, sex: "male" });
});
it("CNS não achado: o registro rápido abre com o CNS e pede o resto", async () => {
  vi.mocked(api.lookupVisitPerson).mockResolvedValue(null);
  mount();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "898000000000005" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  expect(((await screen.findByLabelText("CNS (opcional)")) as HTMLInputElement).value).toBe("898000000000005");
  fireEvent.click(screen.getByRole("button", { name: "Usar esta pessoa" }));
  expect(screen.getAllByRole("alert").length).toBeGreaterThan(1);
});
it("sem documento: 'é outra pessoa' só depois de buscar exatamente o digitado", async () => {
  vi.mocked(api.findVisitDuplicates).mockResolvedValue([ candidate() ]);
  const onChoose = vi.fn();
  mount({ onChoose });
  fireEvent.click(screen.getByRole("button", { name: "Não tem o documento" }));
  fireEvent.change(screen.getByLabelText("Nome completo"), { target: { value: "Ana Prado" } });
  fireEvent.change(screen.getByLabelText("Data de nascimento"), { target: { value: "1990-04-02" } });
  expect(screen.queryByRole("button", { name: /É outra pessoa/ })).toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Buscar possíveis duplicados" }));
  expect(await screen.findByText(/Ana Prado · nascido em 02\/04\/1990 · sem CPF/)).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Nome completo"), { target: { value: "Ana Prado Silva" } });
  expect(screen.queryByRole("button", { name: /É outra pessoa/ })).toBeNull();
});
it("restart do 409 duplicate_suspected: frase e candidatos sem nova busca", async () => {
  mount({ restart: { query: { full_name: "Ana Prado", birth_date: "1990-04-02" }, candidates: [ candidate() ] } });
  expect(screen.getByText(DUPLICATE_SUSPECTED_TEXT)).not.toBeNull();
  expect(screen.getByRole("button", { name: "Escolher Ana Prado" })).not.toBeNull();
  expect(api.findVisitDuplicates).not.toHaveBeenCalled();
});
it("429 da busca vira frase; o documento não aparece em mensagem", async () => {
  vi.mocked(api.lookupVisitPerson).mockRejectedValue(new ApiError(429, { error: "too_many_requests" }, "x"));
  mount();
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "52998224725" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  const msg = await screen.findByText(HOME_VISIT_ERRORS.too_many_requests);
  expect(msg.textContent).not.toMatch(/529/);
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/homeVisits/PersonPick.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces; `useMutation` para as duas buscas (a resposta da busca velha, depois de trocar o texto, é ignorada por `seq`); o documento digitado só em estado local, apagado ao escolher.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/homeVisits/PersonPick.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/homeVisits/PersonPick.tsx src/modules/homeVisits/QuickPatientForm.tsx src/modules/homeVisits/PersonPick.test.tsx
$G commit -m "feat: add person lookup, duplicate check and quick registration for visits and activities" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 5 · busca e registro rápido · lookup sem sexo e par validado sem paciente (D-6, D-7)`.

---

### Task 6: Folha do dia — visitas de hoje, `blockers`, fechar e retirar (F-24.10)

**Files:**
- Modify: `src/modules/HomeVisitDay.tsx`
- Create: `src/modules/homeVisits/DaySheet.tsx`, `src/modules/homeVisits/VisitCard.tsx`, `src/modules/homeVisits/WithdrawForm.tsx`
- Test: `src/modules/homeVisits/DaySheet.test.tsx`

**Interfaces:**
- Consumes: `getHomeVisitContext`, `listHomeVisitDays`, `getHomeVisitDay`, `closeHomeVisitDay`, `withdrawHomeVisit` (Task 1); rótulos, `blockerText`, `fichaLabel`, `CLOSE_EFFECT`, `DAY_CHANGED`, `isDayStale`, `homeVisitRefusals`, `textProblem`, chaves e `NOMINAL_QUERY`/`STATIC_QUERY` (Task 2); `useFormProblems` (`src/modules/psychosocial/useFormProblems.tsx`); `ConfirmButton`, `FrozenTextNotice`, `Tag`, `EmptyState`.
- Produces:
  - `HomeVisitDay({ now?, start? })`: `HomeVisitGate` (`home_visits`, profissional); lê o contexto; equipe (seletor só com mais de uma equipe `can_register`; nenhuma → `HV_CANNOT_REGISTER` com o atalho para "Visitas da equipe"); lê **as folhas de hoje** por `listHomeVisitDays({ team_id, from: today, to: today })` (sem pessoa, sem trilha) e a folha aberta com visitas por `getHomeVisitDay(id)` (`NOMINAL_QUERY`; trilha só quando há visita com pessoa); folhas fechadas de hoje aparecem como cartões com a situação da ficha; "Nova visita" abre o `VisitForm` (Task 7) — com `start` (atalho do Quem visitar) já aberto e preenchido.
  - `DaySheet({ day, domains, onChanged })`: cabeçalho (data, "folha N" quando `sequence > 1`, visitas e retiradas, `fichaLabel`); **lista de cartões** (`<ul>`, nenhuma `<table>`); `blockers` em `<ul>` com `blockerText` e, com `visit_id`, o horário e o nome da visita; "Fechar o dia" (`ConfirmButton` com `CLOSE_EFFECT`; só `own && state === "open"`).
  - `VisitCard({ visit, domains, canWithdraw, onWithdraw })`: hora, turno, desfecho (`Tag`), imóvel/endereço curto, pessoa ("imóvel sem pessoa" quando `subject = none`; "família — responsável <nome>" em `family_responsible`), motivos por rótulo, medidas ("PA 130/85", "Peso 72,5 kg"), "retirada: <motivo>" quando houver; **nunca a observação** (só no detalhe, Task 8).
  - `WithdrawForm({ visit, onDone, onClose })`: motivo 10–500 com `FrozenTextNotice`; corpo com `expected_state`.

- [ ] **Step 1: Escreva os testes** `src/modules/homeVisits/DaySheet.test.tsx`

```tsx
vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof import("../../lib/api")>()), fetchCurrentSession: vi.fn(),
  getHomeVisitContext: vi.fn(), listHomeVisitDays: vi.fn(), getHomeVisitDay: vi.fn(), closeHomeVisitDay: vi.fn(), withdrawHomeVisit: vi.fn() }));
beforeEach(() => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(hvUser());
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context());
  vi.mocked(api.listHomeVisitDays).mockResolvedValue({ items: [ day({ visits: undefined }) ], next_before: null });
  vi.mocked(api.getHomeVisitDay).mockResolvedValue(day());
});
const mount = () => renderWithProviders(<HomeVisitDay now={NOW24B} />);

it("cartões, nunca tabela; motivos por rótulo; sem observação", async () => {
  vi.mocked(api.getHomeVisitDay).mockResolvedValue(day({ visits: [ visit({ note: "Pediu retorno." }) ] }));
  const { container } = mount();
  expect(await screen.findByText("Maria Lima")).not.toBeNull();
  expect(container.querySelector("table")).toBeNull();
  expect(screen.getByText(/Visita periódica/)).not.toBeNull();
  expect(screen.getByText(/PA 130\/85/)).not.toBeNull();
  expect(screen.queryByText("Pediu retorno.")).toBeNull();
  expect(api.listHomeVisitDays).toHaveBeenCalledWith({ team_id: "t1", from: TODAY24B, to: TODAY24B });
});
it("blockers em linguagem simples antes de fechar", async () => {
  vi.mocked(api.getHomeVisitDay).mockResolvedValue(day({ blockers: [ { code: "professional_without_cns", visit_id: null },
    { code: "patient_without_identifier", visit_id: "v1" } ] }));
  mount();
  expect(await screen.findByText("Seu CNS não está no cadastro de profissionais")).not.toBeNull();
  expect(screen.getByText(/Pessoa visitada sem CPF, sem CNS e sem justificativa · 09:15 · Maria Lima/)).not.toBeNull();
});
it("fechar: confirma, manda uma vez e mostra a ficha", async () => {
  let resolve!: (d: HomeVisitDay) => void;
  vi.mocked(api.closeHomeVisitDay).mockReturnValue(new Promise((r) => { resolve = r; }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Fechar o dia" }));
  expect(screen.getByText(CLOSE_EFFECT)).not.toBeNull();
  const confirm = screen.getByRole("button", { name: "Confirmar" });
  fireEvent.click(confirm); fireEvent.click(confirm);
  expect(api.closeHomeVisitDay).toHaveBeenCalledTimes(1);
  resolve(day({ state: "closed", closed_by: "user", closed_at: "2026-11-12T10:01:00-03:00", ficha: { status: "pending", reasons: [] } }));
  expect(await screen.findByText("pendente")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Fechar o dia" })).toBeNull();
});
it("fechar: 409 day_changed mostra a frase e relê uma vez, sem reenviar", async () => {
  vi.mocked(api.closeHomeVisitDay).mockRejectedValue(new ApiError(409, { error: "day_changed" }, "x"));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Fechar o dia" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByText(DAY_CHANGED)).not.toBeNull();
  await waitFor(() => expect(api.getHomeVisitDay).toHaveBeenCalledTimes(2));
  expect(api.closeHomeVisitDay).toHaveBeenCalledTimes(1);
});
it("retirar: motivo 10–500 com o aviso congelado; expected_state; 409 visit_changed relê", async () => {
  vi.mocked(api.withdrawHomeVisit).mockRejectedValueOnce(new ApiError(409, { error: "visit_changed" }, "x"))
    .mockResolvedValueOnce(visit({ state: "withdrawn", withdrawn: { at: "2026-11-12T10:05:00-03:00", reason: "Registrei na casa errada.", by: { name: "Joana ACS" } } }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Retirar visita das 09:15" }));
  const reason = screen.getByLabelText("Motivo da retirada");
  expectFrozenNotice(reason);
  fireEvent.change(reason, { target: { value: "curto" } });
  fireEvent.click(screen.getByRole("button", { name: "Retirar" }));
  expect(api.withdrawHomeVisit).not.toHaveBeenCalled();
  fireEvent.change(reason, { target: { value: "Registrei na casa errada." } });
  fireEvent.click(screen.getByRole("button", { name: "Retirar" }));
  expect(await screen.findByText(VISIT_CHANGED)).not.toBeNull();
  await waitFor(() => expect(api.getHomeVisitDay).toHaveBeenCalledTimes(2));
  expect(api.withdrawHomeVisit).toHaveBeenCalledWith("v1", "Registrei na casa errada.");
});
it("sem can_withdraw, retirada ou folha fechada: sem botão Retirar", async () => {
  vi.mocked(api.getHomeVisitDay).mockResolvedValue(day({ visits: [ visit({ can_withdraw: false }), visit({ id: "v2", state: "withdrawn", occurred_at: "2026-11-12T09:40:00-03:00" }) ] }));
  mount();
  await screen.findAllByText("Maria Lima");
  expect(screen.queryByRole("button", { name: /Retirar visita/ })).toBeNull();
});
it("enfermeiro (não registra): frase e atalho para Visitas da equipe", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  mount();
  expect(await screen.findByText(HV_CANNOT_REGISTER)).not.toBeNull();
  expect(api.listHomeVisitDays).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/homeVisits/DaySheet.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces. Releitura do 409 = `queryClient.invalidateQueries({ queryKey: [ HV_DAY_KEY, id ] })` **uma** vez (e a lista de folhas de hoje), por `seq` (resposta de folha trocada é ignorada). A folha aberta nunca é relida sozinha (`NOMINAL_QUERY`). O `aria-label` "Retirar visita das <hh:mm>" usa só a hora (nada do nome).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/homeVisits/DaySheet.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/HomeVisitDay.tsx src/modules/homeVisits/DaySheet.tsx src/modules/homeVisits/VisitCard.tsx src/modules/homeVisits/WithdrawForm.tsx src/modules/homeVisits/DaySheet.test.tsx
$G commit -m "feat: show the daily visit sheet with blockers, closing and withdrawal" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 6 · folha do dia, fechar e retirar · own/can_withdraw (D-4), withdrawn.reason nulo (D-12)`.

---

### Task 7: Nova visita — lugar, pessoa, desfecho, motivos, medidas e observação (F-24.10)

**Files:**
- Create: `src/modules/homeVisits/VisitForm.tsx`, `src/modules/homeVisits/PlacePick.tsx`, `src/modules/homeVisits/ReasonsField.tsx`, `src/modules/homeVisits/MeasurementsField.tsx`, `src/modules/homeVisits/useClientUuid.ts`
- Modify: `src/modules/HomeVisitDay.tsx`
- Test: `src/modules/homeVisits/VisitForm.test.tsx`

**Interfaces:**
- Consumes: `recordHomeVisit`, `listVisitHouseholds`, `getVisitHousehold` (Task 1); `emptyVisitDraft`, `visitProblems`, `visitBody`, `reasonsFor`, `measurementsAllowed`, `isNonResidential`, `shiftPreview`, `homeVisitRefusals`, `isNetworkOrServer`, `REPLAYED_VISIT`, `OTHER_DAY_TEXT`, `NETWORK_RETRY` (Task 2); `PersonPick` (Task 5); `useFormProblems`, `fieldErrors`; `FrozenTextNotice`.
- Produces:
  - `useClientUuid(): { uuid: string; renew(): void }` (`crypto.randomUUID()`; nasce com o formulário).
  - `VisitForm({ team, context, start?, onSaved(r: { visit; day; replayed }): void, onClose() })`, campos empilhados (375 px):
    1. **Quando:** data (`min=context.min_date`, `max=today`) e hora, com o turno ao lado ("Manhã").
    2. **Onde** (`PlacePick`): microárea (as do `team.micro_areas`; "Fora de área" só com `out_of_area_allowed`) → "Domicílio cadastrado" (lista por `listVisitHouseholds`, "Carregar mais" pelo cursor) **ou** "Imóvel sem cadastro" (tipo de imóvel do domínio). O domicílio escolhido fixa imóvel e microárea (não editáveis).
    3. **Quem:** com imóvel não residencial, a seção some e diz "imóvel não residencial: sem dados de pessoa"; residencial: "Morador" (os `residents` do `getVisitHousehold`, ou `PersonPick` sem domicílio / "Outra pessoa"), "Família" (as famílias ativas do domicílio; `responsible_exited` indisponível com "responsável saiu — troque no Cadastro territorial"), "Imóvel sem pessoa".
    4. **Desfecho:** Realizada, Recusada, Ausente.
    5. **Motivos** (`ReasonsField`): checkboxes por grupo (`fieldset` com `legend` = `group_label`), só com Realizada; indisponíveis com o motivo ao lado.
    6. **Medidas** (`MeasurementsField`): só quando `measurementsAllowed`; pressão em par, glicemia com o tipo.
    7. **Visita compartilhada** (checkbox) e **Observação** (até 500, contador, `FrozenTextNotice`, "não vai ao e-SUS nem ao cidadão").
  - Envio: `useFormProblems({ refusals: homeVisitRefusals, isStale: () => false })`; `check = visitProblems`; `call = recordHomeVisit({ client_uuid: uuid, team_id, ...visitBody(draft) })`; 201/200 → `renew()` e `onSaved`; 4xx → `renew()` e recusas nos campos; rede/5xx → mantém o uuid, `NETWORK_RETRY` e os campos ficam `disabled` com "Tentar de novo" (mesmo corpo) e "Descartar" (fecha). 409 `duplicate_suspected` → reabre o `PersonPick` com `restart` (candidatos do 409); 409 `cpf_taken`/`cns_taken` → a frase e o botão "Buscar pelo documento".
  - `HomeVisitDay` depois de `onSaved`: põe `day` e `visit` no cache da folha (`setQueryData`, sem reler); `replayed` → `REPLAYED_VISIT`; `day.day !== today` ou `sequence > 1` → `OTHER_DAY_TEXT(day)`.

- [ ] **Step 1: Escreva os testes** `src/modules/homeVisits/VisitForm.test.tsx`

```tsx
const openForm = async () => { mount(); fireEvent.click(await screen.findByRole("button", { name: "Nova visita" })); };
const pickHousehold = async () => {
  fireEvent.click(screen.getByLabelText("Domicílio cadastrado"));
  fireEvent.click(await screen.findByRole("button", { name: "Rua das Flores, 120" }));
};

it("domicílio → morador → realizada com motivo e pressão: corpo do contrato", async () => {
  vi.mocked(api.recordHomeVisit).mockResolvedValue({ visit: visit(), day: day({ visits_count: 2 }), replayed: false });
  await openForm();
  fireEvent.change(screen.getByLabelText("Hora"), { target: { value: "09:15" } });
  expect(screen.getByText("Manhã")).not.toBeNull();
  await pickHousehold();
  fireEvent.click(await screen.findByLabelText("Maria Lima"));
  fireEvent.click(screen.getByLabelText("Realizada"));
  fireEvent.click(screen.getByLabelText("Visita periódica"));
  fireEvent.change(screen.getByLabelText("Pressão sistólica"), { target: { value: "130" } });
  fireEvent.change(screen.getByLabelText("Pressão diastólica"), { target: { value: "85" } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  await waitFor(() => expect(api.recordHomeVisit).toHaveBeenCalledTimes(1));
  const body = vi.mocked(api.recordHomeVisit).mock.calls[0][0];
  expect(body).toMatchObject({ team_id: "t1", occurred_at: "2026-11-12T09:15:00-03:00", household_id: "h1", subject: "resident", patient_id: "pa1",
    outcome: 1, reasons: [ 29 ], shared: false, measurements: { systolic: 130, diastolic: 85 } });
  expect(body).not.toHaveProperty("property_type");
  expect(body).not.toHaveProperty("micro_area_id");
  expect(body.client_uuid).toMatch(/^[0-9a-f-]{36}$/);
});
it("não residencial: sem pessoa, sem medida, motivos do domínio", async () => {
  await openForm();
  fireEvent.click(screen.getByLabelText("Imóvel sem cadastro"));
  fireEvent.change(screen.getByLabelText("Tipo de imóvel"), { target: { value: "4" } });
  expect(screen.getByText("imóvel não residencial: sem dados de pessoa")).not.toBeNull();
  expect(screen.queryByLabelText("Morador")).toBeNull();
  fireEvent.click(screen.getByLabelText("Realizada"));
  expect(screen.queryByLabelText("Pressão sistólica")).toBeNull();
  expect(screen.queryByLabelText("Hipertensão")).toBeNull();
  expect(screen.getByLabelText("Imóvel com foco")).not.toBeNull();
});
it("recusada: motivos e medidas somem e não vão", async () => {
  vi.mocked(api.recordHomeVisit).mockResolvedValue({ visit: visit({ outcome: 2, reasons: [], measurements: null }), day: day(), replayed: false });
  await openForm();
  await pickHousehold();
  fireEvent.click(await screen.findByLabelText("Maria Lima"));
  fireEvent.click(screen.getByLabelText("Realizada"));
  fireEvent.click(screen.getByLabelText("Visita periódica"));
  fireEvent.click(screen.getByLabelText("Recusada"));
  expect(screen.queryByLabelText("Visita periódica")).toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  await waitFor(() => expect(api.recordHomeVisit).toHaveBeenCalled());
  expect(vi.mocked(api.recordHomeVisit).mock.calls[0][0]).toMatchObject({ outcome: 2, reasons: [] });
  expect(vi.mocked(api.recordHomeVisit).mock.calls[0][0]).not.toHaveProperty("measurements");
});
it("morador do sexo masculino: Gestante e Puérpera indisponíveis", async () => {
  await openForm();
  await pickHousehold();
  fireEvent.click(await screen.findByLabelText("José Lima"));
  fireEvent.click(screen.getByLabelText("Realizada"));
  expect((screen.getByLabelText("Gestante") as HTMLInputElement).disabled).toBe(true);
  expect(screen.getAllByText("não se aplica a pessoa do sexo masculino").length).toBe(2);
});
it("família: o corpo leva family_id e subject family_responsible, sem patient_id", async () => {
  vi.mocked(api.recordHomeVisit).mockResolvedValue({ visit: visit({ subject: "family_responsible" }), day: day(), replayed: false });
  await openForm();
  await pickHousehold();
  fireEvent.click(await screen.findByLabelText("Família — responsável José Lima"));
  fireEvent.click(screen.getByLabelText("Ausente"));
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  await waitFor(() => expect(api.recordHomeVisit).toHaveBeenCalled());
  const body = vi.mocked(api.recordHomeVisit).mock.calls[0][0];
  expect(body).toMatchObject({ subject: "family_responsible", family_id: "f1", outcome: 3 });
  expect(body).not.toHaveProperty("patient_id");
});
it("client_uuid: mesmo na falha de rede, novo depois de gravar; duplo clique envia uma vez", async () => {
  vi.mocked(api.recordHomeVisit).mockRejectedValueOnce(new TypeError("Failed to fetch"))
    .mockResolvedValueOnce({ visit: visit(), day: day(), replayed: true });
  await openForm();
  fireEvent.click(screen.getByLabelText("Imóvel sem cadastro"));
  fireEvent.change(screen.getByLabelText("Tipo de imóvel"), { target: { value: "3" } });
  fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma1" } });
  fireEvent.click(screen.getByLabelText("Ausente"));
  const send = screen.getByRole("button", { name: "Registrar visita" });
  fireEvent.click(send); fireEvent.click(send);
  expect(await screen.findByText(NETWORK_RETRY)).not.toBeNull();
  expect(api.recordHomeVisit).toHaveBeenCalledTimes(1);
  expect((screen.getByLabelText("Tipo de imóvel") as HTMLSelectElement).disabled).toBe(true);
  fireEvent.click(screen.getByRole("button", { name: "Tentar de novo" }));
  expect(await screen.findByText(REPLAYED_VISIT)).not.toBeNull();
  const [ first, second ] = vi.mocked(api.recordHomeVisit).mock.calls.map((c) => c[0].client_uuid);
  expect(second).toBe(first);
});
it("recusa do api com path cai no campo; o uuid é trocado", async () => {
  vi.mocked(api.recordHomeVisit).mockRejectedValueOnce(new ApiError(422, { error: "visit_date_out_of_range", path: "occurred_at" }, "x"))
    .mockResolvedValueOnce({ visit: visit(), day: day(), replayed: false });
  await openForm();
  fireEvent.click(screen.getByLabelText("Imóvel sem cadastro"));
  fireEvent.change(screen.getByLabelText("Tipo de imóvel"), { target: { value: "3" } });
  fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma1" } });
  fireEvent.click(screen.getByLabelText("Ausente"));
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  expect(await screen.findByText(HOME_VISIT_ERRORS.visit_date_out_of_range)).not.toBeNull();
  expect(screen.getByLabelText("Data").getAttribute("aria-invalid")).toBe("true");
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  await waitFor(() => expect(api.recordHomeVisit).toHaveBeenCalledTimes(2));
  const [ a, b ] = vi.mocked(api.recordHomeVisit).mock.calls.map((c) => c[0].client_uuid);
  expect(b).not.toBe(a);
});
it("visita de ontem entra em outra folha e a tela diz qual", async () => {
  vi.mocked(api.recordHomeVisit).mockResolvedValue({ visit: visit(), day: day({ id: "d0", day: "2026-11-11", sequence: 2 }), replayed: false });
  await openForm();
  fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-11-11" } });
  fireEvent.click(screen.getByLabelText("Imóvel sem cadastro"));
  fireEvent.change(screen.getByLabelText("Tipo de imóvel"), { target: { value: "3" } });
  fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma1" } });
  fireEvent.click(screen.getByLabelText("Ausente"));
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  expect(await screen.findByText("Visita registrada na folha de 11/11/2026 (complementar 2)")).not.toBeNull();
});
it("atalho do Quem visitar: domicílio e pessoa já escolhidos", async () => {
  vi.mocked(api.getVisitHousehold).mockResolvedValue(householdDetail());
  renderWithProviders(<HomeVisitDay now={NOW24B} start={{ teamId: "t1", microAreaId: "ma1", householdId: "h1", familyId: "f1", patientId: "pa1", displayName: "Maria Lima" }} />);
  expect(((await screen.findByLabelText("Maria Lima")) as HTMLInputElement).checked).toBe(true);
});
it("observação: aviso congelado e recusa acima de 500", async () => {
  await openForm();
  const note = screen.getByLabelText("Observação (interna)");
  expectFrozenNotice(note);
  fireEvent.change(note, { target: { value: "x".repeat(501) } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar visita" }));
  expect(api.recordHomeVisit).not.toHaveBeenCalled();
});
```

(O `beforeEach` do arquivo põe `getHomeVisitContext → context()`, `listHomeVisitDays → { items: [ day({ visits: undefined }) ] }`, `getHomeVisitDay → day()`, `listVisitHouseholds → { items: [ household() ], next_before: null }` e `getVisitHousehold → householdDetail()`, com `fetchCurrentSession` no `vi.mock`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/homeVisits/VisitForm.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces. A lista de domicílios e os moradores são leitura nominal (`NOMINAL_QUERY`, chave com o id da microárea / do domicílio). Trocar a microárea limpa domicílio e pessoa; trocar o imóvel para não residencial limpa pessoa, família, medidas e motivos fora do domínio (sem perguntar: a tela diz o que limpou numa linha `role="status"`).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/homeVisits/VisitForm.test.tsx src/modules/homeVisits/DaySheet.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/homeVisits/VisitForm.tsx src/modules/homeVisits/PlacePick.tsx src/modules/homeVisits/ReasonsField.tsx \
  src/modules/homeVisits/MeasurementsField.tsx src/modules/homeVisits/useClientUuid.ts src/modules/HomeVisitDay.tsx src/modules/homeVisits/VisitForm.test.tsx
$G commit -m "feat: record home visits from the daily sheet with idempotent submission" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 7 · nova visita com client_uuid · domicílios e moradores no escopo do 24b (D-3)`.

---

### Task 8: Visitas da equipe — folhas, visitas, contagens e detalhe (F-24.10)

**Files:**
- Modify: `src/modules/HomeVisitsTeam.tsx`
- Create: `src/modules/homeVisits/DayList.tsx`, `src/modules/homeVisits/VisitList.tsx`, `src/modules/homeVisits/VisitDetail.tsx`
- Test: `src/modules/HomeVisitsTeam.test.tsx`

**Interfaces:**
- Consumes: `getHomeVisitContext`, `listHomeVisitDays`, `getHomeVisitDay`, `listHomeVisits`, `countHomeVisits`, `getHomeVisit` (Task 1); rótulos, `fichaLabel`, chaves, `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY` (Task 2); `DataTable`, `StatTile`, `Tag`, `KeyValue`; `DaySheet` (Task 6, só leitura).
- Produces: `HomeVisitsTeam({ now? })`: equipe; filtros microárea (as do contexto), profissional (`team.professionals`, só com `reads_whole_team`; D-5), período (padrão: dia 1 do mês até hoje; o fim antes do início não sai) e desfecho; **só aplica no "Atualizar"**. Abas `role="tablist"`: **"Folhas"** (`DataTable` profissional, dia, folha N, visitas, retiradas, situação da ficha — `card: "title"` no dia, `"actions"` no "Abrir"; sem trilha; "Carregar mais") e **"Visitas"** (`StatTile`s de `countHomeVisits` — realizadas, recusadas, ausentes, retiradas — e a lista nominal por `listHomeVisits`, lida só ao abrir a aba). "Abrir" folha → `DaySheet` só leitura (sem fechar nem retirar quando `!own`); "Abrir" visita → `VisitDetail` com a observação (`getHomeVisit`, trilha), o motivo da retirada (ou "motivo visível só para o dono e o enfermeiro/médico da equipe" quando `reason: null`) e a origem ("app de campo").

- [ ] **Step 1: Escreva os testes** `src/modules/HomeVisitsTeam.test.tsx`

```tsx
it("folhas sem pessoa; filtros só no Atualizar; cursor", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.listHomeVisitDays).mockResolvedValueOnce({ items: [ day({ visits: undefined, own: false, state: "closed", ficha: { status: "not_generated", reasons: [ "professional_without_cns" ] } }) ], next_before: "c2" })
    .mockResolvedValueOnce({ items: [], next_before: null });
  mount();
  expect(await screen.findByText(/ficha não gerada: profissional sem CNS/)).not.toBeNull();
  expect(vi.mocked(api.listHomeVisitDays).mock.calls[0][0]).toEqual({ team_id: "t1", from: "2026-11-01", to: TODAY24B });
  fireEvent.change(screen.getByLabelText("Profissional"), { target: { value: "p-tacs" } });
  expect(api.listHomeVisitDays).toHaveBeenCalledTimes(1);
  fireEvent.click(screen.getByRole("button", { name: "Carregar mais" }));
  await waitFor(() => expect(vi.mocked(api.listHomeVisitDays).mock.calls[1][0]).toMatchObject({ before: "c2" }));
});
it("a aba Visitas só lê ao abrir; contagens sem trilha", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.listHomeVisitDays).mockResolvedValue({ items: [], next_before: null });
  vi.mocked(api.countHomeVisits).mockResolvedValue({ done: 7, refused: 1, absent: 2, withdrawn: 1 });
  vi.mocked(api.listHomeVisits).mockResolvedValue({ items: [ visit() ], next_before: null });
  mount();
  await screen.findByRole("tab", { name: "Folhas" });
  expect(api.listHomeVisits).not.toHaveBeenCalled();
  fireEvent.click(screen.getByRole("tab", { name: "Visitas" }));
  expect(await screen.findByText("Maria Lima")).not.toBeNull();
  expect(screen.getByText("7")).not.toBeNull();
});
it("detalhe: observação só aqui; retirada sem motivo para quem não pode ver", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ tacsTeam() ] }));
  vi.mocked(api.listHomeVisitDays).mockResolvedValue({ items: [], next_before: null });
  vi.mocked(api.countHomeVisits).mockResolvedValue({ done: 1, refused: 0, absent: 0, withdrawn: 1 });
  vi.mocked(api.listHomeVisits).mockResolvedValue({ items: [ visit({ state: "withdrawn", withdrawn: { at: "2026-11-12T10:05:00-03:00", reason: null, by: { name: "Joana ACS" } } }) ], next_before: null });
  vi.mocked(api.getHomeVisit).mockResolvedValue(visit({ note: "Pediu retorno.", state: "withdrawn", withdrawn: { at: "2026-11-12T10:05:00-03:00", reason: null, by: { name: "Joana ACS" } } }));
  mount();
  fireEvent.click(await screen.findByRole("tab", { name: "Visitas" }));
  expect(screen.queryByText("Pediu retorno.")).toBeNull();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir visita das 09:15" }));
  expect(await screen.findByText("Pediu retorno.")).not.toBeNull();
  expect(screen.getByText("motivo visível só para o dono e o enfermeiro ou médico da equipe")).not.toBeNull();
});
it("403 out_of_scope vira frase; sem professionals no contexto, sem filtro de profissional", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ acsTeam() ] }));
  vi.mocked(api.listHomeVisitDays).mockRejectedValue(new ApiError(403, { error: "out_of_scope" }, "x"));
  mount();
  expect(await screen.findByText(HOME_VISIT_ERRORS.out_of_scope)).not.toBeNull();
  expect(screen.queryByLabelText("Profissional")).toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/HomeVisitsTeam.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces; colunas do `DataTable` com `card` (`title` no dia/hora, `actions` no "Abrir", `pair` nos números).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/HomeVisitsTeam.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/HomeVisitsTeam.tsx src/modules/homeVisits/DayList.tsx src/modules/homeVisits/VisitList.tsx src/modules/homeVisits/VisitDetail.tsx src/modules/HomeVisitsTeam.test.tsx
$G commit -m "feat: list team visit sheets and visits with ficha status and visit detail" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 8 · visitas da equipe · profissionais da equipe no contexto (D-5)`.

---

### Task 9: Atribuição de microáreas do enfermeiro (F-24.17, 6A)

**Files:**
- Modify: `src/modules/MicroAreaAssignments.tsx`
- Test: `src/modules/MicroAreaAssignments.test.tsx`

**Interfaces:**
- Consumes: `getHomeVisitContext`, `listFieldAssignments`, `createFieldAssignment`, `endFieldAssignment` (Task 1); `homeVisitError`, chaves, `STATIC_QUERY` (Task 2); `SensitiveAction` (`requiresStepUp: true`, `translateError`), `DataTable`.
- Produces: `MicroAreaAssignments({ onGoToSecurity })`: equipes com `can_assign` (D-4; nenhuma → "só o enfermeiro da equipe atribui microáreas"); lista ativa (microárea, profissional e CBO, desde, por quem; `card`), "Encerrar" (`SensitiveAction` com step-up, descrição "O aparelho dessa pessoa deixa de baixar esta microárea na próxima sincronização."); "Atribuir" (`SensitiveAction` com step-up, campos microárea — as da equipe — e profissional — `candidates`); 409 `already_assigned` → frase e **uma** releitura; 422 `cbo_not_allowed` → frase. A lista é sem pessoa do cidadão (`STATIC_QUERY`) e relida depois de cada ação.

- [ ] **Step 1: Escreva os testes** `src/modules/MicroAreaAssignments.test.tsx`

```tsx
const assignment = { id: "as1", micro_area: { id: "ma2", code: "02", name: "Jardim Sul" }, professional: { id: "p-tacs", name: "Rui TACS", cbo: "322255" },
  assigned_at: "2026-11-01T08:00:00-03:00", assigned_by: { name: "Rita Enfermeira" } };
it("ACS (sem can_assign): frase, nenhuma leitura", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context());
  mount();
  expect(await screen.findByText("só o enfermeiro da equipe atribui microáreas")).not.toBeNull();
  expect(api.listFieldAssignments).not.toHaveBeenCalled();
});
it("atribuir com step-up: microárea e candidato no corpo", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.listFieldAssignments).mockResolvedValue({ items: [], candidates: [ { professional_id: "p-tacs", name: "Rui TACS", cbo: "322255" } ] });
  vi.mocked(api.createFieldAssignment).mockResolvedValue({ id: "as1" } as never);
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Atribuir microárea" }));
  fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma2" } });
  fireEvent.change(screen.getByLabelText("Profissional"), { target: { value: "p-tacs" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.createFieldAssignment).toHaveBeenCalledWith({ micro_area_id: "ma2", professional_id: "p-tacs" }));
});
it("409 already_assigned: frase e uma releitura", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.listFieldAssignments).mockResolvedValue({ items: [ assignment ], candidates: [ { professional_id: "p-tacs", name: "Rui TACS", cbo: "322255" } ] });
  vi.mocked(api.createFieldAssignment).mockRejectedValue(new ApiError(409, { error: "already_assigned" }, "x"));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Atribuir microárea" }));
  fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma2" } });
  fireEvent.change(screen.getByLabelText("Profissional"), { target: { value: "p-tacs" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByText(HOME_VISIT_ERRORS.already_assigned)).not.toBeNull();
  await waitFor(() => expect(api.listFieldAssignments).toHaveBeenCalledTimes(2));
});
it("encerrar com step-up e o efeito no aparelho dito antes", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.listFieldAssignments).mockResolvedValue({ items: [ assignment ], candidates: [] });
  vi.mocked(api.endFieldAssignment).mockResolvedValue({ id: "as1" } as never);
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Encerrar atribuição de Rui TACS na microárea 02" }));
  expect(screen.getByText(/deixa de baixar esta microárea/)).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.endFieldAssignment).toHaveBeenCalledWith("as1"));
});
```

(A sessão do `beforeEach` tem `mfa_verified_at` recente — `sessionWith` — e por isso o `SensitiveAction` não pede código; o caminho do `mfa_required` já é coberto em `src/components/SensitiveAction.test.tsx`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/MicroAreaAssignments.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/MicroAreaAssignments.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/MicroAreaAssignments.tsx src/modules/MicroAreaAssignments.test.tsx
$G commit -m "feat: let the team nurse assign and end field micro-area assignments with step-up" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 9 · atribuição de microáreas · can_assign no contexto (D-4)`.

---

### Task 10: Quem visitar por microárea (F-24.16)

**Files:**
- Modify: `src/modules/WhoToVisit.tsx`, `src/App.tsx` (passa o `startVisit` do handoff)
- Test: `src/modules/WhoToVisit.test.tsx`

**Interfaces:**
- Consumes: `getHomeVisitContext`, `listWhoToVisit`, `countWhoToVisit` (Task 1); `whoReasonText`, chaves, `NOMINAL_QUERY`, `COUNT_QUERY` (Task 2); `VisitHandoff` (Task 4); `StatTile`, `DataTable`.
- Produces: `WhoToVisit({ onStartVisit(s: VisitStart): void })`: equipe e microárea (as do contexto, com o escopo: "sua", "atribuída", "da equipe"); `StatTile`s de `countWhoToVisit` (linhas de cuidado, cadastro a vencer, nunca visitadas, pessoas) com `COUNT_QUERY`; filtro de motivo (checkboxes → `reason[]`, aplicado no "Atualizar"); lista nominal (`NOMINAL_QUERY`; nome, idade, endereço curto, última visita, motivos por `whoReasonText` — nunca CID nem condição; `card: "title"` no nome, `"actions"` no atalho); "Carregar mais" pelo cursor; atalho **"Registrar visita"** só quando a equipe `can_register` → `onStartVisit({ teamId, microAreaId, householdId, familyId, patientId, displayName })` e a navegação para `home-visit-day`.

- [ ] **Step 1: Escreva os testes** `src/modules/WhoToVisit.test.tsx`

```tsx
it("motivos legíveis, sem CID; contagem sem pessoa", async () => {
  vi.mocked(api.countWhoToVisit).mockResolvedValue({ care_line: 1, registration_due: 1, never_visited: 0, people: 1 });
  vi.mocked(api.listWhoToVisit).mockResolvedValue({ items: [ whoItem() ], next_before: null });
  mount();
  expect(await screen.findByText("Maria Lima")).not.toBeNull();
  expect(screen.getByText("Pessoa com hipertensão · Visitas do agente de saúde · vencida desde 01/11/2026")).not.toBeNull();
  expect(screen.getByText("cadastro vence em 20/12/2026")).not.toBeNull();
  expect(vi.mocked(api.listWhoToVisit).mock.calls[0][0]).toEqual({ micro_area_id: "ma1", limit: 50 });
});
it("filtro de motivo só no Atualizar; cursor", async () => {
  vi.mocked(api.countWhoToVisit).mockResolvedValue({ care_line: 1, registration_due: 0, never_visited: 3, people: 4 });
  vi.mocked(api.listWhoToVisit).mockResolvedValueOnce({ items: [ whoItem() ], next_before: "c2" }).mockResolvedValue({ items: [], next_before: null });
  mount();
  await screen.findByText("Maria Lima");
  fireEvent.click(screen.getByLabelText("Nunca visitadas"));
  expect(api.listWhoToVisit).toHaveBeenCalledTimes(1);
  fireEvent.click(screen.getByRole("button", { name: "Atualizar" }));
  await waitFor(() => expect(vi.mocked(api.listWhoToVisit).mock.calls[1][0]).toEqual({ micro_area_id: "ma1", reason: [ "never_visited" ], limit: 50 }));
});
it("Registrar visita leva o domicílio e a pessoa, em memória", async () => {
  vi.mocked(api.countWhoToVisit).mockResolvedValue({ care_line: 1, registration_due: 1, never_visited: 0, people: 1 });
  vi.mocked(api.listWhoToVisit).mockResolvedValue({ items: [ whoItem() ], next_before: null });
  const onStartVisit = vi.fn();
  mount({ onStartVisit });
  fireEvent.click(await screen.findByRole("button", { name: "Registrar visita para Maria Lima" }));
  expect(onStartVisit).toHaveBeenCalledWith({ teamId: "t1", microAreaId: "ma1", householdId: "h1", familyId: "f1", patientId: "pa1", displayName: "Maria Lima" });
});
it("enfermeiro lê a equipe toda, sem o atalho de registrar", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.countWhoToVisit).mockResolvedValue({ care_line: 1, registration_due: 1, never_visited: 0, people: 1 });
  vi.mocked(api.listWhoToVisit).mockResolvedValue({ items: [ whoItem() ], next_before: null });
  mount();
  await screen.findByText("Maria Lima");
  expect(screen.queryByRole("button", { name: /Registrar visita/ })).toBeNull();
  expect(screen.getByRole("option", { name: "02 · Jardim Sul (da equipe)" })).not.toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/WhoToVisit.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces; `App.tsx`: `case "who-to-visit": return <WhoToVisit onStartVisit={(s) => { visit.startVisit(s); setActive("home-visit-day"); }} />;` e `case "home-visit-day": return <HomeVisitDay start={visit.start} />;`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/WhoToVisit.test.tsx src/App.navigation.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/WhoToVisit.tsx src/App.tsx src/modules/WhoToVisit.test.tsx
$G commit -m "feat: add the who-to-visit list per micro-area with a shortcut to record the visit" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 10 · quem visitar · —`.

---

### Task 11: Nova atividade coletiva com as regras do dado (F-24.12)

**Files:**
- Create: `src/modules/collectiveActivities/ActivityForm.tsx`, `src/modules/collectiveActivities/ParticipantsField.tsx`, `src/modules/collectiveActivities/CodeChecklist.tsx`
- Modify: `src/modules/CollectiveActivities.tsx`
- Test: `src/modules/collectiveActivities/ActivityForm.test.tsx`

**Interfaces:**
- Consumes: `getCollectiveOptions`, `recordCollectiveActivity` (Task 1); `emptyActivityDraft`, `activityProblems`, `activityBody`, `fieldRule`, `offeredHealthTopics`, `healthTopicsAllowed`, `offeredPractices`, `needsProcedure`, `smokingAllowed`, `participantWithoutId`, `IDENTIFIER_WARNING` (Task 3); `homeVisitRefusals`, `isNetworkOrServer`, `NETWORK_RETRY` (Task 2); `PersonPick` (Task 5); `useClientUuid` (Task 7); `useFormProblems`.
- Produces:
  - `ActivityForm({ options, onSaved(r: { activity; replayed }), onClose })`, campos empilhados: equipe (as `can_register`), data (`min_date`..`today`), turno, tipo; local ("Unidade" — as com CNES; sem CNES aparece indisponível — **ou** "Outro local" até 250); número de participantes e de avaliações alteradas (este some quando há lista, e a tela mostra o calculado); as listas de códigos (`CodeChecklist`, um `fieldset` cada) aparecem conforme `fieldRule` — `required` com "(obrigatório)", `optional`, `forbidden` some e limpa o que estava marcado; SIGTAP só com `needsProcedure`; profissionais participantes (`options.professionals` sem CNS indisponíveis, "sem CNS no cadastro"); participantes (`ParticipantsField`: `PersonPick` com `scope: "collective_activities"`, avaliação alterada, peso e altura, tabagismo só com `smokingAllowed`; o aviso `IDENTIFIER_WARNING` ao lado de quem não tem CPF nem CNS nos tipos onde a lista é opcional).
  - Envio igual ao da visita (Task 7): `client_uuid` por formulário; 409 `duplicate_suspected` com `path` `participants[i].new_patient` (D-9) reabre o `PersonPick` **daquele** participante com os candidatos; sem `path`, a frase no topo. Sem rascunho (fechar perde o que foi digitado, com `ConfirmButton` "Descartar a atividade?" se algo foi preenchido).

- [ ] **Step 1: Escreva os testes** `src/modules/collectiveActivities/ActivityForm.test.tsx`

```tsx
it("tipo 1: só temas de reunião; público e práticas não aparecem", async () => {
  await openForm();
  fireEvent.change(screen.getByLabelText("Tipo de atividade"), { target: { value: "1" } });
  expect(screen.getByRole("group", { name: "Temas para reunião (obrigatório)" })).not.toBeNull();
  expect(screen.queryByRole("group", { name: /Público-alvo/ })).toBeNull();
  expect(screen.queryByRole("group", { name: /Temas para saúde/ })).toBeNull();
  fireEvent.click(screen.getByLabelText("Educação permanente"));
  expect(screen.getByRole("group", { name: "Temas para saúde" })).not.toBeNull();
});
it("a regra vem do dado: a fixture troca e a tela segue", async () => {
  const d = COLLECTIVE_DOMAINS;
  vi.mocked(api.getCollectiveOptions).mockResolvedValue(options({ domains: { ...d, rules: { ...d.rules,
    by_type: { ...d.rules.by_type, "4": { ...d.rules.by_type["4"], participants: "required" } } } } }));
  await openForm();
  fireEvent.change(screen.getByLabelText("Tipo de atividade"), { target: { value: "4" } });
  expect(screen.getByRole("group", { name: "Participantes (obrigatório)" })).not.toBeNull();
});
it("tipo 5 sem participante não sai; prática 30 pede SIGTAP; todas as recusas de uma vez", async () => {
  await openForm();
  fireEvent.change(screen.getByLabelText("Tipo de atividade"), { target: { value: "5" } });
  fireEvent.click(screen.getByLabelText("Outro procedimento coletivo"));
  expect(screen.getByLabelText("Procedimento SIGTAP (10 dígitos)")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Registrar atividade" }));
  expect(api.recordCollectiveActivity).not.toHaveBeenCalled();
  const texts = screen.getAllByRole("alert").map((a) => a.textContent);
  expect(texts.length).toBeGreaterThanOrEqual(4);
});
it("tipo 5 com participante: corpo do contrato e altered_count calculado", async () => {
  vi.mocked(api.lookupVisitPerson).mockResolvedValue(person({ patient_id: "pa1", display_name: "Maria Lima" }));
  vi.mocked(api.recordCollectiveActivity).mockResolvedValue({ activity: activity(), replayed: false });
  await openForm();
  fireEvent.change(screen.getByLabelText("Tipo de atividade"), { target: { value: "5" } });
  fireEvent.change(screen.getByLabelText("Turno"), { target: { value: "1" } });
  fireEvent.click(screen.getByLabelText("Unidade"));
  fireEvent.change(screen.getByLabelText("Unidade do local"), { target: { value: "u1" } });
  fireEvent.change(screen.getByLabelText("Número de participantes"), { target: { value: "12" } });
  fireEvent.click(screen.getByLabelText("Pessoa idosa"));
  fireEvent.click(screen.getByLabelText("Autocuidado em doenças crônicas"));
  fireEvent.click(screen.getByRole("button", { name: "Adicionar participante" }));
  fireEvent.change(screen.getByLabelText("CPF ou CNS"), { target: { value: "52998224725" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  fireEvent.click(await screen.findByRole("button", { name: "Escolher Maria Lima" }));
  fireEvent.click(screen.getByLabelText("Avaliação alterada — Maria Lima"));
  const send = screen.getByRole("button", { name: "Registrar atividade" });
  fireEvent.click(send); fireEvent.click(send);
  await waitFor(() => expect(api.recordCollectiveActivity).toHaveBeenCalledTimes(1));
  expect(vi.mocked(api.recordCollectiveActivity).mock.calls[0][0]).toMatchObject({ team_id: "t1", activity_type: 5, place_health_unit_id: "u1",
    participants_count: 12, altered_count: 1, meeting_topics: [], target_audience: [ 10 ], health_topics: [ 4 ], health_practices: [],
    participants: [ { patient_id: "pa1", altered_evaluation: true } ] });
});
it("tema 18 nunca é oferecido; unidade sem CNES e profissional sem CNS indisponíveis", async () => {
  const d = COLLECTIVE_DOMAINS;
  vi.mocked(api.getCollectiveOptions).mockResolvedValue(options({ domains: { ...d, health_topics: [ ...d.health_topics, { code: 18, label: "Semana saúde na escola" } ] } }));
  await openForm();
  fireEvent.change(screen.getByLabelText("Tipo de atividade"), { target: { value: "4" } });
  expect(screen.queryByLabelText("Semana saúde na escola")).toBeNull();
  expect((screen.getByRole("option", { name: "Posto Novo (sem CNES)" }) as HTMLOptionElement).disabled).toBe(true);
  expect((screen.getByLabelText("Caio Sem CNS") as HTMLInputElement).disabled).toBe(true);
});
it("409 duplicate_suspected de um participante reabre a busca dele", async () => {
  vi.mocked(api.recordCollectiveActivity).mockRejectedValue(new ApiError(409, { error: "duplicate_suspected", path: "participants[0].new_patient", candidates: [ candidate() ] }, "x"));
  await fillType4WithNewParticipant();
  fireEvent.click(screen.getByRole("button", { name: "Registrar atividade" }));
  expect(await screen.findByText(DUPLICATE_SUSPECTED_TEXT)).not.toBeNull();
  expect(screen.getByRole("button", { name: "Escolher Ana Prado" })).not.toBeNull();
});
it("participante sem CPF e sem CNS no tipo 4: aviso, e a gravação sai", async () => {
  vi.mocked(api.findVisitDuplicates).mockResolvedValue([ candidate({ has_cns: false }) ]);
  await fillType4WithCandidate();
  expect(screen.getByText(IDENTIFIER_WARNING)).not.toBeNull();
});
```

(`openForm`, `fillType4WithNewParticipant` e `fillType4WithCandidate` são ajudantes locais do arquivo, com `getCollectiveOptions → options()` no `beforeEach` e `fetchCurrentSession` no `vi.mock`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/collectiveActivities/ActivityForm.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces. `CodeChecklist({ legend, rule, codes, value, onChange, errorPath })`: `fieldset` com `legend` (com "(obrigatório)" em `required`), um checkbox por código, nada quando `forbidden`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/collectiveActivities/ActivityForm.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/collectiveActivities/ActivityForm.tsx src/modules/collectiveActivities/ParticipantsField.tsx \
  src/modules/collectiveActivities/CodeChecklist.tsx src/modules/CollectiveActivities.tsx src/modules/collectiveActivities/ActivityForm.test.tsx
$G commit -m "feat: record collective activities with cross rules read from the activity domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 11 · nova atividade coletiva · corpo do POST e duplicado por participante (D-8, D-9)`.

---

### Task 12: Atividade coletiva — lista, detalhe e "Registrada por engano" (F-24.12)

**Files:**
- Modify: `src/modules/CollectiveActivities.tsx`
- Create: `src/modules/collectiveActivities/ActivityList.tsx`, `src/modules/collectiveActivities/ActivityDetail.tsx`
- Test: `src/modules/CollectiveActivities.test.tsx`

**Interfaces:**
- Consumes: `getCollectiveOptions`, `listCollectiveActivities`, `getCollectiveActivity`, `markCollectiveMistaken` (Task 1); `fichaLabel`, `homeVisitError`, `textProblem`, chaves, `NOMINAL_QUERY`, `STATIC_QUERY` (Task 2); `COLLECTIVE_NOT_GENERATED` (Task 3); `DataTable`, `FrozenTextNotice`, `useFormProblems`.
- Produces: `CollectiveActivities({ now? })`: `HomeVisitGate` (`collective_activities`); "Nova atividade" (só com alguma equipe `can_register`); filtros equipe, período, tipo e "mostrar as registradas por engano" (→ `state[]=mistaken`), no "Atualizar"; lista sem pessoa (`DataTable`: data, turno, tipo, local, participantes, situação da ficha; `card`); `ActivityDetail` (trilha; participantes com nome, documento mascarado, avaliação, medidas, tabagismo; profissionais; temas por rótulo; ficha com os motivos de "não gerada"); "Registrada por engano" só com `can_mark_mistaken` e `state === "recorded"`: motivo 10–500 com `FrozenTextNotice` e a frase "A atividade sai das telas e das linhas de cuidado; a ficha já enfileirada no e-SUS não é desfeita."; 409 `activity_changed` → frase e uma releitura.

- [ ] **Step 1: Escreva os testes** `src/modules/CollectiveActivities.test.tsx`

```tsx
it("lista sem participantes nem temas; por engano só com o filtro", async () => {
  vi.mocked(api.listCollectiveActivities).mockResolvedValue({ items: [ activity({ participants: undefined, health_topics: undefined }) ], next_before: null });
  mount();
  expect(await screen.findByText("Atendimento em grupo")).not.toBeNull();
  expect(screen.queryByText("Maria Lima")).toBeNull();
  expect(vi.mocked(api.listCollectiveActivities).mock.calls[0][0]).not.toHaveProperty("state");
  fireEvent.click(screen.getByLabelText("Mostrar as registradas por engano"));
  fireEvent.click(screen.getByRole("button", { name: "Atualizar" }));
  await waitFor(() => expect(vi.mocked(api.listCollectiveActivities).mock.calls[1][0]).toMatchObject({ state: [ "recorded", "mistaken" ] }));
});
it("detalhe: participantes e ficha não gerada com motivo", async () => {
  vi.mocked(api.listCollectiveActivities).mockResolvedValue({ items: [ activity() ], next_before: null });
  vi.mocked(api.getCollectiveActivity).mockResolvedValue(activity({ ficha: { status: "not_generated", reasons: [ "participant_without_identifier" ] } }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir atividade de 12/11/2026" }));
  expect(await screen.findByText("Maria Lima")).not.toBeNull();
  expect(screen.getByText(/ficha não gerada: participante sem CPF e sem CNS/)).not.toBeNull();
});
it("engano: motivo com aviso; expected_state; 409 activity_changed relê uma vez", async () => {
  vi.mocked(api.listCollectiveActivities).mockResolvedValue({ items: [ activity() ], next_before: null });
  vi.mocked(api.getCollectiveActivity).mockResolvedValue(activity());
  vi.mocked(api.markCollectiveMistaken).mockRejectedValue(new ApiError(409, { error: "activity_changed" }, "x"));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir atividade de 12/11/2026" }));
  fireEvent.click(await screen.findByRole("button", { name: "Registrada por engano" }));
  const reason = screen.getByLabelText("Motivo");
  expectFrozenNotice(reason);
  fireEvent.change(reason, { target: { value: "Atividade lançada em duplicidade." } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByText(HOME_VISIT_ERRORS.activity_changed)).not.toBeNull();
  await waitFor(() => expect(api.getCollectiveActivity).toHaveBeenCalledTimes(2));
  expect(api.markCollectiveMistaken).toHaveBeenCalledTimes(1);
});
it("sem can_mark_mistaken, sem o botão", async () => {
  vi.mocked(api.listCollectiveActivities).mockResolvedValue({ items: [ activity() ], next_before: null });
  vi.mocked(api.getCollectiveActivity).mockResolvedValue(activity({ can_mark_mistaken: false }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir atividade de 12/11/2026" }));
  await screen.findByText("Maria Lima");
  expect(screen.queryByRole("button", { name: "Registrada por engano" })).toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/CollectiveActivities.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces. "Mostrar as registradas por engano" manda `state: [ "recorded", "mistaken" ]` (a atividade `mistaken` some por padrão, §7).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/CollectiveActivities.test.tsx src/modules/collectiveActivities/ActivityForm.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/CollectiveActivities.tsx src/modules/collectiveActivities/ActivityList.tsx src/modules/collectiveActivities/ActivityDetail.tsx src/modules/CollectiveActivities.test.tsx
$G commit -m "feat: list collective activities, show the detail and mark them as mistaken" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 12 · lista, detalhe e engano · can_mark_mistaken (D-4)`.

---

### Task 13: Aparelhos de campo — cadastrar com step-up e QR, listar, revogar; lista do administrador (F-24.17)

**Files:**
- Modify: `src/modules/FieldDevices.tsx`, `src/modules/FieldDevicesAdmin.tsx`
- Create: `src/modules/homeVisits/DeviceQrDialog.tsx`
- Test: `src/modules/FieldDevices.test.tsx`, `src/modules/FieldDevicesAdmin.test.tsx`

**Interfaces:**
- Consumes: `getHomeVisitContext`, `listFieldDevices`, `registerFieldDevice`, `revokeFieldDevice`, `listAdminFieldDevices`, `adminRevokeFieldDevice` (Task 1); `revokedReasonLabel`, `homeVisitError`, chaves, `STATIC_QUERY` (Task 2); `SensitiveAction`, `ConfirmButton`, `DataTable`; `toDataURL` de `qrcode`.
- Produces:
  - `FieldDevices({ onGoToSecurity })`: o contexto diz se o usuário cadastra (`teams.some(t => t.can_register_device)`, D-4; não → "só agentes de campo da Tabela 7 cadastram aparelho"); lista (nome, cadastrado em, último uso, situação pelo `status` — ativo / "inativo: vínculo encerrado" (Q7) / "revogado: <motivo>"); "Cadastrar aparelho" (`SensitiveAction`, step-up, campo "Nome do aparelho" 1–60; `run` guarda o `FieldDeviceCreated` **só** num `useRef` local e abre o `DeviceQrDialog`); "Revogar" (`ConfirmButton`, sem step-up); 422 `device_limit` → frase.
  - `DeviceQrDialog({ created, onClose })`: `role="dialog"` com `aria-modal`, foco preso, Escape fecha; gera o QR de `qr_payload` por `toDataURL` e mostra `<img alt="QR do aparelho de campo">`, o nome do aparelho e a frase "Leia o QR agora no app de campo. Ele não aparece de novo: se fechar antes, revogue e cadastre outra vez."; **sem o texto do token** (D-15); ao fechar, zera o ref e o `src` da imagem.
  - `FieldDevicesAdmin({ onGoToSecurity })` (admin): lista da cidade com a pessoa (`user.name`), filtro por pessoa (as do próprio resultado; `user_id` na URL), "Revogar" com `SensitiveAction` step-up.

- [ ] **Step 1: Escreva os testes**

`src/modules/FieldDevices.test.tsx`:

```tsx
vi.mock("qrcode", () => ({ toDataURL: vi.fn(async () => "data:image/png;base64,QR") }));
it("cadastrar: step-up, QR uma vez, sem o texto do token; fechar apaga; nada no cache", async () => {
  vi.mocked(api.listFieldDevices).mockResolvedValue([]);
  vi.mocked(api.registerFieldDevice).mockResolvedValue({ device: device(), token: "rsf_SEGREDO", qr_payload: "rotasaude-field:v1:eyJ0b2tlbiI6InJzZl9TRUdSRURPIn0" });
  const { client, container } = mount();
  fireEvent.click(await screen.findByRole("button", { name: "Cadastrar aparelho" }));
  fireEvent.change(screen.getByLabelText("Nome do aparelho"), { target: { value: "Celular da Joana" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  const img = await screen.findByAltText("QR do aparelho de campo");
  expect(img.getAttribute("src")).toBe("data:image/png;base64,QR");
  expect(toDataURL).toHaveBeenCalledWith("rotasaude-field:v1:eyJ0b2tlbiI6InJzZl9TRUdSRURPIn0");
  expect(container.textContent).not.toMatch(/rsf_SEGREDO/);
  expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state.data))).not.toMatch(/rsf_|rotasaude-field/);
  fireEvent.keyDown(screen.getByRole("dialog"), { key: "Escape" });
  await waitFor(() => expect(screen.queryByAltText("QR do aparelho de campo")).toBeNull());
  expect(api.registerFieldDevice).toHaveBeenCalledWith("Celular da Joana");
});
it("nome vazio ou acima de 60 não sai; device_limit tem frase", async () => {
  vi.mocked(api.listFieldDevices).mockResolvedValue([ device(), device({ id: "fd2" }), device({ id: "fd3" }) ]);
  vi.mocked(api.registerFieldDevice).mockRejectedValue(new ApiError(422, { error: "device_limit" }, "x"));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Cadastrar aparelho" }));
  fireEvent.change(screen.getByLabelText("Nome do aparelho"), { target: { value: "x".repeat(61) } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(api.registerFieldDevice).not.toHaveBeenCalled();
  fireEvent.change(screen.getByLabelText("Nome do aparelho"), { target: { value: "Tablet" } });
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByText(HOME_VISIT_ERRORS.device_limit)).not.toBeNull();
});
it("revogar o próprio: confirma, sem step-up; revogado mostra o motivo", async () => {
  vi.mocked(api.listFieldDevices).mockResolvedValueOnce([ device(), device({ id: "fd2", name: "Antigo", status: "revoked", revoked_at: "2026-11-01T08:00:00-03:00", revoked_reason: "membership_ended" }), device({ id: "fd3", name: "Tablet velho", status: "inactive" }) ]);
  vi.mocked(api.revokeFieldDevice).mockResolvedValue(device({ status: "revoked", revoked_at: "2026-11-12T10:00:00-03:00", revoked_reason: "user" }));
  mount();
  expect(await screen.findByText("revogado: fim do vínculo na equipe")).not.toBeNull();
  expect(screen.getByText("inativo: vínculo encerrado")).not.toBeNull(); // Q7
  fireEvent.click(screen.getByRole("button", { name: "Revogar Celular da Joana" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.revokeFieldDevice).toHaveBeenCalledWith("fd1"));
});
it("enfermeiro: frase, sem botão de cadastrar", async () => {
  vi.mocked(api.getHomeVisitContext).mockResolvedValue(context({ teams: [ nurseTeam() ] }));
  vi.mocked(api.listFieldDevices).mockResolvedValue([]);
  mount();
  expect(await screen.findByText("só agentes de campo da Tabela 7 cadastram aparelho")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Cadastrar aparelho" })).toBeNull();
});
```

`src/modules/FieldDevicesAdmin.test.tsx`:

```tsx
it("lista da cidade com a pessoa; filtro manda só o user_id; revogar com step-up", async () => {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(hvUser([ "municipal_admin" ]));
  vi.mocked(api.listAdminFieldDevices).mockResolvedValue([ { ...device(), user: { id: "u-7", name: "Joana ACS" } } ]);
  vi.mocked(api.adminRevokeFieldDevice).mockResolvedValue(device({ revoked_at: "2026-11-12T10:00:00-03:00", revoked_reason: "admin" }));
  mount();
  expect(await screen.findByText("Joana ACS")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Pessoa"), { target: { value: "u-7" } });
  await waitFor(() => expect(api.listAdminFieldDevices).toHaveBeenLastCalledWith("u-7"));
  fireEvent.click(screen.getByRole("button", { name: "Revogar Celular da Joana de Joana ACS" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.adminRevokeFieldDevice).toHaveBeenCalledWith("fd1"));
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/FieldDevices.test.tsx src/modules/FieldDevicesAdmin.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces. A lista dos aparelhos é relida depois do cadastro e da revogação (`invalidateQueries([ FD_KEY ])`), sem nunca guardar a resposta do `POST` no cache.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/modules/FieldDevices.test.tsx src/modules/FieldDevicesAdmin.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/FieldDevices.tsx src/modules/FieldDevicesAdmin.tsx src/modules/homeVisits/DeviceQrDialog.tsx src/modules/FieldDevices.test.tsx src/modules/FieldDevicesAdmin.test.tsx
$G commit -m "feat: register field devices with step-up and a one-time QR, and revoke them" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 13 · aparelhos de campo · can_register_device (D-4), token sem texto (D-15)`.

---

### Task 14: Linha do tempo, `min_gap_days` nas linhas de cuidado e fontes da Produção (F-24.14, F-24.15)

**Files:**
- Modify: `src/lib/api.ts` (`TimelineKind`, `CarePractice`), `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/lib/careLines.ts`, `src/lib/careLines.test.ts`, `src/modules/CareLinesAdmin.tsx`, `src/lib/production.ts`, `src/lib/production.test.ts`
- Test: `src/lib/timeline.homeVisit.test.ts`

**Interfaces:**
- Consumes: `canUseHomeVisits`, `canUseCollectiveActivities` (Task 2).
- Produces:
  - `TimelineKind` ganha `"home_visit" | "collective_activity"`; `TIMELINE_KINDS` ganha `{ key: "home_visit", label: "Visita domiciliar" }` e `{ key: "collective_activity", label: "Atividade coletiva" }`; `timelineKinds(…, urgency, homeVisits = false, collective = false)` — o filtro só com o interruptor; a entrada que o api devolve fica na lista com o interruptor desligado (registro retido). `STATUS_LABEL` ganha `withdrawn: "retirada"` (o `recorded` já existe). O `detail_ref` `{ type: "home_visit" }` **não** abre detalhe nesta entrega (fora de `OPENABLE`, D-13): o resumo do api já traz desfecho, motivos e medidas.
  - `CarePractice` ganha `min_gap_days?: number | null`; `PracticeExtra` inclui `"min_gap_days"`; `practiceDraftOf` preserva-o em `extra` (hoje o `present(p, [ "window_start_days", "code_filter", "applies_to", "all_of" ])` o perderia ao salvar uma linha editada); `practiceSummary` (ou o texto equivalente que `CareLinesAdmin` mostra por prática) acrescenta "intervalo mínimo de N dias entre registros". O editor **não** oferece o campo (D-14).
  - `production.ts`: `GENERATION_REASON` ganha `siaps_deadline_passed` "prazo do SIAPS da competência já passou" (a Produção **não** oferece "Gerar de novo" para ele, Q12), `participant_without_identifier` "participante sem CPF e sem CNS", `participant_without_birth_date` "participante sem data de nascimento", `participant_without_sex` "participante sem sexo no cadastro" (D-10), `participant_professional_without_cns` "profissional participante sem CNS", `place_without_cnes` "local da atividade sem CNES"; `SOURCE_LABEL` ganha `HomeVisitDay: "folha de visitas"` e `CollectiveActivity: "atividade coletiva"`. O "Reenviar" da ficha recusada continua só na Produção (admin), D-16.

- [ ] **Step 1: Escreva os testes**

`src/lib/timeline.homeVisit.test.ts`:

```ts
it("os dois kinds têm rótulo e só entram no filtro com o interruptor", () => {
  expect(timelineKindLabel("home_visit")).toBe("Visita domiciliar");
  expect(timelineKindLabel("collective_activity")).toBe("Atividade coletiva");
  const off = timelineKinds(false, false, false, false, false, false, false).map((k) => k.key);
  expect(off).not.toContain("home_visit");
  const on = timelineKinds(false, false, false, false, false, false, false, true, true).map((k) => k.key);
  expect(on).toEqual(expect.arrayContaining([ "home_visit", "collective_activity" ]));
});
it("retirada tem rótulo; a visita não abre detalhe nesta entrega", () => {
  expect(statusLabel("withdrawn")).toBe("retirada");
  expect(detailTarget({ kind: "home_visit", detail_ref: { type: "home_visit", id: "v1" } } as never, true)).toBeNull();
});
```

Em `src/lib/careLines.test.ts`: "min_gap_days sobrevive à edição" — `customLineBody(draftOfLine(line({ practices: [ practice({ source: "acs_visit", min_gap_days: 30 }) ] }))).practices[0].min_gap_days` é `30`; e o texto da prática contém "intervalo mínimo de 30 dias entre registros".

Em `src/lib/production.test.ts`: `reasonsLabel([ "siaps_deadline_passed", "place_without_cnes" ])` é `"prazo do SIAPS da competência já passou, local da atividade sem CNES"`; `SOURCE_LABEL.HomeVisitDay` é `"folha de visitas"`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/timeline.homeVisit.test.ts src/lib/careLines.test.ts src/lib/production.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente** pelas Interfaces; `PatientTimeline.tsx` passa `canUseHomeVisits(user)` e `canUseCollectiveActivities(user)` às duas posições novas de `timelineKinds`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod24b src/lib/timeline.homeVisit.test.ts src/lib/timeline.test.ts src/lib/careLines.test.ts src/lib/production.test.ts src/modules/timeline src/modules/CareLinesAdmin.test.tsx src/modules/Production.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/timeline.ts src/lib/timeline.homeVisit.test.ts src/modules/timeline/PatientTimeline.tsx src/lib/careLines.ts src/lib/careLines.test.ts \
  src/modules/CareLinesAdmin.tsx src/lib/production.ts src/lib/production.test.ts
$G commit -m "feat: show home visits and collective activities in the timeline, keep min_gap_days and label the new ficha sources" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 14 · linha do tempo, min_gap_days e Produção · detalhe da visita na linha do tempo (D-13), editor sem min_gap_days (D-14)`.

---

### Task 15: Suíte, build, varredura, revisão e prova no navegador (api real na 3057, Vite na 5218)

**Files:** nenhum novo (correções que a prova pedir entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod24b
$R/bin/test dashboard -w dashboard-mod24b
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **16 arquivos de teste novos**: `src/lib/api.homeVisits.test.ts`, `src/lib/homeVisits.test.ts`, `src/lib/collectiveActivities.test.ts`, `src/shell/modules.homeVisits.test.ts`, `src/modules/homeVisits/HomeVisitGate.test.tsx`, `src/modules/homeVisits/PersonPick.test.tsx`, `src/modules/homeVisits/DaySheet.test.tsx`, `src/modules/homeVisits/VisitForm.test.tsx`, `src/modules/HomeVisitsTeam.test.tsx`, `src/modules/MicroAreaAssignments.test.tsx`, `src/modules/WhoToVisit.test.tsx`, `src/modules/collectiveActivities/ActivityForm.test.tsx`, `src/modules/CollectiveActivities.test.tsx`, `src/modules/FieldDevices.test.tsx`, `src/modules/FieldDevicesAdmin.test.tsx`, `src/lib/timeline.homeVisit.test.ts` (`features.test.ts`, `modules.test.ts`, `careLines.test.ts` e `production.test.ts` ganham casos, não arquivo). Contagem que dobra = artefato de build descoberto pelo vitest: confira antes de seguir. **Erro não tratado** no resumo com tudo verde = mock sem `fetchCurrentSession` (Step 2).

- [ ] **Step 2: Varredura de dado sensível, storage, polling, token e mocks**

```bash
cd $W
O="src/modules/homeVisits src/modules/collectiveActivities src/modules/HomeVisit*.tsx src/modules/HomeVisitsTeam.tsx src/modules/MicroAreaAssignments.tsx src/modules/WhoToVisit.tsx src/modules/CollectiveActivities.tsx src/modules/FieldDevices*.tsx src/lib/homeVisits.ts src/lib/collectiveActivities.ts"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
sed -n '/Visita domiciliar e atividade coletiva (módulo 24b/,$p' src/lib/api.ts | grep -nE "cpf|cns|full_name|note|reason[^\[]|token" | grep -nE "URLSearchParams|withQuery|\?\$\{" && echo "revisar: URL" || echo ok-url
grep -rnE "setQueryData\(.*(token|qr_payload)|useQuery\(.*registerFieldDevice" src/modules/FieldDevices.tsx src/modules/homeVisits/DeviceQrDialog.tsx && echo "ERRO: token no cache" || echo ok-token
grep -rnE "\bnote\b" src/modules/homeVisits/VisitCard.tsx src/modules/homeVisits/VisitList.tsx && echo "revisar: observação fora do detalhe" || echo ok-note
for f in $(grep -rlE 'vi\.mock\("\.\./(\.\./)?lib/api", async' src/modules/homeVisits src/modules/collectiveActivities src/modules/HomeVisit*.test.tsx src/modules/HomeVisitsTeam.test.tsx src/modules/MicroAreaAssignments.test.tsx src/modules/WhoToVisit.test.tsx src/modules/CollectiveActivities.test.tsx src/modules/FieldDevices*.test.tsx 2>/dev/null); do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-url`, `ok-token`, `ok-note`, `ok-mocks` sem nenhuma linha `ERRO`. (O `COUNT_QUERY` das contagens é o padrão do 23/24a, sem trilha, e não casa com o grep de polling porque usa a constante.)

- [ ] **Step 3: Conferência do api** — repita o Task 0 Step 4 e confira as Divergências contra o api final; toda forma diferente vira ajuste na task dona (tipo, fixture, teste) antes da prova.

- [ ] **Step 4: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, no `client_uuid` (Task 7 e 11), no `expected_state` das três ações, no 409 com uma releitura, no que a visita a imóvel não residencial não pode levar, nas regras da atividade vindas do dado, no token do aparelho e no `min_gap_days` preservado.

- [ ] **Step 5: Prova no navegador** — **só com autorização do usuário** para migrar o banco de dev das cidades (migrações `20261112900001`–`20261112900003` do api) e semear profissionais de campo (ACS e TACS com CNS, um enfermeiro, um médico, microáreas com ACS responsável, domicílios e cadastros do 24a) e ligar `home_visits` e `collective_activities` no maintenance. Peça antes de qualquer `db:migrate` ou semente.

Com a autorização: api do worktree `apps/api/.claude/mod24b` na **3057**; Vite do worktree na rede do compose, porta **5218**, com proxy para `http://api:3057` (mesmo comando da prova do 33, trocando porta e worktree); abra `curitiba.localhost:5218/dashboard/` e percorra (senhas de dev podem ser mostradas, nunca as de staging):
1. ACS, **telefone 375×812** (`resize_window` preset mobile): Folha do dia vazia → visita a morador (domicílio → morador, realizada, motivo, pressão); à família (ausente); a imóvel não residencial (ponto estratégico, motivo de controle vetorial, sem pessoa nem medida); recusa; registro rápido com duplicado (o 409 e "é outra pessoa"); retirar uma visita com motivo; os `blockers` (um ACS sem CNS na semente); "Fechar o dia" e a situação da ficha.
2. Duplo clique em "Registrar visita" e uma falha de rede simulada (api parado por um instante) com "Tentar de novo": uma visita só.
3. Enfermeiro: Visitas da equipe (folhas, visitas, contagens, detalhe com observação); Atribuição de microáreas (atribuir ao TACS e encerrar, com step-up).
4. TACS: Folha do dia na microárea atribuída; "fora de área" indisponível.
5. Quem visitar (ACS): motivos, filtro, "Registrar visita" abrindo a folha preenchida.
6. Atividade coletiva: tipo 5 com participantes (um sem documento → registro rápido); tipo 1 sem participante; prática 30 com SIGTAP; "Registrada por engano".
7. Aparelhos de campo (ACS): cadastrar com step-up, QR na tela (conferir que o texto do token não aparece e que fechar some com o QR), revogar; admin: lista da cidade e revogação com step-up. Envio e download pelo contrato com `curl` **são da prova do api**, não desta.
8. Linha do tempo do paciente visitado: "Visita domiciliar" e "Atividade coletiva"; Linhas de cuidado: a prática de visita deixa de ser "não acompanhado" e o `min_gap_days` aparece na linha C5 depois da versão nova do catálogo-base (passo do runbook, no maintenance); Produção: as fontes novas com rótulo.
9. Interruptores desligados no maintenance: o grupo some; a folha aberta de ontem fecha pelo job (conferir na Produção no dia seguinte, ou pelo log do api).
10. Só teclado nos passos 1 e 7 (o diálogo do QR prende o foco e o Escape devolve); ao terminar, `resize_window` preset desktop.

Anote cada passo com o que viu. Achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai para o usuário (não vira tarefa).

- [ ] **Step 6: Board e encerramento**

Cards do dashboard prontos: F-24.10, F-24.12, F-24.16 e F-24.17 → **Done** se o api já terminou o deles; F-24.9 espera o maintenance; F-24.14 e F-24.15 esperam o wpda (o último app move). **Verified** só depois da prova do Step 5, com o usuário. Merge, push e tag só com autorização (antes do merge na main, a suíte do dashboard inteira neste worktree basta; a suíte completa do api é da sessão "Testes" e do plano do api). Worktree: `$G -C $R/apps/dashboard worktree remove $W` depois do merge.

**Relato ao coordenador:** `dashboard · dashboard-mod24b/feat/mod-24b-home-visits · Task 15 · suíte <n>/<n>, build ok, varreduras ok, prova <passos ok/achados> · cards movidos`.

---

## Divergências propostas ao contrato

> **Decididas na §16 do contrato (2026-11-12):** ver o bloco "Reconciliação" no topo. A única rejeitada é a rota nova da D-3 (Q11).

Cada uma diz o que a tela precisa, a evidência e a proposta. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D-1 — Domínios da visita fora do contexto.** O §7 dá `options.domains` à atividade "para o dashboard não copiar à mão", mas o `GET /home_visits/context` (§6) não traz domínio nenhum, e o formulário da visita precisa dos rótulos e grupos de `motivo_visita`, de `desfecho`, `turno`, `tipo_glicemia` (a confirmar na task 1), dos tipos não residenciais, dos motivos permitidos em não residencial, dos proibidos para sexo masculino e das faixas das medidas (§3, "limites a confirmar na task 1"). O dicionário do 24a (`src/lib/territoryDictionaries.generated.ts`) só tem `tipo_de_imovel` e `justificativa_nao_possui_cpf`. **Proposta:** `context.domains` com a forma de `HomeVisitDomains` (Task 1): `motivo_visita: [{ code, label, group, group_label }]`, `desfecho`, `turno`, `tipo_glicemia`, `tipo_de_imovel`, `non_residential_property_types`, `non_residential_reasons`, `male_forbidden_reasons`, `measurement_ranges: { <chave>: { min, max, decimals } }` — a mesma do `field-sync/domains-v1.json`.
- **D-2 — Forma de `options.domains` não fixada.** O §7 diz "os do §11 com as regras do §4 em dado" sem forma. **Proposta:** `CollectiveDomains` (Task 1): `activity_types`, `meeting_topics`, `target_audience`, `health_topics` (sem o 18), `health_practices`, todos `[{ code, label }]`, e `rules: { by_type: { "<tipo>": { meeting_topics | target_audience | health_topics | health_practices | participants: "required" | "forbidden" | "optional" } }, health_topics_with_meeting_topic: [6], practices_only_in_type: [{ type: 6, practices: [2, 9, 33] }], procedure_practices: [30], smoking_practices: [25, 26, 27, 28], participant_measurements: { weight_kg, height_cm } }`; o mesmo bloco em `domains-v1.json`.
- **D-3 — Domicílios e moradores para quem não é ACS.** A visita parte "da microárea (domicílio → família ou morador)" (§14), mas as rotas do 24a (`GET /territory/households`, `/territory/registrations?household_id=`) seguem o acesso do 24a, que recusa os CBOs de campo que não são ACS (o próprio C19 cria buscas de pessoa próprias por isso). O TACS, o ACE e os demais da Tabela 7 numa microárea atribuída não teriam como escolher o domicílio. **Proposta:** `GET /home_visits/households?micro_area_id=&before=` → `{ items: [{ id, address_short, property_type, micro_area: { id, code }, families: [{ id, responsible: { patient_id, display_name }, moved_out, responsible_exited }] }], next_before }` e `GET /home_visits/households/:id` → o mesmo com `residents: [{ patient_id, display_name, sex, family_id }]` (cadastro ativo no domicílio), pelo `HomeVisits::Access`, com trilha `access: "home_visits"`.
- **D-4 — A sessão não diz o CBO: capacidades por registro.** `SessionUser` (`src/lib/api.ts:43`) não traz CBO nem profissional, e as regras "só o enfermeiro atribui", "aparelho só para a Tabela 7", "retirar só o dono da folha", "engano só quem registrou" não se deduzem do que o contrato devolve (`<home_visit_day>` tem `professional.id`, mas o dashboard não sabe o próprio; `<collective_activity>` tem `recorded_by: { name }`). **Proposta:** em `context.teams[]`, `can_assign` (enfermeiro da equipe) e `can_register_device` (Tabela 7 com vínculo em equipe 70/76); em `<home_visit_day>`, `own: boolean`; em `<home_visit>`, `can_withdraw: boolean` (dono, `recorded`, folha `open`); em `<collective_activity>`, `can_mark_mistaken: boolean`. O api continua recusando (403/409).
- **D-5 — Profissionais para o filtro de "Visitas da equipe".** O §6 aceita `professional_id` nas listas e o §14 pede o filtro por profissional, mas nenhuma rota dá a lista ao enfermeiro e ao médico (`assignments.candidates` é só do enfermeiro e exclui o ACS responsável). **Proposta:** `context.teams[].professionals: [{ id, name, cbo }]` (os da Tabela 7 com vínculo ativo), só quando `reads_whole_team`; vazio para os outros.
- **D-6 — Busca de pessoa: "ninguém" e par validado sem paciente.** O §5 devolve `{ … }` ou `null`; o 24a devolve 404 `person_not_found` e trata o par validado sem paciente (`citizen_id`, `src/lib/api.ts` `TerritoryPerson`). No 24b, um CPF de cidadão validado sem paciente cairia no registro rápido com CPF, e o `Patients::CreateReviewed` poderia recusar `409 cpf_taken`. **Proposta:** manter o `null` (o dashboard o lê) e o contrato dizer que `new_patient` com o CPF de um par validado sem paciente **cria o paciente ligado a ele** (como o `Patients::Resolve` do 24a), sem `cpf_taken`; `citizen_not_verified` (par só declarado) com frase.
- **D-7 — Sexo e CNS onde a tela espelha a regra.** A regra "motivos 5 e 6 proibidos para `male`" (§3) e o aviso "participante sem CPF e sem CNS fica fora da ficha" (C9) precisam do sexo e de saber se há CNS, mas o lookup traz `has_cpf`/`cns_masked` sem `sex`, e os candidatos do duplicado trazem só `has_cpf`. **Proposta:** `sex` no lookup, nos candidatos e nos moradores da D-3; `has_cns` nos candidatos. Sem o dado, a tela não bloqueia e o api recusa (`reason_not_allowed`).
- **D-8 — Corpo do `POST /collective_activities`.** O §7 diz "corpo = `<collective_activity>` sem ids derivados", mas a forma de leitura tem `place: { health_unit } | { text }`, `unit`, `professionals: [{ id, name, cbo }]` e `participants: [{ patient }]`, enquanto o §1 e o §4 falam em `place_health_unit_id`/`place_text` e `professionals: [{ professional_id }]`. **Proposta:** corpo explícito `CollectiveActivityInput` (Task 1): `{ client_uuid, team_id, held_on, shift, activity_type, place_health_unit_id? | place_text?, participants_count, altered_count, meeting_topics, target_audience, health_topics, health_practices, procedure_code?, professionals: [{ professional_id }], participants: [{ patient_id | new_patient, duplicates_reviewed?, altered_evaluation?, measurements?, smoking? }] }`; campos proibidos pelo tipo vão `[]`.
- **D-9 — Duplicado com vários participantes novos.** `duplicates_reviewed` é um só no corpo da atividade (§7) e o `409 duplicate_suspected` (§5) não diz de qual participante são os candidatos. **Proposta:** `duplicates_reviewed` **por participante** (dentro do item) e o 409 com `path: "participants[i].new_patient"`; na visita, `path: "new_patient"`.
- **D-10 — `participant_without_sex` fora da lista fechada.** A pesquisa (§3, "Nascimento e sexo obrigatórios") exige o sexo do participante e o §11 lista `participant_without_birth_date` mas não o sexo; paciente do 24a pode não ter sexo (`RegistrationPatient.sex: PatientSex | null`). **Proposta:** acrescentar `participant_without_sex` aos motivos de "não gerada" da ficha 6 (a Produção já ganha a frase, Task 14).
- **D-11 — Folhas fechadas de hoje.** `GET /home_visits/days/today` devolve só a folha **aberta** (ou `null`), e a Folha do dia, depois de "Fechar o dia", ficaria vazia sem mostrar a situação da ficha. **Proposta (sem rota nova):** o dashboard usa `GET /home_visits/days?team_id=&from=<hoje>&to=<hoje>` (o dono lê as suas) e lê a aberta pelo `GET /home_visits/days/:id`; o contrato registra que `days/today` é atalho e que a tela pode não usá-lo.
- **D-12 — `withdrawn.reason` nulo para quem não pode ver.** O §6 diz que o motivo da retirada vai "só para o dono e o enfermeiro/médico", mas a forma é `withdrawn: { at, reason, by }`. **Proposta:** `reason: string | null` explícito (nulo para os outros leitores da microárea); o mesmo em `mistaken.reason`.
- **D-13 — Visita na linha do tempo não abre detalhe.** O §12 dá `detail_ref: { type: "home_visit", id }`, mas o `GET /home_visits/:id` só é de quem lê a microárea ou a equipe, e a linha do tempo é lida por outros profissionais (404 para eles); o resumo já traz desfecho, motivos e medidas. **Proposta:** o dashboard não abre o detalhe nesta entrega (a observação fica só em "Visitas da equipe"); o `detail_ref` fica para o 24c ou para um ajuste de acesso.
- **D-14 — `min_gap_days` no editor da cidade.** O §12 põe `min_gap_days` em `PRACTICE_KEYS`, mas não diz se a linha própria da cidade pode usá-lo pelo dashboard; hoje o editor (`src/lib/careLines.ts`, `practiceDraftOf` e `PracticeExtra`) o **apagaria** ao salvar uma linha editada. **Proposta:** o dashboard preserva e mostra o atributo (Task 14) e não o oferece no editor nesta entrega; o contrato registra.
- **D-15 — Texto do token na tela.** O §9.1 diz que a tela "mostra o QR uma vez e não guarda o token", sem dizer se o texto do token aparece para digitação manual. **Proposta:** só o QR (o texto do token nunca no DOM); quem perder o QR revoga e cadastra de novo. Se o 24c precisar de digitação, um código curto de pareamento entra numa versão nova do contrato.
- **D-16 — Onde se reenvia a ficha recusada.** O §10 diz "Reenviar regera da folha, como o 24a" sem a tela; o reenvio do 24a é da Produção, do `municipal_admin`, com step-up (`src/modules/Production.tsx`). **Proposta:** o reenvio das fichas 8 e 6 fica na Produção; "Visitas da equipe" e o detalhe da atividade só mostram a situação, com as fontes `HomeVisitDay` e `CollectiveActivity` rotuladas.
- **D-17 — Contagens com relógio.** O contrato pede "sem polling" na leitura nominal; as contagens (`/home_visits/count`, `/who_to_visit/count`) não gravam trilha. **Proposta:** usar o `COUNT_QUERY` do 23/24a (relê a cada minuto, sem trilha); o contrato registra que a contagem pode ser relida.
- **D-18 — `home_visits` na sessão com o cadastro territorial incompleto.** C1: `home_visits` exige `territory_registry` **utilizável** (IBGE e UF); não está dito se a sessão lista `home_visits` quando o cadastro está ligado mas incompleto. **Proposta:** a sessão só lista `home_visits` quando utilizável; se listar, a tela trata o `403 feature_disabled` com a mesma frase (a Task 2 já cobre).
- **D-19 — Códigos sem nome.** Recusas que a tela encontra e o §13 não nomeia: busca de pessoa com documento inválido (`invalid_cpf`, `invalid_cns` do 24a), registro rápido com identidade travada (`identity_locked`), escolha de domicílio inativo ou família em mudança na visita (proposta: `household_inactive`, `family_moved_out`), microárea inativa (`micro_area_inactive`), revogar aparelho já revogado (`device_already_revoked`), encerrar atribuição já encerrada (`assignment_already_ended`). **Proposta:** nomear no §13; o dashboard dá frase aos que já existem no 24a e mostra os outros crus ("recusado pelo servidor (<código>)") até lá.
- **D-20 — `who_to_visit` sem a equipe.** `GET /home_visits/who_to_visit?micro_area_id=` depende do escopo do usuário e a resposta não diz se ele registra visita ali (o atalho "Registrar visita"). **Proposta:** sem campo novo — a tela usa `context.teams[].can_register` e o `scope` da microárea; o contrato registra que o atalho só aparece para quem registra.

## Self-review

1. **Cobertura da spec (dashboard):** decisão 1A → Tasks 6 e 7 (celular primeiro); 2A → Tasks 3, 11 e 12; 5A → Task 10; 6A → Tasks 9 e 13; 7A → Task 5; decisões 3, 5, 7, 8 → Tasks 2, 6 e 7; 9, 10, 11 → Tasks 3, 11 e 12; 12 → Tasks 2 e 4; 13–15 → Task 14 (só o que o dashboard mostra; a conta é do api); 16 → Task 14; 17 → Task 10; 18 → Task 13; 22–23 → Tasks 6 e 8 (situação da ficha). Decisões 4, 6, 19, 20, 21 são do api e do contracts. §4 telas → Tasks 4–14; §5 → Global Constraints e Task 15 Step 2; §6 dashboard ("folha do dia em largura de celular, atividade com regras espelhadas, quem visitar, aparelhos") → Tasks 6, 7, 10, 11 e 13.
2. **Contrato §5–§9.1, §12–§14:** cada rota de equipe tem função na Task 1 e tela nas Tasks 5–13; `expected_state` nas três ações (Tasks 6, 12); `client_uuid` (Tasks 7, 11); step-up nas quatro rotas que o pedem (Tasks 9, 13); `/field/v1` fica fora (é do app).
3. **Tipos:** `HomeVisit`, `HomeVisitDay`, `HomeVisitContext`, `CollectiveOptions`, `CollectiveActivity` e as funções da Task 1 são os mesmos nas Tasks 2–14; `visitProblems`/`visitBody` (Task 2) governam o `VisitForm` (Task 7); `activityProblems`/`activityBody` (Task 3) o `ActivityForm` (Task 11); `PersonPick` (Task 5) serve às Tasks 7 e 11; `useClientUuid` (Task 7) às Tasks 7 e 11; `DaySheet` (Task 6) às Tasks 6 e 8.
4. **Review Focus:** as cinco linhas têm teste na task dona (6 e 7; 1 e 13; 2 e 7; 3 e 11; 6, 9 e 12).
5. **CI:** todo `vi.mock` do api com `importOriginal` inclui `fetchCurrentSession` (exemplo da Task 6) e a Task 15 varre.
6. **Proporção:** o código do plano é teste, fixture e frase fixa; implementação por assinatura e valores.
