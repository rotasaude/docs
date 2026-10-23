# Módulo 24a — Cadastro individual e domiciliar (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos e bloqueio.** (1) O **api do 24a** (`feat/mod-24a-territory-registry`, worktree `apps/api/.claude/mod24a`, porta de dev **3046**) com as rotas `/territory/*` do contrato (§3–§6), o interruptor `territory_registry` na sessão (`requires: ["feature:clinical_record"]`) e a semente do 24a. Ele é **mergeado antes** deste (contrato §13: api → dashboard → maintenance → catálogo). As Tasks 0–9 não dependem dele (dublês de `src/lib/api`); a prova da Task 14 depende. (2) **As Tasks 10–13 ficam BLOQUEADAS até a Task 0 do api publicar os dicionários das fichas** como acréscimo ao contrato (o "§11 (dicionários)" do contrato §1 — ver Divergência T6 sobre o número). O Step 1 da Task 10 confere isso no `origin/main` do docs; sem a tabela, **pare** depois da Task 9 e avise o coordenador. Nada das Tasks 10–13 é escrito "por adivinhação" a partir do LEDI: os códigos citados nos testes abaixo são os do dicionário oficial LEDI 8.7 (`/ledi/documentacao/referencias/dicionario.html`) e valem **só se o §11 os confirmar**; divergência entre o §11 e este plano se resolve a favor do §11, com nota na Task 14.

**Goal:** Lado dashboard do módulo 24a: o grupo **"Cadastro territorial"** com três telas — **Microáreas** (o `municipal_admin` cria, renomeia, troca o ACS e desativa as microáreas das equipes 70/76, com step-up), **Território da equipe** (microáreas → domicílios → famílias → moradores, com a situação do cadastro de cada um, a **ficha de domicílio e família** e a **ficha de cadastro individual em seções**, com a validação espelhando a do api, a busca de **possíveis duplicados** antes de criar paciente sem CPF, `cpf_taken`, recusa com termo e saída por mudança ou óbito) e **Validade dos cadastros** (vencidos, a vencer em 60 dias e quem falta cadastrar, por microárea, com atalho para cadastrar ou atualizar). Fora do grupo: **ACS e agente de endemias sem conselho** no cadastro de profissionais (F-24.2), **"sem CPF" e o CNS** nas telas de recepção e do paciente, e o **endereço só leitura "do cadastro domiciliar"** nos dados do paciente para o faturamento (22). Nenhum rascunho no navegador.

**Architecture:** O cliente HTTP ganha a seção "Cadastro territorial (módulo 24a)" no fim de `src/lib/api.ts` (tipos do contrato §2–§6 e das Divergências T1–T5; na URL só ids, o filtro de situação e o cursor; CPF, CNS, nome, nascimento, nome da mãe, condições, recusa, situação de rua e moradia sempre no corpo). As regras de tela ficam puras em dois arquivos: `src/lib/territoryRegistry.ts` (papel e interruptor, rótulos dos códigos do contrato §1, frases das recusas, consultas com e sem trilha, código de microárea, busca de duplicados) e `src/lib/territoryDictionaries.ts` (Task 10: os dicionários do §11 e as regras condicionais das fichas — `householdProblem`, `familyProblem`, `registrationProblem`, `exitProblem` —, cada uma devolvendo `{ path, message }` com o mesmo `path` que o api usa em `invalid_household`/`invalid_registration`). As peças ficam em `src/modules/territory/` e as telas em `src/modules/` (`TerritoryMicroAreas`, `TerritoryTeam`, `RegistryStatus`), registradas no grupo novo (`modules.ts`, `catalog.ts`, `App.tsx`). "Sem CPF" é um ajudante único (`src/lib/patientDocument.ts`) usado pelas telas que mostram paciente. O dashboard nunca decide o que o api garante: quem cadastra (CBO e escopo), duplicado de verdade, versão, ficha LEDI, equipe da pessoa e trilha são do api.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-10-23-module-24a-territory-registry-design.md` (§1 intenção e antiobjetivos, §2 decisões 1–13 e as do coordenador, §3 interruptor, §4 dados, §5 telas, §6 segurança e LGPD, §7 testes, §9 F-IDs), `docs/adr/0042.md` e o contrato `docs/superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md` (§1–§7, §10, §12, §13), com as convenções herdadas (21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"**, no fim (T1–T15); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1): interruptor `territory_registry`; situação `valid` | `due_soon` (vence em ≤ 60 dias) | `expired` (> 24 meses) | `missing` | `refused` | `exited`; saída `moved` | `death`; CBO de campo `515105` (ACS) e `515140` (agente de endemias); quem cadastra: ACS responsável pela microárea e membro da equipe com CBO `2235*`, `2251*`, `2252*`, `2253*`; fichas `cadastro_individual` e `cadastro_domiciliar`. Código desconhecido aparece cru, nunca some.
- Interruptor (spec §3): `territory_registry`, que exige `clinical_record`. Desligado, o dashboard é exatamente o de hoje: o grupo "Cadastro territorial" some, o perfil profissional, a recepção e o faturamento não mudam de comportamento. 403 `feature_disabled` → `TERRITORY_DISABLED`. O dashboard nunca escreve interruptor; a chave ganha só rótulo em `features.ts`.
- **Sem rascunho** (spec, decisões do coordenador da revisão): nada da ficha vai para `localStorage`, `sessionStorage`, IndexedDB, URL ou `console`; o formulário guarda o que está na tela só enquanto aberto; cada "Salvar" é uma versão e uma ficha. Nenhum autosave (`useAutosave` não é usado aqui).
- **Dado sensível só no corpo** (contrato, convenções; P20 do 23): na URL vão só ids (`team_id`, `micro_area_id`, `patient_id`, `household_id`, ids de domicílio, família e cadastro), `status` (`expired` | `due_soon` | `missing`), `before` e `limit`. CPF, CNS, nome, nascimento, nome da mãe, condições de saúde, situação de rua, moradia e recusa nunca na URL, em mensagem de erro ou em Analytics.
- **Nenhum polling em rota que grava trilha** (spec §6): lista de domicílios, detalhe do domicílio, moradores, cadastro, lista de validade e a busca de duplicados leem só ao abrir, filtrar, trocar de aba ou depois de uma ação — `NOMINAL_QUERY` do módulo 23 (`refetchInterval: false`, `refetchOnWindowFocus: false`, `refetchOnReconnect: false`, `gcTime: 0`). Só `GET /territory/registry_status/count` (sem trilha) relê a cada minuto (`COUNT_QUERY`). A busca de duplicados é uma mutação no clique, nunca uma consulta reativa ao que se digita.
- **Antes de criar paciente sem CPF, a busca de possíveis duplicados** (spec, decisão 2): o botão "Criar pessoa sem CPF" só aparece depois de uma busca por nome, nascimento e nome da mãe; com candidato, a pessoa escolhe "é esta pessoa" (usa o `patient_id`) ou "é outra pessoa" (cria). Com CPF, `cpf_taken` vira a frase e o formulário fica.
- **Recusa com termo** (spec, decisão 6; LEDI): marcar "o cidadão recusou o cadastro" esconde e **não envia** as seções de sociodemográfico, condições de saúde e situação de rua (o api recusaria) — só a identificação e `refused: true`; no domicílio, a recusa esconde moradia e famílias e mantém o endereço.
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo, inclusive `"{}"`). Criar e alterar microárea passam pelo `SensitiveAction` com step-up (contrato §3); as fichas não pedem step-up (contrato §4–§5).
- Recusar em vez de cortar: nenhum `maxLength` silencioso; limites do §11 (nome 3–70, logradouro até 72, complemento até 30, ponto de referência até 40, nome de microárea até 120) recusados com a frase do limite.
- Quem pode (contrato §3–§6): `municipal_admin` vê e altera microáreas; `health_professional` vê o território e a validade da(s) equipe(s) do seu escopo e grava fichas (o api confere CBO e microárea: `cbo_not_allowed`, `out_of_scope`). A tela não oferece o que o papel não pode; o api recusa de novo.
- Interface (ciclo de interface, subprojetos 1–5 publicados): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG` (o `catalog.test.ts` cobra ícone e descrição), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`var(--ink*)`, `--down`, `--warn`, `--ok`, `--rule*`, `--sunken`, `--panel`; `Tag` com `tone`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). Use os componentes existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `formStyles`, os estilos de `src/modules/exams/styles.ts`); **nenhum componente comum muda**. Achado só visual não vira tarefa.
- "Hoje" é o dia da cidade (`todayInCity()`); as telas recebem `today` por prop (padrão `todayInCity()`) e os testes passam `TODAY24 = "2026-10-23"`, sem relógio falso.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-24.1..F-24.8** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-24.1: api, dashboard e maintenance; F-24.2..F-24.6: api e dashboard; F-24.7 e F-24.8: só api) e a **Verified** depois da prova no navegador (Task 14).

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod24a` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-24a-territory-registry` a partir de `origin/main`, com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod24a`, `G=/opt/homebrew/bin/git`.
- Testes: `bin/test dashboard -w dashboard-mod24a <arquivos>` (o runner único; a saída é o resumo, o log fica em `.test-runs/`). Tipos antes de cada commit: `cd $W && npx tsc --noEmit` (`noUnusedLocals` ligado; o `tsconfig` inclui os testes).
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`. Sessão com `mfa_verified_at` de agora (`sessionWith`) = janela de step-up aberta; com `mfa_verified_at: null`, o `SensitiveAction` pede o código.
- A prova (Task 14) sobe o Vite do worktree num container da rede `rota-saude_default`, porta **5200**, com proxy para o api do 24a na **3046**. O banco de dev só é migrado e semeado para a prova, **com autorização do usuário** (plano do api).

## Review Focus

1. **Pessoa sem CPF cadastrada duas vezes.** O ACS digita nome, nascimento e mãe de alguém que já é paciente (sem CPF ou com CPF). Espera-se: "Criar pessoa sem CPF" só aparece depois da busca de duplicados; com candidato, a tela mostra nome e nascimento e pergunta "é esta pessoa?"; escolher o candidato cadastra com `patient_id` e nada de `new_patient`. Teste: Task 12, "sem CPF: só cria depois da busca; escolher o candidato manda patient_id".
2. **Ficha com dado de saúde deixada aberta, recarregada ou fechada.** Espera-se: nada em `localStorage`/`sessionStorage`, nada na URL de nenhuma chamada além de ids; ao reabrir, o formulário vem do servidor (a última versão), não de um rascunho. Teste: Task 12, "nada no storage nem na URL: CPF, nome, condições só no corpo".
3. **Dois profissionais salvam o mesmo domicílio.** O segundo recebe 409 `stale_version`. Espera-se: a frase "outra pessoa salvou este domicílio — recarregue", o que foi digitado continua na tela, nada é reenviado sozinho, e "Recarregar" traz a versão nova. Teste: Task 11, "409 stale_version: frase, o formulário fica, recarregar relê".
4. **Cidadão que recusa o cadastro.** Espera-se: marcar a recusa esconde e não envia sociodemográfico, condições de saúde e situação de rua (só identificação e `refused: true`); desmarcar devolve as seções vazias; no domicílio, a recusa esconde moradia e famílias e mantém o endereço. Testes: Task 12, "recusa: só identificação e refused; as seções somem"; Task 11, "recusa do domicílio: sem moradia nem família; endereço fica".
5. **Lista de validade aberta o turno todo.** Espera-se: a lista nominal (grava trilha) não relê ao voltar à aba, ao reconectar nem com o tempo; só os contadores releem a cada minuto. Teste: Task 9, "volta à aba e passa o tempo: a lista não relê; os contadores usam COUNT_QUERY".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, situação do §11 | 0 |
| `src/lib/api.ts`, `src/lib/api.territory.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/territoryFixtures.ts` | tipos e cliente HTTP do 24a; interruptor; dados de teste | 1 |
| `src/lib/territoryRegistry.ts`, `src/lib/territoryRegistry.test.ts` | regras de tela que não dependem dos dicionários | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.territory.test.ts`, `src/shell/catalog.ts`, `src/modules/territory/TerritoryGate.tsx`, `src/App.tsx` | grupo "Cadastro territorial", catálogo, portão | 3 |
| `src/modules/TerritoryMicroAreas.tsx`, `src/modules/TerritoryMicroAreas.test.tsx` | microáreas (admin, step-up) | 4 |
| `src/lib/professionals.ts`, `src/lib/api.ts`, `src/modules/professionals/ProfileForm.tsx`, `src/modules/professionals/ProfileForm.test.tsx` | perfil sem conselho (ACS e agente de endemias) | 5 |
| `src/lib/patientDocument.ts`, `src/lib/patientDocument.test.ts`, `src/lib/api.ts`, `src/lib/queueLabels.ts`, telas listadas na task | "sem CPF" e CNS nas telas de recepção e paciente | 6 |
| `src/lib/demographics.ts`, `src/lib/demographics.test.ts`, `src/modules/attendance/DemographicsForm.tsx`, `src/modules/attendance/DemographicsForm.test.tsx`, `src/modules/billing/PendingPanel.tsx`, `src/modules/attendance/CheckIn.tsx`, `src/lib/api.ts` | endereço do cadastro domiciliar só leitura (22) | 7 |
| `src/modules/TerritoryTeam.tsx`, `src/modules/territory/HouseholdList.tsx`, `src/modules/territory/HouseholdDetail.tsx`, `src/modules/TerritoryTeam.test.tsx` | território da equipe: navegação e leitura | 8 |
| `src/modules/RegistryStatus.tsx`, `src/modules/RegistryStatus.test.tsx` | validade de 24 meses e quem falta cadastrar | 9 |
| `src/lib/territoryDictionaries.ts`, `src/lib/territoryDictionaries.test.ts` | dicionários do §11 e regras das fichas (**bloqueada até o §11**) | 10 |
| `src/modules/territory/HouseholdForm.tsx`, `src/modules/territory/FamilyForm.tsx`, `src/modules/territory/HouseholdForm.test.tsx`, `src/modules/territory/HouseholdDetail.tsx`, `src/modules/territory/HouseholdList.tsx` | ficha de domicílio e família | 11 |
| `src/modules/territory/PersonStart.tsx`, `src/modules/territory/RegistrationForm.tsx`, `src/modules/territory/RegistrationForm.test.tsx`, `src/modules/territory/PersonStart.test.tsx`, `src/modules/territory/HouseholdDetail.tsx`, `src/modules/RegistryStatus.tsx` | ficha de cadastro individual em seções, duplicados, recusa, `cpf_taken` | 12 |
| `src/modules/territory/ExitForm.tsx`, `src/modules/territory/ExitForm.test.tsx`, `src/modules/territory/RegistrationForm.tsx` | saída do cadastro (mudança, óbito) | 13 |
| — | suíte, build, revisão, prova no navegador | 14 |

**Estratégia de teste:** cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — sobretudo o que **não** pode ir na URL (CPF, CNS, nome, nascimento, mãe, condições) (Task 1); regras puras com tabela de casos — o que recusa e o que **não** recusa — (Tasks 2 e 10, esta com os domínios e condicionais do §11); menu por papel e interruptor (Task 3); cada tela e peça pelo ponto de entrada real com `renderWithProviders` (`AuthProvider` + `QueryClient`), `vi.mock` do api e sessão por papel e interruptor, cobrindo o caminho feliz, cada recusa que muda a tela (`duplicate_code`, `acs_not_in_team`, `acs_without_cns`, `team_kind_not_allowed`, `stale_version`, `already_registered`, `cpf_taken`, `cpf_or_reason_required`, `invalid_registration`/`invalid_household` com `path`, `cbo_not_allowed`, `out_of_scope`, `feature_disabled`, `address_from_household`), o step-up, a ausência de releitura e a ausência de armazenamento local (Tasks 4–13). A prova final é no navegador, contra o api real na 3046 (Task 14).

---

### Task 0: Board, worktree, base, âncoras e situação do §11

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`a9b2356` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o `origin/main` do docs.
- Produces: cards F-24.1..F-24.8 em In Progress; worktree `$W` na branch `feat/mod-24a-territory-registry`; a base anotada; a situação do §11 anotada (publicado ou não).

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço: a permissão só casa comando que começa com `gh`)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.[1-8] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada card que ainda não estiver em In Progress ou além (os planos do api e do maintenance movem os mesmos cards):

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os oito cards em In Progress ou além. Se algum F-24.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  | "procedure_billing" | "care_lines";\|  care_lines: "Linhas de cuidado"'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const withQuery = \|^async function jsonFetch<T>\|^export interface RecordPatient {\|^export interface BillingPatient \|^export interface ExamPatient \|^  origin_unit_name: string | null; created_at: string; cpf_masked: string; note: string | null;\|^  citizen: { id: string; cpf_masked: string; name?: string };\|^  id: string; cpf_masked: string; checked_in_at: string; protocol_name\|^  race_color_set?: boolean; address_set?: boolean;\|^  patient?: { id: string; display_name: string | null }; professional\|^export interface Professional {\|^  council: string;\|^  council_state: string;\|^  registration_number: string;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "care-lines-team" | "care-lines-preview" | "care-lines-admin";\|      if (item.id === "care-lines-admin") return careOn && isAdmin;\|  const careOn = canUseCareLines(user);'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - 11\|NAV_GROUPS.length - 10'
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '  IconHeartHandshake, IconUserSearch, IconGauge, IconRoute\|    "Linhas de cuidado": { icon: IconHeartHandshake\|    "care-lines-admin": { icon: IconRoute'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "care-lines-admin": return <CareLinesAdmin'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"/territory": proxy(TARGET)'
$G -C $R/apps/dashboard show origin/main:src/lib/careLines.ts | grep -n '^export const NOMINAL_QUERY\|^export const COUNT_QUERY\|^export const STATIC_QUERY\|^export function fmtDay'
$G -C $R/apps/dashboard show origin/main:src/lib/attendance.ts | grep -n '^export function isValidCpf\|^export function maskCpf\|^export const onlyDigits'
$G -C $R/apps/dashboard show origin/main:src/lib/professionals.ts | grep -n '^export function isValidCns'
$G -C $R/apps/dashboard show origin/main:src/modules/attendance/DemographicsForm.tsx | grep -n '^export function DemographicsFields\|^export function DemographicsForm'
```

Expected: cada `grep` acha a linha pedida (o `vite.config.ts` **já** proxa `/territory` desde o módulo 11 — este plano não mexe nele). Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/` e anote a base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod24a -b feat/mod-24a-territory-registry origin/main
cd $R/.worktrees/dashboard-mod24a && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod24a
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(HomeHeart|Hexagons|Map2|CalendarDue)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os quatro ícones. **Anote** a base (arquivos e testes). A Task 14 espera a base mais **13 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira o api do 24a e o §11** (informativo para as Tasks 1–9; **decisivo** para as Tasks 10–13)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api branch --list feat/mod-24a-territory-registry
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod24a api bin/rails routes 2>/dev/null | grep -E " /territory/" | head -60
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md | grep -n '^## ' 
```

Expected: a branch e as rotas (compare com o contrato §3–§6 e com as Divergências T1–T5: diferença de forma vira ajuste em `src/lib/api.ts` e `src/test/territoryFixtures.ts` antes da prova, com nota para a Task 14). No contrato, **anote** se já existe a seção de dicionários publicada pela Task 0 do api (cabeçalho com "Dicionários" — §11 ou o número que ela receber, Divergência T6). Sem ela, as Tasks 10–13 esperam.

---

