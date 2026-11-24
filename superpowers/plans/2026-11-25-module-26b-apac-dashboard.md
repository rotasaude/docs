# Módulo 26b — APAC, OCI, teto em dinheiro e AIH (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-25-module-26b-apac-contracts.md` (fonte única), §1, §3–§11 e Q1–Q13 (§12; docs `60908e1`). As divergências D1–D16 deste plano foram decididas pelas Q6–Q10 e Q13 (seção final), e o plano já aplica as decisões. Pela Q10, rotas, corpos e códigos de recusa são os do plano do api (`2026-11-25-module-26b-apac-api.md`, Valores 3–11 e Interfaces das Tasks 5–14); o que ele ainda não fixa (a rota da prévia, Q13, e os nomes do número inteiro da faixa, Q8) é conferido na Task 0. A D17 (as listas dos `select`) segue aberta.
> 2. A **`main` do dashboard com o 26c mergeado** (o contrato manda executar o 26b depois do 26c). Quando este plano foi escrito, o `origin/main` estava em `2e3a2a9`, só com o 35a (sem o 27b e sem o 26c). As âncoras deste plano são de `2e3a2a9`; a Task 0 confere as que o 27b e o 26c mudam.
> 3. O **api do 26b** na branch `feat/mod-26b-apac` (`apps/api/.claude/mod26b`, porta **3064**). As Tasks 1–11 seguem pelas formas deste plano e pelas fixtures. Só a Task 12 (prova) depende do api rodando.
>
> Ordem: api (com a Task 0 dele: layout da APAC, dígito verificador e SIGTAP completa) → **dashboard** e maintenance → suíte completa do api (sessão "Testes") → prova.

**Goal:** Lado dashboard do 26b:
- na consulta, o **pedido por laudo** (APAC, OCI com secundários, AIH), com o laudo em PDF e a assinatura;
- na fila, o tipo do pedido e a **autorização numerada**, com a linha da FPO, o saldo, a reserva, o número e o "Autorizar acima do teto";
- no grupo **Regulação**, as telas novas: numeração e momento da autorização, programação (FPO) com importação conferida, "OCI em andamento", "Revisão de APAC", "Remessa APAC" e "Fila nominal da OCI";
- no faturamento (22), o motivo `oci_secondary_active` e o número da autorização; em "Pendentes de assinatura", o laudo.

**Architecture:** O cliente ganha a seção "Regulação por laudo (módulo 26b)" no fim do bloco da regulação em `src/lib/api.ts`, com `regPath`/`regWrite` do 26a e o `binaryFetch` do 19b. Onde fica cada coisa:
- frases: `src/lib/regulationApacErrors.ts` (`REGULATION_APAC_ERRORS`, contrato §11);
- portão, papéis, rótulos, rascunho do laudo, regras de formato, faixa, FPO e importação: `src/lib/regulationApac.ts`;
- telas: `src/modules/RegulationApac*.tsx` e os painéis em `src/modules/regulation/apac/`;
- extensões do 26a: `RequestForm`, `RequestDetail`, `AuthorizeForm`, `RequestTable`, `RegulationRequestsField`, `RegulationReturned`.

