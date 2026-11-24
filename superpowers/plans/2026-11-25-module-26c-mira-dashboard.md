# Módulo 26c — Envio da regulação à RNDS (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-25-module-26c-mira-contracts.md` (fonte única), §1, §5, §6, §7 e Q1–Q14 (§10; docs `5d49ef7`). As divergências D1–D10 deste plano foram resolvidas pelas Q9, Q11, Q12, Q13 e Q14 (seção final), e o plano já aplica as decisões. As formas de `/status` e `/syncs` são as do plano do api (`2026-11-25-module-26c-mira-api.md`, Task 14), pela Q11.
> 2. A **`main` do dashboard com o 27b mergeado.** O contrato manda executar o 26c depois do 27b. Quando este plano foi escrito, o `origin/main` estava em `2e3a2a9`, só com o 35a (nem o 35b, nem o 27b).
> 3. O **api do 26c** na branch `feat/mod-26c-mira` (`apps/api/.claude/mod26c`, porta **3063**). As Tasks 1–7 seguem pelas formas deste plano e pelas fixtures. Só a Task 8 (prova) depende do api rodando.
>
> Ordem: api → **dashboard** e maintenance → suíte completa do api (sessão "Testes") → prova contra o simulado (`fake_rnds`, porta 8094).

**Goal:** Lado dashboard do 26c:
- no grupo **Regulação**, a tela "Envio à RNDS", com faixa do alarme, totais, lista por motivo com atalho para corrigir e "Reenviar";
- em "Prazos e lista do enfermeiro", a seção "Envio à RNDS" (cor → caráter, SIGTAP por especialidade, envio dos externos);
- o campo "CBO do executante" na oferta e no prestador;
- a credencial `rnds_regulation` em Integrações.

**Architecture:** O cliente ganha a seção "Envio da regulação à RNDS (módulo 26c)" no fim do bloco da regulação em `src/lib/api.ts`, com `regPath`/`regWrite` do 26a e `INTEGRATIONS_BASE` do 16. Onde fica cada coisa:
- frases: `src/lib/regulationRndsErrors.ts` (`REGULATION_RNDS_ERRORS`, contrato §6);
- portão, papéis, rótulos, faixa, atalhos e regras de formato: `src/lib/regulationRnds.ts`;
- telas: `src/modules/RegulationRnds.tsx` e os painéis em `src/modules/regulation/`.