### Task 1: Tipos, cliente HTTP, interruptor e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do arquivo), `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/test/territoryFixtures.ts`
- Test: `src/lib/api.territory.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `withQuery`, `SessionUser` (existentes em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces (em `src/lib/api.ts`): os tipos `RegistryStatus`, `ExitReason`, `MicroArea`, `MicroAreaInput`, `MicroAreaPatch`, `TerritoryAdminTeam`, `TerritoryTeam`, `TerritoryContext`, `HouseholdAddress`, `Household`, `HouseholdInput`, `HouseholdQuery`, `HouseholdPage`, `Family`, `FamilyInput`, `DuplicateQuery`, `DuplicateCandidate`, `TerritoryPerson`, `PatientIdentity`, `RegistrationPatient`, `Registration`, `RegistrationInput`, `RegistrationQuery`, `ExitInput`, `PersonRow`, `RegistryStatusQuery`, `RegistryStatusPage`, `RegistryCounts`, `CodedSection`; a constante `TERRITORY_PAGE_SIZE = 50`; as funções `listTerritoryAdminTeams(): Promise<TerritoryAdminTeam[]>`, `listMicroAreas(teamId: string): Promise<MicroArea[]>`, `createMicroArea(input: MicroAreaInput): Promise<MicroArea>`, `updateMicroArea(id: string, patch: MicroAreaPatch): Promise<MicroArea>`, `getTerritoryContext(): Promise<TerritoryContext>`, `listHouseholds(q: HouseholdQuery): Promise<HouseholdPage>`, `getHousehold(id: string): Promise<Household>`, `createHousehold(input: HouseholdInput): Promise<Household>`, `updateHousehold(id: string, input: HouseholdInput, version: number): Promise<Household>`, `addFamily(householdId: string, input: FamilyInput, version: number): Promise<Household>`, `updateFamily(id: string, input: FamilyInput, version: number): Promise<Household>`, `findDuplicates(q: DuplicateQuery): Promise<DuplicateCandidate[]>`, `lookupTerritoryPerson(cpf: string): Promise<TerritoryPerson>`, `createRegistration(input: RegistrationInput): Promise<Registration>`, `updateRegistration(id: string, input: RegistrationInput, version: number): Promise<Registration>`, `exitRegistration(id: string, input: ExitInput): Promise<Registration>`, `getRegistration(id: string): Promise<Registration>`, `listRegistrations(q: RegistrationQuery): Promise<Registration[]>`, `listRegistryStatus(q: RegistryStatusQuery): Promise<RegistryStatusPage>`, `countRegistryStatus(teamId: string, microAreaId?: string): Promise<RegistryCounts>`. `FeatureKey` ganha `territory_registry`.
- Produces (em `src/test/territoryFixtures.ts`): `TODAY24`, `TEAM_ESF`, `TEAM_EAP`, `ACS_ANA`, `territoryUser(roles?, over?)`, `territoryContext(over?)`, `microArea(over?)`, `adminTeam(over?)`, `household(over?)`, `family(over?)`, `registration(over?)`, `personRow(over?)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/api.territory.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import {
  addFamily, countRegistryStatus, createHousehold, createMicroArea, createRegistration, exitRegistration, findDuplicates, getHousehold,
  getRegistration, getTerritoryContext, listHouseholds, listMicroAreas, listRegistrations, listRegistryStatus, listTerritoryAdminTeams,
  lookupTerritoryPerson, updateFamily, updateHousehold, updateMicroArea, updateRegistration
} from "./api";

function ok(body: unknown): Promise<Response> {
  return Promise.resolve(new Response(JSON.stringify(body), { status: 200, headers: { "Content-Type": "application/json" } }));
}

const SENSITIVE = /cpf|cns|name|nome|mother|birth|nasc|health|saude|street|rua|refus|living|moradia|\d{11}/i;

