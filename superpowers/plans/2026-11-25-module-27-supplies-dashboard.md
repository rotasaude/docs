# Módulo 27 — Insumos (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-25-module-27-supplies-contracts.md` (fonte única), com as decisões Q1–Q13 do §11 (docs `7348ae0`). As divergências D1–D10 deste plano foram resolvidas pelas Q7–Q12 (seção final). As formas das rotas e os códigos de recusa são os do **plano do api** (`2026-11-25-module-27-supplies-api.md`, Valores 4–6 e Tasks 4–17), pela Q10; a Task 0 confere tudo contra o api real.
> 2. A `main` do dashboard **com o 26c mergeado** (o contrato manda executar depois dele). Quando este plano foi escrito, o `origin/main` estava em `2e3a2a9`, só com o 35a: nem o 27b, nem o 35b, nem o 26c.
> 3. O **api do 27 (insumos)** na branch `feat/mod-27-supplies` (`apps/api/.claude/mod27s`, porta **3065**). As Tasks 1–13 seguem pelas formas deste plano e pelas fixtures. Só a Task 14 (prova) depende do api rodando.
>
> Ordem: api → **dashboard** e maintenance → suíte completa do api (sessão "Testes") → prova.

**Goal:** Lado dashboard dos insumos do módulo 27:
- a seção "Programas de insumos" na consulta (inscrever, renovar, encerrar), com a cota calculada pelo api à vista;
- no grupo **Farmácia**, quatro telas: "Retirada de insumos", "Inscrição com laudo externo", "Aparelhos em comodato" e "Importar pacientes dos programas";
- em "Locais e configurações", o painel "Regras de cota" e os modelos de aparelho (Q12);
- o insumo nas telas de estoque (entradas, abertura, posição, mínimos; transferência e inventário pela posição);
- os campos de ponto de estomia na tela de unidades;
- as frases em `SUPPLIES_ERRORS` (`src/lib/suppliesErrors.ts`).

**Architecture:** O cliente ganha a seção "Insumos: catálogo de materiais, programas, cota, retirada e comodato (módulo 27; contrato 2026-11-25 §1–§9)" no fim de `src/lib/api.ts`, e estende os tipos do 19 (consulta), do 20 (linha do tempo), do 27a (estoque e dispensação) e das unidades.

Onde fica cada coisa:
- portões, chaves de cache, rótulos e regras da farmácia: `src/lib/supplies.ts`;
- regras do bloco da consulta e das regras de cota: `src/lib/supplyPrograms.ts`;
- regras da planilha: `src/lib/supplyImport.ts`;
- frases de recusa: `src/lib/suppliesErrors.ts` (`SUPPLIES_ERRORS`, contrato §8);
- telas: raízes em `src/modules/Supply*.tsx` e `src/modules/DeviceLoans.tsx`; peças em `src/modules/supplies/`, `src/modules/consultation/` e `src/modules/pharmacy/`.