O dashboard não decide o que o api calcula: o tipo do alvo, a compatibilidade dos secundários, a linha da FPO mais específica, o valor SIGTAP, a reserva, o 5º dígito, o número e o dígito verificador, a validade, o alarme de oncologia e o conteúdo dos arquivos. Ele mostra o que vem e só não oferece o que sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-25-module-26b-apac-oci-aih-design.md` (decisões 4, 6–13, 16–22; §3). **Contrato:** `docs/superpowers/plans/2026-11-25-module-26b-apac-contracts.md` §1, §3–§11, Q1–Q5. **Moldes:** `2026-11-25-module-26c-mira-dashboard.md` (forma do plano); a Task 11 do `2026-11-24-module-27b-bnafar-dashboard.md` ("Saldo inicial": importação de CSV com conferência); no código, `src/modules/psychosocial/RaasExports.tsx` (remessa por unidade e mês), `src/modules/regulation/openGuide.ts` (PDF numa janela nova) e `src/modules/RegulationSettings.tsx` (painéis do admin com step-up).

## Global Constraints

- **Nomes no código** (contrato §1, §11): interruptor `care_regulation_apac`; frases em `REGULATION_APAC_ERRORS` (`src/lib/regulationApacErrors.ts`); `authorization_kind` `bpa | apac | oci | aih` (ausente = `bpa`); `stage` `issued | awaiting_review | review_denied`; `financing` `mac | faec | credits | other` (`other` = padrão sem reserva de teto, Q9); nível da FPO `procedure | organization_form | subgroup | group`; `authorization_moment` `before | after`; caráter da AIH `elective | urgent`. **Código desconhecido aparece cru, nunca some.**
- **Frases fixas** (letra por letra):
  - o interruptor: "Regulação por laudo (APAC, OCI e AIH)";
  - as telas: "OCI em andamento", "Remessa APAC", "Fila nominal da OCI", "Revisão de APAC";
  - o botão "Autorizar acima do teto" (§11) e, na revisão, "Autorizar a APAC" (spec decisão 19);
  - as telas existentes "Fila regulada", "Prestadores e cotas", "Prazos e lista do enfermeiro", "Pendentes de assinatura".

  Toda outra frase deste plano é **proposta** e está marcada assim.
- **Valores do contrato:** resumo clínico do laudo com 20–4000 caracteres; OCI com o principal do grupo `09` e ao menos 2 secundários; 5º dígito APAC `2`, `7`, `6`, `9` e AIH `1`, `5`, `8`; número com 13 dígitos; APAC Única e OCI valem 2 competências; alarme da OCI de oncologia aberta há mais de 25 dias corridos, contados do primeiro procedimento realizado (prazo de 30), e da OCI vencida (Q7); ao menos um secundário da OCI é consulta ou teleconsulta (Q7); `findings` até 4000, até 30 procedimentos, quantidade de 1 a 999 e `clinic` com 2 dígitos (Q12). O dashboard mostra esses valores; quem calcula é o api.
- **Portões:**
  - `canUseRegulationApac(user)` = `canUseRegulation(user)` e `care_regulation_apac` na sessão (a sessão traz as chaves **ligadas**, `src/lib/features.ts:1-4`). Com o interruptor ligado mas sem a SIGTAP completa, o api recusa com `403 feature_disabled`, e a tela mostra a frase do interruptor.
  - Sem o interruptor, nada do 26b aparece, e o 26a fica exatamente como hoje (os testes existentes do 26a não mudam).
  - Pedido por laudo: quem já pede regulação (`canRequestRegulation`); o solicitante escolhe o instrumento (BPA, APAC, OCI ou AIH), e a busca filtra por ele com `kind` e devolve o `authorization_kind` (Q6).
  - Fila, autorização, "OCI em andamento" e "Revisão de APAC": `care_regulator` (leitura nominal com trilha, `NOMINAL_QUERY`).
  - Numeração, momento e FPO: escrita do `municipal_admin` com step-up; o `care_regulator` lê a FPO e o momento (Q13).
  - "Remessa APAC": `municipal_admin` e `billing_clerk` (Q10). "Fila nominal da OCI": `municipal_admin`.
- **Step-up** (`SensitiveAction` com `requiresStepUp`) em: gravar faixa, ativar/desativar faixa, salvar o momento, criar/editar linha da FPO, aplicar a importação da FPO, "Autorizar acima do teto", gerar a remessa e gerar a fila nominal. A autorização comum segue como no 26a (sem step-up). Toda escrita leva corpo JSON (`{}` quando não há campo).
- **Concorrência** (Q13): nas ações sobre o pedido vale o 26a (`version` no corpo, `409 stale_version` → "Recarregar"); nos recursos novos, `expected_version` → `409 apac_settings_changed`, `fpo_line_changed`, `fpo_import_changed` (ausente → `422 expected_version_required`): a frase, **uma** releitura e o digitado mantido. Botão de escrita desabilitado enquanto a chamada corre: o duplo clique manda um pedido só.
- **Formato** (recusar, nunca cortar; o api confere de novo): SIGTAP com 10 dígitos depois de tirar `.`, `-`, `/` e espaço, mostrado como `03.01.01.007-2`; CID-10 pelo `normalizeCid10`/`cid10Problem` (`src/lib/consultation.ts:208-212`); quantidade inteira de 1 a 999; dinheiro pelo `parseReais`/`fmtCents` (`src/lib/transport.ts:103,122`); competência `AAAAMM` (Q13) pelo `competenceLabel` (`src/lib/competence.ts:9`); sequencial da faixa com 1–7 dígitos (Q8); ano com 4 dígitos.
- **Sigilo** (contrato, convenções e §11 LGPD):
  - o laudo (resumo, CID principal e secundário, achados, caráter, clínica) só no corpo e no estado do formulário: nunca em URL, chave do `queryClient`, `localStorage`, `console` ou lista;
  - o laudo aparece só no detalhe do pedido e na lista da própria consulta (Q13); a fila, os devolvidos, "OCI em andamento", "Revisão de APAC" e os arquivos nunca o mostram;
  - o PDF do laudo e os arquivos vão por `blob:` numa janela nova ou pelo `saveFile`, com nome sem dado pessoal;
  - na URL vão só ids, competência, tipo, situação e cursor.
- **Leituras:** fila, detalhe, OCI e revisão são nominais (`NOMINAL_QUERY`, sem polling, relidas só depois de uma ação ou no "Recarregar"). Faixas, momento, FPO, opções e arquivos são sem pessoa (`STATIC_QUERY`, relidos ao abrir e depois de cada escrita). **Nenhum polling.**
- **Interface:**
  - toda tela nova entra em `NAV_GROUPS` **e** em `CATALOG.screens` (o `catalog.test.ts` cobra);
  - `DataTable` com `card: "title"`/`"actions"`, e 375 px sem rolagem horizontal;
  - cores só por papéis do tema (`Tag` com `tone`), com as guardas `src/theme/*.test.ts` verdes;
  - componentes existentes: `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `SegmentedControl` (`src/shell/SegmentedControl.tsx`), `formStyles`, os estilos de `src/modules/exams/styles.ts` e `src/modules/regulation/styles.ts`, `RegulationGate`, `TargetSearch`, `saveFile`.
- **CI (lição do `443faca`):** todo `vi.mock("…/lib/api", async (orig) => …)` inclui `fetchCurrentSession: vi.fn()` resolvendo o usuário.
- **Datas:** as telas recebem `today` por prop (como o 26a). Os testes usam `TODAY26B = "2026-12-02"` (competência `202612`, `TZ=America/Sao_Paulo`).
- **Testes:** só `$R/bin/test dashboard -w dashboard-mod26b [arquivos do vitest]`, nunca `vitest` direto. Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge, push e tag só com autorização explícita.
- **Board #1:** o dashboard é superfície dos cards do 26b (criados pelo plano do api). Card vai a Done só quando o **último** app dele termina, e a Verified depois da prova (Task 12).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`): `dashboard · dashboard-mod26b/feat/mod-26b-apac · Task N · <pronto> · <divergência nova>`.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod26b`, `G=/opt/homebrew/bin/git`, `A=$R/apps/api/.claude/mod26b`.
- Worktree **fora de `apps/dashboard`** (o compose monta `./apps/dashboard:/app`, e um `node_modules` ali duplica o React no Vite). Branch `feat/mod-26b-apac` a partir de `origin/main`, com `npm ci`. Os caminhos das tasks são relativos a `$W`.
- Testes sem jest-dom (`not.toBeNull()`, `toBe()`), com `globals: false` e `cleanup` no `afterEach`.
- Fixtures:
  - `renderWithProviders`/`sessionWith`: `src/test/campaignFixtures.tsx`;
  - do 26a: `src/test/regulationFixtures.ts` (`regUser`, `regRequest`, `offer`, `provider`, `ECO`, `UBS_CENTRO`, `TODAY26`);
  - novas: `src/test/regulationApacFixtures.ts` (Task 1).

## Review Focus

1. **Autorizar com o teto esgotado, com duplo clique ou com o step-up vencido no "acima do teto".** Espera-se:
   - um pedido só;
   - `409 ceiling_exhausted` mostra a frase com o saldo da prévia relida e oferece "Autorizar acima do teto", com motivo (10–500) e step-up;
   - `401 mfa_required` volta ao step-up sem reenviar;
   - `409 number_range_exhausted` mostra a frase que manda o administrador cadastrar a faixa, e `403 authorizer_not_allowed`, a frase do autorizador;
   - o número emitido aparece só como o api devolveu.

   Teste: Task 5, "autorizar: teto esgotado, acima do teto com step-up, um pedido, faixa esgotada e autorizador recusado".
2. **OCI montada como as pessoas montam:** um secundário só, o mesmo código duas vezes, quantidade `"2,5"` ou `"0"`, SIGTAP `"03.01.01.007-2"`, e a recusa `422 incompatible_secondary` com `path: "procedures[2]"`. Espera-se a recusa antes de enviar (menos de 2 secundários, repetido, quantidade) e as recusas do api: `oci_needs_consultation` acima dos secundários e `incompatible_secondary` na linha certa (a terceira, contando o principal), com o digitado mantido. Teste: Task 4, "OCI: secundários conferidos antes e recusa do api na linha certa".
3. **Planilha da FPO:** `.xlsx`, 6 MB, cabeçalho com vírgula, CSV com BOM, valor `"1.234,56"`, aplicar duas vezes e uma linha FAEC por grupo. Espera-se as recusas antes de enviar sem chamada, o BOM aceito, um pedido só no aplicar e a linha `faec_not_procedure` com a frase. Teste: Task 7, os casos de mesmo nome.
4. **Faixa digitada como o ofício da SES traz:** o número inteiro de 13 dígitos colado no campo do sequencial, primeiro maior que o último, 5º dígito `7` numa faixa de AIH, e a sobreposição recusada pelo api. Espera-se a frase que pede só o sequencial (Q8), as recusas antes de enviar e `409 number_range_overlap` com a frase, sem perder o digitado. Teste: Task 6, "faixa: número inteiro colado, ordem, 5º dígito do tipo e sobreposição".
5. **Api diferente da tela:** alvo sem `authorization_kind` (api sem o 26b), tipo, etapa, financiamento ou motivo de saída novos. Espera-se o alvo tratado como BPA (o formulário do 26a, sem laudo), os códigos crus e nenhuma linha sumindo. Teste: Task 2, `kindOf` e rótulos por tabela; Task 5, "tipo, etapa e financiamento desconhecidos aparecem crus".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.regulationApac.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/regulationApacFixtures.ts` | tipos e cliente do 26b; extensões do 26a e do 19b; rótulo do interruptor; fixtures | 1 |
| `src/lib/regulationApacErrors.ts`, `src/lib/regulationApacErrors.test.ts`, `src/lib/regulationApac.ts`, `src/lib/regulationApac.test.ts` | frases; portão, papéis, rótulos, laudo, procedimentos, faixa, FPO, planilha | 2 |
| `src/shell/modules.ts`, `src/shell/modules.regulationApac.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/regulation/apac/ApacGate.tsx`, cascas `src/modules/RegulationApacSettings.tsx`, `RegulationFpo.tsx`, `RegulationOci.tsx`, `RegulationApacReview.tsx`, `RegulationApacExports.tsx`, `RegulationOciQueue.tsx` | itens do menu, portão, casca das telas | 3 |
| `src/modules/regulation/RequestForm.tsx`, `src/modules/regulation/TargetSearch.tsx`, `src/modules/regulation/apac/LaudoFields.tsx`, `src/modules/regulation/apac/ProceduresField.tsx`, `src/modules/regulation/openGuide.ts`, `src/modules/consultation/RegulationRequestsField.tsx`, `src/modules/RegulationReturned.tsx`, `src/modules/regulation/RequestForm.laudo.test.tsx`, `src/modules/consultation/RegulationRequestsField.laudo.test.tsx` | pedido por laudo na consulta e no complemento; laudo em PDF | 4 |
| `src/modules/regulation/RequestTable.tsx`, `src/modules/regulation/RequestDetail.tsx`, `src/modules/regulation/AuthorizeForm.tsx`, `src/modules/regulation/apac/AuthorizationPreview.tsx`, `src/modules/regulation/apac/LaudoView.tsx`, `src/modules/RegulationQueue.apac.test.tsx`, `src/modules/regulation/AuthorizeForm.apac.test.tsx` | tipo na fila; laudo e autorização no detalhe; autorização com teto e número | 5 |
| `src/modules/RegulationApacSettings.tsx`, `src/modules/regulation/apac/NumberRangesPanel.tsx`, `src/modules/regulation/apac/MomentPanel.tsx`, `src/modules/RegulationApacSettings.test.tsx` | faixas de numeração e momento da autorização | 6 |
| `src/modules/RegulationFpo.tsx`, `src/modules/regulation/apac/FpoLinesPanel.tsx`, `src/modules/regulation/apac/FpoLineForm.tsx`, `src/modules/regulation/apac/FpoImportPanel.tsx`, `src/modules/RegulationFpo.test.tsx` | programação (FPO): linhas, edição e importação conferida | 7 |
| `src/modules/RegulationOci.tsx`, `src/modules/regulation/apac/OciProcedures.tsx`, `src/modules/RegulationOci.test.tsx` | "OCI em andamento" | 8 |
| `src/modules/RegulationApacReview.tsx`, `src/modules/RegulationApacReview.test.tsx` | "Revisão de APAC" | 9 |
| `src/modules/RegulationApacExports.tsx`, `src/modules/RegulationOciQueue.tsx`, `src/modules/RegulationApacExports.test.tsx`, `src/modules/RegulationOciQueue.test.tsx` | "Remessa APAC" e "Fila nominal da OCI" | 10 |
| `src/lib/billing.ts`, `src/lib/billing.test.ts`, `src/modules/billing/ProcedureTable.tsx`, `src/modules/billing/ProcedureTable.apac.test.tsx`, `src/lib/signature.ts`, `src/lib/signature.test.ts` | motivo `oci_secondary_active`, número no procedimento, laudo em "Pendentes de assinatura" | 11 |
| — | suíte, build, varreduras, revisão, prova no navegador (5225 → 3064) | 12 |
| — | board, relato, parada antes do merge | 13 |

**Estratégia de teste:**
- **Cliente** (Task 1): `fetch` falso (`urlOf`, `initOf`, `bodyOf`, como em `api.regulation.test.ts`), conferindo URL, método e corpo; o laudo só no corpo; filtro vazio nunca na URL; envelope ausente = vazio.
- **Regras puras** (Task 2): tabela de casos, com as bordas do Review Focus 2, 4 e 5 (secundários, faixa, código desconhecido) e o dinheiro e o cabeçalho da planilha do Review Focus 3.
- **Menu** (Task 3): por papel e por interruptor; sem `care_regulation_apac`, a lista do 26a/26c não muda.
- **Telas** (Tasks 4–11): pelo ponto de entrada, com `renderWithProviders`, `vi.mock` do api com `fetchCurrentSession` e sessão por papel e por interruptor. Cobrem:
  - o caminho feliz;
  - cada recusa que muda a tela;
  - o duplo envio;
  - o 409 com uma releitura;
  - o que **não** pode aparecer: laudo na fila ou em chave de cache, botão de escrita para quem só lê, nada do 26b com o interruptor desligado.
- **Regressão do 26a:** os arquivos de teste do 26a (`RequestForm.test.tsx`, `RequestDetail.test.tsx`, `RegulationQueue.test.tsx`, `RegulationReturned.test.tsx`, `ConsultationWorkspace.regulation.test.tsx`) rodam em toda task que toca o componente, sem mudar uma asserção.
- **Fim** (Task 12): `tsc`, build, suíte, varreduras e prova no navegador, com o api real na 3064 e o Vite na 5225.

O dashboard não testa o que é do api: a resolução do alvo, a compatibilidade, a escolha da linha da FPO, o valor SIGTAP, a trava, o número e o dígito verificador, a migração da reserva, a liberação, a crítica do BPA e o conteúdo dos arquivos.

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (com o 26c); board #1; o api `feat/mod-26b-apac`.
- Produces: cards em In Progress; `$W` na branch; base, âncoras, ícones e formas do api anotados.

- [ ] **Step 1: Cards.** Um `gh` simples por vez, sem `$(...)` nem laço:

  ```bash
  gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-26\\.")) | {id, title, status}'
  ```

  Os cards do 26b que o dashboard toca (os criados pelo plano do api) vão a In Progress, com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <In Progress>`. Card ausente: **pare e pergunte**.

- [ ] **Step 2: Base e âncoras**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin && $G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -c 'rnds_regulation'
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^const regWrite\|^export interface RegulationTarget\|^export interface RegulationRequest {\|^export interface RegulationRequestInput\|^export interface AuthorizeInput\|^export interface ComplementInput\|^export type SignatureDocumentType\|^export function fetchRegulationGuide\|^export async function removeNurseItem\|^export interface PerformedProcedure {'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '"regulation-settings"\|"regulation-rnds"\|if (item.id === "regulation-settings") return'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n 'case "regulation-queue"\|case "regulation-settings"\|case "regulation-rnds"'
$G -C $R/apps/dashboard show origin/main:src/lib/transport.ts | grep -n '^export function parseReais\|^export const fmtCents'
$G -C $R/apps/dashboard show origin/main:src/lib/signature.ts | grep -n '^const DOCUMENT_LABEL'
$G -C $R/apps/dashboard show origin/main:src/lib/billing.ts | grep -n '^const REASON_LABEL'
```

Expected:
- o 26c presente (contagem > 0). Se for 0, **pare**: o contrato manda começar depois dele;
- cada linha achada. Linha mudada pelo 27b ou pelo 26c (por exemplo, `regWrite` já com `"PUT"`, `updateCareOffer(id, { quantity?, performer_cbo? })`, `RegulationQueue({ today, openRequestId })`): a task dona aplica a mesma intenção sobre o texto real e anota no commit.

- [ ] **Step 3: Worktree, base e ícones**

```bash
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod26b -b feat/mod-26b-apac origin/main
cd $R/.worktrees/dashboard-mod26b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod26b
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Numbers|ReportMoney|Stethoscope|ClipboardCheck|FileExport|ListDetails)\.mjs$'
```

Expected: `tsc-ok`; suíte verde (**anote** arquivos e testes); os seis ícones. Faltou um: use `IconFileText`, na Task 3.

- [ ] **Step 4: Formas do api.** Leia, na branch `feat/mod-26b-apac` do api:
  - as rotas: `$G -C $R/apps/api show feat/mod-26b-apac:config/routes.rb | grep -n 'apac\|fpo\|number_ranges\|oci\|authorization_preview\|laudo\|review'`;
  - os controllers e o `CareRegulation::Json` (o `request` com `laudo`, `procedures`, `oci` e a `authorization` nova; o `list_item` sem `laudo`);
  - os specs de request e o `spec/security/regulation_apac_secrecy_spec.rb`;
  - o `Platform::Features::CATALOG` (`care_regulation_apac`, `requires`) e a busca de alvo com `kind` e `authorization_kind` (Q6);
  - o que o plano do api ainda não fixa: a rota e a forma da prévia (Q13), os nomes do número inteiro da faixa (Q8), o `complement_cents_per_unit` da linha da FPO (Q9) e se há rota para as listas dos `select` (D17).

  Confira tudo contra as formas da Task 1. **Onde o api diz outra coisa, vale o api**: a task dona ajusta tipo, fixture e teste e registra no commit; a diferença vai ao coordenador como divergência nova. Sem branch ainda: siga pelas formas deste plano e repita este passo antes da Task 12.

---

### Task 1: Tipos, cliente HTTP, rótulo do interruptor e fixtures

**Files:**
- Modify: `src/lib/api.ts` (seção nova depois de `removeNurseItem`; extensões no 26a e no 19b), `src/lib/features.ts`
- Create: `src/lib/api.regulationApac.test.ts`, `src/test/regulationApacFixtures.ts`
- Test: `src/lib/features.test.ts`

**Interfaces:**
- Produces: em `src/lib/api.ts`, a seção "Regulação por laudo (módulo 26b; contrato 2026-11-25 §3–§11, Q6–Q13)". Pela Q10, rotas, corpos e códigos são os do plano do api (Valor 11 e Interfaces das Tasks 5–14); o que ele ainda não fixa está marcado e é conferido na Task 0:

```ts
export type AuthorizationKind = "bpa" | "apac" | "oci" | "aih";
export type AuthorizationStage = "issued" | "awaiting_review" | "review_denied";
export type Financing = "mac" | "faec" | "credits" | "other";          // Q9: `other` = padrão sem reserva de teto
export type FpoLevel = "procedure" | "organization_form" | "subgroup" | "group";
export type AihCharacter = "elective" | "urgent";
export type AuthorizationMoment = "before" | "after";
export type OciOutcome = "concluded" | "not_concluded";                // Q7
// Q13: só no detalhe, no 201 e na lista da própria consulta; nunca na fila. Q6: sem CID principal (é o `cid10` do pedido).
// Q10: a assinatura do laudo (abre na promoção, autor = solicitante).
export interface RegulationLaudo {
  clinical_summary: string; cid_secondary: string | null; findings: string | null;
  character: AihCharacter | (string & {}) | null; clinic: string | null; version: number; signature: SignatureBlock | null;
}
export interface RegulationLaudoInput {
  clinical_summary: string; cid_secondary?: string | null; findings?: string | null; character?: AihCharacter; clinic?: string;
}
export interface RequestedProcedure {
  id: string; sigtap_code: string; name: string; role: "principal" | "secondary" | (string & {}); quantity: number;
  performed_at: string | null; performed_source: "performed_procedure" | "manual" | (string & {}) | null;
}
// Plano do api, Task 5: exatamente um principal, igual ao `target_code`, primeiro; `procedures[i]` indexa este array; até 30 (Q12).
export interface RequestedProcedureInput { sigtap_code: string; role: "principal" | "secondary"; quantity: number }
// Q13: prévia da autorização, lida antes do POST (rota e nomes conferidos na Task 0: o plano do api ainda não a lista).
export interface AuthorizationPreview {
  ceiling_applies: boolean; competence: string; unit_cents: number; quantity: number; reserve_cents: number;
  financing: Financing | (string & {}) | null; faec_program: string | null; financing_default: Financing | (string & {}) | null;
  fpo_line: { id: string; level: FpoLevel | (string & {}); code: string; amount_cents: number; balance_cents: number } | null;
  numbered: boolean; fifth_digit: string | null; range_remaining: number | null; moment: AuthorizationMoment | (string & {});
}
export interface AuthorizationReview { decision: "issue" | "deny"; reason?: string; version: number }   // Q10: motivo na negativa
// Plano do api, Task 6 (`ApacJson.range`); Q8: o api devolve também o número inteiro (nomes conferidos na Task 0).
export interface NumberRange {
  id: string; kind: "apac" | "aih" | (string & {}); fifth_digit: string; year: number;
  first_number: number; last_number: number; next_number: number; remaining: number; active: boolean;
  first_full: string; last_full: string;
}
export interface NumberRangeInput { kind: "apac" | "aih"; fifth_digit: string; year: number; first_number: number; last_number: number }
export interface ApacSettings { authorization_moment: AuthorizationMoment | (string & {}); version: number }
// Q9: `complement_cents_per_unit` entra na reserva; a forma da linha é conferida na Task 0.
export interface FpoLine {
  id: string; competence: string; care_provider: { id: string; name: string; kind: ProviderKind | string }; level: FpoLevel | (string & {});
  code: string; name: string | null; financing: Financing | (string & {}); faec_program: string | null; quantity: number; amount_cents: number;
  complement_cents_per_unit: number; reserved_cents: number; balance_cents: number; version: number;
}
export interface FpoLineInput {
  competence: string; care_provider_id: string; level: FpoLevel; code: string; financing: Exclude<Financing, "other">;
  faec_program?: string | null; quantity: number; amount_cents: number;
}
export interface FpoLinePage { items: FpoLine[]; next_before: string | null }
export type FpoImportStatus = "validated" | "applied" | "discarded";
// Plano do api, Task 7 (`UploadFpoImport`): o relatório.
export interface FpoImport {
  id: string; status: FpoImportStatus | (string & {}); version: number; created_at: string; applied_at: string | null;
  report: { rows_valid: number; rows_error: number; errors_by_code: Record<string, number>;
    totals_by_provider: { care_provider_id: string; lines: number; amount_cents: number }[] };
}
export interface FpoImportRow {
  line: number; status: "valid" | "error" | (string & {}); error_codes: string[];
  cnes: string | null; level: string | null; code: string | null; financing: string | null; quantity: number | null; amount_cents: number | null;
}
export interface FpoImportDetail extends FpoImport { rows: FpoImportRow[] }
// Plano do api, Task 11 (`OciList.page`): sem laudo nem CID.
export interface OciItem {
  id: string; request_ref: string; target_name: string; care_provider: { id: string; name: string } | null; authorization_number: string | null;
  valid_until_competence: string | null; secondaries: { id: string; sigtap_code: string; name: string; performed_at: string | null }[];
  days_open: number; oncology_alert: boolean; expired: boolean;
}
// Plano do api, Task 9 (`GET /regulation/authorizations?stage=awaiting_review`): sem laudo.
export interface ReviewItem {
  id: string; regulation_request_id: string; request_ref: string; authorization_kind: AuthorizationKind | (string & {});
  target_name: string; care_provider: { id: string; name: string } | null; performed: { done: number; total: number };
}
// D17 (aberta): as listas dos `select` vêm dos `config/regulation/*.yml` do api.
export interface CodeName { code: string; name: string }
export interface ApacOptions { oci_exit_reasons: CodeName[]; aih_clinics: CodeName[]; faec_programs: CodeName[]; budget_sources: CodeName[] }
export interface ApacExport {
  id: string; competence: string; health_unit: { id: string; name: string }; apacs: number; sha256: string;
  layout_version: string | null; version: number; generated_at: string; generated_by: { name: string | null } | null;
}
export interface OciQueueExport { id: string; competence: string; rows: number; sha256: string; version: number; generated_at: string }
```

- Extensões (26a), todas opcionais para o api sem o 26b:
  - `RegulationTarget.authorization_kind?: AuthorizationKind | (string & {})` (Q6);
  - `searchRegulationTargets(q, type, kind?: AuthorizationKind | null)`: `kind` vai na URL só quando não é nulo nem `bpa` (Q6; plano do api, Valor 11);
  - `RegulationRequest`: `authorization_kind?`, `laudo?: RegulationLaudo | null` (Q13), `procedures?: RequestedProcedure[]`, `oci?: { outcome: OciOutcome | (string & {}) | null; exit_reason: string | null; closed_at: string | null } | null` (Q7; nomes conferidos na Task 0);
  - `RegulationRequest.authorization` ganha `id`, `number: string | null`, `authorization_kind`, `stage`, `valid_from_competence`, `valid_until_competence`, `financing`, `faec_program`, `budget_source`, `reserved_cents: number | null`, `fpo_line: { id: string; level: string; code: string; amount_cents: number; balance_cents: number } | null`, `ceiling_overridden: boolean` e `reservation_mismatch?: boolean` (todos opcionais; plano do api, Tasks 8–9);
  - `RegulationRequestInput` e `ComplementInput` (Q13): `authorization_kind?`, `laudo?: RegulationLaudoInput`, `procedures?: RequestedProcedureInput[]`;
  - `AuthorizeInput`: `financing?: Financing`, `faec_program?: string | null`, `budget_source?: string` (Q9: exigida nos créditos), `ceiling_override_reason?: string`;
  - `regWrite` aceita `"PUT"` (se o 26c ainda não o fez).
- Extensões (19b e 22): `SignatureDocumentType` + `"regulation_laudo"` (Q10); `PerformedProcedure.authorization_number?: string | null` e `regulation_authorization_id?: string | null` (§7).
- Funções (todas `Promise`; listas `{ items, next_before }` ou `{ items }`, envelope ausente = vazio):
  - `fetchRegulationLaudoPdf(id)` → `binaryFetch` em `/regulation/requests/:id/laudo.pdf` (`application/pdf`, Q10).
  - `getAuthorizationPreview(id, q: { offerId: string; financing?: Financing | null; faecProgram?: string | null })` → `GET /regulation/requests/:id/authorization_preview?offer_id=&financing=&faec_program=` (Q13; rota conferida na Task 0).
  - `listAwaitingReview(before?: string | null): { items: ReviewItem[]; next_before }` → `GET /regulation/authorizations?stage=awaiting_review&before=&limit=`; `reviewAuthorization(authorizationId, i: AuthorizationReview): RegulationRequest` → `POST /regulation/authorizations/:id/review` (§5).
  - `listOci(before?: string | null): { items: OciItem[]; next_before }` → `GET /regulation/oci?before=&limit=`; `markProcedurePerformed(requestId, procedureId, i: { performed_at: string; version: number }): RegulationRequest` → `POST /regulation/requests/:id/procedures/:pid/performed`; `closeOci(requestId, i: { outcome: OciOutcome; exit_reason?: string; version: number }): RegulationRequest` → `POST /regulation/requests/:id/oci_close` (§6, Q7).
  - `getApacOptions(): ApacOptions` → `GET /regulation/apac_options` (D17, aberta).
  - `listNumberRanges(): NumberRange[]` → `GET /regulation/admin/number_ranges`; `createNumberRange(i)` → `POST`; `setNumberRangeActive(id, active)` → `PATCH /regulation/admin/number_ranges/:id` com `{ active }`.
  - `getApacSettings()` → `GET /regulation/admin/apac_settings` (o regulador lê, Q13); `updateApacSettings(i: { authorization_moment: AuthorizationMoment; expected_version: number })` → `PUT`.
  - `listFpoLines(q: { competence: string; careProviderId?: string | null; before?: string | null }): FpoLinePage` → `GET /regulation/admin/fpo_lines?competence=&care_provider_id=&before=&limit=` (o regulador lê, Q13); `createFpoLine(i)` → `POST`; `updateFpoLine(id, i: { quantity: number; amount_cents: number; expected_version: number })` → `PATCH /regulation/admin/fpo_lines/:id`.
  - `uploadFpoImport(file: Blob): FpoImportDetail` → `POST /regulation/admin/fpo_imports` com `{ file_base64 }` (a competência vem na planilha; o `readBase64` do módulo; o nome do arquivo não vai); `getFpoImport(id)`; `applyFpoImport(id, expectedVersion)` e `discardFpoImport(id, expectedVersion)` → `POST …/:id/apply|discard` com `{ expected_version }`; `fetchFpoImportTemplate(): SignatureFile` → `/regulation/admin/fpo_imports/template.csv` (Q2, Q10).
  - `listApacExports(q: { healthUnitId?: string | null; competence?: string | null }): ApacExport[]` → `GET /regulation/apac_exports?health_unit_id=&competence=`; `createApacExport(healthUnitId, competence)` → `POST /regulation/apac_exports` com `{ health_unit_id, competence }` (§7); `fetchApacExportFile(id)` → `/regulation/apac_exports/:id/file` (`application/octet-stream, */*`).
  - `listOciQueueExports(): OciQueueExport[]` → `GET /regulation/oci_queue_exports`; `createOciQueueExport(competence)` → `POST` com `{ competence }`; `fetchOciQueueExportFile(id)` → `/regulation/oci_queue_exports/:id/file` (`text/csv`) (§8).
- `src/lib/features.ts`: `FeatureKey` + `"care_regulation_apac"`; `FEATURE_LABEL.care_regulation_apac = "Regulação por laudo (APAC, OCI e AIH)"`, com o comentário de uma linha do módulo ("Módulo 26b: exige care_regulation e a SIGTAP com financiamento").
- Fixtures (`src/test/regulationApacFixtures.ts`):
  - `TODAY26B`; `APAC_FEATURES = [ "care_regulation", "clinical_record", "care_regulation_apac" ]`; `apacUser(roles = [ "care_regulator" ], features = APAC_FEATURES)`;
  - alvos: `CATARATA` (`0405050372`, `apac`), `OCI_CARDIO` (`0901010014`, `oci`), `COLECISTO` (`0407030026`, `aih`: só o instrumento `03`, Q6), `ECO_BPA` (o `ECO` do 26a, sem `authorization_kind`);
  - `laudo(o?)` (resumo de 40 caracteres, o resto nulo, `version: 1`, `signature: null`), `aihLaudo(o?)` (`character: "elective"`, `clinic: "01"`);
  - `procedures(o?)`: o principal `0901010014` e dois secundários `0301010072` (consulta) e `0211020036`, sem `performed_at`;
  - `apacRequest(o?)` (`regRequest` com `authorization_kind: "apac"`, `cid10: "H251"`, `laudo()`, `procedures` só com o principal), `ociRequest(o?)` (com os três procedimentos e `oci: null`), `aihRequest(o?)`;
  - `issuedAuthorization(o?)` (`id: "au1"`, `number: "4126200000015"`, `stage: "issued"`, `financing: "mac"`, `reserved_cents: 77_134`, `fpo_line: { id: "fl1", level: "procedure", code: "0405050372", amount_cents: 1_000_000, balance_cents: 922_866 }`, `valid_from_competence: "202612"`, `valid_until_competence: "202701"`, `ceiling_overridden: false`);
  - `preview(o?)` (`ceiling_applies: true`, `competence: "202612"`, `unit_cents: 77_134`, `quantity: 1`, `reserve_cents: 77_134`, `financing: "mac"`, `fpo_line` com `balance_cents: 1_000_000`, `numbered: true`, `fifth_digit: "2"`, `range_remaining: 9_000`, `moment: "before"`), `exhaustedPreview(o?)` (`balance_cents: 30_000`);
  - `numberRange(o?)`, `apacSettings(o?)`, `fpoLine(o?)`, `fpoImport(o?)`, `ociItem(o?)`, `reviewItem(o?)`, `apacOptions()` (dois motivos de saída `1.1` e `4.1`, três clínicas, um programa FAEC e as fontes `01`–`04`), `apacExport(o?)`, `ociQueueExport(o?)`.

- [ ] **Step 1: Testes que falham** — `src/lib/api.regulationApac.test.ts` (molde `src/lib/api.regulation.test.ts`):

```ts
it("pedido por laudo: tipo, laudo e procedimentos só no corpo, sem CID principal no laudo", async () => {
  await createRegulationRequest("c1", { type: "procedure", target_code: "0901010014", justification: "dor torácica aos esforços",
    suggested_color: "yellow", cid10: "I209", authorization_kind: "oci", laudo: { clinical_summary: "a".repeat(40) },
    procedures: [ { sigtap_code: "0901010014", role: "principal", quantity: 1 }, { sigtap_code: "0301010072", role: "secondary", quantity: 1 } ] });
  expect([ urlOf(0).pathname, urlOf(0).search, initOf(0).method ]).toEqual([ "/consultations/c1/regulation_requests", "", "POST" ]);
  expect([ bodyOf(0).cid10, "cid_principal" in bodyOf(0).laudo, bodyOf(0).procedures.length ]).toEqual([ "I209", false, 2 ]);
});
it("busca por instrumento: kind só quando não é BPA", async () => {
  await searchRegulationTargets("catarata", "procedure", "apac"); expect(urlOf(0).searchParams.get("kind")).toBe("apac");
  await searchRegulationTargets("eco", "procedure", "bpa"); expect(urlOf(1).searchParams.has("kind")).toBe(false);
});
it("prévia, autorização com financiamento, fonte orçamentária e acima do teto", async () => {
  await getAuthorizationPreview("rr1", { offerId: "of1", financing: null, faecProgram: "" });
  expect(urlOf(0).pathname + urlOf(0).search).toBe("/regulation/requests/rr1/authorization_preview?offer_id=of1");
  await authorizeRegulationRequest("rr1", { offer_id: "of1", version: 3, financing: "credits", budget_source: "02", ceiling_override_reason: "paciente com risco de perda visual" });
  expect(bodyOf(1)).toEqual({ offer_id: "of1", version: 3, financing: "credits", budget_source: "02", ceiling_override_reason: "paciente com risco de perda visual" });
});
it("revisão, OCI e encerramento com outcome", async () => {
  await listAwaitingReview(null); expect([ urlOf(0).pathname, urlOf(0).searchParams.get("stage") ]).toEqual([ "/regulation/authorizations", "awaiting_review" ]);
  await reviewAuthorization("au1", { decision: "deny", reason: "atendimento não comprovado", version: 4 });
  expect([ urlOf(1).pathname, bodyOf(1).decision ]).toEqual([ "/regulation/authorizations/au1/review", "deny" ]);
  await listOci(null); expect(urlOf(2).pathname).toBe("/regulation/oci");
  await markProcedurePerformed("rr1", "rp2", { performed_at: "2026-12-01", version: 4 });
  expect(urlOf(3).pathname).toBe("/regulation/requests/rr1/procedures/rp2/performed");
  await closeOci("rr1", { outcome: "not_concluded", version: 5 });
  expect([ urlOf(4).pathname, bodyOf(4) ]).toEqual([ "/regulation/requests/rr1/oci_close", { outcome: "not_concluded", version: 5 } ]);
});
it("momento por PUT com expected_version; faixa e FPO", async () => {
  await updateApacSettings({ authorization_moment: "after", expected_version: 2 });
  expect([ urlOf(0).pathname, initOf(0).method, bodyOf(0) ]).toEqual([ "/regulation/admin/apac_settings", "PUT", { authorization_moment: "after", expected_version: 2 } ]);
  await setNumberRangeActive("nr1", false); expect([ initOf(1).method, bodyOf(1) ]).toEqual([ "PATCH", { active: false } ]);
  await listFpoLines({ competence: "202612", careProviderId: null });
  expect([ urlOf(2).searchParams.get("competence"), urlOf(2).searchParams.has("care_provider_id") ]).toEqual([ "202612", false ]);
});
it("planilha da FPO: só o base64 no corpo, nome fora", async () => {
  await uploadFpoImport(new File([ "abc" ], "fpo-dezembro.csv")); expect(bodyOf(0)).toEqual({ file_base64: "YWJj" });
  await applyFpoImport("fi1", 1); expect([ urlOf(1).pathname, bodyOf(1) ]).toEqual([ "/regulation/admin/fpo_imports/fi1/apply", { expected_version: 1 } ]);
});
it("remessa e fila nominal: listas vazias sem envelope", async () => {
  fetchMock.mockImplementationOnce(() => ok({})).mockImplementationOnce(() => ok({}));
  expect(await listApacExports({ healthUnitId: "u1", competence: null })).toEqual([]);
  expect(urlOf(0).search).toBe("?health_unit_id=u1");
  expect(await listOciQueueExports()).toEqual([]);
});
```

  Em `src/lib/features.test.ts`: `featureLabel("care_regulation_apac")` = `"Regulação por laudo (APAC, OCI e AIH)"`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod26b src/lib/api.regulationApac.test.ts src/lib/features.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces. O `searchRegulationTargets` ganha o `kind` opcional, e o `TargetSearch` o repassa (Task 4).
- [ ] **Step 4: Rode e veja passar; `tsc`.** Inclua `src/lib/api.regulation.test.ts` (o 26a não muda). Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.regulationApac.test.ts src/lib/features.ts src/lib/features.test.ts src/test/regulationApacFixtures.ts
$G commit -m "feat: add the regulation laudo, numbered authorization, FPO and APAC export client" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Frases, portão, rótulos, laudo, procedimentos, faixa e FPO

**Files:**
- Create: `src/lib/regulationApacErrors.ts`, `src/lib/regulationApacErrors.test.ts`, `src/lib/regulationApac.ts`, `src/lib/regulationApac.test.ts`

**Interfaces:**
- Consumes: `hasFeature`, `featureDisabledKey`; `canUseRegulation`, `canRequestRegulation`, `regulationRoles`, `regulationError`, `STALE_TEXT`, `STATIC_QUERY`, `NOMINAL_QUERY` (`src/lib/regulation.ts`); `normalizeCid10`, `cid10Problem`; `parseReais`, `fmtCents`; `competenceLabel`, `isCompetence`; `BILLING_CLERK_ROLE`; `ApiError`, `errorCode`.
- Produces (`src/lib/regulationApacErrors.ts`):
  - `REGULATION_APAC_DISABLED = "a regulação por laudo (APAC, OCI e AIH) está desligada nesta cidade ou ainda sem a SIGTAP completa"` (proposta);
  - `REGULATION_APAC_ERRORS: Record<string, string>`: uma frase por código, todas distintas, sem `_`, sem dado de pessoa;
  - `regulationApacErrorText(code)`: código ausente → `null` (quem chama cai no `regulationError` do 26a).

  **As frases são todas proposta.** Os códigos são do contrato (§) e das decisões (Q); pela Q10, os do plano do api (Valor 10 e Interfaces das Tasks 5–14). `invalid_transition`, `stale_version` e `quota_exhausted` ficam com as frases do 26a; `invalid_request` com `path` cai no campo (abaixo):

| Código | Frase |
|---|---|
| `invalid_target` (§1, Q6) | "este procedimento é autorizado por APAC ou AIH, e a regulação por laudo está desligada nesta cidade" |
| `incompatible_secondary` (§3) | "este secundário não é compatível com o principal na SIGTAP — troque ou retire" |
| `oci_needs_two_secondaries` (Q7) | "a OCI precisa de pelo menos 2 procedimentos secundários" |
| `oci_needs_consultation` (Q7) | "um dos secundários da OCI precisa ser consulta ou teleconsulta" |
| `oci_principal_not_group_09` (Q7) | "o principal da OCI é um procedimento do grupo 09" |
| `authorizer_not_allowed` (§4) | "você não pode autorizar este pedido: é preciso ser regulador de nível superior, com vínculo ativo numa unidade da cidade, e não ter feito o pedido" |
| `number_range_exhausted` (§4) | "a faixa de numeração deste tipo acabou — o administrador da cidade cadastra a nova faixa recebida da SES" |
| `number_range_overlap` (Q10) | "esta faixa se sobrepõe a outra já cadastrada" |
| `ceiling_exhausted` (§4) | "o teto do prestador nesta competência não cobre este procedimento" |
| `duplicate_fpo_line` (Q10) | "já existe linha da programação com este prestador, nível, código e financiamento nesta competência" |
| `apac_settings_changed` (Q13) | "o momento da autorização foi salvo por outra pessoa — foi relido; confira e salve de novo" |
| `fpo_line_changed` (Q13) | "esta linha da programação mudou enquanto você editava — foi relida; confira e salve de novo" |
| `fpo_import_changed` (Q13) | "a importação mudou enquanto a tela estava aberta — foi relida; confira e repita" |
| `expected_version_required` (Q10) | "a tela não mandou a versão — recarregue a página" |
| `nothing_to_export` (Q10) | "nenhuma APAC ou OCI desta unidade com procedimento realizado nesta competência" |
| `settings_missing` (Q10) | "faltam os dados do órgão emissor em Configuração do faturamento — complete antes de gerar a remessa" |
| `not_own_provider` (Q10) | "esta unidade não é prestador da rede própria — a remessa APAC é só da rede própria" |
| `feature_disabled` | `REGULATION_APAC_DISABLED` |

- Produces (`src/lib/regulationApac.ts`):
  - **Portão e papéis:** `canUseRegulationApac(user)`; `apacRoles(user): RegulationRoles & { clerk: boolean }`.
  - **Chaves:** `APAC_PREVIEW_KEY`, `APAC_OPTIONS_KEY`, `APAC_SETTINGS_KEY`, `NUMBER_RANGES_KEY`, `FPO_LINES_KEY`, `FPO_IMPORT_KEY`, `APAC_EXPORTS_KEY`, `OCI_QUEUE_EXPORTS_KEY`, `OCI_LIST_KEY`, `REVIEW_LIST_KEY`; reexporta `STATIC_QUERY` e `NOMINAL_QUERY`. A chave da prévia leva só `[ id, offerId, financing, faecProgram ]`.
  - **Tipo:** `kindOf(t: { authorization_kind?: string } | null): string` → ausente ou nulo = `"bpa"`; `isLaudoKind(k)` = `apac | oci | aih`; `INSTRUMENTS: { kind: AuthorizationKind; label: string }[]` na ordem BPA, APAC, OCI, AIH (rótulos de `kindLongLabel`, com "BPA (procedimento)" para o BPA).
  - **Rótulos** (proposta; código desconhecido cru):
    - `kindLabel`: `bpa` "BPA", `apac` "APAC", `oci` "OCI", `aih` "AIH"; `kindLongLabel`: `apac` "APAC (laudo)", `oci` "OCI (cuidado integrado)", `aih` "AIH (internação eletiva)";
    - `stageLabel`/`stageTone`: `issued` "número emitido" (ok), `awaiting_review` "aguardando a revisão" (warn), `review_denied` "negada na revisão" (down);
    - `financingLabel`: `mac` "MAC (teto do prestador)", `faec` "FAEC", `credits` "créditos do programa federal (fora do teto)", `other` "outro financiamento (fora do teto)" (Q9);
    - `levelLabel`: `procedure` "procedimento", `organization_form` "forma de organização", `subgroup` "subgrupo", `group` "grupo";
    - `characterLabel`: `elective` "eletivo", `urgent` "urgência"; `momentLabel`: `before` "antes do atendimento", `after` "depois do atendimento comprovado"; `outcomeLabel`: `concluded` "concluída", `not_concluded` "não concluída";
    - `performedLabel(p)`: sem `performed_at` "a fazer"; `performed_procedure` "realizado (prontuário) em <dd/mm/aaaa>"; `manual` "marcado pela regulação em <dd/mm/aaaa>";
    - `optionLabel(code, list: CodeName[])`: o nome da lista, ou o código cru.
  - **Formato:** `sigtapOf(raw): string | null` (10 dígitos depois de tirar `.`, `-`, `/` e espaço); `fmtSigtap`; `quantityOf(raw): number | null` (inteiro 1–999, Q12); `fmtNumber13(n)` devolve o número cru (13 dígitos, sem agrupar: é o que se digita no SISAIH01 e no APAC-Mag).
  - **Laudo:** `type LaudoDraft = { summary: string; cidSecondary: string; findings: string; character: AihCharacter | ""; clinic: string }`; `EMPTY_LAUDO`; `laudoDraftFrom(l: RegulationLaudo)`; `LAUDO_SUMMARY_MIN = 20`, `LAUDO_SUMMARY_MAX = 4000`, `LAUDO_FINDINGS_MAX = 4000` (Q12); `laudoProblems(d, kind, cid10: string): { field: LaudoField | "cid10"; text: string }[]`: CID-10 vazio → "o pedido por laudo exige o CID-10 principal" (Q6: o CID principal é o `cid10` do pedido, obrigatório para todo CBO nos tipos por laudo); resumo fora de 20–4000 → "o resumo clínico do laudo tem de 20 a 4000 caracteres"; CID secundário válido se preenchido; achados até 4000; na AIH, caráter e clínica com 2 dígitos obrigatórios. `laudoBody(d, kind): RegulationLaudoInput` (sem espaços nas pontas, nunca cortado; `character`/`clinic` só na AIH; nunca o CID principal).
  - **Procedimentos:** `type ProcedureLine = { key: string; code: string; name: string | null; quantity: string }`; `PROCEDURES_MAX = 30` (Q12); `proceduresProblems(kind, principal: string, lines: ProcedureLine[]): { index: number; text: string }[]` (`index` conta o principal como 0): mais de 30 → "no máximo 30 procedimentos por pedido"; na OCI, menos de 2 secundários → a frase de `oci_needs_two_secondaries`; código repetido (inclusive o do principal) → "este procedimento já está no pedido"; SIGTAP fora do formato; quantidade fora de 1–999 → "quantidade inteira de 1 a 999". A regra da consulta (Q7) é do api (`oci_consultation_codes.yml`): a tela não a repete. `proceduresBody(principal, principalQty, lines): RequestedProcedureInput[]` (principal primeiro); `procedureIndexOf(err): number | null` (lê `path` `procedures[<n>]` e `procedures[<n>].quantity`).
  - **Faixa:** `FIFTH_DIGITS: Record<"apac" | "aih", { digit: string; label: string }[]>` (APAC `2` "geral", `7` "OCI", `6` "eletiva", `9` "créditos"; AIH `1` "geral", `5` "eletiva", `8` "créditos"; os rótulos são da decisão 7 da spec); `type RangeDraft = { kind: "apac" | "aih"; fifthDigit: string; year: string; first: string; last: string }`; `rangeProblems(d, today): { field: string; text: string }[]`: sequencial com mais de 7 dígitos → "digite só o sequencial (até 7 dígitos, depois do 5º dígito) — o número inteiro do ofício da SES tem 13 dígitos" (Q8); fora de 1–9999999 ou primeiro > último → "a faixa precisa de primeiro e último números de 1 a 9999999, com o primeiro menor ou igual ao último"; 5º dígito fora do tipo → "o 5º dígito não serve para este tipo: APAC usa 2, 7, 6 ou 9; AIH usa 1, 5 ou 8"; ano diferente do corrente e do seguinte → "o ano da faixa é o corrente ou o seguinte"; `rangeBody(d): NumberRangeInput`; `rangeFieldOf(err)` (o `path` do `invalid_request`: `first_number`, `last_number`, `fifth_digit`, `year`).
  - **FPO:** `FPO_LEVELS`, `FINANCINGS` (`mac`, `faec`, `credits`: o `other` nunca é escolhido na linha); `fpoLineProblems(d)` (competência `AAAAMM`, código com o tamanho do nível — 10, 6, 4 ou 2 dígitos —, FAEC exige o programa e é só por procedimento (Q9) → "a linha FAEC é só por procedimento", quantidade inteira ≥ 0, valor pelo `parseReais`); `balanceText(l: FpoLine)` → "saldo R$ … de R$ … (reservado R$ …)".
  - **Planilha da FPO** (Q2, Q10; molde do 27b): `FPO_IMPORT_HEADER = "competencia;cnes;nivel;codigo;financiamento;programa_faec;quantidade;valor"` (plano do api, Task 7), `FPO_IMPORT_MAX_BYTES = 5 * 1024 * 1024`; `fpoFileProblem(file)` (sem arquivo, outra extensão, acima de 5 MB; as frases do 27b com "programação" no lugar de "saldo"); `fpoHeaderProblem(firstLine)` (tira BOM e `\r`); `readFirstLine(file)`; `FPO_ROW_ERRORS` e `fpoRowErrorText` (desconhecido cru), com os códigos do plano do api: `provider_not_found` "CNES sem prestador da cidade (a rede própria vale pelo CNES da unidade)", `invalid_level` "nível deve ser procedimento, forma, subgrupo ou grupo", `invalid_code` "código fora da SIGTAP da competência ou do tamanho do nível", `invalid_financing` "financiamento deve ser MAC, FAEC ou créditos", `faec_not_procedure` "a linha FAEC é só por procedimento", `invalid_quantity` "quantidade inteira maior ou igual a zero", `invalid_amount` "valor com vírgula, maior ou igual a zero", `duplicate_line` "linha repetida (mesma competência, prestador, nível, código e financiamento)"; `FPO_IMPORT_ERRORS` com `invalid_file` e `file_too_large` (as frases do 27b); `fpoStatusLabel`: `validated` "conferida", `applied` "aplicada", `discarded` "descartada".
  - **OCI:** `ociAlarms(item: OciItem): string[]` (proposta): `oncology_alert` → "OCI de oncologia aberta há <days_open> dias, contados do primeiro atendimento — o prazo é de 30 dias corridos"; `expired` → "OCI vencida — encerre como concluída ou não concluída" (Q7); `ociProgress(item)` → "<feitos> de <total> secundários".
  - **Recusas:** `regulationApacError(err)`: `TypeError` → "sem conexão com o servidor — tente de novo"; `featureDisabledKey` → `REGULATION_APAC_DISABLED`; código da tabela → a frase; `invalid_request` com `path` `laudo.<campo>`, `procedures…`, `budget_source`, `exit_reason`, `outcome` ou `performed_at` → a frase do campo (`exit_reason` "escolha o motivo de saída da lista oficial"; `outcome` "para concluir, os secundários e a consulta precisam estar realizados"; `performed_at` "a data do atendimento fica dentro da validade e não passa de hoje"; `budget_source` "os créditos exigem a fonte orçamentária"; o laudo, "confira o campo: <rótulo>"); o resto → `regulationError` do 26a. `isStale(err)` = `stale_version`, `invalid_transition` ou um dos `*_changed`.

- [ ] **Step 1: Testes que falham.**
  - `src/lib/regulationApacErrors.test.ts`: cada código da tabela tem frase; as frases são distintas; nenhuma tem `_`; código ausente → `null`.
  - `src/lib/regulationApac.test.ts`:

```ts
it.each([ [ undefined, "bpa" ], [ null, "bpa" ], [ "apac", "apac" ], [ "oci", "oci" ], [ "bpa_c", "bpa_c" ] ])("kindOf(%j) = %j", (k, out) =>
  expect(kindOf(k === undefined ? {} : { authorization_kind: k as string })).toBe(out));