describe("cliente do cadastro territorial (contrato §3–§6)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(() => ok({})); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  const urlOf = (i = 0) => new URL(String(fetchMock.mock.calls[i][0]), "http://curitiba.localhost");
  const initOf = (i = 0) => (fetchMock.mock.calls[i][1] ?? {}) as RequestInit;
  const headersOf = (i = 0) => initOf(i).headers as Record<string, string>;
  const bodyOf = (i = 0) => JSON.parse(String(initOf(i).body));

  it("microáreas: equipes do admin e lista por equipe em envelope; escrita com JSON", async () => {
    fetchMock
      .mockImplementationOnce(() => ok({ teams: [ { id: "tm1" } ] }))
      .mockImplementationOnce(() => ok({ micro_areas: [ { id: "ma1" } ] }))
      .mockImplementationOnce(() => ok({ id: "ma2" }))
      .mockImplementationOnce(() => ok({ id: "ma2" }));
    expect(await listTerritoryAdminTeams()).toEqual([ { id: "tm1" } ]);
    expect(urlOf(0).pathname).toBe("/territory/admin/teams");
    expect(await listMicroAreas("tm1")).toEqual([ { id: "ma1" } ]);
    expect(urlOf(1).pathname + urlOf(1).search).toBe("/territory/admin/micro_areas?team_id=tm1");
    await createMicroArea({ team_id: "tm1", code: "02", name: "Rua das Flores", acs_professional_id: "pr9" });
    expect(initOf(2).method).toBe("POST");
    expect(headersOf(2)["Content-Type"]).toBe("application/json");
    expect(bodyOf(2)).toEqual({ team_id: "tm1", code: "02", name: "Rua das Flores", acs_professional_id: "pr9" });
    await updateMicroArea("ma2", { active: false });
    expect(urlOf(3).pathname).toBe("/territory/admin/micro_areas/ma2");
    expect(initOf(3).method).toBe("PATCH");
    expect(bodyOf(3)).toEqual({ active: false });
  });

  it("contexto do profissional (T2); ausente vira vazio", async () => {
    fetchMock.mockImplementationOnce(() => ok({ teams: [ { id: "tm1", micro_areas: [] } ] })).mockImplementationOnce(() => ok({}));
    expect(await getTerritoryContext()).toEqual({ teams: [ { id: "tm1", micro_areas: [] } ] });
    expect(urlOf(0).pathname).toBe("/territory/teams");
    expect(await getTerritoryContext()).toEqual({ teams: [] });
  });

  it("domicílios: lista paginada só com ids e cursor; versão esperada no corpo", async () => {
    fetchMock
      .mockImplementationOnce(() => ok({ households: [], next_before: "c1" }))
      .mockImplementationOnce(() => ok({ id: "hh1" }))
      .mockImplementationOnce(() => ok({ id: "hh1" }))
      .mockImplementationOnce(() => ok({ id: "hh1" }))
      .mockImplementationOnce(() => ok({ id: "hh1" }))
      .mockImplementationOnce(() => ok({ id: "hh1" }));
    expect(await listHouseholds({ microAreaId: "ma1", before: "c0" })).toEqual({ households: [], next_before: "c1" });
    expect(urlOf(0).pathname + urlOf(0).search).toBe("/territory/households?micro_area_id=ma1&before=c0&limit=50");
    await getHousehold("hh1");
    expect(urlOf(1).pathname).toBe("/territory/households/hh1");
    const input = { micro_area_id: "ma1", address: { zip: "81000000", street: "Rua das Flores", number: "10", complement: null, neighborhood_id: null, reference: null },
      property_type: 1, living_conditions: { localizacao: 83 }, contact_phone: "41999990000", refused: false };
    await createHousehold(input);
    expect(initOf(2).method).toBe("POST");
    expect(bodyOf(2)).toEqual(input);
    await updateHousehold("hh1", input, 3);
    expect(initOf(3).method).toBe("PATCH");
    expect(bodyOf(3)).toEqual({ ...input, version: 3 });
    await addFamily("hh1", { responsible_patient_id: "pa1", income_band: 3, members_count: 4, living_since: "2019-05", moved_out: false }, 3);
    expect(urlOf(4).pathname).toBe("/territory/households/hh1/families");
    expect(bodyOf(4)).toEqual({ responsible_patient_id: "pa1", income_band: 3, members_count: 4, living_since: "2019-05", moved_out: false, version: 3 });
    await updateFamily("fa1", { responsible_patient_id: "pa1", income_band: null, members_count: 3, living_since: null, moved_out: true }, 4);
    expect(urlOf(5).pathname).toBe("/territory/families/fa1");
    expect(initOf(5).method).toBe("PATCH");
    for (let i = 0; i < 6; i++) expect(urlOf(i).search).not.toMatch(SENSITIVE);
  });

  it("duplicados e busca por CPF: tudo no corpo de um POST (contrato §5; T4)", async () => {
    fetchMock.mockImplementationOnce(() => ok({ candidates: [ { patient_id: "pa7" } ] })).mockImplementationOnce(() => ok({ person: { patient_id: "pa1" } }));
    expect(await findDuplicates({ full_name: "Maria da Silva", birth_date: "1960-02-01", mother_name: "Ana da Silva" })).toEqual([ { patient_id: "pa7" } ]);
    expect(urlOf(0).pathname + urlOf(0).search).toBe("/territory/patients/duplicates");
    expect(initOf(0).method).toBe("POST");
    expect(bodyOf(0)).toEqual({ full_name: "Maria da Silva", birth_date: "1960-02-01", mother_name: "Ana da Silva" });
    expect(await lookupTerritoryPerson("52998224725")).toEqual({ patient_id: "pa1" });
    expect(urlOf(1).pathname + urlOf(1).search).toBe("/territory/person_lookup");
    expect(bodyOf(1)).toEqual({ cpf: "52998224725" });
  });

  it("cadastro individual: criar, atualizar com versão, saída, leitura e lista só com ids na URL", async () => {
    fetchMock.mockImplementation(() => ok({ id: "rg1", registrations: [ { id: "rg1" } ] }));
    const input = { new_patient: { full_name: "José Souza", birth_date: "2026-09-30", sex: 0, no_cpf_reason: 3 }, household_id: "hh1", family_id: "fa1",
      is_responsible: false, sociodemographic: { grau_instrucao_cidadao: 51 }, health_conditions: { status_tem_diabetes: false }, refused: false };
    await createRegistration(input);
    expect(urlOf(0).pathname).toBe("/territory/registrations");
    expect(bodyOf(0)).toEqual(input);
    await updateRegistration("rg1", { patient_id: "pa1", identity: { cpf: "52998224725" }, sociodemographic: {}, health_conditions: {} }, 2);
    expect(initOf(1).method).toBe("PATCH");
    expect(bodyOf(1).version).toBe(2);
    expect(bodyOf(1).identity).toEqual({ cpf: "52998224725" });
    await exitRegistration("rg1", { reason: "death", on: "2026-10-20" });
    expect(urlOf(2).pathname).toBe("/territory/registrations/rg1/exit");
    expect(bodyOf(2)).toEqual({ reason: "death", on: "2026-10-20" });
    await getRegistration("rg1");
    expect(urlOf(3).pathname).toBe("/territory/registrations/rg1");
    expect(await listRegistrations({ householdId: "hh1" })).toEqual([ { id: "rg1" } ]);
    expect(urlOf(4).pathname + urlOf(4).search).toBe("/territory/registrations?household_id=hh1");
    await listRegistrations({ patientId: "pa1" });
    expect(urlOf(5).search).toBe("?patient_id=pa1");
    for (let i = 0; i < 6; i++) expect(urlOf(i).search).not.toMatch(SENSITIVE);
  });

  it("validade: lista com filtros e cursor; contagem sem trilha", async () => {
    fetchMock.mockImplementationOnce(() => ok({ people: [], next_before: null })).mockImplementationOnce(() => ok({ valid: 120, due_soon: 8, expired: 31, missing: 44 }));
    expect(await listRegistryStatus({ teamId: "tm1", microAreaId: "ma1", status: "expired", before: null })).toEqual({ people: [], next_before: null });
    expect(urlOf(0).pathname + urlOf(0).search).toBe("/territory/registry_status?team_id=tm1&micro_area_id=ma1&status=expired&limit=50");
    expect(await countRegistryStatus("tm1")).toEqual({ valid: 120, due_soon: 8, expired: 31, missing: 44 });
    expect(urlOf(1).pathname + urlOf(1).search).toBe("/territory/registry_status/count?team_id=tm1");
  });
});
```

E, em `src/lib/features.test.ts`, no fim:

```ts
describe("módulo 24a", () => {
  it("territory_registry tem rótulo em português", () => {
    expect(featureLabel("territory_registry")).toBe("Cadastro individual e domiciliar");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/api.territory.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções não existem; `featureLabel("territory_registry")` devolve a chave.

- [ ] **Step 3: Implemente a seção "Cadastro territorial (módulo 24a)" no fim de `src/lib/api.ts`**

Base `const TERRITORY_REGISTRY_BASE = import.meta.env.VITE_TERRITORY_BASE || "/territory"` (o mesmo prefixo do módulo 11, já proxado), `territoryPath(...parts)` com `encodeURIComponent` em cada parte, `TERRITORY_PAGE_SIZE = 50`. Listas desembrulham o envelope com `?? []`; escritas e detalhe devolvem o objeto puro. Os tipos (os campos opcionais são das Divergências; os objetos das seções têm as chaves do IDL em `snake_case`, contrato §1):

```ts
export type RegistryStatus = "valid" | "due_soon" | "expired" | "missing" | "refused" | "exited";
export type ExitReason = "moved" | "death";
// Seção da ficha: chave do IDL LEDI 8.7.0 em snake_case → código (i64), lista de códigos ou booleano (§11; T13).
export type CodedSection = Record<string, number | number[] | boolean | string | null>;

export interface MicroArea {
  id: string; team: { id: string; ine: string; name: string }; code: string; name: string;
  acs: { professional_id: string; name: string } | null; active: boolean;
}
export interface MicroAreaInput { team_id: string; code: string; name: string; acs_professional_id?: string }
export interface MicroAreaPatch { name?: string; acs_professional_id?: string | null; active?: boolean }
// T1: equipes 70/76 da cidade e os ACS (CBO 515105) com vínculo na unidade delas.
export interface TerritoryAdminTeam {
  id: string; ine: string; name: string; kind: string; unit: { id: string; name: string };
  acs_candidates: { professional_id: string; name: string; has_cns: boolean }[];
}
// T2: equipes do escopo do profissional, com as microáreas ativas e se ele pode gravar ficha nelas.
export interface TerritoryTeam {
  id: string; ine: string; name: string; unit: { id: string; name: string };
  micro_areas: (MicroArea & { can_register: boolean })[];
}
export interface TerritoryContext { teams: TerritoryTeam[] }

export interface HouseholdAddress {
  zip: string; street: string; number: string; complement: string | null; neighborhood_id: string | null; reference: string | null;
  // T8: campos que a ficha exige e o contrato §2 não tem; opcionais até o api decidir.
  street_type?: string | null; no_number?: boolean;
}
export interface Family {
  id: string; responsible: { patient_id: string; display_name: string | null }; income_band: number | null; members_count: number;
  living_since: string | null; moved_out: boolean;
}
export interface Household {
  id: string; micro_area: { id: string; code: string }; address: HouseholdAddress; property_type: number;
  living_conditions?: CodedSection; contact_phone: string | null; refused: boolean; status: "active" | "inactive" | string;
  version: number; registered_at: string; registered_by: { name: string | null }; families: Family[];
}
export interface HouseholdInput {
  micro_area_id: string; address: HouseholdAddress; property_type: number; living_conditions: CodedSection | null;
  contact_phone: string | null; refused: boolean;
}
export interface HouseholdQuery { microAreaId: string; before?: string | null }
export interface HouseholdPage { households: Household[]; next_before: string | null }
export interface FamilyInput {
  responsible_patient_id: string; income_band: number | null; members_count: number; living_since: string | null; moved_out: boolean;
}

export interface DuplicateQuery { full_name: string; birth_date: string; mother_name?: string }
export interface DuplicateCandidate { patient_id: string; display_name: string; birth_date: string; has_cpf: boolean }
// T4: a pessoa pelo CPF (paciente, par validado sem paciente, ou nada).
export interface TerritoryPerson {
  patient_id: string | null; citizen_id: string | null; display_name: string | null; cpf_masked: string; registration_id: string | null;
}
// T5: identificação editável (paciente sem identidade validada no posto); raça/cor e etnia gravam no paciente (spec, uma fonte por dado).
export interface PatientIdentity {
  full_name?: string; social_name?: string | null; birth_date?: string; sex?: number; mother_name?: string | null; mother_unknown?: boolean;
  // Raça/cor e etnia no código do paciente (o do 22: "01"–"05", etnia de 4 dígitos), não no da ficha (Divergência T15).
  cpf?: string; no_cpf_reason?: number; cns?: string | null; race_color?: string; ethnicity?: string | null; nationality?: number;
}
export interface RegistrationPatient {
  id: string; display_name: string; birth_date: string; sex: number | string; has_cpf: boolean; no_cpf_reason: number | null;
  cns_masked?: string | null; identity_validated?: boolean;
}
export interface Registration {
  id: string; patient: RegistrationPatient; household_id: string | null; family_id: string | null; is_responsible: boolean;
  sociodemographic: CodedSection; health_conditions: CodedSection; street_situation: CodedSection | null; refused: boolean;
  exit: { reason: ExitReason | string; on: string } | null; status: RegistryStatus | string; version: number;
  registered_at: string; registered_by: { name: string | null }; identity_changed_at: string | null;
}
export type RegistrationInput = (
  { patient_id: string } | { citizen_id: string } | { new_patient: PatientIdentity & { full_name: string; birth_date: string; sex: number } }
) & {
  identity?: PatientIdentity; household_id?: string | null; family_id?: string | null; is_responsible?: boolean;
  sociodemographic: CodedSection; health_conditions: CodedSection; street_situation?: CodedSection | null; refused?: boolean;
};
export type RegistrationQuery = { patientId: string } | { householdId: string };
export interface ExitInput { reason: ExitReason; on: string }

export interface PersonRow {
  patient_id: string | null; citizen_id: string | null; display_name: string | null; age: number | null; micro_area_code: string | null;
  status: RegistryStatus | string; registered_at: string | null; registration_id: string | null;
}
export interface RegistryStatusQuery { teamId: string; microAreaId?: string; status: "expired" | "due_soon" | "missing"; before?: string | null }
export interface RegistryStatusPage { people: PersonRow[]; next_before: string | null }
export interface RegistryCounts { valid: number; due_soon: number; expired: number; missing: number }
```

Rotas: `GET admin/teams`, `GET admin/micro_areas?team_id=`, `POST admin/micro_areas`, `PATCH admin/micro_areas/:id`, `GET teams`, `GET households?micro_area_id=&before=&limit=` (`before` só quando houver), `GET households/:id`, `POST households`, `PATCH households/:id` (corpo `{ ...input, version }`), `POST households/:id/families` e `PATCH families/:id` (corpo `{ ...input, version }`), `POST patients/duplicates` (devolve `candidates`), `POST person_lookup` (devolve `person`), `POST registrations`, `PATCH registrations/:id` (`{ ...input, version }`), `POST registrations/:id/exit`, `GET registrations/:id`, `GET registrations?patient_id=` ou `?household_id=` (devolve `registrations`), `GET registry_status?team_id=&micro_area_id=&status=&before=&limit=` e `GET registry_status/count?team_id=&micro_area_id=` (só os presentes, na ordem do teste).

Em `src/lib/features.ts`: `FeatureKey` ganha `| "territory_registry"` e `FEATURE_LABEL` ganha, depois de `care_lines`:

```ts
  // Módulo 24a (ADR 0042): exige clinical_record (modo record).
  territory_registry: "Cadastro individual e domiciliar"
```

Crie `src/test/territoryFixtures.ts` com os dados abaixo (nomes fictícios; CPF e CNS com dígito válido, semente realista):

```ts
// src/test/territoryFixtures.ts
import type { Family, Household, MicroArea, PersonRow, Registration, SessionUser, TerritoryAdminTeam, TerritoryContext } from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const TODAY24 = "2026-10-23";
export const TEAM_ESF = { id: "tm1", ine: "0001234567", name: "ESF Vila Verde" };
export const TEAM_EAP = { id: "tm2", ine: "0001234575", name: "EAP Boqueirão" };
export const ACS_ANA = { professional_id: "pr1", name: "Ana Prado" };

export function territoryUser(roles: string[] = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, { id: "us24", email_address: "acs@curitiba.demo", features: [ "clinical_record", "territory_registry" ], ...over });
}
export function microArea(over: Partial<MicroArea> = {}): MicroArea {
  return { id: "ma1", team: TEAM_ESF, code: "01", name: "Vila Verde de cima", acs: ACS_ANA, active: true, ...over };
}
export function adminTeam(over: Partial<TerritoryAdminTeam> = {}): TerritoryAdminTeam {
  return { ...TEAM_ESF, kind: "70", unit: { id: "u1", name: "UBS Vila Verde" },
    acs_candidates: [ { ...ACS_ANA, has_cns: true }, { professional_id: "pr2", name: "Bruno Lima", has_cns: false } ], ...over };
}
export function territoryContext(over: Partial<TerritoryContext> = {}): TerritoryContext {
  return { teams: [ { ...TEAM_ESF, unit: { id: "u1", name: "UBS Vila Verde" },
    micro_areas: [ { ...microArea(), can_register: true }, { ...microArea({ id: "ma2", code: "02", name: "Vila Verde de baixo", acs: null }), can_register: false } ] } ], ...over };
}
export function family(over: Partial<Family> = {}): Family {
  return { id: "fa1", responsible: { patient_id: "pa1", display_name: "Maria das Dores" }, income_band: 3, members_count: 3,
    living_since: "2019-05", moved_out: false, ...over };
}
export function household(over: Partial<Household> = {}): Household {
  return {
    id: "hh1", micro_area: { id: "ma1", code: "01" },
    address: { zip: "81020000", street: "Rua das Flores", number: "120", complement: null, neighborhood_id: "nb1", reference: "perto da escola" },
    property_type: 1, living_conditions: { localizacao: 83, situacao_moradia_posse_terra: 75, nu_comodos: "4", nu_moradores: "3" },
    contact_phone: "41999990000", refused: false, status: "active", version: 2, registered_at: "2026-03-02T10:00:00-03:00",
    registered_by: { name: "Ana Prado" }, families: [ family() ], ...over
  };
}
export function registration(over: Partial<Registration> = {}): Registration {
  return {
    id: "rg1", patient: { id: "pa1", display_name: "Maria das Dores", birth_date: "1968-04-12", sex: 1, has_cpf: true, no_cpf_reason: null,
      cns_masked: null, identity_validated: true },
    household_id: "hh1", family_id: "fa1", is_responsible: true, sociodemographic: {}, health_conditions: { status_tem_diabetes: true },
    street_situation: null, refused: false, exit: null, status: "valid", version: 1, registered_at: "2026-03-02T10:00:00-03:00",
    registered_by: { name: "Ana Prado" }, identity_changed_at: null, ...over
  };
}
export function personRow(over: Partial<PersonRow> = {}): PersonRow {
  return { patient_id: "pa3", citizen_id: null, display_name: "Joaquim Ribeiro", age: 71, micro_area_code: "01", status: "expired",
    registered_at: "2024-09-01T10:00:00-03:00", registration_id: "rg3", ...over };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/api.territory.test.ts src/lib/features.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/api.territory.test.ts src/lib/features.ts src/lib/features.test.ts src/test/territoryFixtures.ts
$G commit -m "feat: add the territory registry api client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela que não dependem dos dicionários (`src/lib/territoryRegistry.ts`)

**Files:**
- Create: `src/lib/territoryRegistry.ts`
- Test: `src/lib/territoryRegistry.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `errorCode` (`api.ts`); `featureDisabledKey`, `hasFeature` (`features.ts`); `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `fmtDay` (`careLines.ts`); `isValidCpf`, `onlyDigits` (`attendance.ts`); `isValidCns` (`professionals.ts`); `describeActionError` (`actionErrors.ts`).
- Produces: `TERRITORY_DISABLED`, `TERRITORY_NO_ROLE`, `STALE_TEXT`, `canUseTerritory(user): boolean`, `territoryRoles(user): { professional: boolean; admin: boolean }`, re-export de `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `fmtDay`; as chaves `TERRITORY_CONTEXT_KEY`, `MICRO_AREAS_KEY`, `ADMIN_TEAMS_KEY`, `HOUSEHOLDS_KEY`, `HOUSEHOLD_KEY`, `RESIDENTS_KEY`, `REGISTRATION_KEY`, `REGISTRY_STATUS_KEY`, `REGISTRY_COUNT_KEY`; `registryStatusLabel(code)`, `registryStatusTone(code)`, `EXIT_REASONS`, `exitReasonLabel(code)`, `microAreaCodeProblem(code): string | null`, `microAreaNameProblem(name): string | null`, `duplicateQueryProblem(q, today): string | null`, `cpfProblem(cpf): string | null`, `cnsProblem(cns): string | null`, `territoryError(err): string`, `errorPath(err): string | null`, `isStale(err): boolean`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/territoryRegistry.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  TERRITORY_DISABLED, canUseTerritory, cnsProblem, cpfProblem, duplicateQueryProblem, errorPath, exitReasonLabel, isStale,
  microAreaCodeProblem, microAreaNameProblem, registryStatusLabel, registryStatusTone, territoryError, territoryRoles
} from "./territoryRegistry";

const user = (roles: string[], features: string[], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });
const refusal = (status: number, body: unknown) => new ApiError(status, body, `${status}`);

describe("papel e interruptor", () => {
  it("exige territory_registry E clinical_record, e nunca o operador", () => {
    expect(canUseTerritory(user([ "health_professional" ], [ "clinical_record", "territory_registry" ]))).toBe(true);
    expect(canUseTerritory(user([ "health_professional" ], [ "territory_registry" ]))).toBe(false);
    expect(canUseTerritory(user([ "health_professional" ], [ "clinical_record" ]))).toBe(false);
    expect(canUseTerritory(user([], [ "clinical_record", "territory_registry" ], true))).toBe(false);
    expect(canUseTerritory(null)).toBe(false);
    expect(territoryRoles(user([ "municipal_admin" ], []))).toEqual({ professional: false, admin: true });
  });
});

describe("rótulos (contrato §1)", () => {
  it.each([
    [ "valid", "válido", "ok" ], [ "due_soon", "vence em até 60 dias", "warn" ], [ "expired", "vencido (mais de 24 meses)", "down" ],
    [ "missing", "sem cadastro", "down" ], [ "refused", "recusou o cadastro", "neutral" ], [ "exited", "saiu do cadastro", "neutral" ],
    [ "novo_codigo", "novo_codigo", "neutral" ]
  ])("%s → %s (%s)", (code, label, tone) => {
    expect(registryStatusLabel(code)).toBe(label);
    expect(registryStatusTone(code)).toBe(tone);
  });
  it("saída", () => {
    expect(exitReasonLabel("moved")).toBe("mudou de território");
    expect(exitReasonLabel("death")).toBe("óbito");
    expect(exitReasonLabel("x")).toBe("x");
  });
});

describe("microárea", () => {
  it.each([ [ "01", null ], [ "99", null ], [ "00", "código de 01 a 99, com dois dígitos" ], [ "1", "código de 01 a 99, com dois dígitos" ],
    [ "100", "código de 01 a 99, com dois dígitos" ], [ "a1", "código de 01 a 99, com dois dígitos" ] ])("código %s", (code, problem) => {
    expect(microAreaCodeProblem(code)).toBe(problem);
  });
  it("nome de 1 a 120, recusado (nunca cortado)", () => {
    expect(microAreaNameProblem("  ")).toBe("informe o nome da microárea");
    expect(microAreaNameProblem("x".repeat(120))).toBeNull();
    expect(microAreaNameProblem("x".repeat(121))).toBe("nome com mais de 120 caracteres");
  });
});

describe("busca de duplicados e documentos", () => {
  it("nome com nome e sobrenome, nascimento até hoje; mãe opcional", () => {
    expect(duplicateQueryProblem({ full_name: "Maria", birth_date: "1960-01-01" }, "2026-10-23")).toBe("informe nome e sobrenome");
    expect(duplicateQueryProblem({ full_name: "Maria Silva", birth_date: "" }, "2026-10-23")).toBe("informe a data de nascimento");
    expect(duplicateQueryProblem({ full_name: "Maria Silva", birth_date: "2026-10-24" }, "2026-10-23")).toBe("nascimento depois de hoje");
    expect(duplicateQueryProblem({ full_name: "Maria Silva", birth_date: "2026-10-23" }, "2026-10-23")).toBeNull();
  });
  it("CPF e CNS pelo dígito", () => {
    expect(cpfProblem("529.982.247-25")).toBeNull();
    expect(cpfProblem("529.982.247-24")).toBe("CPF inválido: confira os 11 dígitos");
    expect(cnsProblem("")).toBeNull();
    expect(cnsProblem("123")).toBe("CNS inválido: confira os 15 dígitos");
  });
});

describe("recusas do api (contrato §3–§5)", () => {
  it.each([
    [ 409, "duplicate_code", "já existe microárea com este código na equipe" ],
    [ 422, "acs_not_in_team", "o ACS escolhido não tem vínculo com a unidade da equipe" ],
    [ 422, "acs_without_cns", "o ACS precisa ter CNS no cadastro de profissionais para ser responsável pela microárea" ],
    [ 422, "team_kind_not_allowed", "só equipes de saúde da família (70) e de atenção primária (76) têm microáreas" ],
    [ 409, "already_registered", "esta pessoa já tem cadastro individual ativo — abra o cadastro dela para atualizar" ],
    [ 409, "cpf_taken", "este CPF já pertence a outro paciente — confira o número; a junção de pacientes ainda não existe" ],
    [ 422, "cpf_or_reason_required", "informe o CPF ou o motivo de não ter CPF" ],
    [ 403, "cbo_not_allowed", "só o ACS da microárea, enfermeiro ou médico da equipe grava o cadastro" ],
    [ 403, "out_of_scope", "esta microárea não é da sua equipe" ],
    [ 409, "stale_version", "outra pessoa salvou este cadastro enquanto você editava — recarregue para ver a versão nova" ]
  ])("%s %s", (status, error, text) => {
    expect(territoryError(refusal(status, { error }))).toBe(text);
  });
  it("feature_disabled, path e stale", () => {
    expect(territoryError(refusal(403, { error: "feature_disabled", feature: "territory_registry" }))).toBe(TERRITORY_DISABLED);
    expect(errorPath(refusal(422, { error: "invalid_registration", path: "health_conditions.status_eh_gestante" })))
      .toBe("health_conditions.status_eh_gestante");
    expect(errorPath(new Error("x"))).toBeNull();
    expect(isStale(refusal(409, { error: "stale_version" }))).toBe(true);
    expect(territoryError(new TypeError("fetch failed"))).toBe("sem conexão com o servidor — tente de novo");
  });
  it("a frase nunca repete o que foi enviado", () => {
    expect(territoryError(refusal(422, { error: "invalid_registration", path: "identity.cpf", value: "52998224725" }))).not.toMatch(/\d{11}/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/territoryRegistry.test.ts`
Expected: FAIL — o módulo não existe.

- [ ] **Step 3: Implemente `src/lib/territoryRegistry.ts`**

`TERRITORY_DISABLED = "o cadastro individual e domiciliar está desligado nesta cidade"`, `TERRITORY_NO_ROLE = "seu papel não permite usar o cadastro territorial"`, `STALE_TEXT` igual à frase de `stale_version` do teste. `canUseTerritory` segue `canUseCareLines` (sessão, não operador, os dois interruptores). Re-exporte `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY` e `fmtDay` de `./careLines` (uma regra só para "lê sem trilha"). `territoryError`: `featureDisabledKey` primeiro; depois o mapa do teste mais `invalid_household`/`invalid_family`/`invalid_registration` ("confira o campo destacado"), `address_from_household` ("o endereço desta pessoa vem do cadastro domiciliar — altere pelo domicílio"), `not_found` ("não encontrado — recarregue"), `missing_role` (`TERRITORY_NO_ROLE`), `invalid_cursor` ("a lista mudou — recarregue"); fora do mapa, o mesmo fallback de `careLineError` (`describeActionError`, depois `não foi possível concluir (<status>) — tente de novo`), **nunca** o corpo da resposta. `errorPath` lê `body.path` (string) de um `ApiError`. Nenhuma função loga nada.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/territoryRegistry.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/territoryRegistry.ts src/lib/territoryRegistry.test.ts
$G commit -m "feat: add territory registry screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Grupo "Cadastro territorial", catálogo do shell e portão

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`
- Create: `src/modules/territory/TerritoryGate.tsx`, `src/modules/TerritoryMicroAreas.tsx`, `src/modules/TerritoryTeam.tsx`, `src/modules/RegistryStatus.tsx` (as três telas nascem como portão com `EmptyState` "em construção"; as Tasks 4, 8 e 9 trocam o corpo)
- Test: `src/shell/modules.territory.test.ts`

**Interfaces:**
- Consumes: `canUseTerritory`, `territoryRoles`, `TERRITORY_DISABLED`, `TERRITORY_NO_ROLE` (Task 2).
- Produces: `ModuleId` ganha `"territory-team" | "registry-status" | "territory-micro-areas"`; o grupo `{ label: "Cadastro territorial", items: [ territory-team "Território da equipe" ⌂, registry-status "Validade dos cadastros" ⏲, territory-micro-areas "Microáreas" ⬡ ] }` entre "Linhas de cuidado" e "Exames"; `TerritoryGate({ title, sub, allow(roles), children(roles) })`; `TerritoryTeam({ today? })`, `RegistryStatus({ today?, onRegister? })`, `TerritoryMicroAreas({ onGoToSecurity? })`.

- [ ] **Step 1: Write the failing test**

```ts
// src/shell/modules.territory.test.ts
import { describe, expect, it } from "vitest";
import { NAV_GROUPS, navGroupsFor } from "./modules";
import { CATALOG } from "./catalog";

const user = (roles: string[], features = [ "clinical_record", "territory_registry" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });
const itemsOf = (groups: ReturnType<typeof navGroupsFor>) =>
  groups.find((g) => g.label === "Cadastro territorial")?.items.map((i) => i.id) ?? [];

describe("grupo Cadastro territorial (módulo 24a)", () => {
  it("profissional vê território e validade; admin vê microáreas", () => {
    expect(itemsOf(navGroupsFor(user([ "health_professional" ])))).toEqual([ "territory-team", "registry-status" ]);
    expect(itemsOf(navGroupsFor(user([ "municipal_admin" ])))).toEqual([ "territory-micro-areas" ]);
    expect(itemsOf(navGroupsFor(user([ "municipal_admin", "health_professional" ])))).toEqual([ "territory-team", "registry-status", "territory-micro-areas" ]);
  });
  it("some sem o interruptor, sem o prontuário, para a recepção e para o operador", () => {
    expect(itemsOf(navGroupsFor(user([ "health_professional" ], [ "clinical_record" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "health_professional" ], [ "territory_registry" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([ "citizen_verifier" ])))).toEqual([]);
    expect(itemsOf(navGroupsFor(user([], undefined, true)))).toEqual([]);
  });
  it("vem depois de Linhas de cuidado e tem ícone e descrição", () => {
    const labels = NAV_GROUPS.map((g) => g.label);
    expect(labels.indexOf("Cadastro territorial")).toBe(labels.indexOf("Linhas de cuidado") + 1);
    expect(CATALOG.groups["Cadastro territorial"].description).toBe("Microáreas, domicílios, famílias e cadastro individual");
    expect(CATALOG.screens["registry-status"].description).toBe("Vencidos, a vencer e quem falta cadastrar");
  });
});
```

Em `src/shell/modules.test.ts`, nos dois testes de "sem papel" e "sem sessão", acrescente `expect(groups.some((g) => g.label === "Cadastro territorial")).toBe(false);` e troque `NAV_GROUPS.length - 11` por `NAV_GROUPS.length - 12` e `NAV_GROUPS.length - 10` por `NAV_GROUPS.length - 11`.

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/shell/modules.territory.test.ts src/shell/modules.test.ts src/shell/catalog.test.ts`
Expected: FAIL — o grupo não existe; as contagens antigas mudam.

- [ ] **Step 3: Implemente**

- `modules.ts`: os três ids no `ModuleId`; o grupo no `NAV_GROUPS` (ícones de menu `⌂`, `⏲`, `⬡`); em `navGroupsFor`, `const territoryOn = canUseTerritory(user);` (comentário: "Módulo 24a (ADR 0042): `territory_registry` E `clinical_record`; território e validade do profissional — o api confere CBO e escopo —, microáreas do admin") e os filtros `territory-team`/`registry-status` → `territoryOn && isProfessional`, `territory-micro-areas` → `territoryOn && isAdmin`.
- `catalog.ts`: importe `IconHomeHeart, IconMap2, IconCalendarDue, IconHexagons`; grupo `"Cadastro territorial": { icon: IconHomeHeart, description: "Microáreas, domicílios, famílias e cadastro individual" }`; telas `"territory-team": { icon: IconMap2, description: "Domicílios, famílias e moradores por microárea" }`, `"registry-status": { icon: IconCalendarDue, description: "Vencidos, a vencer e quem falta cadastrar" }`, `"territory-micro-areas": { icon: IconHexagons, description: "Microáreas das equipes e o ACS responsável" }`.
- `TerritoryGate.tsx`: cópia do molde do `CareLinesGate` com `canUseTerritory`/`territoryRoles` e as frases da Task 2.
- As três telas: `TerritoryGate` com o título ("Território da equipe", "Validade dos cadastros", "Microáreas") e, por enquanto, `EmptyState title="em construção"`.
- `App.tsx`: os três `case` depois de `care-lines-admin` (`<TerritoryMicroAreas onGoToSecurity={() => setActive("security")} />`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/shell && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.territory.test.ts src/shell/catalog.ts src/App.tsx src/modules/territory/TerritoryGate.tsx src/modules/TerritoryMicroAreas.tsx src/modules/TerritoryTeam.tsx src/modules/RegistryStatus.tsx
$G commit -m "feat: add the territory registry menu group

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Microáreas — criar, renomear, trocar o ACS e desativar (admin, step-up)

**Files:**
- Modify: `src/modules/TerritoryMicroAreas.tsx`
- Test: `src/modules/TerritoryMicroAreas.test.tsx`

**Interfaces:**
- Consumes: `listTerritoryAdminTeams`, `listMicroAreas`, `createMicroArea`, `updateMicroArea` (Task 1); `ADMIN_TEAMS_KEY`, `MICRO_AREAS_KEY`, `STATIC_QUERY`, `microAreaCodeProblem`, `microAreaNameProblem`, `territoryError` (Task 2); `SensitiveAction`.
- Produces: tela `TerritoryMicroAreas({ onGoToSecurity? })`.

Comportamento: seletor de equipe (as de `listTerritoryAdminTeams`, rótulo `"<nome> · INE <ine>"`); `DataTable` das microáreas da equipe (código `card: "title"`, nome, ACS ou "sem ACS", situação com `Tag` "ativa"/"inativa", ações `card: "actions"`: Renomear, Trocar ACS, Desativar/Reativar); "Nova microárea" com código, nome e ACS (opcional; só os `acs_candidates` da equipe; candidato com `has_cns: false` aparece desabilitado com "(sem CNS)"). O formulário confere código e nome **antes** de abrir o `SensitiveAction`; toda escrita vai pelo `SensitiveAction` com `requiresStepUp` e `translateError={territoryError}`; depois, invalida `MICRO_AREAS_KEY` e mostra o aviso. A desativação diz antes do Confirmar: "Os domicílios ficam guardados; a microárea some das novas fichas."

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/TerritoryMicroAreas.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listTerritoryAdminTeams: vi.fn(), listMicroAreas: vi.fn(), createMicroArea: vi.fn(),
    updateMicroArea: vi.fn(), stepUpMfa: vi.fn() };
});

import * as api from "../lib/api";
import { TerritoryMicroAreas } from "./TerritoryMicroAreas";
import { renderWithProviders } from "../test/campaignFixtures";
import { adminTeam, microArea, territoryUser } from "../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.listTerritoryAdminTeams, api.listMicroAreas, api.createMicroArea, api.updateMicroArea ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser([ "municipal_admin" ]));
  mocked(api.listTerritoryAdminTeams).mockResolvedValue([ adminTeam() ]);
  mocked(api.listMicroAreas).mockResolvedValue([ microArea() ]);
  mocked(api.createMicroArea).mockResolvedValue(microArea({ id: "ma9", code: "02" }));
  mocked(api.updateMicroArea).mockResolvedValue(microArea());
});

const typeIn = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("TerritoryMicroAreas", () => {
  it("lista as microáreas da equipe com o ACS", async () => {
    renderWithProviders(<TerritoryMicroAreas />);
    expect(await screen.findByText("Vila Verde de cima")).not.toBeNull();
    expect(screen.getByText("Ana Prado")).not.toBeNull();
    expect(api.listMicroAreas).toHaveBeenCalledWith("tm1");
  });

  it("cria com step-up; código ruim é recusado antes de abrir a confirmação", async () => {
    renderWithProviders(<TerritoryMicroAreas />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova microárea" }));
    typeIn("Código", "0");
    typeIn("Nome", "Rua das Flores");
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(screen.getByRole("alert").textContent).toBe("código de 01 a 99, com dois dígitos");
    typeIn("Código", "02");
    fireEvent.change(screen.getByLabelText("ACS responsável"), { target: { value: "pr1" } });
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar microárea" }));
    await waitFor(() => expect(api.createMicroArea).toHaveBeenCalledWith({ team_id: "tm1", code: "02", name: "Rua das Flores", acs_professional_id: "pr1" }));
    expect((await screen.findByRole("status")).textContent).toBe("Microárea 02 criada.");
  });

  it("ACS sem CNS aparece desabilitado; 409 duplicate_code vira frase e o formulário fica", async () => {
    mocked(api.createMicroArea).mockRejectedValue(new api.ApiError(409, { error: "duplicate_code" }, "409"));
    renderWithProviders(<TerritoryMicroAreas />);
    fireEvent.click(await screen.findByRole("button", { name: "Nova microárea" }));
    expect((screen.getByRole("option", { name: "Bruno Lima (sem CNS)" }) as HTMLOptionElement).disabled).toBe(true);
    typeIn("Código", "01");
    typeIn("Nome", "Repetida");
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar microárea" }));
    expect(await screen.findByText("já existe microárea com este código na equipe")).not.toBeNull();
  });

  it("desativar diz o efeito antes e manda active: false", async () => {
    renderWithProviders(<TerritoryMicroAreas />);
    fireEvent.click(await screen.findByRole("button", { name: "Desativar 01" }));
    expect(screen.getByText("Os domicílios ficam guardados; a microárea some das novas fichas.")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Desativar microárea" }));
    await waitFor(() => expect(api.updateMicroArea).toHaveBeenCalledWith("ma1", { active: false }));
  });

  it("sem a janela de step-up, pede o código antes de gravar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser([ "municipal_admin" ], { mfa_verified_at: null }));
    renderWithProviders(<TerritoryMicroAreas />);
    fireEvent.click(await screen.findByRole("button", { name: "Renomear 01" }));
    typeIn("Novo nome", "Vila Verde alta");
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(await screen.findByLabelText(/código/i)).not.toBeNull();
    expect(api.updateMicroArea).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/TerritoryMicroAreas.test.tsx`
Expected: FAIL — a tela ainda é o portão vazio.

- [ ] **Step 3: Implemente o corpo de `TerritoryMicroAreas`** (`allow={(r) => r.admin}`; consultas com `STATIC_QUERY`; estado `pending` no molde do `CareLinesAdmin`: `{ kind: "create", input } | { kind: "rename", area, name } | { kind: "acs", area, id } | { kind: "toggle", area }`; `confirmLabel` "Criar microárea", "Renomear microárea", "Trocar ACS", "Desativar microárea"/"Reativar microárea"; avisos "Microárea <código> criada.", "Microárea <código> renomeada.", "ACS da microárea <código> trocado.", "Microárea <código> desativada."/"reativada.").

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/TerritoryMicroAreas.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS (as guardas de tema também).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/TerritoryMicroAreas.tsx src/modules/TerritoryMicroAreas.test.tsx
$G commit -m "feat: let the municipal admin manage micro-areas

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: ACS e agente de endemias sem conselho (F-24.2)

**Files:**
- Modify: `src/lib/api.ts` (`Professional`: `council`, `council_state`, `registration_number` passam a `string | null`), `src/lib/professionals.ts`, `src/modules/professionals/ProfileForm.tsx`, `src/modules/professionals/ProfessionalDetail.tsx` (onde mostra o conselho)
- Test: `src/modules/professionals/ProfileForm.test.tsx` (novo)

**Interfaces:**
- Consumes: `ProfileForm` e `ProfessionalFields` existentes.
- Produces: `NO_COUNCIL = "none"` e `councilText(p: { council: string | null; council_state: string | null; registration_number: string | null }): string` em `professionals.ts` ("CRM-PR 12345" ou "sem conselho (ACS ou agente de endemias)"); `ProfileForm` com a opção "Sem conselho (ACS ou agente de endemias)", que esconde UF e número e envia `council: null, council_state: null, registration_number: null`. CNS continua obrigatório no perfil novo (contrato §7). O api recusa um vínculo de outro CBO com perfil sem conselho (`council_mismatch`, frase já existente).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/professionals/ProfileForm.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { ProfileForm } from "./ProfileForm";
import { councilText } from "../../lib/professionals";

afterEach(cleanup);
const typeIn = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });

describe("ProfileForm — sem conselho (módulo 24a, F-24.2)", () => {
  it("ACS: sem conselho esconde UF e número e manda nulos; CNS continua obrigatório", async () => {
    const onSubmit = vi.fn().mockResolvedValue(undefined);
    render(<ProfileForm submitLabel="Criar perfil" onSubmit={onSubmit} />);
    typeIn("Nome profissional", "Ana Prado");
    fireEvent.change(screen.getByLabelText("Conselho"), { target: { value: "none" } });
    expect(screen.queryByLabelText("UF do conselho")).toBeNull();
    expect(screen.queryByLabelText("Número do registro")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Criar perfil" }));
    expect(screen.getByRole("alert").textContent).toBe("CNS inválido: confira os 15 dígitos");
    typeIn("CNS", "898001160660003");
    fireEvent.click(screen.getByRole("button", { name: "Criar perfil" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith(expect.objectContaining({
      professional_name: "Ana Prado", council: null, council_state: null, registration_number: null, cns: "898001160660003" })));
  });

  it("perfil sem conselho abre com a opção marcada; texto do conselho", () => {
    render(<ProfileForm submitLabel="Salvar" initial={{ id: "p1", council: null, council_state: null, registration_number: null }} onSubmit={vi.fn()} />);
    expect((screen.getByLabelText("Conselho") as HTMLSelectElement).value).toBe("none");
    expect(councilText({ council: null, council_state: null, registration_number: null })).toBe("sem conselho (ACS ou agente de endemias)");
    expect(councilText({ council: "COREN", council_state: "PR", registration_number: "123456" })).toBe("COREN-PR 123456");
  });
});
```

(O CNS `898001160660003` passa no `isValidCns` existente; se o algoritmo do arquivo recusar, troque por outro CNS válido da semente — anote.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/professionals/ProfileForm.test.tsx`
Expected: FAIL — não há a opção "none" nem `councilText`.

- [ ] **Step 3: Implemente** a opção `<option value="none">Sem conselho (ACS ou agente de endemias)</option>` no fim do `select`, o estado inicial `initial.council === null && initial.id ? "none" : initial.council ?? "CRM"`, e o `Object.assign` com nulos quando `council === "none"`; `ProfessionalFields` aceita `null` nesses três campos; `ProfessionalDetail` usa `councilText` onde hoje concatena conselho, UF e número.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/professionals src/modules/Professionals.test.tsx src/lib && cd $W && npx tsc --noEmit`
Expected: PASS (os testes antigos do perfil seguem iguais).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/professionals.ts src/modules/professionals/ProfileForm.tsx src/modules/professionals/ProfileForm.test.tsx src/modules/professionals/ProfessionalDetail.tsx
$G commit -m "feat: allow field agents without a council registration

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: "sem CPF" e o CNS nas telas de recepção e do paciente

**Files:**
- Create: `src/lib/patientDocument.ts`, `src/lib/patientDocument.test.ts`
- Modify: `src/lib/api.ts` (`RecordPatient`, `BillingPatient`, `ExamPatient`, `RequestRow`, `QueueRow`, `AppointmentView.citizen`: `cpf_masked: string | null` e `cns_masked?: string | null` — Divergência T10), `src/lib/queueLabels.ts`, `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/billing/BillingPatientLookup.tsx`, `src/modules/exams/PatientLookup.tsx`, `src/modules/attendance/Requests.tsx`, `src/modules/attendance/UnassignedRequests.tsx`, `src/modules/attendance/BookPanel.tsx`, `src/modules/attendance/Agenda.tsx`, `src/modules/attendance/UnitQueue.tsx`

**Interfaces:**
- Produces: `patientDocument(p: { cpf_masked?: string | null; cns_masked?: string | null }): { k: "CPF" | "CNS"; v: string }` e `documentText(p): string`: com CPF → `{ k: "CPF", v: cpf }` / `"CPF ***.982.247-**"`; sem CPF com CNS → `{ k: "CNS", v: "sem CPF · CNS 7** **** **** 1234" }` / `"sem CPF · CNS 7** **** **** 1234"`; sem os dois → `{ k: "CPF", v: "sem CPF" }` / `"sem CPF"`. `personLabel` aceita `cpf_masked: string | null` e usa `documentText`.

As telas da recepção que partem do **cidadão** pelo CPF (balcão, check-in, abertura justificada) não mudam: lá o CPF é a chave.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/patientDocument.test.ts
import { describe, expect, it } from "vitest";
import { documentText, patientDocument } from "./patientDocument";
import { personLabel } from "./queueLabels";

describe("documento do paciente (contrato §10)", () => {
  it("CPF, sem CPF com CNS, sem os dois — nunca undefined/null", () => {
    expect(patientDocument({ cpf_masked: "***.982.247-**" })).toEqual({ k: "CPF", v: "***.982.247-**" });
    expect(patientDocument({ cpf_masked: null, cns_masked: "7** **** **** 1234" })).toEqual({ k: "CNS", v: "sem CPF · CNS 7** **** **** 1234" });
    expect(patientDocument({ cpf_masked: null })).toEqual({ k: "CPF", v: "sem CPF" });
    expect(documentText({ cpf_masked: "***.982.247-**" })).toBe("CPF ***.982.247-**");
    expect(documentText({ cpf_masked: null, cns_masked: null })).toBe("sem CPF");
  });
  it("fila: sem nome e sem CPF, o título é 'sem CPF'", () => {
    expect(personLabel({ display_name: null, cpf_masked: null })).toEqual({ title: "sem CPF", sub: null });
    expect(personLabel({ display_name: "José", cpf_masked: null, cns_masked: "7** **** **** 1234" }))
      .toEqual({ title: "José", sub: "sem CPF · CNS 7** **** **** 1234" });
  });
});
```

E um caso por tela que mostra paciente, no arquivo de teste existente dela (com o dado da fixture existente trocado por `cpf_masked: null`): `PatientPanel` (`src/modules/ClinicalRecord.test.tsx`: "paciente sem CPF mostra 'sem CPF' e o CNS"), `BillingPatientLookup` (`src/modules/BillingProcedures.test.tsx`), `Requests` e `BookPanel` (`src/modules/Attendance.test.tsx` ou o teste de agendamento existente), cada um com `expect(document.body.textContent).toMatch(/sem CPF/)` e `not.toMatch(/undefined|null/)`.

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/patientDocument.test.ts src/modules/ClinicalRecord.test.tsx src/modules/BillingProcedures.test.tsx src/modules/Attendance.test.tsx`
Expected: FAIL — o ajudante não existe e as telas mostram vazio ou "null".

- [ ] **Step 3: Implemente** `patientDocument.ts` e troque cada exibição: `KeyValue k="CPF" v={patient.cpf_masked}` → `const doc = patientDocument(patient); <KeyValue k={doc.k} v={doc.v} />`; os textos `CPF ${x.cpf_masked}` → `documentText(x)`; as colunas `CPF` de `Requests`, `UnassignedRequests` e `Agenda` → rótulo "Documento" e `documentText(r)`; `BookPanel` e `UnitQueue` → `documentText`. Em `ConsultationWorkspace`, `patientCpfMasked: record.data.patient.cpf_masked ?? undefined`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/lib src/modules/ClinicalRecord.test.tsx src/modules/BillingProcedures.test.tsx src/modules/Attendance.test.tsx src/modules/attendance src/modules/exams && cd $W && npx tsc --noEmit`
Expected: PASS; testes antigos que procuravam o cabeçalho "CPF" nas três colunas trocadas passam a procurar "Documento" (ajuste explícito, um por arquivo).

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/patientDocument.ts src/lib/patientDocument.test.ts src/lib/api.ts src/lib/queueLabels.ts src/modules/consultation/PatientPanel.tsx src/modules/consultation/ConsultationWorkspace.tsx src/modules/billing/BillingPatientLookup.tsx src/modules/exams/PatientLookup.tsx src/modules/attendance/Requests.tsx src/modules/attendance/UnassignedRequests.tsx src/modules/attendance/BookPanel.tsx src/modules/attendance/Agenda.tsx src/modules/attendance/UnitQueue.tsx src/modules/ClinicalRecord.test.tsx src/modules/BillingProcedures.test.tsx src/modules/Attendance.test.tsx
$G commit -m "feat: show patients without cpf by cns or as without cpf

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se o Step 1 tocou outro arquivo de teste existente, acrescente-o pelo caminho.)

---

### Task 7: Endereço do cadastro domiciliar só leitura nos dados do paciente (módulo 22)

**Files:**
- Modify: `src/lib/api.ts` (`PendingItem.patient` e `CheckInCitizen` ganham `address_from_household?: boolean` — Divergência T9), `src/lib/demographics.ts`, `src/lib/demographics.test.ts`, `src/modules/attendance/DemographicsForm.tsx`, `src/modules/billing/PendingPanel.tsx`, `src/modules/attendance/CheckIn.tsx`
- Test: `src/modules/attendance/DemographicsForm.test.tsx` (existente: acrescente o `describe` abaixo no fim; os imports que faltarem vão para o topo)

**Interfaces:**
- Consumes: `DemographicsFields`, `DemographicsForm`, `demographicsBody`, `demographicsFilled`, `demographicsError` existentes.
- Produces: `ADDRESS_FROM_HOUSEHOLD = "Endereço: do cadastro domiciliar — altere pelo domicílio, no Cadastro territorial."`; `DemographicsFields` e `DemographicsForm` aceitam `addressLocked?: boolean`; com ele, os cinco campos de endereço somem, a nota aparece, `demographicsFilled` só olha raça/cor e `demographicsBody(draft, { addressLocked: true })` nunca leva `address_*`; `demographicsError` traduz 422 `address_from_household` com a frase acima. `PendingPanel` passa `addressLocked={fixing.patient?.address_from_household === true}` e tira `"address"` de `missing` nesse caso; `CheckIn` idem com `found.citizen.address_from_household`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/attendance/DemographicsForm.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { ApiError } from "../../lib/api";
import { ADDRESS_FROM_HOUSEHOLD } from "../../lib/demographics";
import { DemographicsForm } from "./DemographicsForm";

afterEach(cleanup);

describe("DemographicsForm — endereço do cadastro domiciliar (módulo 24a)", () => {
  it("com cadastro ativo: sem campos de endereço, com a nota; salva só a raça/cor", async () => {
    const save = vi.fn().mockResolvedValue(undefined);
    render(<DemographicsForm title="Completar dados" missing={[ "race_color" ]} addressLocked save={save} />);
    expect(screen.queryByLabelText("CEP")).toBeNull();
    expect(screen.queryByLabelText("Logradouro")).toBeNull();
    expect(screen.getByText(ADDRESS_FROM_HOUSEHOLD)).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Raça/cor"), { target: { value: "03" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar dados do paciente" }));
    await waitFor(() => expect(save).toHaveBeenCalledWith({ race_color: "03" }));
  });

  it("sem cadastro: continua como hoje (campos de endereço)", () => {
    render(<DemographicsForm title="Completar dados" save={vi.fn()} />);
    expect(screen.getByLabelText("CEP")).not.toBeNull();
    expect(screen.queryByText(ADDRESS_FROM_HOUSEHOLD)).toBeNull();
  });

  it("422 address_from_household vira a frase do cadastro domiciliar", async () => {
    const save = vi.fn().mockRejectedValue(new ApiError(422, { error: "address_from_household" }, "422"));
    render(<DemographicsForm title="Completar dados" save={save} />);
    fireEvent.change(screen.getByLabelText("CEP"), { target: { value: "81020000" } });
    fireEvent.change(screen.getByLabelText("Logradouro"), { target: { value: "Rua das Flores" } });
    fireEvent.change(screen.getByLabelText("Número"), { target: { value: "120" } });
    fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "Vila Verde" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar dados do paciente" }));
    expect((await screen.findByRole("alert")).textContent).toBe(ADDRESS_FROM_HOUSEHOLD);
  });
});
```

Em `src/lib/demographics.test.ts`: `demographicsBody({ ...EMPTY_DEMOGRAPHICS, raceColor: "01", zip: "81020000" }, { addressLocked: true })` → `{ race_color: "01" }`; e um caso em `src/modules/BillingPeriods.demographics.test.tsx` com `patient: { id: "pa1", display_name: "Joana Lima", address_from_household: true }` e `reasons: [ "missing_race_color" ]`: o formulário da pendência sem "CEP" e com a nota.

(Raça/cor no código de 2 dígitos que `RACE_COLORS` do 22 já usa: "03" parda.)

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/attendance/DemographicsForm.test.tsx src/lib/demographics.test.ts src/modules/BillingPeriods.demographics.test.tsx`
Expected: FAIL — não há `addressLocked` nem a frase.

- [ ] **Step 3: Implemente** como descrito em Produces; sem `addressLocked`, nada muda (os testes do 22 seguem iguais).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/attendance src/lib/demographics.test.ts src/modules/BillingPeriods.demographics.test.tsx src/modules/Attendance.demographics.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/lib/api.ts src/lib/demographics.ts src/lib/demographics.test.ts src/modules/attendance/DemographicsForm.tsx src/modules/attendance/DemographicsForm.test.tsx src/modules/billing/PendingPanel.tsx src/modules/attendance/CheckIn.tsx src/modules/BillingPeriods.demographics.test.tsx
$G commit -m "feat: show the household address as read-only for registered patients

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Território da equipe — microáreas → domicílios → famílias → moradores

**Files:**
- Modify: `src/modules/TerritoryTeam.tsx`
- Create: `src/modules/territory/HouseholdList.tsx`, `src/modules/territory/HouseholdDetail.tsx`
- Test: `src/modules/TerritoryTeam.test.tsx`

**Interfaces:**
- Consumes: `getTerritoryContext`, `listHouseholds`, `getHousehold`, `listRegistrations` (Task 1); `TERRITORY_CONTEXT_KEY`, `HOUSEHOLDS_KEY`, `HOUSEHOLD_KEY`, `RESIDENTS_KEY`, `NOMINAL_QUERY`, `STATIC_QUERY`, `registryStatusLabel`, `registryStatusTone`, `fmtDay`, `territoryError` (Task 2); `patientDocument` (Task 6).
- Produces: `TerritoryTeam({ today? })`; `HouseholdList({ microAreaId, canRegister, onOpen(id), actions? })`; `HouseholdDetail({ householdId, canRegister, onBack(), actions? })` — os slots `actions` (`ReactNode`) recebem, nas Tasks 11–13, "Novo domicílio", "Editar domicílio", "Nova família" e "Adicionar morador"/"Abrir cadastro".

Comportamento: contexto com `STATIC_QUERY` (sem pessoa); seletor de equipe (só se houver mais de uma) e de microárea (`"<código> · <nome>"`); lista de domicílios da microárea (`NOMINAL_QUERY`, `DataTable`: endereço `card: "title"` — "Rua das Flores, 120" —, tipo de imóvel (código cru até a Task 10 dar o rótulo), famílias, situação `Tag`, `registered_at` com `fmtDay`; "Carregar mais" com `next_before`); "Abrir" mostra o detalhe (domicílio e moradores lidos uma vez cada, `NOMINAL_QUERY`): endereço, ponto de referência, famílias (responsável, número de membros, "mudou-se") e moradores (`listRegistrations({ householdId })`: nome, documento por `patientDocument`, situação, responsável). Microárea com `can_register: false` mostra "só leitura nesta microárea" e esconde as ações. Nenhum dado de moradia ou condição aparece na lista.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/TerritoryTeam.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen } from "@testing-library/react";
import { focusManager, onlineManager } from "@tanstack/react-query";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getTerritoryContext: vi.fn(), listHouseholds: vi.fn(), getHousehold: vi.fn(), listRegistrations: vi.fn() };
});