O dashboard **não calcula** a cota, o saldo, o período, o teto do SIGTAP nem a inativação: mostra o que o api devolve e só não envia o que já sabe que será recusado (cota à vista: o api devolve a cota calculada no PATCH do rascunho, Q7).

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-25-module-27-supplies-design.md` (decisões 1–18; §3.1–§3.5). **Contrato:** `docs/superpowers/plans/2026-11-25-module-27-supplies-contracts.md` §1–§9 e Q1–Q13. **Plano do api:** `2026-11-25-module-27-supplies-api.md` (Valores 2, 4, 5 e 6; Tasks 4, 5, 8–17). **Moldes:** `2026-11-24-module-27b-bnafar-dashboard.md` (Task 11, "Saldo inicial", para a importação); `DentalEditor`/`TreatmentPlanPanel` (bloco na consulta); `PharmacyDispensing`, `PharmacyExternal`, `DispenseForm`, `PatientHistory`, `ReturnForm` (farmácia).

## Global Constraints

- **Nomes no código** (contrato §1–§5, Q1):
  - interruptores `pharmacy_supplies` e `supply_programs`; o faturamento da bolsa exige também `procedure_billing` (`src/lib/billing.ts:37-39`, a chave do 22);
  - bloco da consulta `supply_programs` (nunca `supplies`, que é da odontologia: `DentalFields.supplies`, `src/lib/api.ts:5248`);
  - `program`: `diabetes` | `ostomy`; `group`: `glucose_strip` | `lancet` | `insulin_syringe` | `pen_needle` | `ostomy_bag` | `ostomy_accessory` | `other`;
  - perfil: `insulin_type` (`nph` | `regular` | `analog` | `none`), `applications_per_day` e `measurements_per_day` (inteiros 0–10), `stoma_type` (`colostomy` | `ileostomy` | `urostomy` | `other`);
  - inscrição `status` `active` | `inactive` | `closed`, `origin` `consultation` | `external_report` | `import`;
  - regra `kind` `fixed` | `per_prescription`, `factor_field` `measurements_per_day` | `applications_per_day`, `period_days` padrão 30, `cap` opcional;
  - aparelho `glucometer` | `cgm_reader`; empréstimo `active` | `returned` | `replaced` | `lost`.

  **Código, grupo ou situação desconhecidos aparecem crus, nunca somem.**
- **Frases fixas** (letra por letra, contrato §1 e §8):
  - interruptores: "Insumos na farmácia" e "Programas de insumos (diabetes e estomia)";
  - seção da consulta: "Programas de insumos";
  - itens do menu: "Retirada de insumos", "Inscrição com laudo externo", "Aparelhos em comodato", "Importar pacientes dos programas";
  - painel: "Regras de cota".

  Toda outra frase deste plano é **proposta** e está marcada assim.
- **Códigos de recusa** (contrato §4–§6 e plano do api, Valor 4 e Tasks 8–16, pela Q10): `quota_exceeded`, `ostomy_return_not_allowed`, `not_ostomy_service`, `patient_without_cns`, `sigtap_max_quantity`, `supply_without_sigtap`, `enrollment_not_active`, `enrollment_expired`, `enrollment_exists` (409), `supply_enrollment_changed`, `supply_quota_rules_changed`, `device_loan_changed`, `device_model_changed`, `supply_enrollment_import_changed`, `serial_in_use` (todos 409), `item_not_in_program`, `invalid_profile`, `invalid_cid10`, `cid_not_in_program`, `invalid_validity`, `invalid_quantity`, `term_required`, `enroller_not_allowed` (403), `override_requires_pharmacist` (403), `override_code_invalid` (401), `invalid_file`, `file_too_large` (413), `expected_version_required`, os da unidade (`ostomy_service_not_allowed_in_ubs`, `ostomy_service_requires_cnes`, `ostomy_responsible_not_linked`), `feature_disabled` e `mfa_required`. As frases são do dashboard (Task 2).
- **Portões** (contrato §1, §3; decisão 11):
  - `canUseSupplies(user)` = `canUsePharmacy(user)` e `pharmacy_supplies` na sessão; `canUseSupplyPrograms(user)` = `canUseSupplies(user)` e `supply_programs`. A sessão traz as chaves **ligadas**, não as utilizáveis (`src/lib/features.ts:77-86`): o api responde `403 feature_disabled` quando falta o prontuário.
  - Inscrever na consulta: CBO da consulta `225*` (médico) ou `2235*` (enfermeiro) (`ENROLLER_CBO_PREFIXES`), só na consulta `general`: nem odontológica nem de urgência (o api recusa outra com `invalid_request` path `supply_programs`).
  - Retirada e comodato: `pharmacist` ou `pharmacy_assistant` do local (`locationRoles`). Exceção de cota: o farmacêutico, na própria sessão; o auxiliar, com o farmacêutico do local escolhido e o código dele (Q10).
  - Inscrição com laudo externo: só o `pharmacist`.
  - Regras de cota, modelos de aparelho e importação: o `municipal_admin`.
  - Ponto de estomia na unidade: o `municipal_admin`, com `supply_programs` ligado. O CNS e o teto do SIGTAP só valem com `procedure_billing` (Q11): a tela não os confere.
- **Formato** (recusar, nunca cortar; o api confere de novo; valores do plano do api, Valor 2):
  - CID do diabetes: prefixo `E10`–`E14` ou `O24` (Q7, `config/supplies/programs.yml`), regra `^(E1[0-4]|O24)\d?$` depois do `trim`, da maiúscula e sem o ponto; CID da estomia: qualquer CID-10 no formato `^[A-Z]\d{2}\d?$`;
  - aplicações e medições por dia: inteiros 0–10 (`PER_DAY`);
  - validade: na consulta, 3 a 12 meses a partir de hoje (`VALIDITY_MONTHS`); no laudo externo, até 3 meses e depois de hoje (`EXTERNAL_MAX_MONTHS`); a tela oferece meses num `select` (consulta 3–12; laudo 1–3) e manda a data `AAAA-MM-DD`;
  - justificativa da cota acima da regra e da exceção no balcão: 10–500 caracteres depois do `trim`;
  - quantidade da retirada: **inteira** positiva (o api recusa `1.5` com `invalid_quantity`), até o saldo do lote; até 20 linhas (`MAX_ITEMS`);
  - quem retira por terceiro: nome 3–120 e documento `^[0-9A-Za-z./-]{4,30}$` (`COLLECTOR_*`), **os dois ou nenhum**;
  - código do farmacêutico (TOTP): `^\d{6}$`;
  - número de série e boletim de ocorrência: `^[0-9A-Za-z./-]{1,40}$` (`SERIAL`, `POLICE_REPORT`);
  - conselho do laudo: `CRM` ou `COREN` (`REPORT_COUNCILS`);
  - regra de cota: `quantity` e `cap` inteiros 1–9999 (`QUOTA_QUANTITY`), `period_days` 1–365, até 100 regras; `fixed` só com `quantity` (sem fator nem teto), `per_prescription` só com `factor_field` (teto opcional).
- **Step-up** (`SensitiveAction` com `requiresStepUp`) em três escritas: gravar as regras de cota (Q8), gravar o modelo de aparelho (Q12) e aplicar a importação (contrato §3). A exceção do auxiliar **não** usa o `SensitiveAction`: o código é do farmacêutico, não da sessão (Q10). Retirada, devolução, comodato e inscrição pelo laudo não pedem step-up. Toda escrita leva corpo JSON (`{}` quando não há campo).
- **Concorrência:**
  - devolver, trocar e perder o aparelho, editar o modelo, gravar as regras e aplicar ou descartar a importação mandam `expected_version`; o bloco da consulta não leva versão (o `Materialize` revalida na finalização, Valor 5);
  - um `409 *_changed` mostra a frase, faz **uma** releitura e mantém o digitado, sem reenvio;
  - botão de escrita desabilitado enquanto a chamada corre: duplo clique manda um pedido só.
- **Sigilo** (contrato §9):
  - CID, perfil, laudo externo, justificativa, documento de quem retira e o código do farmacêutico só no corpo, nunca em URL, cache, log, storage ou `console`;
  - na URL vão só ids, situação, programa e o cursor (o `patient_id` é id, como no 27a: `listDispensations`, `src/lib/api.ts:5149-5153`);
  - as listas de inscrições (farmácia e painel) não trazem CID nem perfil; o CID e o perfil só vêm no detalhe `GET /supply_enrollments/:id`, lido para renovar (Q7);
  - a planilha e o base64 só vivem no estado do formulário; o nome do arquivo não vai ao api.
- **Leituras:** inscrições, histórico e empréstimos são **nominais** (`NOMINAL_QUERY`, `gcTime: 0`, lidos ao abrir e depois de cada escrita); regras, modelos e importações são `STATIC_QUERY`. **Nenhum polling.**
- **Interface:**
  - toda tela nova entra em `NAV_GROUPS` **e** em `CATALOG.screens` (o `catalog.test.ts` cobra);
  - `DataTable` com `card: "title"`/`"actions"`, e 375 px sem rolagem horizontal;
  - cores só por papéis do tema (`Tag` com `tone`), com as guardas `src/theme/*.test.ts` verdes;
  - componentes existentes: `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `PharmacyGate`, `LocationPick`, `ItemSearch`, `PersonStart` com `PHARMACY_PERSON_SOURCE`, `saveFile` (`src/modules/lab/saveFile.ts`).
- **CI (lição do `443faca`):** todo `vi.mock("…/lib/api", async (orig) => …)` inclui `fetchCurrentSession: vi.fn()` resolvendo o usuário.
- **Datas:** "hoje" vem de `todayInCity` (`src/lib/campaigns.ts:111`), por prop nas telas. Testes: `TODAY27S = "2026-11-26"`, `NOW27S = new Date("2026-11-26T10:00:00-03:00")` (`TZ=America/Sao_Paulo`).
- **Testes:** só `$R/bin/test dashboard -w dashboard-mod27s [arquivos do vitest]`, nunca `vitest` direto. Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge, push e tag só com autorização explícita.
- **Board #1:** o dashboard é superfície dos cards do 27 (insumos) criados pelo plano do api. Card vai a Done só quando o **último** app dele termina, e a Verified depois da prova (Task 14).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`): `dashboard · dashboard-mod27s/feat/mod-27-supplies · Task N · <pronto> · <divergência nova>`.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod27s`, `G=/opt/homebrew/bin/git`, `A=$R/apps/api/.claude/mod27s`.
- Worktree **fora de `apps/dashboard`** (o compose monta `./apps/dashboard:/app`, e um `node_modules` ali duplica o React no Vite). Branch `feat/mod-27-supplies` a partir de `origin/main`, com `npm ci`. Os caminhos das tasks são relativos a `$W`.
- Testes sem jest-dom (`not.toBeNull()`, `toBe()`), com `globals: false` e `cleanup` no `afterEach`.
- Fixtures: `renderWithProviders`/`sessionWith` (`src/test/campaignFixtures.tsx`); `phUser`, `UBS_FARM`, `CENTRAL`, `stockLot`, `lotBalance`, `position`, `dispensation` (`src/test/pharmacyFixtures.ts`); `record`, `consultation`, `options` (`src/test/consultationFixtures.ts`); novas em `src/test/suppliesFixtures.ts` (Task 1).

## Review Focus

1. **Retirada acima da cota pelo auxiliar, com o código errado ou com duplo clique.** Espera-se:
   - a tela já mostra "acima da cota" antes de enviar e pede justificativa, farmacêutico e código;
   - um pedido só; o código sai do estado no instante do envio;
   - `422 quota_exceeded` (corrida de saldo) abre a exceção e relê as inscrições uma vez;
   - código recusado (`401 override_code_invalid`, que não é sessão vencida) volta ao campo do código, sem reenviar;
   - o código nunca vai a URL, cache nem `console`.

   Teste: Task 7, "exceção do auxiliar: farmacêutico e código, um pedido, código limpo, recusa no campo".
2. **Rascunho da consulta com a inscrição incompleta, ou aberta pela farmácia ao mesmo tempo.** Espera-se:
   - a linha incompleta **não** vai no PATCH (o `SaveDraft` recusaria o rascunho inteiro), e "Finalizar" lista o que falta;
   - `409 enrollment_exists` na finalização (a farmácia abriu a mesma inscrição entre o rascunho e a finalização; plano do api, Review Focus 4) mostra a frase, relê as inscrições do paciente uma vez, a consulta continua rascunho e nada é refinalizado sozinho.

   Teste: Task 5, "incompleto fica fora do PATCH e trava a finalização; inscrição aberta ao mesmo tempo relê".
3. **Devolução de bolsa de estomia e de tira.** Espera-se: a bolsa não tem "Registrar devolução"; a tira tem, e o saldo da inscrição é relido; `422 ostomy_return_not_allowed` (api mais rígido) mostra a frase. Teste: Task 8, "bolsa sem devolução; tira devolve e relê o saldo".
4. **Insumo e medicamento misturados.** Espera-se:
   - a busca do insumo só aparece onde o insumo cabe (entrada, abertura, mínimos, retirada);
   - a receita (`DispenseForm`, `ExternalPrescriptionForm`) continua só com medicamento;
   - a linha de insumo vai só com `supply_catalog_item_id`, nunca com `catalog_item_id`.

   Teste: Task 4, "insumo só onde cabe; linha com a chave do insumo".
5. **Retirada por terceiro pela metade.** Espera-se: só o nome, ou só o documento, é recusado antes de enviar; nome e documento vão no corpo, aparados; sem a marca, `collector` não vai. Teste: Task 7, "terceiro: os dois ou nenhum".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.supplies.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/suppliesFixtures.ts` | tipos e cliente; extensões do 19, 20, 27a e unidades; rótulos dos interruptores; fixtures | 1 |
| `src/lib/suppliesErrors.ts`, `src/lib/suppliesErrors.test.ts`, `src/lib/supplies.ts`, `src/lib/supplies.test.ts`, `src/lib/supplyPrograms.ts`, `src/lib/supplyPrograms.test.ts` | frases; portões, rótulos, chaves, regras de tela | 2 |
| `src/shell/modules.ts`, `src/shell/modules.supplies.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/supplies/SuppliesGate.tsx`, `src/modules/supplies/SuppliesGate.test.tsx`, cascas das 4 telas | quatro itens no grupo Farmácia e o portão | 3 |
| `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts`, `src/modules/pharmacy/ItemSearch.tsx`, `src/modules/pharmacy/ReceiptForm.tsx`, `src/modules/pharmacy/MinimumsPanel.tsx`, `src/modules/pharmacy/ReceiptForm.supply.test.tsx`, `src/modules/pharmacy/MinimumsPanel.supply.test.tsx` | insumo no estoque | 4 |
| `src/lib/consultation.ts`, `src/modules/consultation/SupplyProgramsField.tsx`, `src/modules/supplies/ProfileFields.tsx`, `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationEditor.supplies.test.tsx` | "Programas de insumos" no rascunho | 5 |
| `src/modules/consultation/SupplyEnrollmentsCard.tsx`, `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`, `src/modules/consultation/ConsultationWorkspace.supplies.test.tsx`, `src/lib/timeline.supplies.test.ts` | inscrições no painel e na linha do tempo | 6 |
| `src/modules/SupplyWithdrawal.tsx`, `src/modules/supplies/EnrollmentCard.tsx`, `src/modules/supplies/SupplyDispenseForm.tsx`, `src/modules/SupplyWithdrawal.test.tsx` | "Retirada de insumos" | 7 |
| `src/modules/pharmacy/PatientHistory.tsx`, `src/modules/pharmacy/ReturnForm.tsx`, `src/modules/pharmacy/PatientHistory.supply.test.tsx` | histórico e devolução do insumo | 8 |
| `src/modules/SupplyExternalEnrollment.tsx`, `src/modules/supplies/ExternalEnrollmentForm.tsx`, `src/modules/SupplyExternalEnrollment.test.tsx` | "Inscrição com laudo externo" | 9 |
| `src/modules/DeviceLoans.tsx`, `src/modules/supplies/DeviceLoanForm.tsx`, `src/modules/DeviceLoans.test.tsx` | "Aparelhos em comodato" | 10 |
| `src/modules/pharmacy/QuotaRulesPanel.tsx`, `src/modules/pharmacy/DeviceModelsPanel.tsx`, `src/modules/PharmacyAdmin.tsx`, `src/modules/PharmacyAdmin.supplies.test.tsx` | "Regras de cota" e modelos de aparelho | 11 |
| `src/lib/supplyImport.ts`, `src/lib/supplyImport.test.ts`, `src/modules/SupplyEnrollmentImport.tsx`, `src/modules/supplies/ImportReport.tsx`, `src/modules/SupplyEnrollmentImport.test.tsx` | "Importar pacientes dos programas" | 12 |
| `src/modules/attendance/UnitForm.tsx`, `src/modules/attendance/Units.tsx`, `src/modules/Attendance.tsx`, `src/modules/attendance/UnitForm.ostomy.test.tsx` | ponto de estomia na unidade | 13 |
| — | suíte, build, varreduras, revisão, prova no navegador (5226 → 3065) | 14 |
| — | board, relato, parada antes do merge | 15 |

**Estratégia de teste:**
- **Cliente** (Task 1): `fetch` falso (`ok`, `urlOf`, `initOf`, `bodyOf`, `writeOf`, como em `src/lib/api.pharmacy.test.ts`), conferindo URL, método e corpo; CID, perfil, laudo, justificativa, quem retira e o código só no corpo (um `SENSITIVE` próprio contra a URL).
- **Regras puras** (Tasks 2 e 12): tabela de casos, com as bordas de formato.
- **Menu** (Task 3): por papel e por interruptor.
- **Telas** (Tasks 4–13): pelo ponto de entrada, com `renderWithProviders`, `vi.mock` do api com `fetchCurrentSession` e sessão por papel e por interruptor. Cobrem o caminho feliz, cada recusa que muda a tela, o duplo envio, o 409 com uma releitura e o que **não** pode aparecer (botão para quem não pode, código no cache, linha incompleta no PATCH, planilha enviada quando o arquivo é recusado).
- **Fim** (Task 14): `tsc`, build, suíte, varreduras e prova no navegador, com o api real na 3065 e o Vite na 5226.

O dashboard não testa o que é do api: o cálculo da cota e do saldo, o período a partir da primeira retirada, a soma entre unidades, o teto do SIGTAP, o CNS, o faturamento, a inativação em 180 dias, a validação de cada linha da planilha e o sigilo de evento.

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (com o 26c); board #1; o api `feat/mod-27-supplies`.
- Produces: cards em In Progress; `$W` na branch; base, âncoras, ícones e formas do api anotados.

- [ ] **Step 1: Cards.** Um `gh` simples por vez, sem `$(...)` nem laço:

  ```bash
  gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-27\\.")) | select(.title | test("(?i)insumo|comodato|estomia|materia|cota")) | {id, title, status}'
  ```

  Os cards que o dashboard toca vão a In Progress, com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <In Progress>`. Card ausente: **pare e pergunte** (quem cria é o plano do api).

- [ ] **Step 2: Base e âncoras**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin && $G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/regulationRnds.ts >/dev/null && echo 26c-ok
$G -C $R/apps/dashboard show origin/main:src/lib/bnafar.ts >/dev/null && echo 27b-ok
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export interface StockItem {\|^export type ReceiptItemInput\|^export interface PositionQuery\|^export interface StockMinimum\|^export interface DispensedItem\|^export interface Dispensation {\|^export async function searchPharmacyItems\|^export async function putMinimums\|^export interface ConsultationDraftInput\|^export interface Consultation {\|^export interface ClinicalRecord {\|^export type TimelineKind\|^export interface UnitCapsInput\|^function readBase64\|^async function binaryFetch'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "vector_control" | "zoonoses"\|^  zoonoses:'
$G -C $R/apps/dashboard show origin/main:src/lib/pharmacy.ts | grep -n '^export const itemKey\|^export function receiptItems\|^function receiptLineProblem\|^export const returnRemaining'
$G -C $R/apps/dashboard show origin/main:src/lib/consultation.ts | grep -n '^export function draftFrom\|^export function checkDraft\|^export function finalizeProblems'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '{ id: "pharmacy-admin", label: "Locais e configurações"\|if (item.id === "pharmacy-admin") return'
$G -C $R/apps/dashboard show origin/main:src/lib/timeline.ts | grep -n '^export function timelineKinds\|{ key: "collective_activity"'
```

Expected:
- `26c-ok`. Sem ele, **pare**: o contrato manda começar depois do 26c;
- **anote** se há `27b-ok`: com o 27b na `main`, a linha de insumo também recebe os campos do envio (Task 4, passo 3);
- cada linha achada. Linha mudada: a task dona aplica a mesma intenção sobre o texto real e anota no commit. Uma chave nova depois de `zoonoses` vira a âncora.

- [ ] **Step 3: Worktree, base e ícones**

```bash
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod27s -b feat/mod-27-supplies origin/main
cd $R/.worktrees/dashboard-mod27s && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod27s
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(Needle|FileCertificate|DeviceWatch|FileImport)\.mjs$'
```

Expected: `tsc-ok`; suíte verde (**anote** arquivos e testes); os quatro ícones. Faltou algum: use outro que exista, e o mesmo na Task 3.

- [ ] **Step 4: Formas do api.** Leia, na branch `feat/mod-27-supplies` do api:
  - as rotas: `$G -C $R/apps/api show feat/mod-27-supplies:config/routes.rb | grep -n 'supply\|device_\|pharmacists\|items/search'`;
  - os serializadores da inscrição (`SupplyPrograms::Json.summary` e `.detail`), da dispensação (`Pharmacy::Json.dispensation` e `.supply_item`), do empréstimo e da importação; o `Consultations::Json` (bloco `supply_programs` com `quotas`);
  - o `SupplyPrograms::DraftInput` (Valor 5 do plano do api) e o `Pharmacy::DispenseSupplies` (corpo e `override`);
  - o `status` das entradas `supply_enrollment` da linha do tempo;
  - os specs de request;
  - o `Platform::Features::CATALOG`: as duas chaves e os `requires`.

  Confira tudo contra as formas da Task 1 e os códigos de recusa da Task 2, que seguem o plano do api (Q10). **Onde o api diz outra coisa, vale o api**: a task dona ajusta tipo, fixture e teste e registra no commit, e o relato cita a diferença. Sem branch ainda: siga pelas formas deste plano e repita este passo antes da Task 14.

---

### Task 1: Tipos, cliente HTTP, rótulos dos interruptores e fixtures

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; extensões nas seções do 19, 20, 27a e das unidades), `src/lib/features.ts`
- Create: `src/lib/api.supplies.test.ts`, `src/test/suppliesFixtures.ts`
- Test: `src/lib/features.test.ts`

**Interfaces:**
- Produces: em `src/lib/api.ts`, a seção "Insumos (módulo 27; contrato 2026-11-25 §1–§9, Q1–Q13)". Caminhos por `phPath`/`phWrite` do 27a; listas novas em `{ items, next_before }` (plano do api, Valor 6; as do 27a mantêm o envelope delas). As formas são as do plano do api (Q7–Q12); a Task 0 confere:

```ts
export type SupplyProgram = "diabetes" | "ostomy";
export type SupplyGroup = "glucose_strip" | "lancet" | "insulin_syringe" | "pen_needle" | "ostomy_bag" | "ostomy_accessory" | "other";
export type InsulinType = "nph" | "regular" | "analog" | "none";
export type StomaType = "colostomy" | "ileostomy" | "urostomy" | "other";
export type EnrollmentStatus = "active" | "inactive" | "closed";
export type EnrollmentOrigin = "consultation" | "external_report" | "import";
export interface DiabetesProfile { insulin_type: InsulinType; applications_per_day: number; measurements_per_day: number }
export interface OstomyProfile { stoma_type: StomaType }
export type SupplyProfile = DiabetesProfile | OstomyProfile;
// Cota congelada (Quota.compute): `rule_id` nulo quando veio de `quota_overrides` sem regra.
export interface SupplyQuota { rule_id: string | null; group: SupplyGroup | (string & {}); quantity: number; period_days: number }
// Farmácia (api Task 10): a cota com o saldo do ciclo, sem CID nem perfil.
export interface SupplyQuotaBalance { group: SupplyGroup | (string & {}); quantity: number; period_days: number; available: number; period_start: string | null; period_end: string | null }
export interface EnrollmentDevice { loan_id: string; model: { id: string; name: string; kind: string }; serial_number: string; compatible_item: StockItem | null }
export interface PharmacySupplyEnrollment {
  id: string; program: SupplyProgram | (string & {}); status: EnrollmentStatus | (string & {}); valid_until: string; expired: boolean;
  preferred_unit: { id: string; name: string } | null; last_withdrawal_at: string | null;
  quotas: SupplyQuotaBalance[]; devices: EnrollmentDevice[];
}
// Painel do paciente (Q7; api Task 9): lista sem CID nem perfil; o detalhe traz os dois.
export interface SupplyEnrollmentSummary {
  id: string; version: number; program: SupplyProgram | (string & {}); status: EnrollmentStatus | (string & {}); valid_until: string;
  preferred_unit: { id: string; name: string } | null; quotas: SupplyQuota[]; last_withdrawal_at: string | null;
}
export interface SupplyEnrollmentDetail extends SupplyEnrollmentSummary {
  cid10: string; profile: SupplyProfile; origin: EnrollmentOrigin | (string & {}); external_report: ExternalReportInput | null;
}
// Bloco do rascunho (Q7; api Valor 5): até 4 linhas; `close` só leva `enrollment_id`; a cota do cliente é ignorada.
export type SupplyProgramAction =
  | { action: "open"; program: SupplyProgram; cid10: string; profile: SupplyProfile; valid_until: string; preferred_unit_id: string;
      quota_overrides?: { group: SupplyGroup; quantity: number }[]; override_reason?: string }
  | { action: "renew"; program: SupplyProgram; enrollment_id: string; cid10: string; profile: SupplyProfile; valid_until: string; preferred_unit_id: string;
      quota_overrides?: { group: SupplyGroup; quantity: number }[]; override_reason?: string }
  | { action: "close"; program: SupplyProgram; enrollment_id: string };
// Leitura do rascunho: cada linha volta com a `quotas` calculada pelo api (a cota à vista).
export type SupplyProgramView = SupplyProgramAction & { quotas?: SupplyQuota[] };
export interface CollectorInput { name: string; document: string }
// Q10 (api Task 11): `override` só quando há linha acima do saldo; o farmacêutico na própria sessão não manda `pharmacist_user_id` nem `code`.
export interface SupplyOverride { reason: string; pharmacist_user_id?: string; code?: string }
export interface SupplyDispensationInput {
  location_id: string; enrollment_id: string;
  items: { supply_catalog_item_id: string; lot_id: string; quantity: string }[];
  collector?: CollectorInput; override?: SupplyOverride;
}
export interface LocationPharmacist { user_id: string; name: string | null }
// api Task 10: `CRM` ou `COREN`; só paciente existente (a busca da farmácia cria o paciente do par validado, como no 27a).
export interface ExternalReportInput { professional_name: string; council: "CRM" | "COREN"; council_state: string; council_number: string; issued_on: string }
export interface ExternalEnrollmentInput {
  location_id: string; patient_id: string; program: SupplyProgram; cid10: string; profile: SupplyProfile; preferred_unit_id: string;
  valid_until: string; external_report: ExternalReportInput;
}
export type DeviceKind = "glucometer" | "cgm_reader";
export type DeviceLoanStatus = "active" | "returned" | "replaced" | "lost";
export interface DeviceModel {
  id: string; version: number; name: string; kind: DeviceKind | (string & {}); supplier: string | null;
  compatible_supply_catalog_item_id: string | null; compatible_item?: StockItem | null; active: boolean;
}
export interface DeviceModelInput { name: string; kind: DeviceKind; supplier: string | null; compatible_supply_catalog_item_id: string | null; active: boolean }
// api Task 14: a lista por unidade traz `patient_id` e o aparelho, nunca o nome do paciente nem o boletim.
export interface DeviceLoan {
  id: string; version: number; status: DeviceLoanStatus | (string & {}); model: { id: string; name: string; kind: string }; serial_number: string;
  patient_id: string; health_unit: { id: string; name: string }; term_accepted_at: string; loaned_at: string;
  returned_at: string | null; replaced_by_loan_id: string | null;
}
export interface DeviceLoanInput { location_id: string; patient_id: string; device_model_id: string; serial_number: string; term_accepted: true }
// Q8 (api Task 8): por programa; a ordem da lista é a ordem de aplicação (a primeira que casa vale).
export interface SupplyQuotaRule {
  id?: string; group: SupplyGroup; profile_match: Record<string, string[]>;
  kind: "fixed" | "per_prescription"; quantity: number | null; factor_field: "measurements_per_day" | "applications_per_day" | null;
  period_days: number; cap: number | null;
}
export interface SupplyQuotaRuleSet { program: SupplyProgram; version: number; rules: SupplyQuotaRule[] }
// Q12 (api Task 16): o relatório só traz códigos e contagens; as linhas, número, situação e códigos.
export type SupplyImportStatus = "uploaded" | "validated" | "applied" | "discarded";
export interface SupplyEnrollmentImport {
  id: string; status: SupplyImportStatus | (string & {}); version: number; rows_total: number; rows_valid: number;
  applied_at: string | null; report: { errors_by_code: { code: string; count: number }[] } & Record<string, unknown>;
}
export interface SupplyImportRow { line: number; status: "valid" | "error" | "applied" | (string & {}); error_codes: string[] }
export interface SupplyEnrollmentImportDetail extends SupplyEnrollmentImport { rows: SupplyImportRow[]; next_after: number | null }
```

- Funções (todas `Promise`; rotas do plano do api, Valor 6):

  | Função | Rota |
  |---|---|
  | `listSupplyEnrollments(patientId)` → `PharmacySupplyEnrollment[]` | `GET /pharmacy/supply_enrollments?patient_id=` (`{ items }`) |
  | `createExternalSupplyEnrollment(i)` → `SupplyEnrollmentSummary` | `POST /pharmacy/supply_enrollments` |
  | `recordSupplyDispensation(i)` → `Dispensation` | `POST /pharmacy/supply_dispensations` |
  | `listLocationPharmacists(locationId)` → `LocationPharmacist[]` | `GET /pharmacy/locations/:id/pharmacists` (`{ items }`, Q10) |
  | `listPatientSupplyEnrollments(patientId)` → `SupplyEnrollmentSummary[]`; `getSupplyEnrollment(id)` → `SupplyEnrollmentDetail` | `GET /patients/:id/supply_enrollments`; `GET /supply_enrollments/:id` (Q7, com trilha) |
  | `listDeviceModels()` (ativos); `listAdminDeviceModels()`; `createDeviceModel(i)`; `updateDeviceModel(id, i, expectedVersion)` | `GET /pharmacy/device_models`; `GET`/`POST /pharmacy/admin/device_models`; `PATCH /pharmacy/admin/device_models/:id` (Q12) |
  | `listDeviceLoans(q: { healthUnitId: string; status?; before? })` | `GET /pharmacy/device_loans?health_unit_id=&status=&before=&limit=` |
  | `createDeviceLoan(i)`; `returnDeviceLoan(id, v)`; `replaceDeviceLoan(id, { expected_version, device_model_id, serial_number })`; `reportDeviceLoanLost(id, { expected_version, police_report_number })` | `POST /pharmacy/device_loans`, `…/:id/return`, `…/:id/replace`, `…/:id/loss` |
  | `getSupplyQuotaRules(program)` → `SupplyQuotaRuleSet`; `putSupplyQuotaRules(program, rules, expectedVersion)` | `GET /pharmacy/admin/supply_quota_rules?program=`; `PUT` com `{ program, expected_version, rules }` (Q8) |
  | `uploadSupplyEnrollmentImport(file: Blob)` → `SupplyEnrollmentImportDetail` | `POST /pharmacy/supply_enrollment_imports` com `{ file_base64 }` (o `readBase64` do módulo; sem nome) |
  | `getSupplyEnrollmentImport(id, q: { after?; status?: "error" })` | `GET …/:id?after=&status=` |
  | `applySupplyEnrollmentImport(id, v)`; `discardSupplyEnrollmentImport(id, v)` | `POST …/:id/apply|discard` com `{ expected_version }` |
  | `fetchSupplyEnrollmentImportTemplate()` | `binaryFetch` em `/pharmacy/supply_enrollment_imports/template.csv` (`text/csv`) |

  Não há lista de importações no api (Valor 6): a tela abre a que acabou de enviar.

- Extensões:
  - 27a (Q9; api Tasks 4, 5, 11):
    - `StockItem.item_kind` + `"supply"`; `StockItem.supply_catalog_item_id?: string | null`; `StockItem.group?: SupplyGroup | (string & {}) | null` (o `Json.supply_item` chama o campo de `group`);
    - `searchPharmacyItems(q, kind?: "vector_control" | "supply")`: com `"supply"`, o corpo é `{ q, item_kind: "supply" }`; com `"vector_control"`, o `{ q, domain }` de hoje (os dois juntos o api recusa);
    - `ReceiptItemInput` ganha o ramo `{ supply_catalog_item_id: string; catalog_item_id?: never; vector_product_id?: never; package?: PackageInput }` (a embalagem segue a regra do medicamento; a Task 0 confere);
    - `PositionQuery.supplyCatalogItemId` → `supply_catalog_item_id`;
    - `StockMinimum.catalog_item_id` passa a `string | null`, com `supply_catalog_item_id?: string | null`; `putMinimums` aceita linhas `{ supply_catalog_item_id, minimum }`;
    - `Dispensation.prescription` passa a `… | null`, com `supply_enrollment?: { id: string; program: string } | null` e `collector?: { name: string } | null` (nunca o documento);
    - `DispensedItem.catalog_item_id` passa a `string | null`, com `supply_catalog_item_id?: string | null`.
  - 19: `ConsultationDraftInput.supply_programs?: SupplyProgramAction[]`; `Consultation.supply_programs?: SupplyProgramView[]` (só no rascunho; a finalização zera `draft_items`).
  - 20: `TimelineKind` + `"supply_enrollment"` (Q7).
  - Unidades (Q3, Q11; api Task 13): `HealthUnit.ostomy_service?: boolean` e `ostomy_responsible?: { id: string; name: string } | null`; `UnitCapsInput` ganha `ostomy_service?: boolean` e `ostomy_responsible_professional_id?: string | null` (cada uma só vai quando recebida).
- `src/lib/features.ts`: `FeatureKey` + `"pharmacy_supplies" | "supply_programs"`; `FEATURE_LABEL` `pharmacy_supplies: "Insumos na farmácia"`, `supply_programs: "Programas de insumos (diabetes e estomia)"`, com o comentário: "Módulo 27 (insumos): pharmacy_supplies exige pharmacy; supply_programs exige pharmacy_supplies e clinical_record. O CNS e o teto do SIGTAP da bolsa só valem com procedure_billing (Q11)."
- Fixtures (`src/test/suppliesFixtures.ts`):
  - `TODAY27S`, `NOW27S`;
  - `SUPPLY_FEATURES = [ "pharmacy", "pharmacy_supplies", "supply_programs", "clinical_record" ]` e `supplyUser(roles = [ "pharmacist" ], features = SUPPLY_FEATURES)`;
  - `STRIP: StockItem` (`item_kind: "supply"`, `supply_catalog_item_id: "sc1"`, `group: "glucose_strip"`, `catalog_item_id: null`, `dispensing_unit: "unidade"`), `LANCET` (`"sc2"`, `lancet`), `OSTOMY_BAG` (`"sc3"`, `ostomy_bag`);
  - `pharmacyEnrollment(o?)` (diabetes ativa; tiras `quantity: 90`, `available: 40`; lancetas `quantity: 100`, `available: 100`; um aparelho com `compatible_item: STRIP`), `ostomyEnrollment(o?)`;
  - `enrollmentSummary(o?)` e `enrollmentDetail(o?)` (com `cid10: "E119"` e perfil NPH);
  - `supplyDispensation(o?)` (`prescription: null`, `supply_enrollment: { id: "se1", program: "diabetes" }`, um item de tira);
  - `deviceModel(o?)`, `deviceLoan(o?)`, `quotaRules(o?)` (`program: "diabetes"`, `version: 3`), `supplyImport(o?)` (`validated`, `version: 1`, 10 linhas, 8 válidas), `importRows(n, o?)`;
  - `PHARMACISTS = [ { user_id: "u-ph1", name: "Ana Farmacêutica" } ]`.

- [ ] **Step 1: Testes que falham** — `src/lib/api.supplies.test.ts` (molde `src/lib/api.pharmacy.test.ts`; `SENSITIVE = /E11|nph|Joana|RG12345|justific|123456|laudo/i`):

```ts
it("retirada: lotes, motivo, terceiro e código só no corpo", async () => {
  fetchMock.mockReturnValueOnce(ok(supplyDispensation()));
  await recordSupplyDispensation({ location_id: "l2", enrollment_id: "se1", items: [ { supply_catalog_item_id: "sc1", lot_id: "lt9", quantity: "100" } ],
    collector: { name: "Joana Lima", document: "RG12345" }, override: { reason: "viagem longa do paciente", pharmacist_user_id: "u-ph1", code: "123456" } });
  expect(writeOf()).toEqual([ "/pharmacy/supply_dispensations", "POST", "application/json" ]);
  expect(bodyOf().override).toEqual({ reason: "viagem longa do paciente", pharmacist_user_id: "u-ph1", code: "123456" });
  expect(SENSITIVE.test(urlOf().href)).toBe(false);
});
it("inscrições do paciente: só o id na URL, na farmácia e no prontuário", async () => {
  fetchMock.mockReturnValueOnce(ok({ items: [ pharmacyEnrollment() ] }));
  expect(await listSupplyEnrollments("p1")).toHaveLength(1);
  expect(urlOf().pathname + urlOf().search).toBe("/pharmacy/supply_enrollments?patient_id=p1");
  /* listPatientSupplyEnrollments("p1") → /patients/p1/supply_enrollments; getSupplyEnrollment("se1") → /supply_enrollments/se1 */
});
it("comodato: perda com o boletim no corpo; lista por unidade e situação", async () => { /* reportDeviceLoanLost("dl1", { expected_version: 2, police_report_number: "BO-778" }) → POST /pharmacy/device_loans/dl1/loss; listDeviceLoans({ healthUnitId: "u1", status: "active" }) → ?health_unit_id=u1&status=active&limit=50 */ });
it("regras de cota: GET por programa; PUT com o programa, a versão e as regras", async () => { /* getSupplyQuotaRules("diabetes") → ?program=diabetes; putSupplyQuotaRules("diabetes", [...], 3) → { program: "diabetes", expected_version: 3, rules: [...] } */ });
it("importação: base64 sem nome; aplicar com expected_version; modelo pelo binaryFetch", async () => { /* molde do 27b */ });
it("estoque: busca pelo item_kind; posição e mínimos pela chave do insumo", async () => {
  fetchMock.mockReturnValueOnce(ok({ items: [ STRIP ] })); await searchPharmacyItems("tira", "supply");
  expect(bodyOf()).toEqual({ q: "tira", item_kind: "supply" });
  /* searchPharmacyItems("larvicida", "vector_control") → { q, domain: "vector_control" }; listPositions({ locationId: "l2", supplyCatalogItemId: "sc1" }) → supply_catalog_item_id=sc1 */
});
```

Em `src/lib/features.test.ts`: os dois rótulos, letra por letra.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod27s src/lib/api.supplies.test.ts src/lib/features.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS. O `tsc` vai acusar os usos de `Dispensation.prescription` e `DispensedItem.catalog_item_id` que agora aceitam nulo: trate cada um com o mesmo comportamento de hoje para o medicamento (o nulo só existe no insumo) e anote no commit.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.supplies.test.ts src/lib/features.ts src/lib/features.test.ts src/test/suppliesFixtures.ts
$G commit -m "feat: add the supplies API client with the stock, dispensation, consultation and unit extensions" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Inclua no `add` cada arquivo que o Step 4 tocou.)

---

### Task 2: Frases, portões, rótulos e regras de tela

**Files:**
- Create: `src/lib/suppliesErrors.ts`, `src/lib/suppliesErrors.test.ts`, `src/lib/supplies.ts`, `src/lib/supplies.test.ts`, `src/lib/supplyPrograms.ts`, `src/lib/supplyPrograms.test.ts`

**Interfaces:**
- Consumes: `hasFeature`, `featureLabel` (Task 1); `canUsePharmacy`, `pharmacyError`, `pharmacyRefusals`, `toApiQuantity`, `cents`, `prescriberProblem`, `issuedOnProblem`, `UFS`, `Problem`, `NOMINAL_QUERY`, `STATIC_QUERY` (`src/lib/pharmacy.ts`).
- Produces (`src/lib/suppliesErrors.ts`):
  - `SUPPLIES_DISABLED = "os insumos na farmácia estão desligados nesta cidade"` e `SUPPLY_PROGRAMS_DISABLED = "os programas de insumos estão desligados nesta cidade"` (propostas);
  - `SUPPLIES_ERRORS: Record<string, string>`: uma frase por código, distintas, sem `_`, sem dado de pessoa;
  - `suppliesError(err)`: `SUPPLIES_ERRORS` primeiro; `feature_disabled` com `pharmacy_supplies`/`supply_programs` → as frases acima; o resto cai em `pharmacyError`;
  - `suppliesRefusals(err)`: como `pharmacyRefusals`, com a tabela daqui primeiro.

  Os códigos são os do contrato e do plano do api (Valor 4 e Tasks 8–16, Q10); as frases são todas **proposta**:

| Código | Frase |
|---|---|
| `quota_exceeded` | "passa da cota do período — a entrega acima da cota pede a liberação do farmacêutico, com justificativa" |
| `ostomy_return_not_allowed` | "bolsa de estomia entregue não é devolvida" |
| `not_ostomy_service` | "esta unidade não é ponto de atenção à pessoa estomizada — a bolsa só sai num ponto marcado pela cidade" |
| `patient_without_cns` | "paciente sem CNS — a bolsa não sai sem o CNS, que vai no BPA-I; atualize o cadastro" |
| `sigtap_max_quantity` | "passa da quantidade máxima do SIGTAP para este paciente no mês" |
| `supply_without_sigtap` | "este material ainda não tem o código SIGTAP da bolsa — a entrega espera a revisão do catálogo" |
| `enrollment_exists` | "o paciente já tem inscrição ativa neste programa — renove ou encerre a que existe" |
| `enrollment_not_active` | "inscrição inativa ou encerrada — a reativação é na consulta ou pela inscrição com laudo" |
| `enrollment_expired` | "inscrição vencida — renove na consulta" |
| `supply_enrollment_changed` | "a inscrição mudou enquanto a tela estava aberta — ela foi relida; confira e repita" |
| `supply_quota_rules_changed` | "as regras de cota foram salvas por outra pessoa — elas foram relidas; confira antes de salvar" |
| `device_loan_changed` | "o empréstimo mudou enquanto a tela estava aberta — a lista foi relida" |
| `device_model_changed` | "o modelo foi salvo por outra pessoa — ele foi relido; confira antes de salvar" |
| `supply_enrollment_import_changed` | "a importação mudou enquanto a tela estava aberta — ela foi relida; confira e repita" |
| `serial_in_use` | "este número de série já está emprestado a outro paciente" |
| `term_required` | "marque o termo de comodato assinado pelo paciente" |
| `item_not_in_program` | "este material não está na cota da inscrição" |
| `invalid_profile` | "confira o perfil: tipo de insulina, aplicações e medições por dia (0 a 10)" |
| `invalid_cid10` | "CID-10 inválido" |
| `cid_not_in_program` | "CID fora da lista do programa (diabetes: E10 a E14 ou O24)" |
| `invalid_validity` | "validade fora do prazo do programa" |
| `invalid_quantity` | "quantidade inteira maior que zero" |
| `enroller_not_allowed` | "só médico (CBO 225) ou enfermeiro (CBO 2235) inscreve no programa" |
| `override_requires_pharmacist` | "a entrega acima da cota pede a liberação de um farmacêutico deste local" |
| `override_code_invalid` | "o código do farmacêutico não valeu — peça o código atual do autenticador dele" |
| `invalid_file` | "o arquivo não pôde ser lido — confira se é o CSV UTF-8 da planilha padrão" |
| `file_too_large` | "o arquivo passa de 2 MB — divida a planilha" |
| `ostomy_service_not_allowed_in_ubs` | "UBS não é ponto de atenção à pessoa estomizada" |
| `ostomy_service_requires_cnes` | "o ponto de estomia precisa do CNES da unidade" |
| `ostomy_responsible_not_linked` | "o responsável precisa ter vínculo ativo nesta unidade" |
| `expected_version_required` | "a tela está desatualizada — recarregue e repita" |

- Produces (`src/lib/supplies.ts`):
  - **Portões:** `canUseSupplies(user)`, `canUseSupplyPrograms(user)`, `canMarkOstomyService(user)` (= `canUseSupplyPrograms` e `municipal_admin`).
  - **Chaves:** `SUPPLY_ENROLLMENTS_KEY`, `PATIENT_SUPPLY_ENROLLMENTS_KEY`, `DEVICE_LOANS_KEY`, `DEVICE_MODELS_KEY`, `QUOTA_RULES_KEY`, `SUPPLY_IMPORT_KEY`, `LOCATION_PHARMACISTS_KEY`; reexporta `NOMINAL_QUERY` e `STATIC_QUERY`.
  - **Rótulos** (propostos; desconhecido cru): `programLabel` (`diabetes` "Diabetes", `ostomy` "Estomia"); `groupLabel` (`glucose_strip` "tira de glicemia", `lancet` "lanceta", `insulin_syringe` "seringa de insulina", `pen_needle` "agulha de caneta", `ostomy_bag` "bolsa de estomia", `ostomy_accessory` "acessório de estomia", `other` "outro material"); `enrollmentStatusLabel`/`Tone` (`active` "ativa"/ok, `inactive` "inativa"/warn, `closed` "encerrada"/neutral); `originLabel` (`consultation` "consulta", `external_report` "laudo externo", `import` "importação"); `insulinLabel` (`nph` "NPH", `regular` "regular", `analog` "análogo", `none` "sem insulina"); `stomaLabel` (`colostomy` "colostomia", `ileostomy` "ileostomia", `urostomy` "urostomia", `other` "outro"); `deviceKindLabel` (`glucometer` "glicosímetro", `cgm_reader` "leitor de sensor contínuo"); `loanStatusLabel`/`Tone` (`active` "emprestado", `returned` "devolvido", `replaced` "trocado", `lost` "perdido").
  - **Cota:** `quotaLine(q: { group; quantity; period_days }): string` (proposta `"<n> · <grupo> a cada <d> dias"`); `quotaOf(e, group): SupplyQuotaBalance | null`.
  - **Retirada:**
    - `type WithdrawalLine = { key: string; item: StockItem | null; lotId: string; quantity: string }`;
    - `withdrawalUsage(lines): Record<string, number>`: soma inteira por `item.group`;
    - `overQuotaGroups(e, lines): string[]`: os grupos em que o uso passa de `available`;
    - `withdrawalProblems(d: { lines; balances: Record<lotId, string>; enrollment; overReason; asAssistant: boolean; pharmacistId: string; code: string; collector: { on; name; document } }): Problem[]`:
      - de 1 a 20 linhas com quantidade;
      - item, lote e quantidade **inteira** `^\d+$` ≥ 1 por linha, até o saldo do lote;
      - grupo do item fora das `quotas` → a frase de `item_not_in_program`;
      - com grupo acima da cota: justificativa 10–500; com `asAssistant`, farmacêutico escolhido e código `^\d{6}$`;
      - com `collector.on`: nome 3–120 e documento `^[0-9A-Za-z./-]{4,30}$`, os dois;
    - `withdrawalBody(...)`: as linhas com `supply_catalog_item_id`, `lot_id` e `quantity`; `override` só com grupo acima da cota (`{ reason }` para o farmacêutico; `{ reason, pharmacist_user_id, code }` para o auxiliar); `collector` só com a marca, aparado.
  - **Comodato:** `SERIAL = /^[0-9A-Za-z./-]{1,40}$/`; `serialProblem`, `policeReportProblem` (texto ou `null`).

- Produces (`src/lib/supplyPrograms.ts`):
  - `DIABETES_CID = /^(E1[0-4]|O24)\d?$/` e `OSTOMY_CID = /^[A-Z]\d{2}\d?$/` (Q7); `normalizeCid(raw)` (maiúscula, sem ponto nem espaço);
  - `canEnrollSupplies(user, cboCode)`: `canUseSupplyPrograms(user)` e CBO `225*` ou `2235*`;
  - `MAX_PROGRAM_LINES = 4`;
  - `type SupplyProgramDraft = { key: string; action: "open" | "renew" | "close"; program: SupplyProgram; enrollmentId: string | null; cid: string; insulin: InsulinType | ""; applications: string; measurements: string; stoma: StomaType | ""; unitId: string; months: string; override: boolean; overrideQuotas: Record<string, string>; overrideReason: string }`;
  - `draftsFrom(views: SupplyProgramView[] | undefined, today): SupplyProgramDraft[]` (o caminho de volta do rascunho salvo; os meses saem da `valid_until`);
  - `newProgramDraft(program, enrollment?: SupplyEnrollmentDetail)`: `open` sem inscrição; `renew` com ela (pré-preenche CID, perfil, unidade); `months` padrão `"12"`;
  - `programDraftProblems(d, today): string[]` (as regras de formato; `close` não tem problema);
  - `supplyProgramsPayload(list, today): SupplyProgramAction[]`: **só as linhas sem problema**; `valid_until` = hoje + meses (`addMonths` em `AAAA-MM-DD`, no fim do mês quando o dia não existe); `quota_overrides` e `override_reason` só com `override`;
  - `supplyProgramsMissing(list, today): string[]`: `"Programas de insumos: <primeiro problema>"` por linha incompleta, para o `missing` do editor;
  - `supplyProgramsFinalizeError(err): string | null`: recusa com `path` `supply_programs…` ou código de `SUPPLIES_ERRORS` → a frase; senão `null`;
  - **Regras de cota:** `type RuleDraft` (strings da tela) e `EMPTY_RULE(program)`; `ruleProblems(r, program): string[]`:
    - `fixed` exige `quantity` e não tem fator nem teto;
    - `per_prescription` exige `factor_field` e não tem `quantity`; `applications_per_day` só no diabetes; o teto é opcional;
    - `period_days` 1–365; `quantity` e `cap` 1–9999; o grupo do programa (`PROGRAM_GROUPS` do api);
  - `rulesBody(list): SupplyQuotaRule[]` (na ordem da tela).

- [ ] **Step 1: Testes que falham.**
  - `suppliesErrors.test.ts`: cada código da tabela tem frase; frases distintas, sem `_`; desconhecido cai no `pharmacyError`; `feature_disabled` de `supply_programs` → `SUPPLY_PROGRAMS_DISABLED`.
  - `supplies.test.ts`:

```ts
it("acima da cota: o auxiliar manda override com o farmacêutico e o código", () => {
  const e = pharmacyEnrollment(); // tiras available 40; lancetas available 100
  const lines = [ { key: "a", item: STRIP, lotId: "lt9", quantity: "50" }, { key: "b", item: LANCET, lotId: "lt8", quantity: "10" } ];
  expect(overQuotaGroups(e, lines)).toEqual([ "glucose_strip" ]);
  const body = withdrawalBody({ ...base, lines, overReason: " viagem longa do paciente ", asAssistant: true, pharmacistId: "u-ph1", code: "123456" });
  expect(body.items).toEqual([ { supply_catalog_item_id: "sc1", lot_id: "lt9", quantity: "50" }, { supply_catalog_item_id: "sc2", lot_id: "lt8", quantity: "10" } ]);
  expect(body.override).toEqual({ reason: "viagem longa do paciente", pharmacist_user_id: "u-ph1", code: "123456" });
  expect(withdrawalBody({ ...base, lines, overReason: "viagem longa do paciente", asAssistant: false }).override).toEqual({ reason: "viagem longa do paciente" });
});
it.each([ [ "2,5" ], [ "1.5" ], [ "0" ] ])("quantidade %s recusada antes", (q) => { /* withdrawalProblems com a frase de invalid_quantity */ });
it.each([
  [ { on: true, name: "Joana", document: "" }, "quem retira" ], [ { on: true, name: "", document: "RG12345" }, "quem retira" ],
  [ { on: true, name: "Joana", document: "RG 123" }, "documento" ], [ { on: false, name: "x", document: "" }, null ]
])("terceiro: os dois ou nenhum (%j)", (collector, hint) => { /* withdrawalProblems contém (ou não) a frase */ });
it("item fora da cota da inscrição é recusado antes", () => { /* OSTOMY_BAG numa inscrição de diabetes → item_not_in_program */ });
```

  - `supplyPrograms.test.ts`:

```ts
it.each([ [ "e11.9", true ], [ "E14", true ], [ "O24.4", true ], [ "E15", false ], [ "E1", false ], [ "Z93.3", false ] ])("CID do diabetes %s", (cid, okCid) => {
  expect(DIABETES_CID.test(normalizeCid(cid))).toBe(okCid);
});
it("incompleto fica fora do corpo e entra no que falta", () => {
  const d = { ...newProgramDraft("diabetes"), cid: "E11", insulin: "nph" as const, applications: "11", measurements: "3", unitId: "u1" };
  expect(supplyProgramsPayload([ d ], TODAY27S)).toEqual([]);
  expect(supplyProgramsMissing([ d ], TODAY27S)[0]).toMatch(/^Programas de insumos: /);
});
it("validade em meses; encerrar só com o enrollment_id", () => {
  const d = { ...newProgramDraft("diabetes"), cid: "E11.9", insulin: "none" as const, applications: "0", measurements: "1", unitId: "u1", months: "3" };
  expect(supplyProgramsPayload([ d ], "2026-11-30")).toEqual([ { action: "open", program: "diabetes", cid10: "E119", valid_until: "2027-02-28", preferred_unit_id: "u1",
    profile: { insulin_type: "none", applications_per_day: 0, measurements_per_day: 1 } } ]);
  const c = { ...newProgramDraft("diabetes", enrollmentDetail()), action: "close" as const };
  expect(supplyProgramsPayload([ c ], TODAY27S)).toEqual([ { action: "close", program: "diabetes", enrollment_id: "se1" } ]);
});
it("regras: fixed sem fator nem teto; aplicações só no diabetes", () => { /* ruleProblems por tabela */ });
```

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod27s src/lib/suppliesErrors.test.ts src/lib/supplies.test.ts src/lib/supplyPrograms.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/suppliesErrors.ts src/lib/suppliesErrors.test.ts src/lib/supplies.ts src/lib/supplies.test.ts src/lib/supplyPrograms.ts src/lib/supplyPrograms.test.ts
$G commit -m "feat: add the supplies phrases, gates, labels and screen rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Quatro itens no grupo Farmácia e o portão

**Files:**
- Create: `src/modules/supplies/SuppliesGate.tsx`, `src/modules/supplies/SuppliesGate.test.tsx`, `src/shell/modules.supplies.test.ts` e as cascas `src/modules/SupplyWithdrawal.tsx`, `src/modules/SupplyExternalEnrollment.tsx`, `src/modules/DeviceLoans.tsx`, `src/modules/SupplyEnrollmentImport.tsx` (portão e título; o corpo vem nas Tasks 7, 9, 10 e 12)
- Modify: `src/shell/modules.ts`, `src/shell/catalog.ts`, `src/App.tsx`

**Interfaces:**
- Consumes: `canUseSupplyPrograms`, `SUPPLY_PROGRAMS_DISABLED` (Task 2); `pharmacyRoles`, `PHARMACY_NO_ROLE`.
- Produces:
  - `ModuleId` + `"supply-withdrawal" | "supply-external-enrollment" | "device-loans" | "supply-enrollment-import"`.
  - No grupo **Farmácia**, depois de `pharmacy-external`: `{ id: "supply-withdrawal", label: "Retirada de insumos", icon: "✚" }`, `{ id: "supply-external-enrollment", label: "Inscrição com laudo externo", icon: "✎" }`, `{ id: "device-loans", label: "Aparelhos em comodato", icon: "⌚" }`; antes de `pharmacy-admin`: `{ id: "supply-enrollment-import", label: "Importar pacientes dos programas", icon: "⇲" }`.
  - Regras do menu (`programsOn = canUseSupplyPrograms(user)`):
    - `supply-withdrawal`, `device-loans` → `programsOn && (isPharmacist || isPharmacyAssistant)`;
    - `supply-external-enrollment` → `programsOn && isPharmacist`;
    - `supply-enrollment-import` → `programsOn && isAdmin`.
  - `CATALOG.screens` (descrições proposta): `supply-withdrawal` (`IconNeedle`) "Inscrições, saldo do período, entrega por lote e exceção"; `supply-external-enrollment` (`IconFileCertificate`) "Inscrição no programa a partir de laudo de fora, por até 3 meses"; `device-loans` (`IconDeviceWatch`) "Glicosímetros emprestados, devolução, troca e perda"; `supply-enrollment-import` (`IconFileImport`) "Planilha dos pacientes atuais, conferência e aplicação".
  - `SuppliesGate({ title, sub, allow, children })`, no molde do `PharmacyGate`: sem `pharmacy` → `PHARMACY_DISABLED`; sem `supply_programs` → `SUPPLY_PROGRAMS_DISABLED`; sem papel → `PHARMACY_NO_ROLE`.
  - `App.tsx`: `case "supply-withdrawal": return <SupplyWithdrawal />;`, `case "supply-external-enrollment": return <SupplyExternalEnrollment />;`, `case "device-loans": return <DeviceLoans />;`, `case "supply-enrollment-import": return <SupplyEnrollmentImport onGoToSecurity={() => setActive("security")} />;`.

- [ ] **Step 1: Testes que falham.**
  - `modules.supplies.test.ts`: farmacêutico com os interruptores vê três (sem a importação); auxiliar vê dois; `municipal_admin` vê a importação; sem `supply_programs`, nenhum dos quatro e os do 27a intactos; o operador não vê nenhum; o `CATALOG` tem as quatro descrições.
  - `SuppliesGate.test.tsx`: as três frases e o `children` com os papéis.
- [ ] **Step 2–4:** rode e veja falhar (`src/shell/modules.supplies.test.ts src/modules/supplies/SuppliesGate.test.tsx src/shell/catalog.test.ts src/shell/modules.pharmacy.test.ts`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.supplies.test.ts src/shell/catalog.ts src/App.tsx src/modules/supplies/SuppliesGate.tsx src/modules/supplies/SuppliesGate.test.tsx src/modules/SupplyWithdrawal.tsx src/modules/SupplyExternalEnrollment.tsx src/modules/DeviceLoans.tsx src/modules/SupplyEnrollmentImport.tsx
$G commit -m "feat: add the supply withdrawal, external enrollment, device loan and enrollment import screens to the pharmacy group" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Insumo no estoque (entradas, abertura, posição e mínimos)

**Files:**
- Modify: `src/lib/pharmacy.ts` (`itemKey`, `receiptLineProblem`, `receiptItems`), `src/modules/pharmacy/ItemSearch.tsx`, `src/modules/pharmacy/ReceiptForm.tsx`, `src/modules/pharmacy/MinimumsPanel.tsx`, `src/lib/pharmacy.test.ts`; com o 27b na `main`, também `src/lib/openingImport.ts` e `src/lib/openingImport.test.ts`
- Create: `src/modules/pharmacy/ReceiptForm.supply.test.tsx`, `src/modules/pharmacy/MinimumsPanel.supply.test.tsx`

**Interfaces:**
- Consumes: `searchPharmacyItems(q, "supply")`, `StockItem.supply_catalog_item_id`, `StockItem.group`, `ReceiptItemInput` (ramo do insumo), `putMinimums` (Task 1); `canUseSupplies`, `groupLabel` (Task 2).
- Produces:
  - `itemKey`: `sup:<supply_catalog_item_id>` antes dos outros ramos sem `catalog_item_id`.
  - `receiptItems`: a linha com `item.supply_catalog_item_id` vai `{ supply_catalog_item_id, lot_number, expires_on, manufacturer?, quantity, package? }`, sem `catalog_item_id` (as duas chaves o api recusa com `invalid_request` `items[i]`).
  - `receiptLineProblem`: sem item e com o tipo "Insumo" escolhido → "escolha o material do catálogo" (proposta).
  - `ItemSearch` ganha `kind?: "vector_control" | "supply"` no lugar de `domain` (o 35a passa a `kind="vector_control"`, mesmo corpo). Com `kind="supply"`: o vazio é "nenhum material encontrado" (proposta); o botão mostra `"<nome> · <unidade de fornecimento>"` e o `groupLabel` ao lado (`Tag tone="info"`). Sem `kind`, nada muda.
  - `ReceiptForm`, fora do local `vector_control` e com `canUseSupplies(user)`: um `SegmentedControl` "Medicamento" | "Insumo" (proposta) acima da busca. "Insumo" troca a busca por `ItemSearch kind="supply" label="Buscar material"`. As duas espécies convivem na mesma nota (Q9: o insumo é um domínio próprio de busca, separado da receita).
  - `MinimumsPanel`, com `canUseSupplies(user)`: um segundo `ItemSearch kind="supply" label="Acrescentar material"`; as linhas de insumo vão com `supply_catalog_item_id`.
  - Transferências, posição e inventário não mudam de código: leem a posição, que passa a trazer o insumo. O teste confere que um lote de insumo aparece na posição com o rótulo do grupo.
  - Com o 27b na `main` (Task 0, `27b-ok`):
    - a linha de insumo recebe os `BnafarLotFields` como o medicamento (domínio `medication`), sem AMPP;
    - a frase de `invalid_product_type` da planilha do saldo inicial passa a citar `insumo` (Q12): "tipo_produto deve ser medicamento, vacina, vetorial ou insumo" (proposta).

- [ ] **Step 1: Testes que falham.**
  - `pharmacy.test.ts`: `itemKey(STRIP)` = `"sup:sc1"`; `receiptItems` com uma linha de tira por embalagem → `{ supply_catalog_item_id: "sc1", …, package: { units_per_package: 50, packages: 4 }, quantity: "200" }`, sem `catalog_item_id`.
  - `ReceiptForm.supply.test.tsx`, o caso do Review Focus 4, "insumo só onde cabe; linha com a chave do insumo":
    - com `pharmacy_supplies`, "Insumo" busca com `searchPharmacyItems(…, "supply")`; nota com um medicamento e uma tira → corpo com uma linha de cada chave;
    - sem `pharmacy_supplies`, nenhum seletor;
    - no local `vector_control`, nenhum seletor, e a busca de campo segue com `"vector_control"`;
    - `DispenseForm` e `ExternalPrescriptionForm`, montados com a mesma sessão, chamam `searchPharmacyItems` sem o segundo argumento (um caso cada, pelo `ItemSearch` deles).
  - `MinimumsPanel.supply.test.tsx`: acrescentar uma tira manda `putMinimums("l2", [ …, { supply_catalog_item_id: "sc1", minimum: "500" } ])`.
  - Com o 27b: em `openingImport.test.ts`, a frase nova de `invalid_product_type`.
- [ ] **Step 2–4:** rode e veja falhar (os arquivos e `src/modules/PharmacyEntries.test.tsx src/modules/PharmacyStock.test.tsx src/modules/pharmacy/DispenseForm.test.tsx src/modules/PharmacyExternal.test.tsx src/modules/pharmacy/ReceiptForm.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/pharmacy.ts src/lib/pharmacy.test.ts src/modules/pharmacy/ItemSearch.tsx src/modules/pharmacy/ReceiptForm.tsx src/modules/pharmacy/MinimumsPanel.tsx src/modules/pharmacy/ReceiptForm.supply.test.tsx src/modules/pharmacy/MinimumsPanel.supply.test.tsx
$G commit -m "feat: receive, open and set minimums for supplies in the pharmacy stock screens" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Com o 27b, inclua `src/lib/openingImport.ts` e `src/lib/openingImport.test.ts`, e os testes do 35a que o `kind` tocou.)