it("rótulos desconhecidos aparecem crus", () => {
  expect([ kindLabel("sus_x"), stageLabel("paused"), financingLabel("ppi"), levelLabel("rubric"), outcomeLabel("partial") ])
    .toEqual([ "sus_x", "paused", "ppi", "rubric", "partial" ]);
});
it("laudo: CID-10 obrigatório, resumo 20–4000, AIH com caráter e clínica de 2 dígitos", () => {
  const d = { ...EMPTY_LAUDO, summary: "curto" };
  expect(laudoProblems(d, "apac", "H251").map((p) => p.field)).toEqual([ "summary" ]);
  expect(laudoProblems({ ...d, summary: "x".repeat(20) }, "apac", "").map((p) => p.field)).toEqual([ "cid10" ]);
  expect(laudoProblems({ ...d, summary: "x".repeat(4001) }, "apac", "H251")[0].text).toBe("o resumo clínico do laudo tem de 20 a 4000 caracteres");
  expect(laudoProblems({ ...d, summary: "x".repeat(20), clinic: "1" }, "aih", "H251").map((p) => p.field)).toEqual([ "character", "clinic" ]);
  expect(laudoBody({ ...d, summary: `  ${"x".repeat(20)}  `, character: "urgent", clinic: "01" }, "apac")).toEqual({ clinical_summary: "x".repeat(20) });
});
it("OCI: menos de 2 secundários, repetido, quantidade e teto de 30, com o índice contando o principal", () => {
  const l = (code: string, quantity = "1") => ({ key: code, code, name: null, quantity });
  expect(proceduresProblems("oci", "0901010014", [ l("0301010072") ]).map((p) => p.text)).toEqual([ "a OCI precisa de pelo menos 2 procedimentos secundários" ]);
  expect(proceduresProblems("oci", "0901010014", [ l("03.01.01.007-2"), l("0301010072"), l("0211020036", "2,5") ]))
    .toEqual([ { index: 2, text: "este procedimento já está no pedido" }, { index: 3, text: "quantidade inteira de 1 a 999" } ]);
  const many = Array.from({ length: 30 }, (_, i) => l(`03010100${String(10 + i)}`));
  expect(proceduresProblems("apac", "0405050372", many)[0].text).toBe("no máximo 30 procedimentos por pedido");
  expect(procedureIndexOf(new ApiError(422, { error: "incompatible_secondary", path: "procedures[2]" }, ""))).toBe(2);
});
it("faixa: número inteiro colado, ordem, 5º dígito do tipo e ano", () => {
  const d = { kind: "aih" as const, fifthDigit: "7", year: "2026", first: "4126200000015", last: "10" };
  expect(rangeProblems(d, TODAY26B).map((p) => p.field)).toEqual([ "fifthDigit", "first" ]);
  expect(rangeProblems({ ...d, fifthDigit: "5", first: "200", last: "100" }, TODAY26B)[0].text).toMatch(/primeiro menor ou igual/);
  expect(rangeProblems({ ...d, fifthDigit: "5", first: "1", last: "9999999", year: "2025" }, TODAY26B)[0].field).toBe("year");
});
it("FPO: FAEC só por procedimento; planilha com BOM e \\r passa, com vírgula recusa", () => {
  expect(fpoLineProblems({ competence: "202612", level: "group", code: "04", financing: "faec", faecProgram: "0001", quantity: "1", amount: "10,00" })
    .map((p) => p.text)).toEqual([ "a linha FAEC é só por procedimento" ]);
  expect(fpoHeaderProblem(`﻿${FPO_IMPORT_HEADER}\r`)).toBeNull(); expect(fpoHeaderProblem(FPO_IMPORT_HEADER.replace(/;/g, ","))).not.toBeNull();
});
it("recusas: frase do 26b, campo do laudo e da OCI, e queda no 26a", () => {
  expect(regulationApacError(new ApiError(409, { error: "ceiling_exhausted" }, ""))).toBe("o teto do prestador nesta competência não cobre este procedimento");
  expect(regulationApacError(new ApiError(422, { error: "invalid_request", path: "laudo.clinical_summary" }, ""))).toBe("confira o campo: resumo clínico");
  expect(regulationApacError(new ApiError(422, { error: "invalid_request", path: "exit_reason" }, ""))).toBe("escolha o motivo de saída da lista oficial");
  expect(regulationApacError(new ApiError(409, { error: "stale_version" }, ""))).toBe(STALE_TEXT);
});
```

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod26b src/lib/regulationApacErrors.test.ts src/lib/regulationApac.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/regulationApacErrors.ts src/lib/regulationApacErrors.test.ts src/lib/regulationApac.ts src/lib/regulationApac.test.ts
$G commit -m "feat: add the regulation laudo phrases, labels and the laudo, procedure, range and FPO rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Itens do menu, portão e casca das telas

**Files:**
- Create: `src/shell/modules.regulationApac.test.ts`, `src/modules/regulation/apac/ApacGate.tsx`, e as cascas (portão e título; o corpo vem nas Tasks 6–10) `src/modules/RegulationApacSettings.tsx`, `src/modules/RegulationFpo.tsx`, `src/modules/RegulationOci.tsx`, `src/modules/RegulationApacReview.tsx`, `src/modules/RegulationApacExports.tsx`, `src/modules/RegulationOciQueue.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/catalog.ts`, `src/App.tsx`

**Interfaces:**
- Consumes: `canUseRegulationApac`, `apacRoles`, `REGULATION_APAC_DISABLED` (Task 2); `getApacSettings` (Task 1).
- Produces:
  - `ApacGate({ title, sub, allow, children })`: o molde do `RegulationGate`; sem `canUseRegulation` → a frase do 26a; sem `care_regulation_apac` → `EmptyState` com `REGULATION_APAC_DISABLED`; sem papel → `REGULATION_NO_ROLE`. Nenhuma leitura antes do portão.
  - `ModuleId` + seis ids; no grupo **Regulação**, depois de `regulation-settings` (e antes de `regulation-rnds`, se existir). Rótulos fixos ou proposta:

| id | Rótulo | Ícone | Quem vê (com `regOn` e `care_regulation_apac`) |
|---|---|---|---|
| `regulation-apac-settings` | "Numeração e momento da autorização" (proposta) | `IconNumbers` | admin |
| `regulation-fpo` | "Programação e teto (FPO)" (proposta) | `IconReportMoney` | admin, regulador |
| `regulation-oci` | "OCI em andamento" | `IconStethoscope` | regulador |
| `regulation-apac-review` | "Revisão de APAC" | `IconClipboardCheck` | regulador, só no modo `after` (Q13) |
| `regulation-apac-exports` | "Remessa APAC" | `IconFileExport` | admin, `billing_clerk` (Q10) |
| `regulation-oci-queue` | "Fila nominal da OCI" | `IconListDetails` | admin |

  - Descrições do `CATALOG` (proposta): "Faixas recebidas da SES e quando o número sai"; "Teto em dinheiro por prestador e competência, com a planilha"; "Secundários, encerramento e prazo da oncologia"; "Autorizar a APAC depois do atendimento comprovado"; "Arquivo da APAC da rede própria por competência"; "Lista mensal da OCI para o Ministério".
  - **O item "Revisão de APAC" e o modo:** o menu lê `getApacSettings` (`STATIC_QUERY`, chave `APAC_SETTINGS_KEY`) só para o regulador com o interruptor; a leitura recusada ou ausente esconde o item (proposta; o regulador lê o momento pela Q13). A tela confere de novo e mostra "a cidade autoriza antes do atendimento — não há revisão" (proposta) no modo `before`.
  - `renderModule`: um `case` por id, com `onGoToSecurity={() => setActive("security")}` nas telas com step-up e `today` onde a tela usa a data.

- [ ] **Step 1: Testes que falham** (`src/shell/modules.regulationApac.test.ts`):
  - com o interruptor e o modo `after`, o regulador vê `regulation-fpo`, `regulation-oci` e `regulation-apac-review`; o admin, `regulation-apac-settings`, `regulation-fpo`, `regulation-apac-exports` e `regulation-oci-queue`; o faturista, só `regulation-apac-exports` no grupo Regulação;
  - no modo `before`, o regulador não vê "Revisão de APAC";
  - sem `care_regulation_apac`, a lista do grupo Regulação é a de hoje (a do 26a e do 26c);
  - o profissional e o operador não veem nenhum;
  - cada rótulo aparece uma vez só em `NAV_GROUPS`, e o `CATALOG` tem as seis descrições.
- [ ] **Step 2–4:** rode e veja falhar (o arquivo e `src/shell/modules.regulation.test.ts src/shell/catalog.test.ts`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.regulationApac.test.ts src/shell/catalog.ts src/App.tsx src/modules/regulation/apac/ApacGate.tsx src/modules/RegulationApacSettings.tsx src/modules/RegulationFpo.tsx src/modules/RegulationOci.tsx src/modules/RegulationApacReview.tsx src/modules/RegulationApacExports.tsx src/modules/RegulationOciQueue.tsx
$G commit -m "feat: add the APAC, OCI and AIH screens to the regulation group behind care_regulation_apac" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Pedido por laudo na consulta e no complemento; laudo em PDF

**Files:**
- Create: `src/modules/regulation/apac/LaudoFields.tsx`, `src/modules/regulation/apac/ProceduresField.tsx`, `src/modules/regulation/RequestForm.laudo.test.tsx`, `src/modules/consultation/RegulationRequestsField.laudo.test.tsx`
- Modify: `src/modules/regulation/RequestForm.tsx`, `src/modules/regulation/TargetSearch.tsx` (prop `kind`), `src/lib/regulation.ts` (`RequestDraft` e `draftFromRequest` levam o instrumento, o laudo e os procedimentos), `src/modules/regulation/openGuide.ts`, `src/modules/consultation/RegulationRequestsField.tsx`, `src/modules/RegulationReturned.tsx`

**Interfaces:**
- Consumes: `kindOf`, `isLaudoKind`, `INSTRUMENTS`, `kindLongLabel`, `EMPTY_LAUDO`, `laudoDraftFrom`, `laudoProblems`, `laudoBody`, `proceduresProblems`, `proceduresBody`, `procedureIndexOf`, `sigtapOf`, `fmtSigtap`, `regulationApacError`, `canUseRegulationApac`, `APAC_OPTIONS_KEY` (Task 2); `getApacOptions`, `fetchRegulationLaudoPdf`, `searchRegulationTargets(q, type, kind)` (Task 1); `signatureMarker` (`src/lib/signature.ts`).
- Produces:
  - `RequestDraft` ganha `kind: AuthorizationKind`, `laudo: LaudoDraft | null` e `procedures: ProcedureLine[]`; `EMPTY_DRAFT` e `draftFromRequest` os preenchem (`"bpa"`, `null` e `[]` no BPA). `requestBody` não muda para o BPA.
  - `TargetSearch` ganha `kind?: AuthorizationKind | null` (repassado ao `searchRegulationTargets`; entra na chave da busca). Sem `kind`, a busca é a de hoje.
  - `RequestForm` (prop nova `apac: boolean`, de `canUseRegulationApac(user)`):
    - com `apac` falso, ou no tipo "Especialidade", o formulário é o de hoje, letra por letra;
    - com `apac` e o tipo "Procedimento", aparece "Instrumento" (`radio` por `INSTRUMENTS`: "BPA (procedimento)", "APAC (laudo)", "OCI (cuidado integrado)", "AIH (internação eletiva)"), e o `TargetSearch` busca com o `kind` escolhido (Q6). Trocar o instrumento limpa o alvo, o laudo e os secundários;
    - no instrumento por laudo, a nota (proposta) "Pedido por laudo: o regulador autoriza com o número de APAC ou AIH e reserva o valor no teto do prestador.";
    - o campo "CID-10" vira "CID-10 principal", obrigatório para todo CBO nos tipos por laudo; o laudo não leva CID principal (Q6). A "Justificativa" do 26a fica;
    - `LaudoFields({ kind, draft, options, onChange, fieldRef })`: "Resumo clínico (laudo)" (`textarea`, contador "n de 4000"), "CID-10 secundário (opcional)", "Achados e exames (opcional)" (até 4000); na AIH, "Caráter da internação" (`select`: "eletivo", "urgência") e "Clínica" (`select` com `options.aih_clinics`, D17);
    - `ProceduresField({ kind, principal, lines, problems, onChange })`: o principal (o alvo) com a quantidade (padrão 1); na OCI, "Secundários previstos": "Acrescentar secundário" abre a busca de procedimento (o `TargetSearch` com `kind` `bpa`, sem filtro de CBO), e cada linha tem código (`fmtSigtap`), nome, quantidade e "Retirar"; a nota (proposta) "Pelo menos 2 secundários, um deles consulta ou teleconsulta. O regulador marca cada um como realizado.";
    - antes de enviar: `requestProblem` do 26a, `laudoProblems(d, kind, cid10)` e `proceduresProblems`; a primeira recusa recebe o foco, e cada frase fica abaixo do seu campo ou da sua linha;
    - corpo: `requestBody` + `authorization_kind: kind`, `laudo: laudoBody(...)` e `procedures: proceduresBody(...)` (na APAC e na AIH, só o principal);
    - recusa do api: `incompatible_secondary` cai na linha de `procedureIndexOf`; `oci_needs_two_secondaries`, `oci_needs_consultation` e `oci_principal_not_group_09` (Q7) ficam acima dos secundários; `invalid_request` em `laudo.<campo>` ou `cid10` cai no campo; o resto, pela `regulationApacError`; o digitado fica.
  - `RegulationRequestsField`: a linha do pedido mostra a `Tag` do tipo quando não é BPA; nos pedidos por laudo já enviados, "Laudo em PDF" ao lado da "Guia em PDF" e o marcador da assinatura do laudo (`signatureMarker(laudo.signature)`), com a frase de papel (proposta) "sem certificado ou sem CPF do paciente: imprima o laudo e assine à mão" quando `mode` = `manual` (Q12).
  - `openLaudo(id): Promise<GuideOutcome>` em `openGuide.ts`, com o mesmo `window.open` antes do `await`, o `blob:` revogado em 60 s e `LAUDO_NOT_FOUND = "laudo não encontrado"` (proposta).
  - `RegulationReturned`: o complemento de um pedido por laudo abre o mesmo formulário com o laudo e os procedimentos lidos e envia `laudo` e `procedures` junto (Q13). O "CID-10 principal" fica só leitura (Q6: o complemento não troca o CID do laudo já assinado).

- [ ] **Step 1: Testes que falham.**
  - `RequestForm.laudo.test.tsx` (molde `RequestForm.test.tsx`; `vi.mock` do api com `searchRegulationTargets` e `getApacOptions`):
    - "BPA e interruptor desligado: o formulário do 26a": com `apac: false`, nenhum "Instrumento" nem campo do laudo, e a busca sem `kind`; com `apac` e "BPA (procedimento)", `ECO_BPA` sem laudo;
    - "APAC: instrumento, busca com kind e laudo no corpo": "APAC (laudo)" → `searchRegulationTargets(<termo>, "procedure", "apac")`; `CATARATA`, CID `"h25.1"`, resumo de 25 caracteres → `onSubmit` com `authorization_kind: "apac"`, `cid10: "H251"`, `laudo: { clinical_summary }` (sem CID principal) e `procedures: [ { sigtap_code: "0405050372", role: "principal", quantity: 1 } ]`;
    - "resumo curto ou CID-10 vazio recusados antes, com foco no campo": 19 caracteres → a frase; enfermeiro sem CID-10 num pedido por laudo → "o pedido por laudo exige o CID-10 principal"; nenhuma chamada;
    - o caso do Review Focus 2, "OCI: secundários conferidos antes e recusa do api na linha certa": um secundário só → a frase; o mesmo código duas vezes → a frase na linha 3; `"2,5"` → a frase da quantidade; com tudo certo, `422 oci_needs_consultation` → a frase acima dos secundários; noutra rodada, `422 incompatible_secondary` `path: "procedures[2]"` → a frase na segunda linha de secundário; o digitado fica;
    - "AIH: caráter e clínica obrigatórios": sem eles, as duas frases; com eles, `laudo.character: "elective"` e `laudo.clinic: "01"`;
    - "trocar o instrumento limpa o alvo e o laudo";
    - "o laudo nunca vai a chave de cache": depois de digitar e enviar, nenhuma chave do `queryClient` contém o resumo nem `H251`.
  - `RegulationRequestsField.laudo.test.tsx`: a `Tag` "OCI (cuidado integrado)"; "Laudo em PDF" abre `fetchRegulationLaudoPdf("rr1")` numa janela só no duplo clique; `404` → "laudo não encontrado"; marcador `manual` → a frase de papel; o rascunho por laudo abre em "Editar" com o instrumento, o laudo e os secundários preenchidos.
  - Em `RegulationReturned.test.tsx`, um caso a mais: o complemento de um pedido por laudo envia `laudo` e `procedures`, e o CID-10 principal aparece só leitura.
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos, `src/modules/regulation/RequestForm.test.tsx`, `src/modules/regulation/TargetSearch.test.tsx`, `src/modules/RegulationReturned.test.tsx`, `src/modules/consultation/ConsultationWorkspace.regulation.test.tsx`); implemente; rode e veja passar (o 26a sem mudar asserção); `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/regulation/RequestForm.tsx src/modules/regulation/TargetSearch.tsx src/lib/regulation.ts src/modules/regulation/apac/LaudoFields.tsx src/modules/regulation/apac/ProceduresField.tsx src/modules/regulation/openGuide.ts src/modules/consultation/RegulationRequestsField.tsx src/modules/RegulationReturned.tsx src/modules/regulation/RequestForm.laudo.test.tsx src/modules/consultation/RegulationRequestsField.laudo.test.tsx src/modules/RegulationReturned.test.tsx
$G commit -m "feat: request APAC, OCI and AIH with the laudo in the consultation and open the laudo PDF" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Tipo na fila, laudo no detalhe e autorização com teto e número