import * as api from "../lib/api";
import { TerritoryTeam } from "./TerritoryTeam";
import { TERRITORY_DISABLED } from "../lib/territoryRegistry";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY24, household, registration, territoryContext, territoryUser } from "../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getTerritoryContext, api.listHouseholds, api.getHousehold, api.listRegistrations ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser());
  mocked(api.getTerritoryContext).mockResolvedValue(territoryContext());
  mocked(api.listHouseholds).mockResolvedValue({ households: [ household() ], next_before: null });
  mocked(api.getHousehold).mockResolvedValue(household());
  mocked(api.listRegistrations).mockResolvedValue([
    registration(),
    registration({ id: "rg2", is_responsible: false, status: "expired",
      patient: { id: "pa2", display_name: "José das Dores", birth_date: "2026-09-30", sex: 0, has_cpf: false, no_cpf_reason: 3, cns_masked: "7** **** **** 1234" } })
  ]);
});

describe("TerritoryTeam", () => {
  it("abre na primeira microárea; domicílio → famílias → moradores com a situação", async () => {
    renderWithProviders(<TerritoryTeam today={TODAY24} />);
    expect(await screen.findByText("Rua das Flores, 120")).not.toBeNull();
    expect(api.listHouseholds).toHaveBeenCalledWith({ microAreaId: "ma1", before: null });
    fireEvent.click(screen.getByRole("button", { name: "Abrir Rua das Flores, 120" }));
    expect(await screen.findByText("José das Dores")).not.toBeNull();
    expect(api.listRegistrations).toHaveBeenCalledWith({ householdId: "hh1" });
    expect(screen.getByText("sem CPF · CNS 7** **** **** 1234")).not.toBeNull();
    expect(screen.getByText("vencido (mais de 24 meses)")).not.toBeNull();
    expect(screen.getByText("Responsável: Maria das Dores · 3 pessoas")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|status_tem_diabetes/);
  });

  it("microárea sem permissão de gravar: só leitura", async () => {
    renderWithProviders(<TerritoryTeam today={TODAY24} />);
    await screen.findByText("Rua das Flores, 120");
    fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma2" } });
    expect(await screen.findByText("só leitura nesta microárea")).not.toBeNull();
    expect(api.listHouseholds).toHaveBeenLastCalledWith({ microAreaId: "ma2", before: null });
  });

  it("volta à aba e reconecta: lista e detalhe não releem", async () => {
    renderWithProviders(<TerritoryTeam today={TODAY24} />);
    fireEvent.click(await screen.findByRole("button", { name: "Abrir Rua das Flores, 120" }));
    await screen.findByText("José das Dores");
    act(() => { focusManager.setFocused(false); focusManager.setFocused(true); onlineManager.setOnline(false); onlineManager.setOnline(true); });
    expect(api.listHouseholds).toHaveBeenCalledTimes(1);
    expect(api.getHousehold).toHaveBeenCalledTimes(1);
    expect(api.listRegistrations).toHaveBeenCalledTimes(1);
  });

  it("403 feature_disabled vira a frase do interruptor", async () => {
    mocked(api.getTerritoryContext).mockRejectedValue(new api.ApiError(403, { error: "feature_disabled", feature: "territory_registry" }, "403"));
    renderWithProviders(<TerritoryTeam today={TODAY24} />);
    expect(await screen.findByText(TERRITORY_DISABLED)).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/TerritoryTeam.test.tsx`
Expected: FAIL — a tela é o portão vazio.

- [ ] **Step 3: Implemente** `TerritoryTeam` (`allow={(r) => r.professional}`), `HouseholdList` e `HouseholdDetail` como descrito; a linha da família é exatamente `Responsável: <nome> · <n> pessoas` (`1 pessoa` no singular), com "mudou-se" em `Tag tone="neutral"` quando `moved_out`.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/TerritoryTeam.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/TerritoryTeam.tsx src/modules/territory/HouseholdList.tsx src/modules/territory/HouseholdDetail.tsx src/modules/TerritoryTeam.test.tsx
$G commit -m "feat: browse the team territory by micro-area and household

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Validade de 24 meses e quem falta cadastrar

**Files:**
- Modify: `src/modules/RegistryStatus.tsx`
- Test: `src/modules/RegistryStatus.test.tsx`

**Interfaces:**
- Consumes: `getTerritoryContext`, `listRegistryStatus`, `countRegistryStatus` (Task 1); `REGISTRY_STATUS_KEY`, `REGISTRY_COUNT_KEY`, `NOMINAL_QUERY`, `COUNT_QUERY`, `STATIC_QUERY`, `registryStatusLabel`, `fmtDay`, `territoryError` (Task 2).
- Produces: `RegistryStatus({ today?, onRegister?(row: PersonRow) })`; `onRegister` liga o atalho "Cadastrar" (sem cadastro) / "Atualizar" (vencido ou a vencer); a Task 12 passa a abrir a ficha aqui mesmo.

Comportamento: equipe e microárea ("todas" = sem `micro_area_id`); quatro números (válidos, a vencer em 60 dias, vencidos, sem cadastro) de `countRegistryStatus` com `COUNT_QUERY` (sem trilha); abas `expired` "Vencidos (N)", `due_soon` "A vencer (N)", `missing` "Sem cadastro (N)" — a lista nominal (`NOMINAL_QUERY`) lê ao abrir a aba e ao trocar filtro; `DataTable`: nome `card: "title"` ("cidadão sem nome no cadastro" quando nulo), idade, microárea, último cadastro (`fmtDay` ou "—"), ação `card: "actions"`. "Carregar mais" com `next_before`. Sem `onRegister`, a ação não aparece.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/RegistryStatus.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen } from "@testing-library/react";
import { focusManager, onlineManager } from "@tanstack/react-query";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getTerritoryContext: vi.fn(), listRegistryStatus: vi.fn(), countRegistryStatus: vi.fn() };
});

import * as api from "../lib/api";
import { RegistryStatus } from "./RegistryStatus";
import { COUNT_QUERY } from "../lib/territoryRegistry";
import { renderWithProviders } from "../test/campaignFixtures";
import { TODAY24, personRow, territoryContext, territoryUser } from "../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(() => { cleanup(); vi.useRealTimers(); });
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.getTerritoryContext, api.listRegistryStatus, api.countRegistryStatus ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser());
  mocked(api.getTerritoryContext).mockResolvedValue(territoryContext());
  mocked(api.countRegistryStatus).mockResolvedValue({ valid: 120, due_soon: 8, expired: 31, missing: 44 });
  mocked(api.listRegistryStatus).mockResolvedValue({ people: [ personRow() ], next_before: null });
});

describe("RegistryStatus", () => {
  it("contadores sem trilha e a lista de vencidos", async () => {
    renderWithProviders(<RegistryStatus today={TODAY24} />);
    expect(await screen.findByRole("tab", { name: "Vencidos (31)" })).not.toBeNull();
    expect(screen.getByRole("tab", { name: "Sem cadastro (44)" })).not.toBeNull();
    expect(await screen.findByText("Joaquim Ribeiro")).not.toBeNull();
    expect(api.listRegistryStatus).toHaveBeenCalledWith({ teamId: "tm1", microAreaId: undefined, status: "expired", before: null });
    expect(api.countRegistryStatus).toHaveBeenCalledWith("tm1", undefined);
  });

  it("troca de aba e de microárea relê com os filtros; atalho por linha", async () => {
    const onRegister = vi.fn();
    mocked(api.listRegistryStatus).mockResolvedValue({ people: [ personRow({ status: "missing", registered_at: null, registration_id: null, patient_id: null, citizen_id: "ci4", display_name: null }) ], next_before: null });
    renderWithProviders(<RegistryStatus today={TODAY24} onRegister={onRegister} />);
    fireEvent.click(await screen.findByRole("tab", { name: "Sem cadastro (44)" }));
    fireEvent.change(screen.getByLabelText("Microárea"), { target: { value: "ma1" } });
    expect(api.listRegistryStatus).toHaveBeenLastCalledWith({ teamId: "tm1", microAreaId: "ma1", status: "missing", before: null });
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar cidadão sem nome no cadastro" }));
    expect(onRegister).toHaveBeenCalledWith(expect.objectContaining({ citizen_id: "ci4" }));
  });

  it("volta à aba e passa o tempo: a lista não relê; os contadores usam COUNT_QUERY", async () => {
    vi.useFakeTimers({ shouldAdvanceTime: true });
    renderWithProviders(<RegistryStatus today={TODAY24} />);
    await screen.findByText("Joaquim Ribeiro");
    act(() => { focusManager.setFocused(false); focusManager.setFocused(true); onlineManager.setOnline(false); onlineManager.setOnline(true); });
    await act(async () => { await vi.advanceTimersByTimeAsync(3 * 60_000); });
    expect(api.listRegistryStatus).toHaveBeenCalledTimes(1);
    expect(mocked(api.countRegistryStatus).mock.calls.length).toBeGreaterThanOrEqual(3);
    expect(COUNT_QUERY.refetchInterval).toBe(60_000);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/RegistryStatus.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** `RegistryStatus` (`allow={(r) => r.professional}`) como descrito. O botão por linha: "Cadastrar <nome>" em `missing`, "Atualizar <nome>" nos demais.

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/RegistryStatus.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/RegistryStatus.tsx src/modules/RegistryStatus.test.tsx
$G commit -m "feat: list expired, expiring and missing registrations

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Dicionários do §11 e regras das fichas (`src/lib/territoryDictionaries.ts`) — BLOQUEADA até o §11

**Files:**
- Create: `src/lib/territoryDictionaries.ts`
- Test: `src/lib/territoryDictionaries.test.ts`

**Interfaces:**
- Consumes: o acréscimo de dicionários do contrato (publicado pela Task 0 do api); `isValidCpf`, `onlyDigits`; `isValidCns`.
- Produces:
  - `interface DictEntry { code: number; label: string }`; `DICTIONARIES: Record<DictKey, DictEntry[]>` com as chaves (nomes do IDL LEDI 8.7.0 em `snake_case`, contrato §1): `sexo`, `nacionalidade`, `justificativa_nao_possui_cpf`, `motivo_saida`, `tipo_de_imovel`, `renda_familiar`, `situacao_peso`, `tempo_situacao_rua`, `grau_instrucao_cidadao`, `situacao_mercado_trabalho_cidadao`, `relacao_parentesco_cidadao`, `orientacao_sexual_cidadao`, `identidade_genero_cidadao`, `deficiencias_cidadao`, `responsavel_por_crianca`, `doenca_cardiaca`, `doenca_respiratoria`, `doenca_rins`, `higiene_pessoal_situacao_rua`, `origem_alimento_situacao_rua`, `quantidade_alimentacoes_ao_dia_situacao_rua`, `localizacao`, `situacao_moradia_posse_terra`, `tipo_acesso_domicilio`, `tipo_domicilio`, `material_predominante_paredes_ext_domicilio`, `abastecimento_agua`, `agua_consumo_domicilio`, `forma_escoamento_banheiro`, `destino_lixo`, `tipo_origem_energia_eletrica`, `area_producao_rural` (a lista final é a do §11: chave a mais ou a menos lá, a mais ou a menos aqui);
  - `dictLabel(key: DictKey, code: number | null | undefined): string` (código desconhecido → o número cru; nulo → "—");
  - `SECTIONS`: a ordem e o tipo de cada campo das seções `sociodemographic`, `health_conditions`, `street_situation` e `living_conditions` — `{ key, label, kind: "code" | "codes" | "bool" | "text", dict?: DictKey, max?: number }[]` — transcritos do §11;
  - `interface Problem { path: string; message: string }`;
  - `householdProblem(input: HouseholdInput, families: FamilyInput[]): Problem | null`, `familyProblem(input: FamilyInput, today: string): Problem | null`, `registrationProblem(draft: RegistrationDraft, today: string): Problem | null`, `exitProblem(input: ExitInput, birthDate: string, today: string): Problem | null`, com `RegistrationDraft = { identity: PatientIdentity; hasValidatedIdentity: boolean; is_responsible: boolean; sociodemographic: CodedSection; health_conditions: CodedSection; street_situation: CodedSection | null; refused: boolean }`;
  - `allowsLivingConditions(propertyType: number): boolean`, `allowsFamilies(propertyType: number): boolean`, `canBePregnant(sex: number | undefined, birthDate: string | undefined, today: string): boolean`.

- [ ] **Step 1: Confira o §11 publicado (portão da task)**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/docs fetch -q origin
$G -C $R/docs show origin/main:superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md | sed -n '/^## .*[Dd]icion/,/^## [0-9]/p'
```

Expected: a tabela dos dicionários (chave, domínio, obrigatoriedade, condicionais). **Sem ela, pare** (as Tasks 10–13 esperam) e avise o coordenador via `SendMessage`: `dashboard · .worktrees/dashboard-mod24a/feat/mod-24a-territory-registry · Tasks 10–13 · aguardando dicionários do contrato (§11)`. Com ela: confira cada código usado nos testes do Step 2 contra a tabela; diferença → troque o valor no teste **e** anote para a Task 14 (o §11 vence).

- [ ] **Step 2: Write the failing test**

Os códigos abaixo são os do dicionário LEDI 8.7 (`referencias/dicionario.html`) e as regras, as de `estrutura_arquivos/dicionario-fci.html` e `dicionario-fcd.html`; valem se o §11 confirmar (Step 1).

```ts
// src/lib/territoryDictionaries.test.ts
import { describe, expect, it } from "vitest";
import {
  DICTIONARIES, allowsFamilies, allowsLivingConditions, canBePregnant, dictLabel, exitProblem, familyProblem, householdProblem, registrationProblem,
  type RegistrationDraft
} from "./territoryDictionaries";
import type { HouseholdInput } from "./api";

const T = "2026-10-23";

describe("dicionários", () => {
  it("códigos oficiais, rótulo e código desconhecido cru", () => {
    expect(DICTIONARIES.justificativa_nao_possui_cpf.map((e) => e.code)).toEqual([ 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13 ]);
    expect(dictLabel("justificativa_nao_possui_cpf", 3)).toBe("Recém-nascidos");
    expect(dictLabel("sexo", 0)).toBe("Masculino");
    expect(dictLabel("sexo", 1)).toBe("Feminino");
    expect(dictLabel("motivo_saida", 135)).toBe("Óbito");
    expect(dictLabel("tipo_de_imovel", 99)).toBe("Outros");
    expect(dictLabel("renda_familiar", 8)).toBe("Ausência de renda");
    expect(dictLabel("tempo_situacao_rua", 17)).toBe("Menos de 6 meses");
    expect(dictLabel("tipo_de_imovel", 42)).toBe("42");
    expect(dictLabel("tipo_de_imovel", null)).toBe("—");
  });
});

const ADDRESS = { zip: "81020000", street: "Rua das Flores", number: "120", complement: null, neighborhood_id: "nb1", reference: null };
const HH: HouseholdInput = { micro_area_id: "ma1", address: ADDRESS, property_type: 1,
  living_conditions: { localizacao: 83, situacao_moradia_posse_terra: 75, nu_comodos: "4", nu_moradores: "3" }, contact_phone: null, refused: false };

describe("domicílio (FCD)", () => {
  it("moradia só para 01 e 07–11; família só para 01", () => {
    expect([ 1, 7, 8, 9, 10, 11 ].every(allowsLivingConditions)).toBe(true);
    expect([ 2, 3, 4, 5, 6, 12, 13, 99 ].some(allowsLivingConditions)).toBe(false);
    expect(allowsFamilies(1)).toBe(true);
    expect(allowsFamilies(7)).toBe(false);
  });
  it.each([
    [ { ...HH, address: { ...ADDRESS, zip: "8102000" } }, "address.zip", "CEP com 8 dígitos" ],
    [ { ...HH, address: { ...ADDRESS, street: "" } }, "address.street", "informe o logradouro" ],
    [ { ...HH, address: { ...ADDRESS, street: "x".repeat(73) } }, "address.street", "logradouro com mais de 72 caracteres" ],
    [ { ...HH, address: { ...ADDRESS, number: "" } }, "address.number", "informe o número ou marque sem número" ],
    [ { ...HH, address: { ...ADDRESS, complement: "x".repeat(31) } }, "address.complement", "complemento com mais de 30 caracteres" ],
    [ { ...HH, address: { ...ADDRESS, reference: "x".repeat(41) } }, "address.reference", "ponto de referência com mais de 40 caracteres" ],
    [ { ...HH, contact_phone: "4199" }, "contact_phone", "telefone com DDD: 10 ou 11 dígitos" ],
    [ { ...HH, living_conditions: { situacao_moradia_posse_terra: 75 } }, "living_conditions.localizacao", "informe a localização (urbana, rural…)" ],
    [ { ...HH, living_conditions: { ...HH.living_conditions, nu_comodos: "0" } }, "living_conditions.nu_comodos", "número de cômodos não pode ser 0" ],
    [ { ...HH, property_type: 2 }, "living_conditions", "este tipo de imóvel não leva condições de moradia" ]
  ])("recusa %#", (input, path, message) => {
    expect(householdProblem(input as HouseholdInput, [])).toEqual({ path, message });
  });
  it("moradores ≥ soma dos membros das famílias; recusa mantém só o endereço", () => {
    const fam = { responsible_patient_id: "pa1", income_band: null, members_count: 4, living_since: null, moved_out: false };
    expect(householdProblem(HH, [ fam ])).toEqual({ path: "living_conditions.nu_moradores", message: "moradores menos que a soma das famílias (4)" });
    expect(householdProblem({ ...HH, property_type: 2, living_conditions: null }, [])).toBeNull();
    expect(householdProblem({ ...HH, refused: true, living_conditions: null }, [])).toBeNull();
    expect(householdProblem({ ...HH, refused: true }, [])).toEqual({ path: "living_conditions", message: "com recusa, o domicílio leva só o endereço" });
    expect(householdProblem(HH, [ { ...fam, members_count: 3 } ])).toBeNull();
  });
  it("família: membros de 1 a 999; reside desde não no futuro", () => {
    const fam = { responsible_patient_id: "pa1", income_band: 3, members_count: 3, living_since: "2019-05", moved_out: false };
    expect(familyProblem(fam, T)).toBeNull();
    expect(familyProblem({ ...fam, members_count: 0 }, T)).toEqual({ path: "members_count", message: "número de membros de 1 a 999" });
    expect(familyProblem({ ...fam, living_since: "2026-11" }, T)).toEqual({ path: "living_since", message: "reside desde depois de hoje" });
    expect(familyProblem({ ...fam, responsible_patient_id: "" }, T)).toEqual({ path: "responsible_patient_id", message: "escolha o responsável" });
  });
});

const BASE: RegistrationDraft = {
  identity: { full_name: "José das Dores", birth_date: "1990-01-10", sex: 0, mother_name: "Maria das Dores", cpf: "52998224725", race_color: "03", nationality: 1 },
  hasValidatedIdentity: false, is_responsible: true, sociodemographic: {}, health_conditions: {}, street_situation: { status_situacao_rua: false }, refused: false
};
const draft = (over: Partial<RegistrationDraft> & { identity?: Partial<RegistrationDraft["identity"]> } = {}): RegistrationDraft =>
  ({ ...BASE, ...over, identity: { ...BASE.identity, ...(over.identity ?? {}) } });

describe("cadastro individual (FCI)", () => {
  it("o caminho feliz não recusa", () => {
    expect(registrationProblem(draft(), T)).toBeNull();
  });
  it.each([
    [ draft({ identity: { cpf: undefined } }), "identity.no_cpf_reason", "sem CPF, informe o motivo" ],
    [ draft({ identity: { cpf: "52998224724" } }), "identity.cpf", "CPF inválido: confira os 11 dígitos" ],
    [ draft({ identity: { cpf: undefined, no_cpf_reason: 3, cns: "123" } }), "identity.cns", "CNS inválido: confira os 15 dígitos" ],
    [ draft({ identity: { race_color: undefined } }), "identity.race_color", "informe a raça/cor" ],
    [ draft({ identity: { race_color: "05", ethnicity: null } }), "identity.ethnicity", "raça/cor indígena pede a etnia (código de 4 dígitos)" ],
    [ draft({ identity: { mother_name: "Al" } }), "identity.mother_name", "nome da mãe de 3 a 70 letras, ou marque que não sabe" ],
    [ draft({ identity: { birth_date: "2026-10-24" } }), "identity.birth_date", "nascimento depois de hoje" ],
    [ draft({ health_conditions: { status_eh_gestante: true } }), "health_conditions.status_eh_gestante", "gestante só para sexo feminino entre 9 e 60 anos" ],
    [ draft({ identity: { sex: 1 }, health_conditions: { status_eh_gestante: false, maternidade_de_referencia: "Hospital X" } }),
      "health_conditions.maternidade_de_referencia", "maternidade de referência só para gestante" ],
    [ draft({ street_situation: null }), "street_situation.status_situacao_rua", "diga se está em situação de rua" ],
    [ draft({ street_situation: { status_situacao_rua: false, tempo_situacao_rua: 17 } }), "street_situation.tempo_situacao_rua",
      "tempo em situação de rua só para quem está" ],
    [ draft({ is_responsible: true, sociodemographic: { relacao_parentesco_cidadao: 137 } }), "sociodemographic.relacao_parentesco_cidadao",
      "parentesco só para quem não é o responsável" ]
  ])("recusa %#", (d, path, message) => {
    expect(registrationProblem(d, T)).toEqual({ path, message });
  });
  it("sem CPF com motivo: o CNS segue o §11 (Divergência T7); recusa ignora as seções", () => {
    expect(registrationProblem(draft({ identity: { cpf: undefined, no_cpf_reason: 3, cns: "898001160660003" } }), T)).toBeNull();
    expect(registrationProblem(draft({ refused: true, street_situation: null, health_conditions: { status_eh_gestante: true } }), T)).toBeNull();
  });
  it("identidade validada no posto não é conferida aqui (não é editável)", () => {
    expect(registrationProblem(draft({ hasValidatedIdentity: true, identity: { mother_name: "Al" } }), T)).toBeNull();
  });
  it("gestante: sexo feminino de 9 a 60 anos na data", () => {
    expect(canBePregnant(1, "2000-01-01", T)).toBe(true);
    expect(canBePregnant(1, "2018-01-01", T)).toBe(false);
    expect(canBePregnant(0, "2000-01-01", T)).toBe(false);
  });
  it("saída: óbito entre o nascimento e hoje", () => {
    expect(exitProblem({ reason: "death", on: "2026-10-20" }, "1940-01-01", T)).toBeNull();
    expect(exitProblem({ reason: "death", on: "2026-10-24" }, "1940-01-01", T)).toEqual({ path: "on", message: "data depois de hoje" });
    expect(exitProblem({ reason: "death", on: "1939-12-31" }, "1940-01-01", T)).toEqual({ path: "on", message: "data antes do nascimento" });
    expect(exitProblem({ reason: "moved", on: "" }, "1940-01-01", T)).toEqual({ path: "on", message: "informe a data" });
  });
});
```

(O código `137` de `relacao_parentesco_cidadao` e `75` de `situacao_moradia_posse_terra` são exemplos: use os do §11. A Divergência T7 decide se, sem CPF, o CNS é obrigatório — o LEDI exige `cnsCidadao` quando `stNaoPossuiCpf`, a spec diz "CNS quando houver"; acrescente o caso "sem CPF e sem CNS" com a decisão.)

- [ ] **Step 3: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/territoryDictionaries.test.ts`
Expected: FAIL — o módulo não existe.

- [ ] **Step 4: Implemente `src/lib/territoryDictionaries.ts`**

Transcreva o §11 (dicionários, rótulos em português do LEDI, ordem do LEDI) e `SECTIONS`. Cada função devolve o **primeiro** problema na ordem dos campos da tela (identificação → sociodemográfico → condições → situação de rua; endereço → moradia), com o `path` igual ao do api (`<seção>.<chave snake_case>`), para a tela focar o campo do `path` de uma recusa 422 também. Nada de cortar: limite estourado é problema.

- [ ] **Step 5: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/lib/territoryDictionaries.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd $W
$G add src/lib/territoryDictionaries.ts src/lib/territoryDictionaries.test.ts
$G commit -m "feat: mirror the ledi registration dictionaries and rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Ficha de domicílio e família

**Files:**
- Create: `src/modules/territory/HouseholdForm.tsx`, `src/modules/territory/FamilyForm.tsx`
- Modify: `src/modules/territory/HouseholdList.tsx` ("Novo domicílio"), `src/modules/territory/HouseholdDetail.tsx` ("Editar domicílio", "Nova família", "Editar família")
- Test: `src/modules/territory/HouseholdForm.test.tsx`

**Interfaces:**
- Consumes: `createHousehold`, `updateHousehold`, `addFamily`, `updateFamily`, `listPanelNeighborhoods` (existente, módulo 11), `listRegistrations` (Task 1); `territoryError`, `errorPath`, `isStale` (Task 2); `DICTIONARIES`, `SECTIONS`, `householdProblem`, `familyProblem`, `allowsLivingConditions`, `allowsFamilies` (Task 10).
- Produces: `HouseholdForm({ microAreaId, initial?: Household, onSaved(h: Household), onCancel() })`; `FamilyForm({ household: Household, initial?: Family, candidates: { patient_id: string; display_name: string | null }[], onSaved(h: Household), onCancel() })` — `candidates` são os moradores com cadastro (o responsável é paciente).

Comportamento: seções "Endereço" (CEP, logradouro, número + "sem número", complemento, bairro do 11 por `listPanelNeighborhoods`, ponto de referência, telefone de contato), "Tipo de imóvel", "Condições de moradia" (só quando `allowsLivingConditions`; campos de `SECTIONS.living_conditions`) e "Recusa" ("o morador recusou o cadastro do domicílio"). Trocar o tipo para um que não leva moradia **limpa** a seção (não a envia). Salvar: `householdProblem` → frase com o campo; depois `createHousehold`/`updateHousehold(id, input, initial.version)`. Recusa 422 `invalid_household` com `path` → frase e foco no campo do `path`. 409 `stale_version` → `STALE_TEXT`, formulário **fica**, botão "Recarregar" (invalida `HOUSEHOLD_KEY` e fecha). Família: responsável (`select` dos `candidates`), renda (`renda_familiar`), membros, reside desde (mês), "mudou-se"; mesma regra de `stale_version`. Nenhum estado sai do componente (nada em storage).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/territory/HouseholdForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), createHousehold: vi.fn(), updateHousehold: vi.fn(), addFamily: vi.fn(), updateFamily: vi.fn(),
    listPanelNeighborhoods: vi.fn() };
});

import * as api from "../../lib/api";
import { HouseholdForm } from "./HouseholdForm";
import { FamilyForm } from "./FamilyForm";
import { STALE_TEXT } from "../../lib/territoryRegistry";
import { renderWithProviders } from "../../test/campaignFixtures";
import { household, territoryUser } from "../../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const typeIn = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.createHousehold, api.updateHousehold, api.addFamily, api.updateFamily, api.listPanelNeighborhoods ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser());
  mocked(api.listPanelNeighborhoods).mockResolvedValue([ { id: "nb1", name: "Vila Verde", active: true } ]);
  mocked(api.createHousehold).mockResolvedValue(household());
  mocked(api.updateHousehold).mockResolvedValue(household({ version: 3 }));
  mocked(api.addFamily).mockResolvedValue(household({ version: 3 }));
});

function fillAddress() {
  typeIn("CEP", "81020-000");
  typeIn("Logradouro", "Rua das Flores");
  typeIn("Número", "120");
  fireEvent.change(screen.getByLabelText("Bairro"), { target: { value: "nb1" } });
}

describe("HouseholdForm", () => {
  it("cria um domicílio com moradia; o corpo tem só o que a ficha leva", async () => {
    const onSaved = vi.fn();
    renderWithProviders(<HouseholdForm microAreaId="ma1" onSaved={onSaved} onCancel={vi.fn()} />);
    fillAddress();
    fireEvent.change(await screen.findByLabelText("Tipo de imóvel"), { target: { value: "1" } });
    fireEvent.change(screen.getByLabelText("Localização"), { target: { value: "83" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar domicílio" }));
    await waitFor(() => expect(api.createHousehold).toHaveBeenCalledTimes(1));
    const body = mocked(api.createHousehold).mock.calls[0][0];
    expect(body.micro_area_id).toBe("ma1");
    expect(body.address).toEqual(expect.objectContaining({ zip: "81020000", street: "Rua das Flores", number: "120", neighborhood_id: "nb1" }));
    expect(body.property_type).toBe(1);
    expect(body.living_conditions.localizacao).toBe(83);
    expect(onSaved).toHaveBeenCalled();
  });

  it("comércio: a seção de moradia some e não vai no corpo", async () => {
    renderWithProviders(<HouseholdForm microAreaId="ma1" onSaved={vi.fn()} onCancel={vi.fn()} />);
    fillAddress();
    fireEvent.change(await screen.findByLabelText("Tipo de imóvel"), { target: { value: "1" } });
    fireEvent.change(screen.getByLabelText("Localização"), { target: { value: "83" } });
    fireEvent.change(screen.getByLabelText("Tipo de imóvel"), { target: { value: "2" } });
    expect(screen.queryByLabelText("Localização")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Salvar domicílio" }));
    await waitFor(() => expect(mocked(api.createHousehold).mock.calls[0][0].living_conditions).toBeNull());
  });

  it("recusa do domicílio: sem moradia nem família; endereço fica", async () => {
    renderWithProviders(<HouseholdForm microAreaId="ma1" onSaved={vi.fn()} onCancel={vi.fn()} />);
    fillAddress();
    fireEvent.change(await screen.findByLabelText("Tipo de imóvel"), { target: { value: "1" } });
    fireEvent.click(screen.getByLabelText("O morador recusou o cadastro do domicílio"));
    expect(screen.queryByLabelText("Localização")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Salvar domicílio" }));
    await waitFor(() => expect(api.createHousehold).toHaveBeenCalled());
    const body = mocked(api.createHousehold).mock.calls[0][0];
    expect(body.refused).toBe(true);
    expect(body.living_conditions).toBeNull();
    expect(body.address.street).toBe("Rua das Flores");
  });

  it("CEP ruim é recusado antes de enviar, com o campo", async () => {
    renderWithProviders(<HouseholdForm microAreaId="ma1" onSaved={vi.fn()} onCancel={vi.fn()} />);
    fillAddress();
    typeIn("CEP", "8102");
    fireEvent.click(screen.getByRole("button", { name: "Salvar domicílio" }));
    expect((await screen.findByRole("alert")).textContent).toBe("CEP com 8 dígitos");
    expect(api.createHousehold).not.toHaveBeenCalled();
  });

  it("409 stale_version: frase, o formulário fica, recarregar relê", async () => {
    mocked(api.updateHousehold).mockRejectedValue(new api.ApiError(409, { error: "stale_version" }, "409"));
    const onCancel = vi.fn();
    renderWithProviders(<HouseholdForm microAreaId="ma1" initial={household()} onSaved={vi.fn()} onCancel={onCancel} />);
    typeIn("Ponto de referência", "em frente à praça");
    fireEvent.click(screen.getByRole("button", { name: "Salvar domicílio" }));
    expect(await screen.findByText(STALE_TEXT)).not.toBeNull();
    expect(mocked(api.updateHousehold).mock.calls[0][2]).toBe(2);
    expect((screen.getByLabelText("Ponto de referência") as HTMLInputElement).value).toBe("em frente à praça");
    fireEvent.click(screen.getByRole("button", { name: "Recarregar" }));
    expect(onCancel).toHaveBeenCalled();
    expect(api.updateHousehold).toHaveBeenCalledTimes(1);
  });

  it("422 invalid_household com path mostra a frase e marca o campo", async () => {
    mocked(api.createHousehold).mockRejectedValue(new api.ApiError(422, { error: "invalid_household", path: "address.number" }, "422"));
    renderWithProviders(<HouseholdForm microAreaId="ma1" onSaved={vi.fn()} onCancel={vi.fn()} />);
    fillAddress();
    fireEvent.change(await screen.findByLabelText("Tipo de imóvel"), { target: { value: "2" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar domicílio" }));
    expect(await screen.findByText("confira o campo destacado")).not.toBeNull();
    expect(screen.getByLabelText("Número").getAttribute("aria-invalid")).toBe("true");
  });
});

describe("FamilyForm", () => {
  it("nova família com a versão do domicílio; responsável é morador com cadastro", async () => {
    renderWithProviders(<FamilyForm household={household()} candidates={[ { patient_id: "pa1", display_name: "Maria das Dores" } ]} onSaved={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.change(await screen.findByLabelText("Responsável"), { target: { value: "pa1" } });
    typeIn("Número de membros", "4");
    fireEvent.change(screen.getByLabelText("Renda familiar"), { target: { value: "3" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar família" }));
    await waitFor(() => expect(api.addFamily).toHaveBeenCalledWith("hh1",
      { responsible_patient_id: "pa1", income_band: 3, members_count: 4, living_since: null, moved_out: false }, 2));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/territory/HouseholdForm.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** `HouseholdForm` e `FamilyForm` como descrito; ligue "Novo domicílio" no `HouseholdList` (só com `canRegister`) e "Editar domicílio", "Nova família" (só se `allowsFamilies`) e "Editar <responsável>" no `HouseholdDetail`. Depois de salvar, invalide `HOUSEHOLDS_KEY` e `HOUSEHOLD_KEY` (uma releitura — ação do usuário, não polling) e mostre "Domicílio salvo. Nova versão enviada ao e-SUS." / "Família salva. Nova versão do domicílio enviada ao e-SUS.".

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/territory src/modules/TerritoryTeam.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/territory/HouseholdForm.tsx src/modules/territory/FamilyForm.tsx src/modules/territory/HouseholdForm.test.tsx src/modules/territory/HouseholdList.tsx src/modules/territory/HouseholdDetail.tsx
$G commit -m "feat: add the household and family registration forms

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Ficha de cadastro individual em seções — pessoa, duplicados, recusa e `cpf_taken`

**Files:**
- Create: `src/modules/territory/PersonStart.tsx`, `src/modules/territory/RegistrationForm.tsx`
- Modify: `src/modules/territory/HouseholdDetail.tsx` ("Adicionar morador", "Abrir cadastro"), `src/modules/RegistryStatus.tsx` (o `onRegister` abre a ficha na própria tela)
- Test: `src/modules/territory/PersonStart.test.tsx`, `src/modules/territory/RegistrationForm.test.tsx`

**Interfaces:**
- Consumes: `lookupTerritoryPerson`, `findDuplicates`, `createRegistration`, `updateRegistration`, `getRegistration`, `listRegistrations` (Task 1); `duplicateQueryProblem`, `cpfProblem`, `territoryError`, `errorPath`, `isStale`, `STALE_TEXT`, `fmtDay` (Task 2); `DICTIONARIES`, `SECTIONS`, `registrationProblem`, `canBePregnant` (Task 10).
- Produces:
  - `type PersonChoice = { kind: "patient"; patientId: string; registrationId: string | null } | { kind: "citizen"; citizenId: string } | { kind: "new"; identity: PatientIdentity }`;
  - `PersonStart({ today, onChoose(choice: PersonChoice), onCancel() })` — passo 1: "Tem CPF?" → CPF → `lookupTerritoryPerson` (paciente com cadastro → `registrationId` para atualizar; par sem paciente → `citizen`; ninguém → `new` com o CPF); sem CPF → nome completo, nascimento, nome da mãe → **Buscar possíveis duplicados** (`findDuplicates`) → candidatos ("é esta pessoa" → `patient`) ou "é outra pessoa — criar sem CPF" (só depois da busca) → `new` com os dados digitados;
  - `RegistrationForm({ today, household?: { id: string; familyId: string | null }, choice?: PersonChoice, registrationId?: string, onSaved(r: Registration), onCancel() })` — seções colapsáveis "Identificação", "Sociodemográfico", "Condições de saúde", "Situação de rua", "Recusa"; editando (`registrationId`), lê `getRegistration` (`NOMINAL_QUERY`, uma vez) e manda `updateRegistration(id, input, version)`.

Comportamento:
- Identificação: raça/cor pelo `RACE_COLORS` e etnia como no `DemographicsFields` do 22 (uma fonte por dado: o valor grava no paciente); paciente com identidade validada no posto (`identity_validated: true`) mostra nome, nascimento, sexo e documento só leitura ("validados no posto"); sem validação, os campos são editáveis e vão em `identity` (Divergência T5). Paciente sem CPF mostra "Informar CPF" (vai em `identity.cpf`); 409 `cpf_taken` → frase, campo marcado, formulário fica. `identity_changed_at` mostra o aviso "Os dados de identificação foram atualizados pela validação no posto em <dd/mm/aaaa>; confira e salve para gerar a nova versão." (decisão 13).
- Seções pelas `SECTIONS` do §11; gestante e maternidade só aparecem com `canBePregnant`; "situação de rua" com o sim/não obrigatório e o resto só com "sim".
- Recusa: marcar esconde as três seções; o corpo vai com `refused: true`, `sociodemographic: {}`, `health_conditions: {}`, `street_situation: null`.
- Salvar: `registrationProblem` (frase e campo) → `createRegistration`/`updateRegistration`; recusas `already_registered` (com `registration_id` no corpo? — Divergência T5; sem ele, a frase e "Abrir cadastro" some), `cpf_or_reason_required`, `invalid_registration` + `path`, `cbo_not_allowed`, `out_of_scope`, `stale_version` como na Task 11. Aviso: "Cadastro salvo. Nova versão enviada ao e-SUS.".
- **Nada** de `localStorage`/`sessionStorage`; nada no `console`.

- [ ] **Step 1: Write the failing tests**

```tsx
// src/modules/territory/PersonStart.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), lookupTerritoryPerson: vi.fn(), findDuplicates: vi.fn() };
});

import * as api from "../../lib/api";
import { PersonStart } from "./PersonStart";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY24, territoryUser } from "../../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const typeIn = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
afterEach(cleanup);
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.lookupTerritoryPerson, api.findDuplicates ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser());
});