---

### Task 5: "Programas de insumos" no rascunho da consulta

**Files:**
- Create: `src/modules/consultation/SupplyProgramsField.tsx`, `src/modules/supplies/ProfileFields.tsx`, `src/modules/consultation/ConsultationEditor.supplies.test.tsx`
- Modify: `src/lib/consultation.ts` (`ConsultationDraft`, `draftFrom`, `checkDraft`), `src/modules/consultation/ConsultationEditor.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx` (passa as props)

**Interfaces:**
- Consumes: `SupplyProgramDraft`, `MAX_PROGRAM_LINES`, `draftsFrom`, `newProgramDraft`, `supplyProgramsPayload`, `supplyProgramsMissing`, `supplyProgramsFinalizeError`, `canEnrollSupplies`, `quotaLine`, `programLabel`, `PATIENT_SUPPLY_ENROLLMENTS_KEY` (Task 2); `listPatientSupplyEnrollments`, `getSupplyEnrollment`, `Consultation.supply_programs` (Task 1); `props.units`.
- Produces:
  - `ConsultationDraft.supplyPrograms: SupplyProgramDraft[]`; `draftFrom` lê `draftsFrom(c.supply_programs, today)`; `checkDraft` recebe `today` opcional e põe `supply_programs: supplyProgramsPayload(…)` no `input` **só quando a lista não é vazia**. A consulta sem a seção manda o corpo de hoje, e o `initialKey` não muda.
  - `ConsultationEditorProps.supplyPrograms?: { patientId: string }`. A Workspace passa `canEnrollSupplies(user, consultation.cbo_code) && consultation.kind !== "dental" && !props.urgency`.
  - `SupplyProgramsField({ patientId, value, onChange, units, quotasFromApi, today })`, uma seção `aria-label="Programas de insumos"` com o título "Programas de insumos":
    - lê `listPatientSupplyEnrollments(patientId)` (`NOMINAL_QUERY`, chave `PATIENT_SUPPLY_ENROLLMENTS_KEY`, a mesma do cartão da Task 6);
    - por programa (`diabetes`, `ostomy`):
      - sem inscrição ativa ou inativa → "Inscrever em <programa>" (proposta);
      - com ativa → "Renovar" e "Encerrar";
      - com inativa → "Reativar" (= `renew`, proposta);
      - "Renovar" e "Reativar" leem `getSupplyEnrollment(id)` uma vez (o CID e o perfil só vêm no detalhe, Q7) e pré-preenchem a linha;
      - até `MAX_PROGRAM_LINES` linhas;
    - a linha `open`/`renew` mostra:
      - `ProfileFields`: no diabetes, o CID com a dica "E10 a E14 ou O24", o tipo de insulina, as aplicações e as medições por dia; na estomia, o CID e o tipo de estoma;
      - a unidade de retirada preferida (`units`);
      - a validade em meses (3–12);
      - "Fixar cota acima da regra" (proposta), que abre a quantidade por grupo e a justificativa;
    - a linha `close` diz "a inscrição será encerrada ao finalizar" (proposta) e tem "Desfazer";
    - a cota calculada é `quotasFromApi[key]` (as `quotas` de cada linha na resposta do último PATCH, Q7), mostrada em `quotaLine`:
      - lista vazia → "sem regra de cota da cidade para este perfil — a entrega fica bloqueada até a cidade preencher as regras ou você fixar a cota" (proposta);
      - antes do primeiro salvamento → "a cota aparece quando o rascunho salvar" (proposta);
    - o problema de cada linha aparece nela (`programDraftProblems`).
  - `ProfileFields({ program, value, onChange, idPrefix })`: os campos do perfil, reaproveitados na Task 9.
  - `ConsultationEditor`:
    - o `save` do autosave guarda `c.supply_programs` da resposta em estado (`quotasFromApi`, pela posição na lista enviada);
    - `missing` ganha `supplyProgramsMissing(…)`;
    - na recusa da finalização, `supplyProgramsFinalizeError(err)` vem antes de `consultationError`. Com `enrollment_exists` ou `enrollment_not_active`, invalida `PATIENT_SUPPLY_ENROLLMENTS_KEY` uma vez e não refinaliza. A consulta segue rascunho (o api desfaz tudo).
    - Depois de finalizar, a Workspace invalida `PATIENT_SUPPLY_ENROLLMENTS_KEY`: a lista é por paciente e pode ser relida, ao contrário do prontuário em contexto.