O dashboard não decide o que o api calcula: situação desejada, passos, digest, conferência, atraso, carga inicial e backoff. Ele mostra o que vem e só não oferece o que sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-25-module-26c-mira-design.md` (decisões 1, 5, 6, 8, 9, 13, 14, 15; §3.3–§3.5). **Contrato:** `docs/superpowers/plans/2026-11-25-module-26c-mira-contracts.md` §1, §3 (configuração e colunas novas), §4, §5, §6, §9 e Q1–Q14. **Plano do api:** `2026-11-25-module-26c-mira-api.md` (Valores 1, 2, 8 e 10; Tasks 6, 13 e 14). **Molde:** `docs/superpowers/plans/2026-11-24-module-27b-bnafar-dashboard.md`; `src/modules/VaccinationPending.tsx` (pendências) e `src/modules/integrations/RndsCredentialForm.tsx` (certificado).

## Global Constraints

- **Nomes no código** (contrato §1, §5, §6): interruptor `rnds_regulation`; credencial `rnds_regulation`; tela `regulation-rnds`; frases em `REGULATION_RNDS_ERRORS` (`src/lib/regulationRndsErrors.ts`). **Código desconhecido aparece cru, nunca some.**
- **Frases fixas** (letra por letra):
  - o interruptor: "Envio da regulação à RNDS";
  - o item e a seção: "Envio à RNDS";
  - o botão "Reenviar";
  - a caixa "Enviar também os pedidos regulados pelo estado", com o aviso "Ligue só se a cidade pactuou com o estado ser a responsável pelo envio desses pedidos.";
  - o campo "CBO do executante";
  - os campos da credencial "Identificador do solicitante (DATASUS)" e "CNS do responsável pelo envio";
  - o pré-requisito `rnds_sending_disabled`: "ligue antes um dos envios à RNDS (vacina ou regulação)" (Q14);
  - as telas existentes "Prazos e lista do enfermeiro", "Prestadores e cotas" e "Integrações".

  Toda outra frase deste plano é **proposta** e está marcada assim.
- **Portões** (contrato §5):
  - `canUseRegulationRnds(user)` = `canUseRegulation(user)` e `rnds_regulation` na sessão. A sessão traz as chaves **ligadas** (`src/lib/features.ts:1-4`). Pela Q10, as leituras (status, lista, configurações) só exigem o interruptor **ligado**: com a credencial recusada, a tela abre e a faixa diz o que falta (`usable: false`, `missing`).
  - Tela "Envio à RNDS" (status e lista): `municipal_admin` e `care_regulator`.
  - "Reenviar": só o `municipal_admin`, com step-up, e só em `rejected` ou `failed`. Ele exige o envio **utilizável** (Q10): pausado, o api responde `403 feature_disabled`.
  - Configurações (Q13):
    - a seção editável fica em "Prazos e lista do enfermeiro", que é só do `municipal_admin` (`src/shell/modules.ts:497`). Ela aparece **sem** exigir `rnds_regulation`, porque as configurações valem antes de ligar (§5);
    - o `care_regulator` vê um resumo só de leitura na tela "Envio à RNDS".
  - Credencial: o `municipal_admin`, sem exigir o interruptor.
  - CBO da oferta e do prestador: o mesmo formulário de hoje (escrita do admin; o regulador só lê a coluna).
- **Formato** (recusar, nunca cortar; o api confere de novo). Pontos, hífens, barras e espaços saem antes de contar:
  - CBO com 6 dígitos (`"2251-20"` → `"225120"`);
  - SIGTAP com 10 dígitos (`"03.01.01.007-2"` → `"0301010072"`), mostrado como `03.01.01.007-2`;
  - CNES com 7 dígitos (o `cnesProblem` de `src/lib/integrations.ts`);
  - CNS com 15 dígitos e dígito válido (`isValidCns`, `src/lib/professionals.ts:22`);
  - identificador do solicitante com 1–20 dígitos (Q13);
  - caráter só `urgent` ou `routine`.
- **Step-up** (`SensitiveAction` com `requiresStepUp`) em quatro escritas: gravar a credencial, "Reenviar", salvar as configurações (Q13) e salvar a oferta ou o prestador (já hoje). O teste de conexão não pede. Toda escrita leva corpo JSON (`{}` quando não há campo).
- **Concorrência:** o PUT das configurações manda `expected_version`. Um `409 rnds_settings_changed` mostra a frase, faz **uma** releitura e mantém o digitado. O botão de escrita fica desabilitado enquanto a chamada corre: o duplo clique manda um pedido só.
- **Sigilo** (contrato, convenções e §9):
  - a lista nunca mostra paciente: nem nome, CPF, CNS, CID nem justificativa;
  - na URL vão só `status`, `reason`, o cursor e ids;
  - o certificado, a senha, o identificador e o CNS do responsável só vivem no estado do formulário e no `SensitiveAction`, nunca em URL, cache, log, storage ou `console`;
  - abrir o pedido pelo atalho é a leitura nominal normal da fila (com trilha), só para o `care_regulator`.
- **Leituras:** tudo nesta tela é sem pessoa → `STATIC_QUERY`, relido ao abrir (`staleTime: 0`, `refetchOnMount: "always"`, como em `VaccinationPending.tsx`), depois de cada escrita e no "Atualizar". **Nenhum polling.**
- **Interface:**
  - toda tela nova entra em `NAV_GROUPS` **e** em `CATALOG.screens` (o `catalog.test.ts` cobra);
  - `DataTable` com `card: "title"`/`"actions"`, e 375 px sem rolagem horizontal;
  - cores só por papéis do tema (`Tag` com `tone`), com as guardas `src/theme/*.test.ts` verdes;
  - componentes existentes: `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `KeyValue`, `SensitiveAction`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts` e `src/modules/regulation/styles.ts`, `RegulationGate`, `TargetSearch`.
- **CI (lição do `443faca`):** todo `vi.mock("…/lib/api", async (orig) => …)` inclui `fetchCurrentSession: vi.fn()` resolvendo o usuário.
- **Datas:** "agora" é `new Date()`, e as telas recebem `now` por prop. Os testes usam `NOW26C = new Date("2026-11-27T10:00:00-03:00")` (`TZ=America/Sao_Paulo`).
- **Testes:** só `$R/bin/test dashboard -w dashboard-mod26c [arquivos do vitest]`, nunca `vitest` direto. Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge, push e tag só com autorização explícita.
- **Board #1:** o dashboard é superfície dos cards do 26c (F-26.8 em diante, criados pelo plano do api). Card vai a Done só quando o **último** app dele termina, e a Verified depois da prova (Task 8).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`): `dashboard · dashboard-mod26c/feat/mod-26c-mira · Task N · <pronto> · <divergência nova>`.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod26c`, `G=/opt/homebrew/bin/git`, `A=$R/apps/api/.claude/mod26c`.
- Worktree **fora de `apps/dashboard`**, porque o compose monta `./apps/dashboard:/app` e um `node_modules` ali duplica o React no Vite. Branch `feat/mod-26c-mira` a partir de `origin/main`, com `npm ci`. Os caminhos das tasks são relativos a `$W`.
- Testes sem jest-dom (`not.toBeNull()`, `toBe()`), com `globals: false` e `cleanup` no `afterEach`.
- Fixtures:
  - `renderWithProviders`/`sessionWith`: `src/test/campaignFixtures.tsx`;
  - do 26a: `src/test/regulationFixtures.ts` (`regUser`, `offer`, `provider`, `CARDIO`, `ECO`, `UBS_CENTRO`);
  - novas: `src/test/regulationRndsFixtures.ts` (Task 1).

## Review Focus

1. **"Reenviar" com duplo clique, com o step-up vencido ou num pedido que já mudou.** Espera-se:
   - um pedido só;
   - `401 mfa_required` volta ao step-up sem reenviar;
   - `409 not_retryable` mostra a frase e relê a lista uma vez;
   - `403 feature_disabled` (envio pausado, Q10) mostra a frase do interruptor parado, sem reenviar;
   - o regulador nunca vê o botão, e linha `blocked` ou `queued` também não.

   Teste: Task 7, "reenviar: step-up, um pedido, estado mudado relê, só admin e só recusado ou falho".
2. **Duas abas salvando as configurações.** Espera-se que o `409 rnds_settings_changed` mostre a frase, releia **uma** vez e mantenha o digitado, e que o segundo envio leve a versão relida. Teste: Task 5, "configurações: 409 relê uma vez, mantém o digitado e reenvia com a versão nova".
3. **CBO e SIGTAP digitados como as pessoas digitam:** `"2251-20"`, `" 225120 "`, `"22512"`, `"03.01.01.007-2"`, `"030101007"` e `"0301010072 "`. Espera-se o código limpo ou a recusa com a frase, nunca corte. Teste: Task 2, `cboOf` e `sigtapOf` por tabela.
4. **Api mais novo que a tela:** um motivo novo, `skeleton:Appointment.start`, uma recusa `EHR-ERR999` ou uma situação nova. Espera-se o código cru ou a frase genérica do esqueleto, nunca `undefined`, e a linha nunca some da lista. Teste: Task 7, "motivo, recusa e situação desconhecidos aparecem crus".
5. **O atalho certo para cada papel.** Espera-se:
   - o regulador abre o pedido pelo `request_ref` (Q11) e por "Abrir o pedido", e não vê "Abrir CNES" nem "Abrir a oferta";
   - o admin vê os atalhos da oferta, do prestador, do CNES e das configurações;
   - paciente sem CPF e CNS nunca ganha atalho para o cadastro, só a frase;
   - nenhuma coluna de pessoa.

   Teste: Task 7, "atalhos por motivo e por papel, sem paciente".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.regulationRnds.test.ts`, `src/lib/api.regulation.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/regulationRndsFixtures.ts`, `src/test/regulationFixtures.ts` | tipos e cliente do 26c; extensões do 16 e do 26a; rótulo do interruptor; fixtures | 1 |
| `src/lib/regulationRndsErrors.ts`, `src/lib/regulationRndsErrors.test.ts`, `src/lib/regulationRnds.ts`, `src/lib/regulationRnds.test.ts` | frases; portão, rótulos, faixa, atalhos, formato, configurações | 2 |
| `src/shell/modules.ts`, `src/shell/modules.regulationRnds.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/regulation/useRegulationHandoff.ts`, `src/modules/RegulationRnds.tsx` (casca), `src/modules/RegulationQueue.tsx`, `src/modules/RegulationQueue.handoff.test.tsx` | item do menu, portão, atalho até o pedido | 3 |
| `src/modules/regulation/OfferForm.tsx`, `src/modules/regulation/ProviderForm.tsx`, `src/modules/RegulationOffers.tsx`, `src/modules/RegulationOffers.cbo.test.tsx`, `src/modules/RegulationOffers.test.tsx` | "CBO do executante"; abrir a oferta ou o prestador pelo atalho | 4 |
| `src/modules/regulation/RndsSettingsPanel.tsx`, `src/modules/RegulationSettings.tsx`, `src/modules/RegulationSettings.rnds.test.tsx` | seção "Envio à RNDS" | 5 |
| `src/lib/integrations.ts`, `src/lib/integrations.test.ts`, `src/modules/Integrations.tsx`, `src/modules/integrations/RndsRegulationCredentialForm.tsx`, `src/modules/Integrations.rndsRegulation.test.tsx` | credencial `rnds_regulation` | 6 |
| `src/modules/RegulationRnds.tsx`, `src/modules/regulation/RndsAlertBand.tsx`, `src/modules/regulation/RndsFixCell.tsx`, `src/modules/regulation/RndsSettingsSummary.tsx`, `src/modules/RegulationRnds.test.tsx` | "Envio à RNDS", com o resumo das configurações para o regulador | 7 |
| — | suíte, build, varreduras, revisão, prova no navegador (5224 → 3063) | 8 |
| — | board, relato, parada antes do merge | 9 |

**Estratégia de teste:**
- **Cliente** (Task 1): `fetch` falso (`urlOf`, `initOf`, `bodyOf`, como em `api.regulation.test.ts`), conferindo URL, método e corpo; o certificado, a senha, o identificador e o CNS só no corpo; o filtro vazio nunca na URL.
- **Regras puras** (Task 2): tabela de casos, com as bordas de formato do Review Focus 3, a faixa por contagem e o atalho por motivo e papel.
- **Menu** (Task 3): por papel e por interruptor.
- **Telas** (Tasks 3–7): pelo ponto de entrada, com `renderWithProviders`, `vi.mock` do api com `fetchCurrentSession` e sessão por papel e por interruptor. Cobrem:
  - o caminho feliz;
  - cada recusa que muda a tela;
  - o duplo envio;
  - o 409 com uma releitura;
  - o que **não** pode aparecer: paciente na lista, segredo no cache, botão de escrita para quem só lê, atalho para quem não pode agir.
- **Fim** (Task 8): `tsc`, build, suíte, varreduras e prova no navegador, com o api real na 3063 e o Vite na 5224.

O dashboard não testa o que é do api: a correspondência de situação, os passos da falta, o digest, a conferência do esqueleto, o envio e a substituição, o `GET /identifier`, o backoff, a carga inicial e o cálculo do atraso.

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (com o 27b); board #1; o api `feat/mod-26c-mira`.
- Produces: cards em In Progress; `$W` na branch; base, âncoras, ícones e formas do api anotados.

- [ ] **Step 1: Cards.** Um `gh` simples por vez, sem `$(...)` nem laço:

  ```bash
  gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-26\\.([89]|[1-9][0-9]) ")) | {id, title, status}'
  ```

  Os cards do 26c que o dashboard toca vão a In Progress, com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <In Progress>`. Card ausente: **pare e pergunte** (quem cria é o plano do api).

- [ ] **Step 2: Base e âncoras**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin && $G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard log --oneline origin/main | grep -ci bnafar
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export type CredentialKind\|^export interface CareProvider {\|^export interface CareProviderInput\|^export interface CareOffer {\|^export interface CareOfferInput\|^export function updateCareOffer\|^export async function setRndsCredential\|^export function removeNurseItem\|^export async function removeNurseItem'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "vector_control"\|bnafar\|rnds_mock:'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '{ id: "regulation-settings", label: "Prazos e lista do enfermeiro"\|if (item.id === "regulation-settings") return'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'case "regulation-queue"\|case "regulation-offers"\|export function renderModule\|export function useOmbudsmanHandoff'
$G -C $R/apps/dashboard show origin/main:src/lib/integrations.ts | grep -n '^const CREDENTIAL_LABEL\|case "credential_unauthorized":\|kind === "rnds"\|export function credentialLabel'
$G -C $R/apps/dashboard show origin/main:src/modules/Integrations.tsx | grep -n 'editing?.kind === "rnds"\|editing.kind !== "rnds"'
```

Expected:
- o 27b presente na `main` (contagem > 0). Se for 0, **pare**: o contrato manda começar depois dele;
- cada linha achada. Linha mudada pelo 27b (por exemplo, `credentialLabel(kind, environment?)` e o `editing?.kind === "bnafar"`): a task dona aplica a mesma intenção sobre o texto real e anota no commit.

- [ ] **Step 3: Worktree, base e ícones**

```bash
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod26c -b feat/mod-26c-mira origin/main
cd $R/.worktrees/dashboard-mod26c && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod26c
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^IconSend\.mjs$'
```

Expected: `tsc-ok`; suíte verde (**anote** arquivos e testes); o ícone. Faltou: use `IconCloudUpload`, na Task 3.

- [ ] **Step 4: Formas do api.** Leia, na branch `feat/mod-26c-mira` do api:
  - as rotas: `$G -C $R/apps/api show feat/mod-26c-mira:config/routes.rb | grep -n 'rnds\|rnds_settings\|credentials'`;
  - os controllers e serializadores de `/regulation/rnds/*` e `/regulation/admin/rnds_settings`, e o `Json.offer`/`Json.provider` com o CBO;
  - os specs de request;
  - o `Platform::Features::CATALOG` (`rnds_regulation`, `requires`) e o `IntegrationCredential::SECRET_KEYS`.

  Confira tudo contra as formas da Task 1. **Onde o api diz outra coisa, vale o api**: a task dona ajusta tipo, fixture e teste e registra no commit. Sem branch ainda: siga pelas formas deste plano e repita este passo antes da Task 8.

  Confira também o que as decisões fixaram:
  - as formas da Q11: `request_ref` no item, `accepted` nas contagens, `usable` e `missing` no status;
  - o PATCH da oferta só com o CBO e o POST com `performer_cbo` (Q12);
  - os códigos e formatos da Q13 (os da Task 2). Se o api ainda responder `invalid_request` com `path` em vez de `invalid_cbo`/`invalid_sigtap`/`invalid_priority`, a Task 2 já traduz os dois; avise o coordenador.

---

### Task 1: Tipos, cliente HTTP, rótulo do interruptor e fixtures

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim do bloco da regulação; extensões no 16 e no 26a), `src/lib/features.ts`, `src/lib/api.regulation.test.ts` (o `updateCareOffer` novo), `src/test/regulationFixtures.ts` (os campos novos com `null`)
- Create: `src/lib/api.regulationRnds.test.ts`, `src/test/regulationRndsFixtures.ts`
- Test: `src/lib/features.test.ts`

**Interfaces:**
- Produces: em `src/lib/api.ts`, depois de `removeNurseItem`, a seção "Envio da regulação à RNDS (módulo 26c; contrato 2026-11-25 §1, §3–§6)". Listas `{ items, next_before }`. O que é proposta está marcado:

```ts
export type RndsSyncStatus = "idle" | "queued" | "sending" | "blocked" | "rejected" | "failed" | "excluded" | "erased";
export type MiraStatus = "pending" | "returned-to-requester" | "booked" | "attended" | "cancelled" | "absence" | "excluded";
export type RndsBlockReason =
  | "missing_diagnosis" | "requester_unit_without_cnes" | "performer_without_cnes" | "patient_without_cpf_or_cns"
  | "performer_cbo_missing" | "specialty_without_sigtap";                       // mais `skeleton:<campo>`
export type MiraPriority = "urgent" | "routine";
// Q11: as formas do plano do api (Task 14). `accepted` = linhas `idle` com documento aceito. Q10: `usable`/`missing` dizem se o envio está parado.
export interface RegulationRndsCounts { queued: number; blocked: number; rejected: number; failed: number; late: number; accepted: number }
export interface RegulationRndsStatus {
  usable: boolean; missing: string[]; counts: RegulationRndsCounts; late_banner: boolean; initial_load_pending: number;
}
// §5 e Q11: sem paciente, CID, alvo nem payload. `request_ref` = 8 hex do id do pedido, em maiúsculas, calculado pelo api.
export interface RegulationRndsSync {
  id: string; regulation_request_id: string; request_ref: string;
  status: RndsSyncStatus | (string & {}); desired_status: MiraStatus | (string & {}) | null; sent_status: MiraStatus | (string & {}) | null;
  blocked_reasons: string[]; last_error_codes: string[]; attempts: number;
  enqueued_at: string | null; last_accepted_at: string | null; next_attempt_at: string | null; late: boolean;
  fix: { offer_id: string | null; provider_id: string | null; unit_id: string | null; settings: boolean };
}
export interface RegulationRndsPage { items: RegulationRndsSync[]; next_before: string | null }
// §3: linha única; o padrão 0301010072 vale para todo CBO sem entrada.
export interface RegulationRndsSettings {
  priority_by_color: Record<RiskColor, MiraPriority>; specialty_sigtap: Record<string, string>; send_external: boolean; version: number;
}
export type RegulationRndsSettingsInput = Omit<RegulationRndsSettings, "version"> & { expected_version: number };
```

- Funções (todas `Promise`):
  - `getRegulationRndsStatus(): RegulationRndsStatus` → `GET /regulation/rnds/status`.
  - `listRegulationRndsSyncs(q: { status?: string | null; reason?: string | null; before?: string | null }): RegulationRndsPage` → `GET /regulation/rnds/syncs?status=&reason=&before=`, só com as chaves preenchidas; envelope ausente = vazio.
  - `retryRegulationRndsSync(id): RegulationRndsSync` → `POST /regulation/rnds/syncs/:id/retry` com `{}`.
  - `getRegulationRndsSettings()` → `GET /regulation/admin/rnds_settings`.
  - `updateRegulationRndsSettings(i: RegulationRndsSettingsInput)` → `PUT /regulation/admin/rnds_settings` com o corpo inteiro.
  - `setRndsRegulationCredential(file: Blob, password: string, requesterCnes: string, requesterId: string, responsibleCns: string): IntegrationCredential` → `PUT /integrations/credentials/rnds_regulation` com `{ pfx_base64, pfx_password, requester_cnes, requester_id, responsible_cns }`. O `readBase64` do módulo é reaproveitado, e o nome do arquivo não vai. O teste de conexão usa o `checkIntegrationCredential("rnds_regulation")` que já existe.
- Extensões:
  - 16: `CredentialKind` + `"rnds_regulation"`.
  - 26a:
    - `CareOffer.performer_cbo: string | null` e `CareOfferInput.performer_cbo?: string | null`;
    - `CareProvider.default_performer_cbo: string | null` e `CareProviderInput.default_performer_cbo?: string | null`;
    - `updateCareOffer(id: string, input: { quantity?: number; performer_cbo?: string | null })` troca a assinatura `(id, quantity)`. O corpo leva só as chaves que vêm: pela Q12, o PATCH aceita mudar só o CBO, sem `quantity`;
    - `createCareOffer` leva `performer_cbo` quando preenchido (Q12). Em branco, o api herda o da oferta do mês anterior do mesmo prestador e alvo.
- `src/lib/features.ts`: `FeatureKey` + `"rnds_regulation"`; `FEATURE_LABEL.rnds_regulation = "Envio da regulação à RNDS"`, com o comentário de uma linha do módulo.
- Fixtures (`src/test/regulationRndsFixtures.ts`):
  - `NOW26C`;
  - `RNDS_FEATURES = [ "care_regulation", "clinical_record", "rnds_regulation" ]` e `rndsUser(roles = [ "care_regulator" ], features = RNDS_FEATURES)`;
  - `rndsStatus(o?)` (`usable: true`, `missing: []`, `counts` com `accepted: 40` e `late: 0`, `late_banner: false`, `initial_load_pending: 0`), `lateStatus(o?)` (`late: 3`, `late_banner: true`, `blocked: 2`, `rejected: 1`), `pausedStatus(o?)` (`usable: false`, `missing: [ "credential_unauthorized:rnds_regulation" ]`);
  - `rndsSync(o?)` (`idle`, `regulation_request_id: "rr1"`, `request_ref: "4F9A1C2E"`, `late: false`, `fix` todo nulo e `settings: false`), `blockedSync(reasons, o?)`, `rejectedSync(codes, o?)`, `failedSync(o?)`;
  - `rndsSettings(o?)` (`{ red: "urgent", yellow: "routine", green: "routine", blue: "routine" }`, `{ "225120": "0301010072" }`, `send_external: false`, `version: 1`);
  - `rndsRegulationCredential(o?)`.

- [ ] **Step 1: Testes que falham** — `src/lib/api.regulationRnds.test.ts` (molde `src/lib/api.regulation.test.ts`):

```ts
it("status, lista sem filtro vazio na URL e reenvio com corpo vazio", async () => {
  fetchMock.mockImplementationOnce(() => ok(rndsStatus())).mockImplementationOnce(() => ok({})).mockImplementationOnce(() => ok(rejectedSync([ "EHR-ERR906" ])));
  await getRegulationRndsStatus(); expect(urlOf(0).pathname).toBe("/regulation/rnds/status");
  expect(await listRegulationRndsSyncs({ status: "blocked", reason: "", before: null })).toEqual({ items: [], next_before: null });
  expect(urlOf(1).pathname + urlOf(1).search).toBe("/regulation/rnds/syncs?status=blocked");
  await retryRegulationRndsSync("rs1");
  expect([ urlOf(2).pathname, initOf(2).method, bodyOf(2) ]).toEqual([ "/regulation/rnds/syncs/rs1/retry", "POST", {} ]);
});
it("configurações: PUT com o corpo inteiro e expected_version", async () => {
  const s = rndsSettings();
  await updateRegulationRndsSettings({ priority_by_color: s.priority_by_color, specialty_sigtap: s.specialty_sigtap, send_external: true, expected_version: 1 });
  expect([ urlOf(0).pathname, initOf(0).method, bodyOf(0).expected_version, bodyOf(0).send_external ]).toEqual([ "/regulation/admin/rnds_settings", "PUT", 1, true ]);
});
it("credencial da regulação: certificado, senha, CNES, identificador e CNS só no corpo", async () => {
  await setRndsRegulationCredential(new File([ "abc" ], "cert.pfx"), " s3nh@ ", "2384310", "12345", "898001160660761");
  expect([ urlOf(0).pathname, urlOf(0).search, initOf(0).method ]).toEqual([ "/integrations/credentials/rnds_regulation", "", "PUT" ]);
  expect(bodyOf(0)).toEqual({ pfx_base64: "YWJj", pfx_password: " s3nh@ ", requester_cnes: "2384310", requester_id: "12345", responsible_cns: "898001160660761" });
});
it("oferta: o PATCH leva só as chaves que vêm (Q12) e o POST leva o CBO", async () => {
  await updateCareOffer("of1", { performer_cbo: "225120" }); expect(bodyOf(0)).toEqual({ performer_cbo: "225120" });
  await updateCareOffer("of1", { quantity: 12 }); expect(bodyOf(1)).toEqual({ quantity: 12 });
  await updateCareOffer("of1", { quantity: 12, performer_cbo: null }); expect(bodyOf(2)).toEqual({ quantity: 12, performer_cbo: null });
  await createCareOffer({ provider_id: "pr2", target_type: "procedure", target_code: "0205010032", month: "2026-12", quantity: 4, performer_cbo: "225120" });
  expect(bodyOf(3).performer_cbo).toBe("225120");
});
```

  Em `src/lib/api.regulation.test.ts`, a chamada antiga `updateCareOffer("of1", 12)` vira `updateCareOffer("of1", { quantity: 12 })`, com o mesmo corpo esperado. Em `src/lib/features.test.ts`: `featureLabel("rnds_regulation")` = `"Envio da regulação à RNDS"`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod26c src/lib/api.regulationRnds.test.ts src/lib/api.regulation.test.ts src/lib/features.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces. O `OfferForm` passa a chamar `updateCareOffer(offer.id, { quantity })` (a Task 4 acrescenta o CBO).
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS. O `tsc` pede `performer_cbo: null` e `default_performer_cbo: null` nas fixtures `offer()` e `provider()` do 26a; acrescente.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.regulationRnds.test.ts src/lib/api.regulation.test.ts src/lib/features.ts src/lib/features.test.ts src/test/regulationRndsFixtures.ts src/test/regulationFixtures.ts src/modules/regulation/OfferForm.tsx
$G commit -m "feat: add the regulation RNDS client, the performer CBO fields and the rnds_regulation credential kind" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Frases, portão, rótulos, faixa, atalhos e formato

**Files:**
- Create: `src/lib/regulationRndsErrors.ts`, `src/lib/regulationRndsErrors.test.ts`, `src/lib/regulationRnds.ts`, `src/lib/regulationRnds.test.ts`

**Interfaces:**
- Consumes: `hasFeature`, `featureDisabledKey` (`src/lib/features.ts`); `missingPhrase` (`src/lib/integrations.ts`); `canUseRegulation`, `regulationRoles`, `colorLabel`, `COLORS`, `STATIC_QUERY` (`src/lib/regulation.ts`); `ApiError`, `errorCode`; `isValidCns`.
- Produces (`src/lib/regulationRndsErrors.ts`):
  - `REGULATION_RNDS_DISABLED = "o envio da regulação à RNDS está desligado nesta cidade"` (proposta);
  - `REGULATION_RNDS_ERRORS: Record<string, string>` (contrato §6): uma frase por código, todas distintas, sem `_`, sem dado de pessoa;
  - `regulationRndsErrorText(code: string): string`: `skeleton:<campo>` → "o documento ficou sem um campo obrigatório (<campo>) — avise a equipe do Rota Saúde"; código ausente → `"recusa da RNDS: <código>"`.

  **As frases são todas proposta.** Os códigos são os do contrato §3 e os da pesquisa (achados 5, 14 e a seção de integração); os marcados com Q13 foram fixados pela decisão (o api os emite):

| Código | Frase |
|---|---|
| `missing_diagnosis` | "pedido sem CID-10 nem CIAP-2 — a regulação devolve o pedido à unidade para completar" |
| `requester_unit_without_cnes` | "unidade solicitante sem CNES — confira a unidade na importação do CNES" |
| `performer_without_cnes` | "executante sem CNES — confira o prestador ou a unidade da rede própria" |
| `patient_without_cpf_or_cns` | "paciente sem CPF e sem CNS no cadastro — a unidade completa o cadastro do cidadão" |
| `performer_cbo_missing` | "falta o CBO do executante — informe na oferta ou no prestador" |
| `specialty_without_sigtap` | "o SIGTAP desta especialidade (o configurado ou o padrão) não é regulável ou não está vigente — troque em Prazos e lista do enfermeiro, na seção Envio à RNDS" (Q9) |
| `EHR-ERR906` | "a RNDS não aceitou o CNS do responsável pelo envio — ele precisa ter vínculo num CNES da credencial" |
| `EHR-ERR881` | "a RNDS não reconheceu o identificador do solicitante — confira a credencial em Integrações" |
| `EDS-MSG009` | "a RNDS já tinha substituído este documento — o pedido é conferido antes de reenviar" |
| `EHR-ERR825` | "a RNDS não achou o documento duplicado a remover" |
| `feature_disabled` | `REGULATION_RNDS_DISABLED` |
| `rnds_settings_changed` | "as configurações foram salvas por outra pessoa — foram relidas; confira e salve de novo" |
| `not_retryable` (Q13) | "este pedido não está mais recusado nem com falha — a lista foi relida" |
| `invalid_priority` (Q13) | "o caráter de cada cor é urgência ou eletivo" |
| `invalid_cbo` (Q13) | "o CBO tem 6 dígitos" |
| `invalid_sigtap` (Q13) | "o código SIGTAP tem 10 dígitos e precisa ser um procedimento regulável vigente" |
| `invalid_requester_id` (Q13) | "o identificador do solicitante tem só dígitos, como o DATASUS informou" |
| `invalid_responsible_cns` (Q13) | "o CNS do responsável tem 15 dígitos e precisa ser válido" |
| `expected_version_required` | "a tela não mandou a versão das configurações — recarregue a página" |

- Produces (`src/lib/regulationRnds.ts`):
  - **Portão e papéis:** `canUseRegulationRnds(user)`; `rndsRoles(user): { admin: boolean; regulator: boolean; reader: boolean }`, com `reader = admin || regulator`.
  - **Chaves:** `RNDS_STATUS_KEY`, `RNDS_SYNCS_KEY`, `RNDS_SETTINGS_KEY`; reexporta `STATIC_QUERY`.
  - **Rótulos** (proposta; código desconhecido cru):
    - `syncStatusLabel`/`syncStatusTone`: `idle` "aceito" (ok), `queued` "na fila" (info), `sending` "enviando" (info), `blocked` "parado: dado a corrigir" (down), `rejected` "recusado pela RNDS" (down), `failed` "falhou: a RNDS não respondeu por 15 dias" (down), `excluded` "fora da regulação municipal" (neutral), `erased` "paciente apagado: nada mais é enviado" (neutral);
    - `miraStatusLabel`: `pending` "pendente", `returned-to-requester` "devolvido ao solicitante", `booked` "agendado", `attended` "realizado", `cancelled` "cancelado", `absence` "falta", `excluded` "excluído";
    - `priorityLabel`: `urgent` "urgência", `routine` "eletivo".
  - **Faixa** (`alertLines(s: RegulationRndsStatus): string[]`, uma linha por condição, frases proposta; Q13: a faixa só existe nesta tela):
    - `usable: false` → "O envio à RNDS está parado. Falta: <missing por `missingPhrase`, unidos por "; ">." (Q10). Com `credential_unauthorized:rnds_regulation`, a frase do `missingPhrase` diz que o certificado foi recusado (Task 6);
    - `late_banner` → "N pedidos mudados até ontem ainda sem aceite da RNDS. O envio precisa ser aceito até as 23h de hoje.", com o singular "1 pedido mudado até ontem ainda sem aceite…";
    - `initial_load_pending > 0` → "Carga inicial: N pedidos na fila, no ritmo de 60 por minuto. Eles ficam fora do atraso nas primeiras 72 horas.".
  - **Totais:** `totals(s)` devolve `[ { label, value } ]` na ordem "Aceitos" (`accepted`, Q11), "Na fila", "Parados", "Recusados", "Com falha", "Atrasados".
  - **Atalhos:** `fixActions(item: RegulationRndsSync, roles): FixAction[]`, com `type FixAction = { label: string; target: FixTarget } | { text: string }` e `type FixTarget = { screen: "regulation-offers"; offerId?: string; providerId?: string } | { screen: "regulation-queue"; requestId: string } | { screen: "regulation-settings" | "cnes" | "integrations" }`. O `fix` do api (Q11) traz `offer_id`, `provider_id`, `unit_id` e `settings`, sem o mês da oferta. Por motivo (rótulos proposta):
    - `performer_cbo_missing` (só procedimento dos grupos 03 e 04, porque na especialidade o CBO do alvo conta primeiro, Q12): admin → "Abrir a oferta" (com `fix.offer_id`) e "Abrir o prestador" (com `fix.provider_id`); regulador → `{ text: "o administrador da cidade informa o CBO na oferta ou no prestador" }`;
    - `performer_without_cnes`: admin → "Abrir o prestador" e, com `fix.unit_id`, "Abrir CNES"; regulador → texto;
    - `requester_unit_without_cnes`: admin → "Abrir CNES"; regulador → texto;
    - `specialty_without_sigtap` (ou `fix.settings`): admin → "Abrir Prazos e lista do enfermeiro"; regulador → texto;
    - `missing_diagnosis`: regulador → "Abrir o pedido" (`regulation-queue`, `regulation_request_id`); admin → `{ text: "a regulação devolve o pedido à unidade para completar" }`;
    - `patient_without_cpf_or_cns` e `skeleton:*`: só a frase, sem atalho;
    - recusado (`last_error_codes`): `EHR-ERR906` ou `EHR-ERR881` → admin "Abrir Integrações"; os demais → regulador "Abrir o pedido".
  - **Link do pedido** (Q11): `requestLink(item, roles): FixTarget | null` → `{ screen: "regulation-queue", requestId }` para o regulador; `null` para quem não abre a fila (o admin vê o `request_ref` como texto).
  - **Formato:** `digitsOf(raw): string` tira espaço, `.`, `-` e `/`; `cboOf(raw): string | null` (6 dígitos depois de `digitsOf`); `sigtapOf(raw): string | null` (10); `fmtSigtap("0301010072")` = `"03.01.01.007-2"`; `requesterIdProblem(raw)` (1–20 dígitos, Q13) e `responsibleCnsProblem(raw)` (`isValidCns` depois de `digitsOf`): texto ou `null`.
  - **Configurações:**
    - `DEFAULT_SIGTAP = "0301010072"` e `DEFAULT_SIGTAP_TEXT = "As demais especialidades vão como 03.01.01.007-2, consulta médica em atenção especializada. Se um código deixar de ser regulável ou vigente, os pedidos da especialidade param, com o motivo na lista."` (proposta; Q9);
    - `SEND_EXTERNAL_LABEL = "Enviar também os pedidos regulados pelo estado"` e `SEND_EXTERNAL_WARNING = "Ligue só se a cidade pactuou com o estado ser a responsável pelo envio desses pedidos."` (contrato §6);
    - `type RndsSettingsDraft = { priority: Record<RiskColor, MiraPriority>; rows: { key: string; cbo: string; name: string | null; sigtap: string }[]; sendExternal: boolean }`, `draftOf(s)`;
    - `settingsProblems(d): { field: string; text: string }[]`: CBO e SIGTAP de cada linha pelo formato; CBO repetido → "esta especialidade já está na lista"; caráter fora de `urgent`/`routine`. Se o SIGTAP é regulável e vigente, só o api sabe (Q9): a recusa volta com `invalid_sigtap`;
    - `settingsFieldOf(err): string | null`: o campo da recusa, pelo `path` (`specialty_sigtap.<CBO>`, `priority_by_color.<cor>`), tanto para os códigos da Q13 quanto para `invalid_request` com `path`;
    - `settingsBody(d, version): RegulationRndsSettingsInput`, com os códigos limpos.
  - **Recusas:** `regulationRndsError(err: unknown): string` (`TypeError` → "sem conexão com o servidor — tente de novo"; `featureDisabledKey` → `REGULATION_RNDS_DISABLED`; código → `regulationRndsErrorText`; o resto → `regulationError` do 26a); `isStale(err, code): boolean`.

- [ ] **Step 1: Testes que falham.**
  - `src/lib/regulationRndsErrors.test.ts`: cada código da tabela tem frase; as frases são distintas; nenhuma tem `_`; `skeleton:Appointment.start` cita o campo; `EHR-ERR999` volta como `"recusa da RNDS: EHR-ERR999"`.
  - `src/lib/regulationRnds.test.ts`:

```ts
it.each([
  [ "2251-20", "225120" ], [ " 225120 ", "225120" ], [ "22512", null ], [ "2251201", null ], [ "22A120", null ]
])("cboOf(%j) = %j", (raw, code) => expect(cboOf(raw)).toBe(code));
it.each([
  [ "03.01.01.007-2", "0301010072" ], [ "0301010072 ", "0301010072" ], [ "030101007", null ], [ "03010100721", null ]
])("sigtapOf(%j) = %j", (raw, code) => expect(sigtapOf(raw)).toBe(code));
it("CNS do responsável e identificador do solicitante", () => {
  expect(responsibleCnsProblem("898 0011 6066 0761")).toBeNull(); expect(responsibleCnsProblem("898001160660762")).not.toBeNull();
  expect(requesterIdProblem("12345")).toBeNull(); expect(requesterIdProblem("BRRNDS-12345")).not.toBeNull();
});
it("faixa: envio parado pelo missing, atraso no singular e carga inicial", () => {
  const s = pausedStatus({ counts: { queued: 0, blocked: 0, rejected: 0, failed: 0, late: 1, accepted: 0 }, late_banner: true, initial_load_pending: 120 });
  expect(alertLines(s)).toEqual([
    `O envio à RNDS está parado. Falta: ${missingPhrase("credential_unauthorized:rnds_regulation")}.`,
    "1 pedido mudado até ontem ainda sem aceite da RNDS. O envio precisa ser aceito até as 23h de hoje.",
    "Carga inicial: 120 pedidos na fila, no ritmo de 60 por minuto. Eles ficam fora do atraso nas primeiras 72 horas."
  ]);
  expect(alertLines(rndsStatus())).toEqual([]);
});
it("atalhos por motivo e papel; paciente sem identificação nunca tem atalho", () => {
  const admin = { admin: true, regulator: false, reader: true }; const reg = { admin: false, regulator: true, reader: true };
  const cbo = blockedSync([ "performer_cbo_missing" ], { fix: { offer_id: "of1", provider_id: "pr2", unit_id: null, settings: false } });
  expect(fixActions(cbo, admin).map((a) => "label" in a ? a.label : a.text)).toEqual([ "Abrir a oferta", "Abrir o prestador" ]);
  expect(fixActions(cbo, reg).every((a) => "text" in a)).toBe(true);
  expect(fixActions(blockedSync([ "missing_diagnosis" ]), reg)[0]).toEqual({ label: "Abrir o pedido", target: { screen: "regulation-queue", requestId: "rr1" } });
  expect(fixActions(blockedSync([ "patient_without_cpf_or_cns" ]), admin).some((a) => "label" in a)).toBe(false);
  expect(requestLink(rndsSync(), reg)).toEqual({ screen: "regulation-queue", requestId: "rr1" }); expect(requestLink(rndsSync(), admin)).toBeNull();
});
it("recusa das configurações cai no campo, pelo código da Q13 ou pelo invalid_request", () => {
  expect(settingsFieldOf(new ApiError(422, { error: "invalid_sigtap", path: "specialty_sigtap.225120" }, ""))).toBe("specialty_sigtap.225120");
  expect(settingsFieldOf(new ApiError(422, { error: "invalid_request", path: "priority_by_color.blue" }, ""))).toBe("priority_by_color.blue");
});
it("configurações: código limpo, repetido recusado, corpo com a versão", () => {
  const d = { ...draftOf(rndsSettings()), rows: [ { key: "a", cbo: "2251-20", name: "Cardiologia", sigtap: "03.01.01.007-2" }, { key: "b", cbo: "225120", name: null, sigtap: "0301010072" } ] };
  expect(settingsProblems(d).map((p) => p.text)).toEqual([ "esta especialidade já está na lista" ]);
  const { version: _v, ...rest } = rndsSettings();
  expect(settingsBody({ ...d, rows: d.rows.slice(0, 1) }, 3)).toEqual({ ...rest, specialty_sigtap: { "225120": "0301010072" }, expected_version: 3 });
});
```

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod26c src/lib/regulationRndsErrors.test.ts src/lib/regulationRnds.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/regulationRndsErrors.ts src/lib/regulationRndsErrors.test.ts src/lib/regulationRnds.ts src/lib/regulationRnds.test.ts
$G commit -m "feat: add the regulation RNDS phrases, labels, alert lines, fix shortcuts and format rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Item do menu, portão e atalho até o pedido

**Files:**
- Create: `src/shell/modules.regulationRnds.test.ts`, `src/modules/regulation/useRegulationHandoff.ts`, `src/modules/RegulationRnds.tsx` (casca: portão e título; o corpo vem na Task 7), `src/modules/RegulationQueue.handoff.test.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/RegulationQueue.tsx`

**Interfaces:**
- Consumes: `canUseRegulationRnds`, `rndsRoles`, `REGULATION_RNDS_DISABLED`, `FixTarget` (Task 2); `RegulationGate`.
- Produces:
  - `ModuleId` + `"regulation-rnds"`; no grupo **Regulação**, depois de `regulation-settings`: `{ id: "regulation-rnds", label: "Envio à RNDS", icon: "⇪" }`.
  - Regra do menu: `regulation-rnds` → `regOn && hasFeature(user, "rnds_regulation") && (isAdmin || isCareRegulator)`.
  - `CATALOG.screens["regulation-rnds"]` (`IconSend`, ou o ícone da Task 0): "Pedidos aceitos, parados e recusados pela RNDS, com o motivo" (proposta).
  - `useRegulationHandoff(active): RegulationHandoff`, no molde do `useOmbudsmanHandoff` (`src/App.tsx`): `type RegulationHandoff = { target: FixTarget | null; open(t: FixTarget): void }`. Só ids e o mês, só em memória; limpa quando a tela ativa não é a do alvo.
  - `renderModule(…, reg: RegulationHandoff = NO_REG_HANDOFF)`:
    - `case "regulation-rnds": return <RegulationRnds onGoTo={(t) => { reg.open(t); setActive(t.screen); }} onGoToSecurity={() => setActive("security")} />;`
    - `case "regulation-queue": return <RegulationQueue openRequestId={reg.target?.screen === "regulation-queue" ? reg.target.requestId : null} />;`
    - `case "regulation-offers": return <RegulationOffers focus={reg.target?.screen === "regulation-offers" ? reg.target : null} onGoToSecurity={…} />;` (o `focus` é da Task 4; até lá a prop é aceita e ignorada).
  - `RegulationQueue({ today, openRequestId })`: com `openRequestId`, abre o `RequestDetail` daquele id na entrada, como se a linha tivesse sido clicada. "Voltar" cai na aba "Aguardando".
  - `RegulationRnds({ now, onGoTo, onGoToSecurity })`: o `RegulationGate title="Envio à RNDS" sub="pedidos aceitos, parados e recusados pela RNDS" allow={(r) => r.admin || r.regulator}`. Sem `rnds_regulation`, mostra `EmptyState` com `REGULATION_RNDS_DISABLED`, sem leitura.

- [ ] **Step 1: Testes que falham.**
  - `src/shell/modules.regulationRnds.test.ts`:
    - com `rnds_regulation`, o regulador vê `[ "regulation-queue", "regulation-offers", "regulation-rnds" ]` e o admin `[ "regulation-offers", "regulation-settings", "regulation-rnds" ]`;
    - sem `rnds_regulation`, a lista do 26a não muda;
    - o profissional e o operador não veem;
    - o rótulo "Envio à RNDS" aparece uma vez só em `NAV_GROUPS`;
    - o `CATALOG` tem a descrição.
  - `src/modules/RegulationQueue.handoff.test.tsx`: com `openRequestId: "rr1"`, `getRegulationRequest("rr1")` é lido uma vez e o detalhe aparece; sem ele, nenhuma leitura de detalhe.
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos e `src/shell/modules.regulation.test.ts src/shell/catalog.test.ts src/modules/RegulationQueue.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.regulationRnds.test.ts src/shell/catalog.ts src/App.tsx src/modules/regulation/useRegulationHandoff.ts src/modules/RegulationRnds.tsx src/modules/RegulationQueue.tsx src/modules/RegulationQueue.handoff.test.tsx
$G commit -m "feat: add the regulation RNDS screen to the regulation group with a shortcut to the request" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: "CBO do executante" na oferta e no prestador

**Files:**
- Create: `src/modules/RegulationOffers.cbo.test.tsx`
- Modify: `src/modules/regulation/OfferForm.tsx`, `src/modules/regulation/ProviderForm.tsx`, `src/modules/RegulationOffers.tsx`, `src/modules/RegulationOffers.test.tsx` (o botão renomeado)

**Interfaces:**
- Consumes: `cboOf` (Task 2); `CareOffer.performer_cbo`, `CareProvider.default_performer_cbo`, `updateCareOffer(id, { quantity?, performer_cbo? })` e `createCareOffer` com `performer_cbo` (Task 1); `FixTarget` (Task 2).
- Produces:
  - `OfferForm`:
    - campo "CBO do executante" (opcional), **só em oferta de procedimento**: na especialidade, o CBO do próprio alvo conta primeiro (Q12), e o campo não aparece;
    - ajuda (proposta): "6 dígitos. Exigido no envio à RNDS para procedimento dos grupos 03 e 04. Em branco numa oferta nova, vale o CBO da oferta do mês anterior do mesmo prestador e alvo; sem nenhum, o CBO padrão do prestador.";
    - recusa antes da confirmação pelo `cboOf`, com "o CBO tem 6 dígitos";
    - na criação, vai `performer_cbo` só se preenchido; em branco, o api herda o do mês anterior (Q12), e a frase de sucesso mostra o CBO que a resposta trouxer;
    - na edição, vão só as chaves que mudaram: `{ performer_cbo }`, `{ quantity }` ou as duas (Q12: o PATCH aceita só o CBO). O CBO apagado vai `null`. "nada mudou" considera a quantidade **e** o CBO;
    - o botão da linha "Mudar quantidade" vira "Editar oferta" (proposta), e o título da seção "Editar a oferta".
  - `ProviderForm`: campo "CBO do executante" (opcional, todos os tipos), com a ajuda (proposta) "CBO usado quando nem a especialidade do pedido, nem o profissional da agenda, nem a oferta trazem um." (a ordem da Q12); corpo com `default_performer_cbo` (`null` quando vazio).
  - `RegulationOffers({ today, focus, onGoToSecurity })`:
    - colunas novas "CBO do executante" nas duas tabelas (`—` quando nulo; "do alvo" na oferta de especialidade, proposta);
    - `focus` (`{ offerId?, providerId? }`): com `canWrite`, abre o editor da oferta ou do prestador do id quando a leitura chega. O `fix` do api não traz o mês (Q11), então a busca é no mês mostrado (o corrente); oferta não achada → `p role="status"` "a oferta do atalho é de outro mês — escolha o mês em Ver o mês, ou corrija o CBO padrão do prestador" (proposta), e o editor do prestador abre se o `providerId` veio.

- [ ] **Step 1: Testes que falham** (`RegulationOffers.cbo.test.tsx`, molde `RegulationOffers.test.tsx`, sessão admin; `of3` = `offer({ id: "of3", target: ECO, quantity: 6, performer_cbo: null })`):
  - criar oferta de procedimento (`ECO`) com CBO `"2251-20"` → `createCareOffer` com `performer_cbo: "225120"`; com `"22512"` → a frase e nenhuma chamada;
  - oferta nova de procedimento com o CBO em branco → corpo sem `performer_cbo`; oferta de especialidade (`CARDIO`) → o campo não aparece;
  - editar só o CBO → `updateCareOffer("of3", { performer_cbo: "225120" })`, sem `quantity`; apagar → `{ performer_cbo: null }`; mudar só a quantidade → `{ quantity: 7 }`;
  - prestador com CBO → `updateCareProvider` com `default_performer_cbo`;
  - `focus: { offerId: "of3" }` abre "Editar a oferta" da `of3`; `focus: { offerId: "ofX", providerId: "pr2" }` mostra a frase do outro mês e abre o editor do prestador `pr2`;
  - o regulador vê a coluna "CBO do executante" e não vê "Editar oferta".

  Em `RegulationOffers.test.tsx`, o nome do botão "Mudar quantidade" vira "Editar oferta", e a chamada esperada vira `("of1", { quantity: 5 })`.
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/regulation/OfferForm.tsx src/modules/regulation/ProviderForm.tsx src/modules/RegulationOffers.tsx src/modules/RegulationOffers.cbo.test.tsx src/modules/RegulationOffers.test.tsx
$G commit -m "feat: edit the performer CBO on care offers and providers and open them from a shortcut" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Seção "Envio à RNDS" em "Prazos e lista do enfermeiro"

**Files:**
- Create: `src/modules/regulation/RndsSettingsPanel.tsx`, `src/modules/RegulationSettings.rnds.test.tsx`
- Modify: `src/modules/RegulationSettings.tsx` (um painel a mais, depois do da lista do enfermeiro)

**Interfaces:**
- Consumes: `getRegulationRndsSettings`, `updateRegulationRndsSettings` (Task 1); `draftOf`, `settingsProblems`, `settingsBody`, `fmtSigtap`, `priorityLabel`, `DEFAULT_SIGTAP_TEXT`, `SEND_EXTERNAL_LABEL`, `SEND_EXTERNAL_WARNING`, `RNDS_SETTINGS_KEY`, `regulationRndsError`, `isStale` (Task 2); `COLORS`, `colorLabel`; `TargetSearch` (tipo `specialty`).
- Produces: `RndsSettingsPanel({ onSaved, onGoToSecurity })`, `Panel title="Envio à RNDS" sub="como os pedidos vão ao padrão do Ministério"` (sub proposta):
  - **Caráter por cor:** um `select` por cor ("Vermelho", "Amarelo", "Verde", "Azul"), com "urgência" e "eletivo".
  - **SIGTAP por especialidade:**
    - tabela "Especialidade" (o nome, quando conhecido, e o CBO), "Código SIGTAP" (campo, mostrado por `fmtSigtap`) e "Retirar";
    - "Acrescentar especialidade" abre o `TargetSearch` de especialidade; escolher acrescenta a linha com o CBO e o nome, e o SIGTAP começa com o padrão;
    - abaixo, `DEFAULT_SIGTAP_TEXT`.
  - **Externos:** a caixa `SEND_EXTERNAL_LABEL`, com o `SEND_EXTERNAL_WARNING` sempre visível ao lado (`role="note"`).
  - Nota de topo (proposta): "Estas configurações valem quando o envio da regulação à RNDS estiver ligado. Mudar uma delas reenvia os pedidos já aceitos que ela afeta."
  - "Salvar envio à RNDS" (proposta): `settingsProblems` antes; depois o `SensitiveAction` (`requiresStepUp`, Q13). A descrição repete as conversões e, se `send_external` mudou para ligado, o aviso. → `updateRegulationRndsSettings(settingsBody(draft, version))`.
  - `409 rnds_settings_changed` → a frase, **uma** releitura, o digitado mantido e a `version` nova no próximo envio.
  - Recusa com `path` cai no campo, por `settingsFieldOf`: `invalid_sigtap` em `specialty_sigtap.<CBO>` (SIGTAP fora do formato, não regulável ou não vigente, Q9) na linha daquele CBO; `invalid_priority` na cor. `expected_version_required` mostra a frase.
  - Sucesso: "Configurações do envio à RNDS salvas." (proposta) e releitura.
  - A seção não depende de `rnds_regulation` (§5).

- [ ] **Step 1: Testes que falham** (`RegulationSettings.rnds.test.tsx`, sessão admin):
  - abre com os quatro caracteres e a linha `225120`, e o SIGTAP mostrado como `03.01.01.007-2`;
  - trocar amarelo para "urgência", acrescentar "Endodontia" (`ENDO`) com `"03.01.01.007-2"` e ligar os externos: o `SensitiveAction` mostra o aviso, e o corpo é `{ priority_by_color: { red: "urgent", yellow: "urgent", green: "routine", blue: "routine" }, specialty_sigtap: { "225120": "0301010072", "223212": "0301010072" }, send_external: true, expected_version: 1 }`;
  - o caso do Review Focus 2, "configurações: 409 relê uma vez, mantém o digitado e reenvia com a versão nova";
  - SIGTAP com 9 dígitos → a frase na linha e nenhuma chamada;
  - `422 invalid_sigtap` com `path: "specialty_sigtap.223212"` (código de grupo 01) → a frase na linha da Endodontia, e o digitado fica;
  - o duplo clique em "Confirmar" manda um pedido;
  - sem `rnds_regulation` na sessão, a seção aparece igual.
- [ ] **Step 2–4:** rode e veja falhar (o arquivo e `src/modules/RegulationSettings.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/regulation/RndsSettingsPanel.tsx src/modules/RegulationSettings.tsx src/modules/RegulationSettings.rnds.test.tsx
$G commit -m "feat: configure the color to priority mapping, specialty SIGTAP codes and state requests for the RNDS" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Credencial `rnds_regulation` em Integrações

**Files:**
- Create: `src/modules/integrations/RndsRegulationCredentialForm.tsx`, `src/modules/Integrations.rndsRegulation.test.tsx`
- Modify: `src/lib/integrations.ts`, `src/lib/integrations.test.ts`, `src/modules/Integrations.tsx`

**Interfaces:**
- Consumes: `setRndsRegulationCredential` (Task 1); `requesterIdProblem`, `responsibleCnsProblem`, `digitsOf` (Task 2); `pfxProblem`, `cnesProblem`, `rndsPasswordProblem`, `RNDS_INVALID_CREDENTIAL` (`src/lib/integrations.ts`); `SensitiveAction`. Molde: `RndsCredentialForm.tsx`.
- Produces:
  - `src/lib/integrations.ts`:
    - `CREDENTIAL_LABEL.rnds_regulation = "RNDS — regulação (certificado A1)"` (proposta);
    - `checkLabel("unauthorized", "rnds_regulation")` = "certificado recusado", como a `rnds`;
    - `missingPhrase`:
      - `credential_unauthorized:rnds_regulation` → "a RNDS recusou o certificado da regulação — troque o certificado A1 e teste de novo" (proposta);
      - `rnds_sending_disabled` → "ligue antes um dos envios à RNDS (vacina ou regulação)" (Q14, letra por letra);
      - `uf_missing` → "falta a UF da cidade (quem cadastra é o operador da plataforma)" (proposta; hoje sai como "pendência não reconhecida");
      - `credential_missing:rnds_regulation` e `care_regulation_disabled` saem pelas regras genéricas, com o rótulo novo;
    - `MESSAGES` + `invalid_requester_id` e `invalid_responsible_cns`, com as frases de `REGULATION_RNDS_ERRORS` (Q13).
  - `RndsRegulationCredentialForm({ credential, onDone, onCancel, onGoToSecurity })`:
    - passo 1: "Certificado A1" (arquivo), "CNES solicitante", "Identificador do solicitante (DATASUS)" e "CNS do responsável pelo envio", conferidos por `pfxProblem`, `cnesProblem`, `requesterIdProblem` e `responsibleCnsProblem`, cada frase abaixo do campo;
    - nota (proposta): "Pode ser o mesmo certificado da vacina. O identificador e o CNS vêm do credenciamento da regulação no Portal de Serviços do DATASUS.";
    - passo 2: a senha no `SensitiveAction` (`requiresStepUp`) → `setRndsRegulationCredential(file, password, digitsOf(cnes), digitsOf(requesterId), digitsOf(cns))`;
    - sucesso: "Certificado da regulação salvo. Teste a conexão para conferir a troca de token." (proposta);
    - fechar descarta o arquivo e os campos.
  - `Integrations.tsx`: a linha `rnds_regulation` abre o `RndsRegulationCredentialForm`, e o formulário de usuário e senha deixa de valer para ela. O teste de conexão é o genérico.

- [ ] **Step 1: Testes que falham** (`Integrations.rndsRegulation.test.tsx`, molde `Integrations.rnds.test.tsx`):
  - a linha "RNDS — regulação (certificado A1)" aparece com "Cadastrar";
  - os quatro campos recusam antes do step-up: `.cer`, CNES com 6 dígitos, `"BRRNDS-12345"` e CNS com dígito errado;
  - com tudo certo, o step-up pede a senha, e a chamada é `("…", " s3nh@ ", "2384310", "12345", "898001160660761")`, com o CNS digitado com espaços;
  - nenhuma chave do `queryClient` tem a senha, o identificador ou o CNS;
  - "Testar conexão" chama `checkIntegrationCredential("rnds_regulation")`;
  - os `missing` `credential_missing:rnds_regulation`, `rnds_sending_disabled` e `uf_missing` aparecem com as frases;
  - `422 invalid_requester_id` do api (formato que a tela aceitou) aparece no step-up com a frase.

  `src/lib/integrations.test.ts`: `credentialLabel("rnds_regulation")`, `checkLabel("unauthorized", "rnds_regulation")` e `missingPhrase("care_regulation_disabled")` = `"ligue antes: Regulação municipal"`.
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos e `src/modules/Integrations.test.tsx src/modules/Integrations.rnds.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/integrations.ts src/lib/integrations.test.ts src/modules/Integrations.tsx src/modules/integrations/RndsRegulationCredentialForm.tsx src/modules/Integrations.rndsRegulation.test.tsx
$G commit -m "feat: set and test the RNDS regulation certificate with the requester id and the responsible CNS" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Tela "Envio à RNDS"

**Files:**
- Create: `src/modules/regulation/RndsAlertBand.tsx`, `src/modules/regulation/RndsFixCell.tsx`, `src/modules/regulation/RndsSettingsSummary.tsx`, `src/modules/RegulationRnds.test.tsx`
- Modify: `src/modules/RegulationRnds.tsx`

**Interfaces:**
- Consumes: a casca e `onGoTo` (Task 3); `getRegulationRndsStatus`, `listRegulationRndsSyncs`, `retryRegulationRndsSync` (Task 1); `alertLines`, `totals`, `fixActions`, `requestLink`, os rótulos, `fmtSigtap`, `priorityLabel`, `regulationRndsErrorText`, `regulationRndsError`, `isStale`, as chaves (Task 2); `getRegulationRndsSettings` (Task 1); `COLORS`, `colorLabel` do 26a. Molde: `VaccinationPending.tsx`.
- Produces: `RegulationRnds({ now, onGoTo, onGoToSecurity })`, dentro do portão:
  - `RndsAlertBand({ status })`: `role="alert"`, uma linha por `alertLines`; nada sem alerta. É a única faixa do 26c no dashboard (Q13): nem a fila nem o menu ganham faixa ou contagem.
  - Os totais em `KeyValue` (rótulos de `totals`).
  - A nota (proposta): "A lista não mostra o paciente. Corrija o dado de origem pelo atalho: o pedido parado volta à fila sozinho na verificação de hora em hora, e o recusado volta quando o dado muda."
  - Filtros:
    - "Situação" (`SegmentedControl`): "Todos", "Parados" (`blocked`), "Recusados" (`rejected`), "Com falha" (`failed`), "Na fila" (`queued`);
    - "Motivo" (`select`, só com "Parados"): os seis motivos por `regulationRndsErrorText`;
    - os filtros vão à URL do api (`status`, `reason`).
  - A lista (`useInfiniteQuery`, "Carregar mais", `staleTime: 0`, `refetchOnMount: "always"`), com as colunas:
    - "Pedido": o `request_ref` (Q11). Para o regulador, é um botão-link "Pedido <request_ref>" → `onGoTo(requestLink(...))`, que abre o detalhe na fila; para o admin, texto. A lista não traz alvo nem unidade (forma do api, Q11);
    - "Envio": a `Tag` de `syncStatusLabel`/`Tone`, a `Tag` "atrasado" (tom `down`, proposta) quando `late`, e abaixo a situação desejada e a enviada por `miraStatusLabel`;
    - "Motivo": os `blocked_reasons` e os `last_error_codes` por `regulationRndsErrorText`, um por linha; `failed` mostra também as tentativas;
    - "Datas": "mudou em" (`enqueued_at`) e "último aceite" (`last_accepted_at`, ou "nunca aceito");
    - "Ação": o `RndsFixCell` e, para o admin em `rejected` ou `failed`, "Reenviar — <request_ref>".
  - `RndsFixCell({ item, roles, onGoTo })`: um botão por `FixAction` com `target` (`onGoTo(target)`), e o texto de cada `FixAction` sem alvo.
  - "Reenviar": `SensitiveAction` (`requiresStepUp`; título "Reenviar o pedido <request_ref>"; descrição proposta: "Reenvia o mesmo documento, sem mudar o conteúdo. Use quando a recusa foi passageira. Se o dado estava errado, corrija a origem: o pedido volta sozinho.") → `retryRegulationRndsSync(id)`. Depois, relê a lista e o status. `409 not_retryable` → a frase e uma releitura. Com `usable: false`, o botão aparece desabilitado com a nota "o envio está parado — resolva o que falta na faixa acima" (proposta); se mesmo assim vier `403 feature_disabled` (Q10), a frase do interruptor.
  - `RndsSettingsSummary()` (Q13), só para o regulador (o admin vê o atalho "Abrir Prazos e lista do enfermeiro"): `Panel title="Como os pedidos vão à RNDS"` (proposta), só leitura, lido de `getRegulationRndsSettings` (`STATIC_QUERY`):
    - "Caráter por cor": as quatro cores com `priorityLabel`;
    - "SIGTAP por especialidade": o CBO e o código por `fmtSigtap`, mais o `DEFAULT_SIGTAP_TEXT`;
    - "Pedidos regulados pelo estado": "enviados" ou "não enviados" (proposta);
    - a nota "Quem muda é o administrador da cidade, em Prazos e lista do enfermeiro." (proposta).
  - O "Atualizar" relê o status, a lista e o resumo.

- [ ] **Step 1: Testes que falham** (`RegulationRnds.test.tsx`):

```ts
it("faixa, totais e lista sem paciente, só leitura para o regulador", async () => { /*
  rndsUser([ "care_regulator" ]); lateStatus(); três linhas (idle, blocked missing_diagnosis, rejected EHR-ERR906 com late: true);
  role=alert com a linha do atraso; "Aceitos" 40 e "Parados" 2; a Tag "atrasado" na terceira linha;
  nenhuma ocorrência de "Joana", CPF, CNS ou "I209" no body; sem "Reenviar" */ });