function fillNoCpf() {
  fireEvent.click(screen.getByLabelText("Não tem CPF"));
  typeIn("Nome completo", "José das Dores");
  typeIn("Data de nascimento", "2026-09-30");
  typeIn("Nome da mãe", "Maria das Dores");
}

describe("PersonStart", () => {
  it("sem CPF: só cria depois da busca; escolher o candidato manda patient_id", async () => {
    mocked(api.findDuplicates).mockResolvedValue([ { patient_id: "pa7", display_name: "José das Dores", birth_date: "2026-09-30", has_cpf: false } ]);
    const onChoose = vi.fn();
    renderWithProviders(<PersonStart today={TODAY24} onChoose={onChoose} onCancel={vi.fn()} />);
    fillNoCpf();
    expect(screen.queryByRole("button", { name: /criar sem CPF/ })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Buscar possíveis duplicados" }));
    await waitFor(() => expect(api.findDuplicates).toHaveBeenCalledWith({ full_name: "José das Dores", birth_date: "2026-09-30", mother_name: "Maria das Dores" }));
    expect(await screen.findByText("José das Dores · nascido em 30/09/2026 · sem CPF")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "É esta pessoa" }));
    expect(onChoose).toHaveBeenCalledWith({ kind: "patient", patientId: "pa7", registrationId: null });
  });

  it("sem candidato: 'é outra pessoa — criar sem CPF' leva os dados digitados; trocar o nome pede nova busca", async () => {
    mocked(api.findDuplicates).mockResolvedValue([]);
    const onChoose = vi.fn();
    renderWithProviders(<PersonStart today={TODAY24} onChoose={onChoose} onCancel={vi.fn()} />);
    fillNoCpf();
    fireEvent.click(screen.getByRole("button", { name: "Buscar possíveis duplicados" }));
    expect(await screen.findByText("Nenhum paciente parecido.")).not.toBeNull();
    typeIn("Nome completo", "José Dores");
    expect(screen.queryByRole("button", { name: "Criar pessoa sem CPF" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Buscar possíveis duplicados" }));
    fireEvent.click(await screen.findByRole("button", { name: "Criar pessoa sem CPF" }));
    expect(onChoose).toHaveBeenCalledWith({ kind: "new", identity: { full_name: "José Dores", birth_date: "2026-09-30", mother_name: "Maria das Dores" } });
  });

  it("com CPF: paciente com cadastro abre para atualizar; par validado vira citizen", async () => {
    mocked(api.lookupTerritoryPerson).mockResolvedValueOnce({ patient_id: "pa1", citizen_id: null, display_name: "Maria das Dores", cpf_masked: "***.982.247-**", registration_id: "rg1" })
      .mockResolvedValueOnce({ patient_id: null, citizen_id: "ci4", display_name: "Joana", cpf_masked: "***.982.247-**", registration_id: null });
    const onChoose = vi.fn();
    renderWithProviders(<PersonStart today={TODAY24} onChoose={onChoose} onCancel={vi.fn()} />);
    typeIn("CPF", "529.982.247-25");
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Atualizar o cadastro de Maria das Dores" }));
    expect(onChoose).toHaveBeenLastCalledWith({ kind: "patient", patientId: "pa1", registrationId: "rg1" });
    fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar Joana" }));
    expect(onChoose).toHaveBeenLastCalledWith({ kind: "citizen", citizenId: "ci4" });
  });
});
```

```tsx
// src/modules/territory/RegistrationForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), createRegistration: vi.fn(), updateRegistration: vi.fn(), getRegistration: vi.fn() };
});