**Files:**
- Create: `src/modules/regulation/apac/AuthorizationPreview.tsx`, `src/modules/regulation/apac/LaudoView.tsx`, `src/modules/RegulationQueue.apac.test.tsx`, `src/modules/regulation/AuthorizeForm.apac.test.tsx`
- Modify: `src/modules/regulation/RequestTable.tsx`, `src/modules/regulation/RequestDetail.tsx`, `src/modules/regulation/AuthorizeForm.tsx`

**Interfaces:**
- Consumes: `getAuthorizationPreview`, `authorizeRegulationRequest`, `getApacOptions` (Task 1); `kindOf`, `kindLabel`, `stageLabel`, `stageTone`, `financingLabel`, `levelLabel`, `performedLabel`, `optionLabel`, `fmtNumber13`, `balanceText`, `regulationApacError`, `isStale`, `APAC_PREVIEW_KEY` (Task 2); `fmtCents`, `competenceLabel`; `SensitiveAction`; `reasonProblem` do 26a.
- Produces:
  - `RequestTable`: em "Encaminhamento", abaixo do tipo do 26a, a `Tag` `kindLabel` quando não é BPA. A tabela nunca mostra o laudo.
  - `LaudoView({ laudo, cid10, kind })` no `RequestDetail`, depois da justificativa: "CID-10 principal" (o `cid10` do pedido), resumo, CID secundário, achados e, na AIH, caráter e clínica (`KeyValue`); "Laudo em PDF" ao lado da "Guia em PDF". Os procedimentos em lista: código (`fmtSigtap`), nome, papel, quantidade e `performedLabel`.
  - O bloco da autorização ganha (quando vierem): "Número" (`fmtNumber13`, `mono`), "Etapa" (`Tag` de `stageLabel`), "Validade" (`competenceLabel` de/até), "Financiamento" (e "Fonte orçamentária" nos créditos), "Linha da FPO" (`levelLabel` + código), "Reservado" (`fmtCents`), a `Tag` "acima do teto" (tom `warn`, proposta) quando `ceiling_overridden`, e a `Tag` "reserva na competência antiga" (tom `warn`, proposta) quando `reservation_mismatch`. Sem número na etapa `awaiting_review`, "o número sai na revisão, depois do atendimento comprovado" (proposta).
  - O cancelamento de um pedido com `reserved_cents` diz (proposta) "Pedido cancelado; a vaga volta para a cota do mês e a reserva volta ao teto do prestador."
  - `AuthorizeForm`, quando `kindOf(request)` não é `bpa` **ou** a prévia diz `ceiling_applies` (Q5: o teto vale também no BPA com FPO):
    - **Mês:** nos pedidos por laudo, "Mês previsto do atendimento" (`select` com o mês corrente e os 2 seguintes, `monthLabel`), e as ofertas lidas são as desse mês (Q13); no BPA, só o corrente, como hoje;
    - **Prévia:** escolhida a oferta, `AuthorizationPreview` lê `getAuthorizationPreview(request.id, { offerId, financing, faecProgram })` (`NOMINAL_QUERY`, relida a cada troca de oferta ou de financiamento) e mostra: "Competência", "Valor SIGTAP" (`unit_cents` × `quantity`), "Linha da FPO" (`levelLabel` + código, ou "sem programação para este prestador — fora do teto", proposta), "Saldo" (`balance_cents`), "Reserva desta autorização" (`reserve_cents`) e, se `numbered`, "Número da faixa <5º dígito>, restam <range_remaining>" (proposta). Sem teto (`ceiling_applies: false`: prestador `ppi`, pedido do estado ou financiamento `other`), só a nota "fora do teto: sem reserva nesta autorização" (proposta);
    - **Financiamento:** `select` "Financiamento" com `mac`, `faec` e `credits`, começando no `financing_default` (o `other`, quando é o padrão, aparece como a opção já escolhida, Q9); FAEC pede o "Programa FAEC" (`options.faec_programs`); créditos pedem a "Fonte orçamentária" (`options.budget_sources`, Q9); trocar relê a prévia;
    - **Saldo menor que a reserva** (pela prévia, antes de enviar): o "Confirmar autorização" fica desabilitado, com a frase de `ceiling_exhausted` e o botão "Autorizar acima do teto";
    - **"Autorizar acima do teto"**: `SensitiveAction` (`requiresStepUp`; título "Autorizar acima do teto"; campo "Motivo para passar do teto", 10–500, obrigatório; descrição proposta: "A autorização passa do teto programado do prestador e fica marcada. Use só quando o caso não pode esperar a próxima competência.") → `authorizeRegulationRequest` com `ceiling_override_reason`;
    - **Recusas:** `409 ceiling_exhausted` → a frase, **uma** releitura da prévia e o botão "Autorizar acima do teto"; `409 number_range_exhausted` e `403 authorizer_not_allowed` → a frase, sem nova tentativa; `invalid_request` em `budget_source` → a frase no campo; `stale_version`/`invalid_transition` → `onStale` como hoje; `quota_exhausted` → como hoje;
    - **Sucesso** (proposta): com número, "Autorizado em <prestador>. Número <número> — informe ao prestador para faturar."; na etapa `awaiting_review`, "Acesso liberado em <prestador>, com o valor reservado. O número sai na Revisão de APAC, depois do atendimento."
  - O `AuthorizeForm` do BPA sem FPO fica como hoje: o teste do 26a não muda.