it("o regulador abre o pedido pelo request_ref e vê o resumo das configurações; o admin não", async () => { /*
  regulador: "Pedido 4F9A1C2E" → onGoTo({ screen: "regulation-queue", requestId: "rr1" }); o resumo com "urgência" no vermelho e 03.01.01.007-2;
  admin: "4F9A1C2E" é texto, sem resumo, com "Abrir Prazos e lista do enfermeiro" */ });
it("credencial recusada: a tela abre, a faixa diz o que falta e o reenvio fica parado", async () => { /*
  pausedStatus(); a linha "O envio à RNDS está parado. Falta: …"; "Reenviar" desabilitado; 403 feature_disabled no reenvio → a frase */ });
it("filtro: situação e motivo vão à URL, e Todos limpa", async () => { /* listRegulationRndsSyncs({ status: "blocked", reason: "performer_cbo_missing", before: null }) */ });
it("reenviar: step-up, um pedido, estado mudado relê, só admin e só recusado ou falho", async () => { /*
  admin; linha blocked sem botão; duplo clique → 1 chamada; 401 mfa_required volta ao step-up sem nova chamada;
  409 not_retryable → frase e listRegulationRndsSyncs relido uma vez */ });
it("atalhos por motivo e por papel, sem paciente", async () => { /*
  admin: "Abrir a oferta" → onGoTo({ screen: "regulation-offers", offerId: "of3" }); com fix.unit_id, "Abrir CNES" → onGoTo({ screen: "cnes" });
  regulador: "Abrir o pedido" → onGoTo({ screen: "regulation-queue", requestId: "rr1" }); nenhum "Abrir CNES";
  patient_without_cpf_or_cns: só a frase */ });