import * as api from "../../lib/api";
import { RegistrationForm } from "./RegistrationForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY24, registration, territoryUser } from "../../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const typeIn = (label: string, value: string) => fireEvent.change(screen.getByLabelText(label), { target: { value } });
afterEach(() => { cleanup(); vi.restoreAllMocks(); });
beforeEach(() => {
  for (const fn of [ api.fetchCurrentSession, api.createRegistration, api.updateRegistration, api.getRegistration ]) mocked(fn).mockReset();
  mocked(api.fetchCurrentSession).mockResolvedValue(territoryUser());
  mocked(api.createRegistration).mockResolvedValue(registration());
  mocked(api.updateRegistration).mockResolvedValue(registration({ version: 2 }));
  mocked(api.getRegistration).mockResolvedValue(registration({ patient: { id: "pa2", display_name: "José das Dores", birth_date: "2026-09-30", sex: 0,
    has_cpf: false, no_cpf_reason: 3, cns_masked: null, identity_validated: false }, street_situation: { status_situacao_rua: false } }));
});

const NEW = { kind: "new" as const, identity: { full_name: "José das Dores", birth_date: "2026-09-30", mother_name: "Maria das Dores" } };

function fillNewbornIdentity() {
  fireEvent.change(screen.getByLabelText("Sexo"), { target: { value: "0" } });
  fireEvent.change(screen.getByLabelText("Raça/cor"), { target: { value: "03" } });
  fireEvent.change(screen.getByLabelText("Motivo de não ter CPF"), { target: { value: "3" } });
  typeIn("CNS", "898001160660003");
  fireEvent.click(screen.getByLabelText("Não está em situação de rua"));
}