- [ ] **Step 1: Testes que falham** — `ConsultationEditor.supplies.test.tsx` (molde `ConsultationEditor.test.tsx`; `vi.mock` do api com `fetchCurrentSession`; `autosaveDelayMs={0}`):

| Caso | Asserções |
|---|---|
| "seção só para médico e enfermeiro, só na consulta comum" | CBO `225142` e `223505` → a seção; `223208` (dentista), `dental` e `urgency` → nenhuma |
| "inscrever: PATCH com o bloco, cota do api à vista" | preenche diabetes NPH com 4 medições → PATCH com `supply_programs: [ { action: "open", program: "diabetes", cid10: "E119", … } ]`; a resposta com `quotas` mostra "120 · tira de glicemia a cada 30 dias" |
| "incompleto fica fora do PATCH e trava a finalização; inscrição aberta ao mesmo tempo relê" | aplicações `"11"` → o PATCH não tem `supply_programs`; "Finalizar" lista "Programas de insumos: …"; corrigido, finaliza; `409 enrollment_exists` → a frase, `listPatientSupplyEnrollments` relido uma vez, nenhuma segunda finalização |
| "renovar e encerrar a ativa; reativar a inativa" | com `enrollmentSummary()` ativa, "Renovar" lê `getSupplyEnrollment("se1")`, pré-preenche CID e perfil e manda `{ action: "renew", enrollment_id: "se1", … }`; "Encerrar" manda `{ action: "close", program: "diabetes", enrollment_id: "se1" }`; inativa → "Reativar" |
| "cota acima da regra exige justificativa" | marcada sem justificativa → a linha não vai; com 10+ caracteres → `quota_overrides: [ { group: "glucose_strip", quantity: 200 } ]` e `override_reason` |
| "rascunho salvo volta igual" | `consultation({ supply_programs: [ … ] })` abre a linha preenchida e não dispara PATCH ao abrir |
| "sem a seção, o corpo é o de hoje" | sem `supplyPrograms`, o PATCH não tem a chave `supply_programs` |