it("motivo, recusa e situação desconhecidos aparecem crus", async () => { /* "skeleton:Appointment.start", "EHR-ERR999", status "paused_by_platform" */ });
it("sem rnds_regulation ou com 403 feature_disabled: a frase do interruptor", async () => { /* sem leitura no primeiro; frase no segundo */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/RegulationRnds.tsx src/modules/RegulationRnds.test.tsx src/modules/regulation/RndsAlertBand.tsx src/modules/regulation/RndsFixCell.tsx src/modules/regulation/RndsSettingsSummary.tsx
$G commit -m "feat: show the regulation RNDS status with the alert band, the reasons, the fix shortcuts, retry and the settings summary" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Suíte, build, varreduras, revisão e prova no navegador (api na 3063, Vite na 5224)

**Files:** `$R/.claude/launch.json` (entrada nova; fora de git). Correção que a prova pedir entra na task dona, com teste.

- [ ] **Step 1: Suíte, tipos e build**

```bash
$R/bin/test dashboard -w dashboard-mod26c
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde e `build-ok`. A contagem é a base da Task 0 + **9 arquivos de teste novos**:
- `src/lib/api.regulationRnds.test.ts`, `src/lib/regulationRndsErrors.test.ts`, `src/lib/regulationRnds.test.ts`;
- `src/shell/modules.regulationRnds.test.ts`;
- `src/modules/RegulationQueue.handoff.test.tsx`, `src/modules/RegulationOffers.cbo.test.tsx`, `src/modules/RegulationSettings.rnds.test.tsx`;
- `src/modules/Integrations.rndsRegulation.test.tsx`, `src/modules/RegulationRnds.test.tsx`;
- e os casos novos em `features.test.ts`, `integrations.test.ts`, `api.regulation.test.ts` e `RegulationOffers.test.tsx` (sem arquivo novo).

Contagem que dobra = artefato de build achado pelo vitest. Erro não tratado com tudo verde = mock sem `fetchCurrentSession`.

- [ ] **Step 2: Varreduras**

```bash
cd $W
O="src/modules/RegulationRnds.tsx src/modules/regulation/RndsAlertBand.tsx src/modules/regulation/RndsFixCell.tsx src/modules/regulation/RndsSettingsSummary.tsx src/modules/regulation/RndsSettingsPanel.tsx src/modules/regulation/useRegulationHandoff.ts src/modules/integrations/RndsRegulationCredentialForm.tsx src/lib/regulationRnds.ts src/lib/regulationRndsErrors.ts"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
grep -rnE "queryKey:.*(password|requester_id|responsible_cns|pfx)" $O && echo "ERRO: segredo no cache" || echo ok-cache
grep -rnE "display_name|patient\.|cpf|cns_masked|cid10|justification" src/modules/RegulationRnds.tsx src/modules/regulation/RndsFixCell.tsx && echo "ERRO: paciente na tela de envio" || echo ok-paciente
sed -n '/Envio da regulação à RNDS (módulo 26c/,/^\/\/ ─── Farmácia/p' src/lib/api.ts | grep -nE "pfx|password|requester_id|responsible_cns" | grep -E "withQuery|URLSearchParams|\?\$\{" && echo "revisar: URL" || echo ok-url
for f in src/modules/RegulationRnds.test.tsx src/modules/RegulationOffers.cbo.test.tsx src/modules/RegulationSettings.rnds.test.tsx src/modules/Integrations.rndsRegulation.test.tsx src/modules/RegulationQueue.handoff.test.tsx; do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-cache`, `ok-paciente`, `ok-url`, `ok-mocks`, sem `ERRO`.

- [ ] **Step 3: Conferência do api.** Repita a Task 0 Step 4 contra o api final. Forma diferente vira ajuste na task dona antes da prova.

- [ ] **Step 4: Revisão do ramo** (superpowers:requesting-code-review). Foco: as cinco linhas do Review Focus; a tabela `REGULATION_RNDS_ERRORS`; nenhuma coluna de paciente; o step-up das quatro escritas; o segredo fora de URL e de cache; as frases fixas letra por letra.

- [ ] **Step 5: Entrada do Vite.** Acrescente ao `$R/.claude/launch.json`, depois da última entrada `dashboard-mod*`:

```json
{
  "name": "dashboard-mod26c",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/dashboard-mod26c && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3063 npx vite --port 5224 --host 0.0.0.0" ],
  "port": 5224
}
```

- [ ] **Step 6: Prova no navegador.** Antes de começar:
  - **Autorização do usuário:** migrar os bancos de dev (as migrações de cidade do 26c, seguintes às do 27b) e rodar a semente do 26c em Curitiba (plano do api) só com autorização dele. Peça antes de qualquer `db:migrate` ou `db:seed`.
  - **Dono de cada serviço:** o api do worktree `apps/api/.claude/mod26c` na **3063** e o `fake_rnds` na **8094** são da sessão do api; o maintenance na 5177 é da sessão dele.
  - **Abrir:** `curitiba.localhost:5224/dashboard/` (`preview_start` com `dashboard-mod26c`).
  - **Senhas:** a senha e o TOTP de dev podem ser mostrados; os de staging, nunca.

  Roteiro, um passo por linha:

1. **Interruptores** (maintenance 5177): `care_regulation`, `rnds_regulation` e `rnds_mock` ligados. Antes da credencial, "Envio à RNDS" mostra a frase do interruptor.
2. **Credencial** (`municipal_admin`): em Integrações, cadastrar a `rnds_regulation` com step-up e "Testar conexão" contra o simulado. Na aba de rede, a senha, o identificador e o CNS só aparecem no corpo do `PUT`.
3. **Configurações**: em "Prazos e lista do enfermeiro", trocar uma cor, acrescentar uma especialidade e ver o aviso dos externos; salvar com step-up; numa segunda aba, salvar de novo e ver o 409 relido.
4. **CBO**: numa oferta de procedimento do grupo 03, o "CBO do executante", mudado sozinho (PATCH só com o CBO); uma oferta nova do mês seguinte, em branco, herda o CBO; num prestador, o CBO padrão; numa oferta de especialidade, o campo não aparece.
5. **Envio** (a sessão do api roda o `RndsDeliverJob` contra o simulado, com os marcadores da semente). "Envio à RNDS" mostra:
   - aceitos e a carga inicial na faixa;
   - um pedido parado por `performer_cbo_missing`, cujo atalho abre a oferta; depois de salvar o CBO e da verificação de parados, ele sai da lista;
   - um pedido parado por `missing_diagnosis` que, pelo regulador, abre o pedido na fila pelo `request_ref`;
   - o resumo das configurações na tela do regulador;
   - com a credencial recusada por um momento (a sessão do api marca `unauthorized`), a tela abre e a faixa mostra o que falta;
   - um recusado (`RECUSA`) com "Reenviar" e step-up.
6. **Sigilo**: nenhuma coluna de pessoa; na aba de rede, a URL da lista só tem `status`, `reason` e `before`.

  Também:
  - preset mobile 375×812 nos passos 3 e 5;
  - só teclado nos passos 2 e 3;
  - `resize_window` com o preset desktop ao terminar.

  Anote cada passo. Achado que muda comportamento vira teste na task dona antes da correção. Achado só visual vai ao usuário (memória "Interface em ciclo próprio").

---

### Task 9: Entrega — board, relato e parada antes do merge

**Files:** nenhum.

- [ ] **Step 1: Board.** Os cards do 26c que o dashboard toca vão a **Done** só se o api já terminou o deles, e a **Verified** só depois da prova da Task 8, com o usuário.
- [ ] **Step 2: Suíte completa do api.** Não é desta sessão: a sessão dona do api faz o pedido à sessão "Testes" (`api · mod26c/feat/mod-26c-mira · --full · mudança transversal do 26c`). A suíte inteira do dashboard neste worktree (Task 8 Step 1) basta para o dashboard.
- [ ] **Step 3: Pare e peça autorização** para merge e push (depois do api e da suíte completa).
  - Antes do push, confira `$G -C $W log --oneline origin/main..HEAD` e `origin/main..main` (sessões paralelas).
  - O worktree só sai depois do merge: `$G -C $R/apps/dashboard worktree remove $W`.

**Relato:** `dashboard · dashboard-mod26c/feat/mod-26c-mira · Task 9 · suíte <n>/<n>, build ok, varreduras ok, prova <passos> · cards movidos · aguardando autorização de merge`.

---

## Self-review

1. **Cobertura do contrato §6, da spec §3.5 e das Q9–Q14:**
   - item `regulation-rnds` "Envio à RNDS", com faixa (única do 26c), totais com `accepted`, lista por motivo, `request_ref` com link, atalhos e "Reenviar": Tasks 3 e 7 (Q11, Q13);
   - resumo só de leitura das configurações para o regulador: Task 7 (Q13);
   - seção "Envio à RNDS" com cor → caráter, SIGTAP por especialidade (recusa de SIGTAP não regulável, Q9) e os externos com o aviso, com step-up: Task 5 (Q13);
   - "CBO do executante" na oferta de procedimento e no prestador, com o PATCH só do CBO e a herança do mês anterior: Task 4 (Q12);
   - credencial `rnds_regulation` com os dois campos a mais: Task 6;
   - `REGULATION_RNDS_ERRORS` em `src/lib/regulationRndsErrors.ts`, com os códigos da Q13: Task 2;
   - `rnds_sending_disabled` com a frase da Q14: Task 6;
   - o rótulo do interruptor: Task 1.
2. **Bordas que tocam a tela:** CBO e SIGTAP com pontuação (Task 2); CNS com espaço e dígito errado (Tasks 2 e 6); 409 das configurações e SIGTAP não regulável (Task 5); reenvio repetido, estado mudado e envio pausado (Task 7); código novo do api (Tasks 2 e 7); atalho e link por papel (Tasks 2 e 7); oferta de outro mês no atalho (Task 4).
3. **Tipos:** `RegulationRndsSync.fix` (`offer_id`, `provider_id`, `unit_id`, `settings`) e `FixTarget` (Tasks 1–2) são os mesmos nas Tasks 3, 4 e 7; `RegulationRndsStatus` (`usable`, `missing`, `late_banner`) serve à Task 7; `RndsSettingsDraft` e `settingsFieldOf` (Task 2) servem à Task 5; `updateCareOffer(id, { quantity?, performer_cbo? })` (Task 1) é o da Task 4.
4. **Review Focus:** as cinco linhas têm teste na task dona (Tasks 7, 5, 2, 7 e 7).
5. **Proporção:** o código do plano é tipo, frase e teste; as telas vão por assinatura, props e comportamento.

## Divergências propostas ao contrato

Todas foram decididas no §10 do contrato (docs `5d49ef7`), e o plano já aplica as decisões. O texto de cada uma fica como registro.

- **D1 — Resolvida pela Q11.** "Número do pedido" não existia: o pedido do 26a só tem `id` (uuid) (`apps/api/db/city_migrate/20261028900001_add_care_regulation.rb:84-115`; `apps/api/app/services/care_regulation/json.rb:18-24`; `src/lib/api.ts:4608-4624`). **Decisão:** a lista mostra o `request_ref`, os 8 primeiros caracteres hex do id, em maiúsculas, calculados pelo api, com o link para o detalhe (Tasks 1, 2 e 7).
- **D2 — Resolvida pela Q11 (e pela Q10).** As formas do status e da lista não estavam fixadas, e o §4 não contava os aceitos. **Decisão:** as contagens trazem `accepted`; as formas de `/status` (`usable`, `missing`, `counts`, `late_banner`, `initial_load_pending`) e de `/syncs` (o item sem alvo, com `late` e `fix { offer_id, provider_id, unit_id, settings }`) são as do plano do api, Task 14. Pela Q10, as leituras só exigem o interruptor ligado. Como o `fix` não traz o mês da oferta, o atalho procura a oferta no mês corrente e avisa quando ela é de outro mês (Task 4).
- **D3 — Resolvida pela Q9.** `specialty_without_sigtap` nunca aconteceria com o padrão `0301010072`. **Decisão:** o motivo vale quando o SIGTAP configurado, ou o padrão, não for regulável ou não estiver vigente na competência. A frase e o texto do padrão dizem isso (Tasks 2 e 5).
- **D4 — Resolvida pela Q12.** No pedido por especialidade o alvo já é o CBO (`apps/api/app/services/care_regulation/catalog.rb:32`). **Decisão:** o CBO do alvo de especialidade é a primeira fonte, antes da agenda, da oferta e do prestador. O campo da oferta só aparece em procedimento (Task 4).
- **D5 — Resolvida pela Q12.** A oferta é mensal (`apps/api/app/commands/care_regulation/save_offer.rb:10`), e o PATCH exigia `quantity` (`save_offer.rb:51`). **Decisão:** a oferta nova herda o `performer_cbo` da oferta do mês anterior do mesmo prestador e alvo; o POST aceita `performer_cbo`; o PATCH aceita mudar só o CBO (Tasks 1 e 4).
- **D6 — Resolvida pela Q13.** O PUT das configurações pede step-up (Task 5).
- **D7 — Resolvida pela Q13.** O regulador vê as configurações só para leitura, num resumo da tela "Envio à RNDS"; a seção editável fica no admin (Task 7).
- **D8 — Resolvida pela Q13.** Não há painel da regulação: a faixa fica só na tela "Envio à RNDS", sem contagem no menu (Task 7).
- **D9 — Resolvida pela Q13.** Os códigos de recusa e os formatos são os deste plano: `invalid_cbo`, `invalid_sigtap`, `invalid_priority` (com `path`), `invalid_requester_id`, `invalid_responsible_cns`, `409 not_retryable`; CBO de 6 dígitos, SIGTAP de 10, identificador com 1–20 dígitos, CNS de 15 (Task 2). O plano do api (Valores 2 e 10) ainda cita `REQUESTER_ID_FORMAT` alfanumérico até 30 e `invalid_request` com `path` nas configurações; a Task 0 confere, e a Task 2 traduz os dois formatos de recusa.
- **D10 — Resolvida pela Q14.** O motivo do pré-requisito "um dos envios" é `rnds_sending_disabled`, com a frase "ligue antes um dos envios à RNDS (vacina ou regulação)" (Task 6). O api cria o requisito alternativo no `missing_for`.