describe("RegistrationForm", () => {
  it("recém-nascido sem CPF: cria o paciente com o motivo; sem gestante na tela", async () => {
    renderWithProviders(<RegistrationForm today={TODAY24} household={{ id: "hh1", familyId: "fa1" }} choice={NEW} onSaved={vi.fn()} onCancel={vi.fn()} />);
    fillNewbornIdentity();
    expect(screen.queryByLabelText("Está gestante")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Salvar cadastro" }));
    await waitFor(() => expect(api.createRegistration).toHaveBeenCalledTimes(1));
    const body = mocked(api.createRegistration).mock.calls[0][0];
    expect(body.new_patient).toEqual(expect.objectContaining({ full_name: "José das Dores", birth_date: "2026-09-30", sex: 0, no_cpf_reason: 3,
      cns: "898001160660003", race_color: "03", mother_name: "Maria das Dores" }));
    expect(body.new_patient.cpf).toBeUndefined();
    expect(body).toEqual(expect.objectContaining({ household_id: "hh1", family_id: "fa1", refused: false }));
    expect(body.street_situation).toEqual({ status_situacao_rua: false });
  });

  it("recusa: só identificação e refused; as seções somem", async () => {
    renderWithProviders(<RegistrationForm today={TODAY24} choice={{ kind: "patient", patientId: "pa1", registrationId: null }} onSaved={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.click(await screen.findByLabelText("O cidadão recusou o cadastro (termo de recusa)"));
    expect(screen.queryByRole("group", { name: "Condições de saúde" })).toBeNull();
    expect(screen.queryByRole("group", { name: "Situação de rua" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Salvar cadastro" }));
    await waitFor(() => expect(api.createRegistration).toHaveBeenCalled());
    const body = mocked(api.createRegistration).mock.calls[0][0];
    expect(body).toEqual(expect.objectContaining({ patient_id: "pa1", refused: true, sociodemographic: {}, health_conditions: {}, street_situation: null }));
  });

  it("paciente sem CPF: informar CPF já usado → cpf_taken, o formulário fica", async () => {
    mocked(api.updateRegistration).mockRejectedValue(new api.ApiError(409, { error: "cpf_taken" }, "409"));
    renderWithProviders(<RegistrationForm today={TODAY24} registrationId="rg1" onSaved={vi.fn()} onCancel={vi.fn()} />);
    await screen.findByLabelText("Informar CPF");
    typeIn("Informar CPF", "529.982.247-25");
    fireEvent.change(screen.getByLabelText("Raça/cor"), { target: { value: "03" } });
    fireEvent.click(screen.getByRole("button", { name: "Salvar cadastro" }));
    expect(await screen.findByText("este CPF já pertence a outro paciente — confira o número; a junção de pacientes ainda não existe")).not.toBeNull();
    expect(mocked(api.updateRegistration).mock.calls[0][1].identity.cpf).toBe("52998224725");
    expect(mocked(api.updateRegistration).mock.calls[0][2]).toBe(1);
    expect((screen.getByLabelText("Informar CPF") as HTMLInputElement).value).toBe("529.982.247-25");
  });

  it("identidade validada no posto é só leitura; aviso de mudança pela validação", async () => {
    mocked(api.getRegistration).mockResolvedValue(registration({ identity_changed_at: "2026-10-20T09:00:00-03:00" }));
    renderWithProviders(<RegistrationForm today={TODAY24} registrationId="rg1" onSaved={vi.fn()} onCancel={vi.fn()} />);
    expect(await screen.findByText("Os dados de identificação foram atualizados pela validação no posto em 20/10/2026; confira e salve para gerar a nova versão.")).not.toBeNull();
    expect(screen.queryByLabelText("Nome completo")).toBeNull();
    expect(screen.getByText("validados no posto")).not.toBeNull();
  });

  it("nada no storage nem na URL: CPF, nome, condições só no corpo", async () => {
    const setItem = vi.spyOn(Storage.prototype, "setItem");
    const log = vi.spyOn(console, "log");
    renderWithProviders(<RegistrationForm today={TODAY24} household={{ id: "hh1", familyId: null }} choice={NEW} onSaved={vi.fn()} onCancel={vi.fn()} />);
    fillNewbornIdentity();
    fireEvent.click(screen.getByLabelText("Tem diabetes"));
    fireEvent.click(screen.getByRole("button", { name: "Salvar cadastro" }));
    await waitFor(() => expect(api.createRegistration).toHaveBeenCalled());
    expect(setItem).not.toHaveBeenCalled();
    expect(log).not.toHaveBeenCalled();
    expect(window.location.search + window.location.hash).toBe("");
  });

  it("gestante fora de 9–60 anos ou sexo masculino é recusada antes de enviar (espelho do api)", async () => {
    renderWithProviders(<RegistrationForm today={TODAY24} choice={{ kind: "new", identity: { full_name: "Ana Souza", birth_date: "1990-01-10", mother_name: "Rita Souza" } }}
      onSaved={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.change(screen.getByLabelText("Sexo"), { target: { value: "1" } });
    expect(screen.getByLabelText("Está gestante")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Sexo"), { target: { value: "0" } });
    expect(screen.queryByLabelText("Está gestante")).toBeNull();
  });
});
```

(Os rótulos "Tem diabetes", "Está gestante", "Não está em situação de rua" são os de `SECTIONS` da Task 10 para `status_tem_diabetes`, `status_eh_gestante` e `status_situacao_rua: false`; se o §11 der outro rótulo, use o de lá aqui e na tela.)

- [ ] **Step 2: Run tests to verify they fail**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/territory/PersonStart.test.tsx src/modules/territory/RegistrationForm.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** `PersonStart` e `RegistrationForm` como descrito. A busca de duplicados é **invalidada** quando nome, nascimento ou mãe mudam (estado `searchedFor` com os três valores; o botão "Criar pessoa sem CPF" só aparece se `searchedFor` bate com o digitado e a busca voltou). Ligue: no `HouseholdDetail`, "Adicionar morador" (`PersonStart` → `RegistrationForm` com `household`), "Abrir cadastro" por morador (`registrationId`); no `RegistryStatus`, `onRegister` abre `PersonStart`/`RegistrationForm` no próprio painel (linha com `registration_id` → `registrationId`; com `patient_id` sem cadastro → `choice: patient`; com `citizen_id` → `choice: citizen`).

- [ ] **Step 4: Run tests to verify they pass**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/territory src/modules/RegistryStatus.test.tsx src/modules/TerritoryTeam.test.tsx src/theme && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/territory/PersonStart.tsx src/modules/territory/RegistrationForm.tsx src/modules/territory/PersonStart.test.tsx src/modules/territory/RegistrationForm.test.tsx src/modules/territory/HouseholdDetail.tsx src/modules/RegistryStatus.tsx
$G commit -m "feat: add the individual registration form with duplicate search

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Saída do cadastro — mudança de território ou óbito

**Files:**
- Create: `src/modules/territory/ExitForm.tsx`
- Modify: `src/modules/territory/RegistrationForm.tsx` (botão "Registrar saída" no cadastro aberto, ativo e sem recusa)
- Test: `src/modules/territory/ExitForm.test.tsx`

**Interfaces:**
- Consumes: `exitRegistration` (Task 1); `EXIT_REASONS`, `territoryError` (Task 2); `exitProblem` (Task 10).
- Produces: `ExitForm({ registration: Registration, today: string, onDone(r: Registration), onCancel() })`.

Comportamento: motivo (rádio "mudou de território" / "óbito"), data (padrão vazio; obrigatória). Antes do Confirmar, a frase do efeito: mudança → "A pessoa sai da equipe pelo cadastro; consulta recente e bairro continuam valendo."; óbito → "A pessoa fica marcada como falecida nas linhas de cuidado e as inscrições são encerradas por óbito." `exitProblem` antes de enviar; recusa do api pela frase. Cadastro recusado (`refused`) não mostra "Registrar saída" (LEDI: motivo de saída vazio com termo de recusa).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/territory/ExitForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), exitRegistration: vi.fn() };
});

import * as api from "../../lib/api";
import { ExitForm } from "./ExitForm";
import { renderWithProviders } from "../../test/campaignFixtures";
import { TODAY24, registration, territoryUser } from "../../test/territoryFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);
beforeEach(() => {
  mocked(api.fetchCurrentSession).mockReset().mockResolvedValue(territoryUser());
  mocked(api.exitRegistration).mockReset().mockResolvedValue(registration({ status: "exited", exit: { reason: "death", on: "2026-10-20" } }));
});

describe("ExitForm", () => {
  it("óbito: diz o efeito, recusa data no futuro e envia motivo e data no corpo", async () => {
    const onDone = vi.fn();
    renderWithProviders(<ExitForm registration={registration()} today={TODAY24} onDone={onDone} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByLabelText("óbito"));
    expect(screen.getByText("A pessoa fica marcada como falecida nas linhas de cuidado e as inscrições são encerradas por óbito.")).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-10-24" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar saída" }));
    expect(screen.getByRole("alert").textContent).toBe("data depois de hoje");
    fireEvent.change(screen.getByLabelText("Data"), { target: { value: "2026-10-20" } });
    fireEvent.click(screen.getByRole("button", { name: "Registrar saída" }));
    await waitFor(() => expect(api.exitRegistration).toHaveBeenCalledWith("rg1", { reason: "death", on: "2026-10-20" }));
    expect(onDone).toHaveBeenCalled();
  });

  it("mudança: a frase da equipe", () => {
    renderWithProviders(<ExitForm registration={registration()} today={TODAY24} onDone={vi.fn()} onCancel={vi.fn()} />);
    fireEvent.click(screen.getByLabelText("mudou de território"));
    expect(screen.getByText("A pessoa sai da equipe pelo cadastro; consulta recente e bairro continuam valendo.")).not.toBeNull();
  });
});
```

E em `RegistrationForm.test.tsx`: "cadastro recusado não oferece Registrar saída" (`getRegistration` com `refused: true` → `queryByRole("button", { name: "Registrar saída" })` nulo).

- [ ] **Step 2: Run test to verify it fails**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/territory/ExitForm.test.tsx src/modules/territory/RegistrationForm.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente** `ExitForm` e o botão no `RegistrationForm` (com o cadastro lido, `status` diferente de `exited` e `refused: false`).

- [ ] **Step 4: Run test to verify it passes**

Run: `bin/test dashboard -w dashboard-mod24a src/modules/territory && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W
$G add src/modules/territory/ExitForm.tsx src/modules/territory/ExitForm.test.tsx src/modules/territory/RegistrationForm.tsx src/modules/territory/RegistrationForm.test.tsx
$G commit -m "feat: register a person's exit from the territory registry

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Suíte, build, revisão e prova no navegador (api real na 3046)

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–13 na branch `feat/mod-24a-territory-registry`; o api do 24a na **3046** com a semente do 24a.
- Produces: branch pronta para merge (depois do merge do api do 24a); evidências em `evidencias/2026-10-23-modulo-24a/dashboard/`.

- [ ] **Step 1: Suíte, tipos e build**

Run: `bin/test dashboard -w dashboard-mod24a && cd $W && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base da Task 0 mais **13 arquivos de teste novos** (`api.territory`, `territoryRegistry`, `modules.territory`, `TerritoryMicroAreas`, `ProfileForm`, `patientDocument`, `TerritoryTeam`, `RegistryStatus`, `territoryDictionaries`, `HouseholdForm`, `PersonStart`, `RegistrationForm`, `ExitForm`) e os existentes com casos novos ou ajustados (`features.test.ts`, `modules.test.ts`, `demographics.test.ts`, `DemographicsForm.test.tsx`, `ClinicalRecord.test.tsx`, `BillingProcedures.test.tsx`, `Attendance.test.tsx`, `BillingPeriods.demographics.test.tsx`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd $W
$G status --short
$G diff origin/main --stat -- src/components src/shell/Shell.tsx src/shell/Cards.tsx src/theme src/responsive.css vite.config.ts package.json package-lock.json
```

Expected: `status` vazio; nenhum componente comum, peça do shell, tema, `responsive.css`, proxy ou dependência mudados.

- [ ] **Step 2: Nada vaza, nada fica no navegador, nada que grava trilha relê sozinho**

```bash
cd $W
grep -rn "console\.\|localStorage\|sessionStorage\|indexedDB\|useAutosave" src/lib/territoryRegistry.ts src/lib/territoryDictionaries.ts src/lib/patientDocument.ts src/modules/territory src/modules/Territory*.tsx src/modules/RegistryStatus.tsx | grep -v "\.test\."
grep -rn "refetchInterval\|refetchOnWindowFocus: true" src/modules/territory src/modules/Territory*.tsx src/modules/RegistryStatus.tsx | grep -v "\.test\."
grep -rn "COUNT_QUERY" src/modules/territory src/modules/Territory*.tsx src/modules/RegistryStatus.tsx | grep -v "\.test\."
sed -n '/─── Cadastro territorial (módulo 24a/,$p' src/lib/api.ts | grep -on 'p.set("[a-z_]*"\|new URLSearchParams({ [A-Za-z_, :]* })'
grep -rn "maxLength" src/modules/territory src/modules/Territory*.tsx src/modules/RegistryStatus.tsx
```

Expected: o primeiro, o segundo e o último sem saída; o terceiro só na consulta de contagem de `RegistryStatus.tsx`; o quarto só com `team_id`, `micro_area_id`, `status`, `before`, `limit`, `patient_id`, `household_id` — nenhum `cpf`, `cns`, `name`, `birth`, `mother`.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§1–§7), o ADR 0042, o contrato (§1–§7, §10, §12) com o acréscimo dos dicionários e as Divergências deste plano. Pontos de atenção:
- sem `territory_registry`, o menu, o perfil profissional, a recepção e o faturamento são os de hoje (testes antigos verdes; só as contagens de `modules.test.ts` e os rótulos "Documento" mudaram);
- "Criar pessoa sem CPF" só depois da busca de duplicados, refeita quando nome, nascimento ou mãe mudam; candidato vira `patient_id`;
- recusa sem as três seções no corpo; domicílio recusado só com endereço; moradia e família só para os tipos de imóvel do LEDI;
- `stale_version` e `cpf_taken` mantêm o formulário; nenhuma reescrita automática;
- validação espelha o §11 (Task 10) campo a campo, com o mesmo `path` do api; nenhum código cru nas telas (rótulo do dicionário; desconhecido cru);
- listas, detalhe, moradores, cadastro e validade com `NOMINAL_QUERY`; só a contagem com `COUNT_QUERY`;
- nada de dado sensível em URL, storage, console ou erro;
- microáreas pelo `SensitiveAction` com step-up; ACS sem CNS não escolhível;
- endereço do paciente com cadastro ativo só leitura no check-in e na pendência do faturamento; "sem CPF" e o CNS onde o paciente aparece;
- interface sem redesign: só componentes existentes; cores por papéis; cada tela nova no `CATALOG`.

Diferenças entre o §11 e os valores deste plano (Task 10) entram aqui como nota no relatório da task.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 24a na **3046** (plano do api: servidor do worktree `apps/api/.claude/mod24a`, migrações de cidade a partir de `20261023900001` aplicadas no banco de dev **com autorização do usuário**, e a semente do 24a: Curitiba em modo `record`, a ESF Vila Verde (tipo 70, INE) com a ACS Ana Prado (CBO 515105, sem conselho, com CNS), uma enfermeira da equipe, um domicílio com família, um paciente com cadastro de mais de 24 meses, um de 23 meses, um cidadão validado sem paciente e um paciente sem CPF). Ligue em Curitiba, pelo maintenance, `clinical_record` e `territory_registry`. O Vite do worktree sobe num container da rede do compose (o `node_modules` do container é um volume anônimo):

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name dashboard-mod24a --network rota-saude_default -p 5200:5200 \
  -v $R/.worktrees/dashboard-mod24a:/app -v /app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3046 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5200 --host 0.0.0.0"
docker logs -f dashboard-mod24a   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 24a num container próprio, use o nome dele no lugar de `api`.)

Abra `http://curitiba.localhost:5200/dashboard/`. O usuário faz o login de cada papel; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Roteiro, com screenshot de cada passo em `evidencias/2026-10-23-modulo-24a/dashboard/`:
1. **Admin** — Cadastro territorial → **Microáreas**: equipe ESF Vila Verde; criar a microárea "02 · Vila Verde de baixo" com a ACS Ana Prado (step-up); código "00" recusado antes de confirmar; o ACS sem CNS aparece desabilitado; trocar o ACS e desativar uma microárea (o efeito é dito antes).
2. **Admin** — Equipe → **Profissionais**: o perfil da ACS mostra "sem conselho (ACS ou agente de endemias)"; criar um perfil novo "Sem conselho" exige o CNS.
3. **ACS** — **Território da equipe**: microárea 01 → o domicílio da semente → famílias ("Responsável: … · N pessoas") e moradores com a situação ("válido", "vencido (mais de 24 meses)") e "sem CPF · CNS …" no paciente sem CPF; nenhuma condição de saúde na lista.
4. **Novo domicílio** — endereço, tipo "Domicílio", localização urbana → salvar ("Domicílio salvo. Nova versão enviada ao e-SUS."); trocar o tipo para "Comércio" some a moradia; CEP com 7 dígitos é recusado com o campo; **nova família** com responsável e renda.
5. **Concorrência** — abrir o mesmo domicílio em duas abas; salvar na primeira e depois na segunda → "outra pessoa salvou…", o texto continua; "Recarregar" traz a versão nova.
6. **Recém-nascido sem CPF** — "Adicionar morador" → "Não tem CPF" → nome, nascimento, mãe → **Buscar possíveis duplicados** (nenhum) → "Criar pessoa sem CPF" → ficha em seções: motivo "Recém-nascidos", CNS, sexo, raça/cor, "não está em situação de rua" → salvar; o morador aparece "válido" e "sem CPF · CNS …".
7. **Duplicado** — repetir o passo 6 com o mesmo nome e nascimento: o candidato aparece; "É esta pessoa" abre o cadastro existente (nada de paciente novo; o api recusa `already_registered` se insistir).
8. **CPF de quem não tinha** — abrir o cadastro do paciente sem CPF e "Informar CPF" com o CPF de outro paciente da semente → "este CPF já pertence a outro paciente…", o formulário fica.
9. **Recusa** — num morador, marcar a recusa: as seções somem; salvar; o morador aparece "recusou o cadastro".
10. **Saída** — num cadastro, "Registrar saída" → "óbito" com a data de ontem; a frase do efeito antes; depois, nas Linhas de cuidado (se ligadas), a pessoa aparece como falecida (prova do api, passo de equipe e óbito).
11. **Validade dos cadastros** — os quatro números; "Vencidos" lista o paciente de mais de 24 meses; "A vencer" o de 23 meses; "Sem cadastro" lista o cidadão validado sem paciente → "Cadastrar" abre a ficha e, salvo, cria o paciente (o api, por `Patients::Resolve`).
12. **Sem polling** — com a aba de rede aberta na Validade por 3 minutos: nenhuma chamada a `/territory/registry_status?` sem ação; só `/territory/registry_status/count` a cada minuto. Nenhuma URL de `/territory` com CPF, CNS ou nome.
13. **Sem rascunho** — com a ficha individual preenchida pela metade, recarregar a página: a ficha volta vazia (nada guardado); em DevTools → Application, `localStorage` e `sessionStorage` sem nada do cadastro.
14. **Faturamento e recepção** — como faturista, a pendência de raça/cor de um paciente com cadastro ativo mostra "Endereço: do cadastro domiciliar…" sem os campos de endereço; no check-in de um cidadão com cadastro ativo, idem.
15. **Telefone** (largura 375): microáreas, domicílios e validade viram cartões, sem rolagem lateral; o formulário em seções cabe na tela.
16. No maintenance, desligar `territory_registry` e recarregar: o grupo "Cadastro territorial" some e o faturamento volta a mostrar os campos de endereço; religar devolve.

Depois: `docker rm -f dashboard-mod24a`; religue o que a prova desligou.

- [ ] **Step 5: Cards e parada**

F-24.2, F-24.3, F-24.4, F-24.5 e F-24.6 vão a **Done** se a parte do api já está Done (o dashboard é o último app deles); F-24.1 espera o maintenance; F-24.7 e F-24.8 são só do api. Os que passaram na prova do Step 4 com a parte do api Done vão a **Verified**. Use o mesmo `gh project item-edit` da Task 0 com o id da opção Done (`98236657`) e o de Verified (da listagem da Task 0). **Pare.** O merge do dashboard só vem depois do merge do api do 24a, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-24a-territory-registry
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod24a
/opt/homebrew/bin/git branch -d feat/mod-24a-territory-registry
```

---

## Divergências propostas ao contrato

Numeração própria deste plano (T1–T15). **Abertas** = o contrato não tem e a tela precisa; quem decide é o plano do api (ou o coordenador), e a forma usada aqui é a proposta.

1. **T1 — Equipes e ACS do admin.** O §3 lista microáreas por `team_id`, mas o admin não tem de onde tirar as equipes 70/76 nem os ACS candidatos. Proposta: `GET /territory/admin/teams` → `{ "teams": [ { "id", "ine", "name", "kind", "unit": { "id", "name" }, "acs_candidates": [ { "professional_id", "name", "has_cns" } ] } ] }` (`municipal_admin`; sem trilha; só equipes 70/76; candidatos = vínculo ativo com CBO 515105 na unidade).
2. **T2 — Contexto do profissional.** O §4 parte de `micro_area_id`, mas não diz como o profissional sabe suas equipes e microáreas. Proposta: `GET /territory/teams` → `{ "teams": [ { "id", "ine", "name", "unit", "micro_areas": [ <micro_area> + "can_register": bool ] } ] }` (escopo do vínculo; sem trilha; `can_register` = a mesma regra do api para gravar: ACS responsável pela microárea ou CBO 2235*/2251*/2252*/2253* na equipe).
3. **T3 — Moradores do domicílio.** O contrato só tem `GET /territory/registrations?patient_id=`. Proposta: aceitar também `?household_id=` (mesma forma `{ "registrations": [<registration>] }`, trilha com os ids mostrados).
4. **T4 — Pessoa pelo CPF.** Para começar uma ficha de quem tem CPF, a tela precisa achar paciente, par validado ou ninguém. Proposta: `POST /territory/person_lookup { "cpf" }` → `{ "person": { "patient_id"|null, "citizen_id"|null, "display_name"|null, "cpf_masked", "registration_id"|null } }` (trilha; 404 `person_not_found` quando não há nada; 422 `invalid_cpf`), no molde do `/care_lines/person_lookup` do 23.
5. **T5 — Identificação no corpo.** O §5 tem `new_patient` só na criação e diz "PATCH mesmo corpo"; não diz onde vão raça/cor, etnia, nacionalidade (campos da ficha que a spec diz gravarem **no paciente**) nem como informar CPF depois (`cpf_taken`). Proposta: `identity` (forma de `PatientIdentity` deste plano) em POST e PATCH, aceito só para o que o paciente ainda não tem validado no posto (o validado é só leitura, `identity_validated` em `<registration>.patient`); `identity.cpf` em paciente sem CPF → 409 `cpf_taken`. E `already_registered` devolve `registration_id` para a tela abrir o cadastro existente.
6. **T6 — Número da seção dos dicionários.** O contrato §1 manda publicar os dicionários "como acréscimo §11", mas o §11 já é "Maintenance". Proposta: a Task 0 do api publica como §14 ("Dicionários das fichas") e corrige a referência do §1; este plano procura a seção pelo título (Task 0, Step 4; Task 10, Step 1).
7. **T7 — CNS de quem não tem CPF.** O LEDI 8.7 (`dicionario-fci`) exige `cnsCidadao` quando `stNaoPossuiCpf` é verdadeiro; a spec diz "CNS quando houver" (recém-nascido muitas vezes não tem). Sem CNS, a ficha individual não passa. Decisão necessária (§11): CNS obrigatório no cadastro sem CPF, ou cadastro gravado com a ficha caindo em `ledi_generation_failures` até o CNS chegar. A Task 10 segue o que o §11 disser.
8. **T8 — Endereço da ficha domiciliar.** O LEDI exige `tipoLogradouroNumeroDne`, `stSemNumero`, `bairro` em texto, `codigoIbgeMunicipio` e `numeroDneUf`; para os tipos de imóvel 07–11, `instituicaoPermanencia` (responsável técnico obrigatório); e `stForaArea`. O `<household>.address` do §2 não tem tipo de logradouro, "sem número" nem instituição. Proposta: `address.street_type` (código DNE), `address.no_number`, e `institution { name, technical_lead_name, technical_lead_cns?, role?, phone?, other_staff? }` só para 07–11; bairro em texto e IBGE derivados no api (bairro do 11 e cidade).
9. **T9 — Endereço só leitura no faturamento e no check-in.** O §10 diz "só leitura, com o rótulo", mas o api do 22 nunca devolve o endereço e a tela não sabe quem tem cadastro ativo. Proposta: `address_from_household: bool` em `PendingItem.patient`, em `CheckInCitizen` e na resposta da busca do balcão; e o PATCH de dados do paciente com endereço de quem tem cadastro ativo → 422 `address_from_household`.
10. **T10 — Paciente sem CPF nas formas existentes.** As formas com paciente (`RecordPatient`, `BillingPatient`, `ExamPatient`, `RequestRow`, `QueueRow`, `AppointmentView.citizen`) têm `cpf_masked: string`. Proposta: `cpf_masked: null` e `cns_masked` (mascarado) quando não há CPF; o inventário de leitores de `patients.cpf` do plano do api confirma quais rotas mandam o quê.
11. **T11 — Óbito com número da DO e saída com recusa.** O LEDI aceita `numeroDO` (9 caracteres) com motivo 135 e proíbe motivo de saída com termo de recusa. Proposta: `exit` aceita `death_certificate?` (fora desta tela até alguém pedir); `exit` em cadastro recusado → 422 `invalid_registration` com `path: "exit"`.
12. **T12 — Validade para o admin.** O §6 não diz quem lê. Este plano mostra a validade só ao profissional (escopo da equipe). Proposta: registrar isso; se o coordenador quiser o admin vendo as contagens da cidade, `GET /registry_status/count` sem `team_id` para `municipal_admin` (sem trilha) e um cartão na tela de microáreas.
13. **T13 — Tipo dos valores das seções.** O §1 diz "chaves do IDL em snake_case e os domínios oficiais", sem o tipo JSON. Proposta: código LEDI (i64) como **inteiro**, lista de códigos como array de inteiros, `st*`/`status*` como booleano, texto como string; `nu_comodos`/`nu_moradores` como string (são `string` no IDL). Se o §11 fixar string para código, muda só o `toBody` das Tasks 11–12.
14. **T14 — Código de motivo de saída.** O §1 usa `moved`/`death`; o LEDI usa 136/135. Registrado: a tela usa `moved`/`death` (contrato) e o api traduz para a ficha.
15. **T15 — Raça/cor em dois códigos.** O paciente (módulo 22, BPA) guarda raça/cor como `01` branca, `02` preta, `03` parda, `04` amarela, `05` indígena; a ficha LEDI usa `1` branca, `2` preta, `3` amarela, `4` parda, `5` indígena (parda e amarela trocadas). Como a spec manda uma fonte por dado (o paciente), a tela usa o código do paciente em `identity.race_color` e o api traduz para a ficha. O §11 deve dizer isso explicitamente, ou a tradução se perde.

## Self-review

- **Cobertura (spec §2, §3, §5–§7; contrato §1–§7, §10):** interruptor `territory_registry` — Tasks 1 (rótulo) e 3 (menu e portão); microáreas com step-up, ACS da equipe com CNS — Task 4 (F-24.1); ACS e agente de endemias sem conselho — Task 5 (F-24.2); paciente sem CPF: "sem CPF" e CNS nas telas — Task 6; duplicados antes de criar, `cpf_taken` — Task 12 (F-24.3); domicílio e família com versões e `stale_version` — Task 11 (F-24.4); cadastro individual em seções, recusa com termo — Task 12; saída por mudança e óbito — Task 13 (F-24.5); validade de 24 meses, a vencer em 60 dias e quem falta cadastrar, com atalho — Tasks 9 e 12 (F-24.6); território microáreas → domicílios → famílias → moradores — Task 8; endereço só leitura do 22 — Task 7; validação espelhando o api com os dicionários do §11, dependência explícita e bloqueante — Tasks 0 (Step 4) e 10 (Step 1); sem rascunho no navegador — Global Constraints, Task 12 (teste) e Task 14 (Step 2 e prova 13); nada na URL — Tasks 1, 12 e 14; sem polling em rota com trilha — Tasks 8, 9 e 14; prova no navegador com roteiro — Task 14. F-24.7 (fichas LEDI) e F-24.8 (equipe e óbito) são do api; aqui só aparecem (prova 10).
- **Placeholders:** os únicos valores lidos na execução são os do §11 (Task 10, com a regra escrita: o §11 vence e a diferença vira nota), os ids dos cards (Task 0), o CNS de teste se o `isValidCns` do arquivo recusar o do plano (Task 5) e o código de raça/cor de 2 dígitos do `RACE_COLORS` do 22 (Task 7). Nenhum passo diz "trate os casos".
- **Tipos e nomes:** `MicroArea`, `TerritoryAdminTeam`, `TerritoryTeam`/`TerritoryContext`, `Household`/`HouseholdInput`, `Family`/`FamilyInput`, `DuplicateCandidate`, `TerritoryPerson`, `PatientIdentity`, `Registration`/`RegistrationInput`, `ExitInput`, `PersonRow`, `RegistryCounts` (Task 1) são os usados nas Tasks 4–13; `updateHousehold(id, input, version)`, `addFamily(householdId, input, version)`, `updateFamily(id, input, version)` e `updateRegistration(id, input, version)` têm a versão como 3º argumento em todas as tasks e testes; `countRegistryStatus(teamId, microAreaId?)` igual nas Tasks 1 e 9; `TERRITORY_DISABLED`, `STALE_TEXT`, `territoryError`, `errorPath`, `isStale` (Task 2) batem com os usos; `RegistrationDraft`, `registrationProblem(draft, today)`, `householdProblem(input, families)`, `familyProblem(input, today)`, `exitProblem(input, birthDate, today)`, `canBePregnant(sex, birthDate, today)` (Task 10) são os chamados nas Tasks 11–13; `PersonChoice` (Task 12) é o que `RegistryStatus.onRegister` monta; os slots `actions` de `HouseholdList`/`HouseholdDetail` (Task 8) são os preenchidos nas Tasks 11–12.
- **Review Focus:** 1 — Task 12 ("sem CPF: só cria depois da busca; escolher o candidato manda patient_id"); 2 — Task 12 ("nada no storage nem na URL: CPF, nome, condições só no corpo"); 3 — Task 11 ("409 stale_version: frase, o formulário fica, recarregar relê"); 4 — Task 12 ("recusa: só identificação e refused; as seções somem") e Task 11 ("recusa do domicílio: sem moradia nem família; endereço fica"); 5 — Task 9 ("volta à aba e passa o tempo: a lista não relê; os contadores usam COUNT_QUERY").
- **Proporção:** os blocos de código são testes (os nomes e valores que o implementador não decide sozinho) e os tipos do contrato; os corpos das telas ficam com a assinatura e o comportamento.