- [ ] **Step 2–4:** rode e veja falhar (`src/modules/consultation/ConsultationEditor.supplies.test.tsx src/modules/consultation/ConsultationEditor.test.tsx src/modules/consultation/ConsultationWorkspace.dental.test.tsx src/lib/consultation.test.ts`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/consultation.ts src/modules/consultation/SupplyProgramsField.tsx src/modules/supplies/ProfileFields.tsx src/modules/consultation/ConsultationEditor.tsx src/modules/consultation/ConsultationWorkspace.tsx src/modules/consultation/ConsultationEditor.supplies.test.tsx
$G commit -m "feat: enroll, renew and close supply programs in the consultation draft with the quota from the API" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Inscrições no painel do paciente e na linha do tempo

**Files:**
- Create: `src/modules/consultation/SupplyEnrollmentsCard.tsx`, `src/modules/consultation/ConsultationWorkspace.supplies.test.tsx`, `src/lib/timeline.supplies.test.ts`
- Modify: `src/modules/consultation/PatientPanel.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/lib/timeline.ts`, `src/modules/timeline/PatientTimeline.tsx`

**Interfaces:**
- Consumes: `listPatientSupplyEnrollments`, `TimelineKind` (Task 1); `programLabel`, `enrollmentStatusLabel`, `quotaLine`, `canUseSupplyPrograms`, `PATIENT_SUPPLY_ENROLLMENTS_KEY` (Task 2).
- Produces:
  - `SupplyEnrollmentsCard({ patientId })`: "Programas de insumos" no Resumo, depois dos problemas. Lê `listPatientSupplyEnrollments` (`NOMINAL_QUERY`), sem CID nem perfil (Q7). Por inscrição:
    - o programa e a situação (`Tag`);
    - "válida até <data>" e a unidade preferida;
    - a cota, por `quotaLine`.

    Sem inscrição, o cartão não aparece. Com `403`, também não: falta o acesso ao prontuário ou o interruptor, e o resto do painel segue.
  - `PatientPanel` ganha a prop `supplies?: { patientId: string }`; a Workspace passa o paciente com `canUseSupplyPrograms(user)`.
  - `timeline.ts`:
    - `TIMELINE_KINDS` + `{ key: "supply_enrollment", label: "Programa de insumos" }` (proposta);
    - `timelineKinds(…, supplies = false)` (último parâmetro), e `PatientTimeline` passa `canUseSupplyPrograms(user)`;
    - o título e o resumo vêm prontos do api ("Inscrição em programa de insumos", "Renovação…", "Encerramento…"; plano do api, Task 17), e a inativação não entra;
    - `STATUS_LABEL` + `opened` "aberta" e `renewed` "renovada" (`closed` já é "encerrado"), para o caso de o api mandar o tipo do evento no `status` (a Task 0 confere).

- [ ] **Step 1: Testes que falham.**
  - `ConsultationWorkspace.supplies.test.tsx`:
    - com `listPatientSupplyEnrollments` → `[ enrollmentSummary() ]`, o cartão mostra "Diabetes", "ativa" e a cota, sem CID;
    - sem `supply_programs` na sessão, não lê nem aparece;
    - `403` → nenhum cartão, e o resto do painel segue;
    - depois de finalizar, a lista é relida uma vez.
  - `timeline.supplies.test.ts`:
    - o filtro só aparece com `supplies`;
    - `timelineKindLabel("supply_enrollment")`;
    - os dois rótulos de situação;
    - um status futuro volta cru.
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos e `src/lib/timeline.test.ts src/modules/consultation/ConsultationWorkspace.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/consultation/SupplyEnrollmentsCard.tsx src/modules/consultation/PatientPanel.tsx src/modules/consultation/ConsultationWorkspace.tsx src/lib/timeline.ts src/modules/timeline/PatientTimeline.tsx src/modules/consultation/ConsultationWorkspace.supplies.test.tsx src/lib/timeline.supplies.test.ts
$G commit -m "feat: show the supply program enrollments in the patient panel and the timeline" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: "Retirada de insumos"

**Files:**
- Create: `src/modules/supplies/EnrollmentCard.tsx`, `src/modules/supplies/SupplyDispenseForm.tsx`, `src/modules/SupplyWithdrawal.test.tsx`
- Modify: `src/modules/SupplyWithdrawal.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `listSupplyEnrollments`, `recordSupplyDispensation`, `listLocationPharmacists` e `listPositions` (com `supplyCatalogItemId`);
  - da Task 2: `withdrawalProblems`, `withdrawalBody`, `overQuotaGroups`, `quotaOf`, `suppliesRefusals`, os rótulos e as chaves;
  - da Task 3: `SuppliesGate`;
  - do 27a: `LocationPick`, `PersonStart` + `PHARMACY_PERSON_SOURCE`, `ItemSearch kind="supply"`, `availableLots`, `suggestedLotId`, `lotOptionLabel`, `locationRoles`, `PatientHistory`.
- Produces:
  - `SupplyWithdrawal({ today = todayInCity() })`: `SuppliesGate title="Retirada de insumos" sub="inscrição, saldo do período, lote e retirada" allow={(r) => r.pharmacist || r.assistant}` → `LocationPick` (farmacêutico ou auxiliar) → corpo:
    - `PersonStart` com a fonte da farmácia. Pessoa sem paciente → "esta pessoa não tem inscrição em programa de insumos — a inscrição nasce na consulta ou pelo laudo externo" (proposta);
    - com o paciente, lê `listSupplyEnrollments(patientId)` (`NOMINAL_QUERY`, `gcTime: 0`);
    - nenhuma inscrição → a frase acima. Inscrição `inactive` ou `expired` → o cartão sem "Entregar", com a frase de `enrollment_not_active` ou de `enrollment_expired`;
    - para cada `devices[]` da inscrição: "Aparelho em comodato: <modelo> · nº <série> — tira compatível: <nome>" (proposta). Sem `compatible_item`, a linha não tem a parte da tira;
    - "Histórico de retiradas" abre o `PatientHistory` (Task 8).
  - `EnrollmentCard({ enrollment, onDeliver })`:
    - programa, situação, validade e unidade preferida, com a nota "o paciente retira em qualquer unidade; a cota é somada entre elas" (proposta);
    - por grupo, "restam <available> de <quantity> · período até <period_end>". Sem ciclo vigente, "o período começa na próxima retirada" (proposta).
  - `SupplyDispenseForm({ location, enrollment, today, onDone, onStale, onCancel })`, `aria-label="Registrar entrega de insumos"`:
    - linhas, até 20 (`ItemSearch kind="supply"`): o material; os lotes dele no local (`listPositions` com `supplyCatalogItemId`, lote sugerido FEFO); a quantidade inteira, na unidade de fornecimento. Um botão "Acrescentar material";
    - "Acima da cota" (proposta) aparece quando `overQuotaGroups` não é vazio:
      - sempre a justificativa;
      - para o auxiliar (`!locationRoles(location).pharmacist`), também o `select` "Farmacêutico que libera" (`listLocationPharmacists`, `STATIC_QUERY`) e o campo "Código do autenticador do farmacêutico" (`inputMode="numeric"`, `autoComplete="one-time-code"`), limpo no envio;
    - "Retirada por outra pessoa" (proposta): nome e documento;
    - "Registrar entrega" (proposta) roda `withdrawalProblems`. Com problema, a lista `role="alert"` e nada sai; sem problema, `recordSupplyDispensation(withdrawalBody(…))`.
  - Recusas:
    - `quota_exceeded` (`path: items[i].quantity`): a frase, a exceção aberta e uma releitura das inscrições;
    - `401 override_code_invalid`: a frase no campo do código. Não é sessão vencida: nada de `auth.reload`;
    - `403 override_requires_pharmacist`: a frase no `select`;
    - `insufficient_balance` ou `lot_expired`: relê os lotes;
    - `enrollment_not_active` ou `enrollment_expired`: `onStale`, que fecha o formulário, relê as inscrições e deixa a frase;
    - `not_ostomy_service`, `patient_without_cns`, `sigtap_max_quantity` e `supply_without_sigtap`: a frase, sem reler;
    - `item_not_in_program` e `invalid_quantity`: na linha pelo `items[i]`.
  - Sucesso: "Entrega registrada." (proposta) e uma releitura das inscrições e das posições.

- [ ] **Step 1: Testes que falham** — `SupplyWithdrawal.test.tsx`. Sessão `supplyUser([ "pharmacy_assistant" ])` no local `UBS_FARM` com `roles: [ "pharmacy_assistant" ]`, e farmacêutico onde dito:

| Caso | Asserções |
|---|---|
| "inscrição e saldo à vista; tira compatível do aparelho" | o cartão com "restam 40 de 90"; a linha do aparelho com a tira |
| "entrega dentro da cota" | tira 30 → corpo `{ location_id: "l2", enrollment_id: "se1", items: [ { supply_catalog_item_id: "sc1", lot_id, quantity: "30" } ] }`, sem `override` nem `collector`; "Entrega registrada."; inscrições relidas uma vez |
| "exceção do auxiliar: farmacêutico e código, um pedido, código limpo, recusa no campo" | tira 50 → "Acima da cota" pede justificativa, farmacêutico e código; sem eles nada sai. Preenchidos, duplo clique → um `recordSupplyDispensation` com `override: { reason, pharmacist_user_id: "u-ph1", code: "123456" }`; o campo do código fica vazio. `401 override_code_invalid` → a frase no campo, nenhum segundo pedido, nenhum `auth.reload` |
| "exceção do farmacêutico: só a justificativa" | sessão farmacêutico → sem `select` nem código; corpo com `override: { reason }` |
| "corrida de saldo: quota_exceeded abre a exceção e relê" | `422 quota_exceeded` com `path: "items[0].quantity"` → a frase na linha; uma releitura das inscrições |
| "terceiro: os dois ou nenhum" | marca com só o nome → recusa antes; nome e documento com espaços nas pontas → corpo aparado |
| "quantidade fracionária recusada antes" | `"2,5"` → a frase de `invalid_quantity`, nenhuma chamada |
| "inscrição inativa não entrega" | `status: "inactive"` → sem "Entregar", a frase |
| "bolsa fora do ponto de estomia" | `422 not_ostomy_service` → a frase; nenhuma releitura |
| "código nunca no cache" | depois do envio, nenhuma `queryKey` do `QueryClient` contém o código nem o documento |

- [ ] **Step 2–4:** rode e veja falhar (`src/modules/SupplyWithdrawal.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/SupplyWithdrawal.tsx src/modules/supplies/EnrollmentCard.tsx src/modules/supplies/SupplyDispenseForm.tsx src/modules/SupplyWithdrawal.test.tsx
$G commit -m "feat: deliver program supplies by lot within the quota, with the pharmacist exception and third-party pickup" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Histórico e devolução do insumo

**Files:**
- Modify: `src/modules/pharmacy/PatientHistory.tsx`, `src/modules/pharmacy/ReturnForm.tsx`
- Create: `src/modules/pharmacy/PatientHistory.supply.test.tsx`

**Interfaces:**
- Consumes: `Dispensation.supply_enrollment`, `Dispensation.collector`, `DispensedItem.supply_catalog_item_id`, `StockItem.group` (Task 1); `groupLabel`, `programLabel`, `SUPPLY_ENROLLMENTS_KEY`, `suppliesError` (Task 2).
- Produces:
  - `PatientHistory`:
    - a dispensação por inscrição mostra "programa <Diabetes>" (proposta) no lugar da receita;
    - com `collector`, mostra "retirada por <nome>" (proposta). O api manda só o nome, nunca o documento;
    - "Registrar devolução" **não** aparece no item de `group` `ostomy_bag` (decisão 17);
    - depois da devolução de um insumo, invalida `SUPPLY_ENROLLMENTS_KEY` (o saldo voltou à cota), além das chaves de hoje.
  - `ReturnForm`: `ostomy_return_not_allowed` → a frase de `SUPPLIES_ERRORS`.

- [ ] **Step 1: Testes que falham** — `PatientHistory.supply.test.tsx`, o caso do Review Focus 3, "bolsa sem devolução; tira devolve e relê o saldo":
  - a bolsa sem botão; a tira com botão;
  - devolver 10 tiras → `recordReturn` com o `dispensation_item_id`, e `SUPPLY_ENROLLMENTS_KEY` invalidada;
  - `422 ostomy_return_not_allowed` → a frase;
  - a dispensação por receita do 27a continua igual (um caso com `dispensation()`).
- [ ] **Step 2–4:** rode e veja falhar (o arquivo e `src/modules/pharmacy/PatientHistory.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/pharmacy/PatientHistory.tsx src/modules/pharmacy/ReturnForm.tsx src/modules/pharmacy/PatientHistory.supply.test.tsx
$G commit -m "feat: show program supply withdrawals in the patient history and block ostomy bag returns" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: "Inscrição com laudo externo"

**Files:**
- Create: `src/modules/supplies/ExternalEnrollmentForm.tsx`, `src/modules/SupplyExternalEnrollment.test.tsx`
- Modify: `src/modules/SupplyExternalEnrollment.tsx`

**Interfaces:**
- Consumes: `createExternalSupplyEnrollment` (Task 1); `ProfileFields` (Task 5); `normalizeCid`, `DIABETES_CID`, `OSTOMY_CID`, `suppliesError` (Task 2); `PersonStart`, `PHARMACY_PERSON_SOURCE`, `UFS`, `issuedOnProblem`, `listActiveUnits`.
- Produces:
  - `SupplyExternalEnrollment({ today })`: `SuppliesGate title="Inscrição com laudo externo" sub="laudo de fora da rede, válido por até 3 meses" allow={(r) => r.pharmacist}` → `LocationPick` (farmacêutico) → passos: a pessoa (`PersonStart`) e o formulário.
    - Só o paciente que já existe segue. O corpo do api leva `patient_id` (api Task 10), e o `PersonStart` da farmácia já cria o paciente do par validado, como no 27a.
    - A pessoa nova ou o cidadão sem paciente veem a frase "cadastre o paciente pela busca de CPF antes de inscrever" (proposta).
  - `ExternalEnrollmentForm({ patientId, location, today, onCreated, onChangePerson })`:
    - o programa (`SegmentedControl` Diabetes | Estomia), `ProfileFields`, a unidade de retirada preferida e a validade em meses (1, 2 ou 3; padrão 3; depois de hoje);
    - "Laudo": o profissional (3–120), o conselho (`CRM` | `COREN`), a UF, o número e a data do laudo (`issued_on`, não futura);
    - "Registrar inscrição" (proposta) chama `createExternalSupplyEnrollment`.
  - Criada: "Inscrição registrada. A cota vale até <data>; a renovação é na consulta." (proposta). Se o paciente tinha uma inscrição inativa do programa, o api a reativa, e a frase é a mesma.
  - Recusas:
    - `409 enrollment_exists` → a frase;
    - `invalid_validity`, `invalid_cid10` e `cid_not_in_program` → no campo, pelo `path`;
    - `external_report.<campo>` → no campo do laudo.

- [ ] **Step 1: Testes que falham** — `SupplyExternalEnrollment.test.tsx`:
  - só o farmacêutico (o auxiliar vê `PHARMACY_NO_ROLE`);
  - estomia com colostomia e laudo de `COREN` → corpo `{ location_id, patient_id, program: "ostomy", profile: { stoma_type: "colostomy" }, valid_until: <hoje + 3 meses>, external_report: { professional_name, council: "COREN", council_state, council_number, issued_on } }`; nada disso na URL;
  - a validade só oferece de 1 a 3 meses, e o conselho só `CRM` e `COREN`;
  - laudo com data futura é recusado antes;
  - pessoa nova não chega ao formulário;
  - `409 enrollment_exists` mostra a frase.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/SupplyExternalEnrollment.tsx src/modules/supplies/ExternalEnrollmentForm.tsx src/modules/SupplyExternalEnrollment.test.tsx
$G commit -m "feat: enroll a patient in a supply program from an external report at the pharmacy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: "Aparelhos em comodato"

**Files:**
- Create: `src/modules/supplies/DeviceLoanForm.tsx`, `src/modules/DeviceLoans.test.tsx`
- Modify: `src/modules/DeviceLoans.tsx`

**Interfaces:**
- Consumes: `listDeviceLoans`, `createDeviceLoan`, `returnDeviceLoan`, `replaceDeviceLoan`, `reportDeviceLoanLost`, `listDeviceModels` (Task 1); `loanStatusLabel`, `deviceKindLabel`, `serialProblem`, `policeReportProblem`, `suppliesError`, `DEVICE_LOANS_KEY` (Task 2); `LocationPick`, `PersonStart`, `ConfirmButton`, `DataTable`.
- Produces:
  - `DeviceLoans()`: `SuppliesGate title="Aparelhos em comodato" sub="glicosímetro e leitor emprestados, por unidade" allow={(r) => r.pharmacist || r.assistant}` → `LocationPick` → só local com `health_unit`. No central, a tela diz "o comodato é registrado na farmácia da unidade" (proposta).
  - A lista da unidade: `listDeviceLoans({ healthUnitId, status })`, `NOMINAL_QUERY`, filtro "emprestados" | "todos" e "Carregar mais". As colunas: modelo, nº de série, desde e situação. A lista do api não traz o nome do paciente nem o boletim, e a tela não os pede.
  - "Emprestar aparelho" (proposta) abre o `DeviceLoanForm`:
    - a pessoa (`PersonStart`, só paciente), o modelo ativo e o número de série;
    - a marca obrigatória "Termo de comodato assinado pelo paciente" (proposta);
    - envia `createDeviceLoan({ location_id, patient_id, device_model_id, serial_number, term_accepted: true })`.
  - Por empréstimo ativo:
    - "Devolver" (`ConfirmButton`) → `returnDeviceLoan(id, version)`;
    - "Trocar" (modelo e série novos) → `replaceDeviceLoan`;
    - "Registrar perda" (número do boletim) → `reportDeviceLoanLost`.
  - Recusas: `409 device_loan_changed` → a frase e uma releitura; `409 serial_in_use` → no campo da série; `term_required` → na marca.

- [ ] **Step 1: Testes que falham** — `DeviceLoans.test.tsx`:
  - sem a marca do termo, nada sai; com ela, o corpo leva `term_accepted: true` e o `location_id` do local;
  - trocar manda `expected_version`, o modelo e a série novos, e relê a lista;
  - a perda sem boletim é recusada antes; com ele, o número vai só no corpo;
  - `409 device_loan_changed` → a frase e uma releitura, sem reenvio;
  - o almoxarifado central não oferece o empréstimo.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/DeviceLoans.tsx src/modules/supplies/DeviceLoanForm.tsx src/modules/DeviceLoans.test.tsx
$G commit -m "feat: lend, return, replace and report lost glucose meters on loan by unit" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: "Regras de cota" e modelos de aparelho em "Locais e configurações"

**Files:**
- Create: `src/modules/pharmacy/QuotaRulesPanel.tsx`, `src/modules/pharmacy/DeviceModelsPanel.tsx`, `src/modules/PharmacyAdmin.supplies.test.tsx`
- Modify: `src/modules/PharmacyAdmin.tsx`

**Interfaces:**
- Consumes: `getSupplyQuotaRules`, `putSupplyQuotaRules`, `listAdminDeviceModels`, `createDeviceModel`, `updateDeviceModel` (Task 1); `RuleDraft`, `EMPTY_RULE`, `ruleProblems`, `rulesBody`, os rótulos, `QUOTA_RULES_KEY`, `DEVICE_MODELS_KEY`, `suppliesError` (Task 2); `SensitiveAction`, `SegmentedControl`, `ItemSearch kind="supply"`.
- Produces:
  - `PharmacyAdmin`: com `canUseSupplyPrograms(user)`, os dois painéis depois do `SettingsPanel`.
  - `QuotaRulesPanel({ onGoToSecurity })`, `Panel title="Regras de cota"`:
    - o programa no `SegmentedControl` Diabetes | Estomia, que dispara `getSupplyQuotaRules(program)` (Q8);
    - a nota (proposta): "Não há padrão nacional: a cidade preenche pelo protocolo dela. A primeira regra que casa com o perfil vale. A cota fica gravada na inscrição e só muda na renovação.";
    - lista vazia: "nenhuma regra — sem regra, a inscrição nasce sem cota e a entrega fica bloqueada" (proposta);
    - cada regra mostra:
      - "vale para": os tipos de insulina ou de estoma marcados; nenhum marcado vale para todos;
      - o grupo do programa;
      - o `kind`: "quantidade fixa" ou "pela prescrição" (propostas);
      - a quantidade, ou o fator ("medições por dia" | "aplicações por dia") com o teto;
      - o período em dias;
      - "subir" e "descer";
    - "Salvar regras" (proposta) abre o `SensitiveAction requiresStepUp` e chama `putSupplyQuotaRules(program, rulesBody(list), version)`. Com problema, nada abre. A recusa com `path: rules[i].campo` vai à linha.
  - `DeviceModelsPanel({ onGoToSecurity })`, `Panel title="Modelos de aparelho"` (proposta, Q12):
    - os campos: nome (3–120), tipo, fornecedor (até 120), tira compatível e ativo. A tira vem do `ItemSearch kind="supply"`, que só aceita `glucose_strip`;
    - criar e editar pelo `SensitiveAction requiresStepUp`; a edição manda `expected_version`.
  - `409 supply_quota_rules_changed` e `409 device_model_changed` → a frase e uma releitura, com o digitado mantido.

- [ ] **Step 1: Testes que falham** — `PharmacyAdmin.supplies.test.tsx`:
  - sem `supply_programs`, nenhum dos painéis;
  - uma regra `per_prescription` com "medições por dia", 30 dias e teto 150 → step-up → corpo `{ program: "diabetes", expected_version: 3, rules: [ { group: "glucose_strip", profile_match: { insulin_type: [ "nph", "regular" ] }, kind: "per_prescription", quantity: null, factor_field: "measurements_per_day", period_days: 30, cap: 150 } ] }`;
  - "subir" troca a ordem no corpo;
  - `fixed` sem quantidade, ou com teto, é recusado antes, sem step-up;
  - `409` relê uma vez e mantém o digitado;
  - o modelo só aceita uma tira de `glucose_strip`, e a edição manda `expected_version`.
- [ ] **Step 2–4:** rode e veja falhar (o arquivo e `src/modules/PharmacyAdmin.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/PharmacyAdmin.tsx src/modules/pharmacy/QuotaRulesPanel.tsx src/modules/pharmacy/DeviceModelsPanel.tsx src/modules/PharmacyAdmin.supplies.test.tsx
$G commit -m "feat: configure the city supply quota rules and the loaned device models" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: "Importar pacientes dos programas"

**Files:**
- Create: `src/lib/supplyImport.ts`, `src/lib/supplyImport.test.ts`, `src/modules/supplies/ImportReport.tsx`, `src/modules/SupplyEnrollmentImport.test.tsx`
- Modify: `src/modules/SupplyEnrollmentImport.tsx`

**Interfaces:**
- Consumes: as funções da importação (Task 1); `SuppliesGate`, `suppliesError`, `SUPPLY_IMPORT_KEY` (Tasks 2–3); `SensitiveAction`, `ConfirmButton`, `saveFile`, `DataTable`. Molde: a Task 11 do plano do dashboard do 27b, com os mesmos passos e sem "Desfazer" (Q12).
- Produces (`src/lib/supplyImport.ts`, Q12; plano do api, Task 16):
  - `SUPPLY_IMPORT_HEADER = "programa;cpf;cns;cid10;tipo_insulina;aplicacoes_dia;medicoes_dia;tipo_estoma;validade;cnes_unidade"`, `SUPPLY_IMPORT_MAX_BYTES = 2 * 1024 * 1024` e `ROWS_PAGE = 100`;
  - `fileProblem(file)`, `headerProblem(firstLine)` (tira o BOM e o `\r`) e `readFirstLine(file)`, com as frases do 27b adaptadas ("o arquivo passa de 2 MB — divida a planilha");
  - `ROW_ERRORS` e `rowErrorText`, com o código desconhecido cru. As frases são proposta:

    | Código | Frase |
    |---|---|
    | `invalid_program` | "programa deve ser diabetes ou estomia" |
    | `invalid_cpf` | "CPF inválido" |
    | `invalid_cns` | "CNS inválido" |
    | `person_not_found` | "CPF/CNS sem cadastro e sem par validado — vai para o relatório" |
    | `citizen_not_verified` | "cadastro do cidadão ainda sem validação presencial" |
    | `invalid_cid10` | "CID-10 inválido" |
    | `cid_not_in_program` | "CID fora da lista do programa" |
    | `invalid_profile` | "perfil incompleto ou fora de 0 a 10 por dia" |
    | `invalid_validity` | "validade fora de hoje a 12 meses" |
    | `unit_unknown` | "CNES sem unidade nesta cidade" |
    | `enrollment_exists` | "o paciente já tem inscrição ativa neste programa" |
    | `duplicate_line` | "linha repetida (mesmo paciente e programa)" |
  - `importStatusView`: `uploaded` "recebida", `validated` "conferida", `applied` "aplicada", `discarded` "descartada" (proposta);
  - `appliedText(i)`: "Pacientes importados: <n> inscrições ativas. A cota de cada uma sai das regras da cidade; o paciente que faltava foi criado agora." (proposta); `discardedText`: "Importação descartada; o arquivo foi apagado.".
- Produces (telas):
  - `SupplyEnrollmentImport({ onGoToSecurity })`: `SuppliesGate title="Importar pacientes dos programas" sub="planilha dos pacientes atuais, conferência e aplicação" allow={(r) => r.admin}`.
    - A nota (proposta): "Cada linha vira uma inscrição ativa depois de conferida e aplicada. A conferência casa o CPF ou o CNS sem criar ninguém; o paciente que falta só é criado na aplicação.".
    - "Baixar o modelo".
    - O envio:
      - `<input type="file" accept=".csv,text/csv" aria-label="Planilha dos pacientes dos programas">`;
      - o arquivo é conferido antes, e nenhuma chamada sai se ele for recusado;
      - "enviar e conferir" (`ConfirmButton`);
      - depois do envio, o arquivo sai do estado.
    - O api não tem lista de importações: a tela abre a que acabou de enviar. Recarregar a página volta ao envio.
  - `ImportReport({ detail, onGoToSecurity, onChanged })`:
    - mostra os totais, os erros por código (`rowErrorText` e a contagem) e as linhas em `DataTable` (linha, situação e erros; **sem** CPF, CNS nem CID);
    - tem o filtro "só com erro" e "carregar mais" pelo `next_after`;
    - na situação `validated`: "Aplicar <válidas> linhas" (`SensitiveAction requiresStepUp`) e "Descartar" (`ConfirmButton`). Em `applied` e `discarded`, só leitura;
    - `409 supply_enrollment_import_changed` → a frase e uma releitura.

- [ ] **Step 1: Testes que falham.**
  - `supplyImport.test.ts`:
    - `fileProblem` por tabela: sem arquivo, `.xlsx`, 2 MB + 1 e `PACIENTES.CSV` com exatamente 2 MB;
    - `headerProblem`: o cabeçalho exato, com BOM e `\r`, com vírgula e com colunas trocadas;
    - frases sem `_`; código desconhecido volta cru.
  - `SupplyEnrollmentImport.test.tsx`:
    - "arquivo recusado antes: nenhuma chamada";
    - "envio com BOM: relatório aberto e nome fora do corpo";
    - "linhas em páginas: after na segunda leitura";
    - "aplicar: step-up, um pedido, versão na chamada": duplo clique → um `applySupplyEnrollmentImport("si1", 1)`; `401 mfa_required` volta ao step-up;
    - "descartar: só leitura depois";
    - "só o administrador": o farmacêutico vê `PHARMACY_NO_ROLE`;
    - "relatório sem CPF nem CNS": o texto da tabela não casa `/\d{11}|\d{15}/`.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/supplyImport.ts src/lib/supplyImport.test.ts src/modules/SupplyEnrollmentImport.tsx src/modules/supplies/ImportReport.tsx src/modules/SupplyEnrollmentImport.test.tsx
$G commit -m "feat: import, review and apply the current supply program patients from a spreadsheet" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Ponto de estomia na unidade

**Files:**
- Modify: `src/modules/attendance/UnitForm.tsx` (`UnitFormValue`, os campos), `src/modules/attendance/Units.tsx` (`capsPayload`, a linha da tabela para o formulário), `src/modules/Attendance.tsx` (a prop)
- Create: `src/modules/attendance/UnitForm.ostomy.test.tsx`

**Interfaces:**
- Consumes: `HealthUnit.ostomy_service`, `HealthUnit.ostomy_responsible`, `UnitCapsInput` (Task 1); `canMarkOstomyService`, `suppliesError` (Task 2); `listProfessionals` (`src/lib/api.ts:795`).
- Produces:
  - `Units` e `UnitForm` ganham `ostomyOn?: boolean` (o `Attendance` passa `canMarkOstomyService(user)`).
  - `UnitFormValue.ostomyService?: boolean` e `ostomyResponsibleId?: string` (lido de `ostomy_responsible?.id`).
  - Com `ostomyOn`, `kind !== "ubs"` e o CNES preenchido aparecem:
    - a marca "Ponto de atenção à pessoa estomizada (entrega e faturamento da bolsa)" (proposta);
    - marcada, o `select` "Responsável pelo serviço (o CBO dele vai no BPA-I)" (proposta). Ele lista os profissionais de `listProfessionals()` com vínculo ativo (`ended_at === null`) **nesta** unidade, em `"<nome> · CBO <cbo_code>"`.

    Sem o CNES, a frase "o ponto de estomia precisa do CNES da unidade" (proposta). Na unidade nova, a marca fica para depois de salvar: "salve a unidade antes de escolher o responsável" (proposta).
  - Trocar o tipo para `ubs` com a marca ligada manda `ostomy_service: false` (o api limpa o responsável).
  - `capsPayload` manda as duas chaves só quando mudaram.
  - Recusas: `ostomy_service_not_allowed_in_ubs`, `ostomy_service_requires_cnes` e `ostomy_responsible_not_linked` → a frase.

- [ ] **Step 1: Testes que falham** — `UnitForm.ostomy.test.tsx`:
  - sem `ostomyOn`, nenhum campo, e o corpo de hoje, sem as chaves;
  - policlínica com CNES, marca e responsável → corpo com `ostomy_service: true` e `ostomy_responsible_professional_id: "pr1"`; o `select` só lista quem tem vínculo ativo na unidade;
  - sem CNES, a frase e nenhuma marca;
  - UBS: nenhum campo; trocar para UBS manda `ostomy_service: false`;
  - `422 ostomy_responsible_not_linked` → a frase.
- [ ] **Step 2–4:** rode e veja falhar (o arquivo e `src/modules/attendance/Units.test.tsx src/modules/attendance/UnitForm.test.tsx src/modules/attendance/Units.caps.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/attendance/UnitForm.tsx src/modules/attendance/Units.tsx src/modules/Attendance.tsx src/modules/attendance/UnitForm.ostomy.test.tsx
$G commit -m "feat: mark a unit as an ostomy care point with its responsible professional" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Suíte, build, varreduras, revisão e prova no navegador (api na 3065, Vite na 5226)

**Files:** `$R/.claude/launch.json` (entrada nova; fora de git). Correção que a prova pedir entra na task dona, com teste.

- [ ] **Step 1: Suíte, tipos e build**

```bash
$R/bin/test dashboard -w dashboard-mod27s
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde e `build-ok`. A contagem é a base da Task 0 + **19 arquivos de teste novos**: `src/lib/api.supplies.test.ts`, `src/lib/suppliesErrors.test.ts`, `src/lib/supplies.test.ts`, `src/lib/supplyPrograms.test.ts`, `src/lib/supplyImport.test.ts`, `src/lib/timeline.supplies.test.ts`, `src/shell/modules.supplies.test.ts`, `src/modules/supplies/SuppliesGate.test.tsx`, `src/modules/pharmacy/ReceiptForm.supply.test.tsx`, `src/modules/pharmacy/MinimumsPanel.supply.test.tsx`, `src/modules/consultation/ConsultationEditor.supplies.test.tsx`, `src/modules/consultation/ConsultationWorkspace.supplies.test.tsx`, `src/modules/SupplyWithdrawal.test.tsx`, `src/modules/pharmacy/PatientHistory.supply.test.tsx`, `src/modules/SupplyExternalEnrollment.test.tsx`, `src/modules/DeviceLoans.test.tsx`, `src/modules/PharmacyAdmin.supplies.test.tsx`, `src/modules/SupplyEnrollmentImport.test.tsx`, `src/modules/attendance/UnitForm.ostomy.test.tsx`.

Contagem que dobra = artefato de build achado pelo vitest. Erro não tratado com tudo verde = mock sem `fetchCurrentSession`.

- [ ] **Step 2: Varreduras**

```bash
cd $W
O="src/modules/supplies src/modules/Supply*.tsx src/modules/DeviceLoans.tsx src/modules/consultation/SupplyProgramsField.tsx src/modules/consultation/SupplyEnrollmentsCard.tsx src/modules/pharmacy/QuotaRulesPanel.tsx src/modules/pharmacy/DeviceModelsPanel.tsx src/lib/supplies.ts src/lib/supplyPrograms.ts src/lib/supplyImport.ts src/lib/suppliesErrors.ts"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
grep -rnE "queryKey:.*(code|document|collector|cid|profile|reason|file)" $O && echo "ERRO: dado sensível no cache" || echo ok-cache
sed -n '/Insumos (módulo 27; contrato 2026-11-25/,$p' src/lib/api.ts | grep -nE "cid10|profile|collector|override|external_report|police_report|file_base64" | grep -E "withQuery|URLSearchParams|\?\$\{" && echo "revisar: URL" || echo ok-url
for f in src/modules/*.supplies.test.tsx src/modules/Supply*.test.tsx src/modules/DeviceLoans.test.tsx src/modules/pharmacy/*.supply.test.tsx src/modules/consultation/*.supplies.test.tsx src/modules/attendance/UnitForm.ostomy.test.tsx src/modules/supplies/*.test.tsx; do
  grep -q 'vi\.mock(".*lib/api", async' "$f" && ! grep -q "fetchCurrentSession" "$f" && echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
grep -rn '"supplies"' src/modules/consultation/SupplyProgramsField.tsx src/lib/supplyPrograms.ts && echo "ERRO: chave da odontologia (Q1)" || echo ok-q1
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-cache`, `ok-url`, `ok-mocks`, `ok-q1`, sem `ERRO`.

- [ ] **Step 3: Conferência do api.** Repita a Task 0 Step 4 contra o api final. Forma diferente vira ajuste na task dona antes da prova.

- [ ] **Step 4: Revisão do ramo** (superpowers:requesting-code-review). Foco: as cinco linhas do Review Focus; a tabela `SUPPLIES_ERRORS`; as frases fixas letra por letra; o bloco `supply_programs` (nunca `supplies`); o que fica fora do PATCH; o código do farmacêutico fora de cache e URL; nenhum CID ou perfil na lista da farmácia.

- [ ] **Step 5: Entrada do Vite.** Acrescente ao `$R/.claude/launch.json`, depois da última `dashboard-mod*`:

```json
{
  "name": "dashboard-mod27s",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/dashboard-mod27s && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3065 npx vite --port 5226 --host 0.0.0.0" ],
  "port": 5226
}
```

- [ ] **Step 6: Prova no navegador.** Antes de começar:
  - **Autorização do usuário:** migrar os bancos de dev (as migrações de cidade e de plataforma do 27, conferidas na Task 0 do api) e rodar a semente dos insumos em Curitiba só com autorização dele. Peça antes de qualquer `db:migrate` ou `db:seed`.
  - **Dono de cada serviço:** o api do worktree `apps/api/.claude/mod27s` na **3065** é da sessão do api; o maintenance na 5177 é da sessão dele (interruptores e carga do catálogo de materiais).
  - **Abrir:** `curitiba.localhost:5226/dashboard/` (`preview_start` com `dashboard-mod27s`).
  - **Senhas:** a senha e o TOTP de dev podem ser mostrados; os de staging, nunca.

  Roteiro, um passo por linha:

1. **Interruptores** (maintenance 5177): `pharmacy_supplies` e `supply_programs` ligados em Curitiba; o catálogo de materiais carregado. Antes de ligar, "Retirada de insumos" não aparece no menu.
2. **Configurações** (`municipal_admin`): "Regras de cota" (diabetes NPH e regular: tiras pela prescrição, medições × 30, teto 150; lancetas iguais; seringas pelas aplicações) com step-up; um modelo de glicosímetro com a tira compatível; a policlínica marcada como ponto de estomia, com o responsável.
3. **Estoque** (`stock_keeper` do central): nota com um medicamento e tiras por embalagem; bolsas; transferência das tiras à UBS e das bolsas à policlínica; a posição da UBS mostra a tira com o grupo; um mínimo de tiras.
4. **Consulta** (médico 225 numa UBS): inscrever em diabetes, NPH, 3 medições → a cota "90 · tira de glicemia a cada 30 dias" aparece depois do salvamento; uma linha incompleta trava "Finalizar"; finalizar; o painel do paciente relê e mostra a inscrição, sem CID; a linha do tempo mostra "Inscrição em programa de insumos". Numa segunda consulta, "Renovar" pré-preenche o CID e o perfil.
5. **Retirada** (auxiliar da UBS): 90 tiras entram; mais 10 → "Acima da cota", com justificativa, farmacêutico e código dele; retirada por terceiro com nome e documento. Na aba de rede, o código e o documento só aparecem no corpo do `POST`.
6. **Comodato**: emprestar o glicosímetro com o termo; a retirada mostra a tira compatível; trocar; registrar perda com boletim.
7. **Devolução**: 10 tiras voltam e o saldo sobe; a bolsa não tem "Registrar devolução".
8. **Estomia** (farmacêutico): inscrição com laudo externo (colostomia, 3 meses); bolsa na UBS → "não é ponto de atenção…"; na policlínica → entregue. Com `procedure_billing` ligado (Q11): paciente sem CNS → a frase do CNS, e o procedimento da bolsa aparece em "Procedimentos" da competência no 22. Com ele desligado, a bolsa sai sem essas conferências.
9. **Importação** (`municipal_admin`): baixar o modelo (o cabeçalho fixo da Q12); um `.xlsx` recusado antes; a planilha da semente gera o relatório, sem criar paciente; aplicar com step-up cria os pacientes que faltavam.

  Também: preset mobile 375×812 nos passos 4, 5 e 8; só teclado no passo 5; `resize_window` com o preset desktop ao terminar.

  Anote cada passo. Achado que muda comportamento vira teste na task dona antes da correção. Achado só visual vai ao usuário (memória "Interface em ciclo próprio").

---

### Task 15: Entrega — board, relato e parada antes do merge

**Files:** nenhum.

- [ ] **Step 1: Board.** Os cards do 27 que o dashboard toca vão a **Done** só se o api e o maintenance já terminaram os deles, e a **Verified** só depois da prova da Task 14, com o usuário.
- [ ] **Step 2: Suíte completa do api.** Não é desta sessão: a sessão dona do api pede à sessão "Testes" (`api · mod27s/feat/mod-27-supplies · --full · mudança transversal do 27 (estoque, consulta, faturamento)`). A suíte inteira do dashboard neste worktree (Task 14 Step 1) basta para o dashboard.
- [ ] **Step 3: Pare e peça autorização** para merge e push (depois do api e da suíte completa).
  - Antes do push, confira `$G -C $W log --oneline origin/main..HEAD` e `origin/main..main` (sessões paralelas).
  - O worktree só sai depois do merge: `$G -C $R/apps/dashboard worktree remove $W`.

**Relato:** `dashboard · dashboard-mod27s/feat/mod-27-supplies · Task 15 · suíte <n>/<n>, build ok, varreduras ok, prova <passos> · cards movidos · aguardando autorização de merge`.

---

## Self-review

1. **Cobertura do contrato §8 (dashboard), da spec §3 e das Q6–Q13:**
   - consulta, "Programas de insumos" com a cota à vista (Q7): Task 5; inscrições no painel e na linha do tempo (§7, Q7): Task 6;
   - "Retirada de insumos" (busca, inscrições e saldos, lote, exceção, terceiro, tira compatível; Q10): Task 7; devolução (§4, decisão 17): Task 8;
   - "Inscrição com laudo externo": Task 9; "Aparelhos em comodato": Task 10; "Regras de cota" (Q8) e modelos de aparelho (Q12): Task 11; "Importar pacientes dos programas" (Q12): Task 12;
   - estoque com o insumo (§2, §8, Q9) e o `tipo_produto` `insumo` do saldo inicial (Q12): Task 4; unidades (§6, Q11): Task 13; `SUPPLIES_ERRORS`: Task 2.
2. **Bordas que tocam a tela:** CID e perfil fora da faixa (Tasks 2 e 5); validade no fim do mês (Task 2); quantidade fracionária, grupo fora da cota e acima da cota (Tasks 2 e 7); terceiro pela metade (Tasks 2 e 7); bolsa sem devolução (Task 8); série repetida e boletim (Task 10); regra incompleta e ordem das regras (Tasks 2 e 11); planilha com BOM, grande ou errada (Task 12); UBS ou unidade sem CNES como ponto de estomia (Task 13).
3. **Tipos:** `SupplyProgramDraft`/`SupplyProgramAction`/`SupplyProgramView` (Tasks 1–2) são os da Task 5; `SupplyEnrollmentSummary`/`SupplyEnrollmentDetail` servem às Tasks 5 e 6; `WithdrawalLine`, `withdrawalBody` e `SupplyOverride` (Tasks 1–2) servem à Task 7; `ProfileFields` (Task 5) serve à Task 9; `StockItem.group` (Task 1) governa as Tasks 4, 7 e 8.
4. **Review Focus:** as cinco linhas têm teste na task dona (Tasks 7, 5, 8, 4 e 7).
5. **Proporção:** o código do plano é tipo, frase e teste; as telas vão por assinatura, props e comportamento.

## Divergências propostas ao contrato

Todas foram decididas no §11 do contrato (docs `7348ae0`), e o plano já aplica as decisões. O texto de cada uma fica como registro, resumido.

- **D1 — Resolvida pela Q7.** O bloco `supply_programs` não tinha forma (corpo, cota à vista, finalizada), e o `Finalize` zera `draft_items` (`apps/api app/commands/consultations/finalize.rb:64`). **Decisão:** a forma do Valor 5 do plano do api (até 4 linhas; `close` só com `enrollment_id`; `quota_overrides` e `override_reason`; sem versão no bloco); o `PATCH` devolve a cota calculada; a inscrição ganha `closed_in_consultation_id`. A finalizada não mostra o bloco (o `draft_items` é zerado); o painel relê as inscrições do paciente (Task 5).
- **D2 — Resolvida pela Q7.** Na consulta ninguém lia as inscrições: a única rota era da farmácia, e o prontuário em contexto não é relido depois da finalização (`ConsultationWorkspace.tsx:133-143`). **Decisão:** `GET /patients/:id/supply_enrollments` (lista sem CID nem perfil) para o painel e `GET /supply_enrollments/:id` (com CID e perfil) para quem atende em contexto; a Task 5 lê o detalhe para renovar.
- **D3 — Resolvida pela Q7.** A lista de CIDs do diabetes "da cidade" não tinha rota. **Decisão:** lista fixa em YAML no api, `E10`–`E14` e `O24`; na estomia, qualquer CID-10 válido.
- **D4 — Resolvida pela Q10.** O `override_by` não dizia de quem era o código, e o `SensitiveAction` só faz step-up da própria sessão (`src/components/SensitiveAction.tsx:75-76`). **Decisão:** `override { reason, pharmacist_user_id, code }`, com `GET /pharmacy/locations/:id/pharmacists` para escolher o farmacêutico; o farmacêutico na própria sessão manda só o `reason`; recusas `401 override_code_invalid` e `403 override_requires_pharmacist`.
- **D5 — Resolvida pela Q10.** Faltavam as formas da lista com saldos, da retirada e do laudo, os códigos de recusa e a dispensação com receita nula. **Decisão:** as do plano do api (Tasks 10 e 11): saldo dentro de `quotas[]` com `devices[]`; linhas com `supply_catalog_item_id`, `lot_id` e quantidade inteira; laudo com `CRM`/`COREN` e só `patient_id` (o mesmo `POST` reativa a inativa); `prescription: null` com `supply_enrollment` e `collector: { name }`; `quota_exceeded` pelo `path` da linha; as listas novas em `{ items, next_before }`.
- **D6 — Resolvida pela Q9.** A busca só conhecia `domain: "vector_control"` (`src/lib/api.ts:5055-5058`), e misturar levaria o insumo à receita. **Decisão:** a busca aceita `item_kind: "supply"`; entrada, posição e mínimo aceitam `supply_catalog_item_id`; o insumo é um domínio próprio de busca, separado da receita (Task 4, `ItemSearch kind`).
- **D7 — Resolvida pela Q12.** O comodato não tinha rota de modelos nem formas. **Decisão:** `device_models` gravado pelo admin, com step-up, e lido por quem entrega; as rotas e formas do empréstimo são as do plano do api (Task 14), com a lista por unidade sem nome do paciente nem boletim.
- **D8 — Resolvida pela Q8.** As regras só tinham o `PUT`. **Decisão:** regras só de acréscimo, com uma versão única do conjunto por programa; `GET` e `PUT` (`{ program, expected_version, rules }`), o `PUT` com step-up.
- **D9 — Resolvida pela Q12.** A importação não tinha rotas, cabeçalho nem códigos. **Decisão:** cabeçalho fixo do plano do api (`programa;cpf;cns;cid10;tipo_insulina;aplicacoes_dia;medicoes_dia;tipo_estoma;validade;cnes_unidade`), conferência em `dry_run` sem criar paciente, aplicação com step-up, sem "desfazer". O api não tem lista de importações: a tela abre a que acabou de enviar.
- **D10 — Resolvida pela Q7.** Os códigos da fonte `supply_enrollment` na linha do tempo. **Decisão:** os do plano do api (Task 17): entradas de `opened`, `renewed` e `closed`, com título e resumo prontos do api, sem CID; a inativação fica fora; a fonte não vai ao PEC externo.