- [ ] **Step 1: Testes que falham.**
  - `AuthorizeForm.apac.test.tsx`:
    - "prévia: linha, saldo, reserva e número da faixa": `apacRequest()`, oferta `of1` → `getAuthorizationPreview("rr1", { offerId: "of1", financing: null, faecProgram: null })` uma vez; "R$ 771,34", "saldo", "restam 9000";
    - "mês previsto: as ofertas do mês escolhido": trocar para o mês seguinte → `listCareOffers({ month: "2027-01", targetCode: "0405050372" })`;
    - "FAEC exige o programa e créditos exigem a fonte orçamentária": sem eles, a frase e nenhuma autorização; com eles, nova prévia e `budget_source: "02"` no corpo;
    - o caso do Review Focus 1, "autorizar: teto esgotado, acima do teto com step-up, um pedido, faixa esgotada e autorizador recusado": com `exhaustedPreview()`, "Confirmar autorização" desabilitado e "Autorizar acima do teto" visível; motivo de 9 caracteres recusado; duplo clique em "Confirmar" → uma chamada com `ceiling_override_reason`; `401 mfa_required` volta ao step-up sem nova chamada; noutra rodada, `409 number_range_exhausted` → a frase; `403 authorizer_not_allowed` → a frase;
    - "corrida do teto: 409 relê a prévia uma vez": prévia com saldo, POST `409 ceiling_exhausted` → a frase e `getAuthorizationPreview` lido de novo uma vez;
    - "sucesso com número e no modo depois": as duas frases;
    - "pedido do estado, ppi ou outro financiamento: fora do teto": `ceiling_applies: false` → a nota e nenhuma linha de saldo.
  - `RegulationQueue.apac.test.tsx`:
    - "fila: tipo na linha e nenhum laudo": uma linha `apacRequest()` com `laudo()` → a `Tag` "APAC"; nem o resumo nem `H251` no `body`;
    - "detalhe: laudo, procedimentos e autorização emitida": `getRegulationRequest` com `issuedAuthorization()` → "4126200000015", "número emitido", "12/2026" a "01/2027", "R$ 771,34";
    - "tipo, etapa e financiamento desconhecidos aparecem crus" (Review Focus 5);
    - "reserva na competência antiga: a marca"; "cancelar com reserva: a frase da reserva".
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos, `src/modules/regulation/RequestDetail.test.tsx`, `src/modules/RegulationQueue.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/regulation/RequestTable.tsx src/modules/regulation/RequestDetail.tsx src/modules/regulation/AuthorizeForm.tsx src/modules/regulation/apac/AuthorizationPreview.tsx src/modules/regulation/apac/LaudoView.tsx src/modules/RegulationQueue.apac.test.tsx src/modules/regulation/AuthorizeForm.apac.test.tsx
$G commit -m "feat: authorize laudo requests with the FPO line, the reserve, the number and the over-ceiling override" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Faixas de numeração e momento da autorização

**Files:**
- Create: `src/modules/regulation/apac/NumberRangesPanel.tsx`, `src/modules/regulation/apac/MomentPanel.tsx`, `src/modules/RegulationApacSettings.test.tsx`
- Modify: `src/modules/RegulationApacSettings.tsx`

**Interfaces:**
- Consumes: `listNumberRanges`, `createNumberRange`, `setNumberRangeActive`, `getApacSettings`, `updateApacSettings` (Task 1); `FIFTH_DIGITS`, `rangeProblems`, `rangeBody`, `rangeFieldOf`, `momentLabel`, `regulationApacError`, `isStale`, as chaves (Task 2); `ApacGate` (Task 3).
- Produces: `RegulationApacSettings({ today, onGoToSecurity })`, `ApacGate title="Numeração e momento da autorização" sub="faixas recebidas da SES e quando o número sai" allow={(r) => r.admin}` (proposta):
  - `MomentPanel` (`Panel title="Momento da autorização"`, proposta): dois `radio` com `momentLabel` e as notas (proposta) "Antes: o número sai quando o regulador libera o acesso (modelo nacional)." e "Depois: o valor é reservado na liberação, e o número sai na Revisão de APAC, com o atendimento comprovado."; "Salvar momento" → `SensitiveAction` (`requiresStepUp`) → `updateApacSettings({ authorization_moment, expected_version })`. `409 apac_settings_changed` → a frase, **uma** releitura, a escolha mantida e a versão nova no próximo envio.
  - `NumberRangesPanel` (`Panel title="Faixas de numeração"`, proposta):
    - `DataTable`: "Tipo" (APAC/AIH e o rótulo do 5º dígito), "Ano", "Do número" (`first_full`), "Ao número" (`last_full`), "Próximo" (sequencial), "Restam" (`remaining`, `Tag` `warn` abaixo de 100, proposta), "Situação" ("ativa"/"inativa") e a ação "Desativar"/"Ativar" (`SensitiveAction`);
    - "Cadastrar faixa": "Tipo" (APAC/AIH), "5º dígito" (`select` com `FIFTH_DIGITS[tipo]`), "Ano", "Primeiro sequencial", "Último sequencial", com a ajuda (proposta) "O ofício da SES traz o número inteiro (13 dígitos). Digite só o sequencial: os 7 dígitos depois do 5º dígito, sem o dígito verificador. O 5º dígito de cada autorização é o sistema que escolhe.";
    - `rangeProblems` antes, cada frase no seu campo; depois `SensitiveAction` (`requiresStepUp`) com a descrição "Faixa <tipo> <5º dígito>/<ano>, do <primeiro> ao <último>: <n> números." → `createNumberRange`;
    - `409 number_range_overlap` → a frase; `422 invalid_request` com `path` → a frase no campo de `rangeFieldOf`; o digitado fica; sucesso → "Faixa cadastrada." (proposta) e releitura.

- [ ] **Step 1: Testes que falham** (`RegulationApacSettings.test.tsx`, sessão admin):
  - "faixas: lista com o número inteiro e o que resta": duas faixas; "4126200000001", "restam 9000";
  - o caso do Review Focus 4, "faixa: número inteiro colado, ordem, 5º dígito do tipo e sobreposição": `"4126200000015"` no primeiro → a frase da Q8 e nenhuma chamada; AIH com `7` → o `select` não oferece o `7`; primeiro 200 e último 100 → a frase; com tudo certo, `409 number_range_overlap` → a frase e os campos mantidos;
  - "cadastrar: step-up e um pedido": duplo clique em "Confirmar" → `createNumberRange({ kind: "apac", fifth_digit: "2", year: 2026, first_number: 1, last_number: 10000 })` uma vez;
  - "desativar com step-up": `setNumberRangeActive("nr1", false)`;
  - "momento: 409 relê uma vez, mantém a escolha e reenvia com a versão nova";
  - "o regulador não chega aqui": `REGULATION_NO_ROLE` e nenhuma leitura.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/RegulationApacSettings.tsx src/modules/regulation/apac/NumberRangesPanel.tsx src/modules/regulation/apac/MomentPanel.tsx src/modules/RegulationApacSettings.test.tsx
$G commit -m "feat: register the APAC and AIH number ranges and choose the authorization moment" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Programação e teto (FPO): linhas, edição e importação conferida

**Files:**
- Create: `src/modules/regulation/apac/FpoLinesPanel.tsx`, `src/modules/regulation/apac/FpoLineForm.tsx`, `src/modules/regulation/apac/FpoImportPanel.tsx`, `src/modules/RegulationFpo.test.tsx`
- Modify: `src/modules/RegulationFpo.tsx`

**Interfaces:**
- Consumes: `listFpoLines`, `createFpoLine`, `updateFpoLine`, `uploadFpoImport`, `getFpoImport`, `applyFpoImport`, `discardFpoImport`, `fetchFpoImportTemplate`, `listCareProviders`, `getApacOptions` (Task 1); `fpoLineProblems`, `balanceText`, `FPO_IMPORT_HEADER`, `fpoFileProblem`, `fpoHeaderProblem`, `readFirstLine`, `fpoRowErrorText`, `fpoStatusLabel`, as chaves (Task 2); `parseReais`, `fmtCents`, `competenceOptions`; `saveFile`; `SensitiveAction`; `ConfirmButton`. Molde: a Task 11 do 27b.
- Produces: `RegulationFpo({ today, onGoToSecurity })`, `ApacGate title="Programação e teto (FPO)" sub="teto em dinheiro por prestador e competência" allow={(r) => r.admin || r.regulator}` (proposta; o regulador lê, Q13):
  - **Competência** (`select` por `competenceOptions(today)`, mais as 2 seguintes) e "Prestador" (todos ou um, por `listCareProviders`).
  - `FpoLinesPanel`: lista paginada (`useInfiniteQuery`, "Carregar mais" pelo `next_before`), `DataTable` com "Prestador", "Nível e código" (`levelLabel`, código e nome), "Financiamento" (`financingLabel` e o programa), "Programado" (quantidade e `fmtCents`), "Complemento por unidade" (`fmtCents(complement_cents_per_unit)`, Q9), "Reservado" (`fmtCents`) e "Saldo" (`fmtCents(balance_cents)`, `Tag` `down` quando ≤ 0, proposta). O admin tem "Editar" por linha e "Nova linha"; o regulador só lê.
  - `FpoLineForm` (criar: todos os campos; editar: quantidade e valor, como o PATCH do plano do api): `fpoLineProblems` antes (FAEC só por procedimento, Q9); `SensitiveAction` (`requiresStepUp`) → `createFpoLine` ou `updateFpoLine(id, { quantity, amount_cents, expected_version })`. `409 fpo_line_changed` → a frase, uma releitura e o digitado mantido; `409 duplicate_fpo_line` → a frase; `invalid_request` com `path` → a frase no campo.
  - `FpoImportPanel` (só admin), no molde do "Saldo inicial":
    - a nota (proposta): "A planilha cria as linhas novas e atualiza a quantidade e o valor das que já existem (mesma competência, prestador, nível, código e financiamento). A competência de cada linha vem na planilha.";
    - "Baixar o modelo": `fetchFpoImportTemplate` e `saveFile(file, file.filename ?? "modelo-fpo.csv")`;
    - `<input type="file" accept=".csv,text/csv" aria-label="Planilha da programação (FPO)">`; ao escolher, `fpoFileProblem` e `fpoHeaderProblem(await readFirstLine(file))`; recusado → a frase e **nenhuma** chamada;
    - "enviar e conferir" (`ConfirmButton`) → `uploadFpoImport(file)`; o arquivo sai do estado; `invalid_file` e `file_too_large` → as frases;
    - o relatório (`getFpoImport(id)`): linhas válidas e com erro, erros por código (`fpoRowErrorText` e a contagem), "Totais por prestador" (o nome pelo `care_provider_id`, linhas e `fmtCents`) e as linhas em `DataTable`, com o filtro "só com erro";
    - `validated` → "Aplicar <válidas> linhas" (`SensitiveAction`, `requiresStepUp`) → `applyFpoImport(id, version)`, e "Descartar" (`ConfirmButton`) → `discardFpoImport`; `applied` e `discarded` só leitura;
    - `409 fpo_import_changed` → a frase e uma releitura; depois de cada ato, relê o relatório e as linhas. Sem lista de importações no api, a tela mostra só a que acabou de enviar.

- [ ] **Step 1: Testes que falham** (`RegulationFpo.test.tsx`):
  - "linhas: programado, complemento, reservado e saldo; o regulador só lê": duas linhas; "R$ 10.000,00", "R$ 771,34"; o regulador não vê "Editar" nem a importação;
  - "nova linha: valor digitado com milhar, FAEC com programa e só por procedimento": `"1.234,56"` → `amount_cents: 123456`; FAEC sem programa ou por grupo → a frase e nenhuma chamada;
  - "editar: 409 relê uma vez e mantém o digitado; linha repetida na criação mostra a frase";
  - os quatro casos do Review Focus 3: "arquivo recusado antes: nenhuma chamada" (`.xlsx`, 6 MB, cabeçalho com vírgula); "envio com BOM: relatório aberto e nome fora do corpo" (`uploadFpoImport(file)`, "Totais por prestador" com `R$`); "aplicar: step-up, um pedido, versão na chamada" (duplo clique → `applyFpoImport("fi1", 1)` uma vez); "linha FAEC por grupo no relatório" (`faec_not_procedure` com a frase);
  - "descartar: confirmação e só leitura depois".
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/RegulationFpo.tsx src/modules/regulation/apac/FpoLinesPanel.tsx src/modules/regulation/apac/FpoLineForm.tsx src/modules/regulation/apac/FpoImportPanel.tsx src/modules/RegulationFpo.test.tsx
$G commit -m "feat: show and edit the FPO ceiling per provider and import the spreadsheet with review" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: "OCI em andamento"

**Files:**
- Create: `src/modules/regulation/apac/OciProcedures.tsx`, `src/modules/RegulationOci.test.tsx`
- Modify: `src/modules/RegulationOci.tsx`

**Interfaces:**
- Consumes: `listOci`, `getRegulationRequest`, `markProcedurePerformed`, `closeOci`, `getApacOptions` (Task 1); `performedLabel`, `ociAlarms`, `ociProgress`, `outcomeLabel`, `optionLabel`, `fmtSigtap`, `regulationApacError`, `isStale`, `OCI_LIST_KEY` (Task 2); `competenceLabel`; `ConfirmButton`.
- Produces: `RegulationOci({ today })`, `ApacGate title="OCI em andamento" sub="secundários, encerramento e prazo da oncologia" allow={(r) => r.regulator}`:
  - a lista (`useInfiniteQuery` de `listOci`, `NOMINAL_QUERY`, "Carregar mais"), sem laudo nem CID (forma do plano do api): "Pedido" (`request_ref`), "OCI" (`target_name`), "Prestador", "Número" (`authorization_number`), "Validade até" (`competenceLabel`), "Andamento" (`ociProgress`) e as `Tag` `down` de `ociAlarms` ("oncologia: <n> dias", "vencida", proposta).
  - Acima da lista, a faixa `role="alert"` (proposta) "N OCI com alarme: oncologia perto dos 30 dias ou validade vencida." só com alarme nas linhas lidas (Q7).
  - "Abrir" lê o pedido (`getRegulationRequest(id)`, leitura nominal com trilha, que traz a `version` e os procedimentos) e mostra `OciProcedures({ request, options, today, readOnly?, onChanged, onStale })`:
    - cada secundário com `performedLabel`; os sem data têm "Marcar realizado" → campo "Data do atendimento" (`input type="date"`, até hoje) → `markProcedurePerformed(request.id, p.id, { performed_at, version })`;
    - o que veio do prontuário (`performed_procedure`, ligado sozinho pelo api, Q10) não tem botão;
    - "Encerrar a OCI": "Resultado" (`radio`: "concluída", "não concluída", Q7); na concluída, "Motivo de saída" (`select` com `options.oci_exit_reasons`, D17); `ConfirmButton` com a pergunta (proposta) "Encerrar a OCI como <resultado>?" e, na não concluída, "Os secundários feitos vão para o BPA-I, e a reserva volta ao teto." → `closeOci(request.id, { outcome, exit_reason?, version })`;
    - `invalid_request` em `outcome`, `exit_reason` ou `performed_at` → a frase no campo; `stale_version` → "Recarregar"; `invalid_transition` → a frase do 26a e a lista relida uma vez; sucesso → "OCI encerrada." (proposta) e a lista relida uma vez.

- [ ] **Step 1: Testes que falham** (`RegulationOci.test.tsx`):
  - "lista: andamento, alarme da oncologia e da vencida, sem laudo": `ociItem({ oncology_alert: true, days_open: 27 })` e `ociItem({ id: "rr2", expired: true })` → a faixa e as duas `Tag`; "1 de 2 secundários"; nenhum resumo do laudo nem CID no `body`;
  - "marcar realizado: data e versão do pedido lido, um pedido": abrir → `getRegulationRequest("rr1")` uma vez; duplo clique → `markProcedurePerformed("rr1", "rp2", { performed_at: "2026-12-01", version: 4 })` uma vez; data futura recusada antes; `422 invalid_request` em `performed_at` → a frase;
  - "o realizado pelo prontuário não tem botão";
  - "encerrar concluída exige o motivo; não concluída vai sem motivo": `closeOci("rr1", { outcome: "concluded", exit_reason: "1.1", version: 4 })` e `{ outcome: "not_concluded", version: 4 }`; `422 invalid_request` em `outcome` → a frase;
  - "estado mudado: invalid_transition relê a lista uma vez";
  - "o admin não chega aqui".
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/RegulationOci.tsx src/modules/regulation/apac/OciProcedures.tsx src/modules/RegulationOci.test.tsx
$G commit -m "feat: follow open OCI with the secondary procedures, the closing outcome and the deadline alarm" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: "Revisão de APAC"

**Files:**
- Create: `src/modules/RegulationApacReview.test.tsx`
- Modify: `src/modules/RegulationApacReview.tsx`, `src/modules/regulation/apac/OciProcedures.tsx` (só leitura)

**Interfaces:**
- Consumes: `listAwaitingReview`, `getRegulationRequest`, `reviewAuthorization`, `getApacSettings` (Task 1); `OciProcedures` (Task 8, com `readOnly`); `kindLabel`, `regulationApacError`, `isStale`, `REVIEW_LIST_KEY` (Task 2); `ReasonAction` do 26a; `reasonProblem`.
- Produces: `RegulationApacReview({ today })`, `ApacGate title="Revisão de APAC" sub="autorizar a APAC depois do atendimento comprovado" allow={(r) => r.regulator}`:
  - no modo `before` (lido de `getApacSettings`, Q13), `EmptyState` "a cidade autoriza antes do atendimento — não há revisão" (proposta), sem ler a lista;
  - a lista (`listAwaitingReview`, sem laudo, forma do plano do api): "Pedido" (`request_ref`), "Tipo" (`kindLabel`), "Procedimento" (`target_name`), "Prestador" e "Realizados" ("<done> de <total>");
  - "Abrir" lê o pedido (`getRegulationRequest(regulation_request_id)`), mostra os procedimentos (`OciProcedures` com `readOnly`) e duas ações, com a `version` do pedido e o `authorization.id`:
    - "Autorizar a APAC" (`ConfirmButton`, pergunta proposta "Emitir o número da APAC agora?") → `reviewAuthorization(auth.id, { decision: "issue", version })`; sucesso → "Número <número> emitido." (proposta);
    - "Negar a APAC" (`ReasonAction`, "Motivo da negativa", 10–500, Q10) → `{ decision: "deny", reason, version }`; sucesso → "APAC negada: a reserva voltou ao teto, e o que foi feito vai para o BPA-I." (proposta);
  - `invalid_transition` → a frase do 26a e a lista relida uma vez; `number_range_exhausted` e `authorizer_not_allowed` → a frase; `stale_version` → "Recarregar".

- [ ] **Step 1: Testes que falham** (`RegulationApacReview.test.tsx`):
  - "modo antes: nada a revisar e nenhuma leitura da lista";
  - "lista sem laudo: tipo, procedimento e realizados";
  - "autorizar a APAC: um pedido e o número na frase";
  - "negar: motivo obrigatório e frase da reserva";
  - "estado mudado: invalid_transition relê uma vez"; "faixa esgotada na revisão: a frase".
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/RegulationApacReview.tsx src/modules/RegulationApacReview.test.tsx src/modules/regulation/apac/OciProcedures.tsx
$G commit -m "feat: review APAC authorizations after the proven service in the authorize-after mode" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: "Remessa APAC" e "Fila nominal da OCI"

**Files:**
- Create: `src/modules/RegulationApacExports.test.tsx`, `src/modules/RegulationOciQueue.test.tsx`
- Modify: `src/modules/RegulationApacExports.tsx`, `src/modules/RegulationOciQueue.tsx`

**Interfaces:**
- Consumes: `listApacExports`, `createApacExport`, `fetchApacExportFile`, `listOciQueueExports`, `createOciQueueExport`, `fetchOciQueueExportFile`, `listAllUnits` (Task 1 e o que já existe); `regulationApacError`, `APAC_EXPORTS_KEY`, `OCI_QUEUE_EXPORTS_KEY` (Task 2); `competenceOptions`, `competenceLabel`, `fmtDateTime`; `saveFile`; `SensitiveAction`. Molde: `src/modules/psychosocial/RaasExports.tsx`.
- Produces:
  - `RegulationApacExports({ today, onGoToSecurity })`, `ApacGate title="Remessa APAC" sub="arquivo da APAC da rede própria por competência" allow={(r) => r.admin || r.clerk}` (Q10):
    - "Unidade" (`listAllUnits`) e "Competência" (`competenceOptions(today)`);
    - a lista: "Versão", "Gerada em", "Por", "APAC" (`apacs`), "Conferência (SHA-256)" (16 primeiros caracteres, `title` com o resto) e "Baixar versão <n>" → `fetchApacExportFile` + `saveFile(file, file.filename ?? "apac-<competência>-v<n>.txt")`;
    - "Gerar remessa" (`SensitiveAction`, `requiresStepUp`; descrição proposta "Gera o arquivo das APAC e OCI emitidas para esta unidade que tiveram procedimento realizado nesta competência. Confira no VERSIA antes de usar.") → `createApacExport(unitId, competence)`, e a lista relida;
    - `nothing_to_export`, `settings_missing` e `not_own_provider` (Q10) → a frase; duplo clique → um pedido.
  - `RegulationOciQueue({ today, onGoToSecurity })`, `ApacGate title="Fila nominal da OCI" sub="lista mensal da OCI para o Ministério" allow={(r) => r.admin}`:
    - a nota (proposta) "Os pedidos de OCI já vão ao MIRA pelo envio à RNDS. Este arquivo, no formato do e-SUS Captação de Filas, é o plano B até o Ministério confirmar o canal. Ele sai sozinho no 1º dia útil; envie até o 5º dia útil.";
    - a lista: "Competência", "Versão", "Gerada em", "Pedidos" (`rows`), "Conferência (SHA-256)" e "Baixar" → `fetchOciQueueExportFile` + `saveFile(file, file.filename ?? "fila-oci-<competência>.csv")`;
    - "Gerar agora" (competência; `SensitiveAction`, `requiresStepUp`) → `createOciQueueExport(competence)`.

- [ ] **Step 1: Testes que falham.**
  - `RegulationApacExports.test.tsx`: "lista e baixar: nome do api ou por competência"; "gerar: step-up, um pedido e releitura"; "faltam os dados do faturamento: a frase de settings_missing"; "sem APAC com procedimento na competência: a frase"; "unidade que não é própria: a frase"; "o faturista gera; o regulador não chega aqui".
  - `RegulationOciQueue.test.tsx`: "lista e baixar"; "gerar agora: step-up e um pedido"; "só o admin".
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/RegulationApacExports.tsx src/modules/RegulationOciQueue.tsx src/modules/RegulationApacExports.test.tsx src/modules/RegulationOciQueue.test.tsx
$G commit -m "feat: generate and download the own-network APAC file and the OCI nominal queue" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: BPA (22) e "Pendentes de assinatura"

**Files:**
- Create: `src/modules/billing/ProcedureTable.apac.test.tsx`
- Modify: `src/lib/billing.ts`, `src/lib/billing.test.ts`, `src/modules/billing/ProcedureTable.tsx`, `src/lib/signature.ts`, `src/lib/signature.test.ts`

**Interfaces:**
- Consumes: `PerformedProcedure.authorization_number` e `SignatureDocumentType` `regulation_laudo` (Task 1).
- Produces:
  - `REASON_LABEL.oci_secondary_active = "secundário de OCI ainda em andamento — entra no BPA-I só se a OCI for encerrada sem conclusão"` (proposta; o código é do §7);
  - `ProcedureTable`: com `authorization_number`, a linha mostra "autorização <número>" (`mono`, proposta) abaixo do procedimento; sem ele, nada muda;
  - `DOCUMENT_LABEL.regulation_laudo = "laudo de regulação"` (proposta). O "Voltar ao papel" vale para ele como para a consulta (sem cert, o api já abre `manual`).

- [ ] **Step 1: Testes que falham.**
  - `billing.test.ts`: `reasonLabel("oci_secondary_active")` tem a frase; `REASON_CODES` o inclui.
  - `ProcedureTable.apac.test.tsx`: a linha com `authorization_number: "4126200000015"` mostra "autorização 4126200000015"; a linha sem ele fica igual.
  - `signature.test.ts`: `documentLabel("regulation_laudo")` = "laudo de regulação".
- [ ] **Step 2–4:** rode e veja falhar (os três arquivos e `src/modules/billing/ProceduresPanel.test.tsx src/modules/SignaturePending.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/billing.ts src/lib/billing.test.ts src/modules/billing/ProcedureTable.tsx src/modules/billing/ProcedureTable.apac.test.tsx src/lib/signature.ts src/lib/signature.test.ts
$G commit -m "feat: show the OCI secondary block and the authorization number in billing and the laudo in pending signatures" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Suíte, build, varreduras, revisão e prova no navegador (api na 3064, Vite na 5225)

**Files:** `$R/.claude/launch.json` (entrada nova; fora de git). Correção que a prova pedir entra na task dona, com teste.

- [ ] **Step 1: Suíte, tipos e build**

```bash
$R/bin/test dashboard -w dashboard-mod26b
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde e `build-ok`. A contagem é a base da Task 0 + **15 arquivos de teste novos**:
- `src/lib/api.regulationApac.test.ts`, `src/lib/regulationApacErrors.test.ts`, `src/lib/regulationApac.test.ts`;
- `src/shell/modules.regulationApac.test.ts`;
- `src/modules/regulation/RequestForm.laudo.test.tsx`, `src/modules/consultation/RegulationRequestsField.laudo.test.tsx`;
- `src/modules/RegulationQueue.apac.test.tsx`, `src/modules/regulation/AuthorizeForm.apac.test.tsx`;
- `src/modules/RegulationApacSettings.test.tsx`, `src/modules/RegulationFpo.test.tsx`, `src/modules/RegulationOci.test.tsx`, `src/modules/RegulationApacReview.test.tsx`;
- `src/modules/RegulationApacExports.test.tsx`, `src/modules/RegulationOciQueue.test.tsx`, `src/modules/billing/ProcedureTable.apac.test.tsx`;
- e os casos novos em `features.test.ts`, `billing.test.ts`, `signature.test.ts` e `RegulationReturned.test.tsx` (sem arquivo novo).

Contagem que dobra = artefato de build achado pelo vitest. Erro não tratado com tudo verde = mock sem `fetchCurrentSession`.

- [ ] **Step 2: Varreduras**

```bash
cd $W
O="src/lib/regulationApac.ts src/lib/regulationApacErrors.ts src/modules/regulation/apac src/modules/RegulationApacSettings.tsx src/modules/RegulationFpo.tsx src/modules/RegulationOci.tsx src/modules/RegulationApacReview.tsx src/modules/RegulationApacExports.tsx src/modules/RegulationOciQueue.tsx"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O src/modules/regulation/RequestForm.tsx && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
grep -rnE "queryKey:.*(summary|clinical_summary|cid_principal|findings|laudo\.)" $O src/modules/regulation && echo "ERRO: laudo no cache" || echo ok-cache
grep -rnE "clinical_summary|cid_principal|findings" src/modules/regulation/RequestTable.tsx src/modules/RegulationOci.tsx src/modules/RegulationApacReview.tsx src/modules/RegulationApacExports.tsx src/modules/RegulationOciQueue.tsx && echo "ERRO: laudo em lista" || echo ok-lista
sed -n '/Regulação por laudo (módulo 26b/,/^\/\/ ─── Farmácia/p' src/lib/api.ts | grep -nE "summary|cid_|findings" | grep -E "withQuery|URLSearchParams|\?\$\{" && echo "revisar: URL" || echo ok-url
for f in src/modules/regulation/RequestForm.laudo.test.tsx src/modules/RegulationQueue.apac.test.tsx src/modules/regulation/AuthorizeForm.apac.test.tsx src/modules/RegulationApacSettings.test.tsx src/modules/RegulationFpo.test.tsx src/modules/RegulationOci.test.tsx src/modules/RegulationApacReview.test.tsx src/modules/RegulationApacExports.test.tsx src/modules/RegulationOciQueue.test.tsx; do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-cache`, `ok-lista`, `ok-url`, `ok-mocks`, sem `ERRO`.

- [ ] **Step 3: Conferência do api.** Repita a Task 0 Step 4 contra o api final. Forma diferente vira ajuste na task dona antes da prova.

- [ ] **Step 4: Revisão do ramo** (superpowers:requesting-code-review). Foco: as cinco linhas do Review Focus; o 26a intocado com o interruptor desligado; a tabela `REGULATION_APAC_ERRORS`; o laudo fora de lista, URL e cache; o step-up das escritas; as frases fixas letra por letra.

- [ ] **Step 5: Entrada do Vite.** Acrescente ao `$R/.claude/launch.json`, depois da última entrada `dashboard-mod*`:

```json
{
  "name": "dashboard-mod26b",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/dashboard-mod26b && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3064 npx vite --port 5225 --host 0.0.0.0" ],
  "port": 5225
}
```

- [ ] **Step 6: Prova no navegador.** Antes de começar:
  - **Autorização do usuário:** migrar os bancos de dev (cidade e plataforma do 26b), carregar a SIGTAP completa da Task 0 do api e rodar a semente do 26b em Curitiba (plano do api) só com autorização dele. Peça antes de qualquer `db:migrate`, `db:seed` ou carga.
  - **Dono de cada serviço:** o api do worktree `apps/api/.claude/mod26b` na **3064** é da sessão do api; o maintenance na 5177 é da sessão dele.
  - **Abrir:** `curitiba.localhost:5225/dashboard/` (`preview_start` com `dashboard-mod26b`).
  - **Senhas:** a senha e o TOTP de dev podem ser mostrados; os de staging, nunca.

  Roteiro, um passo por linha:

1. **Interruptor** (maintenance 5177): `care_regulation_apac` ligado. Antes dele, nada do 26b no menu, e o pedido de um procedimento só de APAC é recusado com a frase de `invalid_target`.
2. **Configurações** (`municipal_admin`): o momento "antes"; uma faixa de APAC `2`, uma de OCI `7` e uma de AIH `5`, com step-up; a faixa sobreposta recusada; a FPO da competência pela planilha (conferência, um erro de CNES, aplicar com step-up) e uma linha editada à mão.
3. **Consulta** (médico): um pedido APAC (catarata), uma OCI com 2 secundários (um incompatível recusado na linha) e uma AIH eletiva; "Laudo em PDF"; finalizar; o laudo em "Pendentes de assinatura" (ou "assine à mão" sem certificado).
4. **Fila** (`care_regulator`): a `Tag` do tipo; o detalhe com o laudo; autorizar a APAC (prévia com linha, saldo e reserva; número emitido); autorizar a AIH com teto esgotado → "Autorizar acima do teto" com step-up; um autorizador sem nível superior recusado (usuário da semente).
5. **OCI em andamento**: marcar um secundário externo, ver o do prontuário, encerrar com o motivo; a OCI de oncologia da semente com o alarme.
6. **Modo depois**: o admin troca o momento; uma liberação sem número; "Revisão de APAC" → "Autorizar a APAC" e uma negativa.
7. **Arquivos**: "Remessa APAC" da unidade própria (ou o status do layout não homologado) e "Fila nominal da OCI" gerada e baixada; no faturamento, o secundário de OCI ativa com `oci_secondary_active` e o BPA com o número da autorização.
8. **Sigilo**: na aba de rede, o laudo só no corpo dos `POST`/`PATCH`; as URLs só com ids, competência, tipo e cursor; a lista da fila sem o laudo.

  Também:
  - preset mobile 375×812 nos passos 3, 4 e 7;
  - só teclado nos passos 2 e 3;
  - `resize_window` com o preset desktop ao terminar.

  Anote cada passo. Achado que muda comportamento vira teste na task dona antes da correção. Achado só visual vai ao usuário (memória "Interface em ciclo próprio").

---

### Task 13: Entrega — board, relato e parada antes do merge

**Files:** nenhum.

- [ ] **Step 1: Board.** Os cards do 26b que o dashboard toca vão a **Done** só se o api já terminou o deles, e a **Verified** só depois da prova da Task 12, com o usuário.
- [ ] **Step 2: Suíte completa do api.** Não é desta sessão: a sessão dona do api faz o pedido à sessão "Testes" (`api · mod26b/feat/mod-26b-apac · --full · mudança transversal do 26b`). A suíte inteira do dashboard neste worktree (Task 12 Step 1) basta para o dashboard.
- [ ] **Step 3: Pare e peça autorização** para merge e push (depois do api e da suíte completa).
  - Antes do push, confira `$G -C $W log --oneline origin/main..HEAD` e `origin/main..main` (sessões paralelas).
  - O worktree só sai depois do merge: `$G -C $R/apps/dashboard worktree remove $W`.

**Relato:** `dashboard · dashboard-mod26b/feat/mod-26b-apac · Task 13 · suíte <n>/<n>, build ok, varreduras ok, prova <passos> · cards movidos · aguardando autorização de merge`.

---

## Self-review

1. **Cobertura do contrato §11 e da spec §3.6:**
   - consulta, pedido por laudo com tipo, laudo e procedimentos (OCI com secundários): Task 4;
   - fila com o tipo; autorização com linha da FPO, saldo, reserva, número e "Autorizar acima do teto": Task 5;
   - faixas e momento: Task 6; FPO com importação e edição (Q2): Task 7;
   - "OCI em andamento" com secundários, encerramento e alarme: Task 8; "Revisão de APAC" no modo depois (§5): Task 9;
   - "Remessa APAC" e "Fila nominal da OCI" (§7, §8, Q3): Task 10;
   - BPA com número e `oci_secondary_active` (§7); laudo em PDF e assinatura (§10): Tasks 4 e 11;
   - `REGULATION_APAC_ERRORS` em `src/lib/regulationApacErrors.ts`: Task 2; o rótulo do interruptor (§1): Task 1.
   - Fora do dashboard, por desenho: SIGTAP da plataforma (§2), eventos (§11), MIRA (§9) e o arquivo de AIH (spec "fora desta entrega").
2. **Bordas que tocam a tela:** secundários e quantidade (Tasks 2 e 4); teto, faixa esgotada, autorizador e corrida (Task 5); faixa colada inteira e sobreposição (Tasks 2 e 6); planilha e dinheiro (Tasks 2 e 7); estado mudado na OCI e na revisão (Tasks 8 e 9); código novo do api (Tasks 2 e 5); interruptor desligado (Tasks 3 e 4).
3. **Tipos:** `AuthorizationPreview` (Task 1) é o da Task 5; `ProcedureLine`, `proceduresBody` e `procedureIndexOf` (Task 2) são os da Task 4; `RangeDraft` e `FIFTH_DIGITS` (Task 2) são os da Task 6; `OciProcedures` (Task 8) é reusado com `readOnly` na Task 9; `listOci`/`OciItem` e `listAwaitingReview`/`ReviewItem` (Task 1) servem às Tasks 8 e 9; `searchRegulationTargets(q, type, kind)` (Task 1) é o do `TargetSearch` (Task 4).
4. **Review Focus:** as cinco linhas têm teste na task dona (Tasks 5, 4, 7, 6 e 2/5).
5. **Proporção:** o código do plano é tipo, frase e teste; as telas vão por assinatura, props e comportamento.

## Divergências propostas ao contrato

D1–D16 foram decididas no §12 do contrato (Q6–Q10 e Q13; docs `60908e1`), e o plano já aplica as decisões. O texto de cada uma fica como registro. A D17 é nova e segue aberta.

- **D1 — Resolvida pela Q13.** A convenção `expected_version` → `409 <entidade>_changed` não batia com o 26a, que usa `version` e `409 stale_version` nas ações do pedido (`apps/dashboard/src/lib/api.ts:4634`; `src/lib/regulation.ts:415`; `apps/api/app/services/care_regulation/json.rb:24`). **Decisão:** autorizar, revisar, marcar realizado e encerrar a OCI seguem o 26a; as entidades novas usam `expected_version`, com `apac_settings_changed`, `fpo_line_changed` e `fpo_import_changed` (Tasks 1, 2, 6, 7).
- **D2 — Resolvida pela Q10.** Faltavam rotas, formas e códigos. **Decisão:** rotas, corpos e códigos de recusa são os do plano do api (Valor 11 e Interfaces das Tasks 5–14). O plano trocou as propostas por eles: `kind` na busca; `care_provider_id` e paginação nas linhas da FPO; a planilha só com `file_base64` e o cabeçalho `competencia;cnes;nivel;codigo;financiamento;programa_faec;quantidade;valor`; os códigos de linha `provider_not_found`, `invalid_code`, `faec_not_procedure` e os demais; `number_range_overlap` e `duplicate_fpo_line` (409); `invalid_request` com `path` no lugar dos códigos de faixa, de motivo de saída e de estado mudado; `GET /regulation/oci` e `GET /regulation/authorizations?stage=awaiting_review` com as formas próprias (`OciItem`, `ReviewItem`); `oci_close` com `outcome`; `apacs`/`rows` nas remessas; `settings_missing`. A prévia vem da Q13 (D9).
- **D3 — Resolvida pela Q6.** A busca não dizia o tipo do alvo (`apps/api/app/services/care_regulation/targets.rb:81,110`). **Decisão:** `Targets.resolve` e `search` ganham `kind`, e a busca devolve o `authorization_kind`. Vale só para procedimento sem instrumento BPA dos grupos 02–04 e para a OCI do grupo 09; `aih_instruments` é só `03`. Na tela, o solicitante escolhe o instrumento antes de buscar (Task 4).
- **D4 — Resolvida pela Q6.** O laudo tinha `cid_principal` e o pedido já tinha `cid10` (`src/lib/regulation.ts:218-281`). **Decisão:** `cid_principal` é o `cid10` do pedido (o laudo não o leva); a `justification` do 26a continua; o complemento não troca o CID de laudo já assinado (Tasks 1, 2, 4).
- **D5 — Resolvida pela Q7.** O array `procedures` e os códigos da OCI não estavam fixados. **Decisão:** um principal igual ao `target_code`, primeiro, com `procedures[i]` indexando o array (plano do api, Task 5); ao menos um secundário é consulta ou teleconsulta; as recusas são `oci_needs_two_secondaries`, `oci_needs_consultation` e `oci_principal_not_group_09` (Tasks 2 e 4).
- **D6 — Resolvida pela Q13.** "Nunca em lista" × a lista da consulta (`json.rb:9-10`; `RegulationRequestsField.tsx:151`). **Decisão:** a lista de pedidos da própria consulta mostra o laudo (é o rascunho do autor); a fila nunca mostra (Tasks 4, 5).
- **D7 — Resolvida pela Q13.** O complemento não levava o laudo (`src/lib/api.ts:4635`). **Decisão:** o complemento aceita laudo e procedimentos (Task 4).
- **D8 — Resolvida pela Q13.** A autorização só lia ofertas do mês corrente (`AuthorizeForm.tsx:28-33`). **Decisão:** os pedidos por laudo aceitam a oferta do mês corrente e dos 2 seguintes; o mês da oferta continua `YYYY-MM` (Task 5).
- **D9 — Resolvida pela Q13.** Não havia leitura antes do POST. **Decisão:** há uma rota de prévia (linha da FPO, saldo, reserva), e o POST aceita o financiamento escolhido. O plano do api ainda não lista a rota: o nome `GET /regulation/requests/:id/authorization_preview` e a forma `AuthorizationPreview` são conferidos na Task 0 (Tasks 1, 5).
- **D10 — Resolvida pela Q8.** O tipo não separava o 5º dígito. **Decisão:** o api escolhe (APAC: OCI → 7, créditos → 9, atributo 052 → 6, senão 2; AIH: créditos → 8, atributo 051 → 5, senão 1). A tela não manda o dígito (Task 5).
- **D11 — Resolvida pela Q8.** O que era `first_number`/`last_number`. **Decisão:** a faixa guarda o sequencial de 7 dígitos, e o api devolve também o número inteiro (`first_full`/`last_full`, nomes conferidos na Task 0); a UF sai do IBGE da cidade (Tasks 2, 6).
- **D12 — Resolvida pela Q13.** O `GET` do momento era só do admin. **Decisão:** o regulador lê o momento e a FPO; a escrita continua do admin, com step-up (Tasks 3, 7, 9).
- **D13 — Resolvida pela Q10.** O tipo assinável e a rota do PDF não estavam fixados (`src/lib/api.ts:1891`, `:4677`). **Decisão:** `regulation_laudo`, rota `laudo.pdf`, assinatura aberta na promoção com autor = solicitante (Tasks 1, 4, 11). Pela Q12, o paciente sem CPF tem o laudo impresso para assinar à mão.
- **D14 — Resolvida pela Q13.** Formato da competência (`src/lib/competence.ts:5` × `src/lib/regulation.ts:349`). **Decisão:** `AAAAMM`; o mês da oferta continua `YYYY-MM`.
- **D15 — Resolvida pela Q10.** Quem gera a remessa. **Decisão:** o `municipal_admin` ou o `billing_clerk`; unidade que não é própria → `422 not_own_provider`; a remessa leva as APAC e OCI emitidas para unidade própria com procedimento realizado na competência (Task 10).
- **D16 — Resolvida pela Q10.** Quem liga o realizado e se a negativa leva motivo. **Decisão:** o api liga sozinho (paciente, SIGTAP e unidade executante dentro da validade), e a negativa da revisão leva motivo (Tasks 8, 9).
- **D17 — Aberta. As listas dos `select`.** A tela precisa dos motivos de saída da OCI (`config/regulation/oci_exit_reasons.yml`), das clínicas do SIH (`sih_clinics.yml`), dos programas FAEC (`faec_programs.yml`, Q9) e das fontes orçamentárias dos créditos (`BUDGET_SOURCES`, Q9), mas o plano do api não expõe nenhuma delas (Valor 4 e Valor 11). Copiar as listas no dashboard faria as duas pontas divergirem quando a Task 0 do api conferir os nomes oficiais. **Proposta:** `GET /regulation/apac_options` → `{ oci_exit_reasons, aih_clinics, faec_programs, budget_sources }`, cada item `{ code, name }`, lido dos mesmos `config/regulation/*.yml` (Tasks 1, 4, 5, 8). Até a decisão, a Task 0 confere se o api já expõe as listas; sem elas, o campo aceita o código digitado, e o api confere.
